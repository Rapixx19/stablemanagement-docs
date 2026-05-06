# Slice 02 — Stall Map

**Phase:** Operational core · **Estimate:** 5 dev-days · **Owner:** Sharad backend, freelancer frontend

The visual identity of the product. A grid layout of the physical barn where each cell is a stall, paddock, or maintenance area. Click a stall, see the horse. Owner manages physical layout once; everyone uses it as the navigation surface.

## Goal

Owner defines the barn layout (rows × cols) once during onboarding, places stalls into cells, labels them. Stalls can be assigned to horses with a from/to history. The map renders in 3 places: dashboard tile (slice 16), the dedicated stall map page, and read-only on the client side.

## What ships

### Owner side

1. **Stall map editor** at `/stalls`. Grid of cells (rows × cols). Each cell can hold a stall, paddock, or be empty. Drag horses from a sidebar onto stalls. Click a stall to: assign horse, mark for maintenance, mark vacant, edit code/label.
2. **Stall states**, color-coded:
   - Occupied (default)
   - Featured (manually flagged for visual emphasis)
   - Attention (alert badge — surfaced from horse expiry, vet due, etc. in slice 16)
   - Vacant (dashed hatching)
   - Maintenance (greyed out)
3. **Layout setup wizard** during stable onboarding. "How many aisles? How many stalls per aisle? Any paddocks?" → generates initial grid.
4. **Stall detail drawer**: opens on click. Shows stall code, current horse (link to `/horses/[id]`), assignment history, notes, edit controls.
5. **Dashboard tile**: slice 16 reuses this component at smaller scale.

### Client side

6. **Read-only map** at `/stalls`. Owner's stalls visible. Owner stall is highlighted. Click → opens client-side `/horses/[id]`.

## Schema diff

```sql
-- 003_stalls.sql
create table stalls (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  code text not null,                          -- "A·01"
  row int not null,
  col int not null,
  kind text not null check (kind in ('stall','paddock','maintenance','empty')),
  capacity int default 1,
  active boolean not null default true,
  notes text,
  created_at timestamptz not null default now(),
  unique (stable_id, code)
);

create table horse_assignments (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  horse_id uuid not null references horses(id),
  stall_id uuid not null references stalls(id),
  starts_on date not null,
  ends_on date,
  notes text,
  created_at timestamptz not null default now()
);

create index on horse_assignments (stable_id, horse_id, ends_on);
create index on horse_assignments (stable_id, stall_id, ends_on);
```

A horse's current stall = `horse_assignments` row where `ends_on is null`. Reassignment closes the previous row (sets `ends_on`) and inserts a new row in the same transaction.

## RLS

- `stalls`, `horse_assignments`: scoped by `stable_id`
- Workers can SELECT but not UPDATE/INSERT/DELETE in V1
- Clients can SELECT only — no edit

## V1 constraints

- **Grid-based only.** Free-form canvas (drag arbitrary shapes, upload barn photo) is V2. Document this clearly in onboarding so expectations are set.
- Max grid: 30 rows × 30 cols (900 cells). Larger barns split into multiple stable tenants — unlikely in CH SMB.
- Stall code uniqueness scoped to stable.
- Reassigning a horse closes the old assignment in the same transaction. No two open assignments per horse.

## Acceptance criteria

- [ ] Onboarding wizard generates initial stall grid from "2 aisles × 10 stalls each"
- [ ] Owner can add/edit/delete stalls visually
- [ ] Drag a horse onto an occupied stall → confirm modal "swap or replace?"
- [ ] Reassigning a horse correctly closes the previous assignment
- [ ] Vacant stall renders with dashed hatch
- [ ] Attention badge appears when horse has document expiring in 30 days (cross-slice with 01)
- [ ] Client sees their horse highlighted on read-only map
- [ ] Mobile (<640px): map collapses to a vertical aisle list, still usable

## Acceptance integration test

`apps/owner/tests/integration/stalls.test.ts`

```ts
test('reassigning a horse closes prior assignment atomically', async () => {
  const { ownerCtx, horseId, stallA, stallB } = await seed.horseInStall();
  await reassignHorse(ownerCtx, { horseId, toStallId: stallB.id });
  const assignments = await getAssignments(ownerCtx, { horseId });
  expect(assignments.filter(a => !a.ends_on)).toHaveLength(1);
  expect(assignments.find(a => a.stall_id === stallA.id).ends_on).not.toBeNull();
});
```

## Out of scope

- Free-form canvas editor (V2)
- Photo background of the barn (V2)
- Group paddock turnout assignments (slice 03 owns turnout events)
