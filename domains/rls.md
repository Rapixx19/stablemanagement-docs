# Domain: RLS — Row-Level Security

The single most important defensive pattern in this codebase. If RLS leaks, we ship a privacy disaster. Every table, every join, every Storage bucket gets paired tests.

## The mental model

Every query Supabase runs on behalf of an authenticated user is automatically rewritten to include the active RLS policy. The user's JWT contains their `auth.uid()`. Our policy uses that to look up their `stable_id`(s) via `memberships` and filter every row.

## The helper function

Used in nearly every policy. Returns the set of `stable_id`s the current user belongs to.

```sql
create or replace function auth.current_stable_ids() returns setof uuid
language sql stable as $$
  select stable_id from memberships where user_id = auth.uid();
$$;

create or replace function auth.current_role(p_stable_id uuid) returns text
language sql stable as $$
  -- highest privilege wins (owner > manager > worker > client)
  select role from memberships
  where user_id = auth.uid() and stable_id = p_stable_id
  order by array_position(ARRAY['owner','manager','worker','client']::text[], role)
  limit 1;
$$;
```

## Standard policy template

```sql
alter table horses enable row level security;

create policy horses_select on horses for select
  using (stable_id in (select auth.current_stable_ids()));

create policy horses_owner_write on horses for all
  using (
    stable_id in (select auth.current_stable_ids())
    and auth.current_role(stable_id) in ('owner','manager')
  );
```

## Client-restricted reads (e.g., own horses only)

```sql
create policy horses_client_select on horses for select
  using (
    stable_id in (select auth.current_stable_ids())
    and (
      auth.current_role(stable_id) in ('owner','manager','worker')
      or owner_person_id in (
        select id from people
        where user_id = auth.uid() and stable_id = horses.stable_id
      )
    )
  );
```

## Join-table isolation (the leak vector)

`event_horses` joins `events` and `horses`. The join row itself must enforce `stable_id`. We **denormalize** `stable_id` onto the join table:

```sql
create table event_horses (
  event_id uuid not null references events(id) on delete cascade,
  horse_id uuid not null references horses(id),
  stable_id uuid not null,                -- denormalized
  primary key (event_id, horse_id)
);

create policy event_horses_isolation on event_horses for all
  using (stable_id in (select auth.current_stable_ids()));
```

Server-side validation: when inserting into `event_horses`, the action verifies the `event_id` and `horse_id` are in the same stable as `stable_id`.

## Storage bucket policies

Supabase Storage RLS is **separate** from Postgres RLS. Two policies per bucket: one for SELECT (download), one for INSERT (upload).

```sql
-- Bucket: horse-documents
create policy "documents_isolation_select" on storage.objects for select
  using (
    bucket_id = 'horse-documents'
    and (storage.foldername(name))[1] in (
      select stable_id::text from auth.current_stable_ids()
    )
  );
```

File path convention: `{stable_id}/{horse_id}/{document_id}.{ext}`. The folder structure encodes the tenancy. Never put two stables' files in the same folder.

## Service-role escapes (rare, dangerous)

The service role (used only in seed scripts and rare admin tools) bypasses RLS. Three rules: never use it from a user-facing code path; pair every service-role query with an explicit `WHERE stable_id = $1`; comment every usage with `// service_role: reason`. Vercel Cron route handlers run as authenticated server actions, not service role — they look up `stable_id` per row from the data they process.

## RLS test pattern

Every table with `stable_id` has paired tests in `apps/web/tests/rls/`:

```ts
describe('horses RLS', () => {
  test('cross-tenant SELECT returns empty', async () => {
    const a = await seed.stableWithHorse('A');
    const b = await seed.stableWithHorse('B');
    const r = await a.ctx.from('horses').select().eq('id', b.horse.id);
    expect(r.data).toEqual([]);
  });

  test('cross-tenant UPDATE fails', async () => {
    const a = await seed.stableWithHorse('A');
    const b = await seed.stableWithHorse('B');
    const r = await a.ctx.from('horses').update({ name: 'hack' }).eq('id', b.horse.id);
    expect(r.error).toBeDefined();
    const verify = await b.ctx.from('horses').select().eq('id', b.horse.id).single();
    expect(verify.data.name).not.toBe('hack');
  });

  test('client cannot read horses they do not own', async () => {
    const { client, otherHorse } = await seed.stableWithTwoClients();
    const r = await client.ctx.from('horses').select().eq('id', otherHorse.id);
    expect(r.data).toEqual([]);
  });
});
```

## Common mistakes to avoid

- New table without `stable_id` — every multi-tenant table needs it
- Forgetting RLS on a join table — denormalize and policy
- Service-role data leaking through a non-tenant-checked endpoint
- Trusting `stable_id` from the request body (must come from auth context)
- New role without updating policies that branch on role

## Performance

RLS adds ~5–10% query overhead — acceptable. `current_stable_ids()` is `stable` so Postgres caches per-statement. Indexes on `stable_id` are mandatory on every multi-tenant table.

## When RLS isn't enough

Some operations need **explicit server-action validation** in addition to RLS:

- Cross-table consistency (e.g., this horse and this event are in the same stable)
- Role transitions (worker becoming owner — explicit re-auth)
- Sensitive actions (deleting a paid invoice)

`packages/lib/auth/tenant-scope.ts` has the helpers.
