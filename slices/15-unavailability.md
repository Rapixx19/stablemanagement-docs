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
9. **Operations calendar** at `/owner/ops` — the internal counterpart to `/owner/schedule`. **Never visible to clients** (route guard + RLS). Renders:
   - **Worker rows** (one per active worker user), days across (week or month view).
   - **Shift bars** showing scheduled `shifts` per worker per day, color-coded by status: `scheduled` (ink-300), `clocked_in` (moss-600), `completed` (ink-500), `sick`/`cancelled` (rust outline).
   - **Task pills** (slice 04) inside each shift bar showing tasks assigned to that worker that day. Drag a pill from one worker's row to another to reassign (`tasks.assigned_to_user_id` updated atomically, both rows re-render via Realtime).
   - **Owner row** at the top showing owner-self-assigned tasks (`tasks.assigned_to_user_id = owner_user_id`) and owner-private events (slice 03 `visibility='private'`).
   - **Coverage gaps** highlighted: business-hours (07:00–18:00) bands with zero scheduled shifts render in `--ochre-200` with the label "Keine Abdeckung."
   - **Arrival vs. expected**: each shift bar shows the actual `time_punches` overlay (slice 14) — green tick if clocked in within 15 min of `shifts.starts_at`, ochre pill "13 min spät" if late, rust if no-show by 30 min past start.
   - **Quick actions**: drag-drop creates/edits a shift, click-empty-cell opens task-add, right-click for assign-to-worker menu.
   Performance: fetches a 14-day window (1 week back, 1 week forward) via `Promise.all` parallel fetch on shifts + tasks + private events + punches. Budget: <1s p95 (same as slice 16 dashboard tiles).

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
  version int not null default 0,              -- optimistic locking (E-CQ-2)
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
  version int not null default 0,              -- optimistic locking (E-CQ-2)
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

## RLS

Three tables scoped by `stable_id` per `domains/rls.md`.

- `shifts`:
  - `owner`: full access.
  - `worker`: SELECT for `worker_user_id = auth.uid()` only. **No INSERT/UPDATE/DELETE** — shift edits go through owner; sick conversion flows via unavailability approval.
  - `client`: **no access**.
- `unavailability`:
  - `worker`: **INSERT** with `WITH CHECK (worker_user_id = auth.uid() AND status = 'pending')`. SELECT own rows. **No UPDATE, no DELETE** post-submit — worker can't retroactively change a declined to approved or revoke after owner approves. Amend = submit a new row.
  - `owner`: SELECT all; UPDATE limited to `status`, `decided_by_user_id`, `decided_at`. No DELETE for any role.
  - `client`: **no access**.
- `ical_tokens`: worker SELECT/INSERT/UPDATE (revoke) own row only. Owner has no access — the token is the worker's secret.

**Column-level constraint** — `BEFORE UPDATE` trigger on `unavailability` blocks any column change outside `{status, decided_by_user_id, decided_at}` for owner and blocks all UPDATE for worker. Server-action zod schema mirrors.

## Approval flow

When worker submits unavailability:
1. Insert row with `status='pending'`
2. Notify all owner users in stable (in-app + email)
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
- [ ] **`/owner/ops` renders** the 14-day window with worker rows + shift bars + task pills + coverage gaps in <1s p95
- [ ] **Ops calendar route guard**: client/worker context GET on `/owner/ops` returns 403; only `owner` role can access
- [ ] **Drag-drop task reassignment**: pill dragged from Marco's row to Sara's row atomically updates `tasks.assigned_to_user_id`, both rows re-render via Realtime within 2s
- [ ] **Coverage gap detection**: weekday 07:00–18:00 with zero scheduled shifts → `--ochre-200` band with "Keine Abdeckung" label
- [ ] **Arrival overlay accuracy**: worker clocks in 13 min after shift start → shift bar shows "13 min spät" ochre pill; 30 min past with no clock-in → rust "no-show" status
- [ ] **Worker INSERT impersonation**: INSERT with `worker_user_id != auth.uid()` rejected by WITH CHECK
- [ ] **Worker self-approval guard**: INSERT with `status='approved'` rejected by WITH CHECK (must be `pending`)
- [ ] **Worker UPDATE post-submit** on own unavailability rejected by trigger (amend via new row)
- [ ] **Owner UPDATE on `starts_on`** of unavailability rejected by trigger (whitelist enforcement)
- [ ] **Worker SELECT on another worker's `ical_tokens`** rejected by RLS

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
