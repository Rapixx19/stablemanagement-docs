# Slice 06 — Subscriptions and Running Tabs

**Phase:** Billing · **Estimate:** 4 dev-days · **Owner:** Sharad backend, freelancer frontend

The recurring billing engine + the per-client running tab. Each client has zero or more subscriptions (recurring services like full board) and a tab that accumulates one-off line items (lessons, transport, supplements) throughout the month.

## Goal

Owner sees, per client, a clean overview of (a) what recurring services the client pays, (b) what one-off items have been added this month, (c) the total. Two tab views: client-level (overview) and per-horse breakdown.

## What ships

### Owner side

1. **Client billing page** at `/people/[id]/billing`. Tabs: Subscriptions · Current tab · Invoice history.
2. **Subscriptions tab.** List active recurring services for this client. Add/edit/end-date a subscription. Per-subscription `auto_bill` toggle (auto-include in next month's invoice vs. manual review).
3. **Per-client tab toggle.** Master setting on the client: `auto_bill_default` (overall behaviour). Falls back to per-subscription if a sub overrides.
4. **Current tab (overview)** at `/people/[id]/billing/tab`. Shows current period (default = this calendar month). Total at top. Sections per horse (tab granularity = client overview + per-horse breakdown).
5. **Add line item modal.** Pick service from catalog OR free-text. Pick horse (or client-level if no horse). Pick date. Set qty and unit price (defaults from catalog). VAT code editable.
6. **Mobile quick-add** (owner-mobile only in V1). Two-tap flow: client → service → confirm. Common services pinned to top. Worker INSERT on `tab_lines` is **out of V1 scope** — see RLS section + `domains/v1-deferred.md`.
7. **Remove line item.** One tap, undo within 5 seconds (toast).
8. **Bulk subscription assignment** at `/billing/subscriptions/bulk-assign`. Three-step wizard:
   - **Step 1** — pick services from catalog (`DataTable` multi-select, filter by category).
   - **Step 2** — pick clients (`DataTable` multi-select, filter by active status + horse count, search by name).
   - **Step 3** — review grid: every `(client × service)` pair as a row with per-row override fields (`price_cents` defaults from catalog, `horse_id` if service needs one, `active_from` defaults to first of next period). **Duplicate detection**: any pair where `(person_id, service_id)` already has an active `client_subscriptions` row is pre-flagged "Wird übersprungen" (skipped). Owner can untick to skip, or tick "Ersetzen" to close-and-replace (atomic close of old + open of new in one TX).
   - **Save** — all rows inserted in **one transaction**, idempotent on retry via `(person_id, service_id, active_from)`. Toast summary: "17 angelegt, 3 übersprungen, 0 ersetzt." Audit log records bulk action with shared `at` timestamp + `actor_kind='owner:bulk-assign'`.
9. **Catalog price change cascade.** When owner edits `services.default_price_cents` and saves, **post-save `ConfirmDialog`** opens listing every affected active subscription: "12 aktive Abonnements bei CHF 850.00. Anpassen auf CHF 880.00?" Three actions:
   - **Ab nächster Periode** (recommended) — close affected subscriptions at current `period_end`, open replacements at next `period_start` at new price. Sets `client_subscriptions.replaced_by_id` so the chain is auditable. **Preserves current-period billing** — no retroactive change to in-flight tab lines.
   - **Sofort wirksam** — bulk `UPDATE client_subscriptions SET price_cents = $new WHERE ...` for affected rows. Affects current-period proration on the next cron run (subscription tab line is regenerated; this is why proration math is idempotent on `(person_id, service_id, period_start, period_end)`).
   - **Nur Katalog anpassen** — catalog `default_price_cents` updates, no subscriptions touched. Future bulk-assigns use new price.
   Optional **"Klienten informieren"** toggle drafts a slice 10 broadcast addressed to affected clients in their language.

### Client side

10. **My current tab** at `/billing/tab`. Read-only. Shows running tab for current period, breakdown per horse. Numeric columns use `JetBrains Mono` tabular figures right-aligned per `DESIGN.md`; status pills use the locked palette (`paid`=moss, `overdue`=ochre, `draft`=ink-300).

## Schema diff

```sql
-- 007_subscriptions_tabs.sql
create table client_subscriptions (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  person_id uuid not null references people(id),
  horse_id uuid references horses(id),
  service_id uuid not null references services(id),
  price_cents bigint not null,
  auto_bill boolean not null default true,
  active_from date not null,
  active_to date,
  notes text,
  replaced_by_id uuid references client_subscriptions(id),  -- price-change-cascade chain
  version int not null default 0,                           -- optimistic locking (E-CQ-2)
  created_at timestamptz not null default now()
);

-- Bulk-assign idempotency: prevent duplicate active subscription for same (person, service, active_from)
create unique index client_subscriptions_active_unique
  on client_subscriptions (person_id, service_id, active_from)
  where active_to is null;

create table tab_lines (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  person_id uuid not null references people(id),
  horse_id uuid references horses(id),
  service_id uuid references services(id),
  description text not null,
  qty numeric(10,2) not null default 1,
  unit_price_cents bigint not null,
  vat_code text not null references vat_rates(code),
  occurred_on date not null,
  period_start date,                          -- set on subscription lines (covers prorated cycles)
  period_end date,                            -- exclusive: line covers [period_start, period_end)
  invoice_id uuid,                            -- set when invoice generated
  origin text not null check (origin in ('subscription','one_off')),
  created_by_user_id uuid references auth.users(id),
  deleted_at timestamptz,                     -- soft delete for audit
  created_at timestamptz not null default now()
);

create index on tab_lines (stable_id, person_id, occurred_on);
create index on tab_lines (stable_id, person_id, invoice_id);
create unique index tab_lines_subscription_period_unique
  on tab_lines (person_id, service_id, period_start, period_end)
  where origin = 'subscription' and deleted_at is null;
create index on client_subscriptions (stable_id, person_id, active_to);
```

## RLS

All tables scoped by `stable_id` per the standard policy template in `domains/rls.md`.

- `client_subscriptions`, `tab_lines`: `owner` full access (SELECT/INSERT/UPDATE/DELETE).
- `worker`: **SELECT only** on both tables (operational visibility for the dashboard, no billing writes). Decided 2026-05-13: billing trust boundary stays at owner in V1. Worker catalog-INSERT path is logged as V1.1 (`domains/v1-deferred.md`).
- `client`: SELECT only their own rows — `tab_lines.person_id = (people row where user_id = auth.uid())`, same for `client_subscriptions`. No INSERT/UPDATE/DELETE.
- **Bulk operations** (items 8 + 9) check `auth.current_role(stable_id) = 'owner'` at the server-action layer **before** the transaction opens — RLS alone is insufficient because the action enumerates multiple `person_id`s and a single rejected row mid-batch would still appear in the log.

Cron writes (subscription-run) execute under the standard authenticated server-action path with a service-action context, not service-role — they look up `stable_id` per `client_subscriptions` row.

## Subscription materialisation

A monthly **pg_cron** job (`supabase/cron/subscription-run.sql`, runs `0 2 1 * *` Europe/Zurich) materialises subscription tab lines for the new period. Pure SQL — no RRULE expansion needed (always month boundary), so pg_cron is the right fit (vs Vercel Cron for slice 04 task materialisation per `DECISIONS.md` D7). Idempotent via the unique partial index `tab_lines_subscription_period_unique` defined in the schema diff above. Re-running produces zero new rows.

For mid-month subscriptions (`active_from = 15th`), **prorated by per-day rate using days-in-period** (so February uses /28 or /29, not /30): `floor(price_cents * days_active / days_in_period)`. `active_to` is **inclusive** (subscription active *through* that date). Helper in `packages/lib/billing/proration.ts` is the source of truth; the SQL materialisation function calls a single SQL helper that mirrors the TS logic exactly. Both sides share the same fixture-driven test suite.

Edge cases the proration helper must handle (each pinned in tests below): start mid-period; end mid-period; both in same period; February 28-day month; February 29-day leap month; period that is fully suspended (`active_to < period_start` → no line); subscription that ends on the last day of the period (full charge, no proration).

## Tab calculation

`getTab(personId, period)` returns:
```
{
  total_excl_vat_cents: number,
  total_vat_cents: number,
  total_incl_vat_cents: number,
  by_horse: [
    { horse: {...}, lines: [...], subtotal_cents: number },
    ...
  ],
  client_level_lines: [...],   // services not tied to a horse
  period_start, period_end
}
```

VAT computation: see `domains/vat.md`. Per line: `(qty * unit_price) * (1 + rate)`.

## Acceptance criteria

- [ ] Owner adds full-board subscription for client → next month's tab shows it
- [ ] **Proration: starts on the 15th** of a 30-day month at CHF 600 → CHF 320.00 (16 days × CHF 20/day), exact, no rounding drift
- [ ] **Proration: 28-day February** at CHF 600 monthly → daily rate CHF 21.43, mid-month start on Feb 15 → CHF 300.00 (14 days × 21.428... rounded once at the end)
- [ ] **Proration: 29-day leap February** (2028-02) → CHF 600 / 29 daily, half-month start on Feb 15 → CHF 310.34 (15 days)
- [ ] **Proration: subscription suspended mid-period** (active_to = 14th of a 30-day period) → CHF 280.00 (14 days), no double-billing
- [ ] **Proration: subscription ends on last day of period** (active_to = 30th of 30-day month) → full charge CHF 600, no proration
- [ ] Mid-month line item added → appears in current tab, removable within 5s
- [ ] Tab shows per-horse breakdown when client has 2+ horses
- [ ] Soft-deleted line removed from tab but visible in audit log
- [ ] Client viewing tab sees current month total with per-horse breakdown (mobile + desktop)
- [ ] **pg_cron job is idempotent across reruns** — calling the SQL function twice produces zero net new rows for the same period
- [ ] **Dashboard performance**: `getTab(personId, period)` returns in <500ms p95 with seed of 30 stables × 20 clients × ~50 tab_lines per period (≈30k rows). EXPLAIN shows Index Scan on `(stable_id, person_id, occurred_on)`, no Sort+Hash on the hot path
- [ ] **Bulk-assign atomic**: owner assigns 19 clients × 2 services in one transaction. Simulated DB error after row 15 → 0 rows committed (all-or-nothing). Retry produces the same 38 rows, not 53
- [ ] **Bulk-assign duplicate handling**: 19 clients selected but 3 already have an active sub for the same service → 16 new rows inserted, 3 skipped, toast reports "16 angelegt, 3 übersprungen"
- [ ] **Bulk-assign replace mode**: ticking "Ersetzen" on a duplicate pair closes the old subscription (`active_to = today`) and opens a new one (`active_from = today+1`) in one TX; old row's `replaced_by_id` points to the new row
- [ ] **Price-change "ab nächster Periode"**: 12 active subs at CHF 850 → 12 `active_to` set to current `period_end`, 12 new rows with `active_from = next period_start` at CHF 880; current-period tab_lines for May untouched
- [ ] **Price-change "sofort wirksam"**: bulk UPDATE bumps `price_cents`; next subscription-run cron regenerates the current-period subscription tab line at the new rate (idempotent partial index permits the rewrite within the same period bucket only via explicit cron re-run flag, otherwise leaves existing line — document the chosen flavour)
- [ ] **Price-change "nur Katalog"**: edits `services.default_price_cents` only; zero rows in `client_subscriptions` change; next bulk-assign uses new default
- [ ] **Price-change audit**: bulk operation writes one `audit_log` row per affected subscription with `actor_kind='owner:price-change'`, shared `at` timestamp clusters them queryably
- [ ] **Worker INSERT denied**: worker context calls `from('tab_lines').insert(...)` → RLS rejects, no row written, error returned to action layer; same test on `client_subscriptions`
- [ ] **Client-restricted SELECT**: client A in stable X cannot SELECT client B's `tab_lines` rows in stable X (client RLS pattern verified, not just cross-stable)
- [ ] **Bulk-op role gate**: a worker user with a forged client list calling bulk-assign server action is rejected by the role check **before** the transaction opens; zero audit_log rows written

## Acceptance integration test

`apps/web/tests/integration/owner/tabs.test.ts` — owner adds subscription, runs materialisation cron twice, asserts single line per period.

## Out of scope

- Discounts / credit notes (V1.1)
- Time-based pricing (peak vs off-peak lessons) (V1.5)
- Multi-currency line items (V2)
