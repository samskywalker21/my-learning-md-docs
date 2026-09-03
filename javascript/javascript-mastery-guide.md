# Mastering JavaScript — A Hands-On Reference Guide

## About This Document

- **Framing:** Curiosity-driven, modern-JS-forward. No upcoming task is driving this — the goal is durable, hands-on understanding of JavaScript itself. It is *not* framed around preparing for TypeScript, and it is *not* a historical/evolution narrative — theory ("why it's designed this way") only shows up where it's needed to explain a mechanism (e.g. the event loop, the prototype chain), not as standalone history.
- **Style:** A guided tutorial — a sequence of runnable examples and exercises you're meant to actually type into a REPL or file, in order. Each section builds on the last.
- **Depth tiers:** Every major section is structured as four progressive, explicitly labeled tiers — **Beginner → Working Knowledge → Advanced → Mastery** — so you can stop at whichever level you need right now, or come back later for the next tier. Tiers are collapsed where they'd be trivial or redundant for that topic (this happens most in [Part 10 — Modules](#10-part-10--modules), least in [Functions & Closures](#5-part-5--functions--closures), [Objects & Prototypes](#6-part-6--objects--prototypes), and [Asynchronous JavaScript](#9-part-9--asynchronous-javascript), which are where JS's real "aha" moments live).
- **Scope:** Core JavaScript language only — no DOM/browser APIs (beyond using the console as a scratchpad), no Node.js-specific runtime APIs beyond what's needed to run the examples, no frameworks.
- **Modern-JS-forward:** Every example is written in current JavaScript (ES2015+) by default — `let`/`const`, arrow functions, destructuring, spread/rest, template literals, optional chaining, `async`/`await`, ES modules, classes. Older forms (`var`, callback-style async, CommonJS `require`) are mentioned only where you'll actually run into them reading older code, or where they explain a gotcha.
- **Assumed background:** Zero prior programming/JavaScript experience. Beginner tier is not a formality here — it carries real weight throughout.
- **Sourcing standard:** [MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript) is the primary source for language behavior, cited inline. Stack Overflow is used for real-world gotchas and common mistakes. Checked as of September 2026; anywhere a "classic" pattern has been superseded by newer syntax, that's flagged explicitly.
- **Update instructions for future-me:** If asked to update this later — match this exact structure (numbered Parts, each with tier subheadings), keep the wrong-vs-right snippet pairs for gotchas, keep the TOC + back-to-top links, and prefer adding new Parts/tiers over reverting to freeform prose. Collapsing tiers that feel forced for a given topic is expected, not a deviation to "fix."

---

## Table of Contents

