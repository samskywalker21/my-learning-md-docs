# Node.js — Mastery Guide (Overview)

The entry point for this repo's Node.js material. This document holds the shared spec, the runtime primer, setup and version management, the map of the focused sub-docs, the "what changed under you" delta that invalidates most older tutorials, and the reference apparatus (learning order, self-check, cross-doc cheat sheet).

The actual teaching lives in the sub-docs. Start here, then follow the [Suggested Learning Order](#8-suggested-learning-order).

---

## About This Document

This section is the canonical spec for **every doc in this set** — the sub-docs each carry a short pointer back here rather than repeating it.

- **Framing: curiosity-driven.** No upcoming task is driving this. The goal is durable, hands-on understanding of Node.js itself. The docs are built as a *guided tutorial* — a sequence of runnable examples and **Try It** exercises you are meant to actually type into a terminal, each stating what correct output looks like. Theory ("why is it designed this way") appears only where it explains a mechanism — why `process.nextTick` starves the loop, why streams need backpressure, why `require()` of ESM was impossible for a decade — never as a standalone history section.
- **Confirmed scope: Node.js as a whole.** The runtime and its event loop, modules and packages, the filesystem/Buffer/stream layer, processes and threads, HTTP and networking, the built-in test runner, and the production/diagnostics surface. Fourteen Parts across seven sub-docs.
- **Deliberately excluded** (with pointers): web frameworks (Express/Fastify/Hono/Nest), ORMs and database drivers, deployment platforms, and native addon authoring in C++. See [Deliberately Not Covered](#7-deliberately-not-covered-and-where-to-look-instead).
- **JavaScript with ESM throughout.** Examples are plain ES-module JavaScript, because that is what the Node docs themselves lead with and what the module system now defaults toward. TypeScript appears where it is genuinely the realistic choice — the native type-stripping section in [Part 4](./nodejs-modules-packages.md#part-4--modules-cjs-esm-and-typescript), and typed configuration in [Part 13](./nodejs-production-diagnostics.md#part-13--production-node). CommonJS is taught, not avoided: you will read it for the rest of your career.
- **Depth tiers.** Each Part runs **Beginner → Working Knowledge → Advanced → Mastery**, so you can stop at whichever level you need today. Tiers are collapsed where one would be filler, and each collapse is stated inline. Collapsing is expected behaviour, not a defect to fix on a later pass. The weight of this set sits in Working Knowledge and Advanced; Mastery concentrates in [Runtime Foundations](./nodejs-runtime-foundations.md) and [Production & Diagnostics](./nodejs-production-diagnostics.md).
- **Assumed background: strong JavaScript and TypeScript, new-ish to Node.** You are comfortable with closures, promises, `async`/`await`, ES module syntax, and basic types — see [`javascript/javascript-errors-async-modules.md`](../javascript/javascript-errors-async-modules.md) and [`typescript/typescript-foundations.md`](../typescript/typescript-foundations.md) if any of that is shaky, because this set does not re-teach the language. What it *does* teach from zero is everything Node adds on top: the event loop's phases, CommonJS vs. ESM resolution, Buffers, streams and backpressure, child processes and worker threads, and the diagnostics surface.
- **Bun contrast callouts.** Short `> **vs. Bun**` callouts appear where the two runtimes genuinely diverge (TypeScript support, `.env` loading, the test runner's API, `Bun.file` vs. `node:fs`, JavaScriptCore vs. V8), each cross-linking the relevant [`bun/`](../bun/bun-mastery-guide.md) doc. They are orientation, not a prerequisite — skip them freely.
- **Per-section ingredients**, used where they earn their place rather than forced into every subsection:
  - a short plain-language explanation of what a thing is and why it matters;
  - a **wrong vs. right** snippet pair wherever a gotcha has a common wrong form;
  - a **Try It** — a runnable exercise with the expected observable output — since this is a curiosity-driven set (production-style **Real Scenario** blocks appear only where a bug genuinely only bites under real load, in CI, or in a container);
  - an **ASCII diagram** where the concept is spatial or structural — the event loop's phases, a stream pipeline, the module resolution walk, worker topology.
- **Sourcing rule.** The [Node.js API docs](https://nodejs.org/docs/latest-v24.x/api/) and the [Node.js Learn guides](https://nodejs.org/en/learn) are the primary sources, cited inline at the specific claim they support and pinned to the `latest-v24.x` path so the links stay stable as new majors ship. GitHub issues and the release blog are supplementary. Where a popular-but-outdated convention disagrees with the official docs, the official docs win and the doc says so explicitly. **Deviation from the repo template, stated openly:** Stack Overflow was not reachable from the authoring environment on the date below, so no SO links appear in this set — the same deviation the [Bun set](../bun/bun-mastery-guide.md#about-this-document) records. Gotchas are instead sourced to the caveats the official docs state themselves (Node's docs are unusually candid about their own sharp edges — the `nextTick` docs say the names "should be swapped"; the permissions docs list six ways to bypass the sandbox). If you extend these docs from an environment that *can* reach SO, adding real-world citations at the gotchas is a welcome improvement — just don't construct SO URLs from memory.
- **Written against.** **Node.js 24.x ("Krypton"), the Active LTS line**, with Node 22 ("Jod") differences flagged wherever a behaviour or flag changed between them. Node 26 is the Current line as of this writing and is noted only where it matters ([Node.js Releases](https://nodejs.org/en/about/previous-releases), [nodejs/Release](https://github.com/nodejs/Release)). Verified **September 14, 2026**.
- **To update this doc later.** *Preserve*: the curiosity-driven framing, the numbered Part structure with explicit tier subheadings, the wrong-vs-right pairs, the Try It exercises with expected output, the `> **vs. Bun**` callout convention, the TOC/back-to-top/anchor apparatus, and the per-Part cheat sheets. *Re-verify against current docs first*: the LTS line and the release-schedule change described in [§3](#3-orientation--setup); **everything in [§5](#5-the-delta-that-invalidates-old-tutorials)**, which is the fastest-rotting section here; the stability labels on `--experimental-test-coverage`, `node:sqlite`, `--experimental-transform-types`, and single-executable applications, all of which were still moving at the time of writing; and the `node:test` default glob patterns, which have gained extensions over time.

---

## Table of Contents

1. [How This Set Is Organised](#1-how-this-set-is-organised)
2. [The Sub-Docs](#2-the-sub-docs)
3. [Orientation & Setup](#3-orientation--setup)
4. [Runtime Primer — What Node Actually Is](#4-runtime-primer--what-node-actually-is)
5. [The Delta That Invalidates Old Tutorials](#5-the-delta-that-invalidates-old-tutorials)
6. [What Changed Under You: Node 22 → 24](#6-what-changed-under-you-node-22--24)
7. [Deliberately Not Covered (and where to look instead)](#7-deliberately-not-covered-and-where-to-look-instead)
8. [Suggested Learning Order](#8-suggested-learning-order)
9. [Quick Self-Check](#9-quick-self-check)
10. [Cross-Doc Cheat Sheet](#10-cross-doc-cheat-sheet)

---

## 1. How This Set Is Organised

Node.js is 14 Parts of teaching material spread over 7 sub-docs, plus this overview. The split follows the order in which the concepts actually become necessary — the event loop early, because every other subject in the set (streams, servers, workers, diagnostics) is a consequence of how Node schedules work:

```
  nodejs-mastery-guide.md   ← you are here: spec, primer, setup, map, delta, reference
        │
        ├── nodejs-runtime-foundations.md     Parts 1–3    how Node runs your code
        ├── nodejs-modules-packages.md        Parts 4–5    how your code finds other code
        ├── nodejs-files-streams-buffers.md   Parts 6–7    bytes, files, and flow control
        ├── nodejs-processes-workers.md       Parts 8–9    the process, and escaping one thread
        ├── nodejs-http-networking.md         Parts 10–11  serving and calling over the wire
        ├── nodejs-testing-debugging.md       Part 12      node:test and the inspector
        └── nodejs-production-diagnostics.md  Parts 13–14  shipping it, and seeing inside it
```

Each Part is self-contained enough to reopen on its own, and each ends with a quick-reference table. Within a Part, the tiers mean:

| Tier | What you get |
|---|---|
| **Beginner** | What it is, the minimal correct usage, nothing else |
| **Working Knowledge** | Everyday patterns, the API surface you actually reach for |
| **Advanced** | Edge cases, gotchas, performance and correctness implications |
| **Mastery** | Internals, powerful-but-rare capabilities, expert tradeoffs |

Where a tier would be filler, it is collapsed and the doc says so — e.g. "*Buffers get Beginner → Advanced only; the Mastery material is really V8's external-memory accounting, which [Part 14](./nodejs-production-diagnostics.md#part-14--diagnostics) covers with the right tooling to hand.*"

[↑ Back to top](#table-of-contents)

---

## 2. The Sub-Docs

### [`nodejs-runtime-foundations.md`](./nodejs-runtime-foundations.md) — Parts 1–3

| Part | Covers |
|---|---|
| **1. Running code** | `node file.js`, the REPL, `--watch`, `--env-file`, `--run`, `NODE_OPTIONS`, shebangs, and what the CLI surface actually replaced |
| **2. The event loop** | The six phases, the poll phase's blocking decision, microtasks vs. `process.nextTick` vs. `setImmediate`, starvation, libuv's thread pool |
| **3. Async patterns in Node** | Error-first callbacks and `util.promisify`, `AbortSignal` everywhere, `EventEmitter`, `events.once`/`on`, async iteration, unhandled rejections |

### [`nodejs-modules-packages.md`](./nodejs-modules-packages.md) — Parts 4–5

| Part | Covers |
|---|---|
| **4. Modules: CJS, ESM and TypeScript** | The resolution algorithm, `"type"` and syntax detection, `exports`/`imports` conditions, the dual-package hazard, `require(esm)`, `createRequire`, loader hooks, native type stripping |
| **5. Packages & the npm surface** | The `package.json` fields that matter, semver ranges, lockfiles and `npm ci`, `npx`, workspaces, `overrides`, audit and supply-chain hygiene |

### [`nodejs-files-streams-buffers.md`](./nodejs-files-streams-buffers.md) — Parts 6–7

| Part | Covers |
|---|---|
| **6. Files, paths & Buffers** | `node:fs/promises`, sync vs. async vs. callback, `node:path` and cross-platform pitfalls, `Buffer` as a `Uint8Array`, encodings, `node:os` |
| **7. Streams & backpressure** | Readable/Writable/Duplex/Transform, `pipeline` vs. `pipe`, backpressure mechanics, object mode, async iteration, Web Streams interop |

### [`nodejs-processes-workers.md`](./nodejs-processes-workers.md) — Parts 8–9

| Part | Covers |
|---|---|
| **8. The process** | `process.argv`/`env`/`exitCode`, `util.parseArgs`, signals and graceful shutdown, `child_process` (`spawn`/`exec`/`fork`), stdio wiring, shell-injection safety |
| **9. Threads & parallelism** | `worker_threads`, `SharedArrayBuffer` and `Atomics`, transferables, worker pools, `cluster` vs. a process manager, when threads are the wrong answer |

### [`nodejs-http-networking.md`](./nodejs-http-networking.md) — Parts 10–11

| Part | Covers |
|---|---|
| **10. HTTP servers** | `node:http` from first principles, routing by hand, request bodies and backpressure, the timeout matrix, graceful shutdown, the framework landscape |
| **11. Clients & the rest of the stack** | `fetch`/undici, agents and connection pooling, timeouts and retries, `node:net`/`dgram`/`tls`, WebSocket, DNS |

### [`nodejs-testing-debugging.md`](./nodejs-testing-debugging.md) — Part 12

| Part | Covers |
|---|---|
| **12. `node:test` & debugging** | Discovery and glob defaults, `describe`/`it`, subtests, `node:assert`, mocks and `mock.timers`, snapshots, coverage, reporters, isolation and concurrency, the inspector |

### [`nodejs-production-diagnostics.md`](./nodejs-production-diagnostics.md) — Parts 13–14

| Part | Covers |
|---|---|
| **13. Production Node** | Configuration and secrets, the permission model, structured logging, containers and signals, memory limits, single-executable applications, security hygiene |
| **14. Diagnostics** | `AsyncLocalStorage`, `diagnostics_channel`, `perf_hooks`, CPU profiles and heap snapshots, tracing, finding the leak and the blocked loop |

[↑ Back to top](#table-of-contents)

---

## 3. Orientation & Setup

### Which version to install

Node ships two kinds of release line, and the difference decides whether your production incident is interesting or boring ([Node.js Releases](https://nodejs.org/en/about/previous-releases)):

| Line | What it means | Use it for |
|---|---|---|
| **Active LTS** | Supported for roughly 30 months from release; only low-risk changes land | Everything you deploy |
| **Current** | The newest line; new features land here first | Trying features early, library CI matrices |
| **Maintenance** | Critical fixes only, until end-of-life | Nothing new; plan your exit |

As of **September 2026**: **Node 24 ("Krypton") is Active LTS** (supported to 2028-05-31), Node 22 ("Jod") is in maintenance (to 2027-04-30), Node 20 ("Iron") reached end-of-life in April 2026, and **Node 26 is Current**. This set targets **24**.

> **Flagged change — the release schedule itself is being replaced.** The long-standing "even numbers become LTS, odd numbers never do" rule is ending. From October 2026 Node moves to **one major release per year in April, with every release promoted to LTS in October** ([Evolving the Node.js Release Schedule](https://nodejs.org/en/blog/announcements/evolving-the-nodejs-release-schedule)). Node 26 is the last line under the old model; Node 27 is the first under the new one. Any article explaining "why Node skips odd versions for LTS" is describing a rule with an expiry date on it.

### Installing — don't use the OS package manager

The most common setup mistake is `apt install nodejs` or `brew install node`: it pins you to one version system-wide and puts npm's global prefix somewhere that needs `sudo`. Use a version manager instead, so each project can declare its own Node:

```bash
# fnm — fast, written in Rust, works on macOS/Linux/Windows
fnm install 24
fnm use 24
fnm default 24

# nvm — the long-standing POSIX option
nvm install 24
nvm use 24
```

Then pin the version in the repo so everyone (and CI) agrees:

```bash
# .nvmrc — read by nvm, fnm, mise, asdf, and most CI setup actions
24
```

and state the runtime contract in `package.json`, which is what npm and your dependants actually check:

```json
{
  "engines": { "node": ">=24.0.0" }
}
```

**Try It — confirm what you're actually running.**

```bash
node -v                          # → v24.x.y
npm -v                           # → 11.x   (Node 24 ships npm 11)
node -p "process.versions.v8"    # → 13.6.x or later
node -p "process.versions.uv"    # → the libuv version — the other half of Node
```

That last line is the point of the exercise: `process.versions` lists every C library compiled into your binary. Node is not one program, and [§4](#4-runtime-primer--what-node-actually-is) is about the two that matter most.

If `node -v` disagrees with `.nvmrc`, your shell is resolving a different binary — `which -a node` (PowerShell: `Get-Command node -All`) shows you which one wins.

> **Flagged change — Corepack is leaving.** Corepack, the shim that let `package.json`'s `packageManager` field pin pnpm or yarn, is still bundled in Node 24 as experimental, but the TSC voted to **stop distributing it from Node 25 onward** ([nodejs/corepack](https://github.com/nodejs/corepack)). Tutorials that open with `corepack enable` as a setup step are describing something that will not exist on the next LTS you install. Install pnpm or yarn directly instead.

> **vs. Bun** — Bun is a single binary with no version-manager ecosystem of its own (`bun upgrade` updates in place), and ships its package manager, test runner and bundler in the same executable. Node's equivalents are assembled from the CLI flags and built-in modules this set covers — which is more moving parts, but also why you can swap any one of them out. See [`bun/bun-mastery-guide.md`](../bun/bun-mastery-guide.md).

[↑ Back to top](#table-of-contents)

---

## 4. Runtime Primer — What Node Actually Is

Read this before the sub-docs if "Node" has so far been a black box labelled *JavaScript, but on the server*.

Node is three layers bolted together:

```
   your JavaScript
        │
        ▼
  ┌────────────────────────────────────────────────┐
  │  V8             compiles and runs the JS       │  ← the language
  │                 (heap, GC, JIT, microtask queue)│
  ├────────────────────────────────────────────────┤
  │  Node bindings  the node:* modules, written in │  ← the API surface
  │                 C++ and JS, exposed to you     │
  ├────────────────────────────────────────────────┤
  │  libuv          the event loop, the thread     │  ← the concurrency model
  │                 pool, async file and net I/O   │
  └────────────────────────────────────────────────┘
        │
        ▼
   the operating system
```

Three consequences follow, and between them they explain most of Node's reputation:

1. **Your JavaScript runs on exactly one thread.** There is one call stack. A `for` loop that takes 400 ms blocks *every* pending request, timer and socket for 400 ms. This is why [Part 2](./nodejs-runtime-foundations.md#part-2--the-event-loop) comes so early in the set, and why [Part 9](./nodejs-processes-workers.md#part-9--threads--parallelism) exists at all.
2. **I/O is not on that thread.** libuv hands network I/O to the OS's own event notification machinery (epoll on Linux, kqueue on macOS, IOCP on Windows) and pushes filesystem and DNS work onto a small thread pool, then invokes your callback back on the main thread when the result is ready. "Node is fast at I/O" means, precisely, "Node was never sitting there waiting."
3. **JavaScript's own job queue sits *inside* this.** Promise continuations are V8 microtasks; they drain completely *between* event-loop phases rather than occupying a phase of their own. That one sentence is the source of most "why did this log in that order?" confusion, and [Part 2](./nodejs-runtime-foundations.md#part-2--the-event-loop) takes it apart properly.

A one-paragraph mental model to carry into the sub-docs: **Node is a single-threaded JavaScript program wrapped in a C library that does the waiting for it.** Everything Node is good at — many concurrent connections, streaming data through transformations, glueing services together — is what that architecture is good at. Everything Node is bad at — image resizing, huge JSON parses, cryptography in a loop, anything CPU-bound — is what that architecture is bad at. And the fix is always to move the work *off* the loop, never to make the loop faster.

[↑ Back to top](#table-of-contents)

---

## 5. The Delta That Invalidates Old Tutorials

This is the section to re-read before trusting a search result. Each row is a case where the current official docs and a widely-copied convention disagree — and the docs win.

| You'll see this in older material | Current reality on Node 24 |
|---|---|
| `npm i dotenv` + `require('dotenv').config()` | `node --env-file=.env app.js` is built in and **stable** as of 24.10.0 ([CLI docs](https://nodejs.org/docs/latest-v24.x/api/cli.html)) |
| `npm i -D nodemon` for restart-on-save | `node --watch app.js` is built in |
| `npm i -D jest`/`mocha` before writing one test | `node --test` — the runner has been **stable since 20.0.0** ([test docs](https://nodejs.org/docs/latest-v24.x/api/test.html)) |
| `npm i node-fetch` | `fetch`, `Response`, `Headers`, `FormData` and `WebSocket` are globals |
| "You can never `require()` an ES module" | `require(esm)` is **stable** as of 24.15.0, for graphs without top-level `await` |
| `npm i -D ts-node` just to run a `.ts` file | `node app.ts` strips types natively — **stable** as of 24.12.0 ([TypeScript docs](https://nodejs.org/docs/latest-v24.x/api/typescript.html)) |
| `--experimental-permission` | `--permission`; the permission model is **stable** ([permissions docs](https://nodejs.org/docs/latest-v24.x/api/permissions.html)) |
| `npm i sqlite3`/`better-sqlite3` for a scratch DB | `node:sqlite` is built in, no flag needed — a release candidate rather than fully stable ([sqlite docs](https://nodejs.org/docs/latest-v24.x/api/sqlite.html)) |
| `url.parse(...)` | Deprecated; use the WHATWG `URL` class ([Node 24 release notes](https://nodejs.org/en/blog/release/v24.0.0)) |
| `corepack enable` as a setup step | Bundled in 24 but experimental, and **not distributed from 25 onward** |
| "Odd-numbered releases never go LTS" | Ends October 2026 — see [§3](#3-orientation--setup) |

The pattern is hard to miss: **a large fraction of the standard Node starter kit from a few years ago is now a dependency you don't need.** Each row gets its proper treatment in the Part that owns it; this table exists so you recognise the outdated advice on sight — and so you don't add four dev dependencies to a project that needs none.

[↑ Back to top](#table-of-contents)

---

## 6. What Changed Under You: Node 22 → 24

[§5](#5-the-delta-that-invalidates-old-tutorials) is about *advice*. This table is about *behaviour you might trip over* when moving a real codebase from the previous LTS ([Node 24 release notes](https://nodejs.org/en/blog/release/v24.0.0)).

| Area | Node 22 | Node 24 |
|---|---|---|
| V8 | 12.4 | **13.6** — `Float16Array`, `RegExp.escape`, `Error.isError`, explicit resource management (`using`) |
| npm | 10 | **11** |
| `AsyncLocalStorage` | `AsyncResource`-based | **`AsyncContextFrame`** by default — faster, and different enough that context-propagation bugs in older APM agents surfaced here |
| undici (powers `fetch`) | 6 | **7** |
| `URLPattern` | import required | **global** |
| Permission model | `--experimental-permission` | `--permission`, stable |
| Type stripping | flagged, experimental | **on by default**, stable as of 24.12.0 |
| Test runner subtests | you had to `await` them yourself | the runner **waits automatically** |
| Windows builds | MSVC | **ClangCL required** |
| macOS floor | older | **13.5+**, Xcode 16.1+ |

Two of these bite silently rather than loudly:

- **`AsyncContextFrame`** changes how context flows across async boundaries. If you have hand-rolled request-context propagation, or an older tracing agent, verify it on 24 rather than assuming — [Part 14](./nodejs-production-diagnostics.md#part-14--diagnostics) covers `AsyncLocalStorage` properly.
- **Type stripping being on by default** means `node app.ts` now *runs* instead of erroring. That is convenient right up until a file uses an `enum` and fails with a syntax error that mentions nothing about TypeScript. [Part 4](./nodejs-modules-packages.md#part-4--modules-cjs-esm-and-typescript) has the full list of unsupported syntax and the `tsconfig.json` that stops you writing it by accident.

[↑ Back to top](#table-of-contents)

---

## 7. Deliberately Not Covered (and where to look instead)

Per the depth-over-breadth rule, these are genuinely important and genuinely out of scope. Half-covering them would be worse than pointing at them.

| Not covered | Why | Where to look |
|---|---|---|
| **Express / Fastify / Hono / Nest** | Framework choice is its own topic; [Part 10](./nodejs-http-networking.md#part-10--http-servers) teaches `node:http` so that any of them stops being magic, and surveys the landscape in one table | [fastify.dev](https://fastify.dev), [hono.dev](https://hono.dev) |
| **ORMs and DB drivers** (Prisma, Drizzle, `pg`) | Database work deserves its own doc set; [Part 13](./nodejs-production-diagnostics.md#part-13--production-node) covers connection lifecycle and shutdown, which is the Node-shaped half of it | This repo's future `postgres/` set; `legacy/postgres/` meanwhile |
| **Deployment platforms** (Vercel, Fly, Lambda) | Platform-specific and fast-rotting | [Part 13](./nodejs-production-diagnostics.md#part-13--production-node) covers the container/signal/config contract they all sit on |
| **Native addons in C++ / Node-API** | A different language and toolchain | [Node-API docs](https://nodejs.org/docs/latest-v24.x/api/n-api.html) |
| **Frontend bundling for the browser** | Not a Node concern, though it runs on Node | [`vite/vite-mastery-guide.md`](../vite/vite-mastery-guide.md) |
| **The Next.js server runtime** | Covered as its own topic | [`nextjs/nextjs-mastery-guide.md`](../nextjs/nextjs-mastery-guide.md) |
| **`node:vm`, embedding, WASI** | Rare enough that a survey would be filler | [vm docs](https://nodejs.org/docs/latest-v24.x/api/vm.html) |

[↑ Back to top](#table-of-contents)

---

## 8. Suggested Learning Order

**The shortest path to being productive** (roughly a weekend):

1. [Part 1 — Running code](./nodejs-runtime-foundations.md#part-1--running-code) — Beginner + Working Knowledge
2. [Part 2 — The event loop](./nodejs-runtime-foundations.md#part-2--the-event-loop) — **all tiers**; this is the one Part where skipping Advanced costs you later
3. [Part 4 — Modules](./nodejs-modules-packages.md#part-4--modules-cjs-esm-and-typescript) — Beginner + Working Knowledge
4. [Part 6 — Files, paths & Buffers](./nodejs-files-streams-buffers.md#part-6--files-paths--buffers) — Beginner + Working Knowledge
5. [Part 10 — HTTP servers](./nodejs-http-networking.md#part-10--http-servers) — Beginner + Working Knowledge
6. [Part 12 — `node:test`](./nodejs-testing-debugging.md#part-12--nodetest--debugging) — Beginner + Working Knowledge

**The full pass**: read the sub-docs in file order — they are sequenced so nothing depends on a later doc. Two notes: [Part 7 (streams)](./nodejs-files-streams-buffers.md#part-7--streams--backpressure) assumes [Part 2](./nodejs-runtime-foundations.md#part-2--the-event-loop), and [Part 14 (diagnostics)](./nodejs-production-diagnostics.md#part-14--diagnostics) is far more useful *after* you have written the servers in Part 10, because it profiles them.

**If you already write Node daily**, the highest-value sections are: the poll phase and starvation in [Part 2](./nodejs-runtime-foundations.md#part-2--the-event-loop), `exports` conditions and the dual-package hazard in [Part 4](./nodejs-modules-packages.md#part-4--modules-cjs-esm-and-typescript), backpressure in [Part 7](./nodejs-files-streams-buffers.md#part-7--streams--backpressure), the timeout matrix in [Part 10](./nodejs-http-networking.md#part-10--http-servers), and all of [Part 14](./nodejs-production-diagnostics.md#part-14--diagnostics).

[↑ Back to top](#table-of-contents)

---

## 9. Quick Self-Check

If you can answer these without looking, you have the material. Each links to where it's covered.

1. Name the six event loop phases in order. Which one can block, and what decides for how long? → [Part 2](./nodejs-runtime-foundations.md#part-2--the-event-loop)
2. `setTimeout(fn, 0)` vs. `setImmediate(fn)` at the top level of a script: which runs first, and why is the honest answer "it depends"? → [Part 2](./nodejs-runtime-foundations.md#part-2--the-event-loop)
3. Why can an unbounded chain of `process.nextTick()` calls hang a server when an unbounded chain of `setImmediate()` calls would not? → [Part 2](./nodejs-runtime-foundations.md#part-2--the-event-loop)
4. Given a `.js` file and no `"type"` field in `package.json`, how does Node decide whether it's CommonJS or ESM? → [Part 4](./nodejs-modules-packages.md#part-4--modules-cjs-esm-and-typescript)
5. What is the dual-package hazard, and what's the recommended way to avoid it? → [Part 4](./nodejs-modules-packages.md#part-4--modules-cjs-esm-and-typescript)
6. Why must `"default"` be the last key in a conditional `exports` object? → [Part 4](./nodejs-modules-packages.md#part-4--modules-cjs-esm-and-typescript)
7. What does `writable.write()` returning `false` actually mean, and what are you supposed to do about it? → [Part 7](./nodejs-files-streams-buffers.md#part-7--streams--backpressure)
8. Why is `pipeline()` preferred over a chain of `.pipe()` calls? → [Part 7](./nodejs-files-streams-buffers.md#part-7--streams--backpressure)
9. When does `exec()` become a security hole that `spawn()` isn't? → [Part 8](./nodejs-processes-workers.md#part-8--the-process)
10. Worker threads or child processes — what's the deciding question? → [Part 9](./nodejs-processes-workers.md#part-9--threads--parallelism)
11. Your container receives `SIGTERM`. What has to happen before the process exits, and what happens if you never handle it? → [Part 13](./nodejs-production-diagnostics.md#part-13--production-node)
12. Memory climbs steadily over 48 hours. What's the first measurement you take, and what produces it? → [Part 14](./nodejs-production-diagnostics.md#part-14--diagnostics)

[↑ Back to top](#table-of-contents)

---

## 10. Cross-Doc Cheat Sheet

The commands and flags that come up across the whole set. Everything here is stable on Node 24 unless marked.

### CLI

| Want | Command |
|---|---|
| Run a file | `node app.js` |
| Run TypeScript directly | `node app.ts` (types stripped, not checked) |
| Restart on change | `node --watch app.js` |
| Load `.env` | `node --env-file=.env app.js` |
| Load `.env` only if present | `node --env-file-if-exists=.env app.js` |
| Run a `package.json` script without npm | `node --run dev` |
| Run tests | `node --test` |
| Tests, watched | `node --test --watch` |
| Coverage | `node --test --experimental-test-coverage` *(experimental)* |
| Debug with Chrome DevTools | `node --inspect app.js` / `--inspect-brk app.js` |
| Preload an ES module | `node --import ./setup.mjs app.js` |
| Preload a CommonJS module | `node --require ./setup.cjs app.js` |
| Readable stack traces from compiled code | `node --enable-source-maps app.js` |
| Stack trace for process warnings | `node --trace-warnings app.js` |
| Raise the heap limit | `node --max-old-space-size=4096 app.js` |
| Sandbox filesystem/process access | `node --permission --allow-fs-read=./data app.js` |
| Set flags globally | `NODE_OPTIONS="--enable-source-maps"` |
| Evaluate an expression | `node -p "process.versions.uv"` |

### Diagnosis, by symptom

| Symptom | First move | Covered in |
|---|---|---|
| Requests stall in bursts | Measure loop delay with `perf_hooks.monitorEventLoopDelay` | [Part 14](./nodejs-production-diagnostics.md#part-14--diagnostics) |
| Memory grows and never falls | Two heap snapshots, compare retained sets | [Part 14](./nodejs-production-diagnostics.md#part-14--diagnostics) |
| One core pinned at 100% | `--cpu-prof`, then read the flame chart | [Part 14](./nodejs-production-diagnostics.md#part-14--diagnostics) |
| `ERR_MODULE_NOT_FOUND` for a package that exists | The package's `exports` map | [Part 4](./nodejs-modules-packages.md#part-4--modules-cjs-esm-and-typescript) |
| `ERR_REQUIRE_ASYNC_MODULE` | Top-level `await` in a required ESM graph | [Part 4](./nodejs-modules-packages.md#part-4--modules-cjs-esm-and-typescript) |
| Writes pile up in RAM until OOM | Ignored backpressure | [Part 7](./nodejs-files-streams-buffers.md#part-7--streams--backpressure) |
| Container takes 30 s to stop | No `SIGTERM` handler | [Part 13](./nodejs-production-diagnostics.md#part-13--production-node) |
| Two module instances of one package | Dual-package hazard | [Part 4](./nodejs-modules-packages.md#part-4--modules-cjs-esm-and-typescript) |
| `EADDRINUSE` on restart | A previous process still holds the socket | [Part 10](./nodejs-http-networking.md#part-10--http-servers) |

### Cross-links in this repo

| Topic | Doc |
|---|---|
| The language itself | [`javascript/javascript-mastery-guide.md`](../javascript/javascript-mastery-guide.md) |
| Async and modules in the language | [`javascript/javascript-errors-async-modules.md`](../javascript/javascript-errors-async-modules.md) |
| Types | [`typescript/typescript-mastery-guide.md`](../typescript/typescript-mastery-guide.md) |
| `tsconfig` and module resolution from the TS side | [`typescript/typescript-modules-ecosystem.md`](../typescript/typescript-modules-ecosystem.md) |
| The other runtime | [`bun/bun-mastery-guide.md`](../bun/bun-mastery-guide.md) |
| Node's role in a frontend build | [`vite/vite-mastery-guide.md`](../vite/vite-mastery-guide.md) |
| Node as a framework server | [`nextjs/nextjs-mastery-guide.md`](../nextjs/nextjs-mastery-guide.md) |

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 1 — Running Code](./nodejs-runtime-foundations.md)

*Written against Node.js 24 LTS · Verified September 14, 2026*
