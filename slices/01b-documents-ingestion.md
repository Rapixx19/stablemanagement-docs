# Slice 01b — Documents, Ingestion, Search

**Phase:** Operational core · **Estimate:** 9 dev-days · **Owner:** Sharad backend, freelancer frontend · **Depends on:** slice 00 (`vector` extension), slice 01a (people + horses + photos), all 5 API keys provisioned (TODOS.md pre-slice-00 sprint)

The document layer. Every PDF, photo, contract, vet report, insurance renewal, x-ray, ID copy lives in one polymorphic `documents` table — searchable across all entities by natural-language query via Reducto + pgvector. **Five ingestion paths** get docs into the platform without owner upload being the bottleneck.

## Goal

A vet emails a PDF report to the stable's catch-all address → it appears in `/owner/inbox/documents` within 60s, pre-categorised as "Diva · vet" with high confidence → owner one-tap confirms → doc joins the horse's Documents tab + becomes searchable in DE/FR/IT/EN. Same flow for client-bound docs (insurance, contracts) via per-client aliases.

## What ships

### Owner side

1. **Unified document vault.** Upload PDF / JPG / PNG attached to a horse, a client, or the stable. Tag with kind (passport / vet / farrier / insurance / contract / id_copy / authorisation / xray / invoice_attachment / other). Set expiry date. List, download, replace, archive (never hard-delete per `domains/data-retention.md`).
2. **Documents tab on horse profile** — slice 01a horse profile gains this tab. Passport, vet reports, farrier records, insurance, x-rays, contracts, authorisations.
3. **Documents tab on client profile** — slice 01a client profile gains this tab. Contracts, ID copies, insurance, vet authorisations.
4. **Document expiry alerts.** When `documents.expires_at` < 30 days, yellow badge on parent profile + row in dashboard alerts panel (slice 16). Applies to horse docs (passport, insurance) and client docs (insurance renewal, contract).
5. **Document inbox** at `/owner/inbox/documents` — review queue for docs arriving via the 5 paths. Each row: filename + first-page thumbnail + sender + LLM-proposed entity (confidence pill: moss ≥0.9, ochre 0.7–0.9, rust <0.7) + proposed kind + proposed expiry. One-tap confirm / edit / reject. Bulk-confirm for high-conf rows. Mockup: `mockups/owner-document-inbox.html`. Pipeline: `domains/document-ingestion.md`.
6. **Per-entity email aliases.** "Generate inbox alias" button on every horse + client profile creates a per-entity address (`diva.lafattoria@inbox.stableplatform.ch`). Inbound to an alias bypasses LLM triage — owner only picks `kind`. Sidebar of inbox lists active aliases.
7. **Mobile document scan** — slice 01a upload UI gains "Dokument scannen" CTA on mobile opening device camera (`<input capture="environment">`). iOS auto-edge-detects and produces PDF, routes to inbox queue.
8. **Global search** at `/owner/search` — Reducto-indexed natural-language search across documents + horses + clients + events + messages + invoices. `⌘K` opens the same surface as a quick-search modal. Mockup: `mockups/owner-search.html`. Spec: `domains/document-search.md`.

### Client side

9. **Documents tab on My horse profile** — read-only download of documents attached to client's horse(s).
10. **Documents tab on My profile** — read-only download of documents the stable has attached to the client (contracts, insurance, etc.). Surfaces redaction request via Datenschutz settings.

## Schema diff

```sql
-- 003_documents_ingestion.sql
create table documents (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  entity_kind text not null check (entity_kind in ('horse','client','stable')),
  entity_id uuid not null,
  kind text not null check (kind in (
    'passport','vet','farrier','insurance','contract',
    'id_copy','authorisation','xray','invoice_attachment','other'
  )),
  filename text not null,
  file_path text not null,
  size_bytes bigint not null,
  mime_type text not null,
  expires_at date,
  -- Reducto + embedding pipeline (see domains/document-search.md)
  extracted_text text,
  extracted_text_lang text,
  embedding vector(1536),                            -- requires slice 00 `vector` extension
  reducto_job_id text,
  reducto_status text not null default 'pending' check (reducto_status in ('pending','processing','done','failed','skipped')),
  reducto_processed_at timestamptz,
  reducto_error text,
  -- Ingestion provenance (see domains/document-ingestion.md)
  ingestion_source text not null default 'manual' check (ingestion_source in (
    'manual','email','camera','share_target','bulk_import','bexio_harvest','vendor_alias'
  )),
  ingestion_email_id text,
  ingestion_sender text,
  triage_status text not null default 'confirmed' check (triage_status in ('pending_review','confirmed','rejected')),
  triage_proposed_jsonb jsonb,
  uploaded_by_user_id uuid references auth.users(id),
  archived_at timestamptz,
  redacted_at timestamptz,
  created_at timestamptz not null default now()
);

create index on documents (stable_id, entity_kind, entity_id, archived_at);
create index on documents (stable_id, triage_status, created_at desc) where triage_status = 'pending_review';
create index on documents (stable_id, expires_at) where expires_at is not null and archived_at is null;
create index on documents using ivfflat (embedding vector_cosine_ops) with (lists = 100)
  where embedding is not null and archived_at is null;

create table email_aliases (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  alias text not null,                               -- local part: "diva.lafattoria" or "documents.lafattoria"
  entity_kind text not null check (entity_kind in ('horse','client','catch_all')),
  entity_id uuid,                                    -- null when entity_kind='catch_all'
  default_kind text,                                 -- e.g. 'vet' for an alias handed to the vet
  active boolean not null default true,
  version int not null default 0,
  created_at timestamptz not null default now()
);
create unique index email_aliases_unique on email_aliases (alias) where active = true;
create index on email_aliases (stable_id, entity_kind, entity_id);
```

