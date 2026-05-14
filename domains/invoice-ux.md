# Domain: Invoice UX Reference

Extracted from `slices/08-invoice-pdf.md` to keep the slice spec under 150 lines. Three reference tables: failure modes & rescue paths, info hierarchy on the draft detail screen, and the DE/FR/IT state matrix for every invoice-related surface. Slice 08 acceptance tests reference rows here; this doc is the canonical source.

## Failure modes & rescue paths

Every failure class is named, caught, and surfaced to the owner — never silent. All five rescue paths log to Sentry with `stable_id`, `invoice_id`, `error_class`, set `invoices.pdf_generation_failed_at` + `pdf_generation_error`, and route the invoice into `/billing/invoices/sync-issues` for manual retry.

| Error class | Trigger | UX | Recovery |
|---|---|---|---|
| `SwissQrSchemaError` | `swissqrbill` rejects payload (bad IBAN, malformed address, currency != CHF) | Inline `rust-600` banner on draft, with offending field highlighted | Owner edits stable/client settings, regenerate |
| `SixValidatorUnreachable` | SIX validator HTTP timeout / 5xx during pre-launch validation | Toast "Validierung temporär nicht erreichbar — Entwurf gespeichert" + sticky retry | Cron retries every 5 min for 1h, then DLQ |
| `SwissQrBillVersionDrift` | Library version on runtime ≠ pinned version in `package.json` | Refuse to generate; Slack `#stable-warn`; deploy gate fails | Re-pin + redeploy (CI prevents auto-merge) |
| `PdfGenerationOOM` | react-pdf > 512MB on 50+ line invoice | Same draft state retained; queued retry on cold instance | Auto-retry once on Fluid instance with fresh memory; if still OOM, DLQ |
| `StorageWriteFailure` | Supabase Storage write fails after PDF generated | Status stays `draft`, no `sent_at`, owner sees retry button | Manual retry; idempotent via `(stable_id, number)` |

## Info Hierarchy — draft detail screen

What the owner sees, in order of scan priority (matches the two-pane DES-LOCK-1 layout):

1. **Client name + invoice number + period** — `lg 20/28`, top-left of left pane
2. **Grand total incl. VAT** — `xl 28/36`, tabular-nums, top-right of left pane
3. **Line items table** — `DataTable` primitive, sortable, tabular-nums on numeric columns
4. **VAT breakdown** — collapsed by default, click to expand per-rate detail
5. **PDF preview** — right pane (`PreviewPane` primitive, defaults open on `lg`/`xl`, closed on `md`, drawer-from-bottom on `sm`)
6. **Notes-to-client** — collapsed text input below table
7. **Confirm & Send** — primary CTA bottom-right of left pane; disabled until all required fields valid

## State matrix (DE/FR/IT)

Each cell has copy reviewed by native speaker. Full strings land in `docs/domains/copy-library.md` (CEO sweep DES-1) when that doc is written.

| Surface | Loading | Empty (first-run) | Empty (filtered) | Error | Partial |
|---|---|---|---|---|---|
| Draft list | Skeleton, 5 rows matching final layout | "Noch keine Entwürfe. Schliesse den Monat ab, um den ersten zu erzeugen." + CTA | "Keine Treffer. Filter zurücksetzen." | Inline banner + retry | "12 von 15 Entwürfen geladen — weitere laden" |
| Invoice list | Skeleton | "Noch keine Rechnungen versandt." | Same as above | Same | Same |
| Sync issues | Skeleton | "Keine Synchronisationsfehler — alles sauber." | n/a | Same | Same |
| Draft detail (PDF preview) | Pulse on preview pane only, table stays interactive | n/a (always has lines on draft) | n/a | "Vorschau konnte nicht generiert werden — `{error_class}`" + retry | n/a |

## When this doc updates

- A new rescue path lands in slice 08 → add the row here, not in the slice spec
- A new invoice-related surface ships (e.g., V1.1 credit-note list) → add a row to the state matrix
- Copy library `docs/domains/copy-library.md` lands → the State matrix here links to the authoritative DE/FR/IT strings instead of inlining them
