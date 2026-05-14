# Slice 09 — Bexio Integration

**Phase:** Billing · **Estimate:** 6 dev-days · **Owner:** Sharad backend, freelancer frontend

Push-only sync to Bexio (the Swiss SMB accounting platform with ~80–100k users). Plus reconciliation poll so payment status flows back to us. Bexio is the most common Swiss bookkeeping target — supporting it is a clear differentiator from non-Swiss competitors.

## Goal

Owner connects their Bexio account once via OAuth. Confirmed invoices push to Bexio with line items + QR-bill PDF. A scheduled poll (every 30 min) updates our invoice status when Bexio marks paid.

## Scope and choice

Bexio is an **optional** integration in V1 — not a hard dependency. During onboarding (slice 16) and at any time after via `/owner/settings/billing`, the stable picks one of two accounting workflows:

- **Bexio** — full push + 30-min reconciliation (the path this slice specs). Recommended for the ~80% of CH SMB stables already on Bexio.
- **Kein/Anderes** — no automated integration. Owner uses the **manual mark-paid** action + **invoice CSV export** (both specced in slice 08) to bridge into their own bookkeeping tool (Banana, KLARA, CashCtrl, paper). Reconciliation is manual: owner sees the bank statement, ticks the invoice paid in our app.

The choice is **per-stable**, not per-invoice. Switching from Bexio to Kein revokes tokens and stops the poll (reversible — re-OAuth restores the connection). Default = decided at pilot kickoff with the stable owner; La Fattoria's default captured in the slice 16 onboarding script.

When stable has no active `bexio_connections` row, everything in this slice is dormant — no cron poll, no push button, no sync-issues panel entries from this slice. The settings page renders the "Kein" state with a "Mit Bexio verbinden" CTA.

## What ships

### Owner side

1. **Connect Bexio** at `/settings/integrations/bexio`. OAuth 2.0 flow (authorization code grant). Required scopes: `contact_show contact_edit kb_invoice_show kb_invoice_edit article_show article_edit accounting`.
2. **Connection status panel.** Shows: connected / disconnected, account name, last sync time, recent sync errors.
3. **Per-invoice push.** When invoice is confirmed in slice 08, "push to Bexio" runs automatically (configurable: auto / manual). On error, marked as `bexio_sync_failed` and shown in admin alerts.
4. **Push retry**. Failed pushes appear in `/billing/invoices/sync-issues` with a manual retry button.
5. **Customer mapping.** First push for a client creates the Bexio contact if not linked; subsequent pushes use stored `bexio_contact_id` on `people`.

### Backend

6. **Reconciliation poll** (**Vercel Cron**, every 30 min, hits `/api/cron/bexio-poll`). External-API job — outside Postgres, so Vercel Cron is the right fit (pg_cron is reserved for DB-internal work). Lists Bexio invoices updated since last poll. For each match (by `bexio_invoice_id`), if Bexio status = paid and ours ≠ paid → update ours, set `paid_at`, write audit log.
7. **Idempotent push.** Idempotency key = `stable_id + invoice_id`. Repeat push on same key returns the existing remote ID.
8. **Dead-letter queue.** Pushes that exceed retry budget (5 retries with exponential backoff) land in `bexio_sync_log` with `status='error'` and surface in `/billing/invoices/sync-issues`. Owner can manually retry from the UI; engineering paged on > 10 DLQ entries in 1h via Sentry.

## Schema diff

```sql
-- 009_bexio.sql
create table bexio_connections (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  oauth_access_token bytea not null,         -- pgp_sym_encrypt output (per DECISIONS.md D8)
  oauth_refresh_token bytea not null,
  oauth_expires_at timestamptz not null,
  bexio_company_id text,
  scopes text not null,
  last_sync_at timestamptz,
  last_poll_at timestamptz,
  last_refresh_at timestamptz,               -- proactive keep-alive on day 25 of refresh-token life
  last_error text,
  last_error_class text,                       -- 'AdvisoryLockTimeout', 'BexioRefreshTokenRace', etc.
  active boolean not null default true,
  version int not null default 0,              -- optimistic locking (E-CQ-2)
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (stable_id)
);

-- Maps our service catalog to Bexio articles to prevent duplicate-article creation on push
create table bexio_article_map (
  stable_id uuid not null references stables(id),
  service_id uuid not null references services(id),
  bexio_article_id text not null,
  created_at timestamptz not null default now(),
  primary key (stable_id, service_id)
);

create table bexio_sync_log (
  id bigserial primary key,
  stable_id uuid not null references stables(id),
  entity text not null check (entity in ('contact','invoice')),
  entity_id uuid not null,
  bexio_id text,
  action text not null check (action in ('push','poll','retry')),
  status text not null check (status in ('ok','error')),
  error_message text,
  request_jsonb jsonb,
  response_jsonb jsonb,
  at timestamptz not null default now()
);

create index on bexio_sync_log (stable_id, entity, entity_id);
```

## Encryption of OAuth tokens

