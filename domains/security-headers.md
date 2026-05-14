# Domain: Security Headers

HTTP-header defenses. Locked in `reviews/2026-05-07-ceo-review.md` as SEC-2. CSP, HSTS preload, X-Frame, Permissions-Policy. Set in `apps/web/middleware.ts` so every response gets them.

## Baseline

Every response includes:

```ts
const headers = {
  'Strict-Transport-Security': 'max-age=63072000; includeSubDomains; preload',
  'X-Frame-Options': 'DENY',
  'X-Content-Type-Options': 'nosniff',
  'Referrer-Policy': 'strict-origin-when-cross-origin',
  'Permissions-Policy': 'camera=(), microphone=(), geolocation=(self), payment=(), usb=(), fullscreen=(self)',
  'Content-Security-Policy': cspString,
}
```

Apply via `apps/web/middleware.ts` matching `/(.*)`. Smoke test: `curl -I https://app.lafattoria.app` shows all six.

## Content-Security-Policy

Per-request nonce for inline scripts (Next.js 16 Server Components emit some). Generated in middleware, passed via header AND read in the layout for `<script nonce={x}>`.

```
default-src 'self';
script-src 'self' 'nonce-{NONCE}' https://browser.sentry-cdn.com;
style-src 'self' 'unsafe-inline';
img-src 'self' data: blob: https://{SUPABASE_PROJECT}.supabase.co;
connect-src 'self' https://{SUPABASE_PROJECT}.supabase.co https://o123456.ingest.sentry.io https://api.anthropic.com https://api.bexio.com;
font-src 'self';
frame-ancestors 'none';
form-action 'self';
base-uri 'self';
report-uri /api/csp-report;
```

`style-src 'unsafe-inline'` is unavoidable with Tailwind JIT in V1. Long-term: migrate to `style-src 'self' 'nonce-{NONCE}'` if Tailwind v4+ supports nonce-style injection.

## Why `geolocation=(self)`

Slice 14 (timeclock) needs `navigator.geolocation`. Restrict to first-party only. Camera / microphone / payment / USB are off — `domains/v1-deferred.md` includes photo-task-completion in V1.1; if that ships, this policy gets a camera entry.

## Worker PWA exception

`/worker/sw.ts` needs `Service-Worker-Allowed: /worker/` header. The service worker registration runs without nonces. Set in the route handler that serves `sw.ts`, not in middleware.

## HSTS preload submission

Before pilot launch: submit `lafattoria.app` to https://hstspreload.org/. Pre-warming + preload gives clean Chrome/Firefox/Safari behavior from the first visit. Once submitted, removal takes weeks — only do it once the domain is committed. Document the submission date in `runbooks/deploy.md`.

## CSP report-uri

`/api/csp-report` (slice 00) accepts the CSP report payload, logs to Pino with `stable_id` from auth context if present, and does not echo. Throttled to 100/min per IP (a misconfigured CSP can dump thousands of reports/min from one client). Reports surface to Sentry at >10/min sustained.

## Testing pattern

```ts
test('every response has the baseline headers', async () => {
  const res = await fetch(`${appUrl}/owner/dashboard`)
  expect(res.headers.get('strict-transport-security')).toMatch(/max-age=63072000/)
  expect(res.headers.get('x-frame-options')).toBe('DENY')
  expect(res.headers.get('content-security-policy')).toMatch(/default-src 'self'/)
})
```

CI snapshot of header output catches accidental removals.

## CSP rollout pattern

For the pilot: ship CSP in `Content-Security-Policy-Report-Only` mode for 1 week, monitor `/api/csp-report` for unexpected violations, then flip to enforcement. Documented in `runbooks/deploy.md`.

## Common mistakes

- Setting CSP per-route instead of globally → easy to forget on a new route
- Allowing `'unsafe-eval'` in `script-src` → defeats most of CSP
- Forgetting to include Sentry's CDN in `script-src` → Sentry stops loading silently
- Submitting HSTS preload before subdomains are HTTPS-ready → locks them out for ~6 months
- Missing the `nonce-{NONCE}` substitution in middleware → inline scripts blocked, app blank

## When this doc updates

- New third-party domain → update `connect-src` / `img-src` / `script-src`
- New permission needed (camera in V1.1?) → update Permissions-Policy
- Removing a third-party → schedule a CSP audit; can't quietly forget
- Tailwind v4+ ships nonce support → migrate `style-src` off `unsafe-inline`
