# Design System

The design source of truth. Every slice references back here. If a slice introduces a token, color, or pattern not in this file, the slice is wrong, not this file. See `slices/NN-name.md` for slice-level UI scope; `domains/i18n.md` for language rules.

## Brand posture

Swiss boarding-stable software, not fintech. The product is the digital backbone of a barn — pragmatic, calm, precise, a little tactile. Aspires to feel like a well-organized clipboard, not a Silicon Valley dashboard. Voice is competent and unfussy in DE / FR / IT. No exclamation marks. No emoji. No "Welcome aboard!" copy.

Three audiences, one system, three different densities:

| Surface | Audience | Density | Primary device |
|---|---|---|---|
| `/owner/*` | Stable manager | High (data-dense, multi-pane) | Desktop |
| `/client/*` | Horse owner / boarder | Medium (calm, scan-first) | Desktop + mobile |
| `/worker/*` | Groom / barn staff | Low (one-action-per-screen, gloves on) | Mobile-first PWA |

What we are **not**: a generic SaaS card grid. A purple-gradient dashboard. A celebration of features. A consumer app pretending barns are an Instagram aesthetic.

## Type system

- **Sans (UI + body)**: `Inter Tight` — clean, neutral, handles DE/FR/IT diacritics including `ß ç à ò`. Variable font, weights 400 / 500 / 600. Self-hosted via `next/font`, no Google Fonts CDN.
- **Serif (display, restrained use)**: `Source Serif 4`. Used only for invoice headers, receipt numbers, and the in-app stable name plaque. Adds Swiss-craft warmth without a marketing-page feel.
- **Mono (numbers, IDs, QRR)**: `JetBrains Mono`. Only for invoice numbers, QRR references, IBANs, time-clock readouts. Tabular figures.

Scale (4-step harmonic, base 16):

```
xs   12 / 16   meta, audit timestamps
sm   14 / 20   secondary labels
md   16 / 24   body, default UI
lg   20 / 28   page headers, key totals
xl   28 / 36   dashboard headline numbers
2xl  40 / 48   invoice grand total only
```

No font stacks of `system-ui, sans-serif`. Variable Inter Tight is loaded — if it fails to load we fall back to `ui-sans-serif`, never to `Arial` or `Roboto`. Numbers in tables: tabular-nums + right-aligned.

## Color system

Neutral-warm spine, one earth accent, two functional signals. CSS variables only. No raw hex in components.

```
--ink-900   #1B1814   near-black, primary text
--ink-700   #3A3530   secondary text
--ink-500   #6B635B   tertiary, captions
--ink-300   #B8B0A6   borders, dividers
--ink-100   #E8E2D8   hairlines, deep surface
--paper     #F7F3EC   default page background (warm off-white)
--canvas    #FFFFFF   card surface, modal
--terracotta-600  #B5532E   accent, "your horse" / primary CTA
--terracotta-700  #973F1F   pressed state
--clay-200  #F1DCC9   accent surface, calendar event fill
--moss-600  #4F6B3A   success, paid status
--moss-200  #DDE7C9   success surface
--ochre-600 #B57E1C   warning, overdue, weak signal
--ochre-200 #F2DFAB   warning surface
--rust-600  #9C3520   error, failed, lost signal
--rust-200  #EFC9BD   error surface
--ink-overlay  rgba(27,24,20,0.55)  modal backdrop
```

**Calendar event colors** (slice 03): own horse = `--terracotta-600`, stable-public = `--ink-500`, bookable open slot = `--moss-600` dashed. No more colors than these three on the calendar.

**Forbidden**: blue (any), purple (any), gradients on UI surfaces (decorative blobs, hero washes), pure `#000000` text, pure `#FFFFFF` page background.

Contrast: every text-on-surface combo is verified ≥ AA in `packages/ui/tokens.test.ts` (a contrast script, not a vibe).

## Spacing and layout

8px grid. Spacing tokens: `space-1` (4) `space-2` (8) `space-3` (12) `space-4` (16) `space-6` (24) `space-8` (32) `space-12` (48) `space-16` (64). Nothing in between.

Radius: `radius-sm` (4) for inputs, `radius-md` (8) for cards, `radius-lg` (12) for modals only. Same radius on every surface = AI slop. Different radii at different layers, by purpose.

Owner desktop: 12-column grid, 24px gutter, 1280px max content width with two-pane layouts where the left pane is fixed 320px navigation rail. Client desktop: same 12-col, single-pane, 960px max read width. Worker mobile: single column, 16px page padding, full-bleed cards on small screens.

Breakpoints: `sm 640` `md 768` `lg 1024` `xl 1280`. Worker only renders the `sm`/`md` versions; owner only renders `lg`/`xl`; client renders all.

## Motion

Three motions, total. More than three = too much.

- **Tap response** (140ms ease-out): button press, check circle fill, tab switch.
- **Reveal** (220ms ease-out): drawer / modal enter, toast slide-in, dropdown open. Reverse for exit at 180ms ease-in.
- **List reorder** (260ms ease-in-out): task slides to "done" section, calendar event drag.

