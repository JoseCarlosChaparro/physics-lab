# Physics Lab — Product Vision & Phase 1 (MVP) Design

- **Status:** Draft for owner review
- **Date:** 2026-10-09
- **Decision record:** [`decisions-log.md`](decisions-log.md) — every `Dn` / `Fn` reference below points there.

---

## 1. Intent

### 1.1 What the owner asked for
A Roblox/Minecraft-style game that works as a **digital physics laboratory**: a creative/sandbox mode where players place objects such as catapults, so they play and learn at the same time. Primary audience: university students. A simplified version for young children is desirable later.

### 1.2 Goals
1. A product that can be **sold to universities and schools**, and a **portfolio piece** (D1).
2. **Physics rigor:** every number the game shows comes from a simulation that is validated against analytic solutions with published tolerances (D4, D17).
3. **Real lab skills:** prediction, measurement with uncertainty, and comparison between theory and experiment (D15, D21).
4. **Zero friction for pilots:** open a link in a desktop browser; no installs, no accounts, no data collection (D6, D7, D19, D20).

### 1.3 Success criteria for Phase 1
- A teacher can assign one of the six challenges with randomized parameters, students complete it in a browser without an account, and the teacher can **verify each submitted result by deterministic replay**.
- Every challenge solver agrees with the simulation within the tolerance in §8 across its full publishable parameter range (CI-enforced).
- WCAG 2.2 AA conformance for the UI, with accessible alternatives for the 3D canvas (D14, D29).
- ≥ 30 fps on a Celeron-class school Chromebook at the low quality tier; offline after first load (D18, D19).
- Full Spanish and English content (D9).

### 1.4 Non-goals for Phase 1
Free-play progression (F1), kids mode (F2), accounts/backend/LMS (F3), installable build (F4), touch/tablets (F5), multiplayer (F6), large/infinite worlds (F7), full challenge editor (F8, committed as the first post-MVP phase), command scripting files (F12), final art direction (F11; Phase 1 ships greybox, D22).

---

## 2. Users and scenarios

| Actor | Scenario (Phase 1) |
|-------|--------------------|
| **Instructor (class demo)** | Projects the sandbox in class, builds or loads an experiment, runs it with instruments visible, discusses results live. |
| **Instructor (assigned lab)** | Picks a built-in challenge, sets parameter ranges, tolerance and instrument mode, generates a link or `.json` file, posts it in the course LMS. Later opens students' result files to review and verify them. |
| **Student** | Opens the link, enters a name or student ID, predicts, builds/adjusts within the allowed scope, runs, measures, compares, explains, exports a report and uploads it to the LMS. |
| **Explorer** | Uses the sandbox freely: builds, experiments, uses the command console. Runs are not verifiable. |

---

## 3. Product roadmap

**Phase 1 (this spec):** sandbox core + challenge layer (D2) with rigid-body physics (D3), hybrid voxel/parts building (D5), six challenges + tutorial (D10, D21), teacher customization "C inside, B outside" (D11), command console (D24), greybox visuals (D22).

**Later phases** (each gets its own spec and plan when its turn comes, D12):
1. Full challenge editor (F8) + command scripting (F12)
2. Pilot efficacy study with the Force Concept Inventory (F9) + business model and pricing (F10)
3. Teacher backend / LMS integration via LTI (F3)
4. Kids mode (F2) + touch controls (F5)
5. Classroom multiplayer (F6)
6. Installable / native performance build when needed (F4)

Art direction (F11) runs in parallel from milestone M2.

---

## 4. Architecture

TypeScript monorepo (D13). Each package has one responsibility; dependencies point only downward, and nothing below `apps/` imports DOM or rendering code.

```
apps/web  (UI, rendering, input, a11y, i18n, theme, PWA, reports)
  ├─► packages/challenges   (challenge data, analytic solvers, parameter generators, teacher guides)
  ├─► packages/instruments  (measurement model: resolution, seeded noise, uncertainty)
  ├─► packages/console      (command-line parser → core commands, permissions)
  └─► packages/sim-core     (runs in a Web Worker: Rapier, hybrid world, commands, sensors)
            └─► packages/formats (versioned JSON schemas + migrations: World, Challenge, Result)
```

