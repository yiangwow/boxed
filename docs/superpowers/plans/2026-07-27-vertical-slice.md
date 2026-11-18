# Boxed Vertical Slice Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A playable browser slice of Boxed: one chasm-crossing scenario where the player specs a bridge from clause chips, the Instantiator returns a random conforming (usually wrong) bridge, the sim plays out deterministically, failure shows the all-green receipt, and betrayals unlock new predicate chips.

**Architecture:** Pure deterministic core (`src/core/`: rng, model, predicates, instantiator, sim, scenario) with no DOM/canvas imports, fully unit-tested with Vitest. Thin UI layer (`src/ui/`: Canvas2D renderer, DOM spec panel, receipt overlay, main shell) wired to the core. No engine, no runtime dependencies.

**Tech Stack:** TypeScript (strict), Vite, Vitest, Canvas2D, plain DOM.

## Global Constraints

- Zero runtime dependencies — devDependencies only (`typescript`, `vite`, `vitest`).
- `src/core/` files must never import from `src/ui/`, use `Math.random`, `Date`, or any browser API. All randomness flows through the seeded RNG.
- **Fairness invariant:** every model returned by the Instantiator satisfies every clause of the spec that summoned it — enforced by test in Task 4; never weaken that test.
- The receipt never explains the failure. Clauses + green checkmarks only. No hint text anywhere.
- Git: work on branch `feat/vertical-slice` (created in Task 1). Never commit to `main`. Authorship is repo-configured (`yiangwow`); no `Co-Authored-By` lines in commit messages.
- Run commands from repo root `~/boxed`.

## Game constants (used throughout — single source of truth is `src/core/model.ts`)

Grid 16×9. Support row 5. Ground cols 0–5 and 10–15; chasm cols 6–9. Pawn starts col 1, goal col 14. Bridge = set of support-row columns. Drift = bridge shifts −1 col every 3 ticks when not anchored. Sway = bridge vacates the support row on odd ticks when not rigid. Collapse = cells the pawn has left vanish when not sturdy (drama only — it can never kill the pawn, who only moves forward). Clause budget 3, max 5 attempts, starting vocabulary `SPANS_GAP` + `BEARS_WEIGHT`, unlocks: drift-death → `ANCHORED`, sway-death → `RIGID`.

Designed outcome check (verify in Task 5 tests): a fully-good bridge crosses; `{SPANS_GAP, ANCHORED, RIGID}` always wins regardless of `sturdy` (fragile cells collapse *behind* the pawn); any spec omitting `ANCHORED` or `RIGID` can fail with the matching cause, which unlocks the matching chip.

---

### Task 1: Project scaffold

**Files:**
- Create: `package.json`, `tsconfig.json`, `vite.config.ts`, `index.html`, `src/style.css`, `src/ui/main.ts`, `tests/smoke.test.ts`, `.gitignore`
- Modify: `CLAUDE.md` (Commands section)

**Interfaces:**
- Consumes: nothing
- Produces: working `npm run dev` / `npm test` / `npm run build` toolchain; `index.html` DOM skeleton with `#game` canvas (768×432), `#spec-panel`, `#overlay` divs that later tasks target by id.

- [ ] **Step 1: Create branch**

```bash
git checkout -b feat/vertical-slice
```

- [ ] **Step 2: Write config files**

`package.json`:
```json
{
  "name": "boxed",
  "private": true,
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc --noEmit && vite build",
    "preview": "vite preview",
    "test": "vitest run"
  },
  "devDependencies": {
    "typescript": "^5.5.0",
    "vite": "^5.4.0",
    "vitest": "^2.0.0"
  }
}
```

`tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noEmit": true,
    "lib": ["ES2022", "DOM", "DOM.Iterable"]
  },
  "include": ["src", "tests"]
}
```

`vite.config.ts`:
```ts
/// <reference types="vitest" />
import { defineConfig } from 'vite';

export default defineConfig({
  test: { include: ['tests/**/*.test.ts'] },
});
```

`.gitignore`:
```
node_modules/
dist/
```

- [ ] **Step 3: Write DOM skeleton**

`index.html`:
```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Boxed</title>
    <link rel="stylesheet" href="/src/style.css" />
  </head>
  <body>
    <main id="layout">
      <canvas id="game" width="768" height="432"></canvas>
      <aside id="spec-panel"></aside>
    </main>
    <div id="overlay" hidden></div>
    <script type="module" src="/src/ui/main.ts"></script>
  </body>
</html>
```

`src/style.css`:
```css
:root {
  --bg: #0a0a0f;
  --panel: #121218;
  --ink: #cfcabe;
  --dim: #6b675e;
  --ok: #6fae6f;
}
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
  background: var(--bg);
  color: var(--ink);
  font-family: ui-monospace, "Cascadia Mono", monospace;
  min-height: 100vh;
  display: grid;
  place-items: center;
}
#layout { display: flex; gap: 16px; align-items: stretch; }
#game { background: var(--bg); outline: 1px solid #1c1c24; }
#spec-panel {
  width: 260px;
  background: var(--panel);
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.chip {
  border: 1px solid var(--dim);
  background: none;
  color: var(--ink);
  font: inherit;
  padding: 8px 10px;
  text-align: left;
  cursor: pointer;
}
.chip.selected { border-color: var(--ink); background: #1c1c24; }
.chip.new { animation: pulse 1.2s ease-out 2; }
@keyframes pulse {
  0% { border-color: var(--ink); box-shadow: 0 0 12px #cfcabe44; }
  100% { border-color: var(--dim); box-shadow: none; }
}
#summon {
  margin-top: auto;
  border: 1px solid var(--ink);
  background: none;
  color: var(--ink);
  font: inherit;
  padding: 10px;
  cursor: pointer;
}
#summon:disabled { border-color: var(--dim); color: var(--dim); cursor: default; }
.meta { color: var(--dim); font-size: 12px; }
#overlay {
  position: fixed;
  inset: 0;
  background: #0a0a0fdd;
  display: grid;
  place-items: center;
}
#overlay[hidden] { display: none; }
.receipt {
  background: var(--panel);
  border: 1px solid var(--dim);
  padding: 24px;
  min-width: 320px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.receipt .row { display: flex; justify-content: space-between; gap: 24px; }
.receipt .check { color: var(--ok); }
```

