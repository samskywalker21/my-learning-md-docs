# Bun — Package Manager (Parts 4–5)

`bun install`, the lockfile, the two `node_modules` layouts, and the monorepo surface. This is the half of Bun most likely to change how a real project behaves, and the half where the defaults differ most sharply from npm.

> **Spec:** this doc follows the shared spec in [`bun-mastery-guide.md`](./bun-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Mastery tiers (collapsed where a tier would be filler, always stated), **Try It** exercises with expected output, wrong-vs-right pairs, official docs cited inline. Written against **Bun v1.4.2**, verified **September 6, 2026**.
>
> **Prerequisite:** [Part 2 — Modules & Resolution](./bun-runtime-foundations.md#part-2--modules--resolution), because a `node_modules` layout is only interesting in terms of what the resolver does with it.

---

## Table of Contents

- [Part 4 — `bun install`](#part-4--bun-install)
  - [Beginner: install, add, remove](#p4-beginner)
  - [Working Knowledge: the lockfile, CI, and the cache](#p4-working)
  - [Advanced: lifecycle scripts and the trust model](#p4-advanced)
  - [Advanced: hoisted vs. isolated linkers](#p4-linkers)
  - [Mastery: resolution, peers, and supply-chain controls](#p4-mastery)
  - [Part 4 cheat sheet](#part-4-cheat-sheet)
- [Part 5 — Monorepos & the Rest of `bun pm`](#part-5--monorepos--the-rest-of-bun-pm)
  - [Beginner: a workspace in five lines](#p5-beginner)
  - [Working Knowledge: filtering and catalogs](#p5-working)
  - [Advanced: `bunx`, auditing, and housekeeping](#p5-advanced)
  - [Mastery: migrating an existing repo](#p5-mastery)
  - [Part 5 cheat sheet](#part-5-cheat-sheet)

---

## Part 4 — `bun install`

<a id="p4-beginner"></a>

### Beginner: install, add, remove

```bash
bun install                 # install everything in package.json
bun add zod                 # add a dependency
bun add -d typescript       # add a devDependency  (also: --dev)
bun add -g @biomejs/biome   # install a global CLI
bun remove zod              # remove it
bun update                  # update within your declared ranges
```

`bun install` is a drop-in for `npm install` in the sense that it reads the same `package.json` and produces a `node_modules/` that Node itself can use. The docs put it at roughly 25× npm's speed ([install docs](https://bun.sh/docs/cli/install)); in practice the interesting part is that a warm-cache install is fast enough to stop being a step you think about.

Non-registry dependencies work the way you'd hope:

```jsonc
{
  "dependencies": {
    "zod": "github:colinhacks/zod",
    "dayjs": "git+https://github.com/iamkun/dayjs.git",
    "lodash": "git+ssh://github.com/lodash/lodash.git#4.17.21",
    "react": "https://registry.npmjs.org/react/-/react-18.2.0.tgz",
    "bun-types": "npm:@types/bun"
  }
}
```

<a id="p4-working"></a>

### Working Knowledge: the lockfile, CI, and the cache

#### `bun.lock` — text, and commit it

Bun's lockfile is **`bun.lock`**, a text file, and has been the default since 1.2 ([install docs](https://bun.sh/docs/cli/install)). It stores resolved versions, integrity hashes, and platform metadata (`cpu`/`os`), which lets Bun skip downloading packages irrelevant to the current platform.

If you inherit a repo with the old binary `bun.lockb`:

```bash
bun install --save-text-lockfile --frozen-lockfile --lockfile-only
rm bun.lockb
```

That converts without changing a single resolved version — `--frozen-lockfile` guarantees it, `--lockfile-only` skips touching `node_modules`.

> Any tutorial telling you to install a special differ, or to run `bun bun.lockb` to read your lockfile, predates 1.2. The current lockfile reviews like any other text file in a pull request, which is the entire point of the change.

#### CI

```bash
bun ci
```

`bun ci` is `bun install --frozen-lockfile`: it installs exactly what the lockfile says and **fails** if `package.json` and `bun.lock` disagree. That failure is a feature — it catches the pull request where someone edited `package.json` by hand and never re-installed.

```yaml
# .github/workflows/ci.yml
- uses: actions/checkout@v4
- uses: oven-sh/setup-bun@v2
- run: bun ci
- run: bun test
```

```bash
# ❌ in CI: silently resolves new versions, so CI tests code your machine never ran
bun install

# ✅ in CI: reproducible, and loud when the lockfile is stale
bun ci
```

#### `--production`

```bash
bun install --production   # skips devDependencies, implies --frozen-lockfile
```

Note the implication — `--production` is already frozen, so you don't need both.

#### The global cache

Packages are extracted once into `~/.bun/install/cache/` and reused across every project on the machine ([install docs](https://bun.sh/docs/cli/install)).

```bash
bun pm cache      # print the cache path
bun pm cache rm   # clear it
```

Offline behaviour is controllable: `--offline` never touches the network (and fails if the cache is cold), `--prefer-offline` uses cached metadata regardless of its age.

**Try It — feel the cache.**

```bash
mkdir cache-demo && cd cache-demo && bun init -y
time bun add zod            # cold-ish: downloads
rm -rf node_modules bun.lock
time bun add zod            # warm: extracted from the local cache
```

Expected: the second run is dramatically faster — commonly sub-second — and produces an identical `node_modules`. The network was not the bottleneck; it was never being asked.

<a id="p4-advanced"></a>

### Advanced: lifecycle scripts and the trust model

This is the Bun default most likely to bite you, and it is a deliberate security decision.

**npm runs `postinstall` scripts from every package you install, transitively.** That is arbitrary code execution from hundreds of authors, triggered by typing `npm install`, and it is how a large share of supply-chain attacks actually land.

**Bun does not.** Lifecycle scripts are blocked unless the package is trusted ([install docs](https://bun.sh/docs/cli/install)). Bun ships a default allowlist of well-known packages that legitimately need to build; anything else is silently skipped.

The failure mode is what makes this Advanced material: **a package that needed to compile a native binary installs "successfully" and then fails at runtime**, with an error about a missing `.node` file that says nothing about lifecycle scripts.

Diagnose and fix:

```bash
bun pm untrusted            # list packages whose scripts were blocked
bun pm trust sharp          # run them and add to trustedDependencies
bun pm trust --all          # trust everything currently blocked
bun pm default-trusted      # see the built-in allowlist
```

`bun pm trust` writes into `package.json`, which is the durable form:

```jsonc
{
  "trustedDependencies": ["sharp", "better-sqlite3"]
}
```

```bash
# ❌ the reflex from npm-land — hides the real problem, may not even help
rm -rf node_modules && bun install

# ✅ ask what was blocked, then trust deliberately
bun pm untrusted
bun pm trust sharp
```

Treat `--all` as a last resort. The whole value of the model is that adding a name to `trustedDependencies` is a review-visible decision, and a diff someone can object to.

<a id="p4-linkers"></a>

### Advanced: hoisted vs. isolated linkers

Two ways to lay out `node_modules`, with different correctness properties ([install docs](https://bun.sh/docs/cli/install)).

```
  HOISTED (npm-style)                  ISOLATED (pnpm-style)

  node_modules/                        node_modules/
    express/                             express -> .bun/express@5.0.0/…
    body-parser/     ← express's dep      .bun/
    debug/           ← transitive           express@5.0.0/node_modules/
    your-app can import ANY of these          express/
                                              body-parser -> …
                                        your-app can import ONLY
                                        what it declared
```

The difference that matters is the **phantom dependency**. Under a hoisted layout, `express`'s transitive dependency `debug` lands at the top of `node_modules`, so `import debug from "debug"` works in *your* code even though you never declared it. It keeps working until express drops that dependency in a patch release, and then your app breaks with a "Cannot find module" from a package you never touched.

Isolated makes that import fail immediately, which is the correct time to fail.

```bash
bun install --linker hoisted
bun install --linker isolated
```

```toml
# bunfig.toml
[install]
linker = "isolated"
```

**The default depends on your repo shape:** hoisted for a single package, isolated for workspaces — a 1.3 change ([Bun 1.3](https://bun.com/blog/bun-v1.3)). So the same command produces a different layout before and after you add a `workspaces` field, which is worth knowing before it surprises you.

Bun 1.4 added a global virtual store on top of isolated installs: packages are extracted once into the cache and symlinked into each project's `node_modules/.bun/`, rather than copied per project — the release post cites 7× faster warm CI installs on a 1,400-package tree ([Bun 1.4](https://bun.com/blog/bun-v1.4)).

**When you must use hoisted:** some tools walk `node_modules` themselves and assume a flat layout (older bundlers, some Electron packagers, a few serverless zip builders). If a tool breaks under isolated in a way you can't fix, `hoistPattern` / `publicHoistPattern` in `bunfig.toml` let you hoist specific packages rather than abandoning isolation entirely.

**Real Scenario — the phantom that only broke in production.** A service imported `ms` for a timeout calculation. Nobody declared it; it was hoisted in as a transitive dependency of the logging library. CI passed for months. Then a dependency update dropped `ms` from that library's tree, `bun ci` produced a `node_modules` without it, and the service crashed on boot in the deploy — never on a developer machine, because those had stale `node_modules` with `ms` still sitting in them. The one-line preventative is `linker = "isolated"`, which turns this into a local failure the moment the import is written.

<a id="p4-mastery"></a>

### Mastery: resolution, peers, and supply-chain controls

#### How a range becomes a version

```
  package.json ranges  ──┐
                         ├──► resolver ──► bun.lock (exact versions + integrity)
  existing bun.lock    ──┘                      │
                                                ▼
                                    cache (~/.bun/install/cache)
                                                │
                                                ▼
                                    node_modules/ (hoisted | isolated)
```

The lockfile is an **input**, not just an output. With a lockfile present and `package.json` unchanged, Bun downloads lazily — only what is actually missing. Change a dependency range and it resolves eagerly again. This is why a no-op `bun install` is nearly instantaneous.

One caching nuance the docs state openly: Bun ignores the registry's `Age`/`Cache-Control` header for manifests, accepting up to about a 5-minute staleness window in exchange for speed. If you publish a package and immediately can't install it, that's the window — not a broken cache.

#### Peer dependencies

Bun handles peers **like Yarn**: it installs them automatically, and respects `peerDependenciesMeta` for optional ones ([install docs](https://bun.sh/docs/cli/install)). This differs from npm 7+, which also auto-installs but is stricter about conflicts, and from older npm, which only warned. If you have a plugin ecosystem with tight peer ranges, this is worth an explicit test rather than an assumption.

#### Supply-chain controls worth turning on

```toml
[install]
# quarantine: refuse versions published in the last 3 days
minimumReleaseAge = 259200
minimumReleaseAgeExcludes = ["@types/node", "typescript"]

# a third-party scanner that vets packages before they install
[install.security]
scanner = "@socketsecurity/bun-security-scanner"
```

`minimumReleaseAge` deserves the emphasis. The dominant npm attack shape is: compromise a maintainer account, publish a malicious patch version, get installed by CI worldwide within minutes, get unpublished within hours. A three-day quarantine turns that entire window into a non-event, at the cost of not getting brand-new releases instantly. The Security Scanner API (added in 1.3) is the pluggable version of the same concern ([Bun 1.3](https://bun.com/blog/bun-v1.3)).

Combined with blocked lifecycle scripts, this is the strongest default posture of any mainstream JavaScript package manager — and it is worth knowing it is there, because none of it is on by default except the script blocking.

#### Cross-platform installs

```bash
bun install --cpu=x64 --os=linux
```

Builds a `node_modules` for a platform you are not on — the honest way to produce a deployable tree from a macOS laptop for a Linux container, without Docker.

<a id="part-4-cheat-sheet"></a>

### Part 4 cheat sheet

| Want | Command |
|---|---|
| Install | `bun install` |
| Reproducible CI install | `bun ci` |
| Add / remove | `bun add pkg` / `bun remove pkg` |
| Dev dependency | `bun add -d pkg` |
| Production tree | `bun install --production` |
| Convert `bun.lockb` → `bun.lock` | `bun install --save-text-lockfile --frozen-lockfile --lockfile-only` |
| See blocked install scripts | `bun pm untrusted` |
| Allow one package's scripts | `bun pm trust <pkg>` |
| pnpm-style strictness | `bun install --linker isolated` |
| Where's the cache? | `bun pm cache` |
| Clear the cache | `bun pm cache rm` |
| Build for another platform | `bun install --cpu=x64 --os=linux` |
| Quarantine new releases | `[install] minimumReleaseAge` in `bunfig.toml` |

[↑ Back to top](#table-of-contents)

---

## Part 5 — Monorepos & the Rest of `bun pm`

*This Part collapses the Mastery tier for `bunx` and the housekeeping commands — their internals are just Part 4's resolver — and spends the depth on workspaces and migration, which is where real decisions live.*

<a id="p5-beginner"></a>

### Beginner: a workspace in five lines

```jsonc
// package.json at the repo root
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": ["packages/*"]
}
```

```jsonc
// packages/api/package.json
{
  "name": "@acme/api",
  "version": "1.0.0",
  "dependencies": {
    "@acme/core": "workspace:*"
  }
}
```

One `bun install` at the root installs every workspace and links `@acme/core` to the **local folder**, not the registry ([workspaces docs](https://bun.sh/docs/install/workspaces)). Edit `@acme/core`, and `@acme/api` sees the change immediately — no build, no `npm link`, no watcher.

The `workspaces` field takes full globs, including negations:

```jsonc
{ "workspaces": ["packages/**", "!packages/**/test/**", "!packages/**/template/**"] }
```

On publish, Bun rewrites the `workspace:` protocol to real semver: `workspace:*` → `1.0.1`, `workspace:^` → `^1.0.1`.

<a id="p5-working"></a>

### Working Knowledge: filtering and catalogs

#### `--filter`

Run a script, or install, in a subset of workspaces:

```bash
bun run --filter '@acme/*' build        # build every @acme package
bun run --filter './packages/api' dev   # by path
bun run --filter '@acme/api...' build   # …and everything it depends on
bun install --filter '!@acme/docs'      # everything except docs
```

`--parallel` and `--sequential` control how multi-package script runs are scheduled, and `--elide-lines N` truncates each package's output to its last N lines, which is what makes a 12-package build watchable ([run docs](https://bun.sh/docs/cli/run)).

The `pkg...` trailing-dots syntax is the one worth remembering: "this package plus its dependencies", which is exactly the set you need to rebuild before running one app.

#### Catalogs

The monorepo version-drift problem: fifteen packages each declaring `"react": "^19.0.0"`, and one that quietly says `^18.2.0`. **Catalogs** put the version in one place ([workspaces docs](https://bun.sh/docs/install/workspaces), [catalogs docs](https://bun.sh/docs/pm/catalogs)):

```jsonc
// root package.json
{
  "workspaces": {
    "packages": ["packages/*"],
    "catalog": {
      "react": "^19.0.0",
      "zod": "^3.23.8"
    },
    "catalogs": {
      "testing": { "vitest": "^2.0.0" }
    }
  }
}
```

```jsonc
// packages/web/package.json
{
  "dependencies": { "react": "catalog:" },
  "devDependencies": { "vitest": "catalog:testing" }
}
```

Now upgrading React is one edit at the root instead of fifteen edits and a review that has to notice the one you missed.

#### Self-contained workspaces

Some tools (Electron packagers, serverless bundlers) need a workspace to have a complete, real `node_modules` rather than links up to the root:

```jsonc
{
  "workspaces": {
    "packages": ["apps/*", "packages/*"],
    "selfContained": ["apps/desktop"]
  }
}
```

<a id="p5-advanced"></a>

### Advanced: `bunx`, auditing, and housekeeping

#### `bunx`

```bash
bunx cowsay "hello"          # run a package binary without installing it
bunx --bun vite              # …and force it onto Bun's runtime
bun x cowsay "hello"         # same command, spelled differently
```

`bunx` is `npx` with Bun's cache and Bun's startup time. The `--bun` flag matters here for exactly the reason it did in [Part 1](./bun-runtime-foundations.md#p1-working): without it, a tool with a `#!/usr/bin/env node` shebang runs on Node.

#### Auditing and pruning

```bash
bun audit                # report known vulnerabilities
bun audit fix            # ...and apply fixable upgrades      (1.4)
bun outdated             # what's behind, with the range that's holding it back
bun update               # update within ranges
bun update --latest      # ignore ranges, take the newest
bun dedupe               # collapse duplicate versions        (1.4)
bun prune                # remove extraneous packages          (1.4)
bun pm licenses          # every dependency grouped by license
```

`bun outdated` is the one to run before `bun update`: it shows the *declared range* next to the available version, so you can see whether you're one `bun update` away or need to widen a caret by hand.

`bun pm licenses` answers the legal-review question in one command, with `--json` for scripting and `--prod` to exclude dev dependencies — considerably less painful than the license-checker packages this replaces.

#### The rest of `bun pm`

| Command | Does |
|---|---|
| `bun pm ls` / `bun pm ls --all` | Direct / full dependency tree |
| `bun pm bin` / `bun pm bin -g` | Path to the local / global bin directory |
| `bun pm pack` | Build a publishable `.tgz` (npm-pack rules); `--dry-run` to preview contents |
| `bun why <pkg>` | Why is this installed? — the dependency chain (top-level command, not under `pm`) |
| `bun pm diff react@18.2.0 19.0.0` | Diff two published versions of a package |
| `bun pm version patch\|minor\|major` | Bump `package.json` version |
| `bun pm pkg get scripts.build` | Read/write `package.json` fields by path |
| `bun pm hash` | Hash of the current lockfile — cheap CI cache key |
| `bun pm migrate` | Convert another manager's lockfile, without installing |

`bun pm pack --dry-run` is the underrated one: it lists exactly what would ship in your tarball. Publishing your `.env` or your `test/fixtures/` is a mistake you make once, and this is the thirty-second check that prevents it.

<a id="p5-mastery"></a>

### Mastery: migrating an existing repo

Bun reads `package-lock.json`, `yarn.lock`, and `pnpm-lock.yaml` and converts them, preserving your resolved versions rather than re-resolving from scratch ([Bun 1.3](https://bun.com/blog/bun-v1.3)):

```bash
bun pm migrate     # convert the lockfile only — no install, nothing else touched
```

A migration order that fails cheaply:

1. **Convert without installing.** `bun pm migrate`, then read the resulting `bun.lock` diff. If versions moved, understand why before continuing.
2. **Install and look at what got blocked.** `bun install && bun pm untrusted`. Every name printed is a package that ran a build step under npm and didn't here. Trust them explicitly — this is the step people skip, and it produces the "works locally, broken in Docker" class of bug.
3. **Pick a linker deliberately.** Coming from pnpm, use `linker = "isolated"` to keep the strictness you already have. Coming from npm or Yarn, `isolated` will surface phantom dependencies as immediate failures — that's a real fix list, not a Bun bug, but do it as its own commit.
4. **Run the test suite on Node first, with Bun's `node_modules`.** This separates "the install is different" from "the runtime is different." If `node ./node_modules/.bin/jest` passes, your install is fine and any remaining failure is a runtime compat question — [Part 13](./bun-node-compat-internals.md).
5. **Then switch the runtime.** Change scripts to `bun run`, add `--bun` where a tool must not shell out to Node, and only now port tests to `bun test` if you want to ([Part 11](./bun-testing-bundling.md)).
6. **Update CI last**, to `bun ci`, and delete the old lockfile in the same commit so nothing can silently fall back to it.

The ordering principle: change one variable at a time. Most failed Bun migrations swap the package manager, the runtime, and the test runner in one pull request and then cannot tell which of the three broke.

<a id="part-5-cheat-sheet"></a>

### Part 5 cheat sheet

| Want | How |
|---|---|
| Declare workspaces | `"workspaces": ["packages/*"]` |
| Depend on a sibling | `"@acme/core": "workspace:*"` |
| One version for all packages | `catalog:` / `"catalog": { … }` at the root |
| Build one package + its deps | `bun run --filter '@acme/api...' build` |
| Run without installing | `bunx <pkg>` (add `--bun` to force Bun's runtime) |
| Vulnerability report | `bun audit` / `bun audit fix` |
| What's out of date | `bun outdated` |
| Collapse duplicate versions | `bun dedupe` |
| Why is this installed? | `bun pm why <pkg>` |
| What would I publish? | `bun pm pack --dry-run` |
| License inventory | `bun pm licenses` |
| Convert from npm/yarn/pnpm | `bun pm migrate` |
| CI cache key | `bun pm hash` |

[↑ Back to top](#table-of-contents)

---

**Next:** [Parts 6–8 — APIs, I/O & Processes](./bun-apis-io-processes.md) · **Up:** [Overview](./bun-mastery-guide.md)

*Written against Bun v1.4.2 · Verified September 6, 2026*
