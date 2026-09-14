# Node.js — Production & Diagnostics (Parts 13–14)

What changes when a Node process has to stay up: configuration that fails fast, a shutdown sequence that survives a deploy, memory limits that agree with the container, the permission model, and packaging. Then the tools for when it misbehaves anyway — request context that follows `await`, tracing hooks, loop-delay measurement, CPU profiles, and heap snapshots.

> **Spec:** this doc follows the shared spec in [`nodejs-mastery-guide.md`](./nodejs-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Mastery tiers (collapsed where a tier would be filler, always stated), **Try It** exercises with expected output, wrong-vs-right pairs, official docs cited inline. Written against **Node.js 24 LTS**, verified **September 14, 2026**.
>
> **Prerequisite:** [Part 2 — The Event Loop](./nodejs-runtime-foundations.md#part-2--the-event-loop), [Part 8 — The Process](./nodejs-processes-workers.md#part-8--the-process) and [Part 10 — HTTP Servers](./nodejs-http-networking.md#part-10--http-servers). Part 13 assembles them; Part 14 measures them.

---

## Table of Contents

- [Part 13 — Production Node](#part-13--production-node)
  - [Beginner: `NODE_ENV`, config, and failing fast](#p13-beginner)
  - [Working Knowledge: logging, and a shutdown that survives deploys](#p13-working)
  - [Advanced: containers — PID 1, signals, and memory limits](#p13-advanced)
  - [Mastery: the permission model, single executables, and security hygiene](#p13-mastery)
  - [Part 13 cheat sheet](#part-13-cheat-sheet)
- [Part 14 — Diagnostics](#part-14--diagnostics)
  - [Beginner: what to measure first](#p14-beginner)
  - [Working Knowledge: `AsyncLocalStorage`](#p14-working)
  - [Advanced: CPU profiles and the blocked loop](#p14-advanced)
  - [Mastery: heap snapshots, `diagnostics_channel`, and diagnostic reports](#p14-mastery)
  - [Part 14 cheat sheet](#part-14-cheat-sheet)

---

## Part 13 — Production Node

None of this Part is exotic. It is the set of things that separate a Node service that deploys cleanly from one that drops requests on every release and restarts at 3am with an out-of-memory kill nobody can explain.

<a id="p13-beginner"></a>

### Beginner: `NODE_ENV`, config, and failing fast

#### `NODE_ENV=production` still matters

Node itself mostly ignores `NODE_ENV`, but the ecosystem does not: frameworks and libraries use it to disable development checks, verbose errors and template recompilation, sometimes with large performance differences.

```dockerfile
ENV NODE_ENV=production
```

The misuse to avoid is treating it as your deployment-environment switch:

```js
// ✗ wrong — staging and production both run NODE_ENV=production; this can't tell them apart
const dbUrl = process.env.NODE_ENV === 'production' ? PROD_DB : STAGING_DB;

// ✓ right — NODE_ENV says "optimised build"; the config itself says where you are
const dbUrl = process.env.DATABASE_URL;
```

`NODE_ENV` should only ever mean "is this an optimised build". Everything that differs between environments belongs in its own variable.

#### Validate config once, at startup, and crash if it's wrong

```js
// config.mjs
function required(name) {
  const v = process.env[name];
  if (v === undefined || v === '') throw new Error(`Missing required env var: ${name}`);
  return v;
}

export const config = Object.freeze({
  port: Number(process.env.PORT ?? 3000),
  databaseUrl: required('DATABASE_URL'),
  logLevel: process.env.LOG_LEVEL ?? 'info',
});

if (!Number.isInteger(config.port)) throw new Error(`Invalid PORT: ${process.env.PORT}`);
```

This is the Node form of a general principle, and it's worth stating strongly: **a service that is missing its configuration should refuse to start**, not start and fail on the first request that happens to need the missing value — possibly hours later, possibly only for one customer. A crash at boot fails the deploy, which is precisely where you want the failure to be.

In TypeScript, a schema library (Zod, Valibot) gives you parsing, coercion and a typed `config` object in the same few lines. The principle is identical; the types are a bonus.

Locally, load the variables with `--env-file` from [Part 1](./nodejs-runtime-foundations.md#p1-working). In production, let the platform inject them — and never bake a `.env` containing secrets into an image, where anyone who can pull the image can read it.

<a id="p13-working"></a>

### Working Knowledge: logging, and a shutdown that survives deploys

#### Structured logs, to stdout

```js
// ✗ wrong — unparseable, unsearchable, and synchronous to a TTY
console.log(`User ${id} failed to pay: ${err}`);

// ✓ right — one JSON object per line
log.error({ userId: id, err }, 'payment failed');
// {"level":50,"time":1757851200000,"userId":"u_42","err":{"type":"CardDeclined","message":"…","stack":"…"},"msg":"payment failed"}
```

Three rules, independent of which logger you pick (pino is the performance-focused common choice):

1. **JSON, one object per line.** Every log platform parses it; nobody can reliably parse free text.
2. **Write to stdout.** The container runtime collects it. Log files inside a container fill the disk and die with the container.
3. **Log the error object, not `String(err)`.** The stack trace and `err.cause` are the parts you'll need, and string interpolation throws both away.

A note on `console.log` specifically: when stdout is a pipe (as it is in every container) writes are asynchronous on POSIX, but when it's a file or a TTY they're synchronous, and on Windows pipes they're synchronous too. A service logging heavily with `console.log` can therefore block its event loop in some environments and not others. Dedicated loggers exist largely to avoid that.

#### The complete shutdown sequence

[Part 8](./nodejs-processes-workers.md#p8-working) introduced signals, and [Part 10](./nodejs-http-networking.md#p10-advanced) the server-side close. Assembled, with the piece that both of them omit:

```js
import { setTimeout as sleep } from 'node:timers/promises';

let shuttingDown = false;

// health check the load balancer polls
function health(req, res) {
  res.statusCode = shuttingDown ? 503 : 200;
  res.end();
}

async function shutdown(signal) {
  if (shuttingDown) return;
  shuttingDown = true;
  log.info({ signal }, 'shutdown started');

  // 1. fail the readiness check, and WAIT for the load balancer to notice
  await sleep(Number(process.env.SHUTDOWN_DELAY_MS ?? 5_000));

  // 2. stop accepting connections; finish in-flight requests
  server.close();
  const force = setTimeout(() => server.closeAllConnections(), 10_000).unref();
  await once(server, 'close');
  clearTimeout(force);

  // 3. release everything else
  await Promise.allSettled([db.end(), queue.close(), cache.quit()]);
  await log.flush?.();

  log.info('shutdown complete');
}

process.once('SIGTERM', () => shutdown('SIGTERM').catch(() => process.exit(1)));
process.once('SIGINT',  () => shutdown('SIGINT').catch(() => process.exit(1)));
```

Step 1 is the one that is missing from nearly every tutorial. When an orchestrator decides to stop a pod, it sends `SIGTERM` and removes the pod from the load balancer **concurrently**, and the load balancer takes seconds to act on that. A server that closes its listener instantly on `SIGTERM` refuses connections the balancer is still sending it — which shows up as a burst of 502s on every deploy, exactly as regular as your release schedule.

**The timing budget has to fit the orchestrator's grace period** (Kubernetes defaults to 30 s): readiness delay + drain timeout + resource cleanup must finish before `SIGKILL` arrives, or the ending is abrupt anyway.

**Real Scenario — errors on every deploy, and only on deploy.** A team's dashboards show a spike of 502s at the same minute every afternoon. It lines up exactly with the daily release. The service handles `SIGTERM` correctly by the textbook — `server.close()`, await, exit — and the logs confirm a clean shutdown each time. The failures are in the load balancer's logs, not the application's: for about three seconds after each `SIGTERM`, the balancer kept routing to pods that had already closed their listening sockets. Adding a five-second readiness-failure delay before `server.close()` removed the spike entirely, with no other change.

<a id="p13-advanced"></a>

### Advanced: containers — PID 1, signals, and memory limits

#### Don't let `npm` be PID 1

```dockerfile
# ✗ wrong — npm becomes PID 1 and does not forward SIGTERM to your process.
#   Every stop waits out the full grace period, then SIGKILLs mid-request.
CMD ["npm", "start"]

# ✗ also wrong — the shell form wraps your command in /bin/sh -c, which also swallows signals
CMD node server.js

# ✓ right — exec form, node is PID 1 and receives signals directly
CMD ["node", "server.js"]
```

Two separate problems are hiding here. Wrappers (`npm`, `sh -c`) don't reliably forward signals to their child, so your carefully written `SIGTERM` handler from above never runs. And a process running as PID 1 gets special kernel treatment: signals with no handler installed are *ignored* rather than taking their default action, and orphaned child processes become its responsibility to reap.

The robust answer is a tiny init process in front of Node:

```bash
docker run --init my-image          # Docker's bundled tini
```

or `tini` installed in the image as the `ENTRYPOINT`. It forwards signals and reaps zombies, and it makes the rest of this section hold even if someone changes the `CMD` later.

#### A production Dockerfile, annotated

```dockerfile
FROM node:24-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev                     # reproducible, production deps only (Part 5)

FROM node:24-slim
WORKDIR /app
ENV NODE_ENV=production
COPY --from=deps /app/node_modules ./node_modules
COPY . .
USER node                                 # never run as root; the image ships this user
EXPOSE 3000
CMD ["node", "--enable-source-maps", "server.js"]
```

The layer order is the performance trick: dependency installation is cached until `package.json` or the lockfile changes, so an ordinary code change rebuilds in seconds. `USER node` is the security one — a compromised dependency running as root in a container is a much larger incident than the same dependency running unprivileged.

#### Memory: make the heap limit agree with the container

V8 picks a default heap limit based on the memory it detects. The failure mode is a heap limit *above* the container limit: V8 believes it has room, never runs a hard GC, and the kernel's OOM killer terminates the process with no JavaScript error and no stack trace — just `exit 137` and a restart.

Set it explicitly, leaving headroom for everything that isn't the JS heap (Buffers, native memory, the thread pool, code):

```dockerfile
# container limit 1 GiB → heap ~75% of it
ENV NODE_OPTIONS="--max-old-space-size=768"
```

Then verify rather than assume — this is the number V8 is actually using:

```bash
node -p "v8.getHeapStatistics().heap_size_limit / 1024 / 1024"
```

**The distinction to carry into Part 14:** `JavaScript heap out of memory` in your logs means V8 hit *its* limit — a leak, or a limit set too low. A silent `137` / `OOMKilled` with nothing logged means the *container* hit its limit first — the heap limit is set too high, or the growth is outside the heap (usually Buffers). They are different problems with different fixes, and telling them apart is the first step.

```bash
# get a heap snapshot automatically as the process approaches its limit, for the post-mortem
node --heapsnapshot-near-heap-limit=1 --max-old-space-size=768 server.js
```

<a id="p13-mastery"></a>

### Mastery: the permission model, single executables, and security hygiene

#### The permission model

Node can restrict what a process may touch. **It is stable on Node 24, and the flag is `--permission`** — renamed from `--experimental-permission`, which is what most existing articles still show ([permissions docs](https://nodejs.org/docs/latest-v24.x/api/permissions.html)):

```bash
node --permission \
     --allow-fs-read=/app \
     --allow-fs-write=/app/uploads \
     server.js
```

With `--permission` on, everything not explicitly allowed is denied: filesystem reads and writes, `child_process`, `worker_threads`, native addons, WASI and the inspector. A denied operation throws `ERR_ACCESS_DENIED`. Code can check before trying:

```js
if (!process.permission.has('fs.write', '/app/uploads')) { /* degrade */ }
```

There is also an audit mode that logs violations without enforcing them, which is the practical way to adopt this on an existing service — run in audit mode, collect what the service actually touches, then write the allow-list:

```bash
node --permission-audit server.js
```

**What it is not.** The docs are unusually direct about the limits, and they deserve quoting in substance: Node "trusts any code it is asked to run", so this is a seat belt for trusted code, not a sandbox for hostile code. The documented gaps include ([permissions docs](https://nodejs.org/docs/latest-v24.x/api/permissions.html)):

- **symbolic links are followed** even when they point outside the granted paths;
- **permissions are not inherited by worker threads**;
- **operations on an already-open file descriptor bypass the checks**;
- **flags processed before the model initialises** (`--env-file`, `--openssl-config`, V8 flags) read files regardless.

So: a good defence-in-depth measure that meaningfully shrinks what a compromised dependency can reach, and not a substitute for a container, a VM, or not running untrusted code.

#### Single executable applications

Node can embed a script into a copy of its own binary, producing one file that runs with no Node installed ([SEA docs](https://nodejs.org/docs/latest-v24.x/api/single-executable-applications.html)):

```json
// sea-config.json
{
  "main": "dist/cli.cjs",
  "output": "sea-prep.blob",
  "disableExperimentalSEAWarning": true,
  "useCodeCache": true,
  "assets": { "templates/default.txt": "templates/default.txt" }
}
```

```bash
node --experimental-sea-config sea-config.json      # build the blob
cp "$(command -v node)" mycli                         # copy the node binary
npx postject mycli NODE_SEA_BLOB sea-prep.blob \
  --sentinel-fuse NODE_SEA_FUSE_fce680ab2cc467b6e072b8b5df1996b2   # inject
```

On macOS you then re-sign the binary; the SEA docs give the exact `codesign` steps.

The constraints decide whether this fits, so read them first. SEA is still **active development** (stability 1.1) on Node 24. The entry point must be a **single CommonJS file** — so in practice you bundle first (esbuild, Rolldown, or `bun build`) — and `require()` inside it can only load built-in modules. Extra files go in `assets` and are read with `sea.getAsset()`. `useCodeCache` and `useSnapshot` must be off for cross-platform builds.

> **vs. Bun** — `bun build --compile` produces a single executable in one command, supports ESM and TypeScript directly, and can cross-compile for other platforms. For shipping a CLI binary it is currently the much smoother path; Node's SEA wins when you need the exact Node runtime semantics in the output. See [`bun-testing-bundling.md`](../bun/bun-testing-bundling.md).

#### Security hygiene, as a checklist

Most Node security incidents are not Node-specific. The Node-shaped ones, each covered elsewhere in the set:

| Risk | Defence | Where |
|---|---|---|
| Shell injection | `execFile`/`spawn`, never `exec` with input | [Part 8](./nodejs-processes-workers.md#p8-advanced) |
| ReDoS | Simple regexes, no nested quantifiers on user input | [Part 2](./nodejs-runtime-foundations.md#p2-advanced) |
| Unbounded request bodies | Enforce a size limit | [Part 10](./nodejs-http-networking.md#p10-working) |
| Slowloris | Leave `headersTimeout`/`requestTimeout` non-zero | [Part 10](./nodejs-http-networking.md#p10-advanced) |
| Exposed inspector | Never `--inspect=0.0.0.0` | [Part 12](./nodejs-testing-debugging.md#p12-mastery) |
| Malicious dependency | `npm ci`, lockfile, release-age gate, `--ignore-scripts` | [Part 5](./nodejs-modules-packages.md#p5-mastery) |
| Disabled TLS validation | `NODE_EXTRA_CA_CERTS`, never `rejectUnauthorized: false` | [Part 11](./nodejs-http-networking.md#p11-mastery) |
| Prototype pollution | Don't merge untrusted JSON into objects; `Object.create(null)` for maps | — |
| Leaking `Buffer.allocUnsafe` memory | Only when every byte is immediately overwritten | [Part 6](./nodejs-files-streams-buffers.md#p6-advanced) |
| Running as root | `USER node` | [above](#p13-advanced) |
| Outdated runtime | Stay on a supported LTS; security releases land there | [Overview §3](./nodejs-mastery-guide.md#3-orientation--setup) |

<a id="part-13-cheat-sheet"></a>

### Part 13 cheat sheet

| Want | How |
|---|---|
| Optimised library behaviour | `NODE_ENV=production` — and use it for nothing else |
| Safe config | Validate every variable at startup; throw if invalid |
| Logs | JSON lines to stdout; log the error object |
| Zero-downtime shutdown | Fail readiness → wait → `server.close()` → drain → close resources |
| Correct container command | `CMD ["node", "server.js"]` (exec form) |
| Signals + zombie reaping | `docker run --init` or `tini` |
| Don't run as root | `USER node` |
| Reproducible deps in the image | `npm ci --omit=dev` |
| Heap limit that fits the container | `--max-old-space-size` ≈ 75% of the limit |
| Check the real heap limit | `v8.getHeapStatistics().heap_size_limit` |
| Snapshot before OOM | `--heapsnapshot-near-heap-limit=1` |
| Restrict filesystem/process access | `--permission --allow-fs-read=/app` |
| Discover what it touches | `--permission-audit` |
| Check a permission in code | `process.permission.has('fs.write', path)` |
| One-file binary | SEA: `--experimental-sea-config` + `postject` *(active development)* |
| Readable production stack traces | `--enable-source-maps` |

**Gotchas:** `CMD ["npm", "start"]` swallows `SIGTERM` · closing the listener before the load balancer notices causes deploy-time 502s · a heap limit above the container limit produces silent `137` kills · the flag is `--permission`, not `--experimental-permission` · the permission model follows symlinks and doesn't cover workers · SEA needs a single CommonJS bundle.

[↑ Back to top](#table-of-contents)

---

## Part 14 — Diagnostics

The previous thirteen Parts are about making Node behave. This one is about finding out why it didn't. The tooling is better than its reputation — the problem is usually knowing which of the handful of tools answers which question.

<a id="p14-beginner"></a>

### Beginner: what to measure first

Every production Node problem shows up as one of four symptoms, and each has a first measurement:

```
   Symptom                          First measurement                  Tool
   ───────────────────────────────  ─────────────────────────────────  ─────────────────────────
   slow / latency spikes        ──► event loop delay                ──► perf_hooks histogram
   high CPU                     ──► where the CPU time goes         ──► --cpu-prof → flame chart
   memory grows                 ──► what is being retained          ──► heap snapshots, compared
   crashes / hangs              ──► process state at the moment     ──► diagnostic report
```

Before any of those, the process's own view of itself costs nothing to collect:

```js
setInterval(() => {
  const m = process.memoryUsage();
  log.info({
    rssMb: Math.round(m.rss / 1e6),               // total process memory — what the container sees
    heapUsedMb: Math.round(m.heapUsed / 1e6),     // live JS objects
    externalMb: Math.round(m.external / 1e6),     // Buffers and other off-heap memory
    handles: process.getActiveResourcesInfo().length,
  }, 'process stats');
}, 30_000).unref();
```

The **gap between `rss` and `heapUsed`** is the diagnostic in that snippet. If `heapUsed` grows, it's a JavaScript leak and heap snapshots will find it. If `rss` grows while `heapUsed` stays flat, the growth is outside the JS heap — Buffers (`external`), native addons, or memory fragmentation — and heap snapshots will show you nothing, which is a very expensive way to find that out.

<a id="p14-working"></a>

### Working Knowledge: `AsyncLocalStorage`

[Part 3](./nodejs-runtime-foundations.md#p3-mastery) introduced this as the tool for context that follows `await`. It belongs in the diagnostics Part because its main job in practice is making every log line and trace span attributable to a request.

```js
// context.mjs
import { AsyncLocalStorage } from 'node:async_hooks';
export const requestContext = new AsyncLocalStorage();

// server.mjs
createServer((req, res) => {
  const store = { requestId: req.headers['x-request-id'] ?? crypto.randomUUID(), start: performance.now() };
  requestContext.run(store, () => handle(req, res));
});

// logger.mjs — every log line now carries the request id, with no parameter threading
import pino from 'pino';
export const log = pino({
  mixin: () => ({ requestId: requestContext.getStore()?.requestId }),
});
```

Everything called from inside `run()` — synchronously, after any number of `await`s, inside timers and event callbacks started from that request — sees the same store. That's what makes it possible to grep one request's entire journey out of a busy log.

#### Where context gets lost

Context propagates through promises, timers and Node's own async APIs automatically. It breaks where something manages its own queue of callbacks and runs them later from somewhere else:

```js
// ✗ context lost — a pooled connection's callback runs in whichever context created the pool
legacyPool.query(sql, (err, rows) => {
  requestContext.getStore();   // undefined, or worse: ANOTHER request's store
});

// ✓ bind the callback to the current context
import { AsyncResource } from 'node:async_hooks';
legacyPool.query(sql, AsyncResource.bind((err, rows) => {
  requestContext.getStore();   // correct
}));
```

"Another request's store" is the dangerous version: logs attributed to the wrong user are worse than logs attributed to none. Promise-based libraries almost never have this problem; old callback-based connection pools and custom event queues sometimes do.

**Node 24 changed the implementation.** `AsyncLocalStorage` now uses `AsyncContextFrame` by default ([Node 24 release notes](https://nodejs.org/en/blog/release/v24.0.0)), which is faster and removes much of the historical overhead that led people to avoid it. The change is behaviour-preserving for correct code, but hand-rolled context propagation built on `async_hooks` internals, and some older APM agents, behaved differently after upgrading — verify yours on 24 rather than assuming.

**Try It — prove context survives `await`.**

```js
// als.mjs
import { AsyncLocalStorage } from 'node:async_hooks';
import { setTimeout as sleep } from 'node:timers/promises';

const als = new AsyncLocalStorage();

async function deep() {
  await sleep(Math.random() * 50);
  return als.getStore();
}

await Promise.all([1, 2, 3].map((id) => als.run({ id }, async () => {
  await sleep(Math.random() * 50);
  console.log('request', id, 'sees', await deep());
})));
```

Expected output: three lines in an unpredictable *order*, but each one pairing correctly — `request 2 sees { id: 2 }` and so on. Three interleaved async flows, random delays, and not one mix-up.

<a id="p14-advanced"></a>

### Advanced: CPU profiles and the blocked loop

#### Is the loop blocked? Measure it continuously

The `monitorEventLoopDelay` sampler from [Part 2](./nodejs-runtime-foundations.md#p2-mastery), exported as a metric, is the single most useful production number for a Node service. Pair it with event-loop *utilisation*, which answers a different question — not "how long did callbacks wait" but "what fraction of the time was the thread busy":

```js
import { monitorEventLoopDelay, performance } from 'node:perf_hooks';

const delay = monitorEventLoopDelay({ resolution: 10 });
delay.enable();
let lastElu = performance.eventLoopUtilization();

setInterval(() => {
  const elu = performance.eventLoopUtilization(lastElu);
  lastElu = performance.eventLoopUtilization();
  metrics.gauge('eventloop.delay.p99_ms', delay.percentile(99) / 1e6);
  metrics.gauge('eventloop.utilization', elu.utilization);   // 0..1
  delay.reset();
}, 10_000).unref();
```

Reading the pair:

| Delay p99 | Utilisation | Meaning |
|---|---|---|
| Low | Low | Healthy, idle capacity |
| Low | High | Busy but keeping up — scale before it isn't |
| **High** | **Low** | Occasional long synchronous blocks — **find them with a profile** |
| High | High | Saturated — too much work, or everything is slow |

The third row is the classic Node bug — mostly idle, but a single `JSON.parse` of a huge payload or a catastrophic regex freezes everything periodically — and averages hide it completely.

#### CPU profiles

```bash
node --cpu-prof server.js
# … exercise it, then stop the process …
# → CPU.20260914.103012.48213.0.001.cpuprofile in the working directory
```

Open the `.cpuprofile` in Chrome DevTools (**Performance** panel → load profile) or in VS Code, and read the flame chart:

```
   ┌──────────────────────── handleRequest ─────────────────────────┐
   │ ┌────── parseBody ──────┐ ┌──────────── renderReport ────────┐ │
   │ │ ┌─── JSON.parse ───┐  │ │ ┌── formatRows ──┐ ┌─ sortBy ─┐   │ │
   │ │ └──────────────────┘  │ │ └────────────────┘ └──────────┘   │ │
   │ └───────────────────────┘ └───────────────────────────────────┘ │
   └─────────────────────────────────────────────────────────────────┘
     width = time on CPU                     look for WIDE bars with
     vertical = call stack depth             NOTHING on top of them
```

Width is time. A wide bar with nothing stacked above it is a function spending its time in its own code — that's your hot spot. A wide bar with a tall stack above it is just a caller; keep looking upward.

For a running process you don't want to restart, attach the inspector (see the security rule in [Part 12](./nodejs-testing-debugging.md#p12-mastery)) and record from the DevTools **Performance** panel, or use `inspector.Session` from code to start and stop a profile on demand — typically behind an authenticated admin endpoint.

**Real Scenario — p99 of two seconds, CPU at 15%.** An API's median latency is 20 ms and its p99 is over 2 s, but CPU graphs look idle and nothing in the logs stands out. Event-loop delay p99 matches the latency p99 almost exactly — so something is occupying the thread, briefly and rarely. A CPU profile captured during a load test shows a wide `RegExpExecInternal` bar under a validation helper: an email regex with nested quantifiers, backtracking catastrophically on a handful of malformed addresses a client was sending. Median requests never hit it; the rare one froze *every* in-flight request for two seconds. The fix was a simpler pattern. The lesson is the reading order: loop delay pointed at blocking, the profile named the function, and neither metric alone would have got there.

<a id="p14-mastery"></a>

### Mastery: heap snapshots, `diagnostics_channel`, and diagnostic reports

#### Heap snapshots

A snapshot is a complete graph of every object on the JS heap and what holds a reference to it. There are several ways to take one ([heap snapshot guide](https://nodejs.org/en/learn/diagnostics/memory/using-heap-snapshot)):

```bash
node --heapsnapshot-signal=SIGUSR2 server.js   # then: kill -USR2 <pid>
```

```js
import { writeHeapSnapshot } from 'node:v8';
writeHeapSnapshot();   // returns the filename it wrote
```

plus the inspector's **Memory** tab, and `--heapsnapshot-near-heap-limit` from [Part 13](#p13-advanced) for automatic capture before a crash.

**The costs are serious, and the guide is explicit about them:** taking a snapshot **blocks the main thread** for the whole duration — which can exceed a minute on a large heap — and **needs roughly as much memory again as the heap itself**, which can push a process that is already close to its limit into an OOM crash. On a live service, take snapshots on an instance you can afford to lose, after removing it from the load balancer.

The technique that finds leaks is **comparison**:

1. Start the process and let it warm up.
2. Exercise the suspect feature a little, then take **snapshot 1**.
3. Exercise it a lot more — hundreds or thousands of iterations.
4. Take **snapshot 2**.
5. In DevTools' **Memory** tab, load both, select the second, and switch the view from **Summary** to **Comparison**.
6. Sort by **# Delta** or **Size Delta**. Whatever grew in proportion to your iterations is the leak; expand it and follow the **Retainers** panel up to the thing holding it.

The retainer chain almost always ends somewhere unglamorous: a module-level `Map` used as a cache with no eviction, listeners added per request and never removed (which the [max-listeners warning](./nodejs-runtime-foundations.md#p3-working) was trying to tell you about), a closure captured by a long-lived timer, or an `AsyncLocalStorage` store kept alive by a stray promise.

#### `diagnostics_channel` — observing without monkey-patching

For years, APM and tracing tools worked by monkey-patching `http.request` and friends, which was fragile and order-dependent. `diagnostics_channel` is the replacement: libraries *publish* named events, and observers *subscribe*, with near-zero cost when nobody is listening ([diagnostics_channel docs](https://nodejs.org/docs/latest-v24.x/api/diagnostics_channel.html)):

```js
import diagnostics_channel from 'node:diagnostics_channel';

diagnostics_channel.subscribe('http.server.request.start', ({ request }) => {
  request.__start = performance.now();
});

diagnostics_channel.subscribe('http.server.response.finish', ({ request, response }) => {
  metrics.histogram('http.duration_ms', performance.now() - request.__start, {
    method: request.method,
    status: response.statusCode,
  });
});
```

Request timing for every server in the process, with no changes to any handler. The module is stable, but mind the fine print: most of the **built-in channel names** — HTTP, net, child process, modules, workers — are individually marked **experimental**, so pin to a Node version when you depend on a specific one.

Publishing from your own library costs nothing when unobserved, which is the point of the design:

```js
const queryChannel = diagnostics_channel.tracingChannel('mydb.query');

export function query(sql, params) {
  return queryChannel.tracePromise(() => runQuery(sql, params), { sql });
}
// subscribers receive start / end / asyncStart / asyncEnd / error for every query
```

`TracingChannel` itself is still experimental. OpenTelemetry's Node instrumentations are moving onto these channels, which is the main reason to understand them even if you never subscribe to one directly.

#### Diagnostic reports — the post-mortem you didn't know to ask for

A diagnostic report is a JSON document describing the process at a moment: JS and native stack traces, heap statistics, active handles, loaded shared libraries, environment variables, resource usage, and libuv's handle list.

```bash
node --report-on-fatalerror --report-uncaught-exception \
     --report-directory=/var/log/node-reports server.js
```

```js
process.report.writeReport();   // on demand, e.g. from a SIGUSR2 handler
```

For crashes that leave nothing useful in the logs — a native assertion failure, an OOM where the heap snapshot was too expensive to take — the report is often the only evidence. Two cautions: it includes **environment variables**, so treat report files as containing secrets; and the libuv handle list at the bottom is exactly what you want when diagnosing "why won't this process exit".

#### Putting it together

```
   metric alert
        │
        ├── latency ↑ ──► loop delay ↑?
        │                    ├── yes ─► CPU profile ─► the blocking function
        │                    └── no ──► it's downstream: DB, network, pool exhaustion
        │                               (Part 11: unconsumed bodies, dns.lookup)
        │
        ├── memory ↑ ───► heapUsed ↑?
        │                    ├── yes ─► 2 heap snapshots ─► Comparison ─► Retainers
        │                    └── no ──► rss/external ↑ ─► Buffers, streams, native
        │                               (Part 7: ignored backpressure)
        │
        └── crash ──────► exit 137, nothing logged?
                             ├── yes ─► container OOM ─► heap limit vs container limit (Part 13)
                             └── no ──► diagnostic report + the uncaught error
```

<a id="part-14-cheat-sheet"></a>

### Part 14 cheat sheet

| Want | How |
|---|---|
| Heap vs. off-heap memory | `process.memoryUsage()` — compare `heapUsed` with `rss` and `external` |
| What's keeping the process alive | `process.getActiveResourcesInfo()` |
| Per-request context | `AsyncLocalStorage` → `run(store, fn)` / `getStore()` |
| Fix lost context in callback APIs | `AsyncResource.bind(fn)` |
| Is the loop blocked? | `monitorEventLoopDelay()` p99 |
| How busy is the thread? | `performance.eventLoopUtilization()` |
| CPU profile | `node --cpu-prof app.js` → Chrome DevTools |
| Heap snapshot on signal | `--heapsnapshot-signal=SIGUSR2` |
| Heap snapshot from code | `v8.writeHeapSnapshot()` |
| Snapshot before OOM | `--heapsnapshot-near-heap-limit=1` |
| Find a leak | Two snapshots → **Comparison** view → **Retainers** |
| Observe `http` without patching | `diagnostics_channel.subscribe('http.server.request.start', …)` |
| Instrument your own library | `tracingChannel('name').tracePromise(fn)` *(experimental)* |
| Post-mortem on crash | `--report-on-fatalerror --report-uncaught-exception` |
| Report on demand | `process.report.writeReport()` |

**Gotchas:** heap snapshots block the thread and roughly double memory · growing `rss` with flat `heapUsed` is invisible to heap snapshots · callback-based pools can leak `AsyncLocalStorage` context across requests · most built-in diagnostics channel names are still experimental · diagnostic reports contain environment variables.

[↑ Back to top](#table-of-contents)

---

**Previous:** [Part 12 — `node:test` & Debugging](./nodejs-testing-debugging.md) · **Up:** [Overview](./nodejs-mastery-guide.md) · **Self-check:** [Overview §9](./nodejs-mastery-guide.md#9-quick-self-check)

*Written against Node.js 24 LTS · Verified September 14, 2026*
