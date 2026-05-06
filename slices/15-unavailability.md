# Slice 15 — Sick, Unavailability, Schedule Sync

**Phase:** Worker · **Estimate:** 5 dev-days · **Owner:** Sharad backend, freelancer frontend

The worker-side schedule. Workers see their next 7–14 days. Can request sick / unavailable. Owner approves before the schedule actually updates. iCal feed exports for personal calendar sync.

## Goal

Marco taps "report sick" Saturday morning. Owner gets notified. Owner approves → Marco's Saturday shift is removed, owner sees a gap to fill. iCal feed lets Marco subscribe in his iPhone calendar so he sees shifts alongside personal events.

## What ships

### Worker side

1. **Schedule view** at `/schedule`. List of upcoming shifts: today + 7 days. Tap a shift → detail (start, end, expected tasks, notes).
2. **Week strip** on Home (worker density per `DESIGN.md`). Visual pill-grid: Mon–Sun, current day highlighted with `--terracotta-600` underline, off-days rendered with `--ink-300` dashed border (no fill), available days `--canvas` over `--paper`.
3. **Report sick** quick action. Modal: dates affected (default today), reason (sick/other), optional note. Submit → owner notified.
4. **Request unavailable** for a future date. Same modal as sick but for vacation / personal.
5. **Status of requests.** Worker sees pending / approved / declined for each request.
6. **iCal feed.** Per-worker secret URL `https://crew.lafattoria.app/api/ical/{token}`. Subscribe in iOS/Google Calendar. Updates on shift changes within 24h (subscriber refresh interval).

### Owner side

7. **Approval queue** at `/workers/approvals`. List of pending sick + unavailability requests. Approve / decline. On approve: shift status updated, schedule view reflects gap, optional auto-broadcast to other workers ("shift open Saturday").
8. **Schedule editor** at `/schedule/shifts`. Owner creates shifts manually for now (slice 15 doesn't ship auto-scheduler — that's V1.1). Drag-drop on a calendar.

## Schema diff

```sql
-- 014_shifts_unavail.sql
create table shifts (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  worker_user_id uuid not null references auth.users(id),
  starts_at timestamptz not null,
  ends_at timestamptz not null,
  status text not null default 'scheduled' check (status in (
    'scheduled','clocked_in','completed','sick','no_show','cancelled'
  )),
  break_minutes int default 30,
  notes text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create index on shifts (stable_id, worker_user_id, starts_at);

create table unavailability (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  worker_user_id uuid not null references auth.users(id),
  starts_on date not null,
  ends_on date not null,
  reason text not null check (reason in ('sick','vacation','personal','other')),
  notes text,
  status text not null default 'pending' check (status in ('pending','approved','declined')),
  decided_by_user_id uuid references auth.users(id),
  decided_at timestamptz,
  created_at timestamptz not null default now()
);

create index on unavailability (stable_id, worker_user_id, starts_on);

create table ical_tokens (
  user_id uuid primary key references auth.users(id),
  stable_id uuid not null references stables(id),
  token text not null unique,                  -- random 32-byte base64url
  revoked_at timestamptz,
  created_at timestamptz not null default now()
);
```

## Approval flow

When worker submits unavailability:
1. Insert row with `status='pending'`
2. Notify all owner/manager users in stable (in-app + email)
3. Owner approves: `status='approved'`, decided_by, decided_at, then update overlapping shifts to `status='sick'` or `cancelled`
4. Owner declines: `status='declined'`, no shift change, worker notified

Owner approval is required per locked decision — no auto-apply, no cool-down logic.

## iCal feed

- Token-based URL (no auth header — calendar subscribers don't send them)
- Token revocable via `/settings/calendar-feed`
- Returns RFC 5545 compliant `.ics` with VEVENTS for each shift
- Updates: subscribers refresh every 24h (we can't push)

## Acceptance criteria

- [ ] Worker reports sick today; owner sees pending request within 5s
- [ ] Owner approves → today's shift status becomes 'sick'; schedule view updates
- [ ] Owner declines → shift unchanged, worker notified
- [ ] Future unavailability (vacation 2 weeks out) → approval doesn't change current schedule until that date
- [ ] iCal subscription URL works in iOS Calendar (manual test)
- [ ] Shift cancellation propagates to iCal within 24h subscriber refresh
- [ ] Token revocation breaks the feed
- [ ] Approving sick when no shift exists for that day → just records the unavailability, no error

## Acceptance integration test

`apps/worker/tests/integration/unavailability.test.ts`

```ts
test('owner approval converts shift to sick status', async () => {
  const { ownerCtx, workerCtx, shift } = await seed.workerWithShiftToday();
  await reportSick(workerCtx, { date: shift.starts_at });
  const pending = await getPendingApprovals(ownerCtx);
  expect(pending).toHaveLength(1);
  await approveUnavailability(ownerCtx, { id: pending[0].id });
  const updated = await getShift(ownerCtx, shift.id);
  expect(updated.status).toBe('sick');
});
```

## Out of scope

- Auto-scheduler / shift-template generator (V1.1)
- Shift-swap requests between workers (V2)
- Skill-matching auto-assignment (V2)
- SMS notifications (V2)
- ML staffing forecast (V1.5)
