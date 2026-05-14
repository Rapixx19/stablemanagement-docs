# Design Review — 2026-05-07

`/plan-design-review` ran after CEO + Eng reviews on the same branch (`fix/pre-coding-review`, docs `30eea68`). All 7 passes at full depth. Codex unavailable, outside voices skipped (consistent with CEO + Eng decisions). Companion docs: `2026-05-07-ceo-review.md`, `2026-05-07-eng-review.md`.

Initial design completeness: **8/10**. After fixes: **9/10**. The team's `DESIGN.md` + `_tokens.css` + 3 hero HTML mockups (owner-dashboard, client-home, worker-today) are unusually mature for a pre-implementation plan.

## Locked decisions (this review)

- **DES-LOCK-1 — Slice 08 invoice draft detail layout**: two-pane with collapsible PDF preview. Defaults open on desktop, closed on tablet. Live re-render with 500ms debounce on edit. Update slice 08 spec.
- **DES-LOCK-2 — Owner mobile (375–767px)**: build a real owner-mobile view. Owner now renders all four viewports (sm/md/lg/xl). DESIGN.md edit required: replace "Owner only renders lg/xl" with "Owner renders all viewports." Implication: 11 slices with owner UI gain a mobile spec; visual regression matrix roughly doubles for the owner surface. **This is a deliberate scope addition inside HOLD mode** — track it in slice 16 + per-slice acceptance.

## Per-pass ratings

| Pass | Initial | After fix | What's missing for 10/10 |
|------|---------|-----------|--------------------------|
| 1. Information Architecture | 6/10 | 8/10 | Per-slice info hierarchy + owner-tablet behavior + slice 08 lock (done) |
| 2. Interaction State Coverage | 7/10 | 9/10 | Per-feature DE/FR/IT state copy library + ConfirmDialog primitive + toast-with-retry spec |
| 3. User Journey & Emotional Arc | 5/10 | 8/10 | Three storyboards in `docs/journeys/` (owner monthly close, client pay, worker first day) |
| 4. AI Slop Risk | 9/10 | 10/10 | AI-slop greppable lint enforced in slice 00 (already in CEO TODO DES-3) |
| 5. Design System Alignment | 9/10 | 10/10 | Slice 00 enumerates the primitive APIs; add ConfirmDialog, DataTable, PreviewPane primitives |
| 6. Responsive & Accessibility | 7/10 | 9/10 | Owner-mobile lock (done); axe-core CI; DD.MM.YYYY locale test; per-slice keyboard pattern + landmarks |
| 7. Unresolved Design Decisions | n/a | n/a | 8 lower-stakes decisions deferred to slice spec PRs |

## Spec additions (sweep before slice 00 + per-slice work)

### Slice 00
- Slice 00 spec: enumerate `packages/ui/primitives/` APIs (Button props/variants, Field types, Card, Modal, Drawer, TabBar, Pill, EmptyState, Toast). Plus three new primitives: **ConfirmDialog** (title + body + destructive CTA + cancel CTA, focus traps to cancel by default, escape closes, max-width 480px), **DataTable** (column headers, sortable, tabular-nums right-aligned, keyboard row select), **PreviewPane** (collapsible, defaults-open desktop, defaults-closed tablet).
- AI-slop greppable lint enforced (DES-3 from CEO carryover).
- DD.MM.YYYY locale unit test (DES-6 from CEO carryover).
- axe-core CI step on key flows (DES-5 from CEO carryover).

### DESIGN.md edits
- Replace "Owner only renders lg/xl" with "Owner renders all viewports (sm/md/lg/xl)."
- Add owner-tablet behavior: 320px nav rail collapses to 56px icon rail at 768–1023px; 12-col reflows to 8-col with 16px gutters; KPI row reflows from 4-across to 2x2 stacked.
- Add owner-mobile (375–767px) behavior: bottom tab bar pattern (4 items max: Übersicht / Stall / Tagebuch / Mehr), full-bleed cards, edit flows go to drawer-from-bottom, KPI row stacks to single column.
- Add **Confirmation dialog** to component primitives section.
- Add **DataTable** to component primitives section.
- Add **PreviewPane** to component primitives section.

### Per-slice spec additions
- **Each slice with new screens (01–17 except dashboards)**: add an "Info Hierarchy" subsection — what the user sees first, second, third. ASCII or prose. Owner-tablet + owner-mobile behavior listed explicitly.
- **Slice 04 + 06 + 08 + 09 + 11 + 13**: per-feature state matrix — for each list/table region, fill in loading + empty (first-run + filtered) + error + partial copy in DE/FR/IT.
- **Slice 08**: lock two-pane invoice draft layout (DES-LOCK-1). Live PDF re-render debounced 500ms or render-on-blur. Owner-mobile: invoice draft becomes single-pane stacked, "Vorschau" button opens preview as drawer from bottom.
- **Slice 13**: worker offline visual treatment — pending pill on each task in the IndexedDB queue (border-color rust-200, dashed, "wartet" label); "syncing" pill in header (already specified). Resolution on reconnect: pending pill clears with the 140ms tap-response motion when server confirms; on conflict, pill turns ochre with "Aktualisiert" + drawer auto-opens.
- **Slice 02**: lock stall map interaction pattern (deferred to slice spec PR).
- **Slice 05**: lock AI catalog upload review UX (deferred to slice spec PR; recommend side-by-side PDF + extracted rows).
- **Slice 10**: lock comms tone primitive — "professional, not WhatsApp-like." Spec the reply composer: no emoji shortcuts, no GIF picker, no read receipts (out of scope), one Send button + auto-save draft.

