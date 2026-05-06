# TODOS

Living checklist of pre-slice-00 prerequisites, per-slice spec follow-ups, and post-pilot work. Anything that touches multiple slices or doesn't have a natural home in a single slice file lives here.

Convention: tick the box, leave the line, add the date and commit SHA in parens. Don't delete done items inside V1 — they're the audit trail. Sweep at V1.0 GA.

## Pre-slice-00 (must do before week 1 starts)

Infra and trust setup that has long lead times. Cheaper to do once now than to chase in week 8.

- [ ] **Domain** `lafattoria.app` registered, DNSSEC on, registrar lock on
- [ ] **Single app host** `app.lafattoria.app` Vercel project provisioned (Frankfurt edge region preferred)
- [ ] **SPF/DKIM/DMARC pre-warming** for `lafattoria.app` started 4 weeks before first transactional email — required to land in `gmx.ch`, `bluewin.ch`, `hispeed.ch` inboxes (the Swiss long tail). Verified via mail-tester.com to a 10/10 score before pilot
- [ ] **Resend** sender domain verified, dedicated IP if budget allows (otherwise shared pool — accept the risk for V1)
- [ ] **Supabase Pro project** in Frankfurt provisioned, daily backup verified, PITR window = 7 days enabled
- [ ] **Connection Pooler** enabled, transaction-mode connection string captured for app traffic, direct connection captured for migrations + pg_cron
- [ ] **`statement_timeout = 30s`** set on the role used by app traffic; default kept on direct/migration role
- [ ] **`pg_cron`, `pgcrypto`, `vault`** extensions enabled in Supabase
- [ ] **Supabase Vault** seeded with `bexio_token_encryption_key` mirror of the Vercel env var
- [ ] **GitHub Actions runners** authorized; required-status-checks set on `main` (lint, typecheck, unit, integration, RLS, migration grep, schema-dump-diff, e2e stub)
- [ ] **`gitleaks`** pre-commit hook installed locally + in CI
- [ ] **`swissqrbill`** pinned to an exact version in `package.json` (no `^`); regression run scheduled before any upgrade
- [ ] **`@react-pdf/renderer`** pinned to an exact version
- [ ] **`rrule.js`** pinned to an exact version (used on both server and client for slice 03 + slice 04)
- [ ] **SIX QR-bill validator** (`paymentstandards.ch`) credentials acquired, CI integration scripted
- [ ] **Sentry org + project** created, DSN minted, source-map upload token in CI
- [ ] **Pino → Better Stack (or Axiom)** sink wired, EU region, 30-day retention
- [ ] **UptimeRobot** account, single monitor on `app.lafattoria.app/api/healthz`
- [ ] **Slack workspace** + `#stable-prod`, `#stable-warn`, `#stable-deploys` channels, Sentry + UptimeRobot integrations attached
- [ ] **1Password** vault for the team, all provider creds stored there
- [ ] **`runbooks/disaster-recovery.md`** rehearsed once (the slice 00 backup-restore drill — see acceptance criteria)
- [ ] **`runbooks/auth-recovery.md`** written and reviewed by Ferdinand
- [ ] **`runbooks/bexio-token-recovery.md`** written and reviewed by Sharad
- [ ] **`runbooks/deploy.md`** documents the "redeploy + rollback" path
- [ ] **CLAUDE.md / .cursorrules.md** updated to reflect locked V1 scope (single app, no Python, CH-only, Walter network for design partners)
- [ ] **Walter intro list** drafted: 5–10 candidate boarding stables for design-partner outreach via Walter
- [ ] **La Fattoria pilot agreement** drafted (free V1 access in exchange for weekly feedback, named in marketing once they consent)
- [ ] **Privacy notice** for nFADP — DE/FR/IT — covers what we collect, where it lives (Frankfurt), how to delete

## Per-slice spec follow-ups

Items that surfaced in CEO/eng review and need to land alongside the slice they belong to. Spec already updated to reference these; the work itself is captured here so we don't lose it.

