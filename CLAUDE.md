# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A personal collection of long-form, reusable reference docs (Markdown only — no source code, no build/test/lint tooling). Each topic gets its own folder with one or more "mastery guide" documents meant to be searched and reopened repeatedly, not read once and discarded. There is nothing to build or run here; the only "commands" are file edits.

## Structure

```
topic-name/
  topic-mastery-guide.md   (top-level overview, or sole doc for the topic)
  topic-subarea-a.md       (large topics split into focused sub-docs)
  topic-subarea-b.md
```

Some topics (e.g. `mysql`, `postgres`) are split into a `*-mastery-guide.md` overview plus focused sub-docs (`*-foundations.md`, `*-core-querying.md`, `*-data-modeling-types.md`, `*-query-engine-indexing.md`, `*-transactions-concurrency.md`, `*-administration-operations.md`). Smaller topics (e.g. `docker`, `javascript`) are a single mastery guide.

`README.md` maintains a top-level index table (topic → doc path → last updated). **Update this table whenever a doc is added or substantially revised** — it's the entry point for finding docs.

## Doc conventions (established by existing docs — follow them when creating or editing a doc)

- **"About This Document" spec section** near the top of every doc, capturing: the framing (goal-driven vs. curiosity-driven), confirmed scope, depth tiers used, assumed background, per-section ingredients used, sourcing rule, and formatting choices. This section exists so a doc can be updated later *without* the user re-explaining the style — always read it before editing a doc, and keep it in sync with any structural changes you make.
- **Depth tiers**: major sections are typically structured as four progressive, explicitly labeled tiers — Beginner → Working Knowledge → Advanced → Mastery — collapsing tiers that would be trivial/redundant for a given topic (a purely conceptual topic may skip tiers entirely).
- **Per-section ingredients**, used only where they add real value (not forced into every subsection): a short plain-language explanation; a "wrong vs. right" code/command/config snippet pair for gotchas; a "Real Scenario" (production incident for goal-driven docs) or a runnable "Try It" exercise (curiosity-driven docs); an ASCII diagram where the concept is spatial/structural.
- **Sourcing**: cite primary sources inline as links — official docs first, Stack Overflow for real-world gotchas/common mistakes, GitHub/other reputable sites only as supplementary. When official docs and a popular-but-outdated convention disagree, prefer official docs and say so explicitly.
- **Accuracy**: search current official docs before writing anything version-specific (commands, flags, config syntax, deprecations); flag where practice has notably changed in recent versions.
- **Formatting**: clickable table of contents with anchor links at the top; "back to top" links after each major section; quick-reference/cheat-sheet tables at the end of major parts; fenced code blocks with language tags.

## Generating a new doc or extending an existing one

`prompt.md` is the canonical prompt template for generating a doc in this repo's style (a newer, tiered version of the template embedded in `README.md`). When asked to create or substantially extend a topic doc:

1. Follow the scoping flow in `prompt.md`: ask which framing applies (goal-driven vs. curiosity-driven) if unspecified, ask about sub-area focus for broad topics, propose a section structure adapted to the topic (tiers are the default progression, not a forced template), and confirm scope before writing the full doc — always pairing open questions with a sensible labeled default.
2. When extending an existing doc, read its "About This Document" section first and match its established framing/scope/tier choices rather than re-deriving style from scratch.
3. Save the result as a Markdown file in the appropriate topic folder, include an "About This Document" section, and add/update the row in `README.md`'s index table.

Note: `README.md`'s index currently references a `react-nextjs/react-concepts-before-nextjs.md` doc that does not exist in the repo — verify before assuming any indexed doc is present.
