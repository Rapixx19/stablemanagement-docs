# Domain: Metrics & Owner Dashboard Panels

Operational metrics owners and engineers actually look at. Locked in `reviews/2026-05-07-ceo-review.md` as OBS-2.

## Two audiences, two surfaces

| Audience | Surface | Tool |
|---|---|---|
| Stable owner | `/owner/dashboard` tiles | Server-rendered, queried from primary DB |
| Sharad (oncall) | Sentry dashboards + Slack `#stable-prod` alerts | Sentry + UptimeRobot + Better Stack logs |

No separate analytics stack in V1. PostHog / Mixpanel / Grafana are V1.5 candidates.

## Owner metrics — dashboard tiles (slice 16)

| Tile | Metric | Source | Refresh |
|---|---|---|---|
| Umsatz (Monat) | Sum of `invoice_lines.line_total_incl_vat_cents` where `invoice.period` overlaps current month | `invoices` join `invoice_lines` | `cacheLife('hours')` + tag invalidation on confirm |
| Ausstehend | Sum where `status IN ('sent','overdue')` | `invoices` | `cacheLife('minutes')` |
| Belegung | `count(horses where status='active' and current assignment exists)` / `count(stalls where kind='stall' and active)` | `horse_assignments`, `stalls` | `cacheLife('hours')` |
| Aktive Klienten | `count(distinct person_id where active subscription exists)` | `client_subscriptions` | `cacheLife('hours')` |

Each renders with `xl 28/36` tabular numerals per `DESIGN.md`. **Not** carded — they're stats, not cards.

## Owner billing-insights surface (V1.1)

`/owner/billing/insights` is V1.1 scope. V1 ships the four dashboard tiles only. V1.1 candidates: revenue-by-service breakdown, average days-to-paid, top-N overdue clients.

## Engineering metrics — Sentry tags

Every server action wraps execution with Sentry context:

```ts
Sentry.withScope((scope) => {
  scope.setTag('stable_id', stableId)
  scope.setTag('user_id', userId)
  scope.setTag('role', currentRole)
  scope.setTag('slice', '08')
  scope.setTag('action', 'confirmInvoice')
})
```

Sentry custom dashboards (configured in slice 00, panel layouts captured in `runbooks/oncall.md`):

- Error rate by `slice` tag (PR fails if rate > baseline + 50%)
- p50/p95/p99 latency by route
- PDF generation duration + failure rate (slice 08 SLO)
- Bexio push DLQ row count (slice 09)
- Cron job success/failure (slices 04, 06, 09, 11, 15)
- Realtime channel reconnect rate (slice 13)

## SLO targets

| Metric | Target | Source of truth | Alert |
|---|---|---|---|
| Page p95 (dashboard) | < 1s | Sentry transactions | > 2s for 10 min → Slack `#stable-warn` |
| PDF gen p95 | < 3s | Sentry custom span | > 5s for 10 min → page Sharad |
| Bexio sync error rate | < 2% rolling 1h | `bexio_sync_log` count | > 10/h → page Sharad |
| Magic-link bounce rate | < 5% rolling 24h | Resend webhook (V1.1) | > 10% → Slack `#stable-warn` |
| Uptime | 99.5% rolling 30d | UptimeRobot `/api/healthz` | < 99.5% → Slack `#stable-prod` |
| Cron success | 100% | `cron_runs` table | Any failure → Slack `#stable-warn`; 3 consecutive → page |
| RLS test pass | 100% | CI | Any failure → blocks merge |

## What we do not track in V1

- User behavior analytics (clicks, scroll depth) — privacy + V1.5 candidate
- A/B tests
- Funnel analytics
- Per-feature usage rates

Reason: pilot is one stable. Aggregate metrics don't have signal at n=1.

## Where metrics live

- **Sentry** (Frankfurt EU instance): errors, performance, custom spans, alerts
- **Better Stack / Axiom**: Pino logs, 30-day retention, EU region
- **UptimeRobot**: single health check on `/api/healthz`
- **Slack**: `#stable-prod` (paging), `#stable-warn` (non-paging), `#stable-deploys` (CI/CD)

No Datadog, no Grafana, no New Relic in V1.

## Owner-visible alerts (not engineering)

The Sentry alert surface is for engineering. Owner-visible alerts live in-app:
- Document expiry → yellow badge on horse + dashboard alert tile (slice 01b + 16)
- Bexio sync issue count → dashboard tile / connection panel (slice 09)
- PDF generation failure → invoice list badge + sync-issues panel (slice 08)
- Off-site worker punch awaiting approval → workers panel (slice 14)

## Common mistakes

- Adding a metric without an alert threshold → noise without signal
- Tagging Sentry with a value derived from request body (potential PII) → use UUID-only fields (`stable_id`, `user_id`)
- Skipping the `slice` tag → can't filter errors by feature area
- Caching dashboard metrics too aggressively → owner doesn't see their own confirmed invoice for an hour

## When this doc updates

- New SLO → add row to the table + matching Sentry alert
- New tile on owner dashboard → add row to the dashboard tiles table
- V1.1 ships PostHog or Grafana → update "Where metrics live"
