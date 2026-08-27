# 📚 Learning Notes

A personal collection of reference docs for topics I'm learning — meant to be searched and reopened, not just read once.

## 📂 Structure

One folder per topic, one or more markdown docs inside:

```
.
├── topic-name/
│   └── topic-notes.md
└── README.md
```

## 📖 Index

| Topic | Doc | Last Updated |
|---|---|---|
| React / JS / TS | [`react-nextjs/react-concepts-before-nextjs.md`](./react-nextjs/react-concepts-before-nextjs.md) | August 27, 2026 |

*(Add a row whenever a new doc is added. Status: 📝 in progress · ✅ done · 🔄 needs update — or whatever markers are useful.)*

## 🧭 Doc Conventions

Loose defaults — adjust per topic as needed:

- Explanation → example → real scenario for each concept
- Sources linked inline where relevant
- Table of contents + "back to top" links for longer docs
- A short note at the top of each doc on how it's structured, so it's easy to update later

## ➕ Adding a Topic

1. `mkdir topic-name`
2. Add the doc(s)
3. Add a row to the Index

## 🤖 Prompt Template

Copy this into a new chat to generate a doc in the same style:

```
Act as a Senior [ROLE, e.g. DevOps Engineer / Full Stack Dev] mentoring a junior dev.
I want to master [TOPIC, e.g. "Docker"] before [GOAL, e.g. "working on our
containerized deployment pipeline"].

Create a detailed reference document I can come back to repeatedly, not just read once.

STRUCTURE — organize into parts that build from fundamentals to real-world use:
1. Core/foundational concepts (the stuff that's easy to skip but everything else depends on)
2. Main [TOPIC] concepts and commands/APIs
3. Advanced concepts commonly used in production (skip if not applicable to this topic)
4. Patterns and conventions professionals actually use day-to-day
End with a Suggested Learning Order and a Quick Self-Check list of questions.

PER-SECTION REQUIREMENTS — every concept/subsection should include:
- A short plain-language explanation of what it is and why it matters
- A code/command snippet — prefer a "wrong vs. right" pair when illustrating a
  common mistake or gotcha, not just the "correct" version in isolation
- A "Real Scenario" — a concrete bug, incident, or situation where this actually
  bites someone, not just abstract syntax

SOURCING — cite primary sources inline as links: official docs for the technology
itself, and GitHub/Stack Overflow/dev.to for real-world patterns, common mistakes,
and professional conventions. Search for these rather than relying on training data,
since links and specifics should be current.

FORMATTING — make it genuinely easy to reference later, not just read once:
- A clickable table of contents with anchor links at the top
- "back to top" links after each section
- Quick-reference tables (cheat sheets) at the end of major parts
- Code fenced with proper language tags

Save this as a markdown file, and add a short "About This Document" spec section
near the top capturing these instructions, so that if I ask you to update it later,
you'll know the structure and style to match without me re-explaining it.
```
