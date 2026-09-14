# Node.js — HTTP & Networking (Parts 10–11)

Node over the wire. A server built from `node:http` alone — so that every framework you use afterwards is recognisably a layer over it — plus the timeout settings that cause most production HTTP incidents; then the client side: `fetch` and the undici engine under it, connection pooling, timeouts that actually fire, and the lower layers (`net`, `tls`, `dgram`, WebSocket, DNS) for when HTTP isn't the right shape.

> **Spec:** this doc follows the shared spec in [`nodejs-mastery-guide.md`](./nodejs-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Mastery tiers (collapsed where a tier would be filler, always stated), **Try It** exercises with expected output, wrong-vs-right pairs, official docs cited inline. Written against **Node.js 24 LTS**, verified **September 14, 2026**.
>
> **Prerequisite:** [Part 7 — Streams & Backpressure](./nodejs-files-streams-buffers.md#part-7--streams--backpressure). A request is a Readable and a response is a Writable; this doc assumes you know what that implies.

---

## Table of Contents

- [Part 10 — HTTP Servers](#part-10--http-servers)
  - [Beginner: a server in eight lines](#p10-beginner)
  - [Working Knowledge: routing, bodies, and errors by hand](#p10-working)
  - [Advanced: the timeout matrix, and graceful shutdown](#p10-advanced)
  - [Mastery: keep-alive behind a load balancer, and the framework landscape](#p10-mastery)
  - [Part 10 cheat sheet](#part-10-cheat-sheet)
- [Part 11 — Clients & the Rest of the Stack](#part-11--clients--the-rest-of-the-stack)
  - [Beginner: `fetch`](#p11-beginner)
  - [Working Knowledge: timeouts, errors, and bodies you must consume](#p11-working)
  - [Advanced: connection pooling and undici](#p11-advanced)
  - [Mastery: `net`, `tls`, `dgram`, WebSocket, DNS](#p11-mastery)
  - [Part 11 cheat sheet](#part-11-cheat-sheet)

---

## Part 10 — HTTP Servers

Every Node web framework — Express, Fastify, Hono's Node adapter, Next.js's server — ends at `http.createServer`. Twenty minutes with the raw module removes a surprising amount of magic, and the timeout section in Advanced removes a surprising number of 502s.

<a id="p10-beginner"></a>

### Beginner: a server in eight lines

```js
// server.mjs
import { createServer } from 'node:http';

const server = createServer((req, res) => {
  res.writeHead(200, { 'content-type': 'application/json' });
  res.end(JSON.stringify({ path: req.url, method: req.method }));
});

server.listen(3000, () => console.log('http://localhost:3000'));
```

```bash
node server.mjs
curl -s localhost:3000/hello   # → {"path":"/hello","method":"GET"}
```

The handler runs once per request. `req` is an `IncomingMessage` — a **Readable stream** of the request body, plus headers and metadata. `res` is a `ServerResponse` — a **Writable stream** you write the body into. Everything else in this Part follows from those two sentences.

**The one bug every beginner writes:** forgetting `res.end()`.

```js
// ✗ wrong — the client waits forever (well, until a timeout)
createServer((req, res) => {
  if (req.url === '/health') res.end('ok');
  // any other URL: no response, ever
});

// ✓ right — every code path ends the response
createServer((req, res) => {
  if (req.url === '/health') return res.end('ok');
  res.statusCode = 404;
  res.end('not found');
});
```

<a id="p10-working"></a>

### Working Knowledge: routing, bodies, and errors by hand

#### Routing

`req.url` is a string containing the path *and* query. Parse it with the WHATWG `URL`, never the deprecated `url.parse()` ([Node 24 release notes](https://nodejs.org/en/blog/release/v24.0.0)):

```js
const url = new URL(req.url, `http://${req.headers.host}`);
url.pathname;                    // '/users/42'
url.searchParams.get('fields');  // 'name,email'
```

For path parameters, `URLPattern` is a global in Node 24 and does the matching a router would:

```js
const userRoute = new URLPattern({ pathname: '/users/:id' });

const match = userRoute.exec(url);
if (match) {
  const { id } = match.pathname.groups;   // '42'
}
```

That is most of what a router library does, in the standard library, with the same syntax browsers use.

#### Reading a body — it's a stream, and it has no size limit

```js
// ✗ wrong — no limit. A client can stream 10 GB and you will try to hold all of it.
async function readJson(req) {
  let body = '';
  for await (const chunk of req) body += chunk;
  return JSON.parse(body);
}

// ✓ right — cap it, decode it properly, and fail with the right status
async function readJson(req, limit = 1_000_000) {
  const chunks = [];
  let size = 0;
  for await (const chunk of req) {
    size += chunk.length;
    if (size > limit) throw Object.assign(new Error('Payload too large'), { status: 413 });
    chunks.push(chunk);
  }
  return JSON.parse(Buffer.concat(chunks).toString('utf8'));
}
```

Three fixes in one: a **size limit** (the unbounded version is a trivial memory-exhaustion attack), collecting `Buffer`s and decoding once at the end (the `body += chunk` form corrupts multi-byte characters at chunk boundaries — the [Part 7 bug](./nodejs-files-streams-buffers.md#p7-mastery)), and a status code on the error. Frameworks do exactly this; now you know what their `bodyLimit` option is for.

#### Errors: the handler is not wrapped in anything

An exception thrown inside an `async` handler is an unhandled rejection — which, per [Part 3](./nodejs-runtime-foundations.md#p3-advanced), **terminates the process**. `node:http` does not catch it for you:

```js
// ✓ a minimal error boundary — the thing every framework gives you
const server = createServer(async (req, res) => {
  try {
    await route(req, res);
  } catch (err) {
    const status = err.status ?? 500;
    if (status >= 500) log.error({ err, url: req.url }, 'request failed');
    if (!res.headersSent) {
      res.writeHead(status, { 'content-type': 'application/json' });
    }
    res.end(JSON.stringify({ error: status >= 500 ? 'Internal error' : err.message }));
  }
});

server.on('clientError', (err, socket) => {
  socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');   // malformed requests, before your handler
});
```

`res.headersSent` matters: if you had already started streaming a body, you cannot send a status code any more, and trying throws `ERR_HTTP_HEADERS_SENT`. The only honest move at that point is to destroy the socket, which the client sees as a truncated response.

#### Streaming responses

A response is a Writable, so [Part 7's](./nodejs-files-streams-buffers.md#part-7--streams--backpressure) rule applies: **use `pipeline`**, which respects backpressure and cleans up if the client disconnects mid-download.

```js
import { pipeline } from 'node:stream/promises';

res.writeHead(200, { 'content-type': 'text/csv' });
await pipeline(createReadStream('export.csv'), res);
```

**Try It — see a disconnect.** Serve a slow stream and hang up on it:

```js
// slow.mjs
import { createServer } from 'node:http';
import { pipeline } from 'node:stream/promises';
import { Readable } from 'node:stream';
import { setTimeout as sleep } from 'node:timers/promises';

createServer(async (req, res) => {
  const ticks = Readable.from((async function* () {
    for (let i = 0; ; i++) { yield `tick ${i}\n`; await sleep(500); }
  })());
  try { await pipeline(ticks, res); }
  catch (err) { console.log('pipeline ended:', err.code); }
}).listen(3000);
```

```bash
node slow.mjs
curl -N localhost:3000     # watch ticks arrive, then Ctrl-C
```

Expected output in the server terminal after Ctrl-C: `pipeline ended: ERR_STREAM_PREMATURE_CLOSE`. The generator was stopped too — without `pipeline`, it would keep ticking into a closed socket forever.

<a id="p10-advanced"></a>

### Advanced: the timeout matrix, and graceful shutdown

#### Four timeouts, four different jobs

This table is the most operationally important thing in this doc. Each setting guards a different phase of a connection ([http docs](https://nodejs.org/docs/latest-v24.x/api/http.html)):

```
   connection opened
        │
        ├── headersTimeout ──► the client must finish sending headers within this
        │                      default: min(requestTimeout, 60 s)  → 408, close
        │
        ├── requestTimeout ──► the client must finish sending the whole request
        │                      default: 300 s                      → 408, close
        │
        │   … your handler runs; NO built-in limit on how long it takes …
        │
        ├── response sent
        │
        └── keepAliveTimeout ► how long an idle connection waits for the NEXT request
                               default: 5 s                        → close
```

| Setting | Default | Guards against |
|---|---|---|
| `headersTimeout` | `min(requestTimeout, 60000)` | Slowloris — a client dribbling headers one byte at a time |
| `requestTimeout` | `300000` (5 min) | A client sending a body very slowly |
| `keepAliveTimeout` | `5000` | Idle connections piling up |
| `timeout` | `0` (none) | General socket inactivity; opt-in |

The critical gap is in the middle of the diagram: **nothing limits how long your handler takes.** A request that awaits a hung database call hangs until the *client* gives up. You must add that deadline yourself, typically with an `AbortSignal` passed down through every call (see [Part 3](./nodejs-runtime-foundations.md#p3-advanced)):

```js
createServer(async (req, res) => {
  const signal = AbortSignal.timeout(10_000);
  const data = await db.query(sql, { signal });   // ← the deadline follows the work
  res.end(JSON.stringify(data));
});
```

#### Graceful shutdown

The sequence that stops a deploy from severing in-flight requests:

```js
async function shutdown() {
  server.close();                  // 1. stop accepting; idle keep-alive sockets close too (19+)
  const timer = setTimeout(() => {
    server.closeAllConnections();  // 3. after a deadline, stop waiting for stragglers
  }, 10_000).unref();
  await once(server, 'close');     // 2. resolves once every connection has finished
  clearTimeout(timer);
  await db.end();
}
```

Two version notes worth knowing, because older articles contradict them. Since Node 19, **`server.close()` closes idle keep-alive connections itself** — previously an idle keep-alive socket would hold `close()` open until `keepAliveTimeout` expired, and you needed `closeIdleConnections()` to avoid that. And `closeAllConnections()` exists precisely so you no longer need to track sockets in a `Set` by hand, which is what every shutdown snippet from before Node 18 does ([http docs](https://nodejs.org/docs/latest-v24.x/api/http.html)).

#### `EADDRINUSE`

```
Error: listen EADDRINUSE: address already in use :::3000
```

Something already holds the port — almost always a previous instance of your own server that didn't shut down (a `--watch` restart racing an old process, a detached child, a zombie from a crashed terminal). `lsof -i :3000` (or `netstat -ano | findstr :3000` on Windows) names the PID. If this happens on every restart, your shutdown handler isn't closing the server — which is the previous section, not a port problem.

<a id="p10-mastery"></a>

### Mastery: keep-alive behind a load balancer, and the framework landscape

#### The 502 that only appears under moderate load

This one is famous enough to deserve full treatment, because it is invisible in every test environment.

```
   Load balancer                              Node server
   (idle timeout: 60 s)                       (keepAliveTimeout: 5 s)
        │                                           │
        │── request ──────────────────────────────►│
        │◄───────────────────────────── response ──│
        │                                           │
        │    connection idle… LB thinks it's good   │
        │    for 60 s                               │
        │                                           │── 5 s pass: Node closes it
        │── new request, reusing the connection ──►│  ✗ socket already closed
        │◄── RST ──────────────────────────────────│
        │
   502 Bad Gateway to the user
```

The load balancer keeps connections alive longer than Node does. Node closes an idle connection at 5 s; the balancer, believing it has 55 seconds left, sends the next request down it at precisely the wrong moment. Low traffic hides it (connections rarely get reused near the 5 s mark) and high traffic hides it (connections never sit idle for 5 s). Moderate traffic produces a steady trickle of 502s that nobody can reproduce.

**Real Scenario — the one-in-a-thousand 502.** A team migrates an API from Nginx-fronted VMs to an AWS ALB (default idle timeout 60 s). Error dashboards show a flat ~0.1% rate of 502s with no corresponding error in application logs — because from Node's perspective, nothing went wrong: it closed an idle socket, exactly as configured. The fix is one rule: **the upstream's keep-alive must outlive the load balancer's**.

```js
server.keepAliveTimeout = 65_000;   // longer than the LB's 60 s idle timeout
server.headersTimeout  = 66_000;    // must exceed keepAliveTimeout, or Node races itself
```

The second line matters as much as the first. If `headersTimeout` is shorter than `keepAliveTimeout`, a reused connection can be timed out *while waiting for the next request's headers*, recreating the same failure from the other side.

#### Why frameworks exist — and which is which

With the raw module in hand, a framework is recognisably four things: a router, body parsing with limits, an error boundary, and a middleware/plugin model. The landscape as of this writing:

| Framework | What it optimises for | Notes |
|---|---|---|
| **Express 5** | Ubiquity, ecosystem | 5.x finally catches async errors; most tutorials are still 4.x |
| **Fastify** | Throughput, schema-driven validation and serialisation | Plugin encapsulation model; strong TypeScript story |
| **Hono** | Portability across runtimes | Built on Web `Request`/`Response`; runs on Node, Bun, Deno, edge |
| **NestJS** | Structure for large teams | Angular-style DI and decorators, on top of Express or Fastify |

Choosing between them is outside this set's scope (see the [overview](./nodejs-mastery-guide.md#7-deliberately-not-covered-and-where-to-look-instead)). The point of this Part is that whichever you choose, the timeout matrix, body limits and shutdown sequence above are still yours to configure — frameworks set *some* of them, and the defaults they choose are worth reading.

> **vs. Bun** — `Bun.serve` is built on Web `Request`/`Response` objects rather than Node's stream-based `req`/`res`, with routes declared as an object. Hono-style code ports between the two; raw `node:http` code runs on Bun via its compatibility layer but is not idiomatic there. See [`bun-servers-networking-data.md`](../bun/bun-servers-networking-data.md).

<a id="part-10-cheat-sheet"></a>

### Part 10 cheat sheet

| Want | How |
|---|---|
| A server | `createServer((req, res) => …).listen(port)` |
| Parse the URL | ``new URL(req.url, `http://${req.headers.host}`)`` |
| Match a path param | `new URLPattern({ pathname: '/users/:id' }).exec(url)` |
| Read a JSON body | Collect `Buffer`s with a **size limit**, then `JSON.parse` once |
| Stream a response | `await pipeline(source, res)` |
| Can I still set status? | `res.headersSent` |
| Catch handler errors | Your own `try`/`catch` — `node:http` has none |
| Malformed requests | `server.on('clientError', …)` |
| Bound your handler's time | `AbortSignal.timeout(ms)`, passed down |
| Behind a load balancer | `keepAliveTimeout` > LB idle timeout; `headersTimeout` > `keepAliveTimeout` |
| Graceful shutdown | `server.close()` → await `'close'` → `closeAllConnections()` after a deadline |
| Who has my port? | `lsof -i :3000` |

**Gotchas:** a code path without `res.end()` hangs the client · no default body size limit · an `async` handler that throws kills the process · nothing limits handler duration · Node's 5 s keep-alive behind a 60 s load balancer produces sporadic 502s.

[↑ Back to top](#table-of-contents)

---

## Part 11 — Clients & the Rest of the Stack

Making requests *from* Node. `fetch` is built in and is the right default; the Advanced tier is about the engine underneath it, and Mastery drops below HTTP to the sockets it's made of.

<a id="p11-beginner"></a>

### Beginner: `fetch`

It's the browser API, globally available, no import ([Fetch guide](https://nodejs.org/en/learn/getting-started/fetch)):

```js
const res = await fetch('https://api.github.com/repos/nodejs/node');
const repo = await res.json();
console.log(repo.stargazers_count);

await fetch('https://httpbin.org/post', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ hello: 'world' }),
});
```

The rule that catches everyone who came from `axios`:

```js
// ✗ wrong — fetch does NOT reject on 404 or 500. This happily parses an error page.
const data = await fetch(url).then((r) => r.json());

// ✓ right — check status yourself
const res = await fetch(url);
if (!res.ok) throw new Error(`${res.status} ${res.statusText} for ${url}`);
const data = await res.json();
```

`fetch` rejects only when no HTTP response was obtained at all: DNS failure, connection refused, TLS error, abort. A 500 is a successful fetch of an unsuccessful response.

<a id="p11-working"></a>

### Working Knowledge: timeouts, errors, and bodies you must consume

#### `fetch` has no short default timeout

undici, the engine under Node's `fetch`, defaults `headersTimeout` and `bodyTimeout` to **300 seconds** each ([undici Client docs](https://github.com/nodejs/undici/blob/main/docs/docs/api/Client.md)). A slow upstream therefore holds your request — and whatever is awaiting it — for up to five minutes before anything fails. Always set a deadline:

```js
// ✗ wrong — can hang for minutes
const res = await fetch(url);

// ✓ right
const res = await fetch(url, { signal: AbortSignal.timeout(5_000) });
```

A timeout surfaces as an error with `name === 'TimeoutError'`; a manual `abort()` as `'AbortError'`. They are different on purpose — one is "too slow", the other is "we changed our minds".

#### Reading the real error

`fetch` network failures throw a `TypeError: fetch failed` whose *useful* information is in `err.cause`:

```js
try {
  await fetch('http://localhost:9999');
} catch (err) {
  console.log(err.message);       // 'fetch failed'           ← useless
  console.log(err.cause?.code);   // 'ECONNREFUSED'           ← the actual problem
}
```

Log `err.cause`. Every "fetch failed" with no further detail in a log file is a missing `.cause`.

#### Always consume or cancel the body

A response body is a stream holding a connection. If you never read it, the connection is not returned to the pool ([Fetch guide](https://nodejs.org/en/learn/getting-started/fetch)):

```js
// ✗ wrong — the body is never read; that socket is stuck until GC eventually notices
const res = await fetch(url);
if (!res.ok) throw new Error(res.status);

// ✓ right — cancel what you won't read
const res = await fetch(url);
if (!res.ok) {
  await res.body?.cancel();
  throw new Error(`HTTP ${res.status}`);
}
```

Under load, the wrong version shows up as requests queuing behind a connection pool that has quietly run dry — slowdowns with no errors at all.

#### Retries

Retry only what is safe to retry, with backoff and jitter:

```js
async function fetchWithRetry(url, init = {}, attempts = 3) {
  for (let i = 0; ; i++) {
    try {
      const res = await fetch(url, { ...init, signal: AbortSignal.timeout(5_000) });
      if (res.status < 500 || i === attempts - 1) return res;
      await res.body?.cancel();
    } catch (err) {
      if (i === attempts - 1) throw err;
    }
    await sleep(2 ** i * 200 + Math.random() * 100);   // 200, 400, 800 ms + jitter
  }
}
```

Two rules the snippet encodes: **never retry a 4xx** (the request is wrong; asking again won't fix it), and **be careful retrying non-idempotent methods** — a `POST` that timed out may well have succeeded, and retrying it charges the card twice. Idempotency keys exist for that case.

<a id="p11-advanced"></a>

### Advanced: connection pooling and undici

#### Every request to a host shares a pool

`fetch` reuses connections through undici's global dispatcher. For most applications that is fine. For a service making thousands of requests per second to one upstream, you want control:

```js
import { Agent, setGlobalDispatcher } from 'undici';

setGlobalDispatcher(new Agent({
  connections: 50,              // per origin
  keepAliveTimeout: 10_000,
  headersTimeout: 10_000,       // ← replace the 300 s default globally
  bodyTimeout: 30_000,
}));
```

`undici` here is the npm package — the same library Node bundles, but installed separately to expose its full API. Pin its major version to the one your Node ships with (undici 7 on Node 24, per the [release notes](https://nodejs.org/en/blog/release/v24.0.0)) to avoid two different copies behaving differently.

For a single hot upstream, a dedicated `Pool` gives you the lowest overhead undici offers:

```js
import { Pool } from 'undici';

const api = new Pool('https://internal-api', { connections: 20 });
const { statusCode, body } = await api.request({ path: '/v1/items', method: 'GET' });
const items = await body.json();
```

#### The legacy client still exists

`node:http`'s `http.request` / `http.get` predate `fetch` and still power much of npm (`axios` in Node, many SDKs). It pools through `http.Agent`, and one default changed under it: **since Node 19 the global agent keeps connections alive by default** ([Node 19 announcement](https://nodejs.org/en/blog/announcements/v19-release-announce)). Before that, every `http.get` opened a fresh TCP connection, which is why older performance guides tell you to create a `keepAlive: true` agent by hand — advice that is now redundant for the global agent, though still relevant if you construct your own `new http.Agent()`.

**Real Scenario — `ECONNRESET` after upgrading Node.** A service upgrades from Node 18 to 20 and starts seeing intermittent `ECONNRESET` on calls to an internal service. Nothing in the code changed. What changed is that the global HTTP agent now reuses connections, and the upstream — an old server with a 2 s keep-alive — closes idle sockets faster than the client expects. It is the [load-balancer 502](#p10-mastery) problem with the roles reversed. The fix is either to lower the client's idle timeout below the server's, or to retry idempotent requests on `ECONNRESET` — which is worth doing regardless, because a reused socket can always be closed out from under you.

<a id="p11-mastery"></a>

### Mastery: `net`, `tls`, `dgram`, WebSocket, DNS

*This tier is a survey rather than a deep dive: each module deserves more space than a single tier allows, and most applications only ever touch them through a library.*

#### TCP with `node:net`

HTTP is a text protocol over a TCP stream; `net` is the stream:

```js
import { createServer, connect } from 'node:net';

createServer((socket) => {
  socket.write('hello\n');
  socket.pipe(socket);                       // echo
}).listen(4000);

const client = connect(4000, 'localhost');
client.on('data', (d) => console.log(String(d)));
client.write('ping\n');
```

The thing to internalise: **TCP is a byte stream, not a message stream.** Two `write('ping\n')` calls can arrive as one `'data'` event, or one call can arrive as three. Any protocol on top needs framing — a delimiter or a length prefix — and "it works locally" is because loopback rarely fragments.

#### TLS

`node:tls` wraps a TCP socket; `node:https` is `http` over `tls`. In production TLS almost always terminates at a load balancer or proxy, and Node speaks plain HTTP behind it. When you do terminate in Node, the non-negotiables are to never set `rejectUnauthorized: false` outside a test (it disables certificate validation entirely — that is a man-in-the-middle vulnerability, not a config tweak), and to use `NODE_EXTRA_CA_CERTS=/path/ca.pem` for a private CA rather than turning validation off.

#### UDP with `node:dgram`

Datagrams: no connection, no ordering, no delivery guarantee, and very low overhead. Metrics shipping (StatsD), DNS and game state use it:

```js
import { createSocket } from 'node:dgram';
const sock = createSocket('udp4');
sock.send('page.views:1|c', 8125, 'localhost', () => sock.close());
```

#### WebSocket

The WebSocket **client** is a global in Node — the same API as the browser:

```js
const ws = new WebSocket('wss://echo.websocket.org');
ws.addEventListener('open', () => ws.send('hi'));
ws.addEventListener('message', (e) => console.log(e.data));
```

There is no built-in WebSocket **server** in Node. Servers still use a library (`ws` is the standard), attached to an `http.Server`'s `'upgrade'` event. Note from the [shutdown section](#p10-advanced) that `closeAllConnections()` does *not* close upgraded sockets — a WebSocket server needs its own shutdown step.

#### DNS — the `lookup` vs. `resolve` split

```js
import { lookup, resolve4 } from 'node:dns/promises';

await lookup('example.com');     // OS resolver: /etc/hosts, nsswitch — on the THREAD POOL
await resolve4('example.com');   // network DNS query via c-ares — does NOT use the thread pool
```

This distinction is the reason DNS appears in [Part 2's](./nodejs-runtime-foundations.md#p2-mastery) thread-pool diagram. `lookup` is what `http`, `net` and `fetch` use by default, because it honours `/etc/hosts` — and it occupies one of libuv's four threads for the duration. A slow resolver combined with heavy `fs` or `crypto` use can stall everything sharing that pool. High-throughput clients sometimes cache lookups or supply a custom `lookup` function for exactly this reason.

<a id="part-11-cheat-sheet"></a>

### Part 11 cheat sheet

| Want | How |
|---|---|
| A request | `await fetch(url, init)` |
| Treat 4xx/5xx as failure | `if (!res.ok) throw …` — fetch doesn't |
| A deadline | `{ signal: AbortSignal.timeout(5_000) }` |
| The real network error | `err.cause.code` (`ECONNREFUSED`, `ENOTFOUND`, …) |
| Discard a body you won't read | `await res.body?.cancel()` |
| Stream a download to disk | `pipeline(Readable.fromWeb(res.body), createWriteStream(p))` |
| Change global pooling/timeouts | `setGlobalDispatcher(new Agent({…}))` from `undici` |
| A pool for one upstream | `new Pool(origin, { connections })` from `undici` |
| Raw TCP | `node:net` — remember to frame messages |
| WebSocket client | Global `WebSocket` |
| WebSocket server | The `ws` package, on the server's `'upgrade'` event |
| Trust a private CA | `NODE_EXTRA_CA_CERTS=ca.pem` — never `rejectUnauthorized: false` |
| DNS without the thread pool | `resolve4()` rather than `lookup()` |

**Gotchas:** `fetch` doesn't reject on HTTP errors · undici waits up to 300 s by default · unread bodies leak pooled connections · "fetch failed" hides the cause in `err.cause` · TCP delivers bytes, not messages · `dns.lookup` uses the thread pool.

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 12 — `node:test` & Debugging](./nodejs-testing-debugging.md) · **Previous:** [Parts 8–9](./nodejs-processes-workers.md) · **Up:** [Overview](./nodejs-mastery-guide.md)

*Written against Node.js 24 LTS · Verified September 14, 2026*
