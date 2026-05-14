# Slice 17 — Inventory and Reordering (basic)

> **DEFERRED TO V1.1** (decision 2026-05-13). Not built in V1 pilot. Spec kept intact here so V1.1 doesn't redo the work. Schema, RLS, acceptance criteria below are the V1.1 starting point. See `domains/v1-deferred.md` for rationale.

**Phase:** V1.1 · **Estimate:** 5 dev-days · **Owner:** Sharad backend, freelancer frontend

The daily-pain feature. Feed, bedding, medication, supplies — currently tracked on a clipboard. V1 ships stock-on-hand + worker-recorded consumption + one-tap reorder. Forecasting (consumption rate, time-to-stockout, auto-reorder) is V1.5 alongside the ML layer.

## Goal

Owner sees stock-on-hand at a glance with low-stock badges. Worker records "used 2 bales" in two taps. Items below threshold appear in a "Reorder" inbox; one tap creates draft purchase orders grouped by supplier; owner reviews and emails the supplier with a PDF attached.

## What ships

### Owner side

1. **Items list** `/inventory` — filter by category (feed/bedding/medication/supplement/supplies/other), search, low/critical badges.
2. **Item detail** `/inventory/items/[id]` — quantity, threshold, default supplier, 30-day sparkline + movements table, inline-edit threshold/reorder qty.
3. **Quick adjust** modal: `receive` / `adjust` / `count` with note → writes `inventory_movements`.
4. **Reorder inbox** `/inventory/reorder` — items below threshold, grouped by supplier. **One tap "Create draft POs"** spawns one draft per supplier.
5. **PO list / detail** `/inventory/orders` — review, edit lines, "Send" emails supplier (DE/FR/IT) with PO PDF.
6. **Receive flow.** Mark received → set `received_qty` per line → writes `kind='receive'` movements, updates `current_quantity`, status `received` or `partial`.
7. **Suppliers** `/inventory/suppliers` — CRUD.

### Worker side

8. **`/worker/inventory` quick consume.** Top tiles = 5 most-consumed items. Two-tap: Item → qty (1/2/5/custom) → confirm. Writes `kind='consume'` with `related_user_id`. Optimistic UI.

## Schema diff

```sql
-- 015_inventory.sql
create table suppliers (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  name text not null, contact_name text, email text, phone text, address text,
  language text check (language in ('de','fr','it')) default 'de',
  payment_terms_days int default 30, notes text, archived_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table inventory_items (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id), name text not null,
  category text not null check (category in ('feed','bedding','medication','supplement','supplies','other')),
  unit text not null check (unit in ('kg','g','litre','ml','bale','bag','bottle','dose','piece','box','sack')),
  current_quantity numeric(12,3) not null default 0,
  reorder_threshold numeric(12,3) not null default 0,
  reorder_quantity numeric(12,3),
  default_supplier_id uuid references suppliers(id),
  default_unit_price_cents bigint, vat_code text references vat_rates(code),
  notes text, archived_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

-- Append-only ledger. Truth = sum(delta). current_quantity is a trigger-maintained cache.
create table inventory_movements (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  item_id uuid not null references inventory_items(id),
  kind text not null check (kind in ('receive','consume','adjust','count')),
  delta numeric(12,3) not null, quantity_after numeric(12,3) not null,
  occurred_on date not null default current_date,
  occurred_at timestamptz not null default now(),
  related_purchase_order_id uuid,
  related_user_id uuid references auth.users(id), note text,
  created_at timestamptz not null default now()
);

create table purchase_orders (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  supplier_id uuid not null references suppliers(id),
  number text not null,                    -- "PO-2026-0042"
  status text not null default 'draft' check (status in ('draft','sent','received','partial','cancelled')),
  ordered_at timestamptz, expected_at date, received_at timestamptz,
  total_excl_vat_cents bigint not null default 0,
  total_vat_cents bigint not null default 0,
  total_incl_vat_cents bigint not null default 0,
  language text not null check (language in ('de','fr','it')),
  notes_to_supplier text, created_by_user_id uuid references auth.users(id),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (stable_id, number)
);

create table purchase_order_lines (
  id uuid primary key default gen_random_uuid(),
  purchase_order_id uuid not null references purchase_orders(id) on delete cascade,
  stable_id uuid not null, item_id uuid not null references inventory_items(id),
  description text not null,                -- frozen at PO time
  qty numeric(12,3) not null, received_qty numeric(12,3) not null default 0,
  unit_price_cents bigint not null,
  vat_code text not null references vat_rates(code), vat_rate numeric(5,3) not null,
  line_total_excl_vat_cents bigint not null, line_total_vat_cents bigint not null,
  sort_order int not null default 0
);

alter table inventory_movements add constraint fk_movements_po
  foreign key (related_purchase_order_id) references purchase_orders(id) on delete set null;

create index on inventory_movements (stable_id, item_id, occurred_at desc);
create index on inventory_items (stable_id, category, archived_at);
create index on inventory_items (stable_id) where current_quantity <= reorder_threshold;
create index on purchase_orders (stable_id, status, ordered_at desc);
create index on purchase_order_lines (purchase_order_id);
create index on suppliers (stable_id, archived_at);
```

