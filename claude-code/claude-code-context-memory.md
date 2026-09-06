# Claude Code — Context & Memory (Parts 4–5)

What Claude knows when a turn starts, what that knowledge costs, what happens to it when the window fills, and how to make the parts that matter survive.

> **Spec:** this doc follows the canonical spec in [`claude-code-mastery-guide.md`](./claude-code-mastery-guide.md#about-this-document). Written against **Claude Code v2.1.263**, verified **September 6, 2026**.
>
> **Prerequisite:** [§4 of the overview](./claude-code-mastery-guide.md#4-primer-what-an-agentic-coding-tool-actually-is). If "the context window is the fundamental constraint" isn't yet obvious to you, read that first — this whole doc is the elaboration of it.

---

## Table of Contents

- [Part 4 — The Context Window](#part-4--the-context-window)
  - [4.1 Beginner: what's in there](#41-beginner-whats-in-there)
  - [4.2 Working Knowledge: reading `/context`](#42-working-knowledge-reading-context)
  - [4.3 Working Knowledge: the four ways to free space](#43-working-knowledge-the-four-ways-to-free-space)
  - [4.4 Advanced: what survives compaction](#44-advanced-what-survives-compaction)
  - [4.5 Advanced: tuning when compaction happens](#45-advanced-tuning-when-compaction-happens)
  - [4.6 Mastery: context cost per feature, and the deferral tricks](#46-mastery-context-cost-per-feature-and-the-deferral-tricks)
  - [4.7 Part 4 cheat sheet](#47-part-4-cheat-sheet)
- [Part 5 — Memory That Persists](#part-5--memory-that-persists)
  - [5.1 Beginner: `CLAUDE.md` and `/init`](#51-beginner-claudemd-and-init)
  - [5.2 Working Knowledge: where `CLAUDE.md` files live and how they load](#52-working-knowledge-where-claudemd-files-live-and-how-they-load)
  - [5.3 Working Knowledge: writing one that actually gets followed](#53-working-knowledge-writing-one-that-actually-gets-followed)
  - [5.4 Working Knowledge: auto memory](#54-working-knowledge-auto-memory)
  - [5.5 Advanced: `.claude/rules/` and path scoping](#p5-rules)
  - [5.6 Advanced: imports, `AGENTS.md`, and the trust dialog](#56-advanced-imports-agentsmd-and-the-trust-dialog)
  - [5.7 Advanced: monorepos and excluding other teams' files](#57-advanced-monorepos-and-excluding-other-teams-files)
  - [5.8 Mastery: memory is context, not configuration](#58-mastery-memory-is-context-not-configuration)
  - [5.9 Part 5 cheat sheet](#59-part-5-cheat-sheet)

---

## Part 4 — The Context Window

<a id="part-4"></a>

### 4.1 Beginner: what's in there

The context window is Claude's working memory for a session. It holds ([how it works](https://code.claude.com/docs/en/how-claude-code-works)):

- the system prompt
- your conversation history — every message, both directions
- every file Claude read, in full
- every command's output
- `CLAUDE.md` files and `.claude/rules/`
- auto memory (`MEMORY.md`)
- skill descriptions, and the full body of any skill that got used
- MCP tool names (schemas stay deferred)

Two consequences:

1. **It fills up fast.** A single debugging session can burn tens of thousands of tokens on file reads and test output alone.
2. **Performance degrades as it fills.** Claude starts "forgetting" earlier instructions and making more mistakes ([best practices](https://code.claude.com/docs/en/best-practices)).

One command to know:

```text
/context
```

It shows what is consuming space right now, as a coloured grid, with optimisation suggestions for context-heavy tools and memory bloat. `/context all` expands the per-item breakdown.

### 4.2 Working Knowledge: reading `/context`

Here is roughly what a real session's startup looks like, using the representative figures from the official [context window walkthrough](https://code.claude.com/docs/en/context-window):

```text
  BEFORE YOU TYPE ANYTHING                              ~7,850 tokens
  ┌──────────────────────────────────────────────────────────────────┐
  │ System prompt                                    ~4,200  hidden  │
  │ Project CLAUDE.md                                ~1,800  yours   │
  │ Auto memory (MEMORY.md)                            ~680  hidden  │
  │ Skill descriptions                                 ~450  yours   │
  │ ~/.claude/CLAUDE.md                                ~320  yours   │
  │ Environment info (cwd, platform, shell, git)       ~280  hidden  │
  │ MCP tool names (schemas deferred)                  ~120  hidden  │
  └──────────────────────────────────────────────────────────────────┘

  AS CLAUDE WORKS — this is where it actually goes
  ┌──────────────────────────────────────────────────────────────────┐
  │ Read src/api/auth.ts                             ~2,400          │
  │ Read middleware.ts                               ~1,800          │
  │ Read auth.test.ts                                ~1,600          │
  │ Read src/lib/tokens.ts                           ~1,100          │
  │ npm test output                                  ~1,200          │
  │ grep "refreshToken"                                ~600          │
  │ Rule: api-conventions.md (path-scoped, loaded      ~380          │
  │       because a matching file was read)                          │
  │ Rule: testing.md (same)                            ~290          │
  │ Edits, hook outputs, Claude's own reasoning      ~2,000          │
  └──────────────────────────────────────────────────────────────────┘
```

The shape of that is the lesson. **Your configuration is a rounding error; file reads and command output are the budget.** A 1,800-token `CLAUDE.md` is not what fills your window — four file reads are. This is why "delegate large reads to a subagent" beats "trim your `CLAUDE.md`" as a context strategy, even though the second is the advice people reach for first.

The corollary: a *bloated* `CLAUDE.md` still hurts, but through a different mechanism than you'd assume — not by consuming space, but by burying the rules that matter in noise so Claude stops following them. Part 5 covers that.

### 4.3 Working Knowledge: the four ways to free space

| Tool | What it does | When |
|---|---|---|
| `/clear` | Start a **new conversation** with empty context. The previous one is saved and resumable. | Switching to unrelated work |
| `/compact [instructions]` | Replace history with a **summary**, staying in the same conversation | Same task, too much accumulated detail |
| `/rewind` → **Summarize from here** / **up to here** | Compress **part** of the conversation | A verbose detour you want gone, without losing the rest |
| Subagents | Move the work into a **separate context window** | Research and exploration that produces a lot of reading |

The distinctions matter more than they look:

- **`/clear` vs. `/compact`** — `/clear` throws the conversation away (recoverably: `/resume`, or the rewind menu's previous-session entry in the same process). `/compact` keeps a summarised version. If the next task genuinely doesn't need what came before, `/clear` is strictly better — a summary you don't need still costs tokens on every subsequent request.
- **`/compact` with a focus beats bare `/compact`.** `/compact focus on the API changes` keeps what *you* choose instead of what the automatic pass guesses matters.
- **The rewind summarize options are the surgical version.** "Summarize from here" condenses everything after a chosen message while keeping earlier context intact; "Summarize up to here" does the reverse. Use the first when a long debugging tail is now irrelevant but your original instructions aren't; use the second when the recent work is what matters and the setup is done.
- **`/btw`** is the zero-cost option for questions that don't need to stay in context — the answer never enters conversation history.

**Automatic compaction.** Claude Code compacts on its own as you approach the limit, so a full window doesn't end your session. It clears older tool outputs first, then summarises the conversation if needed. Your requests and key code snippets are preserved; **detailed instructions from early in the conversation may be lost** ([how it works](https://code.claude.com/docs/en/how-claude-code-works)).

You can steer the automatic pass by adding a **"Compact Instructions"** section to `CLAUDE.md`:

```markdown
## Compact Instructions

When compacting, always preserve the full list of modified files, any test
commands that were run, and the decisions made about the migration strategy.
```

> **Wrong vs. right — the long session.**
>
> ```text
> # Wrong: keep going, let auto-compaction handle it, wonder later why
> # Claude forgot the constraint you gave it an hour ago.
> ```
>
> ```text
> # Right, before starting a big new phase in the same session:
> /compact focus on the auth refactor: which files changed, what the new
> token flow is, and the two edge cases we decided to punt on
> ```
>
> The right version costs you five seconds and produces a summary built around what you actually need next, instead of one built around what a summariser guessed.

**Compaction thrashing.** If a single file or tool output is so large that context refills immediately after each summary, Claude Code stops auto-compacting after a few attempts and shows an error rather than looping ([troubleshooting](https://code.claude.com/docs/en/troubleshooting#auto-compaction-stops-with-a-thrashing-error)). The fix is nearly always: stop reading that file into context, and have Claude grep it or process it in a subagent instead.

<a id="p4-survives"></a>

### 4.4 Advanced: what survives compaction

This table answers the most common "why did Claude forget that?" question in the tool ([context window](https://code.claude.com/docs/en/context-window)):

| Mechanism | After compaction |
|---|---|
| System prompt and output style | **Unchanged** — not part of message history |
| Project-root `CLAUDE.md` and unscoped rules | **Re-injected from disk** |
| Auto memory | **Re-injected from disk** |
| The plan Claude wrote in plan mode | **Re-injected from disk** |
| Rules with `paths:` frontmatter | Reloaded **when Claude next reads a matching file** |
| Nested `CLAUDE.md` in subdirectories | Reloaded **when Claude next reads a file in that subdirectory** |
| Files Claude read or edited | Claude Code **re-reads up to five**, most recently modified first |
| Invoked skill bodies | **Re-injected**, capped at **5,000 tokens per skill and 25,000 total**; oldest dropped first |
| Context that hooks added earlier | **Summarised** with the rest of the conversation |
| `SessionStart` hooks matching the `compact` source | **Run again**, output added to the compacted context |
| Anything you only said in chat | **Summarised, possibly away** |

Three practical readings of that table:

1. **"My `CLAUDE.md` rule stopped being followed after compaction" is usually not about `CLAUDE.md`.** Project-root `CLAUDE.md` is re-injected. If a rule disappeared, it was either given only in conversation, lives in a **nested** `CLAUDE.md` that hasn't reloaded yet, or is a **path-scoped rule** that hasn't matched a file since ([memory](https://code.claude.com/docs/en/memory)).
2. **Skill truncation keeps the start of the file.** Since re-injection caps each skill at 5,000 tokens and truncates from the end, **put the most important instructions near the top of `SKILL.md`**.
3. **A `SessionStart` hook with the `compact` matcher is the supported way to re-inject anything else** you need after every compaction — a live deployment state, an on-call rota, the current sprint's ticket list. Part 12 in [Extensibility](./claude-code-extensibility.md).

> ### Real Scenario — the constraint that evaporated
>
> A team is migrating a service off a deprecated internal HTTP client. Two hours into a session, the engineer told Claude in chat: *"Never use `LegacyHttpClient` in new code — it leaks connections. Use `HttpClient2` even where the surrounding file still uses the old one."*
>
> Ninety minutes and one auto-compaction later, Claude adds three new call sites using `LegacyHttpClient`, matching the surrounding file's style. The engineer is baffled — it followed the rule perfectly for an hour.
>
> **What happened:** the instruction lived only in conversation history. Auto-compaction summarised the conversation, and "use `HttpClient2` in new code" wasn't judged important enough to survive verbatim. Meanwhile the *surrounding code* — which uses the legacy client everywhere — was re-read after compaction as one of the five most-recently-modified files. So the strongest remaining signal pointed the wrong way.
>
> **The fix, in escalating order of reliability:**
> 1. Put it in `CLAUDE.md`, which is re-injected from disk after every compaction.
> 2. Better, since it's file-specific: put it in `.claude/rules/http-client.md` with `paths: ["src/**/*.ts"]`, so it reloads whenever Claude touches a matching file.
> 3. Best, since it's a hard rule: a `PreToolUse` hook that blocks an `Edit` whose new content introduces `LegacyHttpClient`. `CLAUDE.md` is context; a hook is enforcement ([features overview](https://code.claude.com/docs/en/features-overview)).
>
> The general form: **anything you find yourself repeating in chat is a `CLAUDE.md` entry, and anything that must hold every time is a hook.**

<a id="p4-thresholds"></a>

### 4.5 Advanced: tuning when compaction happens

By default, models compact at their context limit. You can move that point ([model config](https://code.claude.com/docs/en/model-config)):

```text
/autocompact 500k      # compact when the window hits 500,000 tokens
/autocompact auto      # back to the window tuned for your model
/autocompact           # dialog showing the current window
```

```bash
claude --autocompact 500k     # for this session only
export CLAUDE_CODE_AUTO_COMPACT_WINDOW=500000   # takes precedence
```

Accepted formats: a raw token count (`200000`), a suffix (`500k`, `1M`), or shorthand where `200` means 200,000. `/autocompact` saves the value to user settings and applies it to the current session (v2.1.221+).

**Why you would lower it:** compacting earlier means each request carries less history, so per-request cost stays lower and the model works in a less-crowded window. **Why you would not:** every compaction is a lossy summarisation, and more compactions means more accumulated loss.

**Extended context.** Fable 5.1, Fable 5, Sonnet 5, Opus 4.7+ and Sonnet 4.6 support a **1 million token** window. Some are native; some need an explicit variant:

```text
/model opus[1m]
/model sonnet[1m]
```

**Sonnet 5 is the special case** — it runs at 1M on the Anthropic API with no `[1m]` variant to select, and auto-compacts around ~967K by default. `CLAUDE_CODE_DISABLE_1M_CONTEXT=1` treats 1M-native models as 200K, which also changes compaction behaviour. Availability by plan differs (Max/Team/Enterprise include Opus 1M; Pro needs usage credits for it), so check [extended context](https://code.claude.com/docs/en/model-config#extended-context) before relying on it.

> A bigger window is not a substitute for context hygiene. Performance still degrades as the window fills — 1M tokens of accumulated debugging output is a worse working environment than 80K tokens of relevant material.

**Turning compaction off** is possible (`autoCompactEnabled: false`) and almost always wrong for interactive work; it exists for scripted runs where you would rather fail loudly than silently lose detail.

<a id="p4-mastery"></a>

### 4.6 Mastery: context cost per feature, and the deferral tricks

Every extension you add costs something. The loading strategies differ, and knowing them is what lets you have a rich setup without a crowded window ([features overview](https://code.claude.com/docs/en/features-overview)):

| Feature | When it loads | What loads | Cost |
|---|---|---|---|
| **`CLAUDE.md`** | Session start | Full content | **Every request** |
| **Skills** | Start + on use | Descriptions at start; full body when used | Low (descriptions every request) |
| **MCP servers** | Session start | Tool **names** only; schemas deferred | Low until a tool is used |
| **Code intelligence** | After edits, on demand | Diagnostics; symbol locations | Low — often *reduces* net context by replacing file reads |
| **Subagents** | When spawned | Their own fresh context | **Isolated** from yours |
| **Hooks** | On trigger | Nothing | **Zero**, unless the hook returns output |

Four deferral mechanisms worth knowing by name:

**1. `disable-model-invocation: true` on a skill.** By default, skill descriptions load at session start so Claude can decide when to use them. This flag hides the skill from Claude entirely until *you* invoke it with `/name` — zero context cost until then. Use it for anything with side effects that you want to trigger deliberately anyway.

**2. `skillOverrides` in settings**, for skills you didn't write:

```json
{
  "skillOverrides": {
    "legacy-context": "name-only",
    "internal-docs": "user-invocable-only",
    "deploy": "off"
  }
}
```

States are `"on"` (default), `"name-only"` (name without description), `"user-invocable-only"` (hidden from Claude), and `"off"`. In-session, `/skills` → highlight → `Space` cycles the states, `Esc` saves. `/skill-doctor` (v2.1.261+) identifies skills that are loaded but never used.

**3. MCP tool search**, on by default. Only names and server instructions load; schemas arrive on demand. `ENABLE_TOOL_SEARCH=auto` loads schemas upfront when they fit within 10% of the window; `ENABLE_TOOL_SEARCH=false` loads everything. Run `/context all` to see how many tokens each loaded MCP tool actually uses.

**4. Path-scoped rules**, covered in [§5.5](#p5-rules) — the only mechanism that makes *instructions* conditional.

**Two settings for output volume**, added in v2.1.261: `bashOutputMaxChars` and `taskOutputMaxChars` (up to 128K) control how much of a command's or task's output reaches Claude inline. Raise them when you keep truncating something Claude needs; lower them when a chatty test runner is eating your window. `BASH_MAX_OUTPUT_LENGTH` (default 30,000, max 150,000) is the environment-variable equivalent for Bash.

<a id="47-part-4-cheat-sheet"></a>

### 4.7 Part 4 cheat sheet

| Command | Effect |
|---|---|
| `/context` · `/context all` | Live usage grid · expanded per-item breakdown |
| `/clear [name]` | New empty conversation; optionally name the one you're leaving |
| `/compact [focus]` | Summarise history, optionally guided |
| `/rewind` → Summarize from/up to here | Compress part of the conversation |
| `/autocompact 500k` · `/autocompact auto` | Set / reset the auto-compact window |
| `/btw <question>` | Answer stays out of context |
| `/memory` | Open and edit the files that load every session |
| `/skill-doctor` | Find loaded-but-unused skills (v2.1.261+) |

| Survives compaction | Doesn't |
|---|---|
| System prompt, output style | Instructions given only in chat |
| Project-root `CLAUDE.md`, unscoped rules | Nested `CLAUDE.md` (until re-triggered) |
| Auto memory | Path-scoped rules (until re-matched) |
| The plan from plan mode | Skill bodies beyond 5K/skill, 25K total |
| Up to 5 recently-modified read files | Everything else, as a summary |

| Setting / var | Does |
|---|---|
| `autoCompactWindow`, `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | When auto-compaction fires |
| `autoCompactEnabled` | Turn it off (rarely right) |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT=1` | Treat 1M-native models as 200K |
| `ENABLE_TOOL_SEARCH=false` | Load all MCP schemas upfront |
| `bashOutputMaxChars`, `taskOutputMaxChars` | Inline output caps (≤128K) |
| `BASH_MAX_OUTPUT_LENGTH` | Bash output cap (default 30,000, max 150,000) |
| `skillOverrides` | Hide or collapse skills without editing them |

[↑ Back to top](#table-of-contents)

---

## Part 5 — Memory That Persists

<a id="part-5"></a>

Every session starts with a fresh context window. Two mechanisms carry knowledge across that boundary ([memory](https://code.claude.com/docs/en/memory)):

|  | `CLAUDE.md` files | Auto memory |
|---|---|---|
| **Who writes it** | You | Claude |
| **What it contains** | Instructions and rules | Learnings and patterns |
| **Scope** | Project, user, or org | Per repository, shared across worktrees |
| **Loaded into** | Every session | Every session (first 200 lines or 25KB) |
| **Use for** | Coding standards, workflows, architecture | Your preferences, your corrections, project context Claude can't derive from code |

**Both are context, not enforced configuration.** Claude reads them and tries to follow them; there is no compliance guarantee. To *block* an action regardless of what Claude decides, use a `PreToolUse` hook. This distinction is the single most important thing in Part 5 and it recurs throughout.

### 5.1 Beginner: `CLAUDE.md` and `/init`

`CLAUDE.md` is a Markdown file in your project root that Claude reads at the start of every session.

```text
/init
```

Claude analyses your codebase and writes a starting `CLAUDE.md` with build commands, test instructions, and the conventions it discovers. If one already exists, `/init` suggests improvements rather than overwriting. It also reads Cursor rules (`.cursor/rules/`, `.cursorrules`) and Copilot rules (`.github/copilot-instructions.md`) and folds the relevant parts in.

Setting `CLAUDE_CODE_NEW_INIT=1` enables an interactive multi-phase flow: it asks which artifacts to set up (`CLAUDE.md` files, skills, hooks), explores with a subagent, asks follow-up questions, and presents a reviewable proposal before writing anything. It also reads `AGENTS.md`, `.devin/rules/`, `.windsurf/rules/`, and `.clinerules`.

A minimal, good `CLAUDE.md`:

```markdown
# Code style
- Use ES modules (import/export) syntax, not CommonJS (require)
- Destructure imports when possible (eg. import { foo } from 'bar')

# Workflow
- Be sure to typecheck when you're done making a series of code changes
- Prefer running single tests, and not the whole test suite, for performance
```

Verify it loaded: run `/context` and check the list under **Memory files**. If it isn't there, Claude can't see it.

### 5.2 Working Knowledge: where `CLAUDE.md` files live and how they load

Four locations, listed in **load order — broadest scope first** ([memory](https://code.claude.com/docs/en/memory)):

| Scope | Location | For | Shared with |
|---|---|---|---|
| **Managed policy** | macOS `/Library/Application Support/ClaudeCode/CLAUDE.md`<br>Linux/WSL `/etc/claude-code/CLAUDE.md`<br>Windows `C:\Program Files\ClaudeCode\CLAUDE.md` | Org-wide standards, security policy, compliance | Everyone on the machine |
| **User** | `~/.claude/CLAUDE.md` | Personal preferences, all projects | Just you |
| **Project** | `./CLAUDE.md` **or** `./.claude/CLAUDE.md` | Team-shared project rules | Team, via version control |
| **Local** | `./CLAUDE.local.md` | Your personal project notes | Just you — **gitignore it** |

**The loading algorithm**, which is worth knowing precisely because it determines which instruction wins a conflict:

```text
  Launched in foo/bar/ :

  filesystem root
       │
       ▼
  foo/CLAUDE.md            ─┐
  foo/CLAUDE.local.md       │  loaded at LAUNCH, concatenated
  foo/bar/CLAUDE.md         │  in this order (root → cwd)
  foo/bar/CLAUDE.local.md  ─┘  ← read last, so "closest" wins in practice

  foo/bar/baz/CLAUDE.md    ← loaded ON DEMAND, when Claude first
  foo/bar/qux/CLAUDE.md       reads a file in that subdirectory
```

All discovered files are **concatenated, not overridden**. Within each directory, `CLAUDE.local.md` is appended after `CLAUDE.md`, so your personal notes are the last thing Claude reads at that level. Subdirectory files load lazily.

Two file facts that catch people:

- **Block-level HTML comments are stripped** before injection: `<!-- maintainer notes -->` costs you nothing in context. Comments inside code blocks are preserved, and everything is visible if Claude opens the file with the Read tool.
- **Size limits:** Claude Code loads a `CLAUDE.md` up to **4 MiB** in full and skips anything larger. That is not a target — see the next section.

**`--add-dir` does not load memory from the added directory** by default. To change that:

```bash
CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1 claude --add-dir ../shared-config
```

That loads `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`, and `CLAUDE.local.md` from the additional directory.

### 5.3 Working Knowledge: writing one that actually gets followed

The official guidance is unusually direct here, and it is the opposite of what most people's instinct produces ([best practices](https://code.claude.com/docs/en/best-practices)):

> *"Bloated `CLAUDE.md` files cause Claude to ignore your actual instructions!"*

**Target under 200 lines.** Longer files consume more context *and reduce adherence*. For each line, ask: **"Would removing this cause Claude to make mistakes?"** If not, cut it.

| ✅ Include | ❌ Exclude |
|---|---|
| Bash commands Claude can't guess | Anything Claude can figure out by reading code |
| Code style rules that **differ from defaults** | Standard language conventions Claude already knows |
| Testing instructions and preferred test runners | Detailed API documentation (link instead) |
| Repository etiquette (branch naming, PR conventions) | Information that changes frequently |
| Architectural decisions specific to your project | Long explanations or tutorials |
| Developer environment quirks (required env vars) | File-by-file descriptions of the codebase |
| Common gotchas or non-obvious behaviours | Self-evident practices like "write clean code" |

Three writing rules:

- **Specificity.** "Use 2-space indentation" beats "format code properly." "Run `npm test` before committing" beats "test your changes." "API handlers live in `src/api/handlers/`" beats "keep files organised."
- **Structure.** Markdown headers and bullets. Claude scans structure the way readers do; organised sections beat dense paragraphs.
- **Consistency.** If two rules contradict, Claude may pick one arbitrarily. Review your project `CLAUDE.md`, nested ones, and `.claude/rules/` periodically for stale and conflicting entries.

**Emphasis is a limited resource.** If Claude keeps skipping one instruction, add `IMPORTANT` to *that line alone*. Emphasise many lines and none of them stands out.

**Two diagnostics for when it isn't working:**

- If Claude keeps doing something despite a rule against it, the file is probably too long and the rule is getting lost.
- If Claude asks you questions that `CLAUDE.md` answers, the phrasing is ambiguous.

**`/doctor` will trim it for you.** For a checked-in `CLAUDE.md`, `/doctor` proposes cuts: it removes content Claude can derive from the codebase — directory layouts, dependency lists, architecture overviews — and keeps pitfalls, rationale, and conventions that differ from tool defaults. It reports findings and asks before changing anything (trim check requires v2.1.206+).

> **Wrong vs. right — the `CLAUDE.md` that stopped working.**
>
> ```markdown
> <!-- Wrong: 300 lines. Excerpt: -->
> ## Architecture
> The project is a monorepo with packages/api (Express server), packages/web
> (Next.js frontend), packages/shared (types and utils)...
> [40 more lines describing the directory tree]
>
> ## Code Quality
> - Write clean, readable code
> - Add comments where appropriate
> - Follow best practices
> - Use meaningful variable names
> ```
>
> ```markdown
> <!-- Right: the same file, trimmed to what Claude can't derive -->
> ## Commands
> - `pnpm dev` runs api + web together; `pnpm dev --filter web` for web only
> - `pnpm test -- <path>` for one test file. Never run the full suite locally,
>   it needs a Postgres container that CI provides and dev machines don't.
>
> ## Gotchas
> - `packages/shared` is built, not transpiled on the fly. After changing it,
>   run `pnpm build --filter shared` or the API silently uses stale types.
> - IMPORTANT: never edit `packages/api/src/generated/` — it is regenerated
>   from the OpenAPI spec by `pnpm codegen`.
> ```
>
> The wrong version describes what `ls` would tell Claude, and gives quality advice the model already holds. The right version is entirely non-derivable facts. It is a fifth the length and produces better behaviour.

### 5.4 Working Knowledge: auto memory

Auto memory lets Claude accumulate knowledge across sessions **without you writing anything**. It is on by default ([memory](https://code.claude.com/docs/en/memory)).

As it works, Claude saves four kinds of note, recorded as a `type` field in the memory file's frontmatter:

| Type | Contains |
|---|---|
| `user` | Your role, expertise, and working preferences |
| `feedback` | Corrections you gave Claude, and approaches you confirmed |
| `project` | Ongoing work, deadlines, and decisions Claude can't derive from code or git history |
| `reference` | Where to find information outside the project — an issue tracker, a dashboard |

Claude **skips** anything derivable from the codebase (architecture, file paths, debugging fixes) and anything your `CLAUDE.md` already says. It doesn't save every session; it decides what would be useful in a future conversation.

**Storage.** Each project gets `~/.claude/projects/<project>/memory/`, where `<project>` is derived from the **git repository** — so all worktrees and subdirectories of the same repo share one memory directory. Auto memory is machine-local and is **not** shared across machines or cloud environments.

```text
~/.claude/projects/<project>/memory/
├── MEMORY.md            # index, one line per memory — loaded every session
├── user_role.md         # one memory
├── feedback_testing.md  # one memory
└── ...
```

**How much loads:** the **first 200 lines of `MEMORY.md`, or the first 25KB, whichever comes first**, at the start of every conversation. Content beyond that is not loaded. Topic files are *not* loaded at startup — Claude reads them on demand with its normal file tools.

Claude Code enforces the limit actively: after a write, if `MEMORY.md` is near a limit it reminds Claude to shorten it (one line per entry, detail into topic files, merge or drop stale entries); if it's over, the write succeeds but Claude Code returns [an error telling Claude to rewrite the index](https://code.claude.com/docs/en/errors#memory-index-is-over-its-read-limit), because everything past the limit is dropped on the next load.

**Controls:**

```json
{
  "autoMemoryEnabled": false,
  "autoMemoryDirectory": "~/my-custom-memory-dir"
}
```

Or `/memory` in a session (toggle plus a "open the auto memory folder" option), or `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`. `autoMemoryDirectory` must be absolute or start with `~/`, and when set in a project settings file it is honoured under the same workspace-trust rule as hooks.

**Two details worth knowing:**

- Memory files are **excluded from the transcript retention sweep** — `MEMORY.md` and topic files stay until you or Claude edits or deletes them, regardless of `cleanupPeriodDays`.
- When Claude writes a memory file that begins with YAML frontmatter, Claude Code stamps a `modified` field with an ISO 8601 timestamp (v2.1.214+), so both you and Claude can see how current a fact is. Files without frontmatter never get one added.

**Auditing.** Everything is plain Markdown you can read, edit, or delete. Run `/memory` and select the auto memory folder. When you see "Saved 2 memories" or "Recalled 2 memories" in the interface, that's this directory.

**Subagents and auto memory:** the main conversation's auto memory is **not** loaded into subagents — the exception is a fork, which inherits the parent conversation and system prompt. A subagent can have its own separate memory directory via its `memory:` frontmatter field.

<a id="p5-rules"></a>

### 5.5 Advanced: `.claude/rules/` and path scoping

For anything beyond a small `CLAUDE.md`, rules are the mechanism that matters — because they are the only way to make **instructions conditional on what Claude is touching**.

```text
your-project/
├── .claude/
│   ├── CLAUDE.md           # main project instructions
│   └── rules/
│       ├── code-style.md   # loaded every session
│       ├── testing.md
│       └── frontend/
│           └── react.md    # discovered recursively
```

All `.md` files are discovered recursively. Rules **without** `paths:` frontmatter load at launch with the same priority as `.claude/CLAUDE.md`. User-level rules live in `~/.claude/rules/` and load **before** project rules, giving project rules higher priority.

**Path-scoped rules** are the payoff:

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API Development Rules

- All API endpoints must include input validation
- Use the standard error response format
- Include OpenAPI documentation comments
```

This loads **only when Claude works with files matching the pattern** — triggered when Claude *reads* a matching file, not on every tool use. As of v2.1.198, matching also works when Claude reaches a file through a symlinked path.

| Pattern | Matches |
|---|---|
| `**/*.ts` | All TypeScript files in any directory |
| `src/**/*` | All files under `src/` |
| `*.md` | Markdown files in the project root |
| `src/components/*.tsx` | React components in that specific directory |
| `src/**/*.{ts,tsx}` | Brace expansion works |

**The brace-expansion budget.** Each brace group multiplies the pattern count: `src/*.{ts,tsx}` expands to two, `{a,b}/{c,d}/*.{ts,tsx}` to eight. A rule's whole `paths` list shares a budget of **1,000 expanded patterns and 4 MiB**; patterns without braces don't count. A pattern exceeding the budget is used **unexpanded**, so its literal braces match nothing. (Before v2.1.217 a `paths` value with many brace groups could stall or crash the CLI at startup.)

**The `[` trap.** Glob syntax treats `[` as the start of a bracket expression. A pattern like `photos [2024/**` is invalid: it matches nothing, though the rule's other patterns keep working. Escape it: `photos \[2024/**`. (Before v2.1.207 one invalid pattern made the Read tool fail for *every* file the rule was evaluated against.)

**Sharing rules across projects** via symlinks — resolved and loaded normally, with circular symlinks detected:

```bash
ln -s ~/shared-claude-rules .claude/rules/shared
ln -s ~/company-standards/security.md .claude/rules/security.md
```

**Rules vs. skills.** Rules load every session, or when matching files are opened. Skills load only when invoked or when Claude judges them relevant. For task-specific instructions that don't need to be in context all the time, **use a skill**.

> **Wrong vs. right — the 400-line `CLAUDE.md`.**
>
> ```markdown
> <!-- Wrong: one file, everything loaded every session -->
> # CLAUDE.md (412 lines)
> ## General conventions ...
> ## React component rules ...        ← irrelevant when editing the Go API
> ## Go service rules ...             ← irrelevant when editing React
> ## Terraform module conventions ... ← irrelevant 95% of the time
> ## Our full REST API style guide ...← reference material, not a rule
> ```
>
> ```text
> Right: split by what triggers it
>
> CLAUDE.md                       ~60 lines: commands, gotchas, always-true rules
> .claude/rules/react.md          paths: ["apps/web/**/*.tsx"]
> .claude/rules/go.md             paths: ["services/**/*.go"]
> .claude/rules/terraform.md      paths: ["infra/**/*.tf"]
> .claude/skills/api-style/       invoked, or auto-loaded when relevant
>   SKILL.md
> ```
>
> Note what did *not* fix this: splitting the file with `@` imports. Imported files load at launch alongside the file that references them, so the context cost is identical — the docs say so explicitly. Only `paths:` scoping and skills actually defer the load.

<a id="p5-imports"></a>

### 5.6 Advanced: imports, `AGENTS.md`, and the trust dialog

**Imports.** `CLAUDE.md` files can pull in other files with `@path/to/import`:

```text
See @README for project overview and @package.json for available npm commands.

# Additional Instructions
- git workflow @docs/git-instructions.md
```

Rules:

- Relative paths resolve **relative to the file containing the import**, not the working directory.
- Imports can recurse, to a maximum depth of **four hops**.
- Import parsing **skips Markdown code spans and fenced code blocks** — write `` `@README` `` in backticks to mention a path without importing it.
- **Imports do not save context.** They are for organisation. Imported files are expanded and loaded at launch.

**Worktrees and `CLAUDE.local.md`.** A gitignored `CLAUDE.local.md` only exists in the worktree where you created it. To share personal instructions across worktrees, import from your home directory instead:

```text
# Individual Preferences
- @~/.claude/my-project-instructions.md
```

**The external-import trust dialog.** An import in a **project-level** memory file is *external* when its path resolves outside your working directory — like the home-directory import above. The first time Claude Code sees external imports in a project, it shows an approval dialog listing the files. **Decline and they stay disabled, and the dialog doesn't appear again.** This exists to protect you from files other people commit to a shared project.

User-scope memory files (`~/.claude/CLAUDE.md`, `~/.claude/rules/`) are files you wrote yourself, so their imports load without the dialog — except in Cowork sessions on the desktop, which skip user-scope imports resolving outside the session's working directory and skip a `~/.claude/CLAUDE.md` that is itself a symlink or hard link.

**`AGENTS.md`.** Claude Code reads `CLAUDE.md`, **not** `AGENTS.md`. If your repo already uses `AGENTS.md` for other agents, import it:

```markdown
@AGENTS.md

## Claude Code

Use plan mode for changes under `src/billing/`.
```

A symlink also works if you don't need Claude-specific content:

```bash
ln -s AGENTS.md CLAUDE.md
```

On Windows a symlink needs Administrator privileges or Developer Mode, so use the `@AGENTS.md` import instead. Either way, verify with `/context` → **Memory files**.

`/import [codex|gemini]` (v2.1.213+) brings configuration from OpenAI Codex or Google Gemini CLI — instruction files, MCP servers, commands, subagents, and skills — with `--dry-run` to preview and `--yes` to skip the picker.

<a id="p5-monorepo"></a>

### 5.7 Advanced: monorepos and excluding other teams' files

In a large monorepo, ancestor `CLAUDE.md` files from other teams get picked up automatically and add irrelevant instructions to every session. `claudeMdExcludes` skips them by path or glob:

```json
{
  "claudeMdExcludes": [
    "**/monorepo/CLAUDE.md",
    "/home/user/monorepo/other-team/.claude/rules/**"
  ]
}
```

Patterns are matched against **absolute** paths using glob syntax. The key can live at any settings layer (user, project, local, managed) and **arrays merge across layers**. Put it in `.claude/settings.local.json` when the exclusion is personal to your machine.

To exclude a rules file you reach through a symlink, a pattern matching **either** the path under `.claude/rules/` or the link target works (v2.1.239+; earlier versions only matched the link target).

**Managed policy `CLAUDE.md` cannot be excluded** — that's the point of it.

The recommended monorepo shape:

```text
monorepo/
├── CLAUDE.md                    ← genuinely global: monorepo tooling, PR rules
├── .claude/rules/
│   └── security.md              ← global, unscoped
├── apps/web/
│   ├── CLAUDE.md                ← loads when Claude reads files here
│   └── .claude/skills/          ← nested skills, namespaced on clash
└── services/api/
    └── CLAUDE.md
```

Nested `CLAUDE.md` files load on demand, so a session working in `apps/web/` never pays for `services/api/CLAUDE.md`. See [large codebases](https://code.claude.com/docs/en/large-codebases) for the full layout guidance.

<a id="p5-mastery"></a>

### 5.8 Mastery: memory is context, not configuration

The most important structural fact in this Part, restated at the level it deserves:

```text
   CONTEXT (advisory)                    ENFORCEMENT (deterministic)
   ────────────────────                  ───────────────────────────
   CLAUDE.md                             permissions.deny rules
   .claude/rules/                        PreToolUse hooks
   auto memory                           sandbox.enabled
   skills                                managed settings
   output styles

   Claude reads these and tries          Claude Code applies these
   to comply. Compliance is not          regardless of what Claude
   guaranteed, especially for            decides. The trigger is
   vague or conflicting rules.           guaranteed.
```

`CLAUDE.md` content is delivered as **a user message after the system prompt**, not as part of the system prompt itself ([memory](https://code.claude.com/docs/en/memory)). That's why there's no compliance guarantee, and why the debugging checklist for "Claude isn't following my `CLAUDE.md`" is:

1. Run `/context`, check **Memory files** — did it actually load?
2. Is the file in a location that loads for this session's working directory?
3. Is the instruction specific enough to verify?
4. Is another `CLAUDE.md` or rule contradicting it?
5. Is the file too long, so the rule is buried?

And the escalation, when it must hold every time:

| Need | Mechanism |
|---|---|
| Claude should usually do X | `CLAUDE.md` |
| Claude should do X when working on certain files | `.claude/rules/` with `paths:` |
| Claude should do X when it decides the situation applies | A skill |
| X must happen at a specific lifecycle point, every time | A **hook** |
| Claude must never be able to do Y | `permissions.deny`, or a `PreToolUse` hook |
| I want it at system-prompt level | `--append-system-prompt` (must be passed every invocation — better for scripts than interactive use) |

**The `InstructionsLoaded` hook** is the debugging tool for all of this: it logs exactly which instruction files loaded, when, and why — invaluable for path-specific rules and lazily-loaded subdirectory files ([hooks](https://code.claude.com/docs/en/hooks#instructionsloaded)).

**Managed `CLAUDE.md` for organisations.** Deploy a file at the managed policy location, or inline the content in `managed-settings.json`:

```json
{
  "claudeMd": "Always run `make lint` before committing.\nNever push directly to main."
}
```

It loads before user and project `CLAUDE.md`, applies to every session on the machine in every repository, and cannot be excluded. `claudeMd` is honoured **only** in managed and policy settings — setting it in user, project, or local settings does nothing. The division of labour the docs recommend:

| Concern | Configure in |
|---|---|
| Block specific tools, commands, or paths | Managed settings: `permissions.deny` |
| Enforce sandbox isolation | Managed settings: `sandbox.enabled` |
| Env vars and API routing | Managed settings: `env` |
| Login method and org restrictions | Managed settings: `forceLoginMethod`, `forceLoginOrgUUID` |
| Code style and quality guidelines | Managed `CLAUDE.md` |
| Data handling and compliance reminders | Managed `CLAUDE.md` |

<a id="59-part-5-cheat-sheet"></a>

### 5.9 Part 5 cheat sheet

| Location | Scope |
|---|---|
| `/Library/Application Support/ClaudeCode/CLAUDE.md` (macOS)<br>`/etc/claude-code/CLAUDE.md` (Linux/WSL)<br>`C:\Program Files\ClaudeCode\CLAUDE.md` (Windows) | Managed policy — can't be excluded |
| `~/.claude/CLAUDE.md` · `~/.claude/rules/*.md` | You, every project |
| `./CLAUDE.md` or `./.claude/CLAUDE.md` · `./.claude/rules/*.md` | Project, checked in |
| `./CLAUDE.local.md` | You, this project — gitignore it |
| `~/.claude/projects/<project>/memory/MEMORY.md` | Auto memory index |

| Command | Effect |
|---|---|
| `/init` | Generate or improve `CLAUDE.md` (`CLAUDE_CODE_NEW_INIT=1` for the interactive flow) |
| `/memory` | List and edit memory files; toggle auto memory; open the auto memory folder |
| `/context` | Verify what actually loaded, under **Memory files** |
| `/doctor` | Propose trims for a checked-in `CLAUDE.md`; dedupe local vs. checked-in |
| `/import codex\|gemini` | Bring config from another coding agent (v2.1.213+) |

| Setting / var | Effect |
|---|---|
| `claudeMdExcludes` | Skip specific `CLAUDE.md` files (glob, absolute paths, merges across layers) |
| `claudeMd` | Inline org-wide instructions — **managed/policy settings only** |
| `autoMemoryEnabled` · `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` | Turn auto memory off |
| `autoMemoryDirectory` | Move the auto memory directory |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` | Load memory from `--add-dir` directories |

| Limit | Value |
|---|---|
| `CLAUDE.md` recommended size | **Under 200 lines** |
| `CLAUDE.md` hard limit | 4 MiB (larger files are skipped) |
| `MEMORY.md` loaded per session | First 200 lines **or** 25KB |
| Import depth | 4 hops |
| `paths:` brace expansion budget | 1,000 patterns / 4 MiB per rule |
| Skill re-injection after compaction | 5,000 tokens per skill, 25,000 total |

[↑ Back to top](#table-of-contents)

---

**Next:** [Config & Settings (Parts 6–7)](./claude-code-config-settings.md) — the five-layer settings chain, the key inventory, and model/interface configuration.
