# TypeScript — Type-Level Programming (Parts 9–10)

> **About this document:** this is one of the sub-docs in this repo's TypeScript set. The canonical spec — framing, scope, tier policy, sourcing rule, versions, and how to update it — lives once in [`typescript-mastery-guide.md`](./typescript-mastery-guide.md#about-this-document). Written against **TypeScript 7.0**, September 6, 2026. Assumes [Part 8 — Generics](./typescript-generics.md#part-8).

Part 9 is the expert tier: conditional types, `infer`, mapped types, template literal types — the machinery library authors use, and the machinery you need to *read* library types even if you never write them. Part 10 rebuilds the standard utility types from that machinery, which is the fastest way to stop finding them magical.

---

## Table of Contents

- [Part 9 — Type-Level Programming](#part-9)
  - [Beginner — conditional types](#p9-beginner)
  - [Working Knowledge — `infer`, mapped types, `satisfies`](#p9-working)
  - [Advanced — distributivity, key remapping, template literal types](#p9-advanced)
  - [Mastery — recursion, performance, and knowing when to stop](#p9-mastery)
  - [Part 9 cheat sheet](#p9-cheatsheet)
- [Part 10 — The Utility Types, Rebuilt](#part-10)
  - [Working Knowledge — the ones you'll use weekly](#p10-working)
  - [Advanced — rebuilding them from scratch](#p10-advanced)
  - [Mastery — the ones with sharp edges, and what's missing](#p10-mastery)
  - [Part 10 cheat sheet](#p10-cheatsheet)

---

<a id="part-9"></a>

## Part 9 — Type-Level Programming

<a id="p9-beginner"></a>

### Beginner — conditional types

A [conditional type](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html) is a ternary that runs in the type system:

```ts
type IsString<T> = T extends string ? "yes" : "no";

type A = IsString<"hello">;   // "yes"
type B = IsString<42>;        // "no"
```

`T extends U` here asks **"is T assignable to U?"** — the same question from [Part 2's lattice](./typescript-foundations.md#part-2), now used as a branch condition rather than a constraint.

The everyday use is making a return type depend on an argument type:

```ts
type Unwrap<T> = T extends Promise<infer U> ? U : T;

type C = Unwrap<Promise<string>>;   // string
type D = Unwrap<number>;            // number
```

<a id="p9-working"></a>

### Working Knowledge — `infer`, mapped types, `satisfies`

**`infer` binds a matched type to a variable**, like a capture group in a regex. It is only legal in the `extends` clause of a conditional type:

```ts
type ElementOf<T>   = T extends (infer E)[] ? E : never;
type ReturnOf<T>    = T extends (...args: never[]) => infer R ? R : never;
type FirstParam<T>  = T extends (first: infer P, ...rest: never[]) => unknown ? P : never;
type Awaited1<T>    = T extends Promise<infer U> ? U : T;

type E1 = ElementOf<string[]>;                    // string
type E2 = ReturnOf<() => number>;                 // number
type E3 = FirstParam<(id: string, n: number) => void>;   // string
```

You can constrain an `infer` (`infer E extends string`, TypeScript 4.8+), and you can use several in one pattern:

```ts
type Split2<T> = T extends [infer A, infer B] ? { a: A; b: B } : never;
type E4 = Split2<[string, number]>;   // { a: string; b: number }
```

**Mapped types** transform every member of an existing type. They are `for…of` for the type system:

```ts
type Optional<T> = { [K in keyof T]?: T[K] };
type Nullable<T> = { [K in keyof T]: T[K] | null };
type Stringify<T> = { [K in keyof T]: string };

type User = { id: number; name: string };
type F1 = Optional<User>;   // { id?: number; name?: string }
type F2 = Nullable<User>;   // { id: number | null; name: string | null }
```

The modifiers `readonly` and `?` can be **added with `+`** or **removed with `-`**:

```ts
type Mutable<T>  = { -readonly [K in keyof T]: T[K] };
type Required2<T> = { [K in keyof T]-?: T[K] };
type Frozen<T>   = { +readonly [K in keyof T]+?: T[K] };   // `+` is the default; explicit here
```

Mapped types over a **union of keys** rather than an existing object are how you build lookup tables:

```ts
type Status = "idle" | "done";
type Handlers = { [K in Status]: () => void };   // { idle: () => void; done: () => void }
```

**`satisfies`** (TypeScript 4.9) is the operator most people wish they'd learned earlier. It checks a value against a type **without widening it to that type**:

```ts
type Config = Record<string, string | number>;

// ❌ annotation: checked, but the specific types are lost
const c1: Config = { port: 3000, host: "localhost" };
c1.port.toFixed();   // ts(2339) — Property 'toFixed' does not exist on
                     // type 'string | number'.

// ❌ assertion: keeps the type, but checks nothing
const c2 = { port: 3000, hostt: "localhost" } as Config;   // typo sails through

// ✅ satisfies: checked AND specific
const c3 = { port: 3000, host: "localhost" } satisfies Config;
c3.port.toFixed();   // ✅ port is number
c3.hots;             // ❌ ts(2339) — typos still caught
```

| Operator | Checks? | Resulting type |
|---|---|---|
| `const x: T = …` | ✅ | `T` (widened) |
| `const x = … as T` | ⚠️ assignability only, both directions | `T` |
| `const x = … satisfies T` | ✅ | the **inferred** narrow type |

Use `satisfies` for config objects, route tables, constant maps — anywhere you want validation against a contract but need the precise literal types afterwards.

<a id="p9-advanced"></a>

### Advanced — distributivity, key remapping, template literal types

**Distributivity is the behaviour that surprises everyone.** When a conditional type's checked type is a *naked type parameter* and you hand it a union, the conditional distributes over each member and re-unions the results:

```ts
type ToArray<T> = T extends unknown ? T[] : never;

type G1 = ToArray<string | number>;
//   distributes → ToArray<string> | ToArray<number>
//              → string[] | number[]        ← NOT (string | number)[]
```

This is exactly what you want for filtering unions:

```ts
type NonNull<T> = T extends null | undefined ? never : T;
type G2 = NonNull<string | null | undefined>;   // string
// (each member is tested separately; the `never`s vanish from the union)
```

…and exactly what you don't want when you're asking a question about the union *as a whole*. **Wrap both sides in a tuple to switch distribution off:**

```ts
type IsUnion1<T> = T extends unknown ? true : false;
type H1 = IsUnion1<string | number>;    // boolean  ← distributed into true | false

type Exact<T> = [T] extends [string] ? true : false;   // ✅ non-distributive
type H2 = Exact<string | number>;        // false
type H3 = Exact<string>;                 // true
```

The `[T] extends [U]` idiom appears throughout library type code for precisely this reason. Two related gotchas fall out of it:

```ts
// `never` is the empty union, so distributing over it produces nothing
type Distribute<T> = T extends unknown ? T[] : never;
type H4 = Distribute<never>;       // never   ← not never[]!

// `boolean` is secretly `true | false`, so it distributes too
type H5 = ToArray<boolean>;        // true[] | false[]
```

**Key remapping with `as`** (TypeScript 4.1) lets a mapped type rename or drop keys:

```ts
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K]
};

type User = { name: string; age: number };
type I1 = Getters<User>;
// { getName: () => string; getAge: () => number }

// Mapping a key to `never` REMOVES it — this is how filtering works
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K]
};
type I2 = OnlyStrings<{ a: string; b: number; c: string }>;   // { a: string; c: string }
```

**Template literal types** (TypeScript 4.1) compute string types:

```ts
type Event = "click" | "focus";
type Handler = `on${Capitalize<Event>}`;      // "onClick" | "onFocus"
type CssVar = `--${string}`;                   // any string starting with --

// Unions multiply across positions
type Size = "sm" | "lg";
type Colour = "red" | "blue";
type Class = `${Size}-${Colour}`;   // "sm-red" | "sm-blue" | "lg-red" | "lg-blue"
```

The four intrinsic string types — `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize` — are implemented in the compiler, not in `.d.ts`, which is why you can't write your own.

Combined with `infer`, template literals become a parser:

```ts
type Split<S extends string, D extends string> =
  S extends `${infer Head}${D}${infer Tail}` ? [Head, ...Split<Tail, D>] : [S];

type J1 = Split<"a.b.c", ".">;   // ["a", "b", "c"]
```

> **TypeScript 7 change worth knowing:** template-literal `infer` now splits by **Unicode code point** rather than UTF-16 code unit, so `` `${infer H}${infer T}` `` on `"😀abc"` yields `["😀", "abc"]` instead of a broken surrogate half ([7.0 announcement](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)). String-manipulating types written before 2026 may quietly behave better now — and any test that asserted the old broken behaviour will fail.

<a id="p9-mastery"></a>

### Mastery — recursion, performance, and knowing when to stop

**Recursive conditional types** (uncapped since TypeScript 4.5 for tail-recursive forms) let you walk arbitrarily deep structures:

```ts
type DeepReadonly<T> = T extends (infer E)[]
  ? readonly DeepReadonly<E>[]
  : T extends object
    ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
    : T;

type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;
```

Watch out for what counts as an `object`: functions, `Date`, `Map`, and arrays all pass `T extends object`, so a naive `DeepPartial` will happily make `Date`'s internal methods optional. Production versions bail out explicitly:

```ts
type DeepPartial2<T> = T extends Date | RegExp | ((...a: never[]) => unknown)
  ? T
  : T extends object
    ? { [K in keyof T]?: DeepPartial2<T[K]> }
    : T;
```

**Type-level arithmetic** exists, is genuinely used by libraries, and should be treated as a last resort:

```ts
type BuildTuple<N extends number, R extends unknown[] = []> =
  R["length"] extends N ? R : BuildTuple<N, [...R, unknown]>;

type Add<A extends number, B extends number> =
  [...BuildTuple<A>, ...BuildTuple<B>]["length"];

type K1 = Add<3, 4>;   // 7
```

It counts by *building tuples*, because `["length"]` is the only numeric readout the type system offers. It works up to roughly the tuple-length limit and then falls over. If you find yourself here in application code, you have gone too far.

**Performance and the recursion limits.** The checker enforces two hard caps:

```ts
type Infinite<T> = { value: Infinite<T> };   // fine — lazy
type Explode<N extends number> = BuildTuple<N>;
type L1 = Explode<10000>;
// ❌ ts(2589): Type instantiation is excessively deep and possibly infinite.
```

- **instantiation depth ~50** for non-tail-recursive types, ~1000 for tail-recursive ones;
- a **complexity budget** across the whole check — you can hit `ts(2589)` in a file that individually looks fine.

When a build gets slow, find the culprit rather than guessing:

```bash
npx tsc --noEmit --diagnostics       # phase timings; is `check` the hot phase?
npx tsc --generateTrace ./trace      # then open trace/trace.json in https://ui.perfetto.dev
```

The trace names the specific type whose instantiation is expensive. In practice the answer is nearly always: a deeply recursive conditional type applied to a large union, or a mapped type over `keyof` of a very wide interface.

**Knowing when to stop.** Type-level programming has a cost curve that turns sharply upward, and the costs are borne by the *next* person:

| Reach for it when | Don't when |
|---|---|
| You're authoring a library API others infer through | It's application code with a known shape |
| The alternative is dozens of near-identical hand-written types | You could just write the type out |
| Correctness depends on the relationship (e.g. keys of a form matching its validators) | You're proving you can |
| A generated type would drift from its source | A code generator (`zod-to-ts`, `prisma`, `openapi-typescript`) would be simpler and debuggable |

The honest rule: **if you can't explain the type to a colleague in two sentences, write the concrete types instead, or generate them.** A `ts(2589)` at 5pm on a Friday is not a good trade for saving twenty lines of interface.

**Try It — distributivity, empirically.**

```ts
type Box<T> = T extends unknown ? { v: T } : never;
type M1 = Box<string | number>;
type M2 = Box<never>;
type M3 = [string | number] extends [unknown] ? "yes" : "no";

type NonDist<T> = [T] extends [unknown] ? { v: T } : never;
type M4 = NonDist<string | number>;
```

*Expected:* `M1` = `{v: string} | {v: number}`; `M2` = `never` (not `{v: never}`); `M3` = `"yes"`; `M4` = `{ v: string | number }`. If you predicted `M2`, you have the model.

<a id="p9-cheatsheet"></a>

### Part 9 cheat sheet

| Construct | Syntax | Notes |
|---|---|---|
| Conditional | `T extends U ? A : B` | asks "is T assignable to U" |
| Infer | `T extends Promise<infer R> ? R : T` | only in an `extends` clause |
| Constrained infer | `infer R extends string` | 4.8+ |
| Mapped | `{ [K in keyof T]: … }` | |
| Add/remove modifiers | `-readonly`, `-?`, `+readonly`, `+?` | |
| Key remap | `{ [K in keyof T as NewKey]: … }` | 4.1+; map to `never` to drop a key |
| Template literal | `` `on${Capitalize<E>}` `` | unions multiply |
| Intrinsics | `Uppercase` `Lowercase` `Capitalize` `Uncapitalize` | compiler-implemented |
| Block distribution | `[T] extends [U] ? … : …` | the standard idiom |
| Check without widening | `x satisfies T` | 4.9+ |

| Symptom | Cause | Fix |
|---|---|---|
| Conditional returned a union you didn't expect | distributivity | wrap in `[T]` |
| `never` input produced `never` output | `never` is the empty union | `[T] extends [never]` to test for it |
| `boolean` behaved like two types | it is `true \| false` | same tuple trick |
| `ts(2589)` | recursion/complexity limit | reduce depth, add a base case, or simplify |
| Slow build after adding a type | expensive instantiation | `--generateTrace` |
| Mapped type lost `readonly`/`?` | homomorphic mapping preserves them only over `keyof T` | map over `keyof T`, not an arbitrary union |

[↑ Back to top](#table-of-contents)

---

<a id="part-10"></a>

## Part 10 — The Utility Types, Rebuilt

> **Tier note:** this Part runs **Working Knowledge → Mastery**, with no Beginner tier. The utility types have no meaningful beginner layer — you either know one exists or you don't — and their internals are Part 9's machinery applied.

<a id="p10-working"></a>

### Working Knowledge — the ones you'll use weekly

All of these are in [`lib.es5.d.ts`](https://www.typescriptlang.org/docs/handbook/utility-types.html), globally available, no import.

```ts
type User = { id: number; name: string; email?: string };

Partial<User>            // all properties optional
Required<User>           // all properties required
Readonly<User>           // all properties readonly
Pick<User, "id" | "name">      // { id: number; name: string }
Omit<User, "email">            // { id: number; name: string }
Record<"a" | "b", number>      // { a: number; b: number }

Exclude<"a" | "b" | "c", "a">      // "b" | "c"       (union subtraction)
Extract<"a" | "b", "a" | "z">      // "a"             (union intersection)
NonNullable<string | null>          // string

ReturnType<() => User>              // User
Parameters<(a: string, b: number) => void>   // [a: string, b: number]
Awaited<Promise<Promise<string>>>   // string          (recursively unwraps)
ConstructorParameters<typeof Date>  // constructor args
InstanceType<typeof Date>           // Date
```

The four string intrinsics: `Uppercase<S>`, `Lowercase<S>`, `Capitalize<S>`, `Uncapitalize<S>`.

The everyday combinations worth memorising:

```ts
type UserUpdate = Partial<Omit<User, "id">>;              // patch payload
type UserSummary = Pick<User, "id" | "name">;             // list projection
type CreateUser = Omit<User, "id"> & { password: string }; // creation DTO
type Config = Readonly<Record<string, string>>;           // frozen lookup
```

<a id="p10-advanced"></a>

### Advanced — rebuilding them from scratch

Every one of these is a few lines of Part 9 machinery. Reading them once removes the last of the magic.

```ts
// Mapped types with modifier changes
type MyPartial<T>  = { [K in keyof T]?: T[K] };
type MyRequired<T> = { [K in keyof T]-?: T[K] };
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };
type MyMutable<T>  = { -readonly [K in keyof T]: T[K] };

// Mapped over a key union
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };
type MyRecord<K extends keyof never, V> = { [P in K]: V };

// Distributive conditionals over unions
type MyExclude<T, U> = T extends U ? never : T;
type MyExtract<T, U> = T extends U ? T : never;
type MyNonNullable<T> = T & {};        // the actual lib definition since 4.8

// Omit is Pick + Exclude — and this composition explains its famous quirk
type MyOmit<T, K extends keyof never> = MyPick<T, MyExclude<keyof T, K>>;

// infer-based extraction
type MyReturnType<T extends (...a: never[]) => unknown> =
  T extends (...a: never[]) => infer R ? R : never;
type MyParameters<T extends (...a: never[]) => unknown> =
  T extends (...a: infer P) => unknown ? P : never;

// Recursive
type MyAwaited<T> = T extends Promise<infer U> ? MyAwaited<U> : T;
```

**Now the famous quirk, explained.** `Omit`'s second parameter is `keyof never` (i.e. `string | number | symbol`), *not* `keyof T`:

```ts
type Bad = Omit<User, "emial">;   // ✅ compiles! Typo silently ignored.
type Bad2 = Pick<User, "emial">;  // ❌ ts(2344) — Pick DOES constrain
```

This is deliberate — a looser constraint makes `Omit` usable on generic `T` where the keys aren't yet known — but it means `Omit` will never catch your typos. If you want that safety, define a strict version:

```ts
type StrictOmit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;
type Bad3 = StrictOmit<User, "emial">;
// ❌ ts(2344): Type '"emial"' does not satisfy the constraint 'keyof User'.
```

**`Omit` also flattens unions**, which is the sharper edge of the two:

```ts
type Shape =
  | { kind: "circle"; r: number; id: string }
  | { kind: "square"; s: number; id: string };

type WithoutId = Omit<Shape, "id">;
// { kind: "circle" | "square"; r?: … } — NOT a union any more!
// The discriminated union is destroyed; narrowing no longer works.

// ✅ distribute manually
type DistributiveOmit<T, K extends keyof never> =
  T extends unknown ? Omit<T, K> : never;
type WithoutId2 = DistributiveOmit<Shape, "id">;
// { kind: "circle"; r: number } | { kind: "square"; s: number }   ✅
```

`Omit`, `Pick`, and `Partial` are all non-distributive. Any time you apply one to a discriminated union and narrowing mysteriously stops working, this is why.

<a id="p10-mastery"></a>

### Mastery — the ones with sharp edges, and what's missing

**`Record<string, T>` claims total coverage it doesn't have.** Combine with [`noUncheckedIndexedAccess`](./typescript-foundations.md#part-3) or accept that every lookup is a lie:

```ts
const users: Record<string, User> = {};
users["missing"].name;   // without the flag: compiles, crashes
```

**`Function`, `object`, and `{}` are traps.** `Function` accepts any callable and gives you `any` arguments; `{}` means "anything except null/undefined" (including numbers and strings). Prefer:

```ts
// ❌            // ✅
Function         (...args: never[]) => unknown
object           Record<string, unknown>
{}               unknown
any[]            unknown[]
```

**`Parameters` and `ReturnType` only see the last overload.** A function with three overloads returns just the final signature's types — a real limitation with no clean workaround.

**Useful types the standard library *doesn't* have**, and which you'll end up writing (or getting from [type-fest](https://github.com/sindresorhus/type-fest), the de facto standard collection):

```ts
// Make specific keys optional / required
type PartialBy<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;
type RequiredBy<T, K extends keyof T> = Omit<T, K> & Required<Pick<T, K>>;

// Distribute over unions properly
type DistributiveOmit2<T, K extends keyof never> = T extends unknown ? Omit<T, K> : never;

// Deep variants
type DeepPartial<T> = T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } : T;

// Keys whose values match a type
type KeysOfType<T, V> = { [K in keyof T]-?: T[K] extends V ? K : never }[keyof T];
type StringKeys = KeysOfType<{ a: string; b: number }, string>;   // "a"

// Exactly one of a set of keys
type ExactlyOne<T, K extends keyof T = keyof T> =
  K extends unknown ? Required<Pick<T, K>> & Partial<Record<Exclude<keyof T, K>, never>> : never;

// Force the editor to display a computed type expanded instead of as an alias soup
type Prettify<T> = { [K in keyof T]: T[K] } & {};
```

That last one is worth its weight — wrap any gnarly computed type in `Prettify<…>` and hover tooltips become readable. It's a no-op at the type level; it just forces the checker to eagerly evaluate the mapping.

**Real Scenario — the `Partial` that hid a required field.** An API accepted `Partial<User>` as its update payload so callers could send any subset. Six months later `User` gained `tenantId: string`, required everywhere else in the system. Because the update endpoint took `Partial<User>`, the new field was optional there, and updates silently wrote rows with no tenant — invisible to every existing test, and invisible to the type system, which was doing exactly what it had been told. The fix was `type UserUpdate = PartialBy<Omit<User, "id">, "email" | "avatar">` — enumerating what may be omitted rather than blanket-optionalising everything. **`Partial<T>` on a payload type is a decision to stop tracking that payload's requirements**, and it ages badly.

<a id="p10-cheatsheet"></a>

### Part 10 cheat sheet

| Utility | Does | Distributes over unions? |
|---|---|---|
| `Partial<T>` | all optional | ❌ |
| `Required<T>` | all required | ❌ |
| `Readonly<T>` | all readonly | ❌ |
| `Pick<T, K>` | keep keys `K` (K constrained to `keyof T`) | ❌ |
| `Omit<T, K>` | drop keys `K` (**K not constrained**) | ❌ |
| `Record<K, V>` | build an object type | ❌ |
| `Exclude<T, U>` | remove union members | ✅ |
| `Extract<T, U>` | keep union members | ✅ |
| `NonNullable<T>` | drop `null`/`undefined` | ✅ |
| `ReturnType<F>` | function's return | last overload only |
| `Parameters<F>` | parameters as a tuple | last overload only |
| `Awaited<T>` | unwrap promises recursively | ✅ |
| `InstanceType<C>` / `ConstructorParameters<C>` | class construct signature | — |

| Trap | Guard |
|---|---|
| `Omit` ignores typo'd keys | use `StrictOmit<T, K extends keyof T>` |
| `Omit`/`Pick` flatten discriminated unions | use a distributive variant |
| `Record<string, T>` lookups are never `undefined` | `noUncheckedIndexedAccess` |
| `Partial<T>` on a payload hides new required fields | enumerate optional keys with `PartialBy` |
| Tooltip shows unreadable alias soup | wrap in `Prettify<T>` |
| Writing your own for the tenth time | use [type-fest](https://github.com/sindresorhus/type-fest) |

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 11 — Modules & Declaration Files](./typescript-modules-ecosystem.md#part-11).

*Part of the [TypeScript doc set](./typescript-mastery-guide.md) in [my-learning-md-docs](../README.md). Last updated: September 6, 2026.*
