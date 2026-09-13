# Next.js — Mastery Guide (Overview)

The entry point for this repo's Next.js material. This document holds the shared spec, the setup you need before running anything, the map of the focused sub-docs, the Next.js 15 → 16 delta, the Pages Router translation table, and the reference apparatus (learning order, self-check, cross-doc cheat sheet).

The actual teaching lives in the sub-docs. Start here, then follow the [Suggested Learning Order](#8-suggested-learning-order).

---

## About This Document

This section is the canonical spec for **every doc in this set** — the sub-docs each carry a short pointer back here rather than repeating it.

- **Framing: goal-driven.** The goal is **building a full-stack Next.js project**. Depth, examples and prioritisation are aimed at the files, APIs, flags and config you will actually touch: `app/` conventions, `next.config.ts`, Server Components, Server Actions, a Postgres schema, a session cookie, a Dockerfile. Where a concept only matters in production, it is taught as a **Real Scenario** — a concrete incident, not an abstraction.
- **Confirmed scope: App Router only, end to end.** Project setup → routing → the server/client boundary → data fetching and streaming → caching and revalidation → mutations via Server Actions → Route Handlers → database layer → auth and sessions → navigation, metadata and SEO → styling and assets → production concerns → self-hosted deployment and framework internals.
- **The stack this set commits to.** Parts 8–9 need real code, not three abstract options, so the set commits to **PostgreSQL + [Drizzle ORM](https://orm.drizzle.team) + [Auth.js v5](https://authjs.dev) (`next-auth@beta`)**. Everything outside those two Parts is stack-agnostic; where the choice would change your code (Prisma's generated client, Supabase's RLS, Better Auth's adapters) the doc says so and points elsewhere.
- **Deployment stance: self-hosted first.** Part 13 is written against **`output: 'standalone'` + Docker + a Node server behind a reverse proxy**. Vercel is covered as the contrast case — specifically, which behaviours you get for free there and must configure yourself when self-hosting (the ISR cache, `revalidateTag` across instances, image optimisation, streaming through nginx, Server Action encryption keys).
- **Deliberately excluded** (with pointers): the Pages Router as a thing to *write* — see the [translation table](#6-pages-router-translation-table) so you can read it; testing; internationalisation; Draft Mode / CMS preview; React Native; and React itself. See [Deliberately Not Covered](#7-deliberately-not-covered-and-where-to-look-instead).
- **Depth tiers.** Each Part runs **Beginner → Working Knowledge → Advanced → Mastery**, so you can stop at whichever level today's task needs. Tiers are collapsed where one would be filler, and every collapse is stated inline — collapsing is expected behaviour, not a defect to fix on a later pass. The weight of this set sits in Working Knowledge and Advanced; Mastery concentrates in [Data & Caching](./nextjs-data-caching.md) and [Production & Deployment](./nextjs-production-deployment.md).
- **Assumed background: this repo's React, TypeScript and CSS sets.** You are comfortable with hooks, component composition, referential identity, and modern TypeScript. This set does **not** re-teach React — see [`react/`](../react/react-mastery-guide.md), [`typescript/`](../typescript/typescript-mastery-guide.md), [`tailwindcss/`](../tailwindcss/tailwindcss-mastery-guide.md) and [`css/`](../css/css-mastery-guide.md). The **one exception** is React Server Components, which [`react/` explicitly excludes](../react/react-mastery-guide.md#6-deliberately-not-covered-and-where-to-look-instead) and which this set therefore teaches from scratch in [Part 3](./nextjs-routing-rendering.md#part-3-server-and-client-components).
- **Examples are TypeScript.** Types are minimal and idiomatic — enough to be realistic, never so much that type noise obscures the Next.js concept.
- **Per-section ingredients**, used where they earn their place rather than forced into every subsection:
  - a short plain-language explanation of what a thing is and why it matters;
  - a **wrong vs. right** snippet pair wherever a gotcha has a common wrong form;
  - a **Real Scenario** — a production-style incident where the concept actually bites (this is a goal-driven set, so incidents replace the curiosity-driven "Try It" exercises);
  - an **ASCII diagram** where the concept is spatial or structural — the request pipeline, the module graph split at `"use client"`, the cache layers, the container topology.
- **Version stance: Next.js 16 only, with a 15 → 16 delta Part.** Every example is written for Next.js 16. Next.js 15 and earlier are covered **only** in [Part 14](#5-part-14-nextjs-15--16) as a migration and code-reading reference, because 15 → 16 changed things that every pre-2026 blog post still gets wrong (`middleware.ts` → `proxy.ts`, sync `params`, implicit caching, `next lint`, `error.tsx`'s `reset` prop).
- **Sourcing rule.** [nextjs.org/docs](https://nextjs.org/docs) is the primary source and is cited inline at the specific claim it supports. Stack Overflow and GitHub discussions are used for real-world gotchas and how practitioners actually resolve them. Where a popular-but-outdated convention disagrees with the official docs, the official docs win and the doc says so explicitly. No URL appears in this set that was not actually retrieved.
- **Written against.** **Next.js 16.3.4** (latest stable, released Aug 31 2026 — [support policy](https://nextjs.org/support-policy)), React 19.2 canary as bundled by the App Router, **Node.js 20.9+**, **TypeScript 5.1+**, `drizzle-orm` **0.45.2**, `next-auth` **5.0.0-beta.32**, Docker with Node 24 LTS images. Verified **September 11, 2026**.
- **To update this doc later.** *Preserve*: the goal-driven framing, the numbered Part structure with explicit tier subheadings, the wrong-vs-right pairs, the Real Scenario incidents, the TOC/back-to-top/anchor apparatus, and the per-Part cheat sheets. *Re-verify against current docs first*: the Next.js patch version and everything in [Part 14](#5-part-14-nextjs-15--16) (deprecations move — `middleware.ts` still works but is slated for removal, and `revalidateTag`'s single-argument form is deprecated, not gone); whether **Cache Components** (`cacheComponents: true`) has become the default, which is the single most likely thing in this set to go stale; whether **`next-auth` v5 has finally shipped stable** (it was still `@beta` at the time of writing — see [Part 9](./nextjs-database-auth.md#part-9-authentication-and-sessions)); whether **Drizzle 1.0** has shipped stable (the official get-started page already instructs `@rc`); and the Node LTS version in the Dockerfile.

---

## Table of Contents

1. [How This Set Is Organised](#1-how-this-set-is-organised)
2. [The Sub-Docs](#2-the-sub-docs)
3. [Orientation & Setup](#3-orientation--setup)
4. [The One-Paragraph Mental Model](#4-the-one-paragraph-mental-model)
5. [Part 14: Next.js 15 → 16](#5-part-14-nextjs-15--16)
6. [Pages Router Translation Table](#6-pages-router-translation-table)
7. [Deliberately Not Covered (and where to look instead)](#7-deliberately-not-covered-and-where-to-look-instead)
8. [Suggested Learning Order](#8-suggested-learning-order)
9. [Quick Self-Check](#9-quick-self-check)
10. [Cross-Doc Cheat Sheet](#10-cross-doc-cheat-sheet)

---

## 1. How This Set Is Organised

Next.js is 13 Parts of teaching material spread over 6 sub-docs, plus this overview. The split follows the order in which the concepts become necessary when you actually build the thing:

```
  nextjs-mastery-guide.md   ← you are here: spec, setup, map, 15→16 delta, reference
        │
        ├── nextjs-routing-rendering.md      Parts 1–3    what the framework renders
        ├── nextjs-data-caching.md           Parts 4–5    where the data comes from
        ├── nextjs-mutations-api.md          Parts 6–7    how the data changes
        ├── nextjs-database-auth.md          Parts 8–9    who the data belongs to
        ├── nextjs-navigation-assets.md      Parts 10–11  how it looks and is found
        └── nextjs-production-deployment.md  Parts 12–13  how it survives production
```

Every Part is numbered globally (1–13, plus Part 14 in this document), so a cross-reference like "see Part 5" is unambiguous no matter which file you are in.

[↑ Back to top](#table-of-contents)

---

## 2. The Sub-Docs

| Doc | Parts | What it answers |
|---|---|---|
| [Routing & Rendering](./nextjs-routing-rendering.md) | 1–3 | How do I create a project, turn folders into URLs, and decide what runs on the server? |
| [Data & Caching](./nextjs-data-caching.md) | 4–5 | How do I read data, stream it, and control what is cached and for how long? |
| [Mutations & API Layer](./nextjs-mutations-api.md) | 6–7 | How do I write data from a form, and when do I need a real HTTP endpoint instead? |
| [Database & Auth](./nextjs-database-auth.md) | 8–9 | How do I connect Postgres safely, and how do I know who is asking? |
| [Navigation & Assets](./nextjs-navigation-assets.md) | 10–11 | How do transitions stay fast, and how do images, fonts and metadata work? |
| [Production & Deployment](./nextjs-production-deployment.md) | 12–13 | What breaks when I put this in a container behind nginx, and why? |

**Part index**

1. Project Setup & the `app` Directory — [Routing & Rendering](./nextjs-routing-rendering.md#part-1-project-setup-and-the-app-directory)
2. Routing: Layouts, Pages, Segments — [Routing & Rendering](./nextjs-routing-rendering.md#part-2-routing)
3. Server and Client Components — [Routing & Rendering](./nextjs-routing-rendering.md#part-3-server-and-client-components)
4. Data Fetching & Streaming — [Data & Caching](./nextjs-data-caching.md#part-4-data-fetching-and-streaming)
5. Caching & Revalidation — [Data & Caching](./nextjs-data-caching.md#part-5-caching-and-revalidation)
6. Mutations: Server Actions & Forms — [Mutations & API](./nextjs-mutations-api.md#part-6-mutations-server-actions-and-forms)
7. Route Handlers & the API Layer — [Mutations & API](./nextjs-mutations-api.md#part-7-route-handlers)
8. The Database Layer — [Database & Auth](./nextjs-database-auth.md#part-8-the-database-layer)
9. Authentication & Sessions — [Database & Auth](./nextjs-database-auth.md#part-9-authentication-and-sessions)
10. Navigation, Metadata & SEO — [Navigation & Assets](./nextjs-navigation-assets.md#part-10-navigation-metadata-and-seo)
11. Styling, Images & Fonts — [Navigation & Assets](./nextjs-navigation-assets.md#part-11-styling-images-and-fonts)
12. Production Concerns — [Production & Deployment](./nextjs-production-deployment.md#part-12-production-concerns)
13. Deployment & Internals — [Production & Deployment](./nextjs-production-deployment.md#part-13-deployment-and-internals)
14. Next.js 15 → 16 — [this document](#5-part-14-nextjs-15--16)

[↑ Back to top](#table-of-contents)

---

## 3. Orientation & Setup

Everything in this set assumes a project created this way. Create it once and keep it open while you read.

```bash
npx create-next-app@latest my-app --yes
cd my-app
npm run dev
```

`--yes` takes the recommended defaults: **TypeScript, ESLint, Tailwind CSS, App Router, Turbopack**, import alias `@/*`, and an `AGENTS.md` that points coding agents at version-matched docs ([Installation](https://nextjs.org/docs/app/getting-started/installation)).

**System requirements** ([Installation](https://nextjs.org/docs/app/getting-started/installation#system-requirements)):

| Requirement | Minimum |
|---|---|
| Node.js | 20.9 (Node 18 is no longer supported) |
| TypeScript | 5.1 |
| Browsers | Chrome/Edge/Firefox 111+, Safari 16.4+ |

The scripts you will actually run:

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint",
    "lint:fix": "eslint --fix"
  }
}
```

> **Outdated convention alert.** Blog posts from 2024–2025 tell you to write `next dev --turbopack`. Turbopack is the **default bundler** in 16 for both `dev` and `build`; the flag is a no-op. The flag that matters now is the opt-*out*: `next build --webpack` ([Upgrade guide](https://nextjs.org/docs/app/guides/upgrading/version-16#turbopack-by-default)). They also tell you to run `next lint` — that command was **removed** in 16, and `next build` no longer lints ([Upgrade guide](https://nextjs.org/docs/app/guides/upgrading/version-16#next-lint-command)).

For the database and auth Parts you will also need:

```bash
npm i drizzle-orm pg
npm i -D drizzle-kit tsx @types/pg
npm i next-auth@beta zod
npx auth secret
```

A local Postgres is easiest as a container:

```bash
docker run --name app-pg -e POSTGRES_PASSWORD=dev -p 5432:5432 -d postgres:17
```

[↑ Back to top](#table-of-contents)

---

## 4. The One-Paragraph Mental Model

A Next.js App Router request walks a fixed pipeline, and almost every bug you will hit is a misunderstanding of *where in this pipeline* your code ran.

```
   request
      │
      ▼
  ┌──────────┐   proxy.ts — runs on every matched request, Node runtime.
  │  proxy   │   Rewrites, redirects, header/cookie edits. No slow data access.
  └────┬─────┘
       ▼
  ┌────────────────────────────────────────────────────────────┐
  │  route match: app/**/layout.tsx + page.tsx  (or route.ts)  │
  └────┬───────────────────────────────────────────────────────┘
       ▼
  ┌──────────────────────────────────────────┐
  │  SERVER render (React Server Components) │  async/await, db, secrets
  │  ──────────────────────────────────────  │  produces the RSC Payload
  │   "use client" modules are NOT run here  │
  │   — they become placeholders + JS refs   │
  └────┬─────────────────────────────────────┘
       ▼  RSC Payload  ──►  prerendered HTML (+ streamed Suspense chunks)
  ┌──────────────────────────────────────────┐
  │  CLIENT: HTML paints, then hydration     │  useState, onClick, browser APIs
  │  runs Client Components                  │
  └────┬─────────────────────────────────────┘
       ▼
  user submits a form / clicks
       ▼
  ┌──────────────────────────────────────────┐
  │  Server Action (POST back to this route) │  mutate → revalidate → re-render
  │  response carries return value + new RSC │  in ONE roundtrip
  └──────────────────────────────────────────┘
```

Two consequences are worth internalising before you read any further. First, **the server/client split is a module-graph split, not a runtime toggle** — `"use client"` marks a boundary, and everything imported below it ships to the browser. Second, **a Server Action is a public POST endpoint** whose implementation happens to live next to your UI; the framework gives you a CSRF check and encrypted action IDs, but authorisation is yours ([Server Actions](https://nextjs.org/docs/app/guides/server-actions#security)).

[↑ Back to top](#table-of-contents)

---

## 5. Part 14: Next.js 15 → 16

This Part exists so you can read the wild. Next.js 16 (released October 2025, currently 16.3.4) is the version this set teaches; almost every tutorial, Stack Overflow answer and AI-generated snippet you will meet was written for 15 or earlier. This table is the complete list of things that will mislead you.

### 14.1 Breaking changes that change the code you write

| Next.js 15 (what you'll see in blog posts) | Next.js 16 (what to write) | Source |
|---|---|---|
| `middleware.ts`, `export function middleware()` | `proxy.ts`, `export function proxy()` — Node runtime only, not configurable | [Upgrade](https://nextjs.org/docs/app/guides/upgrading/version-16#middleware-to-proxy) |
| `skipMiddlewareUrlNormalize` | `skipProxyUrlNormalize` | [Upgrade](https://nextjs.org/docs/app/guides/upgrading/version-16#middleware-to-proxy) |
| `const { id } = params` (sync worked with a warning) | `const { id } = await params` — sync access **removed** | [Upgrade](https://nextjs.org/docs/app/guides/upgrading/version-16#async-request-apis-breaking-change) |
| `cookies()`, `headers()`, `draftMode()` sync | All three are async: `await cookies()` | [Upgrade](https://nextjs.org/docs/app/guides/upgrading/version-16#async-request-apis-breaking-change) |
| `export default function Error({ error, reset })` | `({ error, retry })` — the prop is now `retry` | [Error handling](https://nextjs.org/docs/app/getting-started/error-handling#nested-error-boundaries) |
| `revalidateTag('posts')` | `revalidateTag('posts', 'max')` — profile now required; single-arg deprecated | [Upgrade](https://nextjs.org/docs/app/guides/upgrading/version-16#revalidatetag) |
| `experimental.ppr: true`, `export const experimental_ppr` | `cacheComponents: true` — both PPR flags **removed** | [Upgrade](https://nextjs.org/docs/app/guides/upgrading/version-16#partial-prerendering-ppr) |
| `experimental.dynamicIO`, `experimental.useCache` | `cacheComponents: true` | [Upgrade](https://nextjs.org/docs/app/guides/upgrading/version-16#experimentaldynamicio-and-experimentalusecache) |
| `unstable_cacheLife`, `unstable_cacheTag` | `cacheLife`, `cacheTag` — now stable | [Upgrade](https://nextjs.org/docs/app/guides/upgrading/version-16#cachelife-and-cachetag) |
| `experimental.turbopack: {}` | top-level `turbopack: {}` | [Upgrade](https://nextjs.org/docs/app/guides/upgrading/version-16#turbopack-configuration-location) |
| `serverRuntimeConfig` / `publicRuntimeConfig` | **Removed.** Use env vars + `connection()` for runtime reads | [Upgrade](https://nextjs.org/docs/app/guides/upgrading/version-16#runtime-configuration) |
| `next lint`, `eslint: {}` in config | **Removed.** Run `eslint` directly; flat config is the default | [Upgrade](https://nextjs.org/docs/app/guides/upgrading/version-16#next-lint-command) |
| `images.domains: ['x.com']` | `images.remotePatterns` (domains deprecated) | [Upgrade](https://nextjs.org/docs/app/guides/upgrading/version-16#imagesdomains-configuration-deprecated) |
| Parallel-route slots worked without `default.js` | Every `@slot` needs an explicit `default.js` or the **build fails** | [Upgrade](https://nextjs.org/docs/app/guides/upgrading/version-16#parallel-routes-defaultjs-requirement) |
| `unstable_rootParams()` | Removed; use `next/root-params` | [Upgrade](https://nextjs.org/docs/app/guides/upgrading/version-16#unstable_rootparams) |
| AMP (`useAmp`, `config = { amp: true }`) | Removed entirely | [Upgrade](https://nextjs.org/docs/app/guides/upgrading/version-16#amp-support) |

### 14.2 Default behaviour changes (your code compiles; the behaviour differs)

| Default | Was | Is now |
|---|---|---|
| Bundler | webpack (Turbopack opt-in via `--turbopack`) | **Turbopack**; opt out with `--webpack` |
| `images.minimumCacheTTL` | 60s | **14400s (4 hours)** |
| `images.qualities` | all of 1–100 | **`[75]`**; other values coerce to the nearest allowed |
| `images.imageSizes` | included `16` | `16` removed |
| `images.maximumRedirects` | unlimited | **3** |
| Local-IP image optimisation | allowed | blocked; `dangerouslyAllowLocalIP` to re-enable |
| `scroll-behavior: smooth` override | Next.js forced `auto` during transitions | no longer overridden; opt in with `data-scroll-behavior="smooth"` on `<html>` |
| `next dev` / `next build` output | shared `.next` | separate dirs (`.next/dev`), so they can run concurrently; a lockfile prevents duplicate instances |
| `next build` output metrics | printed `size` / `First Load JS` | removed as inaccurate for RSC; use Lighthouse instead |

Full list: [Next.js 16 blog post](https://nextjs.org/blog/next-16#breaking-changes-and-other-updates).

### 14.3 The two caching models — the thing most likely to confuse you

Next.js 16 ships **two** caching models, and which one you are in changes what the APIs mean. This set teaches both, in [Part 5](./nextjs-data-caching.md#part-5-caching-and-revalidation).

```
  cacheComponents: false          │  cacheComponents: true
  (the default in 16.3)           │  ("Cache Components", opt-in)
 ─────────────────────────────────┼──────────────────────────────────────
  fetch uncached by default       │  everything uncached by default
  opt in: { cache: 'force-cache' }│  opt in: "use cache" directive
  non-fetch: unstable_cache()     │  non-fetch: "use cache" + cacheLife()
  route segment config            │  <Suspense> is mandatory around
   (dynamic / revalidate /        │   uncached + runtime-API access,
    fetchCache) steers the route  │   enforced at build time
  static-or-dynamic per route     │  Partial Prerendering: static shell +
                                  │   streamed dynamic holes, per route
```

The trap: `use cache`, `cacheLife` and `cacheTag` **do nothing without `cacheComponents: true`**, and enabling that flag is not a rename — it surfaces build errors for any uncached read that is not behind `<Suspense>` ([migration guide](https://nextjs.org/docs/app/guides/migrating-to-cache-components)).

### 14.4 Upgrading an existing app

```bash
npx @next/codemod@canary upgrade latest       # config, proxy rename, unstable_ prefixes
npx @next/codemod@canary next-async-request-api .   # if still on sync params/cookies
npx @next/codemod@canary next-lint-to-eslint-cli .  # if you used `next lint`
```

The `upgrade` codemod does **not** run every migration — the async request API one is separate ([Upgrade guide](https://nextjs.org/docs/app/guides/upgrading/version-16#using-the-codemod)). For routine version bumps thereafter, `npx next upgrade` also refreshes the docs bundled at `node_modules/next/dist/docs/`.

[↑ Back to top](#table-of-contents)

---

## 6. Pages Router Translation Table

You will not write Pages Router code, but you will read it — in older repos, in Stack Overflow answers, and in half the tutorials that rank on Google. This table appears **once**, here, so nothing in the sub-docs has to hedge.

| Pages Router | App Router equivalent |
|---|---|
| `pages/about.tsx` | `app/about/page.tsx` |
| `pages/blog/[slug].tsx` | `app/blog/[slug]/page.tsx` |
| `pages/_app.tsx`, `pages/_document.tsx` | `app/layout.tsx` (one root layout, owns `<html>`/`<body>`) |
| `pages/api/users.ts` | `app/api/users/route.ts` with named `GET`/`POST` exports |
| `getServerSideProps` | just `await` in an `async` Server Component |
| `getStaticProps` | `await` in a Server Component + `use cache` or `fetch` caching |
| `getStaticProps` + `revalidate: 60` | `cacheLife` / `next: { revalidate: 60 }` / ISR |
| `getStaticPaths` | `generateStaticParams` |
| `res.redirect()` | `redirect()` from `next/navigation` |
| `res.status(404)` | `notFound()` from `next/navigation` |
| `next/head` `<Head>` | `export const metadata` or `generateMetadata` |
| `useRouter().query` | `await params` / `await searchParams`, or `useSearchParams()` |
| `useRouter()` from `next/router` | `useRouter()` from **`next/navigation`** (different API) |
| `_error.tsx` | `error.tsx` + `global-error.tsx` |
| API route with `req`/`res` (Node objects) | Route Handler with Web `Request`/`Response` |

Two of these are genuine traps rather than translations. `useRouter` exists in **both** `next/router` and `next/navigation` with incompatible shapes — importing the wrong one is a runtime error that reads like a framework bug. And a Route Handler receives a Web `Request`, not Node's `req`, so `req.body` is a stream, not a parsed object; you call `await request.json()`.

[↑ Back to top](#table-of-contents)

---

## 7. Deliberately Not Covered (and where to look instead)

Covering these properly would double the set; half-covering them would be worse than not covering them.

| Topic | Why it's out | Where to look |
|---|---|---|
| **React itself** — hooks, state, effects, composition | This set assumes it | [`react/`](../react/react-mastery-guide.md) |
| **TypeScript the language** | Same | [`typescript/`](../typescript/typescript-mastery-guide.md) |
| **CSS and Tailwind fundamentals** | Same; Part 11 covers only the Next.js *integration* | [`css/`](../css/css-mastery-guide.md), [`tailwindcss/`](../tailwindcss/tailwindcss-mastery-guide.md) |
| **Writing Pages Router code** | App Router is the whole scope; §6 covers reading it | [Pages Router docs](https://nextjs.org/docs/pages) |
| **Testing** (Vitest, Playwright, RSC testing) | A full topic of its own, and RSC testing is still unsettled | [Testing guide](https://nextjs.org/docs/app/guides/testing) |
| **Internationalisation / i18n routing** | Only matters if you need it, and it interacts with every Part | [i18n guide](https://nextjs.org/docs/app/guides/internationalization) |
| **Draft Mode / CMS preview** | Assumes a CMS this set doesn't pick | [Draft Mode guide](https://nextjs.org/docs/app/guides/draft-mode) |
| **Static export (`output: 'export'`)** | Excludes most of what this set teaches (Actions, Route Handlers, proxy) | [Static exports](https://nextjs.org/docs/app/guides/static-exports) |
| **Edge runtime** | `proxy.ts` is Node-only in 16, and the edge story is in flux | [Edge runtime reference](https://nextjs.org/docs/app/api-reference/edge) |
| **Deploying to Vercel specifically** | Deployment stance is self-hosted; Part 13 covers the contrast | [Vercel docs](https://vercel.com/docs/frameworks/full-stack/nextjs) |
| **MDX, analytics, Sentry, third-party scripts** | Integrations, not framework mechanics | [Guides index](https://nextjs.org/docs/app/guides) |

[↑ Back to top](#table-of-contents)

---

## 8. Suggested Learning Order

**If you are building the project now** (the goal-driven path — roughly the order you'll need things):

1. [Part 1](./nextjs-routing-rendering.md#part-1-project-setup-and-the-app-directory) and [Part 2](./nextjs-routing-rendering.md#part-2-routing) — get routes on screen. One sitting.
2. [Part 3](./nextjs-routing-rendering.md#part-3-server-and-client-components) — **do not skip or skim.** Every later confusion traces back here.
3. [Part 4](./nextjs-data-caching.md#part-4-data-fetching-and-streaming) — read real data, add `loading.tsx` and `<Suspense>`.
4. [Part 8](./nextjs-database-auth.md#part-8-the-database-layer) — schema, migrations, the connection singleton. Jumping here early is deliberate: you want real data before you tune caching.
5. [Part 6](./nextjs-mutations-api.md#part-6-mutations-server-actions-and-forms) — forms that write.
6. [Part 9](./nextjs-database-auth.md#part-9-authentication-and-sessions) — sessions, the DAL, protected routes.
7. [Part 5](./nextjs-data-caching.md#part-5-caching-and-revalidation) — now that you have real reads and writes, decide what to cache.
8. [Part 7](./nextjs-mutations-api.md#part-7-route-handlers) — only when you actually need an endpoint (webhooks, mobile clients, file uploads).
9. [Part 10](./nextjs-navigation-assets.md#part-10-navigation-metadata-and-seo) and [Part 11](./nextjs-navigation-assets.md#part-11-styling-images-and-fonts) — polish pass.
10. [Part 12](./nextjs-production-deployment.md#part-12-production-concerns) and [Part 13](./nextjs-production-deployment.md#part-13-deployment-and-internals) — before the first real deploy, not after.

**If you are inheriting an existing Next.js codebase**: [Part 14](#5-part-14-nextjs-15--16) → [§6](#6-pages-router-translation-table) → [Part 3](./nextjs-routing-rendering.md#part-3-server-and-client-components) → [Part 5](./nextjs-data-caching.md#part-5-caching-and-revalidation). Those four tell you what version conventions you are looking at, what the boundaries are, and why the data is stale.

**If you are debugging production**: [Part 12](./nextjs-production-deployment.md#part-12-production-concerns) → [Part 13](./nextjs-production-deployment.md#part-13-deployment-and-internals) → the [cross-doc cheat sheet](#10-cross-doc-cheat-sheet) below.

[↑ Back to top](#table-of-contents)

---

## 9. Quick Self-Check

If you can answer these without looking, you have the set. Each links to where the answer lives.

1. Why does adding `"use client"` to a file affect files it imports, but not components passed to it as `children`? ([Part 3](./nextjs-routing-rendering.md#part-3-server-and-client-components))
2. What exactly is in the RSC Payload, and how is it different from the HTML? ([Part 3](./nextjs-routing-rendering.md#part-3-server-and-client-components))
3. A page's `layout.tsx` awaits `cookies()` at the top level. Why doesn't `loading.tsx` show? ([Part 4](./nextjs-data-caching.md#part-4-data-fetching-and-streaming))
4. Two `await`s in one component vs. `Promise.all` — when does it actually matter? ([Part 4](./nextjs-data-caching.md#part-4-data-fetching-and-streaming))
5. Is `fetch()` cached by default in Next.js 16? What about in 14? ([Part 5](./nextjs-data-caching.md#part-5-caching-and-revalidation))
6. When do you reach for `updateTag` instead of `revalidateTag`? ([Part 5](./nextjs-data-caching.md#part-5-caching-and-revalidation))
7. Your Server Action only renders inside an authenticated page. Is that a security boundary? ([Part 6](./nextjs-mutations-api.md#part-6-mutations-server-actions-and-forms))
8. Why can't you `Promise.all` three Server Actions from a click handler? ([Part 6](./nextjs-mutations-api.md#part-6-mutations-server-actions-and-forms))
9. Server Action or Route Handler — which for a Stripe webhook, and why? ([Part 7](./nextjs-mutations-api.md#part-7-route-handlers))
10. Why does your Postgres run out of connections after 20 minutes of `next dev`? ([Part 8](./nextjs-database-auth.md#part-8-the-database-layer))
11. Why is an auth check in `layout.tsx` not sufficient protection? ([Part 9](./nextjs-database-auth.md#part-9-authentication-and-sessions))
12. Why must `proxy.ts` only read the session cookie, never hit the database? ([Part 9](./nextjs-database-auth.md#part-9-authentication-and-sessions))
13. What does `<Link>` prefetch for a dynamic route with no `loading.tsx`? ([Part 10](./nextjs-navigation-assets.md#part-10-navigation-metadata-and-seo))
14. Why did your `NEXT_PUBLIC_API_URL` not change when you promoted the same Docker image to staging? ([Part 12](./nextjs-production-deployment.md#part-12-production-concerns))
15. You run three containers behind a load balancer. Why does `revalidateTag` only fix one of them? ([Part 13](./nextjs-production-deployment.md#part-13-deployment-and-internals))
16. What is `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` for, and what breaks without it? ([Part 13](./nextjs-production-deployment.md#part-13-deployment-and-internals))
17. Your streamed page arrives all at once in production but streams fine locally. What's in between? ([Part 13](./nextjs-production-deployment.md#part-13-deployment-and-internals))

[↑ Back to top](#table-of-contents)

---

## 10. Cross-Doc Cheat Sheet

### File conventions (`app/`)

| File | Purpose |
|---|---|
| `layout.tsx` | Shared UI; preserves state across navigation; root layout owns `<html>`/`<body>` |
| `page.tsx` | Makes a segment publicly routable |
| `loading.tsx` | Auto-wraps the page in `<Suspense>` |
| `error.tsx` | Client-side error boundary; props `{ error, retry }` |
| `global-error.tsx` | Replaces the root layout on error; must render its own `<html>`/`<body>` |
| `not-found.tsx` | UI for `notFound()` |
| `route.ts` | HTTP endpoint; cannot coexist with `page.tsx` in the same segment |
| `template.tsx` | Like a layout, but remounts on navigation |
| `default.tsx` | Parallel-route fallback — **required** for every slot in 16 |
| `[slug]` / `[...slug]` / `[[...slug]]` | dynamic / catch-all / optional catch-all |
| `(group)` | Route group — organises without affecting the URL |
| `_folder` | Private folder — opted out of routing entirely |
| `@slot` | Named parallel-route slot |
| `proxy.ts` (root) | Request interception (formerly `middleware.ts`) |
| `instrumentation.ts` (root) | `register()` runs once at server start |

### Imports you will type constantly

| Import | From |
|---|---|
| `redirect`, `permanentRedirect`, `notFound`, `useRouter`, `usePathname`, `useSearchParams`, `useParams` | `next/navigation` |
| `cookies`, `headers`, `draftMode` | `next/headers` |
| `revalidatePath`, `revalidateTag`, `updateTag`, `refresh`, `cacheLife`, `cacheTag`, `unstable_cache` | `next/cache` |
| `NextRequest`, `NextResponse`, `after`, `connection` | `next/server` |
| `Link`, `useLinkStatus` | `next/link` |
| `Image` | `next/image` |
| `ImageResponse` | `next/og` |
| `catchError` | `next/error` |
| `cache`, `use`, `useActionState`, `useOptimistic`, `useTransition` | `react` |

### Config flags worth memorising

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  output: 'standalone',        // Part 13 — minimal Docker image
  cacheComponents: false,      // Part 5  — true enables "use cache" + PPR
  reactCompiler: false,        // stable in 16, off by default (Babel = slower builds)
  deploymentId: process.env.DEPLOYMENT_VERSION, // Part 13 — version skew
  images: { remotePatterns: [/* ... */] },      // Part 11
  turbopack: {},               // top-level in 16, not experimental
}

export default nextConfig
```

### Which API updates what, after a mutation

| API | Invalidates cache? | Re-renders current route in the same response? | Where |
|---|---|---|---|
| `updateTag(tag)` | yes, immediately | **yes** — read-your-writes | Server Actions only |
| `revalidateTag(tag, profile)` | yes, stale-while-revalidate | **no** | Actions / Route Handlers |
| `revalidatePath(path)` | yes, by path | yes | Actions / Route Handlers |
| `refresh()` | no | yes (uncached data only) | Server Actions only |
| `router.refresh()` | no | yes | Client Components |

Source: [Server Actions — choosing a cache update](https://nextjs.org/docs/app/guides/server-actions#choosing-a-cache-update).

### Runtime APIs that force request-time rendering

`cookies()` · `headers()` · `draftMode()` · `searchParams` · uncached `params` · `connection()` · `request.url` / `request.headers` in a Route Handler · `Math.random()` / `Date.now()` / `crypto.randomUUID()` (under Cache Components).

### Diagnostic commands

```bash
next build                    # the real check — type errors + prerender validation
next typegen                  # regenerate PageProps/LayoutProps/RouteContext helpers
next build --webpack          # rule out a Turbopack-specific problem
ANALYZE=true next build       # with @next/bundle-analyzer
node .next/standalone/server.js   # run the production artifact locally
```

[↑ Back to top](#table-of-contents)
