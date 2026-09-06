# Claude Code — Config & Settings (Parts 6–7)

Where every knob lives, which layer wins when two of them disagree, and how to make the tool behave the way you want across a machine, a repo, and a team.

> **Spec:** this doc follows the canonical spec in [`claude-code-mastery-guide.md`](./claude-code-mastery-guide.md#about-this-document). Written against **Claude Code v2.1.263**, verified **September 6, 2026**.

---

## Table of Contents

- [Part 6 — Settings Files](#part-6--settings-files)
  - [6.1 Beginner: four files and `/config`](#61-beginner-four-files-and-config)
  - [6.2 Working Knowledge: the precedence stack](#62-working-knowledge-the-precedence-stack)
  - [6.3 Working Knowledge: the keys you'll actually set](#63-working-knowledge-the-keys-youll-actually-set)
  - [6.4 Advanced: lists merge, and the four keys that don't](#64-advanced-lists-merge-and-the-four-keys-that-dont)
  - [6.5 Advanced: when a setting you wrote is ignored](#65-advanced-when-a-setting-you-wrote-is-ignored)
  - [6.6 Advanced: `--settings` and `--setting-sources`](#66-advanced---settings-and---setting-sources)
  - [6.7 Mastery: the full key inventory, grouped](#67-mastery-the-full-key-inventory-grouped)
  - [6.8 Part 6 cheat sheet](#68-part-6-cheat-sheet)
- [Part 7 — Model, Output, and Interface](#part-7--model-output-and-interface)
  - [7.1 Beginner: picking a model](#71-beginner-picking-a-model)
  - [7.2 Working Knowledge: effort, thinking, and fast mode](#72-working-knowledge-effort-thinking-and-fast-mode)
  - [7.3 Working Knowledge: output styles](#73-working-knowledge-output-styles)
  - [7.4 Advanced: the model resolution order and restricting it](#74-advanced-the-model-resolution-order-and-restricting-it)
  - [7.5 Advanced: the status line](#75-advanced-the-status-line)
  - [7.6 Mastery: environment variables that matter](#76-mastery-environment-variables-that-matter)
  - [7.7 Part 7 cheat sheet](#77-part-7-cheat-sheet)

---

## Part 6 — Settings Files

<a id="part-6"></a>

*This Part runs Beginner → Advanced in full. Its "Mastery" tier is a key inventory rather than internals, because the genuinely deep configuration material — how permission rules are evaluated — belongs to [Part 8](./claude-code-permissions-security.md) and is covered there instead.*

### 6.1 Beginner: four files and `/config`

Claude Code reads settings from four JSON files, plus managed settings an organisation can deploy ([settings](https://code.claude.com/docs/en/settings)):

| Scope | File | Who it affects |
|---|---|---|
| **User** | `~/.claude/settings.json` | You, in every project on this machine |
| **Shared project** | `.claude/settings.json` | Everyone working in that folder — commit it |
| **Project local** | `.claude/settings.local.json` | You, in this one project — kept out of git |
| **Managed** | `managed-settings.json`, MDM policy, or the claude.ai console | Everyone your org deploys it to |

> On Windows, `~/.claude` means `%USERPROFILE%\.claude`. `CLAUDE_CONFIG_DIR` moves the whole home-directory tree — settings, session history, plugins — somewhere else.

**Installing Claude Code creates none of these.** You create them, or Claude Code creates them for you: it writes `~/.claude/settings.json` the first time you change an option in `/config` that belongs at user scope, and it writes `.claude/settings.local.json` the first time you answer *"Yes, and don't ask again"* to a permission prompt.

The easiest way in:

```text
/config                      # open the settings interface
/config thinking=false       # set a key directly (v2.1.181+)
```

`/config --help` lists the keys it accepts. The `key=value` form works in non-interactive mode too, which is how you change a setting from a `-p` invocation.

A first, useful `~/.claude/settings.json`:

```json
{
  "model": "sonnet",
  "includeGitInstructions": true,
  "permissions": {
    "allow": [
      "Bash(npm run test:*)",
      "Bash(git status)",
      "Bash(git diff *)"
    ]
  }
}
```

> There is a **fifth** file, `~/.claude.json`, that Claude Code writes for itself — sign-in session, MCP server config, and similar. You don't edit it ([the `.claude` directory](https://code.claude.com/docs/en/claude-directory)).

### 6.2 Working Knowledge: the precedence stack

When the same key appears in more than one place, the highest level that sets it wins:

```text
  ┌─────────────────────────────────────────────────────────┐
  │ 1. MANAGED SETTINGS                                     │  your organization
  │    managed-settings.json, MDM, claude.ai console        │  nothing you set
  │                                                          │  overrides it*
  ├─────────────────────────────────────────────────────────┤
  │ 2. COMMAND LINE                                         │  you, this session
  │    claude --settings '{"..."}'                          │  lasts one session
  ├─────────────────────────────────────────────────────────┤
  │ 3. PROJECT LOCAL                                        │  you, this project
  │    .claude/settings.local.json                          │  gitignored
  ├─────────────────────────────────────────────────────────┤
  │ 4. SHARED PROJECT                                       │  everyone in the repo
  │    .claude/settings.json                                │  commit this
  ├─────────────────────────────────────────────────────────┤
  │ 5. USER                                                 │  you, every project
  │    ~/.claude/settings.json                              │
  └─────────────────────────────────────────────────────────┘
        * except a few security-sensitive keys, where a stricter
          value from a lower level wins — see §6.5
```

Worked example, using `spinnerTipsEnabled` (from the [official walkthrough](https://code.claude.com/docs/en/settings#precedence-examples)):

| Situation | Result | Can you get your value back? |
|---|---|---|
| You set `false` in user settings; team sets `true` in `.claude/settings.json` | Tips show — shared project beats user | Yes: set `false` in `.claude/settings.local.json` |
| Org's managed settings set `true` | Tips show, everywhere | No. Run `/status` to see which managed source applies, then talk to your admin |
| You launched with `claude --settings '{"spinnerTipsEnabled": true}'` | Tips show for that session only | Yes, next session — `--settings` writes to no file |

**Environment variables are not a level in this stack.** When a behaviour has both a shell variable and a settings key, which one applies is decided **per pair**, not by level. `ANTHROPIC_MODEL` exported in your shell overrides the `model` setting from any file. Check the specific key's entry in the [settings reference](https://code.claude.com/docs/en/settings-reference) and the variable's row in [environment variables](https://code.claude.com/docs/en/env-vars) rather than assuming.

**When edits take effect.** Settings files are re-read at launch; some changes apply immediately mid-session, some don't. Output style is the classic one that doesn't — it's part of the system prompt, which is read once at session start, so a change needs `/clear` or a new session.

### 6.3 Working Knowledge: the keys you'll actually set

Out of a key inventory well over a hundred entries, these are the ones that come up in ordinary use:

```json
{
  "model": "sonnet",
  "effortLevel": "high",
  "outputStyle": "Concise",

  "permissions": {
    "defaultMode": "acceptEdits",
    "allow": ["Bash(npm run lint)", "Bash(npm run test:*)"],
    "ask":   ["Bash(git push *)"],
    "deny":  ["Read(./.env)", "Read(./secrets/**)"]
  },

  "env": {
    "BASH_DEFAULT_TIMEOUT_MS": "300000",
    "DISABLE_TELEMETRY": "1"
  },

  "hooks": {
    "PostToolUse": [
      { "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "npm run lint:fix" }] }
    ]
  },

  "statusLine": { "type": "command", "command": "~/.claude/statusline.sh" },
  "cleanupPeriodDays": 90,
  "autoMemoryEnabled": true,
  "claudeMdExcludes": ["**/other-team/CLAUDE.md"],
  "enabledPlugins": { "code-review@claude-plugins-official": true }
}
```

| Key | What it does |
|---|---|
| `model` | Model Claude Code starts with |
| `effortLevel` | Default reasoning effort for models without a saved level |
| `outputStyle` | Role, tone, and default response format |
| `permissions` | `allow` / `ask` / `deny` rules plus `defaultMode` — see [Part 8](./claude-code-permissions-security.md) |
| `env` | Environment variables for every session **and its subprocesses** |
| `hooks` | Lifecycle automation — see [Part 12](./claude-code-extensibility.md) |
| `statusLine` | Custom status bar command |
| `cleanupPeriodDays` | Transcript and checkpoint-snapshot retention (default 30) |
| `autoMemoryEnabled` · `autoMemoryDirectory` | Auto memory on/off · where it lives |
| `claudeMdExcludes` | Skip specific `CLAUDE.md` files |
| `enabledPlugins` | Turn individual plugins on or off per scope |
| `agent` | Start every session as a named subagent |
| `skillOverrides` | Hide or collapse a skill without editing it |
| `disableAllHooks` | Kill switch for hooks, the status line, and a custom file-suggestion command |
| `sandbox` | Bash sandbox configuration — see [Part 9](./claude-code-permissions-security.md) |
| `includeGitInstructions` | Set `false` to strip the built-in commit and PR instructions from the system prompt |
| `attribution` | Customise (or remove) the attribution Claude adds to commits and PRs — supersedes the deprecated `includeCoAuthoredBy` |

**`env` is more powerful than it looks.** It sets variables for every session *and its subprocesses*, which means it is how you configure Bash timeouts, telemetry, tool search, and provider routing in a way that survives across machines when you commit it. It is also, for exactly the same reason, a thing to look at carefully in someone else's repo before you trust the folder.

### 6.4 Advanced: lists merge, and the four keys that don't

When the same **list** key appears in more than one file, Claude Code **combines the lists** rather than picking one, so each file can add entries without removing another's ([settings](https://code.claude.com/docs/en/settings#lists-merge-instead-of-overriding)). That is what makes `permissions.allow` workable across user, project, and local scope, and it's how `claudeMdExcludes` accumulates.

Four keys deliberately break the merge rule, because position or completeness carries meaning:

| Key | Behaviour |
|---|---|
| `fallbackModel` | An ordered chain — the whole value is taken from the highest-precedence file that defines it |
| `modelPicker` | One ordered list of rows plus a replace flag; never merged across sources |
| `availableModels` | When applied managed settings define it, that list is used **as-is** and entries you add lower down are ignored |
| `modelSettings` | Resolved one model at a time, together with `effortLevel` |

> **Wrong vs. right — trying to remove a team allow rule.**
>
> ```json
> // Wrong: .claude/settings.local.json
> // "I'll just not list Bash(git push *) in my local allow rules."
> { "permissions": { "allow": ["Bash(npm run *)"] } }
> // The team's .claude/settings.json entry still applies — lists MERGE.
> ```
>
> ```json
> // Right: deny beats allow, at any layer
> { "permissions": { "deny": ["Bash(git push *)"] } }
> ```
>
> You cannot subtract from a merged list by omission. To reverse an allow, add a `deny`.

### 6.5 Advanced: when a setting you wrote is ignored

Start with `/status`, which shows which files Claude Code actually loaded, then work through these causes ([settings](https://code.claude.com/docs/en/settings#troubleshoot-a-setting-that-doesnt-apply)):

**1. A higher level sets it.** Another settings file, a `--settings` flag, or a managed source. `/status` names the managed source when one applies. As of v2.1.261, `/status` and `claude doctor` also show an **Organization policy** line explaining policy load failures.

**2. A flag or environment variable overrides it regardless of file.** `ANTHROPIC_MODEL` beats the `model` setting from anywhere.

**3. A security key keeps its strict value.** For a handful of security-sensitive keys, Claude Code honours the **restrictive** value from *any* file — so a project setting can tighten something even against a managed value, and a managed permissive value doesn't loosen a locally strict one. `disableClaudeAiConnectors` is one example.

**4. That file can't set that value.** The important instance: **`permissions.defaultMode` values `auto` and `bypassPermissions` do not take effect from project or local settings.** They must come from user, `--settings`, or managed settings. This is deliberate — it stops a repository you clone from starting your sessions in a permissive mode. If you set `defaultMode: "auto"` in `.claude/settings.json` and sessions keep starting in Manual with no error, this is why.

**5. The file didn't load, or didn't parse.** A malformed JSON file is skipped. `/doctor` explicitly checks for unparseable settings files.

**6. The value needs workspace trust.** Some of what you commit waits until each teammate trusts the folder. Project `allow` rules are the main case — see [Part 8](./claude-code-permissions-security.md). Note the asymmetry: the *local* file's allow rules don't wait for trust while the file stays untracked, because it's yours rather than the repository's.

> ### Real Scenario — the settings file that only worked for one person
>
> A team lead adds a shared `.claude/settings.json` so everyone gets the same setup, commits it, and tells the team it's live:
>
> ```json
> {
>   "permissions": {
>     "defaultMode": "auto",
>     "allow": ["Bash(pnpm *)", "Bash(git *)"]
>   },
>   "hooks": {
>     "PostToolUse": [{ "matcher": "Edit|Write",
>       "hooks": [{ "type": "command", "command": "pnpm lint:fix" }] }]
>   }
> }
> ```
>
> It works perfectly on their machine. Two teammates report that nothing changed: still prompted for every command, no lint on save.
>
> **Three separate causes, all in one file:**
>
> 1. **`defaultMode: "auto"` is ignored from project settings.** It only takes effect from user, `--settings`, or managed settings. The lead's machine already had `"defaultMode": "auto"` in `~/.claude/settings.json` from an earlier experiment, which is why it "worked" for them.
> 2. **The `allow` rules waited on workspace trust.** Teammates who hadn't yet accepted the trust dialog for that folder didn't get the project allow rules.
> 3. **The hook waited on trust too**, for the same reason — and a hook that silently doesn't fire looks exactly like a hook that isn't configured.
>
> **Diagnosis path:** `/status` on a teammate's machine showed `.claude/settings.json` loaded but the mode as Manual — the tell that the file loaded and one key inside it was rejected rather than the whole file failing. **The fix:** move `defaultMode` guidance into the team's onboarding docs as a user-settings step, keep the allow rules and hook in the shared file, and have everyone open the repo once interactively to accept trust.

### 6.6 Advanced: `--settings` and `--setting-sources`

Two flags that matter for automation and for reproducing a bug:

```bash
claude --settings ./ci-settings.json          # a file
claude --settings '{"model":"haiku"}'         # or inline JSON
```

`--settings` sits above every file except managed, applies for one session, and writes nothing to disk. JSON you pass is merged with the files rather than replacing them.

```bash
claude --setting-sources user,project         # load only these sources
```

`--setting-sources` restricts which sources load at all. Excluding `project` also skips project rules in `.claude/rules/`; excluding `local` skips `CLAUDE.local.md`. This is the clean way to answer *"is this behaviour coming from my machine or from the repo?"*

For CI, the stronger tool is `--bare`, which skips auto-discovery of hooks, skills, commands, subagents, plugins, MCP servers, auto memory, and `CLAUDE.md` entirely — see [Part 13](./claude-code-automation-agents.md).

**Reproducing a report cleanly:**

```bash
claude --safe-mode            # start with all customizations disabled
claude --bare -p "..."        # or, for scripted runs
```

<a id="p6-inventory"></a>

### 6.7 Mastery: the full key inventory, grouped

The [settings reference](https://code.claude.com/docs/en/settings-reference) is the authority and is long. What follows is a map of the territory so you know a key exists when you need one. Groups, with representative keys:

**Models and responses** — `model`, `availableModels`, `enforceAvailableModels`, `modelPicker`, `modelOverrides`, `modelSettings`, `modelPricing`, `fallbackModel`, `effortLevel`, `alwaysThinkingEnabled`, `showThinkingSummaries`, `fastMode`, `fastModePerSessionOptIn`, `advisorModel`, `outputStyle`, `language`

**Context and memory** — `autoCompactEnabled`, `autoCompactWindow`, `autoMemoryEnabled`, `autoMemoryDirectory`, `claudeMd` *(managed only)*, `claudeMdExcludes`, `promptCacheTtl`, `subagentPromptCacheTtl`, `bashOutputMaxChars`, `taskOutputMaxChars`

**Permissions and safety** — `permissions` (`allow`/`ask`/`deny`/`defaultMode`/`additionalDirectories`/`blockReadsOutsideWorkingDirectories`), `autoMode`, `disableAutoMode`, `sandbox`, `skipDangerousModePermissionPrompt`, `skipAutoPermissionPrompt`, `allowManagedPermissionRulesOnly`

**Extensions** — `hooks`, `disableAllHooks`, `allowedHttpHookUrls`, `httpHookAllowedEnvVars`, `allowManagedHooksOnly`, `skillOverrides`, `skillListingBudgetFraction`, `skillListingMaxDescChars`, `disableBundledSkills`, `disableSkillShellExecution`, `agent`, `enabledPlugins`, `extraKnownMarketplaces`, `strictKnownMarketplaces`, `blockedMarketplaces`, `pluginConfigs`, `strictPluginOnlyCustomization`

**MCP** — `allowedMcpServers`, `deniedMcpServers`, `enabledMcpjsonServers`, `disabledMcpjsonServers`, `enableAllProjectMcpServers`, `allowManagedMcpServersOnly`, `allowAllClaudeAiMcps`, `disableClaudeAiConnectors`

**Parallelism and messaging** — `disableAgentView`, `disableWorkflows`, `enableWorkflows`, `crossSessionInbound`, `isolatePeerMachines`, `channelsEnabled`, `allowedChannelPlugins`, `worktree` (`baseRef`)

**Interface** — `statusLine`, `editorMode`, `diffTool`, `externalEditorContext`, `autoScrollEnabled`, `spinnerTipsEnabled`, `spinnerTipsOverride`, `spinnerVerbs`, `showTurnDuration`, `prefersReducedMotion`, `axScreenReader`, `emojiCompletionEnabled`, `spellcheck`, `promptSuggestionEnabled`, `footerLinksRegexes`, `prUrlTemplate`, `respectGitignore`, `fileSuggestion`, `respondToBashCommands`, `awaySummaryEnabled`

**Session and data** — `cleanupPeriodDays`, `desktopSessionCleanupPeriodDays`, `fileCheckpointingEnabled`, `attribution`, `includeGitInstructions`, `plansDirectory`

**Auth and org control** — `apiKeyHelper`, `forceLoginMethod`, `forceLoginOrgUUID`, `forceLoginGatewayUrl`, `awsAuthRefresh`, `awsCredentialExport`, `gcpAuthRefresh`, `policyHelper`, `managedSourcesBehavior`, `parentSettingsBehavior`, `minimumVersion`, `requiredMinimumVersion`, `requiredMaximumVersion`, `autoUpdatesChannel`, `companyAnnouncements`

**Two deprecated keys to recognise in old configs:** `includeCoAuthoredBy` (use `attribution`) and `keybindingFlavor` (no effect at all since readline conventions became standard in v2.1.261). `permissionExplainerEnabled` was removed in v2.1.257.

<a id="68-part-6-cheat-sheet"></a>

### 6.8 Part 6 cheat sheet

| File | Scope | Commit it? |
|---|---|---|
| `~/.claude/settings.json` | You, all projects | n/a |
| `.claude/settings.json` | Everyone in the repo | **Yes** |
| `.claude/settings.local.json` | You, this project | No — auto-gitignored when Claude Code creates it |
| `managed-settings.json` / MDM / console | Org-wide | Deployed by admins |
| `~/.claude.json` | Claude Code's own state | Never edit |

| Precedence, highest first |
|---|
| Managed → command line (`--settings`) → project local → shared project → user |
| Lists **merge** across layers (except `fallbackModel`, `modelPicker`, `availableModels`, `modelSettings`) |
| Env vars are **not** a layer — resolved per key/variable pair |

| Command / flag | Effect |
|---|---|
| `/config` · `/config key=value` | Settings UI · set a key directly |
| `/status` | Which files loaded, which managed source applies, org policy line |
| `/doctor` | Setup checkup, including unparseable settings |
| `--settings <file\|json>` | One-session override above every file but managed |
| `--setting-sources user,project` | Load only these sources |
| `--safe-mode` | Start with all customisations disabled |
| `--bare` | Skip auto-discovery entirely (CI) |

| Gotcha | |
|---|---|
| `permissions.defaultMode: "auto"` / `"bypassPermissions"` | **Ignored** from project and local settings |
| Project `allow` rules and hooks | Wait for **workspace trust** |
| Output style changes | Need `/clear` or a new session |
| Omitting an entry from a list | Does **not** remove it — add a `deny` |

[↑ Back to top](#table-of-contents)

---

## Part 7 — Model, Output, and Interface

<a id="part-7"></a>

*Beginner and Working Knowledge carry most of this Part. The Advanced tier is the resolution order and the status line; there is no separate Mastery tier beyond the environment-variable inventory, because the deep model material belongs to the Claude API rather than to Claude Code.*

### 7.1 Beginner: picking a model

```text
/model              # open the picker (press `s` on a row to switch for this session only)
/model opus         # switch and save as your default for new sessions
```

```bash
claude --model sonnet
```

**Aliases**, which are what you should normally use ([model config](https://code.claude.com/docs/en/model-config)):

| Alias | Resolves to |
|---|---|
| `best` | Latest Fable where available, otherwise Opus |
| `fable` | Most capable for long, complex tasks |
| `opus` | Complex reasoning |
| `sonnet` | Daily coding |
| `haiku` | Fast and cheap for simple tasks |
| `opusplan` | **Opus for planning, auto-switching to Sonnet for execution** |
| `sonnet[1m]`, `opus[1m]` | 1M-token context variants |
| `default` | Your account type's default (Opus 5 for Max/Enterprise, Sonnet 5 for Pro) |

Full IDs when you need to pin: `claude-opus-5`, `claude-sonnet-5`, `claude-fable-5-1`, `claude-haiku-4-5`.

`opusplan` is worth knowing about specifically: it gives you Opus-quality planning without paying Opus rates through a long implementation phase.

### 7.2 Working Knowledge: effort, thinking, and fast mode

Three separate dials that people routinely conflate.

**Effort level** — how much reasoning the model does before answering:

```text
/effort high
/effort status
```

```bash
claude --effort xhigh
export CLAUDE_CODE_EFFORT_LEVEL=xhigh
```

| Level | Use for |
|---|---|
| `low` | Short, latency-sensitive, non-critical |
| `medium` | Cost-sensitive work tolerating lower intelligence |
| `high` | **Default** — balanced for most coding |
| `xhigh` | Deeper reasoning at higher token cost |
| `max` | Demanding tasks — test before adopting broadly (session-only) |
| `ultracode` | Dynamic workflows with `xhigh` reasoning (session-only, key persists) |

Fable 5.1, Fable 5, Opus 5, Sonnet 5, Opus 4.8 and 4.7 support all five levels; Opus 4.6 and Sonnet 4.6 support `low`/`medium`/`high`/`max`. Include **`ultrathink`** in a prompt for deeper reasoning on that one turn without changing the session setting.

**Extended thinking** — whether the model reasons visibly before responding:

- Toggle for the session: `Option+T` / `Alt+T`
- Global default: `/config` → thinking, saved as `alwaysThinkingEnabled`
- Off entirely: `MAX_THINKING_TOKENS=0` — **except on Fable 5.1 and Fable 5, where it can't be turned off**
- Thinking output collapses by default; `showThinkingSummaries: true` shows full summaries

> **You are charged for thinking tokens even when they're collapsed.** Collapsing is a display choice, not a billing one.

**Fast mode** — same model, lower latency, higher cost per token ([fast mode](https://code.claude.com/docs/en/fast-mode)):

```text
/fast
```

- **Research preview.** Supported on **Opus 5 and Opus 4.8 only** — not Sonnet, not Haiku. Enabling it on an unsupported model switches you to Opus.
- Toggle with `/fast`, `Option+O`/`Alt+O`, or `"fastMode": true` in user settings. A `↯` icon shows while it's on.
- **Enable it at the start of a session.** The first time you enable it in a conversation you pay the full fast-mode uncached input price for the *entire* existing context — the deeper into a conversation, the more that costs. The charge applies once per conversation, so toggling off and on again later doesn't repeat it.
- On subscription plans it draws from **usage credits**, not your plan's included usage.
- Hitting the fast-mode rate limit falls back to standard speed automatically (the `↯` turns grey) and re-enables when the cooldown expires.
- `fastModePerSessionOptIn: true` makes every session start with it off — the cost-control setting for orgs where people run many concurrent sessions.

**Effort vs. fast mode**, since they both "make it faster":

| Setting | Effect |
|---|---|
| **Fast mode** | Same model quality, lower latency, **higher** cost |
| **Lower effort level** | Less thinking time, faster, potentially lower quality on complex tasks |

They combine — fast mode plus low effort is the maximum-speed configuration for straightforward work.

### 7.3 Working Knowledge: output styles

Output styles change **how Claude responds, not what Claude knows**. They modify the system prompt to set role, tone, and format ([output styles](https://code.claude.com/docs/en/output-styles)).

Five built-ins:

| Style | Behaviour |
|---|---|
| **Default** | The standard software-engineering system prompt |
| **Proactive** | Executes immediately, makes reasonable assumptions, prefers action over planning. **Stronger autonomous guidance than auto mode**, and independent of it — your permission mode still decides what runs |
| **Concise** | Leads with the result, skips preamble and narration, short by default — while doing the engineering just as thoroughly. Always keeps error reports, security warnings, and destructive-action confirmations complete. (v2.1.237+) |
| **Explanatory** | Adds educational "Insights" while working |
| **Learning** | Asks *you* to write small strategic pieces, leaving `TODO(human)` markers |

> **The `/output-style` command was deprecated in v2.1.73 and removed in v2.1.91.** Use `/config` → **Output style**, or set the key directly. Older guides still tell you to run `/output-style`; it won't work.

```json
{ "outputStyle": "Concise" }
```

A **custom** style is a Markdown file in `~/.claude/output-styles`, `.claude/output-styles`, or the managed settings directory:

```markdown
---
name: Diagrams first
description: Lead every explanation with a diagram
keep-coding-instructions: true
---

When explaining code, architecture, or data flow, start with a Mermaid diagram
showing the structure, then explain in prose.

## Diagram conventions

Use `flowchart TD` for control flow and `sequenceDiagram` for request paths.
Keep diagrams under 15 nodes.
```

The critical field is **`keep-coding-instructions`**, which defaults to **`false`**. Leaving it out **removes Claude Code's built-in software-engineering instructions** — how to scope changes, write comments, verify work. That's correct when Claude isn't doing engineering at all (a writing assistant, a data analyst) and quietly wrong when you only wanted to change the tone.

Three behavioural facts:

1. Output style is part of the system prompt, read **once at session start** — changes need `/clear` or a new session.
2. Styles apply to the **main conversation only**. A subagent runs its own system prompt, so styles don't affect it. A fork is the exception, since it inherits the parent's full system prompt.
3. Plugins can ship styles in an `output-styles/` directory, and a plugin style with `force-for-plugin: true` applies automatically whenever that plugin is enabled, **overriding your `outputStyle` setting**.

**Choosing between the customisation mechanisms** — this table is the one to remember:

| Feature | How it works | Use when |
|---|---|---|
| **Output style** | Modifies the system prompt | You want a different role, tone, or format **every turn** |
| **`CLAUDE.md`** | Adds a user message after the system prompt | Claude should always know your project conventions |
| **`--append-system-prompt`** | Appends to the system prompt without removing anything | A one-off addition for a single invocation |
| **Subagent** | Its own system prompt, model, and tools | A separately scoped helper for a focused task |
| **Skill** | Task-specific instructions, loaded when invoked or relevant | A reusable workflow |

<a id="p7-resolution"></a>

### 7.4 Advanced: the model resolution order and restricting it

First match wins ([model config](https://code.claude.com/docs/en/model-config)):

```text
  1. /model <name>            during the session
  2. claude --model <name>    at startup
  3. ANTHROPIC_MODEL          environment variable
  4. "model" in settings.json
  5. ANTHROPIC_DEFAULT_MODEL  default for new sessions (v2.1.236+)
  6. Organization default     set by an admin
```

The alias-resolution variables are separate and useful on third-party providers, where you need the alias to point at a provider-specific deployment ID:

```bash
export ANTHROPIC_DEFAULT_OPUS_MODEL='us.anthropic.claude-opus-4-8'
export ANTHROPIC_DEFAULT_SONNET_MODEL='us.anthropic.claude-sonnet-4-5'
export ANTHROPIC_DEFAULT_OPUS_MODEL='us.anthropic.claude-opus-4-8[1m]'   # with 1M context
```

Also: `ANTHROPIC_DEFAULT_HAIKU_MODEL`, `ANTHROPIC_DEFAULT_FABLE_MODEL`, and `CLAUDE_CODE_SUBAGENT_MODEL` (the default for subagents and teammates).

**Restricting model choice**, for an org that wants to control spend or availability:

| Key | Effect |
|---|---|
| `availableModels` | Allowlist. When managed settings define it, that list is applied **as-is** and lower-layer additions are ignored |
| `enforceAvailableModels` | Keep the `/model` **Default** choice inside the allowlist |
| `modelPicker` | Which models the picker lists, in your order, with your labels |
| `modelOverrides` | Map model IDs to provider IDs, such as Bedrock ARNs |
| `fallbackModel` | Backup chain when the primary is overloaded (`claude --fallback-model sonnet,haiku`) |
| `modelPricing` | Report spend at contracted rates instead of list price |

One interaction worth knowing: with `availableModels` restricting choice, `opusplan` substitutes the newest **permitted** Opus version if the latest is excluded.

**Resume interacts with all of this.** A resumed session continues on the model it was using — unless that model is retired, isn't in `availableModels`, a `--model` flag or `ANTHROPIC_MODEL`-family variable picks one at launch, or you're on a provider using deployment IDs. See [Part 2](./claude-code-daily-driver.md).

<a id="p7-statusline"></a>

### 7.5 Advanced: the status line

The status line runs **any shell script you configure**, receives JSON session data on stdin, and displays whatever the script prints ([status line](https://code.claude.com/docs/en/statusline)). It runs locally and consumes no API tokens.

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/statusline.sh",
    "padding": 2
  }
}
```

Or inline, without a script file:

```json
{
  "statusLine": {
    "type": "command",
    "command": "jq -r '\"[\\(.model.display_name)] \\(.context_window.used_percentage // 0)% context\"'"
  }
}
```

Optional fields: `padding` (extra horizontal spacing, default 0), `refreshInterval` (re-run every N seconds, minimum 1 — set it when you show time-based data), and `hideVimModeIndicator` (suppress the built-in `-- INSERT --` when your script renders `vim.mode` itself).

**When it re-runs:** at session start (including resume), then when a new assistant message arrives, `/compact` finishes, the permission mode changes, vim mode toggles, you change the `command`, a `refreshInterval` elapses, a rate-limit window reaches its `resets_at`, or a warm prompt cache reaches its `expires_at`. Updates are debounced at 300ms; a change to `command` itself skips the debounce.

**The data you get** — the fields worth building around:

| Field | Contains |
|---|---|
| `model.id`, `model.display_name` | Current model |
| `workspace.current_dir`, `workspace.project_dir`, `workspace.added_dirs` | Directories (prefer `workspace.current_dir` over `cwd`) |
| `workspace.git_worktree` | Worktree name, when inside a linked worktree |
| `workspace.repo.{host,owner,name}` | Repo identity parsed from `origin` |
| `context_window.used_percentage`, `.remaining_percentage` | **Pre-calculated** — use these rather than computing |
| `context_window.context_window_size` | 200000, or 1000000 for extended-context models |
| `context_window.current_usage.{input,output,cache_creation,cache_read}_tokens` | Per-component breakdown |
| `cost.total_cost_usd`, `.total_duration_ms`, `.total_lines_added/removed` | Session cost and churn |
| `effort.level`, `thinking.enabled`, `fast_mode` | Live session settings |
| `rate_limits.five_hour.*`, `.seven_day.*` | Usage percentage and `resets_at` |
| `prompt_cache` | Hit ratio, misses, expiry |
| `session_id`, `session_name`, `prompt_id`, `transcript_path` | Identity and transcript location |
| `exceeds_200k_tokens` | Fixed 200k threshold flag, regardless of actual window size |

Two precision notes: `used_percentage` is calculated from **input tokens only** (`input + cache_creation + cache_read`), so match that formula if you compute it yourself; and `current_usage` is `null` before the first API call in a session, and again right after `/compact` until the next call repopulates it.

**Output capabilities:** multiple lines (each `print`/`echo` is a row), ANSI colours, and OSC 8 clickable links. Claude Code captures the output rather than connecting your script to the terminal, so `tput cols` cannot read the terminal width from inside the script.

> **Wrong vs. right — caching expensive status line work.**
>
> ```bash
> # Wrong: process ID changes on every invocation, so the cache never hits
> CACHE_FILE="/tmp/statusline-git-cache-$$"
> ```
>
> ```bash
> # Right: session_id is stable for the session and unique across sessions
> SESSION_ID=$(echo "$input" | jq -r '.session_id')
> CACHE_FILE="/tmp/statusline-git-cache-$SESSION_ID"
> ```
>
> The wrong version defeats the cache entirely and, worse, a fixed filename would let concurrent sessions in different repositories read each other's git state. The docs call this out specifically.

A minimal working script:

```bash
#!/usr/bin/env bash
input=$(cat)
model=$(echo "$input" | jq -r '.model.display_name')
pct=$(echo "$input"  | jq -r '.context_window.used_percentage // 0')
branch=$(git branch --show-current 2>/dev/null)
printf '[%s] %s%% ctx  %s' "$model" "$pct" "${branch:+⎇ $branch}"
```

Related: `footerLinksRegexes` makes issue or review IDs in output into clickable badges below the input box **without writing a script** — reach for that first if links are all you want.

<a id="p7-envvars"></a>

### 7.6 Mastery: environment variables that matter

The full list is in [environment variables](https://code.claude.com/docs/en/env-vars). These are the ones that come up.

**Authentication**, in priority order: `ANTHROPIC_API_KEY` → `ANTHROPIC_AUTH_TOKEN` (custom `Authorization` header) → `ANTHROPIC_PROFILE` → claude.ai subscription via `/login`.

| Variable | Purpose | Default |
|---|---|---|
| `ANTHROPIC_API_KEY` | API key auth (overrides subscription login) | — |
| `ANTHROPIC_BASE_URL` | Route through a proxy or gateway | — |
| `ANTHROPIC_MODEL`, `ANTHROPIC_DEFAULT_MODEL` | Session model · default for new sessions | — |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Default model for subagents | Main conversation's |
| `CLAUDE_CODE_EFFORT_LEVEL` | Session effort | `high` |
| `MAX_THINKING_TOKENS=0` | Disable extended thinking (not on Fable 5.x) | — |
| `API_TIMEOUT_MS` | API request timeout | 600000 (10 min) |
| `BASH_DEFAULT_TIMEOUT_MS` | Bash command timeout | 120000 (2 min) |
| `BASH_MAX_OUTPUT_LENGTH` | Max Bash output chars | 30000 (max 150000) |
| `CLAUDE_CONFIG_DIR` | Move `~/.claude` elsewhere | `~/.claude` |
| `CLAUDE_CODE_PROJECT_DIR_NAME` | Name the project dir yourself (v2.1.234+, needs `CLAUDE_CONFIG_DIR`) | Derived from path |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | Auto-compact threshold | Model-tuned |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT=1` | Treat 1M-native models as 200K | — |
| `ENABLE_TOOL_SEARCH` | `auto` / `false` for MCP schema loading | On |
| `MCP_TIMEOUT`, `MAX_MCP_OUTPUT_TOKENS` | MCP startup timeout · per-tool output cap | 30s · 25,000 |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` | No background Bash | — |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` | No auto memory | — |
| `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` | Concurrency cap | 20 |
| `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` | Nesting cap | 3 |
| `CLAUDE_CODE_FORK_MODE=off` | Disable fork mode | On (interactive) |
| `CLAUDE_CODE_SKIP_PROMPT_HISTORY` | Suppress transcript writes | — |
| `CLAUDE_CODE_DISABLE_FAST_MODE=1` | Disable fast mode | — |
| `DISABLE_TELEMETRY`, `DISABLE_ERROR_REPORTING` | Turn off reporting | — |
| `CLAUDE_CODE_NEW_INIT=1` | Interactive `/init` flow | — |
| `CLAUDE_CODE_SYNC_SKILLS=1` | Sync claude.ai skills into local sessions | — |

**Setting them.** Shell exports take effect on the next `claude` launch. The `env` settings key takes effect immediately and applies to subprocesses too:

```json
{
  "env": {
    "API_TIMEOUT_MS": "1200000",
    "BASH_DEFAULT_TIMEOUT_MS": "300000",
    "DISABLE_TELEMETRY": "1"
  }
}
```

Two parsing conveniences: numeric values accept scientific notation (`2e3` = 2000) and digit separators (`64_000`); boolean variables take `1`/`true` or `0`/`false`.

**The one exception to `env`:** `CLAUDE_CODE_PROJECT_DIR_NAME` is read once at startup from the shell environment, so an `env` block in a settings file cannot set it.

<a id="77-part-7-cheat-sheet"></a>

### 7.7 Part 7 cheat sheet

| Command | Effect |
|---|---|
| `/model [name]` | Picker (press `s` for session-only) or direct switch |
| `/effort <level\|auto\|status>` | Reasoning effort |
| `/fast` | Toggle fast mode (Opus 5 / 4.8 only) |
| `/config` → Output style | Change output style — **`/output-style` was removed in v2.1.91** |
| `/statusline` | Set up or remove the status line |
| `/usage` (`/cost`) | Spend, cache metrics, rate-limit state |
| `/status` | Loaded settings files, login method, org policy |

| Dial | Effect |
|---|---|
| Model | Capability tier |
| Effort (`low`…`max`, `ultracode`) | How much reasoning before answering |
| Extended thinking | Whether it reasons visibly (billed either way) |
| Fast mode | Same quality, lower latency, higher cost, Opus only |
| `ultrathink` in a prompt | One-turn deep reasoning, no setting change |

| Model resolution, first match wins |
|---|
| `/model` → `--model` → `ANTHROPIC_MODEL` → `model` setting → `ANTHROPIC_DEFAULT_MODEL` → org default |

| Status line field | Use |
|---|---|
| `context_window.used_percentage` | Pre-calculated; input tokens only |
| `session_id` | Stable cache key (not `$$`) |
| `rate_limits.*.used_percentage` / `.resets_at` | Quota display; triggers a re-run at reset |
| `cost.total_cost_usd` | Client-side estimate at list price |

[↑ Back to top](#table-of-contents)

---

**Next:** [Permissions & Security (Parts 8–9)](./claude-code-permissions-security.md) — what Claude may do, what stops it, and how to contain a mistake.
