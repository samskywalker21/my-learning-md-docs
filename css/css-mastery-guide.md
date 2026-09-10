# Modern CSS — Mastery Guide (Overview)

The entry point for this repo's CSS material. This document holds the shared spec, the playground you need before running anything, the map of the focused sub-docs, the browser-baseline table, the cross-cutting accessibility rules, and the reference apparatus (learning order, self-check, cross-doc cheat sheet).

The actual teaching lives in the sub-docs. Start here, then follow the [Suggested Learning Order](#8-suggested-learning-order).

---

## About This Document

This section is the canonical spec for **every doc in this set** — the sub-docs each carry a short pointer back here rather than repeating it.

- **Framing: curiosity-driven.** No upcoming task is driving this. The goal is durable, hands-on understanding of CSS itself. The docs are built as a *guided tutorial* — a sequence of runnable examples and **Try It** exercises you are meant to actually paste into a browser, each stating what correct output looks like. Theory ("why is it designed this way") appears only where it explains a mechanism — the cascade, containing blocks, formatting contexts, the pixel pipeline — never as a standalone history section.
- **Confirmed scope: the CSS language and the browser's rendering model.** Selectors and the cascade, the box model and normal flow, positioning and stacking, flexbox and grid, responsive design and container queries, typography, colour and visual effects, custom properties and architecture (`@layer`, `@scope`, nesting), animation, and rendering performance.
- **Deliberately excluded** (with pointers): Sass and other preprocessors, CSS-in-JS, build tooling, and component-library theming. Tailwind appears **only** as an interop section in [Architecture](./css-architecture.md) — how the native features map onto a utility-first workflow — not as a tutorial. See [§7](#7-deliberately-not-covered-and-where-to-look-instead).
- **Assumed background: HTML, and not much else.** You can write markup and you have opened DevTools. **No solid CSS foundation is assumed** — [Foundations](./css-foundations.md) genuinely teaches the cascade, the box model and normal flow from zero, and the Beginner tier throughout this set is real teaching rather than a formality. No JavaScript is required except in two clearly-marked places (the View Transition API, and toggling `will-change`).
- **Depth tiers.** Each Part runs **Beginner → Working Knowledge → Advanced → Mastery**, so you can stop at whichever level you need today. Tiers are collapsed where one would be filler, and each collapse is stated inline. Collapsing is expected behaviour, not a defect to fix on a later pass. The weight of this set sits in Working Knowledge and Advanced; Mastery concentrates in [Architecture](./css-architecture.md) and [Animation & Performance](./css-animation-performance.md).
- **Per-section ingredients**, used where they earn their place rather than forced into every subsection:
  - a short plain-language explanation of what a thing is and why it matters;
  - a **wrong vs. right** snippet pair wherever a gotcha has a common wrong form;
  - a **Try It** — a runnable exercise with the expected observable result — since this is a curiosity-driven set (production "Real Scenario" blocks appear only where a bug genuinely only bites at scale);
  - an **ASCII diagram** where the concept is spatial or structural — the box model, the flex axes, grid line numbering, the stacking-context tree, the pixel pipeline.
- **Sourcing rule.** [MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@supports) and the W3C specs are the primary sources, cited inline at the specific claim they support. Where a popular-but-outdated convention disagrees with the official docs, the official docs win and the doc says so explicitly — CSS has more of these than most topics, and they are called out as **Outdated convention** notes. **Deviation from the repo template, stated openly:** Stack Overflow was not reachable from the authoring environment on the date below, so no SO links appear in this set. Gotchas are instead sourced to the caveats the official docs state themselves. If you extend these docs from an environment that *can* reach SO, adding real-world SO citations at the gotchas is a welcome improvement — just don't construct SO URLs from memory.
- **Formatting.** Clickable TOC per doc, "back to top" links after each Part, explicit `<a id="…"></a>` anchors where tier headings repeat, a cheat-sheet table at the end of each Part, and language-tagged fences (`css`, `html`, `js`).
- **Written against.** Evergreen Chrome, Firefox, Safari and Edge as of **September 8, 2026**. CSS has no version number, so every feature newer than roughly 2022 carries an inline **Baseline** note taken from MDN and the [web.dev Baseline digests](https://web.dev/blog/baseline-digest-may-2026). Tailwind references are to **v4** ([Tailwind theme docs](https://tailwindcss.com/docs/theme)).
- **To update this doc later.** *Preserve*: the curiosity-driven framing, the numbered Part structure with explicit tier subheadings, the wrong-vs-right pairs, the Try It exercises with expected results, the **Outdated convention** notes, the TOC/back-to-top/anchor apparatus, and the per-Part cheat sheets. *Re-verify against current docs first*: **everything in [§5](#5-the-baseline-table)** — Baseline dates move, and the features currently marked *newly available* or *limited* (anchor positioning, scroll-driven animations, `interpolate-size`, `sibling-index()`, `field-sizing`, cross-document view transitions) are exactly the ones where a stale support note misleads; also re-check the Tailwind v4 interop section in [Architecture](./css-architecture.md) against the current Tailwind release.

---

## Table of Contents

1. [How This Set Is Organised](#1-how-this-set-is-organised)
2. [The Sub-Docs](#2-the-sub-docs)
3. [Setup: Your Playground File](#3-setup-your-playground-file)
4. [The One-Paragraph Mental Model](#4-the-one-paragraph-mental-model)
5. [The Baseline Table](#5-the-baseline-table)
6. [Cross-Cutting: Accessibility & Feature Detection](#6-cross-cutting-accessibility--feature-detection)
7. [Deliberately Not Covered (and where to look instead)](#7-deliberately-not-covered-and-where-to-look-instead)
8. [Suggested Learning Order](#8-suggested-learning-order)
9. [Quick Self-Check](#9-quick-self-check)
10. [Cross-Doc Cheat Sheet](#10-cross-doc-cheat-sheet)

---

## 1. How This Set Is Organised

CSS is 14 Parts of teaching material spread over 6 sub-docs, plus this overview. The split follows the order in which the concepts actually become necessary — the cascade and the box model first, because every layout bug you will ever have is downstream of one of them:

```
  css-mastery-guide.md   <- you are here: spec, setup, map, baseline table, reference
        |
        +-- css-foundations.md            Parts 1-3    how a style reaches a pixel
        +-- css-layout-flex-grid.md       Parts 4-6    where boxes go
        +-- css-responsive-typography.md  Parts 7-8    how it adapts, and how it reads
        +-- css-color-visuals.md          Parts 9-10   how it looks
        +-- css-architecture.md           Parts 11-12  how it stays maintainable
        +-- css-animation-performance.md  Parts 13-14  how it moves, and how it stays fast
```

Each Part is self-contained enough to reopen on its own, and each ends with a quick-reference table. Within a Part, the tiers mean:

| Tier | What you get |
|---|---|
| **Beginner** | What it is, the minimal correct usage, nothing else |
| **Working Knowledge** | Everyday patterns, the property surface you actually reach for |
| **Advanced** | Edge cases, gotchas, correctness and performance implications |
| **Mastery** | Internals, powerful-but-rare capabilities, expert tradeoffs |

Where a tier would be filler, it is collapsed and the doc says so — e.g. "*Logical properties get Beginner → Working Knowledge only; their Advanced material is really Part 8's writing-mode work applied.*"

[↑ Back to top](#table-of-contents)

---

## 2. The Sub-Docs

### [`css-foundations.md`](./css-foundations.md) — Parts 1–3

| Part | Covers |
|---|---|
| 1. How CSS Reaches the Page | Rules, declarations, at-rules; how the browser builds the CSSOM; the ways to attach styles; DevTools as your primary instrument; inherited vs. initial values and `inherit`/`initial`/`unset`/`revert` |
| 2. Selectors & the Cascade | The selector families; specificity as three columns; `:is()`, `:where()`, `:not()`, `:has()`; the full cascade sort order; why `!important` is a symptom |
| 3. The Box Model & Normal Flow | Content/padding/border/margin; `box-sizing`; `display` as two values; block vs. inline formatting contexts; margin collapsing; `overflow`; logical properties |

**Read this if:** anything at all. Every other Part assumes these three. If "my CSS isn't applying" or "why is there a gap there" is a sentence you say often, the answer is in Part 2 or Part 3.

### [`css-layout-flex-grid.md`](./css-layout-flex-grid.md) — Parts 4–6

| Part | Covers |
|---|---|
| 4. Positioning & Stacking | `static`/`relative`/`absolute`/`fixed`/`sticky`, containing blocks, the initial containing block, `inset`, `z-index` and stacking contexts, `isolation`, why `overflow` breaks `sticky` |
| 5. Flexbox | Main/cross axes, the `flex` shorthand decoded, `flex-basis` vs. `width`, `min-width: auto`, alignment, `gap`, wrapping, the honest limits of one dimension |
| 6. Grid & Subgrid | Tracks, `fr`, `repeat()`, `minmax()`, `auto-fill` vs. `auto-fit`, line placement, named areas, implicit tracks, `subgrid`, the `place-*` shorthands |

**Read this if:** you are choosing between flex and grid, or something is 3px off and you can't see why.

### [`css-responsive-typography.md`](./css-responsive-typography.md) — Parts 7–8

| Part | Covers |
|---|---|
| 7. Responsive Design & Container Queries | Every unit and when it lies; `clamp()` and fluid type; media queries and the modern range syntax; **container queries** (size and style); intrinsic sizing; `aspect-ratio` |
| 8. Typography & Text | `@font-face`, `font-display`, variable fonts, the system font stack; the vertical-rhythm properties; `text-wrap: balance`/`pretty`; hyphenation, truncation, `line-clamp`; writing modes; `field-sizing` |

**Read this if:** you are still writing `@media (max-width: 768px)` for component-level decisions. Part 7 explains why that was always a workaround.

### [`css-color-visuals.md`](./css-color-visuals.md) — Parts 9–10

| Part | Covers |
|---|---|
| 9. Colour in Modern CSS | sRGB and its ceiling; `lab`/`lch`/`oklab`/`oklch`; why `hsl()` lightness lies; `color-mix()`; relative colour syntax; wide gamut; `light-dark()` and `color-scheme`; contrast in practice |
| 10. Backgrounds, Borders & Effects | The `background` longhands; gradients (linear, radial, conic, repeating) and interpolation space; shadows; `filter` and `backdrop-filter`; blend modes; `clip-path` and masking; `border-radius` and `border-image`; forced-colors mode |

**Read this if:** your dark-mode palette goes muddy, or your gradient has a grey band through the middle. Both are the same bug — [Part 9](./css-color-visuals.md).

### [`css-architecture.md`](./css-architecture.md) — Parts 11–12

| Part | Covers |
|---|---|
| 11. Custom Properties & Theming | `--x` and `var()`, the fallback chain, inheritance and the invalid-at-computed-value-time rule, `@property` and typed properties, theming patterns, JS interop, what custom properties **cannot** do |
| 12. Architecture: Layers, Scope & Nesting | `@layer` and the inverted `!important` order; `@scope` and proximity; native nesting and how `&` differs from Sass; BEM/ITCSS/utility-first compared; design tokens; `@supports`; **the Tailwind v4 interop section** |

**Read this if:** your stylesheet has an `!important` count above zero, or you are about to invent a naming convention.

### [`css-animation-performance.md`](./css-animation-performance.md) — Parts 13–14

| Part | Covers |
|---|---|
| 13. Transitions, Animations & the Frontier | `transition` and what is animatable; `@keyframes`; easing including `linear()`; `transform` and the individual `translate`/`rotate`/`scale`; `@starting-style` and `transition-behavior: allow-discrete`; `interpolate-size`; view transitions; scroll-driven animations; anchor positioning; `prefers-reduced-motion` |
| 14. Rendering & Performance | The pixel pipeline; what triggers style/layout/paint/composite; layout thrashing; `contain` and `content-visibility`; `will-change` used correctly; selector performance and `:has()`; reading a DevTools performance trace |

**Read this if:** something is janky, or you want to build the modern-feeling interactions (morphing page transitions, tooltips that flip, scroll progress bars) without a JS library.

[↑ Back to top](#table-of-contents)

---

## 3. Setup: Your Playground File

Everything in this set runs in a browser with no build step, no framework, and no `npm install`. Create one file and reuse it:

```html
<!-- play.html - open this directly in a browser (File > Open, or drag it in) -->
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>CSS playground</title>
  <style>
    /* ---- a deliberately minimal reset, explained in Part 3 ---- */
    *, *::before, *::after { box-sizing: border-box; }
    body { margin: 0; font: 16px/1.5 system-ui, sans-serif; }

    /* ---- paste each exercise below this line ---- */
  </style>
</head>
<body>
  <!-- paste each exercise's markup here -->
</body>
</html>
```

Three habits that make the exercises actually teach you something:

1. **Open DevTools before you start** (F12). The **Elements → Styles** pane shows which rules matched and which were struck through by the cascade; the **Computed** pane shows the single value that won. Part 1 covers reading both.
2. **Use the box-model diagram** at the bottom of the Computed pane constantly. Most "why is this the wrong size" questions are answered by looking at it for five seconds.
3. **Add a temporary outline when you are lost:** `* { outline: 1px solid red; }`. Use `outline`, not `border` — outlines don't take up space, so they don't change the layout you are trying to debug.

Two toggles you will want during the accessibility and dark-mode exercises, both in DevTools under **Rendering** (open the command menu with `Ctrl/Cmd+Shift+P` → "Show Rendering"):

- **Emulate CSS media feature `prefers-color-scheme`** — light/dark without touching your OS settings.
- **Emulate CSS media feature `prefers-reduced-motion`** — for the Part 13 exercises.

[↑ Back to top](#table-of-contents)

---

## 4. The One-Paragraph Mental Model

CSS is a system for answering one question per property per element: **what is the value?** The browser gathers every declaration that matches an element from every stylesheet, sorts them by the cascade (origin and importance first, then cascade layers, then specificity, then source order — the full order is in [Part 2](./css-foundations.md)), and picks one winner per property. Whatever is left unspecified is filled in by inheritance from the parent or by the property's initial value. Only *then* does layout begin: the winning values are handed to a layout algorithm — normal flow, flex, or grid — which is chosen by the `display` value of each element's **parent**, and which sizes and positions boxes inside a **containing block** determined by the `position` value. Finally the positioned boxes are painted in a defined order, grouped into **stacking contexts**, and composited onto the screen. Almost every confusing CSS behaviour is one of these four stages misidentified: you think it's a specificity problem when it's an inheritance problem, or a `z-index` problem when it's a stacking-context problem, or a `width` problem when it's a formatting-context problem.

```
  Author CSS --+
  User CSS ----+--> CASCADE --> winning declared value
  UA CSS ------+       |
                       v
             INHERITANCE / INITIAL --> computed value --+
                                                        |
     +--------------------------------------------------+
     v
  LAYOUT (flow | flex | grid, inside a containing block) --> box geometry
     |
     v
  PAINT (in stacking-context order) --> COMPOSITE --> pixels
```

Keep that diagram in your head. Each Part in this set is a zoom-in on one arrow.

[↑ Back to top](#table-of-contents)

---

## 5. The Baseline Table

"Baseline" is the cross-browser interoperability status the browser vendors publish jointly: **newly available** means the feature has shipped in all of Chrome, Edge, Firefox and Safari; **widely available** means that happened at least 30 months ago, so it is safe without a fallback for essentially any audience.

This is the single table in this set most likely to go out of date. **Re-verify before relying on a "newly available" or "limited" row.**

| Feature | Status as of Sept 2026 | Where it's taught | Fallback needed? |
|---|---|---|---|
| Flexbox, Grid, custom properties, `calc()` | Widely available | Parts 5, 6, 11 | No |
| `@supports` ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@supports)) | Widely available | Part 12 | No |
| Logical properties ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_logical_properties_and_values)) | Widely available | Part 3 | No |
| `@layer` ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer)) | Widely available (March 2022) | Part 12 | No |
| `forced-colors` media feature ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/forced-colors)) | Widely available (Sept 2022) | Part 10 | No |
| `color-mix()` ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/color-mix)), `oklch()` ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/oklch)) | Widely available (May 2023) | Part 9 | No |
| Subgrid ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_grid_layout/Subgrid)) | Widely available (Sept 2023) | Part 6 | No |
| `:has()` ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/:has)) | Widely available (Dec 2023) | Part 2 | No |
| Container **size** queries ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_containment/Container_queries)) | Widely available | Part 7 | No |
| `text-wrap: balance` / `pretty` ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/text-wrap)) | Widely available (March 2024) | Part 8 | No — degrades silently |
| `@property` ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@property)) | Newly available (July 2024) | Part 11 | Only for animated custom properties |
| `@starting-style`, `transition-behavior: allow-discrete` ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@starting-style)) | Newly available (Aug 2024) | Part 13 | No — degrades to no entry animation |
| Native nesting ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_nesting/Using_CSS_nesting)) | Newly available | Part 12 | No |
| `@scope` ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@scope)) | Newly available (March 2026) | Part 12 | Yes, for older browsers |
| Container **style** queries ([web.dev digest](https://web.dev/blog/baseline-digest-may-2026)) | Newly available (May 2026) | Part 7 | Yes |
| `field-sizing: content` ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/field-sizing)) | Newly available (June 2026) | Part 8 | No — degrades to fixed sizing |
| `sibling-index()` / `sibling-count()` ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/sibling-index)) | Newly available (Aug 2026) | Part 13 | Yes |
| Anchor positioning ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_anchor_positioning/Using)) | Rolling out — **verify** | Part 13 | Yes |
| Scroll-driven animations ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_scroll-driven_animations)) | Rolling out — **verify** | Part 13 | Yes, via `@supports` |
| Same-document view transitions ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API)) | Available; cross-document newer — **verify** | Part 13 | Yes — feature-detect in JS |
| `interpolate-size` / `calc-size()` ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/interpolate-size)) | **Limited** / experimental | Part 13 | Yes |

