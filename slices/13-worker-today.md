# Slice 13 — Today View and Task Completion

**Phase:** Worker · **Estimate:** 3 dev-days · **Owner:** Sharad backend, freelancer frontend

The closed-loop piece. Worker sees their assigned tasks for today. Taps to mark done. Owner sees the trail.

## Goal

When Marco arrives in the barn at 06:45, his Worker Home shows: shift block (07:00–12:00), 6 tasks ordered by time, two checked off (already done), four to go. Tap the circle → task is `done`, owner sees it instantly.

## What ships

### Worker side

1. **Today's tasks block** on Home (worker density per `DESIGN.md`: paper background, ink-900 body, 44px tap rows). Header: shift time range + count "2/6 done." Each task: round check circle, title, italic subtitle, time. Tap circle → toggle done with the 140ms tap-response motion; completed task slides to "done" section using the 260ms list-reorder motion.
2. **Tasks tab** at `/tasks`. Full list of today's tasks + tomorrow's preview. Group by shift if multiple shifts.
3. **Task detail drawer** on tap of task body (not circle). Shows: full description, owner notes, photo upload (V1.1 placeholder for now — slice 13 doesn't ship the photo capture; the drawer shows "comments" to type a note instead).
4. **Skip with reason.** Long-press or "skip" button → modal requires reason text → sets status `skipped`.
5. **Realtime sync.** Owner adds a task at 09:30 → worker sees it appear on Home within 2s.

### Owner side

6. **Today's task progress** on Owner dashboard tile. "Marco: 4/6 done. Sara (worker): 2/3 done."
7. **Task history** at `/tasks/history`. Per-day view of who completed what, with timestamps.

## Wiring to slice 04

This slice does not change schema. It wires the `tasks` schema from slice 04 to the worker UI and adds Realtime subscriptions. The `assigned_to_user_id` field is now used.

## Realtime — hybrid channel + polling fallback

The barn is a flaky-connectivity environment. Realtime alone is not safe. We run a hybrid:

- **Primary**: Supabase Realtime subscription on `tasks` filtered by `(stable_id, user_id, due_date)`. RLS-aware. Connection state surfaced in a small "live" / "syncing" pill in the header.
- **Heartbeat**: every 30s the client sends a Realtime presence ping. Three consecutive missed heartbeats (90s of silence) → drop into polling mode, show "syncing" pill.
- **Polling fallback**: poll `/api/worker/today` every 10s while in syncing mode. Same payload shape as the Realtime delta apply path, so the UI reducer doesn't care which transport delivered the update.
- **Reconciliation on reconnect**: when Realtime channel comes back, client immediately fetches the canonical state from `/api/worker/today` once, replaces local state, then resumes deltas. Prevents drift caused by missed events during the gap.
- **Optimistic UI on tap**: circle fills immediately, server action commits, on error reverts with a toast. Optimistic state is keyed on `(taskId, expectedVersion)` — if the server-confirmed version doesn't match, we re-fetch and re-render rather than blindly applying the local change.
- **First-claim-wins on unassigned tasks** uses Postgres-side conditional update: `UPDATE tasks SET assigned_to_user_id = $worker, status = 'in_progress' WHERE id = $task AND assigned_to_user_id IS NULL RETURNING id`. Empty result = lost the race; UI shows "claimed by Sara" toast.

## Mobile UX details

- Check circle: 22 px touch target (44 px tap area with padding)
- Haptic feedback on tap (where supported via Web Vibration API)
- Done tasks slide to bottom of list with animation
- "All done" state: empty state illustration + "Nice. Wrap up your shift." copy
- Pull-to-refresh on Home

## Acceptance criteria

- [ ] Worker sees their assigned tasks for today, ordered by time
- [ ] Tapping circle marks task done; UI updates optimistically
- [ ] Server failure reverts the UI with toast "couldn't save, retry?"
- [ ] Owner adding a task in real time → worker sees it within 2s
- [ ] Skip without reason rejected; with reason saved
- [ ] Owner dashboard reflects worker progress
- [ ] Unassigned tasks (assigned_to_user_id null) visible to all workers; first to tap claims it (optimistic locking)
- [ ] Tasks completed yesterday don't appear on today's list
- [ ] **Realtime disconnect+reconnect produces consistent state**: simulate channel drop for 60s while owner adds 2 tasks, edits 1, deletes 1 → on reconnect worker UI converges to canonical server state with zero phantom rows and zero missing rows
- [ ] **Heartbeat-miss fallback**: kill the Realtime channel silently; within 90s the UI flips to "syncing" pill and starts polling at 10s; new tasks added by owner appear within 10s in this mode
- [ ] **First-claim-wins race**: 5 workers tap the same unassigned task within 100ms → exactly one `success: true` result; the other four get "claimed by X" with no DB row corruption (validated by SELECT after race)
- [ ] **Optimistic version conflict**: worker A taps a task, owner edits it server-side before commit, server rejects with version mismatch → worker A's UI re-fetches and re-renders without crash, no false "done" state persisted

## Acceptance integration test

`apps/worker/tests/integration/task-complete.test.ts`

```ts
test('completing a task records who and when', async () => {
  const { workerCtx, taskId } = await seed.assignedTask();
  await completeTask(workerCtx, { taskId });
  const t = await getTask(workerCtx, taskId);
  expect(t.status).toBe('done');
  expect(t.completed_by_user_id).toBe(workerCtx.userId);
  expect(t.completed_at).not.toBeNull();
});

test('unassigned task: first claim wins', async () => {
  const { task, workerA, workerB } = await seed.unassignedTask();
  const [a, b] = await Promise.all([
    completeTask(workerA, { taskId: task.id }),
    completeTask(workerB, { taskId: task.id }),
  ]);
  const winners = [a, b].filter(r => r.success);
  expect(winners).toHaveLength(1);
});
```

## Out of scope

- Photo attachment on completion (V1.1)
- Subtasks within a task (V1.1)
- Voice note completion ("Marco said: water tank A was empty, I refilled") (V1.5 with STT)
- Predictive task ordering (V1.5 ML)
