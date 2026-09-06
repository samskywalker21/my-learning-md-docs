# JavaScript Mastery Guide (Pre-React)

> 🔄 **Needs re-write** — predates the current `prompt.md` template. Will be regenerated/updated to match it; treat as possibly stale on structure/format until then.

## About This Document

- **Framing:** **Goal-driven** — learning JavaScript specifically as preparation for React. Depth, examples, and prioritization are tuned to the language mechanics React leans on hardest (referential identity, closures, immutable updates, array transforms, async cancellation). This is *not* a React tutorial: you will not learn `useState` here. You will learn the JavaScript that makes `useState` behave the way it does.
- **React dosage:** **JS-first, React-motivated.** Every concept is taught as plain JavaScript. Each "Real Scenario" is a production-style React bug that the concept explains. React code appears only when it is the clearest way to show *why the JS matters* — never as React instruction in its own right.
- **Depth tiers requested:** **Beginner → Working Knowledge → Advanced → Mastery**, applied per Part. Beginner tiers are deliberately short (see assumed background) and are collapsed entirely where they would be filler. The weight of this doc sits in Working Knowledge and Advanced.
- **Assumed background:** Programming-fluent, JavaScript-rusty. You know variables, loops, functions, and objects from *some* language, and you have written some JavaScript. You do not need prior React experience.
- **Confirmed scope:**
  1. How React Changes What "Good JS" Means
  2. References, Equality & Immutability
  3. Functions, Closures & `this`
  4. Destructuring, Spread & Rest
  5. Arrays as the Rendering Toolkit
  6. Objects: Dynamic Keys, Optional Chaining, Nullish Coalescing
  7. Truthiness & Conditional Rendering
  8. Events & the DOM (the part React wraps)
  9. Asynchronous JavaScript in a UI
  10. Modules & Imports
  11. JSX Is Just JavaScript
  12. Mastery Grab Bag
