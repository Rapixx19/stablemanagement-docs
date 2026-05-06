# Slice 14 — Time Clock and Geofence

**Phase:** Worker · **Estimate:** 6 dev-days · **Owner:** Sharad backend, freelancer frontend

The labour-records-of-record. Workers clock in / out / break. Geofence verifies on-site without storing GPS. Hours log exportable to CSV per pay period. Swiss labour law (Arbeitsgesetz Art. 46, ArGV 1 Art. 73) compliant. Note: full Treuhänder review deferred until first external paying stable; La Fattoria pilot uses internally, low risk.

## Goal

Marco taps "clock in" at 06:38. Browser GPS pings, server checks if within 100m of stable's geofence center, records boolean `geo_within_fence = true`, never stores raw coordinates. Owner sees "Marco: clocked in 2h 14m, on-site" in real time. End of pay period, owner exports CSV: worker, day, in/out, break, total hours.

## What ships

### Worker side

1. **Clock card** on Home (worker density per `DESIGN.md`). States: not-clocked-in (big terracotta "Clock in" button, 52px tall, full-bleed card), clocked-in (live timer in `JetBrains Mono` tabular + on-site badge in `--moss-600` + Break / Clock-out secondary buttons), on-break (paused timer, ochre tint).
2. **Geofence prompt.** First clock-in asks for location permission once. If denied, owner gets notified; manual approval flow available.
3. **Off-site flag.** Punch from outside the geofence: still recorded, but `geo_within_fence = false`, badge shows "Off-site, manual review" in `--rust-600` (DESIGN.md error/lost-signal token), owner notified, owner can approve.
4. **Break tracking.** Break time excluded from total hours. Break start/end recorded as separate punches.
5. **Auto-punch-out at 23:59.** If a worker forgets to clock out, system inserts a punch at 23:59 with note "auto-closed" and notifies owner + worker.

### Owner side

6. **Live status panel** at `/workers`. Each worker: clocked in (since when, on/off-site) or last seen. Refresh in realtime.
7. **Hours log** at `/workers/[id]/hours`. Calendar grid of days × workers with total hours per cell. Click for daily breakdown.
8. **CSV export** per pay period (default monthly). Columns: worker, date, clock_in, clock_out, breaks, total_hours, geo_within_fence_ratio. Importable into Swiss payroll software.

## Schema diff

```sql
-- 013_timeclock.sql
create table stable_geofence (
  stable_id uuid primary key references stables(id),
  center_latitude numeric(10,7) not null,
  center_longitude numeric(10,7) not null,
  radius_meters int not null default 100,
  updated_at timestamptz not null default now()
);

create table time_punches (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  worker_user_id uuid not null references auth.users(id),
  punched_at timestamptz not null,
  kind text not null check (kind in ('in','out','break_start','break_end','auto_out')),
  geo_within_fence boolean,                    -- null when location denied
  notes text,
  approved_by_user_id uuid references auth.users(id),  -- for off-site overrides
  approved_at timestamptz,
  created_at timestamptz not null default now()
);

create index on time_punches (stable_id, worker_user_id, punched_at);
```

**Critical: no raw GPS coordinates stored.** The browser pings the geofence-check function, which returns `{ within: true | false }`. Only the boolean is persisted.

## Privacy and labour law (V1 minimum)

- nFADP compliance: only `geo_within_fence` boolean persisted, never raw coords
- Worker consent (slice 12) explains this in plain DE/FR/IT
- Audit log of every punch (`audit_log` row): who, when, what, geofence boolean, IP, user agent
- Retention: 5 years per Swiss labour-records requirement (ArGV 1 Art. 73)
- Owner cannot delete punches; can only "annotate" with a note. Original punch immutable.
- See `domains/labour-law.md` for the full compliance map

## Geofence math

```ts
function isWithinFence(lat, lng, fence) {
  const earthR = 6_371_000;
  const dLat = toRad(lat - fence.lat);
  const dLng = toRad(lng - fence.lng);
  const a = Math.sin(dLat/2)**2
    + Math.cos(toRad(fence.lat)) * Math.cos(toRad(lat)) * Math.sin(dLng/2)**2;
  const distance = earthR * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
  return distance <= fence.radius_meters;
}
```

GPS accuracy 10–50m typical; default radius 100m. Owner can set radius per stable (50–500m). Mobile browser may take 2–5s for first fix.

## Acceptance criteria

- [ ] Worker on-site clocks in within 5s; geo_within_fence = true; live timer starts
- [ ] Worker at home tries to clock in; geo_within_fence = false; badge renders in `--rust-600`; punch recorded; owner notified
- [ ] Owner approves off-site punch → `approved_by_user_id` set; punch counts toward hours
- [ ] Break time excluded from total hours
- [ ] Auto-clock-out at 23:59 if no manual punch; owner + worker notified
- [ ] CSV export for last month: every clocked-in shift represented; total hours match sum
- [ ] Geofence permission denied: punch recorded with `geo_within_fence = null`; owner gets approval queue
- [ ] No raw coordinate ever appears in DB or logs (audit-tested)

## Acceptance integration test

`apps/worker/tests/integration/timeclock.test.ts`

```ts
test('off-site punch is recorded with flag, never with coords', async () => {
  const { ctx } = await seed.worker();
  await clockIn(ctx, { latitude: 47.0, longitude: 8.0 }); // far from stable
  const p = await getLatestPunch(ctx);
  expect(p.geo_within_fence).toBe(false);
  expect(JSON.stringify(p)).not.toMatch(/47\.0/);
  const audit = await getAuditLog({ entity: 'time_punches', entity_id: p.id });
  expect(JSON.stringify(audit)).not.toMatch(/47\.0/);
});
```

## Out of scope

- Multi-stable workers (V2)
- Overtime auto-calculation per CH labour law (V1.1)
- Direct payroll software integration (V1.1+ — CSV bridge sufficient)
- Photo selfie verification (V2)
