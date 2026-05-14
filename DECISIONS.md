# Decisions — Locked V1 Architecture

The 12 choices that everything else assumes. Bake these in before slice 00. If a slice contradicts this file, the slice is wrong.

## Why this file exists

These were re-litigated during the pre-coding review (2026-05-07). The answer is locked here so we don't re-debate at slice-time. Override only via PR with explicit reason. Pair every change with a sweep of affected slices/domains.

## D1. App topology — one Next.js app

One Next.js project at `apps/web/` with three role-based route trees: `app/owner/*`, `app/client/*`, `app/worker/*`. **Not three apps.** PWA scope confines the worker face. Bundle isolation enforced via ESLint `no-restricted-imports` per route tree (`packages/lib/eslint-config/{owner,client,worker}.js`) and per-tree Tailwind `content` glob.

## D2. Next.js version — 16

Greenfield; we adopt **Next.js 16** with App Router + Cache Components. The "tile < 1s p95" budget is a `use cache` + `cacheLife` solution, not ad-hoc `revalidate` numbers. Pin minor (`16.x.y`) in `package.json` exact, no `^`.

## D3. DB access — Drizzle + postgres-js

`drizzle-orm` with the `postgres-js` driver. Type-safe schema mirrors `packages/db/schema.sql`. Raw SQL escape hatch via `db.execute(sql\`...\`)` when RLS policies or pg-specific features need it. Migrations: Supabase CLI + Drizzle types regenerated on every migration apply.

## D4. Vercel runtime — Fluid Compute (Node 24)

Every server action and route handler runs on **Fluid Compute Node 24 LTS**. No Edge runtime — `react-pdf`, `sharp`, `pgcrypto`, and `pg` need Node. Fluid Compute reuses instances across concurrent requests, killing cold-start cost. Set `export const runtime = 'nodejs'` explicitly in PDF + image routes; rely on default elsewhere.

## D5. Worker PWA — `@serwist/next`

Service worker via `@serwist/next` (current App Router-compatible fork of `next-pwa`). SW served from a route handler at `/worker/sw.ts` with `Service-Worker-Allowed: /worker/` header. Manifest at `/worker/manifest.webmanifest`, `scope: "/worker/"`, `start_url: "/worker/home"`.

## D6. Email queue drain — Vercel Cron

`email_queue` table in Postgres; **Vercel Cron** at `/api/cron/email-drain` every 60s pulls and sends via Resend (Postmark fallback via env flip). No Railway. Authenticated by `CRON_SECRET` bearer.

## D7. Task materialisation — Vercel Cron, not pg_cron

Slice 04's nightly job is **Vercel Cron** at `/api/cron/task-materialisation` daily 03:00 Europe/Zurich. Pure SQL cannot expand RRULEs; Node + `rrule.js` (already a dep) does. Subscription run (slice 06) and audit-log partition rotation (slice 00) stay in pg_cron because both are pure SQL.

## D8. Pgcrypto + Pooler pattern

OAuth tokens are `bytea` columns (`oauth_access_token bytea not null`), not `text`. The encryption key is passed in by the server action on every call. Pattern (must be inside one transaction):
```sql
BEGIN;
  SELECT set_config('app.crypto_key', $1, true);  -- true = txn-local
  SELECT pgp_sym_decrypt(oauth_access_token, current_setting('app.crypto_key'))
    FROM bexio_connections WHERE stable_id = $2;
COMMIT;
```
Transaction-mode pooler discards session state between transactions, so `set_config` must be `true` (txn-local) and inside the same TX as the decrypt.

## D9. Virus scan — defer to V1.1

V1 ships **without ClamAV.** Slice 01 enforces: server-side MIME sniffing via `file-type` npm + magic-byte check + size cap (25 MB documents, 10 MB photos, 10 MB attachments). Reject executables and macro-enabled Office formats by extension *and* magic bytes. Document the residual risk in `domains/storage.md` and pilot agreement. V1.1 adds either Cloudmersive virus-scan API or a single Fly.io ClamAV VM — re-evaluate after pilot.

