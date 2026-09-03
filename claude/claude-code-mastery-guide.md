# Mastering Claude Code, the Agent SDK & the Claude API — A Reference Guide

## About This Document

- **Framing:** Goal-driven. Written for a developer learning this before moving into Agentic Engineering / AI-Driven development — depth, examples, and prioritization are tailored toward the concrete commands, APIs, flags, and config actually touched for that goal, not general topical coverage.
- **Depth tiers:** Every major Part below is structured as four progressive, explicitly labeled tiers — **Beginner → Working Knowledge → Advanced → Mastery** — so you can stop reading at whichever level you currently need. A tier is collapsed or merged into its neighbor where it would otherwise be trivial or redundant for that topic (this happens most in Parts IV–V, which get Beginner → Working Knowledge → Advanced, since Mastery-level SDK/API internals matter less for this reader's stated goal than daily-driver CLI fluency does).
- **Scope:** Everything — daily CLI usage, memory, permissions, subagents, skills, hooks, MCP, automation/headless mode, the Claude Agent SDK, and Claude API fundamentals (Messages API, tool use, streaming, caching, model selection). SDK/API code examples are given in **both TypeScript and Python**.
- **Assumed background:** None. Written from zero.
- **Sourcing standard:** Version-specific details (commands, flags, config syntax, model IDs) were checked against official docs (`code.claude.com/docs`, `platform.claude.com/docs`) as of September 2026, cited inline. Stack Overflow / community sources are used only for real-world gotchas, and are flagged as such — official docs win on any conflict. **Exact pricing is deliberately not hardcoded** anywhere in this document — it changes too often to be trustworthy in a reference doc; live links to the pricing page are given instead.
- **Update instructions for future-me:** If asked to update this later — match this exact structure (Parts, each with explicit tier subheadings), keep the wrong-vs-right snippet pattern for gotchas, keep TOC + back-to-top links, and add new Parts/tiers rather than reverting to freeform prose. If a section's tiers feel forced or redundant, collapsing tiers is expected and consistent with this doc's own stated approach — don't treat that as a deviation to fix.

---

## Table of Contents

1. [Orientation — The Agentic Loop & Where Everything Fits](#1-orientation--the-agentic-loop--where-everything-fits)
2. [Part I — Core CLI Daily Driver](#2-part-i--core-cli-daily-driver)
3. [Part II — Extensibility & Config](#3-part-ii--extensibility--config)
4. [Part III — Automation & Headless](#4-part-iii--automation--headless)
5. [Part IV — Claude Agent SDK](#5-part-iv--claude-agent-sdk)
6. [Part V — Claude API Fundamentals](#6-part-v--claude-api-fundamentals)
7. [Cheat Sheets](#7-cheat-sheets)
8. [Suggested Learning Order](#8-suggested-learning-order)
9. [Quick Self-Check](#9-quick-self-check)

---

## 1. Orientation — The Agentic Loop & Where Everything Fits

**What it is.** Claude Code is an agentic coding tool: instead of suggesting the next line (autocomplete-style), it reads your whole project, plans an approach, edits files, runs commands, observes results, and iterates — largely on its own. ([overview](https://code.claude.com/docs/en/overview))

**Why it matters.** This is the mental-model shift that everything else in this document hangs off. You're not reviewing suggestions token-by-token — you're delegating a task and reviewing an outcome. Every mechanism below (permissions, hooks, subagents, skills, MCP) is a way of shaping one step of the same underlying loop:

```text
 ┌─────────────────────────────────────────────────────────┐
 │  1. Understand the goal (prompt + CLAUDE.md + auto       │
 │     memory + conversation history)                       │
 │  2. Gather info (read files, search, grep)                │
 │  3. Decide next action, pick a tool                       │
 │  4. Permission check (deny → ask → allow → hooks)          │
 │  5. Execute (edit file / run command / call MCP tool)      │
 │  6. Observe result                                         │
 │  7. Verify (run tests, check output)                       │
 │  8. If insufficient → back to step 2                        │
 │  9. Stop when the goal is met                               │
 └─────────────────────────────────────────────────────────┘
```

**Where the layers sit.** Claude Code (the CLI), the Agent SDK, and the raw Claude API are the *same* underlying engine at increasing levels of exposure — this is worth internalizing before Part IV/V, because it means everything you learn in Parts I–III (permission modes, tool concepts, hooks) transfers directly into code:

```text
        Claude API (raw Messages API — you build everything yourself)
                          │
                          │  adds: agentic loop, built-in tools
                          │  (Read/Edit/Bash/...), context/compaction,
                          │  permission system
                          ▼
              Claude Agent SDK (embeddable library — Part IV)
                          │
                          │  adds: terminal UI, REPL, session files,
                          │  slash commands, CLAUDE.md auto-loading
                          ▼
              Claude Code (the CLI — Parts I–III)
```

**Real scenario.** A dev asks Claude Code to "add a `deletedAt` soft-delete column to the `users` table, update the model, and fix every query that assumes hard deletes." A Copilot-style autocomplete tool can't do this — it can't *find* every affected query and *act* across files. Claude Code greps the codebase, edits the migration, the model, and each affected query, runs the test suite, and reports what changed. Your job shifts from writing each fix to reviewing the diff and test output.

[⬆ back to top](#table-of-contents)

---

## 2. Part I — Core CLI Daily Driver

Covers: installation & auth, sessions, context management, permission modes, built-in tools, invoking subagents & skills.

### Beginner

**Install & first run.**

```bash
# macOS/Linux/WSL
curl -fsSL https://claude.ai/install.sh | bash
```

```powershell
# Windows PowerShell
irm https://claude.ai/install.ps1 | iex
```

```bash
# or via npm (Node.js 22+)
npm install -g @anthropic-ai/claude-code
```

Native installs (`install.sh`/`install.ps1`) auto-update in the background; Homebrew/WinGet installs do **not** — run `brew upgrade claude-code` or `winget upgrade Anthropic.ClaudeCode` periodically. ([setup](https://code.claude.com/docs/en/setup.md))

```bash
cd your-project
claude          # starts an interactive session; prompts for /login on first use
```

**Two fundamentally different modes:**

| Mode | Command | What happens |
| --- | --- | --- |
| Interactive (REPL) | `claude` | Opens a conversational session; you converse turn by turn |
| Print / headless | `claude -p "task"` | Runs once, prints the result, exits — no back-and-forth (foundation for Part III) |

```bash
# Wrong — expecting -p to behave interactively
claude -p "fix the failing tests"
# ...then trying to answer a follow-up — there's no session left to answer into.

# Right — use -p for one-shot/scripted work, plain `claude` to steer as it happens
claude "fix the failing tests"
```

**Auth precedence** (highest wins): cloud provider env var (Bedrock/Vertex/Foundry) → `ANTHROPIC_AUTH_TOKEN` → `ANTHROPIC_API_KEY` → `apiKeyHelper` script → `CLAUDE_CODE_OAUTH_TOKEN` → `/login` credential. A stray `ANTHROPIC_API_KEY` left set in your shell silently overrides a Claude Pro/Max subscription login — if `claude` seems to be billing your API key instead of your subscription, `unset ANTHROPIC_API_KEY` first. ([authentication](https://code.claude.com/docs/en/authentication.md))

### Working Knowledge

**Resuming sessions:**

```bash
claude -c                          # continue the most recent session, this directory
claude -r "session-name" "task"    # resume a specific named session
claude --resume                    # open an interactive picker
```

Sessions are stored as `.jsonl` files under `~/.claude/projects/<project>/`; the session ID is printed at session start.

**Context management — what auto-loads every session:** system prompt, auto memory (`MEMORY.md`, capped at ~200 lines / 25 KB), environment info (platform/shell/git branch), skill descriptions (one-liners only, not full bodies), then CLAUDE.md files in order: managed policy → `~/.claude/CLAUDE.md` (user) → ancestor dirs → project root → `.claude/CLAUDE.local.md` (personal, gitignored). ([memory](https://code.claude.com/docs/en/memory.md), [context-window](https://code.claude.com/docs/en/context-window.md))

```bash
/context           # shows current token usage, loaded files, tools
/compact [notes]    # summarizes conversation history, optionally focused on your criteria
/clear [name]        # discards conversation, starts fresh (old one stays in the session picker)
```

**Permission modes** — the mode governs how much runs without asking you:

| Mode | Behavior |
| --- | --- |
| `default` (aka "Manual") | Reads auto-approved; everything else prompts |
| `acceptEdits` | File edits auto-approved; other shell commands still confirm |
| `plan` | Read-only research + proposal; nothing executes until you approve the plan |
| `auto` | A classifier reviews actions in the background; safe ones proceed silently |
| `bypassPermissions` (`--dangerously-skip-permissions`) | Nothing prompts — sandboxed environments only |

Cycle modes live with `Shift+Tab`; set one for the session with `claude --permission-mode plan`. ([permission-modes](https://code.claude.com/docs/en/permission-modes.md))

```bash
# Plan before editing anything risky or unfamiliar
claude --permission-mode plan "propose a plan to migrate this table's soft-deletes, don't change anything yet"
```

**Built-in tools:**

| Tool | Does | Key gotcha |
| --- | --- | --- |
| Read | Returns file contents (text/image/PDF≤20pg/notebook) | No prompt inside working dir |
| Write | Creates/overwrites a **whole** file | Full replacement only — no partial edits |
| Edit | Targeted exact-string replacement | Must `Read` the file first in-session; no regex/fuzzy matching |
| Bash | Runs shell commands | Sandboxed by default on macOS/Linux/WSL2, **not** Windows native |
| Grep | ripgrep-based search | Respects `.gitignore` by default |
| Glob | Filename pattern search | Does **not** respect `.gitignore` (opposite of Grep) |
| WebFetch | Fetches a URL, extracts an answer via a small model | Lossy by design — use `curl` via Bash for raw HTML |
| WebSearch | Web search integrated into context | Domain safety checks apply |

**Invoking a subagent or skill directly:**

```bash
claude "use a subagent to find every call site of the deprecated claims API, summarize — don't change anything"
/deploy      # runs a custom skill or slash command by name
```

### Advanced

**Permission rule syntax and evaluation order** — this is the part people get wrong. A `deny` rule always wins, even against a broader `allow` rule elsewhere:

```text
   deny  ─────▶  ask  ─────▶  allow  ─────▶  (no match) defaultMode
  (blocks,      (prompts,     (proceeds
  first match    first match   silently)
  wins)          wins)
```

```json
{
  "permissions": {
    "allow": ["Bash(git status)", "Bash(git diff:*)", "Read(./**)"],
    "deny":  ["Bash(rm -rf:*)", "Read(./.env)", "Read(./.env.*)"],
    "ask":   ["Bash(git push:*)"]
  }
}
```

```bash
# Wrong — assuming this fully blocks .env from ever reaching Claude's context
"deny": ["Read(./.env)"]
# Only blocks the Read tool (and a few related built-ins) from that specific path.
# Another tool path could still surface its contents.

# Right — for anything truly sensitive, don't rely on a deny rule alone; keep
# secrets out of the working directory Claude has access to, or enforce with a
# hook (Part II) that hard-blocks regardless of tool path.
```

**`bypassPermissions` vs. `--dangerously-skip-permissions`:** the flag sets the mode and disables all prompts for that launch; the mode itself can also be cycled to via `Shift+Tab` mid-session. Either way, this should only run inside a fully isolated environment (container/VM/CI runner with no real credentials) — never on your local machine with real credentials present. Claude Code refuses to start in this mode with root/sudo on Linux/macOS specifically to make accidental misuse harder. ([permissions](https://code.claude.com/docs/en/permissions.md))

**Real scenario — the failure mode this guards against.** Multiple developers in 2026 reported losing significant work after granting an autonomous agent broad filesystem/infra access without oversight, including at least one production database wiped by an agent stuck in a correction loop. The permission system exists specifically to keep a human in the loop for anything destructive while still allowing everything else to run smoothly. ([community writeup](https://neurohive.io/en/guides/claude-code-getting-started-guide/))

### Mastery

- **CLAUDE.md loading is a hierarchy, not a merge of equals.** Project `.claude/settings.json` setting `defaultMode: "auto"` or `"bypassPermissions"` is **silently ignored** — those two modes only take effect from `~/.claude/settings.json` (user) or managed/enterprise settings, never from project-level config. This is a deliberate guard: a malicious or careless project-committed settings file can't unilaterally disable your prompts.
- **Auto memory internals.** Claude writes durable notes to `~/.claude/projects/<project>/memory/` as it works — without being asked — capturing build commands, debugging insights, and your corrections. `MEMORY.md` is the always-loaded index (hard-capped at ~200 lines / 25 KB — Claude Code errors and tells Claude to trim it if it grows past that); detailed notes overflow into topic files that load on demand. `/memory` browses/edits it directly.
- **Session resumption and permission-mode restoration have edge cases**: `claude --continue`/`--resume` restore the permission mode of the resumed session, with specific exceptions around `plan` and `auto` mode — a `claude -p --continue` run doesn't necessarily inherit the same restoration logic as an interactive resume. If a scripted resume behaves differently than the interactive one did, this is the first thing to check.
- **Glob's 100-file cap and mtime-sort** mean a broad glob on a large repo silently returns only the 100 most-recently-modified matches — a common source of "it didn't find X" bugs when X hasn't been touched recently. Narrow the pattern rather than assuming Glob is exhaustive.

[⬆ back to top](#table-of-contents)

---

## 3. Part II — Extensibility & Config

Covers: CLAUDE.md authoring, `settings.json`, hooks, MCP servers, authoring skills, authoring subagents.

### Beginner

**CLAUDE.md** is project instructions loaded into every session — conventions, architecture notes, gotchas Claude can't infer from the code itself.

```markdown
# Wrong — re-derives what the codebase already shows
## Directory Structure
- /app - main application
- /app/controllers - HTTP controllers
... (200 more lines `ls` would tell Claude in two seconds)

# Right — pitfalls and conventions Claude genuinely can't derive from the code
## Conventions
- Use AdonisJS validators (`app/validators`), never inline `request.input()` validation
- Run `node ace migration:run` before starting the dev server after pulling
```

**A minimal skill** is a folder with one file:

```text
.claude/skills/lint-js/SKILL.md
```

```markdown
---
name: lint-js
description: Run ESLint and report issues. Use when the user says "lint" or "check for lint errors".
---
Run ESLint on the project and explain any issues found.
```

**A minimal subagent** is a markdown file with YAML frontmatter in `.claude/agents/`:

```markdown
---
name: code-reviewer
description: Reviews a recent diff and reports issues by severity.
tools: Read, Grep, Glob, Bash
---
You are a code reviewer. Read the most recent diff, check for bugs, security
issues, and style problems, and return a prioritized list with file/line refs.
```

### Working Knowledge

**`settings.json` precedence** (highest to lowest): `.claude/settings.local.json` (machine-specific, gitignored) → `.claude/settings.json` (project, committed) → `~/.claude/settings.json` (user, all projects) → managed/enterprise settings. More restrictive always wins at any level. ([settings-reference](https://code.claude.com/docs/en/settings-reference.md))

```json
{
  "permissions": {
    "defaultMode": "acceptEdits",
    "allow": ["Bash(npm run lint:*)"],
    "deny": ["Bash(rm -rf:*)"]
  },
  "env": { "NODE_ENV": "development" }
}
```

**Hooks** are shell commands the harness runs automatically at lifecycle points — the enforcement layer CLAUDE.md can't be, since CLAUDE.md is a request the model tries to follow and a hook is a guarantee the harness enforces regardless of what the model decides.

```text
                 Tool call about to happen
                          │
                          ▼
                 ┌─────────────────┐
                 │   PreToolUse    │──── exit code 2 ──▶ BLOCKED
                 │      hook       │      (stderr shown to the model as the reason)
                 └────────┬────────┘
                          │ allowed
                          ▼
                    permission check (deny → ask → allow)
                          │
                          ▼
                     tool executes
                          │
                          ▼
                 ┌─────────────────┐
                 │  PostToolUse    │──── e.g. auto-format after every Edit/Write
                 │      hook       │
                 └─────────────────┘
```

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{ "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write" }]
    }]
  }
}
```

**Adding an MCP server** (extends Claude's tools beyond the built-ins — issue trackers, databases, internal tooling):

```bash
claude mcp add --transport http github --scope project https://api.githubcopilot.com/mcp/
claude mcp add --transport stdio airtable --env AIRTABLE_API_KEY=YOUR_KEY -- npx -y airtable-mcp-server
```

Scopes: `local` (default, private to you) → `project` (`.mcp.json`, shared via git) → `user` (all your projects). Manage connected servers with `/mcp`.

### Advanced

**Hook types beyond `command`:** `type: "prompt"` (LLM-evaluated), `type: "agent"`, and `type: "http"` also exist, alongside the standard shell `command` hooks — useful when the gate condition needs judgment rather than a fixed shell check. Timeouts differ by type (`command`/`http`/`mcp_tool`: 10 min default; `prompt`: 30s; `agent`: 60s). ([hooks-guide](https://code.claude.com/docs/en/hooks-guide.md))

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "FILE=$(cat | jq -r '.tool_input.file_path // empty'); if echo \"$FILE\" | grep -qE '(\\.env|\\.lock|id_rsa|\\.pem)'; then echo \"BLOCKED: protected file\" >&2; exit 2; fi"
      }]
    }]
  }
}
```

```bash
# Wrong — relying on a CLAUDE.md line: "never edit .env files"
# This is a request. A determined or confused agent can still edit it.

# Right — enforce with a PreToolUse hook that exits 2 (above).
# The instruction becomes structurally impossible to violate, not just discouraged.
```

Important nuance: a hook can only make things *stricter*, never looser — if a hook returns "allow" but a `deny`/`ask` rule in settings still matches, the settings rule wins. A hook cannot grant more access than the permission system already allows.

**MCP OAuth flow:** add the server without credentials, then either run `/mcp` inside a session and follow the browser login, or `claude mcp login <server>` from the CLI (v2.1.186+). Use `--` to separate Claude's own flags from the server command when adding a stdio server — omitting it causes the server's args to be misparsed as Claude Code options.

**Custom subagent frontmatter fields in full:** `name`, `description` (drives auto-delegation matching — be concrete: "fixes failing tests" beats "helps with testing"), `tools` (allowlist; defaults to all if omitted), `model` (`inherit`/`sonnet`/`opus`/`haiku`/`fable`), `permissionMode`, `memory` (`user`/`project`/`local` — enables persistent auto memory scoped to that subagent). `/agents` walks through creating one interactively.

**Skill triggering mechanics:** Claude matches your request against each skill's `description` at session start (only the one-liner loads then — the full body loads on demand). A vague description ("manages my project") won't match a concrete request ("set up the project here"); write descriptions using the literal phrases a user would actually type. Set `disable-model-invocation: true` to make a skill manual-only, invoked strictly via `/name`.

### Mastery

- **`.claude/commands/` and `.claude/skills/` are functionally unified** (since v2.1.199) — both create a `/name` command; skills add support for supporting files (`references/` for background docs Claude reads but doesn't copy, `scripts/` for executables) and optional auto-invocation, so prefer skills for anything beyond a one-file procedure going forward.
- **Skills are a trust boundary, not just a convenience.** A skill gives Claude new instructions and can direct tool use in ways that don't match its stated purpose — treat a downloaded community skill the way you'd treat an unreviewed dependency; only run skills you wrote or that come from Anthropic directly.
- **Subagent context isolation is real, not just described in the docs.** A subagent receives its own system prompt, the delegation task message, CLAUDE.md files, a git status snapshot, and any preloaded skills — but *not* the parent session's conversation history or auto memory (unless `memory: project`/`user` is explicitly set on that subagent). This is why a subagent can't "remember" something you told the main session two turns ago; it was never in its context to begin with.
- **Plugins bundle skills + subagents + hooks + MCP servers + commands into one versioned, installable unit** (`/plugin marketplace add <repo>`, `/plugin install <name>@<marketplace>`) — the mechanism for standardizing a whole team's setup with one command instead of everyone hand-copying `.claude/` folders. Each installed plugin adds to context load the same way a connected MCP server does, so install for an actual friction point, not speculatively.

[⬆ back to top](#table-of-contents)

---

## 4. Part III — Automation & Headless

Covers: `-p`/`--print` scripting, structured output, exit codes, scheduled/background execution, CI integration.

### Beginner

```bash
claude -p "explain what this error means" < build-error.txt
claude -p --output-format json "list all REST endpoints in the API"
```

`-p` runs once with no interactive prompts and exits — the foundation for every automation pattern below. Piped stdin is capped at 10 MB; for larger inputs, write to a file and reference the path instead.

### Working Knowledge

**Output formats:**

| Format | Use |
| --- | --- |
| `text` (default) | Plain text response |
| `json` | Structured envelope with `result`, `session_id`, `total_cost_usd` |
| `stream-json` | Newline-delimited streaming events, for real-time consumption |

```bash
session_id=$(claude -p "start review" --output-format json | jq -r '.session_id')
claude -p "continue that review" --resume "$session_id"
```

**Being explicit about permissions in a script** — `-p` does not prompt, so an unconfigured permission mode either hangs or silently proceeds under whatever `defaultMode` happens to apply:

```bash
# Wrong — assuming -p prompts for permission like an interactive session would
claude -p "deploy the app"
# In CI, with no one watching, this either hangs waiting on a prompt that never
# comes, or silently proceeds under the ambient defaultMode.

# Right — be explicit about what's safe to automate
claude -p --permission-mode acceptEdits "run tests and fix any that fail"
# Reserve bypassPermissions strictly for fully sandboxed CI runners.
```

**Exit codes:** `0` success, `1` failure, `2` partial (e.g. cost ceiling hit), `130` interrupted (SIGINT), `143` terminated (SIGTERM) — check these in a CI step rather than assuming a non-zero exit always means "the model failed."

### Advanced

**`--bare` mode** skips loading hooks, skills, custom commands, subagents, plugins, MCP servers, auto memory, and CLAUDE.md — for reproducible CI runs across machines that shouldn't depend on a developer's local `.claude/` config. Requires `ANTHROPIC_API_KEY` (not subscription login); context can still be loaded explicitly via `--settings`, `--mcp-config`, `--agents`.

**Scheduling recurring work** — three distinct mechanisms with different tradeoffs:

| Mechanism | Runs on | Needs session open | Survives restart |
| --- | --- | --- | --- |
| `/schedule` (cloud Routine) | Anthropic's cloud | No | Yes |
| Desktop scheduled task | Your machine | No | Yes |
| `/loop` | Your machine | Yes | No (session-scoped, ~7-day expiry) |

```bash
/loop 5m check if deployment finished     # fixed interval
/loop check CI and review comments         # Claude chooses the interval dynamically
```

**Background Bash gotcha:** background dev-server/watch-build processes started via Bash are terminated ~5 seconds after Claude's final result in a `-p` run — a script that expects a spawned server to keep running after the CLI exits will find it already dead.

**CI integration:**

```yaml
- name: Claude review
  run: claude -p "review this PR diff" --output-format json --allowedTools "Read,Grep"
```

[⬆ back to top](#table-of-contents)

---

## 5. Part IV — Claude Agent SDK

### Beginner

**What it is.** The Agent SDK is Claude Code's own harness — the agentic loop, built-in tools, context management, and permission system — packaged as a library you embed in your own application, rather than run as a terminal CLI. You host and deploy it yourself; Anthropic does not. Packages: `@anthropic-ai/claude-agent-sdk` (npm), `claude-agent-sdk` (pip). ([Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview))

**Hello world:**

**Python:**

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="Review utils.py for bugs and fix them.",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Glob"],
            permission_mode="acceptEdits",
        ),
    ):
        print(message)

asyncio.run(main())
```

**TypeScript:**

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Review utils.py for bugs and fix them.",
  options: {
    allowedTools: ["Read", "Edit", "Glob"],
    permissionMode: "acceptEdits",
  },
})) {
  console.log(message);
}
```

```bash
# Wrong — expecting the SDK to include the Claude Code binary on every platform silently
pip install claude-agent-sdk
# On some platforms (e.g. ARM64 Windows) this pulls a source distribution with no
# bundled binary — the agent fails at runtime with no clear reason why.

# Right — install Claude Code natively as a fallback the SDK can find on PATH
npm install -g @anthropic-ai/claude-code
# TypeScript: pass pathToClaudeCodeExecutable in ClaudeAgentOptions if autodetection fails
```

### Working Knowledge

**Custom tools** become an in-process MCP server your agent can call — the same MCP concept from Part II, just without a separate process:

**Python:**

```python
from claude_agent_sdk import tool, create_sdk_mcp_server, ClaudeAgentOptions

@tool("add", "Add two numbers", {"a": float, "b": float})
async def add(args):
    return {"content": [{"type": "text", "text": f"Sum: {args['a'] + args['b']}"}]}

calculator = create_sdk_mcp_server(name="calculator", tools=[add])
options = ClaudeAgentOptions(
    mcp_servers={"calc": calculator},
    allowed_tools=["mcp__calc__add"],  # pattern: mcp__<server>__<tool>
)
```

**TypeScript:**

```typescript
import { tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

const addTool = tool(
  "add", "Add two numbers", { a: z.number(), b: z.number() },
  async ({ a, b }) => ({ content: [{ type: "text", text: `Sum: ${a + b}` }] })
);
const calculator = createSdkMcpServer({ name: "calculator", tools: [addTool] });
// options.mcpServers = { calc: calculator }, allowedTools: ["mcp__calc__add"]
```

A silently-denied tool call almost always means the name in `allowed_tools` doesn't exactly match `mcp__<server>__<tool>` — check that before assuming the tool implementation is broken.

**Permission callbacks** are the code-level equivalent of a `PreToolUse` hook — same decision point in the loop, expressed as a function instead of a shell command:

```python
async def custom_permission(tool_name, input_data, context):
    if tool_name == "Write" and "/system/" in input_data.get("file_path", ""):
        return PermissionResultDeny(message="System writes blocked")
    return PermissionResultAllow()

options = ClaudeAgentOptions(can_use_tool=custom_permission)
```

**Real scenario.** You're building an internal Slack bot that triages bug reports: reads the relevant repo, reproduces the issue, opens a draft PR — but must never touch `/infra/` and must never run `terraform apply`. You'd set `allowed_tools` to a narrow set (Read, Grep, Edit, Bash for test commands only) and add a `can_use_tool` callback that hard-denies any path under `/infra/` and any Bash command containing `terraform` — the same layered defense as Part I/II's permissions + hooks, expressed as SDK config instead of `.claude/settings.json`.

### Advanced

**SDK vs. Tool Runner vs. Managed Agents — three distinct products, easy to conflate:**

| Product | Built-in tools? | Who hosts it? | Use when |
| --- | --- | --- | --- |
| **Agent SDK** | Yes — Read/Edit/Bash/etc. bundled | You | You want Claude-Code-equivalent autonomy (file edits, shell commands) inside your own app |
| **Tool Runner** (`client.beta.messages.tool_runner`, in the API SDKs) | No — only tools *you* define | You | You want a managed agentic loop (retries, approval hooks) with full control over what tools exist — no filesystem/shell access implied |
| **Managed Agents** | Defined per-agent | Anthropic (hosted sandbox) | You don't want to run any infrastructure yourself |

If you only need "call my own functions in a loop," Tool Runner is lighter weight than pulling in the full Agent SDK. If you need filesystem/shell autonomy like Claude Code itself has, the Agent SDK is the right layer. ([Tool Runner](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner))

[⬆ back to top](#table-of-contents)

---

## 6. Part V — Claude API Fundamentals

### Beginner

**What it is.** The Messages API is the raw substrate everything above is built on — Claude Code, the Agent SDK, and Tool Runner all ultimately send Messages API requests. Understanding it directly matters once you're debugging *why* an agent behaved a certain way, tuning cost, or building something the higher-level tools don't cover. ([Messages API](https://platform.claude.com/docs/en/api/messages))

**Python:**

```python
import anthropic

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from env
message = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    system="You are a senior Python developer. Follow PEP 8.",
    messages=[{"role": "user", "content": "Write a hello world function"}],
)
print(message.content[0].text)
```

**TypeScript:**

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();  // reads ANTHROPIC_API_KEY from env
const message = await client.messages.create({
  model: "claude-sonnet-5",
  max_tokens: 1024,
  system: "You are a senior Python developer. Follow PEP 8.",
  messages: [{ role: "user", content: "Write a hello world function" }],
});
console.log(message.content[0].text);
```

```bash
# Wrong — putting the system prompt inside the messages array
messages=[{"role": "system", "content": "..."}, {"role": "user", "content": "..."}]
# The Messages API has no "system" role inside messages — this errors or is
# ignored depending on SDK version.

# Right — system is always its own top-level parameter, never a message role
system="..."
messages=[{"role": "user", "content": "..."}]
```

`max_tokens` is required and is a hard ceiling on the *response* — it does not affect input. Current model IDs: `claude-opus-5`, `claude-sonnet-5`, `claude-haiku-4-5-20251001`, `claude-fable-5` — always confirm against the live [models overview](https://platform.claude.com/docs/en/about-claude/models/overview) before hardcoding one, since IDs and availability change.

### Working Knowledge

**Tool use (function calling)** is the mechanism the Agent SDK's built-in tools and every MCP tool call are built on underneath:

```python
tools = [{
    "name": "get_weather",
    "description": "Get current weather for a location",
    "input_schema": {
        "type": "object",
        "properties": {"location": {"type": "string", "description": "City, State"}},
        "required": ["location"],
    },
}]

response = client.messages.create(
    model="claude-sonnet-5", max_tokens=1024, tools=tools,
    messages=[{"role": "user", "content": "What's the weather in SF?"}],
)

if response.stop_reason == "tool_use":
    for block in response.content:
        if block.type == "tool_use":
            result = run_my_weather_lookup(block.input["location"])
            # Round-trip: echo the assistant turn back, then the result,
            # tagged with the SAME tool_use_id Claude generated
            messages = [
                {"role": "user", "content": "What's the weather in SF?"},
                {"role": "assistant", "content": response.content},
                {"role": "user", "content": [
                    {"type": "tool_result", "tool_use_id": block.id, "content": result}
                ]},
            ]
```

```bash
# Wrong — generating your own id for the tool_result, or reusing a stale one
{"type": "tool_result", "tool_use_id": "my-own-id-123", "content": result}
# Claude can't match this back to the tool_use block it issued — the turn breaks.

# Right — always pass back the exact id from the tool_use block you're responding to
{"type": "tool_result", "tool_use_id": block.id, "content": result}
```

`tool_choice` controls whether Claude *must* call a tool: `{"type": "auto"}` (default), `{"type": "any"}` (must call some tool), `{"type": "tool", "name": "..."}` (force a specific one), `{"type": "none"}`. Claude can request multiple tools in one turn — check for more than one `tool_use` block in `response.content` rather than assuming exactly one. ([Tool use guide](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview))

**Streaming**, for anything user-facing:

```python
with client.messages.stream(
    model="claude-sonnet-5", max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

```typescript
await client.messages.stream({
  model: "claude-sonnet-5", max_tokens: 1024,
  messages: [{ role: "user", content: "Hello" }],
}).on("text", (text) => process.stdout.write(text));
```

Final token usage arrives only in the closing `message_delta` event — you can't know the exact cost of a streamed request until it finishes. ([Streaming](https://platform.claude.com/docs/en/build-with-claude/streaming))

### Advanced

**Prompt caching** is the API-level mechanism behind why a long, stable CLAUDE.md or system prompt doesn't cost full price on every turn — mark a stable content block with `cache_control`, and repeated requests that reuse it pay a much smaller "cache read" rate instead of full input price:

```python
response = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    system=[{
        "type": "text",
        "text": "<your long, stable system prompt / doc corpus>",
        "cache_control": {"type": "ephemeral"},  # default TTL; a longer TTL is also available
    }],
    messages=[{"role": "user", "content": "..."}],
)
print(response.usage.cache_creation_input_tokens, response.usage.cache_read_input_tokens)
```

There's a minimum token count below which a block won't actually get cached (it varies by model — check the [prompt caching docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) for the current thresholds rather than assuming a number here). Put cache breakpoints on genuinely stable content — never on something that changes every request (a timestamp, per-user data), or you pay the cache-write premium every call and never see a hit.

**Token counting** (`client.messages.count_tokens(...)`) is free to call and estimates a request's input cost *before* sending it — worth using in a loop that processes many documents, to catch an oversized prompt before you pay for it.

**Model selection** — rough rule of thumb for agentic work: start with a mid-tier model (Sonnet) for everyday agentic coding, iterate down to a cheaper/faster one (Haiku) once prompts and evals are solid, or up to a higher-reasoning one (Opus/Fable) for genuinely hard, long-horizon tasks. Always check the live [pricing page](https://platform.claude.com/docs/en/about-claude/pricing) and [choosing a model guide](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model) before making a cost-sensitive decision — prices and per-model discounts change.

**Real scenario.** An agent you built with the Agent SDK is processing hundreds of support tickets a day, each time re-sending your entire 8,000-token style guide as part of the system prompt. Without caching, that's 8,000 tokens of full-price input on every ticket. Marking the style guide block with `cache_control` means only the first ticket pays full price — every subsequent ticket within the cache TTL reads it at the much cheaper cache-read rate, usually the single biggest lever for cutting cost on a high-volume agent like this.

[⬆ back to top](#table-of-contents)

---

## 7. Cheat Sheets

### CLI essentials

| Command | Purpose |
| --- | --- |
| `claude` | Start interactive session |
| `claude -p "task"` | Headless: run once, print, exit |
| `claude -c` | Continue most recent session, this directory |
| `claude -r "name" "task"` | Resume a named session |
| `claude --output-format json` | Structured output (print mode only) |
| `claude --permission-mode acceptEdits` | Files auto-approved; shell still confirms |
| `claude --dangerously-skip-permissions` | Skip all checks (sandboxed environments only) |
| `claude mcp add ...` | Register an MCP server |
| `claude update` | Update the CLI |

### Permission rule syntax

| Pattern | Matches |
| --- | --- |
| `Bash` / `Bash(*)` | All Bash commands |
| `Bash(npm run:*)` | Any `npm run ...` variant |
| `Read(./.env)` | That exact file, for Read (and a few related built-ins) |
| `Read(./**)` | Everything under the project root |

Evaluation order: **deny → ask → allow → defaultMode fallback.**

### Hook events (most common)

| Event | Fires | Can block? |
| --- | --- | --- |
| `PreToolUse` | Before any tool executes | Yes (`exit 2`) |
| `PostToolUse` | After a tool succeeds | No (side effects only) |
| `Stop` | Claude is about to finish responding | Yes |
| `SubagentStop` | A subagent finishes | Yes |
| `SessionStart` | New session begins | — |

### Memory scopes (broadest → narrowest)

Enterprise policy → Project (`./CLAUDE.md`) → User (`~/.claude/CLAUDE.md`) → Project-local (`./CLAUDE.local.md`)

### SDK / API layer decision

| Need | Reach for |
| --- | --- |
| Interactive daily-driver terminal use | Claude Code (Parts I–III) |
| Embed Claude-Code-equivalent autonomy in your own app | Agent SDK (Part IV) |
| Managed loop over tools *you* define, no filesystem autonomy | Tool Runner |
| Zero infra to run yourself | Managed Agents |
| Full manual control (custom loop, custom caching strategy) | Raw Messages API (Part V) |

[⬆ back to top](#table-of-contents)

---

## 8. Suggested Learning Order

1. **Install and run one real task** (§2 Beginner/Working Knowledge) — a small, low-risk bug fix in a project you know well. Get comfortable reviewing a Claude-generated diff.
2. **Write a minimal CLAUDE.md** (§3 Beginner) — just the pitfalls and conventions your project needs, under 50 lines to start.
3. **Configure permissions deliberately** (§2 Advanced) — set up `allow`/`deny` for your common safe commands so you stop rubber-stamping `git status`.
4. **Try one hook** (§3 Working Knowledge/Advanced) — start with a `PostToolUse` formatter hook; low-risk, immediate feedback.
5. **Delegate one research task to a subagent** (§2 Working Knowledge, §3 Mastery) — notice how it keeps your main context clean.
6. **Write one custom skill** (§3 Beginner/Advanced) — turn a checklist you paste repeatedly into a `/`-command.
7. **Connect one MCP server** (§3 Working Knowledge) — GitHub is a good first choice if your repos live there.
8. **Automate one recurring task** (§4) — a `-p` script or a scheduled routine for something you do weekly.
9. **Read the Claude API fundamentals** (§6) — tool use and prompt caching in particular, since they're what everything above is quietly doing on your behalf.
10. **Build one throwaway agent on the Agent SDK** (§5) — a minimal script with 2–3 allowed tools, on a low-stakes local task. The goal is feeling the same permission-mode/tool-allowlist concepts from code, not shipping something real yet.

[⬆ back to top](#table-of-contents)

---

## 9. Quick Self-Check

- What's the difference between what CLAUDE.md does and what auto memory does — and who writes each?
- Why can't a CLAUDE.md instruction alone guarantee Claude never touches a file — what mechanism actually guarantees that?
- In what order are permission rules evaluated, and which one wins on a conflict?
- Why does project-level `.claude/settings.json` setting `defaultMode: "bypassPermissions"` get silently ignored?
- When would you reach for a subagent instead of just asking directly in the main session, and what does it *not* inherit from the parent session?
- What's the actual trigger mechanism for a skill, and what's the most common reason one fails to fire?
- Why does connecting an MCP server have a cost even in a turn where you don't use it?
- What's the practical difference between `claude` and `claude -p` for a CI script, and what's the exit-code gotcha to check for?
- What's the actual difference between the Claude Agent SDK, Tool Runner, and Managed Agents — and which one has built-in filesystem/shell tools?
- In a `tool_use` / `tool_result` round trip, what field has to match exactly, and what breaks if it doesn't?
- Why would marking a `cache_control` breakpoint on a timestamp-containing block actively cost you money instead of saving it?

[⬆ back to top](#table-of-contents)
