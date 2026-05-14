# Domain: V1 Deferred — What We Don't Build (And Why)

The discipline document. Every item below was considered, scoped, and explicitly deferred. When in doubt mid-build, ask: "is this on the list?" If yes, defer. If no, decide explicitly and add it here.

## V1.1 (next 3 months after V1 ships)

### TAMV journal (animal medicine treatment record)

- **Why deferred**: V1's horse-document vault covers passport / vet reports as PDFs. Full TAMV journal (every dose, every withdrawal period, every Equidenpass entry) is a separate feature with regulatory implications.
- **Why V1.1 is right**: it's the differentiator from competitors. We need V1 stable first, then this.
- **Effort estimate**: 4–6 weeks, 2 engineers + vet domain reviewer.

### Inventory module

- **Why deferred** (decision 2026-05-13): V1 pilot scope holds at 16 slices. Inventory was specced at "Polish" phase but conflicted with `README.md` and `TODOS.md`, which both already listed it as deferred. Resolved in favour of the smaller pilot — La Fattoria runs the clipboard for one more cycle, and we ship inventory in the V1.1 wave.
- **What V1.1 ships**: stock-on-hand, low-stock badges, worker quick-consume, one-tap reorder POs, supplier email — full spec preserved in `slices/17-inventory.md` (5 dev-days).
- **V1.5 follow-on**: consumption-rate forecasting (time-to-stockout, dynamic thresholds, auto-reorder triggers) needs ~6 weeks of `inventory_movements` data, so it sits with the V1.5 ML layer. Adds 3 dev-days once the ledger has data.

### Reporting module

- **Why deferred**: occupancy trends, revenue by service, client churn. Useful for owners with data culture; La Fattoria runs on intuition for now.
- **Effort**: 2–3 weeks (relies on V1.1 read-only reporting role + dashboard library).

### Saldosatz VAT method

- **Why deferred**: V1 is effektiv-only. Saldosatz is simpler bookkeeping but adds complexity to the engine. La Fattoria uses effektiv.
- **Effort**: 1–2 weeks.

### Payment reconciliation outside Bexio (camt.054 import)

- **Why deferred**: Bexio reconciliation poll covers Bexio users. Stables on other accounting (Banana, KLARA, CashCtrl) need direct camt.054 SEPA notification import.
- **Effort**: 2 weeks.

### Photo task completion

- **Why deferred**: workers attach a photo to a completed task ("water tank refilled, see photo"). Adds visual proof. Increases storage cost.
- **Effort**: 1 week.

### Worker mobile quick-add for tab_lines

- **Why deferred** (decision 2026-05-13): real-world workflow signal — barn workers generate billable events (Marco gives a lesson, Sara sells extra hay). V1 keeps the billing trust boundary at owner because (a) the fraud surface widens once workers touch financials, (b) La Fattoria's pilot owner is on-site daily and absorbs the friction, (c) we want pilot data on how often this gap bites before designing for it.
- **What V1.1 ships**: worker mobile screen at `/worker/billing/quick-add`. Two-tap flow restricted to **pinned catalog services only** (no free-text description, no `unit_price` override). RLS check: `INSERT` allowed only when `created_by_user_id = auth.uid()` AND `unit_price_cents = services.default_price_cents` AND `service_id IN (pinned subset)`. Owner can revoke any line within 24h via existing remove-line-item flow. Audit log captures the worker actor.
- **Effort**: 1 dev-day on top of slice 06's existing schema (no schema change — `created_by_user_id` already there). Adds 1 worker route + 1 RLS policy + 3 acceptance tests.

### Auto-scheduler / shift template generator

- **Why deferred**: V1 owner manually creates shifts. V1.1 generates from a template ("Mon–Fri 06:30–14:30 for Marco").
- **Effort**: 2 weeks.

### Italian translation polish round 2

- **Why deferred**: V1 launches with adequate IT translations. Round 2 with native-Ticinese reviewer post-launch.
- **Effort**: 1 week (translator service).

### Inbound email parsing

