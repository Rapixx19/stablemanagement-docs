# Slice 08 — Invoice Draft and QR-Bill PDF

**Phase:** Billing · **Estimate:** 6 dev-days · **Owner:** Sharad backend, freelancer frontend

The headline feature. Generates Swiss QR-bill PDFs that pass the SIX validator. Without this, V1 doesn't ship.

## Goal

Owner reviews end-of-month draft invoices for each client, edits if needed, confirms. System generates a Swiss-compliant QR-bill PDF in DE/FR/IT (per client language), stores it, marks invoice `sent`. Client downloads and pays via their banking app's QR scanner.

## What ships

### Owner side

1. **Draft list** at `/billing/invoices?status=draft`. Auto-generated drafts when month closes (or on-demand). Per-client review. **Empty-period skip (EDGE-1)**: clients with zero billable lines in the period produce no draft. Surface count of skipped clients in the list header ("3 Klienten ohne Position übersprungen").
2. **Draft detail** at `/billing/invoices/[id]` — **two-pane layout (DES-LOCK-1)**. Left pane: line items + VAT breakdown (editable). Right pane: live `PreviewPane` PDF preview. Defaults: open on `lg`/`xl`, collapsed on `md` (tablet) with a "Vorschau" toggle. Edit → re-render preview with **500ms debounce**. Owner-mobile (`sm`): single-pane stacked, "Vorschau" button opens preview as drawer-from-bottom.
3. **Confirm and send** action — strict ordering: validate → generate PDF bytes → **write to Storage first** → update `status='sent'` + `sent_at` + `pdf_path` in one transaction → enqueue email (EDGE-5). If Storage write fails, status stays `draft`, owner sees rescue UI, no half-sent state. If status update fails after Storage write, retry idempotent via `(stable_id, number)`.
4. **Invoice list view** at `/billing/invoices`. Filters: status, period, client. Table of invoice number, client, amount, status, sent date. Numeric columns tabular-nums right-aligned (`JetBrains Mono`). Status pills: `draft`=ink-300, `sent`=ink-500, `paid`=moss, `overdue`=ochre, `cancelled`=rust-outline.
5. **Sync issues panel** at `/billing/invoices/sync-issues` — invoices in `bexio_sync_failed` (slice 09) + invoices with `pdf_generation_failed` + DLQ overflow. Manual retry per row. Hidden when stable has no Bexio connection.
6. **Manual mark-paid** action on `/billing/invoices/[id]`. Single button "Als bezahlt markieren" + date picker (defaults today). Sets `status='paid'`, `paid_at`, `paid_source='manual'`, `marked_paid_by_user_id = auth.uid()`. Audit-logged. **If stable has an active Bexio connection**, a `ConfirmDialog` warns: "Bexio synchronisiert diesen Status nicht zurück — fortfahren?" — owner can still proceed; manual overrides the auto path. **If no Bexio**, the button is the primary "paid" action and no warning fires.
7. **Invoice CSV export** at `/billing/invoices/export?period=YYYY-MM`. Downloads a UTF-8 CSV with columns: `invoice_number, client_name, period_start, period_end, total_excl_vat_cents, total_vat_cents, total_incl_vat_cents, currency, status, sent_at, paid_at, paid_source, qr_reference`. Importable into Banana / KLARA / CashCtrl / Excel. Available **regardless** of Bexio status — Bexio-connected stables can still use it as a backup.

### Client side

6. **My invoices** at `/invoices`. List + download PDF + status pill. Tap an invoice → detail view + download.

## Schema diff

```sql
-- 008_invoices.sql
create table invoices (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  person_id uuid not null references people(id),
  number text not null,                       -- "2026-0142", unique per stable
  period_start date not null,
  period_end date not null,
  status text not null default 'draft' check (status in ('draft','sent','paid','overdue','cancelled')),
  total_excl_vat_cents bigint not null,
  total_vat_cents bigint not null,
  total_incl_vat_cents bigint not null,
  vat_breakdown_jsonb jsonb not null,
  qr_reference text,                          -- 27-digit QRR
  pdf_path text,                              -- Storage path
  pdf_generated_at timestamptz,
  bexio_invoice_id text,                      -- set in slice 09
  sent_at timestamptz,
  paid_at timestamptz,
  due_date date not null,
  language text not null check (language in ('de','fr','it')),
  notes_to_client text,
  version int not null default 0,             -- optimistic locking (E-CQ-2), bumped on every UPDATE
  pdf_generation_failed_at timestamptz,       -- set on PDF gen failure, cleared on success
  pdf_generation_error text,                  -- last error class, e.g. 'SixValidatorUnreachable'
  paid_source text check (paid_source in ('bexio','manual','imported')),  -- 'imported' reserved for V1.1 camt.054
  marked_paid_by_user_id uuid references auth.users(id),  -- non-null for paid_source='manual'
  created_at timestamptz not null default now(),
  unique (stable_id, number)
);

-- Cross-stable QR reference uniqueness (EDGE-3). QRR is 27 digits, deterministic from invoice id.
create unique index invoices_qr_reference_unique on invoices (qr_reference) where qr_reference is not null;

create table invoice_lines (
  id uuid primary key default gen_random_uuid(),
  invoice_id uuid not null references invoices(id) on delete cascade,
  stable_id uuid not null,
  horse_id uuid references horses(id),
  service_id uuid references services(id),
  description text not null,
  qty numeric(10,2) not null,
  unit_price_cents bigint not null,
  vat_code text not null,
  vat_rate numeric(5,3) not null,             -- frozen at invoice time
  line_total_excl_vat_cents bigint not null,
  line_total_vat_cents bigint not null,
  occurred_on date not null,
  sort_order int not null
);
```