Tokens encrypted at rest using `pgcrypto` symmetric encryption (`pgp_sym_encrypt`/`pgp_sym_decrypt`) with the key in a Vercel env var (`BEXIO_TOKEN_ENCRYPTION_KEY`, marked `Sensitive`). **Columns are `bytea`, not `text`** (per `DECISIONS.md` D8). The key is passed in by the server action on every call via `set_config('app.crypto_key', $1, true)` (txn-local) inside the same TX as the decrypt — required because the transaction-mode pooler discards session state between transactions. Never logged, never sent to Sentry, scrubbed from request snapshots. Recovery + rotation procedure: `runbooks/bexio-token-recovery.md`. See `domains/secrets.md`.

## RLS

All three tables scoped by `stable_id` per `domains/rls.md`. Bexio is owner-business data — workers and clients have no access.

- `bexio_connections`: `owner` SELECT/UPDATE (UPDATE limited to `active` flag — full row mutation goes through OAuth flow). **`worker` / `client`: no access.** Plaintext tokens never returned by SELECT (the `bytea` columns are returned but never decrypted in user-facing reads — only inside server actions that need them).
- `bexio_article_map`: `owner` full access. `worker`/`client`: no access.
- `bexio_sync_log`: `owner` SELECT only. INSERT happens via server actions / cron under the service-action context. **No UPDATE/DELETE for any role** — the log is append-only.

## Reconciliation poll details

