# TypeScript — Modules & Ecosystem (Parts 11–12)

> **About this document:** this is one of the sub-docs in this repo's TypeScript set. The canonical spec — framing, scope, tier policy, sourcing rule, versions, and how to update it — lives once in [`typescript-mastery-guide.md`](./typescript-mastery-guide.md#about-this-document). Written against **TypeScript 7.0**, **Node.js 24 LTS / 26**, and **Zod 4.5**, September 6, 2026. Assumes [ES modules](../javascript/javascript-errors-async-modules.md) from the JavaScript set.
>
> ⚠️ **This is the doc in the set most likely to go stale.** TypeScript 7.1, expected around October 2026, restores the programmatic API that typescript-eslint, `ts-jest`, and the framework template checkers need. Re-verify [§ TypeScript 7 and the tooling split](#p12-advanced) before acting on it.

Part 11 is modules and declaration files — where the type system meets the module system, and where "it works in the editor but crashes at runtime" comes from. Part 12 is the surrounding toolchain and the boundary where types stop being enough.

---

## Table of Contents

- [Part 11 — Modules & Declaration Files](#part-11)
  - [Beginner — import, export, and `import type`](#p11-beginner)
  - [Working Knowledge — resolution, extensions, `verbatimModuleSyntax`](#p11-working)
  - [Advanced — ESM/CJS interop and dual publishing](#p11-advanced)
  - [Mastery — writing `.d.ts`, augmentation, and `declare global`](#p11-mastery)
  - [Part 11 cheat sheet](#p11-cheatsheet)
- [Part 12 — Ecosystem & Boundaries](#part-12)
  - [Beginner — running TypeScript without a build step](#p12-beginner)
  - [Working Knowledge — check and build are two separate jobs](#p12-working)
  - [Advanced — TypeScript 7 and the tooling split](#p12-advanced)
  - [Mastery — where the type system stops: runtime validation](#p12-mastery)
  - [Part 12 cheat sheet](#p12-cheatsheet)

---

<a id="part-11"></a>

## Part 11 — Modules & Declaration Files

<a id="p11-beginner"></a>

### Beginner — import, export, and `import type`

TypeScript uses ES module syntax, plus one addition: **type-only imports and exports.**

```ts
import { createUser } from "./users.js";        // a value
import type { User } from "./types.js";         // a type — guaranteed erased
import { type Role, hasRole } from "./auth.js"; // inline type specifier (4.5+)

export type { User };
export { createUser };
```

Why the distinction matters: a plain `import { User }` where `User` is only a type may or may not be elided from the output depending on your settings, and if it isn't elided you get a runtime import of a module that may not exist, or a circular-import crash. `import type` is a promise that this import contributes nothing at runtime — and the compiler holds you to it.

```ts
// ❌ This compiles but breaks at runtime under some configurations
import { User } from "./types.js";     // types.ts contains ONLY types
const u: User = { id: 1 };
// emitted: import { User } from "./types.js";  →  SyntaxError: does not provide
// an export named 'User'

// ✅
import type { User } from "./types.js";   // emits nothing at all
```

A file with no top-level `import`/`export` is a **script**, not a module, and its declarations go into the global scope — a common source of "Duplicate identifier" errors in a fresh project. Add `export {}` to make an otherwise-empty file a module.

<a id="p11-working"></a>

### Working Knowledge — resolution, extensions, `verbatimModuleSyntax`

**Module resolution** is how a specifier becomes a file. TypeScript 7 removed the old `node`/`node10`/`classic` strategies; the live options are:

| `moduleResolution` | Use with | Behaviour |
|---|---|---|
| `nodenext` | `module: nodenext` | Node's real ESM/CJS rules, per-file, honouring `package.json` `"type"` and `"exports"` |
| `node16` | `module: node16/node18/node20` | pinned older versions of the same rules |
| `bundler` | `module: esnext`/`preserve` | extensionless imports allowed; `exports` honoured — what Vite/webpack actually do |

```text
  import { x } from "./util"    under moduleResolution:
  ┌──────────────────────────────────────────────────────────────┐
  │ bundler   → ./util.ts, ./util.tsx, ./util/index.ts   ✅       │
  │ nodenext  → ERROR: Relative import paths need explicit file   │
  │             extensions in ECMAScript imports. ts(2835)        │
  │             …you must write "./util.js"                       │
  └──────────────────────────────────────────────────────────────┘
```

**The `.js` extension that points at a `.ts` file** is the single most confusing thing in modern TypeScript, and it is correct:

```ts
// util.ts exists. You write:
import { x } from "./util.js";   // ✅ under nodenext
```

You import the **output** filename, because the emitted JavaScript will contain that specifier verbatim and Node needs it to resolve at runtime. TypeScript maps `./util.js` back to `./util.ts` for checking. Alternatives, if that offends you:

```jsonc
{ "compilerOptions": {
    "rewriteRelativeImportExtensions": true,   // 5.7+: write "./util.ts", emit "./util.js"
    "allowImportingTsExtensions": true          // implied by the above
} }
```

**`verbatimModuleSyntax`** (5.0+, replacing the deprecated `importsNotUsedAsValues` and `preserveValueImports`) makes the emit rule trivial: **imports and exports emit exactly as written, and anything marked `type` is dropped.** No cleverness, no elision surprises:

```ts
import { A } from "./a.js";        // always emitted, even if A is only used as a type
import type { B } from "./b.js";   // always dropped
```

Turn it on. It makes your output predictable, it makes `isolatedModules`-style per-file transpilation safe, and it forces the `import type` discipline that prevents runtime import errors. The one thing it will not let you do is write `import { SomeType }` and rely on elision — which is the behaviour you wanted gone.

<a id="p11-advanced"></a>

### Advanced — ESM/CJS interop and dual publishing

The interop rules are a runtime concern that TypeScript models rather than invents. Under `nodenext`, **a file's module format comes from its extension or the nearest `package.json` `"type"` field**:

```text
  package.json { "type": "module" }        package.json { "type": "commonjs" }
  ├── a.ts   → ESM                          ├── a.ts   → CJS
  ├── b.mts  → ESM (always)                 ├── b.mts  → ESM (always)
  └── c.cts  → CJS (always)                 └── c.cts  → CJS (always)
```

The rules that follow, and the errors they produce:

```ts
// In an ESM file importing a CJS package:
import express from "express";        // ✅ default import gets module.exports
import { Router } from "express";     // ⚠️ named imports from CJS work only when
                                      //    Node's cjs-module-lexer can detect them

// In a CJS file importing an ESM-only package:
const chalk = require("chalk");       // 💥 ERR_REQUIRE_ESM (Node < 22.12)
const chalk2 = await import("chalk"); // ✅ dynamic import works everywhere
// Node 22.12+/24 can `require()` a synchronous ESM graph, which softens this
// considerably — but the TypeScript-level error ts(1479) still fires when the
// config says otherwise.
```

`esModuleInterop` used to be the flag governing default-import behaviour; **in TypeScript 7 it is always on and can no longer be disabled**. Any config that sets `"esModuleInterop": false` is now a hard error.

**Publishing a library** is where this all becomes load-bearing. The modern shape:

```jsonc
{
  "name": "my-lib",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",     // must come FIRST in the object
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    },
    "./package.json": "./package.json"
  },
  "files": ["dist"]
}
```

Two rules people get wrong: **`"types"` must be the first key** in each condition object (resolution is order-sensitive), and if you ship both formats each needs its own declaration file (`index.d.ts` for ESM, `index.d.cts` for CJS) or consumers get subtly wrong types. Run [`@arethetypeswrong/cli`](https://github.com/arethetypeswrong/arethetypeswrong.github.io) in CI — it catches every variant of this, and there are more variants than you'd believe.

<a id="p11-mastery"></a>

### Mastery — writing `.d.ts`, augmentation, and `declare global`

A **declaration file** describes types for JavaScript that has none. It contains only declarations — no implementations, and nothing that emits.

```ts
// legacy-lib.d.ts
declare module "legacy-lib" {
  export interface Options { retries?: number }
  export function connect(url: string, opts?: Options): Promise<void>;
  export default connect;
}
```

Where TypeScript looks for types, in order: the package's own `types`/`exports` entry → `@types/<name>` from [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) → your local `.d.ts` files (via `include`) → nothing, which under `noImplicitAny` is `ts(7016)`.

```bash
npm i -D @types/express      # the DefinitelyTyped route, still the norm for older packages
```

**Module augmentation** adds to an existing module's types. This is where [interface declaration merging](./typescript-objects-classes.md#part-6) stops being trivia and becomes essential — a `type` alias cannot do this:

```ts
// express.d.ts — add a property your middleware attaches to every request
import "express";                 // ← the import makes this a module, which is required

declare module "express-serve-static-core" {
  interface Request {
    user?: { id: string; role: string };
  }
}
```

```ts
// vite-env.d.ts — the same pattern for typed environment variables
interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_FEATURE_FLAG: "on" | "off";
}
interface ImportMeta { readonly env: ImportMetaEnv }
```

**`declare global`** reaches the global scope from inside a module:

```ts
export {};   // make this file a module first

declare global {
  interface Window { __APP_VERSION__: string }
  namespace NodeJS {
    interface ProcessEnv {
      DATABASE_URL: string;
      NODE_ENV: "development" | "production" | "test";
    }
  }
}
```

That `ProcessEnv` augmentation is popular and **actively dangerous**, which is worth stating plainly: it makes `process.env.DATABASE_URL` type `string` when the honest type is `string | undefined`. The type system now asserts something no one verified. Prefer validating the environment once at startup and exporting the result:

```ts
// ✅ env.ts — parse, don't declare
import { z } from "zod";

export const env = z.object({
  DATABASE_URL: z.string().url(),
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  PORT: z.coerce.number().int().positive().default(3000),
}).parse(process.env);
// env is fully typed AND actually checked. Missing DATABASE_URL fails at boot,
// loudly, instead of at 3am as `undefined` in a connection string.
```

**Writing declarations by hand** is a last resort — prefer generating them (`tsc --declaration`, or `isolatedDeclarations` for speed; see [Part 3](./typescript-foundations.md#part-3)). When you must:

```ts
declare const VERSION: string;                    // ambient value
declare function legacyGlobal(x: number): void;   // ambient function
declare namespace MyLib { interface Config {} }   // ambient namespace (legacy pattern)

// A file with NO top-level import/export is a global script — its declarations
// are ambient. Add `export {}` to scope them to the module instead. This single
// rule explains most "why is my .d.ts being ignored / applying everywhere" bugs.
```

<a id="p11-cheatsheet"></a>

### Part 11 cheat sheet

| Need | Write |
|---|---|
| Import a type | `import type { T } from "./m.js"` |
| Mixed import | `import { fn, type T } from "./m.js"` |
| Make a file a module | `export {}` |
| Node-style resolution | `"module": "nodenext"` |
| Bundler resolution | `"module": "esnext"`, `"moduleResolution": "bundler"` |
| Predictable emit | `"verbatimModuleSyntax": true` |
| Write `.ts` in import paths | `"rewriteRelativeImportExtensions": true` (5.7+) |
| Add a property to a library type | `declare module "pkg" { interface X {…} }` |
| Add a global | `declare global { … }` inside a module |
| Third-party types | `npm i -D @types/<pkg>` |

| Error | Meaning |
|---|---|
| `ts(2835)` | relative import needs an explicit extension (`nodenext`) |
| `ts(1479)` | can't `require()` an ESM-only module under this config |
| `ts(7016)` | no declaration file — install `@types/…` or write one |
| `ts(2306)` / "is not a module" | the file has no top-level import/export |
| `ts(1259)` "can only be default-imported using esModuleInterop" | pre-TS7 config; interop is now always on |
| `SyntaxError: does not provide an export named 'X'` at runtime | a type imported as a value — use `import type` |
| Types resolve, runtime `MODULE_NOT_FOUND` | `paths` not mirrored in the runtime/bundler ([Part 3](./typescript-foundations.md#part-3)) |

[↑ Back to top](#table-of-contents)

---

<a id="part-12"></a>

## Part 12 — Ecosystem & Boundaries

<a id="p12-beginner"></a>

### Beginner — running TypeScript without a build step

In 2026 you can run a `.ts` file directly, three ways:

```bash
node app.ts        # Node's built-in type stripping. Stable since v24.12.0,
                   # on by default since v22.18/v23.6. No install, no config.
npx tsx app.ts     # tsx — esbuild-based. Handles JSX, enums, decorators, CJS/ESM.
deno run app.ts    # Deno — types natively, and type-checks by default.
```

Node's support is the notable one because it needs nothing at all. Per the [Node.js TypeScript docs](https://nodejs.org/api/typescript.html), type stripping is now stable and enabled by default (`--no-strip-types` disables it; the old `--experimental-transform-types` flag was **removed in v26.0.0**). But it only *strips*, so anything requiring code generation is refused:

```ts
enum Color { Red }                        // 💥 ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX
namespace N { export const x = 1 }        // 💥 (type-only namespaces are fine)
class P { constructor(private x: number) {} }  // 💥 parameter properties
@decorator class C {}                     // 💥 decorators
```

Plus: `.tsx` is not supported, and **import specifiers need explicit extensions** (`import "./util.ts"` — Node resolves the `.ts` itself).

The tsconfig Node's own docs recommend for this mode:

```jsonc
{ "compilerOptions": {
    "noEmit": true, "target": "esnext", "module": "nodenext",
    "rewriteRelativeImportExtensions": true,
    "erasableSyntaxOnly": true, "verbatimModuleSyntax": true } }
```

`erasableSyntaxOnly` is what turns a runtime crash into a compile error, which is the whole point ([Part 3](./typescript-foundations.md#part-3)).

<a id="p12-working"></a>

### Working Knowledge — check and build are two separate jobs

This is the mental model shift that modern TypeScript demands, and the source of most "but it compiled!" confusion:

```text
   ┌──────────────────────────┐        ┌──────────────────────────┐
   │  TYPE CHECKING           │        │  TRANSFORMING            │
   │                          │        │                          │
   │  tsc --noEmit            │        │  esbuild / swc / Vite    │
   │  vue-tsc, svelte-check   │        │  tsx, Node, Bun          │
   │  your editor             │        │  tsc (also can emit)     │
   │                          │        │                          │
   │  SLOW, correct, needs    │        │  FAST, per-file, knows   │
   │  the whole program       │        │  NOTHING about types     │
   └────────────┬─────────────┘        └────────────┬─────────────┘
                │                                   │
         run in CI + editor                  runs on every save
                └───────────────┬───────────────────┘
                                ▼
                   Neither one alone is enough.
```

Because bundlers transform **one file at a time**, they cannot know whether `import { Foo }` refers to a type or a value — hence `isolatedModules` and `verbatimModuleSyntax`, which forbid the constructs that would make per-file transformation ambiguous. Turn both on whenever a bundler does your emit.

The practical setup:

```jsonc
// package.json
{
  "scripts": {
    "dev": "vite",                        // transform only, instant
    "build": "tsc --noEmit && vite build", // check THEN build
    "typecheck": "tsc --noEmit",
    "typecheck:watch": "tsc --noEmit --watch"
  }
}
```

**Never let CI's only type check be `vite build`** — it doesn't type-check, and it will happily ship a build full of type errors.

<a id="p12-advanced"></a>

### Advanced — TypeScript 7 and the tooling split

> Re-verify this section first; it is the most time-sensitive material in the set.

[TypeScript 7.0](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/) (July 8, 2026) is the Go rewrite: **7.7×–11.9× faster** full builds, 6–26% less memory, and editor responsiveness on a large codebase down from ~17.5s to ~1.3s. On the VS Code codebase a full check went from 125.7s to 10.6s.

It shipped **without a stable programmatic API**, which is the catch. Tools that reached into the compiler's internals cannot run on it:

| Tool | Status (September 2026) |
|---|---|
| `tsc` / editor language service | ✅ TypeScript 7 |
| esbuild / swc / Vite / tsx / Node stripping | ✅ unaffected — they never used the API |
| **typescript-eslint** (type-aware rules) | ❌ needs TS 6 |
| **ts-jest**, **ts-morph**, webpack `ts-loader` | ❌ needs TS 6 |
| **vue-tsc**, **svelte-check**, Astro, Angular, MDX checkers | ❌ needs TS 6 |

The typical failure is an `ERESOLVE` peer conflict on install, or `Cannot read properties of undefined (reading 'Cjs')` at runtime — `typescript-estree` reaching for an internal that no longer exists. The official workaround is to alias the old compiler:

```jsonc
// package.json — use the JS-based compiler where a tool still needs the old API
{ "devDependencies": { "typescript": "npm:@typescript/typescript6@^6.0.2" } }
```

TypeScript 7.1 is expected around October 2026 with the new programmatic API, at which point this whole section should collapse to a footnote.

**Decision guide as of today:**

| Project | Use |
|---|---|
| New Node/backend service, no typed ESLint rules | TypeScript 7 |
| Vue / Svelte / Astro / Angular app | TypeScript 6 until the template checkers catch up |
| Anything with type-aware ESLint (`@typescript-eslint/no-floating-promises` and friends) | TypeScript 6, or run 7 for `tsc` and 6 for lint |
| Library published to npm | TypeScript 7 for the build; verify `.d.ts` output with `attw` |

**Typed linting is still worth the trouble**, whichever compiler you pin it to — it catches a category `tsc` deliberately doesn't:

```ts
// tsc is silent on all of these; type-aware ESLint is not
async function save() { db.write() }         // no-floating-promises
if (someObject) { }                          // no-unnecessary-condition (always truthy)
const s = `${someObject}`;                   // restrict-template-expressions
arr.map(async x => await f(x));              // no-misused-promises
```

<a id="p12-mastery"></a>

### Mastery — where the type system stops: runtime validation

Everything in this doc set is erased before your program runs ([Part 1](./typescript-foundations.md#part-1)). At every boundary where data enters from outside, your types are **claims, not guarantees**:

```text
                     ┌─────────────────────────────────┐
   HTTP responses ──▶│                                 │
   JSON.parse     ──▶│   VALIDATE HERE                 │──▶ typed, trusted
   process.env    ──▶│   (Zod, Valibot, ArkType,       │    interior
   DB driver rows ──▶│    typia, TypeBox…)             │
   CLI args       ──▶│                                 │
   localStorage   ──▶│   unknown ──▶ parse ──▶ T       │
   3rd-party SDKs ──▶│                                 │
                     └─────────────────────────────────┘

   Inside the box: `as User` is a lie you are choosing to believe.
   Outside it: the type system has already told you it's `any`/`unknown`.
```

**Zod** ([zod.dev](https://zod.dev/), v4.5 at the time of writing) is the default answer because a schema and a type are the same declaration — no drift possible:

```ts
import { z } from "zod";

const UserSchema = z.object({
  id: z.uuid(),
  name: z.string().min(1),
  email: z.email(),
  role: z.enum(["admin", "user"]),
  createdAt: z.iso.datetime().transform(s => new Date(s)),
});

type User = z.infer<typeof UserSchema>;   // the type is DERIVED from the schema

// ❌ Wrong — a claim
const user1 = (await res.json()) as User;

// ✅ Right — a check. Throws a ZodError listing every failing path.
const user2 = UserSchema.parse(await res.json());

// ✅ Right, without exceptions
const result = UserSchema.safeParse(await res.json());
if (!result.success) {
  logger.warn({ issues: result.error.issues }, "invalid user payload");
  return;
}
result.data;   // User — genuinely, verifiably
```

The direction matters: **derive the type from the schema (`z.infer`), never write both by hand.** Two hand-written declarations of the same shape will diverge; one will be right and the other will be the one your code trusts.

Where to put the boundary:

| Boundary | Validate? |
|---|---|
| Inbound HTTP request bodies, query params | **Always** |
| Outbound API responses you consume | **Always** — this is the one people skip |
| `process.env` | **Always**, once at startup ([Part 11](#p11-mastery)) |
| Config files, CLI args | **Always** |
| Database rows | Depends — a typed query builder (Prisma, Kysely, Drizzle) may already guarantee the shape; validate if the driver returns `any` |
| Internal function calls | **No** — that's what the compiler is for |
| Every layer of your own app | **No** — validating in three places is cost without benefit |

The alternatives, briefly, because Zod isn't the only answer: **Valibot** (much smaller bundle, modular imports — good for the browser), **ArkType** (TypeScript-syntax schemas, very fast), **TypeBox** (JSON Schema output — the right pick if you also need OpenAPI). All four implement [Standard Schema](https://standardschema.dev/), so libraries can accept any of them interchangeably.

**Real Scenario — the field that was `null` for 0.3% of users.** An API contract said `profile: { displayName: string }`. Six months in, deleted accounts began returning `profile: null`. The frontend did `const user = await res.json() as User` and rendered `user.profile.displayName`. TypeScript reported nothing — `res.json()` is `Promise<any>`, and `any` satisfies `User` without inspection — so the crash appeared only for the small slice of users who had a deleted account in their feed, in production, with no compile-time signal anywhere. Adding `UserSchema.parse()` at the fetch boundary turned a silent runtime crash into a single logged validation error naming the exact path (`profile: expected object, received null`), on the first affected request. **The type system's guarantees are only as good as the honesty of the values entering it, and `as` is where honesty goes to die.**

<a id="p12-cheatsheet"></a>

### Part 12 cheat sheet

| Task | Command |
|---|---|
| Run a `.ts` file, no install | `node app.ts` (Node 24+) |
| Run with JSX/enums/decorators | `npx tsx app.ts` |
| Type-check only | `npx tsc --noEmit` |
| Check on save | `npx tsc --noEmit --watch` |
| Check a Vue project | `npx vue-tsc --noEmit` (TS 6 for now) |
| Faster CI check | `npx tsc --noEmit --checkers 8` (TS 7) |
| Verify published types | `npx @arethetypeswrong/cli --pack .` |
| Find the slow type | `npx tsc --generateTrace ./trace` |

| Runtime | Type-checks? | Enums/decorators? | JSX? |
|---|---|---|---|
| `tsc` | ✅ | ✅ | ✅ |
| `node app.ts` | ❌ | ❌ | ❌ |
| `tsx` | ❌ | ✅ | ✅ |
| esbuild / Vite | ❌ | ✅ | ✅ |
| Deno | ✅ | partial | ✅ |
| Bun | ❌ | ✅ | ✅ |

| Boundary rule | |
|---|---|
| `JSON.parse` / `res.json()` returns | `any` — validate |
| `process.env.X` is really | `string \| undefined` — validate at boot |
| `as T` guarantees | nothing |
| `z.infer<typeof S>` guarantees | the shape, **if** you called `parse` |
| Validate at | the edge, once |
| Don't validate | between your own internal functions |

[↑ Back to top](#table-of-contents)

---

**You've reached the end of the set.** Loop back to the [Quick Self-Check](./typescript-mastery-guide.md#9-quick-self-check) in the overview.

*Part of the [TypeScript doc set](./typescript-mastery-guide.md) in [my-learning-md-docs](../README.md). Last updated: September 6, 2026.*