## RLS

Both tables scoped by `stable_id` per `domains/rls.md`.

- `invoices`:
  - `owner`: full access. UPDATE on `draft` rows is unconstrained; UPDATE on `sent`/`paid`/`cancelled` rows is limited to `{status, paid_at, paid_source, marked_paid_by_user_id, bexio_invoice_id, version}` via `BEFORE UPDATE` trigger (already-sent invoices are accounting records — line items frozen).
  - `worker`: **no access** — billing is owner-only.
  - `client`: SELECT own invoices only (`person_id` matches their `people` row via `user_id = auth.uid()`). **No INSERT/UPDATE/DELETE.**
- `invoice_lines`:
  - `owner`: full access while parent invoice `status='draft'`; **read-only** once parent is `sent` or beyond (enforced by `BEFORE UPDATE/DELETE` trigger checking `invoices.status`).
  - `worker`: **no access**.
  - `client`: SELECT lines of own invoices only.

**DELETE denied for all roles** on `invoices` once `status != 'draft'` — accounting records per Code of Obligations Art. 958f (see `domains/data-retention.md`). Cancellation is a `status` flip, not a row delete.

## Invoice numbering

Format: `YYYY-NNNN`. Sequential per stable, starts at 0001 each year. Generated by Postgres function with row lock — no race conditions.

## QR-bill generation

- **Renderer**: `swissqrbill` (npm) for the QR payment slip page, composed into the full invoice via **`react-pdf`** (`@react-pdf/renderer`). One renderer stack across the whole PDF — no headless Chromium, no Puppeteer, keeps the cold-start budget intact on Vercel serverless.
- **Pinned versions** in `package.json` exact (no `^`): `swissqrbill@<lock>`, `@react-pdf/renderer@<lock>`. Upgrades go through a manual SIX-validator regression run, never auto-merged.
- Inputs: stable IBAN/QR-IBAN, stable address, client address, amount, currency CHF, **structured creditor reference (QRR)** — 27 digits, deterministic encoding of the invoice ID.
- See `domains/qr-bill.md` for the QRR encoding algorithm (mod-10 recursive checksum) and SIX validator integration.
- **Pre-launch validation**: every PDF round-tripped through `paymentstandards.ch` validator in CI before launch. After launch, sample 5% of invoices for ongoing validation, surface failures to Sentry.

## Templates

DE / FR / IT invoice templates. Mandatory fields from slice 07. Stable logo (uploaded in `/settings`). Notes-to-client field (free text, e.g., "Thank you for boarding with us"). Footer with UID-MWST.

Typography per `DESIGN.md`: invoice header / stable name plaque / receipt number in **Source Serif 4** (the one place serif is allowed); body and line items in **Inter Tight**; invoice number, QRR reference, IBAN, amounts in **JetBrains Mono** with tabular figures, right-aligned in the line-items table; grand total in `2xl 40/48` — DESIGN.md notes this is the *only* place `2xl` appears in the product. Numbers formatted Swiss style (`1'234.50 CHF`) via `Intl.NumberFormat('de-CH')`. No exclamation marks, no marketing copy in the notes-to-client default ("Thank you for boarding with us" is fine; "Welcome aboard!" is not).

Page 1: invoice. Page 2: QR-bill payment slip (mandatory layout per SIX standard). Library handles the layout.

## Performance

Targets, measured in Sentry performance traces and asserted in CI via a dedicated load test:

- **p50 < 1.5s**, **p95 < 3s**, **p99 < 8s** end-to-end (server action invoked → PDF bytes returned).
- **Cold start budget**: first invoice after 15-min idle finishes < 5s. Asserted by a CI job that warms a fresh Vercel preview, sleeps 15 min, then times one PDF.
- **Concurrent generation**: 100 invoices generated in parallel (simulates an end-of-month "Confirm all" press) — no failures, p95 stays < 4s, no Vercel function timeouts. Run as a nightly job in CI on the preview deployment.
- **Per-instance concurrency cap = 2** for PDF routes (E-PERF-3). Set `export const maxDuration` + Vercel concurrency config to 2 per Fluid Compute instance — 200MB+ PDFs OOM if 3+ generate on one instance. Excess requests queue, never error.
- **Failure-rate SLO**: alert if PDF generation failures > 2% over any rolling 1h window. Pager goes to Sharad.
- Memory: each PDF gen capped at 512MB Vercel function memory. If exceeded, log + Sentry capture, fall back to queued retry rather than 500ing.

## UX reference

