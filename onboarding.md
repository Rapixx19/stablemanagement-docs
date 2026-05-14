# Onboarding — First Week

For a new engineer joining the project mid-stream. Reading order, setup steps, who to ping, anti-patterns. Locked in `reviews/2026-05-07-ceo-review.md` as LT-4.

## Welcome

We're building **Stableplatform** — a Swiss-localized stable management SaaS replacing the WhatsApp + Excel + paper-whiteboard workflow at boarding stables (Pensionsställe) of 11–40 horses. V1 pilot partner is La Fattoria. The product runs as one Next.js app with three role-based route trees (owner / client / worker) on Supabase + Vercel.

This is a real product with real money and real Swiss legal exposure. Move thoughtfully on schema and billing; move fast on UI polish.

## Reading order — day 1

In this exact order:

1. **`README.md`** — what we're building, build sequence, pilot exit criteria
2. **`.cursorrules.md`** — the global rulebook (also for human readers)
3. **`ARCHITECTURE.md`** — file tree, three layers, deployment story
4. **`DECISIONS.md`** — the 12 locked architecture decisions; if a slice contradicts this, the slice is wrong
5. **`DESIGN.md`** — tokens, primitives, voice rules; design is locked
6. **`SCHEMA.md`** — DB schema overview
7. **`TODOS.md`** — pre-slice-00 prerequisites, per-slice follow-ups, sweep policy
8. **`INTEGRATION_TESTS.md`** — what good test coverage looks like

If you're confused, read in this order again before asking.

## Reading order — your first slice

1. **`slices/NN-name.md`** — the slice you're implementing; has the full spec, schema, acceptance criteria
2. The `domains/*.md` files referenced from that slice (RLS, VAT, idempotency, optimistic-locking, etc.) — domain docs are cross-cutting reference; the slice is the contract
3. Recent reviews — `reviews/2026-05-07-{ceo,eng,design}-review.md` — flag any sweep item not yet folded into the slice spec
4. The mockup if one exists — `mockups/*.html` is the visual reference; if a mockup drifts from `DESIGN.md`, `DESIGN.md` wins

## Setup — day 1

```
# Clone
git clone https://github.com/{org}/stablemanagement.git
cd stablemanagement

# Toolchain: Node 24, pnpm, Supabase CLI
nvm install 24
corepack enable
npm install -g supabase

# Install
pnpm install

# Local Supabase (Docker required)
supabase start
# Outputs local DB URL, anon key, service-role key

# Env vars
cp .env.example .env.local
# Fill from `supabase status` output

# Seed
pnpm db:seed
# Creates "La Fattoria Dev" stable + 3 clients + 3 horses + Sharad as worker

# Run
pnpm dev
# Visit http://localhost:3000; magic-link emails land at http://localhost:54324 (inbucket)
```

## Branch and PR conventions

- Branch from `main`: `slice-NN/short-feature-description`
- One slice per PR (or sub-slice if the slice is large); never bundle unrelated changes
- PR title: `Slice NN: <feature>` (no `feat:` prefix — slice number is the prefix; no emoji)
- PR body: Summary (1-3 bullets), Test plan (acceptance criteria checklist), screenshots if UI
- Commit messages: imperative present, no emoji, no semantic prefix; subject < 70 chars; body explains "why" not "what"

## Code review expectations

- One reviewer required (Sharad or Ferdinand)
- All CI checks green: lint, typecheck, unit, integration, RLS, migration grep, schema-dump-diff, e2e
- New migration → paired RLS test mandatory (see `domains/rls.md`)
- New external dep → review with Sharad before merge; we pin exact versions for `swissqrbill`, `@react-pdf/renderer`, `rrule.js`
- Bug fixes don't include refactors; refactors don't include bug fixes

## Who to ping

- **Ferdinand** — product decisions, La Fattoria liaison, design, algo review
- **Sharad** — backend, Supabase, integrations (Bexio, Resend), CI infrastructure, on-call
- **Freelancer** — frontend, design system implementation

## When you're stuck

1. Re-read the relevant slice + domain docs
2. Search closed PRs for similar patterns (we copy ourselves often, by design)
3. Check `reviews/2026-05-07-*-review.md` for sweep items that might cover your edge case
4. Ask in `#stable-eng` Slack — don't suffer silently for >30 min
5. If genuinely blocked, 15-min sync with the owner of that slice

## Anti-patterns already locked out

- Emojis in commits, PRs, code, or docs (allowed: empty-state copy, owner-visible toasts when designed)
- Creating a new doc when an existing domain doc fits (extend, don't sprout)
- Money on `numeric(12,2)` columns (use `bigint *_cents` per `domains/money.md`)
- New table without `stable_id` + RLS policy + paired RLS test (CI catches this, but don't make CI do the work)
- Cross-cutting state inside a slice (state goes in a `domains/*.md`)
- Touching `DECISIONS.md` without a PR explicitly titled "Override decision DN"
- A slice doc > 150 lines (split or move reference material to `domains/`)
- Bare `cacheTag('something')` without `stable:${stableId}:` prefix (see `domains/cache-strategy.md`)
- Adding `version int` but not using it in the WHERE clause (see `domains/optimistic-locking.md`)

## Local development tips

- Magic-link emails land in inbucket: http://localhost:54324
- Reset local DB: `supabase db reset` (re-runs migrations + seed)
- Type-check on save: workspace TS server config (committed)
- Tailwind autocomplete: install the official Tailwind CSS IntelliSense extension
- Storybook is V1.1 — don't add it now

## First-week milestones

- **Day 1**: setup works, you can log in as a seeded user, you've read the day-1 list
- **Day 2-3**: pick a small follow-up from `TODOS.md` per-slice spec sections; ship a PR
- **Day 4-5**: take on the smaller half of an active slice; review with its owner

By end of week 1 you should have: one merged PR, familiarity with the slice you'll own next sprint, and one question nobody has answered yet.

## Pilot context (read once, refer back)

La Fattoria is the only pilot stable in V1. The owner is Ferdinand's contact (likely Italian-speaking primarily). DE is canonical for code and design; FR/IT translate from DE, not from English. The exit criteria are in `README.md`. Anything that risks pilot exit is treated as P0.

## When this doc updates

- A new top-level doc joins the reading order → add it
- The setup story changes → update the commands
- An anti-pattern bites a new hire → add it to the list
- A new common stumbling block surfaces → add a "When you're stuck" subsection