## RLS

- `documents` scoped by `stable_id` per `domains/rls.md`. Polymorphic guard at server-action layer: verify `entity_id` exists in `(horses | people | stables)` of the same `stable_id` before INSERT.
- `owner`: full CRUD on `documents` and `email_aliases`.
- `worker`: SELECT only where `entity_kind='horse'` (operational context). **No client-document access.**
- `client`: SELECT where `(entity_kind='horse' AND entity_id IN (own horses)) OR (entity_kind='client' AND entity_id = own people row)`. No INSERT/UPDATE/DELETE.
- `archived_at` (soft-delete) vs. `redacted_at` (nFADP scrubbed) — **never hard DELETE** per `domains/data-retention.md`.
- Storage policies mirror the same logic. File path convention: `{stable_id}/{entity_kind}/{entity_id}/{document_id}.{ext}`.
- `email_aliases`: owner-only CRUD. Workers + clients no access.

## Constraints

- Document upload: ≤25 MB per file. Allowed MIME: `application/pdf`, `image/jpeg`, `image/png`. Server-side magic-byte validation (`file-type` npm). ClamAV deferred to V1.1 per D9.
- Per-stable cost caps (`domains/document-search.md` + `domains/document-ingestion.md`): Reducto $50/mo, OpenAI embeddings $20/mo, Haiku triage $5/mo. Soft alert at 50%, hard block at 100%.
- Per-stable rate limits: 5 manual uploads/day, 100 inbound emails/day, 1 bulk import/day (≤200 PDFs / ≤100 MB).
- `BookingTriageMultiMatch`: when Haiku proposes an entity name matching ≥2 records, drop confidence to 0.5 and force owner pick.

## Cross-slice dependencies

- **Slice 00** must enable `vector` extension (added to slice 00 acceptance 2026-05-14).
- **Slice 12** PWA manifest gains `share_target` entry pointing at `/owner/inbox/share-receive` — owned by slice 12.
- **Slice 16** onboarding wizard gains "Bulk historical document import" step calling this slice's bulk-import endpoint — owned by slice 16.
- **Ferdinand-owned pre-slice-00 sprint** (TODOS.md): `REDUCTO_API_KEY` + signed DPA, `OPENAI_EMBEDDING_API_KEY`, `ANTHROPIC_API_KEY`, `RESEND_API_KEY` + inbound MX on `inbox.stableplatform.ch` + webhook secret. Without all five, slice 01b cannot ship.

## Acceptance criteria

- [ ] Owner uploads a 5 MB PDF passport on horse profile → file persisted, MIME-validated, expiry surfaces in horse card after 11 months
- [ ] Vet emails a PDF to `documents.lafattoria@…` → within 60s, document in `/owner/inbox/documents` with proposed entity Diva (confidence ≥0.9)
- [ ] Owner taps Confirm → doc moves to Diva's Documents tab, becomes searchable
- [ ] Per-horse alias `diva.lafattoria@…` routes directly to Diva, skips LLM triage
- [ ] PWA Share Target on Android Chrome: PDF shared from Gmail → lands in queue
- [ ] Mobile camera scan → PDF/JPG lands in queue
- [ ] Bulk import: ZIP of 50 PDFs → all triaged within 5 min, owner sees batch
- [ ] Webhook signature validation: forged POST returns 401, no row written
- [ ] **EXPLAIN ANALYZE** on polymorphic SELECT under 30-stable × 200-doc seed shows Index Scan on `(stable_id, entity_kind, entity_id, archived_at)`, not Sort+Hash
- [ ] **Cross-language search recall**: 50 fixture pairs (DE query / FR doc, IT/DE, EN/IT) → top-5 recall > 80%
- [ ] Stable A's docs never appear in stable B's search (RLS test)
- [ ] Client A's `/client/search` returns only own + own-horse docs (no cross-client leak)
- [ ] **Redaction propagation**: `redactPerson(personId)` scrubs `extracted_text` + `embedding` for all their docs in one TX
- [ ] Reducto cost hits 50% → email alert; 100% → hard block; existing search still works
- [ ] `ReductoApiUnreachable` rescue: mock 503 for 5 attempts → doc enters DLQ, banner on alerts panel
- [ ] `BookingTriageMultiMatch`: doc mentioning "Marta" with two Martas in stable → confidence drops to 0.5, owner forced to pick
- [ ] **Polymorphic FK guard (P1 per `/plan-eng-review`)** — three mechanisms:
  - Server-action validates `entity_id` exists in the matching table (`horses` | `people` | `stables`) of the same `stable_id` pre-INSERT. Returns `EntityNotFound` rescue path if missing.
  - Nightly Vercel Cron at `/api/cron/documents-drift` compares `documents.entity_id` against valid references; orphans get `archived_at` set + DLQ entry.
  - `BEFORE UPDATE` trigger on `horses` + `people` archiving → soft-archives their documents in same TX.

## Acceptance integration test

`apps/web/tests/integration/owner/documents-ingestion.test.ts` — vet emails PDF to catch-all alias, Reducto extracts, Haiku triages, owner confirms, doc appears searchable on horse profile. Tenant isolation verified across all 5 ingestion paths.

## Out of scope

- ClamAV virus scanning — V1.1 per D9
- Chunked embeddings for long docs (>8 chunks) — V1.1 per `domains/document-search.md`
- WhatsApp inbound — V2
- Bexio attachment harvest — V1.1 (`bexio_harvest` enum value reserved in `ingestion_source`)
- TAMV treatment journal export from documents — V1.1
