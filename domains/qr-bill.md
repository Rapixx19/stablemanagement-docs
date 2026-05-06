# Domain: Swiss QR-bill (QR-Rechnung / QR-facture)

The standardized Swiss payment slip with embedded QR code that replaced the orange / red ESR slips in 2022. Banks use the QR to populate payment forms. Misimplementation = customer's bank rejects, customer hates us.

## Library

`swissqrbill` on npm. Maintained, used in production by many CH SaaS. Generates SIX-compliant PDFs. We use it server-side.

```ts
import { SwissQRBill } from 'swissqrbill/pdf';
const bill = new SwissQRBill({
  currency: 'CHF',
  amount: 1820.50,
  reference: '210000000003139471430009017',  // 27-digit QRR
  creditor: { name, address, zip, city, account: 'CH44 3199 9123 0008 8901 2', country: 'CH' },
  debtor: { name, address, zip, city, country: 'CH' },
  message: 'Invoice 2026-0142',
});
bill.attachTo(pdfDoc);
```

## QR-IBAN vs IBAN

- **QR-IBAN**: an IBAN reserved for QR-bills with structured creditor reference (QRR). Issued by banks specifically for this purpose.
- **Regular IBAN**: works with creditor reference type `NON` (no reference) or `SCOR` (Creditor Reference per ISO 11649).

Our recommendation: stables use a **QR-IBAN** with **QRR** for cleaner reconciliation. Bexio supports both. Settings page asks the stable owner during onboarding which they have; we adapt.

## QRR — structured creditor reference

27-digit reference, mod-10 recursive checksum. Deterministic encoding from invoice ID = stable can reverse-lookup who paid.

Encoding scheme (see `packages/lib/billing/qrr.ts`):

```
QRR = leftPad(stable_short_id, 6, '0')
    + leftPad(invoice_year, 4, '0')
    + leftPad(invoice_seq, 16, '0')
    + checksum_digit
```

Total 26 digits + 1 check digit = 27.

The check digit uses ISO 11649 mod-10 recursive:
```ts
function checksum(numStr: string): string {
  const lookup = [0,9,4,6,8,2,7,1,3,5];
  let carry = 0;
  for (const c of numStr) {
    carry = lookup[(carry + parseInt(c)) % 10];
  }
  return ((10 - carry) % 10).toString();
}
```

Tests for the algorithm in `packages/lib/billing/qrr.test.ts` use SIX-published reference values.

## SIX validator

The SIX Group (operator of the standard) provides a validator at `paymentstandards.ch`. Two endpoints:
- Web UI for manual checks
- API/SDK for automated validation (we use this in CI)

Pre-launch: every PDF round-tripped. Post-launch: 5% sample monitored daily.

In CI, the validator runs as part of the `pnpm test:e2e` suite. Failed validation blocks merge.

## DE / FR / IT templates

The `swissqrbill` library has built-in localization for the QR slip itself (the "Payment part" header, etc.). Our invoice page is our own template and must be translated. Three Mustache templates:

- `templates/invoice-de.mustache`
- `templates/invoice-fr.mustache`
- `templates/invoice-it.mustache`

Same data, three texts. The bottom QR slip is library-rendered.

## PDF generation infrastructure

- Runs as a server action in `packages/lib/billing/pdf.ts`
- Uses `pdfkit` underneath (via `swissqrbill`)
- Generated PDF is uploaded to Supabase Storage at `invoices/{stable_id}/{invoice_id}.pdf`
- Invoice record updated with `pdf_path` and `pdf_generated_at`
- Cached: regenerating an unchanged invoice returns the cached file
- Idempotency key prevents double-generation under retry

## Performance

Target: < 3s p95, < 8s p99. The PDF library is fast; bottleneck is font loading. Cache fonts in memory across invocations (Vercel Edge Function with `regional` execution).

## Common pitfalls

- **Wrong account number format**. IBAN must be space-grouped (CH44 3199 ...). Library handles, but don't pass concatenated.
- **Wrong currency code**. Must be exactly `CHF` or `EUR`. Lowercase = invalid.
- **Amount with comma decimal**. The data field uses `.` (point); display can localize.
- **Reference with non-digits**. Strip spaces, hyphens before validation.
- **Missing creditor address**. Mandatory; library throws.
- **Custom paper sizes**. Stick to A4. The QR slip occupies the bottom third, perforated for tear-off.

## What banks do with the QR

A debtor scans the QR with their banking app. The app pre-fills the payment form: amount, currency, reference, creditor IBAN, creditor name. Debtor confirms. Bank routes payment.

Once paid, the bank may send a `camt.054` notification file referencing the QRR. We can ingest this V1.1 to auto-mark invoices paid (currently we rely on Bexio reconciliation poll — slice 09).

## Security

- The QR encodes payment data in cleartext. That's the standard — payment data isn't secret, it's instructions.
- Don't put PII in the message field beyond invoice number.
- Unstructured message field max 140 chars.

## Testing fixtures

We maintain three test invoices in `packages/db/fixtures/qr-bills/`:
- DE invoice with mixed VAT rates
- FR invoice with single rate
- IT invoice with all-day-rate (lesson session)

All three pass SIX validator. Used as regression tests on every PR.
