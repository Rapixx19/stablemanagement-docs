# Runbook: Bexio Token Recovery

When Bexio OAuth tokens are unrecoverable, suspected leaked, or the encryption key needs to rotate. Three procedures, in order of escalation. Pick the one that matches the actual failure mode — don't escalate by default, each procedure costs the affected stables real friction.

**Owner**: Sharad. **Reviewed by**: Ferdinand. **Touch frequency expected**: 0 in V1, but rehearsed once on staging before pilot so we know the SQL and the comms templates exist.

## Decision tree

```
Did pgcrypto key leak / get rotated?
├─ Yes → Procedure 1 (key rotation, re-encrypt in place)
│        Tokens are still valid; only the wrapper changes.
│
Did Bexio say tokens are revoked / suspect token compromise?
├─ Yes → Procedure 2 (force re-auth all tenants)
│        Tokens are dead; users must re-OAuth.
│
Did we lose the pgcrypto key entirely (no backup)?
└─ Yes → Procedure 3 (recover from key loss)
         Tokens are unreadable; users must re-OAuth, but we
         also have to scrub the encrypted ciphertext.
```

## Pre-flight (run before any procedure)

1. Acknowledge in `#stable-prod` that you are starting a Bexio token recovery, name the procedure number.
2. Snapshot `bexio_connections` to a timestamped table:
   ```sql
   create table bexio_connections_snapshot_YYYYMMDD as
     select * from bexio_connections;
   ```
3. Pause Vercel Cron for `/api/cron/bexio-poll` (set the schedule to disabled in `vercel.json` and redeploy, or remove the cron job from the dashboard). This prevents in-flight polls from racing the recovery.
4. Note start time. All three procedures are reversible up to the moment we DELETE/UPDATE rows in `bexio_connections`.

---

## Procedure 1 — pgcrypto key rotation

Use when: scheduled annual rotation, suspected key compromise without token revocation, or when adding a new on-call engineer who shouldn't have the old key.

The plain-text tokens themselves are still valid on Bexio's side. We only swap the wrapping key.

### Steps

1. Generate the new key (32 bytes, base64):
   ```bash
   openssl rand -base64 32
   ```
