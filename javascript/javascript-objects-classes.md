# JavaScript — Objects & Classes (Parts 5 & 7)

Property access and descriptors, the prototype chain, and the `class` syntax that sits on top of it. Parts 5 and 7 are together in one file because the second is only comprehensible in terms of the first.

> **About this document.** Part of the [JavaScript Mastery Guide](./javascript-mastery-guide.md) set — see that document's "About This Document" section for the framing (curiosity-driven, hands-on), scope, tier conventions, sourcing rule and update instructions. Written against ECMAScript 2026; Baseline data verified September 6, 2026.

**Prerequisite:** [Part 4 — Functions & Closures](./javascript-functions-closures.md), especially the `this` rules.

---

## Table of Contents

1. [Part 5 — Objects, Descriptors & Prototypes](#1-part-5--objects-descriptors--prototypes)
   - [Beginner](#part-5-beginner) · [Working Knowledge](#part-5-working) · [Advanced](#part-5-advanced) · [Mastery](#part-5-mastery) · [Cheat sheet](#part-5-cheatsheet)
2. [Part 7 — Classes](#2-part-7--classes)
   - [Beginner](#part-7-beginner) · [Working Knowledge](#part-7-working) · [Advanced](#part-7-advanced) · [Mastery](#part-7-mastery) · [Cheat sheet](#part-7-cheatsheet)

---

## 1. Part 5 — Objects, Descriptors & Prototypes

<a id="part-5"></a>

### Beginner — creating and accessing

<a id="part-5-beginner"></a>

An object is a collection of key–value pairs. Keys are strings or symbols; values are anything.

```js
const user = {
  name: "Sam",
  age: 30,
  "favourite colour": "blue",     // quotes needed: the key has a space
  greet() { return `Hi, ${this.name}`; },
};

user.name;                  // "Sam"          — dot notation
user["favourite colour"];   // "blue"         — bracket notation, required here
const key = "age";
user[key];                  // 30             — bracket notation for dynamic keys
user.greet();               // "Hi, Sam"

user.email = "s@x.com";     // add
delete user.age;            // remove
"name" in user;             // true
```

Dot notation is for keys you know at write time; brackets are for keys that are computed, contain spaces, or are symbols. A very common beginner error is using dots with a variable:

```js
const key = "name";
user.key;      // ❌ undefined — looks for a literal property called "key"
user[key];     // ✅ "Sam"
```

#### Shorthand and computed keys

```js
const name = "Sam", age = 30;

const user = { name, age };              // shorthand: { name: name, age: age }

const field = "status";
const record = {
  [field]: "active",                     // computed key → { status: "active" }
  [`${field}_at`]: Date.now(),           // any expression works
};
```

#### Checking for a property

```js
const user = { name: "Sam", email: undefined };

user.email !== undefined;         // false — but the property DOES exist
"email" in user;                  // true  — checks own + inherited
Object.hasOwn(user, "email");     // true  — own only. Preferred.
user.hasOwnProperty("email");     // true  — legacy; breaks on null-prototype objects
```

`Object.hasOwn` ([Baseline widely available](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/hasOwn), ES2022) is the modern replacement for `hasOwnProperty`. The old form fails on objects created with `Object.create(null)`, which have no `hasOwnProperty` to call:

```js
const bare = Object.create(null);
bare.x = 1;
bare.hasOwnProperty("x");         // ❌ TypeError: bare.hasOwnProperty is not a function
Object.hasOwn(bare, "x");         // ✅ true
```

> **Try It.** Run `const o = { a: undefined }; console.log(o.a === undefined, "a" in o, Object.hasOwn(o, "a"), Object.hasOwn(o, "b"))`. Expected: `true true true false`. The first check cannot distinguish "absent" from "present but undefined"; the other two can.

### Working Knowledge — iterating, copying, and the built-in collections

<a id="part-5-working"></a>

#### Iterating an object

```js
const user = { name: "Sam", age: 30 };

Object.keys(user);      // ["name", "age"]
Object.values(user);    // ["Sam", 30]
Object.entries(user);   // [["name","Sam"], ["age",30]]

for (const [k, v] of Object.entries(user)) {
  console.log(k, v);
}

// Round-trip: transform an object through its entries.
const upper = Object.fromEntries(
  Object.entries(user).map(([k, v]) => [k.toUpperCase(), v]),
);
// { NAME: "Sam", AGE: 30 }
```

All three skip symbol keys and inherited properties, which is almost always what you want. Avoid `for…in` — it walks the prototype chain:

```js
// ❌ for...in includes inherited enumerable properties.
for (const key in user) { /* may include things you did not put there */ }

// ✅ Object.keys / entries — own, enumerable, string-keyed only.
for (const key of Object.keys(user)) { /* … */ }
```

#### Property order

Object key order is specified, not arbitrary ([spec: OrdinaryOwnPropertyKeys](https://tc39.es/ecma262/2026/)):

1. Integer-like keys, ascending numerically.
2. Other string keys, in insertion order.
3. Symbol keys, in insertion order.

```js
const o = { b: 1, 2: 2, a: 3, 1: 4 };
Object.keys(o);      // ["1", "2", "b", "a"]  ← numeric keys jumped to the front
```

If insertion order matters and your keys might be numeric strings, use a `Map`.

#### Copying

```js
const original = { name: "Sam", address: { city: "London" } };

// Shallow — nested objects are SHARED.
const shallow = { ...original };
shallow.address.city = "Oslo";
original.address.city;             // "Oslo"  ← mutated the original

// Deep — structuredClone handles cycles, Map, Set, Date, TypedArray.
const deep = structuredClone(original);
deep.address.city = "Paris";
original.address.city;             // "Oslo"  ← untouched
```

```js
// ❌ The old JSON round-trip. Silently destroys data.
const copy = JSON.parse(JSON.stringify(original));
// Dates become strings; undefined, functions and symbols vanish;
// NaN and Infinity become null; cycles throw.

// ✅ structuredClone — Baseline, and correct.
const copy = structuredClone(original);
```

`structuredClone` ([Baseline widely available](https://developer.mozilla.org/en-US/docs/Web/API/Window/structuredClone)) cannot clone functions, DOM nodes, or symbols, and it returns a plain object for class instances — the prototype is not preserved. Those are its real limits; everything else the JSON trick got wrong, it gets right.

`Object.assign(target, ...sources)` is the pre-spread shallow merge. It differs from spread in one meaningful way: it **triggers setters** on the target, where spread defines plain data properties.

```js
const target = { set x(v) { console.log("setter ran", v); } };
Object.assign(target, { x: 1 });     // logs "setter ran 1"
const spread = { ...target, x: 1 };  // no setter — plain property
```

#### `Map` and `Set` — when a plain object is the wrong tool

```js
const map = new Map();
map.set("a", 1);
map.set(42, "number key");        // any type as key
map.set({ id: 1 }, "object key"); // object identity as key
map.get("a");                     // 1
map.size;                         // 3
map.has(42);                      // true
for (const [k, v] of map) { }     // insertion order, guaranteed

const set = new Set([1, 2, 2, 3]);
set.size;                         // 3 — duplicates dropped
set.has(2);                       // true
[...new Set(array)];              // the standard dedupe idiom
```

| | Plain object | `Map` |
|---|---|---|
| Key types | String, symbol | Any value |
| Ordering | Integer keys first, then insertion | Pure insertion order |
| Size | `Object.keys(o).length` | `map.size` |
| Prototype keys | Inherits from `Object.prototype` | None |
| Iteration | `Object.entries` | Directly iterable |
| JSON-serialisable | ✅ | ❌ (convert first) |
| Frequent add/delete | Slower | Optimised for it |

Use a `Map` when keys are dynamic, non-string, or numerous. Use a plain object for fixed, known-at-write-time shapes and anything you will `JSON.stringify`.

The **prototype-pollution hazard** is a security-relevant reason to prefer `Map` for user-controlled keys:

```js
// ❌ A user-supplied key of "__proto__" can corrupt Object.prototype.
const store = {};
store[userKey] = userValue;      // userKey === "__proto__" → trouble

// ✅ Map treats "__proto__" as an ordinary key.
const store = new Map();
store.set(userKey, userValue);

// ✅ Or a null-prototype object.
const store = Object.create(null);
store[userKey] = userValue;
```

### Advanced — descriptors, getters/setters, and immutability

<a id="part-5-advanced"></a>

Every property is really a **descriptor** — a record of the value plus three flags.

```js
const user = { name: "Sam" };
Object.getOwnPropertyDescriptor(user, "name");
// { value: "Sam", writable: true, enumerable: true, configurable: true }
```

| Flag | Meaning | Default via `obj.x = 1` | Default via `defineProperty` |
|---|---|---|---|
| `writable` | Value can be reassigned | `true` | `false` |
| `enumerable` | Shows up in `Object.keys`, spread, `for…in`, `JSON.stringify` | `true` | `false` |
| `configurable` | Can be deleted or redefined | `true` | `false` |

Those inverted defaults are the trap: `Object.defineProperty` with a bare `{ value }` creates a read-only, non-enumerable, non-configurable property.

```js
const o = {};
Object.defineProperty(o, "id", { value: 1 });
o.id = 2;                    // silently ignored (throws in strict mode)
Object.keys(o);              // []  — invisible to iteration
JSON.stringify(o);           // "{}"
delete o.id;                 // false — cannot be removed
```

```js
// ✅ Be explicit when you want ordinary behaviour.
Object.defineProperty(o, "id", {
  value: 1, writable: true, enumerable: true, configurable: true,
});
```

That non-enumerable default is a *feature* when used deliberately — it is how you attach internal bookkeeping that will not leak into `JSON.stringify` or a spread copy.

#### Accessor properties

```js
const account = {
  _balanceCents: 0,

  get balance() {
    return (this._balanceCents / 100).toFixed(2);
  },

  set balance(pounds) {
    if (typeof pounds !== "number" || Number.isNaN(pounds)) {
      throw new TypeError("balance must be a number");
    }
    this._balanceCents = Math.round(pounds * 100);
  },
};

account.balance = 12.5;     // calls the setter
account.balance;            // "12.50" — calls the getter
```

Getters look like data but run code. Two consequences worth internalising: a getter that does real work makes an innocuous-looking property read expensive, and a getter that throws turns a debugger hover or a `console.log` into an exception. Keep them cheap and total.

#### Freezing, sealing, preventing extensions

```js
const o = { a: 1, nested: { b: 2 } };

Object.preventExtensions(o);  // no new properties; existing ones still writable/deletable
Object.seal(o);               // + no deletions (configurable: false)
Object.freeze(o);             // + no writes    (writable: false)

Object.isFrozen(o);           // true
o.a = 99;                     // silently ignored (throws in strict mode)
o.nested.b = 99;              // ✅ WORKS — freeze is shallow
```

```js
// ✅ Deep freeze, when you actually need it.
function deepFreeze(obj) {
  for (const value of Object.values(obj)) {
    if (value && typeof value === "object" && !Object.isFrozen(value)) {
      deepFreeze(value);
    }
  }
  return Object.freeze(obj);
}
```

> **Try It.** Run:
> ```js
> "use strict";
> const cfg = Object.freeze({ retries: 3, limits: { max: 10 } });
> cfg.limits.max = 999;
> console.log(cfg.limits.max);
> try { cfg.retries = 5; } catch (e) { console.log(e.constructor.name); }
> ```
> Expected: `999` then `TypeError`. The nested write succeeded; the top-level one threw. That contrast *is* "freeze is shallow".

#### The prototype chain

Every object has an internal `[[Prototype]]` link. When a property is not found on the object, the lookup follows that link, and keeps following until it hits `null`.

```text
const arr = [1, 2, 3];
arr.map(...)

  arr  ───────────▶  Array.prototype  ───────────▶  Object.prototype  ──▶ null
  {0:1,1:2,2:3,      { map, filter,                 { toString,
   length: 3}          push, slice, … }               hasOwnProperty, … }
       │                     ▲
       │  "map"? no          │ "map"? yes — call it with this = arr
       └─────────────────────┘
```

```js
const arr = [1, 2, 3];
Object.hasOwn(arr, "map");                    // false — not on the array itself
Object.getPrototypeOf(arr) === Array.prototype;         // true
Object.getPrototypeOf(Array.prototype) === Object.prototype;  // true
Object.getPrototypeOf(Object.prototype);      // null — end of the chain
```

Creating and inspecting prototypes:

```js
const base = { greet() { return `Hi, ${this.name}`; } };
const sam = Object.create(base);              // sam's [[Prototype]] is base
sam.name = "Sam";
sam.greet();                                  // "Hi, Sam" — found on base, `this` is sam

Object.getPrototypeOf(sam) === base;          // true
base.isPrototypeOf(sam);                      // true
```

```js
// ❌ __proto__ — deprecated, and mutating it deoptimises the object badly.
obj.__proto__ = base;

// ✅ Set it at creation, or with the explicit API.
const obj = Object.create(base);
Object.setPrototypeOf(obj, base);   // works, but avoid on hot objects
```

[MDN is explicit](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/proto) that `__proto__` is deprecated and normatively optional; use `Object.getPrototypeOf`/`setPrototypeOf`. Changing an object's prototype after creation forces engines to discard the optimised shape they had built for it, so treat `setPrototypeOf` as a construction-time tool only.

**Writes never walk the chain.** Reads do; assignments always create an own property on the target:

```js
const base = { count: 0 };
const child = Object.create(base);
child.count;              // 0 — read from base
child.count++;            // reads base.count (0), writes own child.count (1)
base.count;               // 0 — untouched
Object.hasOwn(child, "count");   // true — now shadowed
```

This asymmetry is why "shared state on the prototype" almost never behaves the way people expect, and why mutable prototype properties are a bad idea.

### Mastery — `Object.create(null)`, `Symbol.hasInstance`, and shapes

<a id="part-5-mastery"></a>

#### Null-prototype objects

```js
const dict = Object.create(null);
dict.constructor;        // undefined
dict.toString;           // undefined
"toString" in dict;      // false — nothing inherited at all

// Downside: no toString, so it breaks in string contexts.
`${dict}`;               // TypeError: Cannot convert object to primitive value
console.log(dict);       // [Object: null prototype] {} — Node shows the difference
```

This is the right structure for a pure lookup table with untrusted keys: no inherited keys to accidentally match, and no `__proto__` setter to exploit. `Object.groupBy` returns a null-prototype object for exactly this reason ([Part 6](./javascript-arrays-iteration.md)).

#### Customising `instanceof`

`instanceof` walks the prototype chain looking for `Ctor.prototype`. `Symbol.hasInstance` overrides that entirely.

```js
class Even {
  static [Symbol.hasInstance](n) {
    return typeof n === "number" && n % 2 === 0;
  }
}
4 instanceof Even;       // true
5 instanceof Even;       // false
```

Useful for branded checks and duck-typing guards; deeply confusing if overused. The realm caveat from [Part 1](./javascript-foundations.md#part-1-advanced) still applies: `instanceof` compares against *your* realm's constructor, so cross-realm objects fail.

#### Property lookup performance and "shapes"

Engines optimise objects by grouping those with identical property layouts into a **hidden class** (V8's term; also "shape" or "map"). Objects sharing a shape get fast inline-cached property access. Adding properties in a different order, or later, creates a different shape and can push a call site into the slow path.

```js
// ❌ Two shapes for the same logical type — property order differs.
const a = { x: 1, y: 2 };
const b = { y: 2, x: 1 };

// ❌ Shape transitions after construction.
const p = {};
p.x = 1;      // shape: {}  → {x}
p.y = 2;      // shape: {x} → {x,y}

// ✅ One shape, declared at once, same order every time.
const point = { x: 1, y: 2 };
```

Treat this as a tiebreaker, not a design driver. The practical takeaways — initialise all fields in the constructor, keep property order consistent, avoid `delete` on hot objects (prefer setting to `null`) — are also just good style. Anything more specific is engine-version-dependent; see the [V8 blog](https://v8.dev/blog) rather than trusting any advice frozen in a document.

### Part 5 quick reference

<a id="part-5-cheatsheet"></a>

| Task | Use |
|---|---|
| Own-property existence | `Object.hasOwn(obj, key)` |
| Own enumerable keys | `Object.keys(obj)` |
| Key/value pairs | `Object.entries(obj)` |
| Build from pairs | `Object.fromEntries(pairs)` |
| Shallow copy / merge | `{ ...a, ...b }` |
| Shallow copy triggering setters | `Object.assign(target, src)` |
| Deep copy | `structuredClone(obj)` |
| Read prototype | `Object.getPrototypeOf(obj)` |
| Create with a prototype | `Object.create(proto)` |
| Untrusted-key dictionary | `new Map()` or `Object.create(null)` |
| Shallow immutability | `Object.freeze(obj)` |
| Non-enumerable internals | `Object.defineProperty` |
| Inspect flags | `Object.getOwnPropertyDescriptor` |

[↑ Back to top](#table-of-contents)

---

## 2. Part 7 — Classes

<a id="part-7"></a>

### Beginner — syntax

<a id="part-7-beginner"></a>

```js
class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }

  greet() {
    return `Hi, ${this.name}`;
  }

  static fromJSON(json) {
    const { name, email } = JSON.parse(json);
    return new User(name, email);
  }
}

const sam = new User("Sam", "s@x.com");
sam.greet();                            // "Hi, Sam"
User.fromJSON('{"name":"Alex"}');       // a new User
```

- **`constructor`** runs on `new`. One per class.
- **Methods** go on `User.prototype` — shared by every instance, defined once.
- **`static` members** live on the class itself, not on instances. Use them for factories and constants.

Two rules that differ from ordinary functions: class declarations are **not hoisted** (they sit in the TDZ like `let`), and class bodies are **always strict mode**.

```js
new User();          // ❌ ReferenceError: Cannot access 'User' before initialization
class User {}
```

### Working Knowledge — fields, private members, inheritance

<a id="part-7-working"></a>

#### Class fields

```js
class Counter {
  count = 0;                    // public instance field
  #secret = "hidden";           // private instance field
  static instances = 0;         // public static field
  static #registry = new Map(); // private static field

  constructor() {
    Counter.instances++;
  }

  increment() { return ++this.count; }

  reveal() { return this.#secret; }        // only readable from inside the class

  static has(id) { return Counter.#registry.has(id); }
}

const c = new Counter();
c.count;         // 0
c.reveal();      // "hidden"
c.#secret;       // ❌ SyntaxError: Private field '#secret' must be declared in an enclosing class
```

Private fields (`#`) are [Baseline widely available](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_properties) and are **genuinely private** — not a convention. They are invisible to `Object.keys`, `JSON.stringify`, spread, `Object.getOwnPropertyNames`, and `Proxy` traps.

```js
// ❌ The old convention. Underscore is a hint, not a barrier.
class User { constructor() { this._token = "abc"; } }
JSON.stringify(new User());     // '{"_token":"abc"}'  ← leaked

// ✅ A real private field.
class User { #token = "abc"; }
JSON.stringify(new User());     // '{}'
```

That `JSON.stringify` difference is a concrete security win, not just tidiness: underscore-prefixed secrets end up in logs and API responses constantly.

You can also brand-check with `#field in obj` ([ES2022](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/in#using_the_in_operator_to_check_for_private_fields)), which is realm-safe where `instanceof` is not:

```js
class Point {
  #x;
  static isPoint(o) { return #x in o; }    // true only for real Point instances
}
```

#### Field initialisation order

This order causes real bugs, so it is worth knowing exactly:

```text
new Sub(args)
  1. Sub's constructor begins
  2. super(...) is called
       a. Base fields initialise (in declaration order)
       b. Base constructor body runs
  3. Sub's fields initialise (in declaration order)   ← AFTER super() returns
  4. Sub's constructor body runs
```

```js
// ❌ The base constructor calls an overridden method before the subclass field exists.
class Base {
  constructor() { this.setup(); }
  setup() { }
}
class Sub extends Base {
  items = [];                       // initialised AFTER super() returns
  setup() { this.items.push("x"); } // TypeError: Cannot read properties of undefined
}
new Sub();
```

```js
// ✅ Do not call overridable methods from a base constructor. Initialise explicitly.
class Base {
  constructor() { }
  init() { this.setup(); }
  setup() { }
}
class Sub extends Base {
  items = [];
  setup() { this.items.push("x"); }
}
const s = new Sub();
s.init();     // safe — fields exist by now
```

> **Try It.** Run the ❌ version. Expected: `TypeError: Cannot read properties of undefined (reading 'push')`. Then move `items = []` into the `Sub` constructor *after* `super()` — still broken, because `super()` runs first either way. The only fix is not calling overridable methods from a constructor.

#### Inheritance

```js
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);                    // MUST come before any use of `this`
    this.breed = breed;
  }
  speak() {
    return `${super.speak()} — specifically, a bark`;
  }
}

new Dog("Rex", "Lab").speak();      // "Rex makes a sound — specifically, a bark"
```

```js
// ❌ Using `this` before super().
class Dog extends Animal {
  constructor(name) {
    this.legs = 4;      // ReferenceError: Must call super constructor…
    super(name);
  }
}
```

In a derived class, `this` does not exist until `super()` has run — `super()` is what creates the instance. That is why the rule is absolute rather than stylistic.

Prefer composition to deep hierarchies. Three-plus levels of `extends` is where classes stop clarifying and start obscuring; a class that holds its collaborators as fields is easier to test and to change.

### Advanced — what `class` desugars to, and `super`'s real mechanism

<a id="part-7-advanced"></a>

`class` is (mostly) syntax over the prototype machinery from Part 5.

```js
// This class…
class User {
  constructor(name) { this.name = name; }
  greet() { return `Hi, ${this.name}`; }
  static create(n) { return new User(n); }
}

// …is approximately this.
function User(name) { this.name = name; }
User.prototype.greet = function () { return `Hi, ${this.name}`; };
User.create = function (n) { return new User(n); };
```

"Approximately", because `class` adds things the function form cannot express:

| `class` gives you | The function form |
|---|---|
| Throws without `new` | Silently runs with `this === undefined` |
| Always strict mode | Inherits the surrounding mode |
| Methods are non-enumerable | Assigned methods are enumerable |
| TDZ, not hoisted | Fully hoisted |
| `super` | No equivalent |
| `#private` fields | Only closures or `WeakMap` |
| `new.target` in constructors | Available, but clumsier |

```js
class C { m() {} }
Object.keys(C.prototype);                       // []  — methods are non-enumerable
Object.getOwnPropertyNames(C.prototype);        // ["constructor", "m"]

function F() {}
F.prototype.m = function () {};
Object.keys(F.prototype);                       // ["m"] — enumerable, shows up in for…in
```

Non-enumerable methods are why `for…in` over a class instance yields only data, and why spreading an instance gives you fields without methods.

#### The full instance/prototype picture

```text
class Dog extends Animal { … }
const rex = new Dog("Rex", "Lab");

  INSTANCES                PROTOTYPES                    CONSTRUCTORS
  ┌──────────────┐         ┌──────────────────┐          ┌──────────┐
  │ rex          │────────▶│ Dog.prototype    │          │ Dog      │
  │  name: "Rex" │         │  constructor ────┼─────────▶│          │
  │  breed:"Lab" │         │  speak()         │          │  (static)│
  └──────────────┘         └────────┬─────────┘          └────┬─────┘
                                    │                          │ Dog.__proto__
                                    ▼                          ▼   = Animal
                           ┌──────────────────┐          ┌──────────┐
                           │ Animal.prototype │◀─────────│ Animal   │
                           │  constructor     │          └────┬─────┘
                           │  speak()         │               ▼
                           └────────┬─────────┘         Function.prototype
                                    ▼
                           ┌──────────────────┐
                           │ Object.prototype │
                           └────────┬─────────┘
                                    ▼
                                  null
```

`extends` wires up *two* chains: the instance chain (`Dog.prototype` → `Animal.prototype`) and the static chain (`Dog` → `Animal`). The second is why static members are inherited:

```js
class A { static hello() { return "hi"; } }
class B extends A {}
B.hello();      // "hi" — found via B.__proto__ === A
```

#### How `super` actually resolves

`super.method()` does **not** mean "look up the prototype chain of `this`". It uses the method's `[[HomeObject]]` — the object the method was defined on, captured at definition time — and looks at *that* object's prototype. This is why extracting a method breaks `super`:

```js
class A { greet() { return "A"; } }
class B extends A { greet() { return `B→${super.greet()}`; } }

const detached = B.prototype.greet;
detached.call(new B());     // "B→A" — works: [[HomeObject]] travelled with the function

// But an object literal method copied elsewhere loses nothing either,
// because [[HomeObject]] is fixed at definition:
const obj = { __proto__: { hi() { return "proto"; } }, hi() { return super.hi(); } };
obj.hi();                   // "proto"

// ❌ An arrow or a plain function assigned later has no [[HomeObject]] — no super.
const broken = { hi: function () { return super.hi(); } };   // SyntaxError
```

`super` is only available in methods defined with method shorthand or class syntax — never in a function expression assigned to a property.

#### `new.target`

```js
class Abstract {
  constructor() {
    if (new.target === Abstract) {
      throw new TypeError("Abstract cannot be instantiated directly");
    }
  }
}
class Concrete extends Abstract {}

new Concrete();     // ✅
new Abstract();     // ❌ TypeError
```

`new.target` is the constructor that was actually invoked with `new`, or `undefined` for a plain call. It is the standard way to express "abstract base class" without a language-level `abstract` keyword.

#### Extending built-ins

```js
class ValidationError extends Error {
  constructor(message, field, options) {
    super(message, options);           // options carries { cause }
    this.name = "ValidationError";
    this.field = field;
  }
}

class Stack extends Array {
  peek() { return this[this.length - 1]; }
}
const s = new Stack();
s.push(1, 2);
s.peek();                              // 2
s.map(x => x).constructor;             // Stack — derived methods return the subclass
```

Setting `this.name` explicitly in an `Error` subclass matters: without it, `err.name` is inherited as `"Error"` and your stack traces and logs say `Error:` instead of `ValidationError:`. More on error subclassing in [Part 8](./javascript-errors-async-modules.md).

That `map` returning a `Stack` is `Symbol.species` at work — and it is the reason `Array` subclassing is often more trouble than it is worth. `Symbol.species` lets a class say "derived operations should produce *this* instead":

```js
class PlainStack extends Array {
  static get [Symbol.species]() { return Array; }
}
new PlainStack(1, 2).map(x => x).constructor;   // Array
```

### Mastery — mixins, accessors on classes, and when not to use a class

<a id="part-7-mastery"></a>

#### Mixins

JavaScript has single inheritance. Mixins — functions from a base class to a subclass — give you composition of behaviour without multiple inheritance.

```js
const Serializable = (Base) => class extends Base {
  toJSON() { return { ...this, __type: this.constructor.name }; }
};

const Timestamped = (Base) => class extends Base {
  createdAt = new Date();
  age() { return Date.now() - this.createdAt.getTime(); }
};

class Record {}
class User extends Serializable(Timestamped(Record)) {
  constructor(name) { super(); this.name = name; }
}

const u = new User("Sam");
JSON.stringify(u);       // includes name, createdAt, __type: "User"
u.age();                 // milliseconds since construction
```

Each mixin inserts a real class into the prototype chain, so `super` works normally through them and `instanceof` behaves. The cost is a deeper chain and anonymous classes in stack traces.

#### Static blocks

```js
class Config {
  static #defaults;

  static {                                  // runs once, when the class is defined
    Config.#defaults = loadDefaults();
    Object.freeze(Config.#defaults);
  }

  static get(key) { return Config.#defaults[key]; }
}
```

Static initialisation blocks ([Baseline widely available](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Static_initialization_blocks), ES2022) run once at class-definition time, in order with static field initialisers, and can access private static members. They replace the old "define the class, then patch its statics afterwards" pattern.

#### Getters and setters on classes

```js
class Temperature {
  #celsius = 0;

  get celsius() { return this.#celsius; }
  set celsius(v) {
    if (v < -273.15) throw new RangeError("below absolute zero");
    this.#celsius = v;
  }

  get fahrenheit() { return this.#celsius * 9 / 5 + 32; }
  set fahrenheit(v) { this.celsius = (v - 32) * 5 / 9; }
}

const t = new Temperature();
t.fahrenheit = 212;
t.celsius;          // 100
```

A computed getter like `fahrenheit` is the strongest case for accessors: it presents a derived value as a property so callers never have to keep two fields in sync.

#### When not to use a class

Classes earn their place when you have **state plus behaviour plus multiple instances**, or genuine subtype polymorphism. They cost you when you do not:

```js
// ❌ A class with no state — this is just a namespace with extra ceremony.
class MathUtils {
  static add(a, b) { return a + b; }
  static sub(a, b) { return a - b; }
}

// ✅ A module with exported functions. Tree-shakeable, testable, no `this`.
export const add = (a, b) => a + b;
export const sub = (a, b) => a - b;
```

```js
// ❌ A class used once, as a config bag.
class Options { constructor(o) { Object.assign(this, o); } }

// ✅ A plain object.
const options = { retries: 3, timeout: 5000 };
```

The static-only class in particular is a habit imported from languages where free functions do not exist. In JavaScript, a module *is* the namespace.

### Part 7 quick reference

<a id="part-7-cheatsheet"></a>

| Feature | Syntax | Notes |
|---|---|---|
| Constructor | `constructor() {}` | One per class; runs on `new` |
| Instance method | `method() {}` | On the prototype, non-enumerable |
| Public field | `x = 1;` | Per instance, set before the constructor body |
| Private field | `#x = 1;` | Truly private; invisible to JSON/spread/Proxy |
| Static member | `static m() {}` | On the class; inherited via `extends` |
| Static field | `static x = 1;` | |
| Static block | `static { … }` | Runs once at definition |
| Getter / setter | `get x() {}` / `set x(v) {}` | Accessor, not data |
| Inheritance | `class B extends A` | Wires instance *and* static chains |
| Parent call | `super(...)` / `super.m()` | Must precede `this` in a derived constructor |
| Brand check | `#x in obj` | Realm-safe, unlike `instanceof` |
| Abstract guard | `new.target === Base` | |
| Derived-result type | `static get [Symbol.species]()` | |

**Construction order:** base fields → base constructor body → derived fields → derived constructor body. Never call an overridable method from a constructor.

| Choose | When |
|---|---|
| `class` | State + behaviour + multiple instances, or real subtyping |
| Module of functions | Stateless utilities |
| Plain object | A one-off bag of data |
| Closure factory | One or two instances, simple private state |

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 6 — Arrays & Iteration](./javascript-arrays-iteration.md) · **Up:** [Mastery Guide overview](./javascript-mastery-guide.md)

*Written against ECMAScript 2026. Baseline data verified September 6, 2026.*
