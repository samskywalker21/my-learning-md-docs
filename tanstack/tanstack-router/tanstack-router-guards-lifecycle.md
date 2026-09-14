# TanStack Router — Guards & Lifecycle (Parts 8–9)

> `beforeLoad` as subtree middleware, authenticated routes done properly, and every state a route can be in other than "rendered successfully".

**Written against:** `@tanstack/react-router` **1.170.x**, React 19, TypeScript 5.9+
**Last updated:** September 14, 2026
**Part of:** [TanStack Router Mastery Guide](./tanstack-router-mastery-guide.md) — read its [About This Document](./tanstack-router-mastery-guide.md#about-this-document) before editing this file.

---

## Table of Contents

- [Part 8 — `beforeLoad`, Guards and Auth](#part-8)
  - [Beginner: the navigation lifecycle](#p8-beginner)
  - [Working Knowledge: the authenticated-routes pattern](#p8-working)
  - [Advanced: the four caveats](#p8-advanced)
  - [Mastery: guards are not authorization](#p8-mastery)
  - [Part 8 cheat sheet](#p8-cheat)
- [Part 9 — Errors, Not-Found, Pending and Interruptions](#part-9)
  - [Beginner: `errorComponent` and `pendingComponent`](#p9-beginner)
  - [Working Knowledge: `notFound()` and its modes](#p9-working)
  - [Advanced: navigation blocking](#p9-advanced)
  - [Advanced: scroll restoration](#p9-scroll)
  - [Part 9 cheat sheet](#p9-cheat)
- [Where to go next](#next)

---

<a id="part-8"></a>
## Part 8 — `beforeLoad`, Guards and Auth

<a id="p8-beginner"></a>
### Beginner: the navigation lifecycle

Every navigation runs the same sequence. Knowing the order tells you where to put things.

```
  User clicks a <Link to="/admin/users/7">
          │
          ▼
  ① MATCH      Walk the route tree, produce the match stack:
               __root → _authed → /admin → /admin/users/$id
          │
          ▼
  ② VALIDATE   Run each match's validateSearch + params.parse
          │    (a throw here = error before anything else runs)
          ▼
  ③ beforeLoad Run PARENT-FIRST, down the stack.
          │    Each may: throw redirect() · throw notFound() · throw an error
          │              · return an object that extends context for descendants
          │    A throw stops the subtree — children never run.
          ▼
  ④ LOAD       Run loaderDeps then loader for every match, IN PARALLEL
          │    (pendingComponent shows if this exceeds pendingMs)
          ▼
  ⑤ RENDER     Nested components render together, top-down
```

Two properties of step ③ do most of the work in real apps:

- **It is parent-first and sequential down the stack.** The docs put it plainly: `beforeLoad` "executes *before child route guards*, acting as middleware for the entire subtree." A guard on `_authed.tsx` protects every descendant without any of them knowing it exists.
- **A throw prevents children from loading.** "Throwing errors in `beforeLoad` prevents child routes from loading." So an unauthenticated user never triggers a single loader inside the guarded subtree — no wasted requests, no 401s in the console.

Contrast with step ④, which is **parallel**. Loaders across the match stack all run at once, because by then the tree of matches is known and there is nothing to sequence.

[↑ Back to top](#table-of-contents)

---

<a id="p8-working"></a>
### Working Knowledge: the authenticated-routes pattern

The [authenticated routes guide](https://tanstack.com/router/latest/docs/framework/react/guide/authenticated-routes) describes a four-piece pattern. It is worth following exactly, because the obvious shortcuts each break in a specific way.

**Piece 1 — auth in the router context.**

```tsx
// src/routes/__root.tsx
interface MyRouterContext {
  auth: AuthState
}

export const Route = createRootRouteWithContext<MyRouterContext>()({
  component: RootComponent,
})
```

```tsx
// src/router.tsx
export const router = createRouter({
  routeTree,
  context: { auth: undefined! },   // ← filled in at render time; see piece 4
})
```

That `undefined!` is not a hack to be embarrassed about — it is the documented approach. The context type is required, but the real value cannot exist until React is rendering.

**Piece 2 — the guard, on a pathless layout route.**

```tsx
// src/routes/_authed.tsx
import { createFileRoute, redirect, Outlet } from '@tanstack/react-router'

export const Route = createFileRoute('/_authed')({
  beforeLoad: ({ context, location }) => {
    if (!context.auth.isAuthenticated) {
      throw redirect({
        to: '/login',
        search: { redirect: location.href },
      })
    }
  },
  component: () => <Outlet />,
})
```

A [pathless layout route](./tanstack-router-routing-navigation.md#p3-working) is exactly the right shape for this: it contributes nothing to the URL, so `_authed.admin.tsx` is `/admin` and not `/_authed/admin`, and every file you put under `_authed.` is guarded by construction. Adding a new protected page is a rename, not a code change.

**Piece 3 — the login route reads the redirect back.**

```tsx
export const Route = createFileRoute('/login')({
  validateSearch: z.object({
    redirect: z.string().optional().catch(undefined),
  }),
  component: LoginComponent,
})

function LoginComponent() {
  const { redirect } = Route.useSearch()
  const router = useRouter()
  const auth = useAuth()

  const onSubmit = async (creds: Credentials) => {
    await auth.login(creds)
    await router.invalidate()             // ← recompute context with the new auth
    router.history.push(redirect ?? '/')  // ← back where they were headed
  }
}
```

The guide shows `router.history.push(search.redirect)` for the final step. `router.invalidate()` before it is the piece people omit — see the caveats below.

**Piece 4 — inject the live auth state into the router.**

Hooks cannot be called outside components, so `createRouter()` cannot call `useAuth()`. The solution is a wrapper that reads the hook and passes the value down:

```tsx
function App() {
  const auth = useAuth()
  return <RouterProvider router={router} context={{ auth }} />
}

export function Root() {
  return (
    <AuthProvider>
      <App />
    </AuthProvider>
  )
}
```

`RouterProvider`'s `context` prop merges into the router's context on every render, so as `auth` changes in React state, the router sees the new value.

```
  <AuthProvider>            ← React state lives here
        │  useAuth()
        ▼
     <App/>                 ← reads the hook
        │  context={{ auth }}
        ▼
  <RouterProvider>          ← merges into router context each render
        │
        ▼
   beforeLoad({ context })  ← guards read the current value
```

[↑ Back to top](#table-of-contents)

---

<a id="p8-advanced"></a>
### Advanced: the four caveats

**① Context is not reactive — `invalidate()` is how you tell it.**

```tsx
// ❌ WRONG — login succeeds, React state updates, but the router's
//    already-resolved matches still carry the logged-out context
await auth.login(creds)
navigate({ to: '/admin' })     // guard still sees isAuthenticated: false

// ✅ RIGHT
await auth.login(creds)
await router.invalidate()      // recompute context + reload matches
navigate({ to: '/admin' })
```

The [router context guide](https://tanstack.com/router/latest/docs/framework/react/guide/router-context) documents `router.invalidate()` for exactly this: "when context state changes, call `router.invalidate()` to recompute context across all routes."

**② `try/catch` around a guard will swallow redirects.** `redirect()` works by throwing, so an over-broad catch turns a redirect into a silently ignored error. The guide's remedy is `isRedirect()`:

```tsx
import { isRedirect } from '@tanstack/react-router'

beforeLoad: async ({ context, location }) => {
  try {
    await context.auth.ensureSession()
  } catch (err) {
    if (isRedirect(err)) throw err          // ← let it through
    throw redirect({ to: '/login', search: { redirect: location.href } })
  }
}
```

**③ Guard at the layout, not at each leaf.**

```tsx
// ❌ WRONG — five files, five chances to forget the sixth
// admin.users.tsx, admin.billing.tsx, admin.settings.tsx … each with its own check

// ✅ RIGHT — one guard, inherited by construction
// _authed.tsx holds the beforeLoad; _authed.admin.users.tsx et al. hold nothing
```

**④ A guard is not an authorization boundary.** This has its own section below, because it is the one that actually matters.

**🧪 Try It — see the subtree short-circuit.**

1. Put `console.log('GUARD')` in `_authed.tsx`'s `beforeLoad` and `console.log('LOADER')` in `_authed.admin.tsx`'s loader.
2. While logged out, navigate to `/admin`.

**Expected output:** `GUARD` logs, `LOADER` does not, and the URL becomes `/login?redirect=%2Fadmin`. Then log in, call `router.invalidate()`, and navigate again:

```
GUARD
LOADER
```

If you skip the `invalidate()` call, you will see `GUARD` and another bounce to `/login` — which is caveat ① reproduced on demand.

[↑ Back to top](#table-of-contents)

---

<a id="p8-mastery"></a>
### Mastery: guards are not authorization

The authenticated-routes guide states this outright: **"A route guard is not a data authorization boundary."** Its reasoning is that server functions and endpoints "must authorize independently since they're callable outside the route context."

This is worth sitting with, because the pattern above is so ergonomic that it *feels* like security. It is not. Everything in a `beforeLoad` runs in the user's browser, in code the user can read, against state the user can modify.

```
  What a guard actually does              What it does NOT do
  ────────────────────────────            ─────────────────────────────────
  ✔ Stops the wrong UI rendering          ✘ Stop anyone calling your API
  ✔ Avoids loaders that would 401         ✘ Hide the code in the JS bundle
  ✔ Sends the user somewhere sensible     ✘ Hide data the loader already fetched
  ✔ Improves perceived correctness        ✘ Survive devtools, a patched bundle,
                                            or a plain curl to your endpoint
```

The mental model that keeps you honest:

> **A route guard is a UX affordance. Authorization is a server property.**

Two concrete rules that follow:

1. **Every endpoint re-checks.** `GET /api/admin/users` must verify the session itself. If the only thing standing between a user and that data is `_authed.tsx`, you do not have access control — you have a hidden link.
2. **Never treat "the guard passed" as proof of anything downstream.** A loader inside `_authed` should still handle a 403 from the API, because the guard's view of auth can be stale (a session expiring mid-session is the normal case, not the exotic one).

**The related code-splitting nuance.** Route components are code-split, so `/admin`'s component is not in the initial bundle — but it *is* in a chunk any visitor can request. Code splitting is a performance feature that happens to obscure things; it is not a confidentiality feature. Do not put secrets in a component because "it's lazy-loaded."

[↑ Back to top](#table-of-contents)

---

<a id="p8-cheat"></a>
### Part 8 cheat sheet

| Task | API |
|---|---|
| Subtree middleware | `beforeLoad: ({ context, location, params, search }) => …` |
| Redirect out | `throw redirect({ to, search: { redirect: location.href } })` |
| Let a redirect escape a catch | `if (isRedirect(err)) throw err` |
| Extend context downward | `beforeLoad: () => ({ extra })` |
| Guard a whole area | Put `beforeLoad` on a pathless layout `_authed.tsx` |
| Seed an unavailable context value | `context: { auth: undefined! }` at `createRouter` |
| Inject live values | `<RouterProvider router={router} context={{ auth }} />` |
| After auth changes | `await router.invalidate()` |
| Return to the original target | `router.history.push(search.redirect ?? '/')` |

**Execution order:** match → validate → `beforeLoad` (parent-first, sequential) → loaders (parallel) → render.

[↑ Back to top](#table-of-contents)

---

<a id="part-9"></a>
## Part 9 — Errors, Not-Found, Pending and Interruptions

<a id="p9-beginner"></a>
### Beginner: `errorComponent` and `pendingComponent`

A route match has four possible outcomes, and a component slot for each:

| Slot | Shows when |
|---|---|
| `component` | Everything resolved |
| `pendingComponent` | Loading has exceeded `pendingMs` |
| `errorComponent` | Something threw |
| `notFoundComponent` | `notFound()` was thrown |

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params }) => fetchPost(params.postId),
  component: PostComponent,
  pendingComponent: () => <p>Loading post…</p>,
  errorComponent: ({ error, reset }) => (
    <div role="alert">
      <p>Could not load this post: {error.message}</p>
      <button onClick={reset}>Retry</button>
    </div>
  ),
})
```

`errorComponent` receives the thrown error and a `reset` function that retries the match. Set router-wide fallbacks with `defaultErrorComponent` and `defaultPendingComponent` so you only override where a route deserves bespoke treatment.

**Why pending UI is delayed rather than immediate.** Showing a spinner for a 60 ms load produces a flash that reads as jank. The router waits `pendingMs` before showing `pendingComponent`, and once shown keeps it for at least `pendingMinMs` so it does not flicker away instantly. Tune these per route for known-slow pages; leave them alone otherwise.

[↑ Back to top](#table-of-contents)

---

<a id="p9-working"></a>
### Working Knowledge: `notFound()` and its modes

A 404 is not an error — it is a distinct outcome with its own component. Throw `notFound()` from a loader or `beforeLoad` ([not-found errors guide](https://tanstack.com/router/latest/docs/framework/react/guide/not-found-errors)):

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params: { postId } }) => {
    const post = await getPost(postId)
    if (!post) throw notFound()
    return { post }
  },
})
```

Handle it at whichever level makes sense:

```tsx
export const Route = createFileRoute('/settings')({
  component: () => <div><p>Settings page</p><Outlet /></div>,
  notFoundComponent: () => <p>This setting page doesn't exist!</p>,
})
```

…or globally:

```tsx
const router = createRouter({
  defaultNotFoundComponent: () => (
    <div>
      <p>Not found!</p>
      <Link to="/">Go home</Link>
    </div>
  ),
})
```

**The two modes.** `notFoundMode` decides where a not-found surfaces:

```
  notFoundMode: 'fuzzy'  (default)
  ──────────────────────────────────
  The router "intelligently finds the closest matching suitable route"
  with a notFoundComponent, and renders it THERE.

    /settings/nope  →  __root
                         └─ /settings         ← still rendered: nav, sidebar
                              └─ notFoundComponent ("no such setting")

  The user keeps their context and can navigate onward.


  notFoundMode: 'root'
  ──────────────────────────────────
  Every not-found bypasses intermediates and renders the ROOT's
  notFoundComponent.

    /settings/nope  →  __root
                         └─ notFoundComponent (a full-page 404)
```

Fuzzy is the better default for app-like UIs — a bad sub-path should not throw away the whole shell. Root is right for content sites where a 404 is a genuine dead end.

**Targeting a specific level** overrides the search:

```tsx
throw notFound({ routeId: '/_pathlessLayout' })
throw notFound({ routeId: rootRouteId })
```

**Data-less vs. data-full.** By default, a not-found component gets no loader data — which is correct, since the load failed. If you have partial data worth showing:

```tsx
throw notFound({ data: someIncompleteLoaderData })
```

Otherwise, inside `notFoundComponent`, the docs direct you to route context rather than loader data: `Route.useParams()`, `Route.useSearch()`, `Route.useRouteContext()` all work.

> **Wrong vs. right — 404 as an error.**
>
> ```tsx
> // ❌ WRONG — the user sees "Something went wrong. Retry?" for a deleted post.
> //    Retrying will fail identically, forever.
> loader: async ({ params }) => {
>   const post = await getPost(params.postId)
>   if (!post) throw new Error('Post not found')
>   return post
> }
>
> // ✅ RIGHT — a distinct outcome, with a distinct UI and a way out
> loader: async ({ params }) => {
>   const post = await getPost(params.postId)
>   if (!post) throw notFound()
>   return post
> }
> ```
>
> The distinction matters to users: an error means "try again", a 404 means "this is gone, go elsewhere." Offering a retry button for a deleted resource is a small cruelty.

**🧪 Try It — watch fuzzy mode pick a level.**

1. Give `/settings` a `notFoundComponent` and leave the router's `notFoundMode` at its default.
2. Visit `/settings/does-not-exist`.

**Expected output:** the settings shell (nav, sidebar) is still on screen, with the settings-specific not-found message inside it. Now set `notFoundMode: 'root'` and reload: the entire shell is replaced by the root 404.

[↑ Back to top](#table-of-contents)

---

<a id="p9-advanced"></a>
### Advanced: navigation blocking

`useBlocker` stops navigation when leaving would lose work — a half-written form, an in-flight payment ([navigation blocking guide](https://tanstack.com/router/latest/docs/framework/react/guide/navigation-blocking)).

**Simple form** — return `true` to block:

```tsx
function EditPost() {
  const [isDirty, setIsDirty] = React.useState(false)

  useBlocker({
    shouldBlockFn: () => isDirty,
    enableBeforeUnload: () => isDirty,
  })
}
```

`shouldBlockFn` receives an object with `current` and `next` location data, so you can compare routes, params and search — useful for allowing navigation *within* a wizard while blocking exits from it:

```tsx
useBlocker({
  shouldBlockFn: ({ current, next }) => {
    if (next.routeId.startsWith('/checkout')) return false   // moving within checkout
    return hasUnsavedChanges
  },
})
```

**With a resolver** — for custom UI rather than the browser's dialog. Setting `withResolver: true` returns `{ proceed, reset, status }`, where `status` becomes `'blocked'` when navigation is held:

```tsx
function EditPost() {
  const { proceed, reset, status } = useBlocker({
    shouldBlockFn: () => isDirty,
    withResolver: true,
  })

  return (
    <>
      <form>…</form>
      {status === 'blocked' && (
        <Dialog>
          <p>You have unsaved changes.</p>
          <button onClick={proceed}>Leave anyway</button>
          <button onClick={reset}>Stay</button>
        </Dialog>
      )}
    </>
  )
}
```

There is a `<Block>` component with the same options and a render-prop API — `{({ status, proceed, reset }) => …}` — if you prefer that shape.

**`enableBeforeUnload`** is separate and important. In-app navigation is blocked by the router; **closing the tab or hitting reload is blocked by the browser**, via `beforeunload`. Pass a boolean or a function to register that handler conditionally.

> **Wrong vs. right — the always-on `beforeunload`.**
>
> ```tsx
> // ❌ WRONG — the browser's "Leave site?" dialog on every reload, even
> //    with a pristine form. Users learn to dismiss it reflexively.
> useBlocker({ shouldBlockFn: () => isDirty, enableBeforeUnload: true })
>
> // ✅ RIGHT — register the handler only when there is something to lose
> useBlocker({ shouldBlockFn: () => isDirty, enableBeforeUnload: () => isDirty })
> ```

[↑ Back to top](#table-of-contents)

---

<a id="p9-scroll"></a>
### Advanced: scroll restoration

Browsers restore scroll positions natively, but SPAs defeat it: they navigate with `history.pushState`, render asynchronously, and often scroll a nested container rather than the window. The router's answer is to watch scroll events, cache positions, and restore them before the next paint ([scroll restoration guide](https://tanstack.com/router/latest/docs/framework/react/guide/scroll-restoration)).

**Turn it on with one option:**

```tsx
const router = createRouter({
  scrollRestoration: true,
})
```

That handles the window and body.

> **Outdated convention, named explicitly:** you will find tutorials rendering a `<ScrollRestoration />` component in `__root.tsx`. The guide states: *"The `<ScrollRestoration />` component still works, but has been deprecated."* Use the router option. If you inherit a codebase with the component, replacing it is a two-line change.

**Custom cache keys.** By default positions are keyed on `location.state.__TSR_key`, a unique id per history entry — so going back to a list restores where you were, but visiting the same list fresh starts at the top. To key on pathname instead:

```tsx
const router = createRouter({
  getScrollRestorationKey: (location) => location.pathname,
})
```

Or selectively, which is usually what you want:

```tsx
const router = createRouter({
  getScrollRestorationKey: (location) => {
    const paths = ['/', '/chat']
    return paths.includes(location.pathname)
      ? location.pathname
      : location.state.__TSR_key!
  },
})
```

The distinction is real: a chat view should return to the same scroll position however you arrived; a search results page should not, because the results differ.

**Virtualised lists** need manual handling, since their scroll container is not the window and their content does not exist until measured. `useElementScrollRestoration` gives you the cached offset to seed the virtualizer:

```tsx
function Component() {
  const scrollRestorationId = 'myVirtualizedContent'
  const scrollEntry = useElementScrollRestoration({ id: scrollRestorationId })
  const virtualizerParentRef = React.useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: 10000,
    getScrollElement: () => virtualizerParentRef.current,
    estimateSize: () => 100,
    initialOffset: scrollEntry?.scrollY,
  })

  return (
    <div
      ref={virtualizerParentRef}
      data-scroll-restoration-id={scrollRestorationId}
      className="flex-1 border rounded-lg overflow-auto relative"
    >
      …
    </div>
  )
}
```

The `data-scroll-restoration-id` attribute is what pairs the DOM element with the cached entry — omit it and the hook has nothing to track.

**Opting out per navigation** with `resetScroll: false`, accepted by `<Link>`, `navigate()` and `redirect()`:

```tsx
<Link resetScroll={false} />
navigate({ resetScroll: false })
redirect({ resetScroll: false })
```

Use it for tab strips and filter changes — navigations that change content in place and should not throw the user back to the top.

[↑ Back to top](#table-of-contents)

---

<a id="p9-cheat"></a>
### Part 9 cheat sheet

| Situation | API |
|---|---|
| Loading UI | `pendingComponent` (+ `pendingMs`, `pendingMinMs`) |
| Error UI | `errorComponent: ({ error, reset }) => …` |
| App-wide fallbacks | `defaultPendingComponent`, `defaultErrorComponent` |
| Resource missing | `throw notFound()` from a loader or `beforeLoad` |
| 404 UI | `notFoundComponent` per route, `defaultNotFoundComponent` globally |
| Where a 404 renders | `notFoundMode: 'fuzzy'` (default, nearest handler) or `'root'` |
| Target a level | `throw notFound({ routeId })` / `rootRouteId` |
| 404 with partial data | `throw notFound({ data })` |
| Block navigation | `useBlocker({ shouldBlockFn })` or `<Block>` |
| Custom block UI | `withResolver: true` → `{ proceed, reset, status }` |
| Block tab close / reload | `enableBeforeUnload: () => isDirty` |
| Scroll restoration | `scrollRestoration: true` (**not** `<ScrollRestoration/>`) |
| Custom scroll key | `getScrollRestorationKey: (location) => …` |
| Virtualised list scroll | `useElementScrollRestoration({ id })` + `data-scroll-restoration-id` |
| Skip scroll reset | `resetScroll: false` |

**Three rules to carry forward**

1. A missing resource is `notFound()`, not `new Error()`. Users need different affordances.
2. `enableBeforeUnload` should be conditional, always.
3. `scrollRestoration: true` replaced the `<ScrollRestoration/>` component.

[↑ Back to top](#table-of-contents)

---

<a id="next"></a>
## Where to go next

- **Next in sequence:** [Performance & Internals (Parts 10–13)](./tanstack-router-performance-internals.md).
- **Related:** `beforeLoad` as a context extender is in [Part 7](./tanstack-router-data-loading.md#p7-working); pathless layouts are in [Part 3](./tanstack-router-routing-navigation.md#p3-working).
- **Back to:** [Overview](./tanstack-router-mastery-guide.md)

### Quick self-check for Parts 8–9

1. Put these in order and say which are parallel: loaders, `validateSearch`, `beforeLoad`, matching, render.
2. Why does a `beforeLoad` throw on a parent mean zero network requests from its children?
3. What is `context: { auth: undefined! }` for, and what fills it in?
4. Which two things must happen after a successful login, besides navigating?
5. Why can a `try/catch` in `beforeLoad` break your redirects, and what is the fix?
6. Give two things a route guard genuinely provides, and two it definitely does not.
7. When should a 404 render inside the existing layout, and which option controls that?
8. Why is `enableBeforeUnload: true` worse than `enableBeforeUnload: () => isDirty`?
9. A chat view and a search results page want different scroll behaviour. Which option distinguishes them?

[↑ Back to top](#table-of-contents)
