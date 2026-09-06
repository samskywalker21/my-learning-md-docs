# TypeScript — Objects & Classes (Parts 6–7)

> **About this document:** this is one of the sub-docs in this repo's TypeScript set. The canonical spec — framing, scope, tier policy, sourcing rule, versions, and how to update it — lives once in [`typescript-mastery-guide.md`](./typescript-mastery-guide.md#about-this-document). Written against **TypeScript 7.0**, September 6, 2026. Assumes the [JavaScript doc set](../javascript/javascript-mastery-guide.md) — in particular [Objects & Classes](../javascript/javascript-objects-classes.md), whose prototype and `#private` material is not repeated here.

Part 6 is how you describe the shape of data — the bulk of the typing you'll ever write. Part 7 is classes, where TypeScript adds real syntax to JavaScript rather than just annotations.

---

## Table of Contents

- [Part 6 — Object Types](#part-6)
  - [Beginner — describing a shape](#p6-beginner)
  - [Working Knowledge — `interface` vs `type`, index signatures, `readonly`](#p6-working)
  - [Advanced — excess property checks, intersections, declaration merging](#p6-advanced)
  - [Mastery — variance, `Record` vs `Map`, and modelling to make bad states impossible](#p6-mastery)
  - [Part 6 cheat sheet](#p6-cheatsheet)
- [Part 7 — Classes](#part-7)
  - [Beginner — fields, constructors, methods](#p7-beginner)
  - [Working Knowledge — modifiers, `implements`, `abstract`](#p7-working)
  - [Advanced — `private` vs `#private`, `override`, initialization](#p7-advanced)
  - [Mastery — decorators, and when not to use a class at all](#p7-mastery)
  - [Part 7 cheat sheet](#p7-cheatsheet)

---

<a id="part-6"></a>

## Part 6 — Object Types

<a id="p6-beginner"></a>

### Beginner — describing a shape

Three ways to write the same object type. They differ in what else they can do, not in what they describe:

```ts
// inline (anonymous)
function print(u: { name: string; age: number }) {}

// type alias
type User = { name: string; age: number };

// interface
interface User2 { name: string; age: number }
```

Members can be optional, read-only, or methods:

```ts
interface Post {
  readonly id: string;      // assignable at creation only
  title: string;
  subtitle?: string;        // optional → string | undefined
  publish(): void;          // method syntax
  onSave: () => void;       // property syntax (see Part 5 on variance)
}
```

Nesting works as you'd expect, and so does composition:

```ts
interface Address { city: string; postcode: string }
interface Customer {
  name: string;
  address: Address;
  orders: { id: string; total: number }[];
}
```

<a id="p6-working"></a>

### Working Knowledge — `interface` vs `type`, index signatures, `readonly`

**The one real difference: interfaces are *open*, type aliases are *closed*.** An interface declared twice merges; a type alias declared twice is an error.

```ts
interface Window { title: string }
interface Window { theme: string }   // ✅ merges → { title: string; theme: string }

type Config = { a: string };
type Config = { b: string };         // ❌ ts(2300): Duplicate identifier 'Config'.
```

Everything else follows from that, plus the fact that a type alias can name things an interface cannot:

| Capability | `interface` | `type` |
|---|---|---|
| Object shapes | ✅ | ✅ |
| Extend / compose | `extends` | `&` |
| **Declaration merging** | ✅ | ❌ |
| **Union types** | ❌ | ✅ `A \| B` |
| **Primitives, tuples, mapped, conditional types** | ❌ | ✅ |
| Implemented by a class | ✅ | ✅ (if it's an object type) |
| Error messages | shows the interface **name** | often expands the whole shape |
| Performance on large shapes | cached by name; usually faster | intersections can be recomputed |

**Practical rule**, which matches how the TypeScript team's own codebase is written: **`interface` for object shapes you expect others to extend or augment (especially public API surface), `type` for everything else** — unions, function types, tuples, and anything computed. If you're genuinely undecided on a plain object shape, `interface` gives slightly better error messages and slightly better checker performance. Do not let anyone tell you this is a significant decision; it isn't, as long as you're consistent.

The one place it *is* significant is augmenting library types, which only `interface` can do — see [module augmentation in Part 11](./typescript-modules-ecosystem.md#part-11).

**Index signatures** describe objects with dynamic keys:

```ts
interface Dictionary { [key: string]: number }

const scores: Dictionary = { alice: 10, bob: 8 };
scores.charlie = 12;   // ✅ any string key is allowed
```

Two rules bite immediately:

```ts
// 1. Declared properties must be compatible with the index signature
interface Bad {
  [key: string]: number;
  name: string;   // ❌ ts(2411): Property 'name' of type 'string' is not
}                 //    assignable to 'string' index type 'number'.

interface Good {
  [key: string]: number | string;
  name: string;   // ✅
}

// 2. Index signatures LIE by default about missing keys
const s: Dictionary = {};
s.missing.toFixed(2);   // compiles. Crashes. → number, but it's undefined.
```

The fix for (2) is [`noUncheckedIndexedAccess`](./typescript-foundations.md#part-3), which makes every index access `T | undefined`. This is the single most important reason to enable that flag.

**`readonly`** is shallow and compile-time only:

```ts
interface Point { readonly x: number; readonly y: number }
const p: Point = { x: 1, y: 2 };
p.x = 5;   // ❌ ts(2540): Cannot assign to 'x' because it is a read-only property.

// but:
interface Wrapper { readonly inner: { value: number } }
declare const w: Wrapper;
w.inner.value = 5;   // ✅ readonly is SHALLOW — only `inner` itself is protected

// and it is erased — nothing stops JavaScript at runtime
const escaped = p as { x: number };
escaped.x = 5;       // ✅ compiles, and mutates. Use Object.freeze for real immutability.
```

`readonly T[]` (or `ReadonlyArray<T>`) removes the mutating methods, and is worth using on every function parameter you don't intend to mutate:

```ts
function sum(xs: readonly number[]) {
  xs.push(1);   // ❌ ts(2339): Property 'push' does not exist on type 'readonly number[]'.
  return xs.reduce((a, b) => a + b, 0);
}
sum([1, 2, 3]);        // ✅ mutable arrays are assignable to readonly ones (not vice versa)
```

<a id="p6-advanced"></a>

### Advanced — excess property checks, intersections, declaration merging

**Excess property checking (freshness)** was introduced in [Part 2](./typescript-foundations.md#part-2); here is the complete rule, because it confuses everyone at least once.

```ts
interface Options { url: string; timeout?: number }

// ① fresh literal → checked
const a: Options = { url: "/x", retries: 3 };
// ❌ ts(2353): Object literal may only specify known properties, and 'retries'
//    does not exist in type 'Options'.

// ② not fresh → not checked
const raw = { url: "/x", retries: 3 };
const b: Options = raw;                    // ✅

// ③ freshness is lost through a widening assertion
const c: Options = { url: "/x", retries: 3 } as Options;   // ✅ (don't do this)

// ④ spread results are not fresh
const d: Options = { ...raw };             // ✅

// ⑤ arrays of literals ARE checked elementwise
const e: Options[] = [{ url: "/x", retries: 3 }];   // ❌ ts(2353)
```

Why does structural typing permit ②? Because extra properties genuinely don't break anything — `raw` really does satisfy everything `Options` requires. The freshness rule is a **heuristic layered on top** to catch typos in the one situation where extra properties are almost certainly a mistake: you just wrote the literal, so why would you include a property the target doesn't have?

When you legitimately want extra properties, say so in the type rather than reaching for `as`:

```ts
interface Options2 { url: string; timeout?: number; [extra: string]: unknown }
const f: Options2 = { url: "/x", retries: 3 };   // ✅
```

**Intersections (`&`)** combine shapes. They are not "merge" — they are "must satisfy both":

```ts
type WithId = { id: string };
type WithTimestamps = { createdAt: Date; updatedAt: Date };
type Entity = WithId & WithTimestamps;   // must have all four properties
```

Conflicting members do not error at the declaration; they produce an impossible type at the member level:

```ts
type Weird = { a: string } & { a: number };
declare const w2: Weird;
w2.a;   // string & number  →  never. The type is uninhabitable, but only
        // discovered when you try to create a value.
const attempt: Weird = { a: "x" };
// ❌ ts(2322): Type 'string' is not assignable to type 'never'.
```

`interface X extends Y` differs from `type X = Y & {…}` in exactly this respect: `extends` **errors immediately** on an incompatible member override, while `&` defers to `never`. That is an argument for `extends` when you're modelling a hierarchy.

**Declaration merging** — the interface superpower. Two same-named interfaces in the same scope combine:

```ts
interface Config { host: string }
interface Config { port: number }
const c2: Config = { host: "localhost", port: 5432 };   // both required
```

Conflicting property types are an error; conflicting *methods* become overloads (later declarations sort first). The main real-world use is teaching TypeScript about things a library adds at runtime — `declare global`, `declare module`, augmenting `Express.Request` or `ImportMetaEnv`. That is [Part 11](./typescript-modules-ecosystem.md#part-11) territory.

<a id="p6-mastery"></a>

### Mastery — variance, `Record` vs `Map`, and modelling to make bad states impossible

**Object property variance is covariant, and deliberately unsound for mutable properties:**

```ts
type Animal = { name: string };
type Dog = Animal & { breed: string };

const dogs: Dog[] = [{ name: "Rex", breed: "Lab" }];
const animals: Animal[] = dogs;              // ✅ arrays are covariant
animals.push({ name: "Tweety" });            // ✅ compiles…
dogs[1].breed.toUpperCase();                 // 💥 undefined at runtime
```

TypeScript knows this is unsound and permits it anyway, because the alternative (full variance checking on mutable containers, as in Java's `List<? extends T>`) is too painful for the value. **`readonly T[]` is the sound version** — it has no `push`, so the hole closes:

```ts
const safe: readonly Animal[] = dogs;   // ✅ and nothing can be pushed into it
```

Use `readonly T[]` for parameters by default. It costs nothing and removes an entire class of aliasing bug.

**`Record<K, V>` vs `Map`.** A frequent modelling decision:

```ts
type ByIdRecord = Record<string, User>;   // an object; keys are strings; JSON-friendly
const m = new Map<string, User>();        // a Map; any key type; ordered; has .size
```

| Choose | When |
|---|---|
| `Record<string, V>` / index signature | JSON-shaped data, config, anything serialised. Remember `noUncheckedIndexedAccess`. |
| `Record<'a' \| 'b', V>` | a **fixed, known** key set — the compiler then requires every key, which is the real win |
| `Map<K, V>` | keys aren't strings, insertion order matters, keys are user-controlled (no prototype-pollution surface), or you mutate a lot |

The `Record` with a union key type is underused and genuinely valuable — it gives you exhaustiveness on a lookup table:

```ts
type Status = "idle" | "loading" | "error";
const LABELS: Record<Status, string> = {
  idle: "Ready",
  loading: "Working…",
  // ❌ ts(2741): Property 'error' is missing in type … but required in
  //    type 'Record<Status, string>'.
};
```

Add a member to `Status` and every `Record<Status, …>` in the codebase fails to compile — the object-shaped counterpart to the `assertNever` trick from [Part 4](./typescript-narrowing-functions.md#part-4).

**Modelling: make illegal states unrepresentable.** This is the payoff for everything in Parts 4 and 6 together.

```ts
// ❌ Wrong — 2⁴ = 16 representable states, of which 3 are valid
interface RequestState {
  isLoading: boolean;
  data?: User;
  error?: Error;
  isSuccess: boolean;
}
// nothing prevents { isLoading: true, isSuccess: true, error: … }

// ✅ Right — exactly 3 representable states, and each carries only what it has
type RequestState2 =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: User }
  | { status: "failure"; error: Error };
```

The second version deletes a category of bug rather than detecting it. Every `if (state.isLoading && state.data)` dance in the first version becomes a `switch` the compiler checks for you.

**Try It — the optional-vs-union distinction, made concrete.**

```ts
type A = { data?: string };
type B = { data: string | undefined };

const a1: A = {};                  // ✅
const b1: B = {};                  // ❌ ts(2741): Property 'data' is missing…
const b2: B = { data: undefined }; // ✅

// and with exactOptionalPropertyTypes (Part 3):
const a2: A = { data: undefined }; // ❌ ts(2375)
```

*Expected:* errors on `b1` and, if the flag is on, `a2`. `?` means "the key may be absent"; `| undefined` means "the key is present and may hold undefined". Conflating them is how `Object.keys()` and `JSON.stringify()` end up disagreeing with your types.

<a id="p6-cheatsheet"></a>

### Part 6 cheat sheet

| Need | Write |
|---|---|
| Object shape, extensible/augmentable | `interface X { … }` |
| Union, tuple, function, computed type | `type X = …` |
| Combine shapes | `interface X extends Y` or `type X = Y & Z` |
| Dynamic keys | `{ [k: string]: V }` or `Record<string, V>` |
| Fixed key set, all required | `Record<'a' \| 'b', V>` |
| Immutable property / array | `readonly x: T` / `readonly T[]` |
| Allow extra properties | add `[k: string]: unknown` |
| Deep immutability | `Object.freeze` + a `DeepReadonly` mapped type ([Part 9](./typescript-type-level.md#part-9)) |

| Symptom | Cause |
|---|---|
| `ts(2353)` / `ts(2561)` "may only specify known properties" | fresh-literal excess property check |
| Same object works via a variable but not inline | freshness — working as designed |
| `ts(2411)` on a declared property | incompatible with the index signature |
| Property is `never` | conflicting members in an intersection |
| `readonly` didn't stop a mutation | it's shallow, and erased at runtime |
| Union type "not assignable" to a `Record` key | the key union changed — good, that's the point |

[↑ Back to top](#table-of-contents)

---

<a id="part-7"></a>

## Part 7 — Classes

> **Tier note:** the JavaScript half of classes — prototypes, `super`, `#private` semantics, static blocks — lives in [the JavaScript set](../javascript/javascript-objects-classes.md) and is not repeated. This Part covers only what *TypeScript* adds.

<a id="p7-beginner"></a>

### Beginner — fields, constructors, methods

```ts
class Point {
  x: number;                 // field declarations are required — TS won't infer them
  y: number;

  constructor(x: number, y: number) {
    this.x = x;
    this.y = y;
  }

  distanceTo(other: Point): number {
    return Math.hypot(this.x - other.x, this.y - other.y);
  }
}
```

Fields can be initialised inline, and their types inferred from the initialiser:

```ts
class Counter {
  count = 0;               // inferred: number
  readonly created = new Date();
  static instances = 0;
}
```

<a id="p7-working"></a>

### Working Knowledge — modifiers, `implements`, `abstract`

**Parameter properties** are TypeScript-only sugar that declares and assigns a field from a constructor parameter:

```ts
class User {
  constructor(
    public readonly id: string,
    private name: string,
    protected role: "admin" | "user" = "user",
  ) {}
}
// equivalent to declaring three fields and assigning all three in the body
```

Convenient — but note that this syntax **emits runtime code** (the assignments), so it is banned by [`erasableSyntaxOnly`](./typescript-foundations.md#part-3) and rejected by Node's native type stripping. In a project that runs `.ts` directly, write the fields out longhand.

**Visibility modifiers:**

| Modifier | Visible in | Enforced at runtime? |
|---|---|---|
| `public` (default) | everywhere | n/a |
| `protected` | the class and subclasses | ❌ compile-time only |
| `private` | the class only | ❌ compile-time only |
| `#field` (JavaScript) | the class only | ✅ genuinely private |
| `readonly` | assignable in the constructor only | ❌ compile-time only |
| `static` | on the constructor, not instances | ✅ |

**`implements` checks, it doesn't add.** This is the most common class-related misunderstanding:

```ts
interface Serializable { serialize(): string }

class Doc implements Serializable {
  serialize() { return "{}" }
}

// ❌ implements does NOT give you the interface's members or infer parameter types
interface Handler { handle(e: MouseEvent): void }
class H implements Handler {
  handle(e) { }   // ts(7006): Parameter 'e' implicitly has an 'any' type.
}                 // implements does not contextually type parameters!
```

And because typing is structural, `implements` is **optional**: `Doc` would satisfy `Serializable` with or without it. Use it as a documented assertion that gives you an error at the class rather than at some distant call site.

**`abstract`** classes cannot be instantiated and may declare members without implementations:

```ts
abstract class Shape {
  abstract area(): number;                    // subclasses must implement
  describe(): string { return `area ${this.area()}` }   // shared behaviour
}

class Circle extends Shape {
  constructor(private r: number) { super() }
  area() { return Math.PI * this.r ** 2 }
}

new Shape();   // ❌ ts(2511): Cannot create an instance of an abstract class.
```

<a id="p7-advanced"></a>

### Advanced — `private` vs `#private`, `override`, initialization

**`private` is a compile-time fiction; `#private` is real.** The difference matters more than style:

```ts
class Wallet {
  private balanceA = 100;
  #balanceB = 100;
}

const w = new Wallet();
(w as any).balanceA;        // 100 — right there in the emitted object
(w as any).balanceB;        // undefined — genuinely inaccessible
JSON.stringify(w);          // {"balanceA":100} — `private` fields still serialise!
```

That last line is a real security-adjacent bug: a `private` field named `passwordHash` is in your JSON responses. `#` fields are excluded from `JSON.stringify`, `Object.keys`, and `for…in`.

There is also a typing consequence — **`private` participates in assignability, and makes structural typing partly nominal**:

```ts
class A { private secret = 1 }
class B { private secret = 1 }
let a: A = new B();
// ❌ ts(2322): Types have separate declarations of a private property 'secret'.
```

Two identically-shaped classes are *not* interchangeable once a `private` member is involved — TypeScript compares the declaration site, not the shape. This is occasionally exploited deliberately as a poor man's branding, but [the `unique symbol` brand from Part 1](./typescript-foundations.md#part-1) is clearer.

**Recommendation:** prefer `#private` for genuine privacy, `private` when you only want a compile-time convention and need the field to be visible to tests or serialisation. Note that `#` is standard JavaScript and survives erasure, which is why it works under `erasableSyntaxOnly`.

**`override` and `noImplicitOverride`:**

```ts
class Base { greet() { return "hi" } }

class Child extends Base {
  override greet() { return "hello" }    // documented, and checked
  override gret() { return "typo" }
  // ❌ ts(4113): This member cannot have an 'override' modifier because it is
  //    not declared in the base class 'Base'.
}
```

With `noImplicitOverride: true` (not in `strict` — turn it on), omitting `override` on an overriding member is an error. That catches the reverse bug: a base class *removes* a method and your subclass silently becomes a new, uncalled method.

**`strictPropertyInitialization`** (in `strict`) requires every non-optional field to be definitely assigned by the end of the constructor:

```ts
class Service {
  client: Client;
  // ❌ ts(2564): Property 'client' has no initializer and is not definitely
  //    assigned in the constructor.
}

// Four honest fixes, in order of preference:
class S1 { client: Client = createClient() }                      // initialise
class S2 { constructor(public client: Client) {} }                // inject
class S3 { client?: Client }                                      // model it as optional
class S4 { client!: Client }                                      // definite assignment
```

`!` (the definite assignment assertion) is a promise to the compiler that something else assigns it — a DI framework, a lifecycle hook. Like every assertion, it's a place a bug can hide; use it last.

<a id="p7-mastery"></a>

### Mastery — decorators, and when not to use a class at all

**Decorators.** TypeScript 5.0 shipped the **standard** (TC39 stage 3) decorators, which are a different feature from the old `experimentalDecorators` ones despite sharing syntax. This distinction is a live source of confusion, because most decorator tutorials and every Angular/NestJS/TypeORM codebase you've seen use the *legacy* flavour.

```ts
// Standard decorators (TS 5.0+, no flag required) — a function with a
// (value, context) signature
function logged<This, Args extends unknown[], Return>(
  target: (this: This, ...args: Args) => Return,
  context: ClassMethodDecoratorContext<This, typeof target>,
) {
  return function (this: This, ...args: Args): Return {
    console.log(`calling ${String(context.name)}`);
    return target.call(this, ...args);
  };
}

class Api {
  @logged
  fetchUser(id: string) { return { id } }
}
```

| | Legacy (`experimentalDecorators: true`) | Standard (default since 5.0) |
|---|---|---|
| Signature | `(target, key, descriptor)` | `(value, context)` |
| Parameter decorators | ✅ | ❌ not yet in the standard |
| Metadata | `emitDecoratorMetadata` + `reflect-metadata` | `context.metadata` (Symbol.metadata) |
| Used by | Angular, NestJS, TypeORM, class-validator | new code |
| Runtime-erasable? | ❌ | ❌ — both emit code |

**Practical guidance:** if you're using a framework built on legacy decorators, keep `experimentalDecorators: true` and don't mix the two in one project. If you're writing new code that doesn't need a DI container, decorators are rarely worth the complexity — a higher-order function does the same job with better inference. And note that **neither flavour is erasable**, so a project using decorators cannot run `.ts` files directly under Node ([Part 12](./typescript-modules-ecosystem.md#part-12)); `tsx` handles them, plain `node` does not.

**When not to use a class.** TypeScript makes the alternative genuinely better than it is in most languages:

```ts
// Class-based — needed if you have instance state + polymorphism + identity
class Rectangle {
  constructor(private w: number, private h: number) {}
  area() { return this.w * this.h }
}

// Data + functions — usually better in TypeScript, because unions and
// exhaustiveness checking do the polymorphism for you (Part 4)
type Shape =
  | { kind: "rect"; w: number; h: number }
  | { kind: "circle"; r: number };

const area = (s: Shape): number =>
  s.kind === "rect" ? s.w * s.h : Math.PI * s.r ** 2;
```

The union version serialises cleanly, works with structural typing, gets exhaustiveness checking when you add a shape, and needs no `instanceof`. Reach for a class when you need **identity, encapsulated mutable state, or `instanceof`-based runtime dispatch** — the last one being the thing unions genuinely cannot do, since interfaces don't survive erasure.

**Real Scenario — the `private` field in the API response.** A team shipped `class SessionUser { private passwordHash: string; … }` and returned instances directly from an Express handler. `res.json(user)` serialises own enumerable properties, and `private` is erased — so every login response included the hash. The type system reported nothing, because `private` is a rule about *your source code*, not about the object. The fix was `#passwordHash` (excluded from `JSON.stringify`) plus an explicit DTO mapping at the boundary. Compile-time visibility modifiers are not a security boundary; the serialisation boundary is.

<a id="p7-cheatsheet"></a>

### Part 7 cheat sheet

| Feature | Syntax | Erasable? | Runtime-enforced? |
|---|---|---|---|
| Field declaration | `x: number` | ✅ | — |
| Parameter property | `constructor(private x: number)` | ❌ | — |
| Public/protected/private | `private x` | ✅ | ❌ |
| True private | `#x` | ✅ (standard JS) | ✅ |
| Readonly | `readonly x` | ✅ | ❌ |
| Abstract | `abstract class` / `abstract m()` | ✅ | partially (`new` throws) |
| Implements | `class C implements I` | ✅ | ❌ |
| Override | `override m()` | ✅ | ❌ |
| Definite assignment | `x!: T` | ✅ | ❌ |
| Enum | `enum E { … }` | ❌ | ✅ |
| Decorator | `@dec` | ❌ | ✅ |

| Symptom | Fix |
|---|---|
| `ts(2564)` no initializer | initialise, inject, make optional, or `!` |
| `ts(7006)` in an `implements` class | annotate — `implements` doesn't contextually type |
| `ts(2511)` cannot instantiate | it's `abstract` |
| `ts(4113)` bad `override` | the base has no such member (probably a typo or a removal) |
| Two identical classes not assignable | one has a `private`/`protected` member — declaration-site compared |
| `private` field leaked in JSON | use `#`, or map to a DTO at the boundary |
| Decorators error under `erasableSyntaxOnly` | expected — decorators emit code |

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 8 — Generics](./typescript-generics.md#part-8).

*Part of the [TypeScript doc set](./typescript-mastery-guide.md) in [my-learning-md-docs](../README.md). Last updated: September 6, 2026.*
