# JavaScript — Metaprogramming & Modern JS (Parts 13–14)

Symbols, `Proxy`/`Reflect`, and the weak collections — the machinery that lets code reason about code. Then a survey of what actually landed in ES2023 through ES2026, with Baseline dates so you can tell what is safe to ship.

> **About this document.** Part of the [JavaScript Mastery Guide](./javascript-mastery-guide.md) set — see that document's "About This Document" section for the framing (curiosity-driven, hands-on), scope, tier conventions, sourcing rule and update instructions. Written against ECMAScript 2026; Baseline data verified September 6, 2026.

**Prerequisite:** [Parts 5 & 7 — Objects & Classes](./javascript-objects-classes.md). Everything here operates on the property-access machinery described there.

---

## Table of Contents

1. [Part 13 — Metaprogramming](#1-part-13--metaprogramming)
   - [Beginner](#part-13-beginner) · [Working Knowledge](#part-13-working) · [Advanced](#part-13-advanced) · [Mastery](#part-13-mastery) · [Cheat sheet](#part-13-cheatsheet)
2. [Part 14 — Modern JS Grab Bag](#2-part-14--modern-js-grab-bag)
   - [ES2023](#es2023) · [ES2024](#es2024) · [ES2025](#es2025) · [ES2026](#es2026) · [ES2027 track](#es2027) · [Cheat sheet](#part-14-cheatsheet)

---

## 1. Part 13 — Metaprogramming

<a id="part-13"></a>

> **Tier note.** Part 13's Beginner tier is deliberately thin. Nothing here is needed to write ordinary JavaScript — the tier exists only to establish what a symbol is before the interesting material starts.

### Beginner — symbols as unique keys

<a id="part-13-beginner"></a>

A `Symbol` is a primitive whose only property is that it is unique. Its purpose is to be a property key that cannot collide with anything.

```js
const id = Symbol("id");            // the string is a description, for debugging only
Symbol("id") === Symbol("id");      // false — always a new one
id.description;                     // "id"

const user = { name: "Sam", [id]: 123 };
user[id];                           // 123
Object.keys(user);                  // ["name"]      — symbols are skipped
JSON.stringify(user);               // '{"name":"Sam"}'
Object.getOwnPropertySymbols(user); // [Symbol(id)]  — the explicit way to find them
```

That invisibility is the feature. Symbol-keyed data cannot be accidentally serialised, iterated, or overwritten by a library that does not know about it.

The **global symbol registry** is the exception — for when two independent pieces of code need the *same* symbol:

```js
Symbol.for("app.id") === Symbol.for("app.id");   // true — looked up by key
Symbol.keyFor(Symbol.for("app.id"));             // "app.id"
Symbol.keyFor(Symbol("app.id"));                 // undefined — not in the registry
```

### Working Knowledge — well-known symbols and weak collections

<a id="part-13-working"></a>

#### Well-known symbols

These are the language's extension points: implement one on your object and you change how a built-in operation treats it.

| Symbol | Controls |
|---|---|
| `Symbol.iterator` | `for…of`, spread, destructuring ([Part 6](./javascript-arrays-iteration.md)) |
| `Symbol.asyncIterator` | `for await…of` |
| `Symbol.toPrimitive` | Coercion to number/string ([Part 3](./javascript-foundations.md#part-3-mastery)) |
| `Symbol.toStringTag` | `Object.prototype.toString.call(x)` |
| `Symbol.hasInstance` | `instanceof` |
| `Symbol.species` | The constructor derived methods use |
| `Symbol.dispose` / `Symbol.asyncDispose` | `using` declarations (ES2027 track) |

```js
class Collection {
  #items = [];
  constructor(...items) { this.#items = items; }

  get [Symbol.toStringTag]() { return "Collection"; }
  *[Symbol.iterator]() { yield* this.#items; }
  static [Symbol.hasInstance](o) { return #items in o; }
  [Symbol.toPrimitive](hint) {
    return hint === "number" ? this.#items.length : `Collection(${this.#items.length})`;
  }
}

const c = new Collection(1, 2, 3);
[...c];                                          // [1, 2, 3]
+c;                                              // 3
`${c}`;                                          // "Collection(3)"
Object.prototype.toString.call(c);               // "[object Collection]"
c instanceof Collection;                         // true, via the brand check
```

`Symbol.toStringTag` is the small one worth adopting habitually — it makes your objects identifiable to the realm-safe type check from [Part 1](./javascript-foundations.md#part-1-advanced) and improves how they print in debuggers.

#### `WeakMap`, `WeakSet`, `WeakRef`

A `Map` holds its keys strongly: an object used as a key can never be garbage collected while the map lives. A `WeakMap` holds keys **weakly** — when nothing else references the key, both key and value are collected.

```js
const strong = new Map();
const weak = new WeakMap();

let obj = { big: "data" };
strong.set(obj, "metadata");
weak.set(obj, "metadata");

obj = null;
// strong still holds the object and its value forever — a leak.
// weak's entry becomes collectable.
```

| | `Map` / `Set` | `WeakMap` / `WeakSet` |
|---|---|---|
| Key types | Anything | Objects and non-registered symbols only |
| Holds keys | Strongly | Weakly |
| Iterable | ✅ | ❌ |
| `.size` | ✅ | ❌ |
| Use for | General storage | Metadata attached to objects you do not own |

`WeakMap` is not iterable and has no size *by design*: exposing either would let you observe garbage collection timing, which would make GC behaviour a program-visible detail.

```js
// The canonical use: private data for objects you did not define.
const privateData = new WeakMap();

class User {
  constructor(name, ssn) {
    this.name = name;
    privateData.set(this, { ssn });          // collected with the instance
  }
  get ssn() { return privateData.get(this).ssn; }
}
```

With `#private` fields ([Part 7](./javascript-objects-classes.md#part-7-working)) this specific use is mostly historical — but `WeakMap` remains correct when you need to attach data to objects from *another* library, where you cannot add a field.

```js
// Caching derived values keyed by object identity, without leaking.
const layoutCache = new WeakMap();
function getLayout(node) {
  let layout = layoutCache.get(node);
  if (!layout) {
    layout = expensiveMeasure(node);
    layoutCache.set(node, layout);
  }
  return layout;
}
// When the node is removed and dereferenced, its cache entry goes too.
```

`WeakRef` and `FinalizationRegistry` (ES2021) go further, letting you hold a collectable reference and register a cleanup callback:

```js
const ref = new WeakRef(someObject);
const obj = ref.deref();          // the object, or undefined if collected

const registry = new FinalizationRegistry((heldValue) => {
  console.log("collected:", heldValue);
});
registry.register(someObject, "someObject's label");
```

[MDN warns explicitly](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef) that correct use of these is hard and that they should be avoided where possible: collection timing is unspecified, callbacks may never run, and code that depends on either is not portable. `WeakMap`/`WeakSet` are the ones to reach for; `WeakRef` is a last resort.

> **Try It.** With `node --expose-gc`:
> ```js
> const wm = new WeakMap();
> let key = {};
> wm.set(key, new Array(1e6).fill("x"));
> global.gc();
> console.log(process.memoryUsage().heapUsed / 1e6);
> key = null;
> global.gc();
> console.log(process.memoryUsage().heapUsed / 1e6);
> ```
> Expected: a drop of roughly 8 MB on the second line. Swap `WeakMap` for `Map` and the memory stays held.

### Advanced — `Proxy` and `Reflect`

<a id="part-13-advanced"></a>

A `Proxy` wraps an object and intercepts the fundamental operations performed on it. `Reflect` provides the default implementation of each of those operations, so a trap can do its work and then delegate.

```js
const target = { name: "Sam" };

const proxy = new Proxy(target, {
  get(obj, prop, receiver) {
    console.log(`read ${String(prop)}`);
    return Reflect.get(obj, prop, receiver);
  },
  set(obj, prop, value, receiver) {
    if (prop === "age" && typeof value !== "number") {
      throw new TypeError("age must be a number");
    }
    return Reflect.set(obj, prop, value, receiver);
  },
});

proxy.name;         // logs "read name" → "Sam"
proxy.age = 30;     // ok
proxy.age = "x";    // TypeError
```

```text
        code                    Proxy                      target
  ─────────────────      ┌────────────────────┐      ┌──────────────┐
   proxy.name    ───────▶│ get trap           │─────▶│ { name: … }  │
   proxy.x = 1   ───────▶│ set trap           │─────▶│              │
   "x" in proxy  ───────▶│ has trap           │─────▶│              │
   delete proxy.x───────▶│ deleteProperty     │─────▶│              │
   Object.keys   ───────▶│ ownKeys            │─────▶│              │
   proxy()       ───────▶│ apply              │      └──────────────┘
   new proxy()   ───────▶│ construct          │
                         └────────────────────┘
```

The thirteen traps: `get`, `set`, `has`, `deleteProperty`, `ownKeys`, `getOwnPropertyDescriptor`, `defineProperty`, `getPrototypeOf`, `setPrototypeOf`, `isExtensible`, `preventExtensions`, `apply`, `construct`.

**Always delegate through `Reflect`** rather than reimplementing the default:

```js
// ❌ obj[prop] loses the receiver, so getters on the prototype see the wrong `this`.
get(obj, prop) { return obj[prop]; }

// ✅ Reflect.get forwards the receiver.
get(obj, prop, receiver) { return Reflect.get(obj, prop, receiver); }
```

`Reflect` also gives you the non-throwing, function-shaped versions of operations that are otherwise operators or throw:

```js
Reflect.has(obj, "x");                  // the `in` operator, as a function
Reflect.ownKeys(obj);                   // string AND symbol keys
Reflect.defineProperty(obj, "x", desc); // returns false instead of throwing
Reflect.construct(Ctor, args);          // `new Ctor(...args)`, as a function
Reflect.apply(fn, thisArg, args);
```

#### Practical proxies

```js
// A safe default for missing keys.
const withDefault = (target, fallback) =>
  new Proxy(target, {
    get: (o, p, r) => (Reflect.has(o, p) ? Reflect.get(o, p, r) : fallback),
  });
withDefault({}, 0).anything;                 // 0

// Negative array indices.
const negatable = (arr) =>
  new Proxy(arr, {
    get(o, p, r) {
      const i = Number(p);
      return Number.isInteger(i) && i < 0
        ? Reflect.get(o, String(o.length + i), r)
        : Reflect.get(o, p, r);
    },
  });
negatable([1, 2, 3])[-1];                    // 3   (though `.at(-1)` exists now)

// Loud typo detection during development.
const strict = (obj) =>
  new Proxy(obj, {
    get(o, p, r) {
      if (!(p in o) && typeof p === "string") {
        throw new ReferenceError(`Unknown property: ${p}`);
      }
      return Reflect.get(o, p, r);
    },
  });

// Deep observability — the core of reactivity systems.
function observable(obj, onChange, path = []) {
  return new Proxy(obj, {
    get(o, p, r) {
      const v = Reflect.get(o, p, r);
      return v && typeof v === "object"
        ? observable(v, onChange, [...path, p])     // wrap nested objects lazily
        : v;
    },
    set(o, p, v, r) {
      const ok = Reflect.set(o, p, v, r);
      onChange([...path, p].join("."), v);
      return ok;
    },
  });
}

const state = observable({ user: { name: "Sam" } }, (p, v) => console.log(p, "=", v));
state.user.name = "Alex";        // logs "user.name = Alex"
```

That last one is, in outline, how Vue 3's reactivity works — a proxy that tracks reads to build a dependency graph and intercepts writes to trigger updates. Seeing it in twenty lines demystifies a lot of framework behaviour.

#### The invariants

A proxy cannot lie about certain things. If the target has a non-configurable, non-writable property, the `get` trap **must** return its actual value; if the target is non-extensible, `ownKeys` must report exactly its keys. Violating an invariant throws a `TypeError` at the point of the operation:

```js
const target = {};
Object.defineProperty(target, "fixed", { value: 1, configurable: false, writable: false });

const p = new Proxy(target, { get: () => "lie" });
p.fixed;    // ❌ TypeError: 'get' on proxy: property 'fixed' is a read-only and
            //    non-configurable data property on the proxy target but the proxy
            //    did not return its actual value
```

These invariants are what keep `Object.freeze` and non-configurable properties meaningful in the presence of proxies.

#### Costs and limits

- Every intercepted operation goes through a JavaScript function call. Proxies are **substantially slower** than direct property access and are not a good fit for hot paths.
- Proxies cannot intercept `#private` field access — private fields are keyed to the target's own brand, so a proxy wrapping a class instance will throw when a method reaches for `this.#x`. Wrap with care, or expose an ordinary accessor.
- `Object.is(proxy, target)` is `false`. A proxy is a distinct object identity, which matters for `Map` keys, `Set` membership, and `===` comparisons.

> **Try It.**
> ```js
> class C { #x = 1; getX() { return this.#x; } }
> const p = new Proxy(new C(), {});
> console.log(p.getX());
> ```
> Expected: a `TypeError` about reading a private member from an object whose class did not declare it. This is the single most common proxy surprise, and the reason ORM and reactivity libraries special-case class instances.

### Mastery — a small dependency-tracking system

<a id="part-13-mastery"></a>

Putting symbols, weak collections, and proxies together produces something recognisably framework-shaped in a page of code:

```js
let activeEffect = null;
const targetMap = new WeakMap();          // target → (key → Set<effect>)

function track(target, key) {
  if (!activeEffect) return;
  let depsMap = targetMap.get(target);
  if (!depsMap) targetMap.set(target, (depsMap = new Map()));
  let deps = depsMap.get(key);
  if (!deps) depsMap.set(key, (deps = new Set()));
  deps.add(activeEffect);
}

function trigger(target, key) {
  const depsMap = targetMap.get(target);
  if (!depsMap) return;
  for (const effect of depsMap.get(key) ?? []) effect();
}

function reactive(obj) {
  return new Proxy(obj, {
    get(o, k, r) { track(o, k); return Reflect.get(o, k, r); },
    set(o, k, v, r) {
      const had = Reflect.get(o, k, r) !== v;
      const ok = Reflect.set(o, k, v, r);
      if (had) trigger(o, k);
      return ok;
    },
  });
}

function effect(fn) {
  activeEffect = fn;
  fn();                                   // run once to collect dependencies
  activeEffect = null;
}

// Usage
const state = reactive({ count: 0, name: "Sam" });
effect(() => console.log("count is", state.count));   // logs "count is 0"
state.count = 1;                                       // logs "count is 1"
state.name = "Alex";                                   // logs nothing — not a dependency
```

Three design decisions worth noticing, because they are the same ones real frameworks make:

1. **`WeakMap` for `targetMap`** — a reactive object that goes out of scope takes its subscriptions with it, with no manual teardown.
2. **Tracking on `get`, triggering on `set`** — dependencies are discovered by *running* the effect, not declared. That is why an effect that reads a property conditionally can have different dependencies on different runs.
3. **The `had` check before triggering** — writing the same value must not re-run effects, or a self-referential effect loops forever.

### Part 13 quick reference

<a id="part-13-cheatsheet"></a>

| Tool | Use for |
|---|---|
| `Symbol()` | A unique, collision-proof property key |
| `Symbol.for(k)` | A shared symbol across independent code |
| `Symbol.iterator` | Make an object work with `for…of` and spread |
| `Symbol.toStringTag` | Make an object identify itself to type checks and debuggers |
| `Symbol.hasInstance` | Custom `instanceof` |
| `Symbol.toPrimitive` | Custom coercion |
| `WeakMap` | Metadata for objects you do not own, without leaking |
| `WeakSet` | "Have I seen this object?" without pinning it |
| `WeakRef` / `FinalizationRegistry` | Last resort; timing is unspecified |
| `Proxy` | Intercept property access, calls, construction |
| `Reflect` | The default behaviour, as callable functions |

| Trap | Fires on |
|---|---|
| `get` / `set` | Property read / write |
| `has` | `in` |
| `deleteProperty` | `delete` |
| `ownKeys` | `Object.keys`, spread, `for…in` |
| `apply` / `construct` | Calling / `new` |
| `getPrototypeOf` / `setPrototypeOf` | Prototype access |

**Proxy caveats:** slow on hot paths; cannot intercept `#private` fields; distinct object identity from the target; cannot violate the target's invariants.

[↑ Back to top](#table-of-contents)

---

## 2. Part 14 — Modern JS Grab Bag

<a id="part-14"></a>

> **Tier note.** Part 14 has no tiers. It is organised by specification edition instead, since that is how you will need to look things up. Each entry carries its [Baseline](https://developer.mozilla.org/en-US/docs/Glossary/Baseline/Compatibility) status where it matters.

**Read the ES-year as a publication label, not an availability date.** Engines ship features at Stage 3–4, and the annual spec catches up afterwards. `Array.fromAsync` is formally ES2026 yet has been Baseline widely available since January 2024. Decide what you can ship from **Baseline status**, never from the edition number.

### ES2023

<a id="es2023"></a>

```js
// Change array by copy — non-mutating counterparts. Baseline widely available.
const arr = [3, 1, 2];
arr.toSorted((a, b) => a - b);   // [1,2,3]  — arr unchanged
arr.toReversed();                // [2,1,3]
arr.toSpliced(1, 1);             // [3,2]
arr.with(0, 99);                 // [99,1,2]

// Search from the end.
[1, 2, 3, 4].findLast(n => n < 4);        // 3
[1, 2, 3, 4].findLastIndex(n => n < 4);   // 2

// Hashbang grammar — #!/usr/bin/env node is now standard syntax at the top of a file.
```

The `toSorted` family is the most consequential of these: it removes the single most common accidental-mutation bug in JavaScript ([Part 6](./javascript-arrays-iteration.md#part-6-working)).

### ES2024

<a id="es2024"></a>

```js
// Object.groupBy / Map.groupBy — Baseline 2024, newly available since March 2024.
Object.groupBy(people, p => p.dept);      // null-prototype object
Map.groupBy(people, p => p.team);         // any key type

// Promise.withResolvers — Baseline 2024, newly available since March 2024.
const { promise, resolve, reject } = Promise.withResolvers();

// String.prototype.isWellFormed / toWellFormed — lone surrogates.
"\uD800".isWellFormed();                  // false
"\uD800".toWellFormed();                  // "�" — replacement character

// RegExp v flag (unicodeSets) — set operations in character classes.
/[\p{Letter}&&\p{ASCII}]/v.test("é");     // false

// Also: resizable ArrayBuffer, Atomics.waitAsync.
```

`toWellFormed` matters more than it looks: a lone surrogate in a string thrown at `JSON.stringify`, `encodeURIComponent`, or a `TextEncoder` used to produce mangled output or throw. Sanitising user input through it is now a one-liner.

### ES2025

<a id="es2025"></a>

```js
// Iterator helpers — Baseline 2025, newly available since March 2025. Lazy.
function* naturals() { let n = 1; while (true) yield n++; }
naturals().filter(n => n % 3 === 0).map(n => n ** 2).take(5).toArray();
// [9, 36, 81, 144, 225]

Iterator.from([1, 2, 3]).map(x => x * 2).toArray();    // [2, 4, 6]

// New Set methods — Baseline 2024, newly available since June 2024.
const a = new Set([1, 2, 3]), b = new Set([2, 3, 4]);
a.union(b);                   // Set {1,2,3,4}
a.intersection(b);            // Set {2,3}
a.difference(b);              // Set {1}
a.symmetricDifference(b);     // Set {1,4}
a.isSubsetOf(b);              // false
a.isSupersetOf(b);            // false
a.isDisjointFrom(new Set([9])); // true

// RegExp.escape — Baseline 2025, newly available since May 2025.
new RegExp(RegExp.escape(userInput), "g");

// Promise.try — Baseline 2025, newly available since January 2025.
Promise.try(maybeThrowsSynchronously).catch(handle);

// Import attributes and JSON modules — note `with`, NOT the obsolete `assert`.
import config from "./config.json" with { type: "json" };

// RegExp modifiers — scoped flags.
/^(?i:hello) world$/.test("HELLO world");   // true

// Also: duplicate named capture groups in alternations, Float16Array,
// Math.f16round, ArrayBuffer.prototype.transfer.
```

The `Set` methods deserve adoption on sight — before them, every "intersect two sets" was a hand-written filter, usually O(n·m).

### ES2026

<a id="es2026"></a>

The current published edition ([ES2026 spec](https://tc39.es/ecma262/2026/)).

```js
// Map upsert — Baseline 2026, newly available since February 2026.
const m = new Map([["bar", "foo"]]);
m.getOrInsert("bar", "default");                 // "foo"  — existing value kept
m.getOrInsert("baz", "default");                 // "default" — inserted
m.getOrInsertComputed(key, () => expensive());   // callback runs only on a miss

// The multimap idiom, in one line.
m.getOrInsert(key, []).push(value);

// Error.isError — LIMITED AVAILABILITY, not Baseline. Verify before shipping.
Error.isError(new TypeError());                  // true
Error.isError({ __proto__: Error.prototype });   // false — instanceof says true

// Iterator sequencing.
Iterator.concat([1, 2].values(), [3, 4].values()).toArray();   // [1,2,3,4]

// Math.sumPrecise — exact summation, no floating-point drift.
[0.1, 0.2, 0.3].reduce((a, b) => a + b, 0);      // 0.6000000000000001
Math.sumPrecise([0.1, 0.2, 0.3]);                // 0.6

// Uint8Array to/from Base64 and hex — no more atob/btoa round-trips.
new Uint8Array([72, 105]).toBase64();            // "SGk="
Uint8Array.fromBase64("SGk=");                   // Uint8Array [72, 105]
new Uint8Array([255]).toHex();                   // "ff"

// JSON.parse source text access — exact numbers without precision loss.
JSON.parse('{"big": 12345678901234567890}', (key, value, context) => {
  if (typeof value === "number") return BigInt(context.source);
  return value;
});

// Array.fromAsync — formally ES2026, but Baseline widely available since Jan 2024.
await Array.fromAsync(asyncIterable);
await Array.fromAsync([1, 2], async (n) => n * 2);   // [2, 4]
```

`getOrInsertComputed` cleans up the memoisation pattern from [Part 4](./javascript-functions-closures.md#part-4-mastery):

```js
// ❌ Double lookup, and has() is needed because a cached `undefined` looks like a miss.
if (cache.has(key)) return cache.get(key);
const v = compute(key);
cache.set(key, v);
return v;

// ✅ One call, correct for undefined values.
return cache.getOrInsertComputed(key, () => compute(key));
```

`JSON.parse` source access finally solves the "large integer IDs lose precision" problem that has forced APIs to send ids as strings for two decades.

### ES2027 track — finished, not yet published

<a id="es2027"></a>

These have reached Stage 4 and are slated for ES2027 per the [TC39 finished-proposals list](https://github.com/tc39/proposals/blob/main/finished-proposals.md). **They are widely and incorrectly described as ES2026 online** — if a blog post tells you `using` is ES2026, that is your signal to check the primary source.

```js
// Explicit Resource Management — `using` / `await using`.
// LIMITED AVAILABILITY, not Baseline.
class FileHandle {
  constructor(path) { this.path = path; }
  [Symbol.dispose]() { console.log("closed", this.path); }
}

function read() {
  using file = new FileHandle("/tmp/x");   // disposed at end of block, even on throw
  return doWork(file);
}                                          // logs "closed /tmp/x"

async function readAsync() {
  await using conn = await openConnection();   // Symbol.asyncDispose
  return conn.query();
}

// DisposableStack, for composing several resources.
using stack = new DisposableStack();
const a = stack.use(openA());
const b = stack.use(openB());
// both disposed in reverse order at block exit
```

Also on the ES2027 track: `Temporal` ([Part 12](./javascript-regex-dates-intl.md#part-12-mastery)), `Atomics.pause`, and Joint Iteration (`Iterator.zip`).

`using` is the most significant of these for everyday code — it is `try`/`finally` for resource cleanup, made declarative, and it composes correctly through early returns and exceptions. Its `SuppressedError` type handles the awkward case where the disposal itself throws while an exception is already in flight.

### Part 14 quick reference

<a id="part-14-cheatsheet"></a>

| Feature | Edition | Baseline status (September 6, 2026) |
|---|---|---|
| `toSorted`/`toReversed`/`toSpliced`/`with` | ES2023 | Widely available |
| `findLast`/`findLastIndex` | ES2023 | Widely available |
| `Array.fromAsync` | ES2026 | **Widely available** since Jan 2024 |
| `Object.groupBy` / `Map.groupBy` | ES2024 | Newly available, March 2024 |
| `Promise.withResolvers` | ES2024 | Newly available, March 2024 |
| Set methods (`union`, …) | ES2025 | Newly available, June 2024 |
| RegExp `v` flag | ES2024 | Widely available |
| `Promise.try` | ES2025 | Newly available, January 2025 |
| Iterator helpers | ES2025 | Newly available, March 2025 |
| `RegExp.escape` | ES2025 | Newly available, May 2025 |
| Import attributes (`with`) | ES2025 | Check MDN per runtime |
| `Map.getOrInsert` / `getOrInsertComputed` | ES2026 | Newly available, February 2026 |
| `Math.sumPrecise` | ES2026 | Check MDN |
| `Uint8Array` base64/hex | ES2026 | Check MDN |
| `JSON.parse` source access | ES2026 | Check MDN |
| `Error.isError` | ES2026 | **Limited availability — not Baseline** |
| `using` / `await using` | **ES2027** | **Limited availability — not Baseline** |
| `Temporal` | **ES2027** | **Limited availability — not Baseline** |

| Outdated claim you will meet online | Reality |
|---|---|
| "`using` is ES2026" | ES2027 track |
| "`Temporal` is ES2026" | ES2027 track; not Baseline |
| "`import x from './d.json' assert { type: 'json' }`" | Renamed to `with`; `assert` is being removed |
| "Even-numbered Node releases are LTS" | Changing from October 2026 to one major per year, all LTS |
| "Use `JSON.parse(JSON.stringify(x))` to deep copy" | `structuredClone(x)` |
| "Use `_private` by convention" | `#private` fields are real and Baseline |
| "Use a date library for formatting" | `Intl.DateTimeFormat` |
| "`arr.sort()` sorts numbers" | It stringifies; pass a comparator |

[↑ Back to top](#table-of-contents)

---

**Up:** [Mastery Guide overview](./javascript-mastery-guide.md) · **Self-check:** [Quick Self-Check](./javascript-mastery-guide.md#8-quick-self-check)

*Written against ECMAScript 2026. Baseline data verified September 6, 2026 — the availability table above is the fastest-ageing content in this doc set; re-verify it against MDN before relying on any "limited availability" row.*
