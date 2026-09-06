# TypeScript — Mastery Guide (Overview)

The entry point for this repo's TypeScript material. This document holds the shared spec, the setup you need before running anything, the map of the focused sub-docs, the 2026 compiler landscape (TypeScript 7 changed more defaults than any release in a decade), and the reference apparatus — learning order, self-check, cross-doc cheat sheet.

The actual teaching lives in the sub-docs. Start here, then follow the [Suggested Learning Order](#8-suggested-learning-order).

---

## About This Document

This section is the canonical spec for **every doc in this set** — the sub-docs each carry a short pointer back here rather than repeating it.

- **Framing: curiosity-driven.** No upcoming task is driving this. The goal is durable, hands-on understanding of TypeScript's *type system* — what it is actually computing, and why it answers the way it does. Accordingly these docs are a **guided tutorial**: a sequence of snippets you are meant to paste into a checker, each **Try It** stating the error code and message you should see. Theory appears only where it is needed to explain a mechanism — structural assignability, control-flow analysis, inference sites, distributivity — never as a standalone history section.
- **Confirmed scope.** The language and its checker: erasure and the compile model, everyday types, narrowing and control-flow analysis, functions and overloads, interfaces/type aliases/objects, classes, generics, type-level programming (conditional, mapped, template-literal types), modules and declaration files. Plus three pieces of periphery that were explicitly confirmed in scope: **`tsconfig` and the compiler in depth**, **ecosystem tooling** (`tsx`, Node's native type stripping, Vite/esbuild strip-vs-check, typed ESLint), and **runtime validation** (Zod) at the boundary where the type system stops.
- **Deliberately excluded** (with pointers): framework-specific typing (Vue/React/Angular component and template types), testing frameworks, monorepo/build orchestration beyond project references, and JS-to-TS migration strategy for a large legacy codebase. See [Deliberately Not Covered](#7-deliberately-not-covered-and-where-to-look-instead).
- **Depth tiers.** Each Part runs **Beginner → Working Knowledge → Advanced → Mastery**, so you can stop at whichever level you need today. Tiers are collapsed where one would be filler, and each collapse is stated inline. Collapsing is expected behaviour, not a defect to fix on a later pass. Two Parts collapse hard by design: **Part 3 (`tsconfig`)** runs Working Knowledge → Mastery, because "beginner tsconfig" in 2026 is one line (`{}` — the defaults are now strict), and **Part 10 (utility types)** runs Working Knowledge → Mastery, because it is Part 9's machinery applied.
- **Assumed background: solid JavaScript, no TypeScript.** This set assumes the [`javascript/`](../javascript/javascript-mastery-guide.md) docs in this repo: closures, prototypes, `this`, iteration protocols, Promises and the event loop, ES modules. Those are **not** re-explained — they are cross-linked at the point they matter. There is no JavaScript primer section. If a snippet's JavaScript half confuses you, follow the link and come back.
- **Per-section ingredients**, used where they earn their place rather than forced into every subsection:
  - a short plain-language explanation of what a thing is and why it matters;
  - a **wrong vs. right** snippet pair wherever a gotcha has a common wrong form;
  - a **Try It** — a snippet with the exact error TypeScript should produce, including the `TSxxxx` code, since this is a curiosity-driven doc set (production "Real Scenario" incidents are the goal-driven equivalent and appear only where a gotcha genuinely only bites in production, e.g. the `any` that escaped through a JSON boundary);
  - an **ASCII diagram** where the concept is spatial or structural — the compile pipeline, the assignability lattice, module resolution, the inference algorithm's flow.
- **TypeScript-7-forward.** Every example assumes **TypeScript 7.0** defaults: `strict` on, `module: esnext`, `types: []`, `rootDir: ./`. Options that TypeScript 6 deprecated and TypeScript 7 *removed* (`baseUrl`, `moduleResolution: node`, `target: es5`, `esModuleInterop: false`) appear only in "you will see this in old blog posts and old repos" callouts, clearly labelled as dead. This matters more than usual right now: the majority of TypeScript writing on the web predates March 2026 and configures a compiler that no longer exists.
- **Sourcing rule.** The [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html) and [TSConfig Reference](https://www.typescriptlang.org/tsconfig/) are the primary sources for language and compiler behaviour, cited inline at the specific claim they support. Release-note and [devblog](https://devblogs.microsoft.com/typescript/) posts are the authority for when something changed and what a version's defaults are. [Node.js docs](https://nodejs.org/api/typescript.html) are the authority for runtime type stripping. Stack Overflow is used for real-world gotchas. Everything else is supplementary. Where a popular-but-outdated convention disagrees with the official docs, the official docs win and the doc says so explicitly.
- **A note on the TSConfig Reference.** It is authoritative per-option but, as of this writing, several of its **"Default:" lines still describe pre-6.0 behaviour** (the `module` and `moduleResolution` defaults, for instance). Where this set states a default, it is the TypeScript 7 default from the [7.0 announcement](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/), and any disagreement with the reference page is flagged inline.
- **Formatting.** Clickable TOC at the top of every doc; "back to top" links after each major section; explicit `<a id="..."></a>` anchors on tier headings, because "Advanced" repeats in every Part and bare-heading anchors would collide; a quick-reference table at the end of each Part; fenced code blocks with language tags.
- **Written against.** **TypeScript 7.0** (GA July 8, 2026), with TypeScript 6.0 (March 23, 2026) referenced wherever a tool still requires it. Runtime examples assume **Node.js 24 LTS or 26**. Runtime-validation examples use **Zod 4.5**. Checked **September 6, 2026**.
- **To update this doc later.** *Preserve*: the curiosity-driven framing, the globally numbered Part structure with explicit tier subheadings, the wrong-vs-right pairs, the Try It exercises with expected error codes, the TOC/back-to-top/anchor apparatus, and the per-Part cheat sheets. *Re-verify against current docs first*: the TypeScript version and its defaults (7.1 was expected around October 2026 and is the release that restores the **programmatic API** — everything in [Part 12](./typescript-modules-ecosystem.md) about typescript-eslint, `ts-jest`, and framework template checkers being stuck on TS 6 should be re-checked first and will likely be stale); Node's type-stripping flags and LTS line; Zod's major version; and every error-code and message string in a Try It, since messages get reworded between releases.

---

## Table of Contents

1. [How This Set Is Organised](#1-how-this-set-is-organised)
2. [The Sub-Docs](#2-the-sub-docs)
3. [Orientation & Setup](#3-orientation--setup)
4. [The One-Paragraph Mental Model](#4-the-one-paragraph-mental-model)
5. [The 2026 Compiler Landscape](#5-the-2026-compiler-landscape)
6. [How to Read a TypeScript Error](#6-how-to-read-a-typescript-error)
7. [Deliberately Not Covered (and where to look instead)](#7-deliberately-not-covered-and-where-to-look-instead)
8. [Suggested Learning Order](#8-suggested-learning-order)
9. [Quick Self-Check](#9-quick-self-check)
10. [Cross-Doc Cheat Sheet](#10-cross-doc-cheat-sheet)

---

## 1. How This Set Is Organised

<a id="overview-organisation"></a>

TypeScript is too large for one file you can comfortably search. It is split here into an overview (this doc) plus six focused sub-docs, each covering a cluster of Parts that genuinely belong together — the same shape as the [`javascript/`](../javascript/javascript-mastery-guide.md) set.

Two rules make the split work:

1. **The spec lives here, once.** Each sub-doc's "About This Document" is a short pointer back to this file. Change the style here and the whole set changes.
2. **Parts are numbered globally, not per-doc.** Part 8 is Generics no matter which file it lives in, so cross-references stay stable if a Part later moves.

```text
typescript/
├── typescript-mastery-guide.md          ← you are here (spec, setup, map, landscape, self-check)
├── typescript-foundations.md            Parts 1–3
├── typescript-narrowing-functions.md    Parts 4–5
├── typescript-objects-classes.md        Parts 6–7
├── typescript-generics.md               Part 8
├── typescript-type-level.md             Parts 9–10
└── typescript-modules-ecosystem.md      Parts 11–12
```

[↑ Back to top](#table-of-contents)

---

## 2. The Sub-Docs

<a id="overview-subdocs"></a>

| Doc | Parts | What it covers | Read it when |
|---|---|---|---|
| [Foundations](./typescript-foundations.md) | 1–3 | Erasure and the compile model; structural typing; everyday types; `any`/`unknown`/`never`; `tsconfig` and the compiler in depth | You want the substrate — and a config you actually understand |
| [Narrowing & Functions](./typescript-narrowing-functions.md) | 4–5 | Control-flow analysis, discriminated unions, type predicates, assertion functions, exhaustiveness; parameter/return inference, overloads, `this` typing | A value is "possibly undefined" and you cannot see why |
| [Objects & Classes](./typescript-objects-classes.md) | 6–7 | `interface` vs `type`, index signatures, `readonly`, excess property checks, declaration merging, variance; class modifiers, `abstract`, `implements`, `#private`, decorators | You are designing the shape of your data, or writing classes |
| [Generics](./typescript-generics.md) | 8 | Constraints, inference sites, defaults, variance annotations, `const` type parameters, generic functions that actually infer well | Your generic function keeps inferring `unknown` |
| [Type-Level Programming](./typescript-type-level.md) | 9–10 | Conditional types, `infer`, distributivity, mapped types, key remapping, template-literal types, recursion, `satisfies`; the utility types rebuilt from scratch | You want the expert tier, or want `Partial`/`Omit` to stop being magic |
| [Modules & Ecosystem](./typescript-modules-ecosystem.md) | 11–12 | ESM/CJS interop, `verbatimModuleSyntax`, `.d.ts`, module augmentation, DefinitelyTyped; `tsx`, Node type stripping, Vite/esbuild, typed ESLint, Zod at the boundary | An import resolves in the editor but explodes at runtime |

[↑ Back to top](#table-of-contents)

---

## 3. Orientation & Setup

<a id="overview-setup"></a>

### Beginner — what TypeScript actually is

<a id="part-0-beginner"></a>

TypeScript is **JavaScript plus a static type layer that is deleted before the code runs**. That sentence is the whole design, and almost every beginner confusion is a failure to fully believe it.

```text
┌──────────────────────────────────────────────────────────────────┐
│  YOUR .ts FILE                                                   │
│  const user: User = JSON.parse(body)   ← types are just syntax   │
└───────────────┬──────────────────────────────────────────────────┘
                │
       ┌────────┴────────┐
       │                 │
       ▼                 ▼
┌──────────────┐   ┌───────────────────────────────────────────────┐
│ TYPE CHECKER │   │ EMITTER / STRIPPER                            │
│              │   │                                               │
│ reads types, │   │ deletes ": User", interfaces, type aliases,   │
│ reports      │   │ generic params, `as`, `satisfies` …           │
│ TS2322 etc.  │   │                                               │
│              │   │ emits:                                        │
│ produces NO  │   │   const user = JSON.parse(body)               │
│ runtime code │   └───────────────┬───────────────────────────────┘
└──────────────┘                   │
       ▲                           ▼
       │                  ┌─────────────────────┐
   errors do NOT  ────────│ NODE / BROWSER      │
   stop the emit          │ runs plain JS.      │
   (by default)           │ Knows nothing about │
                          │ `User`. Checks      │
                          │ nothing.            │
                          └─────────────────────┘
```

Three consequences fall straight out of that diagram, and they are the source of most real bugs:

1. **No type survives to runtime.** You cannot write `if (x instanceof User)` for an `interface User`. There is no `User` at runtime. ([Part 1](./typescript-foundations.md#part-1))
2. **A type annotation is a *claim*, not a check.** `JSON.parse(body) as User` does not verify anything. The checker believes you; the network does not care. This is exactly where [Zod](./typescript-modules-ecosystem.md#part-12) earns its place.
3. **Type errors do not stop the emit.** `tsc` writes JavaScript even when it reported errors, unless you set `noEmitOnError`. Tools like `tsx`, `esbuild`, and Node itself don't even *have* a checker — they strip and run. ([Part 12](./typescript-modules-ecosystem.md#part-12))

### Working Knowledge — getting a checker in front of you

<a id="part-0-working"></a>

Every **Try It** in these docs runs in any of these. Pick one and keep it open.

```bash
# Option A — the TypeScript playground. Zero install, shareable links, shows the
# emitted JS and inferred types on hover. Best for type-level work.
#   https://www.typescriptlang.org/play

# Option B — a real local project. Best for anything touching config or modules.
mkdir ts-scratch && cd ts-scratch
npm init -y
npm install -D typescript          # installs TypeScript 7
npx tsc --init                     # writes a tsconfig.json
npx tsc --noEmit                   # type-check only, no output files
npx tsc --watch --noEmit           # re-check on save — leave this running

# Option C — run a .ts file directly, no build step.
node scratch.ts                    # Node 24+/26: strips types and runs. NO type checking.
npx tsx scratch.ts                 # same idea, plus JSX / enums / decorators support.
```

**A trap worth internalising on day one:** Option C does **not** type-check. `node scratch.ts` will happily run a file full of type errors, because it strips types without ever consulting a checker. Checking and running are two separate jobs in TypeScript 2026, and you need both. See [Part 12](./typescript-modules-ecosystem.md#part-12).

### Advanced — the minimal honest `tsconfig.json`

<a id="part-0-advanced"></a>

As of TypeScript 6.0, [the defaults flipped to strict](https://devblogs.microsoft.com/typescript/announcing-typescript-6-0/) — `strict: true`, `module: esnext`, `target: es2025`. TypeScript 7 kept that. So a modern config is short, and every line in it should be a decision you made rather than a line you copied:

```jsonc
{
  "compilerOptions": {
    "rootDir": "./src",                  // 7.0 defaults rootDir to ./ — set it explicitly
    "outDir": "./dist",
    "module": "nodenext",                // for Node; leave the esnext default for bundlers
    "types": ["node"],                   // 6.0+ defaults to [] — you must opt in to @types now
    "verbatimModuleSyntax": true,        // imports emit exactly as written — see Part 11
    "erasableSyntaxOnly": true,          // ban enums/namespaces so Node can run your .ts — Part 12
    "noUncheckedIndexedAccess": true,    // NOT in `strict`. Turn it on. See Part 3
    "exactOptionalPropertyTypes": true   // NOT in `strict`. See Part 3
  },
  "include": ["src"]
}
```

Everything in that file is explained in [Part 3](./typescript-foundations.md#part-3). If you only ever copy one thing from this doc set, copy the last two lines — they are the two strictness flags `strict: true` does *not* include, and they close the two most common holes in an otherwise strict codebase.

[↑ Back to top](#table-of-contents)

---

## 4. The One-Paragraph Mental Model

<a id="overview-model"></a>

TypeScript's checker answers exactly one question, over and over: **"is this type assignable to that one?"** It answers *structurally* — by comparing the shapes of the types, never their names or their declaration sites — so anything with the right members qualifies, no `implements` required. Types form a lattice with `never` at the bottom (assignable to everything, nothing assignable to it), `unknown` at the top (everything assignable to it, it assignable to nothing), and `any` off to the side as a hole that is assignable in *both* directions and therefore switches the question off. Unions widen a value's possibilities and narrowing shrinks them back down as control flow proves things; generics let a type be a *function of* another type, and conditional and mapped types make that function-of-types effectively a programming language of its own. Everything else in this doc set is detail hanging off those five sentences.

[↑ Back to top](#table-of-contents)

---

## 5. The 2026 Compiler Landscape

<a id="overview-landscape"></a>

**Read this section before you copy any TypeScript config or advice you find online.** More changed in 2026 than in the whole 5.x line, and the web has not caught up.

### What happened

| Date | Release | What it means for you |
|---|---|---|
| March 23, 2026 | [**TypeScript 6.0**](https://devblogs.microsoft.com/typescript/announcing-typescript-6-0/) | Last release built on the JavaScript codebase. Flipped the defaults (`strict`, `module`, `target`, `types`, `rootDir`) and deprecated everything 7.0 was going to remove. Explicitly a bridge release. |
| July 8, 2026 | [**TypeScript 7.0**](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/) | The compiler rewritten in **Go** (called `tsgo` during the preview). 7.7×–11.9× faster full builds, 6–26% less memory, editor "open a file with errors" down from ~17.5s to ~1.3s on the VS Code codebase. Deprecations became hard errors. |
| ~Autumn 2026 | TypeScript 7.1 (expected) | Restores a **stable programmatic API**. Until it lands, a large slice of the ecosystem cannot run on 7.0 at all. |

### The defaults that flipped (TypeScript 7.0)

| Option | Old default | 7.0 default |
|---|---|---|
| `strict` | `false` | **`true`** |
| `module` | `commonjs` | **`esnext`** |
| `types` | all visible `@types` packages | **`[]`** (opt in explicitly) |
| `rootDir` | inferred from inputs | **`./`** |
| `noUncheckedSideEffectImports` | `false` | **`true`** |
| `stableTypeOrdering` | n/a | **`true`**, and cannot be disabled |

### The options that are now hard errors

`target: es5` · `downlevelIteration` · `moduleResolution: node` / `node10` / `classic` · `module: amd` / `umd` / `systemjs` / `none` · `baseUrl` · `esModuleInterop: false` · `allowSyntheticDefaultImports: false` · `alwaysStrict: false` · (`outFile` went in 6.0)

**`baseUrl` deserves a special callout.** Roughly every "TypeScript path aliases" tutorial written before 2026 opens with `"baseUrl": "."`. That is now an error. Use `paths` with relative patterns instead — see [Part 3](./typescript-foundations.md#part-3).

### The catch: the ecosystem is split

TypeScript 7.0 shipped without a stable programmatic API, so anything that reached into the compiler's internals cannot run on it yet — **typescript-eslint**, **ts-jest**, **ts-morph**, and the template type-checkers behind **Vue**, **Svelte**, **Astro**, **Angular** and **MDX**. The [official guidance](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/) is to alias the old compiler where a tool still needs the old API:

```jsonc
// package.json
{ "devDependencies": { "typescript": "npm:@typescript/typescript6@^6.0.2" } }
```

So in September 2026 the realistic answers are: **new Node/backend project → TypeScript 7, no reservations**; **Vue/Svelte/Astro app, or anything using type-aware ESLint rules → TypeScript 6 until 7.1**. [Part 12](./typescript-modules-ecosystem.md#part-12) covers running both side by side. Re-check this before trusting it — 7.1 was expected around October 2026 and is designed to dissolve exactly this problem.

### New in 7.0 that you will actually notice

```bash
npx tsc --checkers 8       # parallel type-checking workers (default 4)
npx tsc --builders 4       # parallel project-reference builds
npx tsc --singleThreaded   # turn all parallelism off (useful when profiling)
```

And one genuine language-visible change: **template-literal types now split by Unicode code point**, not by UTF-16 code unit. `` `${infer Head}${infer Tail}` `` applied to `"😀abc"` now yields `["😀", "abc"]` instead of a broken surrogate half. If you wrote a string-manipulating type before 2026, it may quietly behave better now. ([Part 9](./typescript-type-level.md#part-9))

[↑ Back to top](#table-of-contents)

---

## 6. How to Read a TypeScript Error

<a id="overview-errors"></a>

TypeScript errors are famously long, and they are long for a reason: they are a **stack trace through the assignability check**. Read them from the bottom up — the last line is the actual disagreement.

```text
Type '{ id: number; name: string; }' is not assignable to type 'User'.      ← the claim that failed
  Property 'email' is missing in type '{ id: number; name: string; }'       ← why it failed
    but required in type 'User'. ts(2741)                                   ← the specific rule
```

Codes you will meet constantly, and what they usually mean:

| Code | Message shape | Usually means |
|---|---|---|
| `TS2322` | Type 'X' is not assignable to type 'Y' | The core assignability failure. Read bottom-up. |
| `TS2345` | Argument of type 'X' is not assignable to parameter of type 'Y' | The same thing, at a call site. |
| `TS2339` | Property 'x' does not exist on type 'Y' | You have not narrowed yet, or the type is wider than you think. |
| `TS18048` | 'x' is possibly 'undefined' | `strictNullChecks`. See [Part 4](./typescript-narrowing-functions.md#part-4). |
| `TS2532` / `TS18047` | Object is possibly 'undefined' / 'null' | Same family. |
| `TS2741` | Property 'x' is missing but required | A shape is incomplete. |
| `TS2353` / `TS2561` | Object literal may only specify known properties | **Excess property check** — only fires on fresh literals. [Part 6](./typescript-objects-classes.md#part-6). |
| `TS7006` | Parameter 'x' implicitly has an 'any' type | `noImplicitAny`. Annotate, or fix the context. |
| `TS2367` | This comparison appears unintentional | The types have no overlap — often a genuine bug. |
| `TS2589` | Type instantiation is excessively deep and possibly infinite | A recursive type ran away. [Part 9](./typescript-type-level.md#part-9). |

Two habits make errors tractable, and both are used throughout this set:

```ts
// 1. Ask what the checker actually thinks, instead of guessing.
//    Hover the value — or force the checker to print it by assigning it somewhere wrong.
const debug: 1 = someValue; // the error text contains someValue's real type

// 2. Bisect. Annotate the intermediate step; the error moves to whichever half is wrong.
const step1: Something = compute();   // error moves here → compute() is the problem
const step2 = transform(step1);       // error stays here → transform() is the problem
```

[↑ Back to top](#table-of-contents)

---

## 7. Deliberately Not Covered (and where to look instead)

<a id="overview-excluded"></a>

| Topic | Why it's out | Where to look |
|---|---|---|
| Framework component typing (Vue `defineProps`, React props/hooks, Angular) | Each is a doc-sized topic and moves with the framework, not the language | [Vue TS guide](https://vuejs.org/guide/typescript/overview.html); this repo's Vue and React docs under [`legacy/`](../legacy/) |
| JSX / TSX type checking | Only meaningful alongside a framework | Handbook [JSX](https://www.typescriptlang.org/docs/handbook/jsx.html) |
| Migrating a large JS codebase to TS | A strategy topic (`allowJs`, `checkJs`, strictness ramps), not a language one | Handbook [Migrating from JavaScript](https://www.typescriptlang.org/docs/handbook/migrating-from-javascript.html) |
| Testing (Vitest/Jest type testing, `expectTypeOf`) | Tooling-specific, and `ts-jest` is mid-migration to TS 7 | [Vitest testing types](https://vitest.dev/guide/testing-types) |
| Monorepo build orchestration | Project references are covered in [Part 3](./typescript-foundations.md#part-3); Turborepo/Nx orchestration is not a TypeScript topic | Tool docs |
| The compiler's own AST API (`ts.createProgram`, transformers) | 7.0 has no stable programmatic API; anything written now is likely wrong by 7.1 | Wait for the 7.1 announcement |
| Typing the DOM | A library-surface topic, not a language one | `lib.dom.d.ts`, [MDN](https://developer.mozilla.org/) |

[↑ Back to top](#table-of-contents)

---

## 8. Suggested Learning Order

<a id="overview-order"></a>

**If you are new to TypeScript and reading front to back:**

1. [Part 1 — What TypeScript actually is](./typescript-foundations.md#part-1) (erasure, structural typing). Do not skip; it prevents a class of confusion that otherwise lasts months.
2. [Part 2 — Everyday types](./typescript-foundations.md#part-2). Enough to annotate real code.
3. [Part 4 — Narrowing](./typescript-narrowing-functions.md#part-4). This is where TypeScript starts feeling powerful rather than obstructive.
4. [Part 5 — Functions](./typescript-narrowing-functions.md#part-5) and [Part 6 — Object types](./typescript-objects-classes.md#part-6).
5. [Part 3 — tsconfig](./typescript-foundations.md#part-3). Deliberately *fifth*, not first: config makes sense once you have felt what the flags do.
6. [Part 8 — Generics](./typescript-generics.md#part-8), then [Part 11 — Modules](./typescript-modules-ecosystem.md#part-11).
7. [Part 7 — Classes](./typescript-objects-classes.md#part-7) when you need them; [Parts 9–10 — Type-level](./typescript-type-level.md) when the library types you read stop making sense; [Part 12 — Ecosystem](./typescript-modules-ecosystem.md#part-12) when you set up a real project.

**If you already write annotated TypeScript and want the parts that actually pay:** Part 4 Advanced/Mastery (control-flow analysis, exhaustiveness) → Part 8 (inference sites — the single highest-leverage section in this set) → Part 9 (conditional and mapped types) → Part 11 (ESM/CJS, `verbatimModuleSyntax`) → Part 3 Mastery (project references, `isolatedDeclarations`).

**If you are here because a specific thing is broken:** use the sub-doc table in [§2](#2-the-sub-docs) — the "Read it when" column is written for exactly that.

[↑ Back to top](#table-of-contents)

---

## 9. Quick Self-Check

<a id="overview-selfcheck"></a>

If you can answer these without looking, you have the material. Each links to where it is covered.

1. Why can't you write `x instanceof MyInterface`? What *can* you write instead? — [Part 1](./typescript-foundations.md#part-1)
2. What does "structural typing" let you do that a nominal type system wouldn't? What does it let you do that you'd rather it didn't? — [Part 1](./typescript-foundations.md#part-1)
3. `unknown` vs `any` vs `never`: which is assignable to which, and in which direction? — [Part 2](./typescript-foundations.md#part-2)
4. Which two commonly-wanted strictness flags does `strict: true` *not* turn on? — [Part 3](./typescript-foundations.md#part-3)
5. Why does `arr[0]` have type `T` by default, and what changes that? — [Part 3](./typescript-foundations.md#part-3)
6. Why does narrowing get "lost" inside a callback, and what are the two standard fixes? — [Part 4](./typescript-narrowing-functions.md#part-4)
7. Write a type predicate and an assertion function for the same check. When do you need each? — [Part 4](./typescript-narrowing-functions.md#part-4)
8. How do you make the compiler fail the build when someone adds a case to a union? — [Part 4](./typescript-narrowing-functions.md#part-4)
9. Why is `(a: string) => void` assignable to `(a: string, b: number) => void` but not the reverse? — [Part 5](./typescript-narrowing-functions.md#part-5)
10. When does an object literal get excess-property-checked, and when does the same object sail through? — [Part 6](./typescript-objects-classes.md#part-6)
11. `interface` vs `type`: name one thing each can do that the other cannot. — [Part 6](./typescript-objects-classes.md#part-6)
12. What's the difference between `private name: string` and `#name`? Which survives to runtime? — [Part 7](./typescript-objects-classes.md#part-7)
13. Why does `function first<T>(arr: T[])` called with `["a", "b"]` infer `string[]` and not `["a", "b"]`, and what two things change that? — [Part 8](./typescript-generics.md#part-8)
14. What is a "distributive" conditional type, and how do you switch distribution off? — [Part 9](./typescript-type-level.md#part-9)
15. Implement `Omit` yourself. Then explain why the built-in one doesn't error on a key that doesn't exist. — [Part 10](./typescript-type-level.md#part-10)
16. `as` vs `satisfies`: what does each do to the resulting type? — [Part 9](./typescript-type-level.md#part-9)
17. What does `verbatimModuleSyntax` change about your emitted output, and why does `import type` matter? — [Part 11](./typescript-modules-ecosystem.md#part-11)
18. Node runs your `.ts` file. Did it type-check it? Which syntax will it refuse outright? — [Part 12](./typescript-modules-ecosystem.md#part-12)
19. You `JSON.parse` a response and annotate it as `User`. What has the compiler guaranteed? — [Part 12](./typescript-modules-ecosystem.md#part-12)
20. Which tools currently block a TypeScript 7 upgrade, and what's the interim fix? — [§5](#5-the-2026-compiler-landscape)

[↑ Back to top](#table-of-contents)

---

## 10. Cross-Doc Cheat Sheet

<a id="overview-cheatsheet"></a>

### The special types

| Type | Assignable **to** it | Assignable **from** it | Use it for |
|---|---|---|---|
| `unknown` | everything | nothing (must narrow first) | the safe top type — parsed JSON, `catch` variables |
| `any` | everything | everything | escape hatch; switches checking off in both directions |
| `never` | nothing | everything | impossible states, exhaustiveness, "this function doesn't return" |
| `void` | anything (in return position) | `undefined` | "ignore my return value" |
| `object` | any non-primitive | — | rarely what you want; prefer `Record<string, unknown>` |
| `{}` | anything except `null`/`undefined` | — | almost never what you want |

### Type-level operators, at a glance

| Syntax | Reads as | Part |
|---|---|---|
| `keyof T` | union of T's keys | [8](./typescript-generics.md#part-8) |
| `typeof x` (in type position) | the type of the *value* `x` | [8](./typescript-generics.md#part-8) |
| `T[K]` | indexed access — the type at key `K` | [8](./typescript-generics.md#part-8) |
| `T extends U ? A : B` | conditional type | [9](./typescript-type-level.md#part-9) |
| `infer X` | bind a matched type to a variable | [9](./typescript-type-level.md#part-9) |
| `{ [K in keyof T]: … }` | mapped type | [9](./typescript-type-level.md#part-9) |
| `{ [K in keyof T as …]: … }` | mapped type with key remapping | [9](./typescript-type-level.md#part-9) |
| `` `a-${string}` `` | template literal type | [9](./typescript-type-level.md#part-9) |
| `x satisfies T` | check against `T`, keep the narrow type | [9](./typescript-type-level.md#part-9) |
| `x as T` | assert — the checker believes you, verifies nothing | [2](./typescript-foundations.md#part-2) |
| `x as const` | infer the narrowest literal / `readonly` type | [2](./typescript-foundations.md#part-2) |

### The flags worth knowing by heart

| Flag | In `strict`? | What it buys |
|---|---|---|
| `strictNullChecks` | ✅ | `null` / `undefined` are real, distinct types |
| `noImplicitAny` | ✅ | un-inferrable parameters must be annotated |
| `strictFunctionTypes` | ✅ | contravariant parameter checking for function types |
| `strictPropertyInitialization` | ✅ | class fields must be assigned |
| `useUnknownInCatchVariables` | ✅ | `catch (e)` gives `unknown`, not `any` |
| `noUncheckedIndexedAccess` | ❌ | `arr[i]` is `T \| undefined` — the big one |
| `exactOptionalPropertyTypes` | ❌ | `{ a?: string }` ≠ `{ a: string \| undefined }` |
| `noImplicitOverride` | ❌ | `override` keyword required |
| `verbatimModuleSyntax` | ❌ | imports emit exactly as written |
| `erasableSyntaxOnly` | ❌ | bans syntax runtimes can't strip (enums, namespaces) |
| `isolatedDeclarations` | ❌ | every `.d.ts` emittable from one file alone |

### TypeScript 7 CLI

```bash
npx tsc --noEmit              # check only
npx tsc --watch --noEmit      # check on save
npx tsc --build               # project references (composite builds)
npx tsc --checkers 8          # parallel checkers (default 4, TS 7+)
npx tsc --singleThreaded      # disable parallelism
npx tsc --showConfig          # print the fully-resolved config — settles all arguments
npx tsc --traceResolution     # why did that import resolve there?
npx tsc --explainFiles        # why is that file in my program at all?
npx tsc --generateTrace out/  # perf trace for slow builds
```

[↑ Back to top](#table-of-contents)

---

*Part of the [my-learning-md-docs](../README.md) collection. Prerequisite: the [JavaScript doc set](../javascript/javascript-mastery-guide.md). Last updated: September 6, 2026.*
