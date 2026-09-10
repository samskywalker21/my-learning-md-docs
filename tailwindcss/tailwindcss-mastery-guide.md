# Tailwind CSS — Mastery Guide (Overview)

The entry point for this repo's Tailwind material. This document holds the shared spec, the playground you need before running anything, the map of the focused sub-docs, the version table, the table of outdated v3 conventions, and the reference apparatus (learning order, self-check, cross-doc cheat sheet).

The actual teaching lives in the sub-docs. Start here, then follow the [Suggested Learning Order](#9-suggested-learning-order).

---

## About This Document

This section is the canonical spec for **every doc in this set** — the sub-docs each carry a short pointer back here rather than repeating it.

- **Framing: curiosity-driven.** No upcoming task is driving this. The goal is durable, hands-on understanding of Tailwind itself. The docs are built as a *guided tutorial* — a sequence of runnable examples and **Try It** exercises you are meant to actually run against the playground in [§4](#4-setup-your-playground), each stating what correct output looks like. Theory ("why is it designed this way") appears only where it explains a mechanism — why the engine scans files as plain text, why cascade layers decide override order, why variant order is not commutative — never as a standalone history section.
- **Confirmed scope: framework-agnostic core Tailwind.** The build pipeline, utility-first fundamentals and the generated CSS, the full variant system, layout and spacing utilities, responsive design and container queries, the `@theme` design-token system, dark mode, colour/typography/effects, reusability strategies, cascade-layer architecture, the engine and content detection, the `@utility`/`@variant` extension API, and migration from v3.
- **Deliberately excluded** (with pointers): framework integration (React/Vue/Nuxt/Svelte component patterns, `clsx`/`cva` recipes beyond the one comparison in [Part 8](./tailwindcss-reuse-architecture.md)), the first-party plugins (`@tailwindcss/typography`, `@tailwindcss/forms`), component libraries built on Tailwind (shadcn/ui, daisyUI, Tailwind UI), and CSS itself. See [§8](#8-deliberately-not-covered-and-where-to-look-instead).
- **Assumed background: the [`css/`](../css/css-mastery-guide.md) set in this repo, or equivalent.** You know the cascade and specificity, the box model, flexbox and grid, container queries, `@layer`, custom properties, and `oklch()`. This set therefore **does not re-teach CSS** — it teaches how Tailwind exposes CSS, and cross-links to the `css/` docs at every point where the underlying mechanism is what's actually confusing you. A basic Node/npm workflow is assumed. **No prior Tailwind experience is assumed** — the Beginner tier is real teaching.
- **Depth tiers.** Each Part runs **Beginner → Working Knowledge → Advanced → Mastery**, so you can stop at whichever level you need today. Tiers are collapsed where one would be filler, and each collapse is stated inline. Collapsing is expected behaviour, not a defect to fix on a later pass. The weight of this set sits in Working Knowledge and Advanced; Mastery concentrates in [Reuse & Architecture](./tailwindcss-reuse-architecture.md) and [Engine & Extensibility](./tailwindcss-engine-extensibility.md).
- **Per-section ingredients**, used where they earn their place rather than forced into every subsection:
  - a short plain-language explanation of what a thing is and why it matters;
  - a **wrong vs. right** snippet pair wherever a gotcha has a common wrong form (Tailwind has an unusual number of these, because v3 habits still compile-and-silently-misbehave in v4);
  - a **Try It** — a runnable exercise with the expected observable result — since this is a curiosity-driven set. Production **Real Scenario** blocks appear only where a bug genuinely only bites at scale (bundle size, monorepo scanning, build times);
  - an **ASCII diagram** where the concept is spatial or structural — the build pipeline, the layer stack, variant nesting order.
- **Sourcing rule.** [tailwindcss.com](https://tailwindcss.com/docs) is the primary source, cited inline at the specific claim it supports. Where a popular-but-outdated convention disagrees with the official docs, the official docs win and the doc says so explicitly — these are called out as **Outdated convention** notes and collected in [§7](#7-the-v3-convention-table). **Deviation from the repo template, stated openly:** Stack Overflow was not reachable from the authoring environment on the date below (the same restriction the [`css/`](../css/css-mastery-guide.md) set hit), so no SO links appear in this set. Gotchas are instead sourced to the caveats the official docs state themselves — Tailwind's docs are unusually candid about their own footguns, so this costs less here than it would elsewhere. If you extend these docs from an environment that *can* reach SO, adding real-world SO citations at the gotchas is a welcome improvement — just don't construct SO URLs from memory.
- **Formatting.** Clickable TOC per doc, "back to top" links after each Part, explicit `<a id="…"></a>` anchors where tier headings repeat, a cheat-sheet table at the end of each Part, and language-tagged fences (`css`, `html`, `js`, `bash`, `vue`).
- **Written against: Tailwind CSS v4.3.3** (the latest release as of **September 8, 2026**; published July 16, 2026 per [npm](https://www.npmjs.com/package/tailwindcss?activeTab=versions)). Every feature introduced after v4.0 carries an inline **Since v4.x** note, so a reader pinned to an older 4.x knows what they don't have — see [§6](#6-the-version-table). Browser floor is Chrome 111 / Safari 16.4 / Firefox 128 ([Compatibility](https://tailwindcss.com/docs/compatibility)).
- **To update this doc later.** *Preserve*: the curiosity-driven framing, the numbered Part structure with explicit tier subheadings, the wrong-vs-right pairs, the Try It exercises with expected results, the **Outdated convention** notes, the TOC/back-to-top/anchor apparatus, and the per-Part cheat sheets. *Re-verify against current docs first*: **[§6](#6-the-version-table) and [§7](#7-the-v3-convention-table)** — the version table is the whole point of this set's currency, and utility renames are what drift fastest across releases; also re-check [Part 11's](./tailwindcss-engine-extensibility.md) `@utility` value-resolution syntax and [Part 6's](./tailwindcss-theming-design.md) theme-namespace table, since both are young APIs that have already gained parameters within 4.x.

---

## Table of Contents

1. [How This Set Is Organised](#1-how-this-set-is-organised)
2. [The Sub-Docs](#2-the-sub-docs)
3. [What Tailwind Actually Is](#3-what-tailwind-actually-is)
4. [Setup: Your Playground](#4-setup-your-playground)
5. [The One-Paragraph Mental Model](#5-the-one-paragraph-mental-model)
6. [The Version Table](#6-the-version-table)
7. [The v3-Convention Table](#7-the-v3-convention-table)
8. [Deliberately Not Covered (and where to look instead)](#8-deliberately-not-covered-and-where-to-look-instead)
9. [Suggested Learning Order](#9-suggested-learning-order)
10. [Quick Self-Check](#10-quick-self-check)
11. [Cross-Doc Cheat Sheet](#11-cross-doc-cheat-sheet)

---

## 1. How This Set Is Organised

Tailwind is 12 Parts of teaching material spread over 5 sub-docs, plus this overview. The split follows the order in which the concepts actually become necessary — the build pipeline first, because in v4 the pipeline *is* the configuration, and almost every "why isn't my class working" question is a content-detection question in disguise:

```
  tailwindcss-mastery-guide.md   <- you are here: spec, setup, map, version tables, reference
        |
        +-- tailwindcss-foundations.md          Parts 1-3    how a class becomes CSS
        +-- tailwindcss-layout-responsive.md    Parts 4-5    where boxes go, and how they adapt
        +-- tailwindcss-theming-design.md       Parts 6-7    the design system behind the utilities
        +-- tailwindcss-reuse-architecture.md   Parts 8-9    how it survives a real codebase
        +-- tailwindcss-engine-extensibility.md Parts 10-12  the engine, the extension API, and v3 migration
```

Each Part is self-contained enough to reopen on its own, and each ends with a quick-reference table. Within a Part, the tiers mean:

| Tier | What you get |
|---|---|
| **Beginner** | What it is, the minimal correct usage, nothing else |
| **Working Knowledge** | Everyday patterns, the utility surface you actually reach for |
| **Advanced** | Edge cases, gotchas, correctness and build-size implications |
| **Mastery** | Internals, powerful-but-rare capabilities, expert tradeoffs |

Where a tier would be filler, it is collapsed and the doc says so — e.g. "*Dark mode gets Beginner → Advanced only; its 'Mastery' material is really Part 11's custom-variant work applied.*"

[↑ Back to top](#table-of-contents)

---

## 2. The Sub-Docs

### [`tailwindcss-foundations.md`](./tailwindcss-foundations.md) — Parts 1–3

| Part | Covers |
|---|---|
| 1. Setup & the Build Pipeline | The four install paths (CLI, Vite, PostCSS, webpack); what `@import "tailwindcss"` actually expands to; the three layers it defines; reading the generated stylesheet; why there is no `tailwind.config.js` |
| 2. Utility-First Fundamentals | The spacing scale and `--spacing()`; arbitrary values `[…]`, arbitrary properties, and the `(--var)` shorthand; the underscore-for-space rule; the `!` important modifier and why it moved; conflicting classes and why "last class wins" is a lie |
| 3. Variants | Pseudo-class and pseudo-element variants; `group-*`/`peer-*` including named and arbitrary forms; `data-*`, `aria-*`, `has-*`, `not-*`, `in-*`, `nth-*`; arbitrary variants `[&…]`; **variant stacking order**, which reads left-to-right outside-in |

**Read this if:** anything at all. Every other Part assumes these three. If "my class isn't applying" or "why did the wrong one win" is a sentence you say often, the answer is in Part 2 or Part 3.

### [`tailwindcss-layout-responsive.md`](./tailwindcss-layout-responsive.md) — Parts 4–5

| Part | Covers |
|---|---|
| 4. Layout & Spacing Utilities | Flex and grid utility surfaces mapped onto the CSS you already know; sizing; the **v4.2 logical-property expansion** (`mbs-*`, `mbe-*`, `pbs-*`, `pbe-*`, `inline-*`, `block-*`, `inset-s-*`, `inset-bs-*`) and the `start-*`/`end-*` deprecation; why `gap` beats `space-x-*` |
| 5. Responsive Design & Container Queries | The mobile-first rule and the single most common beginner error; `max-*` variants and breakpoint ranges; custom breakpoints via `--breakpoint-*`; `@container`, named containers, `@max-*`, container query units, and the **v4.3 `@container-size`** |

**Read this if:** you are still reaching for `md:` when the decision is about a component's own width. Part 5 explains why that was always a workaround.

### [`tailwindcss-theming-design.md`](./tailwindcss-theming-design.md) — Parts 6–7

| Part | Covers |
|---|---|
| 6. `@theme` & the Design-Token System | The namespace table and which utilities each namespace powers; add vs. override vs. replace with `--*: initial`; `@theme inline` and `@theme static` and when each is mandatory; theme variables as real CSS custom properties; runtime theming; dark mode via `dark:` and `@custom-variant` |
| 7. Colour, Typography & Effects | The oklch palette including the **v4.2 `mauve`/`olive`/`mist`/`taupe`**; the `/` opacity modifier and `--alpha()`; typography utilities and **v4.2 `font-features-*`**; shadows, gradients, filters; the **v4.3 `scrollbar-*`, `zoom-*`, `tab-*`** utilities |

**Read this if:** you are about to hard-code a hex value, or you are wondering where "the config file" went.

### [`tailwindcss-reuse-architecture.md`](./tailwindcss-reuse-architecture.md) — Parts 8–9

| Part | Covers |
|---|---|
| 8. Reusability | The honest four-way comparison: loops, component extraction, `@layer components`, and `@apply` — and why `@apply` is the wrong default despite being the first thing everyone reaches for; `tailwind-merge` and `cva`; `@reference` for scoped `<style>` blocks and CSS modules, and why the docs recommend plain CSS variables instead |
| 9. Architecture at Scale | Cascade layers and the real override order; `@import "tailwindcss" important` and `prefix()` as legacy-integration escape hatches; design tokens across a monorepo; `source(none)` with explicit `@source`; when Tailwind stops being the right answer |

**Read this if:** your codebase has more than one developer, or you have just written your fourth `@apply` rule and something feels wrong. It does.

### [`tailwindcss-engine-extensibility.md`](./tailwindcss-engine-extensibility.md) — Parts 10–12

| Part | Covers |
|---|---|
| 10. The Engine & Content Detection | Plain-text scanning and what it implies; the dynamic-class-name rule, which is the single most common Tailwind bug; `@source`, `@source not`, `@source inline()` with brace expansion, and `source(none)`; build performance |
| 11. Extending Tailwind | `@utility` from simple to functional — `--value()`, `--modifier()`, theme/bare/literal/arbitrary resolution, and the **v4.3 `--default()`**; `@custom-variant` and `@slot`; `@variant` in custom CSS including the **v4.3 stacked and compound forms**; the legacy `@plugin`/`@config` bridge |
| 12. Migrating from v3 | `npx @tailwindcss/upgrade`; the complete rename table; the silent behaviour changes (default border colour, ring width, `hover` on touch) that the codemod cannot catch; the browser floor |

**Read this if:** you are debugging a class that should exist but doesn't, you want to add a utility properly instead of writing custom CSS, or you have inherited a v3 codebase.

[↑ Back to top](#table-of-contents)

---

## 3. What Tailwind Actually Is

Tailwind is a **CSS generator**. That is the whole thing, and holding onto it will save you a great deal of confusion.

It is not a component library — it ships no components. It is not a runtime — nothing of Tailwind reaches the browser except plain CSS. It is not really a "framework" in the sense that Bootstrap is one, because it has no opinions about what a card or a navbar looks like. What it does is read your source files, notice that the string `p-4` appears in one of them, and emit `.p-4 { padding: calc(var(--spacing) * 4) }` into a stylesheet. Everything else in this set is detail on that sentence.

Two consequences follow immediately, and both are load-bearing:

1. **Your CSS bundle is proportional to the number of *distinct* utilities you use, not to the size of your app.** Using `p-4` in a thousand places emits one rule. This is why utility-first CSS scales in a way that hand-written component CSS does not — the tenth screen adds almost no CSS, because it reuses the same utilities as the first nine.
2. **Tailwind only knows what it can *see* as literal text.** It does not parse your JavaScript or resolve your variables. `bg-${color}-600` is, to Tailwind, a string that matches no utility, so no rule is emitted and your element renders unstyled. This one fact accounts for most Tailwind bugs — [Part 10](./tailwindcss-engine-extensibility.md) covers it properly.

The trade Tailwind offers is **naming for composing**. Traditional "semantic CSS" asks you to invent a name (`.promo-card__title--featured`) and maintain the mapping between that name and its styles, in a different file. Tailwind asks you to compose from a fixed vocabulary, in place. You lose the name; you gain the ability to change one element's styles without wondering what else uses that class. Whether that is a good trade is a real design debate, and [Part 8](./tailwindcss-reuse-architecture.md) treats it as one rather than assuming the answer.

[↑ Back to top](#table-of-contents)

---

## 4. Setup: Your Playground

Everything in this set runs against one small project with no framework and no bundler. Create it once and reuse it for every **Try It**:

```bash
mkdir tailwind-playground && cd tailwind-playground
npm init -y
npm install tailwindcss @tailwindcss/cli
```

Two files. First the stylesheet — in v4 this single line *is* your configuration file, and everything you customise later goes in it ([Tailwind CLI install docs](https://tailwindcss.com/docs/installation/tailwind-cli)):

```css
/* src/input.css */
@import "tailwindcss";
```

Then the page:

```html
<!-- index.html -->
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Tailwind playground</title>
  <link href="./src/output.css" rel="stylesheet">
</head>
<body>
  <!-- paste each exercise's markup here -->
  <h1 class="text-3xl font-bold underline">Hello world!</h1>
</body>
</html>
```

Run the compiler in watch mode and leave it running:

```bash
npx @tailwindcss/cli -i ./src/input.css -o ./src/output.css --watch
```

Open `index.html` directly in a browser. The heading should be large, bold and underlined; if it is plain, the stylesheet path is wrong.

Three habits that make the exercises actually teach you something:

1. **Keep `src/output.css` open in your editor.** This is the single highest-value habit in this entire set. Tailwind is a code generator, and watching the generated code change as you add classes turns every abstraction in these docs into something you can see. When a class does not work, `output.css` tells you immediately whether the rule was never generated (a content-detection problem — [Part 10](./tailwindcss-engine-extensibility.md)) or was generated and then lost (a cascade problem — [Part 9](./tailwindcss-reuse-architecture.md)). Those two bugs look identical in the browser and have nothing in common.
2. **Open DevTools and read the Styles pane**, exactly as in the [`css/`](../css/css-mastery-guide.md) set. Tailwind's output is ordinary CSS in ordinary cascade layers, and the browser will happily show you which rule won.
3. **Search the docs by utility name, not by CSS property.** [tailwindcss.com](https://tailwindcss.com/docs) is organised by utility, and its search is good. The reflex to build is "what is this called in Tailwind", not "how do I do this in Tailwind".

> **A note on the Play CDN.** [tailwindcss.com](https://tailwindcss.com/docs/installation/using-vite) lists a Play CDN build for quick experiments. It is genuinely useful for a throwaway CodePen, but do not use it for this set's exercises — it compiles in the browser at runtime, which hides exactly the pipeline behaviour Parts 1 and 10 are teaching you to see.

[↑ Back to top](#table-of-contents)

---

## 5. The One-Paragraph Mental Model

Tailwind runs as a **build step over your source files, not over your CSS**. It scans every file it is allowed to look at as **plain text**, extracts every token that could plausibly be a class name, and throws away the ones it doesn't recognise. Each surviving token is matched against a utility definition — either a built-in one or one you defined with `@utility` — and the values in that definition are resolved against the **theme**, which is just a pile of CSS custom properties you declared in `@theme`. Any **variants** on the token (`md:`, `hover:`, `dark:`, `group-has-[a]:`) wrap the generated rule in nested selectors and at-rules, applied left-to-right from the outside in. The resulting rules are emitted into **cascade layers** — `theme`, `base`, `components`, `utilities` — and the layer order, not specificity, is what decides who wins. What reaches the browser is a plain stylesheet with no Tailwind in it at all.

```
  your source files (.html, .jsx, .vue, .py, anything)
        |
        |  scanned as PLAIN TEXT -- never parsed, never evaluated
        v
  candidate tokens:  "flex"  "md:p-4"  "bg-${c}-600"  "container"  "mx-auto"
        |                                    |
        |                                    +--> no utility matches: DISCARDED SILENTLY
        v
  utility lookup  --------------------> @theme  (--spacing, --color-*, --breakpoint-*, ...)
        |                                    resolves the values
        v
  variant wrapping  (outside-in, left-to-right)
        md:p-4  ->  @media (width >= 48rem) { .md\:p-4 { padding: ... } }
        |
        v
  emitted into cascade layers:   theme -> base -> components -> utilities
        |                        (LAYER ORDER decides the winner, not specificity)
        v
  output.css  --  plain CSS, no runtime, no Tailwind
```

Keep that diagram in your head. Each Part in this set is a zoom-in on one arrow — Part 1 on the pipeline itself, Part 3 on variant wrapping, Part 6 on the theme, Part 9 on the layers, Part 10 on the scanner.

**The one thing that trips up people coming from CSS:** the scanner step has no knowledge of your program. It is not a compiler with a symbol table; it is closer to `grep`. Once that is genuinely internalised, the dynamic-class-name bug stops being surprising and becomes obvious.

[↑ Back to top](#table-of-contents)

---

## 6. The Version Table

This set is written against **v4.3.3**. Tailwind 4.x has shipped real features in minor releases, so "I'm on v4" is not specific enough to know whether an example here will work. Check your `package.json` against this table.

| Landed in | Feature | Covered in |
|---|---|---|
| **v4.0** | CSS-first config (`@theme`), no `tailwind.config.js` | [Part 6](./tailwindcss-theming-design.md) |
| v4.0 | Automatic content detection (respects `.gitignore`) | [Part 10](./tailwindcss-engine-extensibility.md) |
| v4.0 | Theme exposed as native CSS custom properties | [Part 6](./tailwindcss-theming-design.md) |
| v4.0 | Container queries built in (`@container`, `@min-*`, `@max-*`) — no plugin | [Part 5](./tailwindcss-layout-responsive.md) |
| v4.0 | OKLCH / P3 colour palette | [Part 7](./tailwindcss-theming-design.md) |
| v4.0 | 3D transforms (`rotate-x-*`, `translate-z-*`, …) | [Part 7](./tailwindcss-theming-design.md) |
| v4.0 | Expanded gradients (angles, conic, radial, interpolation modifiers) | [Part 7](./tailwindcss-theming-design.md) |
| v4.0 | `@starting-style` support, `not-*`, `in-*`, `nth-*`, `inert` variants | [Part 3](./tailwindcss-foundations.md) |
| v4.0 | Dynamic utility values (`grid-cols-15` without config) | [Part 2](./tailwindcss-foundations.md) |
| v4.0 | First-party Vite plugin `@tailwindcss/vite` | [Part 1](./tailwindcss-foundations.md) |
| **v4.2** | `@tailwindcss/webpack` loader | [Part 1](./tailwindcss-foundations.md) |
| v4.2 | Four new palettes: `mauve`, `olive`, `mist`, `taupe` | [Part 7](./tailwindcss-theming-design.md) |
| v4.2 | Logical-property expansion: `mbs-*`, `mbe-*`, `pbs-*`, `pbe-*`, `border-bs`, `border-be-*`, `scroll-mbs-*`, `inline-*`, `block-*`, `min-/max-inline-*`, `min-/max-block-*`, `inset-s-*`, `inset-e-*`, `inset-bs-*`, `inset-be-*` | [Part 4](./tailwindcss-layout-responsive.md) |
| v4.2 | **`start-*` and `end-*` deprecated** in favour of `inset-s-*` / `inset-e-*` | [Part 4](./tailwindcss-layout-responsive.md) |
| v4.2 | `font-features-*` (OpenType features) | [Part 7](./tailwindcss-theming-design.md) |
| **v4.3** | `scrollbar-auto/thin/none`, `scrollbar-thumb-*`, `scrollbar-track-*`, `scrollbar-gutter-*` | [Part 7](./tailwindcss-theming-design.md) |
| v4.3 | `@container-size` (and `@container-size/{name}`) | [Part 5](./tailwindcss-layout-responsive.md) |
| v4.3 | `zoom-*` | [Part 7](./tailwindcss-theming-design.md) |
| v4.3 | `tab-*` | [Part 7](./tailwindcss-theming-design.md) |
| v4.3 | Stacked (`@variant hover:focus`) and compound (`@variant hover, focus`) variants in CSS | [Part 11](./tailwindcss-engine-extensibility.md) |
| v4.3 | `--default(…)` for functional `@utility` definitions | [Part 11](./tailwindcss-engine-extensibility.md) |

Sources: the [v4.0 announcement](https://tailwindcss.com/blog/tailwindcss-v4) and the [v4.3 release post](https://tailwindcss.com/blog/tailwindcss-v4-3), which covers both v4.2 and v4.3.

> **On version numbers below 4.2:** I could not retrieve a dedicated v4.1 release post, so this table does not attempt to attribute anything to it. Features listed as v4.0 were confirmed in the v4.0 announcement; features listed as v4.2/v4.3 were confirmed in the v4.3 post. If you need exact minor-version attribution for something not listed here, check the [changelog](https://github.com/tailwindlabs/tailwindcss/releases) rather than trusting this table's silence.

**Browser floor:** Chrome 111, Safari 16.4, Firefox 128 ([Compatibility](https://tailwindcss.com/docs/compatibility)). v4 leans on native cascade layers, `@property` and `color-mix()`, none of which can be polyfilled meaningfully. If you must support older browsers, stay on v3.4 — this is one of the rare cases where the honest answer is "don't upgrade".

[↑ Back to top](#table-of-contents)

---

## 7. The v3-Convention Table

Tailwind v4 was a ground-up rewrite, and the internet has not caught up. The majority of Tailwind tutorials, blog posts and Stack Overflow answers you will find are v3-era, and — this is the dangerous part — **most of them fail silently rather than erroring.** A `tailwind.config.js` that v4 never reads does not throw; it just does nothing, and you spend an hour wondering why your custom colour doesn't exist.

Every row here is a pattern you *will* see in the wild. Learn to recognise them.

| Outdated (v3) convention | What v4 does instead | Fails how? | Covered in |
|---|---|---|---|
| `@tailwind base; @tailwind components; @tailwind utilities;` | `@import "tailwindcss";` | Errors — the directives are gone | [Part 1](./tailwindcss-foundations.md) |
| `tailwind.config.js` with `theme.extend` | `@theme { … }` in your CSS | **Silent** — the file is simply not read | [Part 6](./tailwindcss-theming-design.md) |
| `content: ["./src/**/*.{html,js}"]` | Automatic detection; `@source` to adjust | **Silent** | [Part 10](./tailwindcss-engine-extensibility.md) |
| `darkMode: 'class'` | `@custom-variant dark (&:where(.dark, .dark *));` | **Silent** — `dark:` stays on `prefers-color-scheme` | [Part 6](./tailwindcss-theming-design.md) |
| `theme(colors.red.500)` | `var(--color-red-500)` | Deprecated but still works — don't write new ones | [Part 6](./tailwindcss-theming-design.md) |
| `bg-opacity-50`, `text-opacity-*`, `ring-opacity-*` | `bg-black/50` — the `/` modifier | **Silent** — class removed, no rule emitted | [Part 7](./tailwindcss-theming-design.md) |
| `!flex` (leading `!`) | `flex!` (trailing `!`) | **Silent** | [Part 2](./tailwindcss-foundations.md) |
| `bg-[--brand-color]` | `bg-(--brand-color)` | **Silent** | [Part 2](./tailwindcss-foundations.md) |
| `tw-flex` prefix | `tw:flex` — the prefix is now a variant | **Silent** | [Part 9](./tailwindcss-reuse-architecture.md) |
| `flex-shrink-0`, `flex-grow-1` | `shrink-0`, `grow-1` | **Silent** | [Part 4](./tailwindcss-layout-responsive.md) |
| `overflow-ellipsis` | `text-ellipsis` | **Silent** | [Part 7](./tailwindcss-theming-design.md) |
| `shadow-sm` meaning the small shadow | `shadow-xs`; `shadow-sm` is now the old bare `shadow` | **Worst case: silently different** — it still renders, just wrong | [Part 12](./tailwindcss-engine-extensibility.md) |
| `rounded`, `blur`, `drop-shadow` (bare) | `rounded-sm`, `blur-sm`, `drop-shadow-sm` | Silently different | [Part 12](./tailwindcss-engine-extensibility.md) |
| `outline-none` | `outline-hidden` | Silently different | [Part 12](./tailwindcss-engine-extensibility.md) |
| `ring` = 3px | `ring` = 1px; use `ring-3` | Silently different | [Part 12](./tailwindcss-engine-extensibility.md) |
| Default border colour is `gray-200` | Default is `currentColor` | Silently different | [Part 12](./tailwindcss-engine-extensibility.md) |
| **v4.2:** `start-4`, `end-4` | `inset-s-4`, `inset-e-4` | Deprecated, still works | [Part 4](./tailwindcss-layout-responsive.md) |

Sources: the [upgrade guide](https://tailwindcss.com/docs/upgrade-guide) for the v3→v4 rows, the [v4.3 release post](https://tailwindcss.com/blog/tailwindcss-v4-3) for the `start-*`/`end-*` deprecation, and [Functions & Directives](https://tailwindcss.com/docs/functions-and-directives), which is where `theme()` is marked deprecated and `@config`/`@plugin` are marked compatibility-only.

**The rule of thumb:** if a Tailwind tutorial tells you to create or edit a JavaScript config file, it is v3. Close it and open the [official docs](https://tailwindcss.com/docs) instead. This is the single highest-yield filter you can apply.

[↑ Back to top](#table-of-contents)

---

## 8. Deliberately Not Covered (and where to look instead)

Per [prompt.md §4](../prompt.md), these are genuinely important and deliberately left out rather than half-covered:

| Not covered | Why | Where to look |
|---|---|---|
| **CSS itself** — cascade, specificity, box model, flex, grid, container queries, `@layer`, custom properties, `oklch` | This set assumes it and cross-links instead of duplicating | The [`css/`](../css/css-mastery-guide.md) set in this repo |
| **Framework integration** — React/Vue/Svelte/Nuxt component patterns, `clsx`, `cva` recipes, shadcn-style variant APIs | Deserves its own goal-driven doc; the class-merging *problem* is explained in [Part 8](./tailwindcss-reuse-architecture.md), but the ecosystem solutions are surveyed, not taught | [Framework guides](https://tailwindcss.com/docs/installation/framework-guides); the [`vue/`](../vue/vue-mastery-guide.md) and [`react/`](../react/react-mastery-guide.md) sets here for the component side |
| **First-party plugins** — `@tailwindcss/typography`, `@tailwindcss/forms` | They are separate packages with their own docs, and neither teaches you anything about Tailwind itself | [Plugin docs](https://tailwindcss.com/docs) |
| **Component libraries** — Tailwind UI, shadcn/ui, daisyUI, Nuxt UI | Consuming a component library is a different skill from understanding the generator underneath it | Their own docs; this repo's `nuxt-ui` skill for that specific stack |
| **v3 as a target** | Everything here is v4.3. v3 is only discussed as the thing you are migrating *from* ([Part 12](./tailwindcss-engine-extensibility.md)) | The v3 docs are archived on a separate subdomain — find the current link from [tailwindcss.com](https://tailwindcss.com/docs) rather than guessing it; I could not confirm the archive URL when writing this |
| **The JS plugin API** (`plugin()`, `addUtilities`, `matchUtilities`) | v4's `@utility`/`@custom-variant` supersede it; the legacy bridge is shown in [Part 11](./tailwindcss-engine-extensibility.md) but not taught as the way to do things | [Functions & Directives](https://tailwindcss.com/docs/functions-and-directives) (`@plugin`, `@config`) |

[↑ Back to top](#table-of-contents)

---

## 9. Suggested Learning Order

**If you are new to Tailwind** — read straight through, doing the Try Its:

1. This overview: [§3](#3-what-tailwind-actually-is), [§4](#4-setup-your-playground), [§5](#5-the-one-paragraph-mental-model). Do not skip the playground; nothing below works without it.
2. [Part 1](./tailwindcss-foundations.md) — the pipeline. Watch `output.css` while you work.
3. [Part 2](./tailwindcss-foundations.md) — utility fundamentals. This is where fluency actually comes from.
4. [Part 3](./tailwindcss-foundations.md) — variants. The largest single payoff in the set.
5. [Part 4](./tailwindcss-layout-responsive.md) and [Part 5](./tailwindcss-layout-responsive.md) — layout and responsive. Fast if your CSS is solid, since it's mostly a vocabulary mapping.
6. [Part 6](./tailwindcss-theming-design.md) — `@theme`. Stop here if you just want to be productive; this is the end of "enough to work".
7. Everything else, as it becomes relevant.

**If you already use Tailwind but learned it on v3** — read in this order instead:

1. [§7](#7-the-v3-convention-table), the v3-convention table. Read every row.
2. [Part 12](./tailwindcss-engine-extensibility.md) — migration, including what the codemod misses.
3. [Part 6](./tailwindcss-theming-design.md) — `@theme` replaces everything you knew about config.
4. [Part 3](./tailwindcss-foundations.md) — the variant system grew a lot in v4 (`not-*`, `in-*`, `nth-*`, `has-*`).
5. [Part 11](./tailwindcss-engine-extensibility.md) — `@utility` replaces the JS plugin API.

**If something is broken right now:**

| Symptom | Go to |
|---|---|
| A class does nothing, and no rule exists in `output.css` | [Part 10](./tailwindcss-engine-extensibility.md) — content detection |
| A class does nothing, but the rule *is* in `output.css` | [Part 9](./tailwindcss-reuse-architecture.md) — layers and override order |
| The wrong utility wins between two you wrote | [Part 2](./tailwindcss-foundations.md) — conflicting classes |
| A variant applies at the wrong time | [Part 3](./tailwindcss-foundations.md) — stacking order |
| A custom colour/spacing value doesn't exist | [Part 6](./tailwindcss-theming-design.md) — `@theme` namespaces |
| Styles broke after `npm update` | [Part 12](./tailwindcss-engine-extensibility.md), then [§7](#7-the-v3-convention-table) |
| Build is slow, or the CSS bundle is huge | [Part 10](./tailwindcss-engine-extensibility.md) — scanning and `@source not` |

[↑ Back to top](#table-of-contents)

---

## 10. Quick Self-Check

If you can answer these without looking, you have the set. Each links to where it's answered.

**Foundations**

1. What does `@import "tailwindcss"` expand to, and which cascade layers does it define? → [Part 1](./tailwindcss-foundations.md)
2. Why does `bg-${color}-600` produce no styles, and what are the two supported ways to fix it? → [Part 10](./tailwindcss-engine-extensibility.md)
3. In `class="p-2 p-4"`, which wins — and why is "the last one in the class attribute" the wrong answer? → [Part 2](./tailwindcss-foundations.md)
4. What is the difference between `bg-[--brand]` and `bg-(--brand)` in v4? → [Part 2](./tailwindcss-foundations.md)
5. Why is `grid-cols-[1fr_2fr]` written with an underscore, and how would you write a value that needs a literal underscore? → [Part 2](./tailwindcss-foundations.md)

**Variants**

6. In `dark:md:hover:bg-red-500`, what is the nesting order of the generated CSS? → [Part 3](./tailwindcss-foundations.md)
7. What does `group-hover/item:visible` do that `group-hover:visible` cannot? → [Part 3](./tailwindcss-foundations.md)
8. When would you use `in-*` instead of `group-*`? → [Part 3](./tailwindcss-foundations.md)
9. Why does `hover:` not fire on a phone tap in v4, and how do you opt out of that? → [Part 12](./tailwindcss-engine-extensibility.md)

**Layout & responsive**

10. Why is `sm:text-center` the wrong way to centre text on mobile? → [Part 5](./tailwindcss-layout-responsive.md)
11. How do you target *only* the `md` breakpoint? → [Part 5](./tailwindcss-layout-responsive.md)
12. What is the difference between `md:flex-row` and `@md:flex-row`? → [Part 5](./tailwindcss-layout-responsive.md)
13. Which utility replaced `start-4` in v4.2, and why did the name change? → [Part 4](./tailwindcss-layout-responsive.md)

**Theme**

14. What is the difference between `@theme`, `@theme inline` and `@theme static`, and when is `inline` mandatory? → [Part 6](./tailwindcss-theming-design.md)
15. How do you delete the entire default colour palette but keep everything else? → [Part 6](./tailwindcss-theming-design.md)
16. Where does `--spacing` come from, and what happens if you change it? → [Part 6](./tailwindcss-theming-design.md)
17. How do you switch `dark:` from the media query to a class, and what exact line does it take? → [Part 6](./tailwindcss-theming-design.md)

**Architecture**

18. Give two concrete reasons `@apply` is discouraged, and name the case where it's still correct. → [Part 8](./tailwindcss-reuse-architecture.md)
19. Why does a utility override a `@layer components` rule even when the component rule has higher specificity? → [Part 9](./tailwindcss-reuse-architecture.md)
20. What problem does `tailwind-merge` solve that Tailwind itself cannot? → [Part 8](./tailwindcss-reuse-architecture.md)
21. What does `@reference` do, and why do the docs suggest avoiding the situation that needs it? → [Part 8](./tailwindcss-reuse-architecture.md)

**Engine & extension**

22. What does Tailwind scan by default, and what does it skip? → [Part 10](./tailwindcss-engine-extensibility.md)
23. Write an `@source inline()` that safelists `bg-red-100` through `bg-red-900` plus their hover variants. → [Part 10](./tailwindcss-engine-extensibility.md)
24. Write a functional `@utility` that accepts a theme value, a bare integer, an arbitrary value, and a bare-word default. → [Part 11](./tailwindcss-engine-extensibility.md)
25. Name three v3→v4 changes that `npx @tailwindcss/upgrade` cannot fix for you. → [Part 12](./tailwindcss-engine-extensibility.md)

[↑ Back to top](#table-of-contents)

---

## 11. Cross-Doc Cheat Sheet

Every Part has its own cheat sheet at its end. This one is the cross-cutting surface — the things you look up regardless of which Part you're in.

### Directives (all of them)

| Directive | Purpose | Part |
|---|---|---|
| `@import "tailwindcss"` | The whole framework. Accepts `source(none)`, `important`, `prefix(tw)` | [1](./tailwindcss-foundations.md), [9](./tailwindcss-reuse-architecture.md) |
| `@theme { … }` | Define design tokens. Options: `inline`, `static` | [6](./tailwindcss-theming-design.md) |
| `@source "…"` | Add a scan path. Also `@source not`, `@source inline()`, `@source not inline()` | [10](./tailwindcss-engine-extensibility.md) |
| `@utility name-*` | Define a custom utility that works with variants | [11](./tailwindcss-engine-extensibility.md) |
| `@variant name { … }` | Apply a Tailwind variant inside custom CSS | [11](./tailwindcss-engine-extensibility.md) |
| `@custom-variant name (…)` | Define a new variant | [11](./tailwindcss-engine-extensibility.md) |
| `@apply …` | Inline utilities into a custom rule. Use sparingly | [8](./tailwindcss-reuse-architecture.md) |
| `@reference "…"` | Make theme/utilities visible to an isolated CSS context without emitting them | [8](./tailwindcss-reuse-architecture.md) |
| `@layer base \| components` | Plain CSS in a Tailwind layer | [9](./tailwindcss-reuse-architecture.md) |
| `@config "…"` | **Compatibility only** — load a v3 JS config | [11](./tailwindcss-engine-extensibility.md) |
| `@plugin "…"` | **Compatibility only** — load a v3 JS plugin | [11](./tailwindcss-engine-extensibility.md) |

### Functions

| Function | Purpose | Part |
|---|---|---|
| `--spacing(n)` | Compute a spacing value from the theme scale | [2](./tailwindcss-foundations.md) |
| `--alpha(color / pct)` | Adjust a colour's opacity via `color-mix()` | [7](./tailwindcss-theming-design.md) |
| `--value(…)` / `--modifier(…)` / `--default(…)` | Value resolution inside `@utility` | [11](./tailwindcss-engine-extensibility.md) |
| `theme(…)` | **Deprecated.** Use `var(--…)`; still needed inside media queries | [6](./tailwindcss-theming-design.md) |

Source: [Functions & Directives](https://tailwindcss.com/docs/functions-and-directives).

### Class-name syntax

| Syntax | Means | Part |
|---|---|---|
| `p-4` | Utility from the theme scale | [2](./tailwindcss-foundations.md) |
| `p-[13px]` | Arbitrary value | [2](./tailwindcss-foundations.md) |
| `bg-(--my-var)` | CSS variable shorthand (v4; was `[--my-var]` in v3) | [2](./tailwindcss-foundations.md) |
| `[mask-type:luminance]` | Arbitrary property | [2](./tailwindcss-foundations.md) |
| `grid-cols-[1fr_2fr]` | Underscore = space | [2](./tailwindcss-foundations.md) |
| `bg-red-500/75` | Opacity modifier | [7](./tailwindcss-theming-design.md) |
| `bg-red-500!` | Important (v4; was `!bg-red-500`) | [2](./tailwindcss-foundations.md) |
| `md:` `hover:` `dark:` | Variants, applied outside-in left-to-right | [3](./tailwindcss-foundations.md) |
| `group-hover/name:` | Named group variant | [3](./tailwindcss-foundations.md) |
| `data-[size=lg]:` | Attribute variant with a value | [3](./tailwindcss-foundations.md) |
| `[&_p]:mt-4` | Arbitrary variant | [3](./tailwindcss-foundations.md) |
| `@md:` | Container query variant | [5](./tailwindcss-layout-responsive.md) |
| `tw:flex` | Prefixed build (v4; was `tw-flex`) | [9](./tailwindcss-reuse-architecture.md) |

### Install commands

```bash
npm install tailwindcss @tailwindcss/cli       # standalone CLI
npm install tailwindcss @tailwindcss/vite      # Vite
npm install tailwindcss @tailwindcss/postcss   # PostCSS
npm install tailwindcss @tailwindcss/webpack   # webpack (v4.2+)
npx @tailwindcss/upgrade                       # v3 -> v4 codemod (needs Node 20+)
```

[↑ Back to top](#table-of-contents)

---

*Part of the [Learning Notes](../README.md) collection. Written against Tailwind CSS v4.3.3 on September 8, 2026. Assumes the [Modern CSS set](../css/css-mastery-guide.md).*