- **Included by explicit request:** DOM event basics (event objects, bubbling, delegation, `preventDefault`) and browser async APIs (`fetch`, `AbortController`, `FormData`, `URLSearchParams`).
- **Excluded by design:** Build tooling and module resolution internals (bundlers, tree-shaking, HMR) — see [`vite/vite-mastery-guide.md`](../vite/vite-mastery-guide.md). CSS/styling. React's own API surface. TypeScript — see [`javascript/javascript-for-typescript-mastery.md`](./javascript-for-typescript-mastery.md).
- **Relationship to the other JS docs in this repo:** [`javascript-mastery-guide.md`](./javascript-mastery-guide.md) is the zero-experience, curiosity-driven language tutorial; [`javascript-for-typescript-mastery.md`](./javascript-for-typescript-mastery.md) is the JS→TS prep doc. This doc is the JS→React prep doc and cross-links both rather than repeating them.
- **Runtime for examples:** Modern evergreen browsers and Node.js 22+/24 LTS (Node 26 is Current as of September 2026). Snippets marked `js` run in a browser DevTools console or `node` REPL unless labelled otherwise.
- **Sourcing standard:** [MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript) and [react.dev](https://react.dev) first, Stack Overflow for real-world gotchas, other sources only as supplement. Verified September 2026 against React 19.2.x and the React Compiler 1.0 stable line.
- **If you ask me to update this doc later:** keep this structure (numbered Parts, tier subheadings inside each), keep the wrong-vs-right snippet pairs, keep the TOC and back-to-top links, keep the "Real Scenario" blocks tied to production-style React bugs, and prefer adding Parts/tiers over reverting to freeform prose. Collapsing a tier that would be filler is expected, not a defect to fix.

---

## Table of Contents

1. [How React Changes What "Good JS" Means](#1-how-react-changes-what-good-js-means)
2. [References, Equality & Immutability](#2-references-equality--immutability)
3. [Functions, Closures & `this`](#3-functions-closures--this)
4. [Destructuring, Spread & Rest](#4-destructuring-spread--rest)
5. [Arrays as the Rendering Toolkit](#5-arrays-as-the-rendering-toolkit)
6. [Objects: Dynamic Keys, Optional Chaining, Nullish Coalescing](#6-objects-dynamic-keys-optional-chaining-nullish-coalescing)
7. [Truthiness & Conditional Rendering](#7-truthiness--conditional-rendering)
8. [Events & the DOM](#8-events--the-dom)
9. [Asynchronous JavaScript in a UI](#9-asynchronous-javascript-in-a-ui)
10. [Modules & Imports](#10-modules--imports)
11. [JSX Is Just JavaScript](#11-jsx-is-just-javascript)
12. [Mastery Grab Bag](#12-mastery-grab-bag)
13. [Cheat Sheets](#13-cheat-sheets)
14. [Suggested Learning / Reference Order](#14-suggested-learning--reference-order)
15. [Quick Self-Check](#15-quick-self-check)

---

## 1. How React Changes What "Good JS" Means

**What it is.** React does not add features to JavaScript. It adds *constraints*. Three of them account for the overwhelming majority of "why is my component doing that" bugs:

1. **Your render function must be an expression pipeline, not a statement script.** JSX slots accept *values*, so you reach for ternaries, `&&`, and `.map()` where a plain script would use `if` and `for`.
2. **State must be replaced, never mutated.** React decides whether to re-render by comparing the *reference* of the old state to the new one with `Object.is`. Mutating in place leaves the reference identical, so React concludes nothing changed.
3. **Referential identity is a public API.** In plain JS, whether two functions or objects are `===` is usually an implementation detail. In React it determines whether a `useEffect` re-runs, whether a memoized child re-renders, and whether a list item is reused or destroyed.

Everything in this document is downstream of those three sentences.

```text
Plain JS mental model              React mental model
──────────────────────             ──────────────────────
obj.count += 1                     next = { ...obj, count: obj.count + 1 }
   ↓                                  ↓
same object, new contents          new object, old one untouched
   ↓                                  ↓
"it changed"                       Object.is(prev, next) === false → re-render

Mutation is invisible to React:

  prev ──┐
         ├──► { count: 1 }   (mutated to { count: 2 } — same box)
  next ──┘

  Object.is(prev, next) === true  →  React: "nothing to do"

Replacement is visible:

  prev ─────► { count: 1 }
  next ─────► { count: 2 }

  Object.is(prev, next) === false →  React: "re-render"
```

### Working Knowledge — the expression/statement split

JSX interpolation (`{ ... }`) evaluates an **expression**. `if`, `for`, `switch`, and `throw` are **statements** and cannot go there.

```jsx
// ❌ Wrong — `if` is a statement; this is a syntax error inside JSX.
<div>{ if (isLoggedIn) { <Dashboard /> } }</div>

// ✅ Right — ternary and && are expressions.
<div>{ isLoggedIn ? <Dashboard /> : <Login /> }</div>
```

You can always *hoist* statements above the `return` and interpolate the result:

```jsx
function Status({ state }) {
  let label;                                  // statements are fine out here
  switch (state) {
    case "ok":   label = "All good"; break;
    case "warn": label = "Degraded"; break;
    default:     label = "Down";
  }
  return <p>{label}</p>;                      // only the expression goes in the slot
}
```

**Real Scenario.** A junior dev converts a working `for` loop into JSX and gets `Unexpected token`. The fix is not a React fix — it is recognising that `for` produces no value. `[...items].map(...)` does. Part 5 covers the full toolkit.

[⬆ back to top](#table-of-contents)

---

## 2. References, Equality & Immutability

This is the single highest-leverage Part in the document. If you internalise only one thing before touching React, make it this.

### Beginner — primitives are copied, objects are referenced

```js
let a = 1;
let b = a;
b = 2;
console.log(a);            // 1 — numbers are copied by value

const x = { n: 1 };
const y = x;
y.n = 2;
console.log(x.n);          // 2 — objects are copied by *reference*
console.log(x === y);      // true — same object
```

`===` on objects asks "is this the *same* object?", never "do these look alike?" ([MDN — Equality comparisons](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Strict_equality)).

```js
{ n: 1 } === { n: 1 };     // false — two different objects
[1, 2] === [1, 2];         // false
```

### Working Knowledge — `Object.is`, and why React uses it

React compares state with [`Object.is`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is), not `===`. They differ in exactly two cases:

```js
NaN === NaN;               // false
Object.is(NaN, NaN);       // true

0 === -0;                  // true
Object.is(0, -0);          // false
```

For objects, they are identical: both compare references. So "React re-renders when the reference changes" is the whole rule.

```js
// ❌ Wrong — mutation. Reference is unchanged, so React bails out.
function addTodo(todos, text) {
  todos.push({ text, done: false });
  return todos;                            // same array!
}

// ✅ Right — replacement. New array, new reference.
function addTodo(todos, text) {
  return [...todos, { text, done: false }];
}
```

**The mutating methods to unlearn** — every one of these changes the array in place and returns something other than a fresh array:

| Mutating (avoid on state) | Immutable replacement |
|---|---|
| `arr.push(x)` | `[...arr, x]` |
| `arr.unshift(x)` | `[x, ...arr]` |
| `arr.pop()` / `arr.shift()` | `arr.slice(0, -1)` / `arr.slice(1)` |
| `arr.splice(i, 1)` | `arr.filter((_, j) => j !== i)` or `arr.toSpliced(i, 1)` |
| `arr[i] = v` | `arr.with(i, v)` |
| `arr.sort(fn)` | `arr.toSorted(fn)` |
| `arr.reverse()` | `arr.toReversed()` |

`toSorted`, `toReversed`, `toSpliced`, and `with` are the [ES2023 change-by-copy methods](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/toSorted) — Baseline across all modern browsers and Node 20+. They exist *precisely* for this pattern, and they are the current recommendation over the older `[...arr].sort()` workaround. `[...arr].sort()` is still correct and still common in codebases written before 2023; prefer `toSorted()` in new code.

```js
// ❌ Wrong — sort() mutates. This mutates React state in place, and the
//    reference is unchanged, so the UI silently doesn't update.
const sorted = users.sort((a, b) => a.age - b.age);

// ⚠️ Older-but-correct workaround, ubiquitous in pre-2023 code:
const sorted = [...users].sort((a, b) => a.age - b.age);

// ✅ Right — current, intention-revealing:
const sorted = users.toSorted((a, b) => a.age - b.age);
```

### Advanced — shallow vs. deep copies

Spread and `Object.assign` are **shallow**. Nested objects are still shared.

```js
const state = { user: { name: "Ada", prefs: { theme: "dark" } } };

const copy = { ...state };
copy.user.prefs.theme = "light";
console.log(state.user.prefs.theme);   // "light" — you mutated the original
```

To update nested state immutably you must rebuild every level down the path you touch:

```js
// ✅ Right — spread at each level on the path
const next = {
  ...state,
  user: {
    ...state.user,
    prefs: { ...state.user.prefs, theme: "light" },
  },
};
```

That verbosity is a real cost, and it is the reason [Immer](https://immerjs.github.io/immer/) exists (and why Redux Toolkit bundles it). It is worth writing the spreads by hand until the shape of the operation is second nature — then reach for a library.

[`structuredClone`](https://developer.mozilla.org/en-US/docs/Web/API/Window/structuredClone) gives you a real deep copy, available in all modern browsers and Node 17+:

```js
const deep = structuredClone(state);
deep.user.prefs.theme = "light";
state.user.prefs.theme;                // "dark" — original safe
```

Caveats that matter: it throws on functions, DOM nodes, and class instances lose their prototype (you get a plain object back). It is also *whole-tree* work — fine for a settings blob, wasteful for a 10,000-row table where you changed one cell.

> Do **not** use `JSON.parse(JSON.stringify(obj))` as a deep clone. It silently destroys `undefined`, `Date` (becomes a string), `Map`, `Set`, `NaN`/`Infinity` (become `null`), and throws on circular references. `structuredClone` handles all of those correctly. This is one of the clearest cases where a popular Stack Overflow answer from 2011 is now simply outdated — see the discussion on [How to deep clone a JavaScript object](https://stackoverflow.com/questions/122102/what-is-the-most-efficient-way-to-deep-clone-an-object-in-javascript).

**Real Scenario — the disappearing checkbox.** A todo list renders fine, but toggling a todo's `done` flag does nothing until an unrelated state update forces a re-render, at which point *all* the pending toggles appear at once.

```js
// ❌ The bug
function toggle(todos, id) {
  const todo = todos.find(t => t.id === id);
  todo.done = !todo.done;      // mutates the object *inside* the array
  return todos;                // and returns the same array reference
}
```

Two references are unchanged here: the array's, and the todo object's. React's bail-out check sees the same array and skips the render entirely. The mutation *did* happen — it is sitting in memory, invisible — which is why it flushes later when something else triggers a render. That "changes appear one interaction late" symptom is the fingerprint of a mutation bug.

```js
// ✅ The fix
function toggle(todos, id) {
  return todos.map(t => (t.id === id ? { ...t, done: !t.done } : t));
}
```

Note that `map` returns a new array, *and* the ternary returns a new object for the one changed row while passing every other row through by reference. Untouched rows keep their identity — which is exactly what lets a memoized `<TodoRow>` skip re-rendering. This is **structural sharing**, and it is why the immutable style is fast rather than slow.

```text
prev array:  [ A ,  B ,  C ,  D ]
                     │
                     └── toggled

next array:  [ A ,  B',  C ,  D ]
               ▲        ▲    ▲
               └────────┴────┴── same references — memoized children skip re-render
```

**Mastery.** Two tools for *catching* mutation rather than avoiding it:

```js
// Object.freeze — throws in strict mode (which modules always are)
const frozen = Object.freeze({ count: 1 });
frozen.count = 2;                        // TypeError in a module

// Proxy — log every write, dev-only
function watched(obj, label) {
  return new Proxy(obj, {
    set(target, prop, value) {
      console.trace(`mutation: ${label}.${String(prop)} =`, value);
      return Reflect.set(target, prop, value);
    },
  });
}
```

React's StrictMode double-invokes render functions in development precisely to surface impure renders. It does not freeze your state — these are the manual tools when you need to hunt a mutation down.

[⬆ back to top](#table-of-contents)

---

## 3. Functions, Closures & `this`

### Beginner — the three function forms

```js
function decl(a, b) { return a + b; }          // hoisted, has own `this`, has `arguments`
const expr = function (a, b) { return a + b; };// not hoisted
const arrow = (a, b) => a + b;                 // no own `this`, no `arguments`, implicit return
```

Arrow bodies: `x => x * 2` returns implicitly. `x => { x * 2 }` returns `undefined` — the braces are a block, not an object. To return an object literal, wrap it in parentheses:

```js
// ❌ Wrong — braces read as a function body; returns undefined
const toPair = id => { id, label: "x" };

// ✅ Right
const toPair = id => ({ id, label: "x" });
```

That second form bites constantly in `.map()` callbacks that build objects.

### Working Knowledge — closures

A closure is a function plus the variables it captured from where it was *defined* (not where it is called). ([MDN — Closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Closures))

```js
function counter() {
  let count = 0;                 // captured
  return () => ++count;          // closure over `count`
}
const next = counter();
next(); // 1
next(); // 2
```

The critical property: the closure captures the **variable binding**, not the value at capture time. If the variable is reassigned later, the closure sees the new value.

```js
let n = 1;
const show = () => console.log(n);
n = 2;
show();                          // 2 — sees the current value of the binding
```

But `const` bindings inside a function *call* can never be reassigned, so each call creates a fresh, permanently-fixed binding:

```js
function make(n) {
  return () => console.log(n);   // `n` is this call's parameter — frozen for this closure
}
const a = make(1), b = make(2);
a(); // 1
b(); // 2  — two independent closures
```

**That distinction is the entire mechanism behind React's stale-closure bug.**

### Advanced — stale closures

Every React render is a fresh call to your component function. Every value inside it — props, state, locals, and every function you define — belongs to *that* render and is frozen there. A callback that outlives its render (passed to `setTimeout`, an event listener, or an effect that doesn't re-subscribe) keeps seeing that render's values forever.

```text
Render #1                    Render #2                    Render #3
count = 0                    count = 1                    count = 2
handler₁ closes over 0       handler₂ closes over 1       handler₃ closes over 2
   │
   └── passed to setInterval on mount, never replaced
       → fires forever, always reading 0
```

Here is the mechanism in plain JavaScript, no React needed:

```js
// Simulate: a "render" that captures state, and a listener registered only once.
function render(count, register) {
  const handler = () => console.log("count is", count);   // closes over THIS count
  register(handler);
}

let registered = null;
const registerOnce = fn => { registered ??= fn; };          // only the first one sticks

render(0, registerOnce);
render(1, registerOnce);
render(2, registerOnce);
registered();                                              // "count is 0" — stale
```

In React that `registerOnce` is `useEffect(..., [])`. The empty dependency array is the "only the first one sticks" behaviour, and the logged `0` is the stale closure.

**Real Scenario — the counter that stops at 1.**

```jsx
// ❌ The bug
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, []);                     // ← lying about dependencies
```

The interval is created once, during the render where `count === 0`. Its callback captured that `0`. Every tick computes `setCount(0 + 1)`, so the counter goes `0 → 1` and freezes.

Three fixes, in ascending order of preference:

```jsx
// Fix 1 — updater function: don't read the captured value at all.
setInterval(() => setCount(c => c + 1), 1000);   // `c` is React's current value
```

```jsx
// Fix 2 — be honest about dependencies, accept the re-subscribe.
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, [count]);                 // tears down and re-creates each tick — correct, but wasteful
```

```jsx
// Fix 3 — useEffectEvent (React 19.2+, stable): a callback that always sees the
// latest values but does NOT count as a dependency.
const onTick = useEffectEvent(() => setCount(count + 1));
useEffect(() => {
  const id = setInterval(onTick, 1000);
  return () => clearInterval(id);
}, []);
```

`useEffectEvent` shipped stable in **React 19.2** (October 2025) after years as an experimental API; a great deal of blog content still describes it as "experimental" or spells it `useEvent`. See [react.dev — Separating Events from Effects](https://react.dev/learn/separating-events-from-effects) for the current guidance, and [LogRocket's React 19.2 write-up](https://blog.logrocket.com/react-19-2-is-here/) for the release context.

Fix 1 is still the right instinct for pure state arithmetic — it needs no hook at all. The underlying JS lesson: **when a long-lived callback needs a current value, don't capture the value — capture a way to get it.**

### Advanced — referential identity of functions

A function expression creates a *new function object* every time it is evaluated.

```js
const f = () => {};
const g = () => {};
f === g;                     // false — identical source, different objects
```

So a function defined inside a component body is a new reference on every render. That matters in exactly two places: `useEffect` dependency arrays, and props passed to memoized children.

```js
// Plain-JS demonstration of why this triggers work every time:
function makeDeps() { return [() => {}]; }
const d1 = makeDeps(), d2 = makeDeps();
d1[0] === d2[0];             // false → a dependency-array comparison sees "changed"
```

`useCallback` exists to pin that reference. **As of React Compiler 1.0 (stable, October 2025) this is largely automated** — the compiler inserts memoization for you, and the React team's current guidance is to treat `useMemo`/`useCallback` as escape hatches rather than defaults ([React Compiler v1.0](https://react.dev/blog/2025/10/07/react-compiler-1)). Manual memoization is still needed in a few places the compiler bails on: code inside `try/catch`, generators and async iterators, and third-party libraries whose internals depend on stable references. Understanding *why* the reference changes is not obsoleted by the compiler — the compiler removes the boilerplate, not the concept.

### Mastery — `this`, and why arrows won

`this` in a normal function is determined by *how it is called*, not where it is defined ([MDN — `this`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)):

```js
const obj = {
  name: "widget",
  regular() { return this.name; },
  arrow: () => this?.name,
};
obj.regular();               // "widget" — called as a method
obj.arrow();                 // undefined — arrow has no own `this`; uses module scope

const detached = obj.regular;
detached();                  // undefined (strict mode) — `this` was never bound
```

That last case is the whole reason class components needed `this.handleClick = this.handleClick.bind(this)` in constructors. Passing `onClick={this.handleClick}` detaches the method from its object; by the time React calls it, `this` is gone. Arrow functions and class fields solved it by capturing `this` lexically.

In function components you will essentially never write `this`. Know it anyway: you will read class components in older codebases, and `this` still appears in `Array.prototype` callbacks' optional `thisArg`, in `bind`/`call`/`apply`, and in DOM event handlers registered with `addEventListener` (where `this` is the element).

[⬆ back to top](#table-of-contents)

---

## 4. Destructuring, Spread & Rest

### Beginner

```js
const user = { id: 1, name: "Ada", role: "admin" };
const { name, role } = user;                       // object destructuring
const [first, second] = [10, 20];                  // array destructuring

const { name: displayName } = user;                // rename
const { missing = "n/a" } = user;                  // default for `undefined`
```

Defaults fire on `undefined` **only**, never on `null`:

```js
const { theme = "dark" } = { theme: null };
theme;                       // null — NOT "dark"
```

That is the same trap as `??` vs `||` in Part 6, and it hits any API that returns `null` for "unset".

### Working Knowledge — the React shapes

```jsx
// Props destructuring in the signature, with a default:
function Button({ label, variant = "primary", onClick }) { /* ... */ }

// Rest props — collect the leftovers and forward them to the DOM element:
function Input({ label, ...rest }) {
  return <label>{label}<input {...rest} /></label>;
}
```

Rest **must be last**, and it is shallow (it copies own enumerable properties one level deep).

```js
const { a, ...others } = { a: 1, b: 2, c: 3 };
others;                      // { b: 2, c: 3 }
```

Array destructuring is what makes `useState` read the way it does — `useState` returns a two-element array, and `const [count, setCount] = useState(0)` is ordinary array destructuring with names you choose. Nothing magic; that is also why the names are arbitrary and the *order* is fixed.

### Advanced — the gotchas that actually bite

**1. Nested destructuring throws on a missing parent.**

```js
// ❌ Wrong — if `user` is undefined, this throws before any default applies
const { user: { name } } = response;

// ✅ Right — default the parent too
const { user: { name } = {} } = response;

// ✅ Also right — often clearer
const name = response.user?.name;
```

**2. Spread order decides the winner.** Later wins.

```js
const defaults = { size: "md", variant: "primary" };
const props    = { variant: "danger" };

{ ...defaults, ...props };   // { size: "md", variant: "danger" }  ← props win
{ ...props, ...defaults };   // { size: "md", variant: "primary" } ← defaults clobber props
```

**Real Scenario.** A `<Button>` accepts `className` and merges it with its own base classes. Written as `{...rest, className: base}`, every caller's `className` is silently discarded and the design system team spends an afternoon wondering why overrides don't apply. The fix is to merge explicitly rather than rely on spread order:

```jsx
// ❌ Wrong — caller's className is thrown away
function Button({ ...rest }) {
  return <button {...rest} className="btn" />;
}

// ✅ Right — merge, and let the caller extend
function Button({ className = "", ...rest }) {
  return <button {...rest} className={`btn ${className}`} />;
}
```

**3. Spread copies own enumerable properties only.** Prototype methods and non-enumerable properties are dropped, so spreading a class instance gives you a plain object with no methods. Spreading a `Map` or `Set` into an object gives you `{}` — use `Object.fromEntries(map)` instead.

```js
class User { constructor(n) { this.name = n; } greet() { return `hi ${this.name}`; } }
const u = new User("Ada");
const copy = { ...u };
copy.greet;                  // undefined — method lived on the prototype
```

[⬆ back to top](#table-of-contents)

---

## 5. Arrays as the Rendering Toolkit

In React, arrays *are* the loop construct. Almost all list UI is a chain of array transforms ending in `.map()` that produces elements.

### Beginner — the core four

```js
const nums = [1, 2, 3, 4, 5];

nums.map(n => n * 2);                    // [2, 4, 6, 8, 10]      — transform, same length
nums.filter(n => n % 2 === 0);           // [2, 4]                — select, ≤ length
nums.find(n => n > 3);                   // 4                     — first match or undefined
nums.reduce((acc, n) => acc + n, 0);     // 15                    — fold to a single value
```

All four are non-mutating and return new arrays (or a value). That is why they are the React-safe set.

### Working Knowledge — the full working vocabulary

```js
const users = [
  { id: 1, name: "Ada",   team: "core", active: true  },
  { id: 2, name: "Grace", team: "core", active: false },
  { id: 3, name: "Alan",  team: "ops",  active: true  },
];

users.some(u => !u.active);              // true  — "any?"
users.every(u => u.active);              // false — "all?"
users.findIndex(u => u.id === 2);        // 1
users.flatMap(u => u.active ? [u.name] : []);   // ["Ada","Alan"] — map + filter in one pass
users.at(-1);                            // last element (ES2022) — clearer than [len-1]

// ES2024: group into an object keyed by a callback's return value
Object.groupBy(users, u => u.team);
// { core: [Ada, Grace], ops: [Alan] }
```

[`Object.groupBy`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/groupBy) landed in ES2024 and is Baseline in modern browsers and Node 21+. It replaces the hand-rolled `reduce` accumulator that appears in every "group an array by property" Stack Overflow answer. Note it returns a **null-prototype object**, so `result.hasOwnProperty` is `undefined` — use `Object.hasOwn(result, key)` or `key in result`. `Map.groupBy` is the sibling when your keys aren't strings.

Chaining reads top-to-bottom and each step is independent:

```jsx
const rows = users
  .filter(u => u.active)
  .toSorted((a, b) => a.name.localeCompare(b.name))
  .map(u => <Row key={u.id} user={u} />);
```

`localeCompare` rather than `a.name > b.name` matters as soon as your data has accents or non-Latin scripts — `>` compares UTF-16 code units, which puts `"Z"` before `"a"` and mis-sorts every diacritic.

### Advanced — `key`, and what identity means to a list

`key` is React's answer to "which of these array elements is the same element as last render?" It is the array equivalent of the referential-identity story from Part 2.

```jsx
// ❌ Wrong — index as key. Correct output, wrong identity.
{items.map((item, i) => <Row key={i} item={item} />)}

// ✅ Right — a stable id that travels with the data.
{items.map(item => <Row key={item.id} item={item} />)}
```

Index keys are *not* always wrong — they are fine for a list that is append-only and never reordered or filtered. They break the moment items can be inserted, deleted, or sorted, because index `0` then refers to a *different* item than it did last render, while React believes it is the same one and reuses its DOM node and internal state.

**Real Scenario — the typed text that jumps rows.** A list of editable inputs keyed by index. The user types "hello" in the third row, then deletes the first row. React sees three keys before (`0,1,2`) and two after (`0,1`), concludes that key `2` was removed and keys `0`/`1` survived — so it keeps the DOM nodes for `0` and `1` and updates their props. The uncontrolled input value "hello" stays attached to position index 1, which is now a *different* item. The text appears to jump to the wrong row. Nothing about this is fixable inside the component; the only fix is a key that identifies the *item*, not the *slot*.

```text
Before delete:  key 0 → [Item A]   key 1 → [Item B]   key 2 → [Item C: "hello"]
Delete A:
After delete:   key 0 → [Item B]   key 1 → [Item C]
                        ▲                  ▲
                 reuses node from   reuses node from key 1,
                 key 0 (was A)      which held B's DOM state
```

If your data genuinely has no id, generate one when you create the item — `crypto.randomUUID()` is available in all modern browsers and Node 19+ — and store it *in state*, not computed during render. Generating a key during render produces a new key every render, which destroys and recreates every row.

```js
// ❌ Wrong — new key every render; the whole list remounts each time
{items.map(item => <Row key={crypto.randomUUID()} item={item} />)}

// ✅ Right — id assigned once, at creation time
setItems(prev => [...prev, { id: crypto.randomUUID(), text }]);
```

### Mastery — `reduce`, and when not to use it

`reduce` is the general fold; `map`, `filter`, `some`, and `every` are all special cases of it. Its signature — `reduce((accumulator, current, index, array) => next, initialValue)` — is worth knowing cold, along with the rule that **omitting `initialValue` throws on an empty array**.

```js
[].reduce((a, b) => a + b);              // TypeError: Reduce of empty array with no initial value
[].reduce((a, b) => a + b, 0);           // 0 ✅
```

The other `reduce` trap is mutating the accumulator, which is a real performance-vs-clarity tradeoff rather than a bug:

```js
// ⚠️ Mutates the accumulator — fast, and safe *because* the accumulator is
//    created fresh by this reduce and never escapes until it's returned.
const byId = users.reduce((acc, u) => { acc[u.id] = u; return acc; }, {});

// ❌ Actually wrong — spreading rebuilds the whole object on every iteration: O(n²)
const byId = users.reduce((acc, u) => ({ ...acc, [u.id]: u }), {});
```

The spread version appears in a great many tutorials as the "immutable" form. It is immutable in a way that buys nothing — the intermediate objects are garbage the moment they're created — and it turns a linear operation quadratic. For 10 items nobody notices; for 10,000 it is a visible freeze. Prefer `Object.fromEntries(users.map(u => [u.id, u]))`, which is both clear and linear.

Rule of thumb: if a `for...of` loop would be clearer than the `reduce`, write the loop. Building a local array or object with a loop and returning it is perfectly immutable from the caller's perspective — nothing outside the function ever saw the mutation.

[⬆ back to top](#table-of-contents)

---

## 6. Objects: Dynamic Keys, Optional Chaining, Nullish Coalescing

### Working Knowledge — computed keys and form state

Square brackets in an object literal evaluate an expression as the key ([MDN — computed property names](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Object_initializer#computed_property_names)). This is the backbone of generic form handling.

```js
const field = "email";
const patch = { [field]: "a@b.com" };    // { email: "a@b.com" }
```

```jsx
// One handler for every field in a form:
function handleChange(e) {
  const { name, value } = e.target;
  setForm(prev => ({ ...prev, [name]: value }));   // note: computed key + spread
}
```

```js
// ❌ Wrong — quotes make it the literal string "field"
setForm(prev => ({ ...prev, field: value }));

// ✅ Right
setForm(prev => ({ ...prev, [field]: value }));
```

Note the parentheses around the object in `prev => ({ ... })` — without them the braces are a function body (Part 3) and the updater returns `undefined`, wiping your state. This exact typo is one of the most common React-beginner bugs, and it is a pure JavaScript arrow-function issue.

### Working Knowledge — optional chaining

```js
const res = { data: { user: null } };

res.data.user.name;                      // TypeError: Cannot read properties of null
res.data?.user?.name;                    // undefined — short-circuits safely

res.data?.items?.[0];                    // optional element access
res.onSelect?.(id);                      // optional call — no-op if undefined
```

`?.` short-circuits the **entire rest of the chain** the moment it hits `null` or `undefined`. That's why `a?.b.c.d` is safe if `a` is nullish — but it will still throw if `a` exists and `b` is nullish. Put `?.` at each genuinely-optional link, not just the first.

The optional-call form `fn?.()` is the clean replacement for `if (fn) fn()` and is ideal for optional callback props.

### Advanced — `??` vs `||`

`||` falls back on any **falsy** value: `false`, `0`, `""`, `NaN`, `null`, `undefined`.
`??` falls back only on `null` and `undefined` ([MDN — Nullish coalescing](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing)).

```js
0        || 10;    // 10   ← the bug
0        ?? 10;    // 0    ← what you meant
""       || "N/A"; // "N/A"
""       ?? "N/A"; // ""
null     ?? "N/A"; // "N/A"
```

**Real Scenario — the quantity that won't go to zero.** A cart line item renders `<span>{quantity || 1}</span>` and the "set quantity to 0" button appears to do nothing. Zero is falsy, so `||` swaps in the default. Worse, the same `||` in the submit handler sends `1` to the API, so the server records a quantity the user never chose. The fix is one character: `quantity ?? 1`.

The same trap wearing a different hat, with a boolean prop:

```jsx
// ❌ Wrong — passing `disabled={false}` explicitly gets overridden to true
function Button({ disabled }) {
  const isDisabled = disabled || true;   // always true!
}

// ✅ Right — use a parameter default, which fires only on `undefined`
function Button({ disabled = false }) { /* ... */ }
```

Syntax note: `??` cannot be mixed with `||` or `&&` without explicit parentheses — `a ?? b || c` is a **SyntaxError**, deliberately, because the precedence would be ambiguous. Write `(a ?? b) || c`.

There is also a logical-assignment family worth knowing, all ES2021:

```js
opts.retries ??= 3;          // assign only if null/undefined
config.debug ||= false;      // assign if falsy
flags.ready &&= isValid;     // assign if truthy
```

### Mastery — object iteration and safe key checks

```js
const o = { a: 1, b: 2 };

Object.keys(o);              // ["a", "b"]
Object.values(o);            // [1, 2]
Object.entries(o);           // [["a",1], ["b",2]]
Object.fromEntries([["a",1]]);// { a: 1 }

Object.hasOwn(o, "a");       // true — ES2022, replaces obj.hasOwnProperty(...)
```

[`Object.hasOwn`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/hasOwn) is the current recommendation over `obj.hasOwnProperty(key)`, which breaks on null-prototype objects (like `Object.groupBy`'s result) and on any object with a property literally named `hasOwnProperty` — a real hazard when the object came from parsed JSON you don't control.

The `Object.entries` → `map` → `Object.fromEntries` round-trip is the object equivalent of an array transform:

```js
// Transform every value, immutably
const doubled = Object.fromEntries(
  Object.entries(o).map(([k, v]) => [k, v * 2])
);                           // { a: 2, b: 4 }
```

[⬆ back to top](#table-of-contents)

---

## 7. Truthiness & Conditional Rendering

### Working Knowledge — the falsy list

Exactly eight values are falsy. Memorise them; everything else, including `[]`, `{}`, `"0"`, and `"false"`, is truthy.

```js
false, 0, -0, 0n, "", null, undefined, NaN
```

```js
if ([])  console.log("runs");    // runs — empty array is truthy
if ({})  console.log("runs");    // runs
if ("0") console.log("runs");    // runs — non-empty string
```

The empty-array case matters: `if (items)` is always true, so the check you actually want is `if (items.length)` or `if (items.length > 0)`.

### Advanced — `&&` returns a value, and React renders it

`&&` is not a boolean operator; it returns one of its **operands**. `a && b` returns `a` if `a` is falsy, otherwise `b`. Likewise `a || b` returns `a` if truthy, otherwise `b`.

```js
0 && "hello";                // 0        ← not false!
"" && "hello";               // ""
1 && "hello";                // "hello"
```

React renders `null`, `undefined`, `false`, and `true` as nothing. It renders `0` and `""` as **text**. That asymmetry produces the single most-reported conditional-rendering bug:

**Real Scenario — the stray "0" in the UI.**

```jsx
// ❌ Wrong — when items is empty, `items.length` is 0, `&&` returns 0,
//    and React renders a literal "0" on the page.
{items.length && <ItemList items={items} />}

// ✅ Right — coerce to a real boolean
{items.length > 0 && <ItemList items={items} />}

// ✅ Also right — ternary makes the empty branch explicit
{items.length ? <ItemList items={items} /> : null}
```

This ships to production regularly because it only manifests on the empty state, which is often the least-tested path. The `> 0` version is preferable to `!!items.length` — same behaviour, but it states the intent.

The same rule applies to any numeric guard: `{count && <Badge count={count} />}` renders `0` when the count is zero, which is exactly when you wanted nothing.

### Mastery — early returns and the guard pattern

The cleanest conditional rendering often isn't conditional JSX at all — it is an early `return`, which is a statement and therefore lives above the JSX:

```jsx
function Profile({ user, isLoading, error }) {
  if (isLoading) return <Spinner />;
  if (error)     return <ErrorPanel error={error} />;
  if (!user)     return null;                      // render nothing

  return <div>{user.name}</div>;                   // the happy path, undented
}
```

This scales far better than a nested ternary chain. Nested ternaries in JSX (`a ? x : b ? y : z`) are legal and occasionally fine for two branches, but they become unreadable at three and are worth refactoring into early returns or a lookup object:

```jsx
// A lookup object beats a ternary chain when branches are a closed set
const VIEWS = { loading: Spinner, error: ErrorPanel, ready: Profile };
const View = VIEWS[status] ?? NotFound;
return <View {...props} />;
```

Note that the component variable must be **capitalised** — `<View />` compiles to a reference to the variable `View`, while `<view />` compiles to the string `"view"` (an HTML tag). Part 11 explains why.

[⬆ back to top](#table-of-contents)

---

## 8. Events & the DOM

React wraps DOM events rather than replacing them. Knowing the underlying model tells you what React's handlers actually receive.

### Beginner — listeners and the event object

```js
const btn = document.querySelector("#save");
btn.addEventListener("click", (event) => {
  console.log(event.type);            // "click"
  console.log(event.target);          // the element that was actually clicked
  console.log(event.currentTarget);   // the element the listener is attached to
});
```

`target` vs. `currentTarget` is the distinction people get wrong. If you attach a listener to a `<button>` containing an `<svg>` icon, clicking the icon gives you `target === <svg>` and `currentTarget === <button>`. In a form handler, `e.target` is the input that fired; in a submit handler on the `<form>`, `e.target` is the form.

```jsx
// The React equivalent — same event object shape:
<input name="email" onChange={e => console.log(e.target.name, e.target.value)} />
```

### Working Knowledge — bubbling, capturing, and `preventDefault`

An event travels down from `window` to the target (**capture phase**), fires on the target, then travels back up (**bubble phase**). Listeners run in the bubble phase by default.

```text
              capture ↓                 bubble ↑
  window  ──────────────────────────────────────────
    document ────────────────────────────────────
      body ──────────────────────────────────
        div#list ────────────────────────
          button  ◄── target
```

```js
el.addEventListener("click", fn);                 // bubble phase (default)
el.addEventListener("click", fn, true);           // capture phase
el.addEventListener("click", fn, { capture: true, once: true, passive: true });
```

Two methods that are constantly confused:

- **`preventDefault()`** — cancel the browser's default action (form submission, link navigation, checkbox toggle). Does not stop propagation.
- **`stopPropagation()`** — stop the event travelling further up. Does not cancel the default action.

```js
// ❌ Wrong — the page still reloads; stopPropagation doesn't cancel submission
form.addEventListener("submit", e => { e.stopPropagation(); send(); });

// ✅ Right
form.addEventListener("submit", e => { e.preventDefault(); send(); });
```

In React, returning `false` from a handler does **nothing** — that's a jQuery convention. You must call `preventDefault()` explicitly.

**Real Scenario — the modal that closes when you click inside it.** A modal closes on backdrop click via a listener on the overlay div. Clicks inside the dialog bubble up to the overlay and close it. The naive fix — `stopPropagation()` on the dialog — works but silently breaks anything else listening higher up (analytics click tracking, a "close all dropdowns" handler on `document`). The better fix uses `target`/`currentTarget`:

```jsx
// ⚠️ Works, but swallows the event for every other listener above
<div onClick={close}><div onClick={e => e.stopPropagation()}>…</div></div>

// ✅ Better — only close when the backdrop itself was the click target
<div onClick={e => { if (e.target === e.currentTarget) close(); }}>
  <div>…</div>
</div>
```

### Advanced — event delegation

Instead of one listener per row, attach one listener to the container and identify the source from `event.target`. This is how you handle a 10,000-row table without 10,000 listeners, and it works for rows added later.

```js
// ❌ Wrong — N listeners, and new rows get none
document.querySelectorAll(".row").forEach(row =>
  row.addEventListener("click", handle)
);

// ✅ Right — one listener, works for future rows too
list.addEventListener("click", (e) => {
  const row = e.target.closest("[data-id]");
  if (!row || !list.contains(row)) return;
  handle(row.dataset.id);
});
```

`closest()` walks up from the target to the nearest matching ancestor — the standard way to recover "which row" from "which pixel". `dataset` reads `data-*` attributes.

**React does this for you.** React attaches listeners at the root container of your app, not to individual DOM nodes, and dispatches through its own tree. This is why:

- `onClick` on a React component works even though there is no listener on that DOM node.
- Mixing `addEventListener` on `document` with React handlers can produce surprising ordering — the React root's listener fires when the event reaches the root, which may be before or after your manual `document` listener depending on phase.
- `e.stopPropagation()` inside a React handler stops propagation *within React's tree* and also prevents the native event from reaching listeners attached above React's root container.

### Mastery — controlled inputs, `FormData`, and passive listeners

A **controlled input** is one whose `value` prop comes from state — the DOM element is told what to display on every render and its own internal value is overwritten. An **uncontrolled** input owns its own value, and you read it out on submit.

```jsx
// Uncontrolled — no state per keystroke, read everything at submit time.
<form onSubmit={(e) => {
  e.preventDefault();
  const data = Object.fromEntries(new FormData(e.currentTarget));
  console.log(data);       // { email: "...", password: "..." }
}}>
```

[`FormData`](https://developer.mozilla.org/en-US/docs/Web/API/FormData) reads every named form control, so `Object.fromEntries(new FormData(form))` turns an entire form into a plain object in one line. The catch: it only captures inputs with a `name` attribute, it *drops* unchecked checkboxes entirely, and duplicate names collapse to the last value (use `formData.getAll(name)` for multi-selects and checkbox groups).

Related: [`URLSearchParams`](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams) has the same entries-based shape and handles query-string encoding correctly:

```js
const qs = new URLSearchParams({ q: "a b&c", page: "2" }).toString();
// "q=a+b%26c&page=2"  — encoding handled; never hand-build query strings
Object.fromEntries(new URLSearchParams(location.search));
```

One performance note that surfaces in scroll handlers: `addEventListener("scroll", fn, { passive: true })` promises you will not call `preventDefault()`, letting the browser scroll without waiting for your handler. Modern browsers already default `touchstart`/`touchmove`/`wheel` on the document to passive; be explicit anyway on your own scroll containers.

[⬆ back to top](#table-of-contents)

---

## 9. Asynchronous JavaScript in a UI

### Beginner — promises and `async`/`await`

A promise is a value that will exist later, in one of three states: pending, fulfilled, or rejected ([MDN — Using promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises)).

```js
const p = fetch("/api/users");           // pending immediately
p.then(res => res.json())
 .then(data => console.log(data))
 .catch(err => console.error(err))
 .finally(() => console.log("done"));
```

`async`/`await` is the same thing with statement syntax. An `async` function *always* returns a promise; `await` unwraps one.

```js
async function load() {
  try {
    const res = await fetch("/api/users");
    if (!res.ok) throw new Error(`HTTP ${res.status}`);   // ← see gotcha below
    return await res.json();
  } catch (err) {
    console.error(err);
    throw err;
  }
}
```

**The `fetch` gotcha every developer hits once:** `fetch` only rejects on *network* failure. A `404` or `500` is a perfectly successful HTTP transaction as far as `fetch` is concerned, so the promise **fulfills**. You must check `res.ok` yourself ([MDN — Using Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)).

```js
// ❌ Wrong — a 500 sails straight into the success path and .json() throws
//    an unhelpful "Unexpected token < in JSON" when the server returns an HTML error page.
const data = await fetch(url).then(r => r.json());

// ✅ Right
const res = await fetch(url);
if (!res.ok) throw new Error(`Request failed: ${res.status}`);
const data = await res.json();
```

### Working Knowledge — running things in parallel

```js
// ❌ Sequential — 300ms total, for no reason
const user  = await getUser(id);
const posts = await getPosts(id);
const likes = await getLikes(id);

// ✅ Parallel — ~100ms; all three start immediately
const [user, posts, likes] = await Promise.all([
  getUser(id), getPosts(id), getLikes(id),
]);
```

`await` inside a `for` loop is the same trap at scale — it serialises every iteration. If the operations are independent, `map` to promises and `Promise.all` the array.

| Combinator | Settles when | Rejects when |
|---|---|---|
| `Promise.all` | all fulfill | **any** rejects (fail-fast) |
| `Promise.allSettled` | all settle | never — returns `{status, value/reason}[]` |
| `Promise.race` | first settles | if the first to settle rejects |
| `Promise.any` | first **fulfills** | all reject (`AggregateError`) |

Use `allSettled` for dashboards where one dead widget shouldn't blank the page:

```js
const results = await Promise.allSettled(widgets.map(w => w.load()));
const loaded = results.filter(r => r.status === "fulfilled").map(r => r.value);
```

Also worth knowing: [`Promise.try`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/try) (ES2025, Node 22.6+) wraps a function that might be sync *or* async and guarantees a promise, routing synchronous throws into rejection instead of letting them escape:

```js
// A sync throw inside this callback becomes a rejection, not an uncaught error
Promise.try(() => maybeSyncMaybeAsync()).catch(handle);
```

### Advanced — race conditions and `AbortController`

**This is the async bug React developers actually ship.** Async responses arrive in completion order, not request order.

```text
User types "a"  ──► request A ────────────────────────────► response A (slow)
User types "ab" ──► request B ──────────► response B (fast)
                                              │                    │
                                        setResults(B)        setResults(A)
                                                                   ▲
                                          stale response overwrites fresh one —
                                          UI now shows results for "a" while
                                          the input reads "ab"
```

```js
// ❌ Wrong — no cancellation, no ordering guarantee
async function search(q) {
  const res = await fetch(`/api/search?q=${q}`);
  setResults(await res.json());          // may be a stale response
}
```

[`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) is the standard fix. One controller per request; abort the previous one when a new request starts, or when the component unmounts.

```js
// ✅ Right — cancel the in-flight request when inputs change or on unmount
useEffect(() => {
  const controller = new AbortController();

  (async () => {
    try {
      const res = await fetch(`/api/search?q=${query}`, { signal: controller.signal });
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      setResults(await res.json());
    } catch (err) {
      if (err.name === "AbortError") return;    // expected — not a real error
      setError(err);
    }
  })();

  return () => controller.abort();              // cleanup runs before the next effect
}, [query]);
```

Three details that matter:

1. **Aborting rejects the fetch promise** with an `AbortError` (a `DOMException`). You must filter it out, or every keystroke logs a scary error. Check `err.name === "AbortError"`.
2. **The effect cleanup function is the cancellation hook.** React calls it before re-running the effect and on unmount — precisely the two moments you want to abort.
3. **`AbortSignal` is a general protocol, not a fetch feature.** `addEventListener(type, fn, { signal })` removes the listener when the signal aborts, which lets one controller tear down a whole cluster of subscriptions.

```js
// One controller, many teardowns
const c = new AbortController();
window.addEventListener("resize", onResize, { signal: c.signal });
window.addEventListener("keydown", onKey,   { signal: c.signal });
c.abort();                               // both listeners removed
```

Handy helpers: `AbortSignal.timeout(5000)` gives you a signal that aborts itself (a timeout without any `setTimeout` bookkeeping), and `AbortSignal.any([a, b])` combines several.

```js
const res = await fetch(url, {
  signal: AbortSignal.any([controller.signal, AbortSignal.timeout(5_000)]),
});
```

An alternative to cancellation, when you can't abort, is an **ignore flag** — you still receive the stale response, you just refuse to act on it:

```js
useEffect(() => {
  let ignore = false;
  load(query).then(data => { if (!ignore) setResults(data); });
  return () => { ignore = true; };
}, [query]);
```

react.dev uses this pattern in [Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects#fetching-data) because it works for any promise, not just `fetch`. Prefer `AbortController` when the underlying API supports it — it actually stops the network request rather than discarding the result.

### Mastery — the event loop, microtasks, and why batching looks odd

Understanding execution order explains several React behaviours that otherwise look arbitrary.

```text
┌─────────────────────────────────────────────┐
│  Call stack — runs synchronous code to       │
│  completion. Nothing else can interleave.    │
└──────────────────────┬──────────────────────┘
                       │ stack empties
                       ▼
┌─────────────────────────────────────────────┐
│  Microtask queue — DRAINED COMPLETELY        │
│  .then / await continuations, queueMicrotask │
│  (new microtasks added here also run now)    │
└──────────────────────┬──────────────────────┘
                       ▼
┌─────────────────────────────────────────────┐
│  Render / paint (browser)                    │
└──────────────────────┬──────────────────────┘
                       ▼
┌─────────────────────────────────────────────┐
│  ONE macrotask — setTimeout, DOM event,      │
│  message. Then back to the top.              │
└─────────────────────────────────────────────┘
```

```js
console.log("1");
setTimeout(() => console.log("2"), 0);      // macrotask
Promise.resolve().then(() => console.log("3"));  // microtask
queueMicrotask(() => console.log("4"));     // microtask
console.log("5");

// Output: 1, 5, 3, 4, 2
```

Microtasks always run before the next macrotask, and the microtask queue is drained *completely* — a microtask that queues another microtask delays rendering further. An infinite microtask loop freezes the page in a way an infinite `setTimeout` loop does not.

Two consequences in React:

**1. State updates are asynchronous within the same synchronous block.**

```js
// ❌ Wrong — reading state right after setting it
function handleClick() {
  setCount(count + 1);
  console.log(count);                    // still the OLD value
}
```

`count` is a `const` from *this* render's closure (Part 3). It cannot change. React schedules the update and re-renders with a new closure. If you need the new value, compute it yourself or use the updater form.

**2. Multiple updates in one handler are batched into a single re-render.**

```js
setA(1);
setB(2);
setC(3);
// → ONE re-render, not three
```

Since React 18 this batching applies everywhere — including inside promises, `setTimeout`, and native event handlers — via *automatic batching*. Before React 18, batching only happened inside React event handlers, so `setTimeout(() => { setA(1); setB(2); })` produced two renders. A lot of pre-2022 blog content still describes the old behaviour. `flushSync` from `react-dom` is the opt-out when you genuinely need a synchronous DOM update (measuring an element immediately after a state change, for example).

**3. `await` always yields, even on an already-resolved value.**

```js
async function f() {
  console.log("a");
  await null;                            // yields to the microtask queue
  console.log("c");                      // runs as a microtask
}
f();
console.log("b");
// a, b, c
```

Everything after the first `await` in an async function runs asynchronously, even if you awaited a non-promise. That's why an early `await` inside a click handler can push subsequent `setState` calls into a different batch than the code before it.

[⬆ back to top](#table-of-contents)

---

## 10. Modules & Imports

### Beginner — named and default exports

```js
// utils.js
export const formatDate = d => d.toISOString();      // named
export function slugify(s) { /* ... */ }             // named
export default function Button() { /* ... */ }       // default (one per file)
```

```js
// consumer.js
import Button, { formatDate, slugify } from "./utils.js";
import Btn from "./utils.js";              // default import — name is arbitrary
import { formatDate as fmt } from "./utils.js";
import * as utils from "./utils.js";       // namespace import
```

The asymmetry is the point: **named imports must match the exported name; default imports can be called anything.** That flexibility is also default exports' main weakness — rename a component and every import site keeps the old name, so grep stops finding things. Many teams standardise on named exports for that reason; React itself has no preference, though `React.lazy` requires a default export (see below).

### Working Knowledge — module semantics that surprise people

**Modules are singletons.** A module's top-level code runs exactly once, no matter how many files import it. That is what makes a module-scoped variable a de facto global.

```js
// counter.js
export let hits = 0;
export const bump = () => ++hits;
```

Every importer shares the same `hits`. This is genuinely useful (a shared client instance) and a genuine footgun (module-scoped mutable state in SSR leaks between requests).

**Imports are live bindings, not copies.**

```js
import { hits, bump } from "./counter.js";
console.log(hits);   // 0
bump();
console.log(hits);   // 1 — the binding updated, even though `hits` looks like a const here
```

You still cannot assign to an imported binding (`hits = 5` is a TypeError) — it's read-only from the consumer's side, but it tracks the exporter.

**Modules are always strict mode**, and always deferred (they run after the document is parsed). There is no `"use strict"` to add.

**Imports are hoisted and static.** `import` statements are resolved before any code in the file runs, which is why they must be at the top level — you cannot conditionally `import` inside an `if`.

### Advanced — dynamic `import()` and code splitting

`import()` is a *function-like expression* that returns a promise for the module namespace. It can appear anywhere.

```js
// Load a heavy dependency only when it's actually needed
async function exportToPdf(data) {
  const { jsPDF } = await import("jspdf");     // fetched on demand
  new jsPDF().text(data, 10, 10).save();
}
```

This is the mechanism behind `React.lazy`:

```jsx
const Settings = React.lazy(() => import("./Settings.jsx"));
// Settings.jsx must have a DEFAULT export — React.lazy reads `.default`
```

If your component uses a named export, adapt it in the loader:

```jsx
const Settings = React.lazy(() =>
  import("./Settings.jsx").then(m => ({ default: m.Settings }))
);
```

**Real Scenario — the barrel file that kills your code splitting.** A team adds `components/index.js` re-exporting all 80 components, then lazy-loads one route:

```js
// components/index.js
export * from "./Button.jsx";
export * from "./DataGrid.jsx";      // pulls in a 400 kB charting dependency
// ...78 more
```

```js
// Route.jsx
import { Button } from "../components";     // ← imports the barrel
```

Importing the barrel evaluates the whole barrel module, so `DataGrid` and its charting library land in the bundle even though nothing renders a grid. Bundlers can tree-shake this *if* every module in the chain is side-effect-free and marked as such (`"sideEffects": false` in `package.json`), but a single module with a top-level side effect — a CSS import, a polyfill, a registry `push` — defeats it. Symptom: a lazy chunk that is suspiciously large, or a "shared" chunk containing things no route uses.

The reliable fix is to import from the source module directly in hot paths:

```js
// ✅ Deep import — no barrel evaluation
import { Button } from "../components/Button.jsx";
```

Barrels are fine for a small, genuinely-cohesive folder and for a package's public entry point. They are a liability as an app-wide `components/index.js`. Bundler specifics live in [`vite/vite-mastery-guide.md`](../vite/vite-mastery-guide.md).

### Mastery — ESM vs. CommonJS, and where you still meet CJS

You will write ESM (`import`/`export`) in React apps exclusively. CommonJS (`require`/`module.exports`) still shows up in config files, older Node tooling, and dependencies.

| | ESM | CommonJS |
|---|---|---|
| Syntax | `import` / `export` | `require()` / `module.exports` |
| Resolution | static, hoisted | dynamic, runtime |
| Loading | asynchronous | synchronous |
| Strict mode | always | opt-in |
| Top-level `await` | yes | no |
| File extension in imports | **required** in Node ESM | optional |

The extension rule catches everyone moving from bundled code to plain Node: `import "./utils"` works in Vite (the bundler resolves it) but throws `ERR_MODULE_NOT_FOUND` in Node ESM, which requires `import "./utils.js"`.

The mixing rules: ESM can `import` a CJS module (you get `module.exports` as the default export). CJS cannot `require` an ESM module synchronously — it must use dynamic `import()`. Node signals which mode a `.js` file is in via `"type": "module"` in `package.json`; `.mjs` and `.cjs` force it per-file. Node 22+ has experimental support for `require()`ing synchronous ESM graphs, which softens this, but the ESM-only direction remains the safe assumption.

Top-level `await` is ESM-only and worth knowing:

```js
// config.js — an ES module
const config = await fetch("/config.json").then(r => r.json());
export default config;                 // importers block until this resolves
```

[⬆ back to top](#table-of-contents)

---

## 11. JSX Is Just JavaScript

### Working Knowledge — what JSX compiles to

JSX is syntax sugar, transformed at build time. Since **React 17**, the default output is the *automatic runtime* — a call to `jsx()` imported from `react/jsx-runtime` — not `React.createElement`.

```jsx
// You write:
<Button variant="primary" onClick={save}>Save</Button>
```

```js
// The automatic runtime emits (conceptually):
import { jsx as _jsx } from "react/jsx-runtime";
_jsx(Button, { variant: "primary", onClick: save, children: "Save" });
```

```js
// The legacy (classic) runtime emitted:
React.createElement(Button, { variant: "primary", onClick: save }, "Save");
```

Two practical consequences of the automatic runtime:

1. **`import React from "react"` is no longer required** in files that only use JSX. Tutorials and Stack Overflow answers written before 2020 all include it; it is dead weight now, and the `react/react-in-jsx-scope` ESLint rule is disabled in modern configs. You still need the import if you use `React.something` explicitly.
2. The transform is a build-time concern configured by your toolchain (`@vitejs/plugin-react` etc.), which is why JSX never appears in a plain `.js` file run by `node`.

The return value is a plain object — a **React element**, which is a *description* of UI, not a DOM node:

```js
console.log(<div className="a">hi</div>);
// { $$typeof: Symbol(react.element), type: "div", key: null,
//   props: { className: "a", children: "hi" }, ... }
```

Elements are immutable and cheap. Creating them is not rendering; React diffs the descriptions and touches the DOM only where they differ.

### Working Knowledge — the JS rules that explain JSX's quirks

Almost every "JSX rule" is really a JavaScript rule:

| JSX behaviour | The JavaScript reason |
|---|---|
| Components must be **Capitalised** | `<foo />` compiles to the *string* `"foo"`; `<Foo />` compiles to the *variable* `Foo`. Lowercase = HTML tag. |
| `className`, not `class` | `class` is a reserved word; props are object keys in a JS object literal. (`htmlFor` for the same reason.) |
| Attributes are camelCase | They are JS object properties matching DOM IDL names (`onClick`, `tabIndex`, `readOnly`). |
| One root element required | A function returns *one* value. Fragments (`<>…</>`) provide a wrapper with no DOM output. |
| `{}` takes an expression, not a statement | It becomes an argument in a function call. |
| `{{ }}` for inline styles | Outer braces = JSX slot; inner braces = an object literal. Not special syntax. |
| Self-closing tags mandatory (`<br />`) | JSX is XML-like; the parser requires closure. |
| Attribute values can't use `?.` on the tag | Not a JS limitation — `<Foo?.Bar />` simply isn't in the grammar. Assign to a variable first. |

```jsx
// {{ }} demystified
<div style={{ color: "red", fontSize: 14 }} />
//        ▲▲                              ▲▲
//        ││                              └┴─ closing object + slot
//        │└─ object literal
//        └── JSX expression slot
```

Note `fontSize: 14` — React adds `px` to numeric values for most dimensional CSS properties. `lineHeight: 2` correctly stays unitless.

### Advanced — children are just a prop

`children` is an ordinary prop that JSX fills from the tag's contents. Anything a JS expression can produce can be a child, including a function.

```jsx
// These two are identical:
<Card>Hello</Card>
<Card children="Hello" />
```

Because children can be *any value*, the "render prop" pattern requires no special support:

```jsx
function List({ items, children }) {
  return <ul>{items.map(item => <li key={item.id}>{children(item)}</li>)}</ul>;
}

<List items={users}>{user => <strong>{user.name}</strong>}</List>
```

Arrays of elements are flattened into children, which is why `.map()` works directly in a slot — and why each element in that array needs a `key` (Part 5).

React ignores `null`, `undefined`, `true`, and `false` as children but renders `0` and `""` — the Part 7 gotcha, stated at the element level.

### Mastery — spread props and the resulting hazards

```jsx
<Component {...props} />
```

compiles to an object spread in the props object — same semantics as Part 4, same "later wins" ordering rule. Two hazards:

```jsx
// ❌ Passing unknown props straight to a DOM element produces React warnings and
//    invalid HTML attributes (e.g. isActive={true} → <div isActive="true">).
function Row({ isActive, ...rest }) { return <div {...rest} isActive={isActive} />; }

// ✅ Destructure out the props that are yours before forwarding the rest.
function Row({ isActive, ...rest }) {
  return <div {...rest} data-active={isActive} />;
}
```

And the security one, which is the only genuinely dangerous escape hatch in JSX:

```jsx
// React escapes all interpolated text by default — this is safe:
<div>{userInput}</div>              // <script> arrives as literal text

// This is not — it bypasses escaping entirely:
<div dangerouslySetInnerHTML={{ __html: userInput }} />
```

The API is deliberately ugly to make review catch it. If you must render user-authored HTML, sanitise it first with something like [DOMPurify](https://github.com/cure53/DOMPurify) — the escaping React gives you for free everywhere else is a large part of why XSS is rare in React apps.

[⬆ back to top](#table-of-contents)

---

## 12. Mastery Grab Bag

Short entries — each one is a JS feature that explains a React idiom you will otherwise take on faith.

### `Symbol.iterator` — what "iterable" means

The spread operator, `for...of`, and array destructuring all work on anything with a `Symbol.iterator` method. That is why spread works on strings, `Set`, `Map`, and `NodeList`, but not on plain objects.

```js
[..."abc"];                              // ["a","b","c"]
[...new Set([1,1,2])];                   // [1,2] — the standard dedupe idiom
[...document.querySelectorAll("li")];    // real array from a NodeList
[...{ a: 1 }];                           // TypeError — objects aren't iterable
```

`Object.keys(obj)` exists precisely because objects aren't iterable. And note that `Array.from` takes a mapping function, so `Array.from({ length: 5 }, (_, i) => i)` builds `[0,1,2,3,4]` — the standard "render N placeholders" trick for skeleton loaders.

### `Set` and `Map` — the right data structures for lookups

```js
// ❌ O(n) per check inside a render — quadratic over a list
{items.map(i => <Row selected={selectedIds.includes(i.id)} />)}

// ✅ O(1) per check
const selected = new Set(selectedIds);
{items.map(i => <Row selected={selected.has(i.id)} />)}
```

`Map` also permits non-string keys and preserves insertion order reliably. Objects coerce every key to a string, so `obj[1]` and `obj["1"]` are the same slot — a bug source when ids arrive as numbers from one endpoint and strings from another.

Both are mutable, so treat them the same as arrays in state: replace, don't mutate.

```js
setSelected(prev => new Set(prev).add(id));     // new Set → new reference
```

### `WeakMap` — caching keyed by object without leaking

A `WeakMap` holds its keys weakly, so an entry disappears when the key object is garbage collected. This is how you attach metadata to objects you don't own — memoization caches keyed on a props object, for instance — without pinning them in memory forever.

```js
const cache = new WeakMap();
function expensive(obj) {
  if (cache.has(obj)) return cache.get(obj);
  const result = compute(obj);
  cache.set(obj, result);
  return result;                         // entry vanishes when `obj` does
}
```

### Generators — the shape behind async iteration

You will rarely write one, but they explain streaming APIs and `for await`:

```js
function* range(n) { for (let i = 0; i < n; i++) yield i; }
[...range(3)];                           // [0,1,2] — generators are iterable

// Streaming a fetch response body, line by line:
for await (const chunk of response.body) { /* ... */ }
```

Note that React Compiler does not optimize generator functions or async iterators — one of its documented bail-outs.

### Labelled template literals and `String.raw`

Tagged templates are how `styled-components` and `graphql` work — a function call with an odd syntax:

```js
function tag(strings, ...values) { /* strings is an array of the literal parts */ }
tag`hello ${name}`;                      // tag(["hello ", ""], name)
```

Nothing magic; useful to recognise so `css\`...\`` doesn't look like new syntax.

### `structuredClone`, `Object.freeze`, `Proxy`

Covered in Part 2 — listed here because they are the three "I need to debug a mutation" tools and they belong in your mental index together.

### Numeric separators, `BigInt`, and `Intl`

```js
const TIMEOUT = 5_000;                   // numeric separators, ES2021 — pure readability
new Intl.NumberFormat("en-US", { style: "currency", currency: "USD" }).format(1234.5);
new Intl.RelativeTimeFormat("en").format(-3, "day");     // "3 days ago"
new Intl.DateTimeFormat("en-GB", { dateStyle: "medium" }).format(new Date());
```

[`Intl`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl) is built into every runtime and removes the reflexive reach for `moment`/`date-fns` for formatting. `Intl.NumberFormat` instances are expensive to construct — build one and reuse it, rather than constructing inside a `.map()`.

[⬆ back to top](#table-of-contents)

---

## 13. Cheat Sheets

### Immutable update patterns

| Operation | Immutable form |
|---|---|
| Add to end | `[...arr, item]` |
| Add to start | `[item, ...arr]` |
| Insert at `i` | `arr.toSpliced(i, 0, item)` |
| Remove at `i` | `arr.toSpliced(i, 1)` |
| Remove by id | `arr.filter(x => x.id !== id)` |
| Replace at `i` | `arr.with(i, item)` |
| Update by id | `arr.map(x => x.id === id ? { ...x, ...patch } : x)` |
| Sort | `arr.toSorted(fn)` |
| Reverse | `arr.toReversed()` |
| Object field | `{ ...obj, key: value }` |
| Object dynamic field | `{ ...obj, [key]: value }` |
| Remove object field | `const { [key]: _, ...rest } = obj` |
| Nested field | `{ ...o, a: { ...o.a, b: v } }` |
| Deep copy | `structuredClone(obj)` |
| Set add | `new Set(prev).add(x)` |
| Map set | `new Map(prev).set(k, v)` |

### Equality and nullish quick reference

| Expression | Result | Note |
|---|---|---|
| `{} === {}` | `false` | different objects |
| `NaN === NaN` | `false` | use `Object.is` or `Number.isNaN` |
| `Object.is(NaN, NaN)` | `true` | React's comparison |
| `Object.is(0, -0)` | `false` | only other `===` divergence |
| `0 \|\| 10` | `10` | falsy fallback — usually a bug |
| `0 ?? 10` | `0` | nullish fallback — usually correct |
| `null ?? "x"` | `"x"` | |
| `{ a = 1 }` from `{ a: null }` | `null` | defaults fire on `undefined` only |
| `a ?? b \|\| c` | SyntaxError | parenthesise |

### Falsy values (all eight)

`false` · `0` · `-0` · `0n` · `""` · `null` · `undefined` · `NaN`

Everything else is truthy — including `[]`, `{}`, `"0"`, `"false"`, and `function(){}`.

### Array method returns

| Method | Returns | Mutates? |
|---|---|---|
| `map` | new array, same length | no |
| `filter` | new array, ≤ length | no |
| `find` / `findLast` | element or `undefined` | no |
| `findIndex` | index or `-1` | no |
| `some` / `every` | boolean | no |
| `reduce` | anything | no |
| `flatMap` | new flattened array | no |
| `at(-1)` | element | no |
| `slice` | new array | no |
| `toSorted` / `toReversed` / `toSpliced` / `with` | new array | no |
| `sort` / `reverse` / `splice` | the same array | **yes** |
| `push` / `pop` / `shift` / `unshift` | new length / element | **yes** |
| `includes` / `indexOf` | boolean / index | no |
| `join` | string | no |

### Promise combinators

| | Resolves with | Fails when |
|---|---|---|
| `Promise.all(ps)` | array of values | any rejects |
| `Promise.allSettled(ps)` | `{status, value\|reason}[]` | never |
| `Promise.race(ps)` | first settled value | first settled rejects |
| `Promise.any(ps)` | first fulfilled value | all reject |
| `Promise.try(fn)` | fn's value | fn throws or rejects |

### Async in effects — the checklist

```js
useEffect(() => {
  const controller = new AbortController();
  (async () => {
    try {
      const res = await fetch(url, { signal: controller.signal });
      if (!res.ok) throw new Error(`HTTP ${res.status}`);   // fetch doesn't throw on 4xx/5xx
      setData(await res.json());
    } catch (e) {
      if (e.name === "AbortError") return;                  // expected on cleanup
      setError(e);
    }
  })();
  return () => controller.abort();                          // cancel on deps change/unmount
}, [url]);
```

- [ ] `res.ok` checked
- [ ] `AbortError` filtered out
- [ ] cleanup aborts
- [ ] the effect function itself is **not** `async` (it must return a cleanup function, not a promise)
- [ ] every value read from the render scope is in the dependency array, or accessed via `useEffectEvent` / an updater function

### DOM event essentials

| Need | API |
|---|---|
| Element that was clicked | `e.target` |
| Element the handler is on | `e.currentTarget` |
| Cancel default behaviour | `e.preventDefault()` |
| Stop bubbling | `e.stopPropagation()` |
| Nearest matching ancestor | `el.closest(sel)` |
| `data-*` attributes | `el.dataset.foo` |
| Auto-removing listener | `addEventListener(t, fn, { signal })` |
| Form → object | `Object.fromEntries(new FormData(form))` |
| Multi-value field | `formData.getAll(name)` |
| Query string → object | `Object.fromEntries(new URLSearchParams(location.search))` |

### Version/practice changes worth knowing (as of September 2026)

| Old advice | Current | Since |
|---|---|---|
| `import React from "react"` in every JSX file | not needed (automatic JSX runtime) | React 17 |
| `[...arr].sort()` to sort immutably | `arr.toSorted()` | ES2023 |
| `JSON.parse(JSON.stringify(x))` to deep clone | `structuredClone(x)` | Node 17 / all modern browsers |
| `obj.hasOwnProperty(k)` | `Object.hasOwn(obj, k)` | ES2022 |
| hand-rolled `reduce` to group | `Object.groupBy` / `Map.groupBy` | ES2024 |
| batching only in React event handlers | automatic batching everywhere | React 18 |
| `useMemo`/`useCallback` by default | React Compiler; manual memo as escape hatch | Compiler 1.0, Oct 2025 |
| `useEvent` / experimental `useEffectEvent` | `useEffectEvent`, stable | React 19.2, Oct 2025 |
| `moment` for formatting | `Intl.*` | built in |

[⬆ back to top](#table-of-contents)

---

## 14. Suggested Learning / Reference Order

**If you have one evening before touching React**, read these in order and skip everything else:

1. [Part 2 — References, Equality & Immutability](#2-references-equality--immutability) (all tiers)
2. [Part 3 — Functions, Closures & `this`](#3-functions-closures--this) (Beginner → Advanced; skip the `this` section)
3. [Part 5 — Arrays as the Rendering Toolkit](#5-arrays-as-the-rendering-toolkit) (Beginner → Advanced)
4. [Part 7 — Truthiness & Conditional Rendering](#7-truthiness--conditional-rendering) (all — it's short)

Those four cover the JS behind an estimated majority of React beginner bugs.

**If you have a week**, add in this order:

5. [Part 4 — Destructuring, Spread & Rest](#4-destructuring-spread--rest) — you will read this syntax in every component before you write it
6. [Part 6 — Objects: Dynamic Keys, Optional Chaining, Nullish Coalescing](#6-objects-dynamic-keys-optional-chaining-nullish-coalescing) — forms and API data
7. [Part 9 — Asynchronous JavaScript in a UI](#9-asynchronous-javascript-in-a-ui) (Beginner → Advanced) — stop at the event loop for now
8. [Part 11 — JSX Is Just JavaScript](#11-jsx-is-just-javascript) (Working Knowledge) — demystifies the syntax before it becomes cargo cult
9. [Part 1 — How React Changes What "Good JS" Means](#1-how-react-changes-what-good-js-means) — re-read; it lands differently once 2/3/5 are internalised

**Come back to these when the matching symptom appears:**

| Symptom | Go to |
|---|---|
| "My state changed but nothing re-rendered" | Part 2 |
| "My interval/listener sees an old value" | Part 3, Advanced |
| "Typing in one row affects another row" | Part 5, Advanced |
| "A literal `0` appeared in my UI" | Part 7, Advanced |
| "Search results show the wrong query's data" | Part 9, Advanced |
| "My modal closes when I click inside it" | Part 8 |
| "My lazy-loaded chunk is huge" | Part 10, Advanced |
| "Console warns about unknown DOM attribute" | Part 11, Mastery |
| "Everything is slow with a large list" | Part 5 Mastery, Part 12 (`Set`/`Map`) |

**Read once for depth, when you're comfortable:** [Part 9's event loop section](#mastery--the-event-loop-microtasks-and-why-batching-looks-odd), [Part 10 Mastery](#mastery--esm-vs-commonjs-and-where-you-still-meet-cjs), and [Part 12](#12-mastery-grab-bag). None are prerequisites; all three convert "React does something weird here" into "of course it does."

**Then go learn React itself.** Start with [react.dev's Learn section](https://react.dev/learn) — it is genuinely excellent and assumes exactly the JavaScript this document covers. If TypeScript is also on the list, [`javascript-for-typescript-mastery.md`](./javascript-for-typescript-mastery.md) overlaps this doc on Parts 2–6 and diverges usefully on types and coercion.

[⬆ back to top](#table-of-contents)

---

## 15. Quick Self-Check

Answer these out loud. If you can explain the *mechanism*, not just the rule, you're ready.

**References & immutability**

1. Why does `todos.push(newTodo)` fail to update the UI, even though the array genuinely contains the new item?
2. What are the only two cases where `Object.is` and `===` disagree?
3. `{ ...state }` is a copy — of what, exactly? What did it *not* copy?
4. Why is `structuredClone` preferable to `JSON.parse(JSON.stringify(x))`? Name three things the JSON round-trip destroys.
5. Why is the immutable style *faster* than mutation for a memoized list, not slower?

**Closures**

6. Explain the stale-closure bug in terms of function calls and variable bindings — without using the word "React."
7. Why does `setCount(c => c + 1)` fix a stale closure when `setCount(count + 1)` doesn't?
8. Why is a function defined inside a component body a different reference on every render, and where does that matter?
9. What does `useEffectEvent` do that a `useCallback` with an empty dependency array does not?

**Arrays & keys**

10. When is an index key actually fine, and what specific event makes it break?
11. Why does `crypto.randomUUID()` as a key destroy your list?
12. Why is `reduce((acc, x) => ({ ...acc, [x.id]: x }), {})` a performance bug, and what's the linear alternative?

**Operators & rendering**

13. `{items.length && <List />}` renders a literal `0`. Trace exactly why, naming what `&&` returns and what React does with it.
14. Give a concrete case where `||` produces a wrong value and `??` produces the right one.
15. Why is `a ?? b || c` a SyntaxError rather than just confusing?
16. `const f = () => { id: 1 }` returns what, and why?

**Events**

17. `e.target` vs. `e.currentTarget` — give a case where they differ.
18. Which of `preventDefault` / `stopPropagation` stops a form from reloading the page? What does the other one do?
19. Why does event delegation work for elements that didn't exist when the listener was attached?

**Async**

20. Does `fetch` reject on a 500? What must you check, and what confusing error do you get if you don't?
21. Draw the timeline of the search race condition. What is the *fix*, and why does the effect's cleanup function happen to be the right place for it?
22. Why must you filter out `AbortError`, and how do you detect it?
23. Predict the output: `console.log(1); setTimeout(()=>console.log(2)); Promise.resolve().then(()=>console.log(3)); console.log(4);`
24. Why does `console.log(count)` immediately after `setCount(count + 1)` print the old value? (The answer is about closures, not about "setState is async.")

**Modules & JSX**

25. What does `<Button />` compile to, and why does `<button />` compile to something different?
26. Why is `className` not `class`?
27. Why does a barrel file undermine code splitting, and when is a barrel fine?
28. What does `React.lazy` require of the module it loads, and how do you adapt a named export?
29. Modules are singletons — give one case where that's useful and one where it's a bug.

[⬆ back to top](#table-of-contents)

---

*Last verified: September 2026 — against React 19.2.x, React Compiler 1.0, ES2025, and Node 22/24 LTS.*
