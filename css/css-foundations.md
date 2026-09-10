# Modern CSS — Foundations (Parts 1–3)

How a declaration you type becomes a pixel on a screen: the language itself, the cascade that picks winners, and the box model and normal flow that turn winning values into geometry.

> **Spec:** this doc follows the canonical spec in [`css-mastery-guide.md`](./css-mastery-guide.md#about-this-document) — curiosity-driven framing, **Try It** exercises with stated expected results, wrong-vs-right pairs, MDN-first sourcing, tiers collapsed where they would be filler. Set up [the playground file](./css-mastery-guide.md#3-setup-your-playground-file) before you start.

---

## Table of Contents

- [Part 1 — How CSS Reaches the Page](#part-1--how-css-reaches-the-page)
  - [Beginner: rules, declarations, and attaching a stylesheet](#p1-beginner)
  - [Working Knowledge: at-rules, shorthands, and reading DevTools](#p1-working)
  - [Advanced: inheritance, initial values, and the four global keywords](#p1-advanced)
  - [Mastery: value stages, and why `em` in `font-size` behaves oddly](#p1-mastery)
  - [Part 1 cheat sheet](#part-1-cheat-sheet)
- [Part 2 — Selectors & the Cascade](#part-2--selectors--the-cascade)
  - [Beginner: the selector families](#p2-beginner)
  - [Working Knowledge: combinators, pseudo-classes, pseudo-elements](#p2-working)
  - [Advanced: specificity, `:is()`/`:where()`/`:not()`, and `:has()`](#p2-advanced)
  - [Mastery: the full cascade sort order, and why `!important` is a symptom](#p2-mastery)
  - [Part 2 cheat sheet](#part-2-cheat-sheet)
- [Part 3 — The Box Model & Normal Flow](#part-3--the-box-model--normal-flow)
  - [Beginner: the four boxes and `box-sizing`](#p3-beginner)
  - [Working Knowledge: `display`, block vs. inline, and the reset](#p3-working)
  - [Advanced: formatting contexts, margin collapsing, `overflow`](#p3-advanced)
  - [Mastery: sizing keywords and logical properties](#p3-mastery)
  - [Part 3 cheat sheet](#part-3-cheat-sheet)

---

## Part 1 — How CSS Reaches the Page

<a id="p1-beginner"></a>

### Beginner: rules, declarations, and attaching a stylesheet

CSS has exactly one basic sentence shape. Learn its parts and the vocabulary stops being intimidating:

```
  selector      declaration block
  --------   ----------------------
  .card      { color: red; margin: 8px; }
               -----  ---
               prop   value
               ------------
                declaration
```

- A **selector** says *which elements*.
- A **declaration** is `property: value`, ended with `;`.
- A **rule** (or ruleset) is a selector plus its declaration block.

That is the whole grammar. Everything else in CSS is more properties, more selector syntax, and at-rules (below).

There are three ways a rule reaches an element, and you should use exactly one of them habitually:

```html
<!-- 1. External stylesheet - what you should almost always do -->
<link rel="stylesheet" href="styles.css">

<!-- 2. A <style> element in the document - fine for learning and for critical CSS -->
<style> .card { color: red; } </style>

<!-- 3. An inline style attribute - avoid; it cannot be overridden without !important -->
<div style="color: red">…</div>
```

Option 3 is not "faster" or "more specific in a useful way" — it is a dead end, because an inline declaration beats every selector you could write (see [Part 2's cascade order](#p2-mastery)). Use it only when a *value* is genuinely computed at runtime, and prefer setting a custom property in that case ([Part 11](./css-architecture.md)).

**Try It.** Put this in your playground and confirm both rules apply:

```html
<style>
  p { color: rebeccapurple; }
  .loud { font-size: 2rem; }
</style>
<p>quiet</p>
<p class="loud">loud</p>
```

**Expected:** both paragraphs are purple; the second is twice the size. The `.loud` rule adds a property rather than replacing the `p` rule — CSS resolves **per property**, not per rule. That single fact prevents a lot of later confusion.

<a id="p1-working"></a>

### Working Knowledge: at-rules, shorthands, and reading DevTools

**At-rules** start with `@` and wrap or configure things rather than styling an element directly. You will meet all of these in this set:

| At-rule | What it does | Taught in |
|---|---|---|
| `@import` | Pulls in another stylesheet (with optional `layer()`) | Part 12 |
| `@media` | Applies rules conditionally on the viewport/device | Part 7 |
| `@container` | Applies rules conditionally on an ancestor's size or custom properties | Part 7 |
| `@supports` | Applies rules only if the browser understands a declaration | Part 12 |
| `@layer` | Declares a cascade layer and its priority | Part 12 |
| `@scope` | Bounds rules to a DOM subtree | Part 12 |
| `@font-face` | Declares a downloadable font | Part 8 |
| `@keyframes` | Declares an animation's waypoints | Part 13 |
| `@property` | Declares a *typed* custom property | Part 11 |
| `@starting-style` | Declares the "from" state for an entry transition | Part 13 |

**Shorthands** set several longhand properties at once — and, crucially, **reset every longhand they cover to its initial value**, including the ones you didn't mention. This is the single most common accidental-override bug in CSS:

```css
/* WRONG - the second rule silently wipes background-image */
.hero {
  background-image: url("photo.jpg");
  background-size: cover;
}
.hero.dark {
  background: #111;          /* resets background-image to none! */
}

/* RIGHT - set the longhand you actually mean */
.hero.dark {
  background-color: #111;
}
```

The same trap exists for `font` (resets `line-height`, `font-weight`, …), `border`, `flex`, `grid`, `transition`, `animation` and `background`. **Rule of thumb: reach for a shorthand when you are defining something from scratch, and a longhand when you are overriding one aspect of it.**

**Reading DevTools** is the actual skill of this Part. Open the Elements panel and select an element:

- The **Styles** pane lists every matching rule, most-specific first, with the source file and line. Declarations that lost the cascade are shown ~~struck through~~. If your rule is not in this list at all, your **selector** doesn't match — a Part 2 problem. If it is listed but struck through, you lost the cascade — also Part 2. If it is listed and applied but the result looks wrong, it's a Part 3 layout problem.
- The **Computed** pane shows the single winning value per property, and expanding a property shows which rule produced it. Tick "Show all" to see every property's computed value, including inherited ones.
- The **box-model diagram** at the bottom of the Computed pane is the fastest way to answer "why is this element this size".

**Try It.** Add a second, conflicting rule and watch DevTools strike one out:

```html
<style>
  p { color: rebeccapurple; }
  p { color: seagreen; }
</style>
<p>which colour?</p>
```

**Expected:** green. In the Styles pane, the first `color: rebeccapurple` is struck through with no warning icon — that is "lost on source order", the last tiebreak in the cascade.

<a id="p1-advanced"></a>

### Advanced: inheritance, initial values, and the four global keywords

Every property on every element has a value, even when you never wrote one. It comes from one of two places:

1. **Inheritance** — some properties (mostly typographic: `color`, `font-*`, `line-height`, `text-align`, `visibility`, `cursor`, and custom properties) take the parent's computed value when unspecified.
2. **The initial value** — everything else (all the box and layout properties: `margin`, `padding`, `border`, `display`, `width`, `background`, `position`) falls back to a per-property default defined in the spec, *not* to the parent's.

This is why setting `font-family` on `body` styles the whole page, but setting `border` on `body` doesn't put a border on every element. MDN documents "Inherited: yes/no" on every property page ([CSS cascade reference](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_cascade/Cascade)).

Four **global keywords** work on *every* property and let you reach these values explicitly:

| Keyword | Meaning |
|---|---|
| `inherit` | Take the parent's computed value, even for a non-inherited property |
| `initial` | Use the property's spec-defined initial value (note: `display: initial` is `inline`, not `block` — a classic trap) |
| `unset` | `inherit` if the property is inherited, `initial` otherwise |
| `revert` | Roll back to the value the **previous cascade origin** would have given — in practice, the browser's default stylesheet |

`revert` is the one people don't know and usually want. `all: unset` on a `<button>` strips it back to an unstyled inline box; `all: revert` gives you back the browser's own button.

```css
/* WRONG - "reset this button" using initial, which produces an inline box
   with no padding and no default anything, including no focus ring */
.reset-btn { all: initial; }

/* RIGHT - when you want the browser default back */
.reset-btn { all: revert; }

/* RIGHT - when you genuinely want a blank slate, re-add what you removed */
.bare-btn {
  all: unset;
  cursor: pointer;
}
.bare-btn:focus-visible { outline: 2px solid; outline-offset: 2px; }
```

That last rule matters: `all: unset` removes the default focus outline, and an element you can tab to with no visible focus indicator is a keyboard-accessibility failure ([MDN on `:focus-visible`](https://developer.mozilla.org/en-US/docs/Web/CSS/:focus-visible)). More in [Part 2](#p2-working).

**Try It.**

```html
<style>
  .parent { color: crimson; border: 2px solid crimson; }
  .child-a { }                       /* inherits colour, no border */
  .child-b { border: inherit; }      /* forced to inherit the border */
</style>
<div class="parent">
  parent
  <p class="child-a">A: red text, no border</p>
  <p class="child-b">B: red text AND a border</p>
</div>
```

**Expected:** both paragraphs are crimson (inherited), but only B has a border. That is the inherited/non-inherited split made visible.

<a id="p1-mastery"></a>

### Mastery: value stages, and why `em` in `font-size` behaves oddly

A value passes through defined stages before it becomes geometry. Knowing the names makes several confusing behaviours obvious:

```
  declared  -->  cascaded  -->  specified  -->  computed  -->  used   -->  actual
  (all the      (the one       (after         (after        (after      (after
   matching      that won)      defaulting/    relative      layout:     device
   ones)                        inherit)       units are     e.g. a %    rounding)
                                               resolved)     width in px)
```

Two consequences you will actually hit:

**1. Inherited values are the *computed* value, not the declared one.** If a parent has `font-size: 2em` and the grandparent is `16px`, the parent computes to `32px` and the child inherits `32px` — not "2em applied again". Otherwise nesting would compound infinitely.

But `em` *inside* `font-size` is special: it resolves against the **parent's** font size, whereas `em` in any other property resolves against the **element's own** font size. That is why this compounds and surprises people:

```css
/* WRONG - each level doubles: 32px, 64px, 128px … */
li { font-size: 2em; }

/* RIGHT - rem is always relative to the root, so it doesn't compound */
li { font-size: 1.25rem; }
```

**2. Percentages resolve at *used*-value time, against a containing block that may not exist yet.** `height: 50%` does nothing if the parent's height is `auto`, because there is no resolved height to take 50% of. This is the mechanism behind the perennial "why won't my `height: 100%` work" — the answer is in [Part 4](./css-layout-flex-grid.md), not in the height property.

**Try It.** Confirm both stages at once:

```html
<style>
  .outer { font-size: 20px; }
  .inner { font-size: 1.5em; padding: 1em; background: #eee; }
</style>
<div class="outer"><div class="inner">padded</div></div>
```

**Expected:** `.inner` computes `font-size: 30px` (1.5 × the *parent's* 20px), and its padding is `30px` (1em × its *own* 30px). Check both numbers in the Computed pane. Same unit, two different reference points — that is the rule in one screenshot.

### Part 1 cheat sheet

| Thing | Syntax | Note |
|---|---|---|
| Rule | `selector { prop: value; }` | Resolution is per-property, not per-rule |
| External sheet | `<link rel="stylesheet" href="…">` | The default choice |
| Inline style | `style="…"` | Beaten only by `!important`; avoid |
| Shorthand risk | `background: #111` | Resets `background-image` and friends |
| Inherited props | `color`, `font-*`, `line-height`, `visibility`, `--*` | Layout props are **not** inherited |
| Force inherit | `border: inherit` | Works on any property |
| Spec default | `display: initial` | = `inline`, **not** `block` |
| Browser default | `all: revert` | What you usually want over `all: initial` |
| Blank slate | `all: unset` | Re-add `:focus-visible` styling yourself |
| `em` in `font-size` | Relative to **parent** font size | Compounds when nested |
| `em` elsewhere | Relative to **own** font size | |
| `rem` | Relative to root font size | Never compounds |
| Rule not in DevTools Styles | Selector doesn't match | Part 2 |
| Rule struck through | Lost the cascade | Part 2 |

[↑ Back to top](#table-of-contents)

---

## Part 2 — Selectors & the Cascade

<a id="p2-beginner"></a>

### Beginner: the selector families

There are five basic selectors. Everything else is a combination of them.

```css
p            { }   /* type      - every <p> */
.card        { }   /* class     - every element with class="card" */
#checkout    { }   /* id        - the one element with id="checkout" */
[data-open]  { }   /* attribute - every element with a data-open attribute */
*            { }   /* universal - everything */
```

Attribute selectors have a small pattern language of their own, which is more useful than most people realise:

```css
[href^="https://"] { }   /* starts with */
[href$=".pdf"]     { }   /* ends with */
[href*="docs"]     { }   /* contains */
[lang|="en"]       { }   /* exactly "en", or "en-" prefixed */
[class~="card"]    { }   /* one of a space-separated list */
[data-state="on" i] { }  /* the i flag makes the match case-insensitive */
```

Write compound selectors by jamming them together with no space — `a.button[href^="/"]` means "an `<a>` that also has class `button` and whose href starts with `/`". Write a selector *list* with commas: `h1, h2, h3 { … }` applies the same block to three selectors.

**Use classes for almost everything.** IDs are legitimate for anchors and `for`/`id` form pairing, but as style hooks they cause the specificity problems that Part 12's `@layer` exists to clean up. This is a case where the "just use classes" convention and the spec agree.

<a id="p2-working"></a>

### Working Knowledge: combinators, pseudo-classes, pseudo-elements

**Combinators** describe relationships between elements:

```css
.card p        { }   /* descendant       - any p inside .card, at any depth */
.card > p      { }   /* child            - only direct children */
h2 + p         { }   /* next sibling     - the p immediately after an h2 */
h2 ~ p         { }   /* subsequent sibs  - every p after an h2, same parent */
```

There is no "previous sibling" or "parent" combinator — but `:has()` gives you both, which is why it changed so much when it landed ([Advanced](#p2-advanced) below).

**Pseudo-classes** (single colon) match a *state* or *position*:

| Selector | Matches |
|---|---|
| `:hover`, `:active`, `:focus` | Interaction state |
| `:focus-visible` | Focused *and* the browser thinks a focus ring is warranted |
| `:focus-within` | The element contains the focused element |
| `:disabled`, `:checked`, `:required`, `:invalid` | Form state |
| `:user-invalid` | Invalid, but only **after** the user has interacted — vastly better UX than `:invalid` |
| `:first-child`, `:last-child`, `:only-child` | Position among siblings |
| `:nth-child(2n+1)`, `:nth-of-type(3)` | Positional formulas |
| `:nth-child(2 of .featured)` | The 2nd element that is `.featured`, not the 2nd child that happens to be |
| `:empty` | No children, not even text |
| `:target` | Its id matches the URL fragment |
| `:root` | The `<html>` element — where you put custom properties |
| `:open` | Open `<details>`, shown `<dialog>`, showing popover — Baseline newly available, May 2026 ([web.dev](https://web.dev/blog/baseline-digest-may-2026)) |

**Pseudo-elements** (double colon) style a part of an element that has no tag of its own:

```css
.quote::before { content: "\201C"; }   /* generated content */
p::first-line  { font-variant: small-caps; }
p::first-letter{ float: left; font-size: 3em; }
::selection    { background: gold; }
::placeholder  { color: #888; }
::marker       { color: crimson; }     /* the list bullet/number */
::backdrop     { background: rgb(0 0 0 / 50%); }  /* behind a dialog/fullscreen */
```

`content` is mandatory for `::before`/`::after` — without it, nothing renders. Use `content: ""` for purely decorative shapes.

**The focus rule you must not get wrong.** `:focus` matches whenever an element is focused, including after a mouse click, which is why designers historically deleted the outline and broke keyboard navigation. `:focus-visible` matches only when the browser's heuristics say a focus indicator would help — keyboard navigation and programmatic focus, not mouse clicks ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/:focus-visible)).

```css
/* WRONG - removes the only affordance keyboard users have */
button:focus { outline: none; }

/* RIGHT - style it instead of deleting it */
button:focus-visible {
  outline: 3px solid CanvasText;
  outline-offset: 2px;
}
/* Optional fallback for very old browsers, via a feature query (Part 12) */
@supports not selector(:focus-visible) {
  button:focus { outline: 3px solid; outline-offset: 2px; }
}
```

**Try It.**

```html
<style>
  input:user-invalid { outline: 2px solid crimson; }
  input:user-invalid + .msg { display: block; }
  .msg { display: none; color: crimson; }
</style>
<input type="email" required placeholder="you@example.com">
<span class="msg">That doesn't look like an email address.</span>
```

**Expected:** the field is *not* red on page load, even though it is empty and invalid. It turns red only after you type something wrong and blur. Swap `:user-invalid` for `:invalid` and reload to see the difference — the whole form lights up red before the user has done anything. That is why `:user-invalid` exists.

<a id="p2-advanced"></a>

### Advanced: specificity, `:is()`/`:where()`/`:not()`, and `:has()`

When two rules set the same property on the same element and neither is more important, **specificity** decides. It is three numbers compared left to right, never carried like decimal digits ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_cascade/Specificity)):

| Column | Counts | Example |
|---|---|---|
| **ID** | `#id` selectors | `#main` = `1-0-0` |
| **CLASS** | classes, attribute selectors, pseudo-**classes** | `.card`, `[open]`, `:hover` = `0-1-0` |
| **TYPE** | type selectors and pseudo-**elements** | `p`, `::before` = `0-0-1` |

```
  #main .card p::before
   |     |    |   |
   |     |    |   +-- pseudo-element  -> TYPE   +1
   |     |    +------ type selector   -> TYPE   +1
   |     +----------- class           -> CLASS  +1
   +----------------- id              -> ID     +1
                                       = 1-1-2
```

The comparison is **lexicographic**: `1-0-0` beats `0-9-9`. Ten classes do not out-weigh one ID. The universal selector `*` and combinators (`>`, `+`, `~`, space) contribute nothing.

Three functional pseudo-classes change specificity in ways worth memorising:

| Selector | Specificity | Forgiving? |
|---|---|---|
| `:is(a, b)` | The **most specific** argument | Yes — an unsupported selector inside doesn't kill the rule |
| `:not(a, b)` | The **most specific** argument | No |
| `:has(a, b)` | The **most specific** argument | No |
| `:where(a, b)` | Always **`0-0-0`** | Yes |

```css
/* :is() flattens a repetitive list, but inherits the #id's weight */
:is(#main, .sidebar) p { }     /* 1-0-1 - the #main dominates */

/* :where() is the escape hatch: matches the same elements, weighs nothing */
:where(#main, .sidebar) p { }  /* 0-0-1 - trivially overridable */
```

`:where()` is the correct tool for **defaults you want people to override** — a reset, a base layer, a library's opinionated styles. Wrapping a reset in `:where()` means a single class always wins over it, with no `!important` anywhere.

**`:has()` — the parent (and previous-sibling) selector.** Widely available since December 2023 ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/:has)). It matches an element based on what it *contains* or what *follows* it:

```css
/* Style a card differently when it contains an image */
.card:has(img) { grid-template-columns: 120px 1fr; }

/* Tighten a heading's margin only when a subheading follows it */
h2:has(+ p.subtitle) { margin-block-end: 0.25rem; }

/* AND logic: chain them. OR logic: comma inside. */
form:has(:invalid):has(.touched) { border-color: crimson; }
body:has(dialog[open]) { overflow: hidden; }
```

That last line is the pattern that replaced a whole category of JavaScript: locking body scroll when a modal opens, in CSS, with no class toggling.

Two limits MDN states explicitly: `:has()` cannot be nested inside another `:has()`, and pseudo-elements are not valid inside it. And a performance caution that matters in [Part 14](./css-animation-performance.md) — anchoring `:has()` on `body` or `:root` makes the browser re-evaluate a very large subtree:

```css
/* WRONG - forces evaluation against the whole document */
body:has(.sidebar-expanded) { }

/* RIGHT - anchor it as tightly as the design allows */
.layout:has(> .sidebar-expanded) { }
```

**Try It.**

```html
<style>
  .field:has(input:focus) { outline: 2px solid dodgerblue; }
  .field { padding: .5rem; border: 1px solid #ccc; }
</style>
<label class="field">Name <input></label>
```

**Expected:** clicking into the input outlines the whole `.field` wrapper, not just the input. Before `:has()` this required either `:focus-within` (which exists, and is better here) or JavaScript — try both and note that `:focus-within` is the more specific tool for this exact case, while `:has()` generalises to anything.

<a id="p2-mastery"></a>

### Mastery: the full cascade sort order, and why `!important` is a symptom

Specificity is only step four of six. The complete order the browser uses, from the [MDN cascade reference](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_cascade/Cascade):

1. **Relevance** — filter to declarations that actually match, in media/container queries that are currently true.
2. **Origin and importance** — this is the big one, and it is *inverted* for `!important`.
3. **Cascade layers** — later-declared layers win for normal declarations, **earlier-declared layers win for `!important`** ([Part 12](./css-architecture.md)).
4. **Specificity**.
5. **Scoping proximity** — for `@scope`, fewer DOM hops to the scope root wins ([Part 12](./css-architecture.md)).
6. **Order of appearance** — last one in the source wins.

The origin/importance ladder, low to high, for author styles:

```
  lowest  1. layer A (declared first)
          2. layer B
          3. layer C (declared last)
          4. unlayered author styles
          5. inline style="" attribute
          6. CSS animations (@keyframes)
          7. unlayered author !important
          8. layer C !important
          9. layer B !important
         10. layer A !important        <- note the inversion
         11. inline style !important
  highest 12. CSS transitions
```

Above and below the author band sit the user agent (browser) and user origins, with the same inversion: normal author beats normal user beats normal UA; `!important` UA beats `!important` user beats `!important` author. That inversion exists so a user's accessibility stylesheet can always win — it is a feature, not a quirk.

Two things surprise people here:

- **Animations outrank normal declarations.** A running `@keyframes` animation beats your `.card { color: red }` regardless of specificity. That is why "my colour won't apply while the animation runs" happens.
- **Transitions outrank everything, including `!important`.** During the transition only.

**Why `!important` is a symptom, not a tool.** Every `!important` you add is a claim that you could not win the cascade legitimately, and it escalates: the next person's override needs `!important` *plus* higher specificity. The three legitimate uses are utility classes that must always win (and `@layer` does this better), overriding a third-party stylesheet you cannot edit, and user stylesheets. Everything else is better solved by:

- putting library/reset styles in an **earlier `@layer`** so your styles beat them at any specificity ([Part 12](./css-architecture.md));
- wrapping defaults in **`:where()`** so they weigh nothing;
- **removing an ID** from a selector.

**Try It — see the inversion.**

```html
<style>
  @layer base, theme;
  @layer base   { p { color: green !important; } }
  @layer theme  { p { color: red   !important; } }
</style>
<p>which colour?</p>
```

**Expected:** **green**. Without `!important` on both, `theme` (declared last) would win and you would get red. Delete both `!important`s and reload to confirm the flip. This is the clearest demonstration that `!important` does not mean "more important" — it means "search the priority list from the other end".

### Part 2 cheat sheet

| Selector | Specificity | Notes |
|---|---|---|
| `*`, `>`, `+`, `~` | `0-0-0` | Combinators are free |
| `p`, `::before` | `0-0-1` | Types and pseudo-elements |
| `.card`, `[open]`, `:hover` | `0-1-0` | Classes, attributes, pseudo-classes |
| `#main` | `1-0-0` | Beats any number of classes |
| `style="…"` | Above all selectors | Only `!important` beats it |
| `:is(x, y)` / `:not(…)` / `:has(…)` | Most specific argument | `:is()` is forgiving; the others are not |
| `:where(x, y)` | `0-0-0` | Use for resets and defaults |

| Task | Selector |
|---|---|
| Parent of something | `.card:has(img)` |
| Element before another | `h2:has(+ p)` |
| Focus ring, keyboard only | `:focus-visible` |
| Wrapper contains focus | `:focus-within` |
| Form error, after interaction | `:user-invalid` |
| Nth of a filtered set | `:nth-child(2 of .featured)` |
| Lock scroll behind a modal | `body:has(dialog[open]) { overflow: hidden }` |
| Zero-weight defaults | `:where(…)` |

| Cascade tiebreak, in order |
|---|
| 1. Relevance (matching + active media/container query) |
| 2. Origin & importance (inverted for `!important`) |
| 3. Cascade layer (inverted for `!important`) |
| 4. Specificity |
| 5. `@scope` proximity |
| 6. Source order |

[↑ Back to top](#table-of-contents)

---

## Part 3 — The Box Model & Normal Flow

<a id="p3-beginner"></a>

### Beginner: the four boxes and `box-sizing`

Every element is drawn as four nested rectangles:

```
   +-------------------------------------------+
   |                 MARGIN                    |  transparent, pushes neighbours away
   |   +-----------------------------------+   |
   |   |             BORDER                |   |  drawn, has a colour and style
   |   |   +---------------------------+   |   |
   |   |   |          PADDING          |   |   |  inside the border, shows the background
   |   |   |   +-------------------+   |   |   |
   |   |   |   |     CONTENT       |   |   |   |  text, images, child boxes
   |   |   |   +-------------------+   |   |   |
   |   |   +---------------------------+   |   |
   |   +-----------------------------------+   |
   +-------------------------------------------+
```

The question that trips everyone: **what does `width: 300px` measure?** By default, the *content* box only. So:

```css
.box {
  width: 300px;
  padding: 20px;
  border: 2px solid;
}
```

...occupies **344px** on screen (300 + 20 + 20 + 2 + 2). Add padding and the box gets *bigger*, which is almost never what a layout wants.

`box-sizing: border-box` changes `width` to mean the border box — content + padding + border. The same rule then occupies exactly **300px**, and padding eats into the content instead of growing the box.

This is why essentially every stylesheet written since about 2012 opens with:

```css
*, *::before, *::after { box-sizing: border-box; }
```

Put it in your playground and leave it there. This is a case where the near-universal convention is right and the spec default is a historical artefact.

**Try It.**

```html
<style>
  .box { width: 300px; padding: 20px; border: 2px solid crimson; background: #eee; }
  .border-box { box-sizing: border-box; }
</style>
<div class="box">content-box: 344px wide</div>
<div class="box border-box">border-box: 300px wide</div>
```

**Expected:** the second box is visibly narrower. Confirm the exact numbers in the DevTools box-model diagram — the "content" number shrinks from 300 to 256 while the outer width goes from 344 to 300.

<a id="p3-working"></a>

### Working Knowledge: `display`, block vs. inline, and the reset

`display` is really **two** values: how the box behaves in its parent (**outer**), and how it lays out its own children (**inner**). Modern CSS lets you write both, and understanding the split explains a lot:

```css
display: block flow;        /* == display: block  */
display: inline flow;       /* == display: inline */
display: block flow-root;   /* == display: flow-root */
display: block flex;        /* == display: flex   */
display: inline flex;       /* == display: inline-flex */
display: block grid;        /* == display: grid   */
```

The single-keyword forms are what you will write and read in practice; the two-value form is the mental model. **The crucial consequence: `display: flex` on a parent changes how its *children* are laid out, not how the parent itself sits in the page.**

| Outer role | Behaviour |
|---|---|
| `block` | Takes the full inline width available; starts on a new line; `width`/`height`/all margins apply |
| `inline` | Flows with text; `width`/`height` are **ignored**; vertical padding/border render but don't push neighbours; horizontal margins do apply |
| `inline-block` | Flows with text, but `width`/`height`/margins apply normally |

That "`width` is ignored on inline elements" rule is behind a large share of beginner confusion:

```css
/* WRONG - nothing happens; <span> is inline */
span.badge { width: 80px; height: 24px; }

/* RIGHT */
span.badge { display: inline-block; width: 80px; height: 24px; }
```

**A minimal, honest reset.** Browsers ship a default stylesheet, and it is mostly good — headings should be bold, lists should have markers. You do not need a 400-line reset. Four rules cover the genuinely annoying defaults:

```css
*, *::before, *::after { box-sizing: border-box; }
body { margin: 0; }                         /* the default 8px body margin */
img, picture, svg, video { display: block; max-width: 100%; }
input, button, textarea, select { font: inherit; }  /* form controls don't inherit fonts */
```

The `img { display: block }` line deserves a note: images are `inline` by default, which means they sit on the text baseline, which leaves a mysterious ~4px gap under every image in a container. Making them `block` removes it. (`vertical-align: middle` also fixes it, if you need images to stay inline.)

**Try It — the phantom gap.**

```html
<style>
  .frame { background: crimson; width: 200px; }
  .frame img { width: 100%; }
</style>
<div class="frame"><img src="data:image/gif;base64,R0lGODlhAQABAIAAAP///wAAACH5BAEAAAAALAAAAAABAAEAAAICRAEAOw==" alt="" style="height:100px"></div>
```

**Expected:** a red sliver visible below the image — that is the baseline gap. Add `.frame img { display: block }` and it vanishes.

<a id="p3-advanced"></a>

### Advanced: formatting contexts, margin collapsing, `overflow`

**Normal flow** is the default layout algorithm, and it has two modes:

- A **block formatting context (BFC)** stacks block boxes vertically, each filling the available inline width.
- An **inline formatting context (IFC)** lays text and inline boxes out horizontally into line boxes, wrapping when they run out of room.

An element establishes a *new* BFC — an independent layout island — when it is a float, absolutely positioned, an `inline-block`, a table cell, has `overflow` other than `visible`, or is `display: flow-root`. `flow-root` is the modern, side-effect-free way to say "make a new BFC", and it is the correct answer to two classic problems: containing floats, and stopping margin collapse.

**Margin collapsing** is the mechanism that surprises people most. Adjacent vertical margins *merge into one*, taking the larger value rather than adding. It happens in three cases ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_box_model/Mastering_margin_collapsing)):

1. **Adjacent siblings** — a `1.5rem` bottom margin next to a `1rem` top margin produces `1.5rem` of space, not `2.5rem`.
2. **Parent and first/last child** — a child's `margin-top` escapes through a parent that has no `border-top`, `padding-top`, or inline content before it, and pushes the *parent* down instead.
3. **Empty blocks** — a block with no height, border, padding or content collapses its own top and bottom margins together.

Case 2 is the one that produces "why is there space *above* my container?":

```html
<div class="card">
  <h2>Title</h2>   <!-- h2 has a default margin-top -->
</div>
```

```css
/* WRONG diagnosis: "the h2 has a weird margin" - you delete it and the design breaks elsewhere */

/* RIGHT - stop the collapse at the boundary. Any one of these works: */
.card { padding-block-start: 1px; }   /* padding blocks it - crude */
.card { border-block-start: 1px solid transparent; }  /* border blocks it - crude */
.card { display: flow-root; }          /* establishes a BFC - clean, no side effects */
.card { display: flex; flex-direction: column; }  /* flex/grid disable it entirely */
```

**Margin collapsing does not happen at all in flex or grid containers.** That is a large part of why modern layouts feel more predictable: choosing `display: grid` for a card's internals removes an entire class of bug. It also does not happen for floated or absolutely positioned elements, and only ever applies to *vertical* (block-direction) margins — horizontal margins never collapse.

**`overflow`** decides what happens when content doesn't fit:

| Value | Behaviour |
|---|---|
| `visible` (default) | Content spills out and is still painted |
| `hidden` | Clipped, not scrollable by the user (but still programmatically scrollable) |
| `clip` | Clipped, and creates no scroll container at all |
| `scroll` | Always shows scrollbars |
| `auto` | Scrollbars when needed |

The important side effect: **any `overflow` value other than `visible` establishes a BFC and a scroll container.** That is why `overflow: hidden` on an ancestor silently breaks `position: sticky` on a descendant — a trap covered in [Part 4](./css-layout-flex-grid.md). Use `overflow: clip` when you want clipping *without* the scroll container.

**Try It — margin collapse, then stop it.**

```html
<style>
  .card { background: #cfe; }
  .card h2 { margin-block-start: 2rem; background: gold; }
</style>
<div class="card"><h2>Title</h2></div>
```

**Expected:** the gold heading sits flush with the top of the blue card, and there is 2rem of *white* space above the card — the margin escaped. Now add `.card { display: flow-root }` and reload: the white space moves inside, the blue card grows, and the heading sits 2rem down from its top. Same declaration, opposite result — the difference is only whether a BFC exists.

<a id="p3-mastery"></a>

### Mastery: sizing keywords and logical properties

Beyond lengths and percentages, `width` and `height` accept **intrinsic sizing keywords** that ask the content how big it wants to be:

| Keyword | Meaning |
|---|---|
| `min-content` | The narrowest the box can be without overflowing — for text, the longest word |
| `max-content` | The width it would take if it never wrapped |
| `fit-content` | `min(max-content, max(min-content, available))` — shrink-wrap, but never overflow |
| `stretch` | Fill the containing block (the modern, better-defined replacement for the `width: 100%` + `box-sizing` dance) |

```css
/* A caption that shrink-wraps to its text but wraps rather than overflowing */
figcaption { width: fit-content; max-width: 100%; }
```

These matter enormously in grid and flex, where they are the default track and item sizing behaviours — [Parts 5 and 6](./css-layout-flex-grid.md) lean on them constantly.

**Logical properties** replace physical directions (top/right/bottom/left) with flow-relative ones (block/inline start/end), so a layout works unchanged in right-to-left and vertical writing modes ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_logical_properties_and_values)):

```
  horizontal-tb (English)          vertical-rl (some CJK)
  +------------------------+       +------------------------+
  |  block-start (top)     |       |  inline-start (top)    |
  |i                      i|       |b                      b|
  |n                      n|       |l                      l|
  |l  inline axis  ------> |       |o   block axis <------  |
  |i                      i|       |c                      c|
  |  block-end (bottom)    |       |  inline-end (bottom)   |
  +------------------------+       +------------------------+
```

| Physical | Logical |
|---|---|
| `width` / `height` | `inline-size` / `block-size` |
| `margin-top` / `margin-bottom` | `margin-block-start` / `margin-block-end`, or both via `margin-block` |
| `margin-left` / `margin-right` | `margin-inline-start` / `margin-inline-end`, or both via `margin-inline` |
| `padding-*`, `border-*` | Same `-block-`/`-inline-` pattern |
| `top`/`right`/`bottom`/`left` | `inset-block-start` … , or the shorthands `inset`, `inset-block`, `inset-inline` |
| `text-align: left` | `text-align: start` |

Two of these earn their place even in a monolingual English site:

```css
/* The old two-property horizontal-centring idiom */
.container { margin-left: auto; margin-right: auto; }

/* One property, and correct in RTL for free */
.container { margin-inline: auto; }
```

```css
/* Vertical rhythm without a leading-margin exception on the first child */
.prose > * + * { margin-block-start: 1.5rem; }
```

*This subsection is Working Knowledge → Mastery only; there is no separate Advanced tier for logical properties, because their advanced material is really [Part 8's](./css-responsive-typography.md) writing-mode work applied.*

**Try It.**

```html
<style>
  .box { inline-size: 200px; padding-inline: 1rem; border-inline-start: 4px solid crimson; }
</style>
<div class="box">Left border in LTR…</div>
<div class="box" dir="rtl">…right border in RTL, same CSS.</div>
```

**Expected:** the crimson bar is on the left of the first box and on the **right** of the second, with no extra CSS. Swap `border-inline-start` for `border-left` and reload — now both bars are on the left, and the RTL box looks wrong.

### Part 3 cheat sheet

| Problem | Fix |
|---|---|
| Padding makes boxes wider than I set | `*, *::before, *::after { box-sizing: border-box }` |
| `width`/`height` ignored | The element is `inline` — use `inline-block` or `block` |
| Mystery ~4px gap under an image | `img { display: block }` (or `vertical-align: middle`) |
| Space appearing *outside* a container | Margin collapse — add `display: flow-root` to the container |
| Floats not contained by the parent | `display: flow-root` |
| Sticky element stopped working | An ancestor has `overflow` other than `visible` — see [Part 4](./css-layout-flex-grid.md) |
| Want clipping but no scroll container | `overflow: clip` |
| Centre a block horizontally | `margin-inline: auto` |
| Shrink-wrap to content, but wrap | `width: fit-content` |
| Form controls using the wrong font | `input, button, textarea, select { font: inherit }` |

| Margin collapsing | Happens? |
|---|---|
| Adjacent siblings, vertical margins | Yes |
| Parent / first child, no border or padding between | Yes |
| Empty block with no height or border | Yes |
| Horizontal margins | Never |
| Inside a flex or grid container | Never |
| Floated or absolutely positioned elements | Never |
| Element establishing a BFC (`flow-root`, `overflow` != `visible`) | Blocked at that boundary |

| Sizing keyword | Effect |
|---|---|
| `min-content` | Narrowest without overflow (longest word) |
| `max-content` | Width with no wrapping |
| `fit-content` | Shrink-wrap, capped at available space |
| `stretch` | Fill the containing block |

[↑ Back to top](#table-of-contents)

---

**Next:** [Layout — Positioning, Flexbox & Grid (Parts 4–6)](./css-layout-flex-grid.md) applies all three of these Parts. Everything there assumes you can explain the cascade, the box model, and what a formatting context is.
