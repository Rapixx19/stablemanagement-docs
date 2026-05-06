# Domain: Money

How we handle money values across the codebase. Wrong floats lose customers; the rules below are not negotiable.

## The fundamental rule

**Money is integer cents. Always.** Never `850.00` — always `85000`. Never `1820.50` — always `182050`. JS floats produce silent errors (`0.1 + 0.2 === 0.30000000000000004`). Not allowed near an invoice.

## Type discipline

```ts
type Cents = number & { __brand: 'Cents' };
function asCents(n: number): Cents {
  if (!Number.isInteger(n)) throw new Error('non-integer cents');
  return n as Cents;
}
```

Database:

```sql
amount_cents bigint not null check (amount_cents >= 0)
```

`bigint` because some accumulations (annual revenue cents) exceed 32-bit safely.

## Where floats are allowed

- VAT rates: `numeric(5,3)` like `0.081`. Apply via `roundHalfToEven(cents * rate)`.
- Display formatting: as floats inside `Intl.NumberFormat`. Final character output, not used in further math.
- Quantities: `numeric(10,2)` like `2.5 lessons`. Multiplied with cents → result rounded back to cents.

## Rounding rules

**Bankers' rounding** (round-half-to-even, IEEE 754 default for some operations) per Swiss accounting convention:

- `0.5` → `0`
- `1.5` → `2`
- `2.5` → `2`
- `3.5` → `4`

JavaScript doesn't have it natively. Implementation in `packages/lib/billing/round.ts`:

```ts
export function roundHalfToEven(n: number): number {
  const r = Math.round(n);
  if (Math.abs(n - Math.floor(n) - 0.5) < Number.EPSILON) {
    // exactly half → round to even
    return r % 2 === 0 ? r : r - 1;
  }
  return r;
}
```

Use this for line VAT computations:

```ts
const vatCents = roundHalfToEven(linePriceCents * rate);
```

## Decimal library

For decimal-ish ops (proration, discounts in V1.1), use `decimal.js`:

```ts
import Decimal from 'decimal.js';
const d = new Decimal(linePriceCents).mul(new Decimal(rate));
const cents = d.toDecimalPlaces(0, Decimal.ROUND_HALF_EVEN).toNumber();
```

For simple cents × rate, `roundHalfToEven` is enough.

## Currency

V1: CHF only. Schema fields hardcoded. Multi-currency is V2 if foreign clients show up.

Currency symbol position varies by locale (see `domains/i18n.md`):
- DE: `CHF 1'820.50`
- FR: `1 820,50 CHF`
- IT: `CHF 1'820.50`

## Display format (Switzerland)

```
DE: 1'820.50      (apostrophe thousands, dot decimal)
FR: 1 820,50      (space thousands, comma decimal)
IT: 1'820.50      (Ticino style)
```

Helper:

```ts
formatChf(182050, 'de') → "CHF 1'820.50"
formatChf(182050, 'fr') → "1 820,50 CHF"
formatChf(0, 'de') → "CHF 0.00"
formatChf(99, 'de') → "CHF 0.99"
formatChf(-50, 'de') → "CHF -0.50"
```

Negative amounts use `-` prefix (not parentheses). Credit notes will use this in V1.1.

## Aggregation

Sum cents directly: `lines.reduce((acc, l) => acc + l.line_total_incl_vat_cents, 0)`. Never sum strings or floats. Watch `Number.MAX_SAFE_INTEGER` (9×10^15) on cross-stable aggregates — use `BigInt` when relevant.

## Database constraints

- Cents columns: `bigint not null check (>= 0)`
- Credits (V1.1): drop the check, allow negative
- VAT rates: `numeric(5,3) not null check (rate >= 0 and rate <= 1)`
- Quantities: `numeric(10,2) not null check (qty >= 0)`

## Canonical test fixture

Invoice fixture in `packages/lib/billing/vat.test.ts`:
```
Lines:
- Full board × 1 month  @ CHF 850.00 STD
- Lessons × 4           @ CHF 80.00  STD
- Hay × 30 days         @ CHF 12.00  RED
- Transport × 2 trips   @ CHF 75.00  STD

Expected (cents):
total_excl_vat = 168000, total_vat = 11628, total_incl_vat = 179628
```
CI runs this exact assertion. Any deviation fails the build.

## Common pitfalls

- Showing `850.0` in UI → use `formatChf`
- Multiply by VAT rate then round total → round per line, sum rounded values
- Naming column `850` not `_cents` → confusion
- `parseFloat` of user input → use `parseInt(... × 100)` with validation
- Currency code lowercase → ISO 4217 is uppercase

## Hard rules

- All money columns suffixed `_cents`
- Type `Cents` in TypeScript signatures
- `roundHalfToEven` for VAT
- `Intl.NumberFormat` for display
- One source of truth per amount
- Invoice totals stored (frozen at invoice time), not derived live
