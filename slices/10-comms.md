# Slice 10 — Communications Hub

**Phase:** Comms + booking · **Estimate:** 4 dev-days · **Owner:** Sharad backend, freelancer frontend

Threaded messaging between owner and clients. Per-client thread, optional per-horse subthreads. Email outbound. Owner can broadcast announcements. Inbound email reply-in deferred to V1.1; WhatsApp deferred to V2.

## Goal

Replace the WhatsApp-and-SMS chaos. Owner has one place to message every client. Clients have one place to message the stable. Conversations are tied to context (the horse, the invoice, the request).

## What ships

### Owner side

1. **Inbox** at `/messages`. List of threads ordered by last message. Unread count badge. Filter by client, by horse, by unread.
2. **Thread view**. Standard chat UI: messages bubble-left (other), bubble-right (me). Timestamps. Read receipts.
3. **Compose modal.** Pick recipient (client or per-horse), type, send. File attachment (PDF/JPG/PNG, ClamAV scanned). Send-on-Enter, Shift-Enter for newline.
4. **Broadcast** at `/messages/broadcast`. Compose once → sends to all (or filtered) clients. Subject + body + optional attachment. Recipients receive in their preferred language — owner writes in DE/FR/IT and tags the message with origin language; we don't auto-translate in V1.
5. **Email outbound** for every message. Recipient gets an email with the message body and a deep-link back to the thread. Email templates trilingual.

### Client side

6. **Messages** at `/messages`. One thread per stable (in V1, client only at one stable; V2 multi-stable clients possible). Same chat UI.
7. **Per-horse subthread.** If client has 2+ horses, optional horse-tag on each message ("about Bella"). Filter the thread by horse.

## Schema diff

```sql
-- 010_comms.sql
create table message_threads (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  person_id uuid not null references people(id),
  last_message_at timestamptz,
  unread_count_owner int not null default 0,
  unread_count_client int not null default 0,
  created_at timestamptz not null default now(),
  unique (stable_id, person_id)
);

create table messages (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  thread_id uuid not null references message_threads(id),
  sender_user_id uuid not null references auth.users(id),
  sender_role text not null check (sender_role in ('owner','worker','client')),
  body text not null,
  horse_id uuid references horses(id),        -- optional per-horse tag
  attachment_path text,
  attachment_filename text,
  read_at timestamptz,
  created_at timestamptz not null default now()
);

create index on message_threads (stable_id, last_message_at desc);
create index on messages (thread_id, created_at);

create table broadcasts (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  sender_user_id uuid not null references auth.users(id),
  subject text not null,
  body text not null,
  origin_language text not null check (origin_language in ('de','fr','it')),
  attachment_path text,
  recipient_count int not null,
  sent_at timestamptz not null default now()
);
```

## Email transactional

- Provider: Resend or Postmark (low-cost, Swiss-friendly). See `domains/email.md`.
- Templates: trilingual; owner writes in stable's default; client receives in their language preference (no machine translation V1).
- Reply-to: a unique inbox address per thread (e.g., `thread-uuid@inbox.lafattoria.app`). Inbound parsing in V1.1; for V1 the reply just bounces with "reply via app" auto-response in DE/FR/IT.

## Realtime

- Supabase Realtime subscription on `messages` table per thread. UI updates without refresh.
- Unread-count update via Postgres trigger on insert.

## RLS

All three tables scoped by `stable_id` per `domains/rls.md`.

- `message_threads`, `messages`, `broadcasts`: `owner` full access.
- `client`: SELECT and INSERT on `messages` for threads where `message_threads.person_id = (people row WHERE user_id = auth.uid())`. SELECT on their own `message_threads` only. **No access to `broadcasts`** — clients receive broadcasts as email; the in-app surface is owner-only.
- `worker`: **no access** in V1 — comms is owner↔client only. SELECT/INSERT/UPDATE/DELETE all denied on all three tables. V1.1 candidate: separate worker↔owner barn-chat channel on a new schema.
- **Sender attribution**: `messages.sender_user_id` enforced equal to `auth.uid()` via `WITH CHECK` on the INSERT policy. No spoofing.
- **Cross-thread guard**: `messages.thread_id` must reference a `message_threads` row in the same `stable_id` as the inserter — server-action validates before INSERT (RLS alone permits any thread in the user's tenant).

## Acceptance criteria

- [ ] Owner sends message to client; client receives email + sees in-app within 2s
- [ ] Reply from client appears in owner's thread realtime
- [ ] Per-horse tag filters the thread correctly when client has 2 horses
- [ ] Broadcast to 19 clients sends 19 emails, creates 1 broadcast record + 19 messages, no duplicates on retry
- [ ] Attachment over 10 MB rejected; under, virus-scanned and persisted
- [ ] Email landing page deep-links back to the thread (auto-login via signed token)
- [ ] Read receipts: thread `unread_count` decrements when client opens thread
- [ ] DE/FR/IT email templates render correctly per recipient
- [ ] **Worker SELECT on `messages` denied** by RLS
- [ ] **Worker INSERT on `messages` denied** by RLS
- [ ] **Sender spoofing**: client INSERT with `sender_user_id != auth.uid()` rejected by WITH CHECK
- [ ] **Cross-thread leak**: client A cannot INSERT a message into client B's thread (same stable, different `person_id`)

## Acceptance integration test

`apps/client/tests/integration/messaging.test.ts`

```ts
test('owner broadcast reaches all active clients with correct language', async () => {
  const { ownerCtx, deClient, frClient, itClient } = await seed.threeClients();
  await broadcast(ownerCtx, { subject: 'Stable closed Sunday', body: '...' });
  expect(emailQueue).toHaveLength(3);
  expect(emailQueue.find(e => e.to === deClient.email).template).toBe('broadcast.de');
  expect(emailQueue.find(e => e.to === frClient.email).template).toBe('broadcast.fr');
  expect(emailQueue.find(e => e.to === itClient.email).template).toBe('broadcast.it');
});
```

## Out of scope

- Inbound email parsing (V1.1)
- WhatsApp Business inbound/outbound (V2)
- Group chats (V2)
- Voice notes (V2)
- Auto-translation between languages (V2)
