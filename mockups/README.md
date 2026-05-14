# Mockups

Twenty-two screens calibrated to `DESIGN.md`. Static HTML with inline CSS, sharing one tokens file.

### Hero screens (one per surface)

| File | Surface | Audience | Density | Slices it anchors |
|---|---|---|---|---|
| `owner-dashboard.html` | `/owner` | Stable manager | High (12-col, 320px nav rail, 1280px max) | 03, 06, 08, 09, 16 |
| `client-home.html` | `/client` | Horse owner | Medium (12-col, 960px max read width) | 03, 06, 08, 11, 16 |
| `worker-today.html` | `/worker` | Groom | Low (single column, 16px padding, 44px tap rows, bottom tab bar) | 12, 13, 14, 15, 16 |

### Slice-specific screens (added 2026-05-07 from /plan-design-review)

| File | Surface | Locks |
|---|---|---|
| `owner-invoice-draft.html` | `/owner/billing/invoices/[id]` | DES-LOCK-1: two-pane editor + collapsible PDF preview, defaults open desktop / closed tablet, live re-render with 500ms debounce. Anchors slice 08. |
| `owner-stall-map-edit.html` | `/owner/horses-and-boxes` (stall map) | Per-row interactive grid (8 stalls per row), side panel for stall details, type segmented control (Box / Paddock / Wartung), legend with five state colors. Anchors slice 02. |
| `owner-catalog-ai-upload.html` | `/owner/settings/catalog/import` | Side-by-side PDF (left) + extracted-rows table (right), per-row owner-confirms-or-edits with confidence pill (high/mid/low), 4-step progress. LLM output stays untrusted; nothing writes to `services` until owner confirms. Anchors slice 05. |
| `client-thread.html` | `/client/messages` | Locks the "professional, not WhatsApp" tone. Flat-box messages with hairline borders (NOT chat bubbles), Source-Serif subjects, asymmetric stable-vs-client visual treatment, no emoji/GIF/read-receipts in composer. Anchors slice 10. |

### Slice-specific screens (added 2026-05-13 from owner-workflow expansion + DES-LOCK-2)

