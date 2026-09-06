# Bun — Runtime Foundations (Parts 1–3)

How Bun turns a file on disk into a running program: the transpiler and the `bun run` surface, module resolution, and the configuration that silently shapes both.

> **Spec:** this doc follows the shared spec in [`bun-mastery-guide.md`](./bun-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Mastery tiers (collapsed where a tier would be filler, always stated), **Try It** exercises with expected output, wrong-vs-right pairs, official docs cited inline. Written against **Bun v1.4.2**, verified **September 6, 2026**.
>
> **Prerequisite:** if "CommonJS interop" or "module resolution" are fuzzy terms, read the [Runtime Primer](./bun-mastery-guide.md#4-runtime-primer-if-node-has-always-been-a-black-box) in the overview first. This doc assumes it.

---

## Table of Contents

- [Part 1 — Running Code](#part-1--running-code)
  - [Beginner: the two ways to run something](#p1-beginner)
  - [Working Knowledge: scripts, `--bun`, and the transpiler](#p1-working)
  - [Advanced: `--watch` vs. `--hot`](#p1-advanced)
  - [Mastery: what the transpiler will and won't do](#p1-mastery)
  - [Part 1 cheat sheet](#part-1-cheat-sheet)
- [Part 2 — Modules & Resolution](#part-2--modules--resolution)
  - [Beginner: import and require in the same file](#p2-beginner)
  - [Working Knowledge: the resolution order](#p2-working)
  - [Advanced: `exports` conditions and path mapping](#p2-advanced)
  - [Mastery: `import.meta`, custom conditions, `Bun.resolveSync`](#p2-mastery)
  - [Part 2 cheat sheet](#part-2-cheat-sheet)
- [Part 3 — Configuration & Environment](#part-3--configuration--environment)
  - [Beginner: `.env` just works](#p3-beginner)
  - [Working Knowledge: load order, expansion, `NODE_ENV`](#p3-working)
  - [Advanced: `bunfig.toml`](#p3-advanced)
  - [Mastery: `preload` and runtime plugins](#p3-mastery)
  - [Part 3 cheat sheet](#part-3-cheat-sheet)

---

## Part 1 — Running Code

The whole of Bun's runtime pitch is in one sentence: you point it at a TypeScript file and it runs, because transpiling happens in memory on every import. Everything in this Part is a consequence.

<a id="p1-beginner"></a>

### Beginner: the two ways to run something

There are exactly two things `bun run` can be given — a **file** or a **`package.json` script**:

```bash
bun run index.ts     # a file
bun index.ts         # same thing; `run` is optional for files
bun run dev          # a script from package.json
```

`.ts`, `.tsx`, `.jsx`, `.js`, `.mjs`, `.cjs` all run directly, with no configuration and no build step ([run docs](https://bun.sh/docs/cli/run)).

To see what scripts exist, run `bun run` with no argument — it prints every script in `package.json` with its command.

**Try It — the no-config claim, tested.** In an empty directory (no `package.json`, no `tsconfig.json`):

```ts
// t.ts
enum Color { Red, Green }
const pick = <T,>(xs: T[]): T => xs[0]!;
console.log(Color[pick([Color.Green, Color.Red])]);
```

```bash
bun t.ts
```

Expected output: `Green`. That file uses an `enum` and a generic arrow function — both of which require a compiler under Node. No `package.json` was needed at all.

<a id="p1-working"></a>

### Working Knowledge: scripts, `--bun`, and the transpiler

#### Script resolution order

When you type `bun run thing`, Bun looks in this order ([run docs](https://bun.sh/docs/cli/run)):

```
  bun run thing
        │
        ├─ 1. a "thing" script in package.json      ← usually this
        ├─ 2. a source file named thing
        ├─ 3. a binary from node_modules/.bin       ← project package binaries
        └─ 4. a system command on your PATH
```

This is why a script named `test` shadows the `bun test` command in some contexts, and why `bun run tsc` works without `npx`.

#### The `--bun` flag, and why your `dev` script still uses Node

This surprises everyone. Most CLI tools in `node_modules/.bin` start with `#!/usr/bin/env node`. **Bun respects that shebang** and genuinely launches Node to run them. So this:

```jsonc
{ "scripts": { "dev": "vite" } }
```

...runs Vite on Node, even though you typed `bun run dev`. To force Bun's runtime:

```bash
bun run --bun vite
```

```jsonc
// or bake it into the script
{ "scripts": { "dev": "bun --bun vite" } }
```

> **vs. Node** — there is no equivalent of this problem in npm, because npm has only one runtime to offer. Under Bun, "am I actually running on Bun right now?" is a real question with a real answer: check `process.versions.bun`.

```ts
if (process.versions.bun) console.log(`Bun ${process.versions.bun}`);
else console.log("running on Node");
```

#### Flag placement

The single most common Bun CLI mistake:

```bash
# ❌ wrong — --watch is handed to your script as an argument
bun run dev --watch

# ✅ right — runtime flags belong to bun, so they come first
bun --watch run dev
```

The docs call this out explicitly ([run docs](https://bun.sh/docs/cli/run)). The wrong form fails *silently*: your script runs once, normally, and you sit there wondering why saving a file does nothing.

#### The REPL

```bash
bun repl
```

TypeScript works in it, and so do top-level `await` and `Bun.*`. Handy for checking a resolution question in five seconds rather than creating a scratch file.

<a id="p1-advanced"></a>

### Advanced: `--watch` vs. `--hot`

Two reload modes that look interchangeable and are not ([hot reload docs](https://bun.sh/docs/runtime/hot)):

| | `--watch` | `--hot` |
|---|---|---|
| Mechanism | **Hard restart** of the process | **Soft reload** of changed modules, same process |
| `globalThis` | Wiped | **Preserved** |
| Open sockets / server | Closed and rebound | Stay open |
| In-memory caches, DB pools | Lost | Kept |
| Use for | Tests, builds, CLI scripts | Long-running HTTP servers |

```
  --watch                          --hot
  ┌──────────┐  file change        ┌──────────┐  file change
  │ process  │ ──────► ✕ kill      │ process  │ ──────► re-evaluate module
  │  state   │         ↺ respawn   │  state   │         (process lives on)
  └──────────┘         fresh state └──────────┘         state survives
```

The `--hot` behaviour is deliberate and useful: your server keeps its port, its WebSocket connections, and its connection pool while you edit a request handler.

**Try It — watch state survive a reload.**

```ts
// server.ts
globalThis.count ??= 0;
globalThis.count++;

Bun.serve({
  port: 3000,
  fetch: () => new Response(`reloaded ${globalThis.count} times\n`),
});
console.log("listening on http://localhost:3000");
```

```bash
bun --hot server.ts
```

Hit `curl localhost:3000` — you get `reloaded 1 times`. Now edit the file (change the response text, save) and curl again: the count **increments** rather than resetting, and you never saw "listening on..." print twice or an `EADDRINUSE` error. Re-run the same file under `bun --watch server.ts` and repeat: the count goes back to `1` every save and the startup log reprints. That difference *is* the feature.

#### The `--hot` gotchas the docs name

1. **Startup-only code doesn't re-run the way you expect.** If you changed the code that builds a singleton at import time, the old singleton may still be the live one. When in doubt, restart.
2. **Memory creeps.** `--hot` garbage-collects but doesn't clean up everything, so it is a development tool, not a production deployment strategy.
3. **It is server-side only.** Browser hot reload is a different mechanism (Bun's frontend dev server, out of scope for this set — see [Deliberately Not Covered](./bun-mastery-guide.md#7-deliberately-not-covered-and-where-to-look-instead)).

<a id="p1-mastery"></a>

### Mastery: what the transpiler will and won't do

Bun's transpiler runs on every file it loads, and it is a **transpiler, not a type checker**. Understanding the exact boundary prevents a whole class of confusion.

**What it does:** strips type annotations, compiles `enum`, JSX, decorators, and modern syntax down to what JavaScriptCore runs; applies `define` substitutions; handles `.tsx` and `.jsx`.

**What it does not do:** check types, resolve `declare` merging, honour `tsconfig` options that affect *emit semantics* beyond the basics, or fail on a type error.

```ts
// ❌ what people assume
const n: number = "not a number";  // "Bun will catch this at runtime"
```

```ts
// ✅ what actually happens
const n: number = "not a number";  // Bun deletes `: number` and runs.
                                   // n is the string. No error, ever.
```

The fix is not a Bun flag — it is running the real type checker as a separate step:

```jsonc
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "check": "bun run typecheck && bun test"
  }
}
```

**One `tsconfig.json` setting genuinely matters at runtime:** `verbatimModuleSyntax` (or, in older configs, `importsNotUsedAsValues`). Because Bun strips types per-file without whole-program knowledge, an import used only as a type can, in edge cases, be preserved and cause a side-effectful module to load. Writing `import type { Foo } from "./foo"` explicitly makes the intent unambiguous and is good practice regardless of runtime — see [`typescript/typescript-modules-ecosystem.md`](../typescript/typescript-modules-ecosystem.md).

#### Bun.Transpiler — the transpiler as an API

The same machinery is exposed directly, which is occasionally the neatest way to scan a codebase for its imports without running it:

```ts
const transpiler = new Bun.Transpiler({ loader: "tsx" });
const { imports, exports } = transpiler.scan(await Bun.file("./app.tsx").text());
console.log(imports); // [{ path: "react", kind: "import-statement" }, ...]
```

<a id="part-1-cheat-sheet"></a>

### Part 1 cheat sheet

| Want | Command |
|---|---|
| Run a file | `bun index.ts` |
| Run a script | `bun run dev` |
| List scripts | `bun run` |
| Restart on change | `bun --watch index.ts` |
| Reload without restarting | `bun --hot server.ts` |
| Force Bun over a node shebang | `bun run --bun vite` |
| Am I on Bun? | `process.versions.bun` |
| Actually check types | `tsc --noEmit` (separate step) |
| REPL | `bun repl` |

[↑ Back to top](#table-of-contents)

---

## Part 2 — Modules & Resolution

Resolution is the machinery that turns the string in your `import` into a file. It is invisible until it fails, and then it is the whole problem.

<a id="p2-beginner"></a>

### Beginner: import and require in the same file

Bun supports ESM and CommonJS simultaneously, in the same file, with no `"type"` field and no configuration ([module docs](https://bun.sh/docs/runtime/modules)):

```ts
import { readFileSync } from "node:fs";   // ESM
const path = require("node:path");        // CJS — in the same file
export const here = path.resolve(".");
```

What each form gives you when the two systems meet:

| Importing a… | via `require()` | via `import * as` |
|---|---|---|
| **ES Module** | the module namespace | the module namespace |
| **CommonJS module** | `module.exports` | `default` is `module.exports`; its keys also become named exports |

**The one hard limit:** you cannot `require()` a module that uses top-level `await`. `require()` must return synchronously; top-level `await` means the module isn't finished. There is no workaround other than using `import`.

<a id="p2-working"></a>

### Working Knowledge: the resolution order

Two different algorithms depending on the specifier shape.

#### Relative specifiers (`./thing`)

Bun tries extensions in a fixed order ([module docs](https://bun.sh/docs/runtime/modules)):

```
  import x from "./hello"
        │
        ├─ 1. ESM extensions:  .tsx  .jsx  .mts  .ts  .mjs  .js
        ├─ 2. CJS extensions:  .cts  .cjs
        ├─ 3. data:            .json
        └─ 4. directory index: ./hello/index.<same order>
```

Note that **`.tsx` and `.ts` are tried before `.js`**. If you have both `hello.ts` and a stale compiled `hello.js` sitting in the same directory, Bun runs the TypeScript one — the opposite of what a `tsc`-built workflow would do, and a genuinely confusing debugging session if you don't know it.

#### The `.js` → `.ts` substitution

This one looks like a bug and is a deliberate TypeScript-compatibility rule:

```ts
import { hello } from "./hello.js";   // resolves to ./hello.ts if that's what exists
```

TypeScript's `NodeNext` module mode requires you to write the `.js` extension in source even though the file on disk is `.ts` (because that is what the *output* will be called). Bun honours that convention, so ESM-correct TypeScript source runs unmodified.

#### Bare specifiers (`express`)

Standard Node resolution: walk up the directory tree checking each `node_modules/`, then read that package's `package.json` to find the entry point. Bun checks entry fields in this priority ([module docs](https://bun.sh/docs/runtime/modules)):

```
  "exports" conditions:  "bun" → "node"/"node-addons" → "require"/"import" → "default"
        ↓ (if no "exports" field)
  legacy fields:         "main" → "module" → implicit index.*
```

`NODE_PATH` is also supported for extra resolution roots:

```bash
NODE_PATH=./packages:./lib bun run src/index.ts
```

<a id="p2-advanced"></a>

### Advanced: `exports` conditions and path mapping

#### The `"bun"` condition

A package can ship un-transpiled TypeScript specifically for Bun:

```jsonc
{
  "exports": {
    ".": {
      "bun": "./index.ts",       // Bun takes this — original TS source
      "import": "./index.mjs",   // other ESM consumers
      "require": "./index.js",   // CJS consumers
      "default": "./index.js"
    }
  }
}
```

This is genuinely useful when publishing a library: consumers on Bun get source (better stack traces, no sourcemap indirection), everyone else gets the built artifacts.

#### Subpath exports are a wall, not a suggestion

```ts
// package "foo" declares exports for "." only
import pkg from "foo";          // ✅ works
import util from "foo/utils";   // ❌ fails — not exported
```

Bun enforces this the same way Node ≥18 does. Deep-importing into a package's internals (`lodash/fp/curry`-style) only works if that package explicitly exports the subpath. If a tutorial written before `exports` became widespread tells you to deep-import, it is out of date — and the failure is the package's policy, not Bun's bug.

#### Path re-mapping

Two mechanisms, both supported ([module docs](https://bun.sh/docs/runtime/modules)):

```jsonc
// tsconfig.json — the TypeScript way
{
  "compilerOptions": {
    "paths": {
      "@config": ["./src/config.ts"],
      "@components/*": ["./src/components/*"]
    }
  }
}
```

```jsonc
// package.json — the Node-standard way, works without TypeScript
{
  "imports": {
    "#config": "./src/config.ts",
    "#components/*": "./src/components/*"
  }
}
```

Bun reads `tsconfig.json` `paths` **at runtime** — you don't need a bundler or a `tsconfig-paths` loader to make aliases work. That is a real quality-of-life difference from Node, where `paths` is a type-only fiction unless something rewrites it.

> **vs. Node** — the number one reason `tsconfig` aliases "work in my editor but crash at runtime" under Node is that `tsc` only checks them; nothing rewrites the emitted import. Bun resolves them for real.

#### Real Scenario — the phantom stale `.js`

A build script once wrote compiled output next to source (`src/db.js` beside `src/db.ts`), then the build was removed from the workflow but the artifact stayed, untracked-but-present, on one developer's machine. Under `tsc`+Node the running code was `db.js` — the compiled artifact. Under Bun, resolution order picks `.ts` first, so Bun ran the *source*, which had since drifted. Same repo, same command, two different programs, and a bug reproducible on exactly one machine. The fix is dull and permanent: never emit next to source (`outDir`), and `.gitignore` compiled output.

<a id="p2-mastery"></a>

### Mastery: `import.meta`, custom conditions, `Bun.resolveSync`

#### `import.meta`

Bun's `import.meta` is richer than Node's ([module docs](https://bun.sh/docs/runtime/modules)):

```ts
import.meta.dir       // directory containing this file
import.meta.file      // "index.ts"
import.meta.path      // absolute path to this file
import.meta.url       // file:// URL
import.meta.main      // true if this file is the entry point
import.meta.resolve() // resolve a specifier to a URL
```

`import.meta.main` is the ESM answer to Python's `if __name__ == "__main__"`, and the idiomatic way to write a module that is also a CLI:

```ts
export function run(args: string[]) { /* ... */ }

if (import.meta.main) run(Bun.argv.slice(2));
```

`import.meta.dir` deserves a note: under ESM, Node gives you neither `__dirname` nor a clean replacement (you assemble one from `fileURLToPath(import.meta.url)`). Bun just gives you the directory.

#### Custom export conditions

```bash
bun build --conditions="react-server" ./app/route.ts
```

This is how frameworks ship different code to a server component graph than to the client. If you have ever wondered how one import specifier yields different modules in a React Server Components app, this is the mechanism.

#### Resolving by hand

```ts
Bun.resolveSync("./utils", import.meta.dir);   // → absolute path, or throws
```

Useful in plugins and tooling where you need to answer "what would this import resolve to?" without importing it (and running its side effects).

<a id="part-2-cheat-sheet"></a>

### Part 2 cheat sheet

| Question | Answer |
|---|---|
| Extension order, relative import | `.tsx` `.jsx` `.mts` `.ts` `.mjs` `.js` `.cts` `.cjs` `.json`, then `index.*` |
| Does `./x.js` find `x.ts`? | Yes — deliberate TypeScript compatibility |
| Can I mix `import` and `require`? | Yes, in the same file |
| The exception? | `require()` of a module with top-level `await` |
| `exports` condition priority | `bun` → `node` → `require`/`import` → `default` |
| Do `tsconfig` `paths` work at runtime? | Yes, natively |
| `__dirname` under ESM | `import.meta.dir` |
| "Is this the entry point?" | `import.meta.main` |
| Resolve without importing | `Bun.resolveSync(spec, dir)` |

[↑ Back to top](#table-of-contents)

---

## Part 3 — Configuration & Environment

*This Part collapses Beginner and Working Knowledge into environment variables, because that is the only configuration most projects ever touch, and gives Advanced/Mastery to `bunfig.toml` and preloading — which are where the surprising behaviour lives.*

<a id="p3-beginner"></a>

### Beginner: `.env` just works

Create a `.env`:

```bash
API_TOKEN=secret
```

Read it — three equivalent aliases for the same object ([env docs](https://bun.sh/docs/runtime/env)):

```ts
process.env.API_TOKEN;      // "secret"
Bun.env.API_TOKEN;          // "secret"
import.meta.env.API_TOKEN;  // "secret"
```

No `dotenv`, no `import "dotenv/config"` at the top of your entry file, no ordering hazard where a module reads config before dotenv ran.

> **vs. Node** — this is one of the changes you feel every day. Node requires the `dotenv` package (or `--env-file`, added recently), and the classic Node bug is a module that reads `process.env.X` at import time, before `dotenv.config()` executed. Bun loads `.env` **before your first line of code**, so that bug cannot happen.

<a id="p3-working"></a>

### Working Knowledge: load order, expansion, `NODE_ENV`

#### Load order

Later files override earlier ones ([env docs](https://bun.sh/docs/runtime/env)):

```
  1.  .env
  2.  .env.production | .env.development | .env.test        ← by NODE_ENV
  3.  .env.local                                            ← SKIPPED when NODE_ENV=test
  4.  .env.production.local | .env.development.local | .env.test.local
```

Two things to internalise:

- `.env.local` **is deliberately skipped when `NODE_ENV=test`**. This is intentional — your personal local overrides should not silently change test results — and it is a genuinely baffling half-hour if you don't know it. If a value appears in dev and vanishes in tests, this is why.
- Actual shell environment variables still win over everything in a file. `API_TOKEN=x bun run start` beats any `.env`.

#### Expansion

Bun expands variables inside `.env` automatically (what `dotenv-expand` does for Node):

```bash
FOO=world
BAR=hello$FOO      # → "helloworld"
BAZ=hello\$FOO     # → "hello$FOO"  (escaped)
```

That backslash escape matters for a very concrete case: **passwords and connection strings containing `$`**.

```bash
# ❌ silently mangled — everything from $ onward is treated as a variable
DATABASE_URL=postgres://user:pa$$word@host/db

# ✅ escaped
DATABASE_URL=postgres://user:pa\$\$word@host/db
```

The wrong form doesn't error. It produces a connection string with a chunk missing, and you get an authentication failure that looks like a credentials problem.

#### Manual control

```bash
bun --env-file=.env.staging src/index.ts   # override which file(s) load
bun run --no-env-file index.ts             # disable automatic loading entirely
```

#### Typing your environment

`process.env.FOO` is `string | undefined`, which is correct but tedious. Declare the vars you rely on:

```ts
// env.d.ts
declare module "bun" {
  interface Env {
    API_TOKEN: string;
    DATABASE_URL: string;
  }
}
```

Now `Bun.env.API_TOKEN` is `string`. This is a claim you are making, not a check Bun performs — if the variable is missing at runtime it is still `undefined`, and the type lied. For anything that must be present, validate at startup:

```ts
const token = Bun.env.API_TOKEN ?? (() => { throw new Error("API_TOKEN is required"); })();
```

<a id="p3-advanced"></a>

### Advanced: `bunfig.toml`

`bunfig.toml` configures Bun itself, as opposed to `package.json`, which describes your package. It is read from the project directory, with a global `~/.bunfig.toml` underneath it (project wins) ([bunfig docs](https://bun.sh/docs/runtime/bunfig)).

```toml
# bunfig.toml — a realistic starting point
preload = ["./src/instrument.ts"]      # runs before everything
logLevel = "warn"
telemetry = false

[test]
preload = ["./test/setup.ts"]
coverage = true
coverageThreshold = 0.9
coverageReporter = ["text", "lcov"]

[install]
exact = true                            # save "1.2.3", not "^1.2.3"
linker = "isolated"
minimumReleaseAge = 259200              # refuse packages published <3 days ago
minimumReleaseAgeExcludes = ["@types/node", "typescript"]

[run]
bun = true                              # auto-alias node → bun for scripts
```

The keys worth knowing, by section:

| Section | Keys that actually come up |
|---|---|
| top level | `preload`, `jsx`, `define`, `loader`, `logLevel`, `smol`, `telemetry`, `env` |
| `[test]` | `preload`, `coverage`, `coverageThreshold`, `coverageReporter`, `randomize`, `seed`, `retry`, `reporter` |
| `[install]` | `exact`, `linker`, `frozenLockfile`, `production`, `registry`, `scopes`, `cache`, `minimumReleaseAge`, `security.scanner` |
| `[run]` | `bun`, `shell`, `silent`, `elide-lines` |
| `[serve]` | `port` |

Three that pay for themselves immediately:

- **`[run] bun = true`** — makes `bun run <script>` use Bun even for `#!/usr/bin/env node` binaries, so you stop typing `--bun`. Set it deliberately: it changes which runtime your whole toolchain executes on.
- **`[install] exact = true`** — no caret ranges in `package.json`. Combined with a committed lockfile, this is the strictest reproducibility posture.
- **`[install] minimumReleaseAge`** — refuses to install a version published within the last N seconds. Most npm supply-chain compromises are caught and unpublished within hours; a 3-day quarantine converts a large class of them into a non-event. This is one of the genuinely good ideas Bun shipped in 1.4 that has no npm equivalent.

`smol` deserves a note: it reduces memory at the cost of throughput by telling JavaScriptCore to use a small heap. It is the right default in a memory-capped container and the wrong default on a workstation.

<a id="p3-mastery"></a>

### Mastery: `preload` and runtime plugins

`preload` runs a module before your entry point ([bunfig docs](https://bun.sh/docs/runtime/bunfig)). Two uses: instrumentation, and extending module loading.

#### Instrumentation

```toml
preload = ["./src/instrument.ts"]
```

APM agents, `AsyncLocalStorage` context setup, and global error handlers all need to be installed before anything else imports. This is that hook.

#### Runtime plugins

The same plugin API powers the runtime and the bundler ([plugin docs](https://bun.sh/docs/runtime/plugins)). Registered via `preload`, a plugin can teach Bun's *runtime* to import file types it doesn't know:

```ts
// plugins/yaml.ts
import { plugin } from "bun";
import { parse } from "yaml";

plugin({
  name: "yaml loader",
  setup(build) {
    build.onLoad({ filter: /\.ya?ml$/ }, async ({ path }) => ({
      contents: `export default ${JSON.stringify(parse(await Bun.file(path).text()))}`,
      loader: "js",
    }));
  },
});
```

```toml
# bunfig.toml
preload = ["./plugins/yaml.ts"]
```

```ts
import config from "./config.yaml";   // now works at runtime, no build step
console.log(config.database.host);
```

The two hooks:

- **`onResolve({ filter, namespace })`** — intercept *which file* a specifier maps to.
- **`onLoad({ filter, namespace })`** — intercept *what a file's contents are*, returning `contents` plus a `loader` (`"js"`, `"ts"`, `"tsx"`, `"json"`, `"toml"`, `"yaml"`, `"text"`, `"css"`, `"wasm"`, `"file"`, `"napi"`, `"html"`, `"jsonc"`).

`filter` is a regex matched against the path, and it is worth writing tightly: a `filter: /.*/` plugin is invoked for every module in your dependency graph and will be visible in startup time.

Namespaces let a plugin invent module specifiers that don't correspond to files at all:

```ts
build.onResolve({ filter: /^env$/ }, () => ({ path: "env", namespace: "env" }));
build.onLoad({ filter: /.*/, namespace: "env" }, () => ({
  contents: `export default ${JSON.stringify(process.env)}`,
  loader: "js",
}));
// import env from "env";  ← a module that exists only in the plugin
```

**Bundler plugins get more hooks** (`onStart`, and the native `onBeforeParse` implemented as a NAPI module) — those are covered in [Part 12](./bun-testing-bundling.md#part-12--bun-build). A plugin using only `onResolve`/`onLoad` works in both contexts unchanged.

<a id="part-3-cheat-sheet"></a>

### Part 3 cheat sheet

| Want | How |
|---|---|
| Read an env var | `Bun.env.X` / `process.env.X` / `import.meta.env.X` |
| `.env` precedence | `.env` → `.env.<NODE_ENV>` → `.env.local` → `.env.<NODE_ENV>.local` |
| Why is `.env.local` ignored? | `NODE_ENV=test` skips it, by design |
| Escape `$` in a value | `pa\$\$word` |
| Use a specific env file | `bun --env-file=.env.staging app.ts` |
| Turn env loading off | `bun run --no-env-file app.ts` |
| Type env vars | `declare module "bun" { interface Env { … } }` |
| Configure Bun | `bunfig.toml` (project) / `~/.bunfig.toml` (global) |
| Run code before everything | `preload = ["./setup.ts"]` |
| Teach imports a new file type | `plugin({ setup(b) { b.onLoad(…) } })` via `preload` |
| Quarantine fresh npm releases | `[install] minimumReleaseAge = 259200` |
| Stop typing `--bun` | `[run] bun = true` |

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 4 — `bun install`](./bun-package-manager.md) · **Up:** [Overview](./bun-mastery-guide.md)

*Written against Bun v1.4.2 · Verified September 6, 2026*