2. Add to Vercel env as `BEXIO_TOKEN_ENCRYPTION_KEY_NEXT` (don't replace the live one yet) and to Supabase Vault as `bexio_token_encryption_key_next`. Redeploy so both are loaded.
3. Re-encrypt every row, in batches of 100, transactionally. Run from a `psql` session connected via the direct (non-pooled) URL with the service role:
   ```sql
   begin;
   set local app.crypto_key_old = '<old-key>';
   set local app.crypto_key_new = '<new-key>';
   with batch as (
     select id, oauth_access_token_encrypted, oauth_refresh_token_encrypted
     from bexio_connections
     where active = true
     order by id
     limit 100
     for update
   )
   update bexio_connections b set
     oauth_access_token_encrypted = pgp_sym_encrypt(
       pgp_sym_decrypt(batch.oauth_access_token_encrypted::bytea, current_setting('app.crypto_key_old')),
       current_setting('app.crypto_key_new')
     ),
     oauth_refresh_token_encrypted = pgp_sym_encrypt(
       pgp_sym_decrypt(batch.oauth_refresh_token_encrypted::bytea, current_setting('app.crypto_key_old')),
       current_setting('app.crypto_key_new')
     ),
     updated_at = now()
   from batch
   where b.id = batch.id;
   commit;
   ```
   Repeat the batch until 0 rows updated.
4. Smoke-test: pick one stable, decrypt with the new key, hit Bexio `/2.0/contact` with the resulting access token, expect 200.
5. Promote: set `BEXIO_TOKEN_ENCRYPTION_KEY = BEXIO_TOKEN_ENCRYPTION_KEY_NEXT` in Vercel + Supabase Vault. Redeploy. Drop the `_NEXT` env var.
6. Re-enable the Vercel Cron poll. Watch `bexio_sync_log` for 30 min — any decrypt failure surfaces here.
7. Drop the snapshot table only after 7 days clean.
8. Audit-log a `key_rotation` event with reason and engineer.

Expected total time: 15 min for 30 stables. Linear in stable count.

---

## Procedure 2 — Force re-auth all tenants

Use when: Bexio confirms tokens are revoked, we detect anomalous Bexio API activity, or a current/former team member with key access leaves under non-trivial circumstances.

The tokens are dead either way. The job is to tell users to re-OAuth and clean up gracefully.

### Steps

1. NULL out tokens, mark connections inactive, but keep the `bexio_company_id` so re-auth can re-link to the same Bexio tenant:
   ```sql
   begin;
   update bexio_connections set
     oauth_access_token_encrypted = null,
     oauth_refresh_token_encrypted = null,
     oauth_expires_at = now(),
     active = false,
     last_error = 'force_reauth_' || to_char(now(), 'YYYY-MM-DD'),
     updated_at = now();
   commit;
   ```
2. Email every owner of an affected stable using the templated `bexio_reauth_required` Resend template (DE/FR/IT — content lives in `packages/i18n/locales/*/emails.ts`). Subject: "Bexio-Verbindung erneut bestätigen". Body explains: nothing leaked from your data, our routine security rotation requires you to reconnect, click here.
3. The owner reconnect flow at `/settings/integrations/bexio` already handles the case of inactive + null tokens — it re-runs the OAuth authorization-code grant, writes new encrypted tokens, sets `active = true`. Verified in slice 09 acceptance.
4. Re-enable Vercel Cron poll. Polls for inactive connections are already a no-op.
5. Track recovery: dashboard query (or one-off `psql`):
   ```sql
   select count(*) filter (where active) as reconnected,
          count(*) filter (where not active) as still_pending
   from bexio_connections;
   ```
   Page Ferdinand if `still_pending > 0` after 72h. He calls those owners directly.
6. Audit-log a `force_reauth` event with the reason and the count of affected connections.

Expected total time: 30 min to flip the rows + send emails. 24–72h for owners to reconnect.

---

## Procedure 3 — Recover from key loss

Use when: the pgcrypto key is genuinely gone (no Vercel env, no Vault, no 1Password backup, no team member's password manager). The encrypted tokens cannot be decrypted by anyone, including us.

This is the worst case. Worse than Procedure 2 because we cannot even gracefully revoke the old tokens on Bexio's side — we don't have the access tokens to call `/oauth/revoke`. They will eventually time out (1h access, 30 days refresh).

### Steps

1. Generate a new pgcrypto key per Procedure 1, step 1. Set as `BEXIO_TOKEN_ENCRYPTION_KEY` in Vercel + Vault. Redeploy.
2. Hard-clear the encrypted columns. They're unreadable garbage now anyway:
   ```sql
   begin;
   update bexio_connections set
     oauth_access_token_encrypted = null,
     oauth_refresh_token_encrypted = null,
     oauth_expires_at = now(),
     active = false,
     last_error = 'key_loss_recovery_' || to_char(now(), 'YYYY-MM-DD'),
     updated_at = now();
   commit;
   ```
3. Contact Bexio support directly. Provide the list of affected `bexio_company_id` values. Ask them to revoke our OAuth grants on those tenants ahead of the natural 30-day refresh expiry. They are usually responsive within 1 business day.
4. Email all affected owners using the `bexio_reauth_required` template, plus a manually-edited cover note acknowledging this was caused by an internal key-management failure on our side, no data leaked from their stable, and offering a one-month free credit on V1.1 paid tier.
5. Same dashboard tracking as Procedure 2. Same Ferdinand-calls-laggards rule, but at 24h not 72h, because reputational risk is higher here.
6. Audit-log a `key_loss_recovery` event. Required: post-mortem in `docs/post-mortems/YYYY-MM-DD-bexio-key-loss.md` within 48h. Include: how the key was lost, what we changed in the runbook, what we changed in CI / vault config, who reviewed.

Expected total time: 1h infra work + 1–2 business days of comms. Reputational cost is the real cost.

---

## Post-recovery (run after any procedure)

1. Re-enable the Vercel Cron poll if you paused it.
2. Drop snapshot tables after 7 days clean (Procedure 1) or 30 days (Procedures 2 + 3).
3. Sentry: check for any spike in `bexio.oauth.refresh.failed` or `bexio.poll.error` in the next 24h. Investigate any cluster of stable_ids.
4. Update `last_error = null` on now-healthy `bexio_connections` rows.
5. If the cause was preventable, file a TODO in `TODOS.md` under "post-pilot V1.0 → V1.1" or add a new pre-slice item if it's slice-00-grade.
6. If the cause was a genuine incident (Procedure 2 or 3), Ferdinand writes a customer-facing transparency post on `lafattoria.app/blog` within 7 days. Builds the trust-bank we need for the Walter-network sales motion.

## What we never do

- Email a key value
- Slack a key value
- Re-use a rotated key for any reason
- Skip the audit log entry
- Skip the post-mortem on Procedure 3
- Tell a customer "your data was at risk" when it wasn't (and conversely, never minimize when it was)

## Test schedule

- Procedure 1: rehearsed once on staging during slice 09 acceptance. Re-rehearsed annually thereafter.
- Procedures 2 and 3: walked-through in conversation between Sharad + Ferdinand pre-pilot. Not rehearsed against live data — staging-only fire drill once before V1.0 GA.
