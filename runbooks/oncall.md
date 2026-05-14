# Runbook: On-Call

Single on-call rotation in V1: Sharad. Ferdinand backs up for product/scope decisions but can't resolve infrastructure issues directly. V1.1 expands to a two-person rotation per `domains/v1-deferred.md`.

## Severity definitions

- **P0 — incident**: production down, data loss, wrong invoice amount, auth break, cross-tenant data leak. Page immediately. Target acknowledge < 15 min.
- **P1 — degraded**: a slice's core flow fails (PDF gen, Bexio sync, comms) but other surfaces work. Target acknowledge < 1h business hours, < 4h off-hours.
- **P2 — minor**: alert but not user-blocking (single DLQ entry, slow tile, expiring cert). Address next business day.

## Receive a page

Slack `#stable-prod` posts come from Sentry + UptimeRobot. Severity inferred from tag:
- `severity=fatal` or P0 alert rule → P0
- `severity=error` from oncall rule → P1
- Everything else → P2

Sharad uses Slack DND for off-hours; P0 paging bypasses DND. UptimeRobot pings phone via SMS for P0 only.

## Initial triage (first 5 minutes)

1. **Acknowledge** in Slack: "On it, looking now."
2. Open **Sentry**: filter last 1h, group by `stable_id` and `slice` — is one stable affected or all?
3. Open **UptimeRobot**: is `/api/healthz` up?
4. Open **Better Stack**: tail Pino logs for the affected `slice` / `stable_id`.
5. Open Supabase status + Vercel status pages — is upstream degraded?

If stable-specific (one `stable_id`), it's P1 at most. If all stables, escalate to P0.

## Common incident playbooks

### PDF generation failing (slice 08)

Symptom: Sentry alert "PDF gen failure rate > 2% rolling 1h."

1. Check Sentry `error_class` distribution — `SwissQrBillVersionDrift` (deploy issue), `PdfGenerationOOM` (memory), or `StorageWriteFailure` (Supabase Storage)?
2. **`SwissQrBillVersionDrift`** → identify the offending deploy; roll back via `runbooks/deploy.md`.
3. **`PdfGenerationOOM`** → check Vercel function memory; persistent → bump memory tier or split PDF worker (V1.1 plan in `domains/scaling.md`).
4. **`StorageWriteFailure`** → check Supabase Storage status; if Supabase-side, post to `#stable-warn` and wait. Retry queue is idempotent.

Affected invoices appear in `/billing/invoices/sync-issues`. Owner manually retries when root cause clears.

### Bexio sync degraded (slice 09)

Symptom: Sentry alert "Bexio sync error rate > 2% rolling 1h" or DLQ > 10/h.

1. Sentry → filter by `error_class`. One stable's connection broken (`BexioRefreshTokenRace`) or all (`BexioApiUnreachable`)?
2. **One stable**: check `bexio_connections.last_error_class`. If `BexioRefreshTokenRace`, run `runbooks/bexio-token-recovery.md`.
3. **All stables**: check Bexio API status. If down, post to `#stable-warn`; auto-retry handles reconciliation on recovery.

### Magic-link bounce spike (slice 00 / 10)

Symptom: bounce rate > 10% rolling 24h.

1. Resend dashboard: which recipient domains? Is one (e.g., `gmx.ch`) failing systematically?
2. Re-run mail-tester: did SPF/DKIM/DMARC drift?
3. Domain-specific → likely shared-pool IP reputation; post to `#stable-warn`, open Resend ticket.

### DB connection exhaustion

Symptom: Sentry "pooler timeout" errors, app slow across surfaces.

1. Supabase dashboard → active connections vs. pool size.
2. Sentry for a runaway query (locked table, missing index).
3. Transient → redeploy hanging Vercel functions. Persistent → scale pool via Supabase plan upgrade (`domains/scaling.md`).

### Cron job missed (slices 04 / 06 / 09 / 11)

Symptom: `cron_runs` table has no row for the expected interval, or `#stable-warn` alert.

1. Check Vercel cron dashboard: did the job fire?
2. Not fired → check `vercel.ts` schedule for drift (CI snapshot test should have caught this).
3. Fired but errored → check Sentry by `slice` tag. Resolve the underlying error; cron is idempotent so manual re-trigger via `/api/cron/{name}` (with `CRON_SECRET`) is safe.

## Escalation

- **Product/scope decisions** → Ferdinand
- **Bexio account issues** → support@bexio.com; thread to Sharad
- **Supabase escalation** → Pro tier ticket; response within 1 business day
- **Legal / nFADP** → Ferdinand routes to legal counsel

## Post-incident

For P0 + P1:
1. Within 24h: write `runbooks/incidents/YYYY-MM-DD-short-name.md` with timeline, root cause, what we tried, what worked, what to add to monitoring.
2. If a new playbook entry would help: add it to this runbook.
3. Pilot exit-criterion #3 (0 P0 bugs for 30 days) resets on any P0 — track in `TODOS.md`.

## Sentry dashboards (configured in slice 00)

Panels Sharad checks during triage:
- Error rate by `slice` tag
- p95 latency by route
- PDF gen duration + failure rate
- Bexio DLQ count (1h, 24h windows)
- Cron success/failure (last 7 days)
- Magic-link bounce rate (24h)

## Sister runbooks

- `runbooks/auth-recovery.md` — owner can't log in / magic-link broken
- `runbooks/bexio-token-recovery.md` — Bexio reauthorization + token rotation
- `runbooks/disaster-recovery.md` — PITR restore drill
- `runbooks/deploy.md` — redeploy + rollback path
- `runbooks/cron-secret-rotation.md` — rotate `CRON_SECRET`

## When this runbook updates

- After a P0 / P1 → add a playbook section if existing ones don't cover it
- V1.1 ships two-person rotation → add rotation schedule
- New slice ships with its own alert class → add a playbook section
