# Schema — V1

Canonical reference. Source of truth lives in `packages/db/schema.sql`. This doc is the human-readable index.

## Conventions

- All tables have `id uuid primary key default gen_random_uuid()`
- All multi-tenant tables have `stable_id uuid not null references stables(id)`
- Timestamps are `timestamptz`, defaulted to `now()` where applicable
- Money is **`bigint not null check (>= 0)`** suffixed `_cents` (e.g. `unit_price_cents`). Never `numeric` for amounts. See `domains/money.md`.
- Encrypted columns (e.g. OAuth tokens) are `bytea`, not `text`. See `DECISIONS.md` D8.
- Soft delete via `deleted_at timestamptz null` where audit matters; `archived_at timestamptz null` where the row should disappear from default lists but is not "deleted" (people, horses); hard delete elsewhere
- All mutable tables have `updated_at timestamptz not null default now()` maintained by trigger
- Every table has an RLS policy in `supabase/policies/<table>.sql` and a paired test

## Tenants and identity

| Table | Purpose |
|---|---|
| `stables` | Tenant root. UID-MWST, address, default locale (DE/FR/IT), VAT method (effektiv/saldo). |
| `users` | Supabase Auth users. Mirrored from `auth.users`. |
| `memberships` | (`user_id`, `stable_id`, `role`). Role enum: `owner`, `manager`, `worker`, `client`. One user can have many memberships. |
| `audit_log` | Append-only mutation log. (`stable_id`, `actor_user_id`, `entity`, `entity_id`, `action`, `before_jsonb`, `after_jsonb`, `at`). |

## People and horses

| Table | Purpose |
|---|---|
| `people` | Clients, vets, farriers, trainers, staff (non-login records). Type enum, contact, language, optional `bexio_contact_id`. |
| `horses` | (`stable_id`, `name`, `ueln`, `microchip`, `fei_id`, `owner_person_id`, `status`, `nutztier_heimtier`). |
| `horse_documents` | (`horse_id`, `kind`, `file_path`, `expires_at`). Kinds: passport / vet / farrier / insurance / contract / other. |
| `horse_assignments` | (`horse_id`, `stall_id`, `from`, `to`). Current = `to is null`. |

## Physical layout

| Table | Purpose |
|---|---|
| `stalls` | (`stable_id`, `code`, `row`, `col`, `kind`, `capacity`, `active`). Kind enum: stall / paddock / maintenance. |

## Calendar

| Table | Purpose |
|---|---|
| `events` | (`stable_id`, `kind`, `starts_at`, `ends_at`, `recurrence_rule`, `visibility`, `notes`). Kind enum: lesson / vet / farrier / transport / competition / turnout / clinic / social. Visibility enum: `private` / `public_stable` / `public_open`. |
| `event_horses` | (`event_id`, `horse_id`). Many-to-many. |
| `event_people` | (`event_id`, `person_id`, `role`). Role enum: rider / vet / farrier / trainer / attendee. |
| `availability_slots` | (`stable_id`, `service_id`, `starts_at`, `ends_at`, `capacity`, `status`). For bookable services. |
| `tasks` | (`stable_id`, `title`, `due_date`, `recurrence_rule`, `assigned_to_user_id`, `completed_by_user_id`, `completed_at`). |

## Workers, shifts, time

| Table | Purpose |
|---|---|
| `shifts` | (`stable_id`, `worker_user_id`, `starts_at`, `ends_at`, `status`, `break_minutes`, `notes`). Status enum: scheduled / clocked-in / completed / sick / no-show. |
| `time_punches` | (`stable_id`, `worker_user_id`, `punched_at`, `kind`, `geo_within_fence` bool). Kind enum: in / out / break_start / break_end. **No raw GPS stored.** |
| `unavailability` | (`stable_id`, `worker_user_id`, `from`, `to`, `reason`, `notified_at`, `approved_by`, `approved_at`). Reason enum: sick / vacation / personal. |

## Service catalog and billing

| Table | Purpose |
|---|---|
| `services` | (`stable_id`, `name`, `default_price`, `vat_code`, `unit`, `default_recurrence`). Unit enum: month / session / km / kg / piece / once. |
| `client_subscriptions` | (`stable_id`, `person_id`, `horse_id` nullable, `service_id`, `price`, `auto_bill`, `active_from`, `active_to`). |
| `tab_lines` | (`stable_id`, `person_id`, `horse_id` nullable, `service_id` nullable, `description`, `qty`, `unit_price`, `vat_code`, `occurred_on`, `invoice_id` nullable, `created_by_user_id`). |
| `invoices` | (`stable_id`, `person_id`, `number`, `period_start`, `period_end`, `status`, `total_excl_vat`, `total_vat`, `total_incl_vat`, `qr_reference`, `pdf_path`, `bexio_invoice_id`, `sent_at`, `paid_at`). Status enum: draft / sent / paid / overdue / cancelled. |
| `invoice_lines` | Frozen copy of `tab_lines` at invoice time. |
| `vat_rates` | (`code`, `rate`, `label`). Reference data, seeded with CH 2026 codes. |

## Communication and requests

| Table | Purpose |
|---|---|
| `message_threads` | (`stable_id`, `person_id`, `horse_id` nullable). |
| `messages` | (`thread_id`, `sender_user_id`, `body`, `created_at`, `read_at`). |
| `service_requests` | (`stable_id`, `person_id`, `horse_id` nullable, `kind`, `description`, `status`, `requested_for`, `slot_id` nullable, `decided_by_user_id`, `decided_at`). Status enum: pending / confirmed / declined / scheduled / cancelled. |

## Indexes (V1 minimum)

```sql
create index on horses (stable_id, status);
create index on events (stable_id, starts_at);
create index on events using gist (tstzrange(starts_at, ends_at));
create index on tab_lines (stable_id, person_id, occurred_on);
create index on invoices (stable_id, period_start, status);
create index on time_punches (stable_id, worker_user_id, punched_at);
create index on messages (thread_id, created_at);
create index on audit_log (stable_id, entity, entity_id, at);
```

## Migrations rule

- Numbered files: `001_init.sql`, `002_horses.sql`, …
- Every migration ships with: schema change, RLS policy, RLS test
- Never edit a merged migration. Add a new one.
- See `domains/migrations.md`.