### Slice 00 — Foundation
- [ ] Migration grep CI step: any migration touching a table with `stable_id` must include `enable row level security` in the same file
- [ ] Schema-dump-diff CI step: `packages/db/schema.sql` must match what migrations produce
- [ ] EXPLAIN ANALYZE baseline on canonical RLS-protected query (horse list × tab line aggregate, 30-stable seed); committed to slice 00 doc as the perf baseline

### Slice 03 — Schedule
- [ ] DST fixtures committed to `apps/web/tests/fixtures/recurrence/`: spring-forward gap (2026-03-29 02:30 Zurich), fall-back ambiguity (2026-10-25 02:30), leap-day yearly, full-year cross-DST series

### Slice 04 — Tasks
- [ ] `supabase/cron/task-materialisation.sql` committed; pg_cron schedule pinned to `0 3 * * *` in `Europe/Zurich`; CI verifies the schedule string in `cron.job`

### Slice 06 — Subscriptions and Tabs
- [ ] `supabase/cron/subscription-run.sql` committed; pg_cron schedule pinned to `0 2 1 * *` in `Europe/Zurich`
- [ ] Proration helper unit tests for: 28-day Feb, 29-day leap Feb, suspended mid-period, ends on last day of period
- [ ] `getTab(personId, period)` <500ms p95 with 30k tab_lines seed

### Slice 08 — Invoice PDF
- [ ] Visual regression goldens for DE/FR/IT in `tests/fixtures/invoices/golden/`
- [ ] Mod-10 collision sweep test (100k synthetic invoice IDs)
- [ ] Cold-start budget CI job (15-min idle Vercel preview → first PDF < 5s)
- [ ] Concurrent-gen load test (100 in parallel, p95 < 4s)
- [ ] Manual QR-scan test against UBS, ZKB, PostFinance, Raiffeisen apps before pilot

### Slice 09 — Bexio
- [ ] Vercel Cron schedule for `/api/cron/bexio-poll` set to every 30 min, `CRON_SECRET` wired
- [ ] Postgres advisory-lock implementation for OAuth refresh (key = `stable_id`)
- [ ] Pact-style cassette of Bexio API responses checked into `tests/fixtures/bexio/`
- [ ] DLQ row-count alert in Sentry (> 10 in 1h pages Sharad)

### Slice 12 — Worker shell
- [ ] `/worker/manifest.webmanifest` committed; service worker scoped to `/worker/*` only
- [ ] iOS install instructions screen — DE/FR/IT copy + Share-icon arrow asset
- [ ] BrowserStack matrix run on iOS 16/17/18 + Android Chrome before pilot
- [ ] `apps/web/tests/perf/worker-budget.test.ts` enforces <150KB gzip + Lighthouse mobile ≥90 + LCP <2.5s on slow 4G

### Slice 13 — Worker today
- [ ] Hybrid Realtime+polling reducer implementation; presence heartbeat at 30s, fallback at 90s missed
- [ ] Reconcile-on-reconnect path tested with simulated 60s outage during owner edits
- [ ] Optimistic version field on `tasks` for first-claim-wins race resolution