1. [Orientation & Setup](#1-orientation--setup)
2. [Part 1 — Values & Types](#2-part-1--values--types)
3. [Part 2 — Variables & Scope](#3-part-2--variables--scope)
4. [Part 3 — Operators, Coercion & Control Flow](#4-part-3--operators-coercion--control-flow)
5. [Part 4 — Functions & Closures](#5-part-4--functions--closures)
6. [Part 5 — Objects & Prototypes](#6-part-5--objects--prototypes)
7. [Part 6 — Arrays & Iteration](#7-part-6--arrays--iteration)
8. [Part 7 — Classes](#8-part-7--classes)
9. [Part 8 — Asynchronous JavaScript](#9-part-8--asynchronous-javascript)
10. [Part 9 — Modules](#10-part-9--modules)
11. [Part 10 — Modern JS Grab Bag](#11-part-10--modern-js-grab-bag)
12. [Cheat Sheets](#12-cheat-sheets)
13. [Suggested Learning Order](#13-suggested-learning-order)
14. [Quick Self-Check](#14-quick-self-check)

---

## 1. Orientation & Setup

**What it is.** JavaScript is a dynamically-typed, single-threaded (per-context) scripting language. It runs inside a *host environment* — a browser or [Node.js](https://nodejs.org) — which supplies the actual language engine (V8 in Chrome/Node, SpiderMonkey in Firefox, JavaScriptCore in Safari) plus extra APIs (`document` in browsers, `fs` in Node) that aren't part of the language itself. This guide only teaches the language — the part that's identical everywhere.

**Why it matters.** Knowing where the line between "the language" and "the host" sits stops a lot of confusion later — `fetch`, `document.querySelector`, and `require` are *not* JavaScript; `let`, `class`, and `Promise` are.

**Running code — pick one:**

```bash
# Option A: Node REPL — instant feedback, best for short experiments
node
> 1 + 1
2
> .exit

# Option B: run a file
node hello.js
```

```js
// hello.js
console.log("hello, world");
```

```text
# Option C: browser DevTools console (Chrome/Firefox/Edge: F12 → Console tab)
# Good for quick one-liners; no file needed.
```

Throughout this guide, every runnable snippet is written assuming Option A or B — paste it into the REPL, or save it to a `.js` file and run `node filename.js`.

**Real Scenario — try it now.** Open a terminal, run `node`, and type:

```js
console.log(typeof console.log);
```

If you see `function`, your setup works and you're ready to move on.

[⬆ back to top](#table-of-contents)

---

## 2. Part 1 — Values & Types

Covers: primitive types, `typeof`, template literals.

### Beginner

JavaScript has 7 primitive types and one non-primitive (`object`, which includes arrays and functions):

```js
"hello"        // string
42             // number
42n            // bigint
true           // boolean
undefined      // undefined
null           // null (yes, its own type conceptually)
Symbol("id")   // symbol
{ a: 1 }       // object
```

`typeof` tells you what you're dealing with:

```js
typeof "hi"        // "string"
typeof 42          // "number"
typeof true         // "boolean"
typeof undefined    // "undefined"
typeof {}           // "object"
typeof [1, 2, 3]    // "object"  — arrays are objects
typeof function(){} // "function"
```

**Wrong vs. right — building strings:**

```js
// Wrong — concatenation gets unreadable fast, and easy to mistype a space
const name = "Sam";
const age = 34;
console.log("Hi " + name + ", you are " + age + " years old.");

// Right — template literals: backticks, ${} for interpolation
console.log(`Hi ${name}, you are ${age} years old.`);
```

Template literals also span multiple lines without `\n`:

```js
const msg = `line one
line two`;
```
([Template literals — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals))

### Working Knowledge

**The `typeof null` gotcha.**

```js
typeof null   // "object"  — this is a 25+ year old bug baked into the spec, kept for compatibility
```

If you need to check for `null` specifically, compare directly: `value === null`.
([typeof — MDN, see "typeof null"](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/typeof))

**`undefined` vs `null` — both mean "no value," different intents:**

| | Meaning |
| --- | --- |
| `undefined` | A variable exists but hasn't been assigned; a function returned nothing; a missing object property |
| `null` | A deliberate "no value" assigned by your code |

```js
let x;
console.log(x); // undefined — you never assigned it

let y = null;
console.log(y); // null — you explicitly said "nothing here"
```

**Real Scenario — try it.**

```js
function greet(name) {
  return `Hello, ${name}`;
}
console.log(greet()); // "Hello, undefined" — not an error!
```

JS doesn't error on a missing argument; the parameter is just `undefined` inside the function. This is the root cause of a huge class of "why does my page say `undefined`" bugs — nothing crashes, it just silently prints the wrong thing.

### Advanced

**`bigint` for numbers beyond safe integer range.** Regular `number` uses IEEE-754 double precision — integers above `Number.MAX_SAFE_INTEGER` (2^53 − 1) lose precision.

```js
console.log(9007199254740993);        // 9007199254740992 — wrong! precision lost
console.log(9007199254740993n);       // 9007199254740993n — correct, bigint
```

`bigint` and `number` cannot be mixed with arithmetic operators — `1n + 1` throws `TypeError`.
([BigInt — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt))

### Mastery

**`Symbol` — guaranteed-unique values, used as non-colliding object keys.**

```js
const id1 = Symbol("id");
const id2 = Symbol("id");
console.log(id1 === id2); // false — every Symbol() call produces a unique value, even with the same description
```

Symbols are mostly used internally by the language (e.g. `Symbol.iterator`, covered in [Part 6](#7-part-6--arrays--iteration)) to add behavior to objects without risking a name collision with a real property.

[⬆ back to top](#table-of-contents)

---

## 3. Part 2 — Variables & Scope

Covers: `let`/`const`/`var`, block scope, hoisting, the temporal dead zone.

### Beginner

Three ways to declare a variable — use `const` by default, `let` when you need to reassign, and avoid `var`:

```js
const pi = 3.14159;   // cannot be reassigned
let count = 0;
count = count + 1;    // fine, let allows reassignment

var oldStyle = "avoid this"; // legacy — explained below
```

`const` prevents *reassignment*, not mutation — an object or array declared `const` can still have its contents changed:

```js
const arr = [1, 2, 3];
arr.push(4);      // fine — mutating the array, not reassigning arr
console.log(arr); // [1, 2, 3, 4]

arr = [5, 6];      // TypeError: Assignment to constant variable.
```

### Working Knowledge

**Block scope.** `let` and `const` are scoped to the nearest `{}` block — `if`, `for`, or a bare block. This is the behavior you want and expect.

```js
if (true) {
  let secret = "hidden";
}
console.log(secret); // ReferenceError: secret is not defined — correct, expected behavior
```

**Wrong vs. right — the classic `var` loop bug:**

```js
// Wrong — var is function-scoped, not block-scoped, so all three callbacks share ONE i
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// logs: 3, 3, 3   (by the time the callbacks run, the loop has finished and i is 3)

// Right — let creates a fresh binding per iteration
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// logs: 0, 1, 2
```

This single bug is the main historical reason `let`/`const` replaced `var` as the default — it's worth understanding even though you'll rarely write `var` yourself.
([`let` — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let), related [Stack Overflow: JavaScript closure inside loops](https://stackoverflow.com/questions/750486/how-do-i-create-a-closure-with-a-for-loop))

### Advanced

**Hoisting & the Temporal Dead Zone (TDZ).** All declarations are "hoisted" (registered) at the top of their scope before code runs — but `let`/`const` bindings stay uninitialized until their actual declaration line executes. Accessing them before that point throws, instead of silently returning `undefined`.

```js
console.log(a); // ReferenceError: Cannot access 'a' before initialization
let a = 1;

console.log(b); // undefined — var hoists AND initializes to undefined
var b = 1;
```

```text
Scope entry
  │
  ▼
┌─────────────────────────────────┐
│ let/const: binding exists,       │  ← "temporal dead zone" — accessing
│ but reading/writing it throws    │     throws ReferenceError here
├─────────────────────────────────┤
│ declaration line runs             │  ← binding becomes usable
│ (let a = 1;)                      │
└─────────────────────────────────┘
```

The TDZ is a deliberate safety feature — it turns "used a variable before it was ready" from a silent `undefined` bug into a loud, immediate error.
([let — MDN, Temporal Dead Zone section](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let#temporal_dead_zone_tdz))

### Mastery

Function declarations are hoisted *with their body*, so they're callable before the line they're written on — this is different from `let`/`const`/`var` hoisting and worth knowing when reading code that calls a function "before" its definition:

```js
sayHi(); // works — "Hi!"
function sayHi() { console.log("Hi!"); }
```

Function *expressions* (including arrow functions) assigned to `let`/`const` do **not** get this treatment — they follow the normal TDZ rules above, since only the variable binding is hoisted, not the function body.

[⬆ back to top](#table-of-contents)

---

## 4. Part 3 — Operators, Coercion & Control Flow

Covers: `==` vs `===`, truthy/falsy, optional chaining, nullish coalescing, control flow.

### Beginner

```js
if (age >= 18) {
  console.log("adult");
} else if (age >= 13) {
  console.log("teen");
} else {
  console.log("child");
}

const label = age >= 18 ? "adult" : "minor"; // ternary — shorthand if/else for a value
```

### Working Knowledge

**Wrong vs. right — equality:**

```js
// Wrong — == coerces types before comparing, producing surprising results
0 == "";        // true
0 == "0";       // true
"" == "0";      // false  ← inconsistent! this is why == is avoided
null == undefined; // true

// Right — === compares value AND type, no coercion, no surprises
0 === "";       // false
0 === "0";      // false
"" === "0";     // false
```

Default to `===`/`!==` always; reach for `==` only in the one idiomatic case of checking for both `null` and `undefined` at once (`value == null`), which some style guides still allow deliberately.
([Equality comparisons and sameness — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Equality_comparisons_and_sameness), [Stack Overflow: Which equals operator should be used?](https://stackoverflow.com/questions/359494/which-equals-operator-should-be-used-in-javascript-comparisons))

**Truthy / falsy.** Every value is "truthy" or "falsy" when used where a boolean is expected (`if`, `&&`, `||`, `!`). There are exactly **8 falsy values** — everything else is truthy:

```js
false, 0, -0, 0n, "", null, undefined, NaN
```

```js
if ("0") console.log("runs");   // "0" is a non-empty string → truthy! (common gotcha)
if (0) console.log("skipped");  // number 0 → falsy
if ([]) console.log("runs");    // empty array → truthy (objects are always truthy)
```

**Optional chaining (`?.`) and nullish coalescing (`??`).**

```js
const user = { profile: { name: "Sam" } };

// Wrong — manual guarding against missing nested properties is verbose and error-prone
const city = user.profile && user.profile.address && user.profile.address.city;

// Right — ?. short-circuits to undefined the moment it hits null/undefined
const city2 = user.profile?.address?.city; // undefined, no error

user.sayHi?.(); // calls sayHi only if it exists; no-op (not a TypeError) if it doesn't
```

```js
// ?? only falls back on null/undefined — NOT on other falsy values like || does
const count = 0;
console.log(count || 10); // 10 — WRONG if 0 is a valid value you wanted to keep!
console.log(count ?? 10); // 0  — RIGHT, 0 is not null/undefined so it's kept
```

Both are [Baseline widely available](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining) — safe to use everywhere as of this writing. `?.` cannot appear on the left side of an assignment (`obj?.prop = 1` is a `SyntaxError`), and short-circuiting only propagates along an unbroken chain — wrapping part of it in parentheses breaks that guarantee.
([Optional chaining — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining), [Nullish coalescing — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing))

### Advanced

**`NaN` is the only value that isn't equal to itself.**

```js
NaN === NaN;          // false
Number.isNaN(NaN);    // true — the correct way to check
isNaN("hello");        // true — global isNaN coerces first, "hello" → NaN — often NOT what you want
Number.isNaN("hello"); // false — no coercion, "hello" is not literally NaN
```

Always prefer `Number.isNaN` over the global `isNaN` for this reason.
([Number.isNaN — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/isNaN))

### Mastery

**Short-circuit evaluation as control flow.** `&&` and `||` return one of their *operands*, not necessarily a boolean — this is exploited idiomatically:

```js
user && user.name;          // returns user.name if user is truthy, else returns user itself (falsy)
isLoggedIn && renderDashboard(); // only calls renderDashboard() if isLoggedIn is truthy
```

This works because `&&`/`||` short-circuit: `a && b` evaluates `b` only if `a` is truthy; `a || b` evaluates `b` only if `a` is falsy — the "unused" side is never touched, which matters if it has side effects.

[⬆ back to top](#table-of-contents)

---

## 5. Part 4 — Functions & Closures

Covers: declarations vs. expressions vs. arrow functions, default/rest params, closures, higher-order functions.

### Beginner

Three ways to write a function:

```js
// Function declaration — hoisted, callable before its definition
function add(a, b) {
  return a + b;
}

// Function expression — a function value assigned to a variable
const subtract = function (a, b) {
  return a - b;
};

// Arrow function — shorter syntax, and (crucially) no own `this` — see Part 5
const multiply = (a, b) => a * b;
```

**Default and rest parameters:**

```js
function greet(name = "friend") {
  return `Hello, ${name}`;
}
greet();        // "Hello, friend"
greet("Sam");   // "Hello, Sam"

function sum(...numbers) { // rest: gathers any number of args into a real array
  return numbers.reduce((total, n) => total + n, 0);
}
sum(1, 2, 3, 4); // 10
```

### Working Knowledge

**Closures — a function "remembers" the variables from where it was defined, even after that outer function has returned.**

```js
function makeCounter() {
  let count = 0;
  return function () {
    count += 1;
    return count;
  };
}

const counter = makeCounter();
console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3
```

`count` lives on inside the returned function's closure — it isn't reset each call, and nothing outside can reach it directly. This is the mechanism behind private state in JS before classes had private fields.

```text
makeCounter() called
  │
  ▼
┌───────────────────────────┐
│ scope: { count: 0 }        │◄─── returned function keeps a
└───────────────────────────┘      reference to this scope
  │  returns inner function        (this is the "closure")
  ▼
counter ──► inner function ──► [scope: count]
```

**Real Scenario — the loop-closure bug, functions edition.**

```js
// Wrong — every button's click handler closes over the SAME shared `i`
const buttons = [];
for (var i = 0; i < 3; i++) {
  buttons.push(() => console.log(`Button ${i} clicked`));
}
buttons[0](); // "Button 3 clicked" — not 0!
buttons[1](); // "Button 3 clicked"

// Right — let gives each iteration its own binding (see Part 2)
const buttons2 = [];
for (let i = 0; i < 3; i++) {
  buttons2.push(() => console.log(`Button ${i} clicked`));
}
buttons2[0](); // "Button 0 clicked" — correct
```
([Closures — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Closures), [Stack Overflow: closure inside loop simple practical example](https://stackoverflow.com/questions/12931369/javascript-closures-inside-loops-simple-practical-example))

**Higher-order functions — functions that take or return other functions:**

```js
function applyTwice(fn, value) {
  return fn(fn(value));
}
applyTwice((x) => x * 2, 3); // 12
```

### Advanced

**Arrow functions don't bind their own `this` — they capture it from the surrounding scope.** This matters most inside object methods and event callbacks (covered more in [Part 5](#6-part-5--objects--prototypes)):

```js
// Wrong — regular function as a method's callback loses `this`
const counter = {
  count: 0,
  start() {
    setInterval(function () {
      this.count++; // `this` here is NOT counter — it's undefined (strict mode) or the global object
      console.log(this.count); // NaN / error
    }, 1000);
  },
};

// Right — arrow function inherits `this` from start()'s scope, i.e. counter
const counter2 = {
  count: 0,
  start() {
    setInterval(() => {
      this.count++; // `this` is counter2, as expected
      console.log(this.count);
    }, 1000);
  },
};
```
([Arrow function expressions — MDN, "No binding of this"](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions#no_binding_of_this))

**Wrong vs. right — memoization gotcha with closures over mutable state:**

```js
// Wrong — cache is shared across ALL calls with any argument, fine here, but note
// it's a MODULE-level closure, not per-call — easy to accidentally leak state this way
function makeAdder(base) {
  let calls = 0; // intended to be per-adder, and it correctly is — each makeAdder() call
  return (n) => { // creates a fresh `calls`/`base` closure. The gotcha is assuming
    calls++;        // this resets per CALL to the returned function — it doesn't.
    return base + n;
  };
}
```

The lesson: a closure's captured variables persist for the *lifetime of the closure*, not per invocation — decide deliberately whether state should live inside or outside the returned function.

### Mastery

**Currying — using closures to partially apply arguments:**

```js
const add = (a) => (b) => (c) => a + b + c;
add(1)(2)(3); // 6

const add5 = add(5); // a is now permanently 5, captured in the closure
add5(10)(1); // 16
```

Each call returns a new function whose closure remembers the arguments applied so far — this is the same mechanism as the counter example above, just chained.

[⬆ back to top](#table-of-contents)

---

## 6. Part 5 — Objects & Prototypes

Covers: object literals, destructuring, spread/rest, `this`, prototypal inheritance.

### Beginner

```js
const person = {
  name: "Sam",
  age: 34,
  greet() {
    return `Hi, I'm ${this.name}`;
  },
};

console.log(person.name);     // "Sam"
console.log(person["name"]);  // "Sam" — bracket notation, needed for dynamic/computed keys
console.log(person.greet());  // "Hi, I'm Sam"
```

**Destructuring — pull values out of an object into variables:**

```js
const { name, age } = person;
console.log(name, age); // "Sam" 34

// Wrong — verbose manual extraction
const name2 = person.name;
const age2 = person.age;

// Right — destructure in one line, rename and default in the same breath
const { name: fullName, city = "Unknown" } = person;
console.log(fullName, city); // "Sam" "Unknown"
```

### Working Knowledge

**Spread and rest with objects:**

```js
const base = { theme: "dark", fontSize: 14 };
const overrides = { fontSize: 16 };

// Wrong — mutating the original object to "merge" settings
Object.assign(base, overrides); // mutates base! surprising if base is shared elsewhere

// Right — spread creates a NEW object, leaving both originals untouched
const settings = { ...base, ...overrides }; // { theme: "dark", fontSize: 16 }
// later keys win on conflict — order matters
```

```js
const { fontSize, ...rest } = settings; // rest: { theme: "dark" }
```

**`this` — determined by *how* a function is called, not where it's defined (for regular functions):**

```js
const obj = {
  name: "obj",
  regular: function () { return this.name; },
};

const fn = obj.regular;
console.log(obj.regular()); // "obj" — called AS a method of obj, so this = obj
console.log(fn());          // undefined (or throws in strict mode) — called standalone, this is lost
```

This is one of the most common real-world JS bugs — passing a method as a callback (`onClick={obj.regular}`) silently detaches `this`. Fix with an arrow function wrapper (`() => obj.regular()`) or `.bind(obj)`.
([this — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this), [Stack Overflow: “this” is undefined in JavaScript class methods](https://stackoverflow.com/questions/40214126/why-is-this-undefined-inside-class-method-when-using-promises))

### Advanced

**Prototypal inheritance — every object has an internal link to another object it delegates to for missing properties.**

```js
const animal = {
  eats: true,
  describe() {
    return "I am an animal";
  },
};

const rabbit = Object.create(animal); // rabbit's prototype is animal
rabbit.jumps = true;

console.log(rabbit.eats);        // true — not on rabbit itself, found via the prototype chain
console.log(rabbit.describe());  // "I am an animal" — inherited method
console.log(Object.getPrototypeOf(rabbit) === animal); // true
```

```text
rabbit  { jumps: true }
   │  [[Prototype]]
   ▼
animal  { eats: true, describe() {...} }
   │  [[Prototype]]
   ▼
Object.prototype  { toString(), hasOwnProperty(), ... }
   │  [[Prototype]]
   ▼
null                          ← chain ends here
```

Property lookup walks up this chain until it finds a match or hits `null`. This is *the* mechanism behind `class`/`extends` in [Part 7](#8-part-7--classes) — classes are ergonomic syntax over exactly this.
([Object prototypes — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Object_prototypes))

**Wrong vs. right — checking for a property:**

```js
// Wrong — `in` and reading a value can't tell "property is undefined" from "property doesn't exist"
if (rabbit.tail !== undefined) { /* ... */ }  // false negative if tail was explicitly set to undefined

// Right, when you specifically want "does this key exist" including inherited ones
if ("eats" in rabbit) { /* ... */ } // true — walks the prototype chain too

// Right, when you want "does THIS object own it" (not inherited)
if (rabbit.hasOwnProperty("jumps")) { /* ... */ } // true
if (rabbit.hasOwnProperty("eats")) { /* ... */ }  // false — eats is inherited, not own
```

### Mastery

**`Object.create(null)` — an object with no prototype at all**, useful for a plain lookup map with zero risk of prototype pollution or accidental collisions with inherited names like `toString`:

```js
const map = Object.create(null);
map.toString = "not a function anymore, just a value";
console.log(map.hasOwnProperty); // undefined — no Object.prototype to inherit from
```

**Property descriptors** — every property has hidden metadata controlling writability/enumerability/configurability, which `Object.freeze`/`Object.seal` manipulate:

```js
const frozen = Object.freeze({ x: 1 });
frozen.x = 2;               // silently fails (throws in strict mode)
console.log(frozen.x);       // 1 — unchanged
```
([Object.freeze — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze)). Note `freeze` is shallow — nested objects inside a frozen object are still mutable unless frozen individually.

[⬆ back to top](#table-of-contents)

---

## 7. Part 6 — Arrays & Iteration

Covers: core array methods, destructuring, `for...of`, iterators & generators.

### Beginner

```js
const nums = [1, 2, 3, 4, 5];

nums.push(6);       // add to end — mutates
nums.pop();          // remove from end — mutates
nums.length;          // 6... wait, pop() removed one, so 5
nums[0];               // 1 — index access
```

**Destructuring and spread with arrays:**

```js
const [first, second, ...remaining] = nums;
console.log(first, second, remaining); // 1 2 [3, 4, 5]

const combined = [...nums, 6, 7]; // new array, original untouched
```

### Working Knowledge

**The core transformation trio — `map`, `filter`, `reduce`. None of these mutate the original array.**

```js
const prices = [10, 25, 40, 15];

const withTax = prices.map((p) => p * 1.1);              // transform each element
const expensive = prices.filter((p) => p > 20);           // keep some elements
const total = prices.reduce((sum, p) => sum + p, 0);        // fold into one value
```

**Wrong vs. right — mutation vs. new-array methods:**

```js
// Wrong — sort() and reverse() mutate IN PLACE, surprising if you expected a new array
const original = [3, 1, 2];
const sorted = original.sort();
console.log(original); // [1, 2, 3] — original was mutated too!

// Right — copy first if you need to preserve the original (toSorted is the modern fix)
const sorted2 = [...original].sort();
const sorted3 = original.toSorted(); // ES2023 — returns a new sorted array, no mutation
```
([Array.prototype.toSorted — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/toSorted)) — `toSorted`/`toReversed`/`toSpliced`/`with` are the modern non-mutating counterparts to `sort`/`reverse`/`splice`, widely supported since 2023.

**`for...of` — the modern default for iterating values:**

```js
for (const price of prices) {
  console.log(price);
}

// Wrong — for...in iterates KEYS (as strings) and walks inherited enumerable
// properties too — almost never what you want for arrays
for (const i in prices) {
  console.log(i, typeof i); // "0" string, "1" string, ... — not the values!
}

// Right — for...of iterates VALUES directly
for (const p of prices) {
  console.log(p); // 10, 25, 40, 15 — actual numbers
}
```
([Stack Overflow: for...in vs for...of](https://stackoverflow.com/questions/9329446/for-each-over-an-array-in-javascript))

### Advanced

**`find`, `some`, `every`, `flat`/`flatMap`:**

```js
prices.find((p) => p > 20);      // 25 — first match, or undefined
prices.some((p) => p > 100);      // false — at least one?
prices.every((p) => p > 0);        // true — all match?

[[1, 2], [3, [4, 5]]].flat();       // [1, 2, 3, [4, 5]] — flattens one level
[[1, 2], [3, [4, 5]]].flat(2);       // [1, 2, 3, 4, 5]  — depth argument
[1, 2, 3].flatMap((n) => [n, n * 2]); // [1, 2, 2, 4, 3, 6] — map then flatten one level
```

### Mastery

**Iterators & the `Symbol.iterator` protocol — what actually makes `for...of` work.** Any object implementing `[Symbol.iterator]()` returning an object with a `.next()` method (each call returning `{ value, done }`) is iterable:

```js
const range = {
  from: 1,
  to: 3,
  [Symbol.iterator]() {
    let current = this.from;
    const last = this.to;
    return {
      next() {
        return current <= last
          ? { value: current++, done: false }
          : { value: undefined, done: true };
      },
    };
  },
};

for (const n of range) console.log(n); // 1, 2, 3
console.log([...range]); // [1, 2, 3] — spread also uses the iterator protocol
```

**Generators — a shorthand for writing iterators**, using `function*` and `yield` instead of manually tracking `current`/`done`:

```js
function* rangeGen(from, to) {
  for (let n = from; n <= to; n++) {
    yield n;
  }
}

for (const n of rangeGen(1, 3)) console.log(n); // 1, 2, 3
```

Each `yield` pauses the function and hands a value out; calling `.next()` again resumes exactly where it left off. This is the same underlying mechanism `async`/`await` builds on in [Part 8](#9-part-8--asynchronous-javascript).
([Iterators and generators — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_generators))

[⬆ back to top](#table-of-contents)

---

## 8. Part 7 — Classes

Covers: `class`/`constructor`/`extends`/`super`, private fields, static members, getters/setters.

### Beginner

```js
class Animal {
  constructor(name) {
    this.name = name;
  }

  describe() {
    return `${this.name} is an animal`;
  }
}

const rex = new Animal("Rex");
console.log(rex.describe()); // "Rex is an animal"
```

`class` is syntax sugar over the prototype mechanism from [Part 5](#6-part-5--objects--prototypes) — `describe` lives on `Animal.prototype`, and `rex` delegates to it exactly like `rabbit` delegated to `animal` earlier.

### Working Knowledge

**Inheritance with `extends`/`super`:**

```js
class Dog extends Animal {
  constructor(name, breed) {
    super(name);      // must call super() before using `this` in a derived class
    this.breed = breed;
  }

  describe() {
    return `${super.describe()}, specifically a ${this.breed}`;
  }
}

const fido = new Dog("Fido", "Labrador");
console.log(fido.describe()); // "Fido is an animal, specifically a Labrador"
console.log(fido instanceof Animal); // true
```

**Wrong vs. right — forgetting `super()`:**

```js
// Wrong
class Cat extends Animal {
  constructor(name) {
    this.name = name; // ReferenceError: Must call super constructor before accessing 'this'
  }
}

// Right
class Cat2 extends Animal {
  constructor(name) {
    super(name); // sets up `this` via the parent constructor first
  }
}
```

**Getters/setters — compute a property on access, syntactically like a plain field:**

```js
class Circle {
  constructor(radius) {
    this.radius = radius;
  }
  get area() {
    return Math.PI * this.radius ** 2;
  }
}
const c = new Circle(2);
console.log(c.area); // 12.566... — called like a property, no ()
```

### Advanced

**Private fields (`#field`) — real, enforced privacy, not just a naming convention.**

```js
class BankAccount {
  #balance; // must be declared here, at the class body level

  constructor(initial) {
    this.#balance = initial;
  }

  deposit(amount) {
    this.#balance += amount;
  }

  get balance() {
    return this.#balance;
  }
}

const acc = new BankAccount(100);
acc.deposit(50);
console.log(acc.balance);  // 150
console.log(acc.#balance);  // SyntaxError: Private field '#balance' must be declared in an enclosing class
```

Unlike the old `_balance` underscore convention (which was never actually private — just a hint), `#balance` is inaccessible outside the class body, throwing a `SyntaxError` at *parse* time for any external access attempt. Supported broadly since 2021.
([Private properties — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_properties))

**Static members — belong to the class itself, not instances:**

```js
class IdGenerator {
  static #nextId = 1; // private AND static
  static next() {
    return IdGenerator.#nextId++;
  }
}
IdGenerator.next(); // 1
IdGenerator.next(); // 2
```

### Mastery

**`class` fields are own (instance) properties, not prototype properties — this affects `this` behavior with class methods used as callbacks:**

```js
class Timer {
  seconds = 0;
  // arrow function as a class field: bound to the instance at creation,
  // NOT a prototype method — safe to pass as a callback without losing `this`
  tick = () => {
    this.seconds++;
  };
}

const t = new Timer();
const detachedTick = t.tick;
detachedTick(); // works! this.seconds is still t.seconds — because tick is an arrow bound at construction
```

Compare this to the `this`-loss problem from [Part 5](#6-part-5--objects--prototypes) — this class-field-arrow pattern is the modern fix, avoiding `.bind(this)` in the constructor.

[⬆ back to top](#table-of-contents)

---

## 9. Part 8 — Asynchronous JavaScript

Covers: the event loop, callbacks → Promises → `async`/`await`, error handling.

### Beginner

JavaScript runs on a **single thread** — one thing at a time. "Async" doesn't mean parallel; it means *not blocking* that one thread while waiting on something slow (a timer, a network request, a file read).

```js
console.log("first");
setTimeout(() => console.log("third"), 0);
console.log("second");

// Output: first, second, third — even with a 0ms delay!
```

`setTimeout`'s callback never runs *during* the current synchronous code — it's queued and only runs once the call stack is empty.

### Working Knowledge

**The evolution: callbacks → Promises → `async`/`await` (same underlying mechanism, increasingly readable syntax).**

```js
// Callback style (old) — "callback hell" once you chain several steps
getUser(id, (user) => {
  getPosts(user.id, (posts) => {
    getComments(posts[0].id, (comments) => {
      console.log(comments);
    });
  });
});
```

```js
// Promise style — chainable, flattens the pyramid
getUser(id)
  .then((user) => getPosts(user.id))
  .then((posts) => getComments(posts[0].id))
  .then((comments) => console.log(comments))
  .catch((err) => console.error(err));
```

```js
// async/await — reads like synchronous code, same Promises underneath
async function loadComments(id) {
  try {
    const user = await getUser(id);
    const posts = await getPosts(user.id);
    const comments = await getComments(posts[0].id);
    console.log(comments);
  } catch (err) {
    console.error(err);
  }
}
```

`await` only works inside an `async function` (or, since ~2022, at a module's top level — see [Part 9](#10-part-9--modules)). It pauses that function (not the whole program) until the Promise settles.
([async function — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function))

**Wrong vs. right — sequential vs. parallel awaits:**

```js
// Wrong — awaiting one at a time when they don't depend on each other wastes time
const user = await fetchUser();       // waits ~200ms
const settings = await fetchSettings(); // THEN waits another ~200ms — 400ms total

// Right — start both requests immediately, await together
const [user2, settings2] = await Promise.all([fetchUser(), fetchSettings()]); // ~200ms total
```
([Promise.all — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all))

### Advanced

**The event loop — why `setTimeout(fn, 0)` still runs "later," and why Promises jump the queue ahead of it.**

```text
 Call Stack (sync code runs here, one frame at a time)
 ┌───────────────────────────┐
 │  current function          │
 └───────────────────────────┘
             │  empties out
             ▼
 ┌─────────────────────────────────────────────┐
 │  Microtask Queue  (Promise .then/.catch,       │  ← drained COMPLETELY
 │  queueMicrotask, async/await continuations)     │     before each macrotask
 └─────────────────────────────────────────────┘
             │
             ▼
 ┌─────────────────────────────────────────────┐
 │  Macrotask (Task) Queue  (setTimeout,           │  ← one task per loop tick,
 │  setInterval, I/O callbacks)                     │     then microtasks drain again
 └─────────────────────────────────────────────┘
```

```js
console.log("1 sync");
setTimeout(() => console.log("4 macrotask"), 0);
Promise.resolve().then(() => console.log("3 microtask"));
console.log("2 sync");

// Output: 1 sync, 2 sync, 3 microtask, 4 macrotask
```

The engine finishes *all* synchronous code, then drains the entire microtask queue, and only then takes one task off the macrotask queue — this is why a Promise callback always beats a `setTimeout(fn, 0)` even though both were "queued" around the same moment.
([Concurrency model and Event Loop — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Event_loop))

**`Promise.all` vs `Promise.allSettled` — failure behavior matters:**

```js
// Promise.all rejects immediately if ANY promise rejects — you lose the results of the ones that succeeded
const results = await Promise.all([fetchA(), fetchB(), fetchC()]); // one failure = whole thing throws

// Promise.allSettled always resolves, giving you a status per promise — nothing is lost
const outcomes = await Promise.allSettled([fetchA(), fetchB(), fetchC()]);
// [{status: "fulfilled", value: ...}, {status: "rejected", reason: ...}, ...]
```
([Promise.allSettled — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled))

### Mastery

**A `Promise` executor runs *synchronously*, immediately, the instant it's constructed — only `.then`/`await` are deferred:**

```js
console.log("1");
new Promise((resolve) => {
  console.log("2 — this runs immediately, synchronously");
  resolve();
}).then(() => console.log("4 — this is deferred to the microtask queue"));
console.log("3");

// Output: 1, 2, 3, 4
```

This trips people up constantly — "creating" a Promise is not the same as "awaiting" it; the work inside the executor function starts right away.

[⬆ back to top](#table-of-contents)

---

## 10. Part 9 — Modules

Covers: ESM `import`/`export`, dynamic `import()`. (Tiers collapsed — this is mostly a Beginner/Working Knowledge topic for the core language.)

### Beginner → Working Knowledge

```js
// math.js
export const PI = 3.14159;
export function square(x) {
  return x * x;
}
export default function add(a, b) { // one default export per module
  return a + b;
}
```

```js
// main.js
import add, { PI, square } from "./math.js"; // default + named in one line
console.log(add(2, 3), PI, square(4));

import * as math from "./math.js"; // namespace import — everything under one object
console.log(math.PI);
```

**Wrong vs. right — the default-vs-named mixup:**

```js
// Wrong — treating a default export as if it had named members you can destructure directly
import add from "./math.js";
const { PI } = add; // undefined — add is a function, not the module namespace

// Right — import default and named exports explicitly, as shown above
import add, { PI } from "./math.js";
```
([import — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import))

**Dynamic `import()`** — loads a module on demand, returns a Promise, usable inside regular code (not just modules):

```js
async function loadFeature() {
  const { runFeature } = await import("./feature.js");
  runFeature();
}
```

**Running ESM with Node:** either name the file `.mjs`, or set `"type": "module"` in `package.json` — otherwise Node treats `.js` as CommonJS (`require`/`module.exports`), which you'll still see in older code and some Node tooling configs. Top-level `await` (using `await` outside any `async function`, directly in a module) works in ESM modules without wrapping.
([JavaScript modules — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules))

[⬆ back to top](#table-of-contents)

---

## 11. Part 10 — Modern JS Grab Bag

Newer additions that don't fit neatly into the sections above but come up often.

**`Map` and `Set` — better-behaved alternatives to using plain objects/arrays as collections:**

```js
const scores = new Map();
scores.set("alice", 90);
scores.set("bob", 85);
scores.get("alice");     // 90
scores.has("carol");      // false
scores.size;               // 2
for (const [name, score] of scores) console.log(name, score);
```

Unlike a plain object, a `Map` can use *any* value as a key (not just strings/symbols), preserves insertion order reliably, and doesn't carry inherited prototype properties that could collide with real keys.

```js
const unique = new Set([1, 2, 2, 3, 3, 3]);
console.log([...unique]); // [1, 2, 3] — dedupes automatically
```
([Map — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map), [Set — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set))

**`Array.prototype.at()` — negative indexing without the arithmetic:**

```js
const arr = [10, 20, 30];
// Wrong-ish — works, but you have to do the math yourself
arr[arr.length - 1]; // 30

// Right — at() accepts negative indices directly
arr.at(-1); // 30
```
([Array.prototype.at — MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/at))

**`structuredClone()` — a real deep clone, built into the language/runtime, no libraries needed:**

```js
const original = { date: new Date(), nested: { a: 1 } };
original.self = original; // circular reference

// Wrong — the old JSON hack: loses Dates (becomes a string), throws on circular refs, drops functions silently
JSON.parse(JSON.stringify(original)); // throws: Converting circular structure to JSON

// Right — structuredClone handles Dates, Maps, Sets, circular references correctly
const clone = structuredClone(original);
console.log(clone.self === clone); // true — circular reference preserved correctly
```

Note it still can't clone functions or DOM nodes (throws `DataCloneError`) — for plain data, it's the modern default. Baseline supported since March 2022.
([structuredClone — MDN](https://developer.mozilla.org/en-US/docs/Web/API/structuredClone))

**`Object.entries`/`fromEntries` — converting between objects and arrays of pairs, handy with `map`/`filter`:**

```js
const prices = { apple: 1, banana: 0.5 };
const doubled = Object.fromEntries(
  Object.entries(prices).map(([name, price]) => [name, price * 2])
);
console.log(doubled); // { apple: 2, banana: 1 }
```

[⬆ back to top](#table-of-contents)

---

## 12. Cheat Sheets

### Equality & values

| Check | Use |
| --- | --- |
| Strict equality | `===` / `!==` (always default to this) |
| Both null and undefined | `value == null` (the one accepted `==` use) |
| NaN check | `Number.isNaN(x)`, never global `isNaN(x)` |
| Deep clone | `structuredClone(x)` |
| Falsy values (memorize all 8) | `false, 0, -0, 0n, "", null, undefined, NaN` |

### Declarations

| Keyword | Scope | Reassignable | Hoisting behavior |
| --- | --- | --- | --- |
| `const` | block | no | TDZ — error if used before declaration |
| `let` | block | yes | TDZ — error if used before declaration |
| `var` | function | yes | hoisted & initialized to `undefined` (avoid) |

### Functions

| Form | Own `this`? | Hoisted with body? |
| --- | --- | --- |
| `function foo() {}` | yes | yes |
| `const foo = function() {}` | yes | no (TDZ) |
| `const foo = () => {}` | no — inherits from enclosing scope | no (TDZ) |

### Async

| Tool | Behavior on one failure |
| --- | --- |
| `Promise.all` | rejects immediately, other results lost |
| `Promise.allSettled` | always resolves; per-item status/value/reason |
| `await` | pauses the enclosing `async function` only, not the whole program |

### Array: mutating vs. non-mutating

| Mutates original | Returns new array |
| --- | --- |
| `push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse` | `map`, `filter`, `slice`, `concat`, `toSorted`, `toReversed`, `toSpliced`, `with` |

[⬆ back to top](#table-of-contents)

---

## 13. Suggested Learning Order

1. **Orientation & Setup** — get `node` running, confirm you can execute a file.
2. **Values & Types → Variables & Scope → Operators/Coercion/Control Flow** — these three build directly on each other and form the language's bedrock; don't skip ahead until `let`/`const`, `===`, and truthy/falsy feel automatic.
3. **Functions & Closures** — the single highest-leverage topic in the language. Sit with the closure examples until the counter/loop-bug pair genuinely makes sense, not just "looks right."
4. **Objects & Prototypes** — especially `this` and the prototype chain diagram; this is what makes Classes (next) click instead of feeling like magic syntax.
5. **Arrays & Iteration** — `map`/`filter`/`reduce` first (used constantly), iterators/generators last (useful, but lower daily frequency).
6. **Classes** — now that prototypes make sense, `class`/`extends`/`#private` should read as sugar, not new concepts.
7. **Asynchronous JavaScript** — the event loop diagram is worth re-reading after you've written a few `async`/`await` functions and seen the ordering surprise you once.
8. **Modules** — quick, mostly mechanical once everything above is solid.
9. **Modern JS Grab Bag** — pick these up as needed; nothing here is foundational, all of it is convenience.

[⬆ back to top](#table-of-contents)

---

## 14. Quick Self-Check

- What's the difference between `undefined` and `null`, and which one does a missing function argument become?
- Why does `typeof null === "object"`, and is that a bug or intentional behavior?
- What's the actual difference between `const` preventing reassignment and preventing mutation? Give an example of each.
- Explain the classic `var` loop bug (`setTimeout` inside a `for (var i...)` loop) and why `let` fixes it — in terms of scope, not just "because it does."
- What does `0 || 10` return, and how is that different from `0 ?? 10`? When would the difference actually bite you?
- Write a closure-based counter from scratch (`makeCounter`) without looking at Part 4 — what would break if you used `var` instead of `let` inside it?
- Why does an arrow function used as an object method lose access to the object via `this`, when a regular function method doesn't?
- Draw (or describe) the prototype chain for an object created with `Object.create(parent)` — what does `obj.someInheritedMethod()` actually do, mechanically?
- What's the difference between a private field (`#x`) and the old `_x` underscore convention — in terms of what actually happens if outside code tries to access it?
- Put these in the order they'd actually log: a `console.log`, a `setTimeout(fn, 0)`, and a `Promise.resolve().then(fn)`, all triggered together. Why that order?
- When would `Promise.allSettled` be the correct choice over `Promise.all`, concretely?
- What's wrong with `JSON.parse(JSON.stringify(obj))` as a "deep clone," and what replaces it in modern JS?

[⬆ back to top](#table-of-contents)
