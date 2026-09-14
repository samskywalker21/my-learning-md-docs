# Node.js — Runtime Foundations (Parts 1–3)

How Node turns a file on disk into a running program: the CLI surface that has quietly absorbed half your dev dependencies, the event loop that schedules everything else in this set, and the async vocabulary — callbacks, emitters, signals — that every `node:` module is written in.

> **Spec:** this doc follows the shared spec in [`nodejs-mastery-guide.md`](./nodejs-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Mastery tiers (collapsed where a tier would be filler, always stated), **Try It** exercises with expected output, wrong-vs-right pairs, official docs cited inline. Written against **Node.js 24 LTS**, verified **September 14, 2026**.
>
> **Prerequisite:** if "V8", "libuv" or "the event loop" are fuzzy terms, read the [Runtime Primer](./nodejs-mastery-guide.md#4-runtime-primer--what-node-actually-is) in the overview first. This doc assumes it.

---

## Table of Contents

- [Part 1 — Running Code](#part-1--running-code)
  - [Beginner: four ways to execute JavaScript](#p1-beginner)
  - [Working Knowledge: the flags that replaced your dev dependencies](#p1-working)
  - [Advanced: where `--watch`, `--env-file` and `--run` stop short](#p1-advanced)
  - [Mastery: flag precedence, `NODE_OPTIONS`, and startup cost](#p1-mastery)
  - [Part 1 cheat sheet](#part-1-cheat-sheet)
- [Part 2 — The Event Loop](#part-2--the-event-loop)
  - [Beginner: one thread, one stack](#p2-beginner)
  - [Working Knowledge: the six phases](#p2-working)
  - [Advanced: microtasks, `nextTick`, and starvation](#p2-advanced)
  - [Mastery: the thread pool, and measuring the loop](#p2-mastery)
  - [Part 2 cheat sheet](#part-2-cheat-sheet)
- [Part 3 — Async Patterns in Node](#part-3--async-patterns-in-node)
  - [Beginner: the three calling conventions](#p3-beginner)
  - [Working Knowledge: `EventEmitter`](#p3-working)
  - [Advanced: `AbortSignal`, and the two ways async code kills a process](#p3-advanced)
  - [Mastery: async iteration and the modern primitives](#p3-mastery)
  - [Part 3 cheat sheet](#part-3-cheat-sheet)

---

## Part 1 — Running Code

The interesting thing about Node's CLI in 2026 is how much of it is new. Watch mode, env files, a script runner, a test runner and a TypeScript loader all arrived in the last few release lines, and each one replaced a package that tutorials still tell you to install. This Part is the surface area, and the places where the built-in version is genuinely *not* a drop-in replacement.

<a id="p1-beginner"></a>

### Beginner: four ways to execute JavaScript

```bash
node app.js                      # 1. run a file
node -e "console.log(2 ** 10)"   # 2. evaluate a string
node -p "2 ** 10"                # 3. evaluate and print the result
node                             # 4. the REPL
```

`-e` evaluates; `-p` evaluates *and prints the completion value*, which makes it the fastest way to interrogate a running Node ([CLI docs](https://nodejs.org/docs/latest-v24.x/api/cli.html)):

```bash
node -p "process.version"
node -p "process.platform"
node -p "Object.keys(process.versions).join(', ')"
```

One asymmetry worth knowing early: `--print` does not support ES module syntax, so `node --input-type=module -p "..."` throws. Use `-e` with `--input-type=module` and an explicit `console.log` instead.

The REPL is more useful than it looks — `_` holds the last result, `.editor` opens a multi-line buffer, `.load file.js` pulls a file in, and `.help` lists the rest. Every built-in module is available without importing it: type `os.cpus().length` and it answers.

**Try It — the smallest possible Node program.**

```js
// hello.js
console.log('pid', process.pid, 'on', process.platform);
```

```bash
node hello.js
```

Expected output: something like `pid 48213 on darwin` (or `win32`, or `linux`). Two things just happened that are worth noticing: you did not need a `package.json`, and `process` was available without importing anything. `process`, `console`, `Buffer`, `fetch`, `URL` and the timer functions are globals; everything else comes from a `node:` module.

<a id="p1-working"></a>

### Working Knowledge: the flags that replaced your dev dependencies

#### `--watch` — restart on save

```bash
node --watch app.js
```

Node watches the file you ran and every file it imports, and restarts the process on change. It prints `Restarting app.js` between runs. To watch files that aren't in the import graph (templates, SQL, config), add them:

```bash
node --watch --watch-path=./src --watch-path=./config app.js
```

Two companion flags matter in practice: `--watch-preserve-output` stops it clearing your terminal on every restart, and `--watch-kill-signal` changes the signal sent to the old process (useful if yours needs longer than the default to shut down — see [Part 13](./nodejs-production-diagnostics.md#part-13--production-node)).

#### `--env-file` — load `.env` without a package

```bash
node --env-file=.env app.js
node --env-file=.env --env-file=.env.local app.js   # later files win
node --env-file-if-exists=.env.local app.js         # no error if missing
```

Stable since 24.10.0. The format is the one you already know, with rules worth reading rather than guessing ([CLI docs](https://nodejs.org/docs/latest-v24.x/api/cli.html)):

```bash
# a comment
PORT=3000                 # trailing comments work too
USERNAME="nodejs"         # surrounding ' " ` are stripped
export API_KEY=abc123     # a leading `export` is ignored
MULTI_LINE="THIS IS
A MULTILINE"              # becomes "THIS IS\nA MULTILINE"
```

**The rule people get wrong:** if a variable exists in the real environment *and* in the file, **the environment wins**. That is the opposite of what most people assume, and it is exactly what you want in production — your orchestrator's `DATABASE_URL` is not silently overwritten by a `.env` that got baked into the image.

#### `--run` — run a `package.json` script without npm

```bash
node --run build
node --run test -- --watch     # everything after -- goes to the script
```

This is a fast path: no npm process, no lifecycle machinery. It is stable, and it is *deliberately* not a full `npm run` replacement — see [Advanced](#p1-advanced) below for the three things it won't do.

#### Preloading: `--import` and `--require`

```bash
node --import ./instrument.mjs app.js   # ES module, runs before app.js
node --require ./instrument.cjs app.js  # CommonJS equivalent
```

This is how tracing agents, source-map hooks and module loaders attach themselves without you editing your entry point. `--import` is the modern one; `--require` predates ESM and cannot load an ES module.

#### Arguments and exit codes

```js
// args.js
console.log(process.argv);
```

```bash
node args.js --port 8080 extra
```

Expected output: an array whose first element is the path to the `node` binary, whose second is the absolute path to `args.js`, and whose remaining elements are `[ '--port', '8080', 'extra' ]`. Your arguments start at index 2 — `process.argv.slice(2)` is the idiom, and [Part 8](./nodejs-processes-workers.md#part-8--the-process) shows `util.parseArgs`, which turns that array into something you'd want to use.

To exit with a failure code, set `process.exitCode` and let the process end naturally:

```js
// ✗ wrong — kills the process immediately; pending writes to stdout can be lost
process.exit(1);

// ✓ right — sets the code, lets the loop drain, exits cleanly
process.exitCode = 1;
```

That distinction matters more than it looks: `process.exit()` does not wait for queued I/O, which is why a CLI that "works locally but loses its last line of output in CI" is usually a `process.exit()` call away from a pipe.

> **vs. Bun** — Bun's equivalents are `bun --watch` / `--hot`, `.env` loaded with no flag at all, and `bun run`. The biggest divergence is TypeScript: `bun app.ts` runs any TypeScript, while `node app.ts` strips types only (no `enum`, no parameter properties) — see [Part 4](./nodejs-modules-packages.md#part-4--modules-cjs-esm-and-typescript) and [`bun-runtime-foundations.md`](../bun/bun-runtime-foundations.md).

<a id="p1-advanced"></a>

### Advanced: where `--watch`, `--env-file` and `--run` stop short

Each of these built-ins is narrower than the package it replaces. Knowing the edges saves you an afternoon.

**`node --run` has intentional limitations** ([CLI docs](https://nodejs.org/docs/latest-v24.x/api/cli.html)). It does not run `pre`/`post` scripts, does not understand workspaces, and does not read npm's `config` fields. So:

```jsonc
{
  "scripts": {
    "prebuild": "rimraf dist",   // ← node --run build will NOT run this
    "build": "tsc"
  }
}
```

```bash
npm run build        # runs prebuild, then build
node --run build     # runs build only — dist is never cleaned
```

If your scripts rely on `pre`/`post` hooks, `node --run` will silently do less work than you expect and the failure shows up as stale build output. The fix is to stop relying on implicit hooks (`"build": "rimraf dist && tsc"`), which is better practice anyway.

**`--watch` is a restarter, not a reloader.** It tears the process down and starts a new one. Anything expensive at startup — a connection pool, a JIT warm-up, an in-memory cache — is paid again on every save. There is no state-preserving hot reload in Node; if you need one, that is a framework-level feature (see [`nextjs/`](../nextjs/nextjs-mastery-guide.md)) or a different runtime.

**`--env-file` is not `dotenv`.** No variable expansion (`${HOST}/path` stays literal), no `.env.production` cascade by convention, no programmatic API. You choose files explicitly, in order, on the command line. For anything more elaborate — schema validation, typed config — see [Part 13](./nodejs-production-diagnostics.md#part-13--production-node).

**Real Scenario — the `.env` that stopped working in staging.** A service reads `LOG_LEVEL` from `--env-file=.env`, and the `.env` in the image says `LOG_LEVEL=debug`. In staging, logs come out at `info` and nobody can work out why the file is being ignored. It isn't: the deployment template sets `LOG_LEVEL=info` as a container env var, and **the environment takes precedence over the file**. Under `dotenv`'s default behaviour the result is the same, which is why the team had never hit it — but they had also never read either doc, and spent a morning on it. The lesson generalises: with `--env-file`, the file is a *default*, not an override.

<a id="p1-mastery"></a>

### Mastery: flag precedence, `NODE_OPTIONS`, and startup cost

**Precedence.** Command-line options beat `NODE_OPTIONS` ([CLI docs](https://nodejs.org/docs/latest-v24.x/api/cli.html)). That ordering is what makes `NODE_OPTIONS` safe to set org-wide in a base image:

```bash
export NODE_OPTIONS="--enable-source-maps --max-old-space-size=2048"
node --max-old-space-size=4096 app.js   # 4096 wins
```

`NODE_OPTIONS` only accepts a subset of flags — anything that would let an environment variable change *which code runs* (like `-e`, `-p`, or `--run`) is rejected, because `NODE_OPTIONS` is inherited by every child process and would otherwise be an arbitrary-code-execution vector. When a flag is rejected, Node refuses to start and names it, which makes this easy to diagnose the one time it happens.

**Startup cost is real and measurable.** Node's startup is dominated by bootstrapping the JS layer of the built-in modules, which is why a CLI that does nothing still takes tens of milliseconds:

```bash
node -e "console.log(process.uptime() * 1000)"   # → typically 20–40 (ms)
```

Two levers exist if that matters to you. A **startup snapshot** lets Node serialise the heap after your initialisation code has run and restore it on the next boot; a **single executable application** bundles your script into the Node binary itself. Both are covered in [Part 13](./nodejs-production-diagnostics.md#part-13--production-node), where the deployment context makes them concrete. Neither is worth reaching for until you have measured.

**The flag-discovery trick** worth carrying: `node --help` lists the stable flags, `node --v8-options` lists V8's (there are hundreds, most of which you should not touch), and `process.allowedNodeEnvironmentFlags` is a live `Set` of what `NODE_OPTIONS` will accept on *this* binary — more reliable than any list in a blog post, including this one:

```bash
node -p "[...process.allowedNodeEnvironmentFlags].filter(f => f.includes('watch'))"
```

<a id="part-1-cheat-sheet"></a>

### Part 1 cheat sheet

| Want | Command |
|---|---|
| Run a file | `node app.js` |
| Evaluate and print | `node -p "expr"` |
| Restart on save | `node --watch app.js` |
| Also watch non-imported files | `node --watch --watch-path=./sql app.js` |
| Keep terminal history across restarts | `node --watch --watch-preserve-output app.js` |
| Load env files, in order | `node --env-file=.env --env-file=.env.local app.js` |
| Load env file if it exists | `node --env-file-if-exists=.env.local app.js` |
| Run a package script, fast | `node --run build` |
| Pass args to that script | `node --run test -- --watch` |
| Preload an ES module | `node --import ./setup.mjs app.js` |
| Your CLI arguments | `process.argv.slice(2)` |
| Fail a CLI correctly | `process.exitCode = 1` (not `process.exit(1)`) |
| Set flags for every process | `NODE_OPTIONS="--enable-source-maps"` |
| See what `NODE_OPTIONS` accepts | `node -p "[...process.allowedNodeEnvironmentFlags].length"` |

**Gotchas:** `node --run` skips `pre`/`post` scripts and workspaces · real env vars beat `--env-file` values · `--watch` restarts, it does not hot-reload · `--print` rejects ESM syntax.

[↑ Back to top](#table-of-contents)

---

## Part 2 — The Event Loop

This is the Part that pays for itself. Almost every Node behaviour that surprises people — ordering, stalls, timeouts that fire late, one slow request poisoning a hundred fast ones — is the event loop being exactly what it is. All four tiers are here; none would be filler.

<a id="p2-beginner"></a>

### Beginner: one thread, one stack

Node runs your JavaScript on a single thread. When you call something asynchronous, you are not starting a parallel computation — you are *registering interest* and returning immediately. Later, when the result is ready, Node calls you back, on that same single thread, once it is free.

```js
import { readFile } from 'node:fs/promises';

console.log('1');
readFile('big.txt', 'utf8').then(() => console.log('3'));
console.log('2');
```

Prints `1`, `2`, `3`. The file read happened *somewhere else* (a libuv thread, as it turns out); your thread carried on to `console.log('2')` and only came back to the `.then()` once it had nothing else to do.

The single most important corollary, stated plainly: **while your code is running, nothing else can.** Not another request, not a timer, not a socket close. A synchronous 400 ms is 400 ms of total deafness.

```js
// ✗ wrong — 400ms where the server answers nobody
const hash = crypto.pbkdf2Sync(password, salt, 600_000, 32, 'sha512');

// ✓ right — the work happens on libuv's thread pool; the loop stays free
const hash = await promisify(crypto.pbkdf2)(password, salt, 600_000, 32, 'sha512');
```

<a id="p2-working"></a>

### Working Knowledge: the six phases

The "loop" is literally a loop, and each pass through it visits six phases in a fixed order ([Event Loop guide](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)):

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout / setInterval callbacks whose time has come
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  a few I/O callbacks deferred from the last iteration
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  internal only
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │  ← the loop BLOCKS here
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │  setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  socket.on('close'), etc.
   └───────────────────────────┘
```

Five of these you can mostly ignore. **The poll phase is the one to understand**, because it is where a Node process spends nearly all of its idle life. On entering poll, the loop:

- executes anything already in the poll queue, synchronously, until it is empty or a system limit is hit; then
- if `setImmediate()` callbacks are pending, it stops and moves on to **check**; or
- if not, it **blocks here, waiting for I/O** — for as long as the nearest pending timer allows.

That last clause is the whole design. An idle Node server is not spinning; it is parked in a `epoll_wait`-shaped hole with a timeout equal to "when is my next timer due", consuming no CPU at all.

#### `setTimeout(fn, 0)` vs. `setImmediate(fn)`

The classic interview question, and the honest answer is *it depends on where you are*.

```js
// timers.js — at the top level of a module
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
```

Run this repeatedly and **the order is non-deterministic**. `setTimeout(fn, 0)` is clamped to 1 ms, and whether that millisecond has already elapsed by the time the loop reaches the timers phase depends on how long process startup happened to take on that run.

Inside an I/O callback, though, the order is guaranteed:

```js
import { readFile } from 'node:fs';

readFile('package.json', () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});
```

`immediate` always wins — you are already inside the poll phase, and **check** comes immediately after it, whereas timers are a whole lap away.

**Try It — watch both behaviours.** Save both snippets and run the first one five times:

```bash
for i in 1 2 3 4 5; do node timers.js; done
```

Expected output: a mix of orderings across the five runs. Then run the I/O version five times: `immediate` before `timeout`, every single time. That difference — the same two lines, deterministic in one context and not the other — *is* the phase model, observable from the outside.

The docs' own recommendation, which is good advice: **use `setImmediate()` when you mean "after the current phase", because it is easier to reason about.**

<a id="p2-advanced"></a>

### Advanced: microtasks, `nextTick`, and starvation

There are two queues that are *not* phases, and they outrank everything above.

```
  ... end of any phase / between each callback ...
        │
        ├─ 1. drain the process.nextTick queue  ← completely
        └─ 2. drain the microtask queue         ← completely (promises)
        │
  ... next phase ...
```

Both are drained **to empty**, not one item at a time, and `nextTick` is drained first. Which produces the canonical ordering puzzle:

```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
console.log('sync');
```

Output, reliably:

```
sync
nextTick
promise
timeout     ← or immediate; these two race, per the previous section
immediate
```

Synchronous code first (it is the current stack), then the `nextTick` queue, then microtasks, then the loop resumes its phases.

#### Starvation

Because those queues are drained *completely* before the loop can advance, a queue that refills itself never lets the loop advance at all:

```js
// ✗ wrong — this hangs the process. No timer, no socket, no signal will ever be serviced.
function spin() { process.nextTick(spin); }
spin();

// ✓ right — setImmediate yields; each call lands in the NEXT check phase
function spin() { setImmediate(spin); }
spin();
```

The second version lets the loop complete a lap between iterations, so I/O still gets serviced. The Node docs put it memorably: the two names "should be swapped" — `process.nextTick()` fires *more* immediately than `setImmediate()` does, and that is an artifact of history that is not going to change ([Event Loop guide](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)).

Note the same trap exists with promises: an `async` function that recurses without ever awaiting something that actually yields to the loop will starve it just as thoroughly. The queue is different; the mechanism is identical.

#### What actually blocks, in practice

From the official "Don't Block the Event Loop" guide ([nodejs.org](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)), the realistic culprits are:

| Culprit | Why it hurts | What to do |
|---|---|---|
| `JSON.parse` / `JSON.stringify` on big payloads | Fully synchronous; a 50 MB object can cost a second or more | Stream it, cap request body size, or move it to a worker |
| Vulnerable regexes (ReDoS) | Nested quantifiers backtrack exponentially — `/(\/.+)+$/` on a crafted input can run effectively forever | Simplify the pattern, prefer `indexOf`/`startsWith`, or use a linear-time engine like RE2 |
| `*Sync` APIs | `readFileSync`, `execSync`, `pbkdf2Sync` — the whole family | Fine at startup, never per-request |
| Long loops over big arrays | Ordinary CPU work | Partition with `setImmediate`, or offload |

The partitioning trick is worth seeing once, because it is the cheapest fix available:

```js
// ✗ wrong — 10M iterations, one uninterrupted block
function sum(n) { let s = 0; for (let i = 1; i <= n; i++) s += i; return s; }

// ✓ right — the same work, in chunks, yielding between them
async function sum(n, chunk = 100_000) {
  let s = 0;
  for (let i = 1; i <= n; i++) {
    s += i;
    if (i % chunk === 0) await new Promise(setImmediate);  // let the loop breathe
  }
  return s;
}
```

This makes the total work *slower* and the process *responsive*. That trade is almost always the right one for a server, and almost always the wrong one for a CLI — which is the useful way to think about it.

<a id="p2-mastery"></a>

### Mastery: the thread pool, and measuring the loop

#### The thread pool nobody configures

libuv keeps a small pool of real OS threads — **four by default** — for work that has no non-blocking OS primitive. Four APIs families use it ([Don't Block the Event Loop](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)):

```
   your single JS thread
        │
        ├── network I/O ──────────► the OS (epoll/kqueue/IOCP) — no threads involved
        │
        └── fs.*, dns.lookup(), ──► libuv thread pool  ← 4 threads, shared, FIFO
            crypto.pbkdf2/scrypt/
            randomBytes, zlib.*
```

This is a genuinely load-bearing detail with two consequences:

1. **Four concurrent `pbkdf2` hashes will saturate the pool**, and the fifth waits — as does every `fs` read and every `dns.lookup()` in the process, because they are queued behind it. A login endpoint under load can therefore make *file serving* slow, which looks like nothing in your profiler until you know to look.
2. **`UV_THREADPOOL_SIZE` must be set before the pool is created** — i.e. as an environment variable at launch, not from inside your program:

```bash
# ✗ wrong — the pool already exists by the time this runs
node -e "process.env.UV_THREADPOOL_SIZE = 16; require('./app.js')"

# ✓ right
UV_THREADPOOL_SIZE=16 node app.js
```

Note the split in the diagram: **sockets are not on the pool.** A Node server handling 10,000 idle connections uses zero pool threads for them. That is precisely why Node scales on connection count while being poor at CPU work — the two facts are the same fact.

#### Measuring the loop, properly

"Is the loop blocked?" has a real answer, and it is not `console.time`. `perf_hooks` ships a histogram sampler:

```js
import { monitorEventLoopDelay } from 'node:perf_hooks';

const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();

setInterval(() => {
  console.log({
    p50: Math.round(h.percentile(50) / 1e6),   // ns → ms
    p99: Math.round(h.percentile(99) / 1e6),
    max: Math.round(h.max / 1e6),
  });
  h.reset();
}, 5_000).unref();     // ← unref: this timer must not keep the process alive
```

A healthy server shows p50 under a millisecond and p99 in single digits. A p99 of 400 ms means that, once in a while, something occupies the thread for 400 ms — and the *number* tells you what to go looking for. This is the first measurement in the diagnosis table in the [overview](./nodejs-mastery-guide.md#10-cross-doc-cheat-sheet), and [Part 14](./nodejs-production-diagnostics.md#part-14--diagnostics) takes it further.

That `.unref()` is worth a sentence of its own, because it is the other half of the loop's lifecycle: **Node exits when nothing is left that could still produce an event.** Every active timer, server and socket is a "handle" holding the process open. `unref()` says "keep this running, but don't count it" — which is why a monitoring interval needs it and a `setInterval` you forgot about is why your script won't exit.

**Try It — watch a process refuse to die.**

```js
// hang.js
const t = setInterval(() => {}, 1000);
console.log('done');
// add `t.unref()` here and run again
```

`node hang.js` prints `done` and then hangs forever. Add `t.unref()` and it prints `done` and exits immediately. No other change.

<a id="part-2-cheat-sheet"></a>

### Part 2 cheat sheet

| Concept | The thing to remember |
|---|---|
| Phase order | timers → pending → idle/prepare → **poll** → check → close |
| Where Node idles | The poll phase, blocked on I/O until the next timer is due |
| `setTimeout(fn, 0)` | Clamped to 1 ms; races with `setImmediate` at the top level |
| `setImmediate(fn)` | The **check** phase; always beats timers inside an I/O callback |
| `process.nextTick(fn)` | Before microtasks, before the loop advances at all |
| Microtasks (promises) | Drained to empty between every phase and callback |
| Starvation | A self-refilling `nextTick`/microtask queue stops the loop forever |
| Yield to the loop | `await new Promise(setImmediate)` |
| Thread pool | 4 threads; `fs`, `dns.lookup`, `crypto` KDFs, `zlib` — **not** sockets |
| Resize the pool | `UV_THREADPOOL_SIZE=16 node app.js` (env var only, at launch) |
| Measure blocking | `monitorEventLoopDelay()` from `node:perf_hooks` |
| Why won't it exit? | An active handle — `.unref()` the ones that shouldn't count |

[↑ Back to top](#table-of-contents)

---

## Part 3 — Async Patterns in Node

You know promises from JavaScript. What Node adds is a second and third convention that predate them and are not going away — error-first callbacks, and `EventEmitter` — plus a cancellation primitive (`AbortSignal`) that is now wired through the entire standard library. This Part is the vocabulary; every other doc in the set speaks it.

*This Part runs Beginner → Mastery, but the Beginner tier is deliberately thin: promises and `async`/`await` are language features, covered in [`javascript/javascript-errors-async-modules.md`](../javascript/javascript-errors-async-modules.md). Only the Node-specific shapes appear here.*

<a id="p3-beginner"></a>

### Beginner: the three calling conventions

Node's standard library speaks three dialects, and most modules offer more than one:

```js
// 1. error-first callback — the original, still the base layer of everything
import { readFile } from 'node:fs';
readFile('a.txt', 'utf8', (err, data) => {
  if (err) return console.error(err);      // error is ALWAYS the first argument
  console.log(data);
});

// 2. promise — the one to reach for; note the different import path
import { readFile } from 'node:fs/promises';
const data = await readFile('a.txt', 'utf8');

// 3. synchronous — blocks the thread; fine at startup, never per-request
import { readFileSync } from 'node:fs';
const data = readFileSync('a.txt', 'utf8');
```

The `node:` prefix on those imports is not decoration. It guarantees you get the built-in module and can never be shadowed by a package in `node_modules` named `fs`. It is the current convention in Node's own documentation, and this set uses it everywhere.

**The one rule for error-first callbacks**: `if (err)` must be the first line, and it must `return`. The classic bug is omitting the `return` and running the success path with `data` undefined.

Converting dialect 1 into dialect 2 is a one-liner:

```js
import { promisify } from 'node:util';
import { pbkdf2 } from 'node:crypto';

const pbkdf2Async = promisify(pbkdf2);
const key = await pbkdf2Async('secret', 'salt', 600_000, 32, 'sha512');
```

`promisify` is for callback APIs you don't control. For Node's own modules, check for a `/promises` submodule first — `fs/promises`, `dns/promises`, `timers/promises`, `stream/promises` all exist.

<a id="p3-working"></a>

### Working Knowledge: `EventEmitter`

The third convention. Where a callback delivers one result once, an emitter delivers many results over time: a server emitting `'request'`, a stream emitting `'data'`, a child process emitting `'exit'`. If you understand `EventEmitter`, most of `node:http`, `node:stream` and `node:child_process` stops being surprising.

```js
import { EventEmitter } from 'node:events';

class Job extends EventEmitter {
  async run() {
    this.emit('start');
    for (let i = 0; i <= 100; i += 25) this.emit('progress', i);
    this.emit('done', { ok: true });
  }
}

const job = new Job();
job.on('progress', (pct) => console.log(`${pct}%`));
job.once('done', (result) => console.log('finished', result));
await job.run();
```

`on` listens repeatedly, `once` listens exactly once and then removes itself, `off` removes a specific listener. Listeners are called **synchronously**, in registration order — `emit()` does not queue anything, it just calls your functions.

#### The `'error'` event will crash your process

This is the single most important `EventEmitter` fact, and it is a deliberate design decision rather than an accident ([events docs](https://nodejs.org/docs/latest-v24.x/api/events.html)):

> If an `EventEmitter` emits an `'error'` event with no registered listener, the error is thrown, a stack trace is printed, and the process exits.

```js
// ✗ wrong — one bad socket takes the whole server down
const server = net.createServer();
server.listen(3000);

// ✓ right
server.on('error', (err) => log.error({ err }, 'server error'));
```

Every emitter that can fail — servers, sockets, streams, child processes — needs an `'error'` listener. "Unhandled `'error'` event" in a crash log is always this.

#### The max-listeners warning is a leak detector

Add more than **10** listeners for the same event on one emitter and Node prints a `MaxListenersExceededWarning` about a "possible EventEmitter memory leak". It is not a limit, and the temptation is to silence it:

```js
// ✗ wrong — silences a real leak signal
emitter.setMaxListeners(Infinity);

// ✓ right — find out why listeners are accumulating; usually a missing off()
```

Nine times out of ten the warning is correct: something adds a listener per request and never removes it. Raise the limit only when you can explain why the real number is legitimately higher — and then raise it to that number, not to `Infinity`.

#### Awaiting an event

Two helpers bridge emitters back into `async`/`await`:

```js
import { once, on } from 'node:events';

// wait for exactly one event
const [code] = await once(child, 'exit');

// iterate events as they arrive
for await (const [chunk] of on(socket, 'data')) {
  process.stdout.write(chunk);
}
```

`once()` rejects if the emitter emits `'error'` first, which is exactly the behaviour you want and is tedious to write by hand.

<a id="p3-advanced"></a>

### Advanced: `AbortSignal`, and the two ways async code kills a process

#### Cancellation is standardised now

Node adopted the web's `AbortController`/`AbortSignal` and wired it through the standard library. The same signal cancels a `fetch`, an `fs` read, a timer, an event wait, and a stream:

```js
const ac = new AbortController();
const { signal } = ac;

setTimeout(() => ac.abort(), 5_000);        // one deadline...

await fetch(url, { signal });                // ...cancels all of these
await readFile('big.bin', { signal });
await once(emitter, 'ready', { signal });
await setTimeoutPromise(10_000, null, { signal });   // node:timers/promises
```

Aborting rejects with an error whose `name` is `'AbortError'`, so the handling idiom is:

```js
try {
  await work({ signal });
} catch (err) {
  if (err.name === 'AbortError') return;   // expected — we cancelled it
  throw err;
}
```

Two convenience constructors save real code: `AbortSignal.timeout(ms)` builds a self-aborting signal, and `AbortSignal.any([a, b])` combines several — "whichever happens first, the request being cancelled or the deadline passing":

```js
const signal = AbortSignal.any([req.signal, AbortSignal.timeout(3_000)]);
```

**Pass `signal` down.** A function that accepts an options bag and doesn't forward `signal` to whatever it awaits is the reason cancellation "doesn't work" — the abort fires and the work continues.

#### Death by unhandled rejection

A promise that rejects with nothing attached to catch it **terminates the process**. That is the default mode (`--unhandled-rejections=throw`), and it is the correct default: a rejection nobody handled is a bug, and a process in an unknown state should not keep serving traffic.

```js
// ✗ wrong — a floating promise; if it rejects, the process dies
sendAnalytics(event);

// ✓ right — either await it...
await sendAnalytics(event);

// ...or state explicitly that you don't care whether it fails
void sendAnalytics(event).catch((err) => log.warn({ err }, 'analytics failed'));
```

The corresponding last-resort handlers belong in exactly one place, your entry point, and they should **log and exit**, not log and continue:

```js
process.on('uncaughtException', (err) => {
  log.fatal({ err }, 'uncaught exception');
  process.exitCode = 1;
  shutdown();          // close the server, flush logs — see Part 13
});
```

The temptation is to swallow these and stay up. Don't: after an uncaught exception, some `finally` blocks did not run, some locks were not released, and some connections are half-written. Restart is the only safe response — which is why [Part 13](./nodejs-production-diagnostics.md#part-13--production-node) pairs this with a process supervisor.

#### Async functions as event listeners

A subtle one. `emit()` calls listeners synchronously and ignores their return values — so an `async` listener that throws produces an unhandled rejection, not an `'error'` event:

```js
// ✗ wrong — if this throws, it becomes an unhandled rejection and kills the process
emitter.on('data', async (row) => { await save(row); });

// ✓ right — handle it inside the listener
emitter.on('data', async (row) => {
  try { await save(row); } catch (err) { log.error({ err }); }
});
```

There is an opt-in that routes such rejections to the `'error'` event instead, which is cleaner when you have many async listeners ([events docs](https://nodejs.org/docs/latest-v24.x/api/events.html)):

```js
const emitter = new EventEmitter({ captureRejections: true });
emitter.on('error', (err) => log.error({ err }));
```

<a id="p3-mastery"></a>

### Mastery: async iteration and the modern primitives

#### `for await` is the shape most Node data has

Once you notice it, async iteration is everywhere in Node: streams are async iterables, `events.on()` returns one, `readline` gives you one per line, and a paginated API client should return one. It is the idiom that keeps memory flat:

```js
import { createReadStream } from 'node:fs';
import { createInterface } from 'node:readline';

const rl = createInterface({
  input: createReadStream('huge.log'),
  crlfDelay: Infinity,           // treat \r\n as one break — needed on Windows files
});

let errors = 0;
for await (const line of rl) {
  if (line.includes('ERROR')) errors++;
}
console.log(errors);
```

That reads a file of any size in constant memory, because the loop body is *backpressure*: while you are inside it, nothing more is read. [Part 7](./nodejs-files-streams-buffers.md#part-7--streams--backpressure) makes that mechanism explicit.

Writing your own is just a generator:

```js
async function* paginate(url) {
  let next = url;
  while (next) {
    const page = await fetch(next).then((r) => r.json());
    yield* page.items;
    next = page.next;
  }
}

for await (const item of paginate('/api/items')) { /* one at a time, ever */ }
```

#### `timers/promises`, and the timer you forgot to clear

```js
import { setTimeout as sleep, scheduler } from 'node:timers/promises';

await sleep(1_000);                       // a real sleep, cancellable via { signal }
await scheduler.yield();                  // yield to the loop — the setImmediate idiom, named
```

`scheduler.yield()` is the readable form of `await new Promise(setImmediate)` from [Part 2](#p2-advanced) — worth preferring in new code precisely because it says what it means.

#### Concurrency without a library

Two patterns cover most needs. Bounded parallelism, using the array as a queue:

```js
async function mapLimit(items, limit, fn) {
  const results = new Array(items.length);
  let i = 0;
  await Promise.all(
    Array.from({ length: Math.min(limit, items.length) }, async () => {
      while (i < items.length) {
        const idx = i++;                       // ← claim an index synchronously
        results[idx] = await fn(items[idx]);
      }
    }),
  );
  return results;
}
```

And "don't let one failure discard the other nineteen results":

```js
// ✗ wrong — one rejection throws away every other result
const results = await Promise.all(urls.map(fetchJson));

// ✓ right — every outcome, labelled
const settled = await Promise.allSettled(urls.map(fetchJson));
const ok = settled.filter((r) => r.status === 'fulfilled').map((r) => r.value);
```

#### The escape hatch: context that follows async boundaries

The thing `async_hooks` was invented for and `AsyncLocalStorage` made usable — a request id that stays attached across every `await`, without threading a parameter through forty functions:

```js
import { AsyncLocalStorage } from 'node:async_hooks';
export const ctx = new AsyncLocalStorage();

server.on('request', (req, res) => {
  ctx.run({ requestId: crypto.randomUUID() }, () => handle(req, res));
});

// anywhere, arbitrarily deep, after any number of awaits:
log.info({ requestId: ctx.getStore()?.requestId }, 'querying');
```

Node 24 changed the implementation underneath this to `AsyncContextFrame` ([release notes](https://nodejs.org/en/blog/release/v24.0.0)), which is faster and is why this is now a reasonable default rather than a performance compromise. The full treatment — including what it costs and where context still gets lost — is in [Part 14](./nodejs-production-diagnostics.md#part-14--diagnostics).

<a id="part-3-cheat-sheet"></a>

### Part 3 cheat sheet

| Want | How |
|---|---|
| Promise version of a core module | `node:fs/promises`, `node:dns/promises`, `node:timers/promises`, `node:stream/promises` |
| Promisify a callback API | `promisify(fn)` from `node:util` |
| Guarantee you get the built-in | Always import with the `node:` prefix |
| Listen repeatedly / once / stop | `emitter.on` / `.once` / `.off` |
| Prevent an emitter from crashing you | Always attach an `'error'` listener |
| Await a single event | `await once(emitter, 'name')` from `node:events` |
| Iterate events | `for await (const [x] of on(emitter, 'name'))` |
| Async listeners that may throw | `new EventEmitter({ captureRejections: true })` + an `'error'` listener |
| Cancel anything | Pass `{ signal }`; catch `err.name === 'AbortError'` |
| A deadline | `AbortSignal.timeout(ms)` |
| Combine cancellations | `AbortSignal.any([a, b])` |
| Sleep | `await setTimeout(ms)` from `node:timers/promises` |
| Yield to the event loop | `await scheduler.yield()` |
| All results, failures included | `Promise.allSettled` |
| Ambient per-request context | `AsyncLocalStorage` |
| Fire-and-forget on purpose | `void p.catch(handler)` — never a bare floating promise |

**Gotchas:** a missing `'error'` listener exits the process · so does an unhandled rejection · `async` listeners bypass `'error'` unless `captureRejections` is on · a `signal` you don't forward cancels nothing.

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 4 — Modules: CJS, ESM and TypeScript](./nodejs-modules-packages.md) · **Up:** [Overview](./nodejs-mastery-guide.md)

*Written against Node.js 24 LTS · Verified September 14, 2026*
