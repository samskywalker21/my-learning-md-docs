# Roadmap Prompt Template

Copy this into a new chat (run from the repo root so the existing docs can be read), fill
in the bracketed placeholders, and delete any section that doesn't apply.

Where [`prompt.md`](./prompt.md) produces a **mastery guide** that teaches one topic, this
template produces a **roadmap**: a prioritized checklist for getting from one topic to a
related one. It teaches nothing itself. Each item points into the mastery guides (or
official docs) that do the teaching.

---

Act as a Senior [ROLE, e.g. "Frontend Engineer" / "Backend Developer"] mentoring a junior
dev. I want a roadmap from **[FROM TOPIC, e.g. "JavaScript"]** to **[TO TOPIC, e.g.
"TypeScript"]**.

Produce a checklist I can work through and come back to, not an explainer.

## 1. Pair type: what kind of move this is

The two kinds need different roadmaps. Pick one, or tell me which applies:

- **Prerequisite → dependent**: [FROM] is a foundation [TO] is built on (JavaScript →
  TypeScript, CSS → Tailwind CSS, React → Next.js). The roadmap answers *which parts of
  [FROM] do I actually need before starting [TO], in what order, and what can wait?*
- **Lateral / adjacent**: I already know [FROM], and [TO] solves similar problems
  differently (React → Vue, Node.js → Bun, MySQL → Postgres). The roadmap answers
  *what carries over, what's different, what do I need to unlearn, and what's new
  in [TO]?*

If the pair could be read either way, or I don't say, **ask which one before scoping
further**.

## 2. Scoping: do this before writing anything

1. **Inventory the repo.** Read `README.md`'s index and list the docs in this repo
   that cover [FROM] and [TO] (current docs and anything under `legacy/`). Show me
   what you found.
2. **Missing docs: stop and ask.** If [FROM], [TO], or a sub-area the roadmap clearly
   needs has **no current (non-legacy) doc** in this repo, stop and ask me what to do
   before going on. Offer these options and recommend one:
   - **(a) Generate it first**: write the mastery guide with [`prompt.md`](./prompt.md),
     then come back and build the roadmap on top of it.
   - **(b) Use the legacy doc**, if one exists under `legacy/`, linked with a
     `⚠️ legacy` marker and a note on what is likely out of date.
   - **(c) Use official docs for now**: link retrieved official docs for those items and
     list the missing doc under **Gaps in This Repo** so it can be generated later.
3. **My starting point.** Ask how well I know [FROM] (just started / productive /
   comfortable with the advanced material). For lateral moves, also ask how much of
   [TO] I've already touched. Default: *productive in [FROM], new to [TO]*.
4. **My goal with [TO].**
   - **Goal-driven**: I'm moving to [TO] for [GOAL, e.g. "a Next.js project starting next
     month"]. Weight **Must** items toward what that goal touches, and move the rest to
     Should or Skip-for-now.
   - **Curiosity-driven**: no deadline. Cover what gives durable understanding of [TO],
     still ordered so I can start hands-on work early.

   Default: *curiosity-driven*.
5. **Time budget** (optional). If I give one (e.g. "~5 hours a week"), include rough
   per-phase time estimates. Default: *no estimates*.
6. **Proposed outline.** Propose the phase breakdown (phase names, 1–2 lines each, and
   roughly how many items in each) based on how the two topics actually connect. Don't
   force a fixed number of phases.

Pair every open question with your own **clearly labeled default**, so I can reply "go
with your defaults". **Wait for my confirmation on pair type, gap handling, and the
outline before writing the roadmap.**

## 3. Prioritization

Label every checklist item with how much [TO] actually depends on it:

- **🔴 Must**: [TO] is blocked or will clearly mislead me without it. Learn it before
  or during the phase it appears in.
- **🟡 Should**: makes [TO] noticeably easier or avoids a known gotcha. Can be learned
  alongside [TO].
- **⚪ Skip-for-now**: real [FROM] material that [TO] doesn't need yet. Collected into an
  explicit **Safe to Skip Before Starting [TO]** list with a one-line reason each, so I
  don't feel I have to master all of [FROM] first.

Order phases so I can **start real work in [TO] as early as possible**: Must items first,
Should items spread through later phases where they become relevant. Getting me into
[TO] early matters more than a complete tour of [FROM].

For **lateral** roadmaps, group items under these labels inside each phase where they
apply:

- **✅ Carries over**: works the same way. One line and a link, no re-learning.
- **🔀 Different**: same goal, different mechanism. Say what [FROM] habit this replaces.
- **🚫 Unlearn**: a [FROM] habit that is actively wrong or harmful in [TO]. Use a
  short "[FROM] habit vs. [TO] way" contrast. This is the one place a small snippet
  pair is allowed.
- **🆕 New**: no equivalent in [FROM].

## 4. Checklist item format

Each item is a single checkbox with exactly three parts:

```markdown
- [ ] 🔴 **Closures and lexical scope**: TypeScript's narrowing and callback typing assume you
  can already predict what a closure captures.
  → [JavaScript — Functions & Closures § Closures](../javascript/javascript-functions-closures.md#closures)
```

