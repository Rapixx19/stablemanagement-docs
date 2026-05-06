# Slice 07 — Swiss VAT Engine

**Phase:** Billing · **Estimate:** 3 dev-days · **Owner:** Sharad backend

The VAT computation that every invoice depends on. Swiss-correct: 8.1% standard, 2.6% reduced, 3.8% accommodation, 0% exempt. Mandatory invoice fields per Art. 26 MWSTG. Saldosatz support deferred to V1.1 (effektive method only in V1).

## Goal

Given a tab of line items with VAT codes, produce a correct VAT breakdown for an invoice. Mandatory fields for the invoice template (UID-MWST with suffix, MWST/TVA/IVA label per language, totals per rate). Audit-traceable.

## What ships

### Backend

1. **`packages/lib/billing/vat.ts`** with pure functions:
   - `computeLine(qty, unitPriceCents, vatCode)` → `{ exclVat, vat, inclVat }`
   - `computeBreakdown(lines)` → `{ totals, byRate: [{ code, rate, subtotalExcl, vatAmount, subtotalIncl }] }`
   - `formatChf(cents, locale)` → string with Swiss separators (`1'820.50` DE, `1 820,50` FR)
2. **`packages/lib/billing/uid.ts`** with:
   - `formatUid('CHE-115.295.448')` → `CHE-115.295.448 MWST` (DE), `... TVA` (FR), `... IVA` (IT)
   - `validateUid(string)` → boolean
3. **VAT line storage on invoices** — invoice has columns for `total_excl_vat_cents`, `total_vat_cents`, `total_incl_vat_cents`. Per-rate breakdown stored as JSONB `vat_breakdown_jsonb` (array of `{ code, rate, subtotal, vat }`).
4. **Effektive method only.** Saldosatz toggle in stable settings is read but ignored in V1 — emits a warning. `domains/vat.md` documents the saldosatz-V1.1 plan.

## Settings UI (Owner)

5. **Stable VAT settings** at `/settings/vat`. Edit UID-MWST. Pick method (effektive locked in V1). Default VAT codes per service category. Preview of how an invoice line renders.

## Constraints

- Money math uses **integer cents** only. Never JS floats.
- Rounding: each line rounded to whole cents using bankers' rounding (`round-half-to-even`). Totals are sums of rounded lines, not re-rounded.
- VAT rate effective dates respected: a line with `occurred_on = 2025-12-31` uses the rate active that day.
- Invoice-level VAT amount = sum of per-line VAT (not totals × rate).
- See `domains/vat.md` for the exact algorithm and rounding rules.

## Mandatory invoice fields (per Art. 26 MWSTG)

Surfaced on the invoice template (slice 08):

- Stable name and address
- Client name and address
- Invoice number (unique per stable)
- Invoice date
- Service period
- Description of each line item
- Quantity, unit price, line total
- VAT rate per line
- **UID-MWST suffix** (DE: MWST, FR: TVA, IT: IVA — never EN "VAT")
- VAT breakdown table (per rate)
- Total excl VAT, total VAT, total incl VAT

## Acceptance criteria

- [ ] `computeLine(2, 8500, 'STD')` → `{ exclVat: 17000, vat: 1377, inclVat: 18377 }` (8.1% on 170)
- [ ] Breakdown of 5 lines at mixed VAT codes correctly aggregates per-rate
- [ ] `formatChf(182050, 'de')` returns `1'820.50`; for `'fr'` returns `1 820,50`; for `'it'` returns `1'820.50`
- [ ] `formatUid` returns DE/FR/IT suffix per locale
- [ ] Invoice with no VAT (all ZERO code) shows breakdown with rate=0 row
- [ ] Bankers' rounding test: `(0.075 * 100)` → 8 (not 7), but `(0.085 * 100)` → 8 (not 9) — verifies half-to-even
- [ ] `validateUid('CHE-115.295.448')` → true; `validateUid('CHE-XXX')` → false
- [ ] Effective-date test: line on 2024-12-31 uses old rate, on 2024-01-02 uses new rate

## Acceptance integration test

`packages/lib/billing/vat.test.ts`

```ts
test('mixed-rate breakdown matches Treuhänder-validated fixture', async () => {
  const lines = [
    { qty: 1, unitPriceCents: 85000, vatCode: 'STD' },
    { qty: 30, unitPriceCents: 1200, vatCode: 'RED' },
    { qty: 4, unitPriceCents: 8000, vatCode: 'STD' },
  ];
  const breakdown = computeBreakdown(lines);
  expect(breakdown.totals.totalInclVatCents).toBe(126_536);
  expect(breakdown.byRate.find(b => b.code === 'STD').vatAmount).toBe(9_369);
  expect(breakdown.byRate.find(b => b.code === 'RED').vatAmount).toBe(936);
});
```

## Risks

- **Wrong VAT default on a service.** Mitigated by slice 05 review screen requiring explicit confirmation.
- **2026 VAT rate change.** Currently delayed to ~2028 per Federal Council. When it lands, add a new row in `vat_rates` with new effective_from; algorithm picks correct rate automatically.
- **Treuhänder review.** Skip for La Fattoria pilot (internal use, family operation). Mandatory before first external paying stable.

## Out of scope

- Saldosatz method (V1.1)
- Reverse-charge for foreign clients (V1.1)
- VAT recovery on imports (V2)
- Multi-jurisdictional VAT (EU clients) (V2)
