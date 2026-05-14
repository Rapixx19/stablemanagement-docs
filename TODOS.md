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
- [ ] **`pg_cron`, `pgcrypto`, `vault`, `vector`** extensions enabled in Supabase (`vector` is required by slice 01b for Reducto embeddings per `domains/document-search.md`)
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

## Pre-slice-00 API-key provisioning (Ferdinand owns, 5-day sprint)

Locked 2026-05-14 in `/plan-ceo-review` HOLD SCOPE session. Slice 00 PR blocks if these aren't done.

- [ ] **Reducto API** — sign up, get API key, **review and sign DPA** (EU residency + no model-training-on-customer-data). Store in `REDUCTO_API_KEY` Vercel env. Refs `domains/document-search.md` + `domains/document-ingestion.md`.
- [ ] **OpenAI** — org account + `text-embedding-3-small` API key. Store in `OPENAI_EMBEDDING_API_KEY`. Refs `domains/document-search.md`.
- [ ] **Anthropic** — API key for Claude Sonnet 4.6 (slice 05 catalog AI) + Haiku 4.5 (slice 01b document triage). Store in `ANTHROPIC_API_KEY`. Refs `domains/ai-extraction.md` + `domains/document-ingestion.md`.
- [ ] **Resend** — sender domain verified, inbound MX records on `inbox.stableplatform.ch` pointing at Resend inbound, webhook secret. Store as `RESEND_API_KEY` + `RESEND_INBOUND_WEBHOOK_SECRET`. Refs slice 00 D6 + slice 10 + `domains/document-ingestion.md`.
- [ ] **Bexio** — developer account + sandbox + OAuth app registration. Production credentials gated until La Fattoria onboards. Refs slice 09.

## Design review follow-ups (2026-05-14)

From `/plan-design-review` session 2026-05-14. Plan scored 8.5/10; held the May 7 baseline.

