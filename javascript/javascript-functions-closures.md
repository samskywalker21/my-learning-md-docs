# JavaScript — Functions & Closures (Part 4)

Function forms, parameters, `this`, arrow functions, closures, and higher-order patterns. This is the Part where JavaScript stops resembling other languages you may know, and it is worth reading slowly.

> **About this document.** Part of the [JavaScript Mastery Guide](./javascript-mastery-guide.md) set — see that document's "About This Document" section for the framing (curiosity-driven, hands-on), scope, tier conventions, sourcing rule and update instructions. Written against ECMAScript 2026; Baseline data verified September 6, 2026.

**Prerequisite:** [Part 2 — Variables & Scope](./javascript-foundations.md#2-part-2--variables-scope--the-tdz). Closures are the scope chain plus time; if the scope chain is not solid yet, go back first.

---

## Table of Contents

1. [Part 4 — Functions & Closures](#1-part-4--functions--closures)
   - [Beginner — declaring and calling](#part-4-beginner)
   - [Working Knowledge — parameters, arrows, first-class functions](#part-4-working)
   - [Advanced — `this`, and closures over time](#part-4-advanced)
   - [Mastery — currying, memoisation, and what closures cost](#part-4-mastery)
   - [Cheat sheet](#part-4-cheatsheet)

---

## 1. Part 4 — Functions & Closures

<a id="part-4"></a>

### Beginner — declaring and calling

<a id="part-4-beginner"></a>

There are three forms you will write, and they differ in more than syntax.

```js
// 1. Function declaration — hoisted completely.
function add(a, b) {
  return a + b;
}

// 2. Function expression — the binding follows normal let/const rules.
const subtract = function (a, b) {
  return a - b;
};

// 3. Arrow function — concise, and does NOT bind its own `this`.
const multiply = (a, b) => a * b;
```

**Declarations are hoisted with their body**, so you can call them before the line they appear on. Expressions and arrows are not — the *binding* hoists into the TDZ, but the function value is not assigned until execution reaches the line.

```js
sayHi();          // ✅ "hi" — declarations hoist entirely
function sayHi() { console.log("hi"); }

sayBye();         // ❌ ReferenceError: Cannot access 'sayBye' before initialization
const sayBye = () => console.log("bye");
```

This is the one real argument for declarations: helper functions defined at the bottom of a module can be used at the top, so the file reads top-down in importance order. The argument for `const` + arrow is uniformity with every other binding. Both are defensible; pick one per codebase.

### Arrow function shorthand

```js
(a, b) => a * b            // implicit return of the expression
a => a * 2                 // single parameter: parens optional
() => 42                   // no parameters: parens required
(a) => { return a * 2; }   // block body: `return` required
() => ({ ok: true })       // returning an object literal needs parens
```

```js
// ❌ The braces are read as a function body, not an object literal.
const make = () => { ok: true };
make();                    // undefined

// ✅ Wrap the object in parentheses.
const make = () => ({ ok: true });
make();                    // { ok: true }
```

### Return values

A function with no `return` returns `undefined`. And **automatic semicolon insertion** makes a `return` on its own line a trap:

```js
// ❌ ASI inserts a semicolon right after `return`.
function get() {
  return
    { value: 42 };
}
get();     // undefined

// ✅ Keep the value on the same line as `return`.
function get() {
  return {
    value: 42,
  };
}
```

> **Try It.** Run both `get` definitions. Expected: `undefined`, then `{ value: 42 }`. This is the single most damaging ASI hazard in the language; every other one is cosmetic.

### Working Knowledge — parameters, arrows, first-class functions

<a id="part-4-working"></a>

#### Default, rest, and destructured parameters

```js
// Defaults — applied only when the argument is `undefined`.
function greet(name = "friend", greeting = "Hello") {
  return `${greeting}, ${name}!`;
}
greet();                    // "Hello, friend!"
greet("Sam");               // "Hello, Sam!"
greet(undefined, "Hi");     // "Hi, friend!"
greet(null);                // "Hello, null!"  ← null is a value, not an absence

// Defaults are expressions, evaluated at call time, left to right.
function makeId(prefix = "id", suffix = Date.now(), full = `${prefix}-${suffix}`) {
  return full;
}

// Rest — collects the remaining arguments into a real array.
function sum(first, ...rest) {
  return rest.reduce((t, n) => t + n, first);
}
sum(1, 2, 3, 4);            // 10

// Destructured parameters — the standard "options object" shape.
function connect({ host = "localhost", port = 5432, ssl = false } = {}) {
  return `${ssl ? "https" : "http"}://${host}:${port}`;
}
connect();                             // "http://localhost:5432"
connect({ port: 8080, ssl: true });    // "https://localhost:8080"
```

That trailing `= {}` matters. Without it, calling `connect()` with no arguments tries to destructure `undefined` and throws:

```js
// ❌ Throws when called with no arguments.
function connect({ host = "localhost" }) { /* … */ }
connect();     // TypeError: Cannot destructure property 'host' of 'undefined'

// ✅ Default the whole parameter to an empty object.
function connect({ host = "localhost" } = {}) { /* … */ }
connect();     // fine
```

#### `arguments` — legacy, and why arrows do not have it

Non-arrow functions get an implicit `arguments` object: array-*like*, but not an array.

```js
// ❌ Legacy. arguments has no .map, .filter, .reduce.
function sum() {
  return Array.prototype.slice.call(arguments).reduce((a, b) => a + b, 0);
}

// ✅ Rest parameters give a real array, and are visible in the signature.
const sum = (...nums) => nums.reduce((a, b) => a + b, 0);
```

Arrow functions have no `arguments` of their own — a reference inside an arrow reaches the enclosing function's. Use rest parameters and the question never comes up.

#### Functions are values

Functions are objects. They can be stored, passed, returned, and given properties.

```js
const ops = {
  add: (a, b) => a + b,
  sub: (a, b) => a - b,
};
ops.add(2, 3);                     // 5

[1, 2, 3].map(n => n * 2);         // functions as arguments
const twice = f => x => f(f(x));   // functions as return values
twice(n => n + 3)(10);             // 16

function counter() { }
counter.calls = 0;                 // functions can hold properties
counter.name;                      // "counter"  — inferred, read-only-ish
counter.length;                    // 0          — declared parameter count
```

`fn.length` counts parameters *before* the first default or rest parameter, which occasionally surprises library authors:

```js
((a, b) => 0).length;         // 2
((a, b = 1) => 0).length;     // 1
((a, ...r) => 0).length;      // 1
```

> **Try It.** Run `const f = (a, b = 2, c) => 0; f.length`. Expected: `1`. The count stops at the first default — `c` is not counted even though it comes after.

### Advanced — `this`, and closures over time

<a id="part-4-advanced"></a>

#### The four bindings of `this`

`this` is not determined by where a function is defined. For ordinary functions it is determined by **how the function is called**. There are four rules, in precedence order:

```text
Highest precedence
  1. new binding        new Fn()              → this = the fresh object
  2. explicit binding   fn.call/apply/bind    → this = what you passed
  3. implicit binding   obj.fn()              → this = obj
  4. default binding    fn()                  → this = undefined (strict)
Lowest precedence                                    or globalThis (sloppy)

  ARROW FUNCTIONS OPT OUT OF ALL FOUR.
  An arrow has no `this`; it reads the enclosing scope's, lexically.
```

```js
const obj = {
  name: "obj",
  show() { return this.name; },
};

obj.show();                       // "obj"        — rule 3, implicit
const loose = obj.show;
loose();                          // undefined    — rule 4: `this` is undefined in strict mode
loose.call({ name: "other" });    // "other"      — rule 2, explicit
new (function () { this.name = "new"; return this.name; })();   // rule 1
```

**The detached-method bug** is the single most common `this` failure, and it appears the moment you pass a method as a callback:

```js
// ❌ The method loses its receiver when passed as a value.
class Timer {
  constructor() { this.count = 0; }
  tick() { this.count++; console.log(this.count); }
}
const t = new Timer();
setInterval(t.tick, 1000);        // TypeError: Cannot read properties of undefined
```

```js
// ✅ Option A — wrap in an arrow, which keeps the call shape `t.tick()`.
setInterval(() => t.tick(), 1000);

// ✅ Option B — bind once.
setInterval(t.tick.bind(t), 1000);

// ✅ Option C — define the method as a class field holding an arrow.
class Timer {
  count = 0;
  tick = () => { this.count++; console.log(this.count); };
}
setInterval(new Timer().tick, 1000);   // works — the arrow captured `this` at construction
```

Option C is worth understanding rather than reaching for reflexively: a class-field arrow is a **per-instance property**, not a prototype method. A thousand instances means a thousand function objects, and it is not on the prototype so it cannot be overridden by a subclass in the usual way. Use it for callbacks you hand out; use ordinary methods otherwise.

#### `call`, `apply`, `bind`

```js
function intro(greeting, punct) {
  return `${greeting}, I am ${this.name}${punct}`;
}
const sam = { name: "Sam" };

intro.call(sam, "Hi", "!");          // args listed
intro.apply(sam, ["Hi", "!"]);       // args as an array
const bound = intro.bind(sam, "Hi"); // returns a new function, args partially applied
bound("!");                          // "Hi, I am Sam!"
```

`bind` is permanent — a bound function cannot be re-bound:

```js
const b = intro.bind({ name: "A" });
b.call({ name: "B" }, "Hi", "!");    // "Hi, I am A!" — the call's `this` is ignored
```

Since ES6, `apply`'s array-spreading role is usually better served by spread syntax:

```js
Math.max.apply(null, nums);   // legacy
Math.max(...nums);            // ✅ modern equivalent
```

#### Arrow functions and `this`

An arrow has no `this` binding at all. `this` inside an arrow resolves up the scope chain like any other free variable — which makes arrows correct by default in callbacks and wrong by default as methods.

```js
// ❌ Arrow as an object method: `this` is the enclosing scope, not the object.
const counter = {
  count: 0,
  inc: () => { this.count++; },     // `this` is module scope / undefined
};
counter.inc();
counter.count;                       // 0 — never incremented

// ✅ Method shorthand for methods.
const counter = {
  count: 0,
  inc() { this.count++; },
};
```

```js
// ❌ Ordinary function as a callback loses `this`.
class Store {
  constructor() { this.items = []; }
  loadAll(names) {
    names.forEach(function (n) {
      this.items.push(n);            // TypeError — `this` is undefined
    });
  }
}

// ✅ Arrow callback inherits `this` from loadAll.
class Store {
  constructor() { this.items = []; }
  loadAll(names) {
    names.forEach(n => this.items.push(n));
  }
}
```

The rule that covers ~95% of cases: **method → shorthand method; callback → arrow.**

> **Try It.** Run:
> ```js
> const o = { n: "obj", a: () => this?.n, m() { return this.n; } };
> console.log(o.a(), o.m());
> ```
> Expected: `undefined obj`. Same call syntax, opposite `this` — because one is an arrow.

#### Closures

A **closure** is a function together with the scope it was created in. It keeps that scope alive after the enclosing function has returned.

```js
function makeCounter() {
  let count = 0;                       // not garbage collected — the closure holds it

  return {
    increment() { return ++count; },
    get value()  { return count; },
  };
}

const c = makeCounter();
c.increment();      // 1
c.increment();      // 2
c.value;            // 2
c.count;            // undefined — genuinely inaccessible from outside
```

```text
After makeCounter() returns:

  ┌─ closure scope (kept alive) ─┐
  │  count: 2                    │◀───┐
  └──────────────────────────────┘    │
                                      │ both functions reference
  c = { increment ──────────────────▶ ┤ the SAME scope
        value     ──────────────────▶ ┘
      }
```

That shared-scope detail is the whole mechanism. `increment` and `value` see the same `count` because they closed over the same environment record, not over a copy.

**Closures capture bindings, not values.** This is what makes the `var` loop bug work the way it does:

```js
function demo() {
  let x = 1;
  const read = () => x;      // captures the binding
  x = 2;
  return read();             // 2, not 1
}
```

Which is exactly why `for (var …)` produces `3, 3, 3` and `for (let …)` produces `0, 1, 2` — one shared binding versus a fresh binding per iteration ([Part 2](./javascript-foundations.md#part-2-working)).

**Real Scenario — the closure that leaked a request.** A service cached results in a module-level `Map` keyed by user id, and the cached value was a closure created inside the request handler. Because the closure captured the enclosing scope, it kept a reference to the entire `req` object — headers, body, socket — for every cached user. Memory grew steadily until the pod OOM-killed itself nightly. The fix was to capture only what was needed:

```js
// ❌ The closure pins the whole request object in the cache.
function handler(req, res) {
  cache.set(req.user.id, () => computeFrom(req.user.plan));
}

// ✅ Extract the value first; the closure captures only a string.
function handler(req, res) {
  const plan = req.user.plan;
  cache.set(req.user.id, () => computeFrom(plan));
}
```

The general principle: a closure keeps its *entire* enclosing scope reachable, not just the variables it mentions in modern engines' worst case. Destructure what you need at the boundary.

### Mastery — currying, memoisation, and what closures cost

<a id="part-4-mastery"></a>

#### Higher-order function patterns

```js
// Partial application — fix some arguments now, the rest later.
const partial = (fn, ...preset) => (...later) => fn(...preset, ...later);
const log = (level, msg) => `[${level}] ${msg}`;
const warn = partial(log, "WARN");
warn("disk almost full");             // "[WARN] disk almost full"

// Currying — one argument at a time.
const curry = (fn) =>
  function curried(...args) {
    return args.length >= fn.length
      ? fn.apply(this, args)
      : (...more) => curried.apply(this, [...args, ...more]);
  };

const volume = curry((l, w, h) => l * w * h);
volume(2)(3)(4);        // 24
volume(2, 3)(4);        // 24
volume(2, 3, 4);        // 24

// Composition — right to left, like maths.
const compose = (...fns) => (x) => fns.reduceRight((acc, f) => f(acc), x);
const pipe    = (...fns) => (x) => fns.reduce((acc, f) => f(acc), x);

const slugify = pipe(
  (s) => s.trim(),
  (s) => s.toLowerCase(),
  (s) => s.replace(/\s+/g, "-"),
);
slugify("  Hello World  ");           // "hello-world"
```

`curry` above relies on `fn.length`, which is why it breaks on functions with default or rest parameters — a real limitation of every hand-rolled curry implementation, and worth knowing before you ship one.

#### Memoisation

```js
function memoize(fn, keyFn = (...a) => JSON.stringify(a)) {
  const cache = new Map();
  return function (...args) {
    const key = keyFn(...args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

const slowFib = (n) => (n < 2 ? n : slowFib(n - 1) + slowFib(n - 2));
const fastFib = memoize((n) => (n < 2 ? n : fastFib(n - 1) + fastFib(n - 2)));

fastFib(40);      // instant
slowFib(40);      // ~1s — exponential
```

Two things this naive version gets wrong that production versions must handle:

1. **`cache.has` before `cache.get`** — necessary because a legitimately cached `undefined` is indistinguishable from a miss otherwise. Since [Baseline 2026](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map/getOrInsert) you can use `cache.getOrInsertComputed(key, () => fn(...args))` and skip the double lookup entirely — see [Part 14](./javascript-metaprogramming-modern.md).
2. **Unbounded growth.** A `Map` keyed by arguments never evicts. For object arguments, a `WeakMap` lets entries be collected when the key object dies ([Part 13](./javascript-metaprogramming-modern.md)); for primitives you need an explicit size cap.

#### The module pattern, then and now

Before ES modules, closures *were* the encapsulation mechanism. You will still meet this in older code:

```js
// Legacy — the IIFE module pattern. Recognise it; don't write it.
var Counter = (function () {
  var count = 0;                    // private by closure
  return {
    increment: function () { return ++count; },
  };
})();
```

Today the same privacy comes from module scope (nothing is exported unless you say so) or `#private` class fields ([Part 7](./javascript-objects-classes.md)). The IIFE version survives only in bundled legacy scripts.

#### Function identity, and the re-render trap

Every evaluation of a function expression creates a **new function object**. Two functions with identical source are never `===`.

```js
const a = () => 1;
const b = () => 1;
a === b;          // false
```

This is invisible until something compares functions by identity — a `Set`, a memo cache, `removeEventListener`, or a UI framework's props diff.

```js
// ❌ The listener cannot be removed: a different function object each time.
element.addEventListener("click", () => handle());
element.removeEventListener("click", () => handle());   // removes nothing

// ✅ Keep a reference to the exact function you registered.
const onClick = () => handle();
element.addEventListener("click", onClick);
element.removeEventListener("click", onClick);
```

#### What closures actually cost

A closure keeps its enclosing environment record alive for as long as the closure is reachable. Modern engines do escape analysis and will often keep only the variables actually referenced — but this is an optimisation, not a guarantee, and it does not apply when `eval` or `with` are in scope, or across some debugger-attached paths.

The practical consequences:

- A long-lived closure (a cache entry, an event listener, a module-level callback) pins whatever it captured. That is the leak in the Real Scenario above.
- A short-lived closure (a `map` callback) costs an allocation the engine usually elides. Do not contort code to avoid these.
- Removing listeners and clearing caches is what actually prevents closure-driven leaks — not avoiding closures.

> **Try It.** Run this in Node with `--expose-gc` to watch a closure keep memory alive:
> ```js
> let holder;
> function build() {
>   const big = new Array(1e6).fill("x");
>   holder = () => big.length;     // the closure pins `big`
> }
> build();
> global.gc();
> console.log(process.memoryUsage().heapUsed / 1e6, "MB");  // ~8+ MB
> holder = null;
> global.gc();
> console.log(process.memoryUsage().heapUsed / 1e6, "MB");  // back to ~4 MB
> ```
> Run with `node --expose-gc leak.js`. The exact numbers vary; the drop after `holder = null` is the point.

### Part 4 quick reference

<a id="part-4-cheatsheet"></a>

| Form | Hoisted | Own `this` | Own `arguments` | `new`-able | Use for |
|---|---|---|---|---|---|
| `function f() {}` | ✅ fully | ✅ | ✅ | ✅ | Top-level helpers |
| `const f = function () {}` | Binding only (TDZ) | ✅ | ✅ | ✅ | Rare; prefer arrow |
| `const f = () => {}` | Binding only (TDZ) | ❌ lexical | ❌ lexical | ❌ | Callbacks, one-liners |
| `{ f() {} }` shorthand | n/a | ✅ | ✅ | ❌ | Object/class methods |
| `class { f = () => {} }` | n/a | ❌ lexical (per instance) | ❌ | ❌ | Methods handed out as callbacks |

**`this` precedence:** `new` > `call`/`apply`/`bind` > `obj.method()` > plain call (`undefined` in strict mode). Arrows opt out entirely.

| Task | Do this |
|---|---|
| Keep `this` in a callback | Arrow function |
| Pass a method as a callback | `() => obj.method()` or `obj.method.bind(obj)` |
| Collect variadic arguments | `(...args)` rest, not `arguments` |
| Spread an array into arguments | `f(...arr)`, not `f.apply(null, arr)` |
| Options object with all-optional keys | `function f({ a = 1 } = {})` |
| Private state | Module scope, or `#private` fields |
| Cache pure results | `memoize`, with a bounded or weak cache |
| Remove an event listener | Register a named reference, not an inline arrow |

[↑ Back to top](#table-of-contents)

---

**Next:** [Parts 5 & 7 — Objects & Classes](./javascript-objects-classes.md) · **Up:** [Mastery Guide overview](./javascript-mastery-guide.md)

*Written against ECMAScript 2026. Baseline data verified September 6, 2026.*