1. **The concept**, named the way the linked doc names it
2. **Why it matters for [TO]**, in one line. Say how it connects to [TO], not what it is
   in general.
3. **The pointer**: a link to the exact section that teaches it

Don't explain the concept itself. If you want to write a paragraph about an item, that
content belongs in a mastery guide (see §2, option a).

## 5. Linking rules

- **Link to section anchors, not just files**, using relative paths from `roadmaps/`
  (e.g. `../typescript/typescript-generics.md#...`).
- **Verify every anchor.** Open the target doc and confirm the heading or explicit
  `<a id="..."></a>` exists before linking to it. Anchors follow GitHub's slug rules:
  lowercase, punctuation removed except hyphens, spaces become hyphens, and repeated
  headings get `-1`, `-2` suffixes. Explicit `<a id>` anchors are safer, so use one
  wherever the doc provides it. **Never guess an anchor.**
- Prefer the **current** doc over a `legacy/` one whenever both cover the item. Only link
  a legacy doc if I chose option (b), and mark it `⚠️ legacy`.
- **External links** (official docs first, Stack Overflow for gotchas) only for items no
  repo doc covers, or when I chose option (c). Only link URLs you actually retrieved.
  Never construct a doc URL from memory.
- If a repo doc and current official docs disagree on something version-specific, link
  the official docs, say so in the item, and add the repo doc to **Gaps in This Repo** as
  needing a refresh.

## 6. Readiness checks

End each phase with a **Ready When You Can…** block of 2–4 concrete, testable statements.
At least one should be a small hands-on exercise with a clear success condition:

```markdown
**Ready when you can…**
- [ ] Explain why `this` is `undefined` inside a detached method, without looking it up
- [ ] Write a `debounce(fn, ms)` that preserves `this` and arguments, and say what it closes over
- [ ] Read an unfamiliar async function and name the order its logs print in
```

Checking every item box is not enough to move on. The readiness checks are the gate.

## 7. Output structure

```markdown
# [FROM] → [TO] Roadmap

One paragraph: pair type, who this is for, and where it ends (e.g. "ready to start the
TypeScript mastery guide from Part 4").

## About This Roadmap
## Table of Contents
## How to Use This Roadmap        ← legend for 🔴/🟡/⚪ (and ✅/🔀/🚫/🆕), how to tick boxes
## Map                            ← ASCII diagram of phases, and how they feed into [TO]'s docs
## Phase 1: <name>
   _Started: ____ · Finished: _____
   (checklist items)
   **Ready when you can…**
   [↑ Back to top]
## Phase 2: <name>
   …
## Safe to Skip Before Starting [TO]
## Where to Go Next               ← the [TO] mastery guide Part/section to continue with
## Gaps in This Repo              ← omit if there are none
## Quick-Reference Table          ← every item in one table: concept · priority · phase · link
```

- Clickable table of contents with anchor links, and a **back to top** link after each
  phase
- Put the **Started / Finished** date lines directly under each phase heading, left blank
  for me to fill in
- Use fenced code blocks with language tags for the few snippets allowed (lateral 🚫
  Unlearn contrasts only)

## 8. "About This Roadmap" spec section

Include this near the top, so the roadmap can be refreshed later without me
re-explaining anything:

- **Pair type** (prerequisite → dependent, or lateral) and direction
- **Framing** (goal-driven with the goal, or curiosity-driven)
- **Assumed starting point** in [FROM] (and [TO], for lateral)
- **Confirmed scope**: what the roadmap covers and where it hands off to the [TO] docs
- **Gap decisions**: for each missing doc, which option (a/b/c) was chosen
- **Linked docs and their "Last Updated" dates** as listed in `README.md` when the
  roadmap was written, plus the versions those docs are written against
- **Time budget**, if one was given
- **Date written**
- **To update this roadmap later**:
  - *Preserve*: the phases, the priority labels, and the item format.
  - *Re-verify first*: **if any linked doc's Last Updated date in `README.md` is newer
    than the date recorded here, re-check every anchor into that doc**, since mastery
    guide revisions rename and renumber sections. Also check whether a doc listed under
    Gaps in This Repo has since been written (if so, replace the external links with it).

## 9. Output and repo integration

- Save as `roadmaps/<from>-to-<to>-roadmap.md` using the topic folder names from this repo
  where they exist (e.g. `roadmaps/javascript-to-typescript-roadmap.md`,
  `roadmaps/react-to-nextjs-roadmap.md`).
- Add a row to the **Roadmaps** table in `README.md` with the from/to topics, pair type,
  path, and last-updated date.
- If any linked mastery guide doesn't already cross-reference the other topic, **mention
  it to me** rather than editing that guide unasked.

## 10. If you're updating an existing roadmap

Read its **About This Roadmap** section first and keep the pair type, framing, and phase
structure it records. Re-verify anchors per §8, move items between priorities if my
starting point or goal has changed, and update the About section and the `README.md`
row.
