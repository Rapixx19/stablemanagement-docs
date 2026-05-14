# Slice 01a — People and Horses

**Phase:** Operational core · **Estimate:** 5 dev-days · **Owner:** Sharad backend, freelancer frontend

The master objects everything else attaches to. People are clients, vets, farriers, trainers. Horses belong to people. Horses have photos. **The document vault + ingestion + Reducto search live in slice 01b**; this slice lays the people + horses + photos foundation that 01b attaches documents to.

## Goal

Owner can create, edit, view, archive people and horses. Each entity has a profile page. Per-client spending summary surfaces from slice 06 in the Billing tab.

## What ships

### Owner side

1. **People list** at `/owner/people`. Filter by type (client / vet / farrier / trainer / other), search by name, sort by name or recent activity.
2. **Client profile** at `/owner/clients/[id]`. Tabs: **Profile · Horses · Communication · Billing · Activity**. (The **Documents** tab is added in slice 01b.) Profile: contact info, language (DE/FR/IT/EN per D14), payment terms, Bexio mapping, notes, member-since. Horses: linked horses with primary photo. Communication: deep-link to slice 10 thread. Billing: summary of current tab + invoice history + per-year spending sparkline (links to slice 06). Activity: chronological event log (events, invoices, requests, messages).
3. **Horse list** at `/owner/horses`. Filter by status (active / sold / deceased / retired), search by name, sort. Grid view with primary photo or list view with details.
4. **Horse profile** at `/owner/horses/[id]`. Hero photo (`is_primary` of `horse_photos`) + name in serif. Tabs: **Profile · Health · Schedule · Photos**. (The **Documents** tab is added in slice 01b.) Profile: UELN, microchip, FEI passport, owner (link to client profile), boarding plan (slice 06), arrival date, stall (slice 02), allergies, medical notes. Health: read-only log of events with kind in vet/farrier (slice 03). Schedule: read-only view of events involving this horse. Photos: gallery of customer-uploaded photos, mark primary.
5. **Per-client spending summary.** On the Billing tab: total spent year-to-date, broken down by service category (boarding / training / care / other). Stacked-bar sparkline showing last 12 months. Mockup reference: `mockups/client-profile.html`.

### Client side

6. **My horses list** at `/client/horses`. Read-only. Shows horses the logged-in client owns at this stable.
7. **My horse profile** at `/client/horses/[id]`. Read-only. Same tabs as owner side but no edit, no Health tab in V1. Photo gallery available.
8. **My profile** at `/client/profile`. Read-only summary of own data (contact, payment terms, linked horses, total spent year-to-date). Settings → Datenschutz link to redaction request (`domains/data-retention.md`).

## Schema diff

```sql
-- 002_people_horses.sql
create table people (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  type text not null check (type in ('client','vet','farrier','trainer','other')),
  user_id uuid references auth.users(id),
  full_name text not null,
  email text,
  phone text,
  language text check (language in ('de','fr','it','en')) default 'de',  -- EN added per D14
  bexio_contact_id text,
  payment_terms_days int default 30,
  notes text,
  archived_at timestamptz,
  redacted_at timestamptz,                         -- nFADP redaction marker (see domains/data-retention.md)
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table horses (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  name text not null,
  ueln text,
  microchip text,
  fei_id text,
  owner_person_id uuid references people(id),
  status text not null default 'active' check (status in ('active','sold','deceased','retired')),
  nutztier_heimtier text default 'heimtier' check (nutztier_heimtier in ('nutztier','heimtier')),
  birth_date date,
  breed text,
  color text,
  sex text check (sex in ('mare','gelding','stallion')),
  height_cm int,
  arrival_date date,
  allergies text,
  medical_notes text,
  archived_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table horse_photos (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  horse_id uuid not null references horses(id),
  file_path text not null,
  caption text,
  is_primary boolean not null default false,
  uploaded_by_user_id uuid references auth.users(id),
  archived_at timestamptz,
  created_at timestamptz not null default now()
);
create unique index horse_photos_one_primary on horse_photos (horse_id) where is_primary = true and archived_at is null;
```

## RLS

- `people`, `horses`, `horse_photos`: scoped by `stable_id` per `domains/rls.md`.
- `owner`: full CRUD on all three tables.
- `worker`: SELECT all horses (operational context) + SELECT linked people of type vet/farrier/trainer/other. **No client-row SELECT** (privacy + nFADP boundary).
- `client`: SELECT own `people` row only. SELECT horses where `owner_person_id = (own people row)`. SELECT `horse_photos` for own horses. UPDATE limited to own `people.language` and `people.notes` (preference fields only) via `BEFORE UPDATE` trigger whitelist.

## Constraints

- UELN format validated (15-char ISO 11784 ID). If invalid, save with warning, don't block.
- Horse photo upload: max 10 MB per photo, allowed MIME (`image/jpeg`, `image/png`). Server-side magic-byte validation per D9.
- ClamAV virus scan deferred to V1.1 per D9.

## Acceptance criteria

- [ ] Owner can create a horse with required fields and link to a client
- [ ] Owner uploads a 5 MB JPG horse photo → persisted, surfaces on profile as `is_primary` if first
- [ ] Owner uploads a 30 MB photo → rejected with clear error
- [ ] Owner uploads an `.exe` file → rejected at MIME check
- [ ] Client logs in, sees only horses they own; cannot see other clients' horses (RLS)
- [ ] Worker sees all horses; UPDATE buttons hidden; SELECT on `people` of type='client' returns 0 rows
- [ ] Archiving a horse keeps audit log intact and removes it from default list
- [ ] DE/FR/IT/EN labels render on horse profile (4 languages per D14)
- [ ] Client UPDATE on own `people.full_name` rejected by trigger; UPDATE on `people.language` succeeds
- [ ] `horse_photos_one_primary` unique index prevents two primary photos per horse
- [ ] Per-client spending summary (Billing tab) renders with last-12-month stacked-bar sparkline once slice 06 data exists

## Acceptance integration test

`apps/web/tests/integration/owner/horses.test.ts` — owner creates horse, uploads photo, marks primary, RLS test confirms client in stable A cannot fetch horse from stable B.

## Out of scope (handled in other slices)

- **Documents** (passport, vet reports, insurance, x-rays, contracts) — slice 01b
- Stall assignment — slice 02
- Calendar events for horses — slice 03
- Billing line items — slice 06
- TAMV treatment journal — V1.1
- Full health timeline — V1.1
