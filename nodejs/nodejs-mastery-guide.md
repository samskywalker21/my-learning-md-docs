# Mastering Node.js — A Hands-On Reference Guide

## About This Document

- **Framing:** Curiosity-driven. No upcoming task is driving this — the goal is durable, hands-on understanding of the Node.js runtime itself. Theory ("why it's built this way") only shows up where it's needed to explain a mechanism (the event loop, module resolution, backpressure), not as standalone history.
- **Style:** A guided tutorial — a sequence of runnable examples and exercises meant to be typed into a REPL or file, in order, matching [javascript/javascript-mastery-guide.md](../javascript/javascript-mastery-guide.md)'s format.
- **Depth tiers:** Every major Part is structured as four progressive, explicitly labeled tiers — **Beginner → Working Knowledge → Advanced → Mastery** — collapsed where they'd be trivial or redundant for that topic (this happens most in [Part 9 — npm & the Package Ecosystem](#10-part-9--npm--the-package-ecosystem), least in [Part 2 — The Event Loop](#3-part-2--the-event-loop--async-model) and [Part 6 — Streams & Buffers](#7-part-6--streams--buffers), where Node's real "aha" moments live).
- **Scope:** Full breadth of the Node.js runtime — event loop, modules, core built-in APIs (`fs`, `http`, `net`, streams, `process`), npm/package ecosystem, child processes/worker threads, and debugging. Not covered: frameworks (Express, Fastify, Nest), databases, deployment/infra. A brief native-TypeScript-in-Node note is folded into Part 3 (Modules) rather than given its own Part.
- **Assumed background:** Solid JavaScript (closures, promises, `async`/`await`, ES modules) — see [javascript/javascript-mastery-guide.md](../javascript/javascript-mastery-guide.md) — but no prior Node.js runtime/API experience.
- **Sourcing standard:** [nodejs.org/api](https://nodejs.org/api/) is the primary source, cited inline. Stack Overflow is used for real-world gotchas and common mistakes. Checked as of September 2026 against Node.js 26 (Current) / Node.js 24 (Active LTS); version-specific behavior is flagged inline, especially the 2026 release-cadence change (see [Part 1](#2-part-1--orientation--the-runtime)) and the recent stabilization of `require(esm)` and native TypeScript stripping (see [Part 3](#4-part-3--modules--commonjs--esm)).
- **Update instructions for future-me:** If asked to update this later — match this exact structure (numbered Parts, each with tier subheadings), keep the wrong-vs-right snippet pairs for gotchas, keep the TOC + back-to-top links, and prefer adding new Parts/tiers over reverting to freeform prose. Collapsing tiers that feel forced for a given topic is expected, not a deviation to "fix." Re-verify the current Node version and LTS status before editing version-specific claims — the release model changed in October 2026 (one release/year, every release becomes LTS).

---

## Table of Contents

1. [Orientation & the Runtime](#2-part-1--orientation--the-runtime)
2. [The Event Loop & Async Model](#3-part-2--the-event-loop--async-model)
3. [Modules — CommonJS & ESM](#4-part-3--modules--commonjs--esm)
4. [The `process` Object & Globals](#5-part-4--the-process-object--globals)
5. [File System (`fs`)](#6-part-5--file-system-fs)
6. [Streams & Buffers](#7-part-6--streams--buffers)
7. [Networking — `http` & `net`](#8-part-7--networking--http--net)
8. [Error Handling & Process Lifecycle](#9-part-8--error-handling--process-lifecycle)
9. [npm & the Package Ecosystem](#10-part-9--npm--the-package-ecosystem)
10. [Child Processes & Worker Threads](#11-part-10--child-processes--worker-threads)
11. [Debugging & Diagnostics](#12-part-11--debugging--diagnostics)
12. [Cheat Sheets](#13-cheat-sheets)
13. [Suggested Learning Order](#14-suggested-learning-order)
14. [Quick Self-Check](#15-quick-self-check)

---

## 1. Part 1 — Orientation & the Runtime

**What it is.** Node.js is a JavaScript runtime built on Chrome's [V8 engine](https://v8.dev/) plus [libuv](https://libuv.org/), a C library that provides the event loop, thread pool, and async I/O (file system, DNS, networking) on every OS. V8 executes your JavaScript; libuv is what lets that JavaScript do non-blocking I/O without you managing OS threads directly. Node glues the two together and exposes the result as built-in modules (`fs`, `http`, `process`, ...).

**Why it matters.** Everything unusual about Node — the single-threaded-but-non-blocking model, why CPU-bound work is a bad fit, why `fs.readFileSync` is dangerous in a server — traces back to this V8 + libuv split.

```
┌─────────────────────────────────────────┐
│              Your JS code                │
├─────────────────────────────────────────┤
│   Node.js APIs (fs, http, process, ...)  │
├──────────────────┬────────────────────── ┤
│   V8 (executes    │   libuv (event loop,  │
│   your JS)        │   thread pool, I/O)   │
└──────────────────┴────────────────────── ┘
```

### Beginner

Install via the [official installer](https://nodejs.org/en/download) or a version manager (`nvm`, `fnm`) — a version manager is strongly preferred once you have more than one project, since Node versions drift per-project.

```bash
node --version        # check installed version
node                  # start the REPL
node script.js        # run a file
node --watch script.js   # re-run automatically on file change (stable since Node 20)
```

**Real Scenario — try it now.** Start the REPL and confirm you're on a supported version:

```js
$ node
> process.version
'v24.9.0'
> process.platform
'win32'
```

### Working Knowledge

- **Release lines, as of September 2026:** Node.js 26 is the **Current** release; Node.js 24 is **Active LTS**; Node.js 22 is in **Maintenance LTS**. Production code should target an LTS line.
- **The release model changed in 2026.** Starting with Node.js 27 (October 2026), Node moves from the old even/odd scheme (new major every 6 months, even majors promoted to LTS after 6 months) to **one major release per year in April**, with **every release promoted to LTS in October** the same year. Node 26 is the last release under the old model. If you read older blog posts about "even-numbered = LTS," that rule stops applying going forward — check [nodejs.org/en/about/previous-releases](https://nodejs.org/en/about/previous-releases) for the current schedule rather than trusting the old rule of thumb.
- `.nvmrc` / `engines` field in `package.json` pin a project's expected version — always check for one before assuming your global Node version is right for a repo.

### Advanced

- Node ships a **single-threaded JS execution model** with a background thread pool (libuv, default size 4, tunable via `UV_THREADPOOL_SIZE`) used for things that can't be done async at the OS level (some `fs` operations, DNS lookups via `dns.lookup`, crypto). Understanding this split is the prerequisite for [Part 2](#3-part-2--the-event-loop--async-model).
- `process.release`, `process.versions` expose the exact V8/libuv/OpenSSL versions bundled with your Node build — useful when debugging version-specific native-module or TLS issues.

### Mastery

- Node's ABI (application binary interface) version, exposed via `process.versions.modules`, is what native addons (`.node` files, built with `node-gyp` or N-API) are compiled against — an addon built for one Node major often won't load on another without a rebuild. N-API (`node-api`) exists specifically to give native addon authors an ABI-stable surface across Node versions.

[⬆ back to top](#table-of-contents)

---

## 2. Part 2 — The Event Loop & Async Model

**What it is.** The event loop is the mechanism that lets single-threaded Node handle thousands of concurrent I/O operations: your JS code registers a callback for an async operation, control returns immediately, and libuv invokes the callback once the operation completes — via one of several ordered **phases**.

**Why it matters.** Timer callbacks, I/O callbacks, and `setImmediate` don't all fire in the order you'd naively guess — they're bucketed by phase. Getting this wrong causes real bugs: starved I/O, out-of-order logs, "why did my `setTimeout(fn, 0)` run after this other thing."

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout / setInterval callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks      │  deferred callbacks for some system ops
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │        poll                │  fetch new I/O events; execute I/O callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │        check               │  setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤   close callbacks          │  e.g. socket.on('close', ...)
   └───────────────────────────┘

Between EVERY phase transition, Node drains:
  1. process.nextTick queue (fully, including nextTicks added during draining)
  2. Promise microtask queue
```

### Beginner

`setTimeout`/`setInterval` schedule work for a future event-loop tick; they never guarantee exact timing, only a *minimum* delay.

```js
console.log('start');
setTimeout(() => console.log('timeout'), 0);
Promise.resolve().then(() => console.log('promise'));
console.log('end');
// Output: start, end, promise, timeout
// Microtasks (promises) always drain before the next macrotask (timers).
```

### Working Knowledge

`process.nextTick()` queues a callback that runs **before** any promise microtask and before the event loop continues to the next phase — it has higher priority than `Promise.then`. This surprises people coming from browser-only async experience, where there's no `nextTick`.

```js
// WRONG — assumes Promise microtasks run first
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
// Actual output: nextTick, promise — nextTick always wins
```

`setImmediate()` vs `setTimeout(fn, 0)`: inside the main module, their relative order is not guaranteed (it depends on process startup performance), but inside an I/O callback, `setImmediate` always fires before any timer, because you're already past the `poll` phase and heading straight into `check`.

```js
const fs = require('node:fs');
fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});
// Inside an I/O callback, immediate always logs first — deterministic here,
// unlike the same race run at the top of the file.
```

### Advanced

**Real Scenario.** A Node HTTP server feels sluggish under load despite low CPU usage. The cause: a hot path does synchronous, CPU-heavy JSON parsing/serialization or a tight synchronous loop, which blocks the single JS thread and starves the event loop — no other request's callback runs until it finishes, even though the OS-level sockets are idle and ready. Fixes: move the heavy work off-thread (a [worker thread](#11-part-10--child-processes--worker-threads)), break it into chunks yielded via `setImmediate`, or use a native/streaming parser that processes incrementally instead of materializing everything synchronously.

- **Starvation gotcha:** because `process.nextTick` and the microtask queue drain *fully* (including new callbacks queued during the drain) before the loop proceeds, recursively scheduling `process.nextTick` from within itself can starve I/O entirely — this was common enough that Node added `process.maxTickDepth` in early versions (later removed in favor of just documenting the hazard). Prefer `setImmediate` over `process.nextTick` for recursive/deferred work you want to *not* starve I/O.

### Mastery

- The event loop itself is implemented in libuv as `uv_run`, and each phase corresponds to a specific libuv queue (`uv__run_timers`, `uv__io_poll`, etc.) — reading the [libuv design overview](https://docs.libuv.org/en/v1.x/design.html) is the natural next step if you want to go below Node's abstraction.
- `async_hooks` (and its successor `AsyncLocalStorage`, see [Part 4](#5-part-4--the-process-object--globals)) let you trace the causal chain of async operations across the event loop — this is how APM tools (Datadog, New Relic) attribute a database query back to the HTTP request that triggered it, despite the loop having no concept of "which request am I in."

[⬆ back to top](#table-of-contents)

---

## 3. Part 3 — Modules — CommonJS & ESM

**What it is.** Node has two module systems: **CommonJS** (`require`/`module.exports`, Node's original system, synchronous) and **ECMAScript Modules** (`import`/`export`, the JS-language-standard system, supports top-level `await`). Which one a file uses is determined by its extension (`.cjs`/`.mjs` are unambiguous) or, for plain `.js`, by the nearest `package.json`'s `"type"` field (`"commonjs"` — the default — or `"module"`).

**Why it matters.** Mixing the two badly is the single most common source of "why won't this import" issues in real Node projects.

### Beginner

```js
// CommonJS (default for .js unless package.json says "type": "module")
const fs = require('node:fs');
module.exports = { hello: () => 'hi' };
```

```js
// ESM — requires "type": "module" in package.json, or a .mjs extension
import fs from 'node:fs';
export const hello = () => 'hi';
```

The `node:` prefix (`node:fs` vs bare `fs`) is optional for built-ins but recommended — it's unambiguous and, for some built-ins going forward, will be required. [Node's docs](https://nodejs.org/api/modules.html#core-modules) recommend it explicitly.

### Working Knowledge

| | CommonJS | ESM |
|---|---|---|
| Import syntax | `require()` | `import` |
| Loading | Synchronous | Asynchronous (but usually invisible to you) |
| Top-level `await` | Not allowed | Allowed |
| `__dirname` / `__filename` | Built-in | Not defined — use `import.meta.url` instead |
| File resolution | Extension optional, resolves `index.js` | Extension usually **required** |

```js
// WRONG in an ESM file — __dirname doesn't exist
console.log(__dirname);

// RIGHT
import { fileURLToPath } from 'node:url';
import { dirname } from 'node:path';
const __dirname = dirname(fileURLToPath(import.meta.url));
```

### Advanced

**`require(esm)` — a 2025/2026 change worth knowing about.** Historically, CommonJS code could never `require()` an ES module — you'd get `ERR_REQUIRE_ESM`. This has changed: `require(esm)` was unflagged in Node 23, backported to the Node 22 and 20 LTS lines, and marked **stable** as of Node 25.4 (January 2026) — every currently supported Node release can now `require()` an ES module directly, as long as that module (and everything it *synchronously* `require`s in turn) contains no top-level `await`. If any file in that require-chain has top-level `await`, you still get `ERR_REQUIRE_ASYNC_MODULE`. ([Joyee Cheung's implementer notes](https://joyeecheung.github.io/blog/2025/12/30/require-esm-in-node-js-from-experiment-to-stability/) cover a real 2026 incident where `lru-cache` added top-level await to its ESM build and broke `jsdom` transitively for everyone using `require()`.) The reverse direction — `import`-ing a CommonJS module from ESM — has always worked, since CJS has no async loading requirement.

- **Native TypeScript, no transpiler.** As of Node 22.18 / 24.3 (mid-2026), running a `.ts` file with only "erasable" TypeScript syntax (types, interfaces, no `enum`/namespaces/decorators) works with **no flag and no build step** — Node strips the types at load time. `--experimental-transform-types` extends this to `enum`/namespace syntax. See [nodejs.org/api/typescript.html](https://nodejs.org/api/typescript.html). This does *not* replace `tsc` for full type-checking — it only makes the file runnable; you still need a real TypeScript toolchain (or your editor) to catch type errors.

```bash
node script.ts   # works out of the box on Node 22.18+/24.3+ for erasable-syntax TS
```

### Mastery

- **Dual packages** (publishing both CJS and ESM builds of one npm package) are notoriously fragile because of the "dual package hazard" — if both builds get loaded in the same process (e.g. one dependency `require()`s the CJS build while another `import`s the ESM build), you can end up with two separate module instances, breaking `instanceof` checks and shared singleton state. `package.json` `"exports"` conditional maps (`"import"`/`"require"` keys) are the sanctioned way to publish both while steering consumers to one instance where possible — see [Node's dual package docs](https://nodejs.org/api/packages.html#dual-commonjses-module-packages).
- Node's ESM loader supports **loader hooks** (`--experimental-loader` / `register()` from `node:module`) for intercepting resolution and loading — this is how tools like `tsx` and `ts-node --esm` inject on-the-fly TypeScript compilation into the ESM pipeline.

[⬆ back to top](#table-of-contents)

---

## 4. Part 4 — The `process` Object & Globals

**What it is.** `process` is a global object giving you access to the current Node process: arguments, environment variables, standard streams, exit control, and signals. Unlike browser globals, most of Node's other "globals" (`__dirname`, `require`, `module`) are actually per-module, injected by the CommonJS wrapper — `process` is a true global.

### Beginner

```js
process.argv;          // ['node', '/path/to/script.js', 'arg1', 'arg2']
process.env.NODE_ENV;  // reads an environment variable (or undefined)
process.exit(1);       // exit immediately with code 1 — avoid; see Advanced tier
```

### Working Knowledge

```js
// WRONG — process.exit() during pending async work can truncate output/writes
fs.writeFile('out.txt', data, () => {});
process.exit(0); // may exit before the write's callback (or even the write) completes

// RIGHT — let the event loop drain naturally, or exit only from the callback
fs.writeFile('out.txt', data, (err) => {
  if (err) process.exitCode = 1;
});
// Setting process.exitCode (not calling exit()) lets Node finish pending work
// and exit gracefully with that code once the event loop empties on its own.
```

- `process.env` values are always strings — `process.env.PORT` is `"3000"`, not `3000`; a common bug is comparing it to a number without coercion.
- Command-line argument parsing: `node:util`'s `parseArgs()` (stable since Node 20) is a built-in alternative to reaching for `yargs`/`commander` for simple CLIs.

### Advanced

- **Signals:** `process.on('SIGINT', ...)` / `process.on('SIGTERM', ...)` are how you implement graceful shutdown (close DB pools, finish in-flight requests, then exit) — see [Part 8](#9-part-8--error-handling--process-lifecycle) for the full pattern.
- **`AsyncLocalStorage`** (`node:async_hooks`) lets you carry request-scoped context (a request ID, a user session) through an entire async call chain without threading a parameter through every function — the mechanism most logging/tracing libraries use for "which request is this log line from" in a server handling many concurrent requests.

```js
const { AsyncLocalStorage } = require('node:async_hooks');
const als = new AsyncLocalStorage();

function handleRequest(id, next) {
  als.run({ requestId: id }, next);
}

function logSomewhereDeep() {
  console.log(als.getStore().requestId); // available without being passed in
}
```

### Mastery

- `process.binding()` (internal, undocumented, do not use in application code) is the low-level bridge to C++ internals that most `node:*` built-in modules are themselves implemented on top of — useful only for understanding *how* built-ins are built, never for direct use.
- `process.memoryUsage()` / `process.resourceUsage()` expose heap, RSS, and CPU-time stats used for building custom health checks or diagnosing memory leaks in long-running servers (paired with `--inspect` heap snapshots — see [Part 11](#12-part-11--debugging--diagnostics)).

[⬆ back to top](#table-of-contents)

---

## 5. Part 5 — File System (`fs`)

**What it is.** The `fs` module ships in three flavors: **synchronous** (`fs.readFileSync`, blocks the event loop until done), **callback-based** (`fs.readFile(path, cb)`, the original async style), and **promise-based** (`fs.promises` / `node:fs/promises`, works with `async`/`await`). All three wrap the same underlying libuv operations.

### Beginner

```js
const fs = require('node:fs/promises');

async function main() {
  const data = await fs.readFile('notes.txt', 'utf8');
  console.log(data);
}
main();
```

### Working Knowledge

```js
// WRONG — sync fs calls in a request handler block every other request
app.get('/report', (req, res) => {
  const data = fs.readFileSync('big-report.json', 'utf8'); // blocks the event loop
  res.send(data);
});

// RIGHT — async fs keeps the event loop free for other requests
app.get('/report', async (req, res) => {
  const data = await fs.promises.readFile('big-report.json', 'utf8');
  res.send(data);
});
```

Sync `fs` calls are fine at **startup** (reading config before the server starts listening) — the rule is "never in a hot path that runs per-request," not "never."

### Advanced

**Real Scenario.** A file-upload endpoint works fine in dev but OOMs in production under real traffic. The cause: `fs.readFile`/`writeFile` load the *entire* file into memory as a single Buffer before resolving — fine for small config files, disastrous for large uploads processed concurrently. The fix is streaming (see [Part 6](#7-part-6--streams--buffers)): `fs.createReadStream`/`createWriteStream` process the file in chunks, keeping memory bounded regardless of file size or concurrency.

- `fs.watch()` provides OS-level file-change notification, but its behavior (recursive support, event coalescing, filename availability) varies meaningfully across Windows/macOS/Linux — [the docs explicitly warn about this](https://nodejs.org/api/fs.html#caveats) rather than presenting it as a portable abstraction. For portable production file-watching, most people still reach for `chokidar`.

### Mastery

- File descriptors: every `fs` open (`fs.open`) returns a numeric file descriptor that must eventually be `close()`d — promise-based `fs.promises.open()` returns a `FileHandle` object which is *not* automatically closed on garbage collection in a timely way (Node emits a warning, but leaked FDs can still exhaust the OS limit under load); always close explicitly (`try`/`finally` or `using`, once explicit resource management lands broadly).

[⬆ back to top](#table-of-contents)

---

## 6. Part 6 — Streams & Buffers

**What it is.** A **Buffer** is Node's structure for raw binary data (outside V8's normal string/array memory model) — used anywhere you're dealing with bytes rather than text (file contents, network payloads, crypto). A **Stream** is an abstraction for processing data piece-by-piece instead of all-at-once: `Readable` (a source), `Writable` (a sink), `Duplex` (both), and `Transform` (a duplex that modifies data as it passes through, e.g. gzip).

**Why it matters.** Streams are how Node handles data larger than available memory, and are the backbone of `fs`, `http`, and `net`.

```
Readable ──pipe──> Writable
   (source)          (sink)

Readable ──pipe──> Transform ──pipe──> Writable
   (source)        (e.g. gzip)           (sink)
```

### Beginner

```js
const fs = require('node:fs');

const readStream = fs.createReadStream('input.txt');
const writeStream = fs.createWriteStream('output.txt');
readStream.pipe(writeStream); // reads chunk-by-chunk, writes chunk-by-chunk
```

### Working Knowledge

```js
// WRONG — buffers the whole file in memory, defeats the point of streaming
const data = fs.readFileSync('huge-video.mp4');
res.end(data);

// RIGHT — streams the file to the response in bounded chunks
fs.createReadStream('huge-video.mp4').pipe(res);
```

**Backpressure** is the mechanism preventing a fast source from overwhelming a slow destination: `writable.write()` returns `false` when its internal buffer is full, signaling the source to pause; `.pipe()` handles this automatically, which is exactly why `.pipe()` (or `stream.pipeline()`) is preferred over manually wiring `.on('data', ...)` to `.write()`.

### Advanced

**Real Scenario.** A script piping a large file through a slow network `Writable` (e.g. an upload to S3) crashes with an out-of-memory error, despite using streams. The cause: an unhandled error partway through the pipe leaves the source stream still emitting `data` events into a destination that's stopped consuming, and without proper cleanup, buffered chunks accumulate. The fix: use `stream.pipeline()` (not raw `.pipe()`), which propagates errors and guarantees all streams in the chain are destroyed/cleaned up on failure — plain `.pipe()` does not do this.

```js
const { pipeline } = require('node:stream/promises');

// WRONG — errors on readStream aren't cleanly propagated to writeStream
readStream.pipe(writeStream);

// RIGHT — pipeline handles errors and cleanup across the whole chain
await pipeline(readStream, transformStream, writeStream);
```

### Mastery

- Streams operate internally on Buffers by default (`objectMode: false`); setting `{ objectMode: true }` lets a stream carry arbitrary JS values instead of bytes — the basis for building streaming data-processing pipelines (e.g. row-by-row CSV/JSON transforms) that aren't about raw I/O at all.
- `Buffer` shares underlying memory with `ArrayBuffer`/`TypedArray` — `Buffer.from(arrayBuffer)` can create a *view* over existing memory rather than copying, which matters for high-throughput binary processing where copies are the bottleneck. See [nodejs.org/api/buffer.html](https://nodejs.org/api/buffer.html).

[⬆ back to top](#table-of-contents)

---

## 7. Part 7 — Networking — `http` & `net`

**What it is.** `node:http` builds HTTP servers/clients directly on top of `node:net` (raw TCP sockets). Frameworks like Express and Fastify are built on `node:http` — understanding the raw module demystifies what those frameworks are actually doing underneath middleware chains and routers.

### Beginner

```js
const http = require('node:http');

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello, Node\n');
});

server.listen(3000, () => console.log('listening on :3000'));
```

### Working Knowledge

```js
// WRONG — assumes the whole request body arrives in one 'data' event
server.on('request', (req, res) => {
  req.on('data', (chunk) => res.end(chunk)); // truncates multi-chunk bodies
});

// RIGHT — req is a Readable stream; accumulate all chunks before using the body
server.on('request', (req, res) => {
  const chunks = [];
  req.on('data', (chunk) => chunks.push(chunk));
  req.on('end', () => {
    const body = Buffer.concat(chunks).toString('utf8');
    res.end(`received ${body.length} bytes`);
  });
});
```

Node 18+ ships a built-in `fetch()` global (powered by [undici](https://github.com/nodejs/undici) under the hood) — for making *outbound* HTTP requests, you generally don't need `axios`/`node-fetch` anymore.

### Advanced

**Real Scenario.** A Node API server handles normal traffic fine but falls over under a connection-heavy load test, with sockets piling up in `TIME_WAIT` or requests hanging. Common causes: not calling `res.end()` on every code path (including error paths — a forgotten `return` after `res.end()` can double-send), not setting `server.timeout`/`keepAliveTimeout` appropriately for a load balancer in front of it (mismatched keep-alive timeouts between Node and an upstream LB is a classic 502 source), or exhausting `http.Agent`'s default connection pool on the *client* side when Node itself is making many outbound requests (`http.globalAgent.maxSockets` defaults to `Infinity` as of modern Node, but custom agents used to default to 5, tripping people up).

- `node:net` gives raw TCP: `net.createServer()`/`net.connect()` — useful for understanding that `http` is just a well-defined byte protocol (a request line, headers, blank line, body) parsed on top of a socket, not something magical.

### Mastery

- HTTP/2 (`node:http2`) and the `Agent` connection-pooling model diverge meaningfully from HTTP/1.1 — HTTP/2's multiplexing means one TCP connection can carry many concurrent streams, changing the calculus around connection-pool sizing entirely.
- Cluster module vs. worker threads for scaling an HTTP server across CPU cores: `node:cluster` forks full processes (each with its own event loop and memory), sharing the listening socket via round-robin or OS-level load balancing — a different tradeoff than [worker_threads](#11-part-10--child-processes--worker-threads), covered in Part 10.

[⬆ back to top](#table-of-contents)

---

## 8. Part 8 — Error Handling & Process Lifecycle

**What it is.** Node distinguishes **operational errors** (expected failures — a file doesn't exist, a network call times out — handle these with `try`/`catch` or `.catch()`) from **programmer errors** (bugs — a `TypeError` from calling a method on `undefined`). The two process-level events `uncaughtException` and `unhandledRejection` exist as a last line of defense for the latter, not a substitute for the former.

### Beginner

```js
try {
  JSON.parse(invalidJson);
} catch (err) {
  console.error('bad JSON:', err.message);
}
```

### Working Knowledge

```js
// WRONG — an unhandled rejection is silently swallowed pre-Node 15,
// and crashes the process by default on Node 15+ — either way, don't rely on it
somePromiseReturningFn(); // no .catch, no await inside an async fn

// RIGHT — handle it, or let it propagate to a caller that will
somePromiseReturningFn().catch((err) => log.error(err));
```

As of Node 15+, an unhandled promise rejection **terminates the process by default** (matching `uncaughtException` behavior) — this is a real, easy-to-miss change from earlier Node versions where it only printed a warning. See [Node's `--unhandled-rejections` flag docs](https://nodejs.org/api/cli.html#--unhandled-rejectionsmode) if you need to tune this.

### Advanced

**Real Scenario — graceful shutdown.** A container orchestrator (Kubernetes, ECS) sends `SIGTERM` before killing a pod. Without a handler, Node exits immediately, dropping any in-flight requests. The standard pattern:

```js
let shuttingDown = false;

process.on('SIGTERM', async () => {
  if (shuttingDown) return;
  shuttingDown = true;
  server.close(() => {           // stop accepting new connections
    dbPool.end(() => {           // close resources once existing ones finish
      process.exit(0);
    });
  });
  // Safety net: force-exit if graceful shutdown hangs
  setTimeout(() => process.exit(1), 10_000).unref();
});
```

- `process.on('uncaughtException', ...)` should, per [Node's own guidance](https://nodejs.org/api/process.html#warning-using-uncaughtexception-correctly), be used only to perform synchronous cleanup and then exit — the process is in an undefined state after an uncaught exception, and continuing to run risks corrupted state or resource leaks. Don't use it as a global try/catch to keep serving requests.

### Mastery

- `--unhandled-rejections=strict` makes unhandled rejections behave identically to uncaught exceptions (crash immediately) rather than warning-then-crash — useful for catching these earlier in CI.
- Domains (`node:domain`) were an early attempt at this problem and are **deprecated** — if you see them in older code, they should be migrated to `AsyncLocalStorage` + explicit error handling, not extended.

[⬆ back to top](#table-of-contents)

---

## 9. Part 9 — npm & the Package Ecosystem

**What it is.** `npm` (bundled with Node) manages dependencies via `package.json` (what you depend on) and a lockfile (`package-lock.json`, the exact resolved tree) — the lockfile is what makes installs reproducible across machines.

### Beginner

```bash
npm init -y                  # create a package.json
npm install express          # add a runtime dependency
npm install --save-dev vitest # add a dev-only dependency
npm run build                 # run a script defined in package.json's "scripts"
npx cowsay hello              # run a package's binary without installing it globally
```

### Working Knowledge

```json
// package.json (excerpt)
{
  "name": "my-app",
  "version": "1.0.0",
  "type": "module",
  "engines": { "node": ">=22" },
  "dependencies": { "express": "^4.19.0" },
  "devDependencies": { "vitest": "^2.0.0" }
}
```

**Semver ranges:** `^4.19.0` allows updates that don't change the leftmost non-zero digit (so `4.x.x` but not `5.0.0`); `~4.19.0` allows only patch updates (`4.19.x`). `npm ci` (vs `npm install`) installs *exactly* what's in the lockfile and fails if `package.json` and the lockfile disagree — this is what CI pipelines should use, not `npm install`, precisely to catch that drift.

### Advanced

**Real Scenario.** "Works on my machine" — a teammate's install produces a subtly different dependency tree than yours, and a bug only reproduces on their machine. Cause: someone ran `npm install` (which can update the lockfile within semver ranges) instead of `npm ci`, or `package-lock.json` wasn't committed. Fix: always commit the lockfile, always use `npm ci` in CI/deploy, and treat any lockfile diff in a PR as worth reviewing, not auto-accepting.

- **Workspaces** (`"workspaces"` in a root `package.json`) let one repo manage multiple packages (a monorepo) with npm handling cross-linking and a single top-level `node_modules` — see [npm's workspaces docs](https://docs.npmjs.com/cli/v10/using-npm/workspaces).

### Mastery

- `package.json` `"exports"` field controls exactly what a package's consumers can `import`/`require` (and under which conditions — `"import"` vs `"require"` vs `"node"` vs `"browser"`), superseding the older "just import any file by path" convention — this is the mechanism dual CJS/ESM packages use (see [Part 3, Mastery](#4-part-3--modules--commonjs--esm)).
- Supply-chain hygiene: `npm audit`, and treating `postinstall` scripts in third-party packages as arbitrary code execution on your machine (because they are) — pin and review dependencies accordingly for anything security-sensitive.

[⬆ back to top](#table-of-contents)

---

## 10. Part 10 — Child Processes & Worker Threads

**What it is.** Node offers two different ways to get parallelism beyond one JS thread: **`child_process`** (spawn a whole separate OS process — could be another Node script, or any executable at all) and **`worker_threads`** (spawn another JS thread *within the same process*, with its own event loop, sharing memory only via explicit message-passing or `SharedArrayBuffer`).

**Why it matters.** Picking the wrong one is a common mistake: worker threads are for CPU-bound JS work you want off the main thread; child processes are for running other programs or for OS-level process isolation.

### Beginner

```js
const { spawn } = require('node:child_process');
const ls = spawn('ls', ['-la']);
ls.stdout.on('data', (data) => console.log(data.toString()));
```

```js
const { Worker } = require('node:worker_threads');
const worker = new Worker('./heavy-computation.js');
worker.on('message', (result) => console.log('got:', result));
```

### Working Knowledge

| | `child_process` | `worker_threads` |
|---|---|---|
| Runs | Any executable (or another Node script) | Only JS, in the same Node process |
| Memory | Fully isolated | Can share memory (`SharedArrayBuffer`) |
| Overhead | Higher (new OS process) | Lower (thread, not process) |
| Use case | Running external tools, isolating untrusted code | Offloading CPU-bound JS (image processing, crypto, parsing) |

```js
// WRONG — CPU-bound work on the main thread blocks every request the server handles
app.get('/hash', (req, res) => {
  const hash = expensiveHashLoop(req.query.data); // blocks event loop for everyone
  res.json({ hash });
});

// RIGHT — offload to a worker thread, main thread stays free to serve other requests
app.get('/hash', async (req, res) => {
  const hash = await runInWorker('./hash-worker.js', req.query.data);
  res.json({ hash });
});
```

### Advanced

- `child_process.fork()` is a specialized `spawn()` for launching another **Node** script specifically, with a built-in IPC channel for message-passing — different from `spawn()`, which is generic and has no IPC unless you wire up stdio pipes yourself.
- `exec()`/`execFile()` buffer the entire stdout/stderr in memory and hand you a single callback — fine for short-lived commands with small output, wrong for anything long-running or high-volume (use `spawn()`'s streaming stdout instead, same reasoning as [Part 6](#7-part-6--streams--buffers)).
- **Security:** `exec()` runs its command through a shell, meaning unsanitized user input concatenated into the command string is a **command injection** vulnerability. `spawn()`/`execFile()` with an argument array bypass the shell entirely and are the safer default when any part of the command is derived from user input.

```js
// WRONG — command injection if userInput is attacker-controlled
exec(`convert ${userInput} out.png`);

// RIGHT — arguments passed as an array, no shell interpolation
execFile('convert', [userInput, 'out.png']);
```

### Mastery

- `node:cluster` (mentioned in [Part 7](#8-part-7--networking--http--net)) is built on `child_process.fork()` under the hood — each cluster worker is a full separate process sharing a listening socket, which is why cluster workers don't share in-memory state (a cache, a WebSocket connection map) the way `worker_threads` can via `SharedArrayBuffer` — a frequent source of confusion when someone expects an in-memory rate limiter to work correctly across cluster workers and it silently doesn't.
- `Atomics` paired with `SharedArrayBuffer` across worker threads enables genuine shared-memory concurrency primitives (locks, wait/notify) — the closest Node gets to traditional multithreaded programming, and correspondingly easy to get wrong (races, torn reads) if you're not deliberate about it.

[⬆ back to top](#table-of-contents)

---

## 11. Part 11 — Debugging & Diagnostics

**What it is.** Beyond `console.log`, Node has a built-in inspector protocol (the same one Chrome DevTools speaks), a native test runner, and diagnostic flags for profiling CPU and memory.

### Beginner

```bash
node --inspect script.js        # start with debugger attached; open chrome://inspect
node --inspect-brk script.js    # same, but pause on the first line
```

```js
console.table(arrayOfObjects);   // tabular output, easy to overlook
console.time('label'); /* ... */ console.timeEnd('label'); // quick timing
```

### Working Knowledge

Node's built-in test runner (`node:test`, stable since Node 20) removes the need for a third-party test framework for straightforward cases:

```js
// hash.test.js
const { test } = require('node:test');
const assert = require('node:assert');

test('adds two numbers', () => {
  assert.strictEqual(1 + 1, 2);
});
```

```bash
node --test              # runs all *.test.js files it finds
node --test --watch      # re-run on change
```

### Advanced

**Real Scenario.** A long-running Node service's memory usage climbs steadily over days until it OOMs. The workflow: attach `--inspect`, open `chrome://inspect`, take a heap snapshot early and another after significant growth, and use DevTools' comparison view to find object types accumulating between snapshots (a common culprit: event listeners or timers added per-request that are never removed, or a growing in-memory cache with no eviction). `process.memoryUsage()` logged periodically is a cheap way to *detect* the trend before reaching for full heap snapshots.

- `--prof` generates a V8 CPU profile (`isolate-*.log`); `node --prof-process` turns it into a readable summary — useful for finding hot functions when a service is CPU-bound rather than I/O-blocked.

### Mastery

- `node --inspect` combined with the **diagnostic report** feature (`node --report-uncaught-exception`, or triggered on-demand via `process.report.writeReport()`) produces a structured JSON dump (stack traces of all threads, resource usage, loaded modules) on crash — designed for post-mortem debugging in production where you can't attach a live debugger.
- Clinic.js and 0x are common third-party tools built on top of these same primitives (`--prof`, the inspector protocol) for flame-graph visualization — worth knowing they exist, though not built-in.

[⬆ back to top](#table-of-contents)

---

## 12. Cheat Sheets

### Module system

| Signal | Result |
|---|---|
| File ends in `.mjs` | Always ESM |
| File ends in `.cjs` | Always CommonJS |
| File ends in `.js`, nearest `package.json` has `"type": "module"` | ESM |
| File ends in `.js`, no `"type"` field or `"type": "commonjs"` | CommonJS |
| `.ts` file, erasable syntax only, Node ≥ 22.18/24.3 | Runs natively, no flag |

### Event loop ordering (fastest to slowest priority)

1. Currently executing synchronous code
2. `process.nextTick()` queue (drained fully)
3. Promise microtask queue (drained fully)
4. Next event loop phase (timers → pending → poll → check → close)

### `fs` API style — when to use which

| Style | When |
|---|---|
| `fs.promises` / `async`/`await` | Default choice for anything async |
| `fs.readFileSync` etc. | Startup-only, one-off scripts, never in a request handler |
| `fs.createReadStream`/`createWriteStream` | Large files, anything that shouldn't fully buffer in memory |

### `child_process` vs `worker_threads`

| Need | Use |
|---|---|
| Run an external program | `child_process.spawn`/`execFile` |
| Run another Node script with IPC | `child_process.fork` |
| Offload CPU-bound JS from the main thread | `worker_threads` |
| Share memory between concurrent JS execution | `worker_threads` + `SharedArrayBuffer` |

### npm commands

| Command | Effect |
|---|---|
| `npm install` | Install deps, may update lockfile within semver ranges |
| `npm ci` | Install exactly what the lockfile says; fails on drift — use in CI |
| `npm run <script>` | Run a `package.json` `"scripts"` entry |
| `npx <pkg>` | Run a package's binary without a permanent install |

[⬆ back to top](#table-of-contents)

---

## 13. Suggested Learning Order

1. **Part 1 (Orientation)** — get the runtime installed and understand the V8/libuv split conceptually.
2. **Part 3 (Modules)** — you'll hit CJS/ESM friction immediately in any real code, so front-load it.
3. **Part 2 (Event Loop)** — the conceptual core; revisit this after Parts 5–7 once you have concrete async APIs to anchor it to.
4. **Part 5 (File System)** → **Part 6 (Streams & Buffers)** — these build on each other directly.
5. **Part 4 (`process` & Globals)** — lighter, can be read anytime, but the `AsyncLocalStorage` material lands better after Part 2.
6. **Part 7 (Networking)** — the payoff section; this is "how do frameworks actually work."
7. **Part 8 (Error Handling & Process Lifecycle)** — read this before deploying anything real.
8. **Part 9 (npm)** — mostly reference; skim now, return to the Mastery tier once you're publishing/maintaining packages.
9. **Part 10 (Child Processes & Worker Threads)** — return to this once you have a concrete CPU-bound problem to solve.
10. **Part 11 (Debugging)** — most useful once you have something running you actually need to debug — pair it with a real bug rather than reading cold.

[⬆ back to top](#table-of-contents)

---

## 14. Quick Self-Check

- Can you explain, without looking, why V8 alone isn't enough to make Node non-blocking — what does libuv actually add?
- Given a snippet mixing `setTimeout`, `setImmediate`, `process.nextTick`, and a `Promise.then`, can you predict the exact output order?
- Why does `require(esm)` still throw for some ESM modules even though it's "stable" — what's the specific condition that breaks it?
- Why is `fs.readFileSync` fine in a startup script but wrong inside an Express route handler?
- What does `stream.pipeline()` guarantee that plain `.pipe()` doesn't?
- Why does `exec()` with unsanitized user input create a security vulnerability that `execFile()` doesn't?
- When would you reach for `worker_threads` instead of `child_process`, and what capability does that choice unlock (or give up)?
- What's the difference in practice between `npm install` and `npm ci`, and why does it matter specifically in CI?
- Why should `process.exit()` inside an `uncaughtException` handler be treated as mandatory rather than optional?

[⬆ back to top](#table-of-contents)
