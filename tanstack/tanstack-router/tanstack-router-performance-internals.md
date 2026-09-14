# TanStack Router — Performance & Internals (Parts 10–13)

> Code splitting, preloading, devtools and testing — then SSR in brief, and the two mechanisms that explain how the whole thing works.

**Written against:** `@tanstack/react-router` **1.170.x**, `@tanstack/router-plugin` 1.170.x, `@tanstack/react-router-devtools` 1.170.x, Vitest 3.x, React 19
**Last updated:** September 14, 2026
**Part of:** [TanStack Router Mastery Guide](./tanstack-router-mastery-guide.md) — read its [About This Document](./tanstack-router-mastery-guide.md#about-this-document) before editing this file.

---

## Table of Contents

- [Part 10 — Code Splitting and Preloading](#part-10)
  - [Beginner: the critical / non-critical split](#p10-beginner)
  - [Working Knowledge: `autoCodeSplitting` and `.lazy.tsx`](#p10-working)
  - [Advanced: preloading strategies](#p10-advanced)
  - [Mastery: why splitting and preloading are the same design](#p10-mastery)
  - [Part 10 cheat sheet](#p10-cheat)
- [Part 11 — Devtools, Plugin Config and Testing](#part-11)
  - [Devtools](#p11-devtools)
  - [Plugin configuration reference](#p11-plugin)
  - [Testing](#p11-testing)
  - [Part 11 cheat sheet](#p11-cheat)
- [Part 12 — SSR and TanStack Start (brief)](#part-12)
- [Part 13 — Internals](#part-13)
  - [Mastery: the segment trie](#p13-trie)
  - [Mastery: how the types actually work](#p13-types)
- [Where to go next](#next)

---

<a id="part-10"></a>
## Part 10 — Code Splitting and Preloading

<a id="p10-beginner"></a>
### Beginner: the critical / non-critical split

Route config divides in two, and everything about performance in this router follows from where the line falls ([code splitting guide](https://tanstack.com/router/latest/docs/framework/react/guide/code-splitting)):

| **Critical** — always in the main bundle | **Non-critical** — code-split |
|---|---|
| path parsing | `component` |
| `validateSearch` | `errorComponent` |
| `loader` | `pendingComponent` |
| route `context` | `notFoundComponent` |
| static data | |

The guide's own framing: critical config is "everything needed to render a route and initiate data loading immediately."

The reason is causal, not stylistic. To preload data on hover, the router must know — *without downloading anything* — which route a URL matches, what its params parse to, what search schema applies, and what loader to call. If any of that lived in a lazily-loaded chunk, preloading would require a round trip just to discover what to fetch.

```
  If loaders were code-split:
    hover → download route chunk → discover the loader → fetch data → navigate
            └───── round trip ─────┘                     └─ round trip ─┘

  As designed:
    hover → fetch data immediately (loader is already loaded)
          → download component chunk in parallel
            └────────────── one wait, two parallel streams ──────────────┘
```

**Loaders are not split by default**, and the guide gives the reason directly: to avoid "double async costs" and to keep routes preloadable. **`__root.tsx` cannot be split at all** — it is always matched, so deferring it can never help. Keep it small.

[↑ Back to top](#table-of-contents)

---

<a id="p10-working"></a>
### Working Knowledge: `autoCodeSplitting` and `.lazy.tsx`

**The recommended way is one config line.** For file-based routing with a supported bundler, the guide calls automatic splitting "the preferred approach":

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { tanstackRouter } from '@tanstack/router-plugin/vite'

export default defineConfig({
  plugins: [
    tanstackRouter({
      autoCodeSplitting: true,
    }),
    react(),
  ],
})
```

With this on, the plugin separates critical from non-critical config itself. You write ordinary single-file routes and get split output. Note the default is `false` ([file-based routing API](https://tanstack.com/router/latest/docs/api/file-based-routing)) — turning it on is a deliberate step.

**The manual way** is the `.lazy.tsx` suffix, useful when you want explicit control or are not on a supported bundler:

```tsx
// src/routes/posts.tsx — critical config
import { createFileRoute } from '@tanstack/react-router'
import { fetchPosts } from './api'

export const Route = createFileRoute('/posts')({
  loader: fetchPosts,
})
```

```tsx
// src/routes/posts.lazy.tsx — non-critical config
import { createLazyFileRoute } from '@tanstack/react-router'

export const Route = createLazyFileRoute('/posts')({
  component: Posts,
})

function Posts() {
  // ...
}
```

Two files, same route path, different `create*` function. And a neat detail: if you move *everything* out of the non-lazy file, **delete it** — the guide notes a virtual route is generated automatically to anchor the split files.

> **Wrong vs. right — hand-rolling `React.lazy`.**
>
> ```tsx
> // ❌ WRONG — the route's loader now lives behind the same chunk boundary
> //    as the component, so preloading cannot start the fetch early,
> //    and you have reinvented a Suspense boundary the router already manages.
> const Posts = React.lazy(() => import('./Posts'))
> export const Route = createFileRoute('/posts')({
>   component: () => <Suspense fallback={<Spinner/>}><Posts/></Suspense>,
> })
>
> // ✅ RIGHT — let the plugin split along the critical boundary
> export const Route = createFileRoute('/posts')({
>   loader: fetchPosts,
>   component: Posts,
> })
> // plus autoCodeSplitting: true
> ```

**Keep components in other files split.** If a component outside the route file imports `Route` from the route file to get typed hooks, it creates an import edge that can pull critical config into the wrong chunk. Use `getRouteApi` instead — the guide names this as one of its purposes:

```tsx
// src/routes/-components/PostsTable.tsx
const route = getRouteApi('/posts')
export function PostsTable() {
  const posts = route.useLoaderData()
  // …
}
```

[↑ Back to top](#table-of-contents)

---

<a id="p10-advanced"></a>
### Advanced: preloading strategies

Preloading fetches a route's **data and chunk** before the user commits to navigating. Three strategies ([preloading guide](https://tanstack.com/router/latest/docs/framework/react/guide/preloading)):

| Strategy | Trigger | Best for |
|---|---|---|
| `'intent'` | hover and touch-start on a `<Link>` | the common case — nav links, list rows |
| `'viewport'` | Intersection Observer — link enters the viewport | content further down a long page |
| `'render'` | as soon as the `<Link>` renders | the one route you know they will visit |

Enable globally:

```tsx
const router = createRouter({
  defaultPreload: 'intent',
})
```

This is the single highest-value line in a TanStack Router config. On a typical link, the gap between hover and click is 100–300 ms — long enough to have the data in hand by the time the click lands, which turns a spinner into an instant transition.

**Tuning:**

```tsx
const router = createRouter({
  defaultPreloadDelay: 100,       // default 50ms — wait this long after hover
  defaultPreloadStaleTime: 10_000, // default 30s — freshness of preloaded data
})
```

`defaultPreloadDelay` is a sensitivity dial: too low and a pointer sweeping across a nav bar preloads six routes; too high and the preload has not finished by click time. 50 ms is a good default; raise it for dense lists.

**With an external cache, set it to zero:**

```tsx
const router = createRouter({
  defaultPreloadStaleTime: 0,
})
```

This makes preloaded data immediately stale from the router's perspective so your external cache decides freshness — the reasoning is in [Part 7](./tanstack-router-data-loading.md#p7-advanced). The router still keeps the work useful: it retains preloads via `preloadGcTime` and dedupes overlapping preload/navigation requests.

**Per-link overrides** work as you would expect:

```tsx
<Link to="/heavy-report" preload={false}>Report</Link>
<Link to="/next-page" preload="render">Next</Link>
```

**Manual preloading**, for cases the `<Link>` heuristics do not cover — a keyboard-focused row, a wizard step you know comes next:

```tsx
const matches = await router.preloadRoute({
  to: postRoute,
  params: { id: 1 },
})

await router.loadRouteChunk(router.routesByPath['/posts'])   // JS chunk only
```

`preloadRoute` does data and chunk; `loadRouteChunk` does the chunk alone — useful when the data is user-specific and you only want the code warm.

> **Wrong vs. right — `preload: 'render'` as a blanket default.**
>
> ```tsx
> // ❌ WRONG — a 50-row list now fires 50 loaders on mount.
> //    You have replaced lazy loading with eager loading and called it a perf win.
> defaultPreload: 'render'
>
> // ✅ RIGHT — speculate on intent; reserve 'render' for the specific link
> defaultPreload: 'intent'
> // …and on the one link that always gets clicked:
> <Link to="/checkout/payment" preload="render">Continue</Link>
> ```

**🧪 Try It — measure the difference.**

1. Set `defaultPreload: false`. Open the Network tab, hover a `/posts` link for a second, then click.
2. Note when the request fires: on click.
3. Set `defaultPreload: 'intent'`, hard-reload, hover for a second, click.

**Expected output:** in step 3 the request fires roughly 50 ms *after hover begins*, and by click time the response is already in. The route transition renders without a pending state at all — the `pendingComponent` you wrote in [Part 9](./tanstack-router-guards-lifecycle.md#p9-beginner) simply never appears for that navigation.

[↑ Back to top](#table-of-contents)

---

<a id="p10-mastery"></a>
### Mastery: why splitting and preloading are the same design

These are usually taught as two optimisations. They are one.

```
  The critical/non-critical split exists SO THAT preloading can work.
  Preloading exists to hide the cost that splitting introduces.

         ┌──────────────────────────────────────────────┐
         │  CRITICAL config stays in the main bundle    │
         │  · match the URL                             │
         │  · parse params, validate search             │
         │  · know which loader to call                 │
         └───────────────────┬──────────────────────────┘
                             │  all available at hover time,
                             │  with zero network cost
                             ▼
         ┌──────────────────────────────────────────────┐
         │  On hover, the router can therefore START    │
         │  BOTH streams in parallel:                   │
         │     ① loader()        → the data             │
         │     ② import(chunk)   → the component        │
         └───────────────────┬──────────────────────────┘
                             │
                             ▼
              On click: both are usually already resolved.
              Render is synchronous. No spinner.
```

Every design decision in this area falls out of that diagram:

- **Why `loader` is critical** — otherwise stream ① could not start until stream ② finished. A sequential waterfall instead of two parallel streams.
- **Why `validateSearch` is critical** — the loader's deps often derive from search params, so you cannot even construct the request without it.
- **Why `__root` cannot be split** — it is on the critical path of every navigation; there is no "later" to defer it to.
- **Why `defaultPreloadStaleTime` defaults to 30s rather than 0** — preloads are speculative and often wasted; caching them prevents a hover-heavy UI from hammering the network.
- **Why `getRouteApi` exists at all** — importing `Route` across files would defeat the chunk boundary that makes the split work.

The router's whole performance story is: **make the decision layer cheap and always-present, make the presentation layer lazy, and use the gap between intent and action to load both halves at once.**

[↑ Back to top](#table-of-contents)

---

<a id="p10-cheat"></a>
### Part 10 cheat sheet

| Task | API |
|---|---|
| Split automatically | `tanstackRouter({ autoCodeSplitting: true })` (default `false`) |
| Split manually | `posts.tsx` (critical) + `posts.lazy.tsx` via `createLazyFileRoute` |
| All config moved out? | Delete the non-lazy file; a virtual route is generated |
| Typed hooks in other files | `getRouteApi('/path')` — never import `Route` across the boundary |
| Never split | `__root.tsx` |
| Preload globally | `defaultPreload: 'intent'` |
| Preload strategies | `'intent'` · `'viewport'` · `'render'` · `false` |
| Hover sensitivity | `defaultPreloadDelay` (default `50`) |
| Preload freshness | `defaultPreloadStaleTime` (default `30_000`; `0` with Query) |
| Per-link override | `<Link preload="render" />` / `preload={false}` |
| Manual, data + chunk | `router.preloadRoute({ to, params })` |
| Manual, chunk only | `router.loadRouteChunk(router.routesByPath['/x'])` |

[↑ Back to top](#table-of-contents)

---

<a id="part-11"></a>
## Part 11 — Devtools, Plugin Config and Testing

<a id="p11-devtools"></a>
### Devtools

```bash
npm install -D @tanstack/react-router-devtools
```

```tsx
import { TanStackRouterDevtools } from '@tanstack/react-router-devtools'

export const Route = createRootRoute({
  component: () => (
    <>
      <Outlet />
      <TanStackRouterDevtools />
    </>
  ),
})
```

Rendered inside the root route it finds the router automatically ([devtools docs](https://tanstack.com/router/latest/docs/framework/react/devtools)). Elsewhere, pass `router` explicitly.

Options worth knowing: `initialIsOpen` (defaults to closed), `position` (`bottom-left`, `top-right`, …), `panelProps` / `closeButtonProps` / `toggleButtonProps` for styling, and `shadowDOMTarget` when your app renders into a shadow root. There is an embedded mode via `TanStackRouterDevtoolsPanel` for putting the panel in your own layout, and `TanStackRouterDevtoolsInProd` when you deliberately want it in production.

**What to actually look at.** The **Matches** list is the highest-value panel: it shows the match stack for the current URL with each match's status, params, search and loader data. Three things it answers instantly:

- *Why is my component not rendering?* — the match is there but a parent is missing an `<Outlet/>` ([Part 2](./tanstack-router-foundations.md#p2-beginner)).
- *Why did my loader re-run?* — compare `deps` between navigations ([Part 6](./tanstack-router-data-loading.md#p6-working)).
- *Why is the wrong route matching?* — the match stack shows precisely which route won ([Part 3](./tanstack-router-routing-navigation.md#p3-advanced)).

[↑ Back to top](#table-of-contents)

---

<a id="p11-plugin"></a>
### Plugin configuration reference

The full option set and defaults, from the [file-based routing API](https://tanstack.com/router/latest/docs/api/file-based-routing):

| Option | Default | Notes |
|---|---|---|
| `routesDirectory` | `./src/routes` | Required |
| `generatedRouteTree` | `./src/routeTree.gen.ts` | Required; commit it |
| `routeFilePrefix` | `''` | Only files with this prefix are routes |
| `routeFileIgnorePrefix` | `-` | The excluded-files convention |
| `routeFileIgnorePattern` | `undefined` | Regex alternative |
| `routeToken` | `route` | Filename token for a layout route |
| `indexToken` | `index` | Filename token for an index route |
| `quoteStyle` | `single` | Generated-file formatting |
| `semicolons` | `false` | Generated-file formatting |
| `autoCodeSplitting` | `false` | See [Part 10](#p10-working) |
| `disableTypes` | `false` | Emit JS-only route tree |
| `addExtensions` | `false` | Include file extensions in generated imports |
| `disableLogging` | `false` | |
| `enableRouteTreeFormatting` | `true` | |
| `virtualRouteConfig` | `undefined` | Programmatic route trees |
| `routeTreeFileHeader` | eslint / ts-nocheck comments | |
| `routeTreeFileFooter` | `[]` | |
| `tmpDir` | `.tanstack/tmp` (or `TSR_TMP_DIR`) | |

Most projects change exactly one thing: `autoCodeSplitting: true`. `routeToken` and `indexToken` matter if you are migrating from a convention that used different filenames — you can, for instance, set `routeToken: 'layout'` so `posts.layout.tsx` means what `posts.tsx` means by default.

> **A discrepancy worth flagging.** The Vite installation snippet shows `target: 'react'` as an option, but the API reference table does not list `target`. Both are current documentation. The safe reading: pass `target: 'react'` as the install guide shows — it is what the guide's own example does — and treat its absence from the table as a documentation gap rather than a signal that it is unsupported. If your build works without it, it is being inferred.

[↑ Back to top](#table-of-contents)

---

<a id="p11-testing"></a>
### Testing

The router is testable because it does not require a browser history — you can hand it a **memory history** and drive it programmatically ([testing how-to](https://tanstack.com/router/latest/docs/how-to/test-file-based-routing)).

```ts
import { createRouter, createMemoryHistory } from '@tanstack/react-router'
import { routeTree } from '../routeTree.gen'

const router = createRouter({
  routeTree,
  history: createMemoryHistory({
    initialEntries: [initialLocation],
  }),
})
```

```tsx
function Wrapper({ children }: { children: React.ReactNode }) {
  return <RouterProvider router={router}>{children}</RouterProvider>
}
```

**Vitest config points** from the how-to: install `@tanstack/router-plugin/vite` in the *test* config too (so routes are generated before tests run), use the `jsdom` environment, point `setupFiles` at your setup module, enable `typecheck: { enabled: true }` so the generated tree's types are actually checked, and configure `routesDirectory` / `generatedRouteTree` on the plugin.

The how-to recommends building a `renderWithFileRoutes` helper that takes `initialLocation` and `routerContext`, creates the router from the generated tree, and returns both the render result and the router instance for assertions — so individual tests never construct route trees by hand.

```tsx
// src/test/file-route-utils.tsx  (sketch, following the how-to's shape)
export function renderWithFileRoutes(
  initialLocation: string,
  routerContext?: Partial<RouterContext>,
) {
  const router = createRouter({
    routeTree,
    history: createMemoryHistory({ initialEntries: [initialLocation] }),
    context: { ...defaultTestContext, ...routerContext },
  })
  return { router, ...render(<RouterProvider router={router} />) }
}
```

This is where the `createAppRouter()` factory from [Part 1](./tanstack-router-foundations.md#p1-working) pays off — a fresh router per test means no state leaking between cases.

> **Wrong vs. right — reusing the app router in tests.**
>
> ```tsx
> // ❌ WRONG — one module-level router shared across tests. The loader cache,
> //    the history stack and the context all leak between cases, and failures
> //    depend on test order.
> import { router } from '../router'
>
> // ✅ RIGHT — a fresh router with a fresh memory history per test
> const { router } = renderWithFileRoutes('/posts/1')
> ```

The `routerContext` parameter is the natural seam for test doubles: pass a fake `auth` to exercise a [guard](./tanstack-router-guards-lifecycle.md#part-8), or a `QueryClient` seeded with fixtures to exercise the [Query integration](./tanstack-router-data-loading.md#p7-advanced) without network access.

**For unit-testing a single route's behaviour**, code-based routing ([Part 2](./tanstack-router-foundations.md#p2-advanced)) is often cleaner than the generated tree: build a two-route tree in the test file, and you have isolated the thing under test entirely.

[↑ Back to top](#table-of-contents)

---

<a id="p11-cheat"></a>
### Part 11 cheat sheet

| Task | API |
|---|---|
| Devtools in root | `<TanStackRouterDevtools />` inside `createRootRoute`'s component |
| Devtools elsewhere | pass `router={router}` |
| Embedded panel | `TanStackRouterDevtoolsPanel` |
| Production devtools | `TanStackRouterDevtoolsInProd` |
| Enable auto splitting | `tanstackRouter({ autoCodeSplitting: true })` |
| Rename layout/index tokens | `routeToken`, `indexToken` |
| Test history | `createMemoryHistory({ initialEntries: ['/posts/1'] })` |
| Test wrapper | `<RouterProvider router={router} />` |
| Inject test doubles | `createRouter({ context: { auth: fakeAuth } })` |
| Typecheck generated routes in CI | Vitest `typecheck: { enabled: true }` |

[↑ Back to top](#table-of-contents)

---

<a id="part-12"></a>
## Part 12 — SSR and TanStack Start (brief)

*Scope note, per the [About This Document](./tanstack-router-mastery-guide.md#about-this-document) section: this part is deliberately concept-level. SSR and TanStack Start warrant their own doc; treat this as orientation, and re-verify anything here before building on it — Start moves fast and the router docs label parts of it experimental.*

**The router supports SSR natively**, in two modes ([SSR guide](https://tanstack.com/router/latest/docs/framework/react/guide/ssr)):

**Non-streaming** — render the whole page on the server, send complete HTML with serialized loader data, hydrate on the client. The utilities are `defaultRenderHandler` or `renderRouterToString` paired with `RouterServer`, and `RouterClient` on the client side.

**Streaming** — send critical markup first, then stream the rest as it renders, via `defaultStreamHandler` or `renderRouterToStream`. This is where [deferred loader promises](./tanstack-router-data-loading.md#p6-mastery) pay off: unresolved promises are serialized into the HTML and streamed as they settle.

The router handles **loader dehydration/hydration automatically** and ships a lightweight serializer that covers `undefined`, `Date`, `Error` and `FormData` — the values that plain `JSON.stringify` silently mangles.

**What TanStack Start adds:** `createServerFn` for server functions, and automatic history creation on both server and client. The SSR guide notes these "can be subject to change and should be regarded as experimental until Start reaches stable status." The guide does *not* require Start — it presents the router's own SSR capabilities as sufficient, with Start as an enhancement.

**How to think about the choice:**

| | Client-only SPA | Router + your own SSR | TanStack Start |
|---|---|---|---|
| Setup cost | none | you own the server entry, hydration, serialization wiring | framework-managed |
| Server functions | — | your own API layer | `createServerFn` |
| Stability | stable | stable | experimental in parts |
| Good for | dashboards, internal tools, anything behind a login | SEO-critical pages on an existing server | greenfield full-stack |

**The one thing to carry back to earlier parts:** if SSR is even possible in your future, create the router with a **factory** rather than a module-level singleton ([Part 1](./tanstack-router-foundations.md#p1-working)). A shared router — and a shared `QueryClient` — across server requests leaks one user's data into another's response. The [external data loading guide](https://tanstack.com/router/latest/docs/framework/react/guide/external-data-loading) makes the same point about creating the `queryClient` inside `createRouter()` so "each request gets its own isolated instance."

[↑ Back to top](#table-of-contents)

---

<a id="part-13"></a>
## Part 13 — Internals

*Mastery tier only. Nothing here changes how you write routes; it changes how confidently you predict what they will do.*

<a id="p13-trie"></a>
### Mastery: the segment trie

[Part 3](./tanstack-router-routing-navigation.md#p3-advanced) described matching as "sort by specificity, then walk the list." That is the correct mental model for *precedence*. The implementation is different, and the story of why is instructive.

The [TanStack engineering post on the rewrite](https://tanstack.com/blog/tanstack-router-route-matching-tree-rewrite) (published November 18, 2025) describes the old approach as "a sorted flat list of all routes, iterating through each to find a match." That had two problems, and the second is the interesting one:

1. **Performance** — O(N) in the number of routes, for every navigation.
2. **Correctness** — the sort "lacked strict ordering and behaved differently between Chrome and Firefox."

The second is a genuinely subtle class of bug. JavaScript's `Array.prototype.sort` is only required to be stable, not to produce a particular order for elements a comparator deems equal — and different engines have historically used different algorithms. So a comparator that returned `0` for two routes it could not distinguish left their relative order engine-dependent. An app could route correctly in Chrome and incorrectly in Firefox, from the same bundle.

**The fix was a segment trie** — a tree where each node is one pathname segment:

```
  Routes:  /posts            /posts/new       /posts/$postId
           /about            /about/us        /files/$

                        (root)
                          │
        ┌─────────────────┼──────────────────┐
        │                 │                  │
     "posts"           "about"            "files"
        │                 │                  │
   ┌────┴────┐         ┌──┴───┐              │
   │         │         │      │              ▼
 "new"   $postId      (idx)  "us"          $ (splat)
 static  dynamic                           lowest priority

  Matching /posts/new:
    segment "posts" → follow the static edge
    segment "new"   → static child exists → done. Two lookups.

  Matching /posts/42:
    segment "posts" → follow the static edge
    segment "42"    → no static child named "42" → fall to the dynamic
                       child $postId → capture. Two lookups.
```

Complexity moves from **O(N)** in the number of routes to **O(M)** in the number of segments in the pathname. The blog reports ~60× faster for a small app (10 routes) and ~10,000× for a large one (450 routes) — and notes explicitly that the headline "20,000×" figure came from cherry-picked production scenarios, which is an unusually honest thing for a performance post to say.

**Why precedence still works as documented.** The ordering (index → static → dynamic → splat) is now encoded in *which edge the traversal tries first at each node*, rather than in a global sort. Static children are consulted before dynamic ones, dynamic before splat. Same observable rules, deterministic across engines, and no longer dependent on comparator behaviour.

**What this means for you day to day:** route count is essentially free. An app with 500 routes matches as fast as one with 10. Split your routes as finely as the domain wants; performance is not the constraint.

[↑ Back to top](#table-of-contents)

---

<a id="p13-types"></a>
### Mastery: how the types actually work

The "100% inferred TypeScript support" claim rests on three mechanisms, none of them magic, all of them worth being able to name when something goes wrong.

**① Codegen turns filenames into a literal-typed tree.**

The plugin reads `src/routes/`, computes each route's full path, and writes `routeTree.gen.ts`. That file is a chain of `createRoute(...).addChildren([...])` calls — every path is a **string literal type**, not `string`. `'/posts/$postId'` is its own type.

```ts
// what the generated file effectively expresses
const routeTree = rootRoute.addChildren({
  PostsRoute: postsRoute.addChildren({
    PostsPostIdRoute: postsPostIdRoute,   // path: '/posts/$postId' (literal)
  }),
  AboutRoute: aboutRoute,                  // path: '/about' (literal)
})
```

**② Declaration merging publishes that type to the library.**

```ts
declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router
  }
}
```

The library's own types reference `Register['router']` wherever they need to know about your app. This is the inversion that makes it work: `<Link>` is defined in a package that has never seen your code, yet its `to` prop is typed against your route tree, because your application *augments the library's interface*. If you have wondered why this library is so insistent about that four-line block, this is why — remove it and `Register` falls back to a default with no routes, `to` widens to `string`, and every downstream type degrades silently rather than erroring.

**③ Template-literal types derive params from paths.**

```
  '/posts/$postId/comments/$commentId'
          ──────           ─────────
              │                │
              └────────┬───────┘
                       ▼
      recursive conditional type splits on '/',
      keeps segments starting with '$',
      strips the '$', builds a union of keys
                       ▼
         { postId: string; commentId: string }
```

That derived object type is what makes `<Link params={{ postId }}>` require exactly the right keys, and what makes `Route.useParams()` return them without annotation. `params.parse` then *overrides* the value types — declare `parse: ({ postId }) => ({ postId: Number(postId) })` and the whole chain becomes `{ postId: number }`, including what `<Link>` demands.

**Why this occasionally hurts, and what to do about it.**

Type-level computation is not free. Large route trees (several hundred routes) produce large union and conditional types, and you may notice editor latency or slow `tsc` runs. Practical mitigations, in order of what to try first:

1. **Check your TypeScript version.** Inference performance for this style of type has improved substantially across recent releases; being one major version behind is a common cause.
2. **Use `getRouteApi` rather than importing `Route` across files.** Fewer cross-file type dependencies means less to re-check on every edit.
3. **Avoid unnecessarily wide search schemas.** A `z.record(z.unknown())` in `validateSearch` propagates an unhelpful type through everything downstream that touches it.
4. **Do not re-wrap `Link` with hand-written prop types** — as [Part 4](./tanstack-router-routing-navigation.md#p4-advanced) covers, this both loses safety and can make inference work harder. `createLink` exists for this.

The tradeoff is deliberate and, for most apps, plainly worth it: compile-time cost in exchange for a class of routing bug that simply cannot ship.

[↑ Back to top](#table-of-contents)

---

<a id="next"></a>
## Where to go next

- **Back to:** [Overview](./tanstack-router-mastery-guide.md) — the master cheat sheet collects everything from all six docs.
- **Sibling docs in this repo:** [Vite](../../vite/vite-mastery-guide.md) for the plugin and build pipeline, [React](../../react/react-mastery-guide.md) for Suspense and error boundaries, [TypeScript](../../typescript/typescript-mastery-guide.md) for the declaration merging and template-literal types this router leans on.
- **Planned:** a `tanstack/tanstack-query/` set, at which point the [Part 7 primer](./tanstack-router-data-loading.md#query-primer) should be trimmed to a cross-link.

### Quick self-check for Parts 10–13

1. Why is `loader` critical config while `component` is not? Give the causal reason, not the list.
2. What happens if you move every option out of `posts.tsx` into `posts.lazy.tsx`?
3. `defaultPreload: 'render'` on a 50-row list. What did you just do?
4. Which two things does `router.preloadRoute` fetch, and which one does `loadRouteChunk` skip?
5. Name the single option most projects change in the plugin config, and its default.
6. Why does a test need `createMemoryHistory` rather than the app's router?
7. Under SSR, what specifically goes wrong with a module-level `QueryClient`?
8. The old matcher "behaved differently between Chrome and Firefox." What property of `Array.prototype.sort` made that possible?
9. Route count is no longer a performance concern. What is the complexity now, and in what?
10. You delete the `declare module` block. Describe the failure — and note that it is not a compile error.

[↑ Back to top](#table-of-contents)