## D10. Money — `bigint _cents` + `decimal.js`

All money columns are `bigint not null check (amount_cents >= 0)`. No `numeric(12,2)`, anywhere. VAT line math uses **`decimal.js` with `Decimal.ROUND_HALF_EVEN`**. The custom `roundHalfToEven` in `domains/money.md` is removed — the helper in `packages/lib/billing/round.ts` is a thin wrapper around `decimal.js`. Bankers' rounding choice confirmed by Treuhänder pre-pilot.

## D11. Worker login — email-only V1

Workers must have an email to receive a magic link. Workers without email cannot be onboarded in V1 — owner sees a clear error and is told to enter the worker's email later. **V1.1** adds owner-managed 4-digit PIN login as the casual-labour path. Documented in `domains/auth.md` and pilot agreement.

## D12. Cron auth — single `CRON_SECRET`

Every `/api/cron/*` route handler verifies `Authorization: Bearer ${process.env.CRON_SECRET}` on first line. Rejects with 401 otherwise. Vercel Cron config sends the header automatically. Same secret works for all crons; rotate via the standard rotation procedure in `runbooks/`.

## D13. Membership role enum — drop `manager` from V1

`memberships.role` enum is `'owner' | 'worker' | 'client'` in V1. **No `manager`.** Pilot is one stable with one owner; no slice has manager-specific permissions distinct from owner; every previously-written `owner/manager` clause collapses cleanly to `owner`. Adding the role back is a one-line migration if multi-stable chain ops or stable delegates emerge in V1.1+ — dropping a role later is destructive, so we start narrow. All slice RLS sections, `domains/rls.md`, `domains/auth.md`, and `SCHEMA.md` swept on 2026-05-13.

## D14. Document storage — unified `documents` table + Reducto search

V1 ships a single `documents` table polymorphic over `(entity_kind, entity_id)` for horses, clients, and the stable itself, replacing the original `horse_documents` table. All uploaded PDFs flow through Reducto for layout-aware extraction + embedding into pgvector, indexed for natural-language search per `domains/document-search.md`. Customer-uploaded horse photos are allowed in three defined contexts per `DESIGN.md` (logo / horse profile / dashboard "active horses" strip) — no other imagery permitted. Slice 01 schema diff updated; slice 16 onboarding gets a Reducto consent screen.

## D15. Document ingestion — five paths, one review queue

V1 ships **five automated ingestion paths** for documents (per `domains/document-ingestion.md`): per-stable email catch-all (`documents.{stable}@inbox.stableplatform.ch`) with Reducto+Haiku triage; per-entity email aliases (owner-generated, vendor-bound) that skip triage; PWA Web Share Target manifest entry; mobile camera capture via `<input capture="environment">`; bulk historical import as an onboarding step (slice 16). All five land in a single review queue at `/owner/inbox/documents` for one-tap confirm. Costs capped at $5/mo Resend inbound + $5/mo Haiku triage + shared $50/mo Reducto per stable. La Fattoria pilot adoption depends on this — manual upload alone leaves the document vault empty. Adds ~5 dev-days to V1; pilot timeline still ≤ 19 weeks.

## D16. Client-bookable services — opt-in per service