### Design system (calibrated to DESIGN.md)
- [ ] `packages/ui/tokens.ts` exports the full CSS-variable palette from `DESIGN.md` (ink scale, terracotta/clay, moss, ochre, rust, paper/canvas, ink-overlay) and the spacing/radius/motion tokens; no raw hex anywhere else in `packages/ui` or `apps/web`
- [ ] `packages/ui/tokens.test.ts` contrast script verifies every text-on-surface combo ≥ AA, AAA on tabular numerics
- [ ] `next/font` self-hosts `Inter Tight` (variable, weights 400/500/600), `Source Serif 4`, `JetBrains Mono`; no Google Fonts CDN at runtime
- [ ] Primitives in `packages/ui/primitives/` (Button 36/44/52, Field, Card hairline-not-shadow, Modal/Drawer, TabBar, Pill, EmptyState, Toast) match DESIGN.md exactly; visual snapshot tests block drift
- [ ] Calendar event color check (slice 03 acceptance): only the three locked hues render — `--terracotta-600` / `--ink-500` / `--moss-600` dashed; CI snapshot + grep ban on `blue|purple|gradient` in client-facing CSS
- [ ] Empty-state copy DE/FR/IT for every list/table region (per DESIGN.md state matrix), reviewed by native speakers before pilot
- [ ] Two pen-line emptiness illustrations ("no horses yet", "no events today") — drawn by Ferdinand, committed to `packages/ui/illustrations/`
- [ ] iconography lint: `lucide-react` only, 1.5px stroke, no icon-in-colored-circle decoration (greppable failure on `bg-*-100 rounded-full` wrapping an `<Icon`)
- [ ] `prefers-reduced-motion` reducer cuts all motions to 100ms opacity crossfade; verified with axe + manual VoiceOver pass
- [ ] Forbidden-phrase lint: copy linter blocks `Welcome to`, `Unlock the power of`, `Your all-in-one`, `Let's get started`, and any `!` in production string files (DE/FR/IT)
- [x] Hero mockups for owner dashboard, client home, worker today (`mockups/owner-dashboard.html`, `mockups/client-home.html`, `mockups/worker-today.html`) — generated 2026-05-06, calibrated to `DESIGN.md` via shared `mockups/_tokens.css`. Pre-pilot review with Walter / La Fattoria still pending

## Post-pilot V1.0 → V1.1

Things deliberately deferred to keep V1 small. Don't pull these forward without a real pilot signal.

- [ ] **Stripe SaaS subscription** (we're free to La Fattoria during pilot; V1.1 turns on paid tier for stable #2 onwards)
- [ ] **Photo attachment on task completion** (slice 13 captured in V1; V1.1 uploads + thumbnails + Storage policies)
- [ ] **Credit notes** for invoices (slice 08 only ships `cancelled` status in V1)
- [ ] **eBill dispatch** via Bexio (V1 emails the QR-bill PDF; V1.1 adds eBill push)
- [ ] **CSV export** for non-Bexio bookkeeping (Banana / KLARA / CashCtrl)
- [ ] **iCal export** for clients (workers get it in slice 13)
- [ ] **Audit log archival** — year 5+ partitions exported to cold storage and dropped from primary
- [ ] **Discounts and credits** on invoices
- [ ] **Inbound calendar sync** (read clients' Google Calendar to suggest lesson times)
- [ ] **Web push notifications** (V1 uses banners; V1.1 adds permission-flow + push)
- [ ] **Per-stable on-call rotation** (V1 has 1 on-call: Sharad)

## Speculative V1.5 / V2

Don't promise these. Only revisit if a real customer signal lands.

- [ ] ML layer — predictive task ordering, dropped-customer prediction, etc. Decision gate: does an actual stable owner ask for it twice unprompted? If yes, then evaluate Python service / Edge Function / Supabase + pgvector
- [ ] Sentavita biometrics integration (horse health monitoring)
- [ ] TAMV journal (vet record formal export)
- [ ] Inventory module (feed bag tracking, etc.)
- [ ] Reporting / BI dashboard
- [ ] Identitas API (CH horse-passport authority)
- [ ] Abacus integration (for stables > 50 horses)
- [ ] FNCH integration (Swiss equestrian federation)
- [ ] WhatsApp Business API for client comms (currently we replace WhatsApp; V2 may bridge)
- [ ] Native iOS / Android wrapper via Capacitor
- [ ] Multi-stable consolidated dashboard for chains
- [ ] DACH or international expansion (V1 is CH-only)
- [ ] GPS-verified task completion
- [ ] Voice-note task completion with STT
- [ ] Biometric login on worker app

## Sweep policy

Quarterly sweep: anything in "Pre-slice-00" still open after week 1 starts is a blocker — escalate. Anything in "Per-slice spec follow-ups" still open at the end of its slice is a blocker for that slice's acceptance, full stop. "Post-pilot" and "Speculative" don't sweep — they get re-prioritized at the V1.0 retro.
