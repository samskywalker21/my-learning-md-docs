# Tailwind CSS — Foundations (Parts 1–3)

How a class in your markup becomes a rule in your stylesheet: the build pipeline, the utility syntax, and the variant system.

> **Spec:** this doc follows the shared spec in [`tailwindcss-mastery-guide.md`](./tailwindcss-mastery-guide.md#about-this-document) — curiosity-driven, tiered Beginner → Mastery, Tailwind **v4.3.3**, assuming the [`css/`](../css/css-mastery-guide.md) set. Set up the [playground](./tailwindcss-mastery-guide.md#4-setup-your-playground) before starting; every **Try It** below assumes it is running.

---

## Table of Contents

- [Part 1 — Setup & the Build Pipeline](#part-1--setup--the-build-pipeline)
  - [Beginner](#p1-beginner) · [Working Knowledge](#p1-working) · [Advanced](#p1-advanced) · [Mastery](#p1-mastery) · [Cheat sheet](#p1-cheat)
- [Part 2 — Utility-First Fundamentals](#part-2--utility-first-fundamentals)
  - [Beginner](#p2-beginner) · [Working Knowledge](#p2-working) · [Advanced](#p2-advanced) · [Mastery](#p2-mastery) · [Cheat sheet](#p2-cheat)
- [Part 3 — Variants](#part-3--variants)
  - [Beginner](#p3-beginner) · [Working Knowledge](#p3-working) · [Advanced](#p3-advanced) · [Mastery](#p3-mastery) · [Cheat sheet](#p3-cheat)

---

# Part 1 — Setup & the Build Pipeline

<a id="p1-beginner"></a>
## Beginner

Tailwind is a build step. It reads your source files, finds class names, and writes a CSS file. You link that CSS file. That's the entire architecture.

The minimum viable project is three files:

```css
/* src/input.css — in v4 this file IS your config */
@import "tailwindcss";
```

```bash
npx @tailwindcss/cli -i ./src/input.css -o ./src/output.css --watch
```

```html
<link href="./src/output.css" rel="stylesheet">
```

That's it. There is no `tailwind.config.js`, no `content` array, no `postcss.config.js`. If a tutorial hands you those, it is teaching v3 — see the [v3-convention table](./tailwindcss-mastery-guide.md#7-the-v3-convention-table).

### Try It: watch the generator work

With the watcher running, open `src/output.css` in your editor beside `index.html`. Add a class to the HTML:

```html
<div class="p-4">Hello</div>
```

Save. `output.css` regenerates. Search it for `.p-4` and you will find exactly this:

```css
.p-4 {
  padding: calc(var(--spacing) * 4);
}
```

Now delete the class and save again. The rule **disappears from `output.css`**. Nothing else in the file changes.

That round trip is the single most important thing to internalise in this entire set. Tailwind is not shipping a big stylesheet and hiding the unused parts — the rule genuinely does not exist until a file somewhere contains the text `p-4`. Keep `output.css` open for the rest of these docs.

<a id="p1-working"></a>
## Working Knowledge

### The four install paths

All four produce identical CSS. Pick by what's already in your build.

| Path | Install | Wire-up | Use when |
|---|---|---|---|
| **CLI** | `npm i tailwindcss @tailwindcss/cli` | `npx @tailwindcss/cli -i in.css -o out.css --watch` | No bundler. Static sites, server-rendered templates, learning |
| **Vite** | `npm i tailwindcss @tailwindcss/vite` | `plugins: [tailwindcss()]` | Anything Vite-based. The fastest path — it hooks the module graph directly |
| **PostCSS** | `npm i tailwindcss @tailwindcss/postcss` | `plugins: { "@tailwindcss/postcss": {} }` | An existing PostCSS chain you can't remove |
| **webpack** | `npm i tailwindcss @tailwindcss/webpack` | loader entry | webpack projects. **Since v4.2** |

The Vite setup, in full ([official install docs](https://tailwindcss.com/docs/installation/using-vite)):

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [tailwindcss()],
})
```

> **Since v4.2:** the first-party webpack loader `@tailwindcss/webpack` is new, and the [v4.3 release post](https://tailwindcss.com/blog/tailwindcss-v4-3) reports it at 2.17× the speed of routing webpack through PostCSS. If you are on webpack and still using `@tailwindcss/postcss`, that's the upgrade.

**Prefer the dedicated plugin over PostCSS whenever one exists.** PostCSS is a generic pipeline that Tailwind has to work through; the Vite and webpack plugins hook the bundler's own module graph and can invalidate far more precisely.

### What `@import "tailwindcss"` actually expands to

This is the question that makes the rest of the framework legible. Run a build and look at the top of `output.css`:

```css
/*! tailwindcss v4.3.3 | MIT License | https://tailwindcss.com */
@layer properties;
@layer theme, base, components, utilities;
```

That second line is the whole architecture in one statement. It declares four [cascade layers](../css/css-architecture.md) **in priority order**, and everything Tailwind generates goes into one of them:

```
  @layer theme      -->  :root { --color-red-500: oklch(...); --spacing: 0.25rem; ... }
        |                 your design tokens, as real CSS custom properties
        v
  @layer base       -->  Preflight: the cross-browser reset
        |                 *, ::before { box-sizing: border-box; margin: 0; border: 0 solid }
        v
  @layer components -->  EMPTY unless you put something here
        |                 the layer that exists so utilities can beat your components
        v
  @layer utilities  -->  .p-4 { padding: ... }   .flex { display: flex }   ...
                         one rule per distinct utility you actually used
```

Because these are native cascade layers, **later layers beat earlier ones regardless of specificity**. That is why a single `.p-4` utility overrides your `.card { padding: 2rem }` component rule even though both have specificity (0,1,0). It is not a Tailwind trick — it is [`@layer`](../css/css-architecture.md) doing exactly what the spec says. [Part 9](./tailwindcss-reuse-architecture.md) covers the consequences.

`@layer properties` (the first line) is a separate, smaller thing: a holding pen for `@property` registrations and the `@supports` fallback block Tailwind emits for browsers that lack registered custom properties. You can ignore it; it is machinery, not surface.

### Preflight, and the border-colour surprise

`@layer base` holds **Preflight**, Tailwind's reset. It is more aggressive than `normalize.css` — it zeroes margins on everything, removes list styling, and makes headings inherit font size. Real excerpt from the build:

```css
@layer base {
  *, ::after, ::before, ::backdrop, ::file-selector-button {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
    border: 0 solid;
  }
}
```

Note `border: 0 solid` with **no colour**. That means border colour falls back to `currentColor`. In v3 it was `gray-200`. This is one of the silent v4 changes — a `border` class on a dark-text element now draws a near-black border where it used to draw light grey, and nothing errors. See [Part 12](./tailwindcss-engine-extensibility.md).

<a id="p1-advanced"></a>
## Advanced

### Import options

`@import "tailwindcss"` takes modifiers that change the whole build:

```css
@import "tailwindcss" source(none);      /* disable auto-detection; see Part 10 */
@import "tailwindcss" important;         /* mark every utility !important; see Part 9 */
@import "tailwindcss" prefix(tw);        /* namespace every class as tw:flex; see Part 9 */
```

These are documented in [Styling with utility classes](https://tailwindcss.com/docs/styling-with-utility-classes) and [Detecting classes](https://tailwindcss.com/docs/detecting-classes-in-source-files). All three exist for integrating into a codebase you don't fully control — reach for them deliberately, not by default.

### Importing your own CSS

Tailwind bundles `@import` itself, so you don't need `postcss-import`:

```css
@import "tailwindcss";
@import "./typography.css";
@import "./legacy-widgets.css";
```

Order matters in the ordinary CSS way for anything *not* in a layer. Anything you want to lose to utilities should be inside `@layer components`.

> **Outdated convention.** Tailwind v4 is explicitly **not designed to work with Sass, Less or Stylus** — the [Compatibility docs](https://tailwindcss.com/docs/compatibility) say "think of Tailwind as your preprocessor instead", and note that using both defeats the purpose. Everything preprocessors were for is now native or built in: `@import` bundling (Tailwind does it), variables (custom properties), nesting (Tailwind flattens it via Lightning CSS), and colour maths (`color-mix()`, and `--alpha()` — [Part 7](./tailwindcss-theming-design.md)). A great many blog posts still show a Sass + Tailwind setup. Don't.

### Wrong vs. right: the PostCSS-by-default reflex

```js
// WRONG — v3 muscle memory, in a Vite project
// postcss.config.js
module.exports = { plugins: { tailwindcss: {}, autoprefixer: {} } }
```

Three things are wrong here. `tailwindcss` is no longer itself a PostCSS plugin (that's `@tailwindcss/postcss`), `autoprefixer` is redundant because Tailwind runs Lightning CSS internally, and in a Vite project you should be using `@tailwindcss/vite` anyway.

```ts
// RIGHT
// vite.config.ts
import tailwindcss from '@tailwindcss/vite'
export default defineConfig({ plugins: [tailwindcss()] })
// ...and delete postcss.config.js entirely
```

<a id="p1-mastery"></a>
## Mastery

**Tier note:** Part 1's Mastery material is genuinely small, because the deep engine content — plain-text scanning, incremental rebuild strategy, the performance numbers — belongs to [Part 10](./tailwindcss-engine-extensibility.md), where it can be taught alongside `@source`. What follows is only the part that is about the *pipeline shape* rather than the scanner.

### Why the config moved into CSS

The v3 design had a JavaScript config that produced CSS. That meant two languages, a serialisation boundary between them, and a config file the browser could never see. v4 collapses this: your theme is *declared* in CSS as custom properties, and it *stays* custom properties all the way into the browser.

The payoff is not aesthetic. Because `--color-red-500` is a real custom property in `:root`, you can:

```css
/* use theme values in hand-written CSS, with no build-time function */
.chart-line { stroke: var(--color-blue-500); }
```

```html
<!-- compose theme values inside arbitrary values -->
<div class="rounded-[calc(var(--radius-xl)-1px)]">…</div>
```

```js
// read them at runtime
getComputedStyle(document.documentElement).getPropertyValue("--shadow-xl")
```

None of these were possible in v3, where `theme()` was a build-time text substitution with no runtime existence. This is also why `theme()` is now [marked deprecated](https://tailwindcss.com/docs/functions-and-directives) — it solves a problem that no longer exists. [Part 6](./tailwindcss-theming-design.md) covers the one place it survives.

### The pipeline has no runtime — and what that costs

Nothing of Tailwind ships to the browser. There is no JS payload, no hydration, no runtime style injection. Compared to runtime CSS-in-JS this is a large win.

The cost is symmetrical and worth naming honestly: **Tailwind cannot react to anything it can't see at build time.** A colour that comes from a database, a user-chosen theme, a value computed from props — none of these can produce a utility class, because the class must exist as literal text in a scanned file. The escape hatches are CSS custom properties (which *are* runtime values) and `@source inline()` safelisting. Both are in [Part 10](./tailwindcss-engine-extensibility.md), and the choice between them is a real architectural decision, not a workaround.

<a id="p1-cheat"></a>
## Part 1 cheat sheet

| Thing | Value |
|---|---|
| Config file | Your CSS file. `@import "tailwindcss";` |
| Layers declared | `theme, base, components, utilities` (+ `properties`) |
| Layer priority | Later beats earlier, **regardless of specificity** |
| Preflight lives in | `@layer base` |
| Default border colour | `currentColor` (was `gray-200` in v3) |
| Default `--spacing` | `0.25rem` |
| CLI build | `npx @tailwindcss/cli -i in.css -o out.css --watch` |
| Vite | `@tailwindcss/vite` → `plugins: [tailwindcss()]` |
| PostCSS | `@tailwindcss/postcss` (not `tailwindcss`) |
| webpack | `@tailwindcss/webpack` (v4.2+) |
| Import options | `source(none)`, `important`, `prefix(tw)` |
| Autoprefixer needed? | No — Lightning CSS is built in |
| Sass/Less/Stylus? | No — explicitly unsupported |

[↑ Back to top](#table-of-contents)

---

# Part 2 — Utility-First Fundamentals

<a id="p2-beginner"></a>
## Beginner

A utility class sets one thing. `p-4` sets padding. `flex` sets `display: flex`. `text-red-500` sets colour. You compose them in the `class` attribute:

```html
<button class="bg-blue-500 text-white px-4 py-2 rounded-md">Save</button>
```

The names are predictable once you see the pattern: **property abbreviation, then value**. `p` padding, `m` margin, `w` width, `h` height, `text` colour or size depending on the value, `bg` background. Directions suffix the property: `pt` padding-top, `px` padding-inline.

### The spacing scale

Numeric utilities are multiples of one variable, not arbitrary pixel counts:

```css
.p-4 { padding: calc(var(--spacing) * 4); }
```

`--spacing` defaults to `0.25rem`, so `p-4` is `1rem`. `p-2` is `0.5rem`. `p-8` is `2rem`.

This is the design-system constraint doing its job: you cannot accidentally use `13px` in one place and `14px` in another, because the vocabulary doesn't contain them. It also means changing `--spacing` rescales your entire UI proportionally — [Part 6](./tailwindcss-theming-design.md).

> **v4 change worth knowing:** in v3 the spacing scale was a fixed list, and `p-15` did not exist because nobody had defined it. In v4 numeric utilities are computed, so `p-15`, `grid-cols-15` and `mt-72` all work without configuration. The [v4.0 announcement](https://tailwindcss.com/blog/tailwindcss-v4) calls this "dynamic utility values". If you have ever added a number to a config file just to use it, that reflex is now obsolete.

<a id="p2-working"></a>
## Working Knowledge

### Arbitrary values — the escape hatch

When the design system genuinely doesn't have your value, square brackets generate it on the spot:

```html
<div class="top-[117px]">…</div>
<div class="bg-[#316ff6]">Sign in with Facebook</div>
<div class="grid grid-cols-[24rem_2.5rem_minmax(0,1fr)]">…</div>
```

These are ordinary utilities in every other respect — they take variants:

```html
<div class="top-[117px] lg:top-[344px]">…</div>
```

**The underscore rule.** Class names can't contain spaces, so Tailwind uses `_` and converts it back at build time:

```html
<div class="grid-cols-[1fr_500px_2fr]">  <!-- becomes: 1fr 500px 2fr -->
```

If you need a *literal* underscore — most often in a `content-['…']` string or a font name — escape it with a backslash: `content-['hello\_world']`. This is a real trap with fonts like `Open_Sans`, where the underscore is part of the name.

### Arbitrary properties

Brackets with a `property:value` pair inside generate CSS for a property Tailwind has no utility for at all:

```html
<div class="[mask-type:luminance]">…</div>
<div class="hover:[mask-type:luminance]">…</div>
```

This is the "Tailwind doesn't support X" answer for essentially all X. Combined with `@utility` ([Part 11](./tailwindcss-engine-extensibility.md)) for anything you use more than twice, you never actually need to drop out of the system.

### CSS variables in utilities — the syntax that changed

```html
<button class="bg-(--bg-color) text-(--text-color) hover:bg-(--bg-color-hover)">
```

> **Outdated convention.** In v3 this was `bg-[--bg-color]` — square brackets. In v4 it is **parentheses**: `bg-(--bg-color)`. The [upgrade guide](https://tailwindcss.com/docs/upgrade-guide) lists this as a breaking change. The old form doesn't error; it is interpreted as an arbitrary *value* of the literal text `--bg-color`, which produces invalid CSS and silently no styling. This one is genuinely hard to spot in review.

You can also *set* a variable with an arbitrary property, which makes responsive variables possible:

```html
<div class="[--gutter-width:1rem] lg:[--gutter-width:2rem]">…</div>
```

That is a legitimately powerful pattern — a value that changes at a breakpoint and is then consumed by ordinary CSS deeper in the tree.

### The `!` important modifier

```html
<div class="bg-teal-500 bg-red-500!">  <!-- red wins -->
```

> **Outdated convention.** v3 put the `!` in front (`!bg-red-500`); v4 puts it at the **end**. Per the [upgrade guide](https://tailwindcss.com/docs/upgrade-guide). The leading form silently generates nothing.

Verified output:

```css
.bg-red-500\! {
  background-color: var(--color-red-500) !important;
}
```

Use it for fighting third-party CSS you can't edit. Inside your own code it's a symptom — you almost certainly have a layer problem ([Part 9](./tailwindcss-reuse-architecture.md)).

<a id="p2-advanced"></a>
## Advanced

### Conflicting classes: "the last one wins" is false

This is the most commonly-believed wrong thing about Tailwind, and it causes real bugs.

```html
<div class="p-4 p-2">  <!-- which padding? -->
```

The intuition is "`p-2`, it's last in the attribute". **Wrong.** The `class` attribute is an unordered set as far as CSS is concerned; the browser resolves the conflict using the *stylesheet*, and in the stylesheet both rules have identical specificity, so **source order in `output.css` decides**.

Here is the actual generated output for exactly that markup:

```css
  .p-2 {
    padding: calc(var(--spacing) * 2);
  }
  .p-4 {
    padding: calc(var(--spacing) * 4);
  }
```

`.p-4` comes second, so **`p-4` wins** — the opposite of the intuition. And note that Tailwind emitted them in *its own* canonical order, not the order they appeared in my HTML. Reversing the attribute to `class="p-2 p-4"` produces byte-identical CSS and therefore the identical result.

The consequence is important: **you cannot override a Tailwind utility with another Tailwind utility of the same property by ordering them in the class attribute.** This is why `tailwind-merge` exists ([Part 8](./tailwindcss-reuse-architecture.md)) — when a component takes a `className` prop meant to override its defaults, string concatenation does not work and you need to resolve the conflict in JavaScript before it reaches the DOM.

The [docs' own advice](https://tailwindcss.com/docs/styling-with-utility-classes) is to not create the conflict in the first place:

```jsx
// WRONG — hoping the second class wins
<div className={`grid ${isFlex ? "flex" : ""}`}>

// RIGHT — only one class exists
<div className={isFlex ? "flex" : "grid"}>
```

### Wrong vs. right: reaching for arbitrary values too early

```html
<!-- WRONG: rebuilding the design system by hand, one bracket at a time -->
<div class="p-[16px] text-[14px] rounded-[6px] text-[#3b82f6]">
```

Every one of those has a theme equivalent. Written this way they bypass the design system entirely: change the brand blue later and this element doesn't move.

```html
<!-- RIGHT -->
<div class="p-4 text-sm rounded-md text-blue-500">
```

Arbitrary values are for genuine one-offs — a third-party widget that needs exactly `117px`, a brand hex not in your palette *yet*. If you write the same arbitrary value three times, it belongs in `@theme` ([Part 6](./tailwindcss-theming-design.md)).

### Try It: prove the conflict rule to yourself

In the playground:

```html
<div class="p-4 p-2 bg-blue-200">A</div>
<div class="p-2 p-4 bg-blue-200">B</div>
```

**Expected result:** both boxes have *identical* padding — `1rem`, the `p-4` value. If you believed "last wins", you'd expect A to be tighter than B. Confirm in DevTools: inspect either box, and the Styles pane shows `.p-2` struck through with `.p-4` winning.

Now search `output.css` for `padding: calc(var(--spacing) * 2)` and confirm it appears **above** the `* 4` rule. That ordering is the entire explanation.

<a id="p2-mastery"></a>
## Mastery

### The design-system argument, honestly

The [docs' case](https://tailwindcss.com/docs/styling-with-utility-classes) for utilities over inline styles is three points: design constraints (values come from a scale, not from nowhere), state variants (you cannot write `:hover` in a `style` attribute), and media queries (same). All three are true and all three are real.

The case worth adding, which the docs undersell: **utilities make deletion safe**. The reason CSS codebases rot is that nobody can prove a rule is unused, so nothing is ever removed and the stylesheet only grows. When styles live in the markup, deleting the component deletes its styles, atomically, with no orphan left behind. The `output.css` round-trip in [Part 1's Try It](#p1-beginner) is that property made visible.

The honest cost, which utility advocates undersell: **markup becomes dense and diffs get noisy**. A `class` attribute with eighteen utilities is genuinely harder to scan than `class="card"`, and a one-utility change produces a diff line that looks like a rewrite. This is a real trade and [Part 8](./tailwindcss-reuse-architecture.md) treats the mitigation strategies as a genuine design question rather than assuming the answer.

### Why the bundle stops growing

Because one rule is emitted per *distinct* utility, CSS size is a function of the vocabulary you use, not the code you write. The first screen of an app might introduce 200 utilities. The fiftieth screen introduces perhaps three, because it reuses the same spacing scale, the same palette, the same type ramp as the other forty-nine.

This asymptote is the strongest structural argument for utility-first CSS, and it's why the approach gets *more* attractive as a codebase grows — the opposite of hand-written component CSS, which grows at least linearly with screens and often super-linearly once overrides start accumulating.

<a id="p2-cheat"></a>
## Part 2 cheat sheet

| Syntax | Means | Note |
|---|---|---|
| `p-4` | Theme scale value | `calc(var(--spacing) * 4)` = 1rem |
| `p-15` | Any number works | v4 computes it; no config needed |
| `p-[13px]` | Arbitrary value | For genuine one-offs only |
| `bg-[#316ff6]` | Arbitrary colour | |
| `grid-cols-[1fr_2fr]` | Underscore = space | Escape a literal one as `\_` |
| `[mask-type:luminance]` | Arbitrary property | Covers anything with no utility |
| `bg-(--my-var)` | CSS variable | **v4.** v3 was `bg-[--my-var]` |
| `[--gutter:1rem]` | Set a variable | Combines with variants |
| `bg-red-500!` | Important | **v4.** v3 was `!bg-red-500` |
| `bg-red-500/75` | Opacity modifier | [Part 7](./tailwindcss-theming-design.md) |

| Question | Answer |
|---|---|
| `class="p-4 p-2"` — which wins? | `p-4`. Source order in `output.css`, not attribute order |
| Can I override a utility by putting mine last? | No. Use `tailwind-merge` ([Part 8](./tailwindcss-reuse-architecture.md)) |
| How do I use a value not in the theme? | `[brackets]` once; `@theme` if repeated |
| Where does `--spacing` come from? | `@layer theme`; default `0.25rem` |

[↑ Back to top](#table-of-contents)

---

# Part 3 — Variants

<a id="p3-beginner"></a>
## Beginner

A variant is a prefix ending in `:` that makes a utility conditional.

```html
<button class="bg-sky-500 hover:bg-sky-700">Save</button>
```

`bg-sky-500` always applies. `hover:bg-sky-700` applies only on hover. The generated CSS is exactly what you'd write by hand:

```css
.hover\:bg-sky-700:hover { background-color: var(--color-sky-700); }
```

The backslashes are just CSS escaping for the `:` in the class name. They are not meaningful.

**Every utility takes every variant.** That is the whole design — there is no list of "which utilities support hover", because the variant system wraps rules generically. `hover:`, `md:`, `dark:`, `focus:`, `first:`, `disabled:` all compose with anything.

<a id="p3-working"></a>
## Working Knowledge

### The variant families

From the [states documentation](https://tailwindcss.com/docs/hover-focus-and-other-states):

| Family | Examples |
|---|---|
| **Pseudo-classes** | `hover:` `focus:` `active:` `visited:` `target:` `first:` `last:` `only:` `odd:` `even:` `disabled:` `enabled:` `checked:` `indeterminate:` `required:` `valid:` `invalid:` `read-only:` `empty:` `focus-within:` `focus-visible:` |
| **Pseudo-elements** | `before:` `after:` `placeholder:` `file:` `marker:` `selection:` `first-line:` `first-letter:` `backdrop:` |
| **Parent state** | `group-hover:` `group-focus:` … (requires `group` on the ancestor) |
| **Sibling state** | `peer-checked:` `peer-invalid:` … (requires `peer` on a *previous* sibling) |
| **Attributes** | `data-*:` `aria-*:` |
| **Structural** | `has-*:` `not-*:` `in-*:` `nth-*:` |
| **Media/feature** | `sm:` `md:` `dark:` `motion-reduce:` `contrast-more:` `forced-colors:` `print:` `portrait:` `pointer-coarse:` `supports-[…]:` `noscript:` |
| **State** | `open:` `inert:` `starting:` |
| **Arbitrary** | `[&_p]:` `[@supports(display:grid)]:` |

### `group-*` — styling a child from a parent's state

Mark the ancestor `group`, then any descendant can react to it:

```html
<a href="#" class="group block rounded-lg p-4 hover:bg-sky-500">
  <h3 class="text-gray-900 group-hover:text-white">New project</h3>
  <p class="text-gray-500 group-hover:text-white">Create a new project</p>
</a>
```

Hovering the link changes both text colours. Without `group-*` this needs JavaScript or a hand-written `.parent:hover .child` rule.

**Named groups** disambiguate nesting — essential in lists:

```html
<li class="group/item flex justify-between p-4 hover:bg-slate-100">
  <span>Contact</span>
  <a href="#" class="group/edit invisible group-hover/item:visible">
    <span class="group-hover/edit:text-gray-700">Call</span>
  </a>
</li>
```

`group-hover/item:` reacts to the `li`; `group-hover/edit:` reacts to the `a`. Without the names, the inner `group` would shadow the outer and the edit link would never appear.

### `peer-*` — styling from a sibling's state

Same idea, sideways. The classic use is form validation with no JavaScript:

```html
<label class="block">
  <span>Email</span>
  <input type="email" required class="peer border p-2" />
  <p class="invisible peer-invalid:visible text-red-600 text-sm">
    Please provide a valid email address.
  </p>
</label>
```

**The one hard constraint:** `peer` must be on a *previous* sibling of the element being styled. CSS's sibling combinators only look backwards, so a peer that comes after cannot be targeted. If your markup has the message before the input, you must either reorder the DOM or reach for `has-*`.

### `data-*` and `aria-*` — the ones that scale

For component state, attribute variants beat class-toggling because the state lives in one place and is accessible for free:

```html
<div data-active class="border-gray-300 data-active:border-purple-500">…</div>
<div data-size="large" class="data-[size=large]:p-8">…</div>
<div aria-checked="true" class="bg-gray-600 aria-checked:bg-sky-700">…</div>
<th aria-sort="ascending" class="aria-[sort=ascending]:bg-blue-50">Invoice #</th>
```

Bare `data-active:` tests attribute *presence*; `data-[size=large]:` tests a value. The `aria-*` variants have built-in shorthands for the boolean ARIA attributes (`aria-busy`, `aria-checked`, `aria-disabled`, `aria-expanded`, `aria-hidden`, `aria-pressed`, `aria-readonly`, `aria-required`, `aria-selected`); anything else uses the bracket form.

This is the pattern to prefer for design systems. Driving styles off `aria-expanded` means your accessibility attribute and your visual state cannot drift apart, because they are the same fact.

### `has-*` — styling a parent from its children

```html
<label class="rounded-lg border p-4 has-checked:bg-indigo-50 has-checked:text-indigo-900">
  <input type="radio" name="payment" />
  Google Pay
</label>
```

The label restyles itself when the radio inside it is checked. This is the CSS `:has()` selector ([see the css/ set](../css/css-foundations.md)), and it removes an enormous amount of the JavaScript that used to exist purely to add a class to a parent.

Compose it: `group-has-[a]:block`, `peer-has-checked:hidden`.

<a id="p3-advanced"></a>
## Advanced

### Stacking order — read left to right, outside in

This is the part people get wrong, and it's worth doing precisely. Given:

```html
<button class="dark:md:hover:bg-red-500">Save</button>
```

Here is the **actual generated output**, verified against v4.3.3:

```css
@media (prefers-color-scheme: dark) {
  @media (width >= 48rem) {
    @media (hover: hover) {
      .dark\:md\:hover\:bg-red-500:hover {
        background-color: var(--color-red-500);
      }
    }
  }
}
```

Read the class left to right and you get the nesting from outside in: `dark` is the outermost wrapper, then `md`, then `hover`, then the rule. The rule fires only when **all three** hold.

Two things fall out of this that matter:

1. **For conditions that must all be true simultaneously, order is semantically irrelevant** — `dark:md:hover:` and `hover:md:dark:` both mean "dark AND md AND hover". They generate different (equivalent) nesting, and produce two *different class names*, but the same behaviour.
2. **For variants that transform the selector rather than wrap it, order absolutely matters.** `[&_p]:hover:underline` and `hover:[&_p]:underline` are different selectors: the first is "a `p` descendant, when hovered", the second is "when the element is hovered, its `p` descendants". Compound selector variants (`group-*`, `peer-*`, arbitrary `[&…]`) are where ordering bugs actually live.

### The `@media (hover: hover)` wrapper

Look again at that output: `hover:` generated an extra `@media (hover: hover)` wrapper that you did not ask for.

> **Silent v4 behaviour change.** In v3, `hover:` compiled to a bare `:hover`. In v4 it is gated behind `@media (hover: hover)`, so it does **not** fire on touch devices ([upgrade guide](https://tailwindcss.com/docs/upgrade-guide)). This is usually what you want — it fixes the classic "hover style sticks after tapping on mobile" bug. But if your design *relied* on tap-triggering a hover style, it silently stopped working on phones after upgrading, and nothing in your build will tell you.

The opt-out is one line:

```css
@custom-variant hover (&:hover);
```

### `not-*` and `in-*` — the two newer ones worth learning

`not-*` negates any other variant:

```html
<button class="bg-indigo-600 hover:not-focus:bg-indigo-700">…</button>
<div class="not-supports-[display:grid]:flex">…</div>
```

`in-*` is `group-*` without needing to mark the parent:

```html
<div tabindex="0">
  <div class="opacity-50 in-focus:opacity-100">…</div>
</div>
```

The difference from `group-focus:` is that `in-*` matches *any* ancestor, so you don't have to own the parent's markup. That makes it the right tool when the ancestor is rendered by a third-party component you can't add a class to. The tradeoff is precision: `group-*` targets exactly the element you marked, `in-*` targets whichever ancestor happens to match first.

### Arbitrary variants

When no variant exists, write the selector directly with `&` as the placeholder:

```html
<li class="[&.is-dragging]:cursor-grabbing">…</li>
<div class="[&_p]:mt-4">…</div>          <!-- every p descendant -->
<div class="[&>*]:border-b">…</div>       <!-- direct children -->
<div class="flex [@supports(display:grid)]:grid">…</div>
<li class="lg:[&:nth-child(-n+3)]:hover:underline">…</li>
```

`[&_p]:mt-4` is the one you'll actually reach for constantly — it's how you style markup you don't control, like the output of a Markdown renderer.

**Advice:** if you write the same arbitrary variant twice, promote it to `@custom-variant` ([Part 11](./tailwindcss-engine-extensibility.md)). They're unreadable in bulk.

### Try It: see the nesting for yourself

Paste into the playground:

```html
<button class="p-4 bg-slate-200 dark:md:hover:bg-red-500">Hover me</button>
```

Now open `output.css` and find the rule. **Expected:** three nested `@media` blocks in the order `prefers-color-scheme: dark` → `width >= 48rem` → `hover: hover`, exactly as shown above.

Then test the behaviour. In Chrome DevTools open the command menu (`Ctrl/Cmd+Shift+P`) → "Show Rendering" → set **Emulate prefers-color-scheme** to `dark`. Widen the window past 768px and hover. **Expected:** red. Now narrow the window below 768px and hover. **Expected:** no red — the `md` condition failed. This is the "all conditions must hold" rule made visible.

<a id="p3-mastery"></a>
## Mastery

### Why variants aren't a fixed list

A variant is a *selector transformation*, not an entry in a lookup table. That's why `data-[size=large]:`, `[&_p]:`, `group-hover/item:` and `supports-[display:grid]:` can all exist without Tailwind enumerating the possibilities — the engine parses the variant into a transformation and applies it to whatever rule the utility produced.

This is also why variants compose arbitrarily deep. `dark:lg:group-has-[a]:hover:not-focus:underline` is a legal class. It is also a terrible class, and if you find yourself writing it, the answer is `@custom-variant`.

### Variant order and the emitted stylesheet

Variants don't only nest — they also affect *where* a rule lands in the output, which matters for the conflict rule from [Part 2](#p2-advanced). Tailwind sorts generated rules so that more-specific-in-intent variants land later. This is why `md:p-8` reliably beats `p-4` without either being `!important`: the responsive rule is emitted after the base rule, and inside a matching media query it therefore wins on source order.

The practical consequence: **you do not need to think about variant precedence.** Mobile-first ordering, dark-mode overriding light, `hover` overriding rest state — these all work because the sort is deterministic and matches intent. It is one of the places where the framework's opinionated ordering earns its keep. The place it *stops* working is same-variant conflicts (`md:p-4 md:p-8`), which behave exactly like the Part 2 case and need the same solution.

<a id="p3-cheat"></a>
## Part 3 cheat sheet

| Variant | Applies when | Note |
|---|---|---|
| `hover:` | Pointer hover | Wrapped in `@media (hover: hover)` in v4 |
| `focus:` `focus-visible:` `focus-within:` | Focus states | |
| `first:` `last:` `odd:` `even:` `only:` `empty:` | Structural position | |
| `disabled:` `checked:` `invalid:` `required:` `read-only:` | Form state | |
| `before:` `after:` | Pseudo-elements | Needs `content-['']` unless set |
| `placeholder:` `selection:` `marker:` `file:` `backdrop:` | Other pseudo-elements | |
| `group-hover:` | Ancestor with `group` is hovered | |
| `group-hover/name:` | *Named* ancestor is hovered | Use in nested lists |
| `peer-invalid:` | **Previous sibling** with `peer` is invalid | Backwards only |
| `data-active:` | `data-active` attribute present | |
| `data-[size=large]:` | Attribute equals a value | |
| `aria-checked:` | Boolean ARIA attribute | Built-ins for the common 9 |
| `aria-[sort=ascending]:` | Any ARIA value | |
| `has-checked:` | A descendant is checked | `:has()` |
| `group-has-[a]:` `peer-has-checked:` | Composed `:has()` | |
| `not-hover:` | Negation of any variant | |
| `in-focus:` | *Any* ancestor is focused | No `group` class needed |
| `nth-3:` `nth-[2n+1_of_li]:` | `:nth-child()` | Also `nth-last-*`, `nth-of-type-*` |
| `sm:` … `2xl:` | Min-width breakpoints | [Part 5](./tailwindcss-layout-responsive.md) |
| `max-md:` | Max-width breakpoint | Stack for ranges: `md:max-lg:` |
| `@md:` | Container query | [Part 5](./tailwindcss-layout-responsive.md) |
| `dark:` | `prefers-color-scheme: dark` | Retarget via `@custom-variant` — [Part 6](./tailwindcss-theming-design.md) |
| `motion-reduce:` `contrast-more:` `forced-colors:` | Accessibility media queries | |
| `print:` `portrait:` `landscape:` `noscript:` | Other media | |
| `supports-[display:grid]:` | `@supports` | |
| `open:` `inert:` `starting:` | Element state / `@starting-style` | |
| `[&_p]:` | Arbitrary — every `p` descendant | |
| `[&>*]:` | Arbitrary — direct children | |
| `[&.is-x]:` | Arbitrary — with another class | |

**The three rules that matter**

1. Stacking reads **left to right, outside in**. All conditions must hold.
2. `peer` must precede its target in the DOM. `group` must be an ancestor. `in-*` needs neither.
3. If you write the same arbitrary variant twice, make it a `@custom-variant`.

[↑ Back to top](#table-of-contents)

---

*Parts 1–3 of the [Tailwind CSS set](./tailwindcss-mastery-guide.md). Next: [Layout & Responsive (Parts 4–5)](./tailwindcss-layout-responsive.md).*
