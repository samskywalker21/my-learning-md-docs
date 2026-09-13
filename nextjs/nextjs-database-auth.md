# Next.js — Database & Auth (Parts 8–9)

Connecting PostgreSQL with Drizzle ORM, and knowing who is asking — sessions, the Data Access Layer, and route protection.

> **Spec:** this doc follows the shared spec in [`nextjs-mastery-guide.md`](./nextjs-mastery-guide.md#about-this-document). Written against **Next.js 16.3.4**, `drizzle-orm` **0.45.2**, `next-auth` **5.0.0-beta.32**. Verified **September 11, 2026**.

> **Stack commitment, and an honest caveat.** These two Parts commit to **Postgres + Drizzle + Auth.js v5** so the code is real rather than hypothetical. Two things you should know before typing `npm install`:
>
> - **Auth.js v5 is still published as `next-auth@beta`.** `next-auth@latest` is **4.24.15** — the Pages-Router-era v4 — and the v5 line sits at `5.0.0-beta.32`. v5 has been in beta for over two years and is widely used in production, but "beta" is not a formality you should ignore. [§9.5](#95--choosing-an-auth-library) covers when to pick something else.
> - **Drizzle's official get-started page now instructs `drizzle-orm@rc`** (the 1.0 release candidate, `1.0.0-rc.4`). The stable `latest` is **0.45.2**, which is what this doc is written against. Check which line you are on before copying config from the docs site.
>
> Everything Next.js-specific here — the connection singleton, the DAL pattern, `proxy.ts`, where checks belong — is stack-independent and applies equally to Prisma, Supabase, or raw SQL.

---

## Table of Contents

- [Part 8: The Database Layer](#part-8-the-database-layer)
  - [8.1 Beginner — schema, client, first query](#81-beginner--schema-client-first-query)
  - [8.2 Working Knowledge — migrations and query patterns](#82-working-knowledge--migrations-and-query-patterns)
  - [8.3 Advanced — the connection singleton](#83-advanced--the-connection-singleton)
  - [8.4 Mastery — pooling, transactions, and the build step](#84-mastery--pooling-transactions-and-the-build-step)
  - [8.5 Cheat sheet: Part 8](#85-cheat-sheet-part-8)
- [Part 9: Authentication and Sessions](#part-9-authentication-and-sessions)
  - [9.1 Beginner — what the three words mean](#91-beginner--what-the-three-words-mean)
  - [9.2 Working Knowledge — Auth.js v5 in a Next.js 16 app](#92-working-knowledge--authjs-v5-in-a-nextjs-16-app)
  - [9.3 Advanced — the Data Access Layer](#93-advanced--the-data-access-layer)
  - [9.4 Mastery — proxy.ts, layouts, and where checks actually belong](#94-mastery--proxyts-layouts-and-where-checks-actually-belong)
  - [9.5 Choosing an auth library](#95--choosing-an-auth-library)
  - [9.6 Cheat sheet: Part 9](#96-cheat-sheet-part-9)

---

## Part 8: The Database Layer

<a id="part-8-the-database-layer"></a>

### 8.1 Beginner — schema, client, first query

```bash
npm i drizzle-orm pg
npm i -D drizzle-kit tsx @types/pg
```

```bash
# .env.local
DATABASE_URL="postgresql://postgres:dev@localhost:5432/postgres"
```

**The schema** is TypeScript, and it is the single source of truth — both your types and your migrations derive from it:

```ts
// lib/db/schema.ts
import { pgTable, text, timestamp, boolean, uuid, index } from 'drizzle-orm/pg-core'
import { relations } from 'drizzle-orm'

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: text('email').notNull().unique(),
  name: text('name'),
  passwordHash: text('password_hash'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
})

export const posts = pgTable(
  'posts',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    title: text('title').notNull(),
    content: text('content').notNull(),
    published: boolean('published').notNull().default(false),
    authorId: uuid('author_id')
      .notNull()
      .references(() => users.id, { onDelete: 'cascade' }),
    createdAt: timestamp('created_at').notNull().defaultNow(),
  },
  (t) => [index('posts_author_idx').on(t.authorId)]
)

export const usersRelations = relations(users, ({ many }) => ({ posts: many(posts) }))
export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, { fields: [posts.authorId], references: [users.id] }),
}))

export type User = typeof users.$inferSelect
export type NewPost = typeof posts.$inferInsert
```

**The config** for `drizzle-kit`:

```ts
// drizzle.config.ts
import 'dotenv/config'
import { defineConfig } from 'drizzle-kit'

export default defineConfig({
  schema: './lib/db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: { url: process.env.DATABASE_URL! },
})
```

> `drizzle-kit` runs outside the Next.js runtime, so it does **not** see `.env.local` automatically. Either use `dotenv/config` as above, or load Next's env files with `@next/env` so both read the same values ([Environment variables](https://nextjs.org/docs/app/guides/environment-variables#loading-environment-variables-with-nextenv)):
>
> ```ts
> import { loadEnvConfig } from '@next/env'
> loadEnvConfig(process.cwd())
> ```

**The client** — the naive version, which you will replace in [§8.3](#83-advanced--the-connection-singleton):

```ts
// lib/db/index.ts
import 'server-only'
import { drizzle } from 'drizzle-orm/node-postgres'
import * as schema from './schema'

export const db = drizzle(process.env.DATABASE_URL!, { schema })
export * from './schema'
export { eq, and, or, desc, asc, sql } from 'drizzle-orm'
```

That `import 'server-only'` on line one is not optional. It turns any accidental client-side import of this module into a **build error** instead of a runtime surprise ([Preventing environment poisoning](https://nextjs.org/docs/app/getting-started/server-and-client-components#preventing-environment-poisoning)).

**The first query**, straight in a Server Component:

```tsx
// app/posts/page.tsx
import { db, posts, desc } from '@/lib/db'

export default async function PostsPage() {
  const all = await db.select().from(posts).orderBy(desc(posts.createdAt)).limit(20)
  return <ul>{all.map((p) => <li key={p.id}>{p.title}</li>)}</ul>
}
```

No API route, no serialisation layer, no `useEffect`. This is the whole point of Server Components ([Fetching data — with an ORM or database](https://nextjs.org/docs/app/getting-started/fetching-data#with-an-orm-or-database)).

### 8.2 Working Knowledge — migrations and query patterns

**Two migration workflows**, and the choice matters ([Drizzle get started](https://orm.drizzle.team/docs/get-started/postgresql-new)):

```bash
npx drizzle-kit push       # diff schema → apply directly. Local dev only.
npx drizzle-kit generate   # write a versioned SQL file into ./drizzle
npx drizzle-kit migrate    # apply pending files. This is what CI/prod runs.
npx drizzle-kit studio     # browse the database in a local GUI
```

`push` is for iterating on a local database you don't mind losing. `generate` + `migrate` is for anything with data in it: the SQL is committed, reviewable, and replayable.

**Wrong vs. right — where migrations run:**

```dockerfile
# ✗ Running migrations at container start means N replicas race to migrate,
#   and a bad migration takes down every instance simultaneously.
CMD ["sh", "-c", "npx drizzle-kit migrate && node server.js"]

# ✓ Migrate once, as a deploy step, before rolling out new containers.
CMD ["node", "server.js"]
```

Same reasoning applies to `instrumentation.ts` — `register()` runs once **per server instance**, not once per deployment. It is the right place for a tracing SDK; the wrong place for a migration.

**Query patterns** you'll use constantly:

```ts
import { db, posts, users, eq, and, desc, sql } from '@/lib/db'

// Relational query API — reads well, handles joins for you
const post = await db.query.posts.findFirst({
  where: eq(posts.id, id),
  with: { author: { columns: { id: true, name: true } } },  // never select the hash
})

// Filtered list
const published = await db.query.posts.findMany({
  where: and(eq(posts.authorId, userId), eq(posts.published, true)),
  orderBy: [desc(posts.createdAt)],
  limit: 20,
})

// Insert returning
const [created] = await db.insert(posts).values(newPost).returning()

// Update scoped by ownership — see Part 6 §6.4
await db.update(posts)
  .set({ published: true })
  .where(and(eq(posts.id, id), eq(posts.authorId, session.userId)))

// Aggregate
const [{ count }] = await db
  .select({ count: sql<number>`count(*)::int` })
  .from(posts)
  .where(eq(posts.authorId, userId))
```

Note the `columns:` selection on the join. Getting into the habit of naming columns rather than taking whole rows is the cheapest defence against leaking a password hash into an RSC payload.

**Composing with Next.js caching.** Drizzle queries are plain async functions, so they slot into both layers from [Part 5](./nextjs-data-caching.md#part-5-caching-and-revalidation):

```ts
// Per-request deduplication — many components, one query.
import { cache } from 'react'
export const getPost = cache(async (id: string) =>
  db.query.posts.findFirst({ where: eq(posts.id, id) })
)

// Cross-request persistence (default caching model).
import { unstable_cache } from 'next/cache'
export const getPublishedPosts = unstable_cache(
  async () => db.query.posts.findMany({ where: eq(posts.published, true) }),
  ['published-posts'],
  { tags: ['posts'], revalidate: 300 }
)

// Cross-request persistence (Cache Components model).
export async function getPublishedPostsCC() {
  'use cache'
  cacheLife('minutes')
  cacheTag('posts')
  return db.query.posts.findMany({ where: eq(posts.published, true) })
}
```

### 8.3 Advanced — the connection singleton

This is the section that saves you a confusing afternoon.

Next.js dev hot-reloads modules on every change. Each reload re-evaluates `lib/db/index.ts` and creates **a new connection pool**, without closing the old one. Twenty minutes of editing later, Postgres says `FATAL: sorry, too many clients already`. It is one of the most reported issues in the ecosystem — see [vercel/next.js #26427](https://github.com/vercel/next.js/discussions/26427), [#23867](https://github.com/vercel/next.js/discussions/23867), and the Prisma equivalent [prisma/prisma#1983](https://github.com/prisma/prisma/issues/1983).

**Wrong vs. right:**

```ts
// ✗ A fresh Pool on every hot reload. Dies after ~20 minutes of dev.
import { drizzle } from 'drizzle-orm/node-postgres'
export const db = drizzle(process.env.DATABASE_URL!, { schema })
```

```ts
// ✓ lib/db/index.ts — one pool, stashed on globalThis, which survives HMR.
import 'server-only'
import { Pool } from 'pg'
import { drizzle } from 'drizzle-orm/node-postgres'
import * as schema from './schema'

const globalForDb = globalThis as unknown as { pool?: Pool }

const pool =
  globalForDb.pool ??
  new Pool({
    connectionString: process.env.DATABASE_URL!,
    max: Number(process.env.DB_POOL_MAX ?? 10),
    idleTimeoutMillis: 30_000,
    connectionTimeoutMillis: 5_000,
  })

// Only cache in development. In production the module is evaluated once
// anyway, and keeping globals clean avoids surprises across instances.
if (process.env.NODE_ENV !== 'production') globalForDb.pool = pool

export const db = drizzle(pool, { schema })
export * from './schema'
export { eq, and, or, desc, asc, sql } from 'drizzle-orm'
```

The mechanism: `globalThis` persists across HMR module re-evaluation, so the second evaluation finds the existing pool instead of building another.

> **Real Scenario — "the database goes down every afternoon."**
> A two-person team reported that their local Postgres "crashed" most afternoons; restarting Docker fixed it. It only ever happened to whoever was doing frontend work.
> **Cause:** no singleton. Each save created a pool of 10. After ~10 saves they had exhausted Postgres's default 100 connections. It hit the frontend developer hardest because they saved most often — and never in CI, where the process starts once.
> **Diagnosis:** `SELECT count(*), state FROM pg_stat_activity GROUP BY state;` showed ~100 idle connections from the same application.
> **Fix:** the singleton above. Three lines.

### 8.4 Mastery — pooling, transactions, and the build step

**Sizing the pool.** `max` is per process, not per deployment. Three containers with `max: 20` is 60 connections, plus whatever migrations and your admin tooling hold. Postgres's default `max_connections` is 100, and each connection costs real memory.

```
   3 replicas × max:20  =  60
   + migration job      =  ~5
   + psql / Studio      =  ~5
   ────────────────────────────
   = 70 of your 100. One autoscale event away from an outage.
```

Either budget explicitly, or put **PgBouncer** (or your provider's pooler) in front and give each app instance a small `max`. If you use a transaction-mode pooler, disable prepared statements in the driver — they don't survive connection reuse.

**Transactions** for multi-statement invariants:

```ts
export async function transferOwnership(postId: string, toUserId: string, actorId: string) {
  return db.transaction(async (tx) => {
    const post = await tx.query.posts.findFirst({
      where: and(eq(posts.id, postId), eq(posts.authorId, actorId)),
    })
    if (!post) throw new Error('Not found or not yours')

    await tx.update(posts).set({ authorId: toUserId }).where(eq(posts.id, postId))
    await tx.insert(auditLog).values({ actorId, action: 'transfer', targetId: postId })
  })
}
```

Everything inside the callback uses `tx`, not `db` — using `db` by accident silently escapes the transaction, which is the classic bug here.

**The build-time trap.** `next build` *executes your code*: `generateStaticParams`, prerendering, and metadata generation all run against whatever `DATABASE_URL` is set at build time. Consequences:

- Building in Docker without a reachable database fails at the prerender step, not at compile.
- A build that prerenders 5,000 pages opens a pool and hammers the database — see [vercel/next.js #71439](https://github.com/vercel/next.js/discussions/71439).

Three ways out, in preference order: prerender fewer pages (`generateStaticParams` returning the top N, letting the rest render on demand); keep a small dedicated build-time pool; or make the routes dynamic and cache them at request time instead.

**Runtime vs. build-time env.** `process.env.DATABASE_URL` read at module scope is evaluated whenever the module is first evaluated — which during a build may be the build environment. If you promote one Docker image across environments, force a runtime read ([Self-hosting — environment variables](https://nextjs.org/docs/app/guides/self-hosting#environment-variables)):

```ts
import { connection } from 'next/server'

export default async function Page() {
  await connection()                     // defers to request time
  const value = process.env.MY_VALUE     // now read at runtime
}
```

More on this in [Part 12](./nextjs-production-deployment.md#part-12-production-concerns).

### 8.5 Cheat sheet: Part 8

| Command | Purpose |
|---|---|
| `drizzle-kit push` | Sync schema directly — dev only |
| `drizzle-kit generate` | Write a versioned migration file |
| `drizzle-kit migrate` | Apply pending migrations — CI/deploy step |
| `drizzle-kit studio` | Local database GUI |

| Pattern | Code |
|---|---|
| Row types | `typeof users.$inferSelect` / `$inferInsert` |
| Find one, with relation | `db.query.posts.findFirst({ where, with: { author: { columns } } })` |
| Insert returning | `db.insert(t).values(v).returning()` |
| Ownership-scoped update | `.where(and(eq(t.id, id), eq(t.ownerId, userId)))` |
| Transaction | `db.transaction(async (tx) => { … })` — use `tx`, never `db` |
| Dedupe per request | `cache()` from `react` |
| Persist across requests | `unstable_cache` or `'use cache'` |

| Rule | Why |
|---|---|
| `import 'server-only'` in `lib/db` | Client import becomes a build error |
| Singleton the pool on `globalThis` | HMR otherwise leaks a pool per save |
| Migrate as a deploy step, not at boot | Replicas would race |
| Select columns, don't take whole rows | Password hashes end up in RSC payloads |
| `max` is per process | Multiply by replica count before sizing |

[↑ Back to top](#table-of-contents)

---

## Part 9: Authentication and Sessions

<a id="part-9-authentication-and-sessions"></a>

### 9.1 Beginner — what the three words mean

Three separate problems that get conflated ([Authentication](https://nextjs.org/docs/app/guides/authentication)):

| Concept | Question | Where it lives |
|---|---|---|
| **Authentication** | Are you who you claim to be? | Login form → Server Action |
| **Session management** | Are you still the same person on this next request? | A cookie, and maybe a `sessions` table |
| **Authorisation** | Are you allowed to do *this*? | The Data Access Layer, every Action, every Handler |

Two session strategies:

- **Stateless** — signed/encrypted payload in the cookie. Simple, no lookup per request, but you cannot revoke a session before it expires.
- **Database** — the cookie holds only an encrypted session ID; the row is the truth. Revocable, supports "log out all devices", costs a query.

Most apps want database sessions with the *optimistic* check reading the cookie alone — which is exactly what [§9.4](#94-mastery--proxyts-layouts-and-where-checks-actually-belong) describes.

### 9.2 Working Knowledge — Auth.js v5 in a Next.js 16 app

```bash
npm i next-auth@beta @auth/drizzle-adapter
npx auth secret      # generates AUTH_SECRET into .env.local
```

**The config file**, at the project root:

```ts
// auth.ts
import NextAuth from 'next-auth'
import GitHub from 'next-auth/providers/github'
import { DrizzleAdapter } from '@auth/drizzle-adapter'
import { db } from '@/lib/db'

export const { handlers, signIn, signOut, auth } = NextAuth({
  adapter: DrizzleAdapter(db),
  providers: [GitHub],
  session: { strategy: 'database' },
  callbacks: {
    session({ session, user }) {
      session.user.id = user.id
      return session
    },
  },
})
```

**The route handler:**

```ts
// app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/auth'
export const { GET, POST } = handlers
```

**And the piece every Auth.js tutorial gets wrong on Next.js 16:**

```ts
// ✗ middleware.ts — deprecated in 16. This is what authjs.dev's older
//   pages and every pre-2026 tutorial still show.
export { auth as middleware } from '@/auth'

// ✓ proxy.ts — the Next.js 16 filename and export name.
export { auth as proxy } from '@/auth'
```

`middleware.ts` still *works* in 16 (kept for Edge-runtime cases) but is deprecated and slated for removal ([Upgrade guide](https://nextjs.org/docs/app/guides/upgrading/version-16#middleware-to-proxy)). Auth.js's own installation page now documents the `proxy.ts` form for Next.js 16+, but plenty of its deeper pages and the entire third-party tutorial corpus still say `middleware`. Auth.js v5's stated minimum is Next.js 14, so v5 and Next 16 are compatible — the lag is documentation, not code.

**Reading the session.** In a Server Component, Server Action or Route Handler:

```tsx
import { auth } from '@/auth'

export default async function ProfilePage() {
  const session = await auth()
  if (!session?.user) redirect('/login')
  return <h1>Hello, {session.user.name}</h1>
}
```

In a Client Component, don't. Pass what's needed as props from a Server Component, or use a context provider seeded on the server. The DAL cannot be imported into client code, by design ([Auth and streaming](https://nextjs.org/docs/app/guides/authentication#auth-and-streaming)).

**Sign in and out** are Server Actions:

```tsx
import { signIn, signOut } from '@/auth'

export function SignIn() {
  return (
    <form action={async () => { 'use server'; await signIn('github') }}>
      <button>Sign in with GitHub</button>
    </form>
  )
}

export function SignOut() {
  return (
    <form action={async () => { 'use server'; await signOut() }}>
      <button>Sign out</button>
    </form>
  )
}
```

**Rolling your own sessions.** If you'd rather not take the beta dependency, the Next.js docs' own pattern is a signed JWT in an HttpOnly cookie using [`jose`](https://github.com/panva/jose) ([Stateless sessions](https://nextjs.org/docs/app/guides/authentication#stateless-sessions)). The cookie options are the part to get exactly right:

```ts
// lib/session.ts
import 'server-only'
import { cookies } from 'next/headers'
import { SignJWT, jwtVerify } from 'jose'

const key = new TextEncoder().encode(process.env.SESSION_SECRET)

export async function encrypt(payload: { userId: string; expiresAt: Date }) {
  return new SignJWT(payload)
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('7d')
    .sign(key)
}

export async function decrypt(session = '') {
  try {
    const { payload } = await jwtVerify(session, key, { algorithms: ['HS256'] })
    return payload
  } catch {
    return null            // never throw here — proxy.ts calls this on every request
  }
}

export async function createSession(userId: string) {
  const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
  const session = await encrypt({ userId, expiresAt })

  ;(await cookies()).set('session', session, {
    httpOnly: true,     // JS cannot read it → XSS can't steal it
    secure: true,       // HTTPS only
    sameSite: 'lax',    // CSRF mitigation
    expires: expiresAt,
    path: '/',
  })
}

export async function deleteSession() {
  ;(await cookies()).delete('session')
}
```

Generate the secret with `openssl rand -base64 32`. Keep the payload minimal — a user ID and an expiry. Never a role you won't re-check, never an email, never anything you'd mind a user decoding, because a signed JWT is *readable*, just not *forgeable*.

### 9.3 Advanced — the Data Access Layer

The DAL is the single most valuable pattern in this Part. The idea: **authorisation lives next to data access, not next to UI**, so it cannot be forgotten ([Creating a Data Access Layer](https://nextjs.org/docs/app/guides/authentication#creating-a-data-access-layer-dal)).

```ts
// lib/dal.ts
import 'server-only'
import { cache } from 'react'
import { cookies } from 'next/headers'
import { redirect } from 'next/navigation'
import { decrypt } from '@/lib/session'
import { db, users, posts, eq, and } from '@/lib/db'

export const verifySession = cache(async () => {
  const cookie = (await cookies()).get('session')?.value
  const session = await decrypt(cookie)

  if (!session?.userId) redirect('/login')
  return { isAuth: true as const, userId: String(session.userId) }
})

export const getUser = cache(async () => {
  const session = await verifySession()
  try {
    return await db.query.users.findFirst({
      where: eq(users.id, session.userId),
      columns: { id: true, name: true, email: true },   // DTO at the query level
    })
  } catch {
    return null
  }
})

// Every read is scoped by the session, not by a caller-supplied id.
export const getMyPosts = cache(async () => {
  const session = await verifySession()
  return db.query.posts.findMany({ where: eq(posts.authorId, session.userId) })
})

export const getMyPost = cache(async (id: string) => {
  const session = await verifySession()
  return db.query.posts.findFirst({
    where: and(eq(posts.id, id), eq(posts.authorId, session.userId)),
  })
})
```

Four deliberate choices in there:

1. **`import 'server-only'`** — a client import is a build error.
2. **`cache()` from React** — `verifySession()` can be called by a layout, a page and four leaf components; the cookie is decrypted (or the session row fetched) once per request.
3. **`redirect` inside `verifySession`** — callers cannot forget to handle the unauthenticated case.
4. **Ownership is in the `where` clause** — `getMyPost(id)` returns `undefined` for someone else's post. There is no code path that returns a row you don't own.

**Data Transfer Objects.** Return what the UI renders, never whole records ([Using DTOs](https://nextjs.org/docs/app/guides/authentication#using-data-transfer-objects-dto)). This matters more in the App Router than in a traditional SPA, because props passed to Client Components are serialised into the RSC payload, which is plain text in the network tab. `<UserMenu user={rawUserRow} />` publishes the password hash.

```ts
// lib/dto.ts
import 'server-only'

export async function getProfileDTO(slug: string) {
  const viewer = await getUser()
  const user = await db.query.users.findFirst({ where: eq(users.slug, slug) })
  if (!user) return null

  return {
    username: user.username,
    phone: viewer?.isAdmin || viewer?.team === user.team ? user.phone : null,
  }
}
```

For extra insurance on genuinely sensitive fields, React's [`taintUniqueValue`](https://react.dev/reference/react/experimental_taintUniqueValue) makes passing a specific value to the client a runtime error.

### 9.4 Mastery — proxy.ts, layouts, and where checks actually belong

**The layered model.** Three places a check can live, and each has a different job:

```
  ┌─────────────────────────────────────────────────────────────┐
  │  proxy.ts        OPTIMISTIC only — read the cookie, redirect │
  │                  Runs on EVERY request, including prefetches │
  │                  ✗ No database. ✗ Not a security boundary.   │
  ├─────────────────────────────────────────────────────────────┤
  │  page / layout   UX shaping — which dashboard, which buttons │
  │                  ✗ A layout hiding a route does NOT stop it  │
  ├─────────────────────────────────────────────────────────────┤
  │  DAL / Action    THE boundary. Session verified, ownership   │
  │  / Handler       enforced, at the point of data access.      │
  └─────────────────────────────────────────────────────────────┘
```

**`proxy.ts` — optimistic checks only.**

```ts
// proxy.ts
import { NextResponse, type NextRequest } from 'next/server'
import { decrypt } from '@/lib/session'

const protectedRoutes = ['/dashboard', '/settings']
const publicRoutes = ['/login', '/signup', '/']

export default async function proxy(req: NextRequest) {
  const path = req.nextUrl.pathname
  const isProtected = protectedRoutes.some((r) => path.startsWith(r))
  const isPublic = publicRoutes.includes(path)

  // Cookie only. No DB. This runs on every prefetch too.
  const session = await decrypt(req.cookies.get('session')?.value)

  if (isProtected && !session?.userId) {
    return NextResponse.redirect(new URL('/login', req.nextUrl))
  }
  if (isPublic && session?.userId && !path.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/dashboard', req.nextUrl))
  }
  return NextResponse.next()
}

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|.*\\.png$).*)'],
}
```

The constraint the docs are emphatic about: *"since Proxy runs on every route, including prefetched routes, it's important to only read the session from the cookie (optimistic checks), and avoid database checks to prevent performance issues"* ([Optimistic checks with Proxy](https://nextjs.org/docs/app/guides/authentication#optimistic-checks-with-proxy-optional)). And: *"Proxy should not be used as a full session management or authorization solution"* ([Proxy](https://nextjs.org/docs/app/getting-started/proxy#use-cases)).

Two Next.js 16 specifics: `proxy.ts` runs on the **Node.js runtime, and that is not configurable** — good news, since your session library probably wanted Node crypto. And `fetch` cache options (`options.cache`, `next.revalidate`, `next.tags`) have **no effect** inside proxy.

Only one `proxy.ts` per project is supported, but you can split the logic into modules and import them.

**Why a layout check is not protection.** This is subtle and worth reading twice:

> Due to Partial Rendering, be cautious when doing checks in Layouts as these don't re-render on navigation, meaning the user session won't be checked on every route change. […] A layout also does not control whether the rest of the route renders. Route segments and parallel route slots are rendered by the router, so a layout that hides or swaps them does not stop them from running or from appearing in the RSC Payload. — [Layouts and auth checks](https://nextjs.org/docs/app/guides/authentication#layouts-and-auth-checks)

```tsx
// ✗ Two separate failures: the layout doesn't re-run on client navigation,
//   AND the child page renders regardless — its data is in the RSC payload.
export default async function Layout({ children }: { children: React.ReactNode }) {
  const session = await auth()
  if (!session) return <p>Not authorised</p>
  return <>{children}</>
}

// ✓ Every read goes through the DAL. The check is unforgettable because
//   there is no way to get the data without it.
export default async function Page() {
  const posts = await getMyPosts()   // verifySession() inside
  return <PostList posts={posts} />
}
```

**Auth in the shell, without blocking every page.** A user menu needs the session on every route. Awaiting it at the top of the root layout delays the first streamed chunk of the entire app ([Auth and streaming](https://nextjs.org/docs/app/guides/authentication#auth-and-streaming)):

```tsx
// ✗ Every page in the app now waits for the session query before streaming.
export default async function Layout({ children }: { children: React.ReactNode }) {
  const user = await getUser()
  return <><Header user={user} />{children}</>
}

// ✓ The shell streams; only the user chip waits.
import { Suspense } from 'react'

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <>
      <header>
        <Logo />
        <Suspense fallback={<AvatarSkeleton />}><UserMenu /></Suspense>
      </header>
      {children}
    </>
  )
}

async function UserMenu() {
  const user = await getUser()
  return user ? <Avatar name={user.name} /> : <SignInLink />
}
```

> **Real Scenario — the paywall that leaked every article.**
> A publisher gated premium articles with a check in `app/(premium)/layout.tsx`: no subscription, render an upsell instead of `{children}`. Visually perfect. Then someone posted the full text of a paywalled article on a forum, with a screenshot of the browser's network tab.
> **Cause:** the layout controlled what was *displayed*, not what was *rendered*. The page still ran, still queried the article body, and the body still went into the RSC payload — which anyone can read in DevTools or `curl`. The upsell was a `display: none` for grown-ups.
> **Fix:** `getArticle(slug)` moved into the DAL, where it verifies the session's subscription status and returns the body only if entitled — otherwise an excerpt. `proxy.ts` kept its optimistic redirect for the fast path, and the layout kept rendering the upsell, but neither was load-bearing any more.
> **The rule:** if the data reached the render, it reached the user. Check at the source.

### 9.5 — Choosing an auth library

<a id="95--choosing-an-auth-library"></a>

This set commits to Auth.js because you asked for it, and it is a defensible choice. But you should make it with open eyes.

| Option | Case for | Case against |
|---|---|---|
| **Auth.js v5** (`next-auth@beta`) | Free, self-hosted, huge provider list, Drizzle adapter, the default in most tutorials | Still `@beta` after two years; docs lag Next.js 16 (`middleware` vs `proxy`); v4 → v5 breaking changes are real |
| **Better Auth** | Stable releases, TypeScript-first, Drizzle-native, strong plugin story (2FA, orgs, passkeys) | Younger; smaller community; fewer tutorials to lean on |
| **Clerk / WorkOS / Auth0** | Managed — you stop owning session security, MFA, account recovery | Per-MAU cost; vendor lock-in; user data leaves your database |
| **Supabase Auth** | Sensible if Supabase is already your database; RLS ties auth to data cleanly | Couples auth to that specific database |
| **Roll your own** (jose + cookies) | Total control; the Next.js docs walk it end to end; no dependency risk | You now own password resets, rate limiting, MFA, and every CVE |

Next.js's own [list of auth libraries](https://nextjs.org/docs/app/guides/authentication#auth-libraries) is longer. **Everything else in this Part — the DAL, DTOs, `proxy.ts`, where checks belong — is identical whichever you pick.** That is deliberate: the library handles authentication and session storage; authorisation is always yours.

### 9.6 Cheat sheet: Part 9

| Need | Code |
|---|---|
| Read session (server) | `await auth()` — or `await verifySession()` via the DAL |
| Protect a read | Put `verifySession()` inside the DAL function |
| Protect an Action/Handler | First line: `const session = await verifySession()` |
| Optimistic redirect | `proxy.ts` — cookie only, never the database |
| Set a session cookie | `cookies().set(name, v, { httpOnly, secure, sameSite:'lax', expires, path })` |
| Dedupe session reads | Wrap in `cache()` from `react` |
| Session in the header without blocking | Nested Server Component + `<Suspense>` |
| Client needs user data | Pass props from a Server Component |

| Cookie option | Value | Why |
|---|---|---|
| `httpOnly` | `true` | XSS cannot read it |
| `secure` | `true` | HTTPS only |
| `sameSite` | `'lax'` | CSRF mitigation |
| `expires` / `maxAge` | set explicitly | Sessions must end |
| `path` | `'/'` | Sent on every route |

| Anti-pattern | Do instead |
|---|---|
| Auth check only in `layout.tsx` | Check in the DAL at the point of data access |
| Database lookup in `proxy.ts` | Cookie only; verify in the DAL |
| `return null` from a layout as protection | The route still renders into the RSC payload |
| Passing whole user rows to Client Components | DTO with named columns |
| Trusting an ID from a hidden input or action arg | Re-look-up scoped by `session.userId` |
| Role stored in a JWT and never re-checked | Re-check on every privileged operation |

[↑ Back to top](#table-of-contents)

---

**Next:** [Navigation & Assets (Parts 10–11)](./nextjs-navigation-assets.md) — fast transitions, metadata, images and fonts.
