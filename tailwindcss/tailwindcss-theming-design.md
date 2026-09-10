# Tailwind CSS — Theming & Design (Parts 6–7)

Where the values come from. `@theme` replaced the entire v3 configuration story, and understanding it is the difference between using Tailwind and owning it.

> **Spec:** this doc follows the shared spec in [`tailwindcss-mastery-guide.md`](./tailwindcss-mastery-guide.md#about-this-document) — curiosity-driven, tiered Beginner → Mastery, Tailwind **v4.3.3**, assuming the [`css/`](../css/css-mastery-guide.md) set.

---

## Table of Contents

- [Part 6 — `@theme` & the Design-Token System](#part-6--theme--the-design-token-system)
  - [Beginner](#p6-beginner) · [Working Knowledge](#p6-working) · [Advanced](#p6-advanced) · [Mastery](#p6-mastery) · [Cheat sheet](#p6-cheat)
- [Part 7 — Colour, Typography & Effects](#part-7--colour-typography--effects)
  - [Beginner](#p7-beginner) · [Working Knowledge](#p7-working) · [Advanced](#p7-advanced) · [Cheat sheet](#p7-cheat)

---

# Part 6 — `@theme` & the Design-Token System

<a id="p6-beginner"></a>
## Beginner

Your theme is a block of CSS custom properties. Declaring one creates utilities.

```css
@import "tailwindcss";

@theme {
  --color-brand: oklch(0.72 0.11 178);
}
```

That single line gives you `bg-brand`, `text-brand`, `border-brand`, `fill-brand`, `ring-brand`, `text-brand/50`, `hover:bg-brand`, `dark:md:bg-brand` — the whole cross-product of colour utilities and variants, from one declaration.

This is the entire configuration system. There is no JavaScript, no `theme.extend`, no restart.

> **Outdated convention.** If a tutorial tells you to create `tailwind.config.js` and add colours under `theme.extend.colors`, it is v3. In v4 that file is **not read at all** unless you explicitly point at it with `@config` ([Functions & Directives](https://tailwindcss.com/docs/functions-and-directives) marks `@config` as compatibility-only). It fails silently, which is why this is the most time-wasting v3 leftover in the ecosystem.

### Try It

Add the `@theme` block above to `src/input.css`, then in your HTML:

```html
<div class="bg-brand p-8 text-white">Brand</div>
```

**Expected:** a teal box. Now check `output.css` — you'll find both a token and a utility:

```css
:root { --color-brand: oklch(0.72 0.11 178); }
.bg-brand { background-color: var(--color-brand); }
```

The utility references the variable rather than inlining the value. That indirection is what makes runtime theming possible — see [Advanced](#p6-advanced).

<a id="p6-working"></a>
## Working Knowledge

### The namespaces

The prefix of the variable determines which utilities it generates. From the [theme docs](https://tailwindcss.com/docs/theme):

| Namespace | Powers | Example |
|---|---|---|
| `--color-*` | All colour utilities | `bg-*` `text-*` `border-*` `ring-*` `fill-*` |
| `--spacing` | The whole spacing/sizing scale | `p-*` `m-*` `w-*` `gap-*` |
| `--breakpoint-*` | Responsive variants | `sm:` `md:` |
| `--container-*` | Container query variants & `max-w-*` | `@sm:` |
| `--font-*` | Font families | `font-sans` |
| `--text-*` | Font sizes | `text-xl` |
| `--font-weight-*` | Weights | `font-bold` |
| `--tracking-*` | Letter spacing | `tracking-wide` |
| `--leading-*` | Line height | `leading-tight` |
| `--tab-size-*` | Tab size | `tab-4` |
| `--radius-*` | Border radius | `rounded-lg` |
| `--shadow-*` `--inset-shadow-*` `--drop-shadow-*` | Shadows | `shadow-md` |
| `--blur-*` | Blur filters | `blur-sm` |
| `--perspective-*` | Perspective | `perspective-near` |
| `--zoom-*` | Zoom (v4.3) | `zoom-125` |
| `--aspect-*` | Aspect ratio | `aspect-video` |
| `--ease-*` | Easing functions | `ease-out` |
| `--animate-*` | Animations | `animate-spin` |

**The naming is the API.** `--color-brand` makes `bg-brand` because of the `--color-` prefix, not because Tailwind knows what a colour is. Name it `--brand-color` and you get nothing — a common and genuinely confusing mistake.

### Add, override, remove, replace

```css
@theme {
  /* ADD — new utilities alongside the defaults */
  --color-mint-500: oklch(0.72 0.11 178);
  --font-script: "Great Vibes", cursive;

  /* OVERRIDE — change an existing value, keep the name */
  --breakpoint-sm: 30rem;
  --color-gray-500: oklch(0.5 0.01 250);

  /* REMOVE one */
  --color-lime-*: initial;

  /* REPLACE a whole namespace */
  --color-*: initial;
  --color-white: #fff;
  --color-ink: #121063;
}
```

`--*: initial` is the nuclear option — it clears *everything*, so you start from an empty design system:

```css
@theme {
  --*: initial;
  --spacing: 4px;
  --font-body: Inter, sans-serif;
  --color-lagoon: oklch(0.72 0.11 221.19);
}
```

After that only your utilities exist. `p-4` works (it's computed from `--spacing`), `bg-red-500` does not. This is the right starting point for a strict design system where using an off-palette colour should be impossible rather than merely discouraged.

### `--spacing` is one variable

Everything numeric derives from it:

```css
.p-4 { padding: calc(var(--spacing) * 4); }
```

Default `0.25rem`. Change it and **the entire UI rescales proportionally** — every padding, margin, width, height and gap at once. It's a genuinely powerful lever, and a genuinely dangerous one: `--spacing: 0.2rem` makes your whole app 20% tighter in one line, including places you didn't think about.

<a id="p6-advanced"></a>
## Advanced

### `@theme` vs `@theme inline` vs `@theme static`

These differ in ways that produce confusing bugs if you don't know them.

**`@theme`** (default) — emits the variable, and utilities reference it via `var()`. Utilities are generated lazily; only variables actually used appear in the output.

**`@theme inline`** — utilities get the variable's **value** substituted rather than a `var()` reference:

```css
@theme inline {
  --font-sans: var(--font-inter);
}
```

produces `.font-sans { font-family: var(--font-inter) }` rather than a nested reference chain.

**When `inline` is mandatory:** whenever your theme variable's value is *itself* a variable that's defined somewhere with different scoping — the canonical case being a font loaded by a framework (Next.js's `next/font`, Nuxt fonts) that defines `--font-inter` on a specific element rather than `:root`. Without `inline`, you get `var(--font-sans)` → `var(--font-inter)` resolved in the wrong scope, and the font silently doesn't apply. This is one of the most-reported Tailwind v4 confusions and the fix is one keyword.

**`@theme static`** — emits every variable whether used or not:

```css
@theme static {
  --color-primary: var(--color-red-500);
}
```

Use it when something outside Tailwind's view needs the variable — a chart library reading `getComputedStyle`, a JS animation, a web component. Tailwind can't see those usages, so without `static` the variable may not be emitted at all.

### Runtime theming

Because theme variables are real custom properties, you can override them per-scope with ordinary CSS:

```css
@import "tailwindcss";

@theme {
  --color-surface: white;
  --color-ink: #111;
}

[data-theme="dark"] {
  --color-surface: #111;
  --color-ink: #eee;
}

[data-theme="sepia"] {
  --color-surface: #f4ecd8;
  --color-ink: #5b4636;
}
```

Now `bg-surface text-ink` renders differently under each theme with **no extra utilities generated** and no `dark:` variants in your markup. Switching is one attribute change.

This is the strongest argument for v4's CSS-first design, and it was simply impossible in v3 where `theme()` was a build-time substitution with no runtime existence. For a multi-brand app it's the difference between one stylesheet and N.

### Dark mode

By default `dark:` is the media query:

```css
@media (prefers-color-scheme: dark) { … }
```

To drive it from a class instead, retarget the variant — one line ([dark mode docs](https://tailwindcss.com/docs/dark-mode)):

```css
@import "tailwindcss";
@custom-variant dark (&:where(.dark, .dark *));
```

Or from an attribute:

```css
@custom-variant dark (&:where([data-theme=dark], [data-theme=dark] *));
```

Verified output for the class strategy:

```css
.dark\:bg-brand:where(.dark, .dark *) { background-color: var(--color-brand); }
```

And it composes correctly with everything else — `dark:md:hover:bg-red-500` becomes:

```css
@media (width >= 48rem) {
  @media (hover: hover) {
    .dark\:md\:hover\:bg-red-500:where(.dark, .dark *):hover { … }
  }
}
```

Note the `:where()` — it keeps specificity at zero, so the dark variant doesn't accidentally outrank your other utilities. That detail is why the docs' exact line is worth copying rather than writing your own selector.

> **Outdated convention.** v3 used `darkMode: 'class'` in the JS config. There is no equivalent setting in v4 — you retarget the variant. Since the v3 config is silently ignored, a migrated project will appear to have dark mode "not working" with nothing to debug.

The three-way toggle (light / dark / follow-system), from the docs:

```js
document.documentElement.classList.toggle(
  "dark",
  localStorage.theme === "dark" ||
    (!("theme" in localStorage) &&
      window.matchMedia("(prefers-color-scheme: dark)").matches),
);

localStorage.theme = "light";    // explicit light
localStorage.theme = "dark";     // explicit dark
localStorage.removeItem("theme"); // follow the OS
```

Run that **before first paint** (an inline `<script>` in `<head>`) or you get a flash of the wrong theme.

### Wrong vs. right: `theme()`

```css
/* WRONG — deprecated */
.card { background-color: theme(colors.red.500); }

/* RIGHT */
.card { background-color: var(--color-red-500); }
```

`theme()` is [marked deprecated](https://tailwindcss.com/docs/functions-and-directives). It still works, but there's no reason to write a new one — the variable is right there.

**The one exception:** custom properties don't work in media query conditions, so `theme()` survives there:

```css
@media (width >= theme(--breakpoint-xl)) { … }
```

Note the argument is the *variable name* now, not the v3 dot path (`theme(screens.xl)`).

<a id="p6-mastery"></a>
## Mastery

### Why "the theme is CSS" is the whole design

v3's model: JS config → build-time computation → CSS output. The theme existed only during the build.

v4's model: CSS custom properties → CSS output → **still custom properties in the browser**. The theme is a live runtime object.

Every v4 capability that felt impossible in v3 follows from that single change:

- Runtime theming without regenerating CSS (above)
- `var(--color-blue-500)` in hand-written CSS with no build-time function
- `rounded-[calc(var(--radius-xl)-1px)]` — arbitrary values that *compose with* theme values
- Reading design tokens from JavaScript: `getComputedStyle(root).getPropertyValue("--shadow-xl")`
- A charting library sharing the exact palette of the rest of the UI, with no duplication

It's also why `theme()` had to be deprecated: it was a build-time string substitution, an abstraction whose entire purpose was to work around the theme not existing at runtime. Once the theme is real, the workaround is noise.

### Theme variables are ordinary CSS, with ordinary consequences

Because they're real custom properties, they follow the [custom-property rules](../css/css-architecture.md) — they inherit, they can be set on any element, and they resolve at computed-value time. This gives you scoping for free:

```html
<section class="[--color-brand:oklch(0.6_0.2_25)]">
  <button class="bg-brand">This button is red, not teal</button>
</section>
```

The utility `.bg-brand` is unchanged; the *variable it reads* was overridden for that subtree. One rule, arbitrarily many appearances.

The corresponding footgun is also the ordinary custom-property one: a typo in a variable name doesn't error, it just resolves to nothing. `bg-(--color-brnad)` produces `background-color: var(--color-brnad)` — valid CSS, invalid at computed-value time, and the property falls back to its inherited or initial value with no warning anywhere.

<a id="p6-cheat"></a>
## Part 6 cheat sheet

| Task | Syntax |
|---|---|
| Add a colour | `--color-brand: oklch(…)` → `bg-brand` |
| Override a default | `--color-gray-500: …` |
| Remove one value | `--color-lime-*: initial` |
| Replace a namespace | `--color-*: initial` then your own |
| Empty everything | `--*: initial` |
| Add a breakpoint | `--breakpoint-xs: 30rem` → `xs:` |
| Rescale the whole UI | `--spacing: 0.2rem` |
| Reference another variable | `@theme inline { --font-sans: var(--font-inter) }` |
| Force emission for JS/other tools | `@theme static { … }` |
| Dark mode by class | `@custom-variant dark (&:where(.dark, .dark *));` |
| Dark mode by attribute | `@custom-variant dark (&:where([data-theme=dark], [data-theme=dark] *));` |
| Runtime theme swap | Redefine the variables under a selector |
| Use a token in custom CSS | `var(--color-red-500)` |
| Use a token in a media query | `theme(--breakpoint-xl)` (only place `theme()` survives) |

**Three traps**

1. The **prefix is the API** — `--color-brand` works, `--brand-color` generates nothing.
2. `@theme inline` is **mandatory** when the value is a variable defined outside `:root` (framework fonts).
3. `tailwind.config.js` is **not read**. Silently.

[↑ Back to top](#table-of-contents)

---

# Part 7 — Colour, Typography & Effects

**Tier note:** Part 7 runs **Beginner → Advanced only.** Its "Mastery" material would be colour science and font rendering — `oklch` vs `hsl`, gamut mapping, gradient interpolation, variable-font axes — all of which is CSS and is covered properly in [css-color-visuals.md](../css/css-color-visuals.md) and [css-responsive-typography.md](../css/css-responsive-typography.md). This Part is the Tailwind vocabulary for it, plus the newer utilities.

<a id="p7-beginner"></a>
## Beginner

```html
<p class="text-lg font-semibold text-slate-700">Hello</p>
<div class="bg-blue-500 rounded-lg shadow-md p-4">Card</div>
```

Colours are `{utility}-{name}-{shade}`, shades running 50 (lightest) to 950 (darkest). The default palette ([colours docs](https://tailwindcss.com/docs/colors)):

**Chromatic:** red, orange, amber, yellow, lime, green, emerald, teal, cyan, sky, blue, indigo, violet, purple, fuchsia, pink, rose
**Neutral:** slate, gray, zinc, neutral, stone
**Since v4.2:** mauve, olive, mist, taupe
**Plus:** black, white

The v4.2 four are neutral-adjacent with a hint of colour — for when `gray` is too dead but `slate` is too blue. Verified from the build:

```css
--color-mauve-500: oklch(54.2% 0.034 322.5);
--color-olive-700: oklch(39.4% 0.023 107.4);
--color-mist-200:  oklch(92.5% 0.005 214.3);
--color-taupe-400: oklch(71.4% 0.014 41.2);
```

All defaults are in **oklch**, which is why the v4 palette looks more even across shades than v3's RGB one — equal shade steps are equal *perceptual* steps. See [css-color-visuals.md](../css/css-color-visuals.md) for why that matters.

<a id="p7-working"></a>
## Working Knowledge

### The opacity modifier

```html
<div class="bg-blue-500/75">75% opaque</div>
<div class="bg-sky-500/10 border-black/20 text-white/90">…</div>
<div class="bg-pink-500/[71.37%]">arbitrary</div>
<div class="bg-cyan-400/(--my-alpha)">from a variable</div>
```

Verified output — it compiles to `color-mix()` with an `@supports` upgrade:

```css
.text-brand\/50 {
  color: color-mix(in srgb, oklch(0.72 0.11 178) 50%, transparent);
  @supports (color: color-mix(in lab, red, red)) {
    color: color-mix(in oklab, var(--color-brand) 50%, transparent);
  }
}
```

The srgb version is the fallback; browsers that support `lab` colour spaces get the perceptually-correct `oklab` mix.

> **Outdated convention.** v3's `bg-opacity-50`, `text-opacity-*`, `border-opacity-*`, `divide-opacity-*` and `ring-opacity-*` were **removed** in v4 ([upgrade guide](https://tailwindcss.com/docs/upgrade-guide)). They generate nothing. The `/` modifier replaced all of them and has for a long time — but the old classes are still all over older tutorials.

### `--alpha()` in custom CSS

The `/` modifier is a class-name feature. In hand-written CSS use `--alpha()`:

```css
@layer components {
  .search-hit { background-color: --alpha(var(--color-gray-950) / 10%); }
}
```

Verified — it produces the same `color-mix()` pattern the `/` modifier does.

### Typography utilities

| Need | Utility |
|---|---|
| Size | `text-xs` … `text-9xl` |
| Weight | `font-thin` … `font-black` |
| Family | `font-sans` `font-serif` `font-mono` |
| Line height | `leading-tight` `leading-relaxed`, or `text-lg/7` |
| Letter spacing | `tracking-tight` `tracking-wide` |
| Alignment | `text-left` `text-center` `text-balance` `text-pretty` |
| Truncate | `truncate`, `line-clamp-3` |
| Transform | `uppercase` `capitalize` |
| Decoration | `underline` `line-through` `decoration-2` `underline-offset-4` |
| Overflow | `text-ellipsis` (**not** v3's `overflow-ellipsis`) |

`text-lg/7` is the shorthand worth knowing: size *and* line-height in one utility.

`text-balance` and `text-pretty` map to `text-wrap: balance|pretty` — the modern headline-wrapping properties covered in [css-responsive-typography.md](../css/css-responsive-typography.md). `balance` for headings, `pretty` for body copy.

### `font-features-*` — new in v4.2

OpenType feature control, which previously needed an arbitrary property:

```html
<table class="font-features-[tnum]">  <!-- tabular numerals -->
```

```css
.font-features-\[tnum\] { font-feature-settings: tnum; }
```

`tnum` is the one you'll use most — it makes digits monospaced so numbers in a table column line up. Also useful: `ss01` (stylistic sets), `zero` (slashed zero), `case` (case-sensitive punctuation).

### Effects

| Need | Utility |
|---|---|
| Shadow | `shadow-xs` … `shadow-2xl`, `inset-shadow-*`, `drop-shadow-*` |
| Rounding | `rounded-xs` … `rounded-full` |
| Ring (outline-ish) | `ring`, `ring-2`, `ring-blue-500`, `ring-offset-2` |
| Opacity | `opacity-50` |
| Blur | `blur-sm`, `backdrop-blur-md` |
| Filters | `grayscale` `brightness-110` `saturate-150` |
| Blend | `mix-blend-multiply` `bg-blend-overlay` |
| Gradient | `bg-linear-to-r from-cyan-500 to-blue-500` |
| Transform | `rotate-45` `scale-110` `translate-x-4` |
| 3D transform | `rotate-x-45` `translate-z-8` `perspective-near` (v4.0) |
| Transition | `transition-colors duration-200 ease-out` |

> **The rename trap.** `shadow-sm`, `rounded`, `blur` and `drop-shadow` **all shifted meaning** between v3 and v4. v3's `shadow-sm` is v4's `shadow-xs`; v3's bare `shadow` is v4's `shadow-sm`. These do not error — they render a *different* shadow. It's the nastiest category of migration bug because everything looks almost right. Full table in [Part 12](./tailwindcss-engine-extensibility.md).

### The v4.3 additions

**Scrollbars** — previously required a third-party plugin:

```html
<div class="overflow-auto scrollbar-thin scrollbar-thumb-slate-500 scrollbar-track-slate-100">
<div class="scrollbar-none">              <!-- hide it -->
<div class="scrollbar-gutter-stable">     <!-- reserve space, no layout shift -->
```

Verified:

```css
.scrollbar-thin { scrollbar-width: thin; }
.scrollbar-thumb-red-500 {
  --tw-scrollbar-thumb: var(--color-red-500);
  scrollbar-color: var(--tw-scrollbar-thumb) var(--tw-scrollbar-track);
}
.scrollbar-gutter-stable { scrollbar-gutter: stable; }
```

`scrollbar-gutter-stable` is the quietly valuable one: it reserves scrollbar space so content doesn't jump horizontally when a scrollbar appears — a layout-shift bug that's otherwise annoying to fix.

**Zoom:**

```html
<div class="zoom-125">   <!-- zoom: 125% -->
<div class="zoom-[1.1]">
```

**Tab size:**

```html
<pre class="tab-4">  <!-- tab-size: 4 -->
```

<a id="p7-advanced"></a>
## Advanced

### Wrong vs. right: hard-coded colour

```html
<!-- WRONG -->
<div class="bg-[#3b82f6] text-[#ffffff] border-[#1e40af]">
```

Three off-system values that no theme change will ever reach.

```html
<!-- RIGHT: in the system -->
<div class="bg-blue-500 text-white border-blue-800">

<!-- RIGHT: a genuine brand colour, added once -->
<!-- @theme { --color-brand: #3b82f6; } -->
<div class="bg-brand text-white">
```

### Wrong vs. right: dark mode by inversion

```html
<!-- WRONG: a dark: variant on every single utility -->
<div class="bg-white dark:bg-slate-900 text-slate-900 dark:text-slate-100
            border-slate-200 dark:border-slate-700">
```

This works, and it's what most Tailwind code does. It also doubles your class lists, and every new element is a fresh opportunity to forget one — producing white-on-white text that only appears in dark mode.

```css
/* RIGHT: semantic tokens that already know both themes */
@theme {
  --color-surface: white;
  --color-ink: oklch(0.2 0 0);
  --color-edge: oklch(0.9 0 0);
}
[data-theme="dark"] {
  --color-surface: oklch(0.2 0 0);
  --color-ink: oklch(0.95 0 0);
  --color-edge: oklch(0.35 0 0);
}
```

```html
<div class="bg-surface text-ink border-edge">
```

No `dark:` variants at all. Adding a third theme is three more lines of CSS and zero markup changes. This is the pattern to reach for once an app has more than a handful of components — the `dark:`-on-everything approach doesn't scale, and converting later is a large mechanical edit.

The tradeoff, stated fairly: semantic tokens are an indirection. `bg-surface` requires knowing what "surface" means in your system, where `bg-white` is self-evident. For a small site, `dark:` variants are genuinely simpler. The crossover is somewhere around "more than one person edits this".

### Try It: semantic tokens end to end

In `src/input.css`:

```css
@import "tailwindcss";

@theme {
  --color-surface: white;
  --color-ink: oklch(0.2 0 0);
}

[data-theme="dark"] {
  --color-surface: oklch(0.2 0 0);
  --color-ink: oklch(0.95 0 0);
}
```

In your HTML:

```html
<body>
  <div class="bg-surface text-ink p-8">
    <p>Theme test</p>
    <button onclick="document.body.toggleAttribute('data-theme') ||
                     document.body.setAttribute('data-theme','dark')">
      Toggle
    </button>
  </div>
</body>
```

Simpler, put `data-theme="dark"` on `<body>` by hand and reload.

**Expected:** dark background, light text — with **no `dark:` classes anywhere in the markup**. Confirm in `output.css` that `.bg-surface` appears exactly once, as `background-color: var(--color-surface)`. One utility, two appearances. That's the whole technique.

<a id="p7-cheat"></a>
## Part 7 cheat sheet

| Need | Utility |
|---|---|
| Colour | `bg-blue-500` `text-slate-700` `border-red-300` |
| Transparency | `bg-blue-500/75` (**not** `bg-opacity-75`) |
| Arbitrary alpha | `bg-pink-500/[71.37%]` |
| Alpha in custom CSS | `--alpha(var(--color-gray-950) / 10%)` |
| Neutral-with-warmth | `taupe` `mauve` `olive` `mist` (v4.2) |
| Size + leading | `text-lg/7` |
| Headline wrapping | `text-balance` |
| Body wrapping | `text-pretty` |
| Truncate to N lines | `line-clamp-3` |
| Ellipsis | `text-ellipsis` (**not** `overflow-ellipsis`) |
| Tabular numerals | `font-features-[tnum]` (v4.2) |
| Gradient | `bg-linear-to-r from-cyan-500 to-blue-500` |
| Ring | `ring-2 ring-blue-500 ring-offset-2` |
| Thin scrollbar | `scrollbar-thin` (v4.3) |
| No scrollbar | `scrollbar-none` (v4.3) |
| No scrollbar layout shift | `scrollbar-gutter-stable` (v4.3) |
| Zoom | `zoom-125` (v4.3) |
| Tab width | `tab-4` (v4.3) |

**Renamed in v4 — these render differently, silently**

| v3 | v4 |
|---|---|
| `shadow-sm` | `shadow-xs` |
| `shadow` | `shadow-sm` |
| `rounded` | `rounded-sm` |
| `blur` | `blur-sm` |
| `drop-shadow` | `drop-shadow-sm` |
| `outline-none` | `outline-hidden` |
| `ring` (3px) | `ring-3` |
| `bg-opacity-50` | `bg-black/50` |
| `overflow-ellipsis` | `text-ellipsis` |

[↑ Back to top](#table-of-contents)

---

*Parts 6–7 of the [Tailwind CSS set](./tailwindcss-mastery-guide.md). Previous: [Layout & Responsive (Parts 4–5)](./tailwindcss-layout-responsive.md) · Next: [Reuse & Architecture (Parts 8–9)](./tailwindcss-reuse-architecture.md).*
