# Next.js — Production & Deployment (Parts 12–13)

Environment variables, observability, the build output, Docker, and everything that breaks the first time your app runs as more than one container behind a reverse proxy.

> **Spec:** this doc follows the shared spec in [`nextjs-mastery-guide.md`](./nextjs-mastery-guide.md#about-this-document). **Deployment stance: self-hosted first** — `output: 'standalone'` + Docker + a Node server behind nginx. Vercel appears only as the contrast case. Written against **Next.js 16.3.4**, verified **September 11, 2026**.

Read Part 13 *before* your first production deploy, not after the first incident. Almost everything in it is invisible on a laptop and on a single container, and becomes load-bearing the moment you scale to two.

---

## Table of Contents

- [Part 12: Production Concerns](#part-12-production-concerns)
  - [12.1 Beginner — environment variables](#121-beginner--environment-variables)
  - [12.2 Working Knowledge — build-time vs. runtime](#122-working-knowledge--build-time-vs-runtime)
  - [12.3 Advanced — observability](#123-advanced--observability)
  - [12.4 Mastery — performance budget and the build output](#124-mastery--performance-budget-and-the-build-output)
  - [12.5 Cheat sheet: Part 12](#125-cheat-sheet-part-12)
- [Part 13: Deployment and Internals](#part-13-deployment-and-internals)
  - [13.1 Beginner — what `next build` produces](#131-beginner--what-next-build-produces)
  - [13.2 Working Knowledge — the Dockerfile](#132-working-knowledge--the-dockerfile)
  - [13.3 Advanced — the reverse proxy and streaming](#133-advanced--the-reverse-proxy-and-streaming)
  - [13.4 Mastery — multi-instance: cache, keys, and version skew](#134-mastery--multi-instance-cache-keys-and-version-skew)
  - [13.5 Self-hosted vs. Vercel: the honest comparison](#135-self-hosted-vs-vercel-the-honest-comparison)
  - [13.6 Cheat sheet: Part 13](#136-cheat-sheet-part-13)

---

## Part 12: Production Concerns

<a id="part-12-production-concerns"></a>

### 12.1 Beginner — environment variables

Next.js loads `.env*` files into `process.env` automatically ([Environment variables](https://nextjs.org/docs/app/guides/environment-variables)).

**Load order** — first match wins:

```
1. process.env                      (the real environment — always wins)
2. .env.$(NODE_ENV).local           e.g. .env.production.local
3. .env.local                       (skipped when NODE_ENV=test)
4. .env.$(NODE_ENV)                 e.g. .env.development
5. .env
```

**The one rule that matters:** variables are server-only unless prefixed `NEXT_PUBLIC_`. Anything without the prefix is replaced with an empty string in client bundles.

```bash
# .env.local — never committed
DATABASE_URL="postgresql://…"
AUTH_SECRET="…"
STRIPE_SECRET_KEY="sk_live_…"

NEXT_PUBLIC_SITE_URL="https://example.com"    # visible to everyone, forever
```

Three practical notes:

- With a `src/` directory, `.env*` files still live at the **project root**, not in `src/`.
- `$VAR` references expand — `TWITTER_URL=https://x.com/$TWITTER_USER` works. Escape a literal `$` as `\$`.
- Tools outside the Next.js runtime (`drizzle.config.ts`, a test runner) don't see these files. Use `@next/env` so they read the same values:

```ts
import { loadEnvConfig } from '@next/env'
loadEnvConfig(process.cwd())
```

### 12.2 Working Knowledge — build-time vs. runtime

This distinction causes more confused deploys than anything else in the framework.

```
   next build                        next start
   ──────────                        ──────────
   NEXT_PUBLIC_* values are          Server-only env vars are read
   INLINED into the JS bundle.       from the live process environment.
   Frozen. Immutable. Shipped.       Change them, restart, done.
```

> After being built, your app will no longer respond to changes to these environment variables. […] if you build and deploy a single Docker image to multiple environments, all `NEXT_PUBLIC_` variables will be frozen with the value evaluated at build time. — [Environment variables](https://nextjs.org/docs/app/guides/environment-variables#runtime-environment-variables)

> **Real Scenario — staging pointing at the production API.**
> A team built one image in CI and promoted it: build → staging → production. Staging's frontend kept calling the **production** API. Server-side calls were correct; only browser requests were wrong.
> **Cause:** `NEXT_PUBLIC_API_URL` was inlined at build time with the CI environment's value. Setting it in the staging container changed nothing — the string was already baked into the JavaScript. Server code read `process.env` live, which is why the two halves disagreed.
> **Fix:** the variable was dropped from the client entirely. The browser now calls a same-origin path, and the server resolves the upstream at request time.
> **The rule for promotable images: `NEXT_PUBLIC_` is for values that are identical in every environment.** Anything environment-specific must be read on the server.

**Reading env vars at runtime** on a page that would otherwise be prerendered ([Self-hosting — environment variables](https://nextjs.org/docs/app/guides/self-hosting#environment-variables)):

```tsx
import { connection } from 'next/server'

export default async function Page() {
  await connection()                      // defer to request time
  const value = process.env.MY_VALUE      // now read from the live environment
  return <p>{value}</p>
}
```

**Runtime config is gone.** `serverRuntimeConfig` and `publicRuntimeConfig` were **removed** in 16 ([Upgrade guide](https://nextjs.org/docs/app/guides/upgrading/version-16#runtime-configuration)). If you meet `getConfig()` from `next/config` in an old codebase, that's the replacement.

**Need a runtime value in the browser?** There is no built-in mechanism. Either expose it from a Route Handler the client calls on startup, or — usually better — render it into the page from a Server Component and pass it down as a prop.

### 12.3 Advanced — observability

**`instrumentation.ts`** runs `register()` **once per server instance**, before the server handles requests ([Instrumentation](https://nextjs.org/docs/app/guides/instrumentation)):

```ts
// instrumentation.ts — project root, not inside app/
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    await import('./instrumentation.node')
  }
}
```

```ts
// instrumentation.node.ts
import { registerOTel } from '@vercel/otel'
registerOTel({ serviceName: 'my-app' })
```

The runtime guard matters: `register` is called in every environment, so anything Node-specific must be conditionally imported.

Once per *instance*, not once per deployment. Right place for a tracing SDK; wrong place for database migrations (see [Part 8](./nextjs-database-auth.md#82-working-knowledge--migrations-and-query-patterns)).

**`onRequestError`**, exported from the same file, catches server-side errors with request context:

```ts
export async function onRequestError(
  err: unknown,
  request: { path: string; method: string; headers: Record<string, string> },
  context: { routerKind: string; routePath: string; renderSource: string }
) {
  await fetch(process.env.ERROR_SINK!, {
    method: 'POST',
    body: JSON.stringify({ message: String(err), ...request, ...context }),
  })
}
```

**Error digests.** In production, server error messages are redacted before reaching the client; `error.tsx` receives `error.digest` — a hash you correlate with your server logs. Log the digest alongside the stack trace or you'll have a hash with nothing to match it against.

```tsx
'use client'
useEffect(() => {
  captureException(error, { tags: { digest: error.digest } })
}, [error])
```

**Work after the response** with `after()`:

```ts
import { after } from 'next/server'

export async function POST(request: Request) {
  const result = await handle(request)
  after(async () => { await logToWarehouse(result) })   // runs post-response
  return Response.json(result)
}
```

Fully supported self-hosted with `next start` — but only if shutdown drains properly. See [§13.2](#132-working-knowledge--the-dockerfile).

**Dev-time diagnostics.** Next.js 16 improved request logging, splitting **Compile** from **Render** time so you can tell a slow bundle from a slow query ([Logging improvements](https://nextjs.org/blog/next-16#logging-improvements)). There's also a [DevTools MCP](https://nextjs.org/docs/app/guides/mcp) integration that gives coding agents access to your routing, caching and rendering behaviour plus unified browser/server logs.

### 12.4 Mastery — performance budget and the build output

**`next build` no longer prints `size` and `First Load JS`.** They were removed in 16 as inaccurate for server-driven architectures — the Turbopack and webpack implementations disagreed on how to account for Client Component payloads ([Performance improvements](https://nextjs.org/docs/app/guides/upgrading/version-16#performance-improvements)). Measure with Lighthouse or real-user monitoring instead.

What the build *does* tell you, and what to read it for:

```
   ▲ Next.js 16 (Turbopack)

 ✓ Compiled successfully in 615ms
 ✓ Finished TypeScript in 1114ms
 ✓ Collecting page data in 208ms
 ✓ Generating static pages in 239ms
 ✓ Finalizing page optimization in 5ms
```

Plus a route table marking each route static (○), dynamic (ƒ) or prerendered with params. **That table is the thing to check.** A route you expected to be static showing as dynamic means something in its tree reads `cookies()`, `headers()` or `searchParams` — see [Part 5](./nextjs-data-caching.md#part-5-caching-and-revalidation).

**Bundle analysis:**

```bash
npm i -D @next/bundle-analyzer
```

```ts
// next.config.ts
import bundleAnalyzer from '@next/bundle-analyzer'

const withBundleAnalyzer = bundleAnalyzer({ enabled: process.env.ANALYZE === 'true' })
export default withBundleAnalyzer(nextConfig)
```

```bash
ANALYZE=true npm run build
```

What to look for, in order: a large library in the **shared** chunk (something in a layout pulled it in); a `"use client"` boundary higher than it needs to be ([Part 3](./nextjs-routing-rendering.md#part-3-server-and-client-components)); and duplicate copies of the same dependency at different versions.

**Core Web Vitals**, mapped to the Parts that move them:

| Metric | Biggest lever | Where |
|---|---|---|
| **LCP** | `priority` on the hero image; prerender the route | [Part 11](./nextjs-navigation-assets.md#112-working-knowledge--nextimage), [Part 5](./nextjs-data-caching.md#part-5-caching-and-revalidation) |
| **CLS** | Always give images dimensions; `next/font` prevents font swap shift | [Part 11](./nextjs-navigation-assets.md#part-11-styling-images-and-fonts) |
| **INP** | Less client JS — move `"use client"` down | [Part 3](./nextjs-routing-rendering.md#part-3-server-and-client-components) |
| **TTFB** | `<Suspense>` so the shell ships before the slow query | [Part 4](./nextjs-data-caching.md#part-4-data-fetching-and-streaming) |

**Reproduce production locally before blaming the platform.** Dev never caches pages and never runs the optimised build:

```bash
npm run build && npm run start        # the real thing
node .next/standalone/server.js       # the exact artifact you'll ship
```

**React Compiler** is stable in 16 but off by default, because it relies on Babel and slows compiles ([React Compiler support](https://nextjs.org/docs/app/guides/upgrading/version-16#react-compiler-support)):

```ts
const nextConfig: NextConfig = { reactCompiler: true }
```

```bash
npm i -D babel-plugin-react-compiler
```

Worth enabling if your app is Client-Component-heavy and you are hand-writing `useMemo`/`useCallback`. Less compelling in a Server-Component-first app, where there is less client rendering to memoise.

### 12.5 Cheat sheet: Part 12

| Need | How |
|---|---|
| Server-only secret | any name without `NEXT_PUBLIC_` |
| Value the browser needs | `NEXT_PUBLIC_*` — **frozen at build time** |
| Read env at request time | `await connection()` then `process.env.X` |
| Env vars in non-Next tools | `loadEnvConfig()` from `@next/env` |
| Run code at server start | `register()` in `instrumentation.ts` |
| Capture server errors | `onRequestError` in `instrumentation.ts` |
| Work after the response | `after(async () => …)` from `next/server` |
| Correlate a client error | `error.digest` ↔ server logs |
| Inspect bundles | `ANALYZE=true next build` |
| Test production behaviour | `next build && next start` |

| Removed / changed in 16 | Replacement |
|---|---|
| `serverRuntimeConfig`, `publicRuntimeConfig` | env vars + `connection()` |
| `size` / `First Load JS` in build output | Lighthouse, RUM |
| `next lint` | `eslint` directly |
| `devIndicators.appIsrStatus` etc. | removed; the indicator itself remains |

[↑ Back to top](#table-of-contents)

---

## Part 13: Deployment and Internals

<a id="part-13-deployment-and-internals"></a>

### 13.1 Beginner — what `next build` produces

Next.js supports four deployment shapes ([Deploying](https://nextjs.org/docs/app/getting-started/deploying)):

| Option | Feature support |
|---|---|
| **Node.js server** (`next start`) | All |
| **Docker container** | All |
| Static export (`output: 'export'`) | Limited — no Actions, Handlers, proxy, or ISR |
| Adapters (Vercel, Bun, others) | Varies |

This Part covers the middle-first path: a Docker container running a Node server.

**`output: 'standalone'`** is the key config. During the build, Next.js traces every `import`, `require` and `fs` call with [`@vercel/nft`](https://github.com/vercel/nft) to determine exactly which files each route needs, then copies them — including the necessary slice of `node_modules` — into `.next/standalone` ([`output`](https://nextjs.org/docs/app/api-reference/config/next-config-js/output)):

```ts
// next.config.ts
const nextConfig: NextConfig = { output: 'standalone' }
```

```
.next/standalone/
├── server.js          ← a minimal server; replaces `next start`
├── node_modules/      ← only what's actually reachable
├── .next/             ← the server build
└── package.json
```

**The two folders it does not copy** — because it assumes a CDN will serve them:

```bash
cp -r public .next/standalone/ && cp -r .next/static .next/standalone/.next/
```

Forget those two lines and you get a running app with no CSS, no JS and no images — an unstyled HTML skeleton. It is the single most common self-hosting mistake. Once they're in place, `server.js` serves them itself.

```bash
PORT=8080 HOSTNAME=0.0.0.0 node .next/standalone/server.js
```

### 13.2 Working Knowledge — the Dockerfile

The official multi-stage Dockerfile, adapted from [vercel/next.js/examples/with-docker](https://github.com/vercel/next.js/tree/canary/examples/with-docker):

```dockerfile
ARG NODE_VERSION=24.13.0-slim

# ── deps ───────────────────────────────────────────────────────
FROM node:${NODE_VERSION} AS dependencies
WORKDIR /app
COPY package.json package-lock.json* ./
RUN --mount=type=cache,target=/root/.npm npm ci --no-audit --no-fund

# ── build ──────────────────────────────────────────────────────
FROM node:${NODE_VERSION} AS builder
WORKDIR /app
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
ENV NODE_ENV=production
# Anything the build inlines must be present HERE, not at runtime:
ARG NEXT_PUBLIC_SITE_URL
ENV NEXT_PUBLIC_SITE_URL=$NEXT_PUBLIC_SITE_URL
# Keep Server Action references decryptable across instances and deploys:
ARG NEXT_SERVER_ACTIONS_ENCRYPTION_KEY
ENV NEXT_SERVER_ACTIONS_ENCRYPTION_KEY=$NEXT_SERVER_ACTIONS_ENCRYPTION_KEY
RUN npm run build

# ── runtime ────────────────────────────────────────────────────
FROM node:${NODE_VERSION} AS runner
WORKDIR /app
ENV NODE_ENV=production PORT=3000 HOSTNAME="0.0.0.0"

COPY --from=builder --chown=node:node /app/public ./public
RUN mkdir .next && chown node:node .next
COPY --from=builder --chown=node:node /app/.next/standalone ./
COPY --from=builder --chown=node:node /app/.next/static ./.next/static

USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

Seven details that matter:

1. **`.next/standalone` and `.next/static` are copied separately.** See §13.1.
2. **`chown node:node .next`** — the prerender/ISR cache is written at runtime, so the non-root user needs write access.
3. **`USER node`** — do not run as root.
4. **Build-time args for anything inlined.** `NEXT_PUBLIC_*` set only on the runtime container does nothing ([Part 12](#122-working-knowledge--build-time-vs-runtime)).
5. **`NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` at build time**, because the key is embedded in the build output ([Self-hosting](https://nextjs.org/docs/app/guides/self-hosting#server-functions-encryption-key)).
6. **Slim base image** — matched between builder and runner, because native modules like `sharp` compile against a specific libc.
7. **No migration in `CMD`.** Migrations are a deploy step ([Part 8](./nextjs-database-auth.md#82-working-knowledge--migrations-and-query-patterns)).

**Graceful shutdown.** When stopping, send `SIGINT` or `SIGTERM` and **wait**: the Next.js server finishes in-flight requests and executes pending `after()` callbacks before exiting. The docs recommend a **10–30 second drain period** ([Self-hosting — `after`](https://nextjs.org/docs/app/guides/self-hosting#after)). In Kubernetes that's `terminationGracePeriodSeconds: 30`; in Compose, `stop_grace_period: 30s`. Too short and you silently drop background work.

Note also: `docker run` sends SIGTERM to PID 1, and Node doesn't reap zombies — use `--init` (or `tini`) so signals are handled correctly.

### 13.3 Advanced — the reverse proxy and streaming

**Use a reverse proxy.** The official recommendation is explicit: put nginx (or equivalent) in front rather than exposing the Next.js server directly, so malformed requests, slow-connection attacks, payload limits and rate limiting are handled before they reach your renderer ([Self-hosting — reverse proxy](https://nextjs.org/docs/app/guides/self-hosting#reverse-proxy)).

**And then the proxy will break streaming.** nginx buffers responses by default, so the streamed chunks that make `<Suspense>` and `loading.tsx` worthwhile get held until the response completes. The user waits for everything, then gets everything — exactly the behaviour streaming exists to avoid.

```js
// next.config.js — tell nginx not to buffer
module.exports = {
  async headers() {
    return [
      { source: '/:path*{/}?', headers: [{ key: 'X-Accel-Buffering', value: 'no' }] },
    ]
  },
}
```

Or in nginx directly:

```nginx
location / {
  proxy_pass http://app:3000;
  proxy_buffering off;
  proxy_http_version 1.1;
  proxy_set_header Connection '';
  proxy_set_header Host $host;
  proxy_set_header X-Forwarded-Host $host;      # required for the CSRF check
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_read_timeout 300s;
}
```

The docs are clear that this is an **end-to-end** requirement: load balancers must support chunked transfer encoding or HTTP/2 streaming, and every proxy in the path must pass chunks through. AWS ALB with Lambda integration is named specifically as a buffering default ([Streaming and Suspense](https://nextjs.org/docs/app/guides/self-hosting#streaming-and-suspense)).

`X-Forwarded-Host` deserves its own mention: the Server Action CSRF check compares `Origin` to `Host`/`X-Forwarded-Host`. Get it wrong and every mutation is rejected in production while working perfectly locally. If you terminate TLS at a CDN with a different hostname, add it to `serverActions.allowedOrigins`.

> **Real Scenario — "streaming doesn't work in production."**
> A team spent two days convinced their `<Suspense>` boundaries were misconfigured. Locally, the shell painted in 40ms and widgets filled in. In production, the page appeared complete after 1.8 seconds — no skeletons, ever.
> **Cause:** nginx `proxy_buffering on` (the default). The server streamed correctly; nginx collected every chunk and forwarded one complete response. Zero application code was wrong.
> **Diagnosis:** `curl -N https://prod/dashboard` showed the whole body arriving at once, while `curl -N http://app:3000/dashboard` from inside the network streamed properly. That two-command comparison isolates it instantly.
> **Fix:** `proxy_buffering off` plus the `X-Accel-Buffering` header, so a future proxy inherits the intent.

**Other self-hosting notes worth knowing** ([Self-hosting](https://nextjs.org/docs/app/guides/self-hosting)):

- **`proxy.ts` works with zero config** under `next start`. Not supported with static export, since it needs the incoming request.
- **Automatic cache headers:** immutable hashed assets get `public, max-age=31536000, immutable` and this **cannot be overridden**. Dynamically rendered pages get `private, no-cache, no-store, max-age=0, must-revalidate`. Fully prerendered pages get `public` so a CDN can cache them.
- **`assetPrefix`** moves static assets to a CDN domain, at the cost of extra DNS and TLS negotiation.

### 13.4 Mastery — multi-instance: cache, keys, and version skew

Everything so far works on one container. Here is what changes at two.

```
                    ┌──────────────┐
   users ─────────► │  nginx / LB  │
                    └──────┬───────┘
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         ┌────────┐   ┌────────┐   ┌────────┐
         │ app-1  │   │ app-2  │   │ app-3  │
         │ cache ✗│   │ cache ✗│   │ cache ✗│  ← in-memory + local disk,
         └───┬────┘   └───┬────┘   └───┬────┘     NOT shared
             └────────────┼────────────┘
                          ▼
                   ┌─────────────┐   ┌──────────────┐
                   │  Postgres   │   │ Redis (cache │
                   └─────────────┘   │  handler) ✓  │
                                     └──────────────┘
```

**Problem 1: the cache is per instance.** ISR pages and cached data default to each instance's local disk and memory. On Kubernetes, every pod has its own copy, so users see different versions depending on routing ([Configuring caching](https://nextjs.org/docs/app/guides/self-hosting#configuring-caching)).

```js
// next.config.js
module.exports = {
  cacheHandler: require.resolve('./cache-handler.js'),
  cacheMaxMemorySize: 0,     // disable the in-memory layer
}
```

```js
// cache-handler.js — the shape; back it with Redis for real use
const cache = new Map()

module.exports = class CacheHandler {
  constructor(options) { this.options = options }
  async get(key) { return cache.get(key) }
  async set(key, data, ctx) {
    cache.set(key, { value: data, lastModified: Date.now(), tags: ctx.tags })
  }
  async revalidateTag(tags) {
    tags = [tags].flat()
    for (const [key, value] of cache) {
      if (value.tags.some((t) => tags.includes(t))) cache.delete(key)
    }
  }
  resetRequestCache() {}
}
```

Vercel maintains a [Redis cache-handler example](https://github.com/vercel/next.js/tree/canary/examples/cache-handler-redis). For `'use cache'` entries specifically, the config key is [`cacheHandlers`](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheHandlers) (plural) and the directive is `'use cache: remote'` — see [Part 5](./nextjs-data-caching.md#54-mastery--cache-components-and-partial-prerendering).

**Problem 2: `revalidateTag` only invalidates the instance that ran it.** The others keep serving stale content until they independently discover the invalidation. Implement [`refreshTags()`](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheHandlers#refreshtags) in your cache handler — it is called before each request and should sync tag state from shared storage ([Multi-instance cache coordination](https://nextjs.org/docs/app/guides/self-hosting#multi-instance-cache-coordination)).

**Problem 3: Server Action encryption keys must match.** Closure variables are encrypted with a per-build key. Different keys across instances means an action encrypted by app-1 cannot be decrypted by app-2 — *"Failed to find Server Action"* ([Server Functions encryption key](https://nextjs.org/docs/app/guides/self-hosting#server-functions-encryption-key)):

```bash
NEXT_SERVER_ACTIONS_ENCRYPTION_KEY=$(openssl rand -base64 32)
```

Base64, and a valid AES length (16, 24 or 32 bytes). Set it **at build time**; it's embedded in the output.

**Problem 4: build IDs must match.** Next.js generates a build ID per build to identify the running version. **Build once, deploy that artifact everywhere.** If your pipeline rebuilds per environment, pin it ([Build cache](https://nextjs.org/docs/app/guides/self-hosting#build-cache)):

```js
module.exports = {
  generateBuildId: async () => process.env.GIT_HASH,
}
```

**Problem 5: version skew during rolling deploys.** Mid-rollout, a browser holding old assets can hit a new instance, producing missing chunks, unknown action IDs, and incompatible prefetch data ([Version skew](https://nextjs.org/docs/app/guides/self-hosting#version-skew)):

```js
module.exports = { deploymentId: process.env.DEPLOYMENT_VERSION }
```

With it set, static assets carry `?dpl=<id>`, navigations send an `x-deployment-id` header, and a mismatch triggers a **hard navigation** instead of a broken client transition. Component state is lost on that reload — URL state and `localStorage` survive.

> Note: when `deploymentId` is set, Next.js uses a constant build ID and `generateBuildId` has no effect.

**And one behaviour to plan capacity around:** all caches are scoped to a deployment, because the cache key includes the build ID. **Every deploy starts cold.** Deploying at peak means every page rebuilds at once — origin load spikes, and if your database is the bottleneck, that is when it shows.

**The multi-instance checklist:**

```ts
// next.config.ts
const nextConfig: NextConfig = {
  output: 'standalone',
  cacheHandler: require.resolve('./cache-handler.js'),
  cacheMaxMemorySize: 0,
  deploymentId: process.env.DEPLOYMENT_VERSION,
}
```
```bash
NEXT_SERVER_ACTIONS_ENCRYPTION_KEY=…   # at build time, stable across instances
```
Plus: one build artifact for all instances · `refreshTags()` in the cache handler · `proxy_buffering off` · 10–30s drain on shutdown · migrations as a deploy step.

### 13.5 Self-hosted vs. Vercel: the honest comparison

Not an endorsement either way — a list of what you take on.

| Concern | Vercel | Self-hosted |
|---|---|---|
| ISR / data cache | Managed, globally shared | Local disk per instance until you write a cache handler |
| `revalidateTag` across instances | Handled | Yours — `refreshTags()` + shared storage |
| Image optimisation | Managed service, separate capacity | In-process; CPU and memory in *your* container ([Part 11](./nextjs-navigation-assets.md#112-working-knowledge--nextimage)) |
| Streaming | Works | Works — until a proxy buffers it |
| Server Action keys | Handled | Set `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` yourself |
| Version skew | Handled | Set `deploymentId` yourself |
| CDN in front | Built in | You choose and configure it |
| PPR / Cache Components | Reference implementation | Supported with `next start` and Docker; streaming is a hard requirement |
| Cost model | Per usage | Per container |
| Where your data lives | Vercel's edge | Wherever you put it |

PPR is worth singling out: without end-to-end streaming support, the static shell and the dynamic content are delivered together after the full render, **eliminating PPR's entire TTFB advantage** ([Streaming and Suspense](https://nextjs.org/docs/app/guides/self-hosting#streaming-and-suspense)). Self-hosting PPR without fixing buffering gets you the complexity and none of the benefit.

Next.js 16 also shipped an alpha **[Build Adapters API](https://nextjs.org/docs/app/api-reference/config/next-config-js/adapterPath)** (`adapterPath`, promoted to a top-level option in 16.2) so platforms can hook into the build. Vercel and Bun are currently the verified adapters; Cloudflare and Netlify are working on theirs and ship their own integrations meanwhile.

### 13.6 Cheat sheet: Part 13

| Task | Command / config |
|---|---|
| Minimal deployable output | `output: 'standalone'` |
| Copy the two missing folders | `cp -r public .next/standalone/ && cp -r .next/static .next/standalone/.next/` |
| Run the artifact | `PORT=3000 HOSTNAME=0.0.0.0 node .next/standalone/server.js` |
| Monorepo tracing root | `outputFileTracingRoot: path.join(__dirname, '../../')` |
| Force-include a runtime asset | `outputFileTracingIncludes: { '/*': ['node_modules/sharp/**/*'] }` |
| Shared cache | `cacheHandler` + `cacheMaxMemorySize: 0` |
| Shared cache for `use cache` | `cacheHandlers` + `'use cache: remote'` |
| Stable action references | `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` at build |
| Version-skew protection | `deploymentId` |
| Stable build ID | `generateBuildId` (ignored if `deploymentId` is set) |
| Unbuffered streaming | `X-Accel-Buffering: no` + `proxy_buffering off` |
| Graceful shutdown | SIGTERM + 10–30s drain |
| Assets on a CDN | `assetPrefix` |

| Symptom in production | Cause |
|---|---|
| App loads unstyled, no JS | `public` / `.next/static` not copied into standalone |
| "Failed to find Server Action" | Mismatched encryption keys, or stale client after deploy |
| Server Actions rejected as CSRF | `X-Forwarded-Host` not forwarded, or missing `allowedOrigins` |
| Page arrives all at once | Proxy buffering |
| Some users see stale content | Per-instance cache; no shared cache handler |
| `EACCES` writing `.next` | `.next` not chowned to the runtime user |
| Origin load spike after deploy | Expected — caches are keyed by build ID and start cold |
| Missing chunks mid-rollout | Version skew; set `deploymentId` |
| Container OOM on image-heavy pages | In-process `sharp` optimisation ([Part 11](./nextjs-navigation-assets.md#112-working-knowledge--nextimage)) |
| Background work silently lost | Drain period too short for pending `after()` callbacks |

[↑ Back to top](#table-of-contents)

---

**You've reached the end of the set.** Back to the [overview](./nextjs-mastery-guide.md) for the [suggested learning order](./nextjs-mastery-guide.md#8-suggested-learning-order), the [quick self-check](./nextjs-mastery-guide.md#9-quick-self-check), and the [cross-doc cheat sheet](./nextjs-mastery-guide.md#10-cross-doc-cheat-sheet).
