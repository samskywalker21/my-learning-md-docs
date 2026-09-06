# TypeScript — Narrowing & Functions (Parts 4–5)

> **About this document:** this is one of the sub-docs in this repo's TypeScript set. The canonical spec — framing, scope, tier policy, sourcing rule, versions, and how to update it — lives once in [`typescript-mastery-guide.md`](./typescript-mastery-guide.md#about-this-document). Written against **TypeScript 7.0**, September 6, 2026. Assumes the [JavaScript doc set](../javascript/javascript-mastery-guide.md).

Part 4 is where TypeScript stops feeling like an obstacle. Part 2 gave you unions; narrowing is how you *use* them. Part 5 covers functions, which is where inference, variance, and overloads all show up at once.

---

## Table of Contents

- [Part 4 — Narrowing & Control-Flow Analysis](#part-4)
  - [Beginner — the built-in guards](#p4-beginner)
  - [Working Knowledge — discriminated unions](#p4-working)
  - [Advanced — type predicates, assertion functions, exhaustiveness](#p4-advanced)
  - [Mastery — how control-flow analysis actually works, and where it gives up](#p4-mastery)
  - [Part 4 cheat sheet](#p4-cheatsheet)
- [Part 5 — Functions](#part-5)
  - [Beginner — parameters, returns, optionals, defaults](#p5-beginner)
  - [Working Knowledge — function types, callbacks, `this`](#p5-working)
  - [Advanced — variance, overloads, and why they're usually wrong](#p5-advanced)
  - [Mastery — contextual typing and the shape of inference](#p5-mastery)
  - [Part 5 cheat sheet](#p5-cheatsheet)

---

<a id="part-4"></a>

## Part 4 — Narrowing & Control-Flow Analysis

<a id="p4-beginner"></a>

### Beginner — the built-in guards

**Narrowing** is TypeScript reading your `if` statements and concluding that inside a branch, a value has a more specific type than it did outside. The [Handbook's Narrowing chapter](https://www.typescriptlang.org/docs/handbook/2/narrowing.html) is the reference; this section is the working version.

```ts
function pad(value: string | number) {
  if (typeof value === "number") {
    return " ".repeat(value);   // value: number   ← narrowed
  }
  return value.toUpperCase();   // value: string   ← narrowed by elimination
}
```

The checker follows control flow the way you do. After the `if` returns, the only remaining possibility is `string`, so that's what `value` is.

The guards it understands out of the box:

```ts
declare const v: string | number | null | string[] | Date | undefined;

if (typeof v === "string") {}       // typeof — primitives only
if (v) {}                           // truthiness — removes null, undefined, "", 0, NaN
if (v === null) {}                  // equality against a literal or null/undefined
if (v != null) {}                   // != null removes BOTH null and undefined (JS coercion)
if (Array.isArray(v)) {}            // built-in predicates from lib.d.ts
if (v instanceof Date) {}           // instanceof — classes only
if (typeof v === "object" && v && "length" in v) {}   // the `in` operator
```

**The truthiness trap**, which is a real bug, not a nitpick:

```ts
// ❌ Wrong — 0 and "" are falsy, so valid values take the "missing" path
function setLimit(limit?: number) {
  if (!limit) return DEFAULT;   // limit === 0 silently becomes DEFAULT
  return limit;
}

// ✅ Right — ask the question you actually mean
function setLimit2(limit?: number) {
  if (limit === undefined) return DEFAULT;
  return limit;
}
// or: return limit ?? DEFAULT;   // ?? only falls through for null/undefined
```

**Try It.**

```ts
function f(x: string | number | undefined) {
  if (typeof x === "string" || typeof x === "number") {
    x;   // hover: string | number
  } else {
    x;   // hover: undefined
  }
  x.toString();
}
```

*Expected:* the two hovers as commented, and on the last line `Error: 'x' is possibly 'undefined'. ts(18048)` — narrowing inside the `if` does not survive past it.

<a id="p4-working"></a>

### Working Knowledge — discriminated unions

The single most valuable pattern in applied TypeScript. A **discriminated (tagged) union** is a union whose members all carry a common property holding a distinct literal type. That property is a real, runtime-visible value — which is exactly what [Part 1](./typescript-foundations.md#part-1) said you need, since types themselves don't survive erasure.

```ts
type Result =
  | { status: "success"; data: string }
  | { status: "error"; error: Error }
  | { status: "loading" };

function render(r: Result) {
  switch (r.status) {
    case "success": return r.data;          // r: { status: "success"; data: string }
    case "error":   return r.error.message; // r: { status: "error"; error: Error }
    case "loading": return "…";
  }
}
```

One check on `r.status` narrowed the whole object, including which *other* properties exist. That is the payoff.

Compare with the shape people reach for first, which is worse in every way:

```ts
// ❌ Wrong — optional properties instead of a discriminant
type BadResult = {
  status: string;
  data?: string;
  error?: Error;
};
function renderBad(r: BadResult) {
  if (r.status === "success") {
    return r.data.toUpperCase();
    // Error: 'r.data' is possibly 'undefined'. ts(18048)
    // The type system has no idea that status:"success" implies data exists.
  }
}
```

The bad version also permits nonsense states — `{ status: "success", error: new Error() }` — which the union simply cannot represent. **Making illegal states unrepresentable** is the design goal here, and discriminated unions are the main tool for it.

Rules for the discriminant, learned the hard way:

```ts
// ❌ won't discriminate — the literal widened to `string` (see Part 2, widening)
const ok = { status: "success", data: "x" };
const r: Result = ok;   // ts(2322): Type 'string' is not assignable to type '"success"'…

// ✅ options
const ok2 = { status: "success", data: "x" } as const;
const ok3: Result = { status: "success", data: "x" };   // annotate the target
```

The discriminant must be a **literal type** (string, number, or boolean literal), present on **every** member, and each member's value must be distinct.

<a id="p4-advanced"></a>

### Advanced — type predicates, assertion functions, exhaustiveness

**Type predicates** teach the checker about a check it can't see through on its own. A function returning `x is T` narrows its argument at every call site:

```ts
type Fish = { swim: () => void };
type Bird = { fly: () => void };

function isFish(pet: Fish | Bird): pet is Fish {
  return "swim" in pet;
}

declare const pet: Fish | Bird;
if (isFish(pet)) pet.swim();
else pet.fly();
```

The predicate is **unchecked** — TypeScript trusts your `is` claim the same way it trusts `as`. `return "fly" in pet;` inside `isFish` would compile and lie to every caller. Keep predicate bodies short and obviously correct.

Since **TypeScript 5.5**, many predicates are [inferred automatically](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html) — a function with no explicit return type, a single `return`, no implicit returns, and a boolean expression refining a parameter now infers `x is T` on its own:

```ts
// TS 5.5+ infers `(x: string | null) => x is string`
const isNonNull = (x: string | null) => x !== null;

const values = ["a", null, "b"];
const clean = values.filter(isNonNull);   // string[]  ← used to be (string | null)[]
```

That is why `.filter(Boolean)` advice from older blog posts (and the `as string[]` casts that followed it) is largely obsolete. Prefer `.filter(x => x !== null)` and let inference do the work — though note that the inference only fires under those specific conditions, so an explicit predicate is still the reliable choice for anything more complex.

**Assertion functions** narrow by *throwing* rather than returning:

```ts
function assertIsString(x: unknown): asserts x is string {
  if (typeof x !== "string") throw new TypeError("expected string");
}

function useIt(x: unknown) {
  assertIsString(x);
  x.toUpperCase();      // x: string, for the rest of the scope
}

function assertDefined<T>(x: T): asserts x is NonNullable<T> {
  if (x == null) throw new Error("unexpected nullish value");
}
```

| Use | When |
|---|---|
| `x is T` predicate | you want to branch — both outcomes are valid |
| `asserts x is T` | a failure is a bug; you want to stop, and narrow everything after |

One sharp edge: **an assertion function must have an explicit type annotation on the variable holding it.** `const assertIsString = (x: unknown): asserts x is string => {…}` fails with `ts(2775)` — "Assertions require every name in the call target to be declared with an explicit type annotation." Use a `function` declaration and the problem disappears.

**Exhaustiveness checking with `never`** — the highest-value five lines in this doc set:

```ts
function render(r: Result): string {
  switch (r.status) {
    case "success": return r.data;
    case "error":   return r.error.message;
    case "loading": return "…";
    default: {
      const _exhaustive: never = r;   // ✅ compiles today
      throw new Error(`Unhandled: ${JSON.stringify(_exhaustive)}`);
    }
  }
}
```

In the `default` branch every case is handled, so `r` has narrowed to `never`, and assigning `never` to `never` is fine. Now add a fourth member to `Result`:

```ts
type Result = /* … */ | { status: "cancelled" };
// Error in render(): Type '{ status: "cancelled"; }' is not assignable
//   to type 'never'. ts(2322)
```

**Every `switch` over that union across your entire codebase now fails to compile, pointing exactly at the code that needs updating.** That is compile-time refactoring safety, and it is the reason to prefer discriminated unions over strings-and-optionals in the first place.

A reusable helper is worth keeping around:

```ts
export function assertNever(x: never, msg = "Unexpected variant"): never {
  throw new Error(`${msg}: ${JSON.stringify(x)}`);
}
// default: return assertNever(r);
```

<a id="p4-mastery"></a>

### Mastery — how control-flow analysis actually works, and where it gives up

The checker builds a **control-flow graph** and walks it, maintaining a narrowed type for each reference at each node. Understanding the walk explains every "why did my narrowing vanish" question.

```text
  let x: string | number | undefined

           ┌─────────────────────────┐
           │ declaration             │  x: string | number | undefined
           └───────────┬─────────────┘
                       │
           ┌───────────▼─────────────┐
           │ if (typeof x === "string")
           └─────┬─────────────┬─────┘
          true   │             │  false
           ┌─────▼──────┐  ┌───▼─────────────┐
           │ x: string  │  │ x: number | undefined │
           └─────┬──────┘  └───┬─────────────┘
                 └──────┬──────┘
                 ┌──────▼──────┐
                 │ JOIN        │  x: string | number | undefined   ← union of branches
                 └──────┬──────┘
                        │
             ┌──────────▼───────────┐
             │ ANY assignment to x  │  → narrowing resets to the DECLARED type,
             └──────────────────────┘     re-narrowed from the assigned value
```

Three consequences, each a common frustration:

**1. Narrowing is per-*reference*, and only for references the checker can prove are stable.**

```ts
declare const obj: { a: string | null };

if (obj.a !== null) {
  obj.a.toUpperCase();     // ✅ property access paths ARE tracked
}

declare const arr: (string | null)[];
declare const i: number;
if (arr[i] !== null) {
  arr[i].toUpperCase();    // ❌ ts(18047) — an element access with a variable index
}                          //    is not a stable reference; `i` could change
```

Fix by pulling the value into a `const`:

```ts
const item = arr[i];
if (item !== null) item.toUpperCase();   // ✅
```

**2. Narrowing is lost inside callbacks**, because the checker cannot know *when* the callback runs.

```ts
class Store {
  data: string | null = null;

  load() {
    if (this.data !== null) {
      setTimeout(() => {
        this.data.toUpperCase();
        // ❌ ts(18047): 'this.data' is possibly 'null'.
      });
    }
  }
}
```

This is correct, not pedantic: something else could set `this.data = null` before the timer fires. The two standard fixes:

```ts
// ✅ A — capture into a const; the const cannot change, so the narrowing holds
load2() {
  const data = this.data;
  if (data !== null) setTimeout(() => data.toUpperCase());
}

// ✅ B — re-check inside the callback
load3() {
  setTimeout(() => { if (this.data !== null) this.data.toUpperCase(); });
}
```

The same rule explains why a `let` narrowed before an `await` may be widened after it, and why mutable module-level state is hostile to narrowing. **`const` is a type-system tool, not just a style preference.**

**3. Aliased conditions work — since TypeScript 4.4, within limits.**

```ts
declare const value: string | number;
const isString = typeof value === "string";   // must be `const`, and value must be `const`
if (isString) {
  value.toUpperCase();   // ✅ TypeScript follows the aliased discriminant
}
```

Make `isString` a `let`, or `value` a `let`, and it stops working.

**Narrowing generics is the last hard case.** Narrowing a value of type `T` narrows the *value*, not `T` itself:

```ts
function process<T>(x: T) {
  if (typeof x === "string") {
    x;   // T & string   ← an intersection, not `string`
  }
}
```

`T & string` is usually what you want, but it means you can't return `x` as a plain `string` without an assertion. This is one of the places where generics and narrowing rub against each other — [Part 8](./typescript-generics.md#part-8) covers the rest.

<a id="p4-cheatsheet"></a>

### Part 4 cheat sheet

| Guard | Narrows | Notes |
|---|---|---|
| `typeof x === "string"` | primitives | `"object"` includes `null` — check separately |
| `x === null` / `x != null` | nullish | `!= null` removes both `null` and `undefined` |
| `if (x)` | truthy | drops `0`, `""`, `NaN` too — usually a bug |
| `Array.isArray(x)` | arrays | a built-in type predicate |
| `x instanceof C` | classes | not interfaces |
| `"k" in x` | object shapes | good for un-discriminated unions |
| `x.kind === "a"` | discriminated unions | the workhorse |
| `isFoo(x)` (`x is Foo`) | anything you can test | unchecked — you take responsibility |
| `assertFoo(x)` (`asserts x is Foo`) | anything, by throwing | needs a `function` declaration |

| Problem | Fix |
|---|---|
| Narrowing lost in a callback | capture into a `const`, or re-check inside |
| Narrowing lost after `await` / assignment | use `const`; avoid mutable narrowed state |
| `arr[i]` won't narrow | assign to a `const` first |
| Union member added, nothing broke | add an `assertNever` default branch |
| `.filter(x => x !== null)` still `(T \| null)[]` | TS 5.5+ infers it; if not, write an explicit predicate |
| Optional props instead of a discriminant | restructure into a tagged union |

[↑ Back to top](#table-of-contents)

---

<a id="part-5"></a>

## Part 5 — Functions

<a id="p5-beginner"></a>

### Beginner — parameters, returns, optionals, defaults

```ts
function add(a: number, b: number): number {
  return a + b;
}

const sub = (a: number, b: number): number => a - b;
```

**Annotate parameters; usually let the return type infer.** The exception is exported API surface, where an explicit return type is a contract — it stops an internal refactor from silently widening what callers see, and it makes errors point at the function rather than at some distant call site.

```ts
function greet(name: string, greeting = "Hello"): string {    // default → optional
  return `${greeting}, ${name}`;
}

function log(msg: string, level?: "warn" | "error"): void {   // optional → `| undefined`
  console.log(level ?? "info", msg);
}

function sum(...nums: number[]): number {                     // rest
  return nums.reduce((a, b) => a + b, 0);
}
```

```ts
// ❌ Wrong — optional parameters must come last
function bad(a?: string, b: number) {}
// Error: A required parameter cannot follow an optional parameter. ts(1016)

// ✅ Right — reorder, or take an options object
function good(b: number, a?: string) {}
function good2(opts: { b: number; a?: string }) {}
```

<a id="p5-working"></a>

### Working Knowledge — function types, callbacks, `this`

A **function type** describes a callable value. There are two syntaxes, and the difference matters:

```ts
type Handler = (event: string) => void;          // shorthand, most common

type Handler2 = {                                 // call-signature form:
  (event: string): void;                          //   callable…
  displayName: string;                            //   …AND has properties
};

type Ctor = new (name: string) => Person;         // construct signature
```

**Callbacks and the `void`-return rule.** A function returning anything is assignable to a function type returning `void`, which is what makes this everyday code compile:

```ts
const src = [1, 2, 3];
const dst: number[] = [];
src.forEach(n => dst.push(n));   // ✅ push returns number; forEach expects void
```

The corollary bites when you *define* a callback type as returning `void` and then rely on the value:

```ts
// ❌ Wrong — declares void, so callers may return anything and you can't use it
type Validate = (v: string) => void;
declare const validate: Validate;
if (validate("x")) {}   // Error: An expression of type 'void' cannot be tested
                        //   for truthiness. ts(1345)

// ✅ Right — say what you actually need back
type Validate2 = (v: string) => boolean;
```

**Typing `this`.** A fake first parameter named `this` types the receiver and is erased on emit (see [`this` in the JavaScript set](../javascript/javascript-functions-closures.md)):

```ts
interface Button {
  label: string;
  onClick(this: Button, e: MouseEvent): void;
}

// Standalone functions can declare it too:
function reset(this: HTMLFormElement) { this.reset(); }
```

With `strictFunctionTypes` and `noImplicitThis` (both in `strict`), this catches the classic detached-method bug:

```ts
const btn: Button = { label: "OK", onClick() { console.log(this.label); } };
const cb = btn.onClick;
cb(new MouseEvent("click"));
// Error: The 'this' context of type 'void' is not assignable to method's
//   'this' of type 'Button'. ts(2684)
```

<a id="p5-advanced"></a>

### Advanced — variance, overloads, and why they're usually wrong

**Variance** is the rule for when one function type is assignable to another. Two halves, and the second is the counter-intuitive one:

```text
   Is  (a: A) => R   assignable to   (b: B) => S  ?

   RETURNS  — covariant:      R must be assignable to S
                              "I may return something MORE specific"

   PARAMS   — contravariant:  B must be assignable to A
                              "I must accept something MORE general"
```

```ts
type Animal = { name: string };
type Dog = Animal & { breed: string };

// Returns: covariant ✅
const makeDog: () => Dog = () => ({ name: "Rex", breed: "Lab" });
const makeAnimal: () => Animal = makeDog;      // ✅ a Dog IS an Animal

// Params: contravariant ✅
const handleAnimal = (a: Animal) => a.name;
const handleDog: (d: Dog) => string = handleAnimal;  // ✅ handles any Animal, so any Dog
const handleAnimal2: (a: Animal) => string = (d: Dog) => d.breed;
// ❌ ts(2322): Type '(d: Dog) => string' is not assignable to type '(a: Animal) => string'.
//    Types of parameters 'd' and 'a' are incompatible.
```

The intuition: a handler that needs a `breed` cannot be used where any `Animal` may arrive. This is what `strictFunctionTypes` enforces — **for function-type positions only**. Methods declared with method syntax are deliberately checked **bivariantly** (both directions allowed), a documented unsoundness kept because `Array<Dog>` being usable as `Array<Animal>` depends on it:

```ts
interface A { f(x: Dog): void }        // method syntax → bivariant, unsound but convenient
interface B { f: (x: Dog) => void }    // property syntax → contravariant, sound
```

If you want the strict check, declare callbacks as **properties**, not methods. That one-character difference is a real design decision in an interface.

**Overloads.** Multiple signatures for one implementation:

```ts
function parse(input: string): object;
function parse(input: string, reviver: (k: string, v: unknown) => unknown): object;
function parse(input: string, reviver?: (k: string, v: unknown) => unknown): object {
  return JSON.parse(input, reviver);
}
```

The implementation signature is **not callable** — only the overloads above it are. And the implementation's types are not checked against callers, only against the overloads, which makes overloads a quiet source of unsoundness.

**Most overloads should be a union or a generic instead.** The Handbook's [own guidance](https://www.typescriptlang.org/docs/handbook/2/functions.html) is to prefer union parameters where possible:

```ts
// ❌ Overloads that add nothing
function len(x: string): number;
function len(x: unknown[]): number;
function len(x: string | unknown[]): number { return x.length; }

// ✅ Just a union
function len2(x: string | unknown[]): number { return x.length; }
```

Overloads earn their place only when the return type genuinely **depends on** the argument types in a way a union cannot express — and even then a generic with a conditional return type ([Part 9](./typescript-type-level.md#part-9)) is often cleaner.

The classic overload failure:

```ts
declare function f(x: string): string;
declare function f(x: number): number;

declare const v: string | number;
f(v);
// ❌ ts(2769): No overload matches this call.
// Overload resolution picks ONE signature; it never unions them.
```

<a id="p5-mastery"></a>

### Mastery — contextual typing and the shape of inference

**Contextual typing** is inference running *backwards*: the expected type of an expression supplies types to the expression's interior. It is why you almost never annotate callback parameters:

```ts
const nums = [1, 2, 3];
nums.map(n => n * 2);        // n: number, inferred from Array<number>.map's signature

window.addEventListener("click", e => {
  e.clientX;                 // e: MouseEvent — from the "click" literal overload
});

// But context has to come from somewhere:
const f = (n) => n * 2;
// ❌ ts(7006): Parameter 'n' implicitly has an 'any' type.

const g: (n: number) => number = (n) => n * 2;    // ✅ context from the annotation
```

**The order matters, and this is the part worth internalising.** TypeScript infers a call's type arguments **left to right**, and a later argument can be contextually typed by what earlier arguments produced:

```ts
declare function pipe<A, B, C>(a: A, ab: (a: A) => B, bc: (b: B) => C): C;

pipe(
  "hello",
  s => s.length,      // s: string  (from A), returns number → B = number
  n => n > 3          // n: number  (from B), returns boolean → C = boolean
);                    // result: boolean
```

Reorder the parameters so the function comes first and the whole thing collapses to `unknown`. When a generic helper "won't infer", the fix is very often to move the value argument earlier in the parameter list.

**Return-type inference and the `never` surprise:**

```ts
function fail(msg: string) { throw new Error(msg); }        // inferred: void  (!)
function fail2(msg: string): never { throw new Error(msg); } // explicit: never
```

A function whose body only throws infers `void`, not `never`, unless you annotate it — a deliberate compatibility decision. If you want `fail()` usable in expression position (`const x = cond ? v : fail("…")`), annotate `never`.

**Try It — watch inference get it wrong, then fix it.**

```ts
function firstOrDefault<T>(arr: T[], fallback: T): T {
  return arr.length ? arr[0]! : fallback;
}

const a = firstOrDefault([1, 2], 0);          // number
const b = firstOrDefault([], "x");            // string  — T inferred from fallback
const c = firstOrDefault([1, 2], "x");        // ???
```

*Expected:* `c` gives `Error: Type 'string' is not assignable to type 'number'. ts(2345)` — with two inference sites for `T`, TypeScript picks the best common supertype and reports the conflict rather than silently widening to `number | string`. Adding a second type parameter (`<T, F>`) and returning `T | F` is how you'd allow it deliberately. Inference sites are covered fully in [Part 8](./typescript-generics.md#part-8).

<a id="p5-cheatsheet"></a>

### Part 5 cheat sheet

| Need | Syntax |
|---|---|
| Function type | `(a: string) => number` |
| Callable with properties | `{ (a: string): number; name: string }` |
| Constructor type | `new (a: string) => Foo` |
| Optional param | `(a: string, b?: number)` |
| Default param | `(a: string, b = 3)` |
| Rest param | `(...xs: number[])` |
| Typed receiver | `function f(this: Ctx, …)` |
| Never returns | `function f(): never` |
| Async | `async function f(): Promise<T>` |
| Generator | `function* g(): Generator<Y, R, N>` |

| Rule | Statement |
|---|---|
| Return types | **covariant** — you may return something more specific |
| Parameter types | **contravariant** — you must accept something more general |
| Method syntax `f(x: T): void` | bivariant (deliberately unsound; enables `Array<Dog>` → `Array<Animal>`) |
| Property syntax `f: (x: T) => void` | contravariant (sound; use it when you want the check) |
| Fewer params | assignable to a type with more (`() => void` fits `(a, b) => void`) |
| `=> void` return | any return value is accepted and discarded |
| Overload resolution | picks one signature; never unions them |
| Contextual typing | flows left to right through arguments |

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 6 — Object Types](./typescript-objects-classes.md#part-6).

*Part of the [TypeScript doc set](./typescript-mastery-guide.md) in [my-learning-md-docs](../README.md). Last updated: September 6, 2026.*
