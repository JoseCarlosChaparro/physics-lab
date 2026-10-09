# M1 — Catapult Vertical Slice (Greybox) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
> **Execution method (owner, 2026-10-09): subagent-driven.** Not started; begin with Task 1 on branch `feature/m1-vertical-slice`.
> 21 tasks, in dependency order. Each task's **Interfaces** block lists the exact names and types neighboring tasks use.

**Goal:** A student opens the app in a desktop browser, completes the catapult challenge end to end (launch test → measure release → predict → run → measure → compare → explain → export result JSON) in greybox, the catapult benchmark passes, and the exported result JSON verifies by deterministic replay.

**Architecture:** pnpm TypeScript monorepo. Headless, deterministic core packages (`det-math`, `formats`, `sim-core`, `instruments`, `console`, `challenges`) with ports-and-adapters; Rapier sits behind a `PhysicsEngine` port. `apps/web` (Vite + React) is the only package that touches the DOM: it runs a `LabSession` inside a Web Worker (composition root wires Rapier), renders with three.js `WebGPURenderer` (automatic WebGL 2 fallback), and keeps all React UI outside the canvas.

**Tech Stack:** Node 24, pnpm 10.34.6, TypeScript 6.0.x (D38), Zod 4, `@dimforge/rapier3d-deterministic-compat` 0.21.0 (exact, D43), three.js, React 19, Vite, Tailwind CSS v4 + shadcn/ui, react-intl, Vitest 5 (+ browser mode), fast-check, Playwright, ESLint 9 flat config + typescript-eslint, Prettier, Husky + lint-staged.

**Spec:** `docs/design/2026-10-09-physics-lab-mvp-design.md` (approved, D35) and `docs/design/decisions-log.md` (D1–D45, F1–F17). Engineering rules: `docs/engineering/guidelines.md`.

## Global Constraints

- SI units everywhere; the unit is in the name, the type or the doc comment (`launchSpeedMps`, `FIXED_STEP_S`).
- Fixed step **1/240 s**; solver substeps on; CCD on projectiles; slow motion changes steps per frame, never `dt` (spec §5.4, D23).
- Default gravity **9.8 m/s²** (spec §5.1).
- World **128 × 64 × 128** cells of **0.5 m**; static blocks merged per **16³** chunk (spec §5.1, D34).
- Instruments exactly as D34: ruler 0.01 m / σ 0.005 m; stopwatch 0.01 s / σ 0.10 s; photogate 0.001 s / σ 0.0005 s; protractor 0.5° / σ 0.25°; force probe 0.01 N / σ 1 % of reading; mass scale 0.001 kg / σ 0.0005 kg. `u = sqrt((resolution/√12)² + σ²)`; `ideal` mode has σ = 0.
- Catapult benchmark tolerance **≤ 0.5 % relative** on range (spec §8). Changing any tolerance requires a decisions-log entry.
- Pass rule: `|prediction − measurement| ≤ k·u_c`, default `k = 2`, `u_c = sqrt(u_meas² + u_theory² + (tol·theory)²)` (D42).
- Core packages (`packages/*`): no DOM/browser APIs, no `three`/`react`, no `Math.random`, `Date`, `performance`, `Math.sin/cos/tan/atan/atan2/exp/log/pow/hypot/cbrt…`, no `**`; iterate in stable entity-ID order; use `@physics-lab/det-math` (D16, D31, D45). Enforced by lint (Task 1) and by compiling core packages without the DOM lib.
- Dependency direction: `web → challenges, console, instruments, sim-core, formats, det-math`; `challenges → sim-core, instruments, formats, det-math`; `console → sim-core, formats`; `sim-core → formats, det-math`; `instruments → formats, det-math`; `formats`, `det-math` → nothing.
- Every world change is a serializable `Command` from `@physics-lab/formats`; console and mouse UI produce the same commands.
- Dependency injection: ports (`PhysicsEngineFactory`, `RandomSource`, `Clock`, `SimClient`, renderer factory, download, frame scheduler) wired only in composition roots (`apps/web/src/main.tsx`, `apps/web/src/sim/sim.worker.ts`). No module-level mutable state, no singletons.
- i18n: no hard-coded user-facing strings in `apps/web`; every key in `es` and `en`; ICU messages; locale-aware numbers (decimal comma in `es`).
- WCAG 2.2 AA: keyboard path for everything, visible focus, focus never obscured, a non-drag alternative for every drag (camera), targets ≥ 24×24 CSS px, color never the only signal, live scene-state region, data table for measurements.
- Visuals only through theme tokens (`apps/web/src/theme/`); hard-coded colors elsewhere fail lint.
- Zero third-party requests; CSP `script-src 'self' 'wasm-unsafe-eval'` (D41); Zod `jitless: true` in composition roots.
- Responsive (D32): Home works from 320 px to 2560 px, no horizontal scroll at 320 px; the 3D World is desktop-only and shows a notice on small or touch-only screens.
- Prettier: `semi: false`, `singleQuote: true`, `trailingComma: "es5"`, `printWidth: 100`, `tabWidth: 2`.
- Conventional Commits in English, one logical change per commit. Work on branch `feature/m1-vertical-slice`.
- Per-task verification: `pnpm lint && pnpm typecheck && pnpm test` (+ `pnpm --filter @physics-lab/web build` when web config changes). **Do not run Playwright or Vitest browser mode during tasks** — only in Task 20 at milestone end (owner directive). Report e2e specs written but not run.

## Review Focus

1. **Decimal comma input** — a student in `es` types `12,5` (or `12.5`) as a prediction; expected: both parse to 12.5, `1.234,5`-style grouping is rejected with a clear message, never silently misread. Test: Task 18 (`number-format.test.ts`, `ChallengePanel.test.tsx`).
2. **Student ID with whitespace or composed Unicode** — `"  José "` vs `"José"`; expected: trimmed, NFC-normalized, 1–64 chars, the same ID always gives the same result file. Tests: Task 3 (`primitives.test.ts`), Task 13 (`LabSession.create`), Task 15 (`HomeScreen.test.tsx`).
3. **Tampered or malformed result JSON** — edited measurement, `atStep` going backwards, unknown quantity, oversize file; expected: `verifyResult` returns `mismatch` / `invalidLog`, the loader returns typed errors, nothing throws. Tests: Task 4 (loader), Task 13 (`verify-result.test.ts`).
4. **Out-of-order student actions** — measuring range before predicting, launching twice, continuing without a prediction, revising a prediction after measuring the range; expected: typed `LabError` codes with translated messages, no state corruption, buttons disabled where possible. Tests: Task 13 (`lab-session.test.ts`), Task 18 (`ChallengePanel.test.tsx`), Task 19 (`WorldScreen.test.tsx`).
5. **Long frame gaps** — tab hidden for minutes, then shown; expected: steps per frame are capped (no spiral of death), simulation stays deterministic because time is only steps. Test: Task 17 (`frame-loop.test.ts`).

## Model routing (CLAUDE.md, D30)

| Tasks | Implementer | Reviewers after the task |
|-------|-------------|--------------------------|
| 2, 3, 4, 5, 6, 10, 14 | `implementer` (Haiku) | `code-reviewer`; `physics-reviewer` for 2, 5, 10 |
| 1, 7, 8, 9, 11, 12, 13, 16, 20, 21 | `senior-implementer` (Sonnet) | `code-reviewer`; `physics-reviewer` for 7, 8, 9, 11, 12, 13, 21 |
| 15, 17, 18, 19 | `senior-implementer` (Sonnet) | `code-reviewer` + `a11y-reviewer` |

Escalate one tier after two failed attempts. Final whole-branch review: `code-reviewer`, `physics-reviewer` and `a11y-reviewer` on Opus.

## File Structure

```
package.json · pnpm-workspace.yaml · tsconfig.base.json · tsconfig.json · eslint.config.js
vitest.config.ts · .prettierrc · .prettierignore · .gitignore · README.md
.husky/pre-commit · .github/workflows/{ci,e2e}.yml · .claude/settings.json
tools/tsconfig.json · tools/eslint/lint-rules.test.ts · tools/hooks/stop-verify.sh
scripts/check-bundle-size.mjs
docs/engineering/m1-exit-checklist.md

packages/det-math/src/
  random.ts            RandomSource port + SeededRandom (sfc32 seeded by splitmix32)
  hash.ts              fnv1a32, SnapshotHasher (FNV-1a 64)
  trig.ts              detSin, detCos, detAtan, detAtan2 (fdlibm kernels)
  ln.ts                detLn
  angles.ts            degToRad, radToDeg
  index.ts

packages/formats/src/
  primitives.ts        EntityId, EntityRef, Vec3, GridCoord, LocalizedText, Uint32, student id
  world.ts             WorldDoc v1 (blocks, parts, joints, latches) + world constants
  measurement.ts       QuantityId, InstrumentId, Unit, Measurement
  commands.ts          Command union, CommandLogEntry
  challenge.ts         Challenge v1
  result.ts            Result v1
  load.ts              versioned loader: size cap, JSON, format, version, migrations, schema
  index.ts
packages/formats/fixtures/{world,challenge,result}.v1.json   golden fixtures

packages/sim-core/src/
  geometry/{vec,quat}.ts
  materials.ts
  voxel/{voxel-grid,static-merge}.ts
  commands/build-commands.ts     applyBuildCommand (pure WorldDoc → WorldDoc)
  ports/physics-engine.ts        PhysicsEngine + PhysicsEngineFactory ports
  adapters/rapier/{rapier-physics-engine,index}.ts
  simulation/simulation.ts       Simulation: build from WorldDoc, step, trigger, latches, events, hash
  sensors/{sensor-host,projectile-flight-sensor}.ts
  testing/{physics-engine-contract,fake-sensor-host,catapult-test-world}.ts
  validation/free-projectile.validation.test.ts
  determinism/{determinism.test.ts,determinism.browser.test.ts,__golden__/}
  index.ts

packages/instruments/src/{instrument-specs,gaussian,measure,derived,index}.ts

packages/console/src/{tokenize,console-types,commands,create-console,index}.ts

packages/challenges/src/
  solvers/{solver,registry,projectile-range,uncertainty}.ts
  comparison.ts
  parameters.ts
  catalog/{catapult-range.challenge.json,catalog.ts}
  session/{lab-error,run-state,create-run,lab-session,verify-result}.ts
  validation/catapult.validation.test.ts
  index.ts

apps/web/
  index.html · vite.config.ts · vitest.config.ts · tsconfig.json · components.json
  playwright.config.ts · e2e/flows/{catapult,home-responsive}.spec.ts
  src/main.tsx                     composition root
  src/app/{App,dependencies}.tsx
  src/theme/{tokens,apply-theme,contrast}.ts
  src/index.css
  src/i18n/{locales.ts,messages/es.json,messages/en.json}
  src/components/ui/*              shadcn (generated)
  src/lib/utils.ts
  src/format/number-format.ts
  src/home/HomeScreen.tsx
  src/sim/{protocol,describe-scene,sim-worker-handler,sim-worker-client,sim.worker}.ts
  src/render/{orbit-camera,interpolation,frame-loop,scene-renderer}.ts
  src/world/{WorldScreen,WorldCanvas,ChallengePanel,MeasurementTable,PredictionForm,
             ComparisonView,ExplanationForm,RunControls,CameraControls,SceneStatusRegion,
             lab-reducer}.tsx|ts
  src/console/ConsolePanel.tsx
  src/bench/{BenchScreen.tsx,benchmark-world.ts}
  src/io/download-json.ts
  src/test/{setup,axe,fake-sim-client}.ts
```

---

### Task 1: Monorepo scaffold, tooling and enforcement

**Files:**
- Create: `package.json`, `pnpm-workspace.yaml`, `tsconfig.base.json`, `tsconfig.json`, `eslint.config.js`, `vitest.config.ts`, `.prettierrc`, `.prettierignore`, `.gitignore`, `README.md`, `.husky/pre-commit`, `.github/workflows/ci.yml`, `.claude/settings.json`, `tools/tsconfig.json`, `tools/hooks/stop-verify.sh`, `tools/eslint/lint-rules.test.ts`
- Create for each of `det-math`, `formats`, `sim-core`, `instruments`, `console`, `challenges`: `packages/<name>/package.json`, `packages/<name>/tsconfig.json`, `packages/<name>/vitest.config.ts`, `packages/<name>/src/index.ts`

**Interfaces:**
- Produces: workspace packages `@physics-lab/<name>` consumed as TS source (`exports` → `./src/index.ts`, D37); root scripts `lint`, `format`, `format:check`, `typecheck`, `test`, `test:validation`, `test:browser`, `test:e2e`; the lint rules every later task relies on.

- [ ] **Step 1: Repair the toolchain and create the branch**

```bash
corepack enable
corepack prepare pnpm@10.34.6 --activate || npm install -g pnpm@10.34.6
pnpm -v            # expected: 10.34.6
node -v            # expected: v24.x
git switch -c feature/m1-vertical-slice
```

- [ ] **Step 2: Root workspace files**

`package.json`:
```json
{
  "name": "physics-lab",
  "private": true,
  "type": "module",
  "packageManager": "pnpm@10.34.6",
  "engines": { "node": ">=24" },
  "scripts": {
    "lint": "eslint .",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "typecheck": "tsc --noEmit -p tsconfig.json && tsc --noEmit -p tools/tsconfig.json && pnpm -r typecheck",
    "test": "vitest run --project '!sim-core-browser'",
    "test:validation": "vitest run --project '!sim-core-browser' validation",
    "test:browser": "vitest run --project sim-core-browser",
    "test:e2e": "pnpm --filter @physics-lab/web test:e2e",
    "prepare": "husky"
  },
  "lint-staged": {
    "*.{ts,tsx,js,mjs}": ["eslint --fix", "prettier --write"],
    "*.{json,md,css,yml,yaml}": ["prettier --write"]
  }
}
```

`pnpm-workspace.yaml`:
```yaml
packages:
  - 'apps/*'
  - 'packages/*'

onlyBuiltDependencies:
  - '@tailwindcss/oxide'
  - esbuild
```

`tsconfig.base.json`:
```json
{
  "compilerOptions": {
    "target": "ES2023",
    "lib": ["ES2023"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "noEmit": true,
    "types": []
  }
}
```

`tsconfig.json` (root config files only):
```json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": { "types": ["node"] },
  "include": ["vitest.config.ts"]
}
```

`tools/tsconfig.json`:
```json
{
  "extends": "../tsconfig.base.json",
  "compilerOptions": { "types": ["node"] },
  "include": ["**/*.ts"]
}
```

`.prettierrc`:
```json
{ "semi": false, "singleQuote": true, "trailingComma": "es5", "printWidth": 100, "tabWidth": 2 }
```

`.prettierignore`:
```
pnpm-lock.yaml
**/dist
**/coverage
**/playwright-report
**/test-results
**/__golden__
apps/web/src/components/ui
```

`.gitignore` (framework baseline + D40 additions):
```gitignore
# Environment
.env
.env.local
.env.*.local

# Dependencies
node_modules/

# Build output
.next/
dist/
out/
.vite/
*.tsbuildinfo
coverage/

# OS
.DS_Store
Thumbs.db

# Editor
.vscode/*
!.vscode/extensions.json
.idea/

# Logs
*.log
npm-debug.log*

# Test artifacts
playwright-report/
test-results/
```

`vitest.config.ts`:
```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    projects: [
      'packages/*',
      'packages/sim-core/vitest.browser.config.ts',
      'apps/*',
      { test: { name: 'tools', include: ['tools/**/*.test.ts'], environment: 'node' } },
    ],
  },
})
```

> `packages/sim-core/vitest.browser.config.ts` and `apps/web` are created in Tasks 20 and 15. Until then Vitest warns about the missing project paths; that is expected. If Vitest 5 fails hard on a missing path, add those two entries in the tasks that create them instead.

- [ ] **Step 3: Package skeletons**

For each `<name>` in `det-math formats sim-core instruments console challenges`:

`packages/<name>/package.json` (example for `formats`; set `name` accordingly):
```json
{
  "name": "@physics-lab/formats",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "exports": { ".": "./src/index.ts" },
  "scripts": { "typecheck": "tsc --noEmit -p tsconfig.json" }
}
```

`packages/<name>/tsconfig.json` (no DOM lib — portability is checked by the compiler):
```json
{
  "extends": "../../tsconfig.base.json",
  "include": ["src", "vitest.config.ts"]
}
```

`packages/<name>/vitest.config.ts`:
```ts
import { defineProject } from 'vitest/config'

export default defineProject({
  test: { name: 'formats', include: ['src/**/*.test.ts'], environment: 'node' },
})
```
(Use the package's short name as `test.name`.)

`packages/<name>/src/index.ts`:
```ts
export {}
```

Workspace dependencies (later tasks import them):
```bash
pnpm --filter @physics-lab/sim-core add @physics-lab/formats@workspace:* @physics-lab/det-math@workspace:*
pnpm --filter @physics-lab/instruments add @physics-lab/formats@workspace:* @physics-lab/det-math@workspace:*
pnpm --filter @physics-lab/console add @physics-lab/formats@workspace:* @physics-lab/sim-core@workspace:*
pnpm --filter @physics-lab/challenges add @physics-lab/formats@workspace:* @physics-lab/sim-core@workspace:* @physics-lab/instruments@workspace:* @physics-lab/det-math@workspace:*
pnpm --filter @physics-lab/formats add zod@^4.6.5
pnpm --filter @physics-lab/sim-core add -E @dimforge/rapier3d-deterministic-compat@0.21.0
```

- [ ] **Step 4: Root dev tooling**

```bash
pnpm add -D -w typescript@~6.0.3 eslint@^9 @eslint/js@^9 typescript-eslint globals \
  eslint-plugin-jsx-a11y eslint-plugin-react-hooks prettier vitest@^5.0.3 fast-check \
  husky lint-staged @types/node@^24
pnpm exec husky init
printf 'pnpm exec lint-staged\n' > .husky/pre-commit
```

- [ ] **Step 5: Write the failing lint-rule tests**

`tools/eslint/lint-rules.test.ts`:
```ts
import { rm, writeFile } from 'node:fs/promises'
import { join } from 'node:path'
import { ESLint } from 'eslint'
import { afterEach, describe, expect, it } from 'vitest'

const REPO_ROOT = join(import.meta.dirname, '..', '..')
const eslint = new ESLint({ cwd: REPO_ROOT })
const probes: string[] = []

async function ruleIdsForProbe(directory: string, code: string, fileName = 'probe.ts'): Promise<string[]> {
  const path = join(REPO_ROOT, directory, `lint-${process.pid}-${probes.length}-${fileName}`)
  probes.push(path)
  await writeFile(path, code)
  const [result] = await eslint.lintFiles([path])
  return (result?.messages ?? []).map((message) => message.ruleId ?? `fatal: ${message.message}`)
}

afterEach(async () => {
  await Promise.all(probes.splice(0).map((path) => rm(path, { force: true })))
})

describe('determinism rules in core packages (D16, D31)', () => {
  it('rejects Math.random', async () => {
    const ids = await ruleIdsForProbe('packages/sim-core/src', 'export const r = Math.random()\n')
    expect(ids).toContain('no-restricted-properties')
  })

  it('rejects Math.sin', async () => {
    const ids = await ruleIdsForProbe('packages/instruments/src', 'export const s = Math.sin(1)\n')
    expect(ids).toContain('no-restricted-properties')
  })

  it('rejects new Date()', async () => {
    const ids = await ruleIdsForProbe('packages/challenges/src', 'export const d = new Date()\n')
    expect(ids).toContain('no-restricted-syntax')
  })

  it('rejects the exponentiation operator', async () => {
    const ids = await ruleIdsForProbe('packages/det-math/src', 'export const p = 2 ** 3\n')
    expect(ids).toContain('no-restricted-syntax')
  })

  it('allows Math.sin in core test files for reference comparisons', async () => {
    const ids = await ruleIdsForProbe(
      'packages/det-math/src',
      "import { expect, it } from 'vitest'\nit('x', () => { expect(Math.sin(0)).toBe(0) })\n",
      'probe.test.ts'
    )
    expect(ids).not.toContain('no-restricted-properties')
  })
})

describe('portability rules in core packages (D8)', () => {
  it('rejects DOM globals', async () => {
    const ids = await ruleIdsForProbe('packages/formats/src', 'export const w = window\n')
    expect(ids).toContain('no-restricted-globals')
  })

  it('rejects rendering imports', async () => {
    const ids = await ruleIdsForProbe('packages/challenges/src', "export * from 'three'\n")
    expect(ids).toContain('no-restricted-imports')
  })
})

describe('dependency direction (spec §4)', () => {
  it('rejects formats importing sim-core', async () => {
    const ids = await ruleIdsForProbe('packages/formats/src', "export * from '@physics-lab/sim-core'\n")
    expect(ids).toContain('no-restricted-imports')
  })

  it('rejects instruments importing sim-core', async () => {
    const ids = await ruleIdsForProbe('packages/instruments/src', "export * from '@physics-lab/sim-core'\n")
    expect(ids).toContain('no-restricted-imports')
  })

  it('allows sim-core importing formats', async () => {
    const ids = await ruleIdsForProbe('packages/sim-core/src', "export * from '@physics-lab/formats'\n")
    expect(ids).not.toContain('no-restricted-imports')
  })

  it('rejects sim-core core code importing Rapier directly', async () => {
    const ids = await ruleIdsForProbe(
      'packages/sim-core/src',
      "export * from '@dimforge/rapier3d-deterministic-compat'\n"
    )
    expect(ids).toContain('no-restricted-imports')
  })

  it('allows the Rapier adapter to import Rapier', async () => {
    const ids = await ruleIdsForProbe(
      'packages/sim-core/src/adapters',
      "export * from '@dimforge/rapier3d-deterministic-compat'\n"
    )
    expect(ids).not.toContain('no-restricted-imports')
  })
})
```

Create the directory the last test writes into: `mkdir -p packages/sim-core/src/adapters`.

- [ ] **Step 6: Run the tests to verify they fail**

Run: `pnpm install && pnpm vitest run --project tools`
Expected: FAIL — ESLint reports that no config file exists.

- [ ] **Step 7: Write the ESLint config**

`eslint.config.js`:
```js
import js from '@eslint/js'
import jsxA11y from 'eslint-plugin-jsx-a11y'
import reactHooks from 'eslint-plugin-react-hooks'
import globals from 'globals'
import tseslint from 'typescript-eslint'

// Dependency direction (spec §4, D45): workspace packages each core package may import.
const ALLOWED_WORKSPACE_IMPORTS = {
  'det-math': [],
  formats: [],
  'sim-core': ['det-math', 'formats'],
  instruments: ['det-math', 'formats'],
  console: ['formats', 'sim-core'],
  challenges: ['det-math', 'formats', 'sim-core', 'instruments'],
}
const CORE_PACKAGES = Object.keys(ALLOWED_WORKSPACE_IMPORTS)
const WORKSPACE_PACKAGES = [...CORE_PACKAGES, 'web']
const RENDERING_AND_UI_MODULES = ['three', 'three/*', 'react', 'react/*', 'react-dom', 'react-dom/*']
const NON_DETERMINISTIC_MATH = [
  'random', 'sin', 'cos', 'tan', 'asin', 'acos', 'atan', 'atan2', 'sinh', 'cosh', 'tanh',
  'asinh', 'acosh', 'atanh', 'exp', 'expm1', 'log', 'log1p', 'log2', 'log10', 'pow', 'cbrt', 'hypot',
]
const HEADLESS_FORBIDDEN_GLOBALS = [
  'window', 'document', 'navigator', 'localStorage', 'sessionStorage', 'indexedDB',
  'requestAnimationFrame', 'performance', 'self', 'postMessage', 'fetch', 'crypto',
]
const EXPONENT_MESSAGE = 'Exponentiation is not correctly rounded across engines; multiply explicitly.'
const CLOCK_MESSAGE = 'Inject a Clock port instead (D30).'
const THEME_MESSAGE = 'Colors come from theme tokens (D22).'

function forbiddenWorkspaceImports(packageName) {
  const allowed = ALLOWED_WORKSPACE_IMPORTS[packageName]
  return WORKSPACE_PACKAGES.filter((name) => name !== packageName && !allowed.includes(name)).flatMap(
    (name) => [`@physics-lab/${name}`, `@physics-lab/${name}/*`]
  )
}

function coreImportRules(packageName, extraPatterns = []) {
  return {
    'no-restricted-imports': [
      'error',
      {
        patterns: [
          { group: forbiddenWorkspaceImports(packageName), message: 'Violates the package dependency direction (spec §4).' },
          { group: RENDERING_AND_UI_MODULES, message: 'Core packages must not import rendering or UI code (D8).' },
          ...extraPatterns,
        ],
      },
    ],
  }
}

export default tseslint.config(
  {
    ignores: [
      '**/dist/**', '**/node_modules/**', '**/coverage/**', '**/playwright-report/**',
      '**/test-results/**', 'apps/web/src/components/ui/**',
    ],
  },
  js.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,
  { languageOptions: { parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname } } },
  { files: ['**/*.js', '**/*.mjs'], ...tseslint.configs.disableTypeChecked, languageOptions: { globals: globals.node } },
  {
    files: ['**/*.{ts,tsx}'],
    rules: {
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      'no-console': ['error', { allow: ['warn', 'error'] }],
    },
  },
  // Core packages: determinism (D16) and portability (D8), enforced per D31.
  {
    files: CORE_PACKAGES.map((name) => `packages/${name}/src/**/*.ts`),
    rules: {
      'no-restricted-properties': [
        'error',
        ...NON_DETERMINISTIC_MATH.map((property) => ({
          object: 'Math',
          property,
          message: 'Not deterministic across JS engines; use @physics-lab/det-math.',
        })),
        { object: 'Date', property: 'now', message: CLOCK_MESSAGE },
      ],
      'no-restricted-syntax': [
        'error',
        { selector: "NewExpression[callee.name='Date']", message: CLOCK_MESSAGE },
        { selector: "BinaryExpression[operator='**']", message: EXPONENT_MESSAGE },
        { selector: "AssignmentExpression[operator='**=']", message: EXPONENT_MESSAGE },
      ],
      'no-restricted-globals': [
        'error',
        ...HEADLESS_FORBIDDEN_GLOBALS.map((name) => ({
          name,
          message: 'Core packages are headless and portable (D8); inject a port.',
        })),
      ],
    },
  },
  ...CORE_PACKAGES.map((name) => ({ files: [`packages/${name}/src/**/*.ts`], rules: coreImportRules(name) })),
  {
    files: ['packages/sim-core/src/**/*.ts'],
    ignores: ['packages/sim-core/src/adapters/**'],
    rules: coreImportRules('sim-core', [
      {
        group: ['@dimforge/*', '**/adapters/**'],
        message: 'Core code depends on ports; adapters are wired in composition roots (D30).',
      },
    ]),
  },
  // Tests may use Math trig/log as reference values.
  {
    files: ['packages/*/src/**/*.test.ts'],
    rules: { 'no-restricted-properties': 'off', 'no-restricted-syntax': 'off' },
  },
  {
    files: ['apps/web/src/**/*.{ts,tsx}'],
    plugins: { 'jsx-a11y': jsxA11y, 'react-hooks': reactHooks },
    languageOptions: { globals: globals.browser },
    rules: {
      ...jsxA11y.flatConfigs.recommended.rules,
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'error',
    },
  },
  {
    files: ['apps/web/src/**/*.{ts,tsx}'],
    ignores: ['apps/web/src/theme/**'],
    rules: {
      'no-restricted-syntax': [
        'error',
        { selector: 'Literal[value=/^#(?:[0-9a-fA-F]{3,4}|[0-9a-fA-F]{6}|[0-9a-fA-F]{8})$/]', message: THEME_MESSAGE },
        { selector: 'Literal[value=/^(?:rgb|hsl|oklch)a?\\(/]', message: THEME_MESSAGE },
      ],
    },
  }
)
```

- [ ] **Step 8: Run the tests to verify they pass**

Run: `pnpm vitest run --project tools`
Expected: PASS (12 tests).

- [ ] **Step 9: CI workflow, Stop hook and README**

`.github/workflows/ci.yml`:
```yaml
name: CI
on: [push]

permissions:
  contents: read

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm audit --audit-level high
      - run: pnpm lint
      - run: pnpm format:check
      - run: pnpm typecheck
      - run: pnpm test
```

`tools/hooks/stop-verify.sh` (`chmod +x`):
```sh
#!/usr/bin/env sh
# Claude Code Stop hook (D31): typecheck + tests related to uncommitted changes.
# Exit code 2 returns the failure to the agent; a second consecutive stop is let through
# (stop_hook_active) so a failure the agent cannot fix never loops forever.
input=$(cat)
if output=$(pnpm -s typecheck 2>&1 && pnpm -s vitest run --changed --project '!sim-core-browser' --passWithNoTests 2>&1); then
  exit 0
fi
printf '%s\n' "$output" >&2
case "$input" in
  *'"stop_hook_active":true'*|*'"stop_hook_active": true'*) exit 0 ;;
esac
exit 2
```

`.claude/settings.json`:
```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [{ "type": "command", "command": "tools/hooks/stop-verify.sh", "timeout": 600 }]
      }
    ]
  }
}
```

`README.md`:
```markdown
# Physics Lab

Minecraft-style digital physics laboratory (working title, D33).
Design: `docs/design/`. Engineering rules: `docs/engineering/guidelines.md`.

## Development

Requires Node 24 and pnpm 10 (`corepack enable`).

| Command | What it does |
|---------|--------------|
| `pnpm install` | Install workspace dependencies |
| `pnpm lint` / `pnpm typecheck` / `pnpm test` | Per-change verification |
| `pnpm test:validation` | Physics validation suites only |
| `pnpm --filter @physics-lab/web dev` | Run the web app |
| `pnpm test:browser` / `pnpm test:e2e` | Cross-browser determinism and e2e (milestone end only) |
```

- [ ] **Step 10: Verify everything**

Run: `pnpm lint && pnpm format:check && pnpm typecheck && pnpm test`
Expected: all succeed (`pnpm format` first if Prettier reports files).

- [ ] **Step 11: Commit**

```bash
git add -A
git add -- . ":!.github" ":!.claude" ":!tools/hooks"
git commit -m "build: scaffold pnpm monorepo tooling"
git add .github .claude tools/hooks
git commit -m "ci: add verification workflow and claude stop hook"
```

- [ ] **Step 12: Open the PR and make CI a required check (D46)**

```bash
git push -u origin feature/m1-vertical-slice
gh pr create --draft --title "feat: m1 catapult vertical slice" --body "Implements docs/superpowers/plans/2026-10-09-m1-vertical-slice.md"
# after the first CI run has reported the `verify` check:
gh api -X PATCH repos/JoseCarlosChaparro/physics-lab/branches/main/protection/required_status_checks \
  -f strict=true -f 'contexts[]=verify'
```
Expected: the draft PR shows the `verify` check; `main` now refuses merges while it fails.

---
### Task 2: `det-math` — seeded PRNG, deterministic trig/log, hashing

**Files:**
- Create: `packages/det-math/src/random.ts`, `hash.ts`, `trig.ts`, `ln.ts`, `angles.ts`
- Modify: `packages/det-math/src/index.ts`
- Test: `packages/det-math/src/random.test.ts`, `hash.test.ts`, `trig.test.ts`, `ln.test.ts`

**Interfaces:**
- Produces:
  - `interface RandomSource { nextUint32(): number; nextFloat(): number }`
  - `class SeededRandom implements RandomSource { static fromSeed(seed: number): SeededRandom; fork(label: string): SeededRandom }`
  - `fnv1a32(text: string): number`; `class SnapshotHasher { addFloat64(v: number): this; addUint32(v: number): this; addString(s: string): this; digest(): string }`
  - `detSin(x)`, `detCos(x)`, `detAtan(x)`, `detAtan2(y, x)`, `detLn(x)` (radians, `number → number`); `MAX_TRIG_ARGUMENT_RAD`
  - `DEG_TO_RAD`, `degToRad(deg)`, `radToDeg(rad)`

Background: IEEE-754 `+ − × ÷` and `Math.sqrt` are correctly rounded and identical on every engine; `Math.sin`, `Math.log` & co. are not (D16). Everything here uses only exact operations. The trig kernels are the public-domain fdlibm polynomials.

- [ ] **Step 1: Write the failing tests**

`packages/det-math/src/random.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { SeededRandom } from './random'

function take(random: SeededRandom, count: number): number[] {
  return Array.from({ length: count }, () => random.nextUint32())
}

describe('SeededRandom', () => {
  it('produces the same sequence for the same seed', () => {
    expect(take(SeededRandom.fromSeed(42), 50)).toEqual(take(SeededRandom.fromSeed(42), 50))
  })

  it('produces different sequences for different seeds', () => {
    expect(take(SeededRandom.fromSeed(1), 10)).not.toEqual(take(SeededRandom.fromSeed(2), 10))
  })

  it('returns unsigned 32-bit integers', () => {
    for (const value of take(SeededRandom.fromSeed(7), 1000)) {
      expect(Number.isInteger(value) && value >= 0 && value <= 0xffff_ffff).toBe(true)
    }
  })

  it('returns floats uniformly distributed in [0, 1)', () => {
    const random = SeededRandom.fromSeed(3)
    let sum = 0
    for (let i = 0; i < 100_000; i++) {
      const value = random.nextFloat()
      expect(value >= 0 && value < 1).toBe(true)
      sum += value
    }
    expect(sum / 100_000).toBeCloseTo(0.5, 2)
  })

  it('forks reproducible streams without consuming the parent stream', () => {
    const parent = SeededRandom.fromSeed(9)
    const forkA = take(parent.fork('instruments'), 5)
    const parentAfterFork = take(parent, 5)
    expect(take(SeededRandom.fromSeed(9).fork('instruments'), 5)).toEqual(forkA)
    expect(take(SeededRandom.fromSeed(9), 5)).toEqual(parentAfterFork)
    expect(take(SeededRandom.fromSeed(9).fork('parameters'), 5)).not.toEqual(forkA)
  })
})
```

`packages/det-math/src/hash.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { SnapshotHasher, fnv1a32 } from './hash'

describe('fnv1a32', () => {
  it('matches the FNV-1a reference value for "a"', () => {
    expect(fnv1a32('a')).toBe(0xe40c292c)
  })
})

describe('SnapshotHasher', () => {
  it('matches the FNV-1a 64 reference value for the UTF-16LE bytes of "a" (0x61 0x00)', () => {
    expect(new SnapshotHasher().addString('a').digest()).toBe('089be207b544f1e4')
  })

  it('is order sensitive', () => {
    const ab = new SnapshotHasher().addFloat64(1).addFloat64(2).digest()
    const ba = new SnapshotHasher().addFloat64(2).addFloat64(1).digest()
    expect(ab).not.toBe(ba)
  })

  it('treats -0 and +0 as the same value', () => {
    expect(new SnapshotHasher().addFloat64(-0).digest()).toBe(new SnapshotHasher().addFloat64(0).digest())
  })

  it('returns 16 lowercase hex characters', () => {
    expect(new SnapshotHasher().addUint32(123).digest()).toMatch(/^[0-9a-f]{16}$/)
  })
})
```
(`addString` hashes each UTF-16 code unit as two little-endian bytes; the reference values were computed independently from the FNV-1a definitions.)

`packages/det-math/src/trig.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { SeededRandom } from './random'
import { MAX_TRIG_ARGUMENT_RAD, detAtan, detAtan2, detCos, detSin } from './trig'

const ABSOLUTE_TOLERANCE = 1e-14

function samples(seed: number, count: number, min: number, max: number): number[] {
  const random = SeededRandom.fromSeed(seed)
  return Array.from({ length: count }, () => min + (max - min) * random.nextFloat())
}

describe('detSin / detCos', () => {
  it('agree with Math.sin/Math.cos within 1e-14 on [-100, 100]', () => {
    for (const x of samples(1, 20_000, -100, 100)) {
      expect(Math.abs(detSin(x) - Math.sin(x))).toBeLessThanOrEqual(ABSOLUTE_TOLERANCE)
      expect(Math.abs(detCos(x) - Math.cos(x))).toBeLessThanOrEqual(ABSOLUTE_TOLERANCE)
    }
  })

  it('returns exact values at zero', () => {
    expect(detSin(0)).toBe(0)
    expect(detCos(0)).toBe(1)
  })

  it('rejects arguments beyond the accurate reduction range', () => {
    expect(() => detSin(MAX_TRIG_ARGUMENT_RAD * 2)).toThrow(RangeError)
  })
})

describe('detAtan / detAtan2', () => {
  it('agrees with Math.atan within 1e-14', () => {
    for (const x of samples(2, 20_000, -50, 50)) {
      expect(Math.abs(detAtan(x) - Math.atan(x))).toBeLessThanOrEqual(ABSOLUTE_TOLERANCE)
    }
  })

  it('agrees with Math.atan2 in all quadrants', () => {
    const ys = samples(3, 5000, -10, 10)
    const xs = samples(4, 5000, -10, 10)
    ys.forEach((y, i) => {
      const x = xs[i] ?? 1
      expect(Math.abs(detAtan2(y, x) - Math.atan2(y, x))).toBeLessThanOrEqual(ABSOLUTE_TOLERANCE)
    })
  })

  it('handles the axes', () => {
    expect(detAtan2(0, -1)).toBeCloseTo(Math.PI, 15)
    expect(detAtan2(1, 0)).toBeCloseTo(Math.PI / 2, 15)
    expect(detAtan2(-1, 0)).toBeCloseTo(-Math.PI / 2, 15)
    expect(detAtan2(0, 0)).toBe(0)
  })
})
```

`packages/det-math/src/ln.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { detLn } from './ln'
import { SeededRandom } from './random'

describe('detLn', () => {
  it('agrees with Math.log within 1e-14 relative on (1e-12, 1e12)', () => {
    const random = SeededRandom.fromSeed(5)
    for (let i = 0; i < 20_000; i++) {
      const x = Math.exp(-27.6 + 55.2 * random.nextFloat())
      const reference = Math.log(x)
      const error = Math.abs(detLn(x) - reference)
      expect(error).toBeLessThanOrEqual(1e-14 * Math.max(1, Math.abs(reference)))
    }
  })

  it('handles special values', () => {
    expect(detLn(1)).toBe(0)
    expect(detLn(0)).toBe(-Infinity)
    expect(detLn(Infinity)).toBe(Infinity)
    expect(detLn(-1)).toBeNaN()
    expect(detLn(Number.NaN)).toBeNaN()
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run --project det-math`
Expected: FAIL — modules `./random`, `./hash`, `./trig`, `./ln` not found.

- [ ] **Step 3: Implement**

`packages/det-math/src/random.ts`:
```ts
import { fnv1a32 } from './hash'

/** Port for seeded randomness (D16). Core code receives a RandomSource; it never calls Math.random. */
export interface RandomSource {
  /** Next uniformly distributed unsigned 32-bit integer. */
  nextUint32(): number
  /** Next float uniformly distributed in [0, 1). */
  nextFloat(): number
}

const UINT32_RANGE = 4_294_967_296
const GOLDEN_GAMMA = 0x9e3779b9
const SPLITMIX_MULTIPLIER_1 = 0x85ebca6b
const SPLITMIX_MULTIPLIER_2 = 0xc2b2ae35
const STATE_WORDS = 4
const WARMUP_ROUNDS = 12

function splitMix32(state: number): { state: number; output: number } {
  const next = (state + GOLDEN_GAMMA) >>> 0
  let z = next
  z = Math.imul(z ^ (z >>> 16), SPLITMIX_MULTIPLIER_1) >>> 0
  z = Math.imul(z ^ (z >>> 13), SPLITMIX_MULTIPLIER_2) >>> 0
  return { state: next, output: (z ^ (z >>> 16)) >>> 0 }
}

/** sfc32 seeded through splitmix32. Integer-only arithmetic: identical on every JS engine. */
export class SeededRandom implements RandomSource {
  private constructor(
    private readonly seed: number,
    private a: number,
    private b: number,
    private c: number,
    private d: number
  ) {}

  static fromSeed(seed: number): SeededRandom {
    let state = seed >>> 0
    const words: number[] = []
    for (let i = 0; i < STATE_WORDS; i++) {
      const step = splitMix32(state)
      state = step.state
      words.push(step.output)
    }
    const [a = 0, b = 0, c = 0, d = 0] = words
    const random = new SeededRandom(seed >>> 0, a, b, c, d)
    for (let i = 0; i < WARMUP_ROUNDS; i++) random.nextUint32()
    return random
  }

  /** Independent, reproducible stream derived from the original seed and a label. Does not consume this stream. */
  fork(label: string): SeededRandom {
    return SeededRandom.fromSeed((this.seed ^ fnv1a32(label)) >>> 0)
  }

  nextUint32(): number {
    const sum = (this.a + this.b) | 0
    this.a = this.b ^ (this.b >>> 9)
    this.b = (this.c + (this.c << 3)) | 0
    this.c = (this.c << 21) | (this.c >>> 11)
    this.d = (this.d + 1) | 0
    const result = (sum + this.d) | 0
    this.c = (this.c + result) | 0
    return result >>> 0
  }

  nextFloat(): number {
    return this.nextUint32() / UINT32_RANGE
  }
}
```

`packages/det-math/src/hash.ts`:
```ts
const FNV32_OFFSET = 0x811c9dc5
const FNV32_PRIME = 0x01000193
const FNV64_OFFSET = 0xcbf29ce484222325n
const FNV64_PRIME = 0x100000001b3n
const UINT64_MASK = 0xffffffffffffffffn
const FLOAT64_BYTES = 8
const UINT32_BYTES = 4
const HEX_DIGITS_64 = 16

/** FNV-1a 32-bit hash of the UTF-16 code units of `text` (low byte only, used for stream labels). */
export function fnv1a32(text: string): number {
  let hash = FNV32_OFFSET
  for (let i = 0; i < text.length; i++) {
    hash ^= text.charCodeAt(i) & 0xff
    hash = Math.imul(hash, FNV32_PRIME) >>> 0
  }
  return hash >>> 0
}

/** Incremental FNV-1a 64-bit hash over little-endian bytes; used for determinism snapshot hashes. */
export class SnapshotHasher {
  private hash = FNV64_OFFSET
  private readonly scratch = new DataView(new ArrayBuffer(FLOAT64_BYTES))

  addFloat64(value: number): this {
    // Canonicalize -0 and NaN so equal physical states hash equally on every engine.
    const canonical = Number.isNaN(value) ? Number.NaN : value === 0 ? 0 : value
    if (Number.isNaN(canonical)) {
      this.scratch.setUint32(0, 0, true)
      this.scratch.setUint32(UINT32_BYTES, 0x7ff80000, true)
    } else {
      this.scratch.setFloat64(0, canonical, true)
    }
    return this.addScratchBytes(FLOAT64_BYTES)
  }

  addUint32(value: number): this {
    this.scratch.setUint32(0, value >>> 0, true)
    return this.addScratchBytes(UINT32_BYTES)
  }

  addString(text: string): this {
    for (let i = 0; i < text.length; i++) {
      const unit = text.charCodeAt(i)
      this.addByte(unit & 0xff)
      this.addByte(unit >>> 8)
    }
    return this
  }

  digest(): string {
    return this.hash.toString(16).padStart(HEX_DIGITS_64, '0')
  }

  private addScratchBytes(count: number): this {
    for (let i = 0; i < count; i++) this.addByte(this.scratch.getUint8(i))
    return this
  }

  private addByte(byte: number): void {
    this.hash ^= BigInt(byte)
    this.hash = (this.hash * FNV64_PRIME) & UINT64_MASK
  }
}
```

`packages/det-math/src/trig.ts`:
```ts
// Deterministic trigonometry: fdlibm kernels (public domain) using only correctly rounded operations (D16).

/** Largest |x| for which the two-term Cody–Waite reduction stays accurate to ~1e-15. */
export const MAX_TRIG_ARGUMENT_RAD = 1e5

const PI = 3.141592653589793
const HALF_PI = 1.5707963267948966
const TWO_OVER_PI = 6.36619772367581382433e-1
const PIO2_HI = 1.57079632673412561417
const PIO2_LO = 6.07710050650619224932e-11

const S1 = -1.66666666666666324348e-1
const S2 = 8.33333333332248946124e-3
const S3 = -1.98412698298579493134e-4
const S4 = 2.75573137070700676789e-6
const S5 = -2.50507602534068634195e-8
const S6 = 1.58969099521155010221e-10

const C1 = 4.16666666666666019037e-2
const C2 = -1.38888888888741095749e-3
const C3 = 2.48015872894767294178e-5
const C4 = -2.75573143513906633035e-7
const C5 = 2.08757232129817482790e-9
const C6 = -1.13596475577881948265e-11

const ATAN_HI = [4.63647609000806093515e-1, 7.85398163397448278999e-1, 9.82793723247329054082e-1, 1.5707963267948965580] as const
const ATAN_LO = [2.26987774529616870924e-17, 3.06161699786838301793e-17, 1.39033110312309984516e-17, 6.12323399573676603587e-17] as const
const AT0 = 3.33333333333329318027e-1
const AT1 = -1.99999999998764832476e-1
const AT2 = 1.42857142725034663711e-1
const AT3 = -1.11111104054623557880e-1
const AT4 = 9.09088713343650656196e-2
const AT5 = -7.69187620504482999495e-2
const AT6 = 6.66107313738753120669e-2
const AT7 = -5.83357013379057348645e-2
const AT8 = 4.97687799461593236017e-2
const AT9 = -3.65315727442169155270e-2
const AT10 = 1.62858201153657823623e-2
const ATAN_HUGE_ARGUMENT = 7.378697629483821e19 // 2^66

type Quadrant = 0 | 1 | 2 | 3
type Reduced = { r: number; quadrant: Quadrant }

function sinKernel(r: number): number {
  const z = r * r
  return r + r * z * (S1 + z * (S2 + z * (S3 + z * (S4 + z * (S5 + z * S6)))))
}

function cosKernel(r: number): number {
  const z = r * r
  return 1 - 0.5 * z + z * z * (C1 + z * (C2 + z * (C3 + z * (C4 + z * (C5 + z * C6)))))
}

function reduce(x: number): Reduced {
  if (!(Math.abs(x) <= MAX_TRIG_ARGUMENT_RAD)) {
    throw new RangeError(`Trig argument ${x} outside ±${MAX_TRIG_ARGUMENT_RAD} rad`)
  }
  const k = Math.round(x * TWO_OVER_PI)
  const r = x - k * PIO2_HI - k * PIO2_LO
  return { r, quadrant: (((k % 4) + 4) % 4) as Quadrant }
}

export function detSin(x: number): number {
  const { r, quadrant } = reduce(x)
  switch (quadrant) {
    case 0:
      return sinKernel(r)
    case 1:
      return cosKernel(r)
    case 2:
      return -sinKernel(r)
    case 3:
      return -cosKernel(r)
  }
}

export function detCos(x: number): number {
  const { r, quadrant } = reduce(x)
  switch (quadrant) {
    case 0:
      return cosKernel(r)
    case 1:
      return -sinKernel(r)
    case 2:
      return -cosKernel(r)
    case 3:
      return sinKernel(r)
  }
}

export function detAtan(x: number): number {
  if (Number.isNaN(x)) return Number.NaN
  const sign = x < 0 ? -1 : 1
  let ax = Math.abs(x)
  if (ax >= ATAN_HUGE_ARGUMENT) return sign * (ATAN_HI[3] + ATAN_LO[3])
  let id: -1 | Quadrant
  if (ax < 0.4375) {
    id = -1
  } else if (ax < 1.1875) {
    if (ax < 0.6875) {
      id = 0
      ax = (2 * ax - 1) / (2 + ax)
    } else {
      id = 1
      ax = (ax - 1) / (ax + 1)
    }
  } else if (ax < 2.4375) {
    id = 2
    ax = (ax - 1.5) / (1 + 1.5 * ax)
  } else {
    id = 3
    ax = -1 / ax
  }
  const z = ax * ax
  const w = z * z
  const s1 = z * (AT0 + w * (AT2 + w * (AT4 + w * (AT6 + w * (AT8 + w * AT10)))))
  const s2 = w * (AT1 + w * (AT3 + w * (AT5 + w * (AT7 + w * AT9))))
  if (id === -1) return sign * (ax - ax * (s1 + s2))
  return sign * (ATAN_HI[id] - (ax * (s1 + s2) - ATAN_LO[id] - ax))
}

/** atan2 in (−π, π]. Signed zeros are not distinguished: atan2(0, 0) = 0. */
export function detAtan2(y: number, x: number): number {
  if (Number.isNaN(x) || Number.isNaN(y)) return Number.NaN
  if (x === 0) {
    if (y === 0) return 0
    return y > 0 ? HALF_PI : -HALF_PI
  }
  const base = detAtan(y / x)
  if (x > 0) return base
  return y >= 0 ? base + PI : base - PI
}
```

`packages/det-math/src/ln.ts`:
```ts
const LN2 = 0.6931471805599453
const SQRT2 = 1.4142135623730951
const SQRT_HALF = 0.7071067811865476
/** Odd terms of 2·atanh(s); |s| ≤ 0.1716 so 14 terms reach below 1e-17. */
const SERIES_TERMS = 14

/** Natural logarithm from exact operations only (D16): x = m·2^k, ln m = 2·atanh((m−1)/(m+1)). */
export function detLn(x: number): number {
  if (Number.isNaN(x) || x < 0) return Number.NaN
  if (x === 0) return -Infinity
  if (x === Infinity) return Infinity
  let mantissa = x
  let exponent = 0
  while (mantissa >= SQRT2) {
    mantissa /= 2
    exponent += 1
  }
  while (mantissa < SQRT_HALF) {
    mantissa *= 2
    exponent -= 1
  }
  const s = (mantissa - 1) / (mantissa + 1)
  const s2 = s * s
  let term = s
  let sum = 0
  for (let n = 1; n <= 2 * SERIES_TERMS; n += 2) {
    sum += term / n
    term *= s2
  }
  return 2 * sum + exponent * LN2
}
```

`packages/det-math/src/angles.ts`:
```ts
export const DEG_TO_RAD = Math.PI / 180
export const RAD_TO_DEG = 180 / Math.PI

export function degToRad(degrees: number): number {
  return degrees * DEG_TO_RAD
}

export function radToDeg(radians: number): number {
  return radians * RAD_TO_DEG
}
```

`packages/det-math/src/index.ts`:
```ts
export { DEG_TO_RAD, RAD_TO_DEG, degToRad, radToDeg } from './angles'
export { SnapshotHasher, fnv1a32 } from './hash'
export { detLn } from './ln'
export { SeededRandom, type RandomSource } from './random'
export { MAX_TRIG_ARGUMENT_RAD, detAtan, detAtan2, detCos, detSin } from './trig'
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm vitest run --project det-math`
Expected: PASS. If a trig accuracy test fails by a few ulps near ±π/4 boundaries, report it to `physics-reviewer` rather than loosening the tolerance.

- [ ] **Step 5: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test
git add packages/det-math
git commit -m "feat(det-math): add seeded prng, deterministic trig, log and hashing"
```

---

### Task 3: `formats` — primitives, world document and measurement schemas

**Files:**
- Create: `packages/formats/src/primitives.ts`, `world.ts`, `measurement.ts`
- Modify: `packages/formats/src/index.ts`
- Test: `packages/formats/src/primitives.test.ts`, `world.test.ts`

**Interfaces:**
- Produces (all types inferred from the Zod schemas):
  - `EntityIdSchema`/`EntityId` (`/^[a-z][a-z0-9-]{0,47}$/`), `EntityRefSchema`/`EntityRef` (`"#id"`), `refToId(ref): EntityId`, `idToRef(id): EntityRef`
  - `Vec3Schema`/`Vec3` (`readonly [number, number, number]`), `GridCoordSchema`/`GridCoord`, `LocalizedTextSchema`/`LocalizedText` (`{ es, en }`), `Uint32Schema`
  - `normalizeStudentId(raw: string): string`, `StudentIdSchema`, `MAX_STUDENT_ID_CHARS = 64`
  - World: `WORLD_FORMAT`, `WORLD_FORMAT_VERSION`, `WORLD_SIZE_CELLS`, `BLOCK_SIZE_M`, `DEFAULT_GRAVITY_MPS2`, `TERRAIN_ENTITY_ID = 'terrain'`, `MaterialIdSchema`/`MaterialId`, `MotionSchema`/`Motion`, `CollisionLayerSchema`/`PartLayer`, `BlockFill`, `Part` (`BeamPart | SpherePart`), `Joint` (`HingeJoint | FixedJoint`), `AngleLatch`, `WorldDocSchema`/`WorldDoc`, `PartSchema`, `JointSchema`, `BlockFillSchema`
  - Measurement: `QuantityIdSchema`/`QuantityId` (`"sensor.reading"`), `splitQuantityId(q): { sensorId: EntityId; reading: string }`, `InstrumentIdSchema`/`InstrumentId`, `UnitSchema`/`Unit`, `MeasurementSchema`/`Measurement` (`{ quantity, instrument, value, uncertainty, unit }`)

- [ ] **Step 1: Write the failing tests**

`packages/formats/src/primitives.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { EntityIdSchema, EntityRefSchema, StudentIdSchema, idToRef, normalizeStudentId, refToId } from './primitives'

describe('entity ids and refs', () => {
  it('accepts lowercase kebab ids', () => {
    expect(EntityIdSchema.safeParse('launch-arm').success).toBe(true)
  })

  it('rejects ids with uppercase, leading digits or over 48 chars', () => {
    for (const bad of ['Arm', '1arm', 'a'.repeat(49), '']) {
      expect(EntityIdSchema.safeParse(bad).success).toBe(false)
    }
  })

  it('converts between refs and ids', () => {
    expect(EntityRefSchema.safeParse('#ball').success).toBe(true)
    expect(refToId('#ball')).toBe('ball')
    expect(idToRef('ball')).toBe('#ball')
  })
})

describe('student ids', () => {
  it('trims and NFC-normalizes', () => {
    expect(normalizeStudentId('  José  ')).toBe('José')
  })

  it('accepts normalized ids of 1 to 64 chars', () => {
    expect(StudentIdSchema.safeParse('José').success).toBe(true)
    expect(StudentIdSchema.safeParse('x'.repeat(64)).success).toBe(true)
  })

  it('rejects empty, too long or non-normalized ids', () => {
    for (const bad of ['', 'x'.repeat(65), ' padded ', 'José']) {
      expect(StudentIdSchema.safeParse(bad).success).toBe(false)
    }
  })
})
```

`packages/formats/src/world.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { WorldDocSchema, type WorldDoc } from './world'

function validWorld(): WorldDoc {
  return {
    format: 'physics-lab/world',
    formatVersion: 1,
    gravityMps2: 9.8,
    blocks: [{ from: [0, 0, 0], to: [127, 1, 127], material: 'stone' }],
    parts: [
      { id: 'post', type: 'beam', motion: 'static', material: 'wood', layer: 'machine', positionM: [10, 1.8, 32], rotationDeg: [0, 0, 0], ccd: false, sizeM: [0.2, 1.6, 0.2] },
      { id: 'arm', type: 'beam', motion: 'dynamic', material: 'wood', layer: 'machine', positionM: [9.434, 2.034, 32], rotationDeg: [0, 0, 225], ccd: false, sizeM: [1.6, 0.1, 0.1] },
      { id: 'ball', type: 'sphere', motion: 'dynamic', material: 'stone', layer: 'projectile', positionM: [8.939, 1.539, 32], rotationDeg: [0, 0, 0], ccd: true, radiusM: 0.1 },
    ],
    joints: [
      { id: 'pivot', type: 'hinge', a: 'post', b: 'arm', anchorM: [10, 2.6, 32], axis: [0, 0, -1], motor: { targetSpeedDegPerS: 450, gain: 10000 } },
      { id: 'hold', type: 'fixed', a: 'arm', b: 'ball' },
    ],
    latches: [{ id: 'latch', type: 'angleLatch', hinge: 'pivot', releasesJoint: 'hold', releaseAtHingeAngleDeg: 90 }],
  }
}

function issuePaths(input: unknown): string[] {
  const result = WorldDocSchema.safeParse(input)
  return result.success ? [] : result.error.issues.map((issue) => issue.path.join('.'))
}

describe('WorldDocSchema', () => {
  it('accepts a valid catapult world', () => {
    expect(WorldDocSchema.safeParse(validWorld()).success).toBe(true)
  })

  it('rejects a fill outside the 128 × 64 × 128 world', () => {
    const world = validWorld()
    expect(issuePaths({ ...world, blocks: [{ from: [0, 0, 0], to: [128, 0, 0], material: 'stone' }] })).toContain('blocks.0.to')
  })

  it('rejects a fill whose "from" exceeds "to"', () => {
    const world = validWorld()
    expect(issuePaths({ ...world, blocks: [{ from: [5, 0, 0], to: [4, 0, 0], material: 'stone' }] })).toContain('blocks.0.from')
  })

  it('rejects duplicate ids across parts, joints and latches', () => {
    const world = validWorld()
    expect(issuePaths({ ...world, latches: [{ ...world.latches[0], id: 'arm' }] })).toContain('latches.0.id')
  })

  it('reserves the terrain id', () => {
    const world = validWorld()
    const [post, ...rest] = world.parts
    expect(issuePaths({ ...world, parts: [{ ...post, id: 'terrain' }, ...rest] })).toContain('parts.0.id')
  })

  it('rejects joints that reference unknown parts', () => {
    const world = validWorld()
    expect(issuePaths({ ...world, joints: [world.joints[0], { id: 'hold', type: 'fixed', a: 'arm', b: 'ghost' }] })).toContain('joints.1.b')
  })

  it('rejects latches that do not reference a hinge and a fixed joint', () => {
    const world = validWorld()
    expect(issuePaths({ ...world, latches: [{ ...world.latches[0], hinge: 'hold', releasesJoint: 'pivot' }] })).toEqual(
      expect.arrayContaining(['latches.0.hinge', 'latches.0.releasesJoint'])
    )
  })

  it('rejects a zero hinge axis', () => {
    const world = validWorld()
    expect(issuePaths({ ...world, joints: [{ ...world.joints[0], axis: [0, 0, 0] }, world.joints[1]] })).toContain('joints.0.axis')
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run --project formats`
Expected: FAIL — `./primitives`, `./world` not found.

- [ ] **Step 3: Implement**

`packages/formats/src/primitives.ts`:
```ts
import { z } from 'zod'

export const ENTITY_ID_PATTERN = /^[a-z][a-z0-9-]{0,47}$/
export const ENTITY_REF_PATTERN = /^#[a-z][a-z0-9-]{0,47}$/
export const MAX_STUDENT_ID_CHARS = 64

export const EntityIdSchema = z.string().regex(ENTITY_ID_PATTERN)
export type EntityId = z.infer<typeof EntityIdSchema>

/** Reference to an entity as written in challenge data and the console: "#ball". */
export const EntityRefSchema = z.string().regex(ENTITY_REF_PATTERN)
export type EntityRef = z.infer<typeof EntityRefSchema>

export function refToId(ref: EntityRef): EntityId {
  return ref.slice(1)
}

export function idToRef(id: EntityId): EntityRef {
  return `#${id}`
}

export const Vec3Schema = z.tuple([z.number(), z.number(), z.number()]).readonly()
export type Vec3 = z.infer<typeof Vec3Schema>

const CellIndexSchema = z.number().int().min(0)
export const GridCoordSchema = z.tuple([CellIndexSchema, CellIndexSchema, CellIndexSchema]).readonly()
export type GridCoord = z.infer<typeof GridCoordSchema>

export const LocalizedTextSchema = z.object({ es: z.string().min(1), en: z.string().min(1) })
export type LocalizedText = z.infer<typeof LocalizedTextSchema>

export const Uint32Schema = z.number().int().min(0).max(0xffff_ffff)

/** Canonical student id: trimmed and Unicode NFC, so the same person always gets the same seed (spec §7.3). */
export function normalizeStudentId(raw: string): string {
  return raw.trim().normalize('NFC')
}

export const StudentIdSchema = z
  .string()
  .min(1)
  .max(MAX_STUDENT_ID_CHARS)
  .refine((id) => id === normalizeStudentId(id), { message: 'Student id must be trimmed and NFC-normalized' })
```

`packages/formats/src/world.ts`:
```ts
import { z } from 'zod'
import { EntityIdSchema, GridCoordSchema, Vec3Schema, type GridCoord } from './primitives'

export const WORLD_FORMAT = 'physics-lab/world'
export const WORLD_FORMAT_VERSION = 1
/** World size in cells (D34). */
export const WORLD_SIZE_CELLS = { x: 128, y: 64, z: 128 } as const
/** Edge of a voxel block in metres (D23, D34). */
export const BLOCK_SIZE_M = 0.5
export const DEFAULT_GRAVITY_MPS2 = 9.8
/** Entity id reserved for the merged static terrain. */
export const TERRAIN_ENTITY_ID = 'terrain'
const MAX_LATCH_ANGLE_DEG = 360

export const MaterialIdSchema = z.enum(['wood', 'stone', 'metal', 'rubber', 'ice'])
export type MaterialId = z.infer<typeof MaterialIdSchema>
export const MotionSchema = z.enum(['static', 'dynamic'])
export type Motion = z.infer<typeof MotionSchema>
export const CollisionLayerSchema = z.enum(['machine', 'projectile'])
export type PartLayer = z.infer<typeof CollisionLayerSchema>

const PositiveVec3Schema = z
  .tuple([z.number().positive(), z.number().positive(), z.number().positive()])
  .readonly()

function isInsideWorld(cell: GridCoord): boolean {
  return cell[0] < WORLD_SIZE_CELLS.x && cell[1] < WORLD_SIZE_CELLS.y && cell[2] < WORLD_SIZE_CELLS.z
}

/** Static blocks filling the inclusive cell box [from, to] (v1 supports static blocks only). */
export const BlockFillSchema = z
  .object({ from: GridCoordSchema, to: GridCoordSchema, material: MaterialIdSchema })
  .refine((fill) => isInsideWorld(fill.from) && isInsideWorld(fill.to), {
    message: 'Cell outside the world',
    path: ['to'],
  })
  .refine((fill) => fill.from.every((value, axis) => value <= (fill.to[axis] ?? -1)), {
    message: '"from" must not exceed "to" on any axis',
    path: ['from'],
  })
export type BlockFill = z.infer<typeof BlockFillSchema>

const partBase = {
  id: EntityIdSchema,
  motion: MotionSchema,
  material: MaterialIdSchema,
  layer: CollisionLayerSchema,
  /** Centre of the part in world metres. */
  positionM: Vec3Schema,
  /** Intrinsic XYZ Euler angles in degrees. */
  rotationDeg: Vec3Schema,
  ccd: z.boolean(),
  initialVelocityMps: Vec3Schema.optional(),
}

export const BeamPartSchema = z.object({ ...partBase, type: z.literal('beam'), sizeM: PositiveVec3Schema })
export const SpherePartSchema = z.object({ ...partBase, type: z.literal('sphere'), radiusM: z.number().positive() })
export const PartSchema = z.discriminatedUnion('type', [BeamPartSchema, SpherePartSchema])
export type BeamPart = z.infer<typeof BeamPartSchema>
export type SpherePart = z.infer<typeof SpherePartSchema>
export type Part = z.infer<typeof PartSchema>

export const HingeMotorSchema = z.object({ targetSpeedDegPerS: z.number(), gain: z.number().positive() })
const NonZeroVec3Schema = Vec3Schema.refine((v) => v[0] !== 0 || v[1] !== 0 || v[2] !== 0, {
  message: 'Axis must not be zero',
})

export const HingeJointSchema = z.object({
  id: EntityIdSchema,
  type: z.literal('hinge'),
  a: EntityIdSchema,
  b: EntityIdSchema,
  anchorM: Vec3Schema,
  axis: NonZeroVec3Schema,
  motor: HingeMotorSchema.nullable(),
})
export const FixedJointSchema = z.object({ id: EntityIdSchema, type: z.literal('fixed'), a: EntityIdSchema, b: EntityIdSchema })
export const JointSchema = z.discriminatedUnion('type', [HingeJointSchema, FixedJointSchema])
export type HingeJoint = z.infer<typeof HingeJointSchema>
export type FixedJoint = z.infer<typeof FixedJointSchema>
export type Joint = z.infer<typeof JointSchema>

/** Removes `releasesJoint` and brakes the hinge motor once the hinge angle reaches the release angle (D44). */
export const AngleLatchSchema = z.object({
  id: EntityIdSchema,
  type: z.literal('angleLatch'),
  hinge: EntityIdSchema,
  releasesJoint: EntityIdSchema,
  releaseAtHingeAngleDeg: z.number().min(-MAX_LATCH_ANGLE_DEG).max(MAX_LATCH_ANGLE_DEG),
})
export type AngleLatch = z.infer<typeof AngleLatchSchema>

const WorldDocShape = z.object({
  format: z.literal(WORLD_FORMAT),
  formatVersion: z.literal(WORLD_FORMAT_VERSION),
  gravityMps2: z.number().positive(),
  blocks: z.array(BlockFillSchema),
  parts: z.array(PartSchema),
  joints: z.array(JointSchema),
  latches: z.array(AngleLatchSchema),
})

function checkReferences(doc: z.infer<typeof WorldDocShape>, ctx: z.RefinementCtx): void {
  const seen = new Set<string>([TERRAIN_ENTITY_ID])
  const claimId = (id: string, path: (string | number)[]): void => {
    if (seen.has(id)) ctx.addIssue({ code: 'custom', message: `Duplicate or reserved id "${id}"`, path })
    seen.add(id)
  }
  doc.parts.forEach((part, index) => claimId(part.id, ['parts', index, 'id']))
  doc.joints.forEach((joint, index) => claimId(joint.id, ['joints', index, 'id']))
  doc.latches.forEach((latch, index) => claimId(latch.id, ['latches', index, 'id']))

  const partIds = new Set(doc.parts.map((part) => part.id))
  doc.joints.forEach((joint, index) => {
    for (const end of ['a', 'b'] as const) {
      if (!partIds.has(joint[end])) {
        ctx.addIssue({ code: 'custom', message: `Unknown part "${joint[end]}"`, path: ['joints', index, end] })
      }
    }
    if (joint.a === joint.b) {
      ctx.addIssue({ code: 'custom', message: 'A joint needs two different parts', path: ['joints', index, 'b'] })
    }
  })

  const jointTypes = new Map(doc.joints.map((joint) => [joint.id, joint.type]))
  doc.latches.forEach((latch, index) => {
    if (jointTypes.get(latch.hinge) !== 'hinge') {
      ctx.addIssue({ code: 'custom', message: 'Must reference a hinge joint', path: ['latches', index, 'hinge'] })
    }
    if (jointTypes.get(latch.releasesJoint) !== 'fixed') {
      ctx.addIssue({ code: 'custom', message: 'Must reference a fixed joint', path: ['latches', index, 'releasesJoint'] })
    }
  })
}

export const WorldDocSchema = WorldDocShape.superRefine(checkReferences)
export type WorldDoc = z.infer<typeof WorldDocSchema>
```

`packages/formats/src/measurement.ts`:
```ts
import { z } from 'zod'
import type { EntityId } from './primitives'

/** "<sensorId>.<reading>", e.g. "flight.range". */
export const QuantityIdSchema = z.string().regex(/^[a-z][a-z0-9-]{0,47}\.[a-z][a-zA-Z0-9]{0,47}$/)
export type QuantityId = z.infer<typeof QuantityIdSchema>

export function splitQuantityId(quantity: QuantityId): { sensorId: EntityId; reading: string } {
  const dot = quantity.indexOf('.')
  return { sensorId: quantity.slice(0, dot), reading: quantity.slice(dot + 1) }
}

export const InstrumentIdSchema = z.enum(['ruler', 'stopwatch', 'photogate', 'protractor', 'forceProbe', 'massScale'])
export type InstrumentId = z.infer<typeof InstrumentIdSchema>

export const UnitSchema = z.enum(['m', 's', 'm/s', 'deg', 'N', 'kg'])
export type Unit = z.infer<typeof UnitSchema>

/** A reading `value ± uncertainty` (standard uncertainty) with unit and instrument (spec §6). */
export const MeasurementSchema = z.object({
  quantity: QuantityIdSchema,
  instrument: InstrumentIdSchema,
  value: z.number(),
  uncertainty: z.number().min(0),
  unit: UnitSchema,
})
export type Measurement = z.infer<typeof MeasurementSchema>
```

`packages/formats/src/index.ts`:
```ts
export * from './measurement'
export * from './primitives'
export * from './world'
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm vitest run --project formats`
Expected: PASS. (If a `validWorld()` literal fails type inference because tuples widen to `number[]`, annotate the helper's return type as `WorldDoc` — already done — and add `as const` to the inner tuple literals only if TypeScript still complains.)

- [ ] **Step 5: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test
git add packages/formats
git commit -m "feat(formats): add world document and measurement schemas"
```

---
### Task 4: `formats` — commands, challenge, result and the versioned loader

**Files:**
- Create: `packages/formats/src/commands.ts`, `challenge.ts`, `result.ts`, `load.ts`
- Create: `packages/formats/fixtures/world.v1.json`, `challenge.v1.json`, `result.v1.json`
- Modify: `packages/formats/src/index.ts`
- Test: `packages/formats/src/commands.test.ts`, `challenge.test.ts`, `load.test.ts`

**Interfaces:**
- Consumes: Task 3 schemas.
- Produces:
  - `CommandSchema`/`Command` — discriminated union on `type`: `placeBlock { cell, material }`, `fill { from, to, material }`, `addPart { part }`, `connect { joint }`, `setProperty { target: EntityRef, property: string, value: number }`, `setPhysics { gravityMps2 }`, `trigger { target: EntityRef, action: 'startMotor' }`, `step { count }`, `reset {}`, `measure { quantity: QuantityId }`; `CommandLogEntrySchema`/`CommandLogEntry` = `{ atStep: number; command: Command }` (D44); `MAX_STEP_COMMAND_COUNT`
  - `CHALLENGE_FORMAT`, `CHALLENGE_FORMAT_VERSION`, `ChallengeSchema`/`Challenge`, `ParameterDef`, `SensorDef` (`{ id, type: 'projectileFlight', body: EntityRef, latch: EntityRef, gateWidthM }`), `Task` (`{ id, prompt, quantity, unit, tolerance: { kind: 'combined-uncertainty', coverage } }`), `InstrumentModeSchema`/`InstrumentMode` (`'ideal' | 'realistic'`), `parseBindPath(bind): { entityId, property }`
  - `RESULT_FORMAT`, `RESULT_FORMAT_VERSION`, `ResultSchema`/`Result`, `Prediction`, `Explanation`, `RecordedMeasurement` (`Measurement & { logIndex }`), `LearningEvent`, `LEARNING_EVENT_TYPES`, `MAX_EXPLANATION_CHARS = 2000`, `MAX_COMMAND_LOG_ENTRIES = 10000`
  - `MAX_DOCUMENT_CHARS`, `LoadError`, `LoadOutcome<T>`, `DocumentIssue`, `loadWorld(text)`, `loadChallenge(text)`, `loadResult(text)`, `loadVersioned(text, spec)`, `VersionedFormat<T>`

- [ ] **Step 1: Write the golden fixtures**

`packages/formats/fixtures/world.v1.json`:
```json
{
  "format": "physics-lab/world",
  "formatVersion": 1,
  "gravityMps2": 9.8,
  "blocks": [{ "from": [0, 0, 0], "to": [127, 1, 127], "material": "stone" }],
  "parts": [
    {
      "id": "ball",
      "type": "sphere",
      "motion": "dynamic",
      "material": "stone",
      "layer": "projectile",
      "positionM": [8, 1.1, 32],
      "rotationDeg": [0, 0, 0],
      "ccd": true,
      "radiusM": 0.1,
      "initialVelocityMps": [7.0710678118654755, 7.0710678118654755, 0]
    }
  ],
  "joints": [],
  "latches": []
}
```

`packages/formats/fixtures/challenge.v1.json`:
```json
{
  "format": "physics-lab/challenge",
  "formatVersion": 1,
  "id": "fixture-catapult",
  "version": 1,
  "title": { "es": "Catapulta de prueba", "en": "Fixture catapult" },
  "topic": "projectile-motion",
  "scenario": {
    "world": {
      "format": "physics-lab/world",
      "formatVersion": 1,
      "gravityMps2": 9.8,
      "blocks": [{ "from": [0, 0, 0], "to": [127, 1, 127], "material": "stone" }],
      "parts": [
        { "id": "post", "type": "beam", "motion": "static", "material": "wood", "layer": "machine", "positionM": [10, 1.8, 32], "rotationDeg": [0, 0, 0], "ccd": false, "sizeM": [0.2, 1.6, 0.2] },
        { "id": "arm", "type": "beam", "motion": "dynamic", "material": "wood", "layer": "machine", "positionM": [9.434, 2.034, 32], "rotationDeg": [0, 0, 225], "ccd": false, "sizeM": [1.6, 0.1, 0.1] },
        { "id": "ball", "type": "sphere", "motion": "dynamic", "material": "stone", "layer": "projectile", "positionM": [8.939, 1.539, 32], "rotationDeg": [0, 0, 0], "ccd": true, "radiusM": 0.1 }
      ],
      "joints": [
        { "id": "pivot", "type": "hinge", "a": "post", "b": "arm", "anchorM": [10, 2.6, 32], "axis": [0, 0, -1], "motor": { "targetSpeedDegPerS": 450, "gain": 10000 } },
        { "id": "hold", "type": "fixed", "a": "arm", "b": "ball" }
      ],
      "latches": [{ "id": "latch", "type": "angleLatch", "hinge": "pivot", "releasesJoint": "hold", "releaseAtHingeAngleDeg": 90 }]
    },
    "locked": ["#post", "#arm", "#ball", "#pivot", "#hold", "#latch"]
  },
  "parameters": [
    { "name": "releaseHingeAngle", "bind": "#latch.releaseAtHingeAngleDeg", "unit": "deg", "range": [75, 105], "distribution": "uniform", "step": 1 }
  ],
  "sensors": [{ "id": "flight", "type": "projectileFlight", "body": "#ball", "latch": "#latch", "gateWidthM": 0.2 }],
  "launch": { "target": "#pivot", "action": "startMotor" },
  "calibration": {
    "pauseOnLatch": "#latch",
    "measurable": ["flight.releaseSpeed", "flight.releaseAngle", "flight.releaseHeight"]
  },
  "tasks": [
    {
      "id": "predict-range",
      "prompt": { "es": "Predice el alcance.", "en": "Predict the range." },
      "quantity": "flight.range",
      "unit": "m",
      "tolerance": { "kind": "combined-uncertainty", "coverage": 2 }
    }
  ],
  "solverId": "projectile.range.v1",
  "solverInputs": { "speed": "flight.releaseSpeed", "angle": "flight.releaseAngle", "height": "flight.releaseHeight" },
  "instrumentMode": "realistic",
  "allowedCommands": ["help", "measure", "reset", "time"]
}
```

`packages/formats/fixtures/result.v1.json`:
```json
{
  "format": "physics-lab/result",
  "formatVersion": 1,
  "appVersion": "0.1.0",
  "engineVersion": "@dimforge/rapier3d-deterministic-compat@0.21.0",
  "formatVersions": { "world": 1, "challenge": 1, "result": 1 },
  "challenge": { "id": "fixture-catapult", "version": 1 },
  "seed": 12345,
  "studentId": "José",
  "parameters": { "releaseHingeAngle": 90 },
  "predictions": [{ "taskId": "predict-range", "value": 15.2, "unit": "m", "submittedAtMs": 61000 }],
  "explanations": [{ "taskId": "predict-range", "text": "Air drag is ignored." }],
  "commandLog": [
    { "atStep": 0, "command": { "type": "trigger", "target": "#pivot", "action": "startMotor" } },
    { "atStep": 48, "command": { "type": "measure", "quantity": "flight.releaseSpeed" } }
  ],
  "measurements": [
    { "quantity": "flight.releaseSpeed", "instrument": "photogate", "value": 11.764705882352942, "uncertainty": 0.4, "unit": "m/s", "logIndex": 1 }
  ],
  "learningEvents": [
    { "type": "launch", "atMs": 1200 },
    { "type": "predictionSubmitted", "atMs": 61000, "taskId": "predict-range" }
  ]
}
```

- [ ] **Step 2: Write the failing tests**

`packages/formats/src/commands.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { CommandLogEntrySchema, CommandSchema } from './commands'

describe('CommandSchema', () => {
  it('accepts each command type', () => {
    const commands = [
      { type: 'placeBlock', cell: [1, 2, 3], material: 'wood' },
      { type: 'fill', from: [0, 0, 0], to: [3, 0, 3], material: 'stone' },
      { type: 'setProperty', target: '#latch', property: 'releaseAtHingeAngleDeg', value: 80 },
      { type: 'setPhysics', gravityMps2: 9.8 },
      { type: 'trigger', target: '#pivot', action: 'startMotor' },
      { type: 'step', count: 240 },
      { type: 'reset' },
      { type: 'measure', quantity: 'flight.range' },
    ]
    for (const command of commands) expect(CommandSchema.safeParse(command).success).toBe(true)
  })

  it('rejects unknown command types and malformed quantities', () => {
    expect(CommandSchema.safeParse({ type: 'teleport' }).success).toBe(false)
    expect(CommandSchema.safeParse({ type: 'measure', quantity: 'range' }).success).toBe(false)
  })

  it('requires a non-negative integer atStep in log entries', () => {
    expect(CommandLogEntrySchema.safeParse({ atStep: -1, command: { type: 'reset' } }).success).toBe(false)
    expect(CommandLogEntrySchema.safeParse({ atStep: 1.5, command: { type: 'reset' } }).success).toBe(false)
    expect(CommandLogEntrySchema.safeParse({ atStep: 0, command: { type: 'reset' } }).success).toBe(true)
  })
})
```

`packages/formats/src/challenge.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import fixture from '../fixtures/challenge.v1.json'
import { ChallengeSchema, parseBindPath } from './challenge'

function issuePaths(input: unknown): string[] {
  const result = ChallengeSchema.safeParse(input)
  return result.success ? [] : result.error.issues.map((issue) => issue.path.join('.'))
}

describe('ChallengeSchema', () => {
  it('accepts the golden fixture', () => {
    expect(ChallengeSchema.safeParse(fixture).success).toBe(true)
  })

  it('rejects a parameter bound to an unknown entity', () => {
    const parameters = [{ ...fixture.parameters[0], bind: '#ghost.releaseAtHingeAngleDeg' }]
    expect(issuePaths({ ...fixture, parameters })).toContain('parameters.0.bind')
  })

  it('rejects an empty or inverted parameter range', () => {
    const parameters = [{ ...fixture.parameters[0], range: [105, 75] }]
    expect(issuePaths({ ...fixture, parameters })).toContain('parameters.0.range')
  })

  it('rejects a task quantity whose sensor does not exist', () => {
    const tasks = [{ ...fixture.tasks[0], quantity: 'ghost.range' }]
    expect(issuePaths({ ...fixture, tasks })).toContain('tasks.0.quantity')
  })

  it('rejects a sensor body that is not in the scenario', () => {
    const sensors = [{ ...fixture.sensors[0], body: '#ghost' }]
    expect(issuePaths({ ...fixture, sensors })).toContain('sensors.0.body')
  })

  it('parses bind paths', () => {
    expect(parseBindPath('#latch.releaseAtHingeAngleDeg')).toEqual({ entityId: 'latch', property: 'releaseAtHingeAngleDeg' })
  })
})
```

`packages/formats/src/load.test.ts`:
```ts
import { readFileSync } from 'node:fs'
import { describe, expect, it } from 'vitest'
import { z } from 'zod'
import { MAX_DOCUMENT_CHARS, loadChallenge, loadResult, loadVersioned, loadWorld } from './load'

const fixtureText = (name: string): string =>
  readFileSync(new URL(`../fixtures/${name}`, import.meta.url), 'utf8')

describe('golden fixtures (one per format version)', () => {
  it.each([
    ['world.v1.json', loadWorld],
    ['challenge.v1.json', loadChallenge],
    ['result.v1.json', loadResult],
  ] as const)('%s loads unchanged', (name, load) => {
    const text = fixtureText(name)
    const outcome = load(text)
    expect(outcome.ok).toBe(true)
    if (outcome.ok) {
      expect(outcome.migratedFrom).toBeNull()
      expect(outcome.value).toEqual(JSON.parse(text))
    }
  })
})

describe('loader errors (spec §7.6)', () => {
  it('rejects documents above the size cap before parsing', () => {
    const outcome = loadWorld(' '.repeat(MAX_DOCUMENT_CHARS + 1))
    expect(outcome).toEqual({ ok: false, error: { kind: 'tooLarge', chars: MAX_DOCUMENT_CHARS + 1, limit: MAX_DOCUMENT_CHARS } })
  })

  it('rejects invalid JSON', () => {
    expect(loadWorld('{ nope')).toEqual({ ok: false, error: { kind: 'invalidJson' } })
  })

  it('rejects another format', () => {
    expect(loadWorld(fixtureText('result.v1.json'))).toEqual({
      ok: false,
      error: { kind: 'wrongFormat', expected: 'physics-lab/world' },
    })
  })

  it('refuses a newer format version', () => {
    const newer = { ...JSON.parse(fixtureText('world.v1.json')), formatVersion: 2 }
    expect(loadWorld(JSON.stringify(newer))).toEqual({ ok: false, error: { kind: 'newerVersion', found: 2, supported: 1 } })
  })

  it('reports schema issues with their path and applies nothing', () => {
    const broken = JSON.parse(fixtureText('world.v1.json'))
    broken.parts[0].id = 'Bad Id'
    const outcome = loadWorld(JSON.stringify(broken))
    expect(outcome.ok).toBe(false)
    if (!outcome.ok && outcome.error.kind === 'invalid') {
      expect(outcome.error.issues.map((issue) => issue.path)).toContain('parts.0.id')
    }
  })

  it('rejects a tampered result with an atStep that is not an integer', () => {
    const tampered = JSON.parse(fixtureText('result.v1.json'))
    tampered.commandLog[1].atStep = 'later'
    expect(loadResult(JSON.stringify(tampered)).ok).toBe(false)
  })
})

describe('migration chain', () => {
  const V2 = z.object({ format: z.literal('demo'), formatVersion: z.literal(2), name: z.string() })

  it('migrates older documents step by step and reports the source version', () => {
    const outcome = loadVersioned('{"format":"demo","formatVersion":1,"title":"x"}', {
      format: 'demo',
      currentVersion: 2,
      migrations: new Map([[1, (doc: Record<string, unknown>) => ({ format: 'demo', formatVersion: 2, name: doc['title'] })]]),
      schema: V2,
    })
    expect(outcome).toEqual({ ok: true, value: { format: 'demo', formatVersion: 2, name: 'x' }, migratedFrom: 1 })
  })

  it('fails cleanly when a migration step is missing', () => {
    const outcome = loadVersioned('{"format":"demo","formatVersion":1}', {
      format: 'demo',
      currentVersion: 2,
      migrations: new Map(),
      schema: V2,
    })
    expect(outcome.ok).toBe(false)
  })
})
```

The `load.test.ts` file uses `node:fs`; add `"types": ["node"]` **only for test files** by creating `packages/formats/tsconfig.json` as:
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": { "types": ["node"] },
  "include": ["src", "fixtures", "vitest.config.ts"]
}
```
(Node types add no DOM APIs; the lint rules still forbid browser globals and the code under `src/` that is not a test never imports `node:*` — reviewers check this.)

- [ ] **Step 3: Run the tests to verify they fail**

Run: `pnpm vitest run --project formats`
Expected: FAIL — `./commands`, `./challenge`, `./load` not found.

- [ ] **Step 4: Implement**

`packages/formats/src/commands.ts`:
```ts
import { z } from 'zod'
import { QuantityIdSchema } from './measurement'
import { EntityRefSchema, GridCoordSchema } from './primitives'
import { JointSchema, MaterialIdSchema, PartSchema } from './world'

/** Upper bound for one `step` command: one minute of simulated time. */
export const MAX_STEP_COMMAND_COUNT = 240 * 60

export const CommandSchema = z.discriminatedUnion('type', [
  z.object({ type: z.literal('placeBlock'), cell: GridCoordSchema, material: MaterialIdSchema }),
  z.object({ type: z.literal('fill'), from: GridCoordSchema, to: GridCoordSchema, material: MaterialIdSchema }),
  z.object({ type: z.literal('addPart'), part: PartSchema }),
  z.object({ type: z.literal('connect'), joint: JointSchema }),
  z.object({
    type: z.literal('setProperty'),
    target: EntityRefSchema,
    property: z.string().regex(/^[a-zA-Z][a-zA-Z0-9]{0,63}$/),
    value: z.number(),
  }),
  z.object({ type: z.literal('setPhysics'), gravityMps2: z.number().positive() }),
  z.object({ type: z.literal('trigger'), target: EntityRefSchema, action: z.enum(['startMotor']) }),
  z.object({ type: z.literal('step'), count: z.number().int().min(1).max(MAX_STEP_COMMAND_COUNT) }),
  z.object({ type: z.literal('reset') }),
  z.object({ type: z.literal('measure'), quantity: QuantityIdSchema }),
])
export type Command = z.infer<typeof CommandSchema>

/** A command and the run-local step index at which it was applied (D44). */
export const CommandLogEntrySchema = z.object({ atStep: z.number().int().min(0), command: CommandSchema })
export type CommandLogEntry = z.infer<typeof CommandLogEntrySchema>
```

`packages/formats/src/challenge.ts`:
```ts
import { z } from 'zod'
import { QuantityIdSchema, UnitSchema, splitQuantityId } from './measurement'
import { EntityIdSchema, EntityRefSchema, LocalizedTextSchema, refToId, type EntityId } from './primitives'
import { WorldDocSchema } from './world'

export const CHALLENGE_FORMAT = 'physics-lab/challenge'
export const CHALLENGE_FORMAT_VERSION = 1
const BIND_PATH_PATTERN = /^#([a-z][a-z0-9-]{0,47})\.([a-zA-Z][a-zA-Z0-9]{0,63})$/

export function parseBindPath(bind: string): { entityId: EntityId; property: string } {
  const match = BIND_PATH_PATTERN.exec(bind)
  if (!match?.[1] || !match[2]) throw new Error(`Invalid bind path "${bind}"`)
  return { entityId: match[1], property: match[2] }
}

export const InstrumentModeSchema = z.enum(['ideal', 'realistic'])
export type InstrumentMode = z.infer<typeof InstrumentModeSchema>

export const ParameterDefSchema = z.object({
  name: z.string().regex(/^[a-z][a-zA-Z0-9]{0,47}$/),
  bind: z.string().regex(BIND_PATH_PATTERN),
  unit: UnitSchema,
  range: z
    .tuple([z.number(), z.number()])
    .readonly()
    .refine(([min, max]) => min < max, { message: 'Range minimum must be below maximum' }),
  distribution: z.literal('uniform'),
  step: z.number().positive(),
})
export type ParameterDef = z.infer<typeof ParameterDefSchema>

export const SensorDefSchema = z.discriminatedUnion('type', [
  z.object({
    id: EntityIdSchema,
    type: z.literal('projectileFlight'),
    body: EntityRefSchema,
    latch: EntityRefSchema,
    /** Width of the photogate beam crossed at release, in metres (D44). */
    gateWidthM: z.number().positive(),
  }),
])
export type SensorDef = z.infer<typeof SensorDefSchema>

export const TaskSchema = z.object({
  id: EntityIdSchema,
  prompt: LocalizedTextSchema,
  quantity: QuantityIdSchema,
  unit: UnitSchema,
  tolerance: z.object({ kind: z.literal('combined-uncertainty'), coverage: z.number().positive() }),
})
export type Task = z.infer<typeof TaskSchema>

const ChallengeShape = z.object({
  format: z.literal(CHALLENGE_FORMAT),
  formatVersion: z.literal(CHALLENGE_FORMAT_VERSION),
  id: EntityIdSchema,
  version: z.number().int().min(1),
  title: LocalizedTextSchema,
  topic: z.string().min(1),
  scenario: z.object({ world: WorldDocSchema, locked: z.array(EntityRefSchema) }),
  parameters: z.array(ParameterDefSchema),
  sensors: z.array(SensorDefSchema).min(1),
  launch: z.object({ target: EntityRefSchema, action: z.enum(['startMotor']) }),
  calibration: z.object({ pauseOnLatch: EntityRefSchema, measurable: z.array(QuantityIdSchema).min(1) }).nullable(),
  tasks: z.array(TaskSchema).min(1),
  solverId: z.string().min(1),
  solverInputs: z.record(z.string(), QuantityIdSchema),
  instrumentMode: InstrumentModeSchema,
  allowedCommands: z.array(z.string().regex(/^[a-z]+$/)),
})

function checkChallengeReferences(challenge: z.infer<typeof ChallengeShape>, ctx: z.RefinementCtx): void {
  const world = challenge.scenario.world
  const entityIds = new Set<string>([
    ...world.parts.map((part) => part.id),
    ...world.joints.map((joint) => joint.id),
    ...world.latches.map((latch) => latch.id),
  ])
  const sensorIds = new Set(challenge.sensors.map((sensor) => sensor.id))
  const requireEntity = (ref: string, path: (string | number)[]): void => {
    if (!entityIds.has(refToId(ref))) ctx.addIssue({ code: 'custom', message: `Unknown entity "${ref}"`, path })
  }
  const requireSensor = (quantity: string, path: (string | number)[]): void => {
    if (!sensorIds.has(splitQuantityId(quantity).sensorId)) {
      ctx.addIssue({ code: 'custom', message: `Unknown sensor in "${quantity}"`, path })
    }
  }
  challenge.scenario.locked.forEach((ref, index) => requireEntity(ref, ['scenario', 'locked', index]))
  challenge.parameters.forEach((parameter, index) => {
    requireEntity(`#${parseBindPath(parameter.bind).entityId}`, ['parameters', index, 'bind'])
  })
  challenge.sensors.forEach((sensor, index) => {
    requireEntity(sensor.body, ['sensors', index, 'body'])
    requireEntity(sensor.latch, ['sensors', index, 'latch'])
  })
  requireEntity(challenge.launch.target, ['launch', 'target'])
  if (challenge.calibration) {
    requireEntity(challenge.calibration.pauseOnLatch, ['calibration', 'pauseOnLatch'])
    challenge.calibration.measurable.forEach((quantity, index) => requireSensor(quantity, ['calibration', 'measurable', index]))
  }
  challenge.tasks.forEach((task, index) => requireSensor(task.quantity, ['tasks', index, 'quantity']))
  for (const [name, quantity] of Object.entries(challenge.solverInputs)) requireSensor(quantity, ['solverInputs', name])
}

export const ChallengeSchema = ChallengeShape.superRefine(checkChallengeReferences)
export type Challenge = z.infer<typeof ChallengeSchema>
```

`packages/formats/src/result.ts`:
```ts
import { z } from 'zod'
import { CommandLogEntrySchema } from './commands'
import { MeasurementSchema, UnitSchema } from './measurement'
import { EntityIdSchema, StudentIdSchema, Uint32Schema } from './primitives'

export const RESULT_FORMAT = 'physics-lab/result'
export const RESULT_FORMAT_VERSION = 1
export const MAX_EXPLANATION_CHARS = 2000
export const MAX_COMMAND_LOG_ENTRIES = 10_000
export const LEARNING_EVENT_TYPES = [
  'launch',
  'reset',
  'measure',
  'predictionSubmitted',
  'predictionRevised',
  'comparison',
  'explanationSubmitted',
] as const

export const PredictionSchema = z.object({
  taskId: EntityIdSchema,
  value: z.number(),
  unit: UnitSchema,
  submittedAtMs: z.number().min(0),
})
export type Prediction = z.infer<typeof PredictionSchema>

export const ExplanationSchema = z.object({ taskId: EntityIdSchema, text: z.string().max(MAX_EXPLANATION_CHARS) })
export type Explanation = z.infer<typeof ExplanationSchema>

/** A measurement plus the index of the `measure` command that produced it in the command log. */
export const RecordedMeasurementSchema = MeasurementSchema.extend({ logIndex: z.number().int().min(0) })
export type RecordedMeasurement = z.infer<typeof RecordedMeasurementSchema>

export const LearningEventSchema = z.object({
  type: z.enum(LEARNING_EVENT_TYPES),
  /** Milliseconds since session start (spec §7.4). */
  atMs: z.number().min(0),
  taskId: EntityIdSchema.optional(),
})
export type LearningEvent = z.infer<typeof LearningEventSchema>

export const ResultSchema = z.object({
  format: z.literal(RESULT_FORMAT),
  formatVersion: z.literal(RESULT_FORMAT_VERSION),
  appVersion: z.string().min(1).max(64),
  engineVersion: z.string().min(1).max(128),
  formatVersions: z.object({ world: z.number().int(), challenge: z.number().int(), result: z.number().int() }),
  challenge: z.object({ id: EntityIdSchema, version: z.number().int().min(1) }),
  seed: Uint32Schema,
  studentId: StudentIdSchema,
  parameters: z.record(z.string(), z.number()),
  predictions: z.array(PredictionSchema),
  explanations: z.array(ExplanationSchema),
  commandLog: z.array(CommandLogEntrySchema).max(MAX_COMMAND_LOG_ENTRIES),
  measurements: z.array(RecordedMeasurementSchema),
  learningEvents: z.array(LearningEventSchema),
})
export type Result = z.infer<typeof ResultSchema>
```

`packages/formats/src/load.ts`:
```ts
import type { z } from 'zod'
import { CHALLENGE_FORMAT, CHALLENGE_FORMAT_VERSION, ChallengeSchema, type Challenge } from './challenge'
import { RESULT_FORMAT, RESULT_FORMAT_VERSION, ResultSchema, type Result } from './result'
import { WORLD_FORMAT, WORLD_FORMAT_VERSION, WorldDocSchema, type WorldDoc } from './world'

/** Untrusted documents above this size are refused before parsing (D41). */
export const MAX_DOCUMENT_CHARS = 2_000_000

export type DocumentIssue = { path: string; message: string }
export type LoadError =
  | { kind: 'tooLarge'; chars: number; limit: number }
  | { kind: 'invalidJson' }
  | { kind: 'wrongFormat'; expected: string }
  | { kind: 'newerVersion'; found: number; supported: number }
  | { kind: 'invalid'; issues: DocumentIssue[] }
export type LoadOutcome<T> = { ok: true; value: T; migratedFrom: number | null } | { ok: false; error: LoadError }

type JsonRecord = Record<string, unknown>
type Migration = (doc: JsonRecord) => JsonRecord
export type VersionedFormat<T> = {
  format: string
  currentVersion: number
  /** Migration from version `key` to `key + 1`. */
  migrations: ReadonlyMap<number, Migration>
  schema: z.ZodType<T>
}

function fail<T>(error: LoadError): LoadOutcome<T> {
  return { ok: false, error }
}

function isRecord(value: unknown): value is JsonRecord {
  return typeof value === 'object' && value !== null && !Array.isArray(value)
}

function parseJson(text: string): { ok: true; value: unknown } | { ok: false } {
  try {
    return { ok: true, value: JSON.parse(text) }
  } catch {
    return { ok: false }
  }
}

export function loadVersioned<T>(text: string, spec: VersionedFormat<T>): LoadOutcome<T> {
  if (text.length > MAX_DOCUMENT_CHARS) return fail({ kind: 'tooLarge', chars: text.length, limit: MAX_DOCUMENT_CHARS })
  const parsed = parseJson(text)
  if (!parsed.ok) return fail({ kind: 'invalidJson' })
  const document = parsed.value
  if (!isRecord(document) || document['format'] !== spec.format) return fail({ kind: 'wrongFormat', expected: spec.format })
  const version = document['formatVersion']
  if (typeof version !== 'number' || !Number.isInteger(version) || version < 1) {
    return fail({ kind: 'invalid', issues: [{ path: 'formatVersion', message: 'Must be a positive integer' }] })
  }
  if (version > spec.currentVersion) return fail({ kind: 'newerVersion', found: version, supported: spec.currentVersion })

  let migrated = document
  for (let from = version; from < spec.currentVersion; from++) {
    const migration = spec.migrations.get(from)
    if (!migration) {
      return fail({ kind: 'invalid', issues: [{ path: 'formatVersion', message: `No migration from version ${from}` }] })
    }
    migrated = migration(migrated)
  }

  const result = spec.schema.safeParse(migrated)
  if (!result.success) {
    return fail({
      kind: 'invalid',
      issues: result.error.issues.map((issue) => ({ path: issue.path.map(String).join('.'), message: issue.message })),
    })
  }
  return { ok: true, value: result.data, migratedFrom: version === spec.currentVersion ? null : version }
}

const NO_MIGRATIONS: ReadonlyMap<number, Migration> = new Map()

export function loadWorld(text: string): LoadOutcome<WorldDoc> {
  return loadVersioned(text, { format: WORLD_FORMAT, currentVersion: WORLD_FORMAT_VERSION, migrations: NO_MIGRATIONS, schema: WorldDocSchema })
}

export function loadChallenge(text: string): LoadOutcome<Challenge> {
  return loadVersioned(text, { format: CHALLENGE_FORMAT, currentVersion: CHALLENGE_FORMAT_VERSION, migrations: NO_MIGRATIONS, schema: ChallengeSchema })
}

export function loadResult(text: string): LoadOutcome<Result> {
  return loadVersioned(text, { format: RESULT_FORMAT, currentVersion: RESULT_FORMAT_VERSION, migrations: NO_MIGRATIONS, schema: ResultSchema })
}
```

`packages/formats/src/index.ts`:
```ts
export * from './challenge'
export * from './commands'
export * from './load'
export * from './measurement'
export * from './primitives'
export * from './result'
export * from './world'
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `pnpm vitest run --project formats`
Expected: PASS.

- [ ] **Step 6: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test
git add packages/formats
git commit -m "feat(formats): add command, challenge and result schemas with versioned loader"
```

---

### Task 5: `sim-core` — geometry, materials, voxel grid and static-block merging

**Files:**
- Create: `packages/sim-core/src/geometry/vec.ts`, `geometry/quat.ts`, `materials.ts`, `voxel/voxel-grid.ts`, `voxel/static-merge.ts`
- Modify: `packages/sim-core/src/index.ts`
- Test: `packages/sim-core/src/geometry/quat.test.ts`, `voxel/voxel-grid.test.ts`, `voxel/static-merge.test.ts`

**Interfaces:**
- Consumes: `@physics-lab/formats` (`Vec3`, `GridCoord`, `MaterialId`, `MaterialIdSchema`, `BlockFill`, `WORLD_SIZE_CELLS`, `BLOCK_SIZE_M`), `@physics-lab/det-math` (`detSin`, `detCos`, `detAtan2`, `degToRad`, `radToDeg`).
- Produces:
  - `addVec3`, `subVec3`, `scaleVec3`, `dotVec3`, `lengthVec3`, `horizontalDistanceM(a, b)`
  - `type Quat = readonly [x, y, z, w]`, `IDENTITY_QUAT`, `multiplyQuat`, `conjugateQuat`, `rotateVec3(q, v)`, `quatFromEulerDeg(eulerDeg: Vec3)` (intrinsic XYZ, same convention as three.js `'XYZ'`), `angleAboutAxisDeg(q, unitAxis)` in (−180, 180], `normalizeVec3`
  - `type MaterialProps = { densityKgPerM3: number; friction: number; restitution: number }`, `MATERIALS: Readonly<Record<MaterialId, MaterialProps>>`
  - `class VoxelGrid { static fromFills(fills): VoxelGrid; fill(from, to, material): void; materialAt(x, y, z): MaterialId | null; topSurfaceYM(xM, zM): number | null; readonly size }`
  - `CHUNK_SIZE_CELLS = 16`, `type CellBox = { min: GridCoord; max: GridCoord; material: MaterialId }` (inclusive), `mergeStaticBlocks(grid): CellBox[]`, `cellBoxToWorldBox(box): { centerM: Vec3; halfExtentsM: Vec3 }`

- [ ] **Step 1: Write the failing tests**

`packages/sim-core/src/geometry/quat.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { angleAboutAxisDeg, conjugateQuat, multiplyQuat, quatFromEulerDeg, rotateVec3 } from './quat'

function expectVecClose(actual: readonly number[], expected: readonly number[]): void {
  actual.forEach((value, i) => expect(value).toBeCloseTo(expected[i] ?? Number.NaN, 12))
}

describe('quaternions', () => {
  it('rotates +x by 90° about z to +y', () => {
    expectVecClose(rotateVec3(quatFromEulerDeg([0, 0, 90]), [1, 0, 0]), [0, 1, 0])
  })

  it('composes XYZ intrinsic rotations like three.js', () => {
    const q = quatFromEulerDeg([90, 0, 90])
    expectVecClose(rotateVec3(q, [1, 0, 0]), [0, 0, 1])
  })

  it('undoes a rotation with its conjugate', () => {
    const q = quatFromEulerDeg([10, 20, 30])
    expectVecClose(rotateVec3(multiplyQuat(conjugateQuat(q), q), [1, 2, 3]), [1, 2, 3])
  })

  it('measures the signed angle about an axis', () => {
    expect(angleAboutAxisDeg(quatFromEulerDeg([0, 0, 30]), [0, 0, 1])).toBeCloseTo(30, 10)
    expect(angleAboutAxisDeg(quatFromEulerDeg([0, 0, 30]), [0, 0, -1])).toBeCloseTo(-30, 10)
    expect(angleAboutAxisDeg(quatFromEulerDeg([0, 0, 200]), [0, 0, 1])).toBeCloseTo(-160, 10)
  })
})
```

`packages/sim-core/src/voxel/voxel-grid.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { VoxelGrid } from './voxel-grid'

describe('VoxelGrid', () => {
  it('stores fills with later fills overriding earlier ones', () => {
    const grid = VoxelGrid.fromFills([
      { from: [0, 0, 0], to: [3, 0, 3], material: 'stone' },
      { from: [1, 0, 1], to: [1, 0, 1], material: 'wood' },
    ])
    expect(grid.materialAt(0, 0, 0)).toBe('stone')
    expect(grid.materialAt(1, 0, 1)).toBe('wood')
    expect(grid.materialAt(0, 1, 0)).toBeNull()
  })

  it('returns null outside the world', () => {
    expect(VoxelGrid.fromFills([]).materialAt(-1, 0, 0)).toBeNull()
    expect(VoxelGrid.fromFills([]).materialAt(128, 0, 0)).toBeNull()
  })

  it('reports the top surface height of a column in metres', () => {
    const grid = VoxelGrid.fromFills([{ from: [0, 0, 0], to: [127, 1, 127], material: 'stone' }])
    expect(grid.topSurfaceYM(10.2, 32.7)).toBe(1)
    expect(grid.topSurfaceYM(-1, 0)).toBeNull()
  })
})
```

`packages/sim-core/src/voxel/static-merge.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { cellBoxToWorldBox, mergeStaticBlocks, type CellBox } from './static-merge'
import { VoxelGrid } from './voxel-grid'

function cellsCovered(boxes: readonly CellBox[]): number {
  return boxes.reduce((sum, box) => sum + (box.max[0] - box.min[0] + 1) * (box.max[1] - box.min[1] + 1) * (box.max[2] - box.min[2] + 1), 0)
}

describe('mergeStaticBlocks', () => {
  it('merges a 128 × 2 × 128 ground into one box per 16³ chunk column (64 boxes)', () => {
    const grid = VoxelGrid.fromFills([{ from: [0, 0, 0], to: [127, 1, 127], material: 'stone' }])
    const boxes = mergeStaticBlocks(grid)
    expect(boxes).toHaveLength(64)
    expect(cellsCovered(boxes)).toBe(128 * 2 * 128)
  })

  it('never merges different materials', () => {
    const grid = VoxelGrid.fromFills([
      { from: [0, 0, 0], to: [3, 0, 0], material: 'stone' },
      { from: [4, 0, 0], to: [7, 0, 0], material: 'wood' },
    ])
    const boxes = mergeStaticBlocks(grid)
    expect(boxes.map((box) => box.material).sort()).toEqual(['stone', 'wood'])
  })

  it('covers an L-shape exactly once', () => {
    const grid = VoxelGrid.fromFills([
      { from: [0, 0, 0], to: [4, 0, 0], material: 'stone' },
      { from: [0, 0, 1], to: [0, 0, 4], material: 'stone' },
    ])
    expect(cellsCovered(mergeStaticBlocks(grid))).toBe(9)
  })

  it('is deterministic', () => {
    const fills = [{ from: [3, 0, 5], to: [40, 3, 20], material: 'metal' }] as const
    expect(mergeStaticBlocks(VoxelGrid.fromFills(fills))).toEqual(mergeStaticBlocks(VoxelGrid.fromFills(fills)))
  })
})

describe('cellBoxToWorldBox', () => {
  it('converts inclusive cell boxes to centre and half extents in metres', () => {
    expect(cellBoxToWorldBox({ min: [0, 0, 0], max: [15, 1, 15], material: 'stone' })).toEqual({
      centerM: [4, 0.5, 4],
      halfExtentsM: [4, 0.5, 4],
    })
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run --project sim-core`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement**

`packages/sim-core/src/geometry/vec.ts`:
```ts
import type { Vec3 } from '@physics-lab/formats'

export function addVec3(a: Vec3, b: Vec3): Vec3 {
  return [a[0] + b[0], a[1] + b[1], a[2] + b[2]]
}

export function subVec3(a: Vec3, b: Vec3): Vec3 {
  return [a[0] - b[0], a[1] - b[1], a[2] - b[2]]
}

export function scaleVec3(v: Vec3, factor: number): Vec3 {
  return [v[0] * factor, v[1] * factor, v[2] * factor]
}

export function dotVec3(a: Vec3, b: Vec3): number {
  return a[0] * b[0] + a[1] * b[1] + a[2] * b[2]
}

export function lengthVec3(v: Vec3): number {
  return Math.sqrt(dotVec3(v, v))
}

export function normalizeVec3(v: Vec3): Vec3 {
  return scaleVec3(v, 1 / lengthVec3(v))
}

/** Distance between two points projected on the horizontal (x, z) plane. */
export function horizontalDistanceM(a: Vec3, b: Vec3): number {
  const dx = b[0] - a[0]
  const dz = b[2] - a[2]
  return Math.sqrt(dx * dx + dz * dz)
}
```

`packages/sim-core/src/geometry/quat.ts`:
```ts
import { degToRad, detAtan2, detCos, detSin, radToDeg } from '@physics-lab/det-math'
import type { Vec3 } from '@physics-lab/formats'

/** Unit quaternion (x, y, z, w). */
export type Quat = readonly [number, number, number, number]
export const IDENTITY_QUAT: Quat = [0, 0, 0, 1]
const FULL_TURN_DEG = 360
const HALF_TURN_DEG = 180

export function multiplyQuat(a: Quat, b: Quat): Quat {
  const [ax, ay, az, aw] = a
  const [bx, by, bz, bw] = b
  return [
    aw * bx + ax * bw + ay * bz - az * by,
    aw * by - ax * bz + ay * bw + az * bx,
    aw * bz + ax * by - ay * bx + az * bw,
    aw * bw - ax * bx - ay * by - az * bz,
  ]
}

export function conjugateQuat(q: Quat): Quat {
  return [-q[0], -q[1], -q[2], q[3]]
}

export function rotateVec3(q: Quat, v: Vec3): Vec3 {
  const [x, y, z, w] = q
  const tx = 2 * (y * v[2] - z * v[1])
  const ty = 2 * (z * v[0] - x * v[2])
  const tz = 2 * (x * v[1] - y * v[0])
  return [
    v[0] + w * tx + (y * tz - z * ty),
    v[1] + w * ty + (z * tx - x * tz),
    v[2] + w * tz + (x * ty - y * tx),
  ]
}

/** Intrinsic XYZ Euler angles (three.js order 'XYZ') to a quaternion, using deterministic trig. */
export function quatFromEulerDeg(eulerDeg: Vec3): Quat {
  const hx = degToRad(eulerDeg[0]) / 2
  const hy = degToRad(eulerDeg[1]) / 2
  const hz = degToRad(eulerDeg[2]) / 2
  const c1 = detCos(hx)
  const c2 = detCos(hy)
  const c3 = detCos(hz)
  const s1 = detSin(hx)
  const s2 = detSin(hy)
  const s3 = detSin(hz)
  return [
    s1 * c2 * c3 + c1 * s2 * s3,
    c1 * s2 * c3 - s1 * c2 * s3,
    c1 * c2 * s3 + s1 * s2 * c3,
    c1 * c2 * c3 - s1 * s2 * s3,
  ]
}

/** Signed rotation angle of `q` about `unitAxis` (twist component), in (−180°, 180°]. */
export function angleAboutAxisDeg(q: Quat, unitAxis: Vec3): number {
  const projection = q[0] * unitAxis[0] + q[1] * unitAxis[1] + q[2] * unitAxis[2]
  const angle = radToDeg(2 * detAtan2(projection, q[3]))
  if (angle > HALF_TURN_DEG) return angle - FULL_TURN_DEG
  if (angle <= -HALF_TURN_DEG) return angle + FULL_TURN_DEG
  return angle
}
```

`packages/sim-core/src/materials.ts`:
```ts
import type { MaterialId } from '@physics-lab/formats'

export type MaterialProps = { densityKgPerM3: number; friction: number; restitution: number }

/**
 * Phase 1 material coefficients (spec §5.1). Densities are typical handbook values; friction is a single
 * coefficient because Rapier has one per collider (static vs kinetic is F17).
 */
export const MATERIALS: Readonly<Record<MaterialId, MaterialProps>> = {
  wood: { densityKgPerM3: 600, friction: 0.5, restitution: 0.4 },
  stone: { densityKgPerM3: 2400, friction: 0.6, restitution: 0.2 },
  metal: { densityKgPerM3: 7800, friction: 0.4, restitution: 0.3 },
  rubber: { densityKgPerM3: 1100, friction: 0.9, restitution: 0.8 },
  ice: { densityKgPerM3: 917, friction: 0.05, restitution: 0.1 },
}
```

`packages/sim-core/src/voxel/voxel-grid.ts`:
```ts
import { BLOCK_SIZE_M, MaterialIdSchema, WORLD_SIZE_CELLS, type BlockFill, type GridCoord, type MaterialId } from '@physics-lab/formats'

type GridSize = { readonly x: number; readonly y: number; readonly z: number }
const MATERIAL_ORDER = MaterialIdSchema.options
const EMPTY_CELL = 0

/** Dense voxel grid; each cell stores 0 (empty) or 1 + the index of its material. */
export class VoxelGrid {
  private readonly cells: Uint8Array

  private constructor(readonly size: GridSize) {
    this.cells = new Uint8Array(size.x * size.y * size.z)
  }

  static fromFills(fills: readonly BlockFill[], size: GridSize = WORLD_SIZE_CELLS): VoxelGrid {
    const grid = new VoxelGrid(size)
    for (const fill of fills) grid.fill(fill.from, fill.to, fill.material)
    return grid
  }

  fill(from: GridCoord, to: GridCoord, material: MaterialId): void {
    const code = MATERIAL_ORDER.indexOf(material) + 1
    for (let y = from[1]; y <= to[1]; y++) {
      for (let z = from[2]; z <= to[2]; z++) {
        for (let x = from[0]; x <= to[0]; x++) {
          if (this.contains(x, y, z)) this.cells[this.index(x, y, z)] = code
        }
      }
    }
  }

  materialAt(x: number, y: number, z: number): MaterialId | null {
    if (!this.contains(x, y, z)) return null
    const code = this.cells[this.index(x, y, z)] ?? EMPTY_CELL
    return code === EMPTY_CELL ? null : (MATERIAL_ORDER[code - 1] ?? null)
  }

  /** Height in metres of the top face of the highest solid cell in the column containing (xM, zM). */
  topSurfaceYM(xM: number, zM: number): number | null {
    const x = Math.floor(xM / BLOCK_SIZE_M)
    const z = Math.floor(zM / BLOCK_SIZE_M)
    if (!this.contains(x, 0, z)) return null
    for (let y = this.size.y - 1; y >= 0; y--) {
      if (this.materialAt(x, y, z) !== null) return (y + 1) * BLOCK_SIZE_M
    }
    return null
  }

  private contains(x: number, y: number, z: number): boolean {
    return x >= 0 && y >= 0 && z >= 0 && x < this.size.x && y < this.size.y && z < this.size.z
  }

  private index(x: number, y: number, z: number): number {
    return (y * this.size.z + z) * this.size.x + x
  }
}
```

`packages/sim-core/src/voxel/static-merge.ts`:
```ts
import { BLOCK_SIZE_M, type GridCoord, type MaterialId, type Vec3 } from '@physics-lab/formats'
import type { VoxelGrid } from './voxel-grid'

/** Static blocks are merged per 16³ chunk into box colliders (spec §5.1). */
export const CHUNK_SIZE_CELLS = 16

/** Inclusive box of cells sharing one material. */
export type CellBox = { min: GridCoord; max: GridCoord; material: MaterialId }

type ChunkBounds = { origin: GridCoord; limit: GridCoord }

class ChunkVisitMap {
  private readonly visited = new Uint8Array(CHUNK_SIZE_CELLS * CHUNK_SIZE_CELLS * CHUNK_SIZE_CELLS)

  constructor(private readonly origin: GridCoord) {}

  isVisited(x: number, y: number, z: number): boolean {
    return this.visited[this.index(x, y, z)] === 1
  }

  markBox(min: GridCoord, max: GridCoord): void {
    for (let y = min[1]; y <= max[1]; y++) {
      for (let z = min[2]; z <= max[2]; z++) {
        for (let x = min[0]; x <= max[0]; x++) this.visited[this.index(x, y, z)] = 1
      }
    }
  }

  private index(x: number, y: number, z: number): number {
    const lx = x - this.origin[0]
    const ly = y - this.origin[1]
    const lz = z - this.origin[2]
    return (ly * CHUNK_SIZE_CELLS + lz) * CHUNK_SIZE_CELLS + lx
  }
}

/** Greedy box merge inside each chunk; chunks and cells are visited in a fixed order, so output is deterministic. */
export function mergeStaticBlocks(grid: VoxelGrid): CellBox[] {
  const boxes: CellBox[] = []
  for (let cy = 0; cy < grid.size.y; cy += CHUNK_SIZE_CELLS) {
    for (let cz = 0; cz < grid.size.z; cz += CHUNK_SIZE_CELLS) {
      for (let cx = 0; cx < grid.size.x; cx += CHUNK_SIZE_CELLS) {
        const origin: GridCoord = [cx, cy, cz]
        const limit: GridCoord = [
          Math.min(cx + CHUNK_SIZE_CELLS, grid.size.x),
          Math.min(cy + CHUNK_SIZE_CELLS, grid.size.y),
          Math.min(cz + CHUNK_SIZE_CELLS, grid.size.z),
        ]
        boxes.push(...mergeChunk(grid, { origin, limit }))
      }
    }
  }
  return boxes
}

function mergeChunk(grid: VoxelGrid, bounds: ChunkBounds): CellBox[] {
  const visits = new ChunkVisitMap(bounds.origin)
  const boxes: CellBox[] = []
  const [x0, y0, z0] = bounds.origin
  const [xEnd, yEnd, zEnd] = bounds.limit
  for (let y = y0; y < yEnd; y++) {
    for (let z = z0; z < zEnd; z++) {
      for (let x = x0; x < xEnd; x++) {
        const material = grid.materialAt(x, y, z)
        if (material === null || visits.isVisited(x, y, z)) continue
        const box = growBox(grid, visits, bounds, [x, y, z], material)
        visits.markBox(box.min, box.max)
        boxes.push(box)
      }
    }
  }
  return boxes
}

function growBox(grid: VoxelGrid, visits: ChunkVisitMap, bounds: ChunkBounds, start: GridCoord, material: MaterialId): CellBox {
  const canTake = (x: number, y: number, z: number): boolean =>
    grid.materialAt(x, y, z) === material && !visits.isVisited(x, y, z)
  const [x, y, z] = start
  let x1 = x
  while (x1 + 1 < bounds.limit[0] && canTake(x1 + 1, y, z)) x1++
  let z1 = z
  while (z1 + 1 < bounds.limit[2] && rangeFree(canTake, [x, x1], [y, y], [z1 + 1, z1 + 1])) z1++
  let y1 = y
  while (y1 + 1 < bounds.limit[1] && rangeFree(canTake, [x, x1], [y1 + 1, y1 + 1], [z, z1])) y1++
  return { min: [x, y, z], max: [x1, y1, z1], material }
}

type Span = readonly [number, number]

function rangeFree(canTake: (x: number, y: number, z: number) => boolean, xs: Span, ys: Span, zs: Span): boolean {
  for (let y = ys[0]; y <= ys[1]; y++) {
    for (let z = zs[0]; z <= zs[1]; z++) {
      for (let x = xs[0]; x <= xs[1]; x++) if (!canTake(x, y, z)) return false
    }
  }
  return true
}

export function cellBoxToWorldBox(box: CellBox): { centerM: Vec3; halfExtentsM: Vec3 } {
  const center = (axis: 0 | 1 | 2): number => ((box.min[axis] + box.max[axis] + 1) / 2) * BLOCK_SIZE_M
  const half = (axis: 0 | 1 | 2): number => ((box.max[axis] - box.min[axis] + 1) / 2) * BLOCK_SIZE_M
  return { centerM: [center(0), center(1), center(2)], halfExtentsM: [half(0), half(1), half(2)] }
}
```

`packages/sim-core/src/index.ts`:
```ts
export * from './geometry/quat'
export * from './geometry/vec'
export * from './materials'
export * from './voxel/static-merge'
export * from './voxel/voxel-grid'
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm vitest run --project sim-core`
Expected: PASS.

- [ ] **Step 5: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test
git add packages/sim-core
git commit -m "feat(sim-core): add geometry helpers, materials and voxel block merging"
```

---

### Task 6: `sim-core` — build commands

**Files:**
- Create: `packages/sim-core/src/commands/build-commands.ts`
- Modify: `packages/sim-core/src/index.ts`
- Test: `packages/sim-core/src/commands/build-commands.test.ts`

**Interfaces:**
- Consumes: `Command`, `WorldDoc`, `WorldDocSchema`, `EntityId`, `refToId`, `DocumentIssue` from formats.
- Produces:
  - `type BuildCommand = Extract<Command, { type: 'placeBlock' | 'fill' | 'addPart' | 'connect' | 'setProperty' | 'setPhysics' }>`
  - `isBuildCommand(command: Command): command is BuildCommand`
  - `type BuildError = { code: 'locked'; target: EntityId } | { code: 'unknownEntity'; target: EntityId } | { code: 'unknownProperty'; target: EntityId; property: string } | { code: 'outOfRange'; property: string; min: number; max: number } | { code: 'invalidWorld'; issues: DocumentIssue[] }`
  - `type BuildOutcome = { ok: true; doc: WorldDoc } | { ok: false; error: BuildError }`
  - `applyBuildCommand(doc: WorldDoc, command: BuildCommand, locks: ReadonlySet<EntityId>): BuildOutcome` — pure; never mutates `doc`.
  - `NO_LOCKS: ReadonlySet<EntityId>`
  - `TERRAIN_LOCK_ID`: the id `'terrain'` used in `locks` to forbid block edits.

Settable properties in M1 (D44): latch `releaseAtHingeAngleDeg` ∈ [−360, 360]; hinge `motorTargetSpeedDegPerS` ∈ [−3600, 3600] and `motorGain` ∈ (0, 1e7] (only when the hinge has a motor).

- [ ] **Step 1: Write the failing tests**

`packages/sim-core/src/commands/build-commands.test.ts`:
```ts
import type { WorldDoc } from '@physics-lab/formats'
import { describe, expect, it } from 'vitest'
import { NO_LOCKS, applyBuildCommand } from './build-commands'

function world(): WorldDoc {
  return {
    format: 'physics-lab/world',
    formatVersion: 1,
    gravityMps2: 9.8,
    blocks: [],
    parts: [
      { id: 'post', type: 'beam', motion: 'static', material: 'wood', layer: 'machine', positionM: [0, 1, 0], rotationDeg: [0, 0, 0], ccd: false, sizeM: [0.2, 2, 0.2] },
      { id: 'arm', type: 'beam', motion: 'dynamic', material: 'wood', layer: 'machine', positionM: [0.8, 2, 0], rotationDeg: [0, 0, 0], ccd: false, sizeM: [1.6, 0.1, 0.1] },
      { id: 'ball', type: 'sphere', motion: 'dynamic', material: 'stone', layer: 'projectile', positionM: [1.5, 2, 0], rotationDeg: [0, 0, 0], ccd: true, radiusM: 0.1 },
    ],
    joints: [
      { id: 'pivot', type: 'hinge', a: 'post', b: 'arm', anchorM: [0, 2, 0], axis: [0, 0, -1], motor: { targetSpeedDegPerS: 450, gain: 10000 } },
      { id: 'hold', type: 'fixed', a: 'arm', b: 'ball' },
    ],
    latches: [{ id: 'latch', type: 'angleLatch', hinge: 'pivot', releasesJoint: 'hold', releaseAtHingeAngleDeg: 90 }],
  }
}

describe('applyBuildCommand', () => {
  it('sets a latch release angle without mutating the input', () => {
    const input = world()
    const outcome = applyBuildCommand(input, { type: 'setProperty', target: '#latch', property: 'releaseAtHingeAngleDeg', value: 80 }, NO_LOCKS)
    expect(outcome.ok && outcome.doc.latches[0]?.releaseAtHingeAngleDeg).toBe(80)
    expect(input.latches[0]?.releaseAtHingeAngleDeg).toBe(90)
  })

  it('sets the hinge motor speed', () => {
    const outcome = applyBuildCommand(world(), { type: 'setProperty', target: '#pivot', property: 'motorTargetSpeedDegPerS', value: 300 }, NO_LOCKS)
    expect(outcome.ok && outcome.doc.joints[0]?.type === 'hinge' && outcome.doc.joints[0].motor?.targetSpeedDegPerS).toBe(300)
  })

  it('refuses to modify a locked entity', () => {
    const outcome = applyBuildCommand(world(), { type: 'setProperty', target: '#latch', property: 'releaseAtHingeAngleDeg', value: 80 }, new Set(['latch']))
    expect(outcome).toEqual({ ok: false, error: { code: 'locked', target: 'latch' } })
  })

  it('reports unknown entities and properties', () => {
    expect(applyBuildCommand(world(), { type: 'setProperty', target: '#ghost', property: 'x', value: 1 }, NO_LOCKS)).toEqual({
      ok: false,
      error: { code: 'unknownEntity', target: 'ghost' },
    })
    expect(applyBuildCommand(world(), { type: 'setProperty', target: '#latch', property: 'colour', value: 1 }, NO_LOCKS)).toEqual({
      ok: false,
      error: { code: 'unknownProperty', target: 'latch', property: 'colour' },
    })
  })

  it('rejects values outside the property range', () => {
    expect(applyBuildCommand(world(), { type: 'setProperty', target: '#latch', property: 'releaseAtHingeAngleDeg', value: 400 }, NO_LOCKS)).toEqual({
      ok: false,
      error: { code: 'outOfRange', property: 'releaseAtHingeAngleDeg', min: -360, max: 360 },
    })
  })

  it('places blocks and fills regions', () => {
    const placed = applyBuildCommand(world(), { type: 'placeBlock', cell: [1, 2, 3], material: 'wood' }, NO_LOCKS)
    expect(placed.ok && placed.doc.blocks).toEqual([{ from: [1, 2, 3], to: [1, 2, 3], material: 'wood' }])
    const filled = applyBuildCommand(world(), { type: 'fill', from: [0, 0, 0], to: [3, 0, 3], material: 'stone' }, NO_LOCKS)
    expect(filled.ok && filled.doc.blocks).toHaveLength(1)
  })

  it('refuses block edits when the terrain is locked', () => {
    const outcome = applyBuildCommand(world(), { type: 'placeBlock', cell: [1, 2, 3], material: 'wood' }, new Set(['terrain']))
    expect(outcome).toEqual({ ok: false, error: { code: 'locked', target: 'terrain' } })
  })

  it('rejects an added part whose id already exists, with the schema path', () => {
    const duplicate = { id: 'arm', type: 'sphere', motion: 'dynamic', material: 'stone', layer: 'projectile', positionM: [3, 2, 0], rotationDeg: [0, 0, 0], ccd: false, radiusM: 0.1 } as const
    const outcome = applyBuildCommand(world(), { type: 'addPart', part: duplicate }, NO_LOCKS)
    expect(outcome.ok).toBe(false)
    if (!outcome.ok && outcome.error.code === 'invalidWorld') {
      expect(outcome.error.issues.map((issue) => issue.path)).toContain('parts.3.id')
    }
  })

  it('refuses a joint that touches a locked part', () => {
    const outcome = applyBuildCommand(world(), { type: 'connect', joint: { id: 'extra', type: 'fixed', a: 'post', b: 'ball' } }, new Set(['post']))
    expect(outcome).toEqual({ ok: false, error: { code: 'locked', target: 'post' } })
  })

  it('changes gravity', () => {
    const outcome = applyBuildCommand(world(), { type: 'setPhysics', gravityMps2: 1.62 }, NO_LOCKS)
    expect(outcome.ok && outcome.doc.gravityMps2).toBe(1.62)
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `pnpm vitest run --project sim-core build-commands`
Expected: FAIL — `./build-commands` not found.

- [ ] **Step 3: Implement**

`packages/sim-core/src/commands/build-commands.ts`:
```ts
import { TERRAIN_ENTITY_ID, WorldDocSchema, refToId, type Command, type DocumentIssue, type EntityId, type WorldDoc } from '@physics-lab/formats'

export type BuildCommand = Extract<Command, { type: 'placeBlock' | 'fill' | 'addPart' | 'connect' | 'setProperty' | 'setPhysics' }>
export type BuildError =
  | { code: 'locked'; target: EntityId }
  | { code: 'unknownEntity'; target: EntityId }
  | { code: 'unknownProperty'; target: EntityId; property: string }
  | { code: 'outOfRange'; property: string; min: number; max: number }
  | { code: 'invalidWorld'; issues: DocumentIssue[] }
export type BuildOutcome = { ok: true; doc: WorldDoc } | { ok: false; error: BuildError }

export const NO_LOCKS: ReadonlySet<EntityId> = new Set()
export const TERRAIN_LOCK_ID = TERRAIN_ENTITY_ID

const BUILD_COMMAND_TYPES: ReadonlySet<Command['type']> = new Set(['placeBlock', 'fill', 'addPart', 'connect', 'setProperty', 'setPhysics'])

export function isBuildCommand(command: Command): command is BuildCommand {
  return BUILD_COMMAND_TYPES.has(command.type)
}

type PropertyRule = { min: number; max: number; apply: (doc: WorldDoc, id: EntityId, value: number) => WorldDoc | null }

const LATCH_PROPERTIES: Readonly<Record<string, PropertyRule>> = {
  releaseAtHingeAngleDeg: {
    min: -360,
    max: 360,
    apply: (doc, id, value) => ({
      ...doc,
      latches: doc.latches.map((latch) => (latch.id === id ? { ...latch, releaseAtHingeAngleDeg: value } : latch)),
    }),
  },
}

function updateMotor(doc: WorldDoc, id: EntityId, patch: { targetSpeedDegPerS?: number; gain?: number }): WorldDoc | null {
  const hinge = doc.joints.find((joint) => joint.id === id)
  if (hinge?.type !== 'hinge' || hinge.motor === null) return null
  const motor = { ...hinge.motor, ...patch }
  return { ...doc, joints: doc.joints.map((joint) => (joint.id === id ? { ...hinge, motor } : joint)) }
}

const HINGE_PROPERTIES: Readonly<Record<string, PropertyRule>> = {
  motorTargetSpeedDegPerS: { min: -3600, max: 3600, apply: (doc, id, value) => updateMotor(doc, id, { targetSpeedDegPerS: value }) },
  motorGain: { min: Number.MIN_VALUE, max: 1e7, apply: (doc, id, value) => updateMotor(doc, id, { gain: value }) },
}

function propertyRulesFor(doc: WorldDoc, id: EntityId): Readonly<Record<string, PropertyRule>> | null {
  if (doc.latches.some((latch) => latch.id === id)) return LATCH_PROPERTIES
  if (doc.joints.some((joint) => joint.id === id && joint.type === 'hinge')) return HINGE_PROPERTIES
  if (doc.parts.some((part) => part.id === id) || doc.joints.some((joint) => joint.id === id)) return {}
  return null
}

function setProperty(doc: WorldDoc, command: Extract<BuildCommand, { type: 'setProperty' }>, locks: ReadonlySet<EntityId>): WorldDoc | BuildError {
  const target = refToId(command.target)
  const rules = propertyRulesFor(doc, target)
  if (rules === null) return { code: 'unknownEntity', target }
  if (locks.has(target)) return { code: 'locked', target }
  const rule = rules[command.property]
  if (!rule) return { code: 'unknownProperty', target, property: command.property }
  if (command.value < rule.min || command.value > rule.max) {
    return { code: 'outOfRange', property: command.property, min: rule.min, max: rule.max }
  }
  return rule.apply(doc, target, command.value) ?? { code: 'unknownProperty', target, property: command.property }
}

function firstLocked(ids: readonly EntityId[], locks: ReadonlySet<EntityId>): BuildError | null {
  const locked = ids.find((id) => locks.has(id))
  return locked === undefined ? null : { code: 'locked', target: locked }
}

function applyUnvalidated(doc: WorldDoc, command: BuildCommand, locks: ReadonlySet<EntityId>): WorldDoc | BuildError {
  // The command union is closed by the formats schema; a new build command starts there.
  switch (command.type) {
    case 'placeBlock':
      return firstLocked([TERRAIN_LOCK_ID], locks) ?? { ...doc, blocks: [...doc.blocks, { from: command.cell, to: command.cell, material: command.material }] }
    case 'fill':
      return firstLocked([TERRAIN_LOCK_ID], locks) ?? { ...doc, blocks: [...doc.blocks, { from: command.from, to: command.to, material: command.material }] }
    case 'addPart':
      return { ...doc, parts: [...doc.parts, command.part] }
    case 'connect':
      return firstLocked([command.joint.a, command.joint.b], locks) ?? { ...doc, joints: [...doc.joints, command.joint] }
    case 'setProperty':
      return setProperty(doc, command, locks)
    case 'setPhysics':
      return { ...doc, gravityMps2: command.gravityMps2 }
  }
}

function isBuildError(value: WorldDoc | BuildError): value is BuildError {
  return 'code' in value
}

/** Pure: returns a new, schema-valid WorldDoc or a typed error. */
export function applyBuildCommand(doc: WorldDoc, command: BuildCommand, locks: ReadonlySet<EntityId>): BuildOutcome {
  const next = applyUnvalidated(doc, command, locks)
  if (isBuildError(next)) return { ok: false, error: next }
  const validated = WorldDocSchema.safeParse(next)
  if (!validated.success) {
    const issues = validated.error.issues.map((issue) => ({ path: issue.path.map(String).join('.'), message: issue.message }))
    return { ok: false, error: { code: 'invalidWorld', issues } }
  }
  return { ok: true, doc: validated.data }
}
```

Append to `packages/sim-core/src/index.ts`:
```ts
export * from './commands/build-commands'
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `pnpm vitest run --project sim-core build-commands`
Expected: PASS.

- [ ] **Step 5: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test
git add packages/sim-core
git commit -m "feat(sim-core): add pure build commands with locks and validation"
```

---
### Task 7: `sim-core` — `PhysicsEngine` port, Rapier adapter and contract tests

**Files:**
- Create: `packages/sim-core/src/ports/physics-engine.ts`, `adapters/rapier/rapier-physics-engine.ts`, `adapters/rapier/index.ts`, `testing/physics-engine-contract.ts`
- Modify: `packages/sim-core/package.json` (add `"./rapier": "./src/adapters/rapier/index.ts"` to `exports`), `packages/sim-core/src/index.ts`
- Test: `packages/sim-core/src/adapters/rapier/rapier-physics-engine.test.ts`

**Interfaces:**
- Consumes: `Vec3`, `EntityId` (formats); `Quat`, `IDENTITY_QUAT`, `rotateVec3`, `conjugateQuat`, `multiplyQuat`, `subVec3` (Task 5); `MaterialProps` (Task 5).
- Produces (port, `ports/physics-engine.ts`):
```ts
export type CollisionLayer = 'terrain' | 'machine' | 'projectile'
export type ShapeSpec = { kind: 'box'; halfExtentsM: Vec3 } | { kind: 'sphere'; radiusM: number }
export type BodySpec = {
  entityId: EntityId
  motion: 'static' | 'dynamic'
  shape: ShapeSpec
  positionM: Vec3
  rotation: Quat
  material: MaterialProps
  layer: CollisionLayer
  ccd: boolean
  initialVelocityMps: Vec3
}
export type TerrainBox = { centerM: Vec3; halfExtentsM: Vec3; material: MaterialProps }
export type HingeSpec = { jointId: EntityId; bodyA: EntityId; bodyB: EntityId; anchorM: Vec3; axisWorld: Vec3 }
export type FixedSpec = { jointId: EntityId; bodyA: EntityId; bodyB: EntityId }
export type BodyState = { positionM: Vec3; rotation: Quat; linearVelocityMps: Vec3; angularVelocityRadPerS: Vec3 }
/** Entity ids of a contact that started during the step, sorted ascending. */
export type ContactPair = readonly [EntityId, EntityId]
export type EngineConfig = { gravityMps2: number; fixedStepS: number; solverIterations: number }
export interface PhysicsEngine {
  addTerrain(entityId: EntityId, boxes: readonly TerrainBox[]): void
  addBody(spec: BodySpec): void
  addHinge(spec: HingeSpec): void
  addFixed(spec: FixedSpec): void
  removeJoint(jointId: EntityId): void
  setHingeMotor(jointId: EntityId, targetSpeedRadPerS: number, gain: number): void
  /** Advances one fixed step; returns contact pairs that started, deduplicated and sorted. */
  step(): readonly ContactPair[]
  readBody(entityId: EntityId): BodyState
  dispose(): void
}
export interface PhysicsEngineFactory {
  /** Recorded in results as engineVersion (D16, D43). */
  readonly engineId: string
  create(config: EngineConfig): PhysicsEngine
}
```
- Produces (adapter, `@physics-lab/sim-core/rapier`): `RAPIER_ENGINE_ID`, `createRapierEngineFactory(): Promise<PhysicsEngineFactory>`.
- Produces (testing): `describePhysicsEngineContract(name: string, createFactory: () => Promise<PhysicsEngineFactory>): void`.

Collision layers: `terrain` collides with everything; `machine` with terrain and machine; `projectile` with terrain and projectile (so the released ball never hits the arm). Jointed bodies never collide with each other.

- [ ] **Step 1: Write the port** (`ports/physics-engine.ts`, exactly the block above, importing `Vec3`, `EntityId` from formats, `Quat` from `../geometry/quat`, `MaterialProps` from `../materials`). Export it from `src/index.ts`:
```ts
export * from './ports/physics-engine'
```

- [ ] **Step 2: Write the contract suite and the failing adapter test**

`packages/sim-core/src/testing/physics-engine-contract.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { IDENTITY_QUAT } from '../geometry/quat'
import { MATERIALS } from '../materials'
import type { BodySpec, EngineConfig, PhysicsEngine, PhysicsEngineFactory } from '../ports/physics-engine'

const STEP_S = 1 / 240
const ONE_SECOND_STEPS = 240

function sphere(entityId: string, positionM: BodySpec['positionM'], overrides: Partial<BodySpec> = {}): BodySpec {
  return {
    entityId,
    motion: 'dynamic',
    shape: { kind: 'sphere', radiusM: 0.1 },
    positionM,
    rotation: IDENTITY_QUAT,
    material: MATERIALS.stone,
    layer: 'projectile',
    ccd: true,
    initialVelocityMps: [0, 0, 0],
    ...overrides,
  }
}

function beam(entityId: string, positionM: BodySpec['positionM'], overrides: Partial<BodySpec> = {}): BodySpec {
  return { ...sphere(entityId, positionM), shape: { kind: 'box', halfExtentsM: [0.5, 0.05, 0.05] }, layer: 'machine', ccd: false, material: MATERIALS.wood, ...overrides }
}

function stepMany(engine: PhysicsEngine, count: number): void {
  for (let i = 0; i < count; i++) engine.step()
}

/** Contract every PhysicsEngine adapter must honor (Liskov, D30): units, layers, joints, events, determinism. */
export function describePhysicsEngineContract(name: string, createFactory: () => Promise<PhysicsEngineFactory>): void {
  describe(`PhysicsEngine contract: ${name}`, () => {
    const config = (gravityMps2: number): EngineConfig => ({ gravityMps2, fixedStepS: STEP_S, solverIterations: 8 })

    it('integrates free fall in SI units', async () => {
      const engine = (await createFactory()).create(config(9.8))
      engine.addBody(sphere('ball', [0, 100, 0]))
      stepMany(engine, ONE_SECOND_STEPS)
      const state = engine.readBody('ball')
      expect(state.linearVelocityMps[1]).toBeCloseTo(-9.8, 6)
      expect(state.positionM[1]).toBeCloseTo(100 - 0.5 * 9.8, 1)
      engine.dispose()
    })

    it('reports a sorted contact pair when a ball lands on terrain', async () => {
      const engine = (await createFactory()).create(config(9.8))
      engine.addTerrain('terrain', [{ centerM: [0, 0.5, 0], halfExtentsM: [10, 0.5, 10], material: MATERIALS.stone }])
      engine.addBody(sphere('ball', [0, 2, 0]))
      let contacts: readonly (readonly [string, string])[] = []
      for (let i = 0; i < 2 * ONE_SECOND_STEPS && contacts.length === 0; i++) contacts = engine.step()
      expect(contacts).toEqual([['ball', 'terrain']])
      expect(engine.readBody('ball').positionM[1]).toBeCloseTo(1.1, 1)
      engine.dispose()
    })

    it('lets projectiles pass through machine parts', async () => {
      const engine = (await createFactory()).create(config(9.8))
      engine.addBody(beam('beam', [0, 1, 0], { motion: 'static', shape: { kind: 'box', halfExtentsM: [2, 0.1, 2] } }))
      engine.addBody(sphere('ball', [0, 2, 0]))
      const contacts = Array.from({ length: ONE_SECOND_STEPS }, () => engine.step()).flat()
      expect(contacts).toEqual([])
      expect(engine.readBody('ball').positionM[1]).toBeLessThan(1)
      engine.dispose()
    })

    it('drives a hinge motor to its target angular speed', async () => {
      const engine = (await createFactory()).create(config(0))
      engine.addBody(beam('post', [0, 0, 0], { motion: 'static', shape: { kind: 'box', halfExtentsM: [0.05, 0.05, 0.05] } }))
      engine.addBody(beam('arm', [0.5, 0, 0]))
      engine.addHinge({ jointId: 'pivot', bodyA: 'post', bodyB: 'arm', anchorM: [0, 0, 0], axisWorld: [0, 0, 1] })
      engine.setHingeMotor('pivot', 2, 1e4)
      stepMany(engine, ONE_SECOND_STEPS / 2)
      expect(engine.readBody('arm').angularVelocityRadPerS[2]).toBeCloseTo(2, 1)
      engine.dispose()
    })

    it('frees a body when its fixed joint is removed', async () => {
      const engine = (await createFactory()).create(config(0))
      engine.addBody(beam('post', [0, 0, 0], { motion: 'static', shape: { kind: 'box', halfExtentsM: [0.05, 0.05, 0.05] } }))
      engine.addBody(beam('arm', [0.5, 0, 0]))
      engine.addBody(sphere('ball', [1, 0, 0]))
      engine.addHinge({ jointId: 'pivot', bodyA: 'post', bodyB: 'arm', anchorM: [0, 0, 0], axisWorld: [0, 0, 1] })
      engine.addFixed({ jointId: 'hold', bodyA: 'arm', bodyB: 'ball' })
      engine.setHingeMotor('pivot', 3, 1e4)
      stepMany(engine, 60)
      engine.removeJoint('hold')
      engine.step()
      const released = engine.readBody('ball').linearVelocityMps
      stepMany(engine, 60)
      const later = engine.readBody('ball').linearVelocityMps
      later.forEach((component, axis) => expect(component).toBeCloseTo(released[axis] ?? Number.NaN, 6))
      expect(Math.abs(released[1])).toBeGreaterThan(1)
      engine.dispose()
    })

    it('does not let a fast CCD projectile tunnel through thin terrain', async () => {
      const engine = (await createFactory()).create(config(0))
      engine.addTerrain('terrain', [{ centerM: [5, 0, 0], halfExtentsM: [0.01, 2, 2], material: MATERIALS.stone }])
      engine.addBody(sphere('ball', [0, 0, 0], { shape: { kind: 'sphere', radiusM: 0.05 }, initialVelocityMps: [200, 0, 0] }))
      stepMany(engine, 60)
      expect(engine.readBody('ball').positionM[0]).toBeLessThan(5)
      engine.dispose()
    })

    it('is bit-for-bit deterministic for identical inputs', async () => {
      const run = async (): Promise<unknown> => {
        const engine = (await createFactory()).create(config(9.8))
        engine.addTerrain('terrain', [{ centerM: [0, 0.5, 0], halfExtentsM: [10, 0.5, 10], material: MATERIALS.stone }])
        engine.addBody(sphere('ball', [0, 3, 0], { initialVelocityMps: [1.3, 2.1, -0.4] }))
        stepMany(engine, 500)
        const state = engine.readBody('ball')
        engine.dispose()
        return state
      }
      expect(await run()).toEqual(await run())
    })

    it('throws for an unknown entity', async () => {
      const engine = (await createFactory()).create(config(9.8))
      expect(() => engine.readBody('ghost')).toThrow()
      engine.dispose()
    })
  })
}
```

`packages/sim-core/src/adapters/rapier/rapier-physics-engine.test.ts`:
```ts
import { expect, it } from 'vitest'
import packageJson from '../../../package.json'
import { describePhysicsEngineContract } from '../../testing/physics-engine-contract'
import { RAPIER_ENGINE_ID, createRapierEngineFactory } from './index'

describePhysicsEngineContract('Rapier (deterministic compat)', createRapierEngineFactory)

it('records the exact pinned Rapier version as the engine id (D43)', () => {
  const pinned = packageJson.dependencies['@dimforge/rapier3d-deterministic-compat']
  expect(RAPIER_ENGINE_ID).toBe(`@dimforge/rapier3d-deterministic-compat@${pinned}`)
})
```
Add `"package.json"` to `packages/sim-core/tsconfig.json` `include`.

- [ ] **Step 3: Run the test to verify it fails**

Run: `pnpm vitest run --project sim-core rapier`
Expected: FAIL — `./index` not found.

- [ ] **Step 4: Implement the adapter**

`packages/sim-core/src/adapters/rapier/rapier-physics-engine.ts`:
```ts
import RAPIER from '@dimforge/rapier3d-deterministic-compat'
import type { EntityId, Vec3 } from '@physics-lab/formats'
import { IDENTITY_QUAT, conjugateQuat, multiplyQuat, rotateVec3, type Quat } from '../../geometry/quat'
import { subVec3 } from '../../geometry/vec'
import type { MaterialProps } from '../../materials'
import type {
  BodySpec, BodyState, CollisionLayer, ContactPair, EngineConfig, FixedSpec, HingeSpec, PhysicsEngine,
  PhysicsEngineFactory, TerrainBox,
} from '../../ports/physics-engine'

// Adapter pattern: Rapier stays behind the PhysicsEngine port so the core is engine-agnostic (D8).
export const RAPIER_ENGINE_ID = '@dimforge/rapier3d-deterministic-compat@0.21.0'

const LAYER_MEMBERSHIP: Readonly<Record<CollisionLayer, number>> = { terrain: 0b001, machine: 0b010, projectile: 0b100 }
const LAYER_FILTER: Readonly<Record<CollisionLayer, number>> = { terrain: 0b111, machine: 0b011, projectile: 0b101 }
const MEMBERSHIP_SHIFT = 16

function interactionGroups(layer: CollisionLayer): number {
  return ((LAYER_MEMBERSHIP[layer] << MEMBERSHIP_SHIFT) | LAYER_FILTER[layer]) >>> 0
}

const toVector = (v: Vec3): RAPIER.Vector => ({ x: v[0], y: v[1], z: v[2] })
const toRotation = (q: Quat): RAPIER.Rotation => ({ x: q[0], y: q[1], z: q[2], w: q[3] })
const fromVector = (v: RAPIER.Vector): Vec3 => [v.x, v.y, v.z]
const fromRotation = (q: RAPIER.Rotation): Quat => [q.x, q.y, q.z, q.w]

function withMaterial(desc: RAPIER.ColliderDesc, material: MaterialProps, layer: CollisionLayer): RAPIER.ColliderDesc {
  return desc
    .setDensity(material.densityKgPerM3)
    .setFriction(material.friction)
    .setRestitution(material.restitution)
    .setCollisionGroups(interactionGroups(layer))
    .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS)
}

function compareIds(a: EntityId, b: EntityId): number {
  return a < b ? -1 : a > b ? 1 : 0
}

class RapierPhysicsEngine implements PhysicsEngine {
  private readonly world: RAPIER.World
  private readonly events = new RAPIER.EventQueue(true)
  private readonly bodies = new Map<EntityId, RAPIER.RigidBody>()
  private readonly entityByCollider = new Map<number, EntityId>()
  private readonly joints = new Map<EntityId, RAPIER.ImpulseJoint>()

  constructor(config: EngineConfig) {
    this.world = new RAPIER.World({ x: 0, y: -config.gravityMps2, z: 0 })
    this.world.timestep = config.fixedStepS
    this.world.numSolverIterations = config.solverIterations
  }

  addTerrain(entityId: EntityId, boxes: readonly TerrainBox[]): void {
    const body = this.world.createRigidBody(RAPIER.RigidBodyDesc.fixed())
    this.bodies.set(entityId, body)
    for (const box of boxes) {
      const desc = RAPIER.ColliderDesc.cuboid(...box.halfExtentsM).setTranslation(...box.centerM)
      const collider = this.world.createCollider(withMaterial(desc, box.material, 'terrain'), body)
      this.entityByCollider.set(collider.handle, entityId)
    }
  }

  addBody(spec: BodySpec): void {
    const bodyDesc = (spec.motion === 'static' ? RAPIER.RigidBodyDesc.fixed() : RAPIER.RigidBodyDesc.dynamic())
      .setTranslation(...spec.positionM)
      .setRotation(toRotation(spec.rotation))
      .setLinvel(...spec.initialVelocityMps)
      .setCcdEnabled(spec.ccd)
    const body = this.world.createRigidBody(bodyDesc)
    const shapeDesc =
      spec.shape.kind === 'box' ? RAPIER.ColliderDesc.cuboid(...spec.shape.halfExtentsM) : RAPIER.ColliderDesc.ball(spec.shape.radiusM)
    const collider = this.world.createCollider(withMaterial(shapeDesc, spec.material, spec.layer), body)
    this.bodies.set(spec.entityId, body)
    this.entityByCollider.set(collider.handle, spec.entityId)
  }

  addHinge(spec: HingeSpec): void {
    const a = this.requireBody(spec.bodyA)
    const b = this.requireBody(spec.bodyB)
    const data = RAPIER.JointData.revoluteWithAxes(
      toVector(this.toLocalPoint(a, spec.anchorM)),
      toVector(this.toLocalPoint(b, spec.anchorM)),
      toVector(rotateVec3(conjugateQuat(fromRotation(a.rotation())), spec.axisWorld)),
      toVector(rotateVec3(conjugateQuat(fromRotation(b.rotation())), spec.axisWorld))
    )
    this.addJoint(spec.jointId, data, a, b)
  }

  addFixed(spec: FixedSpec): void {
    const a = this.requireBody(spec.bodyA)
    const b = this.requireBody(spec.bodyB)
    const rotationA = fromRotation(a.rotation())
    const data = RAPIER.JointData.fixed(
      toVector(this.toLocalPoint(a, fromVector(b.translation()))),
      toRotation(multiplyQuat(conjugateQuat(rotationA), fromRotation(b.rotation()))),
      toVector([0, 0, 0]),
      toRotation(IDENTITY_QUAT)
    )
    this.addJoint(spec.jointId, data, a, b)
  }

  removeJoint(jointId: EntityId): void {
    this.world.removeImpulseJoint(this.requireJoint(jointId), true)
    this.joints.delete(jointId)
  }

  setHingeMotor(jointId: EntityId, targetSpeedRadPerS: number, gain: number): void {
    const joint = this.requireJoint(jointId)
    if (!(joint instanceof RAPIER.RevoluteImpulseJoint)) throw new Error(`Joint "${jointId}" is not a hinge`)
    joint.configureMotorVelocity(targetSpeedRadPerS, gain)
  }

  step(): readonly ContactPair[] {
    this.world.step(this.events)
    const pairs = new Map<string, ContactPair>()
    this.events.drainCollisionEvents((handle1, handle2, started) => {
      if (!started) return
      const first = this.entityByCollider.get(handle1)
      const second = this.entityByCollider.get(handle2)
      if (first === undefined || second === undefined || first === second) return
      const pair: ContactPair = compareIds(first, second) < 0 ? [first, second] : [second, first]
      pairs.set(`${pair[0]}|${pair[1]}`, pair)
    })
    return [...pairs.values()].sort((p, q) => compareIds(p[0], q[0]) || compareIds(p[1], q[1]))
  }

  readBody(entityId: EntityId): BodyState {
    const body = this.requireBody(entityId)
    return {
      positionM: fromVector(body.translation()),
      rotation: fromRotation(body.rotation()),
      linearVelocityMps: fromVector(body.linvel()),
      angularVelocityRadPerS: fromVector(body.angvel()),
    }
  }

  dispose(): void {
    this.events.free()
    this.world.free()
  }

  private addJoint(jointId: EntityId, data: RAPIER.JointData, a: RAPIER.RigidBody, b: RAPIER.RigidBody): void {
    const joint = this.world.createImpulseJoint(data, a, b, true)
    joint.setContactsEnabled(false)
    this.joints.set(jointId, joint)
  }

  private toLocalPoint(body: RAPIER.RigidBody, worldPoint: Vec3): Vec3 {
    return rotateVec3(conjugateQuat(fromRotation(body.rotation())), subVec3(worldPoint, fromVector(body.translation())))
  }

  private requireBody(entityId: EntityId): RAPIER.RigidBody {
    const body = this.bodies.get(entityId)
    if (!body) throw new Error(`Unknown body "${entityId}"`)
    return body
  }

  private requireJoint(jointId: EntityId): RAPIER.ImpulseJoint {
    const joint = this.joints.get(jointId)
    if (!joint) throw new Error(`Unknown joint "${jointId}"`)
    return joint
  }
}

export async function createRapierEngineFactory(): Promise<PhysicsEngineFactory> {
  await RAPIER.init()
  return { engineId: RAPIER_ENGINE_ID, create: (config) => new RapierPhysicsEngine(config) }
}
```

`packages/sim-core/src/adapters/rapier/index.ts`:
```ts
export { RAPIER_ENGINE_ID, createRapierEngineFactory } from './rapier-physics-engine'
```

> If a Rapier type name differs in 0.21 (e.g. `RAPIER.Vector` vs `RAPIER.Vector3`), check `node_modules/@dimforge/rapier3d-deterministic-compat/dist/*.d.ts`; do not change the port.

- [ ] **Step 5: Run the test to verify it passes**

Run: `pnpm vitest run --project sim-core rapier`
Expected: PASS (9 tests). If the hinge-motor or free-fall tolerance fails, stop and report the measured values to `physics-reviewer`; do not loosen tolerances.

- [ ] **Step 6: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test
git add packages/sim-core
git commit -m "feat(sim-core): add physics engine port with rapier adapter"
```

---

### Task 8: `sim-core` — `Simulation` (build, step, trigger, latches, events, hash) and Node determinism

**Files:**
- Create: `packages/sim-core/src/sensors/sensor-host.ts`, `simulation/simulation.ts`, `testing/catapult-test-world.ts`, `determinism/determinism.test.ts`
- Modify: `packages/sim-core/src/index.ts`
- Test: `packages/sim-core/src/simulation/simulation.test.ts`, `determinism/determinism.test.ts`

**Interfaces:**
- Consumes: Tasks 5–7; `WorldDoc`, `TERRAIN_ENTITY_ID`, `EntityId` (formats); `SnapshotHasher`, `degToRad` (det-math).
- Produces:
```ts
// simulation/simulation.ts
export const FIXED_STEP_S = 1 / 240        // D23
export const SOLVER_ITERATIONS = 8         // solver substeps (D17)
export type SimEvent =
  | { type: 'contactStarted'; a: EntityId; b: EntityId; stepIndex: number }
  | { type: 'latchReleased'; latchId: EntityId; stepIndex: number }
  | { type: 'stepCompleted'; stepIndex: number }
export type SimEventListener = (event: SimEvent) => void
export class Simulation implements SensorHost {
  static create(doc: WorldDoc, factory: PhysicsEngineFactory): Simulation
  readonly gravityMps2: number
  get stepIndex(): number
  get timeS(): number
  subscribe(listener: SimEventListener): () => void
  trigger(target: EntityId, action: 'startMotor'): void
  step(): void
  hingeAngleDeg(hingeId: EntityId): number
  readBody(entityId: EntityId): BodyState
  partIds(): readonly EntityId[]                // all parts, sorted
  terrainTopYAt(xM: number, zM: number): number | null
  sphereRadiusM(entityId: EntityId): number
  stateHash(): string
  dispose(): void
}
export class SimulationError extends Error { constructor(readonly code: 'noMotor' | 'unknownEntity', message: string) }

// sensors/sensor-host.ts
export interface SensorHost {
  subscribe(listener: SimEventListener): () => void
  readBody(entityId: EntityId): BodyState
  readonly timeS: number
  readonly gravityMps2: number
  terrainTopYAt(xM: number, zM: number): number | null
  sphereRadiusM(entityId: EntityId): number
}

// testing/catapult-test-world.ts
export function catapultTestWorld(releaseAtHingeAngleDeg?: number): WorldDoc
```

Event order inside `step()`: engine step → `contactStarted` events (sorted) → latch checks (`latchReleased`, sorted by latch id) → `stepCompleted`. Sensors in Task 9 rely on this order.

Hinge angle (D44): with `R = conj(qA)·qB` (orientation of B in A's frame) and `R0` its value at build time, `Δ = R·conj(R0)`; the angle is the twist of `Δ` about the hinge axis expressed in A's frame at build time. With axis `(0, 0, −1)` a clockwise swing (seen from +z) is positive.

- [ ] **Step 1: Shared catapult test world**

`packages/sim-core/src/testing/catapult-test-world.ts`:
```ts
import type { WorldDoc } from '@physics-lab/formats'

/** The M1 catapult geometry (same numbers as the challenge in Task 12), for engine-level tests. */
export function catapultTestWorld(releaseAtHingeAngleDeg = 90): WorldDoc {
  return {
    format: 'physics-lab/world',
    formatVersion: 1,
    gravityMps2: 9.8,
    blocks: [{ from: [0, 0, 0], to: [127, 1, 127], material: 'stone' }],
    parts: [
      { id: 'arm', type: 'beam', motion: 'dynamic', material: 'wood', layer: 'machine', positionM: [9.434, 2.034, 32], rotationDeg: [0, 0, 225], ccd: false, sizeM: [1.6, 0.1, 0.1] },
      { id: 'ball', type: 'sphere', motion: 'dynamic', material: 'stone', layer: 'projectile', positionM: [8.939, 1.539, 32], rotationDeg: [0, 0, 0], ccd: true, radiusM: 0.1 },
      { id: 'post', type: 'beam', motion: 'static', material: 'wood', layer: 'machine', positionM: [10, 1.8, 32], rotationDeg: [0, 0, 0], ccd: false, sizeM: [0.2, 1.6, 0.2] },
    ],
    joints: [
      { id: 'hold', type: 'fixed', a: 'arm', b: 'ball' },
      { id: 'pivot', type: 'hinge', a: 'post', b: 'arm', anchorM: [10, 2.6, 32], axis: [0, 0, -1], motor: { targetSpeedDegPerS: 450, gain: 10000 } },
    ],
    latches: [{ id: 'latch', type: 'angleLatch', hinge: 'pivot', releasesJoint: 'hold', releaseAtHingeAngleDeg }],
  }
}
```

- [ ] **Step 2: Write the failing tests**

`packages/sim-core/src/simulation/simulation.test.ts`:
```ts
import { beforeAll, describe, expect, it } from 'vitest'
import { createRapierEngineFactory } from '../adapters/rapier'
import type { PhysicsEngineFactory } from '../ports/physics-engine'
import { catapultTestWorld } from '../testing/catapult-test-world'
import { FIXED_STEP_S, Simulation, SimulationError, type SimEvent } from './simulation'

let factory: PhysicsEngineFactory
beforeAll(async () => {
  factory = await createRapierEngineFactory()
})

const MAX_STEPS = 2400

function stepUntil(sim: Simulation, done: () => boolean): void {
  for (let i = 0; i < MAX_STEPS && !done(); i++) sim.step()
}

describe('Simulation', () => {
  it('keeps the arm at rest until the motor is triggered', () => {
    const sim = Simulation.create(catapultTestWorld(), factory)
    expect(sim.hingeAngleDeg('pivot')).toBeCloseTo(0, 6)
    expect(sim.stepIndex).toBe(0)
    sim.dispose()
  })

  it('swings the arm clockwise (positive hinge angle) after startMotor', () => {
    const sim = Simulation.create(catapultTestWorld(), factory)
    sim.trigger('pivot', 'startMotor')
    for (let i = 0; i < 24; i++) sim.step()
    expect(sim.hingeAngleDeg('pivot')).toBeGreaterThan(5)
    expect(sim.timeS).toBeCloseTo(24 * FIXED_STEP_S, 12)
    sim.dispose()
  })

  it('releases the latch on the first step at or past the release angle and frees the ball', () => {
    const sim = Simulation.create(catapultTestWorld(90), factory)
    const events: SimEvent[] = []
    sim.subscribe((event) => events.push(event))
    sim.trigger('pivot', 'startMotor')
    stepUntil(sim, () => events.some((event) => event.type === 'latchReleased'))
    const angleAtRelease = sim.hingeAngleDeg('pivot')
    expect(angleAtRelease).toBeGreaterThanOrEqual(90)
    expect(angleAtRelease).toBeLessThan(90 + 3)
    const vyBefore = sim.readBody('ball').linearVelocityMps[1]
    for (let i = 0; i < 10; i++) sim.step()
    const vyAfter = sim.readBody('ball').linearVelocityMps[1]
    expect(vyAfter - vyBefore).toBeCloseTo(-9.8 * 10 * FIXED_STEP_S, 3)
    sim.dispose()
  })

  it('emits contact and latch events before the stepCompleted of the same step', () => {
    const sim = Simulation.create(catapultTestWorld(), factory)
    const events: SimEvent[] = []
    sim.subscribe((event) => events.push(event))
    sim.trigger('pivot', 'startMotor')
    stepUntil(sim, () => events.some((event) => event.type === 'contactStarted'))
    for (const type of ['latchReleased', 'contactStarted'] as const) {
      const index = events.findIndex((event) => event.type === type)
      const event = events[index]
      const completedIndex = events.findIndex((other) => other.type === 'stepCompleted' && other.stepIndex === event?.stepIndex)
      expect(index).toBeGreaterThanOrEqual(0)
      expect(completedIndex).toBeGreaterThan(index)
    }
    sim.dispose()
  })

  it('rejects triggering a hinge without a motor or an unknown entity', () => {
    const world = catapultTestWorld()
    const noMotor = { ...world, joints: world.joints.map((joint) => (joint.type === 'hinge' ? { ...joint, motor: null } : joint)) }
    const sim = Simulation.create(noMotor, factory)
    expect(() => sim.trigger('pivot', 'startMotor')).toThrow(SimulationError)
    expect(() => sim.trigger('ghost', 'startMotor')).toThrow(SimulationError)
    sim.dispose()
  })

  it('exposes terrain height, sphere radius and sorted part ids for sensors', () => {
    const sim = Simulation.create(catapultTestWorld(), factory)
    expect(sim.terrainTopYAt(20, 32)).toBe(1)
    expect(sim.sphereRadiusM('ball')).toBe(0.1)
    expect(sim.partIds()).toEqual(['arm', 'ball', 'post'])
    sim.dispose()
  })
})
```

`packages/sim-core/src/determinism/determinism.test.ts`:
```ts
import { beforeAll, describe, expect, it } from 'vitest'
import { createRapierEngineFactory } from '../adapters/rapier'
import type { PhysicsEngineFactory } from '../ports/physics-engine'
import { Simulation } from '../simulation/simulation'
import { catapultTestWorld } from '../testing/catapult-test-world'

export const DETERMINISM_STEPS = 600

let factory: PhysicsEngineFactory
beforeAll(async () => {
  factory = await createRapierEngineFactory()
})

function hashAfterRun(releaseAngleDeg: number): string {
  const sim = Simulation.create(catapultTestWorld(releaseAngleDeg), factory)
  sim.trigger('pivot', 'startMotor')
  for (let i = 0; i < DETERMINISM_STEPS; i++) sim.step()
  const hash = sim.stateHash()
  sim.dispose()
  return hash
}

describe('determinism (D16)', () => {
  it('produces identical state hashes for identical runs', () => {
    expect(hashAfterRun(90)).toBe(hashAfterRun(90))
  })

  it('produces a different hash when the scenario changes', () => {
    expect(hashAfterRun(80)).not.toBe(hashAfterRun(90))
  })

  it('matches the committed Node golden hash (compared against browsers in Task 20)', async () => {
    await expect(hashAfterRun(90)).toMatchFileSnapshot('./__golden__/catapult-600-steps.hash')
  })
})
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `pnpm vitest run --project sim-core simulation determinism`
Expected: FAIL — `./simulation` not found.

- [ ] **Step 4: Implement**

`packages/sim-core/src/sensors/sensor-host.ts`:
```ts
import type { EntityId } from '@physics-lab/formats'
import type { BodyState } from '../ports/physics-engine'
import type { SimEventListener } from '../simulation/simulation'

/** What a sensor may read from a running simulation (ground truth only, spec §5.5). */
export interface SensorHost {
  subscribe(listener: SimEventListener): () => void
  readBody(entityId: EntityId): BodyState
  readonly timeS: number
  readonly gravityMps2: number
  terrainTopYAt(xM: number, zM: number): number | null
  sphereRadiusM(entityId: EntityId): number
}
```

`packages/sim-core/src/simulation/simulation.ts`:
```ts
import { SnapshotHasher, degToRad } from '@physics-lab/det-math'
import { TERRAIN_ENTITY_ID, type AngleLatch, type EntityId, type HingeJoint, type Part, type Vec3, type WorldDoc } from '@physics-lab/formats'
import { angleAboutAxisDeg, conjugateQuat, multiplyQuat, quatFromEulerDeg, rotateVec3, type Quat } from '../geometry/quat'
import { normalizeVec3 } from '../geometry/vec'
import { MATERIALS } from '../materials'
import type { BodySpec, BodyState, PhysicsEngine, PhysicsEngineFactory } from '../ports/physics-engine'
import type { SensorHost } from '../sensors/sensor-host'
import { cellBoxToWorldBox, mergeStaticBlocks } from '../voxel/static-merge'
import { VoxelGrid } from '../voxel/voxel-grid'

/** Fixed simulation step (D23). Slow motion changes steps per frame, never this value. */
export const FIXED_STEP_S = 1 / 240
/** Solver substeps per step (D17). */
export const SOLVER_ITERATIONS = 8
const BRAKE_TARGET_SPEED_RAD_PER_S = 0
const ZERO_VELOCITY: Vec3 = [0, 0, 0]

export type SimEvent =
  | { type: 'contactStarted'; a: EntityId; b: EntityId; stepIndex: number }
  | { type: 'latchReleased'; latchId: EntityId; stepIndex: number }
  | { type: 'stepCompleted'; stepIndex: number }
export type SimEventListener = (event: SimEvent) => void

export class SimulationError extends Error {
  constructor(
    readonly code: 'noMotor' | 'unknownEntity',
    message: string
  ) {
    super(message)
    this.name = 'SimulationError'
  }
}

type HingeRestPose = { joint: HingeJoint; relativeRotation0: Quat; axisInA: Vec3 }

function byId<T extends { id: EntityId }>(items: readonly T[]): T[] {
  return [...items].sort((a, b) => (a.id < b.id ? -1 : a.id > b.id ? 1 : 0))
}

function toBodySpec(part: Part): BodySpec {
  return {
    entityId: part.id,
    motion: part.motion,
    shape: part.type === 'beam'
      ? { kind: 'box', halfExtentsM: [part.sizeM[0] / 2, part.sizeM[1] / 2, part.sizeM[2] / 2] }
      : { kind: 'sphere', radiusM: part.radiusM },
    positionM: part.positionM,
    rotation: quatFromEulerDeg(part.rotationDeg),
    material: MATERIALS[part.material],
    layer: part.layer,
    ccd: part.ccd,
    initialVelocityMps: part.initialVelocityMps ?? ZERO_VELOCITY,
  }
}

function relativeRotation(engine: PhysicsEngine, joint: HingeJoint): Quat {
  return multiplyQuat(conjugateQuat(engine.readBody(joint.a).rotation), engine.readBody(joint.b).rotation)
}

/** Deterministic run of a WorldDoc (spec §5). Iterates entities in id order everywhere (D16). */
export class Simulation implements SensorHost {
  private stepCount = 0
  private readonly listeners = new Set<SimEventListener>()
  private readonly releasedLatches = new Set<EntityId>()
  private readonly latches: readonly AngleLatch[]
  private readonly partsById: ReadonlyMap<EntityId, Part>

  private constructor(
    private readonly doc: WorldDoc,
    private readonly engine: PhysicsEngine,
    private readonly grid: VoxelGrid,
    private readonly hinges: ReadonlyMap<EntityId, HingeRestPose>
  ) {
    this.latches = byId(doc.latches)
    this.partsById = new Map(doc.parts.map((part) => [part.id, part]))
  }

  static create(doc: WorldDoc, factory: PhysicsEngineFactory): Simulation {
    const engine = factory.create({ gravityMps2: doc.gravityMps2, fixedStepS: FIXED_STEP_S, solverIterations: SOLVER_ITERATIONS })
    const grid = VoxelGrid.fromFills(doc.blocks)
    engine.addTerrain(
      TERRAIN_ENTITY_ID,
      mergeStaticBlocks(grid).map((box) => ({ ...cellBoxToWorldBox(box), material: MATERIALS[box.material] }))
    )
    for (const part of byId(doc.parts)) engine.addBody(toBodySpec(part))
    const hinges = new Map<EntityId, HingeRestPose>()
    for (const joint of byId(doc.joints)) {
      if (joint.type === 'fixed') {
        engine.addFixed({ jointId: joint.id, bodyA: joint.a, bodyB: joint.b })
        continue
      }
      const axisWorld = normalizeVec3(joint.axis)
      engine.addHinge({ jointId: joint.id, bodyA: joint.a, bodyB: joint.b, anchorM: joint.anchorM, axisWorld })
      const axisInA = rotateVec3(conjugateQuat(engine.readBody(joint.a).rotation), axisWorld)
      hinges.set(joint.id, { joint, relativeRotation0: relativeRotation(engine, joint), axisInA })
    }
    return new Simulation(doc, engine, grid, hinges)
  }

  get gravityMps2(): number {
    return this.doc.gravityMps2
  }

  get stepIndex(): number {
    return this.stepCount
  }

  get timeS(): number {
    return this.stepCount * FIXED_STEP_S
  }

  // Observer pattern: sensors and the UI subscribe to simulation events instead of polling.
  subscribe(listener: SimEventListener): () => void {
    this.listeners.add(listener)
    return () => this.listeners.delete(listener)
  }

  trigger(target: EntityId, action: 'startMotor'): void {
    const hinge = this.hinges.get(target)
    if (!hinge) throw new SimulationError('unknownEntity', `No hinge "${target}" for ${action}`)
    if (hinge.joint.motor === null) throw new SimulationError('noMotor', `Hinge "${target}" has no motor`)
    this.engine.setHingeMotor(target, degToRad(hinge.joint.motor.targetSpeedDegPerS), hinge.joint.motor.gain)
  }

  step(): void {
    const contacts = this.engine.step()
    this.stepCount += 1
    for (const [a, b] of contacts) this.emit({ type: 'contactStarted', a, b, stepIndex: this.stepCount })
    this.releaseLatches()
    this.emit({ type: 'stepCompleted', stepIndex: this.stepCount })
  }

  hingeAngleDeg(hingeId: EntityId): number {
    const hinge = this.hinges.get(hingeId)
    if (!hinge) throw new SimulationError('unknownEntity', `No hinge "${hingeId}"`)
    const delta = multiplyQuat(relativeRotation(this.engine, hinge.joint), conjugateQuat(hinge.relativeRotation0))
    return angleAboutAxisDeg(delta, hinge.axisInA)
  }

  readBody(entityId: EntityId): BodyState {
    return this.engine.readBody(entityId)
  }

  partIds(): readonly EntityId[] {
    return byId(this.doc.parts).map((part) => part.id)
  }

  terrainTopYAt(xM: number, zM: number): number | null {
    return this.grid.topSurfaceYM(xM, zM)
  }

  sphereRadiusM(entityId: EntityId): number {
    const part = this.partsById.get(entityId)
    if (part?.type !== 'sphere') throw new SimulationError('unknownEntity', `No sphere "${entityId}"`)
    return part.radiusM
  }

  /** FNV-1a 64 over step index and the full state of every dynamic part, in id order (D16). */
  stateHash(): string {
    const hasher = new SnapshotHasher().addUint32(this.stepCount)
    for (const part of byId(this.doc.parts)) {
      if (part.motion !== 'dynamic') continue
      const state = this.engine.readBody(part.id)
      hasher.addString(part.id)
      for (const value of [...state.positionM, ...state.rotation, ...state.linearVelocityMps, ...state.angularVelocityRadPerS]) {
        hasher.addFloat64(value)
      }
    }
    return hasher.digest()
  }

  dispose(): void {
    this.listeners.clear()
    this.engine.dispose()
  }

  private releaseLatches(): void {
    for (const latch of this.latches) {
      if (this.releasedLatches.has(latch.id)) continue
      if (this.hingeAngleDeg(latch.hinge) < latch.releaseAtHingeAngleDeg) continue
      this.engine.removeJoint(latch.releasesJoint)
      const motor = this.hinges.get(latch.hinge)?.joint.motor
      if (motor) this.engine.setHingeMotor(latch.hinge, BRAKE_TARGET_SPEED_RAD_PER_S, motor.gain)
      this.releasedLatches.add(latch.id)
      this.emit({ type: 'latchReleased', latchId: latch.id, stepIndex: this.stepCount })
    }
  }

  private emit(event: SimEvent): void {
    for (const listener of this.listeners) listener(event)
  }
}
```

Append to `packages/sim-core/src/index.ts`:
```ts
export * from './sensors/sensor-host'
export * from './simulation/simulation'
export * from './testing/catapult-test-world'
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `pnpm vitest run --project sim-core simulation determinism`
Expected: PASS; `packages/sim-core/src/determinism/__golden__/catapult-600-steps.hash` is created on the first run. Commit it. (In CI Vitest does not write missing snapshots, so a missing golden fails the build.)

- [ ] **Step 6: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test
git add packages/sim-core
git commit -m "feat(sim-core): add deterministic simulation with angle latches"
```

---

### Task 9: `sim-core` — projectile flight sensor and free-projectile validation benchmark

**Files:**
- Create: `packages/sim-core/src/sensors/projectile-flight-sensor.ts`, `testing/fake-sensor-host.ts`, `validation/free-projectile.validation.test.ts`
- Modify: `packages/sim-core/src/index.ts`
- Test: `packages/sim-core/src/sensors/projectile-flight-sensor.test.ts`

**Interfaces:**
- Consumes: `SensorHost`, `SimEvent`, `FIXED_STEP_S`, `Simulation` (Task 8); `horizontalDistanceM` (Task 5); `detAtan2`, `radToDeg` (det-math).
- Produces:
```ts
export const FLIGHT_QUANTITIES = ['releaseSpeed', 'releaseAngle', 'releaseHeight', 'range', 'time'] as const
export type FlightQuantity = (typeof FLIGHT_QUANTITIES)[number]
export function isFlightQuantity(reading: string): reading is FlightQuantity
export type ReleaseTrigger = { kind: 'latch'; latchId: EntityId } | { kind: 'start' }
export type ProjectileFlightConfig = { bodyId: EntityId; surfaceId: EntityId; release: ReleaseTrigger }
export class ProjectileFlightSensor {
  constructor(config: ProjectileFlightConfig, host: SensorHost)
  /** Ground-truth readings available so far (SI: m/s, deg, m, m, s). */
  groundTruth(): Partial<Record<FlightQuantity, number>>
  dispose(): void
}
export class FakeSensorHost implements SensorHost { /* scripted host for unit tests */ }
```

Definitions (D44): release state = body state at the end of the release step (or at construction for `start`). `releaseSpeed` = |v|; `releaseAngle` = atan2(v_y, √(v_x² + v_z²)) in degrees; `releaseHeight` = y_center − radius − terrain top under the release point (= the drop of the centre to landing on flat ground); landing = the first `contactStarted` with the surface while the body was descending in the previous step; the contact point is refined by solving `y_prev + v_y·τ − ½gτ² = terrainTop + radius` for τ ∈ [0, FIXED_STEP_S] from the pre-step state; `range` = horizontal distance release → landing centre; `time` = landing time − release time.

- [ ] **Step 1: Fake host for unit tests**

`packages/sim-core/src/testing/fake-sensor-host.ts`:
```ts
import type { EntityId, Vec3 } from '@physics-lab/formats'
import { IDENTITY_QUAT } from '../geometry/quat'
import type { BodyState } from '../ports/physics-engine'
import type { SensorHost } from '../sensors/sensor-host'
import { FIXED_STEP_S, type SimEvent, type SimEventListener } from '../simulation/simulation'

/** In-memory SensorHost: tests script body states and events step by step (fakes over mocks, D30). */
export class FakeSensorHost implements SensorHost {
  timeS = 0
  readonly gravityMps2: number
  private readonly terrainTop: number
  private readonly radius: number
  private readonly listeners = new Set<SimEventListener>()
  private readonly bodies = new Map<EntityId, BodyState>()

  constructor(options: { gravityMps2?: number; terrainTopYM?: number; radiusM?: number } = {}) {
    this.gravityMps2 = options.gravityMps2 ?? 9.8
    this.terrainTop = options.terrainTopYM ?? 1
    this.radius = options.radiusM ?? 0.1
  }

  setBody(entityId: EntityId, positionM: Vec3, linearVelocityMps: Vec3): void {
    this.bodies.set(entityId, { positionM, linearVelocityMps, rotation: IDENTITY_QUAT, angularVelocityRadPerS: [0, 0, 0] })
  }

  /** Advances time by one step and emits the given events followed by stepCompleted. */
  completeStep(stepIndex: number, events: readonly SimEvent[] = []): void {
    this.timeS = stepIndex * FIXED_STEP_S
    for (const event of events) this.emit(event)
    this.emit({ type: 'stepCompleted', stepIndex })
  }

  subscribe(listener: SimEventListener): () => void {
    this.listeners.add(listener)
    return () => this.listeners.delete(listener)
  }

  readBody(entityId: EntityId): BodyState {
    const body = this.bodies.get(entityId)
    if (!body) throw new Error(`Unknown body "${entityId}"`)
    return body
  }

  terrainTopYAt(): number {
    return this.terrainTop
  }

  sphereRadiusM(): number {
    return this.radius
  }

  private emit(event: SimEvent): void {
    for (const listener of this.listeners) listener(event)
  }
}
```

- [ ] **Step 2: Write the failing tests**

`packages/sim-core/src/sensors/projectile-flight-sensor.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { FIXED_STEP_S } from '../simulation/simulation'
import { FakeSensorHost } from '../testing/fake-sensor-host'
import { ProjectileFlightSensor } from './projectile-flight-sensor'

const config = { bodyId: 'ball', surfaceId: 'terrain', release: { kind: 'latch', latchId: 'latch' } } as const

describe('ProjectileFlightSensor', () => {
  it('has no readings before release', () => {
    const host = new FakeSensorHost()
    host.setBody('ball', [0, 3, 0], [0, 0, 0])
    expect(new ProjectileFlightSensor(config, host).groundTruth()).toEqual({})
  })

  it('records the release state when its latch releases, ignoring other latches', () => {
    const host = new FakeSensorHost()
    host.setBody('ball', [9, 3.1, 32], [6, 8, 0])
    const sensor = new ProjectileFlightSensor(config, host)
    host.completeStep(10, [{ type: 'latchReleased', latchId: 'other', stepIndex: 10 }])
    expect(sensor.groundTruth()).toEqual({})
    host.completeStep(11, [{ type: 'latchReleased', latchId: 'latch', stepIndex: 11 }])
    const readings = sensor.groundTruth()
    expect(readings.releaseSpeed).toBeCloseTo(10, 12)
    expect(readings.releaseAngle).toBeCloseTo(53.13010235415598, 10)
    expect(readings.releaseHeight).toBeCloseTo(3.1 - 0.1 - 1, 12)
    expect(readings.range).toBeUndefined()
  })

  it('ignores contacts before release and while ascending', () => {
    const host = new FakeSensorHost()
    host.setBody('ball', [0, 1.1, 0], [5, 5, 0])
    const sensor = new ProjectileFlightSensor({ ...config, release: { kind: 'start' } }, host)
    host.completeStep(1, [{ type: 'contactStarted', a: 'ball', b: 'terrain', stepIndex: 1 }])
    expect(sensor.groundTruth().range).toBeUndefined()
  })

  it('refines the landing point inside the contact step', () => {
    const host = new FakeSensorHost()
    host.setBody('ball', [0, 3, 0], [10, 0, 0])
    const sensor = new ProjectileFlightSensor({ ...config, release: { kind: 'start' } }, host)
    host.setBody('ball', [10, 1.12, 0], [10, -6, 0])
    host.completeStep(100)
    host.setBody('ball', [10.04, 1.1, 0], [10, -1, 0])
    host.completeStep(101, [{ type: 'contactStarted', a: 'ball', b: 'terrain', stepIndex: 101 }])
    const tau = (-6 + Math.sqrt(36 + 2 * 9.8 * 0.02)) / 9.8
    const readings = sensor.groundTruth()
    expect(readings.range).toBeCloseTo(10 + 10 * tau, 12)
    expect(readings.time).toBeCloseTo(100 * FIXED_STEP_S + tau, 12)
  })

  it('keeps the first landing and ignores later bounces', () => {
    const host = new FakeSensorHost()
    host.setBody('ball', [0, 3, 0], [10, 0, 0])
    const sensor = new ProjectileFlightSensor({ ...config, release: { kind: 'start' } }, host)
    host.setBody('ball', [10, 1.12, 0], [10, -6, 0])
    host.completeStep(100)
    host.completeStep(101, [{ type: 'contactStarted', a: 'ball', b: 'terrain', stepIndex: 101 }])
    const first = sensor.groundTruth().range
    host.setBody('ball', [14, 1.15, 0], [9, -3, 0])
    host.completeStep(150)
    host.completeStep(151, [{ type: 'contactStarted', a: 'ball', b: 'terrain', stepIndex: 151 }])
    expect(sensor.groundTruth().range).toBe(first)
  })
})
```

`packages/sim-core/src/validation/free-projectile.validation.test.ts`:
```ts
import type { WorldDoc } from '@physics-lab/formats'
import { beforeAll, describe, expect, it } from 'vitest'
import { createRapierEngineFactory } from '../adapters/rapier'
import type { PhysicsEngineFactory } from '../ports/physics-engine'
import { ProjectileFlightSensor } from '../sensors/projectile-flight-sensor'
import { Simulation } from '../simulation/simulation'

/** Spec §8 row 1: range of a free projectile vs v²·sin2θ/g, ≤ 0.5 % relative. */
const FREE_PROJECTILE_RANGE_TOLERANCE = 0.005
const GRAVITY_MPS2 = 9.8
const MAX_STEPS = 240 * 10
const GROUND_TOP_M = 1
const RADIUS_M = 0.1

/*
 * Sweep restricted to v·sinθ ≥ 5 m/s: a semi-implicit Euler step biases the vertical launch speed by
 * about g·dt/2 (0.02 m/s at 1/240 s), i.e. a relative range error of ~0.02/(v·sinθ). Slower lobs are
 * outside the catapult's operating range (Task 12) and would need a log entry to validate (spec §8).
 */
const SPEEDS_MPS = [10, 15, 20]
const ANGLES_DEG = [30, 45, 60]

function freeProjectileWorld(speedMps: number, angleDeg: number): WorldDoc {
  const angle = (angleDeg * Math.PI) / 180
  return {
    format: 'physics-lab/world',
    formatVersion: 1,
    gravityMps2: GRAVITY_MPS2,
    blocks: [{ from: [0, 0, 0], to: [127, 1, 127], material: 'stone' }],
    parts: [
      {
        id: 'ball', type: 'sphere', motion: 'dynamic', material: 'stone', layer: 'projectile',
        positionM: [8, GROUND_TOP_M + RADIUS_M, 32], rotationDeg: [0, 0, 0], ccd: true, radiusM: RADIUS_M,
        initialVelocityMps: [speedMps * Math.cos(angle), speedMps * Math.sin(angle), 0],
      },
    ],
    joints: [],
    latches: [],
  }
}

let factory: PhysicsEngineFactory
beforeAll(async () => {
  factory = await createRapierEngineFactory()
})

describe('engine validation: free projectile range (spec §8)', () => {
  const cases = SPEEDS_MPS.flatMap((speed) => ANGLES_DEG.map((angle) => [speed, angle] as const))

  it.each(cases)('v = %d m/s, θ = %d° lands within 0.5 %% of v²·sin2θ/g', (speedMps, angleDeg) => {
    const sim = Simulation.create(freeProjectileWorld(speedMps, angleDeg), factory)
    const sensor = new ProjectileFlightSensor({ bodyId: 'ball', surfaceId: 'terrain', release: { kind: 'start' } }, sim)
    for (let i = 0; i < MAX_STEPS && sensor.groundTruth().range === undefined; i++) sim.step()
    const measured = sensor.groundTruth().range
    const expected = (speedMps * speedMps * Math.sin((2 * angleDeg * Math.PI) / 180)) / GRAVITY_MPS2
    expect(measured).toBeDefined()
    expect(Math.abs((measured ?? 0) - expected) / expected).toBeLessThanOrEqual(FREE_PROJECTILE_RANGE_TOLERANCE)
    sensor.dispose()
    sim.dispose()
  })
})
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `pnpm vitest run --project sim-core flight free-projectile`
Expected: FAIL — `./projectile-flight-sensor` not found.

- [ ] **Step 4: Implement**

`packages/sim-core/src/sensors/projectile-flight-sensor.ts`:
```ts
import { detAtan2, radToDeg } from '@physics-lab/det-math'
import type { EntityId, Vec3 } from '@physics-lab/formats'
import { horizontalDistanceM } from '../geometry/vec'
import { FIXED_STEP_S, type SimEvent } from '../simulation/simulation'
import type { SensorHost } from './sensor-host'

export const FLIGHT_QUANTITIES = ['releaseSpeed', 'releaseAngle', 'releaseHeight', 'range', 'time'] as const
export type FlightQuantity = (typeof FLIGHT_QUANTITIES)[number]

export function isFlightQuantity(reading: string): reading is FlightQuantity {
  return (FLIGHT_QUANTITIES as readonly string[]).includes(reading)
}

export type ReleaseTrigger = { kind: 'latch'; latchId: EntityId } | { kind: 'start' }
export type ProjectileFlightConfig = { bodyId: EntityId; surfaceId: EntityId; release: ReleaseTrigger }

type Snapshot = { positionM: Vec3; velocityMps: Vec3; timeS: number }
type Landing = { positionM: Vec3; timeS: number }

/** Ground-truth projectile flight: release state and refined first landing (spec §5.5, D44). */
export class ProjectileFlightSensor {
  private release: Snapshot | null = null
  private landing: Landing | null = null
  private previous: Snapshot
  private readonly unsubscribe: () => void

  constructor(
    private readonly config: ProjectileFlightConfig,
    private readonly host: SensorHost
  ) {
    this.previous = this.snapshot()
    if (config.release.kind === 'start') this.release = this.previous
    this.unsubscribe = host.subscribe((event) => this.onEvent(event))
  }

  groundTruth(): Partial<Record<FlightQuantity, number>> {
    const release = this.release
    if (release === null) return {}
    const [vx, vy, vz] = release.velocityMps
    const horizontalSpeed = Math.sqrt(vx * vx + vz * vz)
    const readings: Partial<Record<FlightQuantity, number>> = {
      releaseSpeed: Math.sqrt(vx * vx + vy * vy + vz * vz),
      releaseAngle: radToDeg(detAtan2(vy, horizontalSpeed)),
      releaseHeight: release.positionM[1] - this.radius() - this.surfaceTopUnder(release.positionM),
    }
    if (this.landing !== null) {
      readings.range = horizontalDistanceM(release.positionM, this.landing.positionM)
      readings.time = this.landing.timeS - release.timeS
    }
    return readings
  }

  dispose(): void {
    this.unsubscribe()
  }

  private onEvent(event: SimEvent): void {
    switch (event.type) {
      case 'latchReleased':
        if (this.config.release.kind === 'latch' && event.latchId === this.config.release.latchId && this.release === null) {
          this.release = this.snapshot()
        }
        return
      case 'contactStarted':
        if (this.involvesBodyAndSurface(event.a, event.b)) this.recordLanding()
        return
      case 'stepCompleted':
        this.previous = this.snapshot()
        return
    }
  }

  private involvesBodyAndSurface(a: EntityId, b: EntityId): boolean {
    const { bodyId, surfaceId } = this.config
    return (a === bodyId && b === surfaceId) || (a === surfaceId && b === bodyId)
  }

  private recordLanding(): void {
    if (this.release === null || this.landing !== null) return
    if (this.previous.velocityMps[1] >= 0) return
    this.landing = this.refineLanding()
  }

  /** Free flight inside the step: solve y_prev + v_y·τ − ½gτ² = landing height for τ ∈ [0, dt]. */
  private refineLanding(): Landing {
    const { positionM: p, velocityMps: v, timeS } = this.previous
    const g = this.host.gravityMps2
    const landingY = this.surfaceTopUnder(p) + this.radius()
    const discriminant = v[1] * v[1] + 2 * g * (p[1] - landingY)
    const rawTau = discriminant >= 0 ? (v[1] + Math.sqrt(discriminant)) / g : 0
    const tau = Math.min(Math.max(rawTau, 0), FIXED_STEP_S)
    return { positionM: [p[0] + v[0] * tau, landingY, p[2] + v[2] * tau], timeS: timeS + tau }
  }

  private surfaceTopUnder(positionM: Vec3): number {
    return this.host.terrainTopYAt(positionM[0], positionM[2]) ?? 0
  }

  private radius(): number {
    return this.host.sphereRadiusM(this.config.bodyId)
  }

  private snapshot(): Snapshot {
    const state = this.host.readBody(this.config.bodyId)
    return { positionM: state.positionM, velocityMps: state.linearVelocityMps, timeS: this.host.timeS }
  }
}
```

Append to `packages/sim-core/src/index.ts`:
```ts
export * from './sensors/projectile-flight-sensor'
export * from './testing/fake-sensor-host'
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `pnpm vitest run --project sim-core flight free-projectile`
Expected: PASS (5 sensor tests, 9 benchmark cases). **If a benchmark case fails, do not change the tolerance or the sweep**: record the measured relative errors and hand them to `physics-reviewer` (candidate causes: integrator bias, CCD clamping — spec §13). A tolerance change needs a decisions-log entry by the main session.

- [ ] **Step 6: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test
git add packages/sim-core
git commit -m "feat(sim-core): add projectile flight sensor and range benchmark"
```

---
### Task 10: `instruments` — resolution, seeded noise and uncertainty

**Files:**
- Create: `packages/instruments/src/instrument-specs.ts`, `gaussian.ts`, `measure.ts`, `derived.ts`
- Modify: `packages/instruments/src/index.ts`
- Test: `packages/instruments/src/gaussian.test.ts`, `measure.test.ts`, `derived.test.ts`

**Interfaces:**
- Consumes: `Measurement`, `InstrumentId`, `QuantityId`, `Unit`, `InstrumentMode` (formats); `RandomSource`, `detLn` (det-math).
- Produces:
  - `type NoiseModel = { kind: 'absolute'; sigma: number } | { kind: 'relative'; fraction: number }`
  - `type InstrumentSpec = { resolution: number; resolutionDecimals: number; noise: NoiseModel; unit: Unit }`, `INSTRUMENT_SPECS: Readonly<Record<InstrumentId, InstrumentSpec>>` (D34)
  - `standardNormal(random: RandomSource): number`
  - `type MeasureRequest = { quantity: QuantityId; instrument: InstrumentId; trueValue: number; mode: InstrumentMode }`, `measure(request, random): Measurement`, `roundToResolution(value, spec): number`
  - `photogateTransitTimeS(gateWidthM, speedMps): number`, `deriveSpeedFromGate(gateWidthM: number, transit: Measurement): Measurement` (value `w/t`, uncertainty `v·u_t/t`, unit `m/s`, instrument `photogate`)

Rules (spec §6): `realistic` adds seeded Gaussian noise with σ from the table (relative σ uses |true value|), then rounds to the resolution; reported `u = sqrt((resolution/√12)² + σ²)` with σ evaluated on the reading. `ideal` rounds only, `u = resolution/√12`, and consumes no random numbers.

- [ ] **Step 1: Write the failing tests**

`packages/instruments/src/gaussian.test.ts`:
```ts
import { SeededRandom } from '@physics-lab/det-math'
import { describe, expect, it } from 'vitest'
import { standardNormal } from './gaussian'

describe('standardNormal', () => {
  it('has mean ≈ 0 and standard deviation ≈ 1 over 20 000 seeded draws', () => {
    const random = SeededRandom.fromSeed(11)
    const draws = Array.from({ length: 20_000 }, () => standardNormal(random))
    const mean = draws.reduce((sum, x) => sum + x, 0) / draws.length
    const variance = draws.reduce((sum, x) => sum + (x - mean) * (x - mean), 0) / (draws.length - 1)
    expect(Math.abs(mean)).toBeLessThan(0.03)
    expect(Math.sqrt(variance)).toBeGreaterThan(0.98)
    expect(Math.sqrt(variance)).toBeLessThan(1.02)
  })

  it('puts about 68 % of draws within one sigma', () => {
    const random = SeededRandom.fromSeed(12)
    const draws = Array.from({ length: 20_000 }, () => standardNormal(random))
    const inside = draws.filter((x) => Math.abs(x) <= 1).length / draws.length
    expect(inside).toBeGreaterThan(0.67)
    expect(inside).toBeLessThan(0.70)
  })

  it('is reproducible for a seed', () => {
    const a = SeededRandom.fromSeed(5)
    const b = SeededRandom.fromSeed(5)
    expect(Array.from({ length: 10 }, () => standardNormal(a))).toEqual(Array.from({ length: 10 }, () => standardNormal(b)))
  })
})
```

`packages/instruments/src/measure.test.ts`:
```ts
import { SeededRandom, type RandomSource } from '@physics-lab/det-math'
import { describe, expect, it } from 'vitest'
import { INSTRUMENT_SPECS } from './instrument-specs'
import { measure, roundToResolution } from './measure'

const neverCalled: RandomSource = {
  nextUint32: () => {
    throw new Error('ideal mode must not draw random numbers')
  },
  nextFloat: () => {
    throw new Error('ideal mode must not draw random numbers')
  },
}

describe('INSTRUMENT_SPECS (D34)', () => {
  it('matches the approved resolution and noise table', () => {
    expect(INSTRUMENT_SPECS.ruler).toMatchObject({ resolution: 0.01, noise: { kind: 'absolute', sigma: 0.005 }, unit: 'm' })
    expect(INSTRUMENT_SPECS.stopwatch).toMatchObject({ resolution: 0.01, noise: { kind: 'absolute', sigma: 0.1 }, unit: 's' })
    expect(INSTRUMENT_SPECS.photogate).toMatchObject({ resolution: 0.001, noise: { kind: 'absolute', sigma: 0.0005 }, unit: 's' })
    expect(INSTRUMENT_SPECS.protractor).toMatchObject({ resolution: 0.5, noise: { kind: 'absolute', sigma: 0.25 }, unit: 'deg' })
    expect(INSTRUMENT_SPECS.forceProbe).toMatchObject({ resolution: 0.01, noise: { kind: 'relative', fraction: 0.01 }, unit: 'N' })
    expect(INSTRUMENT_SPECS.massScale).toMatchObject({ resolution: 0.001, noise: { kind: 'absolute', sigma: 0.0005 }, unit: 'kg' })
  })
})

describe('measure', () => {
  it('rounds to the resolution without noise in ideal mode', () => {
    const m = measure({ quantity: 'flight.range', instrument: 'ruler', trueValue: 12.34567, mode: 'ideal' }, neverCalled)
    expect(m).toEqual({ quantity: 'flight.range', instrument: 'ruler', value: 12.35, uncertainty: 0.01 / Math.sqrt(12), unit: 'm' })
  })

  it('rounds protractor readings to 0.5°', () => {
    expect(roundToResolution(44.74, INSTRUMENT_SPECS.protractor)).toBe(44.5)
    expect(roundToResolution(44.76, INSTRUMENT_SPECS.protractor)).toBe(45)
  })

  it('reports u = sqrt((res/√12)² + σ²) in realistic mode', () => {
    const m = measure({ quantity: 'flight.range', instrument: 'ruler', trueValue: 5, mode: 'realistic' }, SeededRandom.fromSeed(1))
    expect(m.uncertainty).toBeCloseTo(Math.sqrt((0.01 * 0.01) / 12 + 0.005 * 0.005), 15)
  })

  it('produces realistic ruler readings with the expected spread', () => {
    const random = SeededRandom.fromSeed(2)
    const values = Array.from({ length: 10_000 }, () => measure({ quantity: 'flight.range', instrument: 'ruler', trueValue: 5, mode: 'realistic' }, random).value)
    const mean = values.reduce((sum, x) => sum + x, 0) / values.length
    const sd = Math.sqrt(values.reduce((sum, x) => sum + (x - mean) * (x - mean), 0) / (values.length - 1))
    expect(Math.abs(mean - 5)).toBeLessThan(0.0005)
    expect(sd).toBeGreaterThan(0.95 * Math.sqrt(0.005 * 0.005 + 0.0001 / 12))
    expect(sd).toBeLessThan(1.05 * Math.sqrt(0.005 * 0.005 + 0.0001 / 12))
    for (const value of values) expect(Math.round(value * 100) / 100).toBe(value)
  })

  it('scales force-probe noise with the reading', () => {
    const m = measure({ quantity: 'probe.force', instrument: 'forceProbe', trueValue: 100, mode: 'realistic' }, SeededRandom.fromSeed(3))
    expect(m.uncertainty).toBeCloseTo(Math.sqrt((0.01 * 0.01) / 12 + (0.01 * Math.abs(m.value)) * (0.01 * Math.abs(m.value))), 12)
  })

  it('reproduces the same readings from the same seed', () => {
    const request = { quantity: 'flight.time', instrument: 'stopwatch', trueValue: 2.2, mode: 'realistic' } as const
    expect(measure(request, SeededRandom.fromSeed(9))).toEqual(measure(request, SeededRandom.fromSeed(9)))
  })
})
```

`packages/instruments/src/derived.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { deriveSpeedFromGate, photogateTransitTimeS } from './derived'

describe('photogate speed', () => {
  it('converts speed to transit time over the gate width', () => {
    expect(photogateTransitTimeS(0.2, 10)).toBeCloseTo(0.02, 15)
  })

  it('derives speed and propagates the relative time uncertainty', () => {
    const speed = deriveSpeedFromGate(0.2, { quantity: 'flight.releaseSpeed', instrument: 'photogate', value: 0.02, uncertainty: 0.0006, unit: 's' })
    expect(speed).toEqual({ quantity: 'flight.releaseSpeed', instrument: 'photogate', value: 10, uncertainty: 10 * (0.0006 / 0.02), unit: 'm/s' })
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run --project instruments`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement**

`packages/instruments/src/instrument-specs.ts`:
```ts
import type { InstrumentId, Unit } from '@physics-lab/formats'

export type NoiseModel = { kind: 'absolute'; sigma: number } | { kind: 'relative'; fraction: number }
export type InstrumentSpec = { resolution: number; resolutionDecimals: number; noise: NoiseModel; unit: Unit }

/** Resolutions and realistic-mode noise (1σ) approved in D34 (spec §6). */
export const INSTRUMENT_SPECS: Readonly<Record<InstrumentId, InstrumentSpec>> = {
  ruler: { resolution: 0.01, resolutionDecimals: 2, noise: { kind: 'absolute', sigma: 0.005 }, unit: 'm' },
  stopwatch: { resolution: 0.01, resolutionDecimals: 2, noise: { kind: 'absolute', sigma: 0.1 }, unit: 's' },
  photogate: { resolution: 0.001, resolutionDecimals: 3, noise: { kind: 'absolute', sigma: 0.0005 }, unit: 's' },
  protractor: { resolution: 0.5, resolutionDecimals: 1, noise: { kind: 'absolute', sigma: 0.25 }, unit: 'deg' },
  forceProbe: { resolution: 0.01, resolutionDecimals: 2, noise: { kind: 'relative', fraction: 0.01 }, unit: 'N' },
  massScale: { resolution: 0.001, resolutionDecimals: 3, noise: { kind: 'absolute', sigma: 0.0005 }, unit: 'kg' },
}

/** σ of a noise model for a given reading. */
export function noiseSigma(model: NoiseModel, reading: number): number {
  return model.kind === 'absolute' ? model.sigma : model.fraction * Math.abs(reading)
}
```

`packages/instruments/src/gaussian.ts`:
```ts
import { detLn, type RandomSource } from '@physics-lab/det-math'

/** Standard normal draw via the Marsaglia polar method: only ln and sqrt, both deterministic (D16). */
export function standardNormal(random: RandomSource): number {
  for (;;) {
    const u = 2 * random.nextFloat() - 1
    const v = 2 * random.nextFloat() - 1
    const s = u * u + v * v
    if (s > 0 && s < 1) return u * Math.sqrt((-2 * detLn(s)) / s)
  }
}
```

`packages/instruments/src/measure.ts`:
```ts
import type { RandomSource } from '@physics-lab/det-math'
import type { InstrumentId, InstrumentMode, Measurement, QuantityId } from '@physics-lab/formats'
import { standardNormal } from './gaussian'
import { INSTRUMENT_SPECS, noiseSigma, type InstrumentSpec } from './instrument-specs'

const SQRT_12 = Math.sqrt(12)

export type MeasureRequest = { quantity: QuantityId; instrument: InstrumentId; trueValue: number; mode: InstrumentMode }

/** Nearest multiple of the resolution, printed exactly (toFixed is specified bit-for-bit by ECMAScript). */
export function roundToResolution(value: number, spec: InstrumentSpec): number {
  return Number((Math.round(value / spec.resolution) * spec.resolution).toFixed(spec.resolutionDecimals))
}

/** Simulated reading of a ground-truth value (spec §6). Ideal mode consumes no random numbers. */
export function measure(request: MeasureRequest, random: RandomSource): Measurement {
  const spec = INSTRUMENT_SPECS[request.instrument]
  const realistic = request.mode === 'realistic'
  const trueSigma = realistic ? noiseSigma(spec.noise, request.trueValue) : 0
  const noisy = trueSigma > 0 ? request.trueValue + trueSigma * standardNormal(random) : request.trueValue
  const value = roundToResolution(noisy, spec)
  const readingSigma = realistic ? noiseSigma(spec.noise, value) : 0
  const resolutionTerm = spec.resolution / SQRT_12
  return {
    quantity: request.quantity,
    instrument: request.instrument,
    value,
    uncertainty: Math.sqrt(resolutionTerm * resolutionTerm + readingSigma * readingSigma),
    unit: spec.unit,
  }
}
```

`packages/instruments/src/derived.ts`:
```ts
import type { Measurement } from '@physics-lab/formats'

export function photogateTransitTimeS(gateWidthM: number, speedMps: number): number {
  return gateWidthM / speedMps
}

/** Speed from a photogate transit time; relative uncertainty of t carries over to v = w/t (w is exact). */
export function deriveSpeedFromGate(gateWidthM: number, transit: Measurement): Measurement {
  const speed = gateWidthM / transit.value
  return {
    quantity: transit.quantity,
    instrument: 'photogate',
    value: speed,
    uncertainty: speed * (transit.uncertainty / transit.value),
    unit: 'm/s',
  }
}
```

`packages/instruments/src/index.ts`:
```ts
export { deriveSpeedFromGate, photogateTransitTimeS } from './derived'
export { standardNormal } from './gaussian'
export { INSTRUMENT_SPECS, noiseSigma, type InstrumentSpec, type NoiseModel } from './instrument-specs'
export { measure, roundToResolution, type MeasureRequest } from './measure'
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm vitest run --project instruments`
Expected: PASS.

- [ ] **Step 5: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test
git add packages/instruments
git commit -m "feat(instruments): add seeded noise, resolution rounding and uncertainty"
```

---

### Task 11: `challenges` — solver registry, projectile solver, uncertainty propagation, comparison

**Files:**
- Create: `packages/challenges/src/solvers/solver.ts`, `solvers/registry.ts`, `solvers/projectile-range.ts`, `solvers/uncertainty.ts`, `comparison.ts`
- Modify: `packages/challenges/src/index.ts`
- Test: `packages/challenges/src/solvers/projectile-range.test.ts`, `solvers/uncertainty.test.ts`, `comparison.test.ts`, `solvers/registry.test.ts`

**Interfaces:**
- Consumes: `Measurement` (formats); `detSin`, `detCos`, `degToRad` (det-math).
- Produces:
```ts
export interface Solver {
  readonly id: string
  readonly inputNames: readonly string[]
  /** Relative model tolerance from spec §8; enters u_c (D42). */
  readonly relativeModelTolerance: number
  solve(inputs: Readonly<Record<string, number>>, gravityMps2: number): Readonly<Record<string, number>>
}
export class SolverInputError extends Error {}
export type SolverRegistry = { get(id: string): Solver }
export function createSolverRegistry(solvers: readonly Solver[]): SolverRegistry
export const BUILT_IN_SOLVERS: readonly Solver[]
export const PROJECTILE_RANGE_SOLVER_ID = 'projectile.range.v1'
export const projectileRangeSolver: Solver   // inputs speed (m/s), angle (deg), height (m); outputs range (m), time (s)
export type ValueWithUncertainty = { value: number; uncertainty: number }
export function propagateUncertainty(solver: Solver, output: string, inputs: Readonly<Record<string, ValueWithUncertainty>>, gravityMps2: number): ValueWithUncertainty
export type Comparison = {
  taskId: string; prediction: number; measurement: Measurement; theory: ValueWithUncertainty
  combinedUncertainty: number; coverage: number; passed: boolean
}
export function comparePrediction(input: { taskId: string; prediction: number; measurement: Measurement; theory: ValueWithUncertainty; relativeModelTolerance: number; coverage: number }): Comparison
```

Projectile model (spec §8, no drag): `v_x = v cosθ`, `v_y = v sinθ`, `t = (v_y + sqrt(v_y² + 2gh)) / g`, `range = v_x·t`, where `h` is the drop of the ball centre from release to landing.

- [ ] **Step 1: Write the failing tests**

`packages/challenges/src/solvers/projectile-range.test.ts`:
```ts
import fc from 'fast-check'
import { describe, expect, it } from 'vitest'
import { SolverInputError } from './solver'
import { projectileRangeSolver } from './projectile-range'

const G = 9.8

describe('projectile.range.v1', () => {
  it('matches v²·sin2θ/g for launches from landing height (property)', () => {
    fc.assert(
      fc.property(fc.double({ min: 1, max: 50, noNaN: true }), fc.double({ min: 5, max: 85, noNaN: true }), (speed, angle) => {
        const { range } = projectileRangeSolver.solve({ speed, angle, height: 0 }, G)
        const expected = (speed * speed * Math.sin((2 * angle * Math.PI) / 180)) / G
        expect(Math.abs((range ?? 0) - expected)).toBeLessThanOrEqual(1e-12 * Math.max(1, expected))
      })
    )
  })

  it('travels farther when released higher (property)', () => {
    fc.assert(
      fc.property(fc.double({ min: 1, max: 50, noNaN: true }), fc.double({ min: 5, max: 85, noNaN: true }), fc.double({ min: 0.01, max: 20, noNaN: true }), (speed, angle, height) => {
        const low = projectileRangeSolver.solve({ speed, angle, height: 0 }, G).range ?? 0
        const high = projectileRangeSolver.solve({ speed, angle, height }, G).range ?? 0
        expect(high).toBeGreaterThan(low)
      })
    )
  })

  it('returns the flight time', () => {
    const { time } = projectileRangeSolver.solve({ speed: 9.8, angle: 90, height: 0 }, G)
    expect(time).toBeCloseTo(2, 12)
  })

  it('rejects missing inputs and impossible landings', () => {
    expect(() => projectileRangeSolver.solve({ speed: 10, angle: 45 }, G)).toThrow(SolverInputError)
    expect(() => projectileRangeSolver.solve({ speed: 1, angle: -80, height: -10 }, G)).toThrow(SolverInputError)
  })
})
```

`packages/challenges/src/solvers/uncertainty.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import type { Solver } from './solver'
import { propagateUncertainty } from './uncertainty'

const linear: Solver = {
  id: 'test.linear',
  inputNames: ['a', 'b'],
  relativeModelTolerance: 0,
  solve: (inputs) => ({ y: 2 * (inputs['a'] ?? 0) + 3 * (inputs['b'] ?? 0) }),
}

describe('propagateUncertainty', () => {
  it('is exact for a linear model: u_y = sqrt((2u_a)² + (3u_b)²)', () => {
    const result = propagateUncertainty(linear, 'y', { a: { value: 1, uncertainty: 0.1 }, b: { value: 2, uncertainty: 0.2 } }, 9.8)
    expect(result.value).toBe(8)
    expect(result.uncertainty).toBeCloseTo(Math.sqrt(0.2 * 0.2 + 0.6 * 0.6), 12)
  })

  it('ignores exact inputs', () => {
    const result = propagateUncertainty(linear, 'y', { a: { value: 1, uncertainty: 0 }, b: { value: 2, uncertainty: 0 } }, 9.8)
    expect(result.uncertainty).toBe(0)
  })

  it('throws when an input is missing', () => {
    expect(() => propagateUncertainty(linear, 'y', { a: { value: 1, uncertainty: 0.1 } }, 9.8)).toThrow()
  })
})
```

`packages/challenges/src/comparison.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { comparePrediction } from './comparison'

const measurement = { quantity: 'flight.range', instrument: 'ruler', value: 15, uncertainty: 0.1, unit: 'm' } as const

describe('comparePrediction (D42)', () => {
  it('combines measurement, theory and model uncertainty', () => {
    const c = comparePrediction({ taskId: 't', prediction: 15, measurement, theory: { value: 15.2, uncertainty: 0.3 }, relativeModelTolerance: 0.005, coverage: 2 })
    expect(c.combinedUncertainty).toBeCloseTo(Math.sqrt(0.01 + 0.09 + (0.005 * 15.2) * (0.005 * 15.2)), 12)
  })

  it('passes exactly at k·u_c and fails just beyond it', () => {
    const base = { taskId: 't', measurement, theory: { value: 15, uncertainty: 0 }, relativeModelTolerance: 0, coverage: 2 }
    expect(comparePrediction({ ...base, prediction: 15.2 }).passed).toBe(true)
    expect(comparePrediction({ ...base, prediction: 15.2001 }).passed).toBe(false)
  })
})
```

`packages/challenges/src/solvers/registry.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { BUILT_IN_SOLVERS, createSolverRegistry } from './registry'

describe('solver registry', () => {
  it('finds built-in solvers by id', () => {
    expect(createSolverRegistry(BUILT_IN_SOLVERS).get('projectile.range.v1').id).toBe('projectile.range.v1')
  })

  it('throws for unknown ids and rejects duplicate registrations', () => {
    expect(() => createSolverRegistry(BUILT_IN_SOLVERS).get('nope')).toThrow()
    expect(() => createSolverRegistry([...BUILT_IN_SOLVERS, ...BUILT_IN_SOLVERS])).toThrow()
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run --project challenges`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement**

`packages/challenges/src/solvers/solver.ts`:
```ts
/** Strategy pattern: each analytic model is a Solver; challenges reference it by id (spec §7.1). */
export interface Solver {
  readonly id: string
  readonly inputNames: readonly string[]
  /** Relative model tolerance from spec §8; enters the combined uncertainty (D42). */
  readonly relativeModelTolerance: number
  solve(inputs: Readonly<Record<string, number>>, gravityMps2: number): Readonly<Record<string, number>>
}

export class SolverInputError extends Error {
  override name = 'SolverInputError'
}

export function requireInput(inputs: Readonly<Record<string, number>>, name: string): number {
  const value = inputs[name]
  if (value === undefined || !Number.isFinite(value)) throw new SolverInputError(`Missing or invalid input "${name}"`)
  return value
}

export function requireOutput(outputs: Readonly<Record<string, number>>, name: string): number {
  const value = outputs[name]
  if (value === undefined) throw new SolverInputError(`Solver has no output "${name}"`)
  return value
}
```

`packages/challenges/src/solvers/projectile-range.ts`:
```ts
import { degToRad, detCos, detSin } from '@physics-lab/det-math'
import { SolverInputError, requireInput, type Solver } from './solver'

export const PROJECTILE_RANGE_SOLVER_ID = 'projectile.range.v1'
/** Spec §8, challenge 1 tolerance. */
const PROJECTILE_RANGE_TOLERANCE = 0.005

/** Range and time from the release state, no drag (spec §8). `height` is the drop of the centre to landing. */
export const projectileRangeSolver: Solver = {
  id: PROJECTILE_RANGE_SOLVER_ID,
  inputNames: ['speed', 'angle', 'height'],
  relativeModelTolerance: PROJECTILE_RANGE_TOLERANCE,
  solve(inputs, gravityMps2) {
    const speed = requireInput(inputs, 'speed')
    const angleRad = degToRad(requireInput(inputs, 'angle'))
    const height = requireInput(inputs, 'height')
    const vx = speed * detCos(angleRad)
    const vy = speed * detSin(angleRad)
    const discriminant = vy * vy + 2 * gravityMps2 * height
    if (discriminant < 0) throw new SolverInputError('The projectile never reaches the landing height')
    const time = (vy + Math.sqrt(discriminant)) / gravityMps2
    if (time <= 0) throw new SolverInputError('The projectile never reaches the landing height')
    return { range: vx * time, time }
  },
}
```

`packages/challenges/src/solvers/registry.ts`:
```ts
import { projectileRangeSolver } from './projectile-range'
import type { Solver } from './solver'

export type SolverRegistry = { get(id: string): Solver }

/** Registry pattern: new solvers are added to BUILT_IN_SOLVERS; callers look them up by id (open/closed). */
export const BUILT_IN_SOLVERS: readonly Solver[] = [projectileRangeSolver]

export function createSolverRegistry(solvers: readonly Solver[]): SolverRegistry {
  const byId = new Map<string, Solver>()
  for (const solver of solvers) {
    if (byId.has(solver.id)) throw new Error(`Duplicate solver "${solver.id}"`)
    byId.set(solver.id, solver)
  }
  return {
    get(id) {
      const solver = byId.get(id)
      if (!solver) throw new Error(`Unknown solver "${id}"`)
      return solver
    },
  }
}
```

`packages/challenges/src/solvers/uncertainty.ts`:
```ts
import { SolverInputError, requireOutput, type Solver } from './solver'

export type ValueWithUncertainty = { value: number; uncertainty: number }

/**
 * First-order propagation (GUM): u_y² = Σ (∂f/∂x_i · u_i)², with each term from a central difference of
 * step u_i — exact for linear models, and inputs are visited in the solver's fixed order (D16).
 */
export function propagateUncertainty(
  solver: Solver,
  output: string,
  inputs: Readonly<Record<string, ValueWithUncertainty>>,
  gravityMps2: number
): ValueWithUncertainty {
  const nominal: Record<string, number> = {}
  for (const name of solver.inputNames) {
    const input = inputs[name]
    if (!input) throw new SolverInputError(`Missing input "${name}"`)
    nominal[name] = input.value
  }
  const value = requireOutput(solver.solve(nominal, gravityMps2), output)
  let variance = 0
  for (const name of solver.inputNames) {
    const uncertainty = inputs[name]?.uncertainty ?? 0
    if (uncertainty === 0) continue
    const centre = nominal[name] ?? 0
    const up = requireOutput(solver.solve({ ...nominal, [name]: centre + uncertainty }, gravityMps2), output)
    const down = requireOutput(solver.solve({ ...nominal, [name]: centre - uncertainty }, gravityMps2), output)
    const contribution = (up - down) / 2
    variance += contribution * contribution
  }
  return { value, uncertainty: Math.sqrt(variance) }
}
```

`packages/challenges/src/comparison.ts`:
```ts
import type { Measurement } from '@physics-lab/formats'
import type { ValueWithUncertainty } from './solvers/uncertainty'

export type Comparison = {
  taskId: string
  prediction: number
  measurement: Measurement
  theory: ValueWithUncertainty
  combinedUncertainty: number
  coverage: number
  passed: boolean
}

/** Pass when |prediction − measurement| ≤ k·u_c, u_c = sqrt(u_meas² + u_theory² + (tol·theory)²) (D42). */
export function comparePrediction(input: {
  taskId: string
  prediction: number
  measurement: Measurement
  theory: ValueWithUncertainty
  relativeModelTolerance: number
  coverage: number
}): Comparison {
  const model = input.relativeModelTolerance * Math.abs(input.theory.value)
  const combined = Math.sqrt(
    input.measurement.uncertainty * input.measurement.uncertainty + input.theory.uncertainty * input.theory.uncertainty + model * model
  )
  return {
    taskId: input.taskId,
    prediction: input.prediction,
    measurement: input.measurement,
    theory: input.theory,
    combinedUncertainty: combined,
    coverage: input.coverage,
    passed: Math.abs(input.prediction - input.measurement.value) <= input.coverage * combined,
  }
}
```

`packages/challenges/src/index.ts`:
```ts
export * from './comparison'
export * from './solvers/projectile-range'
export * from './solvers/registry'
export * from './solvers/solver'
export * from './solvers/uncertainty'
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm vitest run --project challenges`
Expected: PASS. (The `passes exactly at k·u_c` case depends on `0.2 <= 2 * 0.1`; if floating point makes `15.2 − 15` exceed `0.2`, change the test's prediction to `15.19999999` and note it — do not add an epsilon to the rule.)

- [ ] **Step 5: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test
git add packages/challenges
git commit -m "feat(challenges): add projectile solver, uncertainty propagation and comparison"
```

---

### Task 12: `challenges` — parameters, catapult challenge data, `RunState` and the catapult validation sweep

**Files:**
- Create: `packages/challenges/src/parameters.ts`, `catalog/catapult-range.challenge.json`, `catalog/catalog.ts`, `session/lab-error.ts`, `session/run-state.ts`, `session/create-run.ts`, `validation/catapult.validation.test.ts`
- Modify: `packages/challenges/src/index.ts`, `packages/challenges/tsconfig.json` (include `src/**/*.json`)
- Test: `packages/challenges/src/parameters.test.ts`, `session/run-state.test.ts`, `catalog/catalog.test.ts`

**Interfaces:**
- Consumes: formats (`Challenge`, `ChallengeSchema`, `ParameterDef`, `parseBindPath`, `idToRef`, `refToId`, `splitQuantityId`, `Command`, `CommandLogEntry`, `Measurement`, `RecordedMeasurement`, `WorldDoc`, `QuantityId`, `TERRAIN_ENTITY_ID`, `InstrumentId`); sim-core (`Simulation`, `PhysicsEngineFactory`, `ProjectileFlightSensor`, `isFlightQuantity`, `FlightQuantity`, `applyBuildCommand`, `NO_LOCKS`, `SimEventListener`, `BodyState`); instruments (`measure`, `deriveSpeedFromGate`, `photogateTransitTimeS`); det-math (`RandomSource`, `SeededRandom`).
- Produces:
```ts
// parameters.ts
export function drawParameters(defs: readonly ParameterDef[], random: RandomSource): Record<string, number>
export function applyParameters(world: WorldDoc, defs: readonly ParameterDef[], values: Readonly<Record<string, number>>): WorldDoc
// catalog/catalog.ts
export const BUILT_IN_CHALLENGE_IDS = ['catapult-range'] as const
export type BuiltInChallengeId = (typeof BUILT_IN_CHALLENGE_IDS)[number]
export function loadBuiltInChallenge(id: BuiltInChallengeId): Challenge
// session/lab-error.ts
export const LAB_ERROR_CODES = ['quantityUnavailable', 'unknownQuantity', 'stepInPast', 'runTooLong', 'commandNotSupported',
  'invalidPhase', 'predictionRequired', 'predictionLocked', 'unknownTask', 'explanationTooLong', 'challengeData', 'invalidValue'] as const
export type LabErrorCode = (typeof LAB_ERROR_CODES)[number]
export class LabError extends Error { constructor(readonly code: LabErrorCode, readonly params: Readonly<Record<string, string | number>> = {}) }
// session/run-state.ts
export const MAX_RUN_STEPS = 240 * 20
export const FLIGHT_INSTRUMENTS: Readonly<Record<FlightQuantity, InstrumentId>>
export class RunState {
  constructor(deps: { challenge: Challenge; resolvedWorld: WorldDoc; instrumentRandom: RandomSource; engineFactory: PhysicsEngineFactory })
  get stepIndex(): number
  get commandLog(): readonly CommandLogEntry[]
  get recordedMeasurements(): readonly RecordedMeasurement[]
  readonly resolvedWorld: WorldDoc
  subscribe(listener: SimEventListener): () => void      // survives resets
  step(): void
  stepTo(atStep: number): void
  apply(command: Command): Measurement | null            // trigger | reset | measure | step; logs on success
  readGroundTruth(quantity: QuantityId): number | undefined
  readBody(entityId: EntityId): BodyState
  partIds(): readonly EntityId[]
  dispose(): void
}
// session/create-run.ts
export function createRunForSeed(challenge: Challenge, seed: number, engineFactory: PhysicsEngineFactory): { parameters: Record<string, number>; run: RunState }
export function launchCommand(challenge: Challenge): Command
```

Random streams (D16): `SeededRandom.fromSeed(seed).fork('parameters')` draws parameters; `.fork('instruments')` feeds every measurement, in log order, across resets. Parameters are drawn as `min + step·floor(u·n)`, `n = floor((max − min)/step + 1e-9) + 1`, printed with `toFixed(10)`.

Catapult geometry (D44): ground stone fill `[0,0,0]–[127,1,127]` (top at 1.0 m); static wood post 0.2 × 1.6 × 0.2 m centred at (10, 1.8, 32); wood arm 1.6 × 0.1 × 0.1 m rotated 225° about z, centred 0.8 m from the pivot (9.434, 2.034, 32); stone ball r = 0.1 m at 1.5 m from the pivot (8.939, 1.539, 32), CCD on; hinge `pivot` at (10, 2.6, 32), axis (0, 0, −1), motor 450 °/s, gain 1e4; fixed joint `hold` arm–ball; latch at hinge angle `releaseHingeAngle` ∈ [75°, 105°] step 1° → launch angle 135° − hinge ∈ [30°, 60°], release speed ≈ 11.8 m/s.

- [ ] **Step 1: Write the challenge data**

`packages/challenges/src/catalog/catapult-range.challenge.json`:
```json
{
  "format": "physics-lab/challenge",
  "formatVersion": 1,
  "id": "catapult-range",
  "version": 1,
  "title": { "es": "Catapulta: alcance de un proyectil", "en": "Catapult: projectile range" },
  "topic": "projectile-motion",
  "scenario": {
    "world": {
      "format": "physics-lab/world",
      "formatVersion": 1,
      "gravityMps2": 9.8,
      "blocks": [{ "from": [0, 0, 0], "to": [127, 1, 127], "material": "stone" }],
      "parts": [
        { "id": "post", "type": "beam", "motion": "static", "material": "wood", "layer": "machine", "positionM": [10, 1.8, 32], "rotationDeg": [0, 0, 0], "ccd": false, "sizeM": [0.2, 1.6, 0.2] },
        { "id": "arm", "type": "beam", "motion": "dynamic", "material": "wood", "layer": "machine", "positionM": [9.434, 2.034, 32], "rotationDeg": [0, 0, 225], "ccd": false, "sizeM": [1.6, 0.1, 0.1] },
        { "id": "ball", "type": "sphere", "motion": "dynamic", "material": "stone", "layer": "projectile", "positionM": [8.939, 1.539, 32], "rotationDeg": [0, 0, 0], "ccd": true, "radiusM": 0.1 }
      ],
      "joints": [
        { "id": "pivot", "type": "hinge", "a": "post", "b": "arm", "anchorM": [10, 2.6, 32], "axis": [0, 0, -1], "motor": { "targetSpeedDegPerS": 450, "gain": 10000 } },
        { "id": "hold", "type": "fixed", "a": "arm", "b": "ball" }
      ],
      "latches": [{ "id": "latch", "type": "angleLatch", "hinge": "pivot", "releasesJoint": "hold", "releaseAtHingeAngleDeg": 90 }]
    },
    "locked": ["#post", "#arm", "#ball", "#pivot", "#hold", "#latch"]
  },
  "parameters": [
    { "name": "releaseHingeAngle", "bind": "#latch.releaseAtHingeAngleDeg", "unit": "deg", "range": [75, 105], "distribution": "uniform", "step": 1 }
  ],
  "sensors": [{ "id": "flight", "type": "projectileFlight", "body": "#ball", "latch": "#latch", "gateWidthM": 0.2 }],
  "launch": { "target": "#pivot", "action": "startMotor" },
  "calibration": {
    "pauseOnLatch": "#latch",
    "measurable": ["flight.releaseSpeed", "flight.releaseAngle", "flight.releaseHeight"]
  },
  "tasks": [
    {
      "id": "predict-range",
      "prompt": {
        "es": "Con la rapidez, el ángulo y la altura de liberación que mediste, predice el alcance horizontal de la bola desde el punto de liberación hasta su primer contacto con el suelo.",
        "en": "Using the release speed, angle and height you measured, predict the horizontal range of the ball from the release point to its first contact with the ground."
      },
      "quantity": "flight.range",
      "unit": "m",
      "tolerance": { "kind": "combined-uncertainty", "coverage": 2 }
    }
  ],
  "solverId": "projectile.range.v1",
  "solverInputs": { "speed": "flight.releaseSpeed", "angle": "flight.releaseAngle", "height": "flight.releaseHeight" },
  "instrumentMode": "realistic",
  "allowedCommands": ["help", "measure", "reset", "time"]
}
```

- [ ] **Step 2: Write the failing tests**

`packages/challenges/src/catalog/catalog.test.ts`:
```ts
import { catapultTestWorld } from '@physics-lab/sim-core'
import { describe, expect, it } from 'vitest'
import { loadBuiltInChallenge } from './catalog'

describe('built-in challenges', () => {
  it('loads the catapult challenge through the schema', () => {
    const challenge = loadBuiltInChallenge('catapult-range')
    expect(challenge.id).toBe('catapult-range')
    expect(challenge.solverId).toBe('projectile.range.v1')
  })

  it('uses the same geometry as the sim-core catapult test world', () => {
    const sortById = <T extends { id: string }>(items: readonly T[]): T[] => [...items].sort((a, b) => a.id.localeCompare(b.id))
    const scenario = loadBuiltInChallenge('catapult-range').scenario.world
    const reference = catapultTestWorld(90)
    expect(sortById(scenario.parts)).toEqual(sortById(reference.parts))
    expect(sortById(scenario.joints)).toEqual(sortById(reference.joints))
    expect(scenario.latches).toEqual(reference.latches)
  })
})
```

`packages/challenges/src/parameters.test.ts`:
```ts
import { SeededRandom } from '@physics-lab/det-math'
import { catapultTestWorld } from '@physics-lab/sim-core'
import { describe, expect, it } from 'vitest'
import { applyParameters, drawParameters } from './parameters'

const releaseAngle = { name: 'releaseHingeAngle', bind: '#latch.releaseAtHingeAngleDeg', unit: 'deg', range: [75, 105], distribution: 'uniform', step: 1 } as const

describe('drawParameters', () => {
  it('draws values on the step grid inside the range', () => {
    const random = SeededRandom.fromSeed(1)
    for (let i = 0; i < 1000; i++) {
      const { releaseHingeAngle = Number.NaN } = drawParameters([releaseAngle], random)
      expect(Number.isInteger(releaseHingeAngle)).toBe(true)
      expect(releaseHingeAngle).toBeGreaterThanOrEqual(75)
      expect(releaseHingeAngle).toBeLessThanOrEqual(105)
    }
  })

  it('reaches both ends of the range', () => {
    const random = SeededRandom.fromSeed(2)
    const seen = new Set(Array.from({ length: 2000 }, () => drawParameters([releaseAngle], random)['releaseHingeAngle']))
    expect(seen.has(75) && seen.has(105)).toBe(true)
  })

  it('is reproducible for a seed', () => {
    expect(drawParameters([releaseAngle], SeededRandom.fromSeed(3))).toEqual(drawParameters([releaseAngle], SeededRandom.fromSeed(3)))
  })
})

describe('applyParameters', () => {
  it('writes values through setProperty commands without touching the input', () => {
    const world = catapultTestWorld(90)
    const applied = applyParameters(world, [releaseAngle], { releaseHingeAngle: 80 })
    expect(applied.latches[0]?.releaseAtHingeAngleDeg).toBe(80)
    expect(world.latches[0]?.releaseAtHingeAngleDeg).toBe(90)
  })

  it('throws a LabError for a missing value or an invalid binding', () => {
    expect(() => applyParameters(catapultTestWorld(), [releaseAngle], {})).toThrow(/challengeData/)
    expect(() => applyParameters(catapultTestWorld(), [{ ...releaseAngle, bind: '#latch.colour' }], { releaseHingeAngle: 80 })).toThrow(/challengeData/)
  })
})
```

`packages/challenges/src/session/run-state.test.ts`:
```ts
import type { PhysicsEngineFactory } from '@physics-lab/sim-core'
import { createRapierEngineFactory } from '@physics-lab/sim-core/rapier'
import { beforeAll, describe, expect, it } from 'vitest'
import { loadBuiltInChallenge } from '../catalog/catalog'
import { createRunForSeed, launchCommand } from './create-run'
import { LabError } from './lab-error'

let factory: PhysicsEngineFactory
beforeAll(async () => {
  factory = await createRapierEngineFactory()
})

const challenge = loadBuiltInChallenge('catapult-range')

function stepUntilAvailable(run: ReturnType<typeof createRunForSeed>['run'], quantity: string): void {
  for (let i = 0; i < 4800 && run.readGroundTruth(quantity) === undefined; i++) run.step()
}

describe('RunState', () => {
  it('logs commands with the step at which they were applied', () => {
    const { run } = createRunForSeed(challenge, 7, factory)
    run.apply(launchCommand(challenge))
    stepUntilAvailable(run, 'flight.releaseSpeed')
    const at = run.stepIndex
    run.apply({ type: 'measure', quantity: 'flight.releaseSpeed' })
    expect(run.commandLog).toEqual([
      { atStep: 0, command: { type: 'trigger', target: '#pivot', action: 'startMotor' } },
      { atStep: at, command: { type: 'measure', quantity: 'flight.releaseSpeed' } },
    ])
    expect(run.recordedMeasurements[0]).toMatchObject({ quantity: 'flight.releaseSpeed', instrument: 'photogate', unit: 'm/s', logIndex: 1 })
    run.dispose()
  })

  it('refuses unavailable and unknown quantities without logging them', () => {
    const { run } = createRunForSeed(challenge, 7, factory)
    expect(() => run.apply({ type: 'measure', quantity: 'flight.range' })).toThrow(LabError)
    expect(() => run.apply({ type: 'measure', quantity: 'ghost.range' })).toThrow(LabError)
    expect(run.commandLog).toEqual([])
    run.dispose()
  })

  it('reset rebuilds the build-state world and restarts the step count', () => {
    const { run } = createRunForSeed(challenge, 7, factory)
    run.apply(launchCommand(challenge))
    for (let i = 0; i < 30; i++) run.step()
    run.apply({ type: 'reset' })
    expect(run.stepIndex).toBe(0)
    expect(run.readGroundTruth('flight.releaseSpeed')).toBeUndefined()
    const [x, y, z] = run.readBody('ball').positionM
    expect([x, y, z].map((value) => Number(value.toFixed(4)))).toEqual([8.939, 1.539, 32])
    run.dispose()
  })

  it('refuses to step backwards', () => {
    const { run } = createRunForSeed(challenge, 7, factory)
    run.stepTo(10)
    expect(() => run.stepTo(5)).toThrow(LabError)
    run.dispose()
  })

  it('refuses build commands during a run', () => {
    const { run } = createRunForSeed(challenge, 7, factory)
    expect(() => run.apply({ type: 'setPhysics', gravityMps2: 1 })).toThrow(LabError)
    run.dispose()
  })
})
```

`packages/challenges/src/validation/catapult.validation.test.ts`:
```ts
import { SeededRandom } from '@physics-lab/det-math'
import { WORLD_SIZE_CELLS, BLOCK_SIZE_M } from '@physics-lab/formats'
import type { PhysicsEngineFactory } from '@physics-lab/sim-core'
import { createRapierEngineFactory } from '@physics-lab/sim-core/rapier'
import { beforeAll, describe, expect, it } from 'vitest'
import { loadBuiltInChallenge } from '../catalog/catalog'
import { applyParameters } from '../parameters'
import { launchCommand } from '../session/create-run'
import { RunState } from '../session/run-state'
import { projectileRangeSolver } from '../solvers/projectile-range'

/** Spec §8 row 1 tolerance. */
const CATAPULT_RANGE_TOLERANCE = 0.005
const WORLD_LENGTH_X_M = WORLD_SIZE_CELLS.x * BLOCK_SIZE_M

let factory: PhysicsEngineFactory
beforeAll(async () => {
  factory = await createRapierEngineFactory()
})

const challenge = loadBuiltInChallenge('catapult-range')
const [parameter] = challenge.parameters
const [minAngle = 0, maxAngle = 0] = parameter?.range ?? []
// The publishable range is a discrete 31-value grid, so the sweep is exhaustive (stronger than sampling).
const HINGE_ANGLES = Array.from({ length: maxAngle - minAngle + 1 }, (_, i) => minAngle + i)

describe('catapult validation: solver vs simulation across the publishable range (spec §8, §11)', () => {
  it.each(HINGE_ANGLES)('release hinge angle %d°', (hingeAngle) => {
    const world = applyParameters(challenge.scenario.world, challenge.parameters, { releaseHingeAngle: hingeAngle })
    const run = new RunState({ challenge, resolvedWorld: world, instrumentRandom: SeededRandom.fromSeed(1), engineFactory: factory })
    run.apply(launchCommand(challenge))
    for (let i = 0; i < 4800 && run.readGroundTruth('flight.range') === undefined; i++) run.step()
    const speed = run.readGroundTruth('flight.releaseSpeed') ?? Number.NaN
    const angle = run.readGroundTruth('flight.releaseAngle') ?? Number.NaN
    const height = run.readGroundTruth('flight.releaseHeight') ?? Number.NaN
    const range = run.readGroundTruth('flight.range') ?? Number.NaN
    const theory = projectileRangeSolver.solve({ speed, angle, height }, world.gravityMps2).range ?? Number.NaN

    expect(speed).toBeGreaterThan(10)
    expect(speed).toBeLessThan(13)
    expect(angle).toBeGreaterThan(25)
    expect(angle).toBeLessThan(65)
    expect(9 + range).toBeLessThan(WORLD_LENGTH_X_M)
    expect(Math.abs(range - theory) / theory).toBeLessThanOrEqual(CATAPULT_RANGE_TOLERANCE)
    run.dispose()
  })
})
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `pnpm vitest run --project challenges`
Expected: FAIL — modules not found.

- [ ] **Step 4: Implement**

Dependencies: `pnpm --filter @physics-lab/challenges add @physics-lab/det-math@workspace:*` (already added in Task 1) and add `"include": ["src", "src/**/*.json", "vitest.config.ts"]` to `packages/challenges/tsconfig.json`.

`packages/challenges/src/session/lab-error.ts`:
```ts
export const LAB_ERROR_CODES = [
  'quantityUnavailable',
  'unknownQuantity',
  'stepInPast',
  'runTooLong',
  'commandNotSupported',
  'invalidPhase',
  'predictionRequired',
  'predictionLocked',
  'unknownTask',
  'explanationTooLong',
  'challengeData',
  'invalidValue',
] as const
export type LabErrorCode = (typeof LAB_ERROR_CODES)[number]

/** Typed, translatable error for every refused lab action (the UI maps `code` to an i18n key). */
export class LabError extends Error {
  override name = 'LabError'

  constructor(
    readonly code: LabErrorCode,
    readonly params: Readonly<Record<string, string | number>> = {}
  ) {
    super(code)
  }
}
```

`packages/challenges/src/parameters.ts`:
```ts
import type { RandomSource } from '@physics-lab/det-math'
import { idToRef, parseBindPath, type ParameterDef, type WorldDoc } from '@physics-lab/formats'
import { NO_LOCKS, applyBuildCommand } from '@physics-lab/sim-core'
import { LabError } from './session/lab-error'

const STEP_GRID_EPSILON = 1e-9
const PARAMETER_DECIMALS = 10

/** Uniform draw on the parameter's step grid (spec §7.1), in definition order. */
export function drawParameters(defs: readonly ParameterDef[], random: RandomSource): Record<string, number> {
  const values: Record<string, number> = {}
  for (const def of defs) {
    const [min, max] = def.range
    const count = Math.floor((max - min) / def.step + STEP_GRID_EPSILON) + 1
    const index = Math.floor(random.nextFloat() * count)
    values[def.name] = Number((min + index * def.step).toFixed(PARAMETER_DECIMALS))
  }
  return values
}

/** Applies parameter values with the same setProperty command the console and UI use (no locks). */
export function applyParameters(world: WorldDoc, defs: readonly ParameterDef[], values: Readonly<Record<string, number>>): WorldDoc {
  let current = world
  for (const def of defs) {
    const value = values[def.name]
    if (value === undefined) throw new LabError('challengeData', { parameter: def.name })
    const { entityId, property } = parseBindPath(def.bind)
    const outcome = applyBuildCommand(current, { type: 'setProperty', target: idToRef(entityId), property, value }, NO_LOCKS)
    if (!outcome.ok) throw new LabError('challengeData', { parameter: def.name, reason: outcome.error.code })
    current = outcome.doc
  }
  return current
}
```

(`LabError.message` is its code, so `toThrow(/challengeData/)` matches.)

`packages/challenges/src/catalog/catalog.ts`:
```ts
import { ChallengeSchema, type Challenge } from '@physics-lab/formats'
import catapultRange from './catapult-range.challenge.json'

export const BUILT_IN_CHALLENGE_IDS = ['catapult-range'] as const
export type BuiltInChallengeId = (typeof BUILT_IN_CHALLENGE_IDS)[number]

const BUILT_IN_DATA: Readonly<Record<BuiltInChallengeId, unknown>> = { 'catapult-range': catapultRange }

/** Built-in challenges are data validated by the same schema as teacher files. */
export function loadBuiltInChallenge(id: BuiltInChallengeId): Challenge {
  return ChallengeSchema.parse(BUILT_IN_DATA[id])
}
```

`packages/challenges/src/session/run-state.ts`:
```ts
import type { RandomSource } from '@physics-lab/det-math'
import {
  TERRAIN_ENTITY_ID, refToId, splitQuantityId, type Challenge, type Command, type CommandLogEntry, type EntityId,
  type InstrumentId, type Measurement, type QuantityId, type RecordedMeasurement, type SensorDef, type WorldDoc,
} from '@physics-lab/formats'
import { deriveSpeedFromGate, measure, photogateTransitTimeS } from '@physics-lab/instruments'
import {
  ProjectileFlightSensor, Simulation, isFlightQuantity, type BodyState, type FlightQuantity, type PhysicsEngineFactory,
  type SimEventListener,
} from '@physics-lab/sim-core'
import { LabError } from './lab-error'

/** 20 s of simulated time per run (spec §5.4 step). */
export const MAX_RUN_STEPS = 240 * 20

export const FLIGHT_INSTRUMENTS: Readonly<Record<FlightQuantity, InstrumentId>> = {
  releaseSpeed: 'photogate',
  releaseAngle: 'protractor',
  releaseHeight: 'ruler',
  range: 'ruler',
  time: 'stopwatch',
}

export type RunStateDeps = {
  challenge: Challenge
  resolvedWorld: WorldDoc
  instrumentRandom: RandomSource
  engineFactory: PhysicsEngineFactory
}

type SensorEntry = { def: SensorDef; sensor: ProjectileFlightSensor }

/**
 * One student's run: simulation + sensors + instruments + command log. No pedagogy rules here.
 * Memento pattern: `resolvedWorld` is the Build-state snapshot that `reset` restores (spec §5.3).
 */
export class RunState {
  readonly resolvedWorld: WorldDoc
  private simulation: Simulation
  private sensors = new Map<EntityId, SensorEntry>()
  private readonly log: CommandLogEntry[] = []
  private readonly measurements: RecordedMeasurement[] = []
  private readonly listeners = new Set<SimEventListener>()
  private detachSimulation: () => void = () => undefined

  constructor(private readonly deps: RunStateDeps) {
    this.resolvedWorld = deps.resolvedWorld
    this.simulation = this.buildSimulation()
  }

  get stepIndex(): number {
    return this.simulation.stepIndex
  }

  get commandLog(): readonly CommandLogEntry[] {
    return this.log
  }

  get recordedMeasurements(): readonly RecordedMeasurement[] {
    return this.measurements
  }

  subscribe(listener: SimEventListener): () => void {
    this.listeners.add(listener)
    return () => this.listeners.delete(listener)
  }

  step(): void {
    if (this.simulation.stepIndex >= MAX_RUN_STEPS) throw new LabError('runTooLong', { maxSteps: MAX_RUN_STEPS })
    this.simulation.step()
  }

  stepTo(atStep: number): void {
    if (atStep < this.simulation.stepIndex) throw new LabError('stepInPast', { atStep, current: this.simulation.stepIndex })
    while (this.simulation.stepIndex < atStep) this.step()
  }

  /** Applies a run command and appends it to the log only if it succeeded. */
  apply(command: Command): Measurement | null {
    const atStep = this.simulation.stepIndex
    const measurement = this.execute(command)
    this.log.push({ atStep, command })
    if (measurement) this.measurements.push({ ...measurement, logIndex: this.log.length - 1 })
    return measurement
  }

  readGroundTruth(quantity: QuantityId): number | undefined {
    const { sensorId, reading } = splitQuantityId(quantity)
    const entry = this.sensors.get(sensorId)
    if (!entry || !isFlightQuantity(reading)) return undefined
    return entry.sensor.groundTruth()[reading]
  }

  readBody(entityId: EntityId): BodyState {
    return this.simulation.readBody(entityId)
  }

  partIds(): readonly EntityId[] {
    return this.simulation.partIds()
  }

  dispose(): void {
    this.detachSimulation()
    for (const { sensor } of this.sensors.values()) sensor.dispose()
    this.simulation.dispose()
  }

  private execute(command: Command): Measurement | null {
    switch (command.type) {
      case 'trigger':
        this.simulation.trigger(refToId(command.target), command.action)
        return null
      case 'reset':
        this.rebuild()
        return null
      case 'step':
        for (let i = 0; i < command.count; i++) this.step()
        return null
      case 'measure':
        return this.measureQuantity(command.quantity)
      default:
        throw new LabError('commandNotSupported', { command: command.type })
    }
  }

  private measureQuantity(quantity: QuantityId): Measurement {
    const { sensorId, reading } = splitQuantityId(quantity)
    const entry = this.sensors.get(sensorId)
    if (!entry || !isFlightQuantity(reading)) throw new LabError('unknownQuantity', { quantity })
    const trueValue = entry.sensor.groundTruth()[reading]
    if (trueValue === undefined) throw new LabError('quantityUnavailable', { quantity })
    const mode = this.deps.challenge.instrumentMode
    const random = this.deps.instrumentRandom
    if (reading === 'releaseSpeed') {
      const transit = measure({ quantity, instrument: 'photogate', trueValue: photogateTransitTimeS(entry.def.gateWidthM, trueValue), mode }, random)
      return deriveSpeedFromGate(entry.def.gateWidthM, transit)
    }
    return measure({ quantity, instrument: FLIGHT_INSTRUMENTS[reading], trueValue, mode }, random)
  }

  private rebuild(): void {
    this.dispose()
    this.simulation = this.buildSimulation()
  }

  private buildSimulation(): Simulation {
    const simulation = Simulation.create(this.resolvedWorld, this.deps.engineFactory)
    this.sensors = new Map(
      this.deps.challenge.sensors.map((def) => [
        def.id,
        {
          def,
          sensor: new ProjectileFlightSensor(
            { bodyId: refToId(def.body), surfaceId: TERRAIN_ENTITY_ID, release: { kind: 'latch', latchId: refToId(def.latch) } },
            simulation
          ),
        },
      ])
    )
    this.detachSimulation = simulation.subscribe((event) => {
      for (const listener of this.listeners) listener(event)
    })
    return simulation
  }
}
```

> Sensors subscribe before the run-state forwarder, so listeners outside (the UI) see an event after sensors have processed it.

`packages/challenges/src/session/create-run.ts`:
```ts
import { SeededRandom } from '@physics-lab/det-math'
import type { Challenge, Command } from '@physics-lab/formats'
import type { PhysicsEngineFactory } from '@physics-lab/sim-core'
import { applyParameters, drawParameters } from '../parameters'
import { RunState } from './run-state'

export const PARAMETER_STREAM = 'parameters'
export const INSTRUMENT_STREAM = 'instruments'

/** Same seed → same parameters, world and noise stream; shared by live sessions and replay (D25). */
export function createRunForSeed(challenge: Challenge, seed: number, engineFactory: PhysicsEngineFactory): { parameters: Record<string, number>; run: RunState } {
  const random = SeededRandom.fromSeed(seed)
  const parameters = drawParameters(challenge.parameters, random.fork(PARAMETER_STREAM))
  const resolvedWorld = applyParameters(challenge.scenario.world, challenge.parameters, parameters)
  const run = new RunState({ challenge, resolvedWorld, instrumentRandom: random.fork(INSTRUMENT_STREAM), engineFactory })
  return { parameters, run }
}

export function launchCommand(challenge: Challenge): Command {
  return { type: 'trigger', target: challenge.launch.target, action: challenge.launch.action }
}
```

Append to `packages/challenges/src/index.ts`:
```ts
export * from './catalog/catalog'
export * from './parameters'
export * from './session/create-run'
export * from './session/lab-error'
export * from './session/run-state'
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `pnpm vitest run --project challenges`
Expected: PASS, including 31 catapult validation cases. If any case misses 0.5 % or the speed/angle bounds, stop and send the table of measured values to `physics-reviewer` (likely levers: motor gain, latch overshoot, CCD). Geometry or tolerance changes go through the main session and the decisions log.

- [ ] **Step 6: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test
git add packages/challenges
git commit -m "feat(challenges): add catapult challenge with run state and validation sweep"
```

---
### Task 13: `challenges` — `LabSession` (predict → observe loop) and `verifyResult` (replay)

**Files:**
- Create: `packages/challenges/src/session/lab-session.ts`, `session/verify-result.ts`
- Modify: `packages/challenges/src/index.ts`
- Test: `packages/challenges/src/session/lab-session.test.ts`, `session/verify-result.test.ts`

**Interfaces:**
- Consumes: Tasks 11–12; formats (`Result`, `Prediction`, `LearningEvent`, `Task`, `MAX_EXPLANATION_CHARS`, `RESULT_FORMAT*`, `WORLD_FORMAT_VERSION`, `CHALLENGE_FORMAT_VERSION`, `StudentIdSchema`, `normalizeStudentId`); sim-core (`SimulationError`, `SimEventListener`, `BodyState`).
- Produces:
```ts
export type LabPhase = 'ready' | 'calibrating' | 'calibrated' | 'running' | 'landed'
export interface Clock { nowMs(): number }
export type LabSessionDeps = {
  challenge: Challenge; seed: number; studentId: string; engineFactory: PhysicsEngineFactory
  clock: Clock; appVersion: string; solvers: SolverRegistry
}
export class LabSession {
  static create(deps: LabSessionDeps): LabSession
  readonly parameters: Readonly<Record<string, number>>
  get phase(): LabPhase
  get stepIndex(): number
  get challenge(): Challenge
  get resolvedWorld(): WorldDoc
  readBody(entityId: EntityId): BodyState
  partIds(): readonly EntityId[]
  subscribe(listener: SimEventListener): () => void
  launch(): void                       // ready → calibrating (calibration and predictions missing) | running
  advance(steps: number): void         // steps while calibrating/running/landed; stops at calibration pause
  continueRun(): void                  // calibrated → running; needs every prediction
  reset(): void                        // any → ready (predictions kept)
  measure(quantity: QuantityId): Measurement
  submitPrediction(taskId: EntityId, value: number): void
  compare(taskId: EntityId): Comparison
  submitExplanation(taskId: EntityId, text: string): void
  toResult(): Result
  dispose(): void
}
export type VerificationStatus = 'verified' | 'mismatch' | 'engineChanged' | 'challengeMismatch' | 'invalidLog'
export type VerificationDifference = { field: string; recorded: string; replayed: string }
export type VerificationReport = { status: VerificationStatus; differences: VerificationDifference[] }
export function verifyResult(challenge: Challenge, result: Result, engineFactory: PhysicsEngineFactory): VerificationReport
```

Pedagogy rules (spec §7.3, D44): the calibration pause happens when every `calibration.measurable` quantity has a ground-truth value; `landed` when every task quantity has one; task quantities cannot be measured before every prediction is submitted; a prediction is locked once its task quantity has been measured; `compare` measures any missing solver input (logged) and uses the latest measurement of each. Replay (spec §7.5): same seed → same parameters; re-execute the log at each `atStep`; compare every recorded measurement exactly; an engine-id difference reports `engineChanged` instead of `mismatch`.

- [ ] **Step 1: Write the failing tests**

`packages/challenges/src/session/lab-session.test.ts`:
```ts
import { ResultSchema } from '@physics-lab/formats'
import type { PhysicsEngineFactory } from '@physics-lab/sim-core'
import { createRapierEngineFactory } from '@physics-lab/sim-core/rapier'
import { beforeAll, describe, expect, it } from 'vitest'
import { loadBuiltInChallenge } from '../catalog/catalog'
import { BUILT_IN_SOLVERS, createSolverRegistry } from '../solvers/registry'
import { projectileRangeSolver } from '../solvers/projectile-range'
import { LabError } from './lab-error'
import { LabSession, type LabPhase } from './lab-session'
import { verifyResult } from './verify-result'

let factory: PhysicsEngineFactory
beforeAll(async () => {
  factory = await createRapierEngineFactory()
})

const challenge = loadBuiltInChallenge('catapult-range')
const TASK = 'predict-range'

function newSession(seed = 12345): { session: LabSession; clock: { now: number } } {
  const clock = { now: 1000 }
  const session = LabSession.create({
    challenge, seed, studentId: '  José ', engineFactory: factory, clock: { nowMs: () => clock.now },
    appVersion: '0.1.0-test', solvers: createSolverRegistry(BUILT_IN_SOLVERS),
  })
  return { session, clock }
}

function advanceUntil(session: LabSession, phase: LabPhase): void {
  for (let i = 0; i < 1000 && session.phase !== phase; i++) session.advance(10)
  expect(session.phase).toBe(phase)
}

function theoryFromMeasuredRelease(session: LabSession): number {
  const speed = session.measure('flight.releaseSpeed').value
  const angle = session.measure('flight.releaseAngle').value
  const height = session.measure('flight.releaseHeight').value
  return projectileRangeSolver.solve({ speed, angle, height }, 9.8).range ?? Number.NaN
}

function completeChallenge(session: LabSession): void {
  session.launch()
  advanceUntil(session, 'calibrated')
  session.submitPrediction(TASK, theoryFromMeasuredRelease(session))
  session.continueRun()
  advanceUntil(session, 'landed')
  session.measure('flight.range')
}

describe('LabSession', () => {
  it('draws the same parameters for the same seed', () => {
    expect(newSession(1).session.parameters).toEqual(newSession(1).session.parameters)
    const angles = new Set([1, 2, 3, 4, 5, 6, 7, 8].map((seed) => newSession(seed).session.parameters['releaseHingeAngle']))
    expect(angles.size).toBeGreaterThan(1)
  })

  it('pauses the calibration run at release with release quantities measurable', () => {
    const { session } = newSession()
    session.launch()
    expect(session.phase).toBe('calibrating')
    advanceUntil(session, 'calibrated')
    const steps = session.stepIndex
    session.advance(100)
    expect(session.stepIndex).toBe(steps)
    expect(session.measure('flight.releaseSpeed').unit).toBe('m/s')
  })

  it('passes a prediction equal to the theory from measured inputs', () => {
    const { session } = newSession()
    completeChallenge(session)
    const comparison = session.compare(TASK)
    expect(comparison.passed).toBe(true)
    expect(comparison.combinedUncertainty).toBeGreaterThan(comparison.measurement.uncertainty)
  })

  it('exports a schema-valid result that verifies by replay', () => {
    const { session, clock } = newSession()
    completeChallenge(session)
    clock.now = 90_000
    session.compare(TASK)
    session.submitExplanation(TASK, '  Ignoring air drag.  ')
    const result = ResultSchema.parse(session.toResult())
    expect(result.studentId).toBe('José')
    expect(result.explanations).toEqual([{ taskId: TASK, text: 'Ignoring air drag.' }])
    expect(result.learningEvents.map((event) => event.type)).toContain('predictionSubmitted')
    expect(verifyResult(challenge, result, factory)).toEqual({ status: 'verified', differences: [] })
  })

  it('goes straight to the full run when predictions exist, and keeps them across reset', () => {
    const { session } = newSession()
    session.launch()
    advanceUntil(session, 'calibrated')
    session.submitPrediction(TASK, 15)
    session.reset()
    expect(session.phase).toBe('ready')
    session.launch()
    expect(session.phase).toBe('running')
  })

  describe('refused actions (Review Focus 4)', () => {
    const expectCode = (action: () => unknown, code: string): void => {
      try {
        action()
      } catch (error) {
        expect(error).toBeInstanceOf(LabError)
        expect((error as LabError).code).toBe(code)
        return
      }
      throw new Error(`expected LabError ${code}`)
    }

    it('refuses to measure release quantities before launch', () => {
      expectCode(() => newSession().session.measure('flight.releaseSpeed'), 'quantityUnavailable')
    })

    it('refuses to launch twice', () => {
      const { session } = newSession()
      session.launch()
      expectCode(() => session.launch(), 'invalidPhase')
    })

    it('refuses to continue or measure the range without a prediction', () => {
      const { session } = newSession()
      session.launch()
      advanceUntil(session, 'calibrated')
      expectCode(() => session.continueRun(), 'predictionRequired')
      expectCode(() => session.measure('flight.range'), 'predictionRequired')
    })

    it('locks the prediction once the range has been measured', () => {
      const { session } = newSession()
      completeChallenge(session)
      expectCode(() => session.submitPrediction(TASK, 1), 'predictionLocked')
    })

    it('rejects unknown tasks, non-finite predictions and oversized explanations', () => {
      const { session } = newSession()
      expectCode(() => session.submitPrediction('ghost', 1), 'unknownTask')
      expectCode(() => session.submitPrediction(TASK, Number.NaN), 'invalidValue')
      expectCode(() => session.submitExplanation(TASK, 'x'.repeat(2001)), 'explanationTooLong')
    })

    it('rejects an empty student id', () => {
      expectCode(
        () => LabSession.create({ challenge, seed: 1, studentId: '   ', engineFactory: factory, clock: { nowMs: () => 0 }, appVersion: 'x', solvers: createSolverRegistry(BUILT_IN_SOLVERS) }),
        'invalidValue'
      )
    })
  })
})
```

`packages/challenges/src/session/verify-result.test.ts`:
```ts
import type { Result } from '@physics-lab/formats'
import type { PhysicsEngineFactory } from '@physics-lab/sim-core'
import { createRapierEngineFactory } from '@physics-lab/sim-core/rapier'
import { beforeAll, describe, expect, it } from 'vitest'
import { loadBuiltInChallenge } from '../catalog/catalog'
import { BUILT_IN_SOLVERS, createSolverRegistry } from '../solvers/registry'
import { LabSession } from './lab-session'
import { verifyResult } from './verify-result'

let factory: PhysicsEngineFactory
let result: Result
const challenge = loadBuiltInChallenge('catapult-range')

beforeAll(async () => {
  factory = await createRapierEngineFactory()
  const session = LabSession.create({
    challenge, seed: 777, studentId: 'ana', engineFactory: factory, clock: { nowMs: () => 0 },
    appVersion: 'test', solvers: createSolverRegistry(BUILT_IN_SOLVERS),
  })
  session.launch()
  for (let i = 0; i < 1000 && session.phase !== 'calibrated'; i++) session.advance(10)
  session.measure('flight.releaseSpeed')
  session.measure('flight.releaseAngle')
  session.submitPrediction('predict-range', 14)
  session.continueRun()
  for (let i = 0; i < 1000 && session.phase !== 'landed'; i++) session.advance(10)
  session.measure('flight.range')
  result = session.toResult()
  session.dispose()
})

describe('verifyResult (spec §7.5, Review Focus 3)', () => {
  it('verifies an untouched result', () => {
    expect(verifyResult(challenge, result, factory).status).toBe('verified')
  })

  it('reports an edited measurement as a mismatch naming the field', () => {
    const tampered = structuredClone(result)
    const last = tampered.measurements.at(-1)
    if (last) last.value += 0.5
    const report = verifyResult(challenge, tampered, factory)
    expect(report.status).toBe('mismatch')
    expect(report.differences[0]?.field).toBe(`measurements.${tampered.measurements.length - 1}`)
  })

  it('reports edited parameters as a mismatch', () => {
    const tampered = { ...result, parameters: { releaseHingeAngle: -1 } }
    expect(verifyResult(challenge, tampered, factory).status).toBe('mismatch')
  })

  it('reports a log that steps backwards as invalid', () => {
    const tampered = structuredClone(result)
    tampered.commandLog.push({ atStep: 0, command: { type: 'measure', quantity: 'flight.range' } })
    expect(verifyResult(challenge, tampered, factory).status).toBe('invalidLog')
  })

  it('reports an unknown quantity or a build command in the log as invalid', () => {
    const unknown = structuredClone(result)
    unknown.commandLog.push({ atStep: 5000, command: { type: 'measure', quantity: 'ghost.range' } })
    expect(verifyResult(challenge, unknown, factory).status).toBe('invalidLog')
    const build = structuredClone(result)
    build.commandLog.unshift({ atStep: 0, command: { type: 'setPhysics', gravityMps2: 1 } })
    expect(verifyResult(challenge, build, factory).status).toBe('invalidLog')
  })

  it('reports engineChanged instead of a false mismatch when the engine differs', () => {
    const otherEngine: PhysicsEngineFactory = { engineId: 'other-engine@9.9.9', create: (config) => factory.create(config) }
    expect(verifyResult(challenge, result, otherEngine).status).toBe('engineChanged')
  })

  it('refuses a result for another challenge version', () => {
    const tampered = { ...result, challenge: { id: result.challenge.id, version: 99 } }
    expect(verifyResult(challenge, tampered, factory).status).toBe('challengeMismatch')
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run --project challenges session`
Expected: FAIL — `./lab-session`, `./verify-result` not found.

- [ ] **Step 3: Implement**

`packages/challenges/src/session/lab-session.ts`:
```ts
import {
  CHALLENGE_FORMAT_VERSION, MAX_EXPLANATION_CHARS, RESULT_FORMAT, RESULT_FORMAT_VERSION, StudentIdSchema, WORLD_FORMAT_VERSION,
  normalizeStudentId, splitQuantityId, type Challenge, type EntityId, type LearningEvent, type Measurement, type Prediction,
  type QuantityId, type Result, type Task, type WorldDoc,
} from '@physics-lab/formats'
import type { BodyState, PhysicsEngineFactory, SimEventListener } from '@physics-lab/sim-core'
import { comparePrediction, type Comparison } from '../comparison'
import type { SolverRegistry } from '../solvers/registry'
import { SolverInputError } from '../solvers/solver'
import { propagateUncertainty, type ValueWithUncertainty } from '../solvers/uncertainty'
import { createRunForSeed, launchCommand } from './create-run'
import { LabError } from './lab-error'
import { MAX_RUN_STEPS, type RunState } from './run-state'

export type LabPhase = 'ready' | 'calibrating' | 'calibrated' | 'running' | 'landed'

/** Port for wall-clock time; only used for learning-event timestamps, never for physics (D16). */
export interface Clock {
  nowMs(): number
}

export type LabSessionDeps = {
  challenge: Challenge
  seed: number
  studentId: string
  engineFactory: PhysicsEngineFactory
  clock: Clock
  appVersion: string
  solvers: SolverRegistry
}

const STEPPING_PHASES: ReadonlySet<LabPhase> = new Set(['calibrating', 'running', 'landed'])

/** Template Method: the fixed predict → run → observe → compare → explain flow; challenge data fills the hooks. */
export class LabSession {
  private phaseValue: LabPhase = 'ready'
  private readonly predictions = new Map<EntityId, Prediction>()
  private readonly explanations = new Map<EntityId, string>()
  private readonly learningEvents: LearningEvent[] = []
  private readonly startedAtMs: number

  private constructor(
    private readonly deps: LabSessionDeps,
    private readonly studentId: string,
    readonly parameters: Readonly<Record<string, number>>,
    private readonly run: RunState
  ) {
    this.startedAtMs = deps.clock.nowMs()
  }

  static create(deps: LabSessionDeps): LabSession {
    const studentId = normalizeStudentId(deps.studentId)
    if (!StudentIdSchema.safeParse(studentId).success) throw new LabError('invalidValue', { field: 'studentId' })
    const { parameters, run } = createRunForSeed(deps.challenge, deps.seed, deps.engineFactory)
    return new LabSession(deps, studentId, parameters, run)
  }

  get phase(): LabPhase {
    return this.phaseValue
  }

  get stepIndex(): number {
    return this.run.stepIndex
  }

  get challenge(): Challenge {
    return this.deps.challenge
  }

  get resolvedWorld(): WorldDoc {
    return this.run.resolvedWorld
  }

  readBody(entityId: EntityId): BodyState {
    return this.run.readBody(entityId)
  }

  partIds(): readonly EntityId[] {
    return this.run.partIds()
  }

  subscribe(listener: SimEventListener): () => void {
    return this.run.subscribe(listener)
  }

  launch(): void {
    this.requirePhase('ready')
    this.run.apply(launchCommand(this.deps.challenge))
    this.phaseValue = this.deps.challenge.calibration !== null && !this.allPredicted() ? 'calibrating' : 'running'
    this.record('launch')
  }

  advance(steps: number): void {
    for (let i = 0; i < steps; i++) {
      if (!STEPPING_PHASES.has(this.phaseValue) || this.run.stepIndex >= MAX_RUN_STEPS) return
      this.run.step()
      this.updatePhaseAfterStep()
      if (this.phaseValue === 'calibrated') return
    }
  }

  continueRun(): void {
    this.requirePhase('calibrated')
    if (!this.allPredicted()) throw new LabError('predictionRequired')
    this.phaseValue = 'running'
  }

  reset(): void {
    this.run.apply({ type: 'reset' })
    this.phaseValue = 'ready'
    this.record('reset')
  }

  measure(quantity: QuantityId): Measurement {
    if (this.isTaskQuantity(quantity) && !this.allPredicted()) throw new LabError('predictionRequired', { quantity })
    const measurement = this.run.apply({ type: 'measure', quantity })
    if (!measurement) throw new LabError('quantityUnavailable', { quantity })
    this.record('measure')
    return measurement
  }

  submitPrediction(taskId: EntityId, value: number): void {
    const task = this.requireTask(taskId)
    if (!Number.isFinite(value)) throw new LabError('invalidValue', { field: 'prediction' })
    if (this.latestMeasurement(task.quantity)) throw new LabError('predictionLocked', { taskId })
    const revised = this.predictions.has(taskId)
    this.predictions.set(taskId, { taskId, value, unit: task.unit, submittedAtMs: this.elapsedMs() })
    this.record(revised ? 'predictionRevised' : 'predictionSubmitted', taskId)
  }

  compare(taskId: EntityId): Comparison {
    const task = this.requireTask(taskId)
    const prediction = this.predictions.get(taskId)
    if (!prediction) throw new LabError('predictionRequired', { taskId })
    const measurement = this.latestMeasurement(task.quantity)
    if (!measurement) throw new LabError('quantityUnavailable', { quantity: task.quantity })
    const solver = this.deps.solvers.get(this.deps.challenge.solverId)
    const inputs: Record<string, ValueWithUncertainty> = {}
    for (const name of solver.inputNames) {
      const quantity = this.deps.challenge.solverInputs[name]
      if (!quantity) throw new LabError('challengeData', { input: name })
      const input = this.latestMeasurement(quantity) ?? this.measure(quantity)
      inputs[name] = { value: input.value, uncertainty: input.uncertainty }
    }
    const theory = this.theoryFor(task, inputs)
    const comparison = comparePrediction({
      taskId, prediction: prediction.value, measurement, theory,
      relativeModelTolerance: solver.relativeModelTolerance, coverage: task.tolerance.coverage,
    })
    this.record('comparison', taskId)
    return comparison
  }

  submitExplanation(taskId: EntityId, text: string): void {
    this.requireTask(taskId)
    const trimmed = text.trim()
    if (trimmed.length > MAX_EXPLANATION_CHARS) throw new LabError('explanationTooLong', { max: MAX_EXPLANATION_CHARS })
    this.explanations.set(taskId, trimmed)
    this.record('explanationSubmitted', taskId)
  }

  toResult(): Result {
    const byTask = <T>(entries: Iterable<[EntityId, T]>): [EntityId, T][] => [...entries].sort(([a], [b]) => (a < b ? -1 : a > b ? 1 : 0))
    return {
      format: RESULT_FORMAT,
      formatVersion: RESULT_FORMAT_VERSION,
      appVersion: this.deps.appVersion,
      engineVersion: this.deps.engineFactory.engineId,
      formatVersions: { world: WORLD_FORMAT_VERSION, challenge: CHALLENGE_FORMAT_VERSION, result: RESULT_FORMAT_VERSION },
      challenge: { id: this.deps.challenge.id, version: this.deps.challenge.version },
      seed: this.deps.seed >>> 0,
      studentId: this.studentId,
      parameters: { ...this.parameters },
      predictions: byTask(this.predictions.entries()).map(([, prediction]) => prediction),
      explanations: byTask(this.explanations.entries()).map(([taskId, text]) => ({ taskId, text })),
      commandLog: [...this.run.commandLog],
      measurements: [...this.run.recordedMeasurements],
      learningEvents: [...this.learningEvents],
    }
  }

  dispose(): void {
    this.run.dispose()
  }

  private theoryFor(task: Task, inputs: Readonly<Record<string, ValueWithUncertainty>>): ValueWithUncertainty {
    const solver = this.deps.solvers.get(this.deps.challenge.solverId)
    try {
      return propagateUncertainty(solver, splitQuantityId(task.quantity).reading, inputs, this.run.resolvedWorld.gravityMps2)
    } catch (error) {
      if (error instanceof SolverInputError) throw new LabError('invalidValue', { field: 'solverInputs' })
      throw error
    }
  }

  private updatePhaseAfterStep(): void {
    const calibration = this.deps.challenge.calibration
    // The pause condition is data-driven: every calibration quantity is measurable (release has happened).
    if (this.phaseValue === 'calibrating' && calibration && calibration.measurable.every((q) => this.run.readGroundTruth(q) !== undefined)) {
      this.phaseValue = 'calibrated'
      return
    }
    if (this.phaseValue === 'running' && this.deps.challenge.tasks.every((task) => this.run.readGroundTruth(task.quantity) !== undefined)) {
      this.phaseValue = 'landed'
    }
  }

  private requirePhase(expected: LabPhase): void {
    if (this.phaseValue !== expected) throw new LabError('invalidPhase', { expected, actual: this.phaseValue })
  }

  private requireTask(taskId: EntityId): Task {
    const task = this.deps.challenge.tasks.find((candidate) => candidate.id === taskId)
    if (!task) throw new LabError('unknownTask', { taskId })
    return task
  }

  private allPredicted(): boolean {
    return this.deps.challenge.tasks.every((task) => this.predictions.has(task.id))
  }

  private isTaskQuantity(quantity: QuantityId): boolean {
    return this.deps.challenge.tasks.some((task) => task.quantity === quantity)
  }

  private latestMeasurement(quantity: QuantityId): Measurement | undefined {
    return [...this.run.recordedMeasurements].reverse().find((measurement) => measurement.quantity === quantity)
  }

  private elapsedMs(): number {
    return Math.max(0, this.deps.clock.nowMs() - this.startedAtMs)
  }

  private record(type: LearningEvent['type'], taskId?: EntityId): void {
    this.learningEvents.push(taskId === undefined ? { type, atMs: this.elapsedMs() } : { type, atMs: this.elapsedMs(), taskId })
  }
}
```

`packages/challenges/src/session/verify-result.ts`:
```ts
import type { Challenge, CommandLogEntry, RecordedMeasurement, Result } from '@physics-lab/formats'
import { SimulationError, type PhysicsEngineFactory } from '@physics-lab/sim-core'
import { createRunForSeed } from './create-run'
import { LabError } from './lab-error'
import type { RunState } from './run-state'

export type VerificationStatus = 'verified' | 'mismatch' | 'engineChanged' | 'challengeMismatch' | 'invalidLog'
export type VerificationDifference = { field: string; recorded: string; replayed: string }
export type VerificationReport = { status: VerificationStatus; differences: VerificationDifference[] }

const MISSING = 'missing'

function describeMeasurement(measurement: RecordedMeasurement | undefined): string {
  return measurement === undefined ? MISSING : JSON.stringify(measurement)
}

function diffParameters(replayed: Readonly<Record<string, number>>, recorded: Readonly<Record<string, number>>): VerificationDifference[] {
  const names = [...new Set([...Object.keys(replayed), ...Object.keys(recorded)])].sort()
  return names
    .filter((name) => replayed[name] !== recorded[name])
    .map((name) => ({ field: `parameters.${name}`, recorded: String(recorded[name] ?? MISSING), replayed: String(replayed[name] ?? MISSING) }))
}

function diffMeasurements(recorded: readonly RecordedMeasurement[], replayed: readonly RecordedMeasurement[]): VerificationDifference[] {
  const differences: VerificationDifference[] = []
  for (let index = 0; index < Math.max(recorded.length, replayed.length); index++) {
    const a = describeMeasurement(recorded[index])
    const b = describeMeasurement(replayed[index])
    if (a !== b) differences.push({ field: `measurements.${index}`, recorded: a, replayed: b })
  }
  return differences
}

function replayLog(run: RunState, log: readonly CommandLogEntry[]): void {
  for (const entry of log) {
    run.stepTo(entry.atStep)
    run.apply(entry.command)
  }
}

/** Deterministic replay of a student's command log (spec §7.5). Never throws for tampered data. */
export function verifyResult(challenge: Challenge, result: Result, engineFactory: PhysicsEngineFactory): VerificationReport {
  if (challenge.id !== result.challenge.id || challenge.version !== result.challenge.version) {
    return {
      status: 'challengeMismatch',
      differences: [{ field: 'challenge', recorded: `${result.challenge.id}@${result.challenge.version}`, replayed: `${challenge.id}@${challenge.version}` }],
    }
  }
  const { parameters, run } = createRunForSeed(challenge, result.seed, engineFactory)
  try {
    const differences = diffParameters(parameters, result.parameters)
    try {
      replayLog(run, result.commandLog)
    } catch (error) {
      if (error instanceof LabError || error instanceof SimulationError) {
        return { status: 'invalidLog', differences: [{ field: 'commandLog', recorded: `${result.commandLog.length} entries`, replayed: error.code }] }
      }
      throw error
    }
    differences.push(...diffMeasurements(result.measurements, run.recordedMeasurements))
    if (engineFactory.engineId !== result.engineVersion) return { status: 'engineChanged', differences }
    return { status: differences.length === 0 ? 'verified' : 'mismatch', differences }
  } finally {
    run.dispose()
  }
}
```

Append to `packages/challenges/src/index.ts`:
```ts
export * from './session/lab-session'
export * from './session/verify-result'
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm vitest run --project challenges`
Expected: PASS.

- [ ] **Step 5: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test
git add packages/challenges
git commit -m "feat(challenges): add lab session flow and replay verification"
```

---

### Task 14: `console` — parser, M1 commands and permissions

**Files:**
- Create: `packages/console/src/tokenize.ts`, `console-types.ts`, `commands.ts`, `create-console.ts`
- Modify: `packages/console/src/index.ts`
- Test: `packages/console/src/tokenize.test.ts`, `create-console.test.ts`

**Interfaces:**
- Consumes: `Command`, `QuantityIdSchema`, `ENTITY_ID_PATTERN` (formats). (`sim-core` is allowed by the dependency graph but not needed in M1.)
- Produces:
```ts
export type Token = { kind: 'number'; value: number; text: string } | { kind: 'ref'; id: string; property: string | null; text: string } | { kind: 'word'; text: string }
export function tokenize(line: string): Token[]
export type ConsoleMessage = { key: string; params: Readonly<Record<string, string | number>> }
export type UiAction = { type: 'showHelp'; command: string | null } | { type: 'setTimeScale'; factor: number }
export type ConsoleOutcome = { kind: 'command'; command: Command } | { kind: 'ui'; action: UiAction } | { kind: 'error'; message: ConsoleMessage }
export type ConsoleCommandDefinition = { name: string; category: 'build' | 'world' | 'instruments' | 'meta'; parse(args: readonly Token[]): ConsoleOutcome }
export type CommandPermissions = { kind: 'all' } | { kind: 'only'; names: readonly string[] }
export const M1_CONSOLE_COMMANDS: readonly ConsoleCommandDefinition[]   // help, measure, reset, time, set
export const TIME_SCALE_RANGE = { min: 0.1, max: 4 } as const
export const CONSOLE_ERROR_KEYS: readonly string[]
export function consoleMessageKeys(commands: readonly ConsoleCommandDefinition[]): string[]  // console.help.<name>, console.usage.<name>, errors
export type GameConsole = { execute(line: string): ConsoleOutcome; commandNames(): string[] }
export function createConsole(commands: readonly ConsoleCommandDefinition[], permissions: CommandPermissions): GameConsole
```

Grammar (spec §9, M1 subset): `/help [command]`, `/measure <sensor.reading>`, `/reset`, `/time scale <0.1–4>`, `/set #<id>.<property> <number>`. Relative `~` coordinates and `@` selectors arrive in M2. The leading `/` is optional. Every user-facing text is an i18n key; the console holds no strings.

- [ ] **Step 1: Write the failing tests**

`packages/console/src/tokenize.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { tokenize } from './tokenize'

describe('tokenize', () => {
  it('splits on whitespace and drops the leading slash', () => {
    expect(tokenize('/time   scale 0.5')).toEqual([
      { kind: 'word', text: 'time' },
      { kind: 'word', text: 'scale' },
      { kind: 'number', value: 0.5, text: '0.5' },
    ])
  })

  it('recognises entity refs with an optional property', () => {
    expect(tokenize('set #latch.releaseAtHingeAngleDeg -80')).toEqual([
      { kind: 'word', text: 'set' },
      { kind: 'ref', id: 'latch', property: 'releaseAtHingeAngleDeg', text: '#latch.releaseAtHingeAngleDeg' },
      { kind: 'number', value: -80, text: '-80' },
    ])
    expect(tokenize('#ball')[0]).toEqual({ kind: 'ref', id: 'ball', property: null, text: '#ball' })
  })

  it('returns no tokens for blank input', () => {
    expect(tokenize('  / ')).toEqual([])
  })
})
```

`packages/console/src/create-console.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { M1_CONSOLE_COMMANDS, consoleMessageKeys } from './commands'
import { createConsole } from './create-console'

const open = createConsole(M1_CONSOLE_COMMANDS, { kind: 'all' })

describe('M1 console commands', () => {
  it('compiles /measure into the core measure command', () => {
    expect(open.execute('/measure flight.range')).toEqual({ kind: 'command', command: { type: 'measure', quantity: 'flight.range' } })
  })

  it('compiles /reset and /set into core commands', () => {
    expect(open.execute('/reset')).toEqual({ kind: 'command', command: { type: 'reset' } })
    expect(open.execute('/set #latch.releaseAtHingeAngleDeg 80')).toEqual({
      kind: 'command',
      command: { type: 'setProperty', target: '#latch', property: 'releaseAtHingeAngleDeg', value: 80 },
    })
  })

  it('turns /help and /time scale into UI actions', () => {
    expect(open.execute('/help')).toEqual({ kind: 'ui', action: { type: 'showHelp', command: null } })
    expect(open.execute('/help measure')).toEqual({ kind: 'ui', action: { type: 'showHelp', command: 'measure' } })
    expect(open.execute('/time scale 0.25')).toEqual({ kind: 'ui', action: { type: 'setTimeScale', factor: 0.25 } })
  })

  it('reports usage errors with the command name', () => {
    expect(open.execute('/measure range')).toEqual({ kind: 'error', message: { key: 'console.error.usage', params: { command: 'measure' } } })
    expect(open.execute('/set #latch 80')).toEqual({ kind: 'error', message: { key: 'console.error.usage', params: { command: 'set' } } })
  })

  it('reports out-of-range time scales', () => {
    expect(open.execute('/time scale 10')).toEqual({ kind: 'error', message: { key: 'console.error.outOfRange', params: { min: 0.1, max: 4 } } })
  })

  it('reports unknown commands and empty input', () => {
    expect(open.execute('/teleport')).toEqual({ kind: 'error', message: { key: 'console.error.unknownCommand', params: { name: 'teleport' } } })
    expect(open.execute('   ')).toEqual({ kind: 'error', message: { key: 'console.error.empty', params: {} } })
  })
})

describe('permissions (enumerate–classify–ratchet, D39)', () => {
  it.each(M1_CONSOLE_COMMANDS.map((command) => command.name))('/%s is refused when the challenge does not allow it', (name) => {
    const locked = createConsole(M1_CONSOLE_COMMANDS, { kind: 'only', names: [] })
    expect(locked.execute(`/${name}`)).toEqual({ kind: 'error', message: { key: 'console.error.notAllowed', params: { name } } })
  })

  it('allows exactly the whitelisted commands', () => {
    const catapult = createConsole(M1_CONSOLE_COMMANDS, { kind: 'only', names: ['help', 'measure', 'reset', 'time'] })
    expect(catapult.execute('/reset').kind).toBe('command')
    expect(catapult.execute('/set #latch.releaseAtHingeAngleDeg 80')).toEqual({ kind: 'error', message: { key: 'console.error.notAllowed', params: { name: 'set' } } })
    expect(catapult.commandNames()).toEqual(['help', 'measure', 'reset', 'time'])
  })

  it('declares a category and unique help/usage keys for every command', () => {
    const keys = consoleMessageKeys(M1_CONSOLE_COMMANDS)
    expect(new Set(keys).size).toBe(keys.length)
    for (const command of M1_CONSOLE_COMMANDS) {
      expect(['build', 'world', 'instruments', 'meta']).toContain(command.category)
      expect(keys).toEqual(expect.arrayContaining([`console.help.${command.name}`, `console.usage.${command.name}`]))
    }
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run --project console`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement**

`packages/console/src/console-types.ts`:
```ts
import type { Command } from '@physics-lab/formats'

export type Token =
  | { kind: 'number'; value: number; text: string }
  | { kind: 'ref'; id: string; property: string | null; text: string }
  | { kind: 'word'; text: string }
export type ConsoleMessage = { key: string; params: Readonly<Record<string, string | number>> }
export type UiAction = { type: 'showHelp'; command: string | null } | { type: 'setTimeScale'; factor: number }
export type ConsoleOutcome =
  | { kind: 'command'; command: Command }
  | { kind: 'ui'; action: UiAction }
  | { kind: 'error'; message: ConsoleMessage }
export type ConsoleCommandDefinition = {
  name: string
  category: 'build' | 'world' | 'instruments' | 'meta'
  parse(args: readonly Token[]): ConsoleOutcome
}
export type CommandPermissions = { kind: 'all' } | { kind: 'only'; names: readonly string[] }

export function consoleError(key: string, params: Readonly<Record<string, string | number>> = {}): ConsoleOutcome {
  return { kind: 'error', message: { key, params } }
}
```

`packages/console/src/tokenize.ts`:
```ts
import type { Token } from './console-types'

const NUMBER_PATTERN = /^-?\d+(?:\.\d+)?$/
const REF_PATTERN = /^#([a-z][a-z0-9-]{0,47})(?:\.([a-zA-Z][a-zA-Z0-9]{0,63}))?$/

export function tokenize(line: string): Token[] {
  const body = line.trim().replace(/^\//, '')
  return body
    .split(/\s+/)
    .filter((text) => text.length > 0)
    .map((text): Token => {
      if (NUMBER_PATTERN.test(text)) return { kind: 'number', value: Number(text), text }
      const ref = REF_PATTERN.exec(text)
      if (ref?.[1]) return { kind: 'ref', id: ref[1], property: ref[2] ?? null, text }
      return { kind: 'word', text }
    })
}
```

`packages/console/src/commands.ts`:
```ts
import { QuantityIdSchema } from '@physics-lab/formats'
import { consoleError, type ConsoleCommandDefinition, type ConsoleOutcome, type Token } from './console-types'

export const TIME_SCALE_RANGE = { min: 0.1, max: 4 } as const
export const CONSOLE_ERROR_KEYS = [
  'console.error.empty',
  'console.error.unknownCommand',
  'console.error.notAllowed',
  'console.error.usage',
  'console.error.outOfRange',
] as const

const usage = (command: string): ConsoleOutcome => consoleError('console.error.usage', { command })

const helpCommand: ConsoleCommandDefinition = {
  name: 'help',
  category: 'meta',
  parse(args) {
    const [topic, ...rest] = args
    if (rest.length > 0 || (topic && topic.kind !== 'word')) return usage('help')
    return { kind: 'ui', action: { type: 'showHelp', command: topic?.text ?? null } }
  },
}

const measureCommand: ConsoleCommandDefinition = {
  name: 'measure',
  category: 'instruments',
  parse(args) {
    const [quantity, ...rest] = args
    if (!quantity || rest.length > 0 || quantity.kind !== 'word' || !QuantityIdSchema.safeParse(quantity.text).success) return usage('measure')
    return { kind: 'command', command: { type: 'measure', quantity: quantity.text } }
  },
}

const resetCommand: ConsoleCommandDefinition = {
  name: 'reset',
  category: 'meta',
  parse: (args) => (args.length === 0 ? { kind: 'command', command: { type: 'reset' } } : usage('reset')),
}

const timeCommand: ConsoleCommandDefinition = {
  name: 'time',
  category: 'world',
  parse(args) {
    const [subcommand, factor, ...rest] = args
    if (subcommand?.kind !== 'word' || subcommand.text !== 'scale' || factor?.kind !== 'number' || rest.length > 0) return usage('time')
    if (factor.value < TIME_SCALE_RANGE.min || factor.value > TIME_SCALE_RANGE.max) {
      return consoleError('console.error.outOfRange', { min: TIME_SCALE_RANGE.min, max: TIME_SCALE_RANGE.max })
    }
    return { kind: 'ui', action: { type: 'setTimeScale', factor: factor.value } }
  },
}

const setCommand: ConsoleCommandDefinition = {
  name: 'set',
  category: 'build',
  parse(args: readonly Token[]) {
    const [target, value, ...rest] = args
    if (target?.kind !== 'ref' || target.property === null || value?.kind !== 'number' || rest.length > 0) return usage('set')
    return { kind: 'command', command: { type: 'setProperty', target: `#${target.id}`, property: target.property, value: value.value } }
  },
}

/** Registry pattern: each console command is one definition; adding a command means adding an entry (open/closed). */
export const M1_CONSOLE_COMMANDS: readonly ConsoleCommandDefinition[] = [helpCommand, measureCommand, resetCommand, timeCommand, setCommand]

export function consoleMessageKeys(commands: readonly ConsoleCommandDefinition[]): string[] {
  return [...commands.flatMap((command) => [`console.help.${command.name}`, `console.usage.${command.name}`]), ...CONSOLE_ERROR_KEYS]
}
```

`packages/console/src/create-console.ts`:
```ts
import { consoleError, type CommandPermissions, type ConsoleCommandDefinition, type ConsoleOutcome } from './console-types'
import { tokenize } from './tokenize'

export type GameConsole = { execute(line: string): ConsoleOutcome; commandNames(): string[] }

/** Parses a line into the same core commands the mouse UI produces; enforces per-challenge permissions (spec §9). */
export function createConsole(commands: readonly ConsoleCommandDefinition[], permissions: CommandPermissions): GameConsole {
  const byName = new Map(commands.map((command) => [command.name, command]))
  const isAllowed = (name: string): boolean => permissions.kind === 'all' || permissions.names.includes(name)
  return {
    execute(line) {
      const [head, ...args] = tokenize(line)
      if (!head) return consoleError('console.error.empty')
      const command = head.kind === 'word' ? byName.get(head.text) : undefined
      if (!command) return consoleError('console.error.unknownCommand', { name: head.text })
      if (!isAllowed(command.name)) return consoleError('console.error.notAllowed', { name: command.name })
      return command.parse(args)
    },
    commandNames() {
      return commands.map((command) => command.name).filter(isAllowed)
    },
  }
}
```

`packages/console/src/index.ts`:
```ts
export { CONSOLE_ERROR_KEYS, M1_CONSOLE_COMMANDS, TIME_SCALE_RANGE, consoleMessageKeys } from './commands'
export type { CommandPermissions, ConsoleCommandDefinition, ConsoleMessage, ConsoleOutcome, Token, UiAction } from './console-types'
export { createConsole, type GameConsole } from './create-console'
export { tokenize } from './tokenize'
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm vitest run --project console`
Expected: PASS.

- [ ] **Step 5: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test
git add packages/console
git commit -m "feat(console): add command parser with m1 commands and permissions"
```

---
### Task 15: `apps/web` — scaffold, theme tokens, i18n, CSP and the Home screen

**Files:**
- Create: `apps/web/package.json`, `index.html`, `vite.config.ts`, `vitest.config.ts`, `tsconfig.json`, `components.json`
- Create: `apps/web/src/main.tsx`, `src/vite-env.d.ts`, `src/app/App.tsx`, `src/app/dependencies.ts`, `src/index.css`, `src/lib/utils.ts`
- Create: `apps/web/src/theme/tokens.ts`, `theme/apply-theme.ts`, `theme/contrast.ts`
- Create: `apps/web/src/i18n/locales.ts`, `i18n/required-keys.ts`, `i18n/messages/en.json`, `i18n/messages/es.json`
- Create: `apps/web/src/home/HomeScreen.tsx`, `src/test/setup.ts`, `src/test/render.tsx`, `src/test/axe.ts`
- Generated: `apps/web/src/components/ui/{button,input,label,textarea,table}.tsx` (shadcn)
- Modify: `tools/eslint/lint-rules.test.ts`
- Test: `apps/web/src/theme/contrast.test.ts`, `theme/apply-theme.test.ts`, `i18n/messages.test.ts`, `i18n/locales.test.ts`, `home/HomeScreen.test.tsx`

**Interfaces:**
- Consumes: `normalizeStudentId`, `MAX_STUDENT_ID_CHARS`, `InstrumentIdSchema` (formats); `LAB_ERROR_CODES` (challenges); `FLIGHT_QUANTITIES` (sim-core); `M1_CONSOLE_COMMANDS`, `consoleMessageKeys` (console).
- Produces:
  - `type Theme`, `GREYBOX_THEME` (`ui.*` colors named like shadcn tokens, `scene.sky`, `scene.materials[MaterialId]`, `scene.projectile`), `applyThemeCssVariables(element, theme)` (sets `--pl-ui-<kebab-name>`), `contrastRatio(hexA, hexB)`
  - `SUPPORTED_LOCALES`, `type Locale`, `MESSAGES`, `detectLocale(languages)`
  - `WORKER_ERROR_CODES = ['notInitialized', 'internal']`, `APP_ERROR_CODES`, `requiredMessageKeys(): string[]`
  - `type StartRequest = { studentId: string; seed: number }`, `HomeScreen` props `{ locale, onLocaleChange, onStart, randomSeed }`, `parseSeed(text): number | null | 'invalid'`
  - `type AppDependencies = { appVersion: string; randomSeed(): number }` (extended in Tasks 16–19), `App` props `{ dependencies, initialLocale }`
  - Test helpers: `renderWithIntl(ui, locale?)`, `expectNoAxeViolations(container)`

- [ ] **Step 1: Create the app and install dependencies**

`apps/web/package.json`:
```json
{
  "name": "@physics-lab/web",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview --port 4173 --strictPort",
    "typecheck": "tsc --noEmit -p tsconfig.json",
    "test:e2e": "playwright test"
  }
}
```

```bash
cd apps/web
pnpm add react react-dom react-intl three zod class-variance-authority clsx tailwind-merge lucide-react radix-ui \
  @physics-lab/formats@workspace:* @physics-lab/det-math@workspace:* @physics-lab/sim-core@workspace:* \
  @physics-lab/instruments@workspace:* @physics-lab/console@workspace:* @physics-lab/challenges@workspace:*
pnpm add -D vite @vitejs/plugin-react tailwindcss @tailwindcss/vite @types/react @types/react-dom @types/three \
  jsdom @testing-library/react @testing-library/user-event @testing-library/jest-dom axe-core \
  @formatjs/icu-messageformat-parser @playwright/test @axe-core/playwright
cd ../..
```

`apps/web/tsconfig.json`:
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "lib": ["ES2023", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "types": ["vite/client", "node"],
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src", "e2e", "vite.config.ts", "vitest.config.ts", "playwright.config.ts"]
}
```

`apps/web/vite.config.ts`:
```ts
import { fileURLToPath } from 'node:url'
import tailwindcss from '@tailwindcss/vite'
import react from '@vitejs/plugin-react'
import { defineConfig, type Plugin } from 'vite'
import packageJson from './package.json'

/** D41: WASM needs 'wasm-unsafe-eval' (never 'unsafe-eval'); no third-party origins (D20). Header CSP is F16. */
const CONTENT_SECURITY_POLICY = [
  "default-src 'self'",
  "script-src 'self' 'wasm-unsafe-eval'",
  "worker-src 'self'",
  "style-src 'self'",
  "img-src 'self' data: blob:",
  "font-src 'self'",
  "connect-src 'self'",
  "object-src 'none'",
  "base-uri 'self'",
  "form-action 'self'",
].join('; ')

function contentSecurityPolicy(): Plugin {
  return {
    name: 'physics-lab-csp',
    apply: 'build',
    transformIndexHtml: () => [
      { tag: 'meta', attrs: { 'http-equiv': 'Content-Security-Policy', content: CONTENT_SECURITY_POLICY }, injectTo: 'head-prepend' },
    ],
  }
}

export default defineConfig({
  plugins: [react(), tailwindcss(), contentSecurityPolicy()],
  resolve: { alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) } },
  define: { __APP_VERSION__: JSON.stringify(packageJson.version) },
  worker: { format: 'es' },
})
```

`apps/web/vitest.config.ts`:
```ts
import { fileURLToPath } from 'node:url'
import react from '@vitejs/plugin-react'
import { defineProject } from 'vitest/config'

export default defineProject({
  plugins: [react()],
  resolve: { alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) } },
  define: { __APP_VERSION__: JSON.stringify('0.0.0-test') },
  test: { name: 'web', environment: 'jsdom', include: ['src/**/*.test.{ts,tsx}'], setupFiles: ['src/test/setup.ts'] },
})
```

`apps/web/src/vite-env.d.ts`:
```ts
/// <reference types="vite/client" />
declare const __APP_VERSION__: string
```

`apps/web/index.html`:
```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Physics Lab</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

`apps/web/components.json`:
```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": false,
  "tsx": true,
  "tailwind": { "config": "", "css": "src/index.css", "baseColor": "neutral", "cssVariables": true, "prefix": "" },
  "aliases": { "components": "@/components", "utils": "@/lib/utils", "ui": "@/components/ui", "lib": "@/lib", "hooks": "@/hooks" },
  "iconLibrary": "lucide"
}
```

`apps/web/src/lib/utils.ts`:
```ts
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]): string {
  return twMerge(clsx(inputs))
}
```

Generate the shadcn components, then **overwrite `src/index.css` with Step 4's content** (shadcn writes its own color variables there; colors must come only from our theme tokens):
```bash
cd apps/web && pnpm dlx shadcn@latest add button input label textarea table && cd ../..
```

- [ ] **Step 2: Write the failing tests**

`apps/web/src/theme/contrast.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { contrastRatio } from './contrast'
import { GREYBOX_THEME } from './tokens'

const AA_TEXT = 4.5
const AA_NON_TEXT = 3
const ui = GREYBOX_THEME.ui

describe('theme contrast (WCAG 2.2 AA)', () => {
  it('computes the reference ratio for black on white', () => {
    expect(contrastRatio('#000000', '#ffffff')).toBeCloseTo(21, 5)
  })

  it.each([
    ['foreground', ui.foreground, ui.background],
    ['card text', ui.cardForeground, ui.card],
    ['muted text on background', ui.mutedForeground, ui.background],
    ['muted text on muted', ui.mutedForeground, ui.muted],
    ['primary button text', ui.primaryForeground, ui.primary],
    ['secondary button text', ui.secondaryForeground, ui.secondary],
    ['accent text', ui.accentForeground, ui.accent],
    ['error text', ui.destructive, ui.background],
    ['success text', ui.success, ui.background],
  ])('%s reaches 4.5:1', (_, foreground, background) => {
    expect(contrastRatio(foreground, background)).toBeGreaterThanOrEqual(AA_TEXT)
  })

  it.each([
    ['focus ring', ui.ring, ui.background],
    ['input border', ui.input, ui.background],
  ])('%s reaches 3:1', (_, foreground, background) => {
    expect(contrastRatio(foreground, background)).toBeGreaterThanOrEqual(AA_NON_TEXT)
  })
})
```

`apps/web/src/theme/apply-theme.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { applyThemeCssVariables } from './apply-theme'
import { GREYBOX_THEME } from './tokens'

describe('applyThemeCssVariables', () => {
  it('exposes every UI token as a kebab-case CSS variable', () => {
    const element = document.createElement('div')
    applyThemeCssVariables(element, GREYBOX_THEME)
    expect(element.style.getPropertyValue('--pl-ui-primary-foreground')).toBe(GREYBOX_THEME.ui.primaryForeground)
    expect(element.style.getPropertyValue('--pl-ui-background')).toBe(GREYBOX_THEME.ui.background)
  })
})
```

`apps/web/src/i18n/locales.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { detectLocale } from './locales'

describe('detectLocale', () => {
  it('picks the first supported language', () => {
    expect(detectLocale(['es-MX', 'en-US'])).toBe('es')
    expect(detectLocale(['fr-FR', 'en-GB', 'es'])).toBe('en')
  })

  it('falls back to English', () => {
    expect(detectLocale(['de-DE'])).toBe('en')
    expect(detectLocale([])).toBe('en')
  })
})
```

`apps/web/src/i18n/messages.test.ts`:
```ts
import { parse } from '@formatjs/icu-messageformat-parser'
import { describe, expect, it } from 'vitest'
import en from './messages/en.json'
import es from './messages/es.json'
import { requiredMessageKeys } from './required-keys'

const catalogs = { en, es } as const

describe('i18n catalogs (D9, CI gate)', () => {
  it('have exactly the same keys in es and en', () => {
    expect(Object.keys(es).sort()).toEqual(Object.keys(en).sort())
  })

  it.each(Object.entries(catalogs))('%s contains every key required by the core packages', (_, catalog) => {
    const missing = requiredMessageKeys().filter((key) => !(key in catalog))
    expect(missing).toEqual([])
  })

  it.each(Object.entries(catalogs))('%s messages are valid ICU', (_, catalog) => {
    for (const [key, message] of Object.entries(catalog)) {
      expect(() => parse(message), key).not.toThrow()
    }
  })

  it.each(Object.entries(catalogs))('%s has no empty messages', (_, catalog) => {
    expect(Object.entries(catalog).filter(([, message]) => message.trim() === '')).toEqual([])
  })
})
```

`apps/web/src/home/HomeScreen.test.tsx`:
```tsx
import { screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, expect, it, vi } from 'vitest'
import { expectNoAxeViolations } from '@/test/axe'
import { renderWithIntl } from '@/test/render'
import { HomeScreen, parseSeed } from './HomeScreen'

function setup(locale: 'en' | 'es' = 'en') {
  const onStart = vi.fn()
  const onLocaleChange = vi.fn()
  const view = renderWithIntl(<HomeScreen locale={locale} onLocaleChange={onLocaleChange} onStart={onStart} randomSeed={() => 4242} />, locale)
  return { onStart, onLocaleChange, view }
}

describe('HomeScreen', () => {
  it('starts with a trimmed, NFC-normalized student id and a random seed when none is given', async () => {
    const { onStart } = setup()
    await userEvent.type(screen.getByLabelText('Student name or ID'), '  José  ')
    await userEvent.click(screen.getByRole('button', { name: 'Start the catapult challenge' }))
    expect(onStart).toHaveBeenCalledWith({ studentId: 'José', seed: 4242 })
  })

  it('uses the typed seed', async () => {
    const { onStart } = setup()
    await userEvent.type(screen.getByLabelText('Student name or ID'), 'ana')
    await userEvent.type(screen.getByLabelText('Seed (optional)'), '12345')
    await userEvent.click(screen.getByRole('button', { name: 'Start the catapult challenge' }))
    expect(onStart).toHaveBeenCalledWith({ studentId: 'ana', seed: 12345 })
  })

  it('shows an error, marks the field invalid and moves focus when the id is empty', async () => {
    const { onStart } = setup()
    await userEvent.click(screen.getByRole('button', { name: 'Start the catapult challenge' }))
    const field = screen.getByLabelText('Student name or ID')
    expect(onStart).not.toHaveBeenCalled()
    expect(field).toHaveAttribute('aria-invalid', 'true')
    expect(field).toHaveFocus()
    expect(screen.getByText('Enter your name or student ID.')).toBeInTheDocument()
  })

  it('rejects an invalid seed', async () => {
    const { onStart } = setup()
    await userEvent.type(screen.getByLabelText('Student name or ID'), 'ana')
    await userEvent.type(screen.getByLabelText('Seed (optional)'), '-5')
    await userEvent.click(screen.getByRole('button', { name: 'Start the catapult challenge' }))
    expect(onStart).not.toHaveBeenCalled()
    expect(screen.getByLabelText('Seed (optional)')).toHaveAttribute('aria-invalid', 'true')
  })

  it('renders in Spanish and reports language changes', async () => {
    const { onLocaleChange } = setup('es')
    expect(screen.getByLabelText('Nombre o ID de estudiante')).toBeInTheDocument()
    await userEvent.selectOptions(screen.getByLabelText('Idioma'), 'en')
    expect(onLocaleChange).toHaveBeenCalledWith('en')
  })

  it('has no axe violations', async () => {
    const { view } = setup()
    await expectNoAxeViolations(view.container)
  })
})

describe('parseSeed', () => {
  it('accepts blank, uint32 and rejects everything else', () => {
    expect(parseSeed('  ')).toBeNull()
    expect(parseSeed('4294967295')).toBe(4294967295)
    expect(parseSeed('4294967296')).toBe('invalid')
    expect(parseSeed('1.5')).toBe('invalid')
    expect(parseSeed('abc')).toBe('invalid')
  })
})
```

Add to `tools/eslint/lint-rules.test.ts`:
```ts
describe('theme tokens (D22)', () => {
  it('rejects hard-coded colors in the web app', async () => {
    const ids = await ruleIdsForProbe('apps/web/src', "export const color = '#ff0000'\n")
    expect(ids).toContain('no-restricted-syntax')
  })

  it('allows colors inside the theme layer', async () => {
    const ids = await ruleIdsForProbe('apps/web/src/theme', "export const color = '#ff0000'\n")
    expect(ids).not.toContain('no-restricted-syntax')
  })
})
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `pnpm vitest run --project web --project tools`
Expected: FAIL — theme, i18n and home modules not found (the two new tools tests fail until `apps/web/src` exists with the config applied).

- [ ] **Step 4: Implement theme, CSS and i18n**

`apps/web/src/theme/tokens.ts`:
```ts
import type { MaterialId } from '@physics-lab/formats'

export type UiTokens = {
  background: string; foreground: string; card: string; cardForeground: string
  primary: string; primaryForeground: string; secondary: string; secondaryForeground: string
  muted: string; mutedForeground: string; accent: string; accentForeground: string
  destructive: string; destructiveForeground: string; success: string
  border: string; input: string; ring: string
}
export type SceneTokens = { sky: string; light: string; materials: Readonly<Record<MaterialId, string>>; projectile: string; ambientIntensity: number; sunIntensity: number }
export type Theme = { ui: UiTokens; scene: SceneTokens }

/** Greybox theme (D22): the only place colors live; final art direction (F11) replaces this object. */
export const GREYBOX_THEME: Theme = {
  ui: {
    background: '#f5f5f4', foreground: '#1c1917', card: '#ffffff', cardForeground: '#1c1917',
    primary: '#1d4ed8', primaryForeground: '#ffffff', secondary: '#e7e5e4', secondaryForeground: '#1c1917',
    muted: '#e7e5e4', mutedForeground: '#44403c', accent: '#dbeafe', accentForeground: '#1e3a8a',
    destructive: '#b91c1c', destructiveForeground: '#ffffff', success: '#15803d',
    border: '#a8a29e', input: '#78716c', ring: '#1d4ed8',
  },
  scene: {
    sky: '#dfe7ee',
    light: '#ffffff',
    materials: { stone: '#9ca3af', wood: '#b07a4b', metal: '#6b7280', rubber: '#374151', ice: '#bae6fd' },
    projectile: '#c2410c',
    ambientIntensity: 0.9,
    sunIntensity: 1.6,
  },
}
```

`apps/web/src/theme/contrast.ts`:
```ts
const HEX_COLOR = /^#([0-9a-f]{2})([0-9a-f]{2})([0-9a-f]{2})$/i
const SRGB_LINEAR_THRESHOLD = 0.03928
const LUMINANCE_OFFSET = 0.05

function channelToLinear(channel: number): number {
  const c = channel / 255
  return c <= SRGB_LINEAR_THRESHOLD ? c / 12.92 : Math.pow((c + 0.055) / 1.055, 2.4)
}

function relativeLuminance(hex: string): number {
  const match = HEX_COLOR.exec(hex)
  if (!match) throw new Error(`Expected #rrggbb, got "${hex}"`)
  const [r, g, b] = [match[1], match[2], match[3]].map((part) => channelToLinear(Number.parseInt(part ?? '0', 16)))
  return 0.2126 * (r ?? 0) + 0.7152 * (g ?? 0) + 0.0722 * (b ?? 0)
}

/** WCAG 2.x contrast ratio between two #rrggbb colors. */
export function contrastRatio(a: string, b: string): number {
  const [light, dark] = [relativeLuminance(a), relativeLuminance(b)].sort((x, y) => y - x)
  return ((light ?? 0) + LUMINANCE_OFFSET) / ((dark ?? 0) + LUMINANCE_OFFSET)
}
```

`apps/web/src/theme/apply-theme.ts`:
```ts
import type { Theme } from './tokens'

const toKebabCase = (name: string): string => name.replace(/[A-Z]/g, (letter) => `-${letter.toLowerCase()}`)

/** Publishes UI tokens as CSS custom properties consumed by Tailwind's @theme mapping in index.css. */
export function applyThemeCssVariables(element: HTMLElement, theme: Theme): void {
  for (const [name, value] of Object.entries(theme.ui)) element.style.setProperty(`--pl-ui-${toKebabCase(name)}`, value)
}
```

`apps/web/src/index.css`:
```css
@import 'tailwindcss';

@theme inline {
  --color-background: var(--pl-ui-background);
  --color-foreground: var(--pl-ui-foreground);
  --color-card: var(--pl-ui-card);
  --color-card-foreground: var(--pl-ui-card-foreground);
  --color-popover: var(--pl-ui-card);
  --color-popover-foreground: var(--pl-ui-card-foreground);
  --color-primary: var(--pl-ui-primary);
  --color-primary-foreground: var(--pl-ui-primary-foreground);
  --color-secondary: var(--pl-ui-secondary);
  --color-secondary-foreground: var(--pl-ui-secondary-foreground);
  --color-muted: var(--pl-ui-muted);
  --color-muted-foreground: var(--pl-ui-muted-foreground);
  --color-accent: var(--pl-ui-accent);
  --color-accent-foreground: var(--pl-ui-accent-foreground);
  --color-destructive: var(--pl-ui-destructive);
  --color-destructive-foreground: var(--pl-ui-destructive-foreground);
  --color-success: var(--pl-ui-success);
  --color-border: var(--pl-ui-border);
  --color-input: var(--pl-ui-input);
  --color-ring: var(--pl-ui-ring);
}

@layer base {
  html {
    font-family: system-ui, sans-serif;
    font-size: clamp(1rem, 0.95rem + 0.25vw, 1.125rem);
  }
  body {
    @apply bg-background text-foreground;
  }
  :focus-visible {
    outline: 3px solid var(--pl-ui-ring);
    outline-offset: 2px;
  }
}

@media (prefers-reduced-motion: reduce) {
  *,
  ::before,
  ::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

`apps/web/src/i18n/locales.ts`:
```ts
import en from './messages/en.json'
import es from './messages/es.json'

export const SUPPORTED_LOCALES = ['es', 'en'] as const
export type Locale = (typeof SUPPORTED_LOCALES)[number]
export const DEFAULT_LOCALE: Locale = 'en'
export const MESSAGES: Readonly<Record<Locale, Readonly<Record<string, string>>>> = { es, en }

export function detectLocale(languages: readonly string[]): Locale {
  for (const language of languages) {
    const match = SUPPORTED_LOCALES.find((locale) => language.toLowerCase().startsWith(locale))
    if (match) return match
  }
  return DEFAULT_LOCALE
}
```

`apps/web/src/i18n/required-keys.ts`:
```ts
import { LAB_ERROR_CODES } from '@physics-lab/challenges'
import { M1_CONSOLE_COMMANDS, consoleMessageKeys } from '@physics-lab/console'
import { InstrumentIdSchema } from '@physics-lab/formats'
import { FLIGHT_QUANTITIES } from '@physics-lab/sim-core'

export const WORKER_ERROR_CODES = ['notInitialized', 'internal'] as const
export const APP_ERROR_CODES = [...LAB_ERROR_CODES, ...WORKER_ERROR_CODES] as const
export type AppErrorCode = (typeof APP_ERROR_CODES)[number]

/** Ratchet (D39): every key the core packages can emit must exist in both catalogs. */
export function requiredMessageKeys(): string[] {
  return [
    ...consoleMessageKeys(M1_CONSOLE_COMMANDS),
    ...APP_ERROR_CODES.map((code) => `error.${code}`),
    ...FLIGHT_QUANTITIES.map((quantity) => `quantity.${quantity}`),
    ...InstrumentIdSchema.options.map((instrument) => `instrument.${instrument}`),
  ]
}
```

`apps/web/src/i18n/messages/en.json`:
```json
{
  "app.title": "Physics Lab",
  "app.loading": "Loading the lab…",
  "home.intro": "A digital physics laboratory: build, predict, launch and measure.",
  "home.language.label": "Language",
  "home.language.es": "Español",
  "home.language.en": "English",
  "home.challenge.heading": "Catapult challenge",
  "home.challenge.description": "Measure how a catapult releases a ball, predict how far it flies, then test your prediction.",
  "home.studentId.label": "Student name or ID",
  "home.studentId.help": "Used only in the result file you download. Nothing is sent anywhere.",
  "home.studentId.error.empty": "Enter your name or student ID.",
  "home.studentId.error.tooLong": "Use at most {max} characters.",
  "home.seed.label": "Seed (optional)",
  "home.seed.help": "Leave empty unless your teacher gave you a seed.",
  "home.seed.error": "The seed must be a whole number from 0 to 4294967295.",
  "home.start": "Start the catapult challenge",
  "console.help.help": "Lists the commands, or explains one command.",
  "console.usage.help": "/help [command]",
  "console.help.measure": "Measures a quantity with its instrument, for example /measure flight.range.",
  "console.usage.measure": "/measure <sensor.quantity>",
  "console.help.reset": "Returns the experiment to its starting state.",
  "console.usage.reset": "/reset",
  "console.help.time": "Changes the simulation speed: 0.1 is slow motion, 1 is real time, 4 is fast forward.",
  "console.usage.time": "/time scale <0.1–4>",
  "console.help.set": "Changes a property of an object, for example /set #latch.releaseAtHingeAngleDeg 80.",
  "console.usage.set": "/set #<id>.<property> <number>",
  "console.error.empty": "Type a command, for example /help.",
  "console.error.unknownCommand": "Unknown command /{name}. Type /help to see the list.",
  "console.error.notAllowed": "/{name} is not allowed in this challenge.",
  "console.error.usage": "That is not how /{command} is used. Type /help {command}.",
  "console.error.outOfRange": "The value must be between {min} and {max}.",
  "error.quantityUnavailable": "That quantity cannot be measured yet.",
  "error.unknownQuantity": "That quantity does not exist in this experiment.",
  "error.stepInPast": "The recorded steps are out of order.",
  "error.runTooLong": "The run reached its maximum length. Reset to try again.",
  "error.commandNotSupported": "That command cannot be used while the experiment runs.",
  "error.invalidPhase": "That action is not available right now.",
  "error.predictionRequired": "Submit your prediction first.",
  "error.predictionLocked": "Your prediction is locked because you already measured the result.",
  "error.unknownTask": "That task does not exist.",
  "error.explanationTooLong": "Your explanation is too long (at most {max} characters).",
  "error.challengeData": "This challenge file is invalid.",
  "error.invalidValue": "That value is not valid.",
  "error.notInitialized": "The lab is still loading. Try again in a moment.",
  "error.internal": "Something went wrong in the simulation. Reload the page to start again.",
  "quantity.releaseSpeed": "Release speed",
  "quantity.releaseAngle": "Release angle",
  "quantity.releaseHeight": "Release height above the ground",
  "quantity.range": "Horizontal range",
  "quantity.time": "Flight time",
  "instrument.ruler": "Measuring tape",
  "instrument.stopwatch": "Stopwatch",
  "instrument.photogate": "Photogate",
  "instrument.protractor": "Protractor",
  "instrument.forceProbe": "Force probe",
  "instrument.massScale": "Mass scale"
}
```

`apps/web/src/i18n/messages/es.json`:
```json
{
  "app.title": "Physics Lab",
  "app.loading": "Cargando el laboratorio…",
  "home.intro": "Un laboratorio de física digital: construye, predice, lanza y mide.",
  "home.language.label": "Idioma",
  "home.language.es": "Español",
  "home.language.en": "English",
  "home.challenge.heading": "Reto de la catapulta",
  "home.challenge.description": "Mide cómo una catapulta suelta una bola, predice hasta dónde vuela y luego pon a prueba tu predicción.",
  "home.studentId.label": "Nombre o ID de estudiante",
  "home.studentId.help": "Solo se usa en el archivo de resultados que descargas. No se envía nada a ningún sitio.",
  "home.studentId.error.empty": "Escribe tu nombre o ID de estudiante.",
  "home.studentId.error.tooLong": "Usa como máximo {max} caracteres.",
  "home.seed.label": "Semilla (opcional)",
  "home.seed.help": "Déjala vacía salvo que tu docente te haya dado una semilla.",
  "home.seed.error": "La semilla debe ser un número entero de 0 a 4294967295.",
  "home.start": "Empezar el reto de la catapulta",
  "console.help.help": "Muestra los comandos o explica uno.",
  "console.usage.help": "/help [comando]",
  "console.help.measure": "Mide una magnitud con su instrumento, por ejemplo /measure flight.range.",
  "console.usage.measure": "/measure <sensor.magnitud>",
  "console.help.reset": "Devuelve el experimento a su estado inicial.",
  "console.usage.reset": "/reset",
  "console.help.time": "Cambia la velocidad de la simulación: 0.1 es cámara lenta, 1 es tiempo real, 4 es avance rápido.",
  "console.usage.time": "/time scale <0.1–4>",
  "console.help.set": "Cambia una propiedad de un objeto, por ejemplo /set #latch.releaseAtHingeAngleDeg 80.",
  "console.usage.set": "/set #<id>.<propiedad> <número>",
  "console.error.empty": "Escribe un comando, por ejemplo /help.",
  "console.error.unknownCommand": "Comando desconocido /{name}. Escribe /help para ver la lista.",
  "console.error.notAllowed": "/{name} no está permitido en este reto.",
  "console.error.usage": "Así no se usa /{command}. Escribe /help {command}.",
  "console.error.outOfRange": "El valor debe estar entre {min} y {max}.",
  "error.quantityUnavailable": "Esa magnitud todavía no se puede medir.",
  "error.unknownQuantity": "Esa magnitud no existe en este experimento.",
  "error.stepInPast": "Los pasos registrados están desordenados.",
  "error.runTooLong": "La ejecución alcanzó su duración máxima. Reinicia para intentarlo de nuevo.",
  "error.commandNotSupported": "Ese comando no se puede usar mientras corre el experimento.",
  "error.invalidPhase": "Esa acción no está disponible ahora.",
  "error.predictionRequired": "Primero envía tu predicción.",
  "error.predictionLocked": "Tu predicción está bloqueada porque ya mediste el resultado.",
  "error.unknownTask": "Esa tarea no existe.",
  "error.explanationTooLong": "Tu explicación es demasiado larga (máximo {max} caracteres).",
  "error.challengeData": "Este archivo de reto no es válido.",
  "error.invalidValue": "Ese valor no es válido.",
  "error.notInitialized": "El laboratorio aún se está cargando. Inténtalo de nuevo en un momento.",
  "error.internal": "Algo salió mal en la simulación. Recarga la página para empezar de nuevo.",
  "quantity.releaseSpeed": "Rapidez de liberación",
  "quantity.releaseAngle": "Ángulo de liberación",
  "quantity.releaseHeight": "Altura de liberación sobre el suelo",
  "quantity.range": "Alcance horizontal",
  "quantity.time": "Tiempo de vuelo",
  "instrument.ruler": "Cinta métrica",
  "instrument.stopwatch": "Cronómetro",
  "instrument.photogate": "Fotopuerta",
  "instrument.protractor": "Transportador",
  "instrument.forceProbe": "Sensor de fuerza",
  "instrument.massScale": "Balanza"
}
```

- [ ] **Step 5: Implement test helpers, Home, App and the composition root**

`apps/web/src/test/setup.ts`:
```ts
import '@testing-library/jest-dom/vitest'
```

`apps/web/src/test/render.tsx`:
```tsx
import { render, type RenderResult } from '@testing-library/react'
import type { ReactElement } from 'react'
import { IntlProvider } from 'react-intl'
import { MESSAGES, type Locale } from '@/i18n/locales'

export function renderWithIntl(ui: ReactElement, locale: Locale = 'en'): RenderResult {
  return render(
    <IntlProvider locale={locale} messages={MESSAGES[locale]}>
      {ui}
    </IntlProvider>
  )
}
```

`apps/web/src/test/axe.ts`:
```ts
import axe from 'axe-core'
import { expect } from 'vitest'

/** Automated WCAG checks (D27). jsdom cannot compute color contrast; contrast is covered by contrast.test.ts. */
export async function expectNoAxeViolations(container: Element): Promise<void> {
  const results = await axe.run(container, { rules: { 'color-contrast': { enabled: false } } })
  expect(results.violations.map((violation) => `${violation.id}: ${violation.help}`)).toEqual([])
}
```

`apps/web/src/home/HomeScreen.tsx`:
```tsx
import { MAX_STUDENT_ID_CHARS, normalizeStudentId } from '@physics-lab/formats'
import { useId, useRef, useState, type FormEvent } from 'react'
import { useIntl } from 'react-intl'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import { SUPPORTED_LOCALES, type Locale } from '@/i18n/locales'

const MAX_SEED = 0xffff_ffff
const SEED_PATTERN = /^\d{1,10}$/

export type StartRequest = { studentId: string; seed: number }
type HomeScreenProps = {
  locale: Locale
  onLocaleChange(locale: Locale): void
  onStart(request: StartRequest): void
  randomSeed(): number
}
type FormErrors = { studentId?: string; seed?: string }

export function parseSeed(text: string): number | null | 'invalid' {
  const trimmed = text.trim()
  if (trimmed === '') return null
  if (!SEED_PATTERN.test(trimmed)) return 'invalid'
  const value = Number(trimmed)
  return value <= MAX_SEED ? value : 'invalid'
}

function isLocale(value: string): value is Locale {
  return (SUPPORTED_LOCALES as readonly string[]).includes(value)
}

export function HomeScreen({ locale, onLocaleChange, onStart, randomSeed }: HomeScreenProps) {
  const intl = useIntl()
  const [studentId, setStudentId] = useState('')
  const [seedText, setSeedText] = useState('')
  const [errors, setErrors] = useState<FormErrors>({})
  const studentIdRef = useRef<HTMLInputElement>(null)
  const seedRef = useRef<HTMLInputElement>(null)
  const id = useId()
  const ids = {
    language: `${id}-language`, studentId: `${id}-student`, studentIdHelp: `${id}-student-help`, studentIdError: `${id}-student-error`,
    seed: `${id}-seed`, seedHelp: `${id}-seed-help`, seedError: `${id}-seed-error`, heading: `${id}-heading`,
  }

  function validateStudentId(normalized: string): string | undefined {
    if (normalized.length === 0) return intl.formatMessage({ id: 'home.studentId.error.empty' })
    if (normalized.length > MAX_STUDENT_ID_CHARS) return intl.formatMessage({ id: 'home.studentId.error.tooLong' }, { max: MAX_STUDENT_ID_CHARS })
    return undefined
  }

  function handleSubmit(event: FormEvent<HTMLFormElement>): void {
    event.preventDefault()
    const normalized = normalizeStudentId(studentId)
    const seed = parseSeed(seedText)
    const studentIdError = validateStudentId(normalized)
    const seedError = seed === 'invalid' ? intl.formatMessage({ id: 'home.seed.error' }) : undefined
    setErrors({ ...(studentIdError ? { studentId: studentIdError } : {}), ...(seedError ? { seed: seedError } : {}) })
    if (studentIdError) return studentIdRef.current?.focus()
    if (seed === 'invalid') return seedRef.current?.focus()
    onStart({ studentId: normalized, seed: seed ?? randomSeed() })
  }

  const describedBy = (help: string, error: string, hasError: boolean): string => (hasError ? `${help} ${error}` : help)

  return (
    <main className="mx-auto flex min-h-dvh w-full max-w-prose flex-col gap-8 px-4 py-8 sm:px-6">
      <header className="flex flex-wrap items-center justify-between gap-4">
        <h1 className="text-3xl font-bold">{intl.formatMessage({ id: 'app.title' })}</h1>
        <div className="flex items-center gap-2">
          <Label htmlFor={ids.language}>{intl.formatMessage({ id: 'home.language.label' })}</Label>
          <select
            id={ids.language}
            className="min-h-11 rounded-md border border-input bg-card px-3"
            value={locale}
            onChange={(event) => isLocale(event.target.value) && onLocaleChange(event.target.value)}
          >
            {SUPPORTED_LOCALES.map((option) => (
              <option key={option} value={option}>
                {intl.formatMessage({ id: `home.language.${option}` })}
              </option>
            ))}
          </select>
        </div>
      </header>
      <p className="text-lg">{intl.formatMessage({ id: 'home.intro' })}</p>
      <form noValidate onSubmit={handleSubmit} aria-labelledby={ids.heading} className="flex flex-col gap-6 rounded-lg bg-card p-4 sm:p-6">
        <h2 id={ids.heading} className="text-2xl font-semibold">{intl.formatMessage({ id: 'home.challenge.heading' })}</h2>
        <p>{intl.formatMessage({ id: 'home.challenge.description' })}</p>
        <div className="flex flex-col gap-2">
          <Label htmlFor={ids.studentId}>{intl.formatMessage({ id: 'home.studentId.label' })}</Label>
          <Input
            ref={studentIdRef}
            id={ids.studentId}
            autoComplete="off"
            className="min-h-11"
            value={studentId}
            onChange={(event) => setStudentId(event.target.value)}
            aria-invalid={errors.studentId !== undefined}
            aria-describedby={describedBy(ids.studentIdHelp, ids.studentIdError, errors.studentId !== undefined)}
          />
          <p id={ids.studentIdHelp} className="text-sm text-muted-foreground">{intl.formatMessage({ id: 'home.studentId.help' })}</p>
          {errors.studentId && <p id={ids.studentIdError} className="text-sm font-medium text-destructive">{errors.studentId}</p>}
        </div>
        <div className="flex flex-col gap-2">
          <Label htmlFor={ids.seed}>{intl.formatMessage({ id: 'home.seed.label' })}</Label>
          <Input
            ref={seedRef}
            id={ids.seed}
            inputMode="numeric"
            autoComplete="off"
            className="min-h-11"
            value={seedText}
            onChange={(event) => setSeedText(event.target.value)}
            aria-invalid={errors.seed !== undefined}
            aria-describedby={describedBy(ids.seedHelp, ids.seedError, errors.seed !== undefined)}
          />
          <p id={ids.seedHelp} className="text-sm text-muted-foreground">{intl.formatMessage({ id: 'home.seed.help' })}</p>
          {errors.seed && <p id={ids.seedError} className="text-sm font-medium text-destructive">{errors.seed}</p>}
        </div>
        <Button type="submit" className="min-h-11 self-start">{intl.formatMessage({ id: 'home.start' })}</Button>
      </form>
    </main>
  )
}
```

`apps/web/src/app/dependencies.ts`:
```ts
/** Ports the UI needs; concrete adapters are created only in main.tsx (composition root, D30). */
export type AppDependencies = {
  appVersion: string
  randomSeed(): number
}
```

`apps/web/src/app/App.tsx` (the world route replaces the loading branch in Task 18):
```tsx
import { useEffect, useState } from 'react'
import { IntlProvider, FormattedMessage } from 'react-intl'
import { HomeScreen, type StartRequest } from '@/home/HomeScreen'
import { MESSAGES, type Locale } from '@/i18n/locales'
import type { AppDependencies } from './dependencies'

type AppProps = { dependencies: AppDependencies; initialLocale: Locale }

export function App({ dependencies, initialLocale }: AppProps) {
  const [locale, setLocale] = useState<Locale>(initialLocale)
  const [started, setStarted] = useState<StartRequest | null>(null)

  useEffect(() => {
    document.documentElement.lang = locale
  }, [locale])

  return (
    <IntlProvider locale={locale} messages={MESSAGES[locale]}>
      {started ? (
        <p role="status" className="p-8"><FormattedMessage id="app.loading" /></p>
      ) : (
        <HomeScreen locale={locale} onLocaleChange={setLocale} onStart={setStarted} randomSeed={dependencies.randomSeed} />
      )}
    </IntlProvider>
  )
}
```

`apps/web/src/main.tsx`:
```tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { z } from 'zod'
import { App } from '@/app/App'
import type { AppDependencies } from '@/app/dependencies'
import { detectLocale } from '@/i18n/locales'
import { applyThemeCssVariables } from '@/theme/apply-theme'
import { GREYBOX_THEME } from '@/theme/tokens'
import './index.css'

// Composition root (D30): the only place concrete adapters are created and wired.
z.config({ jitless: true }) // D41: no Function() probe under the CSP.
applyThemeCssVariables(document.documentElement, GREYBOX_THEME)

const dependencies: AppDependencies = {
  appVersion: __APP_VERSION__,
  randomSeed: () => crypto.getRandomValues(new Uint32Array(1))[0] ?? 0,
}

const root = document.getElementById('root')
if (!root) throw new Error('Missing #root element')
createRoot(root).render(
  <StrictMode>
    <App dependencies={dependencies} initialLocale={detectLocale(navigator.languages)} />
  </StrictMode>
)
```

- [ ] **Step 6: Run the tests to verify they pass**

Run: `pnpm vitest run --project web --project tools`
Expected: PASS.

- [ ] **Step 7: Verify, including the production build and CSP**

```bash
pnpm lint && pnpm typecheck && pnpm test
pnpm --filter @physics-lab/web build
grep -c "Content-Security-Policy" apps/web/dist/index.html        # expected: 1
grep -rlE '\beval\(|new Function\(' apps/web/dist/assets || echo "no eval"   # expected: no eval
```

- [ ] **Step 8: Commit**

```bash
git add apps/web tools/eslint/lint-rules.test.ts pnpm-lock.yaml
git commit -m "feat(web): add app shell with theme tokens, i18n, csp and home screen"
```

---

### Task 16: `apps/web` — simulation worker: protocol, handler, client

**Files:**
- Create: `apps/web/src/sim/protocol.ts`, `sim/describe-scene.ts`, `sim/sim-worker-handler.ts`, `sim/sim-worker-client.ts`, `sim/sim.worker.ts`, `bench/benchmark-world.ts`
- Modify: `apps/web/src/app/dependencies.ts`, `apps/web/src/main.tsx`
- Test: `apps/web/src/sim/describe-scene.test.ts`, `sim/sim-worker-handler.test.ts`, `sim/sim-worker-client.test.ts`

**Interfaces:**
- Consumes: challenges (`LabSession`, `loadBuiltInChallenge`, `LabError`, `Clock`, `createSolverRegistry`, `BUILT_IN_SOLVERS`, `Comparison`, `LabPhase`, `BuiltInChallengeId`); sim-core (`VoxelGrid`, `mergeStaticBlocks`, `cellBoxToWorldBox`, `Simulation`, `PhysicsEngineFactory`, `SimEvent`); formats (`Measurement`, `Task`, `WorldDoc`, `MaterialId`, `Vec3`, `EntityId`, `PartLayer`); `AppErrorCode` (Task 15).
- Produces:
```ts
// protocol.ts
export const TRANSFORM_STRIDE = 7   // position xyz + quaternion xyzw, per part, in SceneDescription.parts order
export type SceneShape = { kind: 'box'; halfExtentsM: Vec3 } | { kind: 'sphere'; radiusM: number }
export type SceneTerrainBox = { centerM: Vec3; halfExtentsM: Vec3; material: MaterialId }
export type ScenePart = { id: EntityId; shape: SceneShape; material: MaterialId; layer: PartLayer }
export type SceneDescription = { terrain: SceneTerrainBox[]; parts: ScenePart[]; focusM: Vec3 }
export type SceneEvent = 'released' | 'pausedAtRelease' | 'landed'
export type TaskView = { id: string; prompt: { es: string; en: string }; quantity: string; unit: string }
export type ChallengeView = { id: string; title: { es: string; en: string }; tasks: TaskView[]; calibrationQuantities: string[]; allowedCommands: string[] }
export type SimRequest =
  | { type: 'init'; challengeId: BuiltInChallengeId; seed: number; studentId: string; appVersion: string }
  | { type: 'advance'; steps: number }
  | { type: 'launch' } | { type: 'continueRun' } | { type: 'reset' }
  | { type: 'measure'; quantity: string }
  | { type: 'predict'; taskId: string; value: number }
  | { type: 'compare'; taskId: string }
  | { type: 'explain'; taskId: string; text: string }
  | { type: 'exportResult' }
  | { type: 'benchmark'; bodies: number; steps: number }
export type FrameUpdate = { phase: LabPhase; stepIndex: number; transforms: Float32Array; events: SceneEvent[] }
export type SimResponse =
  | { type: 'initialized'; scene: SceneDescription; challenge: ChallengeView; frame: FrameUpdate }
  | { type: 'frame'; frame: FrameUpdate }
  | { type: 'measured'; measurement: Measurement; frame: FrameUpdate }
  | { type: 'compared'; comparison: Comparison }
  | { type: 'done' }
  | { type: 'result'; json: string }
  | { type: 'benchmark'; msPerStep: number }
  | { type: 'error'; code: AppErrorCode; params: Record<string, string | number> }
export type RequestEnvelope = { id: number; request: SimRequest }
export type ResponseEnvelope = { id: number; response: SimResponse }
export function isRequestEnvelope(value: unknown): value is RequestEnvelope
export function isResponseEnvelope(value: unknown): value is ResponseEnvelope
// describe-scene.ts
export function describeScene(world: WorldDoc): SceneDescription
// sim-worker-handler.ts
export type SimWorkerDeps = { engineFactory: () => Promise<PhysicsEngineFactory>; clock: Clock }
export type HandlerReply = { response: SimResponse; transfer: Transferable[] }
export function createSimWorkerHandler(deps: SimWorkerDeps): (request: SimRequest) => Promise<HandlerReply>
// sim-worker-client.ts
export interface MessagePortLike { postMessage(message: unknown, transfer: Transferable[]): void; addEventListener(type: 'message', listener: (event: MessageEvent) => void): void; terminate?(): void }
export class SimRequestError extends Error { constructor(readonly code: AppErrorCode, readonly params: Record<string, string | number>) }
/** Port the UI depends on (D30); SimWorkerClient is the adapter, FakeSimClient (Task 18) the test fake. */
export interface SimClient {
  init(request: Omit<Extract<SimRequest, { type: 'init' }>, 'type'>): Promise<Extract<SimResponse, { type: 'initialized' }>>
  advance(steps: number): Promise<FrameUpdate>
  launch(): Promise<FrameUpdate>
  continueRun(): Promise<FrameUpdate>
  reset(): Promise<FrameUpdate>
  measure(quantity: string): Promise<{ measurement: Measurement; frame: FrameUpdate }>
  predict(taskId: string, value: number): Promise<void>
  compare(taskId: string): Promise<Comparison>
  explain(taskId: string, text: string): Promise<void>
  exportResult(): Promise<string>
  benchmark(bodies: number, steps: number): Promise<number>
  dispose(): void
}
export class SimWorkerClient implements SimClient { constructor(port: MessagePortLike) }
// bench/benchmark-world.ts
export const BENCHMARK_BODIES = 300
export function benchmarkWorld(bodies: number): WorldDoc
```

- [ ] **Step 1: Write the failing tests**

`apps/web/src/sim/describe-scene.test.ts`:
```ts
import { loadBuiltInChallenge } from '@physics-lab/challenges'
import { describe, expect, it } from 'vitest'
import { describeScene } from './describe-scene'

describe('describeScene', () => {
  const scene = describeScene(loadBuiltInChallenge('catapult-range').scenario.world)

  it('lists parts sorted by id with their shapes', () => {
    expect(scene.parts.map((part) => part.id)).toEqual(['arm', 'ball', 'post'])
    expect(scene.parts[1]).toEqual({ id: 'ball', shape: { kind: 'sphere', radiusM: 0.1 }, material: 'stone', layer: 'projectile' })
    expect(scene.parts[0]?.shape).toEqual({ kind: 'box', halfExtentsM: [0.8, 0.05, 0.05] })
  })

  it('uses the merged static terrain boxes', () => {
    expect(scene.terrain).toHaveLength(64)
  })

  it('focuses the camera on the machine', () => {
    expect(scene.focusM).toEqual([10, 2.6, 32])
  })
})
```

`apps/web/src/sim/sim-worker-handler.test.ts`:
```ts
import { loadBuiltInChallenge, verifyResult } from '@physics-lab/challenges'
import { ResultSchema } from '@physics-lab/formats'
import { createRapierEngineFactory } from '@physics-lab/sim-core/rapier'
import { describe, expect, it } from 'vitest'
import { TRANSFORM_STRIDE, type SimRequest, type SimResponse } from './protocol'
import { createSimWorkerHandler } from './sim-worker-handler'

function newHandler() {
  let now = 0
  const handle = createSimWorkerHandler({ engineFactory: createRapierEngineFactory, clock: { nowMs: () => (now += 16) } })
  return async (request: SimRequest): Promise<SimResponse> => (await handle(request)).response
}

async function advanceUntil(send: (r: SimRequest) => Promise<SimResponse>, phase: string): Promise<string[]> {
  const events: string[] = []
  for (let i = 0; i < 1000; i++) {
    const response = await send({ type: 'advance', steps: 8 })
    if (response.type !== 'frame') throw new Error(response.type)
    events.push(...response.frame.events)
    if (response.frame.phase === phase) return events
  }
  throw new Error(`never reached ${phase}`)
}

describe('sim worker handler', () => {
  it('refuses requests before init', async () => {
    const send = newHandler()
    expect(await send({ type: 'launch' })).toEqual({ type: 'error', code: 'notInitialized', params: {} })
  })

  it('initializes the catapult with a scene, challenge view and transforms for every part', async () => {
    const send = newHandler()
    const response = await send({ type: 'init', challengeId: 'catapult-range', seed: 12345, studentId: 'ana', appVersion: 'test' })
    if (response.type !== 'initialized') throw new Error(response.type)
    expect(response.scene.parts.map((part) => part.id)).toEqual(['arm', 'ball', 'post'])
    expect(response.frame.transforms).toHaveLength(3 * TRANSFORM_STRIDE)
    expect(response.challenge.allowedCommands).toEqual(['help', 'measure', 'reset', 'time'])
    expect(response.frame.phase).toBe('ready')
  })

  it('runs the whole flow and exports a result that verifies', async () => {
    const send = newHandler()
    await send({ type: 'init', challengeId: 'catapult-range', seed: 12345, studentId: 'ana', appVersion: 'test' })
    await send({ type: 'launch' })
    const calibrationEvents = await advanceUntil(send, 'calibrated')
    expect(calibrationEvents).toEqual(expect.arrayContaining(['released', 'pausedAtRelease']))
    for (const quantity of ['flight.releaseSpeed', 'flight.releaseAngle', 'flight.releaseHeight']) {
      expect((await send({ type: 'measure', quantity })).type).toBe('measured')
    }
    expect(await send({ type: 'predict', taskId: 'predict-range', value: 15 })).toEqual({ type: 'done' })
    await send({ type: 'continueRun' })
    expect(await advanceUntil(send, 'landed')).toContain('landed')
    await send({ type: 'measure', quantity: 'flight.range' })
    expect((await send({ type: 'compare', taskId: 'predict-range' })).type).toBe('compared')
    await send({ type: 'explain', taskId: 'predict-range', text: 'No drag.' })
    const exported = await send({ type: 'exportResult' })
    if (exported.type !== 'result') throw new Error(exported.type)
    const result = ResultSchema.parse(JSON.parse(exported.json))
    expect(verifyResult(loadBuiltInChallenge('catapult-range'), result, await createRapierEngineFactory()).status).toBe('verified')
  })

  it('maps lab errors to translated error codes', async () => {
    const send = newHandler()
    await send({ type: 'init', challengeId: 'catapult-range', seed: 1, studentId: 'ana', appVersion: 'test' })
    expect(await send({ type: 'measure', quantity: 'flight.range' })).toEqual({ type: 'error', code: 'predictionRequired', params: { quantity: 'flight.range' } })
  })

  it('reports benchmark step time', async () => {
    const send = newHandler()
    const response = await send({ type: 'benchmark', bodies: 20, steps: 10 })
    expect(response.type === 'benchmark' && response.msPerStep).toBeGreaterThan(0)
  })
})
```

`apps/web/src/sim/sim-worker-client.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import type { RequestEnvelope, ResponseEnvelope } from './protocol'
import { SimRequestError, SimWorkerClient, type MessagePortLike } from './sim-worker-client'

class ManualPort implements MessagePortLike {
  readonly sent: RequestEnvelope[] = []
  private listener: ((event: MessageEvent) => void) | null = null
  postMessage(message: unknown): void {
    this.sent.push(message as RequestEnvelope)
  }
  addEventListener(_type: 'message', listener: (event: MessageEvent) => void): void {
    this.listener = listener
  }
  reply(envelope: ResponseEnvelope): void {
    this.listener?.(new MessageEvent('message', { data: envelope }))
  }
}

describe('SimWorkerClient', () => {
  it('correlates out-of-order responses by id', async () => {
    const port = new ManualPort()
    const client = new SimWorkerClient(port)
    const first = client.exportResult()
    const second = client.benchmark(10, 10)
    port.reply({ id: port.sent[1]?.id ?? -1, response: { type: 'benchmark', msPerStep: 0.5 } })
    port.reply({ id: port.sent[0]?.id ?? -1, response: { type: 'result', json: '{}' } })
    expect(await second).toBe(0.5)
    expect(await first).toBe('{}')
  })

  it('rejects with a typed error for error responses', async () => {
    const port = new ManualPort()
    const client = new SimWorkerClient(port)
    const pending = client.launch()
    port.reply({ id: port.sent[0]?.id ?? -1, response: { type: 'error', code: 'invalidPhase', params: {} } })
    await expect(pending).rejects.toBeInstanceOf(SimRequestError)
  })

  it('ignores malformed messages', () => {
    const port = new ManualPort()
    new SimWorkerClient(port)
    expect(() => port.reply({ nope: true } as unknown as ResponseEnvelope)).not.toThrow()
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run --project web sim`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement**

`apps/web/src/sim/protocol.ts`: the types listed in **Interfaces** above, plus:
```ts
function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === 'object' && value !== null
}

export function isRequestEnvelope(value: unknown): value is RequestEnvelope {
  return isRecord(value) && typeof value['id'] === 'number' && isRecord(value['request']) && typeof value['request']['type'] === 'string'
}

export function isResponseEnvelope(value: unknown): value is ResponseEnvelope {
  return isRecord(value) && typeof value['id'] === 'number' && isRecord(value['response']) && typeof value['response']['type'] === 'string'
}
```
(Import `LabPhase`, `Comparison`, `BuiltInChallengeId` from `@physics-lab/challenges`, `Measurement`, `MaterialId`, `Vec3`, `EntityId`, `PartLayer` from `@physics-lab/formats`, `AppErrorCode` from `@/i18n/required-keys`.)

`apps/web/src/sim/describe-scene.ts`:
```ts
import type { Part, WorldDoc } from '@physics-lab/formats'
import { VoxelGrid, cellBoxToWorldBox, mergeStaticBlocks } from '@physics-lab/sim-core'
import type { SceneDescription, ScenePart } from './protocol'

function toScenePart(part: Part): ScenePart {
  const shape = part.type === 'beam'
    ? { kind: 'box' as const, halfExtentsM: [part.sizeM[0] / 2, part.sizeM[1] / 2, part.sizeM[2] / 2] as const }
    : { kind: 'sphere' as const, radiusM: part.radiusM }
  return { id: part.id, shape, material: part.material, layer: part.layer }
}

/** Static description of what to draw; dynamic transforms arrive per frame in the same part order. */
export function describeScene(world: WorldDoc): SceneDescription {
  const terrain = mergeStaticBlocks(VoxelGrid.fromFills(world.blocks)).map((box) => ({ ...cellBoxToWorldBox(box), material: box.material }))
  const parts = [...world.parts].sort((a, b) => (a.id < b.id ? -1 : a.id > b.id ? 1 : 0)).map(toScenePart)
  const pivot = world.joints.find((joint) => joint.type === 'hinge')
  const focusM = pivot?.type === 'hinge' ? pivot.anchorM : (world.parts[0]?.positionM ?? [0, 0, 0])
  return { terrain, parts, focusM }
}
```

`apps/web/src/bench/benchmark-world.ts`:
```ts
import type { Part, WorldDoc } from '@physics-lab/formats'

/** Dynamic-body budget on the medium tier (spec §10.7). */
export const BENCHMARK_BODIES = 300
const SPACING_M = 0.5
const COLUMNS = 20
const ORIGIN_M = [27, 3, 27] as const

/** Spheres dropped in a grid onto the ground: a repeatable load for the Chromebook check (spec §13). */
export function benchmarkWorld(bodies: number): WorldDoc {
  const parts: Part[] = Array.from({ length: bodies }, (_, index) => ({
    id: `body-${String(index).padStart(4, '0')}`,
    type: 'sphere',
    motion: 'dynamic',
    material: 'rubber',
    layer: 'projectile',
    positionM: [ORIGIN_M[0] + (index % COLUMNS) * SPACING_M, ORIGIN_M[1] + Math.floor(index / (COLUMNS * COLUMNS)) * SPACING_M, ORIGIN_M[2] + (Math.floor(index / COLUMNS) % COLUMNS) * SPACING_M],
    rotationDeg: [0, 0, 0],
    ccd: false,
    radiusM: 0.2,
  }))
  return { format: 'physics-lab/world', formatVersion: 1, gravityMps2: 9.8, blocks: [{ from: [0, 0, 0], to: [127, 1, 127], material: 'stone' }], parts, joints: [], latches: [] }
}
```

`apps/web/src/sim/sim-worker-handler.ts`:
```ts
import { BUILT_IN_SOLVERS, LabError, LabSession, createSolverRegistry, loadBuiltInChallenge, type Clock, type LabPhase } from '@physics-lab/challenges'
import { Simulation, type PhysicsEngineFactory, type SimEvent } from '@physics-lab/sim-core'
import { benchmarkWorld } from '@/bench/benchmark-world'
import { describeScene } from './describe-scene'
import { TRANSFORM_STRIDE, type ChallengeView, type FrameUpdate, type SceneEvent, type SimRequest, type SimResponse } from './protocol'

export type SimWorkerDeps = { engineFactory: () => Promise<PhysicsEngineFactory>; clock: Clock }
export type HandlerReply = { response: SimResponse; transfer: Transferable[] }

type ActiveSession = { session: LabSession; partIds: readonly string[]; pendingEvents: SceneEvent[]; lastPhase: LabPhase }

const PHASE_EVENTS: Partial<Record<LabPhase, SceneEvent>> = { calibrated: 'pausedAtRelease', landed: 'landed' }

function reply(response: SimResponse, transfer: Transferable[] = []): HandlerReply {
  return { response, transfer }
}

function errorReply(error: unknown): HandlerReply {
  if (error instanceof LabError) return reply({ type: 'error', code: error.code, params: { ...error.params } })
  console.error(error)
  return reply({ type: 'error', code: 'internal', params: {} })
}

function challengeView(session: LabSession): ChallengeView {
  const challenge = session.challenge
  return {
    id: challenge.id,
    title: challenge.title,
    tasks: challenge.tasks.map((task) => ({ id: task.id, prompt: task.prompt, quantity: task.quantity, unit: task.unit })),
    calibrationQuantities: challenge.calibration?.measurable ?? [],
    allowedCommands: challenge.allowedCommands,
  }
}

/** Request handler run inside the worker; pure of worker globals so it is testable in Node. */
export function createSimWorkerHandler(deps: SimWorkerDeps): (request: SimRequest) => Promise<HandlerReply> {
  let active: ActiveSession | null = null
  let factory: PhysicsEngineFactory | null = null
  const solvers = createSolverRegistry(BUILT_IN_SOLVERS)

  const engineFactory = async (): Promise<PhysicsEngineFactory> => (factory ??= await deps.engineFactory())

  function frame(state: ActiveSession): FrameUpdate {
    const transforms = new Float32Array(state.partIds.length * TRANSFORM_STRIDE)
    state.partIds.forEach((id, index) => {
      const body = state.session.readBody(id)
      transforms.set([...body.positionM, ...body.rotation], index * TRANSFORM_STRIDE)
    })
    const phase = state.session.phase
    const phaseEvent = phase !== state.lastPhase ? PHASE_EVENTS[phase] : undefined
    if (phaseEvent) state.pendingEvents.push(phaseEvent)
    state.lastPhase = phase
    const events = state.pendingEvents.splice(0)
    return { phase, stepIndex: state.session.stepIndex, transforms, events }
  }

  function frameReply(state: ActiveSession, build: (update: FrameUpdate) => SimResponse): HandlerReply {
    const update = frame(state)
    return reply(build(update), [update.transforms.buffer])
  }

  async function init(request: Extract<SimRequest, { type: 'init' }>): Promise<HandlerReply> {
    active?.session.dispose()
    const session = LabSession.create({
      challenge: loadBuiltInChallenge(request.challengeId), seed: request.seed, studentId: request.studentId,
      engineFactory: await engineFactory(), clock: deps.clock, appVersion: request.appVersion, solvers,
    })
    const state: ActiveSession = { session, partIds: session.partIds(), pendingEvents: [], lastPhase: session.phase }
    session.subscribe((event: SimEvent) => {
      if (event.type === 'latchReleased') state.pendingEvents.push('released')
    })
    active = state
    return frameReply(state, (update) => ({ type: 'initialized', scene: describeScene(session.resolvedWorld), challenge: challengeView(session), frame: update }))
  }

  async function benchmark(request: Extract<SimRequest, { type: 'benchmark' }>): Promise<HandlerReply> {
    const simulation = Simulation.create(benchmarkWorld(request.bodies), await engineFactory())
    const started = deps.clock.nowMs()
    for (let i = 0; i < request.steps; i++) simulation.step()
    const elapsed = deps.clock.nowMs() - started
    simulation.dispose()
    return reply({ type: 'benchmark', msPerStep: elapsed / request.steps })
  }

  function handleSessionRequest(state: ActiveSession, request: SimRequest): HandlerReply {
    const { session } = state
    switch (request.type) {
      case 'advance':
        session.advance(request.steps)
        return frameReply(state, (update) => ({ type: 'frame', frame: update }))
      case 'launch':
        session.launch()
        return frameReply(state, (update) => ({ type: 'frame', frame: update }))
      case 'continueRun':
        session.continueRun()
        return frameReply(state, (update) => ({ type: 'frame', frame: update }))
      case 'reset':
        session.reset()
        return frameReply(state, (update) => ({ type: 'frame', frame: update }))
      case 'measure': {
        const measurement = session.measure(request.quantity)
        return frameReply(state, (update) => ({ type: 'measured', measurement, frame: update }))
      }
      case 'predict':
        session.submitPrediction(request.taskId, request.value)
        return reply({ type: 'done' })
      case 'compare':
        return reply({ type: 'compared', comparison: session.compare(request.taskId) })
      case 'explain':
        session.submitExplanation(request.taskId, request.text)
        return reply({ type: 'done' })
      case 'exportResult':
        return reply({ type: 'result', json: JSON.stringify(session.toResult(), null, 2) })
      case 'init':
      case 'benchmark':
        return reply({ type: 'error', code: 'internal', params: {} })
    }
  }

  return async (request) => {
    try {
      if (request.type === 'init') return await init(request)
      if (request.type === 'benchmark') return await benchmark(request)
      if (!active) return reply({ type: 'error', code: 'notInitialized', params: {} })
      return handleSessionRequest(active, request)
    } catch (error) {
      return errorReply(error)
    }
  }
}
```
(`QuantityId` and `EntityId` are plain `string` types; a malformed quantity or task id reaches `LabSession`, which throws `unknownQuantity` / `unknownTask`, mapped to error responses by `errorReply`.)

`apps/web/src/sim/sim-worker-client.ts`:
```ts
import type { Comparison } from '@physics-lab/challenges'
import type { Measurement } from '@physics-lab/formats'
import type { AppErrorCode } from '@/i18n/required-keys'
import { isResponseEnvelope, type FrameUpdate, type SimRequest, type SimResponse } from './protocol'

export interface MessagePortLike {
  postMessage(message: unknown, transfer: Transferable[]): void
  addEventListener(type: 'message', listener: (event: MessageEvent) => void): void
  terminate?(): void
}

export class SimRequestError extends Error {
  override name = 'SimRequestError'
  constructor(
    readonly code: AppErrorCode,
    readonly params: Record<string, string | number>
  ) {
    super(code)
  }
}

type InitRequest = Omit<Extract<SimRequest, { type: 'init' }>, 'type'>
type Initialized = Extract<SimResponse, { type: 'initialized' }>

/** Port the UI depends on (D30). */
export interface SimClient {
  init(request: InitRequest): Promise<Initialized>
  advance(steps: number): Promise<FrameUpdate>
  launch(): Promise<FrameUpdate>
  continueRun(): Promise<FrameUpdate>
  reset(): Promise<FrameUpdate>
  measure(quantity: string): Promise<{ measurement: Measurement; frame: FrameUpdate }>
  predict(taskId: string, value: number): Promise<void>
  compare(taskId: string): Promise<Comparison>
  explain(taskId: string, text: string): Promise<void>
  exportResult(): Promise<string>
  benchmark(bodies: number, steps: number): Promise<number>
  dispose(): void
}

function expectType<T extends SimResponse['type']>(response: SimResponse, type: T): Extract<SimResponse, { type: T }> {
  if (response.type === 'error') throw new SimRequestError(response.code, response.params)
  if (response.type !== type) throw new SimRequestError('internal', { expected: type, actual: response.type })
  return response as Extract<SimResponse, { type: T }>
}

/** Adapter: SimClient over a Worker (or any message port) with id-correlated requests. */
export class SimWorkerClient implements SimClient {
  private nextId = 1
  private readonly pending = new Map<number, (response: SimResponse) => void>()

  constructor(private readonly port: MessagePortLike) {
    port.addEventListener('message', (event) => this.receive(event.data))
  }

  init(request: InitRequest): Promise<Initialized> {
    return this.send({ type: 'init', ...request }).then((r) => expectType(r, 'initialized'))
  }
  advance(steps: number): Promise<FrameUpdate> {
    return this.send({ type: 'advance', steps }).then((r) => expectType(r, 'frame').frame)
  }
  launch(): Promise<FrameUpdate> {
    return this.send({ type: 'launch' }).then((r) => expectType(r, 'frame').frame)
  }
  continueRun(): Promise<FrameUpdate> {
    return this.send({ type: 'continueRun' }).then((r) => expectType(r, 'frame').frame)
  }
  reset(): Promise<FrameUpdate> {
    return this.send({ type: 'reset' }).then((r) => expectType(r, 'frame').frame)
  }
  measure(quantity: string): Promise<{ measurement: Measurement; frame: FrameUpdate }> {
    return this.send({ type: 'measure', quantity }).then((r) => {
      const measured = expectType(r, 'measured')
      return { measurement: measured.measurement, frame: measured.frame }
    })
  }
  predict(taskId: string, value: number): Promise<void> {
    return this.send({ type: 'predict', taskId, value }).then((r) => void expectType(r, 'done'))
  }
  compare(taskId: string): Promise<Comparison> {
    return this.send({ type: 'compare', taskId }).then((r) => expectType(r, 'compared').comparison)
  }
  explain(taskId: string, text: string): Promise<void> {
    return this.send({ type: 'explain', taskId, text }).then((r) => void expectType(r, 'done'))
  }
  exportResult(): Promise<string> {
    return this.send({ type: 'exportResult' }).then((r) => expectType(r, 'result').json)
  }
  benchmark(bodies: number, steps: number): Promise<number> {
    return this.send({ type: 'benchmark', bodies, steps }).then((r) => expectType(r, 'benchmark').msPerStep)
  }
  dispose(): void {
    this.port.terminate?.()
    this.pending.clear()
  }

  private send(request: SimRequest): Promise<SimResponse> {
    const id = this.nextId++
    return new Promise((resolve) => {
      this.pending.set(id, resolve)
      this.port.postMessage({ id, request }, [])
    })
  }

  private receive(data: unknown): void {
    if (!isResponseEnvelope(data)) return
    const resolve = this.pending.get(data.id)
    if (!resolve) return
    this.pending.delete(data.id)
    resolve(data.response)
  }
}
```

`apps/web/src/sim/sim.worker.ts` (worker composition root):
```ts
import { createRapierEngineFactory } from '@physics-lab/sim-core/rapier'
import { z } from 'zod'
import { isRequestEnvelope } from './protocol'
import { createSimWorkerHandler } from './sim-worker-handler'

type WorkerScope = {
  postMessage(message: unknown, transfer: Transferable[]): void
  addEventListener(type: 'message', listener: (event: MessageEvent) => void): void
}

// Composition root of the worker (D30). The DOM lib types `self` as Window; the cast narrows it to what we use.
z.config({ jitless: true })
const scope = self as unknown as WorkerScope
const handle = createSimWorkerHandler({ engineFactory: createRapierEngineFactory, clock: { nowMs: () => performance.now() } })

scope.addEventListener('message', (event) => {
  const data: unknown = event.data
  if (!isRequestEnvelope(data)) return
  void handle(data.request).then(({ response, transfer }) => scope.postMessage({ id: data.id, response }, transfer))
})
```

`apps/web/src/app/dependencies.ts` — add:
```ts
import type { SimClient } from '@/sim/sim-worker-client'

export type AppDependencies = {
  appVersion: string
  randomSeed(): number
  createSimClient(): SimClient
}
```

`apps/web/src/main.tsx` — add to `dependencies`:
```ts
  createSimClient: () => new SimWorkerClient(new Worker(new URL('./sim/sim.worker.ts', import.meta.url), { type: 'module' })),
```
(with `import { SimWorkerClient } from '@/sim/sim-worker-client'`).

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm vitest run --project web sim`
Expected: PASS.

- [ ] **Step 5: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test && pnpm --filter @physics-lab/web build
git add apps/web
git commit -m "feat(web): run the lab session in a web worker behind a sim client port"
```

---
### Task 17: `apps/web` — orbit camera, interpolation, frame loop and the three.js renderer

**Files:**
- Create: `apps/web/src/render/orbit-camera.ts`, `render/interpolation.ts`, `render/frame-loop.ts`, `render/scene-renderer.ts`
- Test: `apps/web/src/render/orbit-camera.test.ts`, `render/interpolation.test.ts`, `render/frame-loop.test.ts`

**Interfaces:**
- Consumes: `SceneDescription`, `TRANSFORM_STRIDE` (Task 16); `Theme` (Task 15); `FIXED_STEP_S` (sim-core); `Vec3` (formats).
- Produces:
```ts
// orbit-camera.ts
export type OrbitCameraState = { targetM: Vec3; azimuthDeg: number; elevationDeg: number; distanceM: number }
export type OrbitCommand = 'rotateLeft' | 'rotateRight' | 'rotateUp' | 'rotateDown' | 'zoomIn' | 'zoomOut' | 'reset'
export const ORBIT_COMMANDS: readonly OrbitCommand[]
export const ORBIT_LIMITS, ORBIT_STEP
export function initialCamera(focusM: Vec3): OrbitCameraState
export function applyOrbitCommand(state: OrbitCameraState, command: OrbitCommand, initial: OrbitCameraState): OrbitCameraState
export function orbitCommandForKey(key: string): OrbitCommand | null
export function dragOrbit(state: OrbitCameraState, dxPx: number, dyPx: number): OrbitCameraState
export function cameraPositionM(state: OrbitCameraState): Vec3
// interpolation.ts
export function interpolateTransforms(previous: Float32Array, current: Float32Array, alpha: number, out: Float32Array): void
// frame-loop.ts
export const MAX_FRAME_DT_S = 0.1
export const MAX_STEPS_PER_FRAME = 48
export type FramePlan = { steps: number; accumulatorS: number; alpha: number }
export function planFrame(accumulatorS: number, frameDtS: number, timeScale: number): FramePlan
export type FrameScheduler = { requestFrame(callback: (timeMs: number) => void): number; cancelFrame(handle: number): void }
export class FrameLoop { constructor(scheduler: FrameScheduler, onFrame: (plan: FramePlan) => Promise<void>); start(): void; stop(): void; setTimeScale(factor: number): void }
// scene-renderer.ts
export interface Renderer { applyTransforms(transforms: Float32Array): void; setCamera(state: OrbitCameraState): void; resize(widthPx: number, heightPx: number, pixelRatio: number): void; render(): void; dispose(): void }
export type RendererFactory = (canvas: HTMLCanvasElement, scene: SceneDescription) => Promise<Renderer>
export class SceneRenderer implements Renderer { static create(canvas: HTMLCanvasElement, scene: SceneDescription, theme: Theme): Promise<SceneRenderer> }
```

Time model (spec §5.4): the browser only decides **how many** fixed steps to run per frame; `dt` never changes. Frame gaps are clamped to 0.1 s and steps per frame to 48 so a hidden tab never triggers a catch-up storm (Review Focus 5).

- [ ] **Step 1: Write the failing tests**

`apps/web/src/render/orbit-camera.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { ORBIT_LIMITS, applyOrbitCommand, cameraPositionM, dragOrbit, initialCamera, orbitCommandForKey, type OrbitCameraState } from './orbit-camera'

const base: OrbitCameraState = { targetM: [0, 0, 0], azimuthDeg: 0, elevationDeg: 0, distanceM: 10 }

describe('orbit camera', () => {
  it('places the camera on +z at azimuth 0 and elevation 0', () => {
    const [x, y, z] = cameraPositionM(base)
    expect(x).toBeCloseTo(0, 12)
    expect(y).toBeCloseTo(0, 12)
    expect(z).toBeCloseTo(10, 12)
  })

  it('wraps azimuth and clamps elevation and distance', () => {
    expect(applyOrbitCommand({ ...base, azimuthDeg: 355 }, 'rotateRight', base).azimuthDeg).toBe(0)
    expect(applyOrbitCommand({ ...base, azimuthDeg: 0 }, 'rotateLeft', base).azimuthDeg).toBe(355)
    expect(applyOrbitCommand({ ...base, elevationDeg: ORBIT_LIMITS.maxElevationDeg }, 'rotateUp', base).elevationDeg).toBe(ORBIT_LIMITS.maxElevationDeg)
    expect(applyOrbitCommand({ ...base, distanceM: ORBIT_LIMITS.minDistanceM }, 'zoomIn', base).distanceM).toBe(ORBIT_LIMITS.minDistanceM)
  })

  it('resets to the initial view', () => {
    const moved = applyOrbitCommand(base, 'rotateRight', base)
    expect(applyOrbitCommand(moved, 'reset', base)).toEqual(base)
  })

  it('maps keys to commands (keyboard alternative, WCAG 2.1.1 / 2.5.7)', () => {
    expect(orbitCommandForKey('ArrowLeft')).toBe('rotateLeft')
    expect(orbitCommandForKey('+')).toBe('zoomIn')
    expect(orbitCommandForKey('-')).toBe('zoomOut')
    expect(orbitCommandForKey('Home')).toBe('reset')
    expect(orbitCommandForKey('x')).toBeNull()
  })

  it('rotates with pointer drags', () => {
    const dragged = dragOrbit(base, 10, -10)
    expect(dragged.azimuthDeg).not.toBe(base.azimuthDeg)
    expect(dragged.elevationDeg).toBeGreaterThan(base.elevationDeg)
  })

  it('frames the launch area from the side', () => {
    const camera = initialCamera([10, 2.6, 32])
    expect(camera.targetM[2]).toBe(32)
    expect(camera.elevationDeg).toBeGreaterThan(0)
  })
})
```

`apps/web/src/render/interpolation.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { interpolateTransforms } from './interpolation'

const SIN_45 = Math.sin(Math.PI / 4)
const previous = new Float32Array([0, 0, 0, 0, 0, 0, 1])
const current = new Float32Array([2, 4, 6, 0, 0, SIN_45, SIN_45])

describe('interpolateTransforms', () => {
  it('returns the previous state at alpha 0 and the current at alpha 1', () => {
    const out = new Float32Array(7)
    interpolateTransforms(previous, current, 0, out)
    expect([...out]).toEqual([...previous])
    interpolateTransforms(previous, current, 1, out)
    out.forEach((value, i) => expect(value).toBeCloseTo(current[i] ?? Number.NaN, 6))
  })

  it('lerps positions and slerps rotations', () => {
    const out = new Float32Array(7)
    interpolateTransforms(previous, current, 0.5, out)
    expect([...out.slice(0, 3)]).toEqual([1, 2, 3])
    expect(out[5]).toBeCloseTo(Math.sin(Math.PI / 8), 6)
    expect(out[6]).toBeCloseTo(Math.cos(Math.PI / 8), 6)
  })
})
```

`apps/web/src/render/frame-loop.test.ts`:
```ts
import { FIXED_STEP_S } from '@physics-lab/sim-core'
import { describe, expect, it, vi } from 'vitest'
import { FrameLoop, MAX_STEPS_PER_FRAME, planFrame, type FrameScheduler } from './frame-loop'

describe('planFrame', () => {
  it('runs 4 fixed steps per 60 Hz frame in real time', () => {
    expect(planFrame(0, 1 / 60, 1).steps).toBe(4)
  })

  it('slows down by running fewer steps, never by changing dt', () => {
    let accumulator = 0
    let steps = 0
    for (let frame = 0; frame < 60; frame++) {
      const plan = planFrame(accumulator, 1 / 60, 0.25)
      accumulator = plan.accumulatorS
      steps += plan.steps
    }
    expect(steps).toBe(60)
  })

  it('clamps long gaps such as a hidden tab (Review Focus 5)', () => {
    const plan = planFrame(0, 120, 1)
    expect(plan.steps).toBe(Math.round(0.1 / FIXED_STEP_S))
    const fast = planFrame(0, 120, 4)
    expect(fast.steps).toBe(MAX_STEPS_PER_FRAME)
    expect(fast.accumulatorS).toBeLessThan(FIXED_STEP_S)
  })

  it('ignores negative frame times and reports interpolation alpha', () => {
    expect(planFrame(0, -1, 1).steps).toBe(0)
    expect(planFrame(FIXED_STEP_S / 2, 0, 1).alpha).toBeCloseTo(0.5, 9)
  })
})

class FakeScheduler implements FrameScheduler {
  private callbacks = new Map<number, (timeMs: number) => void>()
  private nextHandle = 1
  requestFrame(callback: (timeMs: number) => void): number {
    const handle = this.nextHandle++
    this.callbacks.set(handle, callback)
    return handle
  }
  cancelFrame(handle: number): void {
    this.callbacks.delete(handle)
  }
  fire(timeMs: number): void {
    const pending = [...this.callbacks.values()]
    this.callbacks.clear()
    for (const callback of pending) callback(timeMs)
  }
  get scheduled(): number {
    return this.callbacks.size
  }
}

describe('FrameLoop', () => {
  it('plans a frame per animation frame and skips frames while busy', async () => {
    const scheduler = new FakeScheduler()
    let release: () => void = () => undefined
    const onFrame = vi.fn(() => new Promise<void>((resolve) => (release = resolve)))
    const loop = new FrameLoop(scheduler, onFrame)
    loop.start()
    scheduler.fire(0)
    scheduler.fire(16)
    expect(onFrame).toHaveBeenCalledTimes(1)
    release()
    await Promise.resolve()
    await Promise.resolve()
    scheduler.fire(33)
    expect(onFrame).toHaveBeenCalledTimes(2)
    expect(onFrame.mock.calls[1]?.[0]?.steps).toBe(Math.floor(0.033 / FIXED_STEP_S + 1e-9))
  })

  it('stops scheduling after stop()', () => {
    const scheduler = new FakeScheduler()
    const loop = new FrameLoop(scheduler, () => Promise.resolve())
    loop.start()
    loop.stop()
    expect(scheduler.scheduled).toBe(0)
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run --project web render`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement**

`apps/web/src/render/orbit-camera.ts`:
```ts
import type { Vec3 } from '@physics-lab/formats'

export type OrbitCameraState = { targetM: Vec3; azimuthDeg: number; elevationDeg: number; distanceM: number }
export type OrbitCommand = 'rotateLeft' | 'rotateRight' | 'rotateUp' | 'rotateDown' | 'zoomIn' | 'zoomOut' | 'reset'
export const ORBIT_COMMANDS: readonly OrbitCommand[] = ['rotateLeft', 'rotateRight', 'rotateUp', 'rotateDown', 'zoomIn', 'zoomOut', 'reset']
export const ORBIT_LIMITS = { minElevationDeg: 5, maxElevationDeg: 85, minDistanceM: 3, maxDistanceM: 80 } as const
export const ORBIT_STEP = { rotateDeg: 5, zoomFactor: 1.15 } as const

const FULL_TURN_DEG = 360
const DRAG_DEG_PER_PX = 0.3
const DEG_TO_RAD = Math.PI / 180
/** The ball flies +x from the pivot; frame the middle of a typical flight, seen from the +z side. */
const INITIAL_VIEW = { targetOffsetXM: 7, targetHeightM: 2, azimuthDeg: 0, elevationDeg: 18, distanceM: 24 } as const
const KEY_COMMANDS: Readonly<Record<string, OrbitCommand>> = {
  ArrowLeft: 'rotateLeft', ArrowRight: 'rotateRight', ArrowUp: 'rotateUp', ArrowDown: 'rotateDown',
  '+': 'zoomIn', '=': 'zoomIn', '-': 'zoomOut', Home: 'reset',
}

const clamp = (value: number, min: number, max: number): number => Math.min(Math.max(value, min), max)
const wrapDeg = (value: number): number => ((value % FULL_TURN_DEG) + FULL_TURN_DEG) % FULL_TURN_DEG

export function initialCamera(focusM: Vec3): OrbitCameraState {
  return {
    targetM: [focusM[0] + INITIAL_VIEW.targetOffsetXM, INITIAL_VIEW.targetHeightM, focusM[2]],
    azimuthDeg: INITIAL_VIEW.azimuthDeg,
    elevationDeg: INITIAL_VIEW.elevationDeg,
    distanceM: INITIAL_VIEW.distanceM,
  }
}

function rotate(state: OrbitCameraState, azimuthDelta: number, elevationDelta: number): OrbitCameraState {
  return {
    ...state,
    azimuthDeg: wrapDeg(state.azimuthDeg + azimuthDelta),
    elevationDeg: clamp(state.elevationDeg + elevationDelta, ORBIT_LIMITS.minElevationDeg, ORBIT_LIMITS.maxElevationDeg),
  }
}

function zoom(state: OrbitCameraState, factor: number): OrbitCameraState {
  return { ...state, distanceM: clamp(state.distanceM * factor, ORBIT_LIMITS.minDistanceM, ORBIT_LIMITS.maxDistanceM) }
}

export function applyOrbitCommand(state: OrbitCameraState, command: OrbitCommand, initial: OrbitCameraState): OrbitCameraState {
  switch (command) {
    case 'rotateLeft':
      return rotate(state, -ORBIT_STEP.rotateDeg, 0)
    case 'rotateRight':
      return rotate(state, ORBIT_STEP.rotateDeg, 0)
    case 'rotateUp':
      return rotate(state, 0, ORBIT_STEP.rotateDeg)
    case 'rotateDown':
      return rotate(state, 0, -ORBIT_STEP.rotateDeg)
    case 'zoomIn':
      return zoom(state, 1 / ORBIT_STEP.zoomFactor)
    case 'zoomOut':
      return zoom(state, ORBIT_STEP.zoomFactor)
    case 'reset':
      return initial
  }
}

export function orbitCommandForKey(key: string): OrbitCommand | null {
  return KEY_COMMANDS[key] ?? null
}

export function dragOrbit(state: OrbitCameraState, dxPx: number, dyPx: number): OrbitCameraState {
  return rotate(state, -dxPx * DRAG_DEG_PER_PX, -dyPx * DRAG_DEG_PER_PX)
}

export function cameraPositionM(state: OrbitCameraState): Vec3 {
  const azimuth = state.azimuthDeg * DEG_TO_RAD
  const elevation = state.elevationDeg * DEG_TO_RAD
  const horizontal = state.distanceM * Math.cos(elevation)
  return [
    state.targetM[0] + horizontal * Math.sin(azimuth),
    state.targetM[1] + state.distanceM * Math.sin(elevation),
    state.targetM[2] + horizontal * Math.cos(azimuth),
  ]
}
```

`apps/web/src/render/interpolation.ts`:
```ts
import { Quaternion } from 'three'
import { TRANSFORM_STRIDE } from '@/sim/protocol'

const POSITION_COMPONENTS = 3
const ROTATION_OFFSET = 3

/** Display-rate interpolation between two simulation snapshots (spec §5.4); never feeds back into physics. */
export function interpolateTransforms(previous: Float32Array, current: Float32Array, alpha: number, out: Float32Array): void {
  for (let offset = 0; offset < current.length; offset += TRANSFORM_STRIDE) {
    for (let axis = 0; axis < POSITION_COMPONENTS; axis++) {
      const from = previous[offset + axis] ?? 0
      const to = current[offset + axis] ?? 0
      out[offset + axis] = from + (to - from) * alpha
    }
    Quaternion.slerpFlat(out, offset + ROTATION_OFFSET, previous, offset + ROTATION_OFFSET, current, offset + ROTATION_OFFSET, alpha)
  }
}
```

`apps/web/src/render/frame-loop.ts`:
```ts
import { FIXED_STEP_S } from '@physics-lab/sim-core'

/** Longest frame gap honored; longer gaps (hidden tab) are dropped, not replayed. */
export const MAX_FRAME_DT_S = 0.1
export const MAX_STEPS_PER_FRAME = 48
const STEP_EPSILON = 1e-9
const MS_PER_S = 1000

export type FramePlan = { steps: number; accumulatorS: number; alpha: number }
export type FrameScheduler = { requestFrame(callback: (timeMs: number) => void): number; cancelFrame(handle: number): void }

/** How many fixed steps this frame runs (slow motion = fewer steps per frame, never a different dt). */
export function planFrame(accumulatorS: number, frameDtS: number, timeScale: number): FramePlan {
  const scaledDt = Math.min(Math.max(frameDtS, 0), MAX_FRAME_DT_S) * timeScale
  let accumulator = accumulatorS + scaledDt
  let steps = Math.floor(accumulator / FIXED_STEP_S + STEP_EPSILON)
  if (steps > MAX_STEPS_PER_FRAME) {
    steps = MAX_STEPS_PER_FRAME
    accumulator = steps * FIXED_STEP_S
  }
  accumulator = Math.max(accumulator - steps * FIXED_STEP_S, 0)
  return { steps, accumulatorS: accumulator, alpha: accumulator / FIXED_STEP_S }
}

export class FrameLoop {
  private handle: number | null = null
  private lastTimeMs: number | null = null
  private accumulatorS = 0
  private busy = false
  private timeScale = 1

  constructor(
    private readonly scheduler: FrameScheduler,
    private readonly onFrame: (plan: FramePlan) => Promise<void>
  ) {}

  start(): void {
    if (this.handle === null) this.handle = this.scheduler.requestFrame(this.tick)
  }

  stop(): void {
    if (this.handle !== null) this.scheduler.cancelFrame(this.handle)
    this.handle = null
    this.lastTimeMs = null
  }

  setTimeScale(factor: number): void {
    this.timeScale = factor
  }

  private readonly tick = (timeMs: number): void => {
    this.handle = this.scheduler.requestFrame(this.tick)
    if (this.busy) return
    const frameDtS = this.lastTimeMs === null ? 0 : (timeMs - this.lastTimeMs) / MS_PER_S
    this.lastTimeMs = timeMs
    const plan = planFrame(this.accumulatorS, frameDtS, this.timeScale)
    this.accumulatorS = plan.accumulatorS
    this.busy = true
    void this.onFrame(plan).finally(() => {
      this.busy = false
    })
  }
}
```

`apps/web/src/render/scene-renderer.ts`:
```ts
import * as THREE from 'three/webgpu'
import { TRANSFORM_STRIDE, type SceneDescription, type ScenePart } from '@/sim/protocol'
import type { Theme } from '@/theme/tokens'
import { cameraPositionM, type OrbitCameraState } from './orbit-camera'

const FIELD_OF_VIEW_DEG = 50
const NEAR_PLANE_M = 0.1
const FAR_PLANE_M = 500
const SPHERE_SEGMENTS = 24
const SUN_POSITION_M = [20, 40, 30] as const

export interface Renderer {
  applyTransforms(transforms: Float32Array): void
  setCamera(state: OrbitCameraState): void
  resize(widthPx: number, heightPx: number, pixelRatio: number): void
  render(): void
  dispose(): void
}
export type RendererFactory = (canvas: HTMLCanvasElement, scene: SceneDescription) => Promise<Renderer>

/** Adapter: three.js WebGPURenderer (automatic WebGL 2 fallback, D18) drawing the greybox scene from theme tokens. */
export class SceneRenderer implements Renderer {
  private readonly scene = new THREE.Scene()
  private readonly camera = new THREE.PerspectiveCamera(FIELD_OF_VIEW_DEG, 1, NEAR_PLANE_M, FAR_PLANE_M)
  private readonly partMeshes: THREE.Mesh[] = []
  private readonly materials = new Map<string, THREE.MeshStandardNodeMaterial>()

  private constructor(
    private readonly renderer: THREE.WebGPURenderer,
    description: SceneDescription,
    private readonly theme: Theme
  ) {
    this.scene.background = new THREE.Color(theme.scene.sky)
    this.scene.add(new THREE.AmbientLight(theme.scene.light, theme.scene.ambientIntensity))
    const sun = new THREE.DirectionalLight(theme.scene.light, theme.scene.sunIntensity)
    sun.position.set(...SUN_POSITION_M)
    this.scene.add(sun)
    for (const box of description.terrain) {
      const mesh = new THREE.Mesh(new THREE.BoxGeometry(box.halfExtentsM[0] * 2, box.halfExtentsM[1] * 2, box.halfExtentsM[2] * 2), this.material(theme.scene.materials[box.material]))
      mesh.position.set(...box.centerM)
      this.scene.add(mesh)
    }
    for (const part of description.parts) {
      const mesh = new THREE.Mesh(this.geometryFor(part), this.material(this.colorFor(part)))
      this.partMeshes.push(mesh)
      this.scene.add(mesh)
    }
  }

  static async create(canvas: HTMLCanvasElement, description: SceneDescription, theme: Theme): Promise<SceneRenderer> {
    const renderer = new THREE.WebGPURenderer({ canvas, antialias: true })
    await renderer.init()
    return new SceneRenderer(renderer, description, theme)
  }

  applyTransforms(transforms: Float32Array): void {
    this.partMeshes.forEach((mesh, index) => {
      const offset = index * TRANSFORM_STRIDE
      const t = (i: number): number => transforms[offset + i] ?? 0
      mesh.position.set(t(0), t(1), t(2))
      mesh.quaternion.set(t(3), t(4), t(5), t(6))
    })
  }

  setCamera(state: OrbitCameraState): void {
    this.camera.position.set(...cameraPositionM(state))
    this.camera.lookAt(...state.targetM)
  }

  resize(widthPx: number, heightPx: number, pixelRatio: number): void {
    this.renderer.setPixelRatio(pixelRatio)
    this.renderer.setSize(widthPx, heightPx, false)
    this.camera.aspect = widthPx / Math.max(heightPx, 1)
    this.camera.updateProjectionMatrix()
  }

  render(): void {
    void this.renderer.renderAsync(this.scene, this.camera)
  }

  dispose(): void {
    this.scene.traverse((object) => {
      if (object instanceof THREE.Mesh) object.geometry.dispose()
    })
    for (const material of this.materials.values()) material.dispose()
    this.renderer.dispose()
  }

  private geometryFor(part: ScenePart): THREE.BufferGeometry {
    if (part.shape.kind === 'sphere') return new THREE.SphereGeometry(part.shape.radiusM, SPHERE_SEGMENTS, SPHERE_SEGMENTS)
    const [hx, hy, hz] = part.shape.halfExtentsM
    return new THREE.BoxGeometry(hx * 2, hy * 2, hz * 2)
  }

  private colorFor(part: ScenePart): string {
    return part.layer === 'projectile' ? this.theme.scene.projectile : this.theme.scene.materials[part.material]
  }

  private material(color: string): THREE.MeshStandardNodeMaterial {
    let material = this.materials.get(color)
    if (!material) {
      material = new THREE.MeshStandardNodeMaterial({ color })
      this.materials.set(color, material)
    }
    return material
  }
}
```
(The renderer has no unit test: jsdom has neither WebGPU nor WebGL. It is exercised by the e2e flow and the visual checks in Task 21.)

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm vitest run --project web render`
Expected: PASS.

- [ ] **Step 5: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test && pnpm --filter @physics-lab/web build
git add apps/web
git commit -m "feat(web): add orbit camera, frame loop and three.js greybox renderer"
```

---
### Task 18: `apps/web` — number formatting, lab view state and the challenge panel

**Files:**
- Create: `apps/web/src/format/number-format.ts`, `world/lab-reducer.ts`, `world/ChallengePanel.tsx`, `world/MeasurementTable.tsx`, `world/PredictionForm.tsx`, `world/ComparisonView.tsx`, `world/ExplanationForm.tsx`, `world/RunControls.tsx`, `src/test/challenge-fixtures.ts`
- Modify: `apps/web/src/i18n/messages/en.json`, `apps/web/src/i18n/messages/es.json`
- Test: `apps/web/src/format/number-format.test.ts`, `world/lab-reducer.test.ts`, `world/ChallengePanel.test.tsx`

**Interfaces:**
- Consumes: `Measurement`, `Unit`, `splitQuantityId` (formats); `Comparison`, `LabPhase` (challenges); `SceneDescription`, `ChallengeView`, `SceneEvent` (Task 16); `Locale` (Task 15).
- Produces:
```ts
// format/number-format.ts
export function parseLocaleNumber(text: string, locale: Locale): number | null
export function fractionDigitsFor(uncertainty: number): number      // uncertainty shown with 2 significant digits
export function formatMeasurement(value: { value: number; uncertainty: number; unit: Unit }, intl: IntlShape): string
export function formatValue(value: number, uncertainty: number, unit: Unit, intl: IntlShape): string
export function unitSymbol(unit: Unit): string
// world/lab-reducer.ts
export type Announcement = { serial: number } & ({ kind: 'event'; event: SceneEvent } | { kind: 'measured'; measurement: Measurement })
export type ErrorView = { code: string; params: Readonly<Record<string, string | number>> }
export type LabViewState = {
  status: 'loading' | 'ready' | 'failed'; phase: LabPhase; scene: SceneDescription | null; challenge: ChallengeView | null
  measurements: Measurement[]; predictions: Readonly<Record<string, number>>; comparisons: Readonly<Record<string, Comparison>>
  explanations: Readonly<Record<string, string>>; announcement: Announcement | null; error: ErrorView | null; timeScale: number
}
export type LabAction =
  | { type: 'initialized'; scene: SceneDescription; challenge: ChallengeView; phase: LabPhase }
  | { type: 'frame'; phase: LabPhase; events: readonly SceneEvent[] }
  | { type: 'measured'; measurement: Measurement }
  | { type: 'predicted'; taskId: string; value: number }
  | { type: 'compared'; comparison: Comparison }
  | { type: 'explained'; taskId: string; text: string }
  | { type: 'failed'; error: ErrorView }
  | { type: 'errorShown'; error: ErrorView }
  | { type: 'errorDismissed' }
  | { type: 'timeScaleChanged'; factor: number }
export const INITIAL_LAB_STATE: LabViewState
export function labReducer(state: LabViewState, action: LabAction): LabViewState
export function latestMeasurement(state: LabViewState, quantity: string): Measurement | undefined
export function allPredicted(state: LabViewState): boolean
// world/ChallengePanel.tsx
export type LabActions = {
  launch(): void; continueRun(): void; reset(): void; measure(quantity: string): void; predict(taskId: string, value: number): void
  compare(taskId: string): void; explain(taskId: string, text: string): void; exportResult(): void; setTimeScale(factor: number): void; dismissError(): void
}
export const TIME_SCALE_OPTIONS = [0.25, 0.5, 1, 2] as const
export function ChallengePanel(props: { state: LabViewState; locale: Locale; studentId: string; seed: number; actions: LabActions }): JSX.Element
```

Panel rules: Launch only in `ready`; release measurements in `calibrated`/`running`/`landed`; Continue only in `calibrated` with every prediction saved; task measurements only in `landed` with predictions saved; the prediction form locks once its quantity was measured; Compare needs a prediction and a measurement; Download needs a comparison for every task. Disabled controls always have visible text explaining the step (the step bodies), so the next action is discoverable.

- [ ] **Step 1: Add the world messages**

Merge into `apps/web/src/i18n/messages/en.json`:
```json
{
  "world.back": "Back to home",
  "world.seedInfo": "Student: {studentId} · Seed: {seed}",
  "world.canvasLabel": "3D view of the catapult experiment",
  "world.canvasHelp": "Focus the view and use the arrow keys to rotate the camera, + and − to zoom and Home to reset. The camera buttons do the same.",
  "world.smallScreen.title": "This lab needs a larger screen",
  "world.smallScreen.body": "The 3D lab works on a desktop or laptop with a mouse and keyboard. You can read the task here and continue on a computer.",
  "world.phase.ready": "Ready to launch",
  "world.phase.calibrating": "Launching…",
  "world.phase.calibrated": "Paused right after release",
  "world.phase.running": "Running",
  "world.phase.landed": "The ball has landed",
  "world.step.launch.heading": "1. Test launch",
  "world.step.launch.body": "Launch the catapult once. The ball pauses right after release so you can measure it.",
  "world.launch": "Launch test",
  "world.step.measureRelease.heading": "2. Measure the release",
  "world.step.measureRelease.body": "Measure the speed, angle and height of the ball at release. You can repeat a measurement.",
  "world.measure": "Measure: {quantity}",
  "world.step.predict.heading": "3. Predict",
  "world.prediction.label": "Your prediction ({unit})",
  "world.prediction.submit": "Save prediction",
  "world.prediction.saved": "Saved prediction: {value} {unit}",
  "world.prediction.locked": "Your prediction is locked because you already measured the result.",
  "world.prediction.error": "Enter a number, for example {example}.",
  "world.step.run.heading": "4. Launch and measure",
  "world.step.run.body": "Continue the launch after saving your prediction, then measure where the ball lands.",
  "world.continue": "Continue the launch",
  "world.step.compare.heading": "5. Compare",
  "world.compare": "Compare prediction and measurement",
  "world.comparison.prediction": "Your prediction",
  "world.comparison.measurement": "Measurement",
  "world.comparison.theory": "Theory from your measured inputs",
  "world.comparison.combined": "Allowed difference (k·u_c, k = {coverage})",
  "world.comparison.passed": "Within the expected uncertainty",
  "world.comparison.failed": "Outside the expected uncertainty",
  "world.step.explain.heading": "6. Explain and download",
  "world.explanation.label": "Explain any difference between your prediction, the theory and the measurement",
  "world.explanation.save": "Save explanation",
  "world.explanation.saved": "Explanation saved.",
  "world.export": "Download result file",
  "world.export.help": "Upload this file where your teacher asks. It contains your measurements and the steps needed to verify them.",
  "world.reset": "Reset experiment",
  "world.timeScale.label": "Simulation speed",
  "world.timeScale.option": "{factor}×",
  "world.measurements.caption": "Measurements",
  "world.measurements.quantity": "Quantity",
  "world.measurements.value": "Value",
  "world.measurements.instrument": "Instrument",
  "world.measurements.empty": "No measurements yet.",
  "world.error.dismiss": "Dismiss",
  "scene.released": "The ball was released.",
  "scene.pausedAtRelease": "Paused right after release. Measure the release now.",
  "scene.landed": "The ball hit the ground.",
  "scene.measured": "{quantity}: {value}"
}
```

Merge into `apps/web/src/i18n/messages/es.json`:
```json
{
  "world.back": "Volver al inicio",
  "world.seedInfo": "Estudiante: {studentId} · Semilla: {seed}",
  "world.canvasLabel": "Vista 3D del experimento de la catapulta",
  "world.canvasHelp": "Enfoca la vista y usa las flechas para girar la cámara, + y − para acercar o alejar e Inicio para restablecerla. Los botones de cámara hacen lo mismo.",
  "world.smallScreen.title": "Este laboratorio necesita una pantalla más grande",
  "world.smallScreen.body": "El laboratorio 3D funciona en un ordenador de escritorio o portátil con ratón y teclado. Puedes leer la tarea aquí y continuar en un ordenador.",
  "world.phase.ready": "Listo para lanzar",
  "world.phase.calibrating": "Lanzando…",
  "world.phase.calibrated": "En pausa justo después de soltar la bola",
  "world.phase.running": "En marcha",
  "world.phase.landed": "La bola ha caído",
  "world.step.launch.heading": "1. Lanzamiento de prueba",
  "world.step.launch.body": "Lanza la catapulta una vez. La bola se pausa justo después de soltarse para que puedas medirla.",
  "world.launch": "Lanzamiento de prueba",
  "world.step.measureRelease.heading": "2. Mide la liberación",
  "world.step.measureRelease.body": "Mide la rapidez, el ángulo y la altura de la bola al soltarse. Puedes repetir una medida.",
  "world.measure": "Medir: {quantity}",
  "world.step.predict.heading": "3. Predice",
  "world.prediction.label": "Tu predicción ({unit})",
  "world.prediction.submit": "Guardar predicción",
  "world.prediction.saved": "Predicción guardada: {value} {unit}",
  "world.prediction.locked": "Tu predicción está bloqueada porque ya mediste el resultado.",
  "world.prediction.error": "Escribe un número, por ejemplo {example}.",
  "world.step.run.heading": "4. Lanza y mide",
  "world.step.run.body": "Continúa el lanzamiento después de guardar tu predicción y luego mide dónde cae la bola.",
  "world.continue": "Continuar el lanzamiento",
  "world.step.compare.heading": "5. Compara",
  "world.compare": "Comparar predicción y medida",
  "world.comparison.prediction": "Tu predicción",
  "world.comparison.measurement": "Medida",
  "world.comparison.theory": "Teoría con tus medidas de entrada",
  "world.comparison.combined": "Diferencia permitida (k·u_c, k = {coverage})",
  "world.comparison.passed": "Dentro de la incertidumbre esperada",
  "world.comparison.failed": "Fuera de la incertidumbre esperada",
  "world.step.explain.heading": "6. Explica y descarga",
  "world.explanation.label": "Explica cualquier diferencia entre tu predicción, la teoría y la medida",
  "world.explanation.save": "Guardar explicación",
  "world.explanation.saved": "Explicación guardada.",
  "world.export": "Descargar archivo de resultados",
  "world.export.help": "Sube este archivo donde te indique tu docente. Contiene tus medidas y los pasos necesarios para verificarlas.",
  "world.reset": "Reiniciar experimento",
  "world.timeScale.label": "Velocidad de la simulación",
  "world.timeScale.option": "{factor}×",
  "world.measurements.caption": "Medidas",
  "world.measurements.quantity": "Magnitud",
  "world.measurements.value": "Valor",
  "world.measurements.instrument": "Instrumento",
  "world.measurements.empty": "Todavía no hay medidas.",
  "world.error.dismiss": "Cerrar",
  "scene.released": "Se soltó la bola.",
  "scene.pausedAtRelease": "En pausa justo después de soltarse. Mide la liberación ahora.",
  "scene.landed": "La bola tocó el suelo.",
  "scene.measured": "{quantity}: {value}"
}
```

- [ ] **Step 2: Write the failing tests**

`apps/web/src/test/challenge-fixtures.ts`:
```ts
import type { Comparison } from '@physics-lab/challenges'
import type { Measurement } from '@physics-lab/formats'
import type { ChallengeView, SceneDescription } from '@/sim/protocol'
import { INITIAL_LAB_STATE, type LabViewState } from '@/world/lab-reducer'

export const CATAPULT_VIEW: ChallengeView = {
  id: 'catapult-range',
  title: { es: 'Catapulta: alcance de un proyectil', en: 'Catapult: projectile range' },
  tasks: [{ id: 'predict-range', prompt: { es: 'Predice el alcance.', en: 'Predict the range.' }, quantity: 'flight.range', unit: 'm' }],
  calibrationQuantities: ['flight.releaseSpeed', 'flight.releaseAngle', 'flight.releaseHeight'],
  allowedCommands: ['help', 'measure', 'reset', 'time'],
}

export const EMPTY_SCENE: SceneDescription = { terrain: [], parts: [], focusM: [10, 2.6, 32] }

export const RANGE_MEASUREMENT: Measurement = { quantity: 'flight.range', instrument: 'ruler', value: 15.24, uncertainty: 0.0058, unit: 'm' }
export const SPEED_MEASUREMENT: Measurement = { quantity: 'flight.releaseSpeed', instrument: 'photogate', value: 11.764705882352942, uncertainty: 0.42, unit: 'm/s' }

export const PASSING_COMPARISON: Comparison = {
  taskId: 'predict-range', prediction: 15.1, measurement: RANGE_MEASUREMENT, theory: { value: 15.2, uncertainty: 0.9 },
  combinedUncertainty: 0.91, coverage: 2, passed: true,
}

export function labState(overrides: Partial<LabViewState> = {}): LabViewState {
  return { ...INITIAL_LAB_STATE, status: 'ready', scene: EMPTY_SCENE, challenge: CATAPULT_VIEW, ...overrides }
}
```

`apps/web/src/format/number-format.test.ts`:
```ts
import { createIntl } from 'react-intl'
import { describe, expect, it } from 'vitest'
import { formatMeasurement, formatValue, fractionDigitsFor, parseLocaleNumber } from './number-format'

describe('parseLocaleNumber (Review Focus 1)', () => {
  it('accepts a decimal comma or point in Spanish', () => {
    expect(parseLocaleNumber('12,5', 'es')).toBe(12.5)
    expect(parseLocaleNumber(' 12.5 ', 'es')).toBe(12.5)
    expect(parseLocaleNumber('-3', 'es')).toBe(-3)
  })

  it('accepts only a decimal point in English', () => {
    expect(parseLocaleNumber('12.5', 'en')).toBe(12.5)
    expect(parseLocaleNumber('12,5', 'en')).toBeNull()
  })

  it('rejects grouping separators, empty and non-numeric text instead of misreading them', () => {
    for (const text of ['1.234,5', '1,234.5', '', 'abc', '12,', '1e3']) {
      expect(parseLocaleNumber(text, 'es')).toBeNull()
    }
  })
})

describe('formatMeasurement', () => {
  const en = createIntl({ locale: 'en' })
  const es = createIntl({ locale: 'es' })

  it('shows the uncertainty with two significant digits and the value to the same decimal place', () => {
    expect(fractionDigitsFor(0.0058)).toBe(4)
    expect(fractionDigitsFor(0.42)).toBe(2)
    expect(fractionDigitsFor(3.2)).toBe(1)
    expect(formatMeasurement({ value: 11.764705882352942, uncertainty: 0.42, unit: 'm/s' }, en)).toBe('11.76 ± 0.42 m/s')
  })

  it('uses the locale decimal separator', () => {
    expect(formatMeasurement({ value: 15.24, uncertainty: 0.0058, unit: 'm' }, es)).toBe('15,2400 ± 0,0058 m')
  })

  it('formats a bare value with the decimals of a reference uncertainty', () => {
    expect(formatValue(15.1, 0.0058, 'm', en)).toBe('15.1000 m')
  })

  it('writes degrees with the degree sign', () => {
    expect(formatMeasurement({ value: 44.5, uncertainty: 0.3, unit: 'deg' }, en)).toBe('44.50 ± 0.30 °')
  })
})
```

`apps/web/src/world/lab-reducer.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { CATAPULT_VIEW, EMPTY_SCENE, PASSING_COMPARISON, RANGE_MEASUREMENT, SPEED_MEASUREMENT, labState } from '@/test/challenge-fixtures'
import { INITIAL_LAB_STATE, allPredicted, labReducer, latestMeasurement } from './lab-reducer'

describe('labReducer', () => {
  it('becomes ready on initialization', () => {
    const state = labReducer(INITIAL_LAB_STATE, { type: 'initialized', scene: EMPTY_SCENE, challenge: CATAPULT_VIEW, phase: 'ready' })
    expect(state.status).toBe('ready')
    expect(state.challenge).toBe(CATAPULT_VIEW)
  })

  it('announces scene events with an increasing serial', () => {
    const first = labReducer(labState(), { type: 'frame', phase: 'calibrated', events: ['released', 'pausedAtRelease'] })
    expect(first.phase).toBe('calibrated')
    expect(first.announcement).toMatchObject({ kind: 'event', event: 'pausedAtRelease' })
    const second = labReducer(first, { type: 'frame', phase: 'running', events: [] })
    expect(second.announcement).toBe(first.announcement)
    const third = labReducer(second, { type: 'frame', phase: 'landed', events: ['landed'] })
    expect(third.announcement?.serial).toBeGreaterThan(first.announcement?.serial ?? 0)
  })

  it('records measurements, keeps the latest per quantity and announces them', () => {
    const later = { ...SPEED_MEASUREMENT, value: 12 }
    const state = [SPEED_MEASUREMENT, RANGE_MEASUREMENT, later].reduce((s, m) => labReducer(s, { type: 'measured', measurement: m }), labState())
    expect(state.measurements).toHaveLength(3)
    expect(latestMeasurement(state, 'flight.releaseSpeed')?.value).toBe(12)
    expect(state.announcement).toMatchObject({ kind: 'measured', measurement: later })
  })

  it('tracks predictions, comparisons, explanations, errors and time scale', () => {
    let state = labReducer(labState(), { type: 'predicted', taskId: 'predict-range', value: 15 })
    expect(allPredicted(state)).toBe(true)
    state = labReducer(state, { type: 'compared', comparison: PASSING_COMPARISON })
    state = labReducer(state, { type: 'explained', taskId: 'predict-range', text: 'ok' })
    state = labReducer(state, { type: 'errorShown', error: { code: 'invalidPhase', params: {} } })
    expect(state.error?.code).toBe('invalidPhase')
    state = labReducer(state, { type: 'errorDismissed' })
    state = labReducer(state, { type: 'timeScaleChanged', factor: 0.25 })
    expect(state).toMatchObject({ comparisons: { 'predict-range': PASSING_COMPARISON }, explanations: { 'predict-range': 'ok' }, error: null, timeScale: 0.25 })
  })

  it('fails with an error view', () => {
    expect(labReducer(INITIAL_LAB_STATE, { type: 'failed', error: { code: 'internal', params: {} } })).toMatchObject({ status: 'failed', error: { code: 'internal' } })
  })
})
```

`apps/web/src/world/ChallengePanel.test.tsx`:
```tsx
import { screen, within } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, expect, it, vi } from 'vitest'
import { PASSING_COMPARISON, RANGE_MEASUREMENT, SPEED_MEASUREMENT, labState } from '@/test/challenge-fixtures'
import { expectNoAxeViolations } from '@/test/axe'
import { renderWithIntl } from '@/test/render'
import type { LabViewState } from './lab-reducer'
import { ChallengePanel, type LabActions } from './ChallengePanel'

function actions(): LabActions {
  return {
    launch: vi.fn(), continueRun: vi.fn(), reset: vi.fn(), measure: vi.fn(), predict: vi.fn(), compare: vi.fn(),
    explain: vi.fn(), exportResult: vi.fn(), setTimeScale: vi.fn(), dismissError: vi.fn(),
  }
}

function renderPanel(state: LabViewState, locale: 'en' | 'es' = 'en') {
  const handlers = actions()
  const view = renderWithIntl(<ChallengePanel state={state} locale={locale} studentId="ana" seed={12345} actions={handlers} />, locale)
  return { handlers, view }
}

describe('ChallengePanel', () => {
  it('only allows the test launch before anything else', async () => {
    const { handlers } = renderPanel(labState({ phase: 'ready' }))
    expect(screen.getByRole('button', { name: 'Continue the launch' })).toBeDisabled()
    expect(screen.getByRole('button', { name: 'Measure: Release speed' })).toBeDisabled()
    await userEvent.click(screen.getByRole('button', { name: 'Launch test' }))
    expect(handlers.launch).toHaveBeenCalled()
  })

  it('enables release measurements at the calibration pause and keeps continue locked until predicted', async () => {
    const { handlers } = renderPanel(labState({ phase: 'calibrated' }))
    await userEvent.click(screen.getByRole('button', { name: 'Measure: Release speed' }))
    expect(handlers.measure).toHaveBeenCalledWith('flight.releaseSpeed')
    expect(screen.getByRole('button', { name: 'Continue the launch' })).toBeDisabled()
    expect(screen.getByRole('button', { name: 'Measure: Horizontal range' })).toBeDisabled()
  })

  it('saves a prediction typed with a decimal comma in Spanish', async () => {
    const { handlers } = renderPanel(labState({ phase: 'calibrated' }), 'es')
    await userEvent.type(screen.getByLabelText('Tu predicción (m)'), '15,2')
    await userEvent.click(screen.getByRole('button', { name: 'Guardar predicción' }))
    expect(handlers.predict).toHaveBeenCalledWith('predict-range', 15.2)
  })

  it('rejects a non-numeric prediction with an inline error', async () => {
    const { handlers } = renderPanel(labState({ phase: 'calibrated' }))
    const input = screen.getByLabelText('Your prediction (m)')
    await userEvent.type(input, 'far')
    await userEvent.click(screen.getByRole('button', { name: 'Save prediction' }))
    expect(handlers.predict).not.toHaveBeenCalled()
    expect(input).toHaveAttribute('aria-invalid', 'true')
    expect(input).toHaveFocus()
  })

  it('locks the prediction once the range was measured and enables comparison', async () => {
    const { handlers } = renderPanel(labState({ phase: 'landed', predictions: { 'predict-range': 15.1 }, measurements: [RANGE_MEASUREMENT] }))
    expect(screen.getByLabelText('Your prediction (m)')).toBeDisabled()
    expect(screen.getByText('Your prediction is locked because you already measured the result.')).toBeInTheDocument()
    await userEvent.click(screen.getByRole('button', { name: 'Compare prediction and measurement' }))
    expect(handlers.compare).toHaveBeenCalledWith('predict-range')
  })

  it('shows the verdict in words, not only color, and enables the download', async () => {
    const { handlers } = renderPanel(labState({ phase: 'landed', predictions: { 'predict-range': 15.1 }, measurements: [RANGE_MEASUREMENT], comparisons: { 'predict-range': PASSING_COMPARISON } }))
    expect(screen.getByText('Within the expected uncertainty')).toBeInTheDocument()
    await userEvent.click(screen.getByRole('button', { name: 'Download result file' }))
    expect(handlers.exportResult).toHaveBeenCalled()
  })

  it('lists measurements in an accessible table', () => {
    renderPanel(labState({ phase: 'calibrated', measurements: [SPEED_MEASUREMENT] }))
    const table = screen.getByRole('table', { name: 'Measurements' })
    expect(within(table).getByText('11.76 ± 0.42 m/s')).toBeInTheDocument()
    expect(within(table).getByText('Photogate')).toBeInTheDocument()
  })

  it('shows translated errors with a dismiss button', async () => {
    const { handlers } = renderPanel(labState({ error: { code: 'predictionRequired', params: {} } }))
    expect(screen.getByRole('alert')).toHaveTextContent('Submit your prediction first.')
    await userEvent.click(screen.getByRole('button', { name: 'Dismiss' }))
    expect(handlers.dismissError).toHaveBeenCalled()
  })

  it('changes the simulation speed', async () => {
    const { handlers } = renderPanel(labState())
    await userEvent.selectOptions(screen.getByLabelText('Simulation speed'), '0.25')
    expect(handlers.setTimeScale).toHaveBeenCalledWith(0.25)
  })

  it('has no axe violations in a busy state', async () => {
    const { view } = renderPanel(labState({ phase: 'landed', predictions: { 'predict-range': 15.1 }, measurements: [SPEED_MEASUREMENT, RANGE_MEASUREMENT], comparisons: { 'predict-range': PASSING_COMPARISON } }))
    await expectNoAxeViolations(view.container)
  })
})
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `pnpm vitest run --project web format world`
Expected: FAIL — modules not found.

- [ ] **Step 4: Implement formatting and the reducer**

`apps/web/src/format/number-format.ts`:
```ts
import type { Unit } from '@physics-lab/formats'
import type { IntlShape } from 'react-intl'
import type { Locale } from '@/i18n/locales'

/** One decimal separator only; grouping is never accepted, so "1.234,5" cannot be misread (Review Focus 1). */
const DECIMAL_PATTERNS: Readonly<Record<Locale, RegExp>> = {
  en: /^[+-]?\d+(?:\.\d+)?$/,
  es: /^[+-]?\d+(?:[.,]\d+)?$/,
}
const UNIT_SYMBOLS: Readonly<Record<Unit, string>> = { m: 'm', s: 's', 'm/s': 'm/s', deg: '°', N: 'N', kg: 'kg' }
const UNCERTAINTY_SIGNIFICANT_DIGITS = 2
const MAX_FRACTION_DIGITS = 6
const DEFAULT_FRACTION_DIGITS = 3

export function parseLocaleNumber(text: string, locale: Locale): number | null {
  const trimmed = text.trim()
  if (!DECIMAL_PATTERNS[locale].test(trimmed)) return null
  return Number(trimmed.replace(',', '.'))
}

export function unitSymbol(unit: Unit): string {
  return UNIT_SYMBOLS[unit]
}

export function fractionDigitsFor(uncertainty: number): number {
  if (!(uncertainty > 0)) return DEFAULT_FRACTION_DIGITS
  const digits = UNCERTAINTY_SIGNIFICANT_DIGITS - 1 - Math.floor(Math.log10(uncertainty))
  return Math.min(Math.max(digits, 0), MAX_FRACTION_DIGITS)
}

function formatNumberLike(value: number, uncertainty: number, intl: IntlShape): string {
  const digits = fractionDigitsFor(uncertainty)
  return intl.formatNumber(value, { minimumFractionDigits: digits, maximumFractionDigits: digits })
}

/** A value shown with the decimals its uncertainty justifies, plus the unit symbol. */
export function formatValue(value: number, uncertainty: number, unit: Unit, intl: IntlShape): string {
  return `${formatNumberLike(value, uncertainty, intl)} ${unitSymbol(unit)}`
}

/** "value ± u unit" with u to two significant digits (spec §6). The ± notation is not translated (guidelines §5). */
export function formatMeasurement(reading: { value: number; uncertainty: number; unit: Unit }, intl: IntlShape): string {
  const value = formatNumberLike(reading.value, reading.uncertainty, intl)
  const uncertainty = formatNumberLike(reading.uncertainty, reading.uncertainty, intl)
  return `${value} ± ${uncertainty} ${unitSymbol(reading.unit)}`
}
```

`apps/web/src/world/lab-reducer.ts`:
```ts
import type { Comparison, LabPhase } from '@physics-lab/challenges'
import type { Measurement } from '@physics-lab/formats'
import type { ChallengeView, SceneDescription, SceneEvent } from '@/sim/protocol'

export type Announcement = { serial: number } & ({ kind: 'event'; event: SceneEvent } | { kind: 'measured'; measurement: Measurement })
export type ErrorView = { code: string; params: Readonly<Record<string, string | number>> }
export type LabViewState = {
  status: 'loading' | 'ready' | 'failed'
  phase: LabPhase
  scene: SceneDescription | null
  challenge: ChallengeView | null
  measurements: Measurement[]
  predictions: Readonly<Record<string, number>>
  comparisons: Readonly<Record<string, Comparison>>
  explanations: Readonly<Record<string, string>>
  announcement: Announcement | null
  error: ErrorView | null
  timeScale: number
}
export type LabAction =
  | { type: 'initialized'; scene: SceneDescription; challenge: ChallengeView; phase: LabPhase }
  | { type: 'frame'; phase: LabPhase; events: readonly SceneEvent[] }
  | { type: 'measured'; measurement: Measurement }
  | { type: 'predicted'; taskId: string; value: number }
  | { type: 'compared'; comparison: Comparison }
  | { type: 'explained'; taskId: string; text: string }
  | { type: 'failed'; error: ErrorView }
  | { type: 'errorShown'; error: ErrorView }
  | { type: 'errorDismissed' }
  | { type: 'timeScaleChanged'; factor: number }

export const INITIAL_LAB_STATE: LabViewState = {
  status: 'loading', phase: 'ready', scene: null, challenge: null, measurements: [], predictions: {}, comparisons: {},
  explanations: {}, announcement: null, error: null, timeScale: 1,
}

function nextSerial(state: LabViewState): number {
  return (state.announcement?.serial ?? 0) + 1
}

export function labReducer(state: LabViewState, action: LabAction): LabViewState {
  switch (action.type) {
    case 'initialized':
      return { ...state, status: 'ready', scene: action.scene, challenge: action.challenge, phase: action.phase }
    case 'frame': {
      const event = action.events.at(-1)
      return { ...state, phase: action.phase, announcement: event ? { serial: nextSerial(state), kind: 'event', event } : state.announcement }
    }
    case 'measured':
      return { ...state, measurements: [...state.measurements, action.measurement], announcement: { serial: nextSerial(state), kind: 'measured', measurement: action.measurement } }
    case 'predicted':
      return { ...state, predictions: { ...state.predictions, [action.taskId]: action.value } }
    case 'compared':
      return { ...state, comparisons: { ...state.comparisons, [action.comparison.taskId]: action.comparison } }
    case 'explained':
      return { ...state, explanations: { ...state.explanations, [action.taskId]: action.text } }
    case 'failed':
      return { ...state, status: 'failed', error: action.error }
    case 'errorShown':
      return { ...state, error: action.error }
    case 'errorDismissed':
      return { ...state, error: null }
    case 'timeScaleChanged':
      return { ...state, timeScale: action.factor }
  }
}

export function latestMeasurement(state: LabViewState, quantity: string): Measurement | undefined {
  return state.measurements.findLast((measurement) => measurement.quantity === quantity)
}

export function allPredicted(state: LabViewState): boolean {
  return (state.challenge?.tasks ?? []).every((task) => state.predictions[task.id] !== undefined)
}
```

- [ ] **Step 5: Implement the panel components**

`apps/web/src/world/MeasurementTable.tsx`:
```tsx
import { splitQuantityId, type Measurement } from '@physics-lab/formats'
import { useIntl } from 'react-intl'
import { Table, TableBody, TableCaption, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table'
import { formatMeasurement } from '@/format/number-format'

export function MeasurementTable({ measurements }: { measurements: readonly Measurement[] }) {
  const intl = useIntl()
  return (
    <Table>
      <TableCaption className="caption-top text-left text-base font-semibold text-foreground">
        {intl.formatMessage({ id: 'world.measurements.caption' })}
      </TableCaption>
      <TableHeader>
        <TableRow>
          <TableHead scope="col">{intl.formatMessage({ id: 'world.measurements.quantity' })}</TableHead>
          <TableHead scope="col">{intl.formatMessage({ id: 'world.measurements.value' })}</TableHead>
          <TableHead scope="col">{intl.formatMessage({ id: 'world.measurements.instrument' })}</TableHead>
        </TableRow>
      </TableHeader>
      <TableBody>
        {measurements.length === 0 ? (
          <TableRow>
            <TableCell colSpan={3}>{intl.formatMessage({ id: 'world.measurements.empty' })}</TableCell>
          </TableRow>
        ) : (
          measurements.map((measurement, index) => (
            <TableRow key={index}>
              <TableCell>{intl.formatMessage({ id: `quantity.${splitQuantityId(measurement.quantity).reading}` })}</TableCell>
              <TableCell className="tabular-nums">{formatMeasurement(measurement, intl)}</TableCell>
              <TableCell>{intl.formatMessage({ id: `instrument.${measurement.instrument}` })}</TableCell>
            </TableRow>
          ))
        )}
      </TableBody>
    </Table>
  )
}
```

`apps/web/src/world/PredictionForm.tsx`:
```tsx
import { useId, useRef, useState, type FormEvent } from 'react'
import { useIntl } from 'react-intl'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import { parseLocaleNumber } from '@/format/number-format'
import type { Locale } from '@/i18n/locales'
import type { TaskView } from '@/sim/protocol'

const EXAMPLE_VALUE = 12.5

type PredictionFormProps = { task: TaskView; locale: Locale; saved: number | undefined; locked: boolean; onSubmit(value: number): void }

export function PredictionForm({ task, locale, saved, locked, onSubmit }: PredictionFormProps) {
  const intl = useIntl()
  const [text, setText] = useState('')
  const [invalid, setInvalid] = useState(false)
  const inputRef = useRef<HTMLInputElement>(null)
  const id = useId()
  const errorId = `${id}-error`

  function handleSubmit(event: FormEvent<HTMLFormElement>): void {
    event.preventDefault()
    const value = parseLocaleNumber(text, locale)
    setInvalid(value === null)
    if (value === null) return inputRef.current?.focus()
    onSubmit(value)
  }

  return (
    <form noValidate onSubmit={handleSubmit} className="flex flex-col gap-2">
      <p>{task.prompt[locale]}</p>
      <Label htmlFor={id}>{intl.formatMessage({ id: 'world.prediction.label' }, { unit: task.unit })}</Label>
      <div className="flex flex-wrap gap-2">
        <Input
          ref={inputRef}
          id={id}
          inputMode="decimal"
          autoComplete="off"
          className="min-h-11 max-w-40"
          value={text}
          disabled={locked}
          onChange={(event) => setText(event.target.value)}
          aria-invalid={invalid}
          aria-describedby={invalid ? errorId : undefined}
        />
        <Button type="submit" className="min-h-11" disabled={locked}>{intl.formatMessage({ id: 'world.prediction.submit' })}</Button>
      </div>
      {invalid && (
        <p id={errorId} className="text-sm font-medium text-destructive">
          {intl.formatMessage({ id: 'world.prediction.error' }, { example: intl.formatNumber(EXAMPLE_VALUE) })}
        </p>
      )}
      {saved !== undefined && <p className="text-sm">{intl.formatMessage({ id: 'world.prediction.saved' }, { value: intl.formatNumber(saved), unit: task.unit })}</p>}
      {locked && <p className="text-sm text-muted-foreground">{intl.formatMessage({ id: 'world.prediction.locked' })}</p>}
    </form>
  )
}
```

`apps/web/src/world/ComparisonView.tsx`:
```tsx
import type { Comparison } from '@physics-lab/challenges'
import { Check, X } from 'lucide-react'
import { useIntl } from 'react-intl'
import { formatMeasurement, formatValue } from '@/format/number-format'

export function ComparisonView({ comparison }: { comparison: Comparison }) {
  const intl = useIntl()
  const unit = comparison.measurement.unit
  const allowed = comparison.coverage * comparison.combinedUncertainty
  const verdictId = comparison.passed ? 'world.comparison.passed' : 'world.comparison.failed'
  const Icon = comparison.passed ? Check : X
  return (
    <div className="flex flex-col gap-3">
      <dl className="grid grid-cols-[auto_1fr] gap-x-4 gap-y-1 tabular-nums">
        <dt>{intl.formatMessage({ id: 'world.comparison.prediction' })}</dt>
        <dd>{formatValue(comparison.prediction, comparison.measurement.uncertainty, unit, intl)}</dd>
        <dt>{intl.formatMessage({ id: 'world.comparison.measurement' })}</dt>
        <dd>{formatMeasurement(comparison.measurement, intl)}</dd>
        <dt>{intl.formatMessage({ id: 'world.comparison.theory' })}</dt>
        <dd>{formatMeasurement({ ...comparison.theory, unit }, intl)}</dd>
        <dt>{intl.formatMessage({ id: 'world.comparison.combined' }, { coverage: comparison.coverage })}</dt>
        <dd>{formatValue(allowed, comparison.combinedUncertainty, unit, intl)}</dd>
      </dl>
      <p className={comparison.passed ? 'flex items-center gap-2 font-semibold text-success' : 'flex items-center gap-2 font-semibold text-destructive'}>
        <Icon aria-hidden="true" className="size-5" />
        {intl.formatMessage({ id: verdictId })}
      </p>
    </div>
  )
}
```

`apps/web/src/world/ExplanationForm.tsx`:
```tsx
import { MAX_EXPLANATION_CHARS } from '@physics-lab/formats'
import { useId, useState, type FormEvent } from 'react'
import { useIntl } from 'react-intl'
import { Button } from '@/components/ui/button'
import { Label } from '@/components/ui/label'
import { Textarea } from '@/components/ui/textarea'

export function ExplanationForm({ saved, onSubmit }: { saved: string | undefined; onSubmit(text: string): void }) {
  const intl = useIntl()
  const [text, setText] = useState(saved ?? '')
  const id = useId()
  function handleSubmit(event: FormEvent<HTMLFormElement>): void {
    event.preventDefault()
    onSubmit(text)
  }
  return (
    <form onSubmit={handleSubmit} className="flex flex-col gap-2">
      <Label htmlFor={id}>{intl.formatMessage({ id: 'world.explanation.label' })}</Label>
      <Textarea id={id} rows={4} maxLength={MAX_EXPLANATION_CHARS} value={text} onChange={(event) => setText(event.target.value)} />
      <Button type="submit" variant="secondary" className="min-h-11 self-start">{intl.formatMessage({ id: 'world.explanation.save' })}</Button>
      {saved !== undefined && <p className="text-sm">{intl.formatMessage({ id: 'world.explanation.saved' })}</p>}
    </form>
  )
}
```

`apps/web/src/world/RunControls.tsx`:
```tsx
import { useId } from 'react'
import { useIntl } from 'react-intl'
import { Button } from '@/components/ui/button'
import { Label } from '@/components/ui/label'

export const TIME_SCALE_OPTIONS = [0.25, 0.5, 1, 2] as const

type RunControlsProps = { timeScale: number; onReset(): void; onTimeScale(factor: number): void }

export function RunControls({ timeScale, onReset, onTimeScale }: RunControlsProps) {
  const intl = useIntl()
  const id = useId()
  return (
    <div className="flex flex-wrap items-end gap-4">
      <Button type="button" variant="secondary" className="min-h-11" onClick={onReset}>{intl.formatMessage({ id: 'world.reset' })}</Button>
      <div className="flex flex-col gap-1">
        <Label htmlFor={id}>{intl.formatMessage({ id: 'world.timeScale.label' })}</Label>
        <select id={id} className="min-h-11 rounded-md border border-input bg-card px-3" value={timeScale} onChange={(event) => onTimeScale(Number(event.target.value))}>
          {TIME_SCALE_OPTIONS.map((factor) => (
            <option key={factor} value={factor}>{intl.formatMessage({ id: 'world.timeScale.option' }, { factor: intl.formatNumber(factor) })}</option>
          ))}
        </select>
      </div>
    </div>
  )
}
```

`apps/web/src/world/ChallengePanel.tsx`:
```tsx
import type { LabPhase } from '@physics-lab/challenges'
import { splitQuantityId } from '@physics-lab/formats'
import { useId, type ReactNode } from 'react'
import { useIntl } from 'react-intl'
import { Button } from '@/components/ui/button'
import type { Locale } from '@/i18n/locales'
import { ComparisonView } from './ComparisonView'
import { ExplanationForm } from './ExplanationForm'
import { allPredicted, latestMeasurement, type LabViewState } from './lab-reducer'
import { MeasurementTable } from './MeasurementTable'
import { PredictionForm } from './PredictionForm'
import { RunControls } from './RunControls'

export { TIME_SCALE_OPTIONS } from './RunControls'

export type LabActions = {
  launch(): void
  continueRun(): void
  reset(): void
  measure(quantity: string): void
  predict(taskId: string, value: number): void
  compare(taskId: string): void
  explain(taskId: string, text: string): void
  exportResult(): void
  setTimeScale(factor: number): void
  dismissError(): void
}

const AFTER_RELEASE: ReadonlySet<LabPhase> = new Set(['calibrated', 'running', 'landed'])

function Step({ heading, children }: { heading: string; children: ReactNode }) {
  const id = useId()
  return (
    <section aria-labelledby={id} className="flex flex-col gap-3">
      <h3 id={id} className="text-lg font-semibold">{heading}</h3>
      {children}
    </section>
  )
}

type ChallengePanelProps = { state: LabViewState; locale: Locale; studentId: string; seed: number; actions: LabActions }

export function ChallengePanel({ state, locale, studentId, seed, actions }: ChallengePanelProps) {
  const intl = useIntl()
  const headingId = useId()
  const challenge = state.challenge
  if (!challenge) return null
  const t = (id: string, values?: Record<string, string | number>): string => intl.formatMessage({ id }, values)
  const quantityLabel = (quantity: string): string => t(`quantity.${splitQuantityId(quantity).reading}`)
  const predicted = allPredicted(state)
  const allCompared = challenge.tasks.every((task) => state.comparisons[task.id] !== undefined)
  const measureButton = (quantity: string, enabled: boolean) => (
    <Button key={quantity} type="button" variant="secondary" className="min-h-11 justify-start" disabled={!enabled} onClick={() => actions.measure(quantity)}>
      {t('world.measure', { quantity: quantityLabel(quantity) })}
    </Button>
  )

  return (
    <aside aria-labelledby={headingId} className="flex flex-col gap-6 overflow-y-auto bg-card p-4">
      <header className="flex flex-col gap-1">
        <h2 id={headingId} className="text-xl font-bold">{challenge.title[locale]}</h2>
        <p className="text-sm text-muted-foreground">{t('world.seedInfo', { studentId, seed })}</p>
        <p className="font-medium">{t(`world.phase.${state.phase}`)}</p>
      </header>
      {state.error && (
        <div role="alert" className="flex flex-wrap items-center justify-between gap-2 rounded-md border border-destructive p-3 text-destructive">
          <span>{t(`error.${state.error.code}`, { ...state.error.params })}</span>
          <Button type="button" variant="secondary" className="min-h-11" onClick={actions.dismissError}>{t('world.error.dismiss')}</Button>
        </div>
      )}
      <Step heading={t('world.step.launch.heading')}>
        <p>{t('world.step.launch.body')}</p>
        <Button type="button" className="min-h-11 self-start" disabled={state.phase !== 'ready'} onClick={actions.launch}>{t('world.launch')}</Button>
      </Step>
      <Step heading={t('world.step.measureRelease.heading')}>
        <p>{t('world.step.measureRelease.body')}</p>
        <div className="flex flex-col gap-2">{challenge.calibrationQuantities.map((q) => measureButton(q, AFTER_RELEASE.has(state.phase)))}</div>
      </Step>
      <Step heading={t('world.step.predict.heading')}>
        {challenge.tasks.map((task) => (
          <PredictionForm
            key={task.id}
            task={task}
            locale={locale}
            saved={state.predictions[task.id]}
            locked={latestMeasurement(state, task.quantity) !== undefined}
            onSubmit={(value) => actions.predict(task.id, value)}
          />
        ))}
      </Step>
      <Step heading={t('world.step.run.heading')}>
        <p>{t('world.step.run.body')}</p>
        <Button type="button" className="min-h-11 self-start" disabled={state.phase !== 'calibrated' || !predicted} onClick={actions.continueRun}>{t('world.continue')}</Button>
        <div className="flex flex-col gap-2">{challenge.tasks.map((task) => measureButton(task.quantity, state.phase === 'landed' && predicted))}</div>
      </Step>
      <Step heading={t('world.step.compare.heading')}>
        {challenge.tasks.map((task) => {
          const comparison = state.comparisons[task.id]
          const ready = state.predictions[task.id] !== undefined && latestMeasurement(state, task.quantity) !== undefined
          return (
            <div key={task.id} className="flex flex-col gap-3">
              <Button type="button" className="min-h-11 self-start" disabled={!ready} onClick={() => actions.compare(task.id)}>{t('world.compare')}</Button>
              {comparison && <ComparisonView comparison={comparison} />}
            </div>
          )
        })}
      </Step>
      <Step heading={t('world.step.explain.heading')}>
        {challenge.tasks.map((task) => (
          <ExplanationForm key={task.id} saved={state.explanations[task.id]} onSubmit={(text) => actions.explain(task.id, text)} />
        ))}
        <p className="text-sm text-muted-foreground">{t('world.export.help')}</p>
        <Button type="button" className="min-h-11 self-start" disabled={!allCompared} onClick={actions.exportResult}>{t('world.export')}</Button>
      </Step>
      <MeasurementTable measurements={state.measurements} />
      <RunControls timeScale={state.timeScale} onReset={actions.reset} onTimeScale={actions.setTimeScale} />
    </aside>
  )
}
```

- [ ] **Step 6: Run the tests to verify they pass**

Run: `pnpm vitest run --project web`
Expected: PASS (including `messages.test.ts`: both catalogs got the same new keys).

- [ ] **Step 7: Verify and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test
git add apps/web
git commit -m "feat(web): add challenge panel with measurements, prediction and comparison"
```

---
### Task 19: `apps/web` — World screen: canvas, camera controls, console, live region, App route

**Files:**
- Create: `apps/web/src/world/WorldScreen.tsx`, `world/WorldCanvas.tsx`, `world/CameraControls.tsx`, `world/SceneStatusRegion.tsx`, `world/use-media-query.ts`, `console/ConsolePanel.tsx`, `io/download-json.ts`, `src/test/fake-sim-client.ts`, `src/test/fake-renderer.ts`, `src/test/fake-scheduler.ts`
- Modify: `apps/web/src/app/App.tsx`, `app/dependencies.ts`, `main.tsx`, `test/setup.ts`, `render/frame-loop.test.ts` (import the shared `FakeScheduler`), `i18n/messages/{en,es}.json`
- Test: `apps/web/src/world/WorldScreen.test.tsx`, `io/download-json.test.ts`

**Interfaces:**
- Consumes: Tasks 15–18; `createConsole`, `M1_CONSOLE_COMMANDS`, `ConsoleOutcome` (console); `Command` (formats).
- Produces:
```ts
// app/dependencies.ts
export type AppDependencies = {
  appVersion: string
  randomSeed(): number
  createSimClient(): SimClient
  createRenderer: RendererFactory
  scheduler: FrameScheduler
  downloadJson(fileName: string, json: string): void
}
// io/download-json.ts
export function resultFileName(challengeId: string, studentId: string): string   // physics-lab-<challenge>-<slug>.json
export function downloadJson(fileName: string, json: string): void               // browser adapter
// world/WorldScreen.tsx
export const DESKTOP_QUERY = '(min-width: 64rem) and (pointer: fine)'
export function WorldScreen(props: { dependencies: AppDependencies; locale: Locale; studentId: string; seed: number; onExit(): void }): JSX.Element
// world/WorldCanvas.tsx
export type TransformBuffers = { previous: Float32Array; current: Float32Array }
// console/ConsolePanel.tsx
export function ConsolePanel(props: { allowedCommands: readonly string[]; onClose(): void; onTimeScale(factor: number): void; runCommand(command: Command): Promise<string> }): JSX.Element
// test fakes
export class FakeSimClient implements SimClient { phase: LabPhase; failNext: SimRequestError | null; readonly calls: string[] }
export class FakeRenderer implements Renderer { readonly cameras: OrbitCameraState[]; renders: number }
export function fakeRendererFactory(): { factory: RendererFactory; renderers: FakeRenderer[] }
export class FakeScheduler implements FrameScheduler { fire(timeMs: number): void; get scheduled(): number }
```

Keyboard map (WCAG 2.1.1, 2.5.7): `/` or `T` (outside text fields) opens the console with focus in its input; `Escape` closes it and returns focus to where it was; arrow keys, `+`, `−`, `Home` on the focused 3D view move the camera; seven camera buttons (≥ 44 px) do the same without dragging. Pointer drag on the view also orbits.

- [ ] **Step 1: Add messages**

Merge into `en.json`:
```json
{
  "world.camera.group": "Camera",
  "world.camera.rotateLeft": "Rotate camera left",
  "world.camera.rotateRight": "Rotate camera right",
  "world.camera.rotateUp": "Tilt camera up",
  "world.camera.rotateDown": "Tilt camera down",
  "world.camera.zoomIn": "Zoom in",
  "world.camera.zoomOut": "Zoom out",
  "world.camera.reset": "Reset camera",
  "world.console.open": "Command console (/ or T)",
  "world.console.title": "Command console",
  "world.console.label": "Command",
  "world.console.close": "Close console",
  "world.console.helpIntro": "Available commands:",
  "world.console.done": "Done.",
  "world.console.unsupported": "This command is not available in this challenge yet."
}
```

Merge into `es.json`:
```json
{
  "world.camera.group": "Cámara",
  "world.camera.rotateLeft": "Girar la cámara a la izquierda",
  "world.camera.rotateRight": "Girar la cámara a la derecha",
  "world.camera.rotateUp": "Inclinar la cámara hacia arriba",
  "world.camera.rotateDown": "Inclinar la cámara hacia abajo",
  "world.camera.zoomIn": "Acercar",
  "world.camera.zoomOut": "Alejar",
  "world.camera.reset": "Restablecer la cámara",
  "world.console.open": "Consola de comandos (/ o T)",
  "world.console.title": "Consola de comandos",
  "world.console.label": "Comando",
  "world.console.close": "Cerrar la consola",
  "world.console.helpIntro": "Comandos disponibles:",
  "world.console.done": "Hecho.",
  "world.console.unsupported": "Este comando todavía no está disponible en este reto."
}
```

- [ ] **Step 2: Test fakes**

`apps/web/src/test/fake-scheduler.ts` (move the class from `frame-loop.test.ts` here unchanged and import it there):
```ts
import type { FrameScheduler } from '@/render/frame-loop'

export class FakeScheduler implements FrameScheduler {
  private callbacks = new Map<number, (timeMs: number) => void>()
  private nextHandle = 1
  requestFrame(callback: (timeMs: number) => void): number {
    const handle = this.nextHandle++
    this.callbacks.set(handle, callback)
    return handle
  }
  cancelFrame(handle: number): void {
    this.callbacks.delete(handle)
  }
  fire(timeMs: number): void {
    const pending = [...this.callbacks.values()]
    this.callbacks.clear()
    for (const callback of pending) callback(timeMs)
  }
  get scheduled(): number {
    return this.callbacks.size
  }
}
```

`apps/web/src/test/fake-renderer.ts`:
```ts
import type { OrbitCameraState } from '@/render/orbit-camera'
import type { Renderer, RendererFactory } from '@/render/scene-renderer'

export class FakeRenderer implements Renderer {
  readonly cameras: OrbitCameraState[] = []
  renders = 0
  disposed = false
  applyTransforms(): void {}
  setCamera(state: OrbitCameraState): void {
    this.cameras.push(state)
  }
  resize(): void {}
  render(): void {
    this.renders += 1
  }
  dispose(): void {
    this.disposed = true
  }
}

export function fakeRendererFactory(): { factory: RendererFactory; renderers: FakeRenderer[] } {
  const renderers: FakeRenderer[] = []
  const factory: RendererFactory = () => {
    const renderer = new FakeRenderer()
    renderers.push(renderer)
    return Promise.resolve(renderer)
  }
  return { factory, renderers }
}
```

`apps/web/src/test/fake-sim-client.ts`:
```ts
import type { Comparison, LabPhase } from '@physics-lab/challenges'
import type { Measurement } from '@physics-lab/formats'
import type { FrameUpdate, SceneEvent, SimResponse } from '@/sim/protocol'
import type { SimClient, SimRequestError } from '@/sim/sim-worker-client'
import { CATAPULT_VIEW, EMPTY_SCENE, PASSING_COMPARISON, RANGE_MEASUREMENT, SPEED_MEASUREMENT } from './challenge-fixtures'

/** In-memory SimClient with a scripted catapult flow (fakes over mocks, D30). */
export class FakeSimClient implements SimClient {
  phase: LabPhase = 'ready'
  failNext: SimRequestError | null = null
  readonly calls: string[] = []

  init(): Promise<Extract<SimResponse, { type: 'initialized' }>> {
    return this.answer('init', () => ({ type: 'initialized', scene: EMPTY_SCENE, challenge: CATAPULT_VIEW, frame: this.frame() }))
  }
  advance(steps: number): Promise<FrameUpdate> {
    return this.answer(`advance:${steps}`, () => this.frame())
  }
  launch(): Promise<FrameUpdate> {
    return this.answer('launch', () => this.transition('calibrated', ['released', 'pausedAtRelease']))
  }
  continueRun(): Promise<FrameUpdate> {
    return this.answer('continueRun', () => this.transition('landed', ['landed']))
  }
  reset(): Promise<FrameUpdate> {
    return this.answer('reset', () => this.transition('ready', []))
  }
  measure(quantity: string): Promise<{ measurement: Measurement; frame: FrameUpdate }> {
    const measurement = quantity === 'flight.range' ? RANGE_MEASUREMENT : { ...SPEED_MEASUREMENT, quantity }
    return this.answer(`measure:${quantity}`, () => ({ measurement, frame: this.frame() }))
  }
  predict(taskId: string, value: number): Promise<void> {
    return this.answer(`predict:${taskId}:${value}`, () => undefined)
  }
  compare(taskId: string): Promise<Comparison> {
    return this.answer(`compare:${taskId}`, () => PASSING_COMPARISON)
  }
  explain(taskId: string): Promise<void> {
    return this.answer(`explain:${taskId}`, () => undefined)
  }
  exportResult(): Promise<string> {
    return this.answer('exportResult', () => '{"format":"physics-lab/result"}')
  }
  benchmark(bodies: number, steps: number): Promise<number> {
    return this.answer(`benchmark:${bodies}:${steps}`, () => 0.8)
  }
  dispose(): void {
    this.calls.push('dispose')
  }

  private transition(phase: LabPhase, events: SceneEvent[]): FrameUpdate {
    this.phase = phase
    return this.frame(events)
  }

  private frame(events: SceneEvent[] = []): FrameUpdate {
    return { phase: this.phase, stepIndex: 0, transforms: new Float32Array(0), events }
  }

  private answer<T>(call: string, value: () => T): Promise<T> {
    this.calls.push(call)
    const failure = this.failNext
    this.failNext = null
    return failure ? Promise.reject(failure) : Promise.resolve(value())
  }
}
```

Append to `apps/web/src/test/setup.ts` (jsdom has no `matchMedia`; desktop by default, tests override per case):
```ts
if (!('matchMedia' in window)) {
  Object.defineProperty(window, 'matchMedia', {
    writable: true,
    value: (query: string): MediaQueryList =>
      ({ matches: true, media: query, onchange: null, addEventListener() {}, removeEventListener() {}, addListener() {}, removeListener() {}, dispatchEvent: () => false }) as MediaQueryList,
  })
}
```

- [ ] **Step 3: Write the failing tests**

`apps/web/src/io/download-json.test.ts`:
```ts
import { describe, expect, it } from 'vitest'
import { resultFileName } from './download-json'

describe('resultFileName', () => {
  it('builds a safe, readable file name', () => {
    expect(resultFileName('catapult-range', 'José Pérez')).toBe('physics-lab-catapult-range-jose-perez.json')
    expect(resultFileName('catapult-range', '***')).toBe('physics-lab-catapult-range-student.json')
  })
})
```

`apps/web/src/world/WorldScreen.test.tsx`:
```tsx
import { act, screen, waitFor, within } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, expect, it, vi } from 'vitest'
import type { AppDependencies } from '@/app/dependencies'
import { SimRequestError } from '@/sim/sim-worker-client'
import { expectNoAxeViolations } from '@/test/axe'
import { fakeRendererFactory } from '@/test/fake-renderer'
import { FakeScheduler } from '@/test/fake-scheduler'
import { FakeSimClient } from '@/test/fake-sim-client'
import { renderWithIntl } from '@/test/render'
import { WorldScreen } from './WorldScreen'

function setup() {
  const client = new FakeSimClient()
  const scheduler = new FakeScheduler()
  const { factory, renderers } = fakeRendererFactory()
  const downloadJson = vi.fn()
  const onExit = vi.fn()
  const dependencies: AppDependencies = {
    appVersion: 'test', randomSeed: () => 1, createSimClient: () => client, createRenderer: factory, scheduler, downloadJson,
  }
  const view = renderWithIntl(<WorldScreen dependencies={dependencies} locale="en" studentId="ana" seed={12345} onExit={onExit} />)
  return { client, scheduler, renderers, downloadJson, onExit, view }
}

async function ready(): Promise<void> {
  await screen.findByRole('heading', { name: 'Catapult: projectile range' })
}

describe('WorldScreen', () => {
  it('initializes the session and shows the 3D view with keyboard help', async () => {
    const { client } = setup()
    await ready()
    expect(client.calls[0]).toBe('init')
    expect(screen.getByRole('application', { name: '3D view of the catapult experiment' })).toHaveAccessibleDescription(/arrow keys/)
  })

  it('announces the calibration pause in the live region', async () => {
    setup()
    await ready()
    await userEvent.click(screen.getByRole('button', { name: 'Launch test' }))
    expect(await screen.findByText('Paused right after release. Measure the release now.')).toBeInTheDocument()
    expect(screen.getByText('Paused right after release')).toBeInTheDocument()
  })

  it('runs the flow end to end and downloads the result file', async () => {
    const { client, downloadJson } = setup()
    await ready()
    await userEvent.click(screen.getByRole('button', { name: 'Launch test' }))
    await userEvent.click(await screen.findByRole('button', { name: 'Measure: Release speed' }))
    await userEvent.type(screen.getByLabelText('Your prediction (m)'), '15.1')
    await userEvent.click(screen.getByRole('button', { name: 'Save prediction' }))
    await userEvent.click(await screen.findByRole('button', { name: 'Continue the launch' }))
    await userEvent.click(await screen.findByRole('button', { name: 'Measure: Horizontal range' }))
    await userEvent.click(await screen.findByRole('button', { name: 'Compare prediction and measurement' }))
    await userEvent.click(await screen.findByRole('button', { name: 'Download result file' }))
    await waitFor(() => expect(downloadJson).toHaveBeenCalledWith('physics-lab-catapult-range-ana.json', '{"format":"physics-lab/result"}'))
    expect(client.calls).toEqual(expect.arrayContaining(['predict:predict-range:15.1', 'measure:flight.range', 'compare:predict-range']))
  })

  it('shows refused actions as translated alerts', async () => {
    const { client } = setup()
    await ready()
    client.failNext = new SimRequestError('invalidPhase', {})
    await userEvent.click(screen.getByRole('button', { name: 'Launch test' }))
    expect(await screen.findByRole('alert')).toHaveTextContent('That action is not available right now.')
  })

  it('opens the console with "/", runs commands, enforces permissions and restores focus on Escape', async () => {
    setup()
    await ready()
    const launch = screen.getByRole('button', { name: 'Launch test' })
    launch.focus()
    await userEvent.keyboard('/')
    const input = screen.getByLabelText('Command')
    expect(input).toHaveFocus()
    await userEvent.type(input, '/measure flight.releaseSpeed{Enter}')
    const log = screen.getByRole('log')
    expect(await within(log).findByText('Release speed: 11.76 ± 0.42 m/s')).toBeInTheDocument()
    await userEvent.type(input, '/set #latch.releaseAtHingeAngleDeg 80{Enter}')
    expect(await within(log).findByText('/set is not allowed in this challenge.')).toBeInTheDocument()
    await userEvent.keyboard('{Escape}')
    expect(screen.queryByLabelText('Command')).not.toBeInTheDocument()
    expect(launch).toHaveFocus()
  })

  it('does not open the console while typing "t" in a text field', async () => {
    setup()
    await ready()
    await userEvent.type(screen.getByLabelText(/Explain any difference/), 'test')
    expect(screen.queryByLabelText('Command')).not.toBeInTheDocument()
  })

  it('moves the camera with the keyboard and with buttons', async () => {
    const { scheduler, renderers } = setup()
    await ready()
    await waitFor(() => expect(renderers).toHaveLength(1))
    screen.getByRole('application').focus()
    await userEvent.keyboard('{ArrowLeft}')
    await userEvent.click(screen.getByRole('button', { name: 'Zoom in' }))
    await act(async () => {
      scheduler.fire(0)
      await Promise.resolve()
    })
    const camera = renderers[0]?.cameras.at(-1)
    expect(camera?.azimuthDeg).toBe(355)
    expect(camera?.distanceM).toBeLessThan(24)
  })

  it('shows a notice instead of the 3D view on small or touch-only screens', async () => {
    vi.spyOn(window, 'matchMedia').mockImplementation((query: string) => ({ matches: false, media: query, onchange: null, addEventListener() {}, removeEventListener() {}, addListener() {}, removeListener() {}, dispatchEvent: () => false }) as MediaQueryList)
    setup()
    expect(await screen.findByText('This lab needs a larger screen')).toBeInTheDocument()
    expect(screen.queryByRole('application')).not.toBeInTheDocument()
    vi.restoreAllMocks()
  })

  it('has no axe violations', async () => {
    const { view } = setup()
    await ready()
    await expectNoAxeViolations(view.container)
  })
})
```

- [ ] **Step 4: Run the tests to verify they fail**

Run: `pnpm vitest run --project web WorldScreen download-json`
Expected: FAIL — modules not found.

- [ ] **Step 5: Implement**

`apps/web/src/io/download-json.ts`:
```ts
const MAX_SLUG_CHARS = 40
const FALLBACK_SLUG = 'student'
const REVOKE_DELAY_MS = 1000

export function resultFileName(challengeId: string, studentId: string): string {
  const slug = studentId
    .normalize('NFKD')
    .replace(/[̀-ͯ]/g, '')
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-+|-+$/g, '')
    .slice(0, MAX_SLUG_CHARS)
  return `physics-lab-${challengeId}-${slug || FALLBACK_SLUG}.json`
}

/** Browser adapter: saves a local file; nothing leaves the device (D20). */
export function downloadJson(fileName: string, json: string): void {
  const url = URL.createObjectURL(new Blob([json], { type: 'application/json' }))
  const link = document.createElement('a')
  link.href = url
  link.download = fileName
  link.click()
  setTimeout(() => URL.revokeObjectURL(url), REVOKE_DELAY_MS)
}
```

`apps/web/src/world/use-media-query.ts`:
```ts
import { useEffect, useState } from 'react'

export function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(() => window.matchMedia(query).matches)
  useEffect(() => {
    const list = window.matchMedia(query)
    const update = (): void => setMatches(list.matches)
    update()
    list.addEventListener('change', update)
    return () => list.removeEventListener('change', update)
  }, [query])
  return matches
}
```

`apps/web/src/world/SceneStatusRegion.tsx`:
```tsx
import { splitQuantityId } from '@physics-lab/formats'
import { useIntl } from 'react-intl'
import { formatMeasurement } from '@/format/number-format'
import type { Announcement } from './lab-reducer'

/** Screen-reader scene-state region (D14): canvas events become text. Re-keyed per announcement so repeats are read. */
export function SceneStatusRegion({ announcement }: { announcement: Announcement | null }) {
  const intl = useIntl()
  let text = ''
  if (announcement?.kind === 'event') text = intl.formatMessage({ id: `scene.${announcement.event}` })
  if (announcement?.kind === 'measured') {
    const quantity = intl.formatMessage({ id: `quantity.${splitQuantityId(announcement.measurement.quantity).reading}` })
    text = intl.formatMessage({ id: 'scene.measured' }, { quantity, value: formatMeasurement(announcement.measurement, intl) })
  }
  return (
    <div role="status" aria-live="polite" className="sr-only">
      <span key={announcement?.serial ?? 0}>{text}</span>
    </div>
  )
}
```

`apps/web/src/world/CameraControls.tsx`:
```tsx
import { ArrowDown, ArrowUp, House, RotateCcw, RotateCw, ZoomIn, ZoomOut, type LucideIcon } from 'lucide-react'
import { useIntl } from 'react-intl'
import { Button } from '@/components/ui/button'
import { ORBIT_COMMANDS, type OrbitCommand } from '@/render/orbit-camera'

const ICONS: Readonly<Record<OrbitCommand, LucideIcon>> = {
  rotateLeft: RotateCcw, rotateRight: RotateCw, rotateUp: ArrowUp, rotateDown: ArrowDown, zoomIn: ZoomIn, zoomOut: ZoomOut, reset: House,
}

/** Single-click alternative to dragging the camera (WCAG 2.5.7); targets are 44 px (2.5.8). */
export function CameraControls({ onCommand }: { onCommand(command: OrbitCommand): void }) {
  const intl = useIntl()
  return (
    <div role="group" aria-label={intl.formatMessage({ id: 'world.camera.group' })} className="flex flex-wrap gap-1 rounded-md bg-card/90 p-1">
      {ORBIT_COMMANDS.map((command) => {
        const Icon = ICONS[command]
        return (
          <Button key={command} type="button" variant="secondary" size="icon" className="size-11" aria-label={intl.formatMessage({ id: `world.camera.${command}` })} onClick={() => onCommand(command)}>
            <Icon aria-hidden="true" />
          </Button>
        )
      })}
    </div>
  )
}
```

`apps/web/src/world/WorldCanvas.tsx`:
```tsx
import type { LabPhase } from '@physics-lab/challenges'
import { useEffect, useId, useRef, type MutableRefObject } from 'react'
import { useIntl } from 'react-intl'
import { FrameLoop, type FramePlan, type FrameScheduler } from '@/render/frame-loop'
import { interpolateTransforms } from '@/render/interpolation'
import { orbitCommandForKey, type OrbitCameraState, type OrbitCommand } from '@/render/orbit-camera'
import type { Renderer, RendererFactory } from '@/render/scene-renderer'
import type { FrameUpdate, SceneDescription } from '@/sim/protocol'
import type { SimClient } from '@/sim/sim-worker-client'

export type TransformBuffers = { previous: Float32Array; current: Float32Array }

const STEPPING_PHASES: ReadonlySet<LabPhase> = new Set(['calibrating', 'running', 'landed'])

type WorldCanvasProps = {
  client: SimClient
  scene: SceneDescription
  transforms: MutableRefObject<TransformBuffers>
  phase: LabPhase
  timeScale: number
  camera: OrbitCameraState
  createRenderer: RendererFactory
  scheduler: FrameScheduler
  onFrame(frame: FrameUpdate): void
  onCameraCommand(command: OrbitCommand): void
  onCameraDrag(dxPx: number, dyPx: number): void
  onError(error: unknown): void
}

export function WorldCanvas(props: WorldCanvasProps) {
  const intl = useIntl()
  const helpId = useId()
  const canvasRef = useRef<HTMLCanvasElement>(null)
  const latest = useRef(props)
  latest.current = props
  const dragFrom = useRef<{ x: number; y: number } | null>(null)
  const loopRef = useRef<FrameLoop | null>(null)

  useEffect(() => {
    const canvas = canvasRef.current
    if (!canvas) return
    let renderer: Renderer | null = null
    let disposed = false
    const scratch = { buffer: new Float32Array(0) }

    async function onFrame(plan: FramePlan): Promise<void> {
      const current = latest.current
      if (STEPPING_PHASES.has(current.phase) && plan.steps > 0) {
        try {
          const frame = await current.client.advance(plan.steps)
          current.transforms.current = { previous: current.transforms.current.current, current: frame.transforms }
          latest.current.onFrame(frame)
        } catch (error) {
          latest.current.onError(error)
        }
      }
      if (!renderer || disposed) return
      const { previous, current: now } = latest.current.transforms.current
      if (previous.length === now.length) {
        if (scratch.buffer.length !== now.length) scratch.buffer = new Float32Array(now.length)
        interpolateTransforms(previous, now, plan.alpha, scratch.buffer)
        renderer.applyTransforms(scratch.buffer)
      } else {
        renderer.applyTransforms(now)
      }
      renderer.setCamera(latest.current.camera)
      renderer.render()
    }

    const loop = new FrameLoop(props.scheduler, onFrame)
    loopRef.current = loop
    const observer = typeof ResizeObserver === 'undefined' ? null : new ResizeObserver(() => renderer?.resize(canvas.clientWidth, canvas.clientHeight, window.devicePixelRatio))
    observer?.observe(canvas)
    void props.createRenderer(canvas, props.scene).then((created) => {
      if (disposed) return created.dispose()
      renderer = created
      renderer.resize(canvas.clientWidth, canvas.clientHeight, window.devicePixelRatio)
      loop.start()
    }, props.onError)
    return () => {
      disposed = true
      loop.stop()
      observer?.disconnect()
      renderer?.dispose()
    }
    // The renderer is rebuilt only when the scene or adapters change; per-frame inputs flow through `latest`.
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [props.scene, props.createRenderer, props.scheduler])

  useEffect(() => {
    loopRef.current?.setTimeScale(props.timeScale)
  }, [props.timeScale])

  return (
    <>
      <canvas
        ref={canvasRef}
        tabIndex={0}
        role="application"
        aria-label={intl.formatMessage({ id: 'world.canvasLabel' })}
        aria-describedby={helpId}
        className="block h-full w-full touch-none"
        onKeyDown={(event) => {
          const command = orbitCommandForKey(event.key)
          if (!command) return
          event.preventDefault()
          props.onCameraCommand(command)
        }}
        onPointerDown={(event) => {
          event.currentTarget.setPointerCapture?.(event.pointerId)
          dragFrom.current = { x: event.clientX, y: event.clientY }
        }}
        onPointerMove={(event) => {
          if (!dragFrom.current) return
          props.onCameraDrag(event.clientX - dragFrom.current.x, event.clientY - dragFrom.current.y)
          dragFrom.current = { x: event.clientX, y: event.clientY }
        }}
        onPointerUp={() => {
          dragFrom.current = null
        }}
      />
      <p id={helpId} className="sr-only">{intl.formatMessage({ id: 'world.canvasHelp' })}</p>
    </>
  )
}
```
> The `eslint-disable-next-line` is the single justified exception in the app (the comment above it states why); reviewers reject any other.

`apps/web/src/console/ConsolePanel.tsx`:
```tsx
import { M1_CONSOLE_COMMANDS, createConsole, type ConsoleOutcome, type UiAction } from '@physics-lab/console'
import type { Command } from '@physics-lab/formats'
import { X } from 'lucide-react'
import { useEffect, useId, useMemo, useRef, useState, type FormEvent, type KeyboardEvent } from 'react'
import { useIntl } from 'react-intl'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'

type ConsolePanelProps = {
  allowedCommands: readonly string[]
  onClose(): void
  onTimeScale(factor: number): void
  runCommand(command: Command): Promise<string>
}

export function ConsolePanel({ allowedCommands, onClose, onTimeScale, runCommand }: ConsolePanelProps) {
  const intl = useIntl()
  const titleId = useId()
  const inputId = useId()
  const inputRef = useRef<HTMLInputElement>(null)
  const gameConsole = useMemo(() => createConsole(M1_CONSOLE_COMMANDS, { kind: 'only', names: allowedCommands }), [allowedCommands])
  const [line, setLine] = useState('')
  const [log, setLog] = useState<string[]>([])
  const [history, setHistory] = useState<string[]>([])
  const [historyIndex, setHistoryIndex] = useState<number | null>(null)

  useEffect(() => inputRef.current?.focus(), [])

  const t = (id: string, values?: Record<string, string | number>): string => intl.formatMessage({ id }, values)

  function helpLines(action: Extract<UiAction, { type: 'showHelp' }>): string[] {
    const names = action.command === null ? gameConsole.commandNames() : [action.command]
    if (action.command !== null && !gameConsole.commandNames().includes(action.command)) {
      return [t('console.error.unknownCommand', { name: action.command })]
    }
    const lines = names.map((name) => `${t(`console.usage.${name}`)} — ${t(`console.help.${name}`)}`)
    return action.command === null ? [t('world.console.helpIntro'), ...lines] : lines
  }

  async function outputFor(outcome: ConsoleOutcome): Promise<string[]> {
    switch (outcome.kind) {
      case 'error':
        return [t(outcome.message.key, { ...outcome.message.params })]
      case 'ui':
        if (outcome.action.type === 'showHelp') return helpLines(outcome.action)
        onTimeScale(outcome.action.factor)
        return [t('world.console.done')]
      case 'command':
        return [await runCommand(outcome.command)]
    }
  }

  async function handleSubmit(event: FormEvent<HTMLFormElement>): Promise<void> {
    event.preventDefault()
    const entered = line
    setLine('')
    setHistory((previous) => [...previous, entered])
    setHistoryIndex(null)
    const output = await outputFor(gameConsole.execute(entered))
    setLog((previous) => [...previous, `> ${entered}`, ...output])
  }

  function handleKeyDown(event: KeyboardEvent<HTMLInputElement>): void {
    if (event.key !== 'ArrowUp' && event.key !== 'ArrowDown') return
    event.preventDefault()
    if (history.length === 0) return
    const last = history.length - 1
    const next = event.key === 'ArrowUp' ? Math.max((historyIndex ?? history.length) - 1, 0) : Math.min((historyIndex ?? last) + 1, last)
    setHistoryIndex(next)
    setLine(history[next] ?? '')
  }

  return (
    <section
      aria-labelledby={titleId}
      className="absolute inset-x-2 bottom-2 flex max-h-[50%] flex-col gap-2 rounded-md border border-border bg-card p-3 shadow-lg"
      onKeyDown={(event) => {
        if (event.key === 'Escape') onClose()
      }}
    >
      <div className="flex items-center justify-between">
        <h2 id={titleId} className="font-semibold">{t('world.console.title')}</h2>
        <Button type="button" variant="secondary" size="icon" className="size-11" aria-label={t('world.console.close')} onClick={onClose}>
          <X aria-hidden="true" />
        </Button>
      </div>
      <div role="log" aria-live="polite" className="min-h-16 overflow-y-auto font-mono text-sm">
        {log.map((entry, index) => (
          <p key={index}>{entry}</p>
        ))}
      </div>
      <form onSubmit={(event) => void handleSubmit(event)} className="flex items-end gap-2">
        <div className="flex flex-1 flex-col gap-1">
          <Label htmlFor={inputId}>{t('world.console.label')}</Label>
          <Input ref={inputRef} id={inputId} autoComplete="off" spellCheck={false} className="min-h-11 font-mono" value={line} onChange={(event) => setLine(event.target.value)} onKeyDown={handleKeyDown} />
        </div>
      </form>
    </section>
  )
}
```

`apps/web/src/world/WorldScreen.tsx`:
```tsx
import { splitQuantityId, type Command } from '@physics-lab/formats'
import { Terminal } from 'lucide-react'
import { useCallback, useEffect, useReducer, useRef, useState } from 'react'
import { useIntl } from 'react-intl'
import type { AppDependencies } from '@/app/dependencies'
import { Button } from '@/components/ui/button'
import { ConsolePanel } from '@/console/ConsolePanel'
import { formatMeasurement } from '@/format/number-format'
import type { Locale } from '@/i18n/locales'
import { resultFileName } from '@/io/download-json'
import { applyOrbitCommand, dragOrbit, initialCamera, type OrbitCameraState, type OrbitCommand } from '@/render/orbit-camera'
import type { FrameUpdate } from '@/sim/protocol'
import { SimRequestError, type SimClient } from '@/sim/sim-worker-client'
import { CameraControls } from './CameraControls'
import { ChallengePanel, type LabActions } from './ChallengePanel'
import { INITIAL_LAB_STATE, labReducer, type ErrorView } from './lab-reducer'
import { SceneStatusRegion } from './SceneStatusRegion'
import { useMediaQuery } from './use-media-query'
import { WorldCanvas, type TransformBuffers } from './WorldCanvas'

export const DESKTOP_QUERY = '(min-width: 64rem) and (pointer: fine)'
const CHALLENGE_ID = 'catapult-range'
const CONSOLE_KEYS = new Set(['/', 't', 'T'])
const EMPTY_TRANSFORMS = new Float32Array(0)

function toErrorView(error: unknown): ErrorView {
  if (error instanceof SimRequestError) return { code: error.code, params: error.params }
  console.error(error)
  return { code: 'internal', params: {} }
}

function isEditable(target: EventTarget | null): boolean {
  return target instanceof HTMLElement && (target.isContentEditable || ['INPUT', 'TEXTAREA', 'SELECT'].includes(target.tagName))
}

type WorldScreenProps = { dependencies: AppDependencies; locale: Locale; studentId: string; seed: number; onExit(): void }

export function WorldScreen({ dependencies, locale, studentId, seed, onExit }: WorldScreenProps) {
  const intl = useIntl()
  const desktop = useMediaQuery(DESKTOP_QUERY)
  const [state, dispatch] = useReducer(labReducer, INITIAL_LAB_STATE)
  const [client, setClient] = useState<SimClient | null>(null)
  const [camera, setCamera] = useState<OrbitCameraState | null>(null)
  const [initialView, setInitialView] = useState<OrbitCameraState | null>(null)
  const [consoleOpen, setConsoleOpen] = useState(false)
  const transforms = useRef<TransformBuffers>({ previous: EMPTY_TRANSFORMS, current: EMPTY_TRANSFORMS })
  const opener = useRef<HTMLElement | null>(null)

  useEffect(() => {
    const created = dependencies.createSimClient()
    setClient(created)
    created
      .init({ challengeId: CHALLENGE_ID, seed, studentId, appVersion: dependencies.appVersion })
      .then((initialized) => {
        transforms.current = { previous: initialized.frame.transforms, current: initialized.frame.transforms }
        const view = initialCamera(initialized.scene.focusM)
        setInitialView(view)
        setCamera(view)
        dispatch({ type: 'initialized', scene: initialized.scene, challenge: initialized.challenge, phase: initialized.frame.phase })
      })
      .catch((error: unknown) => dispatch({ type: 'failed', error: toErrorView(error) }))
    return () => created.dispose()
  }, [dependencies, seed, studentId])

  const applyFrame = useCallback((frame: FrameUpdate): void => {
    if (frame.transforms.length > 0) transforms.current = { previous: frame.transforms, current: frame.transforms }
    dispatch({ type: 'frame', phase: frame.phase, events: frame.events })
  }, [])

  const attempt = useCallback(async (action: () => Promise<void>): Promise<void> => {
    try {
      await action()
    } catch (error) {
      dispatch({ type: 'errorShown', error: toErrorView(error) })
    }
  }, [])

  const closeConsole = useCallback((): void => {
    setConsoleOpen(false)
    opener.current?.focus()
  }, [])

  useEffect(() => {
    function onKeyDown(event: KeyboardEvent): void {
      if (consoleOpen || event.ctrlKey || event.metaKey || event.altKey || !CONSOLE_KEYS.has(event.key) || isEditable(event.target)) return
      event.preventDefault()
      opener.current = document.activeElement instanceof HTMLElement ? document.activeElement : null
      setConsoleOpen(true)
    }
    window.addEventListener('keydown', onKeyDown)
    return () => window.removeEventListener('keydown', onKeyDown)
  }, [consoleOpen])

  if (!client) return null
  const activeClient = client
  const t = (id: string, values?: Record<string, string | number>): string => intl.formatMessage({ id }, values)

  const actions: LabActions = {
    launch: () => void attempt(async () => applyFrame(await activeClient.launch())),
    continueRun: () => void attempt(async () => applyFrame(await activeClient.continueRun())),
    reset: () => void attempt(async () => applyFrame(await activeClient.reset())),
    measure: (quantity) =>
      void attempt(async () => {
        const { measurement, frame } = await activeClient.measure(quantity)
        dispatch({ type: 'measured', measurement })
        applyFrame(frame)
      }),
    predict: (taskId, value) =>
      void attempt(async () => {
        await activeClient.predict(taskId, value)
        dispatch({ type: 'predicted', taskId, value })
      }),
    compare: (taskId) => void attempt(async () => dispatch({ type: 'compared', comparison: await activeClient.compare(taskId) })),
    explain: (taskId, text) =>
      void attempt(async () => {
        await activeClient.explain(taskId, text)
        dispatch({ type: 'explained', taskId, text })
      }),
    exportResult: () => void attempt(async () => dependencies.downloadJson(resultFileName(CHALLENGE_ID, studentId), await activeClient.exportResult())),
    setTimeScale: (factor) => dispatch({ type: 'timeScaleChanged', factor }),
    dismissError: () => dispatch({ type: 'errorDismissed' }),
  }

  async function runCommand(command: Command): Promise<string> {
    try {
      if (command.type === 'measure') {
        const { measurement, frame } = await activeClient.measure(command.quantity)
        dispatch({ type: 'measured', measurement })
        applyFrame(frame)
        return `${t(`quantity.${splitQuantityId(measurement.quantity).reading}`)}: ${formatMeasurement(measurement, intl)}`
      }
      if (command.type === 'reset') {
        applyFrame(await activeClient.reset())
        return t('world.console.done')
      }
      return t('world.console.unsupported')
    } catch (error) {
      const view = toErrorView(error)
      return t(`error.${view.code}`, { ...view.params })
    }
  }

  const onCameraCommand = (command: OrbitCommand): void => setCamera((current) => (current && initialView ? applyOrbitCommand(current, command, initialView) : current))
  const onCameraDrag = (dx: number, dy: number): void => setCamera((current) => (current ? dragOrbit(current, dx, dy) : current))

  if (state.status === 'failed') {
    return (
      <main className="mx-auto flex max-w-prose flex-col gap-4 p-8">
        <p role="alert">{t(`error.${state.error?.code ?? 'internal'}`)}</p>
        <Button type="button" className="min-h-11 self-start" onClick={onExit}>{t('world.back')}</Button>
      </main>
    )
  }
  if (state.status === 'loading' || !state.scene || !camera) {
    return <p role="status" className="p-8">{t('app.loading')}</p>
  }
  if (!desktop) {
    return (
      <main className="mx-auto flex w-full max-w-prose flex-col gap-4 px-4 py-8">
        <h1 className="text-2xl font-bold">{t('world.smallScreen.title')}</h1>
        <p>{t('world.smallScreen.body')}</p>
        {state.challenge?.tasks.map((task) => <p key={task.id}>{task.prompt[locale]}</p>)}
        <Button type="button" className="min-h-11 self-start" onClick={onExit}>{t('world.back')}</Button>
      </main>
    )
  }

  return (
    <div className="grid h-dvh grid-cols-[minmax(0,1fr)_minmax(22rem,28rem)] grid-rows-[auto_minmax(0,1fr)]">
      <header className="col-span-2 flex items-center justify-between gap-4 border-b border-border bg-card px-4 py-2">
        <h1 className="text-xl font-bold">{t('app.title')}</h1>
        <div className="flex gap-2">
          <Button
            type="button"
            variant="secondary"
            className="min-h-11"
            onClick={(event) => {
              opener.current = event.currentTarget
              setConsoleOpen(true)
            }}
          >
            <Terminal aria-hidden="true" />
            {t('world.console.open')}
          </Button>
          <Button type="button" variant="secondary" className="min-h-11" onClick={onExit}>{t('world.back')}</Button>
        </div>
      </header>
      <main className="relative min-h-0">
        <WorldCanvas
          client={client}
          scene={state.scene}
          transforms={transforms}
          phase={state.phase}
          timeScale={state.timeScale}
          camera={camera}
          createRenderer={dependencies.createRenderer}
          scheduler={dependencies.scheduler}
          onFrame={applyFrame}
          onCameraCommand={onCameraCommand}
          onCameraDrag={onCameraDrag}
          onError={(error) => dispatch({ type: 'errorShown', error: toErrorView(error) })}
        />
        <div className="absolute left-2 top-2">
          <CameraControls onCommand={onCameraCommand} />
        </div>
        {consoleOpen && state.challenge && (
          <ConsolePanel
            allowedCommands={state.challenge.allowedCommands}
            onClose={closeConsole}
            onTimeScale={(factor) => dispatch({ type: 'timeScaleChanged', factor })}
            runCommand={runCommand}
          />
        )}
      </main>
      <div className="min-h-0 overflow-y-auto border-l border-border">
        <ChallengePanel state={state} locale={locale} studentId={studentId} seed={seed} actions={actions} />
      </div>
      <SceneStatusRegion announcement={state.announcement} />
    </div>
  )
}
```

`apps/web/src/app/dependencies.ts`:
```ts
import type { FrameScheduler } from '@/render/frame-loop'
import type { RendererFactory } from '@/render/scene-renderer'
import type { SimClient } from '@/sim/sim-worker-client'

/** Ports the UI needs; concrete adapters are created only in main.tsx (composition root, D30). */
export type AppDependencies = {
  appVersion: string
  randomSeed(): number
  createSimClient(): SimClient
  createRenderer: RendererFactory
  scheduler: FrameScheduler
  downloadJson(fileName: string, json: string): void
}
```

`apps/web/src/app/App.tsx` — replace the loading branch with the world route:
```tsx
      {started ? (
        <WorldScreen dependencies={dependencies} locale={locale} studentId={started.studentId} seed={started.seed} onExit={() => setStarted(null)} />
      ) : (
        <HomeScreen locale={locale} onLocaleChange={setLocale} onStart={setStarted} randomSeed={dependencies.randomSeed} />
      )}
```
(remove the now-unused `FormattedMessage` import; keep `app.loading` — WorldScreen uses it.)

`apps/web/src/main.tsx` — complete `dependencies`:
```ts
const dependencies: AppDependencies = {
  appVersion: __APP_VERSION__,
  randomSeed: () => crypto.getRandomValues(new Uint32Array(1))[0] ?? 0,
  createSimClient: () => new SimWorkerClient(new Worker(new URL('./sim/sim.worker.ts', import.meta.url), { type: 'module' })),
  createRenderer: (canvas, scene) => SceneRenderer.create(canvas, scene, GREYBOX_THEME),
  scheduler: { requestFrame: (callback) => requestAnimationFrame(callback), cancelFrame: (handle) => cancelAnimationFrame(handle) },
  downloadJson,
}
```
(imports: `SceneRenderer` from `@/render/scene-renderer`, `downloadJson` from `@/io/download-json`.)

- [ ] **Step 6: Run the tests to verify they pass**

Run: `pnpm vitest run --project web`
Expected: PASS.

- [ ] **Step 7: Verify, run the app once by hand, and commit**

```bash
pnpm lint && pnpm typecheck && pnpm test && pnpm --filter @physics-lab/web build
pnpm --filter @physics-lab/web preview   # open http://localhost:4173, complete the flow once with the keyboard only
git add apps/web
git commit -m "feat(web): add world screen with 3d view, camera controls and console"
```
Report what was checked by hand (keyboard-only completion, screen reader announcements if available).

---

### Task 20: `apps/web` — performance check screen (Chromebook benchmark)

**Files:**
- Create: `apps/web/src/bench/BenchScreen.tsx`
- Modify: `apps/web/src/home/HomeScreen.tsx` (add `onBenchmark` prop and a secondary button), `apps/web/src/app/App.tsx` (bench route), `i18n/messages/{en,es}.json`, `home/HomeScreen.test.tsx`
- Test: `apps/web/src/bench/BenchScreen.test.tsx`

**Interfaces:**
- Consumes: `SimClient.benchmark`, `BENCHMARK_BODIES` (Task 16).
- Produces: `BENCH_STEPS = 240`, `REAL_TIME_BUDGET_MS_PER_STEP = 1000 / 60 / 4` (4 steps per 60 Hz frame at 1×, spec §10.7), `BenchScreen(props: { dependencies: AppDependencies; onExit(): void })`.

Spec §13 mitigation: benchmark the deterministic Rapier build on the reference Chromebook during M1. The owner runs this screen on the device; the result is recorded in `docs/engineering/m1-exit-checklist.md` (Task 21).

- [ ] **Step 1: Messages**

`en.json`:
```json
{
  "home.benchmark": "Performance check",
  "bench.title": "Performance check",
  "bench.body": "Simulates {bodies} bodies for {steps} steps and measures the time per step on this device.",
  "bench.run": "Run the check",
  "bench.running": "Running…",
  "bench.result": "{ms} ms per step",
  "bench.pass": "Fast enough for real time (budget {budget} ms per step).",
  "bench.fail": "Slower than real time (budget {budget} ms per step).",
  "bench.back": "Back to home"
}
```
`es.json`:
```json
{
  "home.benchmark": "Prueba de rendimiento",
  "bench.title": "Prueba de rendimiento",
  "bench.body": "Simula {bodies} cuerpos durante {steps} pasos y mide el tiempo por paso en este dispositivo.",
  "bench.run": "Ejecutar la prueba",
  "bench.running": "Ejecutando…",
  "bench.result": "{ms} ms por paso",
  "bench.pass": "Suficientemente rápido para tiempo real (presupuesto {budget} ms por paso).",
  "bench.fail": "Más lento que el tiempo real (presupuesto {budget} ms por paso).",
  "bench.back": "Volver al inicio"
}
```

- [ ] **Step 2: Write the failing test**

`apps/web/src/bench/BenchScreen.test.tsx`:
```tsx
import { screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, expect, it, vi } from 'vitest'
import type { AppDependencies } from '@/app/dependencies'
import { fakeRendererFactory } from '@/test/fake-renderer'
import { FakeScheduler } from '@/test/fake-scheduler'
import { FakeSimClient } from '@/test/fake-sim-client'
import { renderWithIntl } from '@/test/render'
import { BenchScreen } from './BenchScreen'

describe('BenchScreen', () => {
  it('runs the benchmark through the sim client and reports the verdict in words', async () => {
    const client = new FakeSimClient()
    const dependencies: AppDependencies = {
      appVersion: 'test', randomSeed: () => 1, createSimClient: () => client, createRenderer: fakeRendererFactory().factory,
      scheduler: new FakeScheduler(), downloadJson: vi.fn(),
    }
    renderWithIntl(<BenchScreen dependencies={dependencies} onExit={vi.fn()} />)
    await userEvent.click(screen.getByRole('button', { name: 'Run the check' }))
    expect(await screen.findByText('0.80 ms per step')).toBeInTheDocument()
    expect(screen.getByText(/Fast enough for real time/)).toBeInTheDocument()
    expect(client.calls).toContain('benchmark:300:240')
  })
})
```
Add to `HomeScreen.test.tsx`:
```tsx
  it('opens the performance check', async () => {
    const onBenchmark = vi.fn()
    renderWithIntl(<HomeScreen locale="en" onLocaleChange={vi.fn()} onStart={vi.fn()} onBenchmark={onBenchmark} randomSeed={() => 1} />)
    await userEvent.click(screen.getByRole('button', { name: 'Performance check' }))
    expect(onBenchmark).toHaveBeenCalled()
  })
```
(and pass `onBenchmark={vi.fn()}` in the existing `setup()`).

- [ ] **Step 3: Run the test to verify it fails**

Run: `pnpm vitest run --project web Bench HomeScreen`
Expected: FAIL.

- [ ] **Step 4: Implement**

`apps/web/src/bench/BenchScreen.tsx`:
```tsx
import { useEffect, useState } from 'react'
import { useIntl } from 'react-intl'
import type { AppDependencies } from '@/app/dependencies'
import { Button } from '@/components/ui/button'
import type { SimClient } from '@/sim/sim-worker-client'
import { BENCHMARK_BODIES } from './benchmark-world'

export const BENCH_STEPS = 240
/** Real time at 1×: four 1/240 s steps per 60 Hz frame (spec §10.7). */
export const REAL_TIME_BUDGET_MS_PER_STEP = 1000 / 60 / 4
const RESULT_DECIMALS = 2

export function BenchScreen({ dependencies, onExit }: { dependencies: AppDependencies; onExit(): void }) {
  const intl = useIntl()
  const [client, setClient] = useState<SimClient | null>(null)
  const [running, setRunning] = useState(false)
  const [msPerStep, setMsPerStep] = useState<number | null>(null)

  useEffect(() => {
    const created = dependencies.createSimClient()
    setClient(created)
    return () => created.dispose()
  }, [dependencies])

  async function run(): Promise<void> {
    if (!client) return
    setRunning(true)
    setMsPerStep(await client.benchmark(BENCHMARK_BODIES, BENCH_STEPS))
    setRunning(false)
  }

  const format = (value: number): string => intl.formatNumber(value, { minimumFractionDigits: RESULT_DECIMALS, maximumFractionDigits: RESULT_DECIMALS })
  const budget = format(REAL_TIME_BUDGET_MS_PER_STEP)

  return (
    <main className="mx-auto flex w-full max-w-prose flex-col gap-4 px-4 py-8">
      <h1 className="text-2xl font-bold">{intl.formatMessage({ id: 'bench.title' })}</h1>
      <p>{intl.formatMessage({ id: 'bench.body' }, { bodies: BENCHMARK_BODIES, steps: BENCH_STEPS })}</p>
      <Button type="button" className="min-h-11 self-start" disabled={!client || running} onClick={() => void run()}>
        {intl.formatMessage({ id: running ? 'bench.running' : 'bench.run' })}
      </Button>
      <div role="status" aria-live="polite">
        {msPerStep !== null && (
          <>
            <p className="text-xl font-semibold tabular-nums">{intl.formatMessage({ id: 'bench.result' }, { ms: format(msPerStep) })}</p>
            <p>{intl.formatMessage({ id: msPerStep <= REAL_TIME_BUDGET_MS_PER_STEP ? 'bench.pass' : 'bench.fail' }, { budget })}</p>
          </>
        )}
      </div>
      <Button type="button" variant="secondary" className="min-h-11 self-start" onClick={onExit}>{intl.formatMessage({ id: 'bench.back' })}</Button>
    </main>
  )
}
```

`HomeScreen.tsx`: add `onBenchmark(): void` to the props and, after the form:
```tsx
      <Button type="button" variant="secondary" className="min-h-11 self-start" onClick={onBenchmark}>{intl.formatMessage({ id: 'home.benchmark' })}</Button>
```

`App.tsx`: replace the `started` state with a screen union and route the bench:
```tsx
type Screen = { kind: 'home' } | { kind: 'world'; request: StartRequest } | { kind: 'bench' }
// ...
const [screen, setScreen] = useState<Screen>({ kind: 'home' })
const home = () => setScreen({ kind: 'home' })
// render:
{screen.kind === 'world' && <WorldScreen dependencies={dependencies} locale={locale} studentId={screen.request.studentId} seed={screen.request.seed} onExit={home} />}
{screen.kind === 'bench' && <BenchScreen dependencies={dependencies} onExit={home} />}
{screen.kind === 'home' && <HomeScreen locale={locale} onLocaleChange={setLocale} onStart={(request) => setScreen({ kind: 'world', request })} onBenchmark={() => setScreen({ kind: 'bench' })} randomSeed={dependencies.randomSeed} />}
```

- [ ] **Step 5: Run the tests to verify they pass, verify and commit**

```bash
pnpm vitest run --project web
pnpm lint && pnpm typecheck && pnpm test && pnpm --filter @physics-lab/web build
git add apps/web
git commit -m "feat(web): add performance check screen for school hardware"
```

---
### Task 21: Milestone gates — e2e flows, cross-browser determinism, bundle budget, exit checklist

**Files:**
- Create: `apps/web/playwright.config.ts`, `apps/web/e2e/flows/catapult.spec.ts`, `apps/web/e2e/flows/home-responsive.spec.ts`
- Create: `packages/sim-core/vitest.browser.config.ts`, `packages/sim-core/src/testing/determinism-run.ts`, `packages/sim-core/src/determinism/determinism.browser.test.ts`, `packages/sim-core/src/determinism/raw-imports.d.ts`
- Create: `scripts/check-bundle-size.mjs`, `.github/workflows/e2e.yml`, `docs/engineering/m1-exit-checklist.md`
- Modify: `packages/sim-core/src/determinism/determinism.test.ts` (use `determinism-run.ts`), `package.json` (`check:bundle` script), `.github/workflows/ci.yml` (build + bundle steps), `.gitignore` (`apps/web/e2e/screens/`)

**Interfaces:**
- Consumes: everything above.
- Produces: `catapultDeterminismHash(factory: PhysicsEngineFactory, releaseAngleDeg?: number): string` and `DETERMINISM_STEPS = 600` (sim-core testing); root script `check:bundle`.

This task's checks are the M1 exit criteria (spec §12): a student completes the catapult challenge in a browser; the catapult benchmark passes (Tasks 9 and 12, in CI); the exported result JSON verifies by replay. **This is the one task where Playwright and Vitest browser mode are run** (owner directive: milestone end).

- [ ] **Step 1: Shared determinism run and the browser test**

`packages/sim-core/src/testing/determinism-run.ts`:
```ts
import type { PhysicsEngineFactory } from '../ports/physics-engine'
import { Simulation } from '../simulation/simulation'
import { catapultTestWorld } from './catapult-test-world'

export const DETERMINISM_STEPS = 600

/** The reference run whose hash must match in Node, Chromium, Firefox and WebKit (spec §11, D16). */
export function catapultDeterminismHash(factory: PhysicsEngineFactory, releaseAngleDeg = 90): string {
  const sim = Simulation.create(catapultTestWorld(releaseAngleDeg), factory)
  sim.trigger('pivot', 'startMotor')
  for (let i = 0; i < DETERMINISM_STEPS; i++) sim.step()
  const hash = sim.stateHash()
  sim.dispose()
  return hash
}
```
Refactor `determinism.test.ts` to call `catapultDeterminismHash(factory, angle)` instead of its local `hashAfterRun` (same assertions, same golden file) and export it from `src/index.ts`.

`packages/sim-core/src/determinism/raw-imports.d.ts`:
```ts
declare module '*.hash?raw' {
  const content: string
  export default content
}
```

`packages/sim-core/src/determinism/determinism.browser.test.ts`:
```ts
import { expect, it } from 'vitest'
import { createRapierEngineFactory } from '../adapters/rapier'
import { catapultDeterminismHash } from '../testing/determinism-run'
import golden from './__golden__/catapult-600-steps.hash?raw'

it('reproduces the Node golden hash in this browser engine (D16)', async () => {
  expect(catapultDeterminismHash(await createRapierEngineFactory())).toBe(golden.trim())
})
```

`packages/sim-core/vitest.browser.config.ts`:
```ts
import { playwright } from '@vitest/browser-playwright'
import { defineProject } from 'vitest/config'

export default defineProject({
  test: {
    name: 'sim-core-browser',
    include: ['src/**/*.browser.test.ts'],
    browser: {
      enabled: true,
      headless: true,
      provider: playwright(),
      instances: [{ browser: 'chromium' }, { browser: 'firefox' }, { browser: 'webkit' }],
    },
  },
})
```
Exclude `*.browser.test.ts` from the Node project: in `packages/sim-core/vitest.config.ts` set `exclude: ['src/**/*.browser.test.ts']`.

```bash
pnpm add -D -w @vitest/browser-playwright@^5.0.3 playwright
pnpm exec playwright install --with-deps chromium firefox webkit
```

- [ ] **Step 2: e2e specs**

`apps/web/playwright.config.ts`:
```ts
import { defineConfig, devices } from '@playwright/test'

const PREVIEW_URL = 'http://localhost:4173'

export default defineConfig({
  testDir: './e2e',
  timeout: 120_000,
  retries: process.env['CI'] ? 1 : 0,
  use: { baseURL: PREVIEW_URL, trace: 'retain-on-failure' },
  webServer: { command: 'pnpm build && pnpm preview', url: PREVIEW_URL, reuseExistingServer: !process.env['CI'], timeout: 180_000 },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
  ],
})
```

`apps/web/e2e/flows/catapult.spec.ts`:
```ts
import { readFile } from 'node:fs/promises'
import AxeBuilder from '@axe-core/playwright'
import { loadBuiltInChallenge, projectileRangeSolver, verifyResult } from '@physics-lab/challenges'
import { ResultSchema } from '@physics-lab/formats'
import { createRapierEngineFactory } from '@physics-lab/sim-core/rapier'
import { expect, test, type Page } from '@playwright/test'

const WCAG_TAGS = ['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'wcag22aa']

async function collectCspViolations(page: Page): Promise<string[]> {
  const violations: string[] = []
  await page.exposeFunction('reportCspViolation', (detail: string) => violations.push(detail))
  await page.addInitScript(() => {
    document.addEventListener('securitypolicyviolation', (event) => {
      void (window as unknown as { reportCspViolation(detail: string): Promise<void> }).reportCspViolation(`${event.violatedDirective} ${event.blockedURI}`)
    })
  })
  return violations
}

async function measuredValue(page: Page, quantityLabel: string): Promise<number> {
  const row = page.getByRole('table', { name: 'Measurements' }).getByRole('row').filter({ hasText: quantityLabel }).last()
  const text = await row.getByRole('cell').nth(1).innerText()
  return Number(text.split(' ± ')[0])
}

test('a student completes the catapult challenge and the result verifies by replay (M1 exit)', async ({ page }) => {
  const cspViolations = await collectCspViolations(page)
  await page.goto('/')
  await page.getByLabel('Language').selectOption('en')
  await page.getByLabel('Student name or ID').fill('e2e-student')
  await page.getByLabel('Seed (optional)').fill('12345')
  await page.getByRole('button', { name: 'Start the catapult challenge' }).click()
  await expect(page.getByRole('application', { name: '3D view of the catapult experiment' })).toBeVisible()

  await page.getByRole('button', { name: 'Launch test' }).click()
  await expect(page.getByText('Paused right after release', { exact: true })).toBeVisible({ timeout: 30_000 })
  for (const quantity of ['Release speed', 'Release angle', 'Release height above the ground']) {
    await page.getByRole('button', { name: `Measure: ${quantity}` }).click()
  }
  const speed = await measuredValue(page, 'Release speed')
  const angle = await measuredValue(page, 'Release angle')
  const height = await measuredValue(page, 'Release height above the ground')
  const prediction = projectileRangeSolver.solve({ speed, angle, height }, 9.8).range ?? 0
  await page.getByLabel('Your prediction (m)').fill(prediction.toFixed(2))
  await page.getByRole('button', { name: 'Save prediction' }).click()

  await page.getByRole('button', { name: 'Continue the launch' }).click()
  await expect(page.getByText('The ball has landed', { exact: true })).toBeVisible({ timeout: 30_000 })
  await page.getByRole('button', { name: 'Measure: Horizontal range' }).click()
  await page.getByRole('button', { name: 'Compare prediction and measurement' }).click()
  await expect(page.getByText('Within the expected uncertainty')).toBeVisible()
  await page.getByLabel(/Explain any difference/).fill('The model ignores air drag; the difference is within the uncertainty.')
  await page.getByRole('button', { name: 'Save explanation' }).click()

  const accessibility = await new AxeBuilder({ page }).withTags(WCAG_TAGS).analyze()
  expect(accessibility.violations.map((violation) => violation.id)).toEqual([])

  const [download] = await Promise.all([page.waitForEvent('download'), page.getByRole('button', { name: 'Download result file' }).click()])
  const result = ResultSchema.parse(JSON.parse(await readFile(await download.path(), 'utf8')))
  const report = verifyResult(loadBuiltInChallenge('catapult-range'), result, await createRapierEngineFactory())
  expect(report).toEqual({ status: 'verified', differences: [] })
  expect(cspViolations).toEqual([])
})

test('the whole challenge is operable with the keyboard alone', async ({ page }) => {
  await page.goto('/')
  await page.getByLabel('Student name or ID').fill('keyboard')
  await page.keyboard.press('Enter')
  await expect(page.getByRole('application')).toBeVisible()
  await page.getByRole('button', { name: 'Launch test' }).focus()
  await page.keyboard.press('Enter')
  await expect(page.getByText('Paused right after release', { exact: true })).toBeVisible({ timeout: 30_000 })
  await page.keyboard.press('Escape')
  await page.keyboard.press('/')
  await expect(page.getByLabel('Command')).toBeFocused()
  await page.keyboard.type('/measure flight.releaseSpeed')
  await page.keyboard.press('Enter')
  await expect(page.getByRole('log')).toContainText('Release speed:')
  await page.keyboard.press('Escape')
  await page.getByRole('application').focus()
  await page.keyboard.press('ArrowLeft')
})
```
> If Playwright cannot import the workspace TypeScript packages (it transpiles files outside `node_modules`; workspace packages resolve to `packages/*/src`), fall back to writing the downloaded JSON to `test-results/` and verifying it with a Vitest test in `packages/challenges` that reads `E2E_RESULT_PATH`. Report which path was used.

`apps/web/e2e/flows/home-responsive.spec.ts`:
```ts
import AxeBuilder from '@axe-core/playwright'
import { expect, test } from '@playwright/test'

/** D32 / owner rule: 320, 768, 1280 and 2560 px, portrait and landscape; no horizontal scroll at 320 px (WCAG 1.4.10). */
const VIEWPORTS = [
  { width: 320, height: 640 }, { width: 640, height: 320 },
  { width: 768, height: 1024 }, { width: 1024, height: 768 },
  { width: 1280, height: 800 }, { width: 2560, height: 1440 },
]

for (const viewport of VIEWPORTS) {
  test(`home reflows at ${viewport.width}×${viewport.height}`, async ({ page }, testInfo) => {
    await page.setViewportSize(viewport)
    await page.goto('/')
    const overflow = await page.evaluate(() => document.documentElement.scrollWidth - window.innerWidth)
    expect(overflow).toBeLessThanOrEqual(0)
    await page.screenshot({ path: `e2e/screens/${testInfo.project.name}-home-${viewport.width}x${viewport.height}.png`, fullPage: true })
    const accessibility = await new AxeBuilder({ page }).withTags(['wcag2a', 'wcag2aa', 'wcag21aa', 'wcag22aa']).analyze()
    expect(accessibility.violations.map((violation) => violation.id)).toEqual([])
  })
}

test('small screens get the desktop notice instead of the 3D world', async ({ page }) => {
  await page.setViewportSize({ width: 320, height: 640 })
  await page.goto('/')
  await page.getByLabel('Student name or ID').fill('phone')
  await page.getByRole('button', { name: 'Start the catapult challenge' }).click()
  await expect(page.getByRole('heading', { name: 'This lab needs a larger screen' })).toBeVisible()
  await page.screenshot({ path: 'e2e/screens/world-notice-320.png', fullPage: true })
})
```
Add `apps/web/e2e/screens/` to `.gitignore`.

- [ ] **Step 3: Bundle budget and CI**

`scripts/check-bundle-size.mjs`:
```js
import { readFile, readdir } from 'node:fs/promises'
import { join } from 'node:path'
import { fileURLToPath } from 'node:url'
import { gzipSync } from 'node:zlib'

/** D19: initial download ≤ 15 MB compressed. */
const BUDGET_BYTES = 15 * 1024 * 1024
const BYTES_PER_MB = 1024 * 1024
const DIST_DIR = fileURLToPath(new URL('../apps/web/dist/', import.meta.url))

async function* walk(directory) {
  for (const entry of await readdir(directory, { withFileTypes: true })) {
    const path = join(directory, entry.name)
    if (entry.isDirectory()) yield* walk(path)
    else yield path
  }
}

let totalBytes = 0
for await (const file of walk(DIST_DIR)) totalBytes += gzipSync(await readFile(file)).length
console.log(`Compressed download: ${(totalBytes / BYTES_PER_MB).toFixed(2)} MB (budget ${BUDGET_BYTES / BYTES_PER_MB} MB)`)
if (totalBytes > BUDGET_BYTES) {
  console.error('Bundle budget exceeded (D19)')
  process.exit(1)
}
```
Root `package.json` scripts: `"check:bundle": "node scripts/check-bundle-size.mjs"`.

Append to `.github/workflows/ci.yml` steps:
```yaml
      - run: pnpm --filter @physics-lab/web build
      - run: pnpm check:bundle
```

`.github/workflows/e2e.yml`:
```yaml
name: E2E
on:
  workflow_dispatch: # milestone end or owner request only (D39)

permissions:
  contents: read

jobs:
  browsers:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec playwright install --with-deps chromium firefox webkit
      - run: pnpm test:browser
      - run: pnpm test:e2e
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: apps/web/playwright-report/
          retention-days: 7
```

- [ ] **Step 4: Exit checklist**

`docs/engineering/m1-exit-checklist.md`:
```markdown
# M1 exit checklist — catapult vertical slice

Spec §12: a student can complete the catapult challenge in a browser; the catapult benchmark passes;
replay of the result JSON verifies. Fill every result with the date and the evidence (command output,
file, screenshot path).

## Automated
| Check | Command | Result |
|-------|---------|--------|
| Lint, format, typecheck, unit + validation tests | `pnpm lint && pnpm format:check && pnpm typecheck && pnpm test` | |
| Engine benchmark (free projectile, 9 cases ≤ 0.5 %) | `pnpm test:validation` | |
| Catapult sweep (31 release angles ≤ 0.5 %) | `pnpm test:validation` | |
| Cross-browser determinism (Chromium, Firefox, WebKit = Node golden) | `pnpm test:browser` | |
| E2E: complete flow + replay verification, 3 browsers | `pnpm test:e2e` | |
| E2E: keyboard-only flow | `pnpm test:e2e` | |
| Home reflow at 320–2560 px, no horizontal scroll | `pnpm test:e2e` | |
| No CSP violations, no `eval` in bundle | `pnpm test:e2e`; `grep -rlE '\beval\(' apps/web/dist/assets` | |
| Bundle ≤ 15 MB compressed | `pnpm check:bundle` | |

## Manual
| Check | How | Result |
|-------|-----|--------|
| Screenshots reviewed at 320, 768, 1280, 2560 px (portrait and landscape) | `apps/web/e2e/screens/` | |
| Screen reader: NVDA (Windows) or VoiceOver (macOS) announces release, pause, landing and measurements | Run the flow once | |
| Focus is always visible and never hidden by the console or camera controls (2.4.11) | Keyboard walk-through | |
| Reference Chromebook: performance check (300 bodies) — device model, ms/step, pass/fail | Home → Performance check | |
| `physics-reviewer`, `code-reviewer`, `a11y-reviewer` whole-branch reviews | Agents | |
| Decisions log updated for every tolerance, geometry or scope change made during M1 | `docs/design/decisions-log.md` | |
```

- [ ] **Step 5: Run the milestone gates**

```bash
pnpm lint && pnpm format:check && pnpm typecheck && pnpm test
pnpm --filter @physics-lab/web build && pnpm check:bundle
pnpm test:browser
pnpm test:e2e
```
Expected: all pass. A browser hash differing from the Node golden is a **D16 failure**: stop and escalate to `physics-reviewer` (do not regenerate the golden). Fill the automated rows of the checklist with the outputs.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "test(web): add milestone e2e flows, cross-browser determinism and bundle budget"
```
(If the reviewer prefers smaller commits: `test(sim-core): …`, `test(web): …`, `ci: …`, `docs(engineering): …`.)

- [ ] **Step 7: Whole-branch review and finish**

Dispatch in parallel (Opus): `code-reviewer` (whole diff vs this plan), `physics-reviewer` (sim-core, instruments, challenges, validation numbers), `a11y-reviewer` (apps/web). Fix findings through the normal task loop, then use superpowers:finishing-a-development-branch: mark the PR ready and merge it once `verify` passes (D46). Report to the owner: which e2e specs ran and on which browsers, the Chromebook number (or that it is pending on the device), and the open manual checklist rows.

---

## Self-review notes (plan author)

- **Spec coverage (M1 row of spec §12):** voxel ground (Tasks 5, 8), basic parts (beam, sphere — Task 3/8), hinge joint with motor (Tasks 7–8), CCD (Tasks 7, 9), sensors (Task 9), instruments with noise (Task 10), basic console (Tasks 14, 19), predict/compare (Tasks 11, 13, 18), result JSON (Tasks 4, 13), replay verification (Task 13, 21), catapult benchmark (Tasks 9, 12), greybox theme layer (Tasks 15, 17), i18n es/en (Tasks 15, 18–20), WCAG 2.2 AA paths (Tasks 15, 17–19, 21), responsive Home + desktop notice (Tasks 15, 19, 21), D31 enforcement (Task 1), Chromebook benchmark (Task 20). Deferred by design to later milestones: walk mode, full builder, undo/redo, hinge limits (D44), dynamic blocks, per-student SHA-256 seeds and teacher view (M4), PDF/CSV export (M4), PWA/offline (M5).
- **Types cross-checked:** `QuantityId` = `"sensor.reading"` everywhere; `CommandLogEntry.atStep` is run-local; `LabPhase` values; `TRANSFORM_STRIDE` = 7; `SimClient` methods match `FakeSimClient`; `LabErrorCode` list matches `error.*` keys through `requiredMessageKeys()`.
- **Known risks handed to reviewers:** integrator bias vs the 0.5 % range tolerance (Tasks 9, 12 — sweep limited to v·sinθ ≥ 5 m/s with the reason documented); photogate speed uncertainty is large by design of D34 values (D42 keeps the pass rule fair); Playwright importing workspace TS (fallback in Task 21).
