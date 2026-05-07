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

## Change log

- 2026-05-07 — Initial decisions captured during pre-coding review. Locks in single-app topology, Next.js 16, Drizzle, Fluid Compute, `@serwist/next`, no-Railway, no-ClamAV-V1, decimal.js, email-only worker login.
