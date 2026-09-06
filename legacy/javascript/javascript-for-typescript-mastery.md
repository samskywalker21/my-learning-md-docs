# JavaScript Mastery Guide (Pre-TypeScript)

> 🔄 **Needs re-write** — predates the current `prompt.md` template. Will be regenerated/updated to match it; treat as possibly stale on structure/format until then.

## About This Document

- **Framing:** **Goal-driven** — learning JavaScript specifically as preparation for TypeScript. Depth and prioritization favor the language-level features TypeScript's type system sits on top of, not general web-dev breadth (DOM/browser APIs excluded by default).
- **Organizing principle:** TypeScript is *erased* at build time. Every runtime behavior in this document is something TS can describe but never change. So each Part ends with a **→ In TypeScript** note naming the exact TS feature that models the JS behavior you just read — that mapping is the point of the doc.
- **Depth tiers:** **Beginner → Working Knowledge → Advanced → Mastery**, applied per Part, collapsing any tier that would be filler for a given topic. [Modern Runtime & Tooling Context](#11-modern-runtime--tooling-context) intentionally stops at Working Knowledge.
- **Confirmed scope:**
  1. Values, Types & Coercion
  2. Variables & Scope
  3. Functions
  4. Objects & Prototypes
  5. Arrays & Iteration
  6. Keyed Collections: Map and Set
  7. Iterators and Generators
  8. Asynchronous JS
  9. Modules
  10. Equality, Errors & Edge Cases
  11. Modern Runtime & Tooling Context (Beginner/Working Knowledge only)
  12. JavaScript to TypeScript Concept Map
- **Assumed background:** General programming fluency (variables, loops, functions in some language), new to JavaScript specifically. No prior TypeScript needed.
- **Runtime for examples:** Node.js **24 (Active LTS)** and **22 (Maintenance LTS)**; **Node 26** is the Current line as of September 2026. Browser differences noted where they matter. Language level assumed: **ES2025**, with ES2026 additions flagged as such.
- **Excluded by design:** DOM/browser APIs, styling/CSS, framework usage, and TypeScript's own syntax (this doc stops at the seam — the `typescript/` docs pick up there). For the JS→React path instead, see [`javascript-for-react-mastery.md`](./javascript-for-react-mastery.md); for JS from zero, see [`javascript-mastery-guide.md`](./javascript-mastery-guide.md).
- **Sourcing standard:** [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript) and the [ECMAScript spec](https://tc39.es/ecma262/) first, [Node.js docs](https://nodejs.org/api/) for runtime specifics, Stack Overflow for real-world gotchas. Where a popular convention disagrees with current official docs, the official doc wins and the disagreement is stated explicitly.
- **If you ask me to update this doc later:** keep this structure, the tier labels, the wrong-vs-right snippet pairs, the "Real Scenario" blocks, the per-Part **→ In TypeScript** notes, the end-of-Part cheat-sheet tables, the TOC, and the back-to-top links.

---

## Table of Contents

1. [Values, Types & Coercion](#1-values-types--coercion)
2. [Variables & Scope](#2-variables--scope)
3. [Functions](#3-functions)
4. [Objects & Prototypes](#4-objects--prototypes)
5. [Arrays & Iteration](#5-arrays--iteration)
6. [Keyed Collections: Map and Set](#6-keyed-collections-map-and-set)
7. [Iterators and Generators](#7-iterators-and-generators)
8. [Asynchronous JS](#8-asynchronous-js)
9. [Modules](#9-modules)
10. [Equality, Errors & Edge Cases](#10-equality-errors--edge-cases)
11. [Modern Runtime & Tooling Context](#11-modern-runtime--tooling-context)
12. [JavaScript to TypeScript Concept Map](#12-javascript-to-typescript-concept-map)
13. [Suggested Learning/Reference Order](#13-suggested-learningreference-order)
14. [Quick Self-Check](#14-quick-self-check)

---

## 1. Values, Types & Coercion

This is the Part that predicts most of your future TypeScript "aha" moments — TS's static type system exists almost entirely to catch the bugs described here.

### Beginner

JavaScript has **7 primitive types** (`string`, `number`, `boolean`, `null`, `undefined`, `bigint`, `symbol`) and one composite type, `object` — arrays, functions, dates, `Map`s, and regexes are all objects. **Variables are not typed; values are.** A variable can hold any type at any time.

```js
let x = 5;
x = "now a string"; // perfectly legal in JS, an error in TS by default
```

Check a type with `typeof` (primitives) or `instanceof` (objects/classes):

```js
typeof "hi";          // "string"
typeof 5;             // "number"
typeof 10n;           // "bigint"
typeof Symbol();      // "symbol"
typeof undefined;     // "undefined"
typeof (() => {});    // "function" — a special case; functions are objects
typeof null;          // "object" — a 1995 spec bug, not a real object type (see Advanced)
[] instanceof Array;  // true
```

`typeof` has **eight** possible results for seven primitives plus objects: `"function"` is broken out as its own answer even though functions are objects ([MDN: `typeof`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/typeof)).

### Working Knowledge

**Coercion** happens implicitly in comparisons, arithmetic, and template literals. This is where JS earns its reputation.

```js
// Wrong — relying on implicit coercion without knowing the rules
if (userInput == 0) { /* also true for "", " ", false, [], and "0" */ }

// Right — convert deliberately, then compare strictly
if (Number(userInput) === 0) { /* ... */ }
```

Coercion rules worth internalizing:

| Expression | Result | Why |
|---|---|---|
| `"5" + 1` | `"51"` | `+` with a string operand concatenates; it does not add |
| `"5" - 1` | `4` | every other arithmetic operator coerces to number |
| `+"5"` | `5` | unary `+` is the terse number cast |
| `[] + []` | `""` | arrays stringify via `join(",")`, and an empty array joins to `""` |
| `[] + {}` | `"[object Object]"` | same rule, applied on the object side |
| `1 < 2 < 3` | `true` | but by accident: `(1 < 2)` → `true`, then `true < 3` → `1 < 3` |
| `3 > 2 > 1` | `false` | same mechanism, and this time the accident bites |

**The falsy values are exactly eight**, and memorizing this list is worth five minutes: `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN`. Everything else is truthy — including `[]`, `{}`, `"0"`, `"false"`, and `new Boolean(false)`. (Browsers add one deliberate legacy exception, `document.all`, which is falsy despite being an object; see [MDN: Falsy](https://developer.mozilla.org/en-US/docs/Glossary/Falsy).)

**Real Scenario:** A form handler validates with `if (formData.age)`. A user enters `0` — legitimate for an age, a quantity, or a discount. The falsy check silently treats `0` as "not provided," the validation branch never runs, and the bug surfaces only for that one input, weeks after ship. Fix: test for presence, not truthiness — `formData.age !== undefined`, or `formData.age ?? fallback` (see [Part 10](#10-equality-errors--edge-cases)).

### Advanced

**`NaN` is the only value not equal to itself.** `NaN !== NaN` is true, because IEEE-754 defines it that way. Use `Number.isNaN()` — never `x === NaN` (always false), and never the global `isNaN()`, which coerces first:

```js
Number.isNaN(NaN);        // true
Number.isNaN("hello");    // false — it's a string, not NaN
isNaN("hello");           // true  — legacy global, coerces first. Avoid.
```

**All JS numbers are IEEE-754 doubles**, including the ones that look like integers. Two consequences you will meet in production:

```js
0.1 + 0.2;                                  // 0.30000000000000004 — binary floating point, not a JS bug
0.1 + 0.2 === 0.3;                          // false
Math.abs(0.1 + 0.2 - 0.3) < Number.EPSILON; // true — the correct float comparison

Number.MAX_SAFE_INTEGER;                    // 9007199254740991 (2**53 - 1)
9007199254740993 === 9007199254740992;      // true — beyond that, integers collide
```

Money and large IDs are the classic casualties. Use integer minor units (cents) or `BigInt` for money, and **keep 64-bit database IDs as strings** — a Postgres `bigint` primary key round-tripped through `JSON.parse` silently loses precision above 2^53.

`typeof null === "object"` is an artifact of the original 1995 implementation (values carried type tags, and the null pointer's tag was the object tag). It is [documented rather than fixed](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/typeof#typeof_null) because fixing it would break the web. The practical consequence: `typeof x === "object"` does **not** prove you have an object.

```js
// Wrong
function isObject(x) { return typeof x === "object"; } // true for null

// Right
function isObject(x) { return x !== null && typeof x === "object"; }
```

`Symbol` and `BigInt` are the two primitives most JS learners never touch but that TypeScript models explicitly:

- **`Symbol()`** creates a guaranteed-unique value, used as a non-colliding object key. The well-known symbols (`Symbol.iterator`, `Symbol.asyncIterator`, `Symbol.toPrimitive`, `Symbol.toStringTag`) are the language's own extension points — `Symbol.iterator` is what makes `for...of` and spread work (see [Part 7](#7-iterators-and-generators)).
- **`BigInt`** (`10n`) is arbitrary-precision integer arithmetic. Mixing `BigInt` and `Number` in arithmetic throws a `TypeError` **by design** rather than silently coercing — one of the very few places JS chose safety over convenience.

```js
10n + 5n;   // 15n
10n + 5;    // TypeError: Cannot mix BigInt and other types, use explicit conversions
10n == 10;  // true  — loose equality still bridges them
10n === 10; // false — different types
```

### Mastery

Loose equality (`==`) follows the [Abstract Equality Comparison algorithm](https://tc39.es/ecma262/#sec-islooselyequal) — a specific, memorizable-but-not-worth-memorizing coercion sequence. The convergent professional practice: **default to `===`, and reach for `==` in exactly one idiom**, `x == null`, which matches `null` and `undefined` together and nothing else.

```js
// wrong vs right for the "is this present at all" check
if (x !== null && x !== undefined) { /* works, verbose */ }
if (x != null)                     { /* idiomatic, deliberate use of == */ }
```

Object-to-primitive conversion is customizable, and this is the mechanism behind every surprising coercion result. When an object is used where a primitive is expected, the engine calls `Symbol.toPrimitive` if present, otherwise `valueOf()` then `toString()` (order reversed for a string hint):

```js
const money = {
  amount: 42,
  [Symbol.toPrimitive](hint) {
    return hint === "string" ? `$${this.amount}` : this.amount;
  },
};
money + 1;  // 43    — "default" hint takes the numeric path
`${money}`; // "$42" — "string" hint
```

This is also the entire explanation for `[] + {}`: both operands take the "default" hint, `[].toString()` is `""`, `({}).toString()` is `"[object Object]"`, and `+` concatenates the two.

> **→ In TypeScript:** `typeof` becomes a *type guard* that narrows a union — inside `if (typeof x === "string")`, `x` is a `string`. The 7 primitives become the primitive types plus the literal types built from them; `null` and `undefined` become distinct types you must handle once `strictNullChecks` is on. Note what TS does **not** fix: `NaN` is still typed `number`, and `0.1 + 0.2 !== 0.3` is a runtime property no type system erases.

### Part 1 Cheat Sheet

| Question | Answer |
|---|---|
| How many primitives? | 7: `string`, `number`, `boolean`, `null`, `undefined`, `bigint`, `symbol` |
| How many falsy values? | 8: `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN` |
| Safe object check | `x !== null && typeof x === "object"` |
| Safe NaN check | `Number.isNaN(x)` (never `=== NaN`, never global `isNaN`) |
| Safe float compare | `Math.abs(a - b) < Number.EPSILON` |
| Integer safety ceiling | `Number.MAX_SAFE_INTEGER` = 2^53 − 1 |
| Only good use of `==` | `x == null` (matches `null` + `undefined`) |

[⤴ back to top](#table-of-contents)

---

## 2. Variables & Scope

### Beginner

Three ways to declare a variable: `var` (function-scoped, legacy), `let` (block-scoped, reassignable), `const` (block-scoped, not reassignable — but the *value* is still mutable if it's an object).

```js
const arr = [1, 2, 3];
arr.push(4); // fine — mutating the object, not reassigning the binding
arr = [5, 6]; // TypeError: Assignment to constant variable.
```

`const` freezes the **binding**, not the value. This distinction matters enormously in TypeScript, where `readonly` and `as const` split the same idea into two separate tools.

### Working Knowledge

**Hoisting**: declarations are processed before code runs, but *how* differs by keyword.

```js
console.log(a); // undefined — var is hoisted AND initialized to undefined
var a = 1;

console.log(b); // ReferenceError: Cannot access 'b' before initialization
let b = 1;
```

`let`/`const` are hoisted too, but land in the **Temporal Dead Zone (TDZ)**: the binding exists in scope from the top of the block, but touching it before the declaration line executes throws. This is a feature — it converts a class of silent `undefined` bugs into loud errors.

**Real Scenario — the classic loop-closure bug**, one of the most-asked JavaScript questions ever ([Stack Overflow: JavaScript closure inside loops](https://stackoverflow.com/questions/750486/javascript-closure-inside-loops-simple-practical-example)):

```js
// Wrong
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// logs: 3 3 3 — var is function-scoped, so all three callbacks close over ONE `i`,
// which has already reached 3 by the time any callback runs.

// Right
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// logs: 0 1 2 — `let` creates a fresh binding per iteration
```

The per-iteration binding is specified behavior, not an implementation quirk: the spec's `CreatePerIterationEnvironment` step copies the loop variable into a new environment on every pass.

### Advanced

**Closures**: a function retains access to the variables of the scope it was *defined* in, even after that scope has returned. This is the mechanism behind the loop bug above, and behind module-private state, memoization, and every "factory function" pattern.

```js
function makeCounter() {
  let count = 0;
  return {
    increment: () => ++count,
    get value() { return count; },
  };
}
const counter = makeCounter();
counter.increment(); // 1
counter.increment(); // 2 — `count` persisted, and is unreachable from outside
```

**Strict mode** matters more than most tutorials suggest, because ES modules and class bodies are *always* strict — so if you write modern JS, you are already in strict mode whether or not you asked for it. Strict mode makes silent failures throw, forbids undeclared globals, and leaves `this` as `undefined` in an unbound function call rather than substituting the global object:

```js
"use strict";
undeclared = 5;   // ReferenceError (in sloppy mode: silently creates a global)
function f() { return this; }
f();              // undefined (in sloppy mode: globalThis)
```

Prefer `let`/`const` over `var` in all new code. This is uncontroversial modern practice per [MDN's `var` documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/var), which now steers readers to `let`/`const` in its own opening.

### Mastery

Every function call creates a new **lexical environment** (a record of that scope's bindings plus a reference to its parent). The chain of these environments is resolved by where code is *written*, not where it is *called* — that is what "lexical scoping" means, and it is why closures work at all.

The practical consequence in long-running Node processes is memory retention. A closure keeps its enclosing environment alive, so a small callback can pin a large object graph:

```js
// Wrong — the interval callback closes over the whole enclosing scope
function startPolling(hugeDataset) {
  const id = summarize(hugeDataset);
  setInterval(() => report(id), 1000); // may retain `hugeDataset` for process lifetime
}

// Right — capture only what you need, and keep a handle to clear it
function startPolling(hugeDataset) {
  const id = summarize(hugeDataset); // hugeDataset now unreachable after this line
  const timer = setInterval(() => report(id), 1000);
  return () => clearInterval(timer);
}
```

Modern V8 does per-variable capture analysis and will usually *not* over-retain, but "usually" is doing real work in that sentence: any `eval` or debugger in scope forces full-scope capture, and shared context objects between sibling closures can keep a variable alive because a *different* closure uses it.

> **→ In TypeScript:** `const` maps to two TS ideas that are easy to confuse — `as const` (freeze the inferred *type* to its literal, e.g. `"GET"` instead of `string`) and `readonly` (forbid property writes at compile time). Neither exists at runtime. TDZ errors stay TDZ errors: TS catches some use-before-declare cases statically, but the runtime throw is the real guarantee.

### Part 2 Cheat Sheet

| Concept | `var` | `let` | `const` |
|---|---|---|---|
| Scope | function | block | block |
| Hoisted | yes, initialized `undefined` | yes, into TDZ | yes, into TDZ |
| Reassignable | yes | yes | no |
| Redeclarable in same scope | yes | no | no |
| Fresh binding per loop iteration | no | yes | yes (`for...of`/`for...in`) |
| Creates a property on `globalThis` at top level | yes (scripts) | no | no |

[⤴ back to top](#table-of-contents)

---

## 3. Functions

### Beginner

Three syntaxes, one underlying concept — a callable value that can be passed around like any other value:

```js
function add(a, b) { return a + b; }             // declaration — hoisted fully
const add2 = function (a, b) { return a + b; };  // expression — binding is in TDZ until this line
const add3 = (a, b) => a + b;                    // arrow — no own `this`, `arguments`, or `new`
```

### Working Knowledge

Default, rest, and spread parameters:

```js
function greet(name = "world") { return `hi ${name}`; }
function sum(...nums) { return nums.reduce((a, b) => a + b, 0); } // rest: gather into an array
sum(...[1, 2, 3]); // 6 — spread: expand an array into arguments
```

A default parameter fires only for `undefined`, not for `null` — the same `??` semantics from [Part 10](#10-equality-errors--edge-cases):

```js
greet(undefined); // "hi world" — default applies
greet(null);      // "hi null"  — null is a real value, so no default
```

**`this` binding** depends on *how* a regular function is called, not where it is defined. Arrow functions are the exception: they have no `this` of their own and resolve it lexically from the enclosing scope ([MDN: `this`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)).

```js
const obj = {
  name: "widget",
  // Wrong — a regular callback is called with no receiver, so `this` is not `obj`
  logLater() {
    setTimeout(function () { console.log(this.name); }, 0);
    // Node: `this` is the Timeout object → undefined. Browser: `this` is window → "".
    // In strict mode with a plain call, `this` is undefined and this line throws.
  },
  // Right — the arrow inherits `this` from logLaterFixed's scope
  logLaterFixed() {
    setTimeout(() => console.log(this.name), 0); // "widget"
  },
};
```

The four `this` rules, in precedence order: `new` binding → explicit binding (`call`/`apply`/`bind`) → implicit binding (`obj.method()`) → default binding (`undefined` in strict mode, `globalThis` in sloppy). Arrow functions opt out of all four and inherit from the enclosing scope.

### Advanced

`call`, `apply`, and `bind` set `this` explicitly:

```js
function greet(greeting) { return `${greeting} ${this.name}`; }
greet.call({ name: "Sam" }, "hi");    // invoke now, args listed
greet.apply({ name: "Sam" }, ["hi"]); // invoke now, args as an array
const bound = greet.bind({ name: "Sam" }); // returns a NEW function, permanently bound
bound.call({ name: "other" }, "hi");  // "hi Sam" — bind wins; it cannot be re-bound
```

**Real Scenario:** A class method is passed as a callback — `element.addEventListener("click", this.handleClick)` or `setTimeout(this.tick, 100)`. The method is now detached from its receiver, so `this` is `undefined` (class bodies are always strict), and the first `this.something` inside it throws `TypeError: Cannot read properties of undefined`. This is the single most common `this` bug in the language, and it is why class fields holding arrows — `handleClick = () => { ... }` — became the idiomatic fix: a field initializer runs with `this` bound to the instance, and the arrow captures it permanently.

```js
class Widget {
  name = "widget";
  broken() { console.log(this.name); }        // breaks when detached
  fixed = () => { console.log(this.name); };  // survives detachment
}
const w = new Widget();
const { broken, fixed } = w;
fixed();  // "widget"
broken(); // TypeError — `this` is undefined
```

The tradeoff is real, not just stylistic: `fixed` is a **per-instance** property (one closure allocated per object, not on the prototype), so it costs memory per instance and is invisible to prototype-level patching and `super`.

Function objects also carry properties worth knowing: `fn.length` is declared arity **counting only parameters before the first default or rest** (`((a, b = 1, ...c) => {}).length` is `1`), and `fn.name` is inferred from the assignment target for anonymous expressions — which is why stack traces from arrow functions are usually still readable.

### Mastery

Arrow functions are not "shorter `function`." They structurally **lack** their own `this`, `arguments`, `super`, and `new.target`, resolving each from the enclosing lexical scope. Three consequences follow directly:

```js
const Arrow = () => {};
new Arrow();                       // TypeError: Arrow is not a constructor
Arrow.prototype;                   // undefined — arrows have no prototype property
(() => this).call({ x: 1 })();     // call cannot override an arrow's `this`
```

This also means an arrow is the wrong tool for an object method you intend to call as `obj.method()`, and the wrong tool for a prototype method — in both cases `this` will resolve to the module or global scope rather than the instance.

```js
// Wrong — `this` is the module scope, not `counter`
const counter = { n: 0, inc: () => { this.n++; } };

// Right — method shorthand gets normal `this` binding
const counter2 = { n: 0, inc() { this.n++; } };
```

The rule that covers every case: **use an arrow when you want the surrounding `this`; use a method or `function` when you want the caller to supply it.**

> **→ In TypeScript:** this Part maps onto more TS surface than any other. Parameters and returns get annotations; optional (`?`) and default parameters change the *call* signature; rest parameters become tuple types; overloads let one implementation expose several call shapes. The `this` rules get their own feature — a fake first parameter, `function f(this: Widget, x: number)`, which TS type-checks and erases. And the arrow-field-vs-method choice above has a direct TS consequence: `strictFunctionTypes` treats method and property function types differently in variance checks.

### Part 3 Cheat Sheet

| You want | Use |
|---|---|
| `this` from the surrounding scope | arrow function |
| `this` supplied by the call site | `function` / method shorthand |
| A constructor | `function` or `class` (never an arrow) |
| To fix `this` permanently | `.bind()`, or an arrow class field |
| Variable argument count | rest `...args` (never `arguments` — not available in arrows) |
| A callback that survives detachment | arrow class field, or bind in the constructor |

[⤴ back to top](#table-of-contents)

---

## 4. Objects & Prototypes

This Part maps most directly onto TypeScript's `interface`, `class`, and structural typing — worth extra attention.

### Beginner

```js
const user = { name: "Sam", age: 30 };
user.name;   // "Sam"
user["age"]; // 30 — bracket notation for dynamic or non-identifier keys

const key = "name";
user[key];   // "Sam" — the reason bracket notation exists
```

Modern object literal syntax you will see constantly:

```js
const name = "Sam", age = 30;
const field = "role";

const obj = {
  name, age,                 // shorthand: { name: name, age: age }
  [field]: "admin",          // computed key → { role: "admin" }
  greet() { return "hi"; },  // method shorthand
  ...defaults,               // object spread — shallow copy, later keys win
};
```

### Working Knowledge

Every object has an internal prototype link (`[[Prototype]]`). Property lookups walk this **prototype chain** until the key is found or the chain ends at `null`.

```js
const animal = { speak() { return "..."; } };
const dog = Object.create(animal); // dog's prototype is animal
dog.speak();                        // "..." — found on animal, not on dog
Object.getPrototypeOf(dog) === animal; // true
dog.hasOwnProperty("speak");        // false — it's inherited, not own
Object.hasOwn(dog, "speak");        // false — ES2022, the preferred form
```

Prefer `Object.getPrototypeOf()` over the legacy `__proto__` accessor, and `Object.hasOwn(obj, key)` over `obj.hasOwnProperty(key)` — the latter breaks on objects created with `Object.create(null)`, which have no prototype and therefore no `hasOwnProperty` method.

`class` syntax (ES2015+) is sugar over this same prototype mechanism. It does not introduce a new inheritance model ([MDN: Classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes)):

```js
class Animal {
  #energy = 100;                    // private field (ES2022) — enforced by the language
  static kingdom = "Animalia";      // static field, on the constructor not instances
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
  get isTired() { return this.#energy < 10; }
  set energy(v) { this.#energy = Math.max(0, v); }
  static isAnimal(x) { return #energy in x; }  // ergonomic brand check (ES2022)
}

class Dog extends Animal {
  speak() { return `${super.speak()} — specifically, a bark`; }
}
```

Methods live on `Animal.prototype` and are shared by every instance; fields declared in the class body (and anything assigned to `this` in the constructor) are per-instance.

### Advanced

The most common prototype surprise is that **assignment never writes through the chain**:

```js
// Wrong — assuming a prototype property is shared mutable state
function Counter() {}
Counter.prototype.count = 0;
const c1 = new Counter(), c2 = new Counter();
c1.count++;   // creates an OWN property `count` on c1, shadowing the prototype
c2.count;     // still 0

// And the mirror-image trap — objects and arrays ARE shared, because you mutate rather than assign
Counter.prototype.log = [];
c1.log.push("x");
c2.log;       // ["x"] — same array, shared by every instance
```

Both halves bite in practice: primitives on a prototype behave as per-instance defaults, and objects on a prototype behave as global shared state. Neither is usually what you meant. Declare instance state in the constructor or as a class field.

`Object.freeze()` prevents adding, removing, or reassigning properties — but **only one level deep**:

```js
const config = Object.freeze({ db: { host: "localhost" } });
config.db = {};          // silently ignored (throws in strict mode)
config.db.host = "prod"; // succeeds — freeze is shallow
```

Property descriptors (`Object.defineProperty`, `Object.getOwnPropertyDescriptor`) control `writable`, `enumerable`, and `configurable` per property. This is how getters/setters, non-enumerable internals, and pre-`#private` privacy patterns were built, and it explains why some built-in properties don't show up in `Object.keys()` or `JSON.stringify()`.

Useful object utilities, all of which operate on **own, enumerable, string-keyed** properties only:

```js
Object.keys(obj);       // ["name", "age"]
Object.values(obj);     // ["Sam", 30]
Object.entries(obj);    // [["name", "Sam"], ["age", 30]]
Object.fromEntries(pairs);        // the inverse — great with Map and URLSearchParams
Object.assign(target, ...sources); // shallow merge, mutates target
structuredClone(obj);   // deep clone — see Part 10
```

### Mastery

**Private class fields (`#field`) are not a naming convention.** Unlike the old `_field` trick and unlike TypeScript's `private`, they are enforced by the language: accessing `#field` from outside the class body is a **`SyntaxError` at parse time**, not a runtime check you can defeat. There is no bracket-notation escape hatch, they never appear in `Object.keys`, `JSON.stringify`, or a `Proxy` trap, and they are the only genuinely private storage in the language.

```js
class Account {
  #balance = 0;
  add(n) { this.#balance += n; }
}
const a = new Account();
a.#balance;        // SyntaxError — the file will not even parse
a["#balance"];     // undefined — that's just an ordinary string key, unrelated
Object.keys(a);    // [] — private fields are invisible to reflection
```

This is exactly why TypeScript's `private` and JS's `#private` behave differently: `private` is a compile-time annotation erased at build (the property is a normal, fully accessible property at runtime), while `#field` survives into the emitted JS with real enforcement. Moving from one to the other is a semantic change, not a syntax change.

On performance: V8 assigns each object a **hidden class** (a "shape") and optimizes property access that always sees the same shape (monomorphic). Adding properties in different orders produces different hidden classes for structurally identical objects, and a call site that sees many shapes (polymorphic/megamorphic) falls back to slower lookup.

```js
// These two objects have DIFFERENT hidden classes despite identical keys
const a = { x: 1 }; a.y = 2;
const b = { y: 2 }; b.x = 1;
```

The takeaway is not to hand-optimize — it is to understand why consistently shaped objects (which `class` gives you for free) tend to outperform ad-hoc literals in hot loops, and why initializing every field in the constructor, even to `null`, is a real technique in performance-sensitive Node code.

> **→ In TypeScript:** this is where TS's defining choice shows up — **structural typing**. A TS `interface` describes a *shape*, and any object with that shape satisfies it, with no `implements` declaration required. That is a direct model of how JS property lookup actually works. Also mapping here: `class` fields become typed properties; `get`/`set` become typed accessors; `Object.keys` returning `string[]` (not a union of your keys) is a deliberate TS decision that surprises everyone once; and `private` vs `#private` is the erasure distinction described above.

### Part 4 Cheat Sheet

| Task | Modern API | Avoid |
|---|---|---|
| Read the prototype | `Object.getPrototypeOf(o)` | `o.__proto__` |
| Own-property check | `Object.hasOwn(o, k)` | `o.hasOwnProperty(k)` |
| Shallow copy | `{ ...o }` / `Object.assign({}, o)` | manual loops |
| Deep copy | `structuredClone(o)` | `JSON.parse(JSON.stringify(o))` |
| Key/value pairs | `Object.entries(o)` / `Object.fromEntries(p)` | `for...in` |
| True privacy | `#field` | `_field` naming convention |
| Shallow immutability | `Object.freeze(o)` | (nothing deep is built in) |

[⤴ back to top](#table-of-contents)

---

## 5. Arrays & Iteration

### Beginner

Arrays are objects with integer-like keys and a `length` property, plus a large built-in method set.

```js
const nums = [1, 2, 3];
nums.push(4);         // mutates → [1, 2, 3, 4]
nums.map(n => n * 2); // returns a NEW array → [2, 4, 6, 8]
nums.at(-1);          // 4 — ES2022, the clean way to read from the end
```

### Working Knowledge

The single most useful thing to memorize about arrays is which methods **mutate** and which **return new**:

| Mutates in place | Returns a new array |
|---|---|
| `push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`, `fill`, `copyWithin` | `map`, `filter`, `slice`, `concat`, `flat`, `flatMap`, `toSorted`, `toReversed`, `toSpliced`, `with` |

The four non-mutating twins (`toSorted`, `toReversed`, `toSpliced`, `with`) landed in **ES2023** and are the modern answer to the copy-then-mutate dance:

```js
// Wrong — sort() mutates, so this silently reorders `original` too
const original = [3, 1, 2];
const sorted = original.sort();
original; // [1, 2, 3] — mutated! Easy to miss inside a larger function.

// Old workaround
const sorted2 = [...original].sort();

// Right (ES2023) — non-mutating by construction
const fresh = [3, 1, 2];
const sorted3 = fresh.toSorted(); // [1, 2, 3]
fresh;                            // [3, 1, 2] — untouched
```

And the `sort()` gotcha that outlives every other: **the default comparator sorts by string**, so numbers come out wrong.

```js
[10, 9, 1].sort();            // [1, 10, 9]  — string comparison
[10, 9, 1].sort((a, b) => a - b); // [1, 9, 10] — numeric comparator
```

Destructuring and spread:

```js
const [first, ...rest] = [1, 2, 3];      // first = 1, rest = [2, 3]
const [, second] = [1, 2];               // hole to skip an element
const [x = 10] = [];                     // default for a missing element
const merged = [...arr1, ...arr2];       // concat, shallow
```

### Advanced

`reduce` is the general-purpose primitive most other array methods can be built from — worth understanding once, then mostly *not* reaching for when a clearer method exists:

```js
// Wrong — reduce doing map's job, harder to read for no benefit
const doubled = nums.reduce((acc, n) => { acc.push(n * 2); return acc; }, []);

// Right
const doubled2 = nums.map(n => n * 2);
```

**Real Scenario:** grouping API results by a field is the most common legitimate `reduce`, and as of **ES2024** it has a purpose-built replacement, [`Object.groupBy`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/groupBy) (with [`Map.groupBy`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map/groupBy) when you need non-string keys):

```js
const inventory = [
  { type: "fruit", name: "apple" },
  { type: "veg", name: "carrot" },
  { type: "fruit", name: "pear" },
];

// Before
const grouped = inventory.reduce((acc, item) => {
  (acc[item.type] ??= []).push(item);
  return acc;
}, {});

// After (ES2024)
Object.groupBy(inventory, item => item.type);
// { fruit: [{apple}, {pear}], veg: [{carrot}] }
```

One sharp edge: `Object.groupBy` returns a **null-prototype object**, so it has no `hasOwnProperty` and won't inherit from `Object.prototype` — deliberate, so that a group key of `"toString"` can't collide with anything.

**Sparse arrays and holes** are the array subsystem's dark corner. Different methods disagree about whether a hole exists:

```js
const sparse = [1, , 3];      // length 3, index 1 is a hole
sparse.map(x => 2);           // [2, <hole>, 2] — map SKIPS holes but preserves them
sparse.forEach(x => count++); // runs twice, not three times
[...sparse];                  // [1, undefined, 3] — spread materializes holes
Array.from(sparse);           // [1, undefined, 3] — as does Array.from
new Array(3).fill(0);         // [0, 0, 0] — the safe way to make a filled array
```

The rule: avoid creating holes, and if you might have them, normalize with `Array.from()` before iterating.

### Mastery

**Iterator helpers (ES2025)** are the biggest change to iteration in a decade. `map`, `filter`, `take`, `drop`, `flatMap`, `reduce`, `toArray`, `some`, `every`, and `find` now exist on the `Iterator` prototype, which means they work on *any* iterator — generators, `Map.keys()`, `Set.values()` — and they are **lazy**, allocating no intermediate arrays:

```js
// Array methods: eager, allocates a full intermediate array at each step
const first3 = hugeArray.filter(isValid).map(transform).slice(0, 3); // processes ALL items

// Iterator helpers (ES2025): lazy, stops after 3 survive
const first3Lazy = hugeArray.values()
  .filter(isValid)
  .map(transform)
  .take(3)
  .toArray();
```

Support: Node 22+, Chrome 122+, Firefox 131+ ([MDN: Iterator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator)). This also finally makes infinite generators practical to consume — see [Part 7](#7-iterators-and-generators).

**`for...in` is not for arrays.** It iterates enumerable *string* keys including inherited ones, gives you keys as strings (`"0"`, not `0`), skips holes, and has no guaranteed order for non-integer keys. Use `for...of` for values, `.entries()` when you need the index, and `Object.keys/entries` for objects.

```js
// Wrong
for (const i in ["a", "b"]) { console.log(i + 1); } // "01", "11" — string keys!

// Right
for (const [i, v] of ["a", "b"].entries()) { console.log(i + 1, v); } // 1 "a", 2 "b"
```

> **→ In TypeScript:** arrays become `T[]` (or `Array<T>`), and fixed-length heterogeneous arrays become **tuples** (`[string, number]`) — the type that makes destructuring safe. The mutate-vs-new table above is exactly what `readonly T[]` and `ReadonlyArray<T>` encode: the readonly type simply omits the mutating methods. `filter` is the classic place you will reach for a **type predicate** (`(x): x is Foo => ...`), because TS cannot otherwise narrow an array through a callback.

### Part 5 Cheat Sheet

| Goal | Method |
|---|---|
| Transform every element | `map` |
| Keep some elements | `filter` |
| Collapse to one value | `reduce` |
| Find one element / its index | `find` / `findIndex` / `findLast` / `findLastIndex` |
| Test membership | `includes` (handles `NaN`) over `indexOf !== -1` |
| Read from the end | `at(-1)` |
| Sort numbers | `sort((a, b) => a - b)` — never bare `sort()` |
| Sort without mutating | `toSorted()` |
| Replace one index immutably | `with(i, v)` |
| Flatten | `flat(depth)` / `flatMap` |
| Group | `Object.groupBy` / `Map.groupBy` |
| Lazily process a huge/infinite sequence | iterator helpers (`.values().filter().take().toArray()`) |
| De-duplicate | `[...new Set(arr)]` |

[⤴ back to top](#table-of-contents)

---

## 6. Keyed Collections: Map and Set

Plain objects were the only dictionary JavaScript had for twenty years, and a great deal of legacy code still misuses them as one. `Map` and `Set` (ES2015) fix that, and TypeScript types them precisely — so knowing when to pick which is worth a short Part of its own.

### Beginner

```js
const m = new Map();
m.set("a", 1).set("b", 2);   // set() returns the map, so calls chain
m.get("a");                   // 1
m.has("a");                   // true
m.delete("a");                // true
m.size;                       // 1 — a property, not a method

const s = new Set([1, 2, 2, 3]);
s.size;                       // 3 — duplicates dropped
s.has(2);                     // true
[...new Set(array)];          // the canonical de-duplicate idiom
```

### Working Knowledge

**Why `Map` over a plain object?** Four concrete reasons, not style preferences:

| | Plain object | `Map` |
|---|---|---|
| Key types | strings and symbols only (everything else is stringified) | **any value**, including objects and functions |
| Ordering | integer-like keys sort first, ascending; others in insertion order | strict insertion order, always |
| Size | `Object.keys(o).length` — O(n) | `m.size` — O(1) |
| Inherited keys | `o.toString` exists on every literal | none; a `Map` has no key it wasn't given |
| Iteration | needs `Object.keys/entries` | directly iterable |

The stringification issue is the one that causes real bugs:

```js
// Wrong — object keys are stringified, so these two collide
const seen = {};
seen[{ id: 1 }] = "a";
seen[{ id: 2 }] = "b";
Object.keys(seen); // ["[object Object]"] — one key, second write clobbered the first

// Right — Map keys are compared by identity
const seen2 = new Map();
seen2.set({ id: 1 }, "a").set({ id: 2 }, "b");
seen2.size; // 2
```

Choose a plain object for fixed, known-at-author-time shapes (a config, a record with named fields) and a `Map` for dynamic, arbitrary-key lookup tables that grow at runtime.

Iterating both:

```js
for (const [k, v] of map) { /* entries by default */ }
for (const v of set) { /* values */ }
Object.fromEntries(map);   // Map → plain object (string keys only)
new Map(Object.entries(o)); // plain object → Map
```

### Advanced

**Set methods (ES2025)** finally make `Set` a real set type rather than a bag with `has()`:

```js
const a = new Set([1, 2, 3]);
const b = new Set([2, 3, 4]);

a.union(b);                // Set {1, 2, 3, 4}
a.intersection(b);         // Set {2, 3}
a.difference(b);           // Set {1}
a.symmetricDifference(b);  // Set {1, 4}
a.isSubsetOf(b);           // false
a.isSupersetOf(b);         // false
a.isDisjointFrom(b);       // false
```

These replace the hand-rolled `[...a].filter(x => b.has(x))` patterns that dominate older codebases, and they run in Node 22+, Chrome 122+, Firefox 127+ ([MDN: Set](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set)).

**Key equality** in both `Map` and `Set` uses **SameValueZero** — like `===`, except `NaN` equals itself. This is the pragmatic middle ground between `===` (where `NaN !== NaN`) and `Object.is` (where `+0` and `-0` differ):

```js
new Set([NaN, NaN]).size; // 1 — NaN deduplicates correctly here
new Set([0, -0]).size;    // 1 — but +0 and -0 are the same key
[NaN].includes(NaN);      // true  — includes also uses SameValueZero
[NaN].indexOf(NaN);       // -1    — indexOf uses ===, so it misses
```

**Real Scenario:** a request-deduplication cache keyed by a config object. Built with a plain object, every distinct config stringifies to `"[object Object]"`, so the cache returns the *first* request's response for every subsequent caller — a data-leak-shaped bug that looks like a caching win in local testing where only one config is ever used. `Map` keyed by the object (or by a canonical serialized string, if identity isn't stable across calls) is the fix.

### Mastery

`WeakMap` and `WeakSet` hold keys **weakly**: an entry does not prevent its key from being garbage collected. They accept only objects (and, since ES2023, non-registered symbols) as keys, have no `size`, and are not iterable — precisely because their contents can vanish between any two lines, so an enumerable view would be unsound.

```js
const metadata = new WeakMap();
function attach(node, data) { metadata.set(node, data); }
// When `node` becomes unreachable everywhere else, the entry is collected automatically.
```

Use them for **side-channel data attached to objects you don't own** — per-instance private state (the pre-`#field` pattern), memoization keyed by argument identity, and DOM node metadata. Using a plain `Map` for any of these is a textbook leak: the map's strong reference keeps every key alive for the process lifetime.

```js
// Wrong — a long-lived Map is an unbounded leak in a server process
const cache = new Map();
function decorate(user) { cache.set(user, expensive(user)); }

// Right — entries disappear with their keys
const cache2 = new WeakMap();
function decorate2(user) { cache2.set(user, expensive(user)); }
```

`WeakRef` and `FinalizationRegistry` (ES2021) expose GC even more directly, but the spec explicitly warns against relying on them: collection timing is not guaranteed, and a finalizer may never run. They are a last resort, not a tool for ordinary caching.

> **→ In TypeScript:** `Map<K, V>` and `Set<T>` are generic types, and they are usually the first place a newcomer *feels* generics pay off — `map.get(key)` returns `V | undefined`, which forces you to handle the miss. That `| undefined` is the whole argument for `strictNullChecks` in miniature. Also note `Object.fromEntries` returns a loosely typed record, so converting a `Map` to an object usually needs a cast or a `Record<K, V>` annotation.

### Part 6 Cheat Sheet

| Need | Use |
|---|---|
| Fixed, known field names | plain object (or a `class`) |
| Dynamic keys added at runtime | `Map` |
| Non-string keys | `Map` |
| Uniqueness / membership test | `Set` |
| Set algebra | `union` / `intersection` / `difference` (ES2025) |
| Metadata on objects you don't own | `WeakMap` |
| Order guarantee | `Map`/`Set` (insertion order, always) |
| Key equality rule | SameValueZero (`NaN` equals itself; `+0` equals `-0`) |

[⤴ back to top](#table-of-contents)

---

## 7. Iterators and Generators

This is the machinery underneath `for...of`, spread, destructuring, `Promise.all`'s argument, and the ES2025 iterator helpers. Most JS developers use it daily without ever seeing it; TypeScript makes you name it (`Iterable<T>`, `Iterator<T>`, `Generator<T, TReturn, TNext>`), so it is worth an hour now.

### Working Knowledge

An object is **iterable** if it has a `[Symbol.iterator]()` method returning an **iterator** — an object with a `next()` method that returns `{ value, done }`. That is the entire protocol.

```js
const range = {
  from: 1,
  to: 3,
  [Symbol.iterator]() {
    let current = this.from, last = this.to;
    return {
      next: () => current <= last
        ? { value: current++, done: false }
        : { value: undefined, done: true },
    };
  },
};

[...range];                   // [1, 2, 3]
for (const n of range) { }    // 1, 2, 3
const [a, b] = range;         // a = 1, b = 2 — destructuring consumes the protocol too
Array.from(range);            // [1, 2, 3]
Math.max(...range);           // 3
```

Nothing here is array-specific. `for...of`, spread, array destructuring, `Array.from`, `Promise.all`, `new Map(...)`, and `yield*` all consume this one protocol — which is why implementing it once makes your type work with all of them.

### Advanced

**Generators** (`function*`) are the ergonomic way to write iterators: `yield` suspends the function, preserving its entire local state, and resumes exactly there on the next `next()` call.

```js
function* range(from, to) {
  for (let i = from; i <= to; i++) yield i;
}
[...range(1, 3)]; // [1, 2, 3]
```

A generator object is both an iterator *and* iterable, so it drops straight into every consumer above. Two capabilities that plain iterators don't have:

```js
// Infinite sequences are safe, because generators are lazy
function* naturals() { let n = 0; while (true) yield n++; }
naturals().take(5).toArray(); // [0, 1, 2, 3, 4] — ES2025 iterator helper stops the loop

// Delegation with yield*
function* inner() { yield 1; yield 2; }
function* outer() { yield 0; yield* inner(); yield 3; }
[...outer()]; // [0, 1, 2, 3]
```

Generators are also two-way channels: `next(value)` sends a value *in*, which becomes the result of the paused `yield` expression. This is the mechanism async/await was originally built from (and is still how transpilers implement it when targeting older runtimes).

```js
function* dialogue() {
  const name = yield "What is your name?";
  return `Hello, ${name}`;
}
const d = dialogue();
d.next();          // { value: "What is your name?", done: false }
d.next("Sam");     // { value: "Hello, Sam", done: true }
```

### Mastery

**Cleanup is part of the protocol.** If a `for...of` loop exits early — `break`, `return`, or a thrown error — the runtime calls the iterator's optional `return()` method. In a generator, that runs your `finally` block, which is what makes generators safe for wrapping resources:

```js
function* readLines(handle) {
  try {
    while (true) yield handle.readLine();
  } finally {
    handle.close(); // runs on break, throw, or normal completion
  }
}

for (const line of readLines(h)) {
  if (line === "STOP") break; // handle.close() still runs
}
```

Manually calling `.next()` in a loop bypasses this. That is one of the strongest arguments for always consuming iterators through `for...of` or an iterator helper rather than by hand.

**Async iteration** is the same protocol one level up: `[Symbol.asyncIterator]()` returns an iterator whose `next()` returns a *Promise* of `{ value, done }`, and `for await...of` consumes it. `async function*` writes them.

```js
async function* paginate(url) {
  let next = url;
  while (next) {
    const res = await fetch(next);
    const page = await res.json();
    yield* page.items;   // yield each item, not the page
    next = page.nextUrl;
  }
}

for await (const item of paginate("/api/items")) {
  process(item); // backpressure for free: the next page isn't fetched until you need it
}
```

This is the idiomatic shape for paginated APIs, streaming responses, and Node streams (`fs.createReadStream` and every other Node readable stream is async-iterable). Note the important limitation: `for await...of` is **serial by design**. If you want concurrency, collect promises and use `Promise.all` instead — see [Part 8](#8-asynchronous-js).

`Array.fromAsync()` (**ES2026**, Node 22+) is the async counterpart to `Array.from`, draining an async iterable into an array:

```js
const all = await Array.fromAsync(paginate("/api/items"));
```

> **→ In TypeScript:** you will meet `Iterable<T>`, `IterableIterator<T>`, `AsyncIterable<T>`, and the three-parameter `Generator<Yield, Return, Next>` — that third parameter is exactly the `next(value)` channel above, and it is the reason generator types look intimidating until you've seen the two-way example. Typing a custom iterable is a good early generics exercise. Also relevant: the `downlevelIteration` compiler flag exists because faithfully emitting this protocol for ES5 targets is expensive, so TS makes you opt in.

### Part 7 Cheat Sheet

| Concept | Protocol member |
|---|---|
| Iterable | `[Symbol.iterator]()` → iterator |
| Iterator | `next()` → `{ value, done }` |
| Early-exit cleanup | `return()` → runs generator `finally` |
| Async iterable | `[Symbol.asyncIterator]()` → promise-returning iterator |
| Write one easily | `function*` / `async function*` |
| Delegate to another | `yield*` |
| Consume | `for...of` / `for await...of` / spread / `Array.from` / `Array.fromAsync` |
| Process lazily | ES2025 iterator helpers (`.map`, `.filter`, `.take`, `.toArray`) |

[⤴ back to top](#table-of-contents)

---

## 8. Asynchronous JS

The most conceptually load-bearing Part for real-world JavaScript — and the one where TypeScript's `Promise<T>` will make the most sense once you understand the runtime mechanics beneath it.

### Beginner

```js
// Callback (legacy, still present in older Node APIs)
fs.readFile("file.txt", (err, data) => { /* error-first convention */ });

// Promise
fetch(url).then(res => res.json()).then(data => console.log(data));

// async/await — syntax over Promises, not a different mechanism
async function load() {
  const res = await fetch(url);
  const data = await res.json();
  return data; // an async function ALWAYS returns a Promise
}
```

A promise is in exactly one of three states — pending, fulfilled, or rejected — and settles at most once. Nothing can un-settle it.

### Working Knowledge

**The event loop.** JavaScript executes on a single thread. Asynchronous operations (timers, I/O, promise callbacks) do not block it; they are queued and run when the call stack empties.

```
        ┌───────────────────────────────┐
        │  Call Stack (synchronous)     │  runs to completion — nothing interrupts it
        └───────────────┬───────────────┘
                        │  stack empties
                        ▼
        ┌───────────────────────────────┐
        │  Microtask Queue              │  drained COMPLETELY, including
        │  .then/.catch/.finally,       │  microtasks added while draining
        │  await continuations,         │
        │  queueMicrotask()             │
        └───────────────┬───────────────┘
                        │  queue empty
                        ▼
        ┌───────────────────────────────┐
        │  Macrotask Queue              │  ONE task, then back to microtasks
        │  setTimeout, setInterval,     │
        │  setImmediate, I/O callbacks  │
        └───────────────────────────────┘
```

The rule that explains nearly every ordering puzzle: **all microtasks drain before the next macrotask runs.**

```js
console.log("1");
setTimeout(() => console.log("2"), 0);           // macrotask
Promise.resolve().then(() => console.log("3"));  // microtask
queueMicrotask(() => console.log("4"));          // microtask
console.log("5");
// Output: 1, 5, 3, 4, 2
```

Because microtasks drain exhaustively, a microtask that schedules another microtask can starve the macrotask queue entirely — an infinite `Promise.resolve().then(loop)` freezes the process in a way an infinite `setTimeout(loop, 0)` does not.

### Advanced

**Error handling has one non-obvious trap.** A `try/catch` only guards what is lexically inside it, and a promise that starts rejecting *outside* the block is an unhandled rejection regardless of any `catch` elsewhere in the function:

```js
// Wrong — the promise is created outside the try
async function load() {
  const p = fetch(url);          // starts now; if it rejects before we await, it's unhandled
  doOtherSyncWorkThatThrows();   // if this throws, we never reach the try at all
  try {
    await p;
  } catch (e) { /* ... */ }
}

// Right — the whole async sequence, creation included, sits inside the try
async function load() {
  try {
    const p = fetch(url);
    doOtherSyncWorkThatThrows();
    await p;
  } catch (e) { /* ... */ }
}
```

This is a [well-documented gotcha](https://advancedweb.hu/how-to-avoid-uncaught-async-errors-in-javascript/), and it matters more in Node than it used to: since Node 15, an unhandled rejection **terminates the process by default** (`--unhandled-rejections=throw`) rather than printing a warning.

**Combinators** — pick based on what partial failure should mean:

| Combinator | Settles when | On rejection |
|---|---|---|
| `Promise.all` | all fulfill | rejects immediately with the first error (others keep running) |
| `Promise.allSettled` | all settle | never rejects; returns `{status, value}` / `{status, reason}` |
| `Promise.race` | first settles | rejects if the first to settle rejected |
| `Promise.any` | first fulfills | rejects only if all reject, with an `AggregateError` |

```js
const results = await Promise.allSettled([fetchA(), fetchB()]);
// [{ status: "fulfilled", value: ... }, { status: "rejected", reason: ... }]
```

Note the sharp edge in `Promise.all`: rejecting fast does **not** cancel the other promises. They run to completion, and if one of them also rejects, that rejection is unhandled. Use `allSettled` when every operation must be accounted for.

**Cancellation** is not built into promises — `AbortController` is the standard mechanism, and it is understood by `fetch`, most Node APIs, and increasingly by libraries:

```js
const ac = new AbortController();
const timer = setTimeout(() => ac.abort(), 5000);
try {
  const res = await fetch(url, { signal: ac.signal });
} catch (e) {
  if (e.name === "AbortError") { /* we cancelled it */ } else { throw e; }
} finally {
  clearTimeout(timer);
}

// Or, for the common timeout case:
await fetch(url, { signal: AbortSignal.timeout(5000) });
AbortSignal.any([userSignal, AbortSignal.timeout(5000)]); // first one to fire wins
```

**Real Scenario:** a Node service issues 500 independent `fetch` calls inside a `for` loop with `await` on each, turning concurrent I/O into serial I/O — a request that should take 200ms takes 100 seconds:

```js
// Wrong — serializes 500 independent network calls
for (const id of ids) { results.push(await fetchUser(id)); }

// Right — all in flight at once
const results = await Promise.all(ids.map(fetchUser));

// Right, with a concurrency limit (500 sockets at once will get you rate-limited)
async function mapLimit(items, limit, fn) {
  const out = new Array(items.length);
  let i = 0;
  await Promise.all(Array.from({ length: limit }, async () => {
    while (i < items.length) { const idx = i++; out[idx] = await fn(items[idx]); }
  }));
  return out;
}
const limited = await mapLimit(ids, 10, fetchUser);
```

The serial loop is *correct* when each iteration depends on the previous one, or when you deliberately want backpressure. It is a bug when the iterations are independent.

### Mastery

**`await` does not block the thread.** It suspends the async function, registers the rest of the function body as a microtask continuation on the awaited promise, and returns control to the caller immediately. That is why calling an async function returns a `Promise` *synchronously*, before any `await` inside it has settled — and why the code up to the first `await` runs synchronously:

```js
async function f() {
  console.log("A");        // synchronous — runs on the caller's stack
  await null;              // suspends here, even for a non-promise
  console.log("C");        // microtask continuation
}
f();
console.log("B");
// Output: A, B, C
```

`await` on a non-promise still costs one microtask tick, because the value is wrapped via `Promise.resolve` semantics.

**`Promise.try` (ES2025)** closes a long-standing gap: it runs a function and captures *synchronous* throws as a rejection, so a mixed sync/async function has one uniform error channel.

```js
// Wrong — a synchronous throw inside maybeAsync escapes the promise chain
maybeAsync(x).then(ok).catch(handle);

// Right (ES2025)
Promise.try(() => maybeAsync(x)).then(ok).catch(handle);
```

**`Promise.withResolvers()` (ES2024)** replaces the "deferred" pattern of hoisting `resolve`/`reject` out of the executor:

```js
const { promise, resolve, reject } = Promise.withResolvers();
socket.on("message", resolve);
socket.on("error", reject);
await promise;
```

**Top-level `await`** is legal in ES modules (never in CommonJS). A module using it delays its own evaluation — and the evaluation of every module importing it — until the awaited promise settles ([Node.js ESM docs](https://nodejs.org/api/esm.html)). That is a genuine deadlock risk in a cyclic graph, and a genuine startup-latency cost in a deep one.

**`await using` (ES2026)** brings scope-based resource cleanup, the async analogue of `try/finally` — the resource's `[Symbol.asyncDispose]()` runs when the block exits, however it exits:

```js
await using conn = await pool.connect(); // released automatically at end of scope
const rows = await conn.query("SELECT 1");
```

TypeScript has supported `using`/`await using` since 5.2 with downlevel emit, so this is available to you in TS ahead of full runtime support.

> **→ In TypeScript:** an `async` function's return type is `Promise<T>` and TS enforces it — you cannot annotate one as returning `T`. Awaited values are unwrapped via the `Awaited<T>` utility type, which flattens nested promises exactly as the runtime does. The uncomfortable part: `catch (e)` is typed `unknown` under `useUnknownInCatchVariables` (on with `strict`), because JS lets you `throw` any value — so you must narrow before use, which is precisely the `instanceof Error` check from [Part 10](#10-equality-errors--edge-cases). TS also cannot detect a floating promise on its own; that is the `@typescript-eslint/no-floating-promises` lint rule's job, and it is worth enabling on day one.

### Part 8 Cheat Sheet

| Situation | Tool |
|---|---|
| Independent operations | `Promise.all(items.map(fn))` |
| Need every result, failures included | `Promise.allSettled` |
| First success wins | `Promise.any` |
| First settle wins (timeouts) | `Promise.race` |
| Timeout a fetch | `AbortSignal.timeout(ms)` |
| Cancel on demand | `AbortController` + `signal` |
| Externally settled promise | `Promise.withResolvers()` (ES2024) |
| Uniform errors from a maybe-sync fn | `Promise.try()` (ES2025) |
| Guaranteed async cleanup | `await using` (ES2026) / `try...finally` |
| Streaming or paginated source | `for await...of` ([Part 7](#7-iterators-and-generators)) |
| Ordering question | microtasks all drain before the next macrotask |

[⤴ back to top](#table-of-contents)

---

## 9. Modules

Directly relevant to TypeScript: your `tsconfig.json` `module` and `moduleResolution` settings configure exactly this system, and misconfiguring them is the single most common source of "why won't my import resolve" pain.

### Beginner

```js
// math.js
export function add(a, b) { return a + b; }
export default function multiply(a, b) { return a * b; }

// main.js
import multiply, { add } from "./math.js";
import * as math from "./math.js";
import { add as plus } from "./math.js";
```

Note the file extension in `"./math.js"`. It is **required** in ESM for relative imports — unlike CommonJS, which guesses.

### Working Knowledge

Two module systems coexist in Node: **CommonJS** (`require`/`module.exports`, the historical default) and **ESM** (`import`/`export`, the standard). New code should be ESM; a great deal of production code is still CJS, and you will meet both.

```js
// CommonJS
const { add } = require("./math.js");
module.exports = { add };

// ESM — requires "type": "module" in package.json, or an .mjs extension
import { add } from "./math.js";
export { add };
```

```js
// Wrong — mixing the two in one ESM file
import fs from "node:fs";
const path = require("path"); // ReferenceError: require is not defined in ES module scope

// Right — one system per file; use createRequire only if you truly must
import fs from "node:fs";
import path from "node:path";
```

Prefix Node built-ins with `node:` (`node:fs`, `node:path`). It is unambiguous, cannot be shadowed by an npm package of the same name, and is required for some newer built-ins like `node:test`.

### Advanced

The behavioral differences that actually bite:

- **CJS is dynamic and synchronous.** `require()` runs anywhere — inside an `if`, inside a function, computed at runtime. **ESM `import` is static**: hoisted, resolved before any code runs, allowed only at the top level. For conditional or lazy loading, use dynamic `import()`, which returns a Promise and works in both systems.
- **ESM exports are live bindings, not copies.** An imported value reflects later changes made by the exporting module. CJS exports a value snapshot at `require()` time for primitives.

```js
// counter.mjs
export let count = 0;
export function increment() { count++; }

// main.mjs
import { count, increment } from "./counter.mjs";
console.log(count); // 0
increment();
console.log(count); // 1 — the binding is live; a CJS require() would still show 0
```

- **Imports are read-only.** You cannot assign to an imported binding (`count = 5` is a `TypeError`), which is why testing libraries had to invent module mocking rather than just overwriting exports.
- **`require(esm)` now works.** As of Node 22.12+ and Node 24, a CommonJS file can synchronously `require()` an ES module, provided that module has no top-level `await`. This is the single biggest reduction in interop pain in years, and it is stable — not flagged — in current LTS ([Node.js modules docs](https://nodejs.org/api/modules.html#loading-ecmascript-modules-using-require)).

**Real Scenario:** a package publishes both CJS and ESM builds, and a bundler resolves both into the same app — the "dual package hazard." Two copies of the module now exist with separate state, so `instanceof` checks across the boundary fail and module-level singletons (a database pool, a config cache) silently duplicate. The fix is on the package side: a single ESM implementation with a thin CJS wrapper, or an `exports` map that cannot resolve to both.

### Mastery

Module **resolution** — how a bare specifier like `"lodash"` becomes a file — differs fundamentally between the two systems. CJS walks `node_modules` directories upward, checking `main`, then index files, then extensions. ESM consults the package's `exports` map, which is an explicit, ordered set of **conditions**:

```json
{
  "name": "my-lib",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "default": "./dist/index.mjs"
    },
    "./utils": "./dist/utils.mjs"
  }
}
```

Three things follow, all of which you will hit in TypeScript:

1. **`exports` is an encapsulation boundary.** Once a package has an `exports` map, deep imports (`my-lib/dist/internal.js`) fail unless explicitly listed. This breaks a lot of older code that reached into `node_modules` internals.
2. **`types` must come first.** Conditions are matched in order, so a `types` entry placed after `import` will never be reached. This is the most common cause of "the package works but TypeScript can't find its types."
3. **`moduleResolution` must match your runtime.** Modern TS projects want `"module": "nodenext"` (or `"bundler"` when a bundler owns resolution) — the legacy `"node"` setting predates `exports` maps entirely and will happily resolve files your runtime cannot.

> **→ In TypeScript:** everything above is `tsconfig.json` surface. `module`/`moduleResolution` select the algorithm; `verbatimModuleSyntax` stops TS from silently rewriting your import syntax; `import type { Foo }` marks an import as type-only so it is fully erased (essential when the imported module has side effects or doesn't exist at runtime); `allowSyntheticDefaultImports`/`esModuleInterop` paper over CJS's lack of a real default export; and `.d.ts` files are resolved through the `types` condition described above.

### Part 9 Cheat Sheet

| | CommonJS | ESM |
|---|---|---|
| Syntax | `require` / `module.exports` | `import` / `export` |
| Loading | synchronous, dynamic | static, asynchronous graph |
| Allowed anywhere? | yes | top level only (use `import()` otherwise) |
| Exports are | value snapshots | live, read-only bindings |
| Top-level `await` | no | yes |
| File extension in path | optional | required (relative imports) |
| `__dirname` | yes | no — use `import.meta.dirname` (Node 21.2+) |
| Opt in via | default | `"type": "module"` or `.mjs` |
| Resolution driver | `node_modules` walk + `main` | `exports` conditions |

[⤴ back to top](#table-of-contents)

---

## 10. Equality, Errors & Edge Cases

### Beginner

```js
null == undefined;  // true  — the one legitimate use of ==
null === undefined; // false — different types
```

`null` means "intentionally empty" — a value you set. `undefined` means "never set" — the language's default for unset variables, missing object properties, missing arguments, and functions with no `return`.

Practically: JSON has `null` but not `undefined`, so anything crossing an API boundary comes back as `null`; and `JSON.stringify` *drops* `undefined` properties entirely rather than serializing them.

```js
JSON.stringify({ a: undefined, b: null }); // '{"b":null}' — `a` vanished
```

### Working Knowledge

```js
// Wrong — swallows the error, destroys the stack trace, hides the real bug
try {
  doWork();
} catch (e) {
  console.log("error");
}

// Right — preserve the original, add context
try {
  doWork();
} catch (e) {
  throw new Error("doWork failed while syncing user 42", { cause: e });
}
```

If you genuinely don't need the error object, ES2019 lets you omit the binding: `try { ... } catch { ... }`.

**Optional chaining and nullish coalescing** (ES2020) remove a whole category of `undefined`-access crashes:

```js
user?.address?.city ?? "unknown"; // short-circuits to undefined, then falls back
user.getName?.();                 // call only if the method exists
arr?.[0];                         // index only if arr is non-nullish
obj.count ??= 0;                  // logical assignment (ES2021): assign only if nullish
```

**Real Scenario:** `user.settings.theme || "light"` looks like a safe default until a user explicitly chooses a value that happens to be falsy — an empty string for "no preference," `0` for a volume slider, `false` for a toggle. `||` overrides all of them. `??` falls back only on `null`/`undefined`, which is what "no value was provided" actually means. This is the [Part 1](#1-values-types--coercion) falsy problem resurfacing at the API layer, and it is the most common one-character bug fix in modern JS review.

```js
settings.volume || 50;  // a user-chosen 0 becomes 50 — wrong
settings.volume ?? 50;  // 0 is respected — right
```

Note that `??` cannot be mixed with `||` or `&&` without parentheses — a deliberate syntax error, because the precedence would be ambiguous.

### Advanced

**Custom errors** should extend `Error` and set `.name`, so that `instanceof` checks work and stack traces stay useful:

```js
class ValidationError extends Error {
  constructor(message, field, options) {
    super(message, options);
    this.name = "ValidationError";
    this.field = field;
  }
}

try {
  throw new ValidationError("bad email", "email");
} catch (e) {
  if (e instanceof ValidationError) { /* handle */ }
}
```

`Error.cause` (ES2022) chains errors across layers without losing the original — the alternative to either swallowing it or stringifying it into a message:

```js
throw new Error("failed to load user", { cause: originalError });
// Node's console prints the full chain, including "[cause]: ..."
```

**Anything can be thrown**, which is why defensive error handling cannot assume it has an `Error`:

```js
// Wrong — assumes the thrown value is an Error
catch (e) { logger.error(e.message); } // undefined if someone threw a string

// Right
catch (e) {
  const err = e instanceof Error ? e : new Error(String(e));
  logger.error(err.message, { stack: err.stack });
}
```

`Error.isError()` (**ES2026**, Node 24+) improves on `instanceof Error` for the cross-realm case — errors passed between a worker, an iframe, or a Node `vm` context fail `instanceof` because each realm has its own `Error` constructor.

The **three equality algorithms** are worth holding in your head together:

| Comparison | `NaN` vs `NaN` | `+0` vs `-0` | Coerces types | Where it's used |
|---|---|---|---|---|
| `==` | `false` | `true` | yes | avoid, except `x == null` |
| `===` | `false` | `true` | no | your default; `indexOf`, `switch` |
| `Object.is` | `true` | `false` | no | when `-0`/`NaN` distinctions matter |
| SameValueZero | `true` | `true` | no | `includes`, `Map`/`Set` keys |

### Mastery

**`structuredClone()`** (Node 17+, all modern browsers) is the standard deep-clone primitive, and it replaces the `JSON.parse(JSON.stringify(x))` hack — which silently drops `undefined`, functions, and symbols, converts `Date` to a string, turns `Map`/`Set` into `{}`, and throws on circular references.

```js
const original = { when: new Date(), tags: new Set(["a"]), self: null };
original.self = original;              // circular
const copy = structuredClone(original); // works — Date stays a Date, Set stays a Set, cycle preserved
```

Its limits are real, though: it throws `DataCloneError` on functions, symbols, and DOM nodes, and it does **not** preserve prototypes — a cloned class instance comes back as a plain object, so `copy instanceof MyClass` is `false`. For class instances, write a `clone()` method.

**Deep equality has no built-in operator.** `===` is always reference identity for objects, and the popular `JSON.stringify(a) === JSON.stringify(b)` workaround is wrong in more ways than most people realize: key order matters, `undefined` properties vanish, `NaN` becomes `null`, and circular references throw. Production code uses `node:util`'s `isDeepStrictEqual`, a library (`lodash.isEqual`), or a purpose-written comparison:

```js
import { isDeepStrictEqual } from "node:util";
isDeepStrictEqual({ a: 1 }, { a: 1 }); // true — built into Node, no dependency
```

> **→ In TypeScript:** `strictNullChecks` is the flag that makes this entire Part visible to the compiler — without it, `null` and `undefined` are assignable to everything and `?.`/`??` are decoration. With it on, `user.address` is `Address | undefined` and TS *forces* the optional chain. `catch (e)` is `unknown` under `strict`, so the `e instanceof Error` narrowing above becomes mandatory rather than merely wise. Custom error classes pair naturally with discriminated unions for typed error handling, and `Object.is` is typed but, like everything here, unchanged at runtime.

### Part 10 Cheat Sheet

| Task | Use |
|---|---|
| Default only when absent | `??` (never `\|\|`, unless you truly mean "any falsy") |
| Assign only when absent | `??=` |
| Safe nested read | `?.` |
| Present-at-all check | `x != null` |
| Deep clone | `structuredClone(x)` |
| Deep equality | `isDeepStrictEqual` (Node) or `lodash.isEqual` |
| Wrap a lower-level error | `new Error(msg, { cause: e })` |
| Normalize an unknown throw | `e instanceof Error ? e : new Error(String(e))` |
| Cross-realm error check | `Error.isError(e)` (ES2026) |
| Compare distinguishing `NaN`/`-0` | `Object.is` |

[⤴ back to top](#table-of-contents)

---

## 11. Modern Runtime & Tooling Context

*(Beginner/Working Knowledge only — deep tooling coverage is out of scope here. This Part exists to orient you before the TypeScript docs, where `tsconfig.json` and the compiler pipeline take over. See also [`nodejs/`](../nodejs/), [`vite/`](../vite/), and [`bunjs/`](../bunjs/) in this repo.)*

### Beginner

- **Node.js** runs JavaScript outside the browser — servers, scripts, and tooling. **npm** is its default package manager, driven by `package.json`.
- `package.json`'s `"type"` field decides how `.js` files are interpreted: `"module"` makes them ESM, `"commonjs"` (or omitting the field) makes them CJS. The `.mjs` and `.cjs` extensions override the field per file, which is how packages ship both.
- `npx <package>` runs a package's CLI without installing it globally — the standard way to run one-off tools, including `npx tsc` later.

### Working Knowledge

**Node release lines as of September 2026** ([nodejs/Release](https://github.com/nodejs/Release)):

| Line | Status | Notes |
|---|---|---|
| Node 22 | Maintenance LTS | security/critical fixes only; plan your upgrade |
| Node 24 | **Active LTS** | the default choice for new production work; supported to April 2028 |
| Node 26 | Current | enters LTS in October 2026 |

Two things worth knowing beyond the version numbers. First, Node 22+ and 24 both support `require(esm)` unflagged, which removes most historical interop friction ([Part 9](#9-modules)). Second, the release model itself is changing: [starting with Node 27 in October 2026](https://nodejs.org/en/blog/announcements/evolving-the-nodejs-release-schedule), Node moves to one major release per year, version numbers align with the calendar year, and every release becomes LTS. Node 26 is the last release under the old even/odd model.

**Node runs TypeScript directly now.** Since Node 22.6 (`--experimental-strip-types`) and unflagged by default from Node 23.6 onward, Node strips type annotations from `.ts` files and executes them without a build step. The critical limitation: it *strips* types, it does not *check* them, and it rejects syntax that requires code generation (enums, namespaces, parameter properties) unless you add `--experimental-transform-types`. Treat it as a fast path for scripts, not a replacement for `tsc --noEmit` in CI.

**Where TypeScript enters.** TS is a build-time layer: `tsc` reads `.ts`, checks types, and emits `.js` with every annotation erased. The JavaScript engine never sees a type, and no type exists at runtime to inspect. Everything in this document is what actually executes; TypeScript's entire value proposition is catching the Advanced and Mastery gotchas above *before* the code runs.

The corollary is the most important thing to carry into the TS docs: **if a bug is possible in this document, it is possible in TypeScript too** — TS only makes it harder to write. Float precision, `NaN`, prototype mutation, event-loop ordering, unhandled rejections, and module resolution are all unchanged.

[⤴ back to top](#table-of-contents)

---

## 12. JavaScript to TypeScript Concept Map

A single index of the whole document, read the other direction: each JS mechanic and the TS feature built on top of it. Use it as a checklist — if a row's left column is unclear, revisit that Part before meeting the right column.

| JavaScript mechanic (Part) | What TypeScript builds on it |
|---|---|
| Dynamic value types ([1](#1-values-types--coercion)) | primitive types, literal types, unions, `strict` mode |
| `typeof` / `instanceof` checks ([1](#1-values-types--coercion), [10](#10-equality-errors--edge-cases)) | type guards and narrowing; user-defined predicates (`x is T`) |
| `null` vs `undefined` ([10](#10-equality-errors--edge-cases)) | `strictNullChecks`, optional properties (`?`), non-null assertion (`!`) |
| Coercion and falsiness ([1](#1-values-types--coercion)) | nothing — TS flags *some* misuse but cannot change runtime coercion |
| `const` bindings ([2](#2-variables--scope)) | `as const`, `readonly`, `const` type parameters |
| Closures and scope ([2](#2-variables--scope)) | unchanged; inference just follows the same lexical rules |
| Function parameters and arity ([3](#3-functions)) | parameter/return annotations, optional and default params, overloads |
| `this` binding ([3](#3-functions)) | the `this` parameter, `ThisParameterType`, `strictFunctionTypes` variance |
| Object shapes and prototypes ([4](#4-objects--prototypes)) | **structural typing**, `interface`, `type`, index signatures |
| Classes, `extends`, `#private` ([4](#4-objects--prototypes)) | class types, `implements`, `abstract`, `private` (erased) vs `#` (real) |
| Property descriptors / decorators ([4](#4-objects--prototypes)) | decorators, `Object.defineProperty` typing, `Readonly<T>` |
| Arrays and mutation ([5](#5-arrays--iteration)) | `T[]`, tuples, `readonly T[]`, `ReadonlyArray<T>` |
| `filter` narrowing ([5](#5-arrays--iteration)) | type predicates — the standard fix for `(T \| undefined)[]` |
| `Map` / `Set` ([6](#6-keyed-collections-map-and-set)) | generics, `Map<K, V>`, `get()` returning `V \| undefined` |
| Iterator protocol ([7](#7-iterators-and-generators)) | `Iterable<T>`, `Generator<Y, R, N>`, `downlevelIteration` |
| Promises and `await` ([8](#8-asynchronous-js)) | `Promise<T>`, `Awaited<T>`, `async` return-type enforcement |
| `catch` binding ([8](#8-asynchronous-js), [10](#10-equality-errors--edge-cases)) | `useUnknownInCatchVariables`, narrowing before use |
| CJS vs ESM, `exports` map ([9](#9-modules)) | `module`, `moduleResolution`, `verbatimModuleSyntax`, `import type`, `.d.ts` |
| Runtime type erasure ([11](#11-modern-runtime--tooling-context)) | the reason for schema validators (Zod, Valibot) at every I/O boundary |

The last row deserves emphasis, because it is the lesson people learn slowly and expensively: TypeScript cannot validate data it did not create. Every API response, every `JSON.parse`, every environment variable, and every database row arrives as `any` or a lie you told the compiler. A runtime validator at those boundaries is not belt-and-braces — it is the only thing that makes the types true.

[⤴ back to top](#table-of-contents)

---

## 13. Suggested Learning/Reference Order

1. **[Values, Types & Coercion](#1-values-types--coercion)** — the foundation for everything else; coercion and falsiness explain half of JavaScript's "weirdness" reputation.
2. **[Variables & Scope](#2-variables--scope)** — `let`/`const`/closures are prerequisite vocabulary for Functions.
3. **[Functions](#3-functions)** — especially `this` binding, the most-asked-about JS debugging topic there is.
4. **[Objects & Prototypes](#4-objects--prototypes)** — structural thinking here is your direct on-ramp to TS interfaces.
5. **[Arrays & Iteration](#5-arrays--iteration)** — the mutate-vs-new table alone prevents a lot of real bugs.
6. **[Asynchronous JS](#8-asynchronous-js)** — jump ahead; this is the highest-payoff Part for production code, and it does not depend on Parts 6–7. Budget real time here.
7. **[Equality, Errors & Edge Cases](#10-equality-errors--edge-cases)** — short, but `??` vs `||` is a frequent live-bug source.
8. **[Keyed Collections](#6-keyed-collections-map-and-set)** and **[Iterators and Generators](#7-iterators-and-generators)** — circle back once async is solid; generators make far more sense after `await`.
9. **[Modules](#9-modules)** — the CJS/ESM split maps directly onto TS's `module` config, so read it right before you configure a project.
10. **[Modern Runtime & Tooling Context](#11-modern-runtime--tooling-context)** and **[the Concept Map](#12-javascript-to-typescript-concept-map)** — read last, as the bridge into the TypeScript docs.

[⤴ back to top](#table-of-contents)

---

## 14. Quick Self-Check

Answer without looking back. Anything shaky, revisit that Part before starting TypeScript.

1. Name all eight falsy values. Why is `[]` not one of them?
2. Why does `typeof null` return `"object"`, and what is the correct object check?
3. Why is `0.1 + 0.2 !== 0.3`, and what should you store money in instead?
4. What does `for (var i = 0; i < 3; i++) { setTimeout(() => console.log(i), 0); }` print, and why does `let` change it?
5. Why does a class method break when passed as a callback, and what are the two standard fixes?
6. Why can't an arrow function be used as an object method when you need `this`?
7. Why does `c1.count++` on a prototype property not affect `c2`, while `c1.log.push("x")` does?
8. When should you use a `Map` instead of a plain object? Name two reasons that aren't style.
9. What does `[NaN].includes(NaN)` return, and what does `[NaN].indexOf(NaN)` return? Why do they differ?
10. Which of these mutate: `sort`, `map`, `splice`, `toSorted`, `filter`, `reverse`, `with`?
11. What is the output order of a `console.log`, a `setTimeout(fn, 0)`, and a `Promise.resolve().then(fn)` in the same tick?
12. Why can a `try/catch` around an `await` still produce an unhandled rejection?
13. What is wrong with `for (const id of ids) results.push(await fetchUser(id))`, and when is it actually correct?
14. How do you cancel an in-flight `fetch`, and how do you give it a 5-second timeout?
15. What is a "live binding," and which module system has them?
16. Why must the `types` condition come first in a package's `exports` map?
17. What does `??` do that `||` doesn't, and give a case where it matters?
18. What does `structuredClone` handle that `JSON.parse(JSON.stringify(x))` doesn't — and what does it still get wrong?
19. Why is `#privateField` a stronger guarantee than TypeScript's `private`?
20. Your API returns JSON that your TS interface says is `User`. What guarantees that it actually is?

[⤴ back to top](#table-of-contents)
