# Slice 03 — Schedule

**Phase:** Operational core · **Estimate:** 6 dev-days · **Owner:** Sharad backend, freelancer frontend

The calendar that ties everything together. Lessons, vet visits, farrier, transport, competitions, turnout, clinics, social events. Recurrence support. Visibility levels (private / public_stable / public_open).

## Goal

Owner creates events and attaches horses + people. Clients see events involving their horses + stable-public events. Workers see all operational events. Recurrence handled correctly across DST transitions.

## What ships

### Owner side

1. **Calendar view** at `/schedule`. Month / week / day views. Color-coded by event kind. Click a day → drawer with that day's events.
2. **Event create/edit modal**. Fields: kind, title, starts_at, ends_at, all-day toggle, recurrence (RRULE picker), horses (multi-select), people (multi-select with role), location, visibility, notes.

   **Anmeldung aktivieren section** (collapsed by default, applies to `clinic`/`competition`/`lesson`/`training`/`social` kinds). When toggled on:
   - **Service** from catalog (slice 05) — links the booking to a billable service ("Springkurs CHF 380").
   - **Capacity** (1 = exclusive, N = group). Default 1.
   - **Sign-up deadline** (`signup_closes_at`), default = `starts_at − 24h`.
   - **Visibility** auto-set to `public_stable` when toggle activates (clients must see it to sign up); owner can override to `public_open`.
   On save, a **single transaction** inserts the `events` row AND a linked `availability_slots` row (slice 11) with `event_id` set. Full bridge spec — atomic create flow, cascade rules on event edit/cancel/delete, `SignupClosed` deadline enforcement, visibility coupling — in `domains/event-signup.md`.
3. **Recurrence editing UI.** When editing a recurring event: "this occurrence only" / "this and future" / "the entire series." RFC 5545 RRULE compliant.
4. **Today widget** (used by dashboard slice 16).

### Client side

5. **Calendar view** at `/calendar`. Same month/week/day, but shows: events involving my horses (`--terracotta-600`) + stable-public events (`--ink-500`) + bookable slots (`--moss-600` dashed). Calendar event colors locked in `DESIGN.md` — no fourth color.
6. **Event detail drawer** read-only.

### Worker side

7. **Schedule view** at `/schedule`. Mobile-first. Shows all operational events the worker is involved in + stable-wide ops. Worker density and tap-targets per `DESIGN.md` (44px minimum, single-column).

## Schema diff

```sql
-- 004_schedule.sql
create table events (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  kind text not null check (kind in (
    'lesson','training','vet','farrier','transport',
    'competition','turnout','clinic','social','other'
  )),
  title text not null,
  starts_at timestamptz not null,
  ends_at timestamptz not null,
  all_day boolean not null default false,
  recurrence_rule text,                      -- RFC 5545 RRULE
  recurrence_parent_id uuid references events(id),
  visibility text not null default 'private' check (visibility in ('private','public_stable','public_open')),
  location text,
  notes text,
  signup_enabled boolean not null default false,    -- "Anmeldung aktivieren" toggle
  signup_service_id uuid references services(id),   -- non-null when signup_enabled
  signup_capacity int,                              -- non-null when signup_enabled
  signup_closes_at timestamptz,                     -- non-null when signup_enabled; default = starts_at - 24h
  cancelled_at timestamptz,
  created_by_user_id uuid not null references auth.users(id),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table event_horses (
  event_id uuid not null references events(id) on delete cascade,
  horse_id uuid not null references horses(id),
  stable_id uuid not null,                   -- denormalized for RLS
  primary key (event_id, horse_id)
);

create table event_people (
  event_id uuid not null references events(id) on delete cascade,
  person_id uuid not null references people(id),
  stable_id uuid not null,
  role text not null check (role in ('rider','vet','farrier','trainer','attendee','organizer')),
  primary key (event_id, person_id, role)
);

create index on events (stable_id, starts_at);
create index on events using gist (tstzrange(starts_at, ends_at));
```

## RLS

- `events`, `event_horses`, `event_people`: scoped by `stable_id`
- Clients see events where: visibility = `public_stable` OR `public_open` OR (one of `event_horses.horse_id` is owned by them)
- Workers see all events (operational visibility)
- See `domains/rls.md` for the join-table isolation pattern

## Tech notes

- **Use `rrule.js`** on both client and server for expansion. Single canonical library, no Python.
- **All times stored UTC.** RRULEs persist `DTSTART;TZID=Europe/Zurich:...` so wall-clock semantics survive DST. Render in `stables.default_locale` zone (CH = Europe/Zurich).
- **DST handling**: events expanded against the stored TZID, not raw UTC, so a 17:00 weekly lesson stays 17:00 wall-clock across spring-forward (CET→CEST) and fall-back (CEST→CET). The 02:00–03:00 spring-forward gap is the trap — events scheduled inside the gap are lifted to 03:00 by `rrule.js` per RFC 5545 §3.3.5; we accept that behavior and assert it in tests.
- **Recurrence expansion** server-side limited to a 12-month window per query to avoid runaway computation.
- **Cancellation**: "this occurrence" creates an EXDATE in the parent's RRULE; "this and future" splits into two events; "entire series" sets `cancelled_at` on parent.
- **Test fixtures** in `apps/web/tests/fixtures/recurrence/` cover: spring-forward gap (2026-03-29 02:30 Zurich), fall-back ambiguity (2026-10-25 02:30), leap-day yearly recurrence, weekly series spanning both DST transitions in one year.

## Acceptance criteria

- [ ] Owner creates a weekly recurring lesson; calendar shows correct occurrences for next 6 months
- [ ] Editing one occurrence does not affect siblings; "all future" splits correctly
- [ ] **DST spring-forward**: weekly 17:00 lesson with `DTSTART;TZID=Europe/Zurich:20260301T170000;RRULE:FREQ=WEEKLY` produces no duplicates and no skips across 2026-03-29; April occurrences render 17:00 CEST
- [ ] **DST fall-back**: weekly 02:30 event spanning 2026-10-25 produces exactly one occurrence on the transition day, not two
- [ ] **Spring-forward gap edge case**: `DTSTART;TZID=Europe/Zurich:20260329T023000` (inside the non-existent hour) materializes at 03:30 local on 2026-03-29 per RFC 5545 §3.3.5, not 02:30
- [ ] **Leap-day yearly**: `RRULE:FREQ=YEARLY;BYMONTH=2;BYMONTHDAY=29` from 2024-02-29 expands to 2024, 2028, 2032 only (skips non-leap years), no crash
- [ ] Client viewing calendar sees their own horse's vet visit (`--terracotta-600`) and the stable's clinic (`--ink-500`); bookable open slots render as `--moss-600` dashed; no fourth hue appears
- [ ] Client cannot see a private vet event for another client's horse
- [ ] Worker sees all operational events on mobile schedule view
- [ ] Bookable slots (slice 11) integrate as `visibility='public_open'`

## Acceptance integration test

`apps/client/tests/integration/calendar-isolation.test.ts`

```ts
test('client A cannot see private events of client B horses', async () => {
  const { clientACtx, clientBPrivateEvent } = await seed.twoClientsWithEvents();
  const events = await getCalendar(clientACtx, { from, to });
  expect(events.find(e => e.id === clientBPrivateEvent.id)).toBeUndefined();
});
```

## Out of scope

- iCal export feed (slice 13 for workers, V1.1 for clients)
- Conflict detection beyond simple overlap warnings (V1.1)
- External calendar sync inbound (V2)
