# JavaScript Mastery Guide (Pre-TypeScript)

## About This Document

- **Framing:** Goal-driven — learning JavaScript specifically as preparation for TypeScript. Depth and prioritization favor the language-level features TypeScript's type system sits on top of, not general web-dev breadth (DOM/browser APIs excluded by default).
- **Depth tiers requested:** Beginner → Working Knowledge → Advanced → Mastery, applied fully to all sections except [Modern Runtime & Tooling Context](#9-modern-runtime--tooling-context), which stops at Working Knowledge.
- **Confirmed scope:**
  1. Values, Types & Coercion
  2. Variables & Scope
  3. Functions
  4. Objects & Prototypes
  5. Arrays & Iteration
  6. Asynchronous JS
  7. Modules
  8. Equality, Errors & Edge Cases
  9. Modern Runtime & Tooling Context (Beginner/Working Knowledge only)
- **Runtime for examples:** Node.js (current LTS lines as of 2026: Node 22 and Node 24; Node 26 is the "Current" release), with browser differences noted where relevant.
- **Prior knowledge assumed:** General programming fluency (variables, loops, functions in some language), new to JS specifically.
- **Excluded by design:** DOM/browser APIs, styling/CSS, general framework usage — out of scope for this doc; a separate doc can cover them if needed.
- **If you ask me to update this doc later:** keep this structure, tier labels, sourcing style (MDN/official docs + Stack Overflow inline links), and the wrong-vs-right snippet format.

---

## Table of Contents

1. [Values, Types & Coercion](#1-values-types--coercion)
2. [Variables & Scope](#2-variables--scope)
3. [Functions](#3-functions)
4. [Objects & Prototypes](#4-objects--prototypes)
5. [Arrays & Iteration](#5-arrays--iteration)
6. [Asynchronous JS](#6-asynchronous-js)
7. [Modules](#7-modules)
8. [Equality, Errors & Edge Cases](#8-equality-errors--edge-cases)
9. [Modern Runtime & Tooling Context](#9-modern-runtime--tooling-context)
10. [Suggested Learning/Reference Order](#10-suggested-learningreference-order)
11. [Quick Self-Check](#11-quick-self-check)

---

## 1. Values, Types & Coercion

This is the section that predicts most of your future TypeScript "aha" moments — TS's static type system exists almost entirely to catch the bugs this section describes.

### Beginner

JS has 7 primitive types (`string`, `number`, `boolean`, `null`, `undefined`, `bigint`, `symbol`) and one composite type, `object` (arrays, functions, and dates are all objects). Variables aren't typed — values are, and a variable can hold any type at any time.

```js
let x = 5;
x = "now a string"; // perfectly legal in JS, illegal in TS by default
```

Check a type with `typeof` (primitives) or `instanceof` (objects/classes):

```js
typeof "hi";        // "string"
typeof 5;            // "number"
typeof null;          // "object" — a 30-year-old spec bug, not a real object type. See MDN below.
[] instanceof Array;   // true
```

### Working Knowledge

Coercion happens implicitly in comparisons, arithmetic, and template literals. This is where JS earns its reputation.

```js
// Wrong — relying on implicit coercion without understanding it
if (userInput == 0) { /* also true for "", false, null-ish comparisons in some paths */ }

// Right — be explicit
if (Number(userInput) === 0) { ... }
```

Common coercion rules worth internalizing:

| Expression | Result | Why |
|---|---|---|
| `"5" + 1` | `"51"` | `+` with a string operand does string concatenation |
| `"5" - 1` | `4` | `-` always coerces to number |
| `[] + []` | `""` | arrays coerce to strings via `toString()` |
| `[] + {}` | `"[object Object]"` | same, on the object side |
| `Boolean("")`, `Boolean(0)`, `Boolean(null)`, `Boolean(undefined)`, `Boolean(NaN)` | `false` | the only 5 falsy primitives (plus `document.all` in browsers, an intentional legacy exception) |

**Real Scenario:** A form handler checks `if (formData.age)` to validate an age field. A user legitimately enters `0` (age zero, or a quantity field). The falsy check silently treats `0` as "not provided" and the validation branch never runs — a bug that only shows up for that one edge-case input, weeks after ship. Fix: check `formData.age !== undefined` or use `??` (see [Equality, Errors & Edge Cases](#8-equality-errors--edge-cases)).

### Advanced

`NaN` is the only value in JS that is not equal to itself (`NaN !== NaN`), because IEEE-754 defines it that way. Use `Number.isNaN()`, never `x === NaN`.

`typeof null === "object"` is a bug from JS's original 1995 implementation, kept for backwards compatibility — the [ECMAScript spec formally documents it as a historical artifact](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/typeof#typeof_null) rather than fixing it, because fixing it would break the web.

`Symbol` and `BigInt` are the two primitives most JS learners never touch but that TypeScript models explicitly:
- `Symbol()` creates a guaranteed-unique value, commonly used as non-colliding object keys (e.g. `Symbol.iterator`, which powers `for...of` — see [Arrays & Iteration](#5-arrays--iteration)).
- `BigInt` (`10n`) represents arbitrary-precision integers beyond `Number.MAX_SAFE_INTEGER` (2^53 - 1); mixing `BigInt` and `Number` in arithmetic throws a `TypeError` by design, not silently coerces.

### Mastery

Under the hood, `==` (loose equality) follows the [Abstract Equality Comparison algorithm](https://tc39.es/ecma262/#sec-abstract-equality-comparison) in the spec — a specific, memorizable-but-not-worth-memorizing sequence of coercions. The practical takeaway experienced devs converge on: **default to `===`, and only reach for `==` in the one legitimate idiom** `x == null` (which matches both `null` and `undefined` in one check — see [Section 8](#8-equality-errors--edge-cases)).

```js
// wrong vs right for the "is this present" check
if (x !== null && x !== undefined) { ... }   // works, verbose
if (x != null) { ... }                        // idiomatic, intentional use of ==
```

[⤴ back to top](#table-of-contents)

---

## 2. Variables & Scope

### Beginner

Three ways to declare a variable: `var` (function-scoped, legacy), `let` (block-scoped, reassignable), `const` (block-scoped, not reassignable — but *mutable* if it holds an object).

```js
const arr = [1, 2, 3];
arr.push(4);      // fine — you're mutating the object, not reassigning the binding
arr = [5, 6];       // TypeError: Assignment to constant variable.
```

### Working Knowledge

**Hoisting**: declarations are processed before code runs, but *how* differs by keyword.

```js
console.log(a); // undefined (var is hoisted and initialized to undefined)
var a = 1;

console.log(b); // ReferenceError: Cannot access 'b' before initialization
let b = 1;
```

`let`/`const` are hoisted too, but land in the **Temporal Dead Zone (TDZ)** — accessible in scope but not usable until the declaration line executes. This is why `let`/`const` catch a whole class of "used before defined" bugs `var` silently allows.

**Real Scenario — the classic loop-closure bug**, notorious enough to be one of the most common JS interview questions ([explained on Stack Overflow](https://stackoverflow.com/questions/750486/how-do-i-make-jquery-wait-for-an-ajax-call-to-finish-before-it-returns) and widely documented):

```js
// Wrong
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// logs: 3 3 3 — var is function-scoped, so all callbacks share one `i`,
// which has already reached 3 by the time the callbacks run.

// Right
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// logs: 0 1 2 — let creates a fresh binding per iteration
```

### Advanced

**Closures**: a function retains access to its defining scope's variables even after that scope has returned. This is the mechanism behind the loop bug above, and also behind common patterns like module-private state and memoization.

```js
function makeCounter() {
  let count = 0;
  return () => ++count; // closes over `count`
}
const counter = makeCounter();
counter(); // 1
counter(); // 2 — count persisted between calls, invisible from outside
```

Prefer `let`/`const` over `var` in all new code — this is uncontroversial modern practice per [MDN's `var` documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/var), which itself now steers readers toward `let`/`const`.

### Mastery

Every function call creates a new **lexical environment** (a scope object plus a reference to its parent), forming a scope chain resolved at *definition* time, not call time — this is what "lexical scoping" means and why closures work at all. Engines like V8 optimize closures heavily but historically could leak memory when a long-lived closure retained a reference to an otherwise-large enclosing scope; modern V8 does per-variable capture analysis to avoid over-retaining, but it's still a real pattern to watch for in long-running Node processes (e.g., closures inside `setInterval` accumulating references).

[⤴ back to top](#table-of-contents)

---

## 3. Functions

### Beginner

Three syntaxes, same underlying concept — a callable value:

```js
function add(a, b) { return a + b; }        // declaration — hoisted fully
const add2 = function (a, b) { return a + b; }; // expression — not hoisted
const add3 = (a, b) => a + b;                 // arrow — not hoisted, no own `this`
```

### Working Knowledge

Default, rest, and spread parameters:

```js
function greet(name = "world") { return `hi ${name}`; }
function sum(...nums) { return nums.reduce((a, b) => a + b, 0); } // rest: gather
sum(...[1, 2, 3]); // spread: expand
```

**`this` binding** depends on *how* a regular function is called, not where it's defined — arrow functions are the one exception, capturing `this` lexically from their enclosing scope ([MDN: `this`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)).

```js
const obj = {
  name: "widget",
  // Wrong — regular function loses `this` when detached from `obj`
  logLater() { setTimeout(function () { console.log(this.name); }, 0); }, // undefined (or throws in strict mode)
  // Right — arrow function inherits `this` from logLater's scope
  logLaterFixed() { setTimeout(() => console.log(this.name), 0); }, // "widget"
};
```

### Advanced

`call`, `apply`, and `bind` explicitly set `this`:

```js
function greet() { return `hi ${this.name}`; }
greet.call({ name: "Sam" });     // invoke now, this = {name: "Sam"}
greet.apply({ name: "Sam" }, []); // same, but args as array
const bound = greet.bind({ name: "Sam" }); // returns a new function, permanently bound
```

**Real Scenario:** A React-style class component (or any object with methods passed as callbacks) passes `this.handleClick` as an event handler without binding it. Inside the handler, `this` is `undefined` (strict mode) instead of the component instance, and `this.setState` throws. This exact bug is one of the most frequently asked JS/React questions on Stack Overflow, and is why arrow-function class fields (`handleClick = () => {...}`) became the idiomatic fix.

Function objects also carry properties: `fn.length` (declared arity, ignoring rest/default params), `fn.name`, and are themselves instances of `Function`.

### Mastery

Arrow functions are not just "shorter syntax" — they structurally lack their own `this`, `arguments`, `super`, and `new.target`, resolving all of them from the enclosing lexical scope. This means arrow functions **cannot** be used as constructors (`new (() => {})()` throws) and `call`/`apply`/`bind` cannot override their `this`. Understanding this distinction matters directly for TypeScript: TS's `this` parameter typing and arrow-vs-method class field typing both hinge on this exact runtime behavior.

[⤴ back to top](#table-of-contents)

---

## 4. Objects & Prototypes

This section maps most directly onto TypeScript's `class`, `interface`, and structural typing — worth extra attention.

### Beginner

```js
const user = { name: "Sam", age: 30 };
user.name;      // "Sam"
user["age"];    // 30 — bracket notation for dynamic/non-identifier keys
```

### Working Knowledge

Every object has an internal prototype link (`[[Prototype]]`), accessible via `Object.getPrototypeOf()` or the legacy `__proto__`. Property lookups walk this **prototype chain** until found or exhausted.

```js
const animal = { speak() { return "..."; } };
const dog = Object.create(animal); // dog's prototype is animal
dog.speak(); // "..." — found via the chain, not on `dog` itself
```

`class` syntax (ES2015+) is sugar over this same prototype mechanism — it doesn't introduce a new inheritance model, just a cleaner syntax for it ([MDN: Classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes)):

```js
class Animal {
  #energy = 100; // private field (ES2022+), truly inaccessible from outside
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
  get isTired() { return this.#energy < 10; }
}
class Dog extends Animal {
  speak() { return `${super.speak()} — specifically, a bark`; }
}
```

### Advanced

```js
// Wrong — mutating a shared prototype property from an instance assumes it's "copied"
function Counter() {}
Counter.prototype.count = 0;
const c1 = new Counter(), c2 = new Counter();
c1.count++; // this actually creates an OWN property `count` on c1, shadowing the prototype
c2.count;   // still 0 — surprising if you expected shared mutable state

// Right — for genuinely shared mutable state, use a shared object/array, not a primitive default
Counter.prototype.log = [];
c1.log.push("x"); // c2.log now also has "x" — arrays/objects ARE shared by reference
```

`Object.freeze()` prevents property addition/removal/reassignment (shallow only — nested objects remain mutable). Property descriptors (`Object.defineProperty`) let you control `writable`, `enumerable`, `configurable` per-property — this is how getters/setters and private-ish patterns worked before `#private` fields existed.

### Mastery

Private class fields (`#field`, ES2022) are **not** just convention-based privacy (like the old `_field` naming trick) — they're enforced at the language/engine level; accessing `#field` from outside the class throws a `SyntaxError` at parse time, not a runtime warning. This is a genuinely new privacy primitive, and it's why TypeScript's `private` modifier (compile-time only, erased at build) and JS's `#field` (runtime-enforced) behave differently — a distinction that surprises people moving from TS's `private` to real `#private` fields.

Prototype chain lookups have a real performance cost proportional to chain depth; V8 optimizes monomorphic property access (same shape every time) via "hidden classes," but polymorphic access (objects with the same property names added in different orders) can de-optimize hot paths — rarely worth hand-optimizing for, but explains why `class`-based consistent object shapes tend to outperform ad-hoc object literals in hot loops.

[⤴ back to top](#table-of-contents)

---

## 5. Arrays & Iteration

### Beginner

Arrays are objects with numeric-ish keys and a `length` property, plus a large built-in method set.

```js
const nums = [1, 2, 3];
nums.push(4);       // mutates: [1,2,3,4]
nums.map(n => n * 2); // returns new array: [2,4,6,8]
```

### Working Knowledge

Know which methods **mutate** vs **return new**:

| Mutates original | Returns new (leaves original alone) |
|---|---|
| `push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse` | `map`, `filter`, `reduce`, `slice`, `concat`, `toSorted`, `toReversed`, `toSpliced` (ES2023) |

```js
// Wrong — sort() mutates in place; this silently corrupts `original`
const original = [3, 1, 2];
const sorted = original.sort();
original; // [1, 2, 3] — also mutated! Easy to miss in a larger function.

// Right — use the ES2023 non-mutating variant when you need to preserve the source
const sorted2 = original.toSorted();
```

Destructuring and spread:

```js
const [first, ...rest] = [1, 2, 3]; // first=1, rest=[2,3]
const merged = [...arr1, ...arr2];
```

### Advanced

`reduce` is the general-purpose primitive most array methods can be built from — useful to understand once, then mostly avoid overusing for readability's sake:

```js
// Wrong — reduce for something map/filter already expresses more clearly
const doubled = nums.reduce((acc, n) => { acc.push(n * 2); return acc; }, []);
// Right
const doubled2 = nums.map(n => n * 2);
```

**Real Scenario:** `Object.groupBy()` (ES2024, [MDN](https://mdn2.netlify.app/en-us/docs/web/javascript/reference/global_objects/array/groupby/)) replaces a very common `reduce`-based grouping pattern for things like grouping API results by category — cleaner and less error-prone than hand-rolled reduce accumulators:

```js
const inventory = [{type: "fruit", name: "apple"}, {type: "veg", name: "carrot"}];
Object.groupBy(inventory, item => item.type);
// { fruit: [{...apple}], veg: [{...carrot}] }
```

### Mastery

Arrays (and strings, Maps, Sets) are iterable because they implement the **iterator protocol** — an object with a `Symbol.iterator` method returning `{ next() }`. `for...of`, spread, and destructuring all consume this protocol under the hood, which is why you can write custom iterables:

```js
const range = {
  [Symbol.iterator]() {
    let i = 0;
    return { next: () => i < 3 ? { value: i++, done: false } : { value: undefined, done: true } };
  },
};
[...range]; // [0, 1, 2] — spread works because `range` implements the protocol, not because it's "array-like"
```

`for...in` iterates enumerable keys (including inherited ones) and is for objects, not arrays — using it on arrays is a well-known footgun (index order isn't guaranteed, and it picks up inherited/added properties). Prefer `for...of` (values) or `Object.keys/entries` for objects.

[⤴ back to top](#table-of-contents)

---

## 6. Asynchronous JS

The most conceptually load-bearing section for real-world JS — and the one where TypeScript's `Promise<T>` typing will make the most sense once you understand the runtime mechanics.

### Beginner

```js
// Callback (legacy pattern, still seen in older Node APIs)
fs.readFile("file.txt", (err, data) => { ... });

// Promise
fetch(url).then(res => res.json()).then(data => console.log(data));

// async/await — syntax sugar over Promises, not a different mechanism
async function load() {
  const res = await fetch(url);
  const data = await res.json();
  return data;
}
```

### Working Knowledge

The **event loop**: JS is single-threaded; async operations (timers, I/O, promises) don't block — they're queued and run when the call stack is empty.

```
Call Stack (sync code)  →  runs to completion first
        │
        ▼
Microtask Queue (Promise .then/.catch, queueMicrotask)  →  drained fully before next macrotask
        │
        ▼
Macrotask/Task Queue (setTimeout, setInterval, I/O callbacks)  →  one per event loop tick
```

```js
console.log("1");
setTimeout(() => console.log("2"), 0); // macrotask
Promise.resolve().then(() => console.log("3")); // microtask
console.log("4");
// Output: 1, 4, 3, 2 — sync first, then ALL microtasks, then macrotasks
```

### Advanced

```js
// Wrong — try/catch around await looks like it catches everything, but misses
// a promise that starts rejecting before it's awaited
async function load() {
  const p = fetch(url); // starts immediately, not wrapped yet
  doOtherSyncWorkThatThrows(); // if this throws, we never reach the try/catch below
  try {
    await p;
  } catch (e) { ... }
}

// Right — wrap the whole async sequence, including the promise creation
async function load() {
  try {
    const p = fetch(url);
    doOtherSyncWorkThatThrows();
    await p;
  } catch (e) { ... }
}
```

This exact class of bug — a promise created outside a `try` block rejecting before it's `await`-ed — is a [documented gotcha](https://advancedweb.hu/how-to-avoid-uncaught-async-errors-in-javascript/) that produces an `UnhandledPromiseRejection` even though a `try/catch` is present elsewhere in the function.

`Promise.all` (fails fast on first rejection) vs `Promise.allSettled` (waits for all, never rejects) vs `Promise.race` vs `Promise.any` — pick based on whether partial failure is acceptable:

```js
const results = await Promise.allSettled([fetchA(), fetchB()]);
// [{status: "fulfilled", value: ...}, {status: "rejected", reason: ...}]
```

### Mastery

`await` doesn't block the thread — it suspends the `async function`'s execution and schedules the rest as a microtask continuation, returning control to the caller immediately. This is why an `async function` call returns a `Promise` synchronously, even before any `await` inside it resolves.

Top-level `await` is legal in ES modules (not CommonJS) — a module using it will delay its own evaluation, and the evaluation of anything importing it, until the awaited promise settles ([Node.js ESM docs](https://nodejs.org/api/esm.html)).

**Real Scenario:** A Node service issues 500 independent `fetch` calls inside a naive `for` loop with `await` on each — turning what should be concurrent I/O into serial I/O, 500x slower than necessary:

```js
// Wrong — serializes 500 independent network calls
for (const id of ids) { results.push(await fetchUser(id)); }

// Right — fires all requests concurrently, awaits together
const results = await Promise.all(ids.map(fetchUser));
```

[⤴ back to top](#table-of-contents)

---

## 7. Modules

Directly relevant to TypeScript: your `tsconfig.json` `module`/`moduleResolution` settings configure exactly this system.

### Beginner

```js
// math.js
export function add(a, b) { return a + b; }
export default function multiply(a, b) { return a * b; }

// main.js
import multiply, { add } from "./math.js";
```

### Working Knowledge

Two competing systems exist in Node: **CommonJS** (`require`/`module.exports`, the historical default) and **ESM** (`import`/`export`, the current standard for new code). As of 2026, [the ecosystem has largely converged on ESM for new code, while CJS remains widespread in production](https://sandeepbansod.medium.com/esm-vs-cjs-why-your-import-still-breaks-in-2026-and-how-to-finally-fix-it-9a16c318a291) — you'll encounter both.

```js
// CommonJS
const { add } = require("./math.js");
module.exports = { add };

// ESM — requires "type": "module" in package.json, or a .mjs extension
import { add } from "./math.js";
export { add };
```

```js
// Wrong — mixing require() and import in the same file
import fs from "fs";
const path = require("path"); // SyntaxError in an ESM file — require doesn't exist there

// Right — pick one system per file/package; use dynamic import() to load ESM from CJS if needed
import fs from "fs";
import path from "path";
```

### Advanced

Key behavioral differences that bite people:

- **CJS loads synchronously and can be conditional/dynamic** (`require()` anywhere, including inside `if` blocks); **ESM `import` is static** — hoisted, resolved at parse time, top-level only (use `import()` for dynamic/conditional loading, which returns a Promise).
- CJS exports are a live *copy* at require-time by default for primitives; ESM exports are **live bindings** — an imported value updates if the exporting module changes it later, which surprises people expecting a snapshot.
- As of current Node LTS (22/24), `require(esm)` now works — CommonJS files can synchronously `require()` an ESM module in many cases, considerably easing the historical interop pain ([Node.js ESM docs](https://nodejs.org/api/esm.html)).

### Mastery

Module resolution (how a bare specifier like `"lodash"` maps to a file) differs between CJS's `node_modules` folder-walk algorithm and ESM's `exports` map in `package.json`, which lets a package expose different entry points for different conditions (`import`, `require`, `browser`, `types`) — the `types` condition is exactly what makes a package's `.d.ts` files resolve correctly in TypeScript, so this mechanism is worth understanding before you start configuring TS module resolution.

[⤴ back to top](#table-of-contents)

---

## 8. Equality, Errors & Edge Cases

### Beginner

```js
null == undefined;   // true  — the one legitimate use of ==
null === undefined;   // false — different types
```

`null` = "intentionally empty" (you set it); `undefined` = "never set" (the language's default for unset variables, missing object properties, missing function args).

### Working Knowledge

```js
// Wrong
try {
  doWork();
} catch (e) {
  console.log("error"); // swallows the error, loses the stack trace, hides the real bug
}

// Right
try {
  doWork();
} catch (e) {
  console.error("doWork failed:", e); // or: throw new Error("context", { cause: e });
}
```

Optional chaining and nullish coalescing (ES2020) reduce a whole class of `undefined`-access errors:

```js
user?.address?.city ?? "unknown"; // safely reads nested, falls back only on null/undefined (not 0 or "")
```

**Real Scenario:** `user.settings.theme || "light"` looks like a safe default, but if a user explicitly sets `theme: ""` (empty string, e.g. "no preference") or `theme: 0`, `||` treats it as falsy and silently overrides it. `??` fixes this because it only falls back on `null`/`undefined`, not all falsy values — this is precisely the `0`/`false` footgun from [Section 1](#1-values-types--coercion) resurfacing.

### Advanced

Custom errors should extend `Error` and set `.name`, so `instanceof` checks and stack traces stay meaningful:

```js
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = "ValidationError";
    this.field = field;
  }
}
try { throw new ValidationError("bad email", "email"); }
catch (e) { if (e instanceof ValidationError) { /* handle */ } }
```

`Error.cause` (ES2022) lets you chain the original error without losing it, important once you have multiple layers of `try/catch` wrapping lower-level errors:

```js
throw new Error("failed to load user", { cause: originalError });
```

### Mastery

`structuredClone()` (Node 17+/all modern browsers) is now the standard deep-clone primitive, replacing the old `JSON.parse(JSON.stringify(x))` hack — which silently drops `undefined`, functions, `Symbol`s, and mangles `Date`/`Map`/`Set`. `structuredClone` handles those correctly but still can't clone functions or DOM nodes (throws `DataCloneError`), and doesn't preserve prototype chains for class instances — worth knowing before reaching for it on complex object graphs.

Deep equality has no built-in operator — `===` is always reference equality for objects. `JSON.stringify(a) === JSON.stringify(b)` is a common but flawed workaround (key order matters, breaks on functions/undefined/circular refs); production code typically reaches for a library (`lodash.isEqual`) or a hand-written recursive check rather than reinventing this.

[⤴ back to top](#table-of-contents)

---

## 9. Modern Runtime & Tooling Context

*(Beginner/Working Knowledge only — deep tooling mastery is out of scope here; this section exists to orient you before the TypeScript doc, where `tsconfig.json` and the compiler pipeline take over.)*

### Beginner

- **Node.js** runs JS outside the browser (server, scripts, tooling); **npm** is its default package manager, driven by `package.json`.
- `package.json`'s `"type": "module"` field switches a project to ESM by default (`.js` files become ES modules); omit it (or set `"commonjs"`) to keep CJS default. `.mjs`/`.cjs` extensions override this per-file regardless of the `type` field.

### Working Knowledge

- Current Node LTS lines as of September 2026: **Node 22** and **Node 24** (both support `require(esm)` interop); **Node 26** is the "Current" non-LTS release, entering LTS in October 2026 — check [Node's release schedule](https://github.com/nodejs/Release) before pinning a version in production.
- `npx <package>` runs a package's CLI without a global install — the standard way to run one-off tools (including, later, `tsc` via `npx tsc`).
- This is also the seam where TypeScript enters: TS is a build-time layer (`tsc` compiles `.ts` → `.js`) that erases at runtime — the JS engine never sees a type. Everything in this document is what actually executes; TypeScript's entire value proposition is catching mistakes in this document's Advanced/Mastery gotchas *before* runtime.

[⤴ back to top](#table-of-contents)

---

## 10. Suggested Learning/Reference Order

1. **[Values, Types & Coercion](#1-values-types--coercion)** — foundation for everything else; the `==`/coercion rules explain half of JS's "weirdness" reputation.
2. **[Variables & Scope](#2-variables--scope)** — `let`/`const`/closures are prerequisite vocabulary for the Functions section.
3. **[Functions](#3-functions)** — especially `this` binding; this is the single most-asked-about JS interview/debugging topic.
4. **[Objects & Prototypes](#4-objects--prototypes)** — class syntax here is your direct on-ramp to TS classes/interfaces.
5. **[Arrays & Iteration](#5-arrays--iteration)** — mutate-vs-new-array table alone will prevent a lot of real bugs.
6. **[Asynchronous JS](#6-asynchronous-js)** — the highest-payoff section for production code; budget real time here.
7. **[Modules](#7-modules)** — CJS/ESM distinction maps directly onto TS's `module` config later.
8. **[Equality, Errors & Edge Cases](#8-equality-errors--edge-cases)** — quick, but the `??` vs `||` distinction is a frequent real-world bug source.
9. **[Modern Runtime & Tooling Context](#9-modern-runtime--tooling-context)** — read last, as a bridge into the TypeScript doc.

[⤴ back to top](#table-of-contents)

---

## 11. Quick Self-Check

Try answering these without looking back — if any are shaky, revisit that section before moving to TypeScript:

1. Why does `typeof null` return `"object"`, and is it actually an object?
2. What's the output of `for (var i = 0; i < 3; i++) { setTimeout(() => console.log(i), 0); }`, and why does changing `var` to `let` change it?
3. Why do arrow functions make bad object methods when you need `this` to refer to the object?
4. What's the difference between `Object.freeze()` and making all properties `const`?
5. Which of these mutate their array, and which return a new one: `sort`, `map`, `splice`, `toSorted`, `filter`?
6. Why can a `try/catch` around an `await` still miss an unhandled promise rejection?
7. What's the practical difference between `??` and `||`, and when does it actually matter?
8. Why does `require()` work anywhere in a CJS file but `import` only work at the top level of an ESM file?
9. What does `Error.cause` give you that a plain `throw new Error(message)` doesn't?
10. Why is `#privateField` a stronger privacy guarantee than TypeScript's `private` keyword?

[⤴ back to top](#table-of-contents)
