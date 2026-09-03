# Mastering Bun — A Hands-On Reference Guide

## About This Document

- **Framing:** Curiosity-driven. No upcoming task is driving this — the goal is durable, hands-on understanding of Bun itself. Theory ("why it's built this way") only shows up where it's needed to explain a mechanism (native TS stripping, the shell's security model, SQLite binding), not as standalone history.
- **Style:** A guided tutorial — a sequence of runnable examples and exercises meant to be typed into a terminal or file, in order, matching [nodejs/nodejs-mastery-guide.md](../nodejs/nodejs-mastery-guide.md) and [javascript/javascript-mastery-guide.md](../javascript/javascript-mastery-guide.md)'s format.
- **Depth tiers:** Every major Part is structured as four progressive, explicitly labeled tiers — **Beginner → Working Knowledge → Advanced → Mastery** — collapsed where they'd be trivial or redundant for that topic (this happens most in [Part 3 — Package Manager](#4-part-3--package-manager) and [Part 6 — SQLite & FFI](#7-part-6--sqlite--ffi), least in [Part 5 — HTTP & WebSockets](#6-part-5--http--websockets) and [Part 9 — Node.js/npm Compatibility](#10-part-9--nodejsnpm-compatibility), where Bun's real "aha" moments and sharpest edges live).
- **Scope:** Full breadth of Bun — runtime + built-in APIs, package manager, bundler, test runner, and the Node.js/npm compatibility layer. Not covered: framework-specific integration (Next.js, Elysia), deployment/infra, `Bun.build`'s HTML-import full-stack dev server (mentioned only in passing), and the newer 1.4-era native APIs (`Bun.WebView`, `Bun.Image`, `Bun.markdown`, `Bun.cron`, `Bun.Terminal`) beyond a brief orientation callout.
- **Assumed background:** None assumed for Node.js/npm — every Bun concept is explained from scratch. Solid core JavaScript (async/await, ES modules, promises) is assumed — see [javascript/javascript-mastery-guide.md](../javascript/javascript-mastery-guide.md).
- **Sourcing standard:** [bun.com/docs](https://bun.com/docs) is the primary source, cited inline. Stack Overflow and the Bun blog are used for real-world gotchas and version-history context. Checked as of September 2026 against **Bun 1.4** (August 2026), which rewrote Bun's core from Zig to Rust — flagged explicitly in [Part 1](#2-part-1--orientation--why-bun-is-fast) since older articles describing Bun as "written in Zig" are now only partially accurate. `bun.lock` (text-based) has been the default lockfile since Bun 1.2, replacing the old binary `bun.lockb` — flagged in [Part 3](#4-part-3--package-manager).
- **Update instructions for future-me:** If asked to update this later — match this exact structure (numbered Parts, each with tier subheadings), keep the wrong-vs-right snippet pairs for gotchas, keep the TOC + back-to-top links, and prefer adding new Parts/tiers over reverting to freeform prose. Collapsing tiers that feel forced for a given topic is expected, not a deviation to "fix." Re-verify the current Bun version and any newly-stabilized APIs before editing version-specific claims — Bun ships fast and un-flags features (routing, the shell, SQL clients) out of experimental status frequently.

---

## Table of Contents

1. [Orientation — Why Bun Is Fast](#2-part-1--orientation--why-bun-is-fast)
2. [Running Code — CLI, Native TS/JSX, Watch Mode](#3-part-2--running-code--cli-native-tsjsx-watch-mode)
3. [Package Manager](#4-part-3--package-manager)
4. [Runtime APIs I — File I/O & the Bun Shell](#5-part-4--runtime-apis-i--file-io--the-bun-shell)
5. [HTTP & WebSockets](#6-part-5--http--websockets)
6. [SQLite & FFI](#7-part-6--sqlite--ffi)
7. [Bundler](#8-part-7--bundler)
8. [Test Runner](#9-part-8--test-runner)
9. [Node.js/npm Compatibility](#10-part-9--nodejsnpm-compatibility)
10. [Cheat Sheets](#11-cheat-sheets)
11. [Suggested Learning Order](#12-suggested-learning-order)
12. [Quick Self-Check](#13-quick-self-check)

---

## 1. Part 1 — Orientation & Why Bun Is Fast

**What it is.** Bun is an all-in-one JavaScript/TypeScript toolkit: a **runtime** (replaces `node`), a **package manager** (replaces `npm`/`yarn`/`pnpm`), a **bundler** (replaces `esbuild`/`webpack` for many use cases), and a **test runner** (replaces `jest`/`vitest`) — one binary, no separate installs. See [bun.com](https://bun.com/).

**Why it matters.** The single-binary design isn't just convenience — it's why Bun is fast. `npm install`, `tsc`/Babel transpilation, and test execution are each separate Node processes with their own startup cost and JS engine warm-up; Bun does all of it inside one process using a shared native core.

```
┌────────────────────────────────────────────────────────┐
│                      bun (one binary)                  │
├───────────┬───────────┬────────────────┬───────────────┤
│  Runtime  │  Package  │    Bundler     │  Test Runner  │
│ (bun run, │  Manager  │  (bun build)   │  (bun test)   │
│  bun x.ts)│(bun install)│               │               │
├───────────┴───────────┴────────────────┴───────────────┤
│         JavaScriptCore (JS engine, from WebKit)          │
│         Rust core (was Zig through Bun 1.3)              │
└────────────────────────────────────────────────────────┘
```

### Beginner

Install Bun (not via npm — Bun replaces the need for a separate Node install for most workflows):

```bash
# macOS / Linux
curl -fsSL https://bun.com/install | bash

# Windows (PowerShell)
powershell -c "irm bun.com/install.ps1 | iex"

bun --version
```

**Real Scenario — try it now.** Confirm the install and peek at what's bundled:

```bash
$ bun --version
1.4.0
$ bun --revision      # exact git commit Bun was built from — useful when filing bugs
```

### Working Knowledge

- Bun uses **JavaScriptCore** (Safari/WebKit's engine), not V8 — chosen for faster startup and lower memory than V8, at the cost of a smaller pool of V8-specific tooling (some Chrome DevTools Node-debugging flows assume V8). See the [Bun FAQ](https://bun.com/docs/faq/typescript).
- As of **Bun 1.4 (August 2026)**, the core was rewritten from Zig to Rust (~1M lines) — every benchmark matched or beat 1.3, with a ~20% smaller binary and ~1,400 more passing Node.js compatibility tests. If you read older material describing Bun as "written in Zig," that's now historical — check [bun.com/blog](https://bun.com/blog) for the current architecture before repeating that claim.
- A version manager (`bun upgrade`, or a tool like [bvm](https://github.com/oven-sh/bvm)) matters less here than for Node — Bun ships a single self-contained binary and upgrades in place with `bun upgrade`.

### Advanced

- Bun's speed advantage is largest in **process-startup-bound** workloads (CLI scripts, test suites, cold `npm install`) and smallest in **long-running CPU-bound** workloads, where JS engine choice matters less than algorithmic work. Don't assume "Bun is 3x faster" generalizes to every workload — benchmark your actual use case.
- 1.4 also shipped native replacements for common npm packages as built-in APIs: `Bun.WebView` (headless browser automation, replacing Puppeteer for some cases), `Bun.Image` (image resize/optimize, replacing `sharp`), `Bun.markdown`, `Bun.cron`, and `Bun.Terminal` (native PTYs). These are out of scope for this guide but worth knowing exist before reaching for an npm package that duplicates them.

### Mastery

- Because Bun ships its own JS engine, bundler, and shell interpreter rather than shelling out to system tools, Bun scripts behave identically across macOS/Linux/Windows in ways Node scripts calling `child_process.exec("ls")` do not — this is the foundation for [Part 4's Bun Shell](#5-part-4--runtime-apis-i--file-io--the-bun-shell) section.

[⬆ back to top](#table-of-contents)

---

## 2. Part 2 — Running Code — CLI, Native TS/JSX, Watch Mode

**What it is.** `bun run` / `bun <file>` executes JavaScript, TypeScript, and JSX files directly — no `tsc`, no Babel, no build step.

**Why it matters.** This is Bun's most immediately visible difference from Node: `node script.ts` fails outright (until very recent experimental Node type-stripping); `bun script.ts` just works.

### Beginner

```bash
bun run script.js       # explicit
bun script.js           # shorthand — "run" is inferred
bun script.ts           # TypeScript, no compile step
bun component.tsx       # JSX/TSX too
```

**Real Scenario — try it now.**

```ts
// hello.ts
interface Greeting {
  name: string;
}
function greet({ name }: Greeting): string {
  return `Hello, ${name}!`;
}
console.log(greet({ name: "Bun" }));
```

```bash
$ bun hello.ts
Hello, Bun!
```

### Working Knowledge

- `bun --watch script.ts` restarts the process on file change; `bun --hot script.ts` does in-place hot reload for supported entry points (e.g. `Bun.serve` handlers) without dropping open connections. See [Bun Docs — Watch mode](https://bun.com/docs/runtime/hot).
- `bunfig.toml` (project root) configures runtime/bundler/test defaults — analogous to a combined `.npmrc` + `jest.config.js`.
- `.env` files load automatically (`.env`, `.env.local`, `.env.$NODE_ENV`) — no `dotenv` package needed. See [Bun Docs — .env](https://bun.com/docs/runtime/env).

```bash
# wrong — installing dotenv out of Node habit
bun add dotenv
```
```ts
// right — Bun reads .env automatically; process.env is already populated
console.log(process.env.API_KEY);
```

### Advanced

- **Bun does not type-check.** `bun run` strips TypeScript types the way `esbuild`/`swc` do — fast, but a program with type errors will still run and only fail at the point the bad code actually executes. Run `tsc --noEmit` (or your editor's TS server) separately for real type-checking; Bun's `tsconfig.json` (auto-generated by `bun init`) sets `"noEmit": true` and `"moduleResolution": "bundler"` for exactly this reason. See [Bun Docs — TypeScript](https://bun.com/docs/runtime/typescript).
- JSX defaults follow `tsconfig.json`'s `"jsx"` field; `"jsx": "react-jsx"` is the recommended setting so files don't need an explicit `import React from "react"`.
- Bun enables TS features standard `tsc` restricts by default in some configurations — importing `.ts` files with an explicit extension, and top-level `await` in CommonJS-shaped files — because Bun's module resolver, not `tsc`, is what actually runs the code.

### Mastery

- `bun build --compile` produces a single standalone executable with the runtime embedded — useful for shipping CLI tools without requiring the target machine to have Bun (or Node) installed at all. Covered further in [Part 7](#8-part-7--bundler).

[⬆ back to top](#table-of-contents)

---

## 3. Part 3 — Package Manager

**What it is.** `bun install` (and friends) is a drop-in `npm`/`yarn`/`pnpm` replacement that reads the same `package.json`.

**Why it matters.** Install speed was Bun's original headline feature — a global cache plus copy-on-write linking on supporting filesystems avoids re-downloading and re-extracting packages you already have from another project.

### Beginner

```bash
bun install          # or: bun i
bun add express       # add a dependency
bun add -d typescript  # dev dependency
bun remove express
```

`bun install` reads and writes standard `package.json`; existing npm/Yarn/pnpm projects work without edits. See [Bun Docs — bun install](https://bun.com/docs/pm/cli/install).

### Working Knowledge

- **Lockfile format changed in Bun 1.2**: the default is now `bun.lock`, a human-readable, diffable **text** format (previously `bun.lockb`, a binary format that couldn't be meaningfully code-reviewed in a PR diff). Existing `yarn.lock`/`package-lock.json`/`pnpm-lock.yaml` are auto-migrated the first time `bun install` runs in a project without a `bun.lock`. See [Bun's new text-based lockfile](https://bun.com/blog/bun-lock-text-lockfile).

```bash
# wrong — assuming an old tutorial's binary lockfile is still current
git add bun.lockb   # this file may not even exist on Bun ≥1.2

# right — commit the text lockfile
git add bun.lock
```

- `bun install --frozen-lockfile` (CI convention, matches `npm ci`) fails instead of updating the lockfile if `package.json` and the lockfile disagree.

### Advanced

- **Workspaces**: declare `"workspaces": ["packages/*"]` in the root `package.json`; each subdirectory becomes an installable package. Cross-workspace dependencies use the `workspace:*` protocol, and `bun install` hoists shared dependencies to the root `node_modules` — same mental model as Yarn/pnpm workspaces. Use `bun install --filter <pkg>` to scope an install to one workspace. See [Bun Docs — Workspaces](https://bun.com/docs/pm/workspaces).
- `bun pm ls`, `bun pm cache`, and `bun outdated` cover day-to-day maintenance without extra tooling.

### Mastery

- Bun's installer uses a **global content-addressed store** plus filesystem-level cloning (reflinks/hardlinks where the OS/filesystem supports it) so that installing a package already present from another project is near-instant and doesn't duplicate disk space — conceptually similar to pnpm's store, but with Bun additionally able to flatten `node_modules` like npm/Yarn when a package expects that layout (an "isolated" linker mode also exists for stricter dependency isolation, tunable via `bunfig.toml`).

[⬆ back to top](#table-of-contents)

---

## 4. Part 4 — Runtime APIs I — File I/O & the Bun Shell

**What it is.** Bun-specific globals (`Bun.file`, `Bun.write`) for fast file I/O, and `Bun.$` — a built-in cross-platform shell for scripting.

**Why it matters.** Node's `fs` module works fine in Bun (compatibility layer, [Part 9](#10-part-9--nodejsnpm-compatibility)), but Bun's own APIs are faster and, for shell scripting, solve a real cross-platform pain point: bash scripts don't run on Windows, and reaching for `child_process.exec("rm -rf ...")` is both non-portable and an injection risk.

### Beginner

```ts
// Bun.file() returns a lazy, readable BunFile — nothing is read until you ask
const file = Bun.file("data.json");
const text = await file.text();
const json = await file.json();

await Bun.write("out.txt", "hello from Bun");
```

### Working Knowledge

```ts
import { $ } from "bun";

await $`echo "Hello World!"`;
const wordCount = await $`echo "Hello Bun" | wc -w`.text();
```

Bun Shell implements its own lexer/parser/interpreter with native versions of common commands (`ls`, `cd`, `rm`, `mv`, `cat`, ...) — scripts run identically on macOS, Linux, *and Windows* without relying on the system having `bash` installed. See [Bun Docs — Shell](https://bun.com/docs/runtime/shell).

```bash
# wrong — a "portable" script that actually isn't
#!/bin/bash
rm -rf ./dist && cp -r ./src ./dist
```
```ts
// right — runs identically on Windows, macOS, Linux, no bash required
import { $ } from "bun";
await $`rm -rf ./dist && cp -r ./src ./dist`;
```

### Advanced

- **Interpolated variables are treated as literal strings by default** — `` $`rm -rf ${userInput}` `` does not let `userInput` inject additional shell syntax, unlike naive `bash -c` string concatenation. This protection is lost the moment you explicitly invoke a nested shell (e.g. `` $`bash -c ${cmd}` ``) — at that point you're back to manual sanitization. See [Bun Docs — Shell security](https://bun.com/docs/runtime/shell).
- Redirection uses standard operators (`>`, `<`, `2>`, `&>`) and can target JS values directly, e.g. `` await $`echo hi > ${buffer}` `` writes into a `Buffer`.
- Command substitution uses `$(...)`, **not backticks** — backticks are already the shell tag's own delimiter, so nested backticks don't parse as substitution.

### Mastery

- `Bun.file()` is backed by an internal reference to the OS file descriptor / mmap-able region rather than eagerly buffering, so passing a `BunFile` into `Bun.serve()`'s response or `Bun.write()` as a copy source can avoid a full read-into-memory round trip for large files — this is part of why static file serving via `Bun.serve` is fast (see [Part 5](#6-part-5--http--websockets)).

[⬆ back to top](#table-of-contents)

---

## 5. Part 5 — HTTP & WebSockets

**What it is.** `Bun.serve()` — a built-in HTTP server with routing, WebSocket upgrade support, and no `express`/`http` module required (though both work via the compat layer).

**Why it matters.** This is the API most people reach for first, and it changed shape meaningfully in Bun 1.2 with a first-class routes object.

### Beginner

```ts
Bun.serve({
  port: 3000,
  fetch(req) {
    return new Response("Hello from Bun!");
  },
});
```

### Working Knowledge

Since **Bun v1.2.3+**, `Bun.serve()` accepts a `routes` object as an alternative (or complement) to the single `fetch` catch-all — static responses, path-param handlers, per-method handlers, wildcards, and direct file serving:

```ts
Bun.serve({
  routes: {
    "/api/status": new Response("OK"),
    "/users/:id": (req) => new Response(`User ${req.params.id}`),
    "/api/posts": {
      GET: () => new Response("List posts"),
      POST: async (req) => {
        const body = await req.json();
        return Response.json({ created: true, ...body });
      },
    },
    "/favicon.ico": Bun.file("./favicon.ico"),
    "/api/*": Response.json({ message: "Not found" }, { status: 404 }),
  },
  // fetch() still runs for anything routes doesn't match
  fetch(req) {
    return new Response("Not Found", { status: 404 });
  },
});
```

See [Bun Docs — Routing](https://bun.com/docs/runtime/http/routing).

```ts
// wrong — hand-rolling path-param parsing out of Express habit
fetch(req) {
  const url = new URL(req.url);
  const match = url.pathname.match(/^\/users\/(\d+)$/);
  if (match) return new Response(`User ${match[1]}`);
}
```
```ts
// right — let the routes object do it
routes: {
  "/users/:id": (req) => new Response(`User ${req.params.id}`),
}
```

### Advanced

- **WebSockets** are configured via a `websocket` handler alongside `fetch` — `fetch` handles the initial upgrade (`server.upgrade(req)`), and `open`/`message`/`close` callbacks handle the connection lifecycle. Bun's WebSocket implementation supports **pub/sub** (`ws.subscribe("channel")`, `server.publish("channel", msg)`) natively, without a separate library. See [Bun Docs — WebSockets](https://bun.com/docs/api/websockets).
- `bun --hot` reloads `Bun.serve` handlers in place, preserving open WebSocket/HTTP connections during development — this only works for the specific hot-reload-aware entry points Bun instruments, not arbitrary long-running processes.

### Mastery

- Static routes (a bare `Response` or `Bun.file()` value, not a function) are pre-resolved once at server startup rather than re-evaluated per-request, which is part of why the routes object outperforms an equivalent hand-written `if`/`switch` chain inside a single `fetch` handler for high-traffic static/semi-static endpoints.

[⬆ back to top](#table-of-contents)

---

## 6. Part 6 — SQLite & FFI

**What it is.** `bun:sqlite` — a built-in, synchronous SQLite driver — and `bun:ffi` — calling native (C-ABI-compatible) libraries directly from JS/TS.

**Why it matters.** Both are examples of Bun including, as built-ins, capabilities that normally require a native npm addon (`better-sqlite3`, `node-ffi-napi`) with its own install/compile step.

### Beginner

```ts
import { Database } from "bun:sqlite";

const db = new Database("mydb.sqlite");
db.run("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT)");
db.run("INSERT INTO users (name) VALUES (?)", ["Ada"]);

const rows = db.query("SELECT * FROM users").all();
console.log(rows); // [{ id: 1, name: "Ada" }]
```

### Working Knowledge

- The API is deliberately close to [`better-sqlite3`](https://github.com/WiseLibs/better-sqlite3) — **synchronous** by design (SQLite itself is synchronous under the hood; wrapping it in promises would add overhead without real concurrency benefit for a local file-backed database).
- Supports prepared statements (`db.prepare(sql)`, reusable across calls), named parameters (`$name` / `:name`), transactions (`db.transaction(fn)`), and mapping rows onto classes via `.as(MyClass)`.
- Benchmarks in Bun's own docs show `bun:sqlite` running several times faster than `better-sqlite3` for read-heavy workloads — a Bun-native benefit you get for free by switching drivers, no other code changes needed. See [Bun Docs — SQLite](https://bun.com/docs/runtime/sqlite).

### Advanced

```ts
// wrong — this npm package needs node-gyp / a native compile step
bun add better-sqlite3
```
```ts
// right — no install needed, it's a built-in module
import { Database } from "bun:sqlite";
```

`bun:ffi` calls into native shared libraries (`.so`/`.dylib`/`.dll`) built in any C-ABI-compatible language (C, Rust, Zig, C++, ...):

```ts
import { dlopen, FFIType, suffix } from "bun:ffi";

const path = `libadd.${suffix}`; // .so / .dylib / .dll depending on OS
const { symbols: { add } } = dlopen(path, {
  add: { args: [FFIType.i32, FFIType.i32], returns: FFIType.i32 },
});

console.log(add(2, 3)); // 5
```

`JSCallback` lets native code call back *into* JS/TS — necessary for async native APIs that report completion via callback. See [Bun Docs — FFI](https://bun.com/docs/runtime/ffi).

### Mastery

- `bun:ffi` uses [tinycc](https://bellard.org/tcc/) under the hood at points to JIT small trampolines for calling into native code with minimal marshalling overhead — this is why simple FFI calls in Bun benchmark close to native speed compared to Node's N-API addon path, which has more fixed per-call overhead.
- FFI/`bun:sqlite` are both **not** covered by Bun's Node.js compatibility layer promises — they're Bun-only APIs. Code using them is not portable back to Node without a shim, which matters if "might need to run this under Node someday" is a real constraint for you.

[⬆ back to top](#table-of-contents)

---

## 7. Part 7 — Bundler

**What it is.** `bun build` (CLI) / `Bun.build()` (JS API) — bundles JS/TS/JSX (and CSS, and more via plugins) for browser, Node, or Bun targets.

**Why it matters.** Replaces `esbuild`/`webpack`/`rollup` for many projects, sharing the same fast native transpiler Bun's runtime uses.

### Beginner

```bash
bun build ./index.tsx --outdir ./dist
```

### Working Knowledge

- Three targets: `"browser"` (default), `"bun"`, `"node"`. `"bun"` currently behaves like `"node"`, but is reserved so Bun can later auto-polyfill Bun-specific globals/`bun:*` modules when bundling for its own runtime. See [Bun Docs — Bundler](https://bun.com/docs/bundler).

```ts
await Bun.build({
  entrypoints: ["./index.tsx"],
  outdir: "./dist",
  target: "browser",
  minify: true,
  sourcemap: "external",
});
```

### Advanced

- **Plugins** hook into the bundler's (and, for the same plugin API, the runtime's) module resolution/loading pipeline — `onResolve`/`onLoad`-style callbacks let a plugin add support for new file extensions (`.scss`, `.yaml`, `.svg`-as-component) or implement framework-level features (CSS extraction, macros). See [Bun Docs — Plugins](https://bun.com/docs/bundler/plugins).
- `bun build --compile` produces a **standalone native executable** with the Bun runtime and your code embedded — the output runs on a machine with no Bun (or Node) installation at all. Useful for shipping CLI tools; not the same thing as a minified browser bundle.

```bash
# wrong — expecting `--compile` output to run in a browser
bun build --compile ./cli.ts --outfile mycli
# this produces a native executable, not a browser bundle

# right — separate commands for separate targets
bun build --compile ./cli.ts --outfile mycli          # native CLI binary
bun build ./web.tsx --outdir ./dist --target browser    # browser bundle
```

### Mastery

- Because the bundler and the runtime share the same underlying transpiler, dev-mode "bundling" for `Bun.serve`'s fullstack HTML-import mode (out of scope here, see [Bun Docs — Fullstack](https://bun.com/docs/bundler/fullstack)) can transform modules on-demand per-request rather than pre-building the whole graph, which is part of how Bun achieves fast cold-start dev servers without a separate watch-and-rebuild daemon process.

[⬆ back to top](#table-of-contents)

---

## 8. Part 8 — Test Runner

**What it is.** `bun test` — a built-in, Jest-compatible test runner (`describe`, `it`/`test`, `expect`, mocks, snapshots, coverage) with no separate install.

**Why it matters.** Most existing Jest test files run under `bun test` with little to no modification, but startup and execution are dramatically faster since there's no separate Jest process/config resolution step.

### Beginner

```ts
// math.test.ts
import { describe, expect, test } from "bun:test";

function add(a: number, b: number) { return a + b; }

describe("add", () => {
  test("adds two numbers", () => {
    expect(add(2, 3)).toBe(5);
  });
});
```

```bash
bun test
```

### Working Knowledge

- **Mocking** is built in — `mock()` or the Jest-familiar `jest.fn()`, both importable from `"bun:test"`, no `jest.mock`-style module registry gymnastics required for simple function mocks. See [Bun Docs — Test runner](https://bun.com/docs/test).
- **Snapshot testing** works like Jest's: `expect(value).toMatchSnapshot()` writes a snapshot file on first run and diffs against it afterward; `bun test --update-snapshots` refreshes them intentionally. See [Bun Docs — Snapshots](https://bun.com/docs/test/snapshots).

```bash
# wrong — assuming coverage needs a separate tool, out of Jest/nyc habit
bun add -d nyc

# right — coverage is a runner flag
bun test --coverage
```

### Advanced

- Coverage thresholds are configurable in `bunfig.toml` (`[test] coverageThreshold = 0.8`); `bun test --coverage` exits non-zero if the suite falls short, making it CI-gate-ready without extra tooling. See [Bun Docs — Code coverage](https://bun.com/docs/test/code-coverage).
- `bun test` auto-discovers `*.test.ts`/`*.spec.ts` (configurable) and runs matching files in parallel across worker processes by default — a suite with slow, stateful tests that assume serial execution (e.g. sharing a single SQLite file without per-test isolation) can produce flaky results that wouldn't show up under a serial runner. Isolate state per test (temp files/tables) rather than relying on execution order.

### Mastery

- Because `bun test` shares Bun's native transpiler rather than invoking Babel/`ts-jest` per file, most of the "why is my Jest config for TS so slow / fragile" class of problems (transform caching, `moduleNameMapper` for path aliases, `esModuleInterop` mismatches) simply doesn't arise — the tradeoff is less configurability for exotic transform pipelines than Jest's plugin ecosystem offers.

[⬆ back to top](#table-of-contents)

---

## 9. Part 9 — Node.js/npm Compatibility

**What it is.** Bun implements large parts of Node's built-in module API surface (`fs`, `path`, `http`, `crypto`, `stream`, ...) and reads standard `package.json`, so most existing Node/npm packages and codebases run under Bun with no code changes.

**Why it matters.** This is what makes Bun a practical drop-in rather than a from-scratch ecosystem — but "mostly compatible" has real, specific gaps worth knowing before you assume a project will "just work."

### Beginner

```ts
// existing Node code, unmodified
import { readFileSync } from "node:fs";
import path from "node:path";

const data = readFileSync(path.join(__dirname, "data.json"), "utf-8");
console.log(JSON.parse(data));
```

Most `node:*` built-ins work as-is under `bun run`.

### Working Knowledge

- npm-package compatibility for widely-used packages is high (commonly cited in the high-90s% for top-1000 packages as of 2026), and popular frameworks/ORMs (Express, Fastify, Hono, Prisma, Drizzle, Mongoose) work without issue for typical usage.
- The recurring gap is **native addons**: packages compiled with `node-gyp` against Node's N-API, or that ship prebuilt `.node` binaries, are the main compatibility risk — `sharp` (image processing) and the native build of `bcrypt` are commonly-cited examples; pure-JS alternatives (`bcryptjs`) or Bun's own native equivalents (`Bun.Image`, see [Part 1](#2-part-1--orientation--why-bun-is-fast)) are the usual workaround.

```bash
# wrong — assuming a native-addon package "should just work" because most do
bun add sharp
# may fail or behave unexpectedly depending on your platform/Bun version

# right — check for a Bun-native equivalent or pure-JS alternative first
bun add bcryptjs   # instead of native bcrypt, if compatibility is a concern
```

### Advanced

- Before migrating an existing Node project, actually run its test suite under `bun test` (or `bun run` its existing Jest suite) rather than assuming compatibility from documentation alone — the fastest way to find a real gap is to hit it, and Bun's compatibility percentage improves fast enough between releases that last year's known-gaps list may already be stale.
- `bun --bun` forces certain shebangs/tools that would otherwise defer to a system Node install to run under Bun instead — relevant when a script or package.json `"bin"` entry assumes `node` explicitly.

### Mastery

- Bun tracks Node compatibility partly by running Node's own internal test suite (`node-test-suite`-style regression tests) against Bun's implementations and counting pass rates — the ~1,400 additional passing tests cited for the 1.4 release ([Part 1](#2-part-1--orientation--why-bun-is-fast)) come from this process, which is a more rigorous compatibility signal than "the top 1000 npm packages install cleanly," since it exercises edge-case Node API behavior directly.

[⬆ back to top](#table-of-contents)

---

## 11. Cheat Sheets

### CLI Quick Reference

| Command | Purpose |
|---|---|
| `bun <file>` | Run a JS/TS/JSX file directly |
| `bun --watch <file>` | Re-run on file change |
| `bun --hot <file>` | In-place hot reload (supported entry points) |
| `bun install` / `bun i` | Install dependencies |
| `bun add <pkg>` / `bun add -d <pkg>` | Add dependency / dev dependency |
| `bun remove <pkg>` | Remove a dependency |
| `bun install --frozen-lockfile` | CI install, fails on lockfile drift |
| `bun build <entry> --outdir <dir>` | Bundle for browser/node/bun |
| `bun build --compile <entry> --outfile <name>` | Standalone native executable |
| `bun test` | Run tests |
| `bun test --coverage` | Run tests with coverage report |
| `bun test --update-snapshots` | Refresh snapshot files |
| `bun upgrade` | Upgrade the Bun binary itself |
| `bun pm ls` / `bun pm cache` / `bun outdated` | Package maintenance |

### Runtime API Quick Reference

| API | Purpose |
|---|---|
| `Bun.file(path)` | Lazy file handle (`.text()`, `.json()`, `.arrayBuffer()`) |
| `Bun.write(dest, data)` | Write a file (or copy a `BunFile`) |
| `Bun.$\`cmd\`` | Cross-platform shell scripting |
| `Bun.serve({ routes, fetch })` | HTTP server with routing |
| `bun:sqlite` `Database` | Built-in synchronous SQLite driver |
| `bun:ffi` `dlopen` | Call native C-ABI libraries |
| `bun:test` `describe/test/expect/mock` | Jest-compatible test API |

[⬆ back to top](#table-of-contents)

---

## 12. Suggested Learning Order

1. [Part 1 — Orientation](#2-part-1--orientation--why-bun-is-fast) and [Part 2 — Running Code](#3-part-2--running-code--cli-native-tsjsx-watch-mode): install Bun, run a `.ts` file, understand the "no type-checking" gotcha.
2. [Part 3 — Package Manager](#4-part-3--package-manager): `bun install` on an existing Node project; look at the `bun.lock` diff after adding a package.
3. [Part 4 — File I/O & Shell](#5-part-4--runtime-apis-i--file-io--the-bun-shell): rewrite one small bash script as a `Bun.$` script.
4. [Part 5 — HTTP & WebSockets](#6-part-5--http--websockets): build a small `Bun.serve` API with the routes object.
5. [Part 8 — Test Runner](#9-part-8--test-runner): write a test file with a mock and a snapshot.
6. [Part 7 — Bundler](#8-part-7--bundler): bundle the HTTP server for `bun` target; try `--compile`.
7. [Part 6 — SQLite & FFI](#7-part-6--sqlite--ffi): wire `bun:sqlite` into the HTTP server as persistence; treat FFI as reference material until you actually need it.
8. [Part 9 — Node.js/npm Compatibility](#10-part-9--nodejsnpm-compatibility): read once for the mental model, then revisit whenever a specific package misbehaves.

[⬆ back to top](#table-of-contents)

---

## 13. Quick Self-Check

- Why does `bun run script.ts` succeed even if the file has a type error `tsc` would catch?
- What changed about Bun's default lockfile in 1.2, and why does that matter for code review?
- Why is interpolating a variable into a `` $`...` `` shell command safer by default than string-concatenating it into `child_process.exec`? What breaks that safety?
- In a `Bun.serve()` routes object, what's the difference between a static route value (`new Response(...)`) and a handler function, in terms of when it executes?
- Why is `bun:sqlite`'s API synchronous rather than promise-based?
- Name the category of npm package most likely to fail under Bun's Node compatibility layer, and why.
- What's the difference between `bun build`'s `"bun"` target and `bun build --compile`?

[⬆ back to top](#table-of-contents)
