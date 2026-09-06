# Mastery Guide Prompt Template

Copy this into a new chat, fill in the bracketed placeholders, and delete any section
that doesn't apply.

---

Act as a Senior [ROLE, e.g. "DevOps Engineer" / "Backend Developer" / "Data Engineer"]
mentoring a junior dev. I want to master [TOPIC, e.g. "Docker", "Git", "SQL indexing"].

Produce a detailed reference document I can come back to repeatedly, not read once and
discard.

## 1. Framing — why I'm learning this

Pick one, or tell me which applies:

- **Goal-driven**: I'm learning this before [GOAL, e.g. "working on our deployment
  pipeline"] — tailor depth, examples, and prioritization toward the concrete commands,
  APIs, flags, and config I'll actually be touching for that goal. Stay technical and
  hands-on, not just topically relevant.
- **Curiosity-driven**: I want to understand [TOPIC] for its own sake — no upcoming task
  driving it. Don't assume a deadline, but also don't default to a historical/evolution
  narrative. Structure it as a guided, hands-on tutorial — a sequence of runnable
  examples and exercises that build technical fluency — and bring in theory or "why it's
  designed this way" only at the points where it's needed to understand a mechanism,
  never as a standalone history section.

If I don't specify, **ask which framing fits before scoping further** — it changes what
"properly covered" means (goal-driven optimizes for the task ahead; curiosity-driven
optimizes for durable, hands-on understanding).

## 2. Scoping — do this before writing anything

1. **Sub-area focus.** If [TOPIC] is broad enough to have distinct sub-areas (e.g.
   "Kubernetes" could mean cluster architecture, workload types, or networking; "Git"
   could mean daily-use commands vs. internals), ask which angle to focus on — and
   whether this should be one document or an overview plus focused sub-docs.
2. **Assumed background.** Ask what I already know that's adjacent (the underlying
   language, the CLI, the protocol, a competing tool). The answer decides whether the
   doc opens with a primer section or dives straight in — say which you're planning.
3. **Proposed structure.** Propose the section breakdown you plan to use, adapted to how
   this topic actually decomposes — a command-line tool, a language feature, and an
   architectural concept all warrant different shapes. Don't force a fixed template.
4. **Everything else ambiguous.** For any other open question with more than one
   reasonable answer (how deep to go, which sub-topics to prioritize if space is limited,
   whether to cover a related tool), ask it — but **always pair the question with your own
   sensible default, clearly labeled**, so I can reply "go with your defaults" instead of
   answering everything myself.

**Wait for my confirmation on scope and structure before writing the full document.** For
the smaller open questions, proceed with your stated defaults if I don't address them.

## 3. Depth tiers

Structure the document — or each major section, whichever fits the topic — as four
progressive, explicitly labeled tiers, so I can stop at whichever level I need:

- **Beginner** — what it is, minimal correct usage
- **Working Knowledge** — everyday usage, common flags/patterns, enough to be productive
- **Advanced** — production concerns, edge cases, performance/security implications
- **Mastery** — internals, less-common but powerful capabilities, expert-level tradeoffs

Tiers are the default progression, not a mandate. Collapse a tier that would be trivial
or redundant for a given section, and say in the doc that you did (e.g. "Compose gets
Beginner → Advanced only; its 'Mastery' material is really Part 5's internals applied").
A purely conceptual topic may not need tiers at all.

## 4. Depth over breadth

Prioritize a well-chosen set of concepts covered *properly* over exhaustive coverage of
everything. If something is genuinely important but out of scope, list it in a short
"Deliberately not covered (and where to look instead)" note rather than half-covering it.

## 5. Per-section ingredients

Use these where they genuinely add value — a one-line concept doesn't need a forced
scenario, and a purely conceptual topic may not need code at all:

- A short plain-language explanation of what it is and why it matters
- A code/command/config snippet — prefer a **"wrong vs. right" pair** when illustrating a
  common mistake, not just the correct version in isolation
- A **"Real Scenario"** — a concrete bug, incident, or gotcha where this actually bites
  someone. Goal-driven docs: production-style situations. Curiosity-driven docs: a
  runnable **"Try It"** I can execute, including what correct output looks like
- A **diagram** (ASCII) where the concept is fundamentally spatial or structural —
  architecture, data flow, process/lifecycle order — and prose alone would obscure it

## 6. Accuracy and currency

- Search current official docs before writing anything version-specific: commands, flags,
  config syntax, defaults, deprecations, current best practice.
- State the versions the doc was written against, and date it.
- **Flag anywhere practice has notably changed in recent versions** — that's exactly where
  outdated blog posts mislead people. Name the outdated convention explicitly so I
  recognize it when I run into it in the wild.
- If you're unsure whether something is still current, say so inline rather than asserting
  it.

## 7. Sourcing

- Cite primary sources **inline as links**, at the specific claim they support.
- Lead with **official docs** (specs, API references, guides) for the technology itself,
  and **Stack Overflow** for real-world gotchas, common mistakes, and how practitioners
  actually resolve edge cases.
- Use GitHub issues/READMEs and other reputable technical sites only as supplementary
  sources, where docs and Stack Overflow don't cover a real-world pattern.
- When official docs and a popular-but-outdated convention disagree, **prefer the official
  docs and say so explicitly**.
- Only link URLs you have actually retrieved. Never construct a plausible-looking doc URL
  from memory.

## 8. Formatting

Make it easy to *reference*, not just read:

- A clickable table of contents with anchor links at the top
- "Back to top" links after each major section
- Explicit `<a id="..."></a>` anchors where headings repeat across sections (e.g. four
  parts that each contain an "Advanced" heading), so TOC links don't collide
- Quick-reference / cheat-sheet tables at the end of each major part
- Code and commands fenced with proper language tags
- End with a **Suggested Learning/Reference Order** and a **Quick Self-Check** list of
  questions

## 9. Output and repo integration

- Save as a Markdown file at `topic-name/topic-mastery-guide.md`. If the topic was split,
  use `topic-name/topic-<subarea>.md` alongside the overview guide.
- Add or update the topic's row in the index table in `README.md`, including the
  last-updated date.
- Cross-link to sibling docs in this repo where a prerequisite or adjacent topic is
  already covered, instead of re-explaining it.

## 10. "About This Document" spec section

Include this near the top of the doc, capturing the decisions above so a later update
doesn't require me to re-explain the style:

- Framing (goal-driven or curiosity-driven, and the goal if any)
- Confirmed scope, including what was deliberately excluded
- Depth tiers used, and where tiers were collapsed and why
- Assumed background
- Which per-section ingredients are in use
- Sourcing rule
- Formatting choices
- Versions written against, and the date
- A short **"To update this doc later"** note: what to preserve, and what to re-verify
  against current docs first

## 11. If you're extending an existing doc rather than creating one

Read its "About This Document" section first and match the framing, scope, and tier
choices already established there — don't re-derive the style. Prefer adding a new part
or tier over restructuring what's there. Update that section and the `README.md` row if
the structure or scope changes.
