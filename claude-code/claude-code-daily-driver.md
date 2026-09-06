# Claude Code — The Daily Driver (Parts 1–3)

The loop you actually live in: running a session, steering it mid-flight, recovering when it goes wrong, and prompting in the way that produces work you don't have to redo.

> **Spec:** this doc follows the canonical spec in [`claude-code-mastery-guide.md`](./claude-code-mastery-guide.md#about-this-document) — goal-driven framing, Beginner → Working Knowledge → Advanced → Mastery tiers (collapsed where a tier would be filler, always stated), wrong-vs-right pairs, production-style Real Scenarios, official docs cited inline. Written against **Claude Code v2.1.263**, verified **September 6, 2026**.
>
> **Prerequisite:** [§4 of the overview](./claude-code-mastery-guide.md#4-primer-what-an-agentic-coding-tool-actually-is) — the agentic loop and the context window. This doc assumes both.

---

## Table of Contents

- [Part 1 — The Session Loop](#part-1--the-session-loop)
  - [1.1 Beginner: starting, typing, stopping](#11-beginner-starting-typing-stopping)
  - [1.2 Working Knowledge: steering without starting over](#12-working-knowledge-steering-without-starting-over)
  - [1.3 Working Knowledge: shell mode, backgrounding, and the transcript](#13-working-knowledge-shell-mode-backgrounding-and-the-transcript)
  - [1.4 Advanced: the input line as a tool](#14-advanced-the-input-line-as-a-tool)
  - [1.5 Mastery: what the interface is actually doing](#15-mastery-what-the-interface-is-actually-doing)
  - [1.6 Part 1 cheat sheet](#16-part-1-cheat-sheet)
- [Part 2 — Sessions, Checkpoints, and Undo](#part-2--sessions-checkpoints-and-undo)
  - [2.1 Beginner: a session is a saved conversation](#21-beginner-a-session-is-a-saved-conversation)
  - [2.2 Working Knowledge: resume, name, branch](#22-working-knowledge-resume-name-branch)
  - [2.3 Working Knowledge: checkpoints and the rewind menu](#23-working-knowledge-checkpoints-and-the-rewind-menu)
  - [2.4 Advanced: what resuming does and does not restore](#24-advanced-what-resuming-does-and-does-not-restore)
  - [2.5 Advanced: the four holes in checkpointing](#25-advanced-the-four-holes-in-checkpointing)
  - [2.6 Mastery: transcripts on disk](#26-mastery-transcripts-on-disk)
  - [2.7 Part 2 cheat sheet](#27-part-2-cheat-sheet)
- [Part 3 — Getting Good Output](#part-3--getting-good-output)
  - [3.1 Beginner: describe the outcome, not the steps](#31-beginner-describe-the-outcome-not-the-steps)
  - [3.2 Working Knowledge: give Claude a way to verify its work](#32-working-knowledge-give-claude-a-way-to-verify-its-work)
  - [3.3 Working Knowledge: explore, plan, implement, commit](#33-working-knowledge-explore-plan-implement-commit)
  - [3.4 Working Knowledge: feeding in rich context](#34-working-knowledge-feeding-in-rich-context)
  - [3.5 Advanced: the five failure patterns](#35-advanced-the-five-failure-patterns)
  - [3.6 Advanced: adversarial review and the interview pattern](#36-advanced-adversarial-review-and-the-interview-pattern)
  - [3.7 Mastery: escalating how hard the stop is gated](#37-mastery-escalating-how-hard-the-stop-is-gated)
  - [3.8 Part 3 cheat sheet](#38-part-3-cheat-sheet)

---

## Part 1 — The Session Loop

<a id="part-1"></a>

### 1.1 Beginner: starting, typing, stopping

A **session** is one conversation, tied to the directory you launched it from.

```bash
cd your-project
claude
```

You get a prompt box. Type a request in plain English and press `Enter`. Claude reads files, runs commands, edits code, and reports back. That's the whole beginner surface.

Three variants worth knowing on day one:

```bash
claude "explain how routing works in this project"   # start with a first prompt
claude -c                                            # continue the most recent session here
claude --resume                                      # pick a session from a list
```

**Stopping.** `Esc` interrupts Claude immediately — the running tool call is cancelled and Claude waits for your next instruction, with context preserved ([how it works](https://code.claude.com/docs/en/how-claude-code-works)). `Ctrl+C` interrupts or clears the input. `Ctrl+D` exits.

> **Wrong vs. right — the interrupt reflex.**
>
> ```text
> # Wrong: Ctrl+C out of the session because Claude went the wrong way,
> # then relaunch and re-explain everything.
> ```
>
> ```text
> # Right: Esc to stop it mid-action, then type the correction.
> # Context is preserved — Claude already knows what it just learned.
> ```
>
> The wrong version throws away the exploration Claude just did. This is the single most common new-user reflex, and it costs a full re-read of the codebase each time.

<a id="p1-working-knowledge"></a>

### 1.2 Working Knowledge: steering without starting over

Claude Code is conversational, and correcting is cheaper than restarting. There are three distinct ways to redirect, and they are not interchangeable:

| You want to | Do this | What happens |
|---|---|---|
| Stop it right now | `Esc` | Running tool call cancelled; Claude waits. Queued messages get sent next. |
| Adjust without stopping | Type the correction, press `Enter` | Claude reads it as soon as the **current tool call** finishes and adjusts before deciding its next step ([how it works](https://code.claude.com/docs/en/how-claude-code-works)). |
| Line up follow-up work | Type it, press `Enter` while Claude works | Queued. See the queueing rules below. |

The middle one is under-used and genuinely useful: you can watch Claude head down a wrong path, type *"that's the wrong file — the logic moved to `src/auth/v2/`"*, and it course-corrects at the next decision point without losing the read it's currently doing.

**Message queueing.** Type while Claude is working and the entry queues; queued entries are listed above the input box ([interactive mode](https://code.claude.com/docs/en/interactive-mode)). When each one reaches Claude depends on what it is:

- **Messages** queued during tool calls reach Claude as soon as those calls finish, *within the same turn*. If the turn ends with messages still queued, only the **oldest** is sent as the next turn; the rest stay queued.
- **Commands and `!` shell commands** are held until the turn ends, then run one at a time.
- **`/model`, `/effort`, and `/fast`** run immediately rather than queueing, because each changes a setting. `/model` and `/effort` apply to the **next request in the current turn**; `/fast` applies from your next turn, because the running turn keeps the fast-mode setting it started with.

Press `Up` from the first line of the input box to **take back** everything queued — it comes back into the input box, one per line, for editing.

**Course-correct early.** The official guidance is blunt about the economics: if you have corrected Claude more than twice on the same issue in one session, the context is now cluttered with failed approaches, and *"a clean session with a better prompt almost always outperforms a long session with accumulated corrections"* ([best practices](https://code.claude.com/docs/en/best-practices)). `/clear` and re-prompt with what you learned.

<a id="p1-shell"></a>

### 1.3 Working Knowledge: shell mode, backgrounding, and the transcript

**Shell mode (`!`).** Prefix your input with `!` to run a command yourself, outside Claude's decision-making:

```bash
! npm test
! git status
! ls -la
```

The command and its output are added to the conversation context, and **Claude responds to the output automatically** once it lands in the transcript ([interactive mode](https://code.claude.com/docs/en/interactive-mode)). So `! npm test` gets you an explanation of the failures without a second prompt — at the cost of a normal prompt's worth of tokens. Set `respondToBashCommands: false` if you want the output in context without the response.

Shell-mode specifics worth knowing:

- Autocomplete works two ways: `Tab` completes from your previous `!` commands in this project, and typing a token containing `/` (use forward slashes even on Windows) opens a live file-path dropdown.
- Exit shell mode with `Escape`, `Backspace`, or `Ctrl+U` on an empty prompt.
- Pasting text that starts with `!` into an empty prompt enters shell mode automatically.
- **Shell-mode commands run outside the sandbox**, even when sandboxing is enabled, because the sandbox applies to commands *Claude* runs ([sandboxing](https://code.claude.com/docs/en/sandboxing)).

**Backgrounding a long command.** Press `Ctrl+B` while a Bash tool call is running to move it to the background (tmux users press it twice, because of tmux's own prefix key). Output is written to a file that Claude can read back with the Read tool ([interactive mode](https://code.claude.com/docs/en/interactive-mode)).

Background task facts that matter in practice:

- Tasks are cleaned up when Claude Code exits. On macOS and Linux, stopping a task also stops processes that detached from its shell (things started under `setsid` or `timeout`).
- A task is killed if its output exceeds **5GB**.
- On macOS and Linux, Claude Code reaps background tasks under OS memory pressure — but only when the session has been idle for **at least 30 minutes** and no turn or subagent is running. `CLAUDE_CODE_DISABLE_BG_SHELL_PRESSURE_REAP=1` turns that off.
- `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` disables backgrounding entirely.

**The transcript viewer (`Ctrl+O`).** Toggles a scrollable view of the full conversation. In [fullscreen rendering](https://code.claude.com/docs/en/fullscreen) it gains real navigation:

| Key | Does |
|---|---|
| `?` | Shortcut help panel |
| `{` / `}` | Jump to previous / next user prompt (vim paragraph motion) |
| `[` | Dump the conversation into the terminal's native scrollback, so `Cmd+F` and tmux copy-mode can search it |
| `v` | Write the conversation to a temp file and open it in `$VISUAL` / `$EDITOR` |
| `q`, `Ctrl+C`, `Esc` | Exit |

<a id="p1-advanced"></a>

### 1.4 Advanced: the input line as a tool

Most people type into the prompt box like it's a chat app. It is closer to a readline shell, and treating it that way saves real time.

**Editing.** Readline conventions are the behaviour in **v2.1.261 and later** — the `keybindingFlavor` setting that used to enable them is deprecated and has no effect ([interactive mode](https://code.claude.com/docs/en/interactive-mode)).

| Key | Does |
|---|---|
| `Ctrl+A` / `Ctrl+E` | Start / end of the current logical line |
| `Ctrl+K` / `Ctrl+U` | Delete to end of line / to line start (both store the text for pasting) |
| `Ctrl+W` | Delete back to the **previous whitespace** — one press removes a whole path or `--flag=value` |
| `Ctrl+Y` | Paste the last text you deleted with `Ctrl+K`/`Ctrl+U`/`Ctrl+W` |
| `Alt+Y` after `Ctrl+Y` | Cycle the paste history |
| `Alt+B` / `Alt+F` | Move back / forward one word |
| `Alt+D` | Delete to end of word |
| `Ctrl+_` | Undo the last input edit |

The word-boundary distinction is worth internalising: `Alt+B`/`Alt+F`/`Alt+D` treat a *word* as a run of letters and digits, so `_`, `.`, and `/` are separators — in `src/utils/foo.ts`, repeated `Alt+B` stops at `ts`, `foo`, `utils`, `src`. **`Ctrl+W` ignores punctuation entirely** and deletes back to whitespace, so one press removes the whole path. These are not remappable in [`keybindings.json`](https://code.claude.com/docs/en/keybindings) — there are no actions for them.

> **macOS caveat:** every `Alt`/`Option` shortcut requires configuring Option as Meta in your terminal. See [terminal config](https://code.claude.com/docs/en/terminal-config#enable-option-key-shortcuts-on-macos).

**Multiline input.** Five ways, in rough order of reliability:

| Method | How | Works where |
|---|---|---|
| Backslash | `\` then `Enter` | Every terminal |
| Control sequence | `Ctrl+J` | Every terminal, no config |
| Shift+Enter | `Shift+Enter` | Native in iTerm2, WezTerm, Ghostty, Kitty, Warp, Apple Terminal, Windows Terminal |
| Option key | `Option+Enter` | macOS, after enabling Option as Meta |
| Paste | Just paste | Code blocks, logs |

**History.** Input history is stored **per working directory** and `Up` reaches prompts from past sessions of the same project. `Ctrl+R` does reverse search; in fullscreen it opens a dialog where `Ctrl+S` cycles the scope through *this session → this project → all projects*. Note that the inline (non-fullscreen) search always searches **all projects** ([interactive mode](https://code.claude.com/docs/en/interactive-mode)).

**The other input prefixes.**

| Prefix | Does |
|---|---|
| `/` | Command or skill menu — built-ins, your skills, plugin skills, MCP prompts |
| `!` | Shell mode |
| `@` | File-path autocomplete; in sessions with cross-session messaging enabled, typing a letter after `@` also suggests your other live sessions |
| `:` | Emoji shortcode (`:name:`), v2.1.217+ |
| `?` on empty input | Shortcut help panel |

**Side questions with `/btw`.** Ask something about the session without adding it to the conversation history:

```text
/btw what model am I on and how much context is left?
```

The answer never enters context, so you can check a detail without paying for it on every subsequent request. `/btw` with no argument shows your most recent side question.

**Vim mode.** Enable via `/config` → Editor mode (or `editorMode` in settings). Claude Code preserves your vim mode and cursor position across `Ctrl+O` and panel toggles — leave the prompt in NORMAL mode and it's still in NORMAL mode when you return.

<a id="p1-mastery"></a>

### 1.5 Mastery: what the interface is actually doing

Three mechanics that explain surprising behaviour:

**1. Some commands bypass the queue because they mutate session state.** `/model`, `/effort`, and `/fast` are applied immediately rather than queued, and each has its own rule about whether the *running* turn is affected. This exists because these settings change how the next API request is formed, and a queued setting change would be useless. The prompt-cache implication is real: switching model or effort mid-turn can invalidate the cache, which is why Claude Code may show you a cache warning first ([prompt caching](https://code.claude.com/docs/en/prompt-caching)).

**2. `Esc` `Esc` is overloaded and the overload is deliberate.** With text in the input box, it clears the draft (saving it to input history — press `Up` to get it back). With an empty input box, it opens the rewind menu. If you have ever pressed it twice expecting a rewind and got nothing, your input box wasn't empty.

**3. The `/` menu is a merge of four sources.** Built-in commands, bundled skills (`/code-review`, `/doctor`, `/batch`, `/debug`, `/loop`…), your own skills and `.claude/commands/` files, and prompts exposed by connected MCP servers all land in the same list ([interactive mode](https://code.claude.com/docs/en/interactive-mode)). When a command behaves unexpectedly, identifying which of the four it is comes first — `/help` groups them, and bundled skills are marked `[Skill]` in the [commands reference](https://code.claude.com/docs/en/commands).

> ### Real Scenario — the phantom `/deploy`
>
> A developer runs `/deploy` expecting the team's checked-in deployment skill at `.claude/skills/deploy/SKILL.md`. Instead Claude starts talking about Kubernetes manifests the repo doesn't have.
>
> The cause: a teammate installed a marketplace plugin that ships its own deploy skill. Plugin skills are namespaced (`/plugin-name:deploy`) precisely to avoid this — but the developer had *also* left a stale `.claude/commands/deploy.md` from before the team migrated to skills, and the file's content was the outdated Kubernetes flow.
>
> Diagnosis path: `/help` → **Custom commands** tab shows every command with its source. The fix was deleting the stale `commands/` file. The lesson: when both a `.claude/commands/<name>.md` and a `.claude/skills/<name>/SKILL.md` exist, **the skill wins** ([skills](https://code.claude.com/docs/en/skills)) — so this only bit because the skill hadn't been created yet in that developer's branch. `/doctor` also flags unused and duplicated skills.

<a id="16-part-1-cheat-sheet"></a>

### 1.6 Part 1 cheat sheet

| Key / prefix | Effect |
|---|---|
| `Esc` | Interrupt Claude, or close a dialog |
| `Esc` `Esc` | Rewind menu (empty input) / clear draft (with text) |
| `Ctrl+C` · `Ctrl+D` | Interrupt or clear input · exit |
| `Ctrl+O` | Transcript viewer |
| `Ctrl+G` (or `Ctrl+X Ctrl+E`) | Open input in `$EDITOR` |
| `Ctrl+B` | Background the running Bash task (×2 in tmux) |
| `Ctrl+R` | Reverse-search prompt history |
| `Ctrl+T` | Toggle task checklist |
| `Ctrl+S` · `Ctrl+L` · `Ctrl+Z` | Stash/restore prompt · redraw · suspend |
| `Ctrl+X Ctrl+K` | Stop all background subagents in this session |
| `Shift+Tab` | Cycle permission modes |
| `Alt+P` / `Option+P` | Switch model |
| `Alt+T` / `Option+T` | Toggle extended thinking |
| `Alt+O` / `Option+O` | Toggle fast mode |
| `Ctrl+A` `Ctrl+E` `Ctrl+K` `Ctrl+U` `Ctrl+W` `Ctrl+Y` | Readline line editing |
| `Alt+B` `Alt+F` `Alt+D` | Word motion / deletion |
| `\`+`Enter`, `Ctrl+J`, `Shift+Enter` | Multiline |
| `/` `!` `@` `:` `?` | Command · shell · file mention · emoji · help |

| Command | Effect |
|---|---|
| `/btw <q>` | Side question, kept out of context |
| `/diff` | Review the working tree including Claude's edits |
| `/copy [N]` | Copy the Nth-latest response (picker for code blocks; `w` writes to file) |
| `/export [file]` | Export the conversation as plain text |
| `/tasks` | Manage background tasks |
| `/focus` | Collapse to prompt + tool summary + response (fullscreen only) |

[↑ Back to top](#table-of-contents)

---

## Part 2 — Sessions, Checkpoints, and Undo

<a id="part-2"></a>

*This Part gets all four tiers. The Advanced material — what resume restores and where checkpointing has holes — is where most real incidents come from.*

### 2.1 Beginner: a session is a saved conversation

Each message, tool use, and result is written to a plaintext JSONL file under `~/.claude/projects/` as you work ([how it works](https://code.claude.com/docs/en/how-claude-code-works)). That persistence is what makes resuming, forking, and rewinding possible.

Two facts to internalise immediately:

1. **Sessions are independent.** Each new session starts with a fresh context window and none of the previous conversation. What carries across sessions is `CLAUDE.md` and auto memory — not chat history. (Part 5 in [Context & Memory](./claude-code-context-memory.md).)
2. **Sessions are tied to a directory.** Switch git branches and Claude sees the new branch's files, but the conversation is unchanged — it still remembers what you discussed.

```bash
claude --continue     # resume the most recent session in this directory
claude --resume       # open the session picker
```

### 2.2 Working Knowledge: resume, name, branch

**The four entry points** ([sessions](https://code.claude.com/docs/en/sessions)):

| Command | What it does |
|---|---|
| `claude --continue` (`-c`) | Resumes the most recent **interactive** session in this directory |
| `claude --resume` (`-r`) | Opens the session picker |
| `claude --resume <name-or-id>` | Resumes directly |
| `claude --from-pr <number>` | Opens the picker filtered to sessions linked to that PR |
| `/resume` | Switch conversations from inside a session |

Important exclusions: sessions created by `claude -p` or the Agent SDK are **left out of the picker and out of `--continue`** — you can still resume one by passing its session ID. `--continue` also skips background sessions and sessions whose first prompt was `/loop`.

**Name your sessions.** This matters more than it sounds once you run more than one workstream:

| When | How |
|---|---|
| At startup | `claude -n auth-refactor` |
| During a session | `/rename auth-refactor` (the name shows on the prompt bar) |
| From the picker | Highlight and press `Ctrl+R` |
| On plan accept | Accepting a plan generates a title from the plan, unless you've already named it |

Unnamed sessions get two labels, and only one of them is a resume handle: a **default display name** like `my-app-3f` (used in agent view listings, *not* resumable) and an **AI-generated title** summarising your first prompt (which *is* resumable). If `claude --resume my-app-3f` says it can't find the session, that's why.

If you start or rename a session into a name another live session on this machine already holds, Claude Code leaves the name with the original and renames yours with a two-word suffix (`auth-refactor-graceful-unicorn`), telling you it did (v2.1.232+).

**The session picker.** `/resume` or `claude --resume` with no argument:

| Key | Action |
|---|---|
| `↑` `↓` | Navigate |
| `→` `←` | Expand / collapse grouped sessions |
| `Enter` | Resume |
| `Space` | Preview session content |
| `Ctrl+R` | Rename |
| `/` or any printable char | Search — **paste a GitHub/GitLab/Bitbucket PR or MR URL to find the session that created it** |
| `Ctrl+A` | Widen to all projects on this machine |
| `Ctrl+W` | Widen to all worktrees of this repo |
| `Ctrl+B` | Filter to the current git branch |
| `Esc` | Exit |

**Branching.** `/branch [name]` copies the conversation so far into a new session and switches you into it, leaving the original untouched:

```text
/branch try-streaming-approach
```

From the command line, `claude --continue --fork-session` does the same thing. The two differ in one way that matters: `/branch` copies the transcript and switches the **running process**, so it carries over your "allow for this session" permission grants, in-flight background subagents and Bash commands, and an attached Remote Control connection. `--fork-session` starts a separate process, so you re-approve permissions there ([sessions](https://code.claude.com/docs/en/sessions)).

> **Wrong vs. right — trying a risky approach.**
>
> ```text
> # Wrong: just try it in the current session and hope /rewind is enough.
> # If the experiment involved Bash commands, rewind can't undo them.
> ```
>
> ```text
> # Right:
> /branch try-async-rewrite
> # Explore freely. If it fails, /resume back to the original,
> # which is untouched on disk and still in the picker.
> ```

### 2.3 Working Knowledge: checkpoints and the rewind menu

**Every prompt you send creates a checkpoint**, and before Claude edits a file it snapshots the current contents ([checkpointing](https://code.claude.com/docs/en/checkpointing)).

Open the menu with `/rewind`, or `Esc` `Esc` on empty input. It lists each prompt you sent. Pick one, then pick an action:

| Action | Effect |
|---|---|
| **Restore code and conversation** | Revert both to that point |
| **Restore conversation** | Rewind the conversation, keep current code |
| **Restore code** | Revert files, keep the conversation |
| **Summarize from here** | Compress everything from this point forward into a summary — frees context |
| **Summarize up to here** | Compress everything *before* this point, keeping later messages intact |
| **Never mind** | Back out |

The two code-restore options only appear when the selected checkpoint has tracked file changes to revert.

The **summarize** options are the underrated half of this menu. They are a targeted `/compact`: they let you throw away a verbose debugging detour from the middle of a session while keeping your original instructions in full, or the reverse. To guide the summary, highlight a Summarize option with the arrow keys and type into the **add context (optional)** field before pressing `Enter` (selecting with its number key summarises immediately, without instructions).

**Retention.** Claude Code keeps file snapshots for the **100 most recent checkpoints** in a session, and deletes a session's snapshots in the retention sweep — by default about **30 days** after the session last saved one. Rewinding to a checkpoint whose snapshots are gone fails with `No files were restored`. Raise [`cleanupPeriodDays`](https://code.claude.com/docs/en/settings-reference#cleanupperioddays) to keep them longer.

**Rewinding past a `/clear`.** If you ran `/clear` earlier in the *same* Claude Code process, the rewind menu shows an extra entry at the top: `/resume <session-id> (previous session)`. Selecting it returns you to the conversation that was active before the clear (v2.1.191+). This is the escape hatch for "I cleared and immediately realised I needed that."

<a id="p2-advanced-resume"></a>

### 2.4 Advanced: what resuming does and does not restore

This is where people get surprised. A resumed session restores the conversation *and some state*, but not your launch flags ([sessions](https://code.claude.com/docs/en/sessions)).

**Restored:**

- Full conversation history, including tool calls and results
- The model the session was using — *unless* it's been retired, isn't allowed by `availableModels`, a `--model` flag or `ANTHROPIC_MODEL`-family variable picks one at launch, or you're on a provider using deployment IDs
- The agent, if the session was started with `--agent`
- An active `/goal` (its turn count, timer, and token baseline reset)
- Non-expired scheduled tasks — but **not** background Bash or monitor tasks

**Not restored — you must pass these again:**

`--mcp-config`, `--settings`, `--plugin-dir`, `--fallback-model`, and directories added with `--add-dir`. Directories added mid-session with `/add-dir` aren't restored either. Standard settings files (`settings.json`, `settings.local.json`) *are* re-read at launch, so anything living in them doesn't need re-passing.

**Permission mode on resume is the subtle one.** It depends on *how* you resume:

| Session ended in | How you resume | Mode after resume |
|---|---|---|
| `bypassPermissions` | Terminal | The mode a **new** session would start in — you must re-enable bypass at launch |
| `plan` | Terminal | The mode a new session would start in |
| `auto` | Terminal | `auto`, if your account still meets the auto-mode requirements |
| Manual (`default`) | Terminal | Manual, when a new session would have started in auto from the built-in default; a settings `defaultMode` wins if one applies |
| Any | Session picker (`--resume` with no arg, `--from-pr`, ambiguous name) | **Not restored** — starts in the mode a new session would |
| Any | `/resume` inside a session | **Not restored** — continues in your *current* session's mode |
| `plan` | `-p` with `--permission-prompt-tool`, no `--permission-mode`/`--dangerously-skip-permissions`/`--fork-session`, not via channels | Plan mode |
| Any | `-p`, any other case | The mode a new `-p` run would start in |

The practical takeaway: **picking from the picker and resuming by exact name are not the same thing.** Only the direct forms (`--continue`, `--resume <id>`, `--resume <name>` matching exactly one session) restore the stored mode.

**Resume from a summary.** On Pro or Max, resuming a session that's been idle over ~an hour and is over **100,000 tokens** opens a dialog before your first message, because the [prompt cache](https://code.claude.com/docs/en/prompt-caching#cache-lifetime) has expired. Three options:

- **Resume from summary** — runs `/compact` immediately; later requests carry the summary instead of full history. Cheaper per request, but whatever the summary dropped is gone.
- **Resume full session as-is** — reprocesses and re-caches the full history, then reads from cache while it stays warm. Every detail available, per-request cost scaling with conversation size.
- **Don't ask me again** — always resume full, stop showing the dialog.

**Cross-project resume.** `claude --resume <session-id>` works from any directory: Claude Code looks in the current project and its worktrees first, then every other project on the machine (v2.1.223+). The cross-project search only resolves when exactly one other project holds a transcript with messages for that ID, so a hand-copied duplicate reports not-found rather than resuming an arbitrary copy.

<a id="p2-advanced-checkpoints"></a>

### 2.5 Advanced: the four holes in checkpointing

Checkpointing is genuinely useful and genuinely partial. Knowing the holes is the difference between trusting it appropriately and trusting it fatally ([checkpointing](https://code.claude.com/docs/en/checkpointing)).

```text
  What /rewind can restore              What it cannot
  ──────────────────────────            ──────────────────────────────────
  ✅ Edit / Write / NotebookEdit        ❌ Anything a Bash command changed
     by the main conversation              (rm, mv, cp, sed, a build script,
                                            a migration, npm install …)

  ✅ A foreground forked skill's        ❌ Any other subagent's edits —
     edits (context: fork with              including background forked
     background: false)                     skills and /code-review --fix

                                        ❌ Your own manual edits, and edits
                                           from other concurrent sessions

                                        ❌ Symlinked and hard-linked paths
                                           (skipped, with a warning)
```

**1. Bash-driven changes aren't tracked.** Only direct file edits through Claude's file tools are. If Claude ran `mv old.ts new.ts` or a codemod script, rewind won't undo it.

**2. Subagent edits usually aren't restored.** The exception is a **foreground forked skill** (`context: fork` with `background: false`), which edits your working tree during your own turn. Everything else — including the *default* background fork and a background `/code-review --fix` — needs git to revert.

**3. External changes aren't captured**, unless they happen to touch the same files the session already edited.

**4. Symlinks and hard links are skipped.** A restore skips any tracked path that is a symlink or hard link and warns `Restored the code, but skipped N files`; the skipped files keep their current contents. This hits two common setups: config files a dotfile manager symlinks into your project, and files **pnpm hard-links** into place. Turn on `/debug` before restoring and the debug log at `~/.claude/debug/<session-id>.txt` names each skipped path.

> ### Real Scenario — the migration that rewind couldn't undo
>
> An engineer asks Claude to "rename the `users` table to `accounts` and update all references." Claude edits 14 TypeScript files with the Edit tool, then runs `npx prisma migrate dev --name rename-users` to apply the schema change. Tests fail in a way that suggests the whole approach was wrong.
>
> The engineer presses `Esc` `Esc`, restores code and conversation to before the request, and re-runs the test suite. It still fails — worse, differently.
>
> **What happened:** the 14 file edits were reverted. The migration was not. Prisma had already written a migration file, applied it to the dev database, and regenerated the client. The codebase now referenced `users` while the database had `accounts`, a state that had never existed before.
>
> **The fix, and the habit:** `git status` immediately after any rewind, before running anything. Better: commit before delegating work that touches infrastructure, so `git reset --hard` is available as the real undo. Best: put irreversible operations behind a permission `ask` rule or a `PreToolUse` hook (see [Permissions & Security](./claude-code-permissions-security.md)), so they never run unattended in the first place. The official docs state it plainly: checkpointing is *"not a replacement for version control."*

<a id="p2-mastery"></a>

### 2.6 Mastery: transcripts on disk

Transcripts are JSONL at `~/.claude/projects/<project>/<session-id>.jsonl`, where `<project>` is your working directory path with non-alphanumeric characters replaced by `-`. A converted name over 200 characters is truncated to 200 with a hash of the full path appended ([sessions](https://code.claude.com/docs/en/sessions)).

**Don't parse these files.** The entry format is internal and changes between versions, so a script reading them directly can break on any release. The supported interfaces, picked by what triggers your script:

| Trigger | Interface |
|---|---|
| Run Claude once, capture the result | `claude -p --output-format json` (or `stream-json`) |
| Ask an existing session a question | `claude -p --resume <id> --output-format json "summarize what we changed"` |
| React to session events | The `transcript_path` field that [hooks](https://code.claude.com/docs/en/hooks#common-input-fields) and [status line](https://code.claude.com/docs/en/statusline) commands receive on stdin — a `SessionEnd` hook can archive it |
| Embed in an app | The [Agent SDK](https://code.claude.com/docs/en/agent-sdk/overview) |

```bash
claude -p --resume <session-id> --output-format json "summarize what we changed" | jq -r '.result'
```

**Storage controls:**

| To | Set | Where |
|---|---|---|
| Move storage off `~/.claude` | `CLAUDE_CONFIG_DIR` | Env var |
| Name the `<project>` directory yourself | `CLAUDE_CODE_PROJECT_DIR_NAME` (v2.1.234+, requires `CLAUDE_CONFIG_DIR` too) | Env var |
| Change the 30-day retention | `cleanupPeriodDays` | `settings.json` |
| Suppress transcript writes everywhere | `CLAUDE_CODE_SKIP_PROMPT_HISTORY` | Env var |
| Suppress for one non-interactive run | `--no-session-persistence` | CLI flag with `-p` |

**Two terminals, one session.** If you resume the same session in two terminals *without* forking, messages from both interleave into one transcript. That is almost never what you want — fork instead.

<a id="27-part-2-cheat-sheet"></a>

### 2.7 Part 2 cheat sheet

| Command | Effect |
|---|---|
| `claude -c` | Continue the most recent interactive session here |
| `claude -r <name\|id>` | Resume directly (restores stored permission mode) |
| `claude --resume` | Session picker (does **not** restore permission mode) |
| `claude --from-pr 123` | Picker filtered to that PR's sessions |
| `claude -n <name>` | Name the session at startup |
| `claude -c --fork-session` | Fork into a new process (permission grants not carried) |
| `/rename <name>` | Name the current session |
| `/branch [name]` | Fork in place, carrying grants and in-flight work |
| `/resume [name]` | Switch conversation from inside a session |
| `/clear [name]` | New empty context; optionally label the conversation you're leaving |
| `/rewind` · `Esc` `Esc` | Rewind / summarize menu |
| `/export [file]` | Export the transcript as readable text |

| Fact | Value |
|---|---|
| Checkpoint snapshots kept | 100 most recent per session |
| Snapshot retention | ~30 days (`cleanupPeriodDays`) |
| Transcript location | `~/.claude/projects/<project>/<session-id>.jsonl` |
| Resume-from-summary trigger | Pro/Max, idle >~1h, >100,000 tokens |
| Not restored on resume | `--mcp-config`, `--settings`, `--plugin-dir`, `--fallback-model`, `--add-dir` |
| Not covered by checkpoints | Bash changes, most subagent edits, external edits, symlinks/hard links |

[↑ Back to top](#table-of-contents)

---

## Part 3 — Getting Good Output

<a id="part-3"></a>

*Beginner is thin here and deliberately so — the beginner version of "prompt well" is one sentence. The weight is in Working Knowledge and Advanced.*

### 3.1 Beginner: describe the outcome, not the steps

Give context and direction, then let Claude figure out the details ([best practices](https://code.claude.com/docs/en/best-practices)):

```text
The checkout flow is broken for users with expired cards.
The relevant code is in src/payments/. Can you investigate and fix it?
```

You do not need to say which files to read or which commands to run.

### 3.2 Working Knowledge: give Claude a way to verify its work

This is the highest-leverage habit in the entire tool, so it gets stated plainly:

> **Claude stops when the work looks done. Without a check it can run, "looks done" is the only signal available — and you become the verification loop.**

Give it something that returns pass or fail and the loop closes on its own: a test suite, a build exit code, a linter, a script that diffs output against a fixture, a browser screenshot compared to a design ([best practices](https://code.claude.com/docs/en/best-practices)).

| Strategy | Before | After |
|---|---|---|
| **Provide verification criteria** | *"implement a function that validates email addresses"* | *"write a `validateEmail` function. example cases: `user@example.com` → true, `invalid` → false, `user@.com` → false. run the tests after implementing"* |
| **Verify UI changes visually** | *"make the dashboard look better"* | *"[paste screenshot] implement this design. take a screenshot of the result and compare it to the original. list differences and fix them"* |
| **Address root causes** | *"the build is failing"* | *"the build fails with this error: [paste]. fix it and verify the build succeeds. address the root cause, don't suppress the error"* |

And ask for **evidence, not assertions**: the test output, the command it ran and what it returned, a screenshot. Reviewing evidence is faster than re-running the verification yourself, and it works for sessions you weren't watching.

### 3.3 Working Knowledge: explore, plan, implement, commit

**Plan mode** tells Claude to research and propose without editing your source. It reads files and runs exploratory commands, then writes a plan ([permission modes](https://code.claude.com/docs/en/permission-modes)).

```bash
claude --permission-mode plan     # start in plan mode
```

or `Shift+Tab` until the status bar shows `⏸ plan mode on`, or `/plan [description]` to enter it for one prompt.

The four-phase workflow:

```text
  1. EXPLORE  ──▶  2. PLAN  ──▶  3. IMPLEMENT  ──▶  4. COMMIT
     (plan mode)     (plan mode)    (leave plan mode)
     read /src/auth  "what files    "implement the      "commit with a
     understand      need to        OAuth flow from     descriptive
     sessions and    change? what's your plan. write    message and
     login           the session    tests for the       open a PR"
                     flow? make     callback handler,
                     a plan"        run the suite and
                                    fix failures"
```

`Ctrl+G` opens the plan in your editor so you can edit it directly before Claude proceeds.

**Plan mode has overhead, and the docs say so.** For tasks where the scope is clear and the fix is small — a typo, a log line, a rename — just ask. *"If you could describe the diff in one sentence, skip the plan."* Planning earns its cost when you're uncertain about the approach, when the change spans multiple files, or when you're unfamiliar with the code.

One behaviour worth knowing: when auto mode is available and `useAutoModeDuringPlan` is on (the default), the classifier reviews shell commands during planning instead of prompting you, so exploration doesn't stop every 30 seconds for approval.

### 3.4 Working Knowledge: feeding in rich context

| Method | How |
|---|---|
| **Reference files with `@`** | `@src/auth/session.ts` — Claude reads the file before responding |
| **Paste images** | Copy/paste or drag and drop into the prompt; `Ctrl+V` (`Cmd+V` in iTerm2, `Alt+V` on Windows/WSL) pastes from clipboard |
| **Give URLs** | Claude fetches them. Use `/permissions` to allowlist domains you use often |
| **Pipe data** | `cat error.log \| claude -p "explain the root cause"` |
| **Let Claude fetch** | Tell it to pull context itself via Bash, MCP tools, or file reads |
| **Shell mode** | `! npm test` puts the output in context and gets a response |

**Point at patterns, not just tasks.** The strongest single prompt improvement in the official guidance:

> *"look at how existing widgets are implemented on the home page to understand the patterns. `HotDogWidget.php` is a good example. follow the pattern to implement a new calendar widget…"*

This works because it converts an open-ended generation task into a constrained transformation task.

**Use CLI tools.** They are the most context-efficient way to reach external services. Install `gh` and Claude uses it for issues, PRs, and comments — without it, unauthenticated API requests hit rate limits. Claude is also good at learning CLIs it doesn't know: *"Use `foo-cli --help` to learn about foo, then use it to solve A, B, C."*

<a id="p3-failures"></a>

### 3.5 Advanced: the five failure patterns

These come straight from the official best-practices page, and they account for most wasted sessions ([best practices](https://code.claude.com/docs/en/best-practices)):

| Pattern | What it looks like | Fix |
|---|---|---|
| **The kitchen sink session** | One task, then something unrelated, then back to the first. Context is full of irrelevance. | `/clear` between unrelated tasks |
| **Correcting over and over** | Wrong → correct → still wrong → correct again. Context is polluted with failed approaches. | After **two** failed corrections, `/clear` and write a better initial prompt incorporating what you learned |
| **The over-specified `CLAUDE.md`** | Too long, so Claude ignores half of it — important rules lost in noise | Prune ruthlessly. If Claude does it right without the instruction, delete it — or convert it to a hook |
| **The trust-then-verify gap** | Plausible-looking implementation that doesn't handle edge cases | Always provide verification. *If you can't verify it, don't ship it* |
| **The infinite exploration** | "Investigate X" with no scope; Claude reads hundreds of files | Scope narrowly, or delegate to a subagent so the reads don't land in your context |

The two-correction rule deserves emphasis because it feels wrong in the moment: your instinct after two failures is that the third correction will land. It usually doesn't, because the context now contains two wrong approaches that keep influencing the next attempt.

**Delegate research to subagents.** Since context is the fundamental constraint:

```text
Use subagents to investigate how our authentication system handles token
refresh, and whether we have any existing OAuth utilities I should reuse.
```

The subagent reads dozens of files in *its* context window and returns a summary to yours ([sub-agents](https://code.claude.com/docs/en/sub-agents)). Part 11 in [Extensibility](./claude-code-extensibility.md) covers this properly.

<a id="p3-adversarial"></a>

### 3.6 Advanced: adversarial review and the interview pattern

**The adversarial review step.** The longer Claude works unattended, the more an independent check matters. A reviewer running in a fresh subagent context sees only the diff and your criteria — not the reasoning that produced the change — so it evaluates the result on its own terms.

```text
Use a subagent to review the rate limiter diff against PLAN.md. Check that
every requirement is implemented, the listed edge cases have tests, and
nothing outside the task's scope changed. Report gaps, not style preferences.
```

Or run the bundled `/code-review` skill, which reviews the current diff for bugs in a fresh subagent and returns findings to the session.

> **The counterweight, which the docs state explicitly:** *"A reviewer prompted to find gaps will usually report some, even when the work is sound, because that is what it was asked to do."* Chasing every finding leads to over-engineering — extra abstraction layers, defensive code, tests for cases that can't happen. Tell the reviewer to flag only gaps affecting correctness or the stated requirements, and treat the rest as optional.

**The interview pattern.** For larger features, have Claude interrogate you before any code exists:

```text
I want to build [brief description]. Interview me in detail using the
AskUserQuestion tool.

Ask about technical implementation, UI/UX, edge cases, concerns, and
tradeoffs. Don't ask obvious questions, dig into the hard parts I might
not have considered.

Keep interviewing until we've covered everything, then write a complete
spec to SPEC.md.
```

Then **start a fresh session to execute it** — clean context focused entirely on implementation, with a written spec to reference. The most useful specs are self-contained: they name the files and interfaces involved, state what is out of scope, and end with an end-to-end verification step.

**The writer/reviewer split.** Two sessions, fresh context in the reviewer:

| Session A (writer) | Session B (reviewer) |
|---|---|
| `Implement a rate limiter for our API endpoints` | |
| | `Review the rate limiter in @src/middleware/rateLimiter.ts. Look for edge cases, race conditions, and consistency with our existing middleware patterns.` |
| `Here's the review feedback: [B's output]. Address these issues.` | |

A fresh context improves review quality because Claude isn't biased toward code it just wrote.

<a id="p3-mastery"></a>

### 3.7 Mastery: escalating how hard the stop is gated

Once you have a check that returns pass/fail, you can decide how hard it gates the end of the turn. Four levels, trading setup effort for how little attention the run needs ([best practices](https://code.claude.com/docs/en/best-practices)):

```text
   less setup                                              less attention
   ─────────────────────────────────────────────────────────────────────▶

   1. IN ONE PROMPT          "run the tests and iterate until they pass"
      ↓                      Works today, on any task. Claude may still
                             stop early if it decides it's done.

   2. ACROSS A SESSION       /goal all tests in src/auth pass
      ↓                      A separate evaluator re-checks after every
                             turn; Claude keeps working until it resolves.

   3. DETERMINISTIC GATE     A Stop hook runs your check as a script and
      ↓                      blocks the turn from ending until it passes.
                             Claude Code overrides it after 8 consecutive
                             blocks, so it can't loop forever.

   4. SECOND OPINION         A verification subagent or dynamic workflow
                             has a fresh model try to refute the result —
                             so the agent doing the work isn't grading it.
```

Levels 2 and 3 are what let an unattended run finish *correctly* without you. Level 4 is what you add when correctness matters more than throughput.

The `/goal` mechanism is worth a specific note: it survives resume (with its turn count, timer, and token-spend baseline reset), and if Claude stalls, Claude Code eventually stops the run with the goal still set rather than looping indefinitely ([goal](https://code.claude.com/docs/en/goal)). Part 14 in [Automation & Agents](./claude-code-automation-agents.md) covers goals, `/loop`, and workflows.

**Develop your own intuition.** The closing advice on the official page is worth repeating because it inoculates against cargo-culting this doc: sometimes you *should* let context accumulate, because you're deep in one complex problem and the history is valuable. Sometimes you should skip planning because the task is exploratory. Sometimes a vague prompt is exactly right, because you want to see how Claude interprets the problem before constraining it. Pay attention to what works.

<a id="38-part-3-cheat-sheet"></a>

### 3.8 Part 3 cheat sheet

| Habit | Concretely |
|---|---|
| Give a verifiable finish condition | Name the test command, the expected output, or the screenshot to match |
| Ask for evidence | "show me the test output", not "confirm it works" |
| Plan before multi-file or unfamiliar work | `Shift+Tab` to plan mode, or `claude --permission-mode plan` |
| Skip planning for one-sentence diffs | Typos, log lines, renames |
| Scope investigations | Or route them through a subagent |
| `/clear` between unrelated tasks | Context hygiene beats a long session |
| `/clear` after two failed corrections | Re-prompt with what you learned |
| Point at an existing pattern | "follow the pattern in `HotDogWidget.php`" |
| Review in a fresh context | `/code-review`, a review subagent, or a second session |
| Constrain the reviewer | "report gaps that affect correctness, not style preferences" |

| Escalation | Mechanism |
|---|---|
| One prompt | "run the check and iterate" |
| Whole session | `/goal <condition>` |
| Deterministic | `Stop` hook (overridden after 8 consecutive blocks) |
| Independent | Verification subagent or dynamic workflow |

| Prompt upgrade | Before → After |
|---|---|
| Scope the task | *"add tests for foo.py"* → *"write a test for foo.py covering the logged-out edge case. avoid mocks."* |
| Point to sources | *"why is this API weird?"* → *"look through `ExecutionFactory`'s git history and summarize how its api came to be"* |
| Describe the symptom | *"fix the login bug"* → *"login fails after session timeout. check `src/auth/`, especially token refresh. write a failing test, then fix it"* |

[↑ Back to top](#table-of-contents)

---

**Next:** [Context & Memory (Parts 4–5)](./claude-code-context-memory.md) — what Claude knows at the start of a turn, what it costs, what survives compaction, and how to make knowledge persist across sessions.
