# Node.js — Files, Streams & Buffers (Parts 6–7)

The byte layer. How to read and write files without blocking the loop, how to handle paths that survive contact with Windows, what a `Buffer` actually is, and — the Part that separates working Node from good Node — how streams keep memory flat by refusing to go faster than their destination.

> **Spec:** this doc follows the shared spec in [`nodejs-mastery-guide.md`](./nodejs-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Mastery tiers (collapsed where a tier would be filler, always stated), **Try It** exercises with expected output, wrong-vs-right pairs, official docs cited inline. Written against **Node.js 24 LTS**, verified **September 14, 2026**.
>
> **Prerequisite:** [Part 2 — The Event Loop](./nodejs-runtime-foundations.md#part-2--the-event-loop). Backpressure is an event-loop story, and Part 7 assumes you know why a blocked thread is fatal.

---

## Table of Contents

- [Part 6 — Files, Paths & Buffers](#part-6--files-paths--buffers)
  - [Beginner: read a file, write a file](#p6-beginner)
  - [Working Knowledge: directories, existence, and the `fs` surface worth knowing](#p6-working)
  - [Advanced: paths that survive Windows, and `Buffer`](#p6-advanced)
  - [Mastery: file handles, atomicity, and what `fs` costs](#p6-mastery)
  - [Part 6 cheat sheet](#part-6-cheat-sheet)
- [Part 7 — Streams & Backpressure](#part-7--streams--backpressure)
  - [Beginner: why streams exist](#p7-beginner)
  - [Working Knowledge: `pipeline`, and the four stream types](#p7-working)
  - [Advanced: backpressure, explicitly](#p7-advanced)
  - [Mastery: object mode, `compose`, and Web Streams interop](#p7-mastery)
  - [Part 7 cheat sheet](#part-7-cheat-sheet)

---

## Part 6 — Files, Paths & Buffers

`node:fs` is the oldest big module in Node and it shows: three parallel APIs, a `Buffer` type that predates `Uint8Array`, and a lot of functions whose names made sense in 2010. The good news is that the modern subset is small, and you can ignore most of the rest.

<a id="p6-beginner"></a>

### Beginner: read a file, write a file

Use the promise API. It is a different import path, and that is the only thing you need to remember:

```js
import { readFile, writeFile } from 'node:fs/promises';

const config = await readFile('config.json', 'utf8');
await writeFile('out.txt', 'hello\n');
```

Leave the encoding off and you get a `Buffer` — raw bytes — instead of a string:

```js
const bytes = await readFile('logo.png');    // Buffer
const text  = await readFile('notes.md', 'utf8');   // string
```

That is the single most common source of "why is my file contents `<Buffer 89 50 4e 47…>`": no encoding argument.

Three APIs exist for almost every operation, and picking between them is mechanical:

```js
import { readFile } from 'node:fs/promises';   // ✓ default choice
import { readFile } from 'node:fs';            // callback — only when an old API forces it
import { readFileSync } from 'node:fs';        // blocks the thread
```

`readFileSync` is not forbidden — it is the right call for loading config at startup, before you are serving anyone. It is catastrophic inside a request handler, for the reasons [Part 2](./nodejs-runtime-foundations.md#p2-beginner) spent a whole tier on.

<a id="p6-working"></a>

### Working Knowledge: directories, existence, and the `fs` surface worth knowing

#### Directories

```js
import { mkdir, readdir, rm, cp } from 'node:fs/promises';

await mkdir('build/assets', { recursive: true });   // like mkdir -p; no error if it exists
await rm('build', { recursive: true, force: true }); // like rm -rf; force ignores ENOENT
await cp('src/static', 'build/static', { recursive: true });

const names = await readdir('src');                                  // ['a.js', 'lib']
const entries = await readdir('src', { withFileTypes: true });       // Dirent objects
const everything = await readdir('src', { recursive: true });        // walks subdirectories
```

`withFileTypes: true` is worth making a habit. Without it you get strings and must `stat()` each one to learn whether it's a directory — an extra syscall per entry, on the thread pool, for information the OS already returned:

```js
// ✗ wrong — one stat() per entry, on a shared 4-thread pool
for (const name of await readdir(dir)) {
  if ((await stat(join(dir, name))).isDirectory()) { /* … */ }
}

// ✓ right — the type came back with the listing
for (const e of await readdir(dir, { withFileTypes: true })) {
  if (e.isDirectory()) { /* … */ }
}
```

#### Don't check whether a file exists

The `access`-then-open pattern is a race condition with extra steps — the file can vanish between the two calls — and it doubles the syscalls:

```js
// ✗ wrong — TOCTOU race, and slower
if (await access(path).then(() => true, () => false)) {
  const data = await readFile(path, 'utf8');
}

// ✓ right — attempt it, handle the failure
try {
  const data = await readFile(path, 'utf8');
} catch (err) {
  if (err.code !== 'ENOENT') throw err;
  // handle "not there"
}
```

`err.code` is the thing to branch on, never the message. The codes you will actually meet: `ENOENT` (no such file), `EACCES` (permission), `EEXIST` (already there), `EISDIR`/`ENOTDIR` (wrong type), `EMFILE` (too many open files — see [Mastery](#p6-mastery)), `ENOSPC` (disk full).

#### The rest of the surface, briefly

```js
import { stat, lstat, rename, glob, watch } from 'node:fs/promises';

const s = await stat('file.txt');      // size, mtime, isFile(), isDirectory()…
await lstat('link');                    // same, but does NOT follow symlinks
await rename('a.txt', 'b.txt');         // atomic within one filesystem; fails across devices

for await (const f of glob('src/**/*.test.js')) console.log(f);  // experimental
for await (const ev of watch('src', { recursive: true })) console.log(ev.eventType, ev.filename);
```

`glob` in `node:fs/promises` is still marked experimental — usable, but check its status before depending on it in a library ([fs docs](https://nodejs.org/docs/latest-v24.x/api/fs.html)).

**Try It — a recursive directory size, in nine lines.**

```js
// du.mjs
import { readdir, stat } from 'node:fs/promises';
import { join } from 'node:path';

async function size(dir) {
  let total = 0;
  for (const e of await readdir(dir, { withFileTypes: true })) {
    const p = join(dir, e.name);
    total += e.isDirectory() ? await size(p) : (await stat(p)).size;
  }
  return total;
}
console.log((await size(process.argv[2] ?? '.') / 1e6).toFixed(1), 'MB');
```

```bash
node du.mjs ./src
```

Expected output: a number of megabytes, e.g. `3.4 MB`. Run it against `node_modules` and notice how long it takes — that delay is thousands of `stat` calls queuing through libuv's four-thread pool, which is [Part 2's](./nodejs-runtime-foundations.md#p2-mastery) thread pool made visible.

<a id="p6-advanced"></a>

### Advanced: paths that survive Windows, and `Buffer`

#### `node:path`

```js
import { join, resolve, dirname, basename, extname, relative, sep } from 'node:path';

join('src', 'lib', 'x.js')      // 'src/lib/x.js'  ('src\\lib\\x.js' on Windows)
resolve('src', 'x.js')          // absolute, from cwd
extname('photo.tar.gz')         // '.gz'   ← only the last one
basename('/a/b/c.js', '.js')    // 'c'
```

Three rules cover the cross-platform pain:

```js
// ✗ wrong — breaks on Windows
const p = dir + '/' + name;

// ✓ right
const p = join(dir, name);
```

```js
// ✗ wrong — relative to the CWD, so it breaks the moment someone runs you from elsewhere
const schema = await readFile('./schema.sql', 'utf8');

// ✓ right — relative to THIS file
const schema = await readFile(new URL('./schema.sql', import.meta.url), 'utf8');
```

That second pair is the important one. `process.cwd()` is wherever the user happened to be standing; `import.meta.url` is where your code lives. Anything your program *ships with* should be resolved from the module, and `fs` accepts `URL` objects directly, so there is no conversion step ([fs docs](https://nodejs.org/docs/latest-v24.x/api/fs.html)).

And when you must turn a `file:` URL into a string path — for a library that only accepts strings:

```js
import { fileURLToPath } from 'node:url';
const p = fileURLToPath(import.meta.url);   // NOT import.meta.url.replace('file://', '')
```

The naive `replace` breaks on Windows drive letters and on any path containing a space or non-ASCII character, both of which are common and neither of which you will have in your test fixtures.

#### `Buffer` — a `Uint8Array` with a friendlier face

A `Buffer` *is* a `Uint8Array`; every typed-array method works on it, and it can be passed anywhere a `Uint8Array` is expected. What it adds is encodings and a set of read/write helpers.

```js
const b = Buffer.from('héllo', 'utf8');
b.length;                 // 6 — BYTES, not characters. 'é' is two bytes.
b.toString('utf8');       // 'héllo'
b.toString('base64');     // 'aMOpbGxv'
b instanceof Uint8Array;  // true

Buffer.alloc(16);           // 16 zeroed bytes
Buffer.concat([a, b]);
b.subarray(0, 4);           // a VIEW — shares memory with b
```

Three things bite people:

**1. Length is bytes.** `'é'.length` is 1; `Buffer.from('é').length` is 2. Any code that slices a buffer by "character count" is broken for every non-ASCII input.

**2. `subarray` shares memory.** Mutating the view mutates the original. That is a feature (zero-copy), but it means a buffer you slice and hold onto keeps the *whole* original alive:

```js
// ✗ wrong — keeps a 10 MB buffer alive to retain 20 bytes
const header = bigBuffer.subarray(0, 20);

// ✓ right — copies out
const header = Uint8Array.prototype.slice.call(bigBuffer, 0, 20);
```

**3. `Buffer.allocUnsafe` is named honestly.** It skips zeroing, so the memory contains whatever was there before — possibly another request's data. Use it only when you immediately overwrite every byte.

**Try It — see the multi-byte problem.**

```js
const b = Buffer.from('naïve café');
console.log('chars', 'naïve café'.length, 'bytes', b.length);
console.log(b.subarray(0, 3).toString());
```

Expected output: `chars 10 bytes 12`, then `na` followed by a replacement character `�` — because byte 3 cut `ï` in half. This is exactly the bug that appears when a stream chunk boundary lands mid-character, and it's why the next Part cares about `StringDecoder`.

<a id="p6-mastery"></a>

### Mastery: file handles, atomicity, and what `fs` costs

#### File handles, and running out of them

Every open file is an OS file descriptor, and processes have a limit (often 1024–8192). Exceed it and every subsequent open — including *sockets*, which share the same table — fails with `EMFILE`:

```js
// ✗ wrong — opens 10,000 descriptors at once
await Promise.all(files.map((f) => readFile(f)));

// ✓ right — bounded concurrency (mapLimit from Part 3)
await mapLimit(files, 32, (f) => readFile(f));
```

`EMFILE` under load is one of the more confusing production failures, because the error surfaces wherever the *next* open happened to be — often in your HTTP server, nowhere near the code that leaked.

When you hold a handle explicitly, Node 24 gives you deterministic cleanup via explicit resource management (new in V8 13.6, see the [overview's delta](./nodejs-mastery-guide.md#6-what-changed-under-you-node-22--24)):

```js
import { open } from 'node:fs/promises';

// ✓ closed automatically at the end of the block, even if this throws
await using fh = await open('data.bin', 'r');
const { bytesRead, buffer } = await fh.read(Buffer.alloc(1024), 0, 1024, 0);
```

That replaces the `try`/`finally`/`fh?.close()` dance, and it is a genuine improvement: the old form is easy to write and easy to write *wrong*.

#### Writing a file without corrupting it

`writeFile` is not atomic. Kill the process halfway through and readers see a truncated file. The standard fix is write-then-rename, because `rename` *is* atomic within a filesystem:

```js
import { writeFile, rename } from 'node:fs/promises';

// ✓ readers see either the old file or the new one, never half of either
const tmp = `${target}.${process.pid}.tmp`;
await writeFile(tmp, data, { flush: true });   // flush: force to disk before we rename
await rename(tmp, target);
```

The `flush: true` option matters if you care about surviving a machine crash rather than just a process crash: without it, the data may still be in the OS page cache when the rename lands. Note the constraint in `rename`'s contract — same filesystem. Renaming from `/tmp` to `/app` across a container mount fails with `EXDEV`, which is why the temp file goes next to the target.

#### What `fs` actually costs

Every async `fs` call is dispatched to libuv's thread pool — the same four threads shared by `dns.lookup()` and `crypto`'s KDFs ([Don't Block the Event Loop](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)). Three consequences:

- **`fs` async is not free concurrency.** A hundred parallel reads are four at a time, queued.
- **A slow filesystem (NFS, a network volume, a throttled EBS disk) will stall unrelated DNS lookups**, because they share the queue. This is a genuinely surprising failure mode and worth remembering when "the database is slow" turns out to be DNS.
- **`readFileSync` at startup is often faster than the async version**, because it skips the dispatch entirely. Startup is the one place where sync wins.

The counter-intuitive conclusion: for *small* config files at boot, sync is correct. For anything during request handling, never. The rule isn't "sync is bad", it's "blocking while anyone is waiting is bad".

<a id="part-6-cheat-sheet"></a>

### Part 6 cheat sheet

| Want | How |
|---|---|
| Read a text file | `await readFile(p, 'utf8')` from `node:fs/promises` |
| Read bytes | `await readFile(p)` → `Buffer` |
| Write | `await writeFile(p, data)` |
| `mkdir -p` | `await mkdir(p, { recursive: true })` |
| `rm -rf` | `await rm(p, { recursive: true, force: true })` |
| Copy a tree | `await cp(a, b, { recursive: true })` |
| List with types | `await readdir(p, { withFileTypes: true })` |
| Walk recursively | `await readdir(p, { recursive: true })` |
| Does it exist? | Don't check — `try`/`catch` on `err.code === 'ENOENT'` |
| Join paths | `join(a, b)` — never `+ '/' +` |
| A file next to this module | `new URL('./x.sql', import.meta.url)` |
| `file:` URL → path | `fileURLToPath(url)` |
| Guaranteed close | `await using fh = await open(p)` |
| Atomic write | `writeFile(tmp, …, { flush: true })` then `rename(tmp, target)` |
| Bytes vs characters | `Buffer.length` is **bytes** |
| Zero-copy slice | `buf.subarray()` — shares memory, keeps the original alive |

**Gotchas:** no encoding argument returns a `Buffer` · `access`-then-read is a race · `readdir` without `withFileTypes` costs a `stat` per entry · unbounded parallel reads hit `EMFILE` · `rename` fails across filesystems (`EXDEV`) · async `fs` shares four threads with DNS and crypto.

[↑ Back to top](#table-of-contents)

---

## Part 7 — Streams & Backpressure

Streams are the part of Node people avoid for years and then wish they'd learned in week two. The API has a reputation for being awkward, most of which was earned by the pre-2018 version of it. The modern subset — `pipeline`, async iteration, and one rule about `write()`'s return value — is small and worth internalising.

<a id="p7-beginner"></a>

### Beginner: why streams exist

Here is a file copy that works perfectly until it doesn't:

```js
// ✗ wrong — allocates the entire file in memory, twice
const data = await readFile('input.mp4');
await writeFile('output.mp4', data);
```

With a 50 KB file, fine. With a 5 GB file, your process dies — or worse, ten concurrent requests each doing this die together. A stream version has a fixed memory cost regardless of file size:

```js
// ✓ right — constant memory, whatever the size
import { pipeline } from 'node:stream/promises';
import { createReadStream, createWriteStream } from 'node:fs';

await pipeline(createReadStream('input.mp4'), createWriteStream('output.mp4'));
```

That is the whole pitch: **process data in chunks as it arrives, instead of collecting it all first.** Everything else in this Part is mechanism.

**Try It — watch memory stay flat.** Make a large file and copy it both ways:

```bash
node -e "require('fs').writeFileSync('big.bin', Buffer.alloc(500e6))"

node --max-old-space-size=128 -e "
  const {readFile,writeFile}=require('fs/promises');
  readFile('big.bin').then(d=>writeFile('copy1.bin',d));
"
```

Expected output: a crash — `RangeError`/heap out of memory, or an `ERR_FS_FILE_TOO_LARGE`. Now the stream version, with the same 128 MB cap:

```bash
node --max-old-space-size=128 -e "
  const {pipeline}=require('stream/promises'), fs=require('fs');
  pipeline(fs.createReadStream('big.bin'), fs.createWriteStream('copy2.bin'));
"
```

Expected output: nothing at all, and a copied file. Same machine, same limit, 500 MB moved through a 128 MB heap.

<a id="p7-working"></a>

### Working Knowledge: `pipeline`, and the four stream types

#### The four types

| Type | Direction | Examples |
|---|---|---|
| **Readable** | source | `fs.createReadStream()`, an HTTP request, `process.stdin` |
| **Writable** | destination | `fs.createWriteStream()`, an HTTP response, `process.stdout` |
| **Duplex** | both, independent | a TCP socket |
| **Transform** | both, connected | `zlib.createGzip()`, a CSV parser, your own mapper |

They connect into a pipeline, and the picture is worth having in your head:

```
   Readable ──► Transform ──► Transform ──► Writable
   (file)       (gunzip)      (parse)       (database)
      │            │             │              │
      └── each link has its own buffer (highWaterMark) ──┘
              and each can say "slow down" to the one behind it
```

#### Always use `pipeline`

`.pipe()` is the old way and it has a real defect: **it does not clean up on error** ([stream docs](https://nodejs.org/docs/latest-v24.x/api/stream.html)). If the destination fails, the source keeps reading, and its file descriptor leaks:

```js
// ✗ wrong — an error in gzip or the destination leaves the source open forever
createReadStream(src).pipe(createGzip()).pipe(createWriteStream(dst));

// ✓ right — every stream is destroyed on any error, and you get one place to handle it
import { pipeline } from 'node:stream/promises';
await pipeline(createReadStream(src), createGzip(), createWriteStream(dst));
```

The promise version from `node:stream/promises` is the one to use; it also accepts an `AbortSignal`:

```js
await pipeline(source, transform, destination, { signal: AbortSignal.timeout(30_000) });
```

#### Consuming a readable

Three ways, in descending order of how often you should reach for them:

```js
// 1. async iteration — backpressure is automatic, errors throw where you can catch them
for await (const chunk of readable) {
  await handle(chunk);        // nothing more is read while you're in here
}

// 2. pipeline — when you're just moving bytes somewhere
await pipeline(readable, writable);

// 3. events — only when you need fine control
readable.on('data', (c) => {});
readable.on('end', () => {});
readable.on('error', (e) => {});
```

Async iteration is the quiet hero: the `await` inside the loop body *is* the backpressure signal, so a slow consumer automatically slows the producer with no code on your part.

#### Making streams

```js
import { Readable, Writable, Transform } from 'node:stream';

// from any iterable or async generator
const lines = Readable.from(['a\n', 'b\n']);

// a Transform, the modern way — as an async generator inside pipeline
await pipeline(
  source,
  async function* (chunks) {
    for await (const chunk of chunks) yield chunk.toString().toUpperCase();
  },
  destination,
);
```

That generator form is worth noticing: you can put a plain async generator into `pipeline` and it becomes a Transform. Most of the time you never need to write a `Transform` subclass at all.

<a id="p7-advanced"></a>

### Advanced: backpressure, explicitly

This is the core idea, and it is one sentence: **`writable.write()` returns `false` when its buffer is full, and ignoring that is how Node processes run out of memory.**

```js
const ok = writable.write(chunk);
// true  → buffered below highWaterMark, keep going
// false → buffer is at or over highWaterMark; STOP, wait for 'drain'
```

Crucially, `write()` **still accepts** the data when it returns `false`. It is advice, not a rejection. Which is why the wrong version looks like it works:

```js
// ✗ wrong — reads as fast as the disk allows, buffers everything the socket can't send.
//   Works perfectly on localhost. Kills the process for a user on hotel wifi.
for await (const chunk of readable) {
  res.write(chunk);
}

// ✓ right — respect the signal
for await (const chunk of readable) {
  if (!res.write(chunk)) await once(res, 'drain');
}

// ✓✓ better — pipeline does all of this for you
await pipeline(readable, res);
```

The failure shape is distinctive and worth memorising: **memory that grows in proportion to the speed mismatch between a fast source and a slow destination.** Fast disk, slow network. Fast producer, slow database. It is invisible locally, where both are fast.

**Real Scenario — the export endpoint that only died in production.** A `/export.csv` route streams a few hundred thousand rows from Postgres to the response with `res.write(row)` in a loop, no return-value check. It passes review, passes staging, and runs fine for months — every internal user is on the office network, fast enough that the socket never backs up. Then a customer downloads it over a mobile connection. Postgres delivers rows at disk speed, the socket drains at 300 KB/s, and the difference accumulates in the response's buffer until the container hits its memory limit and the orchestrator kills it — taking every other in-flight request with it. The diff that fixed it was replacing the loop with a single `await pipeline(rowStream, res)`.

#### `highWaterMark` — the number that defines "full"

```js
createReadStream(p, { highWaterMark: 256 * 1024 });   // read in 256 KB chunks
new Writable({ objectMode: true, highWaterMark: 32 }); // 32 objects
```

Defaults are **64 KiB for byte streams** and **16 for object mode** ([stream docs](https://nodejs.org/docs/latest-v24.x/api/stream.html)). The byte default was raised from 16 KiB in a recent release line, so older articles quoting 16384 are describing a value that changed under them — and rather than trusting either number, ask the runtime:

```bash
node -p "const s=require('stream'); [s.getDefaultHighWaterMark(false), s.getDefaultHighWaterMark(true)]"
```

Raising it trades memory for fewer, larger reads; lowering it makes backpressure kick in sooner. Neither is worth tuning until you have measured, and "make it bigger" is very rarely the fix people hope it is.

#### Errors and cleanup

```js
// one place to handle everything
try {
  await pipeline(a, b, c);
} catch (err) {
  if (err.code === 'ERR_STREAM_PREMATURE_CLOSE') return;   // client hung up — usually fine
  throw err;
}
```

`ERR_STREAM_PREMATURE_CLOSE` is the one to recognise: it means a stream closed before it finished, and in an HTTP server it almost always means the user closed the tab. Logging it as an error will fill your error budget with nothing.

<a id="p7-mastery"></a>

### Mastery: object mode, `compose`, and Web Streams interop

#### Object mode

A stream can carry any JavaScript value, not just bytes. That turns pipelines into a composable data-processing model, with backpressure included free:

```js
import { pipeline } from 'node:stream/promises';
import { Readable } from 'node:stream';

await pipeline(
  Readable.from(rowsFromDatabase()),              // an async generator of objects
  async function* (rows) {                        // filter
    for await (const r of rows) if (r.active) yield r;
  },
  async function* (rows) {                        // map to CSV lines
    for await (const r of rows) yield `${r.id},${r.email}\n`;
  },
  createWriteStream('active.csv'),
);
```

Constant memory over a table of any size, with each stage applying backpressure to the one before it. This is the pattern that makes ETL work in Node, and it needs no library.

The subtlety: **in object mode, `highWaterMark` counts objects, not bytes.** Sixteen objects of unknown size is a memory bound only if the objects are bounded. A pipeline of 16 buffered 50 MB records is 800 MB, and the stream machinery considers that entirely within limits.

#### `compose` — a pipeline as a value

```js
import { compose } from 'node:stream';

const parseNdjson = compose(
  splitLines(),
  async function* (lines) { for await (const l of lines) if (l.trim()) yield JSON.parse(l); },
);

await pipeline(createReadStream('events.ndjson'), parseNdjson, handler);
```

`compose` turns several stages into one reusable Duplex, which is how you build a library of pipeline pieces instead of re-writing them per script.

#### Web Streams interop

Node has both its own streams and the WHATWG `ReadableStream` that `fetch` returns. They convert in both directions ([stream docs](https://nodejs.org/docs/latest-v24.x/api/stream.html)):

```js
import { Readable, Writable } from 'node:stream';

// fetch gives you a Web ReadableStream — bring it into Node's world
const res = await fetch(url);
await pipeline(Readable.fromWeb(res.body), createWriteStream('download.bin'));

// or the reverse, for an API that wants Web streams
const web = Readable.toWeb(createReadStream('x.bin'));
```

In practice: **use Node streams inside Node, and convert at the boundary.** Node's streams have the better ecosystem, `pipeline` has better error semantics than manual `pipeTo` chains, and Web Streams are what cross-runtime APIs speak. The one case for going Web-native throughout is code that must also run in Workers or the browser.

> **vs. Bun** — Bun implements `node:stream` for compatibility but its native APIs (`Bun.file().stream()`, `Bun.write()`) are Web Streams first, so idiomatic Bun code sits on the other side of this boundary. Node code using `pipeline` generally runs unchanged on Bun; the reverse is less often true. See [`bun-apis-io-processes.md`](../bun/bun-apis-io-processes.md).

#### The chunk-boundary bug

A `Buffer` chunk can end mid-character, which — as [Part 6's Try It](#p6-advanced) showed — produces a replacement character. Calling `.toString()` per chunk is therefore wrong for any multi-byte text:

```js
// ✗ wrong — a chunk boundary inside a multi-byte character corrupts it
for await (const chunk of stream) process.stdout.write(chunk.toString('utf8'));

// ✓ right — StringDecoder holds partial characters back until they're complete
import { StringDecoder } from 'node:string_decoder';
const decoder = new StringDecoder('utf8');
for await (const chunk of stream) process.stdout.write(decoder.write(chunk));
process.stdout.write(decoder.end());

// ✓✓ also right — let the stream decode
for await (const text of createReadStream(p, { encoding: 'utf8' })) { /* strings */ }
```

This bug is rare in testing (ASCII fixtures, small files that arrive in one chunk) and reliably present in production the moment a user has an accent in their name.

<a id="part-7-cheat-sheet"></a>

### Part 7 cheat sheet

| Want | How |
|---|---|
| Connect streams safely | `await pipeline(a, b, c)` from `node:stream/promises` |
| Never use | `.pipe()` chains — no cleanup on error |
| Consume a readable | `for await (const chunk of readable)` |
| Build one from data | `Readable.from(iterable)` |
| A transform | An `async function*` passed straight to `pipeline` |
| Reusable multi-stage piece | `compose(...)` |
| Cancel a pipeline | `pipeline(…, { signal })` |
| Backpressure, by hand | `if (!w.write(c)) await once(w, 'drain')` |
| What `write() === false` means | Buffer is full — the data was accepted anyway |
| Buffer size | `highWaterMark`: 64 KiB bytes / 16 objects |
| Check the real defaults | `stream.getDefaultHighWaterMark(objectMode)` |
| Decode text across chunks | `StringDecoder`, or `{ encoding: 'utf8' }` on the stream |
| Web → Node | `Readable.fromWeb(res.body)` |
| Node → Web | `Readable.toWeb(nodeStream)` |
| Client disconnected | `err.code === 'ERR_STREAM_PREMATURE_CLOSE'` |

**Gotchas:** ignoring `write()`'s return value is the classic OOM · object-mode `highWaterMark` counts objects, so big objects blow past any memory bound · `chunk.toString()` per chunk corrupts multi-byte text · `.pipe()` leaks descriptors on error.

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 8 — The Process](./nodejs-processes-workers.md) · **Previous:** [Parts 4–5](./nodejs-modules-packages.md) · **Up:** [Overview](./nodejs-mastery-guide.md)

*Written against Node.js 24 LTS · Verified September 14, 2026*
