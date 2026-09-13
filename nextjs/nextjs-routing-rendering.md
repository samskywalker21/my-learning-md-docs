# Next.js — Routing & Rendering (Parts 1–3)

Project setup, the `app` directory, file-system routing, and the Server/Client Component boundary.

> **Spec:** this doc follows the shared spec in [`nextjs-mastery-guide.md`](./nextjs-mastery-guide.md#about-this-document) — goal-driven framing (building a full-stack app), Beginner → Working Knowledge → Advanced → Mastery tiers, wrong-vs-right pairs, production Real Scenarios, inline primary sourcing. Written against **Next.js 16.3.4**, verified **September 11, 2026**.

**Part 3 is the most important thing in this set.** Parts 1 and 2 are conventions you can look up; Part 3 is a mental model that, if wrong, makes every later Part confusing.

---

## Table of Contents

- [Part 1: Project Setup and the `app` Directory](#part-1-project-setup-and-the-app-directory)
  - [1.1 Beginner — creating the project](#11-beginner--creating-the-project)
  - [1.2 Working Knowledge — `next.config.ts` and the files that matter](#12-working-knowledge--nextconfigts-and-the-files-that-matter)
  - [1.3 Advanced — organising a project that will grow](#13-advanced--organising-a-project-that-will-grow)
  - [1.4 Cheat sheet: Part 1](#14-cheat-sheet-part-1)
- [Part 2: Routing](#part-2-routing)
  - [2.1 Beginner — folders are URLs](#21-beginner--folders-are-urls)
  - [2.2 Working Knowledge — dynamic segments, params, searchParams](#22-working-knowledge--dynamic-segments-params-searchparams)
  - [2.3 Advanced — route groups, loading, error, not-found](#23-advanced--route-groups-loading-error-not-found)
  - [2.4 Mastery — parallel and intercepting routes](#24-mastery--parallel-and-intercepting-routes)
  - [2.5 Cheat sheet: Part 2](#25-cheat-sheet-part-2)
- [Part 3: Server and Client Components](#part-3-server-and-client-components)
  - [3.1 Beginner — the default is the server](#31-beginner--the-default-is-the-server)
  - [3.2 Working Knowledge — the boundary is a module graph](#32-working-knowledge--the-boundary-is-a-module-graph)
  - [3.3 Advanced — composition patterns that keep the boundary small](#33-advanced--composition-patterns-that-keep-the-boundary-small)
  - [3.4 Mastery — the RSC Payload and environment poisoning](#34-mastery--the-rsc-payload-and-environment-poisoning)
  - [3.5 Cheat sheet: Part 3](#35-cheat-sheet-part-3)

---

## Part 1: Project Setup and the `app` Directory

<a id="part-1-project-setup-and-the-app-directory"></a>

This Part gets **Beginner → Advanced** only. There is no "Mastery" tier for project setup — the expert-level material (build output, tracing, adapters) is really [Part 13](./nextjs-production-deployment.md#part-13-deployment-and-internals) and lives there.

### 1.1 Beginner — creating the project

```bash
npx create-next-app@latest my-app --yes
cd my-app
npm run dev
```

`--yes` accepts the recommended defaults: TypeScript, ESLint, Tailwind CSS, App Router, Turbopack, the `@/*` import alias, and an `AGENTS.md` file ([Installation](https://nextjs.org/docs/app/getting-started/installation)). If you run it without `--yes` you get prompts, including "Would you like to use React Compiler?" — say no for now; it is stable but adds Babel to the build and slows compiles, and you can enable it later with one config line.

The minimum viable App Router app is two files:

```tsx
// app/layout.tsx — required; owns <html> and <body>
export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
```

```tsx
// app/page.tsx — makes "/" routable
export default function Page() {
  return <h1>Hello, Next.js!</h1>
}
```

That's it. `app/layout.tsx` is the **root layout**: it is required, and it must render `<html>` and `<body>` — Next.js does not supply them ([Layouts and Pages](https://nextjs.org/docs/app/getting-started/layouts-and-pages#creating-a-layout)).

### 1.2 Working Knowledge — `next.config.ts` and the files that matter

The root-level files you will actually edit:

| File | What it does |
|---|---|
| `next.config.ts` | Framework configuration; typed with `NextConfig` |
| `proxy.ts` | Request interception — see [Part 9](./nextjs-database-auth.md#part-9-authentication-and-sessions) |
| `instrumentation.ts` | `register()` runs once at server start — see [Part 12](./nextjs-production-deployment.md#part-12-production-concerns) |
| `.env`, `.env.local` | Environment variables — see [Part 12](./nextjs-production-deployment.md#part-12-production-concerns) |
| `tsconfig.json` | The `@/*` path alias lives here |
| `eslint.config.mjs` | Flat config; **you run ESLint yourself now** |

A realistic starting config:

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  // Set this now, not the week you deploy — Part 13 depends on it.
  output: 'standalone',
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'images.example.com', pathname: '/uploads/**' },
    ],
  },
}

export default nextConfig
```

**Wrong vs. right — Turbopack config location:**

```ts
// ✗ Next.js 15 shape. Silently ignored in 16.
const nextConfig: NextConfig = {
  experimental: { turbopack: { resolveAlias: { '~*': '*' } } },
}

// ✓ Next.js 16 — turbopack graduated to a top-level key.
const nextConfig: NextConfig = {
  turbopack: { resolveAlias: { '~*': '*' } },
}
```

Source: [Turbopack configuration location](https://nextjs.org/docs/app/guides/upgrading/version-16#turbopack-configuration-location).

**Wrong vs. right — linting:**

```json
// ✗ Removed in Next.js 16. `next build` no longer lints either.
{ "scripts": { "lint": "next lint" } }

// ✓ Run the linter directly.
{ "scripts": { "lint": "eslint", "lint:fix": "eslint --fix" } }
```

This matters more than it looks: teams that relied on `next build` failing on lint errors in CI silently lost that gate on upgrade. Add an explicit `npm run lint` step ([`next lint` removal](https://nextjs.org/docs/app/guides/upgrading/version-16#next-lint-command)).

**The `@/` alias.** `create-next-app` configures it; if you are setting up manually it is plain `tsconfig.json`:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

**Generated types.** Next.js generates global helper types during `next dev`, `next build`, or `next typegen` ([Route props helpers](https://nextjs.org/docs/app/getting-started/layouts-and-pages#route-props-helpers)):

```tsx
// No import needed — PageProps, LayoutProps and RouteContext are global.
export default async function Page(props: PageProps<'/blog/[slug]'>) {
  const { slug } = await props.params
  return <h1>{slug}</h1>
}
```

If your editor reddens these, run `npx next typegen`. They are generated from your actual route tree, so `PageProps<'/blog/[slug]'>` knows `slug` is a string and `PageProps<'/about'>` knows `params` is `{}`.

### 1.3 Advanced — organising a project that will grow

Next.js is explicitly unopinionated about where non-route files go ([Project structure](https://nextjs.org/docs/app/getting-started/project-structure#organizing-your-project)). Two facts constrain the decision:

1. A folder in `app/` is **not routable** until it contains `page.tsx` or `route.ts`. You can colocate anything you like inside route folders.
2. A folder prefixed with `_` is opted out of routing entirely, even if it contains a `page.tsx`.

The layout this set assumes:

```
my-app/
├── app/
│   ├── layout.tsx                 root layout
│   ├── page.tsx                   /
│   ├── (marketing)/               route group — no URL segment
│   │   ├── about/page.tsx         /about
│   │   └── pricing/page.tsx       /pricing
│   ├── (app)/                     route group with its own layout
│   │   ├── layout.tsx             authenticated shell (nav, user menu)
│   │   ├── dashboard/
│   │   │   ├── page.tsx           /dashboard
│   │   │   ├── loading.tsx
│   │   │   └── error.tsx
│   │   └── projects/
│   │       ├── page.tsx           /projects
│   │       └── [id]/page.tsx      /projects/123
│   └── api/
│       └── webhooks/stripe/route.ts
├── components/                    shared UI
│   └── ui/                        buttons, inputs, primitives
├── lib/
│   ├── db/
│   │   ├── index.ts               the db singleton    (Part 8)
│   │   └── schema.ts              Drizzle schema      (Part 8)
│   ├── dal.ts                     Data Access Layer   (Part 9)
│   └── actions/                   Server Actions      (Part 6)
├── proxy.ts
├── instrumentation.ts
└── next.config.ts
```

The two structural decisions worth defending:

**Route groups for layout boundaries, not for tidiness.** `(marketing)` and `(app)` exist because they need *different layouts* — one has a public nav, the other an authenticated shell. Grouping purely to look organised adds indirection for nothing.

**`lib/` outside `app/`.** Colocation is allowed, but a flat `lib/` makes the server-only boundary easy to audit: everything in `lib/db` and `lib/dal.ts` gets `import 'server-only'` at the top (see [§3.4](#34-mastery--the-rsc-payload-and-environment-poisoning)), and you can check that with one grep.

> **Editor tip that saves real time.** The App Router names every file `page.tsx`, so your tab bar becomes twelve identical tabs. Add [custom editor labels](https://nextjs.org/docs/app/getting-started/installation#set-up-your-editor) to `.vscode/settings.json`:
>
> ```json
> {
>   "workbench.editor.customLabels.patterns": {
>     "**/app/**/page.tsx": "${dirname(1)}/${dirname} - page.tsx",
>     "**/app/**/layout.tsx": "${dirname(1)}/${dirname} - layout.tsx",
>     "**/app/**/route.ts": "${dirname(1)}/${dirname} - route.ts"
>   }
> }
> ```
>
> JetBrains IDEs do this automatically.

### 1.4 Cheat sheet: Part 1

| Command | What it does |
|---|---|
| `next dev` | Dev server, Turbopack, output in `.next/dev` |
| `next dev --webpack` | Opt out of Turbopack |
| `next build` | Production build; type-checks and prerenders |
| `next start` | Serve the production build |
| `next typegen` | Regenerate `PageProps` / `LayoutProps` / `RouteContext` |
| `next upgrade` | Bump Next.js and refresh bundled docs |
| `npx @next/codemod@canary upgrade latest` | Run the version codemods |

| Config key | Default in 16 | Notes |
|---|---|---|
| `output` | undefined | `'standalone'` for Docker; `'export'` for static |
| `cacheComponents` | `false` | Enables `use cache` + PPR ([Part 5](./nextjs-data-caching.md#part-5-caching-and-revalidation)) |
| `reactCompiler` | `false` | Stable, but Babel-based, so builds get slower |
| `turbopack` | `{}` | Top-level, no longer `experimental` |
| `images.remotePatterns` | `[]` | Required for remote `<Image>` sources |
| `deploymentId` | undefined | Version-skew protection ([Part 13](./nextjs-production-deployment.md#part-13-deployment-and-internals)) |

[↑ Back to top](#table-of-contents)

---

## Part 2: Routing

<a id="part-2-routing"></a>

### 2.1 Beginner — folders are URLs

**Folders define URL segments; files define what renders there** ([Layouts and Pages](https://nextjs.org/docs/app/getting-started/layouts-and-pages#creating-a-nested-route)).

```
app/
├── page.tsx              →  /
├── blog/
│   ├── page.tsx          →  /blog
│   └── [slug]/
│       └── page.tsx      →  /blog/anything
└── settings/
    └── billing/
        └── page.tsx      →  /settings/billing
```

Layouts nest automatically. `app/layout.tsx` wraps `app/blog/layout.tsx`, which wraps `app/blog/[slug]/page.tsx`:

```tsx
// app/blog/layout.tsx
export default function BlogLayout({ children }: { children: React.ReactNode }) {
  return (
    <section>
      <h1>Blog</h1>
      {children}
    </section>
  )
}
```

The one thing worth internalising early: **layouts preserve state across navigation and do not re-render** when you move between their children ([Layouts and Pages](https://nextjs.org/docs/app/getting-started/layouts-and-pages#creating-a-layout)). That's a performance feature and a security trap — see [Part 9](./nextjs-database-auth.md#part-9-authentication-and-sessions).

### 2.2 Working Knowledge — dynamic segments, params, searchParams

Three bracket forms:

| Folder | Matches |
|---|---|
| `[slug]` | exactly one segment — `/blog/hello` |
| `[...slug]` | one or more — `/shop/a`, `/shop/a/b/c` |
| `[[...slug]]` | zero or more — `/docs`, `/docs/a/b` |

**`params` and `searchParams` are Promises.** This is the single most common way Next.js 16 breaks copied code:

```tsx
// ✗ Next.js 14/15 style. Throws in 16 — sync access was removed.
export default function Page({ params }: { params: { slug: string } }) {
  return <h1>{params.slug}</h1>
}

// ✓ Next.js 16.
export default async function Page({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  return <h1>{slug}</h1>
}

// ✓ Better still — generated types, no hand-written Promise.
export default async function Page(props: PageProps<'/blog/[slug]'>) {
  const { slug } = await props.params
  return <h1>{slug}</h1>
}
```

The same applies to `cookies()`, `headers()` and `draftMode()`, and to the `params`/`id` passed to `opengraph-image`, `icon` and `sitemap` generators ([Async Request APIs](https://nextjs.org/docs/app/guides/upgrading/version-16#async-request-apis-breaking-change)). If you are migrating a real codebase, the codemod handles it: `npx @next/codemod@canary next-async-request-api .`

**`searchParams` is pages-only, and it makes the route dynamic.** A layout does not receive `searchParams` — by design, because layouts don't re-render on navigation, so the value would go stale. Read it in the page, or in a Client Component via `useSearchParams()` ([Rendering with search params](https://nextjs.org/docs/app/getting-started/layouts-and-pages#rendering-with-search-params)).

```tsx
export default async function Page(props: PageProps<'/search'>) {
  const { q, page } = await props.searchParams
  const results = await search(String(q ?? ''), Number(page ?? 1))
  return <Results items={results} />
}
```

Which to use when:

| Need | Use |
|---|---|
| Search params decide what data to load | `searchParams` prop (server) |
| Filtering a list already loaded via props | `useSearchParams()` (client) |
| Reading params inside an event handler only | `new URLSearchParams(window.location.search)` |

### 2.3 Advanced — route groups, loading, error, not-found

**Route groups** `(name)` organise without adding a URL segment. Their real power is layout scoping:

```
app/
├── (marketing)/
│   ├── layout.tsx        public nav + footer
│   └── pricing/page.tsx  →  /pricing
└── (app)/
    ├── layout.tsx        authenticated shell
    └── dashboard/page.tsx →  /dashboard
```

Both groups can even have their own **root** layout (with their own `<html>`/`<body>`) if you delete the top-level `app/layout.tsx` — useful when a marketing site and an app share a domain but nothing else ([Creating multiple root layouts](https://nextjs.org/docs/app/getting-started/project-structure#creating-multiple-root-layouts)).

**The four UI convention files**, and what each actually wraps:

```
  layout.tsx
   └── template.tsx          (remounts on every navigation; rare)
        └── error.tsx         React error boundary  — 'use client'
             └── loading.tsx  React Suspense boundary
                  └── page.tsx  (or the next nested layout)
```

Source: [component hierarchy](https://nextjs.org/docs/app/getting-started/project-structure#component-hierarchy).

`loading.tsx` is sugar: Next.js wraps your page in `<Suspense fallback={<Loading />}>` for you.

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return <DashboardSkeleton />
}
```

`error.tsx` must be a Client Component, and **its second prop is `retry`, not `reset`** — this changed in 16 and every older example has it wrong:

```tsx
// app/dashboard/error.tsx
'use client'

import { useEffect } from 'react'

export default function ErrorPage({
  error,
  retry,
}: {
  error: Error & { digest?: string }
  retry: () => void
}) {
  useEffect(() => {
    console.error(error) // send to your error tracker
  }, [error])

  return (
    <div>
      <h2>Something went wrong.</h2>
      <button onClick={() => retry()}>Try again</button>
    </div>
  )
}
```

Source: [Error handling](https://nextjs.org/docs/app/getting-started/error-handling#nested-error-boundaries). In production, `error.message` is redacted for server errors and you get `error.digest` — a hash you match against your server logs. Don't build UI that depends on the message text.

For component-level boundaries that aren't tied to a route segment, 16 adds `catchError` from `next/error`:

```tsx
'use client'
import { catchError, type ErrorInfo } from 'next/error'

function Fallback(props: { title: string }, { error, retry }: ErrorInfo) {
  return (
    <div>
      <h2>{props.title}</h2>
      <p>{error.message}</p>
      <button onClick={() => retry()}>Try again</button>
    </div>
  )
}

export default catchError(Fallback)
```

`not-found.tsx` renders when you call `notFound()`:

```tsx
import { notFound } from 'next/navigation'

export default async function Page(props: PageProps<'/blog/[slug]'>) {
  const { slug } = await props.params
  const post = await getPost(slug)
  if (!post) notFound() // throws a control-flow exception; nothing after runs
  return <article>{post.title}</article>
}
```

> **Real Scenario — the error boundary that never fired.**
> A team wrapped their dashboard in `app/dashboard/error.tsx` and tested it by throwing inside a button's `onClick`. Nothing happened; the app just died with a console error. They filed a framework bug.
> **Cause:** React error boundaries only catch errors thrown **during rendering**, not in event handlers or async callbacks ([Error handling](https://nextjs.org/docs/app/getting-started/error-handling#nested-error-boundaries)). The boundary was fine — the test was wrong.
> **Fix:** catch in the handler and store in state, or wrap the work in `startTransition`, whose unhandled errors *do* bubble to the nearest boundary.
>
> ```tsx
> const [isPending, startTransition] = useTransition()
> // errors thrown in here reach error.tsx
> const onClick = () => startTransition(() => { throw new Error('boom') })
> ```

### 2.4 Mastery — parallel and intercepting routes

These two conventions solve specific UI shapes. Reach for them when the shape matches; don't reach for them as a default.

**Parallel routes** (`@slot`) let one layout render several independent route trees at once, each with its own loading and error states.

```
app/dashboard/
├── layout.tsx
├── page.tsx
├── @analytics/
│   ├── page.tsx
│   └── default.tsx     ← REQUIRED in Next.js 16
└── @team/
    ├── page.tsx
    └── default.tsx     ← REQUIRED in Next.js 16
```

```tsx
// app/dashboard/layout.tsx — slots arrive as named props
export default function Layout(props: LayoutProps<'/dashboard'>) {
  return (
    <>
      {props.children}
      <aside>{props.analytics}</aside>
      <aside>{props.team}</aside>
    </>
  )
}
```

**The 16 breaking change:** every slot now needs an explicit `default.tsx` or the **build fails** ([Parallel routes `default.js` requirement](https://nextjs.org/docs/app/guides/upgrading/version-16#parallel-routes-defaultjs-requirement)). To reproduce the old implicit behaviour:

```tsx
// app/dashboard/@analytics/default.tsx
export default function Default() {
  return null
}
```

**Intercepting routes** render a different route *inside the current layout* without a full navigation — the "click a photo in a grid, get a modal, but the URL is the photo's real URL and a refresh shows the full page" pattern.

| Convention | Intercepts |
|---|---|
| `(.)folder` | same level |
| `(..)folder` | one level up |
| `(..)(..)folder` | two levels up |
| `(...)folder` | from the app root |

```
app/
├── feed/page.tsx                      the grid
├── photo/[id]/page.tsx                the full page (direct visit / refresh)
└── feed/
    ├── @modal/
    │   ├── default.tsx                → null
    │   └── (..)photo/[id]/page.tsx    the modal, shown on client navigation
    └── layout.tsx                     renders {children} and {modal}
```

The mental model: **intercepting only applies to client-side navigation.** A hard load of `/photo/123` renders the real page. That's the whole point — shareable URLs, modal UX.

> **Real Scenario — the parallel route that 404'd the whole page.**
> A team added an `@modal` slot for a photo lightbox. It worked in dev. In production, navigating to any *other* dashboard route rendered a 404 for the entire page.
> **Cause:** when the router has no match for a slot at the current URL, it renders that slot's `default.tsx`. Theirs called `notFound()` — copied from the docs' example of "preserve previous behaviour" — and `notFound()` in a slot 404s the route, not just the slot.
> **Fix:** `return null` from `default.tsx` unless a missing slot genuinely means the page doesn't exist.

### 2.5 Cheat sheet: Part 2

| Convention | Effect |
|---|---|
| `folder/` | URL segment |
| `(folder)` | route group — no URL segment |
| `_folder` | private — not routable at all |
| `[slug]` | one dynamic segment |
| `[...slug]` | catch-all (one or more) |
| `[[...slug]]` | optional catch-all (zero or more) |
| `@slot` | parallel route slot (needs `default.tsx`) |
| `(.)folder` | intercept same level |
| `(..)folder` | intercept one level up |
| `(...)folder` | intercept from root |

| Navigation helper | From | Notes |
|---|---|---|
| `redirect(path)` | `next/navigation` | 307; throws — code after doesn't run |
| `permanentRedirect(path)` | `next/navigation` | 308 |
| `notFound()` | `next/navigation` | renders nearest `not-found.tsx` |
| `useRouter()` | **`next/navigation`** | not `next/router` — that's Pages Router |
| `usePathname()`, `useSearchParams()`, `useParams()` | `next/navigation` | Client Components only |

[↑ Back to top](#table-of-contents)

---

## Part 3: Server and Client Components

<a id="part-3-server-and-client-components"></a>

This is the Part that makes the rest of the framework make sense. Budget real time for it.

### 3.1 Beginner — the default is the server

**Every component in `app/` is a Server Component unless a `"use client"` directive puts it on the other side of the boundary** ([Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components)).

A Server Component:
- runs on the server, at build time or request time, **never in the browser**;
- can be `async` and `await` anything — a database, a file, an HTTP call;
- can read secrets (`process.env.DATABASE_URL`) because its code never ships;
- ships **zero JavaScript** to the client for itself;
- cannot use `useState`, `useEffect`, event handlers, or browser APIs.

A Client Component is a normal React component, prerendered to HTML on the server and then hydrated in the browser — so it can do all the interactive things, and its code does ship.

```tsx
// app/posts/[id]/page.tsx — Server Component (no directive needed)
import { getPost } from '@/lib/data'
import LikeButton from './like-button'

export default async function Page(props: PageProps<'/posts/[id]'>) {
  const { id } = await props.params
  const post = await getPost(id) // runs on the server; credentials stay here

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
      <LikeButton postId={post.id} initialLikes={post.likes} />
    </article>
  )
}
```

```tsx
// app/posts/[id]/like-button.tsx — Client Component
'use client'

import { useState } from 'react'

export default function LikeButton({
  postId,
  initialLikes,
}: {
  postId: string
  initialLikes: number
}) {
  const [likes, setLikes] = useState(initialLikes)
  return <button onClick={() => setLikes(likes + 1)}>♥ {likes}</button>
}
```

The rule of thumb: **`"use client"` goes on the leaves, not the trunk.** Fetch and compose on the server; mark the small interactive pieces.

### 3.2 Working Knowledge — the boundary is a module graph

Here is the sentence that resolves most confusion:

> Once a file is marked with `"use client"`, **all of its imports and the components it directly renders are included in the client bundle**. — [Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components#using-client-components)

`"use client"` is not "render this on the client." It is **an entry point into the client module graph**. Everything reachable by `import` from that file crosses over with it.

```
   SERVER MODULE GRAPH                 │   CLIENT MODULE GRAPH
                                       │
   app/page.tsx (RSC)                  │
     ├── import { db } from 'lib/db' ──┼──  stays server-side ✓
     ├── import Sidebar (RSC)          │
     └── import Search ────────────────┼──► app/search.tsx  "use client"
                                       │      ├── import { formatDate }  ← ships
                                       │      ├── import lodash          ← ships
                                       │      └── <Icon />               ← ships
                                       │
   ──────────────────────────────────  │  ──────────────────────────────────
   children / props passed IN are      │  Rendered on the SERVER, sent as
   NOT part of this graph:             │  already-rendered output. No JS.
     <Modal><Cart /></Modal>           │
```

Three consequences you will hit this week:

**1. You don't repeat the directive.** A Client Component importing another component does not need `"use client"` in the child — it's already inside the client graph. Adding it everywhere is harmless but noisy.

**2. A utility imported by a Client Component ships to the browser.** If `lib/format.ts` imports a 300 KB date library and one `"use client"` file imports `format`, that library is now in your bundle even though ten Server Components also use it.

**3. Props crossing the boundary must be serializable.** Functions, class instances, `Date`-in-a-`Map`, symbols — none of it crosses ([Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components#passing-data-from-server-to-client-components)). The exception is a Server Action, which React serializes as a reference (see [Part 6](./nextjs-mutations-api.md#part-6-mutations-server-actions-and-forms)).

```tsx
// ✗ Runtime error: functions cannot be passed to Client Components.
<ClientTable rows={rows} formatCell={(v) => v.toUpperCase()} />

// ✓ Pass data; let the client own its own functions.
<ClientTable rows={rows} />

// ✓ Or pass a Server Action — a reference, which IS serializable.
<ClientTable rows={rows} onDeleteAction={deleteRow} />
```

**Third-party components without the directive.** A package using `useState` but shipping no `"use client"` errors when you render it from a Server Component. Wrap it once:

```tsx
// app/carousel.tsx
'use client'
import { Carousel } from 'acme-carousel'
export default Carousel
```

Now Server Components can render `<Carousel />` ([Third-party components](https://nextjs.org/docs/app/getting-started/server-and-client-components#third-party-components)).

**Context providers.** React context does not exist in Server Components. Create a Client provider that takes `children`, then render it in a (server) layout:

```tsx
// app/theme-provider.tsx
'use client'
import { createContext } from 'react'

export const ThemeContext = createContext({})

export default function ThemeProvider({ children }: { children: React.ReactNode }) {
  return <ThemeContext.Provider value="dark">{children}</ThemeContext.Provider>
}
```

```tsx
// app/layout.tsx — still a Server Component
import ThemeProvider from './theme-provider'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <ThemeProvider>{children}</ThemeProvider>
      </body>
    </html>
  )
}
```

This works — and `{children}` stays server-rendered — because of the composition rule in §3.3. Render providers **as deep as possible**; wrapping the whole document limits what Next.js can prerender statically ([Context providers](https://nextjs.org/docs/app/getting-started/server-and-client-components#context-providers)).

### 3.3 Advanced — composition patterns that keep the boundary small

**The children slot.** The most important pattern in the App Router:

> Components passed as `children` or other props are **not** imported into the Client Component's module graph. They're rendered on the server and passed to the Client Component as rendered output. — [Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components#using-client-components)

```tsx
// app/ui/modal.tsx
'use client'
import { useState } from 'react'

export default function Modal({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false)
  return (
    <>
      <button onClick={() => setOpen(true)}>Open cart</button>
      {open && <div className="modal">{children}</div>}
    </>
  )
}
```

```tsx
// app/page.tsx — Server Component
import Modal from './ui/modal'
import Cart from './ui/cart' // async Server Component that hits the database

export default function Page() {
  return (
    <Modal>
      <Cart />
    </Modal>
  )
}
```

`<Cart />` is an async Server Component hitting the database, rendered **inside** a stateful Client Component, and its code never ships. That is the whole design in one snippet.

**Wrong vs. right — the same intent, two bundle sizes:**

```tsx
// ✗ Modal imports Cart, so Cart, its imports, and its data-fetching
//    logic are all pulled into the client graph — and the db import breaks the build.
'use client'
import Cart from './cart'
export default function Modal() {
  const [open, setOpen] = useState(false)
  return open ? <Cart /> : null
}

// ✓ Modal accepts children. Cart stays on the server.
'use client'
export default function Modal({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false)
  return open ? <>{children}</> : null
}
```

**Moving the boundary down.** A layout that is mostly static but has one interactive piece should stay a Server Component:

```tsx
// ✗ One search box turns the entire layout — logo, nav, footer — into client JS.
'use client'
export default function Layout({ children }) { /* ... */ }

// ✓ Only <Search /> is a Client Component.
import Search from './search'   // 'use client'
import Logo from './logo'       // Server Component

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <>
      <nav><Logo /><Search /></nav>
      <main>{children}</main>
    </>
  )
}
```

> **Real Scenario — the 1.2 MB dashboard.**
> A team's dashboard shipped 1.2 MB of JavaScript for a page that was 90% static tables. Every route felt sluggish on mobile.
> **Cause:** `app/(app)/layout.tsx` had `"use client"` at the top — added months earlier so a `useEffect` could run an analytics ping. That single directive pulled the layout, the nav, the sidebar, every icon, the chart library imported by a sibling, and the whole formatting utility module into the client graph. None of it needed to be there.
> **Diagnosis:** `@next/bundle-analyzer` showed the chart library in the shared chunk even on routes with no charts.
> **Fix:** the directive was deleted, and the analytics ping moved into a three-line `<AnalyticsPing />` Client Component rendered by the layout. Shared JS dropped to 180 KB.
> **The lesson:** `"use client"` at the top of a layout is almost always a mistake. It is the highest-leverage line in the codebase, in the wrong direction.

**Streaming a promise across the boundary.** You don't always have to `await` on the server. Pass the promise and let the client unwrap it with `use()` ([Fetching data](https://nextjs.org/docs/app/getting-started/fetching-data#streaming-data-with-the-use-api)):

```tsx
// app/blog/page.tsx — note: NOT awaited
import { Suspense } from 'react'
import Posts from '@/app/ui/posts'

export default function Page() {
  const posts = getPosts()
  return (
    <Suspense fallback={<div>Loading…</div>}>
      <Posts posts={posts} />
    </Suspense>
  )
}
```

```tsx
// app/ui/posts.tsx
'use client'
import { use } from 'react'

export default function Posts({ posts }: { posts: Promise<Post[]> }) {
  const all = use(posts)
  return <ul>{all.map((p) => <li key={p.id}>{p.title}</li>)}</ul>
}
```

This is the right shape when the *client* needs the data for interactivity but the *server* should start the request. More on the streaming mechanics in [Part 4](./nextjs-data-caching.md#part-4-data-fetching-and-streaming).

### 3.4 Mastery — the RSC Payload and environment poisoning

**What actually crosses the wire.** On the server, Next.js renders Server Components into the **RSC Payload** — a compact serialised description of the rendered tree, containing ([Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components#on-the-server)):

- the rendered output of Server Components;
- placeholders marking where Client Components go, plus references to their JS chunks;
- any props passed from a Server Component to a Client Component.

```
  FIRST LOAD                      SUBSEQUENT NAVIGATION
  ──────────                      ─────────────────────
  1. HTML       → paints          RSC Payload only.
     (non-interactive preview)    Client Components render on the
  2. RSC Payload → reconciles     client; no server-rendered HTML.
     server + client trees        Shared layouts are not re-fetched.
  3. JS         → hydrates
```

Two practical consequences:

**Props to Client Components are visible to the user.** They are serialised into the payload, which is plain text in the network tab. `<UserMenu user={fullUserRecordFromDb} />` publishes that record's password hash. Shape props to what the UI renders — that's the DTO discipline in [Part 9](./nextjs-database-auth.md#part-9-authentication-and-sessions).

**Conditional rendering in a layout is not access control.** Route segments and parallel slots are rendered by the router, so a layout that hides them doesn't stop them from running or from appearing in the RSC Payload ([Authentication — layouts and auth checks](https://nextjs.org/docs/app/guides/authentication#layouts-and-auth-checks)).

**Environment poisoning.** Modules are shared between both graphs, so a server-only utility *can* be imported by a Client Component. Next.js replaces non-`NEXT_PUBLIC_` env vars with `''` in client bundles, so the code fails rather than leaking — but failing at runtime is a poor safety net. Make it a build error:

```ts
// lib/db/index.ts
import 'server-only' // any client import of this module now fails the build

export const db = drizzle(process.env.DATABASE_URL!)
```

There is a matching `client-only` package for modules that touch `window`. Installing either is optional — Next.js handles these imports internally and provides its own type declarations — but install them if your lint rules flag extraneous dependencies ([Preventing environment poisoning](https://nextjs.org/docs/app/getting-started/server-and-client-components#preventing-environment-poisoning)).

> **Real Scenario — the API key in the bundle.**
> A `lib/analytics.ts` module read `process.env.ANALYTICS_KEY` and was imported by both a Server Component and, after a refactor, a `"use client"` chart component. Nobody noticed; the chart worked because the key was only used on the server path.
> Then someone renamed the variable to `NEXT_PUBLIC_ANALYTICS_KEY` to "fix" an `undefined` in the browser console. The key was now inlined into the client bundle and shipped to every visitor. It stayed there for two deploys because `NEXT_PUBLIC_` values are **frozen at build time** — even rotating the key didn't help until a rebuild.
> **Prevention:** `import 'server-only'` at the top of `lib/analytics.ts` would have made the client import a build error on the day of the refactor, and the `NEXT_PUBLIC_` rename would never have been attempted.

**When the boundary genuinely has to move up.** Some libraries — drag-and-drop, canvas, certain animation libraries — need a client-side provider high in the tree. The cost is real but bounded: wrap as narrow a subtree as the library allows, and keep `{children}` flowing through it so the content underneath stays server-rendered.

### 3.5 Cheat sheet: Part 3

| Capability | Server Component | Client Component |
|---|---|---|
| `async` / `await` at top level | ✅ | ❌ (use `use()`) |
| `useState`, `useEffect`, `useRef` | ❌ | ✅ |
| `onClick` and other event handlers | ❌ | ✅ |
| `window`, `localStorage`, `navigator` | ❌ | ✅ |
| Direct database / ORM access | ✅ | ❌ |
| Read non-`NEXT_PUBLIC_` env vars | ✅ | ❌ |
| React Context (`createContext`/`useContext`) | ❌ | ✅ |
| Ships JavaScript to the browser | ❌ | ✅ |
| `cookies()` / `headers()` | ✅ | ❌ |
| Can render a Server Component as a child | ✅ | ✅ **only via props/children** |

| Symptom | Likely cause |
|---|---|
| "You're importing a component that needs `useState`" | Client-only code reached from a Server Component — add `"use client"` or wrap it |
| "Functions cannot be passed to Client Components" | Non-serializable prop crossing the boundary |
| "`useRouter` only works in a Client Component" | Missing `"use client"`, or imported from `next/router` |
| Bundle much larger than expected | `"use client"` too high in the tree — check with `@next/bundle-analyzer` |
| Secret visible in the browser | `NEXT_PUBLIC_` prefix, or a prop serialised into the RSC Payload |
| `params.slug` is `undefined` | Not awaited — `params` is a Promise in 16 |

[↑ Back to top](#table-of-contents)

---

**Next:** [Data & Caching (Parts 4–5)](./nextjs-data-caching.md) — reading data, streaming it, and deciding what gets cached.
