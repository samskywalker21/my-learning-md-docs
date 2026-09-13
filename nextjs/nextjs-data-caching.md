# Next.js — Data & Caching (Parts 4–5)

Reading data in Server Components, streaming it with Suspense, and controlling what gets cached — in both of Next.js 16's caching models.

> **Spec:** this doc follows the shared spec in [`nextjs-mastery-guide.md`](./nextjs-mastery-guide.md#about-this-document). Written against **Next.js 16.3.4**, verified **September 11, 2026**.

Part 5 is the highest-churn material in the framework. If you read one thing carefully, make it [§5.1](#51-beginner--the-one-sentence-that-changed) — the default that flipped between Next.js 14 and 15, and which still poisons most tutorials.

---

## Table of Contents

- [Part 4: Data Fetching and Streaming](#part-4-data-fetching-and-streaming)
  - [4.1 Beginner — `await` is the API](#41-beginner--await-is-the-api)
  - [4.2 Working Knowledge — streaming with `loading.tsx` and `<Suspense>`](#42-working-knowledge--streaming-with-loadingtsx-and-suspense)
  - [4.3 Advanced — waterfalls, parallelism, deduplication, preloading](#43-advanced--waterfalls-parallelism-deduplication-preloading)
  - [4.4 Mastery — pushing async access down the tree](#44-mastery--pushing-async-access-down-the-tree)
  - [4.5 Cheat sheet: Part 4](#45-cheat-sheet-part-4)
- [Part 5: Caching and Revalidation](#part-5-caching-and-revalidation)
  - [5.1 Beginner — the one sentence that changed](#51-beginner--the-one-sentence-that-changed)
  - [5.2 Working Knowledge — the default model](#52-working-knowledge--the-default-model)
  - [5.3 Advanced — on-demand revalidation and ISR](#53-advanced--on-demand-revalidation-and-isr)
  - [5.4 Mastery — Cache Components and Partial Prerendering](#54-mastery--cache-components-and-partial-prerendering)
  - [5.5 Cheat sheet: Part 5](#55-cheat-sheet-part-5)

---

## Part 4: Data Fetching and Streaming

<a id="part-4-data-fetching-and-streaming"></a>

### 4.1 Beginner — `await` is the API

There is no `getServerSideProps`, no data-fetching hook, no loader function. A Server Component is `async`, and you `await` ([Fetching data](https://nextjs.org/docs/app/getting-started/fetching-data)):

```tsx
// app/blog/page.tsx — with fetch
export default async function Page() {
  const res = await fetch('https://api.vercel.app/blog')
  const posts = await res.json()
  return <ul>{posts.map((p) => <li key={p.id}>{p.title}</li>)}</ul>
}
```

```tsx
// app/blog/page.tsx — with an ORM, which is what you'll actually do
import { db, posts } from '@/lib/db'

export default async function Page() {
  const allPosts = await db.select().from(posts)
  return <ul>{allPosts.map((p) => <li key={p.id}>{p.title}</li>)}</ul>
}
```

Both are equally first-class. Because the component runs only on the server, the connection string and query logic never reach the client bundle ([Fetching data — with an ORM or database](https://nextjs.org/docs/app/getting-started/fetching-data#with-an-orm-or-database)).

Fetching in a **Client** Component is a different job and a worse default. Two supported ways ([Fetching data — Client Components](https://nextjs.org/docs/app/getting-started/fetching-data#client-components)):

1. React's `use()` on a promise passed down from a Server Component (shown in [Part 3](./nextjs-routing-rendering.md#33-advanced--composition-patterns-that-keep-the-boundary-small));
2. a client library — SWR or TanStack Query — for genuinely client-driven data (polling, infinite scroll, optimistic lists).

If you are reaching for SWR to load a page's initial data, you have probably put the boundary in the wrong place.

### 4.2 Working Knowledge — streaming with `loading.tsx` and `<Suspense>`

Awaiting on the server blocks the whole route until the slowest query finishes. Streaming breaks the page into chunks that arrive as they're ready.

```
   WITHOUT STREAMING                  WITH STREAMING
   ─────────────────                  ──────────────
   [ 900ms of nothing ]               [ 40ms ] shell + skeletons
            │                                  │
            ▼                                  ├── 120ms  header data
   [ entire page at once ]                     ├── 300ms  main list
                                               └── 900ms  slow sidebar widget
```

**Two mechanisms.** `loading.tsx` streams the whole page:

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return <DashboardSkeleton />
}
```

Behind the scenes Next.js nests this inside `layout.tsx` and wraps `page.tsx` in a `<Suspense>` boundary ([Fetching data — with `loading.js`](https://nextjs.org/docs/app/getting-started/fetching-data#with-loadingjs)).

`<Suspense>` gives you per-component granularity:

```tsx
import { Suspense } from 'react'

export default function BlogPage() {
  return (
    <div>
      {/* Ships immediately — no async work above it */}
      <header>
        <h1>Welcome to the Blog</h1>
      </header>
      <main>
        <Suspense fallback={<BlogListSkeleton />}>
          <BlogList />
        </Suspense>
      </main>
    </div>
  )
}

async function BlogList() {
  const posts = await getPosts()
  return <ul>{posts.map((p) => <li key={p.id}>{p.title}</li>)}</ul>
}
```

**The gotcha that costs people an afternoon:** `loading.tsx` does not cover a layout's own async work.

> Because of this, a layout that accesses uncached or runtime data (e.g. `cookies()`, `headers()`, or uncached fetches) does not fall back to a same-route-segment `loading.js`. Instead, it **blocks navigation** until the layout finishes rendering. — [Fetching data](https://nextjs.org/docs/app/getting-started/fetching-data#with-loadingjs)

```tsx
// ✗ loading.tsx never appears. Navigation freezes for the length of this await.
export default async function Layout({ children }: { children: React.ReactNode }) {
  const user = await getCurrentUser() // 250ms
  return (
    <div>
      <Nav user={user} />
      {children}
    </div>
  )
}

// ✓ The await moves into its own Suspense boundary. The shell and
//   {children} stream immediately; only the nav's user chip waits.
import { Suspense } from 'react'

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <div>
      <Suspense fallback={<NavSkeleton />}>
        <NavWithUser />
      </Suspense>
      {children}
    </div>
  )
}

async function NavWithUser() {
  const user = await getCurrentUser()
  return <Nav user={user} />
}
```

This is why the official guidance is to prefer `<Suspense>` close to the data access over relying on `loading.js` for route segments.

**Bots get different treatment.** Next.js detects crawlers by user agent and renders the whole page before responding, rather than streaming — they need a complete document ([Fetching data](https://nextjs.org/docs/app/getting-started/fetching-data#streaming)). Worth knowing when your Lighthouse numbers and your real-user numbers disagree.

### 4.3 Advanced — waterfalls, parallelism, deduplication, preloading

**Layouts and pages render in parallel** by default, so each segment starts fetching as soon as possible. Within a single component, though, sequential `await`s are sequential:

```tsx
// ✗ getAlbums waits for getArtist even though it doesn't need the result.
const artist = await getArtist(username)
const albums = await getAlbums(username)

// ✓ Both requests start immediately; you wait once for the slower.
const artistData = getArtist(username)
const albumsData = getAlbums(username)
const [artist, albums] = await Promise.all([artistData, albumsData])
```

Source: [Parallel data fetching](https://nextjs.org/docs/app/getting-started/fetching-data#parallel-data-fetching). Note the caveat: `Promise.all` rejects wholesale if one request fails — use `Promise.allSettled` when partial data is acceptable.

**Genuine dependencies are fine — just don't block the page on them.** When `<Playlists>` truly needs `artist.id`, put it behind a boundary so the artist renders first and the playlists stream in ([Sequential data fetching](https://nextjs.org/docs/app/getting-started/fetching-data#sequential-data-fetching)).

**Deduplication.** Identical `fetch` requests in one render pass are memoized automatically. ORM calls are not — wrap them in `React.cache`:

```ts
// lib/data.ts
import { cache } from 'react'
import { db, eq, users } from '@/lib/db'

export const getUser = cache(async (id: string) => {
  return db.query.users.findFirst({ where: eq(users.id, id) })
})
```

Now a layout, a page and three leaf components can each call `getUser(id)` and the query runs once ([Reusing data with `React.cache`](https://nextjs.org/docs/app/getting-started/fetching-data#reusing-data-with-reactcache)).

`React.cache` is **scoped to a single request**. It is a deduplication tool, not a cache — nothing is shared between requests. That distinction is the whole of Part 5.

This also solves the `generateMetadata` duplication problem: `generateMetadata` and the page component both need the post, and with `cache` they share one query ([Memoizing data requests](https://nextjs.org/docs/app/getting-started/metadata-and-og-images#memoizing-data-requests)).

**Preloading.** When a component renders after other blocking work, its request starts late. Kick it off early:

```ts
// app/item/[id]/item.tsx
async function getItem(id: string) {
  const res = await fetch(`https://api.example.com/items/${id}`)
  return res.json()
}

export const preload = (id: string) => {
  void getItem(id) // fire, don't await
}

export default async function Item({ id }: { id: string }) {
  const item = await getItem(id)
  return <div>{item.name}</div>
}
```

```tsx
// app/item/[id]/page.tsx
import Item, { preload } from './item'

export default async function Page(props: PageProps<'/item/[id]'>) {
  const { id } = await props.params

  preload(id)                                   // starts now
  const isAvailable = await checkIsAvailable(id) // runs in parallel

  return isAvailable ? <Item id={id} /> : null
}
```

The function must deduplicate for this to work — `fetch` does automatically, ORM calls need `React.cache` ([Preloading data](https://nextjs.org/docs/app/getting-started/fetching-data#preloading-data)).

> **Real Scenario — the 4.1-second dashboard.**
> A dashboard took 4.1s to first byte. Every widget was a tidy async Server Component, each fetching its own data. The team assumed Next.js parallelised them.
> **Cause:** the page component awaited a `getWorkspace()` call at the top to pass `workspaceId` into every widget. Each widget then awaited its own query — but all of them rendered *below* that first `await`, and with no `<Suspense>` boundaries the whole tree had to resolve before a byte went out. Six 600ms queries were effectively serialised behind one 400ms query.
> **Fix, in two steps.** First, each widget got its own `<Suspense>` boundary, so the page shell shipped at 40ms and widgets streamed in independently. Second, `getWorkspace` was wrapped in `React.cache` and called *inside* each widget rather than threaded through props, which removed the top-level `await` entirely. TTFB went to 60ms; the slowest widget still took 600ms, but the user saw the page immediately.

### 4.4 Mastery — pushing async access down the tree

The structural principle behind all of the above:

> The deeper your async work sits in the tree, the more of the page can be prerendered. — [Caching — maximizing the static shell](https://nextjs.org/docs/app/getting-started/caching#maximizing-the-static-shell)

A layout that destructures `params` at the top cannot be prerendered at all, because `params` is runtime data for any URL not listed in `generateStaticParams`. But you can pass the promise down:

```tsx
// ✗ The await is at the layout's top level. Nothing here prerenders.
export default async function Layout({ children, params }: LayoutProps<'/shop/[slug]'>) {
  const { slug } = await params
  return (
    <div>
      <Sidebar />
      <h1>{slug}</h1>
      {children}
    </div>
  )
}

// ✓ Not async. The await happens inside a boundary, so <Sidebar />,
//   {children} and the fallback are all part of the static shell.
import { Suspense } from 'react'

export default function Layout({ children, params }: LayoutProps<'/shop/[slug]'>) {
  return (
    <div>
      <Sidebar />
      <Suspense fallback={<h1>Loading…</h1>}>
        {params.then(({ slug }) => <SlugHeading slug={slug} />)}
      </Suspense>
      {children}
    </div>
  )
}

function SlugHeading({ slug }: { slug: string }) {
  return <h1>{slug}</h1>
}
```

The same principle applies to `cookies()`, `headers()`, `searchParams` and any data fetch. This pattern pays off most under Cache Components ([§5.4](#54-mastery--cache-components-and-partial-prerendering)), where the framework validates it at build time — but it is worth applying regardless, because it is what makes streaming actually stream.

**Auth data in the shell** is the case where this bites hardest. A user menu in the header needs the session on every route; a top-level `await cookies()` in the root layout delays the first streamed chunk of every page in the app. Move it into a nested Server Component behind `<Suspense>` ([Auth and streaming](https://nextjs.org/docs/app/guides/authentication#auth-and-streaming)). [Part 9](./nextjs-database-auth.md#part-9-authentication-and-sessions) has the full pattern.

### 4.5 Cheat sheet: Part 4

| Need | Tool |
|---|---|
| Load data for a page | `async` Server Component + `await` |
| Dedupe an ORM call across a render | `cache()` from `react` |
| Dedupe a `fetch` | automatic |
| Show a skeleton for a whole route | `loading.tsx` |
| Show a skeleton for one component | `<Suspense fallback={…}>` |
| Independent requests | `Promise.all` (or `allSettled`) |
| Start a request before blocking work | `preload()` pattern — `void getItem(id)` |
| Data in a Client Component, fetched on the server | pass a promise + `use()` |
| Genuinely client-driven data | SWR / TanStack Query |

| Symptom | Cause |
|---|---|
| `loading.tsx` never appears | The layout awaits something at its top level |
| Page TTFB = sum of all queries | Missing `<Suspense>`; everything behind one top-level `await` |
| Same query runs 5× per request | ORM call not wrapped in `React.cache` |
| Streaming works locally, not in prod | Proxy buffering — see [Part 13](./nextjs-production-deployment.md#part-13-deployment-and-internals) |

[↑ Back to top](#table-of-contents)

---

## Part 5: Caching and Revalidation

<a id="part-5-caching-and-revalidation"></a>

### 5.1 Beginner — the one sentence that changed

> `fetch` requests are **not cached by default** and will block the page from rendering until the request is complete. — [Fetching data](https://nextjs.org/docs/app/getting-started/fetching-data#with-the-fetch-api)

In Next.js **13 and 14**, `fetch` was cached aggressively by default. That default was reversed in 15 and remains reversed in 16. Every tutorial that says "remember to pass `cache: 'no-store'` or your data will be stale" is describing a framework that no longer exists.

The practical upshot for a new project: **you start uncached and opt in.** Which is the right way round — stale data is a bug, a slow page is a tuning problem.

Next.js 16 has two ways to opt in, and which one you have depends on a single config flag:

```ts
// next.config.ts
const nextConfig: NextConfig = {
  cacheComponents: false, // default in 16.3 → the "previous model" (§5.2, §5.3)
  // cacheComponents: true → Cache Components + PPR (§5.4)
}
```

This Part teaches the default model first, because that is what you get, then Cache Components as the opt-in forward path. **`use cache`, `cacheLife` and `cacheTag` do nothing unless `cacheComponents: true`.**

### 5.2 Working Knowledge — the default model

With `cacheComponents` off, there are four caching levers ([Caching without Cache Components](https://nextjs.org/docs/app/guides/caching-without-cache-components)).

**1. Per-`fetch` caching.**

```ts
// Cache indefinitely until revalidated
await fetch('https://api.example.com/config', { cache: 'force-cache' })

// Time-based: refresh at most once an hour
await fetch('https://api.example.com/posts', { next: { revalidate: 3600 } })

// Tagged, for on-demand invalidation
await fetch(`https://api.example.com/users/${id}`, { next: { tags: ['user', `user-${id}`] } })
```

**2. `unstable_cache` for anything that isn't `fetch`** — which in a real app means your database:

```ts
// lib/data.ts
import { unstable_cache } from 'next/cache'
import { db, eq, users } from '@/lib/db'

export const getCachedUser = unstable_cache(
  async (id: string) => db.select().from(users).where(eq(users.id, id)).then((r) => r[0]),
  ['user'],                              // cache key prefix
  { tags: ['user'], revalidate: 3600 }   // invalidation + TTL
)
```

Yes, it is still called `unstable_`. It has been stable in practice for years; the name signals that Cache Components is the intended replacement. Don't let the prefix scare you off — but don't be surprised when it's deprecated either.

**Wrong vs. right — the two `cache`s:**

```ts
// ✗ React's cache() does NOT persist anything between requests.
//    This runs the query on every single request.
import { cache } from 'react'
export const getSettings = cache(async () => db.select().from(settings))

// ✓ unstable_cache persists across requests; React's cache dedupes within one.
import { unstable_cache } from 'next/cache'
export const getSettings = unstable_cache(
  async () => db.select().from(settings),
  ['settings'],
  { tags: ['settings'], revalidate: 3600 }
)
```

Both are useful and they compose — `React.cache` for per-request deduplication ([Part 4](#43-advanced--waterfalls-parallelism-deduplication-preloading)), `unstable_cache` for cross-request persistence. Confusing them is one of the most common Next.js mistakes.

**3. Route segment config** — coarse, route-wide switches exported from a `page.tsx`, `layout.tsx` or `route.ts`:

```ts
export const dynamic = 'auto'        // 'auto' | 'force-dynamic' | 'error' | 'force-static'
export const revalidate = 3600       // false | 0 | seconds
export const fetchCache = 'auto'     // advanced; rarely needed
```

| Value | Meaning |
|---|---|
| `dynamic = 'auto'` | default — cache what can be cached |
| `dynamic = 'force-dynamic'` | render on every request; all fetches become `no-store` |
| `dynamic = 'force-static'` | prerender; `cookies()`, `headers()`, `useSearchParams()` return empty values |
| `dynamic = 'error'` | prerender, and **fail the build** if anything runtime is used |
| `revalidate = 0` | always dynamic |
| `revalidate = n` | route-level ISR period |

Source: [Route segment config](https://nextjs.org/docs/app/guides/caching-without-cache-components#route-segment-config). Two traps: `revalidate` must be statically analysable (`revalidate = 3600` works, `revalidate = 60 * 10` does not), and **the lowest `revalidate` across a route's layouts and pages wins for the whole route**.

`dynamic = 'error'` is underused and genuinely valuable: put it on a marketing page and the build fails the day someone adds a `cookies()` call that would have silently turned it dynamic.

**4. Static generation of dynamic routes** with `generateStaticParams`:

```tsx
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await db.select({ slug: posts.slug }).from(posts)
  return posts.map((p) => ({ slug: p.slug }))
}
```

Without it, `/blog/[slug]` renders at request time for every visit. With it, the listed slugs are prerendered at build ([Linking and navigating](https://nextjs.org/docs/app/getting-started/linking-and-navigating#dynamic-segments-without-generatestaticparams)) — and, importantly for [Part 10](./nextjs-navigation-assets.md#part-10-navigation-metadata-and-seo), they become fully prefetchable.

**In development, pages are always rendered on demand and never cached** ([Route segment config](https://nextjs.org/docs/app/guides/caching-without-cache-components#route-segment-config-revalidate)). Caching bugs are production-only by construction. Test them with `next build && next start`.

### 5.3 Advanced — on-demand revalidation and ISR

Time-based revalidation is a guess. On-demand revalidation is correct: you know when the data changed, because you just changed it.

```ts
// app/lib/actions.ts
'use server'

import { revalidatePath, revalidateTag, updateTag } from 'next/cache'

export async function publishPost(id: string) {
  await db.update(posts).set({ published: true }).where(eq(posts.id, id))

  updateTag(`post-${id}`)          // this user sees it immediately
  revalidateTag('post-list', 'max') // everyone else, in the background
  revalidatePath('/blog')           // and the blog index page
}
```

**The 16 signature change.** `revalidateTag` now takes a [`cacheLife` profile](https://nextjs.org/docs/app/api-reference/functions/cacheLife#reference) as a second argument:

```ts
// ✗ Deprecated single-argument form. Produces a TypeScript error in 16.
revalidateTag('posts')

// ✓ Built-in profile — 'max' is the recommended default.
revalidateTag('posts', 'max')
revalidateTag('news-feed', 'hours')

// ✓ Or an inline object.
revalidateTag('products', { expire: 3600 })
```

Source: [Caching APIs](https://nextjs.org/docs/app/guides/upgrading/version-16#revalidatetag).

**The three-way choice after a mutation** is the part people get wrong, so here it is as a decision table ([Choosing a cache update](https://nextjs.org/docs/app/guides/server-actions#choosing-a-cache-update)):

| API | Semantics | Re-renders the route in the same response? | Use when |
|---|---|---|---|
| `updateTag(tag)` | expire now, read fresh in this request | **yes** | The user just changed it and must see it — forms, settings, anything with a save button |
| `revalidateTag(tag, profile)` | stale-while-revalidate | **no** | Eventual consistency is fine — blog posts, catalogues, docs |
| `revalidatePath(path)` | invalidate one URL | yes | One route affected; tagging is overkill |
| `refresh()` | refetch uncached data only | yes | The view depends on state outside the cache (a live counter, a status badge) |

`updateTag` and `refresh` are **Server-Actions-only** APIs. All of them except `redirect` are non-throwing, so an action can call them and still return a value.

> **Real Scenario — "I saved it, but it's still showing the old title."**
> A team migrated to 16 and mechanically added `'max'` to every `revalidateTag` call to clear the deprecation warning. Editors then reported that saving a post showed the old title until they hard-refreshed — sometimes for a minute.
> **Cause:** `'max'` means stale-while-revalidate. The action returned immediately with the *stale* cached render while revalidation happened in the background. That's correct behaviour for readers and exactly wrong for the person who just hit Save.
> **Fix:** `updateTag(\`post-${id}\`)` for the editor's own view — read-your-writes, resolved inside the same roundtrip — and `revalidateTag('post-list', 'max')` for the public index, where a few seconds of staleness is free.
> **The lesson:** the second argument is not deprecation noise. It is a semantic choice about who is looking.

**ISR.** Incremental Static Regeneration is `generateStaticParams` + a revalidation period: pages are prerendered at build, served from cache, and rebuilt in the background when stale. Self-hosted, the cache lives on each instance's local disk by default — which is fine for one container and a correctness problem for three. That is [Part 13](./nextjs-production-deployment.md#part-13-deployment-and-internals).

**Where cached content actually lives** ([Caching — where cached content is stored](https://nextjs.org/docs/app/getting-started/caching#where-cached-content-is-stored)):

```
  ┌──────────────────────────────────────────────────────────────┐
  │ Prerendered HTML       on disk (self-host) / CDN storage      │
  │                        rebuilt per `revalidate` / `expire`    │
  ├──────────────────────────────────────────────────────────────┤
  │ Shared store           in-memory per instance by default      │
  │                        ← the one that bites you at 3 replicas │
  ├──────────────────────────────────────────────────────────────┤
  │ Browser                RSC payload from a navigation/prefetch │
  │                        fresh for the `stale` window           │
  └──────────────────────────────────────────────────────────────┘
```

All of these are scoped to a single deployment — the cache key includes the build ID, so **a new deploy starts cold**. Budget for that: a deploy at peak traffic means every page rebuilds at once.

### 5.4 Mastery — Cache Components and Partial Prerendering

Cache Components is the direction the framework is heading. It is opt-in in 16.3, and enabling it is a real migration, not a flag flip.

```ts
// next.config.ts
const nextConfig: NextConfig = { cacheComponents: true }
```

**What it changes.** Caching becomes entirely explicit via the `use cache` directive, and **Partial Prerendering becomes the default rendering model**: a route is no longer static *or* dynamic — it produces a static shell with dynamic holes that stream in ([Caching](https://nextjs.org/docs/app/getting-started/caching)).

```tsx
// Data-level: cache a function
import { cacheLife, cacheTag } from 'next/cache'

export async function getUsers() {
  'use cache'
  cacheLife('hours')
  cacheTag('users')
  return db.select().from(users)
}

// UI-level: cache a whole component or page
export default async function Page() {
  'use cache'
  cacheLife('hours')
  const users = await db.select().from(users)
  return <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>
}
```

Arguments and captured variables automatically become part of the cache key — the compiler generates it, so different inputs get different entries.

**The rule that reshapes your code.** Anything that *cannot* complete during prerendering must be either cached or wrapped in `<Suspense>`, and Next.js **fails the build** otherwise, naming the route and offering fixes. That includes:

- uncached async reads (`fetch`, database queries);
- runtime APIs — `cookies()`, `headers()`, `searchParams`, unknown `params`;
- non-deterministic values — `Math.random()`, `Date.now()`, `crypto.randomUUID()` (use `connection()` first, then `<Suspense>`, to get a per-request value).

```tsx
// The canonical shape: static + cached + streamed on one page.
import { Suspense } from 'react'
import { cookies } from 'next/headers'
import { cacheLife, cacheTag } from 'next/cache'

export default function BlogPage() {
  return (
    <>
      <header><h1>Our Blog</h1></header>   {/* static shell */}
      <BlogPosts />                         {/* cached → also in the shell */}
      <Suspense fallback={<p>Loading your preferences…</p>}>
        <UserPreferences />                 {/* runtime → streams */}
      </Suspense>
    </>
  )
}

async function BlogPosts() {
  'use cache'
  cacheLife('hours')
  cacheTag('posts')
  const res = await fetch('https://api.vercel.app/blog')
  return <PostList posts={await res.json()} />
}

async function UserPreferences() {
  const theme = (await cookies()).get('theme')?.value ?? 'light'
  return <aside>Your theme: {theme}</aside>
}
```

The payoff is in that last component: **reading `cookies()` no longer makes the whole route dynamic.** Under the previous model, one `cookies()` call anywhere opted the entire page out of static rendering. Under PPR, it opts out only its own Suspense boundary ([Caching — static, cached, and streaming](https://nextjs.org/docs/app/getting-started/caching#static-cached-and-streaming)).

**Two variants worth knowing:**

- `'use cache: private'` — gives a lifetime to a function that reads `cookies()`/`headers()` directly, caching the result **in the browser** as part of a per-link prefetch. Per-viewer, never shared.
- `'use cache: remote'` — moves an entry out of the per-instance in-memory store into a durable [cache handler](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheHandlers) shared across instances. A network roundtrip that only pays off at a high hit rate — and the thing you need when self-hosting multiple replicas.

**Should you enable it?** An honest read of where things stand:

| Enable it if | Stay on the default if |
|---|---|
| Starting a new project and you want the model the framework is converging on | You are migrating an existing app under time pressure |
| Your pages genuinely mix per-user and shared content | Your app is almost entirely per-user (a dashboard behind login) — the shell is nearly empty and PPR buys little |
| You can absorb build errors that demand structural changes | You need `unstable_cache` semantics you already understand |

Note that enabling it is explicitly *not* a rename-only change ([Upgrade guide](https://nextjs.org/docs/app/guides/upgrading/version-16#experimentaldynamicio-and-experimentalusecache)), and if you were using `experimental.ppr` on a Next.js 15 canary, PPR in 16 works differently — the docs tell you to **stay on your 15 canary** until you migrate deliberately ([Partial Prerendering](https://nextjs.org/docs/app/guides/upgrading/version-16#partial-prerendering-ppr)).

> **Real Scenario — the crawler that saw a broken page.**
> An e-commerce site enabled Cache Components. Users saw fast pages. Google Search Console started reporting 500s on product pages.
> **Cause:** browsers get the static shell instantly, but bots are detected by user agent and served a **full dynamic render at request time** instead — they need a complete document ([Caching — bots and crawlers](https://nextjs.org/docs/app/getting-started/caching#bots-and-crawlers)). Part of this site's shell read a build-time-generated manifest from the filesystem, which existed in the build image but not in the slim runtime container. Human visitors never hit that path because they got the prebuilt shell; every crawler did.
> **Fix:** ship the manifest into the runtime image. **The rule:** everything your static shell depends on must also be reachable at request time.

### 5.5 Cheat sheet: Part 5

**Default model (`cacheComponents: false`)**

| Goal | API |
|---|---|
| Cache one `fetch` | `fetch(url, { cache: 'force-cache' })` |
| Time-based | `fetch(url, { next: { revalidate: 3600 } })` |
| Tag a `fetch` | `fetch(url, { next: { tags: ['user'] } })` |
| Cache a DB query | `unstable_cache(fn, keyParts, { tags, revalidate })` |
| Dedupe within a request | `cache()` from `react` |
| Force a route dynamic | `export const dynamic = 'force-dynamic'` |
| Fail the build if a route goes dynamic | `export const dynamic = 'error'` |
| Prerender dynamic routes | `generateStaticParams()` |

**Cache Components (`cacheComponents: true`)**

| Goal | API |
|---|---|
| Cache a function or component | `'use cache'` |
| Set its lifetime | `cacheLife('hours' \| 'days' \| 'max' \| …)` |
| Tag it | `cacheTag('posts')` |
| Cache per-viewer runtime data | `'use cache: private'` |
| Durable, cross-instance cache | `'use cache: remote'` + `cacheHandlers` |
| Per-request random/time value | `await connection()` then `<Suspense>` |

**Invalidation**

| API | Timing | Re-render in same response | Where |
|---|---|---|---|
| `updateTag(tag)` | immediate | yes | Server Actions |
| `revalidateTag(tag, profile)` | stale-while-revalidate | no | Actions / Handlers |
| `revalidatePath(path)` | immediate | yes | Actions / Handlers |
| `refresh()` | uncached data only | yes | Server Actions |
| `router.refresh()` | uncached data only | yes | Client Components |

**Debugging**

| Symptom | Check |
|---|---|
| Data never updates in prod, fine in dev | Dev never caches. Reproduce with `next build && next start` |
| `use cache` has no effect | `cacheComponents: true` is not set |
| Stale after save | Used `revalidateTag(…, 'max')` where `updateTag` was needed |
| Cache empty after every deploy | Expected — cache keys include the build ID |
| Only one of N replicas updates | In-memory per-instance cache — [Part 13](./nextjs-production-deployment.md#part-13-deployment-and-internals) |
| Route unexpectedly dynamic | A `cookies()`/`headers()`/`searchParams` read somewhere in the tree |

[↑ Back to top](#table-of-contents)

---

**Next:** [Mutations & API Layer (Parts 6–7)](./nextjs-mutations-api.md) — writing data with Server Actions, and when you need a real endpoint instead.
