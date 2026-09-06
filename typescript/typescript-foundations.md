# TypeScript — Foundations (Parts 1–3)

> **About this document:** this is one of the sub-docs in this repo's TypeScript set. The canonical spec — framing, scope, tier policy, sourcing rule, versions, and how to update it — lives once in [`typescript-mastery-guide.md`](./typescript-mastery-guide.md#about-this-document). Written against **TypeScript 7.0**, September 6, 2026. Assumes the [JavaScript doc set](../javascript/javascript-mastery-guide.md); JavaScript concepts are linked, not re-explained.

Parts 1–3 are the substrate: what the compiler is actually doing, the types you will write every day, and the configuration that decides how strictly it all gets checked.

---

## Table of Contents

- [Part 1 — What TypeScript Actually Is](#part-1)
  - [Beginner — types are deleted](#p1-beginner)
  - [Working Knowledge — structural typing](#p1-working)
  - [Advanced — where the erasure boundary bites](#p1-advanced)
  - [Mastery — the type/value/namespace declaration spaces](#p1-mastery)
  - [Part 1 cheat sheet](#p1-cheatsheet)
- [Part 2 — Everyday Types](#part-2)
  - [Beginner — annotations and primitives](#p2-beginner)
  - [Working Knowledge — unions, literals, arrays, tuples, inference](#p2-working)
  - [Advanced — `any`, `unknown`, `never`, and assertions](#p2-advanced)
  - [Mastery — widening, freshness, and the assignability lattice](#p2-mastery)
  - [Part 2 cheat sheet](#p2-cheatsheet)
- [Part 3 — `tsconfig` and the Compiler In Depth](#part-3)
  - [Working Knowledge — the file, the flags, the strict family](#p3-working)
  - [Advanced — modules, emit, and the flags `strict` forgot](#p3-advanced)
  - [Mastery — project references, `isolatedDeclarations`, diagnosing the compiler](#p3-mastery)
  - [Part 3 cheat sheet](#p3-cheatsheet)

---

<a id="part-1"></a>

## Part 1 — What TypeScript Actually Is

<a id="p1-beginner"></a>

### Beginner — types are deleted

TypeScript adds syntax to JavaScript for describing the *shape* of values, then throws all of it away before your code runs. The [Handbook's opening](https://www.typescriptlang.org/docs/handbook/2/basic-types.html) frames it as "JavaScript with types" and that is literally true: remove every type annotation from a `.ts` file and you have the `.js` file that ships.

```ts
// greet.ts
interface User {
  name: string;
  age: number;
}

function greet(user: User): string {
  return `Hello, ${user.name}`;
}
```

```js
// greet.js — what actually runs. The interface is GONE. The annotations are GONE.
function greet(user) {
  return `Hello, ${user.name}`;
}
```

The interface did not become a class, an object, or a validator. It produced **zero bytes** of output. It existed only to let the checker complain at you.

**Try It.** Paste this into the [playground](https://www.typescriptlang.org/play) and watch the ".JS" output pane while you type:

```ts
type Point = { x: number; y: number };
const p: Point = { x: 1, y: 2 };
const q = p as Point;
const r = p satisfies Point;
```

*Expected:* the JS pane shows only `const p = { x: 1, y: 2 }; const q = p; const r = p;`. `type`, `: Point`, `as Point` and `satisfies Point` all vanish. That is the whole language in one screenshot.

<a id="p1-working"></a>

### Working Knowledge — structural typing

The second load-bearing idea: TypeScript compares types by **shape**, not by name. This is called structural typing (or "duck typing, checked at compile time"), and it is why you almost never write `implements`.

```ts
interface Named { name: string }

class Dog {
  constructor(public name: string) {}      // note: no `implements Named`
}

function announce(thing: Named) { console.log(thing.name); }

announce(new Dog("Rex"));          // ✅ fine — Dog has a `name: string`
announce({ name: "Anonymous" });   // ✅ fine — so does this object literal
```

Nothing declared a relationship between `Dog` and `Named`. The checker just asked "does `Dog` have everything `Named` requires?" and the answer was yes. Contrast Java or C#, where `Dog` would have to say `implements Named` to qualify — that is *nominal* typing.

This buys you enormous flexibility, and costs you one thing: **two unrelated concepts with the same shape are the same type.**

```ts
type UserId = { id: string };
type OrderId = { id: string };

function loadUser(u: UserId) { /* … */ }
const order: OrderId = { id: "ord_123" };

loadUser(order);   // ✅ no error. Structurally identical. This is probably a bug.
```

The standard fix is **branding** — adding a phantom property that exists only in the type system, so the shapes differ:

```ts
// ❌ Wrong — plain aliases of `string` are interchangeable
type UserId = string;
type OrderId = string;
declare function loadUser(id: UserId): void;
loadUser("ord_123" as OrderId);   // no error; wrong ID silently accepted

// ✅ Right — branded types are structurally distinct
declare const brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [brand]: B };

type UserId  = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;

const makeUserId = (s: string) => s as UserId;   // one blessed constructor

declare function loadUser2(id: UserId): void;
loadUser2("ord_123" as OrderId);
// Error: Argument of type 'OrderId' is not assignable to parameter of type 'UserId'.
//   Type '"OrderId"' is not assignable to type '"UserId"'. ts(2345)
```

The brand costs nothing at runtime — `Brand<string, "UserId">` still *is* a string when the types are stripped. You are lying to the type system in exactly one place (`as UserId`, inside `makeUserId`) so it can tell the truth everywhere else. `unique symbol` is used for the brand key because it can never collide with a real property name; see [Symbols](../javascript/javascript-metaprogramming-modern.md) in the JavaScript set.

<a id="p1-advanced"></a>

### Advanced — where the erasure boundary bites

Once you believe "types are deleted", a family of errors becomes obvious rather than mysterious. All of these are the same mistake: **asking a runtime question of a compile-time thing.**

```ts
interface Animal { name: string }

// ❌ 1. Interfaces have no runtime identity
if (x instanceof Animal) {}
// Error: 'Animal' only refers to a type, but is being used as a value here. ts(2693)

// ❌ 2. Type parameters have no runtime identity either
function make<T>(): T {
  return new T();   // Error: 'T' only refers to a type… ts(2693)
}

// ❌ 3. You cannot switch on a type
function area(shape: Circle | Square) {
  switch (typeof shape) { /* both are "object" — typeof can't see your types */ }
}
```

The fix is always the same: **put something at runtime that the check can look at.** Usually a discriminant property (covered properly in [Part 4](./typescript-narrowing-functions.md#part-4)), sometimes a factory function, sometimes a class (classes *do* exist at runtime, so `instanceof` works on them).

```ts
// ✅ 1 & 3 — a discriminant field that survives erasure
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number };

function area(s: Shape) {
  return s.kind === "circle" ? Math.PI * s.radius ** 2 : s.side ** 2;
}

// ✅ 2 — pass a factory, not a type
function make<T>(factory: () => T): T { return factory(); }
```

**Real Scenario — the `any` that got through.** This one bites in production, not in the editor, which is what makes it worth a scenario rather than a Try It:

```ts
const res = await fetch("/api/user");
const user: User = await res.json();     // res.json() returns Promise<any>
console.log(user.profile.displayName);   // compiles clean. Crashes at 3am.
```

The API changed `profile` to `null` for deleted accounts. TypeScript reported nothing, because `res.json()` is typed `Promise<any>` and `any` is assignable to `User` without inspection. The annotation was a *claim*, and the claim was false. Everything inside your program was type-safe; the boundary was not. This is the entire argument for runtime validation at I/O boundaries — see [Part 12](./typescript-modules-ecosystem.md#part-12).

```text
   ┌─────────────── your program ────────────────┐
   │                                             │
   │   the checker's guarantees hold here        │
   │                                             │
   └──▲───────────────────────────────────────▲──┘
      │                                       │
   ═══╪═══════════ TRUST BOUNDARY ════════════╪═══
      │                                       │
  fetch / JSON.parse                   process.env
  DB driver rows                       user input, CLI args
  `as` assertions                      third-party `any`

  Everything crossing this line is a CLAIM until something
  validates it at runtime.
```

<a id="p1-mastery"></a>

### Mastery — the type/value/namespace declaration spaces

Here is the mental model that makes `TS2693` ("only refers to a type, but is being used as a value") permanently legible. Every name in TypeScript lives in up to **three separate declaration spaces**, and different declaration forms populate different ones:

```text
  declaration form        value space   type space   namespace space
  ──────────────────────────────────────────────────────────────────
  const / let / var           ✅            —              —
  function                    ✅            —              —
  interface                   —             ✅             —
  type alias                  —             ✅             —
  class                       ✅            ✅             —      ← both!
  enum                        ✅            ✅             —      ← both!
  namespace                   (✅)          —              ✅
  import                      ✅            ✅             ✅
```

Two spaces means a name can mean *different things* depending on where you write it:

```ts
class Point { constructor(public x: number, public y: number) {} }

const P = Point;          // value position → the constructor function
let p: Point;             // type position  → an INSTANCE of Point
let ctor: typeof Point;   // `typeof` bridges value → type: the constructor's type
```

That last line is the key trick: **`typeof` in type position reads a name out of the value space and gives you its type.** It is how you type a class constructor, a config object, or anything whose shape you would rather not re-declare:

```ts
const config = { retries: 3, endpoint: "https://api.example.com" };
type Config = typeof config;   // { retries: number; endpoint: string }
```

And the reverse bridge — reading a type out of a value that carries both — is why `class` is convenient and `interface` is not:

```ts
interface Animal { name: string }
const a: Animal = { name: "x" };
a instanceof Animal;   // ❌ TS2693 — Animal has no value-space entry

class Cat { name = "x" }
new Cat() instanceof Cat;   // ✅ Cat exists in both spaces
```

The same rule explains why `import type { Foo }` is different from `import { Foo }`: the first imports only the type-space entry, guaranteeing the import is erased. That has real emit consequences — see [`verbatimModuleSyntax`](./typescript-modules-ecosystem.md#part-11).

<a id="p1-cheatsheet"></a>

### Part 1 cheat sheet

| Question | Answer |
|---|---|
| Does an `interface` exist at runtime? | No. Zero bytes emitted. |
| Does a `class` exist at runtime? | Yes — it's in both the value and type spaces. |
| Does an `enum` exist at runtime? | Yes (that's why `erasableSyntaxOnly` bans it — [Part 3](#part-3)). |
| How do types relate to each other? | **Structurally** — by shape, never by name. |
| How do I make two same-shaped types distinct? | Brand them with a `unique symbol` phantom property. |
| How do I get a type from a value? | `typeof value` in type position. |
| How do I check a type at runtime? | You can't. Check a *value* — a discriminant, `typeof`, `instanceof` on a class, or a schema validator. |
| What is `TS2693` telling me? | You used a type-space name in a value position. |

[↑ Back to top](#table-of-contents)

---

<a id="part-2"></a>

## Part 2 — Everyday Types

<a id="p2-beginner"></a>

### Beginner — annotations and primitives

An annotation is a colon and a type after a name. The primitives mirror JavaScript's, in **lowercase**:

```ts
let title: string = "Dune";
let pages: number = 412;          // one number type — no int/float distinction
let big: bigint = 9007199254740993n;
let inPrint: boolean = true;
let nothing: null = null;
let missing: undefined = undefined;
let key: symbol = Symbol("id");
```

```ts
// ❌ Wrong — the capitalised ones are the wrapper-object types, not the primitives
let name: String = "Ada";   // technically legal, always wrong
// ✅ Right
let name2: string = "Ada";
```

`String`/`Number`/`Boolean` describe boxed wrapper *objects*. They accept primitives (which is why the mistake survives) but not vice versa, and they'll fail in surprising places. Always lowercase.

**The most important beginner habit: don't annotate what's obvious.** TypeScript infers, and inferred types are usually better than the ones you'd write:

```ts
// ❌ Noise — the initialiser already says everything
let count: number = 0;
const names: string[] = ["a", "b"];

// ✅ Let it infer
let count2 = 0;               // number
const names2 = ["a", "b"];    // string[]
```

Annotate at **boundaries** — function parameters, exported return types, empty containers — and let inference handle the interior.

<a id="p2-working"></a>

### Working Knowledge — unions, literals, arrays, tuples, inference

**Unions** (`A | B`) say "one of these". They are the single most-used type-level tool in the language:

```ts
type Status = "idle" | "loading" | "error";     // union of string LITERAL types
let s: Status = "idle";
s = "loadng";   // Error: Type '"loadng"' is not assignable to type 'Status'. ts(2322)
```

Those quoted strings are **literal types** — a type inhabited by exactly one value. They are what make `Status` better than `string`: typos become compile errors, and your editor autocompletes the three valid values.

The rule that trips everyone: **on a union, you may only access members that exist on *every* member** until you narrow.

```ts
function len(x: string | string[]) {
  return x.length;        // ✅ both have .length
}
function upper(x: string | string[]) {
  return x.toUpperCase(); // ❌ Property 'toUpperCase' does not exist on type 'string | string[]'. ts(2339)
}
```

Fixing that is [Part 4](./typescript-narrowing-functions.md#part-4)'s entire job.

**Arrays and tuples:**

```ts
const xs: number[] = [1, 2, 3];
const ys: Array<number> = [1, 2, 3];        // identical; pick one and be consistent
const pair: [string, number] = ["a", 1];    // tuple — fixed length, per-position types
const named: [x: number, y: number] = [1, 2];       // labels are documentation only
const rest: [string, ...number[]] = ["a", 1, 2, 3]; // variadic tuple
const ro: readonly string[] = ["a"];        // no push/pop/sort — see Part 6
```

**Objects and optionals:**

```ts
type User = {
  id: string;
  name: string;
  nickname?: string;         // optional → type is `string | undefined`
  readonly createdAt: Date;  // assignable at creation, not after
};
```

**Functions:**

```ts
function add(a: number, b: number): number { return a + b; }
const mul = (a: number, b: number): number => a * b;
type BinaryOp = (a: number, b: number) => number;   // function type, arrow syntax
```

**Try It — watch inference vs. annotation diverge.**

```ts
let a = "hello";            // inferred: string   (let → widened)
const b = "hello";          // inferred: "hello"  (const → literal, it can't change)
const c = { kind: "circle" };        // inferred: { kind: string }  ← widened!
const d = { kind: "circle" } as const; // inferred: { readonly kind: "circle" }
```

*Expected:* hover each. The third one surprises people constantly — a `const` object's *properties* are still mutable, so TypeScript widens them to `string`. That single behaviour is behind a large share of "why doesn't my discriminated union work" questions. `as const` is the fix.

<a id="p2-advanced"></a>

### Advanced — `any`, `unknown`, `never`, and assertions

These four (with `void`) are where TypeScript's type system shows its shape. Get them right and a lot else follows.

```text
                    ┌───────────────┐
                    │    unknown    │   top: everything is assignable TO it
                    └───────┬───────┘
                            │
        ┌───────────┬───────┴───────┬────────────┐
        │           │               │            │
     string      number          boolean      object…
        │           │               │            │
    "a" │ "b"    1 │ 2 │ 3       true │ false   { … }      ← literal types
        │           │               │            │
        └───────────┴───────┬───────┴────────────┘
                            │
                    ┌───────┴───────┐
                    │     never     │   bottom: assignable TO everything
                    └───────────────┘

    `any` is NOT on this lattice. It sits beside it, assignable in
    BOTH directions, and disables the check wherever it appears.
```

**`any` — the off switch.** Assignable to and from everything. One `any` propagates silently through every expression it touches:

```ts
const data: any = JSON.parse(raw);
const n: number = data.user.name;    // no error. `name` is a string. Good luck.
```

Use it as a deliberate, commented, local escape hatch, never as a shrug.

**`unknown` — the safe top type.** Everything is assignable *to* `unknown`; `unknown` is assignable to *nothing* until you narrow it. It is what `any` should have been:

```ts
// ❌ Wrong
function handle(e: any) { console.log(e.message.toUpperCase()); }   // may crash

// ✅ Right
function handle2(e: unknown) {
  if (e instanceof Error) console.log(e.message.toUpperCase());     // narrowed
  else console.log(String(e));
}
```

Under `strict` this is already the default for `catch` clauses, via [`useUnknownInCatchVariables`](https://www.typescriptlang.org/tsconfig/#useUnknownInCatchVariables) (TypeScript 4.4+): `catch (e)` gives you `unknown`, because JavaScript lets you `throw 42`.

**`never` — the bottom type.** No value has type `never`. It shows up in three places, and all three are useful:

```ts
function fail(msg: string): never { throw new Error(msg); }   // 1. never returns
type Impossible = string & number;                            // 2. empty intersection
// 3. exhaustiveness checking — see Part 4, the single best use of `never`
```

**`void` vs `undefined`.** `void` means "I return nothing; ignore my return value". Its special rule is that a function returning *anything* is assignable to a `void`-returning function type — which is why `arr.forEach(x => arr2.push(x))` compiles despite `push` returning a number.

```ts
type Cb = () => void;
const cb: Cb = () => 42;    // ✅ allowed — the return value is simply ignored
const r = cb();             // r: void — you cannot use the 42
```

**Assertions (`as`) — you take responsibility.**

```ts
const el = document.getElementById("root") as HTMLCanvasElement;   // trust me
const n = "5" as unknown as number;   // the double-assertion escape hatch
```

`as` cannot convert between unrelated types in one step (`"5" as number` is `TS2352`), which is the compiler's last line of defence — and the double assertion through `unknown` bypasses it. Every `as` is a place where a runtime bug can hide; treat them as a countable, reviewable thing.

**`as const` — the opposite of an assertion, and far more useful:**

```ts
const ROUTES = { home: "/", about: "/about" } as const;
// { readonly home: "/"; readonly about: "/about" }
type Route = typeof ROUTES[keyof typeof ROUTES];   // "/" | "/about"
```

That pattern — a `const` object plus `as const` plus an indexed access — is the modern replacement for `enum`, and unlike `enum` it emits no runtime code and survives `erasableSyntaxOnly`. See [Part 3](#part-3).

<a id="p2-mastery"></a>

### Mastery — widening, freshness, and the assignability lattice

Three behaviours explain most "why is my type wider/narrower than I expected" moments.

**1. Literal widening.** A literal type widens to its base type when it's assigned to a mutable location, because a mutable location could later hold a different value:

```ts
const a = "circle";        // "circle"        (immutable binding → stays literal)
let   b = "circle";        // string          (widened)
const c = { k: "circle" }; // { k: string }   (property is mutable → widened)

declare function f(s: "circle" | "square"): void;
f(a);      // ✅
f(b);      // ❌ Argument of type 'string' is not assignable… ts(2345)
f(c.k);    // ❌ same
```

Three ways to stop widening, in increasing order of reach:

```ts
let b2 = "circle" as const;                      // 1. as const at the value
const c2 = { k: "circle" } as const;             // 2. as const on the object
const c3: { k: "circle" | "square" } = { k: "circle" };  // 3. annotate the target
```

There is a fourth, newer one for generic functions — `const` type parameters ([Part 8](./typescript-generics.md#part-8)).

**2. Freshness (excess property checking).** An object literal is "fresh" the moment you write it, and freshness makes the checker *stricter* than assignability alone:

```ts
type Options = { url: string; timeout?: number };

// ❌ fresh literal → excess property check fires
const o1: Options = { url: "/x", timout: 500 };
// Error: Object literal may only specify known properties, but 'timout' does not exist
//   in type 'Options'. Did you mean to write 'timeout'? ts(2561)

// ✅ … and the same object assigned via a variable sails through
const raw = { url: "/x", timout: 500 };
const o2: Options = raw;    // no error! `raw` is not fresh; it's just assignable.
```

This is not a bug — it is a deliberate ergonomic patch. Structurally, `{url, timout}` *is* a valid `Options` (extra properties don't break assignability), so without the freshness rule the typo would never be caught. Full treatment in [Part 6](./typescript-objects-classes.md#part-6).

**3. Assignability is the only question.** Formally, `S` is assignable to `T` if every member `T` requires is present in `S` with an assignable type, with special rules for functions (parameters are checked *contra*variantly — [Part 5](./typescript-narrowing-functions.md#part-5)), unions (`S` assignable to `T1 | T2` if assignable to either), and intersections (assignable to `T1 & T2` if assignable to both).

**Try It — the lattice, empirically.**

```ts
declare let unk: unknown;
declare let nev: never;
declare let str: string;

unk = str;   // ✅ everything → unknown
str = unk;   // ❌ Type 'unknown' is not assignable to type 'string'. ts(2322)
str = nev;   // ✅ never → everything
nev = str;   // ❌ Type 'string' is not assignable to type 'never'. ts(2322)
```

*Expected:* two errors, on lines 2 and 4 of that block. If you can predict those four results cold, you understand the lattice.

<a id="p2-cheatsheet"></a>

### Part 2 cheat sheet

| Want | Write |
|---|---|
| One of several values | `type S = "a" \| "b"` |
| Array | `T[]` or `Array<T>` |
| Immutable array | `readonly T[]` |
| Fixed-shape list | `[string, number]` |
| Optional property | `{ x?: number }` |
| Read-only property | `{ readonly x: number }` |
| Function type | `(a: string) => number` |
| Type of an existing value | `typeof value` |
| Keys of a type as a union | `keyof T` |
| Freeze literal inference | `as const` |
| Untyped-but-safe input | `unknown` (never `any`) |
| "This can't happen" | `never` |

| Symptom | Cause | Fix |
|---|---|---|
| `Type 'string' is not assignable to type '"a" \| "b"'` | literal widening | `as const`, or annotate the target |
| `Object literal may only specify known properties` | freshness / excess property check | fix the typo, or assign through a variable if genuinely intended |
| `Property 'x' does not exist on type 'A \| B'` | unnarrowed union | narrow it ([Part 4](./typescript-narrowing-functions.md#part-4)) |
| `'unknown' is not assignable to…` | working as intended | narrow before use |
| Everything silently compiles but crashes | an `any` upstream | find it: `npx tsc --noEmit` won't help — search for `any`, `as`, and `.json()` |

[↑ Back to top](#table-of-contents)

---

<a id="part-3"></a>

## Part 3 — `tsconfig` and the Compiler In Depth

> **Tier note:** this Part runs **Working Knowledge → Mastery**, with no Beginner tier. In 2026 the beginner config is `{}` — since [TypeScript 6.0](https://devblogs.microsoft.com/typescript/announcing-typescript-6-0/) the defaults are already `strict: true`, `module: esnext`, `target: es2025`. There is nothing left to teach at that level.

<a id="p3-working"></a>

### Working Knowledge — the file, the flags, the strict family

`tsconfig.json` marks a directory as the root of a TypeScript **project** and tells the compiler three things: which files are in the program, how to check them, and what to emit.

```jsonc
{
  "compilerOptions": { /* how to check and emit */ },
  "include": ["src"],            // globs of files IN the program
  "exclude": ["**/*.test.ts"],   // filters `include`; does NOT exclude imported files
  "extends": "@tsconfig/node24/tsconfig.json"   // inherit a base config
}
```

**`exclude` does not do what most people think.** It only filters what `include` picked up. If an included file *imports* an excluded file, the excluded file is still pulled into the program. To genuinely keep a file out, stop importing it.

The strictness flags, and the crucial fact about them:

| Flag | Included in `strict`? | What it does |
|---|---|---|
| `noImplicitAny` | ✅ | error when a parameter's type can't be inferred |
| [`strictNullChecks`](https://www.typescriptlang.org/tsconfig/#strictNullChecks) | ✅ | `null`/`undefined` become distinct types you must handle |
| `strictFunctionTypes` | ✅ | function parameter positions checked contravariantly |
| `strictBindCallApply` | ✅ | `bind`/`call`/`apply` are type-checked |
| `strictPropertyInitialization` | ✅ | class fields must be definitely assigned |
| `noImplicitThis` | ✅ | error on an implicitly-`any` `this` |
| [`useUnknownInCatchVariables`](https://www.typescriptlang.org/tsconfig/#useUnknownInCatchVariables) | ✅ | `catch (e)` → `unknown` |
| `alwaysStrict` | ✅ | emit `"use strict"` (and in TS 7, cannot be turned off) |
| [`noUncheckedIndexedAccess`](https://www.typescriptlang.org/tsconfig/#noUncheckedIndexedAccess) | ❌ | `arr[i]` and index signatures include `undefined` |
| [`exactOptionalPropertyTypes`](https://www.typescriptlang.org/tsconfig/#exactOptionalPropertyTypes) | ❌ | `{a?: string}` won't accept an explicit `undefined` |
| `noImplicitOverride` | ❌ | subclass overrides must say `override` |
| `noPropertyAccessFromIndexSignature` | ❌ | index-signature keys need `obj["key"]`, not `obj.key` |
| `noFallthroughCasesInSwitch` | ❌ | catch missing `break` |
| `noUnusedLocals` / `noUnusedParameters` | ❌ | dead-code hygiene (better handled by ESLint) |

**`strictNullChecks` is the one that matters most.** With it off, `null` and `undefined` inhabit every type and TypeScript cannot warn you about the most common JavaScript crash there is. It has been in `strict` forever and `strict` is now the default — but if you inherit a codebase with `"strict": false`, this is the flag to turn on first, and it will produce hundreds of errors that are all real.

**`noUncheckedIndexedAccess` is the one people miss**, because it sounds obscure and is not in `strict`:

```ts
const arr = [1, 2, 3];

// without the flag — a lie the type system tells you
const first = arr[10];        // number   ← it is actually undefined
first.toFixed(2);             // compiles. crashes.

// with "noUncheckedIndexedAccess": true
const first2 = arr[10];       // number | undefined
first2.toFixed(2);            // Error: 'first2' is possibly 'undefined'. ts(18048)
first2?.toFixed(2);           // ✅
```

It is noisy in loop-heavy code — that is the honest tradeoff — but it is the difference between a type system that models array access accurately and one that doesn't. Turn it on in new projects.

**`exactOptionalPropertyTypes`** closes the other hole — the difference between "absent" and "present but `undefined`":

```ts
type Settings = { theme?: "dark" | "light" };

const s: Settings = {};                 // ✅ absent
const t: Settings = { theme: undefined }; // ❌ with the flag: ts(2375)
//   Type '{ theme: undefined; }' is not assignable to type 'Settings' with
//   'exactOptionalPropertyTypes: true'.
```

That distinction matters the moment you spread objects, `JSON.stringify` them (which drops `undefined` values), or write them to a database.

<a id="p3-advanced"></a>

### Advanced — modules, emit, and the flags `strict` forgot

**`module` and `moduleResolution` are the two options that cause the most confusion**, so here is the decision procedure rather than the option list:

```text
  What consumes your output?
  │
  ├─ Node runs it directly (a server, a CLI)
  │     "module": "nodenext"          ← moduleResolution follows automatically
  │     package.json "type": "module" (or "commonjs" — nodenext honours it per-file)
  │
  ├─ A bundler consumes it (Vite, esbuild, webpack, Rollup)
  │     "module": "esnext"  (the TS 7 default) or "preserve"
  │     "moduleResolution": "bundler"
  │     "noEmit": true      ← the bundler emits; tsc only checks
  │
  └─ You're publishing a library
        "module": "nodenext", "declaration": true
        and read Part 11 on dual ESM/CJS publishing before you ship
```

**Dead options you will still see in blog posts.** TypeScript 7 makes each of these a hard error:

```jsonc
// ❌ Every one of these is an error in TypeScript 7
{
  "compilerOptions": {
    "target": "es5",                  // removed
    "moduleResolution": "node",       // removed (node10/classic too)
    "module": "commonjs",             // still valid, but no longer the default
    "baseUrl": ".",                   // removed — see below
    "esModuleInterop": false,         // removed; interop is now always on
    "downlevelIteration": true        // removed (unnecessary without es5)
  }
}
```

**Path aliases without `baseUrl`.** The old recipe was `"baseUrl": "."` plus `"paths": {"@/*": ["src/*"]}`. `baseUrl` is gone; `paths` patterns are now resolved relative to the `tsconfig.json` itself:

```jsonc
// ✅ TypeScript 7
{
  "compilerOptions": {
    "paths": { "@/*": ["./src/*"] }
  }
}
```

Remember that `paths` only teaches **the type checker** where to look. Your bundler or runtime needs the same mapping configured independently (`resolve.alias` in Vite, `imports` in `package.json` for Node). A `paths` entry that your runtime doesn't know about is a `MODULE_NOT_FOUND` waiting for deployment. Node's own [subpath imports](https://nodejs.org/api/packages.html#subpath-imports) (`"imports": { "#lib/*": "./src/*.js" }`) are the runtime-honest alternative, and TypeScript 6 added support for `#/`-prefixed subpath imports.

**Emit options worth knowing:**

| Option | Effect |
|---|---|
| `noEmit` | check only; someone else builds. The normal setting when a bundler is involved. |
| `noEmitOnError` | don't write output if there were errors. **Not** the default — `tsc` emits broken-typed JS happily. |
| `declaration` / `declarationMap` | emit `.d.ts` (+ source maps so "go to definition" lands in your `.ts`) |
| `sourceMap` | emit `.js.map` for debugging |
| `incremental` / `tsBuildInfoFile` | cache program state between runs |
| [`verbatimModuleSyntax`](./typescript-modules-ecosystem.md#part-11) | imports/exports emit exactly as written |
| `isolatedModules` | error on syntax that can't be transpiled file-by-file — required if a bundler does your emit |
| `erasableSyntaxOnly` | error on `enum`, `namespace` with runtime code, parameter properties — anything a stripper can't erase ([5.8+](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html)) |

**`erasableSyntaxOnly` deserves the emphasis.** It exists because runtimes now strip types themselves (Node, esbuild, `tsx`), and stripping can only delete syntax — it can't *generate* the object an `enum` needs:

```ts
// ❌ Not erasable — enum must EMIT a runtime object
enum Color { Red, Green }
// Error with erasableSyntaxOnly: This syntax is not allowed when
//   'erasableSyntaxOnly' is enabled. ts(1294)

// ❌ Not erasable — parameter properties emit assignments
class P { constructor(private x: number) {} }

// ✅ Erasable equivalents
const Color2 = { Red: 0, Green: 1 } as const;
type Color2 = typeof Color2[keyof typeof Color2];

class P2 {
  private x: number;
  constructor(x: number) { this.x = x; }
}
```

**Try It.** Put an `enum` in a file and run `node file.ts` on Node 24+:

*Expected:* `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX`. Now add `"erasableSyntaxOnly": true` to your tsconfig and run `npx tsc --noEmit`: you get `ts(1294)` at compile time instead of a runtime failure. That is the flag's entire value proposition — moving the failure earlier.

<a id="p3-mastery"></a>

### Mastery — project references, `isolatedDeclarations`, diagnosing the compiler

**Project references** split one giant program into several, each checked and cached independently. This is how monorepos stay fast and how you enforce dependency direction at the type level.

```jsonc
// packages/core/tsconfig.json
{ "compilerOptions": { "composite": true, "declaration": true, "outDir": "dist" } }

// packages/api/tsconfig.json
{
  "compilerOptions": { "composite": true, "outDir": "dist" },
  "references": [{ "path": "../core" }]      // api may import core; core may NOT import api
}
```

```bash
npx tsc --build            # builds the graph in dependency order, skipping up-to-date projects
npx tsc --build --clean
npx tsc --build --dry
npx tsc --builders 4       # TS 7: build referenced projects in parallel
```

`composite: true` forces `declaration: true` and a `tsBuildInfo` file, because references work by consuming the *emitted `.d.ts`* of the referenced project rather than its source. That is what makes builds incremental — and it is also why `.d.ts` emit speed suddenly matters, which brings us to:

**`isolatedDeclarations`** (TypeScript 5.5+) requires every exported declaration to have an explicit type annotation, so a `.d.ts` can be produced from a single file without type-checking the whole program:

```ts
// ❌ with isolatedDeclarations: the return type must be inferred from the body
export function makeUser(name: string) {
  return { name, createdAt: new Date() };
}
// Error: Function must have an explicit return type annotation with
//   --isolatedDeclarations. ts(9007)

// ✅
export function makeUser2(name: string): { name: string; createdAt: Date } {
  return { name, createdAt: new Date() };
}
```

The payoff is that non-TypeScript tools (esbuild, oxc, and friends) can generate declaration files in parallel, in milliseconds. The cost is annotation verbosity on your public API — which many people consider a feature. Adopt it for **published libraries**; skip it for application code.

**Diagnosing the compiler.** When a config argument needs settling, or a build is inexplicably slow, these five commands answer almost everything:

```bash
npx tsc --showConfig
# Prints the fully resolved config after `extends` merging. Ends every
# "but I set that option" argument instantly.

npx tsc --explainFiles | head -50
# Why is this file in my program? Traces each file to the import or `types`
# entry that pulled it in. The usual culprit for a slow build is an @types
# package you never asked for.

npx tsc --traceResolution
# Every path the resolver tried for every import. Verbose, but it is the only
# real answer to "why did it resolve to the wrong copy of that package".

npx tsc --diagnostics --noEmit
# Files, lines, memory, and time split by phase (parse / bind / check / emit).

npx tsc --generateTrace ./trace
# Chrome-trace output. Load ./trace/trace.json in chrome://tracing or
# https://ui.perfetto.dev to see which TYPE is expensive. Usually one
# recursive conditional type. See Part 9.
```

**Real Scenario — the 90-second build.** A team's CI type-check crept from 20s to 90s with no obvious cause. `tsc --explainFiles` showed 4,000 files entering the program from `@types/jest` and `@types/node` transitively — because the project predated TypeScript 6 and still relied on `types` auto-including **every** `@types` package in `node_modules`. Setting `"types": ["node"]` explicitly cut the program roughly in half. This is exactly the failure mode the TypeScript 6 default change (`types: []`) was designed to prevent, and it is a good reason to remove any inherited `"types"` line you don't understand rather than keeping it "just in case".

<a id="p3-cheatsheet"></a>

### Part 3 cheat sheet

**A config for each of the three common situations (TypeScript 7):**

```jsonc
// Node server / CLI
{ "compilerOptions": {
    "module": "nodenext", "rootDir": "./src", "outDir": "./dist",
    "types": ["node"], "declaration": true, "sourceMap": true,
    "verbatimModuleSyntax": true, "erasableSyntaxOnly": true,
    "noUncheckedIndexedAccess": true, "exactOptionalPropertyTypes": true,
    "noEmitOnError": true },
  "include": ["src"] }

// Bundled app (Vite/esbuild does the emit; tsc only checks)
{ "compilerOptions": {
    "module": "esnext", "moduleResolution": "bundler",
    "noEmit": true, "isolatedModules": true, "verbatimModuleSyntax": true,
    "noUncheckedIndexedAccess": true, "exactOptionalPropertyTypes": true,
    "paths": { "@/*": ["./src/*"] } },
  "include": ["src"] }

// Published library
{ "compilerOptions": {
    "module": "nodenext", "declaration": true, "declarationMap": true,
    "isolatedDeclarations": true, "composite": true,
    "rootDir": "./src", "outDir": "./dist" },
  "include": ["src"] }
```

| Symptom | Likely cause |
|---|---|
| `error TS5102: Option 'baseUrl' has been removed` | pre-2026 config on TypeScript 7 |
| Types resolve in the editor, `MODULE_NOT_FOUND` at runtime | `paths` not mirrored in the bundler/runtime |
| `tsc` emitted JS despite errors | `noEmitOnError` is off (the default) |
| Build got slow for no reason | stray `@types` in the program — `--explainFiles` |
| `ts(1294)` | non-erasable syntax under `erasableSyntaxOnly` |
| `ts(9007)` | missing explicit type under `isolatedDeclarations` |
| A `.d.ts` change isn't picked up | stale `tsBuildInfo` — `tsc --build --clean` |

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 4 — Narrowing](./typescript-narrowing-functions.md#part-4), where the union types from Part 2 become usable.

*Part of the [TypeScript doc set](./typescript-mastery-guide.md) in [my-learning-md-docs](../README.md). Last updated: September 6, 2026.*
