# Node.js — Modules & Packages (Parts 4–5)

How a specifier like `'./util.js'` or `'lodash'` becomes actual code: the two module systems and the rules that decide which one you're in, the `exports` map that packages use to control what you can reach, native TypeScript support, and the npm surface that puts the files on disk in the first place.

> **Spec:** this doc follows the shared spec in [`nodejs-mastery-guide.md`](./nodejs-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Mastery tiers (collapsed where a tier would be filler, always stated), **Try It** exercises with expected output, wrong-vs-right pairs, official docs cited inline. Written against **Node.js 24 LTS**, verified **September 14, 2026**.
>
> **Prerequisite:** ES module *syntax* (`import`, `export`, dynamic `import()`) is a language feature and is not re-taught here — see [`javascript/javascript-errors-async-modules.md`](../javascript/javascript-errors-async-modules.md). This Part is about what Node does with that syntax.

---

## Table of Contents

- [Part 4 — Modules: CJS, ESM and TypeScript](#part-4--modules-cjs-esm-and-typescript)
  - [Beginner: two systems, and how Node picks one](#p4-beginner)
  - [Working Knowledge: resolution, interop, and the ESM rules that differ](#p4-working)
  - [Advanced: `exports` maps, conditions, and the dual-package hazard](#p4-advanced)
  - [Mastery: `require(esm)`, loader hooks, and the compile cache](#p4-mastery)
  - [Part 4 cheat sheet](#part-4-cheat-sheet)
- [Part 5 — Packages & the npm Surface](#part-5--packages--the-npm-surface)
  - [Beginner: `package.json`, install, lockfile](#p5-beginner)
  - [Working Knowledge: semver, `npm ci`, scripts, `npx`](#p5-working)
  - [Advanced: workspaces, `overrides`, peer dependencies, publishing](#p5-advanced)
  - [Mastery: supply-chain hygiene](#p5-mastery)
  - [Part 5 cheat sheet](#part-5-cheat-sheet)

---

## Part 4 — Modules: CJS, ESM and TypeScript

Node has two module systems, and it will have two module systems forever. The good news is that the decade-long standoff between them ended recently: `require()` can now load ES modules, and Node can run TypeScript without a build step. The rules are finally learnable in an afternoon — which is a change, and is why most of what you'll find written about this is out of date.

<a id="p4-beginner"></a>

### Beginner: two systems, and how Node picks one

**CommonJS** is Node's original system. It is synchronous, and it is what you see in every older codebase:

```js
// math.cjs
function add(a, b) { return a + b; }
module.exports = { add };

// app.cjs
const { add } = require('./math.cjs');
```

**ES modules** are the standard, shared with browsers, and what new code should use:

```js
// math.mjs
export function add(a, b) { return a + b; }

// app.mjs
import { add } from './math.mjs';
```

Node decides which system a file is in by these rules, in order ([packages docs](https://nodejs.org/docs/latest-v24.x/api/packages.html)):

```
   what is the file extension?
        │
        ├── .mjs / .mts  ────────────────────────► ES module. Always. Done.
        ├── .cjs / .cts  ────────────────────────► CommonJS. Always. Done.
        └── .js / .ts
              │
              └── nearest parent package.json says…
                    ├── "type": "module"    ─────► ES module
                    ├── "type": "commonjs"  ─────► CommonJS
                    └── (no "type" field)
                          │
                          └── Node reads the source and guesses:
                              import/export/top-level await/import.meta
                              present → ES module, otherwise → CommonJS
```

That last step — **syntax detection** — has been on by default since 22.7.0. It is why a stray `.js` file with an `import` in it now just works, where a few years ago it threw `Cannot use import statement outside a module`. It is also a fallback, not a feature: relying on it means every file in your project gets sniffed, and ambiguous files (one that is valid in both systems) resolve to CommonJS.

**Say what you mean.** One line in `package.json` removes all of the above:

```json
{ "type": "module" }
```

New projects should set it. That single field is the difference between "modules mostly work" and "modules are predictable".

**Try It — watch the detection happen.**

```js
// probe.js
console.log(typeof require === 'function' ? 'CommonJS' : 'ESM');
```

```bash
node probe.js                                  # → CommonJS (no import syntax present)
echo 'export {};' >> probe.js && node probe.js  # → ESM (an export appeared)
```

The file's *contents* changed what system it was in. That is syntax detection, and seeing it once makes the rule stick.

<a id="p4-working"></a>

### Working Knowledge: resolution, interop, and the ESM rules that differ

#### Specifiers come in three flavours

```js
import x from './local.js';      // relative — a path, resolved against this file
import y from '/abs/path.js';    // absolute
import z from 'lodash';          // bare — a package name; triggers the node_modules walk
import fs from 'node:fs';        // builtin — the node: prefix, unshadowable
```

A bare specifier makes Node walk upward looking for `node_modules`:

```
  /app/src/features/users/service.js   imports 'lodash'
        │
        ├─ /app/src/features/users/node_modules/lodash   ← checked first
        ├─ /app/src/features/node_modules/lodash
        ├─ /app/src/node_modules/lodash
        ├─ /app/node_modules/lodash                      ← usually found here
        └─ /node_modules/lodash
```

This walk is why a package can appear twice in one process at two different versions, and why "it works on my machine" is so often a `node_modules` tree difference rather than a code difference.

#### Three ESM rules that trip up CommonJS habits

ESM follows the browser's resolution semantics, which are stricter than Node's old ones ([ESM docs](https://nodejs.org/docs/latest-v24.x/api/esm.html)):

```js
// ✗ wrong in ESM — all three of these throw ERR_MODULE_NOT_FOUND
import { a } from './util';        // 1. no automatic extension
import { b } from './helpers';     // 2. no directory index resolution
import pkg from './package.json';  // 3. no JSON without an import attribute

// ✓ right
import { a } from './util.js';
import { b } from './helpers/index.js';
import pkg from './package.json' with { type: 'json' };
```

Extensions are mandatory, directories don't resolve to `index.js`, and non-JavaScript needs an explicit import attribute. The upside is that resolution is now a pure string operation with no filesystem guessing — which is exactly what makes bundlers and browsers able to agree with Node.

#### The CommonJS globals don't exist — here's what replaced them

| CommonJS | ESM equivalent |
|---|---|
| `__dirname` | `import.meta.dirname` |
| `__filename` | `import.meta.filename` |
| `require.resolve(x)` | `import.meta.resolve(x)` |
| `require.main === module` | `import.meta.main` |
| `require(x)` | `import x from` / `await import(x)` / `createRequire` |

`import.meta.dirname` and `import.meta.filename` are the quality-of-life additions worth knowing about — for a long time the answer was a three-line `fileURLToPath(new URL('.', import.meta.url))` incantation that you will still find in every StackOverflow answer and most library source. It is no longer needed:

```js
// ✗ outdated — still correct, but nobody needs to write this any more
import { fileURLToPath } from 'node:url';
const __dirname = fileURLToPath(new URL('.', import.meta.url));

// ✓ current
const dir = import.meta.dirname;
```

The `import.meta.url` form is still the right tool for building a URL to a sibling *asset*, because `fs` accepts `URL` objects directly:

```js
const schema = readFileSync(new URL('./schema.sql', import.meta.url), 'utf8');
```

#### Importing CommonJS from ESM, and vice versa

From ESM, a CommonJS module's `module.exports` arrives as the **default** export, and Node also performs static analysis to expose named exports where it can:

```js
import cjs from './legacy.cjs';           // module.exports
import { helper } from './legacy.cjs';    // works if statically detectable
```

The "where it can" is the catch: if a package assigns its exports dynamically (`module.exports[name] = …` in a loop), the named import fails and you must destructure from the default. That is the single most common ESM-consuming-CJS error, and the fix is mechanical:

```js
// ✗ SyntaxError: The requested module does not provide an export named 'helper'
import { helper } from 'some-dynamic-cjs-pkg';

// ✓
import pkg from 'some-dynamic-cjs-pkg';
const { helper } = pkg;
```

Going the other way — `require()`ing an ES module — used to be flatly impossible. It now works, and that change is significant enough that it has its own section in [Mastery](#p4-mastery).

#### Native TypeScript

`node app.ts` runs. Node strips the type annotations and executes the result; it does **not** type-check ([TypeScript docs](https://nodejs.org/docs/latest-v24.x/api/typescript.html)). Stable as of 24.12.0, and on by default.

Because stripping means "replace types with whitespace", anything that requires *generating* JavaScript is unsupported:

```ts
// ✗ these fail under plain type stripping
enum Status { Active, Inactive }          // generates a runtime object
namespace Utils { export const x = 1; }   // generates a runtime object
class User { constructor(private name: string) {} }   // parameter property

// ✓ these are pure type syntax and erase cleanly
type Status = 'active' | 'inactive';
interface User { name: string }
const x = value as string;
function f<T>(x: T): T { return x; }
```

Two rules follow, and both are non-obvious:

```ts
// ✓ import extensions are MANDATORY, and they are the .ts extension
import { helper } from './helper.ts';

// ✓ type-only imports MUST say `type`, or stripping leaves a broken runtime import
import type { User } from './types.ts';
import { create, type Options } from './factory.ts';
```

The `tsconfig.json` that keeps you inside the supported subset — worth copying verbatim, because each line prevents a specific runtime failure:

```json
{
  "compilerOptions": {
    "noEmit": true,
    "target": "esnext",
    "module": "nodenext",
    "erasableSyntaxOnly": true,
    "verbatimModuleSyntax": true,
    "rewriteRelativeImportExtensions": true
  }
}
```

`erasableSyntaxOnly` makes `tsc` reject `enum` and parameter properties at *check* time instead of letting Node fail at *run* time, and `verbatimModuleSyntax` forces the `type` keyword. Node itself never reads this file — it is your type-checker's job to enforce what Node can run. Details in [`typescript/typescript-modules-ecosystem.md`](../typescript/typescript-modules-ecosystem.md).

If you need the unsupported syntax, `--experimental-transform-types` turns stripping into real transformation, or you use a full toolchain (`tsx`, `tsc`, or a bundler). For new code, the better move is to stay in the erasable subset — it is where the language is heading anyway.

> **vs. Bun** — Bun runs any TypeScript, including `enum`, decorators and `.tsx`, with no flags and no extension requirements, and resolves extensionless imports. Code written for Bun therefore does not necessarily run on `node app.ts`, and the failure is a syntax error rather than a helpful message. See [`bun-runtime-foundations.md`](../bun/bun-runtime-foundations.md).

<a id="p4-advanced"></a>

### Advanced: `exports` maps, conditions, and the dual-package hazard

#### `exports` is an access-control list, not just an entry point

`"main"` names one file. `"exports"` names every file a consumer may reach — and **everything else becomes unreachable** ([packages docs](https://nodejs.org/docs/latest-v24.x/api/packages.html)):

```json
{
  "name": "toolkit",
  "exports": {
    ".": "./dist/index.js",
    "./parser": "./dist/parser.js",
    "./package.json": "./package.json"
  }
}
```

```js
import x from 'toolkit';                  // ✓
import p from 'toolkit/parser';           // ✓
import i from 'toolkit/dist/internal.js'; // ✗ ERR_PACKAGE_PATH_NOT_EXPORTED
```

That last error is the "but the file is right there!" error, and it is the package author telling you no. It is also why adding `"exports"` to an existing package is a breaking change: every deep import your users were relying on stops working at once.

#### Conditions, and why order is load-bearing

A condition key picks a different file depending on *how* the module is being loaded:

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "node": {
        "import": "./dist/index.mjs",
        "require": "./dist/index.cjs"
      },
      "default": "./dist/browser.mjs"
    }
  }
}
```

Node evaluates the keys **in the order they appear in the object** and takes the first match ([packages docs](https://nodejs.org/docs/latest-v24.x/api/packages.html)). So:

```json
// ✗ wrong — "default" matches everything, so "node" is dead code
{ "exports": { ".": { "default": "./browser.js", "node": "./node.js" } } }

// ✓ right — most specific first, "default" last
{ "exports": { ".": { "node": "./node.js", "default": "./browser.js" } } }
```

Same for `"types"`: it must come **first**, because TypeScript stops at the first match too. This is one of the few places in JavaScript tooling where JSON key order carries meaning, and it is silently wrong rather than loudly wrong when you get it backwards.

The conditions Node itself understands, in descending specificity: `node-addons`, `node`, `import`, `require`, `module-sync`, `default`. `import` and `require` are mutually exclusive. You can invent your own and select it at runtime:

```bash
node --conditions=development app.js
```

#### `imports` — private aliases inside your own package

The mirror of `exports`, for internal use. Keys must start with `#`:

```json
{
  "imports": {
    "#db": { "node": "./src/db/postgres.js", "default": "./src/db/memory.js" },
    "#utils/*.js": "./src/utils/*.js"
  }
}
```

```js
import { query } from '#db';            // no ../../../ ever again
import { clamp } from '#utils/math.js';
```

This is Node's built-in path aliasing, and it needs no bundler, no `tsconfig` `paths`, and no resolver plugin. Unlike `exports`, an `imports` entry may point at an external package — which makes it the clean way to swap an implementation per environment.

#### The dual-package hazard

If a package ships separate ESM and CJS builds, a single process can end up loading **both**:

```
   your app (ESM)  ──import──►  pkg/index.mjs   ┐
                                                 ├── two copies. Two module states.
   a dependency (CJS) ──require──► pkg/index.cjs ┘
```

Harmless for pure functions. Catastrophic for anything stateful: `instanceof` fails across the boundary, a registry populated by one copy is empty in the other, and a singleton is a singleton twice. Symptoms are surreal — "this error isn't an instance of the error class it literally is".

The official guidance ([packages docs](https://nodejs.org/docs/latest-v24.x/api/packages.html)) is, in order of preference: make both entry points export the *same* thing; or build both from a single source; or — the option that only recently became viable — **ship ESM only and let `require(esm)` handle CommonJS consumers**.

**Real Scenario — the error that wasn't itself.** A service upgrades a validation library. Its own `catch (err) { if (err instanceof ValidationError) … }` stops matching, and every validation failure becomes a 500. The library had added a `require` condition pointing at a CJS build; the app imports it as ESM, an Express middleware `require`s it, and the two copies each define their own `ValidationError` class. Identical source, different identity. The fix was one line in the library's `exports` — but finding it took a day, because `err.constructor.name` said `ValidationError` the whole time. If you ever see that shape, `require.cache` and `process.env.NODE_DEBUG='module'` are where you start.

<a id="p4-mastery"></a>

### Mastery: `require(esm)`, loader hooks, and the compile cache

#### `require()` of an ES module — stable at last

For a decade the answer was "you can't; use dynamic `import()`". As of **24.15.0 this is stable** ([packages docs](https://nodejs.org/docs/latest-v24.x/api/packages.html)):

```js
// in a CommonJS file
const { helper } = require('./modern.mjs');   // ✓ works now
```

One condition, and it is absolute: the module **and its entire dependency graph** must be synchronous. Any top-level `await` anywhere in that graph throws:

```
Error [ERR_REQUIRE_ASYNC_MODULE]: require() cannot be used on an ESM graph
with top-level awaits. Use import() instead.
```

Which is a real constraint but an honest one, and it changes the publishing calculus completely: a library that avoids top-level `await` can now ship ESM only and still be `require()`-able. That is the escape route from the dual-package hazard above, and it is why "always ship dual builds" is advice with a shelf life.

Going the other way inside ESM, when you need CommonJS semantics (`require.cache`, a native addon, a conditional load):

```js
import { createRequire } from 'node:module';
const require = createRequire(import.meta.url);
const addon = require('./build/Release/addon.node');
```

#### Loader hooks — and the one that just got deprecated

Hooks let you intercept resolution and loading: this is how `tsx`, coverage tools and import maps work. **The recommended API changed in 24.15.0**, and this is a live trap ([module docs](https://nodejs.org/docs/latest-v24.x/api/module.html)):

| API | Runs | Status on Node 24 |
|---|---|---|
| `module.register()` | On a **separate loader thread**, async hooks | **Deprecated as of 24.15.0** |
| `module.registerHooks()` | **In-thread**, synchronous hooks | Release candidate, **the recommended one** |

Every tutorial and most tooling READMEs still show `register()`. The in-thread version is simpler, faster (no cross-thread message passing), can share state with your program directly, and — unlike the async hooks — actually affects `require()` calls:

```js
// hooks.mjs
import { registerHooks } from 'node:module';

registerHooks({
  resolve(specifier, context, nextResolve) {
    if (specifier.startsWith('config:')) {
      return { url: new URL(`./config/${specifier.slice(7)}.json`, import.meta.url).href,
               shortCircuit: true };
    }
    return nextResolve(specifier, context);
  },
  load(url, context, nextLoad) {
    return nextLoad(url, context);
  },
});
```

```bash
node --import ./hooks.mjs app.js
```

Hooks chain **last-registered-first**, and each must either call `next*()` or return `shortCircuit: true`. The classic failure is forgetting both, which produces a hang rather than an error.

#### The compile cache — free startup, one line

V8 can cache compiled bytecode on disk, and Node exposes it directly. It came out of experimental in 24.15.0 ([module docs](https://nodejs.org/docs/latest-v24.x/api/module.html)):

```js
// the very first line of your entry point
import { enableCompileCache } from 'node:module';
enableCompileCache();
```

or, with no code change at all:

```bash
NODE_COMPILE_CACHE=./.node-cache node app.js
```

On a CLI with a large dependency tree this is frequently a 30–50% cut in startup time, and the only cost is a cache directory. It is one of the highest value-per-character features Node has shipped — and almost nobody knows it exists.

**Try It — measure it.** On a project with real dependencies:

```bash
rm -rf ./.node-cache
time NODE_COMPILE_CACHE=./.node-cache node -e "require('./index.js')"   # cold: populates the cache
time NODE_COMPILE_CACHE=./.node-cache node -e "require('./index.js')"   # warm
```

Expect the second run to be meaningfully faster. If the two are identical, your app's startup is dominated by I/O or network, not compilation — which is itself a useful thing to have learned.

<a id="part-4-cheat-sheet"></a>

### Part 4 cheat sheet

| Want | How |
|---|---|
| Declare ESM for a whole package | `"type": "module"` in `package.json` |
| Force one file's system | `.mjs`/`.cjs` extension (beats everything) |
| This directory, in ESM | `import.meta.dirname` |
| Am I the entry point? | `import.meta.main` |
| Resolve a specifier | `import.meta.resolve('pkg/file.js')` |
| Import JSON | `import data from './x.json' with { type: 'json' }` |
| `require` inside ESM | `createRequire(import.meta.url)` |
| `require()` an ES module | Just works — unless the graph has top-level `await` |
| Path aliases, no bundler | `"imports": { "#db": "./src/db.js" }` |
| Restrict what consumers can import | `"exports"` map |
| Per-environment builds | Conditions — **most specific first, `default` last, `types` first** |
| Run TypeScript | `node app.ts` — types stripped, not checked |
| Stay inside strippable syntax | `"erasableSyntaxOnly": true` in `tsconfig.json` |
| Customise module loading | `module.registerHooks()` (**not** the deprecated `register()`) |
| Faster startup | `NODE_COMPILE_CACHE=./.cache` or `enableCompileCache()` |

**Gotchas:** ESM requires file extensions and has no directory index · `"default"` before a specific condition makes the specific one dead · `"types"` must be first · a dual ESM/CJS package can load twice and break `instanceof` · top-level `await` makes a module un-`require()`-able · `module.register()` is deprecated in favour of `registerHooks()`.

[↑ Back to top](#table-of-contents)

---

## Part 5 — Packages & the npm Surface

The half of "modules" that happens before Node starts: what `package.json` declares, how versions get chosen, and why the lockfile is the most important file in your repo.

*This Part runs Beginner → Mastery, but the Advanced and Mastery tiers are shorter than elsewhere — publishing and registry mechanics are npm's domain rather than Node's, and this set stops at the boundary where they diverge.*

<a id="p5-beginner"></a>

### Beginner: `package.json`, install, lockfile

```bash
npm init -y          # creates package.json
npm i express        # adds to "dependencies", writes package-lock.json, fills node_modules
npm i -D vitest      # adds to "devDependencies"
npm rm express       # removes it
```

Three artefacts, three jobs:

| File/dir | What it is | In git? |
|---|---|---|
| `package.json` | What you *asked for* — ranges, scripts, metadata | **Yes** |
| `package-lock.json` | What you *got* — exact versions, integrity hashes, full tree | **Yes, always** |
| `node_modules/` | The files themselves | **No** |

The lockfile is the one people get wrong. It records the exact resolved version and hash of every package in the tree, and committing it is what makes your teammate's install, your CI's install and your production install identical. A repo without a committed lockfile is a repo where a transitive dependency's patch release can break production at 3am with no commit to blame.

<a id="p5-working"></a>

### Working Knowledge: semver, `npm ci`, scripts, `npx`

#### Reading a range

```json
{
  "dependencies": {
    "express": "^5.1.0",     // >=5.1.0 <6.0.0    — minor+patch      (npm's default)
    "zod": "~3.23.0",        // >=3.23.0 <3.24.0  — patch only
    "left-pad": "1.3.0"      // exactly 1.3.0
  }
}
```

`^` is what `npm i` writes. It trusts publishers to respect semver, which they mostly do — and the lockfile is what protects you when they don't, because `^` only actually resolves to something new when the lockfile is regenerated.

#### `npm ci` is the one to use in CI and Docker

It is not a synonym for `npm install` ([npm ci docs](https://docs.npmjs.com/cli/v11/commands/npm-ci)):

| | `npm install` | `npm ci` |
|---|---|---|
| Needs a lockfile | No | **Yes — errors without one** |
| Lockfile disagrees with `package.json` | Fixes the lockfile | **Errors** |
| Existing `node_modules` | Reconciled | **Deleted first** |
| Writes `package.json`/lockfile | Can | **Never** |

```dockerfile
# ✗ wrong — can silently resolve different versions than you tested
RUN npm install --omit=dev

# ✓ right — reproducible, and fails loudly if the lockfile is stale
RUN npm ci --omit=dev
```

That "errors if the lockfile disagrees" behaviour is a feature: it catches the pull request where someone edited `package.json` by hand and never re-installed.

#### Scripts

```json
{
  "scripts": {
    "dev": "node --watch --env-file=.env src/server.js",
    "test": "node --test",
    "build": "tsc",
    "start": "node src/server.js"
  }
}
```

Inside a script, `node_modules/.bin` is on the `PATH` — which is why `"build": "tsc"` works without a path or `npx`. Recall from [Part 1](./nodejs-runtime-foundations.md#p1-advanced) that `node --run build` is the fast path, and that it skips `pre`/`post` scripts, so prefer explicit `&&` chaining over implicit hooks.

#### `npx` runs a binary without installing it

```bash
npx tsc --init          # uses the local one if present, else fetches it temporarily
npx --yes cowsay hi     # skip the install prompt
```

Useful for one-shot tools. Not a substitute for a real dev dependency: a tool you run in CI should be pinned in `package.json`, not fetched at whatever version happens to be latest that morning.

<a id="p5-advanced"></a>

### Advanced: workspaces, `overrides`, peer dependencies, publishing

#### Workspaces — a monorepo without a monorepo tool

```json
{
  "name": "root",
  "private": true,
  "workspaces": ["packages/*", "apps/*"]
}
```

```bash
npm i                                   # installs everything, symlinks cross-references
npm i -w apps/api express               # add a dep to one workspace
npm run test -ws --if-present           # run a script in every workspace that has it
npm run dev -w apps/api                 # …or in one
```

Dependencies hoist to the root `node_modules`, and local packages are symlinked — so `import { db } from '@acme/db'` inside `apps/api` resolves to your source, live, with no build or publish step. `"private": true` on the root is not optional; it is what stops a stray `npm publish` uploading your monorepo.

#### `overrides` — force a version somewhere deep in the tree

When a transitive dependency has a CVE and its parent hasn't updated:

```json
{
  "overrides": {
    "semver": "^7.5.2",
    "some-pkg": { "lodash": "^4.17.21" }
  }
}
```

The first form rewrites every `semver` in the tree; the second only rewrites `lodash` where `some-pkg` depends on it. Only the root `package.json`'s `overrides` are honoured ([package.json docs](https://docs.npmjs.com/cli/v11/configuring-npm/package-json)). Treat every override as a temporary patch with a comment explaining why — they are invisible to the next person otherwise, and they lie to the packages they apply to.

#### Peer dependencies

`peerDependencies` means "I plug into this; the host provides it". Plugins, and anything that must share a single instance with its host:

```json
{
  "peerDependencies": { "react": "^18 || ^19" },
  "peerDependenciesMeta": { "react": { "optional": true } }
}
```

npm 7+ installs peers automatically, which means a conflict now surfaces as an `ERESOLVE` install failure rather than a subtle runtime bug. Two notes: `--legacy-peer-deps` makes the error go away without making the problem go away, so treat it as a last resort with an expiry date; and the reason peer deps exist at all is the dual-instance problem from [Part 4](#p4-advanced) — two copies of React is the same failure mode as two copies of a validation library.

#### Publishing: control what ships

```json
{
  "files": ["dist", "README.md"],
  "exports": { ".": "./dist/index.js" },
  "engines": { "node": ">=24" }
}
```

`files` whitelists what goes in the tarball; without it, **everything** that isn't ignored ships — test fixtures, `.env.example`, scratch scripts and all ([package.json docs](https://docs.npmjs.com/cli/v11/configuring-npm/package-json)). `package.json`, the README, the LICENSE and your `main`/`bin` targets are always included; `.git`, `node_modules` and lockfiles can never be.

```bash
npm pack --dry-run     # prints exactly what would be published — run this before every publish
```

<a id="p5-mastery"></a>

### Mastery: supply-chain hygiene

Your `node_modules` is the largest piece of unreviewed code in your product, and installing a package runs its `postinstall` script as you, with your credentials in the environment. The realistic defences are few but cheap:

```bash
npm audit                    # known CVEs in your tree
npm audit fix                # only the ones fixable without a major bump
npm audit signatures         # verify packages were signed by the registry
npm ls some-pkg              # who actually pulled this in?
npm outdated                 # how far behind are you
```

`npm ls <pkg>` is the one to reach for during an incident — it prints the *path* through your dependency graph to the package in question, which turns "why is this here?" into a one-second question.

Three practices that pay for themselves:

```bash
# 1. Don't run lifecycle scripts you haven't thought about, in CI
npm ci --ignore-scripts

# 2. Pin the toolchain, not just the libraries
#    engines + .nvmrc + a committed lockfile = a reproducible build

# 3. Age-gate your dependencies — most malicious publishes are caught within days
npm config set minimum-release-age 3d
```

The third one deserves the emphasis: nearly every npm supply-chain compromise in recent memory was detected and unpublished within 24–72 hours of being published. Refusing to install anything newer than a few days old costs you nothing and removes most of the window.

`--ignore-scripts` needs a caveat: packages with native components (`bcrypt`, `sharp`, `better-sqlite3`) genuinely need their install scripts to build or download binaries. The workable pattern is to disable scripts globally and allow-list the few that need them, rather than the reverse.

> **vs. Bun** — Bun blocks lifecycle scripts by default and requires an explicit `trustedDependencies` list, and has `minimumReleaseAge` built into `bunfig.toml`. The defaults differ; the threat model does not. See [`bun-package-manager.md`](../bun/bun-package-manager.md).

<a id="part-5-cheat-sheet"></a>

### Part 5 cheat sheet

| Want | Command |
|---|---|
| New project | `npm init -y` |
| Add a dependency / dev dependency | `npm i pkg` / `npm i -D pkg` |
| Reproducible install (CI, Docker) | `npm ci` |
| Production install | `npm ci --omit=dev` |
| Why is this package installed? | `npm ls pkg` |
| What's out of date | `npm outdated` |
| Known vulnerabilities | `npm audit` / `npm audit signatures` |
| Force a transitive version | `"overrides"` in the root `package.json` |
| Monorepo | `"workspaces": ["packages/*"]` |
| Add a dep to one workspace | `npm i -w apps/api pkg` |
| Run a script everywhere | `npm run test -ws --if-present` |
| Run a tool once | `npx pkg` |
| See what you'd publish | `npm pack --dry-run` |
| Limit the published tarball | `"files": ["dist"]` |
| Skip install scripts | `npm ci --ignore-scripts` |
| Avoid brand-new releases | `npm config set minimum-release-age 3d` |

**Gotchas:** always commit the lockfile · `npm install` in a Dockerfile defeats reproducibility, use `npm ci` · `--legacy-peer-deps` hides a real conflict · no `"files"` field means you ship your whole working directory · `overrides` are invisible to everyone who didn't write them.

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 6 — Files, Paths & Buffers](./nodejs-files-streams-buffers.md) · **Previous:** [Parts 1–3](./nodejs-runtime-foundations.md) · **Up:** [Overview](./nodejs-mastery-guide.md)

*Written against Node.js 24 LTS · Verified September 14, 2026*
