# Bun — Testing & Bundling (Parts 11–12)

The two tools that usually arrive as separate ecosystems — Jest and esbuild/webpack — shipped inside the same binary that runs your code.

> **Spec:** this doc follows the shared spec in [`bun-mastery-guide.md`](./bun-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Mastery tiers (collapsed where a tier would be filler, always stated), **Try It** exercises with expected output, wrong-vs-right pairs, official docs cited inline. Written against **Bun v1.4.2**, verified **September 6, 2026**.
>
> **Prerequisite:** [Part 2 — Modules & Resolution](./bun-runtime-foundations.md#part-2--modules--resolution). Both the test runner and the bundler are the module system pointed at a different job, and mocking in particular only makes sense once resolution does.

---

## Table of Contents

- [Part 11 — `bun test`](#part-11--bun-test)
  - [Beginner: your first test](#p11-beginner)
  - [Working Knowledge: filtering, lifecycle, and matchers](#p11-working)
  - [Advanced: mocks, spies, and module mocking](#p11-advanced)
  - [Advanced: snapshots and coverage](#p11-coverage)
  - [Mastery: parallelism, isolation, and sharding](#p11-mastery)
  - [Part 11 cheat sheet](#part-11-cheat-sheet)
- [Part 12 — `bun build`](#part-12--bun-build)
  - [Beginner: bundle something](#p12-beginner)
  - [Working Knowledge: targets, formats, and the options that matter](#p12-working)
  - [Advanced: splitting, external, and sourcemaps](#p12-advanced)
  - [Advanced: standalone executables](#p12-compile)
  - [Mastery: plugins and build-time macros](#p12-mastery)
  - [Part 12 cheat sheet](#part-12-cheat-sheet)

---

## Part 11 — `bun test`

<a id="p11-beginner"></a>

### Beginner: your first test

```ts
// math.test.ts
import { test, expect, describe } from "bun:test";

describe("addition", () => {
  test("adds numbers", () => {
    expect(2 + 2).toBe(4);
  });
});
```

```bash
bun test
```

No `jest.config.js`, no `ts-jest`, no `babel.config.js`, no transform pipeline. TypeScript test files run directly.

Discovery is by filename ([test docs](https://bun.sh/docs/cli/test)) — any file matching:

```
*.test.{js,jsx,ts,tsx,mjs,cjs,mts,cts}
*_test.{…}
*.spec.{…}
*_spec.{…}
```

> **vs. Jest** — the API is deliberately Jest-compatible, so `describe`/`test`/`expect`/`beforeEach` and the common matchers work unchanged. There's also a `vi` alias exported from `bun:test` for Vitest muscle memory. The migration cost of an existing suite is usually the mocks, not the assertions.

<a id="p11-working"></a>

### Working Knowledge: filtering, lifecycle, and matchers

#### Running a subset

```bash
bun test                          # everything
bun test users                    # files whose path contains "users"
bun test ./src/api/users.test.ts  # one file
bun test -t "creates a user"      # by test NAME (regex)
bun test --watch                  # re-run on change
bun test --bail                   # stop at the first failure
bun test --bail=3                 # …or the third
bun test --changed                # only tests affected by your changes (1.4)
```

`-t` filters by test name; a bare argument filters by file path. Mixing those up is a two-minute confusion that this line prevents.

#### Lifecycle

```ts
import { beforeAll, beforeEach, afterEach, afterAll } from "bun:test";

beforeAll(async () => { /* once, before everything in this file */ });
beforeEach(() => { /* before each test */ });
afterEach(() => { /* after each test */ });
afterAll(() => { /* once, at the end */ });
```

For hooks shared across every file, use a preload rather than importing a setup module everywhere:

```toml
# bunfig.toml
[test]
preload = ["./test/setup.ts"]
```

#### Test qualifiers

```ts
test.skip("not yet", () => {});
test.todo("write this");
test.only("just this one", () => {});
test.failing("known bug — passes when it throws", () => {});
test.concurrent("runs alongside other concurrent tests", async () => {});
test.serial("never runs concurrently", () => {});

test("with options", { retry: 3, timeout: 10_000 }, () => {});

test.each([[1, 1, 2], [2, 2, 4]])("%i + %i = %i", (a, b, expected) => {
  expect(a + b).toBe(expected);
});
```

`test.each` with a table is how you delete a for-loop from a test file and get one named failure per case instead of one opaque one.

#### Matchers, and the two that get confused

```ts
expect(v).toBe(x);                 // Object.is — identity for objects
expect(v).toEqual(x);              // recursive structural equality
expect(v).toStrictEqual(x);        // …plus type/undefined-key strictness
expect(v).toMatchObject({ a: 1 }); // subset match
expect(fn).toThrow(/pattern/);
expect(p).resolves.toBe(x);
expect(p).rejects.toThrow();
expect(mock).toHaveBeenCalledWith(1, 2);
expect(v).toMatchSnapshot();
expectTypeOf<Foo>().toEqualTypeOf<Bar>();   // type-level assertions
```

```ts
// ❌ fails — two structurally identical objects are not the same object
expect({ a: 1 }).toBe({ a: 1 });

// ✅
expect({ a: 1 }).toEqual({ a: 1 });
```

`expectTypeOf` is worth flagging: it asserts on **types**, evaluated by your type checker, not at runtime. It only means anything if you also run `tsc --noEmit` — `bun test` alone will happily "pass" a type assertion that is false, because it strips types ([Part 1](./bun-runtime-foundations.md#p1-mastery)).

<a id="p11-advanced"></a>

### Advanced: mocks, spies, and module mocking

#### Function mocks

```ts
import { mock, spyOn, expect } from "bun:test";

const fn = mock((x: number) => x * 2);
fn(21);

expect(fn).toHaveBeenCalled();
expect(fn).toHaveBeenCalledTimes(1);
expect(fn).toHaveBeenCalledWith(21);
fn.mock.calls;      // [[21]]
fn.mock.results;    // [{ type: "return", value: 42 }]

fn.mockReturnValue(7);
fn.mockResolvedValue(7);
fn.mockRejectedValue(new Error("nope"));
fn.mockImplementation((x) => x + 1);
fn.mockClear();     // clear history, keep implementation
fn.mockReset();     // clear history AND implementation
fn.mockRestore();   // restore the original
```

`jest.fn()` and `vi.fn()` are aliases of `mock` ([mock docs](https://bun.sh/docs/test/mocks)).

#### Spies — observe without replacing

```ts
const spy = spyOn(logger, "warn");
doSomethingThatWarns();
expect(spy).toHaveBeenCalledTimes(1);
```

The difference from a mock: the real method still runs. Use a spy when the behaviour matters and you only want to assert it happened.

#### Module mocking, and its hoisting trap

```ts
import { mock } from "bun:test";

mock.module("./payments", () => ({
  charge: mock(async () => ({ ok: true })),
}));
```

**The gotcha that costs an afternoon** ([mock docs](https://bun.sh/docs/test/mocks)): if the module was already imported before `mock.module` ran, its **side effects have already happened**. Since ESM imports are hoisted above your statements, a `mock.module` call at the top of a test file still runs *after* that file's imports.

```ts
// ❌ ./db connected to a real database while the imports were evaluated
import { getUser } from "./service";     // → imports ./db → opens a connection
mock.module("./db", () => ({ query: mock() }));

// ✅ mock in a preload, so it wins before anything imports the real module
// bunfig.toml:  [test]  preload = ["./test/mocks.ts"]
```

```bash
bun test --preload ./test/mocks.ts
```

Two further documented behaviours: mocked ESM modules keep **live bindings**, so existing importers see the mock; and there is **no `__mocks__` directory auto-mocking** — if you're porting a Jest suite that relies on it, that convention has to be rewritten as explicit `mock.module` calls.

Reset between tests:

```ts
afterEach(() => {
  mock.restore();        // restore spied/mocked originals
  mock.clearAllMocks();  // clear call history, keep implementations
});
```

Note that `mock.restore()` does **not** undo `mock.module` — module mocks are for the process lifetime, which is another reason they belong in preload rather than inside a single test.

<a id="p11-coverage"></a>

### Advanced: snapshots and coverage

```ts
expect(renderInvoice(data)).toMatchSnapshot();
expect(user).toMatchInlineSnapshot();   // written back into the test file
```

```bash
bun test -u    # update snapshots after an intentional change
```

Snapshots are worth exactly as much as the review they get. A snapshot updated with `-u` without reading the diff is a test that asserts "the code does what the code does."

Coverage:

```bash
bun test --coverage
bun test --coverage --coverage-reporter=lcov --coverage-dir=coverage
```

```toml
[test]
coverage = true
coverageThreshold = 0.9          # fail under 90%
coverageReporter = ["text", "lcov"]
coverageSkipTestFiles = true
coveragePathIgnorePatterns = ["src/generated/**"]
```

`coverageThreshold` turns coverage from a number nobody looks at into a build failure, which is the only version of coverage that changes behaviour.

Reporters for CI:

```bash
bun test --reporter=junit --reporter-outfile=./junit.xml   # GitLab, Jenkins, etc.
bun test --dots                                            # compact
```

On GitHub Actions, `bun test` emits failure annotations automatically — failures appear inline on the diff with no extra configuration.

<a id="p11-mastery"></a>

### Mastery: parallelism, isolation, and sharding

Bun 1.4 changed what a large suite looks like ([Bun 1.4](https://bun.com/blog/bun-v1.4), [test docs](https://bun.sh/docs/cli/test)):

```bash
bun test --parallel                 # distribute test FILES across CPU cores
bun test --parallel --no-isolate    # …sharing global state between files
bun test --concurrent               # run tests within a file concurrently
bun test --max-concurrency 10       # cap that (default 20)
```

Three different axes, easy to conflate:

| Flag | Unit of concurrency | Isolation |
|---|---|---|
| `--parallel` | test **files**, across worker processes | fresh global per file (default) |
| `--concurrent` | **tests**, inside one file | shared module state |
| `test.concurrent(…)` | opt-in per test | shared module state |

`--isolate` runs each test file in a fresh JavaScript global object within the same process. That is what makes `--parallel` safe by default: a file that sets `globalThis.config` cannot leak into another. Opting out with `--no-isolate` is faster and is exactly how a suite develops order-dependent flakiness.

#### Finding flaky and order-dependent tests

```bash
bun test --randomize            # random order — surfaces order dependence
bun test --randomize --seed 42  # reproduce that exact order
bun test --rerun-each 10        # run each test 10× — surfaces flakiness
bun test --retry 2              # retry failures (a band-aid, not a fix)
```

The `--randomize` + `--seed` pair is the good one. A suite that passes in file order and fails randomly has hidden shared state, and the seed makes that failure reproducible rather than folklore.

#### Sharding across CI machines

```bash
# machine 1 of 4
bun test --parallel --shard=1/4 --update-timings
```

`--update-timings` records how long each test took so subsequent shards can be balanced by duration rather than by file count — the difference between four shards finishing together and three finishing while one carries the slow integration tests.

```yaml
# .github/workflows/test.yml
strategy:
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - uses: oven-sh/setup-bun@v2
  - run: bun ci
  - run: bun test --parallel --shard=${{ matrix.shard }}/4
```

**Try It — catch an order-dependent test.**

```ts
// leak.test.ts
import { test, expect } from "bun:test";

let counter = 0;
test("a increments", () => { counter++; expect(counter).toBe(1); });
test("b sees one",   () => { expect(counter).toBe(1); });
```

```bash
bun test leak.test.ts                    # passes
bun test leak.test.ts --randomize        # run a few times
```

Expected: in file order both pass; under `--randomize`, runs where `b` executes first fail with `expected 1, received 0`. The bug was always there — the default ordering was hiding it. Note the seed printed on failure and pass it back with `--seed` to replay that exact order.

<a id="part-11-cheat-sheet"></a>

### Part 11 cheat sheet

| Want | How |
|---|---|
| Run all tests | `bun test` |
| One file / by path | `bun test path/frag` |
| By test name | `bun test -t "regex"` |
| Watch | `bun test --watch` |
| Stop at first failure | `bun test --bail` |
| Only affected tests | `bun test --changed` |
| Shared setup | `[test] preload = ["./setup.ts"]` |
| Mock a function | `mock(fn)` / `jest.fn()` / `vi.fn()` |
| Spy without replacing | `spyOn(obj, "method")` |
| Mock a module | `mock.module(spec, factory)` — **in a preload** |
| Reset mocks | `mock.restore()` / `mock.clearAllMocks()` |
| Snapshots | `toMatchSnapshot()`, update with `-u` |
| Coverage + gate | `--coverage`, `coverageThreshold = 0.9` |
| CI report | `--reporter=junit --reporter-outfile=x.xml` |
| Use all cores | `bun test --parallel` |
| Fresh global per file | `--isolate` (default with `--parallel`) |
| Find order dependence | `--randomize`, replay with `--seed N` |
| Find flakes | `--rerun-each 10` |
| Split across CI machines | `--shard=i/n --update-timings` |

[↑ Back to top](#table-of-contents)

---

## Part 12 — `bun build`

<a id="p12-beginner"></a>

### Beginner: bundle something

Bundling combines many modules into few files, for shipping. Bun does it with a native bundler ([bundler docs](https://bun.sh/docs/bundler)):

```bash
bun build ./src/index.ts --outdir ./dist
```

```ts
// or from JavaScript
const result = await Bun.build({
  entrypoints: ["./src/index.ts"],
  outdir: "./dist",
});

if (!result.success) {
  for (const log of result.logs) console.error(log);
  process.exit(1);
}
```

**Note what you do *not* need this for.** You do not bundle to run TypeScript — Bun already does that. Bundle when you are shipping to a browser, publishing a library, producing a small deploy artifact, or building a standalone binary.

<a id="p12-working"></a>

### Working Knowledge: targets, formats, and the options that matter

#### `target` — who will run this?

| Target | Meaning |
|---|---|
| `"browser"` (default) | No Node built-ins; browser-appropriate resolution conditions |
| `"bun"` | Runs under Bun. Keeps `node:`/`bun:` imports external; can emit bytecode |
| `"node"` | Runs under Node. Keeps `node:` built-ins external |

Getting this wrong is the most common bundler error: with the default `browser` target, an import of `node:fs` is a build failure, because browsers have no filesystem. If you are building a server, say so.

```bash
# ❌ server code with the default target — fails on node:fs
bun build ./server.ts --outdir ./dist

# ✅
bun build ./server.ts --outdir ./dist --target bun
```

#### `format`

`"esm"` (default), `"cjs"`, `"iife"`. ESM unless you're publishing for older consumers or dropping a `<script>` tag straight into a page.

#### The everyday options

```ts
await Bun.build({
  entrypoints: ["./src/index.ts"],
  outdir: "./dist",
  target: "bun",
  format: "esm",
  minify: true,                   // or { whitespace, identifiers, syntax }
  sourcemap: "linked",            // "none" | "linked" | "inline" | "external"
  splitting: true,
  external: ["react"],
  drop: ["console", "debugger"],
  define: { "process.env.NODE_ENV": '"production"' },
  env: "PUBLIC_*",                // inline only env vars with this prefix
  naming: "[dir]/[name]-[hash].[ext]",
  publicPath: "https://cdn.example.com/",
  metafile: true,
});
```

`env: "PUBLIC_*"` deserves emphasis. Inlining environment variables into a browser bundle is how secrets leak — the value ends up in a file anyone can read. The prefix form makes the safe subset explicit instead of relying on you to remember which vars are secret.

```ts
// ❌ every env var, including DATABASE_URL and API keys, baked into client JS
await Bun.build({ entrypoints: ["./app.ts"], env: "inline" });

// ✅ only what you deliberately prefixed as public
await Bun.build({ entrypoints: ["./app.ts"], env: "PUBLIC_*" });
```

#### File types that just work

`.ts` `.tsx` `.js` `.jsx` `.mjs` `.cjs`, plus `.json` `.jsonc` `.toml` `.yaml` `.css` `.txt` `.html`, with unrecognised extensions copied through as assets.

<a id="p12-advanced"></a>

### Advanced: splitting, external, and sourcemaps

#### Code splitting

```ts
await Bun.build({
  entrypoints: ["./admin.ts", "./app.ts"],
  outdir: "./dist",
  splitting: true,
});
```

```
  WITHOUT splitting                WITH splitting

  app.js    [app  + shared]        app.js    [app]   ─┐
  admin.js  [admin + shared]       admin.js  [admin] ─┼─► chunk-abc.js [shared]
            ↑ shared code                              ┘
              duplicated                    downloaded once, cached once
```

Splitting also enables real lazy loading: a dynamic `import()` becomes a separate chunk fetched on demand.

#### `external` — deliberately not bundling something

```ts
external: ["react", "react-dom"]   // leave the import in the output
external: ["*"]                    // bundle nothing — dependencies stay external
```

The library-publishing rule: **`external` your peer dependencies.** If you bundle React into your component library, an app using your library ends up with two Reacts, and hooks break with an error that names neither your library nor the real cause.

```ts
// ❌ a component library that inlines React
await Bun.build({ entrypoints: ["./index.tsx"], outdir: "./dist" });

// ✅ leave peers to the consumer
await Bun.build({ entrypoints: ["./index.tsx"], outdir: "./dist", external: ["react", "react-dom"] });
```

For a server bundle the calculus flips: `--target bun` already externalises built-ins, and bundling your dependencies is often the point (one file to deploy).

#### Sourcemaps

| Value | Result |
|---|---|
| `"none"` | No map |
| `"linked"` | Separate `.map` + a comment pointing at it |
| `"inline"` | Base64 map inside the bundle |
| `"external"` | Separate `.map`, **no** comment |

`"linked"` for development. For production, `"external"` plus uploading the map to your error tracker gives you readable stack traces without publishing your source to every visitor.

<a id="p12-compile"></a>

### Advanced: standalone executables

```bash
bun build ./cli.ts --compile --outfile mycli
./mycli
```

This produces a **single binary with the Bun runtime embedded** — no Node, no `node_modules`, no `bun` on the target machine. For CLI distribution this is the feature that makes Bun worth adopting on its own.

```bash
# smaller and faster to start
bun build ./cli.ts --compile --minify --bytecode --outfile mycli
```

`--bytecode` precompiles to JavaScriptCore bytecode, cutting startup time further; it requires `--target bun`, which `--compile` implies.

Assets can be embedded by importing them, so a binary can ship its own templates or migrations:

```ts
import schema from "./schema.sql" with { type: "text" };
```

> This is why `node:sea` is listed as **not implemented** in Bun's compat table ([Node compat](https://bun.sh/docs/runtime/nodejs-apis)) — `--compile` predates it and does more. If a migration checklist flags missing `node:sea` support, that's the answer.

**Try It — ship a binary.**

```ts
// cli.ts
const [, , name = "world"] = Bun.argv;
console.log(`hello, ${name}`);
```

```bash
bun build ./cli.ts --compile --outfile hello
./hello Sam        # → hello, Sam
ls -lh hello       # tens of megabytes — that's the embedded runtime
```

Expected: a large file (the runtime is in there) that runs on a machine with no JavaScript toolchain installed at all. That size is the trade: one dependency-free artifact, at ~50–100 MB.

<a id="p12-mastery"></a>

### Mastery: plugins and build-time macros

#### Bundler plugins

The plugin API is the same one the runtime uses ([Part 3](./bun-runtime-foundations.md#p3-mastery)), with extra hooks at build time ([plugin docs](https://bun.sh/docs/runtime/plugins)):

```ts
import type { BunPlugin } from "bun";

const envPlugin: BunPlugin = {
  name: "virtual env module",
  setup(build) {
    build.onStart(() => console.log("build started"));

    build.onResolve({ filter: /^env$/ }, () => ({ path: "env", namespace: "env" }));

    build.onLoad({ filter: /.*/, namespace: "env" }, () => ({
      contents: `export default ${JSON.stringify({ MODE: "production" })}`,
      loader: "js",
    }));
  },
};

await Bun.build({ entrypoints: ["./app.ts"], outdir: "./out", plugins: [envPlugin] });
```

Hooks available: `onStart`, `onResolve`, `onLoad`, and `onBeforeParse` — the last implemented as a **NAPI native module** (Rust, C++) for transforms hot enough that a JavaScript callback per file would dominate the build.

Because a plugin using only `onResolve`/`onLoad` works in both contexts, one YAML or GraphQL loader can serve `bun run` in development and `bun build` in CI, which is a real reduction in configuration surface compared with maintaining a Vite plugin plus a Jest transform plus a webpack loader.

#### Macros — run code at build time

```ts
import { getVersion } from "./macro.ts" with { type: "macro" };

const version = getVersion();   // executed during the BUILD; the result is inlined
```

The function runs in the bundler and its **return value is inlined into the output** — no call remains at runtime. Good uses: baking in a git SHA, a build timestamp, or a list of files resolved by a glob at build time. The constraint is that a macro's return value must be serialisable, and it must not depend on runtime state, because there is none yet.

#### Inspecting the output

```ts
const result = await Bun.build({ entrypoints: ["./app.ts"], outdir: "./dist", metafile: true });

for (const output of result.outputs) {
  console.log(output.path, output.size, output.kind);
}
```

`metafile: true` emits the module graph, which is how you answer "why is my bundle 4 MB?" — usually a moment-timezone-shaped answer.

<a id="part-12-cheat-sheet"></a>

### Part 12 cheat sheet

| Want | How |
|---|---|
| Bundle | `bun build ./src/index.ts --outdir ./dist` |
| From JS | `await Bun.build({ entrypoints, outdir })` |
| Server bundle | `--target bun` (or `node`) |
| Browser bundle | `--target browser` (default) |
| Shrink | `--minify` |
| Debuggable | `--sourcemap=linked` |
| Production maps | `sourcemap: "external"` + upload to error tracking |
| Share code between entries | `splitting: true` |
| Don't bundle peers | `external: ["react"]` |
| Strip logging | `drop: ["console", "debugger"]` |
| Safe env inlining | `env: "PUBLIC_*"` |
| Hashed filenames | `naming: "[dir]/[name]-[hash].[ext]"` |
| Single-file executable | `--compile --outfile mycli` |
| Faster startup binary | `--compile --bytecode --minify` |
| Custom file types | a plugin with `onResolve`/`onLoad` |
| Compute at build time | `import { f } from "./m" with { type: "macro" }` |
| Why is it so big? | `metafile: true`, inspect `result.outputs` |

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 13 — Node Compat & Internals](./bun-node-compat-internals.md) · **Up:** [Overview](./bun-mastery-guide.md)

*Written against Bun v1.4.2 · Verified September 6, 2026*