V1 ships a discovery-surface catalog at `/client/services` (slice 11 #11) listing only the services where the owner has explicitly set `services.bookable_by_client = true`. Default is **off** so owners don't accidentally expose internal services like "Notfall-Tierarzt" or "Foaling-Begleitung" to client self-booking. Per-service controls (slice 05 #6): `client_facing_name`, `client_description`, `min_notice_hours`, `max_advance_days`, `default_provider_person_id`. Existing slot mechanism (slice 11 `availability_slots`) is reused — the catalog page is pure UX over the same booking primitive. Client booking through the catalog flows through the same hold-on-request transaction, owner inbox, and confirmation as calendar-driven booking.

## D17. Slice 01 split into 01a + 01b

Slice 01 grew from the May 7 estimate of 5 dev-days to a realistic 12–15 dev-days after D14 (unified documents + Reducto) and D15 (5-path ingestion). Per `/plan-ceo-review` HOLD SCOPE session 2026-05-14, split into two slices:

- **Slice 01a — People and Horses** (5 dev-days): people, horses, horse_photos tables. Owner + client profile UI without Documents tab. The original May 7 scope.
- **Slice 01b — Documents, Ingestion, Search** (9 dev-days): unified polymorphic `documents` table + `email_aliases` table. Reducto embedding pipeline. 5 ingestion paths (email catch-all, per-entity aliases, mobile camera, PWA share-target, bulk import). Global search at `/owner/search`. Adds the Documents tab to slice 01a profiles. Depends on slice 00 `vector` extension + all 5 API keys provisioned (Ferdinand-owned pre-slice-00 sprint).

Realistic V1 timeline revised to **20–22 weeks** for 2 engineers + freelancer (was 17–18 weeks). The split lets 01a ship independently — La Fattoria can use people + horses + photos before 01b's Reducto integration is ready.

## Change log

- 2026-05-14 — Reverted hybrid TS+Python proposal (briefly considered as D18). V1 stays TS-only per original D6. Bulletproof billing math invariants (Decimal discipline, frozen VAT rate, audit-per-step, golden fixture regression, idempotent monthly cron) still apply — implemented in TS via `decimal.js` + property-based testing. Python port deferred to V1.5+ if/when a real precision or scale need surfaces. Rationale: avoid the rewrite trap; single language simplifies onboarding for new full-stack hire and keeps timeline at 20–22 weeks.
- 2026-05-14 — D17: slice 01 split into 01a (people + horses + photos, 5d) and 01b (documents + ingestion + Reducto + search + aliases, 9d). README phase table updated. Realistic V1 timeline 20–22 weeks. Triggered by `/plan-ceo-review` HOLD SCOPE session.
- 2026-05-14 — Pre-slice-00 API-key provisioning sprint locked to Ferdinand (TODOS.md). 5 keys: Reducto + DPA, OpenAI embeddings, Anthropic, Resend inbound, Bexio dev sandbox.
- 2026-05-14 — D16: client-bookable services catalog locked. Slice 05 schema gains `bookable_by_client` + booking rules. Slice 11 gains `/client/services` (catalog) + `/client/services/[id]` (per-service booking page). New mockup `client-services.html`.
- 2026-05-14 — D15: document ingestion architecture locked. New `domains/document-ingestion.md`. Slice 01 extended with inbox queue + per-entity aliases + mobile scan. Slice 12 PWA manifest gains `share_target`. Slice 16 onboarding gains bulk-import step. New mockup `owner-document-inbox.html`.
- 2026-05-14 — D14: unified `documents` table + Reducto integration locked. Slice 01 extended with full client + horse profile specs; `domains/document-search.md` written; `client-profile.html` + `horse-profile.html` mockups added. Owner dashboard refreshed with sparklines + timeline + horse strip + alerts band (visual-density rebalance per user feedback). DESIGN.md amended to allow horse photos in three contexts.
- 2026-05-13 — D13: drop `manager` from membership role enum (V1). Swept across all swept slices + `SCHEMA.md` + `domains/{rls,auth,v1-deferred}.md`.
- 2026-05-13 — Slice 17 inventory deferred to V1.1 (full spec preserved in `slices/17-inventory.md`; rationale in `domains/v1-deferred.md`).
- 2026-05-13 — Bexio integration confirmed optional. V1 ships Bexio OR no-integration (manual mark-paid + invoice CSV export). Onboarding picks per-stable. Spec'd in `slices/09-bexio.md` "Scope and choice" + `slices/08-invoice-pdf.md` items 6–7 + `slices/16-pilot.md` wizard.
- 2026-05-13 — Pilot exit criterion #4 left as self-report only (no objective complement added). Trust-pilot-owner pattern preferred over audit-style metrics for n=1 pilot.
- 2026-05-07 — Initial decisions captured during pre-coding review. Locks in single-app topology, Next.js 16, Drizzle, Fluid Compute, `@serwist/next`, no-Railway, no-ClamAV-V1, decimal.js, email-only worker login.
