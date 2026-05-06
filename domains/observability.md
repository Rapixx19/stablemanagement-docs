# Domain: Observability

When something breaks at 23:00 on a Sunday, this is what makes the difference between knowing in 5 minutes vs hearing from a furious owner on Monday.

## Three pillars

| Pillar | Tool | Purpose |
|---|---|---|
| Errors | Sentry | Stack traces, breadcrumbs, release tracking |
| Logs | Pino → Better Stack (or Axiom) | Structured queryable logs |
| Uptime | UptimeRobot | External pings every 5 min |

All three wired in slice 00. None optional.

## Sentry setup

- One Sentry project: `stablemanagement-`
- Environments: `development`, `preview`, `production`
- Releases tagged with git SHA
- Source maps uploaded on every Vercel deploy

Tags on every event:
- `stable_id` (when known)
- `user_id`
- `role`
- `route_tree` (`owner` / `client` / `worker`) — derived from the URL prefix on the single Next.js app

Filtered out:
- `ResizeObserver loop limit` (browser noise)
- 401/403 from auth (expected)
- AbortError (user navigation)

Alert rules:
- Any unhandled error in production → Slack `#stable-prod`
- Error rate > 1% over 5 min → page Sharad
- New error type (first seen) → email digest

## Structured logging

`pino` with a structured base context:

```ts
const log = baseLogger.child({
  stable_id,
  user_id,
  trace_id,
  route_tree: 'owner',
});
log.info({ action: 'invoice.confirm', invoice_id }, 'Invoice confirmed');
```

Always log with a stable event name (`action` field, dot-separated). Makes log queries possible.

Levels:
- `error`: bug; alerted
- `warn`: degraded but recoverable (Bexio retry, etc.)
- `info`: business events (invoice sent, user login)
- `debug`: dev only, off in prod

## What we log on every server action

```ts
log.info({ action: `${domain}.${verb}`, ...inputs }, 'started');
try {
  const result = await actionImpl();
  log.info({ action: `${domain}.${verb}`, duration_ms }, 'completed');
  return result;
} catch (err) {
  log.error({ action: `${domain}.${verb}`, err }, 'failed');
  Sentry.captureException(err);
  throw err;
}
```

Wrapper utility in `packages/lib/log/wrap.ts` does this for us.

## What we never log

- Passwords (we don't have any)
- OAuth tokens (encrypted at rest, never logged)
- Raw GPS coordinates (per `domains/labour-law.md`)
- Full document content (only file paths and sizes)
- PII outside what's already in `audit_log`

## UptimeRobot

One monitor on the single app host:
- `https://app.lafattoria.app/api/healthz`

Pings every 5 min. Alert on 2 consecutive failures → Slack + email Ferdinand.

`/api/healthz` checks: app responds, DB query works (one trivial `SELECT 1` through the pooler), Storage reachable, pg_cron `cron.job_run_details` shows last successful run within expected window. Returns 200 with `{ ok: true, ts, checks: {...} }` or 503 with the failing component named.

We deliberately keep one healthz, not three. Three subdomains gave us "is the worker up?" / "is the client up?" — but with one app on one host, that question collapses into one. If a route tree has a regression, Sentry catches it, not UptimeRobot. UptimeRobot's job is "is the host serving requests."

## Performance monitoring

Sentry Performance for transaction tracing:

- All server actions
- All page loads
- Notable slow targets: PDF generation (< 3s), dashboard loads (< 1s)

Slack alert when p95 > 2× target for any tracked transaction over 1 hour.

## Audit log vs application log

Two distinct things, often confused:

- **`audit_log` (Postgres table)**: business mutations. Compliance, customer-visible. Never deleted.
- **Application log (Pino)**: technical detail. Operational. 30-day retention.

A user editing a horse: writes to BOTH. Audit log captures who/what/when. App log captures duration, query count, etc.

## PDF generation + Bexio telemetry

PDF gen (slice 08 KPI): every gen logs `{ action, invoice_id, size_bytes, duration_ms, six_validator_passed }`. Better Stack dashboard tracks rolling p95 / p99 / fail rate.

Bexio: every push + poll logs structured event. Dashboard tracks push success > 99%, poll success > 99.5%, avg push time (alert > 5s), OAuth refresh failures (alert immediately).

## Slack + incident response

Channels: `#stable-prod` (P0/P1), `#stable-warn` (non-critical), `#stable-deploys` (releases).

Incident flow: alert → on-call ack within 15 min business hours / 60 min otherwise → Linear incident with severity → investigate via Sentry + logs → comms to affected stables via in-app banner if > 30 min → post-mortem in `docs/post-mortems/YYYY-MM-DD-slug.md`. V1 has 1 on-call (Sharad), rotation in V1.1.

## Privacy

Frankfurt-only logging. All providers (Sentry / Better Stack / UptimeRobot) configured for EU data residency, nFADP-compatible. PII in logs OK for ops debugging; redacted in shared dashboards.

## Hard rules

- Every server action logs start + end + outcome
- Every error captured in Sentry with `stable_id` tag
- No `console.log`; use `log.info` etc.
- No swallowed errors — re-throw or capture explicitly
- Healthz returns within 5s, even with slow DB
