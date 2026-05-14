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
10. **Free-form request**. Quick actions on dashboard (book lesson / request transport / book farrier / message): if no slot fits, free-form request creates a `service_request` without a `slot_id`; owner schedules it manually. For free-form requests, client picks `kind` from the stable's catalog categories (`services.category` distinct values) or types a custom label.
11. **Bookable services catalog** at `/client/services` — discovery surface separate from the calendar. Lists every service where `services.bookable_by_client = true`, grouped by category (Riding · Care · Transport · Veterinary · Farrier · Other). Each card: `client_facing_name` (or fallback `name`), price, `client_description`, default provider avatar if set, and the **next 3 available dates** computed from the future `availability_slots` for that service. Empty state if no service is currently bookable (covers stables not yet exposing anything): "Frage dein Stallpersonal — wir nehmen aktuell keine direkten Buchungen an."
12. **Per-service booking page** at `/client/services/[id]` — landing for a single service. Service description + price at top. **Day picker** (week-strip, 7 days at a time, navigable forward up to `max_advance_days`) showing days that have at least one open slot in `--moss-600`, days with no slots in `--ink-300` dashed. Pick a day → time-slot list below (capacity shown per slot). Tap a slot → existing slice 11 request modal. Enforces `min_notice_hours` — slots inside the notice window render as `--ochre-200` with "Nicht mehr buchbar".

## `kind` field — source of truth

The `service_requests.kind` column is **free-text**, not enum-constrained. The catalog (slice 05) is the source of truth for what's bookable:

- **Slot-backed request** — `kind` auto-populated from `services.category` of the slot's linked service. Owner adds "Solarium" to catalog with `category='care'` → slot bookings inherit `kind='care'`. Owner sees the actual service name (`services.name`) in the inbox, not just the category.
- **Free-form request** — client picks from existing catalog categories or types a label. The owner's inbox filter chips are dynamic — generated from `SELECT DISTINCT kind FROM service_requests WHERE stable_id = $1`, so new categories appear in the filters automatically once a request uses them.

This means **any service in the catalog is bookable** — there's no enum limit. Pilot can add Solarium, Longieren, Fotoshooting, Gastpferd-Box, Foaling-Begleitung, anything. The owner's inbox filter chips evolve with the stable's offering.

## Schema diff

```sql
-- 011_requests_slots.sql
create table availability_slots (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  service_id uuid not null references services(id),
  event_id uuid references events(id),         -- non-null for event-linked slots (slice 03 sign-up flow)
  starts_at timestamptz not null,
  ends_at timestamptz not null,
  capacity int not null default 1,
  capacity_used int not null default 0,
  capacity_held int not null default 0,
  status text not null default 'open' check (status in ('open','full','cancelled')),
  recurrence_parent_id uuid references availability_slots(id),
  notes text,
  version int not null default 0,             -- optimistic locking (E-CQ-2)
  created_at timestamptz not null default now()
);

create index on availability_slots (stable_id, starts_at);

create table service_requests (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  person_id uuid not null references people(id),
  horse_id uuid references horses(id),
  slot_id uuid references availability_slots(id),
  kind text not null,                            -- free text, populated from services.category when slot is involved
  description text,
  requested_for timestamptz,
  status text not null default 'pending' check (status in ('pending','confirmed','declined','scheduled','cancelled')),
  decided_by_user_id uuid references auth.users(id),
  decided_at timestamptz,
  decision_notes text,
  event_id uuid references events(id),         -- set on confirm
  version int not null default 0,              -- optimistic locking (E-CQ-2)
  created_at timestamptz not null default now()
);

create index on service_requests (stable_id, status, created_at);
```

## RLS

Both tables scoped by `stable_id` per `domains/rls.md`.

- `availability_slots`:
  - `owner`: full access (CRUD slots, edit capacity, cancel).
  - `worker`: SELECT only (slot reads in the worker calendar surface).
  - `client`: SELECT slots in their stable with `status IN ('open','full')`. **No INSERT/UPDATE/DELETE** — clients never touch slot state directly; hold/confirm/decline mutate via server actions.
- `service_requests`:
  - `client`: **INSERT** with `WITH CHECK (person_id = (people row WHERE user_id = auth.uid()) AND status = 'pending')`. SELECT own rows only. UPDATE limited to `description` (pre-decision) and `status='cancelled'` (own withdrawal pre-decision). **No DELETE.**
  - `owner`: SELECT all in stable. UPDATE limited to `{status, decided_by_user_id, decided_at, decision_notes, event_id, version}`. No DELETE for any role — declined/cancelled rows stay for audit.
  - `worker`: **no access** in V1.

**Column-level constraint** — `BEFORE UPDATE` trigger on `service_requests` enforces the role-specific whitelists above. Server-action zod schemas mirror.

**Cross-tenant slot guard** — INSERT on `service_requests` must validate `availability_slots.stable_id = service_requests.stable_id` at the server-action layer before the TX opens. RLS alone doesn't catch a client crafting a request with a `slot_id` from another stable they observed via a leaked UUID.

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

On confirm: `capacity_held -= 1; capacity_used += 1`. On decline: `capacity_held -= 1`.

**Confirm atomicity** — the confirm flow is a single transaction: (a) INSERT into `events` (slice 03), (b) UPDATE `service_requests` setting `status='confirmed'`, `event_id`, `decided_by_user_id`, `decided_at`, bump `version`, (c) UPDATE `availability_slots` capacity counters + re-evaluate `status`. Mid-TX failure → 0 changes committed; request stays `pending`, no orphan event, no capacity drift.

## Event-linked slots

