# Slice 04 — Tasks

**Phase:** Operational core · **Estimate:** 3 dev-days · **Owner:** Sharad backend, freelancer frontend

The daily operational checklist. Feed AM, feed PM, turnout group A, muck stalls, water check. Owner creates task templates and schedules them. Workers complete them (slice 13 wires the worker side).

## Goal

Owner defines a recurring set of daily tasks. Each day, the system materialises pending tasks from templates. Owner sees what's done and what isn't. The data model is built so slice 13 (worker app) can consume it without schema changes.

## What ships

### Owner side

1. **Task templates** at `/tasks/templates`. Create reusable tasks: title, description, recurrence (daily / weekly / specific days of week / once), default time of day, estimated duration. Examples: "Morning feed Aisle A — 07:00, daily, 30 min."
2. **Today's tasks** view at `/tasks`. List of all tasks materialised for today, with status (pending / in-progress / done / skipped). Click to mark done. Note required for skip.
3. **Tasks dashboard tile** (used in slice 16). Compact status: "4 of 6 done."
4. **Assignment**: owner can assign a task to a worker user, to **themselves** (or another owner), or leave unassigned ("anyone available"). When assigned to an owner, the task appears on the owner's dashboard tile "Meine Aufgaben heute" (slice 16) and at `/owner/tasks/mine`. Slice 13 surfaces worker-assigned tasks on the worker home. RLS already permits — `assigned_to_user_id` references `auth.users` with no role constraint.
5. **Owner's own task list** at `/owner/tasks/mine`. Same `DataTable` primitive as the owner main task view, in owner density per `DESIGN.md`. Filters: today / this week / overdue. Inline complete + edit. Excluded from the worker mobile bundle.

## Schema diff

```sql
-- 005_tasks.sql
create table task_templates (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  title text not null,
  description text,
  recurrence_rule text,                       -- RRULE; null = manual one-off
  default_time_of_day time,
  estimated_minutes int,
  active boolean not null default true,
  created_at timestamptz not null default now()
);

create table tasks (
  id uuid primary key default gen_random_uuid(),
  stable_id uuid not null references stables(id),
  template_id uuid references task_templates(id),
  title text not null,
  description text,
  due_date date not null,
  due_time time,
  assigned_to_user_id uuid references auth.users(id),
  status text not null default 'pending' check (status in ('pending','in_progress','done','skipped')),
  completed_by_user_id uuid references auth.users(id),
  completed_at timestamptz,
  skip_reason text check (length(skip_reason) <= 500),
  version int not null default 0,             -- optimistic locking, used in slice 13
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create unique index tasks_template_due_unique
  on tasks (template_id, due_date, coalesce(due_time, time '00:00')) where template_id is not null;
create index on tasks (stable_id, due_date, status);
create index on tasks (stable_id, assigned_to_user_id, due_date);
```

## Materialisation strategy

A nightly **Vercel Cron** job (`/api/cron/task-materialisation`, runs `0 3 * * *` Europe/Zurich, authenticated by `CRON_SECRET` bearer per `DECISIONS.md` D7+D12) reads all active templates and materialises the next 7 days' tasks via `rrule.js`. Pure SQL cannot expand RRULEs — the Node handler does. Idempotent via the unique index `(template_id, due_date, coalesce(due_time, '00:00'))`. Re-runs produce zero new rows.

If owner edits a template, future task instances (status='pending', due_date >= today) are regenerated in the same transaction as the template UPDATE via a Postgres trigger that deletes-and-reinserts pending future rows for that template. Past instances and any non-pending future instances are untouched.

Crash safety: the cron handler is fully idempotent on `(template_id, due_date, due_time)`. A crash mid-run, a manual re-run, or a Vercel retry all converge to the same set of tasks. Last successful run timestamp is stored in `cron_run_log` (small audit table, see slice 00).

## RLS

Both tables scoped by `stable_id` per `domains/rls.md`.

- `task_templates`: `owner` full access. `worker`: SELECT only (worker home renders template title + time-of-day). `client`: **no access**.
- `tasks`:
  - `owner`: full access.
  - `worker`: SELECT where `assigned_to_user_id = auth.uid()` OR `assigned_to_user_id IS NULL`. UPDATE allowed only when (a) row is assigned to the worker or unassigned AND (b) the only changed columns are `status`, `completed_by_user_id`, `completed_at`, `skip_reason`, `version`. **INSERT denied, DELETE denied.**
  - `client`: **no access**.

**Column-level UPDATE constraint** — Postgres RLS doesn't enforce columns. Two-layer guard: (1) `BEFORE UPDATE` trigger on `tasks` raises when a `worker`-role session changes any column outside the whitelist; (2) server-action zod schema validates the input shape to the same whitelist. Defense in depth.

## Acceptance criteria

- [ ] Owner creates a daily 07:00 feed template; tasks appear automatically each day
- [ ] Editing template title updates future occurrences but not past completed
- [ ] Marking a task done records `completed_by_user_id` and `completed_at`
- [ ] Skipping a task requires a reason (modal blocks save without text)
- [ ] Materialisation Vercel Cron handler is idempotent: running it twice produces 1 row per `(template_id, due_date, due_time)`
- [ ] **Crash-restart idempotency**: simulate a crash mid-materialisation (kill the handler partway through a 100-template run), restart, end-state has exactly the right set of rows with no duplicates and no missing dates
- [ ] **Schedule drift test**: Vercel Cron schedule pinned to `0 3 * * *` Europe/Zurich in `vercel.ts`; CI verifies the cron config still matches via a snapshot test
- [ ] Worker (slice 13) sees their assigned tasks for today
- [ ] Mobile view: thumb-tap-friendly check circles (44px min)
- [ ] **Worker INSERT on `tasks` denied** by RLS — no row written, error returned
- [ ] **Worker UPDATE on `title`** denied by trigger even when the row is in the worker's scope (whitelist enforcement)
- [ ] **Worker INSERT on `task_templates` denied** by RLS
- [ ] **Client SELECT on `tasks` denied** by RLS (operational data, no client surface)

## Acceptance integration test

`apps/web/tests/integration/owner/tasks.test.ts`

```ts
test('daily template materialises one task per day for next 7 days', async () => {
  const { ownerCtx, stable } = await seed.stable();
  await createTemplate(ownerCtx, { title: 'Morning feed', rrule: 'FREQ=DAILY' });
  await runMaterialisationCron(); // direct invocation
  const tasks = await getTasks(ownerCtx, { from: today(), to: addDays(today(), 7) });
  expect(tasks).toHaveLength(7);
  await runMaterialisationCron(); // again
  const tasks2 = await getTasks(ownerCtx, { from: today(), to: addDays(today(), 7) });
  expect(tasks2).toHaveLength(7); // idempotent
});
```

## Out of scope

- Photo attachment on completion (V1.1)
- Subtasks / checklists within a task (V1.1)
- Predictive task suggestions (V1.5 ML)
- GPS-verified task completion (V2)