| Package | Responsibility | Depends on |
|---------|----------------|------------|
| `formats` | Schemas (runtime-validated), `formatVersion` per format, forward migrations, golden fixtures. | — |
| `sim-core` | Deterministic simulation: world model, command execution, stepping, sensors, snapshots. Headless (Node and Worker). | `formats`, Rapier (deterministic build) |
| `instruments` | Turns sensor ground truth into measurements `value ± u` according to instrument resolution and noise mode. Pure functions with a seeded PRNG. | `formats` |
| `console` | Parses text commands into the same serializable commands the mouse UI produces; resolves selectors and relative coordinates; enforces per-mode permissions. | `formats`, `sim-core` (types only) |
| `challenges` | The six challenges + tutorial as data; analytic solvers; seeded parameter generation; validation of publishable ranges; bilingual texts and teacher guides. | `formats`, `sim-core`, `instruments` |
| `apps/web` | Screens, three.js rendering, input and build tools, instruments UI, challenge flow, reports, accessibility, i18n, theme, PWA. | all packages |

**Portability rule (D8):** `sim-core`, `instruments`, `console`, `challenges` and `formats` must not use DOM, WebGL/WebGPU or browser-only APIs. A future native build (F4) replaces `apps/web` and, if needed, ports `sim-core` to Rust (Rapier has a near-identical Rust API).

---

## 5. Simulation core (`sim-core`)

### 5.1 World model
- **Units:** SI (m, kg, s). Gravity is configurable; default 9.8 m/s².
- **Voxel grid:** fixed-size world (Phase 1: 128 × 64 × 128 cells), **0.5 m** blocks. Each block has a material (density, static and kinetic friction, restitution) and is either **static** or **dynamic**. Static blocks are merged per 16³ chunk into compound colliders; each dynamic block is its own rigid body.
- **Parts** placed on a **0.1 m** snap grid: beam, plate, wheel, sphere/projectile, counterweight, launch arm. Mass and inertia tensor come from density × geometry unless a mass override is set (inertia then scales with mass).
- **Joints:** fixed, hinge (optional limits and motor), slider, spring (explicit stiffness *k* and damping *c*).
- **Materials (Phase 1):** wood, stone, metal, rubber, ice, each with documented coefficients.

### 5.2 Commands (Command pattern)
Every change to the world is a serializable command: `placeBlock`, `removeBlock`, `fill`, `addPart`, `removePart`, `connect`, `disconnect`, `setProperty`, `setPhysics` (gravity), `trigger` (e.g. release a catapult), `step`, `reset`.
- Commands are validated, applied, and appended to the **command log**.
- Undo/redo is implemented as inverse commands.
- *Initial world + command log* fully determines a run (exact replay, D25), and is the future basis for multiplayer (F6).

### 5.3 Modes
- **Build:** physics paused; editing allowed according to permissions.
- **Run:** starts from a snapshot of the Build state; `reset` returns to that snapshot.

### 5.4 Time stepping
- Fixed step **1/240 s**; solver substeps enabled; **CCD** on projectiles and other fast bodies (D17).
- Slow motion and fast-forward change **steps per rendered frame**, never `dt`.
- The Worker posts body transforms in transferable typed arrays; `apps/web` interpolates between snapshots at display rate.

### 5.5 Sensors
Declarative sensors, defined in challenge data or placed by the user, read **ground truth** every step:
`positionAtContact`, `timeOfFlight`, `maxHeight`, `velocityAt`, `angle`, `contactForce`, `energy` (kinetic, gravitational, elastic per body or system), `momentum`.
Sensor output goes to `instruments`, never directly to the UI.

### 5.6 Determinism (D16)
- Rapier via `@dimforge/rapier3d-deterministic`.
- Seeded PRNG in the core; deterministic trigonometry implemented in the core for all setup math; no `Math.random`, `Date` or `Math.sin/cos` in core code (lint-enforced).
- Stable ordering by entity ID everywhere iteration order could matter.
- Every result records app version, engine version and format versions.

### 5.7 Player
- Rapier kinematic character controller: walk (WASD), jump, crouch, sprint; creative flight (double-tap jump) in sandbox (D26).
- In challenge **Run** mode the player never collides with or pushes experiment objects (collision groups), so replays stay deterministic. Sandbox runs where the player pushed objects are flagged **non-verifiable**.

---

## 6. Instruments and uncertainty (`instruments`, D15)

- Each instrument has a **resolution** and a **noise model**:

  | Instrument | Resolution | Realistic-mode noise (1σ) |
  |------------|------------|---------------------------|
  | Ruler / measuring tape | 0.01 m | 0.005 m |
  | Stopwatch (manual) | 0.01 s | 0.10 s (human reaction) |
  | Photogate | 0.001 s | 0.0005 s |
  | Protractor | 0.5° | 0.25° |
  | Force probe | 0.01 N | 1 % of reading |
  | Mass scale | 0.001 kg | 0.0005 kg |

