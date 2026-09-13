# Vite — Testing with Vitest (Part 12)

Testing on the same config, plugins and module pipeline as your app: setup, environments, mocking, coverage, monorepo projects and Browser Mode, written against Vitest 5, released ten days before this doc.

> **Spec:** this doc follows the shared spec in [`vite-mastery-guide.md`](./vite-mastery-guide.md#about-this-document). Written against **Vitest 5.0.0** on **Vite 8.3.0**, jsdom 30.0.1, `@testing-library/react` 16.3.3, `@testing-library/jest-dom` 7.0.1. Verified **September 13, 2026**.

Vitest 5 is **very new**, and it changed defaults that Vitest 4 tutorials rely on (`clearMocks`, `vi.mock` placement, project inheritance, the `-t` filter). Read [§12.6](#part-12-v4-to-v5) if you're following older material.

---

## Table of Contents

- [Part 12: Testing with Vitest](#part-12-testing-with-vitest)
  - [12.1 Beginner — install, first test, run](#part-12-beginner)
  - [12.2 Working Knowledge — config sharing, environments, component tests](#part-12-working)
  - [12.3 Advanced — mocking modules without fighting hoisting](#part-12-advanced)
  - [12.4 Mastery — coverage, projects, Browser Mode, performance](#part-12-mastery)
  - [12.5 Cheat sheet: Part 12](#part-12-cheat-sheet)
  - [12.6 Vitest 4 → 5: what changed under you](#part-12-v4-to-v5)

---

<a id="part-12-testing-with-vitest"></a>

## Part 12: Testing with Vitest

**Why Vitest rather than Jest for a Vite app.** Vitest reads your `vite.config.*` by default ([Vitest Getting Started](https://vitest.dev/guide/)), so aliases, `define`, plugins (JSX, Vue SFCs, virtual modules) and `import.meta.env` work in tests with no second configuration. Jest needs its own transform setup that mirrors Vite's, and the two drift apart.

<a id="part-12-beginner"></a>

### 12.1 Beginner — install, first test, run

**Requirements: Node ≥ 22.12 and Vite ≥ 6.4** ([Vitest Getting Started](https://vitest.dev/guide/)). Note that the Node floor is higher than Vite 8's own.

```bash
npm install -D vitest
```

```ts
// src/lib/math.ts
export function add(a: number, b: number) {
  return a + b
}
```

```ts
// src/lib/math.test.ts: files must contain .test. or .spec.
import { expect, test } from 'vitest'
import { add } from './math'

test('adds', () => {
  expect(add(1, 2)).toBe(3)
})
```

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "coverage": "vitest run --coverage"
  }
}
```

`vitest` starts in **watch mode**. `vitest run` runs once, which is what CI scripts should call explicitly ([Vitest Getting Started](https://vitest.dev/guide/)). Vitest 5 writes reports and artifacts to a `.vitest/` directory, which you should add to `.gitignore` ([Vitest 5 announcement](https://vitest.dev/blog/vitest-5.html)).

`globals` defaults to `false`, so `import { test, expect } from 'vitest'` is explicit ([Vitest Config](https://vitest.dev/config/)). That's the recommended style, since it needs no global type setup.

<a id="part-12-working"></a>

### 12.2 Working Knowledge — config sharing, environments, component tests

**Put test config in `vite.config.ts`.** Add the reference line for types and a `test` block ([Vitest Config](https://vitest.dev/config/)):

```ts
/// <reference types="vitest/config" />
import react from '@vitejs/plugin-react'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [react()],
  resolve: { tsconfigPaths: true },
  test: {
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
  },
})
```

**A separate `vitest.config.ts` replaces `vite.config.ts` rather than merging with it.** The docs say it *"will have the higher priority and will override the configuration from vite.config.ts"* ([Vitest Config](https://vitest.dev/config/)). *Verified locally*: with `resolve.tsconfigPaths: true` in `vite.config.ts`, adding a bare `vitest.config.ts` made an `@/lib/math` import fail:

```text
Error: Cannot find package '@/lib/math' imported from …/src/lib/math.test.ts
```

```ts
// Wrong: aliases, plugins and define from vite.config.ts silently disappear
// vitest.config.ts
import { defineConfig } from 'vitest/config'
export default defineConfig({ test: { environment: 'node' } })

// Right: merge explicitly (verified locally: the alias resolves again)
import { defineConfig, mergeConfig } from 'vitest/config'
import viteConfig from './vite.config'
export default mergeConfig(viteConfig, defineConfig({
  test: { environment: 'node' },
}))
```

If `vite.config.ts` exports a *function*, call it first: `mergeConfig(viteConfig({ command: 'serve', mode: 'test' }), …)`.

**Environments** ([Test Environments](https://vitest.dev/guide/environment)):

| `environment` | Globals | Use | Install |
|---|---|---|---|
| `node` (**default**) | Node | pure logic, API clients, SSR code | — |
| `jsdom` | simulated DOM | component tests; most complete DOM emulation | `jsdom` |
| `happy-dom` | simulated DOM | component tests; "considered to be faster than jsdom, but lacks some API" | `happy-dom` |
| `edge-runtime` | Vercel Edge | edge functions | see [Test Environments](https://vitest.dev/guide/environment) |

Override per file with a comment at the very top:

```ts
// @vitest-environment jsdom
```

Neither simulated DOM does layout. `getBoundingClientRect()` returns zeros, and CSS doesn't apply. Tests that depend on either belong in **Browser Mode** ([§12.4](#part-12-mastery)).

**Env values inside tests.** Vitest runs with `mode: 'test'` and loads `.env`, `.env.test` and their `.local` variants like any other mode. *Verified locally* (Vitest 5.0.0):

| | `jsdom` | `node` |
|---|---|---|
| `import.meta.env.MODE` | `test` | `test` |
| `import.meta.env.DEV` / `PROD` | `true` / `false` | `true` / `false` |
| `import.meta.env.SSR` | `false` | **`true`** |
| `process.env.NODE_ENV` | `test` | `test` |
| `VITE_*` from `.env` | ✔ loaded | ✔ loaded |

That `SSR: true` under `node` matters: a module with `if (import.meta.env.SSR)` branches behaves differently depending on which environment you test it in.

**Component tests: React.** *Verified locally* against the `react-ts` template's counter:

```bash
npm i -D jsdom @testing-library/react @testing-library/user-event @testing-library/jest-dom
```

```ts
// src/test/setup.ts: adds toBeInTheDocument(), toHaveTextContent(), … to expect
import '@testing-library/jest-dom/vitest'
```

```tsx
// src/App.test.tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { expect, test } from 'vitest'
import App from './App'

test('counter increments', async () => {
  render(<App />)
  await userEvent.click(screen.getByRole('button', { name: /count is 0/i }))
  expect(screen.getByRole('button', { name: /count is 1/i })).toBeInTheDocument()
})
```

Result: `Test Files 1 passed · Tests 2 passed · Duration 21.95s (environment 78%, setup 14%, import 6%, tests 1%)`. That percentage breakdown is new in Vitest 5 ([announcement](https://vitest.dev/blog/vitest-5.html)). Here it shows that jsdom startup, not the test, is the cost. The time is a cold first run on a Windows temp directory, so don't treat it as typical.

**Component tests: Vue.** Same setup, with `@testing-library/vue` or `@vue/test-utils` and `@vitejs/plugin-vue` in `plugins`, so `.vue` files compile. One Vitest 5 change: in **Browser Mode**, Vue and Svelte `render()` now returns a promise, so `await render(...)` ([Vitest 5 migration](https://vitest.dev/guide/migration)).

**Real Scenario — "tests pass locally, can't resolve imports in CI".** A developer adds `vitest.config.ts` to set `coverage.thresholds` for a CI gate. Locally they only ever run `npm test -- src/utils`, where the tests have no aliased imports, and everything is green. In CI, the full suite fails on every `@/components/...` import. The new config replaced `vite.config.ts`, so `resolve.tsconfigPaths` and the React plugin vanished. The fix is `mergeConfig` (above), or simply moving `coverage` into the existing `test` block.

<a id="part-12-advanced"></a>

### 12.3 Advanced — mocking modules without fighting hoisting

**`vi.mock` is hoisted.** Vitest moves every `vi.mock()` call above the imports, so the mock is registered before the module under test loads ([Mocking Modules](https://vitest.dev/guide/mocking/modules)). Two consequences:

**1. The factory can't use variables declared in the file.** *Verified locally*:

```ts
// Wrong
import { add } from '@/lib/math'
const fake = vi.fn(() => 42)
vi.mock('@/lib/math', () => ({ add: fake }))
// → Error: [vitest] There was an error when mocking a module. If you are using "vi.mock"
//   factory, make sure there are no top level variables inside, since this call is hoisted…
//   Caused by: ReferenceError: Cannot access 'fake' before initialization

// Right: vi.hoisted runs at the hoisted position too
import { add } from '@/lib/math'
const { fake } = vi.hoisted(() => ({ fake: vi.fn(() => 42) }))
vi.mock('@/lib/math', () => ({ add: fake }))
test('uses the mock', () => {
  expect(add(1, 2)).toBe(42)
  expect(fake).toHaveBeenCalledOnce()
})
```

**2. Vitest 5 makes non-top-level `vi.mock` an error.** Vitest 4 warned. Vitest 5 throws ([Vitest 5 migration](https://vitest.dev/guide/migration)). *Verified locally*:

```ts
// Wrong: looks scoped to one test, but it's hoisted and applies to the whole file
test('nested mock', () => {
  vi.mock('./App', () => ({ default: () => null }))
})
```

```text
Error: 1 call in "src/nested.test.ts" was defined outside of the module's top level scope:
- vi.mock("./App") at src/nested.test.ts:3:3
Although it appears nested, it will be hoisted and executed before anything in this file.
Move it to the top level to reflect its actual execution order.
```

For a mock that really is per test, use `vi.doMock` (not hoisted) plus a dynamic `await import()` after it, or spy on an export with `vi.spyOn`.

**Partial mocks: keep the real module, replace one export** ([Mocking Modules](https://vitest.dev/guide/mocking/modules)):

```ts
vi.mock(import('./api'), async (importOriginal) => {
  const actual = await importOriginal()           // async: must be awaited
  return { ...actual, fetchUser: vi.fn(async () => ({ id: 1, name: 'Test' })) }
})
```

Passing `import('./api')` rather than the string `'./api'` gives the factory a typed `importOriginal`. `vi.mock(path, { spy: true })` keeps every real implementation while recording calls ([Mocking Modules](https://vitest.dev/guide/mocking/modules)).

**A mock can't intercept calls within the same module.** *Verified locally*:

```ts
// math.ts
export function add(a: number, b: number) { return a + b }
export function double(n: number) { return add(n, n) }   // calls add directly
```

```ts
import * as math from '@/lib/math'
vi.spyOn(math, 'add').mockReturnValue(100)
math.add(1, 1)     // → 100  (goes through the module namespace)
math.double(2)     // → 4    (internal call bypasses the spy)
```

The docs say this is intentional and won't be worked around: *"it is not possible to mock calls to methods that are called inside other methods of the same file"* ([Mocking Modules](https://vitest.dev/guide/mocking/modules)). Split the file or inject the dependency.

**Conditional return values (Vitest 5 `vi.when`).** Replaces hand-written `mockImplementation` switch statements ([Vitest 5 announcement](https://vitest.dev/blog/vitest-5.html)):

```ts
const findById = vi.fn()
vi.when(findById)
  .calledWith(1).thenResolve({ id: 1, name: 'Ella' })
  .calledWith(expect.any(Number)).thenReject(new Error('not found'))
```

**`clearMocks` is now on by default.** Call history resets before every test. Implementations are kept ([Vitest 5 migration](https://vitest.dev/guide/migration)). *Verified locally*: a `vi.fn()` called in one test showed `toHaveBeenCalledTimes(0)` in the next. Tests that asserted cumulative call counts across `test()` blocks break on upgrade. Set `clearMocks: false` only as a stopgap.

<a id="part-12-mastery"></a>

### 12.4 Mastery — coverage, projects, Browser Mode, performance

**Coverage** ([Coverage](https://vitest.dev/guide/coverage)):

```bash
npm i -D @vitest/coverage-v8     # version must match vitest
```

```ts
test: {
  coverage: {
    provider: 'v8',                       // default
    include: ['src/**/*.{ts,tsx}'],       // recommended: otherwise untested files are invisible
    exclude: ['src/**/*.test.*', 'src/test/**'],
    reporter: ['text', 'html'],
    thresholds: { lines: 80, branches: 70 },
  },
},
```

*Verified locally*: with `include: ['src/lib/**/*.ts']`, a file with an untested function reported `math.ts | 66.66 | 0 | 66.66 | 66.66 | 3`, i.e. 66.66% of lines and functions covered, 0% of branches, and line 3 uncovered. **Without `include`, only files your tests import show up**, so a completely untested module reads as nothing rather than 0% ([Coverage](https://vitest.dev/guide/coverage)).

| Provider | How | Choose when |
|---|---|---|
| `v8` | V8's built-in coverage + AST remapping | default; faster and lighter. Can't run on non-V8 runtimes (Firefox browser mode, Bun) |
| `istanbul` | instruments source before running | you need Firefox/WebKit browser coverage or a non-V8 runtime |

Ignore hints need `@preserve` so transforms don't strip the comment: `/* v8 ignore next -- @preserve */` ([Coverage](https://vitest.dev/guide/coverage)).

**Projects: several configurations in one run** (monorepos, or node + DOM tests in one app) ([Test Projects](https://vitest.dev/guide/projects)):

```ts
/// <reference types="vitest/config" />
import { defineConfig } from 'vite'
import { configDefaults } from 'vitest/config'

export default defineConfig({
  resolve: { tsconfigPaths: true },
  test: {
    projects: [
      'packages/*',                                              // each package's own config
      {
        test: {
          name: 'unit',
          include: ['src/**/*.test.ts'],
          exclude: [...configDefaults.exclude, 'src/**/*.dom.test.ts'],
          environment: 'node',
        },
      },
      {
        test: { name: 'dom', include: ['src/**/*.dom.test.ts'], environment: 'happy-dom' },
      },
    ],
  },
})
```

Things *verified locally* with exactly this shape:

- **Inline projects inherit the root config by default in Vitest 5** (`extends: true`). `resolve.tsconfigPaths` from the root applied to both projects with no repetition ([Vitest 5 migration](https://vitest.dev/guide/migration)).
- **Overlapping `include` globs run a file in *every* matching project.** Without the `exclude` line, `el.dom.test.ts` also matched `src/**/*.test.ts` and ran under `|unit|`, failing with `ReferenceError: document is not defined`. The error names the project in the output (`FAIL |unit| src/lib/el.dom.test.ts`). Read that prefix before debugging the test itself.
- **Replacing `exclude` drops the defaults** (like `node_modules`) unless you spread `configDefaults.exclude`.

Run one project with `vitest --project dom` (`-p`). Negations and globs work: `--project 'unit*' --project '!unit (browser)'` ([Test Projects](https://vitest.dev/guide/projects)). `coverage`, `reporters` and `resolveSnapshotPath` are **root-only** and can't be set per project ([Test Projects](https://vitest.dev/guide/projects)). Vitest 5 also reuses one Vite server across inline projects that don't change Vite config (`sharedViteServer`, default `true`), and allows nested `projects` inside referenced config files ([Vitest 5 announcement](https://vitest.dev/blog/vitest-5.html)).

**Browser Mode: real browsers instead of a simulated DOM.** Stable as of Vitest 5 ([Browser Mode](https://vitest.dev/guide/browser/)):

```bash
npx vitest init browser
# or manually
npm i -D @vitest/browser-playwright vitest-browser-react   # or vitest-browser-vue
```

```ts
import { playwright } from '@vitest/browser-playwright'
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    browser: {
      enabled: true,
      provider: playwright(),
      headless: true,
      instances: [{ browser: 'chromium' }],
      traceView: true,            // Vitest 5: step-by-step DOM trace on failure
    },
  },
})
```

```tsx
import { render } from 'vitest-browser-react'
import { expect, test } from 'vitest'
import Fetch from './Fetch'

test('loads greeting', async () => {
  const screen = render(<Fetch url="/greeting" />)
  await screen.getByText('Load Greeting').click()
  await expect.element(screen.getByRole('heading')).toHaveTextContent('hello there')
})
```

Vitest 5 made **locators strict**: `getByText('Item')` no longer matches "Item 1", and `toHaveTextContent` is an exact comparison (use `toMatchTextContent` for partial matches) ([Vitest 5 migration](https://vitest.dev/guide/migration)). Use jsdom/happy-dom for fast logic-heavy component tests, and Browser Mode for layout, focus, real events, `IntersectionObserver` and CSS-dependent behaviour. In a project-based config, run both side by side. (Browser Mode was *not run locally* in this pass.)

**Performance levers.**

- `vitest doctor` (new in 5) analyzes the suite and recommends optimizations ([Vitest 5 announcement](https://vitest.dev/blog/vitest-5.html)).
- The duration breakdown shows where time goes. `environment 78%` means switch heavy files to `happy-dom` or to `node` where no DOM is needed.
- `fsModuleCache` persists transformed modules across reruns ([Vitest 5 announcement](https://vitest.dev/blog/vitest-5.html)).
- Barrel files hurt tests as much as dev ([Part 14](./vite-ssr-internals.md#part-14-internals-and-performance)). A test that imports one icon from `@/icons` transforms all of them.
- `--repeats N` runs every test N times to flush out flaky ones ([Vitest 5 announcement](https://vitest.dev/blog/vitest-5.html)).

<a id="part-12-cheat-sheet"></a>

### 12.5 Cheat sheet: Part 12

| Task | How |
|---|---|
| Config location | `test: {}` in `vite.config.ts` + `/// <reference types="vitest/config" />` |
| Separate config | `mergeConfig(viteConfig, defineConfig({ test }))`, never standalone |
| DOM | `environment: 'jsdom'` / `'happy-dom'`, or `// @vitest-environment jsdom` |
| jest-dom matchers | `setupFiles` → `import '@testing-library/jest-dom/vitest'` |
| Mock a module | top-level `vi.mock(import('./x'), factory)` |
| Variables in factory | `vi.hoisted(() => ({ … }))` |
| Per-test module mock | `vi.doMock` + dynamic `import()` |
| Partial mock | `importOriginal()` spread |
| Spy but keep real impl | `vi.mock(path, { spy: true })` / `vi.spyOn(ns, 'fn')` |
| Same-file internal calls | can't be mocked: split the module |
| Conditional mock returns | `vi.when(fn).calledWith(…).thenResolve(…)` (5.0) |
| Coverage | `@vitest/coverage-v8`; always set `coverage.include` |
| Monorepo / mixed env | `test.projects`; exclusive `include` globs; `--project name` |
| Real browser | Browser Mode + `@vitest/browser-playwright` |
| Slow suite | read the duration %; `vitest doctor` |

| CLI | |
|---|---|
| `vitest` | watch mode |
| `vitest run` | once |
| `vitest run --coverage` | with coverage |
| `vitest -t 'math > adds'` | by full name (5.0: joined with `' > '`) |
| `vitest -p unit` | one project |
| `vitest --repeats 10` | flake hunting |
| `DEBUG=vitest:projects vitest` | how projects resolved |

<a id="part-12-v4-to-v5"></a>

### 12.6 Vitest 4 → 5: what changed under you

From the [Vitest 5 migration guide](https://vitest.dev/guide/migration). These are the entries that change the behaviour of an ordinary Vite app's test suite:

| Vitest 4 | Vitest 5 |
|---|---|
| lower Node floor | **Node ≥ 22.12** and **Vite ≥ 6.4** required |
| `clearMocks: false` default | **`clearMocks: true`**: call history resets per test |
| `vi.mock` nested in `test()` warned | **throws** |
| `-t "math adds"` matched space-joined names | matches full name joined by `' > '` |
| inline projects needed `extends: true` | inherit root config by default; `extends: false` to opt out |
| unawaited `.resolves` / `.rejects` warned | **test fails** |
| `toThrow('')` matched only empty messages | matches any message; use `/^$/` for empty |
| `test.sequential` / `describe.sequential` | removed: `{ concurrent: false }` |
| top-level `bench()` | `test('…', ({ bench }) => …)` fixture |
| `vitest/coverage`, `vitest/reporters` imports | `vitest/node` |
| `vitest/environments`, `vitest/snapshot` | `vitest/runtime` |
| `html/index.html`, `.vitest-reports/`, `__screenshots__/` | all under `.vitest/` |
| config file searched in parent directories | not searched: run from the config's directory or pass `--config` |
| `VITEST_POOL_ID` / `VITEST_WORKER_ID` from 0 | start at **1** |
| Browser Mode locators loose, `toHaveTextContent` partial | strict / exact (`browser.locators.exact: false` to revert) |
| `@vitest/browser-webdriverio` | moved to community (`vitest-community/vitest-webdriverio`) |
| fake timers ignored `Temporal` | `Temporal.Now` follows the fake clock |

[↑ Back to top](#table-of-contents)
