# TypeScript Config Mastery

## About This Document

**Framing:** Goal-driven — optimized for the concrete configs, flags, and commands you'll actually touch across real project types, not a tour of every TSConfig option that exists.

**Confirmed scope:**
- Single-app fullstack (Node backend + React/Next frontend)
- Monorepo (Turborepo/Nx/pnpm workspaces) — project references, composite builds, shared base configs
- Published npm library — declaration output, dual ESM/CJS, `exports` map interplay
- Editor/tooling config — ESLint (`typescript-eslint` project service), path aliases, IDE/tsserver behavior
- Vue/Nuxt and Bun runtime specifics folded into the framework-specific section

**Depth tiers used:** Beginner → Working Knowledge → Advanced → Mastery, per section, collapsed where a tier would be trivial or redundant for that concept.

**Relationship to other docs:** This is a standalone deep-dive on TypeScript *configuration* specifically. [typescript-for-fullstack-mastery.md](typescript-for-fullstack-mastery.md) covers the language itself and has a short tsconfig section (§13) that now just points here instead of duplicating content.

**Update policy:** If asked to update this doc later, preserve this structure — per-section tiers, "wrong vs. right" snippets for gotchas, Real Scenario callouts, official-docs-first sourcing, and the cheat-sheet/self-check ending.

---

## Table of Contents

