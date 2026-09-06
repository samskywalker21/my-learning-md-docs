# JavaScript — Foundations (Parts 1–3)

Values and types; variables and scope; operators, coercion and control flow. The substrate every other doc in this set stands on.

> **About this document.** Part of the [JavaScript Mastery Guide](./javascript-mastery-guide.md) set — see that document's "About This Document" section for the framing (curiosity-driven, hands-on), scope, tier conventions, sourcing rule and update instructions. Written against ECMAScript 2026; Baseline data verified September 6, 2026.

---

## Table of Contents

1. [Part 1 — Values & Types](#1-part-1--values--types)
   - [Beginner](#part-1-beginner) · [Working Knowledge](#part-1-working) · [Advanced](#part-1-advanced) · [Mastery](#part-1-mastery) · [Cheat sheet](#part-1-cheatsheet)
2. [Part 2 — Variables, Scope & the TDZ](#2-part-2--variables-scope--the-tdz)
   - [Beginner](#part-2-beginner) · [Working Knowledge](#part-2-working) · [Advanced](#part-2-advanced) · [Mastery](#part-2-mastery) · [Cheat sheet](#part-2-cheatsheet)
3. [Part 3 — Operators, Coercion & Control Flow](#3-part-3--operators-coercion--control-flow)
   - [Beginner](#part-3-beginner) · [Working Knowledge](#part-3-working) · [Advanced](#part-3-advanced) · [Mastery](#part-3-mastery) · [Cheat sheet](#part-3-cheatsheet)

---

## 1. Part 1 — Values & Types

<a id="part-1"></a>

### Beginner — the eight types

<a id="part-1-beginner"></a>

Every value in JavaScript is one of exactly eight types. Seven are **primitives** — immutable, compared by value. The eighth is **object** — mutable, compared by reference.

| Type | Example | Primitive? |
|---|---|---|
| `number` | `42`, `3.14`, `NaN`, `Infinity` | ✅ |
| `string` | `"hi"`, `` `hi ${name}` `` | ✅ |
| `boolean` | `true`, `false` | ✅ |
| `undefined` | `undefined` | ✅ |
| `null` | `null` | ✅ |
| `bigint` | `9007199254740993n` | ✅ |
| `symbol` | `Symbol("id")` | ✅ |
| `object` | `{}`, `[]`, `function(){}`, `new Date()` | ❌ |

Note what is *not* on that list: there is no separate array type, no separate function type, no integer/float distinction. Arrays and functions are objects. All non-BigInt numbers are IEEE-754 doubles.

```js
typeof 42;            // "number"
typeof "hi";          // "string"
typeof true;          // "boolean"
typeof undefined;     // "undefined"
typeof 10n;           // "bigint"
typeof Symbol();      // "symbol"
typeof {};            // "object"
typeof [];            // "object"   ← arrays are objects
typeof function(){};  // "function" ← the one special case
typeof null;          // "object"   ← a bug, kept for compatibility
```

**`typeof null === "object"` is a mistake from 1995** that was never fixed because fixing it would break the web ([MDN: typeof](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/typeof)). Never use `typeof` to test for `null`:

```js
// ❌ Never detects null.
if (typeof value === "null") { /* unreachable */ }

// ✅ Compare directly.
if (value === null) { /* … */ }

// ✅ Or catch both null and undefined at once.
if (value == null) { /* null or undefined, nothing else */ }
```

That last one is the single legitimate use of loose `==` — covered in [Part 3](#part-3-advanced).

> **Try It.** In a REPL, run each `typeof` line above. Then try `typeof undeclaredVariable` — it returns `"undefined"` rather than throwing, which is the only place in the language where touching an undeclared name is safe.

### `undefined` vs. `null`

Both mean "no value", and the difference is intent:

- **`undefined`** — the language's absence. A variable declared but not assigned, a missing function argument, a missing property, a function with no `return`.
- **`null`** — *your* absence. A deliberate "there is intentionally nothing here", set by you.

```js
let a;
console.log(a);                    // undefined — never assigned

const user = { name: "Sam" };
console.log(user.email);           // undefined — property absent

function f() {}
console.log(f());                  // undefined — no return

const selected = null;             // "deliberately nothing selected"
```

The practical rule: never assign `undefined` yourself. Let the language produce it, and use `null` when you mean it.

### Working Knowledge — the number type and its edges

<a id="part-1-working"></a>

Every `number` is a 64-bit float. That has three consequences you will hit within your first month.

**1. Decimal arithmetic is approximate.**

```js
0.1 + 0.2;              // 0.30000000000000004
0.1 + 0.2 === 0.3;      // false
```

This is not a JavaScript bug — it is IEEE-754, shared by Python, Java, C and almost everything else. `0.1` is not exactly representable in binary any more than `1/3` is in decimal.

```js
// ❌ Comparing floats directly.
if (total === 0.3) { /* may never run */ }

// ✅ Compare within a tolerance.
const closeEnough = (a, b, eps = Number.EPSILON * 8) => Math.abs(a - b) < eps;
closeEnough(0.1 + 0.2, 0.3);   // true

// ✅ Better for money: work in integer minor units.
const cents = 10 + 20;         // 30 cents — exact
```

**2. Integers are exact only up to 2^53 − 1.**

```js
Number.MAX_SAFE_INTEGER;              // 9007199254740991
Number.MAX_SAFE_INTEGER + 1;          // 9007199254740992
Number.MAX_SAFE_INTEGER + 2;          // 9007199254740992  ← same! silently wrong
```

Beyond that, use `BigInt`:

```js
9007199254740991n + 2n;               // 9007199254740993n — exact
Number.isSafeInteger(9007199254740993); // false
```

`BigInt` and `number` do not mix in arithmetic — `1n + 1` throws a `TypeError`. That is deliberate: implicit conversion would silently lose precision.

**3. `NaN` is not equal to itself.**

```js
NaN === NaN;            // false
Number.isNaN(NaN);      // true   ← the correct test
isNaN("hello");         // true   ← global isNaN coerces first. Avoid.
Number.isNaN("hello");  // false  ← "hello" is not the NaN value
```

Prefer `Number.isNaN` over the global `isNaN` in all new code — the global coerces its argument first, which is almost never what you want ([MDN: Number.isNaN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/isNaN)).

> **Try It.** Run `0.1 + 0.2` and then `(0.1 + 0.2).toFixed(20)`. Expected output: `0.30000000000000004441`. The extra digits are the actual stored value — `toFixed(2)` was just hiding them.

### Strings

Strings are immutable sequences of UTF-16 code units. Every "mutation" produces a new string.

```js
let s = "hello";
s[0] = "H";        // silently does nothing (throws in strict mode? no — silently fails)
console.log(s);    // "hello"
s = "H" + s.slice(1);
console.log(s);    // "Hello"
```

Template literals are the default in modern code — they interpolate and span lines:

```js
const name = "Sam", count = 3;
const msg = `Hi ${name}, you have ${count} item${count === 1 ? "" : "s"}.`;
// "Hi Sam, you have 3 items."

const block = `line one
line two`;         // real newline, no \n needed
```

The UTF-16 detail bites with emoji and non-BMP characters, because `.length` counts code *units*, not characters:

```js
"café".length;       // 4
"👋".length;          // 2   ← one character, two code units
[..."👋"].length;     // 1   ← spreading iterates by code point
Array.from("👋").length; // 1
```

```js
// ❌ Reversing with split("") corrupts non-BMP characters.
"ab👋".split("").reverse().join("");   // "\udc4b\ud83d ba" — mojibake

// ✅ Spread iterates by code point.
[..."ab👋"].reverse().join("");        // "👋ba"
```

Even the spread version is not fully correct for grapheme clusters (family emoji, combining accents) — for that use [`Intl.Segmenter`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Segmenter), covered in [Part 12](./javascript-regex-dates-intl.md).

### Advanced — primitives, wrapper objects, and autoboxing

<a id="part-1-advanced"></a>

Primitives have no properties. So why does `"hi".toUpperCase()` work?

Because the engine temporarily wraps the primitive in an object (`String`), calls the method, and throws the wrapper away. This is **autoboxing**.

```text
"hi".toUpperCase()

  "hi"  ──wrap──▶  new String("hi")  ──call──▶  "HI"
                          │
                          └── discarded immediately
```

The wrapper being discarded is exactly why assigning to a primitive's property silently does nothing:

```js
const s = "hi";
s.custom = 5;        // writes to a throwaway wrapper
console.log(s.custom); // undefined — the wrapper is already gone
```

```js
// ❌ Never construct wrappers explicitly.
const n = new Number(5);
typeof n;            // "object"
n === 5;             // false
if (new Boolean(false)) { console.log("runs!"); }  // objects are always truthy

// ✅ Use the functions as converters, without `new`.
const n2 = Number("5");   // 5, a real primitive
const b2 = Boolean("");   // false
```

`new Boolean(false)` being truthy is a classic interview trap and a real bug source. The rule is simple: **`String`, `Number`, and `Boolean` are conversion functions, never constructors.**

### Type detection that actually works

`typeof` is too coarse (everything object-shaped is `"object"`) and `instanceof` breaks across realms (an array from an iframe or a Node `vm` context is not `instanceof` your `Array`). The robust general-purpose check:

```js
const typeOf = (v) => Object.prototype.toString.call(v).slice(8, -1);

typeOf(null);          // "Null"
typeOf([]);            // "Array"
typeOf(new Date());    // "Date"
typeOf(/x/);           // "RegExp"
typeOf(new Map());     // "Map"
typeOf(Promise.resolve()); // "Promise"
```

For the common cases prefer the purpose-built checks, which are clearer and realm-safe:

```js
Array.isArray(value);         // the correct array test, always
Number.isInteger(value);
Number.isFinite(value);
value === null;
typeof value === "function";
```

### Mastery — `Object.is`, `-0`, and the three equality algorithms

<a id="part-1-mastery"></a>

The spec defines three distinct sameness algorithms, and knowing which one a given API uses explains several otherwise inexplicable results.

| Algorithm | Surfaced as | `NaN` vs `NaN` | `0` vs `-0` |
|---|---|---|---|
| Loose equality | `==` | `false` | equal |
| Strict equality | `===` | `false` | equal |
| SameValueZero | `includes`, `Map`/`Set` keys | **`true`** | equal |
| SameValue | `Object.is` | **`true`** | **not equal** |

```js
const arr = [NaN];
arr.indexOf(NaN);      // -1     ← indexOf uses ===
arr.includes(NaN);     // true   ← includes uses SameValueZero

Object.is(NaN, NaN);   // true
Object.is(0, -0);      // false
0 === -0;              // true

new Set([NaN, NaN]).size;  // 1  ← SameValueZero, so the two NaNs are one key
```

`-0` exists because IEEE-754 has a signed zero. It usually does not matter, but it surfaces when a computation approaches zero from below:

```js
Math.round(-0.4);      // -0
1 / Math.round(-0.4);  // -Infinity   ← not Infinity
(-0).toString();       // "0"  — hides it, which is why it is hard to spot
```

> **Try It.** Run `const m = new Map([[NaN, "found"]]); m.get(NaN)`. Expected: `"found"`. Then run `[NaN].indexOf(NaN)`. Expected: `-1`. Same value, two different lookup algorithms — that is the whole table above in two lines.

### Symbols as a primitive

`Symbol` is the seventh primitive, and its only job is to be a guaranteed-unique property key.

```js
const id = Symbol("id");         // the string is a description, not an identity
Symbol("id") === Symbol("id");   // false — every call makes a new one

const user = { name: "Sam", [id]: 123 };
Object.keys(user);               // ["name"]  ← symbol keys are skipped
JSON.stringify(user);            // '{"name":"Sam"}'
user[id];                        // 123
```

That invisibility to `Object.keys` and `JSON.stringify` is the point: symbol-keyed metadata cannot collide with, or be accidentally serialised alongside, ordinary data. Full treatment including the well-known symbols is in [Part 13](./javascript-metaprogramming-modern.md).

### Part 1 quick reference

<a id="part-1-cheatsheet"></a>

| Task | Do this | Not this |
|---|---|---|
| Test for `null` | `v === null` | `typeof v === "null"` |
| Test for `null` or `undefined` | `v == null` | `!v` (catches `0`, `""`) |
| Test for `NaN` | `Number.isNaN(v)` | `isNaN(v)`, `v === NaN` |
| Test for an array | `Array.isArray(v)` | `typeof v === "object"` |
| Test for an integer | `Number.isInteger(v)` | `v % 1 === 0` |
| Convert to number | `Number(v)` | `new Number(v)` |
| Precise big integers | `BigInt` | `number` past 2^53 |
| Money | Integer minor units | Floats |
| Count characters | `[...str].length` | `str.length` |
| Identity incl. `NaN`/`-0` | `Object.is(a, b)` | `a === b` |

[↑ Back to top](#table-of-contents)

---

## 2. Part 2 — Variables, Scope & the TDZ

<a id="part-2"></a>

### Beginner — `const` by default, `let` when you must, never `var`

<a id="part-2-beginner"></a>

```js
const MAX = 100;    // cannot be reassigned
let count = 0;      // can be reassigned
count += 1;
var old = 1;        // legacy — see below
```

The rule that survives contact with real code: **`const` by default; switch to `let` only when you actually reassign; never write `var` in new code.** `const` is not a performance optimisation — it is a message to the next reader that this name never changes, which removes a thing they have to track.

**The single most common `const` misunderstanding:** `const` freezes the *binding*, not the *value*.

```js
const user = { name: "Sam" };
user.name = "Alex";      // ✅ fine — mutating the object
user.age = 30;           // ✅ fine — adding a property
user = {};               // ❌ TypeError: Assignment to constant variable

const nums = [1, 2];
nums.push(3);            // ✅ fine — nums is [1, 2, 3]
nums = [];               // ❌ TypeError
```

```text
const user  ──▶ [ reference ] ──▶ { name: "Sam" }
    ▲                                    ▲
    │                                    │
 frozen: cannot point                mutable: contents
 at a different object               can change freely
```

If you want the object itself immutable, that is a separate operation:

```js
const config = Object.freeze({ retries: 3 });
config.retries = 5;      // silently ignored (throws in strict mode)
console.log(config.retries); // 3
```

`Object.freeze` is shallow — nested objects stay mutable. See [Part 5](./javascript-objects-classes.md).

### Working Knowledge — scope, hoisting, and the temporal dead zone

<a id="part-2-working"></a>

A **scope** is a region where a name is visible. JavaScript has three: global, function, and block (`{ … }`).

The decisive difference between `var` and `let`/`const` is that **`var` is function-scoped and `let`/`const` are block-scoped.**

```js
function demo() {
  if (true) {
    var  a = 1;
    let  b = 2;
    const c = 3;
  }
  console.log(a);   // 1          — var leaked out of the block
  console.log(b);   // ReferenceError: b is not defined
}
```

`var` also **hoists** its declaration to the top of the function and initialises it to `undefined`, which produces code that runs when it should not:

```js
// ❌ var: reading before the declaration silently gives undefined.
function f() {
  console.log(x);   // undefined — no error, and now you have a bug downstream
  var x = 5;
}
```

```js
// ✅ let/const: reading before the declaration throws immediately.
function g() {
  console.log(x);   // ReferenceError: Cannot access 'x' before initialization
  let x = 5;
}
```

`let` and `const` are hoisted too — the binding is created at the top of the block — but it is left **uninitialised**. The stretch between the top of the block and the declaration line is the **temporal dead zone (TDZ)**, and touching the name inside it throws ([MDN: TDZ](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let#temporal_dead_zone_tdz)).

```text
function g() {
  ┌─────────────────────────────────┐
  │ TDZ for x begins (block entry)  │
  │                                 │
  │   console.log(x);  ← throws     │
  │                                 │
  │   let x = 5;       ← TDZ ends   │
  │                                 │
  │   console.log(x);  ← 5          │
  └─────────────────────────────────┘
}
```

The TDZ is a feature, not an inconvenience: it converts "silently `undefined`, fails somewhere else" into "throws at the exact line where you made the mistake".

> **Try It.** Run this and predict the output first:
> ```js
> var x = 1;
> {
>   var x = 2;
> }
> console.log(x);
> ```
> Expected: `2` — `var` ignored the block entirely. Now change both `var`s to `let`. Expected: `1` — the inner `x` is a separate, block-scoped binding.

### The classic loop bug

This is the most-asked JavaScript question on Stack Overflow, and `let` fixed it outright ([the canonical thread](https://stackoverflow.com/questions/750486/javascript-closure-inside-loops-simple-practical-example)).

```js
// ❌ var: one binding shared by all three callbacks.
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// 3, 3, 3 — by the time the callbacks run, the single `i` is 3
```

```js
// ✅ let: a fresh binding per iteration.
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// 0, 1, 2
```

`for (let …)` creates a **new binding for each iteration** and copies the previous value into it — that is a specific, deliberate rule in the spec, not a side effect of block scoping. Understanding *why* requires closures; see [Part 4](./javascript-functions-closures.md#part-4-advanced).

Before `let`, the workaround was an IIFE, which you will still see in older code:

```js
// Legacy pattern — recognise it, don't write it.
for (var i = 0; i < 3; i++) {
  (function (j) {
    setTimeout(() => console.log(j), 0);
  })(i);
}
```

### Advanced — the scope chain and shadowing

<a id="part-2-advanced"></a>

When a name is used, the engine looks in the current scope, then the enclosing one, and so on out to global. The chain is fixed by **where the code is written**, not by where it is called from — this is *lexical* (static) scoping.

```js
const level = "global";

function outer() {
  const level = "outer";

  function inner() {
    console.log(level);   // "outer" — found one level up
  }

  return inner;
}

const fn = outer();
fn();                     // "outer", even though it is called from global scope
```

```text
┌─ Global scope ────────────────────────────┐
│  level = "global"                         │
│  outer                                    │
│                                           │
│  ┌─ outer() scope ──────────────────────┐ │
│  │  level = "outer"   ← inner finds this│ │
│  │                                      │ │
│  │  ┌─ inner() scope ─────────────────┐ │ │
│  │  │  (no `level` here)              │ │ │
│  │  │  console.log(level) ──lookup──┐ │ │ │
│  │  └───────────────────────────────┼─┘ │ │
│  └──────────────────────────────────┼───┘ │
└─────────────────────────────────────┴─────┘
```

**Shadowing** is an inner declaration hiding an outer one. It is legal and often fine, but shadowing a name you still need in the same function is a reliable source of confusion:

```js
// ❌ Shadowing that hurts: which `user` is which?
function update(user) {
  if (user.isAdmin) {
    const user = fetchAdminRecord();   // shadows the parameter
    save(user);                        // saves the admin record…
  }
  log(user);                           // …but logs the parameter. Easy to misread.
}

// ✅ Distinct names.
function update(user) {
  if (user.isAdmin) {
    const adminRecord = fetchAdminRecord();
    save(adminRecord);
  }
  log(user);
}
```

### Globals and `globalThis`

```js
globalThis;              // the global object, portable across all hosts
// window   — browsers only
// global   — Node only
// self     — workers
```

Use `globalThis` ([ES2020](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/globalThis)) whenever you genuinely need the global object; the older `typeof window !== "undefined" ? window : global` dance is obsolete.

An important asymmetry: in a classic script, top-level `var` and `function` declarations become properties of the global object; `let`, `const` and `class` do not.

```js
// In a classic <script> or node script.js:
var   a = 1;   globalThis.a;   // 1
let   b = 2;   globalThis.b;   // undefined
const c = 3;   globalThis.c;   // undefined
```

In an **ES module**, none of them do — module top-level scope is its own scope, not the global one. This is one of several reasons module code is easier to reason about; see [Part 10](./javascript-errors-async-modules.md#part-10).

### Mastery — closures over the loop binding, and why `for…of` differs

<a id="part-2-mastery"></a>

The per-iteration binding rule applies to `for (let …)` and to `for…of`/`for…in` with `let`/`const`, but **not** to `while` loops — there is no loop head to create a binding in.

```js
// for-of gives a fresh binding per iteration, so const works.
for (const item of [1, 2, 3]) {
  setTimeout(() => console.log(item), 0);   // 1, 2, 3
}

// while has no per-iteration binding — you must create one yourself.
let i = 0;
while (i < 3) {
  const captured = i;                        // fresh binding, created by hand
  setTimeout(() => console.log(captured), 0);
  i++;
}
// 0, 1, 2
```

A subtlety worth knowing: in `for (let i = …)`, the *update expression* (`i++`) runs against the previous iteration's binding, and its result is copied into the next iteration's fresh binding. That is why you can mutate `i` inside the body and still have the loop advance sensibly:

```js
for (let i = 0; i < 3; i++) {
  if (i === 1) i = 100;     // affects the loop
  setTimeout(() => console.log(i), 0);
}
// 0, 100 — the loop then exits because 100 >= 3
```

Compare with `const` in a C-style head, which is a syntax error precisely because the update expression would have to reassign it:

```js
for (const i = 0; i < 3; i++) { }   // ❌ TypeError: Assignment to constant variable
for (const x of [1, 2, 3]) { }      // ✅ fine — no update expression
```

### Part 2 quick reference

<a id="part-2-cheatsheet"></a>

| | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function | Block | Block |
| Hoisted | Yes, as `undefined` | Yes, into the TDZ | Yes, into the TDZ |
| Reassignable | Yes | Yes | No |
| Redeclarable in same scope | Yes | No | No |
| Creates a global-object property (classic script) | Yes | No | No |
| Fresh binding per loop iteration | No | Yes | Yes (`for…of`) |
| Use in new code | Never | When reassigning | Default |

| Task | Do this |
|---|---|
| Prevent reassignment | `const` |
| Prevent mutation (shallow) | `Object.freeze(obj)` |
| Reach the global object | `globalThis` |
| Capture a loop value in a callback | `for (let …)` or `for…of` |
| Capture a value in a `while` loop | `const captured = …` inside the body |

[↑ Back to top](#table-of-contents)

---

## 3. Part 3 — Operators, Coercion & Control Flow

<a id="part-3"></a>

### Beginner — truthiness and the operators you use hourly

<a id="part-3-beginner"></a>

Every value is either **truthy** or **falsy** when used in a boolean position. There are exactly eight falsy values — memorise them, because everything else in the language is truthy:

```js
false
0
-0
0n
""        // empty string
null
undefined
NaN
```

Notably truthy and frequently surprising: `"0"`, `"false"`, `[]`, `{}`, `function(){}`.

```js
if ([])  console.log("empty array is truthy");   // prints
if ({})  console.log("empty object is truthy");  // prints
if ("0") console.log("the string zero is truthy"); // prints
```

```js
// ❌ "Is this array empty?" — always false, the array is always truthy.
if (!items) { showEmptyState(); }

// ✅ Check the length.
if (items.length === 0) { showEmptyState(); }
```

### Comparison

```js
5 === 5;         // true  — strict: same type, same value
5 === "5";       // false — different types
5 == "5";        // true  — loose: coerces first
```

**Use `===` and `!==` everywhere**, with the single documented exception of `x == null` to test for "null or undefined" ([MDN: Equality comparisons](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Equality_comparisons_and_sameness)).

### Logical operators return values, not booleans

```js
"a" || "b";      // "a"  — || returns the first truthy operand
""  || "b";      // "b"
"a" && "b";      // "b"  — && returns the last operand if all are truthy
""  && "b";      // ""   — short-circuits on the first falsy
```

Both **short-circuit**: the right side is not evaluated if the left decides the outcome. That is the basis of the guard idiom:

```js
user && user.save();        // only calls save if user is truthy
const name = input || "anonymous";
```

### Working Knowledge — `??`, `?.`, and their assignment forms

<a id="part-3-working"></a>

`||` falls back on *any* falsy value, which is wrong whenever `0` or `""` are legitimate inputs. `??` (nullish coalescing, ES2020) falls back only on `null` and `undefined`.

```js
// ❌ || eats legitimate zeros and empty strings.
function setVolume(v) {
  const volume = v || 50;      // setVolume(0) → 50. The mute is ignored.
  return volume;
}
setVolume(0);                  // 50  ← bug

// ✅ ?? only falls back on null/undefined.
function setVolume(v) {
  const volume = v ?? 50;
  return volume;
}
setVolume(0);                  // 0   ← correct
setVolume(undefined);          // 50
```

This is the highest-value single substitution in modern JavaScript. Whenever you see `||` used for a default, ask whether `0`, `""`, or `false` is a valid value — if so, it is a latent bug.

**Optional chaining** `?.` short-circuits to `undefined` instead of throwing:

```js
const city = user?.address?.city;         // undefined if user or address is nullish
const first = list?.[0];                  // optional index access
const result = obj.method?.();            // calls only if method exists
```

```js
// ❌ The pre-2020 idiom, still everywhere in older code.
const city = user && user.address && user.address.city;
// Subtly different: returns "" if user is "", rather than undefined.

// ✅ Optional chaining.
const city = user?.address?.city;
```

One rule that catches people: `?.` guards only the access immediately after it.

```js
// ❌ user?.address is undefined, then .city throws on undefined.
const city = user?.address.city;   // TypeError if address is missing

// ✅ Guard every level that can be nullish.
const city = user?.address?.city;
```

**Logical assignment operators** (ES2021) combine the two ideas:

```js
a ||= b;    // a = a || b     — assign if a is falsy
a &&= b;    // a = a && b     — assign if a is truthy
a ??= b;    // a = a ?? b     — assign if a is null/undefined

const options = { retries: 0 };
options.retries ??= 3;      // stays 0 — correct
options.timeout ??= 5000;   // set to 5000
```

These short-circuit the *assignment*, not just the evaluation — `a ??= b` does not write to `a` at all when `a` is non-nullish, which matters if `a` is a setter or a proxied property.

> **Try It.** Run:
> ```js
> const cfg = { debug: false, retries: 0, name: "" };
> console.log(cfg.debug || true, cfg.debug ?? true);
> console.log(cfg.retries || 3, cfg.retries ?? 3);
> ```
> Expected: `true false` then `3 0`. Every `||` on that line is a bug; every `??` is correct.

### Destructuring and spread/rest

```js
// Object destructuring, with renaming and defaults.
const { name, age = 0, address: { city } = {} } = user;

// Array destructuring, with skipping and rest.
const [first, , third, ...others] = [1, 2, 3, 4, 5];
// first = 1, third = 3, others = [4, 5]

// Swap without a temporary.
let a = 1, b = 2;
[a, b] = [b, a];

// Spread — copies (shallow).
const copy   = { ...original };
const merged = { ...defaults, ...overrides };   // later wins
const joined = [...arr1, ...arr2];

// Rest in parameters.
const sum = (...nums) => nums.reduce((t, n) => t + n, 0);
```

Two rules that trip people up: **defaults apply only to `undefined`, never to `null`**, and object spread is shallow.

```js
const { port = 8080 } = { port: null };
console.log(port);    // null — the default did NOT apply

const { port2 = 8080 } = { port2: undefined };
console.log(port2);   // 8080
```

```js
// ❌ Spread is shallow — nested objects are shared.
const copy = { ...original };
copy.address.city = "Oslo";       // also changes original.address.city

// ✅ Deep copy.
const copy = structuredClone(original);
```

`structuredClone` is [Baseline widely available](https://developer.mozilla.org/en-US/docs/Web/API/Window/structuredClone) and handles cycles, `Map`, `Set`, `Date` and typed arrays — everything the old `JSON.parse(JSON.stringify(x))` trick silently destroys. It cannot clone functions, DOM nodes, or class identity (you get a plain object back). More in [Part 5](./javascript-objects-classes.md).

### Advanced — how coercion actually works

<a id="part-3-advanced"></a>

This is the section that turns `[] == false` from a party trick into something you can reason about.

**`==` between different types** applies a fixed sequence of conversions ([spec: IsLooselyEqual](https://tc39.es/ecma262/2026/)):

1. `null == undefined` → `true`. Nothing else is loosely equal to either.
2. Number vs. string → convert the string to a number.
3. Boolean on either side → convert the boolean to a number (`true`→`1`, `false`→`0`).
4. Object vs. primitive → convert the object to a primitive (`ToPrimitive`), then start over.

Walk `[] == false` through it:

```text
[] == false
[] == 0            (rule 3: false → 0)
"" == 0            (rule 4: [] → "" via ToPrimitive)
0  == 0            (rule 2: "" → 0)
true
```

And the famous non-transitive triple:

```js
"0" == false;      // true   — both become 0
"0" == 0;          // true
false == 0;        // true
"0" == "";         // false  — both strings, no conversion, "0" ≠ ""
null == 0;         // false  — rule 1: null is only loosely equal to undefined
null >= 0;         // true   — relational operators use a different algorithm!
```

That last pair is the sharpest edge: `null == 0` is `false` but `null >= 0` is `true`, because `>=` converts to number (`null` → `0`) while `==` has the special null/undefined rule. There is no consistent story; just use `===` and explicit conversion.

**`ToPrimitive` for objects** calls, in order: `Symbol.toPrimitive` if present, then `valueOf` and `toString` (order depends on the requested hint). That is why:

```js
[] + [];           // ""        — both become "", concatenated
[] + {};           // "[object Object]"
1 + "2";           // "12"      — + prefers string if either side is a string
1 - "2";           // -1        — - has no string meaning, so both become numbers
"5" * "2";         // 10
```

The `+` operator is the only arithmetic operator that is also string concatenation, which makes it the only one that behaves this way.

```js
// ❌ Reading a numeric input and adding.
const total = qty + input.value;    // "1" + "2" = "12"

// ✅ Convert explicitly.
const total = qty + Number(input.value);
```

### Explicit conversion

```js
Number("42");        // 42
Number("");          // 0        ← surprising
Number(" 42 ");      // 42       ← whitespace trimmed
Number("42px");      // NaN
Number(null);        // 0        ← surprising
Number(undefined);   // NaN
Number([]);          // 0
Number([5]);         // 5
Number([1,2]);       // NaN

parseInt("42px");    // 42       ← stops at the first non-digit
parseFloat("3.14x"); // 3.14
parseInt("08");      // 8        (modern; ancient engines guessed octal)

String(42);          // "42"
String(null);        // "null"
String([1,2]);       // "1,2"
String({});          // "[object Object]"

Boolean("");         // false
!!"";                // false — the terse idiom
```

Prefer `Number()` when the whole string must be a number, and `parseInt`/`parseFloat` when you deliberately want a prefix parsed. **Always pass the radix to `parseInt` when the input is untrusted** — `parseInt(str, 10)`.

> **Try It.** Predict, then run:
> ```js
> console.log([] + [], [] + {}, 1 + "2", 1 - "2", "5" * "2", [] == false, "0" == false, "0" == "");
> ```
> Expected: `'' '[object Object]' '12' -1 10 true true false`. If the last three surprise you, re-read the coercion ladder above — the answer is mechanical, not magical.

### Control flow beyond `if`

```js
// switch uses === (no coercion), and falls through without break.
switch (kind) {
  case "a":
  case "b":            // deliberate fall-through: a and b share a branch
    handleAB();
    break;
  case "c":
    handleC();
    break;
  default:
    handleOther();
}
```

```js
// ❌ switch(true) abuse — works, but obscures intent.
switch (true) {
  case score > 90: grade = "A"; break;
  case score > 80: grade = "B"; break;
}

// ✅ Plain if/else if is clearer for ranges.
const grade = score > 90 ? "A" : score > 80 ? "B" : "C";
```

Labelled `break`/`continue` are the one legitimate use of labels — breaking an outer loop from an inner one:

```js
outer:
for (const row of rows) {
  for (const cell of row) {
    if (cell === target) break outer;   // exits both loops
  }
}
```

The alternative — a `found` flag checked in both loop conditions — is strictly worse. Labels have a bad reputation borrowed from `goto`; this specific use is fine.

### Mastery — operator precedence traps and `Symbol.toPrimitive`

<a id="part-3-mastery"></a>

Two precedence facts cause real bugs:

**1. `??` cannot be mixed with `||` or `&&` without parentheses.** This is a syntax error by design, because the intended precedence is genuinely ambiguous:

```js
const x = a || b ?? c;      // ❌ SyntaxError
const x = (a || b) ?? c;    // ✅ explicit
```

**2. `await` binds tighter than most things, but not `**`:**

```js
// ❌ Reads as (await getA()) + getB() — no, actually it reads correctly,
//    but this one genuinely surprises people:
const x = await a ** 2;     // ❌ SyntaxError — ambiguous
const x = (await a) ** 2;   // ✅
```

**3. `typeof` binds tighter than `===`… but not than `+`:**

```js
typeof 1 + 2;               // "number2"  ← (typeof 1) + 2
typeof (1 + 2);             // "number"
```

### Controlling your own object's coercion

`Symbol.toPrimitive` lets a class decide what it becomes in a numeric, string, or default context — the mechanism behind `Date`'s split personality.

```js
class Money {
  constructor(cents) { this.cents = cents; }

  [Symbol.toPrimitive](hint) {
    if (hint === "number") return this.cents;
    if (hint === "string") return `$${(this.cents / 100).toFixed(2)}`;
    return `Money(${this.cents})`;      // hint === "default", e.g. ==, +
  }
}

const m = new Money(1250);
+m;                  // 1250          — number hint
`${m}`;              // "$12.50"      — string hint
m + "";              // "Money(1250)" — default hint
```

`Date` does the same thing in reverse, which explains a long-standing oddity:

```js
const d = new Date();
d + 1;               // string concatenation — Date's default hint is "string"
d - 1;               // a number — the "-" operator forces the number hint
```

This is the only built-in where the default hint is `"string"` rather than `"number"`, and it exists for backwards compatibility with pre-ES6 code ([MDN: Date.prototype[Symbol.toPrimitive]](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/Symbol.toPrimitive)).

> **Try It.** Paste the `Money` class into a REPL, then run `+m`, `` `${m}` ``, and `m + ""`. Expected: `1250`, `'$12.50'`, `'Money(1250)'`. Now delete the `[Symbol.toPrimitive]` method and run the three again. Expected: `NaN`, `'[object Object]'`, `'[object Object]'` — that is the default `ToPrimitive` you were overriding.

### Part 3 quick reference

<a id="part-3-cheatsheet"></a>

| Need | Operator | Note |
|---|---|---|
| Equality | `===` | Always, except `== null` |
| Null/undefined check | `x == null` | The one sanctioned `==` |
| Default on falsy | `\|\|` | Only when `0`/`""` are invalid |
| Default on nullish | `??` | The safer default |
| Safe property access | `?.` | Guards one level each |
| Assign if nullish | `??=` | Skips the write entirely if set |
| Exit nested loops | `break label` | Better than a flag |

**The eight falsy values:** `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN`. Everything else — including `[]`, `{}`, `"0"`, `"false"` — is truthy.

| Conversion | `Number(v)` | `String(v)` | `Boolean(v)` |
|---|---|---|---|
| `""` | `0` | `""` | `false` |
| `"0"` | `0` | `"0"` | **`true`** |
| `null` | `0` | `"null"` | `false` |
| `undefined` | `NaN` | `"undefined"` | `false` |
| `[]` | `0` | `""` | **`true`** |
| `[5]` | `5` | `"5"` | `true` |
| `{}` | `NaN` | `"[object Object]"` | **`true`** |

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 4 — Functions & Closures](./javascript-functions-closures.md) · **Up:** [Mastery Guide overview](./javascript-mastery-guide.md)

*Written against ECMAScript 2026. Baseline data verified September 6, 2026.*
