# Architecture

## One app, three role-based route trees

```
┌──────────────────────────────────────────────┐
│  Single Next.js 14 app — app.lafattoria.app  │
│  ┌────────────┐ ┌────────────┐ ┌──────────┐  │
│  │ /owner/*   │ │ /client/*  │ │/worker/* │  │
│  │ desktop    │ │ desktop+   │ │ mobile   │  │
│  │ -first     │ │ mobile     │ │ -first   │  │
│  └─────┬──────┘ └─────┬──────┘ └─────┬────┘  │
│        └──────────────┼──────────────┘       │
│                       │                      │
│         ┌─────────────┴─────────────┐        │
│         │  packages/ui              │        │
│         │  packages/lib             │        │
│         │  packages/db              │        │
│         │  packages/i18n            │        │
│         └─────────────┬─────────────┘        │
└───────────────────────┼──────────────────────┘
                        │
        ┌───────────────┴────────────────┐
        │                                │
   ┌────▼─────────┐              ┌───────▼──────┐
   │ Supabase     │              │ Vercel       │
   │ Postgres     │              │ 1 Next.js    │
   │ + Auth       │              │ + Cron       │
   │ + Storage    │              │ + Edge       │
   │ + Realtime   │              │              │
   │ + pg_cron    │              │              │
   └──────────────┘              └──────────────┘
```

PWA install for the worker face is scoped to `/worker/*` via a manifest with `scope: /worker/` and `start_url: /worker/home`. iOS install via "Add to Home Screen" on Safari, Android via Chrome install prompt. Worker still appears as a standalone app icon despite living under one Next.js project.

## File tree

```
stablemanagement/
├── apps/
│   └── web/                       Single Next.js 14 app
│       ├── app/
│       │   ├── (auth)/login/
│       │   ├── owner/
│       │   │   ├── dashboard/
│       │   │   ├── stalls/
│       │   │   ├── horses/[id]/
│       │   │   ├── people/[id]/
│       │   │   ├── schedule/
│       │   │   ├── billing/{invoices,catalog,tabs}/
│       │   │   ├── messages/
│       │   │   ├── requests/
│       │   │   ├── workers/
│       │   │   └── settings/
│       │   ├── client/
│       │   │   ├── dashboard/
│       │   │   ├── horses/
│       │   │   ├── calendar/
│       │   │   ├── invoices/
│       │   │   ├── messages/
│       │   │   └── requests/
│       │   ├── worker/
│       │   │   ├── home/
│       │   │   ├── schedule/
│       │   │   ├── tasks/
│       │   │   ├── chat/
│       │   │   ├── manifest.webmanifest
│       │   │   └── sw.ts
│       │   └── api/
│       │       ├── healthz/
│       │       └── cron/{bexio-poll,email-digest}/
│       ├── components/
│       └── tests/
├── packages/
│   ├── ui/                        Shared design system
│   │   ├── primitives/
│   │   ├── tokens.ts
│   │   └── tailwind-preset.ts
│   ├── db/
│   │   ├── migrations/
│   │   ├── schema.sql
│   │   ├── seed.ts
│   │   └── types.ts
│   └── lib/
│       ├── auth/
│       ├── billing/{vat,qrbill,tabs,invoice,bexio}/
│       ├── horses/
│       ├── people/
│       ├── schedule/
│       ├── tasks/
│       ├── shifts/
│       ├── timeclock/
│       ├── catalog/
│       ├── comms/
│       ├── requests/
│       ├── stalls/
│       ├── i18n/
│       └── flags.ts
├── supabase/
│   ├── migrations/                numbered SQL files
│   ├── policies/                  RLS policies, paired with tables
│   └── cron/                      pg_cron job definitions
├── docs/
│   ├── README.md
│   ├── ARCHITECTURE.md            ← you are here
│   ├── SCHEMA.md
│   ├── INTEGRATION_TESTS.md
│   ├── TODOS.md
│   ├── .cursorrules.md
│   ├── slices/                    17 build slices
│   ├── domains/                   cross-cutting concern refs
│   └── runbooks/                  on-call docs
├── .github/workflows/             CI: lint, typecheck, test, RLS, e2e
├── turbo.json
├── package.json (pnpm workspaces)
└── tsconfig.base.json
```

## The three layers (never crossed)

```
PRESENTATION   ← change freely (V1.5+ redesign safe)
   React components in apps/web/app/{owner,client,worker}/, design tokens in packages/ui
LOGIC          ← stable, rarely touch
   Server actions in packages/lib, validation, RLS
DATA           ← never break
   Postgres schema, types, API contracts
```

UI changes touch only top layer. Schema changes go through migration + RLS test pairing.

## Key architectural choices

- **Multi-tenancy: Supabase RLS + `stable_id`.** Single Postgres, every row scoped. See `domains/rls.md`.
- **Money: integer cents.** See `domains/money.md`.
- **i18n: ICU MessageFormat, DE/FR/IT day one.** See `domains/i18n.md`.
- **Auth: Supabase Auth, magic link, role-picker post-login.** See `domains/auth.md`.
- **PDF: `swissqrbill` + react-pdf, SIX validator pre-launch.** See `domains/qr-bill.md`.
- **Cron: hybrid pg_cron + Vercel Cron.** pg_cron for DB-internal jobs (task materialization, subscription run, audit archive). Vercel Cron for external-API jobs (Bexio reconciliation, email digest). See per-slice docs.
- **Realtime: Supabase Realtime channels for worker today view + owner dashboard, with polling fallback on disconnect.** See `slices/13-worker-today.md`.
- **Observability: Sentry + Pino + UptimeRobot + Slack.** See `domains/observability.md`.

## Deployment

- **Vercel** hosts the single Next.js app with role-based routes (`/owner`, `/client`, `/worker`).
- **Supabase Pro** Frankfurt for Postgres + Auth + Storage + Realtime + pg_cron.
- **Domain**: `app.lafattoria.app` (single host). Worker PWA installs from `/worker/*` scope, appears as standalone "Stable Crew" app icon on phones.
- Branch deploys: every PR gets a preview URL on Vercel with a Supabase branch DB.

No Railway. No Python. V1.5 may add a separate ML service when a real use case lands, not before.

See `runbooks/deploy.md` for the deploy + rollback procedure.
