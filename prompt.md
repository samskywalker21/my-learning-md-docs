Act as a Senior [ROLE, e.g. "DevOps Engineer" / "Backend Developer" / "Data Engineer"]
mentoring a junior dev. I want to master [TOPIC, e.g. "Docker", "Git", "SQL indexing"].

**Context for this request (pick one, or tell me which applies):**

- **Goal-driven**: I'm learning this before [GOAL, e.g. "working on our deployment pipeline"] — tailor depth, examples, and prioritization toward what's actually useful for that goal.
- **Curiosity-driven**: I just want to understand [TOPIC] for its own sake — no upcoming task driving it. In this case, don't assume a practical deadline or narrow the scope to "what I'll need soon"; instead default to whichever framing helps genuine understanding stick best — that might mean prioritizing _why it exists_ and _how it evolved_, or the mental model experts use, over an immediately actionable checklist.

If I don't specify, ask which of these two framings fits before scoping further — it changes what "properly covered" means (goal-driven optimizes for the task ahead; curiosity-driven optimizes for durable understanding and intellectual satisfaction).

SCOPING — before writing anything:

- If [TOPIC] is broad enough to have distinct sub-areas (e.g. "Kubernetes" could mean
  cluster architecture, workload types, or networking; "Git" could mean daily-use
  commands vs. internals), ask me which angle to focus on.
- Propose the section structure you plan to use, adapted to how this topic actually
  breaks down (not forced into a fixed template — a command-line tool, a language
  feature, and an architectural concept all warrant different shapes). A reasonable
  default is fundamentals → main concepts/commands → advanced/production concepts →
  real-world patterns, but deviate freely if the topic fits better another way. For
  curiosity-driven topics, this might instead be historical context → core mental
  model → how it's used in practice → interesting edge cases/debates — whatever
  suits the subject.
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
- A "Real Scenario" — a concrete bug, incident, or situation where this actually
  bites someone (for goal-driven topics), or an interesting real-world example/anecdote
  that illustrates the concept well (for curiosity-driven topics) — not just abstract
  description
- A diagram (ASCII or a simple description) where the concept is fundamentally
  spatial or structural (architecture, data flow, process order) and text alone
  would obscure it

ACCURACY — search for current official docs before writing anything version-specific
(commands, flags, config syntax, current best practices, deprecations). Flag anywhere
a practice has notably changed in recent versions, since that's exactly where outdated
blog posts mislead people.

SOURCING — cite primary sources inline as links: official docs for the technology
itself, and GitHub/Stack Overflow/dev.to for real-world patterns, common mistakes,
and professional conventions. When official docs and a common blog convention
disagree, prefer the official docs and say so.

FORMATTING — make it genuinely easy to reference later, not just read once:

- A clickable table of contents with anchor links at the top
- "back to top" links after each section
- Quick-reference tables (cheat sheets) at the end of major parts
- Code/commands fenced with proper language tags

Save this as a markdown file, and add a short "About This Document" spec section
near the top capturing these instructions (including whether this was goal-driven
or curiosity-driven, and the confirmed scope), so that if I ask you to update it
later, you'll know the structure and style to match without me re-explaining it.