- **Why deferred**: V1 emails out, replies-via-email auto-bounce with "reply via app." V1.1 parses inbound for thread updates.
- **Effort**: 2 weeks (Postmark inbound + threading).

### 2FA for owner role

- **Why deferred**: magic link only in V1. 2FA on stable settings + integrations route in V1.1.
- **Effort**: 1 week.

### Excel/CSV catalog import

- **Why deferred**: V1 has paste + AI PDF. CSV would be useful for some stables.
- **Effort**: 1 week.

### Conflict detection on calendar

- **Why deferred**: V1 just shows overlap visually. V1.1 warns "conflict detected" with suggestion.
- **Effort**: 1–2 weeks.

### Discounts / credit notes

- **Why deferred**: V1 has plain invoices. V1.1 adds % or fixed-amount discounts and credit notes.
- **Effort**: 1.5 weeks.

### Cancellation policies on bookings

- **Why deferred**: V1 has manual cancellation. V1.1: "24h notice required, otherwise charged 50%."
- **Effort**: 1 week.

### Waitlist for full slots

- **Why deferred**: V1 declines requests when full. V1.1 lets clients waitlist.
- **Effort**: 1 week.

### Subtasks within a task

- **Why deferred**: V1 has flat tasks. Some operations want "Morning feed → Aisle A, Aisle B, Aisle C subtasks."
- **Effort**: 1 week.

### Overtime auto-calculation

- **Why deferred**: CSV export is enough for payroll software in V1.
- **Effort**: 2 weeks (with Treuhänder review).

## V1.5 (~6 months after V1 ships)

### ML layer

The platform feature. Once we have ~6 months of operational data (tasks, schedules, billing patterns, inventory movements), we train models. Ideas:
- Predictive task ordering ("this worker usually does these tasks in this order")
- Anomaly detection on horse-related events ("Bella's vet visits in last 90 days are 2× last year — flag")
- Revenue forecasting beyond simple linear projection
- Optimal stall assignment (group horses by compatibility from event data)
- Inventory consumption-rate forecasting + auto-reorder thresholds (see slice 17 V1.5 follow-on)

Specs in `domains/ml-roadmap.md` (V1.5 doc, written when V1 ships).

### Sentavita biometric integration

The moat. ECG girth band data flows in. Horse profile gets a "vitals" tab. Anomaly detection from biometric + operational signals combined. This is what differentiates us from every competitor — no one else has the hardware data.

Effort: ~8 weeks once V1 + Sentavita V1 hardware both stabilized.

### Auto-translation for broadcasts

If owner writes broadcast in DE, FR/IT recipients get auto-translated copy with disclaimer.

Effort: 1 week.

## V2 (~12+ months)

- **Native mobile apps (Capacitor wrap)**: V1 PWA → wrap for App Store + Play Store. Worker app benefits most.
- **Identitas API**: mandatory CH horse ID DB sync.
- **Abacus integration**: for larger stables (50+ horses).
- **FNCH integration**: competition entries, licenses, results.
- **WhatsApp Business API**: replace WhatsApp inbound with proper integration. Requires approval, fees, template-only outbound.
- **Multi-stable consolidated dashboard**: for owners running 2+ stables.
- **Free-form stall map canvas**: upload barn photo + drop markers anywhere.
- **Group chats**: V1 is 1-to-1 owner↔client; V2 adds groups.
- **Voice notes**: in chat + on task completion.
- **SSO / SAML / OIDC**: for enterprise customers.

## Permanent NO

- Marketplace for horse sales — separate Sentavita project, not stable management
- Training course marketplace — separate Sentavita project
- Veterinary services platform — out of scope, vets aren't our customer
- Sportsbook / betting features — entirely separate Sentavita workstream
- Public stable directory ("find stables near you") — wrong product

## When to add to this list

If a feature comes up mid-build that isn't on the V1 slice list:

1. Estimate it
2. Ask: does cutting this from V1 cost us La Fattoria pilot success?
3. If no, defer here with reason
4. If yes, push back on something else

Default action is to defer.
