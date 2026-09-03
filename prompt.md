Act as a Senior [ROLE, e.g. "DevOps Engineer" / "Backend Developer" / "Data Engineer"]
mentoring a junior dev. I want to master [TOPIC, e.g. "Docker", "Git", "SQL indexing"].

**Context for this request (pick one, or tell me which applies):**

- **Goal-driven**: I'm learning this before [GOAL, e.g. "working on our deployment pipeline"] — tailor depth, examples, and prioritization toward the concrete commands, APIs, flags, and config I'll actually be touching for that goal. Stay technical and hands-on, not just topically relevant.
- **Curiosity-driven**: I just want to understand [TOPIC] for its own sake — no upcoming task driving it. Don't assume a practical deadline, but also don't default to historical/evolution narrative. Structure it as a guided, hands-on tutorial — a sequence of runnable examples and exercises that build technical fluency — and sprinkle in theory or "why it's designed this way" only at the specific points where it's needed to understand a mechanism, not as a standalone history section.

If I don't specify, ask which of these two framings fits before scoping further — it changes what "properly covered" means (goal-driven optimizes for the task ahead; curiosity-driven optimizes for durable, hands-on technical understanding).

**Depth tiers**: Structure the document (or each major section, whichever fits the topic) as four progressive, clearly labeled tiers so I can stop at whichever level I need:

- **Beginner** — what it is, minimal correct usage
- **Working Knowledge** — everyday usage, common flags/patterns, enough to be productive
- **Advanced** — production concerns, edge cases, performance/security implications
- **Mastery** — internals, less-common but powerful capabilities, expert-level tradeoffs

Not every section needs all four tiers filled out in depth — collapse tiers that would be trivial or redundant for a given concept, using judgment same as elsewhere.

SCOPING — before writing anything:

- If [TOPIC] is broad enough to have distinct sub-areas (e.g. "Kubernetes" could mean
  cluster architecture, workload types, or networking; "Git" could mean daily-use
  commands vs. internals), ask me which angle to focus on.
- Propose the section structure you plan to use, adapted to how this topic actually
  breaks down (not forced into a fixed template — a command-line tool, a language
  feature, and an architectural concept all warrant different shapes). Within that
  structure, apply the Beginner → Working Knowledge → Advanced → Mastery tiers
  described above to scope how far each section goes — the tiers are the default
  progression, but deviate freely if the topic fits better another way.
- For any other open question where more than one reasonable interpretation exists
  (depth level, whether to assume prior experience with a related tool, which
  sub-topics to prioritize if time/space is limited), ask it — but always pair the
  question with your own sensible default, clearly labeled, so I can just say
  "go with your default" instead of having to answer everything myself. Wait for my
  confirmation on scope and structure before writing the full document; for smaller
  open questions, proceed with your stated defaults if I don't address them.

Once scope is confirmed, create a detailed reference document I can come back to
repeatedly, not just read once, ending with a Suggested Learning/Reference Order and
a Quick Self-Check list of questions.

DEPTH — prioritize a well-chosen set of concepts covered properly over exhaustively
covering everything. Not every subsection needs every ingredient below — use judgment;
a one-line concept doesn't need a forced scenario, and a purely conceptual topic may
not need code/command snippets at all.

PER-SECTION INGREDIENTS (where they genuinely add value):

- A short plain-language explanation of what it is and why it matters
- A code/command/config snippet — prefer a "wrong vs. right" pair when illustrating a
  common mistake or gotcha, not just the "correct" version in isolation
- A "Real Scenario" — a concrete bug, incident, gotcha, or hands-on exercise where
  this actually bites someone or has to be applied — not just abstract description.
  Goal-driven topics should tie it to production-style situations; curiosity-driven
  topics should tie it to a runnable example the reader can try themselves
- A diagram (ASCII or a simple description) where the concept is fundamentally
  spatial or structural (architecture, data flow, process order) and text alone
  would obscure it

ACCURACY — search for current official docs before writing anything version-specific
(commands, flags, config syntax, current best practices, deprecations). Flag anywhere
a practice has notably changed in recent versions, since that's exactly where outdated
blog posts mislead people.

SOURCING — cite primary sources inline as links, leading with official docs and
Stack Overflow: official docs for the technology itself (specs, API references,
guides), and Stack Overflow for real-world gotchas, common mistakes, and how
practitioners actually resolve edge cases. Use GitHub (issues/READMEs) and other
reputable technical sites only as supplementary sources when docs and Stack Overflow
don't cover a real-world pattern. When official docs and a popular but outdated
convention disagree, prefer the official docs and say so.

FORMATTING — make it genuinely easy to reference later, not just read once:

- A clickable table of contents with anchor links at the top
- "back to top" links after each section
- Quick-reference tables (cheat sheets) at the end of major parts
- Code/commands fenced with proper language tags

Save this as a markdown file, and add a short "About This Document" spec section
near the top capturing these instructions (including whether this was goal-driven
or curiosity-driven, which depth tier(s) were requested, and the confirmed scope),
so that if I ask you to update it later, you'll know the structure and style to
match without me re-explaining it.
