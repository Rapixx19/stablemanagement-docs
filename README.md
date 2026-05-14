# stablemanagement-

Swiss-localised stable management SaaS. Three faces (Owner, Client, Worker), one schema, one design system. La Fattoria is the V1 design partner.

## What this is

A multi-tenant web platform for Swiss boarding stables (`Pensionsställe`) of 11–40 horses. Replaces the WhatsApp + Excel + paper whiteboard workflow with a closed-loop operational system: horses, people, schedule, tasks, billing with QR-bill, Bexio push, comms.

V1 ships as **one Next.js app with three role-based route trees** sharing one Postgres + one design system:
- **Owner** (`/owner/*`) — desktop-first, the management surface
- **Client** (`/client/*`) — desktop + mobile, the boarder portal
- **Worker** (`/worker/*`) — mobile-first PWA, the barn surface

## Stack

- Next.js 14 + TypeScript on Vercel (single app, Turborepo monorepo for shared packages)
- Supabase Postgres in Frankfurt with RLS, plus Supabase Realtime + Storage + pg_cron
- Vercel Cron for external-API scheduled jobs (Bexio reconciliation, email digest)
- Auth: Supabase Auth (magic link primary)
- PDF: `swissqrbill` + react-pdf
- i18n: DE / FR / IT trilingual day one (ICU MessageFormat)
- Sentry, structured logs (Pino), UptimeRobot

No Python / Railway / FastAPI in V1. V1.5 adds a dedicated ML service if a real use case lands.

## Repo layout

See `ARCHITECTURE.md`. High level:
```
apps/web/app/{owner,client,worker}/    One Next.js app, three role trees
packages/{ui,db,lib,i18n}/             Shared
supabase/{migrations,policies,cron}/   DB schema, RLS, pg_cron jobs
docs/                                  This folder
```

## Build sequence

18 vertical slices, ~20–22 weeks for 2 engineers + freelancer (revised 2026-05-14 per `/plan-ceo-review` HOLD SCOPE session). Each slice ships end-to-end (schema → API → UI → test → demo to La Fattoria). See `slices/00-foundation.md` through `slices/16-pilot.md`. Slice 01 was split into 01a (people + horses + photos + profile UI, 5 dev-days) and 01b (unified documents + 5-path ingestion + Reducto search + email aliases, 9 dev-days) per D14/D15.

| Phase | Weeks | Slices |
|---|---|---|
| Foundation | 1–2 | 00 |
| Operational core | 3–8 | 01a, 01b, 02, 03, 04 |
| Billing | 9–12 | 05, 06, 07, 08, 09 |
| Comms + booking | 13–14 | 10, 11 |
| Worker | 15–16 | 12, 13, 14, 15 |
| Polish + pilot | 17–20 | 16 |

## How to read these docs

- **Cursor agents:** Always read `.cursorrules.md` first. It is the global rulebook.
- **Building a specific slice:** open `slices/NN-name.md`. It has the full spec, schema diff, acceptance criteria.
- **Cross-cutting concerns:** `domains/` has the references (RLS, VAT, QR-bill, i18n, …) referenced by multiple slices.
- **Operational:** `runbooks/` has the on-call docs (auth recovery, Bexio outage, etc.).

## Repo conventions

- Every slice has acceptance criteria. A slice is not done until they pass.
- Every table has an RLS policy. Every policy has a paired test.
- Integration tests are mandatory, not optional. See `INTEGRATION_TESTS.md`.
- 150-line cap per `.md` file in `docs/`. If a file exceeds 150 lines, split it.

## Pilot exit criteria

V1 is not done until La Fattoria meets all of:
1. 4 consecutive weekly invoice cycles run without owner intervention beyond review
2. 0 RLS test failures in CI for 30 days
3. 0 P0 bugs (data loss, wrong invoice amount, auth break) for 30 days
4. La Fattoria reports they have not opened Excel/WhatsApp for stable management in the prior week
5. Mean QR-bill PDF generation < 3s, p99 < 8s

## Out of V1 (V1.1, V2)

ML layer, Sentavita biometrics, TAMV journal, inventory, reporting, Identitas API, Abacus, FNCH, WhatsApp, native mobile apps. See `domains/v1-deferred.md` for the full deferred list and reasoning.

## Owners

- **Ferdinand** — product, algo review, La Fattoria liaison
- **Sharad** — backend, Supabase, integrations
- **Freelancer** — frontend, design system implementation
