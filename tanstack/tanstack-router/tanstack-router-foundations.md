# TanStack Router — Foundations (Parts 1–2)

> Getting a type-safe router running, and understanding what the generated route tree actually does for you.

**Written against:** `@tanstack/react-router` **1.170.x**, `@tanstack/router-plugin` 1.170.x, React 19, Vite 7+, TypeScript 5.9+
**Last updated:** September 14, 2026
**Part of:** [TanStack Router Mastery Guide](./tanstack-router-mastery-guide.md) — read its [About This Document](./tanstack-router-mastery-guide.md#about-this-document) before editing this file.

---

## Table of Contents

- [Part 1 — Setup and the Type-Safety Machine](#part-1)
  - [Beginner: the four moving pieces](#p1-beginner)
  - [Working Knowledge: what the plugin generates, and why registration matters](#p1-working)
  - [Part 1 cheat sheet](#p1-cheat)
- [Part 2 — The Route Tree and the Component Tree](#part-2)
  - [Beginner: the root route and `Outlet`](#p2-beginner)
  - [Working Knowledge: layout routes, nesting, and `getRouteApi`](#p2-working)
  - [Advanced: code-based routing, and when to reach for it](#p2-advanced)
  - [Mastery: what a `Route` object actually is](#p2-mastery)
  - [Part 2 cheat sheet](#p2-cheat)
- [Where to go next](#next)

---

<a id="part-1"></a>
## Part 1 — Setup and the Type-Safety Machine

*Tiers: **Beginner → Working Knowledge** only. There is no meaningful "Advanced setup" tier — the advanced material is the plugin configuration table, which lives in [Part 11](./tanstack-router-performance-internals.md#part-11) next to the build concerns it belongs with.*

<a id="p1-beginner"></a>
### Beginner: the four moving pieces

A TanStack Router app has exactly four moving pieces. Everything in the rest of this doc set is a refinement of one of them.

```
  ┌─────────────────┐   plugin watches   ┌──────────────────┐
  │  src/routes/    │ ─────────────────▶ │ routeTree.gen.ts │  ① generated tree
  │  __root.tsx     │    on every save   │  (do not edit)   │
  │  index.tsx      │                    └────────┬─────────┘
  │  posts.tsx      │                             │ imported by
  └─────────────────┘                             ▼
                                        ┌──────────────────┐
                                        │  createRouter()  │  ② the router instance
                                        └────────┬─────────┘
                                                 │
                    declare module ◀─────────────┤ ③ registration
                    (teaches TS about your tree) │
                                                 ▼
                                        ┌──────────────────┐
                                        │ <RouterProvider> │  ④ rendered once
                                        └──────────────────┘
```

**Install.** Three packages: the router, the build plugin, and the devtools.

```bash
npm install @tanstack/react-router
npm install -D @tanstack/router-plugin @tanstack/react-router-devtools
```

**Wire the plugin.** Per the [Vite installation guide](https://tanstack.com/router/latest/docs/installation/with-vite), the plugin must come **before** the React plugin:

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { tanstackRouter } from '@tanstack/router-plugin/vite'

export default defineConfig({
  plugins: [
    tanstackRouter({
      target: 'react',
      autoCodeSplitting: true,
    }),
    react(),
  ],
})
```

> **Wrong vs. right — plugin order.**
>
> ```ts
> // ❌ WRONG — react() transforms the files before the router plugin sees them
> plugins: [react(), tanstackRouter({ target: 'react' })]
>
> // ✅ RIGHT — router plugin first
> plugins: [tanstackRouter({ target: 'react' }), react()]
> ```
>
> The docs state the router plugin must be "placed before other framework plugins like `@vitejs/plugin-react`". Getting this backwards produces route files that generate but do not hot-reload correctly.

> **Outdated convention you will see in blog posts:** the plugin used to be exported as `TanStackRouterVite` from `@tanstack/router-plugin/vite`, and before that lived in a separate `@tanstack/router-vite-plugin` package. The current export is **`tanstackRouter`**. If a tutorial shows `TanStackRouterVite(...)`, it predates the rename — the options are otherwise the same.

**Write a root route and one page.**

```tsx
// src/routes/__root.tsx
import { createRootRoute, Link, Outlet } from '@tanstack/react-router'
import { TanStackRouterDevtools } from '@tanstack/react-router-devtools'

export const Route = createRootRoute({
  component: () => (
    <>
      <nav>
        <Link to="/">Home</Link> <Link to="/posts">Posts</Link>
      </nav>
      <hr />
      <Outlet />
      <TanStackRouterDevtools />
    </>
  ),
})
```

```tsx
// src/routes/index.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/')({
  component: () => <h1>Home</h1>,
})
```

Two conventions are already doing work here. `__root.tsx` is the one route that is **always matched** — the [routing concepts guide](https://tanstack.com/router/latest/docs/framework/react/routing/routing-concepts) describes the root route as "the top-most route in the entire tree" which "encapsulates all other routes as children", has no path, and is always rendered. And `createFileRoute('/')` takes the route's *path* as its argument — you do not type that string yourself in practice, the plugin writes it for you the first time it sees a new file, and keeps it in sync if you rename the file.

**Bootstrap.**

```tsx
// src/main.tsx
import { StrictMode } from 'react'
import ReactDOM from 'react-dom/client'
import { createRouter, RouterProvider } from '@tanstack/react-router'
import { routeTree } from './routeTree.gen'

const router = createRouter({
  routeTree,
  defaultPreload: 'intent',
  scrollRestoration: true,
})

// ③ Registration — this is the load-bearing step
declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router
  }
}

ReactDOM.createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <RouterProvider router={router} />
  </StrictMode>,
)
```

**🧪 Try It — prove the type safety is real.**

1. `npm run dev` once with the config above. Watch `src/routeTree.gen.ts` appear.
2. In `__root.tsx`, change `<Link to="/posts">` to `<Link to="/postz">`.
3. Run `npx tsc --noEmit`.

**Expected output:**

```
src/routes/__root.tsx:9:15 - error TS2322: Type '"/postz"' is not assignable to type
  '"/" | "/posts" | "/posts/$postId" | ...'
```

That union is not written anywhere by hand. It was derived from your filenames. Now delete the `declare module` block and re-run `tsc` — the error disappears, and `to` degrades to `string`. **That is what registration buys you**, and it is the single most common reason someone reports "the types aren't working."

[↑ Back to top](#table-of-contents)

---

<a id="p1-working"></a>
### Working Knowledge: what the plugin generates, and why registration matters

`routeTree.gen.ts` is a generated file — the plugin writes an eslint-disable and `@ts-nocheck` header into it by default ([file-based routing API](https://tanstack.com/router/latest/docs/api/file-based-routing)). You should commit it (so CI and fresh clones typecheck without a dev server run) and never edit it.

Its job is to produce a single `routeTree` value whose **type** encodes the entire shape of your app: every path, every path param, every `validateSearch` return type, every `loader` return type. `createRouter({ routeTree })` carries that type onto the router instance. The `declare module` block then attaches that instance type to the library's `Register` interface, and every *standalone* export — `<Link>`, `useNavigate`, `redirect`, `useParams` — reads it from there.

```
  filenames  ──►  routeTree.gen.ts  ──►  typeof router  ──►  interface Register
                                                                    │
                    ┌───────────────────────────────────────────────┘
                    ▼
        <Link to=…>   useNavigate()   redirect({to:…})   useParams({from:…})
        all type-checked against your real route tree, with zero manual types
```

This is TypeScript [declaration merging](../../typescript/typescript-modules-ecosystem.md) doing something genuinely clever: the library's own `.d.ts` files defer to a type you supply at the application level.

**The three failure modes, and how to recognise them.**

| Symptom | Cause | Fix |
|---|---|---|
| `to` accepts any string; no autocomplete | No `declare module`, or it is in a file that is never imported | Keep the block in the module where `router` is created, and make sure that module is in the build |
| `routeTree.gen.ts` is missing or stale | Plugin missing, plugin ordered after `react()`, or the dev server has never run | Run the dev server once; check plugin order |
| Types are right in the editor but CI fails | `routeTree.gen.ts` is gitignored | Commit it — it is a build input, not a build output |

> **Wrong vs. right — where the registration lives.**
>
> ```tsx
> // ❌ WRONG — declared in a types-only file with no imports.
> // Some bundler/tsconfig setups never pull this into the program,
> // and the augmentation silently does nothing.
> // src/types/router.d.ts
> declare module '@tanstack/react-router' {
>   interface Register { router: typeof import('../main').router }
> }
>
> // ✅ RIGHT — colocated with createRouter, in a module that is definitely loaded.
> // src/router.tsx
> export const router = createRouter({ routeTree })
> declare module '@tanstack/react-router' {
>   interface Register { router: typeof router }
> }
> ```

**A structure worth adopting now.** Extract the router into its own module rather than leaving it in `main.tsx`. You will need a `createRouter()` you can call from tests and (later) from a server entry:

```tsx
// src/router.tsx
import { createRouter } from '@tanstack/react-router'
import { routeTree } from './routeTree.gen'

export function createAppRouter() {
  return createRouter({
    routeTree,
    defaultPreload: 'intent',
    scrollRestoration: true,
  })
}

export const router = createAppRouter()

declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router
  }
}
```

The factory form matters for two things covered later: tests need a fresh router with a memory history per test ([Part 11](./tanstack-router-performance-internals.md#part-11)), and SSR needs a fresh router per request ([Part 12](./tanstack-router-performance-internals.md#part-12)).

**Scaffolding shortcut.** The [quick start](https://tanstack.com/router/latest/docs/framework/react/quick-start) points at a CLI that generates a working project with all of the above already wired — useful for a throwaway experiment, but do the manual setup at least once. The four pieces above are exactly what the generator produces, and knowing which one broke is most of debugging this library.

[↑ Back to top](#table-of-contents)

---

<a id="p1-cheat"></a>
### Part 1 cheat sheet

| Thing | Value |
|---|---|
| Runtime package | `@tanstack/react-router` |
| Build plugin | `@tanstack/router-plugin/vite`, export `tanstackRouter` |
| Devtools | `@tanstack/react-router-devtools` |
| Routes directory (default) | `./src/routes` |
| Generated tree (default) | `./src/routeTree.gen.ts` — commit it, never edit it |
| Root file | `__root.tsx` |
| Plugin order | **Before** `@vitejs/plugin-react` |
| Registration | `declare module '@tanstack/react-router' { interface Register { router: typeof router } }` |
| Renamed API to watch for | `TanStackRouterVite` → **`tanstackRouter`** |

[↑ Back to top](#table-of-contents)

---

<a id="part-2"></a>
## Part 2 — The Route Tree and the Component Tree

<a id="p2-beginner"></a>
### Beginner: the root route and `Outlet`

TanStack Router maps a URL to a *nested* component tree, not to a single component. The [route trees guide](https://tanstack.com/router/latest/docs/framework/react/routing/route-trees) puts it plainly: the router "uses a nested route tree to match up the URL with the correct component tree to render."

A URL like `/posts/42` does not render one component. It renders a stack:

```
  URL: /posts/42

  __root.tsx            ← always matched
    └─ posts.tsx        ← matched: /posts
         └─ posts.$postId.tsx   ← matched: /posts/42

  renders as:

  <Root>
    <nav/>
    <Outlet>              ← the root's Outlet renders the next match down
      <Posts>
        <ul>…list…</ul>
        <Outlet>          ← Posts' Outlet renders the next match down
          <Post postId="42"/>
        </Outlet>
      </Posts>
    </Outlet>
  </Root>
```

**`<Outlet/>` is the hole where children render.** A route that has children and does not render an `<Outlet/>` will match its children correctly, run their loaders, and then show you nothing — which is a genuinely confusing first bug because the devtools will cheerfully report the child as matched.

> **Wrong vs. right — the missing outlet.**
>
> ```tsx
> // ❌ WRONG — /posts/42 matches, the loader runs, the screen shows only the list
> export const Route = createFileRoute('/posts')({
>   component: () => <ul>{/* list */}</ul>,
> })
>
> // ✅ RIGHT
> export const Route = createFileRoute('/posts')({
>   component: () => (
>     <div>
>       <ul>{/* list */}</ul>
>       <Outlet />
>     </div>
>   ),
> })
> ```

[↑ Back to top](#table-of-contents)

---

<a id="p2-working"></a>
### Working Knowledge: layout routes, nesting, and `getRouteApi`

**Layout routes** are just routes that have children and render an `<Outlet/>`. There is no special syntax — `posts.tsx` becomes a layout the moment `posts.$postId.tsx` exists. From the [routing concepts guide](https://tanstack.com/router/latest/docs/framework/react/routing/routing-concepts): "Layout routes are used to wrap child routes with additional components and logic."

```tsx
// src/routes/posts.tsx
import { Outlet, createFileRoute, Link } from '@tanstack/react-router'

export const Route = createFileRoute('/posts')({
  component: PostsLayout,
})

function PostsLayout() {
  return (
    <div style={{ display: 'flex', gap: 24 }}>
      <aside>
        <Link to="/posts/$postId" params={{ postId: '1' }}>
          First post
        </Link>
      </aside>
      <main>
        <Outlet />
      </main>
    </div>
  )
}
```

**The `Route` export is your typed API for that route.** Every route file exports a `Route` object, and that object carries route-scoped hooks:

```tsx
// src/routes/posts.$postId.tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params }) => fetchPost(params.postId),
  component: PostComponent,
})

function PostComponent() {
  const { postId } = Route.useParams()      // typed: { postId: string }
  const post = Route.useLoaderData()        // typed: whatever fetchPost returns
  return <h2>{post.title} (#{postId})</h2>
}
```

Note that `Route.useParams()` is *strictly* typed to this route. There is no cast, no generic argument, no `as`.

**`getRouteApi` — the same typed hooks, without the import.** When a component lives outside the route file (in `-components/`, say), importing `Route` from the route file creates an awkward dependency and, worse, defeats code splitting — the [code splitting guide](https://tanstack.com/router/latest/docs/framework/react/guide/code-splitting) specifically notes that `getRouteApi` "provides type-safe access to route data across files without importing the route itself."

```tsx
// src/routes/-components/PostBody.tsx
import { getRouteApi } from '@tanstack/react-router'

const route = getRouteApi('/posts/$postId')   // string is autocompleted & checked

export function PostBody() {
  const post = route.useLoaderData()
  const { postId } = route.useParams()
  return <article data-id={postId}>{post.body}</article>
}
```

> **Wrong vs. right — reaching for loose hooks out of habit.**
>
> ```tsx
> // ❌ WRONG-ish — works, but everything is optional and you lose the types
> const params = useParams({ strict: false })
> const postId = params.postId as string      // the cast is the tell
>
> // ✅ RIGHT — strict by route id
> const { postId } = getRouteApi('/posts/$postId').useParams()
> ```
>
> `strict: false` is not a mistake in itself — it is the correct tool for genuinely route-agnostic components like a breadcrumb bar, where the [path params guide](https://tanstack.com/router/latest/docs/framework/react/guide/path-params) shows it as the supported escape hatch. It becomes a mistake the moment you add a cast to undo it.

**🧪 Try It — see the match stack.**

1. Add `posts.tsx` and `posts.$postId.tsx` as above.
2. Navigate to `/posts/42` and open the devtools panel (bottom corner).
3. Look at the **Matches** list.

**Expected output:** three entries, in order — `__root__`, `/posts`, `/posts/$postId` — each with its own status (`success`), its own params, and its own loader data. That list *is* the component nesting, and it is the single most useful debugging surface in this library.

[↑ Back to top](#table-of-contents)

---

<a id="p2-advanced"></a>
### Advanced: code-based routing, and when to reach for it

File-based routing is what the [route trees guide](https://tanstack.com/router/latest/docs/framework/react/routing/route-trees) calls "the preferred and recommended way" because it requires less code. But every file route is sugar for a code route, and understanding the desugaring explains several otherwise-mysterious behaviours.

The same two routes, written by hand:

```tsx
import {
  createRootRoute,
  createRoute,
  createRouter,
  Outlet,
} from '@tanstack/react-router'

const rootRoute = createRootRoute({
  component: () => <Outlet />,
})

const postsRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/posts',
  component: PostsLayout,
})

const postRoute = createRoute({
  getParentRoute: () => postsRoute,
  path: '$postId',
  loader: ({ params }) => fetchPost(params.postId),
  component: PostComponent,
})

const routeTree = rootRoute.addChildren([postsRoute.addChildren([postRoute])])

export const router = createRouter({ routeTree })
```

Three things become visible in this form that the file version hides:

1. **`getParentRoute` is a function, not a value.** It has to be — parent and child reference each other, and a lazy getter is how the circularity is resolved without a temporal dead zone. This is also why the inference can flow downward from parent to child (context, in particular — see [Part 7](./tanstack-router-data-loading.md#part-7)).
2. **Child `path` values are relative.** `'$postId'`, not `'/posts/$postId'`. The full path is composed by the tree. In file-based routing the plugin computes the full path and writes it into `createFileRoute('/posts/$postId')` for you.
3. **`addChildren` is where the tree type is built.** The giant union type you saw in the Part 1 Try It comes from these nested `addChildren` calls — the generated file is one big chain of them.

**When to use code-based routing:**

- A route tree that is genuinely computed at runtime (permissions-driven navigation, a plugin system).
- Library code that ships routes for a host app to mount.
- Tests that need a two-route tree without touching the filesystem ([Part 11](./tanstack-router-performance-internals.md#part-11)).

**When not to:** as a style preference for an app. You give up the filename-driven ergonomics and gain nothing. If you want file-based ergonomics with programmatic control, the intended tool is [virtual file routes](https://tanstack.com/router/latest/docs/routing/virtual-file-routes), not hand-written trees.

[↑ Back to top](#table-of-contents)

---

<a id="p2-mastery"></a>
### Mastery: what a `Route` object actually is

A route is four things bundled together, and separating them explains most of the router's behaviour:

```
  Route
   ├─ MATCHING CONFIG      path, params.parse/stringify, validateSearch
   │                       → used to decide *whether* this route matches a URL
   │
   ├─ LOADING CONFIG       beforeLoad, loaderDeps, loader, staleTime, gcTime
   │                       → used to produce a *match* with data, before render
   │
   ├─ RENDER CONFIG        component, errorComponent, pendingComponent,
   │                       notFoundComponent
   │                       → used *after* a match resolves
   │
   └─ TYPED API SURFACE    useParams, useSearch, useLoaderData, useRouteContext,
                           to, fullPath
                           → derived types, no runtime cost
```

The first two groups are what the code-splitting guide calls **critical route configuration**: "path parsing, search param validation, loaders, route context, and static data — everything needed to render a route and initiate data loading immediately." The third group is **non-critical**: "the route component, error component, pending component, and not-found component."

That split is not an arbitrary bundling decision. It is forced by the architecture: the router must be able to *match a URL and start fetching* without having downloaded the component. If the component chunk were needed to know whether a route matches, preloading on hover could never work, and every navigation would be a two-round-trip waterfall — download route code, then discover what data it needs, then fetch. This is the same waterfall problem that [React's own docs](../../react/react-effects-refs.md) describe for fetch-in-`useEffect`, moved up one level to the route.

Everything in [Part 10](./tanstack-router-performance-internals.md#part-10) — `.lazy.tsx`, `autoCodeSplitting`, `preload: 'intent'` — is downstream of this one split.

**One consequence worth knowing now:** `__root.tsx` cannot be code-split. The code-splitting guide states this directly. The root is always matched, so there is never a moment at which deferring it helps; keep it small and put nothing heavy in it.

[↑ Back to top](#table-of-contents)

---

<a id="p2-cheat"></a>
### Part 2 cheat sheet

| Concept | API | Note |
|---|---|---|
| Always-rendered root | `createRootRoute({...})` in `__root.tsx` | No path; cannot be code-split |
| Root with typed context | `createRootRouteWithContext<T>()({...})` | See [Part 7](./tanstack-router-data-loading.md#part-7) |
| File route | `createFileRoute('/full/path')({...})` | Path string is written by the plugin |
| Code route | `createRoute({ getParentRoute, path, … })` | `path` is relative to the parent |
| Assemble a code tree | `parent.addChildren([...])` | This is where the tree type is built |
| Child render slot | `<Outlet />` | Missing one = silently blank children |
| Route-scoped hooks | `Route.useParams()` / `.useSearch()` / `.useLoaderData()` / `.useRouteContext()` | Strictly typed |
| Same hooks, other files | `getRouteApi('/full/path')` | Preserves code splitting |
| Deliberately loose hooks | `useParams({ strict: false })` | All props optional; fine for generic UI, never pair with a cast |
| Critical vs. non-critical | matching + loading vs. components | Drives all code splitting |

[↑ Back to top](#table-of-contents)

---

<a id="next"></a>
## Where to go next

- **Next in sequence:** [Routing & Navigation (Parts 3–4)](./tanstack-router-routing-navigation.md) — every file convention, matching precedence, and type-safe links.
- **If you want the payoff early:** skip ahead to [Search Params (Part 5)](./tanstack-router-search-params.md); it only depends on Part 1.
- **Back to:** [Overview](./tanstack-router-mastery-guide.md)

### Quick self-check for Parts 1–2

1. You delete the `declare module` block. What still works, and what silently degrades?
2. Why must `getParentRoute` be a function rather than a direct reference?
3. `/posts/42` matches three routes. Which one renders where, and what connects them?
4. Which route config belongs to the "critical" half, and what would break if `component` were counted as critical too?
5. When is `useParams({ strict: false })` the right call, and what is the tell that you have reached for it wrongly?

[↑ Back to top](#table-of-contents)
