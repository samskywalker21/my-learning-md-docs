# Monorepo Mastery Guide (Node.js / Bun)

## Table of Contents

- [About This Document](#about-this-document)
- [1. Why Monorepos](#1-why-monorepos)
- [2. Workspace Fundamentals](#2-workspace-fundamentals)
  - [2.1 npm workspaces](#21-npm-workspaces)
  - [2.2 pnpm workspaces](#22-pnpm-workspaces)
  - [2.3 Bun workspaces](#23-bun-workspaces)
  - [2.4 Hoisting & node_modules layout](#24-hoisting--node_modules-layout)
- [3. Internal Package Linking](#3-internal-package-linking)
- [4. Dependency Management at Scale](#4-dependency-management-at-scale)
- [5. Task Running & Turborepo](#5-task-running--turborepo)
- [6. CI/CD Considerations](#6-cicd-considerations)
- [7. Project Layout Conventions](#7-project-layout-conventions)
- [8. Troubleshooting & Common Gotchas](#8-troubleshooting--common-gotchas)
- [9. Suggested Learning/Reference Order](#9-suggested-learningreference-order)
- [10. Quick Self-Check](#10-quick-self-check)

---

## About This Document

**Spec captured from the originating request** (so future updates can match style without re-explaining):

- **Framing:** Goal-driven — the reader is learning this before building real monorepo projects. Depth/examples are prioritized toward concrete commands, flags, and config actually touched in practice, not abstract survey.
- **Scope confirmed:** Node.js monorepos, comparing **npm workspaces**, **pnpm workspaces**, and **Bun workspaces** side by side, plus **Turborepo** for task orchestration/caching. Assumes solid JS/Node/npm fundamentals but zero prior monorepo-specific experience (starts from zero on this topic).
- **Depth tiers:** Beginner → Working Knowledge → Advanced → Mastery, applied per-section, collapsed where a tier would be trivial/redundant.
- **Per-section ingredients used where they add value:** plain-language explanation, wrong-vs-right code/command snippets, a "Real Scenario" tied to production-style situations, ASCII diagrams for spatial/structural concepts.
- **Accuracy:** Verified against official docs (npm, pnpm, Bun, Turborepo, Changesets) as of **September 2026**. Bun 1.4 (released 2026-08-19) introduced `--catalog`, `--filter` on `add`/`remove`/`update`, and the isolated linker's global virtual store — called out explicitly below since older blog posts predate these.
- **Sourcing convention:** Official docs linked first, Stack Overflow for real-world gotchas, GitHub/other only as supplementary.

[Back to top](#table-of-contents)

---

## 1. Why Monorepos

A monorepo is a single Git repository holding multiple independent packages (apps, libraries, tools) that are versioned and often built together, as opposed to a **polyrepo** where each package lives in its own repository.

**Why it matters:** once you have more than one app sharing code (e.g., a web app and an API sharing a `types` package, or a design system used by three apps), you're choosing between duplicating code across repos, publishing-and-reinstalling internal packages on every change, or putting them in one repo where a single `npm install` links them all together. Monorepos trade repo-level isolation for atomic cross-package commits, simpler local linking, and shared tooling — at the cost of needing workspace tooling and (eventually) a task orchestrator so CI doesn't rebuild/retest everything on every change.

| | Monorepo | Polyrepo |
|---|---|---|
| Cross-package refactor | One PR, one commit | Coordinated PRs across repos |
| Local package linking | Automatic (workspace protocol) | Manual (`npm link`, publish to registry, or `file:`) |
| CI cost as repo grows | Needs affected-only tooling (Turborepo/Nx) or CI runs everything | Naturally scoped per repo |
| Versioning | Can be unified or independent (see [Changesets](#6-cicd-considerations)) | Independent per repo |
| Access control granularity | Repo-wide (unless using sparse checkout / CODEOWNERS) | Per-repo |

[Back to top](#table-of-contents)

---

## 2. Workspace Fundamentals

A **workspace** is a package manager feature: a root `package.json` declares a set of sub-packages (usually via glob patterns), and the package manager installs all of them together, symlinking any workspace package that another workspace package depends on instead of downloading it from the registry.

```
my-monorepo/
├── package.json          ← root, declares "workspaces"
├── apps/
│   ├── web/package.json
│   └── api/package.json
└── packages/
    ├── ui/package.json
    └── config/package.json
```

### 2.1 npm workspaces

**Beginner.** Built into npm (bundled with Node.js) since npm 7 — no extra install. Declare workspaces in the root `package.json`:

```jsonc
// package.json
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": ["apps/*", "packages/*"]
}
```

Run `npm install` once at the root. npm resolves every workspace's dependencies, hoists shared ones to the root `node_modules`, and symlinks workspace packages that reference each other.

**Working Knowledge.**

```bash
# Install a dependency into one workspace only
npm install lodash -w apps/web
# or by package name (must match the "name" field in that workspace's package.json)
npm install lodash --workspace=@myorg/web

# Run a script in one workspace
npm run build -w apps/web

# Run a script in every workspace that has it, skip those that don't
npm run build --workspaces --if-present
```

**Wrong vs. right:**

```bash
# Wrong: installing into the workspace directory directly bypasses the root lockfile
cd apps/web && npm install lodash

# Right: install from the root, targeting the workspace
npm install lodash -w apps/web
```
Running `npm install` inside a workspace subdirectory still works (npm detects the workspace root), but doing it out of habit from the wrong directory is a common source of "why did my lockfile change so much" confusion — always know which directory you're in.

**Real Scenario:** A teammate runs `npm install` inside `apps/web/` instead of the repo root because that's muscle memory from single-package projects. npm still resolves it correctly against the root lockfile in modern versions, but scripts like `postinstall` hooks that assume `cwd` is the repo root can break. Standardize on running install commands from the root.

Reference: [npm workspaces docs](https://docs.npmjs.com/cli/v11/using-npm/workspaces)

### 2.2 pnpm workspaces

**Beginner.** pnpm is a separate install (`npm install -g pnpm` or via corepack). A workspace is declared in a dedicated `pnpm-workspace.yaml` at the root — not in `package.json`:

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

```bash
pnpm install
```

**Working Knowledge.** pnpm's defining feature is a **content-addressable store**: every package version is stored once on disk globally and hard-linked into each project's `node_modules`, rather than duplicated per project. This is why pnpm installs are fast and disk-efficient across many projects, not just within one monorepo.

```bash
# Add a dependency to one workspace package
pnpm add lodash --filter web

# Run a script in every workspace package that has it
pnpm -r run build

# Run in a package and everything that depends on it (useful after editing a shared lib)
pnpm --filter "...ui" run test
```

The **workspace protocol** explicitly pins a dependency to a local sibling package:

```jsonc
// apps/web/package.json
{
  "dependencies": {
    "@myorg/ui": "workspace:*"
  }
}
```

> "When this protocol is used, pnpm will refuse to resolve to anything other than a local workspace package." — [pnpm workspace protocol docs](https://pnpm.io/workspaces)

**Advanced — catalogs.** pnpm's **catalogs** feature centralizes a dependency's version in one place so every workspace package references the same version without manually syncing `package.json` files:

```yaml
# pnpm-workspace.yaml
catalog:
  react: ^19.0.0
  typescript: ^5.9.0
```

```jsonc
// any workspace package.json
{
  "dependencies": { "react": "catalog:" },
  "devDependencies": { "typescript": "catalog:" }
}
```

**Wrong vs. right:**

```jsonc
// Wrong: each package pins its own react version, drifts over time
// apps/web/package.json → "react": "^19.0.0"
// apps/admin/package.json → "react": "^18.3.0"   ← silently different!

// Right: single source of truth via catalog
// pnpm-workspace.yaml → catalog: { react: "^19.0.0" }
// every package.json → "react": "catalog:"
```

**Real Scenario:** Two apps in the monorepo end up on different minor versions of `react` because they were added on different dates. A shared UI package built against React 19 hooks crashes at runtime in the app still on React 18 with a cryptic hooks-dispatcher error — a classic case Stack Overflow threads about "Invalid hook call" trace back to duplicate React copies in the dependency tree. Catalogs (or pinning shared deps at the root) prevent this class of bug entirely.

Reference: [pnpm workspaces](https://pnpm.io/workspaces), [pnpm catalogs](https://pnpm.io/catalogs)

### 2.3 Bun workspaces

**Beginner.** Also declared via the `workspaces` field in the root `package.json`, same shape as npm:

```jsonc
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": ["apps/*", "packages/*"]
}
```

```bash
bun install
```

Bun supports negative glob patterns to exclude subfolders:

```jsonc
{ "workspaces": ["packages/**", "!packages/**/test/**"] }
```

**Working Knowledge.**

```bash
# As of Bun 1.4: --filter works on add/remove/update, not just run
bun add lodash --filter web
bun remove lodash --filter web
bun update --filter "web..."      # web and its dependencies

# Run a script across all workspaces
bun run --filter '*' build
```

`workspace:*` works the same as pnpm's protocol for internal linking.

**Advanced — catalogs (Bun 1.4+).** Bun 1.4 added a `--catalog` flag mirroring pnpm's catalogs:

```bash
bun add react --catalog
```

This writes the version into the root catalog and sets `"react": "catalog:"` in the workspace's `package.json` — flag this as new: **older Bun tutorials (pre-1.4, before 2026-08-19) won't mention this**, and will instead show manual version pinning per package as the only option.

**Mastery — isolated linker / global virtual store (Bun 1.4+).** For large monorepos, `bunfig.toml` can opt into an isolated linker with a shared global store:

```toml
# bunfig.toml
[install]
linker = "isolated"
```

Packages extract once into Bun's global cache and are symlinked into each project's `node_modules/.bun/` store instead of being copied per project — Bun's own benchmarks report up to **7x faster** installs on a CI path with a warm cache and a wiped `node_modules`. This is the Bun analogue of pnpm's content-addressable store; before 1.4, Bun's default linker behaved more like npm's flat-hoisted model.

**Wrong vs. right:**

```bash
# Wrong (pre-1.4 habit): manually editing each package.json to bump a shared dep version
# apps/web/package.json, apps/admin/package.json, packages/ui/package.json — three manual edits

# Right (1.4+): use catalog once, every package references it
bun add typescript@5.9 --catalog
```

**Real Scenario:** A team on Bun evaluates whether to adopt it for a new monorepo and finds a blog post claiming "Bun doesn't support centralized version pinning like pnpm catalogs" — true as of Bun 1.3, false as of 1.4. This is exactly the kind of stale-blog-post trap the sourcing rules above exist to catch: always check the docs' version/date, not just search ranking.

Reference: [Bun workspaces docs](https://bun.sh/docs/install/workspaces), [Bun 1.4 release notes](https://bun.com/blog/bun-v1.4)

### 2.4 Hoisting & node_modules layout

**Working Knowledge.** "Hoisting" means shared dependencies get installed once at a common ancestor `node_modules` instead of duplicated in every package. The three tools differ in how strictly they do this:

```
npm (flat hoist)              pnpm (strict, symlinked)         bun (hoisted by default;
                                                                 isolated linker in 1.4+)

node_modules/                 node_modules/                    node_modules/
├── lodash/      ← hoisted    ├── .pnpm/                       ├── lodash/      ← hoisted
├── @myorg/ui -> ../packages/ │   ├── lodash@4.17.21/          ├── @myorg/ui -> ../packages/ui
│                 ui (symlink)│   └── react@19.0.0/            │
apps/web/node_modules/        ├── lodash -> .pnpm/lodash@.../  apps/web/node_modules/
  (usually empty, uses root)  └── @myorg -> ../packages         .bun/ -> global store (1.4+)
                               apps/web/node_modules/
                                 lodash -> ../../node_modules/.pnpm/lodash@.../
```

npm's flat hoist means a package can accidentally `require()` a dependency it never declared, just because it happens to be hoisted to root (a **phantom dependency** — see [Section 4](#4-dependency-management-at-scale)). pnpm's symlink structure to a content-addressable `.pnpm` store makes this fail loudly instead, because non-declared dependencies simply aren't reachable from a package's own `node_modules`. Bun's default linker is closer to npm's flat model; its opt-in isolated linker (1.4+) behaves more like pnpm's.

**Advanced.** This is *the* main practical reason teams choose pnpm over npm for larger monorepos: catching phantom dependencies at install time is cheaper than catching them when a package is later extracted/published and breaks because a dependency it silently relied on is no longer hoisted into its tree.

[Back to top](#table-of-contents)

---

## 3. Internal Package Linking

**Beginner.** An internal package (e.g., `packages/ui`) is referenced from an app (e.g., `apps/web`) exactly like an npm-registry dependency, just resolved locally:

```jsonc
// apps/web/package.json
{ "dependencies": { "@myorg/ui": "workspace:*" } }
```

`workspace:*` (pnpm/Bun) or a normal semver range that happens to match the local version (npm) tells the package manager to symlink rather than fetch from the registry.

**Working Knowledge — the `exports` field.** Internal packages should define `exports` in their own `package.json` to control what's importable and support both ESM and types cleanly:

```jsonc
// packages/ui/package.json
{
  "name": "@myorg/ui",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    },
    "./styles.css": "./dist/styles.css"
  }
}
```

**Wrong vs. right:**

```jsonc
// Wrong: no "exports", consumers reach into dist/ directly, or main points at src
{ "main": "src/index.ts" }
// apps/web breaks the moment @myorg/ui adds a build step, because src/index.ts
// isn't valid runtime JS without a transpile step web doesn't apply to node_modules.

// Right: exports points at built output; consumers only ever see the published surface
{ "exports": { ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" } } }
```

**Real Scenario:** `apps/web` imports `@myorg/ui` and it works fine in dev because the dev server transpiles everything including `node_modules` symlink targets. In production build, the bundler treats `node_modules` (even symlinked workspace packages) as pre-built and refuses to transpile TypeScript inside it, and the build fails with a syntax error deep in a `.ts` file. Fix: `packages/ui` needs its own build step (`tsc` or `tsup`) producing `dist/`, and `exports`/`main` must point at `dist/`, not `src/`. This is one of the most common "works in dev, breaks in prod" monorepo bugs — several Stack Overflow threads on "Cannot use import statement outside a module" in monorepos trace back to exactly this.

Reference: [Node.js package.json exports field](https://nodejs.org/api/packages.html#exports), [pnpm workspace protocol](https://pnpm.io/workspaces#workspace-protocol-workspace)

[Back to top](#table-of-contents)

---

## 4. Dependency Management at Scale

**Working Knowledge — phantom dependencies.** A phantom dependency is when code `require()`s or `import`s a package it never listed in its own `package.json`, but it happens to resolve anyway because some other package hoisted it into a shared `node_modules`.

```
Real Scenario:
packages/api declares only "express" as a dependency.
Code in packages/api does: import { z } from "zod";
zod isn't declared — but packages/worker (a sibling) depends on zod,
and npm's flat hoist put zod at the shared root node_modules.
packages/api "works" locally.

Six months later, packages/worker drops its zod dependency during a refactor.
packages/api now fails at runtime with "Cannot find module 'zod'" —
in production, with no code change to packages/api itself.
```

pnpm's strict symlink structure ([Section 2.4](#24-hoisting--node_modules-layout)) catches this at install/import time instead of months later in prod. If you're on npm and want similar strictness, add `zod` explicitly wherever it's actually imported — the fix is always "declare what you use," pnpm just enforces it automatically.

**Advanced — version drift.** As seen in [2.2](#22-pnpm-workspaces) and [2.3](#23-bun-workspaces), catalogs solve version drift for pnpm/Bun. For npm (no catalog feature as of npm 11), the common approach is [`syncpack`](https://github.com/JamieMason/syncpack) to lint/fix mismatched versions across `package.json` files, or simply pinning shared deps at the root and relying on hoisting.

**Mastery — peer dependency conflicts.** When two workspace packages require incompatible peer dependency ranges of the same package (e.g., one UI package built for React 18 peer range, another for React 19), the package manager either warns (npm, pnpm with default settings) or, with pnpm's stricter peer resolution, fails install outright. Resolve by aligning peer ranges across internal packages — treat peer dependency ranges of internal packages the same as you'd treat a public library's peer deps, since from the consuming app's point of view, they are one.

[Back to top](#table-of-contents)

---

## 5. Task Running & Turborepo

**Beginner.** Workspaces solve *dependency* management. They don't solve *task* management: running `npm run build --workspaces` on a 50-package repo rebuilds everything, every time, even packages nothing changed in. [Turborepo](https://turborepo.dev/) adds a **task graph** with caching on top of any of the three package managers above.

```bash
npm install turbo --save-dev -w .   # or: bun add -D turbo / pnpm add -D turbo -w
```

```jsonc
// turbo.json
{
  "$schema": "https://turborepo.dev/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": []
    },
    "lint": {
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

> Note the config key is `tasks`, not the older `pipeline` key from Turborepo v1 — flag this if you find older tutorials, since `pipeline` is deprecated in current versions. [Turborepo configuration reference](https://turborepo.dev/docs/reference/configuration)

**`dependsOn` syntax:**
- `"^build"` — run `build` in every dependency of this package first (dependency-graph order)
- `"build"` (no `^`) — run `build` in *this same package* first
- `"utils#build"` — depend on a specific task in a specific package

**Working Knowledge — running and filtering.**

```bash
turbo run build                       # build everything, respecting the task graph
turbo run build --filter=@myorg/web   # build web and (per dependsOn) its dependencies
turbo run build --filter=...ui        # build ui and everything that depends on it
turbo run build --filter="./packages/*"
turbo run build --filter="[main...my-feature]"   # only packages changed vs. main
turbo run build --dry                 # show what WOULD run, without running it
```

**Diagram — task graph with `^build`:**

```
packages/config  (no deps)
      │
      ▼ ^build
packages/ui      (depends on config)
      │
      ▼ ^build
apps/web         (depends on ui)

turbo run build → executes: config#build → ui#build → web#build (in that order)
```

**Advanced — caching.** Turborepo hashes each task's inputs (source files, lockfile, config, declared env vars) into a fingerprint. Same fingerprint → restore cached `outputs` and logs instead of re-running.

```bash
npx turbo login   # authenticate once
npx turbo link    # connect this repo to Vercel Remote Cache
```

```jsonc
// turbo.json — register env vars that should invalidate cache when changed
{
  "tasks": {
    "build": {
      "outputs": ["dist/**"],
      "env": ["API_URL"]
    }
  }
}
```

**Wrong vs. right:**

```jsonc
// Wrong: build reads process.env.API_URL but it's not declared —
// changing API_URL between environments produces a stale cache hit with the OLD value baked in
{ "tasks": { "build": { "outputs": ["dist/**"] } } }

// Right: declare it so a changed value invalidates the cache
{ "tasks": { "build": { "outputs": ["dist/**"], "env": ["API_URL"] } } }
```

**Real Scenario:** A team deploys a "successful" build to staging that's actually still running against the production API URL, because `API_URL` changed between the staging and production CI jobs but wasn't declared in `turbo.json`'s `env` list — Turborepo saw an identical source-code fingerprint, hit cache, and reused the previous build's output (with production's baked-in URL) instead of rebuilding. This exact "remote cache returned stale env-dependent output" pattern is a recurring theme in [Turborepo GitHub issues](https://github.com/vercel/turborepo/issues) and is the single most common Turborepo footgun — always declare every env var a build step reads.

Reference: [Turborepo running tasks](https://turborepo.dev/docs/crafting-your-repository/running-tasks), [Turborepo caching](https://turborepo.dev/docs/crafting-your-repository/caching), [Turborepo configuration](https://turborepo.dev/docs/reference/configuration)

[Back to top](#table-of-contents)

---

## 6. CI/CD Considerations

**Working Knowledge — affected-only CI.** Run only what changed relative to the base branch:

```bash
turbo run build test lint --filter="[origin/main...HEAD]"
```

This is the practical payoff of everything above: as the repo grows, a naive `npm run build --workspaces` in CI gets linearly slower; `--filter` with Turborepo caching keeps CI time roughly proportional to what actually changed.

**Working Knowledge — remote cache in CI.** Set `TURBO_TOKEN` and `TURBO_TEAM` (or self-hosted cache API equivalents) as CI secrets so CI runs share cache with local dev and with each other — a second CI run of an unchanged package should be a cache hit, not a rebuild.

**Advanced — versioning and publishing with Changesets.** For monorepos that publish internal packages to a registry (public npm or a private one), [Changesets](https://github.com/changesets/changesets) is the standard tool:

```bash
npx changeset            # contributor: describe what changed and the semver bump
npx changeset version     # maintainer/CI: consumes changesets, bumps versions, writes CHANGELOGs
npx changeset publish     # publish updated packages to the registry
```

> "In monorepo environments, Changesets handles an additional responsibility: it automatically bumps dependency versions for packages that depend on changed packages." — [Changesets intro docs](https://github.com/changesets/changesets/blob/main/docs/intro-to-using-changesets.md)

**Real Scenario:** `packages/ui` gets a bugfix and a changeset is added for it. `apps/web` depends on `@myorg/ui` via `workspace:*`. Running `changeset version` bumps `ui`'s version *and* automatically bumps the recorded dependency version in `web`'s `package.json` if `web`'s changeset config is set to track it — without this, teams end up manually hunting down every consumer of a bumped package, which doesn't scale past a handful of packages.

Reference: [Changesets docs](https://github.com/changesets/changesets)

[Back to top](#table-of-contents)

---

## 7. Project Layout Conventions

**Working Knowledge.** The `apps/` vs `packages/` split is a convention, not a tooling requirement — but it's near-universal because it separates *deployables* from *libraries*:

```
my-monorepo/
├── package.json              # workspaces field, root devDependencies (turbo, typescript, eslint)
├── turbo.json
├── pnpm-workspace.yaml        # (or workspaces field, if npm/Bun)
├── tsconfig.base.json         # shared compiler options, extended by each package
├── apps/
│   ├── web/                   # deployable: Next.js app
│   └── api/                   # deployable: server
└── packages/
    ├── ui/                    # shared React components
    ├── config/                # shared eslint/tsconfig/tailwind configs, published as packages
    └── tsconfig/               # e.g. @myorg/tsconfig, so apps do: "extends": "@myorg/tsconfig/base.json"
```

**Advanced — shared config as packages.** Treating `eslint-config`, `tsconfig`, and similar as internal packages (not copy-pasted files) means updating lint rules in one place propagates everywhere via the normal dependency graph, and Turborepo's `dependsOn` picks up config changes as cache-invalidating inputs automatically.

```jsonc
// apps/web/tsconfig.json
{ "extends": "@myorg/tsconfig/nextjs.json" }
```

[Back to top](#table-of-contents)

---

## 8. Troubleshooting & Common Gotchas

| Symptom | Likely Cause | Fix |
|---|---|---|
| "Cannot find module '@myorg/ui'" after clone | Forgot to run install at root, or workspace glob doesn't match the package's path | Run install from repo root; check `workspaces` / `pnpm-workspace.yaml` globs match actual folders |
| Works in dev, breaks in prod build | Internal package's `exports`/`main` points at `src/` instead of built `dist/` | Build the internal package first; point `exports` at compiled output ([Section 3](#3-internal-package-linking)) |
| Import resolves locally but fails in prod, no code changed | Phantom dependency — package used something hoisted from a sibling, never declared it | Add the dependency explicitly to the package that imports it ([Section 4](#4-dependency-management-at-scale)) |
| "Invalid hook call" / duplicate library instance errors | Two workspace packages resolved different versions of a peer-like dependency (e.g. React) | Pin the version via catalog (pnpm/Bun 1.4+) or align versions manually ([Section 2.2](#22-pnpm-workspaces)) |
| Turborepo cache hit but output is stale/wrong for the environment | An env var the build reads isn't declared in `turbo.json`'s `env` list | Add it to `env` (or `globalEnv`) so it participates in the cache key ([Section 5](#5-task-running--turborepo)) |
| CI takes as long as if nothing was cached | `TURBO_TOKEN`/`TURBO_TEAM` not set in CI, or filter not scoped to changed packages | Configure remote cache secrets; use `--filter="[origin/main...HEAD]"` |
| Old tutorial's `turbo.json` uses `pipeline` and doesn't work | Following a Turborepo v1-era guide | Use `tasks` — see [Section 5](#5-task-running--turborepo) |

[Back to top](#table-of-contents)

---

## 9. Suggested Learning/Reference Order

1. [Section 2](#2-workspace-fundamentals) — pick **one** package manager to actually build with first (pnpm is the most common real-world default; Bun if you want the 1.4 catalog/isolated-linker speed story). Get a two-package workspace installing and linking correctly before anything else.
2. [Section 3](#3-internal-package-linking) — build one internal package with a proper `exports` field and consume it from an app. Hit the "works in dev, breaks in prod" bug on purpose once, so you recognize it later.
3. [Section 4](#4-dependency-management-at-scale) — deliberately create a phantom dependency, watch it work, then watch it break when the hoisting source is removed. This is the fastest way to internalize why strict resolution matters.
4. [Section 5](#5-task-running--turborepo) — add Turborepo once you have 3+ packages. Build the `turbo.json`, verify a second identical run is a cache hit, then break the cache-hit-with-stale-env-var scenario on purpose.
5. [Section 6](#6-cicd-considerations) — wire up affected-only CI and (if publishing packages) Changesets.
6. [Section 7](#7-project-layout-conventions) — retrofit shared config packages once you're tired of copy-pasting `tsconfig.json`.
7. Keep [Section 8](#8-troubleshooting--common-gotchas) open as a reference while you work — most real monorepo pain shows up as one of those seven rows.

[Back to top](#table-of-contents)

---

## 10. Quick Self-Check

1. What's the difference between what a workspace feature solves and what Turborepo solves?
2. Why does `workspace:*` fail to resolve to anything but a local package, and why is that useful?
3. What is a phantom dependency, and which of the three package managers is structurally resistant to it by default?
4. Why can an internal package "work" in dev but fail in a production bundle build? What field on the package's `package.json` is usually the fix?
5. In `turbo.json`, what's the difference between `"dependsOn": ["build"]` and `"dependsOn": ["^build"]`?
6. Why must an env var a build reads be declared in `turbo.json`'s `env` list, even though the build works fine without declaring it?
7. What does `pnpm-workspace.yaml`'s (or Bun 1.4's) `catalog:` feature solve that plain semver ranges in each `package.json` don't?
8. What command sequence would you run to release a version bump for an internal package that another workspace package depends on, using Changesets?

[Back to top](#table-of-contents)
