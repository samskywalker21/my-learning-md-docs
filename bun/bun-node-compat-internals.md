# Bun — Node Compat & Internals (Part 13)

Why Bun behaves the way it does, where its Node compatibility genuinely stops, and how to look inside a running process.

> **Spec:** this doc follows the shared spec in [`bun-mastery-guide.md`](./bun-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Mastery tiers (collapsed where a tier would be filler, always stated), **Try It** exercises with expected output, wrong-vs-right pairs, official docs cited inline. Written against **Bun v1.4.2**, verified **September 6, 2026**.
>
> **This is the doc that rots fastest.** Bun ships compatibility fixes nearly every release; several 🟡 entries below will be 🟢 by the time you read this. Treat the *shape* of the gaps as durable and re-check any specific row against the [current compatibility page](https://bun.sh/docs/runtime/nodejs-apis) before making a decision on it.

---

## Table of Contents

- [Part 13 — Compatibility & Internals](#part-13--compatibility--internals)
  - [Beginner: what "Node-compatible" actually claims](#p13-beginner)
  - [Working Knowledge: the support matrix](#p13-working)
  - [Advanced: the gaps that will actually bite you](#p13-advanced)
  - [Advanced: debugging and profiling](#p13-debug)
  - [Mastery: JavaScriptCore vs. V8](#p13-mastery)
  - [Mastery: a migration decision checklist](#p13-checklist)
  - [Part 13 cheat sheet](#part-13-cheat-sheet)

---

## Part 13 — Compatibility & Internals

<a id="p13-beginner"></a>

### Beginner: what "Node-compatible" actually claims

Bun aims to be a drop-in Node replacement, and for most application code it is: `node:` modules, `process`, `Buffer`, npm packages, and the CommonJS/ESM machinery are all implemented. Express and Next.js run ([Node compat](https://bun.sh/docs/runtime/nodejs-apis)).

The claim is measured against **Node's own test suite**, which is the honest way to do it — Bun 1.4 reported over 1,500 newly passing Node tests, with `node:http`, `node:fs`, and `node:sqlite` in the 97–100% range ([Bun 1.4](https://bun.com/blog/bun-v1.4)).

What "97%" means in practice is the whole subject of this Part: the missing 3% is not randomly distributed. It clusters in TLS, crypto, and process-management corners — exactly where a production service eventually goes.

<a id="p13-working"></a>

### Working Knowledge: the support matrix

**Fully or near-fully supported** ([Node compat](https://bun.sh/docs/runtime/nodejs-apis)):

`node:assert` · `node:buffer` · `node:console` · `node:path` · `node:url` · `node:util` · `node:querystring` · `node:punycode` · `node:string_decoder` · `node:fs` · `node:stream` · `node:dgram` · `node:dns` · `node:http` · `node:http2` · `node:net` · `node:zlib` · `node:timers` · `node:events` · `node:readline` · `node:vm` · `node:trace_events` · `node:quic` · `node:sqlite`

**Partially supported — the ones with real gaps:**

| Module | What's missing |
|---|---|
| `node:crypto` | `ed448`, `x448`, `rsa-pss`, `secp256k1`; CCM/OCB/XTS ciphers. **Backed by BoringSSL, not OpenSSL** |
| `node:tls` | OCSP stapling, cross-process session resumption, renegotiation |
| `node:https` | SNI callback, session resumption, some certificate methods |
| `node:child_process` | Cannot pass HTTP server handles; `advanced` serialization is Bun-to-Bun only |
| `node:cluster` | HTTP load balancing **only on Linux**; the docs say it is not battle-tested |
| `node:worker_threads` | `resourceLimits` ignored; no `moveMessagePortToContext` |
| `node:async_hooks` | `AsyncLocalStorage` works; **`createHook` is a stub** — async ids are always 0 |
| `node:v8` | No `startCpuProfile` / `takeCoverage` — use `bun:jsc` |
| `node:test` | Runs under `bun test`, but no snapshots, no `mock.module()`, no coverage |
| `node:inspector` | Partial domain support |
| `node:module` | No `registerHooks`, no `stripTypeScriptTypes`; `findSourceMap` returns undefined |
| `node:wasi` | Missing `getImportObject()`, `initialize()`, `sock_accept` |
| `node:process` | `getActiveResourcesInfo()` returns an empty array |

**Not implemented:**

- **`node:sea`** — by design; `bun build --compile` is the answer ([Part 12](./bun-testing-bundling.md#p12-compile)).
- **`node:domain`** — catches synchronous errors only. It's deprecated in Node too.

**Global gaps worth knowing:**

| Global | Status |
|---|---|
| `localStorage` / `sessionStorage` / `Storage` | 🔴 Not implemented |
| `navigator` | 🟡 Only `userAgent`, `platform`, `hardwareConcurrency` |
| `Request` | 🟡 No `keepalive`, no `duplex` |
| `performance` | 🟡 `eventLoopUtilization()` returns zeros; `nodeTiming` is a placeholder |

<a id="p13-advanced"></a>

### Advanced: the gaps that will actually bite you

Ranked by how likely they are to reach production before you notice.

#### 1. `async_hooks.createHook` is a stub

`AsyncLocalStorage` works, and that covers request-scoped context — the common case. But `createHook`, with async ids always `0`, means **any library built on async-hook instrumentation cannot track async context**. That includes some APM agents and distributed-tracing integrations.

The symptom is not a crash. It is traces that lose their parent-child relationships, or a correlation ID that goes missing across an `await`. That is a subtle, late-discovered failure — check your observability vendor's Bun support explicitly rather than assuming.

```ts
// ✅ this is the supported pattern for request context
import { AsyncLocalStorage } from "node:async_hooks";
const ctx = new AsyncLocalStorage<{ requestId: string }>();

Bun.serve({
  fetch: (req) => ctx.run({ requestId: crypto.randomUUID() }, () => handle(req)),
});
```

#### 2. `node:crypto` is BoringSSL, not OpenSSL

Most code never notices. Code that does: anything using `ed448`/`x448`/`secp256k1` (some blockchain and WebAuthn libraries), or CCM/OCB/XTS ciphers. These are missing, not slow — a hard failure at the point of use, which at least is loud.

#### 3. `node:cluster` load balancing is Linux-only

The classic Node scale-out pattern — fork one worker per core and share a listening socket — has HTTP load balancing only on Linux, and the docs describe it as not battle-tested. On macOS your workers may not distribute connections the way you expect, so a local benchmark can mislead you badly about production behaviour.

The Bun-native alternative is `Bun.serve` with `reusePort: true`, letting the kernel distribute connections across independent processes ([cluster guide](https://bun.com/guides/http/cluster)):

```ts
// server.ts — run one of these per core
Bun.serve({ port: 8080, reusePort: true, fetch: () => new Response("hi") });
```

```ts
// spawn.ts
for (let i = 0; i < navigator.hardwareConcurrency; i++) {
  Bun.spawn({ cmd: ["bun", "./server.ts"], stdout: "inherit" });
}
```

Note this shares the same platform caveat: `SO_REUSEPORT` load balancing is a Linux facility, and the guide states Windows and macOS ignore the option (a limitation of those operating systems, not of Bun). Newer changelog entries suggest macOS support has improved — verify on your target platform rather than trusting either claim, including this one.

#### 4. Console output bypasses `process.stdout.write`

Bun's `console` writes **directly to the file descriptor**. So the classic test trick of monkey-patching `process.stdout.write` to capture output does not capture `console.log`:

```ts
// ❌ captures nothing under Bun
const chunks: string[] = [];
process.stdout.write = ((c: string) => { chunks.push(c); return true; }) as any;
console.log("hello");
// chunks is empty

// ✅ spy on the console method itself
import { spyOn } from "bun:test";
const spy = spyOn(console, "log");
console.log("hello");
expect(spy).toHaveBeenCalledWith("hello");
```

This one bites in test suites that assert on CLI output, and the failure looks like "my logging stopped working."

#### 5. IPC with a Node child needs `serialization: "json"`

Covered in [Part 7](./bun-apis-io-processes.md#p7-advanced) — advanced serialization is Bun-to-Bun. A mixed-runtime process tree silently fails to exchange messages otherwise.

#### 6. No `localStorage`

Isomorphic libraries that assume a `localStorage`-shaped store for caching will throw. Under Node they'd throw too, so this usually only appears in code paths that feature-detect wrongly.

<a id="p13-debug"></a>

### Advanced: debugging and profiling

#### The debugger

```bash
bun --inspect server.ts        # start an inspector
bun --inspect-brk script.ts    # break on the first line (short-lived scripts)
bun --inspect-wait script.ts   # don't execute until a debugger attaches
bun --inspect=4000 server.ts   # pick the port
```

Bun speaks the **WebKit Inspector Protocol** and prints two URLs — a `ws://` endpoint and a link to `debug.bun.sh`, a hosted WebKit Web Inspector ([debugger docs](https://bun.sh/docs/runtime/debugger)). There is also a VS Code extension, still marked experimental.

`--inspect-wait` is the one to remember: for a script that finishes in 200 ms, `--inspect` alone means the program is over before you attach.

#### Debugging network calls

```bash
BUN_CONFIG_VERBOSE_FETCH=curl bun app.ts   # print every fetch as a curl command
BUN_CONFIG_VERBOSE_FETCH=true bun app.ts   # print request/response summaries
```

`=curl` is excellent. Every outgoing request is printed as a copy-pasteable `curl` command, which turns "the API says 401" into a thing you can run in a terminal and hand to someone else.

#### Profiling

```bash
bun --cpu-prof-md app.ts     # CPU profile as a markdown report (1.4)
bun --heap-prof-md app.ts    # heap profile as markdown (1.4)
```

The markdown output is a deliberate choice — a readable report rather than a JSON blob needing a viewer, which also makes it pasteable into an issue.

For programmatic access, `bun:jsc` replaces the `node:v8` profiling APIs Bun doesn't implement:

```ts
import { heapStats, generateHeapSnapshot } from "bun:jsc";

console.log(heapStats());                 // object counts by type, heap size
await Bun.write("heap.json", JSON.stringify(generateHeapSnapshot()));
```

`heapStats()` is the fast leak-hunting tool: sample it periodically and watch which object type's count grows without bound.

Memory pressure is also observable, which is what makes graceful degradation in a container possible:

```ts
process.on("memoryPressure", () => cache.clear());
```

#### Stack traces

Bun generates sourcemaps for its own transpilation automatically, so a stack trace points at your TypeScript line, not the transpiled output. It also implements the V8 stack trace API (`Error.prepareStackTrace`, `Error.captureStackTrace`), which is what error-reporting libraries expect ([debugger docs](https://bun.sh/docs/runtime/debugger)).

<a id="p13-mastery"></a>

### Mastery: JavaScriptCore vs. V8

Bun embeds **JavaScriptCore** (Safari's engine); Node embeds **V8** (Chrome's). Nearly every observable difference in this doc descends from that choice.

```
        Bun                              Node
  ┌──────────────────┐            ┌──────────────────┐
  │ Bun APIs         │            │ Node APIs        │
  │ + node: compat   │            │                  │
  ├──────────────────┤            ├──────────────────┤
  │ JavaScriptCore   │            │ V8               │
  │  • LLInt → Baseline           │  • Ignition (interp)
  │    → DFG → FTL    │           │  • TurboFan (JIT) │
  │  • fast startup   │           │  • fast peak      │
  └──────────────────┘            └──────────────────┘
        native core (Rust as of 1.4)      C++ + libuv
```

#### Why startup is faster

JavaScriptCore uses a **four-tier** JIT: a low-level interpreter, then Baseline, DFG, and FTL compilers, promoting code as it proves hot. V8 has two tiers and optimises more eagerly. The practical result is that JSC reaches "running your code" sooner, which is why Bun's docs cite roughly 4× faster startup than Node and a `bun run` script starting in single-digit milliseconds against npm's ~170 ms ([run docs](https://bun.sh/docs/cli/run)).

Where that matters: CLIs, serverless cold starts, test suites that spawn many processes, and pre-commit hooks. Where it doesn't: a long-running server, where after warm-up both engines are in their top tier and the difference is workload-specific rather than categorical.

Be appropriately sceptical of benchmarks in both directions. Bun 1.4 also reports 5× lower idle CPU, 13–48% lower memory for HTTP servers, and 2× faster Linux startup ([Bun 1.4](https://bun.com/blog/bun-v1.4)) — these are the vendor's own numbers on the vendor's own workloads. The startup advantage is structural and reliable; steady-state throughput claims deserve your own measurement on your own workload.

#### Consequences you can actually observe

| Because the engine is JSC… | You see |
|---|---|
| No V8 heap snapshot format | `node:v8` profiling APIs missing; use `bun:jsc` |
| Different GC | Different memory curves; `smol` mode tunes JSC's heap size |
| Different JIT tiers | Faster startup, different peak-performance profile |
| Different stack trace internals | Bun implements the V8 API surface for compatibility |
| Different `Error.stack` formatting | Log parsers keyed to exact V8 text may need adjusting |

#### `smol` mode

```bash
bun --smol app.ts
```

```toml
smol = true
```

Sets JSC's heap to `Small`. Less memory, lower throughput. Correct in a 256 MB container; wrong on a workstation. `new Worker(url, { smol: true })` applies it per worker.

<a id="p13-checklist"></a>

### Mastery: a migration decision checklist

Answer these before committing to Bun for an existing production service. Each is a question with a cheap answer and an expensive surprise.

1. **Does your APM / tracing vendor support Bun?** The `createHook` stub means hook-based instrumentation cannot see async context. Test it in staging, don't read a compatibility page.
2. **Do you use `node:cluster` for scale-out?** Linux-only load balancing, not battle-tested. Consider `Bun.serve` with `reusePort` or a process manager instead.
3. **Do you need OCSP stapling, TLS session resumption across processes, or exotic curves?** Check `node:tls` and `node:crypto` gaps against your actual requirements.
4. **Do any dependencies need native build steps?** They will be silently skipped until you add them to `trustedDependencies` ([Part 4](./bun-package-manager.md#p4-advanced)). `bun pm untrusted` after your first install is a mandatory step, not an optional one.
5. **Do you deep-import into packages?** Isolated linking and strict `exports` enforcement will surface phantom dependencies immediately ([Part 4](./bun-package-manager.md#p4-linkers)). That is a fix list to schedule.
6. **Does your test suite rely on `__mocks__` directories or `jest.mock` hoisting?** Those need rewriting as `mock.module` in a preload ([Part 11](./bun-testing-bundling.md#p11-advanced)).
7. **Does anything parse `Error.stack` text, or monkey-patch `process.stdout.write`?** Both differ.
8. **Are you on Redis Cluster or Sentinel?** Bun's native client supports neither — use an npm client, or stay on Node for that service ([Part 10](./bun-servers-networking-data.md#p10-mastery)).

**The staged approach that de-risks all of this:** adopt Bun as a *package manager and test runner* first, while still running the service on Node. `bun install` produces a `node_modules` Node can use, and `bun test` is a separate process from your production runtime. You get the speed where it's felt most (CI, local iteration) with none of the runtime risk, and you find the dependency-level surprises before any of them can page you. Then move the runtime, one service at a time, starting with the least critical.

**Try It — measure the thing everyone claims.**

```ts
// startup.ts
console.log(process.versions.bun ? "bun" : "node");
```

```bash
time bun startup.ts
time node startup.ts     # if you have Node installed
```

Expected: Bun's total wall time is a small fraction of Node's — typically single-digit versus tens of milliseconds. Now scale it: put that in a `package.json` script and run each 50 times in a loop. The gap is what a pre-commit hook, a CLI, or a serverless cold start feels on every single invocation.

<a id="part-13-cheat-sheet"></a>

### Part 13 cheat sheet

| Want | How |
|---|---|
| Am I on Bun? | `process.versions.bun` |
| Check a module's support | [bun.sh/docs/runtime/nodejs-apis](https://bun.sh/docs/runtime/nodejs-apis) |
| Request-scoped context | `AsyncLocalStorage` ✅ — not `createHook` ❌ |
| Capture console output in tests | `spyOn(console, "log")`, not `process.stdout.write` |
| IPC with a Node child | `serialization: "json"` |
| Single-file executable | `bun build --compile` (no `node:sea`) |
| Debug | `bun --inspect` / `--inspect-brk` / `--inspect-wait` |
| Debug outbound HTTP | `BUN_CONFIG_VERBOSE_FETCH=curl` |
| CPU / heap profile | `bun --cpu-prof-md` / `--heap-prof-md` |
| Heap stats in code | `import { heapStats } from "bun:jsc"` |
| React to memory pressure | `process.on("memoryPressure", …)` |
| Cut memory usage | `bun --smol` |
| Engine | JavaScriptCore (not V8) |
| Not supported | `node:sea`, `localStorage`, Redis Cluster/Sentinel |

[↑ Back to top](#table-of-contents)

---

**Up:** [Overview](./bun-mastery-guide.md) · **Back to:** [Part 12 — Bundling](./bun-testing-bundling.md)

*Written against Bun v1.4.2 · Verified September 6, 2026*
