# Domain: Optimistic Locking

Every status-changing table has `version int not null default 0`. Conditional UPDATE bumps it. Concurrent updates → loser gets `OptimisticLockError`. Locked in `reviews/2026-05-07-eng-review.md` as E-CQ-2.

## Why

Two server actions on the same row racing → one's update silently overwrites the other. The cost class:

- Owner confirms an invoice; mid-confirm, the same owner clicks Edit in a second tab and saves a line change → invoice ends up confirmed with stale lines
- Worker A marks a task done; worker B marks the same task done at the same second → who's the `completed_by_user_id`?
- Two cron runs both regenerate a subscription tab line → double-billing

RLS doesn't catch this. Triggers don't catch this. Only an explicit `WHERE version = $expected` does.

## Tables that use it

| Table | Slice | Why |
|---|---|---|
| `tasks` | 04 | Worker first-claim-wins on unassigned tasks; status mutations from multiple sources |
| `services` | 05 | Concurrent price edits trigger the slice 06 cascade — only one cascade should fire |
| `client_subscriptions` | 06 | Bulk-assign + price-change cascade can target the same row |
| `invoices` | 08 | Concurrent draft edits + manual mark-paid + Bexio poll status update |
| `bexio_connections` | 09 | OAuth refresh + connection panel state updates |
| `availability_slots` | 11 | Hold/confirm/decline + capacity counters under contention |
| `service_requests` | 11 | Owner confirm + client withdraw race |
| `shifts` | 15 | Owner UPDATE + sick-conversion via unavailability approval |
| `unavailability` | 15 | Worker amend (forbidden post-submit) + owner approve |

## The pattern

```ts
type Result<T> = { ok: true, data: T } | { ok: false, error: OptimisticLockError | OtherError }

async function confirmInvoice(invoiceId: string, expectedVersion: number): Promise<Result<Invoice>> {
  const [row] = await db.update(invoices)
    .set({
      status: 'sent',
      sentAt: new Date(),
      version: sql`${invoices.version} + 1`,
    })
    .where(and(
      eq(invoices.id, invoiceId),
      eq(invoices.version, expectedVersion),  // the locking clause
    ))
    .returning()

  if (!row) {
    return { ok: false, error: new OptimisticLockError('invoice', invoiceId) }
  }
  return { ok: true, data: row }
}
```

The conditional UPDATE returns zero rows if the version didn't match — the row was modified by someone else since we read it. The discriminated union (`reviews/2026-05-07-ceo-review.md` CQ-1) carries this back to the UI cleanly.

## UX

When `OptimisticLockError` lands client-side:

- Re-fetch the row server-side
- Show a `Toast` (not modal, not blocking) with `--ochre-600` accent: "Daten wurden zwischenzeitlich geändert — bitte erneut versuchen."
- Re-render the form with the fresh row's values
- Owner reapplies their edits and resubmits

Never auto-retry the action. The whole point is that a concurrent mutation invalidated the assumptions.

## Reading the version client-side

Every server-action input that mutates a versioned table accepts `expectedVersion`. The form reads `row.version` on initial fetch, holds it in state, sends it back on submit. Server compares.

```ts
// component
const [invoice, setInvoice] = useState(initial)
// ...on save
const result = await confirmInvoice(invoice.id, invoice.version)
if (!result.ok && result.error instanceof OptimisticLockError) {
  const fresh = await refetchInvoice(invoice.id)
  setInvoice(fresh)
  toast.warn(t('toast.dataChanged'))
  return
}
```

## What it does NOT do

- Doesn't prevent two appenders inserting duplicate rows — use unique indices for that (`domains/idempotency.md` Pattern 1)
- Doesn't replace RLS — different concern (tenancy vs. concurrency)
- Doesn't protect DELETE — version isn't checked there; audit_log is the trail

## Cron writes and version

Crons that mutate versioned rows bump `version` like any other writer. If two cron runs race (rare, advisory locks prevent this — see `domains/idempotency.md` Pattern 2), the loser sees the bumped version and skips.

## Common mistakes

- Adding `version int` but not using it in the WHERE clause → cosmetic
- Bumping `version` from the client side → owner can spoof and overwrite concurrent edits
- Forgetting to bump `version` in the SET clause → next concurrent UPDATE still races
- Auto-retrying on `OptimisticLockError` → defeats the purpose; user should re-decide

## When this doc updates

- New status-changing table → add a row to the table above
- New conditional-update variant (e.g., `WHERE version = $expected AND status = $expected`) → document as variant