No parallax. No scroll-linked hero. No decorative looping animation. `prefers-reduced-motion: reduce` cuts everything to opacity-only crossfades at 100ms.

## Component primitives

Defined in `packages/ui/primitives/`. Slices may compose, never re-skin.

- `Button` — `primary` (terracotta) / `secondary` (ink-100 fill) / `ghost` (text only) / `destructive` (rust). 36px / 44px / 52px heights. 44px is the worker default.
- `Field` — text, number, date, time, select, multi-select. Always paired with `Label` and `HelpText`. Error state uses `rust-600` border + `rust-600` text below, never red glow.
- `Card` — `radius-md`, `--canvas` surface, `1px solid --ink-100` border. No drop shadows on cards. Elevation by hairline border, not by shadow.
- `Modal` / `Drawer` — `radius-lg` modal, full-height drawer from right (desktop) or bottom (mobile). One soft shadow only on these two.
- `TabBar` (worker bottom nav, slice 12) — 4 items max, 56px tall + safe-area inset, terracotta active indicator.
- `Pill` — status chips: `paid` (moss), `overdue` (ochre), `draft` (ink-300), `sent` (ink-500), `cancelled` (rust outline only).
- `EmptyState` — illustration slot + headline + body + primary action. Never "No items found." See empty-state spec below.
- `Toast` — top-right desktop, top mobile. Auto-dismiss 4s default, 8s for warnings, sticky for errors with retry.

Cards earn their existence. If the only thing inside a card is a label + a number, it's not a card — it's a stat. Reach for `Card` only when the unit is a meaningful boundary (one invoice, one horse, one task).

## Empty / loading / error states

Every list, table, and async region specifies all four:

| State | Owner | Client | Worker |
|---|---|---|---|
| Loading | Skeleton matching final layout, no spinner soup | Same | Same |
| Empty (first-run) | Illustration + "No invoices yet. Confirm a draft to send the first one." + primary CTA | Calmer copy ("Your invoices show up here once your stable sends them.") | "Nothing scheduled. Take a coffee." (DE/FR/IT) |
| Empty (filtered) | "No matches. Clear filters." | Same | Same |
| Error | Inline `rust-600` banner with one-line cause + retry button. Sentry tag attached. Never blank screen + 500. | Same | Same plus offline indicator if `navigator.onLine === false` |

## Voice and copy

- DE is canonical. FR and IT translate from DE, not from English. Translation memory in `packages/i18n/locales/`.
- Numbers: Swiss formatting (`1'234.50 CHF`) via `Intl.NumberFormat('de-CH')`. Never `1,234.50` or `$`.
- Dates: `DD.MM.YYYY` always (Swiss convention), times `HH:mm` (24h). No "yesterday at 4pm" relative phrasing in invoices, ever — but ok in worker chat.
- Tone per surface: owner = factual ("Invoice 2026-0142 sent"), client = warm ("Your invoice for May is ready"), worker = imperative + short ("Feed Aisle A · 07:00").
- Forbidden phrases: "Welcome to ...", "Unlock the power of ...", "Your all-in-one ...", "Let's get started!", any exclamation mark in production copy.

## Iconography

`lucide-react`, 1.5px stroke, never filled. No icon-in-colored-circle decorations (slot 3 of the AI-slop blacklist). Icons live next to labels at `md` size; standalone icons (e.g. tab bar) at `lg` size with an `aria-label`.

## Imagery

No stock photos. No horse-running-on-beach. The product never ships marketing imagery in the app surface. Stable logo upload is the only customer-side image; it sits in a `radius-md` mask at 48px in nav, 96px on invoice header.

## Accessibility

- All interactive targets ≥ 44 × 44 px (worker), ≥ 36 × 36 px (owner desktop, where pointer is mouse).
- Focus visible: 2px `terracotta-600` outline, 2px offset, never removed.
- Keyboard: every action reachable without a mouse. Owner uses `cmd-k` quick-search.
- ARIA: every modal traps focus + restores on close. Every async region has `aria-live="polite"`.
- Color is never the only signal — pills carry icon + label, not just hue.
- Tested with VoiceOver (Safari/iOS), NVDA (Windows), TalkBack (Android) before pilot.
- Contrast: AA minimum on body, AAA on numeric data in tables.

## Localization design rules

DE strings are typically 30% longer than EN; FR is 20% longer than DE; IT is similar to FR. Components must accommodate without truncation:

- Buttons: min-width auto, never fixed width. Icon-only buttons get `aria-label` per locale.
- Tables: column widths flex, never fixed pixel.
- Numbers: use locale-aware formatting from `Intl`, not custom string concat.

## Out of scope for V1

- Dark mode (V1.1)
- A custom illustration set (V1.1 — V1 ships with two pen-line emptiness illustrations: "no horses yet" and "no events today", drawn by Ferdinand)
- Brand photography (V2)
- Marketing site (`lafattoria.app/` is a single landing page in V1; full site V1.1)
- White-label theme switching (V2 — V1 is single-tenant La Fattoria-flavored)
