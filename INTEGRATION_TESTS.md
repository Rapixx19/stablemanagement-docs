# Integration Tests — Contract

Every slice ships with at least one integration test that proves the end-to-end flow works. This document is the contract for what "integration test" means in this repo.

## Definition

An integration test in this repo:
1. Boots a fresh Supabase test instance (Docker)
2. Runs all migrations
3. Seeds two tenants (`stable_a`, `stable_b`) with users in each role (owner, client, worker)
4. Exercises a real flow through server actions (not direct DB calls)
5. Asserts both the positive path and the cross-tenant isolation path
6. Tears down the instance

Unit tests are not integration tests. Mocking the DB defeats the purpose.

## Where they live

```
apps/owner/tests/integration/
apps/client/tests/integration/
apps/worker/tests/integration/
apps/owner/tests/rls/        ← isolation tests, paired per-table
```

## RLS isolation suite (mandatory, runs on every PR)

For every table with `stable_id`, the RLS suite must include:

- **Read isolation**: User in stable A cannot SELECT a row from stable B
- **Write isolation**: User in stable A cannot INSERT/UPDATE/DELETE a row in stable B
- **Join isolation**: User in stable A querying a join table (e.g., `event_horses`) cannot pull a row that bridges to stable B
- **Storage isolation**: User in stable A cannot download a file uploaded by stable B (Supabase Storage RLS)

CI must fail if any RLS test fails. There are no flaky tests in this suite. If a test is flaky, the policy is wrong, not the test.

## Per-slice required test

Each slice file lists a "Acceptance integration test" section. That test must:

- Run as a real user (use Supabase Auth, not service role)
- Use server actions, not direct SQL
- Assert the row exists in the DB after the action
- Assert any side effect (email queued, Bexio call made, audit log row written)

## Examples (trim — full examples in `domains/testing.md`)

**Slice 06 — Recurring subscription billing**:
```ts
test('owner adds recurring service to client; tab shows next month', async () => {
  const { ownerCtx, clientId } = await seed.stableWithClient();
  await addSubscription(ownerCtx, { clientId, serviceId: 'full-board', autoBill: true });
  const tab = await getTab(ownerCtx, { clientId, period: nextMonth() });
  expect(tab.lines).toContainEqual(expect.objectContaining({ kind: 'recurring' }));
});
```

**Slice 08 — QR-bill PDF**:
```ts
test('confirmed invoice produces a SIX-valid QR-bill PDF', async () => {
  const inv = await seed.confirmedInvoice({ totalChf: '1820.50' });
  const pdfBuf = await generateInvoicePdf(inv.id);
  const valid = await sixValidator.validate(pdfBuf); // calls paymentstandards.ch sandbox
  expect(valid.errors).toEqual([]);
  expect(inv.qrReference).toMatch(/^\d{27}$/);
});
```

## What we don't test

- Visual regressions of mockups (use Storybook + Chromatic if needed later)
- Third-party services we don't control (Bexio's own QR rendering, Vercel deploys)
- Hot reload / dev server behaviour

## Test data hygiene

- Never seed real Swiss UID numbers, real client names, or real horse UELNs
- Use the `seed/` helpers, never raw inserts
- Tests must run in any order — no shared state across files
- Each test gets its own tenant (cheap with `pnpm test --parallel`)

## CI gating

PR cannot merge to `main` without:
- `pnpm test` green (unit + integration)
- `pnpm test:rls` green
- `pnpm test:e2e` green for the slice being shipped (Playwright, runs the user flow in a real browser)
- `pnpm lint && pnpm typecheck` green

## When tests get slow

Target: full suite under 5 minutes locally, under 12 minutes on CI. If we exceed:
1. Parallelize per-tenant
2. Move expensive tests (PDF gen, Bexio sandbox) to a nightly suite
3. Never disable a test to make CI fast

## Adding a new test

1. Find the slice file
2. Add the test name to its acceptance criteria
3. Write the test
4. Verify it fails before your code change
5. Make it pass

That sequence — failing test first — is non-negotiable for bug fixes.
