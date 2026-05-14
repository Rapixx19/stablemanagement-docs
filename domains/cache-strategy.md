# Domain: Cache Strategy

The multi-tenant cache safety boundary. Every `cacheTag` includes a stable scope. If you forget that, stable A's invoice list shows up for stable B's owner. Locked in `reviews/2026-05-07-eng-review.md` as E-ARCH-1.

## Mental model

Next.js 16 Cache Components let you mark functions with `use cache` and tag the result. Tagged results are cached across requests until something calls `revalidateTag` or `updateTag`. The cache is shared across all users of the app — so a cache tag without `stable_id` is a cross-tenant leak vector.

```ts
// WRONG — tenant leak
async function getInvoices() {
  'use cache'
  cacheTag('invoices')
  return db.select(...).from(invoices)
}

// RIGHT
async function getInvoices(stableId: string) {
  'use cache'
  cacheTag(`stable:${stableId}:invoices`)
  return db.select(...).from(invoices).where(eq(invoices.stableId, stableId))
}
```

## The invariant

Every `cacheTag` value MUST start with `stable:${stable_id}:`. Exceptions are global read-only data — `vat_rates`, `i18n:locales` — listed in the CI invariant test below.

## Per-surface cacheLife

| Surface | `cacheLife` | Why |
|---|---|---|
| Dashboard tiles | `cacheLife('hours')` | KPIs change slowly; invoice/payment mutations explicit-invalidate |
| Catalog page | `cacheLife('days')` | Owner edits trigger invalidation |
| Invoice list | `cacheLife('minutes')` | Bexio poll updates status every 30 min |
| Stall map | `cacheLife('days')` | Layout edits trigger invalidation |
| Tab calculation | `cacheLife('seconds')` | Hot path; line items mutate frequently — short TTL plus tag-based invalidation |

## Invalidation patterns

Every mutation that affects cached data invalidates the matching tag:

```ts
// after confirming an invoice
revalidateTag(`stable:${stableId}:invoices`)
revalidateTag(`stable:${stableId}:dashboard`)
```

Layer tags for fine-grained invalidation:

```ts
cacheTag(`stable:${stableId}:invoices`, `stable:${stableId}:invoice:${id}`)
// single-invoice change only invalidates that invoice
revalidateTag(`stable:${stableId}:invoice:${invoiceId}`)
// bulk change invalidates the whole list
revalidateTag(`stable:${stableId}:invoices`)
```

## CI invariant test

`apps/web/tests/integration/cache-tenancy.test.ts` greps every `cacheTag(...)` call in `apps/web/` + `packages/lib/` and asserts the first argument starts with `stable:` OR appears in the global-data whitelist. Fails CI on a new bare tag.

End-to-end assertion:

```ts
test('stable A confirms invoice does not affect stable B cached list', async () => {
  const a = await seed.stableWithInvoice('A')
  const b = await seed.stableWithInvoice('B')
  await renderDashboard(b.ownerCtx) // populates B's cache
  const bBefore = await readInvoiceList(b.ownerCtx)
  await confirmInvoice(a.ownerCtx, a.invoice.id)
  const bAfter = await readInvoiceList(b.ownerCtx)
  expect(bAfter).toEqual(bBefore) // no spurious cross-tenant invalidation
})
```

## Global-data whitelist

These tags don't need `stable:` prefix because the data is global:

- `vat_rates` — Swiss VAT codes, read-only, ships in migrations
- `i18n:locales:*` — translation strings, deployed with the app

Add to `packages/lib/cache/globalTags.ts` if a new one shows up. The CI test reads from there, not a hardcoded list.

## Common mistakes

- Forgetting the `stable:` prefix → cross-tenant cache leak (caught by CI)
- Using `revalidateTag('invoices')` without scope → blows every stable's cache
- Caching inside a server action without a tag → can't invalidate
- Tagging at too coarse a granularity → unnecessary revalidations on unrelated mutations
- Including PII in a tag (e.g., `stable:X:client:john@example.com`) → tags appear in logs

## When this doc updates

- New surface → add a row to `cacheLife` table
- New global read-only data → add the tag to the CI whitelist
- Cache strategy fundamentally changes (e.g., Next.js 17 deprecates Cache Components) → version this doc
