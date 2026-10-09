# Engineering Guidelines

Read this before writing code. `CLAUDE.md` holds the short, non-negotiable rules; this document explains how we apply them.
Rules that must hold without exception are enforced by tooling (lint, dependency rules, CI, hooks), not by this text alone (D31).

## 1. Clean code
- Small functions with one responsibility; verbs for actions, nouns for data; names from the physics domain (`launchSpeed`, not `v2`).
- No magic numbers or strings: named constants with units in the name or type (`GRAVITY_EARTH_MPS2`, `FIXED_STEP_S`).
- At most 2–3 nesting levels; prefer guard clauses and early returns.
- No dead or commented-out code; no debug output in commits.
- DRY: extract logic repeated more than twice. Do not abstract on the first occurrence.
- Avoid boolean parameters that switch behavior; use two functions or an options object with an enum.
- Prefer explicit over clever; prefer immutability for data passed between packages.
- Units are part of the contract: public APIs take and return SI values, and the unit is in the name, the type or the doc comment.

## 2. Architecture and decoupling
- **Dependency direction** (spec §4): `apps/*` → `challenges`/`instruments`/`console` → `sim-core` → `formats`. Never the reverse; enforced by import-boundary lint.
- **Ports and adapters.** Core packages define interfaces (ports) for everything external: physics engine, PRNG, clock, storage, logger. Adapters implement them (`RapierPhysicsAdapter`, `IndexedDbWorldRepository`). Core code never imports an adapter.
- **Dependency injection** by constructor or factory parameters. Each app has one **composition root** that creates adapters and wires the object graph. No service locators, no module-level mutable state.
- **SOLID**
  - *Single responsibility:* one reason to change per class or module.
  - *Open/closed:* add a new part type, solver or instrument by adding a module, not by editing a switch statement.
  - *Liskov:* every adapter honors its port's contract, including units and error behavior; contract tests run against every implementation.
  - *Interface segregation:* small ports (`StepClock`, `RandomSource`), not one `Engine` god-interface.
  - *Dependency inversion:* see ports and adapters above.
- **Law of Demeter:** no `a.getB().getC().do()` chains across module boundaries; add an intent-revealing method instead.

## 3. Design patterns we use (and why)
Apply a pattern only where it removes complexity, coupling or duplication. When you apply one, add a one-line comment naming it and why.

| Pattern | Where | Why |
|---------|-------|-----|
| Command | All world mutations (`placeBlock`, `connect`, `trigger`…) | Undo/redo, exact replay, verification, future multiplayer (D23). |
| Adapter | Rapier, renderer, storage, PDF export | Keeps engines swappable and the core portable (D8). |
| Strategy | Analytic solvers, instrument noise models, quality tiers, tolerance rules | New variants without touching callers. |
| Factory / Registry | Part types, sensors, console commands, challenge loaders | Open/closed extension by registration. |
| Observer (typed event bus) | Sensor events, scene-state announcements for screen readers, UI updates | Decouples simulation from presentation and a11y. |
| Repository | Saving/loading worlds, challenges and results | Storage-agnostic persistence (files today, backend later, F3). |
| Template Method | Challenge lifecycle (predict → run → observe → compare → explain) | Fixed flow with per-challenge hooks. |
| Memento / Snapshot | Build-mode snapshot restored by `reset` | Cheap, exact reset of Run mode. |

Avoid: singletons, inheritance hierarchies deeper than one level (prefer composition), "manager" classes without a single purpose.

## 4. Testing
- **TDD:** red → green → refactor. The failing test comes first and is committed with the change.
- **Unit tests** for every public function and class; test behavior through public APIs, not internals.
- **Physics validation tests** compare simulation against analytic solutions with spec §8 tolerances. They run in CI on every change.
- **Property-based tests** (parameter sweeps) for solvers and parameter generators.
- **Determinism tests:** the same command log yields the same snapshot hash in Node and in Chromium, Firefox and WebKit.
- **Contract tests** run the same suite against every adapter of a port.
- **Fakes over mocks:** use in-memory fakes of ports (`FakeClock`, `SeededRandom`); mock only at true boundaries.
- **E2E (Playwright):** write or update the specs a change needs and keep them type-correct; run them at the end of each milestone or when the owner asks. Reports list e2e specs written but not run.
- Per-task verification: unit tests + lint + typecheck (+ build when config or routes change), with output shown.

## 5. Internationalization
- Every user-facing string is an i18n key with `es` and `en` values; CI fails on missing keys.
- Use ICU message formatting for plurals and interpolation; never build sentences by concatenation.
- Numbers, units and dates use locale-aware formatting (decimal comma vs point).
- Physics notation (symbols, SI units) is not translated; prose around it is.
- Challenge content stores both languages side by side in the same file.

## 6. Accessibility — WCAG 2.2 AA
- Full keyboard operation with visible focus; logical focus order; no keyboard traps.
- **2.4.11 Focus Not Obscured (Minimum):** side panels, toasts and the console never fully cover the focused element.
- **2.5.7 Dragging Movements:** every drag (placing, rotating, joining parts, orbiting the camera) has a single-pointer, non-drag alternative (click-to-place, step buttons, keyboard).
- **2.5.8 Target Size (Minimum):** interactive targets ≥ 24×24 CSS px or properly spaced.
- **3.3.7 Redundant Entry:** the student ID is entered once per session and reused.
- **3.2.6 Consistent Help:** help is in the same place on every screen.
- Canvas content has accessible equivalents: scene-state live region, data table for every graph, textual measurement log.
- Never use color alone: vectors differ by color and shape; colorblind-safe palette; AA contrast; reduced-motion mode; text scalable to 200 %.
- Automated checks (axe-core) in CI plus a manual keyboard and screen-reader checklist per milestone.

## 7. Commits and reviews
- Conventional Commits in English, one logical change per commit, imperative lowercase subject ≤ 72 characters.
- Every non-trivial diff gets a fresh-context review (`code-reviewer`, and `physics-reviewer` or `a11y-reviewer` when relevant) against the plan before it counts as done. Reviewers report correctness and requirement gaps, not style preferences.