- Runs every 30 min via **Vercel Cron** (`/api/cron/bexio-poll`, secured by a `CRON_SECRET` shared header — see `domains/secrets.md`)
- For each connected stable: fetch invoices updated since `last_poll_at`
- Match by `bexio_invoice_id` on our `invoices` table
- Update `status`, `paid_at`, **`paid_source='bexio'`** if Bexio reports paid (distinguishes from manual mark-paid path, slice 08)
- Update `last_poll_at` only on success (per-stable; one stable's failure must not block others)
- Backoff on auth errors (token expired); refresh token, retry once. **OAuth refresh is serialized per stable** via a Postgres advisory lock keyed on `stable_id` to prevent two concurrent poll runs from both refreshing and invalidating each other's tokens. Lock acquired with `pg_try_advisory_xact_lock(hashtext('bexio-refresh:' || stable_id::text))` + 5-second SELECT timeout — if not acquired, log `AdvisoryLockTimeout` rescue path (see below), skip this run, retry next cycle. If 3 consecutive cycles miss the lock, surface in connection panel + page Sharad.

## Bexio API quirks (handle)

- Rate limit: 50 requests / sec per OAuth token. Add a queue with 30 req/sec cap.
- No webhook support. Polling required.
- **Article (service) sync via `bexio_article_map`.** First push of a service creates the Bexio article and stores the mapping; subsequent pushes reuse the stored `bexio_article_id`. Prevents duplicate-article creation in the customer's Bexio account.
- **Refresh-token keep-alive.** Refresh tokens expire in 30 days. A nightly Vercel Cron at `/api/cron/bexio-keepalive` proactively refreshes any connection where `last_refresh_at < now() - interval '25 days'`, so a stable that doesn't push for a month doesn't lose its connection silently.
- QR-bill PDF can be generated by Bexio if their account is QR-IBAN configured. We push our PDF as attachment regardless — single source of truth is our app, not Bexio.

## Failure modes & rescue paths

Every failure class is named, caught, and surfaced via `/billing/invoices/sync-issues`. All rescue paths log to Sentry with `stable_id`, `entity`, `entity_id`, `error_class`, set `bexio_connections.last_error` + `last_error_class`, and write a `bexio_sync_log` row with `status='error'`.

| Error class | Trigger | UX | Recovery |
|---|---|---|---|
| `AdvisoryLockTimeout` | Cannot acquire per-stable refresh lock within 5s | Silent on first miss; connection panel banner after 3 consecutive misses; Sharad paged via Slack | Lock auto-releases at end of holding TX; next 30-min cycle retries. If persistent, investigate stuck cron handler |
| `BexioRefreshTokenRace` | Refresh-token API returns `invalid_grant` despite advisory lock (rare leak-through) | Connection auto-marked `active=false`; banner: "Bexio-Verbindung verloren — bitte neu autorisieren"; pages Sharad | Owner re-runs OAuth from `/settings/integrations/bexio`; `runbooks/bexio-token-recovery.md` |
| `BexioLineValidationError` | Bexio returns 422 on push with field-level error (e.g., VAT code mismatch) | Sync-issues row with parsed field path + deep-link to the offending catalog/service entry; copy: "Position N: VAT-Code stimmt nicht mit Bexio überein — Katalog prüfen" | Owner fixes catalog VAT or contact data; manual retry button |
| `BexioRateLimit429` | 6 consecutive 429 responses within the per-token queue | Push lands in DLQ; sync-issues row; no immediate owner action required | Auto-retry with exponential backoff; if DLQ row count > 10/h, page Sharad |
| `BexioApiUnreachable` | Network timeout / 5xx from Bexio (not 4xx) | Sync-issues row "Bexio temporär nicht erreichbar"; auto-retries 5× over 1h | If still failing after 1h, surface as connection-panel banner |

**DLQ overflow alert (operational)**: Sentry rule `bexio_sync_log` rows where `status='error' AND at > now() - interval '1h' COUNT > 10` → page Sharad on Slack `#stable-prod`. Also surfaced in the connection panel as "N Synchronisationsfehler in der letzten Stunde."

## Acceptance criteria

- [ ] Owner connects Bexio via OAuth; connection panel shows account name
- [ ] Confirmed invoice pushes to Bexio within 60s; UI shows "synced" badge
- [ ] Push includes all line items with correct VAT, our QR-bill PDF as attachment
- [ ] First push for a new client creates Bexio contact and stores `bexio_contact_id`
- [ ] Repeating the push of the same invoice returns the same Bexio ID (idempotency)
- [ ] Owner pays the invoice in Bexio → within 30 min our status updates to `paid`
- [ ] OAuth token expiry triggers refresh transparently
- [ ] **OAuth refresh race**: simulate two concurrent poll runs for the same stable with a near-expired token; only one refresh call is sent to Bexio, both runs proceed with the same fresh token, no token invalidation. Asserted via mock Bexio that fails the test if it sees > 1 refresh call within 5s for the same stable.
- [ ] **Rate-limit DLQ**: mock Bexio returns 429 for 6 consecutive push attempts; the push lands in DLQ with `bexio_sync_log.status='error'` and surfaces in `/billing/invoices/sync-issues`. Manual retry succeeds when mock returns 200.
- [ ] **Contract test**: replay a recorded cassette (Pact-style or `nock`) of the Bexio API responses we depend on (push invoice, list invoices, get contact, refresh token). CI fails if Bexio's response shape diverges from the cassette and we have not updated the parser.
- [ ] **Orphan handling**: Bexio returns an invoice in the poll that has no matching `bexio_invoice_id` in our DB → we log it, do not crash, do not create a phantom invoice. Surface count of orphans in the connection panel.
- [ ] Disconnecting Bexio in settings revokes tokens and stops polling
- [ ] Push to a disconnected Bexio shows a clear error, doesn't silently fail
- [ ] **`AdvisoryLockTimeout` rescue**: held lock simulated for 6s → next cron run logs `AdvisoryLockTimeout`, `last_error_class` set, skips, returns success-no-op (does not crash)
- [ ] **`AdvisoryLockTimeout` escalation**: 3 consecutive missed cycles → connection panel banner shown, Sentry alert fired
- [ ] **`BexioLineValidationError` rescue**: mock 422 with `lines[3].vat_code` field-error → sync-issues row shows "Position 4: VAT-Code stimmt nicht mit Bexio überein" with deep-link to the offending service in catalog
- [ ] **`BexioRefreshTokenRace` rescue**: mock returns `invalid_grant` post-lock → connection auto-marked `active=false`, banner shown, Sentry alert fired, retry only succeeds after owner re-OAuths
- [ ] **DLQ overflow alert**: insert 11 `bexio_sync_log` rows with `status='error'` in a 1h window → Sentry rule fires, Slack `#stable-prod` receives page (mocked in CI via Sentry webhook capture)
- [ ] **Token-leak audit**: full `audit_log` snapshot after 10 push+poll cycles → no `oauth_access_token` / `oauth_refresh_token` raw value appears anywhere; same check on Sentry breadcrumb capture
- [ ] **Article-map dedup**: push the same service in 5 invoices → exactly 1 `bexio_article_map` row, exactly 1 Bexio article created (mock asserts 1 article-create call across 5 pushes)
- [ ] **No worker/client access**: worker context SELECT on `bexio_connections` returns 0 rows; same for `bexio_sync_log`, `bexio_article_map`

## Acceptance integration test

`apps/web/tests/integration/owner/bexio-push.test.ts`

```ts
test('push is idempotent and reconciliation updates status', async () => {
  const { ownerCtx, invoice } = await seed.confirmedInvoice();
  await mockBexio.expectPush();
  const r1 = await pushToBexio(ownerCtx, { invoiceId: invoice.id });
  const r2 = await pushToBexio(ownerCtx, { invoiceId: invoice.id });
  expect(r1.bexioInvoiceId).toBe(r2.bexioInvoiceId);
  await mockBexio.markPaid(r1.bexioInvoiceId);
  await runReconciliationPoll();
  const updated = await getInvoice(ownerCtx, invoice.id);
  expect(updated.status).toBe('paid');
});
```

## Out of scope

- Abacus integration (V2 — for stables > 50 horses)
- Banana / KLARA / CashCtrl direct integration (V1.1 — CSV export covers the gap)
- eBill dispatch via Bexio (V1.1)
- Bidirectional contact sync (V1.1 — push-only in V1)