| File | Surface | Locks |
|---|---|---|
| `owner-mobile.html` | `/owner/dashboard` × 4 viewports | DES-LOCK-2: same dashboard rendered at xl/lg/md/sm. xl/lg = 12-col with 320px rail; md (tablet) = 56px icon rail + 8-col + KPIs 2×2; sm (mobile) = bottom tab bar (Übersicht · Stall · Tagebuch · Mehr) + KPIs stacked + full-bleed cards. |
| `owner-ops.html` | `/owner/ops` | Internal-only ops calendar (slice 15 #9). Worker rows × days, shift bars with arrival overlay (on-time / 13 min spät / krank), drag-drop task pills, owner row at top (clay-200) for self-tasks + private events, coverage gaps in ochre. |
| `owner-event-signup.html` | `/owner/schedule/new` | Anmeldung-aktivieren section locks visibility to public_stable when toggled on (slice 03 + `domains/event-signup.md`). One TX creates `events` + linked `availability_slots`. Confirm dialog explains the bridge. |
| `owner-bulk-assign.html` | `/owner/billing/subscriptions/bulk-assign` | 3-step wizard (slice 06 item 8). Step 3 (Prüfung) shows 38-row review table with per-row override fields, duplicate-detection pre-flag (`Wird übersprungen`), summary stats (angelegt / übersprungen / monatl. Summe), atomic transaction badge. |
| `owner-requests.html` | `/owner/requests` | Slice 11 owner inbox. Age pills: fresh (ink-300), warn at 18h (ochre-600), late at 24h (rust-600). 3 actions per row: confirm-into-slot / decline / reschedule. Free-form requests (no slot) get separate action set. |
| `owner-invoice-list.html` | `/owner/billing/invoices` | Slice 08 invoice list with sync-issues banner (ochre, surfaces DLQ count from slice 09), filter chips by status + period, Bexio-status column, tabular-nums right-aligned amounts, footer KPI summary (Gesamt / Bezahlt / Offen / Überfällig). |
| `owner-tab.html` | `/owner/people/[id]/billing/tab` | Slice 06 per-client tab: 3-tab nav (Abonnements · Aktuelles Tab · Rechnungen), grand total in `2xl 40/48`, per-horse breakdown sections + client-level section, origin pills (Abo / Einzel), quick-add pinned services, VAT-summary footer with Rechnungsentwurf-erstellen CTA. |
| `owner-tasks-mine.html` | `/owner/tasks/mine` | Slice 04 owner self-task list. Filter chips today / this week / overdue (rust). Inline add-bar at top. Three panels: Überfällig (rust accent), Heute, Diese Woche. Each task shows time + cross-link to relevant area (Sync-Fehler / Klient / Buchhaltung / Ops-Kalender). |
| `onboarding-accounting-choice.html` | First-run wizard step 7 of 9 | Slice 16 + slice 09 binary accounting choice locked 2026-05-13: Bexio (recommended, blue tick) vs. Kein/Anderes (CSV + manual mark-paid). Progress bar shows "Buchhaltung" as active step. Both choices show feature lists + "next step" copy so the choice has predictable consequences. |
| `worker-install-flow.html` | Slice 12 iOS install sequence | Three-stage phone-frame mockup: ① Magic-link landing in Safari with install-banner + arrow-down asset pointing at Share icon; ② iOS share sheet with "Zum Home-Bildschirm hinzufügen" highlighted in clay-200; ③ Standalone PWA with status bar + empty state + bottom tab bar. Anchors slice 12 acceptance "from email-tap to standalone icon < 90s". |

### Profile screens (added 2026-05-14, anchored by slices 01a + 01b)

| File | Surface | Locks |
|---|---|---|
| `client-profile.html` | `/owner/clients/[id]` | Client profile single-page view: 96px avatar header with all contact metadata, 6-tab nav (Profile · Documents · Horses · Communication · Billing · Activity), Reducto-indexed search bar, mini-horses grid, 12-month stacked spending chart with category breakdown, document grid with expiry badges, recent-activity timeline. Anchors slice 01a (profile + tabs except Documents) + slice 01b (Documents tab + Reducto search) + `domains/document-search.md`. |
| `horse-profile.html` | `/owner/horses/[id]` | Horse profile single-page view: large square hero photo with 5-thumb photo strip overlay, 5-tab nav (Profile · Documents · Health log · Schedule · Photos), Reducto-indexed search bar, document grid (passport / insurance / vet / farrier / x-ray / contract / authorisation), health-log timeline color-coded by event type, current-week schedule strip with terracotta highlighting today. Anchors slice 01a (profile + tabs except Documents) + slice 01b (Documents tab + Reducto search) + `domains/document-search.md`. |
| `owner-search.html` | `/owner/search` | Cross-entity global search (Reducto-indexed). Hero search bar + ⌘K shortcut, filter chips by entity type (All · Documents · Horses · Clients · Events · Messages · Invoices) with counts, ranked results showing entity-type badge + parent path + Reducto-extracted snippet with `<em>`-highlighted matches + similarity score pill (moss ≥0.8, ochre 0.6–0.8, ink-100 <0.6). Sidebar: Reducto index status panel + recent searches + cross-language tip. Locks the entry-point UX from `domains/document-search.md`. |
| `client-services.html` | `/client/services` | Client-side bookable services catalog (slice 11 #11 + slice 05 `bookable_by_client` toggle). Single-pane 960px max, grouped by category (Riding · Care & Hoof · Veterinary & Transport), each service card shows client_facing_name + price + description + provider avatar + next-3-available-dates pills (moss with capacity counts) + per-service rules (`min_notice_hours`, `max_advance_days`). Empty/unavailable variant for sold-out services. Bottom note routes off-catalog requests (emergency vet, special transport) to direct messaging. Locks owner control: only services owner explicitly flips `bookable_by_client=true` appear here. |
| `owner-document-inbox.html` | `/owner/inbox/documents` | Document review queue (slice 01b #5 + `domains/document-ingestion.md`). Per-stable email address banner at top, quarantine warning band, filter chips by source (email / mobile / share / bulk / quarantine), bulk-confirm bar for high-confidence pre-selected rows, document rows showing thumbnail + sender + source pill (email / alias / share / camera / bulk) + LLM triage proposals with confidence pills (moss ≥0.9, ochre 0.7–0.9, rust <0.7), inline edit dropdowns. Sidebar: this-week stats + active aliases list + how-to-use tip. |

The dashboard (`owner-dashboard.html`) was refreshed 2026-05-14 with sparkline KPIs, horizontal Today timeline, horse-photo strip, and alerts band — replacing the older list-heavy invoice table + requests cards.

`_tokens.css` mirrors `DESIGN.md` exactly: the ink scale, terracotta + clay, moss / ochre / rust functional signals, paper / canvas surfaces, 8px spacing, three radii, three motions, the type scale (xs / sm / md / lg / xl / 2xl), and the primitive component classes (`.btn`, `.card`, `.pill`).

## What these are

Visual reference for engineering and design partners. They are **specs, not production code**. When primitives land in `packages/ui/`, they should match these mockups byte-for-token. If the mockup drifts from `DESIGN.md`, `DESIGN.md` wins, then update the mockup.

## What these are not

- Not Figma. We don't keep two sources of truth.
- Not interactive — buttons are styled, not wired.
- Not localized — copy is DE only here. FR / IT happen in `packages/i18n/locales/`.
- Not a styleguide page. The styleguide lives in `DESIGN.md`. These are the screens that *use* it.

## Open one

Open the file in any browser:

```bash
open mockups/owner-dashboard.html
open mockups/client-home.html
open mockups/worker-today.html
```

The fonts (`Inter Tight`, `Source Serif 4`, `JetBrains Mono`) are referenced by name only — your browser will fall back to `ui-sans-serif` / `ui-serif` / `ui-monospace`. In production the app self-hosts them via `next/font` (see `DESIGN.md` → Type system).

## Update policy

When a slice spec changes a screen materially (adds a region, removes a component, changes a color rule), update the corresponding mockup in the same PR. CI does not enforce this — it's a discipline thing. Slip and the mockups become aspirational; nobody trusts them.

`worker-today.html` simulates the PWA standalone experience (status bar row, safe-area-inset padding on the bottom tab bar, 420px frame for desktop preview, full-bleed under 480px).
