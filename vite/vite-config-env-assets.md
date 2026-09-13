# Vite — Config, Env & Assets (Parts 4–6)

How to tell Vite what to do (`vite.config.ts`), how values get from `.env` into your bundle and why that's a publishing decision, and how imports of CSS, images, JSON, workers and TypeScript are actually handled.

> **Spec:** this doc follows the shared spec in [`vite-mastery-guide.md`](./vite-mastery-guide.md#about-this-document). Written against **Vite 8.3.0**, verified **September 13, 2026**.

If you read one thing carefully here, make it [§5.2](#part-5-working). Every `VITE_` variable is published to anyone who can open your site.

---

## Table of Contents

- [Part 4: The Config System](#part-4-the-config-system)
  - [4.1 Beginner — `defineConfig` and zero-config](#part-4-beginner)
  - [4.2 Working Knowledge — conditional config, aliases, `loadEnv`](#part-4-working)
  - [4.3 Advanced — `define`, `base`, `root`, and config for several consumers](#part-4-advanced)
  - [4.4 Mastery — how the config file is loaded](#part-4-mastery)
  - [4.5 Cheat sheet: Part 4](#part-4-cheat-sheet)
- [Part 5: Env Variables & Modes](#part-5-env-variables-and-modes)
  - [5.1 Beginner — `import.meta.env`](#part-5-beginner)
  - [5.2 Working Knowledge — `.env` files, `VITE_`, and what gets published](#part-5-working)
  - [5.3 Advanced — modes vs. `NODE_ENV`, typing, HTML replacement](#part-5-advanced)
  - [5.4 Mastery — build-time values in a build-once, deploy-many world](#part-5-mastery)
  - [5.5 Cheat sheet: Part 5](#part-5-cheat-sheet)
- [Part 6: Assets, CSS & TypeScript](#part-6-assets-css-and-typescript)
  - [6.1 Beginner — importing assets and CSS](#part-6-beginner)
  - [6.2 Working Knowledge — query suffixes, CSS Modules, glob imports, JSON](#part-6-working)
  - [6.3 Advanced — TypeScript: transpile-only, and what that costs you](#part-6-advanced)
  - [6.4 Mastery — workers, WebAssembly, Lightning CSS, CSP](#part-6-mastery)
  - [6.5 Cheat sheet: Part 6](#part-6-cheat-sheet)

---

<a id="part-4-the-config-system"></a>

## Part 4: The Config System

<a id="part-4-beginner"></a>

### 4.1 Beginner — `defineConfig` and zero-config

Vite works without a config file. The `vanilla-ts` template ships none (*verified locally*). You add `vite.config.ts` when you need plugins or want to change a default:

```ts
// vite.config.ts
import react from '@vitejs/plugin-react'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [react()],
})
```

`defineConfig` does nothing at runtime. It exists for type inference and editor autocomplete. `satisfies UserConfig` or a JSDoc `@type` comment does the same job ([Configuring Vite](https://vite.dev/config/)). Vite looks for `vite.config.{js,mjs,ts,cjs,mts,cts}` in the project root. `--config path` points it somewhere else.

The config file **runs in Node, not the browser**. That's why `tsconfig.node.json` types it with `@types/node`, and why `import.meta.env` is **not** available inside it (§4.2).

<a id="part-4-working"></a>

### 4.2 Working Knowledge — conditional config, aliases, `loadEnv`

**Conditional config.** Export a function, and you get `command`, `mode`, `isSsrBuild` and `isPreview` ([Configuring Vite](https://vite.dev/config/)):

```ts
export default defineConfig(({ command, mode, isPreview }) => ({
  plugins: [react(), command === 'build' && visualizer()],   // falsy plugins are ignored
  build: { sourcemap: mode === 'staging' },
  server: isPreview ? {} : { open: true },
}))
```

- `command` is `'serve'` for the dev server and `'build'` for `vite build`. Under `vite preview`, `isPreview` is `true`.
- `mode` is `'development'` or `'production'` by default, or whatever `--mode` says.
- The function can be `async`, which is useful for reading a file or fetching a value at startup.

**Path aliases.** Two ways to get `@/components/Button`:

```ts
// Option A (Vite 8, recommended when tsconfig already has paths): one source of truth
export default defineConfig({
  resolve: { tsconfigPaths: true },
})
```

```jsonc
// tsconfig.app.json
{ "compilerOptions": { "paths": { "@/*": ["./src/*"] } } }
```

```ts
// Option B: explicit alias, which also works for non-TS files and CSS @import
import { fileURLToPath, URL } from 'node:url'
export default defineConfig({
  resolve: { alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) } },
})
```

*Verified locally*: Option A resolves `import { setupCounter } from '@/counter'` in a production build with no plugin. `resolve.tsconfigPaths` is off by default because it has a small performance cost, and it doesn't support Less ([Shared Options](https://vite.dev/config/shared-options)).

> **Outdated convention alert.** Pre-2026 setups install `vite-tsconfig-paths`. Vite 8 has this built in. If Vite detects that plugin, it logs a warning telling you to remove it and set `resolve.tsconfigPaths: true` (found in Vite 8.3.0's source, *verified locally*). Option B's alias must be an **absolute path**. `alias: { '@': '/src' }` appears in many tutorials and only works by accident, because `/src` happens to be read as root-relative ([Shared Options](https://vite.dev/config/shared-options)).

**Env vars inside the config: `loadEnv`.** `.env` files are *not* loaded into `process.env` before your config runs. *Verified locally*:

```text
CONFIG process.env.APP_PORT = undefined | VITE_FROM_FILE = undefined
CONFIG loadEnv APP_PORT = 6123
```

```ts
// Wrong: undefined. .env hasn't been read into process.env at this point.
export default defineConfig({
  server: { port: Number(process.env.APP_PORT) },            // NaN → falls back / errors
  define: { __API__: JSON.stringify(process.env.VITE_API_URL) },
})

// Right: load the env yourself for the current mode
import { defineConfig, loadEnv } from 'vite'
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '')  // '' = all vars, not just VITE_*
  return {
    server: { port: env.APP_PORT ? Number(env.APP_PORT) : 5173 },
  }
})
```

([Configuring Vite — using environment variables in config](https://vite.dev/config/); a common form of the question is [on Stack Overflow](https://stackoverflow.com/questions/66933757).)

The third argument is a prefix filter. `''` gives you **everything** in `.env`, which is right for config-time values like ports and proxy targets, and **dangerous if you then pass the result to `define`** (§4.3).

**`server.proxy`, `server.host` and friends** are in [Part 2](./vite-dev-server.md#part-2-advanced).

<a id="part-4-advanced"></a>

### 4.3 Advanced — `define`, `base`, `root`, and config for several consumers

**`define` does text replacement, not variable injection.** Each key is replaced in your source with the value's *code*. Strings must be `JSON.stringify`-ed or they turn into identifiers ([Shared Options](https://vite.dev/config/shared-options)). *Verified locally*:

```ts
// Wrong: __BAD__ becomes the bare identifier `hello` → ReferenceError at runtime
define: { __BAD__: 'hello' }
// built output: console.log("BADDEF", hello)

// Right
define: { __APP_VERSION__: JSON.stringify(process.env.npm_package_version) }
```

```ts
// src/globals.d.ts: define doesn't type itself
declare const __APP_VERSION__: string
```

```ts
// Wrong: publishes every env var your CI has (tokens, DB URLs) into the bundle
define: { 'process.env': process.env }
// or
define: { 'process.env': loadEnv(mode, process.cwd(), '') }
```

Vite 8.3 detects the first pattern and warns that it *"can unintentionally expose all environment variables"* (warning text found in Vite 8.3.0's source, *verified locally*). The second pattern gets no warning. People usually write this while chasing ["process is not defined"](https://stackoverflow.com/questions/76736942) from a dependency written for webpack. The safe fix is to define only the specific key the library reads: `define: { 'process.env.NODE_ENV': JSON.stringify(mode) }`.

**`base`: the public path your app is served from.** Default `/`. Set it when the app lives at a sub-path (`/dashboard/`, GitHub Pages `/<repo>/`) or on a CDN (`https://cdn.example.com/app/`). Relative `./` works for apps opened from arbitrary locations ([Building for Production](https://vite.dev/guide/build)). It's also available as `vite build --base=/dashboard/` and at runtime as `import.meta.env.BASE_URL`. [Part 8](./vite-build-deploy.md#part-8-deploying-an-spa) covers deploying with it.

**`root`** moves where `index.html` lives (default `process.cwd()`). Relative paths in the config, such as `publicDir`, `build.outDir` and `envDir`, are resolved from `root` ([Shared Options](https://vite.dev/config/shared-options)). If you set `root: 'src'`, `build.outDir` defaults to `src/dist` unless you change it.

**Top-level `input` (8.2+) and `tsconfig` (8.3+).** `input` sets the entry points once for build, dependency optimization and library mode ([Shared Options](https://vite.dev/config/shared-options); `build.lib.entry` falls back to it, per the 8.3 types, *verified locally*). `tsconfig: './tsconfig.app.json'` tells Vite which tsconfig to read for `paths` and compiler options, which matters with the template's solution-style root tsconfig ([Changelog](https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md)).

**One config, several consumers.** `vite.config.ts` is read by `vite`, `vite build`, `vite preview` and **Vitest**. Use that instead of fighting it:

```ts
export default defineConfig(({ command, mode }) => ({
  plugins: [react(), mode !== 'test' && checker()],   // Vitest runs with mode 'test'
  test: { environment: 'jsdom' },                     // ignored by vite itself
}))
```

Creating a separate `vitest.config.ts` **replaces** `vite.config.ts` for tests instead of merging with it. See [Part 12](./vite-testing-vitest.md#part-12-working).

**Real Scenario — the staging build that shipped production analytics.** A team adds `build:staging` as `vite build --mode staging` and a `.env.staging` with a different `VITE_ANALYTICS_ID`. The config also has `plugins: [mode === 'production' && sentryUpload()]`. Staging builds never upload source maps, so Sentry stack traces are unreadable, and nobody notices until an incident. Separately, a component checks `if (import.meta.env.MODE === 'production')` to enable analytics, so staging sends no events. Both bugs come from treating `mode` as meaning "production-ness". Use `command === 'build'` or `import.meta.env.PROD` for "is this an optimized build?", and `mode` only for *which* environment's values to load ([Env and Mode](https://vite.dev/guide/env-and-mode)).

<a id="part-4-mastery"></a>

### 4.4 Mastery — how the config file is loaded

By default Vite **bundles** your config file into a temporary file under `node_modules/.vite-temp/` and imports that. The `--configLoader` flag chooses the strategy ([Configuring Vite](https://vite.dev/config/)):

| Loader | How | Status |
|---|---|---|
| `bundle` | bundles config + its local imports, then imports the result | **current default** |
| `runner` | transforms on the fly with Vite's module runner | alternative |
| `native` | Node imports `vite.config.ts` directly (Node's type stripping) | **planned future default** |

The bundling step hides differences between Node and a bundler. `native` removes it, and some configs stop working. **Since 8.2, Vite warns about these on every run.** *Verified locally* on 8.3.0, from an ordinary-looking config:

```text
(!) Your Vite config uses features that are unsupported by `configLoader: 'native'`,
    which is planned to become the default in a future major version of Vite:
  - `__dirname` (vite.config.ts:7:36). Use `import.meta.dirname` instead
  - import "./config/shared" without a file extension (vite.config.ts:3:22). Add the file extension
  - JSON import "./config/ports.json" without import attributes (vite.config.ts:4:28). Add `with { type: 'json' }`
Set `VITE_CONFIG_NATIVE_IGNORE_WARNING=true` to suppress this warning.
```

With `--configLoader native`, the same config fails: `ERR_MODULE_NOT_FOUND … config\shared`. Code that is future-proof today:

```ts
// Wrong: works under the bundle loader, breaks under native
import { resolve } from 'node:path'
import { port } from './config/shared'
import ports from './config/ports.json'
const src = resolve(__dirname, 'src')

// Right
import { resolve } from 'node:path'
import { port } from './config/shared.ts'
import ports from './config/ports.json' with { type: 'json' }
const src = resolve(import.meta.dirname, 'src')
```

The same pass flags **named imports from JSON** (`import { port } from './x.json'`), because JSON modules only have a default export per spec (Vite 8.3.0 source, *verified locally*). This check covers the *config file*. Named JSON imports in *app code* are still supported by Vite (§6.2).

Other loading facts worth knowing:

- **ESM-only dependencies in config.** If the config is treated as CommonJS, importing an ESM-only plugin fails with "This package is ESM only". Fix it with `"type": "module"` in `package.json` or a `.mts` config ([Troubleshooting](https://vite.dev/guide/troubleshooting)). Templates already set `"type": "module"`.
- **Debugging the config in VS Code.** Use `--configLoader native` for reliable breakpoints, or map `**/node_modules/.vite-temp/**` in source-map resolution ([Configuring Vite](https://vite.dev/config/)).
- **Plugin `config` hooks run after your file** and can change the result. What Vite actually uses is only visible in `configResolved` ([Part 10](./vite-plugins-monorepo.md#part-10-advanced)).
- **`future` option.** `future: { someChange: 'warn' }` turns on warnings for upcoming breaking changes early ([Shared Options](https://vite.dev/config/shared-options)).

<a id="part-4-cheat-sheet"></a>

### 4.5 Cheat sheet: Part 4

| Need | Config |
|---|---|
| Branch on dev/build | `defineConfig(({ command }) => …)` |
| Branch on environment | `defineConfig(({ mode }) => …)` + `--mode staging` |
| `.env` value in config | `loadEnv(mode, process.cwd(), '')` |
| tsconfig `paths` | `resolve.tsconfigPaths: true` (not `vite-tsconfig-paths`) |
| Manual alias | `resolve.alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) }` |
| Compile-time constant | `define: { __X__: JSON.stringify(value) }` + `declare const` |
| Sub-path deployment | `base: '/app/'` |
| Pick tsconfig (8.3+) | `tsconfig: './tsconfig.app.json'` |
| Future-proof config | `.ts` extensions on imports, `import.meta.dirname`, `with { type: 'json' }` |
| See final config | a plugin's `configResolved(config)` |

[↑ Back to top](#table-of-contents)

---

<a id="part-5-env-variables-and-modes"></a>

## Part 5: Env Variables & Modes

<a id="part-5-beginner"></a>

### 5.1 Beginner — `import.meta.env`

Vite exposes build-time constants on `import.meta.env` ([Env and Mode](https://vite.dev/guide/env-and-mode)):

| Constant | Meaning |
|---|---|
| `MODE` | the mode string (`development`, `production`, `staging`, `test`…) |
| `DEV` / `PROD` | is this an unoptimized dev server / an optimized build? |
| `SSR` | is this code running in the server environment? |
| `BASE_URL` | the `base` config value |
| `VITE_*` | your variables from `.env*` and the shell |

```ts
if (import.meta.env.DEV) {
  console.log('dev-only diagnostics')   // removed entirely from production builds
}
fetch(`${import.meta.env.VITE_API_URL}/users`)
```

These are **statically replaced at build time**. Code inside `if (import.meta.env.DEV)` doesn't just skip at runtime: the minifier deletes it.

> **Outdated convention alert.** `process.env.REACT_APP_*` (Create React App) and `process.env.VUE_APP_*` (Vue CLI) don't exist in Vite. There is no `process` in the browser. Rename them to `VITE_*` and read them from `import.meta.env`.

<a id="part-5-working"></a>

### 5.2 Working Knowledge — `.env` files, `VITE_`, and what gets published

**Load order** ([Env and Mode](https://vite.dev/guide/env-and-mode)). Later entries win, and variables **already set in the shell beat all of them**:

```text
  .env                  always loaded
  .env.local            always loaded, git-ignored
  .env.[mode]           only for that mode
  .env.[mode].local     only for that mode, git-ignored
  ─────────────────────────────────────────────
  shell / CI env vars   highest priority, never overwritten by files
```

**The `VITE_` prefix decides what is published.** Only prefixed variables reach client code ([Env and Mode](https://vite.dev/guide/env-and-mode)):

```bash
# .env
VITE_API_URL=https://api.example.com   # → in the bundle
SECRET_TOKEN=sk_live_…                 # → not in the bundle
```

This is what a production build actually contains (*verified locally*, 8.3.0, `.env` with `VITE_API_URL`, `VITE_LEAKED=sk_live_oops` and `SECRET_TOKEN=…`):

```js
// dist/assets/index-*.js (minified, excerpt), from code that did console.log(import.meta.env)
ne={BASE_URL:`/`,DEV:!1,MODE:`staging`,PROD:!0,SSR:!1,
    VITE_API_URL:`https://api.example.com`,VITE_LEAKED:`sk_live_oops`}
```

Two conclusions from that output:

1. **`SECRET_TOKEN` is absent.** The prefix works.
2. **Referencing `import.meta.env` as an object inlines every `VITE_*` variable**, including ones this file never uses. One debug `console.log(import.meta.env)` publishes the full set.

Shell variables work the same way. `VITE_FROM_SHELL=shell-value vite build` put `shell-value` into the bundle with no `.env` entry at all (*verified locally*). A CI system that exports all its secrets as `VITE_*` for convenience publishes all of them.

```bash
# Wrong: the prefix doesn't make a value "safe"; it makes it public
VITE_STRIPE_SECRET_KEY=sk_live_…
VITE_OPENAI_API_KEY=sk-…

# Right: public identifiers only; secrets stay on a server you control
VITE_STRIPE_PUBLISHABLE_KEY=pk_live_…
VITE_API_URL=https://api.example.com    # the backend holds the real keys
```

The docs say it directly: *"VITE_\* variables should not contain sensitive information such as API keys"* ([Env and Mode](https://vite.dev/guide/env-and-mode)).

```ts
// Wrong: envPrefix '' exposes EVERYTHING. The docs warn against it explicitly.
export default defineConfig({ envPrefix: '' })

// Fine: a different public prefix, e.g. while migrating from CRA
export default defineConfig({ envPrefix: ['VITE_', 'REACT_APP_'] })
```

([Shared Options — envPrefix](https://vite.dev/config/shared-options).)

**Values are always strings.** `VITE_ENABLE_X=false` is the string `"false"`, which is truthy ([Env and Mode](https://vite.dev/guide/env-and-mode)):

```ts
// Wrong
if (import.meta.env.VITE_ENABLE_BETA) showBeta()        // "false" is truthy

// Right
if (import.meta.env.VITE_ENABLE_BETA === 'true') showBeta()
```

**Expansion and restarts.** `.env` supports `NEW=${OLD}`-style expansion via dotenv-expand. Escape a literal `$` as `\$`. `.env` is read once at startup, so **restart `vite` after editing it** ([Env and Mode](https://vite.dev/guide/env-and-mode)).

**Real Scenario — the "publishable" key that wasn't.** A developer wiring up a payments sandbox copies both keys from the dashboard into `.env` as `VITE_STRIPE_PK` and `VITE_STRIPE_SK`, "to keep them together". Only the publishable key is used in code, so they assume the secret one is harmless. A week later, the security scanner on a dependency-update PR flags `sk_live_` in `dist/assets/index-*.js`. The cause is an error boundary that logs `import.meta.env` for diagnostics. The key is rotated, `VITE_STRIPE_SK` is removed, and the team adds a CI grep for `sk_live_` in `dist/`.

<a id="part-5-advanced"></a>

### 5.3 Advanced — modes vs. `NODE_ENV`, typing, HTML replacement

**Mode and `NODE_ENV` are separate settings** ([Env and Mode](https://vite.dev/guide/env-and-mode)):

| Command | `NODE_ENV` | `mode` / `MODE` | `PROD` |
|---|---|---|---|
| `vite` | `development` | `development` | `false` |
| `vite build` | `production` | `production` | `true` |
| `vite build --mode staging` | `production` | `staging` | `true` (*verified locally*) |
| `vite build --mode development` | `production` | `development` | — |
| `NODE_ENV=development vite build` | `development` | `production` | — |
| `vitest` | `test` | `test` | `false` (*verified locally*) |

So `--mode staging` gets you an **optimized** build that loads `.env.staging`. It is not a dev build. That's usually what you want. `NODE_ENV=development vite build` is rarely what you want, since it can switch dependencies like React into their development builds.

**Typing your variables.** `vite/client` types the built-ins. Your own go in a declaration file ([Env and Mode](https://vite.dev/guide/env-and-mode)):

```ts
// src/vite-env.d.ts: NO import statements, or augmentation silently stops working
interface ViteTypeOptions {
  strictImportMetaEnv: unknown          // opt in: unknown keys become type errors
}

interface ImportMetaEnv {
  readonly VITE_API_URL: string
  readonly VITE_ENABLE_BETA?: 'true' | 'false'
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

Without `strictImportMetaEnv`, a typo like `import.meta.env.VITE_API_ULR` type-checks as `any`-ish and is `undefined` at runtime.

**Validating at startup** catches missing variables before users do:

```ts
// src/env.ts: import this instead of reading import.meta.env throughout the app
const required = ['VITE_API_URL'] as const
for (const key of required) {
  if (!import.meta.env[key]) throw new Error(`Missing ${key}; check .env.${import.meta.env.MODE}`)
}
export const env = { apiUrl: import.meta.env.VITE_API_URL }
```

`import.meta.env[key]` with a *dynamic* key can't be statically replaced. In production it only works because the whole-object reference inlines every `VITE_*` variable (§5.2). That's harmless here, since `VITE_*` values are public anyway, but it's also why this pattern increases bundle size. For a small app, read each key literally.

**HTML replacement.** `%VITE_API_URL%` and `%MODE%` work in `index.html`. Unknown names are **left as literal text** rather than becoming `undefined` ([Env and Mode](https://vite.dev/guide/env-and-mode)). *Verified locally*: `<title>%VITE_FROM_FILE% %MODE% %NOPE%</title>` built to `<title>file-value production %NOPE%</title>`.

**`envDir`** moves where `.env` files are read from. It's useful in monorepos that share one `.env` at the root. `envDir: false` disables `.env` loading entirely ([Shared Options](https://vite.dev/config/shared-options)).

**Bun caveat.** Bun auto-loads `.env` into `process.env` before scripts run. Variables then arrive as "shell" variables with top priority, and mode-specific files can appear not to apply ([Env and Mode](https://vite.dev/guide/env-and-mode)). See [`bun/`](../bun/bun-mastery-guide.md).

<a id="part-5-mastery"></a>

### 5.4 Mastery — build-time values in a build-once, deploy-many world

The problem: good deployment practice builds **one** Docker image and promotes it from staging to production. But `VITE_API_URL` was frozen into `dist/assets/index-*.js` when the image was built. Promoting the image promotes the staging URL.

```text
  CI: docker build  ──► vite build (VITE_API_URL=https://staging-api) ──► image:abc123
                                                                              │
  staging:    run image:abc123  ✔ calls staging-api                           │
  production: run image:abc123  ✘ ALSO calls staging-api  ◄───────────────────┘
```

People keep asking about this: see ["env variables are undefined inside the docker container"](https://stackoverflow.com/questions/77486735), ["pass environment variables from docker-compose.yml to vite"](https://stackoverflow.com/questions/76457026) and ["change environment variables at runtime with docker and nginx"](https://stackoverflow.com/questions/70617812). Setting `environment:` in compose doesn't help, because the container runs nginx serving static files, not Vite.

Three solutions, from most to least robust:

**1. Same-origin API (no URL needed at all).** Serve the SPA and proxy `/api` from the same host, and call relative URLs: `fetch('/api/users')`. Nothing environment-specific ends up in the bundle. This is the production equivalent of `server.proxy`. See [Part 8](./vite-build-deploy.md#part-8-advanced).

**2. Runtime config file.** Build a placeholder, and let the container write real values at start:

```html
<!-- index.html: loaded before your module script, not processed by Vite -->
<script src="/config.js"></script>
```

```sh
# docker-entrypoint.d/40-config.sh (runs in the nginx image before nginx starts)
cat > /usr/share/nginx/html/config.js <<EOF
window.__APP_CONFIG__ = { apiUrl: "${API_URL}" };
EOF
```

```ts
// src/env.ts
declare global { interface Window { __APP_CONFIG__?: { apiUrl: string } } }
export const apiUrl = window.__APP_CONFIG__?.apiUrl ?? import.meta.env.VITE_API_URL
```

Put `config.js` in `public/` for dev (it's copied verbatim and not hashed), and serve it with `Cache-Control: no-cache` ([Part 8](./vite-build-deploy.md#part-8-advanced)).

**3. Placeholder substitution at container start.** Build with `VITE_API_URL=__API_URL__`, then `sed` the placeholder in `dist/assets/*.js` when the container starts. This is the approach in the top answer to the [runtime-env Stack Overflow question](https://stackoverflow.com/questions/70617812). It works, but it rewrites hashed files without changing their hashes, so any CDN or browser that cached the staging version keeps it. Prefer 1 or 2.

**Real Scenario — the promotion that pointed prod at staging.** A team moves from "build per environment" to "build once, promote" to save CI time. The Dockerfile still runs `vite build` with `ARG VITE_API_URL`. The first promoted release sends production users' logins to the staging API, which has a separate user database, so every login fails. Rollback takes 10 minutes. The postmortem action item is option 1: the frontend has no idea what its API origin is.

<a id="part-5-cheat-sheet"></a>

### 5.5 Cheat sheet: Part 5

| Question | Answer |
|---|---|
| Which files load for `--mode staging`? | `.env`, `.env.local`, `.env.staging`, `.env.staging.local` |
| Who wins, `.env` or shell? | Shell |
| What reaches the browser? | Only `envPrefix` (default `VITE_`) vars, plus built-ins |
| `console.log(import.meta.env)` in prod? | Publishes every `VITE_*` var |
| `VITE_FLAG=false` in an `if`? | Truthy string: compare to `'true'` |
| Changed `.env`, nothing happened? | Restart `vite` |
| `--mode staging` optimized? | Yes: `NODE_ENV=production`, `PROD=true` |
| Env in `vite.config.ts`? | `loadEnv`, not `process.env` |
| Env in `index.html`? | `%VITE_X%`; unknown stays literal |
| Typo safety | `strictImportMetaEnv` in `ViteTypeOptions` |
| One image, many environments | same-origin `/api`, or runtime `config.js` |

[↑ Back to top](#table-of-contents)

---

<a id="part-6-assets-css-and-typescript"></a>

## Part 6: Assets, CSS & TypeScript

<a id="part-6-beginner"></a>

### 6.1 Beginner — importing assets and CSS

```ts
import './styles.css'                   // injected into the page (dev: <style>, build: <link>)
import heroUrl from './assets/hero.png' // a URL string

document.querySelector<HTMLImageElement>('#hero')!.src = heroUrl
```

What the URL turns into (*verified locally*):

| File | Size | Dev URL | Build result |
|---|---|---|---|
| `src/assets/hero.png` | 13 KB | `/src/assets/hero.png` | `/assets/hero-CLDdwZDr.png`, a hashed file |
| `src/tiny.svg` | < 1 KB | `/src/tiny.svg` | `data:image/svg+xml,%3csvg…`, **inlined** (URL-encoded, not base64) |

Assets under `build.assetsInlineLimit` (default **4096 bytes**) are inlined as data URIs, and larger ones become hashed files ([Build Options](https://vite.dev/config/build-options)). `url()` references inside CSS go through the same pipeline and are rebased automatically ([Features — CSS](https://vite.dev/guide/features)).

Hashed filenames change when the content changes, which is what lets you cache `dist/assets/` forever ([Part 8](./vite-build-deploy.md#part-8-advanced)).

<a id="part-6-working"></a>

### 6.2 Working Knowledge — query suffixes, CSS Modules, glob imports, JSON

**Query suffixes override the default handling** ([Features — Static Assets](https://vite.dev/guide/features)):

```ts
import shaderSrc from './shader.glsl?raw'        // file contents as a string
import pdfUrl from './manual.pdf?url'            // URL, never inlined
import iconData from './icon.svg?inline'         // always inlined
import cardCss from './card.css?inline'          // CSS as a string, NOT injected into the page
import MyWorker from './worker.ts?worker'        // Worker constructor (§6.4)
```

**CSS Modules.** Any `*.module.css` file exports a class-name map ([Features — CSS](https://vite.dev/guide/features)). *Verified locally*, `.card` and `.card-title` built to:

```js
{ card: `_card_cgqnp_1`, "card-title": `_card-title_cgqnp_1` }
```

```ts
// Wrong: kebab-case class names aren't converted by default
import styles from './card.module.css'
el.className = styles.cardTitle                      // undefined → class="undefined"

// Right: bracket access, or ask Vite to add camelCase keys
styles['card-title']
// vite.config.ts → css: { modules: { localsConvention: 'camelCaseOnly' } }
```

`css.modules` options go to `postcss-modules` and **have no effect when `css.transformer: 'lightningcss'`** ([Shared Options](https://vite.dev/config/shared-options)).

**PostCSS and preprocessors.** A `postcss.config.js` is picked up automatically. For Sass, install `sass-embedded` (or `sass`) and import `.scss`. Less and Stylus work the same way, with no plugin needed ([Features — CSS](https://vite.dev/guide/features)). For Tailwind v4, use `@tailwindcss/vite`; see [`tailwindcss/`](../tailwindcss/tailwindcss-mastery-guide.md).

**Glob imports** load many modules by pattern, which makes file-based registries straightforward ([Features — Glob Import](https://vite.dev/guide/features)):

```ts
// Lazy (default): each value is () => import(path), so each file becomes its own chunk
const pages = import.meta.glob('./pages/*.tsx')

// Eager: imported up front, all in the same chunk
const locales = import.meta.glob('./locales/*.json', { eager: true, import: 'default' })

// Only one named export, excluding tests
const routes = import.meta.glob(['./routes/**/*.ts', '!**/*.test.ts'], { import: 'route' })

// Raw strings
const mdSources = import.meta.glob('./docs/*.md', { query: '?raw', import: 'default' })
```

Rules: the pattern must be a **string literal** (no variables) and must start with `./`, `/` or an alias. `caseSensitive: false` exists since 8.1 ([8.1 announcement](https://vite.dev/blog/announcing-vite8-1)).

**Dynamic import with variables** is a narrower alternative:

```ts
// Works: static prefix, static extension
const mod = await import(`./locales/${lang}.json`)

// Doesn't: fully dynamic, so Vite can't know which files to include
const mod2 = await import(someRuntimePath)
```

Import paths must start with `./` or `../` (or `#` subpath imports, since 8.3) and end with a file extension ([Features](https://vite.dev/guide/features), [Changelog](https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md)).

**JSON.** `import pkg from './data.json'` and `import { version } from './data.json'` both work in app code, and the named form tree-shakes (`json.namedExports` defaults to `true`). JSON over 10 kB is emitted as `JSON.parse("…")` by default (`json.stringify: 'auto'`), which parses faster ([Shared Options](https://vite.dev/config/shared-options)). Named JSON imports are non-standard, though: Node and browsers only give JSON modules a default export. That's why Vite's future native config loader flags them in *config* files ([§4.4](#part-4-mastery)). Prefer the default import in code that might run outside Vite.

<a id="part-6-advanced"></a>

### 6.3 Advanced — TypeScript: transpile-only, and what that costs you

Vite **does not type-check**. It strips types file by file with Oxc, which is fast enough that HMR updates land in under 50 ms ([Features — TypeScript](https://vite.dev/guide/features)). *Verified locally*: a file with `const n: number = 'not a number'` passes `vite build` and ships the string. `tsc` reports `TS2322`.

```json
// Wrong: type errors reach production
{ "scripts": { "build": "vite build" } }

// Right: what the templates do (verified locally)
{ "scripts": { "build": "tsc -b && vite build" } }        // React / vanilla (vanilla uses plain `tsc`)
{ "scripts": { "build": "vue-tsc -b && vite build" } }    // Vue: tsc can't read .vue files
```

For in-editor feedback during dev, run `tsc --noEmit --watch` in a second terminal, or use `vite-plugin-checker` to show type errors in the browser overlay ([Features — TypeScript](https://vite.dev/guide/features)).

**tsconfig settings that Vite depends on** ([Features — TypeScript](https://vite.dev/guide/features)):

| Setting | Why |
|---|---|
| `isolatedModules: true` (implied by `verbatimModuleSyntax`) | Oxc sees one file at a time. It can't inline a `const enum` from another file or tell whether an import is a type |
| `useDefineForClassFields` | Defaults to `true` for ES2022+ targets. Changes class-field semantics; matters for decorator-based libraries |
| `target` | **Ignored by Vite.** Dev uses `oxc.target` (`esnext`), build uses `build.target` |
| `paths` | Ignored unless `resolve.tsconfigPaths: true` |
| `emitDecoratorMetadata` | Only partially supported (needs type information Oxc doesn't have) |

**`enum` and `erasableSyntaxOnly`.** *Verified locally*: `export enum Color { Red = 'red' }` builds fine with Vite, since Oxc compiles it into an IIFE. But the templates set `erasableSyntaxOnly: true`, so `tsc -b` rejects it: `TS1294: This syntax is not allowed when 'erasableSyntaxOnly' is enabled`. The templates are steering you toward syntax that can be removed without compiling anything: union types or `as const` objects instead of enums, and no `namespace` or parameter properties. That's the same rule Node's built-in type stripping follows.

```ts
// Rejected by the template's tsc (compiles fine in Vite, which makes it confusing)
export enum Role { Admin = 'admin', Member = 'member' }

// Erasable equivalent
export const Role = { Admin: 'admin', Member: 'member' } as const
export type Role = (typeof Role)[keyof typeof Role]
```

**`import type`.** With `verbatimModuleSyntax`, an import used only as a type must say so. Otherwise the import survives into JS and fails at runtime when the module has no such export:

```ts
// Wrong: `User` is a type-only export; the emitted JS still imports it
import { User } from './types'

// Right
import type { User } from './types'
```

**Real Scenario — green CI, broken checkout.** A team's pipeline runs `vite build` and Playwright smoke tests, but not `tsc`. A refactor renames `price` to `unitPrice` on an API type. Every usage but one is updated, and the missed one is in the rarely exercised gift-card path. Vite builds happily and the smoke tests don't cover gift cards. Production shows `NaN` totals for gift cards for two days. The fix takes one line: `"build": "tsc -b && vite build"`, which the template already had before someone "sped up CI" by removing `tsc -b`.

<a id="part-6-mastery"></a>

### 6.4 Mastery — workers, WebAssembly, Lightning CSS, CSP

**Web Workers: prefer the standard constructor form** ([Features — Web Workers](https://vite.dev/guide/features)):

```ts
// Recommended: standard syntax, works outside Vite too
const worker = new Worker(new URL('./worker.ts', import.meta.url), { type: 'module' })

// Vite-specific: handy, but not portable
import MyWorker from './worker.ts?worker'
const w = new MyWorker()
```

Both produce a hashed `dist/assets/worker-*.js` chunk. The `new URL(…, import.meta.url)` pattern **must be written inline**, because Vite detects it statically. Building the URL in a variable first breaks detection. *Verified locally*: the `?worker` form compiled to `new Worker('/assets/worker-….js', { name })`, i.e. a **classic** worker, while the constructor form kept `{ type: 'module' }`. Either way the worker *bundle* is emitted as `iife` unless you set `worker.format: 'es'` (default `'iife'`, per the 8.3 types). You need `'es'` when the worker itself uses code splitting (dynamic `import()`). Since 8.3, query params on worker URLs are preserved, and a worker chunk that is never referenced is removed ([Changelog](https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md)).

**WebAssembly.** Since 8.1 you can import `.wasm` directly with the ESM integration proposal. `?init` gives manual control with an import object ([Features — WebAssembly](https://vite.dev/guide/features), [8.1 announcement](https://vite.dev/blog/announcing-vite8-1)):

```ts
import { add } from './add.wasm'                  // 8.1+: instantiated for you
import init from './legacy.wasm?init'             // manual
const instance = await init({ env: { log: console.log } })
```

TypeScript needs `allowArbitraryExtensions: true` (already set in the templates, *verified locally*) and an `add.d.wasm.ts` declaration.

**Lightning CSS.** Vite 8 always **minifies** CSS with Lightning CSS (`build.cssMinify: 'lightningcss'`). **Transforming** CSS with it is opt-in and experimental: `css.transformer: 'lightningcss'` replaces PostCSS for `@import`, nesting, vendor prefixing and CSS Modules ([Features — CSS](https://vite.dev/guide/features), [Shared Options](https://vite.dev/config/shared-options)). The team plans to make it the default transformer in the next major ([8.1 announcement](https://vite.dev/blog/announcing-vite8-1)). What changes when you switch: `postcss.config.js` stops applying, and `css.modules` options are ignored in favour of `css.lightningcss.cssModules`.

```ts
export default defineConfig({
  css: {
    transformer: 'lightningcss',
    lightningcss: { cssModules: { pattern: '[name]_[local]_[hash]' } },
  },
  build: { cssTarget: ['chrome111', 'safari16.4'] },  // CSS can target differently from JS
})
```

**Content Security Policy** ([Features — CSP](https://vite.dev/guide/features)):

- **Inlined assets are `data:` URIs.** A strict `img-src 'self'` blocks every sub-4 KB image, which shows up as broken icons in production only. Either allow `data:` for `img-src`/`font-src`, or set `build.assetsInlineLimit: 0`. **Never** allow `data:` in `script-src`.
- **Nonces.** `html.cspNonce: 'PLACEHOLDER'` adds `nonce="PLACEHOLDER"` to generated `<script>`, `<style>` and `<link>` tags and a `<meta property="csp-nonce">`. Your server must **replace the placeholder with a fresh random value per request**. A static nonce provides no protection.
- **Dev injects inline scripts and styles** (the HMR client, the React Refresh preamble, CSS `<style>` tags). Keep strict CSP headers off the dev server and test them against `vite preview` or a staging deploy.

**License files.** `build.license: true` writes `dist/.vite/license.md` listing bundled dependencies and their licenses ([Features — License](https://vite.dev/guide/features)). Some legal reviews require this.

<a id="part-6-cheat-sheet"></a>

### 6.5 Cheat sheet: Part 6

| Import / setting | Result |
|---|---|
| `import url from './a.png'` | hashed URL; data URI if < 4096 B |
| `?url` / `?inline` / `?raw` | force URL / force inline / string contents |
| `import './a.css'` | injected; `?inline` → string, not injected |
| `*.module.css` | class map; kebab keys need brackets or `localsConvention` |
| `import.meta.glob('./x/*.ts')` | lazy map; `{ eager: true }` for static |
| `` import(`./l/${x}.json`) `` | limited dynamic import: static prefix and extension |
| `import { v } from './a.json'` | works in app code; prefer default import for portability |
| `new Worker(new URL('./w.ts', import.meta.url), { type: 'module' })` | recommended worker form |
| `import { f } from './a.wasm'` | 8.1+ ESM integration |
| Type-checking | not done by Vite: `tsc -b` / `vue-tsc -b` in `build` |
| `enum`, `namespace` | compile in Vite, rejected by template `erasableSyntaxOnly` |
| CSS minify | Lightning CSS (default) |
| CSS transform | PostCSS (default) or `css.transformer: 'lightningcss'` |
| Strict CSP | allow `data:` for images/fonts or `assetsInlineLimit: 0`; `html.cspNonce` |

[↑ Back to top](#table-of-contents)
