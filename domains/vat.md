# Domain: Swiss VAT (MWST / TVA / IVA)

The exact tax rules. Wrong VAT codes create back-tax liability for our customers and reputational risk for us.

## Rates effective May 2026

| Code | Rate | Use |
|---|---|---|
| `STD` | 8.1% | Standard rate. Boarding (Pferdepension), lessons, training, transport, most services. |
| `RED` | 2.6% | Reduced rate. Feed, bedding sold separately. Books. Some food. |
| `LDG` | 3.8% | Special rate (Sondersatz Beherbergung). Hotel accommodation. **Not applicable to horses.** |
| `ZERO` | 0% | Exempt. Health-related services rendered by licensed vets to the horse owner directly (rare in stable context). |

## The 2026 hike (delayed)

A planned hike to 8.8% / 2.8% / 4.2% was scheduled, then delayed past 2026 — current expectation ~2028 pending parliament + popular vote. V1 ships with current rates; the `vat_rates` table supports `effective_from`/`effective_to` so adding new rows is trivial.

## Pferdepension is taxable

A common misconception is that boarding is agriculture-exempt. **It is not.** Pferdepension is paralandwirtschaftlich (parallel to agriculture) and taxed at standard rate 8.1%. Source: ESTV (Federal Tax Administration), Branchen-Info 02 Pferdesport.

## Mandatory invoice fields (Art. 26 MWSTG)

- Stable name and full address
- Stable's UID with **MWST suffix** (DE), **TVA suffix** (FR), **IVA suffix** (IT)
- Client name and address
- Invoice date
- Service period (`vom 01.05.2026 bis 31.05.2026`)
- Description of services
- Quantity, unit price, line total
- VAT rate per line
- Total excl. VAT, total VAT, total incl. VAT
- VAT breakdown table (per rate)

The "MWST/TVA/IVA" label per locale is non-negotiable. Never write "VAT" on a Swiss invoice.

## UID format

`CHE-XXX.XXX.XXX MWST` (DE)
`CHE-XXX.XXX.XXX TVA` (FR)
`CHE-XXX.XXX.XXX IVA` (IT)

Validation regex: `/^CHE-\d{3}\.\d{3}\.\d{3}( MWST| TVA| IVA)?$/`. Stored normalized without suffix; suffix added at render time per locale.

## Math — exact algorithm

For each line:
```
line_excl_vat_cents = round_half_to_even(qty * unit_price_cents)
line_vat_cents       = round_half_to_even(line_excl_vat_cents * rate)
line_incl_vat_cents  = line_excl_vat_cents + line_vat_cents
```

For the invoice:
```
total_excl_vat_cents = sum(line_excl_vat_cents)
total_vat_cents      = sum(line_vat_cents)        // not total_excl × rate
total_incl_vat_cents = sum(line_incl_vat_cents)
```

**Bankers' rounding** (round-half-to-even) reduces drift. JS doesn't have it built in; use `Math.round` with adjustment or import from `decimal.js`. See `packages/lib/billing/round.ts`.

## VAT registration threshold

Stables with annual revenue ≥ CHF 100k must register for VAT. La Fattoria with ~CHF 240k/yr is comfortably over. UI prompts during onboarding: "annual revenue?" → if < 100k, allow `vat_method = 'not_registered'` (V1.1, deferred).

## Saldosatz method (V1.1 — deferred)

Saldosatz lets small businesses declare VAT as a flat percentage of revenue. Simpler bookkeeping but lower input-VAT recovery. Not in V1. Stable settings has the toggle but it's locked to `effektiv` and shows "saldosatz coming in V1.1" tooltip.

## Saldosatz rates for stables (when V1.1 ships)

Pferdepension Saldosatz rate is approximately 6.0% (verify with Treuhänder). Two rates allowed per business; if a stable also sells feed retail, second rate applies.

## Test fixtures

A canonical mixed-rate invoice we test against in `packages/lib/billing/vat.test.ts`:

```
Lines:
- Full board × 1 month        @ CHF 850.00 STD
- Lessons × 4                 @ CHF 80.00  STD
- Hay supplement × 30 days    @ CHF 12.00  RED
- Transport × 2 trips         @ CHF 75.00  STD

Expected breakdown:
- STD subtotal excl: CHF 1320.00, VAT: CHF 106.92, incl: CHF 1426.92
- RED subtotal excl: CHF 360.00, VAT: CHF 9.36,    incl: CHF 369.36
- Total excl: CHF 1680.00, VAT: CHF 116.28, incl: CHF 1796.28
```

## Deferred to V1.1+

- Saldosatz method
- Reverse-charge for foreign clients (EU services)
- Multi-jurisdictional VAT (boarding for foreign clients)
- Credit notes with negative VAT
- Currency conversion for non-CHF invoices

## What never to do

- Use floating-point arithmetic on cent amounts. Always integer.
- Apply VAT after the total. Always per-line.
- Default boarding to reduced rate. It's standard rate.
- Show "VAT" label on a CH invoice. Use MWST/TVA/IVA.
- Hardcode 8.1%. Always look up by code in `vat_rates`.
