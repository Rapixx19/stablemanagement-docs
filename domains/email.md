# Domain: Email — Transactional

Magic links, invoices, broadcasts, request notifications. Trilingual templates. EU residency. Reliable delivery.

## Provider

**Resend** is the default choice. Reasons:
- EU servers available (Frankfurt, Dublin)
- Postmark-quality deliverability
- React Email integration (templates as JSX)
- Simple pricing ($20/month for 50k emails — fits pilot + V1 scale)

**Postmark** as the fallback. Resend outage → flip env var, Postmark API contract is similar enough.

## Email types we send

| Type | Trigger | Slice |
|---|---|---|
| Magic link | Login attempt | 00, 12 |
| Invitation | Owner invites person | 12 (worker), elsewhere |
| Invoice | Confirm + send invoice | 08 |
| Message | New chat message | 10 |
| Broadcast | Owner broadcast | 10 |
| Request decision | Confirm/decline request | 11 |
| Approval needed | Worker reports sick | 15 |
| Health alert | Document expiry, etc. | 01, 16 |

## Templates

Located in `packages/i18n/email-templates/`. Built with React Email components, rendered at send time.

Naming: `{type}.{locale}.tsx` × DE / FR / IT.

Shared `<Layout>` wraps every email:
- Stable logo (if uploaded) at top
- Body
- Footer with stable contact info
- "Powered by stablemanagement-" small text
- Unsubscribe link (broadcasts only — V1.1)

## From address

`{stable.from_email}` if configured (DKIM-aligned), else `noreply@stablemanagement.app`. Stable owner can configure their own domain in V1.1; V1 uses our default.

Display name: `"{Stable Name} via stablemanagement-"`.

## Reply-to

Per email type:
- Magic link, invitation: no reply (`noreply@`)
- Invoice: stable owner's email
- Message: `thread-{thread_id}@inbox.lafattoria.app` (V1: bounces with auto-response; V1.1 parses inbound)
- Broadcast: stable owner's email
- Request decision: stable owner's email

## Send queue

`packages/lib/email/queue.ts` writes to an `email_queue` table. A **Vercel Cron** at `/api/cron/email-drain` runs every 60s, authenticated by `CRON_SECRET` bearer (per `DECISIONS.md` D6+D12). Picks up to 30 ready rows per tick (`scheduled_for <= now() AND sent_at IS NULL AND attempts < 5`), sends via Resend, marks `sent_at` or increments `attempts` + `last_error` on failure.

```sql
create table email_queue (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null,
  to_email text not null,
  to_name text,
  template_key text not null,
  language text not null,
  payload_jsonb jsonb not null,
  scheduled_for timestamptz not null default now(),
  attempts int not null default 0,
  last_error text,
  sent_at timestamptz,
  created_at timestamptz not null default now()
);
```

Why a queue:
- Survive provider outages
- Throttle (Resend 100 req/sec; we cap at 30)
- Audit trail
- Idempotency (don't double-send on retry)

Failed after 5 attempts → marked `failed`, alerted to Slack.

## Idempotency

Every email has an idempotency key = `{stable_id}:{template}:{ref_id}`. E.g., `lafattoria:invoice:2026-0142`. Resend dedupes by this on send.

## Bounce handling

Resend webhook → our endpoint → mark `email` field on `people` row as `bounced`. Owner sees a flag on the client list. Manual fix.

## Magic-link email

Highest-stakes template. Tested per locale.

```
Subject: Anmeldung bei {stable.name}    (DE)
         Connexion à {stable.name}      (FR)
         Accesso a {stable.name}        (IT)

Body:
  Hello {name},
  Click the link below to sign in. It expires in 15 minutes.
  [Sign in →]
  If you didn't request this, ignore this email.
```

15-minute expiry. Single-use. Tied to the email + IP combination.

## Invoice email

Sent on slice 08 confirm. Attachments: the invoice PDF. Body: total amount, due date, "you can pay by scanning the QR code in the PDF with your banking app."

The CTA does NOT link to a hosted payment page. Swiss QR-bills are paid in the recipient's own banking app, not on the web.

## Broadcast email

One queue entry per recipient. Body shared, recipient-specific personalization (name, language). Sender chooses one origin language; recipients with different language preferences get a small banner: "This message was sent in DE."

Auto-translation deferred to V2.

## Sandbox

In dev + preview, all emails route to `https://mailtrap.io` for inspection. No real delivery to test addresses.

## Hard rules

- Every email goes through the queue (no direct send)
- Every email is template + payload, never freeform body in code
- Every template has DE / FR / IT versions
- Every email has an idempotency key
- Bounces are tracked and surfaced
- Magic links are 15 min, single-use
- No tracking pixels (privacy + GDPR/nFADP)
- Plain-text fallback included on every HTML email
