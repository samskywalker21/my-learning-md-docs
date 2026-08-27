# Mastering Claude Code — A Reference Guide

## About This Document

- **Framing:** Mixed goal-driven + curiosity-driven. Practical enough to start using Claude Code well immediately, but sections also cover *why* things are built the way they are (e.g. why hooks exist as a separate layer from CLAUDE.md) where that context helps the mental model stick.
- **Scope:** Comprehensive coverage of Claude Code — not just daily CLI usage, but memory, permissions, subagents, skills, hooks, MCP, plugins, automation, and the newer non-terminal surfaces (Desktop, Web, Routines).
- **Assumed background:** None. Written from zero — no prior CLAUDE.md/Copilot-instructions knowledge assumed, even though some was mentioned in passing before.
- **Reader context:** Sam, a developer working primarily in an AdonisJS + Nuxt 3/Vue stack, on Windows with Laragon/Nginx. Examples lean toward that stack where a concrete example helps, but the concepts apply to any codebase.
- **Sourcing standard:** Version-specific details (commands, flags, config syntax) are checked against the official docs at `code.claude.com/docs` as of August 2026 and linked inline. Where a community blog's convention conflicts with the official docs, the official docs win and the conflict is flagged.
- **Update instructions for future-me:** If you ask me to update this later, match this structure, this citation style (inline links, official docs preferred), and keep the wrong-vs-right snippet pattern for gotchas. Add new sections rather than restructuring unless the topic itself has changed shape.

---

## Table of Contents