- **Modes:** `ideal` (resolution only) and `realistic` (resolution + seeded Gaussian noise). Challenges choose the mode; sandbox defaults to `realistic`.
- Standard uncertainty: `u = sqrt((resolution/√12)² + σ²)`; a measurement is reported as `value ± u` with units and the instrument used.
- Repeated measurements produce independent noise draws from the run's seeded PRNG, so students can average and compute spread, and replays reproduce the same draws.

---

## 7. Challenges and data flow (D10, D11, D25)

### 7.1 Challenge format (JSON, versioned)
Illustrative example; the normative schema lives in `packages/formats`.
```jsonc
{
  "format": "physics-lab/challenge", "formatVersion": 1,
  "id": "catapult-range", "version": 3,
  "title": { "es": "…", "en": "…" },
  "topic": "projectile-motion",
  "scenario": { "world": { "format": "physics-lab/world", "formatVersion": 1, "blocks": [], "parts": [], "joints": [] },
                "locked": ["#ground", "#target-zone"] },
  "parameters": [
    { "name": "launchAngle", "bind": "#catapult.armReleaseAngle", "unit": "deg",
      "range": [20, 60], "distribution": "uniform", "step": 1 }
  ],
  "tasks": [
    { "id": "predict-range", "prompt": { "es": "…", "en": "…" },
      "quantity": "range", "unit": "m", "sensor": "positionAtContact(#ball, #ground).x",
      "tolerance": { "kind": "combined-uncertainty", "coverage": 2 } }
  ],
  "solverId": "projectile.range.v1",
  "instrumentMode": "realistic",
  "allowedCommands": ["give", "measure", "record", "time", "help"]
}
```

### 7.2 Teacher flow (no backend)
1. Choose a challenge; the app generates a 128-bit **teacher seed** with `crypto.getRandomValues` (the teacher can also type one to reproduce an earlier assignment); set parameter ranges (within the publishable range), instrument mode, tolerance and allowed commands.
2. The generator checks the ranges with the analytic solver (every combination must be physically solvable and inside validated bounds).
3. Output: a **link** with the challenge compressed (deflate + base64url) in the URL fragment, or a `.json` file when the encoded link exceeds 2,000 characters.

