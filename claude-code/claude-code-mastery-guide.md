# Claude Code — Mastery Guide (Overview)

The entry point for this repo's Claude Code material. This document holds the shared spec for the whole set, the from-zero primer on what an agentic coding tool actually is, install and first-session setup, the map of the focused sub-docs, the "what recent releases changed under you" delta, and the reference apparatus (learning order, self-check, cross-doc cheat sheet).

The actual teaching lives in the sub-docs. Start here, then follow the [Suggested Learning Order](#8-suggested-learning-order).

---

## About This Document

This section is the canonical spec for **every doc in this set** — the sub-docs each carry a short pointer back here rather than repeating it.

- **Framing: goal-driven.** The goal is agentic engineering — moving from writing every line yourself to delegating work to an agent and reviewing outcomes. Depth, examples, and prioritisation are aimed at the commands, config files, flags, and lifecycle events you will actually touch while doing that, not at general topical coverage. Where a section could go either way, it goes toward "what do I type, and what breaks in practice."
- **Confirmed scope: the Claude Code CLI, deeply.** Sessions and the interactive loop, context and memory, settings and configuration, permissions and sandboxing, the extension layer (skills, subagents, hooks, MCP), and automation (headless mode, CI, scheduling, parallel agents, plugins).
- **Deliberately excluded** (with pointers): the Claude Agent SDK, the Claude API itself, enterprise deployment (Bedrock / Google Cloud's Agent Platform / Microsoft Foundry, LLM gateways, self-hosted environments, ZDR and compliance), and the non-terminal surfaces (Desktop, web, Slack, mobile, JetBrains). VS Code and the web/`--teleport` handoff get a short orientation only, since they share the same engine and configuration. See [§7](#7-deliberately-not-covered-and-where-to-look-instead).
- **Assumed background: none, on Claude Code specifically.** This set is written for someone who has never run `claude`. It assumes you are a working developer — comfortable with a terminal, git, JSON, YAML, and Markdown — but it teaches the agentic model, the permission model, and every Claude Code concept from zero. The primer is [§4](#4-primer-what-an-agentic-coding-tool-actually-is) of this document; read it before the sub-docs if "agentic loop" or "context window" are fuzzy.
- **Depth tiers.** Each Part runs **Beginner → Working Knowledge → Advanced → Mastery**, so you can stop at whichever level you need today. Tiers are collapsed where one would be filler, and each collapse is stated inline. Collapsing is expected behaviour, not a defect to fix on a later pass. The weight of this set sits in Working Knowledge and Advanced.
- **Per-section ingredients**, used where they earn their place rather than forced into every subsection:
  - a short plain-language explanation of what a thing is and why it matters;
  - a **wrong vs. right** snippet pair wherever a gotcha has a common wrong form;
  - a **Real Scenario** — a production-style incident where the concept actually bites someone. This is a goal-driven set, so scenarios are incidents rather than tutorial exercises;
  - an **ASCII diagram** where the concept is spatial or structural — the agentic loop, the settings precedence chain, the context window's composition, the hook lifecycle.
- **Sourcing rule.** [code.claude.com/docs](https://code.claude.com/docs/en/overview) is the primary source and is cited inline at the specific claim it supports. Where a popular-but-outdated convention disagrees with the official docs, the official docs win and the doc says so explicitly — [§6](#6-what-recent-releases-changed-under-you) is entirely about those disagreements. **Deviation from the repo template, stated openly:** Stack Overflow was not consulted for this set. Claude Code ships weekly and its behaviour changes faster than SO answers age out, so a two-year-old SO answer about `.claude/commands/` or the permission model is more likely to mislead than to help. Real-world gotchas are instead sourced to the caveats the official docs state about themselves, which are unusually detailed and version-stamped, and to the [changelog](https://code.claude.com/docs/en/changelog) and [weekly release notes](https://code.claude.com/docs/en/whats-new/index).
- **Written against.** **Claude Code v2.1.263**, the current stable release ([changelog](https://code.claude.com/docs/en/changelog)). Verified **September 6, 2026**. Claude Code ships multiple releases a week and the docs version-stamp individual behaviours down to the patch (you will see "requires v2.1.221 or later" throughout the official docs and in this set); treat any version note here as a floor, not a ceiling.
- **To update this doc later.** *Preserve*: the goal-driven framing, the numbered Part structure with explicit tier subheadings, the wrong-vs-right pairs, the Real Scenario blocks, the TOC/back-to-top/anchor apparatus, and the per-Part cheat sheets. *Re-verify against current docs first*: **everything in [§6](#6-what-recent-releases-changed-under-you)**, which is the fastest-rotting section here; the model aliases and IDs in [Config & Settings](./claude-code-config-settings.md), which rotate every few months; the auto mode classifier's blocked-category list, which grows release over release; the hook event list, which gains events regularly; and the CLI flag table, which is long and volatile.

---

## Table of Contents

1. [How This Set Is Organised](#1-how-this-set-is-organised)
2. [The Sub-Docs](#2-the-sub-docs)
3. [Install, Authenticate, First Session](#3-install-authenticate-first-session)
4. [Primer: What an Agentic Coding Tool Actually Is](#4-primer-what-an-agentic-coding-tool-actually-is)
5. [The One-Paragraph Mental Model](#5-the-one-paragraph-mental-model)
6. [What Recent Releases Changed Under You](#6-what-recent-releases-changed-under-you)
7. [Deliberately Not Covered (and where to look instead)](#7-deliberately-not-covered-and-where-to-look-instead)
8. [Suggested Learning Order](#8-suggested-learning-order)
9. [Quick Self-Check](#9-quick-self-check)
10. [Cross-Doc Cheat Sheet](#10-cross-doc-cheat-sheet)

---

## 1. How This Set Is Organised

Claude Code is 14 Parts of teaching material spread over 6 sub-docs, plus this overview. The split follows the order in which the concepts actually become necessary — the daily loop first, because everything else (memory, settings, permissions, extensions, automation) is a way of shaping one step of that same loop:

```text
  claude-code-mastery-guide.md   ← you are here: spec, primer, setup, map, delta, reference
        │
        ├── claude-code-daily-driver.md          Parts 1–3    the loop you live in
        ├── claude-code-context-memory.md        Parts 4–5    what Claude knows, and for how long
        ├── claude-code-config-settings.md       Parts 6–7    the knobs and where they live
        ├── claude-code-permissions-security.md  Parts 8–9    what Claude may do, and what stops it
        ├── claude-code-extensibility.md         Parts 10–12  skills, subagents, hooks, MCP
        └── claude-code-automation-agents.md     Parts 13–14  headless, CI, parallelism, plugins
```

Each Part is self-contained enough to reopen on its own, and each ends with a quick-reference table. Within a Part, the tiers mean:

| Tier | What you get |
|---|---|
| **Beginner** | What it is, the minimal correct usage, nothing else |
| **Working Knowledge** | Everyday patterns, the flags and files you actually reach for |
| **Advanced** | Edge cases, gotchas, correctness and security implications |
| **Mastery** | Internals, powerful-but-rare capabilities, expert tradeoffs |

Where a tier would be filler, it is collapsed and the doc says so — e.g. *"Settings precedence gets Beginner → Advanced only; its 'Mastery' material is really the permissions evaluation order, which Part 8 covers properly."*

A note on how to read a goal-driven set: **the Advanced tier is where the money is.** Beginner and Working Knowledge get you typing; Advanced is where you learn why the agent did something surprising at 2am, which is the actual skill.

[↑ Back to top](#table-of-contents)

---

## 2. The Sub-Docs

### [`claude-code-daily-driver.md`](./claude-code-daily-driver.md) — Parts 1–3

| Part | Covers |
|---|---|
| **1. The session loop** | Starting sessions, the prompt input, interrupting and steering, keyboard shortcuts, shell mode (`!`), message queueing, background Bash, the transcript viewer, vim mode |
| **2. Sessions, checkpoints, and undo** | `--continue` vs. `--resume` vs. `--fork-session`, the session picker, naming, `/branch`, `/rewind` and checkpointing, what checkpoints do *not* cover |
| **3. Getting good output** | Plan mode as a workflow, the verification loop, prompt specificity, `@` references and images, the failure patterns that waste the most time |

### [`claude-code-context-memory.md`](./claude-code-context-memory.md) — Parts 4–5

| Part | Covers |
|---|---|
| **4. The context window** | What loads at startup and what it costs, `/context`, compaction (auto and manual), what survives compaction, `/clear` vs. `/compact` vs. summarize-from-here, auto-compact windows |
| **5. Memory that persists** | `CLAUDE.md` hierarchy and load order, `@` imports, `.claude/rules/` and path-scoped rules, auto memory (`MEMORY.md`), `/init`, `/memory`, monorepo strategies |

### [`claude-code-config-settings.md`](./claude-code-config-settings.md) — Parts 6–7

| Part | Covers |
|---|---|
| **6. Settings files** | The five-layer precedence chain, `settings.json` vs. `settings.local.json` vs. managed, `--settings` and `--setting-sources`, the key inventory, `env`, debugging config with `/doctor` and `/status` |
| **7. Model, output, and interface** | Model aliases and resolution order, effort levels, extended thinking, 1M context, fast mode, output styles, the status line, terminal config, environment variables |

### [`claude-code-permissions-security.md`](./claude-code-permissions-security.md) — Parts 8–9

| Part | Covers |
|---|---|
| **8. The permission model** | The six permission modes, `Shift+Tab` cycling, permission rule syntax in full (Bash, Read/Edit, WebFetch, MCP, Agent, Cd), wildcards and their traps, protected and critical paths, workspace trust |
| **9. Containment** | Auto mode's classifier and what it blocks, the Bash sandbox, sandbox environments, prompt injection as a threat model, what to lock down in a shared repo |

### [`claude-code-extensibility.md`](./claude-code-extensibility.md) — Parts 10–12

| Part | Covers |
|---|---|
| **10. Skills** | `SKILL.md` structure and every frontmatter field, invocation and arguments, dynamic context injection, `context: fork`, visibility control, skills vs. `.claude/commands/` |
| **11. Subagents** | Agent definition files, tool and model restriction, what loads at startup, foreground vs. background, forks, built-in agents, concurrency and depth limits |
| **12. Hooks and MCP** | The hook event catalogue, matchers, the five hook types, the input/output contract, exit-code semantics; MCP transports, scopes, tool naming, tool search, and permissioning |

### [`claude-code-automation-agents.md`](./claude-code-automation-agents.md) — Parts 13–14

| Part | Covers |
|---|---|
| **13. Headless and CI** | `claude -p`, output formats and their schemas, `--bare`, structured output with `--json-schema`, budget and turn caps, permission handling with nobody watching, GitHub Actions, fan-out patterns |
| **14. Parallelism and packaging** | Worktrees, background agents and agent view, dynamic workflows, cross-session messaging, scheduled work (`/loop`, routines), goals, channels, and plugins/marketplaces |

[↑ Back to top](#table-of-contents)

---

## 3. Install, Authenticate, First Session

**Beginner tier.** This section exists so the sub-docs never have to stop and explain setup.

### Install

The **native installer is the recommended path** ([overview](https://code.claude.com/docs/en/overview)). Native installs auto-update in the background.

```bash
# macOS, Linux, WSL
curl -fsSL https://claude.ai/install.sh | bash
```

```powershell
# Windows PowerShell
irm https://claude.ai/install.ps1 | iex
```

Alternatives, none of which auto-update:

| Method | Command | Note |
|---|---|---|
| Homebrew | `brew install --cask claude-code` | `claude-code` tracks stable (≈a week behind); `claude-code@latest` tracks latest |
| WinGet | `winget install Anthropic.ClaudeCode` | `winget upgrade Anthropic.ClaudeCode` to update |
| Linux packages | apt, dnf, apk | See [advanced setup](https://code.claude.com/docs/en/setup#install-with-linux-package-managers) |

> **On native Windows**, install [Git for Windows](https://git-scm.com/downloads/win) so Claude Code has a Bash tool. Without it, Claude Code uses PowerShell as the shell tool instead — which works, but every Bash-shaped example you find online (including most of this set's shell snippets) needs translating. WSL setups don't need it.

### Authenticate

```bash
cd your-project
claude
```

You are prompted to log in on first use. Two credential paths:

- **Subscription** (Pro, Max, Team, Enterprise): `claude auth login`, or `/login` in a session. This is what you want for interactive work.
- **API key**: set `ANTHROPIC_API_KEY` and Claude Code skips the login prompt, asking you to approve the key instead. This is what you want in CI.

`claude auth status` prints the current state as JSON — useful in scripts.

### The first session

```bash
claude
```

Then type a request in plain English. Three commands are worth knowing immediately:

| Command | Why now |
|---|---|
| `/init` | Generates a starting `CLAUDE.md` for the project by reading your codebase. Do this once per repo. |
| `/doctor` | Setup checkup — diagnoses install problems, `PATH` issues, unparseable settings, slow hooks. Run it when anything feels wrong. |
| `/help` | Lists commands available to you right now, including skills and plugin commands. |

And two keys:

| Key | Effect |
|---|---|
| `Esc` | Stop Claude immediately. The running tool call is cancelled; context is preserved. |
| `Shift+Tab` | Cycle permission modes. This is the single most important key in the tool — see [Permissions & Security](./claude-code-permissions-security.md). |

**Wrong vs. right, the very first mistake.** New users treat Claude Code like autocomplete and dictate steps:

```text
# Wrong — you are doing the agent's job for it
Open src/auth/session.ts. Find the refreshToken function. On line 42,
change the expiry check to use Date.now() instead of new Date().
```

```text
# Right — describe the outcome and the constraint; let it find the code
Users report that login fails after session timeout. Check the auth flow in
src/auth/, especially token refresh. Write a failing test that reproduces the
issue, then fix it.
```

The second version is not just less typing — it produces better results, because the agent reads the surrounding code before deciding, and because you gave it a verifiable finish condition ([best practices](https://code.claude.com/docs/en/best-practices)).

[↑ Back to top](#table-of-contents)

---

## 4. Primer: What an Agentic Coding Tool Actually Is

**Beginner → Working Knowledge.** *There is no Advanced tier here; the advanced version of this material is the rest of the set.*

### The agentic loop

An autocomplete assistant predicts your next tokens. An **agentic** tool is given a goal and works toward it by taking actions and observing results. Claude Code runs three blended phases — **gather context, take action, verify results** — repeating until the task is done or you interrupt ([how it works](https://code.claude.com/docs/en/how-claude-code-works)):

```text
        ┌──────────────────────── you can interrupt at any point ───────┐
        │                                                               │
        ▼                                                               │
   your prompt                                                          │
        │                                                               │
        ▼                                                               │
  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐           │
  │ 1. GATHER     │───▶│ 2. ACT        │───▶│ 3. VERIFY     │           │
  │    read files │    │    edit files │    │    run tests  │───────────┘
  │    grep/glob  │    │    run cmds   │    │    read output│
  │    web fetch  │    │    git ops    │    │    check types│
  └───────────────┘    └───────────────┘    └───────────────┘
        ▲                                            │
        └──────────── result informs the next step ──┘
                    (loop until done, or until you stop it)
```

The loop is not a fixed script. A question about your codebase might only gather. A bug fix cycles through all three repeatedly. Claude decides what each step requires from what it learned in the previous one, chaining dozens of actions and course-correcting.

Two components power it:

- **The model** reasons. Different models trade capability for speed and cost — Sonnet handles most coding, Opus gives stronger reasoning for architectural decisions. Switch with `/model`.
- **The tools** act. Without tools a model can only emit text; with them it can read, edit, execute, and search.

Claude Code itself is the **agentic harness**: the thing that supplies the tools, manages the context, enforces permissions, and runs the execution environment. Every feature in this set — skills, hooks, subagents, MCP, permission modes — is a way of shaping one step of that loop. Hold onto that framing; it is the organising idea of the whole tool.

### The five tool categories

| Category | What Claude can do |
|---|---|
| **File operations** | Read files, edit code, create files, rename and reorganise |
| **Search** | Find files by pattern, search content with regex, explore codebases |
| **Execution** | Run shell commands, start servers, run tests, use git |
| **Web** | Search the web, fetch documentation, look up error messages |
| **Code intelligence** | See type errors after edits, jump to definitions, find references — requires a [code intelligence plugin](https://code.claude.com/docs/en/discover-plugins#code-intelligence) |

Plus orchestration tools: spawning subagents, asking you questions, managing tasks. Full inventory in the [tools reference](https://code.claude.com/docs/en/tools-reference).

### What Claude can see

When you run `claude` in a directory, it gains access to your project files (that directory and below, plus anything you add with `--add-dir`), your terminal, your git state, your `CLAUDE.md`, its own auto memory, and whatever extensions you have configured ([how it works](https://code.claude.com/docs/en/how-claude-code-works)).

This is the important difference from an inline assistant: **it sees the whole project.** "Fix the authentication bug" means it searches, reads several files, makes coordinated edits across them, runs the tests, and — if you ask — commits.

### The context window: the constraint everything else is about

Claude has a finite working memory called the **context window**. It holds the system prompt, your conversation, every file read, every command output, `CLAUDE.md`, auto memory, and loaded skills. It fills up fast — a single debugging session can consume tens of thousands of tokens — and **model performance degrades as it fills** ([best practices](https://code.claude.com/docs/en/best-practices)).

Almost every best practice in this set is downstream of that one fact:

- Why `/clear` between unrelated tasks? Context hygiene.
- Why put rules in `CLAUDE.md` instead of chat? Chat gets summarised away; `CLAUDE.md` is re-injected.
- Why delegate research to a subagent? Its file reads land in *its* context, not yours.
- Why does Claude "forget" your instruction from 40 messages ago? Compaction summarised it.

Part 4 in [Context & Memory](./claude-code-context-memory.md) covers this properly.

### Two safety mechanisms, up front

You are handing a program the ability to edit files and run shell commands. Two things stand between that and disaster:

1. **Checkpoints.** Before Claude edits a file, it snapshots the contents. Press `Esc` twice (on empty input) to open the rewind menu. Checkpoints are separate from git and survive resuming a session. **They only cover edits made through Claude's file-editing tools** — not changes made by Bash commands, and not most subagent edits ([checkpointing](https://code.claude.com/docs/en/checkpointing)).
2. **Permissions.** A permission mode sets what runs without asking. `Shift+Tab` cycles them. On Pro, Max, and Team plans the built-in starting mode for interactive terminal sessions is **auto mode**, where a classifier model reviews most actions in the background and blocks risky ones rather than asking you ([permission modes](https://code.claude.com/docs/en/permission-modes)).

Neither is a substitute for git. Commit before you let an agent loose.

[↑ Back to top](#table-of-contents)

---

## 5. The One-Paragraph Mental Model

Claude Code is a loop — gather, act, verify — wrapped around a model that has tools and a finite context window. Everything you configure is a way of shaping one step of that loop: **`CLAUDE.md` and rules** shape what enters context at the start; **skills** put knowledge and workflows in on demand; **MCP** adds tools that reach outside your machine; **subagents** run the loop in a separate context and hand back only a summary; **hooks** fire deterministic code at fixed lifecycle points so a rule holds regardless of what the model decides; **permission modes and rules** decide which actions execute without your say-so; and **checkpoints, worktrees, and sandboxes** decide how much damage a wrong decision can do before you notice. When the tool surprises you, the diagnosis is almost always one of three things: something was in context that shouldn't have been, something wasn't in context that should have been, or a permission layer said yes when you assumed it would ask.

[↑ Back to top](#table-of-contents)

---

## 6. What Recent Releases Changed Under You

This is the section that goes stale fastest, and the reason it exists: Claude Code ships several releases a week, and the blog posts and tutorials you will find by searching are frequently describing a tool that no longer behaves that way. Each row names the outdated convention explicitly so you recognise it in the wild.

| You'll read that... | Actually, as of v2.1.263 |
|---|---|
| "Install with `npm install -g @anthropic-ai/claude-code`" | The **native installer** is the recommended path (`curl -fsSL https://claude.ai/install.sh \| bash`, or `irm https://claude.ai/install.ps1 \| iex`), and it auto-updates. Homebrew and WinGet work but don't auto-update ([overview](https://code.claude.com/docs/en/overview)). |
| "Claude asks permission before every edit and command" | On **Pro, Max, and Team plans, auto mode is the built-in starting permission mode** for interactive terminal and VS Code sessions: a classifier model reviews most actions and blocks the risky ones instead of prompting you. The always-ask mode still exists and is now named **Manual** in the UI, config value `default` ([permission modes](https://code.claude.com/docs/en/permission-modes)). This is the single biggest behavioural change relative to older write-ups. |
| "Put custom slash commands in `.claude/commands/`" | `.claude/commands/` **still works**, but **skills are the recommended form** — `.claude/skills/<name>/SKILL.md`. Skills get frontmatter control (`disable-model-invocation`, `context: fork`, `allowed-tools`), supporting files in the skill directory, auto-invocation from the description, and dynamic context injection. When both exist with the same name, **the skill wins** ([skills](https://code.claude.com/docs/en/skills)). |
| "Run `/agents` to open the subagent creation UI" | As of **v2.1.198**, `/agents` prints a reminder to ask Claude to create subagents, or to edit `.claude/agents/` and `~/.claude/agents/` directly. The interactive builder is gone ([commands](https://code.claude.com/docs/en/commands)). |
| "MCP servers eat your context window" | **Tool search is on by default.** Only tool *names* and server instructions load at startup; full JSON schemas stay deferred and load on demand when Claude actually needs a tool ([features overview](https://code.claude.com/docs/en/features-overview)). Idle MCP servers are now cheap. Disable with `ENABLE_TOOL_SEARCH=false` if you need the old behaviour. |
| "`CLAUDE.md` is the only way Claude remembers anything across sessions" | **Auto memory** exists and is on by default. Claude writes its own notes to `~/.claude/projects/<project>/memory/`, indexed by `MEMORY.md`, of which the **first 200 lines or 25KB** loads every session. Toggle it in `/memory`, or with `autoMemoryEnabled` / `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` ([memory](https://code.claude.com/docs/en/memory)). |
| "Split a long `CLAUDE.md` with `@` imports to save context" | Imports help *organisation* but **not context** — imported files are expanded and loaded at launch alongside the file that references them. To actually reduce context, use **`.claude/rules/` with `paths:` frontmatter**, which loads only when Claude touches matching files ([memory](https://code.claude.com/docs/en/memory)). |
| "`keybindingFlavor` turns on readline-style word shortcuts" | Deprecated and has **no effect**. Readline conventions (`Alt+B`, `Alt+F`, `Alt+D`, `Ctrl+W`) are the behaviour in **v2.1.261 and later** ([interactive mode](https://code.claude.com/docs/en/interactive-mode)). |
| "Use `Ctrl+C` to interrupt Claude" | `Esc` interrupts and preserves context. `Ctrl+C` interrupts or clears input; `Ctrl+D` exits. Also: you can **type a correction and press `Enter` without stopping the running tool** — Claude reads it when the current action completes ([how it works](https://code.claude.com/docs/en/how-claude-code-works)). |
| "`--print` mode loads nothing extra, so it's safe in CI" | Without `--bare`, a `-p` run loads the same context an interactive session would — including hooks from the project's `.claude/settings.json` and servers from its `.mcp.json`, **in a folder you have never trusted**, with no trust dialog. `--bare` is the recommended mode for scripted and SDK calls and **will become the default for `-p` in a future release** ([headless](https://code.claude.com/docs/en/headless)). |
| "The latest models are Claude 3.5 / 4 / 4.5" | The current families are **Claude 5** (`claude-opus-5`, `claude-sonnet-5`, `claude-fable-5-1`) plus **Haiku 4.5** (`claude-haiku-4-5`). Sonnet 5 runs a **1M-token context window** on the Anthropic API with no `[1m]` variant to select ([model config](https://code.claude.com/docs/en/model-config)). |
| "Subagents are just parallel Claudes" | They are, but the ergonomics have grown: sessions run up to **20 concurrent subagents**, subagents can spawn subagents up to **3 layers deep**, they can hold their own **persistent memory** (`memory:` frontmatter), run in their own **git worktree** (`isolation: worktree`), and message each other ([sub-agents](https://code.claude.com/docs/en/sub-agents)). |

Two structural changes worth internalising beyond the table:

1. **The permission story is now three layers, not one.** Modes (what asks), rules (what's pre-approved or blocked), and the sandbox (what an approved command can actually reach) are independent and compose. Older write-ups treat permissions as a single allowlist. [Parts 8–9](./claude-code-permissions-security.md).
2. **"Slash command" now means four different things.** Built-in commands, bundled skills (`/code-review`, `/doctor`, `/batch`), your own skills, and MCP-server prompts all appear in the same `/` menu. When something in the menu behaves unexpectedly, the first question is which of the four it is — `/help` groups them.

[↑ Back to top](#table-of-contents)

---

## 7. Deliberately Not Covered (and where to look instead)

Depth over breadth. These are genuinely important and genuinely out of scope for a CLI-focused set; half-covering them would be worse than pointing at the real docs.

| Not covered | Why | Where to look |
|---|---|---|
| **Claude Agent SDK** | A different product surface — building your own agents in TypeScript or Python. It deserves its own doc set, not a thin chapter here. | [Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview), [TypeScript reference](https://code.claude.com/docs/en/agent-sdk/typescript), [Python reference](https://code.claude.com/docs/en/agent-sdk/python) |
| **The Claude API itself** | Messages API, tool use, prompt caching, model selection at the API level. Adjacent, but not something you touch to drive the CLI. | [platform.claude.com/docs](https://platform.claude.com/docs) |
| **Enterprise deployment** | Bedrock, Google Cloud's Agent Platform, Microsoft Foundry, LLM gateways, self-hosted environments, network config, corporate launchers. Relevant to a platform team, not to learning the tool. | [Enterprise deployment overview](https://code.claude.com/docs/en/third-party-integrations), [Gateways](https://code.claude.com/docs/en/gateways) |
| **Managed settings & org policy** | Touched only where it explains why a setting you wrote is being ignored. | [Managed settings](https://code.claude.com/docs/en/managed-settings), [Admin setup](https://code.claude.com/docs/en/admin-setup) |
| **Non-terminal surfaces** | Desktop app, Claude Code on the web, Slack, mobile, JetBrains. Same engine, same config, different UI. VS Code and `--teleport` get a paragraph in Part 1 because they interoperate with terminal sessions. | [Platforms and integrations](https://code.claude.com/docs/en/platforms) |
| **Security scanning products** | `/security-review`, Claude Security codebase scanning, and GitHub Code Review are products built *on* Claude Code, not parts of the CLI. | [Code Review](https://code.claude.com/docs/en/code-review), [Claude Security](https://code.claude.com/docs/en/claude-security) |
| **Compliance and data handling** | ZDR, data usage, retention policy. | [Data usage](https://code.claude.com/docs/en/data-usage), [Zero data retention](https://code.claude.com/docs/en/zero-data-retention) |
| **Exact pricing** | Deliberately never hardcoded in this repo — it changes too fast to be trustworthy in a reference doc. | [claude.com/pricing](https://claude.com/pricing), and `/usage` in a session |

[↑ Back to top](#table-of-contents)

---

## 8. Suggested Learning Order

**If you have never run `claude`** — read in this order, and actually run things between sections:

1. This document, [§3](#3-install-authenticate-first-session) and [§4](#4-primer-what-an-agentic-coding-tool-actually-is). Install, run `/init`, ask it a question about your codebase.
2. [Daily Driver](./claude-code-daily-driver.md), Parts 1–3. This is the highest-value doc in the set. Stop at the Advanced tier on your first pass.
3. [Permissions & Security](./claude-code-permissions-security.md), Part 8 Beginner + Working Knowledge only. You need the modes and `Shift+Tab` before you do anything unattended.
4. [Context & Memory](./claude-code-context-memory.md), Parts 4–5. Write a real `CLAUDE.md`. This is where output quality jumps.
5. [Config & Settings](./claude-code-config-settings.md), Part 6. Learn the precedence chain once, save yourself hours later.
6. Come back to [Permissions & Security](./claude-code-permissions-security.md) in full, including Part 9's sandbox material.
7. [Extensibility](./claude-code-extensibility.md), Parts 10–12, in that order. Skills first — most people reach for hooks when a skill would do.
8. [Automation & Agents](./claude-code-automation-agents.md), Parts 13–14, once you have a workflow worth automating.

**If you already use Claude Code daily and want the depth:** read [§6](#6-what-recent-releases-changed-under-you) of this document first — it is the highest information-per-line section in the set — then Part 8 (permission rule syntax), Part 12 (hooks), and Part 4 (compaction), in that order. Those three are where confident users most often hold a wrong model.

**If you are setting Claude Code up for a team:** [§6](#6-what-recent-releases-changed-under-you), then Part 6 (settings precedence), Part 8 (rules you can check in), Part 5 (`CLAUDE.md` conventions), Part 12 (hooks as enforcement), Part 14 (plugins as the distribution mechanism).

[↑ Back to top](#table-of-contents)

---

## 9. Quick Self-Check

If you can answer these without looking, the set has done its job. Each is answered in the Part named after it.

1. Name the three phases of the agentic loop, and say which one most people forget to give Claude a way to complete. *(§4, Part 3)*
2. Your `CLAUDE.md` rule stopped being followed 50 messages into a session. Give two distinct reasons this happens and the fix for each. *(Part 4, Part 5)*
3. What does `/clear` do that `/compact` doesn't, and when is **Summarize up to here** the better choice than either? *(Part 4)*
4. A settings key you wrote in `.claude/settings.json` is being ignored. List the layers that could be overriding it, in order. *(Part 6)*
5. Explain the difference between `Bash(ls *)` and `Bash(ls*)`, and why it matters. *(Part 8)*
6. Which permission mode does a `claude -p` run start in, and why is that different from an interactive session on the same machine? *(Part 8, Part 13)*
7. Claude edited a file you told it never to touch, and your `CLAUDE.md` says not to. What mechanism should you have used instead, and why is it categorically different? *(Part 5, Part 12)*
8. You have a 400-line `CLAUDE.md`. Name three mechanisms for cutting it down, and say which one actually reduces context cost. *(Part 5)*
9. When does a subagent's work get restored by `/rewind`, and when doesn't it? *(Part 2, Part 11)*
10. What's the difference between a skill with `context: fork` and a custom subagent? When would you reach for each? *(Part 10, Part 11)*
11. Your `PostToolUse` hook runs a linter. Where does its output go, and what does that cost you? *(Part 12)*
12. Name three things that survive compaction and two that don't. *(Part 4)*
13. Why is `--bare` recommended for CI, and what specifically does it stop from loading? *(Part 13)*
14. What does auto mode's classifier block that no allow rule can override? *(Part 8, Part 9)*
15. You want the same skill, hook, and MCP server in six repositories. What's the mechanism? *(Part 14)*

[↑ Back to top](#table-of-contents)

---

## 10. Cross-Doc Cheat Sheet

The commands and files you will reach for most, gathered in one place. Each Part in the sub-docs ends with its own deeper table.

### Launch

```bash
claude                              # interactive session in the current directory
claude "explain this project"       # interactive, with a first prompt
claude -p "list the API endpoints"  # non-interactive, print and exit
claude -c                           # continue the most recent session here
claude --resume auth-refactor       # resume by name or session ID
claude --permission-mode plan       # start in plan mode
claude --model opus                 # pick the model
claude --add-dir ../shared ../lib   # extra readable/editable directories
claude --worktree feature-auth      # isolated git worktree session
claude --bare -p "..."              # skip auto-discovery: CI's friend
```

### The commands you'll actually type

| Command | Does |
|---|---|
| `/init` | Generate or improve the project's `CLAUDE.md` |
| `/doctor` | Setup checkup; finds config, install, and skill-cost problems |
| `/context` | What's consuming context right now, as a grid |
| `/clear` · `/compact` | Reset context · summarise to free space |
| `/rewind` (or `Esc` `Esc`) | Restore code and/or conversation to a checkpoint |
| `/resume` · `/branch` | Switch conversation · fork it in place |
| `/memory` | Edit `CLAUDE.md` files; toggle and browse auto memory |
| `/permissions` | View and edit allow/ask/deny rules and auto mode rules |
| `/sandbox` | Configure the Bash sandbox |
| `/hooks` · `/mcp` · `/plugin` | Inspect hooks · manage MCP servers · manage plugins |
| `/model` · `/effort` · `/fast` | Model · reasoning effort · fast mode |
| `/plan` | Enter plan mode, optionally with a task |
| `/diff` | Review the working tree, including Claude's edits |
| `/usage` (`/cost`) | Spend and cache metrics for the session |
| `/export` | Write the conversation to a file or clipboard |

### The keys

| Key | Effect |
|---|---|
| `Esc` | Interrupt Claude / close a dialog |
| `Esc` `Esc` | Rewind menu (on empty input) — or clear the draft if there's text |
| `Shift+Tab` | Cycle permission modes |
| `Ctrl+O` | Toggle the transcript viewer |
| `Ctrl+G` | Open the current input in `$EDITOR` |
| `Ctrl+B` | Background the running Bash task (twice under tmux) |
| `Ctrl+R` | Reverse-search prompt history |
| `Ctrl+T` | Toggle the task checklist |
| `!` at line start | Shell mode — run a command directly into context |
| `@` | File-path autocomplete (and other live sessions, where enabled) |
| `?` on empty input | Shortcut help panel |

### The files

```text
~/.claude/
├── settings.json                 user settings (all projects)
├── CLAUDE.md                     personal instructions (all projects)
├── rules/*.md                    personal rules
├── skills/<name>/SKILL.md        personal skills
├── agents/<name>.md              personal subagents
└── projects/<project>/
    ├── <session-id>.jsonl        session transcripts
    └── memory/MEMORY.md          auto memory index + topic files

<repo>/
├── CLAUDE.md                     project instructions (checked in)
├── CLAUDE.local.md               personal project notes (gitignore this)
├── .mcp.json                     project MCP servers (checked in)
├── .worktreeinclude              gitignored files to copy into worktrees
└── .claude/
    ├── settings.json             shared project settings (checked in)
    ├── settings.local.json       your machine's project settings (gitignored)
    ├── CLAUDE.md                 alternative location for project instructions
    ├── rules/*.md                project rules, optionally path-scoped
    ├── skills/<name>/SKILL.md    project skills
    ├── agents/<name>.md          project subagents
    └── worktrees/                where --worktree puts checkouts
```

### The precedence chains, at a glance

```text
Settings          managed ▸ --settings ▸ .claude/settings.local.json
                          ▸ .claude/settings.json ▸ ~/.claude/settings.json

Skills            managed ▸ user ▸ project ▸ plugin

Subagents         managed ▸ --agents ▸ project ▸ user ▸ plugin

MCP servers       local ▸ project (.mcp.json) ▸ user ▸ plugin ▸ claude.ai connectors

CLAUDE.md         additive, not overriding — root-down, CLAUDE.local.md last

Hooks             all of them merge; every matching hook fires
```

[↑ Back to top](#table-of-contents)