The practical rule this set teaches: **write the modern version as the default and let `@supports` or graceful degradation handle the rest**, rather than writing the old version everywhere and calling it safe. [Part 12](./css-architecture.md) shows the pattern.

[↑ Back to top](#table-of-contents)

---

## 6. Cross-Cutting: Accessibility & Feature Detection

These are not a separate Part — they appear inside the Part where they actually bite. This table is the index so you can find them.

| Concern | Mechanism | Rule of thumb | Taught in |
|---|---|---|---|
| Keyboard focus | `:focus-visible` | Never `outline: none` without a visible replacement ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/:focus-visible)) | Part 2 |
| Motion sensitivity | `@media (prefers-reduced-motion: reduce)` | *Replace* motion with a fade; don't delete all feedback ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion)) | Part 13 |
| Dark mode | `prefers-color-scheme`, `color-scheme`, `light-dark()` | Set `color-scheme` so form controls and scrollbars follow | Part 9 |
| High contrast | `@media (forced-colors: active)`, `forced-color-adjust` | `box-shadow` and `background-image` are forced off — re-add a `border` ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/forced-colors)) | Part 10 |
| Contrast ratio | `oklch()` lightness, `color-mix()` | Perceptual lightness is not `hsl()` lightness | Part 9 |
| Text zoom | `rem`, `em`, `ch` — never `px` for type | Media-query breakpoints in `em` respect user font size ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@media)) | Part 7 |
| Reading order | `order`, `flex-direction: *-reverse`, `grid-auto-flow: dense` | Visual order diverging from DOM order breaks keyboard and screen-reader order | Parts 5, 6 |
| Content hidden from the a11y tree | `content-visibility: hidden` vs. `auto` | `auto` stays findable and focusable; `hidden` does not ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_containment/Using_CSS_containment)) | Part 14 |
| Feature detection | `@supports`, `@supports selector(…)`, `@supports not (…)` | Test the *specific* property:value pair, not the browser | Part 12 |
| Internationalisation | Logical properties, `writing-mode` | `margin-inline` instead of `margin-left`/`right` | Parts 3, 8 |

