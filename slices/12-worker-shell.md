# Slice 12 — Worker App Shell and Auth

**Phase:** Worker · **Estimate:** 3 dev-days · **Owner:** Sharad backend, freelancer frontend

The third face of the SaaS. Mobile-first PWA scoped to `/worker/*` of the single Next.js app at `app.lafattoria.app`. Once installed, the worker app appears as a standalone "Stable Crew" icon on the phone home screen — the user never sees it living under a shared host.

## Goal

Marco the groom can install the web app to his phone home screen, log in with magic link, see his today view, and navigate Home / Schedule / Tasks / Chat. Same shared auth + RLS backbone as Owner and Client. Worker role membership added in slice 00 schema.

## What ships

1. **`/worker/*` route tree** in the single `apps/web` Next.js app. Mobile-first responsive shell, isolated layout, separate manifest scope.
2. **Worker home** (mobile layout per `DESIGN.md` worker density: single column, 16px page padding, 44px tap targets, paper background, terracotta primary CTA). Greeting + clock card stub (slice 14 makes it functional) + today's tasks (slice 13) + week strip (slice 15) + quick actions + bottom tab bar.
3. **Magic-link auth** with worker role check. If a logged-in user has no `worker` membership at the stable they're trying to access, they get an explicit "you're not a worker here" screen, not a generic 403.
4. **Onboarding consent step.** First login as a worker presents a one-screen consent: "This app records your shifts and clock-in times. Geofence verifies you're on-site at the stable. We do not store raw GPS coordinates. Click I understand to continue." DE/FR/IT. Persisted in `worker_consent` table.
5. **Bottom tab bar** with Home / Schedule / Tasks / Chat. Active state. Safe-area inset support (iOS).
6. **PWA install** scoped to `/worker/*`. Manifest at `/worker/manifest.webmanifest` with `scope: /worker/`, `start_url: /worker/home`, `display: standalone`, `name: "Stable Crew"`, `theme_color`, `background_color`, full icon set (192/256/384/512 + maskable). Service worker registered only inside the `/worker/*` tree so it cannot interfere with `/owner` or `/client`. Android Chrome shows the native install prompt; we surface our own button as a fallback.
7. **iOS install instructions screen.** iOS Safari does not fire `beforeinstallprompt` — users must tap Share → Add to Home Screen manually. We detect iOS Safari and show a one-screen tutorial with arrows pointing at the Share icon, three labelled steps, all in DE/FR/IT. Dismissable with "I've installed it / Skip for now". Re-shown on next session if not dismissed-as-installed. Verified on iOS 16, 17, 18 in BrowserStack before pilot.
8. **Push notifications** stubbed for V2 (web push requires HTTPS + browser permission flow; we ship a banner instead in V1: "open the app to see new tasks").

## Schema diff

```sql
-- 012_worker_consent.sql
create table worker_consent (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  user_id uuid not null references auth.users(id),
  consent_version int not null,
  consent_text_hash text not null,             -- audit trail of what they agreed to
  language text not null,
  ip_address inet,
  user_agent text,
  agreed_at timestamptz not null default now(),
  unique (stable_id, user_id, consent_version)
);
```

## RLS

- `worker_consent`: a worker can SELECT their own; owner/manager can SELECT all in stable
- Membership check on every server action: `requireWorkerMembership(ctx, stableId)`

## Mobile-first design notes

- Tap targets minimum 44 × 44 px (iOS Human Interface Guidelines)
- Bottom tab bar always visible above safe-area
- Avoid hover-only affordances
- One-thumb reachable: primary actions in bottom 60% of screen
- Page transitions snappy (no full reloads); use Next.js client navigation
- Visual reference: `mockups/worker-today.html` (token-calibrated to `DESIGN.md` — worker density column, TabBar primitive, motion vocabulary)

## Performance budgets

The barn is a marginal-connectivity environment. Workers are on whatever 4G the dairy barn's metal roof allows. Hard budgets, asserted in CI:

- **Initial JS bundle for `/worker/home`**: < 150KB gzipped. Tracked via `next-bundle-analyzer` snapshot, fails CI on any PR that pushes it over. RSC + careful client/server split is non-negotiable here — no client-side date libraries (use `Intl.DateTimeFormat`), no full Tailwind CSS in the worker bundle (purged tightly), no heavy state libraries (Zustand only if justified).
- **Lighthouse mobile score** on `/worker/home` ≥ 90 across Performance, Accessibility, Best Practices, SEO. CI runs Lighthouse against a Vercel preview build with mobile preset.
- **LCP < 2.5s on simulated slow 4G** (Lighthouse "Slow 4G" preset = 400ms RTT, 400Kbps down, 4× CPU slowdown). Asserted in CI on the same Lighthouse run.
- **TBT < 200ms** under the same throttling.
- **Service-worker cold install** ships in < 200KB total (precache shell only — no aggressive precaching of route data).
- Assertions live in `apps/web/tests/perf/worker-budget.test.ts` and run on every PR that touches `app/worker/**` or shared packages.

## Acceptance criteria

- [ ] Worker logs in via magic link; arrives on home view
- [ ] User without worker membership at this stable sees clear "not a worker here" screen
- [ ] First-time login shows consent screen; persists agreement; subsequent logins skip
- [ ] Consent text in DE/FR/IT per worker preference
- [ ] PWA installable on iOS Safari + Android Chrome from `/worker/*` only (visiting `/owner` or `/client` does not register the service worker, no install prompt)
- [ ] **iOS install screen** appears on iOS Safari first-visit, has Share-icon arrow, three steps, DE/FR/IT, dismissable; verified on iOS 16/17/18 in BrowserStack
- [ ] Tab bar safe-area-inset-bottom respected (no overlap with home indicator)
- [ ] All 4 tabs route correctly even when their content slices haven't shipped (placeholders)
- [ ] **Bundle budget**: `/worker/home` initial JS < 150KB gzipped, asserted in CI
- [ ] **Lighthouse mobile** on `/worker/home` against slow-4G preset: LCP < 2.5s, TBT < 200ms, score ≥ 90 across all four categories
- [ ] **Manifest scope isolation**: requesting `/owner` from an installed worker PWA opens in Safari (out of scope), not the standalone window

## Acceptance integration test

`apps/worker/tests/integration/auth.test.ts`

```ts
test('worker without membership at stable cannot access', async () => {
  const { user } = await seed.userWithoutMembership();
  const res = await fetchAsUser(user, '/home', { stableId: 'someStable' });
  expect(res.status).toBe(403);
  expect(res.body).toContain('not a worker'); // localized
});

test('first login records consent', async () => {
  const { worker, ctx } = await seed.workerFirstLogin();
  await acceptConsent(ctx);
  const consent = await getConsent(ctx);
  expect(consent.consent_version).toBe(1);
});
```

## Out of scope

- Native iOS / Android wrapper via Capacitor (V2)
- Push notifications (V2)
- Offline mode (V2)
- Biometric login (V2)
