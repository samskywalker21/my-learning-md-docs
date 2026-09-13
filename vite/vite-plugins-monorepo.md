# Vite — Plugins & Monorepos (Parts 10–11)

How plugins plug into Vite's pipeline, how to order and scope them, how to write your own, and what changes once your app consumes packages from the same repository.

> **Spec:** this doc follows the shared spec in [`vite-mastery-guide.md`](./vite-mastery-guide.md#about-this-document). Written against **Vite 8.3.0** / **Rolldown 1.2.8**, `@vitejs/plugin-react` **6.1.1**, `@vitejs/plugin-vue` **6.0.8**, verified **September 13, 2026**.

The hook-order diagram in [§10.3](#part-10-advanced) comes from a tracing plugin run against a real dev server and a real build, not from memory. When a plugin behaves differently in dev and build, start there.

---

## Table of Contents

- [Part 10: Plugins](#part-10-plugins)
  - [10.1 Beginner — adding a plugin](#part-10-beginner)
  - [10.2 Working Knowledge — `enforce`, `apply`, and the framework plugins](#part-10-working)
  - [10.3 Advanced — writing a plugin: hooks, filters, virtual modules](#part-10-advanced)
  - [10.4 Mastery — dev-server hooks, HMR, environments, debugging](#part-10-mastery)
  - [10.5 Cheat sheet: Part 10](#part-10-cheat-sheet)
- [Part 11: Monorepos & Workspaces](#part-11-monorepos-and-workspaces)
  - [11.1 Working Knowledge — how a linked package is resolved](#part-11-working)
  - [11.2 Advanced — duplicate frameworks, `server.fs`, CommonJS packages](#part-11-advanced)
  - [11.3 Mastery — source vs. built packages, shared config](#part-11-mastery)
  - [11.4 Cheat sheet: Part 11](#part-11-cheat-sheet)

---

<a id="part-10-plugins"></a>

## Part 10: Plugins

<a id="part-10-beginner"></a>

### 10.1 Beginner — adding a plugin

A plugin is an object with a `name` and some hook functions. You install one as a dev dependency and add it to `plugins` ([Plugin API](https://vite.dev/guide/api-plugin)):

```ts
import react from '@vitejs/plugin-react'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [react()],
})
```

Two array rules make conditional setups straightforward ([Shared Options](https://vite.dev/config/shared-options)):

- **Falsy entries are ignored**: `plugins: [react(), isCI && reporter()]`.
- **Nested arrays are flattened**. `react()` actually returns an **array** of plugins (`Plugin[]`, *verified locally* in its types), and a preset can bundle several plugins as one entry.

**Where to find plugins.** Vite's official plugins (`@vitejs/plugin-react`, `@vitejs/plugin-vue`, `@vitejs/plugin-vue-jsx`, `@vitejs/plugin-legacy`), framework and tool plugins (`@tailwindcss/vite`, `vite-plugin-checker`), and **most Rollup/Rolldown plugins**, which work unchanged: Rolldown implements Rollup's plugin API, and `rollup-plugin-visualizer` ran fine on Vite 8 (*verified locally*, [Part 7](./vite-build-deploy.md#part-7-working)). The [Vite 8 announcement](https://vite.dev/blog/announcing-vite8) introduced a searchable plugin registry at registry.vite.dev.

<a id="part-10-working"></a>

### 10.2 Working Knowledge — `enforce`, `apply`, and the framework plugins

**Order.** Array order matters, but `enforce` moves a plugin relative to Vite's own core plugins ([Plugin API](https://vite.dev/guide/api-plugin)):

```text
  alias
   → user plugins with enforce: 'pre'
   → Vite core plugins
   → user plugins without enforce          ← where most plugins belong
   → Vite build plugins
   → user plugins with enforce: 'post'
   → Vite post-build plugins (minify, manifest, reporting)
```

Separately, each hook can set its own `order: 'pre' | 'post'` inside the object form (`transform: { order: 'pre', handler }`), which only reorders that hook ([Plugin API](https://vite.dev/guide/api-plugin)).

**Scope with `apply`.**

```ts
// Wrong: a build-time-only plugin also runs in dev, slowing every request or crashing
plugins: [compressAssets()]

// Right
plugins: [{ ...compressAssets(), apply: 'build' }]

// Precise: build, but not the SSR build
plugins: [{ ...compressAssets(), apply: (config, { command }) => command === 'build' && !config.build?.ssr }]
```

([Plugin API — Conditional Application](https://vite.dev/guide/api-plugin).) If a plugin returns an array, like `react()`, spreading it into one object doesn't work. Filter inside the function config instead (`command === 'build' && plugin()`).

**Build-only Rolldown plugins** can go in `build.rolldownOptions.plugins`, where they only take part in the build ([Plugin API](https://vite.dev/guide/api-plugin)).

**`@vitejs/plugin-react` v6** (*options verified locally* from 6.1.1's types; [README](https://github.com/vitejs/vite-plugin-react/tree/main/packages/plugin-react)):

| Option | Default | Use |
|---|---|---|
| `include` / `exclude` | `/\.[tj]sx?$/` / `/\/node_modules\//` | add `.mdx`; exclude worker or non-React JSX files |
| `jsxImportSource` | `'react'` (or tsconfig's) | Emotion, Preact compat |
| `jsxRuntime` | `'automatic'` | `'classic'` only for very old setups |
| `reactRefreshHost` | — | module federation remotes |
| `compiler` | `false` | **experimental** Rust React Compiler; needs `oxc-transform-react` installed |

> **Outdated convention alert.** Pre-v6 guides configure React Compiler or custom Babel plugins through `react({ babel: { plugins: [...] } })`. v6 removed Babel from the plugin: Fast Refresh runs on Oxc ([Vite 8 announcement](https://vite.dev/blog/announcing-vite8)), and there is no `babel` option in 6.1.1's types (*verified locally*). Current options for React Compiler ([README](https://github.com/vitejs/vite-plugin-react/tree/main/packages/plugin-react)):

```ts
// Babel-based (the stable React Compiler)
import react, { reactCompilerPreset } from '@vitejs/plugin-react'
import babel from '@rolldown/plugin-babel'

export default defineConfig({
  plugins: [react(), babel({ presets: [reactCompilerPreset()] })],
})
```

```ts
// Rust-based (experimental, faster)
export default defineConfig({
  plugins: [react({ compiler: true })],   // npm i -D oxc-transform-react
})
```

The `react-compiler-ts` template in `create-vite` sets up the compiler for you ([Getting Started](https://vite.dev/guide/)).

**`@vitejs/plugin-vue` 6** takes SFC compiler options: `script`, `template`, and a `features` object. `features.propsDestructure` and `features.customElement` **replace** the deprecated top-level `script.propsDestructure` and `customElement` (*verified locally* from 6.0.8's types):

```ts
import vue from '@vitejs/plugin-vue'
export default defineConfig({
  plugins: [
    vue({
      template: { compilerOptions: { isCustomElement: (tag) => tag.startsWith('ion-') } },
      features: { customElement: /\.ce\.vue$/ },
    }),
  ],
})
```

<a id="part-10-advanced"></a>

### 10.3 Advanced — writing a plugin: hooks, filters, virtual modules

**Hook order, traced.** Three tracing plugins (`enforce: 'pre'`, default, `'post'`) logged every hook call (*verified locally*, 8.3.0; `main.ts` is the only filtered module):

```text
  ── vite build ─────────────────────────   ── vite (dev) ─────────────────────────
  config            pre → normal → post      config            pre → normal → post
  configResolved    pre → normal → post      configResolved    pre → normal → post
  options           pre → normal → post      options           ×2 (client + ssr environments)
  buildStart        pre → normal → post      configureServer   pre → normal → post
  resolveId(main)   pre ONLY  ◄─┐            buildStart        pre → normal → post
  load(main)        pre → normal → post      transformIndexHtml  on GET /  pre → normal → post
  transform(main)   pre → normal → post      resolveId(main)   pre ONLY  ◄─┐
  buildEnd          pre → normal → post      load / transform  per request, cached
  generateBundle    pre → normal ─┐                             │
  transformIndexHtml pre→normal→post│ (inside                  (no buildEnd / generateBundle /
  generateBundle            post ◄─┘  generateBundle)            writeBundle until server close)
  writeBundle       pre → normal → post
  closeBundle       pre → normal → post
                                  └── resolveId is "first result wins": Vite's core resolver
                                      returned an id, so normal/post plugins never saw it
```

What the trace shows:

1. **`resolveId` and `load` are first-wins hooks.** If a plugin earlier in the order returns a value, later plugins' `resolveId` never runs for that id. Here it was Vite's core resolver, which comes before normal plugins. **To resolve something Vite would otherwise resolve, you need `enforce: 'pre'`.** `transform` is sequential: every plugin gets the previous plugin's output.
2. **Dev has no bundle phase.** `generateBundle`, `renderChunk` and `writeBundle` never run in `vite` dev, and `buildEnd`/`closeBundle` wait for the server to close ([Plugin API — Universal Hooks](https://vite.dev/guide/api-plugin)). A plugin that does its work in `generateBundle` does nothing in dev. That's the most common "only works in build" plugin bug.
3. **Dev calls `options` once per environment** (client and ssr).
4. **`transformIndexHtml` in build runs inside the bundle phase**, after chunks exist, so it can inject hashed filenames.

**Hook filters.** In Vite 8, plugin hooks are called across a Rust ↔ JavaScript boundary. A `transform` that checks `if (!id.endsWith('.md')) return` still pays that crossing cost for every module. A declarative **filter** lets Rolldown skip the call entirely ([Plugin API — Hook Filters](https://vite.dev/guide/api-plugin)):

```ts
// Wrong (slow in Vite 8): invoked for every module in the graph
transform(code, id) {
  if (!id.endsWith('.md')) return
  return { code: compileMarkdown(code), map: null, moduleType: 'js' }
}

// Right: Rolldown only calls the handler for matching ids
transform: {
  filter: { id: /\.md$/ },
  handler(code) {
    return { code: compileMarkdown(code), map: null, moduleType: 'js' }
  },
},
```

Keep the check inside the handler as well if your plugin must also support older Vite or Rollup versions that ignore `filter` ([Plugin API](https://vite.dev/guide/api-plugin)). Note the `moduleType: 'js'`: in Vite 8, a `load`/`transform` that turns non-JS content into JS **must say so** ([Migration](https://vite.dev/guide/migration)).

**Virtual modules** expose generated code as an importable module ([Plugin API — Virtual Modules](https://vite.dev/guide/api-plugin)). This one was *verified locally* in dev and build:

```ts
// vite.config.ts
import { defineConfig, type Plugin } from 'vite'

function buildInfo(): Plugin {
  const id = 'virtual:build-info'
  const resolvedId = '\0' + id            // \0 = "internal module; other plugins, leave it alone"
  let mode = ''
  return {
    name: 'build-info',
    configResolved(config) {
      mode = config.mode
    },
    resolveId: {
      filter: { id: /^virtual:build-info$/ },
      handler(source) {
        if (source === id) return resolvedId
      },
    },
    load: {
      filter: { id: /^\0virtual:build-info$/ },
      handler(loadId) {
        if (loadId === resolvedId) {
          return `export const buildInfo = ${JSON.stringify({ time: new Date().toISOString(), mode })}`
        }
      },
    },
  }
}

export default defineConfig({ plugins: [buildInfo()] })
```

```ts
// src/virtual.d.ts: TypeScript can't see virtual modules
declare module 'virtual:build-info' {
  export const buildInfo: { time: string; mode: string }
}
```

```ts
// src/main.ts
import { buildInfo } from 'virtual:build-info'
```

In dev, the browser fetches it from `/@id/__x00__virtual:build-info` (the `\0` is encoded as `__x00__`), which returned `export const buildInfo = {"time":"…","mode":"development"}`. In build it was inlined as `{time:…,mode:"production"}` (*verified locally*).

**Vite-specific hooks** you'll use most ([Plugin API](https://vite.dev/guide/api-plugin)):

| Hook | Runs | Use |
|---|---|---|
| `config(config, env)` | before resolution | return a partial config to merge (defaults, aliases, `define`) |
| `configResolved(config)` | after resolution | read the final config into a closure variable |
| `configureServer(server)` | dev start | add middleware; return a function to run **after** Vite's middleware |
| `configurePreviewServer(server)` | preview start | same, for `vite preview` |
| `transformIndexHtml(html, ctx)` | per HTML request / in bundle | inject tags, rewrite HTML |
| `hotUpdate` (`handleHotUpdate`) | on file change | custom HMR ([§10.4](#part-10-mastery)) |
| `closeServer` / `closePreviewServer` | server shutdown (8.3+) | clean up watchers, child processes ([Changelog](https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md)) |

```ts
// Return tags instead of string-replacing HTML (safer, and composes with other plugins)
transformIndexHtml() {
  return [
    { tag: 'meta', attrs: { name: 'app-version', content: process.env.npm_package_version }, injectTo: 'head' },
  ]
},
```

**Conventions.** Name Vite-only plugins `vite-plugin-*`. Name plugins that also work in plain Rolldown/Rollup `rolldown-plugin-*` ([Plugin API](https://vite.dev/guide/api-plugin)). Use `normalizePath()` from `vite` before comparing ids, because Windows paths use `\` ([Plugin API](https://vite.dev/guide/api-plugin)). The trace above ran on Windows, and ids arrived with `/`, but paths you build yourself may not.

**Real Scenario — the sitemap that never updated.** A team writes a plugin that scans `src/routes/` in `buildStart`, caches the list, and emits `sitemap.xml` in `generateBundle`. In dev they add a route and fetch `/sitemap.xml`: 404, because `generateBundle` doesn't run in dev. They add `configureServer` middleware to serve it, but it keeps showing the old routes, because `buildStart` ran once at server start. The fix: in dev, compute the sitemap *per request* in the middleware (or invalidate the cache from `server.watcher.on('add', …)`). In build, do it in `generateBundle`. Two code paths, one module-level helper.

<a id="part-10-mastery"></a>

### 10.4 Mastery — dev-server hooks, HMR, environments, debugging

**Middleware ordering in `configureServer`** ([Plugin API](https://vite.dev/guide/api-plugin)):

```ts
configureServer(server) {
  // Runs BEFORE Vite's internal middleware: can intercept anything, including /src/*
  server.middlewares.use('/__health', (_req, res) => res.end('ok'))

  // Returned function runs AFTER Vite's internal middlewares are installed:
  // for requests Vite's own handlers pass through
  return () => {
    server.middlewares.use((req, _res, next) => {
      // …
      next()
    })
  }
},
```

**Talking to the client.** A plugin pushes to the browser with `server.ws.send('my:event', data)` and listens with `server.ws.on('my:event', …)`. The client uses `import.meta.hot.on/send` ([Plugin API — Client-server Communication](https://vite.dev/guide/api-plugin), [Part 3 §3.4](./vite-dev-server.md#part-3-mastery)). Type the events by augmenting `CustomEventMap` in a `.d.ts`.

**Environment-aware plugins.** Since Vite 6, a dev server can host several environments (`client`, `ssr`, custom ones). The trace above shows `options` running twice. Hooks receive `this.environment`, and the older `options.ssr` flag is how pre-Environment-API plugins branched ([SSR guide](https://vite.dev/guide/ssr)). Three helpers exist in 8.3 (*types verified locally*):

- **`applyToEnvironment(environment)`** on a plugin decides per environment whether it applies. Since 8.3, Vite warns if a plugin returned from it uses hooks that can't work per environment ([Changelog](https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md)).
- **`perEnvironmentPlugin(name, fn)`** builds a plugin instance per environment.
- **`perEnvironmentState(init)`** keeps state per environment instead of in a shared closure, so a client transform doesn't see SSR's cache.

[Vite's breaking-changes page](https://vite.dev/changes/) lists **`this.environment` in hooks** and the **`hotUpdate` hook** as *planned* changes, and **shared plugins during build** as *under consideration*. New plugins should use `this.environment` and `hotUpdate` now.

`hotUpdate` receives `{ type: 'create' | 'update' | 'delete', file, timestamp, modules, read, server }`, with `this.environment` bound to the dev environment. `handleHotUpdate` gets an environment-less `HmrContext` (*types verified locally*).

```ts
// hotUpdate: Environment-API-aware replacement for handleHotUpdate
hotUpdate({ file, modules }) {
  if (this.environment.name !== 'client') return
  if (file.endsWith('.content.json')) {
    this.environment.hot.send('content:changed', { file })
    return []                     // handled: no default HMR for these modules
  }
},
```

**Debugging a plugin pipeline.**

- **`vite-plugin-inspect`** (12.0.2 at time of writing; *not run in this pass*) shows each module's transform result after every plugin, which is the fastest way to find which plugin broke something ([Troubleshooting](https://vite.dev/guide/troubleshooting)).
- **`vite --debug transform`** logs transform timings to find slow plugins ([Performance](https://vite.dev/guide/performance)).
- **`vite --profile`**, then press `p + enter` in dev, writes a `.cpuprofile` for speedscope. `vite build --profile` works too, and since 8.3 accepts a name ([Troubleshooting](https://vite.dev/guide/troubleshooting), [Changelog](https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md)).
- **Avoid slow `config`, `configResolved` and `buildStart` hooks.** They block server startup. Import heavy dependencies lazily inside the hook that needs them ([Performance](https://vite.dev/guide/performance)).

**Rolldown plugin compatibility limits.** A Rolldown or Rollup plugin works as a Vite plugin if it doesn't use `moduleParsed`, doesn't depend on Rolldown-only options, and doesn't tightly couple its bundle and output phases ([Plugin API](https://vite.dev/guide/api-plugin)). Also, **Rollup's parallel hooks run sequentially** in Vite 8 ([Migration](https://vite.dev/guide/migration)), so a plugin that relied on parallel `buildEnd` for speed gets slower but keeps working.

<a id="part-10-cheat-sheet"></a>

### 10.5 Cheat sheet: Part 10

| Need | How |
|---|---|
| Run before Vite's resolver | `enforce: 'pre'` |
| Run after minification-adjacent work | `enforce: 'post'` |
| Reorder one hook only | `transform: { order: 'pre', handler }` |
| Build-only / dev-only | `apply: 'build'` / `apply: 'serve'` |
| Skip non-matching modules cheaply | `filter: { id: /\.md$/ }` |
| Non-JS → JS in `load`/`transform` | return `moduleType: 'js'` |
| Generated import | `virtual:x` → resolve to `'\0virtual:x'` → `load` |
| Dev middleware | `configureServer` (return fn = after Vite's) |
| Inject HTML | `transformIndexHtml` returning tag descriptors |
| Custom HMR | `hotUpdate` + `this.environment.hot.send` |
| Per-environment behaviour | `this.environment`, `applyToEnvironment`, `perEnvironmentState` |
| React Compiler (v6) | `@rolldown/plugin-babel` + `reactCompilerPreset()`, or `react({ compiler: true })` |
| Find a broken transform | `vite-plugin-inspect`, `vite --debug transform` |

| Hook runs in… | dev | build |
|---|---|---|
| `config`, `configResolved`, `options`, `buildStart` | ✔ at start | ✔ |
| `resolveId`, `load`, `transform` | ✔ per request | ✔ per module |
| `configureServer`, `hotUpdate` | ✔ | ✘ |
| `renderChunk`, `generateBundle`, `writeBundle` | ✘ | ✔ |
| `buildEnd`, `closeBundle` | at server close | ✔ |
| `transformIndexHtml` | ✔ per HTML request | ✔ inside `generateBundle` |

[↑ Back to top](#table-of-contents)

---

<a id="part-11-monorepos-and-workspaces"></a>

## Part 11: Monorepos & Workspaces

*Tiers collapsed: Working Knowledge → Mastery.* There's no Beginner tier: a monorepo already assumes you know Parts 2, 4 and 7. Vitest's `projects` for monorepos are in [Part 12](./vite-testing-vitest.md#part-12-advanced).

Every behaviour in this Part was *verified locally* against an npm-workspaces repo: `packages/ui` (TypeScript source, React peer) consumed by `apps/web` (the `react-ts` template).

<a id="part-11-working"></a>

### 11.1 Working Knowledge — how a linked package is resolved

```text
  mono/
  ├── package.json            "workspaces": ["packages/*", "apps/*"]
  ├── node_modules/
  │   ├── react/              ← hoisted, shared
  │   └── @acme/ui ──symlink──► ../../packages/ui
  ├── packages/ui/
  │   ├── package.json        "exports": { ".": "./src/index.ts" }  ← points at SOURCE
  │   └── src/index.ts
  └── apps/web/
      ├── vite.config.ts
      └── src/main.tsx        import { useCounter } from '@acme/ui'
```

What Vite does with `import { useCounter } from '@acme/ui'` in dev (*verified locally*):

```js
// GET /src/main.tsx (excerpt)
import { useCounter, UI_VERSION } from "/@fs/C:/…/mono/packages/ui/src/index.ts"
```

```js
// GET /@fs/C:/…/mono/packages/ui/src/index.ts (excerpt)
import __vite__cjsImport0_react from "/node_modules/.vite/deps/react.js?v=2546f15e";
```

Four observations:

1. **The symlink is followed to its real path** (`resolve.preserveSymlinks` defaults to `false`, [Shared Options](https://vite.dev/config/shared-options)), and the file is served through `/@fs/`.
2. **The linked package is treated as source, not pre-bundled.** `.vite/deps/` contained only React, with no `@acme_ui.js`. Vite detects that the dependency doesn't resolve inside `node_modules` ([Dependency Pre-Bundling](https://vite.dev/guide/dep-pre-bundling)). Its files go through your plugins (Oxc, React Refresh) and take part in HMR like your own code.
3. **Its own bare imports (`react`) resolve to the app's pre-bundled copy**, as long as there is one copy. §11.2 covers what happens when there isn't.
4. **`exports` pointing straight at `.ts` source works** in both dev and build with no build step for the package. [§11.3](#part-11-mastery) covers the trade-off.

**Why "my linked package doesn't update" happens anyway.** The above applies to workspace packages. A package installed normally into `node_modules`, or `npm link`ed from **outside** the workspace in a way that doesn't look like source, *is* pre-bundled and cached, so edits don't show up ([Part 2 §2.2](./vite-dev-server.md#part-2-working)). Vite's troubleshooting guide says `npm link` doesn't trigger re-optimization and recommends package-manager `overrides` or `vite --force` ([Troubleshooting](https://vite.dev/guide/troubleshooting)).

<a id="part-11-advanced"></a>

### 11.2 Advanced — duplicate frameworks, `server.fs`, CommonJS packages

**Duplicate React/Vue: "Invalid hook call".** Hooks and Vue's reactivity rely on module-level singletons. If the app and a workspace package each resolve their *own* copy of the framework, a hook called inside the package talks to a React the app never rendered with.

*Reproduced locally*: `packages/ui` declared `"dependencies": { "react": "19.1.0" }` while the app used 19.3, so npm nested a second copy at `packages/ui/node_modules/react`. The unminified production build then contained **two** `react/cjs/react.production.js` regions. In a browser, calling `useCounter()` would throw "Invalid hook call".

```json
// Wrong: the library brings its own React
{ "name": "@acme/ui", "dependencies": { "react": "19.1.0" } }

// Right: the consumer provides React
{ "name": "@acme/ui", "peerDependencies": { "react": "^19" }, "devDependencies": { "react": "^19.3.0" } }
```

```ts
// Belt and braces in the app: force one copy no matter what got installed
export default defineConfig({
  resolve: { dedupe: ['react', 'react-dom'] },   // Vue: ['vue']
})
```

With `resolve.dedupe`, the same broken install built with **one** `react.production.js` (*verified locally*). `dedupe` forces Vite to resolve listed packages from the project root ([Shared Options](https://vite.dev/config/shared-options)). It is a safety net. The `peerDependencies` fix is the real cure, because other tools (Vitest in `node` mode, TypeScript, SSR externals) don't read Vite's `dedupe`. This combination (peer deps + externals + `dedupe`) is the top answer for the [component-library version of the bug](https://stackoverflow.com/questions/72764295). In a [Turborepo report](https://stackoverflow.com/questions/76156784), the fix was aligning every app and package on the same React version, which is the same cause approached from the version side.

**`server.fs.allow`: "outside of Vite serving allow list".** The dev server refuses to serve files outside an allow list (`server.fs.strict`, default `true`, [Server Options](https://vite.dev/config/server-options)). By default the list is the **detected workspace root**: Vite walks up from the project looking for a workspace definition.

*Verified locally*, requesting a non-imported file in `packages/ui` through `/@fs/`:

| `server.fs.allow` | Result |
|---|---|
| (default) | **200**: the npm workspace root was detected, so all of `mono/` is allowed |
| `['.']` | **403** `Restricted … outside of Vite serving allow list` |

Files that are *imported* by your app were still served under `['.']`, since they are already in the module graph. The error typically shows up for files referenced **indirectly**: a font in a linked package's CSS, a `?url` asset, a package linked from outside the repo. It also shows up after someone adds a custom `allow`:

```ts
// Wrong: replaces the default, so the workspace root is no longer allowed
server: { fs: { allow: ['../../shared-assets'] } }

// Right: keep the default and extend it
import { defineConfig, searchForWorkspaceRoot } from 'vite'
export default defineConfig({
  server: {
    fs: { allow: [searchForWorkspaceRoot(process.cwd()), '../../shared-assets'] },
  },
})
```

(`searchForWorkspaceRoot` is exported from `vite`, *types verified locally*.) Since 8.2, entries from the top-level `input` option are allowed automatically ([Changelog](https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md)). Don't set `fs.strict: false` to make the error go away. `fs.deny` (which protects `.env`) still applies, but everything else on disk becomes reachable to anything that can reach your dev server.

**CommonJS workspace packages.** Because linked packages are served as source, a workspace package written in CommonJS (`module.exports`) arrives in the browser unconverted and fails. Add it to `optimizeDeps.include` so it gets pre-bundled and converted ([Dependency Pre-Bundling](https://vite.dev/guide/dep-pre-bundling)). In production builds, Rolldown handles CommonJS natively.

```ts
optimizeDeps: { include: ['@acme/legacy-utils'] }
```

**Real Scenario — the dependency bump that broke login.** A team extracts `useAuth` into `packages/auth`, with `react` listed under `dependencies`. For months that's harmless, because every package asks for the same range and npm hoists one copy. Then an automated dependency PR bumps React in `packages/auth` only. The lockfile now nests a second React under `packages/auth/node_modules`, the PR's unit tests pass (they render `packages/auth` in isolation), and the merged app throws "Invalid hook call" as soon as the login form renders. The fix: `peerDependencies`, `resolve.dedupe` in the app, and a CI step running `npm ls react` that fails on more than one version.

<a id="part-11-mastery"></a>

### 11.3 Mastery — source vs. built packages, shared config

**Consuming workspace packages as source vs. as built output:**

| | `exports` → `./src/index.ts` (source) | `exports` → `./dist/index.js` (built) |
|---|---|---|
| Build step for the package | none | `vite build` / `tsdown` before the app can use it |
| App dev experience | HMR through the package; edits show up instantly | stale until rebuilt; needs a watcher |
| Who compiles it | **the app's** plugins and `build.target` | the package's own build |
| Type-checking | app's `tsc -b` needs project references to the package | package ships `.d.ts` |
| Works for non-Vite consumers | no: Node, Jest, other bundlers can't import raw `.tsx` | yes |
| Package needs its own JSX/CSS plugins? | **no** (and must not rely on any the app lacks) | yes |

Source consumption is the low-friction default for **internal** packages used only by Vite apps. Publish-ready packages need a build ([Part 9](./vite-build-deploy.md#part-9-library-mode)). A middle path is conditional exports:

```json
{
  "exports": {
    ".": {
      "development": "./src/index.ts",
      "default": "./dist/index.js"
    }
  }
}
```

`resolve.conditions` includes `development|production` by default, meaning `development` in dev and `production` in build ([Shared Options](https://vite.dev/config/shared-options)). With the map above, a Vite app gets source in dev and the built file in production. *Verified locally*: the dev server imported `packages/ui/src/index.ts`, and `vite build` bundled `packages/ui/dist/index.js`. That's convenient, but now dev and prod run *different code*. Pick it deliberately, and keep CI building from `dist`.

**Sharing Vite config across apps.** Put a base config in a package and merge it:

```ts
// packages/config/vite.base.ts
import react from '@vitejs/plugin-react'
import { defineConfig } from 'vite'
export default defineConfig({
  plugins: [react()],
  resolve: { dedupe: ['react', 'react-dom'] },
  build: { sourcemap: 'hidden' },
})
```

```ts
// apps/web/vite.config.ts
import { defineConfig, mergeConfig } from 'vite'
import base from '@acme/config/vite.base.ts'

export default mergeConfig(base, defineConfig({
  server: { port: 5174 },
}))
```

`mergeConfig` deep-merges and **concatenates arrays**, including `plugins` (*verified locally*: merging `plugins: [a]` with `plugins: [b]` gave `[a, b]`, and `dedupe` arrays concatenated the same way). Adding `react()` again in the app config gives you two React plugins and confusing Fast Refresh errors. (The `.ts` extension in the import follows the native-loader guidance in [Part 4 §4.4](./vite-config-env-assets.md#part-4-mastery).)

**One `.env` for the repo.** `envDir: '../../'` points every app at a root `.env` ([Shared Options](https://vite.dev/config/shared-options)). The `VITE_` rules from [Part 5](./vite-config-env-assets.md#part-5-env-variables-and-modes) now apply repo-wide, so one app's `VITE_` secret mistake leaks into every app.

**Task orchestration** (Turborepo, Nx, `pnpm -r`) is outside Vite. See the legacy [`monorepo` guide](../legacy/monorepo/monorepo-mastery-guide.md) until it's rewritten.

<a id="part-11-cheat-sheet"></a>

### 11.4 Cheat sheet: Part 11

| Symptom | Cause | Fix |
|---|---|---|
| "Invalid hook call" / two Vue instances | duplicate framework copy | `peerDependencies` + `resolve.dedupe` + `npm ls react` in CI |
| "outside of Vite serving allow list" | custom `fs.allow` replaced the workspace root, or file outside repo | `allow: [searchForWorkspaceRoot(process.cwd()), …]` |
| Linked package edits not showing | it's pre-bundled (not detected as source) | `vite --force`; use workspaces/overrides instead of `npm link` |
| CJS workspace package fails in dev | linked deps served as source | `optimizeDeps.include: ['pkg']` |
| Workspace `.tsx` fails in Node tools | source-exported package | build it, or conditional `exports` |
| Plugin runs twice | `mergeConfig` concatenated `plugins` | keep plugins in one layer |
| Two apps need the same config | copy-paste drift | shared `vite.base.ts` + `mergeConfig` |

[↑ Back to top](#table-of-contents)
