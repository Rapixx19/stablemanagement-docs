# Slice 01 — People and Horses

**Phase:** Operational core · **Estimate:** 5 dev-days · **Owner:** Sharad backend, freelancer frontend

The master objects everything else attaches to. People are clients, vets, farriers, trainers. Horses belong to people. Horses have documents. This is the foundation of the operational core.

## Goal

Owner can create, edit, view, archive people and horses. Each horse has a profile page with all its documentation in one place. Documents have expiry dates that surface as alerts.

## What ships

### Owner side

1. **People list** at `/people`. Filter by type (client / vet / farrier / trainer), search by name, sort by name or recent activity.
2. **People detail** at `/people/[id]`. Contact info, language preference, role, linked horses, payment terms, optional Bexio contact ID. Edit in place.
3. **Horse list** at `/horses`. Filter by status (active / sold / deceased), search by name, sort.
4. **Horse detail** at `/horses/[id]`. Tabs: Profile · Documents · Health · Schedule (read-only stub for slice 03). Profile has UELN, microchip, FEI passport, owner (link), boarding plan (slice 06 stub), arrival date, stall (slice 02 stub), allergies, medical notes, photos.
5. **Document vault.** Upload PDF / JPG / PNG. Tag with kind (passport / vet / farrier / insurance / contract / other). Set expiry date. List, download, replace, delete.
6. **Document expiry alerts.** When `expires_at` < 30 days, surfaces as a yellow badge on the horse card and as a row in the dashboard alerts panel (slice 16).

### Client side

7. **My horses list** at `/horses`. Read-only. Shows the horses the logged-in client owns at this stable.
8. **My horse detail** at `/horses/[id]`. Read-only. Same tabs as owner side but no edit, no Health tab in V1.

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
  language text check (language in ('de','fr','it')) default 'de',
  bexio_contact_id text,
  payment_terms_days int default 30,
  notes text,
  archived_at timestamptz,
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

create table horse_documents (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  horse_id uuid not null references horses(id),
  kind text not null check (kind in ('passport','vet','farrier','insurance','contract','xray','other')),
  filename text not null,
  file_path text not null,
  size_bytes bigint not null,
  expires_at date,
  uploaded_by_user_id uuid references auth.users(id),
  created_at timestamptz not null default now()
);

create table horse_photos (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  horse_id uuid not null references horses(id),
  file_path text not null,
  is_primary boolean not null default false,
  created_at timestamptz not null default now()
);
```

## RLS

- `people`, `horses`, `horse_documents`, `horse_photos`: scoped by `stable_id` via `auth.current_stable_id()`
- Clients (`role='client'`) see only horses where they are the `owner_person_id`
- Workers see all horses but cannot UPDATE
- Storage policies on the documents bucket mirror the same logic — see `domains/rls.md`

## Constraints and rules

- Document upload limit: **25 MB per file**. Reject larger uploads at the server action.
- Allowed MIME types: PDF, JPG, PNG. No videos in V1 (storage cost).
- ClamAV virus scan on every upload before write to Supabase Storage. See `domains/storage.md`.
- UELN format validated (15-char ISO 11784 ID). If invalid, save with warning, don't block.

## Acceptance criteria

- [ ] Owner can create a horse with required fields and link to a client
- [ ] Owner uploads a 5 MB PDF passport with expiry → file is virus-scanned, persisted, expiry surfaces in horse card after 11 months
- [ ] Owner uploads a 30 MB file → rejected with clear error
- [ ] Owner uploads an `.exe` file → rejected at MIME check
- [ ] Client logs in, sees only horses they own; cannot see other clients' horses
- [ ] Worker sees all horses but UPDATE buttons hidden
- [ ] Archiving a horse keeps audit log intact and removes it from default list
- [ ] DE/FR/IT labels render on horse profile

## Acceptance integration test

`apps/owner/tests/integration/horses.test.ts` — owner uploads document, RLS test confirms client in stable A cannot fetch horse from stable B.

## Out of scope

- Stall assignment (slice 02)
- TAMV treatment journal (V1.1)
- Health timeline (V1.1)
