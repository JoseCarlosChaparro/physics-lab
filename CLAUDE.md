# Physics Lab

Minecraft-style digital physics laboratory for university students (kids mode in a later phase).
Sellable product for universities/schools and a portfolio piece.

## Source of truth — read before any work
- Design spec: `docs/design/2026-10-09-physics-lab-mvp-design.md`
- Decisions log: `docs/design/decisions-log.md` — decided items (`Dn`) and deferred topics (`Fn`).

## Rules
- **Decisions log first.** Every new decision or deferred topic is appended to the log (next free `Dn`/`Fn`, with date and rationale). Never drop a deferred topic; scope changes happen only through the log.
- **Physics rigor is a core value.** Every quantity shown to users comes from the simulation and is validated against an analytic solution with the tolerances in spec §8. Changing a tolerance requires a log entry.
- **Portability boundary (D8, D13).** `packages/sim-core`, `instruments`, `console`, `challenges` and `formats` must not import DOM, rendering or browser-only APIs. Only `apps/web` may.
- **Determinism (D16).** In core code: no `Math.random`, `Date`, `Math.sin`/`Math.cos`; use the core's seeded PRNG and deterministic trig; stable ordering by entity ID; Rapier via `@dimforge/rapier3d-deterministic`.
- **Every change is a command (D23).** World mutations go through serializable core commands; the console and the mouse UI produce the same commands.
- **Bilingual (D9).** All user-facing text goes through i18n with both Spanish and English; no hard-coded UI strings.
- **Accessibility (D14).** WCAG 2.1 AA; every canvas feature needs a keyboard path and an accessible alternative (data tables, scene-state announcements).
- **Zero data collection (D20).** No third-party requests: self-host fonts, libraries and WASM; no analytics or telemetry.
- **Visuals through the theme layer (D22).** Greybox for now; no hard-coded colors/materials outside theme tokens.

## Workflow
- Spec-driven: one implementation plan per Phase 1 milestone (M1 → M5, spec §12), created with the writing-plans skill.
- Current status: spec in owner review (open items in spec §14); next step is the M1 plan — catapult vertical slice in greybox.
