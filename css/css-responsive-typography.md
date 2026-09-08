# Modern CSS — Responsive Design & Typography (Parts 7–8)

How a layout adapts to the space it is given, and how text behaves once it gets there. Units and what each one is actually relative to, fluid sizing with `clamp()`, media queries in their modern form, **container queries** — the feature that changed responsive design more than anything since flexbox — and then the typography layer that most stylesheets never get right.

> **Spec:** this doc follows the canonical spec in [`css-mastery-guide.md`](./css-mastery-guide.md#about-this-document). Assumes [Foundations](./css-foundations.md) and [Layout](./css-layout-flex-grid.md).

---

## Table of Contents

- [Part 7 — Responsive Design & Container Queries](#part-7--responsive-design--container-queries)
  - [Beginner: units, and what each is relative to](#p7-beginner)
  - [Working Knowledge: media queries, modern syntax](#p7-working)
  - [Advanced: `clamp()`, fluid type, and intrinsic sizing](#p7-advanced)
  - [Mastery: container queries, size and style](#p7-mastery)
  - [Part 7 cheat sheet](#part-7-cheat-sheet)
- [Part 8 — Typography & Text](#part-8--typography--text)
  - [Beginner: the font stack and the vertical-rhythm properties](#p8-beginner)
  - [Working Knowledge: `@font-face`, `font-display`, variable fonts](#p8-working)
  - [Advanced: line breaking, `text-wrap`, truncation](#p8-advanced)
  - [Mastery: OpenType features, `field-sizing`, writing modes](#p8-mastery)
  - [Part 8 cheat sheet](#part-8-cheat-sheet)

---

## Part 7 — Responsive Design & Container Queries

<a id="p7-beginner"></a>

### Beginner: units, and what each is relative to

A CSS length is a number and a unit, and **the whole skill is knowing what the unit is relative to** ([MDN `<length>`](https://developer.mozilla.org/en-US/docs/Web/CSS/length)).

**Absolute:** `px` is the only one you should use. It is not a physical pixel — it is 1/96 of a CSS inch, a fixed reference the browser scales for device pixel ratio and zoom. `cm`, `in`, `pt`, `pc`, `mm`, `Q` exist for print and are wrong on screen.

**Font-relative** — the ones you will actually reach for:

| Unit | Relative to | Use it for |
|---|---|---|
| `em` | The element's own `font-size` (but the **parent's** when used *in* `font-size` — see [Foundations Part 1](./css-foundations.md#p1-mastery)) | Padding and spacing that should scale with the text it surrounds |
| `rem` | The **root** `font-size` | Almost all sizing. Never compounds. |
| `ch` | The width of the `0` glyph | Measure (line length): `max-inline-size: 65ch` |
| `lh` | The element's computed `line-height` | Vertical rhythm, badge heights matching a line of text |
| `rlh` | The root's `line-height` | Page-wide vertical rhythm |
| `ex`, `cap`, `ic` | x-height, cap height, CJK ideograph width | Fine typographic alignment |

`lh` and `rlh` reached Baseline **widely available** in May 2026 ([web.dev digest](https://web.dev/blog/baseline-digest-may-2026)) — recent enough that older material won't mention them.

**The accessibility rule: never set `font-size` in `px`.** A user who has raised their browser's default font size gets no benefit from `px` type. Use `rem` for type and let the root be the browser default.

```css
/* WRONG - ignores the user's font-size preference */
body { font-size: 16px; }
h1   { font-size: 32px; }

/* RIGHT - scales with the user's setting */
body { font-size: 1rem; }
h1   { font-size: 2rem; }
```

**Outdated convention:** the `html { font-size: 62.5% }` trick (making `1rem` = `10px` for easier arithmetic) is still widely recommended. It works, but it overrides the user's chosen base size by a fixed ratio and makes every `rem` in your stylesheet mean something different from `rem` everywhere else on the web. Do the arithmetic instead, or use `calc()`.

**Viewport units** are percentages of the viewport: `vw`, `vh`, `vmin`, `vmax`, plus the logical `vi`/`vb`. Each has three variants for mobile browsers whose UI chrome expands and retracts:

| Variant | Meaning |
|---|---|
| `svh` | **S**mall — viewport height with browser UI **expanded** (the smallest it gets) |
| `lvh` | **L**arge — with UI retracted (the largest) |
| `dvh` | **D**ynamic — updates live as the UI moves |

`100vh` on mobile is the classic bug: it equals `100lvh`, so a "full-height" hero is taller than the visible area and its bottom is hidden under the browser chrome.

```css
/* WRONG on mobile - bottom content sits under the URL bar */
.hero { block-size: 100vh; }

/* RIGHT - the always-visible height */
.hero { block-size: 100svh; }

/* Or dvh if you want it to track the chrome - but note it can cause
   layout shifts as the user scrolls (MDN cautions about exactly this) */
.hero { block-size: 100dvh; }
```

MDN's own guidance is to prefer `svh`/`lvh` over `dvh` where stability matters, because dynamic units re-layout during scroll.

**Container units** — `cqw`, `cqh`, `cqi`, `cqb`, `cqmin`, `cqmax` — are percentages of a *query container* rather than the viewport, and are covered in [Mastery](#p7-mastery) below.

**Percentages** are not a unit but behave like one: they resolve against the containing block, and *which dimension* depends on the property. `width`, `padding`, and **all** margins resolve against the containing block's **width** — yes, including `margin-top`. That asymmetry is behind the old `padding-top: 56.25%` aspect-ratio hack, which `aspect-ratio` has now replaced.

<a id="p7-working"></a>

### Working Knowledge: media queries, modern syntax

A media query applies rules only when a condition about the *viewport or device* is true.

```css
@media (width >= 40em) {
  .layout { grid-template-columns: 200px 1fr; }
}
```

That `width >= 40em` is the **range syntax**, and it is the modern form ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@media)). It replaces `min-width`/`max-width` and reads the way you think:

```css
/* Outdated convention - still what most tutorials show */
@media (min-width: 640px) { }
@media (max-width: 639.98px) { }     /* the .98 exists to dodge overlap at 640 */

/* Modern - no off-by-one arithmetic, and both bounds in one query */
@media (width >= 40em) { }
@media (width < 40em) { }
@media (40em <= width <= 60em) { }
```

The `.98px` fudge in the old form exists because `min-width: 640px` and `max-width: 640px` both match at exactly 640px. The range syntax's `<` and `>` are genuinely exclusive, so the problem disappears.

**Use `em` for breakpoints, not `px`.** Breakpoints in `em` are relative to the browser's default font size, so a user who has enlarged their text gets the simpler layout at the point where their text actually needs it. Note that `em` in a media query always means the *initial* font size, never your `html` rule — so `40em` is a stable 640px at default settings.

Media features worth knowing beyond `width`:

| Feature | Detects |
|---|---|
| `prefers-color-scheme: dark` | OS/browser dark mode ([Part 9](./css-color-visuals.md)) |
| `prefers-reduced-motion: reduce` | User has asked for less animation ([Part 13](./css-animation-performance.md)) |
| `prefers-contrast: more` | User wants higher contrast |
| `forced-colors: active` | Windows High Contrast and similar ([Part 10](./css-color-visuals.md)) |
| `hover: hover` / `hover: none` | Whether the primary input can hover |
| `pointer: fine` / `coarse` | Mouse vs. finger — use for hit-target sizing |
| `orientation: portrait` | Viewport shape |
| `resolution: >= 2dppx` | High-density displays |

```css
/* Don't hide functionality behind hover on touch devices */
@media (hover: hover) {
  .card:hover .actions { opacity: 1; }
}
@media (hover: none) {
  .card .actions { opacity: 1; }   /* always visible on touch */
}
```

**Mobile-first**, i.e. writing the narrow layout as the base and adding `width >=` queries, remains the right default: the base styles are the simplest ones, and every query adds rather than undoes.

<a id="p7-advanced"></a>

### Advanced: `clamp()`, fluid type, and intrinsic sizing

Media queries change values in steps. **`clamp(MIN, PREFERRED, MAX)`** changes them continuously, and removes most typography breakpoints entirely.

```css
h1 {
  font-size: clamp(1.75rem, 1.2rem + 2.5vw, 3.5rem);
}
```

Read it as: never smaller than `1.75rem`, never larger than `3.5rem`, and in between scale with the viewport. The preferred value **must include a `rem` term** — a pure `vw` value (`font-size: 4vw`) does not respond to browser zoom at all, which is a WCAG failure. Mixing `rem + vw` keeps zoom working.

```css
/* WRONG - text that ignores user zoom */
h1 { font-size: clamp(1.75rem, 5vw, 3.5rem); }

/* RIGHT - the rem term keeps zoom effective */
h1 { font-size: clamp(1.75rem, 1.2rem + 2.5vw, 3.5rem); }
```

`min()` and `max()` are the simpler siblings, and both are excellent for containers:

```css
.container {
  inline-size: min(65ch, 100% - 2rem);   /* the "modern container" idiom */
  margin-inline: auto;
}
```

That one declaration replaces a `max-width` plus horizontal padding plus `box-sizing` juggle: the container is 65 characters wide, or the full width minus a 1rem gutter on each side, whichever is smaller.

**`aspect-ratio`** holds a box's shape without the old padding hack ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/aspect-ratio), widely available since September 2021):

```css
.video-frame { aspect-ratio: 16 / 9; inline-size: 100%; }
.avatar      { aspect-ratio: 1; inline-size: 3rem; border-radius: 50%; }
```

**At least one dimension must be `auto`** for it to do anything — if you set both `width` and `height`, the ratio is ignored. And on `<img>`, always also set `width` and `height` attributes in the HTML: the browser derives an intrinsic ratio from them and reserves space before the image loads, which is the main defence against layout shift.

**Intrinsic sizing** ([Foundations Part 3 Mastery](./css-foundations.md#p3-mastery)) is the other half of "responsive without breakpoints". A layout built from `fit-content`, `minmax()`, `min()` and `flex-basis` adapts because the *content* drives it, not because you enumerated screen sizes.

<a id="p7-mastery"></a>

### Mastery: container queries, size and style

Media queries ask about the **viewport**. That was always a proxy for the question you actually had, which is *how much room does this component have?* A card in a sidebar and the same card in a main column have identical viewport widths and completely different needs. **Container queries** ask the real question ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_containment/Container_queries)).

Two steps: declare a container, then query it.

```css
.card-wrapper {
  container-type: inline-size;
  container-name: card;       /* optional but recommended */
  /* shorthand: container: card / inline-size; */
}

@container card (width > 30rem) {
  .card { grid-template-columns: 160px 1fr; }
}
```

The query targets **descendants** of the container, never the container itself — that is the one structural rule. You need a wrapper element; a component cannot query its own size.

```css
/* WRONG - .card is the container, so it can't be styled by its own query */
.card { container-type: inline-size; }
@container (width > 30rem) { .card { … } }   /* never matches .card */

/* RIGHT - wrapper is the container, .card is the queried descendant */
.card-wrapper { container-type: inline-size; }
@container (width > 30rem) { .card-wrapper > .card { … } }
```

`container-type` values:

| Value | Query on | Containment applied |
|---|---|---|
| `inline-size` | Inline dimension only | layout, style, inline-size |
| `size` | Both dimensions | layout, style, size — **requires the element to have a definite block size**, or it collapses |
| `normal` | Style queries and name-only queries only | style |

Use `inline-size` by default. `size` needs the element's height to be set independently of its content, which is rarely true and produces zero-height containers when it isn't.

**Two side effects you must know**, both consequences of the containment that `container-type` applies:

1. **A size container establishes a stacking context** ([Part 4](./css-layout-flex-grid.md#p4-mastery)) — adding container queries to a component can move an overflowing dropdown behind a sibling.
2. **It becomes a containing block for `fixed` and `absolute` descendants** — a `position: fixed` child inside it will position relative to it rather than the viewport.

Both are the same class of bug as the `transform` trap in Part 4, and both are easy to miss because nothing in the Styles pane looks wrong.

**Container query units** are the real payoff, and are what makes a component genuinely self-contained:

| Unit | Meaning |
|---|---|
| `cqi` | 1% of the container's **inline** size |
| `cqb` | 1% of the container's block size |
| `cqw` / `cqh` | 1% of container width / height |
| `cqmin` / `cqmax` | The smaller / larger of `cqi` and `cqb` |

```css
@container card (width > 30rem) {
  .card h2 { font-size: max(1.5rem, 1.2rem + 2cqi); }
}
```

That heading now scales with **its container**, so the same component is correct in a sidebar, a modal and a full-width hero with no variants.

**Style queries** ask about a container's custom property value rather than its size. Baseline **newly available since May 2026** ([web.dev digest](https://web.dev/blog/baseline-digest-may-2026)) — new enough that you should feature-detect:

```css
.panel { --tone: warning; }

@container style(--tone: warning) {
  .panel__icon { color: darkorange; }
}
```

This is the native answer to "variant props in CSS": set one custom property on a wrapper, and any descendant can respond to it without a class-name convention threading through the markup. Note that style queries do **not** require `container-type` — `normal` (the default) already supports them.

**Try It — the same card, two contexts.**

```html
<style>
  .grid { display: grid; grid-template-columns: 200px 1fr; gap: 2rem; }
  .wrap { container-type: inline-size; }
  .card { display: grid; gap: .5rem; background: #cfe; padding: 1rem; }
  .card img { inline-size: 100%; aspect-ratio: 3/2; background: #789; }
  @container (width > 24rem) {
    .card { grid-template-columns: 120px 1fr; align-items: start; }
  }
</style>
<div class="grid">
  <div class="wrap"><div class="card"><img alt=""><p>In a narrow sidebar.</p></div></div>
  <div class="wrap"><div class="card"><img alt=""><p>In the wide main column.</p></div></div>
</div>
```

**Expected:** identical markup and identical CSS produce a **stacked** card in the sidebar and a **side-by-side** card in the main column, at the same viewport width. Now resize the window: the sidebar card never switches (it never gets wide enough), while the main one flips back to stacked when its own width drops below 24rem. No media query could express that.

### Part 7 cheat sheet

| Need | Use |
|---|---|
| Type size | `rem` — never `px` |
| Spacing that scales with local text | `em` |
| Line length / measure | `ch` (e.g. `max-inline-size: 65ch`) |
| Full mobile screen height | `100svh` (not `100vh`) |
| Height tracking browser chrome | `100dvh` — may cause shift during scroll |
| Page container | `inline-size: min(65ch, 100% - 2rem); margin-inline: auto` |
| Fluid type | `clamp(1.75rem, 1.2rem + 2.5vw, 3.5rem)` — always include a `rem` term |
| Fixed box shape | `aspect-ratio: 16 / 9` with one dimension `auto` |
| Viewport breakpoint | `@media (width >= 40em)` — range syntax, `em` units |
| Touch vs. mouse | `@media (hover: hover)` / `(pointer: coarse)` |
| Component-level breakpoint | `container-type: inline-size` + `@container` |
| Type scaled to its container | `cqi` units |
| Variant driven by a custom property | `@container style(--tone: warning)` |

| Container query gotcha | Note |
|---|---|
| The container can't style itself | Query targets descendants only — add a wrapper |
| `container-type: size` collapses | Needs a definite block size; prefer `inline-size` |
| A dropdown went behind a sibling | Size containers create a stacking context |
| A `fixed` child stopped being fixed | Size containers are containing blocks |
| Style queries | Work with `container-type: normal`; newly available (May 2026) |

[↑ Back to top](#table-of-contents)

---

## Part 8 — Typography & Text

<a id="p8-beginner"></a>

### Beginner: the font stack and the vertical-rhythm properties

`font-family` takes a prioritised list; the browser uses the first family it can resolve, per character.

```css
body {
  font-family: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
}
```

`system-ui` resolves to whatever the operating system uses for its own UI — San Francisco on Apple platforms, Segoe UI Variable on Windows, Roboto on Android. It costs zero bytes, never causes a layout shift, and looks native. **Start here and only add a webfont when the design genuinely requires one.**

Always end a stack with a generic family (`sans-serif`, `serif`, `monospace`, `system-ui`, `ui-monospace`). Quote family names containing spaces.

The four properties that do most of the work:

```css
body {
  font-size: 1rem;
  line-height: 1.5;      /* unitless - see below */
  font-weight: 400;
  letter-spacing: normal;
}
```

**`line-height` should be unitless.** A unitless value is a multiplier that each element computes against *its own* font size. A value with units computes once and is then inherited as a fixed length, so a large heading inheriting `line-height: 24px` gets overlapping lines:

```css
/* WRONG - the h1 inherits a fixed 24px leading and its lines collide */
body { line-height: 24px; }

/* RIGHT - each element scales its own leading */
body { line-height: 1.5; }
h1   { line-height: 1.1; }   /* large text wants tighter leading */
```

Two more that matter more than their obscurity suggests: `letter-spacing` should usually be slightly *negative* on large headings and slightly positive on small caps/uppercase; and `max-inline-size: 65ch` on body text is the single highest-impact readability rule in CSS — lines longer than roughly 75 characters are measurably harder to read.

<a id="p8-working"></a>

### Working Knowledge: `@font-face`, `font-display`, variable fonts

Loading a custom font is three parts: declare it, control how it loads, and preload it.

```css
@font-face {
  font-family: "Inter";
  src: url("/fonts/inter-var.woff2") format("woff2-variations");
  font-weight: 100 900;        /* a range = a variable font */
  font-display: swap;
  font-style: normal;
}
```

**`font-display`** decides what happens during the download ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/font-display)):

| Value | Block period | Swap period | Effect |
|---|---|---|---|
| `auto` | UA decides | UA decides | Usually behaves like `block` |
| `block` | Short | Infinite | Invisible text (FOIT) until loaded — avoid |
| `swap` | Extremely small | Infinite | Fallback shows immediately, swaps when ready |
| `fallback` | Extremely small | Short | Brief window to swap, then gives up |
| `optional` | Extremely small | None | Uses the font only if it is already fast/cached |

`swap` for text that matters (no invisible text), `optional` for decorative faces (no layout shift at all). The tradeoff is real: `swap` guarantees your text is readable immediately but guarantees a reflow when the font arrives; `optional` guarantees no reflow but may never show your font on a slow connection.

To reduce the shift that `swap` causes, match the fallback's metrics to the webfont's:

```css
@font-face {
  font-family: "Inter Fallback";
  src: local("Arial");
  size-adjust: 107%;
  ascent-override: 90%;
  descent-override: 22%;
}
body { font-family: "Inter", "Inter Fallback", sans-serif; }
```

And preload the file itself, since a font is only discovered after the CSS parses:

```html
<link rel="preload" href="/fonts/inter-var.woff2" as="font" type="font/woff2" crossorigin>
```

`crossorigin` is required even for same-origin fonts — fonts are always fetched in CORS mode, and omitting it causes a *second* download.

**Variable fonts** pack an entire weight (and often width/slant) range into one file. Declare the range with `font-weight: 100 900`, then use any value in it — including non-multiples of 100:

```css
h1 { font-weight: 750; }
.tight { font-stretch: 92%; }
.custom { font-variation-settings: "wght" 620, "opsz" 32; }
```

Prefer the high-level properties (`font-weight`, `font-stretch`, `font-style`) over `font-variation-settings` where they exist, because `font-variation-settings` is a low-level override that doesn't animate or inherit as cleanly.

**Ship WOFF2 only.** Every browser that matters supports it, and the old four-format `src` stack (`eot`, `ttf`, `svg`, `woff`) is dead weight. If you want belt-and-braces, `@supports font-format(woff2)` can gate it ([MDN `@supports`](https://developer.mozilla.org/en-US/docs/Web/CSS/@supports)).

<a id="p8-advanced"></a>

### Advanced: line breaking, `text-wrap`, truncation

**`text-wrap`** controls the line-breaking algorithm ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/text-wrap), widely available since March 2024):

| Value | Behaviour |
|---|---|
| `wrap` | Default — greedy, fills each line before moving on |
| `nowrap` | No wrapping; overflows instead |
| `balance` | Evens out the number of characters per line — for **headings** |
| `pretty` | Slower algorithm that avoids orphans and bad breaks — for **body copy** |
| `stable` | Like `wrap`, but doesn't re-wrap earlier lines while editing `contenteditable` |

```css
h1, h2, h3, blockquote { text-wrap: balance; }
p { text-wrap: pretty; }
```

Two lines that fix the "heading with one word stranded on the last line" problem permanently. Note that MDN states `balance` is capped at 6 lines in Chromium and 10 in Firefox — deliberately, since balancing is expensive. That cap is why you apply it to headings, not paragraphs.

**Overflow and long strings.** Text that can't break — URLs, hashes, long German compounds — overflows its container. Three properties, often confused:

```css
.prose {
  overflow-wrap: break-word;  /* break a long word ONLY if it would overflow */
  word-break: normal;         /* leave normal word-breaking rules alone */
  hyphens: auto;              /* insert real hyphens; requires lang="" on the element */
}
```

```css
/* WRONG - breaks every line mid-word, not just the overflowing ones */
.prose { word-break: break-all; }

/* RIGHT */
.prose { overflow-wrap: break-word; }
```

`hyphens: auto` needs a `lang` attribute to pick a dictionary — `<html lang="en">` is doing more work than you thought.

**Truncation.** Single line:

```css
.truncate {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
```

All three are required; `text-overflow` alone does nothing without `overflow: hidden` and a non-wrapping context. In a flex or grid child, add `min-width: 0` ([Layout Part 5](./css-layout-flex-grid.md#p5-advanced)) or the truncation will never trigger.

Multiple lines:

```css
.clamp-3 {
  display: -webkit-box;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 3;
  overflow: hidden;
}
```

This is the one place in this whole set where a `-webkit-` prefix is still the interoperable answer — the prefixed form is what all engines implement. The unprefixed `line-clamp` is standardising; check its current status before switching.

**Try It — balance and clamp.**

```html
<style>
  .col { inline-size: 22rem; font: 1rem/1.5 system-ui, sans-serif; }
  h2 { inline-size: 22rem; }
  .balanced { text-wrap: balance; }
  .clamp-3 { display:-webkit-box; -webkit-box-orient:vertical; -webkit-line-clamp:3; overflow:hidden; background:#eee; }
</style>
<h2>A fairly long heading that will strand a word</h2>
<h2 class="balanced">A fairly long heading that will strand a word</h2>
<p class="col clamp-3">Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris.</p>
```

**Expected:** the second heading's lines are close to equal length and no single word is stranded on the last line. The paragraph shows exactly three lines and cuts off with an ellipsis.

<a id="p8-mastery"></a>

### Mastery: OpenType features, `field-sizing`, writing modes

**Numeric and OpenType features** are the difference between typography that looks designed and typography that looks defaulted:

```css
/* Numbers in a table must line up: fixed-width, same-height digits */
td.numeric { font-variant-numeric: tabular-nums lining-nums; }

/* Fractions, ordinals, small caps */
.recipe { font-variant-numeric: diagonal-fractions; }
.abbr   { font-variant-caps: all-small-caps; }

/* Disable ligatures where they hurt - e.g. in code */
code { font-variant-ligatures: none; }
```

`tabular-nums` is the one to remember: proportional digits in a right-aligned column of numbers jitter horribly, and one declaration fixes it.

`font-feature-settings` is the low-level escape hatch for features with no high-level property (`"ss01"`, `"cv11"`, `"zero"`). As with `font-variation-settings`, prefer the high-level property when one exists.

**`text-rendering`** exists and is mostly a trap: `optimizeLegibility` enables kerning and ligatures but can cause noticeable delays on long documents and has historically had rendering bugs. Modern browsers enable kerning by default; use `font-kerning` if you need explicit control.

**`field-sizing: content`** makes form controls size to their contents ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/field-sizing), Baseline newly available June 2026). It replaces the standard "grow a textarea as the user types" JavaScript entirely:

```css
textarea {
  field-sizing: content;
  min-block-size: 3lh;      /* at least three lines */
  max-block-size: 12lh;
  inline-size: 100%;
}
```

Note the interaction MDN flags: with `field-sizing: content`, the HTML `rows`/`cols`/`size` attributes stop having an effect, and an input with no `min-width` collapses to the width of the caret. Always pair it with `min-inline-size`.

**Writing modes** rotate the whole flow, and are the reason [logical properties](./css-foundations.md#p3-mastery) exist:

```css
.vertical-label {
  writing-mode: vertical-rl;
  text-orientation: mixed;    /* upright for CJK, rotated for Latin */
}
```

`writing-mode` is genuinely useful outside CJK typesetting — vertical axis labels on a chart, sidebar ribbons, rotated table headers — and it does it without `transform`, so the box actually reserves the right space (a rotated element via `transform` does not, because transforms don't affect layout; see [Part 14](./css-animation-performance.md)).

The `direction` property exists but should be set via the HTML `dir` attribute instead, so that the *document's* directionality is correct for assistive technology and for text selection, not just its appearance.

*This Part gets Beginner → Mastery with no separate Advanced/Mastery split on the font-loading material; the font-loading advanced tier is really [Part 14's](./css-animation-performance.md) rendering-performance material applied to fonts.*

### Part 8 cheat sheet

| Goal | Declaration |
|---|---|
| Zero-cost native font | `font-family: system-ui, sans-serif` |
| Correct leading | `line-height: 1.5` — **unitless** |
| Readable measure | `max-inline-size: 65ch` |
| Load a webfont without invisible text | `font-display: swap` |
| Load a webfont without layout shift | `font-display: optional` |
| Reduce swap shift | `size-adjust` / `ascent-override` on a fallback `@font-face` |
| Preload correctly | `<link rel="preload" as="font" crossorigin>` — `crossorigin` is mandatory |
| Variable font weight range | `font-weight: 100 900` in `@font-face` |
| No stranded word in a heading | `text-wrap: balance` |
| Better body-copy line breaks | `text-wrap: pretty` |
| Break long URLs | `overflow-wrap: break-word` (not `word-break: break-all`) |
| Real hyphenation | `hyphens: auto` + `lang` attribute |
| One-line truncation | `overflow: hidden; text-overflow: ellipsis; white-space: nowrap` (+ `min-width: 0` in flex) |
| Multi-line truncation | `-webkit-line-clamp: 3` with `display: -webkit-box` |
| Aligned numbers in a table | `font-variant-numeric: tabular-nums` |
| Auto-growing textarea, no JS | `field-sizing: content` + `min-inline-size` |
| Vertical text that reserves space | `writing-mode: vertical-rl` (not `transform: rotate`) |

[↑ Back to top](#table-of-contents)

---

**Next:** [Colour & Visual Effects (Parts 9–10)](./css-color-visuals.md) — perceptual colour spaces, and everything you can paint into a box.