[↑ Back to top](#table-of-contents)

---

## 7. Deliberately Not Covered (and where to look instead)

| Topic | Why it's out | Where to look |
|---|---|---|
| **Sass / Less / Stylus** | Native CSS has absorbed the three features people actually reached for them for — variables, nesting, and file organisation (now `@layer`). What remains (loops, mixins with arguments, functions) is a real gap, but it is a preprocessor tutorial, not a CSS one. [Part 12](./css-architecture.md) has a short "what Sass still buys you" table so you can make the call. | [sass-lang.com](https://sass-lang.com) |
| **Tailwind as a system** | Utility-first is a methodology with its own docs. This set covers **interop** only: which native features Tailwind v4 is built on, and how to mix hand-written CSS with it. | [Tailwind docs](https://tailwindcss.com/docs/theme); this repo's [`legacy/tailwindcss/`](../legacy/tailwindcss/) |
| **CSS-in-JS** (styled-components, emotion, vanilla-extract) | Runtime/build concerns, framework-specific. The CSS they emit is the CSS taught here. | Each library's docs |
| **Build tooling** (PostCSS, Lightning CSS, autoprefixer, bundler CSS handling) | Tooling, not language. Worth knowing that autoprefixer is largely vestigial for the features in this set — nothing taught here needs vendor prefixes. | This repo's [`legacy/vite/`](../legacy/vite/) |
| **SVG styling and `<canvas>`** | Adjacent graphics languages with their own property sets. `clip-path`, `mask` and `filter` are covered where they apply to HTML elements. | [MDN SVG](https://developer.mozilla.org/en-US/docs/Web/SVG) |
| **Print stylesheets and paged media** | A genuinely separate specialism (`@page`, break control, running headers) that would double [Part 7](./css-responsive-typography.md) for a use case most people meet once. | MDN `@page` |
| **Houdini paint/layout worklets** | Beyond `@property` (which *is* covered, in [Part 11](./css-architecture.md)), the Houdini APIs are not interoperable enough to teach as a default. | MDN CSS Houdini |
| **Framework component styling** (Vue `scoped`, React CSS Modules) | Covered in the framework docs in this repo. `@scope` in [Part 12](./css-architecture.md) is the native equivalent. | [`vue/`](../vue/vue-mastery-guide.md), [`react/`](../react/react-mastery-guide.md) |

[↑ Back to top](#table-of-contents)

---

## 8. Suggested Learning Order

**If you are starting from close to zero** — read in order. Parts 1–6 are not optional and not skimmable; everything else in CSS is a consequence of them.

1. **[Foundations](./css-foundations.md), Parts 1–3.** Do every Try It. The box-model and margin-collapsing exercises in Part 3 in particular — those two mechanisms cause more confusion than the rest of CSS combined.
2. **[Layout](./css-layout-flex-grid.md), Part 5 (Flexbox)**, then **Part 6 (Grid)**. Deliberately out of Part order: flexbox is the smaller model, and grid is easier to appreciate once you have felt flexbox's one-dimensional limit.
3. **[Layout](./css-layout-flex-grid.md), Part 4 (Positioning & Stacking)**, once you have built something with flex and grid and hit your first `z-index` that "doesn't work".
4. **[Responsive & Typography](./css-responsive-typography.md), Part 7.** Learn container queries *before* you build a habit around media queries — the habit is the hard part to unlearn.
5. **[Colour & Visuals](./css-color-visuals.md), Part 9**, at Beginner → Working Knowledge. Enough to build a palette that works in both themes.
6. **[Architecture](./css-architecture.md), Parts 11–12.** Read these the moment your CSS exceeds roughly 200 lines, not before — the problems they solve have to be felt to make sense.
7. **[Animation & Performance](./css-animation-performance.md), Part 13**, for the fun part.
8. Fill in the rest — Part 8 (typography detail), Part 10 (effects), Part 14 (performance) — as questions arise.

**If you already write CSS and want the modern layer only:** Part 2's `:has()`/`:is()`/`:where()` and cascade-order sections → Part 7's container queries → Part 9 in full → Part 12 in full → Part 13's Advanced and Mastery tiers. That is roughly everything that changed between 2020 and now.

**As a reference:** each Part's cheat-sheet table is designed to be the thing you actually reopen. The [Cross-Doc Cheat Sheet](#10-cross-doc-cheat-sheet) below is the index to them.

[↑ Back to top](#table-of-contents)

---

## 9. Quick Self-Check

If you can answer these without looking, you have the set. Each answer lives in the Part named.

1. Two rules both set `color` on the same element. Name the things the browser compares, in order, to decide the winner. *(Part 2)*
2. Why does `:where(.a, #b)` have specificity `0-0-0` while `:is(.a, #b)` has `1-0-0`? *(Part 2)*
3. Your `<div>` has `width: 300px; padding: 20px; border: 2px solid`. How wide is it on screen, with and without `box-sizing: border-box`? *(Part 3)*
4. A parent and its first child both have `margin-top: 2rem`. How much space is above the parent's content, and what single property on the parent would change the answer? *(Part 3)*
5. A `position: fixed` element suddenly scrolls with the page instead of staying put. What did an ancestor most likely get? *(Part 4)*
6. What does `flex: 1` expand to in longhand, and how does it differ from `flex: auto`? *(Part 5)*
7. When does `repeat(auto-fit, …)` produce a different layout from `repeat(auto-fill, …)`? *(Part 6)*
8. Why can a container query do something a media query fundamentally cannot? *(Part 7)*
9. What does `container-type: inline-size` silently also do to the element, that can break `position: fixed` descendants? *(Parts 4 and 7)*
10. Why does a yellow at `hsl(60 100% 50%)` look far lighter than a blue at `hsl(240 100% 50%)`, and which colour function fixes it? *(Part 9)*
11. `@layer base, components;` — you then write an unlayered rule. Does it beat the `components` layer? What if you add `!important` to a `base` rule? *(Part 12)*
12. Why can `&` in native CSS nesting have higher specificity than the same selector written flat? *(Part 12)*
13. You transition `opacity` on an element going from `display: none` to `display: block`. Nothing animates. Name the two things you need. *(Part 13)*
14. Which of `top`, `transform`, `background-color`, `box-shadow` skip layout when animated, and which skip paint as well? *(Part 14)*
15. When does `content-visibility: auto` make a page slower rather than faster? *(Part 14)*

[↑ Back to top](#table-of-contents)

---

## 10. Cross-Doc Cheat Sheet

The per-Part cheat sheets are the detailed ones. This is the "which doc do I open" index.

| Symptom / question | Doc | Part |
|---|---|---|
| "My rule isn't applying" | [Foundations](./css-foundations.md) | 2 |
| "Where is this extra space coming from?" | [Foundations](./css-foundations.md) | 3 |
| "Why is my `width` being ignored?" | [Foundations](./css-foundations.md) | 3 |
| "`z-index: 9999` doesn't work" | [Layout](./css-layout-flex-grid.md) | 4 |
| "`position: sticky` does nothing" | [Layout](./css-layout-flex-grid.md) | 4 |
| "My flex item won't shrink / overflows" | [Layout](./css-layout-flex-grid.md) | 5 |
| "Flex or grid for this?" | [Layout](./css-layout-flex-grid.md) | 5, 6 |
| "Cards in a row won't line their titles up" | [Layout](./css-layout-flex-grid.md) | 6 (subgrid) |
| "This component needs to respond to its own width" | [Responsive](./css-responsive-typography.md) | 7 |
| "How do I do fluid type without media queries?" | [Responsive](./css-responsive-typography.md) | 7 (`clamp()`) |
| "My heading breaks with one orphan word" | [Responsive](./css-responsive-typography.md) | 8 (`text-wrap: balance`) |
| "My font causes a layout shift" | [Responsive](./css-responsive-typography.md) | 8 (`font-display`) |
| "My dark palette looks muddy" | [Colour](./css-color-visuals.md) | 9 |
| "My gradient has a grey band in the middle" | [Colour](./css-color-visuals.md) | 10 |
| "How do I theme without a preprocessor?" | [Architecture](./css-architecture.md) | 11 |
| "How do I stop specificity wars?" | [Architecture](./css-architecture.md) | 12 (`@layer`) |
| "How do I scope styles without BEM?" | [Architecture](./css-architecture.md) | 12 (`@scope`) |
| "How does this fit with Tailwind?" | [Architecture](./css-architecture.md) | 12 |
| "How do I animate in an element that starts hidden?" | [Animation](./css-animation-performance.md) | 13 |
| "How do I animate `height: auto`?" | [Animation](./css-animation-performance.md) | 13 |
| "How do I do page transitions without a router?" | [Animation](./css-animation-performance.md) | 13 |
| "Scrolling is janky" | [Animation](./css-animation-performance.md) | 14 |
| "The page takes forever to render a long list" | [Animation](./css-animation-performance.md) | 14 |

### Adjacent docs in this repo

- [`javascript/javascript-mastery-guide.md`](../javascript/javascript-mastery-guide.md) — for the JS in the view-transition and `will-change` sections.
- [`vue/vue-composables-builtins.md`](../vue/vue-composables-builtins.md) and [`react/react-mastery-guide.md`](../react/react-mastery-guide.md) — framework-level scoping and transition APIs built on the CSS taught here.
- [`legacy/tailwindcss/`](../legacy/tailwindcss/) — the utility-first layer that [Part 12](./css-architecture.md) interops with.

[↑ Back to top](#table-of-contents)