`src/ui/main.ts` (placeholder for now, replaced in Task 8):
```ts
console.log('boxed');
```

- [ ] **Step 4: Write toolchain smoke test**

`tests/smoke.test.ts`:
```ts
import { describe, it, expect } from 'vitest';

describe('toolchain', () => {
  it('runs tests', () => {
    expect(1 + 1).toBe(2);
  });
});
```

- [ ] **Step 5: Install and verify**

Run: `npm install`
Run: `npm test` — Expected: 1 passed.
Run: `npm run build` — Expected: builds with no type errors.

- [ ] **Step 6: Update CLAUDE.md Commands section**

Replace the `## Commands` section body ("None yet — scaffold pending (Vite + Vitest expected).") with:
```markdown
- `npm run dev` — dev server
- `npm test` — unit tests (Vitest)
- `npm run build` — typecheck + production bundle
```
Also replace the `## Status` body with: `Vertical slice in progress on feat/vertical-slice. Plan: docs/superpowers/plans/2026-07-27-vertical-slice.md`

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "chore: scaffold Vite + TypeScript + Vitest toolchain"
```

---

### Task 2: Seeded RNG

**Files:**
- Create: `src/core/rng.ts`
- Test: `tests/rng.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `makeRng(seed: number): () => number` — deterministic PRNG, each call returns a float in [0, 1).

- [ ] **Step 1: Write the failing test**

`tests/rng.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { makeRng } from '../src/core/rng';

describe('makeRng', () => {
  it('same seed produces same sequence', () => {
    const a = makeRng(42);
    const b = makeRng(42);
    for (let i = 0; i < 100; i++) expect(a()).toBe(b());
  });

  it('different seeds produce different sequences', () => {
    const a = makeRng(1);
    const b = makeRng(2);
    const seqA = Array.from({ length: 10 }, () => a());
    const seqB = Array.from({ length: 10 }, () => b());
    expect(seqA).not.toEqual(seqB);
  });

  it('values are in [0, 1)', () => {
    const r = makeRng(7);
    for (let i = 0; i < 1000; i++) {
      const v = r();
      expect(v).toBeGreaterThanOrEqual(0);
      expect(v).toBeLessThan(1);
    }
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test`
Expected: FAIL — cannot resolve `../src/core/rng`.

- [ ] **Step 3: Write implementation (mulberry32)**

`src/core/rng.ts`:
```ts
export function makeRng(seed: number): () => number {
  let a = seed >>> 0;
  return () => {
    a = (a + 0x6d2b79f5) >>> 0;
    let t = a;
    t = Math.imul(t ^ (t >>> 15), t | 1);
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test` — Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git add src/core/rng.ts tests/rng.test.ts
git commit -m "feat: seeded PRNG (mulberry32)"
```

---

### Task 3: Model constants, types, and predicates

**Files:**
- Create: `src/core/model.ts`, `src/core/predicates.ts`
- Test: `tests/predicates.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `model.ts`: constants `GRID_COLS = 16`, `GRID_ROWS = 9`, `SUPPORT_ROW = 5`, `WALK_ROW = 4`, `GROUND_LEFT = [0, 5]`, `GROUND_RIGHT = [10, 15]`, `CHASM = [6, 9]`, `START_COL = 1`, `GOAL_COL = 14` (all `as const` tuples where ranges); types `PredicateId = 'SPANS_GAP' | 'BEARS_WEIGHT' | 'ANCHORED' | 'RIGID'`, `Spec = PredicateId[]`, `interface BridgeModel { cols: number[]; anchored: boolean; rigid: boolean; sturdy: boolean }`; helper `isGround(col: number): boolean`; display map `PREDICATE_LABEL: Record<PredicateId, string>`.
  - `predicates.ts`: `satisfies(model: BridgeModel, predicate: PredicateId): boolean`, `satisfiesSpec(model: BridgeModel, spec: Spec): boolean`.

- [ ] **Step 1: Write the failing test**

`tests/predicates.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { satisfies, satisfiesSpec } from '../src/core/predicates';
import type { BridgeModel } from '../src/core/model';

const base: BridgeModel = { cols: [6, 7, 8, 9], anchored: true, rigid: true, sturdy: true };

describe('satisfies', () => {
  it('SPANS_GAP requires every chasm column covered', () => {
    expect(satisfies(base, 'SPANS_GAP')).toBe(true);
    expect(satisfies({ ...base, cols: [6, 7, 8] }, 'SPANS_GAP')).toBe(false);
    expect(satisfies({ ...base, cols: [5, 6, 7, 8, 9, 10] }, 'SPANS_GAP')).toBe(true);
    expect(satisfies({ ...base, cols: [] }, 'SPANS_GAP')).toBe(false);
  });

  it('BEARS_WEIGHT maps to sturdy', () => {
    expect(satisfies(base, 'BEARS_WEIGHT')).toBe(true);
    expect(satisfies({ ...base, sturdy: false }, 'BEARS_WEIGHT')).toBe(false);
  });

  it('ANCHORED maps to anchored', () => {
    expect(satisfies({ ...base, anchored: false }, 'ANCHORED')).toBe(false);
  });

  it('RIGID maps to rigid', () => {
    expect(satisfies({ ...base, rigid: false }, 'RIGID')).toBe(false);
  });
});

describe('satisfiesSpec', () => {
  it('empty spec is satisfied by anything', () => {
    expect(satisfiesSpec({ cols: [], anchored: false, rigid: false, sturdy: false }, [])).toBe(true);
  });

  it('requires every clause', () => {
    expect(satisfiesSpec(base, ['SPANS_GAP', 'RIGID'])).toBe(true);
    expect(satisfiesSpec({ ...base, rigid: false }, ['SPANS_GAP', 'RIGID'])).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test` — Expected: FAIL, modules not found.

