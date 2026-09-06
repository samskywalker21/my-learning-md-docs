# TypeScript — Generics (Part 8)

> **About this document:** this is one of the sub-docs in this repo's TypeScript set. The canonical spec — framing, scope, tier policy, sourcing rule, versions, and how to update it — lives once in [`typescript-mastery-guide.md`](./typescript-mastery-guide.md#about-this-document). Written against **TypeScript 7.0**, September 6, 2026.

Generics are where TypeScript stops being annotation and starts being computation: a type that is a *function of* another type. This Part is deliberately its own file because inference — how the checker decides what `T` is — is the highest-leverage thing in the language and deserves room.

---

## Table of Contents

- [Part 8 — Generics](#part-8)
  - [Beginner — a type as a parameter](#p8-beginner)
  - [Working Knowledge — constraints, `keyof`, defaults](#p8-working)
  - [Advanced — inference sites, and why your generic infers `unknown`](#p8-advanced)
  - [Mastery — `const` type parameters, variance annotations, generic design rules](#p8-mastery)
  - [Part 8 cheat sheet](#p8-cheatsheet)

---

<a id="part-8"></a>

## Part 8 — Generics

<a id="p8-beginner"></a>

### Beginner — a type as a parameter

A generic is a type with a hole in it. You fill the hole at the point of use.

```ts
// ❌ Without generics you lose information at the boundary
function firstAny(arr: any[]): any { return arr[0] }
const n = firstAny([1, 2, 3]);    // any — the type is gone, and so is your safety

// ✅ With a type parameter the relationship survives
function first<T>(arr: T[]): T | undefined { return arr[0] }
const n2 = first([1, 2, 3]);      // number | undefined
const s2 = first(["a", "b"]);     // string | undefined
```

`T` is a *type variable*: whatever type flows in, that's what flows out. You almost never write `first<number>([1,2,3])` — the checker **infers** `T` from the argument. Explicit type arguments are for the cases where it can't.

Generic types work the same way:

```ts
type Box<T> = { value: T };
interface Repo<T> {
  get(id: string): Promise<T | null>;
  save(entity: T): Promise<void>;
}

class Stack<T> {
  #items: T[] = [];
  push(item: T) { this.#items.push(item) }
  pop(): T | undefined { return this.#items.pop() }
}
```

**Naming:** `T` is conventional for one parameter; use real names once there's more than one (`Repo<Entity, Id>` beats `Repo<T, U>`). `K` for keys, `V` for values, `E` for errors, `R` for return types are the recognised abbreviations.

<a id="p8-working"></a>

### Working Knowledge — constraints, `keyof`, defaults

An unconstrained `T` can be anything, so you can do almost nothing with it. **Constraints** (`extends`) buy you capabilities:

```ts
// ❌ T could be a number; numbers have no .length
function longest<T>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
  // ts(2339): Property 'length' does not exist on type 'T'.
}

// ✅ constrain to "anything with a numeric length"
function longest2<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}

longest2("alpha", "beta");        // string
longest2([1, 2], [1, 2, 3]);      // number[]
longest2(1, 2);                   // ❌ ts(2345): number has no 'length'
```

Read `T extends C` as **"T is at least C"** — a lower bound on capability, an upper bound on the type. It does *not* mean `T` equals `C`: `longest2("alpha", "beta")` returns `string`, not `{length: number}`. That's the whole point of using a generic instead of just typing the parameter as `{length: number}`.

**`keyof` plus a constraint** gives you the single most useful generic pattern in the language — a type-safe property getter:

```ts
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, name: "Ada", active: true };
getProp(user, "name");     // string
getProp(user, "active");   // boolean
getProp(user, "nope");     // ❌ ts(2345): Argument of type '"nope"' is not
                           //    assignable to parameter of type '"id" | "name" | "active"'.
```

Three operators are doing the work, and they compose everywhere:

```ts
type User = { id: number; name: string };

type Keys = keyof User;            // "id" | "name"
type NameType = User["name"];      // string        ← indexed access
type Either = User[keyof User];    // number | string
type UserType = typeof user;       // { id: number; name: string }  ← value → type
```

**Defaults** let a type parameter be omitted:

```ts
interface ApiResponse<TData = unknown, TError = Error> {
  data: TData | null;
  error: TError | null;
}

type Simple = ApiResponse;                 // ApiResponse<unknown, Error>
type Typed = ApiResponse<User>;            // ApiResponse<User, Error>
```

Defaults are **not** constraints, and you often want both — `<T extends object = {}>` reads "T must be an object; if unspecified, it's `{}`".

<a id="p8-advanced"></a>

### Advanced — inference sites, and why your generic infers `unknown`

This is the section that pays for the whole Part. Inference feels magical until you know where the checker is *allowed* to look; then it becomes predictable.

**An inference site is a position in the parameter list where a type parameter appears.** The checker collects a candidate from each site and reconciles them.

```text
  function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K>
                 ▲            ▲            ▲       ▲
                 │            │            │       └── inference site for K
                 │            │            └────────── inference site for T
                 │            └─────────────────────── constraint, NOT a site
                 └──────────────────────────────────── declaration

  pick({ a: 1, b: "x" }, ["a"])
        └── T = { a: number; b: string }   from site 1
                              └── K = "a"  from site 2 (literal, in an array position)
        → returns Pick<{a: number; b: string}, "a"> = { a: number }
```

**Failure mode 1: no inference site at all → the parameter falls back to its constraint, or `unknown`.**

```ts
// ❌ T appears only in the return type. Nothing to infer from.
function parse<T>(json: string): T { return JSON.parse(json) }
const u = parse("{}");      // T = unknown

// This signature is also actively dangerous — the caller "chooses" T with no
// evidence, which is `as` in disguise:
const u2 = parse<User>("{}");   // compiles; verifies nothing. See Part 12.

// ✅ If T can't be inferred, don't make it generic — return `unknown` and
//    force the caller to narrow or validate.
function parse2(json: string): unknown { return JSON.parse(json) }
```

**"A generic type parameter that appears only once is probably not doing anything."** That is a good heuristic for reviewing generic code: a type parameter needs at least one site to be *inferred from* and one place it's *used*, otherwise it's decoration or a disguised assertion.

**Failure mode 2: inference happens before the callback is checked.**

```ts
declare function mapValues<T, U>(obj: Record<string, T>, fn: (v: T) => U): Record<string, U>;

mapValues({ a: 1, b: 2 }, v => v.toFixed(2));   // ✅ T=number from arg 1, then v: number
```

That works because `T`'s site comes **first**. Reverse the parameters and `v` becomes implicitly `any` (`ts(7006)`) because nothing has fixed `T` yet. **Put value parameters before function parameters** in generic helpers — the same left-to-right rule as contextual typing in [Part 5](./typescript-narrowing-functions.md#part-5).

**Failure mode 3: literal widening at the inference site.**

```ts
function wrap<T>(value: T) { return { value } }

wrap("hello");            // { value: string }    ← widened
wrap("hello" as const);   // { value: "hello" }
```

The checker widens literals unless something asks it not to. Three ways to ask:

```ts
// A — constrain to a primitive; constraints prevent widening
function wrap2<T extends string>(value: T) { return { value } }
wrap2("hello");              // { value: "hello" }

// B — as const at the call site
wrap("hello" as const);

// C — a `const` type parameter (TS 5.0+) — see Mastery below
function wrap3<const T>(value: T) { return { value } }
wrap3("hello");              // { value: "hello" }
```

**Failure mode 4: multiple sites disagree.**

```ts
declare function pair<T>(a: T, b: T): [T, T];

pair(1, 2);          // [number, number]
pair(1, "x");        // ❌ ts(2345): Argument of type 'string' is not assignable
                     //    to parameter of type 'number'.
```

TypeScript picks the best common candidate from the sites and reports a conflict rather than silently unioning. If you *want* the union, say so explicitly:

```ts
declare function pair2<T, U>(a: T, b: U): [T, U];      // two parameters
declare function pair3<T>(a: T, b: NoInfer<T>): [T, T]; // or: fix T from `a` only
```

**`NoInfer<T>`** (TypeScript 5.4+) is the precise tool for that last case — it marks a position as "use `T`, but don't collect a candidate from here":

```ts
function createState<T>(initial: T, validValues: NoInfer<T>[]) { /* … */ }

createState("green", ["red", "green"]);
// T = "green"… wait, T = string (widened from `initial`) and the array is checked
// against it. Without NoInfer, T would infer as "green" | "red" | "green",
// silently accepting an initial value that isn't in the valid list.
```

**Try It — see all four failure modes at once.**

```ts
declare function process<T>(items: T[], transform: (item: T) => T): T[];

process([1, 2, 3], n => n * 2);          // ①
process([], n => n);                     // ②
process(["a"], s => s.length);           // ③
```

*Expected:* ① `number[]`. ② `never[]` — an empty array literal gives no candidate, so `T` collapses to `never` (hover `n` to see it). ③ `ts(2345)`, because `transform` must return `T` (`string`) and returns `number`. If those three results are unsurprising, you understand inference sites.

<a id="p8-mastery"></a>

### Mastery — `const` type parameters, variance annotations, generic design rules

**`const` type parameters** (TypeScript 5.0) move the `as const` burden from the caller to the API author. This is the difference between a library that feels good and one that requires ceremony:

```ts
// Without — every caller must remember `as const`
declare function defineRoutes<T>(routes: T): T;
const r1 = defineRoutes({ home: "/" });              // { home: string }
const r2 = defineRoutes({ home: "/" } as const);     // { readonly home: "/" }

// With — the call site stays clean
declare function defineRoutes2<const T>(routes: T): T;
const r3 = defineRoutes2({ home: "/" });             // { readonly home: "/" }
```

The caveat worth knowing: `const T` only affects inference from **literal expressions written at the call site**. Pass a variable and you get its (already widened) type back. And it doesn't apply through a constraint like `T extends string[]` — `T extends readonly string[]` is what you want, since `const` inference produces `readonly` arrays.

**Variance annotations** (TypeScript 4.7) let you state a type parameter's variance explicitly instead of letting the checker compute it:

```ts
interface Producer<out T> { get(): T }        // covariant — T only in output position
interface Consumer<in T> { set(value: T): void }  // contravariant — input only
interface Box<in out T> { get(): T; set(v: T): void }  // invariant — both
```

These are primarily a **performance and intent** tool, not a correctness one: TypeScript already infers variance structurally, but on large recursive types that inference is expensive, and an explicit annotation short-circuits it. It also makes a mistake loud:

```ts
interface Bad<out T> { set(value: T): void }
// ❌ ts(2636): Type 'Bad<T>' is not assignable to type 'Bad<super-T>' as implied
//    by variance annotation.
```

Use them in library code with deep generic types; skip them in application code.

**Generic classes and the static gotcha:**

```ts
class Container<T> {
  constructor(public value: T) {}
  static empty: T;   // ❌ ts(2302): Static members cannot reference class type parameters.
}
```

Type parameters belong to *instances*; statics are shared across every instantiation, so there's no `T` for them to mean. Make the static method generic on its own instead:

```ts
class Container2<T> {
  constructor(public value: T) {}
  static of<U>(value: U): Container2<U> { return new Container2(value) }
}
```

**Generic design rules**, in the order they pay off:

1. **Every type parameter needs a job.** At least one inference site, at least one use. If `T` appears once, delete it (see failure mode 1).
2. **Constrain as loosely as the body allows.** `<T extends { length: number }>` beats `<T extends string>` if you only read `.length`; `readonly T[]` beats `T[]` if you don't mutate.
3. **Order parameters value-first, callback-last.** Inference flows left to right.
4. **Return the specific type, not the constraint.** `function f<T extends Animal>(x: T): T` preserves information; `(x: Animal): Animal` throws it away.
5. **Prefer inference to explicit type arguments.** If callers routinely have to write `f<Something>(…)`, the signature is wrong.
6. **Reach for `NoInfer` when one parameter should follow another**, rather than adding a second type parameter you then have to constrain.

**Real Scenario — the repository that erased its own types.** A team wrote:

```ts
class Repo<T> {
  async findAll(): Promise<T[]> { return this.db.query(this.table) }   // db.query → any
}
```

`db.query` returned `any`, so `T[]` was satisfied without inspection, and `Repo<User>` handed back rows with entirely different columns after a migration. Nothing in the generic layer failed — **`any` satisfies every type parameter**, so a single untyped driver call silently disabled type safety for every consumer of the class. The fix was to type the driver as returning `unknown[]` and validate at the boundary ([Part 12](./typescript-modules-ecosystem.md#part-12)). The general lesson: a generic is only as trustworthy as the narrowest type on the path into it, and generics are unusually good at *hiding* where that path started.

<a id="p8-cheatsheet"></a>

### Part 8 cheat sheet

| Need | Syntax |
|---|---|
| Type parameter | `function f<T>(x: T): T` |
| Constraint | `<T extends Base>` |
| Default | `<T = string>` |
| Both | `<T extends object = {}>` |
| Keys of a type | `<K extends keyof T>` |
| Value at a key | `T[K]` |
| Preserve literals | `<const T>` (5.0+) or `as const` at the call site |
| Block a site from inferring | `NoInfer<T>` (5.4+) |
| Declare variance | `<out T>` / `<in T>` / `<in out T>` |
| Generic type alias | `type Box<T> = { value: T }` |
| Generic class | `class C<T> { … }` (no `T` in statics) |

| Symptom | Cause | Fix |
|---|---|---|
| `T` infers `unknown` | no inference site | make the parameter appear in an argument, or don't use a generic |
| `T` infers `never` | empty array/object literal argument | annotate the argument, or pass an explicit type argument |
| Callback params implicitly `any` | the value parameter comes after the callback | reorder — value first |
| Lost literal types | widening at the inference site | `const T`, `as const`, or constrain to a primitive |
| `ts(2345)` on the second argument | two sites disagree | split into `<T, U>`, or `NoInfer<T>` on the second |
| `ts(2302)` on a static | statics can't see instance `T` | make the static itself generic |
| Generic compiles but types are wrong at runtime | an `any` upstream satisfied `T` | type the boundary as `unknown` and validate |

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 9 — Type-Level Programming](./typescript-type-level.md#part-9), where `T` becomes something you can compute with.

*Part of the [TypeScript doc set](./typescript-mastery-guide.md) in [my-learning-md-docs](../README.md). Last updated: September 6, 2026.*
