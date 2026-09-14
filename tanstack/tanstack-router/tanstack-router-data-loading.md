# TanStack Router — Data Loading (Parts 6–7)

> Loaders, the built-in SWR cache, router context as dependency injection, and the division of labour with TanStack Query.

**Written against:** `@tanstack/react-router` **1.170.x**, `@tanstack/react-query` 5.x, React 19, TypeScript 5.9+
**Last updated:** September 14, 2026
**Part of:** [TanStack Router Mastery Guide](./tanstack-router-mastery-guide.md) — read its [About This Document](./tanstack-router-mastery-guide.md#about-this-document) before editing this file.

---

## Table of Contents

- [Part 6 — Loaders and the Built-in Cache](#part-6)
  - [Beginner: a loader and `useLoaderData`](#p6-beginner)
  - [Working Knowledge: `loaderDeps`, and connecting search to data](#p6-working)
  - [Advanced: `staleTime`, `gcTime`, `shouldReload`](#p6-advanced)
  - [Mastery: deferred data and streaming with `Await`](#p6-mastery)
  - [Part 6 cheat sheet](#p6-cheat)
- [Part 7 — Router Context and External Caches](#part-7)
  - [Beginner: typed router context](#p7-beginner)
  - [Working Knowledge: extending context in `beforeLoad`](#p7-working)
  - [A short TanStack Query primer](#query-primer)
  - [Advanced: the Router + Query integration](#p7-advanced)
  - [Mastery: choosing which cache owns your data](#p7-mastery)
  - [Part 7 cheat sheet](#p7-cheat)
- [Where to go next](#next)

---

<a id="part-6"></a>
## Part 6 — Loaders and the Built-in Cache

<a id="p6-beginner"></a>
### Beginner: a loader and `useLoaderData`

A loader is a function that runs **when a route match loads** — before the component renders. From the [data loading guide](https://tanstack.com/router/latest/docs/framework/react/guide/data-loading), it receives one object with `params`, `context`, `deps`, `abortController` and more.

```tsx
export const Route = createFileRoute('/posts')({
  loader: () => fetchPosts(),
})

function Posts() {
  const posts = Route.useLoaderData()   // typed from the loader's return
  return <ul>{posts.map((p) => <li key={p.id}>{p.title}</li>)}</ul>
}
```

The type of `useLoaderData()` is inferred from the loader's return type. Nothing is declared twice.

**Why this beats fetching in the component.** The component does not render until the loader resolves, so there is no `if (!data) return <Spinner/>` branch, no `data` that might be `undefined`, and no render-then-fetch waterfall. When you navigate to `/posts/42`, the loaders for `/posts` and `/posts/42` run **in parallel** — the router knows the whole match stack up front, which it could not if each level discovered its own data needs during render.

```
  Component-fetching (the thing you are leaving behind)

  navigate → render <Posts/> → useEffect → fetch → render <Post/> → useEffect → fetch
             └──── frame 1 ────┘          └─── frame 2 ────┘         └─ frame 3 ─┘
                   spinner                      spinner                   content

  Loader-fetching

  navigate → match /posts + /posts/42 → run both loaders in parallel → render once
                                        └──────── one wait ────────┘    content
```

There is also an **object form** of `loader`, used when you need to configure it:

```tsx
export const Route = createFileRoute('/posts')({
  loader: {
    handler: () => fetchPosts(),
    staleReloadMode: 'blocking',
  },
})
```

This form is newer than the bare-function form. Both are current; use the object form only when you need one of its options. If you are reading older material, you will only ever see the function form — that is age, not deprecation.

[↑ Back to top](#table-of-contents)

---

<a id="p6-working"></a>
### Working Knowledge: `loaderDeps`, and connecting search to data

**Loaders do not see search params by default.** This surprises people, and it is deliberate: search params change far more often than data dependencies do, and re-running every loader on every search change would be wasteful. So you declare, explicitly, which search params the loader depends on.

```tsx
export const Route = createFileRoute('/posts')({
  loaderDeps: ({ search: { offset, limit } }) => ({ offset, limit }),
  loader: ({ deps: { offset, limit } }) => fetchPosts({ offset, limit }),
})
```

`loaderDeps` does two jobs at once: it feeds `deps` into the loader, and its return value **is the cache key**. Two navigations with equal deps hit the cache; different deps mean a fresh load.

> **Wrong vs. right — the `loaderDeps` footgun.** The docs flag this explicitly: "Only include dependencies actually used in the loader. Returning the entire search object causes unnecessary reloads whenever any search param changes."
>
> ```tsx
> // ❌ WRONG — toggling ?sidebar=open refetches the posts
> loaderDeps: ({ search }) => search,
>
> // ✅ RIGHT — only what the fetch actually varies on
> loaderDeps: ({ search: { offset, limit } }) => ({ offset, limit }),
> ```
>
> This is where [structural sharing](./tanstack-router-search-params.md#mastery) pays off: because the router reuses references for deeply-equal sub-trees, a narrowly-selected deps object stays reference-stable across navigations that do not touch those keys, and the loader correctly does not re-run.

**A rule of thumb for what belongs in deps:** anything that appears in your fetch's URL or request body. `page`, `limit`, `sort`, `filter` — yes. `tab`, `sidebar`, `modal`, `theme` — no, those are view state, and they are exactly the params that would cause spurious refetches.

**🧪 Try It — cause the bug, then fix it.**

1. Give `/posts` a schema with `{ page: number; sidebar: boolean }`, and set `loaderDeps: ({ search }) => search`.
2. Put a `console.log('LOADER RAN')` at the top of the loader.
3. Click a link that only toggles `sidebar`.

**Expected output:** `LOADER RAN` in the console — a full refetch to open a sidebar.

Now change to `loaderDeps: ({ search: { page } }) => ({ page })` and toggle again. **Expected output:** nothing logged. The sidebar toggles instantly, off the existing data.

[↑ Back to top](#table-of-contents)

---

<a id="p6-advanced"></a>
### Advanced: `staleTime`, `gcTime`, `shouldReload`

The router has a built-in stale-while-revalidate cache. Three options control it.

**`staleTime`** — how long data stays fresh. The defaults are asymmetric, and the asymmetry is the interesting part:

| Situation | Default `staleTime` |
|---|---|
| Navigation | `0` ms |
| Preload | `30` seconds (`defaultPreloadStaleTime`) |

```tsx
export const Route = createFileRoute('/posts')({
  loader: () => fetchPosts(),
  staleTime: 10_000,
})
```

Why the asymmetry? A **navigation** is a user asking to see something — they should get current data, so `0` means "always revalidate." A **preload** is speculative — the router is fetching something on hover that the user may never visit, so re-preloading the same route every time the pointer crosses a link would be pure waste. Thirty seconds of "good enough" is the right default for speculation.

When data is stale, the router revalidates **in the background while showing the existing data** — that is the SWR part. Set `staleTime: Infinity` to disable automatic stale reloads entirely, which is right for genuinely immutable data (a published article, a config blob loaded once).

**`gcTime`** — how long *unused* data survives before garbage collection. Default is 5 minutes. Lower it for large payloads; raise it for data the user bounces between.

```tsx
export const Route = createFileRoute('/posts')({
  loader: ({ deps }) => fetchPosts(deps),
  gcTime: 0,
  shouldReload: false,
})
```

That pair is a documented idiom worth recognising: it **opts out of caching while keeping preload benefits**. Data is fetched on preload, used for the navigation that follows, and dropped immediately after. Good for anything you must not show twice — a one-time token, a rapidly-changing dashboard figure.

**`shouldReload`** — finer control than `staleTime`. It takes `false`, or a function receiving the same arguments as `beforeLoad`:

```tsx
shouldReload: false,   // only load on entry, or when deps change
```

The three together, as a decision table:

| You want | Set |
|---|---|
| Always fresh on navigation (default) | nothing |
| Cache for N seconds | `staleTime: N * 1000` |
| Never auto-revalidate | `staleTime: Infinity` |
| Reload only when deps change | `shouldReload: false` |
| Never retain after leaving | `gcTime: 0` |
| Conditional reload (e.g. only when a context flag is set) | `shouldReload: ({ context }) => …` |

> **Wrong vs. right — `staleTime` on a mutable resource.**
>
> ```tsx
> // ❌ WRONG — the user edits a post, navigates back to the list, sees the old title
> export const Route = createFileRoute('/posts')({
>   loader: () => fetchPosts(),
>   staleTime: Infinity,
> })
>
> // ✅ RIGHT — either revalidate…
> staleTime: 30_000,
> // …or keep Infinity and invalidate explicitly after the mutation
> await router.invalidate()
> ```
>
> `staleTime: Infinity` is a promise that the data will not change behind your back. If it can change, you own the invalidation.

**`preload` in the loader.** The loader receives a `preload` boolean so it can behave differently for speculative loads:

```tsx
loader: async ({ preload }) =>
  fetchPosts({
    maxAge: preload ? 10_000 : 0,
  }),
```

Useful for downgrading a preload to a cheaper request — skip an expensive join, accept a staler CDN response — while a real navigation gets the full thing.

[↑ Back to top](#table-of-contents)

---

<a id="p6-mastery"></a>
### Mastery: deferred data and streaming with `Await`

Sometimes one slow query holds up an entire page that is otherwise ready. Deferred loading lets the route render immediately with the fast data, while the slow data arrives later.

The mechanism is pleasingly plain: **return an unresolved promise from the loader and do not await it** ([deferred data loading guide](https://tanstack.com/router/latest/docs/framework/react/guide/deferred-data-loading)).

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => {
    const post = await fetchPost(params.postId)      // awaited: blocks render
    const commentsPromise = fetchComments(params.postId)  // NOT awaited

    return {
      post,
      deferredComments: commentsPromise,
    }
  },
  component: PostComponent,
})
```

Then unwrap it in the component with `<Await>`, which the docs describe as resolving the promise "by triggering the nearest suspense boundary until it is resolved, after which it renders the component's `children` as a function with the resolved data."

```tsx
function PostComponent() {
  const { post, deferredComments } = Route.useLoaderData()

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>

      <React.Suspense fallback={<p>Loading comments…</p>}>
        <Await promise={deferredComments}>
          {(comments) => (
            <ul>{comments.map((c) => <li key={c.id}>{c.text}</li>)}</ul>
          )}
        </Await>
      </React.Suspense>
    </article>
  )
}
```

```
  Timeline

  t=0    navigate
  t=0    loader starts fetchPost + fetchComments
  t=80   fetchPost resolves     → route renders: title, body, "Loading comments…"
  t=900  fetchComments resolves → <Await> children render, only that subtree updates

  Without deferral, nothing renders until t=900.
```

**Under SSR, this streams.** The docs note that deferred promises are serialised into the HTML and then streamed to the client as they resolve, giving progressive rendering without waiting for everything. That is the same machinery described in [Part 12](./tanstack-router-performance-internals.md#part-12).

> **A note on `defer()`.** You will find `defer(promise)` in older examples. The current guide shows returning the raw unresolved promise directly and does not document `defer()` as required. Treat the raw-promise form as the pattern to write; if you meet `defer()` in an existing codebase, it is age rather than error.

> **Wrong vs. right — the accidental await.**
>
> ```tsx
> // ❌ WRONG — Promise.all defeats the entire point; the page waits for both
> const [post, comments] = await Promise.all([fetchPost(id), fetchComments(id)])
> return { post, comments }
>
> // ✅ RIGHT — start both, await only the critical one
> const commentsPromise = fetchComments(id)
> const post = await fetchPost(id)
> return { post, deferredComments: commentsPromise }
> ```
>
> Note that in the right-hand version `fetchComments` is still **started first**. Both requests are in flight from the same tick; only the awaiting differs.

[↑ Back to top](#table-of-contents)

---

<a id="p6-cheat"></a>
### Part 6 cheat sheet

| Option | Purpose | Default |
|---|---|---|
| `loader` | Fetch before render; gets `{ params, deps, context, abortController, preload }` | — |
| `loader: { handler, staleReloadMode }` | Object form, for configuration | — |
| `loaderDeps` | Select search params the loader depends on; **is the cache key** | `undefined` |
| `staleTime` | Freshness window for navigation | `0` |
| `defaultPreloadStaleTime` | Freshness window for preloads (router option) | `30_000` |
| `gcTime` | Retention of unused data | 5 min |
| `shouldReload` | `false`, or a predicate; finer than `staleTime` | — |
| `Route.useLoaderData()` | Read the loader result, typed | — |
| `Await` + `Suspense` | Unwrap a deferred promise | — |
| `router.invalidate()` | Force recompute of context + reload of matches | — |

**Three rules to carry forward**

1. `loaderDeps` should name the exact params your fetch varies on, never the whole search object.
2. `staleTime: Infinity` means you own invalidation.
3. Await the critical data; return the slow promise unawaited and wrap it in `<Await>`.

[↑ Back to top](#table-of-contents)

---

<a id="part-7"></a>
## Part 7 — Router Context and External Caches

<a id="p7-beginner"></a>
### Beginner: typed router context

Router context is dependency injection for the route tree. Instead of importing a fetch function into every route, you hand it to the router once and every loader receives it.

Declare the shape on the root route with `createRootRouteWithContext` ([router context guide](https://tanstack.com/router/latest/docs/framework/react/guide/router-context)):

```tsx
// src/routes/__root.tsx
import { createRootRouteWithContext } from '@tanstack/react-router'

interface MyRouterContext {
  fetchPosts: typeof fetchPosts
}

export const Route = createRootRouteWithContext<MyRouterContext>()({
  component: App,
})
```

Note the **double call**: `createRootRouteWithContext<T>()(...)`. The first call fixes the context type; the second takes the route options. It reads oddly, and the reason is that TypeScript has no partial type-argument inference — you need to specify `T` explicitly while still letting the options infer. It is a workaround made visible in the API.

Supply the value at router creation:

```tsx
const router = createRouter({
  routeTree,
  context: { fetchPosts },
})
```

And consume it in any loader:

```tsx
export const Route = createFileRoute('/posts')({
  loader: ({ context: { fetchPosts } }) => fetchPosts(),
})
```

**A typing caveat from the docs**, worth quoting because it prevents a real confusion: *"`MyRouterContext` only needs to contain content that will be passed directly to `createRouter` below. All other context added in `beforeLoad` will be inferred."* You do not declare the full merged shape — only the root seed.

Required properties in that interface become required at `createRouter()`, which TypeScript enforces. Optional properties make the whole `context` argument optional.

[↑ Back to top](#table-of-contents)

---

<a id="p7-working"></a>
### Working Knowledge: extending context in `beforeLoad`

Any route can **add to** the context for itself and its descendants by returning an object from `beforeLoad`:

```tsx
export const Route = createFileRoute('/org/$orgId')({
  beforeLoad: async ({ params, context }) => {
    const org = await context.api.getOrg(params.orgId)
    return { org }     // merged into context for this route and all children
  },
})

// any descendant route:
loader: ({ context }) => context.api.getProjects(context.org.id),
//                                               ^ typed, inherited
```

```
  Context inheritance

  __root       { api, queryClient }
     │                                        ← seed, from createRouter
     ▼
  /org/$orgId  { api, queryClient, org }      ← beforeLoad returned { org }
     │
     ▼
  /org/$orgId/projects
               { api, queryClient, org }      ← inherited; can add more
```

The merge is per-route: each route's own context is stored separately, which is what makes it possible to walk the match list and build, say, a breadcrumb trail from each level's contribution.

**`router.invalidate()`** recomputes context across all routes and reloads matches. You need it when something *outside* the router changes in a way the routes' context depends on — most commonly an auth state transition, covered in [Part 8](./tanstack-router-guards-lifecycle.md#part-8).

> **Wrong vs. right — context is not reactive.**
>
> ```tsx
> // ❌ WRONG — context was seeded once at createRouter with the logged-out user.
> //    Logging in updates React state, but the router never re-evaluates.
> context: { auth }
>
> // ✅ RIGHT — after the auth state changes, tell the router to recompute
> await auth.login(credentials)
> await router.invalidate()
> ```
>
> This is the single most common router-context bug: context is a snapshot taken at router creation and at each `beforeLoad`, not a subscription. If a value can change, something has to call `invalidate()`.

[↑ Back to top](#table-of-contents)

---

<a id="query-primer"></a>
### A short TanStack Query primer

*You asked for this primed rather than assumed. It is deliberately minimal — just enough vocabulary for the integration section. A full `tanstack/tanstack-query/` doc set is planned.*

TanStack Query is a **client-side async cache**. Three concepts carry the integration:

**`QueryClient`** — the cache instance. One per app (or per server request, under SSR).

```tsx
const queryClient = new QueryClient()
```

**`queryOptions`** — a typed, reusable description of *one query*: its key and its fetcher. This is the unit you share between a loader and a component.

```tsx
import { queryOptions } from '@tanstack/react-query'

const postsQueryOptions = queryOptions({
  queryKey: ['posts'],
  queryFn: () => fetchPosts(),
})
```

**Reading and priming.** Two methods matter here:

- `queryClient.ensureQueryData(options)` — "make sure this is in the cache; fetch it if not; resolve when it is there." Imperative, awaitable, perfect for a loader.
- `useSuspenseQuery(options)` — a hook that reads the cache and *suspends* if the data is not there. It never returns `undefined`, so there is no loading branch in your component.

The property that makes Query worth combining with the router: **the cache is global and reactive**. Two components using the same `queryKey` share one entry and one request, and both update when it changes. The router's own cache is per-route-match, so it cannot do that.

[↑ Back to top](#table-of-contents)

---

<a id="p7-advanced"></a>
### Advanced: the Router + Query integration

This is the pattern most non-trivial TanStack Router apps converge on. The [external data loading guide](https://tanstack.com/router/latest/docs/framework/react/guide/external-data-loading) frames the router as "a perfect **coordinator** for external data fetching" rather than a storage layer.

**Step 1 — put the `queryClient` in router context.**

```tsx
const queryClient = new QueryClient()

const router = createRouter({
  routeTree,
  context: { queryClient },
  defaultPreloadStaleTime: 0,     // ← explained below
})
```

Creating the client inside a router factory matters for SSR: the docs note it ensures "each request gets its own isolated instance."

```tsx
// __root.tsx
interface RouterContext {
  queryClient: QueryClient
}
export const Route = createRootRouteWithContext<RouterContext>()({ ... })
```

**Step 2 — the loader guarantees the data, and returns nothing useful.**

```tsx
const postsQueryOptions = queryOptions({
  queryKey: ['posts'],
  queryFn: () => fetchPosts(),
})

export const Route = createFileRoute('/posts')({
  loader: ({ context: { queryClient } }) =>
    queryClient.ensureQueryData(postsQueryOptions),
  component: Posts,
})
```

**Step 3 — the component reads from Query, not from the loader.**

```tsx
function Posts() {
  const { data: posts } = useSuspenseQuery(postsQueryOptions)
  return <ul>{posts.map((p) => <li key={p.id}>{p.title}</li>)}</ul>
}
```

```
  Who does what

  ROUTER              "this route needs these queries warmed before it renders"
     │                 ensureQueryData in the loader; runs during navigation,
     │                 in parallel across the match stack
     ▼
  QUERY               "I own the cache, the freshness, the dedupe, the refetch,
     │                 the invalidation after mutations"
     ▼
  COMPONENT           "give me the data" — useSuspenseQuery; never undefined,
                       and reactive to any later cache update
```

The guide lists the four properties this buys: no waterfalls (data loads in parallel during navigation, not during render); no loading flashes (components only render once data is guaranteed); SEO-friendly (content exists at render time); and clean separation — "routers coordinate, libraries cache, components consume."

**Why `defaultPreloadStaleTime: 0`.** The router's preload cache would otherwise sit in front of Query's cache, and you would have two caches with two different freshness opinions about the same data. Setting it to `0` makes preloaded data immediately stale from the router's point of view, so **Query's `staleTime` becomes the only freshness authority**. The preloading guide is explicit about this for external libraries. The router still keeps the work useful — it retains preloads via `preloadGcTime` and dedupes overlapping preload and navigation work.

> **Wrong vs. right — reading loader data in the Query pattern.**
>
> ```tsx
> // ❌ WRONG — a snapshot. After a mutation invalidates the query,
> //    Query's cache updates and this component does not.
> const posts = Route.useLoaderData()
>
> // ✅ RIGHT — subscribed to the cache
> const { data: posts } = useSuspenseQuery(postsQueryOptions)
> ```
>
> `ensureQueryData` does return the data, so `useLoaderData()` compiles and shows the right thing on first render. The bug appears later, on the first mutation — which is exactly the kind of bug that survives review.

**🧪 Try It — prove the reactivity difference.**

1. Build the three steps above.
2. Add a button calling `queryClient.invalidateQueries({ queryKey: ['posts'] })`.
3. Render the list twice: once with `useSuspenseQuery`, once with `Route.useLoaderData()`.
4. Click the button.

**Expected output:** the `useSuspenseQuery` list refetches and updates; the `useLoaderData()` list keeps showing the original data until you navigate away and back. That divergence is the whole argument for step 3.

[↑ Back to top](#table-of-contents)

---

<a id="p7-mastery"></a>
### Mastery: choosing which cache owns your data

You now have two caches available. Running both without deciding which owns what is the actual failure mode — not either cache individually.

| | Router loader cache | TanStack Query |
|---|---|---|
| **Scope** | Per route match | Global, by `queryKey` |
| **Sharing across components** | Only via the route's `useLoaderData` | Any component, anywhere |
| **Reactive to external updates** | No — snapshot per match | Yes |
| **Invalidation** | `router.invalidate()` | `invalidateQueries`, per key |
| **Mutations** | Not modelled | `useMutation`, optimistic updates |
| **Setup cost** | Zero | A dependency and a provider |
| **Best for** | Data owned by exactly one route | Data read or mutated in several places |

**A decision rule that holds up:**

- **Router loader only** — the app has no mutations worth speaking of, or every piece of data belongs to exactly one route. A docs site, a dashboard that reads, a marketing app. Do not add Query for its own sake; the built-in cache is genuinely good and `staleTime`/`gcTime`/`shouldReload` cover a lot.
- **Router + Query** — anything with mutations, optimistic updates, or the same entity shown in two places. The moment you write "after saving, the list on the other page should update," you want Query.
- **Both, un-coordinated** — the mistake. Symptoms: stale data after mutations, double fetches on navigation, a `defaultPreloadStaleTime` you never set. If you adopt Query, adopt it wholesale for server state and let the loader be a pure `ensureQueryData` coordinator.

**One more consequence worth knowing.** Under the Query pattern, `loaderDeps` still matters — it determines when the *loader* re-runs, and therefore when `ensureQueryData` is called with fresh arguments. Your `queryKey` and your `loaderDeps` should name the same variables:

```tsx
const postsQueryOptions = (deps: { offset: number; limit: number }) =>
  queryOptions({
    queryKey: ['posts', deps],          // ← same variables
    queryFn: () => fetchPosts(deps),
  })

export const Route = createFileRoute('/posts')({
  loaderDeps: ({ search: { offset, limit } }) => ({ offset, limit }),  // ← here too
  loader: ({ context: { queryClient }, deps }) =>
    queryClient.ensureQueryData(postsQueryOptions(deps)),
})
```

If they drift apart, you get the confusing case where the URL changes, the loader re-runs, and the component still shows the old page — because the query key never changed.

[↑ Back to top](#table-of-contents)

---

<a id="p7-cheat"></a>
### Part 7 cheat sheet

| Task | API |
|---|---|
| Type the root context | `createRootRouteWithContext<T>()({ ... })` — note the double call |
| Seed it | `createRouter({ routeTree, context: { … } })` |
| Extend it for a subtree | `beforeLoad: () => ({ extra })` |
| Read it | `({ context })` in `beforeLoad` / `loader`; `Route.useRouteContext()` in components |
| Recompute after external change | `await router.invalidate()` |
| Query: describe a query | `queryOptions({ queryKey, queryFn })` |
| Query: warm it in a loader | `queryClient.ensureQueryData(opts)` |
| Query: read it in a component | `useSuspenseQuery(opts)` |
| Hand freshness to Query | `defaultPreloadStaleTime: 0` on the router |
| Keep keys aligned | `loaderDeps` variables == `queryKey` variables |

**Three rules to carry forward**

1. Context is a snapshot, not a subscription. Changing it means calling `router.invalidate()`.
2. Under the Query pattern, the loader's job is to *guarantee*, the component's job is to *subscribe*. Do not read `useLoaderData()` there.
3. Pick one owner for each piece of server state. Two caches with two opinions is the bug.

[↑ Back to top](#table-of-contents)

---

<a id="next"></a>
## Where to go next

- **Next in sequence:** [Guards & Lifecycle (Parts 8–9)](./tanstack-router-guards-lifecycle.md) — `beforeLoad` appears here as a context extender; there it does auth, redirects and errors.
- **Related:** `loaderDeps` depends on [structural sharing](./tanstack-router-search-params.md#mastery); preloading behaviour is in [Part 10](./tanstack-router-performance-internals.md#part-10).
- **Back to:** [Overview](./tanstack-router-mastery-guide.md)

### Quick self-check for Parts 6–7

1. Why do loaders not receive search params automatically?
2. `staleTime` defaults to `0` for navigation and `30_000` for preloads. Argue for both numbers.
3. What does the `gcTime: 0` + `shouldReload: false` pair achieve, and when would you reach for it?
4. In a deferred loader, why is it important that the slow fetch is *started* before the fast one is awaited?
5. Why is `createRootRouteWithContext` called twice?
6. You log in successfully but the guarded route still redirects you to `/login`. What did you forget?
7. In the Router+Query pattern, what breaks if the component uses `useLoaderData()` instead of `useSuspenseQuery()`?
8. Why is `defaultPreloadStaleTime: 0` correct specifically when Query is present, and wrong otherwise?

[↑ Back to top](#table-of-contents)