- [ ] **Step 3: Write implementation**

`src/core/model.ts`:
```ts
export const GRID_COLS = 16;
export const GRID_ROWS = 9;
export const SUPPORT_ROW = 5;
export const WALK_ROW = 4;
export const GROUND_LEFT = [0, 5] as const;
export const GROUND_RIGHT = [10, 15] as const;
export const CHASM = [6, 9] as const;
export const START_COL = 1;
export const GOAL_COL = 14;

export type PredicateId = 'SPANS_GAP' | 'BEARS_WEIGHT' | 'ANCHORED' | 'RIGID';
export type Spec = PredicateId[];

export interface BridgeModel {
  cols: number[];
  anchored: boolean;
  rigid: boolean;
  sturdy: boolean;
}

export const PREDICATE_LABEL: Record<PredicateId, string> = {
  SPANS_GAP: 'IT SPANS THE GAP',
  BEARS_WEIGHT: 'IT BEARS MY WEIGHT',
  ANCHORED: 'IT IS ANCHORED',
  RIGID: 'IT IS RIGID',
};

export function isGround(col: number): boolean {
  return (
    (col >= GROUND_LEFT[0] && col <= GROUND_LEFT[1]) ||
    (col >= GROUND_RIGHT[0] && col <= GROUND_RIGHT[1])
  );
}
```

`src/core/predicates.ts`:
```ts
import { CHASM, type BridgeModel, type PredicateId, type Spec } from './model';

export function satisfies(model: BridgeModel, predicate: PredicateId): boolean {
  switch (predicate) {
    case 'SPANS_GAP': {
      for (let c = CHASM[0]; c <= CHASM[1]; c++) {
        if (!model.cols.includes(c)) return false;
      }
      return true;
    }
    case 'BEARS_WEIGHT':
      return model.sturdy;
    case 'ANCHORED':
      return model.anchored;
    case 'RIGID':
      return model.rigid;
  }
}

export function satisfiesSpec(model: BridgeModel, spec: Spec): boolean {
  return spec.every((p) => satisfies(model, p));
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test` — Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git add src/core/model.ts src/core/predicates.ts tests/predicates.test.ts
git commit -m "feat: world constants, bridge model, clause predicates"
```

---

### Task 4: Instantiator with fairness property test

**Files:**
- Create: `src/core/instantiator.ts`
- Test: `tests/instantiator.test.ts`

**Interfaces:**
- Consumes: `BridgeModel`, `Spec` from `model.ts`; `satisfiesSpec` from `predicates.ts`; `makeRng` from `rng.ts`
- Produces: `allCandidates(): BridgeModel[]` (full generative space) and `instantiate(spec: Spec, rand: () => number): BridgeModel` (throws `Error('unsatisfiable spec')` if no candidate conforms — unreachable with the slice vocabulary, but explicit).

- [ ] **Step 1: Write the failing test**

`tests/instantiator.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { allCandidates, instantiate } from '../src/core/instantiator';
import { satisfiesSpec } from '../src/core/predicates';
import { makeRng } from '../src/core/rng';
import type { PredicateId, Spec } from '../src/core/model';

const ALL: PredicateId[] = ['SPANS_GAP', 'BEARS_WEIGHT', 'ANCHORED', 'RIGID'];

function subsets(ids: PredicateId[]): Spec[] {
  return ids.reduce<Spec[]>((acc, id) => acc.concat(acc.map((s) => [...s, id])), [[]]);
}

describe('allCandidates', () => {
  it('contains every contiguous shape in cols 5..10 crossed with all flag combos', () => {
    // 21 contiguous ranges within [5,10] × 2×2×2 flags = 168
    expect(allCandidates()).toHaveLength(168);
  });

  it('contains a model satisfying all four predicates at once', () => {
    expect(allCandidates().some((m) => satisfiesSpec(m, ALL))).toBe(true);
  });
});

