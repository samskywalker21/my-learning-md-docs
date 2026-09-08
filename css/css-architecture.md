# Modern CSS — Architecture: Custom Properties, Layers & Scope (Parts 11–12)

The Parts that matter once a stylesheet outgrows a single file. Custom properties as a real runtime API rather than "variables"; `@layer` and `@scope` as the native answers to specificity wars and BEM; native nesting and how it differs from the Sass you may have seen; and an honest look at how all of it interoperates with Tailwind v4.

> **Spec:** this doc follows the canonical spec in [`css-mastery-guide.md`](./css-mastery-guide.md#about-this-document). Assumes [Foundations Part 2](./css-foundations.md#part-2--selectors--the-cascade) — specificity and the cascade sort order — more than any other prerequisite in this set.
>
> **When to read this:** the moment your CSS exceeds roughly 200 lines. Not before. These Parts solve problems that have to be *felt* before the solutions read as anything but ceremony.

---

## Table of Contents

- [Part 11 — Custom Properties & Theming](#part-11--custom-properties--theming)
  - [Beginner: declaring and using](#p11-beginner)
  - [Working Knowledge: fallbacks, scoping, and theming](#p11-working)
  - [Advanced: inheritance, invalid values, and the JS bridge](#p11-advanced)
  - [Mastery: `@property`, typed properties, and animating them](#p11-mastery)
  - [Part 11 cheat sheet](#part-11-cheat-sheet)
- [Part 12 — Layers, Scope, Nesting & Methodology](#part-12--layers-scope-nesting--methodology)
  - [Beginner: `@layer` and why it exists](#p12-beginner)
  - [Working Knowledge: native nesting](#p12-working)
  - [Advanced: `@scope`, and `@supports` as a progressive-enhancement tool](#p12-advanced)
  - [Mastery: methodologies, design tokens, and Tailwind v4 interop](#p12-mastery)
  - [Part 12 cheat sheet](#part-12-cheat-sheet)

---

## Part 11 — Custom Properties & Theming

<a id="p11-beginner"></a>

### Beginner: declaring and using

A custom property is any property whose name starts with `--`. It holds an arbitrary token stream, and `var()` substitutes it:

```css
:root {
  --brand: oklch(60% 0.16 265);
  --space: 1rem;
  --radius: 8px;
}

.button {
  background: var(--brand);
  padding: var(--space);
  border-radius: var(--radius);
}
```

Three things that are *not* true of preprocessor variables and are true here:

1. **They are live.** Change the value — in a media query, a class, a state, or from JavaScript — and everything using it updates. There is no compile step to re-run.
2. **They inherit.** A custom property set on `.card` is visible to every descendant of `.card`, and nowhere else.
3. **They are case-sensitive.** `--myColor` and `--mycolor` are different properties.

`:root` is the conventional home for global tokens — it is the `<html>` element, selected with a pseudo-class so it carries `0-1-0` specificity.

<a id="p11-working"></a>

### Working Knowledge: fallbacks, scoping, and theming

**`var()` takes a fallback** after a comma, and the fallback can itself contain a `var()`:

```css
color: var(--fg, black);
color: var(--fg, var(--text, black));
```

The comma splits only once — everything after the first comma is the fallback, commas included. So `var(--font, Helvetica, Arial, sans-serif)` is valid and the fallback is the whole three-family list. But this common-looking form is a bug:

```css
/* WRONG - reads as: use --a, or fall back to the literal "--b, pink" */
color: var(--a, --b, pink);

/* RIGHT */
color: var(--a, var(--b, pink));
```

**Scoping by inheritance** is the real design pattern, and it is what makes custom properties a component API rather than a global bag:

```css
.button {
  --btn-bg: var(--brand);
  --btn-fg: white;
  background: var(--btn-bg);
  color: var(--btn-fg);
}

.button--danger { --btn-bg: crimson; }
.button--ghost  { --btn-bg: transparent; --btn-fg: var(--brand); }

.toolbar--dark .button { --btn-bg: white; --btn-fg: black; }
```

Every variant re-declares one *value*, never a property. The `background`/`color` declarations exist exactly once. That is the whole trick, and it scales: a component with six theming hooks has six custom properties and one block of real declarations.

**Theming** then falls out of it. Combined with `light-dark()` from [Part 9](./css-color-visuals.md#p9-working):

```css
:root {
  color-scheme: light dark;
  --surface: light-dark(#fff, #131313);
  --text:    light-dark(#1a1a1a, #ececec);
}
[data-theme="high-contrast"] {
  --surface: white;
  --text: black;
}
```

Note the pattern: the *media query or attribute* changes only variable values; no component rule is duplicated.

**Try It.**

```html
<style>
  .btn { --btn-bg: rebeccapurple; --btn-fg: white;
         background: var(--btn-bg); color: var(--btn-fg);
         border: 0; padding: .6rem 1.2rem; border-radius: 6px; font: inherit; }
  .btn:hover { --btn-bg: oklch(from rebeccapurple calc(l - .08) c h); }
  .danger { --btn-bg: crimson; }
  .panel { --btn-bg: white; --btn-fg: black; background: #222; padding: 1rem; }
</style>
<button class="btn">Default</button>
<button class="btn danger">Danger</button>
<div class="panel"><button class="btn">Inside a dark panel</button></div>
```

**Expected:** three visually different buttons and one hover state, from a single `background`/`color` pair. The third button is restyled by its *ancestor*, without any selector naming the button from outside — that is inheritance doing the work a `.panel .btn { background: … }` rule would otherwise do, at higher specificity and with more coupling.

<a id="p11-advanced"></a>

### Advanced: inheritance, invalid values, and the JS bridge

**Custom properties always inherit** (unless registered otherwise — see [Mastery](#p11-mastery)). Combined with the `*` selector this creates a classic footgun:

```css
/* WRONG - forces every element to re-declare it, killing inheritance
   and creating a large amount of style work */
* { --gap: 1rem; }
```

Set tokens on `:root` or on the component that owns them, never with `*`.

**Invalid at computed-value time** is the behaviour that produces the most confusing custom-property bugs. If `var()` substitutes something the property can't accept, the declaration is not simply ignored — the property becomes *invalid at computed-value time*, which means it takes the **inherited** value if the property inherits, or the **initial** value if it doesn't ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_cascading_variables/Using_CSS_custom_properties)):

```css
:root { --size: 16; }         /* oops - no unit */
p { font-size: var(--size); } /* not "ignored": p inherits the parent's font-size */
```

The practical consequence is that a typo in a custom property can silently produce a *different* value rather than falling back to your earlier declaration — the previous rule in the cascade never gets a chance, because this one won and then failed. `@property` with an `initial-value` is the fix, and DevTools shows the property struck through with a warning when this happens.

A related trap: **`var()` cannot be used everywhere.** It works in property *values* only, so none of these work:

```css
/* All WRONG - var() is not permitted here */
@media (width > var(--bp)) { }        /* not in media queries */
.card { var(--prop): red; }           /* not as a property name */
.--card { }                           /* not as a selector */
content: "Hello " var(--name);        /* string concatenation doesn't work this way */
```

For breakpoints, the `@custom-media` proposal is not shipped; use a preprocessor or repeat the value. For runtime-varying breakpoints, container queries plus container units ([Part 7](./css-responsive-typography.md#p7-mastery)) usually make the need go away.

**The JavaScript bridge** is what makes custom properties genuinely different from a preprocessor:

```js
// read
const brand = getComputedStyle(document.documentElement)
  .getPropertyValue("--brand").trim();

// write - scoped to whatever element you set it on
document.documentElement.style.setProperty("--brand", "oklch(65% .2 20)");
card.style.setProperty("--x", `${event.offsetX}px`);
```

That last line is the pattern behind mouse-tracking spotlight effects, drag positions and scroll-linked values: JavaScript writes one number, CSS does all the rendering. It is dramatically cheaper than JS writing `style.transform` on many elements, because one custom-property write can drive dozens of derived declarations.

<a id="p11-mastery"></a>

### Mastery: `@property`, typed properties, and animating them

By default a custom property is an untyped token stream. The browser has no idea whether `--x` is a colour, a length or a word — which is why **you cannot animate or transition a plain custom property**: there is nothing to interpolate between.

`@property` registers a custom property with a type ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@property), Baseline newly available since July 2024):

```css
@property --progress {
  syntax: "<percentage>";
  inherits: false;
  initial-value: 0%;
}
```

| Descriptor | Required | Purpose |
|---|---|---|
| `syntax` | Yes | The grammar: `"<color>"`, `"<length>"`, `"<number>"`, `"<angle>"`, `"<length> \| <percentage>"`, `"*"` (untyped) |
| `inherits` | Yes | Whether it inherits — the only way to make a custom property *not* inherit |
| `initial-value` | Yes, unless `syntax: "*"` | Must be **computationally independent** — `10px` is fine, `3em` is not |

Registration buys three things:

1. **Animation and transition.** The browser now knows how to interpolate.
2. **Type checking.** An invalid value is rejected and the `initial-value` used, instead of the confusing invalid-at-computed-value-time inheritance above.
3. **Non-inheriting custom properties**, which are cheaper and prevent leakage into subtrees.

The headline use is animating something CSS otherwise cannot — a gradient position, a conic sweep, a hue:

```css
@property --angle {
  syntax: "<angle>";
  inherits: false;
  initial-value: 0deg;
}

.spinner {
  background: conic-gradient(from var(--angle), transparent, var(--brand));
  animation: spin 2s linear infinite;
}
@keyframes spin { to { --angle: 360deg; } }
```

Without the `@property` block, `--angle` is an unknown token and the animation snaps from 0 to 360 with no intermediate frames. With it, it sweeps.

```css
/* WRONG - nothing animates; --pct is an opaque string */
.bar { background: linear-gradient(90deg, var(--brand) var(--pct), #eee var(--pct)); transition: --pct .3s; }

/* RIGHT - register it first */
@property --pct { syntax: "<percentage>"; inherits: false; initial-value: 0%; }
.bar { transition: --pct .3s ease; }
```

**Try It.**

```html
<style>
  @property --pct { syntax: "<percentage>"; inherits: false; initial-value: 0%; }
  .bar { --pct: 15%; block-size: 1.5rem; inline-size: 20rem; border-radius: 999px;
         background: linear-gradient(90deg, rebeccapurple 0 var(--pct), #e5e5e5 var(--pct) 100%);
         transition: --pct .6s ease; }
  .bar:hover { --pct: 85%; }
</style>
<div class="bar"></div>
```

**Expected:** hovering sweeps the bar smoothly from 15% to 85%. Delete the `@property` block and reload — it jumps instantly. Same CSS otherwise; the registration is the entire difference.

### Part 11 cheat sheet

| Task | Syntax |
|---|---|
| Declare a global token | `:root { --brand: … }` |
| Use with fallback | `var(--fg, black)` |
| Nested fallback | `var(--a, var(--b, pink))` — never `var(--a, --b, pink)` |
| Component API | Set `--btn-bg` in variants; declare `background: var(--btn-bg)` once |
| Theme by ancestor | `.panel--dark { --btn-bg: white }` |
| Read from JS | `getComputedStyle(el).getPropertyValue("--x")` |
| Write from JS | `el.style.setProperty("--x", "12px")` |
| Make one non-inheriting | `@property` with `inherits: false` |
| Make one animatable | `@property` with a concrete `syntax` |
| Animate a gradient sweep | `@property --angle { syntax: "<angle>" … }` |

| Trap | Reality |
|---|---|
| `* { --x: … }` | Kills inheritance and adds style cost — use `:root` |
| Typo produces a fallback to my earlier rule | No — invalid at computed-value time inherits or goes initial |
| `var()` in a media query | Not supported |
| `var()` as a property name or selector | Not supported |
| Transitioning an unregistered custom property | Does nothing — register it first |
| `initial-value: 3em` | Rejected — must be computationally independent |

[↑ Back to top](#table-of-contents)

---

## Part 12 — Layers, Scope, Nesting & Methodology

<a id="p12-beginner"></a>

### Beginner: `@layer` and why it exists

Before cascade layers, the only way to make rule A beat rule B was to give A higher specificity or `!important`. That is why stylesheets accumulate `.page .sidebar ul li a.link` selectors and `!important` flags: not because anyone wanted them, but because the cascade offered no other lever.

`@layer` adds that lever ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer), widely available since March 2022). Declare the order once, at the top of your stylesheet:

```css
@layer reset, base, components, utilities;
```

Then put rules into layers, in any order and any file:

```css
@layer components {
  .button { background: var(--brand); padding: 1rem; }
}

@layer reset {
  * { margin: 0; }
}
```

**A later-declared layer beats an earlier one regardless of specificity.** A `0-1-0` rule in `utilities` beats a `1-1-1` rule in `components`. That single fact removes the entire specificity arms race: you stop competing on selector weight and start declaring intent.

Three more rules:

- **Unlayered styles win over all layers** (for normal declarations). Anything you write outside a layer sits above the whole stack. That is a sensible default — your ad-hoc override still works — but it means that if you adopt layers you should put *everything* in one, or the unlayered remainder quietly outranks your `utilities`.
- **Re-opening a layer doesn't move it.** `@layer components { … }` later in the file appends to the layer at its originally declared position.
- **`!important` inverts the order.** An `!important` in the *first* layer beats an `!important` in the last. See [Foundations Part 2 Mastery](./css-foundations.md#p2-mastery) for the full ladder — and note the practical use: a `reset` layer can hold genuinely unbreakable rules.

```css
/* Put a third-party stylesheet in the lowest layer and beat it with plain classes */
@import url("some-framework.css") layer(vendor);
@layer vendor, components;
```

That is the killer application. A vendor stylesheet full of `#id .thing` selectors becomes trivially overridable by your `.thing` class, with no `!important` and no selector escalation.

**Try It.**

```html
<style>
  @layer base, theme;
  @layer theme { p { color: seagreen; } }
  @layer base  { body #main p.text { color: crimson; } }
</style>
<div id="main"><p class="text">which colour?</p></div>
```

**Expected:** **green**. The crimson rule has specificity `1-1-1` and the green one `0-0-1`, but `theme` is declared after `base` and layer order is checked before specificity. Swap the two names in the `@layer base, theme;` line and reload — it flips to crimson without either selector changing.

<a id="p12-working"></a>

### Working Knowledge: native nesting

Nesting is now part of CSS itself, no preprocessor required ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_nesting/Using_CSS_nesting)):

```css
.card {
  padding: 1rem;
  background: var(--surface);

  & .title { font-weight: 600; }
  & > img  { inline-size: 100%; }

  &:hover { border-color: var(--brand); }

  @media (width > 40em) {
    display: grid;
    grid-template-columns: 120px 1fr;
  }

  .theme-dark & { border-color: #333; }   /* & at the end reverses the context */
}
```

`&` is the parent selector. It is required when building a *compound* selector (`&:hover`, `&.is-open`) and when the parent needs to appear anywhere other than the front. A nested selector with no `&` and no combinator gets the descendant relationship implicitly, so `.title { }` inside `.card` means `.card .title`.

**Two ways it differs from Sass, and they both matter:**

**1. No string concatenation.** Sass's `&__title` BEM shorthand does not exist in CSS, and cannot:

```css
/* WRONG - invalid; there is no way to build ".card__title" from ".card" */
.card { &__title { } }

/* RIGHT - write it out */
.card { }
.card__title { }
```

MDN explains why: without a combinator, a nested selector is parsed as a *type* selector, and type selectors must come first in a compound. If you are wedded to BEM's `__` naming, nesting will not shorten it — which is one of several reasons `@scope` (below) is a better fit for native CSS.

**2. `&` carries `:is()` specificity.** In Sass, `&` is textual substitution, so `.a { &.b { } }` compiles to `.a.b` with specificity `0-2-0`. In CSS, `&` is defined as `:is(<parent selector list>)`, which takes the **most specific** selector in the list:

```css
/* Specificity of the nested rule is :is(.card, #featured).title = 1-0-1,
   NOT 0-1-1 as the flat ".card .title" would be */
.card, #featured {
  & .title { color: red; }
}
```

If a parent selector list contains an ID, every nested rule inherits that weight. This is the single most common surprise when moving from Sass to native nesting, and it is a good argument for keeping selector lists in nested contexts homogeneous.

**Depth discipline.** Nesting makes deep selectors effortless to write and just as costly as they always were. A useful rule: **nest states, media queries and direct structural children; do not nest to mirror your DOM tree.** Three levels is plenty; if you are at five, the file is telling you something.

<a id="p12-advanced"></a>

### Advanced: `@scope`, and `@supports` as a progressive-enhancement tool

**`@scope`** bounds a set of rules to a DOM subtree, with an optional lower bound — the native answer to the problem BEM, CSS Modules and Vue's `scoped` all solve by generating unique names ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@scope), Baseline newly available since March 2026).

```css
@scope (.card) {
  :scope { padding: 1rem; border-radius: 8px; }
  img    { inline-size: 100%; }
  .title { font-weight: 600; }
}
```

Inside the block, `img` means "an `img` inside a `.card`" — but with the specificity of a bare `img`. That is the crucial property: **bare selectors and `&` inside `@scope` behave as if wrapped in `:where(:scope)`, so they contribute zero specificity.** Writing `:scope img` explicitly *does* add class-level weight (`0-1-1`), so you have both options.

The **lower bound** creates a "donut" — styles apply from the root down, but stop before the limit:

```css
/* Style images in article bodies, but not images inside a figure */
@scope (.article-body) to (figure) {
  img { border: 4px solid; }
}
```

That is something no selector can express: there is no "descendant of A but not descendant of B" combinator.

**Scoping proximity** is a new cascade tiebreak, sitting *below* specificity and above source order ([Foundations Part 2 Mastery](./css-foundations.md#p2-mastery)). When two scoped rules of equal specificity both match, the one whose scope root is **fewer DOM hops away** wins:

```css
@scope (.light-theme) { p { color: black; } }
@scope (.dark-theme)  { p { color: white; } }
```

Nest a `.light-theme` inside a `.dark-theme` and the paragraphs inside the light block are black — the nearer scope root wins, regardless of which rule was written first. Theme nesting, which used to need careful selector ordering, now just works.

`@scope` also works inline in HTML, scoping to its own parent element with no selector at all:

```html
<div class="widget">
  <style>
    @scope {
      :scope { border: 1px solid; }
      p { margin: 0; }
    }
  </style>
  <p>Scoped to this widget only.</p>
</div>
```

**`@supports`** is the tool that makes adopting any of this safe ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@supports)). It tests a real declaration, a selector, an at-rule, or a font technology:

```css
@supports (display: grid) { }
@supports not (display: grid) { }
@supports selector(:focus-visible) { }
@supports at-rule(@scope) { }
@supports (color: oklch(from white l c h)) { }
@supports (display: grid) and (not (display: inline-grid)) { }
```

Two patterns to internalise:

```css
/* Pattern 1: write the fallback plainly, enhance inside the query.
   Anything that doesn't understand @supports also doesn't understand the new thing. */
.gallery { display: flex; flex-wrap: wrap; }
@supports (grid-template-rows: subgrid) {
  .gallery { display: grid; grid-template-rows: subgrid; }
}

/* Pattern 2: negative query, for when the modern form is the base
   and the fallback is the exception. */
.hero { block-size: 100svh; }
@supports not (block-size: 100svh) {
  .hero { block-size: 100vh; }
}
```

Most modern CSS does not need `@supports` at all, because unknown declarations are simply dropped and the previous one in the cascade wins — that *is* the fallback:

```css
.hero { background: #333; }
.hero { background: color-mix(in oklab, #333 80%, var(--brand)); }  /* dropped if unsupported */
```

Reach for `@supports` when the feature changes an entire layout strategy (grid vs. flex), or when the new property has no natural predecessor.

<a id="p12-mastery"></a>

### Mastery: methodologies, design tokens, and Tailwind v4 interop

**The methodologies, compared honestly.** Every CSS methodology exists to solve two problems: *naming collisions* and *specificity fights*. Native CSS now solves both directly, which changes the calculus.

| Approach | Solves collisions by | Solves specificity by | Still worth it? |
|---|---|---|---|
| **BEM** (`.card__title--large`) | Manual naming convention | Flat, single-class selectors | The naming discipline is still useful; `@scope` replaces the mechanism |
| **ITCSS / layered files** | Nothing | File order + escalating specificity | Superseded by `@layer`, which does it explicitly and reversibly |
| **CSS Modules** | Build-time hashed names | Flat selectors | Still the pragmatic choice inside a component build; `@scope` is the no-build equivalent |
| **Utility-first** (Tailwind) | Single-purpose classes, no naming | Every utility is `0-1-0` | A different tradeoff, not a worse one — see below |
| **Native** (`@layer` + `@scope` + custom properties) | `@scope` | `@layer` | This is what the platform now offers with no tooling |

A reasonable native architecture, in one block:

```css
@layer reset, tokens, base, components, utilities;

@layer reset  { /* :where()-wrapped, so anything overrides it */ }
@layer tokens { :root { --brand: …; --space-1: …; } }
@layer base   { /* element defaults: body, headings, links */ }
@layer components { /* @scope blocks, one per component */ }
@layer utilities  { /* .flow > * + *, .visually-hidden, .stack */ }
```

**Design tokens** are just custom properties with a naming discipline, and the useful discipline is a two-tier one:

```css
:root {
  /* Tier 1 - primitives. Describe the value, never the use. */
  --blue-500: oklch(60% 0.16 265);
  --size-3: 1rem;

  /* Tier 2 - semantic. Describe the use, never the value. */
  --color-action: var(--blue-500);
  --color-surface: light-dark(white, #131313);
  --space-inline: var(--size-3);
}
.button { background: var(--color-action); }
```

Components reference **tier 2 only**. Rebranding then means editing tier 1; adding a theme means overriding tier 2. A component that references `--blue-500` directly has hard-coded a decision in the wrong place.

**What Sass still buys you**, now that CSS has variables, nesting and layers:

| Sass feature | Native equivalent | Verdict |
|---|---|---|
| Variables | Custom properties | Native is **better** — runtime, inheritable, themeable |
| Nesting | Native nesting | Native, with the `&`/`:is()` specificity caveat |
| Partials, `@use` | `@import` + `@layer`, or a bundler | Roughly even |
| `@mixin` with arguments | — | **No native equivalent.** The real remaining gap |
| `@each` / `@for` loops | — | **No native equivalent** (though `sibling-index()` covers some cases — [Part 13](./css-animation-performance.md)) |
| Functions, colour math | `color-mix()`, relative colour, `calc()` | Native is **better** — works on runtime values |
| Maths | `calc()`, `min()`, `max()`, `clamp()`, `round()` | Roughly even |

If you are reaching for Sass today, it should be for mixins and loops. Everything else is now a downgrade.

---

### Tailwind v4 interop

Tailwind v4 is not a competing model any more — it is built **on top of** the features in this doc, which makes mixing it with hand-written CSS much cleaner than it was in v3 ([Tailwind theme docs](https://tailwindcss.com/docs/theme)).

**What it actually is now.** A single CSS import, no JavaScript config file required:

```css
@import "tailwindcss";
```

That import brings in three files, each already assigned to a cascade layer — `theme`, `base` (preflight), and `utilities`. So Tailwind participates in the layer system described above rather than fighting it.

**`@theme` generates custom properties.** Design tokens go in an `@theme` block, which does two things: it defines a CSS custom property, and it tells Tailwind to generate the matching utility classes.

```css
@import "tailwindcss";

@theme {
  --color-mint-500: oklch(0.72 0.11 178);
  --font-display: "Inter", sans-serif;
  --breakpoint-sm: 30rem;
}
```

That produces `bg-mint-500`, `text-mint-500`, `fill-mint-500`, and so on — **and** a real `--color-mint-500` custom property you can use in hand-written CSS. The tokens are not build-time constants; they are the same live custom properties from [Part 11](#part-11--custom-properties--theming).

**The four CSS features it is built on**, all taught in this set:

| Feature | Where Tailwind uses it | Taught in |
|---|---|---|
| **Cascade layers** | `theme` / `base` / `utilities` layering, so utilities beat components without `!important` | [Part 12 Beginner](#p12-beginner) |
| **Custom properties** | Every theme token becomes one | [Part 11](#part-11--custom-properties--theming) |
| **`@property`** | Typed registration so utilities like gradients and transforms can animate | [Part 11 Mastery](#p11-mastery) |
| **`oklch()`** | The default v4 colour palette | [Part 9 Advanced](./css-color-visuals.md#p9-advanced) |

**The practical interop rules:**

1. **Write your custom CSS in a layer**, so its relationship to Tailwind's utilities is explicit rather than accidental:

    ```css
    @import "tailwindcss";

    @layer components {
      .prose-callout {
        background: var(--color-mint-500);
        padding: var(--spacing-4);
      }
    }
    ```

    Utilities are declared after components in Tailwind's order, so a utility class on the element still wins — which is what you want.

2. **Reference theme tokens by their custom property**, not by re-declaring the value:

    ```css
    /* WRONG - the value now exists in two places */
    .callout { background: oklch(0.72 0.11 178); }

    /* RIGHT */
    .callout { background: var(--color-mint-500); }
    ```

3. **Use `@theme inline` when a token's value references another variable** and you need the value substituted rather than the reference stored — the Tailwind docs call this out specifically, and it matters when a token is defined by a font-loading variable.

4. **Container queries and `@scope` compose fine with it.** Tailwind's utilities are ordinary declarations; a `@scope` block or `@container` query around them behaves normally. The one thing to watch is the [Part 7](./css-responsive-typography.md#p7-mastery) side effect: `container-type` creates a stacking context, which can affect utility-driven `z-*` classes.

5. **Don't fight the layer order with `!important`.** If a utility isn't winning, the answer is which layer your rule is in, not an important flag. This is the same lesson as [Foundations Part 2](./css-foundations.md#p2-mastery), applied.

For the framework itself — the utility vocabulary, variants, the config surface — see this repo's [`legacy/tailwindcss/`](../legacy/tailwindcss/) notes and the [official docs](https://tailwindcss.com/docs/theme). This section is deliberately only the seam between Tailwind and the native platform.

### Part 12 cheat sheet

| Task | Syntax |
|---|---|
| Declare layer order | `@layer reset, base, components, utilities;` |
| Add rules to a layer | `@layer components { … }` |
| Import a vendor sheet into a low layer | `@import url("x.css") layer(vendor);` |
| Beat a high-specificity vendor rule | Put it in an earlier layer, use a plain class |
| Nest a state | `&:hover { … }` |
| Reverse the nesting context | `.theme-dark & { … }` |
| Scope a component | `@scope (.card) { … }` |
| Scope with a hole | `@scope (.article) to (figure) { … }` |
| Style the scope root | `:scope { … }` inside the block |
| Inline scope to a parent | `<style>@scope { … }</style>` inside the element |
| Feature-detect a property | `@supports (display: grid)` |
| Feature-detect a selector | `@supports selector(:focus-visible)` |
| Feature-detect an at-rule | `@supports at-rule(@scope)` |
| Two-tier tokens | Primitive `--blue-500` → semantic `--color-action` |
| Tailwind token | `@theme { --color-mint-500: … }` → `bg-mint-500` + `var(--color-mint-500)` |

| Trap | Reality |
|---|---|
| Unlayered rules are "neutral" | They beat **every** layer |
| `!important` in the last layer wins | No — the **first** layer wins for `!important` |
| Re-opening a layer moves it | No — position is set at first declaration |
| `&__title` works in native nesting | No — no string concatenation |
| `&` has Sass specificity | No — it is `:is(parent list)`, taking the most specific |
| Bare selectors in `@scope` add specificity | No — they behave as `:where(:scope)`, zero weight |
| Tailwind is a separate system | v4 is built on layers, custom properties, `@property` and `oklch()` |

[↑ Back to top](#table-of-contents)

---

**Next:** [Animation & Performance (Parts 13–14)](./css-animation-performance.md) — how it moves, and how it stays at 60fps.