## RLS + ledger discipline

All five tables scoped by `stable_id`. `inventory_items`, `suppliers`, `purchase_orders`, `purchase_order_lines`: `owner`/`manager` write, `worker` SELECT only on items + suppliers. `inventory_movements`: `owner`/`manager` INSERT any kind, `worker` INSERT only `kind='consume'`. **Append-only — no UPDATE/DELETE for any role.** Corrections go through `kind='adjust'`.

`current_quantity` is a denormalised cache. Truth = `sum(inventory_movements.delta)` per item. Trigger on movement insert updates atomically. **Nightly Vercel Cron** at `/api/cron/inventory-drift` compares cache vs ledger; mismatch pages Slack.

## Acceptance criteria

- [ ] Item threshold 20, stock 50 → no badge; consume 35 → low badge appears
- [ ] Reorder inbox groups low items by default supplier; one-tap creates one draft PO per supplier with `qty = reorder_quantity`
- [ ] "Send" emails supplier in their language with PO PDF attached
- [ ] Receive flow writes `kind='receive'` movements + updates `current_quantity`
- [ ] Drift check: cache equals `sum(deltas)` for every item
- [ ] Worker cannot UPDATE/DELETE movements (RLS-blocked); cross-tenant isolation on all five tables
- [ ] DE/FR/IT labels; worker quick-consume < 2 taps; `/worker/inventory` keeps bundle < 150KB gzip

## Acceptance integration test

`apps/web/tests/integration/owner/inventory.test.ts`

```ts
test('low-stock → one-tap PO → receive cycle', async () => {
  const { ownerCtx, workerCtx, supplier } = await seed.stableWithSupplier();
  const item = await createItem(ownerCtx, { name: 'Stroh-Ballen', category: 'bedding',
    unit: 'bale', threshold: 20, defaultSupplierId: supplier.id, reorderQty: 50 });
  await receiveStock(ownerCtx, { itemId: item.id, qty: 50 });
  await consume(workerCtx, { itemId: item.id, qty: 35 });
  expect((await getItem(ownerCtx, item.id)).current_quantity).toBe(15);
  const [po] = await createReorderPOs(ownerCtx);
  await sendPO(ownerCtx, { poId: po.id });
  expect(emailQueue.last().to).toBe(supplier.email);
  await receivePO(ownerCtx, { poId: po.id });
  expect((await getItem(ownerCtx, item.id)).current_quantity).toBe(65);
});
```

## Out of scope (V1.5+)

- Forecasting, dynamic thresholds, auto-reorder triggers (V1.5 ML layer)
- Multi-supplier per item, lot/batch tracking, expiry per lot (V1.1)
- Barcode scanning, direct supplier API integrations (V2)
