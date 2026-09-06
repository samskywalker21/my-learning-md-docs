# JavaScript — Errors, Async & Modules (Parts 8–10)

Throwing and catching; the event loop, Promises, `async`/`await` and cancellation; ES modules. These three sit together because in practice they arrive together — the first async bug you debug will involve all of them.

> **About this document.** Part of the [JavaScript Mastery Guide](./javascript-mastery-guide.md) set — see that document's "About This Document" section for the framing (curiosity-driven, hands-on), scope, tier conventions, sourcing rule and update instructions. Written against ECMAScript 2026; Baseline data verified September 6, 2026.

**Prerequisites:** [Part 4 — Functions & Closures](./javascript-functions-closures.md) and [Part 6 — Iteration](./javascript-arrays-iteration.md) (generators, for the Mastery tier of Part 9).

---

## Table of Contents

1. [Part 8 — Errors & Exceptions](#1-part-8--errors--exceptions)
   - [Beginner](#part-8-beginner) · [Working Knowledge](#part-8-working) · [Advanced](#part-8-advanced) · [Mastery](#part-8-mastery) · [Cheat sheet](#part-8-cheatsheet)
2. [Part 9 — Asynchronous JavaScript](#2-part-9--asynchronous-javascript)
   - [Beginner](#part-9-beginner) · [Working Knowledge](#part-9-working) · [Advanced](#part-9-advanced) · [Mastery](#part-9-mastery) · [Cheat sheet](#part-9-cheatsheet)
3. [Part 10 — Modules](#3-part-10--modules)
   - [Beginner](#part-10-beginner) · [Working Knowledge](#part-10-working) · [Advanced](#part-10-advanced) · [Cheat sheet](#part-10-cheatsheet)

---

## 1. Part 8 — Errors & Exceptions

<a id="part-8"></a>

### Beginner — throw, try, catch, finally

<a id="part-8-beginner"></a>

```js
function divide(a, b) {
  if (b === 0) {
    throw new Error("Cannot divide by zero");
  }
  return a / b;
}

try {
  divide(1, 0);
} catch (err) {
  console.error(err.message);      // "Cannot divide by zero"
} finally {
  console.log("always runs");      // even on throw, even on return
}
```

You can throw any value, but **always throw an `Error`**. Only `Error` instances carry a stack trace, and every tool in the ecosystem — loggers, monitors, `console.error` — assumes one.

```js
// ❌ Throwing a string: no stack, and every catch block has to guess the shape.
throw "Something went wrong";

// ✅ Throwing an Error.
throw new Error("Something went wrong");
```

Since ES2019 the `catch` binding is optional, which is occasionally exactly right:

```js
try {
  return JSON.parse(input);
} catch {                        // we genuinely do not care why
  return null;
}
```

#### The built-in error types

| Type | Thrown when |
|---|---|
| `Error` | Generic; the base class |
| `TypeError` | Wrong type, or an operation on an incompatible value (`null.foo`) |
| `RangeError` | A value is outside its legal range (`arr.length = -1`) |
| `SyntaxError` | Invalid syntax — usually from `JSON.parse` at runtime |
| `ReferenceError` | An undeclared name, or TDZ access |
| `URIError` | Malformed URI in `decodeURIComponent` etc. |
| `AggregateError` | Multiple errors at once — thrown by `Promise.any` |

```js
null.foo;                       // TypeError
undefinedVar;                   // ReferenceError
JSON.parse("{bad}");            // SyntaxError
new Array(-1);                  // RangeError
decodeURIComponent("%");        // URIError
```

Throwing the *right* type matters: `TypeError` for a bad argument type, `RangeError` for a bad value, `Error` (or a subclass) for domain failures.

### Working Knowledge — custom errors, `cause`, and what not to catch

<a id="part-8-working"></a>

#### Custom error classes

```js
class ValidationError extends Error {
  constructor(message, field, options) {
    super(message, options);         // options passes { cause } through
    this.name = "ValidationError";   // ← required; see below
    this.field = field;
  }
}

class NotFoundError extends Error {
  constructor(resource, id) {
    super(`${resource} ${id} not found`);
    this.name = "NotFoundError";
    this.resource = resource;
    this.id = id;
  }
}

try {
  throw new ValidationError("Email is invalid", "email");
} catch (err) {
  if (err instanceof ValidationError) {
    console.log(err.field);          // "email"
    console.log(`${err}`);           // "ValidationError: Email is invalid"
  }
}
```

```js
// ❌ Forgetting to set `name` — logs say "Error", not your class.
class ValidationError extends Error {}
`${new ValidationError("bad")}`;      // "Error: bad"

// ✅ Set it explicitly.
class ValidationError extends Error {
  constructor(msg) { super(msg); this.name = "ValidationError"; }
}
`${new ValidationError("bad")}`;      // "ValidationError: bad"
```

`name` is inherited from `Error.prototype.name`, which is the string `"Error"` — subclassing does not change it. Setting it in the constructor is not optional boilerplate; it is what makes your logs readable.

#### `cause` — chaining without losing the original

`Error`'s second argument ([ES2022, Baseline widely available](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/cause)) attaches the underlying error.

```js
// ❌ Re-throwing loses the original stack and message entirely.
try {
  await db.query(sql);
} catch (err) {
  throw new Error("Failed to load user");    // why? gone.
}

// ✅ Preserve the chain.
try {
  await db.query(sql);
} catch (err) {
  throw new Error("Failed to load user", { cause: err });
}
```

```js
// Walking the chain when logging.
function describe(err) {
  const chain = [];
  for (let e = err; e; e = e.cause) {
    chain.push(`${e.name}: ${e.message}`);
  }
  return chain.join("\n  caused by: ");
}
```

Node's `console.error` and most modern loggers print the cause chain automatically. Adopt `cause` everywhere you re-throw — it converts "something failed in the user service" into an actionable trace at zero cost.

#### What not to catch

```js
// ❌ Catching everything and returning a sentinel hides bugs.
function getUser(id) {
  try {
    return db.users.find(id);
  } catch {
    return null;      // was it "not found", or did the DB connection die?
  }
}

// ✅ Catch what you can handle; let the rest propagate.
function getUser(id) {
  try {
    return db.users.find(id);
  } catch (err) {
    if (err instanceof NotFoundError) return null;
    throw err;                             // programmer errors keep travelling
  }
}
```

The distinction worth holding onto: **operational errors** (network down, file missing, invalid user input) are expected and should be handled. **Programmer errors** (`TypeError`, `ReferenceError`) are bugs — catching them converts a loud, fixable crash into silent corruption.

> **Try It.**
> ```js
> class DbError extends Error {
>   constructor(m, o) { super(m, o); this.name = "DbError"; }
> }
> try {
>   try { JSON.parse("{oops}"); }
>   catch (e) { throw new DbError("query failed", { cause: e }); }
> } catch (e) {
>   console.log(e.name, "|", e.message, "|", e.cause.name);
> }
> ```
> Expected: `DbError | query failed | SyntaxError`. The original error survived the re-throw.

### Advanced — `finally` traps and error boundaries

<a id="part-8-advanced"></a>

`finally` runs no matter what — including when the `try` or `catch` block returns. That gives it the power to override a return value, which is almost always a bug:

```js
// ❌ finally's return silently discards both the value and the exception.
function f() {
  try {
    return "from try";
  } finally {
    return "from finally";      // wins
  }
}
f();      // "from finally"

function g() {
  try {
    throw new Error("boom");
  } finally {
    return "swallowed";         // the exception vanishes entirely
  }
}
g();      // "swallowed" — no error propagates
```

```js
// ✅ Use finally only for cleanup. Never return or throw from it.
function f() {
  const conn = open();
  try {
    return conn.query();
  } finally {
    conn.close();
  }
}
```

Linters flag `no-unsafe-finally` for exactly this reason.

Another trap: a `throw` inside `catch` replaces the original error unless you chain it with `cause`, and a `throw` inside `finally` replaces whatever was in flight — with no `cause` mechanism at all.

#### Errors across async boundaries

A `try`/`catch` only catches synchronously thrown errors in its own execution. A callback that throws later is outside it entirely:

```js
// ❌ The catch never runs — the callback executes on a later turn.
try {
  setTimeout(() => { throw new Error("boom"); }, 0);
} catch (err) {
  console.log("never reached");
}
// The error becomes an uncaught exception and can crash the process.

// ✅ Handle inside the callback.
setTimeout(() => {
  try { risky(); } catch (err) { report(err); }
}, 0);
```

The same applies to a non-awaited promise. `try`/`catch` around async code works only when you `await` inside the `try` — covered in [Part 9](#part-9-working).

#### Process-level safety nets

```js
// Node
process.on("uncaughtException", (err, origin) => {
  logger.fatal({ err, origin });
  process.exit(1);              // ← exit. The process state is not trustworthy.
});
process.on("unhandledRejection", (reason) => {
  logger.fatal({ reason });
  process.exit(1);
});

// Browser
window.addEventListener("error", (e) => report(e.error));
window.addEventListener("unhandledrejection", (e) => report(e.reason));
```

These are for **logging and orderly shutdown**, not recovery. After an uncaught exception the program is in an undefined state — [Node's own documentation](https://nodejs.org/api/process.html#event-uncaughtexception) says resuming is not safe. Log, flush, exit, and let your supervisor restart the process.

Since Node 15, an unhandled rejection **terminates the process by default** — the older behaviour, a warning you could ignore, is gone. Code and blog posts written before that assumption changed are a common source of surprise.

### Mastery — `Error.isError`, stack traces, and `AggregateError`

<a id="part-8-mastery"></a>

#### Detecting errors reliably

```js
err instanceof Error;               // fails across realms (iframe, vm, worker)
Object.prototype.toString.call(err) === "[object Error]";  // realm-safe, verbose
Error.isError(err);                 // ES2026 — the correct check
```

`Error.isError` performs a branded check for the private field the `Error` constructor installs, so it accepts cross-realm errors and `DOMException`, and rejects impostors ([MDN: Error.isError](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/isError)):

```js
Error.isError(new TypeError());              // true
Error.isError({ __proto__: Error.prototype }); // false — instanceof says true
```

**Availability caveat:** MDN currently reports `Error.isError` as **limited availability — not Baseline**, because it does not yet work in some widely used browsers. Verify support before relying on it in browser code; in current Node it is fine.

#### Stack traces

```js
const err = new Error("boom");
err.stack;        // "Error: boom\n    at fn (file.js:3:9)\n    at ..."
```

`stack` is **not standardised** — its format differs between engines and it can be truncated. Never parse it; log it.

V8-specific controls, useful in Node:

```js
Error.stackTraceLimit = 50;                     // default is 10
Error.captureStackTrace(this, MyError);         // omit the constructor frame
```

```js
class AppError extends Error {
  constructor(msg, options) {
    super(msg, options);
    this.name = "AppError";
    if (Error.captureStackTrace) {              // guard: V8 only
      Error.captureStackTrace(this, AppError);  // trace starts at the caller
    }
  }
}
```

#### `AggregateError`

```js
try {
  await Promise.any([fetchA(), fetchB(), fetchC()]);
} catch (err) {
  if (err instanceof AggregateError) {
    console.log(err.errors);        // an array of every rejection
  }
}

// You can construct one yourself for batch validation.
const failures = fields.map(validate).filter(Boolean);
if (failures.length) {
  throw new AggregateError(failures, `${failures.length} fields invalid`);
}
```

`Promise.any` rejects with an `AggregateError` only when *every* input rejects — a distinct signal from `Promise.all`, which rejects with the first single error.

### Part 8 quick reference

<a id="part-8-cheatsheet"></a>

| Task | Do this |
|---|---|
| Signal a failure | `throw new Error(msg)` — never a string |
| Preserve the original | `new Error(msg, { cause: err })` |
| Name a custom error | `this.name = "MyError"` in the constructor |
| Ignore the reason | `catch { … }` (no binding) |
| Clean up | `finally` — never `return`/`throw` from it |
| Detect an error value | `Error.isError(v)` (limited availability); else `instanceof` |
| Multiple failures | `AggregateError` |
| Catch async errors | `await` inside the `try` |
| Last-resort logging | `uncaughtException` / `unhandledRejection` → log, then exit |

| Error type | Meaning |
|---|---|
| `TypeError` | Wrong type / incompatible operation |
| `RangeError` | Value out of range |
| `ReferenceError` | Undeclared name or TDZ access |
| `SyntaxError` | Invalid syntax (`JSON.parse`) |
| `AggregateError` | Several errors together |

[↑ Back to top](#table-of-contents)

---

## 2. Part 9 — Asynchronous JavaScript

<a id="part-9"></a>

### Beginner — why async exists, and the event loop

<a id="part-9-beginner"></a>

JavaScript runs your code on **one thread**. If that thread is busy, nothing else happens — no rendering, no other callbacks, no incoming requests. So anything slow (network, disk, timers) is handed to the host, which notifies you when it finishes.

```text
┌──────────────────────────────────────────────────────────────┐
│                         CALL STACK                           │
│  (one thread — only one frame executes at a time)            │
│    ┌──────────────┐                                          │
│    │ inner()      │                                          │
│    │ outer()      │                                          │
│    │ main()       │                                          │
│    └──────────────┘                                          │
└───────────────┬──────────────────────────────────────────────┘
                │ hands slow work to the host
                ▼
┌──────────────────────────────────────────────────────────────┐
│  HOST APIs — timers, network, file I/O (may use real threads)│
└───────────────┬──────────────────────────────────────────────┘
                │ when finished, queues a callback
                ▼
┌─────────────────────────┐      ┌───────────────────────────┐
│   MICROTASK QUEUE       │      │   MACROTASK QUEUE         │
│   promise callbacks,    │      │   setTimeout, setInterval,│
│   queueMicrotask,       │      │   I/O, UI events          │
│   async continuations   │      │                           │
└───────────┬─────────────┘      └─────────────┬─────────────┘
            │                                   │
            └──────────┬────────────────────────┘
                       ▼
              ┌─────────────────┐
              │   EVENT LOOP    │  Runs ONE macrotask, then drains the
              │                 │  ENTIRE microtask queue, then repeats.
              └─────────────────┘
```

The rule that explains almost every ordering puzzle: **after each macrotask, the event loop drains the whole microtask queue before taking the next macrotask.** Promise callbacks are microtasks; `setTimeout` callbacks are macrotasks. Microtasks always win.

```js
console.log("1");
setTimeout(() => console.log("2"), 0);      // macrotask
Promise.resolve().then(() => console.log("3"));  // microtask
console.log("4");

// 1, 4, 3, 2
```

> **Try It.** Run the snippet above and confirm `1 4 3 2`. Then add `queueMicrotask(() => console.log("5"))` after the `setTimeout` line. Expected: `1 4 3 5 2` — both microtasks run before the timer, in queue order.

A microtask that queues another microtask is processed in the *same* drain, which means an infinite microtask loop starves the event loop completely:

```js
// ❌ Freezes the page/process — the microtask queue never empties.
function spin() { Promise.resolve().then(spin); }

// ✅ A macrotask yields between iterations.
function spin() { setTimeout(spin, 0); }
```

### Working Knowledge — Promises and `async`/`await`

<a id="part-9-working"></a>

A **Promise** is an object representing a value that is not available yet. It is in one of three states, and once it settles it never changes again:

```text
        ┌─────────────┐
        │   PENDING   │
        └──────┬──────┘
         ┌─────┴─────┐
         ▼           ▼
   ┌──────────┐  ┌──────────┐
   │ FULFILLED│  │ REJECTED │   ← both are final; a promise settles once
   │ (value)  │  │ (reason) │
   └──────────┘  └──────────┘
```

```js
const p = new Promise((resolve, reject) => {
  setTimeout(() => resolve("done"), 1000);
});

p.then(v => console.log(v))
 .catch(e => console.error(e))
 .finally(() => console.log("cleanup"));
```

You rarely write `new Promise` — only when wrapping a callback-based API that has no promise version:

```js
// ❌ The "explicit construction antipattern": wrapping something already a promise.
new Promise((resolve, reject) => {
  fetch(url).then(resolve, reject);
});

// ✅ It is already a promise.
fetch(url);
```

#### `async`/`await`

```js
async function loadUser(id) {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}

try {
  const user = await loadUser(1);
} catch (err) {
  console.error(err);
}
```

Three rules that cover most usage:

1. An `async` function **always** returns a promise, whatever you return inside it.
2. `await` pauses that function only — the rest of the program keeps running.
3. `throw` inside an `async` function rejects its promise; `await` re-throws a rejection so `try`/`catch` works normally.

#### Sequential vs. concurrent — the highest-impact async decision

```js
// ❌ Sequential: 3 × 1s = 3s. Nothing here depends on anything else.
const a = await fetchA();     // 1s
const b = await fetchB();     // 1s
const c = await fetchC();     // 1s

// ✅ Concurrent: ~1s. All three start immediately.
const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);
```

```js
// ❌ Awaiting in a loop — N sequential round-trips.
const users = [];
for (const id of ids) {
  users.push(await fetchUser(id));
}

// ✅ Concurrent.
const users = await Promise.all(ids.map(id => fetchUser(id)));

// ✅ Concurrent, but bounded — unbounded Promise.all on 10,000 ids
//    will exhaust sockets or trip rate limits.
async function mapLimit(items, limit, fn) {
  const results = [];
  const executing = new Set();
  for (const item of items) {
    const p = Promise.resolve().then(() => fn(item));
    results.push(p);
    executing.add(p);
    p.finally(() => executing.delete(p));
    if (executing.size >= limit) await Promise.race(executing);
  }
  return Promise.all(results);
}
await mapLimit(ids, 10, fetchUser);
```

Sequential `await` in a loop is correct when each iteration depends on the previous one, or when you are deliberately rate-limiting. Otherwise it is the most common performance bug in async JavaScript.

#### The combinators

| Method | Resolves when | Rejects when | Result |
|---|---|---|---|
| `Promise.all` | All fulfil | **First** rejection (fails fast) | Array of values |
| `Promise.allSettled` | All settle | Never | Array of `{status, value\|reason}` |
| `Promise.race` | First **settles** | First settles as a rejection | That one outcome |
| `Promise.any` | First **fulfils** | All reject | First value, or `AggregateError` |

```js
// all — fails fast; the other requests are still in flight but ignored.
const [u, p] = await Promise.all([fetchUser(), fetchPosts()]);

// allSettled — you want every outcome, successes and failures alike.
const results = await Promise.allSettled(ids.map(fetchUser));
const ok   = results.filter(r => r.status === "fulfilled").map(r => r.value);
const bad  = results.filter(r => r.status === "rejected").map(r => r.reason);

// race — timeouts.
const result = await Promise.race([
  fetchData(),
  new Promise((_, rej) => setTimeout(() => rej(new Error("timeout")), 5000)),
]);

// any — first success from redundant mirrors.
const data = await Promise.any([fetchFrom(m1), fetchFrom(m2), fetchFrom(m3)]);
```

Important: `Promise.all` rejecting **does not cancel** the other operations. They continue running and their results are discarded. Cancellation requires `AbortController` — see [Advanced](#part-9-advanced).

#### `forEach` and async

```js
// ❌ forEach ignores the returned promises. "done" prints first, errors are unhandled.
ids.forEach(async (id) => { await save(id); });
console.log("done");

// ✅ Sequential.
for (const id of ids) { await save(id); }

// ✅ Concurrent.
await Promise.all(ids.map(id => save(id)));
```

`forEach` has no way to communicate an async result to its caller — its callback's return value is discarded by design. This applies to `map` only in the sense that `map` gives you the promises back, which is exactly why `Promise.all(arr.map(...))` is the idiom.

> **Try It.** Time both versions:
> ```js
> const sleep = (ms) => new Promise(r => setTimeout(r, ms));
> console.time("seq");
> for (const i of [1,2,3]) await sleep(300);
> console.timeEnd("seq");
> console.time("par");
> await Promise.all([1,2,3].map(() => sleep(300)));
> console.timeEnd("par");
> ```
> Expected: roughly `seq: 900ms` and `par: 300ms`. Save as `.mjs` for top-level `await`.

### Advanced — rejection handling, cancellation, and ordering

<a id="part-9-advanced"></a>

#### Unhandled rejections

```js
// ❌ Fire-and-forget: if this rejects, the process crashes (Node 15+).
saveAnalytics(event);

// ✅ Explicitly decide it does not matter.
saveAnalytics(event).catch(err => logger.warn({ err }, "analytics failed"));
```

A rejection is "unhandled" if no handler is attached **by the end of the current microtask drain**. That timing rule produces a subtle trap:

```js
// ❌ Attaching a handler after an await — the rejection is reported first.
const p = mightReject();
await somethingSlow();        // microtask checkpoint passes here
p.catch(handle);              // too late; already reported as unhandled

// ✅ Attach immediately, then await.
const p = mightReject().catch(handle);
await somethingSlow();
await p;
```

`.catch(fn)` is exactly `.then(undefined, fn)`, and position matters:

```js
// ❌ The catch is before the then — it cannot see errors from the then callback.
fetchData().catch(handle).then(process);

// ✅ Catch at the end.
fetchData().then(process).catch(handle);

// A second argument to then handles the SOURCE only, not the first callback:
fetchData().then(process, handleFetchError).catch(handleProcessError);
```

#### Cancellation with `AbortController`

JavaScript promises are not cancellable. `AbortController` is the standard cooperative-cancellation protocol instead.

```js
const controller = new AbortController();

const res = await fetch(url, { signal: controller.signal });

// Elsewhere:
controller.abort();      // the fetch rejects with an AbortError
```

```js
// Timeout plus manual cancellation, composed.
const timeout = AbortSignal.timeout(5000);
const manual = new AbortController();
const signal = AbortSignal.any([timeout, manual.signal]);

try {
  const res = await fetch(url, { signal });
} catch (err) {
  if (err.name === "AbortError") {
    console.log("cancelled or timed out");
  } else {
    throw err;
  }
}
```

`AbortSignal.timeout()` and `AbortSignal.any()` are [Baseline](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout) and remove the hand-rolled `Promise.race` timeout entirely. Note these are *host* APIs (WHATWG DOM/Node), not ECMAScript — but they are the standard answer across all modern hosts.

Supporting cancellation in your own async function:

```js
async function poll(url, { signal } = {}) {
  while (true) {
    signal?.throwIfAborted();               // throws AbortError if cancelled
    const res = await fetch(url, { signal });
    const data = await res.json();
    if (data.ready) return data;
    await new Promise((resolve, reject) => {
      const id = setTimeout(resolve, 1000);
      signal?.addEventListener("abort", () => {
        clearTimeout(id);
        reject(signal.reason);
      }, { once: true });
    });
  }
}
```

#### Timers are a floor, not a promise

```js
setTimeout(fn, 0);      // "run fn as soon as possible after the current work"
```

The delay is a *minimum*. If the stack is busy or many microtasks are queued, the callback runs later. Browsers also clamp nested timeouts to ≥4ms after five levels of nesting, and throttle timers in background tabs to once per second or less.

```js
// ❌ Sequencing by guessing at timings.
saveData();
setTimeout(() => readData(), 100);      // hope 100ms is enough

// ✅ Sequence by awaiting.
await saveData();
readData();
```

#### `queueMicrotask` and `setTimeout(…, 0)`

```js
queueMicrotask(fn);      // runs before the next render / next macrotask
setTimeout(fn, 0);       // runs after the current macrotask AND all microtasks
```

Use `queueMicrotask` to defer within the current turn (settling state before anything observes it). Use `setTimeout(…, 0)` to yield to the host — to let the browser paint, or to break a long computation into chunks.

### Mastery — the microtask checkpoint, and how `await` works

<a id="part-9-mastery"></a>

#### Predicting execution order exactly

```js
console.log("A: sync");

setTimeout(() => console.log("B: timeout"), 0);

Promise.resolve()
  .then(() => console.log("C: microtask 1"))
  .then(() => console.log("D: microtask 2"));

(async () => {
  console.log("E: async fn body, sync until first await");
  await null;
  console.log("F: after await = a microtask");
})();

queueMicrotask(() => console.log("G: queued microtask"));

console.log("H: sync end");
```

Output:

```text
A: sync
E: async fn body, sync until first await
H: sync end
C: microtask 1
F: after await = a microtask
G: queued microtask
D: microtask 2
B: timeout
```

The reasoning, step by step:

1. All synchronous code runs first: `A`, `E` (an async function body runs synchronously up to its first `await`), `H`.
2. The microtask queue is drained in FIFO order. At the checkpoint it holds: `C`'s callback, `F`'s continuation, `G`.
3. `C` runs and *queues* `D` at the back of the queue.
4. `F` runs, then `G`, then `D` — which was appended during the same drain and is processed in the same drain.
5. Only when the queue is empty does the macrotask `B` run.

> **Try It.** Run the block above and match it against the expected output line by line. If you can explain why `D` comes after `G`, you understand microtask ordering.

#### What `await` compiles to

`await expr` is roughly:

```js
// await expr  ≈
Promise.resolve(expr).then(value => { /* resume the function body here */ });
```

So an `async` function is a state machine that suspends at each `await` and resumes in a microtask. This is exactly the generator mechanism from [Part 6](./javascript-arrays-iteration.md#part-6-advanced) — the pre-`async` community wrote runners like this:

```js
// How co/Babel implemented async before it was syntax. Recognise it in old code.
function run(genFn) {
  const it = genFn();
  return new Promise((resolve, reject) => {
    (function step(method, arg) {
      let result;
      try { result = it[method](arg); } catch (e) { return reject(e); }
      if (result.done) return resolve(result.value);
      Promise.resolve(result.value).then(
        v => step("next", v),
        e => step("throw", e),
      );
    })("next");
  });
}

run(function* () {
  const a = yield fetchA();       // `yield` where you would write `await`
  const b = yield fetchB(a);
  return b;
});
```

Two practical consequences of "`await` is a microtask":

```js
// Even awaiting a non-promise costs a microtask tick.
async function f() { await null; console.log("2"); }
f();
console.log("1");        // 1 then 2
```

```js
// ❌ An await inside a lock/critical section yields — other code can interleave.
async function increment() {
  const current = await read();   // ← another increment() can start here
  await write(current + 1);       // lost update
}
```

Async does not mean parallel, but it does mean **interleaved**. Any state read before an `await` may be stale after it. That is the async equivalent of a race condition, and the fix is the same as anywhere else: a lock, a queue, or an atomic operation at the storage layer.

#### `Promise.withResolvers` and `Promise.try`

```js
// ❌ The deferred pattern — hoisting resolve/reject out of the executor.
let resolve, reject;
const promise = new Promise((res, rej) => { resolve = res; reject = rej; });

// ✅ Promise.withResolvers — ES2024, Baseline 2024 (since March 2024).
const { promise, resolve, reject } = Promise.withResolvers();
```

Useful for bridging event-driven code — a socket message, a user click — into a promise you can `await`.

```js
// ✅ Promise.try — ES2025, Baseline 2025 (since January 2025).
// Runs fn immediately; a SYNCHRONOUS throw becomes a rejection, not an exception.
Promise.try(fn).then(handle).catch(report);

// Without it, a sync throw escapes the promise chain:
Promise.resolve().then(() => fn());   // the old workaround
```

[`Promise.withResolvers`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/withResolvers) and [`Promise.try`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/try) are both Baseline and safe to use in current runtimes.

### Part 9 quick reference

<a id="part-9-cheatsheet"></a>

| Need | Use |
|---|---|
| Independent async work | `Promise.all([...])` |
| Every outcome, failures included | `Promise.allSettled` |
| Timeout | `AbortSignal.timeout(ms)`, or `Promise.race` |
| First success from redundant sources | `Promise.any` |
| Cancellation | `AbortController` + `signal` |
| Bounded concurrency | A limit helper — never unbounded `Promise.all` on huge input |
| Deferred promise | `Promise.withResolvers()` |
| Sync-throw safety | `Promise.try(fn)` |
| Defer within this turn | `queueMicrotask` |
| Yield to the host | `setTimeout(fn, 0)` |
| Async sequence | `for await…of` |

**Ordering:** all sync code → drain the *entire* microtask queue (promises, `await` resumptions, `queueMicrotask`) → one macrotask (`setTimeout`, I/O) → drain microtasks again → repeat.

| Antipattern | Fix |
|---|---|
| `await` in a loop over independent work | `Promise.all(arr.map(…))` |
| `forEach(async …)` | `for…of` + `await`, or `Promise.all(map)` |
| `new Promise` around a promise | Use the promise directly |
| Fire-and-forget async call | `.catch(…)` explicitly |
| `.catch()` before `.then()` | Put `.catch` last |
| `setTimeout` for sequencing | `await` |
| Reading state before an `await` and writing after | Lock, queue, or atomic write |

[↑ Back to top](#table-of-contents)

---

## 3. Part 10 — Modules

<a id="part-10"></a>

> **Tier note.** Part 10 runs **Beginner → Working Knowledge → Advanced** only. Its "Mastery" material is the module-graph and live-bindings discussion inside Advanced rather than a separate tier — there is no expert layer beyond it worth separating out at this scope.

### Beginner — import and export

<a id="part-10-beginner"></a>

```js
// math.js
export const PI = 3.14159;
export function add(a, b) { return a + b; }
export default class Calculator { }

// main.js
import Calculator, { PI, add } from "./math.js";
import { add as sum } from "./math.js";        // rename
import * as math from "./math.js";             // namespace object
```

```js
// Re-exporting — the "barrel" file pattern.
export { add, PI } from "./math.js";
export * from "./strings.js";
export { default as Calculator } from "./math.js";
```

Named exports are the default choice: they are statically analysable, autocompletable, refactorable by tooling, and tree-shakeable. Default exports are renameable at the import site, which means the same module ends up imported under three different names across a codebase. Use a default export only when a module genuinely has one obvious subject.

Enabling ESM:

```json
// package.json
{ "type": "module" }
```

Or name files `.mjs`. In browsers: `<script type="module" src="main.js"></script>`.

### Working Knowledge — how modules differ from scripts

<a id="part-10-working"></a>

| | Classic script | ES module |
|---|---|---|
| Strict mode | Opt-in | Always |
| Top-level `this` | `globalThis` | `undefined` |
| Top-level scope | Global | Module-local |
| `var` at top level | Becomes a global property | Does not |
| Loading | Blocking | Deferred by default |
| `import`/`export` | ❌ | ✅ |
| Top-level `await` | ❌ | ✅ |
| Executed | Once per `<script>` | Once per URL, ever |
| CORS | Not required | Required for cross-origin |

**Modules execute exactly once**, no matter how many files import them. That is what makes module scope a legitimate singleton:

```js
// config.js — evaluated once; every importer shares this object.
export const config = Object.freeze({ apiUrl: process.env.API_URL });
```

Top-level `await` works in modules and makes the whole module asynchronous — every importer waits for it:

```js
// db.js
const connection = await connect(process.env.DATABASE_URL);
export default connection;
```

Convenient, and a real footgun: a slow top-level `await` deep in a dependency stalls the entire graph beneath it.

#### Dynamic import

```js
// Static — hoisted, resolved before any code runs.
import { heavy } from "./heavy.js";

// Dynamic — returns a promise, evaluated on demand.
const { heavy } = await import("./heavy.js");

// Conditional loading.
if (user.isAdmin) {
  const { AdminPanel } = await import("./admin.js");
}

// Computed specifier — impossible with a static import.
const locale = await import(`./locales/${lang}.js`);
```

Dynamic `import()` is the basis of code splitting. It is also the only way to import a module whose path is not known at build time.

#### Import attributes

```js
// ES2025 — JSON modules with an explicit type attribute.
import config from "./config.json" with { type: "json" };
const data = await import("./data.json", { with: { type: "json" } });
```

Note the syntax: **`with`**, not `assert`. The earlier `assert { type: "json" }` spelling shipped in Chrome and appears in a great many blog posts and Stack Overflow answers, but it was renamed during standardisation and is being removed. If you copy an example using `assert`, expect a syntax error in current runtimes — this is one of the clearest cases in modern JS where the popular convention is simply out of date.

### Advanced — the module graph, live bindings, and cycles

<a id="part-10-advanced"></a>

Loading a module graph happens in three distinct phases:

```text
1. CONSTRUCTION   Fetch and parse every module; find its imports;
                  recurse. Result: the full module graph. No code has run.

2. INSTANTIATION  Allocate every module's exported bindings and wire each
                  import to the exporter's binding. Memory is linked,
                  values are still uninitialised (TDZ).

3. EVALUATION     Run module bodies, depth-first post-order — deepest
                  dependency first. Each module body runs exactly once.
```

```text
main.js  ──imports──▶  a.js  ──imports──▶  c.js
   │
   └────imports──▶  b.js  ──imports──▶  c.js

Evaluation order: c.js, a.js, b.js, main.js
                  (c runs once, even though two modules import it)
```

Because imports are wired in phase 2, `import` statements are **hoisted** — they are processed before any other code in the file, regardless of where you write them.

#### Live bindings

An import is a *view* of the exporter's binding, not a copy of its value. When the exporter reassigns, importers see the new value.

```js
// counter.js
export let count = 0;
export function increment() { count++; }

// main.js
import { count, increment } from "./counter.js";
console.log(count);     // 0
increment();
console.log(count);     // 1  ← the imported binding updated
count = 5;              // ❌ TypeError: Assignment to constant variable
```

Imports are read-only bindings on the importing side and live on the exporting side. CommonJS behaves differently — `require` copies the value at the moment of the call — which is a real source of confusion when migrating:

```js
// CommonJS: a snapshot.
const { count } = require("./counter.js");   // captures 0 forever
```

#### Circular imports

Cycles are legal and resolve without error, but a module can observe a dependency **before it has been evaluated**:

```js
// a.js
import { b } from "./b.js";
export const a = "A";
console.log("a.js sees b =", b);

// b.js
import { a } from "./a.js";
export const b = "B";
console.log("b.js sees a =", a);   // ReferenceError: Cannot access 'a' before initialization
```

Entry `a.js` → evaluates `b.js` first → `b.js` reads `a`, which is instantiated but still in its TDZ → throws.

```js
// ✅ Deferring the access to call time avoids the problem entirely.
// b.js
import { getA } from "./a.js";
export const b = "B";
export function describe() { return getA(); }   // called later, after both modules ran
```

Function *declarations* are hoisted and initialised during instantiation, so a cycle that only exchanges functions works. A cycle that exchanges `const` values at module top level does not. The reliable fix is not cleverness — it is extracting the shared piece into a third module that neither imports from the other.

> **Try It.** Create the two files above and run `node a.mjs` (adjusting the import paths). Expected: `ReferenceError: Cannot access 'a' before initialization`. Then change `b.js` to use the function form and it runs clean. That contrast is the whole of "instantiation precedes evaluation".

#### ESM and CommonJS interop

| | CommonJS | ESM |
|---|---|---|
| Syntax | `require` / `module.exports` | `import` / `export` |
| Resolution | Runtime, synchronous | Static, before evaluation |
| Bindings | Value copies | Live views |
| Cycles | Partial object returned | TDZ errors, or works via hoisted functions |
| Top-level `await` | ❌ | ✅ |
| Tree-shakeable | Poorly | Yes |
| `__dirname` | ✅ | `import.meta.dirname` |

```js
// ESM equivalents of the CommonJS globals.
import.meta.url;        // "file:///path/to/module.js"
import.meta.dirname;    // the directory (Node 20.11+/21.2+)
import.meta.filename;   // the file path
```

ESM can import CommonJS (the `module.exports` object arrives as the default export). CommonJS cannot `require` ESM synchronously — use dynamic `import()`. Modern Node has been relaxing this with require-of-ESM support; check the Node docs for your exact version rather than assuming, as this area has changed repeatedly.

### Part 10 quick reference

<a id="part-10-cheatsheet"></a>

| Task | Syntax |
|---|---|
| Named export | `export const x = 1;` |
| Default export | `export default fn;` |
| Export a list | `export { a, b as c };` |
| Named import | `import { a } from "./m.js";` |
| Rename on import | `import { a as b } from "./m.js";` |
| Namespace import | `import * as m from "./m.js";` |
| Re-export | `export { a } from "./m.js";` |
| Dynamic import | `const m = await import("./m.js");` |
| JSON module | `import d from "./d.json" with { type: "json" };` |
| Current file URL | `import.meta.url` |
| Current directory | `import.meta.dirname` |
| Enable ESM in Node | `"type": "module"`, or `.mjs` |

| Rule | Why it matters |
|---|---|
| Modules evaluate once per URL | Module scope is a real singleton |
| Imports are hoisted | Import position in the file is irrelevant |
| Imports are live, read-only bindings | Reassignment in the exporter is visible; assignment by the importer throws |
| Evaluation is depth-first post-order | Dependencies run before dependents |
| Cycles hit the TDZ on `const`/`let` | Defer access into a function, or extract a third module |
| Always strict, `this === undefined` | No accidental globals |
| `with { type: "json" }` | The `assert` spelling is obsolete |

[↑ Back to top](#table-of-contents)

---

**Next:** [Parts 11–12 — Regex, Dates & Intl](./javascript-regex-dates-intl.md) · **Up:** [Mastery Guide overview](./javascript-mastery-guide.md)

*Written against ECMAScript 2026. Baseline data verified September 6, 2026.*