1. [How Config Fits Together](#1-how-config-fits-together)
2. [Anatomy of tsconfig.json](#2-anatomy-of-tsconfigjson)
3. [Core Compiler Options Deep Dive](#3-core-compiler-options-deep-dive)
4. [Module Resolution & Path Aliases](#4-module-resolution--path-aliases)
5. [Project References & Composite Builds](#5-project-references--composite-builds)
6. [Config Composition (extends)](#6-config-composition-extends)
7. [Monorepo Tooling](#7-monorepo-tooling)
8. [Framework & Runtime-Specific Configs](#8-framework--runtime-specific-configs)
9. [Publishing an npm Library](#9-publishing-an-npm-library)
10. [Editor & Lint Tooling Integration](#10-editor--lint-tooling-integration)
11. [Performance at Scale](#11-performance-at-scale)
12. [Strictness Migration Strategy](#12-strictness-migration-strategy)
13. [Common Pitfalls Cheat Sheet](#13-common-pitfalls-cheat-sheet)
14. [Suggested Learning/Reference Order](#14-suggested-learningreference-order)
15. [Quick Self-Check](#15-quick-self-check)

---

## 1. How Config Fits Together

### Working Knowledge

`tsconfig.json` isn't read by one tool — it's read differently by (at least) three consumers, and most "why doesn't this work" confusion comes from forgetting one of them exists:

```
┌─────────────────────────────────────────────────────────────┐
│                      tsconfig.json                            │
└───────┬─────────────────────┬─────────────────────┬──────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌──────────────────┐   ┌──────────────────────┐
│  tsc (CLI)     │   │  Editor/tsserver  │   │  Bundler/runtime      │
│  type-checks + │   │  powers autocomplete,│  (Vite/webpack/esbuild│
│  optionally    │   │  hover, go-to-def, │   │  or Node/Bun itself)  │
│  emits JS      │   │  red squiggles     │   │  actually resolves &  │
│                │   │                    │   │  runs the imports      │
└───────────────┘   └──────────────────┘   └──────────────────────┘
```

- **`tsc`** reads `compilerOptions` to type-check and (if `noEmit` isn't set) emit JS/`.d.ts` files.
- **tsserver** (the language service behind VS Code, JetBrains, etc.) reads the *same* file to power editor feedback — this is why a bad `include`/`exclude` can make files "invisible" to autocomplete even though the app still builds.
- **The bundler or runtime** does not read `tsconfig.json` for module resolution at all in most stacks — it has its own resolution algorithm (Vite's, webpack's, Node's). `moduleResolution` in tsconfig only tells *TypeScript's checker* which algorithm to *simulate*, so its answer needs to match what will actually execute the code, or type-checking will pass while node — or the bundler — fails to find the module.

This mismatch — TS says "this import is fine," runtime says "cannot find module" — is the single most common category of tsconfig-related bug, and nearly every option below exists to keep those three consumers in sync.

[⤴ back to top](#table-of-contents)

---

## 2. Anatomy of tsconfig.json

### Beginner

```jsonc
{
  "compilerOptions": { /* how TS checks and (optionally) emits */ },
  "include": ["src"],           // which files are "in" this project
  "exclude": ["node_modules"],  // subtracted from include (has sane defaults)
  "extends": "./tsconfig.base.json", // inherit another config
  "references": [{ "path": "../shared" }] // links to other tsconfig projects
}
```

- `include`/`exclude` are glob patterns; `exclude` defaults to `node_modules`, `bower_components`, and the `outDir`.
- `files` is a third, rarely-used option: an explicit array of exact file paths (no globs) — used for tiny configs or the "solution" tsconfig pattern in §5.
- If neither `files` nor `include` is present, TS defaults to including every `.ts`/`.tsx` file in the directory tree.

### Working Knowledge

A field ordering/precedence gotcha worth internalizing early: `exclude` only removes files that would otherwise be matched by `include`; it cannot exclude a file that's explicitly listed in `files`. And an empty `include: []` combined with `references` is the deliberate "solution file" pattern used to fan out to sub-projects (§5) — it isn't a mistake, even though it looks like one at a glance.

Official reference: [What is a tsconfig.json](https://www.typescriptlang.org/docs/handbook/tsconfig-json.html)

[⤴ back to top](#table-of-contents)

---

## 3. Core Compiler Options Deep Dive

### Working Knowledge — the options you'll set in nearly every project

| Option | What it controls | Typical value |
|---|---|---|
| `target` | JS language version emitted, and which lib types are ambient | `"ES2022"`+ (or `"ESNext"` if a bundler down-levels for you) |
| `lib` | Which built-in type declarations are available (`DOM`, `ES2022`, etc.) | Inferred from `target`, override for `DOM` on frontend or omit it on backend |
| `module` | Module output format | `"ESNext"`/`"Preserve"` for bundlers, `"NodeNext"` for Node-executed code |
| `moduleResolution` | Which resolution *algorithm* the checker simulates | `"bundler"` or `"nodenext"` — see §4 |
| `strict` | Umbrella for ~8 strictness checks | `true`, always, on any project past initial scaffolding |
| `esModuleInterop` | Allows `import x from 'cjs-pkg'` for CJS-only packages | `true` |
| `skipLibCheck` | Skip type-checking of `.d.ts` files (incl. `node_modules`) | `true` — near-universal, see Mastery below |
| `noEmit` | TS only type-checks; something else (bundler) does the emit | `true` whenever a bundler owns the build |

A minimal, current config for an app where a bundler does the actual build:

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "noEmit": true,
    "jsx": "react-jsx"
  },
  "include": ["src"]
}
```

### Advanced

**`strict` is a bundle, not one flag.** It turns on `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `alwaysStrict`, and `useUnknownInCatchVariables`. You can enable `strict` and selectively disable one sub-flag during a gradual migration (§12), but per the [official strict reference](https://www.typescriptlang.org/tsconfig/strict.html), the recommended direction on any project past initial setup is always toward *more* strictness, never less.

**`verbatimModuleSyntax`** (stable since TS 5.0) replaces the older `isolatedModules`/`importsNotUsedAsValues`/`preserveValueImports` combo with one simpler rule: any import/export *without* a `type` modifier is preserved verbatim in the output exactly as written; anything *with* `type` is elided entirely. This forces you to be explicit about type-only imports:

```ts
// ✗ Wrong under verbatimModuleSyntax — ambiguous whether User is a type or a value,
// and if it's type-only this throws at emit time once the flag is on
import { User } from "./models";

// ✓ Right — explicit type-only import, safely elided from JS output
import type { User } from "./models";
import { createUser } from "./models"; // real runtime import stays separate
```

It also implicitly enables `isolatedModules`, which forbids a handful of TS features (like `const enum`) that single-file transpilers (esbuild, SWC, Babel) can't handle correctly without full-program knowledge. If your build uses one of those transpilers instead of `tsc` to emit JS — which is the common case with Vite/Next — turn this on; it converts a class of "works with tsc, breaks with esbuild" bugs into compile-time errors. ([Official reference](https://www.typescriptlang.org/tsconfig/verbatimModuleSyntax.html))

### Mastery

**`skipLibCheck: true`** is a pragmatic default nearly every real project should keep on. Without it, TS re-type-checks every `.d.ts` file pulled in from `node_modules`, including internal inconsistencies between two libraries' bundled type definitions that have nothing to do with your code. It's a compile-time performance and noise tradeoff, not a strictness one — your own source is still fully checked either way.

**`isolatedDeclarations`** (stable since TS 5.5) is the newest option worth knowing exists even if you don't turn it on yet: it requires every exported value to carry enough explicit type annotation that a `.d.ts` file could be generated by looking at *that file alone*, with no cross-file type inference. This is what lets fast, non-type-checking tools (like esbuild or the `isolatedDeclarations`-aware transpilers) generate declaration files without running the full TS compiler — relevant if your build pipeline is trying to drop `tsc` from the hot path entirely (see §9). ([Official reference](https://www.typescriptlang.org/tsconfig/isolatedDeclarations.html))

**Real Scenario:** A team ships a Vite + React app, sees clean `tsc` output in CI, then gets a runtime crash in production: `const enum` values compiled away to `undefined`. Cause: Vite uses esbuild to strip types file-by-file, and esbuild doesn't understand `const enum` cross-file inlining the way `tsc`'s full-program checker does — `isolatedModules` (or `verbatimModuleSyntax`) would have caught this at type-check time instead of at runtime, because it's specifically the flag that simulates "what would a single-file transpiler be able to do here."

[⤴ back to top](#table-of-contents)

---

## 4. Module Resolution & Path Aliases

### Working Knowledge

`moduleResolution` values, per the [official reference](https://www.typescriptlang.org/tsconfig/moduleResolution.html):

| Value | Use when |
|---|---|
| `"bundler"` | Vite, webpack, Next.js, esbuild does the real resolution; pairs with `noEmit: true`. Allows extensionless relative imports, understands package.json `exports`/`imports`. |
| `"nodenext"` | Code Node executes directly — a published library, or a Node backend with no bundler. Models Node's real ESM resolution algorithm. |
| `"node10"` (formerly `"node"`) | Legacy CommonJS-only, pre-ESM Node. Rarely correct to choose today. |
| `"classic"` | Deprecated, pre-1.6 TS. Never choose this. |

```ts
// nodenext — extension required even though the source file is utils.ts,
// because Node's real ESM loader needs the extension to resolve the .js output
import { helper } from "./utils.js";

// bundler resolution — extensionless works because the bundler resolves it
import { helper } from "./utils";
```

This exact confusion — "why does TS want a `.js` extension on a `.ts` import?" — is one of the most common TypeScript questions once teams adopt ESM; see the [module resolution theory doc](https://www.typescriptlang.org/docs/handbook/modules/theory.html#module-resolution) for the full algorithm.

### Advanced — path aliases don't survive the compiler

`compilerOptions.paths` (with `baseUrl`) lets you write `import { Button } from "@ui/Button"` instead of `../../../ui/Button`. Critically: **`paths` is a checker-only, editor-only feature.** TypeScript rewrites nothing in emitted JS — `tsc` does not touch the specifier text.

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@ui/*": ["src/ui/*"] }
  }
}
```

```ts
// ✗ Wrong assumption: "TS resolved it while type-checking, so it'll run fine"
import { Button } from "@ui/Button";
// Compiled output still says `require("@ui/Button")` or `import ... from "@ui/Button"`
// verbatim — and Node/the runtime has no idea what "@ui" means. Crashes at runtime
// with "Cannot find module '@ui/Button'" even though `tsc` reported zero errors.
```

**Fixes, by pipeline:**
- **Bundler-driven apps (Vite/webpack/Next):** mirror the same aliases in the bundler's own config (`vite.config.ts` `resolve.alias`, webpack `resolve.alias`) — the bundler does the real resolution, tsconfig `paths` only keeps the type-checker and editor in sync with what the bundler will do.
- **Plain Node output (`tsc`-compiled, no bundler):** use a runtime resolver like `tsconfig-paths` (`node -r tsconfig-paths/register dist/index.js`) or a build-time rewriter like `tsc-alias` that rewrites alias specifiers to relative paths in the emitted JS. `tsc-alias` avoids per-request runtime resolution cost, which matters for cold-start-sensitive serverless functions; `tsconfig-paths` is simpler to wire up when startup time doesn't matter.

Keeping the two configs (tsconfig `paths` and the bundler/runtime resolver) in sync is a real maintenance burden — a common pattern is a pre-commit or CI check that diffs them, since nothing else will catch drift until something fails at runtime.

**Real Scenario:** A Next.js API route imports a shared util via `@lib/db`. It works locally (Next's bundler resolves the alias) and passes `tsc --noEmit` in CI. It's then also imported by a standalone Node cron script that's compiled with plain `tsc` and run with `node dist/cron.js` — no bundler in that path — and it crashes immediately with `Cannot find module '@lib/db'`, because nothing in that execution path ever taught the runtime what `@lib` means.

[⤴ back to top](#table-of-contents)

---

## 5. Project References & Composite Builds

### Working Knowledge

Project references let you split a codebase into smaller TS "projects" that reference each other, so `tsc` can build/check only what changed and understand dependency order. This is the backbone of most well-structured monorepos.

```
apps/api/tsconfig.json  ──references──▶  packages/shared/tsconfig.json
apps/web/tsconfig.json  ──references──▶  packages/shared/tsconfig.json
```

A referenced project must set `"composite": true`, which:
- Forces `declaration: true` (composite projects must emit `.d.ts` so dependents can consume them)
- Defaults `rootDir` to the tsconfig's own directory
- Requires all implementation files to be captured by `include`/`files`

```jsonc
// packages/shared/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "outDir": "./dist"
  },
  "include": ["src"]
}
```

```jsonc
// apps/api/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": { "outDir": "./dist" },
  "include": ["src"],
  "references": [{ "path": "../../packages/shared" }]
}
```

Build with `tsc --build` (or `-b`), not plain `tsc` — plain `tsc` does **not** build referenced dependencies automatically:

```bash
tsc -b                # build using tsconfig.json in cwd, respecting the reference graph
tsc -b --watch         # incremental watch mode across the whole graph
tsc -b --clean         # delete build outputs
tsc -b --force         # rebuild everything, ignoring .tsbuildinfo timestamps
```

Official handbook: [Project References](https://www.typescriptlang.org/docs/handbook/project-references.html)

### Advanced

The **"solution" tsconfig pattern**: a root `tsconfig.json` that contains only `files: []` and a `references` array fanning out to every leaf project — nothing else. Editors and `tsc -b` use it as an entry point without it being a "real" project itself:

```jsonc
// root tsconfig.json — a solution file, not a buildable project
{
  "files": [],
  "references": [
    { "path": "./packages/shared" },
    { "path": "./apps/api" },
    { "path": "./apps/web" }
  ]
}
```

Gotchas worth knowing before you hit them:
- `tsc -b` effectively forces `noEmitOnError` across every referenced project — a type error in a dependency blocks emission for everything downstream, even if that dependency built successfully before.
- Build outputs (`.d.ts` + `.tsbuildinfo`) are what dependents actually consume. If they're `.gitignore`d (normal), a fresh clone needs a build pass before the editor stops showing spurious "cannot find module" errors in dependent packages — VS Code's in-memory project reference resolution (TS 3.7+) usually papers over this locally, but CI and other editors may not.
- After a `git` checkout/rebase, file timestamps can look "unchanged" to the incremental build graph even when content changed — if you see stale-looking build output, `tsc -b --force` is the fix.

### Mastery

The honest tradeoff (raised directly by [Nx's own writeup](https://nx.dev/blog/typescript-project-references)): project references give real incremental-build wins and better editor isolation, but the `references` array in every tsconfig has to be kept in sync with actual import dependencies by hand — add an import across a package boundary without adding the matching reference, and you get a confusing "is not listed within the file list of project" error instead of a normal import error. This manual bookkeeping is exactly what monorepo tools like Nx and Turborepo automate (§7) — most teams adopting project references at scale do so *through* one of those tools rather than hand-maintaining the graph.

[⤴ back to top](#table-of-contents)

---

## 6. Config Composition (extends)

### Working Knowledge

`extends` lets a config inherit from a base file; the extending file's fields are merged on top (arrays like `include` are replaced wholesale, not merged, with a couple of documented exceptions).

```jsonc
// tsconfig.base.json — shared across every package
{
  "compilerOptions": {
    "target": "ES2022",
    "strict": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "composite": true
  }
}
```

```jsonc
// packages/api/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "nodenext",
    "outDir": "./dist"
  },
  "include": ["src"]
}
```

### Advanced

`extends` can point at an npm package, not just a relative file — this is how community base configs (the [`@tsconfig/*` collection](https://www.npmjs.com/package/@tsconfig/bun), `@tsconfig/node20`, `@tsconfig/strictest`, etc.) and framework-generated configs (Nuxt's `.nuxt/tsconfig.json`, §8) work: `"extends": "@tsconfig/node20/tsconfig.json"`. Multiple `extends` targets (an array, TS 5.0+) are merged left-to-right, with the extending file's own fields winning last.

A frequent mistake: putting per-package *overrides that matter* (like `paths`) only in the base, then having a leaf config redefine `compilerOptions.paths` for its own needs — since object fields like `paths` are replaced, not deep-merged, the leaf's `paths` silently loses everything the base defined, rather than adding to it.

[⤴ back to top](#table-of-contents)

---

## 7. Monorepo Tooling

### Working Knowledge

Project references (§5) are the TypeScript-native mechanism; Turborepo, Nx, and pnpm workspaces each layer conventions on top rather than replacing it.

**pnpm workspaces** — resolves package installs (`workspace:*` protocol) but has no opinion on tsconfig; you still wire up `extends`/`references` yourself. It's the substrate the other two tools sit on.

**Nx** — per the [Nx TypeScript project references guide](https://nx.dev/blog/typescript-project-references), the recommended shape is a root `tsconfig.json` with `composite: true` and shared `paths`, and each package extending it while declaring its own `references`:

```jsonc
// packages/is-even/tsconfig.json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": { "target": "esnext", "module": "esnext" },
  "references": [{ "path": "../is-odd" }]
}
```

Nx generates and keeps these references in sync via its own graph analysis (`nx sync` in recent versions), which is the practical answer to the manual-bookkeeping tradeoff noted in §5.

**Turborepo** doesn't touch tsconfig directly at all — it orchestrates *task running* (`turbo run build --filter=...`) and caching around whatever build commands (including `tsc -b`) you define per package. Its relevant config is `turbo.json`, not tsconfig; the TypeScript-level setup (composite, references, shared base config) is the same manual pattern from §5/§6 regardless of whether Turborepo sits on top.

### Advanced

**Real Scenario:** A pnpm + Turborepo monorepo has `packages/ui` importing `packages/tokens`. Turborepo's task graph correctly runs `tokens#build` before `ui#build` (declared via `turbo.json` `dependsOn: ["^build"]`), so the *build* succeeds. But the editor in `packages/ui` still shows type errors on imports from `@repo/tokens`, because nothing added `{ "path": "../tokens" }` to `ui/tsconfig.json`'s `references` — Turborepo's task graph and TypeScript's project-reference graph are two independent things that happen to usually mirror each other, and only one of them was updated.

[⤴ back to top](#table-of-contents)

---

## 8. Framework & Runtime-Specific Configs

### Working Knowledge

**Next.js** — generates and partially manages `tsconfig.json` on first run; expects `moduleResolution: "bundler"` (or `"node"` on older setups) since its own bundler (webpack/Turbopack) does resolution, and requires its custom `plugins: [{ "name": "next" }]` entry for the app-router type-checking features.

**Vite + React** — `moduleResolution: "bundler"`, `noEmit: true` (esbuild/Rolldown does the actual JS emission, `tsc` only type-checks), `verbatimModuleSyntax: true` recommended since Vite's dev server transpiles file-by-file.

**Nuxt** (v3/v4) — per the [official Nuxt tsconfig docs](https://nuxt.com/docs/4.x/directory-structure/tsconfig): Nuxt auto-generates `.nuxt/tsconfig.json` with resolved aliases (`~/`, `#imports`, `#build/*`) and sensible defaults. Your project's own `tsconfig.json` should just extend it:

```jsonc
// tsconfig.json
{ "extends": "./.nuxt/tsconfig.json" }
```

Two things the docs explicitly call out:
- **Don't hand-edit `.nuxt/tsconfig.json`** — it's regenerated on every dev/build run; configure Nuxt-level TS behavior via `nuxt.config.ts`'s `typescript` key instead.
- **`compilerOptions.paths` you define in your own `tsconfig.json` *replaces*, not merges with**, the aliases Nuxt generated — since `extends` doesn't deep-merge objects (§6), overriding `paths` yourself silently breaks `~/`-style imports.
- Nuxt 4 additionally splits generated config into `.nuxt/tsconfig.app.json` / `.nuxt/tsconfig.server.json` using project references, for better app-vs-server type isolation — worth checking which your Nuxt version produces before assuming a single generated file.

**Bun** — per [Bun's official TypeScript docs](https://bun.com/docs/typescript), `bun init` scaffolds the recommended config; the shape matters because Bun's runtime resolution differs from Node's in ways TS needs to be told about explicitly:

```jsonc
{
  "compilerOptions": {
    "lib": ["ESNext"],
    "target": "ESNext",
    "module": "Preserve",
    "moduleDetection": "force",
    "jsx": "react-jsx",
    "allowJs": true,
    "types": ["bun"],
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "noEmit": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true
  }
}
```

- `"types": ["bun"]` (needs `bun add -d @types/bun`) is what gives you typed access to the `Bun` global — omit it and `Bun.serve`, `Bun.file`, etc. are untyped or missing.
- `allowImportingTsExtensions: true` + `moduleResolution: "bundler"` is what lets Bun-run code `import` a `.ts` file directly, which plain Node-targeted TS configs reject.
- `module: "Preserve"` leaves your import/export syntax untouched in output, matching Bun's own transpiler behavior rather than TS rewriting to CJS or a fixed ESM target.

**Node backend (no bundler)** — `moduleResolution: "nodenext"` paired with `module: "NodeNext"`, explicit `.js` extensions on relative imports (§4), `declaration`/`sourceMap` if you publish or debug the compiled output.

[⤴ back to top](#table-of-contents)

---

## 9. Publishing an npm Library

### Advanced

Library tsconfig has a different center of gravity than app tsconfig: the output is consumed by *someone else's* toolchain, not run directly by yours, so `declaration`, `moduleResolution: "nodenext"`, and the package.json `exports` map all have to agree with each other.

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "nodenext",
    "declaration": true,
    "declarationMap": true,
    "outDir": "./dist",
    "strict": true
  },
  "include": ["src"]
}
```

```jsonc
// package.json — exports map has to match what tsconfig actually emits
{
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  }
}
```

### Mastery

Shipping **both** ESM and CJS builds from one source tree means running `tsc` twice with different `module`/`outDir` per target (or delegating to a bundler purpose-built for this, like `tsup`), because a single `tsconfig.json` can only pick one `module` value. The `exports` map's condition keys (`import` vs `require`) then route consumers to the matching build — get this wrong (e.g., point `require` at an ESM-only file) and CJS consumers get a runtime `ERR_REQUIRE_ESM` that never shows up during your own `tsc` type-check, since the checker isn't the thing enforcing the `exports` contract at runtime.

`isolatedDeclarations` (§3) is increasingly relevant here: libraries with strict export annotations can generate `.d.ts` via a fast, non-type-checking tool instead of a full `tsc` pass, which matters once a library's build time is dominated by declaration emission rather than JS emission.

[⤴ back to top](#table-of-contents)

---

## 10. Editor & Lint Tooling Integration

### Working Knowledge

`typescript-eslint` needs to know which tsconfig(s) to use for *typed* linting (rules that need type information, like `no-floating-promises`). The modern (v8+) flat-config mechanism is `parserOptions.projectService`, which replaced the older `parserOptions.project: "./tsconfig.json"` pattern:

```js
// eslint.config.js
export default [
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
  },
];
```

`projectService` auto-discovers the closest tsconfig.json per file (like `project: true` did) and — unlike the old mechanism — can also type-lint files that aren't captured by any tsconfig's `include` (e.g. `eslint.config.js` itself), removing the old need for a separate `tsconfig.eslint.json` just to satisfy the linter. ([typescript-eslint project-service announcement](https://typescript-eslint.io/blog/project-service/))

### Advanced

**Real Scenario:** A file is excluded from `tsconfig.json`'s `include` (e.g. a script under `scripts/`) but still imported somewhere. Under the old `parserOptions.project`, ESLint would throw `Parsing error: ... was not found by the project service` for that file — a frequent Stack Overflow question. `projectService: true` resolves most of these automatically; if you still hit it, the fix is either adding the file to `include` or to `projectService`'s `allowDefaultProject` list for genuinely tsconfig-external files, not disabling typed linting wholesale.

[⤴ back to top](#table-of-contents)

---

## 11. Performance at Scale

### Advanced

- **`incremental: true`** (implied by `composite`) writes a `.tsbuildinfo` file recording per-file state, so a subsequent `tsc`/`tsc -b` run only re-checks what changed. Delete the `.tsbuildinfo` (or `tsc -b --force`) if incremental state ever looks wrong after external changes like a rebase.
- **`skipLibCheck`** (§3) is the single biggest lever for reducing check time in projects with many dependencies — re-type-checking `node_modules`' own `.d.ts` files is pure overhead in the common case.
- **Project references** (§5) let `tsc -b` skip re-checking packages whose inputs haven't changed, which is where the real wall-clock win comes from in large monorepos — Nx's own numbers report memory usage dropping from ~3GB to under 1GB on a large graph when references are used correctly instead of one flat project.
- **`tsc -b --watch`** across a reference graph is markedly more efficient than running separate `tsc --watch` processes per package, since it shares the dependency graph instead of each watcher independently re-resolving imports.

[⤴ back to top](#table-of-contents)

---

## 12. Strictness Migration Strategy

### Advanced

Turning on `strict: true` on an existing, previously-loose codebase all at once usually produces an unmanageable wall of errors. The supported gradual path:

1. Turn on `strict: true`, then immediately re-disable the specific sub-flags causing the most noise (commonly `strictNullChecks` and `noImplicitAny` first) so the umbrella flag's *default direction* is captured even while you're not compliant yet.
2. Fix one sub-flag at a time, re-enabling it once its errors are clear — `noImplicitAny` first (forces explicit typing at boundaries), then `strictNullChecks` (the highest-value, highest-effort one), then the rest.
3. Never move a *disabled* flag list in the other direction (don't turn off a flag that's currently on) as a way to unblock a build — that's the officially discouraged direction per the [strict reference](https://www.typescriptlang.org/tsconfig/strict.html).

```jsonc
// Migration-in-progress config — captures the target while still being buildable today
{
  "compilerOptions": {
    "strict": true,
    "strictNullChecks": false,   // TODO: burn down, tracked in TICKET-123
    "noImplicitAny": false       // TODO: burn down, tracked in TICKET-124
  }
}
```

[⤴ back to top](#table-of-contents)

---

## 13. Common Pitfalls Cheat Sheet

| Symptom | Likely cause | Fix |
|---|---|---|
| `tsc` passes, runtime throws `Cannot find module '@x/y'` | `paths` alias not mirrored to the bundler/runtime | Add matching alias in bundler config, or use `tsc-alias`/`tsconfig-paths` (§4) |
| `Cannot find module './utils.js'` even though the file is `utils.ts` | `moduleResolution: nodenext` requires real extensions in relative imports | Import with `.js`, or switch to `moduleResolution: "bundler"` if a bundler owns resolution |
| Editor shows type errors that CI doesn't (or vice versa) | File outside `include`, or stale `.tsbuildinfo`/project-reference build outputs | Check `include`; `tsc -b --force` to rebuild references |
| `is not listed within the file list of project` | Cross-package import added without a matching `references` entry | Add `{ "path": "../other-package" }` to the importing package's tsconfig |
| Nuxt `~/` or `#imports` aliases stop resolving | Your own `tsconfig.json` redefined `compilerOptions.paths`, overwriting `.nuxt/tsconfig.json`'s (extends doesn't deep-merge) | Don't set `paths` yourself; configure via `nuxt.config.ts` instead |
| `const enum` value is `undefined` at runtime in a Vite/esbuild build | Single-file transpiler can't do the cross-file inlining `tsc` does | Enable `isolatedModules`/`verbatimModuleSyntax` to catch this at check time |
| CJS consumer gets `ERR_REQUIRE_ESM` from a published package | `package.json` `exports` condition points `require` at an ESM-only build | Ship a real dual build, or fix the `exports` map's `require` condition |
| ESLint: `... was not found by the project service` | File not covered by any tsconfig `include` | `projectService: true` in flat config; add file to `include` if still failing |

[⤴ back to top](#table-of-contents)

---

## 14. Suggested Learning/Reference Order

1. §1–§3 — get the mental model (three consumers) and the everyday compiler options solid first.
2. §4 — path aliases, because this is where "it works on my machine" bugs come from most often.
3. §8 — whichever framework/runtime row matches what you're actually building right now (Next, Vite+React, Nuxt, Bun, or plain Node).
4. §5–§7 — once you're touching more than one package: project references, then how your specific monorepo tool (Nx/Turborepo/pnpm) layers on top.
5. §9–§10 — only when actually publishing a library or fighting a linter config.
6. §11–§12 — revisit once build times or strictness rollout become the actual bottleneck, not before.

[⤴ back to top](#table-of-contents)

---

## 15. Quick Self-Check

- Why can `tsc --noEmit` succeed while `node dist/index.js` fails with `Cannot find module`? Which of the three consumers in §1 disagreed with which?
- When would you choose `moduleResolution: "bundler"` over `"nodenext"`, and what's the concrete symptom of picking the wrong one?
- What does `composite: true` force to also be true, and why does that requirement exist?
- Why does `tsc -b` need `--force` after some `git` operations but not others?
- What's wrong with overriding `compilerOptions.paths` in a config that `extends` another config that also sets `paths`?
- Why does `verbatimModuleSyntax` matter more for a Vite project than for a project compiled purely with `tsc`?
- In a Turborepo + pnpm monorepo, why can the build succeed while the editor still shows cross-package type errors?

[⤴ back to top](#table-of-contents)
