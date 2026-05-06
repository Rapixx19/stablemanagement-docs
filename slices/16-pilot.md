# Slice 16 — Polish, Dashboards, La Fattoria Pilot

**Phase:** Polish + pilot · **Estimate:** 7 dev-days · **Owner:** Sharad + freelancer + Ferdinand

The closing slice. All three dashboards wired against real data. DE/FR/IT translation audit. Onboarding flow for La Fattoria. Restore drill. Data-export endpoint. Pilot exit-criteria checklist run.

## Goal

La Fattoria opens the app, sees real data, and runs an end-to-end month: clients, horses, schedule, tasks, billing, comms, requests, time clock. No part of the system feels like a placeholder. Every dashboard tile loads in under 1s. Every translated string makes sense to a native speaker.

## What ships

### Dashboards (the visible polish)

1. **Owner dashboard** at `/dashboard` (owner density per `DESIGN.md`: 12-col grid, 1280px max, 320px nav rail, 24px gutter). KPI row (revenue forecast, outstanding, occupancy, active clients — `xl 28/36` numerals, tabular figures, NOT individually carded — they're stats, not cards) + stall map tile + recent invoices + today's events + client requests + quick actions. Visual reference: `mockups/owner-dashboard.html`.
2. **Client dashboard** at `/dashboard` (client density per `DESIGN.md`: 12-col, 960px max read width, single-pane). Horses hero + calendar + this month's tab + invoices + messages + quick actions. Tone: warm per `DESIGN.md` voice rules. Visual reference: `mockups/client-home.html`.
3. **Worker dashboard** (Home tab) per `DESIGN.md` worker density. Already mostly wired through slices 12–15; this slice does the polish and the owner-ping panel. Visual reference: `mockups/worker-today.html`.

### Translation audit

4. **DE/FR/IT review** of every string in the app. Native speaker sweep; La Fattoria's network sources Ticino reviewer (Ferdinand's mother / Italian-speaking family contact).
5. **Email template review.** All trilingual emails: invoice, message, broadcast, magic-link, consent. PDF templates DE/FR/IT.

### Onboarding

6. **First-run wizard** for La Fattoria:
   - Stable info (name, UID-MWST, address, default language)
   - Stall layout (rows × cols, generates initial grid)
   - Add first 3 clients
   - Add first 3 horses
   - Build catalog (paste from existing PDF)
   - Configure VAT (effektiv, rates pre-seeded)
   - Connect Bexio (optional)
   - Invite Sharad as worker (test the role)

### Operational hardening

7. **Restore drill.** Take a snapshot. Restore to a fresh project. Verify data integrity. Document procedure in `runbooks/disaster-recovery.md`.
8. **Data export endpoint.** `GET /api/export/all` returns a ZIP with CSV per table + PDFs from Storage. Owner-only. Audit-logged. Required by nFADP.
9. **Performance audit.** All dashboard tiles < 1s p95 on production data scale (28 horses, 19 clients, 100 invoices, 1000 tab lines, 500 events). Slow queries identified and indexed.
10. **Sentry + UptimeRobot in production.** Alerts wired to Slack channel `#stable-prod`.

### Pilot launch

11. **Pilot exit-criteria checklist** documented in README run against La Fattoria for 4 weeks before declaring V1 done.
12. **Feedback button** in app (per pilot banner mockup) → routes to a Linear/GitHub issues board.

## Acceptance criteria (the pilot exit criteria)

- [ ] 4 consecutive weekly invoice cycles run without owner intervention beyond review
- [ ] 0 RLS test failures in CI for 30 days
- [ ] 0 P0 bugs (data loss, wrong invoice amount, auth break) for 30 days
- [ ] La Fattoria reports they have not opened Excel/WhatsApp for stable management in the prior week
- [ ] Mean QR-bill PDF generation < 3s, p99 < 8s
- [ ] All 17 slices have integration tests passing
- [ ] Translation pass on DE/FR/IT done by native speakers (not just LLM)
- [ ] Restore drill executed and documented
- [ ] Data export tested on La Fattoria's own data
- [ ] Three dashboards match the locked mockups in fonts, colors, spacing, micro-interactions
- [ ] Owner dashboard tiles all load < 1s p95

## Acceptance integration test

`apps/owner/tests/integration/dashboard.test.ts`

```ts
test('dashboard for stable with full pilot data loads in < 1s', async () => {
  await seed.stableAtPilotScale(); // 28 horses, 19 clients, 100 invoices, 1k tab lines
  const start = Date.now();
  const data = await loadDashboard(ownerCtx);
  expect(Date.now() - start).toBeLessThan(1000);
  expect(data.revenue_forecast_cents).toBeGreaterThan(0);
  expect(data.stall_occupancy.assigned).toBe(26);
});
```

## What we don't do in slice 16

- No new features beyond polish
- No ML (V1.5)
- No new integrations
- No native mobile

## Definition of "V1 done"

Pilot exit criteria all green for 30 days running. Then we cut the V1 release tag, write the post-mortem, plan V1.1.

## V1.1 backlog (already specced, ready when V1 ships)

- TAMV journal (the differentiator we deferred)
- Inventory module
- Reporting module
- Saldosatz VAT method
- Payment reconciliation outside Bexio (camt.054 import)
- Photo task completion
- Auto-scheduler
- Italian translation polish round 2

V1.5 = ML layer. V2 = Sentavita biometrics integration, Identitas API, native mobile, full TAMV ecosystem.