Slots come from two sources: **standalone** (owner creates directly) and **event-linked** (created atomically from slice 03 when owner ticks "Anmeldung aktivieren"). Full bridge — schema, atomic create flow, cascade rules, `SignupClosed` deadline enforcement, visibility coupling, worker visibility — is in `domains/event-signup.md`. Slice 11 owns the request/booking side; that doc owns the composition.

## Stale-request expiry (24h cron)

**Vercel Cron** at `/api/cron/expire-pending-requests`, hourly (`0 * * * *` Europe/Zurich, authenticated by `CRON_SECRET` per `DECISIONS.md` D12). For every `service_requests` row where `status='pending'` AND `created_at < now() - interval '24 hours'`:

1. UPDATE row: `status='cancelled'`, `decision_notes='expired'`, `decided_at=now()`, `decided_by_user_id=null` (system actor; `audit_log.actor_kind='system:request-expiry'` per `eng-review.md` E-CQ-5), bump `version`.
2. If `slot_id IS NOT NULL`: decrement `availability_slots.capacity_held` by 1, re-evaluate slot `status` (flip `full` → `open` when `capacity_used + capacity_held < capacity`).
3. Notify the requesting client in-app + email "Anfrage abgelaufen".

Idempotent — the WHERE filter excludes already-cancelled rows, so re-runs produce zero new updates. The 24h window is hardcoded in V1; per-stable configurability is V1.1.

## UX reference

**Info Hierarchy — Request inbox (owner)** at `/requests`:
1. **Unread/pending count** in left rail + page header — `lg 20/28`, terracotta dot when >0
2. **Filter chips**: pending / today / this week / overdue (>18h pending)
3. **Request row** — client + service + slot/free-form time + relative timestamp + **age pill** (ochre at 18h, rust at 24h)
4. **Three actions per row**: Confirm (primary terracotta) / Decline (ghost) / Reschedule (ghost)

**Info Hierarchy — Request modal (client)**:
1. **Service + time** — `lg 20/28`, top
2. **Horse picker** — auto-filled if client has 1 horse, otherwise required dropdown
3. **Note** — optional textarea, max 500 chars
4. **Submit** — primary CTA, disabled until horse selected

**State matrix** — loading / empty (first-run + filtered) / error / partial per surface in DE/FR/IT lives in `docs/domains/copy-library.md` (CEO sweep DES-1). V1 surfaces: owner slot list, owner request inbox, client request modal, client my-requests list.

## Acceptance criteria

- [ ] Owner creates a recurring weekly lesson slot; calendar shows it for next 12 weeks
- [ ] Two clients tap the same slot in the same second; one succeeds, one gets clear "no longer available"
- [ ] Confirming a request creates a calendar event linked to the slot
- [ ] Declining releases capacity; another client can immediately re-request
- [ ] 24h-stale pending request auto-cancels and releases hold
- [ ] Free-form request without slot still works; owner can convert into scheduled event
- [ ] **Arbitrary catalog service is bookable**: owner adds "Solarium" to catalog with `category='care'`, creates an `availability_slots` row pointing to it, client taps it on calendar, request appears in owner inbox with `kind='care'` and service name "Solarium" surfaced; filter chip "Pflege/Care" appears in inbox if not already present
- [ ] **Dynamic filter chips**: inbox filter chips computed from `SELECT DISTINCT kind` over current pending requests; new chip appears within 2s of first request of that kind via Realtime
- [ ] Client gets realtime notification on confirm/decline
- [ ] Owner inbox count badge decrements on action
- [ ] **Optimistic locking on `service_requests`**: concurrent confirm + decline race → second action gets `OptimisticLockError`, UI re-fetches; no half-set state where `status='confirmed'` but `event_id IS NULL`
- [ ] **Confirm atomicity**: simulated failure mid-TX (event INSERT + request UPDATE + capacity counters) → 0 changes committed, request stays `pending`, no orphan event, no capacity drift
- [ ] **Cross-tenant slot guard**: client A in stable X POSTs a request with `slot_id` from stable Y → server-action returns `403`, no row written, audit_log captures the attempt
- [ ] **Worker SELECT/INSERT denied** on `service_requests` by RLS
- [ ] **Client INSERT impersonation**: INSERT with `person_id` not matching `(people WHERE user_id = auth.uid())` rejected by WITH CHECK
- [ ] **Client self-approval guard**: INSERT with `status != 'pending'` rejected by WITH CHECK
- [ ] **24h expiry cron idempotent + capacity release**: cron runs twice → second run no-ops; held capacity released and slot status flips `full` → `open` when no other holds/used
- [ ] **Schedule drift test**: Vercel Cron schedule pinned to `0 * * * *` Europe/Zurich in `vercel.ts`; CI snapshot matches (per slice 04 pattern)
- [ ] **Client services catalog**: `/client/services` lists every service with `bookable_by_client=true`, grouped by category; service with no future open slots renders "Aktuell nicht verfügbar" badge instead of next-3-dates
- [ ] **Owner toggles bookable_by_client**: turning off a previously-bookable service hides it from `/client/services` within 5 s via Realtime; existing pending requests for it are unaffected (continue through the inbox)
- [ ] **Min-notice enforcement**: client tries to book a slot within `min_notice_hours` window → server-action returns `BookingTooSoon`; UI shows the slot in ochre with "Nicht mehr buchbar"; no `service_requests` row written
- [ ] **Max-advance enforcement**: day picker on `/client/services/[id]` blocks navigation past `max_advance_days` from today
- [ ] **Per-service catalog hides empty kinds**: stable with zero `bookable_by_client=true` services shows the empty-state copy at `/client/services`; nav link is still visible (but greyed)

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
