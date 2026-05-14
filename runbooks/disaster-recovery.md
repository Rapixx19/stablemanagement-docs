# Runbook: Disaster Recovery

Catastrophe scenarios and the procedure for each. Tested at least once before pilot launch.

## Backup posture

- **Postgres**: Supabase point-in-time recovery (PITR) on Pro tier. 7-day window.
- **Storage**: Supabase Storage Frankfurt-region replication. No cross-region.
- **Secrets**: 1Password backup, recovery codes printed and held by Ferdinand.
- **Code**: Git, GitHub primary, weekly mirror to GitLab as backup remote.

## Restore drill (slice 16 — runs before pilot)

Quarterly thereafter. Procedure:

1. In Supabase dashboard, pick a PITR point ~24h ago
2. Restore to a fresh project (`stable-restore-test`)
3. Run integrity SQL: row counts, total invoice amounts, audit_log range
4. Verify Storage paths still resolve (signed URL works)
5. Document the time-to-restore in the runbook log
6. Tear down the test project

If any step fails, halt pilot and fix.

## Scenario: Postgres data loss / corruption

### Symptoms

- Sentry errors on critical queries
- "Row not found" on a known-existing entity
- Customer reports data missing

### Procedure

1. **Stop writes immediately**: enable maintenance banner via `MAINTENANCE_MODE=true` Vercel env var on the single app. Existing reads continue, writes return 503.
2. **Diagnose scope**: which table? since when? which `stable_id`s affected?
3. **Decide PITR target**: latest known-good time, before corruption began.
4. **Communicate to affected stables**: in-app banner + email. "We are restoring data; service will resume in <X> minutes."
5. **Execute PITR**: Supabase dashboard or CLI. Restore goes to a new project.
6. **Verify**: integrity SQL on the restored project.
7. **Cut over**: update Vercel env vars (`SUPABASE_DB_POOLER_URL`, `SUPABASE_DB_DIRECT_URL`, `NEXT_PUBLIC_SUPABASE_URL`) to new project. Redeploy.
8. **Re-enable writes**: clear maintenance mode.
9. **Post-mortem**: required, in `docs/post-mortems/YYYY-MM-DD-...md`.

Total time target: < 4 hours for full restore.

## Scenario: Storage data loss

### Symptoms

- Signed URLs return 404
- Customer reports invoice PDF missing

### Procedure

1. Determine scope: single file, single bucket, or all buckets
2. For single file: regenerate if possible (invoices can be regenerated from invoice_lines + RLS on the same period)
3. For widespread loss: contact Supabase support (we have an SLA on Pro tier). Frankfurt-region Storage replication should cover most cases.
4. Customer-supplied uploads (horse documents, photos): can't be regenerated. Communicate honestly. Owner may have the original PDF/photo.

Mitigation: nightly Storage manifest snapshot to S3 Glacier (V1.1) — we know what should exist even if files vanish.

## Scenario: Vercel outage

### Symptoms

- App returns 503 / DNS errors on `app.lafattoria.app`
- Vercel status page shows incident

### Procedure

1. Check Vercel status. If global outage, wait — we have no failover in V1.
2. Communicate via email to affected stables
3. Vercel typically resolves within < 1 hour
4. No data loss; just downtime

V1.1 fallback plan: deploy parallel infrastructure on Cloudflare Pages with manual DNS cutover. Cost vs benefit reviewed quarterly.

## Scenario: Supabase outage

### Symptoms

- All apps fail at DB queries
- Supabase status page shows incident

### Procedure

1. Maintenance banner up (`MAINTENANCE_MODE=true` Vercel env)
2. Wait — no failover in V1 since auth + DB + storage all on Supabase
3. Communicate to stables

V1.1 plan: separate auth from DB so partial outages can be handled. Out of V1 scope.

## Scenario: Bexio outage

### Procedure

1. Push attempts queue and retry with exponential backoff
2. Reconciliation poll skips and retries next cycle
3. Owner alerted in app: "Bexio sync delayed — your data is safe in stablemanagement-, will sync when Bexio recovers."
4. No customer-facing degradation beyond delayed sync

## Scenario: Resend / Anthropic / secret leak

- **Resend outage**: queue accumulates; cron retries with backoff. After 30 min: switch to Postmark via env var flip. Drain backlog. Switch back when recovered.
- **Anthropic API outage**: catalog import fails gracefully ("AI unavailable, use paste mode"). No queueing. Doesn't block operational flow.
- **Secret leak**: see `domains/secrets.md` § Secret leakage incident response.

## Scenario: tenant leak (RLS bug)

**P0 — worst possible.** Customer A sees Customer B's data.

1. Take app offline (`MAINTENANCE_MODE=true`)
2. Identify the leak: query, table, data leaked, to whom
3. Fix the RLS policy
4. Add the missing test
5. Audit-log search: find every query that exploited the bug
6. Notify affected stables (nFADP requires within 72h)
7. Restore service
8. Post-mortem with full timeline, public if material

## Comms, on-call, drills

- **Comms**: pre-written DE/FR/IT outage templates in `packages/i18n/email-templates/incidents/`. V1.1: `status.lafattoria.app`. V1: in-app banner + email.
- **On-call**: V1 = Sharad primary, Ferdinand secondary, both by phone. PagerDuty in V1.1. Business hours 09:00–22:00 CET; outside: best-effort.
- **After incident**: post-mortem in `docs/post-mortems/`, action items in Linear, regression tests added, affected customers contacted.
- **Drill schedule**: quarterly PITR, bi-annually app failover walkthrough, annually tenant-leak tabletop.
