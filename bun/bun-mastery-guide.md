# Bun — Mastery Guide (Overview)

The entry point for this repo's Bun material. This document holds the shared spec, the runtime primer you need if Node has always been a black box, the setup, the map of the focused sub-docs, the 1.2 → 1.4 delta, and the reference apparatus (learning order, self-check, cross-doc cheat sheet).

The actual teaching lives in the sub-docs. Start here, then follow the [Suggested Learning Order](#8-suggested-learning-order).

---

## About This Document

This section is the canonical spec for **every doc in this set** — the sub-docs each carry a short pointer back here rather than repeating it.

- **Framing: curiosity-driven.** No upcoming task is driving this. The goal is durable, hands-on understanding of Bun itself. The docs are built as a *guided tutorial* — a sequence of runnable examples and **Try It** exercises you are meant to actually type into a terminal, each stating what correct output looks like. Theory ("why is it designed this way") appears only where it explains a mechanism — why the transpiler runs on every import, why the lockfile is text now, why `--hot` keeps `globalThis` — never as a standalone history section.
- **Confirmed scope: Bun as a whole.** The runtime and its module resolution, the Bun-native API surface, `bun install`, `bun test`, `bun build`, `Bun.serve`, the built-in data clients, and enough internals to explain why Bun behaves differently from Node under load.
- **Deliberately excluded** (with pointers): Bun's frontend dev server and HMR pipeline, framework integrations (Next.js, Nuxt, Elysia, Hono), and Bun's hosted/cloud products. `Bun.sql` and the Redis client get a working survey rather than a deep treatment. See [Deliberately Not Covered](#7-deliberately-not-covered-and-where-to-look-instead).
- **TypeScript throughout.** Every example is `.ts`, because Bun runs TypeScript directly and writing the examples in JavaScript would misrepresent how Bun is actually used.
- **Depth tiers.** Each Part runs **Beginner → Working Knowledge → Advanced → Mastery**, so you can stop at whichever level you need today. Tiers are collapsed where one would be filler, and each collapse is stated inline. Collapsing is expected behaviour, not a defect to fix on a later pass. The weight of this set sits in Working Knowledge and Advanced; Mastery concentrates in [Node Compat & Internals](./bun-node-compat-internals.md).
- **Assumed background: modern JavaScript and TypeScript, light on Node.** You are comfortable with ES modules, `async`/`await`, destructuring, and basic types — see [`javascript/javascript-errors-async-modules.md`](../javascript/javascript-errors-async-modules.md) and [`typescript/typescript-foundations.md`](../typescript/typescript-foundations.md) if any of that is shaky, because this set does not re-teach the language. What it *does* teach from zero is the **runtime layer**: what a JavaScript runtime is, how module resolution actually works, what a lockfile is for, what a bundler does. That primer is [§4](#4-runtime-primer-if-node-has-always-been-a-black-box) of this document — read it before the sub-docs if terms like "CommonJS interop" or "hoisted node_modules" are fuzzy.
- **Node contrast callouts.** Short `> **vs. Node**` callouts appear only where the models genuinely diverge (lifecycle scripts blocked by default, `.env` loaded without `dotenv`, JavaScriptCore instead of V8, `Bun.file` being lazy). They are orientation, never a prerequisite — skip them freely if you have never written Node.
- **Per-section ingredients**, used where they earn their place rather than forced into every subsection:
  - a short plain-language explanation of what a thing is and why it matters;
  - a **wrong vs. right** snippet pair wherever a gotcha has a common wrong form;
  - a **Try It** — a runnable exercise with the expected observable output — since this is a curiosity-driven set (production-style "Real Scenario" blocks appear only where a bug genuinely only bites under real load or in CI);
  - an **ASCII diagram** where the concept is spatial or structural — the resolution order, the hoisted vs. isolated `node_modules` layout, the request lifecycle, the bundle graph.
- **Sourcing rule.** [bun.sh/docs](https://bun.sh/docs) is the primary source and is cited inline at the specific claim it supports. Where a popular-but-outdated convention disagrees with the official docs, the official docs win and the doc says so explicitly. **Deviation from the repo template, stated openly:** Stack Overflow was not reachable from the authoring environment on the date below, so no SO links appear in this set. Gotchas are instead sourced to the caveats the official docs state themselves, and to the Bun release blog. If you extend these docs from an environment that *can* reach SO, adding real-world SO citations at the gotchas is a welcome improvement — just don't construct SO URLs from memory.
- **Written against.** **Bun v1.4.2**, the current stable release ([bun.sh/docs](https://bun.sh/docs), [oven-sh/bun releases](https://github.com/oven-sh/bun/releases)). Verified **September 6, 2026**. Bun moves fast — the 1.3 and 1.4 releases changed defaults that most published tutorials still get wrong; see [§6](#6-the-12--14-delta-what-changed-under-you).
- **To update this doc later.** *Preserve*: the curiosity-driven framing, the numbered Part structure with explicit tier subheadings, the wrong-vs-right pairs, the Try It exercises with expected output, the `> **vs. Node**` callout convention, the TOC/back-to-top/anchor apparatus, and the per-Part cheat sheets. *Re-verify against current docs first*: the stable version; **everything in [§6](#6-the-12--14-delta-what-changed-under-you)**, which is the fastest-rotting section here; the Node compatibility status table in [Node Compat & Internals](./bun-node-compat-internals.md) (Bun ships compat fixes nearly every release, so any 🟡 there may now be 🟢); whether HTTP/2 and HTTP/3 in `Bun.serve` are still labelled experimental; whether `bun:ffi` and `Worker` are still labelled experimental; and the `bun test` sharding/parallel flags, which are new in 1.4 and still settling.

---

## Table of Contents

1. [How This Set Is Organised](#1-how-this-set-is-organised)
2. [The Sub-Docs](#2-the-sub-docs)
3. [Orientation & Setup](#3-orientation--setup)
4. [Runtime Primer (if Node has always been a black box)](#4-runtime-primer-if-node-has-always-been-a-black-box)
5. [The One-Paragraph Mental Model](#5-the-one-paragraph-mental-model)
6. [The 1.2 → 1.4 Delta (what changed under you)](#6-the-12--14-delta-what-changed-under-you)
7. [Deliberately Not Covered (and where to look instead)](#7-deliberately-not-covered-and-where-to-look-instead)
8. [Suggested Learning Order](#8-suggested-learning-order)
9. [Quick Self-Check](#9-quick-self-check)
10. [Cross-Doc Cheat Sheet](#10-cross-doc-cheat-sheet)

---

## 1. How This Set Is Organised

Bun is 13 Parts of teaching material spread over 6 sub-docs, plus this overview. The split follows the order in which the concepts actually become necessary — how Bun runs a file first, because everything else (install, test, build, serve) is that same machinery pointed at a different job:

```
  bun-mastery-guide.md   ← you are here: spec, primer, setup, map, delta, reference
        │
        ├── bun-runtime-foundations.md       Parts 1–3    how Bun runs your code
        ├── bun-package-manager.md           Parts 4–5    how your dependencies get there
        ├── bun-apis-io-processes.md         Parts 6–8    talking to files, shells, threads
        ├── bun-servers-networking-data.md   Parts 9–10   serving requests, storing rows
        ├── bun-testing-bundling.md          Parts 11–12  bun test and bun build
        └── bun-node-compat-internals.md     Part 13      why it behaves the way it does
```

Each Part is self-contained enough to reopen on its own, and each ends with a quick-reference table. Within a Part, the tiers mean:

| Tier | What you get |
|---|---|
| **Beginner** | What it is, the minimal correct usage, nothing else |
| **Working Knowledge** | Everyday patterns, the API surface you actually reach for |
| **Advanced** | Edge cases, gotchas, performance and correctness implications |
| **Mastery** | Internals, powerful-but-rare capabilities, expert tradeoffs |

Where a tier would be filler, it is collapsed and the doc says so — e.g. "*`bunx` gets Beginner → Working Knowledge only; there is no Advanced tier that isn't just `bun install`'s resolver, which Part 4 already covers.*"

[↑ Back to top](#table-of-contents)

---

## 2. The Sub-Docs

### [`bun-runtime-foundations.md`](./bun-runtime-foundations.md) — Parts 1–3

| Part | Covers |
|---|---|
| **1. Running code** | `bun run` vs. bare `bun file.ts`, the transpiler, `--watch` vs. `--hot`, the REPL, shebangs and `--bun` |
| **2. Modules & resolution** | ESM/CJS interop, the extension search order, `exports` conditions (including `"bun"`), `tsconfig` paths, `import.meta` |
| **3. Configuration & environment** | `bunfig.toml`, `.env` load order and expansion, `NODE_ENV`, `Bun.env`, runtime plugins via `preload` |

### [`bun-package-manager.md`](./bun-package-manager.md) — Parts 4–5

| Part | Covers |
|---|---|
| **4. `bun install`** | `bun.lock`, resolution, the global cache, hoisted vs. isolated linkers, lifecycle scripts and `trustedDependencies`, `bun ci` |
| **5. Monorepos & the rest of `bun pm`** | Workspaces, catalogs, `--filter`, `bunx`, `bun audit` / `dedupe` / `prune` / `update`, migrating off npm/pnpm/yarn |

### [`bun-apis-io-processes.md`](./bun-apis-io-processes.md) — Parts 6–8

| Part | Covers |
|---|---|
| **6. Files & streams** | `Bun.file`, `Bun.write`, `FileSink`, when you still need `node:fs` |
| **7. Processes & the shell** | `Bun.spawn`, `Bun.spawnSync`, IPC, `Bun.$` and its escaping rules |
| **8. Threads, natives & utilities** | `Worker`, `bun:ffi` / Node-API, `Bun.Glob`, `Bun.password`, hashing, compression |

### [`bun-servers-networking-data.md`](./bun-servers-networking-data.md) — Parts 9–10

| Part | Covers |
|---|---|
| **9. HTTP & sockets** | `Bun.serve` routes and handlers, `req.params`, WebSockets, TLS, `fetch`, TCP/UDP, graceful shutdown |
| **10. Data** | `bun:sqlite` in depth (statements, binding, transactions, WAL), `Bun.sql` and Redis as a survey |

### [`bun-testing-bundling.md`](./bun-testing-bundling.md) — Parts 11–12

| Part | Covers |
|---|---|
| **11. `bun test`** | Discovery, matchers, lifecycle, mocks and `mock.module`, snapshots, coverage, `--parallel` / `--isolate` / `--shard` |
| **12. `bun build`** | `Bun.build` vs. the CLI, targets and formats, splitting, sourcemaps, `external`, plugins, `--compile` binaries |

### [`bun-node-compat-internals.md`](./bun-node-compat-internals.md) — Part 13

| Part | Covers |
|---|---|
| **13. Compat & internals** | The `node:*` support matrix and its real gaps, JavaScriptCore vs. V8, startup and memory behaviour, profiling and debugging |

[↑ Back to top](#table-of-contents)

---

## 3. Orientation & Setup

### Install

Bun ships as a single native binary. There is no Node prerequisite ([installation docs](https://bun.sh/docs/installation)):

```bash
# macOS / Linux / WSL
curl -fsSL https://bun.com/install | bash

# Windows (PowerShell) — needs Windows 10 build 1809 or later
powershell -c "irm bun.sh/install.ps1|iex"
```

Package managers work too: `brew install oven-sh/bun/bun`, `scoop install bun`, or `npm install -g bun` if you already have Node and want the lowest-friction path.

Two requirements the docs call out and people trip over: Linux needs `unzip` present for the installer, and x64 CPUs must be Nehalem-or-newer (Intel) / Bulldozer-or-newer (AMD) — Bun's default build uses modern SIMD instructions. macOS 13.0+ is required.

### Verify

```bash
bun --version    # 1.4.2
bun --revision   # 1.4.2+<commit hash>
```

`bun --revision` is the one to paste into a bug report: two people on "1.4.2" can be on different canary builds.

### Upgrade

```bash
bun upgrade            # self-updating binary
bun upgrade --canary   # latest untested build
bun upgrade --stable   # back to stable
```

If you installed via Homebrew or Scoop, use `brew upgrade bun` / `scoop update bun` instead — `bun upgrade` will tell you so rather than fighting your package manager.

### Your first project

```bash
mkdir bun-playground && cd bun-playground
bun init -y
```

`bun init` scaffolds `package.json`, `index.ts`, `tsconfig.json`, and a `.gitignore`. The `tsconfig.json` it writes is worth reading — it wires up Bun's type definitions, which is what makes `Bun.file` and `bun:test` type-check in your editor. Without them, every `Bun.*` call is a red squiggle even though the code runs fine.

```bash
bun run index.ts   # or just: bun index.ts
```

**Try It — prove there is no build step.** Create `hello.ts` with a type annotation that would need compiling under Node:

```ts
// hello.ts
type Greeting = `Hello, ${string}!`;
const greet = (name: string): Greeting => `Hello, ${name}!`;
console.log(greet("Bun"));
```

```bash
bun hello.ts
```

Expected output: `Hello, Bun!` — no `tsc`, no `ts-node`, no `package.json` `"type"` field, no config. Bun transpiled the file in memory and ran it. It also **stripped** the types rather than checking them: change the call to `greet(42)` and Bun still runs, printing `Hello, 42!`. Type *checking* is `tsc --noEmit`'s job, and it stays your job — see [Part 1](./bun-runtime-foundations.md).

[↑ Back to top](#table-of-contents)

---

## 4. Runtime Primer (if Node has always been a black box)

You know JavaScript. This section is the layer underneath it — the vocabulary the sub-docs assume. If you already write Node daily, skip to [§5](#5-the-one-paragraph-mental-model).

### Engine vs. runtime

A JavaScript **engine** parses and executes JavaScript. It knows about `Array`, `Promise`, and `Math` — and nothing else. It cannot open a file, because files are not part of the JavaScript language.

A **runtime** wraps an engine and adds the outside world: file access, network sockets, timers, process arguments, environment variables. The browser is a runtime (it adds `document` and `fetch`). Node is a runtime. Bun is a runtime.

```
   your code
       │
   ┌───▼─────────────────────────────────────┐
   │  runtime  (Bun / Node / the browser)    │
   │   • file I/O, sockets, timers, env      │
   │   • module loading & resolution         │
   │  ┌───────────────────────────────────┐  │
   │  │  engine                           │  │
   │  │   Bun  → JavaScriptCore (Safari)  │  │
   │  │   Node → V8 (Chrome)              │  │
   │  │   • parse, JIT-compile, GC        │  │
   │  └───────────────────────────────────┘  │
   └─────────────────────────────────────────┘
       │
   operating system
```

That engine difference is not trivia. It is why some Node-specific profiling tools don't work under Bun, why stack traces look slightly different, and why Bun starts faster — [Part 13](./bun-node-compat-internals.md) covers the consequences.

### Modules: the two systems

JavaScript acquired module systems twice, and every runtime has to support both.

- **CommonJS (CJS)** — `require()` / `module.exports`. Synchronous: `require()` returns the finished module before the next line runs. This is the older Node system.
- **ES Modules (ESM)** — `import` / `export`. The standardised system, and asynchronous, which is what allows top-level `await`.

In Node, mixing them is a well-known source of pain: the file extension, the `"type"` field in `package.json`, and the import direction all interact. Bun's headline simplification is that **you can `import` and `require` in the same file** and it just works — with one exception, which is that `require()` of a module using top-level `await` cannot work, because a synchronous call cannot wait ([module docs](https://bun.sh/docs/runtime/modules)). [Part 2](./bun-runtime-foundations.md) has the details.

### Module resolution

When you write `import { x } from "./thing"` or `import express from "express"`, something has to turn that string into a file on disk. That is **resolution**, and it has two very different modes:

- **Relative** (`./thing`) — try a list of extensions in order, then try `thing/index.*`.
- **Bare** (`express`) — walk up the directory tree looking in `node_modules/`, then consult that package's `package.json` to find its entry point.

The rules are fiddly and the failure mode ("Cannot find module") is identical regardless of which rule you tripped, which is why [Part 2](./bun-runtime-foundations.md) spends real time on the order.

### Packages, `node_modules`, and the lockfile

- `package.json` records the dependency **ranges** you asked for (`"react": "^19.0.0"` — "19.x, at least 19.0.0").
- The **lockfile** (`bun.lock`) records the exact versions that range actually resolved to, plus integrity hashes. It is what makes your CI install byte-identical to your laptop install. **Commit it.**
- `node_modules/` is the installed result on disk. It is a build artifact — never commit it, and deleting it is always safe.

The package manager's job is: read ranges → resolve to exact versions (respecting the lockfile) → download → lay out `node_modules/`. [Part 4](./bun-package-manager.md) covers all four steps, including the two very different layouts Bun can produce.

### Transpiling vs. bundling vs. type checking

Three jobs that get conflated:

| Job | Question it answers | In Bun |
|---|---|---|
| **Transpile** | "Turn this TS/JSX into runnable JS" | Automatic, on every import, in memory |
| **Type check** | "Is this program type-correct?" | **Not done.** Run `tsc --noEmit` yourself |
| **Bundle** | "Combine many files into few, for shipping" | `bun build`, explicit ([Part 12](./bun-testing-bundling.md)) |

The important half of that table is row two. Bun deletes types and runs; it never rejects your program for a type error. Treating `bun run` as type-safe is the single most common misconception among people arriving from a `tsc`-based workflow.

[↑ Back to top](#table-of-contents)

---

## 5. The One-Paragraph Mental Model

Bun is one native binary that plays four roles which usually take four tools: it is a **runtime** (a JavaScriptCore engine plus a transpiler that turns TypeScript and JSX into JavaScript in memory on every import, so there is no build step), a **package manager** (`bun install`, resolving into a text `bun.lock` and laying out `node_modules/` from a shared global cache), a **test runner** (`bun test`, Jest-compatible), and a **bundler** (`bun build`, including `--compile` to a single executable). Because those four share one process, one transpiler, and one module resolver, things that are separate concerns in the Node ecosystem collapse into one: your test files are TypeScript with no config, your server reloads without restarting because the module registry lives in the same process that owns the socket, and your `.env` is loaded before your first line runs because the runtime — not a library — reads it. Everything surprising about Bun follows from that single-binary consolidation, and everything *painful* about Bun follows from the fact that the Node ecosystem it must stay compatible with assumed four separate tools.

[↑ Back to top](#table-of-contents)

---

## 6. The 1.2 → 1.4 Delta (what changed under you)

This is the section that rots fastest, and the one that saves you the most time. Bun 1.3 and 1.4 changed defaults that most blog posts still get wrong. When something you read online contradicts this table, check the version it was written for.

| Outdated convention you'll see in the wild | Current reality (1.4.2) |
|---|---|
| "`bun.lockb` is a binary lockfile you can't diff" | The **text `bun.lock`** is the default since 1.2. Binary lockfiles are legacy; migrate with `bun install --save-text-lockfile --frozen-lockfile --lockfile-only`, then delete `bun.lockb` ([install docs](https://bun.sh/docs/cli/install)) |
| "Bun hoists everything into a flat `node_modules`, like npm" | Since 1.3 the **isolated linker (pnpm-style) is the default for workspaces**; hoisted remains the default for single-package projects. Choose explicitly with `--linker hoisted` / `--linker isolated` ([Bun 1.3](https://bun.com/blog/bun-v1.3)) |
| "Use `bun install --frozen-lockfile` in CI" | Still works, but **`bun ci`** is the dedicated command and reads better in a workflow file |
| "`Bun.serve` takes a `fetch` handler and you route by parsing `req.url`" | The declarative **`routes` object landed in 1.2.3** — path params, per-method handlers, static `Response` values. `fetch` is now the fallback for unmatched requests ([HTTP docs](https://bun.sh/docs/api/http)) |
| "Bun only speaks Postgres and SQLite" | 1.3 added a **built-in MySQL client** alongside Postgres and SQLite under the same `Bun.sql` API, plus a first-class **Redis/Valkey client** ([Bun 1.3](https://bun.com/blog/bun-v1.3)) |
| "`bun test` runs everything in one process, single-threaded" | 1.4 added **`--parallel`** (test files across CPU cores), **`--isolate`** (a fresh global per file), **`--shard=i/n`**, and **`--changed`** ([Bun 1.4](https://bun.com/blog/bun-v1.4)) |
| "Bun is written in Zig" | As of 1.4 the release post states plainly: **"Bun is now written in Rust — and this is the first release."** Zig references in older material describe the pre-1.4 implementation ([Bun 1.4](https://bun.com/blog/bun-v1.4)) |
| "You need `node:sea` / `pkg` for a single-file executable" | `bun build --compile` has done this for years, and `node:sea` is explicitly **not implemented** in Bun for exactly that reason ([Node compat](https://bun.sh/docs/runtime/nodejs-apis)) |
| "`bun install` runs postinstall scripts like npm" | **Never has.** Lifecycle scripts are blocked unless the package is listed in `trustedDependencies` — a deliberate supply-chain decision that will silently break packages expecting a build step ([install docs](https://bun.sh/docs/cli/install)) |

Newer capabilities worth knowing exist even if you don't need them yet, all from 1.4 ([release post](https://bun.com/blog/bun-v1.4)): `bun audit fix`, `bun dedupe`, `bun prune`, the `minimumReleaseAge` install setting (refuse packages published in the last N seconds — a real supply-chain mitigation), a global virtual store that makes warm-cache CI installs dramatically faster under `--linker=isolated`, `Bun.cron()`, `Bun.markdown`, `Bun.Image`, `Bun.Terminal` PTY support, and experimental HTTP/3 in `Bun.serve`.

[↑ Back to top](#table-of-contents)

---

## 7. Deliberately Not Covered (and where to look instead)

Per the repo's depth-over-breadth rule, these are named rather than half-taught:

| Not covered | Why | Where to look |
|---|---|---|
| The frontend dev server & HMR (`bun ./index.html`, browser hot reload) | A large surface of its own, and the fastest-moving part of Bun | [Bun bundler docs](https://bun.sh/docs/bundler) |
| Framework integration (Next.js, Nuxt, Elysia, Hono) | Each is really a doc about that framework, not about Bun | The framework's own docs; Bun's [Node compat page](https://bun.sh/docs/runtime/nodejs-apis) for what will break |
| Bun's hosted/cloud products | Product surface, not runtime knowledge | [bun.com](https://bun.com/) |
| `Bun.sql` and Redis in depth | Surveyed in [Part 10](./bun-servers-networking-data.md) — enough to use safely, not a database course | [SQL docs](https://bun.sh/docs/api/sql) |
| `Bun.WebView`, `Bun.Image`, `Bun.markdown` | New in 1.4 and narrow | [Bun 1.4 release post](https://bun.com/blog/bun-v1.4) |
| Deployment & Docker | Deployment practice more than Bun knowledge | The `oven/bun` image ([installation docs](https://bun.sh/docs/installation)); `bun build --compile` is in [Part 12](./bun-testing-bundling.md) |
| JavaScript and TypeScript themselves | Already in this repo | [`javascript/`](../javascript/javascript-mastery-guide.md), [`typescript/`](../typescript/typescript-mastery-guide.md) |

[↑ Back to top](#table-of-contents)

---

## 8. Suggested Learning Order

**If you want to be productive this week** (roughly a day of reading, spread over a week of use):

1. [§3 Setup](#3-orientation--setup) and [§4 Runtime Primer](#4-runtime-primer-if-node-has-always-been-a-black-box) here.
2. [Part 1 — Running code](./bun-runtime-foundations.md), Beginner → Working Knowledge.
3. [Part 4 — `bun install`](./bun-package-manager.md), Beginner → Working Knowledge. Stop before the linker internals.
4. [Part 11 — `bun test`](./bun-testing-bundling.md), Beginner → Working Knowledge.
5. [Part 9 — `Bun.serve`](./bun-servers-networking-data.md), Beginner tier. Build something that responds to a request.

**If you want to actually understand it**, read the Parts in order, 1 → 13. The set is written so each Part only depends on the Parts before it. The one exception: [Part 13](./bun-node-compat-internals.md) is readable at any point and answers "why is this different from Node?", so jump to it whenever that question gets loud.

**If you're evaluating Bun for a real project**, read in this order instead: [§6 the delta](#6-the-12--14-delta-what-changed-under-you) → [Part 13 compat](./bun-node-compat-internals.md) → [Part 4 install](./bun-package-manager.md) → [Part 11 test](./bun-testing-bundling.md). Those four are where a migration succeeds or fails; the API surface is the easy part.

[↑ Back to top](#table-of-contents)

---

## 9. Quick Self-Check

If you can answer these without looking, you have the material. Each links to where it's taught.

1. Why does `bun index.ts` run a file that has a type error in it? ([§4](#4-runtime-primer-if-node-has-always-been-a-black-box), [Part 1](./bun-runtime-foundations.md))
2. What is the difference between `bun --watch` and `bun --hot`, and which one loses your in-memory cache? ([Part 1](./bun-runtime-foundations.md))
3. Given `import x from "./util"`, in what order does Bun try file extensions — and why does `./util.js` resolve to `util.ts`? ([Part 2](./bun-runtime-foundations.md))
4. Which `.env` file wins when both `.env` and `.env.local` set the same key, and what does `NODE_ENV=test` change about that? ([Part 3](./bun-runtime-foundations.md))
5. Why did installing a package that needs a native build step silently produce a broken install, and what one `package.json` field fixes it? ([Part 4](./bun-package-manager.md))
6. What does `--linker isolated` prevent that `--linker hoisted` allows? ([Part 4](./bun-package-manager.md))
7. `Bun.file("huge.log")` returns instantly on a 4 GB file. Why? ([Part 6](./bun-apis-io-processes.md))
8. Why is ``await $`rm -rf ${userInput}` `` not the injection disaster it looks like? ([Part 7](./bun-apis-io-processes.md))
9. In `Bun.serve`, when does the `fetch` handler run if you have also supplied `routes`? ([Part 9](./bun-servers-networking-data.md))
10. In `bun:sqlite`, what is the difference between `.query()` and `.prepare()`, and when does that difference cost you? ([Part 10](./bun-servers-networking-data.md))
11. Why must `mock.module()` usually go in a `--preload` file rather than at the top of the test? ([Part 11](./bun-testing-bundling.md))
12. What does `target: "bun"` change about the output of `bun build` compared to `target: "node"`? ([Part 12](./bun-testing-bundling.md))
13. Name two `node:` APIs that will not behave correctly under Bun, and say what to use instead. ([Part 13](./bun-node-compat-internals.md))

[↑ Back to top](#table-of-contents)

---

## 10. Cross-Doc Cheat Sheet

### The commands, in one place

| Command | Does |
|---|---|
| `bun index.ts` | Run a file (`run` is optional for files) |
| `bun run dev` | Run a `package.json` script |
| `bun --watch index.ts` | Re-run on change, hard restart |
| `bun --hot server.ts` | Reload modules in place, keep `globalThis` and open sockets |
| `bun run --bun vite` | Force a `#!/usr/bin/env node` CLI to run under Bun |
| `bun install` | Install from `package.json` + `bun.lock` |
| `bun ci` | Install, failing if `package.json` and the lockfile disagree |
| `bun add pkg` / `bun remove pkg` | Add/remove a dependency |
| `bunx cowsay hi` | Run a package binary without installing it |
| `bun test` | Run tests |
| `bun build ./cli.ts --compile --outfile mycli` | Bundle to a standalone executable |
| `bun repl` | Interactive shell |
| `bun pm cache rm` | Clear the global package cache |

### Flag placement — the one that bites everyone

```bash
bun --watch run dev   # ✅ runtime flag before `run`
bun run dev --watch   # ❌ `--watch` is passed to your script, not to Bun
```

Runtime flags belong to `bun`, so they go **before** the subcommand ([run docs](https://bun.sh/docs/cli/run)).

### Bun API → Node equivalent

| Bun | Node equivalent | Note |
|---|---|---|
| `Bun.file(path)` | `fs.readFile` | Lazy — nothing is read until you `await` a body method |
| `Bun.write(dest, data)` | `fs.writeFile` | Picks the fastest syscall (`copy_file_range`, `sendfile`) per platform |
| `Bun.serve({...})` | `http.createServer` | Web-standard `Request`/`Response`, not `(req, res)` |
| `Bun.spawn([...])` | `child_process.spawn` | Output is a web `ReadableStream` |
| ``Bun.$`cmd` `` | `execSync` / `zx` | Not a system shell — a built-in cross-platform one |
| `bun:sqlite` | `better-sqlite3` | Built in; no native build step |
| `Bun.env` | `process.env` | Aliases of the same object; `.env` files load automatically |
| `bun:test` | `jest` / `vitest` | Jest-compatible API, plus a `vi` alias for Vitest habits |
| `Bun.password` | `bcrypt` | Argon2id by default |
| `Bun.Glob` | `glob` package | Built in |
| — | `node:sea` | Not implemented; use `bun build --compile` |

### The five defaults that surprise people

1. **Lifecycle scripts don't run** unless the package is in `trustedDependencies`.
2. **`.env` is loaded automatically** — no `dotenv`, and load order is `.env` → `.env.<NODE_ENV>` → `.env.local` → `.env.<NODE_ENV>.local`.
3. **Types are stripped, never checked.**
4. **Workspaces install isolated, single packages install hoisted** — the same repo can behave differently before and after you add a `workspaces` field.
5. **`bun run` respects `#!/usr/bin/env node` shebangs** and will genuinely shell out to Node unless you pass `--bun`.

[↑ Back to top](#table-of-contents)

---

*Written against Bun v1.4.2 · Verified September 6, 2026 · Part of [my-learning-md-docs](../README.md)*
