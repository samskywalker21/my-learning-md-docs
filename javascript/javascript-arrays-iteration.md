# JavaScript — Arrays & Iteration (Part 6)

Array methods, the mutating/copying divide, the iteration protocols, generators, and iterator helpers. This is the Part with the highest ratio of everyday usefulness to conceptual difficulty — except for the last two tiers, which are where lazy sequences live.

> **About this document.** Part of the [JavaScript Mastery Guide](./javascript-mastery-guide.md) set — see that document's "About This Document" section for the framing (curiosity-driven, hands-on), scope, tier conventions, sourcing rule and update instructions. Written against ECMAScript 2026; Baseline data verified September 6, 2026.

**Prerequisite:** [Part 4 — Functions & Closures](./javascript-functions-closures.md), since every array method here takes a callback.

---

## Table of Contents

1. [Part 6 — Arrays & Iteration](#1-part-6--arrays--iteration)
   - [Beginner — creating and accessing](#part-6-beginner)
   - [Working Knowledge — the method vocabulary, and mutation](#part-6-working)
   - [Advanced — the iteration protocols and generators](#part-6-advanced)
   - [Mastery — iterator helpers, infinite sequences, async iteration](#part-6-mastery)
   - [Cheat sheet](#part-6-cheatsheet)

---

## 1. Part 6 — Arrays & Iteration

<a id="part-6"></a>

### Beginner — creating and accessing

<a id="part-6-beginner"></a>

An array is an object with integer-like keys and a `length` that maintains itself. It is not a contiguous block of memory the way a C array is, and it is not typed.

```js
const nums = [1, 2, 3];
const mixed = [1, "two", { three: 3 }, [4]];   // heterogeneous is legal

nums[0];              // 1
nums.at(-1);          // 3    — negative indices, ES2022
nums[nums.length - 1];// 3    — the pre-ES2022 idiom
nums.length;          // 3

nums[10] = 11;
nums.length;          // 11   — length follows the highest index
nums[5];              // undefined — a "hole"
```

`Array.prototype.at()` is [Baseline widely available](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/at) and works on strings and typed arrays too. Use it whenever you want the last element.

#### Creating arrays

```js
[];                              // literal — the default
new Array(3);                    // [ <3 empty items> ]  ← holes, not undefined
new Array(1, 2);                 // [1, 2]  ← different behaviour for 2+ args. Avoid.
Array.of(3);                     // [3]     — the unambiguous single-element form
Array.from("abc");               // ["a","b","c"]
Array.from({ length: 3 }, (_, i) => i * 2);   // [0, 2, 4]
Array(3).fill(0);                // [0, 0, 0]
[...Array(3).keys()];            // [0, 1, 2]
```

The `new Array(n)` overload is a genuine wart: one number means "length", two or more mean "elements". `Array.of` exists purely to fix it.

**Holes** are worth understanding once and then avoiding forever. A hole is not `undefined` — it is an absent index, and array methods disagree about how to treat it:

```js
const holey = [1, , 3];          // length 3, index 1 is a hole
holey[1];                        // undefined
1 in holey;                      // false — the index does not exist

holey.map(x => 5);               // [5, <1 empty item>, 5]  ← map SKIPS holes
holey.forEach(x => console.log(x));  // logs 1, 3 — skips the hole
[...holey];                      // [1, undefined, 3]  ← spread FILLS holes
holey.includes(undefined);       // true
holey.indexOf(undefined);        // -1
```

```js
// ❌ Creates holes; map won't run.
const zeros = new Array(3).map(() => 0);   // [ <3 empty items> ]

// ✅ fill first, or use Array.from.
const zeros = new Array(3).fill(0);              // [0, 0, 0]
const zeros = Array.from({ length: 3 }, () => 0); // [0, 0, 0]
```

> **Try It.** Run `const a = new Array(3); console.log(a.map(() => 1), Array.from({length:3}, () => 1))`. Expected: `[ <3 empty items> ] [ 1, 1, 1 ]`. That difference has broken more "initialise an array" one-liners than any other quirk here.

### Working Knowledge — the method vocabulary, and mutation

<a id="part-6-working"></a>

#### The core transformations

```js
const nums = [1, 2, 3, 4, 5];

nums.map(n => n * 2);              // [2,4,6,8,10]      — same length, transformed
nums.filter(n => n % 2 === 0);     // [2, 4]            — subset
nums.reduce((sum, n) => sum + n, 0);      // 15         — fold to one value
nums.find(n => n > 3);             // 4                 — first match, or undefined
nums.findIndex(n => n > 3);        // 3
nums.findLast(n => n < 4);         // 3                 — ES2023
nums.findLastIndex(n => n < 4);    // 2
nums.some(n => n > 4);             // true              — "any"
nums.every(n => n > 0);            // true              — "all"
nums.includes(3);                  // true              — SameValueZero, finds NaN
nums.flat();                       // shallow un-nest
nums.flatMap(n => [n, n * 10]);    // map then flat(1)
nums.join(", ");                   // "1, 2, 3, 4, 5"
```

`reduce` deserves a note because it is the one people either overuse or avoid. Always pass an initial value — without it, `reduce` on an empty array throws:

```js
// ❌ Throws on an empty array.
[].reduce((a, b) => a + b);      // TypeError: Reduce of empty array with no initial value

// ✅ Initial value makes it total.
[].reduce((a, b) => a + b, 0);   // 0
```

```js
// ❌ reduce doing what filter+map already do, but harder to read.
const result = items.reduce((acc, x) => (x.active ? [...acc, x.name] : acc), []);

// ✅ Say what you mean.
const result = items.filter(x => x.active).map(x => x.name);
```

The spread-in-reduce version is also accidentally O(n²) — it builds a new array on every iteration. Reserve `reduce` for genuine folds: sums, grouping into an object, building a `Map`.

#### Mutating vs. copying — the divide that matters

Seven array methods mutate in place. Four of them now have copying counterparts ([change array by copy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/toSorted), ES2023, Baseline widely available).

| Mutates | Copying equivalent |
|---|---|
| `sort()` | `toSorted()` |
| `reverse()` | `toReversed()` |
| `splice()` | `toSpliced()` |
| `arr[i] = v` | `with(i, v)` |
| `push()` / `pop()` | `[...arr, v]` / `arr.slice(0, -1)` |
| `shift()` / `unshift()` | `arr.slice(1)` / `[v, ...arr]` |
| `fill()` | — |
| `copyWithin()` | — |

```js
// ❌ sort() mutates — and this one bites in shared/reactive state.
const sorted = scores.sort((a, b) => b - a);
// `scores` is now reordered too; anything else holding it sees the change.

// ✅ toSorted() returns a new array.
const sorted = scores.toSorted((a, b) => b - a);

// ✅ Pre-ES2023 equivalent.
const sorted = [...scores].sort((a, b) => b - a);
```

**`sort()` compares as strings by default.** This is the single most common array bug in JavaScript:

```js
// ❌ Default comparator stringifies.
[10, 9, 100, 1].sort();                     // [1, 10, 100, 9]

// ✅ Numeric comparator.
[10, 9, 100, 1].sort((a, b) => a - b);      // [1, 9, 10, 100]
[10, 9, 100, 1].toSorted((a, b) => a - b);  // same, non-mutating
```

For strings, `sort()`'s default compares UTF-16 code units, which puts all uppercase before all lowercase and mishandles accents. Use `localeCompare` for anything a user will read:

```js
["banana", "Apple", "cherry", "Éclair"].sort();
// ["Apple", "banana", "cherry", "Éclair"] — accents sort last, uppercase first

["banana", "Apple", "cherry", "Éclair"].toSorted((a, b) => a.localeCompare(b));
// ["Apple", "banana", "cherry", "Éclair"] — locale-correct
```

Sort stability is guaranteed since ES2019 — equal elements keep their relative order, so you can sort by secondary key then primary key and get the expected result.

> **Try It.**
> ```js
> const original = [3, 1, 2];
> const a = original.toSorted();
> const b = original.sort();
> console.log(original, a, b, b === original);
> ```
> Expected: `[1,2,3] [1,2,3] [1,2,3] true`. `toSorted` left a copy; `sort` returned the *same array* it mutated, which is why `b === original`.

#### Grouping

```js
const people = [
  { name: "Sam",  dept: "eng" },
  { name: "Alex", dept: "eng" },
  { name: "Jo",   dept: "design" },
];

Object.groupBy(people, p => p.dept);
// { eng: [{Sam}, {Alex}], design: [{Jo}] }   — a null-prototype object

Map.groupBy(people, p => p.dept);
// Map(2) { "eng" => [...], "design" => [...] }  — any key type
```

`Object.groupBy` is [Baseline 2024, newly available since March 2024](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/groupBy). It replaces the reduce-based idiom you will see everywhere in older code:

```js
// Legacy — recognise it; the built-in is clearer and faster.
const grouped = people.reduce((acc, p) => {
  (acc[p.dept] ||= []).push(p);
  return acc;
}, {});
```

Note that `Object.groupBy` returns a **null-prototype** object deliberately — so a group named `"toString"` or `"__proto__"` cannot collide with inherited properties ([Part 5](./javascript-objects-classes.md#part-5-mastery)). Use `Map.groupBy` when keys are objects or non-strings.

#### Choosing a loop

```js
const nums = [10, 20, 30];

for (const n of nums) { }               // values. Supports break/continue/await.
for (const [i, n] of nums.entries()) { }// index and value
nums.forEach((n, i) => { });            // no break, no await. Just side effects.
for (let i = 0; i < nums.length; i++) { }  // index control, fastest in hot loops
for (const k in nums) { }               // ❌ keys as STRINGS, includes inherited
```

```js
// ❌ for...in over an array: keys are strings, and prototype junk can appear.
for (const i in [10, 20]) {
  console.log(i + 1);      // "01", "11" — string concatenation
}

// ✅ for...of for values.
for (const v of [10, 20]) { }
```

The decisive practical difference: **`forEach` cannot `break` and cannot `await`.**

```js
// ❌ forEach does not wait. All three start at once; the log runs immediately.
ids.forEach(async (id) => { await save(id); });
console.log("done");      // prints before any save finishes

// ✅ for...of awaits each iteration.
for (const id of ids) { await save(id); }
console.log("done");

// ✅ Or run them concurrently, deliberately.
await Promise.all(ids.map(id => save(id)));
```

That `forEach` trap is covered in depth in [Part 9](./javascript-errors-async-modules.md#part-9-working).

### Advanced — the iteration protocols and generators

<a id="part-6-advanced"></a>

`for…of`, spread, destructuring, `Array.from`, `Promise.all`, `new Map(...)`, and `yield*` all work through one small contract.

**The iterable protocol:** an object is *iterable* if it has a `[Symbol.iterator]()` method returning an iterator.

**The iterator protocol:** an object is an *iterator* if it has a `next()` method returning `{ value, done }`.

```text
for (const x of iterable) { … }

  iterable[Symbol.iterator]()  ──▶  iterator
                                        │
                                   ┌────┴────────────────────┐
                                   │ next() → {value:1,       │
                                   │           done:false}    │ ← loop body runs
                                   │ next() → {value:2,       │
                                   │           done:false}    │ ← loop body runs
                                   │ next() → {value:undefined│
                                   │           done:true}     │ ← loop ends
                                   └─────────────────────────┘
```

Implementing it by hand, to see the shape:

```js
const range = {
  from: 1,
  to: 4,
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

[...range];                      // [1, 2, 3, 4]
for (const n of range) { }       // 1, 2, 3, 4
Math.max(...range);              // 4
const [first, second] = range;   // 1, 2
```

Note that the iterator is created fresh on each `[Symbol.iterator]()` call, so `range` can be iterated more than once. An object that *is* its own iterator (returning `this`) is single-use — a real distinction that catches people with generators, below.

#### Generators

A generator function writes an iterator without the boilerplate. `function*` plus `yield`:

```js
function* range(from, to) {
  for (let i = from; i <= to; i++) {
    yield i;
  }
}

[...range(1, 4)];                // [1, 2, 3, 4]
for (const n of range(1, 4)) { } // 1, 2, 3, 4
```

Calling a generator function runs **no code**. It returns a generator object — both an iterator and an iterable — and the body advances one `yield` at a time:

```js
function* g() {
  console.log("start");
  yield 1;
  console.log("middle");
  yield 2;
  console.log("end");
  return 3;
}

const it = g();          // nothing logged yet
it.next();               // logs "start";  { value: 1, done: false }
it.next();               // logs "middle"; { value: 2, done: false }
it.next();               // logs "end";    { value: 3, done: true }
it.next();               //                { value: undefined, done: true }
```

```text
Generator execution — the body is suspended, not restarted:

  g()          ┌──────────────────────────────────┐
  ────────────▶│ (created, nothing run)           │
  next() ─────▶│ start → yield 1 ─── SUSPENDED ───┼──▶ {1, false}
  next() ─────▶│ resumes here → yield 2 ── SUSP ──┼──▶ {2, false}
  next() ─────▶│ resumes → return 3 ── FINISHED ──┼──▶ {3, true}
               └──────────────────────────────────┘
```

**Note the `return` value:** `return 3` gives `{ value: 3, done: true }`, and `for…of` and spread **discard** any value delivered with `done: true`. `[...g()]` is `[1, 2]`, not `[1, 2, 3]`.

A generator object is its own iterator, so it is **single-use**:

```js
const it = range(1, 3);
[...it];        // [1, 2, 3]
[...it];        // []  ← exhausted

// ✅ For a re-iterable object, put the generator in [Symbol.iterator].
const reusable = {
  *[Symbol.iterator]() { yield 1; yield 2; yield 3; },
};
[...reusable];  // [1, 2, 3]
[...reusable];  // [1, 2, 3]  — fresh generator each time
```

> **Try It.** Run:
> ```js
> function* g() { yield 1; yield 2; return 99; }
> console.log([...g()]);
> const it = g();
> console.log(it.next(), it.next(), it.next());
> ```
> Expected: `[1, 2]` then `{value:1,done:false} {value:2,done:false} {value:99,done:true}`. The `99` exists — spread just throws it away.

#### Delegation and two-way communication

```js
function* inner() { yield "a"; yield "b"; }
function* outer() {
  yield 1;
  yield* inner();       // delegates: yields a, then b
  yield* [10, 20];      // yield* works on any iterable
  yield 2;
}
[...outer()];           // [1, "a", "b", 10, 20, 2]
```

`yield` is also an *expression* — the value passed to `next(v)` becomes the result of the paused `yield`:

```js
function* dialogue() {
  const name = yield "What is your name?";
  const age  = yield `Hi ${name}. How old are you?`;
  return `${name} is ${age}`;
}

const d = dialogue();
d.next().value;        // "What is your name?"   ← first next() has no destination
d.next("Sam").value;   // "Hi Sam. How old are you?"
d.next(30).value;      // "Sam is 30"
```

The first `next()` argument is always discarded because there is no suspended `yield` waiting for it. This two-way channel is the mechanism `async`/`await` is built on — an async function is essentially a generator that yields promises, driven by a runner that calls `next(resolvedValue)` ([Part 9](./javascript-errors-async-modules.md#part-9-mastery)).

You can also throw into a generator, and force cleanup:

```js
function* withCleanup() {
  try {
    yield 1;
    yield 2;
  } finally {
    console.log("cleaned up");     // runs on early exit
  }
}

for (const v of withCleanup()) {
  if (v === 1) break;              // logs "cleaned up"
}
```

`for…of` calls the iterator's `return()` method when it exits early — via `break`, `throw`, or `return` — which is what triggers the `finally`. That makes generators a legitimate resource-management tool.

### Mastery — iterator helpers, infinite sequences, async iteration

<a id="part-6-mastery"></a>

#### Iterator helpers

Array methods are eager: `arr.map(f).filter(g).slice(0, 5)` builds two full intermediate arrays before taking five items. **Iterator helpers** ([ES2025; Baseline 2025, newly available since March 2025](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator/map)) are the lazy equivalent, and they work on *any* iterator — including infinite ones.

```js
function* naturals() {
  let n = 1;
  while (true) yield n++;
}

// Lazy: computes exactly 5 values, then stops.
naturals()
  .filter(n => n % 3 === 0)
  .map(n => n * n)
  .take(5)
  .toArray();
// [9, 36, 81, 144, 225]
```

The full set: `map`, `filter`, `take`, `drop`, `flatMap`, `reduce`, `toArray`, `forEach`, `some`, `every`, `find`. They exist on `Iterator.prototype`, so every generator and every built-in iterator gets them:

```js
new Set([1, 2, 3, 4]).values().map(x => x * 2).toArray();   // [2, 4, 6, 8]
new Map([["a", 1]]).entries().map(([k, v]) => `${k}=${v}`).toArray();  // ["a=1"]
"hello"[Symbol.iterator]().drop(1).take(3).toArray();       // ["e","l","l"]
```

```js
// ❌ Eager: materialises 1,000,000 elements to use 10.
const result = Array.from({ length: 1e6 }, (_, i) => i)
  .map(n => expensive(n))
  .slice(0, 10);

// ✅ Lazy: calls expensive() exactly 10 times.
const result = Array.from({ length: 1e6 }, (_, i) => i)
  .values()
  .map(n => expensive(n))
  .take(10)
  .toArray();
```

`Iterator.from(iterable)` wraps a plain iterable so the helpers are available; `Iterator.concat(...iterables)` (ES2026, "Iterator Sequencing") chains several lazily.

> **Try It.**
> ```js
> function* nat() { let n = 1; while (true) yield n++; }
> let calls = 0;
> const sq = (n) => { calls++; return n * n; };
> console.log(nat().map(sq).take(3).toArray(), calls);
> ```
> Expected: `[ 1, 4, 9 ] 3`. Three calls for three results — on an infinite source. Try the same with `.map()` on an array and it never terminates.

#### Async iteration

`Symbol.asyncIterator` and `for await…of` are the asynchronous counterparts. `next()` returns a **promise** of `{ value, done }`.

```js
async function* paginate(url) {
  let next = url;
  while (next) {
    const res = await fetch(next);
    const page = await res.json();
    yield* page.items;              // yield each item
    next = page.nextUrl;
  }
}

for await (const item of paginate("/api/items")) {
  console.log(item);                // items stream in, one page fetched at a time
}
```

This is the cleanest expression of "paginated API as a flat sequence" the language offers: the consumer sees items, the producer handles pages, and memory holds one page at a time regardless of total size.

`for await…of` also accepts a sync iterable of promises, awaiting each in turn:

```js
for await (const result of [fetchA(), fetchB()]) { }   // sequential awaiting
```

Be deliberate about that: it awaits **in order**, so a slow first item blocks the rest. Use `Promise.all` when you want true concurrency ([Part 9](./javascript-errors-async-modules.md#part-9-working)).

Async iterator helpers are a separate, less-settled proposal — as of September 2026 the ES2025 helpers cover **sync** iterators only. Check MDN before assuming `asyncGen().map(...)` exists in your target runtimes.

#### Custom lazy pipelines

Generators compose into readable stream processing without any library:

```js
function* lines(text) {
  for (const line of text.split("\n")) yield line;
}
function* nonEmpty(iter) {
  for (const line of iter) if (line.trim()) yield line;
}
function* parsed(iter) {
  for (const line of iter) yield JSON.parse(line);
}

const records = parsed(nonEmpty(lines(rawText)));
for (const r of records) { /* one record in memory at a time */ }
```

Each stage pulls from the previous one on demand. Nothing is materialised; a 10 GB file works the same as a 10 KB one, provided the source is streamed.

#### `Symbol.iterator` on your own classes

```js
class Matrix {
  #rows;
  constructor(rows) { this.#rows = rows; }

  *[Symbol.iterator]() {              // row-major cell iteration
    for (const row of this.#rows) yield* row;
  }

  *byColumn() {                       // a second, named traversal
    const width = this.#rows[0].length;
    for (let c = 0; c < width; c++) {
      for (const row of this.#rows) yield row[c];
    }
  }
}

const m = new Matrix([[1, 2], [3, 4]]);
[...m];               // [1, 2, 3, 4]
[...m.byColumn()];    // [1, 3, 2, 4]
```

The pattern worth stealing: make the *default* traversal `[Symbol.iterator]`, and expose alternatives as generator methods that return fresh iterables. `[...m]` works, `[...m.byColumn()]` works, and both are re-iterable.

### Part 6 quick reference

<a id="part-6-cheatsheet"></a>

**Mutating vs. copying**

| Mutates | Copies |
|---|---|
| `sort` | `toSorted` |
| `reverse` | `toReversed` |
| `splice` | `toSpliced` |
| `arr[i] = v` | `with(i, v)` |
| `push`, `pop`, `shift`, `unshift`, `fill`, `copyWithin` | spread / `slice` |

Everything else — `map`, `filter`, `slice`, `concat`, `flat`, `flatMap`, `join`, `at`, `find*`, `some`, `every`, `includes`, `indexOf`, `reduce` — is non-mutating.

**Choosing a method**

| Goal | Method |
|---|---|
| Transform every element | `map` |
| Keep some elements | `filter` |
| Fold to one value | `reduce` (always pass an initial value) |
| First match | `find` / `findIndex` |
| Last match | `findLast` / `findLastIndex` |
| Any / all | `some` / `every` |
| Membership (incl. `NaN`) | `includes` |
| Last element | `at(-1)` |
| Deduplicate | `[...new Set(arr)]` |
| Group by key | `Object.groupBy` / `Map.groupBy` |
| Flatten one level | `flat()` / `flatMap` |
| Sort numbers | `toSorted((a, b) => a - b)` |
| Sort user-visible strings | `toSorted((a, b) => a.localeCompare(b))` |

**Choosing a loop**

| Need | Use |
|---|---|
| Values, with `break` or `await` | `for…of` |
| Index and value | `for (const [i, v] of arr.entries())` |
| Side effects only, no control flow | `forEach` |
| Index arithmetic, hot path | classic `for` |
| Object keys | `Object.keys/entries`, never `for…in` on arrays |
| Async sequence | `for await…of` |

**Iteration protocols**

| Concept | Contract |
|---|---|
| Iterable | has `[Symbol.iterator]()` → iterator |
| Iterator | has `next()` → `{ value, done }` |
| Async iterable | has `[Symbol.asyncIterator]()` |
| Generator | `function*` + `yield`; is both, and single-use |
| Delegate | `yield*` |
| Re-iterable object | `*[Symbol.iterator]() { … }` as a method |
| Lazy pipeline | iterator helpers: `.map().filter().take().toArray()` |

[↑ Back to top](#table-of-contents)

---

**Next:** [Parts 8–10 — Errors, Async & Modules](./javascript-errors-async-modules.md) · **Up:** [Mastery Guide overview](./javascript-mastery-guide.md)

*Written against ECMAScript 2026. Baseline data verified September 6, 2026.*