- [ ] **DES-LOCK-3 applied** (this session): `/owner/ops` desktop-only with sm-viewport redirect — slice 15 + DESIGN.md updated 2026-05-14.
- [ ] **Owner-ops summary row** — one-line summary at the top of the ops calendar grid: "This week: N workers, M coverage gaps, K late punches." Reduces wall-of-grid landing impact.
- [ ] **Journey storyboard `owner-document-arrives.md`** — 40-line walkthrough of vet emails PDF → owner confirms → searchable. The killer feature has no narrative.
- [ ] **Journey storyboard `client-books-service.md`** — 40-line walkthrough of client opens `/client/services` → picks service → picks day → submits → owner confirms → calendar updates. The "Anmelden" CTA is the most novel client interaction since May 7.
- [ ] **3 new primitives** to add to DESIGN.md OR `packages/ui/README.md` registry: `Sparkline` (SVG polyline variants for 4 KPI sparklines), `HorseGridItem` (photo grid square for dashboard active-horses strip), `AlertBand` (3 severities for dashboard alerts panel). Implied by the 2026-05-14 dashboard refresh but not yet specced.
- [ ] **Owner-mobile spec for `/owner/search` and `/owner/inbox/documents`** — DES-LOCK-2 says owner renders all viewports. Both lack a sm-viewport behavior; needed before slice 01b ships.
- [ ] **Sparkline empty/seed state** — when stable has < 14 days of history, hide sparkline (don't show a flat line at zero). Locks visual-integrity floor.
- [ ] **Horse-photo strip empty state** — when no horse has `is_primary` photo, show first 6 horses with letter avatars + onboarding prompt "Lade Fotos hoch, damit die Übersicht wärmer wird."
- [ ] **Document inbox quarantine modal threshold** — at N quarantined documents, switch from banner to blocking modal. Recommend N=20.
- [ ] **Owner-mobile bottom tab "Mehr" submenu spec** — bookable services, search, inbox, accounting, ops-redirect all live under Mehr. Spec the secondary nav order + counts.
- [ ] **State matrix copy** for `/owner/search`, `/owner/inbox/documents`, `/client/services` — loading / empty / error / partial in DE/FR/IT/EN. Lands in `domains/copy-library.md` per the existing CEO review TODO.

## CEO review follow-ups (2026-05-14)

From `/plan-ceo-review` HOLD SCOPE session.

- [x] **Slice 01 split** into 01a (people + horses + photos + profile UI, 5 dev-days) and 01b (unified documents + 5-path ingestion + Reducto + email_aliases + content search, 9 dev-days). Done 2026-05-14 in `/plan-ceo-review` session; README phase table updated; D17 logged in `DECISIONS.md`.
- [ ] EXPLAIN ANALYZE on polymorphic `documents` cross-tenant client-restricted SELECT pre-slice-01b merge. Polymorphic refs are a known query-plan footgun.
- [ ] Cross-language search recall test in `domains/document-search.md`: 50 fixture pairs (DE query / FR doc, IT / DE, EN / IT), assert top-5 recall > 80%.
- [ ] `BexioBookkeepingDataInconsistent` rescue path in slice 09 (Bexio reports paid but amount doesn't match ours = partial payment).
- [ ] Haiku triage disambiguation logic in `domains/document-ingestion.md`: when LLM proposes an entity_name that matches multiple records in the stable inventory, drop confidence to 0.5 and force owner pick.
- [ ] SLO for document ingestion latency (< 5 min p95 from upload to embedding populated) in `domains/metrics.md`.
- [ ] Daily AI cost summary email in `runbooks/oncall.md` — top-5 stables by Reducto + OpenAI + Anthropic spend yesterday.
- [ ] Caching strategy for owner-ops calendar arrival overlay (slice 15 #9) — N×M correlation under load needs precomputed view or short-TTL cache.
- [ ] Empty-state copy for `/client/services` when owner has toggled all `bookable_by_client` off (slice 11 / D16).
- [ ] `⌘K` global keyboard shortcut + slash-command spec for the owner search modal (slice 00 + `domains/document-search.md`).
- [ ] `packages/ui/README.md` primitive registry — discovery doc for the design system (sparkline SVG variants + photo-grid item are new additions from dashboard refresh, not in `DESIGN.md` yet).
- [ ] Rescue-path coverage lint promoted from slice 09 to a CI lint that scans every domain rescue table and asserts a paired unit test exists.
- [ ] EN translation of the 12 older DE-only mockups (newer ones already EN per session sweep). Naming: `mockups/*.html` → translation pass.
- [ ] `WebShareTargetUnsupportedOniOS` rescue path — fall back to "Tap Mehr → Dokument scannen" inline hint on iOS until iOS Safari ships share-target support.

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

## CEO + Eng + Design reviews 2026-05-07

Three reviews ran against the locked V1 plan today. Three review docs:
- `reviews/2026-05-07-ceo-review.md` — scope, premise, security/UX, slice-level spec gaps. 4 decisions: discriminated-union server-action contract; 30/90/15s Realtime; nFADP redact-PII-keep-row; slice 13 V1 IndexedDB offline queue.
- `reviews/2026-05-07-eng-review.md` — framework gotchas (cacheTag tenant scope, Fluid Compute module-state, cron poll body race, deep-join policy cost, SW unregister, JWT refresh), idempotency patterns, optimistic-lock convention, audit-log trigger, zod input validation, dashboard fan-out parallelization, partial index, query-count budget. 1 decision: cacheTag stable scope + CI invariant test.
- `reviews/2026-05-07-design-review.md` — 7-pass design audit. Initial 8/10 → 9/10 after fixes. 2 decisions locked: slice 08 invoice draft = two-pane collapsible-preview; **owner now renders all viewports (sm/md/lg/xl)** — V1 scope addition that affects 11 slices touching owner UI. Sweep: per-slice info hierarchy + per-feature state matrix DE/FR/IT + ConfirmDialog/DataTable/PreviewPane primitives + 3 user-journey storyboards + DE-canonical copy library + 4 new HTML mockups in existing style.

Combined: **17 critical silent-failure gaps** (12 CEO + 5 eng) + **owner-mobile scope expansion** (Design). Sweep into the lists above when starting each slice. Review docs supersede nothing — items above are still authoritative; this is the diff layer.

## Sweep policy

Quarterly sweep: anything in "Pre-slice-00" still open after week 1 starts is a blocker — escalate. Anything in "Per-slice spec follow-ups" still open at the end of its slice is a blocker for that slice's acceptance, full stop. "Post-pilot" and "Speculative" don't sweep — they get re-prioritized at the V1.0 retro.
