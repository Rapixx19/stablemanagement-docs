# Domain: Event Sign-up (slice 03 + 11 bridge)

How owner events become bookable. The atomic composition between `events` (slice 03) and `availability_slots` (slice 11). Added 2026-05-13; see `DECISIONS.md` change log.

## The two sources of slots

`availability_slots` rows are created two ways:

1. **Standalone** — owner creates at `/owner/schedule/slots` for recurring lessons, farrier days, transport windows. `event_id IS NULL`. Owner manages the slot directly.
2. **Event-linked** — owner ticks "Anmeldung aktivieren" on an event in slice 03's create/edit modal. `event_id` populated. The event is the primary record; the slot is the booking interface attached to it.

Don't conflate them in the UI. Standalone slots have no event context — they exist purely as booking offers. Event-linked slots inherit time, capacity, and service from the event and stay in sync via cascade triggers.

## Schema (the bridge columns)

On `events` (slice 03):

```sql
signup_enabled boolean not null default false,
signup_service_id uuid references services(id),     -- non-null when signup_enabled
signup_capacity int,                                -- non-null when signup_enabled
signup_closes_at timestamptz,                       -- non-null when signup_enabled; default = starts_at - 24h
```

On `availability_slots` (slice 11):

```sql
event_id uuid references events(id),                -- non-null only for event-linked slots
```

`signup_*` columns are NULL when the toggle is off; check constraint enforces "all four populated together or all null."

## Create flow (owner side)

When owner saves an event with `signup_enabled=true`:

```sql
BEGIN;
  INSERT INTO events (..., signup_enabled, signup_service_id, signup_capacity, signup_closes_at)
    VALUES (...) RETURNING id;
  INSERT INTO availability_slots (
    stable_id, service_id, event_id, starts_at, ends_at,
    capacity, capacity_used, capacity_held, status
  ) VALUES (
    $stable_id, $signup_service_id, $event_id, $event_starts_at, $event_ends_at,
    $signup_capacity, 0, 0, 'open'
  );
COMMIT;
```

One transaction, both rows or neither. If insert fails, owner sees a single error toast; no orphan event without slot or vice versa.

## Sign-up request flow (client side)

Client sees the event on `/client/calendar` with an "Anmelden" CTA — rendered next to the date/title when `signup_enabled=true` AND `now() < signup_closes_at` AND `capacity_used + capacity_held < capacity`. On tap, the standard slice 11 request modal opens.

On owner confirm (single transaction):

1. `service_requests.status = 'confirmed'`, `event_id` set to the parent event
2. `availability_slots.capacity_used += 1`, `capacity_held -= 1`
3. Insert into `event_horses` (slice 03 join) linking the requesting client's `horse_id` to the event
4. Insert into `event_people` linking the `person_id` with `role = 'attendee'`

**No second event is created.** All confirmed sign-ups share the parent event. The event's `event_horses` + `event_people` grow as people sign up. Owner's calendar view of the event shows the attendee list inline.

## Cascade rules

Owner mutations on the parent event propagate to the linked slot in the same TX:

| Owner action | Slot side effect | Request side effect |
|---|---|---|
| Edit `starts_at` / `ends_at` | Mirror update on slot | Notify confirmed attendees of time change |
| Edit `signup_capacity` up | `capacity = new_value` | Opens additional spots |
| Edit `signup_capacity` below `capacity_used` | **Rejected** | Owner must decline excess requests first |
| Cancel event | `status='cancelled'` | Pending requests auto-decline "Anlass abgesagt"; confirmed attendees notified |
| Delete event | Forbidden if confirmed sign-ups exist; otherwise cancel-then-archive | n/a |
| Toggle `signup_enabled` off | Linked slot deleted IF no confirmed sign-ups; else rejected | n/a |

Cascades implemented via `BEFORE UPDATE` triggers on `events` checking `event_id IS NOT NULL` on the linked slot.

## Sign-up deadline (`SignupClosed`)

Enforced at **two layers**:

1. **Server action** — request INSERT rejected with `SignupClosed` rescue path if `now() >= signup_closes_at`, even if the slot still has capacity. Surfaces inline as "Anmeldeschluss erreicht."
2. **Deadline cron** — when `signup_closes_at` passes, the hourly `/api/cron/expire-pending-requests` (slice 11) flips `availability_slots.status` to `full` so stale client caches can't slip a request through.

UX: client tapping a deadline-passed event sees the "Anmelden" CTA disabled with tooltip "Anmeldeschluss erreicht."

## Visibility coupling

When `signup_enabled=true`, the event's `visibility` MUST be `public_stable` or `public_open` — `private` would hide it from clients who need to sign up. Slice 03 form auto-sets to `public_stable` when the toggle activates; owner can override to `public_open` for non-clients (e.g., an open clinic advertised on social channels).

Reverse: setting `visibility='private'` on a sign-up-enabled event is rejected at the server-action layer with copy: "Anmeldung kann nicht aktiviert sein, wenn die Sichtbarkeit auf 'Privat' gesetzt ist."

## Worker visibility

Workers see event-linked sign-ups as regular operational events on `/worker/schedule` (slice 03 worker side). Same color treatment (terracotta for own horse, ink-500 for stable-wide). Sign-up state (capacity, deadline) is not relevant to workers — they just need to know it's happening so they can prepare the arena, hay, etc.

## Common mistakes

- **Forgetting the cascade trigger** → owner edits event time, slot stays stale, clients book the wrong time
- **Allowing a second event on confirm** → calendar fills with duplicate events, attendee join tables fragmented
- **Skipping the visibility check** → owner toggles sign-up on a private event; clients never see it, requests appear from nowhere
- **Treating standalone and event-linked slots as the same in the UI** → owner confused about which screen manages which
- **Letting `signup_capacity` drop below `capacity_used`** → silently dropped already-confirmed attendees

## When this doc updates

- A new event kind needs different sign-up semantics (e.g., paid-on-booking in V2) → add a section
- The cascade rule table changes → update the table
- Worker visibility differs by event kind → add a row to the worker section
- Auto-confirm slots ship (V1.1) → document the path that skips owner confirmation
