# Domain: Data Retention & nFADP Redaction

The legal-grounds-balance between Swiss right-to-erasure (nFADP, in force 2023-09-01) and Swiss Code of Obligations 10-year accounting retention. Locked in `reviews/2026-05-07-ceo-review.md` as SEC-4.

## The conflict

- **nFADP**: data subject can request erasure of personal data. Default response is delete-and-confirm.
- **Code of Obligations Art. 958f**: accounting records (invoices, contracts, books) must be retained for **10 years**.

You can't legally delete an invoice from 2024 in 2026 because a client requests it. But you also can't keep their name and address in the audit log forever.

## V1 policy: redact, don't delete

When a data-subject erasure request lands, replace PII fields with `[redacted-{user_id_hash}]` where `user_id_hash = first 8 hex of sha256(user_id || stable_id)`. The shape of the audit record stays, the accounting record stays, the PII goes.

| Table | Field(s) redacted | Field(s) kept |
|---|---|---|
| `people` | `full_name`, `email`, `phone`, `notes` | `id`, `stable_id`, `type`, `created_at`, `archived_at`, `redacted_at` |
| `audit_log` (rows where `entity_id` = the person) | `before_jsonb`, `after_jsonb` keys matching PII set | `id`, `stable_id`, `actor_user_id`, `entity`, `action`, `at` |
| `horse_documents` | file content (deleted from Storage); `filename` → token | `id`, `stable_id`, `horse_id`, `kind`, `expires_at`, `redacted_at` |
| `messages` | `body` → `[redacted-message]` | thread, timestamps, sender role |
| `invoice_lines` | `description` only if it matches a name pattern (manual review) | qty, price, vat, totals |
| `tab_lines` | `description` (rare PII; rule: only if owner explicitly typed a name) | qty, price, dates |

## What we don't redact

- `stables.name`, `stables.uid_mwst` — not personal data (legal entity)
- `services.*` — catalog content
- `invoices.total_*_cents`, dates, numbers, `qr_reference`, `status` — accounting data
- `vat_rates`, `feature_flags` — system data
- `time_punches` — labour records per ArGV 1 Art. 73 (separate 5-year retention from labour-law side)

## Implementation

Server action `redactPerson(personId)` runs as an atomic TX:

```ts
async function redactPerson(personId: string, requestedBy: string) {
  const hash = sha256(personId + stableId).slice(0, 8)
  const token = `[redacted-${hash}]`
  
  await db.transaction(async (tx) => {
    await tx.update(people).set({
      fullName: token, email: token, phone: token, notes: token,
      redactedAt: new Date(),
    }).where(eq(people.id, personId))
    
    await tx.execute(sql`
      update audit_log
        set before_jsonb = redact_jsonb(before_jsonb, ${token}),
            after_jsonb  = redact_jsonb(after_jsonb,  ${token})
        where entity = 'people' and entity_id = ${personId}
    `)
    
    // messages, horse_documents, invoice_lines sweeps
  })
  
  await logAudit({
    entity: 'people', entityId: personId,
    action: 'redact', actorKind: 'owner:redaction-request',
    actorUserId: requestedBy,
  })
}
```

`redact_jsonb` is a Postgres function defined in slice 00 that recursively swaps known PII keys (`full_name`, `email`, `phone`, `notes`) in JSONB trees.

## Retention timeline

- **Years 0–10**: hot data in primary DB
- **Years 5+**: `audit_log` yearly partitions can move to cold storage (slice 00 V1.1 archival), kept queryable via PITR restore
- **Years 10+**: partitions older than 10 years droppable per CO compliance; runbook-only, requires explicit owner confirmation

V1 does not implement automated archival or drop. Slice 00 lays the partition foundation; V1.1+ ships the cold-storage move.

## Worker labour records (separate timeline)

`time_punches`, `shifts`, `unavailability` retain for **5 years** per ArGV 1 Art. 73. After 5 years, the redact-not-delete policy applies — keep the audit shape, scrub the names.

## Customer-facing erasure flow

`/client/settings/datenschutz` → "Daten löschen anfordern":

1. Client submits request → email confirmation
2. **7-day grace period** (request revocable; copy: "Wir warten 7 Tage, falls du es dir anders überlegst.")
3. Stable owner approves (audit log: who approved, when)
4. Execution: server action runs, owner sees confirmation
5. Final state: client account inactive, can't log in (auth.users row also has `redacted_at`)

The data subject is the person being redacted. The stable owner approves because the data lives in the owner's tenant. If owner refuses, client can escalate to FDPIC (Swiss data protection authority) — documented in privacy notice.

## Privacy disclosure

Pre-pilot privacy notice (DE/FR/IT, pre-slice-00 TODO) names this policy explicitly: "We keep the accounting trace for 10 years per Swiss law. On your request, we replace your name and contact details with a non-identifying token; we never delete the financial record itself."

## Common mistakes

- Hard `DELETE` on `people` → breaks audit_log foreign keys + violates CO 10y
- Forgetting JSONB recursion → audit log retains the name in old `before_jsonb` blobs
- Reversible token (e.g., `[redacted-john-smith]`) → defeats the redaction
- Skipping the 7-day grace → user can't reverse a regret request
- Redacting `auth.users` row but not `people` row (or vice versa) → divergent state, identity leak

## When this doc updates

- New PII column on any table → add to the redaction list
- New retention requirement (employer obligations, GDPR if EU clients) → version this doc, route by stable jurisdiction
- Cold-storage archival ships (V1.1+) → document the move/drop procedure here
