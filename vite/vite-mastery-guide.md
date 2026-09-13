# Vite — Mastery Guide (Overview)

The entry point for this repo's Vite material. This document holds the shared spec, the setup you need before running anything, the map of the focused sub-docs, the mental model everything else hangs off, the Vite 7 → 8 delta, and the reference apparatus (learning order, self-check, cross-doc cheat sheet).

The teaching lives in the sub-docs. Start here, then follow the [Suggested Learning Order](#suggested-learning-order).

---

<a id="about-this-document"></a>

## About This Document

This section is the spec for **every doc in this set**. The sub-docs each link back here instead of repeating it.

- **Framing: goal-driven.** The goal is **building JavaScript/TypeScript apps with Vite**. Depth, examples and priorities follow the files, commands, flags and config you will actually touch: `index.html`, `vite.config.ts`, `.env*`, `tsconfig.*.json`, `package.json` scripts, a Dockerfile, an nginx `location` block, a `vitest` setup file. Where something only matters in production, it is taught as a **Real Scenario**: a concrete incident, not an abstraction.
- **Confirmed scope: SPA-first, framework-agnostic, end to end.** Project anatomy → dev server & dependency pre-bundling → HMR → config → env vars & modes → assets/CSS/TypeScript → production build & chunking → deploying an SPA → library mode → plugins (using and writing) → monorepos/workspaces → testing with Vitest → SSR & the Environment API → internals & performance.
- **Examples are vanilla TypeScript by default.** React (`@vitejs/plugin-react`) and Vue (`@vitejs/plugin-vue`) appear **side by side only where the two plugins actually behave differently**: Fast Refresh vs. SFC HMR, `tsc -b` vs. `vue-tsc -b`, Testing Library setup, duplicate-framework bugs in monorepos.
- **Deliberately excluded** (with pointers): meta-frameworks built on Vite (Nuxt, React Router framework mode, SvelteKit, Astro, TanStack Start); Svelte/Solid/Qwik/Lit/Preact specifics; module federation; `@vitejs/plugin-legacy` for pre-2020 browsers; E2E testing (Playwright as a test runner); the React and Vue languages themselves. See [Deliberately Not Covered](#deliberately-not-covered).
- **Depth tiers.** Each Part runs **Beginner → Working Knowledge → Advanced → Mastery**, so you can stop at whatever level today's task needs. Tiers are collapsed where one would be filler, and every collapse is stated at the top of that Part. Collapsing is intended, not a gap to fill on a later pass. Most of the weight sits in Working Knowledge and Advanced. Mastery material is concentrated in [Plugins](./vite-plugins-monorepo.md#part-10-plugins) and [SSR & Internals](./vite-ssr-internals.md).
- **Vitest gets a full tiered Part** ([Part 12](./vite-testing-vitest.md#part-12-testing-with-vitest)), not just a note on sharing config. Vitest reuses your Vite config and plugin pipeline, so testing is where a Vite misconfiguration usually shows up first.
- **Assumed background: this repo's JavaScript, TypeScript, React and Vue sets.** You know ES modules, `async`/`await`, modern TypeScript, npm scripts, and at least one of React or Vue. This set does **not** re-teach those. See [`javascript/`](../javascript/javascript-mastery-guide.md), [`typescript/`](../typescript/typescript-mastery-guide.md), [`react/`](../react/react-mastery-guide.md), [`vue/`](../vue/vue-mastery-guide.md). No bundler experience is assumed. The one short primer ("why an unbundled dev server?") sits inside [the mental model](#the-mental-model), because every later Part depends on it.
- **Per-section ingredients**, used only where they help:
  - a short plain-language explanation of what a thing is and why it matters;
  - a **wrong vs. right** snippet pair wherever a gotcha has a common wrong form;
  - a **Real Scenario**: a production-style incident where the concept actually bites (this is a goal-driven set, so incidents take the place of "Try It" exercises);
  - an **ASCII diagram** where the concept is spatial or structural: the dev request pipeline, pre-bundling, HMR boundary propagation, the build pipeline, plugin hook order, the SSR request flow, a monorepo's resolution paths.
- **Version stance: Vite 8 only, with a 7 → 8 delta Part.** Every example is written for Vite 8.3. Vite 7 and earlier are covered **only** in [Part 15](#part-15-vite-7-to-8) as a migration and code-reading reference. The 7 → 8 change replaced both underlying bundlers, and nearly every pre-2026 tutorial still shows the old option names.
- **Sourcing rule.** [vite.dev](https://vite.dev/guide/) and [vitest.dev](https://vitest.dev/guide/) are the primary sources, cited inline at the claim they support. [rolldown.rs](https://rolldown.rs/in-depth/manual-code-splitting) and the plugin repos are used where Vite's docs defer to them. Stack Overflow threads are cited for real-world gotchas and how practitioners resolved them. Those threads were retrieved through the Stack Exchange API (the site blocks direct fetching). Each one is cited only for what its top answer says, or, where a question had no answer, only as evidence that the problem is reported. Where a popular-but-outdated convention disagrees with the official docs, the official docs win and the doc says so. **Several claims are verified by running them locally** rather than taken from docs; those are marked *"verified locally"*.
- **Formatting.** Clickable TOC at the top of every doc; explicit `<a id>` anchors on every Part and tier heading, so links never depend on auto-generated slugs; "Back to top" after each major section; a cheat-sheet table closing every Part; fenced code blocks with language tags.
- **Written against.** **Vite 8.3.0** (released Sep 10 2026), **Rolldown 1.2.8**, **Lightning CSS 1.33**, `create-vite` **9.2.1**, `@vitejs/plugin-react` **6.1.1**, `@vitejs/plugin-vue` **6.0.8**, **Vitest 5.0.0** (released Sep 3 2026), jsdom 30, `@testing-library/react` 16.3, TypeScript **6.0** (the version `create-vite` pins), **Node.js 24.21** locally (Vite needs 20.19+/22.12+; Vitest 5 needs 22.12+). Verified **September 13, 2026** by scaffolding `vanilla-ts`, `react-ts` and `vue-ts`, then running `dev`, `build`, `preview`, a library build and a Vitest run.
- **To update this doc later.** *Preserve*: the goal-driven framing, the global Part numbering with explicit tier subheadings and `<a id>` anchors, the wrong-vs-right pairs, the Real Scenarios, the "verified locally" markers, and the per-Part cheat sheets. *Re-verify against current docs first*:
  - the Vite minor version, and whatever [Vite's breaking-changes page](https://vite.dev/changes/) has moved from "planned" to "past";
  - whether **bundled dev mode** (`experimental.bundledDev`) has become stable or the default, which would rewrite [Part 2](./vite-dev-server.md#part-2-dev-server-and-module-graph)'s mental model;
  - whether the **Environment API** has left release-candidate status, which affects [Part 13](./vite-ssr-internals.md#part-13-ssr-and-the-environment-api);
  - whether `server.ssrLoadModule` has been removed;
  - whether `css.transformer: 'lightningcss'` has become the default (planned for the next major, per the [8.1 announcement](https://vite.dev/blog/announcing-vite8-1));
  - whether the deprecated `build.rollupOptions`/`esbuild` compatibility shims are gone ([Part 15](#part-15-vite-7-to-8));
  - Vitest's default for `clearMocks`, and its Node floor;
  - the TypeScript version `create-vite` pins.

---

## Table of Contents

1. [How This Set Is Organised](#how-this-set-is-organised)
2. [The Sub-Docs](#the-sub-docs)
3. [Orientation & Setup](#orientation-and-setup)
4. [The Mental Model](#the-mental-model)
5. [Part 15: Vite 7 → 8 (and what changed inside 8.x)](#part-15-vite-7-to-8)
6. [Deliberately Not Covered](#deliberately-not-covered)
7. [Suggested Learning Order](#suggested-learning-order)
8. [Quick Self-Check](#quick-self-check)
9. [Cross-Doc Cheat Sheet](#cross-doc-cheat-sheet)

---

<a id="how-this-set-is-organised"></a>

## 1. How This Set Is Organised

Vite is 14 Parts of teaching material spread over 6 sub-docs, plus this overview. The order follows when each concept becomes necessary as you take an app from `npm create` to production:

```text
  vite-mastery-guide.md    ← you are here: spec, setup, mental model, 7→8 delta, reference
        │
        ├── vite-dev-server.md          Parts 1–3    what happens when you run `vite`
        ├── vite-config-env-assets.md   Parts 4–6    how you tell it what to do
        ├── vite-build-deploy.md        Parts 7–9    what `vite build` produces and where it goes
        ├── vite-plugins-monorepo.md    Parts 10–11  extending it, and scaling it to many packages
        ├── vite-testing-vitest.md      Part 12      testing on the same pipeline
        └── vite-ssr-internals.md       Parts 13–14  rendering on a server, and how it all works
```

Every Part is numbered globally (1–14, plus Part 15 in this document), so "see Part 7" means the same thing in every file.

[↑ Back to top](#table-of-contents)

---

<a id="the-sub-docs"></a>

## 2. The Sub-Docs

| Doc | Parts | What it answers |
|---|---|---|
| [Dev Server & HMR](./vite-dev-server.md) | 1–3 | What is in a Vite project, what happens when the browser asks for a file, and why did my edit reload the whole page? |
| [Config, Env & Assets](./vite-config-env-assets.md) | 4–6 | How do I configure it per mode, get env vars in without leaking secrets, and import CSS, images, workers and TypeScript correctly? |
| [Build & Deploy](./vite-build-deploy.md) | 7–9 | What does the production bundle look like, why does my deployed SPA 404 on refresh, and how do I publish a package? |
| [Plugins & Monorepos](./vite-plugins-monorepo.md) | 10–11 | How do plugins order and run, how do I write one, and why is my workspace package stale or duplicated? |
| [Testing with Vitest](./vite-testing-vitest.md) | 12 | How do I test on the same config, pick an environment, mock modules, and measure coverage? |
| [SSR & Internals](./vite-ssr-internals.md) | 13–14 | What do SSR and the Environment API actually give me, and how do I find out why the dev server is slow? |

**Part index**

1. Project Anatomy — [Dev Server & HMR](./vite-dev-server.md#part-1-project-anatomy)
2. The Dev Server & Module Graph — [Dev Server & HMR](./vite-dev-server.md#part-2-dev-server-and-module-graph)
3. Hot Module Replacement — [Dev Server & HMR](./vite-dev-server.md#part-3-hot-module-replacement)
4. The Config System — [Config, Env & Assets](./vite-config-env-assets.md#part-4-the-config-system)
5. Env Variables & Modes — [Config, Env & Assets](./vite-config-env-assets.md#part-5-env-variables-and-modes)
6. Assets, CSS & TypeScript — [Config, Env & Assets](./vite-config-env-assets.md#part-6-assets-css-and-typescript)
7. The Production Build — [Build & Deploy](./vite-build-deploy.md#part-7-the-production-build)
8. Deploying an SPA — [Build & Deploy](./vite-build-deploy.md#part-8-deploying-an-spa)
9. Library Mode — [Build & Deploy](./vite-build-deploy.md#part-9-library-mode)
10. Plugins — [Plugins & Monorepos](./vite-plugins-monorepo.md#part-10-plugins)
11. Monorepos & Workspaces — [Plugins & Monorepos](./vite-plugins-monorepo.md#part-11-monorepos-and-workspaces)
12. Testing with Vitest — [Testing with Vitest](./vite-testing-vitest.md#part-12-testing-with-vitest)
13. SSR & the Environment API — [SSR & Internals](./vite-ssr-internals.md#part-13-ssr-and-the-environment-api)
14. Internals & Performance — [SSR & Internals](./vite-ssr-internals.md#part-14-internals-and-performance)
15. Vite 7 → 8 — [this document](#part-15-vite-7-to-8)

[↑ Back to top](#table-of-contents)

---

<a id="orientation-and-setup"></a>

## 3. Orientation & Setup

Every example in this set assumes a project created this way. Create one and keep it open while you read.

```bash
npm create vite@latest my-app -- --template react-ts   # or vue-ts, or vanilla-ts
cd my-app
npm install
npm run dev
```

The `--` is needed with npm 7+ so that `--template` reaches `create-vite` instead of being swallowed by npm ([Getting Started](https://vite.dev/guide/)). Inside a script or CI job, add `--no-interactive` so it never waits for a prompt. `--immediate` installs dependencies and starts the dev server in one go (*verified locally* against `create-vite --help`, 9.2.1).

**Requirements** ([Getting Started](https://vite.dev/guide/), [Vitest Getting Started](https://vitest.dev/guide/)):

| Requirement | Minimum |
|---|---|
| Node.js for Vite 8 | **20.19+ or 22.12+** (required for ESM-only Vite without flags) |
| Node.js for Vitest 5 | **22.12+**, so in practice your whole toolchain needs 22.12+ once you add tests |
| Production browser target (default) | Chrome/Edge 111+, Firefox 114+, Safari 16.4+ ("Baseline Widely Available") |

**Templates.** `vanilla`, `vue`, `react`, `react-compiler`, `preact`, `lit`, `svelte`, `solid`, `qwik`, each with a `-ts` variant ([Getting Started](https://vite.dev/guide/)).

**What the three templates in this set actually give you** (*verified locally*, `create-vite` 9.2.1):

| | `vanilla-ts` | `react-ts` | `vue-ts` |
|---|---|---|---|
| `vite.config.ts` | **none** (zero-config) | `plugins: [react()]` | `plugins: [vue()]` |
| `build` script | `tsc && vite build` | `tsc -b && vite build` | `vue-tsc -b && vite build` |
| tsconfig layout | single `tsconfig.json` | solution file → `tsconfig.app.json` + `tsconfig.node.json` | same, `app` extends `@vue/tsconfig` |
| Client types | `"types": ["vite/client"]` in tsconfig | same | same |
| Linter | — | **Oxlint** (`.oxlintrc.json`); `--eslint` opts back into ESLint | — |
| TypeScript | `~6.0.2` | `~6.0.2` | `~6.0.2` |

> **Outdated convention alert.** Tutorials up to 2025 show a `src/vite-env.d.ts` containing `/// <reference types="vite/client" />`, and a React template linted with ESLint. Current templates put `vite/client` in `compilerOptions.types` and ship no `vite-env.d.ts`. You only create that file when you want to type your own env vars ([Part 5](./vite-config-env-assets.md#part-5-env-variables-and-modes)). Both styles work, so don't "fix" either one.

The scripts:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview"
  }
}
```

`vite`, `vite dev` and `vite serve` are the same command ([Getting Started](https://vite.dev/guide/)). `tsc -b` is in `build` because **Vite never type-checks** ([Part 6](./vite-config-env-assets.md#part-6-assets-css-and-typescript)).

[↑ Back to top](#table-of-contents)

---

<a id="the-mental-model"></a>

## 4. The Mental Model

Vite is two tools that share one config and one plugin pipeline. **In dev, it doesn't bundle your code. In build, it bundles everything.** Almost every "works in dev, breaks in prod" bug comes from one of the places where those two paths differ.

**Why skip bundling in dev?** Browsers have supported `<script type="module">` and native `import` for years. A dev server can serve each source file as a module, transformed on request, and let the browser walk the import graph itself. Startup then costs roughly the same whatever the app's size, because files that are never imported are never transformed. A bundler-based dev server has to build the whole graph before it serves the first page. Your **dependencies** still get bundled even in dev, because a package like `lodash-es` is hundreds of tiny modules and some packages are still CommonJS ([Dependency Pre-Bundling](https://vite.dev/guide/dep-pre-bundling)).

```text
  ───────────────────────── vite (dev) ─────────────────────────
   browser ──GET /──────────────► index.html  (+ /@vite/client injected)
   browser ──GET /src/main.ts──► plugin pipeline: resolveId → load → transform
                                  (Oxc strips types / compiles JSX, per file, on demand)
   browser ──GET /node_modules/.vite/deps/react.js?v=hash
                                  ▲ pre-bundled ONCE by Rolldown at startup,
                                  │ cached on disk, served `immutable`
   browser ◄══ WebSocket ═══════ file watcher → HMR updates  (Part 3)

  ───────────────────────── vite build ─────────────────────────
   index.html ─► same plugin pipeline ─► Rolldown bundle (tree-shake, split, hash)
             ─► Oxc minify JS · Lightning CSS minify CSS
             ─► dist/index.html + dist/assets/*-[hash].{js,css,png}
```

Four consequences come up again and again:

1. **The browser is part of your dev toolchain.** The network tab shows one request per source module. That is normal, not a bug ([Part 2](./vite-dev-server.md#part-2-dev-server-and-module-graph)).
2. **`index.html` is source code.** It is the entry point of the module graph, not a static file in `public/` ([Part 1](./vite-dev-server.md#part-1-project-anatomy)).
3. **Everything client-visible is decided at build time.** `import.meta.env.VITE_*` is text-replaced into the bundle, so a Docker image built once cannot read a different API URL per environment unless you design for it ([Part 5](./vite-config-env-assets.md#part-5-env-variables-and-modes), [Part 8](./vite-build-deploy.md#part-8-deploying-an-spa)).
4. **Vite 8 made the two paths far more alike.** Before 8, dev used esbuild and build used Rollup, two bundlers with different CommonJS interop and resolution quirks. Vite 8 uses **Rolldown** for both pre-bundling and build, and **Oxc** for transforms and minification ([Vite 8 announcement](https://vite.dev/blog/announcing-vite8)). The unbundled-dev vs. bundled-build split still exists, and [Part 14](./vite-ssr-internals.md#part-14-internals-and-performance) covers the experimental bundled dev mode that removes it.

[↑ Back to top](#table-of-contents)

---

<a id="part-15-vite-7-to-8"></a>

## 5. Part 15: Vite 7 → 8 (and what changed inside 8.x)

This Part exists so you can make sense of older material. Vite 8 was released on **March 12, 2026** ([announcement](https://vite.dev/blog/announcing-vite8)), and this set targets 8.3. Most tutorials, Stack Overflow answers and AI-generated configs you will run into were written for Vite 5–7.

*Tiers collapsed:* this Part is reference material with no progression. It is one table plus the traps.

<a id="part-15-renames"></a>

### 15.1 Config you'll see in old code → what to write now

| Vite ≤ 7 (what you'll see) | Vite 8 (what to write) | Status | Source |
|---|---|---|---|
| `build.rollupOptions` | `build.rolldownOptions` | deprecated alias, still works | [Migration](https://vite.dev/guide/migration) |
| `worker.rollupOptions` | `worker.rolldownOptions` | deprecated alias | [Migration](https://vite.dev/guide/migration) |
| top-level `esbuild: { … }` | top-level `oxc: { … }` | deprecated, auto-converted **partially** (see 15.2) | [Migration](https://vite.dev/guide/migration) |
| `optimizeDeps.esbuildOptions` | `optimizeDeps.rolldownOptions` | deprecated, warns | [Migration](https://vite.dev/guide/migration) |
| `esbuild: { jsxFactory: 'h', jsxFragment: 'Fragment' }` | `oxc: { jsx: { pragma: 'h', pragmaFrag: 'Fragment' } }` | converted | [Migration](https://vite.dev/guide/migration) |
| `esbuild: { jsxInject }` | `oxc: { jsxInject }` | converted | [Migration](https://vite.dev/guide/migration) |
| `esbuild: { drop: ['console'] }` | `build.rolldownOptions.output.minify: { compress: { dropConsole: true } }` | **silently ignored**, see 15.2 | *verified locally* |
| `output.manualChunks: { vendor: [...] }` (object) | `output.codeSplitting: { groups: [...] }` | **removed, build fails** | [Migration](https://vite.dev/guide/migration), *verified locally* |
| `output.manualChunks(id) { … }` (function) | `output.codeSplitting` | deprecated, still works | [Migration](https://vite.dev/guide/migration) |
| `output.advancedChunks` (rolldown-vite era) | `output.codeSplitting` | deprecated in Rolldown 1.x | *Rolldown types, verified locally* |
| `build.minify: 'esbuild'` (the default) | default is `'oxc'`; `'esbuild'` needs `esbuild` installed | default changed | [Build Options](https://vite.dev/config/build-options) |
| CSS minified by esbuild | Lightning CSS (`build.cssMinify: 'lightningcss'`) | default changed | [Build Options](https://vite.dev/config/build-options) |
| `build.commonjsOptions` | no-op | removed | [Migration](https://vite.dev/guide/migration) |
| `resolve.alias[].customResolver` | a plugin with `resolveId` | removed | [Migration](https://vite.dev/guide/migration) |
| `import.meta.hot.accept('/some/url', cb)` | pass a module path, not a URL | removed | [Migration](https://vite.dev/guide/migration) |
| `transformWithEsbuild()` in plugins | `transformWithOxc()` | deprecated | [Migration](https://vite.dev/guide/migration) |
| `vite-tsconfig-paths` plugin | `resolve.tsconfigPaths: true` (built in, off by default) | plugin no longer needed | [Vite 8 announcement](https://vite.dev/blog/announcing-vite8) |
| `server.hmr.{port,host,clientPort,path,protocol}` | `server.ws.{…}` (`server.hmr` keeps `overlay`) | deprecated in **8.2** | [Changelog](https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md), *types verified locally* |
| `rolldown-vite` package (the Vite 7 preview) | plain `vite@8` | obsolete | [Vite 8 announcement](https://vite.dev/blog/announcing-vite8) |
| `server.ssrLoadModule(url)` | `server.environments.ssr.runner.import(url)` | still present; ModuleRunner migration "planned" | [Env API for frameworks](https://vite.dev/guide/api-environment-frameworks), [Breaking changes](https://vite.dev/changes/) |

<a id="part-15-silent"></a>

### 15.2 The changes that don't error — they just behave differently

These cost the most time because nothing warns you about them.

**`esbuild` options can be dropped with no warning.** Vite 8 converts a known list of `esbuild` keys to `oxc` (JSX options, `define`, `include`/`exclude`, `jsxInject`). `drop` is not on that list ([Migration](https://vite.dev/guide/migration)). *Verified locally* on 8.3.0:

- a `vanilla-ts` app with `esbuild: { drop: ['console'] }` built cleanly with **no warning** and still contained its `console.log`;
- a `react-ts` app with the same option printed *"Both esbuild and oxc options were set. oxc options will be used and esbuild options will be ignored"*, because `@vitejs/plugin-react` sets `oxc` options itself, and it also kept the `console.log`.

```ts
// Wrong: a Vite 7 config copied forward. No error, and console.* still ships.
export default defineConfig({
  esbuild: { drop: ['console', 'debugger'] },
})

// Right: Vite 8 minifies with Oxc through Rolldown.
export default defineConfig({
  build: {
    rolldownOptions: {
      output: { minify: { compress: { dropConsole: true, dropDebugger: true } } },
    },
  },
})
```

**CommonJS `default` imports follow one rule in dev and build.** In Vite 8, `import x from 'cjs-pkg'` gives you `module.exports` unless the module sets `__esModule: true` and the importer isn't `.mjs`/`.mts` or `type: module` ([Migration](https://vite.dev/guide/migration)). Code that worked around the old dev/build mismatch with `x.default ?? x` can now take the wrong branch. `legacy.inconsistentCjsInterop: true` brings back the old behaviour temporarily.

**Format sniffing is gone.** When a package has both `browser` and `module` fields, Vite no longer guesses from the file contents. It follows `resolve.mainFields` order ([Migration](https://vite.dev/guide/migration)).

**`require()` of an externalized module stays a `require()`** instead of being turned into an `import` ([Migration](https://vite.dev/guide/migration)). This matters for SSR and library builds.

**The default browser target moved up** from Chrome 107 / Firefox 104 / Safari 16.0 to **Chrome 111 / Firefox 114 / Safari 16.4** ([Migration](https://vite.dev/guide/migration)). If your support matrix includes Safari 16.0–16.3, set `build.target` explicitly.

**Oxc can't lower native decorators.** If you use TC39 decorators and target browsers without them, you need a Babel or SWC plugin ([Migration](https://vite.dev/guide/migration)). TypeScript's `experimentalDecorators` + `emitDecoratorMetadata` *is* supported natively in 8 ([Vite 8 announcement](https://vite.dev/blog/announcing-vite8)).

**Output formats.** `system` and `amd` output are no longer supported, and `@vitejs/plugin-legacy` can't produce ES5 ([Migration](https://vite.dev/guide/migration)).

<a id="part-15-minors"></a>

### 15.3 What arrived inside 8.x

| Version (date) | What matters for app developers | Source |
|---|---|---|
| **8.0** (Mar 2026) | Rolldown + Oxc; `resolve.tsconfigPaths`; `server.forwardConsole`; `devtools` option; decorator metadata; Vite ~15 MB larger on install (Lightning CSS is now a required dependency) | [Announcement](https://vite.dev/blog/announcing-vite8) |
| **8.1** (Jun 2026) | experimental **bundled dev mode** (`--experimental-bundle` / `experimental.bundledDev`); experimental `build.chunkImportMap`; direct `import { add } from './add.wasm'`; `import.meta.glob(…, { caseSensitive: false })`; `html.additionalAssetSources` | [8.1 announcement](https://vite.dev/blog/announcing-vite8-1) |
| **8.2** (Jul 2026) | top-level **`input`** option; `server.hmr.*` connection options → `server.ws.*`; worker HMR; **warnings when `vite.config.ts` uses features the future native config loader won't support** (see [Part 4](./vite-config-env-assets.md#part-4-mastery)) | [Changelog](https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md), *verified locally* |
| **8.3** (Sep 2026) | top-level **`tsconfig`** option (path to the tsconfig Vite should read); `closeServer`/`closePreviewServer` plugin hooks; `--profile [name]`; the native-loader warning also flags named imports from JSON in config files; subpath imports (`#foo`) in dynamic `import()` | [Changelog](https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md), *verified locally* |

<a id="part-15-upgrade"></a>

### 15.4 Upgrading an existing Vite 7 app

1. **Bump the framework plugin at the same time.** `@vitejs/plugin-react` v6 moved React Refresh from Babel to Oxc and dropped Babel as a dependency. v5 still works on Vite 8 ([Vite 8 announcement](https://vite.dev/blog/announcing-vite8)). If you relied on the old `babel` option (for React Compiler, say), see [Part 10](./vite-plugins-monorepo.md#part-10-plugins).
2. `npm i -D vite@8`, run `vite build`, and **read every warning**. The deprecation shims print once per key.
3. Grep for `esbuild`, `rollupOptions`, `manualChunks` and `server.hmr` and rename them by the table in 15.1. Don't rely on the shims: they will be removed.
4. Diff the output of `vite build` before and after (chunk names and sizes). Chunking heuristics changed along with the bundler.
5. For a very large app that is risky to move in one step, the team's documented path was to switch to `rolldown-vite` on Vite 7 first ([Vite 8 announcement](https://vite.dev/blog/announcing-vite8)). That package is now a historical artefact; don't start a new migration on it.

[↑ Back to top](#table-of-contents)

---

<a id="deliberately-not-covered"></a>

## 6. Deliberately Not Covered (and where to look instead)

Each of these would either double the size of the set or come out half-covered, which is worse than leaving it out.

| Topic | Why it's out | Where to look |
|---|---|---|
| **Meta-frameworks** (Nuxt, React Router framework mode, SvelteKit, Astro, TanStack Start) | Each one owns routing, SSR and deployment on top of Vite. Part 13 teaches the primitives they're built on | Each framework's own docs; Vite's [SSR guide](https://vite.dev/guide/ssr) links example projects |
| **Next.js** | Not Vite-based (Turbopack/webpack) | [`nextjs/`](../nextjs/nextjs-mastery-guide.md) |
| **React and Vue themselves** | Assumed | [`react/`](../react/react-mastery-guide.md), [`vue/`](../vue/vue-mastery-guide.md) |
| **TypeScript the language / `tsconfig` in depth** | Assumed; Part 6 covers only what Vite needs from tsconfig | [`typescript/`](../typescript/typescript-mastery-guide.md) |
| **CSS and Tailwind** | Assumed; Part 6 covers only Vite's CSS pipeline | [`css/`](../css/css-mastery-guide.md), [`tailwindcss/`](../tailwindcss/tailwindcss-mastery-guide.md) (uses `@tailwindcss/vite`) |
| **Svelte, Solid, Qwik, Lit, Preact** | SPA-first scope uses React/Vue for framework-specific examples | [Getting Started — templates](https://vite.dev/guide/) |
| **`@vitejs/plugin-legacy`** | Only needed for browsers older than the Baseline target; can't emit ES5 in Vite 8 | [Migration](https://vite.dev/guide/migration) |
| **Module federation** | Niche, and plugin-specific | `reactRefreshHost` in the [plugin-react README](https://github.com/vitejs/vite-plugin-react/tree/main/packages/plugin-react) |
| **E2E testing** (Playwright/Cypress as the runner) | Different tool; Part 12 covers Vitest **Browser Mode** for component tests | Playwright / Cypress docs |
| **Backend integration** (Rails/Laravel serving HTML) | Only matters if a non-Node backend owns your HTML; Part 7 covers the `manifest` it relies on | [Backend Integration](https://vite.dev/guide/backend-integration) |
| **Writing a Rolldown plugin for non-Vite use** | Part 10 covers Vite plugins, which are a superset | [Plugin API](https://vite.dev/guide/api-plugin) |
| **Bun as the runtime** | Vite runs under Bun, but Bun auto-loads `.env` before scripts and can interfere with Vite's env handling ([Env and Mode](https://vite.dev/guide/env-and-mode)) | [`bun/`](../bun/bun-mastery-guide.md) |

[↑ Back to top](#table-of-contents)

---

<a id="suggested-learning-order"></a>

## 7. Suggested Learning Order

**If you're building an app now** (the goal-driven path, roughly in the order you'll need each thing):

1. [Part 1](./vite-dev-server.md#part-1-project-anatomy): know every file the template gave you. 20 minutes.
2. [Part 4](./vite-config-env-assets.md#part-4-the-config-system) through Working Knowledge: aliases, proxy, conditional config.
3. [Part 5](./vite-config-env-assets.md#part-5-env-variables-and-modes): **do not skim.** The `VITE_` prefix is a publishing decision, not a naming style.
4. [Part 6](./vite-config-env-assets.md#part-6-assets-css-and-typescript): assets, CSS Modules, and why `vite build` passes type errors straight through.
5. [Part 2](./vite-dev-server.md#part-2-dev-server-and-module-graph) and [Part 3](./vite-dev-server.md#part-3-hot-module-replacement): read these the first time the dev server confuses you. It will.
6. [Part 12](./vite-testing-vitest.md#part-12-testing-with-vitest) through Working Knowledge: set up tests before there are many files to test.
7. [Part 7](./vite-build-deploy.md#part-7-the-production-build) and [Part 8](./vite-build-deploy.md#part-8-deploying-an-spa): **before the first deploy, not after.**
8. [Part 10](./vite-plugins-monorepo.md#part-10-plugins) Beginner–Working: using plugins well. Come back for authoring when you have a concrete need.
9. [Part 11](./vite-plugins-monorepo.md#part-11-monorepos-and-workspaces), [Part 9](./vite-build-deploy.md#part-9-library-mode), [Part 13](./vite-ssr-internals.md#part-13-ssr-and-the-environment-api): only when you have a workspace, a package to publish, or a reason to render on a server.
10. [Part 14](./vite-ssr-internals.md#part-14-internals-and-performance): when the dev server feels slow, or you're curious what bundled dev mode changes.

**If you're inheriting an existing Vite codebase**: [Part 15](#part-15-vite-7-to-8) → [Part 4](./vite-config-env-assets.md#part-4-the-config-system) → [Part 5](./vite-config-env-assets.md#part-5-env-variables-and-modes) → [Part 10](./vite-plugins-monorepo.md#part-10-plugins). Together these tell you which Vite era the config was written for, what each key does, what the bundle exposes, and what the plugins are doing to your code.

**If you're debugging production**: [Part 8](./vite-build-deploy.md#part-8-deploying-an-spa) → [Part 7](./vite-build-deploy.md#part-7-the-production-build) → the [cross-doc cheat sheet](#cross-doc-cheat-sheet).

[↑ Back to top](#table-of-contents)

---

<a id="quick-self-check"></a>

## 8. Quick Self-Check

If you can answer these without looking, you know the material. Each one links to where the answer is.

1. The network tab shows 300 requests on first load in dev. Bug or design? What would change the number? ([Part 2](./vite-dev-server.md#part-2-dev-server-and-module-graph))
2. You edited a file inside a locally linked package and the browser still runs the old code. What is cached, where, and how do you clear *only* that? ([Part 2](./vite-dev-server.md#part-2-dev-server-and-module-graph), [Part 11](./vite-plugins-monorepo.md#part-11-monorepos-and-workspaces))
3. Why does editing `utils.ts` reload the whole page while editing `Button.tsx` doesn't? What does "HMR boundary" mean? ([Part 3](./vite-dev-server.md#part-3-hot-module-replacement))
4. Why does exporting a constant next to a component break React Fast Refresh for that file? ([Part 3](./vite-dev-server.md#part-3-hot-module-replacement))
5. `process.env.VITE_API_URL` is `undefined` inside `vite.config.ts`. Why, and what do you use instead? ([Part 4](./vite-config-env-assets.md#part-4-the-config-system))
6. What exactly ends up in the bundle if a component does `console.log(import.meta.env)`? ([Part 5](./vite-config-env-assets.md#part-5-env-variables-and-modes))
7. `vite build --mode staging`: what is `NODE_ENV`, what is `import.meta.env.PROD`, and which `.env` files load? ([Part 5](./vite-config-env-assets.md#part-5-env-variables-and-modes))
8. `vite build` succeeded but the code has type errors. How? ([Part 6](./vite-config-env-assets.md#part-6-assets-css-and-typescript))
9. When does an imported image become a data URI instead of a file, and why might your CSP care? ([Part 6](./vite-config-env-assets.md#part-6-assets-css-and-typescript))
10. After a deploy, some users get "Failed to fetch dynamically imported module". Explain the sequence, and give two mitigations. ([Part 7](./vite-build-deploy.md#part-7-the-production-build))
11. Your app works under `vite preview` but `/users/42` 404s on refresh behind nginx. Why does preview hide the problem? ([Part 8](./vite-build-deploy.md#part-8-deploying-an-spa))
12. You promote one Docker image from staging to production and it still calls the staging API. Why, and what are your options? ([Part 8](./vite-build-deploy.md#part-8-deploying-an-spa))
13. Your published component library ships its CSS, but consumers get unstyled buttons. Why? ([Part 9](./vite-build-deploy.md#part-9-library-mode))
14. What are `enforce` and `apply` for, and why must a virtual module's resolved ID start with `\0`? ([Part 10](./vite-plugins-monorepo.md#part-10-plugins))
15. "Invalid hook call" appears only when the app consumes a workspace package. What is duplicated, and what are the two fixes? ([Part 11](./vite-plugins-monorepo.md#part-11-monorepos-and-workspaces))
16. You created `vitest.config.ts` and your path aliases stopped resolving in tests. Why? ([Part 12](./vite-testing-vitest.md#part-12-testing-with-vitest))
17. Why does Vitest 5 throw on a `vi.mock` inside a `test()` callback, when Vitest 4 only warned? ([Part 12](./vite-testing-vitest.md#part-12-testing-with-vitest))
18. What does `server.environments.ssr.runner.import()` give you that `ssrLoadModule` didn't? ([Part 13](./vite-ssr-internals.md#part-13-ssr-and-the-environment-api))
19. Why is an `index.ts` that re-exports 200 components slow in dev but fine in production? ([Part 14](./vite-ssr-internals.md#part-14-internals-and-performance))

[↑ Back to top](#table-of-contents)

---

<a id="cross-doc-cheat-sheet"></a>

## 9. Cross-Doc Cheat Sheet

### CLI

```bash
vite                          # dev server (aliases: vite dev, vite serve)
vite --host                   # listen on all interfaces (containers, phones on LAN)
vite --force                  # re-run dependency pre-bundling
vite --open                   # open browser (also warms up the entry)
vite --debug hmr              # why did HMR full-reload?
vite --profile                # press p+enter, open .cpuprofile in speedscope
vite build                    # production build → dist/
vite build --mode staging     # load .env.staging; NODE_ENV stays "production"
vite build --watch            # rebuild on change
vite build --ssr src/entry-server.ts   # SSR bundle
vite preview                  # serve dist/ locally — NOT a production server
vite --experimental-bundle    # 8.1+ experimental bundled dev mode
vitest                        # tests in watch mode
vitest run --coverage         # CI
```

### A realistic `vite.config.ts`

```ts
/// <reference types="vitest/config" />
import react from '@vitejs/plugin-react'
import { defineConfig, loadEnv } from 'vite'

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '')          // Part 4 — .env is NOT in process.env here
  return {
    plugins: [react()],                                  // Part 10
    resolve: { tsconfigPaths: true },                    // Part 4 — replaces vite-tsconfig-paths
    server: {
      port: 5173,
      proxy: { '/api': { target: env.API_ORIGIN ?? 'http://localhost:8080', changeOrigin: true } },
    },
    build: {
      sourcemap: 'hidden',                               // Part 7 — upload, don't serve
      rolldownOptions: {                                 // NOT rollupOptions (Part 15)
        output: {
          codeSplitting: { groups: [{ name: 'vendor', test: /node_modules/ }] },
          minify: { compress: { dropConsole: mode === 'production' } },   // replaces esbuild.drop
        },
      },
    },
    test: { environment: 'jsdom', setupFiles: ['./src/test/setup.ts'] },   // Part 12
  }
})
```

### Imports Vite understands

| Import | You get | Part |
|---|---|---|
| `import url from './a.png'` | hashed URL (or data URI under 4 KiB) | 6 |
| `import url from './a.js?url'` | URL, never inlined | 6 |
| `import text from './a.glsl?raw'` | string contents | 6 |
| `import css from './a.css?inline'` | CSS string, not injected | 6 |
| `import s from './a.module.css'` | class-name map | 6 |
| `import W from './w.ts?worker'` | Worker constructor | 6 |
| `new Worker(new URL('./w.ts', import.meta.url), { type: 'module' })` | the recommended worker form | 6 |
| `import.meta.glob('./pages/*.ts')` | `{ path: () => import(path) }` | 6 |
| `import { add } from './add.wasm'` | instantiated exports (8.1+) | 6 |
| `import { msg } from 'virtual:my-module'` | plugin-generated code | 10 |

### Where things live

| Path | What |
|---|---|
| `node_modules/.vite/deps/` | pre-bundled dependencies (delete = same as `--force`) |
| `node_modules/.vite-temp/` | bundled config file (default config loader) |
| `dist/.vite/manifest.json` | source → hashed-file map (`build.manifest: true`) |
| `dist/.vite/ssr-manifest.json` | module → chunk map for SSR preload tags |
| `dist/.vite/license.md` | `build.license: true` |
| `.vitest/` | Vitest 5 reports, attachments, screenshots |

### Env constants

| `import.meta.env.` | `vite` | `vite build` | `vite build --mode staging` | `vitest` |
|---|---|---|---|---|
| `MODE` | `development` | `production` | `staging` | `test` |
| `DEV` | `true` | `false` | `false` | `true` |
| `PROD` | `false` | `true` | `true` | `false` |
| `SSR` | `false` | `false` | `false` | `false` under `jsdom`, **`true` under `node`** |
| `BASE_URL` | `base` | `base` | `base` | `base` |

The `staging` and `vitest` columns were *verified locally* (Vitest also sets `NODE_ENV=test`). The rest follows from [Env and Mode](https://vite.dev/guide/env-and-mode).

[↑ Back to top](#table-of-contents)