1. [What Claude Code Actually Is](#1-what-claude-code-actually-is)
2. [Getting Started](#2-getting-started)
3. [Memory: CLAUDE.md & Auto Memory](#3-memory-claudemd--auto-memory)
4. [Everyday Workflows](#4-everyday-workflows)
5. [Permissions & Safety](#5-permissions--safety)
6. [Subagents](#6-subagents)
7. [Skills & Slash Commands](#7-skills--slash-commands)
8. [Hooks](#8-hooks)
9. [MCP (Model Context Protocol)](#9-mcp-model-context-protocol)
10. [Plugins & Marketplaces](#10-plugins--marketplaces)
11. [Automation & CI](#11-automation--ci)
12. [Beyond the Terminal](#12-beyond-the-terminal)
13. [Real-World Patterns & Common Mistakes](#13-real-world-patterns--common-mistakes)
14. [Cheat Sheets](#14-cheat-sheets)
15. [Suggested Learning Order](#15-suggested-learning-order)
16. [Quick Self-Check](#16-quick-self-check)

---

## 1. What Claude Code Actually Is

**What it is.** Claude Code is an agentic coding tool: instead of suggesting the next line of code (autocomplete-style, like GitHub Copilot), it reads your whole project, plans an approach, edits files across your codebase, runs commands, observes the results, and iterates — largely on its own. ([overview](https://code.claude.com/docs/en/overview))

**Why it matters.** This is the single biggest mental-model shift if you're coming from Copilot or plain chat-based AI help. You're not reviewing suggestions token-by-token — you're delegating a task and reviewing an outcome. That changes what you need to get good at: not "write a better prompt for this one line," but "scope a task well, set guardrails, and review a diff."

**The agentic loop.** Every action Claude Code takes goes through the same loop:

```
 ┌─────────────────────────────────────────────────────────┐
 │  1. Understand the goal (your prompt + CLAUDE.md +      │
 │     auto memory + conversation history)                 │
 │  2. Gather info (read files, search, grep)               │
 │  3. Decide next action, pick a tool                       │
 │  4. Permission check (rules → hooks → prompt if needed)   │
 │  5. Execute (edit file / run command / call MCP tool)     │
 │  6. Observe result                                        │
 │  7. Verify (run tests, check output)                      │
 │  8. If insufficient → back to step 2                       │
 │  9. Stop when the goal is met                              │
 └─────────────────────────────────────────────────────────┘
```

Nearly everything in this document is a way of shaping one part of that loop: CLAUDE.md and auto memory shape step 1, permissions and hooks gate step 4, subagents and skills change *who* runs steps 2–7, and MCP adds new tools to step 3.

**Surfaces.** The same engine runs everywhere: Terminal CLI (full-featured, scriptable), VS Code / JetBrains extensions (inline diffs in your editor), a standalone Desktop app, and a web version at claude.ai/code. A repo's CLAUDE.md, settings, and MCP servers work identically across all of them. ([overview](https://code.claude.com/docs/en/overview))

**Real scenario.** A dev on a mid-sized Vue/Nuxt team asks Claude Code to "add a `deletedAt` soft-delete column to the `users` table, update the model, and fix every query that assumes hard deletes." A Copilot-style tool can't do this — it has no way to *find* every affected query and *act* across files. Claude Code greps the codebase, finds the query sites, edits the migration, the model, and each affected query, runs the test suite, and reports what it changed. Your job shifts from writing each fix to reviewing the diff and the test output.

[⬆ back to top](#table-of-contents)

---

## 2. Getting Started

**Requirements:** Node.js 18+, and a Claude.ai (recommended) or Anthropic Console account. ([overview](https://code.claude.com/docs/en/overview))

**Install (macOS/Linux/WSL):**
```bash
curl -fsSL https://claude.ai/install.sh | bash
```

**Install (Windows PowerShell)** — this is your default environment on Laragon/Windows:
```powershell
irm https://claude.ai/install.ps1 | iex
```
> Native installs auto-update in the background. If you use Homebrew or WinGet instead, those channels do **not** auto-update — you'll need `brew upgrade claude-code` or `winget upgrade Anthropic.ClaudeCode` periodically. ([overview](https://code.claude.com/docs/en/overview))

**First run:**
```bash
cd your-project
claude
```
You'll be prompted to log in on first use.

**Two fundamentally different modes:**

| Mode | Command | What happens |
|---|---|---|
| Interactive | `claude` | Opens a REPL session; you converse turn by turn |
| Print / headless | `claude -p "task"` | Runs one query, prints the result, exits — no interactive prompts |

```bash
# Wrong: expecting -p to behave interactively
claude -p "fix the failing tests"
# ...then trying to answer a follow-up question — there's no session left to answer into.

# Right: use -p for scripts/CI where you don't want a back-and-forth,
# use plain `claude` when you want to steer the work as it happens.
claude "fix the failing tests"
```

Print mode (`-p`) is the foundation for every automation use case later in this doc (§11). ([cli-reference](https://code.claude.com/docs/en/cli-reference))

[⬆ back to top](#table-of-contents)

---

## 3. Memory: CLAUDE.md & Auto Memory

**What it is.** Every session starts with a *fresh* context window — Claude doesn't remember yesterday's conversation by default. Two mechanisms carry knowledge across sessions instead:

- **CLAUDE.md** — instructions *you* write
- **Auto memory** — notes *Claude* writes itself, based on your corrections and preferences, without you asking

Both load automatically at the start of every conversation and are treated as context Claude tries to follow — not hard-enforced rules. ([memory](https://code.claude.com/docs/en/memory))

**Why the distinction matters:** if something must *never* happen (not just "please don't"), CLAUDE.md is the wrong tool — that's what hooks are for (§8). CLAUDE.md is a request, not a guard rail.

**The four CLAUDE.md scopes**, broadest to narrowest:

| Scope | Location | Purpose | Shared with |
|---|---|---|---|
| Enterprise policy | e.g. `C:\ProgramData\ClaudeCode\CLAUDE.md` (Windows) | Org-wide rules | Everyone in the org |
| Project memory | `./CLAUDE.md` at repo root | Team-shared conventions | Team, via git |
| User memory | `~/.claude/CLAUDE.md` | Your personal preferences, all projects | Just you |
| Project memory (local) | `./CLAUDE.local.md` | Personal, project-specific overrides | Just you, this project |

Higher-in-hierarchy files load first, forming a base the more specific ones build on. ([memory](https://code.claude.com/docs/en/memory))

> **Flag — conflicting info in the wild:** some 2026 community write-ups describe `CLAUDE.local.md` as deprecated. The official docs still treat it as a first-class scope. Follow the official docs on this.

**Imports** — pull other files into CLAUDE.md with `@path`:
```markdown
See @README for project overview and @package.json for available npm commands.

## Additional Instructions
- git workflow: @docs/git-instructions.md
```
Both relative and absolute paths work; importing a file from your home directory is a convenient way to share personal instructions that aren't checked into the repo. ([memory](https://code.claude.com/docs/en/memory))

**Keep it short.** A long CLAUDE.md loads in full, every session, before any real work starts — eating your context budget. A widely-repeated rule from the Claude Code team: keep it around 100 lines; for large domain docs, reference a separate file instead of inlining everything. ([neurohive guide](https://neurohive.io/en/guides/claude-code-getting-started-guide/))

```markdown
# Wrong — a CLAUDE.md that re-derives what the codebase already shows
## Directory Structure
- /app - main application
- /app/controllers - HTTP controllers
- /app/models - Lucid models
- /config - Adonis config files
... (200 more lines Claude could get from `ls` in two seconds)

# Right — pitfalls and conventions Claude can't derive from the code alone
## Conventions
- Use AdonisJS validators (`app/validators`), never inline `request.input()` validation
- Nuxt pages use `<script setup>` only — no Options API, even in old files
- Run `node ace migration:run` before starting the dev server after pulling
```
The `/doctor` command can propose trims for a checked-in CLAUDE.md automatically, cutting content Claude can derive from the codebase while keeping pitfalls and conventions. (Requires Claude Code v2.1.206+.) ([memory](https://code.claude.com/docs/en/memory))

**Auto memory.** On by default. As Claude works, it writes durable notes to `~/.claude/projects/<project>/memory/` — build commands, debugging insights, your corrections — without you asking. `MEMORY.md` is the always-loaded index (kept under ~200 lines); detailed notes overflow into topic files. You can say "remember: always use pnpm, not npm" directly in conversation, or "forget the rule about pnpm" to undo it. Run `/memory` to browse, toggle, or edit what's been saved — it's all plain markdown. ([memory](https://code.claude.com/docs/en/memory))

**Real scenario.** You correct Claude three sessions in a row for using `npm` instead of your project's `pnpm`. With auto memory, by the third correction Claude has usually already written itself a note and stopped making the mistake — you didn't have to add anything to CLAUDE.md by hand. If it *keeps* happening, that's a sign to promote the rule into CLAUDE.md explicitly, since auto memory is a courtesy, not a guarantee.

[⬆ back to top](#table-of-contents)

---

## 4. Everyday Workflows

These are prompt patterns for the tasks you'll do constantly. All work identically across surfaces. ([common-workflows](https://code.claude.com/docs/en/common-workflows))

**Exploring unfamiliar code:**
```bash
claude "explain how authentication works in this codebase and show me the key files"
```

**Fixing a bug from an error message** — paste the error, don't pre-diagnose it:
```bash
# Less effective: you've already guessed the cause, narrowing Claude's search
claude "the JWT expiry check must be wrong, fix it"

# More effective: give the symptom, let Claude trace the actual cause
claude "I'm getting 'jwt malformed' on login after 24h uptime. Find the cause and fix it."
```

**Refactoring:**
```bash
claude "update this file to use Composition API patterns consistent with the rest of the codebase"
```

**Tests** — be specific about *behavior*, not just "add tests":
```bash
claude "add tests for the checkout validator covering: missing email, expired discount code,
and quantity below the minimum order size"
```
Claude examines your existing test files first to match style and conventions already in use — you get tests that look like they were written by your team, not a generic template.

**Resuming across sittings:**
```bash
claude -c              # continue the most recent session in this directory
claude -r "session-name" "continue this PR"   # resume a specific named session
```

**Parallel work with git worktrees** — run two Claude Code sessions on the same repo without them stepping on each other's edits (e.g. one fixing a bug while another builds a feature):
```bash
git worktree add ../myproject-feature-x feature-x
cd ../myproject-feature-x && claude
```

**Plan before editing** — for anything risky or unfamiliar, ask Claude to propose a plan first, review it, then let it execute:
```bash
claude "propose a plan to migrate this table's soft-deletes, but don't make any changes yet"
```
This maps to **Plan mode**, which restricts Claude to read-only exploration and proposal — no file writes, no commands that mutate state — until you approve. ([Best Claude Code Plugins](https://www.marktechpost.com/2026/06/14/claude-code-guide-2026-25-features-with-examples-demo/))

**Delegating research to keep context clean** — see §6 (Subagents) for why this matters.

[⬆ back to top](#table-of-contents)

---

## 5. Permissions & Safety

**What it is.** Every tool call Claude wants to make — editing a file, running a shell command, calling an MCP tool — passes through a permission check first. The result is one of three: **allow** (proceeds silently), **ask** (you're prompted), or **deny** (blocked outright). ([permissions](https://code.claude.com/docs/en/permissions))

**Why it matters — real scenario.** In 2026, multiple developers reported losing significant work after granting an autonomous coding agent broad filesystem/infra access without oversight — including at least one case of a production database being wiped by an agent stuck in a correction loop. Claude Code's permission system exists specifically to keep a human in the loop for anything destructive, and to make it possible to *safely* automate the rest. ([neurohive guide](https://neurohive.io/en/guides/claude-code-getting-started-guide/))

**Evaluation order** — this is the part people get wrong:

```
   deny  ─────▶  ask  ─────▶  allow  ─────▶  (no match) defaultMode
  (blocks,     (prompts,     (proceeds
  first match   first match   silently)
  wins)         wins)
```
A `deny` rule always wins, even against a broader `allow` elsewhere — a `deny` on `Read(./.env)` blocks it even if you also `allow` `Read(./**)`. ([permissions](https://code.claude.com/docs/en/permissions))

**Rule syntax** — `Tool` or `Tool(specifier)`:
```json
{
  "permissions": {
    "allow": ["Bash(git status)", "Bash(git diff:*)", "Bash(npm run lint:*)", "Read(./**)"],
    "deny":  ["Bash(rm -rf:*)", "Read(./.env)", "Read(./.env.*)"],
    "ask":   ["Bash(git push:*)"]
  }
}
```
```bash
# Wrong — assumes this blocks the .env file from ever reaching Claude's context
"deny": ["Read(./.env)"]
# This only blocks the Read tool (and a few related built-ins like Grep/Glob) from
# that specific file. Another tool path could still surface its contents.
# Right — for anything truly sensitive, don't rely on a deny rule alone;
# keep secrets out of the working directory Claude has access to, or use a hook (§8).
```

**Permission modes** (set via `defaultMode` or `--permission-mode`):

| Mode | Behavior |
|---|---|
| `default` | Prompts on first use of each tool |
| `acceptEdits` | File reads/writes proceed freely; shell commands still confirm |
| `plan` | Read-only; Claude can analyze and propose but not execute |
| `bypassPermissions` (`--dangerously-skip-permissions`) | Skips all checks |

`bypassPermissions` should only be used inside a fully isolated environment — a container, VM, or CI runner with no real credentials — never on your local machine. Claude Code even refuses to start in this mode with root/sudo privileges on Linux/macOS, specifically to prevent this misuse. ([permissions](https://code.claude.com/docs/en/permissions), [dangerously-skip-permissions guide](https://www.morphllm.com/claude-code-dangerously-skip-permissions))

Manage rules interactively any time with `/permissions` (aliased `/allowed-tools`).

[⬆ back to top](#table-of-contents)

---

## 6. Subagents

**What it is.** A subagent is a separate Claude instance the main session spawns for a scoped task. It gets its own context window, its own system prompt, and its own tool permissions — and when it finishes, only its *summary* comes back to the main conversation. ([sub-agents](https://code.claude.com/docs/en/sub-agents), [subagents blog](https://claude.com/blog/subagents-in-claude-code))

**Why it matters.** Context is a limited, shared resource. If you ask Claude to "find every call site of this function across the repo," it might read 40 files — and now those 40 files are crowding your main conversation, slowing every future turn and pushing you toward auto-compaction. A subagent absorbs that cost in its own isolated window and hands back only the conclusion.

```
 Main session (your conversation)
 │
 ├──▶ Explore subagent   (read-only search)         ──▶ returns: "auth logic is in
 │                                                        middleware/auth.ts, called
 │                                                        from 6 route files: [...]"
 ├──▶ Plan subagent      (research + propose plan)  ──▶ returns: implementation plan
 │
 └──▶ general-purpose subagent (multi-step task)    ──▶ returns: result summary

 Only the returned summaries enter the main context — not the 40 files each subagent read.
```

**Built-in subagents:** `Explore` (fast, read-only code search), `Plan` (research-then-propose, no edits), and `general-purpose` (multi-step tasks). Claude often spawns these on its own; you can also direct it explicitly:
```bash
claude "use a subagent to find every place we call the deprecated PhilHealth claims API,
then summarize the call sites — don't change anything yet"
```

**Custom subagents** are markdown files with YAML frontmatter, in `.claude/agents/` (project, shared with the team) or `~/.claude/agents/` (personal, all projects):
```markdown
---
name: code-reviewer
description: Reviews a recent diff and reports issues by severity.
tools: Read, Grep, Glob, Bash
---
You are a code reviewer. Read the most recent diff, check for bugs, security
issues, and style problems, and return a prioritized list with file/line refs.
```
Once defined, Claude delegates to it automatically whenever a task matches the `description` — no need to invoke it by name every time. The `/agents` command walks through creating one interactively. ([subagents blog](https://claude.com/blog/subagents-in-claude-code))

**Foreground vs. background:** foreground subagents block the main conversation until done, with permission prompts passed through to you. Background subagents run concurrently while you keep working — press `Ctrl+B` to send a running subagent to the background, and check on it with `/tasks`. ([subagents blog](https://claude.com/blog/subagents-in-claude-code))

**Real scenario.** Reviewing a large PR: rather than one agent doing style checks, security scanning, and test-coverage analysis sequentially, you spawn three specialized subagents in parallel — `style-checker`, `security-scanner`, `test-coverage` — cutting review time from minutes to seconds versus doing it one pass at a time. ([Agent SDK subagents](https://platform.claude.com/docs/en/agent-sdk/subagents))

**Common mistake:** reaching for a subagent for every small task. A subagent has real overhead (spinning up its own context, then summarizing back). Reserve it for genuinely token-heavy exploration or parallelizable work — not a two-line question you could just ask directly.

[⬆ back to top](#table-of-contents)

---

## 7. Skills & Slash Commands

**What it is.** A skill is a folder — `.claude/skills/<name>/` (project) or `~/.claude/skills/<name>/` (personal) — containing a `SKILL.md` file with YAML frontmatter (`name`, `description`) and markdown instructions. Claude reads every skill's description at session start and can invoke the matching one automatically when relevant — or you can invoke it directly with `/<name>`. A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create a `/deploy` command and work the same way; your existing `.claude/commands/` files keep working unchanged. ([skills](https://code.claude.com/docs/en/skills))

**Why the description field is the whole game.** Claude matches your request against the `description` to decide what fires. A vague description ("manages my project") won't match a concrete request ("set up the project here") — write the description using the literal phrases you'd actually type. This is the single most common reason a skill never triggers. ([skill-creation guide](https://codemeetai.substack.com/p/how-to-create-a-claude-code-skill))

```markdown
# Wrong — abstract, won't match real requests
---
name: deploy
description: Handles deployment tasks.
---

# Right — concrete phrases a user would actually type
---
name: deploy
description: Deploy the app to staging or production. Use when the user says
"deploy to staging", "ship this to prod", or "push the release".
---
```

**Minimum viable skill** is about 30 lines. Anything beyond required frontmatter + instructions (subdirectories for scripts, `references/` for background docs Claude reads but doesn't copy, `templates/` for files Claude copies with placeholders) is optional. ([skill-creation guide](https://codemeetai.substack.com/p/how-to-create-a-claude-code-skill))

**Built-in bundled skills:** `/doctor` (diagnostics), `/code-review`, `/batch`, `/debug`, `/loop`. These are prompt-based — they give Claude detailed instructions and let it orchestrate the work with its normal tools, rather than running external code. ([skills](https://code.claude.com/docs/en/skills))

**Trust boundary — flagged deliberately:** skills give Claude new instructions and can direct it to invoke tools in ways that don't match the skill's stated purpose. Only use skills from trusted sources — ones you wrote yourself or that come from Anthropic. Treat a downloaded community skill the way you'd treat an unreviewed dependency. ([Agent Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview))

**Real scenario.** You keep pasting the same "run migrations, seed test data, start the dev server" checklist into chat before every testing session. Turned into a `/verify` skill, it becomes a one-word command — and if Claude has to run it without a recorded recipe, newer versions can *write* the steps that worked back into the skill file automatically, so later runs (yours or a teammate's) follow the same steps. (Requires Claude Code v2.1.200+.) ([skills](https://code.claude.com/docs/en/skills))

[⬆ back to top](#table-of-contents)

---

## 8. Hooks

**What it is.** Hooks are shell commands (or LLM-evaluated prompts) that fire automatically at specific points in Claude Code's lifecycle — before a tool runs, after it runs, when a session starts, when Claude tries to stop. Configured as JSON in your settings files. ([hooks](https://code.claude.com/docs/en/hooks))

**Why this is the most important mental-model shift after the agentic loop itself:** CLAUDE.md and skills are *instructions* — Claude reads them and tries to comply, but there's no runtime guarantee. If something must happen *every single time, no exceptions*, that belongs in a hook instead, because a hook is enforced by the harness, not by the model choosing to comply.

```
                 Tool call about to happen
                          │
                          ▼
                 ┌─────────────────┐
                 │   PreToolUse    │──── exit code 2 ──▶ BLOCKED
                 │      hook       │      (stderr shown to the model as the reason)
                 └────────┬────────┘
                          │ allowed
                          ▼
                    permission check (§5: deny → ask → allow)
                          │
                          ▼
                     tool executes
                          │
                          ▼
                 ┌─────────────────┐
                 │  PostToolUse    │──── runs formatters, linters,
                 │      hook       │     validators after the fact
                 └─────────────────┘
```

Important nuance: a hook can only make things *stricter*, never looser. If a hook returns "allow" but a `deny` or `ask` rule in settings still matches, the settings rule wins — a hook cannot grant more access than the permission system already allows. ([permissions](https://code.claude.com/docs/en/permissions))

**Auto-format after every edit (PostToolUse):**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "f=$(jq -r '.tool_input.file_path'); [ \"${f##*.}\" = ts ] && npx prettier --write \"$f\"; true" }
        ]
      }
    ]
  }
}
```
The trailing `; true` matters — without it, a formatter error would block Claude for no good reason. ([hooks example, dev.to](https://dev.to/ohugonnot/claude-code-hooks-real-examples-posttooluse-stop-pretooluse-620))

**Block edits to protected files (PreToolUse):**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "FILE=$(cat | jq -r '.tool_input.file_path // empty'); if echo \"$FILE\" | grep -qE '(\\.env|\\.lock|id_rsa|\\.pem)'; then echo \"BLOCKED: protected file\" >&2; exit 2; fi" }
        ]
      }
    ]
  }
}
```
```bash
# Wrong — relying on a CLAUDE.md line: "never edit .env files"
# This is a request. A determined or confused agent can still edit it.

# Right — enforce it with a PreToolUse hook that exits 2 (see above).
# The instruction becomes structurally impossible to violate, not just discouraged.
```

**Key events worth knowing** (there are more — see the reference): `PreToolUse`, `PostToolUse`, `Notification`, `Stop` (Claude finished responding), `SubagentStop`, `SessionStart`, `PreCompact`. ([hooks reference](https://code.claude.com/docs/en/hooks), [hooks guide](https://www.pixelmojo.io/blogs/claude-code-hooks-production-quality-ci-cd-patterns))

**The decision rule to internalize:** *Skill* teaches the how. *Hook* enforces the rule. *Subagent* isolates the work. Use whichever matches what you actually need — a skill for a repeatable workflow, a hook for a non-negotiable, a subagent to protect context.

[⬆ back to top](#table-of-contents)

---

## 9. MCP (Model Context Protocol)

**What it is.** MCP is an open standard for connecting Claude Code to tools beyond its built-ins — an issue tracker, a database, a browser, your team's internal tooling. These come from MCP *servers*, which run locally or as a hosted service. ([mcp-quickstart](https://code.claude.com/docs/en/mcp-quickstart))

**Adding a server:**
```bash
# HTTP transport (recommended — more reliable than SSE)
claude mcp add --transport http notion https://mcp.notion.com/mcp

# Local stdio server (runs a program on your machine)
claude mcp add --transport stdio airtable --env AIRTABLE_API_KEY=YOUR_KEY -- npx -y airtable-mcp-server
```
Inside a session, `/mcp` checks and manages servers you've already added.

**Scopes:**

| Scope | Flag | Visibility |
|---|---|---|
| local (default) | *(none)* | Private to you, this project only |
| project | `--scope project` | Shared with the team via `.mcp.json`, checked into git |
| user | `--scope user` | Available across all your projects |

```bash
claude mcp add --transport http github --scope project https://api.githubcopilot.com/mcp/
```

**Why it matters — cost, not just capability.** Each connected server's tool names and instructions load into *every* session's context, whether you use them that turn or not. Remove servers you no longer use to keep that space free. ([mcp-quickstart](https://code.claude.com/docs/en/mcp-quickstart))

```bash
# Wrong (Windows-specific gotcha) — npx doesn't run directly on Windows this way
claude mcp add --transport stdio my-server -- npx -y @some/package
# Fails with "Connection closed" on native Windows.

# Right — wrap it
claude mcp add --transport stdio my-server -- cmd /c npx -y @some/package
```

**Real scenario for your stack:** connecting a GitHub MCP server means you can ask "review PR #456 and suggest improvements" or "create an issue for the bug we just found" without leaving the terminal — useful for the SSO admin panel and Activity Tracker repos if they live on GitHub. ([MCP servers guide](https://www.builder.io/blog/claude-code-mcp-servers))

[⬆ back to top](#table-of-contents)

---

## 10. Plugins & Marketplaces

**What it is.** A plugin bundles skills, subagents, slash commands, hooks, and MCP server definitions into one versioned, installable unit — install with `/plugin install <name>` and it works across terminal and VS Code alike. A marketplace hosts a collection of plugins (Anthropic's own, or a community/company one). ([plugins blog](https://claude.com/blog/claude-code-plugins))

```bash
/plugin marketplace add <marketplace-repo>
/plugin install <plugin-name>@<marketplace-name>
```

**Why it matters for teams:** instead of every developer hand-rolling their own skills, hooks, and agents, a plugin captures the whole set once and every team member gets the identical setup with one command. If your team ever standardizes an AdonisJS+Nuxt workflow (migration conventions, PR review rules, deploy steps), a plugin is how you'd package and share it — rather than everyone copying `.claude/` folders around.

**A well-known example:** the community "Superpowers" plugin reframes Claude Code from a single agent into a small structured dev team — brainstorming, sub-agent-driven development, TDD enforcement, code review — and is now an official entry in Anthropic's marketplace. ([DataCamp best practices](https://www.datacamp.com/tutorial/claude-code-best-practices))

**Common mistake:** installing plugins speculatively. Each one adds to your context load the same way an MCP server does. Start with the one plugin that solves an actual friction point, and expand from there rather than stacking many at once. ([Best Claude Code Plugins](https://buildtolaunch.substack.com/p/best-claude-code-plugins-tested-review))

[⬆ back to top](#table-of-contents)

---

## 11. Automation & CI

**What it is.** Claude Code follows the Unix philosophy — composable, pipeable, scriptable via `-p` (print/headless mode). No interactive session, no approval prompts (assuming your permissions are configured to allow what's needed). ([overview](https://code.claude.com/docs/en/overview))

```bash
# Analyze a log stream for anomalies
tail -200 app.log | claude -p "Slack me if you see any anomalies"

# Automate translations as part of CI
claude -p "translate new strings into French and raise a PR for review"

# Bulk review of changed files
git diff main --name-only | claude -p "review these changed files for security issues"
```

**Structured output for scripting** — `--output-format json` gives a parseable envelope (including cost and duration), useful for monitoring:
```bash
claude -p --output-format json "List all REST endpoints in the API"
```

```bash
# Wrong — assuming -p prompts for permission like an interactive session
claude -p "deploy the app"
# In CI, with no one watching, this either hangs or (worse) silently proceeds
# under whatever defaultMode is set — check your permissions before scripting this.

# Right — be explicit about what's safe to automate
claude -p --permission-mode acceptEdits "run tests and fix any that fail"
# Reserve bypassPermissions strictly for fully sandboxed CI runners (see §5).
```

**CI integration:** GitHub Actions and GitLab CI/CD integrations let Claude Code automate PR review and issue triage directly in your pipeline. ([overview](https://code.claude.com/docs/en/overview))

**Scheduling recurring work:**
- **Routines** run in the cloud (keep running even with your machine off) — create from the web, Desktop app, or `/schedule`.
- **Desktop scheduled tasks** run on your machine, with direct local file/tool access.
- **`/loop`** repeats a prompt within a single CLI session for quick polling.

[⬆ back to top](#table-of-contents)

---

## 12. Beyond the Terminal

Newer surfaces worth knowing exist, even if you mostly live in the terminal day-to-day:

| Surface | What it's for |
|---|---|
| **Desktop app** | Standalone app; review diffs visually, run multiple sessions side by side, schedule recurring tasks |
| **Web** (claude.ai/code) | No local setup; kick off long tasks and check back later, or work on repos you don't have locally |
| **Remote Control** | Continue a local session from your phone or another device |
| **Dispatch** | Message a task from your phone, opens the Desktop session it creates |
| **Teleport** (`claude --teleport`) | Pull a task started on web/mobile into your terminal |
| **`/desktop`** | Hand off your current terminal session into the Desktop app for visual diff review |
| **Slack integration** | Mention `@Claude` with a bug report, get a pull request back |
| **Chrome integration** | Debug live web applications directly |

All connect to the same underlying engine — your CLAUDE.md, settings, and MCP servers apply regardless of which surface you're using. ([overview](https://code.claude.com/docs/en/overview))

**Why this matters for you specifically:** given you work across a Windows/Laragon local setup, the Desktop app's visual diff review plus `/desktop` handoff from a terminal session is worth trying once — it can make reviewing a large agent-generated diff meaningfully easier than scrolling through terminal output.

[⬆ back to top](#table-of-contents)

---

## 13. Real-World Patterns & Common Mistakes

**The failure modes that actually bite people in autonomous operation** ([neurohive guide](https://neurohive.io/en/guides/claude-code-getting-started-guide/)):
- An endless correction loop, where each fix introduces a new regression
- References to file paths that don't exist in the repo
- Breaking changes to shared modules the current task didn't need to touch
- Test regressions in areas unrelated to the original task

**Mitigations that map directly to sections above:**
- Work in a separate git branch; make checkpoint commits before large tasks (Claude Code also auto-snapshots before changes — press Escape twice to rewind when something breaks)
- Don't run fully autonomous (`bypassPermissions`) on anything with real credentials — isolate in a container/VM/CI runner instead (§5)
- Use Plan mode for anything you're not confident about, so you review before anything executes (§4)
- Push non-negotiables into hooks rather than CLAUDE.md wording (§8)

**Common mistake — an over-stuffed CLAUDE.md.** Treating it like project documentation rather than a set of things Claude genuinely can't infer bloats every session's starting context and, per the docs, can *reduce* instruction-following consistency rather than improve it. Prefer trimming it toward pitfalls, rationale, and conventions that differ from tool defaults; let `/doctor` help. (§3)

**Common mistake — reaching for a subagent or plugin for everything.** Both have real context overhead. Reserve subagents for genuinely token-heavy or parallelizable work, and plugins for repeated friction, not speculative "might need this later" installs. (§6, §10)

**Common mistake — confusing "wrote it in CLAUDE.md" with "enforced."** If a mistake keeps recurring despite being in CLAUDE.md, that's the signal to move it to a hook, not to write a stronger sentence. (§8)

[⬆ back to top](#table-of-contents)

---

## 14. Cheat Sheets

### CLI essentials
| Command | Purpose |
|---|---|
| `claude` | Start interactive session |
| `claude "task"` | Interactive session with an initial prompt |
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
|---|---|
| `Bash` / `Bash(*)` | All Bash commands |
| `Bash(npm run:*)` | Any `npm run ...` variant |
| `Read(./.env)` | That exact file, for Read (and a few related built-ins) |
| `Read(./**)` | Everything under the project root |
Evaluation order: **deny → ask → allow → defaultMode fallback.**

### Hook events (most common)
| Event | Fires | Can block? |
|---|---|---|
| `PreToolUse` | Before any tool executes | Yes (`exit 2`) |
| `PostToolUse` | After a tool succeeds | No (side effects only) |
| `Stop` | Claude is about to finish responding | Yes |
| `SubagentStop` | A subagent finishes | Yes |
| `SessionStart` | New session begins | — |

### Memory scopes (broadest → narrowest)
Enterprise policy → Project (`./CLAUDE.md`) → User (`~/.claude/CLAUDE.md`) → Project-local (`./CLAUDE.local.md`)

[⬆ back to top](#table-of-contents)

---

## 15. Suggested Learning Order

1. **Install and run one real task** (§2, §4) — pick a small, low-risk bug fix in a project you know well. Get comfortable reviewing a Claude-generated diff.
2. **Write a minimal CLAUDE.md** (§3) — just the pitfalls and conventions your project needs, under 50 lines to start.
3. **Configure permissions deliberately** (§5) — don't default to clicking "allow" on everything; set up `allow`/`deny` for your common safe commands so you stop rubber-stamping `git status`.
4. **Try one hook** (§8) — start with a PostToolUse formatter hook; it's low-risk and the feedback is immediate.
5. **Delegate one research task to a subagent** (§6) — notice how it keeps your main context clean.
6. **Write one custom skill** (§7) — turn a checklist you paste repeatedly into a `/`-command.
7. **Connect one MCP server** (§9) — GitHub is a good first choice if your repos live there.
8. **Automate one recurring task** (§11) — a `-p` script or a scheduled routine for something you do weekly.
9. Explore plugins (§10) and the non-terminal surfaces (§12) once the fundamentals feel automatic.

## 16. Quick Self-Check

- What's the difference between what CLAUDE.md does and what auto memory does — and who writes each?
- Why can't a CLAUDE.md instruction alone guarantee Claude never touches a file — what mechanism actually guarantees that?
- In what order are permission rules evaluated, and which one wins on a conflict?
- When would you reach for a subagent instead of just asking directly in the main session?
- What's the actual trigger mechanism for a skill — and what's the most common reason one fails to fire?
- Why does connecting an MCP server have a cost even in a turn where you don't use it?
- Name two mitigations for the "endless correction loop" failure mode in autonomous operation.
- What's the practical difference between `claude` and `claude -p` for a CI script?

[⬆ back to top](#table-of-contents)
