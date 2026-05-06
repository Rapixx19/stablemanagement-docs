# Slice 00 — Foundation

**Phase:** Foundation · **Estimate:** 8 dev-days · **Owner:** Sharad + Ferdinand

The slice that makes every other slice possible. Repo, infra, auth, RLS test infra, observability, audit log. Not glamorous, but if this is wrong we pay every week of the project.

## Goal

A two-engineer team can clone the repo, run `pnpm dev`, boot the single Next.js app, log in as a seeded user in any role, navigate to `/owner`, `/client`, or `/worker`, and have changes auto-reload. CI runs on every PR. Sentry catches errors. RLS isolation tests run green. Backup restore drill is rehearsed Day 1, not Week 16.

## What ships

1. **Monorepo bootstrap.** Turborepo, pnpm workspaces, TypeScript strict, ESLint, Prettier, Husky pre-commit.
2. **Single Next.js 14 app at `apps/web`** with three role-based route trees: `app/owner/*`, `app/client/*`, `app/worker/*`. Tailwind preset shared via `packages/ui`. PWA manifest scoped to `/worker/*`.
3. **Supabase project (Frankfurt, Pro tier).** Auth + Postgres + Storage + Realtime + pg_cron extension enabled. Frankfurt region for CH data residency.
4. **Connection Pooler enabled.** Transaction mode for app traffic (PgBouncer). Direct connection reserved for migrations + pg_cron jobs. Documented in `domains/secrets.md`.
5. **Per-role `statement_timeout = 30s`** on Supabase roles used by app traffic. Direct/migration role keeps the default.
6. **Schema bootstrap.** `stables`, `users`, `memberships`, `audit_log`, `feature_flags` tables. Migrations folder, `packages/db/schema.sql` canonical.
7. **`audit_log` partitioned by year from Day 1.** Postgres declarative range partitioning on `at`. Year partitions auto-created via pg_cron. See "Audit log partitioning" below.
8. **RLS scaffolding.** Helper SQL functions `auth.current_stable_ids()` (setof) and `auth.current_stable_ids_array()` (uuid[]) for hot-path policies. First policies on the 5 tables. Paired isolation tests.
9. **Auth flow.** Magic-link login → role picker → stable picker (if multi-membership). DE/FR/IT email templates. Pre-warmed SPF/DKIM/DMARC on `lafattoria.app` before pilot.
10. **Audit log trigger pattern.** Postgres trigger that writes to `audit_log` on `INSERT`/`UPDATE`/`DELETE` for whitelisted tables.
11. **Read-only DB role.** `stable_readonly` Postgres role created, granted SELECT on (empty) views. For V1.5 ML.
12. **Observability.** Sentry on the app. Structured logger (`pino` with `stable_id` context). UptimeRobot pinging `app.lafattoria.app/api/healthz`.
13. **CI pipeline.** GitHub Actions: install → lint → typecheck → unit → integration → RLS test → migration grep → schema dump diff → e2e (Playwright stub). Required for merge.
14. **Feature flag system.** `packages/lib/flags.ts` reading from Supabase `feature_flags` table, scoped per-stable.
15. **Recovery runbook.** `runbooks/auth-recovery.md` written. Owner-recovery procedure documented.
16. **Backup restore drill.** End of slice 00: deliberately drop a non-prod test stable, restore from PITR, verify data integrity, time the procedure. Result documented in `runbooks/disaster-recovery.md`.

## Schema diff

```sql
-- 001_init.sql
create table stables (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  uid_mwst text,
  default_locale text not null check (default_locale in ('de','fr','it')),
  vat_method text not null default 'effektiv' check (vat_method in ('effektiv','saldo')),
  created_at timestamptz not null default now()
);
create table memberships (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  stable_id uuid not null references stables(id),
  role text not null check (role in ('owner','manager','worker','client')),
  created_at timestamptz not null default now(),
  unique (user_id, stable_id, role)
);
-- audit_log is partitioned by year (range on `at`)
create table audit_log (
  id bigserial,
  stable_id uuid not null references stables(id),
  actor_user_id uuid references auth.users(id),
  entity text not null,
  entity_id uuid not null,
  action text not null check (action in ('insert','update','delete')),
  before_jsonb jsonb,
  after_jsonb jsonb,
  at timestamptz not null default now(),
  primary key (id, at)
) partition by range (at);

create table audit_log_2026 partition of audit_log
  for values from ('2026-01-01') to ('2027-01-01');
create table audit_log_2027 partition of audit_log
  for values from ('2027-01-01') to ('2028-01-01');
-- Subsequent year partitions auto-created by pg_cron job (see supabase/cron/audit-partition.sql)
create index on audit_log (stable_id, entity, entity_id);
create index on audit_log (stable_id, at desc);
create table feature_flags (
  stable_id uuid not null references stables(id),
  flag text not null,
  enabled boolean not null default false,
  primary key (stable_id, flag)
);
```

