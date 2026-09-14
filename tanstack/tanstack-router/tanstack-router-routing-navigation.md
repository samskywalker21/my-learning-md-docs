# TanStack Router — Routing & Navigation (Parts 3–4)

> Every file convention, the matching rules that make route order irrelevant, and links that cannot point at a route you do not have.

**Written against:** `@tanstack/react-router` **1.170.x**, `@tanstack/router-plugin` 1.170.x, React 19, TypeScript 5.9+
**Last updated:** September 14, 2026
**Part of:** [TanStack Router Mastery Guide](./tanstack-router-mastery-guide.md) — read its [About This Document](./tanstack-router-mastery-guide.md#about-this-document) before editing this file.

---

## Table of Contents

- [Part 3 — Route Trees and Matching](#part-3)
  - [Beginner: static, index, and dynamic routes](#p3-beginner)
  - [Working Knowledge: the full convention set](#p3-working)
  - [Advanced: matching precedence and path params](#p3-advanced)
  - [Mastery: `params.parse`, priority, and custom segment grammar](#p3-mastery)
  - [Part 3 cheat sheet](#p3-cheat)
- [Part 4 — Navigation](#part-4)
  - [Beginner: `<Link>`](#p4-beginner)
  - [Working Knowledge: active state, relative navigation, imperative navigation](#p4-working)
  - [Advanced: `createLink`, `linkOptions`, and redirects](#p4-advanced)
  - [Mastery: route masking](#p4-mastery)
  - [Part 4 cheat sheet](#p4-cheat)
- [Where to go next](#next)

---

<a id="part-3"></a>
## Part 3 — Route Trees and Matching

<a id="p3-beginner"></a>
### Beginner: static, index, and dynamic routes

Three conventions cover most of an app. All examples below come from the [routing concepts guide](https://tanstack.com/router/latest/docs/framework/react/routing/routing-concepts).

**Static route** — `about.tsx` → `/about`:

```tsx
export const Route = createFileRoute('/about')({
  component: AboutComponent,
})
```

**Index route** — `posts.index.tsx` → `/posts/`. The docs define these precisely: "Index routes specifically target their parent route when it is matched exactly and no child route is matched." That is the whole rule, and it is why `/posts` shows the index while `/posts/42` does not.

```tsx
export const Route = createFileRoute('/posts/')({
  component: PostsIndexComponent,
})
```

**Dynamic segment** — `posts/$postId.tsx` → `/posts/$postId`. A `$`-prefixed segment captures one URL segment into `params`:

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params }) => fetchPost(params.postId),
  component: PostComponent,
})

function PostComponent() {
  const { postId } = Route.useParams()
  return <div>Post ID: {postId}</div>
}
```

**Flat or nested — your choice, same result.** The [file-based routing guide](https://tanstack.com/router/latest/docs/framework/react/routing/file-based-routing) introduces the `.` character in a filename as a nesting separator, so these two layouts are identical:

```
  FLAT                          NESTED
  routes/                       routes/
    posts.tsx                     posts.tsx
    posts.index.tsx               posts/
    posts.$postId.tsx               index.tsx
                                    $postId.tsx
```

The docs note that "both flat and directory routes can be mixed together… where it makes sense." A pragmatic rule: stay flat until a route has more than about three children or needs colocated files, then promote it to a directory. Flat files sort alphabetically in your editor sidebar, which reads well precisely because the `.` separator makes siblings adjacent.

[↑ Back to top](#table-of-contents)

---

<a id="p3-working"></a>
### Working Knowledge: the full convention set

This is the complete table. Every entry is sourced from the [routing concepts guide](https://tanstack.com/router/latest/docs/framework/react/routing/routing-concepts); the ones worth dwelling on are explained below it.

| File | Resulting path | What it does |
|---|---|---|
| `__root.tsx` | — | Always matched, wraps everything |
| `about.tsx` | `/about` | Static |
| `posts.index.tsx` | `/posts/` | Index — parent matched *exactly* |
| `posts.$postId.tsx` | `/posts/$postId` | Dynamic segment |
| `posts.{-$category}.tsx` | `/posts/{-$category}` | **Optional** segment |
| `files.$.tsx` | `/files/$` | Splat — captures the remainder as `_splat` |
| `app.tsx` + `app.dashboard.tsx` | `/app`, `/app/dashboard` | Layout + child |
| `_pathlessLayout.tsx` | *(none)* | **Pathless layout** — wraps, adds no URL segment |
| `posts_.$postId.edit.tsx` | `/posts/$postId/edit` | **Non-nested** — escapes the parent layout |
| `(app)/dashboard.tsx` | `/dashboard` | **Group directory** — organisation only |
| `-posts-table.tsx`, `-components/` | — | **Excluded** from route generation |

**The two underscores do opposite things.** This is the single most confusing pair in the convention set, and it is worth memorising as a contrast rather than individually.

```
  _authed.tsx                 LEADING underscore  → pathless layout
  ────────────                "wrap my children, but contribute nothing to the URL"

    _authed.tsx               (no path of its own)
      └─ _authed.admin.tsx    → /admin     ← note: NOT /_authed/admin
      └─ _authed.billing.tsx  → /billing

  Use for: a shared guard or chrome across unrelated URLs.


  posts_.$postId.edit.tsx     TRAILING underscore → non-nested route
  ───────────────────────     "give me the URL nesting, but not the layout nesting"

    posts.tsx                 → /posts        renders sidebar + <Outlet/>
    posts.$postId.tsx         → /posts/42     renders INSIDE posts.tsx
    posts_.$postId.edit.tsx   → /posts/42/edit renders OUTSIDE posts.tsx

  Use for: a full-screen editor that shares a URL prefix but not the chrome.
```

The docs describe the leading form as prefixed "with an underscore to denote that they are 'pathless'", and the trailing form as suffixing "a parent file route segment with a `_`… to un-nest a route from its parents."

**Group directories `(name)/`** are "purely organisational" — the parentheses vanish from the URL entirely. `(marketing)/pricing.tsx` is `/pricing`. Use them to cluster files without affecting either URL or layout nesting. Note the difference from a pathless layout: a group directory has **no route object and no component**, so it cannot hold a guard or shared UI. If you need shared behaviour, you need `_name.tsx`, not `(name)/`.

**Excluded files `-name`** let you colocate non-route code inside `routes/`. The `-` prefix is the plugin's `routeFileIgnorePrefix`, and it is configurable ([file-based routing API](https://tanstack.com/router/latest/docs/api/file-based-routing)) but there is no good reason to change it.

```tsx
// src/routes/posts.tsx
import { PostsTable } from './-posts-table'

export const Route = createFileRoute('/posts')({
  component: PostComponent,
})
```

> **Wrong vs. right — colocating a component.**
>
> ```
> ❌ WRONG                          ✅ RIGHT
> routes/                           routes/
>   posts.tsx                         posts.tsx
>   PostsTable.tsx   ← becomes        -components/
>                      /PostsTable!     PostsTable.tsx
> ```
>
> Without the `-` prefix, `PostsTable.tsx` is a route. The plugin will helpfully write `createFileRoute('/PostsTable')` into it, at which point you have a mystery route and a confusing diff.

**🧪 Try It — watch the two underscores diverge.**

Create these four files, each rendering its own name plus `<Outlet/>` where relevant:

```
routes/posts.tsx              → <div>LAYOUT<Outlet/></div>
routes/posts.$postId.tsx      → <div>VIEW {postId}</div>
routes/posts_.$postId.edit.tsx→ <div>EDIT {postId}</div>
```

Visit `/posts/1` then `/posts/1/edit`.

**Expected output:**

```
/posts/1        →  LAYOUT VIEW 1
/posts/1/edit   →  EDIT 1            ← "LAYOUT" is absent
```

Now rename `posts_.$postId.edit.tsx` to `posts.$postId.edit.tsx` (drop the trailing underscore) and reload:

```
/posts/1/edit   →  LAYOUT EDIT 1     ← layout is back
```

One character, entirely different component tree.

[↑ Back to top](#table-of-contents)

---

<a id="p3-advanced"></a>
### Advanced: matching precedence and path params

**Route order in your filesystem does not matter.** The router sorts routes by specificity before matching. From the [route matching guide](https://tanstack.com/router/latest/docs/framework/react/routing/route-matching), the hierarchy is:

1. **Index routes**
2. **Static routes** — most specific to least specific
3. **Dynamic routes** — longest to shortest
4. **Splat / wildcard routes**

The guide's own example, unsorted:

```
Root
  - blog
    - $postId
    - /
    - new
  - /
  - *
  - about
  - about/us
```

…becomes, after sorting:

```
Root
  - /
  - about/us
  - about
  - blog
    - /
    - new
    - $postId
  - *
```

And the walkthroughs:

- **`/blog`** — skips the root index and the static routes, enters `blog`, and the index route under it matches.
- **`/blog/my-post`** — enters `blog`; the index and `new` both fail; `$postId` captures the segment.
- **`/not-a-route`** — every index, static and dynamic route fails; the wildcard at the end is the final fallback.

The practical consequence: **you never need to think about declaration order.** `/posts/new` beats `/posts/$postId` because static beats dynamic, full stop — no matter which file was created first, no matter how the generated tree happens to be laid out. If you have used routers where ordering was load-bearing, this is the habit to unlearn.

> **A source note.** The docs describe this as sorting a tree; the [engineering blog on the matching rewrite](https://tanstack.com/blog/tanstack-router-route-matching-tree-rewrite) explains that the current implementation parses the tree into a **segment trie** and traverses it rather than walking a sorted flat list. The observable precedence is the same; the mechanism is covered in [Part 13](./tanstack-router-performance-internals.md#part-13).

**Path params in depth** ([path params guide](https://tanstack.com/router/latest/docs/framework/react/guide/path-params)).

A `$param` matches a single segment, stopping at the next `/`. Access it in a loader via `params`, and in a component via `Route.useParams()`:

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => fetchPost(params.postId),
})

function PostComponent() {
  const { postId } = Route.useParams()
  return <div>Post {postId}</div>
}
```

**Splat routes** capture everything remaining, under the key `_splat`:

```tsx
export const Route = createFileRoute('/files/$')({
  component: FileComponent,
})

function FileComponent() {
  const { _splat } = Route.useParams()
  return <div>Path: {_splat}</div>   // "my-drive/docs/foo.txt"
}
```

**Optional params** use `{-$param}` and match with *or* without the segment:

```tsx
export const Route = createFileRoute('/posts/{-$category}')({
  component: PostsComponent,
})

function PostsComponent() {
  const { category } = Route.useParams()
  return <div>{category ? `Posts in ${category}` : 'All Posts'}</div>
}
```

`/posts` and `/posts/tech` both match; `category` is `undefined` in the first case. This removes the old two-file workaround (`posts.index.tsx` plus `posts.$category.tsx` with duplicated logic) — if you see that pattern in an existing codebase, it predates optional params.

**Prefix and suffix segments** wrap the param in braces so fixed text can sit alongside it:

```tsx
createFileRoute('/posts/post-{$postId}')      // /posts/post-42
createFileRoute('/files/{$fileName}.txt')     // /files/report.txt
createFileRoute('/users/user-{$userId}.json') // /users/user-7.json
```

These compose with splats — the guide's example route `/on-disk/storage-{$postId}/$` yields both `postId` and `_splat`.

> **Wrong vs. right — params are strings unless you say otherwise.**
>
> ```tsx
> // ❌ WRONG — postId is a string; this is "42" - 1 territory
> loader: ({ params }) => fetchPost(params.postId + 1)
>
> // ✅ RIGHT — declare the coercion once, on the route
> export const Route = createFileRoute('/posts/$postId')({
>   params: {
>     parse: ({ postId }) => ({ postId: Number(postId) }),
>     stringify: ({ postId }) => ({ postId: String(postId) }),
>   },
>   loader: ({ params }) => fetchPost(params.postId + 1),  // number, typed
> })
> ```
>
> Once `params.parse` is declared, `Route.useParams()` returns `{ postId: number }` **and** `<Link params={{ postId: 42 }}>` requires a number. The coercion and its type live in one place.

[↑ Back to top](#table-of-contents)

---

<a id="p3-mastery"></a>
### Mastery: `params.parse`, priority, and custom segment grammar

`params.parse` does more than coerce. It can **reject a match**, which turns a dynamic segment into a validated one.

```tsx
export const Route = createFileRoute('/posts/$postId')({
  params: {
    priority: 10,
    parse: ({ postId }) => {
      if (!/^\d+$/.test(postId)) return false   // ← not a match; keep searching
      return { postId: Number(postId) }
    },
    stringify: ({ postId }) => ({ postId: String(postId) }),
  },
})
```

Returning `false` means "this route does not match after all" and routing continues to the next candidate. That lets two dynamic routes coexist on the same segment shape — `/posts/42` (numeric id) and `/posts/my-slug` (slug) can be genuinely different routes with different loaders, disambiguated by the parse function rather than by the URL grammar.

`priority` orders *competing candidates that both use `params.parse`* — higher values are attempted first. The important constraint, per the docs: `priority` **does not override normal route specificity**. Static routes still beat dynamic ones, dynamic still beat optional and wildcard. `priority` is a tiebreaker inside one tier, not a way to jump tiers.

> **The trap this creates.** If you use `parse` to reject, you must make sure *something* eventually matches. A numeric-only `$postId` with no slug sibling and no splat means `/posts/my-slug` falls through to the nearest not-found boundary — which is correct behaviour, but surprising if you expected the route to match and the loader to 404. Decide deliberately between "reject at the route level" (clean URLs, handled by [not-found](./tanstack-router-guards-lifecycle.md#part-9)) and "match and validate in the loader" (more control over the error UI).

**Allowed characters in path params.** By default the router percent-encodes reserved characters. To let some through literally, set `pathParamsAllowedCharacters` on the router:

```tsx
const router = createRouter({
  pathParamsAllowedCharacters: ['@', '+'],
})
```

The accepted set per the docs is `;` `:` `@` `&` `=` `+` `$` `,`. Reach for this when your ids are naturally email-like or contain `+` (common with scoped package names or tag filters), where the encoded form is technically correct but makes URLs unreadable.

[↑ Back to top](#table-of-contents)

---

<a id="p3-cheat"></a>
### Part 3 cheat sheet

| Convention | Example | Path |
|---|---|---|
| Static | `about.tsx` | `/about` |
| Nested via dot | `posts.comments.tsx` | `/posts/comments` |
| Index | `posts.index.tsx` | `/posts/` |
| Dynamic | `posts.$postId.tsx` | `/posts/$postId` |
| Optional | `posts.{-$category}.tsx` | `/posts` **and** `/posts/tech` |
| Prefix / suffix | `files.{$name}.txt.tsx` | `/files/report.txt` |
| Splat | `files.$.tsx` | `/files/**` → `_splat` |
| Pathless layout | `_authed.tsx` | *(no segment)* |
| Non-nested | `posts_.$postId.edit.tsx` | `/posts/$postId/edit`, no parent layout |
| Group dir | `(app)/dash.tsx` | `/dash` |
| Excluded | `-utils.ts` | *(not a route)* |

| Matching precedence | 1. Index → 2. Static (most→least specific) → 3. Dynamic (longest→shortest) → 4. Splat |
|---|---|

| Param API | Purpose |
|---|---|
| `Route.useParams()` | Typed params for this route |
| `useParams({ strict: false })` | Optional params, any route |
| `params.parse` | Coerce types; return `false` to reject the match |
| `params.stringify` | Inverse, used when building URLs |
| `params.priority` | Tiebreak between `parse`-using candidates only |
| `_splat` | The splat capture key |
| `pathParamsAllowedCharacters` | Router option; unescape chosen reserved chars |

[↑ Back to top](#table-of-contents)

---

<a id="part-4"></a>
## Part 4 — Navigation

<a id="p4-beginner"></a>
### Beginner: `<Link>`

```tsx
import { Link } from '@tanstack/react-router'

<Link to="/about">About</Link>

<Link to="/blog/post/$postId" params={{ postId: 'my-first-blog-post' }}>
  Blog Post
</Link>
```

The [navigation guide](https://tanstack.com/router/latest/docs/framework/react/guide/navigation) puts it this way: type checking "validates required parameters at compile time based on your route definition's dynamic segments." Omit `params` on a route that needs them and you get a compile error — not a URL containing a literal `$postId`.

You can also reference a route object instead of a string literal:

```tsx
import { Route as aboutRoute } from './routes/about.tsx'

<Link to={aboutRoute.to}>About</Link>
```

Both are equally type-safe. Prefer the literal — it is shorter, it autocompletes, and it does not create an import edge between unrelated route files.

[↑ Back to top](#table-of-contents)

---

<a id="p4-working"></a>
### Working Knowledge: active state, relative navigation, imperative navigation

**The `<Link>` prop surface**, grouped by what it does:

| Group | Props |
|---|---|
| Destination | `to`, `from`, `params`, `search`, `hash` |
| History & scroll | `replace`, `resetScroll`, `mask` |
| Active styling | `activeProps`, `inactiveProps`, `activeOptions` |
| Performance | `preload` (`false \| 'intent' \| 'viewport' \| 'render'`), `preloadDelay` |

**Active links** are styled by handing over props rather than a class-name callback:

```tsx
<Link
  to="/posts"
  activeProps={{ style: { fontWeight: 'bold' }, 'aria-current': 'page' }}
  inactiveProps={{ style: { opacity: 0.7 } }}
  activeOptions={{ exact: true }}
>
  Posts
</Link>
```

`activeOptions` is the one to know. By default `/posts` is considered active while you are on `/posts/42` — usually what you want for a nav bar, and usually *not* what you want for a tab strip.

> **Wrong vs. right — the "everything is active" bug.**
>
> ```tsx
> // ❌ WRONG — on /posts/42, BOTH tabs render as active
> <Link to="/posts">All</Link>
> <Link to="/posts/$postId" params={{ postId }}>Detail</Link>
>
> // ✅ RIGHT — the parent link opts into exact matching
> <Link to="/posts" activeOptions={{ exact: true }}>All</Link>
> <Link to="/posts/$postId" params={{ postId }}>Detail</Link>
> ```

**Relative navigation.** The guide frames all navigation as relative: you navigate *from* one route *to* another, and `from` establishes the origin. Without it, **the router assumes you are navigating from root `/`**.

```tsx
const postIdRoute = createRoute({ path: '/blog/post/$postId' })

<Link from={postIdRoute.fullPath} to="../categories">
  Categories
</Link>
```

Two special targets:

```tsx
<Link to=".">Reload current route</Link>
<Link to="..">Navigate to parent</Link>
```

`from` also sharpens the types: it tells TypeScript which params and search params are already in scope, which is what makes `search={(prev) => ...}` updaters typed. Inside a route file, `from={Route.fullPath}` is the idiomatic value.

**Imperative navigation** with `useNavigate`:

```tsx
function Component() {
  const navigate = useNavigate({ from: '/posts/$postId' })

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    const response = await fetch('/posts', {
      method: 'POST',
      body: JSON.stringify({ title: 'My First Post' }),
    })
    const { id: postId } = await response.json()
    if (response.ok) {
      navigate({ to: '/posts/$postId', params: { postId } })
    }
  }
}
```

**Declarative redirect on mount** with `<Navigate>`:

```tsx
function Component() {
  return <Navigate to="/posts/$postId" params={{ postId: 'my-first-post' }} />
}
```

The docs call this "superior to `useNavigate` with immediate effects for client-only redirects" — and they are right, because the effect version renders one frame of the wrong page first.

**Outside React entirely**, use the router instance:

```tsx
router.navigate({ to: '/posts/$postId', params: { postId } })
```

> **Wrong vs. right — redirecting on mount.**
>
> ```tsx
> // ❌ WRONG — renders the page once, then bounces. Flash of wrong content.
> useEffect(() => { navigate({ to: '/login' }) }, [])
>
> // ✅ BETTER — no wasted render
> return <Navigate to="/login" />
>
> // ✅ BEST — never renders at all; blocks before the component exists
> // (in the route definition)
> beforeLoad: () => { throw redirect({ to: '/login' }) }
> ```
>
> The third form is the subject of [Part 8](./tanstack-router-guards-lifecycle.md#part-8) and is what you want for auth.

**🧪 Try It — catch a broken link at compile time.**

```tsx
<Link to="/posts/$postId">Broken</Link>
```

Run `npx tsc --noEmit`. **Expected output:** an error saying `params` is required (the exact wording varies by TS version, but it will name `params` and the `postId` key). Now add `params={{ postId: '1' }}` and it passes. In a string-typed router this would have shipped and rendered a link to the literal text `/posts/$postId`.

[↑ Back to top](#table-of-contents)

---

<a id="p4-advanced"></a>
### Advanced: `createLink`, `linkOptions`, and redirects

**Wrapping a design-system component.** You will not use the bare `<Link>` for long — you will have a `<Button as="link">` or a Mantine `<Anchor>`. `createLink` preserves full type safety through the wrapper ([custom link guide](https://tanstack.com/router/latest/docs/framework/react/guide/custom-link)):

```tsx
import * as React from 'react'
import { createLink, LinkComponent } from '@tanstack/react-router'

interface BasicLinkProps extends React.AnchorHTMLAttributes<HTMLAnchorElement> {}

const BasicLinkComponent = React.forwardRef<HTMLAnchorElement, BasicLinkProps>(
  (props, ref) => {
    return <a ref={ref} {...props} className={'block px-3 py-2 text-blue-700'} />
  },
)

const CreatedLinkComponent = createLink(BasicLinkComponent)

export const CustomLink: LinkComponent<typeof BasicLinkComponent> = (props) => {
  return <CreatedLinkComponent preload={'intent'} {...props} />
}
```

Three steps, in order: a base component that forwards a ref and spreads props onto an `<a>`; `createLink()` to add the router behaviour; and a typed re-export that bakes in your defaults. The docs provide worked versions for React Aria, Chakra, MUI and Mantine.

> **Wrong vs. right — the wrapper that eats the types.**
>
> ```tsx
> // ❌ WRONG — `to` is now `string`; the whole point is gone
> function NavLink(props: { to: string; children: React.ReactNode }) {
>   return <Link to={props.to} className="nav">{props.children}</Link>
> }
>
> // ✅ RIGHT — createLink threads Link's generics through your component
> const NavLink = createLink(BaseAnchor)
> ```
>
> This is the most common way a codebase silently loses its type safety: someone writes a convenience wrapper typed with `to: string`, and every call site downstream stops being checked.

**`linkOptions`** lets you define a navigation target once, type-checked, and reuse it across a `<Link>`, a `navigate()` and a `redirect()`:

```tsx
import { linkOptions } from '@tanstack/react-router'

const postLink = linkOptions({
  to: '/posts/$postId',
  params: { postId: '42' },
  search: { tab: 'meta' },
})

<Link {...postLink}>Post 42</Link>
navigate(postLink)
```

Useful for nav config arrays, where an inline object literal would lose its narrowing.

**`redirect()`** is the navigation primitive for route lifecycle code. It is *thrown*, not returned:

```tsx
import { redirect } from '@tanstack/react-router'

beforeLoad: ({ context, location }) => {
  if (!context.auth.isAuthenticated) {
    throw redirect({
      to: '/login',
      search: { redirect: location.href },
    })
  }
}
```

It accepts the same options as `navigate()`. Because it is thrown, it also unwinds any code after it — which is why guards read cleanly, and why you need `isRedirect()` if you wrap a guard in `try/catch` (see [Part 8](./tanstack-router-guards-lifecycle.md#part-8)).

[↑ Back to top](#table-of-contents)

---

<a id="p4-mastery"></a>
### Mastery: route masking

Route masking shows one URL in the address bar while the router is actually on another. The [route masking guide](https://tanstack.com/router/latest/docs/framework/react/guide/route-masking) defines it as "a way to mask the actual URL of a route that gets persisted to the browser's history and URL bar."

The canonical case is a photo modal: clicking a thumbnail should open `/photos/5/modal`, but the address bar should read `/photos/5` so that sharing the link gives you the full page rather than a detached modal.

**Per-navigation, via the `mask` prop:**

```tsx
<Link
  to="/photos/$photoId/modal"
  params={{ photoId: 5 }}
  mask={{
    to: '/photos/$photoId',
    params: { photoId: 5 },
  }}
>
  Open Photo
</Link>
```

**Declaratively, at the router level:**

```tsx
import { createRouteMask } from '@tanstack/react-router'

const photoModalToPhotoMask = createRouteMask({
  routeTree,
  from: '/photos/$photoId/modal',
  to: '/photos/$photoId',
  params: (prev) => ({ photoId: prev.photoId }),
})

const router = createRouter({
  routeTree,
  routeMasks: [photoModalToPhotoMask],
})
```

**How it works**, and why this matters for debugging: the real location is stashed in `location.state.__tempLocation`, and the router uses that instead of parsing the URL bar. So the masked URL survives back/forward navigation within the session — the history entry carries the real destination alongside the fake display string.

On a **full page reload**, though, that state is gone from the perspective of a fresh parse. By default masks persist because the masking data lives in the history stack; set `unmaskOnReload: true` (at router level, per mask, or per navigation) to reveal the real URL on reload instead. Which you want depends on the case: for a photo modal, unmasking on reload is usually right — the user refreshed and should get the full page.

> **The trap.** Masking is a *display* feature, not a security or access feature. The real route matches, its loader runs, and its component renders. Never use it to hide a route from a user; use [`beforeLoad` guards](./tanstack-router-guards-lifecycle.md#part-8) for that.

[↑ Back to top](#table-of-contents)

---

<a id="p4-cheat"></a>
### Part 4 cheat sheet

| Task | API |
|---|---|
| Declarative link | `<Link to="/posts/$postId" params={{ postId }} />` |
| Active styling | `activeProps` / `inactiveProps` / `activeOptions={{ exact: true }}` |
| Relative link | `<Link from={Route.fullPath} to="../siblings" />` |
| Reload current route | `<Link to="." />` |
| Go to parent | `<Link to=".." />` |
| Imperative | `const navigate = useNavigate({ from: '/x' }); navigate({ to, params })` |
| Redirect on mount | `<Navigate to="/login" />` |
| Redirect from lifecycle | `throw redirect({ to: '/login', search: { redirect: location.href } })` |
| Outside React | `router.navigate({ to, params })` |
| Replace instead of push | `replace: true` |
| Skip scroll reset | `resetScroll: false` |
| Reusable target | `linkOptions({ to, params, search })` |
| Custom link component | `createLink(Base)` + `LinkComponent<typeof Base>` |
| Show a different URL | `mask={{ to, params }}` or `createRouteMask` + `routeMasks` |
| Reveal real URL on reload | `unmaskOnReload: true` |

[↑ Back to top](#table-of-contents)

---

<a id="next"></a>
## Where to go next

- **Next in sequence:** [Search Params (Part 5)](./tanstack-router-search-params.md) — the chapter that most distinguishes this router.
- **Related:** `<Link search={...}>` is fully covered there; preloading props are covered in [Part 10](./tanstack-router-performance-internals.md#part-10).
- **Back to:** [Overview](./tanstack-router-mastery-guide.md)

### Quick self-check for Parts 3–4

1. `/posts/new` and `/posts/$postId` both exist. Which wins, and what would you have to do to change that?
2. Describe the difference between `_admin.tsx`, `admin_.tsx` and `(admin)/` in one sentence each.
3. What does `params.parse` returning `false` mean, and what happens next?
4. Why does `priority: 100` not make a dynamic route beat a static one?
5. On `/posts/42`, why does a `<Link to="/posts">` render as active, and how do you stop it?
6. A teammate writes `function AppLink({ to }: { to: string })`. What has been lost, and what is the fix?
7. A masked route shows `/photos/5` but renders the modal. What is in `location.state`, and what happens on reload?

[↑ Back to top](#table-of-contents)
