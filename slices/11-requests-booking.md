# Slice 11 — Service Requests and Bookable Slots

**Phase:** Comms + booking · **Estimate:** 6 dev-days · **Owner:** Sharad backend, freelancer frontend

The two-sided booking system. Owner exposes bookable time slots (lessons, farrier visits, transport). Clients tap to request. Owner manually confirms, declines, or reschedules. Race conditions handled by hold-on-request.

## Goal

Replace the WhatsApp "can I have a lesson Saturday?" workflow. Clients see open slots in their calendar (slice 03 layer), tap to book. Request lands in owner inbox. Owner one-click confirms; the slot is held during request and released on decline.

## What ships

### Owner side

1. **Slot creation** at `/schedule/slots`. Owner defines: service (catalog link), recurrence (e.g., every Saturday 10:00–11:00), capacity (1 = exclusive lesson, N = group), valid from/to.
2. **Slot list** with status counts: open / requested / booked.
3. **Request inbox** at `/requests`. Pending requests sorted by request time. Each request: client, requested service, slot or free-form, message. Three actions: confirm, decline, reschedule.
4. **Confirm flow**: confirms request → creates an `event` (slice 03) tied to the slot/horse → notifies client → slot capacity decrements (or fully booked).
5. **Decline flow**: sets status, sends notification with optional reason, releases held capacity.
6. **Reschedule flow**: opens a "propose new time" modal; sends back to client; client accepts or declines counter-offer.

### Client side

7. **Bookable slots** appear in calendar (slice 03 layer) rendered as `--moss-600` dashed per `DESIGN.md` calendar palette.
8. **Request modal** on tap. Pick horse (auto-filled if 1 horse), add note, submit. Capacity decrements optimistically (hold-on-request).
9. **My requests** at `/requests`. Pending / confirmed / declined / scheduled. Status changes pushed in realtime.
10. **Free-form request**. Quick actions on dashboard (book lesson / request transport / book farrier / message): if no slot fits, free-form request creates a `service_request` without a `slot_id`; owner schedules it manually.

## Schema diff

```sql
-- 011_requests_slots.sql
create table availability_slots (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  service_id uuid not null references services(id),
  starts_at timestamptz not null,
  ends_at timestamptz not null,
  capacity int not null default 1,
  capacity_used int not null default 0,
  capacity_held int not null default 0,
  status text not null default 'open' check (status in ('open','full','cancelled')),
  recurrence_parent_id uuid references availability_slots(id),
  notes text,
  created_at timestamptz not null default now()
);

create index on availability_slots (stable_id, starts_at);

create table service_requests (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  person_id uuid not null references people(id),
  horse_id uuid references horses(id),
  slot_id uuid references availability_slots(id),
  kind text not null check (kind in ('lesson','farrier','vet','transport','clinic_audit','other')),
  description text,
  requested_for timestamptz,
  status text not null default 'pending' check (status in ('pending','confirmed','declined','scheduled','cancelled')),
  decided_by_user_id uuid references auth.users(id),
  decided_at timestamptz,
  decision_notes text,
  event_id uuid references events(id),         -- set on confirm
  created_at timestamptz not null default now()
);

create index on service_requests (stable_id, status, created_at);
```

## Race condition handling — hold-on-request

When a client creates a request against a slot:

```sql
begin;
select capacity, capacity_used, capacity_held from availability_slots
  where id = $1 for update;
-- check capacity_used + capacity_held < capacity
update availability_slots set capacity_held = capacity_held + 1 where id = $1;
insert into service_requests ...;
commit;
```

On confirm: `capacity_held -= 1; capacity_used += 1`. On decline: `capacity_held -= 1`. On expiry (24h pending without action): cron releases hold and sets request status to `cancelled` with note "expired".

## Acceptance criteria

- [ ] Owner creates a recurring weekly lesson slot; calendar shows it for next 12 weeks
- [ ] Two clients tap the same slot in the same second; one succeeds, one gets clear "no longer available"
- [ ] Confirming a request creates a calendar event linked to the slot
- [ ] Declining releases capacity; another client can immediately re-request
- [ ] 24h-stale pending request auto-cancels and releases hold
- [ ] Free-form request without slot still works; owner can convert into scheduled event
- [ ] Client gets realtime notification on confirm/decline
- [ ] Owner inbox count badge decrements on action

## Acceptance integration test

`apps/client/tests/integration/booking.test.ts`

```ts
test('hold-on-request prevents double booking under contention', async () => {
  const slot = await seed.slot({ capacity: 1 });
  const [r1, r2] = await Promise.all([
    requestSlot(clientA, slot.id),
    requestSlot(clientB, slot.id),
  ]);
  const successes = [r1, r2].filter(r => r.success);
  expect(successes).toHaveLength(1);
  expect([r1, r2].find(r => !r.success).reason).toBe('slot_full');
});
```

## Out of scope

- Auto-confirm slots (V1.1 — explicitly deferred per locked decision)
- Cancellation policies (V1.1)
- Waitlist if slot is full (V1.1)
- Payment-on-booking (V2)
