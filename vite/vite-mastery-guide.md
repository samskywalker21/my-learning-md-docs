# Mastering Vite — A Hands-On Reference Guide

## About This Document

- **Framing:** Curiosity-driven. No upcoming task is driving this — the goal is durable, hands-on understanding of Vite itself. Structured as a guided tutorial (runnable examples/exercises you're meant to actually try in a scaffolded project), with theory sprinkled in only where it explains a mechanism (e.g. *why* the dev server doesn't bundle on every request), not as standalone history.
- **Scope:** "Full picture, dev-first" — the dev server and module graph, the config system, the import/runtime features, the plugin ecosystem (using plugins in depth; authoring them at a working-knowledge level), and the production build (Rolldown), with a lighter pass on SSR/Environment API and library mode (both are "know it exists and when to reach for it," not full depth — app SSR is explicitly meta-framework territory per Vite's own docs). Plugin *authoring internals* (full hook-by-hook reference) are intentionally out of scope — flagged inline as a natural follow-up doc if that becomes goal-driven later.
- **Depth tiers:** Major sections use **Beginner → Working Knowledge → Advanced → Mastery**. Collapsed where a topic doesn't have four honest tiers — e.g. SSR and Library Mode stay at Working Knowledge since going deeper would mean documenting a meta-framework, not Vite.
- **Version basis:** **Vite 8** (stable — released March 12, 2026, per the [official announcement](https://vite.dev/blog/announcing-vite8)). Vite 8 replaced the esbuild (dev) + Rollup (build) two-bundler split with **Rolldown**, a single Rust-based bundler, plus the **Oxc** transformer replacing esbuild's transform step. Anywhere Vite 7 and earlier behaved differently (`rollupOptions` → `rolldownOptions`, `esbuild` config → `oxc` config), it's flagged explicitly since this is exactly where outdated blog posts and tutorials mislead people.
- **Assumed background:** Comfortable with Node.js, npm/pnpm, and basic JS/TS module syntax (`import`/`export`). No prior bundler experience assumed — Webpack/Rollup concepts are explained where Vite's model diverges from them.
- **Sourcing standard:** [vite.dev](https://vite.dev) official docs are the primary source, cited inline. Stack Overflow used for real-world gotchas. Checked as of September 2026.
- **Update instructions for future-me:** Match this structure (numbered Parts, tier subheadings, wrong-vs-right pairs for gotchas, TOC + back-to-top links). If Vite ships a new major version, update the "Version basis" line and re-check the Rolldown-migration table in Part 7 first — that's the section most likely to drift.

---

## Table of Contents

1. [Orientation — What Vite Actually Is](#1-orientation--what-vite-actually-is)
2. [Part 1 — The Dev Server & Module Graph](#2-part-1--the-dev-server--module-graph)
3. [Part 2 — Configuration System](#3-part-2--configuration-system)
4. [Part 3 — Import & Runtime Features](#4-part-3--import--runtime-features)
5. [Part 4 — The Plugin Ecosystem](#5-part-4--the-plugin-ecosystem)
6. [Part 5 — Production Build (Rolldown)](#6-part-5--production-build-rolldown)
7. [Part 6 — SSR & the Environment API](#7-part-6--ssr--the-environment-api)
8. [Part 7 — Vite 8 Internals: Rolldown & Oxc](#8-part-7--vite-8-internals-rolldown--oxc)
9. [Cheat Sheets](#9-cheat-sheets)
10. [Suggested Learning Order](#10-suggested-learning-order)
11. [Quick Self-Check](#11-quick-self-check)

---

## 1. Orientation — What Vite Actually Is

**What it is.** Vite ("vite," French for "fast," pronounced `/vit/`) is a frontend tooling layer with two very different jobs bolted together: an **unbundled dev server** that serves your source files over native ES modules, and a **production bundler** (Rolldown, as of Vite 8) that packages everything for deployment. The two halves exist because "fast in dev" and "optimal in prod" pull in opposite directions — dev wants to skip work, prod wants to do all of it well.

**Why it matters.** Older bundlers (Webpack, pre-Vite Rollup setups) rebuild a dependency graph and re-bundle on every file change, even in dev — so dev-server startup and rebuild time scale with your *entire* app's size. Vite's dev server instead serves modules directly over the browser's native `import`, transforming each file only when the browser actually requests it. Startup time becomes roughly constant regardless of app size.

**Try it now — scaffold something to follow along with:**

```bash
npm create vite@latest my-vite-app -- --template vanilla-ts
cd my-vite-app
npm install
npm run dev
```

Open the printed `localhost` URL, then open your browser's Network tab. Notice: no single giant `bundle.js` — instead you'll see individual requests for `main.ts`, its imports, etc. That's the dev server working exactly as designed, not a bug.
([Getting Started — Vite](https://vite.dev/guide/))

[⬆ back to top](#table-of-contents)

---

## 2. Part 1 — The Dev Server & Module Graph

Covers: native ESM serving, dependency pre-bundling, HMR.

### Beginner

Vite's dev server does three things when your browser requests a module:

1. **Resolve** the import path to a real file.
2. **Transform** it if needed (TS → JS, JSX → JS, Vue SFC → JS, etc.).
3. **Serve** it back as a native ES module — the browser then makes its *own* requests for whatever that module imports.

```ts
// main.ts
import { greet } from './greet.ts'
console.log(greet('world'))
```

The browser requests `main.ts`, Vite transforms and serves it, the browser sees the `import` and requests `greet.ts` next — all lazily, only for code actually reached.

### Working Knowledge

**Dependency pre-bundling (`optimizeDeps`).** Your own source files map one-to-one to browser requests just fine. `node_modules` packages don't: a library like `lodash-es` can be made of *hundreds* of internal ES modules, which would mean hundreds of waterfall requests just to use one import. Vite solves this by pre-bundling dependencies with esbuild-speed tooling into a single flat module per package, cached in `node_modules/.vite`.

```bash
# Force Vite to re-run pre-bundling (e.g. after manually editing a dependency)
vite --force
```

**Wrong vs. right — debugging a "stale dependency" issue:**

```bash
# Wrong — deleting node_modules and reinstalling to fix a weird dev-server error,
# without checking whether it's actually a stale pre-bundle cache
rm -rf node_modules && npm install

# Right — clear just Vite's dep cache first, it's a much cheaper first guess
rm -rf node_modules/.vite && npm run dev
```

**Real Scenario.** You `npm link` a local package for testing, edit it, and the dev server keeps serving the old code. This is almost always the pre-bundle cache — Vite pre-bundled the linked package once and doesn't know it changed on disk. `vite --force` or deleting `node_modules/.vite` fixes it immediately, and is the first thing to try before assuming your build is broken.
([Dependency Pre-Bundling — Vite](https://vite.dev/guide/dep-pre-bundling))

### Advanced — HMR (Hot Module Replacement)

Vite's HMR is *module-graph-aware*: when a file changes, Vite walks up the module graph from that file looking for a module that has registered an HMR boundary (`import.meta.hot.accept`). If it finds one, only that module's subtree is invalidated and reloaded — not the whole page.

```ts
if (import.meta.hot) {
  import.meta.hot.accept((newModule) => {
    // patch the running app with the new module's exports
  })

  import.meta.hot.dispose((data) => {
    // cleanup before this module is replaced (e.g. clear an interval)
  })
}
```

Framework plugins (`@vitejs/plugin-vue`, `@vitejs/plugin-react`) wire this up for you automatically — you rarely write raw HMR code in an app that uses one, but understanding the boundary-walking model explains *why* editing a leaf component hot-swaps instantly while editing a shared utility with no HMR boundary triggers a full page reload.
([HMR API — Vite](https://vite.dev/guide/api-hmr))

```
Module Graph (simplified)

  main.ts
    ├─▶ App.vue  ◀── HMR boundary (framework plugin registers this)
    │     └─▶ Button.vue  ◀── edit here → only this subtree reloads
    └─▶ utils.ts  ◀── no boundary → edit here → walks up to main.ts,
                       finds no boundary either → full page reload
```

### Mastery

The dev server and the production build **share the same plugin pipeline** (both are Rolldown-plugin-compatible as of Vite 8) but run it differently: dev applies `transform` per-request, lazily, with results cached per-module; build runs the full Rolldown bundling pass up front. This is *why* a plugin bug can behave differently in `vite dev` vs `vite build` — it's not two unrelated tools, it's one plugin pipeline driven two different ways.

[⬆ back to top](#table-of-contents)

---

## 3. Part 2 — Configuration System

Covers: `vite.config.ts`, conditional/async config, env vars in config.

### Beginner

```ts
// vite.config.ts
import { defineConfig } from 'vite'

export default defineConfig({
  // config options
})
```

`defineConfig` exists purely for TypeScript autocomplete/type-checking — it's an identity function at runtime. Vite auto-discovers `vite.config.ts` (or `.js`/`.mjs`) at the project root; ES module syntax works regardless of your `package.json`'s `"type"` field.
([Configuring Vite — Vite](https://vite.dev/config/))

### Working Knowledge

Export a function instead of an object to branch on **command** and **mode**:

```ts
export default defineConfig(({ command, mode, isSsrBuild }) => {
  if (command === 'serve') {
    return { /* dev-only config */ }
  }
  return { /* build config */ }
})
```

- `command`: `'serve'` while running `vite`/`vite dev`, `'build'` during `vite build`.
- `mode`: `'development'` or `'production'` by default, or a custom string via `vite build --mode staging`.

**Wrong vs. right — reading env vars in the config file itself:**

```ts
// Wrong — process.env.VITE_API_URL is likely undefined here: Vite loads
// .env files AFTER the config file resolves, so this races
export default defineConfig({
  define: { __API_URL__: JSON.stringify(process.env.VITE_API_URL) },
})

// Right — use loadEnv to read .env files explicitly, on your own schedule
import { defineConfig, loadEnv } from 'vite'

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '')
  return {
    define: { __API_URL__: JSON.stringify(env.VITE_API_URL) },
  }
})
```

This is a common real-world gotcha — `.env` loading and config resolution have different timing, and it only bites once you try to *use* an env var inside `vite.config.ts` rather than in application code.
([Env Variables and Modes — Vite](https://vite.dev/guide/env-and-mode))

### Advanced

Common production-relevant options:

```ts
export default defineConfig({
  resolve: { alias: { '@': '/src' } },
  server: {
    port: 3000,
    proxy: { '/api': { target: 'http://localhost:8080', changeOrigin: true } },
  },
  build: { target: 'esnext', outDir: 'dist' },
})
```

`server.proxy` is worth internalizing early: it lets your dev server forward `/api/*` requests to a real backend, sidestepping CORS during development without touching production routing at all — the proxy only exists in `vite dev`.
([Server Options — Vite](https://vite.dev/config/server-options))

[⬆ back to top](#table-of-contents)

---

## 4. Part 3 — Import & Runtime Features

Covers: glob imports, asset handling, `import.meta.env`, CSS modules.

### Beginner — asset & env basics

```ts
import imgUrl from './img.png'
// dev: '/src/img.png' — build: '/assets/img.2d8efhg.png' (hashed)

console.log(import.meta.env.MODE) // 'development' | 'production' | custom
console.log(import.meta.env.PROD) // boolean
```

Only env vars prefixed `VITE_` are exposed to client code — this is a deliberate security boundary, not a naming convention you can opt out of:

```bash
# .env
VITE_API_URL=https://api.example.com
DB_PASSWORD=secret   # NOT exposed — stays server/build-machine only
```

```ts
import.meta.env.VITE_API_URL // works
import.meta.env.DB_PASSWORD  // undefined, always
```

**Real Scenario.** A teammate stores a third-party API key as `VITE_STRIPE_SECRET_KEY` expecting the `VITE_` prefix to just be a convention, then wonders why it's visible in the browser's bundled JS. The prefix isn't a naming style — it's Vite's allowlist for what gets shipped to the client. Anything genuinely secret must never carry the `VITE_` prefix, full stop.
([Env Variables and Modes — Vite](https://vite.dev/guide/env-and-mode))

### Working Knowledge

**Glob imports** — pull in many modules matching a pattern without hand-writing an import per file:

```ts
const modules = import.meta.glob('./pages/*.ts')
// { './pages/home.ts': () => import('./pages/home.ts'), ... }  — lazy by default

const eager = import.meta.glob('./pages/*.ts', { eager: true }) // loaded immediately
```

**Explicit asset queries** — override Vite's default inline/URL heuristics:

```ts
import shaderCode from './shader.glsl?raw'      // raw string
import workletUrl from './worklet.js?url'        // explicit URL, never inlined
import inlined from './small.png?inline'          // force base64 inline
```

**Web Workers** — the constructor pattern is preferred over the `?worker` query for anything beyond the simplest case, since it's plain web-standard syntax Vite just knows how to bundle:

```ts
const worker = new Worker(new URL('./worker.ts', import.meta.url), { type: 'module' })
```
([Static Asset Handling — Vite](https://vite.dev/guide/assets), [Glob Import — Vite](https://vite.dev/guide/features#glob-import))

### Advanced

CSS Modules work automatically for any `*.module.css` file — no plugin required:

```ts
import styles from './component.module.css'
element.className = styles.button
```

JSON imports support named, tree-shaken exports:

```ts
import { version } from './package.json'
```

[⬆ back to top](#table-of-contents)

---

## 5. Part 4 — The Plugin Ecosystem

Covers: using official plugins, ordering, writing a small plugin, virtual modules.

### Beginner

Vite's plugin interface extends Rollup's (and, as of Vite 8, Rolldown's) plugin API with a few Vite-specific hooks. Most day-to-day plugin use is just registering an official plugin:

```ts
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()], // array is flattened; falsy entries ignored (useful for conditional plugins)
})
```

Common official plugins: `@vitejs/plugin-vue`, `@vitejs/plugin-react` (Oxc/Babel-based) or `@vitejs/plugin-react-swc`, `@vitejs/plugin-legacy` for older-browser output.

### Working Knowledge — ordering

Plugin execution order isn't just array order — `enforce` inserts a plugin relative to Vite's own core plugins:

```
Alias → enforce: 'pre' → Vite core plugins → user plugins (no enforce) →
build plugins → enforce: 'post' → post-build plugins
```

```ts
{ name: 'add-alias', enforce: 'pre' }   // runs before Vite's core resolution
{ name: 'strip-console', enforce: 'post' } // runs after everything else, e.g. minify-adjacent work
```

**Wrong vs. right — a plugin that should only run at build time:**

```ts
// Wrong — runs during dev too, and may crash the dev server trying to do
// build-only work (e.g. reading a rollupOptions.output it can't see yet)
plugins: [myBuildOnlyPlugin()]

// Right — gate it explicitly
plugins: [{ ...myBuildOnlyPlugin(), apply: 'build' }]
```
([Plugin API — Vite](https://vite.dev/guide/api-plugin))

### Advanced — writing your own

A minimal real plugin: inject a build-time constant via a Vite-specific hook (`config`) rather than juggling `define` in every environment:

```ts
function injectVersion(version: string) {
  return {
    name: 'inject-version',
    config: () => ({
      define: { __APP_VERSION__: JSON.stringify(version) },
    }),
  }
}
```

**Virtual modules** — serve generated content with no file on disk, useful for things like auto-generated route tables:

```ts
const virtualModuleId = 'virtual:my-module'
const resolvedId = '\0' + virtualModuleId // \0 prefix = "don't let other plugins touch this"

function myVirtualPlugin() {
  return {
    name: 'virtual-module',
    resolveId(id) {
      if (id === virtualModuleId) return resolvedId
    },
    load(id) {
      if (id === resolvedId) return `export const msg = "from virtual module"`
    },
  }
}
```

```ts
// usage in app code
import { msg } from 'virtual:my-module'
```
([Plugin API — Vite](https://vite.dev/guide/api-plugin))

### Mastery

`configureServer` can inject middleware before *or* after Vite's own — returning a function from the hook defers it to run after Vite's internal middlewares, which matters if your middleware needs to see a request Vite would otherwise have already handled (e.g. intercepting `index.html` transforms):

```ts
configureServer(server) {
  return () => {
    server.middlewares.use((req, res, next) => {
      // runs after Vite's own middleware stack
      next()
    })
  }
}
```

Full hook-by-hook plugin authoring (all Rolldown-inherited universal hooks, `handleHotUpdate`, client-server WS messaging) is intentionally out of scope for this doc — worth a dedicated follow-up if plugin authoring becomes a concrete goal.

[⬆ back to top](#table-of-contents)

---

## 6. Part 5 — Production Build (Rolldown)

Covers: `vite build`, `build.target`, library mode, multi-page apps.

### Beginner

```bash
vite build      # production build → outDir (default dist/)
vite preview     # serve the built output locally, to sanity-check it before deploying
```

As of Vite 8, the build step runs on **Rolldown** — previously (Vite ≤7) it ran on Rollup. For typical apps this is invisible: same `plugins` array, same output shape, just faster and — per the [Vite 8 announcement](https://vite.dev/blog/announcing-vite8) — 10-30x faster on large projects.

### Working Knowledge

```ts
export default defineConfig({
  build: {
    target: 'esnext', // default: Baseline Widely Available browsers; tighten/loosen as needed
    outDir: 'dist',
  },
})
```

**Real Scenario.** A team ships `target: 'esnext'` to get the smallest possible bundle, then gets a bug report from a user on an older Safari that doesn't support some newer syntax `esnext` allowed through. `build.target` is a real compatibility contract, not just an optimization knob — set it to match your actual supported browser matrix, and lean on Vite's default (Baseline Widely Available) unless you have a specific reason to diverge.
([Build Options — Vite](https://vite.dev/config/build-options))

### Advanced — library mode

Building a package for distribution (not an app) needs `build.lib`:

```ts
import { resolve } from 'node:path'

export default defineConfig({
  build: {
    lib: {
      entry: resolve(import.meta.dirname, 'lib/main.ts'),
      name: 'MyLib',
      fileName: 'my-lib',
    },
    rolldownOptions: {
      external: ['vue'], // don't bundle peer deps
      output: { globals: { vue: 'Vue' } },
    },
  },
})
```

Multi-page apps configure multiple HTML entries instead:

```ts
export default defineConfig({
  build: {
    rolldownOptions: {
      input: {
        main: resolve(import.meta.dirname, 'index.html'),
        nested: resolve(import.meta.dirname, 'nested/index.html'),
      },
    },
  },
})
```
([Building for Production — Vite](https://vite.dev/guide/build))

### Mastery

The **JavaScript API** lets you drive Vite programmatically — useful for custom tooling, test harnesses, or scripted multi-project builds:

```ts
import { build, createServer, preview } from 'vite'

await build({ root: './project', build: { outDir: 'dist' } })

const server = await createServer({ configFile: false, root: import.meta.dirname })
await server.listen()
```
([JavaScript API — Vite](https://vite.dev/guide/api-javascript))

[⬆ back to top](#table-of-contents)

---

## 7. Part 6 — SSR & the Environment API

Covers: what Vite's SSR support actually is, the Environment API (Vite 6+), when to reach for a meta-framework instead.

### Working Knowledge

Vite's own SSR support is deliberately **low-level** — primitives (`ssrLoadModule`, middleware mode) meant for *framework authors*, not something most app developers should wire up by hand. Per Vite's own guidance, if you need SSR, reach for a Vite-based meta-framework:

- **Nuxt** (Vue), **SvelteKit** (Svelte), **SolidStart** (Solid), **TanStack Start** (React)

Need just a server layer without a full frontend meta-framework? [Nitro](https://nitro.build) — "Vite for servers" — is the framework-agnostic option, and is what powers Nuxt's server engine underneath.
([Server-Side Rendering — Vite](https://vite.dev/guide/ssr))

**Wrong vs. right — building SSR by hand:**

```
Wrong: hand-roll ssrLoadModule + a custom render pipeline for a production app
       — you're re-implementing what Nuxt/SvelteKit/etc. already solved,
         including edge cases you won't discover until they break in prod

Right: use a meta-framework unless you are specifically building
       framework-author-level tooling
```

### Advanced — the Environment API (Vite 6+)

Before Vite 6, Vite implicitly had two environments: `client` and `ssr`. The Environment API formalizes this into an arbitrary set — useful for edge runtimes, workers, or multiple simultaneous targets:

```ts
export default defineConfig({
  build: { sourcemap: false },       // inherited by all environments
  optimizeDeps: { include: ['lib'] }, // client-only by default
  environments: {
    server: {},
    edge: { resolve: { noExternal: true } },
  },
})
```

As an **end user** you'll rarely touch this directly — frameworks (e.g. Cloudflare's Vite plugin, which runs dev code inside the actual `workerd` runtime) configure it for you. It's worth knowing it exists so a framework's config doesn't look like unexplained magic.
([Environment API — Vite](https://vite.dev/guide/api-environment))

[⬆ back to top](#table-of-contents)

---

## 8. Part 7 — Vite 8 Internals: Rolldown & Oxc

Covers: what actually changed under the hood in Vite 8, and how to migrate a Vite 7 config.

### Mastery

Vite 8's headline change: **Rolldown**, a unified Rust-based bundler, replaces the old two-tool split (esbuild for dev pre-bundling, Rollup for production builds). One bundler now drives both paths, which is *why* dev and build behave more consistently than they used to — fewer "works in dev, breaks in build" surprises caused by esbuild and Rollup disagreeing about edge-case module resolution.

| Before (Vite ≤7) | After (Vite 8) |
|---|---|
| esbuild (dev transform) | Oxc Transformer |
| esbuild (dep pre-bundling) | Rolldown |
| Rollup (production build) | Rolldown |
| `build.rollupOptions` | `build.rolldownOptions` |
| top-level `esbuild` option | top-level `oxc` option |

**Config migration — wrong vs. right (i.e. Vite 7 syntax vs. Vite 8):**

```ts
// Vite 7 (Rollup)
export default defineConfig({
  build: {
    rollupOptions: { external: ['vue'], output: { globals: { vue: 'Vue' } } },
  },
  esbuild: { jsxFactory: 'h', jsxFragment: 'Fragment' },
})

// Vite 8 (Rolldown + Oxc)
export default defineConfig({
  build: {
    rolldownOptions: { external: ['vue'], output: { globals: { vue: 'Vue' } } },
  },
  oxc: { jsx: { runtime: 'classic', pragma: 'h', pragmaFrag: 'Fragment' } },
})
```

Most existing Vite plugins work unchanged — Rolldown implements Rollup's plugin API, since that ecosystem compatibility was a hard design constraint, not an accident.

**Gradual migration path** for large Vite 7 projects not ready to jump straight to Vite 8:

```bash
# Step 1 — swap the import only, keep everything else on Vite 7 for now
pnpm add -D rolldown-vite
# then in vite.config.ts: import { defineConfig } from 'rolldown-vite'

# Step 2 — once stable on Rolldown, upgrade fully
pnpm add -D vite@8
```

**Real Scenario.** A tutorial from 2025 (pre-Vite-8) tells you to configure JSX via the top-level `esbuild` option. Following it verbatim on a fresh Vite 8 project silently no-ops — `esbuild` config is now `oxc` config, and Vite won't error, it'll just ignore options it doesn't recognize under the old key. This is exactly the kind of version drift this doc's "About This Document" section flags to check first when something from an older guide doesn't behave as documented.
([Vite 8.0 is out! — Vite](https://vite.dev/blog/announcing-vite8), [Vite 8 Beta announcement — Vite](https://vite.dev/blog/announcing-vite8-beta))

[⬆ back to top](#table-of-contents)

---

## 9. Cheat Sheets

### CLI

```bash
vite              # start dev server
vite build        # production build
vite preview      # preview production build locally
vite build --ssr  # SSR build
vite --force      # clear dep pre-bundle cache and restart
```

### Minimal config skeleton

```ts
import { defineConfig, loadEnv } from 'vite'

export default defineConfig(({ command, mode }) => {
  const env = loadEnv(mode, process.cwd(), '')
  return {
    plugins: [],
    resolve: { alias: { '@': '/src' } },
    server: { port: 3000, proxy: { '/api': 'http://localhost:8080' } },
    build: { target: 'esnext', outDir: 'dist' },
  }
})
```

### Import query suffixes

| Suffix | Effect |
|---|---|
| `?raw` | Import as a raw string |
| `?url` | Import as an explicit resolved URL, never inlined |
| `?inline` / `?no-inline` | Force base64 inline / force separate file |
| `?worker` | Import as a Web Worker constructor |

### Vite 7 → Vite 8 config renames

| Vite 7 | Vite 8 |
|---|---|
| `build.rollupOptions` | `build.rolldownOptions` |
| top-level `esbuild` | top-level `oxc` |

[⬆ back to top](#table-of-contents)

---

## 10. Suggested Learning Order

1. **Orientation** (§1) — scaffold a project, watch the Network tab, internalize "no bundle in dev."
2. **Dev Server & Module Graph** (§2) — the mental model everything else builds on.
3. **Configuration System** (§3) — you'll touch `vite.config.ts` constantly; get the `loadEnv` timing gotcha out of the way early.
4. **Import & Runtime Features** (§4) — glob imports and env var handling come up in almost every real project.
5. **Production Build** (§6) — `build.target`, then library mode only if/when you actually publish a package.
6. **Plugin Ecosystem** (§5) — start with *using* plugins well (ordering, `apply`); come back to authoring once you have a concrete need.
7. **SSR & Environment API** (§7) — skim once, so a meta-framework's config doesn't look like magic; go deeper only if you pick up Nuxt/SvelteKit/etc.
8. **Vite 8 Internals** (§8) — read fully if migrating an existing Vite 7 project; otherwise reference as needed.

[⬆ back to top](#table-of-contents)

---

## 11. Quick Self-Check

- Why does Vite's dev server stay fast regardless of app size, while a traditional bundler's dev server slows down as the app grows?
- What problem does dependency pre-bundling (`optimizeDeps`) solve, and what's the first thing to try when a linked/edited dependency seems "stuck" on old code?
- Why can't you reliably read a `VITE_`-prefixed env var from `process.env` directly inside `vite.config.ts`, and what should you use instead?
- What determines whether a plugin runs before or after Vite's own core plugins, and how do you restrict a plugin to build-only?
- What changed between Vite 7 and Vite 8 at the bundler/transformer level, and which two config keys did that rename?
- Why does Vite's own documentation steer app developers away from hand-rolling SSR, and what should you reach for instead?

[⬆ back to top](#table-of-contents)
