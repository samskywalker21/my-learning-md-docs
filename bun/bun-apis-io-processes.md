# Bun — APIs, I/O & Processes (Parts 6–8)

The Bun-native API surface: reading and writing files, running other programs, and the threads, natives, and utilities that replace a shelf of npm packages.

> **Spec:** this doc follows the shared spec in [`bun-mastery-guide.md`](./bun-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Mastery tiers (collapsed where a tier would be filler, always stated), **Try It** exercises with expected output, wrong-vs-right pairs, official docs cited inline. Written against **Bun v1.4.2**, verified **September 6, 2026**.
>
> **The rule that governs this whole doc:** every `node:` module Bun implements still works. These APIs are alternatives, not replacements — usually faster, usually nicer, always optional. Reach for `node:fs` without guilt when it fits better; [Part 13](./bun-node-compat-internals.md) covers where that compatibility has holes.

---

## Table of Contents

- [Part 6 — Files & Streams](#part-6--files--streams)
  - [Beginner: read a file, write a file](#p6-beginner)
  - [Working Knowledge: BunFile is lazy](#p6-working)
  - [Advanced: incremental writes with FileSink](#p6-advanced)
  - [Mastery: why `Bun.write` beats `fs.writeFile`](#p6-mastery)
  - [Part 6 cheat sheet](#part-6-cheat-sheet)
- [Part 7 — Processes & the Shell](#part-7--processes--the-shell)
  - [Beginner: run another program](#p7-beginner)
  - [Working Knowledge: Bun Shell](#p7-working)
  - [Advanced: streams, IPC, and cleanup](#p7-advanced)
  - [Mastery: timeouts, signals, and PTYs](#p7-mastery)
  - [Part 7 cheat sheet](#part-7-cheat-sheet)
- [Part 8 — Threads, Natives & Utilities](#part-8--threads-natives--utilities)
  - [Beginner: the utility belt](#p8-beginner)
  - [Working Knowledge: Workers](#p8-working)
  - [Advanced: hashing, passwords, and streams](#p8-advanced)
  - [Mastery: `bun:ffi` and Node-API](#p8-mastery)
  - [Part 8 cheat sheet](#part-8-cheat-sheet)

---

## Part 6 — Files & Streams

<a id="p6-beginner"></a>

### Beginner: read a file, write a file

```ts
// read
const text = await Bun.file("./config.json").text();
const data = await Bun.file("./config.json").json();

// write
await Bun.write("./out.txt", "hello");
```

`Bun.write` accepts almost anything as data — a string, a `Blob`, a `BunFile`, an `ArrayBuffer`, a `TypedArray`, or a `Response` ([file I/O docs](https://bun.sh/docs/api/file-io)):

```ts
// copy a file
await Bun.write("./copy.txt", Bun.file("./original.txt"));

// download straight to disk — no intermediate buffer in your code
await Bun.write("./page.html", await fetch("https://bun.sh"));

// print a file to stdout
await Bun.write(Bun.stdout, Bun.file("./out.txt"));
```

That last pair is the whole design in miniature: files, network responses, and stdio are all just sources and sinks, so they compose.

**Directories are not covered by this API.** For `mkdir`, `readdir`, `stat`, `rm`, use `node:fs` — Bun's docs say so directly:

```ts
import { readdir, mkdir } from "node:fs/promises";

await mkdir("./out", { recursive: true });
const files = await readdir("./src", { recursive: true });
```

<a id="p6-working"></a>

### Working Knowledge: BunFile is lazy

`Bun.file(path)` does **no I/O**. It returns a `BunFile` — a reference, a promise-of-a-file, a `Blob` subclass. Disk access happens only when you call a body method.

```ts
const f = Bun.file("./huge.log");   // instant, even on a 4 GB file
f.size;                             // bytes (from stat, not a read)
f.type;                             // MIME type, guessed from extension

await f.text();                     // ← the read actually happens here
await f.json();
await f.bytes();                    // Uint8Array
await f.arrayBuffer();
f.stream();                         // ReadableStream — no full read
await f.exists();                   // false for a missing file (size is 0)
await f.delete();
```

> **vs. Node** — `fs.readFile` reads immediately and takes an encoding argument. `Bun.file` defers, and gives you a `Blob`, so the same object can be handed to `fetch`, `Response`, or `Bun.write` without a conversion step.

This laziness is why a missing file doesn't throw at construction:

```ts
// ❌ expecting a throw here — this line always succeeds
const f = Bun.file("./does-not-exist.txt");

// ✅ the check, or the read, is where you find out
if (!(await f.exists())) throw new Error("missing config");
```

`BunFile` being a `Blob` pays off constantly:

```ts
// serve a file — no read into memory, Bun streams it
return new Response(Bun.file("./video.mp4"));

// upload a file
const form = new FormData();
form.append("file", Bun.file("./report.pdf"));
await fetch("https://example.com/upload", { method: "POST", body: form });
```

**Try It — prove the laziness.** On any large file you have:

```ts
// lazy.ts
const start = performance.now();
const f = Bun.file(process.argv[2]!);
console.log(`handle in ${(performance.now() - start).toFixed(3)}ms, size ${f.size}`);

const t0 = performance.now();
await f.bytes();
console.log(`read  in ${(performance.now() - t0).toFixed(3)}ms`);
```

```bash
bun lazy.ts ./some-large-file
```

Expected: the handle is created in a fraction of a millisecond regardless of file size, while the read scales with the file. Two different operations, and only one of them touches the disk.

<a id="p6-advanced"></a>

### Advanced: incremental writes with FileSink

`Bun.write` is one-shot: it takes all your data and writes it. For a log, an export, or anything produced over time, you want a **`FileSink`** ([file I/O docs](https://bun.sh/docs/api/file-io)):

```ts
const writer = Bun.file("./events.ndjson").writer({ highWaterMark: 1024 * 1024 });

for await (const event of eventStream) {
  writer.write(JSON.stringify(event) + "\n");   // buffered
}

await writer.flush();   // push the buffer to disk
await writer.end();     // flush and close
```

`highWaterMark` is the buffer size — writes accumulate until it fills, then go out in one syscall. That is the difference between one syscall per line and one per megabyte.

```ts
// ❌ a syscall per line, and a race: nothing guarantees ordering or completion
for (const line of lines) Bun.write("./out.txt", line);   // also: overwrites each time!

// ✅ buffered, ordered, closed
const w = Bun.file("./out.txt").writer();
for (const line of lines) w.write(line + "\n");
await w.end();
```

The wrong version has two bugs, and the second is the nastier one: `Bun.write` **replaces** the file, so a loop of `Bun.write` calls to the same path leaves you with only the last line.

`writer.unref()` lets the process exit with the sink still open; `writer.ref()` undoes that.

<a id="p6-mastery"></a>

### Mastery: why `Bun.write` beats `fs.writeFile`

`Bun.write` chooses the fastest syscall available for the source/destination pair and platform, rather than always round-tripping bytes through your process ([file I/O docs](https://bun.sh/docs/api/file-io)):

| Destination | Source | Syscall used | Platform |
|---|---|---|---|
| file | file | `copy_file_range` | Linux |
| file | pipe | `sendfile` | Linux |
| file | Blob / string | `write` | macOS, Linux |

```
  fs.readFile + fs.writeFile          Bun.write(dest, Bun.file(src))

  disk ──► kernel ──► JS heap         disk ──► kernel ──► disk
              │           │                       │
         (copy)      (copy back)          copy_file_range:
                                          bytes never enter
                                          your process at all
```

That is why the docs can show a three-line `cat` that outruns GNU `cat` on Linux:

```ts
await Bun.write(Bun.stdout, Bun.file(process.argv.at(-1)!));
```

The practical rule: **when both ends of a copy are files or streams, use `Bun.write` and let it pick.** The moment you do `await file.text()` and then write the string, you have opted into the slow path by pulling every byte through the JavaScript heap. That is fine for a config file and wasteful for a video.

<a id="part-6-cheat-sheet"></a>

### Part 6 cheat sheet

| Want | How |
|---|---|
| Read text / JSON | `await Bun.file(p).text()` / `.json()` |
| Read bytes | `await Bun.file(p).bytes()` → `Uint8Array` |
| Stream a file | `Bun.file(p).stream()` |
| Does it exist? | `await Bun.file(p).exists()` |
| Delete | `await Bun.file(p).delete()` |
| Write | `await Bun.write(p, data)` |
| Copy | `await Bun.write(dest, Bun.file(src))` |
| Save a download | `await Bun.write(p, await fetch(url))` |
| Append / stream out | `const w = Bun.file(p).writer(); w.write(…); await w.end()` |
| stdio | `Bun.stdin`, `Bun.stdout`, `Bun.stderr` |
| mkdir / readdir / stat | `node:fs` — not covered by `Bun.file` |

[↑ Back to top](#table-of-contents)

---

## Part 7 — Processes & the Shell

<a id="p7-beginner"></a>

### Beginner: run another program

```ts
const proc = Bun.spawn(["git", "rev-parse", "HEAD"]);
const sha = (await proc.stdout.text()).trim();
await proc.exited;               // exit code
```

Synchronously, when you just want the answer:

```ts
const { stdout, exitCode, success } = Bun.spawnSync(["git", "status", "--short"]);
console.log(stdout.toString());  // Buffer, not a stream
```

The difference matters: `Bun.spawn` gives you `stdout` as a web `ReadableStream`; `Bun.spawnSync` gives you a `Buffer` and blocks ([spawn docs](https://bun.sh/docs/api/spawn)).

Defaults worth memorising: `stdin` is `null` (no input), `stdout` is `"pipe"` (captured), `stderr` is `"inherit"` (goes straight to your terminal). That last one is why you sometimes see a child's error output but can't find it in your captured strings — you have to ask for it with `stderr: "pipe"`.

<a id="p7-working"></a>

### Working Knowledge: Bun Shell

For anything that would be a shell one-liner, `Bun.$` is nicer than `spawn` ([shell docs](https://bun.sh/docs/runtime/shell)):

```ts
import { $ } from "bun";

await $`echo hello`;                                   // prints
const branch = await $`git rev-parse --abbrev-ref HEAD`.text();
const pkg = await $`cat package.json`.json();
const { exitCode } = await $`test -f .env`.nothrow().quiet();

for await (const line of $`git log --oneline -n 20`.lines()) {
  console.log(line);
}
```

**It is not your system shell.** It is a bash-like shell implemented inside Bun, which is why the same script works on Windows without WSL. Built-in commands — `cd`, `ls`, `rm`, `echo`, `pwd`, `cat`, `touch`, `mkdir`, `which`, `mv`, `exit`, `true`, `false`, `yes`, `seq`, `dirname`, `basename`, `bun` — are implemented natively, so `ls` works on Windows even though Windows has no `ls`.

#### Interpolation is escaped by default

This is the feature that makes the API safe to use with untrusted input:

```ts
const userInput = "my file.txt; rm -rf /";
await $`ls ${userInput}`;
// treated as ONE argument, literally named "my file.txt; rm -rf /"
// the semicolon is not a command separator; nothing is deleted
```

```ts
// ❌ the classic injection, and the reason `exec` has a bad reputation
import { execSync } from "node:child_process";
execSync(`ls ${userInput}`);   // the `rm -rf /` runs. Really.

// ✅ Bun Shell escapes every interpolation
await $`ls ${userInput}`;
```

To *opt out* — which you should do rarely, deliberately, and never with user input:

```ts
await $`echo ${{ raw: "$(date)" }}`;   // unescaped
```

#### Errors, output, and context

```ts
try {
  await $`exit 1`;
} catch (err) {
  err.exitCode;              // 1
  err.stdout.toString();
  err.stderr.toString();
}

$.nothrow();                 // globally stop throwing on non-zero exit
$.throws(true);              // restore

await $`pwd`.cwd("/tmp");
await $`echo $FOO`.env({ ...process.env, FOO: "bar" });
$.cwd("/tmp");               // or set them globally
```

Redirection and piping work as you'd expect, including into JavaScript objects — which is the genuinely novel part:

```ts
await $`echo hi > out.txt`;
await $`echo hi >> out.txt`;
await $`cmd 2>&1`;
const words = await $`echo "a b c" | wc -w`.text();

const buf = Buffer.alloc(100);
await $`echo hello > ${buf}`;                          // shell output → Buffer

const res = new Response("hello body");
const n = await $`cat < ${res} | wc -w`.text();        // Response → shell stdin
```

**Try It — prove the escaping.**

```ts
// safe.ts
import { $ } from "bun";
const evil = "; echo PWNED";
await $`echo ${evil}`;
```

```bash
bun safe.ts
```

Expected output: `; echo PWNED` — printed as literal text, on one line. If it were interpolating into a real shell, you would instead see an empty line followed by `PWNED`.

<a id="p7-advanced"></a>

### Advanced: streams, IPC, and cleanup

#### Streaming a child's output

```ts
const proc = Bun.spawn(["ffmpeg", "-i", "in.mp4", "out.webm"], { stderr: "pipe" });

for await (const chunk of proc.stderr) {
  process.stdout.write(chunk);    // progress as it happens
}
await proc.exited;
```

Pipe a child straight into a file, with no data passing through your process:

```ts
Bun.spawn(["pg_dump", "mydb"], { stdout: Bun.file("./dump.sql") });
```

Feed stdin incrementally:

```ts
const proc = Bun.spawn(["wc", "-l"], { stdin: "pipe" });
proc.stdin.write("a\nb\nc\n");
proc.stdin.flush();
proc.stdin.end();
console.log(await proc.stdout.text());   // "3"
```

#### IPC between processes

When both ends are Bun, you get structured message passing ([spawn docs](https://bun.sh/docs/api/spawn)):

```ts
// parent.ts
const child = Bun.spawn(["bun", "child.ts"], {
  ipc(message, proc) {
    console.log("from child:", message);
    proc.send({ reply: true });
  },
});
child.send({ job: 1 });
```

```ts
// child.ts
process.on("message", (m) => process.send({ got: m }));
```

**Talking to a Node child requires `serialization: "json"`** — Bun's advanced serialization is Bun-to-Bun only, and this is a documented incompatibility rather than a bug you can work around:

```ts
Bun.spawn({ cmd: ["node", "worker.js"], serialization: "json", ipc(msg) { /* … */ } });
```

#### Cleanup

```ts
proc.kill();              // SIGTERM
proc.kill("SIGKILL");
proc.unref();             // let the parent exit without waiting
await proc.exited;
proc.exitCode;            // number | null
proc.signalCode;          // "SIGTERM" | … | null
proc.killed;              // boolean
```

```ts
// ❌ zombie: nothing ever awaits the child, and the parent may exit first
Bun.spawn(["long-running-thing"]);

// ✅ own the lifetime — await it, or explicitly unref it
const p = Bun.spawn(["long-running-thing"]);
process.on("SIGINT", () => p.kill());
await p.exited;
```

<a id="p7-mastery"></a>

### Mastery: timeouts, signals, and PTYs

#### Timeouts and abort signals

```ts
const proc = Bun.spawn({
  cmd: ["sleep", "100"],
  timeout: 5000,             // ms
  killSignal: "SIGKILL",     // default is SIGTERM
});

// or drive it from an AbortController
const ac = new AbortController();
Bun.spawn({ cmd: ["sleep", "100"], signal: ac.signal });
ac.abort();
```

A built-in `timeout` is a real quality-of-life win: the Node equivalent is a `setTimeout` that races the exit promise and remembers to clear itself, which everyone writes slightly wrong at least once.

#### Resource accounting

```ts
await proc.exited;
const usage = proc.resourceUsage();
usage.maxRSS;             // peak memory, bytes
usage.cpuTime.user;       // µs
usage.cpuTime.system;     // µs
```

This is the honest way to answer "how much memory did that build actually take?" without an external profiler.

On Linux you can also confine a child to a cgroup, which is how you cap a build step rather than letting it take the machine down:

```ts
Bun.spawn({ cmd: ["make", "-j8"], cgroup: "/sys/fs/cgroup/build-jobs" });
```

#### PTYs

Some programs behave differently when they think they're talking to a terminal — colours, progress bars, interactive prompts. `Bun.Terminal` (1.4) gives the child a real pseudo-terminal ([spawn docs](https://bun.sh/docs/api/spawn)):

```ts
const proc = Bun.spawn(["bash"], {
  terminal: {
    cols: 80,
    rows: 24,
    data(_terminal, chunk) { process.stdout.write(chunk); },
  },
});

proc.terminal.write("echo hello\n");
proc.terminal.resize(120, 40);
await proc.exited;
```

**The gotcha:** when `terminal` is used, `proc.stdin`/`stdout`/`stderr` are all `null`. The PTY replaces them — use `proc.terminal` instead. Code that mixes the two models silently reads nothing.

<a id="part-7-cheat-sheet"></a>

### Part 7 cheat sheet

| Want | How |
|---|---|
| Run and capture | `await Bun.spawn([…]).stdout.text()` |
| Run and block | `Bun.spawnSync([…])` → `Buffer` |
| Shell one-liner | ``await $`cmd` `` |
| Shell output as text/JSON/lines | `.text()` / `.json()` / `.lines()` |
| Don't throw on non-zero | `.nothrow()` (or `$.nothrow()` globally) |
| Suppress output | `.quiet()` |
| Set cwd / env | `.cwd(dir)` / `.env({…})` |
| Escape hatch (dangerous) | `${{ raw: "…" }}` |
| Capture stderr | `{ stderr: "pipe" }` — it's `"inherit"` by default |
| Child output → file | `{ stdout: Bun.file("./out") }` |
| Talk to a Bun child | `ipc(msg, proc)` + `proc.send()` |
| Talk to a Node child | add `serialization: "json"` |
| Kill after N ms | `{ timeout: 5000, killSignal: "SIGKILL" }` |
| Peak memory of a child | `proc.resourceUsage().maxRSS` |
| Give the child a TTY | `{ terminal: { cols, rows, data } }` |

[↑ Back to top](#table-of-contents)

---

## Part 8 — Threads, Natives & Utilities

*This Part collapses the Advanced tier for utilities — a hash function has no production edge cases worth a tier — and gives the depth to Workers and FFI instead.*

<a id="p8-beginner"></a>

### Beginner: the utility belt

A meaningful share of a typical `package.json` is functions Bun already has ([Bun APIs](https://bun.sh/docs/runtime/bun-apis)):

```ts
// globbing — replaces the `glob` package
const glob = new Bun.Glob("**/*.test.ts");
for await (const path of glob.scan(".")) console.log(path);
glob.match("src/a.test.ts");                    // true

// timing & ids
await Bun.sleep(100);
Bun.nanoseconds();
Bun.randomUUIDv7();                             // time-sortable UUIDs

// finding things
Bun.which("git");                               // "/usr/bin/git" | null

// inspection
Bun.deepEquals({ a: 1 }, { a: 1 });             // true
Bun.inspect(someObject);                        // what console.log would print
Bun.escapeHTML("<script>");                     // "&lt;script&gt;"
Bun.stringWidth("👋 hi");                        // terminal columns, not code units

// parsing
Bun.TOML.parse(await Bun.file("cfg.toml").text());
Bun.semver.satisfies("1.2.3", "^1.0.0");

// compression
Bun.gzipSync(bytes);
Bun.gunzipSync(compressed);
```

`Bun.stringWidth` deserves the callout: it computes the *display* width of a string, accounting for emoji, wide CJK characters, and ANSI escapes. Every CLI that aligns a table needs it, and getting it right by hand is genuinely hard.

<a id="p8-working"></a>

### Working Knowledge: Workers

JavaScript is single-threaded, so a CPU-bound task blocks everything — including your HTTP server's ability to answer requests. `Worker` moves it to another thread ([workers docs](https://bun.sh/docs/api/workers)):

```ts
// main.ts
const worker = new Worker("./worker.ts");
worker.postMessage({ n: 40 });
worker.onmessage = (e) => console.log("result:", e.data);
```

```ts
// worker.ts
declare var self: Worker;
self.onmessage = (e: MessageEvent) => {
  postMessage(fib(e.data.n));
};
```

TypeScript works directly — no build step, no bundling the worker first, which is the tedious part of the browser and Node equivalents.

Lifecycle:

```ts
worker.terminate();                               // stop it now
worker.unref();                                   // don't keep the process alive
new Worker("./w.ts", { ref: false });             // same, at construction
new Worker("./w.ts", { smol: true });             // small JSC heap: less memory, slower
new Worker("./w.ts", { preload: ["./otel.ts"] }); // run instrumentation first
worker.addEventListener("open", () => {});        // Bun-specific: worker is ready
worker.addEventListener("close", (e) => e.code);  // exit code
Bun.isMainThread;                                 // where am I?
```

**The message-queue gotcha, stated plainly in the docs:** Bun **drops** messages posted before a handler is attached, where Node queues them. So this is a real race:

```ts
// ❌ the message can be delivered before onmessage is assigned
const w = new Worker("./w.ts");
w.postMessage("go");
w.onmessage = handle;

// ✅ attach the handler first
const w = new Worker("./w.ts");
w.onmessage = handle;
w.postMessage("go");
```

The docs also note the Worker API — particularly termination — is still marked experimental. Treat worker pools as something to test under load rather than assume.

**When a Worker is the wrong tool:** for I/O-bound work (waiting on the network or disk), you don't need one — that's what the event loop already does well. Workers are for **CPU-bound** work: parsing a huge file, image processing, cryptography, compression.

<a id="p8-advanced"></a>

### Advanced: hashing, passwords, and streams

#### Passwords

```ts
const hash = await Bun.password.hash("correct horse battery staple");
// → argon2id by default
const ok = await Bun.password.verify("correct horse battery staple", hash);

// bcrypt, when you must match an existing database
await Bun.password.hash(pw, { algorithm: "bcrypt", cost: 12 });
```

```ts
// ❌ never — fast hashes are the wrong tool for passwords
const bad = Bun.hash("hunter2");                         // non-cryptographic!
const alsoBad = new Bun.CryptoHasher("sha256").update(pw).digest("hex");

// ✅ a deliberately slow, salted, memory-hard KDF
const good = await Bun.password.hash(pw);
```

`Bun.password` picks Argon2id, salts automatically, and encodes the parameters into the hash string so verification needs nothing but the hash. This removes the single most common way homegrown auth goes wrong — and it removes a native `bcrypt` dependency that would need a build step (and therefore `trustedDependencies`; see [Part 4](./bun-package-manager.md#p4-advanced)).

#### The three hash families, and when each is right

| Use | API | Why |
|---|---|---|
| Hash table keys, cache buckets, sharding | `Bun.hash(data)` | Fast, non-cryptographic (Wyhash) |
| Checksums, content addressing, signatures | `new Bun.CryptoHasher("sha256")`, `Bun.sha()` | Cryptographic |
| Passwords | `Bun.password.hash()` | Slow and memory-hard **on purpose** |

Picking row 1 or 2 for a password is the classic security bug. The whole point of a password hash is to be *expensive*.

#### Stream helpers

```ts
await Bun.readableStreamToText(stream);
await Bun.readableStreamToJSON(stream);
await Bun.readableStreamToArrayBuffer(stream);
await Bun.readableStreamToBytes(stream);
await Bun.readableStreamToArray(stream);
```

These exist because "collect this whole stream into a value" is the operation everyone hand-writes with a chunk array and a concat, slightly wrong.

#### CSRF tokens

```ts
const token = Bun.CSRF.generate(secret);
Bun.CSRF.verify(token, secret);   // boolean
```

<a id="p8-mastery"></a>

### Mastery: `bun:ffi` and Node-API

Two ways to call native code, and the choice between them is not close.

#### Node-API — the boring, correct choice

Node-API (`.node` addons) is the stable ABI Node uses. Bun implements it, so existing native npm packages work. The docs' own guidance: **use Node-API for production**.

#### `bun:ffi` — call a C ABI directly

No addon, no compile step for the binding — you point at a shared library and declare the signatures ([FFI docs](https://bun.sh/docs/api/ffi)):

```ts
import { dlopen, FFIType, suffix } from "bun:ffi";

const { symbols } = dlopen(`libsqlite3.${suffix}`, {   // suffix: .dylib | .so | .dll
  sqlite3_libversion: { args: [], returns: FFIType.cstring },
});

console.log(symbols.sqlite3_libversion());
```

It is fast — JavaScriptCore JIT-compiles the call into a direct native call, which the docs put at 2–6× Node's FFI-via-Node-API. It is also **experimental**, and the failure mode is not an exception:

> **Get a pointer wrong and the process segfaults.** There is no stack trace, no `catch`, no error message. You get a dead process.

The caveats the docs state, each of which is a real crash:

1. **No memory management.** `bun:ffi` frees nothing. Use a `FinalizationRegistry` from JS, or pass a deallocator to `toArrayBuffer`.
2. **Alignment matters.** A `u64*` is not a `[8]u8*`.
3. **Windows `HANDLE` is `u64`, not `ptr`.**
4. **Callbacks from foreign threads need `threadsafe: true`** on the `JSCallback`, and must be `.close()`d when done.
5. **No async callbacks.**

Strings need explicit handling because JS strings are UTF-16 and C strings are null-terminated bytes:

```ts
import { CString, ptr, read, toArrayBuffer } from "bun:ffi";

new CString(myPtr);                       // null-terminated → JS string (copies)
new CString(myPtr, 0, byteLength);        // known length
ptr(new Uint8Array(32));                  // TypedArray → pointer
read.u32(myPtr, 0);                       // read a value at an offset
toArrayBuffer(myPtr, 0, len);             // wrap native memory
```

**The decision rule:** if a Node-API addon exists, use it. Reach for `bun:ffi` when you are binding a system library that has no JavaScript wrapper, you control the lifetime of everything you touch, and a crash in that path is acceptable while you develop. It is a superb tool for a script that talks to `libc` and a dangerous one at the centre of a service.

<a id="part-8-cheat-sheet"></a>

### Part 8 cheat sheet

| Want | Use | Replaces |
|---|---|---|
| Glob files | `new Bun.Glob(pat).scan(dir)` | `glob` |
| Sleep | `await Bun.sleep(ms)` | hand-rolled promise |
| Sortable UUID | `Bun.randomUUIDv7()` | `uuid` |
| Find a binary | `Bun.which("git")` | `which` |
| Deep equality | `Bun.deepEquals(a, b)` | `lodash.isEqual` |
| Terminal string width | `Bun.stringWidth(s)` | `string-width` |
| Escape HTML | `Bun.escapeHTML(s)` | `escape-html` |
| Parse TOML | `Bun.TOML.parse(s)` | `toml` |
| Semver check | `Bun.semver.satisfies(v, r)` | `semver` |
| Gzip | `Bun.gzipSync` / `Bun.gunzipSync` | `node:zlib` |
| Hash a password | `await Bun.password.hash(pw)` | `bcrypt`, `argon2` |
| Verify a password | `await Bun.password.verify(pw, hash)` | ↑ |
| Cryptographic digest | `new Bun.CryptoHasher("sha256")` | `node:crypto` |
| Fast non-crypto hash | `Bun.hash(data)` | `xxhash` |
| Drain a stream | `Bun.readableStreamToText(s)` | hand-rolled loop |
| CPU-bound work off-thread | `new Worker("./w.ts")` | `worker_threads` |
| Call a C library | `dlopen` from `bun:ffi` (experimental) | `ffi-napi` |
| Production native code | Node-API `.node` addon | — |

[↑ Back to top](#table-of-contents)

---

**Next:** [Parts 9–10 — Servers, Networking & Data](./bun-servers-networking-data.md) · **Up:** [Overview](./bun-mastery-guide.md)

*Written against Bun v1.4.2 · Verified September 6, 2026*
