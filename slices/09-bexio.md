# Slice 09 — Bexio Integration

**Phase:** Billing · **Estimate:** 6 dev-days · **Owner:** Sharad backend, freelancer frontend

Push-only sync to Bexio (the Swiss SMB accounting platform with ~80–100k users). Plus reconciliation poll so payment status flows back to us. Bexio is the most common Swiss bookkeeping target — supporting it is a clear differentiator from non-Swiss competitors.

## Goal

Owner connects their Bexio account once via OAuth. Confirmed invoices push to Bexio with line items + QR-bill PDF. A scheduled poll (every 30 min) updates our invoice status when Bexio marks paid.

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
  active boolean not null default true,
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

## Reconciliation poll details

- Runs every 30 min via **Vercel Cron** (`/api/cron/bexio-poll`, secured by a `CRON_SECRET` shared header — see `domains/secrets.md`)
- For each connected stable: fetch invoices updated since `last_poll_at`
- Match by `bexio_invoice_id` on our `invoices` table
- Update `status`, `paid_at` if Bexio reports paid
- Update `last_poll_at` only on success (per-stable; one stable's failure must not block others)
- Backoff on auth errors (token expired); refresh token, retry once. **OAuth refresh is serialized per stable** via a Postgres advisory lock keyed on `stable_id` to prevent two concurrent poll runs from both refreshing and invalidating each other's tokens.

## Bexio API quirks (handle)

- Rate limit: 50 requests / sec per OAuth token. Add a queue with 30 req/sec cap.
- No webhook support. Polling required.
- **Article (service) sync via `bexio_article_map`.** First push of a service creates the Bexio article and stores the mapping; subsequent pushes reuse the stored `bexio_article_id`. Prevents duplicate-article creation in the customer's Bexio account.
- **Refresh-token keep-alive.** Refresh tokens expire in 30 days. A nightly Vercel Cron at `/api/cron/bexio-keepalive` proactively refreshes any connection where `last_refresh_at < now() - interval '25 days'`, so a stable that doesn't push for a month doesn't lose its connection silently.
- QR-bill PDF can be generated by Bexio if their account is QR-IBAN configured. We push our PDF as attachment regardless — single source of truth is our app, not Bexio.

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
