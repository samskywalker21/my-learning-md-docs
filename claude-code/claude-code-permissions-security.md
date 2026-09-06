# Claude Code — Permissions & Security (Parts 8–9)

What Claude is allowed to do, what asks you first, what nothing can override, and what contains a mistake once one happens.

> **Spec:** this doc follows the canonical spec in [`claude-code-mastery-guide.md`](./claude-code-mastery-guide.md#about-this-document). Written against **Claude Code v2.1.263**, verified **September 6, 2026**.
>
> This is the doc to read before you run anything unattended. If you read only one section, make it [§8.2](#82-working-knowledge-the-six-modes) and [§8.7](#87-mastery-protected-and-critical-paths).

---

## Table of Contents

- [Part 8 — The Permission Model](#part-8--the-permission-model)
  - [8.1 Beginner: `Shift+Tab` and the prompt](#81-beginner-shifttab-and-the-prompt)
  - [8.2 Working Knowledge: the six modes](#82-working-knowledge-the-six-modes)
  - [8.3 Working Knowledge: rules — allow, ask, deny](#83-working-knowledge-rules--allow-ask-deny)
  - [8.4 Advanced: Bash rules and the wildcard traps](#84-advanced-bash-rules-and-the-wildcard-traps)
  - [8.5 Advanced: Read and Edit rules — gitignore syntax with anchors](#85-advanced-read-and-edit-rules--gitignore-syntax-with-anchors)
  - [8.6 Advanced: WebFetch, MCP, Agent, and Cd rules](#86-advanced-webfetch-mcp-agent-and-cd-rules)
  - [8.7 Mastery: protected paths, critical paths, and workspace trust](#87-mastery-protected-and-critical-paths)
  - [8.8 Part 8 cheat sheet](#88-part-8-cheat-sheet)
- [Part 9 — Containment](#part-9--containment)
  - [9.1 Beginner: three layers, not one](#91-beginner-three-layers-not-one)
  - [9.2 Working Knowledge: auto mode and its classifier](#92-working-knowledge-auto-mode-and-its-classifier)
  - [9.3 Working Knowledge: the Bash sandbox](#93-working-knowledge-the-bash-sandbox)
  - [9.4 Advanced: sandbox filesystem and network policy](#94-advanced-sandbox-filesystem-and-network-policy)
  - [9.5 Advanced: prompt injection as the real threat model](#95-advanced-prompt-injection-as-the-real-threat-model)
  - [9.6 Mastery: what the sandbox is not](#96-mastery-what-the-sandbox-is-not)
  - [9.7 Part 9 cheat sheet](#97-part-9-cheat-sheet)

---

## Part 8 — The Permission Model

<a id="part-8"></a>

### 8.1 Beginner: `Shift+Tab` and the prompt

Claude Code decides, for every tool call, whether to run it, ask you, or refuse. Two things drive that decision: the **permission mode** (the baseline) and **permission rules** (specific exceptions).

Press `Shift+Tab` to cycle modes. The status bar tells you which one you're in.

When Claude asks, you get a prompt with the command and your options. Choosing **"Yes, and don't ask again"** saves a rule to `.claude/settings.local.json`, permanently per repository and command. You can also press `Tab` at a permission prompt to add a comment alongside your answer.

To review what you've accumulated:

```text
/permissions
```

That opens a dialog where you can view rules by scope, add or remove them, manage working directories, and review recent auto mode denials. It also has an **Auto mode** tab for the classifier's rules.

### 8.2 Working Knowledge: the six modes

Each mode trades convenience for oversight ([permission modes](https://code.claude.com/docs/en/permission-modes)):

| Mode | Runs without asking | Best for |
|---|---|---|
| `default` (**"Manual"** in the UI) | Reads only | Reviewing every action yourself, sensitive work |
| `acceptEdits` | Reads, file edits, and common filesystem commands (`mkdir`, `touch`, `rm`, `rmdir`, `mv`, `cp`, `sed`) | Iterating on code you're reviewing after the fact |
| `plan` | Reads, plus classifier-approved commands when auto mode is available | Exploring before changing anything |
| `auto` | Everything, with background safety checks | Long tasks, reducing prompt fatigue |
| `dontAsk` | Only pre-approved tools | Locked-down CI and scripts |
| `bypassPermissions` | Everything | Isolated containers and VMs **only** |

> **The naming trap.** The mode that reviews every action is called **Manual** in the CLI, `claude --help`, the VS Code and JetBrains extensions, and the desktop app — but its config value is **`default`**, which is what hooks, settings files, and SDK integrations use. Older guides call it "default mode" and describe it as *the* default, which is no longer true on Pro/Max/Team.

**Which mode a session starts in.** On **Pro, Max, and Team plans, auto mode is the built-in starting mode** for interactive terminal and VS Code sessions. On other plans it's Manual. A `permissions.defaultMode` in settings can change this — but **`auto` and `bypassPermissions` don't take effect from project or local settings**, only from user, `--settings`, or managed settings. That restriction exists so a repository you clone can't start your sessions permissively.

**`claude -p` starts in Manual on every plan**, regardless. Pass the mode you want explicitly.

**The `Shift+Tab` cycle** is not all six modes:

```text
   from auto ──▶ default ──▶ acceptEdits ──▶ plan ──┐
                    ▲                                │
                    └────────────────────────────────┘

   Optional modes slot in after `plan`:
     bypassPermissions  — only if you started with it enabled
     auto               — when auto mode is available
   (with both enabled, you cycle through bypassPermissions on the way to auto)

   dontAsk NEVER appears in the cycle — set it with --permission-mode dontAsk
```

**Entering `bypassPermissions` is deliberately one-way-in.** You cannot enter it from a session you started without it. Enable it at launch:

```bash
claude --permission-mode bypassPermissions      # or --dangerously-skip-permissions
claude --permission-mode plan --allow-dangerously-skip-permissions   # add it to the cycle without starting in it
```

Claude Code refuses this mode when running as root or under `sudo` on Linux and macOS (skipped inside a recognised sandbox), refuses it in a `--restricted` session (v2.1.248+), and shows a one-time responsibility warning the first time. Claude Code on the web ignores `defaultMode: "bypassPermissions"` and `"dontAsk"` from settings files entirely.

**Actions that no mode auto-approves** — including `bypassPermissions`:

- Tools matched by an explicit **ask rule**
- Connector tools your organisation set to `ask`
- Tools requiring user interaction: `AskUserQuestion`, and MCP tools marked `requiresUserInteraction`
- `rm`/`rmdir` targeting a [critical path](#87-mastery-protected-and-critical-paths) — no allow rule and no `PreToolUse` hook can approve these
- The cross-session messaging safeguards
- Reads outside working directories while `permissions.blockReadsOutsideWorkingDirectories` is on

**Common setups**, straight from the docs:

| You want to | Start with |
|---|---|
| Review every action | `claude --permission-mode default` |
| Fewer prompts locally, no classifier | Manual mode **plus** the Bash sandbox in auto-allow: `claude --permission-mode default`, then `/sandbox` → auto-allow |
| Explore before changing anything | `claude --permission-mode plan` |
| Work hands-off | `claude --permission-mode auto` |
| Run in CI with an exact allowlist | `claude -p "run the test suite" --permission-mode dontAsk --allowedTools "Bash(npm test)" "Read"` |
| Run fully unattended in a container | `claude -p "<prompt>" --dangerously-skip-permissions` — **requires real isolation** |

**Plan mode's specifics.** Claude reads, runs exploratory commands, and writes a plan, but does not edit your source. `Ctrl+G` opens the plan in your editor. Accepting a plan names the session after it. Set it as a project default with `defaultMode: "plan"` in `.claude/settings.json` (this one *does* work from project settings). One caveat: in sessions where bypass permissions are available, plan mode's blocks are **not enforced** — Claude is still instructed to plan without editing, but an edit it makes anyway will go through.

### 8.3 Working Knowledge: rules — allow, ask, deny

Rules are the exceptions layered on top of a mode. Format: `Tool` or `Tool(specifier)`. Parentheses inside a specifier are literal and need no escaping.

```json
{
  "permissions": {
    "allow": ["Bash(npm run *)", "Bash(git commit *)", "Read"],
    "ask":   ["Bash(git push *)"],
    "deny":  ["Read(./.env)", "Read(./secrets/**)", "Bash(curl *)"]
  }
}
```

| Rule shape | Matches |
|---|---|
| `Bash` | All Bash commands (`Bash(*)` is equivalent) |
| `Bash(npm run build)` | That exact command |
| `Read(./.env)` | Reading `.env` in the current directory |
| `WebFetch(domain:example.com)` | Fetches to that domain |

**Deny beats everything.** Deny rules block in every mode, including `bypassPermissions`. As a deny rule, a bare tool name removes the tool from Claude's context entirely.

**Deny and ask rules can also match an input parameter** on any built-in tool, with `Tool(param:value)`:

| Rule | Matches |
|---|---|
| `Agent(model:opus)` | Agent calls requesting the Opus tier |
| `Agent(isolation:worktree)` | Agent calls requesting a git worktree |
| `Bash(run_in_background:true)` | Backgrounded Bash calls |

Parameter matching only works on **direct** fields of the tool's input (not nested ones), one parameter per rule, and compares against the **literal input Claude sends, before normalisation** — so `Agent(model:opus)` matches the alias `opus` but not a full model ID. A parameter the model omits is never matched, so `Agent(model:*)` doesn't match a call that leaves `model` unset. Run with `--verbose` to see exact parameter names. You **cannot** match a tool's primary content field this way (`command`, `file_path`, `path`, `url`) — use the tool's own specifier syntax for those.

**Tool-name wildcards** work in deny and ask rules: `"*"` matches every tool, `"mcp__*"` matches every MCP tool. Allow rules accept tool-name globs only after a literal `mcp__<server>__` prefix, so the server segment must be glob-free. A deny or ask rule naming an unknown tool produces a **startup warning** to catch typos (names containing `_` or `*` are exempt).

> One naming detail: the label shown in the transcript can differ from the canonical tool name. The tool labelled `Stop Task` is canonically `TaskStop`. Rules and hook matchers use the canonical name.

<a id="p8-bash"></a>

### 8.4 Advanced: Bash rules and the wildcard traps

Bash rules match the **whole command text**, with `*` standing in for any text including spaces. This is where most people write a rule that doesn't do what they think.

**The space before a trailing `*` is part of the rule:**

| You write | Matches | Doesn't match |
|---|---|---|
| `Bash(npm run build)` | `npm run build` | `npm run build --watch` |
| `Bash(npm run *)` | `npm run build`, `npm run test --watch`, **`npm run`** | `npm install` |
| `Bash(ls *)` | `ls -la`, **`ls`** | `lsof` |
| `Bash(ls*)` | `ls -la`, **`lsof`** | |
| `Bash(git log * main)` | `git log --oneline main`, `git log -5 main` | `git log main`, `git push origin main` |
| `Bash(git * main)` | `git merge main`, **`git push origin main`**, `git -c core.fsmonitor=<script> diff main` | `git log` |
| `Bash(* --version)` | `node --version` | `node -v` |
| `Bash(* --help *)` | `npm --help x` | `npm --help` |

Three rules produce that table:

1. **`*` stands in for whatever text is in its place** — including subcommands and the options before them. `Bash(git * main)` therefore matches `git push origin main` *and* `git -c core.fsmonitor=<script> diff main`, which makes git run a program you named.
2. **A trailing `* ` with a space before it also matches the bare command** — but only when it's the rule's *only* wildcard.
3. **The space is significant.** `Bash(ls *)` requires a space, so `lsof` doesn't match; `Bash(ls*)` has no space, so it does.

The `:*` suffix is equivalent to a trailing ` *`, so `Bash(ls:*)` matches the same as `Bash(ls *)`. The permission dialog writes the **space-separated** form when you accept a prefix. `:*` is only recognised at the *end* of a pattern — in `Bash(git:* push)` the colon is a literal.

> ⚠️ **Put the `*` after the subcommand.** In `git log --oneline main`, `git` is the program and `log` is the subcommand — the word that determines what it does. Claude Code matches everything before the first `*` as written, so `Bash(git * main)` gives away every git subcommand.

**Compound commands.** Claude Code is shell-aware, so `Bash(safe-cmd *)` won't approve `safe-cmd && other-cmd`. Recognised separators: `&&`, `||`, `;`, `|`, `|&`, `&`, and newlines. Deny and ask rules apply when **any** subcommand matches, including inside subshells, command substitution, and control-flow bodies. When `&&` or `||` has nothing after it (`npm test &&`), the command is treated as unparseable and isn't split for allow-rule matching, so `Bash(npm *)` won't approve it. Approving a compound command with "don't ask again" saves a **separate rule per subcommand**.

**Wrappers are stripped before matching**, so `Bash(npm test *)` also matches `timeout 30 npm test`. The stripped set: `timeout`, `time`, `nice`, `nohup`, `stdbuf`, plus shell builtins, plus bare `xargs` (only when it has no flags — `xargs -n1 grep pattern` matches as an `xargs` command). A leading assignment of certain known-safe environment variables is also stripped, so `Bash(npm test *)` matches `NODE_ENV=test npm test`; an allow rule won't match past an assignment of any *other* variable.

**The wrapper list is not configurable and does not include** `direnv exec`, `devbox run`, `mise exec`, `npx`, or `docker exec` — because these execute their arguments as a command, a rule for one of them grants whatever it runs. Exec wrappers like `watch`, `setsid`, `ionice`, and `flock` can't be auto-approved by a prefix rule at all, and neither can `find` with `-exec` or `-delete`.

**Read-only commands** are a built-in set Claude Code runs without prompting in every mode. Unquoted globs are permitted for commands whose every flag is read-only (`ls *.ts`, `wc -l src/*.py`). They still prompt in Manual mode when: an unquoted glob appears with a write- or exec-capable flag (`find`, `sort`, `sed`, `git`); `docker` carries a flag selecting another daemon (`-H`, `--context`); `file` uses `-m`/`-f`; a Windows UNC path appears; or the command is unparseable or over 10,000 characters.

**Redirections are checked as file operations.** `> file`, `>> file`, `2> file` are checked against your `Edit` rules, protected paths, and working directories; `< file` against your `Read` rules. Targets with no file behind them (`/dev/null`, `2>&1`, here-docs) aren't checked.

> **Wrong vs. right — trying to restrict `curl` to one domain.**
>
> ```json
> // Wrong — fragile in at least five ways
> { "permissions": { "allow": ["Bash(curl http://github.com/ *)"] } }
> ```
>
> It doesn't match `curl -X GET http://github.com/…` (option before the URL), `https://…` (different protocol), `curl -L http://short.example.com/xyz` (redirects to GitHub), `URL=http://github.com && curl $URL` (variable), or `curl  http://github.com` (extra space).
>
> ```json
> // Right — deny the shell network tools, allow the tool that can be scoped
> {
>   "permissions": {
>     "deny":  ["Bash(curl *)", "Bash(wget *)"],
>     "allow": ["WebFetch(domain:github.com)"]
>   }
> }
> ```
>
> The docs are blunt about the general principle: **Bash permission patterns that try to constrain command arguments are fragile.** For real URL filtering use a `PreToolUse` hook that validates URLs, or the sandbox's network allowlist. And note the asymmetry: allowing WebFetch alone doesn't prevent network access — if Bash is allowed, Claude can still reach any URL with `curl`.

<a id="p8-readedit"></a>

### 8.5 Advanced: Read and Edit rules — gitignore syntax with anchors

`Read` and `Edit` rules use [gitignore](https://git-scm.com/docs/gitignore) pattern syntax with **four anchor types**, and the anchor is where people go wrong:

| Pattern | Means | Example | Resolves to |
|---|---|---|---|
| `//path` | **Absolute** from filesystem root | `Read(//Users/alice/secrets/**)` | `/Users/alice/secrets/**` |
| `~/path` | From home directory | `Read(~/Documents/*.pdf)` | `/Users/alice/Documents/*.pdf` |
| `/path` | Relative to the **settings source** | `Edit(/src/**/*.ts)` | `<primary working dir>/src/**/*.ts` in project settings |
| `path` or `./path` | Relative to current directory | `Read(*.env)` | `<cwd>/*.env` |

> ⚠️ **`/Users/alice/file` is not an absolute path.** A single leading slash anchors at the settings source, not the filesystem root. Use `//Users/alice/file`.

Where `/path` anchors depends entirely on which file defines the rule:

| Rule defined in | `/path` resolves to |
|---|---|
| `.claude/settings.json` | `<primary working directory>/path` |
| `.claude/settings.local.json` | `<primary working directory>/path` |
| `~/.claude/settings.json` | **`~/.claude/path`** |
| A file passed with `--settings <file>` | `<directory of that file>/path` |
| CLI flags or session rules | `<primary working directory>/path` |

That third row catches people constantly: `Read(/secrets/**)` in **user** settings blocks `~/.claude/secrets/**`, not a `secrets` directory in your project. To write a user-settings rule that applies inside every project, use a `//` absolute path or a bare relative pattern.

**Matching depth differs by rule type** for single-segment relative directory patterns:

| Rule | Matches `src/app.ts` | Matches `vendor/pkg/src/lib.js` |
|---|---|---|
| `Edit(src/**)` as an **allow** rule | Yes | **No** |
| `Edit(src/**)` as a **deny or ask** rule | Yes | **Yes** |
| `Edit(/src/**)` in any rule type | Yes | No |
| `Edit(**/src/**)` in any rule type | Yes | Yes |

The asymmetry is deliberate and it's the safe direction: **deny rules match a directory of that name at any depth**, so `Read(secrets/**)` also covers nested copies; **allow rules match only at the anchored location**, so you can't accidentally widen access to a vendored directory.

Similarly, `Read(.env)` and `Read(**/.env)` as deny rules block any `.env` at or under the current directory — but **not** one in a parent directory or another project. `Read(//**/.env)` blocks any `.env` anywhere on the filesystem.

**Cross-tool coverage, which is more subtle than it looks:**

- `Edit` rules apply to **all** built-in file-editing tools. `Read` rules are applied best-effort to all reading tools (Grep, Glob), to `@file` mentions in your prompts, and to editor selections.
- A `Read` **deny** rule also blocks Edit and Write on the same path, including creating a new file there. **NotebookEdit isn't covered** — add an `Edit` deny rule for notebook paths.
- Claude Code checks file permissions against `Edit(path)` and `Read(path)` rules **only**. A path rule written for `Write`, `NotebookEdit`, `Glob`, or the legacy `MultiEdit` is accepted but never consulted.
- Read and Edit deny rules also apply to **file commands Claude Code recognises in Bash** (`cat`, `head`, `tail`, `sed`) and to Bash redirection targets.

**Symlinks are checked as a pair** — the link path and its target — and the two rule types treat that pair differently:

- **Allow rules** apply only when **both** match. A symlink inside an allowed directory pointing outside it still prompts.
- **Deny rules** apply when **either** matches. A symlink pointing to a denied file is itself denied.

With `Read(./project/**)` allowed and `Read(~/.ssh/**)` denied, a symlink at `./project/key` → `~/.ssh/id_rsa` is blocked: it fails the allow rule and matches the deny rule. Claude Code also [re-confirms the path still resolves where the permission check approved](https://code.claude.com/docs/en/errors#refusing-after-a-symlink-changed) when a tool actually opens the file.

**Windows paths** are normalised to POSIX before matching: `C:\Users\alice` becomes `/c/Users/alice`. Use `//c/**/.env` for one drive, `//**/.env` across all drives.

When you approve a path with "don't ask again", Claude Code **escapes** gitignore special characters (`[`, `]`, `*`) so the generated rule matches only the literal path. Parentheses need no escaping — `Edit(./Finance (2024)/**)` works as spelled.

<a id="p8-other-rules"></a>

### 8.6 Advanced: WebFetch, MCP, Agent, and Cd rules

**WebFetch** rules use a `domain:` prefix, matched case-insensitively against the hostname:

- `WebFetch(domain:example.com)` — that host
- `WebFetch(domain:*.example.com)` — any subdomain at any depth, but **not** `example.com` itself
- `WebFetch(domain:example.*)` — matches `example.org` (the `*` becomes one label between dots), but **not** `example.evil.com`

The bare-vs-`domain:*` distinction matters because only one of them touches the sandbox:

| Rule | In `allow` | In `deny` |
|---|---|---|
| `WebFetch` | Claude fetches without prompting. **Doesn't change** which hosts sandboxed commands can reach | Removes the WebFetch tool entirely. Doesn't change the sandbox |
| `WebFetch(domain:*)` | Claude fetches without prompting, **and sandboxed commands can reach any host** | Tool is kept but each fetch is refused, **and sandboxed commands can reach nothing** |

Use the **bare** form to let Claude fetch freely while leaving the sandbox allowlist as it is.

**MCP** rules use the server name as configured:

- `mcp__puppeteer` — any tool from that server
- `mcp__puppeteer__*` — same, wildcard form
- `mcp__puppeteer__puppeteer_navigate` — that one tool

Claude Code **skips any `mcp__` rule that has parentheses** when loading a settings file, listing the skipped rule. To match a *parameter* on an MCP tool, pass a deny rule via `--disallowedTools`. If your organisation set a claude.ai connector tool to `ask`, allow rules for it don't take effect.

**Agent** rules control which subagents Claude can use: `Agent(Explore)`, `Agent(Plan)`, `Agent(my-custom-agent)`.

```json
{ "permissions": { "deny": ["Agent(Explore)"] } }
```

**Cd** rules control which directories `/cd` can move the session to. `Cd` is **not model-invocable** — Claude can't call it, and the rules apply only when you run `/cd` yourself. A bare `Cd` deny disables `/cd`. **Adding any `Cd` allow rule switches `/cd` to allowlist mode.** Matching is anchored to the whole directory path rather than gitignore-style: `*` matches exactly one segment, `**` matches many.

| Rule | Matches | Doesn't match |
|---|---|---|
| `Cd(~/code/*)` | `~/code/app` | `~/code/app/src`, `~/code` |
| `Cd(~/code/**)` | `~/code` and anything under it | Anything outside `~/code` |
| `Cd(**/node_modules)` | Any `node_modules` at any depth | `node_modules/pkg` |

**Working directories.** Claude reads and edits within the directory you launched from, plus anything added with `--add-dir`, `/add-dir`, or `permissions.additionalDirectories`. Two things to know: **additional directories grant file access, not configuration** — most `.claude/` config isn't discovered from them (see [Part 5](./claude-code-context-memory.md) for the memory-loading exception), and `/add-dir` runs your `DirectoryAdded` hooks on success.

**Hooks extend the permission system.** `PreToolUse` hooks run **before** the permission prompt for every tool, and can allow, deny, or block. That makes them the right tool for anything rule syntax can't express — URL validation, content inspection, time-of-day restrictions. See [Part 12](./claude-code-extensibility.md).

<a id="87-mastery-protected-and-critical-paths"></a>

### 8.7 Mastery: protected paths, critical paths, and workspace trust

Two categories of path get special handling that **your allow rules cannot override**. Knowing them is the difference between "I configured permissions" and "I understand the permission model."

#### Protected paths

Writes to these are never auto-approved, except in `bypassPermissions` mode and in plan-mode sessions where bypass permissions are available:

```text
  Directories                          Files
  ───────────                          ─────
  .git                                 .gitconfig, .gitmodules
  .config/git                          .bashrc, .bash_profile, .bash_login,
  .vscode                                .bash_aliases, .bash_logout, .zshrc,
  .idea                                  .zprofile, .zshenv, .zlogin, .zlogout,
  .husky                                 .profile, .envrc
  .cargo                               .npmrc, .yarnrc, .yarnrc.yml, .pnp.cjs,
  .devcontainer                          .pnp.loader.mjs, .pnpmfile.cjs,
  .yarn                                  bunfig.toml, .bunfig.toml
  .mvn                                 .bazelrc, .bazelversion, .bazeliskrc
  .claude                              .pre-commit-config.yaml, lefthook.{yml,yaml},
    (except .claude/worktrees)           .lefthook.{yml,yaml}
                                       gradle-wrapper.properties,
                                         maven-wrapper.properties
                                       .devcontainer.json
                                       .ripgreprc, pyrightconfig.json
                                       .mcp.json, .claude.json
```

The unifying logic: **every one of these is a file that can cause code to execute, or that changes what Claude Code itself trusts.** A `.envrc`, a git hook, a `.pnpmfile.cjs`, a `.claude/settings.json` — writing to any of them is a privilege escalation, not an edit.

| Mode | Protected-path writes |
|---|---|
| `default`, `acceptEdits` | Prompted |
| `plan` | Allowed with bypass available; otherwise routed to the classifier when auto mode is available |
| `auto` | Routed to the classifier |
| `dontAsk` | Denied |
| `bypassPermissions` | Allowed |

**`permissions.allow` rules do not pre-approve protected-path writes.** The safety check runs *before* allow rules are evaluated, so an `Edit(.claude/**)` entry doesn't help. In a `--restricted` session the classifier can't approve them either.

#### Critical paths

No allow rule and no `PreToolUse` hook returning `"allow"` can approve an `rm` or `rmdir` targeting a critical path. These are:

- The filesystem root, and any **direct child** of it (`/usr`, `/etc`, `/data`)
- Your home directory
- Windows drive roots and their top-level directories (`C:\`, `C:\Windows`)
- Your working directory and its **parents**
- Your additional working directories and their parents — but only when the removal is a **glob** under one of them (`rm -rf <dir>/*`); `rm -rf <dir>` on the directory itself doesn't trigger this

Also treated as critical: a glob or trailing slash directly under a shell variable, such as `rm -rf "$DIR"/*` — because if the variable is empty, that command becomes a removal from the filesystem root. Hiding a removal inside `$(...)`, backticks, or `<(...)` does not skip the check.

| Mode | Critical-path removal |
|---|---|
| `default`, `acceptEdits` | Asks you |
| `plan` | Asks; with auto mode available during planning and no bypass, goes to the classifier |
| `auto` | Goes to the classifier |
| `dontAsk` | Denied |
| `bypassPermissions` | **Asks you** — one of the few things this mode still stops for |

#### Workspace trust

Project `allow` rules — and hooks, and `headersHelper` scripts, and `autoMemoryDirectory` in project settings — **wait until you trust the folder**. This is what stops a repository you clone from configuring your agent before you've looked at it.

The critical asymmetry, and the thing to know before running Claude Code on unfamiliar code:

> **A `claude -p` session shows no workspace trust dialog and no per-server approval prompt.** Without `--bare`, it runs the hooks in a project's `.claude/settings.json` and connects the servers in its `.mcp.json`, in a folder you have never trusted ([headless](https://code.claude.com/docs/en/headless)).

That is the single strongest argument for `--bare` in CI, and for reading a repo's `.claude/` directory before pointing a headless run at it.

Rules you approve in a **worktree** are saved to the **main checkout's** `.claude/settings.local.json` (v2.1.211+), so they apply across all worktrees of the repository and survive the worktree's removal.

> ### Real Scenario — the allow rule that gave away the repository
>
> A team wants Claude to stop asking about git operations, so someone adds what looks like a reasonable rule to the shared `.claude/settings.json`:
>
> ```json
> { "permissions": { "allow": ["Bash(git * main)"] } }
> ```
>
> The reasoning: "only commands that end in `main`, so it's scoped."
>
> Three weeks later, an agent working from a GitHub issue that contained an embedded instruction (see [§9.5](#95-advanced-prompt-injection-as-the-real-threat-model)) runs `git push --force origin main`. It executes without a prompt.
>
> **What happened:** `*` stands in for the subcommand *and every option before it*. `Bash(git * main)` matches `git push origin main`, `git reset --hard main`, and `git -c core.fsmonitor=<arbitrary-script> diff main` — which makes git execute a program the agent named. The rule granted essentially all of git.
>
> **What saved them partially:** the session was in auto mode, and the classifier blocks force push by default regardless of allow rules — but the developer had switched to `acceptEdits` earlier that day to reduce prompts, which took the classifier out of the loop.
>
> **The fix:**
>
> ```json
> {
>   "permissions": {
>     "allow": ["Bash(git status)", "Bash(git diff *)", "Bash(git log *)",
>               "Bash(git add *)", "Bash(git commit *)"],
>     "ask":   ["Bash(git push *)"],
>     "deny":  ["Bash(git push --force *)", "Bash(git reset --hard *)"]
>   }
> }
> ```
>
> **The lessons, in order of importance:** put the `*` *after* the subcommand, never before it; an `ask` rule forces a prompt **even in auto mode**, which makes it the right tool for irreversible operations; and a permission rule is a thing you should read as an attacker would, because eventually something hostile will be reading it that way.

<a id="88-part-8-cheat-sheet"></a>

### 8.8 Part 8 cheat sheet

| Mode | Config value | Auto-approves |
|---|---|---|
| Manual | `default` | Reads only |
| Accept edits | `acceptEdits` | + edits, `mkdir`/`touch`/`rm`/`rmdir`/`mv`/`cp`/`sed` |
| Plan | `plan` | Reads + classifier-approved commands |
| Auto | `auto` | Everything, classifier-reviewed |
| Don't ask | `dontAsk` | Only pre-approved tools |
| Bypass | `bypassPermissions` | Everything |

| Rule syntax | Notes |
|---|---|
| `Bash(cmd *)` | Space before `*` — also matches bare `cmd`; put `*` **after** the subcommand |
| `Bash(cmd*)` | No space — also matches `cmdfoo` |
| `Bash(cmd:*)` | Equivalent to `Bash(cmd *)`; only recognised at the end |
| `Read(//abs/path)` | `//` = filesystem root; `/` = settings source; `~/` = home; bare = cwd |
| `Edit(src/**)` | Allow: only at anchor. Deny/ask: **any depth** |
| `WebFetch(domain:*.example.com)` | Subdomains only, not the apex |
| `WebFetch` vs `WebFetch(domain:*)` | Bare doesn't touch the sandbox; `domain:*` does |
| `mcp__server__tool` | No parentheses — rules with them are skipped |
| `Agent(name)` · `Cd(path)` | Subagent gating · `/cd` gating (any allow rule = allowlist mode) |
| `Tool(param:value)` | Deny/ask only; direct fields only; literal pre-normalised values |

| Never auto-approved, any mode |
|---|
| Explicit `ask` rules · `AskUserQuestion` · `requiresUserInteraction` MCP tools |
| `rm`/`rmdir` on a critical path (no allow rule, no hook `"allow"`) |
| Protected-path writes (except `bypassPermissions`) |
| Cross-session messaging safeguards |

| Command | Effect |
|---|---|
| `Shift+Tab` | Cycle modes |
| `/permissions` | Rules by scope, working dirs, recent auto-mode denials, auto-mode rules |
| `--allowedTools` / `--disallowedTools` | Session allow / deny |
| `--permission-mode <mode>` | Start in a mode |
| `--dangerously-skip-permissions` | `bypassPermissions` |
| `--allow-dangerously-skip-permissions` | Add bypass to the cycle without starting in it |
| `--restricted` | Refuse bypass; classifier can't approve protected paths (v2.1.248+) |

[↑ Back to top](#table-of-contents)

---

## Part 9 — Containment

<a id="part-9"></a>

*This Part runs Beginner → Mastery. Its Beginner tier is short because containment is inherently an intermediate topic — but the layering diagram belongs at the top, not buried.*

### 9.1 Beginner: three layers, not one

Most people think of Claude Code's safety as one thing. It's three independent layers that compose:

```text
  ┌───────────────────────────────────────────────────────────────┐
  │  1. PERMISSION MODE — does this run without asking me?        │
  │     Manual · acceptEdits · plan · auto · dontAsk · bypass      │
  │     Evaluated per tool call, before anything runs.             │
  └───────────────────────────────────────────────────────────────┘
                              │
  ┌───────────────────────────▼───────────────────────────────────┐
  │  2. PERMISSION RULES — is this specific call pre-decided?     │
  │     allow / ask / deny, per tool, per specifier                │
  │     Deny wins everywhere, including bypassPermissions.         │
  └───────────────────────────────────────────────────────────────┘
                              │
  ┌───────────────────────────▼───────────────────────────────────┐
  │  3. SANDBOX — once it runs, what can it actually reach?       │
  │     OS-level filesystem and network isolation, Bash only.      │
  │     Enforced by the kernel, not by reading the command string. │
  └───────────────────────────────────────────────────────────────┘

  Plus: checkpoints (undo file edits) and worktrees (isolate the blast radius).
```

Layers 1 and 2 make a decision by **reading the command text** and, in auto mode, asking a classifier what it means. Layer 3 makes no decision at all — it just makes certain things impossible.

That distinction matters: a permission rule can be fooled by a command it can't parse. The sandbox cannot, because it isn't parsing anything.

### 9.2 Working Knowledge: auto mode and its classifier

In auto mode, a **separate classifier model** reviews actions before they run, blocking anything that escalates beyond your request, targets unrecognised infrastructure, or appears driven by hostile content ([permission modes](https://code.claude.com/docs/en/permission-modes)).

It also reviews every message Claude sends to another agent with `SendMessage`, and it reviews `rm`/`rmdir` removals targeting critical paths.

**What it blocks by default:**

- Downloading and executing code, like `curl | bash`
- Sending sensitive data to external endpoints
- Production deploys and migrations
- Mass deletion on cloud storage
- Granting IAM or repo permissions
- Modifying shared infrastructure
- Irreversibly destroying files that existed before the session
- **Force push**
- Committing or pushing a change that would send secrets outside the repository when it runs, or widen what a deploy exposes
- `git reset --hard`, `git checkout -- .`, `git restore .`, `git clean -fd`, `git stash drop`, `git stash clear` — presumed to discard uncommitted changes
- `git commit --amend` when HEAD wasn't created in this session, or (from v2.1.198) when HEAD has already been pushed. A message-only reword of a commit Claude made this session is not blocked
- `terraform destroy`, `pulumi destroy`, `cdk destroy`, `terragrunt destroy`, and applying a plan that destroys resources

v2.1.195+ added more categories, several depending on `environment` entries you define in [auto mode config](https://code.claude.com/docs/en/auto-mode-config): writing to a secret manager; changing DNS records or TLS certificates; merging a PR no human approved, approving Claude's own PR, or disabling CI checks; posting a comment that is itself a command to automation (`atlantis apply`, a bot's `/deploy`); and toggling, ramping, or deleting a production feature flag. v2.1.257 added a Containment Escape rule.

**What it trusts:** your working directory, and the remotes configured for it **when the session started**. A remote added or repointed mid-session with `git remote add` or `git remote set-url` is **not** trusted.

**Availability requirements** — check these when auto mode is reported unavailable:

- **Plan**: all plans
- **Organization**: available by default on Team and Enterprise; admins can disable with `permissions.disableAutoMode: "disable"` in managed settings
- **Model**: on the Anthropic API and Claude Platform on AWS — Opus 4.6+, Sonnet 4.6+, or a Fable model
- **Provider**: Anthropic API, Claude Platform on AWS, Bedrock, Google Cloud's Agent Platform, Microsoft Foundry, signed-in Claude apps gateway sessions

> In v2.1.158 through v2.1.206, auto mode was **off** on Bedrock/Agent Platform/Foundry until you set `CLAUDE_CODE_ENABLE_AUTO_MODE=1`, and `defaultMode: "auto"` was ignored there without it. That variable is no longer needed — if you find it in a config, it's a leftover.

**A message naming a model and saying auto mode "cannot determine the safety" of an action** means a classifier request failed. That's usually transient.

> ⚠️ **The docs' own warning, worth quoting:** *"Auto mode reduces permission prompts but does not guarantee safety. Use it for tasks where you trust the general direction, not as a replacement for review on sensitive operations."*

You can extend the classifier with your own rules via the `autoMode` setting, and view or edit them in `/permissions` → **Auto mode**. `claude auto-mode defaults` prints the built-in rules as JSON; `claude auto-mode reset` restores them.

### 9.3 Working Knowledge: the Bash sandbox

The sandbox lets Claude run most shell commands **without asking**, because the operating system — not a pattern match — enforces what those commands can touch ([sandboxing](https://code.claude.com/docs/en/sandboxing)).

```text
/sandbox
```

That opens a panel with Mode, filesystem, and network tabs, plus a Dependencies tab on Linux when the optional seccomp filter is missing.

**Platform support: macOS, Linux, and WSL2.** Native Windows is **not** supported — run Claude Code inside a WSL2 distribution. WSL1 doesn't work because bubblewrap needs kernel features it lacks.

| Platform | Mechanism |
|---|---|
| macOS | Seatbelt (built in, nothing to install) |
| Linux | [bubblewrap](https://github.com/containers/bubblewrap) |
| WSL2 | bubblewrap |

**Two sandbox modes**, differing only in approval, not in what's enforced:

- **Auto-allow mode** — when a command *can* be sandboxed, Claude Code runs it inside the sandbox and approves it automatically. Commands that can't be sandboxed fall back to the regular permission flow.
- **Regular permissions mode** — all Bash commands go through the normal flow, even when sandboxed. More control, more approvals.

Even in auto-allow mode, these still apply: explicit **deny** rules; `rm`/`rmdir` on critical paths; and content-scoped **ask** rules like `Bash(git push *)`. A *bare* `Bash` ask rule (or `Bash(*)`) is skipped for sandboxed commands but still applies to those that fall back.

**Auto-allow mode is independent of your permission mode**, with one exception: in **plan mode** it doesn't widen approvals.

> **This is the combination worth knowing about:** Manual mode **plus** the sandbox in auto-allow gives you far fewer prompts than Manual alone, without a classifier making judgement calls on your behalf. If auto mode isn't available to you, or you don't want it, this is the setup to reach for.

**The unsandboxed retry escape hatch.** Some commands can't run inside the sandbox at all. Claude Code reports the violation in the result, naming what was blocked, and Claude may retry outside the sandbox — where the command goes through the **regular** permission flow (a prompt in Manual mode). Disable it with `"allowUnsandboxedCommands": false`.

**Shell mode is not sandboxed.** Commands you type at the `!` prompt run outside the sandbox, because the sandbox applies to commands *Claude* runs. Two exceptions where strict sandbox mode covers them too: a background session, and a Linux session with `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` set. (Before v2.1.260, strict mode sandboxed shell-mode commands in every session.)

<a id="p9-sandbox-policy"></a>

### 9.4 Advanced: sandbox filesystem and network policy

**Default filesystem behaviour:**

- **Write**: the current working directory and its subdirectories, directories added with `--add-dir`/`/add-dir`/`permissions.additionalDirectories`, and the session temp directory (`$TMPDIR` is set to it).
- **Read**: the **entire computer**, minus certain denied directories. Note what that means — **`~/.aws/credentials` and `~/.ssh/` are readable by default.** If that matters to you, configure `denyRead`.
- **Blocked**: modifying anything outside those write paths, including `~/.bashrc` and system files.
- **Worktrees**: writes to the main repository's shared `.git` directory are allowed, so `git commit` works from inside a worktree.

```json
{
  "sandbox": {
    "enabled": true,
    "filesystem": {
      "allowWrite": ["~/.kube", "/tmp/build"]
    }
  }
}
```

> ⚠️ **Sandbox path prefixes are NOT the same as Read/Edit rule prefixes.** Sandbox paths use standard conventions: `/` is absolute from the filesystem root, `~/` is home, and `./` or no prefix is relative to the project root (for project settings) or `~/.claude` (for user settings). Read and Edit rules use `//` for absolute and `/` for settings-source-relative. Mixing them up silently produces a rule that points somewhere you didn't intend.

**Deny/allow interaction** — the narrower rule wins in the safe direction:

| Rules | Result |
|---|---|
| `"denyRead": ["~/"]` + `"allowRead": ["~/projects"]` | `~/projects` readable, rest of home blocked — a narrower allow re-opens part of a denied region |
| `"allowRead": ["~/"]` + `"denyRead": ["~/.env"]` | `~/.env` blocked, rest readable — **a deny holds inside a wider allow**, so a broad allow can't silently re-expose a secret |
| `"allowRead": ["~/"]` + `"denyRead": ["~/**/.env"]` | Every `.env` under home blocked, rest readable |

Blocking home-directory reads while keeping the project readable, in **project** settings (because `.` resolves to the project root there):

```json
{
  "sandbox": {
    "enabled": true,
    "filesystem": { "denyRead": ["~/"], "allowRead": ["."] }
  }
}
```

Put the same thing in `~/.claude/settings.json` and `.` resolves to `~/.claude` instead, leaving your project blocked.

Filesystem arrays **merge across settings scopes**, and edits during a session apply to the running session — the next sandboxed command uses the new paths.

**`sandbox.filesystem.disabled: true`** skips filesystem isolation while keeping network isolation.

**Sandbox protected paths.** Inside the directories sandboxed commands *can* write to, the sandbox still denies writes to the files Claude Code loads configuration and code from — because a command that edits those could grant itself permissions:

- In your working directory **and above**: `.claude` settings files, `.claude/skills`, `.claude/agents`, `.claude/commands`, `.claude/hooks`, `.mcp.json`
- In your working directory **only**: `.bashrc`/`.zshrc`, `.gitconfig`, `.vscode`, `.idea`, and `hooks`/`config` inside `.git`
- Files that would turn your working directory into a bare git repo: `HEAD`, `objects`, `refs`, plus `config`/`hooks` when they exist
- In `~/.claude` (or `CLAUDE_CONFIG_DIR`): most contents, plus `~/.claude.json` and `.credentials.json`

**There is no way to exempt one of these.** An `allowWrite` entry or an `Edit` allow rule covering the path does not lift the protection; only `filesystem.disabled` turns it off. If a symlink appears at a protected path during the session, the sandbox denies writes to its target too, from the next command.

**Network isolation** runs through a proxy **outside** the sandbox:

- **No domains are pre-allowed.** The first time a command needs a new domain, Claude Code prompts (or asks the classifier, in auto mode).
- `network.strictAllowlist: true` (user, managed, or `--settings` only) denies access to anything not explicitly allowed, without prompting.
- `network.allowManagedDomainsOnly` in managed settings blocks non-allowed domains automatically.
- Corporate proxies: set `HTTPS_PROXY`, `HTTP_PROXY`, `NO_PROXY`.
- Restrictions cover **all scripts, programs, and subprocesses** spawned by commands.

In `WebFetch(domain:...)` rules the sandbox honours a leading `*.` and a bare `*` (v2.1.186+); a wildcard in any other position isn't honoured by the sandbox.

**IPv6 addresses must be bracketed** in domain lists: `"[::1]"`. An unbracketed entry with two or more colons is ambiguous (`::1:443` reads as both a complete address and an address-plus-port), and Claude Code resolves the ambiguity conservatively — deny lists block **every** reading, allow lists never allow more than you wrote and may drop the entry entirely. `claude doctor` warns with `Sandbox network domain entries have unreliable spellings` and names up to three offenders.

**Sandbox vs. rules vs. modes**, since all three touch filesystem and network:

| Setting or rule | What it does |
|---|---|
| `sandbox.filesystem.allowWrite` | Subprocess write access outside the working directory |
| `sandbox.filesystem.denyWrite` / `denyRead` | Block subprocess access to paths |
| `sandbox.filesystem.allowRead` | Re-allow reads within a `denyRead` region |
| `sandbox.filesystem.disabled` | Turn the filesystem layer off, keep network |
| `Edit` allow rules | Grant write access, same as `allowWrite` |
| `Read`/`Edit` deny rules | Block access to files or directories |
| `WebFetch(domain:...)` | Domain access |
| Sandbox `allowedDomains` / `deniedDomains` | Which domains Bash can reach; deny beats a broader allow wildcard |

Paths and domains from **both** sandbox settings and permission rules are merged into the final sandbox configuration.

<a id="p9-injection"></a>

### 9.5 Advanced: prompt injection as the real threat model

The failure mode that permission rules exist for is not "Claude decides to be malicious." It's **content Claude reads containing instructions it treats as yours.**

Everything Claude reads is a potential injection vector: a GitHub issue body, a PR comment, a dependency's README, a test fixture, a log line, a web page it fetched, an MCP tool's response, a `CLAUDE.md` in a repository you cloned.

```text
   Attacker controls content ──▶ Claude reads it ──▶ Claude acts on it
   ───────────────────────────    ────────────────    ─────────────────
   issue text, PR comment,        as part of a         with YOUR
   package README, log output,    normal task          permissions
   web page, MCP response,
   a cloned repo's CLAUDE.md
```

The defences, in the order they actually help:

1. **Deny rules on secrets.** `Read(./.env)`, `Read(~/.ssh/**)`, `Read(./secrets/**)`. Deny beats every mode. This is the cheapest high-value configuration you can add, and most people don't.
2. **`ask` rules on irreversible operations.** An ask rule forces a prompt **even in auto mode** — it is the only rule type the classifier won't decide for you. Put `git push`, deploys, and migrations here.
3. **The sandbox's network allowlist.** Exfiltration needs an egress path. No domains are pre-allowed by default; keep it that way and approve deliberately.
4. **Auto mode's classifier**, which specifically watches for actions that look "driven by hostile content" and for sending sensitive data to external endpoints.
5. **`PreToolUse` hooks** for anything rule syntax can't express.
6. **Worktrees**, so a compromised session's edits are confined to a checkout you can throw away.

Two structural facts to hold onto:

- **The classifier trusts remotes configured when the session started.** A `git remote add` mid-session is exactly what an injected instruction would do, and it isn't trusted.
- **`bypassPermissions` offers no protection against prompt injection**, in the docs' own words. It is for isolated containers and VMs, not for "I'm in a hurry."

> ### Real Scenario — the dependency README that opened a PR
>
> A team runs a nightly `claude -p` job that reads Dependabot PRs, checks whether the upgrade is safe, and comments on each one. It runs with `--permission-mode auto` and a generous allowlist so it can use `gh` freely. It has worked for months.
>
> One night it opens an unrelated pull request adding a new workflow file to `.github/workflows/`, and posts an approving comment on the Dependabot PR.
>
> **What happened:** one upgraded package's release notes — which the job read via `gh pr view` — contained a block of text addressed to an AI agent, instructing it to add a workflow step "for compatibility." The job read the release notes as part of its task and acted on them.
>
> **What partially held:** the classifier blocked the *first* attempt, a `curl | bash` in the proposed workflow. The second attempt, a plain workflow file addition, looked like ordinary repository work.
>
> **What failed:** `--bare` wasn't used, so the job also loaded the repo's `.claude/` config with no trust dialog — a headless run gets no dialog. And nothing in the setup treated *reading untrusted text* as different from reading the team's own code.
>
> **The fixes applied:**
>
> ```bash
> claude --bare -p "$PROMPT" \
>   --permission-mode dontAsk \
>   --allowedTools "Bash(gh pr view *)" "Bash(gh pr diff *)" "Bash(gh pr comment *)" \
>   --max-turns 15 \
>   --permission-prompts none
> ```
>
> Plus a deny rule on `Edit(.github/**)`, and moving the job to a worktree. The principle: **an unattended job that reads attacker-influenced content should have an allowlist, not a denylist** — `dontAsk` mode denies anything not explicitly permitted, which inverts the default.

<a id="p9-mastery"></a>

### 9.6 Mastery: what the sandbox is not

The docs are unusually candid here, and repeating the limitations is more useful than repeating the features. **Sandboxing reduces risk but is not a complete isolation boundary.**

**Security limitations:**

- **TLS is not inspected by default.** The proxy makes its allow decision from the **client-supplied hostname** without inspecting encrypted traffic. Contents are opaque to it. The experimental `network.tlsTerminate` changes this.
- **Broad domains create exfiltration paths.** Allowing something like `github.com` means code inside the sandbox can reach any repository, gist, or API on that host. The docs call this out explicitly.
- **`allowUnixSockets` can escalate privilege.** Allowing `/var/run/docker.sock`, for instance, hands out the ability to escape.
- **Broad filesystem writes can escalate privilege.** Write access to directories containing executables in `$PATH`, or to system configuration, is a sandbox bypass in waiting.
- **`enableWeakerNestedSandbox`** exists so the Linux sandbox works inside Docker without privileged mode. It is weaker — that's what the name says.
- **`allowAppleEvents` on macOS removes code-execution isolation.** The sandbox blocks Apple Events by default; lifting it makes `open` and `osascript` work, and gives up the boundary.

**Scope limitations** — the sandbox isolates **Bash subprocesses only**:

- **Read, Edit, and Write** use the permission system directly and do not run through the sandbox.
- **Computer use** runs on your actual desktop, not in an isolated environment.
- MCP tools operate under their own boundary.

**Choosing an isolation level.** The sandbox is one option; when you need a real boundary, go outward ([sandbox environments](https://code.claude.com/docs/en/sandbox-environments)):

```text
  weaker ─────────────────────────────────────────────────▶ stronger

  permission rules   Bash sandbox    dev container /    cloud / self-hosted
  (reads command     (OS-enforced,   VM (whole-machine   environment (someone
   strings)           Bash only)      boundary)           else's machine)
```

`--dangerously-skip-permissions` is only defensible at the third level or beyond. The [dev container](https://code.claude.com/docs/en/devcontainer) configuration runs Claude Code as a non-root user, which is why the root check is skipped inside a recognised sandbox.

**For organisations:** deliver `sandbox` keys through managed settings to require it for everyone, and use `allowManagedDomainsOnly` plus `strictAllowlist` to keep developers from widening the policy. The [claude-code repo's examples directory](https://github.com/anthropics/claude-code/tree/main/examples/settings) has starter configurations for common deployment scenarios.

<a id="97-part-9-cheat-sheet"></a>

### 9.7 Part 9 cheat sheet

| Layer | Controls | Enforced by |
|---|---|---|
| Permission mode | Whether a call runs / prompts | Claude Code, per call |
| Permission rules | Specific pre-decisions | Claude Code, before the call |
| Sandbox | What a Bash command can reach | The OS (Seatbelt / bubblewrap) |

| Sandbox setting | Effect |
|---|---|
| `sandbox.enabled` | Turn it on |
| `filesystem.allowWrite` / `denyWrite` | Write paths |
| `filesystem.denyRead` / `allowRead` | Read paths — deny holds inside a wider allow |
| `filesystem.disabled` | Skip filesystem isolation, keep network |
| `network.strictAllowlist` | Deny non-allowlisted domains without prompting |
| `network.allowManagedDomainsOnly` | Managed lockdown |
| `allowUnsandboxedCommands: false` | Remove the retry escape hatch |
| `allowUnixSockets`, `allowAppleEvents`, `enableWeakerNestedSandbox` | Each weakens the boundary — read the limitations first |

| Defence | Against |
|---|---|
| `deny: ["Read(./.env)", "Read(~/.ssh/**)"]` | Secret exfiltration; beats every mode |
| `ask: ["Bash(git push *)"]` | Irreversible ops — **prompts even in auto mode** |
| Sandbox network allowlist | Egress; nothing pre-allowed by default |
| Auto mode classifier | Hostile-content-driven actions, force push, destructive IaC |
| `PreToolUse` hook | Anything rule syntax can't express |
| `--bare` in CI | Untrusted repo hooks and MCP servers (no trust dialog in `-p`) |
| `dontAsk` + `--allowedTools` | Unattended jobs reading untrusted content |
| Worktree | Blast radius |

| Sandbox limitation |
|---|
| TLS not inspected by default — hostname-only allow decisions |
| Broad domains (`github.com`) are exfiltration paths |
| Bash subprocesses only — Read/Edit/Write use the permission system instead |
| macOS · Linux · WSL2 only; no native Windows, no WSL1 |
| Sandbox protected paths **cannot** be exempted except by disabling the filesystem layer |

[↑ Back to top](#table-of-contents)

---

**Next:** [Extensibility (Parts 10–12)](./claude-code-extensibility.md) — skills, subagents, hooks, and MCP.
