# Tailwind CSS — Engine & Extensibility (Parts 10–12)

How the generator actually finds your classes, how to add your own vocabulary properly, and how to get out of v3.

> **Spec:** this doc follows the shared spec in [`tailwindcss-mastery-guide.md`](./tailwindcss-mastery-guide.md#about-this-document) — curiosity-driven, tiered Beginner → Mastery, Tailwind **v4.3.3**, assuming the [`css/`](../css/css-mastery-guide.md) set.

---

## Table of Contents

- [Part 10 — The Engine & Content Detection](#part-10--the-engine--content-detection)
  - [Beginner](#p10-beginner) · [Working Knowledge](#p10-working) · [Advanced](#p10-advanced) · [Mastery](#p10-mastery) · [Cheat sheet](#p10-cheat)
- [Part 11 — Extending Tailwind](#part-11--extending-tailwind)
  - [Beginner](#p11-beginner) · [Working Knowledge](#p11-working) · [Advanced](#p11-advanced) · [Mastery](#p11-mastery) · [Cheat sheet](#p11-cheat)
- [Part 12 — Migrating from v3](#part-12--migrating-from-v3)
  - [Working Knowledge](#p12-working) · [Advanced](#p12-advanced) · [Cheat sheet](#p12-cheat)

---

# Part 10 — The Engine & Content Detection

<a id="p10-beginner"></a>
## Beginner

Tailwind finds your classes by **reading your files as plain text**.

Not parsing. Not compiling. Not evaluating. It scans for character sequences that look like they could be class names, and checks each against its utility definitions. From the [content-detection docs](https://tailwindcss.com/docs/detecting-classes-in-source-files):

> Tailwind scans your project files as plain text (not parsed code) and looks for tokens matching Tailwind's class name character patterns.

By default it scans **everything except**:

- anything in `.gitignore`
- `node_modules`
- binary files
- CSS files
- package-manager lock files

That's it. No `content` array, no globs to maintain.

> **Outdated convention.** v3 required `content: ["./src/**/*.{html,js}"]` in the JS config. v4 removes it. If you migrate and leave the config in place, it's silently ignored — but auto-detection usually covers the same files anyway, so this one tends to work by accident.

<a id="p10-working"></a>
## Working Knowledge

### The dynamic class-name rule

This is **the single most common Tailwind bug**, and it follows directly from plain-text scanning.

```jsx
// WRONG — Tailwind sees "bg-", not "bg-blue-600"
function Button({ color }) {
  return <button className={`bg-${color}-600`}>Click</button>;
}
```

```html
<!-- WRONG — the template expression is opaque to the scanner -->
<div class="text-{{ error ? 'red' : 'green' }}-600"></div>
```

The string `bg-blue-600` never appears in any file. No rule is generated. The element renders unstyled and nothing errors — not at build time, not at runtime, not in the console.

The fix is always the same shape: **make the complete class name appear literally somewhere.**

```jsx
// RIGHT — full class names in a lookup
const colorVariants = {
  blue: "bg-blue-600 hover:bg-blue-500",
  red: "bg-red-600 hover:bg-red-500",
};

function Button({ color }) {
  return <button className={colorVariants[color]}>Click</button>;
}
```

```html
<!-- RIGHT -->
<div class="{{ error ? 'text-red-600' : 'text-green-600' }}"></div>
```

Note the second form: the *interpolation* is fine, it's the *fragment concatenation* that breaks. `text-red-600` appears literally in the file, so it's found.

**The debugging reflex:** when a class does nothing, search `output.css` for it before anything else. Missing rule → content detection. Present rule → cascade ([Part 9](./tailwindcss-reuse-architecture.md)). These two bugs are indistinguishable in the browser and have completely different fixes.

### `@source` — adding scan paths

The main use is packages inside `node_modules`, which is skipped by default:

```css
@import "tailwindcss";
@source "../node_modules/@acmecorp/ui-lib";
```

Also anything `.gitignore`d that you nonetheless need scanned — generated templates, build output that's re-scanned.

### `@source not` — excluding

```css
@import "tailwindcss";
@source not "../src/components/legacy";
```

For large directories you know don't use Tailwind. This is a build-time optimisation; on a big repo it's a real one.

### `@source inline()` — safelisting

When a class genuinely cannot appear literally — it comes from a database, a CMS, or user configuration — force generation:

```css
@source inline("underline");
@source inline("{hover:,focus:,}underline");
@source inline("bg-red-{50,{100..900..100},950}");
```

The brace syntax is [shell brace expansion](https://www.gnu.org/software/bash/manual/html_node/Brace-Expansion.html). That third line expands to `bg-red-50`, `bg-red-100` … `bg-red-900`, `bg-red-950` — **11 classes from one line**, verified in the build output. The `{100..900..100}` form is `start..end..step`.

The second line shows variants: `{hover:,focus:,}` includes an empty alternative, so you get `underline`, `hover:underline` and `focus:underline`.

And the inverse, to *prevent* generation:

```css
@source not inline("{hover:,focus:,}bg-red-{50,{100..900..100},950}");
```

> **Outdated convention.** v3's `safelist` array in the JS config is **not supported in v4** at all ([upgrade guide](https://tailwindcss.com/docs/upgrade-guide) lists `corePlugins`, `safelist` and `separator` as unsupported). `@source inline()` replaces it.

### `source(none)` — explicit-only

```css
@import "tailwindcss" source(none);
@source "../admin";
@source "../shared";
```

Nothing automatic; you list everything. Correct for monorepos with multiple Tailwind entry points that each need a different scope — see the [Part 9 monorepo scenario](./tailwindcss-reuse-architecture.md).

<a id="p10-advanced"></a>
## Advanced

### Try It: watch the scanner fail, then fix it

In the playground, add to `index.html`:

```html
<script>
  const shade = 600;
  document.write(`<div class="bg-blue-${shade} p-8">Dynamic</div>`);
</script>
<div class="bg-blue-600 p-8">Static</div>
```

**Expected:** both divs render, but only the second is blue. Search `output.css` for `bg-blue-600` — it's there **exactly once**, generated because of the static div.

Now delete the static div and rebuild. **Expected:** `bg-blue-600` is gone from `output.css` entirely, and the dynamic div loses its background. That's the bug, isolated.

Two ways to fix it. Safelist:

```css
@source inline("bg-blue-{100..900..100}");
```

Or — usually better — use a custom property, which is a *runtime* value the scanner never needs to see:

```html
<div class="bg-(--dynamic-bg) p-8" style="--dynamic-bg: var(--color-blue-600)">Dynamic</div>
```

**Expected after either fix:** both divs blue. The second approach generates one rule regardless of how many colours you use, where the safelist generates one per colour. That's the real tradeoff between them, and on a large palette it's a significant bundle difference.

### Real Scenario — the CSS bundle that grew 40×

A team stores brand colours per tenant in a database and renders `class="bg-{{ tenant.color }}-500"`. It doesn't work (of course), so someone adds a safelist covering every colour × every shade × the variants used. The build now emits several thousand rules, almost all unused by any given tenant, and the stylesheet balloons.

The safelist was the wrong tool. Tenant colour is *runtime data*, so it belongs in a custom property:

```html
<body style="--brand: {{ tenant.color_hex }}">
  <button class="bg-(--brand) text-white">…</button>
</body>
```

One generated rule, any number of tenants, and the value can change without a rebuild. **The heuristic: safelist a small, closed set that's known at build time; use custom properties for anything open-ended or genuinely runtime.**

### Build performance

From the [v4.0 announcement](https://tailwindcss.com/blog/tailwindcss-v4):

| Build | v3.4 | v4.0 | Speedup |
|---|---|---|---|
| Full | 378ms | 100ms | 3.78× |
| Incremental, new CSS | 44ms | 5ms | 8.8× |
| Incremental, no new CSS | 35ms | 192µs | **182×** |

That last row is the one that changes how the tool feels. When you edit a file and introduce no new utilities — the overwhelmingly common case — the rebuild is microseconds, so the compile step effectively disappears from your feedback loop.

The [v4.3 post](https://tailwindcss.com/blog/tailwindcss-v4-3) reports a further 3.8× recompilation improvement in v4.2, and that the v4.2 webpack plugin is 2.17× faster than routing webpack through PostCSS.

My own playground build, for calibration: **~43ms** for a full build of a small project on v4.3.3.

If your builds are slow, the causes in order of likelihood: scanning too much (`@source not`, or `source(none)` plus explicit paths), CSS modules (Tailwind runs per module — [Part 8](./tailwindcss-reuse-architecture.md)), or using PostCSS where a dedicated plugin exists.

<a id="p10-mastery"></a>
## Mastery

### Why plain-text scanning, and not a real parser?

It looks like a limitation. It's a deliberate trade, and understanding it explains a lot of the framework's shape.

A parser-based scanner would need to understand every language in your project — JSX, Vue SFCs, Svelte, Astro, ERB, Blade, Twig, Go templates, Python f-strings, and whatever appears next year. Each needs a parser, each parser needs maintaining, each has version skew, and each is a source of "Tailwind doesn't work with X".

Plain-text scanning works with **every language that has ever existed and every one that will**, including ones nobody anticipated. A Tailwind class in a Rust string literal, a database seed file, or a YAML config is found with zero integration work. That universality is why Tailwind spread across ecosystems the way it did.

And crucially, a parser wouldn't actually solve the dynamic-class problem. `bg-${color}-600` is only resolvable by *executing* the program with real data — that's not parsing, that's running your app. No scanner can know that `color` is `"blue"` on Tuesday and `"red"` on Wednesday. The limitation is fundamental to build-time CSS generation, not an artefact of the implementation, and the correct answer is the one the framework gives you: custom properties for runtime values.

### The consequences worth internalising

1. **Any file can contribute classes.** A stray `bg-red-500` in a README or a comment generates a rule. Harmless, occasionally surprising.
2. **Deleting code deletes CSS**, automatically and exactly. This is the property that makes utility-first CSS not rot.
3. **The scanner is the boundary of the system.** Everything Tailwind can do, it does because it saw literal text. Every escape hatch (`@source inline`, custom properties) exists to bridge that boundary.

<a id="p10-cheat"></a>
## Part 10 cheat sheet

| Task | Syntax |
|---|---|
| Scan a package in `node_modules` | `@source "../node_modules/@acme/ui";` |
| Skip a directory | `@source not "../src/legacy";` |
| Safelist a class | `@source inline("underline");` |
| Safelist with variants | `@source inline("{hover:,focus:,}underline");` |
| Safelist a range | `@source inline("bg-red-{50,{100..900..100},950}");` |
| Prevent generation | `@source not inline("…");` |
| Disable auto-detection | `@import "tailwindcss" source(none);` |

| Symptom | Cause |
|---|---|
| Class does nothing, **no rule** in `output.css` | Content detection |
| Class does nothing, **rule exists** | Cascade — [Part 9](./tailwindcss-reuse-architecture.md) |
| Works in dev, unstyled in another app | Monorepo: missing `@source` |
| Bundle is enormous | Over-broad safelist — use custom properties |

**The rule:** `bg-${color}-600` never works. Full class names in a lookup object, or a CSS variable.

[↑ Back to top](#table-of-contents)

---

# Part 11 — Extending Tailwind

<a id="p11-beginner"></a>
## Beginner

`@theme` adds *values*. `@utility` adds *utilities* — new class names with CSS Tailwind has no equivalent for.

```css
@utility content-auto {
  content-visibility: auto;
}
```

```html
<div class="content-auto hover:content-auto md:content-auto">…</div>
```

**Why not just write `.content-auto { … }` in `@layer components`?** Because a plain class doesn't work with variants. `@utility` registers it with the engine, so `hover:`, `md:`, `dark:` and everything else compose with it automatically, and it lands in the `utilities` layer with the right precedence.

Multi-rule utilities use nesting:

```css
@utility scrollbar-hidden {
  &::-webkit-scrollbar { display: none; }
}
```

Verified output: `.scrollbar-hidden::-webkit-scrollbar { display: none; }`.

<a id="p11-working"></a>
## Working Knowledge

### Functional utilities

A trailing `-*` makes the utility take an argument, resolved by `--value()`:

```css
@theme {
  --tab-size-github: 8;
}

@utility tabx-* {
  tab-size: --value(--tab-size-*, integer, [integer], --default(4));
}
```

That one line accepts **four kinds of input**. Verified output:

```css
.tabx        { tab-size: 4; }                        /* --default(4)  */
.tabx-2      { tab-size: 2; }                        /* integer       */
.tabx-\[7\]  { tab-size: 7; }                        /* [integer]     */
.tabx-github { tab-size: var(--tab-size-github); }   /* --tab-size-*  */
```

The four resolution modes, from the [custom styles docs](https://tailwindcss.com/docs/adding-custom-styles):

| Form | Matches | Example |
|---|---|---|
| `--value(--tab-size-*)` | A theme namespace | `tabx-github` |
| `--value(integer)` | A bare value of that type | `tabx-2`, `tabx-76` |
| `--value([integer])` | An arbitrary value | `tabx-[7]` |
| `--value("inherit", "initial")` | Literal keywords | `tabx-inherit` |
| `--default(4)` | Value when the utility is used bare | `tabx` |

> **`--default()` is new in v4.3** ([release post](https://tailwindcss.com/blog/tailwindcss-v4-3)). Before it, a functional utility always required a value; the bare form had to be defined as a separate `@utility`. If you're on 4.0 or 4.1, that's the workaround.

### Modifiers

`--modifier()` handles the `/` suffix, the way `text-lg/7` sets size and leading together:

```css
@utility text-* {
  font-size: --value(--text-*, [length]);
  line-height: --modifier(--leading-*, [length]);
}
```

### Custom variants

`@custom-variant` registers a new variant. Shorthand form:

```css
@custom-variant theme-midnight (&:where([data-theme="midnight"] *));
```

```html
<html data-theme="midnight">
  <button class="theme-midnight:bg-black">…</button>
</html>
```

Verified: `.theme-midnight\:bg-black:where([data-theme="midnight"] *) { … }`.

The block form uses `@slot` to mark where the styles go — necessary when the variant needs more than a selector:

```css
@custom-variant theme-midnight {
  &:where([data-theme="midnight"] *) {
    @slot;
  }
}
```

**The `:where()` habit is worth copying.** It keeps specificity at zero so your variant doesn't accidentally outrank other utilities. Every variant Tailwind ships does this.

More examples from the docs:

```css
@custom-variant aria-asc (&[aria-sort="ascending"]);
@custom-variant data-checked (&[data-ui~="checked"]);
@custom-variant hover (&:hover);        /* opt out of the (hover: hover) gate */
```

### `@variant` — using variants inside custom CSS

The inverse direction: apply a Tailwind variant to hand-written CSS.

```css
.my-element {
  background: white;

  @variant dark {
    background: black;
  }
}
```

**New in v4.3** — stacked and compound forms:

```css
.my-element {
  @variant hover:focus { background: black; }   /* stacked: hover AND focus */
  @variant hover, focus { background: black; }  /* compound: hover OR focus */
}
```

Before v4.3 both required nesting `@variant` blocks by hand.

<a id="p11-advanced"></a>
## Advanced

### Wrong vs. right: adding a utility

```css
/* WRONG — no variant support, wrong layer, wrong precedence */
.content-auto { content-visibility: auto; }
```

`hover:content-auto` generates nothing. And being unlayered, it beats every real utility ([Part 9](./tailwindcss-reuse-architecture.md)).

```css
/* RIGHT */
@utility content-auto {
  content-visibility: auto;
}
```

### When to use which extension point

| You want | Use |
|---|---|
| A new *value* in an existing scale (a colour, a radius) | `@theme` |
| A new *utility* for a CSS property with no Tailwind equivalent | `@utility` |
| A new *conditional* (a theme attribute, a library's state class) | `@custom-variant` |
| A variant applied to hand-written CSS | `@variant` |
| A one-off, used once | Arbitrary property: `[content-visibility:auto]` |
| A reusable multi-property pattern | A component ([Part 8](./tailwindcss-reuse-architecture.md)) |

The row that's easiest to get wrong is the first: people write `@utility brand-bg { background: #123 }` when they wanted `@theme { --color-brand: #123 }`. The theme version gives you `bg-brand`, `text-brand`, `border-brand`, `ring-brand` and the `/50` opacity modifier for free. The utility version gives you one class. **If it's a value, it goes in the theme.**

### The legacy JS bridge

v3's JavaScript plugin and config API still works, explicitly loaded ([Functions & Directives](https://tailwindcss.com/docs/functions-and-directives) marks both **Compatibility**):

```css
@config "../../tailwind.config.js";
@plugin "@tailwindcss/typography";
```

`@plugin` is genuinely needed — the first-party plugins are still distributed as JS. `@config` is a migration crutch: use it to get a v3 project building on v4 today, then move the theme into `@theme` and delete it. Note `corePlugins`, `safelist` and `separator` are **not supported** even through `@config`.

<a id="p11-mastery"></a>
## Mastery

### Why the plugin API moved into CSS

v3 plugins were JavaScript: `plugin(({ addUtilities, matchUtilities, theme }) => …)`. That meant a JS module, a build-time API surface, and a config file to register it — three moving parts to add one class.

The v4 equivalent is a CSS at-rule with no registration step. The functional-utility syntax (`--value()`, `--modifier()`, `--default()`) is doing the same job `matchUtilities` did, declaratively.

The deeper reason is the one from [Part 6](./tailwindcss-theming-design.md): once the theme is CSS custom properties rather than a JS object, a JS API has nothing left to read. `theme('colors.red.500')` existed to reach into a JS object at build time. `var(--color-red-500)` needs no API at all.

**What was genuinely lost:** arbitrary computation. A v3 plugin could loop, do maths, read a file, generate a hundred utilities programmatically. `@utility` cannot. For the rare case that truly needs it, `@plugin` is still there — but the trade (declarative, no JS, no registration, works in any toolchain) is right for essentially all real usage.

### Composing the extension points

The pieces are designed to stack. A complete example — a design-system elevation scale that works in both themes:

```css
@import "tailwindcss";

/* 1. tokens */
@theme {
  --shadow-raised: 0 1px 3px oklch(0 0 0 / 0.12);
  --shadow-floating: 0 8px 24px oklch(0 0 0 / 0.16);
  --color-surface: white;
}

/* 2. dark theme overrides the tokens, not the utilities */
@custom-variant dark (&:where(.dark, .dark *));
.dark {
  --color-surface: oklch(0.2 0 0);
  --shadow-raised: 0 1px 3px oklch(0 0 0 / 0.5);
  --shadow-floating: 0 8px 24px oklch(0 0 0 / 0.6);
}

/* 3. a utility for something Tailwind lacks */
@utility surface-* {
  background-color: --value(--color-*);
  box-shadow: --modifier(--shadow-*);
}
```

```html
<div class="surface-surface/raised">…</div>
```

Tokens carry the values, the variant carries the theme switch, the utility carries the composition — and no markup changes when the theme does. That layering is what "owning" Tailwind looks like.

<a id="p11-cheat"></a>
## Part 11 cheat sheet

| Directive | Purpose |
|---|---|
| `@utility name { … }` | Simple utility |
| `@utility name { &::part { … } }` | Multi-rule utility |
| `@utility name-* { p: --value(…) }` | Functional utility |
| `@custom-variant x (&:…)` | New variant, shorthand |
| `@custom-variant x { … @slot; }` | New variant, block form |
| `@variant dark { … }` | Apply a variant in custom CSS |
| `@variant hover:focus { … }` | Stacked (AND) — **v4.3** |
| `@variant hover, focus { … }` | Compound (OR) — **v4.3** |
| `@plugin "…"` | Load a JS plugin (compatibility) |
| `@config "…"` | Load a v3 JS config (compatibility) |

| `--value()` form | Matches |
|---|---|
| `--value(--namespace-*)` | Theme value → `tabx-github` |
| `--value(integer)` | Bare value → `tabx-2` |
| `--value([integer])` | Arbitrary → `tabx-[7]` |
| `--value("inherit")` | Literal keyword |
| `--default(4)` | Bare utility → `tabx` (**v4.3**) |
| `--modifier(--leading-*)` | The `/x` suffix |

**The rule:** if it's a **value**, use `@theme`. If it's a **property with no utility**, use `@utility`. If it's a **condition**, use `@custom-variant`.

[↑ Back to top](#table-of-contents)

---

# Part 12 — Migrating from v3

**Tier note:** Part 12 skips **Beginner** and **Mastery**. There is no beginner tier — you're migrating, so you already know v3 — and no mastery tier, because the deep material is just Parts 6, 10 and 11 applied. It's a checklist Part, deliberately.

<a id="p12-working"></a>
## Working Knowledge

### Run the codemod first

```bash
npx @tailwindcss/upgrade
```

Requires **Node 20+**. Run it on a clean branch and read the diff ([upgrade guide](https://tailwindcss.com/docs/upgrade-guide)).

It handles: the `@tailwind` → `@import` swap, most utility renames, `tailwind.config.js` → `@theme` where it can, the `!important` position flip, and the arbitrary-variable bracket → parenthesis change.

### Check the browser floor before anything else

**Chrome 111, Safari 16.4, Firefox 128.** v4 depends on native cascade layers, `@property` and `color-mix()`, none of which polyfill. If your analytics say you need older browsers, **stay on v3.4** — this is a genuine blocker, not a nudge.

### The rename table

| v3 | v4 | Silent? |
|---|---|---|
| `@tailwind base/components/utilities` | `@import "tailwindcss"` | No — errors |
| `shadow-sm` | `shadow-xs` | **Yes — renders differently** |
| `shadow` | `shadow-sm` | **Yes** |
| `blur-sm` | `blur-xs` | **Yes** |
| `blur` | `blur-sm` | **Yes** |
| `rounded-sm` | `rounded-xs` | **Yes** |
| `rounded` | `rounded-sm` | **Yes** |
| `drop-shadow-sm` | `drop-shadow-xs` | **Yes** |
| `drop-shadow` | `drop-shadow-sm` | **Yes** |
| `backdrop-blur-sm` | `backdrop-blur-xs` | **Yes** |
| `backdrop-blur` | `backdrop-blur-sm` | **Yes** |
| `outline-none` | `outline-hidden` | **Yes** |
| `ring` (3px) | `ring-3` (bare `ring` is now 1px) | **Yes** |
| `bg-opacity-50` etc. | `bg-black/50` | Yes — no rule |
| `flex-shrink-*` / `flex-grow-*` | `shrink-*` / `grow-*` | Yes — no rule |
| `overflow-ellipsis` | `text-ellipsis` | Yes — no rule |
| `decoration-slice` / `-clone` | `box-decoration-slice` / `-clone` | Yes — no rule |
| `!flex` | `flex!` | Yes — no rule |
| `bg-[--var]` | `bg-(--var)` | Yes — invalid CSS |
| `tw-flex` prefix | `tw:flex` | Yes — no rule |
| `start-*` / `end-*` (v4.2) | `inset-s-*` / `inset-e-*` | Deprecated, still works |

The "renders differently" rows are the dangerous ones. The others produce *no* styling, which is obvious. A shadow that's one step too small is not obvious, and it's spread across every card in your app.

<a id="p12-advanced"></a>
## Advanced

### What the codemod cannot fix

These are behaviour changes with no textual signal. Check each by hand.

**1. Default border colour: `gray-200` → `currentColor`.**

```html
<!-- v3: light grey border. v4: border in the text colour. -->
<div class="border p-4">
```

Verified in Preflight: `*, ::before, ::after { border: 0 solid; }` — no colour, so it inherits `currentColor`. On dark text that's a near-black border where you had light grey.

Fix by being explicit, or restore the old default:

```html
<div class="border border-gray-200 p-4">
```

```css
@layer base {
  *, ::after, ::before { border-color: var(--color-gray-200); }
}
```

**2. Ring width: 3px → 1px, and ring colour: `blue-500` → `currentColor`.**

```html
<!-- v3 -->
<button class="focus:ring ring-blue-500">
<!-- v4 equivalent -->
<button class="focus:ring-3 ring-blue-500">
```

**3. `hover:` no longer fires on touch.** v4 wraps it in `@media (hover: hover)` — verified in [Part 3](./tailwindcss-foundations.md). This is usually a *fix* (no more sticky hover states after tapping), but if a mobile interaction depended on tap-triggering hover, it silently stopped working.

```css
@custom-variant hover (&:hover);   /* opt out */
```

**4. `space-x-*` / `space-y-*` changed selector.** Performance-motivated. Mostly invisible; can shift edge cases with mixed inline/block children. The recommendation is to move to `gap` — [Part 4](./tailwindcss-layout-responsive.md).

**5. `container` lost `center` and `padding` config.** Redefine it:

```css
@utility container {
  margin-inline: auto;
  padding-inline: 2rem;
}
```

**6. `corePlugins`, `safelist` and `separator` are unsupported.** `safelist` becomes `@source inline()` ([Part 10](./tailwindcss-engine-extensibility.md)). The other two have no replacement.

**7. Sass/Less/Stylus are unsupported.** If your v3 setup ran Tailwind through Sass, that has to be unwound — see [Part 1](./tailwindcss-foundations.md).

### A migration order that works

1. Check the browser floor. **Stop here if it fails.**
2. Clean branch, commit everything.
3. `npx @tailwindcss/upgrade`, read the whole diff.
4. Delete `postcss.config.js` if you moved to the Vite or webpack plugin.
5. Grep for the silent-behaviour items: `class="border` without a colour, `ring` without a number, `hover:` on touch-only interactions.
6. Move anything left in `tailwind.config.js` into `@theme`; delete the `@config` line.
7. Convert `darkMode: 'class'` to `@custom-variant dark (&:where(.dark, .dark *));` — **the codemod does not always catch this**, and dark mode failing silently is a common post-migration report.
8. Visually diff the app. Shadows and borders especially.

<a id="p12-cheat"></a>
## Part 12 cheat sheet

| Task | Command / syntax |
|---|---|
| Run the codemod | `npx @tailwindcss/upgrade` (Node 20+) |
| Browser floor | Chrome 111, Safari 16.4, Firefox 128 |
| Too old? | Stay on v3.4 |
| Entry point | `@import "tailwindcss";` |
| Config | `@theme { … }` |
| Dark mode by class | `@custom-variant dark (&:where(.dark, .dark *));` |
| Safelist | `@source inline("…")` |
| Content paths | Automatic; `@source` to adjust |
| Restore v3 border colour | `@layer base { *,::after,::before { border-color: var(--color-gray-200) } }` |
| Restore v3 ring | `ring-3` |
| Restore v3 hover on touch | `@custom-variant hover (&:hover);` |
| Restore container padding | `@utility container { margin-inline: auto; padding-inline: 2rem; }` |
| Keep a v3 config temporarily | `@config "../tailwind.config.js";` |
| Load a JS plugin | `@plugin "@tailwindcss/typography";` |

**The five silent ones to check by hand:** border colour, ring width, ring colour, `hover:` on touch, and the `shadow`/`rounded`/`blur` rename shift.

[↑ Back to top](#table-of-contents)

---

*Parts 10–12 of the [Tailwind CSS set](./tailwindcss-mastery-guide.md). Previous: [Reuse & Architecture (Parts 8–9)](./tailwindcss-reuse-architecture.md) · Back to the [overview](./tailwindcss-mastery-guide.md).*
