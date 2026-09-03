# CI/CD Mastery Guide — A Hands-On Reference (GitHub Actions)

## About This Document

**Spec captured from the original request, so future updates match style without re-explaining:**

- **Framing:** Curiosity-driven — no specific upcoming task drives this. Structured as a guided, hands-on tutorial (runnable examples + exercises) that builds durable technical fluency, not a project-specific how-to. Theory appears only at the point it's needed to explain a mechanism (e.g. why `cancel-in-progress` exists, why OIDC replaces long-lived cloud keys) — never as a standalone history section.
- **Confirmed scope:** GitHub Actions as the primary spine (not a multi-tool comparison), covering CI/CD implementation end to end — workflow mechanics, actions/caching/artifacts, secrets/permissions/environments, testing gates, deployment strategies, reusable workflows, production hardening, and runner internals.
- **Depth tiers:** Every major Part is structured as four progressive, explicitly labeled tiers — **Beginner → Working Knowledge → Advanced → Mastery** — so you can stop at whichever level you need. Tiers are collapsed where trivial for that topic (Part 0 and Part 8 don't need all four).
- **Assumed background:** Basic Git (branches, commits, PRs) is assumed. No prior CI/CD experience assumed.
- **Example project:** A single small Node.js repo (`ci-demo/`) evolves across Parts 1–7 rather than disconnected snippets each time — clone-and-follow-along style.
- **Per-section ingredients (used where they add value):** plain-language explanation; a "wrong vs. right" snippet pair for gotchas; a runnable "Try It" exercise (curiosity-driven default); an ASCII diagram where the concept is spatial/structural (trigger flow, job dependency graph, OIDC handshake).
- **Sourcing rule:** Official GitHub Docs cited first; Stack Overflow / GitHub Discussions used only for real-world gotchas, flagged as such; official docs win whenever they disagree with a popular but outdated blog convention (called out explicitly, e.g. `actions/upload-artifact@v3` deprecation, SHA-pinning).
- **Format:** Clickable TOC, "back to top" links after each Part, cheat-sheet tables at the end, fenced code blocks with language tags.
- **To update this doc later:** keep this section, keep the tiered structure, keep the wrong-vs-right and Try-It patterns, re-verify anything version-specific against current docs before changing it, and prefer adding a new Part/tier over reverting to freeform prose.

---

## Table of Contents

- [Part 0: CI/CD Primer](#part-0-cicd-primer)
- [Part 1: Workflow & Job Anatomy](#part-1-workflow--job-anatomy)
  - [Beginner](#part-1-beginner)
  - [Working Knowledge](#part-1-working-knowledge)
  - [Advanced](#part-1-advanced)
  - [Mastery](#part-1-mastery)
- [Part 2: Actions, Caching & Artifacts](#part-2-actions-caching--artifacts)
  - [Beginner](#part-2-beginner)
  - [Working Knowledge](#part-2-working-knowledge)
  - [Advanced](#part-2-advanced)
- [Part 3: Secrets, Permissions & Environments](#part-3-secrets-permissions--environments)
  - [Beginner](#part-3-beginner)
  - [Working Knowledge](#part-3-working-knowledge)
  - [Advanced](#part-3-advanced)
  - [Mastery](#part-3-mastery)
- [Part 4: CI Proper — Testing & Quality Gates](#part-4-ci-proper--testing--quality-gates)
  - [Beginner](#part-4-beginner)
  - [Working Knowledge](#part-4-working-knowledge)
  - [Advanced](#part-4-advanced)
- [Part 5: CD Proper — Deployment Strategies](#part-5-cd-proper--deployment-strategies)
  - [Beginner](#part-5-beginner)
  - [Working Knowledge](#part-5-working-knowledge)
  - [Advanced](#part-5-advanced)
- [Part 6: Composition — Reusable Workflows & Composite Actions](#part-6-composition--reusable-workflows--composite-actions)
  - [Working Knowledge](#part-6-working-knowledge)
  - [Advanced](#part-6-advanced)
- [Part 7: Production Concerns](#part-7-production-concerns)
  - [Advanced](#part-7-advanced)
  - [Mastery](#part-7-mastery)
- [Part 8: Internals Deep Dive](#part-8-internals-deep-dive)
- [Cheat Sheets](#cheat-sheets)
- [Suggested Learning Order](#suggested-learning-order)
- [Quick Self-Check](#quick-self-check)

---

## Part 0: CI/CD Primer

**What CI/CD actually solves.** Before any tooling: "CI" (Continuous Integration) means every code change is automatically built and tested the moment it's pushed, so integration bugs surface in minutes, not weeks. "CD" is overloaded — **Continuous Delivery** means every change that passes CI is automatically packaged into a releasable artifact (a human still clicks "deploy"); **Continuous Deployment** means passing CI *automatically* ships to production, no human in the loop. Most real pipelines are Continuous Delivery with a manual gate at the production step — pure Continuous Deployment is rarer than the name's popularity suggests.

**GitHub Actions vocabulary, top to bottom:**

```
Workflow  → one YAML file (.github/workflows/*.yml). A workflow is triggered by an event.
  Job     → a unit of work that runs on one runner (one VM/container). Jobs run in parallel by default.
    Step  → one command or one Action, executed in sequence inside a job.
      Action → reusable packaged step (either your own, or from the Marketplace).
Runner    → the VM/container that actually executes a job's steps.
```

```
Event (push, PR, schedule, ...)
        │
        ▼
    Workflow file matches on: trigger
        │
        ▼
  ┌─────────────┐     ┌─────────────┐
  │   Job A     │     │   Job B     │   ← run in parallel unless job B needs: job A
  │ step 1      │     │ step 1      │
  │ step 2      │     │ step 2      │
  │ (on Runner) │     │ (on Runner) │
  └─────────────┘     └─────────────┘
```

**Try It — set up the example repo:**

```bash
mkdir ci-demo && cd ci-demo
git init
npm init -y
npm pkg set scripts.test="node --test"
mkdir -p .github/workflows
```

You now have an empty Node project with a `.github/workflows/` directory — every workflow file you add from here on lives there and is picked up automatically by GitHub the moment you push.

[↑ back to top](#table-of-contents)

---

## Part 1: Workflow & Job Anatomy

<a id="part-1-beginner"></a>

### Beginner

A workflow file needs three things: a name, a trigger (`on:`), and at least one job with steps.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm ci
      - run: npm test
```

`actions/checkout@v4` is almost always your first step — without it, the runner is an empty VM with no copy of your repo. `uses:` runs a packaged Action; `run:` runs a raw shell command on the runner.

**Try It:** commit this file, push to a `main` branch on GitHub, and watch it run under the repo's **Actions** tab. Then open a PR from a branch — notice it runs again, because `pull_request` is also a trigger. [Workflow syntax reference](https://docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions).

[↑ back to top](#table-of-contents)

<a id="part-1-working-knowledge"></a>

### Working Knowledge

**Multiple jobs and dependencies.** Jobs run in parallel unless you say otherwise with `needs:`.

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [...]
  test:
    runs-on: ubuntu-latest
    steps: [...]
  build:
    runs-on: ubuntu-latest
    needs: [lint, test]        # waits for both to succeed
    steps: [...]
```

```
lint ──┐
       ├─▶ build
test ──┘
```

**Common trigger patterns:**

```yaml
on:
  push:
    branches: [main]
    paths: ["src/**", "package.json"]   # skip workflow if only docs changed
  pull_request:
    types: [opened, synchronize, reopened]
  schedule:
    - cron: "0 3 * * *"                 # 3am UTC daily
  workflow_dispatch:                    # manual "Run workflow" button
    inputs:
      environment:
        type: choice
        options: [staging, production]
```

**Wrong vs. right — path filters on required checks:**

```yaml
# WRONG: if this workflow is a "required status check" in branch protection,
# path filtering means it never runs (and never reports) for docs-only PRs —
# the PR gets stuck "waiting for status to be reported", not merged.
on:
  pull_request:
    paths: ["src/**"]
```

```yaml
# RIGHT: either don't filter paths on a required-check workflow, or add a
# second lightweight job that always runs and reports success for excluded paths.
on:
  pull_request:
```

**Contexts and expressions** let steps read runtime data: `${{ github.event_name }}`, `${{ github.ref }}`, `${{ runner.os }}`, `${{ steps.<id>.outputs.<name> }}`. `if:` conditions gate whole steps or jobs:

```yaml
- name: Deploy only from main
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  run: ./deploy.sh
```

[Events that trigger workflows](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows) · [Contexts reference](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts)

[↑ back to top](#table-of-contents)

<a id="part-1-advanced"></a>

### Advanced

**Matrix builds** run the same job across a grid of variables — the classic use is testing multiple language/OS versions in parallel:

```yaml
jobs:
  test:
    strategy:
      fail-fast: false          # don't cancel other cells if one fails
      matrix:
        node: [18, 20, 22]
        os: [ubuntu-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci && npm test
```

This produces 6 parallel jobs (3 × 2). `fail-fast: false` is worth calling out explicitly: the default (`true`) cancels every other matrix cell the instant one fails, which hides whether the *other* cells would also have failed — usually not what you want when you're specifically trying to find out which combinations are broken.

**Concurrency control** prevents redundant runs from piling up — critical once a team pushes frequently to the same PR:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

Every new push to the same ref cancels the in-progress run for that group and starts fresh — you stop burning runner-minutes testing a commit that's already obsolete. [Control workflow concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency).

**Job outputs** pass data between dependent jobs (each job is a separate runner/VM, so this is the only way):

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.ver.outputs.value }}
    steps:
      - id: ver
        run: echo "value=$(node -p require('./package.json').version)" >> "$GITHUB_OUTPUT"
  publish:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Publishing version ${{ needs.build.outputs.version }}"
```

[↑ back to top](#table-of-contents)

<a id="part-1-mastery"></a>

### Mastery

**Reserved job names and default permissions.** Every workflow run gets an auto-generated `GITHUB_TOKEN` scoped to the triggering repo — its default permission level depends on repo/org settings, and it's a frequent source of "why did this fail with 403" confusion when a workflow was written against a repo whose default was later tightened. Covered fully in [Part 3](#part-3-secrets-permissions--environments).

**Composite triggers can double-fire.** A push that's also part of a PR sync can trigger both `push` and `pull_request` for the same commit — this is *why* `concurrency` groups and `if: github.event_name == 'push'` guards exist, not just style preferences.

**Expression evaluation happens server-side before the runner even starts** — `if:` conditions, `matrix` expansion, and `env:` interpolation at the job level are all resolved by GitHub's backend, not inside the VM. This matters for security: an `if:` condition referencing untrusted input (like a PR title) is evaluated in a context that historically has been an injection vector — see [Part 7](#part-7-production-concerns).

[↑ back to top](#table-of-contents)

---

## Part 2: Actions, Caching & Artifacts

<a id="part-2-beginner"></a>

### Beginner

An **Action** is a reusable unit referenced with `uses: owner/repo@ref`. `actions/checkout@v4` and `actions/setup-node@v4` (used above) are both Actions maintained by GitHub itself. Anything in the [GitHub Marketplace](https://github.com/marketplace?type=actions) works the same way.

**Try It:** add a step that uses an Action you haven't used yet — `actions/setup-python@v5` — to a throwaway workflow and confirm `python --version` reports the version you requested. This is the fastest way to build intuition for how `uses:` + `with:` map to an Action's own `action.yml` inputs.

[↑ back to top](#table-of-contents)

<a id="part-2-working-knowledge"></a>

### Working Knowledge

**Dependency caching** avoids re-downloading `node_modules` (or equivalent) on every run. `actions/setup-node` has built-in caching support:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: "20"
    cache: "npm"          # caches based on the lockfile hash automatically
- run: npm ci
```

For anything without built-in support, use `actions/cache` directly, keyed on a lockfile hash so the cache invalidates exactly when dependencies change:

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.cache/some-tool
    key: ${{ runner.os }}-sometool-${{ hashFiles('**/lockfile.lock') }}
    restore-keys: |
      ${{ runner.os }}-sometool-
```

**Wrong vs. right — cache key granularity:**

```yaml
# WRONG: a static key never changes, so the cache never invalidates —
# you'll keep restoring stale dependencies forever after a lockfile bump.
key: node-modules-cache
```

```yaml
# RIGHT: hash the lockfile into the key so a dependency change produces
# a new key (cache miss → fresh install → new cache saved).
key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
```

**Artifacts** move files *between jobs* or out of the run entirely (e.g. a build output a human downloads, or a coverage report). As of `v4`, artifacts uploaded in one job are immediately available for download by another job that `needs:` it — a change from `v3`'s "settle at end of run" behavior:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      - run: ./deploy.sh
```

[↑ back to top](#table-of-contents)

<a id="part-2-advanced"></a>

### Advanced

**`actions/upload-artifact@v3` and `download-artifact@v3` are deprecated and were fully retired on GitHub-hosted runners** — workflows still pinned to `@v3` fail outright, not just warn. If you inherited an older pipeline, this is usually the first thing that's actually broken, not just "using an old version." `v4` is also a genuinely different model, not a drop-in bump:

- Uploads are **immutable** — you can't append to an existing artifact name mid-run, and by default a duplicate `name:` within one run fails the step (this is the #1 gotcha in matrix jobs that all upload an artifact called `build`).
- `download-artifact@v4` creates a **subdirectory per artifact name** when no explicit `name:` is given, whereas `v3` flattened everything — scripts written for `v3`'s flat layout silently look in the wrong path under `v4`.

```yaml
# WRONG under v4 — every matrix cell tries to create an artifact named "coverage",
# and only the first upload in the run succeeds; the rest fail the step.
strategy:
  matrix: { node: [18, 20, 22] }
steps:
  - uses: actions/upload-artifact@v4
    with:
      name: coverage
      path: coverage/
```

```yaml
# RIGHT — give each matrix cell a unique artifact name, then merge if needed.
steps:
  - uses: actions/upload-artifact@v4
    with:
      name: coverage-node${{ matrix.node }}
      path: coverage/
```

[Deprecation notice: v3 of the artifact actions](https://github.blog/changelog/2024-04-16-deprecation-notice-v3-of-the-artifact-actions/) · [actions/upload-artifact MIGRATION.md](https://github.com/actions/upload-artifact) — see the repo's migration guide linked from the README.

**Pinning Actions by SHA, not tag.** A tag like `@v4` is a *mutable* pointer — a compromised or careless maintainer can move it to point at different code without changing the visible version string in your workflow. Pinning to a full commit SHA makes the reference immutable regardless of what the tag later points to. Covered in depth (with the security rationale) in [Part 7](#part-7-production-concerns).

[↑ back to top](#table-of-contents)

---

## Part 3: Secrets, Permissions & Environments

<a id="part-3-beginner"></a>

### Beginner

**Secrets** are encrypted values scoped to a repo, environment, or org, referenced via the `secrets` context — they're never printed to logs (GitHub automatically redacts an exact-match secret value if it ever appears in output).

```yaml
steps:
  - run: curl -H "Authorization: Bearer ${{ secrets.API_TOKEN }}" https://api.example.com
```

Set one under **Settings → Secrets and variables → Actions** in the repo, or via `gh secret set API_TOKEN`.

[↑ back to top](#table-of-contents)

<a id="part-3-working-knowledge"></a>

### Working Knowledge

**`GITHUB_TOKEN` is auto-generated per run** — you never create it — scoped to that repo and expiring when the job ends. Its default permission set depends on a repo/org setting (**Settings → Actions → Workflow permissions**: "read" or "read and write"). Explicitly declaring `permissions:` in the workflow is the recommended practice regardless of that default, because it documents intent and survives someone changing the org default later:

```yaml
permissions:
  contents: read       # least privilege — override per-job as needed
  pull-requests: write # only the job that comments on PRs needs this
```

**Wrong vs. right — token scope:**

```yaml
# WRONG: no permissions block means the workflow silently inherits whatever
# the org/repo default is — could be broad write access nobody remembers granting.
jobs:
  comment-on-pr:
    runs-on: ubuntu-latest
    steps:
      - run: gh pr comment ...
```

```yaml
# RIGHT: explicit least-privilege permissions, scoped at the job level
# so other jobs in the same workflow don't inherit write access they don't need.
jobs:
  comment-on-pr:
    permissions:
      pull-requests: write
    runs-on: ubuntu-latest
    steps:
      - run: gh pr comment ...
```

**Environments** (**Settings → Environments**) group secrets/variables under a name (`staging`, `production`) and let a job reference them:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: ./deploy.sh
        env:
          DEPLOY_KEY: ${{ secrets.PROD_DEPLOY_KEY }}   # only visible to jobs targeting this environment
```

[↑ back to top](#table-of-contents)

<a id="part-3-advanced"></a>

### Advanced

**Environment protection rules** gate a job behind a real approval step — this is how you get a human-in-the-loop production deploy without leaving GitHub Actions:

```yaml
jobs:
  deploy-prod:
    environment: production   # if "production" has required reviewers configured,
    runs-on: ubuntu-latest    # this job PAUSES and waits for approval before running
    steps:
      - run: ./deploy.sh
```

You can list up to 6 required reviewers per environment; only one needs to approve. You can also enable **prevent self-review**, so whoever triggered the run can't be the one who approves it — closes an obvious loophole for solo-approver setups. [Reviewing deployments](https://docs.github.com/actions/managing-workflow-runs/reviewing-deployments) · [Deployments and environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments).

**Real Scenario:** a team stores a cloud provider's long-lived access key as a repo secret, used by every workflow that touches infra. Six months later a dependency-confusion attack compromises a third-party Action in one workflow; because that key is a repo secret (not environment-scoped), *every* workflow in the repo — including ones that had no business touching production — had access to it the moment it leaked into a malicious step's environment. The fix isn't "rotate the key," it's structural: scope deploy credentials to the `production` *environment* (only jobs explicitly targeting it can read the secret) and, better, eliminate the long-lived key entirely via OIDC (next tier).

[↑ back to top](#table-of-contents)

<a id="part-3-mastery"></a>

### Mastery

**OpenID Connect (OIDC) to cloud providers** eliminates long-lived cloud credentials from GitHub secrets altogether. Instead of storing an AWS access key, the runner requests a short-lived signed JWT from GitHub's own OIDC provider (`https://token.actions.githubusercontent.com`), and the cloud provider trusts that token (scoped to a specific repo/branch/environment via a trust policy) to hand back temporary credentials valid only for that job.

```
┌──────────┐   1. request JWT    ┌────────────────────────┐
│  Runner  │ ──────────────────▶ │ GitHub OIDC provider    │
│  (job)   │ ◀────────────────── │ token.actions.githubusercontent.com │
└──────────┘   2. signed JWT     └────────────────────────┘
     │
     │ 3. present JWT to STS (AssumeRoleWithWebIdentity)
     ▼
┌──────────────────┐   4. validate against trust policy   ┌───────────────┐
│   AWS STS         │ ────────────────────────────────────▶ │ short-lived   │
│                    │ ◀──────────────────────────────────── │ credentials   │
└──────────────────┘   5. temporary creds (expire w/ job)  └───────────────┘
```

```yaml
permissions:
  id-token: write     # REQUIRED — without this, the OIDC token isn't issued at all
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-deploy-role
          aws-region: us-east-1
      - run: aws s3 sync dist/ s3://my-bucket
```

The trust policy on the AWS role side restricts *which* repo/branch/environment can assume it (matching claims in the JWT, e.g. `repo:my-org/my-repo:environment:production`) — so even if this workflow file leaked publicly, nobody outside that exact repo/branch context could mint valid credentials with it. [Configuring OpenID Connect in AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) · [About security hardening with OpenID Connect](https://docs.github.com/en/actions/concepts/security/openid-connect).

[↑ back to top](#table-of-contents)

---

## Part 4: CI Proper — Testing & Quality Gates

<a id="part-4-beginner"></a>

### Beginner

A "quality gate" is just a job whose failure blocks something — most simply, a red X on a PR. That alone is CI's core value: nothing merges without at least being *seen* to fail or pass.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "npm" }
      - run: npm ci
      - run: npm test
```

[↑ back to top](#table-of-contents)

<a id="part-4-working-knowledge"></a>

### Working Knowledge

**Branch protection + required status checks** is what turns "a workflow ran" into "a workflow *must pass* to merge." Configure under **Settings → Branches → Branch protection rules**: pick the target branch, enable "Require status checks to pass," and select the job name(s) from workflows that have run at least once (they won't appear in the picker until GitHub has seen them run).

**Wrong vs. right — checking the right thing as a required check:**

```yaml
# WRONG: naming the WORKFLOW as the required check is not possible directly —
# GitHub requires status checks by JOB name. If a workflow has multiple jobs
# and you only mark one required, the others can fail silently without blocking merge.
jobs:
  lint: { ... }
  test: { ... }     # only this one marked "required" in branch protection
  build: { ... }    # this can fail and the PR still merges
```

```yaml
# RIGHT: either mark every job that matters as required, or add a final
# "gate" job that needs: all of them, and mark only the gate as required.
jobs:
  lint: { ... }
  test: { ... }
  build: { ... }
  ci-gate:
    needs: [lint, test, build]
    runs-on: ubuntu-latest
    steps: [{ run: "echo ok" }]   # required check = ci-gate only
```

The gate-job pattern is worth internalizing early — it's the standard way to add/remove jobs later without having to re-touch branch protection settings every time.

[↑ back to top](#table-of-contents)

<a id="part-4-advanced"></a>

### Advanced

**Real Scenario:** a required check named `test` runs fine on `main`, but someone adds `paths: ["src/**"]` to that workflow's trigger (to save runner minutes on docs-only PRs). The next PR that touches only `README.md` shows the PR stuck on "Some checks haven't completed yet" indefinitely — the workflow never even starts, so it can never report success, and GitHub won't let you merge a PR with a required check that's never reported. This is the exact "wrong vs. right" from [Part 1](#part-1-working-knowledge) resurfacing at the quality-gate layer: path-filtered triggers and required-status-checks are fundamentally in tension unless you add a fallback job. [GitHub Community discussion on this exact failure mode](https://github.com/orgs/community/discussions) (search "required check path filter stuck" — a frequently reported gotcha).

**Caching test runs / turbocharging feedback loops:** combine dependency caching ([Part 2](#part-2-working-knowledge)) with splitting test suites across matrix cells for large codebases:

```yaml
strategy:
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - run: npm test -- --shard=${{ matrix.shard }}/4
```

[↑ back to top](#table-of-contents)

---

## Part 5: CD Proper — Deployment Strategies

<a id="part-5-beginner"></a>

### Beginner

The simplest CD job is: build, then push/copy the result somewhere, gated to only run on `main`:

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main'
    needs: test
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - run: ./scripts/deploy.sh
```

[↑ back to top](#table-of-contents)

<a id="part-5-working-knowledge"></a>

### Working Knowledge

**Multi-environment promotion.** A realistic pipeline deploys to `staging` automatically, then `production` only after approval — expressed as two jobs, each with its own `environment:`:

```yaml
jobs:
  deploy-staging:
    needs: test
    environment: staging          # no required reviewers — deploys immediately
    runs-on: ubuntu-latest
    steps: [{ run: ./deploy.sh staging }]

  deploy-production:
    needs: deploy-staging
    environment: production       # required reviewers configured on this environment
    runs-on: ubuntu-latest
    steps: [{ run: ./deploy.sh production }]
```

```
push to main
     │
     ▼
   test ──▶ deploy-staging ──▶ [human approval] ──▶ deploy-production
```

**Deployment strategy concepts** (the *how* of the deploy script itself, not GitHub-Actions-specific, but essential vocabulary):

| Strategy | How it works | Rollback speed | Cost |
|---|---|---|---|
| **Recreate** | Stop old version, start new version | Slow (redeploy old) | Lowest |
| **Rolling** | Replace instances gradually, old+new coexist briefly | Medium | Low |
| **Blue-Green** | Deploy new version fully alongside old, switch traffic at once | Instant (switch back) | Higher (2x infra briefly) |
| **Canary** | Route a small % of traffic to new version, expand gradually | Fast (stop the rollout) | Medium |

[↑ back to top](#table-of-contents)

<a id="part-5-advanced"></a>

### Advanced

**Rollback as a first-class workflow, not an afterthought.** A deploy workflow that can only move forward is a production incident waiting to happen. A `workflow_dispatch`-triggered rollback job, parameterized by the target version, means "roll back" is a button click, not an emergency SSH session:

```yaml
name: Rollback
on:
  workflow_dispatch:
    inputs:
      version:
        description: "Git tag or SHA to roll back to"
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ inputs.version }}
      - run: ./deploy.sh production
```

**Real Scenario:** a canary deploy routes 5% of traffic to a new version; an error-rate spike alert fires 3 minutes in. If "abort the canary" requires re-running the *entire* deploy pipeline from scratch (build → test → deploy), the blast radius window is however long that full pipeline takes — often 10+ minutes. Teams that treat rollback as its own short, dedicated workflow (as above) cut that window to however long it takes the deploy step alone to run in reverse.

[↑ back to top](#table-of-contents)

---

## Part 6: Composition — Reusable Workflows & Composite Actions

<a id="part-6-working-knowledge"></a>

### Working Knowledge

Two different composition mechanisms solve two different problems — mixing them up is the most common confusion here:

- **Composite Action** — bundles multiple **steps** into one reusable step. Lives in its own `action.yml`, used with `uses:` inside a job's `steps:` list.
- **Reusable Workflow** — bundles multiple **jobs** into one reusable workflow. Triggered with `on: workflow_call`, invoked with `uses:` at the job level (not inside `steps:`).

**Reusable workflow** — define once:

```yaml
# .github/workflows/reusable-deploy.yml
on:
  workflow_call:
    inputs:
      environment:
        type: string
        required: true
    secrets:
      DEPLOY_KEY:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - run: ./deploy.sh
        env: { KEY: "${{ secrets.DEPLOY_KEY }}" }
```

Call it from another workflow:

```yaml
jobs:
  deploy-staging:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
    secrets:
      DEPLOY_KEY: ${{ secrets.STAGING_DEPLOY_KEY }}
```

`secrets: inherit` passes every secret the caller has access to, instead of naming them individually — convenient, but it also means the reusable workflow silently gains access to secrets nobody explicitly granted it, which is worth pausing on before using it in anything beyond a small trusted org. [Reuse workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows).

**Composite Action** — for reusing a handful of steps (e.g. "checkout + setup-node + npm ci", repeated identically across five workflows):

```yaml
# .github/actions/setup-project/action.yml
name: Setup Project
runs:
  using: composite
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: { node-version: "20", cache: "npm" }
    - run: npm ci
      shell: bash          # required for every `run:` step inside a composite action
```

```yaml
# used elsewhere as one step
steps:
  - uses: ./.github/actions/setup-project
  - run: npm test
```

[↑ back to top](#table-of-contents)

<a id="part-6-advanced"></a>

### Advanced

**Wrong vs. right — reusable workflow vs. composite action for the wrong job:**

```yaml
# WRONG: trying to share "build, then deploy to staging, then run smoke tests"
# — three JOBS worth of orchestration — as a composite action forces it all
# into one job, losing parallelism and per-stage status visibility in the UI.
```

```yaml
# RIGHT: that's a reusable WORKFLOW's job — composite actions are for
# step-level reuse within a single job, reusable workflows are for
# job-level (or multi-job) orchestration reuse.
uses: ./.github/workflows/build-deploy-verify.yml
```

Reusable workflows can nest up to 10 levels deep and a single workflow can call up to 50 unique reusable workflows — generous limits that rarely bind in practice, but worth knowing exist before architecting something unusually deep. [Reusing workflow configurations](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations).

[↑ back to top](#table-of-contents)

---

## Part 7: Production Concerns

<a id="part-7-advanced"></a>

### Advanced

**Pin third-party Actions to a full commit SHA, not a tag.** A version tag like `@v4` is a mutable Git ref — the maintainer (or an attacker who compromises the maintainer's account) can repoint it to different code at any time without your workflow file changing at all. This is the single highest-leverage supply-chain hardening step available, because it costs nothing except readability:

```yaml
# WRONG: @v4 can be moved by the action's maintainer (or an attacker who
# compromises that account) to point at different, possibly malicious code —
# your workflow silently starts running whatever @v4 now resolves to.
- uses: actions/checkout@v4
```

```yaml
# RIGHT: pinned to an immutable commit, with the human-readable version
# kept in a trailing comment so you still know what you're looking at.
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

Tools like Dependabot can keep SHA-pinned actions updated automatically (it resolves the new tag to a new SHA for you, rather than you doing it by hand). [Pinning GitHub Actions for enhanced security](https://www.stepsecurity.io/blog/pinning-github-actions-for-enhanced-security-a-complete-guide) · [Pin your GitHub Actions to protect against supply chain attacks](https://emmer.dev/blog/pin-your-github-actions-to-protect-against-mutability/) · GitHub's own org-level [Actions policy now supports enforcing SHA pinning](https://github.blog/changelog/2025-08-15-github-actions-policy-now-supports-blocking-and-sha-pinning-actions/) as a first-class setting, rather than relying on individual workflow discipline.

**`pull_request_target` vs. `pull_request` — the injection trap.** `pull_request` runs with a read-only token and no access to secrets when triggered by a fork, specifically so a malicious fork PR can't exfiltrate your secrets. `pull_request_target` runs with the *base* repo's permissions and secrets, even for fork PRs — a legitimate mechanism for things like auto-labeling, but if that workflow also checks out and executes the fork's code (e.g. `actions/checkout@v4` with `ref: ${{ github.event.pull_request.head.sha }}`), you've handed an anonymous fork author your secrets. This has been a real, repeated attack pattern, not a theoretical one — [GitHub Actions security checklist for supply chain attacks](https://corgea.com/learn/github-actions-security-checklist) covers this and "poisoned pipeline execution" more broadly.

**Self-hosted runners** trade GitHub-hosted convenience for your own infrastructure — necessary for private network access, specific hardware, or cost at scale — but a self-hosted runner attached to a *public* repo is a known attack surface: anyone who can open a PR can potentially get arbitrary code execution on your runner unless workflows are carefully scoped (e.g. `pull_request` from forks never touches self-hosted runners without review).

**Concurrency limits on deploy jobs** (extending [Part 1](#part-1-advanced)) prevent two deploys racing each other:

```yaml
concurrency:
  group: deploy-production          # NOT keyed on github.ref — one deploy at a time, period
  cancel-in-progress: false         # let an in-flight deploy finish; queue the next one
```

[↑ back to top](#table-of-contents)

<a id="part-7-mastery"></a>

### Mastery

**Organization-level Actions policy** lets you allowlist specific Actions (or require SHA pinning org-wide) rather than relying on every repo's maintainers remembering to do it individually — the difference between a best practice that exists in a doc and one that's actually enforced. Configured under org **Settings → Actions → General**.

**Least-privilege `GITHUB_TOKEN` as an org default.** Setting the org default to "read" (not "read and write") forces every workflow to explicitly declare `permissions:` for anything beyond reading — turning silent over-privilege ([Part 3](#part-3-working-knowledge)) into a visible, reviewable line in each workflow file.

**Cost and performance at scale:** GitHub-hosted runners bill per minute per job (not per workflow) — a 10-job matrix that each take 2 minutes costs 20 runner-minutes even though wall-clock time was ~2 minutes. Aggressive caching ([Part 2](#part-2-working-knowledge)) and `concurrency` cancellation ([Part 1](#part-1-advanced)) are the two levers with the highest cost impact, in that order.

[↑ back to top](#table-of-contents)

---

## Part 8: Internals Deep Dive

**How a job actually runs.** When a workflow triggers, GitHub's backend evaluates the trigger match, expands any `matrix:`, resolves `if:` conditions at the job level, and only *then* requests a runner (ephemeral VM for GitHub-hosted, a registered process for self-hosted). The runner polls GitHub's Actions service for its assigned job, downloads the workflow's compiled step list, and executes steps sequentially inside that one VM — which is why state (installed packages, files written) persists step-to-step within a job but never across jobs (each job = a fresh VM).

```
GitHub backend                          Runner (ephemeral VM)
──────────────                          ─────────────────────
1. event received
2. match workflow triggers
3. expand matrix → N jobs
4. evaluate job-level if:
5. queue job(s)          ──dispatch──▶  6. runner picks up job
                                        7. execute steps 1..N sequentially
                                        8. report status back
9. record conclusion      ◀──status───
10. evaluate needs: for downstream jobs
```

**The expressions/contexts engine** (`${{ ... }}`) is evaluated by the backend before dispatch for job-level fields (`if`, `matrix`, `env` at job scope) but evaluated *by the runner* for step-level fields — this split matters because step-level expressions can reference `steps.<id>.outputs`, which literally doesn't exist yet until a prior step has run inside that same VM.

**`GITHUB_TOKEN` internals:** it's minted per-job (not per-workflow) as a JWT-derived installation token scoped to the single repo, with an expiry tied to job completion (capped regardless, currently well under a day) — this is why a long-running self-hosted job that outlives the token's expiry starts failing `git push` / API calls partway through, a real gotcha for slow self-hosted pipelines.

[Reference for GitHub Actions](https://docs.github.com/en/actions/reference) · [Workflows and actions reference](https://docs.github.com/en/actions/reference/workflows-and-actions)

[↑ back to top](#table-of-contents)

---

## Cheat Sheets

**Triggers**

| Trigger | Fires on |
|---|---|
| `push` | commits pushed to matching branches |
| `pull_request` | PR opened/synced/reopened against the repo |
| `pull_request_target` | same, but runs with base-repo permissions/secrets — ⚠️ see [Part 7](#part-7-advanced) |
| `workflow_dispatch` | manual "Run workflow" button, supports typed `inputs:` |
| `schedule` | cron expression, UTC only |
| `workflow_call` | invoked by another workflow (reusable workflow) |

**Key contexts**

| Context | Common use |
|---|---|
| `github.ref` / `github.sha` | current branch/tag ref, current commit |
| `github.event_name` | `push`, `pull_request`, etc. |
| `runner.os` | `Linux`, `Windows`, `macOS` |
| `matrix.*` | current cell's variable values |
| `needs.<job>.outputs.*` | data passed from an upstream job |
| `secrets.*` | encrypted secret values |
| `steps.<id>.outputs.*` | data passed from an upstream step, same job |

**Artifacts/caching quick reference**

| Task | Action |
|---|---|
| Cache dependencies | `actions/cache@v4` (or built-in `cache:` on `setup-node`/`setup-python`) |
| Pass files between jobs | `actions/upload-artifact@v4` + `actions/download-artifact@v4` |
| ⚠️ `v3` of either artifact action | retired — will fail, not warn |

**Security checklist**

| Practice | Why |
|---|---|
| Pin third-party Actions to full SHA | tags are mutable; SHAs aren't |
| Declare explicit `permissions:` | don't inherit an unreviewed default |
| Use OIDC instead of long-lived cloud keys | short-lived, scoped, nothing to leak at rest |
| Avoid `pull_request_target` + checkout fork code | classic secret-exfiltration pattern |
| Environment-scope deploy secrets | not every workflow needs prod access |

[↑ back to top](#table-of-contents)

---

## Suggested Learning/Reference Order

1. **Part 0 → Part 1 (Beginner/Working Knowledge)** — get one real workflow running and understand triggers/jobs/steps. Don't skip the Try Its; the mental model clicks faster hands-on than read-only.
2. **Part 2 (Beginner/Working Knowledge)** — caching and artifacts, since almost every real pipeline needs both within its first few iterations.
3. **Part 4 (Beginner/Working Knowledge)** — wire up branch protection so your CI job actually *means* something, before layering on deployment.
4. **Part 3 (Beginner → Advanced)** — secrets and environments, needed the moment you add any deploy step.
5. **Part 5 (Beginner/Working Knowledge)** — deployment strategies, once you have something worth deploying.
6. **Part 1 (Advanced/Mastery) + Part 2 (Advanced)** — matrix builds, concurrency, the `v3`→`v4` artifact migration — revisit once your pipeline has more than one job type.
7. **Part 6** — reusable workflows/composite actions, once you notice yourself copy-pasting the same steps across 2+ workflow files.
8. **Part 3 (Mastery) + Part 7** — OIDC and production hardening, before anything in this pipeline touches real production credentials or a public-facing self-hosted runner.
9. **Part 8** — internals, whenever a bug forces you to understand *why* (e.g. debugging a token-expiry failure on a long self-hosted job).

[↑ back to top](#table-of-contents)

---

## Quick Self-Check

- Why do jobs run in parallel by default, and what keyword makes one wait on another?
- What's the practical difference between `actions/upload-artifact@v3` and `@v4` that would actually break a matrix-job workflow?
- Why does `if: github.event_name == 'push'` matter for a workflow that also triggers on `pull_request`?
- What's the actual risk if a required-check workflow has `paths:` filtering on its trigger?
- What does declaring `permissions: contents: read` at the top of a workflow protect against, concretely?
- Walk through the OIDC handshake from job start to a cloud provider handing back credentials — what expires, and when?
- When would you reach for a composite action versus a reusable workflow?
- Why is pinning to a full SHA meaningfully more secure than pinning to a version tag, given that both look "pinned" at a glance?
- What's the concrete difference in risk between `pull_request` and `pull_request_target` when a workflow also checks out the PR's code?
- Why can a slow self-hosted job start failing partway through with auth errors, even though nothing in the workflow changed?

[↑ back to top](#table-of-contents)
