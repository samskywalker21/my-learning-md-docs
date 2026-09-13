# Vite — Build & Deploy (Parts 7–9)

What `vite build` produces and why, how to get that output onto a real server without 404s, stale chunks or leaked config, and how to build a package for other projects instead of an app.

> **Spec:** this doc follows the shared spec in [`vite-mastery-guide.md`](./vite-mastery-guide.md#about-this-document). Written against **Vite 8.3.0** / **Rolldown 1.2.8**, verified **September 13, 2026**. The Dockerfile and nginx config in Part 8 were **not run locally** in this pass (no Docker daemon was available). They're built from the cited nginx and Vite docs, so test them in your own pipeline.

Read [§7.3](#part-7-advanced) and [§8.2](#part-8-working) before your first production deploy. Between them they cover the two incidents every Vite SPA eventually has.

---

## Table of Contents

- [Part 7: The Production Build](#part-7-the-production-build)
  - [7.1 Beginner — `vite build` and `dist/`](#part-7-beginner)
  - [7.2 Working Knowledge — targets, sourcemaps, chunk warnings, analysis](#part-7-working)
  - [7.3 Advanced — code splitting, preloading, and the failed-chunk incident](#part-7-advanced)
  - [7.4 Mastery — manual chunking, the manifest, and cache-stable output](#part-7-mastery)
  - [7.5 Cheat sheet: Part 7](#part-7-cheat-sheet)
- [Part 8: Deploying an SPA](#part-8-deploying-an-spa)
  - [8.1 Beginner — `vite preview` is not a server](#part-8-beginner)
  - [8.2 Working Knowledge — history fallback and `base`](#part-8-working)
  - [8.3 Advanced — Docker, nginx, caching, same-origin API](#part-8-advanced)
  - [8.4 Cheat sheet: Part 8](#part-8-cheat-sheet)
- [Part 9: Library Mode](#part-9-library-mode)
  - [9.1 Advanced — building a package with `build.lib`](#part-9-advanced)
  - [9.2 Mastery — externals, CSS, types, and when not to use Vite](#part-9-mastery)
  - [9.3 Cheat sheet: Part 9](#part-9-cheat-sheet)

---

<a id="part-7-the-production-build"></a>

## Part 7: The Production Build

<a id="part-7-beginner"></a>

### 7.1 Beginner — `vite build` and `dist/`

```bash
npm run build        # tsc -b && vite build
```

Output from the `react-ts` template (*verified locally*):

```text
vite v8.3.0 building client environment for production...
✓ 20 modules transformed.
dist/index.html                   0.46 kB │ gzip:  0.29 kB
dist/assets/react-CHdo91hT.svg    4.12 kB │ gzip:  2.06 kB
dist/assets/vite-BF8QNONU.svg     8.70 kB │ gzip:  1.60 kB
dist/assets/hero-CLDdwZDr.png    13.05 kB
dist/assets/index-D64VDMd1.css    4.10 kB │ gzip:  1.47 kB
dist/assets/index-jOB7hSkO.js   222.53 kB │ gzip: 69.28 kB
✓ built in 254ms
```

```text
  index.html ──► plugin pipeline (same as dev) ──► Rolldown
                                                   ├─ resolve + bundle + tree-shake
                                                   ├─ split into chunks (entry, dynamic imports, CSS)
                                                   ├─ Oxc: lower syntax to build.target, minify JS
                                                   ├─ Lightning CSS: minify CSS
                                                   └─ hash filenames
                                                          │
  dist/index.html  ◄── <script type="module" crossorigin src="/assets/index-[hash].js">
  dist/assets/*    ◄── hashed JS, CSS, and non-inlined assets
  dist/<public/*>  ◄── copied verbatim
```

`dist/index.html` is **rewritten**: the `<script src="/src/main.tsx">` is replaced by the hashed bundle, moved into `<head>`, and given `crossorigin`. Linked CSS gets a `<link rel="stylesheet">` (*verified locally*).

`dist/` is a folder of static files. Any static host can serve it, subject to the history-fallback rule in [Part 8](#part-8-working).

<a id="part-7-working"></a>

### 7.2 Working Knowledge — targets, sourcemaps, chunk warnings, analysis

**`build.target`** defaults to `'baseline-widely-available'`: Chrome/Edge 111, Firefox 114, Safari 16.4 ([Build Options](https://vite.dev/config/build-options), [Building for Production](https://vite.dev/guide/build)). Oxc **lowers syntax** to that target. It does **not polyfill APIs**, so `Array.prototype.toSorted` stays a call that older browsers don't have.

```ts
// Wrong: "smallest output", with the support matrix never checked
build: { target: 'esnext' }

// Right: match the browsers you actually support
build: { target: ['chrome111', 'edge111', 'firefox114', 'safari15'] }  // must go below the default
```

The lowest possible target is `es2015`. Vite's output needs native dynamic `import()` and `import.meta` regardless ([Building for Production](https://vite.dev/guide/build)).

**Sourcemaps** default to off ([Build Options](https://vite.dev/config/build-options)):

| `build.sourcemap` | `.map` files | `//# sourceMappingURL` comment | Use |
|---|---|---|---|
| `false` | no | no | default |
| `true` | yes | **yes**: browsers fetch them, so anyone can read your source | internal tools |
| `'hidden'` | yes | **no** (*verified locally*) | upload to Sentry etc., don't deploy the `.map` files |
| `'inline'` | embedded | embedded | never in production (doubles size) |

**The 500 kB warning.** Chunks over `build.chunkSizeWarningLimit` (500 kB, pre-gzip) trigger a warning ([Build Options](https://vite.dev/config/build-options)).

```ts
// Wrong: silence the messenger
build: { chunkSizeWarningLimit: 5000 }

// Right: find what's big, then split by route (§7.3) or remove it
```

`build.reportCompressedSize: false` skips the gzip column and speeds up builds of large projects ([Build Options](https://vite.dev/config/build-options)).

**Analysing the bundle.** `rollup-plugin-visualizer` **7.1.1 works unchanged on Vite 8**. *Verified locally*: it produced a treemap showing `react-dom-client.production.js` as most of the `react-ts` bundle.

```ts
import { visualizer } from 'rollup-plugin-visualizer'
export default defineConfig(({ command }) => ({
  plugins: [react(), command === 'build' && visualizer({ filename: 'stats.html', gzipSize: true })],
}))
```

Vite 8 also ships an experimental `devtools` option that enables [Vite Devtools](https://vite.dev/blog/announcing-vite8) for build and module-graph analysis from the dev server.

**Build-time errors to recognise:**

- **Case sensitivity.** `import Button from './button'` resolves on macOS/Windows and fails on Linux CI when the file is `Button.tsx` ([Troubleshooting](https://vite.dev/guide/troubleshooting)).
- **"Module externalized for browser compatibility"**: a dependency imports `fs`, `path` or `crypto`. Vite won't polyfill Node built-ins. Find a browser-compatible package ([Troubleshooting](https://vite.dev/guide/troubleshooting)).
- **Opening `dist/index.html` from disk** fails with CORS errors, because `file://` can't load modules. Serve it: `vite preview` ([Troubleshooting](https://vite.dev/guide/troubleshooting)).

<a id="part-7-advanced"></a>

### 7.3 Advanced — code splitting, preloading, and the failed-chunk incident

**Dynamic `import()` is the unit of splitting.** Each dynamically imported module becomes its own chunk, and so does its CSS ([Features — Build Optimizations](https://vite.dev/guide/features)). *Verified locally*:

```ts
// main.ts
button.addEventListener('click', async () => {
  const { openSettings } = await import('./settings')   // settings.ts imports settings.css
  openSettings()
})
```

```text
dist/assets/index-C_XtqPr5.js      2.26 kB   ← entry
dist/assets/settings-BT2LIDKX.js   0.10 kB   ← dynamic chunk
dist/assets/settings-BgY1KoTg.css  0.02 kB   ← its CSS, split out too
```

For routes, that means lazy route components:

```tsx
// React
const Settings = lazy(() => import('./routes/Settings'))
```

```ts
// Vue Router
{ path: '/settings', component: () => import('./routes/Settings.vue') }
```

**What Vite adds for you** ([Features — Build Optimizations](https://vite.dev/guide/features)):

- **CSS loads before the chunk runs**, so there's no flash of unstyled content.
- **Dependencies of a dynamic chunk are fetched in parallel.** The entry contains a `__vite__mapDeps` table and a preload helper that injects `<link rel="modulepreload">` for everything the chunk needs, instead of discovering them one round trip at a time (*verified locally*: the helper also reads `meta[property=csp-nonce]`).
- **A `modulepreload` polyfill** for browsers without it (`build.modulePreload.polyfill`, default `true`).

**The failed-chunk incident.** This is the most common production bug in Vite SPAs:

```text
  09:00  User loads the app. index-AAA.js is in memory; it knows settings-111.js exists.
  09:30  You deploy. dist/ is replaced: settings-222.js exists, settings-111.js is GONE.
  09:45  User clicks Settings → import('/assets/settings-111.js') → 404
         → "TypeError: Failed to fetch dynamically imported module"
```

The [top Stack Overflow answer](https://stackoverflow.com/questions/69300341) (120+ votes) describes exactly this sequence, and [Vite's troubleshooting guide](https://vite.dev/guide/troubleshooting) adds poor networks and ad-blocking extensions as other causes. Vite emits a **`vite:preloadError`** event when a dynamic import fails ([Building for Production](https://vite.dev/guide/build)):

```ts
// Wrong: an unhandled rejection, a blank route, and a user who reports "the app is broken"
const Settings = lazy(() => import('./routes/Settings'))

// Right, layer 1: reload once to pick up the new index.html
window.addEventListener('vite:preloadError', (event) => {
  if (sessionStorage.getItem('chunk-reload')) return     // don't loop if the chunk is truly gone
  sessionStorage.setItem('chunk-reload', '1')
  event.preventDefault()                                 // stop Vite re-throwing (verified in the built preload helper)
  window.location.reload()
})
window.addEventListener('load', () => sessionStorage.removeItem('chunk-reload'))
```

Layer 2 is deployment, not code:

- **Keep the previous release's `assets/` for a while.** Upload new hashed files alongside the old ones instead of replacing the folder. Hashed names never collide.
- **Serve `index.html` with `Cache-Control: no-cache`**, so a reload actually fetches the new entry ([Building for Production](https://vite.dev/guide/build)). §8.3 has the headers.

**Real Scenario — the Monday-morning error spike.** A team deploys every Friday evening. Every Monday their error tracker shows a burst of `Failed to fetch dynamically imported module` from users who left a tab open over the weekend. The deploy script runs `rsync --delete dist/ server:/var/www/`. Removing `--delete` (plus a weekly job that prunes `assets/` files older than 30 days) and adding the `vite:preloadError` reload listener ends the spike.

<a id="part-7-mastery"></a>

### 7.4 Mastery — manual chunking, the manifest, and cache-stable output

**Manual chunking with `codeSplitting`.** Automatic splitting follows your dynamic imports. Manual groups pull matching modules into named chunks, usually so that rarely-changing vendor code keeps its hash across deploys:

```ts
export default defineConfig({
  build: {
    rolldownOptions: {
      output: {
        codeSplitting: {
          groups: [
            { name: 'react', test: /node_modules[\\/](react|react-dom|scheduler)[\\/]/, priority: 20 },
            { name: 'vendor', test: /node_modules/, priority: 10 },
          ],
        },
      },
    },
  },
})
```

*Verified locally*: a single `vendor` group moved 218 kB out of the `react-ts` entry into `vendor-*.js`, leaving `index-*.js` at 3.76 kB. The Vite docs point to `output.codeSplitting` for chunking strategy ([Building for Production](https://vite.dev/guide/build)).

```ts
// Wrong (Vite 8): the object form of manualChunks was REMOVED. The build fails:
//   Invalid output options … "manualChunks". Invalid type: Expected Function but received Object.
output: { manualChunks: { vendor: ['react', 'react-dom'] } }

// Deprecated but still works: the function form
output: { manualChunks: (id) => (id.includes('node_modules') ? 'vendor' : undefined) }

// Right: codeSplitting groups (above)
```

(Error text *verified locally*; [Migration](https://vite.dev/guide/migration).)

**Groups have sharp edges** ([Rolldown — Manual Code Splitting](https://rolldown.rs/in-depth/manual-code-splitting)):

- **A matched module pulls its dependencies in recursively**, "without considering constraints". A `vendor` group can swallow app code that a vendor module imports. `includeDependenciesRecursively: false` stops that, at the cost of more care with entry signatures.
- **Manual splitting can create circular chunk imports.** Rolldown generates a separate runtime chunk so that its helpers are always initialised first, but *your* module-level side effects can still run in an unexpected order.
- **`maxSize` is a target, not a limit.** A single module bigger than it can't be split.

A single "everything in `node_modules`" group often makes things **worse**. Every page downloads every dependency, including the charting library used on one admin route. Prefer route-level dynamic imports first, then add a small group for the framework runtime.

**The manifest** maps source files to their hashed outputs. You need it whenever something other than `dist/index.html` has to reference built files: a backend template, an SSR server, a service-worker precache list ([Backend Integration](https://vite.dev/guide/backend-integration)). *Verified locally* with `build.manifest: true`:

```json
{
  "index.html": {
    "file": "assets/index-C_XtqPr5.js", "src": "index.html", "isEntry": true,
    "dynamicImports": ["src/settings.ts"]
  },
  "src/settings.ts": {
    "file": "assets/settings-BT2LIDKX.js", "src": "src/settings.ts", "isDynamicEntry": true,
    "imports": ["index.html"], "css": ["assets/settings-BgY1KoTg.css"]
  }
}
```

It's written to **`dist/.vite/manifest.json`**, a dot-folder. Some deploy tools and `COPY` globs skip dot-folders, and some static hosts refuse to serve them.

**Hash cascades and `chunkImportMap`.** When `shared.ts` changes, its chunk hash changes, so every chunk that imports it gets new content (the import string changed), so *their* hashes change as well, all the way up to the entry. A one-line fix can invalidate most of the CDN cache. The experimental `build.chunkImportMap: true` (8.1+) makes chunks import stable IDs resolved through an import map, so only the changed chunk gets a new hash. It excludes CSS and assets, and is incompatible with `experimental.renderBuiltUrl` ([Features — Build Optimizations](https://vite.dev/guide/features), [8.1 announcement](https://vite.dev/blog/announcing-vite8-1)).

**Multi-page apps.** Several HTML entries, each with its own graph and shared chunks between them ([Building for Production](https://vite.dev/guide/build)):

```ts
import { resolve } from 'node:path'
export default defineConfig({
  build: {
    rolldownOptions: {
      input: {
        main: resolve(import.meta.dirname, 'index.html'),
        admin: resolve(import.meta.dirname, 'admin/index.html'),
      },
    },
  },
})
```

Since 8.2, the top-level `input` option does the same thing for build *and* dependency optimization ([Shared Options](https://vite.dev/config/shared-options)). Set `appType: 'mpa'` so the dev server doesn't SPA-fall back to `/index.html` for `/admin/`.

**Programmatic builds.** `import { build } from 'vite'`. In Vite 8, `build()` rejects with a `BundleError` whose `.errors` array holds the individual Rolldown errors ([Migration](https://vite.dev/guide/migration)).

<a id="part-7-cheat-sheet"></a>

### 7.5 Cheat sheet: Part 7

| Option | Default | Notes |
|---|---|---|
| `build.target` | `'baseline-widely-available'` | lowers syntax, never polyfills |
| `build.outDir` / `assetsDir` | `dist` / `assets` | |
| `build.sourcemap` | `false` | `'hidden'` for error trackers |
| `build.minify` | `'oxc'` (client), `false` (SSR) | `'terser'`/`'esbuild'` need installs |
| `build.cssMinify` | `'lightningcss'` | |
| `build.cssCodeSplit` | `true` | CSS per async chunk |
| `build.assetsInlineLimit` | `4096` | `0` for strict CSP |
| `build.chunkSizeWarningLimit` | `500` (kB) | fix the chunk, not the limit |
| `build.manifest` | `false` | → `dist/.vite/manifest.json` |
| `build.license` | `false` | → `dist/.vite/license.md` |
| `build.chunkImportMap` | `false` | experimental; stops hash cascades |
| `rolldownOptions.output.codeSplitting` | `true` | `{ groups: [...] }` replaces `manualChunks` |
| `rolldownOptions.output.minify.compress.dropConsole` | `false` | replaces `esbuild.drop` |
| `vite:preloadError` | — | reload-once handler for stale chunks |

[↑ Back to top](#table-of-contents)

---

<a id="part-8-deploying-an-spa"></a>

## Part 8: Deploying an SPA

*Tiers collapsed: Beginner → Advanced.* There's no Mastery tier here. The deep material under deployment is caching and chunk stability, which is already in [§7.4](#part-7-mastery), and runtime configuration, which is in [Part 5 §5.4](./vite-config-env-assets.md#part-5-mastery).

<a id="part-8-beginner"></a>

### 8.1 Beginner — `vite preview` is not a server

```bash
npm run build && npm run preview     # http://localhost:4173 by default
```

`vite preview` serves `dist/` so you can check the production build locally. The docs say plainly that it is *"not meant as a production server"* ([Deploying a Static Site](https://vite.dev/guide/static-deploy)). It also **hides two production bugs** (*verified locally*, 8.3.0):

| Behaviour | `vite preview` | A plain nginx/S3 deploy |
|---|---|---|
| `GET /users/42` (client-side route) | **200**, returns `index.html` | **404** unless configured |
| `Cache-Control` on `/assets/index-*.js` | **`no-cache`** | whatever you configure (often long) |

So "works in preview" tells you the *bundle* is right. It doesn't tell you the *server* is right. §8.2 and §8.3 are about the server.

<a id="part-8-working"></a>

### 8.2 Working Knowledge — history fallback and `base`

**History fallback.** Client-side routers (React Router, Vue Router in `createWebHistory` mode) change the URL without asking the server. On a refresh or a deep link, the browser *does* ask the server for `/users/42`. There is no such file, so the server must answer with `index.html` and let the router take over.

```nginx
# Wrong: only real files are served, so every deep link and refresh 404s
location / {
    try_files $uri =404;
}

# Right: file → directory → SPA shell
location / {
    try_files $uri $uri/ /index.html;
}
```

`try_files` checks each path in order and internally redirects to the last parameter if none exists ([nginx `try_files`](https://nginx.org/en/docs/http/ngx_http_core_module.html#try_files)). The same bug on other hosts: [Apache `.htaccess` rewrite](https://stackoverflow.com/questions/75272022), a [Vercel 404 on refresh](https://stackoverflow.com/questions/75963788), and Firebase's `rewrites` rule in [Vite's deploy guide](https://vite.dev/guide/static-deploy).

```nginx
# Wrong: the fallback also catches missing ASSETS, so a stale chunk request
# gets index.html (200, text/html) instead of a 404.
# The error becomes "Failed to load module script: MIME type text/html"
# and vite:preloadError's reload logic can't tell what happened.
location / {
    try_files $uri /index.html;
}

# Right: never fall back for /assets/
location /assets/ {
    try_files $uri =404;
}
location / {
    try_files $uri $uri/ /index.html;
}
```

**`base` for sub-path deployments.** An app served at `https://example.com/dashboard/` needs `base: '/dashboard/'` (or `vite build --base=/dashboard/`), and every asset URL in the build gets that prefix ([Building for Production](https://vite.dev/guide/build)). GitHub Pages project sites need `base: '/<REPO>/'` ([Deploying a Static Site](https://vite.dev/guide/static-deploy)). The router needs the same prefix: `createBrowserRouter(routes, { basename: import.meta.env.BASE_URL })` in React Router, `createWebHistory(import.meta.env.BASE_URL)` in Vue Router.

```nginx
# Sub-path deployment: the fallback must also live under the prefix
location /dashboard/ {
    alias /usr/share/nginx/html/;
    try_files $uri $uri/ /dashboard/index.html;
}
```

**Real Scenario — the Friday launch that only worked from the home page.** A marketing site's SPA passes QA: every page loads by clicking through from `/`, and `vite preview` is green. At launch, a campaign email links straight to `/pricing`, and every recipient sees the nginx 404 page. The container's `default.conf` was the stock nginx one, with no `try_files` fallback. QA never tested a deep link *with a cold load*, and preview's built-in fallback hid it. The fix is the `location` block above, plus a smoke test that requests `/pricing` directly with `curl -f`.

<a id="part-8-advanced"></a>

### 8.3 Advanced — Docker, nginx, caching, same-origin API

**A multi-stage Dockerfile** (*not run locally*, see the note at the top):

```dockerfile
# ---- build ----
FROM node:24-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
# Build-time values only. Anything here is published in the bundle (Part 5).
RUN npm run build

# ---- serve ----
FROM nginx:1.29-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
```

Two Docker-specific gotchas:

- **Don't `COPY` your host's `node_modules`.** Add it to `.dockerignore`. Rolldown and Lightning CSS ship **platform-specific native binaries** as optional dependencies. A Windows install contains `@rolldown/binding-win32-x64-msvc` and `lightningcss-win32-x64-msvc` (*verified locally*), and Alpine needs `@rolldown/binding-linux-x64-musl`. The Vite 7 version of this error, [`Cannot find module @rollup/rollup-linux-x64-musl`](https://stackoverflow.com/questions/78336694), is a popular question, and on Vite 8 the missing package is a `@rolldown/binding-*` one instead. That thread's top answer pins the musl binary under `optionalDependencies`. That works, but it hard-codes a platform into `package.json`. The root-cause fix is to never ship host `node_modules` into the image, run `npm ci` inside it, and commit a lockfile generated with optional dependencies intact, so every platform's binding is recorded in it.
- **`.dockerignore` your `.env*.local` files**, or they end up in the build context and possibly in the bundle.

**nginx config with correct caching.** Hashed files can be cached forever. Anything unhashed cannot:

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;

    # Hashed build output: the content never changes under a given name
    location /assets/ {
        try_files $uri =404;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }

    # The SPA shell: always revalidate so users pick up new deployments
    location = /index.html {
        add_header Cache-Control "no-cache";
    }

    # Same-origin API: the production counterpart of server.proxy
    location /api/ {
        proxy_pass http://api:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        try_files $uri $uri/ /index.html;
        add_header Cache-Control "no-cache";
    }
}
```

`Cache-Control: no-cache` on HTML is Vite's own recommendation for avoiding stale asset references ([Building for Production](https://vite.dev/guide/build)). Files from `public/` (`/favicon.svg`, `/config.js`) aren't hashed, so they fall through to the `no-cache` block. That's right for them.

```nginx
# Wrong: one long cache rule for everything
location / {
    expires 1y;
    try_files $uri $uri/ /index.html;
}
# index.html is cached for a year → users run a stale entry → its chunks
# get deleted by a later deploy → the failed-chunk incident from §7.3, for months.
```

**Why the `/api/` block matters.** In dev, `server.proxy` made `/api` same-origin ([Part 2](./vite-dev-server.md#part-2-advanced)). Without a production equivalent you either get CORS errors or bake an API URL into the bundle, and a baked URL breaks build-once/deploy-many ([Part 5 §5.4](./vite-config-env-assets.md#part-5-mastery)). With the `/api/` block, `fetch('/api/users')` works unchanged in both.

**Other hosts.** Vite's deploy guide covers GitHub Pages, GitLab Pages, Netlify, Vercel, Cloudflare, Firebase, Azure Static Web Apps, Render and Surge ([Deploying a Static Site](https://vite.dev/guide/static-deploy)). The checklist is the same everywhere: SPA fallback **except for `/assets/`**, immutable caching for `/assets/`, `no-cache` for HTML, and `base` if the app isn't at the root.

**Real Scenario — the API URL that stayed in staging.** See [Part 5 §5.4](./vite-config-env-assets.md#part-5-mastery). The deployment fix is the `/api/` location block above, which removes the environment-specific URL from the bundle altogether.

<a id="part-8-cheat-sheet"></a>

### 8.4 Cheat sheet: Part 8

| Requirement | nginx | Why |
|---|---|---|
| Deep links / refresh | `try_files $uri $uri/ /index.html;` | client-side routing |
| Missing chunks → real 404 | `location /assets/ { try_files $uri =404; }` | stale-chunk errors stay diagnosable |
| Cache hashed assets | `Cache-Control: public, max-age=31536000, immutable` on `/assets/` | names change with content |
| Pick up deploys | `Cache-Control: no-cache` on `index.html` | Vite recommendation |
| API without CORS / baked URLs | `location /api/ { proxy_pass … }` | prod equivalent of `server.proxy` |
| Sub-path | `base: '/x/'` + router basename + `location /x/` | |
| Deploy without breaking open tabs | don't delete old `assets/` immediately | §7.3 |

| Don't | Because |
|---|---|
| Use `vite preview` in production | not a production server; hides fallback and caching bugs |
| `COPY node_modules` into the image | native Rolldown/Oxc/Lightning CSS binaries are per-platform |
| `rsync --delete` assets on deploy | breaks every open tab's lazy routes |

[↑ Back to top](#table-of-contents)

---

<a id="part-9-library-mode"></a>

## Part 9: Library Mode

*Tiers collapsed: Advanced → Mastery only.* Publishing a package isn't a beginner task for an app developer, and it assumes Parts 6 and 7.

<a id="part-9-advanced"></a>

### 9.1 Advanced — building a package with `build.lib`

Library mode builds **a package that other projects import**, not an app. There is no `index.html` entry. The entry is a module ([Building for Production — Library Mode](https://vite.dev/guide/build)):

```ts
// vite.config.ts
import { resolve } from 'node:path'
import { defineConfig } from 'vite'

export default defineConfig({
  build: {
    lib: {
      entry: resolve(import.meta.dirname, 'lib/main.ts'),
      name: 'MyLib',          // global name; required only for umd/iife
      fileName: 'my-lib',
    },
    rolldownOptions: {
      external: ['react', 'react-dom', 'react/jsx-runtime'],
      output: { globals: { react: 'React', 'react-dom': 'ReactDOM' } },
    },
  },
})
```

*Verified locally* (entry imports a CSS file and re-exports from a second module):

| Entry shape | Default formats | Output |
|---|---|---|
| single entry | `es`, `umd` | `my-lib.js`, `my-lib.umd.cjs`, `my-lib.css` |
| `entry: { main, date }` | `es`, `cjs` | `main.js`, `date.js`, `main.cjs`, `date.cjs`, **`my-lib.css`** |

UMD and IIFE don't support multiple entries (from the `LibraryOptions` doc comments in Vite 8.3.0's types, *verified locally*). With multiple entries the CSS file name falls back to the **`package.json` `name`**, not to an entry name. Set `build.lib.cssFileName` so the file name doesn't change if the package is renamed.

The recommended `package.json` ([Building for Production — Library Mode](https://vite.dev/guide/build)):

```json
{
  "name": "my-lib",
  "type": "module",
  "files": ["dist"],
  "main": "./dist/my-lib.umd.cjs",
  "module": "./dist/my-lib.js",
  "exports": {
    ".": {
      "import": "./dist/my-lib.js",
      "require": "./dist/my-lib.umd.cjs"
    },
    "./style.css": "./dist/my-lib.css"
  }
}
```

**Env in libraries.** `import.meta.env.*` is statically replaced at *library* build time. `process.env.*` is left alone so the *consuming* app's bundler can replace it ([Building for Production — Library Mode](https://vite.dev/guide/build)). A library that wants `if (process.env.NODE_ENV !== 'production')` dev warnings should use `process.env.NODE_ENV`, not `import.meta.env.DEV`. Otherwise the check is fixed at whatever mode the library was built in.

<a id="part-9-mastery"></a>

### 9.2 Mastery — externals, CSS, types, and when not to use Vite

**Externalize peer dependencies, or ship a second copy of them.** Anything not in `rolldownOptions.external` is bundled *into* your library. Bundling React into a component library means the consuming app has two Reacts, and hooks fail with "Invalid hook call". That is the [top answer here](https://stackoverflow.com/questions/72764295): move `react` and `react-dom` to `peerDependencies`, externalize them, and dedupe them in the consumer.

*Verified locally* with a one-component React library built with `@vitejs/plugin-react`: `external: ['react', 'react-dom']` produced an **8.27 kB** `ui.js` containing a bundled copy of React's JSX runtime. The function form below produced **0.18 kB**, starting with `import { jsx as e } from "react/jsx-runtime"`.

```ts
// Wrong: exact-string externals miss subpaths. 'react/jsx-runtime' is still bundled.
external: ['react', 'react-dom']

// Right: externalize every peer and dependency, including subpaths
import pkg from './package.json' with { type: 'json' }
const deps = [...Object.keys(pkg.peerDependencies ?? {}), ...Object.keys(pkg.dependencies ?? {})]
export default defineConfig({
  build: {
    lib: { entry: resolve(import.meta.dirname, 'lib/main.ts'), formats: ['es'] },
    rolldownOptions: {
      external: (id) => deps.some((d) => id === d || id.startsWith(`${d}/`)),
    },
  },
})
```

**CSS is emitted, not imported.** *Verified locally*: the built `main.js` has no `import './my-lib.css'`, while `my-lib.css` sits next to it. Consumers must import it themselves:

```ts
// consuming app
import { Button } from 'my-lib'
import 'my-lib/style.css'          // without this: unstyled components
```

This is the "unstyled buttons" question from the overview's self-check. The reason is intentional: a library that injects its own CSS import forces every consumer's bundler to handle CSS imports from `node_modules`, and SSR and some test runners can't. Document the import in your README, or use a CSS-in-JS or Tailwind-preset approach if you truly need zero-config styling.

**Path aliases don't survive into `dist` type declarations.** An `@/utils` import resolves during the Vite build, but naive `.d.ts` generation copies the alias verbatim, and consumers get `Cannot find module '@/utils'` (reported, and unanswered at the time of writing, as ["Vite React Library mode, path alias error in dist"](https://stackoverflow.com/questions/72130469)). Vite library mode **doesn't generate type declarations at all** ([Building for Production — Library Mode](https://vite.dev/guide/build)). Options as of this writing: `vite-plugin-dts` (5.1.0), `unplugin-dts` (1.1.0), or running `tsc --emitDeclarationOnly` as a separate step. Relative imports inside a library avoid the alias problem entirely.

**When not to use Vite library mode.** For libraries that don't target the browser (Node utilities, CLIs, isomorphic packages without CSS or assets), Vite's docs recommend **tsdown or Rolldown directly** ([Building for Production — Library Mode](https://vite.dev/guide/build)). Library mode earns its place when the library has **CSS, assets, Vue SFCs or JSX that need Vite plugins**, which is to say component libraries.

```text
  Is it a component library with CSS / SFCs / assets?
     ├─ yes → Vite library mode (+ dts plugin, + externals, + ./style.css export)
     └─ no  → tsdown (or Rolldown directly)
```

**Real Scenario — the design-system release that bloated every app.** A team publishes their component library with `external: ['react', 'react-dom']`. Every component file now carries its own copy of React's JSX runtime, because `react/jsx-runtime` doesn't exactly match either string. Consumer bundles grow, and one app that pins a different React minor starts logging "Invalid hook call" in dev. The fix is the function-form `external` above, plus a CI check that the built entry *imports* `react/jsx-runtime` rather than containing it: `grep -q 'from "react/jsx-runtime"' dist/*.js`.

<a id="part-9-cheat-sheet"></a>

### 9.3 Cheat sheet: Part 9

| Concern | Setting |
|---|---|
| Entry | `build.lib.entry` (or top-level `input`, 8.2+) |
| Formats | single: `['es','umd']`; multi: `['es','cjs']`; set `formats: ['es']` for modern-only |
| Global name | `build.lib.name`, only for `umd`/`iife` |
| Output names | `build.lib.fileName`, `build.lib.cssFileName` |
| Peer deps | `rolldownOptions.external` with a function that matches subpaths |
| CSS | emitted, not imported; export `./style.css` and document the import |
| Types | not generated: `vite-plugin-dts` / `unplugin-dts` / `tsc --emitDeclarationOnly` |
| Runtime env | use `process.env.NODE_ENV` (left for consumer), not `import.meta.env.DEV` |
| Non-browser library | tsdown or Rolldown instead |

[↑ Back to top](#table-of-contents)
