# TanStack Router — Mastery Guide (Overview)

> A type-safe, client-first router for React where **the URL is application state** — path params, search params and all — and TypeScript knows about every bit of it.

**Written against:** `@tanstack/react-router` **1.170.x**, `@tanstack/router-plugin` 1.170.x, React 19, Vite 7+, TypeScript 5.9+
**Last updated:** September 14, 2026

---

## Table of Contents

- [About This Document](#about-this-document)
- [Why This Router Exists](#why-this-router-exists)
- [The Five Ideas That Explain Everything Else](#the-five-ideas)
- [The Document Set](#the-document-set)
- [The Running Example](#the-running-example)
- [Deliberately Not Covered](#deliberately-not-covered)
- [Suggested Learning Order](#suggested-learning-order)
- [Quick Self-Check](#quick-self-check)
- [Master Cheat Sheet](#master-cheat-sheet)

---

<a id="about-this-document"></a>
## About This Document

**Framing — curiosity-driven.** No deadline, no specific app driving it. The set is written as a **guided hands-on tutorial**: one example app (a small blog with an admin area) is built up progressively across the sub-docs, and every part carries runnable **"Try It"** exercises with the output you should actually see. Theory appears only where a mechanism cannot be understood without it — there is no standalone history section and no "evolution of routing in React" narrative.

**Confirmed scope.**

- **In scope:** the client-side React router — setup and the Vite plugin, file-based and code-based route trees, route matching, path params, navigation and `<Link>` type safety, search params as validated state, loaders and the built-in SWR cache, router context, `beforeLoad` guards and auth, error/not-found/pending states, navigation blocking, scroll restoration, code splitting, preloading, devtools, testing, and the internals (segment-trie matching, the type-inference machinery).
- **Brief only:** SSR and **TanStack Start** get one part in [Performance & Internals](./tanstack-router-performance-internals.md#part-12), covering the concepts and what Start adds — flagged as deserving its own doc later.
- **Deliberately excluded:** the Solid adapter, TanStack Start's server functions and deployment in depth, and TanStack Query as a topic in its own right (a short primer appears in [Data Loading](./tanstack-router-data-loading.md#query-primer) purely so the Router↔Query division of labour makes sense). Full list in [Deliberately Not Covered](#deliberately-not-covered).

**Depth tiers used.** Major parts run **Beginner → Working Knowledge → Advanced → Mastery**. Tiers are collapsed where a tier would be padding, and each collapse is stated inline. Notable collapses:

- **Setup** (Part 1) is Beginner → Working Knowledge only; its "advanced" material is really the plugin options table in Part 11.
- **Search Params** (Part 5) skips Beginner — there is no meaningful "minimal" tier beyond `validateSearch`, and the interesting material starts at Working Knowledge.
- **Internals** (Part 13) is Mastery-only by definition.

**Assumed background.** Solid **React** (hooks, Suspense, error boundaries, context) and solid **TypeScript** (generics, inference, declaration merging) — no primer; cross-linked to [`react/`](../../react/react-mastery-guide.md) and [`typescript/`](../../typescript/typescript-mastery-guide.md). Solid **Vite** — plugin config is explained tersely and cross-linked to [`vite/`](../../vite/vite-mastery-guide.md). **TanStack Query** gets a short primer before the integration section, because the loader-vs-query division of labour is where most people get confused.

**Per-section ingredients in use.** Plain-language explanation; **wrong vs. right** snippet pairs at every gotcha; a runnable **"Try It"** with expected output (curiosity-driven framing, so exercises rather than incident post-mortems — except where the official docs themselves document a production failure mode); ASCII diagrams where the concept is structural (route tree → component tree, navigation lifecycle, the matching trie).

**Sourcing rule.** Official TanStack docs first, cited inline as links at the specific claim they support. Where the official docs and a widespread blog convention disagree, the docs win and the outdated convention is named explicitly so you recognise it in the wild. *Caveat specific to this set:* Stack Overflow could not be retrieved by the tooling used to write these docs, so real-world gotchas are sourced from the official docs' own warnings, the TanStack blog, and the GitHub release stream rather than from SO threads. Where a claim is inferred rather than quoted, it says so inline.

**Formatting choices.** Clickable TOC per doc; "back to top" after each major section; explicit `<a id="..."></a>` anchors because tier headings (`Beginner`, `Advanced`, …) repeat across parts; a cheat-sheet table ending each major part; fenced blocks with language tags; every doc ends with a Suggested Order and a Quick Self-Check.

**To update this doc later.**

- **Preserve:** the curiosity-driven framing, the single running example app, the tier labels, and the part numbering — sub-docs reference parts by number across files.
- **Re-verify against current docs first:** the version banner at the top of every file; the plugin option table in Part 11 (`autoCodeSplitting`, `routeToken`, `indexToken`); the `loader` object form (`{ handler, staleReloadMode }`), which is newer than the bare-function form; optional path params `{-$param}` and prefix/suffix segments, which are recent additions; and anything about **TanStack Start**, which moves fastest and is labelled experimental in places by the router docs themselves.
- **Known churn point:** the docs site has been migrating URLs from `/docs/framework/react/<x>` to framework-agnostic `/docs/<x>`. Both resolve today; if a link 404s, try dropping `framework/react/`.

[↑ Back to top](#table-of-contents)

---

<a id="why-this-router-exists"></a>
## Why This Router Exists

Most routers treat the URL as a string you parse by hand. TanStack Router treats it as **typed, validated, first-class state**.

The [official overview](https://tanstack.com/router/latest/docs/framework/react/overview) makes two claims worth taking literally:

1. **"100% inferred TypeScript support."** You never hand-write a route type. The Vite plugin generates `routeTree.gen.ts` from your files, you register it once via declaration merging, and from then on `to`, `params`, `search`, and `loaderData` are checked at every call site. A typo in `<Link to="/psots">` is a compile error, not a blank screen at runtime.
2. **Search params are first-class state** — the docs describe it as *"having `useState` right in the URL!"* Every route declares a `validateSearch` schema; reading gives you a typed, structurally-shared object; writing is a type-checked functional update.

The practical consequence, and the thing to internalise before anything else:

```
        ┌──────────────────────── the URL ────────────────────────┐
        /posts/42/edit          ?tab=meta&draft=true          #notes
        └── pathname ──┘        └──── search params ────┘     └hash┘
             │                          │
             │  parsed by the           │  parsed JSON-first, then
             │  route tree into         │  run through validateSearch
             │  typed `params`          │
             ▼                          ▼
        Route.useParams()          Route.useSearch()
        { postId: 42 }             { tab: 'meta', draft: true }
             │                          │
             └──────────┬───────────────┘
                        ▼
            loader({ params, deps, context })   ← deps derive from search
                        │
                        ▼
                Route.useLoaderData()
```

Every arrow in that diagram is typed end to end. That is the whole pitch.

**A note on framing.** This is a *client-side* router first. It does SSR (Part 12), and TanStack Start builds a full-stack framework on top of it — but unlike the Next.js App Router, the router itself has no opinion about servers. If you are coming from [`nextjs/`](../../nextjs/nextjs-mastery-guide.md), the mental shift is: **routing and data loading are library concerns here, not framework concerns.** Nothing is magic, and nothing is hidden behind a build step except the generated route tree.

[↑ Back to top](#table-of-contents)

---

<a id="the-five-ideas"></a>
## The Five Ideas That Explain Everything Else

Understand these five and the rest of the API is mechanical.

| # | Idea | Why it matters | Covered in |
|---|---|---|---|
| 1 | **The route tree is generated, not written.** The plugin scans `src/routes/`, emits `routeTree.gen.ts`, and you register it once with `declare module`. | Types flow from files → generated tree → every `<Link>` | [Foundations, Part 1](./tanstack-router-foundations.md#part-1) |
| 2 | **Routes are sorted by specificity, then matched through a trie.** Index → static → dynamic → splat, deterministically. | You never fight route ordering | [Routing, Part 3](./tanstack-router-routing-navigation.md#part-3) |
| 3 | **Search params are validated state, not strings.** `validateSearch` is the schema; `loaderDeps` connects them to data. | URL-shareable UI state, for free | [Search Params, Part 5](./tanstack-router-search-params.md) |
| 4 | **Loaders run before render, behind a built-in SWR cache.** `staleTime` / `gcTime` / `shouldReload` control it; an external cache can take over. | No render-then-fetch waterfalls | [Data Loading, Part 6](./tanstack-router-data-loading.md#part-6) |
| 5 | **`beforeLoad` is middleware for an entire subtree, and it can extend context.** Throw `redirect()` or `notFound()` from it. | Auth, guards, breadcrumbs, dependency injection | [Guards & Lifecycle, Part 8](./tanstack-router-guards-lifecycle.md#part-8) |

[↑ Back to top](#table-of-contents)

---

<a id="the-document-set"></a>
## The Document Set

| Doc | Parts | What it covers |
|---|---|---|
| **This overview** | — | Framing, the five ideas, learning order, master cheat sheet |
| [Foundations](./tanstack-router-foundations.md) | 1–2 | Install, Vite plugin, `routeTree.gen.ts`, `declare module`, `createRouter`, `RouterProvider`, the root route, `Outlet`, code-based routing |
| [Routing & Navigation](./tanstack-router-routing-navigation.md) | 3–4 | Every file convention, matching precedence, path params (including optional and prefix/suffix), `<Link>`, `useNavigate`, `redirect`, `createLink`, route masking |
| [Search Params](./tanstack-router-search-params.md) | 5 | `validateSearch`, Zod/Valibot adapters, search middlewares, structural sharing, `useSearch`, functional updaters |
| [Data Loading](./tanstack-router-data-loading.md) | 6–7 | Loaders, `loaderDeps`, the SWR cache, router context, deferred data and `Await`, TanStack Query primer and integration |
| [Guards & Lifecycle](./tanstack-router-guards-lifecycle.md) | 8–9 | `beforeLoad`, authenticated routes, `redirect`, error/pending/not-found components, `notFound()` modes, navigation blocking, scroll restoration |
| [Performance & Internals](./tanstack-router-performance-internals.md) | 10–13 | Code splitting, preloading strategies, devtools, testing, SSR and TanStack Start (brief), matching internals, the inference machinery |

[↑ Back to top](#table-of-contents)

---

<a id="the-running-example"></a>
## The Running Example

Every sub-doc builds on the same app — a small blog with an admin area. By the end of Part 13 its route tree looks like this:

```
src/routes/
├── __root.tsx                      → always rendered; holds <Outlet/> + devtools
├── index.tsx                       → /
├── about.tsx                       → /about
├── posts.tsx                       → /posts        (layout: list + <Outlet/>)
├── posts.index.tsx                 → /posts/       (the list's empty state)
├── posts.$postId.tsx               → /posts/$postId
├── posts_.$postId.edit.tsx         → /posts/$postId/edit   (un-nested from posts.tsx)
├── _authed.tsx                     → pathless guard: beforeLoad → redirect
├── _authed.admin.tsx               → /admin
├── (marketing)/pricing.tsx         → /pricing      (group dir: organisation only)
├── files.$.tsx                     → /files/*      (splat)
└── -components/                    → excluded from routing; plain colocated code
```

Each convention on that list is introduced where it first earns its place, never as a list to memorise up front.

[↑ Back to top](#table-of-contents)

---

<a id="deliberately-not-covered"></a>
## Deliberately Not Covered (and where to look instead)

| Topic | Why it is out | Where to look |
|---|---|---|
| **TanStack Start in depth** — server functions, server routes, deployment targets | Deserves its own doc; parts are labelled experimental by the router docs themselves | [Router SSR guide](https://tanstack.com/router/latest/docs/framework/react/guide/ssr); Part 12 has the concepts |
| **TanStack Query as a topic** | A separate `tanstack/tanstack-query/` doc set is planned | [The Part 7 primer](./tanstack-router-data-loading.md#query-primer) covers only what the integration needs |
| **Solid adapter** (`@tanstack/solid-router`) | Same core, different adapter; every concept here transfers | [Solid docs](https://tanstack.com/router/v1/docs/framework/solid/installation/with-vite) |
| **Virtual file routes** | Niche: programmatic route trees for migrations and monorepos | [Virtual File Routes](https://tanstack.com/router/latest/docs/routing/virtual-file-routes) |
| **Migration from React Router** | There is no official migration page at a stable URL as of this writing, so an API mapping would be invention rather than a sourced claim | Compare Parts 3–4 against your existing route config directly |
| **Non-Vite bundlers** (Webpack, Rspack, esbuild, the standalone CLI generator) | Vite is assumed per your background; the config options are identical | [File-based routing API](https://tanstack.com/router/latest/docs/api/file-based-routing) |

[↑ Back to top](#table-of-contents)

---

<a id="suggested-learning-order"></a>
## Suggested Learning Order

**First pass — get an app running and understand the loop (about half a day).**

1. [Foundations, Part 1](./tanstack-router-foundations.md#part-1) — scaffold, plugin, `declare module`. Do the Try It. Do not skip the declaration-merging step; it is where all the type safety comes from.
2. [Routing, Part 3](./tanstack-router-routing-navigation.md#part-3) — file conventions and matching order, through the Working Knowledge tier.
3. [Routing, Part 4](./tanstack-router-routing-navigation.md#part-4) — `<Link>` and `useNavigate`.
4. [Data Loading, Part 6](./tanstack-router-data-loading.md#part-6) — loaders and `useLoaderData`, Beginner and Working Knowledge tiers.

**Second pass — the parts that make this router different (about a day).**

5. [Search Params, Part 5](./tanstack-router-search-params.md) — in full. This is the highest-leverage chapter in the set.
6. [Data Loading, Part 7](./tanstack-router-data-loading.md#part-7) — router context and the Query integration.
7. [Guards & Lifecycle, Part 8](./tanstack-router-guards-lifecycle.md#part-8) — `beforeLoad` and auth.

**Third pass — production concerns.**

8. [Guards & Lifecycle, Part 9](./tanstack-router-guards-lifecycle.md#part-9) — errors, not-found, pending UI, blocking, scroll.
9. [Performance, Parts 10–11](./tanstack-router-performance-internals.md#part-10) — code splitting, preloading, devtools, testing.

**Fourth pass — when you want to know why.**

10. [Parts 12–13](./tanstack-router-performance-internals.md#part-12) — SSR and Start, the segment trie, the inference machinery.

**If you are arriving from the Next.js App Router**, read Part 3 and Part 6 first and hold this contrast throughout: there is no server-component boundary here, loaders are client-side by default, and caching is explicit per route rather than framework-managed.

[↑ Back to top](#table-of-contents)

---

<a id="quick-self-check"></a>
## Quick Self-Check

Work through these without opening the docs. Each links to where it is answered.

1. What exactly does `declare module '@tanstack/react-router'` accomplish, and what breaks if you omit it? → [Part 1](./tanstack-router-foundations.md#part-1)
2. Given routes `/posts/new`, `/posts/$postId` and `/posts/`, which matches `/posts/new`, and why is the answer independent of file order? → [Part 3](./tanstack-router-routing-navigation.md#part-3)
3. What is the difference between `_authed.admin.tsx` and `posts_.$postId.edit.tsx`? Both use an underscore. → [Part 3](./tanstack-router-routing-navigation.md#part-3)
4. Why does the search-params guide recommend Zod's `.catch()` over `.default()`? → [Part 5](./tanstack-router-search-params.md)
5. You write `loaderDeps: ({ search }) => search`. What is the bug? → [Part 6](./tanstack-router-data-loading.md#part-6)
6. Default `staleTime` is `0` for navigation but `30_000` for preloads. Why the asymmetry? → [Part 6](./tanstack-router-data-loading.md#part-6) and [Part 10](./tanstack-router-performance-internals.md#part-10)
7. When integrating TanStack Query, why do you set `defaultPreloadStaleTime: 0`? → [Part 7](./tanstack-router-data-loading.md#part-7)
8. Why can you not call `useAuth()` inside `createRouter()`, and what is the standard workaround? → [Part 8](./tanstack-router-guards-lifecycle.md#part-8)
9. A `beforeLoad` guard blocks the UI. Why is it explicitly *not* an authorization boundary? → [Part 8](./tanstack-router-guards-lifecycle.md#part-8)
10. What is the difference between `notFoundMode: 'fuzzy'` and `'root'`? → [Part 9](./tanstack-router-guards-lifecycle.md#part-9)
11. Which parts of a route config are "critical" and therefore never code-split? → [Part 10](./tanstack-router-performance-internals.md#part-10)
12. Matching went from O(N) in the number of routes to O(M) in the number of path segments. What data structure made that possible? → [Part 13](./tanstack-router-performance-internals.md#part-13)

[↑ Back to top](#table-of-contents)

---

<a id="master-cheat-sheet"></a>
## Master Cheat Sheet

### File conventions

| File | Route | Meaning |
|---|---|---|
| `__root.tsx` | — | Always rendered, wraps everything |
| `about.tsx` | `/about` | Static route |
| `posts.tsx` | `/posts` | Layout route (renders `<Outlet/>`) |
| `posts.index.tsx` | `/posts/` | Index — parent matched exactly |
| `posts.$postId.tsx` | `/posts/$postId` | Dynamic segment |
| `posts.{-$category}.tsx` | `/posts/{-$category}` | **Optional** segment |
| `files.$.tsx` | `/files/$` | Splat — captures the rest as `_splat` |
| `_authed.tsx` | *(no path)* | Pathless layout — wraps children, adds nothing to the URL |
| `posts_.$postId.edit.tsx` | `/posts/$postId/edit` | **Non-nested** — escapes `posts.tsx`'s layout |
| `(marketing)/pricing.tsx` | `/pricing` | Group directory — organisation only |
| `-components/Foo.tsx` | — | Excluded from route generation |
| `posts.lazy.tsx` | — | Non-critical config, code-split |

### Route options

| Option | Runs when | Typical use |
|---|---|---|
| `validateSearch` | On every URL parse | Schema for `?params` |
| `beforeLoad` | Before the loader, parent-first | Guards, context extension, redirects |
| `loaderDeps` | Before the loader | Select the search params the loader depends on |
| `loader` | After `beforeLoad` | Fetch data |
| `component` | After loaders resolve | The UI |
| `pendingComponent` | While loading | Loading UI |
| `errorComponent` | On thrown error | Error UI |
| `notFoundComponent` | On `notFound()` | 404 UI |
| `staleTime` / `gcTime` / `shouldReload` | Cache control | SWR behaviour |
| `params.parse` / `params.stringify` | Match / build | Typed path params |

### Hooks

| Hook | Returns |
|---|---|
| `Route.useParams()` | Typed path params |
| `Route.useSearch()` | Validated search params |
| `Route.useLoaderData()` | The loader's return value |
| `Route.useRouteContext()` | Merged context at this route |
| `useNavigate({ from })` | Type-safe imperative navigate |
| `useBlocker({ shouldBlockFn })` | Navigation blocking |
| `useElementScrollRestoration({ id })` | Scroll offset for virtualised lists |
| `getRouteApi('/path')` | Route-typed hooks without importing the route |

### Router options worth knowing on day one

| Option | Default | Set it to… |
|---|---|---|
| `defaultPreload` | `false` | `'intent'` — the biggest single perf win |
| `defaultPreloadDelay` | `50` (ms) | Tune hover sensitivity |
| `defaultPreloadStaleTime` | `30_000` | `0` when TanStack Query owns the cache |
| `scrollRestoration` | `false` | `true` |
| `defaultNotFoundComponent` | built-in | Your 404 |
| `notFoundMode` | `'fuzzy'` | `'root'` for a single global 404 |
| `context` | `{}` | `{ queryClient, auth }` |

*Defaults above come from the [preloading](https://tanstack.com/router/latest/docs/framework/react/guide/preloading), [data loading](https://tanstack.com/router/latest/docs/framework/react/guide/data-loading), [scroll restoration](https://tanstack.com/router/latest/docs/framework/react/guide/scroll-restoration) and [not-found](https://tanstack.com/router/latest/docs/framework/react/guide/not-found-errors) guides; each is re-cited at the point of use in the sub-docs.*

[↑ Back to top](#table-of-contents)

---

**Next:** [Foundations (Parts 1–2) →](./tanstack-router-foundations.md)
