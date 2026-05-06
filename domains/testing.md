# Domain: Testing — Patterns and Examples

Contract is in `INTEGRATION_TESTS.md`. This file shows how to write each kind.

## Taxonomy

| Kind | Where | Purpose |
|---|---|---|
| Unit | next to source `*.test.ts` | Pure function correctness |
| RLS | `apps/*/tests/rls/` | Cross-tenant isolation per table |
| Integration | `apps/*/tests/integration/` | End-to-end via server actions |
| E2E | `apps/*/tests/e2e/` | Real browser via Playwright |

## Tooling

- `vitest` for unit/integration, `playwright` for E2E
- Supabase running locally via Docker, fresh DB per test file
- `msw` for HTTP mocking — never deep-mock our own modules
- `seed.ts` helpers, never raw SQL in tests

## Unit test pattern

```ts
// packages/lib/billing/vat.test.ts
import { describe, test, expect } from 'vitest';
import { computeLine } from './vat';

describe('computeLine', () => {
  test('STD rate at 8.1%', () => {
    expect(computeLine(2, 8500, 'STD')).toEqual({
      exclVatCents: 17000, vatCents: 1377, inclVatCents: 18377,
    });
  });
  test('RED rate at 2.6%', () => {
    expect(computeLine(30, 1200, 'RED').vatCents).toBe(936);
  });
  test('zero quantity', () => {
    expect(computeLine(0, 8500, 'STD').inclVatCents).toBe(0);
  });
});
```

Pure, fast, no DB.

## RLS test pattern

```ts
// apps/owner/tests/rls/horses.rls.test.ts
describe('horses RLS', () => {
  test('cross-tenant SELECT is empty', async () => {
    const a = await seed.stableWithHorse({ name: 'A' });
    const b = await seed.stableWithHorse({ name: 'B' });
    const r = await a.ctx.from('horses').select().eq('id', b.horse.id);
    expect(r.data).toEqual([]);
  });

  test('cross-tenant UPDATE silently fails', async () => {
    const a = await seed.stableWithHorse();
    const b = await seed.stableWithHorse();
    await a.ctx.from('horses').update({ name: 'hacked' }).eq('id', b.horse.id);
    const verify = await b.ctx.from('horses').select().eq('id', b.horse.id).single();
    expect(verify.data.name).not.toBe('hacked');
  });

  test('client cannot read horses they do not own', async () => {
    const { client, otherHorse } = await seed.stableWithTwoClients();
    const r = await client.ctx.from('horses').select().eq('id', otherHorse.id);
    expect(r.data).toEqual([]);
  });
});
```

Required for every multi-tenant table.

## Integration test pattern

```ts
// apps/owner/tests/integration/invoice-pdf.test.ts
test('confirm + generate produces SIX-valid QR-bill', async () => {
  const inv = await seed.draftInvoice({
    lines: [{ service: 'full-board', qty: 1, priceCents: 85000, vat: 'STD' }],
  });
  await confirmInvoice(seed.ownerCtx, { invoiceId: inv.id });
  const pdf = await generatePdf(inv.id);
  const valid = await sixValidator.validate(pdf);
  expect(valid.errors).toEqual([]);
  const updated = await seed.ownerCtx.from('invoices')
    .select().eq('id', inv.id).single();
  expect(updated.data.status).toBe('sent');
  expect(updated.data.qr_reference).toMatch(/^\d{27}$/);
});
```

## E2E test pattern (Playwright)

```ts
// apps/owner/tests/e2e/full-cycle.spec.ts
test('owner runs a full invoice cycle', async ({ page, signInAs }) => {
  await signInAs('owner@lafattoria.test', 'lafattoria');
  await page.goto('/billing/invoices?status=draft');
  await page.click('text=Sara Ramelli');
  await expect(page.locator("text=CHF 1'820.50")).toBeVisible();
  await page.click('text=Confirm and send');
  await expect(page.locator('text=Status: Sent')).toBeVisible({ timeout: 10000 });
});
```

E2E gates the slice's exit demo.

## Seed helpers and mocking

`tests/helpers/seed.ts` is the single entry point — hide the SQL, expose composable scenarios like `stableWithHorse()`, `stableWithTwoClients()`, `draftInvoice()`. External services: Bexio uses a local mock (`packages/test/bexio-mock`), Anthropic uses snapshot fixtures (live nightly), Resend via msw HTTP mock.

## What we don't mock

- The DB. Always real, always Supabase locally.
- The QR-bill library. Always real — correctness depends on it.
- The PDF generation. Always real.

## Performance targets

- Suite parallelizable: each tenant has its own UUIDs, no shared state
- Full suite < 12 min on CI, < 5 min locally
- Slow tests (PDF + SIX validator) move to nightly when they get heavy

## Flaky test policy

A flaky test means our code is non-deterministic — that's a bug. Either:
1. Fix the underlying nondeterminism
2. Pin the input (clock mock, sorted assertion)

We do not retry-on-fail. We do not skip flaky tests. We fix them.

## CI gating

PR merge requires lint + typecheck + unit + RLS + integration + E2E (Playwright headless on the slice being shipped). All green, no exceptions.

## Hard rules

- Tests use real DB, real server actions
- RLS tests pair every multi-tenant table
- No flaky tests; fix or delete
- No skipped tests in main
- Test data via `seed.ts`, never raw SQL or real PII