### 7.3 Student flow (predict → observe → explain)
1. Open the link; enter name or student ID. **Per-student seed** = first 32 bits of `SHA-256(teacherSeed ‖ normalize(studentId))`, where `normalize` = trim, Unicode NFC, lowercase; parameters are drawn from it.
2. **Predict** each task's quantity with units (notebook calculator available).
3. **Build/adjust** within the challenge locks and **Run**.
4. **Observe:** sensors + instruments give `measurement ± u`.
5. **Compare** prediction, theory (solver) and measurement; a prediction passes when `|prediction − measurement| ≤ k·u_c` (default *k* = 2, `u_c` combines measurement uncertainty and the solver's stated model tolerance). Differences between theory and measurement are shown openly as discussion material.
6. **Explain** in a short written answer.
7. **Export:** human-readable PDF report, CSV of all measurements, and the **result JSON**.

### 7.4 Result format (JSON, versioned)
Contains: format/app/engine versions; challenge id and version; teacher seed and student ID; drawn parameters; predictions and explanations; full **command log**; measurements with uncertainties; learning events (attempt count, timestamps relative to session start, prediction revisions).

### 7.5 Replay verification
The teacher view loads a result JSON, rebuilds the scenario from the challenge + seed, **re-executes the command log**, and compares every recorded measurement. Any mismatch marks the result as **not verified** and lists the differing values. This works without servers or signatures because of D16.

### 7.6 Error handling
| Situation | Behavior |
|-----------|----------|
| Encoded challenge too long for a URL | Offer `.json` download instead of a link. |
| Older format version | Migrate forward automatically; show a notice. |
| Newer format version than the app | Refuse with a clear message to update (reload the PWA). |
| Engine version differs from a result's | Replay still runs; verification is reported as "engine changed — not comparable" instead of a false mismatch. |
| Invalid/corrupt JSON | Schema error with the failing path; nothing is applied. |
| Unsolvable parameter range | Generator blocks publishing and names the offending range. |

---

## 8. Challenge catalog and validation targets

Each challenge has an analytic solver and an engine-validation benchmark. Tolerances are **initial targets**; changing one requires a decision-log entry.

| # | Challenge | Physics | Solver model | Validation benchmark | Tolerance |
|---|-----------|---------|--------------|----------------------|-----------|
| 0 | Tutorial | Controls, prediction loop | — | — | — |
| 1 | Catapult | 2D kinematics, projectile | Range/height/time from release state (no drag) | Range of a free projectile vs `v²·sin2θ/g` | ≤ 0.5 % relative |
| 2 | Inclined plane | 2nd law, static/kinetic friction | Slip condition `tanθ > μs`; time `t = sqrt(2L / (g(sinθ − μk cosθ)))` | Slide time; slip threshold angle | ≤ 1 % time; ±0.5° threshold |
| 3 | Spring launcher | Hooke's law, energy | `½kx² = ½mv² + mgh` | Energy conservation over one oscillation (c = 0) | ≤ 0.5 % energy drift |
| 4 | Cart collisions | Momentum, restitution | 1D elastic/inelastic with coefficient *e* | Total momentum; post-collision velocities | ≤ 0.1 % momentum; ≤ 2 % velocities |
| 5 | Trebuchet | Rotation, torque, moment of inertia, energy transfer | Rigid-arm energy budget → release speed | Counterweight energy → projectile energy accounting | ≤ 3 % energy balance |
| 6 | Tipping tower | Center of mass, stability | Tip when tilt exceeds `arctan(b/h)` | Tipping angle of a block on a tilting platform | ±0.5° |

Additional engine benchmarks (no dedicated challenge): pendulum period at 5° amplitude vs `2π√(L/g)` with finite-amplitude correction (≤ 0.5 %); rolling without slipping on an incline, `a = g sinθ / (1 + I/mr²)` (≤ 1 %); angular momentum of a free spinning body (≤ 0.5 % drift).

Mass ratios between jointed bodies are limited to 10:1 by default (D17); the trebuchet benchmark must pass at its maximum published ratio.

---

## 9. Command console (`console`, D24)

- Open with `/` or `T`. Autocomplete, history, inline bilingual help (`/help <command>`).
- **Grammar:** `/<command> <args…>`; coordinates absolute (`10 2 5`) or relative to the player (`~ ~1 ~-3`); selectors `@selected`, `@all[type=<partType>]`, `#<id>`.
- **Phase 1 commands:**

  | Category | Commands |
  |----------|----------|
  | build | `summon`, `setblock`, `fill`, `remove`, `connect`, `set` |
  | world | `physics gravity`, `time scale`, `seed` |
  | instruments | `give`, `measure`, `record start|stop|export` |
  | meta | `help`, `undo`, `redo`, `reset` |

- Every console command compiles to the same serializable core commands as the mouse UI (§5.2); the console adds no capabilities the core does not already have.
- **Permissions:** a challenge's `allowedCommands` whitelists commands in its Run and Build modes; everything is allowed in sandbox. Denied commands explain why.

---

## 10. Browser experience (`apps/web`, D26)

### 10.1 Screens
- **Home:** sandbox, built-in challenges, open challenge/result (file or link), language, accessibility settings.
- **World:** modes Build, Run, and Teacher view (configure a challenge, verify results).
- **Side panel (contextual):** selected object properties, challenge loop (predict → observe → explain), notebook with calculator.

### 10.2 Controls
Orbit camera and first-person walk mode (§5.7); hotbar with number-key shortcuts; join tool (part A → part B → joint type) with visible snapping; console; undo/redo (Ctrl+Z / Ctrl+Y). Everything is operable by keyboard alone.

### 10.3 Instruments UI
Force, velocity and acceleration vectors distinguished by **color and line/arrowhead shape**; ruler, stopwatch, protractor, strobe ghosts; live graphs, each with an accessible data table; measurements always shown as `value ± u` with units.

### 10.4 Accessibility (D14, D29)
WCAG 2.2 AA, including 2.4.11 Focus Not Obscured, 2.5.7 Dragging Movements (every drag in building and camera control has a single-click or keyboard alternative), 2.5.8 Target Size (≥ 24×24 CSS px), 3.3.7 Redundant Entry and 3.2.6 Consistent Help: full keyboard operation with visible focus; a live **scene-state region** for screen readers announcing events (e.g. "ball hit the ground at x = 12.3 ± 0.1 m"); AA contrast; colorblind-safe palette; reduced-motion mode; scalable text. VPAT/ACR and HECVAT answers are produced before US sales.

### 10.5 Internationalization (D9)
Spanish and English for all UI and content; locale-aware number formatting (decimal comma or point); SI units. CI fails when any key is missing in either language.

### 10.6 Theme layer (D22)
Greybox look defined entirely by theme tokens (colors, materials, outlines, vector/instrument styles), so the final art direction (F11) is a theme swap.

### 10.7 Performance (D18)
three.js `WebGPURenderer` with automatic WebGL 2 fallback; quality tiers low / medium / high, auto-selected by measured FPS and overridable. Budgets: 60 fps with ~300 dynamic bodies on medium; ≥ 30 fps on a Celeron-class Chromebook on low. React UI lives outside the three.js canvas.

### 10.8 Offline and size (D19)
PWA with a service worker caching the app shell, engine and built-in challenges; works offline after first load. Initial download ≤ 15 MB compressed (CI-enforced).

### 10.9 Privacy (D20)
No third-party requests: fonts, libraries (including KaTeX if used) and WASM are self-hosted; no analytics or telemetry. Student names and results exist only in local files the student exports.

---

## 11. Quality and testing (D27)

| Package | Tests |
|---------|-------|
| `formats` | Schema validation, migration chains, golden fixtures per version. |
| `sim-core` | Command unit tests; **determinism**: identical command log → identical snapshot hash in Node, Chromium, Firefox and WebKit; **engine validation suite** (§8). |
| `challenges` | Property-based sweeps across each publishable parameter range: solver vs simulation within tolerance; range-validation tests. |
| `instruments` | Statistical tests: sample mean and standard deviation of the noise within bounds; resolution rounding. |
| `console` | Parser, selectors, relative coordinates, permission enforcement. |
| `apps/web` | Component tests; axe-core automated a11y plus a manual keyboard/screen-reader checklist per milestone; i18n completeness; Playwright e2e of the main flows run at the end of each milestone; benchmark scenes on the reference Chromebook. |

CI on every change: lint (including the core's banned-API rules), typecheck, unit tests, validation suite, i18n completeness, bundle-size budget.

---

## 12. Phase 1 milestones

| Milestone | Delivers | Exit criteria |
|-----------|----------|---------------|
| **M1 — Vertical slice (greybox)** | Catapult challenge end to end: voxel ground, basic parts, hinge joint, CCD, sensors, instruments with noise, basic console, predict/compare, result JSON. | A student can complete the catapult challenge in a browser; the catapult benchmark passes; replay of the result JSON verifies. |
| **M2 — Full builder** | All parts, joints and materials; walk mode (jump, crouch, sprint, flight); undo/redo; save/load worlds; full console. | Sandbox can build every catalog machine; determinism tests pass across browsers. |
| **M3 — Content** | Challenges 2–6, tutorial, all solvers, complete validation suite. | Every §8 tolerance met across publishable ranges in CI. |
| **M4 — Teacher flow** | Parameter ranges, seeded links/files, replay verification view, PDF/CSV reports. | Teacher can assign, collect and verify a full class set offline. |
| **M5 — Pilot-ready** | WCAG 2.2 AA audit, complete es/en, offline PWA, quality tiers on Chromebook, teacher guides. | All success criteria in §1.3 met. |

Art direction (F11) runs in parallel from M2 on the theme layer.

---

## 13. Risks and mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Deterministic Rapier build is too slow on Chromebooks | Missed performance budget | Benchmark in M1 on the reference device; reduce dynamic-body budget per tier; static-block merging; worker isolation. |
| Joint instability at large mass ratios (trebuchet) | Wrong physics, failed validation | Substeps, 10:1 default limit, benchmark at max published ratio, tune solver iterations per scene. |
| CCD motion clamping loses simulated time for fast projectiles | Range/time bias | Increase CCD substeps; include the effect in the catapult benchmark tolerance. |
| 3D canvas accessibility falls short of AA | Blocks US university sales | Design a11y alternatives from M1; console as full keyboard path; audits per milestone. |
| Bilingual content doubles authoring effort | Slower M3/M5 | Single source of truth per challenge with both languages side by side; i18n CI check. |
| Scope creep from deferred features | MVP never ships | Decision log is the only way to change scope; deferred items stay in later phases. |
| WebGPU behavior differs across browsers | Rendering bugs | WebGL 2 fallback path tested in CI on all three engines. |

---

## 14. Items for owner review

1. **Product name:** "Physics Lab" is a working title.
2. **Repository:** this dedicated monorepo (`physics-lab`, created 2026-10-09, D28) holds the design documents and all implementation.
3. **World size and grid** (128 × 64 × 128 cells of 0.5 m) and **instrument noise values** (§6) are proposed defaults; confirm or adjust.
