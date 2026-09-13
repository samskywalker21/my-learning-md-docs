# Vite — SSR & Internals (Parts 13–14)

What Vite's server-side rendering primitives and the Environment API give you (and when you should use a framework instead), and how the dev server and build work underneath, so you can explain a slowdown and fix it.

> **Spec:** this doc follows the shared spec in [`vite-mastery-guide.md`](./vite-mastery-guide.md#about-this-document). Written against **Vite 8.3.0** / **Rolldown 1.2.8**, verified **September 13, 2026**. The SSR dev server and production build in §13.2–13.3 and the bundled dev mode probe in §14.4 were run locally.

Both Parts lean towards Mastery. For app work, the essentials are [§13.1](#part-13-beginner) (should you do SSR by hand at all?) and [§14.2](#part-14-working) (why a big app's dev server gets slow).

---

## Table of Contents

- [Part 13: SSR & the Environment API](#part-13-ssr-and-the-environment-api)
  - [13.1 Beginner — what Vite's SSR is, and who it's for](#part-13-beginner)
  - [13.2 Working Knowledge — a dev SSR server with the module runner](#part-13-working)
  - [13.3 Advanced — production builds, externals, hydration mismatches](#part-13-advanced)
  - [13.4 Mastery — the Environment API](#part-13-mastery)
  - [13.5 Cheat sheet: Part 13](#part-13-cheat-sheet)
- [Part 14: Internals & Performance](#part-14-internals-and-performance)
  - [14.1 Beginner — the toolchain: Vite, Rolldown, Oxc, Lightning CSS](#part-14-beginner)
  - [14.2 Working Knowledge — why dev gets slow, and the cheap fixes](#part-14-working)
  - [14.3 Advanced — profiling the dev server and the build](#part-14-advanced)
  - [14.4 Mastery — bundled dev mode](#part-14-mastery)
  - [14.5 Cheat sheet: Part 14](#part-14-cheat-sheet)

---

<a id="part-13-ssr-and-the-environment-api"></a>

## Part 13: SSR & the Environment API

<a id="part-13-beginner"></a>

### 13.1 Beginner — what Vite's SSR is, and who it's for

In Vite's docs, **SSR** means running your *frontend framework* (React, Vue, Svelte…) on a server to produce HTML, then hydrating it in the browser. Vite's SSR support is explicitly *"a low-level API meant for library and framework authors"* ([Server-Side Rendering](https://vite.dev/guide/ssr)). What it provides:

- a way to **load your app's modules in Node through Vite's plugin pipeline** during dev, with HMR;
- a **server build** (`vite build --ssr`) that outputs a Node-importable bundle;
- an **SSR manifest** for generating preload tags.

What it doesn't provide: routing, data loading, streaming, caching, error pages or deployment adapters. Frameworks built on Vite do that work.

```text
  Need SSR for an app?
    ├─ Vue       → Nuxt
    ├─ React     → React Router (framework mode) / TanStack Start
    ├─ Svelte    → SvelteKit
    ├─ content   → Astro
    └─ writing a framework, a custom renderer, or an SSR-for-one-page tool
                 → Vite's SSR primitives (this Part)
```

The rest of this Part is still worth reading if you use one of those frameworks. Their config exposes `ssr.noExternal`, environments and module runners, and when their abstraction leaks you need to know what those are.

<a id="part-13-working"></a>

### 13.2 Working Knowledge — a dev SSR server with the module runner

Source layout ([Server-Side Rendering](https://vite.dev/guide/ssr)):

```text
  index.html              <div id="app"><!--ssr-outlet--></div>
  server.js               your Node server; owns the HTTP layer
  src/
    app.ts                shared app code
    entry-client.ts       hydrates in the browser
    entry-server.ts       export function render(url) → HTML string
```

**The request flow in dev:**

```text
  GET /about
     │
     ▼
  server.js ──► vite.middlewares   (serves /@vite/client, /src/*.ts, assets; else next())
     │
     ▼ not a Vite asset
  read index.html ──► vite.transformIndexHtml(url, html)      (plugins inject HMR client etc.)
     │
     ▼
  vite.environments.ssr.runner.import('/src/entry-server.ts') ← transformed by the SAME
     │                                                          plugins, cached, HMR-aware
     ▼
  render(url) ──► replace <!--ssr-outlet--> ──► 200 text/html
                                    │
                                    ▼ browser
                    loads /src/entry-client.ts ──► hydrates
```

A working server, *verified locally* with plain `node:http` (Express works the same way: `app.use(vite.middlewares)`):

```js
// server.js
import fs from 'node:fs/promises'
import http from 'node:http'
import { createServer as createViteServer, isRunnableDevEnvironment } from 'vite'

const vite = await createViteServer({
  server: { middlewareMode: true },   // don't start Vite's own HTTP server
  appType: 'custom',                  // no SPA fallback / HTML handling; we own HTML
})
const ssrEnv = vite.environments.ssr
if (!isRunnableDevEnvironment(ssrEnv)) throw new Error('ssr environment is not runnable in-process')

http.createServer((req, res) => {
  vite.middlewares(req, res, async () => {
    try {
      const url = req.url
      let template = await fs.readFile('index.html', 'utf-8')
      template = await vite.transformIndexHtml(url, template)
      const { render } = await ssrEnv.runner.import('/src/entry-server.ts')
      const html = template.replace('<!--ssr-outlet-->', render(url))
      res.writeHead(200, { 'Content-Type': 'text/html' }).end(html)
    } catch (e) {
      vite.ssrFixStacktrace(e)          // map stack traces back to source
      res.writeHead(500).end(e.stack)
    }
  })
}).listen(5173)
```

Response for `GET /about` (*verified locally*):

```html
<html><head>
  <script type="module" src="/@vite/client"></script>
<title>ssr</title></head>
<body><div id="app"><h1>Hello from server at /about</h1></div><script type="module" src="/src/entry-client.ts"></script></body></html>
```

The server log showed `[vite] (ssr) connected.`: the SSR environment's module runner holds its own HMR channel.

> **Outdated convention alert.** Almost every SSR tutorial, and older versions of Vite's own SSR guide, call **`vite.ssrLoadModule('/src/entry-server.ts')`**. It still exists in 8.3, but the Environment API docs describe `ModuleRunner.import()` as *"the modern replacement for `server.ssrLoadModule`"*, with full HMR support ([Environment API for Frameworks](https://vite.dev/guide/api-environment-frameworks)). [Vite's breaking-changes page](https://vite.dev/changes/) lists "SSR using ModuleRunner API" as planned. Write new code against `environments.ssr.runner.import()`.

**Branch on server vs. client** with `import.meta.env.SSR`. It's statically replaced, so server-only code is tree-shaken from the client bundle ([Server-Side Rendering](https://vite.dev/guide/ssr)). For a whole module, use separate entry files instead of `if` branches.

<a id="part-13-advanced"></a>

### 13.3 Advanced — production builds, externals, hydration mismatches

**Two builds** ([Server-Side Rendering](https://vite.dev/guide/ssr)):

```json
{
  "scripts": {
    "build:client": "vite build --outDir dist/client",
    "build:server": "vite build --outDir dist/server --ssr src/entry-server.ts",
    "build": "npm run build:client && npm run build:server"
  }
}
```

*Verified locally*:

```text
dist/client/index.html                0.19 kB
dist/client/assets/index-B3GKZWor.js  0.84 kB
vite v8.3.0 building ssr environment for production...
dist/server/entry-server.js           0.22 kB
```

```js
// node -e "import('./dist/server/entry-server.js').then(m => console.log(m.render('/prod')))"
// → <h1>Hello from server at /prod</h1>
```

In production, `server.js` stops creating a Vite server. It reads `dist/client/index.html`, serves `dist/client` as static files, and imports `dist/server/entry-server.js` directly. **Vite is not a production runtime dependency.** Note that `build.minify` defaults to `false` for SSR builds ([Build Options](https://vite.dev/config/build-options)).

**Preload tags.** `vite build --ssrManifest` (client build) writes `dist/client/.vite/ssr-manifest.json`, mapping module IDs to their chunks and assets. The server records which modules rendered and emits `<link rel="modulepreload">` for them, so the browser doesn't discover them one hop at a time ([Server-Side Rendering](https://vite.dev/guide/ssr)).

**SSR externals: what gets bundled into the server build.** By default, dependencies in `node_modules` are **externalized**: left as `import 'pkg'` and loaded by Node at runtime. That's faster to build and matches how Node resolves packages. Two cases need `ssr.noExternal` ([Server-Side Rendering](https://vite.dev/guide/ssr)):

```ts
export default defineConfig({
  ssr: {
    noExternal: [
      '@acme/ui',          // ships .tsx/.vue/CSS imports Node can't load raw
      /^@fontsource\//,    // imports .css from JS
    ],
    // noExternal: true    // bundle everything, e.g. for a serverless function with no node_modules
  },
})
```

```text
  Symptom in production SSR                               Cause                      Fix
  ─────────────────────────────────────────────────────   ────────────────────────   ──────────────────
  "Unknown file extension .css" / ".vue" / ".tsx"         dep externalized, Node     ssr.noExternal
     at node_modules/…                                     can't import source
  "require is not defined" in an ESM server bundle        CJS dep bundled in; Vite   ssr.external, or
                                                           8 keeps require() as-is    esmExternalRequirePlugin
  Works in dev SSR, breaks in prod SSR                    dev runner transforms       test `build:server`
                                                           deps differently            in CI, not just dev
```

The `require` row is an inference from Vite 8's documented change to **keep `require()` calls for externalized modules as they are**, not a reproduced error. Vite re-exports `esmExternalRequirePlugin` (from `rolldown/plugins`, *types verified locally*) to convert them ([Migration](https://vite.dev/guide/migration)). **Build-side orchestration** of several environments goes through the `builder` config option and the `buildApp` hook, both present in 8.3's types.

**Hydration mismatches.** The server's HTML and the client's first render must match, or the framework discards and re-renders the markup (React warns, Vue warns in dev). The local demo above is itself a mismatch:

```ts
// Wrong: server renders "Hello from server", client's first render says "Hello from client"
export function renderApp(url: string) {
  return `<h1>Hello from ${import.meta.env.SSR ? 'server' : 'client'} at ${url}</h1>`
}

// Right: render identical markup on both sides; do environment-specific work after mount
export function renderApp(url: string) {
  return `<h1>Hello at ${url}</h1>`
}
// entry-client.ts, after hydration:
// document.querySelector('h1')!.dataset.hydrated = 'true'
```

The same class of bug comes from `Date.now()`, `Math.random()`, `window` checks, locale-dependent formatting, and **`import.meta.env.VITE_*` values that differ between the server build and the client build**, e.g. if the two builds run with different `--mode`. Build both with the same mode and `.env` files.

**Real Scenario — the currency that flickered.** An SSR storefront formats prices with `Intl.NumberFormat()` and no explicit locale. The server container runs with `LANG=C`, so it renders `$1,234.00`. A German user's browser hydrates with `1.234,00 $`. React logs a hydration mismatch, re-renders the price subtree, and users see the price flicker on every page load. Lighthouse flags the layout shift. The fix is passing an explicit locale (from the request's `Accept-Language` or a user setting) to both renders.

<a id="part-13-mastery"></a>

### 13.4 Mastery — the Environment API

Before Vite 6 there were implicitly two module graphs: client and SSR. The **Environment API** makes environments explicit and open-ended: `client`, `ssr`, and any others a framework defines, such as an edge worker or a React Server Components graph ([Environment API](https://vite.dev/guide/api-environment)).

**Status in 8.3: release candidate.** The docs plan to stabilize it in a future major, and some sub-APIs are still experimental ([Environment API](https://vite.dev/guide/api-environment)). The [Vite 8 announcement](https://vite.dev/blog/announcing-vite8) lists "Environment API stabilization" as ongoing work.

```ts
export default defineConfig({
  build: { sourcemap: false },           // inherited by every environment
  optimizeDeps: { include: ['lib'] },    // client environment by default
  environments: {
    server: {},                          // a Node server environment
    edge: {
      resolve: { noExternal: true },     // edge runtime: no node_modules at runtime
    },
  },
})
```

Environments **inherit top-level options** unless overridden. Some options are marked non-inheriting in the config reference, such as `resolve.conditions`, `resolve.mainFields` and `input` ([Shared Options](https://vite.dev/config/shared-options)). During dev, `client` and `ssr` always exist. During build, `client` always exists and `ssr` only when configured ([Environment API](https://vite.dev/guide/api-environment)).

**How a framework talks to an environment** ([Environment API for Frameworks](https://vite.dev/guide/api-environment-frameworks)):

| Environment type | Communication | Example |
|---|---|---|
| `RunnableDevEnvironment` | in-process: `environment.runner.import(url)` returns live module objects | Node SSR (§13.2); `isRunnableDevEnvironment()` checks for it |
| `FetchableDevEnvironment` | `handleRequest(request: Request) → Response` | a runtime that can't host Vite: Cloudflare Workers via `workerd` |
| raw `DevEnvironment` | build your own channel (virtual modules, `hot.send`) | anything else |

```text
                         ┌────────────── vite dev server (Node) ───────────────┐
                         │  shared: config, watcher, plugin container          │
                         │                                                     │
   browser ◄── ws/http ──┤  client env   module graph A   ── Oxc transforms    │
                         │  ssr env      module graph B   ── ModuleRunner ─────┼─► runs in this process
                         │  edge env     module graph C   ── fetch transport ──┼─► workerd process
                         └─────────────────────────────────────────────────────┘
```

Each environment has **its own module graph, dependency optimizer and plugin state**. That's why in [Part 10's trace](./vite-plugins-monorepo.md#part-10-advanced) the `options` hook ran twice in dev, and why plugins that keep caches should use `perEnvironmentState`. A custom environment is usually provided by a runtime plugin, such as Cloudflare's, which runs dev code inside `workerd` ([Environment API](https://vite.dev/guide/api-environment)).

**Build side.** Builds of every environment can be orchestrated together (`builder` config, the `buildApp` hook, `createBuilder()`). The CLI log line `building ssr environment for production...` (*verified locally*) comes from that machinery. Plugin sharing across environments during build is still "under consideration" on the [breaking-changes page](https://vite.dev/changes/).

**For app developers, the practical takeaways:**

1. If a framework's config has an `environments` key, it's this API. Top-level options still apply to every environment unless overridden.
2. `server.moduleGraph` still works but is a **mixed** client+SSR view kept for compatibility. Per-environment graphs are `server.environments.<name>.moduleGraph` ([Environment API](https://vite.dev/guide/api-environment)).
3. An SSR bug that isn't a client bug may be a *different module graph* holding a stale module. Restarting the dev server resets all of them.

<a id="part-13-cheat-sheet"></a>

### 13.5 Cheat sheet: Part 13

| Need | API / config |
|---|---|
| Use Vite inside your server | `createServer({ server: { middlewareMode: true }, appType: 'custom' })` |
| Serve Vite assets | `app.use(vite.middlewares)` |
| HMR client in SSR HTML | `vite.transformIndexHtml(url, html)` |
| Load server entry in dev | `vite.environments.ssr.runner.import(path)` (not `ssrLoadModule`) |
| Readable stack traces | `vite.ssrFixStacktrace(e)` |
| Server/client branch | `import.meta.env.SSR` (static) |
| Client build | `vite build --outDir dist/client [--ssrManifest]` |
| Server build | `vite build --outDir dist/server --ssr src/entry-server.ts` |
| Bundle a dep into server build | `ssr.noExternal: ['pkg']` (`true` = everything) |
| CJS `require` in ESM server output | `esmExternalRequirePlugin` |
| Extra runtimes | `environments: { edge: { … } }` (release candidate) |
| Starter templates | `create-vite-extra` SSR templates ([SSR guide](https://vite.dev/guide/ssr)) |

[↑ Back to top](#table-of-contents)

---

<a id="part-14-internals-and-performance"></a>

## Part 14: Internals & Performance

<a id="part-14-beginner"></a>

### 14.1 Beginner — the toolchain: Vite, Rolldown, Oxc, Lightning CSS

Vite 8 is an **orchestrator** over three Rust-based tools. Knowing which one does what tells you where to look when something goes wrong ([Vite 8 announcement](https://vite.dev/blog/announcing-vite8), [Migration](https://vite.dev/guide/migration)):

| Job | Vite ≤ 7 | Vite 8 | Config surface |
|---|---|---|---|
| TS/JSX → JS (dev and build) | esbuild | **Oxc Transformer** | `oxc` |
| Dependency pre-bundling | esbuild | **Rolldown** | `optimizeDeps.rolldownOptions` |
| Production bundling | Rollup | **Rolldown** | `build.rolldownOptions` |
| JS minification | esbuild | **Oxc Minifier** | `build.minify`, `rolldownOptions.output.minify` |
| CSS minification | esbuild | **Lightning CSS** | `build.cssMinify` |
| CSS transforms (`@import`, modules) | PostCSS | PostCSS (Lightning CSS opt-in) | `css.transformer` |
| Plugin API | Rollup-compatible | Rolldown (Rollup-compatible) + Vite hooks | `plugins` |

The team describes Vite, Rolldown and Oxc as an *"end-to-end toolchain with closely collaborating teams"*, so parsing, resolving, transforming and minifying behave consistently ([Vite 8 announcement](https://vite.dev/blog/announcing-vite8)). The practical effect: fewer "works in dev, breaks in build" bugs caused by two bundlers disagreeing. The cost: Vite 8 installs about 15 MB more than Vite 7 (Lightning CSS and the Rolldown binary), including **platform-specific native packages** ([Part 8](./vite-build-deploy.md#part-8-advanced)).

Reported build speed-ups against Rollup are 10–30×, with named examples like Linear going from 46 s to 6 s ([Vite 8 announcement](https://vite.dev/blog/announcing-vite8)). For scale, the `react-ts` template built in **254 ms** on the machine used for this doc (*verified locally*).

<a id="part-14-working"></a>

### 14.2 Working Knowledge — why dev gets slow, and the cheap fixes

Unbundled dev scales with **how many modules a page load touches**, not with app size. Anything that inflates that count, or adds work per module, shows up as a slow first load or a slow reload ([Performance](https://vite.dev/guide/performance)).

**1. Barrel files.** The biggest avoidable cost in most apps.

```ts
// src/components/index.ts: a barrel
export * from './Button'
export * from './DataGrid'     // + 40 files of grid internals
export * from './Chart'        // + a charting library
// … 200 more
```

```ts
// Wrong: to give you Button, the browser must fetch and transform EVERY file the barrel re-exports
import { Button } from '@/components'

// Right: import the module you need
import { Button } from '@/components/Button'
```

In production, tree-shaking removes the unused exports, so the barrel costs nothing there, which is why the problem hides. In dev there's no bundler to tree-shake, so every re-exported file is a real request and a real transform ([Performance — Avoid Barrel Files](https://vite.dev/guide/performance)). The same cost hits Vitest ([Part 12](./vite-testing-vitest.md#part-12-mastery)).

**2. Implicit extensions.** `import './Component'` makes Vite try each entry in `resolve.extensions` in turn, and each try is a filesystem check. Write the extension (`'./Component.tsx'`) and keep `resolve.extensions` short. The docs pair this with `moduleResolution: "bundler"` + `allowImportingTsExtensions`, which the templates already set ([Performance](https://vite.dev/guide/performance)).

**3. Plugins doing too much.** A `transform` without a hook filter runs for every module ([Part 10 §10.3](./vite-plugins-monorepo.md#part-10-advanced)). Slow `config`, `configResolved` or `buildStart` hooks delay startup ([Performance](https://vite.dev/guide/performance)).

**4. Heavy CSS and asset handling.** Prefer plain CSS over preprocessors where you can. Import SVGs as URLs or strings rather than turning each one into a framework component ([Performance](https://vite.dev/guide/performance)).

**5. The browser.** Extensions that intercept requests slow every module fetch. Devtools' **"Disable cache" defeats the `immutable` caching of pre-bundled deps**. Use a clean browser profile for dev ([Performance](https://vite.dev/guide/performance)).

**6. Warmup** for waterfalls you can't restructure away: `server.warmup.clientFiles`, or `vite --open` ([Part 2 §2.4](./vite-dev-server.md#part-2-mastery)).

**Real Scenario — the 9-second reload.** A dashboard app's full reload in dev creeps up to 9 seconds, and the team starts talking about migrating to a different tool. `vite --debug transform` shows nothing slow individually, but the network tab shows **2,300 requests** on the home page. The icons package is imported as `import { IconX } from '@/icons'`, a barrel over 1,900 generated SVG components. Changing the import style (plus a lint rule banning imports from `@/icons` and `@/components` barrels) brings it down to 400 requests and under 2 seconds. Production bundle size doesn't change, because tree-shaking had been removing the unused icons all along.

<a id="part-14-advanced"></a>

### 14.3 Advanced — profiling the dev server and the build

```bash
# Dev server CPU profile
vite --profile --open
#   … reproduce the slow page load, then in the terminal: p + enter
#   → writes a .cpuprofile; open it at speedscope.app

# Build CPU profile (8.3+: optional name)
vite build --profile

# Which transforms are slow / which files trigger full reloads
vite --debug transform
vite --debug hmr
DEBUG="vite:*" vite            # everything (very noisy)
```

([Troubleshooting — Performance](https://vite.dev/guide/troubleshooting), [Performance](https://vite.dev/guide/performance), [Changelog](https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md).)

**Reading the results:**

| Profile hotspot | Meaning | Action |
|---|---|---|
| a plugin's `transform` | that plugin is expensive or unfiltered | add `filter`; inspect its output with `vite-plugin-inspect` |
| `resolveId` / filesystem stat calls | extension probing, alias chains | explicit extensions; fewer aliases |
| Oxc/Rolldown native frames | real compile work | reduce module count (barrels), not much else to do |
| `optimizeDeps` at startup | pre-bundling | expected once; if repeated, something invalidates the cache (lockfile churn, config change) |
| your `config` / `buildStart` | plugin setup | lazy-load heavy imports |

**Build-side knobs**, in order of payoff:

1. `build.reportCompressedSize: false` skips gzip-size computation for every chunk ([Build Options](https://vite.dev/config/build-options)).
2. `build.sourcemap: false` for local/CI builds that don't upload maps.
3. Check that `tsc -b` (not Vite) is the slow part of `npm run build`. It usually is. `tsc -b` is incremental with `tsBuildInfoFile`, which the templates already set (*verified locally*).
4. Custom `codeSplitting` groups with broad `test` regexes make Rolldown do more work, and produce worse chunks too ([Part 7 §7.4](./vite-build-deploy.md#part-7-mastery)).

<a id="part-14-mastery"></a>

### 14.4 Mastery — bundled dev mode

Unbundled dev's weak point is huge module counts. **Bundled dev mode** (formerly "Full Bundle Mode") bundles the app in dev as well, using Rolldown with in-memory output and HMR, trading some startup work for far fewer requests ([8.1 announcement](https://vite.dev/blog/announcing-vite8-1)).

```bash
vite --experimental-bundle
```

```ts
export default defineConfig({
  experimental: { bundledDev: true },
})
```

**What changes on the wire** (*verified locally*, `react-ts`, 8.3.0):

| | Default (unbundled) | `--experimental-bundle` |
|---|---|---|
| Scripts in `index.html` | `/@vite/client`, `/@react-refresh` preamble, `/src/main.tsx` | `/bundledDevClient.mjs`, `/assets/index.js` |
| `GET /src/main.tsx` | 200, transformed JS module | **200 `text/html`**: not served as a module (SPA fallback) |
| App JS | one request per module | **one 1.59 MB dev bundle** (unminified, React dev builds) |

Reported results: *15× faster startup and 10× faster full reloads* for apps with 10,000+ React components, and at Linear *3× faster cold start, 40% faster full reloads, 10× fewer network requests* ([8.1 announcement](https://vite.dev/blog/announcing-vite8-1)).

**Caveats in 8.3:**

- **Experimental.** Third-party plugins and some features may not be compatible. The team is publishing plugin guidance ([8.1 announcement](https://vite.dev/blog/announcing-vite8-1)).
- **Anything that relies on per-module URLs** changes: tooling that reads `/src/…` URLs, `server.warmup` (nothing to warm), and debugging workflows based on the network tab.
- **Watching** switches to Rolldown's watcher. `server.watch` accepts Rolldown watch options for this mode since 8.3 ([Changelog](https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md), *types verified locally*). The `isBundled` environment option defaults to `true` for the client environment when `bundledDev` is on (*types verified locally*).
- 8.2 made it **reload after a rebuild** instead of showing a fallback page ([Changelog](https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md)).

**When to try it:** your dev network tab shows thousands of requests *after* you've dealt with barrels, and full reloads are the complaint. **When not to:** small or medium apps (unbundled is already fast, and bundling adds startup cost), or when you depend on plugins that haven't confirmed compatibility.

**Why this matters for the mental model.** The [overview](./vite-mastery-guide.md#the-mental-model) says "dev doesn't bundle, build does". Bundled dev mode is how that boundary may eventually disappear. The overview's update notes ask whoever next revises this set to check whether it has become stable or the default. If it has, [Part 2](./vite-dev-server.md#part-2-dev-server-and-module-graph) needs rewriting, not a patch.

**Other internals on the roadmap** ([Vite 8 announcement](https://vite.dev/blog/announcing-vite8)): raw AST transfer (JS plugins reading Rust ASTs without serialization), native MagicString transforms, persistent caching, and Module Federation support built on Rolldown.

<a id="part-14-cheat-sheet"></a>

### 14.5 Cheat sheet: Part 14

| Symptom | First check | Fix |
|---|---|---|
| Slow first load, many requests | network tab count | kill barrel imports; explicit extensions |
| Slow startup | `vite --profile` | lazy-load plugin deps; trim `config`/`buildStart` |
| One slow transform | `vite --debug transform` | hook `filter`; replace plugin |
| Deps re-optimized every start | lockfile/config churn | stabilize; don't clear `.vite` in scripts |
| Reloads slow even after that | module count in thousands | try `--experimental-bundle` |
| Slow `npm run build` | time `tsc -b` vs `vite build` separately | incremental tsc; `reportCompressedSize: false` |
| Dev fast, reload slow in one browser | extensions / "Disable cache" | clean profile |

| Tool | Role in Vite 8 |
|---|---|
| Rolldown | pre-bundling, production bundle, bundled dev mode, plugin container |
| Oxc | TS/JSX transform, JS minify, React Refresh (plugin-react v6) |
| Lightning CSS | CSS minify (default); CSS transform (opt-in) |
| chokidar | dev file watching (Rolldown watcher in bundled dev) |

[↑ Back to top](#table-of-contents)
