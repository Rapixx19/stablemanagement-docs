# Stableplatform — Claude Code instructions

Read **`docs/.cursorrules.md`** on every task. It is the canonical operating manual for this project. The same file is read by Cursor; both AI tools follow identical rules.

## Quick orientation for a fresh session

If you're new to this codebase or returning after a break, read in this exact order before doing anything:

1. **`docs/.cursorrules.md`** — operating rules (workflow, hard rules, file-header requirement, debugging map). This is the file you must respect on every task.
2. **`docs/README.md`** — what we're building (Swiss multi-tenant stable management SaaS), pilot exit criteria, build sequence.
3. **`docs/ARCHITECTURE.md`** — file tree, three layers, deployment.
4. **`docs/DECISIONS.md`** — 17 locked architectural decisions. If your code contradicts one, stop and ask.
5. **`docs/DESIGN.md`** — tokens, primitives, voice rules, AI-slop forbidden list, viewport behaviors (DES-LOCK-1, -2, -3).
6. **`docs/SCHEMA.md`** — DB schema overview.
7. **`docs/TODOS.md`** — pre-slice-00 prerequisites, per-slice follow-ups, three review sections (CEO, Eng, Design from 2026-05-14).

## Project state (2026-05-14)

- **Spec phase complete.** No production code yet. Slice 00 is the next gate.
- **18 slices specced** (00 → 17 with 17 deferred to V1.1).
- **22 mockups** in `docs/mockups/` covering every novel-UX surface.
- **22 domain docs** in `docs/domains/` for cross-cutting concerns.
- **17 architectural decisions** locked in `docs/DECISIONS.md` (D1-D17).
- **Three pre-coding reviews** complete: CEO, Eng, Design — see `docs/reviews/2026-05-07-*.md` for the May 7 baseline and the 2026-05-14 follow-up TODO sections in `TODOS.md`.

## Conversation conventions

- Match the rules in `docs/.cursorrules.md` literally. Don't paraphrase.
- For non-trivial tasks: state your plan + file-tree change + test strategy BEFORE writing code. The user will confirm or redirect before you implement.
- Verify in the browser (run `pnpm dev`, click through the feature) before reporting done. If you can't, say so.
- Update `docs/TODOS.md` if you defer work. Don't leave floating `TODO` comments in code.
- Reference slice numbers and domain docs in commit messages: `Slice NN: <feature>` per the project's PR convention.

## When the user says "build feature X"

1. Find the slice or domain doc that covers it (`docs/slices/NN-*.md` or `docs/domains/*.md`).
2. Read the acceptance criteria.
3. State your implementation plan in conversation.
4. Wait for user confirmation.
5. Implement vertically: schema → RLS → server action → UI → tests → manual verification.
6. Tick off acceptance criteria checkboxes in the slice spec in the same PR.

## When the user reports a bug

Use the debugging map in `docs/.cursorrules.md` § "Debugging — where to look when something breaks." Order: Sentry → Better Stack → Supabase logs → owner-visible failure surfaces → `runbooks/oncall.md` → `domains/*.md` failure-modes tables.

Write a failing test first, then fix.

## Who's who

- **Ferdinand** — product, founder, La Fattoria liaison, design decisions.
- **Sharad** — backend, Supabase, integrations (Bexio, Resend, Reducto, OpenAI, Anthropic), CI infrastructure, on-call.
- **Freelancer** — frontend, design system implementation, primarily slices 12 / 14 / 15 / 02 / 10 / 11.

When asked product questions, defer to Ferdinand. Never invent product behaviour.

---

The full operational manual is in `docs/.cursorrules.md`. Read it.
