# CEO Review — 2026-05-07

Pre-coding review of locked V1 plan. Mode: **HOLD SCOPE**. Driver: `/plan-ceo-review` against `DESIGN.md` + `ARCHITECTURE.md` + slices/00–17 + `DECISIONS.md` (D1–D12 locked same day) + `TODOS.md` + `INTEGRATION_TESTS.md` + `.cursorrules.md`.

Scope confirmed by Ferdinand: Stableplatform = owner / client / worker management + communication platform. Sentavita ECG biometrics is a separate project, not part of V1.

## Locked decisions (this review)

- **CQ-1 — Server-action return contract**: discriminated union `{ ok: true, data } | { ok: false, error: <typed> }`. Pick once, hold for the project. Cleaner Sentry context, no `try/catch` sprawl, easier RLS-error → user-facing-error mapping. Implementer doc lives in slice 00.
- **ARCH-2 — Realtime timing (slice 13)**: 30s heartbeat / 90s missed-heartbeat fallback / 15s polling interval when degraded. Confirmed.
- **SEC-4 — nFADP audit-log retention**: on right-to-erasure request, redact PII fields in `before_jsonb` / `after_jsonb` (replace with `[redacted-{user_id_hash}]`), keep audit row. Reference Swiss Code of Obligations 10y accounting retention as the legal-grounds-balance. Add `domains/data-retention.md`.
- **EDGE-6 — Offline task queue (slice 13)**: ship in V1. IndexedDB queues `{task_id, completed_at, version}` (no PII), reconciles on reconnect. ~1 dev-day added to slice 13 estimate. Update slice 13 spec.

## Pre-slice-00 doc additions

- `domains/cache-strategy.md` — per-surface `cacheLife`, `cacheTag` scoping with `stable_id` for multi-tenant cache safety (ARCH-1)
- `domains/security-headers.md` — CSP, HSTS preload, X-Frame, Permissions-Policy baseline (SEC-2)
- `domains/data-retention.md` — audit-log redaction policy (SEC-4 above) + Swiss CO 10y rule (SEC-4)
- `domains/metrics.md` — operational metrics + dashboard panels owner cares about (OBS-2)
- `CONTRIBUTING.md` or `docs/onboarding.md` — first-week walk-through for new engineers (LT-4)
- Verify `domains/observability.md` covers Sentry tags (`stable_id`, `user_id`, `role`, `slice`), Pino fields, Slack channels per severity (OBS-1)
- Verify `domains/ai-extraction.md` covers system-prompt boundary, JSON-schema validation, owner-confirms-each-row, max-rows cap (SEC-1)
- Vault-vs-Vercel-env crypto-key single source of truth documented (SEC-7)
- `runbooks/cron-secret-rotation.md` (SEC-3)
- `domains/auth.md` — manager-role permissions OR drop manager from V1 enum (ARCH-5)
- README.md — ASCII slice-dependency graph (ARCH-3); phase table to include slice 17 inventory or move slice 17 to V1.1
- Slice 00 spec — lock the 8 Step-0E hour-1-to-6 decisions (raw-SQL escape-hatch boundary, audit-log trigger table list, Bexio idempotency key shape, advisory-lock timeout pattern, visual regression tooling, i18n review cadence, statement_timeout per-route, server-action contract above)

## Slice 00 — CI + scaffolding additions

- `packages/lib/_template.action.ts` — discriminated-union return + `withStableContext` Sentry wrapper + Pino entry/exit log
- `supabase/migrations/_template.sql` — paired RLS policy + RLS test stub
- `apps/web/tests/integration/_template.test.ts` — matches `INTEGRATION_TESTS.md` contract
- `cron_runs(name, started_at, completed_at, status, error)` table — every cron writes a row (OBS-4)
- Alert when `audit_log_default` partition has rows (pg_cron partition-create job failed) (OBS-3)
- Per-IP + per-email rate limit on `/api/auth/*` magic-link (LT-3)
- Service-worker cache key includes `stable_id`: `worker-${stable_id}-v1` (SEC-5, EDGE-7)
- axe-core CI step on key flows (login, dashboard, invoice view, task complete) (DES-5)
- `packages/i18n/missing.test.ts` — fails on key drift between DE/FR/IT (TEST-5)
- AI-slop greppable lint — fail on `bg-*-100 rounded-full` wrapping `<Icon`, fail on forbidden phrases incl. `!` in production string files (DES-3)
- DE/FR/IT date-format unit test asserting `DD.MM.YYYY` in all three (DES-6)
- 1 dev-day budgeted for "Cache Components shake-down" — verify canonical RLS+cache query hits perf target (ARCH-4)

## Slice-specific additions

