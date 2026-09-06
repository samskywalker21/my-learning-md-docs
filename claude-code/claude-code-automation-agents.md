# Claude Code — Automation & Agents (Parts 13–14)

Running Claude without watching it: headless invocations, CI, scheduled and event-driven work, many agents at once, and packaging a setup so other repositories and people get it too.

> **Spec:** this doc follows the canonical spec in [`claude-code-mastery-guide.md`](./claude-code-mastery-guide.md#about-this-document). Written against **Claude Code v2.1.263**, verified **September 6, 2026**.
>
> **Prerequisite:** [Permissions & Security (Parts 8–9)](./claude-code-permissions-security.md). Everything here runs with nobody watching, which is exactly when the permission model stops being theoretical.

---

## Table of Contents

- [Part 13 — Headless and CI](#part-13--headless-and-ci)
  - [13.1 Beginner: `claude -p`](#131-beginner-claude--p)
  - [13.2 Working Knowledge: output formats](#132-working-knowledge-output-formats)
  - [13.3 Working Knowledge: `--bare`, and why CI needs it](#133-working-knowledge---bare-and-why-ci-needs-it)
  - [13.4 Advanced: permissions with nobody watching](#134-advanced-permissions-with-nobody-watching)
  - [13.5 Advanced: budgets, caps, and lifecycle](#135-advanced-budgets-caps-and-lifecycle)
  - [13.6 Advanced: GitHub Actions](#136-advanced-github-actions)
  - [13.7 Mastery: fan-out and the stream protocol](#137-mastery-fan-out-and-the-stream-protocol)
  - [13.8 Part 13 cheat sheet](#138-part-13-cheat-sheet)
- [Part 14 — Parallelism and Packaging](#part-14--parallelism-and-packaging)
  - [14.1 Beginner: which kind of parallelism](#141-beginner-which-kind-of-parallelism)
  - [14.2 Working Knowledge: worktrees](#142-working-knowledge-worktrees)
  - [14.3 Working Knowledge: background agents and agent view](#143-working-knowledge-background-agents-and-agent-view)
  - [14.4 Advanced: dynamic workflows](#144-advanced-dynamic-workflows)
  - [14.5 Advanced: scheduled, goal-driven, and event-driven work](#145-advanced-scheduled-goal-driven-and-event-driven-work)
  - [14.6 Advanced: cross-session messaging](#146-advanced-cross-session-messaging)
  - [14.7 Mastery: plugins as the distribution layer](#147-mastery-plugins-as-the-distribution-layer)
  - [14.8 Part 14 cheat sheet](#148-part-14-cheat-sheet)

---

## Part 13 — Headless and CI

<a id="part-13"></a>

### 13.1 Beginner: `claude -p`

Add `-p` (or `--print`) to run non-interactively: Claude does the work, prints the result, and exits ([headless](https://code.claude.com/docs/en/headless)).

```bash
claude -p "What does the auth module do?"
claude -p "Find and fix the bug in auth.py" --allowedTools "Read,Edit,Bash"
cat build-error.txt | claude -p 'concisely explain the root cause of this build error' > output.txt
```

Exit code 0 on success, non-zero on failure. An invalid flag is reported on stderr before the run starts; a failure *inside* the run (missing authentication, say) is printed as the result on stdout.

> **Piped stdin is capped at 10MB.** Beyond that, Claude Code exits with an error. Write the content to a file and reference the path in your prompt instead.

Skills and custom commands work in `-p` — include `/skill-name` in the prompt string and Claude Code expands it. Terminal-only built-ins like `/login` don't. `/model`, `/effort`, `/fast`, `/color`, and `/rename` accept a value as an argument (`/model sonnet`), and `/config key=value` changes a setting from a `-p` run.

### 13.2 Working Knowledge: output formats

```bash
claude -p "Summarize this project"                              # text (default)
claude -p "List all API endpoints" --output-format json         # one JSON object
claude -p "Analyze this log file" --output-format stream-json --verbose
```

| Format | Shape |
|---|---|
| `text` | Plain text |
| `json` | A single object with `result`, `session_id`, usage, and `total_cost_usd` plus a per-model cost breakdown |
| `stream-json` | Newline-delimited JSON, one object per line, starting with an init event and ending with a `result` message |

```bash
claude -p "Summarize this project" --output-format json | jq -r '.result'
```

> Cost figures are **client-side estimates** and can differ from your actual bill.

**Structured output** against a schema:

```bash
claude -p "Extract the main function names from auth.py" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}},"required":["functions"]}' \
  | jq '.structured_output'
```

The structured value lands in `structured_output`, alongside the usual metadata. An invalid schema exits with `Error: --json-schema is not a valid JSON Schema` and the validator's diagnostic. Claude Code accepts the `format` keyword (`"format": "email"`) but treats it as an **annotation and does not enforce it**. Before v2.1.205 an invalid schema was silently ignored and unstructured text returned — worth knowing if you're on an old pin.

**Streaming tokens** as they generate:

```bash
claude -p "Write a poem" --output-format stream-json --verbose --include-partial-messages | \
  jq -rj 'select(.type == "stream_event" and .event.delta.type? == "text_delta") | .event.delta.text'
```

**Customising the system prompt:**

```bash
gh pr diff "$1" | claude -p \
  --append-system-prompt "You are a security engineer. Review for vulnerabilities." \
  --output-format json
```

`--append-system-prompt-file` reads from a file; `--system-prompt` / `--system-prompt-file` **replace** the prompt entirely.

**Continuing a conversation:**

```bash
session_id=$(claude -p "Start a review" --output-format json | jq -r '.session_id')
claude -p "Continue that review" --resume "$session_id"
```

These can run from different directories — Claude Code finds a session by ID across every project on the machine (v2.1.223+).

### 13.3 Working Knowledge: `--bare`, and why CI needs it

`--bare` skips auto-discovery of hooks, skills, custom commands, subagents, plugins, MCP servers, auto memory, and `CLAUDE.md`.

```bash
claude --bare -p "Summarize README.md" --allowedTools "Read"
```

**Two reasons it matters, and the second is the important one:**

1. **Reproducibility.** A hook in a teammate's `~/.claude` or an MCP server in the project's `.mcp.json` won't run, because bare mode never reads them. Same result on every machine.
2. **Safety.** Without `--bare`, a `-p` session runs the hooks in a project's `.claude/settings.json` and connects the servers in its `.mcp.json` **even in a folder you have never trusted** — a `-p` session shows no workspace trust dialog and no per-server approval prompt.

> **The docs say `--bare` is the recommended mode for scripted and SDK calls, and that it will become the default for `-p` in a future release.** Write it now.

In bare mode Claude Code **never reads OAuth credentials or the system keychain** — set `ANTHROPIC_API_KEY`, or supply an `apiKeyHelper` in the `--settings` JSON. (Bedrock, Google Cloud's Agent Platform, and Microsoft Foundry still read their own provider credentials.) Claude gets the Bash, file read, and file edit tools; everything else you pass explicitly:

| To load | Use |
|---|---|
| System prompt additions | `--append-system-prompt`, `--append-system-prompt-file` |
| Settings | `--settings <file-or-json>` |
| MCP servers | `--mcp-config <file-or-json>` (add `--strict-mcp-config`) |
| Custom agents | `--agents <json>` |
| A plugin | `--plugin-dir <path>`, `--plugin-url <url>` |

One partial exception: bare mode loads skills from a `--add-dir` directory's `.claude/skills/`, but still skips its `.claude/commands/` and `.claude/agents/`.

<a id="p13-permissions"></a>

### 13.4 Advanced: permissions with nobody watching

**A `-p` run starts in Manual mode on every plan**, regardless of what an interactive session on the same machine would do. Pass what you want ([permission modes](https://code.claude.com/docs/en/permission-modes)):

```bash
claude -p "Run the test suite and fix any failures" --allowedTools "Bash,Read,Edit"
claude -p "Apply the lint fixes" --permission-mode acceptEdits
claude -p "fix all lint errors" --permission-mode auto
claude -p "run the test suite" --permission-mode dontAsk --allowedTools "Bash(npm test)" "Read"
```

| Mode | For |
|---|---|
| `auto` | Classifier reviews instead of you. On a `-p` run, repeated classifier blocks **don't stop the run** — see [when auto mode falls back](https://code.claude.com/docs/en/permission-modes#when-auto-mode-falls-back) |
| `dontAsk` | Denies anything outside `permissions.allow` and the read-only command set. **The right default for a locked-down job** |
| `acceptEdits` | Writes files and common filesystem commands without prompting; other shell commands still need an allow rule |

> **`--allowedTools` uses permission rule syntax**, so all of [Part 8](./claude-code-permissions-security.md)'s wildcard rules apply. The trailing space matters: `Bash(git diff *)` allows anything starting `git diff`; **`Bash(git diff*)` would also match `git diff-index`.**

**`--permission-prompts none`** (v2.1.259+) is for jobs where nobody can answer:

```bash
claude -p "Update the dependency pins and run the tests" \
  --permission-mode auto --permission-prompts none
```

With it, the run doesn't consult or wait on a permission host. Anything that would prompt is denied unless a `PermissionRequest` hook allows it, Claude is told nobody can approve and not to retry, and the run continues. It also **removes tools that need a human**, so Claude can't call `AskUserQuestion`, and any MCP elicitation no `Elicitation` hook answers is cancelled. With `--output-format stream-json`, denials appear as `permission_denied` system messages and are listed in the final result's `permission_denials`.

> **Wrong vs. right — a nightly job.**
>
> ```bash
> # Wrong: interactive habits in an unattended job
> claude -p "$PROMPT" --dangerously-skip-permissions
> ```
>
> No trust dialog, no classifier, no boundary — running on a repo whose contents may include text written by someone else.
>
> ```bash
> # Right: allowlist, not denylist
> claude --bare -p "$PROMPT" \
>   --permission-mode dontAsk \
>   --allowedTools "Bash(gh pr view *)" "Bash(gh pr comment *)" "Read" \
>   --permission-prompts none \
>   --max-turns 15 \
>   --max-budget-usd 2.00
> ```

### 13.5 Advanced: budgets, caps, and lifecycle

| Flag | Does |
|---|---|
| `--max-turns 3` | Cap agentic turns |
| `--max-budget-usd 5.00` | Cap spend on API calls |
| `--fallback-model sonnet,haiku` | Fall back when the primary is overloaded |
| `--no-session-persistence` | Don't write a transcript for this run |
| `--session-id <uuid>` | Use a specific session ID |
| `--init` / `--maintenance` / `--init-only` | Run `Setup` hooks with that matcher before the session; `--init-only` runs Setup + SessionStart then exits |
| `--verbose` | Required with `stream-json`; turn it off in production |

**Background tasks at exit.** A background Bash task Claude started (a dev server, a watch build) is terminated about **five seconds** after the final result and stdin close. A background **subagent or workflow** is different: `claude -p` stays open until it completes, because its result is part of the output. That wait ends after **10 minutes of continuous idle waiting** by default so a stuck subagent can't hold the process forever — change it with `CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS`, or `0` for no ceiling. A `Monitor` watch is waited on until it times out (default five minutes after Claude starts it) or the ten-minute cap ends the wait.

**Signals.** SIGTERM exits with code **143**, leaves the in-progress turn unfinished with no result recorded, terminates the process tree of any running Bash command, runs `SessionEnd` hooks, and exits. To *end* the turn instead, send SIGINT (or call the SDK's `interrupt()`) first. Resuming the session continues the turn SIGTERM left unfinished.

**CI gates worth wiring up**, all from the `system/init` event in a `stream-json` stream:

| Field | Fail the build when |
|---|---|
| `plugin_errors` | Non-empty — a plugin didn't load |
| `mcp_server_errors` | Non-empty — a `--mcp-config` entry was skipped by validation |
| `mcp_servers[].status` | A required server isn't connected |

Claude Code validates each `--mcp-config` entry at startup and **skips invalid ones while the run continues and exits cleanly** — so without checking these fields, a broken server config produces a green build with a silently degraded agent. When run by hand it also prints a stderr warning (`Warning: 1 MCP server skipped due to invalid config:`), but that warning is suppressed when stderr is captured, as a CI runner does.

The `system/init` event also carries an optional `capabilities` array — **feature-detect with that rather than comparing version strings** (v2.1.205+).

`system/api_retry` events report retryable API failures with `attempt`, `max_retries`, `retry_delay_ms`, `error_status`, and an `error` category (`rate_limit`, `overloaded`, `authentication_failed`, …) — useful for showing retry progress in your own UI.

### 13.6 Advanced: GitHub Actions

Two setup paths ([GitHub Actions](https://code.claude.com/docs/en/github-actions)):

- **Quick**: run `/install-github-app` in the repo. It installs the GitHub App, stores the credential as a repository secret, pushes a branch with the workflow files, and opens a PR. Needs the `gh` CLI and repo admin access.
- **Manual**: install [the Claude GitHub App](https://github.com/apps/claude), add a secret, copy a workflow file.

**Credentials:** `ANTHROPIC_API_KEY` (a Console API key) or `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`, tied to a subscription). For a secret shared across repos, prefer the API key — an OAuth token belongs to whoever generated it. To avoid a long-lived secret entirely, use **workload identity federation**: set `anthropic_federation_rule_id`, `anthropic_organization_id`, and optionally `anthropic_service_account_id` / `anthropic_workspace_id`, and grant the workflow `id-token: write`.

**Two modes, detected from your config:**

- **Interactive** — no `prompt` input. Claude waits for the trigger phrase (`@claude`) in a comment, review, or new issue, and replies in a comment on the triggering item.
- **Automation** — a `prompt` input is present. Claude runs on any GitHub event without waiting for a mention. Results go to the workflow log unless the prompt tells Claude to post and it has a tool that can.

**Responding to `@claude`:**

```yaml
name: Claude Code
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
jobs:
  claude:
    if: contains(github.event.comment.body, '@claude')
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
      actions: read
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 1
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

The non-boilerplate lines: `id-token: write` is **required** for the action's default GitHub App authentication; `actions: read` lets Claude read CI results; the `if` keeps runners from starting on unrelated comments.

**Running a skill on every PR:**

```yaml
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          plugin_marketplaces: "https://github.com/anthropics/claude-code.git"
          plugins: "code-review@claude-code-plugins"
          prompt: "/code-review:code-review --comment ${{ github.repository }}/pull/${{ github.event.pull_request.number }}"
          claude_args: '--allowedTools "mcp__github_inline_comment__create_inline_comment"'
```

> Keep the `claude_args` line **even though the skill's own `allowed-tools` names the same tool** — the action starts the inline-comment MCP server only when `--allowedTools` in `claude_args` names it. That's a real gotcha, called out in the docs.

**Two safety checks run before Claude starts**, and the run fails if either rejects:

- **Write access** — on issue and PR events the triggering user must have write access. `allowed_non_write_users` plus your own `github_token` widens it. Events with no author, like `schedule`, skip the check.
- **Human actor** — bot actors are rejected unless listed in `allowed_bots`, which stops loops. This also applies to scheduled runs, which GitHub attributes to whoever last changed the `cron`.

**Common inputs:** `prompt`, `claude_args`, `github_token`, `plugin_marketplaces`, `plugins`, `settings`, `trigger_phrase`, `use_bedrock` / `use_vertex` / `use_foundry`.

**Cost control:** `--max-turns` in `claude_args`, workflow-level timeouts, GitHub concurrency limits, a concise `CLAUDE.md` (read on every run), and specific `@claude` requests.

**Two troubleshooting facts worth knowing before you hit them:**

- **CI doesn't run on Claude's commits** if you pass `github_token: ${{ secrets.GITHUB_TOKEN }}` — GitHub doesn't trigger workflows on commits made with the default token. Remove it so the action authenticates as the Claude GitHub App, or pass a custom app token.
- On public repositories, GitHub **withholds secrets from fork PR runs**, so a review workflow only runs on branches in the same repository.

**If your workflows still say `@beta`:** change to `@v1`, remove the `mode` input, rename `direct_prompt` → `prompt`, and move `max_turns`/`model` into `claude_args` (`custom_instructions` becomes `--append-system-prompt`).

<a id="p13-mastery"></a>

### 13.7 Mastery: fan-out and the stream protocol

**Fan out across many files** — the loop pattern from the official best practices:

```bash
# 1. Have Claude write the task list
claude -p "list all 2,000 Python files that need migrating and save the list to files.txt"

# 2. Loop, with permissions scoped tightly
for file in $(cat files.txt); do
  claude --bare -p "Migrate $file from Python 2 to Python 3. Return OK or FAIL." \
    --allowedTools "Edit,Bash(git commit *)" \
    --max-turns 10
done

# 3. Refine on 2-3 files, then run the full set
```

Or, inside a session in a git repo, `/batch <instruction>` does the same thing with subagents: it researches, decomposes the work into **5–30 independent units**, presents a plan, then spawns one background subagent per unit **in its own git worktree**, each implementing, testing, and opening a pull request.

Piping into another tool:

```bash
claude -p "<your prompt>" --output-format json | your_command
```

**Following subagent messages in the stream.** Subagent messages appear as `assistant` and `user` messages whose `parent_tool_use_id` is the ID of the tool call that spawned them; main-conversation messages carry `null`. By default only subagent `tool_use` and `tool_result` blocks are emitted — pass `--forward-subagent-text` (or `CLAUDE_CODE_FORWARD_SUBAGENT_TEXT`, v2.1.211+) to also emit text and thinking blocks, so you can reconstruct each subagent's transcript. This forwards messages **at every nesting depth**, with each nested agent carrying its spawner's tool-call ID, so you can rebuild the full tree.

Two more stream flags: `--include-hook-events` (hook lifecycle events in the stream) and `--replay-user-messages` (re-emit stdin user messages on stdout, for stream-json input).

**Slow consumers.** If your consumer reads slowly, Claude Code waits for queued output to drain before exiting, scaling with what's queued, capped at **30 seconds**. Before v2.1.214 that cap was ~2 seconds, which could truncate the end of a large response.

> ### Real Scenario — the green build that fixed nothing
>
> A team adds a CI job that runs Claude to auto-fix lint errors on every PR branch and push the result. It's been green for two weeks. Then someone notices the lint errors are still there — and have been the whole time.
>
> ```yaml
> - run: |
>     claude -p "Fix all lint errors in the changed files and commit" \
>       --permission-mode dontAsk \
>       --mcp-config ./ci-mcp.json
> ```
>
> **What happened, in three layers:**
>
> 1. `./ci-mcp.json` had an entry with a `url` and **no `type`**. Claude Code validated it at startup, **skipped it**, and continued — exit code 0. The stderr warning was captured by the runner and never displayed.
> 2. `dontAsk` denies anything not in `permissions.allow`. Nothing was allowed, and `--allowedTools` wasn't passed, so every `Edit` and `Bash` call was denied. Claude reported that it couldn't make changes — as the *result text*, on stdout, which nothing read.
> 3. The job's last step was `git push` in a separate `run:` block that succeeded because there was nothing to push. **Exit code 0 the whole way down.**
>
> **The fix, which is really one idea applied three times — assert, don't assume:**
>
> ```yaml
> - run: |
>     claude --bare -p "Fix all lint errors in the changed files" \
>       --permission-mode dontAsk \
>       --allowedTools "Read" "Edit" "Bash(npm run lint*)" \
>       --mcp-config ./ci-mcp.json --strict-mcp-config \
>       --output-format stream-json --verbose \
>       --max-turns 20 | tee claude.jsonl
>
>     # Fail on a skipped MCP server or a failed plugin load
>     jq -e 'select(.type=="system" and .subtype=="init")
>            | (.mcp_server_errors // []) + (.plugin_errors // []) | length == 0' \
>            claude.jsonl > /dev/null
>
>     # Fail if the run itself reported an error
>     jq -e 'select(.type=="result") | .is_error != true' claude.jsonl > /dev/null
>
>     # And the real check: does lint actually pass now?
>     npm run lint
> ```
>
> **The generalisable lesson:** a headless Claude run exiting 0 means *the process finished*, not *the work happened*. The only trustworthy CI gate is the same check a human would run — `npm run lint` at the end. Everything else is diagnostics for when that check fails.

<a id="138-part-13-cheat-sheet"></a>

### 13.8 Part 13 cheat sheet

| Flag | Effect |
|---|---|
| `-p` / `--print` | Non-interactive |
| `--bare` | Skip auto-discovery — **recommended for CI**, future default |
| `--output-format text\|json\|stream-json` | Output shape (`stream-json` needs `--verbose`) |
| `--json-schema '<schema>'` | Structured output in `structured_output` (`format` not enforced) |
| `--allowedTools` / `--disallowedTools` | Permission rules for the run |
| `--permission-mode dontAsk` | Deny anything not allowlisted |
| `--permission-prompts none` | Nobody can approve; removes human-input tools (v2.1.259+) |
| `--max-turns` · `--max-budget-usd` | Caps |
| `--continue` / `--resume <id>` | Continue a conversation |
| `--no-session-persistence` | No transcript |
| `--strict-mcp-config` | Only `--mcp-config` servers |
| `--forward-subagent-text` | Subagent text and thinking in the stream |
| `--include-partial-messages` | Token-level streaming |

| Fact | |
|---|---|
| Starting permission mode for `-p` | **Manual, on every plan** |
| Workspace trust dialog in `-p` | **None** — hence `--bare` |
| Piped stdin cap | 10MB |
| Background Bash at exit | Killed ~5s after the result |
| Background subagent/workflow at exit | Waited on; 10-min idle ceiling (`CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS`) |
| SIGTERM | Exit 143; turn unfinished; `SessionEnd` hooks run |
| CI gates | `plugin_errors`, `mcp_server_errors`, `mcp_servers[].status`, `result.is_error` — plus the real check |

[↑ Back to top](#table-of-contents)

---

## Part 14 — Parallelism and Packaging

<a id="part-14"></a>

### 14.1 Beginner: which kind of parallelism

Five ways to have more than one Claude working, differing mainly in **who coordinates** ([run agents in parallel](https://code.claude.com/docs/en/agents)):

| Approach | Gives you | Use when |
|---|---|---|
| **[Subagents](./claude-code-extensibility.md)** | Delegated workers inside one session that return a summary | A side task would flood your conversation with output you won't reference again |
| **Agent view** | One screen to dispatch and monitor background sessions (`claude agents`). *Research preview* | Several independent tasks you want to hand off and check on |
| **Agent teams** | Coordinated sessions with a shared task list and messaging, run by a lead. *Experimental, off by default* | You want Claude to split a project, assign it, and keep workers in sync |
| **Dynamic workflows** | A script that runs many subagents and cross-checks results | A job outgrows a handful of subagents, or findings need verifying |
| **Worktrees** | Separate git checkouts so parallel sessions don't collide | Any of the above, when tasks touch the same files |

Plus **cross-session messaging**, which lets sessions pass findings to each other, and `/batch`, a bundled skill that packages subagents + worktrees into one command.

**The three questions that pick for you:**

1. **Who coordinates?** Claude inside one conversation → subagents. You, checking back later → agent view. Claude planning and supervising a group → agent teams. A script instead of turn-by-turn judgement → dynamic workflows.
2. **Do the workers need to talk?** Cross-session messaging between sessions you run. Subagents report to whoever spawned them. Agent-view sessions report only to you. Teammates message each other directly.
3. **Do the tasks touch the same files?** Then isolate with worktrees. Agent view does this automatically; agent teams **don't**, so partition the work by file ownership.

> **Running several sessions or subagents at once multiplies token usage.** Ten parallel sessions consume quota roughly ten times faster.

**Checking on running work**, since the commands differ:

| Approach | Command |
|---|---|
| Background sessions | `claude agents` |
| Anything backgrounded in the current session | `/tasks` |
| Dynamic workflows | `/workflows` |
| Subagents in this session | Named background subagents appear in the `@` typeahead with status |

> `/agents` (the slash command) prints a notice about subagent file locations. `claude agents` (the shell command) opens agent view. Despite the names, they are unrelated.

### 14.2 Working Knowledge: worktrees

A [git worktree](https://git-scm.com/docs/git-worktree) is a separate working directory with its own files and branch, sharing the repository's history and remote.

```bash
claude --worktree feature-auth      # or -w
claude --worktree                   # generates a name like bright-running-fox
claude --worktree "#1234"           # branch from a PR/MR (quote it!)
```

By default the worktree goes in `.claude/worktrees/<name>/` at the repo root, on a new branch `worktree-<name>` ([worktrees](https://code.claude.com/docs/en/worktrees)).

> **Add `.claude/worktrees/` to `.gitignore`** so worktree contents don't show up as untracked files in your main checkout.

Interactive runs require **workspace trust** — run `claude` once in the directory to accept the dialog, or `--worktree` exits with an error. `claude -p --worktree` skips the trust check.

**Setup and gitignored files.** A worktree is a fresh checkout, so `.env` and friends aren't there. `.worktreeinclude` at the project root copies them in automatically, using `.gitignore` syntax; only files that match **and are gitignored** are copied:

```text
.env
.env.local
config/secrets.json
```

**Cleanup on exit:**

| State | What happens |
|---|---|
| Clean, unnamed session | Worktree and branch removed automatically |
| Clean, named session | You're prompted, so you can keep it |
| Has changes or new commits | You're prompted to keep or remove — removing deletes the work |
| `-p` run | **No exit prompt, no cleanup.** Remove with `git worktree remove` (`git worktree unlock` first if locked) |

**Configuration:**

```json
{ "worktree": { "baseRef": "head" } }
```

`"fresh"` (default) branches from the remote's default branch, so you start clean. `"head"` branches from your current local `HEAD`, carrying unpushed commits — use it when isolating subagents that need to work on in-progress changes. You **can't** set it to a branch name; use `git worktree add` for that.

**What a worktree shares with the main checkout:** the repository's `.git` directory (so `git commit` works, and the sandbox allows those writes), project-scope plugins (v2.1.200+), and **permission approvals** — a "don't ask again" in a worktree saves to the main checkout's `.claude/settings.local.json` (v2.1.211+), so it applies everywhere and survives the worktree's removal.

**Isolation enforcement.** While a session is in a worktree, Claude Code blocks: edits targeting the main checkout; Bash/PowerShell/Monitor commands whose working directory resolves to the main checkout; git redirected into the main checkout (via `git -C`, `--git-dir`, `GIT_DIR`, `GIT_WORK_TREE`, or a `cd`); and commands whose shape makes it unverifiable that git stays inside the worktree. **That last check can't be turned off** — Claude Code tells Claude how to rewrite the refused command.

> **A hook gotcha specific to worktrees:** `${CLAUDE_PROJECT_DIR}` **stays at the project root** where the session started, so `${CLAUDE_PROJECT_DIR}/.claude/hooks/check.sh` still runs the main checkout's script. The hook's `cwd` input field follows Claude into the worktree. Read `cwd` when a hook needs the worktree path.

**Git LFS caveat.** If you set up LFS with `git lfs install --local`, a Claude-created worktree contains **pointer files**, not real files — Claude Code deliberately skips the repository's own filter drivers when creating a worktree, because a filter driver is a shell command and anything that can write to the repo (including Claude) could have put one there. Run `git lfs pull` inside the worktree. A plain `git lfs install` writes to your global config and isn't affected.

### 14.3 Working Knowledge: background agents and agent view

**Background agents** are sessions that keep running without a terminal, hosted by a supervisor process that survives terminal closure and machine sleep. **Agent view** (`claude agents`) is the screen for dispatching and watching them ([agent view](https://code.claude.com/docs/en/agent-view)). *Research preview.*

```bash
claude agents                              # open agent view
claude agents --json                       # machine-readable listing
claude agents --cwd ~/path                 # scope to a directory
claude agents --permission-mode plan --model opus --effort high   # dispatch defaults

claude --bg "investigate the flaky test"   # start one from your shell
claude --bg --name "flaky-test-fix" "investigate the test"
claude --bg --exec 'pytest -x'             # a PTY-backed shell job
```

From inside a session:

| Command | Does |
|---|---|
| `/background` (`/bg`) | Detach this session to the background and free the terminal |
| `/bg <prompt>` | Same, sending one more instruction first |
| `/fork [prompt]` | Copy the conversation into a **new** background session, keep working here |

**Session states:** working (animated), needs input (yellow), idle (dimmed), completed (green), failed (red), stopped (grey). The icon shape shows process status: `✻`/`✽` running, `∙` exited but resumable, `✢` a `/loop` session sleeping between iterations.

**Agent view keys:**

| Key | Action |
|---|---|
| `↑` `↓` | Move between rows |
| `Enter` / `→` | Attach (or dispatch, if there's text in the input) |
| `Space` | Peek panel — see the current question or output without attaching |
| `Ctrl+T` · `Ctrl+R` | Pin (keep the process running while idle) · rename |
| `Ctrl+S` · `Ctrl+X` | Toggle grouping · stop (press again to delete) |
| `?` · `Esc` | All shortcuts · close/exit |

Detaching from an attached session: `←` on an empty prompt, `Ctrl+Z`, or `Ctrl+C` twice. **Detaching never stops the session.**

**Shell management:** `claude attach <id>`, `claude logs <id>`, `claude stop <id>`, `claude respawn <id>` (or `--all`), `claude rm <id>`, `claude daemon status`, `claude daemon stop --any [--keep-workers]`.

**Filtering** — type in the dispatch input to filter instead of dispatch: `a:my-agent`, `s:working`, `s:blocked`, `#1234`, or a URL.

**File isolation is automatic.** Background sessions move into worktrees under `.claude/worktrees/` before editing, so parallel sessions don't conflict. On completion with changes, Claude commits and pushes branches, opens draft PRs where appropriate, and **never force-pushes or merges to `main`/`master`**. Disable with:

```json
{ "worktree": { "bgIsolation": "none" } }
```

**What carries over when you background a session:** running background shell commands, backgrounded subagents, dynamic workflows, `/loop` scheduled tasks, automatic artifact replies, configuration flags (`--mcp-config`, `--settings`, `--add-dir`, `--fallback-model`), and directories added with `/add-dir`.

**Limitations:** rate limits apply and multiply; sessions are **local only** and stop on shutdown; commit before deleting a session in agent view; and it's a research preview, so the interface may change.

<a id="p14-workflows"></a>

### 14.4 Advanced: dynamic workflows

A dynamic workflow is a **JavaScript script that orchestrates many subagents**. Claude writes the script for the task you describe, and a runtime executes it in the background while your session stays responsive ([workflows](https://code.claude.com/docs/en/workflows)).

The distinction from everything else is **who holds the plan**:

| | Subagents | Skills | Agent teams | Workflows |
|---|---|---|---|---|
| What it is | A worker Claude spawns | Instructions Claude follows | A lead supervising peers | **A script the runtime executes** |
| Who decides what's next | Claude, turn by turn | Claude, following the prompt | The lead, turn by turn | **The script** |
| Where intermediate results live | Claude's context | Claude's context | A shared task list | **Script variables** |
| What's repeatable | The worker definition | The instructions | The team definition | **The orchestration itself** |
| Scale | A few per turn | Same | A handful of peers | **Dozens to hundreds per run** |
| Interruption | Restarts the turn | Restarts the turn | Teammates keep running | **Resumable in the same session** |

Because results live in script variables, **Claude's context holds only the final answer**. And because the plan is code, a workflow can apply a repeatable quality pattern: independent agents adversarially reviewing each other's findings before reporting, or drafting a plan from several angles and weighing them.

**Try the bundled one:**

```text
/deep-research What changed in the Node.js permission model between v20 and v22?
```

It fans out web searches across several angles, cross-checks sources, votes on each claim, and returns a cited report with unsurvived claims filtered out. Claims the verifiers *couldn't* check are listed as **unverified**, not refuted.

**Start one for your own task** — include `ultracode` in your prompt, or just ask:

```text
ultracode: audit every API endpoint under src/routes/ for missing auth checks
```

```text
use a workflow to migrate every component under src/components/ from JavaScript
to TypeScript, working on each file in its own isolated copy
```

> **The keyword is an opt-in only in a prompt you type yourself** — at the interactive prompt, an IDE panel, a Remote Control client, or an SDK app that stamps input as human. It does **not** start a workflow from a `-p` prompt, an unstamped SDK prompt, a scheduled task, or **a webhook payload or PR comment relayed into the conversation**. (Before v2.1.210 it did, from all of those.) That is a security boundary, not a convenience.

`Option+W` / `Alt+W` dismisses the highlight for a prompt. `/effort ultracode` makes Claude plan a workflow for **every** substantive task in the session — `xhigh` reasoning plus automatic orchestration.

**Watching a run:** `/workflows` lists running and completed runs. In the progress view: `↑`/`↓` select, `Enter`/`→` drill in, `f` filter by status, `p` pause/resume, `x` stop an agent or the run, `r` restart an agent, `s` **save the script as a command**.

**Approval depends on permission mode:**

| Mode | Prompted |
|---|---|
| Auto | First launch only; **Yes** records consent in user settings. Skipped entirely with ultracode on |
| Manual, acceptEdits | Every run, unless you chose "don't ask again" for that workflow in this project |
| Bypass permissions | Never |
| `claude -p`, Agent SDK | Never prompted — the Workflow tool call goes through normal permission evaluation. Allow it with a `Workflow` or `Workflow(<name>)` rule, auto mode, bypass, a `PreToolUse` hook, or a permission host |

**Runtime limits:**

| Constraint | Value |
|---|---|
| Concurrent agents | Up to **16**, fewer on fewer CPUs |
| Items per `parallel()` / `pipeline()` call | 4,096 (a longer list is rejected, not silently truncated) |
| Agents per run | **1,000** |
| Mid-run user input | Not possible — only agent permission prompts pause a run |
| Filesystem/shell from the script | None — agents do the work, the script coordinates |
| Module loading | `import()` fails before the run starts |

**The saved script's shape**, so you recognise what Claude generated:

```javascript
export const meta = {
  name: 'audit-routes',
  description: 'Audit every route handler for missing auth checks',
}

const found = await agent('List every .ts file under src/routes/.', {
  schema: { type: 'object', required: ['files'],
            properties: { files: { type: 'array', items: { type: 'string' } } } },
})

const audits = await pipeline(found.files, file =>
  agent(`Audit ${file} for missing authentication checks.`, { label: file }),
)

return audits.filter(Boolean)
```

`agent()` spawns one; `pipeline()` runs one per item; `parallel()` runs a set at once. `phase()` groups agents in the progress view and `log()` prints above them. An `agent()` call resolves to **`null`** if you stop it or it hits an unrecoverable API error, and `pipeline()` keeps that `null` — hence `.filter(Boolean)`.

> **`Date.now()`, `Math.random()`, and no-argument `new Date()` throw inside the script.** That's deliberate: a relaunched run must repeat the same `agent()` calls to reuse saved results. Pass a timestamp in through `args` instead.

**Saving and sharing.** `/workflows` → select → `s` saves to `.claude/workflows/` (shared) or `~/.claude/workflows/` (personal); it then runs as `/<name>`. Project beats personal on a name clash. Plugins ship workflows in a `workflows/` directory, namespaced as `/plugin-name:workflow-name`. Run `/workflow-authoring` (v2.1.248+) before editing a saved script, then `/reload-skills` to pick up changes.

**Resuming.** Select a paused run and press `p`. On relaunch, agents replay in start order: **completed** agents return saved results, but the first agent whose prompt differs runs again **and so does every agent after it**; agents **still running** when you stopped start over; a **failed** agent runs again along with everything that started after it. So a failure mid-fan-out reruns work that already finished.

**Cost.** Runs count toward your usage like any other session. Gauge with a small slice first. Claude Code shows a `Large workflow` warning when a run schedules more than 25 agents or projects past 1.5M tokens — advisory only, and suppressed under ultracode. The **size guideline** (`/config` → Dynamic workflow size, or `workflowSizeGuideline`) tells Claude what to aim for: `small` (<5), `medium` (<15, the default), `large` (<50), `unrestricted`.

**Turning them off:** `/config`, `"disableWorkflows": true`, or `CLAUDE_CODE_DISABLE_WORKFLOWS=1`. On Pro, workflows must be turned **on** from the Dynamic workflows row in `/config` first.

### 14.5 Advanced: scheduled, goal-driven, and event-driven work

Three different shapes of "keep going without me," and they solve genuinely different problems:

```text
  POLL on an interval        ──▶  /loop         "check the deploy every 5m"
  WORK toward a condition    ──▶  /goal         "keep going until tests pass"
  REACT to an external event ──▶  channels      CI pushes the failure to you
```

**`/loop`** re-runs a prompt while the session stays open ([scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks)):

| What you give it | Behaviour |
|---|---|
| `/loop 5m check the deploy` | Fixed schedule |
| `/loop check the deploy` | **Claude picks the interval** each iteration, 1 minute to 1 hour, based on what it observed — and prints the delay and its reason |
| `/loop` | Runs the built-in maintenance prompt, or your `loop.md` |

Units are `s`/`m`/`h`/`d`. Seconds round up to a minute (cron granularity); odd intervals like `7m` or `90m` round to the nearest clean cron step and Claude tells you what it picked.

The **built-in maintenance prompt** (bare `/loop`) works through: unfinished work from the conversation → the current branch's PR (review comments, failed CI, merge conflicts) → cleanup passes like bug hunts. It does not start new initiatives, and irreversible actions only proceed when they continue something the transcript already authorised. Replace it with `.claude/loop.md` (project, wins) or `~/.claude/loop.md` (user). Edits take effect on the next iteration; content past 25,000 bytes is truncated.

**Stopping:** `Esc` clears a self-paced loop's pending wakeup. In self-paced mode Claude can also end the loop itself by calling `ScheduleWakeup` with `stop: true`. Fixed-interval loops run until you cancel them or seven days pass.

**Facts that catch people:**

- **Recurring tasks expire after 7 days.** They fire one final time and delete themselves — bounding how long a forgotten loop can run.
- **Jitter.** Recurring tasks fire up to 30 minutes late (or half the interval, for sub-hourly jobs); one-shots at `:00` or `:30` fire up to 90 seconds early. The offset derives from the task ID, so it's stable. If timing matters, pick a minute that isn't `:00` or `:30`.
- **No catch-up.** A missed fire runs once when Claude becomes idle, not once per missed interval.
- **Tasks fire only while Claude Code is running and idle**, between turns. Backgrounding the session carries `/loop` tasks over.
- A session can hold **50** scheduled tasks. `CLAUDE_CODE_DISABLE_CRON=1` disables the scheduler entirely.

A scheduled fire **only runs skills Claude is allowed to invoke on its own.** Built-in commands, `disable-model-invocation: true` skills (including the bundled `/verify`), skills hidden by `skillOverrides` or a `Skill` deny rule, and MCP prompts all arrive as **plain text** instead of executing.

One-shot reminders need no command — `remind me at 3pm to push the release branch` schedules a single-fire task that deletes itself. Under the hood Claude uses `CronCreate`, `CronList`, and `CronDelete` with standard 5-field cron expressions, in **your local timezone**.

**Choosing a scheduler:**

| | Routines (cloud) | Desktop tasks | `/loop` |
|---|---|---|---|
| Runs on | Cloud | Your machine | Your machine |
| Machine must be on | No | Yes | Yes |
| Session must be open | No | No | **Yes** |
| Local file access | No (fresh clone) | Yes | Yes |
| Minimum interval | 1 hour | 1 minute | 1 minute |
| Survives restarts | Yes | Yes | Restored on `--resume` if unexpired |

**`/goal`** is the condition-driven form: Claude keeps working across turns until the condition is met or the goal clears. A separate evaluator re-checks after every turn. If Claude stalls, Claude Code eventually stops the run with the goal still set rather than looping forever. A goal survives resume, with its turn count, timer, and token-spend baseline reset ([goal](https://code.claude.com/docs/en/goal)).

```text
/goal all tests in src/auth pass and `npm run typecheck` is clean
/goal clear
```

**Channels** invert the polling: instead of `/loop` asking every five minutes whether CI finished, your CI **pushes the failure into the running session** ([channels](https://code.claude.com/docs/en/channels)). Enable the servers whose notifications a session listens for with `--channels`, and control them with `channelsEnabled` and `allowedChannelPlugins`. When you're waiting on an external system that can notify, channels are both cheaper and faster than a loop.

<a id="p14-messaging"></a>

### 14.6 Advanced: cross-session messaging

Claude can deliver a message from one of your sessions to another — a finding, a status, a decision ([cross-session messaging](https://code.claude.com/docs/en/cross-session-messaging)). Requires v2.1.224+ (macOS/Linux/WSL2) or v2.1.234+ (native Windows), and is on with nothing to enable.

```text
Let @api-worker know the schema migration finished
Ask the session running in my other terminal whether the migration finished
```

Claude uses `ListAgents` to discover targets and `SendMessage` to deliver; you never call either. `@` plus the first letters of a session name opens a typeahead (v2.1.232+). `/list-agents` (or `/peers`) shows what's reachable: subagents, agent-team teammates, your other local sessions, and — while connected to Remote Control — your cloud and other-machine sessions.

**A message is plain text — never the sender's conversation history or files.** To move a whole conversation, resume the session instead.

**How messages travel:**

| Target | Route |
|---|---|
| This machine | A per-session Unix socket (macOS/Linux) or named pipe (Windows). **Never through Anthropic servers** |
| Another of your machines | Through Anthropic servers, arriving over that machine's Remote Control connection |
| Claude Code on the web | Through Anthropic servers |

Same-machine delivery works by registering in files on disk, so **two sessions can only reach each other if they see the same files**: a container and its host can't, and a WSL 2 session and a native Windows session on the same computer can't.

**The safety model is the interesting part.** When session A messages session B, Claude Code tells B's Claude the message came from another session, not from you, and:

- **It can't approve anything.** A message never counts as your consent and can't answer a pending permission prompt.
- **It can't change configuration.** The receiving Claude is instructed never to change permission settings, `CLAUDE.md`, or other config because another session asked.
- **Commands don't run.** `/compact` in a message arrives as plain text.
- **Permission prompts still fire** for anything the message asks for.
- Claude is instructed **never to ask another session for an action denied or blocked in its own session**, and to route that work back to you.

**Inbound control** — `crossSessionInbound`: `accept` (deliver), `hold` (show a notice, deliver only if an `accept` later applies), `refuse` (drop). Also in `/config` → **Messages from your other sessions**. When no value applies, the default decides per message from the two sessions' permission modes: a session that **prompts** for permissions accepts messages, holding one only from a sender that bypasses prompts; a session that **bypasses** prompts holds every message for approval, delivering only from another bypassing sender.

**`isolatePeerMachines: true`** requires your approval before any message leaves the machine — **even in `bypassPermissions` mode**. A `true` from any settings scope applies, so a checked-in project file can turn it on but not off.

**Turning it off** — sending and receiving are separate:

```json
{
  "permissions": { "deny": ["SendMessage", "ListAgents"] },
  "crossSessionInbound": "refuse"
}
```

**Limits:** plain text only; ~1M characters per same-machine message; rapid bursts to one session are refused **at the sender**; loops are throttled (repeated messages rate-limited per sender, identical repeats within a short window dropped, at most 50 accepted messages queued) — so **a message loop between two sessions stops on its own**.

**A `-p` session binds an inbox socket** and can receive messages; a `--bare` session doesn't and doesn't appear in listings. To let a `-p` worker take messages unattended, set `crossSessionInbound: "accept"` in its `--settings`.

`CLAUDE_CODE_MESSAGING_SOCKET` and `CLAUDE_CODE_MESSAGING_TOKEN` are exported to hooks and Bash commands, so a script can post into its own session — the token auth line is optional on macOS/Linux and **required on native Windows**.

<a id="p14-plugins"></a>

### 14.7 Mastery: plugins as the distribution layer

A plugin bundles skills, agents, hooks, MCP servers, LSP servers, monitors, output styles, and workflows into one installable unit. It is the answer to *"a second repository needs the same setup."*

**Installing** ([discover plugins](https://code.claude.com/docs/en/discover-plugins)):

```shell
/plugin                                              # the manager: Discover, Installed, Marketplaces, Errors, Stats
/plugin install github@claude-plugins-official
/plugin marketplace add anthropics/claude-plugins-community
/plugin install <name>@claude-community
```

`claude-plugins-official` is added automatically on first interactive start. The community marketplace is added manually; its plugins pass automated validation and safety screening and are pinned to a commit SHA.

**Three installation scopes:** user (all your projects), project (everyone on the repo, written to `.claude/settings.json`), local (you, this repo). A **managed** scope exists too, installed by admins and unmodifiable.

The Discover detail pane shows a **context cost** estimate, a **last updated** date, and a **Will install** list of every command, agent, skill, hook, and MCP/LSP server — review it before installing.

**Code intelligence plugins** are the highest-value install for most people: they enable the built-in LSP tool, giving Claude **automatic diagnostics after every edit** and real symbol navigation instead of grep. `typescript-lsp`, `pyright-lsp`, `rust-analyzer-lsp`, `gopls-lsp`, and others — **you must install the language server binary yourself**; the plugin doesn't. `Executable not found in $PATH` in the `/plugin` **Errors** tab means you skipped that step.

**Building one:**

```text
my-plugin/
├── .claude-plugin/plugin.json   ← ONLY this goes in .claude-plugin/
├── skills/<name>/SKILL.md
├── commands/                    ← flat .md skills; use skills/ for new plugins
├── agents/<name>.md
├── hooks/hooks.json
├── .mcp.json
├── .lsp.json
├── monitors/monitors.json
├── output-styles/
├── workflows/
├── bin/                         ← added to the Bash tool's PATH while enabled
└── settings.json                ← only `agent` and `subagentStatusLine` supported
```

> **The single most common mistake:** putting `commands/`, `agents/`, `skills/`, or `hooks/` **inside** `.claude-plugin/`. Only `plugin.json` goes there; everything else sits at the plugin root.

```json
{
  "name": "my-first-plugin",
  "description": "A greeting plugin to learn the basics",
  "version": "1.0.0",
  "author": { "name": "Your Name" }
}
```

`name` is the **skill namespace** — skills become `/my-first-plugin:hello`. `version`, if set, gates updates: users only get them when you bump it.

**Developing:**

```bash
claude --plugin-dir ./my-plugin           # also accepts a .zip
claude --plugin-dir ./a --plugin-dir ./b  # multiple
claude --plugin-url https://example.com/my-plugin.zip
claude plugin init my-tool                # scaffold into ~/.claude/skills/, auto-loads as my-tool@skills-dir
claude plugin validate ./my-plugin        # add --strict to treat warnings as errors
```

`/reload-plugins` picks up changes without restarting — plugins, skills, agents, hooks, and plugin MCP/LSP servers. When a `--plugin-dir` plugin shares a name with an installed one, the local copy wins for that session (except plugins managed settings force-enable or force-disable).

**Team distribution** via `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "my-team-tools": { "source": { "source": "github", "repo": "your-org/claude-plugins" } }
  },
  "enabledPlugins": { "formatter@my-team-tools": true }
}
```

Once a teammate **trusts the folder**, the marketplace is added without a further prompt. As of v2.1.195, adding a marketplace **doesn't install** plugins from external sources — a project-enabled plugin from GitHub or npm stays uninstalled until the teammate runs the `claude plugin install` command Claude Code shows them.

**Hygiene.** The **Installed** tab groups plugins by scope, surfaces load errors first, and lists marketplace plugins you installed but haven't used in **at least two weeks over at least 10 sessions** under **Not used recently**, with a **Last used** line. Those plugins still cost startup time and context. (Plugins contributing only a theme, output style, monitor, or workflow are never listed as unused, since there's no invocation to track; an LSP plugin counts as used when its server delivers diagnostics.)

**Auto-updates** run in the background after startup with a random delay up to ten minutes, so the running session keeps its loaded versions. Official Anthropic marketplaces have it on by default; third-party and local ones don't. `DISABLE_AUTOUPDATER` turns it off; `FORCE_AUTOUPDATE_PLUGINS=1` alongside it keeps plugin updates while disabling Claude Code's own.

> ⚠️ **Plugins and marketplaces execute arbitrary code with your user privileges.** Anthropic doesn't control what's inside a third-party plugin and can't verify it works as intended. Install only from sources you trust. Organisations can restrict this with `strictKnownMarketplaces`, `blockedMarketplaces`, `disableCommandPluginSources`, and `strictPluginOnlyCustomization` (which blocks skills, agents, hooks, and MCP servers from user and project sources entirely, so plugins become the *only* customisation path).

> ### Real Scenario — the plugin that quietly doubled every session's startup
>
> A platform team packages their standard setup as a plugin: five skills, three subagents, a `PostToolUse` lint hook, and MCP servers for GitHub, Jira, and their internal deploy API. They enable it at project scope across twelve repositories. It works.
>
> Six weeks later, engineers report that sessions feel sluggish to start and that Claude has begun missing conventions it used to follow.
>
> **What `/context` and `/plugin` → **Stats** showed:**
>
> - Three of the five skills had never been invoked by anyone, but their descriptions loaded into every request.
> - Two subagents were duplicated: the plugin shipped `code-reviewer`, and eight of the twelve repos still had a `.claude/agents/code-reviewer.md` from before the migration. **Project agents override same-named plugin agents**, so half the fleet was running the old definition and nobody knew which.
> - The Jira MCP server had been misconfigured since week two and showed `✘ Failed to connect` in the **Errors** tab — which nobody had opened.
> - The skill listing had grown enough that descriptions were being truncated, which is why Claude stopped matching some of them.
>
> **The remediation:**
>
> 1. `/skill-doctor` and the **Stats** tab to find the unused skills; marked them `disable-model-invocation: true` so they cost nothing until invoked by name.
> 2. Deleted the stale `.claude/agents/code-reviewer.md` files — the docs are explicit that after migrating to a plugin you must **remove the originals**, because project and user agents override same-named plugin agents. (Plugin *skills* are namespaced, so those coexist instead.)
> 3. Fixed the Jira server and added a CI check asserting `mcp_server_errors` is empty in a `--bare -p` smoke run.
> 4. Raised `skillListingBudgetFraction` and shortened the remaining descriptions.
>
> **The lesson:** a plugin is a **standing context cost paid by every session in every repo that enables it**. Treat enabling one like adding a dependency — review the **Will install** list, check the context-cost estimate, and revisit the **Not used recently** group quarterly.

<a id="148-part-14-cheat-sheet"></a>

### 14.8 Part 14 cheat sheet

| Parallelism | Command |
|---|---|
| Worktree session | `claude --worktree <name>` / `-w` / `"#1234"` |
| Background session | `claude --bg "<prompt>"`, `/background` (`/bg`), `/fork` |
| Agent view | `claude agents` (`--json`, `--cwd`) |
| Manage background | `claude attach\|logs\|stop\|respawn\|rm <id>`, `claude daemon status\|stop` |
| Subagent fork | `/subtask <task>` |
| Workflows | `ultracode: <task>`, `/deep-research`, `/workflows`, `/effort ultracode` |
| Large change | `/batch <instruction>` — 5–30 worktree-isolated subagents, one PR each |
| Check on work | `claude agents` · `/tasks` · `/workflows` |

| Scheduling | |
|---|---|
| `/loop 5m <prompt>` | Fixed interval |
| `/loop <prompt>` | Claude self-paces, 1 min–1 hr |
| `/loop` | Maintenance prompt, or `loop.md` |
| `/goal <condition>` | Work until a condition is met |
| Channels (`--channels`) | External events pushed in — beats polling |
| Recurring task expiry | **7 days** |
| Max tasks per session | 50 |
| `CLAUDE_CODE_DISABLE_CRON=1` | Disable the scheduler |

| Worktree | |
|---|---|
| Default location | `.claude/worktrees/<name>/`, branch `worktree-<name>` |
| `worktree.baseRef` | `"fresh"` (default) vs. `"head"` |
| `.worktreeinclude` | Copy gitignored files in (gitignore syntax) |
| `worktree.bgIsolation: "none"` | Turn off background-session isolation |
| `${CLAUDE_PROJECT_DIR}` in hooks | **Stays at the project root** — read `cwd` for the worktree |
| `-p` runs | No exit cleanup — `git worktree remove` yourself |
| Git LFS `--local` | Pointer files; run `git lfs pull` in the worktree |

| Workflow limit | Value |
|---|---|
| Concurrent agents | 16 (fewer with fewer CPUs) |
| Items per `parallel()`/`pipeline()` | 4,096 |
| Agents per run | 1,000 |
| Large-run warning | >25 agents or >1.5M projected tokens |
| Size guideline | `small` <5 · `medium` <15 (default) · `large` <50 · `unrestricted` |
| `Date.now()`, `Math.random()` | **Throw** inside the script — pass via `args` |

| Plugin | |
|---|---|
| `/plugin` | Discover · Installed · Marketplaces · Errors · Stats |
| `/plugin install <name>@<marketplace>` | Install (scopes: user, project, local) |
| `/plugin marketplace add owner/repo` | Add a catalog |
| `/reload-plugins [--force]` | Apply changes without restarting |
| `claude plugin init\|validate\|install\|details` | Scripting and scaffolding |
| `--plugin-dir` / `--plugin-url` | Local / remote development |
| Layout rule | **Only `plugin.json` goes in `.claude-plugin/`** |
| After migrating to a plugin | **Delete the original `.claude/agents/` files** — project agents override plugin agents |

[↑ Back to top](#table-of-contents)

---

**Back to:** [Claude Code — Mastery Guide (Overview)](./claude-code-mastery-guide.md)
