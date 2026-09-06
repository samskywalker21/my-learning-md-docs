# JavaScript — Mastery Guide (Overview)

The entry point for this repo's JavaScript material. This document holds the shared spec, the setup you need before running anything, the map of the focused sub-docs, and the reference apparatus (learning order, self-check, cross-doc cheat sheet).

The actual teaching lives in the sub-docs. Start here, then follow the [Suggested Learning Order](#7-suggested-learning-order).

---

## About This Document

This section is the canonical spec for **every doc in this set** — the sub-docs each carry a short pointer back here rather than repeating it.

- **Framing: curiosity-driven.** No upcoming task is driving this. The goal is durable, hands-on understanding of the JavaScript language itself. Accordingly, the docs are built as a *guided tutorial* — a sequence of runnable examples and **Try It** exercises you are meant to actually type into a REPL, each stating what correct output looks like. Theory ("why is it designed this way") appears only at the points where it is needed to explain a mechanism — the event loop, the prototype chain, the module graph — never as a standalone history section.
- **Confirmed scope: the core language only.** Values and types, scope, operators, functions and closures, objects and prototypes, arrays and iteration, generators, classes, errors, asynchrony, modules, regular expressions, dates/`Intl`, and metaprogramming.
- **Deliberately excluded** (with pointers): the DOM and browser APIs, Node.js runtime APIs, TypeScript, frameworks, bundlers, testing, and performance profiling. See [Deliberately Not Covered](#6-deliberately-not-covered-and-where-to-look-instead).
- **Depth tiers.** Each Part in each sub-doc runs **Beginner → Working Knowledge → Advanced → Mastery**, so you can stop at whichever level you need today. Tiers are collapsed where one would be filler, and each such collapse is stated inline in the doc. Collapsing is expected behaviour, not a defect to fix on a later pass.
- **Assumed background: none.** Zero prior programming experience is assumed. The Beginner tier carries real weight throughout rather than being a formality. If you already program in another language, skim Beginner tiers and start at Working Knowledge.
- **Per-section ingredients**, used where they earn their place rather than forced into every subsection:
  - a short plain-language explanation of what a thing is and why it matters;
  - a **wrong vs. right** snippet pair wherever a gotcha has a common wrong form;
  - a **Try It** — a runnable exercise with the expected output — since this is a curiosity-driven doc set (production "Real Scenario" incidents are the goal-driven equivalent and are used only where a gotcha genuinely only bites in production);
  - an **ASCII diagram** where the concept is spatial or structural — the call stack, the prototype chain, the event loop, the module graph.
- **Modern-JS-forward.** Every example is current JavaScript by default: `let`/`const`, arrow functions, destructuring, spread/rest, template literals, optional chaining, `async`/`await`, ES modules, classes, private fields. Older forms (`var`, callback-style async, CommonJS `require`, the IIFE module pattern) appear only where you will actually meet them reading existing code, or where they explain a gotcha — and they are labelled as legacy when they do.
- **Sourcing rule.** [MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript) is the primary source for language behaviour and is cited inline at the specific claim it supports. [TC39](https://tc39.es/) proposal and spec pages are the authority for standardisation status. Stack Overflow is used for real-world gotchas and how practitioners actually resolve them. GitHub and other reputable sites are supplementary only. Where a popular-but-outdated convention disagrees with the official docs, the official docs win and the doc says so explicitly.
- **Availability claims.** Anything added in the last few years carries its [Baseline](https://developer.mozilla.org/en-US/docs/Glossary/Baseline/Compatibility) status and date from MDN, so you can tell at a glance what is safe to use untranspiled. Features that are *not* Baseline (`using`, `Temporal`, `Error.isError`) say so explicitly rather than being presented as ready to use.
- **Formatting.** Clickable TOC at the top of every doc; "back to top" links after each major section; explicit `<a id="..."></a>` anchors on tier headings, because "Advanced" repeats in every Part and bare-heading anchors would collide; a quick-reference table at the end of each Part; fenced code blocks with language tags.
- **Written against.** ECMAScript 2026 (published — [ES2026 spec](https://tc39.es/ecma262/2026/)), with ES2027-track features flagged as not-yet-published. Runtime examples assume Node.js 24 LTS or 26. Baseline data checked **September 6, 2026**.
- **To update this doc later.** *Preserve*: the curiosity-driven framing, the numbered Part structure with explicit tier subheadings, the wrong-vs-right pairs, the Try It exercises with expected output, the TOC/back-to-top/anchor apparatus, and the per-Part cheat sheets. *Re-verify against current docs first*: every Baseline status and date (they move), the ES-year attribution of anything on the ES2027 track (`using`, `Temporal`, `Atomics.pause`, `Iterator.zip`), the Node.js LTS line named in [Orientation & Setup](#3-orientation--setup), and any feature this doc currently calls "limited availability".

---

## Table of Contents

1. [How This Set Is Organised](#1-how-this-set-is-organised)
2. [The Sub-Docs](#2-the-sub-docs)
3. [Orientation & Setup](#3-orientation--setup)
4. [The One-Paragraph Mental Model](#4-the-one-paragraph-mental-model)
5. [Language Editions at a Glance](#5-language-editions-at-a-glance)
6. [Deliberately Not Covered (and where to look instead)](#6-deliberately-not-covered-and-where-to-look-instead)
7. [Suggested Learning Order](#7-suggested-learning-order)
8. [Quick Self-Check](#8-quick-self-check)
9. [Cross-Doc Cheat Sheet](#9-cross-doc-cheat-sheet)

---

## 1. How This Set Is Organised

<a id="overview-organisation"></a>

JavaScript is too large for one file you can comfortably search. It is split here into an overview (this doc) plus seven focused sub-docs, each covering a cluster of Parts that genuinely belong together — the way `mysql` and `postgres` are split in this repo.

Two rules make the split work:

1. **The spec lives here, once.** Each sub-doc's "About This Document" is a short pointer back to this file. Change the style here and the whole set changes.
2. **Parts are numbered globally, not per-doc.** Part 7 is Classes no matter which file it lives in, so cross-references stay stable if a Part later moves.

```text
javascript/
├── javascript-mastery-guide.md           ← you are here (spec, setup, map, self-check)
├── javascript-foundations.md             Parts 1–3
├── javascript-functions-closures.md      Part 4
├── javascript-objects-classes.md         Parts 5, 7
├── javascript-arrays-iteration.md        Part 6
├── javascript-errors-async-modules.md    Parts 8–10
├── javascript-regex-dates-intl.md        Parts 11–12
└── javascript-metaprogramming-modern.md  Parts 13–14
```

[↑ Back to top](#table-of-contents)

---

## 2. The Sub-Docs

<a id="overview-subdocs"></a>

| Doc | Parts | What it covers | Read it when |
|---|---|---|---|
| [Foundations](./javascript-foundations.md) | 1–3 | Values, types, coercion; variables, scope, the TDZ; operators and control flow | You want the substrate everything else stands on |
| [Functions & Closures](./javascript-functions-closures.md) | 4 | Declarations vs. expressions, parameters, `this`, arrow functions, closures, higher-order functions | You are confused by `this`, or by a loop variable captured "wrong" |
| [Objects & Classes](./javascript-objects-classes.md) | 5, 7 | Property access, descriptors, the prototype chain, `class`, private fields, inheritance | You want to know what `class` actually desugars to |
| [Arrays & Iteration](./javascript-arrays-iteration.md) | 6 | Array methods, mutating vs. copying, the iteration protocols, generators, iterator helpers | You are choosing between `map`/`reduce`/`for…of`, or want lazy sequences |
| [Errors, Async & Modules](./javascript-errors-async-modules.md) | 8–10 | Throwing and catching, custom errors; the event loop, Promises, `async`/`await`, cancellation; ES modules | You are debugging an unhandled rejection or an import that is `undefined` |
| [Regex, Dates & Intl](./javascript-regex-dates-intl.md) | 11–12 | Regular expressions end to end; `Date`'s traps, `Intl`, and the `Temporal` replacement | You are parsing text, or a date is off by one day |
| [Metaprogramming & Modern JS](./javascript-metaprogramming-modern.md) | 13–14 | Symbols, `Proxy`/`Reflect`, weak collections; the ES2023→ES2026 grab bag | You want the expert-tier machinery, or a survey of what is new |

[↑ Back to top](#table-of-contents)

---

## 3. Orientation & Setup

<a id="overview-setup"></a>

### Beginner — what JavaScript actually is

<a id="part-0-beginner"></a>

JavaScript is a dynamically typed, single-threaded (per execution context) language. It does not run on its own: it runs inside a **host environment** that supplies the engine plus extra APIs the language itself never defines.

```text
┌──────────────────────────────────────────────────────────┐
│ HOST ENVIRONMENT (browser, Node.js, Deno, Bun)           │
│                                                          │
│  Host APIs — NOT JavaScript:                             │
│    document, window, localStorage           (browser)    │
│    fs, path, process, Buffer                (Node.js)    │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ THE ENGINE (V8, SpiderMonkey, JavaScriptCore)      │  │
│  │                                                    │  │
│  │  The language — what this doc set teaches:         │  │
│  │    let, const, function, class, Promise,           │  │
│  │    Object, Array, Map, Symbol, RegExp, Intl        │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  The event loop lives HERE, in the host — not in the     │
│  engine. See Part 9.                                     │
└──────────────────────────────────────────────────────────┘
```

**Why this matters.** Knowing where the line sits removes a lot of early confusion. `document.querySelector`, `require`, and `setTimeout` are *not* JavaScript. `let`, `class`, `Promise`, and `Intl` are. Everything in this doc set behaves identically in every host, which is exactly why it is worth learning as a unit.

### Working Knowledge — getting a REPL

<a id="part-0-working"></a>

Every **Try It** in these docs runs in any of these. Pick one and keep it open.

```bash
# Option A — Node REPL. Best for short experiments; instant feedback.
node
> 1 + 1
2
> .exit

# Option B — run a file. Best for anything longer than three lines.
node scratch.js

# Option C — a module file, when the example uses import/export or top-level await.
node scratch.mjs
```

Option D is the browser DevTools console (F12 → Console), good for one-liners. One caution there: the console auto-displays the value of the last expression, so a trailing `undefined` under your output is the console talking, not your code.

**Which Node?** As of September 2026, Node.js **24.x is Active LTS** and **26.x is Current**, with 26 due for LTS promotion in October 2026 ([Node.js releases](https://nodejs.org/en/about/previous-releases)). Either runs everything here. Node is also [changing its release cadence](https://nodejs.org/en/blog/announcements/evolving-the-nodejs-release-schedule) from October 2026 to one major per year with every release becoming LTS — so the "even numbers are LTS, odd numbers are not" rule you will see in older posts is on its way out. Re-check this before quoting it.

### Advanced — strict mode, and why you are probably already in it

<a id="part-0-advanced"></a>

`"use strict"` turns several sloppy behaviours into errors: assigning to an undeclared variable throws instead of silently creating a global, `this` in a plain function call is `undefined` instead of the global object, and duplicate parameter names become a syntax error.

```js
// ❌ Sloppy mode — this silently creates a global. A typo becomes a bug you find in prod.
function setName() {
  nmae = "Sam";        // no declaration, no error
}
setName();
console.log(globalThis.nmae); // "Sam"  ← the typo is now a global
```

```js
// ✅ Strict mode — the typo throws immediately.
"use strict";
function setName() {
  nmae = "Sam";        // ReferenceError: nmae is not defined
}
setName();
```

The practical point: **ES modules and class bodies are always strict**, with no directive needed ([MDN: Strict mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode)). If you write modern code you are in strict mode already. You only need to think about `"use strict"` when reading old scripts.

> **Try It.** Save the sloppy version as `sloppy.js` and run `node sloppy.js` — it prints `Sam`. Rename the file to `sloppy.mjs` and run again: `ReferenceError: nmae is not defined`. Same code, same engine; the file extension changed the mode.

### Mastery — collapsed

<a id="part-0-mastery"></a>

There is no Mastery tier here. Setup has no expert layer of its own — the internals this Part gestures at (the event loop, the module graph, realms) are taught where they live, in [Parts 9–10](./javascript-errors-async-modules.md) and [Part 13](./javascript-metaprogramming-modern.md).

### Part 0 quick reference

| Thing | Where it comes from | Notes |
|---|---|---|
| `let`, `const`, `class`, `Promise`, `Symbol` | The language | Identical in every host |
| `document`, `window`, `localStorage` | Browser host | Not available in Node |
| `fs`, `process`, `Buffer` | Node host | Not available in browsers |
| `fetch`, `setTimeout`, `queueMicrotask` | Host, but standardised across hosts | Available in modern Node and browsers |
| `globalThis` | The language (ES2020) | The portable way to reach the global object |
| Strict mode | The language | Automatic in ESM and class bodies |

[↑ Back to top](#table-of-contents)

---

## 4. The One-Paragraph Mental Model

<a id="overview-model"></a>

If you internalise one thing before reading further: **JavaScript has values, bindings, and objects, and they are three different things.** A *value* is a primitive (`42`, `"hi"`, `true`, `null`, `undefined`, a `Symbol`, a `BigInt`) or a reference to an object. A *binding* is a name in a scope that holds a value — `const` freezes the binding, never the value. An *object* is a mutable bag of properties that also holds a link to another object, its **prototype**, which is where method lookup goes when the object itself lacks a property. Nearly every JavaScript surprise — `const` arrays you can still `push` to, `this` changing under you, `[] == false` being `true`, a class method "losing itself" when passed as a callback — is one of those three being mistaken for another.

```text
BINDING            VALUE                 OBJECT (heap)
┌──────────┐       ┌──────────┐          ┌────────────────────┐
│ const u  │──────▶│   ref    │─────────▶│ { name: "Sam" }    │
└──────────┘       └──────────┘          │  [[Prototype]] ────┼──▶ Object.prototype
   frozen            what the            └────────────────────┘
   by const          binding holds         freely mutable
```

`u = {}` is an error (the binding is frozen). `u.name = "Alex"` is fine (the object is not). That single diagram resolves more beginner confusion than any other in this set.

[↑ Back to top](#table-of-contents)

---

## 5. Language Editions at a Glance

<a id="overview-editions"></a>

JavaScript ships yearly. A proposal reaches **Stage 4** ("finished"), then appears in the next annual spec ([the TC39 process](https://tc39.es/process-document/)). Two things routinely mislead people here:

1. **The ES-year is a publication label, not an availability date.** `Array.fromAsync` is an ES2026 feature that has been [Baseline widely available since January 2024](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/fromAsync). Engines ship at Stage 3–4; the spec text catches up afterwards.
2. **Blog posts guess the year wrong.** `using` declarations and `Temporal` are very commonly labelled "ES2026". They are not — per the [TC39 finished-proposals list](https://github.com/tc39/proposals/blob/main/finished-proposals.md), both are on the **ES2027** track. Use Baseline status, not the ES-year, to decide whether you can ship something.

| Edition | Selected finished proposals |
|---|---|
| **ES2024** | `Promise.withResolvers`; Array Grouping (`Object.groupBy`, `Map.groupBy`); resizable `ArrayBuffer`s; RegExp `v` flag; well-formed Unicode strings; change-array-by-copy (`toSorted`, `toReversed`, `with`) |
| **ES2025** | `RegExp.escape`; `Promise.try`; sync iterator helpers; JSON modules; import attributes; RegExp modifiers; new `Set` methods; duplicate named capture groups; `ArrayBuffer.prototype.transfer`; `Float16Array` |
| **ES2026** | `Map.prototype.getOrInsert` (Upsert); `JSON.parse` source-text access; Iterator Sequencing (`Iterator.concat`); `Uint8Array` to/from Base64; `Math.sumPrecise`; `Error.isError`; `Array.fromAsync` |
| **ES2027** *(track — not yet published)* | Explicit Resource Management (`using` / `await using`); `Atomics.pause`; Joint Iteration (`Iterator.zip`); `Temporal` |

Full treatment with runnable examples and Baseline dates is in [Part 14 — Modern JS Grab Bag](./javascript-metaprogramming-modern.md#2-part-14--modern-js-grab-bag).

[↑ Back to top](#table-of-contents)

---

## 6. Deliberately Not Covered (and where to look instead)

<a id="overview-not-covered"></a>

These are genuinely important and deliberately absent — half-covering them would be worse than pointing you somewhere better.

| Topic | Why it is out of scope | Where to look |
|---|---|---|
| **DOM & browser APIs** — `document`, events, `fetch`, storage | Host APIs, not the language; they change independently of ECMAScript | [MDN Web APIs](https://developer.mozilla.org/en-US/docs/Web/API) |
| **Node.js runtime** — `fs`, streams, `http`, CommonJS interop | Host APIs; large enough for its own guide | [`legacy/nodejs/`](../legacy/nodejs/nodejs-mastery-guide.md) in this repo |
| **TypeScript** | A different language layered on this one; learn JS semantics first | [`legacy/javascript/javascript-for-typescript-mastery.md`](../legacy/javascript/javascript-for-typescript-mastery.md) |
| **Frameworks** — React, Vue | Built on the language; the language comes first | [`legacy/react/`](../legacy/react/react-mastery-guide.md), [`legacy/vuejs/`](../legacy/vuejs/vue-mastery-guide.md) |
| **Bundlers & tooling** — Vite, npm, ESLint | Ecosystem, not language | [`legacy/vite/`](../legacy/vite/vite-mastery-guide.md) |
| **Testing** | Tooling-specific | [Vitest docs](https://vitest.dev/) |
| **Typed arrays, `ArrayBuffer`, `SharedArrayBuffer`, `Atomics`** | A specialised binary-data corner; mentioned only in the grab bag | [MDN: Typed arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Typed_arrays) |
| **Performance profiling & JIT tuning** | Engine-specific and fast-moving; advice ages badly | [V8 blog](https://v8.dev/blog) |

[↑ Back to top](#table-of-contents)

---

## 7. Suggested Learning Order

<a id="overview-order"></a>

**If you are new to programming** — read straight through, Beginner and Working Knowledge tiers only, doing every Try It. Skip Advanced and Mastery entirely on the first pass; come back for them once you have written something real.

```text
Foundations (1→2→3) ──▶ Functions & Closures (4) ──▶ Objects & Classes (5, 7)
                                                          │
                                                          ▼
                          Arrays & Iteration (6) ──▶ Errors, Async & Modules (8→9→10)
                                                          │
                                                          ▼
                                Regex, Dates & Intl (11, 12) ──▶ Metaprogramming (13, 14)
```

**If you already program in another language** — start at Part 2 (scope and the TDZ), then jump to Part 4 (`this` and closures), Part 5 (prototypes), and Part 9 (the event loop). Those four are where JavaScript actually differs from what you know; the rest will feel familiar.

**If you are here to fix one specific thing** — use this table instead of reading in order:

| Symptom | Go to |
|---|---|
| `this` is `undefined` inside a method | [Part 4 — `this`](./javascript-functions-closures.md#part-4-advanced) |
| `Cannot access 'x' before initialization` | [Part 2 — the TDZ](./javascript-foundations.md#part-2-working) |
| A `const` array changed anyway | [Part 2 — bindings vs. values](./javascript-foundations.md#part-2-beginner) |
| `[] == false` is `true` and you need to know why | [Part 3 — coercion](./javascript-foundations.md#part-3-advanced) |
| `sort()` reordered the original array | [Part 6 — mutating vs. copying](./javascript-arrays-iteration.md#part-6-working) |
| `UnhandledPromiseRejection` crashed the process | [Part 9 — rejection handling](./javascript-errors-async-modules.md#part-9-advanced) |
| An `await` in a loop is unbearably slow | [Part 9 — concurrency](./javascript-errors-async-modules.md#part-9-working) |
| An import is `undefined` at module top level | [Part 10 — circular imports](./javascript-errors-async-modules.md#part-10-advanced) |
| A date is off by one day | [Part 12 — `Date` traps](./javascript-regex-dates-intl.md#part-12-beginner) |
| A `/g` regex skips every other match | [Part 11 — `lastIndex`](./javascript-regex-dates-intl.md#part-11-advanced) |

[↑ Back to top](#table-of-contents)

---

## 8. Quick Self-Check

<a id="overview-selfcheck"></a>

If you can answer these without looking, you have the material.

**Foundations**
1. Why is `typeof null === "object"`, and what should you use instead?
2. What is the difference between `==` and `===`, and when is `==` actually the right choice?
3. What does the temporal dead zone protect you from that `var` did not?

**Functions**
4. What are the four ways `this` gets bound, and which one wins?
5. Why does an arrow function ignore `.call()`'s first argument?
6. What does a closure capture — the value, or the binding?

**Objects**
7. What is the difference between `obj.x = 1` and `Object.defineProperty(obj, "x", {…})`?
8. Where does `[].map` actually live, and how does the lookup find it?
9. What does `class` do that a constructor function plus `prototype` assignment cannot?

**Arrays & iteration**
10. Which array methods mutate, and what are their copying replacements?
11. What must an object have to work with `for…of`?
12. What does a generator give you that returning an array does not?

**Async**
13. What is the difference between a microtask and a macrotask, and which runs first?
14. Why does `array.forEach(async …)` not wait?
15. What is the difference between `Promise.all` and `Promise.allSettled`?

**Modules**
16. Why can you `import` a binding that is assigned later in the exporting module?
17. How does a default export differ from a named export at the binding level?

**The rest**
18. Why does a `/g` regex give different results when reused, and how do you avoid it?
19. Why is `new Date("2026-09-06")` not the same instant as `new Date("2026/09/06")`?
20. What can a `Proxy` intercept that a getter cannot?

[↑ Back to top](#table-of-contents)

---

## 9. Cross-Doc Cheat Sheet

<a id="overview-cheatsheet"></a>

The single table worth keeping open. Per-Part cheat sheets live at the end of each Part.

| I want to… | Use | Doc |
|---|---|---|
| Declare a value that never rebinds | `const` | [Foundations](./javascript-foundations.md) |
| Compare two values safely | `===`, or `Object.is` for `NaN`/`-0` | [Foundations](./javascript-foundations.md) |
| Fall back only on `null`/`undefined` | `??`, not `\|\|` | [Foundations](./javascript-foundations.md) |
| Keep `this` from the enclosing scope | An arrow function | [Functions](./javascript-functions-closures.md) |
| Copy an object shallowly | `{ ...obj }` | [Objects](./javascript-objects-classes.md) |
| Copy an object deeply | `structuredClone(obj)` | [Objects](./javascript-objects-classes.md) |
| Hide state on an instance | `#private` fields | [Objects](./javascript-objects-classes.md) |
| Sort without mutating | `toSorted()` | [Arrays](./javascript-arrays-iteration.md) |
| Group records by a key | `Object.groupBy()` | [Arrays](./javascript-arrays-iteration.md) |
| Get-or-create a `Map` entry | `map.getOrInsertComputed()` | [Modern JS](./javascript-metaprogramming-modern.md) |
| Iterate lazily | A generator, or iterator helpers | [Arrays](./javascript-arrays-iteration.md) |
| Add context to a caught error | `throw new Error(msg, { cause: err })` | [Errors/Async](./javascript-errors-async-modules.md) |
| Run async work concurrently | `Promise.all` / `allSettled` | [Errors/Async](./javascript-errors-async-modules.md) |
| Cancel in-flight async work | `AbortController` | [Errors/Async](./javascript-errors-async-modules.md) |
| Import conditionally or lazily | `await import()` | [Errors/Async](./javascript-errors-async-modules.md) |
| Build a regex from user input | `RegExp.escape()` | [Regex/Dates](./javascript-regex-dates-intl.md) |
| Format a date for a user | `Intl.DateTimeFormat` | [Regex/Dates](./javascript-regex-dates-intl.md) |
| Do calendar-correct date maths | `Temporal` (not Baseline yet) | [Regex/Dates](./javascript-regex-dates-intl.md) |
| Intercept property access | `Proxy` + `Reflect` | [Metaprogramming](./javascript-metaprogramming-modern.md) |
| Attach metadata without leaking memory | `WeakMap` | [Metaprogramming](./javascript-metaprogramming-modern.md) |

[↑ Back to top](#table-of-contents)

---

*JavaScript Mastery Guide — overview. Written against ECMAScript 2026; Baseline data and Node.js release information verified September 6, 2026.*