- **Slice 03**: cross-stable EXPLAIN ANALYZE for `events × event_horses × event_people` query (PERF-2)
- **Slice 04**: SAVEPOINT-per-rrule in materialisation; `UNIQUE (stable_id, source_event_id, due_date)` for idempotent re-runs (Section 2 critical gap)
- **Slice 05**: AI catalog eval suite — 20-PDF golden set DE/FR/IT, regression-on-prompt/model-change, ≥95% accuracy threshold (TEST-6)
- **Slice 06**: EXPLAIN ANALYZE for `getTab` baseline committed to slice doc (PERF-4)
- **Slice 08**: empty-period = skip (no zero-CHF invoice) (EDGE-1); `vat_rate_at_creation` frozen at line creation in `tab_lines` and `invoice_lines` (EDGE-2); cross-stable QR-ref uniqueness in mod-10 sweep (EDGE-3); 50-line invoice visual regression DE/FR/IT (EDGE-4); Storage-write-before-status-update (EDGE-5); `SwissQrSchemaError` + `SixValidatorUnreachable` rescue paths
- **Slice 09**: `AdvisoryLockTimeout` + `BexioLineValidationError` rescue paths; idempotency key shape = per-invoice; advisory-lock blocking with 5s timeout; cross-train freelancer on integration design (LT-2)
- **Slice 12**: PWA install instructions DE/FR/IT
- **Slice 13**: IndexedDB offline tap queue (V1 scope addition per EDGE-6); `RealtimeAuthError` non-silent rescue path; first-claim-wins via `version` column on `tasks`; 5 sample worker-task strings reviewed by native DE speaker (DES-2)
- **Slice 16**: per-slice "confidence test" line ("ship at 2am Friday") (TEST-1); cold-start variant of dashboard load test (PERF-1); k6 / Playwright nightly load runner at 30-stable scale (TEST-3); `runbooks/oncall.md` (OBS-6); post-deploy verification checklist (DEPLOY-5); rollback flowchart in `runbooks/deploy.md` (DEPLOY-3); weekly PII-stripped staging snapshot of pilot DB (DEPLOY-4)

## Process / conventions

- E2E flake budget: ≤1 % with auto-retry 2× (TEST-2)
- RLS-test naming convention added to `INTEGRATION_TESTS.md` (TEST-4)
- DE-canonical copy library — 10 examples per category (invoice subject, magic-link subject, error toast, empty state, push notification), DE/FR/IT (DES-1)
- Empty-state SVG format: ≤5 KB each, single-color `--ink-700` (DES-4)
- Migration safety threshold: ≥5 stables → split add-not-null into add-nullable → backfill → enforce (DEPLOY-1)
- Vercel Cron pinned in actual UTC (e.g., `02:00 UTC`) with documented +1h DST drift (DEPLOY-6)
- Feature-flag seed for La Fattoria stable in slice 00 (DEPLOY-2)
- Connection-pool budget documented in `domains/secrets.md` or new doc (PERF-3)
- Per-tree bundle-size budget enforced in CI (PERF-5 confirmation)

## V1.1 additions to `domains/v1-deferred.md`

- Email bounce ingestion (Resend webhook → `users.email_bounced_at`, owner-visible)
- Cross-stable consolidated invoice generation — formal "won't do in V1" entry (currently silent)
- (Optional) If V1 rate-limit on magic-link is deferred, list it here with residual-risk note

## Failure-mode registry — critical gaps requiring fix before pilot

12 entries flagged as silent-failure-class:

1. Magic-link hard bounce → silent failure (slice 00)
2. Magic-link rate-limit / email bombing → no protection (slice 00)
3. Bexio advisory-lock timeout → indeterminate stuck push (slice 09)
4. Bexio line-VAT-mismatch test gap → DLQ may not surface (slice 09)
5. swissqrbill version drift → 500 (slice 08)
6. 50-line invoice overflow → truncated PDF (slice 08)
7. Realtime channel auth failure → silent stale UI (slice 13)
8. Worker offline tap → silent failure (slice 13) — addressed by EDGE-6 V1 add
9. Task-materialisation `RruleParseError` on one event → whole batch silently fails (slice 04)
10. Task-materialisation duplicate run → duplicate task rows (slice 04)
11. `audit_log_default` partition fills → no alert (slice 00)
12. SW cache key collision after stable switch → wrong-tenant data leak (slice 12)

## Premise questions (open, not blocking)

Items I surfaced and the team chose to keep within the locked plan. Not action items, but worth ambient awareness during pilot:

- **P2** — single-design-partner risk. La Fattoria is the only pilot. By week 22 generalizability is unknown. Walter outreach is in pre-slice-00 TODOs but not milestone-tracked.
- **P3** — DE-canonical for an Italian-named pilot. La Fattoria likely speaks IT primarily; DE→IT translation is the highest-risk locale. Mitigation: native-IT speaker review at slice 16.
- **P4** — V1's named differentiator (TAMV journal) is V1.1. Pilot wins on relationship; stable #2 needs differentiation, which lands in V1.1 only if V1 ships on time.
- **P5** — Next.js 16 + Cache Components is bleeding-edge. Mitigation = ARCH-4 1-day shake-down in slice 00.
- **P6** — Bexio-only billing. Stables on Banana / KLARA / CashCtrl can't pilot until V1.1 camt.054 import.
- **P7** — Worker email-only login. Casual-labour barn workers without email cannot be onboarded in V1 (D11 documented).
- **P8** — Pilot exit criterion #4 ("hasn't opened Excel/WhatsApp") is self-report only. Optional objective complement: "100 % of invoices in last 4 weeks generated via app."

## Diagrams produced (in review thread)

- System architecture (single Next.js app + role trees + Vercel Cron + Supabase + Resend + Bexio)
- User flow (magic link → role picker → density-appropriate dashboard)

## Stale diagram audit

ARCHITECTURE.md ASCII diagram (single-app, three-tree) is current as of today's commit `30eea68` and the locked DECISIONS.md. No stale diagrams in the doc set.

## Reviewer

`/plan-ceo-review` (Claude Opus 4.7) on 2026-05-07. Outside voice (codex / Claude subagent) skipped per Ferdinand. Four live decisions made in-review (CQ-1, ARCH-2, SEC-4, EDGE-6).

The eng-review companion document (`2026-05-07-eng-review.md`, same directory) covers framework-specific gotchas, code-quality patterns, performance, and worktree parallelization at execution depth.