## RLS policies (initial)

See `domains/rls.md` for patterns. At minimum:

- `memberships`: a user can SELECT their own membership rows
- `stables`: a user can SELECT a stable they have a membership in
- `audit_log`: only `owner`/`manager` roles can SELECT, scoped to their stable
- `feature_flags`: only `owner` role can UPDATE

## Audit log partitioning

Yearly partitions on `at`. Reasoning: at 30 stables × ~1000 mutations/day × 5 years ≈ 55M rows. A single table becomes a query hot spot and blocks restore drills. Yearly partitions keep hot years (current + prior) in main DB, allow year 5+ to be archived to cold storage and dropped from the primary table without affecting compliance access (archive remains queryable via restore).

A pg_cron job runs December 1 each year, creating next year's partition. Defined in `supabase/cron/audit-partition.sql`.

Archival to cold storage is a V1.1+ runbook, not implemented in V1. Yearly partitioning at slice 00 is the cheap insurance that lets V1.1 archival be a 1-day job instead of a 5-day migration.

## Acceptance criteria

- [ ] `pnpm dev` starts the single web app locally with no errors
- [ ] `/owner`, `/client`, `/worker` route trees render placeholders without auth errors when seeded
- [ ] Magic-link email arrives in DE/FR/IT depending on user locale
- [ ] User with one membership skips the role picker
- [ ] User with two memberships sees the picker
- [ ] CI fails on a deliberate `any` introduced in a test PR
- [ ] CI fails on a deliberate cross-tenant SELECT in a test PR
- [ ] CI grep step fails when a multi-tenant table (matches `stable_id`) is added without `enable row level security` in same migration
- [ ] CI schema-dump-diff fails when migrations don't match regenerated `packages/db/schema.sql`
- [ ] Sentry captures a thrown error from a server action
- [ ] Audit log records a row in the correct year partition when a `stables` row is updated
- [ ] EXPLAIN ANALYZE on a canonical RLS-protected query (horse list joined to tab line aggregate, 30-stable seed) shows Index Scan on `(stable_id, ...)`, not Sort+Hash. Captured in slice 00 doc as the baseline.
- [ ] Backup restore drill rehearsed: PITR-restore a non-prod test stable, verify integrity, document elapsed time
- [ ] Connection Pooler enabled, app uses transaction-mode connection string, migrations + pg_cron use direct
- [ ] Recovery runbook documented and reviewed by Ferdinand
- [ ] Pre-warmed SPF/DKIM/DMARC for `lafattoria.app` verified via mail-tester.com (score 10/10)

## Acceptance integration test

`apps/owner/tests/integration/foundation.test.ts`

```ts
test('user with membership in stable A cannot read stable B', async () => {
  const a = await seed.stableWithOwner('A');
  const b = await seed.stableWithOwner('B');
  const result = await a.ownerCtx.from('stables').select().eq('id', b.stable.id);
  expect(result.data).toEqual([]);
});
```

## Out of scope

- Stripe (we collect SaaS subscription V1.1)
- Multi-stable consolidated dashboard (V2)
- Native mobile apps (V2)

## Files touched

- `package.json`, `turbo.json`, `pnpm-workspace.yaml`
- `apps/{owner,client,worker}/` — bootstrap
- `packages/ui/`, `packages/db/`, `packages/lib/`, `packages/i18n/`
- `supabase/migrations/001_init.sql`
- `supabase/policies/{stables,memberships,audit_log,feature_flags}.sql`
- `.github/workflows/ci.yml`
- `runbooks/auth-recovery.md`
