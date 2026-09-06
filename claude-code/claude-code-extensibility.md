# Claude Code — Extensibility (Parts 10–12)

The four mechanisms that change what Claude knows, who does the work, what happens automatically, and what it can reach: **skills**, **subagents**, **hooks**, and **MCP**.

> **Spec:** this doc follows the canonical spec in [`claude-code-mastery-guide.md`](./claude-code-mastery-guide.md#about-this-document). Written against **Claude Code v2.1.263**, verified **September 6, 2026**.
>
> **Read [§10.1](#101-beginner-choosing-the-right-mechanism) before anything else in this doc.** Picking the wrong mechanism is the most common and most expensive mistake in this area — people reach for hooks when a skill would do, and for MCP when a CLI tool would do.

---

## Table of Contents

- [Part 10 — Skills](#part-10--skills)
  - [10.1 Beginner: choosing the right mechanism](#101-beginner-choosing-the-right-mechanism)
  - [10.2 Beginner: your first skill](#102-beginner-your-first-skill)
  - [10.3 Working Knowledge: where skills live, and arguments](#103-working-knowledge-where-skills-live-and-arguments)
  - [10.4 Working Knowledge: the frontmatter reference](#104-working-knowledge-the-frontmatter-reference)
  - [10.5 Advanced: dynamic context injection](#105-advanced-dynamic-context-injection)
  - [10.6 Advanced: `context: fork`](#106-advanced-context-fork)
  - [10.7 Mastery: visibility, lifecycle, and permissioning](#107-mastery-visibility-lifecycle-and-permissioning)
  - [10.8 Part 10 cheat sheet](#108-part-10-cheat-sheet)
- [Part 11 — Subagents](#part-11--subagents)
  - [11.1 Beginner: what a subagent is for](#111-beginner-what-a-subagent-is-for)
  - [11.2 Working Knowledge: defining one](#112-working-knowledge-defining-one)
  - [11.3 Working Knowledge: invoking, and the built-ins](#113-working-knowledge-invoking-and-the-built-ins)
  - [11.4 Advanced: what loads at startup, and what doesn't](#114-advanced-what-loads-at-startup-and-what-doesnt)
  - [11.5 Advanced: foreground, background, and forks](#115-advanced-foreground-background-and-forks)
  - [11.6 Mastery: memory, isolation, concurrency, and depth](#116-mastery-memory-isolation-concurrency-and-depth)
  - [11.7 Part 11 cheat sheet](#117-part-11-cheat-sheet)
- [Part 12 — Hooks and MCP](#part-12--hooks-and-mcp)
  - [12.1 Beginner: a hook is a guarantee](#121-beginner-a-hook-is-a-guarantee)
  - [12.2 Working Knowledge: the event catalogue](#122-working-knowledge-the-event-catalogue)
  - [12.3 Working Knowledge: matchers and the five hook types](#123-working-knowledge-matchers-and-the-five-hook-types)
  - [12.4 Advanced: the input/output contract](#124-advanced-the-inputoutput-contract)
  - [12.5 Advanced: exit codes and blocking](#125-advanced-exit-codes-and-blocking)
  - [12.6 Working Knowledge: MCP](#126-working-knowledge-mcp)
  - [12.7 Advanced: MCP scopes, naming, and permissioning](#127-advanced-mcp-scopes-naming-and-permissioning)
  - [12.8 Part 12 cheat sheet](#128-part-12-cheat-sheet)

---

## Part 10 — Skills

<a id="part-10"></a>

### 10.1 Beginner: choosing the right mechanism

Before writing anything, work out which mechanism the problem calls for ([features overview](https://code.claude.com/docs/en/features-overview)):

| Feature | What it does | Use when | Example |
|---|---|---|---|
| **`CLAUDE.md`** | Persistent context, every conversation | "Always do X" rules | "Use pnpm, not npm" |
| **Skill** | Instructions, knowledge, workflows | Reusable content, reference docs, repeatable tasks | `/deploy` runs your checklist |
| **Subagent** | Isolated execution returning a summary | Context isolation, parallel work, specialists | Research that reads many files, returns findings |
| **Dynamic workflow** | Script Claude writes that runs many subagents | Work that outgrows a handful of subagents | Codebase-wide audit with cross-checking |
| **MCP** | Connection to external services | External data or actions | Query your DB, post to Slack |
| **Hook** | Script/HTTP/MCP/prompt/subagent on a lifecycle event | Automation that **must** run on every matching event | ESLint after every edit |
| **Plugin** | Packaging for all of the above | Reuse across repos, distribution | Your team's standard setup |

**The trigger-based version**, which is more useful in practice:

| When this happens | Add |
|---|---|
| Claude gets a convention wrong twice | A line in `CLAUDE.md` |
| You keep typing the same prompt to start a task | A user-invocable **skill** |
| You paste the same playbook into chat a third time | A **skill** |
| You keep copying data from a browser tab Claude can't see | An **MCP server** |
| Claude reads many files to find where a symbol is defined | A **code intelligence plugin** |
| A side task floods your conversation with output | A **subagent** |
| You want something to happen every time without asking | A **hook** |
| A second repository needs the same setup | A **plugin** |

**The two distinctions people get wrong:**

**Hook vs. skill.** *"An instruction like 'never edit `.env`' in `CLAUDE.md` or a skill is a request, not a guarantee. A `PreToolUse` hook that blocks the edit is enforcement."* Use a hook when the action must happen the same way every time and doesn't need Claude to think. Use a skill when Claude should decide *how* to apply the steps.

**Skill vs. subagent.** A skill is **reusable content you load into a context**. A subagent is **an isolated worker that runs separately**. A skill adds to your main window; a subagent uses its own. They combine: a subagent can preload skills (`skills:`), and a skill can run in isolation (`context: fork`).

### 10.2 Beginner: your first skill

A skill is a directory containing `SKILL.md`:

```markdown
<!-- .claude/skills/api-conventions/SKILL.md -->
---
name: api-conventions
description: REST API design conventions for our services
---

# API Conventions

- Use kebab-case for URL paths
- Use camelCase for JSON properties
- Always include pagination for list endpoints
- Version APIs in the URL path (/v1/, /v2/)
```

That's a **reference** skill: Claude loads it when it judges the description relevant.

An **action** skill you trigger yourself:

```markdown
<!-- .claude/skills/fix-issue/SKILL.md -->
---
name: fix-issue
description: Fix a GitHub issue
disable-model-invocation: true
---

Analyze and fix the GitHub issue: $ARGUMENTS.

1. Use `gh issue view` to get the issue details
2. Understand the problem described in the issue
3. Search the codebase for relevant files
4. Implement the necessary changes to fix the issue
5. Write and run tests to verify the fix
6. Ensure code passes linting and type checking
7. Create a descriptive commit message
8. Push and create a PR
```

Run it with `/fix-issue 1234`.

`disable-model-invocation: true` is the flag to reach for on anything with side effects: only you can trigger it, and it costs **zero context** until you do.

### 10.3 Working Knowledge: where skills live, and arguments

Four locations, in precedence order ([skills](https://code.claude.com/docs/en/skills)):

| Location | Path | Scope | Precedence |
|---|---|---|---|
| Enterprise | Managed settings directory | Org-wide | 1 (highest) |
| Personal | `~/.claude/skills/<name>/SKILL.md` | All your projects | 2 |
| Project | `.claude/skills/<name>/SKILL.md` | This project | 3 |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | Where the plugin is enabled | 4 |

Plugin skills are **namespaced** (`/plugin-name:skill-name`) so they never collide. Project skills also load from `.claude/skills/` in parent directories up to the repo root, and subdirectory-specific skills load when Claude first reads or edits a file in that directory.

| Skill location | Command |
|---|---|
| `~/.claude/skills/deploy/SKILL.md` | `/deploy` |
| `.claude/skills/deploy/SKILL.md` | `/deploy` |
| `.claude/commands/deploy.md` | `/deploy` |
| `my-plugin/skills/review/SKILL.md` | `/my-plugin:review` |
| `apps/web/.claude/skills/deploy/SKILL.md` | `/apps/web:deploy` (when the name clashes) |

> **`.claude/commands/` vs. skills.** Commands still work and are functionally equivalent for the simple case, but **skills are recommended**. When both exist with the same name, **the skill wins**. Skills add: supporting files in the skill directory, frontmatter control, auto-invocation from the description, and dynamic context injection. Commands are simpler if you need none of that.

**Arguments.** Three substitution forms:

```markdown
---
arguments: [issue, branch]
---

Fix GitHub issue $0 on branch $1
Or by name: Fix GitHub issue $issue on branch $branch
Or everything: $ARGUMENTS
```

```bash
/migrate-component SearchBar JavaScript TypeScript
```

**Special variables** available in skill content:

| Variable | Is |
|---|---|
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `${CLAUDE_SKILL_DIR}` | This skill's directory |
| `${CLAUDE_PROJECT_DIR}` | Project root |
| `${CLAUDE_PLUGIN_ROOT}` | Plugin directory (plugin skills only) |
| `${CLAUDE_PLUGIN_DATA}` | Plugin persistent data directory |
| `${CLAUDE_EFFORT}` | Current effort level |

**Stacking.** `/write-tests /fix-issue 123` loads up to **6 skills** in one message; trailing text passes to each as `$ARGUMENTS`.

**Structure.** Keep `SKILL.md` under ~500 lines and put detail in linked files:

```text
~/.claude/skills/my-skill/
├── SKILL.md          # required
├── reference.md      # linked from SKILL.md
├── examples.md
└── scripts/
    └── helper.sh     # runs via dynamic injection
```

### 10.4 Working Knowledge: the frontmatter reference

All fields are optional; only `description` is really recommended.

| Field | Purpose | Example |
|---|---|---|
| `name` | Display name in listings (doesn't change the command) | `name: deploy-prod` |
| `description` | **When Claude should invoke it** — keep it short, it's in context every request | `Deploy to production safely` |
| `disable-model-invocation` | Only you can invoke; Claude never sees it | `true` |
| `user-invocable` | Only Claude can invoke; hidden from the `/` menu | `false` |
| `allowed-tools` | Pre-approve tools **for this turn only** | `Bash(git *) Read` |
| `disallowed-tools` | Remove tools from Claude's pool while active | `AskUserQuestion` |
| `context` | `fork` to run in an isolated subagent | `context: fork` |
| `agent` | Subagent type when forking | `Explore`, `Plan`, `general-purpose` |
| `background` | With fork: run in background (default `true`) | `background: false` |
| `model` | Override the session model for this skill | `claude-sonnet-5` |
| `effort` | Override effort level | `high` |
| `argument-hint` | Autocomplete hint | `[issue-number]` |
| `arguments` | Named positional args for `$name` substitution | `[issue, branch]` |
| `paths` | Only auto-invoke when working with matching files | `src/**/*.ts,tests/**/*` |
| `shell` | Shell for injected commands | `bash` or `powershell` |
| `hooks` | Lifecycle hooks scoped to this skill | see [§12.3](#123-working-knowledge-matchers-and-the-five-hook-types) |
| `metadata`, `license`, `compatibility` | Bookkeeping | `compatibility: claude-code >= 2.1.200` |

**The `description` field is the whole game for auto-invocation.** Claude matches your task against skill descriptions to decide what's relevant. Vague or overlapping descriptions mean the wrong skill loads, or none does.

**`allowed-tools` is a turn-scoped grant, not a restriction.** It pre-approves tools so the skill's steps don't stop for permission — and the grants **expire at the end of the turn** while the skill content persists:

```markdown
---
name: commit
description: Stage and commit changes
disable-model-invocation: true
allowed-tools: Bash(git add *) Bash(git commit *) Bash(git status *)
---

Commit the current changes:
1. Stage files with git add
2. Commit with a clear message
```

<a id="p10-injection"></a>

### 10.5 Advanced: dynamic context injection

A skill can run shell commands **before Claude sees it**, and the output replaces the placeholder. This is what makes a skill a live template rather than a static document.

````markdown
---
name: pr-summary
description: Summarize a pull request
---

## PR Diff
!`gh pr diff`

## PR Comments
!`gh pr view --comments`

## Your task
Summarize the above changes in 2-3 bullets and flag risks.
````

Multi-line form:

````markdown
```!
git log --oneline -10
git status
```
````

**Three rules that bite:**

1. **`!` must start the line or follow whitespace.** `KEY=!`cmd`` will not run.
2. **A non-zero exit aborts the entire skill invocation.** Append `|| true` to anything expected to fail.
3. **The output is in your context**, so a chatty command is a context cost you pay every invocation.

> **Wrong vs. right — a skill that stops working in a clean repo.**
>
> ````markdown
> <!-- Wrong: fails outright when there are no changes to stash -->
> ## Current state
> !`git stash list`
> !`git diff --exit-code`
> ````
>
> `git diff --exit-code` returns 1 when there *are* differences — and here it aborts the whole skill.
>
> ````markdown
> <!-- Right -->
> ## Current state
> !`git stash list || true`
> !`git diff --stat || true`
> ````

`disableSkillShellExecution` in settings turns injection off entirely — worth knowing exists, because it's the setting an org uses to block this.

<a id="p10-fork"></a>

### 10.6 Advanced: `context: fork`

Adding `context: fork` runs the skill in an **isolated subagent**, with the skill content as its task:

```markdown
---
name: deep-research
description: Research a topic thoroughly
context: fork
agent: Explore
background: false
---

Research $ARGUMENTS thoroughly:

1. Find relevant files using Glob and Grep
2. Analyze the code
3. Summarize findings with specific file references
```

Behaviour:

- The subagent has **no conversation history** — isolated context.
- It runs in the **background by default**; `background: false` waits for the result.
- `agent` picks the type: `Explore` (read-only), `Plan` (planning/analysis), or `general-purpose` (default, full capabilities).

**Claude Code waits even with `background: true`** in four cases: non-interactive `-p` mode, `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`, the same skill invoked twice before the first finishes, and a scheduled task using the skill as its prompt.

> **The checkpoint consequence, which surprises people.** A **foreground** forked skill (`background: false`) edits your working tree during your own turn, so `/rewind` restores its edits as normal. A **background** fork — the default — is like any other subagent: **`/rewind` does not restore its edits.** Use git. (See [Part 2](./claude-code-daily-driver.md).)

<a id="p10-mastery"></a>

### 10.7 Mastery: visibility, lifecycle, and permissioning

**The three visibility states**, set by two frontmatter flags:

| Frontmatter | You invoke | Claude invokes | Listed to Claude |
|---|---|---|---|
| (default) | ✓ | ✓ | ✓ |
| `disable-model-invocation: true` | ✓ | ✗ | ✗ |
| `user-invocable: false` | ✗ | ✓ | ✓ |

**`skillOverrides`** does the same for skills you didn't write, without editing their files:

```json
{
  "skillOverrides": {
    "legacy-context": "name-only",
    "deploy": "off",
    "internal-docs": "user-invocable-only"
  }
}
```

In-session: `/skills` → highlight → `Space` cycles the states → `Esc` saves.

**Content lifecycle:**

- **Single invocation**: skill content enters context as one message and **stays across turns**.
- **Re-invocation**: if content hasn't changed (same args, same dynamic output), Claude Code notes it's already loaded rather than re-sending.
- **After compaction**: skills are re-injected, capped at **5,000 tokens each and 25,000 total**, oldest dropped first. Truncation keeps the **start** of the file — so put the most important instructions near the top.
- **Tool grants clear** at the end of the turn; the content persists.

**Permission rules for skills:**

```text
Skill(commit)        # exact match
Skill(review-pr *)   # prefix with args
Skill(deploy *)      # as a deny rule
Skill                # deny all skills
```

**Budget settings** for when the skill listing itself gets expensive: `skillListingBudgetFraction` (reserve more or less context for the listing) and `skillListingMaxDescChars` (cap each description). `/skill-doctor` (v2.1.261+) reports which loaded skills go unused. `/doctor` also finds unused skills against their context cost.

**Synced skills from claude.ai.** In Cowork and cloud sessions, Claude loads skills you enabled on claude.ai. Locally, opt in:

```bash
CLAUDE_CODE_SYNC_SKILLS=1 claude -p "List available skills"
```

They download to `~/.claude/skills/synced/` and persist. **When a synced skill's name matches a local skill or built-in command, the local one wins.**

**Troubleshooting:**

| Symptom | Check |
|---|---|
| Skill never triggers | Does `description` contain the words you'd actually say? Ask "What skills are available?". Invoke directly with `/name` to confirm it loads at all |
| Skill triggers too often | Make the description more specific, or add `disable-model-invocation: true` |
| Description truncated in listings | `/skill-doctor`; set low-priority skills to `"name-only"`; raise `skillListingBudgetFraction` |
| YAML not parsing | `claude plugin validate .claude/skills` |

> ### Real Scenario — the skill that shipped a stale diff
>
> A team's `/pr-summary` skill injects `!`gh pr diff`` and asks Claude to summarise. It works well for weeks. Then a reviewer complains that a summary described changes that weren't in the PR.
>
> **What happened:** the developer ran `/pr-summary` early in a session, read the summary, pushed three more commits, and ran `/pr-summary` again to regenerate. Claude Code saw the same skill with the same arguments and — because the *skill content* had not changed — noted it was already loaded rather than re-sending it. But the dynamic injection had already been evaluated once, at the first invocation. The second summary was written against the first diff.
>
> Except that isn't quite the rule: re-invocation skips re-sending only when the dynamic output is *also* unchanged. What actually caught them was subtler — the skill also injected `!`gh pr view --comments``, which errored transiently on the second run against a rate limit. A non-zero exit **aborts the whole skill invocation**, so nothing new was injected at all, and Claude answered from the copy still sitting in context.
>
> **The fixes:**
> 1. `|| true` on every injected command that can fail for reasons unrelated to the task.
> 2. Have the skill state its own freshness: `Diff generated at !`date -u``.` A visible timestamp makes a stale answer self-evident.
> 3. For anything where staleness is a correctness problem rather than an annoyance, use `context: fork` with `background: false` — the forked subagent starts with no conversation history, so there's no stale copy to answer from.

<a id="108-part-10-cheat-sheet"></a>

### 10.8 Part 10 cheat sheet

| Frontmatter | Effect |
|---|---|
| `description` | What Claude matches against — the most important field |
| `disable-model-invocation: true` | Manual-only, **zero context until invoked** |
| `user-invocable: false` | Claude-only, hidden from `/` |
| `allowed-tools` / `disallowed-tools` | Turn-scoped grant / removal |
| `context: fork` + `agent` + `background` | Run isolated; `background: false` waits and is rewindable |
| `paths` | Only auto-invoke for matching files |
| `arguments` / `argument-hint` | `$name` substitution / autocomplete hint |
| `model`, `effort` | Per-skill overrides |

| Syntax | Does |
|---|---|
| `$ARGUMENTS`, `$0`, `$name` | Argument substitution |
| ``!`cmd` `` | Inject command output (non-zero exit **aborts**) |
| ` ```! ` block | Multi-line injection |
| `${CLAUDE_SKILL_DIR}` etc. | Special variables |
| `/a /b args` | Stack up to 6 skills |

| Precedence | managed ▸ user ▸ project ▸ plugin (namespaced) |
|---|---|
| Skill vs. `.claude/commands/` same name | **Skill wins** |
| Synced vs. local same name | **Local wins** |

<a id="part-11"></a>

[↑ Back to top](#table-of-contents)

---

## Part 11 — Subagents

### 11.1 Beginner: what a subagent is for

A subagent runs the agentic loop in **its own context window** and returns a summary. That one property is why they exist ([sub-agents](https://code.claude.com/docs/en/sub-agents)):

```text
   YOUR CONTEXT                          SUBAGENT'S CONTEXT
   ────────────                          ──────────────────
   ...conversation...
   "investigate token refresh"  ─────▶   own system prompt
                                         CLAUDE.md + git status
                                         reads session.ts     (2,400 tok)
                                         reads timeouts.ts    (1,100 tok)
                                         reads config/*.ts    (3,800 tok)
                                         greps, analyses
   summary (420 tok)            ◀─────   returns findings
   ...continue working...
                                         ← 7,300 tokens never
                                           touched your window
```

The simplest way to use one:

```text
Use subagents to investigate how our authentication system handles token
refresh, and whether we have any existing OAuth utilities I should reuse.
```

### 11.2 Working Knowledge: defining one

A Markdown file with YAML frontmatter, then the system prompt:

```markdown
<!-- .claude/agents/security-reviewer.md -->
---
name: security-reviewer
description: Reviews code for security vulnerabilities
tools: Read, Grep, Glob, Bash
model: opus
---

You are a senior security engineer. Review code for:
- Injection vulnerabilities (SQL, XSS, command injection)
- Authentication and authorization flaws
- Secrets or credentials in code
- Insecure data handling

Provide specific line references and suggested fixes.
```

Only `name` and `description` are required. The full field list:

| Field | Purpose |
|---|---|
| `name` | Unique identifier (lowercase letters and hyphens) |
| `description` | **When Claude should delegate to this agent** |
| `tools` | Allowed tools (inherits all if omitted) |
| `disallowedTools` | Tools to deny |
| `model` | `sonnet`, `opus`, `haiku`, `fable`, a full ID, or `inherit` |
| `permissionMode` | `default`, `acceptEdits`, `auto`, `dontAsk`, `bypassPermissions`, `plan` |
| `maxTurns` | Cap agentic turns; a partial result is returned |
| `skills` | Skills to **preload** into its context |
| `mcpServers` | MCP servers available to it |
| `hooks` | Lifecycle hooks for this agent |
| `memory` | Persistent memory scope: `user`, `project`, or `local` |
| `background` | `true` to keep it in the background even when Claude asks for foreground |
| `effort` | `low` … `max` |
| `isolation` | `worktree` for its own git worktree |
| `color` | Display colour in listings |
| `initialPrompt` | Auto-submitted first turn when the agent runs as the main session |
| `experimental` | e.g. `cacheTtl: 5m` or `1h` |

**Locations and precedence:**

| Location | Scope | Priority |
|---|---|---|
| Managed settings | Org-wide | 1 (highest) |
| `--agents` CLI flag | This session | 2 |
| `.claude/agents/` | This project | 3 |
| `~/.claude/agents/` | All your projects | 4 |
| Plugin `agents/` | Where the plugin is enabled | 5 |

> Note the difference from skills: **project beats user** for subagents, and the reverse for skills.

Defining one for a single session:

```bash
claude --agents '{
  "code-reviewer": {
    "description": "Expert code reviewer",
    "prompt": "You are a senior code reviewer...",
    "tools": ["Read", "Grep", "Glob", "Bash"],
    "model": "sonnet"
  }
}'
```

**Model resolution**, first match wins: per-invocation `model` parameter → the definition's `model` frontmatter (`inherit` = the main conversation's) → `CLAUDE_CODE_SUBAGENT_MODEL` → the main conversation's model. To force every subagent onto a cheap model:

```json
{
  "env": {
    "CLAUDE_CODE_SUBAGENT_MODEL": "haiku",
    "CLAUDE_CODE_SUBAGENT_MODEL_FORCE": "1"
  }
}
```

> As of v2.1.198, `/agents` no longer opens a creation UI — it prints a reminder to ask Claude to create subagents, or to edit `.claude/agents/` directly. Asking Claude to write the file is genuinely the fastest path.

### 11.3 Working Knowledge: invoking, and the built-ins

| Method | Guarantees it runs? |
|---|---|
| `Use the code-improver agent to suggest improvements` | No — Claude decides |
| `@"code-reviewer (agent)" look at the auth changes` | **Yes** |
| `@agent-code-reviewer`, `@agent-my-plugin:code-reviewer` | Yes |
| `claude --agent code-reviewer` | Session-wide default |
| `{"agent": "code-reviewer"}` in settings | Session-wide default |

**Built-in subagents:**

| Agent | Purpose | Tools | Notes |
|---|---|---|---|
| **Explore** | Fast read-only codebase exploration | Read-only | **Skips `CLAUDE.md` and git status** |
| **Plan** | Research during plan mode | Read-only | Skips `CLAUDE.md` and git status |
| **general-purpose** | Complex multi-step tasks needing exploration *and* action | Every tool | Uses `CLAUDE_CODE_SUBAGENT_MODEL` if set |
| `claude` | Catch-all | Every tool | |
| `statusline-setup` | Powers `/statusline` | | Sonnet |
| `claude-code-guide` | Answers Claude Code feature questions | | Haiku |

Explore and Plan **don't support resuming** — they're one-shot. Every other subagent can be continued with `SendMessage`, retaining full history and picking up where it stopped, available when it completes or hits `maxTurns`:

```text
Continue that code review and now analyze the authorization logic
```

<a id="p11-startup"></a>

### 11.4 Advanced: what loads at startup, and what doesn't

A non-fork subagent receives:

- **Its own system prompt** — not Claude Code's — plus environment details
- The **task message** from the lead agent
- **`CLAUDE.md` files**, full hierarchy — *except* Explore and Plan, which skip them
- **Git status** — same exception
- **Full content of skills listed in `skills:`** (preloaded, not on-demand)
- A **sibling roster** of named agents it can reach via `SendMessage` (v2.1.206+)

**What does not transfer:**

- Conversation history
- Output style preferences
- **The main conversation's auto memory**
- Skills you already invoked in the main conversation

That list explains the most common subagent complaint — *"it doesn't know what we just discussed"* — and the fix: **put the context in the prompt.** A subagent knows only what you tell it plus what it can read.

Skills in subagents work differently from the main conversation: those in `skills:` are **fully preloaded at launch** rather than loaded on demand, though the subagent can still discover and invoke unlisted project, user, and plugin skills through the Skill tool.

> **Wrong vs. right — delegating research.**
>
> ```text
> # Wrong — the subagent has no idea what "the approach we settled on" means
> Use a subagent to check whether the approach we settled on will work
> with the legacy client.
> ```
>
> ```text
> # Right — self-contained
> Use a subagent to check whether replacing LegacyHttpClient with HttpClient2
> in src/billing/ is safe. Specifically: does anything in that directory rely
> on LegacyHttpClient's connection-pooling behaviour or its retry defaults?
> Report file:line for anything that does.
> ```

### 11.5 Advanced: foreground, background, and forks

**How Claude Code chooses**, in order:

1. Spawned by a teammate → foreground
2. `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` → foreground
3. Fork mode on → background
4. `background: true` in frontmatter → background
5. Otherwise Claude decides — foreground if it needs the result before continuing

| | Foreground | Background |
|---|---|---|
| Main conversation | Blocks until complete | Continues concurrently |
| Permission prompts | Passed through | Surface in the main session |
| `/rewind` restores its edits | Only for a forked *skill* | No |

**Background subagents get a reduced tool set**: `Read`, `Grep`, `Glob`, `Bash`, `PowerShell`, `Edit`, `Write`, `NotebookEdit`, `WebFetch`, `WebSearch`, `TodoWrite`, `Skill`, `ToolSearch`, `EnterWorktree`, `ExitWorktree`, `Monitor`, `TaskStop`, `SendMessage`, and MCP tools. Notably absent: `AskUserQuestion` — a background agent can't stop to ask you something.

**Forks** inherit the entire conversation instead of starting fresh:

```text
/subtask draft unit tests for the parser changes so far
```

A fork gets the same system prompt, tools, model, and **full message history**. Its own tool calls stay isolated; only the final result returns. Fork mode is **on by default in interactive sessions**; disable with `CLAUDE_CODE_FORK_MODE=off`.

**Permission modes in subagents.** Set `permissionMode` in frontmatter, with one hard rule: **if the parent uses `bypassPermissions` or `acceptEdits`, that takes precedence and the subagent cannot override it.** A subagent cannot be more restricted than a permissive parent — plan accordingly.

`dontAsk` is the useful one here: it auto-denies anything not explicitly allowed, so a subagent given a narrow `tools:` list plus `permissionMode: dontAsk` is genuinely constrained.

<a id="p11-mastery"></a>

### 11.6 Mastery: memory, isolation, concurrency, and depth

**Persistent subagent memory.** The `memory` field (`user`, `project`, or `local`) gives a subagent its own auto-memory directory — separate from the main conversation's, which it never sees. Use it for a long-running specialist that should accumulate knowledge: a reviewer that learns your team's recurring issues, a migration agent that remembers which packages it has already converted.

**Worktree isolation.** `isolation: worktree` gives the subagent its own git worktree, so parallel agents don't collide on files:

```markdown
---
name: refactorer
description: Applies mechanical refactors across many files
isolation: worktree
---

Apply the requested refactor across every affected file, then run the tests
and report the results.
```

Or ask Claude to "use worktrees for your agents." Each gets a temporary worktree, removed automatically when the subagent finishes **without changes**; one with changes stays on disk until the periodic sweep can remove it without losing work. Subagent worktrees use the same base branch as `--worktree` — your repo's default branch, unless `worktree.baseRef` is `"head"`.

While an agent runs, Claude Code holds a `git worktree lock` on its worktree so concurrent cleanup can't remove it. See [Part 14](./claude-code-automation-agents.md) for the full worktree story.

**Concurrency and depth:**

| Limit | Default | Override |
|---|---|---|
| Concurrent subagents spawning at once | **20** | `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` |
| Subagent nesting depth | **3 layers** | `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` |

Twenty concurrent agents is a lot of parallel work and a lot of simultaneous token spend. When a job genuinely needs that scale, the docs point at [dynamic workflows](https://code.claude.com/docs/en/workflows) instead — a script Claude writes that orchestrates many subagents and returns one result, optionally with a second set of agents verifying the first set's findings. [Part 14](./claude-code-automation-agents.md) covers them.

**The `Agent` permission rules** from [Part 8](./claude-code-permissions-security.md) apply here:

```json
{
  "permissions": {
    "deny": ["Agent(Explore)", "Agent(model:opus)", "Agent(isolation:worktree)"]
  }
}
```

<a id="117-part-11-cheat-sheet"></a>

### 11.7 Part 11 cheat sheet

| Frontmatter | Effect |
|---|---|
| `name`, `description` | Required; description drives delegation |
| `tools` / `disallowedTools` | Restrict the tool pool |
| `model` (incl. `inherit`) · `effort` | Capability and reasoning |
| `permissionMode` | **Can't be more restrictive than a `bypassPermissions`/`acceptEdits` parent** |
| `maxTurns` | Cap; returns a partial result |
| `skills` | Fully **preloaded** at launch |
| `memory: user\|project\|local` | Its own persistent memory |
| `isolation: worktree` | Its own git checkout |
| `background: true` | Force background |

| Loads at startup | Doesn't |
|---|---|
| Own system prompt + environment | Conversation history |
| Task message from the lead | Output style |
| `CLAUDE.md` + git status (**not** Explore/Plan) | Main conversation's auto memory |
| Preloaded `skills:` content | Skills already invoked in the main conversation |
| Sibling roster for `SendMessage` | |

| Invocation | |
|---|---|
| `Use the X agent to …` | Claude decides |
| `@agent-X` / `@"X (agent)"` | Guaranteed |
| `claude --agent X` · `{"agent": "X"}` | Session default |
| `/subtask <task>` | Fork: inherits the whole conversation |
| `SendMessage` to a finished agent | Resume it with history intact |

| Limit | Value |
|---|---|
| Concurrent subagents | 20 (`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`) |
| Spawn depth | 3 (`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`) |
| Precedence | managed ▸ `--agents` ▸ project ▸ user ▸ plugin |
| Rewind restores edits | Foreground forked skills only |

[↑ Back to top](#table-of-contents)

---

## Part 12 — Hooks and MCP

<a id="part-12"></a>

*Two mechanisms in one Part because they share a role: both connect Claude Code to things outside the model's own decision-making. Hooks get the deeper treatment because their contract is where people get stuck.*

### 12.1 Beginner: a hook is a guarantee

A hook is a shell command, HTTP request, MCP tool call, LLM prompt, or subagent that Claude Code runs **when it reaches a lifecycle event** ([hooks](https://code.claude.com/docs/en/hooks)). Unlike a `CLAUDE.md` instruction, the trigger is guaranteed.

Run a linter after every file edit:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "npm run lint:fix", "timeout": 30 }]
      }
    ]
  }
}
```

Hooks live in the same places settings do:

| Location | Scope | Shareable |
|---|---|---|
| `~/.claude/settings.json` | All projects | No |
| `.claude/settings.json` | One project | **Yes** — commit it |
| `.claude/settings.local.json` | One project, you | No |
| Managed policy settings | Org-wide | Yes |
| Plugin `hooks/hooks.json` | When enabled | Yes |
| Skill / subagent frontmatter | Session or agent scope | Yes |

**Hooks merge across all sources** — every matching hook fires, regardless of where it's defined. All matching hooks run **in parallel**, and the same handler defined in multiple files runs once.

`/hooks` shows what's configured. `disableAllHooks: true` is the kill switch (it also disables a custom status line and file-suggestion command).

> **You can ask Claude to write them.** *"Write a hook that runs eslint after every file edit"* or *"write a hook that blocks writes to the migrations folder"* works well, because the schema is well-specified.

### 12.2 Working Knowledge: the event catalogue

There are **31 hook events**. Grouping them by cadence makes them tractable:

```text
  ONCE PER SESSION          ONCE PER TURN            EVERY TOOL CALL
  ────────────────          ─────────────            ───────────────
  SessionStart              UserPromptSubmit         PreToolUse      ← can block
  SessionEnd                UserPromptExpansion      PostToolUse
  Setup                     Stop           ← blocks  PostToolUseFailure
                            StopFailure              PostToolBatch   ← can block
                                                     PermissionRequest
                                                     PermissionDenied

  EVERYTHING ELSE
  ───────────────
  Notification         MessageDisplay        SubagentStart / SubagentStop
  TaskCreated / TaskCompleted                TeammateIdle
  InstructionsLoaded   ConfigChange          CwdChanged / DirectoryAdded
  FileChanged          WorktreeCreate / WorktreeRemove
  PreCompact / PostCompact                   PreModelSwitch / PostModelSwitch
  Elicitation / ElicitationResult
```

The ones you'll actually reach for:

| Event | Fires | Typical use |
|---|---|---|
| `PreToolUse` | Before a tool executes | **Block** unsafe actions; validate inputs |
| `PostToolUse` | After a tool succeeds | Lint, format, typecheck after edits |
| `UserPromptSubmit` | Before Claude sees your prompt | Inject context; reject prompts |
| `Stop` | When Claude finishes responding | **Gate the turn** on a check passing |
| `SessionStart` | Session begins or resumes | Inject state; matcher `compact` re-injects after compaction |
| `SessionEnd` | Session ends | Archive the transcript |
| `Notification` | Claude Code notifies | Desktop notifications, Slack pings |
| `InstructionsLoaded` | `CLAUDE.md` or rules load | **Debug which instruction files loaded and why** |
| `PreCompact` / `PostCompact` | Around compaction | Preserve or restore state |
| `WorktreeCreate` / `WorktreeRemove` | Worktree lifecycle | Replace git logic for non-git VCS |

`Setup` is worth a note: it runs with `--init-only`, `--init`, or `--maintenance`, which makes it the hook for one-time environment preparation in CI.

### 12.3 Working Knowledge: matchers and the five hook types

**Matchers** filter when a hook fires, and how the string is interpreted depends on its characters:

| Pattern | Evaluated as | Example |
|---|---|---|
| `"*"`, `""`, omitted | Match all | Every event |
| Alphanumeric, `_`, `-`, spaces, `,`, `\|` | Exact string or list | `Bash`, `Edit\|Write` |
| Any other character | **Regex (unanchored)** | `^Notebook`, `mcp__memory__.*` |

That third row is a trap in both directions: `Edit|Write` is treated as a *list*, not a regex alternation — which happens to behave the same — but something like `Bash.` becomes a regex, and an unanchored one.

**What each event matches on:**

| Event | Matches | Values |
|---|---|---|
| `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`, `PermissionDenied` | Tool name | `Bash`, `Edit\|Write`, `mcp__.*` |
| `SessionStart` | Session type | `startup`, `resume`, `clear`, `compact`, `fork` |
| `Setup` | CLI flag | `init`, `maintenance` |
| `SessionEnd` | End reason | `clear`, `resume`, `logout`, `prompt_input_exit`, `other` |
| `Notification` | Type | `permission_prompt`, `auth_success`, `elicitation_dialog` |
| `SubagentStart` / `SubagentStop` | Agent type | `general-purpose`, `Explore`, custom names |
| `PreCompact` / `PostCompact` | Trigger | `manual`, `auto` |
| `PreModelSwitch` / `PostModelSwitch` | Model name | `claude-opus-5`, `.*opus.*` |
| `ConfigChange` | Source | `user_settings`, `project_settings`, `policy_settings` |
| `FileChanged` | Filenames | `.envrc\|.env` |
| `StopFailure` | Error type | `rate_limit`, `authentication_failed`, `server_error` |

MCP tools follow `mcp__<server>__<tool>`, so `"mcp__.*__write.*"` matches write operations across every server.

**The five hook types:**

```json
{ "type": "command", "command": "/path/to/script.sh", "args": [], "shell": "bash", "async": false }
{ "type": "http", "url": "http://localhost:8080/hook", "headers": {"Authorization": "Bearer $TOK"}, "allowedEnvVars": ["TOK"] }
{ "type": "mcp_tool", "server": "my_server", "tool": "security_scan", "input": {"file_path": "${tool_input.file_path}"} }
{ "type": "prompt", "prompt": "Should this be modified? $ARGUMENTS", "model": "claude-opus-5" }
{ "type": "agent", "prompt": "Verify this deployment: $ARGUMENTS" }
```

**Command hooks have two forms**, and the distinction matters for safety:

- **Exec form** (`args` is set): no shell interpretation, direct process spawn, placeholders substituted as strings.
  ```json
  { "command": "node", "args": ["${CLAUDE_PROJECT_DIR}/script.js", "--flag"] }
  ```
- **Shell form** (`args` omitted): full shell features — pipes, `&&`, globs, variable expansion.
  ```json
  { "command": "bash ./setup.sh && npm test" }
  ```

Use exec form whenever a path or value could contain something shell-significant.

**Common options:** `timeout` (seconds), `statusMessage` (custom display text), `once: true` (run once per session), `if` (a permission-rule pattern gating the hook, e.g. `"if": "Bash(rm *)"`), `async` / `asyncRewake` (background execution).

**Timeouts:**

| Hook type | Default |
|---|---|
| Command / HTTP / MCP | 600s — but 30s on `UserPromptSubmit`, `PreModelSwitch`, `PostModelSwitch`; 10s on `MessageDisplay` |
| Prompt | 30s |
| Agent | 60s |
| `SessionEnd` | 1.5s **shared budget** (raised to match a per-hook timeout, max 60) |
| Async | Not enforced |

**Path placeholders:** `${CLAUDE_PROJECT_DIR}`, `${CLAUDE_PLUGIN_ROOT}`, `${CLAUDE_PLUGIN_DATA}`. Note that `${CLAUDE_PROJECT_DIR}` **stays at the project root even after Claude enters a worktree** — read the `cwd` field from the hook's input JSON when you need the worktree path.

**Skill and subagent frontmatter can carry hooks** scoped to that skill or agent:

```yaml
---
name: secure-operations
description: Perform operations with security checks
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/security-check.sh"
---
```

<a id="p12-contract"></a>

### 12.4 Advanced: the input/output contract

**Every hook receives JSON on stdin.** Common fields:

```json
{
  "session_id": "abc123",
  "prompt_id": "550e8400-e29b-41d4-a716-446655440000",
  "transcript_path": "/home/user/.claude/projects/.../transcript.jsonl",
  "cwd": "/home/user/my-project",
  "permission_mode": "default",
  "effort": { "level": "high" },
  "hook_event_name": "PreToolUse",
  "agent_id": "subagent-123",
  "agent_type": "general-purpose"
}
```

A `PreToolUse` hook adds the tool details:

```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf /tmp/build",
    "description": "Clean build",
    "timeout": 120000,
    "run_in_background": false
  },
  "tool_use_id": "toolu_01ABC123..."
}
```

**Output schema** — print JSON to stdout:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|block",
    "permissionDecisionReason": "User-facing reason",
    "additionalContext": "Extra info for Claude",
    "updatedInput": { },
    "retry": true,
    "decision": "continue|stop"
  },
  "systemMessage": "Internal system message",
  "terminalSequence": "]0;Title"
}
```

**Decision fields per event:**

| Event | Fields |
|---|---|
| `PreToolUse` | `permissionDecision` (`allow`/`deny`/`block`), `permissionDecisionReason`, `additionalContext` |
| `PermissionRequest` | `decision` (`allow`/`deny`), `reason` |
| `UserPromptExpansion` | `updatedInput` — a modified prompt string |
| `Stop` | `decision` (`continue`/`stop`) |

A blocking `PreToolUse` hook, in full:

```bash
#!/bin/bash
COMMAND=$(jq -r '.tool_input.command')

if echo "$COMMAND" | grep -q 'rm -rf'; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: "Destructive command blocked"
    }
  }'
else
  exit 0
fi
```

Wired up with an `if` guard so it only runs on the commands it cares about:

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "if": "Bash(rm *)",
        "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh",
        "args": []
      }]
    }]
  }
}
```

**`terminalSequence`** lets a hook emit an escape sequence — useful for setting the terminal title or triggering a desktop notification when Claude needs you.

### 12.5 Advanced: exit codes and blocking

Exit codes carry meaning, and this is the part people get wrong:

| Exit code | Meaning |
|---|---|
| **0** | Success. JSON on stdout is parsed and honoured; plain text goes to the debug log (or into context, for certain events) |
| **2** | **Blocking error.** Blocks the action regardless of JSON, using `permissionDecisionReason` or stderr as the message |
| Anything else | Non-blocking. Valid JSON is still honoured for decision fields; invalid JSON or empty stdout is a non-blocking error, and the action proceeds with a notice |

**What exit 2 actually blocks, per event:**

| Event | Blocks? | Effect |
|---|---|---|
| `PreToolUse` | **Yes** | Blocks the tool call |
| `UserPromptSubmit` | **Yes** | Blocks the prompt and **erases your input** |
| `UserPromptExpansion` | Yes | Blocks the expansion |
| `Stop` | **Yes** | Prevents Claude from stopping |
| `SubagentStop` | Yes | Prevents the subagent stopping |
| `PostToolBatch` | Yes | Stops the agentic loop |
| `ConfigChange` | Yes | Blocks the config change |
| `WorktreeCreate` | Yes | Aborts worktree creation |
| `PostToolUse` | **No** | The tool already ran — stderr is shown to Claude |
| `PermissionRequest` | No | Ignored; use the `decision` field |
| `StopFailure` | No | Ignored entirely |

> **The `Stop` hook is the deterministic quality gate** from [Part 3](./claude-code-daily-driver.md#37-mastery-escalating-how-hard-the-stop-is-gated): it runs your check as a script and blocks the turn from ending until it passes. **Claude Code overrides it and ends the turn after 8 consecutive blocks**, so it can't loop forever.

**Debugging hooks:** run `claude --debug` (or `/debug` mid-session). Claude Code records which hooks matched, their exit codes, and their output in the debug log. `/doctor` flags slow hooks.

**Organisational controls:** `allowManagedHooksOnly` (run only org-deployed hooks), `allowedHttpHookUrls` (limit HTTP hook targets), `httpHookAllowedEnvVars` (limit which env vars can go into headers).

> ### Real Scenario — the hook that blocked every commit for a day
>
> A team adds a `PreToolUse` hook to enforce conventional commit messages, checked into `.claude/settings.json`:
>
> ```json
> {
>   "hooks": {
>     "PreToolUse": [{
>       "matcher": "Bash",
>       "hooks": [{ "type": "command",
>         "command": "jq -r '.tool_input.command' | grep -qE '^git commit' && ~/.claude/hooks/check-commit-msg.sh" }]
>     }]
>   }
> }
> ```
>
> From the next morning, Claude cannot run **any** Bash command. Every call is blocked with an unhelpful message. Nobody changes anything in the repo, so nobody suspects the hook.
>
> **What happened, in three compounding mistakes:**
>
> 1. **The matcher is `Bash`, so the hook runs on every Bash call.** The `grep` guard was meant to filter, but `grep -q` **exits 1 when it doesn't match** — and the `&&` chain then makes the whole command exit 1. Exit 1 is a *non-blocking* error, so this alone would have been noisy rather than fatal.
> 2. **`check-commit-msg.sh` exited 2 on any failure**, including "no commit message found to check" — which is what it hit on every non-commit command that *did* reach it. Exit 2 blocks.
> 3. **The hook path is `~/.claude/hooks/`, a user path, in a shared project file.** It worked for the author and silently did nothing for teammates who hadn't created the script — a *different* wrong behaviour for different people, which is why the reports were inconsistent.
>
> **The fixed version:**
>
> ```json
> {
>   "hooks": {
>     "PreToolUse": [{
>       "matcher": "Bash",
>       "hooks": [{
>         "type": "command",
>         "if": "Bash(git commit *)",
>         "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/check-commit-msg.sh",
>         "args": []
>       }]
>     }]
>   }
> }
> ```
>
> **The four lessons:** use **`if`** to gate a hook on a permission-rule pattern rather than hand-rolling a shell guard; use **`${CLAUDE_PROJECT_DIR}`** and check the script into the repo so it exists for everyone; reserve **exit 2** for "this specific thing is genuinely wrong", never for "I couldn't run"; and run `claude --debug` the moment a tool starts failing for no visible reason, because the debug log names the matched hook and its exit code immediately.

<a id="p12-mcp"></a>

### 12.6 Working Knowledge: MCP

The [Model Context Protocol](https://code.claude.com/docs/en/mcp) connects Claude to external services — databases, issue trackers, design tools, your own internal APIs.

```bash
# HTTP (recommended for remote servers)
claude mcp add --transport http notion https://mcp.notion.com/mcp

# stdio (local processes) — note the `--` separating Claude's options from the command
claude mcp add --transport stdio airtable --env AIRTABLE_API_KEY=KEY -- npx -y airtable-mcp-server

# SSE (deprecated)
claude mcp add --transport sse <name> <url>
```

**Management:**

```bash
claude mcp list                    # all servers with status
claude mcp get sentry              # details for one
claude mcp remove sentry
claude mcp login sentry            # run its OAuth flow
claude mcp logout sentry
claude mcp add-from-claude-desktop
claude mcp reset-project-choices
claude mcp serve                   # run Claude Code itself as an MCP server
```

In-session, `/mcp` opens the panel; `/mcp reconnect <server>` and `/mcp enable|disable <server|all>` act directly. In `-p` mode, `/mcp` with no argument prints a text status summary (v2.1.205+).

**Status indicators:**

```text
✔ Connected            ! Needs authentication      ✘ Failed to connect
⏸ Pending approval     ⊘ Disabled for this project  cached 2h ago
```

**Project servers** go in `.mcp.json` at the repo root, which you commit:

```json
{
  "mcpServers": {
    "shared-server": { "type": "http", "url": "https://example.com/mcp" },
    "database-tools": {
      "type": "stdio",
      "command": "python",
      "args": ["server.py"],
      "env": { "DB_URL": "${DB_URL:-postgresql://localhost}" }
    },
    "api-server": {
      "type": "http",
      "url": "${API_BASE_URL:-https://api.example.com}/mcp",
      "headers": { "Authorization": "Bearer ${API_KEY}" }
    }
  }
}
```

`${VAR}` expands from the environment; `${VAR:-default}` supplies a fallback. For rotating credentials — Kerberos, short-lived tokens, custom SSO — use `headersHelper`, a script that prints JSON headers:

```json
{ "mcpServers": { "internal-api": {
    "type": "http", "url": "https://mcp.internal.example.com",
    "headersHelper": "/opt/bin/get-auth-headers.sh" } } }
```

> **`headersHelper` scripts run only after you trust the containing folder** — the same workspace-trust boundary as hooks.

**Before reaching for MCP, consider a CLI tool.** The official best-practices guidance is explicit that CLI tools are *the most context-efficient way* to interact with external services, and Claude is good at learning ones it doesn't know from `--help`. MCP earns its place when you need structured tools, managed authentication, or access to something with no CLI.

<a id="p12-mcp-advanced"></a>

### 12.7 Advanced: MCP scopes, naming, and permissioning

**Scopes**, in precedence order:

```text
  1. Local   (~/.claude.json, per-project)   default; private to you
  2. Project (.mcp.json in the repo root)    shared, version-controlled
  3. User    (~/.claude.json, global)        private, all projects
  4. Plugin-provided servers
  5. claude.ai connectors
```

```bash
claude mcp add --transport http stripe https://mcp.stripe.com                       # local
claude mcp add --transport http shared --scope project https://example.com/mcp      # project
claude mcp add --transport http hubspot --scope user https://mcp.hubspot.com/...    # user
```

**Tool naming** is what permission rules and hook matchers key off:

```text
mcp__<server-name>__<tool-name>
mcp__github__list_issues
mcp__database__query

Plugin-bundled servers:
mcp__plugin_<plugin>_<server>__<tool>
mcp__plugin_my-plugin_database-tools__query

Server registration name for a plugin server:
plugin:<plugin-name>:<server-name>
```

Characters outside `[A-Za-z0-9_-]` are replaced with `_`.

**Permissioning**, from [Part 8](./claude-code-permissions-security.md):

```json
{
  "permissions": {
    "allow": ["mcp__github__*"],
    "deny":  ["mcp__slack__send_message", "mcp__*"]
  }
}
```

Remember: rules with **parentheses** on an `mcp__` name are **skipped** when loading a settings file (use `--disallowedTools` for parameter matching), and allow rules accept a tool-name glob only after a literal, glob-free `mcp__<server>__` prefix.

A server can also force approval on a specific tool by annotating it in `tools/list`:

```json
{ "name": "grant_access", "_meta": { "anthropic/requiresUserInteraction": true } }
```

That forces a prompt on **every** call, in every permission mode.

**Tool search** is on by default: only tool names and server instructions load at session start, with full schemas deferred until Claude needs a specific tool. It also reports failed server connections to Claude and waits for connecting servers before tool calls. `ENABLE_TOOL_SEARCH=false` loads everything upfront; `auto` loads schemas when they fit within 10% of the window. Run `/context all` to see per-tool token cost.

**Timeouts and limits:**

| Variable | Purpose | Default |
|---|---|---|
| `MCP_TIMEOUT` | Startup connection timeout | 30s |
| `MAX_MCP_OUTPUT_TOKENS` | Max output tokens per tool | 25,000 |
| `CLAUDE_CODE_MCP_TOOL_IDLE_TIMEOUT` | Idle timeout for unresponsive tools | 5 min |

Per-server, set `"timeout"` in `.mcp.json`.

**Trust and control.** Project `.mcp.json` servers require interactive approval on first use. Relevant settings: `enableAllProjectMcpServers`, `enabledMcpjsonServers`, `disabledMcpjsonServers`, `allowedMcpServers`, `deniedMcpServers`, `allowManagedMcpServersOnly`, `disableClaudeAiConnectors`. And `--strict-mcp-config` loads **only** the servers passed with `--mcp-config`, which is what you want in CI.

**Beyond tools:** servers can expose **resources** you reference with `@`, and **prompts** that appear as slash commands in the `/` menu.

<a id="128-part-12-cheat-sheet"></a>

### 12.8 Part 12 cheat sheet

| Hook event | Blocks on exit 2 | Use for |
|---|---|---|
| `PreToolUse` | ✓ | Block unsafe actions |
| `PostToolUse` | ✗ (already ran) | Lint, format, typecheck |
| `UserPromptSubmit` | ✓ (erases input) | Inject context, reject prompts |
| `Stop` | ✓ (overridden after 8) | Gate the turn on a check |
| `SessionStart` (matcher `compact`) | — | Re-inject context after compaction |
| `SessionEnd` | — | Archive the transcript (1.5s shared budget) |
| `InstructionsLoaded` | — | Debug which memory files loaded |
| `WorktreeCreate`/`Remove` | ✓ / — | Non-git VCS support |

| Hook option | Effect |
|---|---|
| `matcher` | Exact/list for simple strings; **regex** if it contains anything else |
| `if` | Gate on a permission-rule pattern — prefer this over shell guards |
| `args` present | Exec form: no shell interpretation |
| `args` absent | Shell form: pipes, `&&`, globs |
| `timeout` · `once` · `async` | Seconds · once per session · background |
| `${CLAUDE_PROJECT_DIR}` | Project root — **does not follow into a worktree**; read `cwd` instead |

| Exit code | Result |
|---|---|
| 0 | Success; JSON honoured |
| 2 | **Blocking**; stderr or `permissionDecisionReason` is the message |
| other | Non-blocking; JSON decision fields still honoured |

| MCP | |
|---|---|
| `claude mcp add --transport http\|stdio\|sse <name> …` | Add (use `--` before a stdio command) |
| `--scope local\|project\|user` | Scope (default local) |
| Precedence | local ▸ project ▸ user ▸ plugin ▸ claude.ai connectors |
| Tool name | `mcp__<server>__<tool>` |
| Rules | No parentheses in `mcp__` rules; allow globs need a literal server prefix |
| `requiresUserInteraction` | Server-side annotation forcing a prompt every call |
| Tool search | **On by default**; schemas deferred |
| `--strict-mcp-config` | Only `--mcp-config` servers — use in CI |

[↑ Back to top](#table-of-contents)

---

**Next:** [Automation & Agents (Parts 13–14)](./claude-code-automation-agents.md) — headless mode, CI, parallelism, and packaging.
