# Physics Lab

Minecraft-style digital physics laboratory for university students (kids mode in a later phase).
Sellable product for universities/schools and a portfolio piece.

## Source of truth — read before any work
- Design spec: `docs/design/2026-10-09-physics-lab-mvp-design.md`
- Decisions log: `docs/design/decisions-log.md` — decided items (`Dn`) and deferred topics (`Fn`).
- Engineering guidelines (read before writing code): `docs/engineering/guidelines.md`

## Project rules
- **Decisions log first.** Every new decision or deferred topic is appended to the log (next free `Dn`/`Fn`, date, rationale). Never drop a deferred topic; scope changes only through the log.
- **Physics rigor.** Every quantity shown to users comes from the simulation and is validated against an analytic solution within the tolerances of spec §8. Changing a tolerance requires a log entry.
- **Portability boundary.** `packages/sim-core`, `instruments`, `console`, `challenges`, `formats` must not import DOM, rendering or browser-only APIs. Only `apps/web` may.
- **Determinism.** In core packages: no `Math.random`, `Date`, `Math.sin`/`Math.cos`; use the core's seeded PRNG and deterministic trig; iterate in stable entity-ID order; Rapier via `@dimforge/rapier3d-deterministic`.
- **Every world change is a Command.** Mouse UI and console produce the same serializable core commands.
- **Dependency injection.** Depend on interfaces (ports); concrete adapters (Rapier, renderer, storage, clock, PRNG) are wired only in the composition root of each app. No module-level singletons or hidden globals.
- **TDD.** Write the failing unit test first; physics code also needs an analytic-validation test. Do not claim a task done without showing test/lint/typecheck output.
- **i18n.** No hard-coded user-facing strings; every key exists in `es` and `en`.
- **Accessibility: WCAG 2.2 AA.** Every canvas interaction needs a keyboard path and an accessible alternative; drag interactions need a single-click alternative (2.5.7); pointer targets ≥ 24×24 CSS px (2.5.8); panels never hide the focused element (2.4.11).
- **Zero data collection.** No third-party requests; self-host fonts, libraries and WASM; no analytics.
- **Visuals through theme tokens.** Greybox for now; no hard-coded colors/materials outside the theme layer.

## Model routing (Claude Code)
- **Main session (Opus):** planning, architecture, decisions-log entries, physics/determinism reasoning, final review. Delegates implementation.
- **Subagents** (defined in `.claude/agents/`, always with an explicit model):
  - `implementer` (Haiku) — one well-specified plan task: code + tests.
  - `senior-implementer` (Sonnet) — multi-file or ambiguous tasks, and any task Haiku returned blocked or failed review twice.
  - `physics-reviewer` (Opus) — reviews physics, solvers, tolerances and determinism.
  - `code-reviewer` (Sonnet) — fresh-context review of each diff against the plan; reports only correctness/requirement gaps.
  - `a11y-reviewer` (Sonnet) — WCAG 2.2 AA and i18n review of UI changes.
- Escalate instead of looping: two failed attempts at one model tier → next tier up.
- Subagents don't see this conversation: every delegation names the plan task, the files, and the check that proves it is done.

## Workflow
- Spec-driven: one implementation plan per Phase 1 milestone (M1 → M5, spec §12), created with the writing-plans skill.
- Current status: spec in owner review (open items in spec §14); next step is the M1 plan — catapult vertical slice in greybox.
