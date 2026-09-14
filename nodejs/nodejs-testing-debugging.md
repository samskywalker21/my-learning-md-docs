# Node.js — Testing & Debugging (Part 12)

Node's built-in test runner — which has been stable for several release lines and is still described as "not ready" in a remarkable number of places — and the inspector, which turns "add a `console.log` and re-run" into setting a breakpoint.

> **Spec:** this doc follows the shared spec in [`nodejs-mastery-guide.md`](./nodejs-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Mastery tiers (collapsed where a tier would be filler, always stated), **Try It** exercises with expected output, wrong-vs-right pairs, official docs cited inline. Written against **Node.js 24 LTS**, verified **September 14, 2026**.
>
> **Prerequisite:** [Part 3 — Async Patterns](./nodejs-runtime-foundations.md#part-3--async-patterns-in-node) and [Part 4 — Modules](./nodejs-modules-packages.md#part-4--modules-cjs-esm-and-typescript). Test isolation is a module-loading story, and most test flakiness is an async story.

---

## Table of Contents

- [Part 12 — `node:test` & Debugging](#part-12--nodetest--debugging)
  - [Beginner: a test file and `node --test`](#p12-beginner)
  - [Working Knowledge: structure, hooks, assertions, filtering](#p12-working)
  - [Advanced: mocking, timers, snapshots, coverage](#p12-advanced)
  - [Mastery: isolation, concurrency, sharding, and the inspector](#p12-mastery)
  - [Part 12 cheat sheet](#part-12-cheat-sheet)

---

## Part 12 — `node:test` & Debugging

> **Flagged change — "use Jest, Node's runner isn't ready" is outdated.** `node:test` has been **stable since Node 20.0.0** ([test docs](https://nodejs.org/docs/latest-v24.x/api/test.html)). Snapshot testing came out of experimental in 23.4.0. What is still experimental on Node 24 is narrower than most comparisons claim: **coverage** (`--experimental-test-coverage`) and **module mocking** (`--experimental-test-module-mocks`). If a project already runs on Vitest or Jest there is no urgent reason to move; for a new Node project, the built-in runner is a legitimate default, not a compromise.

<a id="p12-beginner"></a>

### Beginner: a test file and `node --test`

```js
// math.test.mjs
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { add } from './math.mjs';

test('adds two numbers', () => {
  assert.equal(add(2, 3), 5);
});

test('async works the same way', async () => {
  const result = await Promise.resolve(add(1, 1));
  assert.equal(result, 2);
});
```

```bash
node --test
```

Expected output: a `spec`-style report with a `✔` beside each test name, followed by a summary block listing `tests 2`, `pass 2`, `fail 0`. That's the whole setup — no config file, no dependency, no `package.json` entry required.

A test fails if its function throws, returns a rejected promise, or (for callback-style tests) passes a truthy first argument to `done`.

**Always import `node:assert/strict`**, not `node:assert`:

```js
// ✗ wrong — legacy mode: equal() uses ==, so this PASSES
import assert from 'node:assert';
assert.equal('1', 1);

// ✓ right — strict mode: equal() uses ===, so this fails, as it should
import assert from 'node:assert/strict';
assert.equal('1', 1);
```

The legacy mode exists for backwards compatibility and produces tests that pass when they shouldn't. There is no reason to use it in new code.

#### Which files does `node --test` find?

With no arguments it searches the current directory for these patterns, skipping `node_modules` ([test docs](https://nodejs.org/docs/latest-v24.x/api/test.html)):

```
**/*.test.{cjs,mjs,js}      **/*-test.{cjs,mjs,js}      **/*_test.{cjs,mjs,js}
**/test-*.{cjs,mjs,js}      **/test.{cjs,mjs,js}        **/test/**/*.{cjs,mjs,js}
```

and, because type stripping is on by default in Node 24, the same patterns with `.ts`, `.mts` and `.cts` extensions. Note what is **not** in the list: `*.spec.js`. Projects migrating from Jest or Mocha regularly run `node --test`, see "tests 0", and conclude the runner is broken. Pass the glob explicitly:

```bash
node --test "src/**/*.spec.js"     # quote it, so Node expands it rather than your shell
```

<a id="p12-working"></a>

### Working Knowledge: structure, hooks, assertions, filtering

#### `describe`/`it` or `test` — they're the same thing

`describe` and `it` are aliases for `suite` and `test`. Use whichever reads better; mixing is fine:

```js
import { describe, it, before, after, beforeEach } from 'node:test';
import assert from 'node:assert/strict';

describe('UserService', () => {
  let db;
  before(async () => { db = await createTestDb(); });
  after(async () => { await db.close(); });
  beforeEach(async () => { await db.truncate(); });

  it('creates a user', async () => {
    const user = await new UserService(db).create({ email: 'a@b.c' });
    assert.ok(user.id);
  });

  it('rejects duplicate emails', async () => {
    const svc = new UserService(db);
    await svc.create({ email: 'a@b.c' });
    await assert.rejects(svc.create({ email: 'a@b.c' }), { code: 'DUPLICATE' });
  });
});
```

For per-test cleanup, `t.after()` on the test context is usually cleaner than a suite-level hook, because the teardown sits beside the setup it undoes:

```js
it('writes a temp file', async (t) => {
  const dir = await mkdtemp(join(tmpdir(), 'test-'));
  t.after(() => rm(dir, { recursive: true, force: true }));
  // …
});
```

Multiple `after()` callbacks run last-registered-first, and an error inside one fails the test.

#### The assertions you actually use

| Assertion | Checks |
|---|---|
| `assert.equal(a, b)` | `a === b` (strict mode) |
| `assert.deepEqual(a, b)` | Structural equality, recursively (strict mode: prototypes and types too) |
| `assert.ok(v)` | Truthy |
| `assert.throws(fn, expected)` | `fn` throws something matching `expected` |
| `assert.rejects(promiseOrFn, expected)` | Rejects with something matching `expected` — **must be awaited** |
| `assert.match(str, /re/)` | Regex match |
| `assert.partialDeepStrictEqual(a, b)` | `a` contains at least the structure of `b` |

`expected` in `throws`/`rejects` can be an error class, a regex against the message, or an object whose properties must match — the object form is the most precise and the most useful:

```js
await assert.rejects(fetchUser(-1), { name: 'ValidationError', code: 'INVALID_ID' });
```

The `rejects` trap is a classic, and it produces a test that can never fail:

```js
// ✗ wrong — not awaited. The test finishes before the assertion runs; always passes.
assert.rejects(svc.create(dup));

// ✓ right
await assert.rejects(svc.create(dup));
```

#### Subtests

A test can create child tests through its context. Since Node 24, the runner **waits for subtests automatically** — on Node 22 and earlier you had to `await` each `t.test()` yourself or it was cancelled ([Node 24 release notes](https://nodejs.org/en/blog/release/v24.0.0)):

```js
test('parser', async (t) => {
  await t.test('handles empty input', () => { /* … */ });   // await still reads clearly
  await t.test('handles unicode', () => { /* … */ });
});
```

Keeping the `await` is still worth it: it makes execution order explicit and keeps the code correct if it's ever run on an older line.

#### Running a subset

```bash
node --test --test-name-pattern="duplicate"           # regex against test names
node --test --test-name-pattern="/^UserService/i"     # with flags
node --test src/users/                                 # one directory
```

```js
it.only('the one I am debugging', () => {});
it.skip('broken on Windows', () => {});
it.todo('handle leap seconds');     // runs, but a failure doesn't fail the suite
```

`.only` needs the `--test-only` flag to take effect when process isolation is on (the default) — without it, `.only` is ignored and everything runs, which is confusing exactly once.

#### Watch mode

```bash
node --test --watch
```

Re-runs affected test files on change. The single fastest improvement to a test feedback loop, and it's already installed.

> **vs. Bun** — `bun test` uses a Jest-compatible `expect()` API with matchers like `toEqual` and `toHaveBeenCalledWith`, where `node:test` uses `node:assert`. Test files are not portable between them without an adapter, and that API difference — not speed — is usually what decides which to use. See [`bun-testing-bundling.md`](../bun/bun-testing-bundling.md).

<a id="p12-advanced"></a>

### Advanced: mocking, timers, snapshots, coverage

#### Function and method mocks

The test context carries a `mock` object whose mocks are **restored automatically when the test ends** ([test docs](https://nodejs.org/docs/latest-v24.x/api/test.html)) — the most common source of cross-test pollution in other runners is handled for you:

```js
test('sends a welcome email', async (t) => {
  const send = t.mock.method(mailer, 'send', async () => ({ id: 'msg_1' }));

  await signup({ email: 'a@b.c' });

  assert.equal(send.mock.callCount(), 1);
  assert.deepEqual(send.mock.calls[0].arguments, [{ to: 'a@b.c', template: 'welcome' }]);
});
```

```js
const fn = t.mock.fn((x) => x * 2);   // a standalone spy with an implementation
fn(3);
fn.mock.calls[0].result;               // 6
fn.mock.mockImplementationOnce(() => -1);
```

Prefer `t.mock` over the top-level `mock` import. The top-level one is not scoped to a test, so you must call `mock.restoreAll()` yourself — and forgetting to is exactly the bug `t.mock` exists to prevent.

#### Fake timers

```js
test('retries after a delay', (t) => {
  t.mock.timers.enable({ apis: ['setTimeout', 'Date'] });

  const cb = t.mock.fn();
  scheduleRetry(cb, 30_000);

  t.mock.timers.tick(29_999);
  assert.equal(cb.mock.callCount(), 0);
  t.mock.timers.tick(1);
  assert.equal(cb.mock.callCount(), 1);
});
```

A 30-second wait tested in a millisecond. Enable only the `apis` you need: mocking `Date` alongside `setTimeout` is common, but faking APIs your code under test doesn't use can break libraries that do.

#### Module mocks — still experimental

Replacing an entire import requires a flag on Node 24:

```bash
node --test --experimental-test-module-mocks
```

```js
test('uses the mocked config', async (t) => {
  t.mock.module('./config.mjs', { namedExports: { apiUrl: 'http://fake' } });
  const { client } = await import('./client.mjs');   // must import AFTER mocking
  // …
});
```

The dynamic `import()` after the mock is not optional: static imports are resolved before any test code runs. In practice, **dependency injection is usually the better design** — a function that accepts its collaborators needs no module mocking at all, and it doesn't depend on an experimental flag.

#### Snapshots

Stable since 23.4.0:

```js
test('renders the invoice', (t) => {
  t.assert.snapshot(renderInvoice(fixture));
});
```

```bash
node --test --test-update-snapshots   # first run: writes math.test.mjs.snapshot
node --test                            # later runs: compares
```

Commit the `.snapshot` files. The failure mode of snapshots is universal rather than Node-specific: they're easy to update without reading, and a snapshot someone blindly regenerated tests nothing. Keep them small and specific; a 400-line snapshot of an entire page is a change detector, not a test.

#### Coverage

```bash
node --test --experimental-test-coverage
node --test --experimental-test-coverage \
  --test-coverage-lines=80 --test-coverage-branches=75 \
  --test-coverage-exclude="**/*.test.mjs"
```

Below a threshold, the process exits with code `1` — which is what makes it enforceable in CI. Coverage is still marked **experimental** on Node 24, so re-check its status before building a pipeline gate around the exact flags.

To exclude code that genuinely can't be exercised:

```js
/* node:coverage ignore next */
if (process.platform === 'win32') { /* … */ }
```

**Try It — make coverage find something.**

```js
// grade.mjs
export function grade(score) {
  if (score >= 90) return 'A';
  if (score >= 80) return 'B';
  return 'C';
}
```

```js
// grade.test.mjs
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { grade } from './grade.mjs';

test('an A', () => assert.equal(grade(95), 'A'));
```

```bash
node --test --experimental-test-coverage
```

Expected output: the test passes, and the coverage table lists `grade.mjs` with **line coverage below 100%** and its *uncovered lines* column naming the `'B'` and `'C'` branches. Add a test for `grade(85)` and watch one of them disappear from that column.

<a id="p12-mastery"></a>

### Mastery: isolation, concurrency, sharding, and the inspector

#### How the runner actually executes files

```
   node --test
        │
        ├── discovers files
        │
        ├── isolation: 'process' (default)
        │     ├── child process ── a.test.mjs     ← own module cache, own globals
        │     ├── child process ── b.test.mjs     ← one at a time by default
        │     └── child process ── c.test.mjs
        │
        └── reports aggregated results
```

**Every test file runs in its own child process by default**, and files run **one at a time** ([test docs](https://nodejs.org/docs/latest-v24.x/api/test.html)). That default buys you correctness — no module state leaks between files — and costs you speed. The two dials:

```bash
node --test --test-concurrency=4       # run 4 files in parallel
node --test --test-isolation=none      # all files in ONE process
```

`--test-isolation=none` is much faster for suites with many small files, because process startup dominates. It also lets module-level state leak between files, so it's only safe for suites that were written without shared singletons. It's also the setting to reach for when you need to attach a debugger to "the" test process.

Within a file, tests run sequentially unless you opt in:

```js
describe('independent HTTP checks', { concurrency: 5 }, () => { /* … */ });
```

#### Sharding across CI machines

```bash
# machine 1            # machine 2            # machine 3
node --test \          node --test \          node --test \
  --test-shard=1/3       --test-shard=2/3       --test-shard=3/3
```

Built in, and incompatible with `--watch`. Combined with a CI matrix, it's how a slow suite becomes a fast one without touching a test.

#### The test that never exits

A suite that passes but whose process hangs is almost always a handle left open — a server not closed, a pool not ended, an interval not cleared. It's the [Part 2 handle-lifecycle rule](./nodejs-runtime-foundations.md#p2-mastery) again:

```bash
# ✗ papering over it
node --test --test-force-exit

# ✓ finding it — this lists every handle keeping the loop alive
node --test --test-isolation=none --import ./why-open.mjs
```

```js
// why-open.mjs
import { after } from 'node:test';
after(() => setTimeout(() => console.log(process.getActiveResourcesInfo()), 100).unref());
```

`--test-force-exit` is legitimate in CI as a backstop, but using it *instead of* finding the leak means your production code likely leaks the same handle on shutdown.

#### Debugging with the inspector

Node embeds the V8 inspector — the same protocol Chrome DevTools uses:

```bash
node --inspect app.js          # start normally, accept a debugger on 127.0.0.1:9229
node --inspect-brk app.js      # pause before the first line of user code
node --inspect-wait app.js     # wait for a debugger to attach, then run
```

Then open `chrome://inspect` in Chrome and click **inspect** under your process. You get breakpoints, the call stack, scope inspection, a console evaluating inside the paused frame, and the CPU and memory profilers from [Part 14](./nodejs-production-diagnostics.md#part-14--diagnostics).

From code, `debugger;` pauses execution whenever an inspector is attached, and is a no-op otherwise.

To debug a test:

```bash
node --test --test-isolation=none --inspect-brk
```

Isolation must be off, or the debugger attaches to the parent runner process rather than the child running your test — the second-most-common reason a breakpoint "doesn't hit". (The most common is a source map problem, which `--enable-source-maps` fixes.)

In VS Code, the **JavaScript Debug Terminal** (Command Palette → *Debug: JavaScript Debug Terminal*) auto-attaches to any `node` process started from it, with no flags at all — including test runs, `--watch` restarts and child processes. For day-to-day debugging it is less friction than any of the flags above.

**The security rule:** `--inspect` binds to `127.0.0.1` by default for a reason. Anyone who can reach the inspector port can execute arbitrary code in your process.

```bash
# ✗ wrong — remote code execution for anyone who can reach port 9229
node --inspect=0.0.0.0:9229 app.js

# ✓ right — keep it on localhost, and tunnel to it
ssh -L 9229:127.0.0.1:9229 prod-box
```

Never expose the inspector port from a container or VM to a network. If you need to debug a remote process, forward the port over SSH.

<a id="part-12-cheat-sheet"></a>

### Part 12 cheat sheet

| Want | How |
|---|---|
| Run all tests | `node --test` |
| Run `.spec.js` files | `node --test "**/*.spec.js"` (quoted) |
| Watch | `node --test --watch` |
| Filter by name | `--test-name-pattern="regex"` |
| Focus one test | `it.only(…)` + `--test-only` |
| Assertions | `import assert from 'node:assert/strict'` — always `/strict` |
| Assert a rejection | `await assert.rejects(p, { code: 'X' })` — **await it** |
| Per-test cleanup | `t.after(() => …)` |
| Spy on a method | `t.mock.method(obj, 'name', impl)` — auto-restored |
| Standalone spy | `t.mock.fn(impl)` |
| Fake timers | `t.mock.timers.enable({ apis: ['setTimeout'] })` + `.tick(ms)` |
| Mock a module | `t.mock.module(…)` + `--experimental-test-module-mocks` *(experimental)* |
| Snapshot | `t.assert.snapshot(v)`; update with `--test-update-snapshots` |
| Coverage | `--experimental-test-coverage` *(experimental)* |
| Enforce coverage | `--test-coverage-lines=80` |
| Parallel files | `--test-concurrency=4` |
| One process, faster | `--test-isolation=none` |
| Split across CI machines | `--test-shard=1/3` |
| What's keeping it alive? | `process.getActiveResourcesInfo()` |
| Debug | `node --inspect-brk app.js` → `chrome://inspect` |
| Debug a test | `node --test --test-isolation=none --inspect-brk` |
| Zero-config debugging | VS Code's JavaScript Debug Terminal |

**Gotchas:** `*.spec.js` isn't in the default globs · `node:assert` without `/strict` uses `==` · an un-awaited `assert.rejects` always passes · `.only` needs `--test-only` · module mocks need a flag and a dynamic import · breakpoints in tests need `--test-isolation=none` · `--inspect=0.0.0.0` is remote code execution.

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 13 — Production Node](./nodejs-production-diagnostics.md) · **Previous:** [Parts 10–11](./nodejs-http-networking.md) · **Up:** [Overview](./nodejs-mastery-guide.md)

*Written against Node.js 24 LTS · Verified September 14, 2026*
