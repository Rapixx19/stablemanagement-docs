# Runbook: Cron Secret Rotation

Rotates `CRON_SECRET` — the bearer token authenticating every `/api/cron/*` route handler (per `DECISIONS.md` D12). Locked in `reviews/2026-05-07-ceo-review.md` as SEC-3.

## When to rotate

- **Suspected leak** (logs grep hit, leaked screenshot, ex-engineer offboarding) → rotate immediately
- **Scheduled** → once per quarter; calendar reminder on Sharad's calendar
- **After incident** that touched cron infrastructure

## Pre-rotation checks

1. Confirm cron jobs are healthy via `cron_runs` table — last 7 days no failures
2. Check `#stable-deploys` for any in-flight deploy; pause this rotation if a deploy is mid-flight
3. Notify `#stable-warn`: "Rotating CRON_SECRET in 10 min — cron handlers will accept both old and new secrets during the swap; no downtime expected."

## The procedure

```bash
# 1. Generate the new secret (32 bytes, base64)
openssl rand -base64 32
# capture output as NEW_SECRET
```

```
2. Add NEW_SECRET to Vercel as a NEW env var named CRON_SECRET_NEXT
   Vercel → Settings → Environment Variables → Add
   Type: Sensitive
   Environments: Production + Preview + Development

3. Deploy a code change that accepts EITHER CRON_SECRET or CRON_SECRET_NEXT
   in the bearer-check helper at packages/lib/auth/cron-auth.ts
   PR title: "rotation: accept both CRON_SECRET values (rollback-safe)"

4. Confirm deploy succeeded and crons are landing
   - Check cron_runs table for new entries in the last 5 min
   - Pino log "cron auth: ok" still firing

5. Update vercel.ts to send the NEW secret as the bearer header
   This redeploys; Vercel Cron picks up new env on its next firing
   (Vercel Cron sends the `CRON_SECRET` env value as the bearer — pin the var name in vercel.ts to CRON_SECRET_NEXT temporarily.)

6. Wait 1 full hour. Confirm:
   - cron_runs entries with status='success' for the last 1h
   - Sentry shows no cron 401 errors

7. Remove CRON_SECRET (old) from Vercel env vars
   Rename CRON_SECRET_NEXT → CRON_SECRET (Vercel allows the rename, or remove + re-add)
   Revert the helper PR from step 3 (helper accepts only one env again)
   Revert vercel.ts to read CRON_SECRET (the canonical name)
   Deploy

8. Confirm crons still succeed for another 1h
```

## Verification

- All cron entries in `cron_runs` for the last 1h show `status='success'`
- No 401s in `/api/cron/*` access logs
- Sentry shows no cron-related errors
- Manual smoke: `curl -H "Authorization: Bearer $NEW_SECRET" $APP/api/cron/email-drain` returns 200

## If rotation fails

- **Step 3 deploy fails** → rollback via `runbooks/deploy.md`. Old secret still works; rotation aborted, retry later.
- **Step 5 deploy fails** (Vercel Cron config rejected) → revert `vercel.ts`. Old secret still works. Diagnose and retry.
- **Step 6 crons failing** → rollback the step-5 deploy. The helper from step 3 accepts both, so neither secret breaks anything. Investigate before retrying.

## What this doesn't cover

- `BEXIO_TOKEN_ENCRYPTION_KEY` rotation — see `runbooks/bexio-token-recovery.md` which includes key rotation
- Supabase service-role keys — rotate via Supabase dashboard; separate flow, requires app redeploy
- Sentry DSN — non-secret; only rotate if abused

## Documenting the rotation

After step 8, append to this runbook's "Rotation history" section below:

```
- YYYY-MM-DD by {operator} — reason: {scheduled | suspected-leak | post-incident}
```

## Rotation history

- {none yet — first rotation will be the post-pilot quarterly}

## When this runbook updates

- The auth helper API changes → update step 3 wording
- Vercel Cron config UX changes → update step 5
- New cron added that uses a different secret (none planned in V1) → fork this runbook
