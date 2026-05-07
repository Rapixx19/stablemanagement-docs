# Domain: Secrets and OAuth Token Management

Where secrets live, how OAuth tokens are stored, the unbreakable rules.

## Where secrets live

- **Vercel** environment variables for the single Next.js app (`apps/web`), per env: dev/preview/prod. Vercel Cron jobs read from the same env, no separate secrets store.
- **Supabase Vault** for any secret that pg_cron jobs need to reference inside Postgres (e.g. crypto key passthrough). pg_cron pulls from Vault rather than Vercel env.
- **Local dev**: `.env.local` — gitignored, never committed
- **GitHub Actions**: repository secrets, only injected for prod-relevant tasks
- **No Railway, no FastAPI.** V1 has no Python service. If V1.5 adds an ML service, we revisit secret storage at that point.

## Secret categories

| Type | Where | Rotation |
|---|---|---|
| Supabase `anon` key | Public (in browser) | When project is rebuilt |
| Supabase `service_role` key | Server only, Vercel | Quarterly + on incident |
| Bexio OAuth client secret | Server only | When Bexio rotates |
| Resend API key | Server only | When team member departs |
| Anthropic API key (catalog AI) | Server only | When team member departs |
| Sentry DSN | Server + browser (browser DSN only) | Project-bound |
| Pgcrypto symmetric key | Server only | Annually + on incident |

## Pgcrypto key for OAuth tokens

Bexio OAuth tokens (and any future OAuth secrets) are stored in Postgres encrypted with `pgcrypto` in **`bytea` columns** (per `DECISIONS.md` D8). The symmetric key (`BEXIO_TOKEN_ENCRYPTION_KEY`) lives in Vercel env, marked Sensitive.

**Pattern (must be inside one transaction).** Transaction-mode pooler discards session state between transactions, so `set_config` must be `true` (txn-local) and inside the same TX as the decrypt:

```sql
BEGIN;
  SELECT set_config('app.crypto_key', $1, true);   -- true = txn-local
  -- encrypt path:
  UPDATE bexio_connections
     SET oauth_access_token = pgp_sym_encrypt($2, current_setting('app.crypto_key'))
   WHERE stable_id = $3;
  -- decrypt path:
  SELECT pgp_sym_decrypt(oauth_access_token, current_setting('app.crypto_key'))
    FROM bexio_connections WHERE stable_id = $3;
COMMIT;
```

The key is passed in by the server action on every call, never hardcoded, never in client logs.

**Rotation, force-reauth, and key-loss recovery procedures**: see `runbooks/bexio-token-recovery.md`.

## Token refresh flow

Bexio access token expires in 1h. Refresh token in 30 days.

- Every server action using Bexio first checks `oauth_expires_at`
- If < 5 min remaining: refresh via OAuth, update both encrypted fields, retry
- If refresh fails (token revoked, etc.): mark connection inactive, surface to owner UI

## Vercel env vars (single app)

Vercel (`apps/web`) needs:
- `NEXT_PUBLIC_SUPABASE_URL`
- `NEXT_PUBLIC_SUPABASE_ANON_KEY`
- `SUPABASE_SERVICE_ROLE_KEY` (server-side only — Vercel scopes `NEXT_PUBLIC_*` to the browser bundle and the rest to the server runtime)
- `SUPABASE_DB_DIRECT_URL` (direct connection, used by migrations only)
- `SUPABASE_DB_POOLER_URL` (transaction-mode pooler, used by app traffic)
- `RESEND_API_KEY`
- `ANTHROPIC_API_KEY`
- `SENTRY_DSN`, `NEXT_PUBLIC_SENTRY_DSN`
- `BEXIO_CLIENT_ID`, `BEXIO_CLIENT_SECRET`
- `BEXIO_TOKEN_ENCRYPTION_KEY` (pgcrypto symmetric key)
- `CRON_SECRET` — shared header secret used by Vercel Cron to authenticate calls to `/api/cron/*` (Bexio reconciliation, email digest)

## Cron authentication

- **Vercel Cron** routes (`/api/cron/{email-drain,task-materialisation,bexio-poll,bexio-keepalive,request-expiry,auto-punch-out}`) verify `Authorization: Bearer ${CRON_SECRET}` on every invocation. Handlers reject non-cron requests with 401. Single shared `CRON_SECRET` per env (per `DECISIONS.md` D12).
- **pg_cron** jobs that stay pure SQL (subscription run, audit partition rotation) live entirely inside Postgres and don't need an HTTP secret. RRULE expansion + email send + external API calls live in Vercel Cron Node handlers.

## CI / GitHub Actions

Secrets injected only when needed for prod tasks. Test runs use mock secrets.

Required for CI:
- `SUPABASE_TEST_DB_URL` (ephemeral test DB)
- `SIX_VALIDATOR_TOKEN` (for QR-bill PDF validation in CI)

## Rotation procedure

Documented in `runbooks/disaster-recovery.md`. Summary:

1. Generate new secret in provider
2. Update Vercel env vars + Supabase Vault entry where applicable (requires redeploy)
3. Verify with smoke test
4. Revoke old secret
5. Audit log entry: who, when, why

For `BEXIO_TOKEN_ENCRYPTION_KEY` specifically: rotation requires re-encrypting every row in `bexio_connections`. Procedure in `runbooks/bexio-token-recovery.md` (key rotation section).

## What we never do

- Commit `.env*` files (gitignored, also pre-commit hook check)
- Log a secret value (logger has a known-secret-substring filter)
- Pass `service_role` key to a browser
- Embed a secret in a Vercel `NEXT_PUBLIC_*` env var
- Hardcode a fallback key in code "for emergencies"
- Share secrets in Slack — use 1Password
- Email or SMS a secret

## What we always do

- Treat every secret as compromise-eligible
- Rotate quarterly even without incident
- Log secret-using actions with the secret name (not value): `bexio.oauth.refresh succeeded`
- Encrypt OAuth tokens at rest
- Set the shortest viable token expiry (1h access, 30-day refresh)
- Use HTTPS-only domains everywhere

## Secret leakage incident response

If a secret is suspected leaked:
1. Immediately rotate the secret
2. Revoke all sessions that used it (where applicable)
3. Audit-log search for unusual usage in the affected window
4. Notify Ferdinand
5. Post-mortem in `docs/post-mortems/`

For Bexio token leak specifically: revoke the OAuth grant on Bexio's side first, then rotate our client secret.

## Secret detection in CI

Pre-commit hook + GitHub Actions step uses `gitleaks` to scan diffs for secret patterns. Blocks merge.

## Local development

Each developer has their own Supabase project (Free tier). Their `.env.local` connects to it. Never share `.env.local`s.

For Bexio testing: the team has a sandbox account with throwaway credentials in 1Password.