### `docs/journeys/` (new directory)
- `docs/journeys/owner-monthly-close.md` — full storyboard: month-end → 19 drafts auto-generated → owner reviews each → press "Confirm all" → 19 PDFs go out → 19 Bexio pushes → sync-issues panel verification. ~40 lines. Includes failure paths.
- `docs/journeys/client-pay-invoice.md` — email arrives in DE → click magic link → see invoice → download PDF → open banking app → scan QR → bank pre-fills → pay → reconciliation in 30 min → status flips.
- `docs/journeys/worker-first-day.md` — owner adds worker → magic-link email → install PWA on iPhone Safari (Add to Home Screen instructions) → standalone "Stable Crew" icon → first home view (empty state with `Nichts geplant. Mach dir einen Kaffee.`) → owner assigns first task → worker sees within 2s.

### Copy library
- `docs/domains/copy-library.md` (new doc, ~80 lines) — 10 examples per category (invoice subject, magic-link subject, error toast, empty state, push notification, confirmation dialog title/body, success toast, validation error, button labels, table column headers) in DE / FR / IT. Native-speaker reviewed before pilot. Worker imperative-vs-noun-form locked here: "Fütterung Aisle A · 07:00" (noun-form, Swiss-German barn vocabulary) over "Füttere Aisle A · 07:00".

### Empty-state illustrations
- `packages/ui/illustrations/` — pen-line SVGs by Ferdinand. Format: ≤5 KB each, single-color `--ink-700`. Two for V1: `no-horses-yet.svg`, `no-events-today.svg` (TODOs slice 13 design system).

## Mockup recommendations (4 new HTML files in existing style)

Lower priority but high leverage. Match `_tokens.css` conventions, no new tokens. ~30 min CC each:

- `mockups/owner-invoice-draft.html` — locked two-pane layout (DES-LOCK-1). DE copy. Three states: draft / sent / cancelled.
- `mockups/owner-stall-map-edit.html` — interactive editor mock. DE copy. Shows the 8-col grid + side panel for stall details.
- `mockups/owner-catalog-ai-upload.html` — side-by-side PDF + extracted rows + per-row owner-confirm-or-edit pattern.
- `mockups/client-thread.html` — comms thread reply composer with the "professional, not WhatsApp" tone locked. DE copy.
- `mockups/owner-mobile.html` — locks DES-LOCK-2. Single file with `<section>` per viewport showing 375 / 768 / 1024 / 1280 progressions of the dashboard.

The team's update policy stands: if a mockup drifts from DESIGN.md, DESIGN.md wins; mockup updates in the same PR as the slice that touches it.

## NOT in scope (design)

- Dark mode (V1.1, already in DESIGN.md "Out of scope").
- Custom illustration set beyond two pen-line SVGs (V1.1, already in DESIGN.md).
- Brand photography (V2).
- White-label theme switching (V2).
- Owner-mobile feature parity beyond what V1 ships — basic responsive mobile view per DES-LOCK-2, but not a separate mobile-first re-design. Phone owner gets the same information architecture, reflowed.
- Inbound calendar sync UX (V1.1, deferred).

## What already exists (design leverage)

- `DESIGN.md` — 154 lines of locked design decisions. Source of truth.
- `_tokens.css` — 157 lines mirroring DESIGN.md. Drop-in for `packages/ui/tokens.ts`.
- 3 HTML mockups (owner-dashboard, client-home, worker-today). Calibrated to DESIGN.md. Ready to translate to React with `_tokens.css` as the variable contract.
- AI-slop blacklist explicit (forbidden colors, phrases, font fallbacks; no icon-in-colored-circle; no decorative blobs; no `text-align: center` defaulting; tabular-nums right-aligned for numbers; three radii by purpose; three motions; status pills carry icon + label).
- State matrix per surface in DESIGN.md.
- Voice rules per surface (factual / warm / imperative+short).

## Reviewer

`/plan-design-review` (Claude Opus 4.7) on 2026-05-07. Two locked decisions (DES-LOCK-1 invoice draft layout, DES-LOCK-2 owner mobile). 8 deferred decisions (slice-spec time). 4 new HTML mockup recommendations. 3 new journeys. 1 new copy library doc. Per-slice info-hierarchy and per-feature state matrix sweeps captured.
