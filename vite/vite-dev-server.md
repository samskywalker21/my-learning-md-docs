# Vite — Dev Server & HMR (Parts 1–3)

What's in a Vite project, what happens between "the browser asked for a file" and "the file arrived", and why an edit sometimes hot-swaps and sometimes reloads the whole page.

> **Spec:** this doc follows the shared spec in [`vite-mastery-guide.md`](./vite-mastery-guide.md#about-this-document). Written against **Vite 8.3.0**, verified **September 13, 2026**.

If you read one thing carefully here, make it [§2.2](#part-2-working): dependency pre-bundling and its cache explain most "the dev server is serving stale code" reports.

---

## Table of Contents

- [Part 1: Project Anatomy](#part-1-project-anatomy)
  - [1.1 Beginner — `index.html` is the entry point](#part-1-beginner)
  - [1.2 Working Knowledge — `public/` vs. imported assets, and the tsconfig split](#part-1-working)
  - [1.3 Cheat sheet: Part 1](#part-1-cheat-sheet)
- [Part 2: The Dev Server & Module Graph](#part-2-dev-server-and-module-graph)
  - [2.1 Beginner — one request per module](#part-2-beginner)
  - [2.2 Working Knowledge — dependency pre-bundling and its cache](#part-2-working)
  - [2.3 Advanced — host, proxy, allowed hosts, containers](#part-2-advanced)
  - [2.4 Mastery — the module graph and warmup](#part-2-mastery)
  - [2.5 Cheat sheet: Part 2](#part-2-cheat-sheet)
- [Part 3: Hot Module Replacement](#part-3-hot-module-replacement)
  - [3.1 Beginner — what HMR does for you](#part-3-beginner)
  - [3.2 Working Knowledge — boundaries, and why a file full-reloads](#part-3-working)
  - [3.3 Advanced — file watching in Docker and WSL2](#part-3-advanced)
  - [3.4 Mastery — the `import.meta.hot` API](#part-3-mastery)
  - [3.5 Cheat sheet: Part 3](#part-3-cheat-sheet)

---

<a id="part-1-project-anatomy"></a>

## Part 1: Project Anatomy

*Tiers collapsed: Beginner → Working Knowledge only.* The files in a project have no advanced mode. The advanced questions about them (base paths, multiple entries, tsconfig resolution) belong to [Part 4](./vite-config-env-assets.md#part-4-the-config-system) and [Part 7](./vite-build-deploy.md#part-7-the-production-build).

<a id="part-1-beginner"></a>

### 1.1 Beginner — `index.html` is the entry point

A `react-ts` project from `create-vite` 9.2.1 (*verified locally*):

```text
my-app/
├── index.html            ← the entry. NOT in public/. Part of the module graph.
├── public/
│   ├── favicon.svg       ← copied to dist/ as-is, served at /favicon.svg
│   └── icons.svg
├── src/
│   ├── main.tsx          ← referenced by <script type="module"> in index.html
│   ├── App.tsx
│   ├── App.css / index.css
│   └── assets/hero.png   ← imported from code → hashed into dist/assets/
├── vite.config.ts        ← absent in vanilla-ts: Vite runs with zero config
├── tsconfig.json         ← "solution" file: just references the two below
├── tsconfig.app.json     ← your src/ code (browser)
├── tsconfig.node.json    ← vite.config.ts (Node)
├── .oxlintrc.json        ← React templates only
└── package.json          ← "type": "module"
```

```html
<!-- index.html -->
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>app-react-ts</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

Three things in that file matter:

- **`src="/src/main.tsx"` points at TypeScript source.** Vite treats `index.html` as source code: it resolves `<script type="module">` tags and follows them into the module graph ([Getting Started](https://vite.dev/guide/)). There is no `bundle.js` to point at.
- **`/` means the project root**, not the filesystem root and not `public/`. The root defaults to `process.cwd()`, the folder containing `index.html` ([Shared Options](https://vite.dev/config/shared-options)).
- **No `%PUBLIC_URL%`.** URLs inside `index.html` are rebased automatically ([Getting Started](https://vite.dev/guide/)). If you've seen that placeholder, it came from Create React App.

> **Outdated convention alert.** CRA and webpack projects keep `index.html` in `public/` and inject a bundle into it. Moving a CRA app to Vite means moving `index.html` to the root and adding the `<script type="module">` tag yourself. It is the most common first error in a migration.

What the dev server actually sends back for `index.html` (*verified locally*, `react-ts`, trimmed):

```html
<script type="module">import { injectIntoGlobalHook } from "/@react-refresh";
injectIntoGlobalHook(window); /* …Fast Refresh preamble… */</script>
<script type="module" src="/@vite/client"></script>
<!-- …your original head and body… -->
```

`/@vite/client` is the HMR client (WebSocket, error overlay, CSS injection). `/@react-refresh` comes from `@vitejs/plugin-react`. Both are injected by plugins through the `transformIndexHtml` hook ([Part 10](./vite-plugins-monorepo.md#part-10-plugins)). When a non-Vite backend serves your HTML, it has to inject these tags itself ([Backend Integration](https://vite.dev/guide/backend-integration)).

<a id="part-1-working"></a>

### 1.2 Working Knowledge — `public/` vs. imported assets, and the tsconfig split

**Two ways to ship a static file, with opposite behaviour:**

| | `public/robots.txt` | `src/assets/logo.png` imported in code |
|---|---|---|
| How you reference it | absolute path string: `'/robots.txt'` | `import logo from './assets/logo.png'` |
| Transformed / hashed | no, copied verbatim | yes: `dist/assets/logo-CLDdwZDr.png` |
| Inlined when small | never | under 4 KiB, as a data URI ([Build Options](https://vite.dev/config/build-options)) |
| Missing file detected | at runtime (404) | at build time (import fails) |
| Use for | `robots.txt`, `favicon.ico`, files that need a fixed URL | anything your code references |

```ts
// Wrong: a public/ path written as a relative import string.
// It works in dev, then breaks when the app is deployed under a `base` like /app/.
img.src = './logo.png'

// Right (asset owned by code): import it, and Vite rewrites the URL for any base.
import logo from './assets/logo.png'
img.src = logo

// Right (file must keep a fixed name): public/ plus BASE_URL.
img.src = `${import.meta.env.BASE_URL}logo.png`
```

**The tsconfig split exists because your code runs in two places.** `src/` runs in a browser (DOM types, `vite/client` types, JSX). `vite.config.ts` runs in Node (`@types/node`, `module: nodenext`). One tsconfig can't describe both correctly, so the template uses a solution file with two `references`. `tsc -b` builds both (*verified locally*):

```jsonc
// tsconfig.app.json (trimmed): what matters to Vite
{
  "compilerOptions": {
    "types": ["vite/client"],        // types for import.meta.env, asset imports, import.meta.hot
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,    // type-only imports must say `import type`
    "erasableSyntaxOnly": true,      // no enums/namespaces: Oxc strips types, doesn't compile them
    "noEmit": true                   // Vite emits JS; tsc only checks
  },
  "include": ["src"]
}
```

Why `verbatimModuleSyntax` and `erasableSyntaxOnly`: Vite transpiles one file at a time with Oxc and never sees your whole program, so it can't tell whether an imported name is a type or a value. The docs require `isolatedModules: true` for the same reason ([Features — TypeScript](https://vite.dev/guide/features)). `verbatimModuleSyntax` implies it. [Part 6](./vite-config-env-assets.md#part-6-assets-css-and-typescript) covers what breaks when these are off.

**Vue differences.** `tsconfig.app.json` extends `@vue/tsconfig/tsconfig.dom.json` and includes `src/**/*.vue`, and the build script runs `vue-tsc -b`, because `tsc` can't read `.vue` files (*verified locally*). The template also adds `.vscode/extensions.json` recommending the Vue language extension.

**Real Scenario — the favicon that 404'd in production.** A team deploys to `https://example.com/dashboard/` with `base: '/dashboard/'`. Everything loads except the logo, which a component set with `src="/logo.svg"` pointing into `public/`. In dev the base is `/` and it worked. In production the browser requests `https://example.com/logo.svg`. `index.html` got rewritten because Vite rebases URLs there, but a string literal inside a component is invisible to Vite. The fix: import the asset, or prefix it with `import.meta.env.BASE_URL` ([Building for Production — Public Base Path](https://vite.dev/guide/build)).

<a id="part-1-cheat-sheet"></a>

### 1.3 Cheat sheet: Part 1

| File / folder | Role | Gotcha |
|---|---|---|
| `index.html` (root) | Entry of the module graph | Not in `public/`; no `%PUBLIC_URL%` |
| `public/` | Copied verbatim, fixed URLs | Not hashed, so cache it carefully; reference with `BASE_URL` |
| `src/assets/` | Imported, hashed, maybe inlined | Must be imported, not string-referenced |
| `vite.config.ts` | Optional config | Runs in Node; typed by `tsconfig.node.json` |
| `tsconfig.app.json` | Browser code types | `types: ["vite/client"]` |
| `package.json` `"type": "module"` | ESM everywhere | Needed for ESM-only deps in config |
| `/@vite/client` | Injected HMR client | Backends serving HTML must inject it |

[↑ Back to top](#table-of-contents)

---

<a id="part-2-dev-server-and-module-graph"></a>

## Part 2: The Dev Server & Module Graph

<a id="part-2-beginner"></a>

### 2.1 Beginner — one request per module

When the browser requests a module, Vite **resolves** the path, **transforms** the file if it needs to (TS, JSX, Vue SFC, CSS → JS), and **serves** it as native ESM. The browser then requests that module's own imports. Here is what `/src/main.tsx` looks like on the wire (*verified locally*, trimmed):

```js
import __vite__cjsImport0_react from "/node_modules/.vite/deps/react.js?v=44c9339e";
import __vite__cjsImport1_reactDom_client from "/node_modules/.vite/deps/react-dom_client.js?v=6b777807";
import "/src/index.css";
import App from "/src/App.tsx";
// …JSX compiled to jsxDEV() calls, inline source map…
```

Every bare import has been rewritten to a URL:

- `'react'` → `/node_modules/.vite/deps/react.js?v=…`, a **pre-bundled** copy (§2.2). The `__vite__cjsImport` wrapper exists because React is still CommonJS.
- `'./App.tsx'` → `/src/App.tsx`, your source, transformed when it is requested.
- `'./index.css'` → `/src/index.css`, which comes back as **JavaScript** that injects a `<style>` tag and calls `import.meta.hot.accept()` (§3.2).

That's why the network tab in dev shows dozens or hundreds of requests. It is expected. Production is a different artefact ([Part 7](./vite-build-deploy.md#part-7-the-production-build)).

```text
  browser                         vite dev server
  ───────                         ───────────────
  GET /                 ────────► index.html + injected /@vite/client
  GET /src/main.tsx     ────────► resolveId → load → transform (Oxc) → rewrite imports
  GET /node_modules/.vite/deps/react.js?v=44c9  ► from disk cache (pre-bundled at startup)
  GET /src/App.tsx      ────────► transform on demand … and so on down the graph
       ▲                               │
       └────────── ws: HMR updates ◄───┘ chokidar file watcher
```

<a id="part-2-working"></a>

### 2.2 Working Knowledge — dependency pre-bundling and its cache

**Why dependencies get bundled even in dev** ([Dependency Pre-Bundling](https://vite.dev/guide/dep-pre-bundling)):

1. **CommonJS/UMD → ESM.** Browsers can't `import` CommonJS. Vite converts it and analyses exports so that `import { useState } from 'react'` works even though React assigns its exports dynamically.
2. **Request count.** `lodash-es` has more than 600 internal modules. Pre-bundled, it is one request.

At startup Vite crawls your source for bare imports, bundles them with **Rolldown** into `node_modules/.vite/deps/`, and serves them with `Cache-Control: max-age=31536000,immutable` (*verified locally*). The `?v=hash` query is what invalidates them in the browser.

**The cache is regenerated when** ([Dependency Pre-Bundling](https://vite.dev/guide/dep-pre-bundling)):

- the package manager lockfile changes;
- patches folder modification times change;
- relevant `vite.config` fields change;
- `NODE_ENV` changes.

**What does *not* invalidate it:** editing a file inside `node_modules` by hand, or changing an `npm link`-ed package's source. Vite has no way to know.

```bash
# Wrong: the reflex fix, which costs a full reinstall and can change resolved versions
rm -rf node_modules package-lock.json && npm install

# Right: invalidate just the pre-bundle cache
vite --force
# or, the same thing:
rm -rf node_modules/.vite && npm run dev
```

If the browser still serves the old copy after `--force`, it cached the `immutable` response. Tick "Disable cache" in devtools, reload once, then untick it. Leaving it ticked slows every dev reload ([Performance](https://vite.dev/guide/performance)).

**When discovery misses a dependency.** The crawler only sees imports it can statically find. If a dependency is imported only from a plugin-generated file, or is first reached after the initial page load, Vite discovers it late, re-bundles, and **reloads the page**, logging that new dependencies were optimized. Tell it up front:

```ts
// vite.config.ts
export default defineConfig({
  optimizeDeps: {
    include: ['some-cjs-lib', 'my-dep > nested-cjs-dep'], // force pre-bundling
    exclude: ['small-esm-only-lib'],                      // serve as-is
  },
})
```

The docs' rule of thumb: `include` large or CommonJS dependencies, `exclude` only small, valid ESM ones ([Dependency Pre-Bundling](https://vite.dev/guide/dep-pre-bundling)).

**Linked / workspace packages are treated as source**, not pre-bundled. Vite detects that they don't resolve inside `node_modules` ([Dependency Pre-Bundling](https://vite.dev/guide/dep-pre-bundling)). A linked package that ships CommonJS must be added to `optimizeDeps.include`, as a [Stack Overflow answer on `npm link`](https://stackoverflow.com/questions/67964556) shows. [Part 11](./vite-plugins-monorepo.md#part-11-monorepos-and-workspaces) goes into this properly.

**Real Scenario — the hotfix that wasn't there.** A developer patches a bug inside `node_modules/some-date-lib/index.js` to confirm a fix before opening an upstream PR. They reload, and nothing changes. They add a `console.log` to the file, and it never prints. Forty minutes later they find that the browser is loading `/node_modules/.vite/deps/some-date-lib.js`, a bundle made before the edit. `vite --force` makes the change show up. The durable version of the fix is `patch-package`: patches are one of the cache's invalidation inputs.

<a id="part-2-advanced"></a>

### 2.3 Advanced — host, proxy, allowed hosts, containers

**Defaults you need to know** ([Server Options](https://vite.dev/config/server-options)):

| Option | Default | Change it when |
|---|---|---|
| `server.host` | `'localhost'` | in a container, a VM, or testing on a phone → `true` / `--host` |
| `server.port` | `5173`, tries the next port if busy | CI, OAuth callback URLs → set `strictPort: true` |
| `server.allowedHosts` | `[]` (localhost and IPs only) | a custom hostname, a reverse proxy, a tunnel |
| `server.cors` | localhost origins only | another local origin fetches from the dev server |
| `server.fs.strict` | `true` | only in monorepos, see [Part 11](./vite-plugins-monorepo.md#part-11-monorepos-and-workspaces) |
| `server.fs.deny` | `.env`, `.env.*`, certs, `.npmrc`, `.git/**`… | add more; never remove |

**Host: why the container "isn't running".** On this machine, `vite` with the default host listened on **`[::1]:5199` only**, IPv6 loopback (*verified locally*). Inside a container, loopback is the container's own, so the published port has nothing behind it.

```yaml
# Wrong: compose publishes the port, but Vite binds loopback inside the container
services:
  web:
    command: npm run dev
    ports: ["5173:5173"]

# Right
services:
  web:
    command: npm run dev -- --host 0.0.0.0
    ports: ["5173:5173"]
```

This is the accepted fix in [a well-known Stack Overflow question](https://stackoverflow.com/questions/70012970) (`server.host: true`). VS Code Dev Containers go the other way: set `server.host: '127.0.0.1'` explicitly, because VS Code's port forwarding doesn't support IPv6 ([Troubleshooting](https://vite.dev/guide/troubleshooting)).

**Allowed hosts: "Blocked request".** Since a security release in the Vite 6.0.x line (the [top Stack Overflow answer](https://stackoverflow.com/questions/79372334) dates it to 6.0.11), requests whose `Host` header isn't localhost or an IP are rejected. *Verified locally* on 8.3.0 with `curl -H "Host: myapp.test"`:

```text
Blocked request. This host ("myapp.test") is not allowed.
To allow this host, add "myapp.test" to `server.allowedHosts` in vite.config.js.
```

You'll hit it behind a compose service name, a reverse proxy, or a tunnel.

```ts
// Wrong: the top answer's "or" option. The docs warn it lets any website
// send requests to your dev server and download your source code.
server: { allowedHosts: true }

// Right: name the hosts. A leading dot allows subdomains.
server: { allowedHosts: ['frontend_web', '.my-tunnel.example'] }
```

The same warning applies to `server.cors: true` ([Server Options](https://vite.dev/config/server-options)).

**The proxy: sidestep CORS in dev only.** `server.proxy` forwards matching requests to your backend, so the browser only ever talks to one origin:

```ts
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,                         // rewrite Host header to the target
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
      '/socket.io': { target: 'ws://localhost:8080', ws: true },
      '^/fallback/.*': { target: 'http://localhost:9000' }, // keys starting with ^ are regexes
    },
  },
})
```

The proxy **exists only in `vite` dev**, and `vite preview` has its own `preview.proxy`. Production needs the same routing in nginx or your platform, or the frontend has to call the API's real origin with CORS configured. Forgetting this is [Part 8](./vite-build-deploy.md#part-8-deploying-an-spa)'s first Real Scenario.

Two related proxy gotchas:

- **Inside Docker Compose, `localhost` in the proxy target is the Vite container itself.** Target the backend's *service name* (`http://api:5000`). This is the top answer on ["API call not reaching Vite proxy target (localhost:5000)"](https://stackoverflow.com/questions/74281247).
- **On the host, `localhost` can resolve to IPv6 `::1`** (as Vite's own listener did above). A backend bound only to `127.0.0.1` then never receives the proxied request. `http://127.0.0.1:5000` as the target removes the ambiguity. This is inferred from the local listener behaviour, not taken from a cited source.

**`server.fs.deny` protects `.env` from being served.** *Verified locally*: with a `.env` in the project root, `GET /.env`, `GET /.env?raw` and `GET /@fs/C:/Windows/win.ini` all return **403 Restricted**. Your dev server can still be reached by anything on the network once you pass `--host`, so keep the defaults.

<a id="part-2-mastery"></a>

### 2.4 Mastery — the module graph and warmup

The dev server keeps a **module graph**: one node per resolved module, with edges for imports, transform results cached per node, and HMR metadata (which modules accept updates, §3.2). Two practical consequences:

**Transform waterfalls.** Vite only discovers `App.tsx`'s imports after the browser has requested and received `App.tsx`. A deep import chain turns into a sequence of round trips. `server.warmup.clientFiles` pre-transforms files you know will be requested, so they are already cached when the browser asks ([Server Options](https://vite.dev/config/server-options)):

```ts
export default defineConfig({
  server: {
    warmup: { clientFiles: ['./src/components/Layout.tsx', './src/routes/**/*.tsx'] },
  },
})
```

`vite --open` warms up the entry for free ([Performance](https://vite.dev/guide/performance)). Only warm files that are hit on almost every load. Warming everything just moves the cost to startup.

**Dev ≠ build, even in Vite 8.** Pre-bundling and build both use Rolldown now, but your *source* is still served unbundled in dev and bundled in build. Two things can differ as a result:

- **Module evaluation order** across circular imports. Bundling hoists and orders modules. Native ESM follows the browser's graph walk.
- **CSS order.** In dev, `<style>` tags are injected in import-execution order. In build, CSS is concatenated per chunk. Two stylesheets with equal specificity can win differently.

When something only breaks in build, the first check is `vite build && vite preview`, not staring at dev. The experimental **bundled dev mode** in 8.1+ closes this gap by bundling in dev too ([Part 14](./vite-ssr-internals.md#part-14-internals-and-performance)).

**Browser console forwarding.** `server.forwardConsole` sends browser `console.*` and unhandled errors to the terminal. It defaults to **on when Vite detects an AI coding agent**, and off otherwise ([Server Options](https://vite.dev/config/server-options)). If logs you never printed in the terminal start showing up there, that's why.

<a id="part-2-cheat-sheet"></a>

### 2.5 Cheat sheet: Part 2

| Symptom | Cause | Fix |
|---|---|---|
| Hundreds of requests in dev | Unbundled source, by design | Nothing, or `experimental.bundledDev` for huge apps |
| Edited dependency not updating | `.vite/deps` cache | `vite --force` (+ devtools "Disable cache" once) |
| "new dependencies optimized… reloading" | Late discovery | `optimizeDeps.include` |
| CJS linked package fails to import | Linked deps aren't pre-bundled | `optimizeDeps.include: ['pkg']` |
| Container port has nothing behind it | Bound to loopback | `--host 0.0.0.0` |
| "Blocked request. This host … is not allowed" | Host check (6.0.11+) | `server.allowedHosts: ['name']` |
| Proxy target never hit | `localhost` is the container (Compose), or `::1` vs. IPv4 | service name in Compose; `http://127.0.0.1:port` on the host |
| API calls work in dev, 404 in prod | `server.proxy` is dev-only | Route `/api` in prod ([Part 8](./vite-build-deploy.md#part-8-deploying-an-spa)) |
| Slow first load on big route tree | Transform waterfall | `server.warmup.clientFiles`, `--open` |

[↑ Back to top](#table-of-contents)

---

<a id="part-3-hot-module-replacement"></a>

## Part 3: Hot Module Replacement

<a id="part-3-beginner"></a>

### 3.1 Beginner — what HMR does for you

Hot Module Replacement swaps a changed module into the running page **without a full reload**, so component state, scroll position and open dialogs survive the edit ([Features — HMR](https://vite.dev/guide/features)). You almost never write HMR code yourself, because the framework plugin does it:

| Stack | Who implements HMR | What survives an edit |
|---|---|---|
| React | `@vitejs/plugin-react` → **React Fast Refresh** | `useState` in edited components (unless hook order changes) |
| Vue | `@vitejs/plugin-vue` → SFC HMR | component state; `<template>`-only edits just re-render |
| CSS (any stack) | Vite core | everything; the `<style>` tag is replaced in place |
| Vanilla TS | nobody, unless you write `import.meta.hot` | nothing: **full reload** |

That last row surprises people. In a `vanilla-ts` app, editing `counter.ts` reloads the page, because no module has declared that it can accept updates.

<a id="part-3-working"></a>

### 3.2 Working Knowledge — boundaries, and why a file full-reloads

When a file changes, Vite walks **up** the module graph from it, looking for the nearest module that has declared it **accepts** updates: an *HMR boundary*. If every path up the graph hits a boundary, only those subtrees are re-executed. If any path reaches the root without one, the page does a full reload ([HMR API](https://vite.dev/guide/api-hmr)).

```text
                main.tsx            (no boundary)
               ╱        ╲
         App.tsx          api.ts     (no boundary)
        (boundary)          │
       ╱        ╲       config.ts    (no boundary)
  Button.tsx   format.ts
  (boundary)   (no boundary)

  edit Button.tsx  → Button accepts itself            → hot update ✔
  edit format.ts   → walks up to App.tsx (boundary)   → App re-executes ✔
  edit config.ts   → api.ts → main.tsx → root         → FULL RELOAD ✘
```

You saw a boundary on the wire in §2.1: a CSS module compiles to JS that ends with `import.meta.hot.accept()`, so CSS edits never reload.

**React: why a component file "stops hot-reloading".** Fast Refresh can only treat a file as a boundary if **every export is a React component**. Export a constant, a hook, or a helper next to a component, and the file stops being a boundary. The edit then propagates up, often all the way to a full reload, and state is lost. The `react-ts` template ships the Oxlint rule `react/only-export-components` to catch this (*verified locally*), and the [plugin README](https://github.com/vitejs/vite-plugin-react/tree/main/packages/plugin-react) says the same. The eslint-plugin-react-refresh message ["Fast refresh only works when a file only exports components"](https://stackoverflow.com/questions/77407262) is one of the most-asked Vite questions.

```tsx
// Wrong: UserCard.tsx exports a component AND a non-component
export const ROLE_LABELS = { admin: 'Admin', member: 'Member' }
export function UserCard({ role }: { role: keyof typeof ROLE_LABELS }) {
  return <span>{ROLE_LABELS[role]}</span>
}

// Right: move non-components to their own module
// roles.ts
export const ROLE_LABELS = { admin: 'Admin', member: 'Member' }
// UserCard.tsx
import { ROLE_LABELS } from './roles'
export function UserCard({ role }: { role: keyof typeof ROLE_LABELS }) {
  return <span>{ROLE_LABELS[role]}</span>
}
```

The template's rule uses `allowConstantExport: true` (*verified locally*), so exporting a primitive `const` is tolerated. Objects, functions and hooks are not.

**Vue.** `@vitejs/plugin-vue` makes each `.vue` file its own boundary, so the component-only-exports problem doesn't come up. The general rule still applies to plain `.ts` modules: a store or composable file with no boundary above it propagates to the root and reloads the page, unless the library provides its own HMR hook (check your state library's docs).

**Other full-reload causes** ([Troubleshooting](https://vite.dev/guide/troubleshooting)):

- **Circular imports.** A cycle can stop propagation from finding a clean boundary. Run `vite --debug hmr` to see the chain it walked.
- **Import path casing** that doesn't match the filename (`./button` vs `Button.tsx`). Vite detects the change but can't match it in the graph. On a case-insensitive filesystem the import still resolves, so it looks like a mysterious HMR failure. On Linux CI the build fails instead ([Part 7](./vite-build-deploy.md#part-7-the-production-build)).
- **Changes to `vite.config.ts` or `.env`.** The config restarts the server. `.env` changes need a manual restart ([Env and Mode](https://vite.dev/guide/env-and-mode)).

<a id="part-3-advanced"></a>

### 3.3 Advanced — file watching in Docker and WSL2

HMR can only be as good as the file watcher (chokidar), and the watcher depends on the OS delivering file-change events.

| Setup | Events arrive? | Fix |
|---|---|---|
| Native Linux/macOS/Windows, project on local disk | yes | — |
| **WSL2, project on the Windows side** (`/mnt/c/...`) | **no** | move the project into the Linux filesystem (`~/code`), or `usePolling` |
| **Docker bind mount from a Windows/macOS host** | often no | `usePolling`, or run Vite on the host |
| Linux, huge repo | stops at the inotify limit (`ENOSPC`) | raise `fs.inotify.max_user_watches`, or `server.watch.ignored` |

([Server Options](https://vite.dev/config/server-options), [Troubleshooting](https://vite.dev/guide/troubleshooting).)

Not every "HMR doesn't work on WSL2" report is a watcher problem. The top answer to [one such question](https://stackoverflow.com/questions/78357598) turned out to be a backend-integration plugin whose required script tag was missing from the page. When HMR is dead, first check that `/@vite/client` is actually loaded ([Part 1 §1.1](#part-1-beginner)).

```ts
// Wrong: unconditional polling, which burns CPU on every machine on the team
server: { watch: { usePolling: true } }

// Right: poll only where events don't arrive (set the variable in the container / WSL shell)
server: {
  watch: process.env.WATCH_POLL ? { usePolling: true, interval: 300 } : undefined,
}
```

The docs say plainly that polling has a high CPU cost ([Server Options](https://vite.dev/config/server-options)). Moving the project into WSL's own filesystem fixes the root cause and makes everything faster.

**HMR behind a reverse proxy or over HTTPS.** The HMR client connects a WebSocket back to the page's origin. When a proxy terminates TLS or remaps the port, the socket goes to the wrong place and HMR silently stops, with a console error about the WebSocket connection. Since **8.2** the connection options live under `server.ws`. The older `server.hmr.{port,clientPort,host,path,protocol}` still work but are deprecated (*types verified locally*; [Changelog](https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md)):

```ts
// Wrong (deprecated in 8.2+): Vite 5–8.1 examples look like this
server: { hmr: { clientPort: 443 } }

// Right (8.2+)
server: { ws: { clientPort: 443 }, hmr: { overlay: true } }
```

Putting nginx in front of the dev server means forwarding the WebSocket upgrade, because hop-by-hop headers aren't passed through by default ([nginx WebSocket proxying](https://nginx.org/en/docs/http/websocket.html)):

```nginx
location / {
    proxy_pass http://127.0.0.1:5173;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;         # and add that host to server.allowedHosts
}
```

If you get 404s rather than a broken socket, nginx probably never loaded your proxy config at all. That was the diagnosis in [this Stack Overflow thread](https://stackoverflow.com/questions/76182972): a 502 means the proxy config is active but the target is down, while a 404 means nginx is serving from its static root.

**Real Scenario — "HMR works for everyone except the new hire."** A team's dev container setup works on macOS laptops. A new hire on Windows clones the repo into `C:\code`, opens it from WSL2, and nothing hot-reloads. The team suspects their Vite config, then the Docker image, then the browser. The cause: the files live on the Windows filesystem, so the Linux-side watcher gets no inotify events. Cloning into `~/code` inside WSL fixes it without any config change, and every filesystem-heavy command (`npm install`, `tsc -b`) gets faster too.

The polling variable is deliberately **not** `VITE_`-prefixed: shell variables with that prefix are exposed to client code just like `.env` entries ([Part 5](./vite-config-env-assets.md#part-5-env-variables-and-modes)).

<a id="part-3-mastery"></a>

### 3.4 Mastery — the `import.meta.hot` API

You write HMR code yourself in non-framework modules that hold long-lived state: a WebGL scene, a WebSocket client, a store, a module that registers global listeners.

```ts
// src/ticker.ts: a module with side effects that should hot-swap cleanly
let count = import.meta.hot?.data.count ?? 0
const id = setInterval(() => render(++count), 1000)

function render(n: number) {
  document.querySelector('#ticker')!.textContent = String(n)
}

if (import.meta.hot) {
  import.meta.hot.accept()                       // I'm a boundary
  import.meta.hot.dispose((data) => {            // before the old copy is replaced
    clearInterval(id)
    data.count = count                           // mutate data, never reassign it
  })
}
```

The rules ([HMR API](https://vite.dev/guide/api-hmr)):

- **Guard with `if (import.meta.hot)`** so the whole block tree-shakes out of production.
- **Write `import.meta.hot.accept(` literally.** Vite finds boundaries by static analysis, and the pattern is whitespace-sensitive. `const hot = import.meta.hot; hot.accept()` is *not* detected.
- **Mutate `hot.data`, don't reassign it.** It's the same object across module instances.
- **`dispose` vs `prune`:** `dispose` runs before the module is replaced by a new copy. `prune` runs when the module is no longer imported at all.
- **`invalidate()`** from inside an `accept` callback means "I can't handle this update after all". It forces propagation to importers.
- **`accept(dep, cb)` / `accept([deps], cb)`** makes a module a boundary for its dependencies' updates rather than its own. In Vite 8 these must be module paths. **Passing URLs was removed** ([Migration](https://vite.dev/guide/migration)).

**Events.** `import.meta.hot.on(event, cb)` receives `vite:beforeUpdate`, `vite:afterUpdate`, `vite:beforeFullReload`, `vite:beforePrune`, `vite:invalidate`, `vite:error`, `vite:ws:connect` and `vite:ws:disconnect`. `import.meta.hot.send(event, data)` talks to a plugin's `server.ws.on()` ([HMR API](https://vite.dev/guide/api-hmr)). Custom events are typed by augmenting `CustomEventMap` ([Plugin API](https://vite.dev/guide/api-plugin)).

```ts
// Log exactly why the page is about to full-reload (useful while hunting a missing boundary)
if (import.meta.hot) {
  import.meta.hot.on('vite:beforeFullReload', (payload) => {
    console.warn('[hmr] full reload', payload)
  })
}
```

Plugins take part in HMR from the server side with the `hotUpdate` hook (Environment-API-aware; `handleHotUpdate` is its older form). [Vite's breaking-changes page](https://vite.dev/changes/) lists the move to `hotUpdate` as planned. See [Part 10](./vite-plugins-monorepo.md#part-10-plugins).

<a id="part-3-cheat-sheet"></a>

### 3.5 Cheat sheet: Part 3

| Symptom | Likely cause | Check / fix |
|---|---|---|
| Vanilla TS edit reloads page | No boundary anywhere | Expected; add `import.meta.hot.accept()` where state matters |
| React component loses state on edit | File exports a non-component | Split the file; keep `only-export-components` lint on |
| Edit to shared util reloads page | No boundary above it | Expected; or restructure imports |
| Reload on edit, unclear why | Cycle / casing | `vite --debug hmr`; listen for `vite:beforeFullReload` |
| No updates at all (WSL2 / Docker) | Watcher gets no events | Move project into Linux FS, or `server.watch.usePolling` |
| WebSocket errors behind proxy/HTTPS | Client connects to wrong port | `server.ws.clientPort` (8.2+), forward `Upgrade` headers |
| `.env` edit ignored | Env loaded once at start | Restart `vite` |

| `import.meta.hot` | Use |
|---|---|
| `accept()` | self-accepting boundary |
| `accept(cb)` | boundary, receive new module |
| `accept('./dep', cb)` | accept a dependency's updates |
| `dispose(cb)` | clean up before replacement; stash state on `data` |
| `prune(cb)` | clean up when removed from the graph |
| `invalidate()` | give up; propagate to importers |
| `on / off / send` | HMR events and custom client↔server messages |

[↑ Back to top](#table-of-contents)
