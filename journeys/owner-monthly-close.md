# Journey — Owner Monthly Close

The cycle that has to work. La Fattoria's owner runs this 4 times in 4 weeks before V1 is declared done (pilot exit criterion #1).

## Scene

Tuesday morning, 1 June 2026, 08:45. End-of-May billing cycle. Owner sits at his desktop with a coffee. 19 boarders, 28 horses, 4 weeks of lessons and transport accumulated throughout May. He has an hour before he wants to be back in the barn.

## The walkthrough

1. **08:45** — Opens `/owner/dashboard`. Top tile shows `Entwürfe bereit · 19` in `xl 28/36` tabular numerals. Below: `Umsatz Mai · CHF 24'180.00`.
2. **08:46** — Clicks the tile. Lands on `/billing/invoices?status=draft`. DataTable: 19 rows. **Empty-period skip** (EDGE-1) means one boarder who pulled their horse out mid-May doesn't appear — owner notices the gap and confirms it's correct.
3. **08:47** — Opens draft #1. Two-pane layout (DES-LOCK-1): left pane with total `xl 28/36`, line items, VAT breakdown collapsed. Right pane: PDF preview, rendered in 600ms.
4. **08:48** — Edits one line: a transport entry auto-pulled as "Transport" — types in "Transport CH→FR Lugano." PDF preview re-renders 500ms after he stops typing. Total unchanged.
5. **08:49** — Spots: VAT on the bedding line shows 8.1% (`STD`). Should be 2.6% (`RED`). Same row will be wrong on 11 other drafts. Opens `/billing/catalog` in a second tab. Fixes the bedding service VAT code. Save → cascade dialog (slice 06 item 9): "12 aktive Abonnements bei CHF 80 · VAT-Wechsel betrifft alle." Picks **"Ab nächster Periode"** (preserves current-period billing).
6. **08:52** — Back to draft #1. Clicks "Neu berechnen." Line now shows 2.6%. Total drops CHF 3.50. PDF preview refreshes. Confirm + Send. Status pill: `draft` (ink-300) → `sent` (ink-500). PDF stored. Bexio push initiates.
7. **08:53 — 09:08** — Drafts 2–19. Average 45 seconds per draft, faster as rhythm builds. Two need a line edit; one needs a price override (a long-stay discount). The rest he confirms with a glance at the total + Tab.
8. **09:09** — All 19 confirmed. Watches `/billing/invoices/sync-issues` over the next 60 seconds. 18 turn moss-green (Bexio-synced badge). One lands in DLQ with `BexioLineValidationError` — the field path highlights "Position 4: VAT-Code stimmt nicht mit Bexio überein" + deep-link to the catalog entry.
9. **09:10** — Click through. The bedding service in Bexio's article map still has the old 8.1% rate. Updates in Bexio (2 clicks). Returns to sync-issues, hits Retry. Green within 5 seconds.
10. **09:11** — 19 invoices sent, 19 Bexio-synced. Closes the laptop. Coffee still warm.

## What this surfaces

- **Per-row confirm, no batch**: DES-LOCK-1 deliberately removes "Confirm all" so the owner reads every total. The cost is 19 individual confirms; the value is zero accidental sends.
- **Cascade choice is "next period," not "now"**: slice 06's price-change UX must default to the safe option, not the convenient one.
- **500ms debounce on preview**: load-bearing for trust. Without it, owner doesn't believe his edit landed.
- **Sync-issues panel must surface field-level Bexio errors**, not generic "push failed" — owner recovers in 2 clicks because he can see exactly what's wrong.

## Failure paths

- **Bexio fully down** (`BexioApiUnreachable`): all 19 land in DLQ. Banner: "Bexio temporär nicht erreichbar — automatischer Retry läuft." Owner closes the laptop knowing the email+PDF half already went; Bexio fills in within the hour.
- **PDF generation fails on one draft** (`PdfGenerationOOM`): that draft holds with `pdf_generation_failed_at` set. Banner + Retry button. Succeeds on cold instance. Other drafts unaffected.
- **Owner interrupted mid-review** (mucking emergency): drafts persist as drafts; returns later, picks up where he left off. No half-sent state.

## Success criteria

- 19 drafts → 19 sent + 19 Bexio-synced in under 30 minutes
- Zero data loss on any interruption mid-flow
- DLQ entry recoverable in < 2 minutes
- Owner does not open Excel or WhatsApp during the cycle (pilot exit criterion #4)