describe('instantiate — fairness invariant', () => {
  it('every pick satisfies every clause of its spec, for all 16 specs × 50 seeds', () => {
    for (const spec of subsets(ALL)) {
      for (let seed = 0; seed < 50; seed++) {
        const model = instantiate(spec, makeRng(seed));
        expect(satisfiesSpec(model, spec)).toBe(true);
      }
    }
  });

  it('is deterministic for a given seed', () => {
    const spec: Spec = ['SPANS_GAP'];
    expect(instantiate(spec, makeRng(9))).toEqual(instantiate(spec, makeRng(9)));
  });

  it('empty spec can produce non-spanning bridges (underdetermination is real)', () => {
    const picks = Array.from({ length: 100 }, (_, i) => instantiate([], makeRng(i)));
    expect(picks.some((m) => !satisfiesSpec(m, ['SPANS_GAP']))).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test` — Expected: FAIL, module not found.

- [ ] **Step 3: Write implementation**

`src/core/instantiator.ts`:
```ts
import { satisfiesSpec } from './predicates';
import type { BridgeModel, Spec } from './model';

const SHAPE_MIN = 5;
const SHAPE_MAX = 10;

export function allCandidates(): BridgeModel[] {
  const models: BridgeModel[] = [];
  for (let start = SHAPE_MIN; start <= SHAPE_MAX; start++) {
    for (let end = start; end <= SHAPE_MAX; end++) {
      const cols: number[] = [];
      for (let c = start; c <= end; c++) cols.push(c);
      for (const anchored of [false, true]) {
        for (const rigid of [false, true]) {
          for (const sturdy of [false, true]) {
            models.push({ cols, anchored, rigid, sturdy });
          }
        }
      }
    }
  }
  return models;
}

export function instantiate(spec: Spec, rand: () => number): BridgeModel {
  const conforming = allCandidates().filter((m) => satisfiesSpec(m, spec));
  if (conforming.length === 0) throw new Error('unsatisfiable spec');
  const pick = conforming[Math.floor(rand() * conforming.length)];
  if (!pick) throw new Error('unsatisfiable spec');
  return pick;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test` — Expected: all pass. If the 168 count fails, recount: ranges within [5,10] = 6+5+4+3+2+1 = 21; 21 × 8 = 168 — fix the generator, not the test.

- [ ] **Step 5: Commit**

```bash
git add src/core/instantiator.ts tests/instantiator.test.ts
git commit -m "feat: instantiator with machine-checked fairness invariant"
```

---

### Task 5: Deterministic simulation

**Files:**
- Create: `src/core/sim.ts`
- Test: `tests/sim.test.ts`

**Interfaces:**
- Consumes: `BridgeModel`, `isGround`, `START_COL`, `GOAL_COL` from `model.ts`
- Produces:
  - `DRIFT_PERIOD = 3`, `MAX_TICKS = 60`
  - `type FailCause = 'NO_SUPPORT' | 'DRIFTED' | 'SWAYED' | 'COLLAPSED'`
  - `interface SimState { tick: number; pawnCol: number; bridgeCols: number[]; swayUp: boolean; collapsed: number[]; visited: number[]; status: 'RUNNING' | 'WIN' | 'FAIL'; failCause: FailCause | null }`
  - `interface SimResult { states: SimState[]; verdict: 'WIN' | 'FAIL'; failCause: FailCause | null }`
  - `initialSimState(model: BridgeModel): SimState`, `step(prev: SimState, model: BridgeModel): SimState`, `runSim(model: BridgeModel): SimResult`

Tick semantics (implement exactly): each tick — (1) drift: if `!anchored` and `tick % 3 === 0`, all bridge cols shift −1; (2) sway: `swayUp = !rigid && tick % 2 === 1`; swayed cells don't support; (3) collapse: if `!sturdy`, previously visited bridge cols the pawn is no longer on collapse; (4) pawn moves +1 col; (5) support check at new col: ground, or bridge col present ∧ not collapsed ∧ not swayUp. Fail cause priority: COLLAPSED > SWAYED > DRIFTED > NO_SUPPORT (DRIFTED = model was unanchored and its *original* cols covered the pawn's col).

- [ ] **Step 1: Write the failing test**

`tests/sim.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { runSim } from '../src/core/sim';
import type { BridgeModel } from '../src/core/model';

const good: BridgeModel = { cols: [6, 7, 8, 9], anchored: true, rigid: true, sturdy: true };

describe('runSim', () => {
  it('a fully good spanning bridge wins', () => {
    const r = runSim(good);
    expect(r.verdict).toBe('WIN');
    expect(r.failCause).toBeNull();
  });

  it('an unanchored bridge drifts out from under the pawn: DRIFTED', () => {
    const r = runSim({ ...good, anchored: false });
    expect(r.verdict).toBe('FAIL');
    expect(r.failCause).toBe('DRIFTED');
  });

  it('a non-rigid bridge sways at first chasm step: SWAYED', () => {
    const r = runSim({ ...good, rigid: false });
    expect(r.verdict).toBe('FAIL');
    expect(r.failCause).toBe('SWAYED');
  });

  it('a fragile bridge collapses BEHIND the pawn — still a win', () => {
    const r = runSim({ ...good, sturdy: false });
    expect(r.verdict).toBe('WIN');
    const last = r.states[r.states.length - 1]!;
    expect(last.collapsed.length).toBeGreaterThan(0);
  });

  it('a bridge that does not span fails with NO_SUPPORT', () => {
    const r = runSim({ ...good, cols: [6, 7] });
    expect(r.verdict).toBe('FAIL');
    expect(r.failCause).toBe('NO_SUPPORT');
  });

  it('is deterministic', () => {
    expect(runSim({ ...good, anchored: false })).toEqual(runSim({ ...good, anchored: false }));
  });

  it('always terminates within MAX_TICKS', () => {
    const r = runSim({ cols: [], anchored: false, rigid: false, sturdy: false });
    expect(r.verdict).toBe('FAIL');
    expect(r.states.length).toBeLessThanOrEqual(61);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test` — Expected: FAIL, module not found.

- [ ] **Step 3: Write implementation**

`src/core/sim.ts`:
```ts
import { GOAL_COL, START_COL, isGround, type BridgeModel } from './model';

export const DRIFT_PERIOD = 3;
export const MAX_TICKS = 60;

export type FailCause = 'NO_SUPPORT' | 'DRIFTED' | 'SWAYED' | 'COLLAPSED';

export interface SimState {
  tick: number;
  pawnCol: number;
  bridgeCols: number[];
  swayUp: boolean;
  collapsed: number[];
  visited: number[];
  status: 'RUNNING' | 'WIN' | 'FAIL';
  failCause: FailCause | null;
}

export interface SimResult {
  states: SimState[];
  verdict: 'WIN' | 'FAIL';
  failCause: FailCause | null;
}

export function initialSimState(model: BridgeModel): SimState {
  return {
    tick: 0,
    pawnCol: START_COL,
    bridgeCols: [...model.cols],
    swayUp: false,
    collapsed: [],
    visited: [START_COL],
    status: 'RUNNING',
    failCause: null,
  };
}

export function step(prev: SimState, model: BridgeModel): SimState {
  if (prev.status !== 'RUNNING') return prev;

  const tick = prev.tick + 1;

  let bridgeCols = prev.bridgeCols;
  if (!model.anchored && tick % DRIFT_PERIOD === 0) {
    bridgeCols = bridgeCols.map((c) => c - 1);
  }

  const swayUp = !model.rigid && tick % 2 === 1;

  let collapsed = prev.collapsed;
  if (!model.sturdy) {
    const newlyCollapsed = prev.visited.filter(
      (c) => c !== prev.pawnCol && bridgeCols.includes(c) && !collapsed.includes(c),
    );
    if (newlyCollapsed.length > 0) collapsed = [...collapsed, ...newlyCollapsed];
  }

  const pawnCol = prev.pawnCol + 1;
  const visited = [...prev.visited, pawnCol];
  const base = { tick, pawnCol, bridgeCols, swayUp, collapsed, visited };

  const supported =
    isGround(pawnCol) ||
    (bridgeCols.includes(pawnCol) && !collapsed.includes(pawnCol) && !swayUp);

  if (!supported) {
    let failCause: FailCause = 'NO_SUPPORT';
    if (collapsed.includes(pawnCol)) failCause = 'COLLAPSED';
    else if (swayUp && bridgeCols.includes(pawnCol)) failCause = 'SWAYED';
    else if (!model.anchored && model.cols.includes(pawnCol)) failCause = 'DRIFTED';
    return { ...base, status: 'FAIL', failCause };
  }

  if (pawnCol >= GOAL_COL) return { ...base, status: 'WIN', failCause: null };
  return { ...base, status: 'RUNNING', failCause: null };
}

export function runSim(model: BridgeModel): SimResult {
  const states: SimState[] = [initialSimState(model)];
  let current = states[0]!;
  while (current.status === 'RUNNING' && current.tick < MAX_TICKS) {
    current = step(current, model);
    states.push(current);
  }
  return {
    states,
    verdict: current.status === 'WIN' ? 'WIN' : 'FAIL',
    failCause: current.status === 'FAIL' ? current.failCause : null,
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test` — Expected: all pass. If SWAYED fires where DRIFTED is expected or vice versa, re-read the tick semantics block above — the cause priority and the parity/period constants are the contract; fix the implementation, not the test.

- [ ] **Step 5: Commit**

```bash
git add src/core/sim.ts tests/sim.test.ts
git commit -m "feat: deterministic crossing sim with drift/sway/collapse"
```

---

### Task 6: Scenario state — attempts, betrayal unlocks, clause toggling

**Files:**
- Create: `src/core/scenario.ts`
- Test: `tests/scenario.test.ts`

**Interfaces:**
- Consumes: `PredicateId` from `model.ts`; `FailCause` from `sim.ts`
- Produces:
  - `interface ScenarioState { attempt: number; maxAttempts: number; vocabulary: PredicateId[]; clauseBudget: number; status: 'PLAYING' | 'WON' | 'LOST' }`
  - `initialScenario(): ScenarioState` — attempt 1, maxAttempts 5, vocabulary `['SPANS_GAP', 'BEARS_WEIGHT']`, clauseBudget 3, status PLAYING
  - `unlockFor(cause: FailCause): PredicateId | null` — DRIFTED→ANCHORED, SWAYED→RIGID, else null
  - `applyVerdict(s: ScenarioState, verdict: 'WIN' | 'FAIL', failCause: FailCause | null): ScenarioState`
  - `toggleClause(spec: PredicateId[], p: PredicateId, budget: number): PredicateId[]`

- [ ] **Step 1: Write the failing test**

`tests/scenario.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { initialScenario, unlockFor, applyVerdict, toggleClause } from '../src/core/scenario';

describe('unlockFor', () => {
  it('maps betrayal causes to predicates', () => {
    expect(unlockFor('DRIFTED')).toBe('ANCHORED');
    expect(unlockFor('SWAYED')).toBe('RIGID');
    expect(unlockFor('NO_SUPPORT')).toBeNull();
    expect(unlockFor('COLLAPSED')).toBeNull();
  });
});

describe('applyVerdict', () => {
  it('WIN ends the scenario as WON', () => {
    expect(applyVerdict(initialScenario(), 'WIN', null).status).toBe('WON');
  });

  it('FAIL with DRIFTED unlocks ANCHORED and advances the attempt', () => {
    const next = applyVerdict(initialScenario(), 'FAIL', 'DRIFTED');
    expect(next.vocabulary).toContain('ANCHORED');
    expect(next.attempt).toBe(2);
    expect(next.status).toBe('PLAYING');
  });

  it('does not duplicate an already-unlocked predicate', () => {
    const once = applyVerdict(initialScenario(), 'FAIL', 'DRIFTED');
    const twice = applyVerdict(once, 'FAIL', 'DRIFTED');
    expect(twice.vocabulary.filter((p) => p === 'ANCHORED')).toHaveLength(1);
  });

  it('failing the last attempt loses the scenario', () => {
    let s = initialScenario();
    for (let i = 0; i < 5; i++) s = applyVerdict(s, 'FAIL', 'NO_SUPPORT');
    expect(s.status).toBe('LOST');
  });
});

describe('toggleClause', () => {
  it('adds, removes, and enforces the budget', () => {
    expect(toggleClause([], 'SPANS_GAP', 3)).toEqual(['SPANS_GAP']);
    expect(toggleClause(['SPANS_GAP'], 'SPANS_GAP', 3)).toEqual([]);
    expect(toggleClause(['SPANS_GAP', 'ANCHORED', 'RIGID'], 'BEARS_WEIGHT', 3)).toEqual([
      'SPANS_GAP',
      'ANCHORED',
      'RIGID',
    ]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test` — Expected: FAIL, module not found.

- [ ] **Step 3: Write implementation**

`src/core/scenario.ts`:
```ts
import type { PredicateId } from './model';
import type { FailCause } from './sim';

export interface ScenarioState {
  attempt: number;
  maxAttempts: number;
  vocabulary: PredicateId[];
  clauseBudget: number;
  status: 'PLAYING' | 'WON' | 'LOST';
}

export function initialScenario(): ScenarioState {
  return {
    attempt: 1,
    maxAttempts: 5,
    vocabulary: ['SPANS_GAP', 'BEARS_WEIGHT'],
    clauseBudget: 3,
    status: 'PLAYING',
  };
}

export function unlockFor(cause: FailCause): PredicateId | null {
  if (cause === 'DRIFTED') return 'ANCHORED';
  if (cause === 'SWAYED') return 'RIGID';
  return null;
}

export function applyVerdict(
  s: ScenarioState,
  verdict: 'WIN' | 'FAIL',
  failCause: FailCause | null,
): ScenarioState {
  if (verdict === 'WIN') return { ...s, status: 'WON' };
  const unlock = failCause ? unlockFor(failCause) : null;
  const vocabulary =
    unlock && !s.vocabulary.includes(unlock) ? [...s.vocabulary, unlock] : s.vocabulary;
  const attempt = s.attempt + 1;
  return {
    ...s,
    vocabulary,
    attempt,
    status: attempt > s.maxAttempts ? 'LOST' : 'PLAYING',
  };
}

export function toggleClause(
  spec: PredicateId[],
  p: PredicateId,
  budget: number,
): PredicateId[] {
  if (spec.includes(p)) return spec.filter((x) => x !== p);
  if (spec.length >= budget) return spec;
  return [...spec, p];
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test` — Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git add src/core/scenario.ts tests/scenario.test.ts
git commit -m "feat: scenario state with betrayal-driven vocabulary unlocks"
```

---

### Task 7: Canvas renderer

**Files:**
- Create: `src/ui/renderer.ts`

**Interfaces:**
- Consumes: `SimState` from `sim.ts`; grid constants and `isGround` from `model.ts`
- Produces: `drawFrame(ctx: CanvasRenderingContext2D, state: SimState): void` — renders one sim state to the 768×432 canvas (16×9 grid, 48px cells).

No unit test (browser-only API); verified visually in Step 2 and exercised end-to-end in Task 8.

- [ ] **Step 1: Write implementation**

`src/ui/renderer.ts`:
```ts
import {
  GRID_COLS,
  GRID_ROWS,
  SUPPORT_ROW,
  WALK_ROW,
  GOAL_COL,
  isGround,
} from '../core/model';
import type { SimState } from '../core/sim';

const CELL = 48;
const COLORS = {
  bg: '#0a0a0f',
  ground: '#2a2a33',
  groundEdge: '#3a3a45',
  bridge: '#cfcabe',
  pawn: '#e8e4d8',
  goal: '#6b675e',
};

export function drawFrame(ctx: CanvasRenderingContext2D, state: SimState): void {
  ctx.fillStyle = COLORS.bg;
  ctx.fillRect(0, 0, GRID_COLS * CELL, GRID_ROWS * CELL);

  // ground slabs: support row down to bottom
  for (let col = 0; col < GRID_COLS; col++) {
    if (!isGround(col)) continue;
    ctx.fillStyle = COLORS.ground;
    ctx.fillRect(col * CELL, SUPPORT_ROW * CELL, CELL, (GRID_ROWS - SUPPORT_ROW) * CELL);
    ctx.fillStyle = COLORS.groundEdge;
    ctx.fillRect(col * CELL, SUPPORT_ROW * CELL, CELL, 3);
  }

  // goal marker: thin vertical line above the goal column
  ctx.strokeStyle = COLORS.goal;
  ctx.lineWidth = 1;
  ctx.beginPath();
  ctx.moveTo(GOAL_COL * CELL + CELL / 2, (SUPPORT_ROW - 2) * CELL);
  ctx.lineTo(GOAL_COL * CELL + CELL / 2, SUPPORT_ROW * CELL);
  ctx.stroke();

  // bridge cells (skip collapsed; offset up when swaying)
  const swayOffset = state.swayUp ? -10 : 0;
  ctx.fillStyle = COLORS.bridge;
  for (const col of state.bridgeCols) {
    if (state.collapsed.includes(col)) continue;
    ctx.fillRect(col * CELL + 2, SUPPORT_ROW * CELL + swayOffset, CELL - 4, 8);
  }

  // pawn: small light rect standing on the walk row
  if (state.status !== 'FAIL') {
    const px = state.pawnCol * CELL + CELL / 2 - 7;
    const py = WALK_ROW * CELL + CELL - 26;
    ctx.fillStyle = COLORS.pawn;
    ctx.fillRect(px, py, 14, 26);
  } else {
    // fallen pawn: drawn below the support row, fading
    const px = state.pawnCol * CELL + CELL / 2 - 7;
    ctx.fillStyle = COLORS.pawn + '66';
    ctx.fillRect(px, (SUPPORT_ROW + 1) * CELL, 14, 26);
  }
}
```

- [ ] **Step 2: Visual verification**

Temporarily replace the body of `src/ui/main.ts` with:
```ts
import { drawFrame } from './renderer';
import { initialSimState } from '../core/sim';

const canvas = document.querySelector<HTMLCanvasElement>('#game')!;
const ctx = canvas.getContext('2d')!;
drawFrame(ctx, initialSimState({ cols: [6, 7, 8, 9], anchored: true, rigid: true, sturdy: true }));
```

Run: `npm run dev`, open the served URL, and screenshot the page (headless is fine: `google-chrome --headless --screenshot=/tmp/boxed-render.png --window-size=900,500 http://localhost:5173` or the WSL equivalent; if unavailable, view in the Windows browser). Confirm against this checklist: two dark-grey ground slabs left and right, a black 4-column chasm between them, a pale thin bridge strip across the chasm, a small light pawn standing on the left ground, a faint goal line on the right. Fix drawing bugs before proceeding.

- [ ] **Step 3: Verify types and tests still pass**

Run: `npm run build` — Expected: no type errors.
Run: `npm test` — Expected: all pass.

- [ ] **Step 4: Commit**

```bash
git add src/ui/renderer.ts src/ui/main.ts
git commit -m "feat: canvas renderer for grid, bridge, pawn"
```

---

### Task 8: Spec panel, receipt overlay, game shell

**Files:**
- Create: `src/ui/specPanel.ts`, `src/ui/receipt.ts`
- Modify: `src/ui/main.ts` (full replacement), `CLAUDE.md` (Status + Architecture)

**Interfaces:**
- Consumes: everything from Tasks 2–7. Targets DOM ids `#game`, `#spec-panel`, `#overlay` from Task 1's `index.html`; CSS classes `chip`, `selected`, `new`, `meta`, `receipt`, `row`, `check` from Task 1's `style.css`.
- Produces: the playable slice.
  - `specPanel.ts`: `renderSpecPanel(el: HTMLElement, opts: { vocabulary: PredicateId[]; spec: PredicateId[]; budget: number; attempt: number; maxAttempts: number; newChip: PredicateId | null; canSummon: boolean; onToggle: (p: PredicateId) => void; onSummon: () => void }): void`
  - `receipt.ts`: `showReceipt(el: HTMLElement, opts: { spec: PredicateId[]; verdict: 'WIN' | 'FAIL'; scenarioOver: 'WON' | 'LOST' | null; onDismiss: () => void }): void` and `hideOverlay(el: HTMLElement): void`

- [ ] **Step 1: Write the spec panel**

`src/ui/specPanel.ts`:
```ts
import { PREDICATE_LABEL, type PredicateId } from '../core/model';

export interface SpecPanelOpts {
  vocabulary: PredicateId[];
  spec: PredicateId[];
  budget: number;
  attempt: number;
  maxAttempts: number;
  newChip: PredicateId | null;
  canSummon: boolean;
  onToggle: (p: PredicateId) => void;
  onSummon: () => void;
}

export function renderSpecPanel(el: HTMLElement, opts: SpecPanelOpts): void {
  el.replaceChildren();

  const title = document.createElement('div');
  title.className = 'meta';
  title.textContent = `spec — ${opts.spec.length}/${opts.budget} clauses`;
  el.appendChild(title);

  for (const p of opts.vocabulary) {
    const chip = document.createElement('button');
    chip.className = 'chip';
    if (opts.spec.includes(p)) chip.classList.add('selected');
    if (opts.newChip === p) chip.classList.add('new');
    chip.textContent = PREDICATE_LABEL[p];
    chip.addEventListener('click', () => opts.onToggle(p));
    el.appendChild(chip);
  }

  const summon = document.createElement('button');
  summon.id = 'summon';
  summon.textContent = 'SUMMON';
  summon.disabled = !opts.canSummon;
  summon.addEventListener('click', opts.onSummon);
  el.appendChild(summon);

  const attempts = document.createElement('div');
  attempts.className = 'meta';
  attempts.textContent = `attempt ${opts.attempt} of ${opts.maxAttempts}`;
  el.appendChild(attempts);
}
```

- [ ] **Step 2: Write the receipt overlay**

`src/ui/receipt.ts` — the receipt lists each clause with a green check and *nothing else*. No cause, no hint. Win shows a single quiet line. Scenario end states get one extra line.

```ts
import { PREDICATE_LABEL, type PredicateId } from '../core/model';

export interface ReceiptOpts {
  spec: PredicateId[];
  verdict: 'WIN' | 'FAIL';
  scenarioOver: 'WON' | 'LOST' | null;
  onDismiss: () => void;
}

export function showReceipt(el: HTMLElement, opts: ReceiptOpts): void {
  el.replaceChildren();
  const box = document.createElement('div');
  box.className = 'receipt';

  if (opts.verdict === 'FAIL') {
    const head = document.createElement('div');
    head.className = 'meta';
    head.textContent = 'specification satisfied';
    box.appendChild(head);
    for (const p of opts.spec) {
      const row = document.createElement('div');
      row.className = 'row';
      const label = document.createElement('span');
      label.textContent = PREDICATE_LABEL[p];
      const check = document.createElement('span');
      check.className = 'check';
      check.textContent = '✓';
      row.append(label, check);
      box.appendChild(row);
    }
    if (opts.spec.length === 0) {
      const row = document.createElement('div');
      row.className = 'meta';
      row.textContent = '(no clauses)';
      box.appendChild(row);
    }
  } else {
    const head = document.createElement('div');
    head.textContent = 'crossed.';
    box.appendChild(head);
  }

  if (opts.scenarioOver === 'LOST') {
    const line = document.createElement('div');
    line.className = 'meta';
    line.textContent = 'no summons remain';
    box.appendChild(line);
  }

  const btn = document.createElement('button');
  btn.id = 'summon';
  btn.textContent = opts.scenarioOver ? 'AGAIN' : 'CONTINUE';
  btn.addEventListener('click', opts.onDismiss);
  box.appendChild(btn);

  el.appendChild(box);
  el.hidden = false;
}

export function hideOverlay(el: HTMLElement): void {
  el.hidden = true;
  el.replaceChildren();
}
```

- [ ] **Step 3: Write the game shell**

`src/ui/main.ts` (full replacement):
```ts
import { instantiate } from '../core/instantiator';
import { makeRng } from '../core/rng';
import { runSim, initialSimState, type SimResult } from '../core/sim';
import {
  initialScenario,
  applyVerdict,
  unlockFor,
  toggleClause,
  type ScenarioState,
} from '../core/scenario';
import type { PredicateId, Spec } from '../core/model';
import { drawFrame } from './renderer';
import { renderSpecPanel } from './specPanel';
import { showReceipt, hideOverlay } from './receipt';

const SCENARIO_SEED = 20260727;
const TICK_MS = 250;

const canvas = document.querySelector<HTMLCanvasElement>('#game')!;
const ctx = canvas.getContext('2d')!;
const panelEl = document.querySelector<HTMLElement>('#spec-panel')!;
const overlayEl = document.querySelector<HTMLElement>('#overlay')!;

let scenario: ScenarioState = initialScenario();
let spec: Spec = [];
let newChip: PredicateId | null = null;
let animating = false;

function refresh(): void {
  renderSpecPanel(panelEl, {
    vocabulary: scenario.vocabulary,
    spec,
    budget: scenario.clauseBudget,
    attempt: scenario.attempt,
    maxAttempts: scenario.maxAttempts,
    newChip,
    canSummon: !animating && scenario.status === 'PLAYING',
    onToggle: (p) => {
      if (animating) return;
      spec = toggleClause(spec, p, scenario.clauseBudget);
      refresh();
    },
    onSummon: summon,
  });
}

function summon(): void {
  if (animating || scenario.status !== 'PLAYING') return;
  animating = true;
  newChip = null;
  refresh();

  const rand = makeRng(SCENARIO_SEED + scenario.attempt);
  const model = instantiate(spec, rand);
  const result = runSim(model);
  animate(result);
}

function animate(result: SimResult): void {
  let i = 0;
  const timer = setInterval(() => {
    const state = result.states[i];
    if (!state) {
      clearInterval(timer);
      finish(result);
      return;
    }
    drawFrame(ctx, state);
    i++;
  }, TICK_MS);
}

function finish(result: SimResult): void {
  const before = scenario;
  scenario = applyVerdict(scenario, result.verdict, result.failCause);
  const unlocked =
    result.failCause !== null &&
    scenario.vocabulary.length > before.vocabulary.length
      ? unlockFor(result.failCause)
      : null;

  showReceipt(overlayEl, {
    spec,
    verdict: result.verdict,
    scenarioOver: scenario.status === 'PLAYING' ? null : scenario.status,
    onDismiss: () => {
      hideOverlay(overlayEl);
      animating = false;
      newChip = unlocked;
      if (scenario.status !== 'PLAYING') {
        scenario = initialScenario();
        spec = [];
        newChip = null;
      }
      resetCanvas();
      refresh();
    },
  });
}

function resetCanvas(): void {
  drawFrame(ctx, initialSimState({ cols: [], anchored: true, rigid: true, sturdy: true }));
}

resetCanvas();
refresh();
```

- [ ] **Step 4: Verify types and tests**

Run: `npm run build` — Expected: no type errors.
Run: `npm test` — Expected: all pass.

- [ ] **Step 5: Playtest checklist (dev server, in browser)**

Run: `npm run dev` and walk this exact script:

1. Initial view: empty chasm scene, two chips (`IT SPANS THE GAP`, `IT BEARS MY WEIGHT`), `attempt 1 of 5`.
2. Summon with **no clauses**: some bridge appears (probably wrong/partial), pawn walks, almost certainly falls. Receipt shows `(no clauses)` — no explanation.
3. Summon with `SPANS THE GAP` selected, repeatedly across attempts: bridge always spans at summon time (fairness), but drifting/swaying picks fail. On a drift death the `IT IS ANCHORED` chip appears with a pulse; on a sway death, `IT IS RIGID`. Confirm the receipt shows every selected clause green-checked over the wreckage.
4. Select `SPANS + ANCHORED + RIGID` (drop `BEARS MY WEIGHT` — budget is 3): pawn crosses. If the pick was fragile, cells visibly vanish *behind* the pawn and it still wins. Overlay reads `crossed.`
5. Exhaust 5 attempts failing: `no summons remain`, AGAIN resets to the initial vocabulary.
6. Screenshot at least one mid-crossing frame and one receipt-over-wreckage frame; confirm they match the sparse dark aesthetic (no bright colors, no explanatory text).

Note: the seed sequence is fixed (`SCENARIO_SEED + attempt`), so this script is reproducible. If step 3 can't produce a drift death within the 5 attempts because the seeded picks happen to be benign, adjust `SCENARIO_SEED` to a value where attempts 1–2 with `['SPANS_GAP']` yield an unanchored and a non-rigid pick respectively (check quickly in a scratch script with `instantiate(['SPANS_GAP'], makeRng(seed + attempt))`), and note the chosen seed in the commit message.

- [ ] **Step 6: Update CLAUDE.md**

Replace the `## Architecture (planned — update as built)` section with:
```markdown
## Architecture

- `src/core/` — pure deterministic logic, no DOM/browser APIs, fully unit-tested:
  rng (seeded mulberry32), model (grid constants, BridgeModel, predicates ids),
  predicates (clause satisfaction), instantiator (conforming-model picker;
  fairness invariant property-tested), sim (tick-based crossing sim),
  scenario (attempts, betrayal unlocks, clause budget)
- `src/ui/` — renderer (Canvas2D), specPanel + receipt (DOM), main (game shell)
- Core invariant: every instantiated entity satisfies every clause of its spec
  (property-tested; never violate — it is the game's contract with the player)
```
Replace the `## Status` body with: `Vertical slice playable (one chasm scenario, 4 predicates). Branch feat/vertical-slice.`

- [ ] **Step 7: Commit**

```bash
git add src/ui/specPanel.ts src/ui/receipt.ts src/ui/main.ts CLAUDE.md
git commit -m "feat: spec panel, receipt overlay, playable game shell"
```

---

### Task 9: README

**Files:**
- Create: `README.md`

**Interfaces:**
- Consumes: nothing (prose only)
- Produces: repo landing doc.

- [ ] **Step 1: Write README**

`README.md`:
```markdown
# Boxed

A puzzle game about the gap between what you specify and what you meant.

You need things done — cross the chasm, keep the fire lit. You can't do them
yourself. You can only write specifications, and something will instantiate
them: always conforming to every clause you wrote, never what you intended.
The failure screen shows your spec, fully satisfied, in green, over the
wreckage. The game never cheats and never explains.

## Run

- `npm install`
- `npm run dev` — play at the printed URL
- `npm test` — unit tests
- `npm run build` — production bundle in `dist/`

## Design

- Spec: `docs/superpowers/specs/2026-07-27-boxed-game-design.md`
- Decisions: `ADR.md`

Status: vertical slice — one scenario (the chasm), four predicates.
```

- [ ] **Step 2: Verify everything green one last time**

Run: `npm test` && `npm run build` — Expected: pass, no errors.
Run: `git status` — Expected: only `README.md` untracked; nothing else modified.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: README"
```
