# Next.js — Navigation & Assets (Parts 10–11)

Making transitions feel instant, getting found by search engines and link unfurlers, and shipping images and fonts without wrecking your Core Web Vitals.

> **Spec:** this doc follows the shared spec in [`nextjs-mastery-guide.md`](./nextjs-mastery-guide.md#about-this-document). Written against **Next.js 16.3.4**, verified **September 11, 2026**.

Part 11 covers the Next.js *integration* only — for CSS and Tailwind themselves see [`css/`](../css/css-mastery-guide.md) and [`tailwindcss/`](../tailwindcss/tailwindcss-mastery-guide.md).

---

## Table of Contents

- [Part 10: Navigation, Metadata and SEO](#part-10-navigation-metadata-and-seo)
  - [10.1 Beginner — `<Link>` and programmatic navigation](#101-beginner--link-and-programmatic-navigation)
  - [10.2 Working Knowledge — what prefetching actually does](#102-working-knowledge--what-prefetching-actually-does)
  - [10.3 Advanced — metadata and OG images](#103-advanced--metadata-and-og-images)
  - [10.4 Mastery — diagnosing a slow transition](#104-mastery--diagnosing-a-slow-transition)
  - [10.5 Cheat sheet: Part 10](#105-cheat-sheet-part-10)
- [Part 11: Styling, Images and Fonts](#part-11-styling-images-and-fonts)
  - [11.1 Beginner — Tailwind and CSS in the App Router](#111-beginner--tailwind-and-css-in-the-app-router)
  - [11.2 Working Knowledge — `next/image`](#112-working-knowledge--nextimage)
  - [11.3 Advanced — `next/font` and what changed in 16](#113-advanced--nextfont-and-what-changed-in-16)
  - [11.4 Cheat sheet: Part 11](#114-cheat-sheet-part-11)

---

## Part 10: Navigation, Metadata and SEO

<a id="part-10-navigation-metadata-and-seo"></a>

### 10.1 Beginner — `<Link>` and programmatic navigation

```tsx
import Link from 'next/link'

export default async function Posts() {
  const posts = await getPosts()
  return (
    <ul>
      {posts.map((p) => (
        <li key={p.slug}>
          <Link href={`/blog/${p.slug}`}>{p.title}</Link>
        </li>
      ))}
    </ul>
  )
}
```

`<Link>` extends `<a>` with prefetching and client-side transitions ([Linking and navigating](https://nextjs.org/docs/app/getting-started/linking-and-navigating)). A plain `<a>` does a full page load — state cleared, scroll reset, everything re-downloaded. Use `<a>` only for external URLs and genuine document downloads.

Programmatic navigation, from a Client Component:

```tsx
'use client'
import { useRouter } from 'next/navigation'   // NOT next/router

export function SaveButton() {
  const router = useRouter()
  return <button onClick={() => router.push('/dashboard')}>Go</button>
}
```

| Method | Effect |
|---|---|
| `router.push(href)` | Navigate, add a history entry |
| `router.replace(href)` | Navigate, replace the current entry |
| `router.back()` / `.forward()` | History |
| `router.refresh()` | Refetch the current route's RSC payload; client state preserved |

From the **server** — a Server Component, Action or Handler — use `redirect()` from `next/navigation` instead. It throws a control-flow exception, so nothing after it runs.

The three client hooks you'll reach for, all from `next/navigation` and all Client-Component-only: `usePathname()`, `useSearchParams()`, `useParams()`.

### 10.2 Working Knowledge — what prefetching actually does

`<Link>` prefetches automatically when a link enters the viewport. **What** it prefetches depends on whether the route is static ([Prefetching](https://nextjs.org/docs/app/getting-started/linking-and-navigating#prefetching)):

| Route | Prefetched |
|---|---|
| Static | the full route |
| Dynamic **with** `loading.tsx` | the shared layout + the loading skeleton (*partial* prefetch) |
| Dynamic **without** `loading.tsx` | skipped |

That last row is the whole reason "my app feels laggy" tickets exist. Without `loading.tsx`, clicking a dynamic link means waiting for a server response with **no visual feedback at all** — the app looks frozen.

```tsx
// app/blog/[slug]/loading.tsx — three lines that make the route feel instant
export default function Loading() {
  return <ArticleSkeleton />
}
```

**Next.js 16 rewrote the prefetch machinery** ([Enhanced routing and navigation](https://nextjs.org/blog/next-16#enhanced-routing-and-navigation)):

- **Layout deduplication** — 50 product links sharing a layout download that layout once, not 50 times.
- **Incremental prefetching** — only the parts not already cached are fetched; requests cancel when a link leaves the viewport and are prioritised on hover.

The visible trade-off: **more requests, much less total transfer.** If you upgraded and your network tab suddenly looks busy, that is the new behaviour working, not a bug.

**Controlling prefetch.** On a long list — an infinite-scroll table, a 500-row admin grid — prefetching every row is waste:

```tsx
// Off entirely
<Link prefetch={false} href={href}>…</Link>
```

```tsx
// Better: prefetch on hover only
'use client'
import Link from 'next/link'
import { useState } from 'react'

export function HoverPrefetchLink({ href, children }: { href: string; children: React.ReactNode }) {
  const [active, setActive] = useState(false)
  return (
    <Link href={href} prefetch={active ? null : false} onMouseEnter={() => setActive(true)}>
      {children}
    </Link>
  )
}
```

`prefetch={null}` means "default behaviour", so this flips from off to default on first hover ([Disabling prefetching](https://nextjs.org/docs/app/getting-started/linking-and-navigating#disabling-prefetching)).

**Feedback on slow networks.** When a prefetch hasn't finished, even `loading.tsx` isn't there yet. `useLinkStatus` gives you immediate feedback:

```tsx
'use client'
import { useLinkStatus } from 'next/link'

export default function LoadingIndicator() {
  const { pending } = useLinkStatus()
  return <span aria-hidden className={`link-hint ${pending ? 'is-pending' : ''}`} />
}
```

Render it inside a `<Link>`. Give it an animation delay of ~100ms starting from `opacity: 0`, so it only appears when the navigation is actually slow — otherwise it flickers on every fast click ([Slow networks](https://nextjs.org/docs/app/getting-started/linking-and-navigating#slow-networks)).

**Scroll behaviour changed in 16.** Next.js used to temporarily force `scroll-behavior: auto` during transitions so navigation felt instant even with global smooth scrolling. It no longer does. To get the old behaviour back ([Scroll behavior override](https://nextjs.org/docs/app/guides/upgrading/version-16#scroll-behavior-override)):

```tsx
<html lang="en" data-scroll-behavior="smooth">
```

And if content lands under a sticky header after navigation, that's a CSS fix — `scroll-padding-top`.

**Updating the URL without navigating.** Filters and sort orders belong in the URL, but shouldn't trigger a route transition:

```tsx
'use client'
import { useSearchParams } from 'next/navigation'

export function SortProducts() {
  const searchParams = useSearchParams()

  function updateSorting(order: string) {
    const params = new URLSearchParams(searchParams.toString())
    params.set('sort', order)
    window.history.pushState(null, '', `?${params}`)
  }
  return <button onClick={() => updateSorting('asc')}>Sort ascending</button>
}
```

`pushState`/`replaceState` integrate with the router and stay in sync with `usePathname` and `useSearchParams` ([Native History API](https://nextjs.org/docs/app/getting-started/linking-and-navigating#native-history-api)).

### 10.3 Advanced — metadata and OG images

Two APIs, both **Server-Components-only** ([Metadata and OG images](https://nextjs.org/docs/app/getting-started/metadata-and-og-images)).

**Static**, for pages whose metadata doesn't depend on data:

```tsx
// app/blog/layout.tsx
import type { Metadata } from 'next'

export const metadata: Metadata = {
  title: { default: 'My Blog', template: '%s | My Blog' },
  description: 'Notes on building things.',
  metadataBase: new URL('https://example.com'),   // makes relative OG URLs absolute
  openGraph: { type: 'website', siteName: 'My Blog' },
}
```

The `template` is the piece worth knowing: child pages set `title: 'Hello'` and get `Hello | My Blog` automatically.

**Dynamic**, via `generateMetadata`:

```tsx
// app/blog/[slug]/page.tsx
import type { Metadata } from 'next'
import { getPost } from '@/lib/data'

export async function generateMetadata(props: PageProps<'/blog/[slug]'>): Promise<Metadata> {
  const { slug } = await props.params
  const post = await getPost(slug)          // cached — see below
  if (!post) return { title: 'Not found' }

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: { title: post.title, type: 'article', publishedTime: post.createdAt.toISOString() },
  }
}
```

**Always wrap the shared fetch in `React.cache`**, or `generateMetadata` and the page component each run the same query ([Memoizing data requests](https://nextjs.org/docs/app/getting-started/metadata-and-og-images#memoizing-data-requests)):

```ts
import { cache } from 'react'
export const getPost = cache(async (slug: string) =>
  db.query.posts.findFirst({ where: eq(posts.slug, slug) })
)
```

**Streaming metadata.** On dynamically rendered pages, Next.js streams metadata separately and injects it once `generateMetadata` resolves, so visual content isn't blocked. It is **disabled for bots** that expect metadata in `<head>` — Twitterbot, Slackbot, Bingbot — detected by user agent, configurable via `htmlLimitedBots`. Prerendered pages don't stream at all, since metadata is resolved at build ([Streaming metadata](https://nextjs.org/docs/app/getting-started/metadata-and-og-images#streaming-metadata)).

**File-based metadata** — drop a file in the right place and Next.js wires it up:

| File | Where | Result |
|---|---|---|
| `favicon.ico` | `app/` | Favicon |
| `icon.png`, `apple-icon.png` | any segment | App icons |
| `opengraph-image.png`, `twitter-image.png` | any segment | Social cards; the deepest match wins |
| `robots.txt` or `robots.ts` | `app/` | robots |
| `sitemap.xml` or `sitemap.ts` | `app/` | sitemap |

```ts
// app/sitemap.ts
import type { MetadataRoute } from 'next'

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await getPublishedPosts()
  return [
    { url: 'https://example.com', lastModified: new Date(), priority: 1 },
    ...posts.map((p) => ({
      url: `https://example.com/blog/${p.slug}`,
      lastModified: p.updatedAt,
    })),
  ]
}
```

**Generated OG images** with `ImageResponse` — JSX and CSS rendered to a PNG at request or build time:

```tsx
// app/blog/[slug]/opengraph-image.tsx
import { ImageResponse } from 'next/og'
import { getPost } from '@/lib/data'

export const size = { width: 1200, height: 630 }
export const contentType = 'image/png'

export default async function Image(props: { params: Promise<{ slug: string }> }) {
  const { slug } = await props.params          // async in 16 — see below
  const post = await getPost(slug)

  return new ImageResponse(
    (
      <div
        style={{
          fontSize: 64,
          background: '#0b0b0f',
          color: 'white',
          width: '100%',
          height: '100%',
          display: 'flex',
          alignItems: 'center',
          justifyContent: 'center',
          padding: 80,
        }}
      >
        {post.title}
      </div>
    ),
    size
  )
}
```

Three constraints that trip everyone up:

1. **`params` (and `id`, if you use `generateImageMetadata`) are Promises in 16.** This was a separate breaking change from the page-level one ([Async parameters for image generation](https://nextjs.org/docs/app/guides/upgrading/version-16#async-parameters-for-icon-and-open-graph-image-breaking-change)). `sitemap`'s `id` from `generateSitemaps` is async now too.
2. **Only flexbox and a subset of CSS work.** No `display: grid`. `ImageResponse` runs on [Satori](https://github.com/vercel/satori), not a browser.
3. **Every element needs an explicit `display`** when it has multiple children, or Satori errors.

The [OG Playground](https://og-playground.vercel.app/) is the fastest way to iterate on these.

### 10.4 Mastery — diagnosing a slow transition

Transitions get slow for exactly five reasons ([What can make transitions slow?](https://nextjs.org/docs/app/getting-started/linking-and-navigating#what-can-make-transitions-slow)). Work down the list:

```
  Click feels dead (nothing happens for ~1s)
        │
        ├─ 1. Dynamic route, no loading.tsx        → add loading.tsx
        │
        ├─ 2. Dynamic segment, no generateStaticParams
        │        → prerender the known values
        │
        ├─ 3. Layout awaits at top level            → move into <Suspense>
        │        (also kills loading.tsx — see Part 4)
        │
        ├─ 4. Hydration not finished                → <Link> can't prefetch
        │        until hydrated. Cut client JS.
        │
        └─ 5. Slow network, prefetch incomplete     → useLinkStatus feedback
```

Cause 4 is the sneaky one: `<Link>` is a Client Component, so it must hydrate before it can prefetch anything. A large client bundle delays hydration, which delays prefetching, which makes the *first* navigation slow in a way that never reproduces on your machine. The fix is Part 3's fix — move `"use client"` down the tree, and check with `@next/bundle-analyzer`.

> **Real Scenario — "the search page is fast for us and slow for customers."**
> An e-commerce team's product pages loaded in ~200ms internally and 2.5s for real users. Server timings were fine. APM showed nothing.
> **Cause, three layers deep.** `/products/[slug]` had no `generateStaticParams`, so every page rendered at request time. It had no `loading.tsx`, so `<Link>` skipped prefetching entirely — nothing was ever warm. And the team's machines were on the office LAN next to the origin, so the request-time render was invisible to them.
> **Fix.** `generateStaticParams` prerendered the top 500 products at build (the long tail still renders on demand, which is fine). `loading.tsx` added a skeleton, which also enabled partial prefetching for the tail. A `useLinkStatus` indicator covered the remainder.
> **Result:** perceived navigation went to ~immediate for the catalogue's head, and to "skeleton in 50ms" for the tail.
> **The lesson:** `loading.tsx` is not a nicety. On dynamic routes it is the switch that turns prefetching on.

### 10.5 Cheat sheet: Part 10

| Need | Code |
|---|---|
| Navigate | `<Link href="/x">` |
| Navigate from a handler | `useRouter().push('/x')` (`next/navigation`) |
| Navigate from the server | `redirect('/x')` (`next/navigation`) |
| Refetch current route | `router.refresh()` / `refresh()` in an Action |
| Make a dynamic route prefetchable | add `loading.tsx` |
| Prerender dynamic URLs | `generateStaticParams()` |
| Disable prefetch | `<Link prefetch={false}>` |
| Prefetch on hover only | `prefetch={active ? null : false}` |
| Transition feedback | `useLinkStatus()` inside a `<Link>` |
| Update URL without navigating | `window.history.pushState` |
| Restore smooth-scroll override | `<html data-scroll-behavior="smooth">` |

| Metadata | Code |
|---|---|
| Static | `export const metadata: Metadata = {…}` |
| Dynamic | `export async function generateMetadata(props)` |
| Title template | `title: { default, template: '%s \| Site' }` |
| Base URL for relative OG paths | `metadataBase: new URL(...)` |
| Generated OG image | `opengraph-image.tsx` + `ImageResponse` from `next/og` |
| Sitemap / robots | `app/sitemap.ts`, `app/robots.ts` |
| Dedupe metadata + page query | `cache()` from `react` |

[↑ Back to top](#table-of-contents)

---

## Part 11: Styling, Images and Fonts

<a id="part-11-styling-images-and-fonts"></a>

This Part gets **Beginner → Advanced** only. There is no Mastery tier: the deep material is CSS and Tailwind themselves, which live in [`css/`](../css/css-mastery-guide.md) and [`tailwindcss/`](../tailwindcss/tailwindcss-mastery-guide.md), not in the framework.

### 11.1 Beginner — Tailwind and CSS in the App Router

`create-next-app` wires Tailwind up for you. The whole integration is one import in the root layout:

```tsx
// app/layout.tsx
import './globals.css'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
```

```css
/* app/globals.css — Tailwind v4 style: one import, config in CSS */
@import "tailwindcss";

@theme {
  --color-brand: oklch(0.62 0.19 264);
}
```

Other supported options, briefly:

- **CSS Modules** — `styles.module.css`, imported by the component. Scoped, works in Server Components, no runtime.
- **Global CSS** — importable from any component in the App Router, not just the root layout.
- **Sass** — supported; note that `sass-loader` moved to v16 in Next.js 16, and Turbopack does **not** support the legacy `~` prefix for `node_modules` imports ([Sass node_modules imports](https://nextjs.org/docs/app/guides/upgrading/version-16#sass-node_modules-imports)):

```scss
/* ✗ Webpack-era tilde syntax. Turbopack doesn't support it. */
@import '~bootstrap/dist/css/bootstrap.min.css';

/* ✓ */
@import 'bootstrap/dist/css/bootstrap.min.css';
```

- **CSS-in-JS** — runtime libraries (styled-components, Emotion) need a Client Component provider and don't work in Server Components. If you're starting fresh, this is a strong reason to pick Tailwind or CSS Modules instead.

### 11.2 Working Knowledge — `next/image`

`<Image>` gives you automatic resizing and modern formats, layout-shift prevention, lazy loading, and on-demand optimisation of remote images ([Image optimization](https://nextjs.org/docs/app/getting-started/images)).

**Static import — the best case.** Width, height and a blur placeholder are derived at build time:

```tsx
import Image from 'next/image'
import profile from './profile.png'

export default function Page() {
  return <Image src={profile} alt="Portrait of the author" placeholder="blur" />
}
```

**From `public/`** — you supply the dimensions:

```tsx
<Image src="/profile.png" alt="…" width={500} height={500} />
```

**Remote** — dimensions plus an explicit allowlist:

```ts
// next.config.ts
images: {
  remotePatterns: [
    { protocol: 'https', hostname: 's3.amazonaws.com', pathname: '/my-bucket/**' },
  ],
}
```

Be as specific as you can. `hostname: '**'` turns your image endpoint into an open proxy that anyone can use to serve arbitrary images on your bandwidth.

**Wrong vs. right — the deprecated config:**

```ts
// ✗ Deprecated in 16 — no path or protocol restriction.
images: { domains: ['example.com'] }

// ✓
images: { remotePatterns: [{ protocol: 'https', hostname: 'example.com', pathname: '/img/**' }] }
```

**`fill` for unknown dimensions**, e.g. a responsive card:

```tsx
<div className="relative aspect-video">
  <Image
    src={post.coverUrl}
    alt=""
    fill
    sizes="(max-width: 768px) 100vw, 50vw"
    className="object-cover"
  />
</div>
```

Two rules with `fill`: the parent needs `position: relative` (or absolute/fixed), and you should always pass `sizes` — without it the browser assumes `100vw` and downloads a far larger file than it needs.

**`priority` for the LCP image.** Images are lazy-loaded by default, which is wrong for the hero:

```tsx
<Image src={hero} alt="" priority />
```

One `priority` per page, on the largest above-the-fold image. Adding it everywhere defeats the purpose.

**The four `next/image` defaults that changed in 16** — this is where a "we upgraded and images look different" ticket comes from ([Upgrade guide](https://nextjs.org/docs/app/guides/upgrading/version-16#nextimage-changes)):

| Setting | Was | Now | Impact |
|---|---|---|---|
| `minimumCacheTTL` | 60s | **14400s (4h)** | Upstream images without `cache-control` update far less often |
| `qualities` | 1–100 | **`[75]`** | `quality={90}` is silently **coerced to 75** unless you configure it |
| `imageSizes` | included 16 | 16 removed | Slightly smaller `srcset` |
| `maximumRedirects` | unlimited | **3** | Redirect-chained CDN URLs can now fail |
| Local IPs | allowed | **blocked** | Optimising from a VPC host gives 400 until `dangerouslyAllowLocalIP` |

The `qualities` one is the most surprising, because nothing warns you:

```ts
// If you genuinely need other quality levels, declare them.
images: { qualities: [50, 75, 90] }
```

There's also a new restriction on local sources with query strings — `<Image src="/assets/photo?v=1" />` now requires `images.localPatterns` with a matching `search`, to prevent enumeration attacks.

**Self-hosting note.** Image optimisation works with zero config under `next start`, including in Docker. On glibc-based Linux, `sharp` may need [memory-allocator configuration](https://sharp.pixelplumbing.com/install#linux-memory-allocator) to avoid runaway memory ([Self-hosting — image optimization](https://nextjs.org/docs/app/guides/self-hosting#image-optimization)). If you'd rather not optimise in-process, configure a custom loader or set `unoptimized` and keep the rest of the component's benefits.

> **Real Scenario — the container that OOM-killed on the gallery page.**
> A self-hosted app ran fine until someone launched a photo gallery. The container started getting OOM-killed a few times a day, always within a minute of high gallery traffic.
> **Cause:** `next/image` optimises on demand, in-process, using `sharp`. Dozens of concurrent 4000×3000 source images meant dozens of concurrent decodes in one Node process with a 512 MB memory limit. The default glibc allocator's fragmentation made it worse — memory was released by sharp but not returned to the OS.
> **Fix, in layers:** raised the container memory limit, set the jemalloc allocator per sharp's Linux guidance, pre-resized source images on upload so the optimiser never saw 12-megapixel originals, and — because `minimumCacheTTL` is now 4 hours by default — let the optimised results cache far longer than they used to.
> **The lesson:** on Vercel, image optimisation is someone else's capacity problem. Self-hosted, it is a CPU- and memory-bound workload sharing a process with your renderer.

### 11.3 Advanced — `next/font` and what changed in 16

`next/font` self-hosts fonts automatically — including Google Fonts, which are downloaded at build time and served from your own domain. No request ever goes to Google, so there's no third-party connection to negotiate and no privacy footnote to write ([Font optimization](https://nextjs.org/docs/app/getting-started/fonts)).

```tsx
// app/layout.tsx
import { Geist } from 'next/font/google'

const geist = Geist({ subsets: ['latin'] })

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={geist.className}>
      <body>{children}</body>
    </html>
  )
}
```

**Variable fonts need no `weight`; static fonts do:**

```tsx
import { Roboto } from 'next/font/google'
const roboto = Roboto({ weight: '400', subsets: ['latin'] })
```

**CSS variables** are the shape you want with Tailwind:

```tsx
import { Geist, Geist_Mono } from 'next/font/google'

const sans = Geist({ subsets: ['latin'], variable: '--font-sans' })
const mono = Geist_Mono({ subsets: ['latin'], variable: '--font-mono' })

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${sans.variable} ${mono.variable}`}>
      <body>{children}</body>
    </html>
  )
}
```

```css
/* app/globals.css */
@theme {
  --font-sans: var(--font-sans), ui-sans-serif, system-ui, sans-serif;
  --font-mono: var(--font-mono), ui-monospace, monospace;
}
```

**Local fonts:**

```tsx
import localFont from 'next/font/local'

const brand = localFont({
  src: [
    { path: './fonts/Brand-Regular.woff2', weight: '400', style: 'normal' },
    { path: './fonts/Brand-Bold.woff2', weight: '700', style: 'normal' },
  ],
  variable: '--font-brand',
})
```

**The constraint that catches everyone:** font loaders must be called at **module scope**, with literal arguments. They are build-time constructs, not runtime functions.

```tsx
// ✗ Fails to build — the loader is not a runtime function.
function Page({ weight }: { weight: string }) {
  const font = Roboto({ weight, subsets: ['latin'] })
}

// ✓ Module scope, literal values.
const roboto = Roboto({ weight: '400', subsets: ['latin'] })
```

Also: fonts are scoped to the component that uses them. Declaring one in the root layout applies it app-wide; declaring one in a single page loads it only for that page — which is a genuine optimisation for a display face used on one marketing route.

### 11.4 Cheat sheet: Part 11

| Need | Code |
|---|---|
| Global styles | `import './globals.css'` in the root layout |
| Scoped styles | `styles.module.css` |
| Static image with auto-dimensions | `import img from './x.png'` → `<Image src={img} />` |
| Image from `public/` | `<Image src="/x.png" width={} height={} />` |
| Remote image | `<Image src="https://…" width={} height={} />` + `remotePatterns` |
| Fill a container | `fill` + `relative` parent + `sizes` |
| LCP image | `priority` — one per page |
| Blur-up | `placeholder="blur"` (automatic on static imports) |
| Skip optimisation | `unoptimized` |
| Google font | `Geist({ subsets: ['latin'] })` at module scope |
| Font as a CSS variable | `variable: '--font-sans'` + `className={font.variable}` |
| Local font | `localFont({ src: [...] })` |

| Next.js 16 gotcha | Detail |
|---|---|
| `quality` silently coerced | `images.qualities` defaults to `[75]` |
| Images cache 4× longer | `minimumCacheTTL` 60s → 14400s |
| `images.domains` deprecated | Use `remotePatterns` |
| Local IP images blocked | `dangerouslyAllowLocalIP` for VPC hosts only |
| Redirect chains fail | `maximumRedirects` defaults to 3 |
| Local `src` with `?query` | Needs `images.localPatterns` with `search` |
| `~` in Sass imports | Unsupported by Turbopack |

[↑ Back to top](#table-of-contents)

---

**Next:** [Production & Deployment (Parts 12–13)](./nextjs-production-deployment.md) — env vars, observability, Docker, and what breaks behind a load balancer.
