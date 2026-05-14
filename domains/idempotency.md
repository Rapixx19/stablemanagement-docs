# Domain: Idempotency Patterns

Four patterns. Pick the right one per situation. Locked in `reviews/2026-05-07-eng-review.md` as E-CQ-1.

## Why

External APIs retry. Vercel Cron retries. Browsers retry. Networks drop mid-request. Without idempotency, every retry is a potential duplicate insert, duplicate side-effect, double-billed line item, double-pushed invoice. Pick the pattern that matches the operation's nature. Don't try to make one pattern fit all four.

## Pattern 1 — DB unique index (for inserts)

Use when: the same logical operation replayed should produce zero net new rows.

Example: slice 04 task materialisation. Nightly cron expands RRULEs into `tasks` rows. Crash mid-run or Vercel retry must not create duplicates.

```sql
create unique index tasks_template_due_unique
  on tasks (template_id, due_date, coalesce(due_time, time '00:00'))
  where template_id is not null;
```

Server code uses `INSERT ... ON CONFLICT DO NOTHING`. Retry-safe by construction.

Slice 06 uses the same pattern for subscription materialisation: `tab_lines_subscription_period_unique (person_id, service_id, period_start, period_end) WHERE origin='subscription'`.

## Pattern 2 — Server-side advisory key (for external API calls)

Use when: an external system is the source of truth and you need to prevent duplicate side-effects.

Example: slice 09 Bexio push. Pushing the same invoice twice should return the same Bexio invoice ID, not create two.

```ts
async function pushToBexio(invoiceId: string, stableId: string) {
  const key = `bexio:${stableId}:invoice:${invoiceId}`
  
  // log-first lookup: if we already pushed, return the stored Bexio ID
  const existing = await db.select().from(bexioSyncLog)
    .where(and(
      eq(bexioSyncLog.stableId, stableId),
      eq(bexioSyncLog.entityId, invoiceId),
      eq(bexioSyncLog.status, 'ok'),
    )).limit(1)
  if (existing.length) return existing[0].bexioId

  // pg advisory lock to serialize per key within this stable
  await db.execute(sql`select pg_advisory_xact_lock(hashtext(${key}))`)
  // ... do the push, log the result ...
}
```

The advisory lock prevents two concurrent pushes for the same key from both creating Bexio invoices. The log lookup catches retries after the lock TX commits.

## Pattern 3 — Content-hash (for caches and artifacts)

Use when: the same content should produce the same artifact, deterministically.

Example: slice 08 PDF generation. Generating the same invoice twice produces byte-identical bytes; we hash the invoice inputs and key the Storage path by the hash.

```ts
const inputHash = sha256({
  invoiceId,
  version: invoice.version,  // critical: include version so stale-input misses
  lines: invoice.lines,
  total: invoice.totalInclVatCents,
}).slice(0, 16)

const storagePath = `${stableId}/invoices/${invoiceId}/${inputHash}.pdf`
// Check Storage before generating
const exists = await storage.exists(storagePath)
if (exists) return storagePath
// Otherwise generate, write to storagePath
```

If the inputs change (`version` bump), the hash changes → new path. No accidental cache hit on stale inputs. Old artifacts garbage-collected by a V1.1 cron (acceptable for V1 to leave them).

## Pattern 4 — Sequence with row lock (for monotonic IDs)

Use when: generating a strictly monotonic, gap-free identifier per tenant.

Example: slice 08 invoice numbers (`2026-0142`). Sequential per stable per year, no gaps, no duplicates, even under concurrent generation.

```sql
create or replace function next_invoice_number(p_stable_id uuid, p_year int)
returns text language plpgsql as $$
declare
  v_seq int;
begin
  -- explicit row lock; serializes concurrent callers
  select seq into v_seq from invoice_sequences
    where stable_id = p_stable_id and year = p_year
    for update;
  if v_seq is null then
    insert into invoice_sequences (stable_id, year, seq) values (p_stable_id, p_year, 1);
    return format('%s-%s', p_year, lpad('1', 4, '0'));
  end if;
  update invoice_sequences set seq = v_seq + 1
    where stable_id = p_stable_id and year = p_year;
  return format('%s-%s', p_year, lpad((v_seq + 1)::text, 4, '0'));
end;
$$;
```

`for update` blocks concurrent callers until the TX commits. Postgres native `SEQUENCE` is gap-prone under TX rollback — explicit lock + counter table is safer for accounting IDs.

## Decision table

| Operation shape | Pattern |
|---|---|
| Insert that may replay | 1 (unique index) |
| External API mutation | 2 (advisory key + log) |
| Expensive deterministic output | 3 (content hash) |
| Per-tenant monotonic ID | 4 (sequence with row lock) |

## Common mistakes

- Pattern 1 without `stable_id` in the unique key → cross-tenant leaks isolation
- Pattern 2 without the log table → can't tell "already done" from "lost in transit"
- Pattern 3 without including `version` in the hash → stale-input cache hit
- Pattern 4 using Postgres `SEQUENCE` → TX rollback leaves gaps (illegal for accounting)
- Layering patterns (advisory lock + unique index for the same op) when one would do → unnecessary

## When this doc updates

- A 5th pattern is genuinely needed → add it with a real example
- A pattern is retired (consolidated into another) → mark deprecated, leave the entry
