# Runbook: Deploy and Rollback

How code goes from PR to production. How to undo it when something breaks.

## Topology (per `DECISIONS.md` D1)

- **1 Vercel project** for the single Next.js 16 app at `apps/web/` — three role-based route trees on one host
- **1 Supabase project** (Frankfurt, Pro) — Postgres + Auth + Storage + Realtime + pg_cron
- **No Railway, no Python, no FastAPI.** Vercel Cron handles all external-API and Node-only scheduled work; pg_cron handles pure-SQL scheduled work.

## Branches

- `main` → production
- `preview/*` → Vercel preview URLs, full DB schema apply on Supabase branch
- Feature branches → PRs into `main`

No `develop`. Trunk-based.

## CI on every PR

```
GitHub Actions:
  ├── lint (ESLint, Prettier)
  ├── typecheck (tsc --noEmit)
  ├── unit tests (vitest)
  ├── rls tests (vitest tagged @rls)
  ├── integration tests (vitest, Supabase Docker template DB)
  ├── e2e tests (Playwright headless)
  ├── migration grep (every multi-tenant table has `enable row level security`)
  ├── schema-dump-diff (`packages/db/schema.sql` matches migrations)
  └── secret scan (gitleaks)
```

PR cannot merge unless all green + 1 review approval.

## Deploy on merge to main

When PR merges:

1. GitHub Actions tags the commit with `prod-YYYYMMDD-HHMM-<sha>`
2. **Vercel** auto-deploys the single app from `main`
3. Supabase migrations apply via `supabase db push --linked` action
4. Sentry release created with the tag
5. Slack `#stable-deploys` notified: "Deploy `<sha>` shipped"

## Migration order

DB before app code. If a migration is incompatible with the previous app version (e.g., dropping a column):

- Phase 1: deploy app code that no longer reads the column
- Wait > 24h
- Phase 2: deploy migration that drops it

This is the only reason a single PR splits across two deploys.

## Rollback

### Code rollback (fastest)

1. Vercel → Deployments → find previous green deploy → "Promote to Production"
2. Cutover < 30s

### Code rollback (clean)

1. Identify last known-good commit on `main`
2. Revert PR via GitHub UI (creates a revert PR, merge it)
3. Vercel auto-deploys the revert

### Migration rollback

We don't `down`. If a migration broke prod:

1. Maintenance banner up
2. Restore DB to PITR point before migration applied
3. Write a *forward* migration that fixes the issue
4. Apply, verify, lift maintenance

Rare; CI catches most migration issues in preview. See `runbooks/disaster-recovery.md`.

## Hotfix procedure

Critical bug in prod:

1. Branch from `main`
2. Make minimal fix
3. Open PR labeled `hotfix`
4. Run CI — even hotfixes must pass tests
5. Merge with single review (not skipped)
6. Deploy auto-runs
7. Post-mortem within 48h

If the bug is severe (data loss, tenant leak): roll back first, fix later. See `runbooks/disaster-recovery.md`.

## Pre-deploy checklist (release manager)

Before approving a PR with risky scope:

- [ ] Migrations included if schema changed?
- [ ] RLS policy paired for new tables?
- [ ] RLS test paired?
- [ ] i18n strings present in DE / FR / IT?
- [ ] Performance impact considered (queries, indexes)?
- [ ] Secrets not in code?
- [ ] Audit log captures any new mutations?
- [ ] Acceptance criteria from slice met?
- [ ] Bundle budget check passed if `app/worker/*` touched (`< 150KB gzipped`)?
- [ ] Manual demo to Ferdinand done if it's a slice exit?

## Maintenance, smoke, conn limits

- Non-emergency deploys: weekdays 10:00–18:00 CET. Avoid Friday afternoons. Schema changes: in-app banner 24h ahead.
- After every deploy `pnpm smoke` runs against prod URLs (login, dashboard, invoice fetch, healthz). Fail → rollback decision within 15 min.
- Supabase Pro: 60 connection limit. **App traffic uses the transaction-mode Pooler URL (`-pooler`)**, migrations + pg_cron use the direct URL with bounded pool. Misconfigure = 5xx storm. See `domains/secrets.md`.

## Env var rotation

Sensitive var rotation (see `domains/secrets.md`):

1. Add new value alongside old (e.g., `BEXIO_CLIENT_SECRET_NEW`)
2. Code reads new if present, falls back to old
3. Deploy
4. After 24h stable: remove old, deploy again

For `BEXIO_TOKEN_ENCRYPTION_KEY`: rotation requires re-encrypting every row in `bexio_connections` — Procedure 1 in `runbooks/bexio-token-recovery.md`.

## Branch protections + release tagging

`main`: requires PR + 1 approval + CI green; no force push, no delete. Every prod deploy tagged `prod-YYYYMMDD-HHMM-<sha>`. Sentry maps errors to releases.

## Post-deploy verification

For high-risk slices (08 invoice PDF, 09 Bexio, 14 timeclock, 17 inventory): run one cycle on La Fattoria's tenant in production, confirm with Ferdinand verbally, watch Sentry for 1 hour, stand down after 24h clean.

## Common deploy failures

- Migration timeout: long ALTER TABLE → pre-check duration on preview
- Vercel build OOM: bump build memory in `vercel.ts`
- Supabase pooler connection storm: scale Vercel functions, confirm pooler URL is in use
- Resend API key expired: rotate first, deploy second
- Sentry source map upload fails: deploy succeeds but errors unmapped — fix + re-upload
- Cron handler returns 401: `CRON_SECRET` mismatch between Vercel env and `vercel.ts` config
