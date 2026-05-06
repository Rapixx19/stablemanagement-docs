# Mockups

Three hero screens calibrated to `DESIGN.md`. Static HTML with inline CSS, sharing one tokens file.

| File | Surface | Audience | Density | Slices it anchors |
|---|---|---|---|---|
| `owner-dashboard.html` | `/owner` | Stable manager | High (12-col, 320px nav rail, 1280px max) | 03, 06, 08, 09, 16 |
| `client-home.html` | `/client` | Horse owner | Medium (12-col, 960px max read width) | 03, 06, 08, 11, 16 |
| `worker-today.html` | `/worker` | Groom | Low (single column, 16px padding, 44px tap rows, bottom tab bar) | 12, 13, 14, 15, 16 |

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
