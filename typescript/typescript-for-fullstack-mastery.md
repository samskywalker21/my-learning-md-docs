# TypeScript for Full-Stack Mastery

## About This Document

- **Framing:** Curiosity-driven — no deadline or task is forcing this. Structured as a guided, hands-on progression through TypeScript's own concepts (not a JS-vs-TS history lesson — see the companion [JavaScript Mastery Guide](../javascript/javascript-for-typescript-mastery.md), which you've already covered). Theory appears only where it explains *why* a mechanism works the way it does, not as standalone history.
- **Depth tiers requested:** Beginner → Working Knowledge → Advanced → Mastery, applied per-section with judgment — collapsed where a tier would be trivial or redundant for that concept.
- **Confirmed scope — "Full-stack practical TS":** core type system, plus tooling/config, plus how TS is actually used across a typical Node backend + React frontend project. Explicitly not: exhaustive type-level-programming golf, framework internals, or a survey of every possible TS use case.
- **Prior knowledge assumed:** Solid JavaScript already (per the companion JS doc) — this document does not re-explain closures, `this`, prototypes, async/await, or ESM basics. It starts where JS stops: what changes once a *type system* sits on top.
- **Version target:** TypeScript 6.0 (current stable as of September 2026), which changed several long-standing defaults — `strict: true`, `module`/`moduleResolution` default to `esnext`/`bundler`, target defaults to `es2025` — see [§13](#13-tsconfigjson--compiler-options). TypeScript 7.0 (a from-scratch Go port, ~10× faster, no new language features) is in Release Candidate; it changes the *compiler's implementation*, not anything covered in this document, so nothing here should need revision when it ships. Anywhere a "common knowledge" practice predates TS 6.0's default flip, it's flagged explicitly, since that's exactly where outdated blog posts and Stack Overflow answers mislead.
- **Example conventions:** React examples use function components/hooks (no class components). Node/backend examples use Express plus a runtime validator (Zod), because "TypeScript types don't exist at runtime" is a recurring gotcha worth demonstrating concretely, not just stating.
- **If you ask me to update this doc later:** keep this structure, tier labels, sourcing style (official TypeScript docs + Stack Overflow inline links, GitHub/blogs only as supplementary), and the wrong-vs-right snippet format. New sections should slot into the existing four-part structure (Core Type System → Tooling & Project Setup → Practical Full-Stack Usage → Wrap-up) rather than being appended arbitrarily.

---

## Table of Contents

**Part 1 — Core Type System**
1. [Setup & Your First TS Program](#1-setup--your-first-ts-program)
2. [Type Inference & Basic Annotations](#2-type-inference--basic-annotations)
3. [Structural Typing](#3-structural-typing)
4. [Unions, Intersections & Narrowing](#4-unions-intersections--narrowing)
5. [Functions: Signatures, Overloads, Generics Basics](#5-functions-signatures-overloads-generics-basics)
6. [Interfaces vs Type Aliases](#6-interfaces-vs-type-aliases)
7. [Generics, Deep Dive](#7-generics-deep-dive)
8. [Classes & OOP Features](#8-classes--oop-features)
9. [Literal Types & Enums](#9-literal-types--enums)
10. [Discriminated Unions & Exhaustiveness Checking](#10-discriminated-unions--exhaustiveness-checking)
11. [Utility Types](#11-utility-types)
12. [Advanced Type-Level Programming](#12-advanced-type-level-programming)

**Part 2 — Tooling & Project Setup**
13. [tsconfig.json & Compiler Options](#13-tsconfigjson--compiler-options)
14. [Declaration Files & Module Augmentation](#14-declaration-files--module-augmentation)
15. [Build Tooling](#15-build-tooling)

**Part 3 — Practical Full-Stack Usage**
16. [TS on the Backend (Node)](#16-ts-on-the-backend-node)
17. [TS on the Frontend (React)](#17-ts-on-the-frontend-react)
18. [Type Safety at Boundaries](#18-type-safety-at-boundaries)

**Part 4 — Wrap-Up**
19. [Common Pitfalls Cheat Sheet](#19-common-pitfalls-cheat-sheet)
20. [Suggested Learning Order](#20-suggested-learning-order)
21. [Quick Self-Check](#21-quick-self-check)

---

## 1. Setup & Your First TS Program

TypeScript is a **structural type checker that compiles to JavaScript** — it never runs directly; every type annotation is erased before your code executes. That single fact explains most of TS's weird edges, and it's worth internalizing before anything else.

### Beginner

The fastest loop for experimenting is [tsx](https://www.npmjs.com/package/tsx) (a TS-aware Node runner) or the [TypeScript Playground](https://www.typescriptlang.org/play). For a real project:

```bash
npm install --save-dev typescript
npx tsc --init      # generates tsconfig.json
```

```ts
// hello.ts
function greet(name: string): string {
  return `Hello, ${name}`;
}

console.log(greet("world"));
// console.log(greet(42)); // compile error: Argument of type 'number' is not assignable to parameter of type 'string'
```

```bash
npx tsc hello.ts        # emits hello.js — compiles but does NOT run it
node hello.js
```

**Real Scenario — try this yourself:** Run `npx tsc hello.ts --strict` after adding `let x; x.toUpperCase();` (no annotation, no initializer). Watch it fail under `--strict` but pass without it — this is `noImplicitAny` catching an `x: any` that would silently crash at runtime. This is the smallest possible demonstration of what "type safety" buys you.

[⤴ back to top](#table-of-contents)

---

## 2. Type Inference & Basic Annotations

### Beginner

TypeScript infers types from initializers — you rarely need to annotate variables that are assigned immediately.

```ts
let count = 5;        // inferred: number
const name = "Sam";    // inferred: "Sam" (a literal type, since const can't be reassigned)
let list = [1, 2, 3];  // inferred: number[]
```

Annotate when there's no initializer, or when the inferred type would be wrong for how you intend to use the value:

```ts
let userId: number;   // must annotate — no initializer to infer from
userId = 42;
```

### Working Knowledge

Function **parameters** are never inferred from usage — only from annotations or contextual typing — so always annotate them. Return types *are* inferred from the function body, but annotating them explicitly is good practice for public APIs (it turns a body-implementation mistake into a caught error at the signature, not a silently-changed contract).

```ts
// Wrong — parameter silently becomes `any`, defeating type-checking for this function entirely
function double(x) {
  return x * 2;
}

// Right
function double(x: number): number {
  return x * 2;
}
```

**Real Scenario:** In a `tsconfig.json` without `noImplicitAny`, the `double(x)` version above compiles with zero errors, and `double("5")` returns `"55"` (string repeat via coercion) instead of a type error — the entire point of using TS quietly evaporates for that one function. `noImplicitAny` is on by default in `strict` mode since TS 6.0; older configs (or `strict: false`) can still hide this. See [§13](#13-tsconfigjson--compiler-options).

### Advanced

`const` narrows literals; `let` widens them. This surprises people moving from "TS infers types" to "wait, why did that widen":

```ts
let a = "hello";        // type: string (widened — could be reassigned)
const b = "hello";      // type: "hello" (narrowed — can never change)

function f(x: "a" | "b") { /* ... */ }
let mode = "a";
f(mode);                // Error: 'mode' is of type 'string', not '"a" | "b"'
f("a");                 // OK — literal passed directly
f(mode as const);        // OK — see `as const` below
```

The `as const` assertion locks a value (and everything nested in it) to its most specific literal type, recursively — critical for config objects and discriminated unions:

```ts
const config = { mode: "dark", retries: 3 };
// type: { mode: string; retries: number }

const configConst = { mode: "dark", retries: 3 } as const;
// type: { readonly mode: "dark"; readonly retries: 3 }
```

[⤴ back to top](#table-of-contents)

---

## 3. Structural Typing

This is the single biggest conceptual shift from most other typed languages (Java, C#), and it underlies almost everything else in this document.

### Working Knowledge

TypeScript's type compatibility is **structural** ("duck typing"), not **nominal**. Two types are compatible if their *shapes* match — regardless of name, declared relationship, or intent.

```ts
interface Point {
  x: number;
  y: number;
}

class Vector {
  constructor(public x: number, public y: number) {}
}

function logPoint(p: Point) {
  console.log(`${p.x}, ${p.y}`);
}

logPoint(new Vector(1, 2));       // OK — Vector has the right shape, no `implements Point` needed
logPoint({ x: 1, y: 2, z: 3 });    // OK for a *variable* — but see excess property checks below
```

### Advanced

**Excess property checks** are a special-cased exception to pure structural typing, applied only to **object literals assigned directly**:

```ts
// Wrong — this looks like it should be fine structurally, and IS fine via a variable...
const p = { x: 1, y: 2, z: 3 };
logPoint(p); // OK — `p`'s type is a supertype-compatible object, checked structurally

// ...but a literal passed directly triggers excess property checking:
logPoint({ x: 1, y: 2, z: 3 });
// Error: Object literal may only specify known properties, and 'z' does not exist in type 'Point'
```

**Real Scenario:** This inconsistency trips people constantly — "it worked when I stored it in a variable but not inline" is one of the most-asked TS questions on Stack Overflow. The rule exists specifically to catch typos in object literals (e.g. `{ colour: "red" }` instead of `{ color: "red" }`, which would otherwise silently pass as an "excess property" and the real bug — a missing `color` — would go unnoticed). See the [TS Handbook on excess property checks](https://www.typescriptlang.org/docs/handbook/2/objects.html#excess-property-checks) and the related [Stack Overflow discussion](https://stackoverflow.com/questions/40295559/how-to-suppress-error-ts2339-property-does-not-exist-on-value-of-type-when).

### Mastery

Structural typing means **empty interfaces (or `{}`) match almost anything**, including primitives — a frequent source of bugs when people reach for `{}` meaning "any object" (it actually means "anything non-`null`/`undefined`"). Use `Record<string, unknown>` or `object` for "some kind of object," and a specific shape wherever possible.

[⤴ back to top](#table-of-contents)

---

## 4. Unions, Intersections & Narrowing

### Beginner

A **union** (`|`) means "one of these types." An **intersection** (`&`) means "all of these types combined."

```ts
type Status = "pending" | "active" | "closed";
type Loud = { volume: number };
type Colored = { color: string };
type LoudAndColored = Loud & Colored; // { volume: number; color: string }
```

### Working Knowledge

You can only access members common to **every** type in a union until you **narrow** it. TypeScript's control-flow analysis narrows automatically based on runtime checks.

```ts
function formatId(id: string | number) {
  // id.toUpperCase(); // Error — number has no toUpperCase

  if (typeof id === "string") {
    return id.toUpperCase();   // narrowed to `string` here
  }
  return id.toFixed(2);         // narrowed to `number` here
}
```

Common narrowing techniques: `typeof`, `instanceof`, `in`, truthiness checks, and equality checks (`===` against a literal).

```ts
// Wrong — checking a property that might not exist on every union member throws at runtime
function getLength(x: string | string[]) {
  return x.length; // actually fine here since both have .length — but the pattern below is the trap:
}

function process(x: { kind: "a"; data: string } | { kind: "b"; count: number }) {
  // if ("data" in x) works, but easy to typo or forget — see discriminated unions (§10) for the robust version
  if ("data" in x) {
    console.log(x.data.toUpperCase());
  }
}
```

### Advanced

Custom **type guards** (functions returning `x is T`) let you encapsulate narrowing logic and reuse it across the codebase:

```ts
interface Cat { meow(): void }
interface Dog { bark(): void }

function isCat(pet: Cat | Dog): pet is Cat {
  return (pet as Cat).meow !== undefined;
}

function speak(pet: Cat | Dog) {
  if (isCat(pet)) {
    pet.meow();   // narrowed via the custom guard
  } else {
    pet.bark();
  }
}
```

As of TS 5.5+, in many cases you don't even need `is T` explicitly — TypeScript can **infer type predicates** automatically from `Array.prototype.filter` and similar patterns:

```ts
const mixed: (string | undefined)[] = ["a", undefined, "b"];
const strings = mixed.filter((x) => x !== undefined); // inferred as string[], no `as` needed
```

### Mastery

Narrowing is powered by TypeScript's **Control Flow Analysis (CFA)** — it tracks the *reachable narrowed type* at every point in the code, not just at declaration. This is why reassigning inside an `if` block, or narrowing via a closure captured before a mutation, can behave unexpectedly: CFA can lose track across function boundaries (a callback isn't guaranteed to run synchronously, so TS can't assume a narrowing done outside it still holds inside it).

```ts
function process(x: string | undefined) {
  if (x === undefined) return;
  setTimeout(() => {
    x.toUpperCase(); // Error in strict mode — TS can't prove `x` is still defined when this callback runs
  }, 1000);
}
```

[⤴ back to top](#table-of-contents)

---

## 5. Functions: Signatures, Overloads, Generics Basics

### Working Knowledge

Optional (`?`) and default parameters, rest parameters, and function types:

```ts
function greet(name: string, greeting: string = "Hello", ...rest: string[]): string {
  return `${greeting}, ${name}${rest.length ? " " + rest.join(" ") : ""}`;
}

type Comparator<T> = (a: T, b: T) => number;
const byLength: Comparator<string> = (a, b) => a.length - b.length;
```

### Advanced

**Overloads** let a single function have multiple valid call signatures — useful when parameter *count/type* determines the *return type* in a way a union can't express cleanly:

```ts
function makeDate(timestamp: number): Date;
function makeDate(year: number, month: number, day: number): Date;
function makeDate(yearOrTimestamp: number, month?: number, day?: number): Date {
  if (month !== undefined && day !== undefined) {
    return new Date(yearOrTimestamp, month, day);
  }
  return new Date(yearOrTimestamp);
}

makeDate(2026, 8, 3); // OK — matches the 3-arg overload
// makeDate(2026, 8);  // Error — no overload takes exactly 2 args
```

Only the overload signatures are visible to callers; the final signature (the "implementation signature") is not part of the public API and must be general enough to handle every overload body.

### Mastery

Generic function signatures (basics — deep dive in [§7](#7-generics-deep-dive)):

```ts
function firstElement<T>(arr: T[]): T | undefined {
  return arr[0];
}

firstElement([1, 2, 3]);      // T inferred as number
firstElement(["a", "b"]);      // T inferred as string
```

[⤴ back to top](#table-of-contents)

---

## 6. Interfaces vs Type Aliases

### Working Knowledge

They overlap heavily. The practical differences, per the [official TS Handbook](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#differences-between-type-aliases-and-interfaces):

| | `interface` | `type` |
|---|---|---|
| Object shapes | ✅ | ✅ |
| Unions/intersections/primitives | ❌ | ✅ |
| Declaration merging (re-opening to add members) | ✅ | ❌ |
| Extending another shape | `extends` (multiple) | `&` intersection |

```ts
interface User {
  id: number;
  name: string;
}
interface User {
  email: string;      // merges into the same User type — intentional or a bug, depending on context
}
// User is now { id, name, email }

type Id = string | number;   // `type` required — interfaces can't alias a union
```

### Advanced

The official guidance (since the TS 5.x handbook) is: **use `interface` until you need a feature only `type` has** (unions, mapped types, conditional types). Declaration merging can be a footgun in application code (accidental re-declaration silently adds fields) but is exactly the mechanism library authors rely on for augmenting third-party types — see [§14](#14-declaration-files--module-augmentation).

```ts
// Wrong — accidentally re-declaring an interface name in application code merges instead of erroring
interface Config {
  apiUrl: string;
}
// ...500 lines later, in a different file, someone reuses the name by mistake
interface Config {
  timeout: number;
}
// No error! Config is now { apiUrl, timeout } — was this intentional?

// Right — use `type` for application-level shapes where accidental merging would be a silent bug
type Config = {
  apiUrl: string;
  timeout: number;
};
```

[⤴ back to top](#table-of-contents)

---

## 7. Generics, Deep Dive

### Working Knowledge

Generics parametrize a type over another type, preserving the relationship between input and output that `any` or `unknown` would destroy.

```ts
// Wrong — using `any` loses all information; callers get no safety back
function identity(x: any): any {
  return x;
}
const result = identity(5); // type: any — TS has no idea it's still a number

// Right — generics preserve the relationship
function identity<T>(x: T): T {
  return x;
}
const result2 = identity(5); // type: number
```

### Advanced

**Constraints** (`extends`) restrict what a type parameter can be, letting you safely access members within the generic function:

```ts
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, name: "Sam" };
getProperty(user, "name");   // OK, inferred as string
// getProperty(user, "age");  // Error — "age" is not a key of `user`
```

**Default type parameters** work like default function parameters:

```ts
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
}
```

### Mastery

Generics are **erased at compile time** just like every other type — there is no runtime reflection on `T`. This means you cannot do `new T()`, `typeof T`, or branch on `T` at runtime; if you need runtime type information, you must pass it explicitly (a class constructor, a discriminant string, a Zod schema — see [§18](#18-type-safety-at-boundaries)).

```ts
// This is impossible and will not compile:
function create<T>(): T {
  return new T(); // Error — 'T' only refers to a type, but is being used as a value here
}
```

**Variance** matters once you have generics over function types: TypeScript checks function parameters *bivariantly* for method syntax but contravariantly-ish in practice for most real-world code under `strictFunctionTypes` (function-typed properties are checked contravariantly on parameters; method-shorthand properties are not, for backward-compatibility reasons documented in the [TS 2.6 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-6.html#strict-function-types)). This is a genuinely advanced/rarely-needed corner — know it exists so a "why did this compile when I expected an error" moment isn't a mystery.

[⤴ back to top](#table-of-contents)

---

## 8. Classes & OOP Features

### Working Knowledge

TypeScript adds access modifiers, `readonly`, abstract classes, and parameter properties (a shorthand for declaring + assigning a field from a constructor argument) on top of JS classes.

```ts
class Account {
  readonly id: string;
  private balance: number;

  constructor(id: string, initialBalance: number) {
    this.id = id;
    this.balance = initialBalance;
  }

  deposit(amount: number): void {
    this.balance += amount;
  }
}

// Parameter properties — shorthand for the same class above
class AccountShort {
  constructor(
    public readonly id: string,
    private balance: number,
  ) {}
}
```

`private`/`protected` are **compile-time only** — erased at runtime, unlike JS's native `#private` fields:

```ts
class Wallet {
  private secret = "shh";
}
const w = new Wallet();
// w.secret; // TS compile error
console.log(JSON.stringify(w)); // {"secret":"shh"} — it's fully visible at runtime!

class WalletTrue {
  #secret = "shh"; // native JS private field — enforced at runtime too
}
```

**Real Scenario:** A team relies on `private` to "hide" a field from serialization, then ships an API that `JSON.stringify()`s a class instance directly — the "private" field appears in the response body, because `private` was never a runtime guarantee. Use native `#` fields when you need actual runtime encapsulation, or don't put sensitive data on classes that get serialized directly.

### Advanced

`abstract` classes can't be instantiated directly and can declare members without implementations, enforced at compile time:

```ts
abstract class Shape {
  abstract area(): number;
  describe(): string {
    return `Area: ${this.area()}`;
  }
}

class Circle extends Shape {
  constructor(private radius: number) { super(); }
  area(): number { return Math.PI * this.radius ** 2; }
}

// new Shape(); // Error — cannot create an instance of an abstract class
```

### Mastery

**Decorators** reached TC39 Stage 3 and are supported natively in TS 5.0+ **without** the `experimentalDecorators` flag. This is a real breaking difference from the legacy decorators still used by Angular and (older) NestJS: standard (Stage 3) decorators have a different runtime signature, no built-in `reflect-metadata` support, and don't need `experimentalDecorators`/`emitDecoratorMetadata` at all. If you're following an older tutorial or working in a codebase with `experimentalDecorators: true`, you're in the legacy system — check which one you're in before copying decorator patterns between projects ([TS decorators handbook](https://www.typescriptlang.org/docs/handbook/decorators.html); the page itself now flags that it documents the older experimental system).

[⤴ back to top](#table-of-contents)

---

## 9. Literal Types & Enums

### Working Knowledge

Literal types let a specific value (not just its general type) *be* the type:

```ts
type Direction = "up" | "down" | "left" | "right";
function move(dir: Direction) { /* ... */ }
move("up");     // OK
// move("north"); // Error
```

`enum` provides a named, runtime-existing set of constants:

```ts
enum Direction2 {
  Up,
  Down,
  Left,
  Right,
}
move2(Direction2.Up);
```

### Advanced

Modern official guidance leans away from `enum` toward `as const` object literals or plain string-literal unions, for several concrete reasons:

1. Regular `enum` generates real runtime JS code (an object plus reverse mappings for numeric enums) — it's not erasable syntax.
2. Since Node.js added [native TypeScript execution via type stripping](https://nodejs.org/api/typescript.html) (Node 22+, unflagged since Node 23), files can be run directly *only* if they contain purely erasable syntax — `enum` does not qualify, so an enum-using file can't be run with Node's built-in TS support without a separate build step.
3. TS 5.8 added the [`erasableSyntaxOnly` compiler flag](https://www.totaltypescript.com/erasable-syntax-only) specifically to catch this: it rejects `enum`, class parameter properties, and `namespace` at compile time.

```ts
// Wrong (in a project targeting Node's native TS execution, or using erasableSyntaxOnly)
enum Status {
  Pending,
  Active,
  Closed,
}

// Right — same ergonomics, purely erasable
const Status = {
  Pending: "pending",
  Active: "active",
  Closed: "closed",
} as const;
type Status = (typeof Status)[keyof typeof Status]; // "pending" | "active" | "closed"
```

`const enum` is a separate, narrower case (fully inlined at compile time, no runtime object at all) — but it has its own portability problems across isolated-modules bundlers and is disallowed by `erasableSyntaxOnly` as well; see the [open GitHub discussion on const enums under erasableSyntaxOnly](https://github.com/microsoft/TypeScript/issues/61490).

**When `enum` is still fine:** small, internal-only codebases with a full build step (not relying on Node's native TS execution or an isolated-modules bundler), where the numeric-enum ergonomics are genuinely wanted.

[⤴ back to top](#table-of-contents)

---

## 10. Discriminated Unions & Exhaustiveness Checking

This is one of TypeScript's most practically powerful patterns for full-stack code — API response shapes, Redux-style actions, form/UI state — and it directly builds on [§4](#4-unions-intersections--narrowing).

### Working Knowledge

A **discriminated union** is a union of object types sharing a common literal-typed field (the "discriminant" or "tag"). Narrowing on that field narrows the whole object:

```ts
type ApiResult =
  | { status: "success"; data: string }
  | { status: "error"; message: string }
  | { status: "loading" };

function render(result: ApiResult) {
  switch (result.status) {
    case "success":
      return result.data;       // narrowed: only `data` exists here
    case "error":
      return result.message;     // narrowed: only `message` exists here
    case "loading":
      return "Loading...";
  }
}
```

### Advanced

Combine with a `never`-typed exhaustiveness check so that adding a new union member *without* updating every `switch` becomes a **compile error**, not a silent runtime gap:

```ts
function assertNever(x: never): never {
  throw new Error(`Unhandled case: ${JSON.stringify(x)}`);
}

function render(result: ApiResult) {
  switch (result.status) {
    case "success": return result.data;
    case "error": return result.message;
    case "loading": return "Loading...";
    default: return assertNever(result); // if a new variant is added above without a new case, this line fails to compile
  }
}
```

**Real Scenario:** A teammate adds a fourth `ApiResult` variant (`{ status: "cancelled" }`) for a new feature but forgets to update this `render` function. Without `assertNever`, the switch silently falls through and returns `undefined` in production. With it, `tsc` fails the build the moment the new variant is added, anywhere it isn't handled — turning a runtime gap into a compile-time to-do list. This pattern is discussed extensively in the [TS Handbook's narrowing section](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#the-never-type) and is a frequent Stack Overflow recommendation for "how do I make switch statements exhaustive in TypeScript."

[⤴ back to top](#table-of-contents)

---

## 11. Utility Types

### Working Knowledge

Built-in generic types that transform other types — covered in the [official Utility Types reference](https://www.typescriptlang.org/docs/handbook/utility-types.html):

```ts
interface User {
  id: number;
  name: string;
  email: string;
}

type PartialUser = Partial<User>;          // all fields optional — great for PATCH/update payloads
type UserPreview = Pick<User, "id" | "name">;  // subset of fields
type UserNoEmail = Omit<User, "email">;      // all fields except one
type ReadonlyUser = Readonly<User>;          // no field reassignable
type UserMap = Record<string, User>;          // { [key: string]: User }
type RequiredUser = Required<PartialUser>;    // inverse of Partial
```

```ts
// Wrong — hand-rolling what Pick already does invites the two definitions drifting apart
type UserPreviewManual = { id: number; name: string };

// Right — derived from the source of truth, stays in sync automatically
type UserPreview = Pick<User, "id" | "name">;
```

### Advanced

`ReturnType<T>`, `Parameters<T>`, and `Awaited<T>` extract types *from* functions/promises rather than transforming object shapes — useful for deriving a type from an existing function instead of duplicating its signature by hand:

```ts
async function fetchUser(id: number) {
  return { id, name: "Sam" };
}

type FetchUserReturn = Awaited<ReturnType<typeof fetchUser>>;
// { id: number; name: string } — derived, not hand-written
```

[⤴ back to top](#table-of-contents)

---

## 12. Advanced Type-Level Programming

This section is genuinely Mastery-tier throughout — useful to recognize when reading library type definitions, less commonly *written* in day-to-day application code.

### Mastery

**Mapped types** transform every property of an existing type:

```ts
type Optional<T> = { [K in keyof T]?: T[K] };
type Nullable<T> = { [K in keyof T]: T[K] | null };
```

**Conditional types** branch on a type-level condition, and `infer` extracts a type from within another type — this is how `ReturnType` and `Awaited` are actually implemented:

```ts
type IsString<T> = T extends string ? true : false;
type A = IsString<"hi">;  // true
type B = IsString<42>;    // false

type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
```

**Template literal types** build string-literal unions programmatically, useful for typed event names, CSS-in-JS, or route paths:

```ts
type EventName = "click" | "focus" | "blur";
type HandlerName = `on${Capitalize<EventName>}`;
// "onClick" | "onFocus" | "onBlur"
```

**Recursive types** allow expressing arbitrarily nested structures (JSON being the canonical example):

```ts
type Json = string | number | boolean | null | Json[] | { [key: string]: Json };
```

**Real Scenario — try this yourself:** Paste the `EventName`/`HandlerName` example into the [TS Playground](https://www.typescriptlang.org/play) and hover over `HandlerName` to watch TypeScript compute the union eagerly at the type level. Then try adding a 4th event name to `EventName` and watch `HandlerName` update automatically — this is "compile-time metaprogramming" made concrete.

**When to reach for this vs. when not to:** if you find yourself writing conditional/mapped types more than occasionally in *application* code, it's often a sign the underlying data model would be simpler as a discriminated union ([§10](#10-discriminated-unions--exhaustiveness-checking)) or several concrete types instead of one clever generic one. This machinery earns its keep most clearly in library/framework code (which is exactly where you'll encounter it when reading `.d.ts` files — see [§14](#14-declaration-files--module-augmentation)).

[⤴ back to top](#table-of-contents)

---

## 13. tsconfig.json & Compiler Options

For the full deep-dive on `tsconfig.json` — compiler options, module resolution and path aliases, project references/monorepo composite builds, framework-specific configs (Next, Vite, Nuxt, Bun), publishing a library, and editor/lint integration — see [typescript-config-mastery.md](typescript-config-mastery.md).

[⤴ back to top](#table-of-contents)

---

## 14. Declaration Files & Module Augmentation

### Working Knowledge

`.d.ts` files contain **type information only** — no runtime code — and are how TypeScript understands the shape of plain-JS libraries. Most popular packages either ship their own (`"types"` field in `package.json`) or have community-maintained ones on [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped), installed as `@types/<package>`:

```bash
npm install --save-dev @types/node
```

### Advanced

Writing your own `.d.ts` for an untyped JS module:

```ts
// legacy-lib.d.ts
declare module "legacy-lib" {
  export function doThing(x: string): number;
}
```

**Module augmentation** extends an *existing* module's types — the standard way to, for example, add a custom property to Express's `Request` object:

```ts
// express-augment.d.ts
import "express";

declare module "express" {
  interface Request {
    userId?: string;   // now every Express Request has this field, project-wide
  }
}
```

### Mastery

Augmentation relies on declaration merging ([§6](#6-interfaces-vs-type-aliases)) applied across module boundaries — TypeScript merges your re-opened `interface Request` with the library's own declaration because both share the exact same module specifier and interface name. Get the module specifier or interface name slightly wrong and the augmentation silently does nothing (no error — it just declares a *new*, unused, unrelated interface), which is a genuinely common and hard-to-spot mistake; double-check by hovering the augmented property in your editor on a real `Request` object.

[⤴ back to top](#table-of-contents)

---

## 15. Build Tooling

### Working Knowledge

`tsc` type-checks *and* can emit JS, but in most real full-stack projects today it's used for type-checking only, while a faster tool does the actual transpilation:

| Tool | Role | Typical use |
|---|---|---|
| `tsc` | Type-check (and optionally emit) | CI type-check step; library `.d.ts` generation |
| [tsx](https://www.npmjs.com/package/tsx) | Run `.ts` directly in dev, no build step | Node backend dev server |
| [esbuild](https://esbuild.github.io/) | Fast transpile (strips types, no type-checking) | Bundling, powers Vite's dev server |
| [Vite](https://vite.dev/) | Dev server + bundler | Frontend apps (React, etc.) |
| [tsup](https://tsup.egoist.dev/) | Bundles a library for npm publishing | Packages, not apps |

**Real Scenario:** esbuild/SWC-based tools (Vite, tsx) **strip types without checking them** — they transpile file-by-file for speed and never look at your whole project's types. This means `npm run dev` can succeed and the app can run in the browser even with real type errors present, because the dev-time bundler never checked types at all. Always run `tsc --noEmit` as a separate CI step (or your editor's live type-checking) — don't rely on the dev server catching type errors.

### Advanced

Node.js can now run `.ts` files **directly**, without any bundler, via built-in type stripping (unflagged since Node 23 — see [official Node docs](https://nodejs.org/api/typescript.html)):

```bash
node server.ts   # works natively on Node 23+, no ts-node/tsx needed for erasable syntax
```

The tradeoff: this only works for **erasable syntax**. Files using `enum`, `namespace`, or class parameter properties will error — see [§9](#9-literal-types--enums). Set `"erasableSyntaxOnly": true` in `tsconfig.json` to catch violations at type-check time rather than discovering them at `node` runtime.

[⤴ back to top](#table-of-contents)

---

## 16. TS on the Backend (Node)

### Working Knowledge

Typing an Express route handler:

```ts
import express, { Request, Response } from "express";

const app = express();
app.use(express.json());

interface CreateUserBody {
  name: string;
  email: string;
}

app.post("/users", (req: Request<{}, {}, CreateUserBody>, res: Response) => {
  const { name, email } = req.body; // typed as CreateUserBody
  res.status(201).json({ name, email });
});
```

### Advanced

**The critical gotcha:** `req.body: CreateUserBody` is a *compile-time claim*, not a runtime guarantee. TypeScript trusts the annotation completely — it does not, and cannot, validate the actual JSON a client sends. A malicious or buggy client can send `{}`, and `req.body.name` will be `undefined` at runtime despite TS's type system insisting it's a `string`.

```ts
// Wrong — TS types give a false sense of runtime safety
app.post("/users", (req: Request<{}, {}, CreateUserBody>, res: Response) => {
  const upperName = req.body.name.toUpperCase(); // crashes if a client omits `name` — TS won't catch this
});

// Right — validate at the boundary, let TS infer the type FROM the validator
import { z } from "zod";

const CreateUserSchema = z.object({
  name: z.string(),
  email: z.string().email(),
});

app.post("/users", (req: Request, res: Response) => {
  const parsed = CreateUserSchema.safeParse(req.body);
  if (!parsed.success) {
    return res.status(400).json({ error: parsed.error.flatten() });
  }
  const { name, email } = parsed.data; // now genuinely guaranteed at runtime, not just by annotation
  res.status(201).json({ name, email });
});
```

This is expanded on generally in [§18](#18-type-safety-at-boundaries) — it's the single most consequential gotcha for anyone moving from "TS in isolated scripts" to "TS in a real backend service."

[⤴ back to top](#table-of-contents)

---

## 17. TS on the Frontend (React)

### Working Knowledge

Typing props, state, and events in a function component:

```tsx
interface ButtonProps {
  label: string;
  onClick: () => void;
  variant?: "primary" | "secondary";
}

function Button({ label, onClick, variant = "primary" }: ButtonProps) {
  return <button className={variant} onClick={onClick}>{label}</button>;
}

function Counter() {
  const [count, setCount] = useState<number>(0); // annotation often unneeded — inferred from `0`
  return <Button label={`Count: ${count}`} onClick={() => setCount((c) => c + 1)} />;
}
```

### Advanced

Typing DOM events precisely (a frequent source of `any`-typed event handlers copy-pasted from Stack Overflow answers that predate strict-mode defaults):

```tsx
// Wrong — implicit `any` on the event parameter under strict mode won't even compile,
// and without strict mode, it silently loses all autocomplete/safety on `e`
function Input({ onChange }: { onChange: (value: string) => void }) {
  return <input onChange={(e) => onChange(e.target.value)} />; // fine here due to contextual typing...
}

// ...but breaks the moment the handler is extracted to a standalone function without an annotation:
function handleChange(e) {           // Error under strict: Parameter 'e' implicitly has an 'any' type
  console.log(e.target.value);
}

// Right — annotate with React's specific event type
function handleChangeTyped(e: React.ChangeEvent<HTMLInputElement>) {
  console.log(e.target.value);
}
```

**Real Scenario:** Inline handlers (`onChange={(e) => ...}`) get their event type for free via **contextual typing** — TS infers `e`'s type from the JSX attribute's expected signature. The instant that same logic is pulled out into a named function for reuse (a very common refactor), the contextual typing is lost and the parameter needs an explicit `React.ChangeEvent<...>` annotation, or it silently becomes `any` on non-strict configs. This is one of the most common "why did TS stop complaining/why did it start complaining" moments when refactoring React components.

[⤴ back to top](#table-of-contents)

---

## 18. Type Safety at Boundaries

### Working Knowledge

TypeScript's types are **fully erased at compile time** — nothing about them exists once your code is running. Any place data crosses a boundary TypeScript didn't generate — an HTTP request/response, `JSON.parse`, `localStorage`, a message queue payload, a third-party webhook — is a place where TS's compile-time guarantee and runtime reality can diverge, and TS gives you no warning about it by default.

```ts
// Wrong — TS trusts this completely, with zero runtime basis for the trust
interface Config {
  retries: number;
}
const config: Config = JSON.parse(localStorage.getItem("config") ?? "{}");
config.retries + 1; // NaN at runtime if localStorage had stale/malformed data — TS had no way to know
```

### Advanced

`unknown` is the type-safe alternative to `any` for exactly this situation — it forces you to narrow (via a validator, a type guard, or explicit checks) before using the value, instead of silently trusting it:

```ts
function parseConfig(raw: string): Config {
  const data: unknown = JSON.parse(raw); // `unknown`, not `any` — forces validation below
  if (
    typeof data === "object" &&
    data !== null &&
    "retries" in data &&
    typeof (data as any).retries === "number"
  ) {
    return data as Config;
  }
  throw new Error("Invalid config");
}
```

In practice, hand-rolled runtime checks like the above don't scale past a couple of fields — a **runtime schema validator** (Zod, Valibot, io-ts) that both validates *and* derives the static type from a single source of truth is the standard modern approach:

```ts
import { z } from "zod";

const ConfigSchema = z.object({ retries: z.number() });
type Config = z.infer<typeof ConfigSchema>; // static type derived from the runtime schema — one source of truth

function parseConfig(raw: string): Config {
  return ConfigSchema.parse(JSON.parse(raw)); // throws with a clear error if shape doesn't match
}
```

### Mastery

This is the practical resolution of a tension that runs through the entire language: **TypeScript is a compile-time-only tool bolted onto a dynamically-typed runtime.** Every "TS caught this bug" story is really "TS caught this bug *at points where the data's actual shape matched what was declared*." The discipline that separates well-typed real-world codebases from ones that merely *look* well-typed is treating every external boundary — not just obviously "external" ones like HTTP, but `JSON.parse`, `Object.keys` results typed loosely, third-party callback payloads — as `unknown` until proven otherwise, and never `as`-asserting past a boundary without validating first.

[⤴ back to top](#table-of-contents)

---

## 19. Common Pitfalls Cheat Sheet

| Pitfall | Why it happens | Fix |
|---|---|---|
| `any` silently everywhere | Missing `noImplicitAny`/`strict`, or explicit `any` used to "make an error go away" | Enable `strict`; use `unknown` + narrowing instead of `any` |
| Object literal excess-property error only sometimes | Excess property checks only apply to literals assigned directly, not to variables ([§3](#3-structural-typing)) | Understand it's a literal-specific lint, not full structural rejection |
| `private` field visible in `JSON.stringify` output | `private` is compile-time only, not runtime ([§8](#8-classes--oop-features)) | Use native `#field` for real runtime privacy |
| `enum` breaks Node's native TS execution / `erasableSyntaxOnly` | Enums require runtime code generation, not just erasure ([§9](#9-literal-types--enums)) | Use `as const` object + derived union type instead |
| Switch statement silently misses a new union case | No exhaustiveness check | Add a `default: assertNever(x)` branch ([§10](#10-discriminated-unions--exhaustiveness-checking)) |
| `req.body` typed but wrong shape crashes in prod | TS types aren't runtime validation ([§16](#16-ts-on-the-backend-node), [§18](#18-type-safety-at-boundaries)) | Validate with Zod/similar at every external boundary |
| Extracted event handler suddenly needs annotation | Lost contextual typing when pulled out of JSX ([§17](#17-ts-on-the-frontend-react)) | Annotate with the specific `React.*Event<...>` type |
| `.ts` relative import needs `.js` extension | `moduleResolution: nodenext` models real Node ESM resolution ([§13](#13-tsconfigjson--compiler-options)) | Use `.js` extensions for Node targets; `bundler` resolution if a bundler owns compilation |
| Type errors don't show up until CI, not in dev server | esbuild/SWC-based dev tools strip types without checking them ([§15](#15-build-tooling)) | Run `tsc --noEmit` as a separate check, don't trust the dev server |

[⤴ back to top](#table-of-contents)

---

## 20. Suggested Learning Order

1. [§1–2](#1-setup--your-first-ts-program) — get a working loop, understand inference vs. annotation.
2. [§3–4](#3-structural-typing) — structural typing and narrowing are the conceptual foundation everything else sits on; don't rush these.
3. [§5–6](#5-functions-signatures-overloads-generics-basics) — functions, interfaces vs types — enough to write everyday application code.
4. [§9–10](#9-literal-types--enums) — literal types and discriminated unions — the highest practical payoff pattern in the whole document.
5. [§13](#13-tsconfigjson--compiler-options) — set up a real project config once you're writing more than snippets.
6. [§7, §11–12](#7-generics-deep-dive) — generics and utility types, as you start noticing repetition in your own type definitions.
7. [§16–18](#16-ts-on-the-backend-node) — apply everything to a real full-stack slice (one Express route + one React form is enough to feel every gotcha in this document firsthand).
8. [§8, §14, §15](#8-classes--oop-features) — classes/decorators, declaration files, build tooling — pick up as needed, not necessarily in order.

[⤴ back to top](#table-of-contents)

---

## 21. Quick Self-Check

1. Why does `logPoint({ x: 1, y: 2, z: 3 })` error but `logPoint(p)` (where `p` has the same shape) doesn't?
2. What's the difference between `private` in TS and `#field` in native JS — and why does it matter for a class instance that gets `JSON.stringify`'d?
3. Why can't you write `function create<T>() { return new T(); }`?
4. What does `as const` change about `{ mode: "dark" }`'s inferred type, and why does that matter for discriminated unions?
5. Why might `req.body: CreateUserBody` be lying to you at runtime, and what's the fix?
6. Why is `enum` discouraged in projects targeting Node's native TypeScript execution or using `erasableSyntaxOnly`? What's the modern replacement?
7. What's the practical difference between `moduleResolution: "bundler"` and `"nodenext"`, and which one needs `.js` extensions on relative imports?
8. Why can a Vite/esbuild dev server run your app successfully even when it has real type errors?
9. What does an `assertNever` branch in a `switch` actually protect against, concretely?
10. Why is `unknown` safer than `any` for data crossing an external boundary (an API response, `JSON.parse`, `localStorage`)?

[⤴ back to top](#table-of-contents)
