# Runbook: Deploy and Rollback

How code goes from PR to production. How to undo it when something breaks.

## Topology

- 3 Vercel projects: `owner`, `client`, `worker` — each deploys independently
- 1 Railway project: FastAPI + cron workers
- 1 Supabase project: Postgres + Auth + Storage

## Branches

- `main` → production
- `preview/*` → Vercel preview URLs, full DB schema apply on Supabase branch
- Feature branches → PRs into `main`

No `develop` branch. Trunk-based development.

## CI on every PR

```
GitHub Actions:
  ├── lint (ESLint, Prettier)
  ├── typecheck (tsc --noEmit)
  ├── unit tests (vitest)
  ├── rls tests (vitest tagged @rls)
  ├── integration tests (vitest, Supabase Docker)
  ├── e2e tests (Playwright headless)
  └── secret scan (gitleaks)
```

PR cannot merge unless all green + 1 review approval.

## Deploy on merge to main

When PR merges:

1. GitHub Actions tags the commit with `prod-YYYYMMDD-HHMM-<sha>`
2. Vercel auto-deploys all 3 apps from `main`
3. Supabase migrations apply via `supabase db push --linked` action
4. Railway redeploys cron workers
5. Sentry release created with the tag
6. Slack `#stable-deploys` notified: "Deploy `<sha>` shipped"

## Migration order

Sequence matters: DB before app code.

If a migration is incompatible with the previous app version (e.g., dropping a column):
- Phase 1 deploy: app stops reading the column
- Wait > 24h
- Phase 2: deploy migration that drops it

This is the only reason a single PR splits across two deploys.

## Rollback

### Code rollback

1. Identify last known-good commit on `main`
2. Either: revert PR via GitHub UI (creates a revert PR, merge it) — preferred
3. Or: in Vercel, "Promote to Production" on a previous deploy — fastest
4. Migrations rolled forward only — see disaster-recovery.md if a migration broke prod

### Migration rollback

We don't `down`. If a migration broke prod:
1. Maintenance banner up
2. Restore DB to PITR point before migration applied
3. Write a *forward* migration that fixes the issue
4. Apply, verify, lift maintenance

This is rare; CI catches most migration issues in preview.

### Vercel rollback

1. Go to Vercel project → Deployments
2. Find the previous green deployment
3. "Promote to Production"
4. Cutover < 30s

For all 3 apps simultaneously, do it in this order: client → worker → owner. Owner last because it has the longest sessions.

## Hotfix procedure

Critical bug in prod:

1. Branch from `main`
2. Make minimal fix
3. Open PR labeled `hotfix`
4. Run CI — even hotfixes must pass tests
5. Merge with single review (not skipped)
6. Deploy auto-runs
7. Post-mortem within 48h

If the bug is severe (data loss, tenant leak): roll back first, fix later. See disaster-recovery.md.

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
- [ ] Manual demo to Ferdinand done if it's a slice exit?

## Maintenance, smoke, conn limits

- Non-emergency deploys: weekdays 10:00–18:00 CET. Avoid Friday afternoons. Schema changes: in-app banner 24h ahead.
- After every deploy `pnpm smoke` runs against prod URLs (login, dashboard, invoice fetch, healthz). Fail → rollback decision within 15 min.
- Supabase Pro: 60 connection limit. Use Pooler URL (`-pooler`) for Vercel serverless; Railway crons use direct URL with pool size cap. Misconfigure = 5xx storm.

## Env var rotation

Sensitive var rotation (see `domains/secrets.md`):
1. Add new value alongside old (e.g., `BEXIO_CLIENT_SECRET_NEW`)
2. Code reads new if present, falls back to old
3. Deploy
4. After 24h stable: remove old, deploy again

## Branch protections + release tagging

`main`: requires PR + 1 approval + CI green; no force push, no delete. Every prod deploy tagged `prod-YYYYMMDD-HHMM-<sha>`. Sentry maps errors to releases.

## Post-deploy verification

For high-risk slices (08 invoice PDF, 09 Bexio, 14 timeclock): run one cycle on La Fattoria's tenant in production, confirm with Ferdinand verbally, watch Sentry for 1 hour, stand down after 24h clean.

## Common deploy failures

- Migration timeout: long ALTER TABLE. Pre-check duration on preview.
- Vercel build OOM: bump build memory in `vercel.json`.
- Supabase pooler connection storm: scale Vercel functions, use pooler URL.
- Resend API key expired: rotate first, deploy second.
- Sentry source map upload fails: deploy succeeds, errors unmapped — fix + re-upload.
