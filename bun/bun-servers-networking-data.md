# Bun — Servers, Networking & Data (Parts 9–10)

Answering requests and storing rows: `Bun.serve` and its routing, WebSockets, the socket APIs underneath, then `bun:sqlite` in depth with `Bun.sql` and Redis surveyed.

> **Spec:** this doc follows the shared spec in [`bun-mastery-guide.md`](./bun-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Mastery tiers (collapsed where a tier would be filler, always stated), **Try It** exercises with expected output, wrong-vs-right pairs, official docs cited inline. Written against **Bun v1.4.2**, verified **September 6, 2026**.
>
> **Scope note:** `Bun.sql` (Postgres/MySQL) and the Redis client are a **survey** here by design — enough to use them correctly, not a database course. `bun:sqlite` gets the full treatment because it's the one with no setup cost, so it's the one you'll actually experiment with.

---

## Table of Contents

- [Part 9 — HTTP & Sockets](#part-9--http--sockets)
  - [Beginner: a server in four lines](#p9-beginner)
  - [Working Knowledge: routes, params, and Response](#p9-working)
  - [Advanced: WebSockets](#p9-advanced)
  - [Advanced: errors, timeouts, and graceful shutdown](#p9-lifecycle)
  - [Mastery: TCP/UDP, HTTP/2 and /3, and `server.reload`](#p9-mastery)
  - [Part 9 cheat sheet](#part-9-cheat-sheet)
- [Part 10 — Data](#part-10--data)
  - [Beginner: `bun:sqlite` in ten lines](#p10-beginner)
  - [Working Knowledge: statements, binding, and result shapes](#p10-working)
  - [Advanced: transactions, WAL, and integer safety](#p10-advanced)
  - [Mastery survey: `Bun.sql` and Redis](#p10-mastery)
  - [Part 10 cheat sheet](#part-10-cheat-sheet)

---

## Part 9 — HTTP & Sockets

<a id="p9-beginner"></a>

### Beginner: a server in four lines

```ts
Bun.serve({
  port: 3000,
  fetch: (req) => new Response("Hello!"),
});
```

```bash
bun server.ts
curl localhost:3000    # → Hello!
```

The handler takes a web-standard `Request` and returns a web-standard `Response` — the same objects you use with `fetch` in a browser or a service worker ([HTTP docs](https://bun.sh/docs/api/http)).

> **vs. Node** — Node's `http.createServer((req, res) => …)` gives you two Node-specific stream objects and expects you to call `res.end()`. Bun gives you one standard `Request` and wants one standard `Response` back. The practical consequence: your handler is a pure function, trivially testable without starting a server, and portable to Cloudflare Workers or Deno unchanged.

Port defaults to `$BUN_PORT`, `$PORT`, `$NODE_PORT`, then `3000`. Hostname defaults to `0.0.0.0`.

<a id="p9-working"></a>

### Working Knowledge: routes, params, and Response

Since 1.2.3 the idiomatic form is the **`routes` object**, not URL parsing in `fetch` ([HTTP docs](https://bun.sh/docs/api/http)):

```ts
const server = Bun.serve({
  port: 3000,
  routes: {
    // static — the Response is built once, not per request
    "/health": new Response("OK"),

    // dynamic params
    "/users/:id": (req) => Response.json({ id: req.params.id }),

    // per-method handlers
    "/api/posts": {
      GET: () => Response.json(posts),
      POST: async (req) => {
        const body = await req.json();
        return Response.json({ created: true, ...body }, { status: 201 });
      },
    },

    // files, redirects, wildcards
    "/favicon.ico": Bun.file("./favicon.ico"),
    "/old": Response.redirect("/new", 301),
    "/api/*": Response.json({ error: "not found" }, { status: 404 }),
  },

  // fallback for anything routes didn't match
  fetch: () => new Response("Not Found", { status: 404 }),
});

console.log(`listening on ${server.url}`);
```

```ts
// ❌ pre-1.2.3 style, still seen everywhere — manual, error-prone routing
Bun.serve({
  fetch(req) {
    const url = new URL(req.url);
    if (url.pathname.startsWith("/users/")) {
      const id = url.pathname.split("/")[2];
      return Response.json({ id });
    }
    return new Response("Not Found", { status: 404 });
  },
});

// ✅ declarative, with params parsed for you
Bun.serve({
  routes: { "/users/:id": (req) => Response.json({ id: req.params.id }) },
});
```

**Static routes are genuinely different, not just tidier.** `"/health": new Response("OK")` constructs the response **once at startup** and serves that object to every request — no allocation, no handler call. For health checks and static assets, that is a measurable difference under load.

The `Response` API is the standard one, so all of this is free:

```ts
Response.json(obj, { status: 201, headers: { "X-Foo": "bar" } });
Response.redirect("/elsewhere", 302);
new Response(Bun.file("./video.mp4"));       // streamed, not buffered
new Response(someReadableStream);
new Response(null, { status: 204 });
```

**Try It — a two-route API.**

```ts
// api.ts
const todos = new Map<string, { id: string; text: string }>();

Bun.serve({
  port: 3000,
  routes: {
    "/todos": {
      GET: () => Response.json([...todos.values()]),
      POST: async (req) => {
        const { text } = await req.json();
        const id = Bun.randomUUIDv7();
        todos.set(id, { id, text });
        return Response.json({ id, text }, { status: 201 });
      },
    },
    "/todos/:id": (req) => {
      const todo = todos.get(req.params.id);
      return todo ? Response.json(todo) : new Response("gone", { status: 404 });
    },
  },
});
```

```bash
bun --hot api.ts
curl -X POST localhost:3000/todos -H 'content-type: application/json' -d '{"text":"read part 9"}'
curl localhost:3000/todos
```

Expected: the POST returns `201` with a generated `id`; the GET returns an array containing it. Now edit a handler while it runs — under `--hot` the map keeps its contents across the reload ([Part 1](./bun-runtime-foundations.md#p1-advanced)), which is exactly why `--hot` is worth using for API work.

<a id="p9-advanced"></a>

### Advanced: WebSockets

WebSockets start as HTTP requests that get **upgraded**. In Bun, that upgrade happens inside `fetch`, and the handlers live in a separate `websocket` object ([WebSocket docs](https://bun.sh/docs/api/websockets)):

```ts
type WSData = { userId: string; room: string };

const server = Bun.serve<WSData, {}>({
  port: 3000,
  fetch(req, server) {
    const url = new URL(req.url);
    if (url.pathname === "/chat") {
      // attach per-connection data at upgrade time
      const ok = server.upgrade(req, {
        data: { userId: url.searchParams.get("user")!, room: "general" },
      });
      return ok ? undefined : new Response("upgrade failed", { status: 400 });
    }
    return new Response("Not Found", { status: 404 });
  },

  websocket: {
    open(ws) {
      ws.subscribe(ws.data.room);
      server.publish(ws.data.room, `${ws.data.userId} joined`);
    },
    message(ws, message) {
      server.publish(ws.data.room, `${ws.data.userId}: ${message}`);
    },
    close(ws) {
      ws.unsubscribe(ws.data.room);
      server.publish(ws.data.room, `${ws.data.userId} left`);
    },
    drain(ws) { /* socket is ready for more data */ },
  },
});
```

Two structural points that trip people up:

1. **Handlers are defined once per server, not per socket.** This is deliberate — one set of closures for ten thousand connections instead of ten thousand sets. Per-connection state lives in `ws.data`, populated at `upgrade` time.
2. **`server.upgrade()` returns a boolean, and on success your `fetch` must return `undefined`.** Returning a `Response` after a successful upgrade is a bug.

#### Pub/sub is built in

```ts
ws.subscribe("room:42");
ws.unsubscribe("room:42");
ws.isSubscribed("room:42");
ws.subscriptions;                     // string[]
server.publish("room:42", payload);   // to all subscribers except the sender
```

This is a real feature, not sugar: broadcasting to a topic from `server.publish` is handled natively rather than by iterating a JavaScript `Set` of sockets and calling `send` on each.

#### Backpressure — the part people skip

`ws.send()` **returns a number**, and ignoring it is how a chat server eats all your memory ([WebSocket docs](https://bun.sh/docs/api/websockets)):

| Return | Meaning |
|---|---|
| `> 0` | bytes sent |
| `-1` | enqueued, **backpressure exists** — the client isn't keeping up |
| `0` | dropped — connection problem |

```ts
// ❌ a fast producer + a slow client = unbounded queue growth = OOM
for (const tick of priceFeed) ws.send(JSON.stringify(tick));

// ✅ respect the signal; resume in drain()
let paused = false;
for (const tick of priceFeed) {
  if (paused) break;
  if (ws.send(JSON.stringify(tick)) === -1) paused = true;
}
// websocket: { drain(ws) { paused = false; /* resume */ } }
```

Configuration for the same concern:

```ts
websocket: {
  idleTimeout: 120,                  // seconds
  maxPayloadLength: 16 * 1024 * 1024,
  backpressureLimit: 16 * 1024 * 1024,
  closeOnBackpressureLimit: true,    // drop the slow client rather than buffer forever
  sendPings: true,
  perMessageDeflate: true,
}
```

`closeOnBackpressureLimit: true` is the blunt, correct default for a public server: a client that cannot keep up gets disconnected instead of consuming your RAM.

<a id="p9-lifecycle"></a>

### Advanced: errors, timeouts, and graceful shutdown

```ts
const server = Bun.serve({
  routes: { /* … */ },

  error(err) {
    console.error(err);
    return new Response("Internal Server Error", { status: 500 });
  },

  idleTimeout: 30,        // seconds; 0 disables
});
```

Without an `error` handler, a throwing handler produces a default 500 — and in development, a rather good error page. Supply one in production so you control what leaks into the response body.

Per-request timeout override, and useful introspection:

```ts
server.timeout(req, 120);        // this request may take 2 minutes
server.requestIP(req);           // { address, port, family }
server.pendingRequests;
server.pendingWebSockets;
server.closeIdleConnections();
```

Graceful shutdown:

```ts
process.on("SIGINT", async () => {
  await server.stop();       // stop accepting; let in-flight requests finish
  // await server.stop(true) → close active connections immediately
  process.exit(0);
});
```

```ts
// ❌ in-flight requests are cut off mid-response
process.on("SIGINT", () => process.exit(0));

// ✅ drain first
process.on("SIGINT", async () => { await server.stop(); process.exit(0); });
```

This matters more than it looks in a container: an orchestrator sends `SIGTERM` and then kills you after a grace period. The difference between the two versions is whether a deploy produces a handful of failed requests every time.

<a id="p9-mastery"></a>

### Mastery: TCP/UDP, HTTP/2 and /3, and `server.reload`

#### Below HTTP

```ts
// TCP server
Bun.listen({
  hostname: "localhost",
  port: 8080,
  socket: {
    data(socket, data) { socket.write(data); },   // echo
    open(socket) {},
    close(socket) {},
    error(socket, err) {},
  },
});

// TCP client
const socket = await Bun.connect({ hostname: "localhost", port: 8080, socket: { data() {} } });

// UDP
const udp = await Bun.udpSocket({ socket: { data(sock, buf, port, addr) {} } });
```

Same design as the WebSocket API: handlers defined once, not per connection.

#### TLS, HTTP/2, HTTP/3, Unix sockets

```ts
Bun.serve({
  tls: { key: Bun.file("./key.pem"), cert: Bun.file("./cert.pem") },
  http2: true,          // experimental
  http3: true,          // experimental (QUIC); requires TLS
  // unix: "/tmp/app.sock",   // instead of port
  fetch: () => new Response("secure"),
});
```

HTTP/2 and HTTP/3 are labelled **experimental** as of 1.4.2 ([HTTP docs](https://bun.sh/docs/api/http)) — re-check that label before relying on either, and note that in most deployments a reverse proxy terminates these anyway.

The Unix socket option is the neat one for containers: your app listens on a socket file, the proxy connects to it, and no TCP port is exposed at all.

#### `server.reload` — swapping handlers on a live server

```ts
server.reload({
  routes: { "/api/version": () => Response.json({ version: "2.0.0" }) },
});
```

This is the mechanism `--hot` uses under the hood: the listening socket is untouched while the request handlers are replaced. Knowing it exists as an API means you can build feature-flagged route swaps or config reloads without dropping a connection.

<a id="part-9-cheat-sheet"></a>

### Part 9 cheat sheet

| Want | How |
|---|---|
| Start a server | `Bun.serve({ port, routes, fetch })` |
| Path param | `"/u/:id"` → `req.params.id` |
| Method-specific | `"/p": { GET: fn, POST: fn }` |
| Zero-cost static route | `"/health": new Response("OK")` |
| Serve a file | `new Response(Bun.file(p))` |
| JSON out | `Response.json(obj, { status })` |
| Catch-all | `"/api/*": …` plus a top-level `fetch` |
| Upgrade to WS | `server.upgrade(req, { data })` → return `undefined` |
| Per-connection state | `ws.data` |
| Broadcast | `ws.subscribe(t)` + `server.publish(t, msg)` |
| Handle backpressure | check `ws.send()` for `-1`, resume in `drain` |
| Drop slow clients | `closeOnBackpressureLimit: true` |
| Error page | `error(err) { return new Response(…) }` |
| Client IP | `server.requestIP(req)` |
| Graceful shutdown | `await server.stop()` |
| Swap routes live | `server.reload({ routes })` |
| Raw TCP / UDP | `Bun.listen` / `Bun.connect` / `Bun.udpSocket` |

[↑ Back to top](#table-of-contents)

---

## Part 10 — Data

<a id="p10-beginner"></a>

### Beginner: `bun:sqlite` in ten lines

SQLite is built into the Bun binary. No `npm install`, no native compile, no `trustedDependencies` ([SQLite docs](https://bun.sh/docs/api/sqlite)):

```ts
import { Database } from "bun:sqlite";

const db = new Database("app.sqlite", { create: true, strict: true });

db.run(`CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT)`);
db.query(`INSERT INTO users (name) VALUES ($name)`).run({ name: "Alice" });

const users = db.query(`SELECT * FROM users`).all();
console.log(users);   // [ { id: 1, name: "Alice" } ]
```

In-memory, for tests and experiments — all three are equivalent:

```ts
new Database(":memory:");
new Database();
new Database("");
```

Constructor options: `readonly`, `create`, `readwrite`, `safeIntegers`, `strict`.

**Turn on `strict: true`.** By default, a typo in a bound parameter name is silently ignored:

```ts
// ❌ default mode: the typo binds nothing, the query returns garbage, no error
const db = new Database(":memory:");
db.query("SELECT $message").all({ messag: "typo" });   // no throw

// ✅ strict mode: throws on the typo, and you can drop the $ prefix
const db = new Database(":memory:", { strict: true });
db.query("SELECT $message").all({ message: "ok" });
```

<a id="p10-working"></a>

### Working Knowledge: statements, binding, and result shapes

#### `.query()` vs. `.prepare()` vs. `.run()`

| Method | What it does | Returns |
|---|---|---|
| `db.query(sql)` | Prepares **and caches** the compiled statement | `Statement` |
| `db.prepare(sql)` | Prepares without caching | `Statement` |
| `db.run(sql)` | Executes immediately, no caching | `{ lastInsertRowid, changes }` |

**`db.query()` is the one you want in a loop or a request handler**, because SQLite's compile step is real work and the cache eliminates it from the second call onward. `db.prepare()` is for statements you build dynamically and don't want filling the cache; `db.run()` is for one-off DDL.

```ts
// ❌ recompiles the same SQL on every request
app.get("/u/:id", (req) => db.prepare("SELECT * FROM users WHERE id = ?").get(req.params.id));

// ✅ compile once at module scope, reuse forever
const getUser = db.query("SELECT * FROM users WHERE id = ?");
app.get("/u/:id", (req) => getUser.get(req.params.id));
```

#### Getting results out

```ts
const q = db.query("SELECT id, name FROM users");

q.all();      // [{ id: 1, name: "Alice" }, …]
q.get();      // { id: 1, name: "Alice" } | null   ← first row only
q.values();   // [[1, "Alice"], …]                 ← arrays, not objects
q.run();      // { lastInsertRowid, changes }      ← for writes

for (const row of q.iterate()) { /* lazy — doesn't materialise the result set */ }
```

`iterate()` is the one to reach for on a large table: `all()` builds every row in memory first, `iterate()` walks the cursor.

#### Binding parameters

```ts
// named
db.query("SELECT * FROM users WHERE id = $id").get({ $id: 1 });
// with strict: true, no prefix needed
db.query("SELECT * FROM users WHERE id = $id").get({ id: 1 });

// positional
db.query("SELECT ?1, ?2").all("hello", "goodbye");
db.query("SELECT * FROM users WHERE id = ?").get(1);
```

```ts
// ❌ string interpolation — SQL injection, and no statement caching either
db.query(`SELECT * FROM users WHERE name = '${name}'`).all();

// ✅ bound parameter
db.query("SELECT * FROM users WHERE name = ?").all(name);
```

#### Mapping rows to a class

```ts
class User {
  id!: number;
  name!: string;
  get initials() { return this.name.split(" ").map((w) => w[0]).join(""); }
}

const users = db.query("SELECT id, name FROM users").as(User).all();
users[0].initials;   // getters and methods work
```

`.as()` sets the prototype rather than copying fields, so it costs essentially nothing per row.

<a id="p10-advanced"></a>

### Advanced: transactions, WAL, and integer safety

#### Transactions

`db.transaction()` wraps a function; it commits on return and rolls back on throw ([SQLite docs](https://bun.sh/docs/api/sqlite)):

```ts
const insertUser = db.prepare("INSERT INTO users (name) VALUES ($name)");

const insertMany = db.transaction((users: { name: string }[]) => {
  for (const u of users) insertUser.run(u);
  return users.length;
});

const count = insertMany([{ name: "Keanu" }, { name: "Salem" }]);

// explicit locking modes
insertMany.deferred(users);
insertMany.immediate(users);
insertMany.exclusive(users);
```

The performance difference is not marginal. SQLite commits each un-wrapped statement separately, meaning an `fsync` each time:

```ts
// ❌ 10,000 separate transactions, 10,000 disk syncs — seconds
for (const u of users) insertUser.run(u);

// ✅ one transaction — typically milliseconds
insertMany(users);
```

Nested `transaction()` calls become savepoints, so composing transactional helpers works correctly.

#### WAL mode

```ts
db.run("PRAGMA journal_mode = WAL;");
```

Write-Ahead Logging lets readers proceed while a write is in progress, instead of blocking on a database-level lock. For any server touching SQLite concurrently this is close to mandatory — and it is not the default.

Clean shutdown, so you don't leave `-wal` and `-shm` sidecar files behind:

```ts
import { Database, constants } from "bun:sqlite";

db.fileControl(constants.SQLITE_FCNTL_PERSIST_WAL, 0);
db.run("PRAGMA wal_checkpoint(TRUNCATE);");
db.close();
```

#### Integers larger than `Number.MAX_SAFE_INTEGER`

SQLite integers are 64-bit; JavaScript numbers are IEEE doubles, exact only to 53 bits. By default Bun returns numbers, which **silently rounds** large values ([SQLite docs](https://bun.sh/docs/api/sqlite)):

```ts
// ❌ default: precision lost, no warning
new Database(":memory:").query("SELECT 9007199254741093 as big").get();
// → { big: 9007199254741092 }   ← note the last digit

// ✅ safeIntegers: exact, as BigInt
new Database(":memory:", { safeIntegers: true }).query("SELECT 9007199254741093 as big").get();
// → { big: 9007199254741093n }
```

If you store Twitter-style snowflake IDs, financial amounts in the smallest unit, or nanosecond timestamps, this is a live correctness bug and `safeIntegers: true` is the fix. The cost is that those columns come back as `BigInt`, which does not survive `JSON.stringify` without a custom replacer.

#### Automatic cleanup

```ts
{
  using db = new Database("app.sqlite");
  using q = db.query("SELECT * FROM users");
  q.get();
}   // db closed, statement finalized, automatically
```

`using` is the TypeScript/ES explicit-resource-management syntax — see [`typescript/typescript-objects-classes.md`](../typescript/typescript-objects-classes.md).

**Real Scenario — the migration that got slower every week.** A nightly job imported ~50,000 rows with a plain `for` loop of `.run()` calls. It took 40 seconds on day one and eleven minutes by month three, because every statement was its own transaction and every transaction was an `fsync` — the cost scaled with row count *and* with the growing table's index maintenance. Wrapping the loop in `db.transaction()` and enabling WAL took it under two seconds. Nothing about the SQL changed.

<a id="p10-mastery"></a>

### Mastery survey: `Bun.sql` and Redis

*This is a survey by scope decision — enough to be safe and productive, with the official docs linked for depth.*

#### `Bun.sql` — Postgres, MySQL, and SQLite behind one API

```ts
import { sql, SQL } from "bun";

// reads POSTGRES_URL / DATABASE_URL etc. from the environment
const users = await sql`SELECT * FROM users WHERE active = ${true}`;

// or construct explicitly; the adapter is detected from the URL scheme
const mysql = new SQL("mysql://user:pass@localhost:3306/mydb");
const lite  = new SQL("sqlite://./app.db");
```

Tagged templates are the whole safety story: **interpolated values become bound parameters, never string concatenation** ([SQL docs](https://bun.sh/docs/api/sql)).

```ts
// ❌ the injection you were trying to avoid
await sql.unsafe("SELECT * FROM users WHERE name = '" + name + "'");

// ✅ parameterised automatically
await sql`SELECT * FROM users WHERE name = ${name}`;
```

The helpers worth knowing:

```ts
// insert an object / bulk insert an array
await sql`INSERT INTO users ${sql({ name, email })} RETURNING *`;
await sql`INSERT INTO users ${sql(manyUsers)}`;
await sql`INSERT INTO users ${sql(user, "name", "email")}`;   // only these columns

// dynamic identifiers and fragments
await sql`SELECT * FROM ${sql("users")}`;
const ageFilter = sql`AND age > ${21}`;
await sql`SELECT * FROM users WHERE active = ${true} ${cond ? ageFilter : sql``}`;

// WHERE IN
await sql`SELECT * FROM users WHERE id IN ${sql([1, 2, 3])}`;
```

Transactions, pooling, and lifecycle:

```ts
await sql.begin(async (tx) => {
  await tx`INSERT INTO users (name) VALUES (${"Alice"})`;
  await tx`UPDATE accounts SET balance = balance - 100`;
  await tx.savepoint(async (sp) => { await sp`…`; });   // nested rollback point
});   // commits on return, rolls back on throw

const db = new SQL({ max: 20, idleTimeout: 30, maxLifetime: 3600, connectionTimeout: 10 });
await db.close({ timeout: 5 });
```

Two Postgres-specific capabilities that save you a dependency: `sql.listen(channel, cb)` / `sql.notify(channel, payload)` for LISTEN/NOTIFY pub/sub, and `bun --sql-preconnect` to establish the connection before your app code runs, removing first-request latency.

**The gotcha to carry with you:** large integers come back as **strings** by default, to avoid the same precision loss described above. Pass `new SQL({ bigint: true })` for `BigInt` instead. A `SELECT` on a `bigint` column that appears to return `"9223372036854777"` is not a bug.

#### Redis / Valkey

```ts
import { redis, RedisClient } from "bun";

// the default client reads REDIS_URL, then VALKEY_URL, else redis://localhost:6379
await redis.set("hello", "world");
await redis.get("hello");
await redis.incr("counter");
await redis.expire("session:abc", 3600);
await redis.hmset("user:123", ["name", "Alice", "email", "a@example.com"]);
```

Connections are lazy (the first command connects), auto-pipelined, and auto-reconnecting with exponential backoff ([Redis docs](https://bun.sh/docs/api/redis)).

**The pub/sub caveat that catches everyone** — and it's inherent to the Redis protocol, not a Bun quirk: a subscribed connection can only do subscription operations. Use `.duplicate()`:

```ts
// ❌ once subscribed, this client can't run normal commands
await redis.subscribe("events", handler);
await redis.set("k", "v");            // fails

// ✅ a second connection for the subscription
const subscriber = await redis.duplicate();
await subscriber.subscribe("events", handler);
await redis.set("k", "v");            // fine
```

Also documented: pub/sub is marked **experimental**, MULTI/EXEC transactions require raw `.send()` commands, and **Redis Sentinel and Cluster are not supported**. That last one is a genuine adoption blocker for some production setups — check it before you plan a migration.

<a id="part-10-cheat-sheet"></a>

### Part 10 cheat sheet

| Want | How |
|---|---|
| Open a database | `new Database("app.sqlite", { create: true, strict: true })` |
| In-memory | `new Database(":memory:")` |
| Cached statement | `db.query(sql)` ← prefer this |
| One-off / DDL | `db.run(sql)` |
| All rows / first row | `.all()` / `.get()` |
| Big result set | `.iterate()` |
| Bind | `.get({ id: 1 })` (strict) or `.get(1)` |
| Rows as class instances | `.as(MyClass)` |
| Fast bulk insert | `db.transaction(fn)` |
| Concurrent readers + writer | `PRAGMA journal_mode = WAL` |
| 64-bit integers, exactly | `{ safeIntegers: true }` → `BigInt` |
| Auto-close | `using db = new Database(…)` |
| Postgres / MySQL | ``await sql`SELECT …` `` |
| Insert an object | ``await sql`INSERT INTO t ${sql(obj)}` `` |
| Transaction | `await sql.begin(async tx => …)` |
| Postgres pub/sub | `sql.listen` / `sql.notify` |
| Redis | `import { redis } from "bun"` |
| Redis subscribe | `(await redis.duplicate()).subscribe(…)` |
| Not supported | Redis Sentinel & Cluster |

[↑ Back to top](#table-of-contents)

---

**Next:** [Parts 11–12 — Testing & Bundling](./bun-testing-bundling.md) · **Up:** [Overview](./bun-mastery-guide.md)

*Written against Bun v1.4.2 · Verified September 6, 2026*
