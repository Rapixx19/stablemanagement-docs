# Runbook: Auth Recovery

When an owner can't log in. V1 has no self-serve recovery; this is the manual procedure.

## Symptoms

- Owner reports "I'm not getting the magic link email"
- Owner says they changed email and can't access
- Owner says they lost access after a phone change
- Owner suspects account compromise

## Step 1: identify the actual problem

Ask the owner:
1. What email are you typing into the login screen?
2. Are you on the right URL? (`lafattoria.app`, not phishing)
3. Have you checked spam? (Resend emails sometimes land in `Promotions`)
4. Are you typing the link from one device on another? (If so, fine — the magic link is device-agnostic)

Most cases resolve here.

## Step 2: check our side

In Supabase dashboard:

1. `auth.users` — does the email exist? (case-insensitive)
2. `memberships` — does this user have an `owner` role at the stable?
3. Better Stack logs: search for `magic_link.send action` with this email in last 24h
4. Resend dashboard: did the email send? Did it bounce?

If email sent + delivered but owner says "didn't get it": clarify it could be in spam, or check exact email spelling.

## Step 3: identity verification

Before any reset, verify the person is who they say:

1. Phone call (not email — they say email is broken). Number from `people.phone` or stable contact info on file.
2. Confirm at least 3 facts: stable name, UID-MWST, current month's revenue ballpark, name of one client/horse.
3. For high-stakes recovery (suspected compromise), require ID photo via secure channel + Ferdinand approval.

Document the verification in the audit log.

## Step 4: execute the recovery

Run the recovery script (in `packages/db/scripts/recover-owner.ts`):

```bash
pnpm tsx packages/db/scripts/recover-owner.ts \
  --stable-id <uuid> \
  --user-id <uuid> \
  --new-email <new@email.ch> \
  --reason "Phone change, verified via call 2026-05-04"
```

What it does:
1. Updates `auth.users.email` for the user
2. Sends a fresh magic link to the new email
3. Writes audit_log row with `entity='auth.users', action='recovery'`
4. Slack notification to `#stable-prod`

## Step 5: confirm

Watch for:
- Owner clicks magic link within 15 min (if not, investigate)
- Owner successfully lands on dashboard
- Owner can access expected data (no role/membership corruption)

If something's off, immediately revert via the audit log and re-run.

## Edge cases

### Owner forgot which email they used

- Search `auth.users` for emails matching their domain
- Cross-check with `people.email` for owner-role memberships at their stable

### Owner email domain disappeared (their company was acquired, etc.)

- Standard recovery to new email
- Update `people.email` for any `people` row owned by this user
- Their email signature in invoices might still reference the old domain — flag for owner to update in settings

### Suspected compromise

- Immediately revoke active sessions (Supabase: `auth.users.banned_until = now()` for 5 min, then clear)
- Force re-auth with new email
- Audit recent activity: check `audit_log` for last 24h actions; any unusual?
- If financial data tampered: roll back via DB snapshot (`runbooks/disaster-recovery.md`)

### Owner is a co-owner couple (e.g., spouses)

- Both should have separate user accounts with `owner` membership
- If one loses access, the other can perform the recovery without our help
- We document this in onboarding for new stables

## V1.1 self-serve plan

- 2FA for owner role (TOTP)
- Email verification + cooldown for changing primary email
- Recovery codes generated at signup, downloadable PDF
- "Trusted devices" so a known phone can verify a new email change

Until V1.1 ships, this manual procedure is the only path.

## Logs and audit

Every recovery action is logged. After resolution, a brief writeup added to `docs/incident-log/YYYY-MM-DD-auth-recovery-<stable>.md`. Pattern of recoveries (3+ per quarter) signals UX problem.

## Who can run it

V1: only Sharad and Ferdinand. Both have prod DB access. Both have the CLI script.

V1.1: an admin UI behind a separate auth.

## Escalation

If the owner is hostile / aggressive / threatening legal action:
1. Don't recover anything immediately
2. Loop in Ferdinand
3. Document everything in writing (email)
4. Don't make commitments on call
