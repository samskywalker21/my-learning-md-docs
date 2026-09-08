# Mastering Tailwind CSS — A Hands-On Reference Guide

> ⤴️ **Superseded by [`tailwindcss/`](../../tailwindcss/tailwindcss-mastery-guide.md)** — an overview plus five sub-docs written against **v4.3.3**. This doc says "Tailwind v4" without a minor version and predates v4.2/v4.3, so its **Part 7 (Engine Internals)** and the **Part 8 cheat-sheet utility tables** are the sections most likely to mislead. Kept for reference, not for extending.

## About This Document

- **Framing:** Curiosity-driven. No upcoming task is driving this — the goal is durable, hands-on understanding of Tailwind CSS itself. Structured as a guided tutorial (a sequence of runnable examples/exercises meant to actually be tried, not just read), with theory sprinkled in only at the points where it explains a mechanism (e.g. *why* the JIT engine scans files as plain text, or *why* cascade layers matter for override order) — not as a standalone history section.
- **Scope:** Framework-agnostic core Tailwind — utility-first fundamentals, responsive/state variants, layout utilities, the CSS-first theming/config system, reusability strategies (`@apply` vs. components vs. class-merging helpers), the JIT/Oxide engine and content-detection internals, the plugin/`@utility` API, dark mode, and container queries. All examples use plain HTML + the standalone CLI or Vite plugin so the material transfers to any framework — no Vue/React/Nuxt-specific integration is covered here (that would warrant its own doc if it becomes goal-driven later).
- **Depth tiers:** Major sections use **Beginner → Working Knowledge → Advanced → Mastery**, collapsed where a topic doesn't have four honest tiers (e.g. Dark Mode and Container Queries stay at Working Knowledge/Advanced since Mastery would just be "know the underlying CSS media/container query spec," which is out of scope).
- **Version basis:** **Tailwind CSS v4** (the CSS-first config version — single `@import "tailwindcss";`, no `tailwind.config.js` required, Rust-based Oxide engine). Wherever v3 conventions still commonly show up in tutorials/blog posts and would mislead in v4, that's flagged explicitly (e.g. `tailwind.config.js` + `theme.extend`, the `content: []` array, `darkMode: 'class'`, opacity-suffix utilities). Checked against official docs as of September 2026.
- **Assumed background:** Comfortable with HTML/CSS fundamentals (box model, flexbox/grid, specificity) and a basic Node.js/npm workflow. No prior utility-first-CSS or Tailwind experience assumed.
- **Sourcing standard:** [tailwindcss.com](https://tailwindcss.com/docs) official docs are the primary source, cited inline. Stack Overflow and GitHub issues/discussions used for real-world v3→v4 migration gotchas. When official docs and an outdated (v3-era) convention disagree, the docs win and the difference is called out explicitly.
- **Update instructions for future-me:** Match this structure (numbered Parts, tier subheadings, wrong-vs-right pairs for gotchas, "Try It" runnable exercises, TOC + back-to-top links). If Tailwind ships a new major version, update the "Version basis" line first and re-check Part 6 (Engine Internals) and the utility-rename table in Part 8's cheat sheet — those are what drift fastest across majors.

---

## Table of Contents

1. [Orientation — What Tailwind Actually Is](#1-orientation--what-tailwind-actually-is)
2. [Part 1 — Setup & the Build Pipeline](#2-part-1--setup--the-build-pipeline)
3. [Part 2 — Utility-First Fundamentals](#3-part-2--utility-first-fundamentals)
4. [Part 3 — Responsive & State Variants](#4-part-3--responsive--state-variants)
5. [Part 4 — Layout Patterns (Flex & Grid)](#5-part-4--layout-patterns-flex--grid)
6. [Part 5 — Theming & Config](#6-part-5--theming--config)
7. [Part 6 — Reusability Strategies](#7-part-6--reusability-strategies)
8. [Part 7 — Engine Internals: JIT & Content Detection](#8-part-7--engine-internals-jit--content-detection)
9. [Part 8 — Plugins & the Custom Utility API](#9-part-8--plugins--the-custom-utility-api)
10. [Part 9 — Dark Mode & Container Queries](#10-part-9--dark-mode--container-queries)
11. [Cheat Sheets](#11-cheat-sheets)
12. [Suggested Learning Order](#12-suggested-learning-order)
13. [Quick Self-Check](#13-quick-self-check)

---

## 1. Orientation — What Tailwind Actually Is

**What it is.** Tailwind is a utility-first CSS framework: instead of writing custom CSS classes (`.card { padding: 1rem; border-radius: 0.5rem; }`), you compose small, single-purpose utility classes directly in markup (`class="p-4 rounded-lg"`). There's no `.card` class to name, maintain, or hunt down in a separate stylesheet — the styling lives right where it's used.

**Why it matters.** The traditional "semantic CSS" approach (one custom class per component, styled in a separate file) sounds cleaner in principle, but in practice it means constantly inventing class names, jumping between HTML and CSS files, and fighting specificity/cascade bugs as stylesheets grow. Utility classes trade "naming things" for "composing things" — and because the same small set of utilities gets reused everywhere, your CSS bundle stops growing linearly with your markup.

**Try it now — get a project running to follow along with:**

```bash
mkdir tailwind-playground && cd tailwind-playground
npm install tailwindcss @tailwindcss/cli
```

```css
/* src/input.css */
@import "tailwindcss";
```

```html
<!-- index.html -->
<!doctype html>
<html>
  <head>
    <link href="./dist/output.css" rel="stylesheet" />
  </head>
  <body>
    <h1 class="text-3xl font-bold text-blue-600 p-4">Hello, Tailwind</h1>
  </body>
</html>
```

```bash
npx @tailwindcss/cli -i src/input.css -o dist/output.css --watch
```

Open `index.html` in a browser. Change `text-blue-600` to `text-red-600` and save — watch `dist/output.css` regenerate and the heading recolor. Notice you never touched a CSS file to do that.
([Get Started with Tailwind CSS](https://tailwindcss.com/docs/installation))

[⬆ back to top](#table-of-contents)

---

## 2. Part 1 — Setup & the Build Pipeline

Covers: the three official ways to run Tailwind, what changed from v3's `tailwind.config.js`-centric setup, and why v4 has no `content: []` array in the example above.

### Beginner

Tailwind v4 ships three integration paths, all built on the same Rust-based engine:

| Path | Package | When to use it |
|---|---|---|
| Standalone CLI | `@tailwindcss/cli` | No existing build tool, quick scripts/prototypes |
| Vite plugin | `@tailwindcss/vite` | Any Vite-based project (fastest integration) |
| PostCSS plugin | `@tailwindcss/postcss` | Existing PostCSS pipeline (webpack, etc.) |

All three read the same CSS entry point containing `@import "tailwindcss";` — that one line replaces v3's three separate directives.

**Wrong vs. right — the v3 → v4 import syntax:**

```css
/* Wrong (v3 syntax — no longer works in v4) */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Right (v4) */
@import "tailwindcss";
```
([Upgrade Guide — Tailwind CSS](https://tailwindcss.com/docs/upgrade-guide))

### Working Knowledge

**Vite setup** (the fastest path if you're already on Vite):

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [tailwindcss()],
})
```

No `postcss.config.js`, no `tailwind.config.js`, no `content` array — just the plugin plus `@import "tailwindcss";` in your main CSS file.

**Is `tailwind.config.js` gone?** No — it's still supported for teams with an existing JS config (theme values built programmatically, shared across projects), but it's no longer auto-detected. You opt in explicitly:

```css
@import "tailwindcss";
@config "../../tailwind.config.js";
```

Note: some JS-only options (`corePlugins`, `safelist`, `separator`) are no longer supported in v4 — their CSS-native replacements are covered in [Part 5](#6-part-5--theming--config) and [Part 7](#8-part-7--engine-internals-jit--content-detection).
([Upgrade Guide](https://tailwindcss.com/docs/upgrade-guide))

### Advanced

**Real Scenario — the missing-import trap.** If you migrate a v3 codebase to v4 by just swapping the package but leave the old three-directive syntax in place, the build either errors outright or silently produces no utility CSS at all, depending on which package version you have installed. The fix is always the same: replace all three `@tailwind` directives with the single `@import "tailwindcss";` line — there is no partial/hybrid syntax.

**Preprocessor gotcha:** v4 is not designed to run *through* Sass, Less, or Stylus — it wants to be the last step in your CSS pipeline. Projects that compile `.scss` files through Tailwind commonly hit broken `@apply` behavior or build errors, tracked in [tailwindlabs/tailwindcss#20209](https://github.com/tailwindlabs/tailwindcss/issues/20209). If you have a legacy Sass setup, plan to remove the preprocessor layer, not layer Tailwind on top of it.

[⬆ back to top](#table-of-contents)

---

## 3. Part 2 — Utility-First Fundamentals

Covers: reading utility class names, the default spacing/color scales, arbitrary values.

### Beginner

Tailwind utility names follow a `property-value` pattern you can usually guess:

```html
<div class="p-4 m-2 bg-blue-500 text-white rounded-lg shadow-md">
  padding: 1rem · margin: 0.5rem · blue background · white text · rounded corners · shadow
</div>
```

- `p-4` → `padding: 1rem` (Tailwind's spacing scale is in multiples of `0.25rem`, so `4` = `1rem`)
- `bg-blue-500` → background color, `blue` palette, shade `500` (shades run 50 → 950, lighter to darker)
- `text-white`, `rounded-lg`, `shadow-md` → text color, border radius, box shadow, each on their own small named scale

**Try it:** open the playground from Part 0 and change `p-4` to `p-8`, then to `p-1`. Watch the padding jump in fixed steps — that's the scale, not arbitrary pixel-pushing.

### Working Knowledge

**Arbitrary values** escape the scale with square brackets when a design genuinely needs an exact, non-scale value:

```html
<div class="w-[327px] bg-[#1da1f2] top-[calc(100%-4px)]">
  exact pixel width · exact hex color · a calc() expression
</div>
```

**Wrong vs. right — reaching for arbitrary values too early:**

```html
<!-- Wrong — arbitrary value used for something the scale already covers -->
<div class="p-[16px]">...</div>

<!-- Right — use the scale; it's consistent across your whole app and themeable later -->
<div class="p-4">...</div>
```

Arbitrary values are for the genuine one-offs (a specific design-spec pixel value, a third-party widget's fixed size) — reaching for them by default defeats the point of having a shared scale.
([Adding Custom Styles — Arbitrary Values](https://tailwindcss.com/docs/adding-custom-styles#using-arbitrary-values))

### Advanced

**Real Scenario.** A teammate hardcodes `mt-[13px]` next to your `mt-4` (`1rem` = `16px`) because a Figma export said "13px margin." Six months later the design system changes its base spacing unit, and every `mt-4` in the app gets remapped automatically via the theme — except that one `mt-[13px]`, which now looks subtly wrong forever. This is the real cost of arbitrary values: they opt a value *out* of the design system's evolution. Reserve them for values that are genuinely supposed to be fixed (an icon's exact pixel dimensions, a `z-index` matching a third-party library).

[⬆ back to top](#table-of-contents)

---

## 4. Part 3 — Responsive & State Variants

Covers: the variant-prefix system — responsive breakpoints, pseudo-class states, `group`/`peer`, stacking multiple variants.

### Beginner

Any utility can be prefixed with a **variant** to apply it conditionally:

```html
<button class="bg-blue-500 hover:bg-blue-700 text-white p-2 rounded">
  Hover me
</button>
```

`hover:bg-blue-700` means "apply `bg-blue-700` only on hover" — Tailwind generates `.hover\:bg-blue-700:hover { background-color: ...; }` and you never write that selector by hand.

### Working Knowledge

**Responsive variants** use breakpoint prefixes, and Tailwind is **mobile-first**: an unprefixed utility applies at all sizes, and a breakpoint-prefixed utility overrides it *from that width up*.

```html
<div class="text-sm md:text-base lg:text-lg">
  Small on mobile, base size from 768px up, large from 1024px up
</div>
```

**Wrong vs. right — fighting mobile-first instead of using it:**

```html
<!-- Wrong — trying to be "desktop-first" by overriding down, which Tailwind's
     breakpoints don't do (md: means "md and up", not "up to md") -->
<div class="text-lg md:text-sm">

<!-- Right — start with the mobile style unprefixed, layer larger sizes up -->
<div class="text-sm md:text-lg">
```

**Combining state + responsive variants** stacks left-to-right in v4 (this order flipped from v3 — see [Part 7](#8-part-7--engine-internals-jit--content-detection)):

```html
<button class="md:hover:bg-blue-700">
  Only applies the hover style at md breakpoint and above
</button>
```

**`group` and `peer`** let one element's state style a *different* element — the two escape hatches you reach for constantly once you stop writing custom CSS:

```html
<div class="group">
  <p class="text-gray-500 group-hover:text-black">
    Hovering the parent div changes this text's color
  </p>
</div>

<input type="checkbox" class="peer" />
<p class="hidden peer-checked:block">Shown only when the checkbox is checked</p>
```
([Hover, Focus, and Other States](https://tailwindcss.com/docs/hover-focus-and-other-states))

### Advanced

**Real Scenario — nested groups.** You have a card inside a list item, and both the card and the list item need their own independent hover styling for different children. A single unnamed `group` class breaks here because a nested `group-hover:` would ambiguously react to *either* ancestor. The fix is **named groups**: `group/list` and `group/card`, referenced as `group-hover/list:` and `group-hover/card:` respectively — this is the kind of thing that only bites you once you're past toy examples, and the naming mechanism exists specifically for it.

```html
<li class="group/list">
  <div class="group/card">
    <span class="opacity-0 group-hover/card:opacity-100">card-hover only</span>
    <span class="opacity-0 group-hover/list:opacity-100">list-hover only</span>
  </div>
</li>
```
([Differentiating Nested Groups](https://tailwindcss.com/docs/hover-focus-and-other-states#differentiating-nested-groups))

[⬆ back to top](#table-of-contents)

---

## 5. Part 4 — Layout Patterns (Flex & Grid)

Covers: flex/grid utility mapping to CSS, common layout recipes.

### Beginner

Flexbox and Grid utilities map almost 1:1 to their CSS properties, so if you know the CSS you already know most of the utility names:

```html
<div class="flex items-center justify-between gap-4">
  <!-- display: flex; align-items: center; justify-content: space-between; gap: 1rem; -->
  <span>Left</span>
  <span>Right</span>
</div>
```

### Working Knowledge

**Try it — build a responsive card grid:**

```html
<div class="grid grid-cols-1 md:grid-cols-3 gap-6 p-6">
  <div class="bg-gray-100 rounded-lg p-4">Card 1</div>
  <div class="bg-gray-100 rounded-lg p-4">Card 2</div>
  <div class="bg-gray-100 rounded-lg p-4">Card 3</div>
</div>
```

Resize your browser window (or the "Try It" playground) below 768px and watch the grid collapse from three columns to one — no media query written by hand.

**Wrong vs. right — flex-grow/shrink naming (v3 → v4 rename):**

```html
<!-- Wrong (v3 names — removed in v4) -->
<div class="flex-grow-0 flex-shrink-0">

<!-- Right (v4) -->
<div class="grow-0 shrink-0">
```
([Upgrade Guide — renamed utilities](https://tailwindcss.com/docs/upgrade-guide))

### Advanced

**Real Scenario.** A grid with `grid-cols-[max-content,auto]` (comma-separated, copied from a v3-era blog post) silently fails to parse in v4's arbitrary-value parser, which treats commas differently. The fix is to use underscores in place of commas inside arbitrary grid template values: `grid-cols-[max-content_auto]`. This is a small but sharp v3→v4 trap because the class doesn't error — Tailwind just doesn't generate the utility, and the grid silently falls back to browser default behavior.
([Upgrade Guide — arbitrary value changes](https://tailwindcss.com/docs/upgrade-guide))

[⬆ back to top](#table-of-contents)

---

## 6. Part 5 — Theming & Config

Covers: the `@theme` directive, theme namespaces, how it replaces `tailwind.config.js`'s `theme.extend`.

### Beginner

In v4, theming is done in CSS, not JavaScript, via `@theme`:

```css
@import "tailwindcss";

@theme {
  --color-brand: oklch(0.6 0.2 260);
  --font-display: "Poppins", sans-serif;
}
```

```html
<h1 class="text-brand font-display">Themed heading</h1>
```

Declaring `--color-brand` inside `@theme` automatically generates `text-brand`, `bg-brand`, `border-brand`, etc. — one CSS variable becomes an entire family of utilities.
([Theme Variables — Tailwind CSS](https://tailwindcss.com/docs/theme))

### Working Knowledge

**Theme namespaces** determine which utility family a variable feeds:

| Namespace | Feeds utilities like |
|---|---|
| `--color-*` | `bg-*`, `text-*`, `border-*`, `ring-*` |
| `--font-*` | `font-*` (family) |
| `--text-*` | `text-*` (size) |
| `--spacing-*` | `p-*`, `m-*`, `gap-*`, `w-*`, `h-*` |
| `--breakpoint-*` | `sm:`, `md:`, `lg:` variants |
| `--radius-*` | `rounded-*` |
| `--shadow-*` | `shadow-*` |
| `--ease-*` | `ease-*` (transition timing) |

**Wrong vs. right — migrating a v3 `theme.extend` mentally to v4:**

```js
// Wrong mental model — this is v3's tailwind.config.js and no longer the default path in v4
module.exports = {
  theme: {
    extend: {
      colors: { brand: '#1da1f2' },
    },
  },
}
```

```css
/* Right — v4 CSS-first equivalent, additive by default just like extend was */
@theme {
  --color-brand: #1da1f2;
}
```

To fully **replace** a default scale instead of extending it, reset the namespace first:

```css
@theme {
  --color-*: initial;      /* wipe all default colors */
  --color-brand: #1da1f2;  /* only your color exists now */
}
```
([Theme Variables — Overriding the default theme](https://tailwindcss.com/docs/theme#overriding-the-default-theme))

### Advanced

**`@theme inline`** forces referenced values to be inlined directly into generated utility CSS at build time, instead of left as a live `var()` reference. This matters when a theme variable points at *another* CSS variable that isn't guaranteed to resolve in every context — the canonical example is forwarding a font variable from a framework's font-loading system:

```css
@theme inline {
  --font-sans: var(--next-font-sans);
}
```

Without `inline`, some cascade contexts (e.g. inside `::before`/`::after` content or certain shadow DOM boundaries) can fail to resolve the nested `var()` chain correctly.

**`@theme static`** forces *all* declared theme variables into the compiled CSS output as real `:root` custom properties, even ones no utility class ends up using — useful when other, non-Tailwind CSS or JavaScript needs to read `--color-brand` via `getComputedStyle`, since by default Tailwind only emits variables its own generated utilities actually reference.
([Theme Variables — @theme inline and static](https://tailwindcss.com/docs/theme))

**Real Scenario.** A component library reads Tailwind's resolved theme in JavaScript at runtime — a common v3 pattern using the `resolveConfig` helper. That helper is removed entirely in v4 (there's no JS config object to resolve). The v4-native replacement is reading the CSS custom property directly at runtime: `getComputedStyle(document.documentElement).getPropertyValue('--color-brand')`. This is a real migration surprise for anyone whose design-system tooling depended on `resolveConfig`.
([Upgrade Guide — resolveConfig removed](https://tailwindcss.com/docs/upgrade-guide))

[⬆ back to top](#table-of-contents)

---

## 7. Part 6 — Reusability Strategies

Covers: when (and when not) to reach for `@apply`, component-class extraction, and class-merging helper patterns.

### Beginner

Repeating the same 8 utility classes across 20 buttons feels wrong at first — the instinct is to extract a `.btn` class:

```css
@import "tailwindcss";

@layer components {
  .btn {
    @apply px-4 py-2 rounded-lg bg-blue-600 text-white font-medium;
  }
}
```

```html
<button class="btn">Save</button>
```

`@apply` lets you compose utilities into a named CSS class, for the cases where a genuinely reusable, semantic name earns its keep (a design-system-level "primary button," not every one-off div).
([Functions and Directives — @apply](https://tailwindcss.com/docs/functions-and-directives#apply))

### Working Knowledge

**Wrong vs. right — reaching for `@apply` as the default instead of the exception:**

```html
<!-- Wrong — extracting a CSS class for something used exactly once,
     which just re-introduces the "hunt down the class definition" problem
     Tailwind was meant to remove -->
<div class="one-off-hero-wrapper">...</div>
```

```html
<!-- Right — for a one-off, keep the utilities inline; extract to @apply only
     once a pattern repeats across many call sites with the *same* intent -->
<div class="flex flex-col items-center gap-6 py-24">...</div>
```

The Tailwind team's own guidance: prefer component extraction at the *framework* level (a real `<Button>` component in React/Vue) over `@apply`-based CSS classes, when you're in a component-based framework — `@apply` is really for teams without componentization (server-rendered templates, email HTML, etc.) or for small structural bits shared across many templates.
([Reusing Styles](https://tailwindcss.com/docs/reusing-styles))

### Advanced

**`@reference` — a v4-specific `@apply` gotcha.** In v4, `@apply` no longer implicitly sees your project's theme/utilities when used inside a separate CSS file (e.g. a CSS Module imported by a single component). You must explicitly reference the file that defines your theme:

```css
/* Wrong — Button.module.css, v4, without a reference */
.btn {
  @apply bg-brand text-white; /* error: bg-brand is unknown here */
}
```

```css
/* Right — tell this file where the theme/utilities come from */
@reference "../app.css";

.btn {
  @apply bg-brand text-white;
}
```

This is one of the most commonly hit v3→v4 migration surprises, widely reported in [tailwindlabs/tailwindcss discussion #16429](https://github.com/tailwindlabs/tailwindcss/discussions/16429) and [issue #16346](https://github.com/tailwindlabs/tailwindcss/issues/16346) — v3 users are used to `@apply` "just working" everywhere because the JS config was globally available; v4's CSS-native model makes context explicit instead.

**Real Scenario — class-merging helpers.** In component-based frameworks, a very common non-`@apply` reusability pattern is a small "merge classes, later wins" utility (`clsx` + `tailwind-merge`, often combined into a `cn()` helper) so a component can accept a `className` prop that correctly *overrides* conflicting default utilities instead of both classes existing side-by-side and fighting on specificity (which, since utilities are all single-property and equal-specificity, resolves by *source order* — a subtle bug source if you just string-concatenate class lists):

```ts
function cn(...classes: (string | undefined | false)[]) {
  return twMerge(clsx(...classes))
}

// Button.tsx
<button className={cn('px-4 py-2 bg-blue-600', className)} />
// caller passes className="bg-red-600" and correctly gets red, not a CSS fight
```

This pattern is framework tooling, not a Tailwind feature — but it's the de facto answer to "how do I make a reusable component whose default classes can be overridden," which `@apply` doesn't solve at all.

[⬆ back to top](#table-of-contents)

---

## 8. Part 7 — Engine Internals: JIT & Content Detection

Covers: how Tailwind decides which CSS to generate, the Rust-based Oxide engine, why v4 needs no `content: []` array.

### Beginner

Tailwind doesn't ship a giant pre-built stylesheet with every possible utility — that would be megabytes of unused CSS. Instead, it scans your project's source files, looking for strings that look like utility classes, and generates **only the CSS for classes it actually finds**. This is why editing HTML and saving triggers a rebuild: the engine is re-scanning.

```
Your source files (HTML/JSX/etc.)
        │
        ▼
  ┌─────────────┐      "found: bg-blue-500, p-4, hover:text-white..."
  │  Oxide scan  │ ───────────────────────────────────────────────┐
  │ (plain-text  │                                                 ▼
  │  regex-like) │                                        ┌─────────────────┐
  └─────────────┘                                        │ Generate only    │
                                                           │ matching utility │
                                                           │ CSS rules        │
                                                           └─────────────────┘
```

### Working Knowledge

**v4's biggest setup change: no `content: []` array.** In v3, you had to manually tell Tailwind which file globs to scan:

```js
// v3 — tailwind.config.js (no longer needed in v4)
module.exports = {
  content: ['./src/**/*.{html,js,jsx,ts,tsx}'],
}
```

In v4, the Rust engine automatically scans your entire project as plain text, intelligently **respecting `.gitignore`** and skipping `node_modules`, binary files, CSS files, and lockfiles — no config needed for the common case.
([Detecting Classes in Source Files](https://tailwindcss.com/docs/detecting-classes-in-source-files))

**When you still need `@source`** — anything git-ignored that you still want scanned (e.g. a generated file, or a UI component library living in `node_modules`):

```css
@import "tailwindcss";
@source "../node_modules/@my-org/ui-lib";
```

**Wrong vs. right — assuming auto-detection covers everything:**

```html
<!-- Wrong — fully dynamic class construction; the scanner can't see
     "bg-red-600" because it never appears as a literal string -->
<div class={`bg-${color}-600`}>...</div>
```

```html
<!-- Right — use a static lookup so every literal class string exists
     somewhere in your source for the scanner to find -->
<div class={colorClasses[color]}>...</div>
```

```ts
const colorClasses = {
  red: 'bg-red-600',
  blue: 'bg-blue-600',
} as const
```

This "can't see runtime-constructed strings" limitation exists in both v3 and v4 — it's inherent to scanning source as text rather than executing it — but it's worth calling out because it's the single most common "why isn't my class working" bug for beginners.
([Dynamic Class Names](https://tailwindcss.com/docs/detecting-classes-in-source-files#dynamic-class-names))

### Advanced

**`@source inline(...)` — the v4 replacement for v3's `safelist`.** When you genuinely need to force-generate classes that never appear literally in source (e.g. classes assembled entirely server-side and injected as raw HTML), v4 replaces v3's config-based `safelist` array with a CSS-native, brace-expansion-capable directive:

```css
@source inline("bg-red-{50,{100..900..100},950}");
/* generates bg-red-50, bg-red-100, bg-red-200, ... bg-red-900, bg-red-950 */
```

`@source not inline(...)` does the inverse — explicitly block generation of matching classes.
([Safelisting Specific Utilities](https://tailwindcss.com/docs/detecting-classes-in-source-files#safelisting-specific-utilities))

### Mastery

**Cascade layers — why override order "just works" in v4.** Generated CSS is organized into real [CSS cascade layers](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer): `theme`, `base`, `components`, `utilities`, in that priority order (utilities always win regardless of source order in your CSS file, because cascade layers — unlike plain specificity — are ordered by *declaration*, not by where a rule physically sits). This is *why* a utility class can reliably override a `@layer components` class even if the component CSS comes later in the file — something that required careful manual ordering in pre-layer CSS.

```css
@layer theme, base, components, utilities;
/* Tailwind declares this order internally; anything you add via
   @utility auto-inserts into the utilities layer, guaranteeing it
   beats components regardless of where you wrote it */
```
([Adding Custom Styles — Layers](https://tailwindcss.com/docs/adding-custom-styles))

**Real Scenario — the `space-x-*` performance rewrite.** In large, deeply nested lists, v3's `space-x-*`/`space-y-*` utilities used a sibling-combinator selector (`:not([hidden]) ~ :not([hidden])`) that scales quadratically (O(n²)) with sibling count in the browser's style-matching engine — a real, measurable jank source in large lists. v4 rewrote this to `:not(:last-child)`, which is far cheaper to match but behaves subtly differently if you're dynamically hiding/reordering children (e.g. an item hidden via the `hidden` attribute in the middle of a list no longer collapses spacing the same way v3 did). This is a genuinely interesting case of a utility's *semantics* changing purely because of a performance rewrite — worth knowing if you ever see spacing look "off by one gap" after upgrading a large dynamic list.
([Upgrade Guide — space-between selector](https://tailwindcss.com/docs/upgrade-guide))

[⬆ back to top](#table-of-contents)

---

## 9. Part 8 — Plugins & the Custom Utility API

Covers: writing custom utilities, the difference between v3 plugin functions and v4's `@utility` directive.

### Beginner

For a simple custom utility that doesn't fit any existing pattern, v4 lets you define it directly in CSS with `@utility` — no plugin file, no JS:

```css
@utility content-auto {
  content-visibility: auto;
}
```

```html
<div class="content-auto">...</div>
```

### Working Knowledge

**Functional custom utilities** (ones that take a value, like Tailwind's own `p-4`) use `--value()` to pull from a theme namespace or accept arbitrary values:

```css
@theme {
  --tab-size-github: 8;
}

@utility tab-* {
  tab-size: --value(--tab-size-*, integer, [integer]);
}
```

```html
<pre class="tab-github">...</pre>   <!-- theme value -->
<pre class="tab-4">...</pre>        <!-- bare integer -->
<pre class="tab-[6]">...</pre>      <!-- arbitrary value -->
```

**Wrong vs. right — v3 plugin mental model vs. v4:**

```js
// Wrong mental model for v4 — this is v3's plugin() function with addUtilities,
// which still works via a JS plugin file but is no longer the default/simplest path
plugin(function ({ addUtilities }) {
  addUtilities({
    '.content-auto': { 'content-visibility': 'auto' },
  })
})
```

```css
/* Right — v4's CSS-native equivalent for the simple case */
@utility content-auto {
  content-visibility: auto;
}
```

JS plugins (`addUtilities`, `addComponents`, `addVariant`) still exist and are still the right tool for genuinely complex, programmatic, or third-party-distributed logic — `@utility` is the CSS-native option for the common case, not a full replacement for the plugin system.
([Adding Custom Styles — Custom Utilities](https://tailwindcss.com/docs/adding-custom-styles#custom-utilities))

### Advanced

**Real Scenario.** You want a `tab-github` utility that resolves to a *theme* value, but also want callers to be able to drop in a one-off arbitrary integer for a special case. The `--value()` function's multi-argument form (`--value(--tab-size-*, integer, [integer])`) is designed exactly for this layered fallback: try the theme namespace first, then a bare typed value, then an arbitrary bracketed value — one utility definition serving three calling conventions, which is the kind of API design that only becomes clear once you've hit the need for it firsthand.

[⬆ back to top](#table-of-contents)

---

## 10. Part 9 — Dark Mode & Container Queries

Covers: the `dark:` variant, class-based vs. OS-preference dark mode, container queries built into core.

### Beginner

`dark:` is a variant like any other, and by default follows the OS-level `prefers-color-scheme` media query — no config needed:

```html
<div class="bg-white dark:bg-gray-900 text-black dark:text-white">
  Switches automatically with OS light/dark setting
</div>
```

**Try it:** toggle your OS between light and dark mode while this is open in a browser and watch it flip live — that's a plain CSS media query under the hood, no JavaScript involved.
([Dark Mode](https://tailwindcss.com/docs/dark-mode))

### Working Knowledge

**Manual/class-based dark mode** (a user-toggleable switch, independent of OS setting) needs one line of CSS config in v4 — this replaces v3's `darkMode: 'class'` JS option entirely:

```css
@import "tailwindcss";
@custom-variant dark (&:where(.dark, .dark *));
```

```html
<html class="dark">
  <!-- now dark: utilities apply because of this class, not OS preference -->
</html>
```

**Wrong vs. right — the v3 config option no longer exists:**

```js
// Wrong — v3 syntax, has no effect in v4 (no tailwind.config.js read by default)
module.exports = { darkMode: 'class' }
```

```css
/* Right — v4 CSS-native equivalent */
@custom-variant dark (&:where(.dark, .dark *));
```
([Dark Mode — Overriding the default variant](https://tailwindcss.com/docs/dark-mode#overriding-the-default-variant))

### Advanced

**Container queries are built into core in v4** — no `@tailwindcss/container-queries` plugin needed (that plugin is now a legacy install step for v3 projects only). Mark a parent as a query container, then use `@`-prefixed variants on its descendants that respond to *the container's* width, not the viewport's:

```html
<div class="@container">
  <div class="flex flex-col @md:flex-row">
    <!-- stacks vertically until the *container* (not the page) hits md width -->
  </div>
</div>
```

This matters for genuinely reusable components: a sidebar-embedded card and a full-width card can use identical container-query classes and each responds correctly to *its own* available space, whereas a viewport-based `md:` variant would treat both cards identically regardless of where they're actually placed.

**Named containers** solve the same nested-ambiguity problem `group/name` solves for hover state:

```html
<div class="@container/sidebar">
  <div class="@container/main">
    <div class="hidden @md/sidebar:block">Shown when the sidebar container hits md</div>
  </div>
</div>
```
([Container Queries — Tailwind CSS](https://tailwindcss.com/docs/responsive-design#container-queries))

[⬆ back to top](#table-of-contents)

---

## 11. Cheat Sheets

### v3 → v4 renamed/changed utilities

| v3 | v4 | Notes |
|---|---|---|
| `@tailwind base/components/utilities` | `@import "tailwindcss";` | Single import replaces three directives |
| `tailwind.config.js` + `theme.extend` | `@theme { }` in CSS | JS config still supported via `@config`, no longer default |
| `content: []` | *(automatic)* | Use `@source` only for git-ignored/extra paths |
| `darkMode: 'class'` | `@custom-variant dark (&:where(.dark, .dark *));` | CSS-native |
| `safelist: []` | `@source inline("...")` | Supports brace/range expansion |
| `flex-shrink-*` / `flex-grow-*` | `shrink-*` / `grow-*` | Renamed |
| `overflow-ellipsis` | `text-ellipsis` | Renamed |
| `outline-none` | `outline-hidden` | `outline-none` now truly means `outline-style: none` |
| `shadow-sm` / `shadow` | `shadow-xs` / `shadow-sm` | Whole scale shifted down, new `xs` added |
| `ring` (default 3px, blue-500) | `ring` (default 1px, currentColor) | Use `ring-3` for old width |
| `bg-opacity-*` etc. | `bg-black/50` (slash syntax) | Opacity-suffix utilities removed |
| `!flex` (important prefix) | `flex!` (important suffix) | Moved to suffix |
| `bg-[--brand-color]` | `bg-(--brand-color)` | Arbitrary CSS-variable syntax |
| `@tailwindcss/container-queries` plugin | Built into core | Just use `@container` / `@md:` |
| `resolveConfig()` JS helper | *(removed)* | Read `getComputedStyle(...).getPropertyValue('--x')` instead |

### Theme namespace quick reference

| Namespace | Utilities generated |
|---|---|
| `--color-*` | `bg-*`, `text-*`, `border-*`, `ring-*`, `fill-*`, `stroke-*` |
| `--font-*` | `font-*` (family) |
| `--text-*` | `text-*` (size) |
| `--spacing-*` | `p-*`, `m-*`, `gap-*`, `w-*`, `h-*`, `top/right/bottom/left-*` |
| `--breakpoint-*` | `sm:` `md:` `lg:` `xl:` `2xl:` variants |
| `--radius-*` | `rounded-*` |
| `--shadow-*` | `shadow-*` |
| `--ease-*` | `ease-*` |
| `--animate-*` | `animate-*` |

### Variant quick reference

| Variant | Meaning |
|---|---|
| `hover:` `focus:` `active:` | Pseudo-class states |
| `sm:` `md:` `lg:` `xl:` `2xl:` | Min-width breakpoints (mobile-first, "and up") |
| `dark:` | Dark mode (OS or `.dark` class, per config) |
| `group-hover:` / `group-hover/name:` | Style based on an ancestor's hover state |
| `peer-checked:` / `peer-checked/name:` | Style based on a sibling's state |
| `@md:` / `@md/name:` | Container-query variant |
| `first:` `last:` `odd:` `even:` | Structural pseudo-classes |

[⬆ back to top](#table-of-contents)

---

## 12. Suggested Learning Order

1. **Orientation + Part 1** — get the CLI/Vite pipeline running; internalize the single-import setup.
2. **Part 2 — Utility-First Fundamentals** — read class names fluently before anything else matters.
3. **Part 3 — Responsive & State Variants** — `hover:`, breakpoints, `group`/`peer`; this unlocks 80% of real layouts.
4. **Part 4 — Layout Patterns** — flex/grid utilities; build a few real page layouts hands-on.
5. **Part 5 — Theming & Config** — `@theme`, namespaces; set up a small custom palette/font from scratch.
6. **Part 6 — Reusability Strategies** — `@apply` vs. component extraction vs. `cn()`; this is where taste matters most.
7. **Part 7 — Engine Internals** — read once for the mental model, revisit when a class "mysteriously doesn't work."
8. **Part 8 — Plugins & `@utility`** — skip until you actually need a utility Tailwind doesn't ship.
9. **Part 9 — Dark Mode & Container Queries** — quick, high-payoff; do this whenever a real project needs either.

[⬆ back to top](#table-of-contents)

---

## 13. Quick Self-Check

- Can you explain, without looking, why `@import "tailwindcss";` replaced three separate `@tailwind` directives?
- What's the difference between `@theme { --color-brand: ... }` and `@theme inline { --color-brand: ... }` — and when does the difference actually matter?
- Why does `class={`bg-${color}-600`}` fail to generate CSS, and what's the fix?
- What generates `text-brand`, `bg-brand`, and `border-brand` from a single `--color-brand` declaration?
- When would you extract a `.btn { @apply ... }` class instead of leaving utilities inline — and what has to be true about the CSS file for `@apply` to see your theme at all?
- Why does a utility class reliably override a `@layer components` class regardless of which one appears later in the compiled CSS?
- What's the difference between `hover:` and `group-hover:`, and when do you need the named form (`group-hover/name:`)?
- Why does `@container` + `@md:` respond differently than plain `md:` for a sidebar-embedded component?
- Name two utilities whose *default value* changed between v3 and v4 (not just renamed — the actual default behavior differs).

[⬆ back to top](#table-of-contents)
