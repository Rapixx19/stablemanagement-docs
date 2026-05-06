# Domain: Auth and Roles

Three apps, four roles, one Supabase Auth tenant. The trickier-than-it-looks part is users with multiple memberships (working students, owner-spouses) and locale-aware invitation flows.

## Roles

| Role | Apps accessible | Powers |
|---|---|---|
| `owner` | Owner | Everything in the stable: people, horses, billing, settings, integrations |
| `manager` | Owner | Same as owner except billing settings, Bexio connect, user management |
| `worker` | Worker | Their schedule, tasks assigned to them, time clock, ask-the-boss messaging |
| `client` | Client | Their horses, calendar (own + public), their invoices, messaging, requests |

Membership is `(user_id, stable_id, role)`. A user can have many — one user can be a worker at stable A and a client at stable B. Even simultaneously at the same stable (working student boards her own horse).

## Login flow

1. User enters email at the login page (which app determines the redirect after success)
2. Magic-link email sent in user's preferred language (or the app's default)
3. Click link → JWT issued
4. Server checks memberships:
   - 0 memberships at this stable: show "no access" screen
   - 1 membership: redirect into the app for that role
   - 2+ memberships: show role picker
5. Selected role stored in JWT custom claim for the session
6. RLS policies use both `auth.uid()` and the role claim

## Magic link only

Password support not in V1. Reasons:
- Stable owners and clients are not technical
- Password reset flows require email anyway, so cut the loop
- Lower attack surface

OAuth (Google/Apple) backlog for V1.1 if customers ask.

## Invitations

Owner invites a person:
1. Owner enters email + name + role + language preference
2. We create a `people` row (no `user_id` yet) and an `invitations` row
3. Send invite email in their language: "{stable.name} invites you to join the {role} app"
4. Recipient clicks link → magic-link login
5. On first login, we link `auth.users.id` ↔ `people.user_id`
6. Membership row created
7. They land on the right app for their role

## Invitation tokens

- Random 32-byte base64url
- 7-day expiry
- Single-use
- Revocable from `/people/[id]/invitations`

```sql
create table invitations (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  email text not null,
  person_id uuid references people(id),
  role text not null,
  language text not null,
  token text not null unique,
  expires_at timestamptz not null,
  accepted_at timestamptz,
  revoked_at timestamptz,
  created_by_user_id uuid references auth.users(id),
  created_at timestamptz not null default now()
);
```

## Account recovery

When an owner forgets their email or loses access:
1. They contact support (Ferdinand, in pilot phase)
2. Support verifies identity (phone call, ID check)
3. Support runs the `recoverOwner` script: generates a new magic link bound to a new email
4. Audit log records the recovery

V1 has no self-serve recovery. Documented in `runbooks/auth-recovery.md`.

## Session lifecycle

- JWT expiry: 1 hour
- Refresh token: 30 days
- Idle timeout: client-side after 4 hours, prompt re-login
- Per-stable session isolation: switching stables forces re-auth (rare scenario)

## Worker-specific: consent

First worker login must accept the time-clock consent (slice 12). This is stored in `worker_consent` with the version of the consent text they agreed to. Version bumps require re-consent.

## RBAC enforcement points

Three layers:

1. **UI**: components conditionally render based on role. Cosmetic only.
2. **Server actions**: `requireRole(ctx, stableId, ['owner', 'manager'])` at the top of every privileged action. Throws.
3. **RLS**: the database is the final line. Policies branch on `auth.current_role(stable_id)`.

Never rely on UI alone. Every server action checks. Every table has RLS.

## Multi-tenancy in JWT

After role pick, the JWT carries:
```
{
  sub: user_id,
  stable_id: <selected>,
  role: <selected>,
  exp: ...
}
```

Switching stables = re-auth. We don't multiplex stables in a single JWT — too much complexity for V1.

## Email templates

Three templates × three languages = nine files in `packages/i18n/email-templates/`:

- `magic-link.de.html`, `.fr.html`, `.it.html`
- `invitation.de.html`, ...
- `welcome.de.html`, ...

All use a shared header/footer partial with the stable logo (if uploaded) and the "Powered by stablemanagement-" footer.

## Failure modes to test

- Magic link clicked from a different device/browser → still works (token, not session, validates)
- Magic link clicked twice → second click no-ops (token already consumed)
- Magic link expired (15 min default for security) → clear error, "request a new link"
- User with 0 memberships logs in → "no access" screen with support contact

## Out of V1

- SSO (SAML, OIDC) — V2 for enterprise customers
- 2FA — V1.1 for owner role
- Password support — backlog
- OAuth via Google/Apple — backlog
- Biometric login on mobile — V2 with native wrapper