Three reference tables live in `docs/domains/invoice-ux.md`:
- **Failure modes & rescue paths** — 5 named error classes (`SwissQrSchemaError`, `SixValidatorUnreachable`, `SwissQrBillVersionDrift`, `PdfGenerationOOM`, `StorageWriteFailure`) with trigger / UX / recovery per row. Acceptance test below references each by name.
- **Info Hierarchy** — scan-priority order for the two-pane draft detail screen (DES-LOCK-1).
- **State matrix** — loading / empty / error / partial per invoice surface in DE/FR/IT.

## Acceptance criteria

- [ ] Draft auto-generated from tab on cycle close
- [ ] Owner edits a line, total recomputes correctly, PDF preview updates
- [ ] Confirm → PDF generated, stored, status = sent, sent_at recorded
- [ ] Generated PDF passes SIX validator (no errors, no warnings on critical fields)
- [ ] QR-IBAN required for stables using QRR; clear error if not configured
- [ ] **DE/FR/IT renders test**: same invoice fixture rendered three times (one per locale) — VAT label, dates, currency formatting, address layout match a golden snapshot per locale
- [ ] **Visual regression**: each rendered PDF rasterized to PNG and compared against a checked-in reference (`tests/fixtures/invoices/golden/{de,fr,it}.png`) within 0.1% pixel delta. Fails CI on any drift
- [ ] **Mod-10 collision sweep**: generate 100k synthetic invoice IDs, encode each to QRR via the production helper, assert zero duplicates and 100% mod-10 checksum validity. **Cross-stable extension (EDGE-3)**: same sweep spanning 10 stables in one run — assert the `invoices_qr_reference_unique` index holds (no QRR collision across tenants)
- [ ] **Empty-period skip (EDGE-1)**: month-close run for a client with zero billable lines produces no draft row and surfaces "1 Klient übersprungen" in the list header
- [ ] **Storage-before-status (EDGE-5)**: simulated Storage write failure → invoice stays `status='draft'`, `sent_at IS NULL`, `pdf_generation_failed_at` populated, retry succeeds idempotently with same `(stable_id, number)`
- [ ] **Rescue path coverage**: each of `SwissQrSchemaError`, `SixValidatorUnreachable`, `SwissQrBillVersionDrift`, `PdfGenerationOOM`, `StorageWriteFailure` has a unit test that asserts the right Sentry tag + the right UX banner copy + the invoice lands in sync-issues
- [ ] **50-line overflow visual regression**: golden invoice with 50 line items rendered in DE/FR/IT — assert page 1+2 layout holds, QR slip is alone on the final page, no truncation, pixel-delta < 0.1% vs `tests/fixtures/invoices/golden/50-line-{de,fr,it}.png` (EDGE-4)
- [ ] **Optimistic locking**: concurrent UPDATE on the same draft (`version` mismatch) → second writer gets `OptimisticLockError`, UI re-fetches and prompts owner to reapply edits
- [ ] **Manual mark-paid (no-Bexio stable)**: owner taps "Als bezahlt markieren" → `status='paid'`, `paid_source='manual'`, `marked_paid_by_user_id` set, audit row written, no warning dialog
- [ ] **Manual mark-paid (Bexio-connected stable)**: same action shows the override warning dialog; on confirm, status flips locally with `paid_source='manual'` and Bexio is **not** auto-poked (next 30-min poll could overwrite if Bexio later reports a different status — documented in slice 09 reconcile path)
- [ ] **CSV export**: 12-invoice month for La Fattoria exports to UTF-8 CSV with all 13 columns; CHF amounts as integer cents (no locale formatting in CSV — that's the importer's job); opens correctly in Excel with `de-CH` regional settings
- [ ] **No-Bexio stable hides sync-issues panel** in nav; deep-link `/billing/invoices/sync-issues` returns a 404-like empty state, not a blank page
- [ ] **Concurrent generation load test**: 100 invoices generated in parallel on Vercel preview → 0 failures, p95 < 4s
- [ ] **Cold-start budget**: 15-min-idle Vercel preview → first PDF returns < 5s
- [ ] Client downloads PDF and bank app reads QR successfully (manual test against UBS, ZKB, PostFinance, and Raiffeisen apps before pilot)
- [ ] Cancelling an invoice creates a credit note (V1.1 — for V1, just sets status `cancelled` and audit-logs)
- [ ] Invoice number sequence is correct under concurrent generation (load test 50 parallel — no gaps, no duplicates, monotonic per stable)

## Acceptance integration test

`apps/owner/tests/integration/invoice-pdf.test.ts`

```ts
test('confirmed invoice produces SIX-valid QR-bill PDF', async () => {
  const inv = await seed.confirmedInvoice({ totalChf: '1820.50' });
  const buf = await generateInvoicePdf(inv.id);
  const res = await sixValidator.validate(buf);
  expect(res.errors).toHaveLength(0);
  expect(inv.qrReference).toMatch(/^\d{27}$/);
});
```

## Out of scope

- eBill dispatch (V1.1)
- Credit notes (V1.1)
- Recurring invoice scheduler (slice 06 covers monthly auto-bill)
- Payment recording UI (V1.1 + slice 09 reconciliation)
