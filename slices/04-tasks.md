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
4. **Assignment**: V1 owner can optionally assign a task to a worker user. If unassigned, "anyone available." Slice 13 surfaces assigned-to-me tasks on worker home.

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
  skip_reason text,
  created_at timestamptz not null default now()
);

create index on tasks (stable_id, due_date, status);
create index on tasks (stable_id, assigned_to_user_id, due_date);
```

## Materialisation strategy

A nightly **pg_cron** job (`supabase/cron/task-materialisation.sql`, runs at 03:00 Europe/Zurich) reads all active templates and materialises tomorrow's tasks. DB-internal job — no Railway, no Vercel function. Idempotent via a unique index on `(template_id, due_date)`. Recurring tasks expanded via `rrule.js` invoked from a `plv8` function or via a Supabase Edge Function called from pg_cron with `net.http_post`. Default: keep it pure SQL — write the next 7 days' worth of materialisations as `INSERT ... ON CONFLICT DO NOTHING` driven by a generated series filtered through the RRULE BYDAY/BYHOUR fields persisted alongside the template.

If owner edits a template, future task instances (status='pending', due_date >= today) are regenerated in the same transaction as the template UPDATE via a Postgres trigger. Past instances and any non-pending future instances are untouched.

Crash safety: pg_cron records last successful run in `cron.job_run_details`. The materialisation function is fully idempotent on `(template_id, due_date)`, so a crash mid-run, a manual re-run, or a Supabase failover replaying the job all converge to the same set of tasks.

## RLS

- `task_templates`, `tasks`: scoped by `stable_id`
- Workers can SELECT tasks where `assigned_to_user_id = auth.uid()` or unassigned in their stable
- Workers can UPDATE tasks (status only) where assigned to them or unassigned
- Owner/manager: full access

## Acceptance criteria

- [ ] Owner creates a daily 07:00 feed template; tasks appear automatically each day
- [ ] Editing template title updates future occurrences but not past completed
- [ ] Marking a task done records `completed_by_user_id` and `completed_at`
- [ ] Skipping a task requires a reason (modal blocks save without text)
- [ ] Materialisation pg_cron job is idempotent: running it twice produces 1 row per `(template_id, due_date)`
- [ ] **Crash-restart idempotency**: simulate a crash mid-materialisation (kill the SQL function partway through a 100-template run), restart the job, end-state has exactly the right set of rows with no duplicates and no missing dates
- [ ] **Schedule drift test**: pg_cron schedule pinned to `'0 3 * * *'` in `Europe/Zurich` (verified via `cron.job` SELECT in CI) — if Supabase changes default timezone, test fails loudly
- [ ] Worker (slice 13) sees their assigned tasks for today
- [ ] Mobile view: thumb-tap-friendly check circles (44px min)

## Acceptance integration test

`apps/owner/tests/integration/tasks.test.ts`

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
