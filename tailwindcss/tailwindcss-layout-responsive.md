# Tailwind CSS — Layout & Responsive (Parts 4–5)

The utility surface for putting boxes where you want them, and for making that adapt — to the viewport, and to the container.

> **Spec:** this doc follows the shared spec in [`tailwindcss-mastery-guide.md`](./tailwindcss-mastery-guide.md#about-this-document) — curiosity-driven, tiered Beginner → Mastery, Tailwind **v4.3.3**, assuming the [`css/`](../css/css-mastery-guide.md) set. These two Parts are the most compressed in the set, because **you already know this CSS** — flexbox, grid and container queries are taught properly in [css-layout-flex-grid.md](../css/css-layout-flex-grid.md) and [css-responsive-typography.md](../css/css-responsive-typography.md). What follows is the *mapping*, plus the parts where Tailwind adds something CSS doesn't have.

---

## Table of Contents

- [Part 4 — Layout & Spacing Utilities](#part-4--layout--spacing-utilities)
  - [Beginner](#p4-beginner) · [Working Knowledge](#p4-working) · [Advanced](#p4-advanced) · [Cheat sheet](#p4-cheat)
- [Part 5 — Responsive Design & Container Queries](#part-5--responsive-design--container-queries)
  - [Beginner](#p5-beginner) · [Working Knowledge](#p5-working) · [Advanced](#p5-advanced) · [Mastery](#p5-mastery) · [Cheat sheet](#p5-cheat)

---

# Part 4 — Layout & Spacing Utilities

**Tier note:** Part 4 runs **Beginner → Advanced only.** Its "Mastery" material would be flexbox and grid internals — `min-width: auto`, formatting contexts, subgrid — which is CSS, not Tailwind, and is covered properly in [css-layout-flex-grid.md](../css/css-layout-flex-grid.md). There is no Tailwind-specific expert layer here; the utilities are a thin naming layer over properties you already understand.

<a id="p4-beginner"></a>
## Beginner

Layout utilities are a direct renaming of CSS properties. If you know the CSS, you know the utility — the only work is vocabulary.

```html
<div class="flex items-center justify-between gap-4">
  <span>Left</span>
  <span>Right</span>
</div>
```

```css
/* what that generates */
.flex           { display: flex; }
.items-center   { align-items: center; }
.justify-between{ justify-content: space-between; }
.gap-4          { gap: calc(var(--spacing) * 4); }
```

The naming convention: `justify-*` is `justify-content`, `items-*` is `align-items`, `self-*` is `align-self`, `content-*` is `align-content`. The value drops the `flex-`/`space-` prefix — `justify-between` not `justify-space-between`.

<a id="p4-working"></a>
## Working Knowledge

### The mapping table

| CSS | Utility |
|---|---|
| `display: flex` | `flex` |
| `display: grid` | `grid` |
| `flex-direction: column` | `flex-col` |
| `flex-wrap: wrap` | `flex-wrap` |
| `justify-content: space-between` | `justify-between` |
| `align-items: center` | `items-center` |
| `align-self: end` | `self-end` |
| `flex: 1 1 0%` | `flex-1` |
| `flex-grow: 1` | `grow` |
| `flex-shrink: 0` | `shrink-0` |
| `gap: 1rem` | `gap-4` |
| `grid-template-columns: repeat(3, minmax(0,1fr))` | `grid-cols-3` |
| `grid-column: span 2 / span 2` | `col-span-2` |
| `grid-template-columns: 1fr 2fr` | `grid-cols-[1fr_2fr]` |
| `place-items: center` | `place-items-center` |
| `position: absolute` | `absolute` |
| `inset: 0` | `inset-0` |
| `z-index: 10` | `z-10` |

> **Outdated convention.** v3's `flex-grow-*` and `flex-shrink-*` were **removed** in v4 in favour of `grow-*` and `shrink-*` ([upgrade guide](https://tailwindcss.com/docs/upgrade-guide)). The old names generate nothing, silently. Same story for `overflow-ellipsis` → `text-ellipsis`, and `decoration-slice`/`decoration-clone` → `box-decoration-*`.

### Sizing

`w-*` and `h-*` take the spacing scale, plus fractions and keywords:

```html
<div class="w-4">     <!-- 1rem -->
<div class="w-1/2">   <!-- 50% -->
<div class="w-full">  <!-- 100% -->
<div class="w-screen"><!-- 100vw -->
<div class="w-fit">   <!-- fit-content -->
<div class="size-8">  <!-- width AND height -->
```

`size-*` is worth calling out — it sets both dimensions, and it's the right choice for icons and avatars where `w-8 h-8` is just noise.

### `gap` vs. `space-*` — prefer `gap`

Tailwind has both `gap-4` (real CSS `gap`) and `space-y-4` (margins injected between children via a `:not(:last-child)`-style selector). They look interchangeable. They are not.

```html
<!-- WRONG-ish: works, but fragile -->
<div class="space-y-4">
  <div>One</div>
  <div>Two</div>
</div>

<!-- RIGHT -->
<div class="flex flex-col gap-4">
  <div>One</div>
  <div>Two</div>
</div>
```

`space-*` is a margin hack from before `gap` had support in flexbox. It breaks when children are reordered, when a child is conditionally rendered, and when children have their own margins. The [upgrade guide](https://tailwindcss.com/docs/upgrade-guide) notes that v4 **changed the `space-*` selector** for performance reasons and explicitly recommends migrating to `gap` — that's the framework telling you the utility is legacy. Use `gap` unless you're on a layout where `gap` genuinely doesn't apply.

### Logical properties — the v4.2 expansion

Tailwind has always had `ms-*`/`me-*` (margin-inline-start/end). **v4.2 filled in the rest of the logical box**, which was a real gap:

| Utility | CSS | Since |
|---|---|---|
| `ms-4` / `me-4` | `margin-inline-start` / `-end` | v4.0 |
| `mbs-4` / `mbe-4` | `margin-block-start` / `-end` | **v4.2** |
| `ps-4` / `pe-4` | `padding-inline-start` / `-end` | v4.0 |
| `pbs-4` / `pbe-4` | `padding-block-start` / `-end` | **v4.2** |
| `inline-8` | `inline-size` (logical width) | **v4.2** |
| `block-6` | `block-size` (logical height) | **v4.2** |
| `min-inline-*` `max-inline-*` `min-block-*` `max-block-*` | logical min/max sizing | **v4.2** |
| `inset-s-4` / `inset-e-4` | `inset-inline-start` / `-end` | **v4.2** |
| `inset-bs-2` / `inset-be-2` | `inset-block-start` / `-end` | **v4.2** |
| `border-bs` / `border-be-2` | `border-block-start/-end-width` | **v4.2** |
| `scroll-mbs-*` `scroll-mbe-*` `scroll-pbs-*` `scroll-pbe-*` | logical scroll margin/padding | **v4.2** |

Verified output:

```css
.mbs-4     { margin-block-start: calc(var(--spacing) * 4); }
.inset-s-4 { inset-inline-start: calc(var(--spacing) * 4); }
.inline-8  { inline-size: calc(var(--spacing) * 8); }
```

> **Deprecated in v4.2:** `start-*` and `end-*` are replaced by `inset-s-*` and `inset-e-*` ([v4.3 release post](https://tailwindcss.com/blog/tailwindcss-v4-3)). Both still compile to identical CSS today — I verified that `start-4` and `inset-s-4` both emit `inset-inline-start` — so this is a rename, not a behaviour change. The reason for it is consistency: `start-4` looked like a standalone property, when it is really the inline-start component of `inset`. New code should use `inset-s-*`.

<a id="p4-advanced"></a>
## Advanced

### When to use logical properties at all

If your app is English-only and will stay that way, `ml-4` and `ms-4` are indistinguishable and the physical names are shorter. The moment there is any chance of an RTL locale, logical properties are the difference between a layout that mirrors correctly and one that needs a parallel RTL stylesheet.

```html
<!-- WRONG for an internationalised app: the icon stays on the left in Arabic -->
<span class="ml-2">→</span>

<!-- RIGHT: flips automatically under dir="rtl" -->
<span class="ms-2">→</span>
```

The [css/ set's Part 3](../css/css-foundations.md) covers the writing-mode model these map onto. The Tailwind-specific advice is just: **pick one convention per project and enforce it**, because a codebase with a mix of `ml-*` and `ms-*` mirrors half-way and looks broken in a way that's genuinely hard to debug.

### Negative values

Prefix the utility, not the number:

```html
<div class="-mt-8">    <!-- margin-top: -2rem -->
<div class="-inset-s-4"><!-- inset-inline-start: -1rem -->
```

### Try It: watch a layout mirror

In the playground:

```html
<div class="flex gap-2 p-4">
  <div class="bg-blue-200 ps-8 pe-2">logical</div>
  <div class="bg-red-200 pl-8 pr-2">physical</div>
</div>
<div dir="rtl" class="flex gap-2 p-4">
  <div class="bg-blue-200 ps-8 pe-2">logical</div>
  <div class="bg-red-200 pl-8 pr-2">physical</div>
</div>
```

**Expected:** in the first row both boxes look identical — wide padding on the left. In the second row (`dir="rtl"`) the blue box has flipped, with its wide padding now on the **right**, while the red box has not moved. That difference is the entire argument for logical properties, in one screen.

<a id="p4-cheat"></a>
## Part 4 cheat sheet

| Need | Utility |
|---|---|
| Flex row, centred, spaced | `flex items-center gap-4` |
| Flex column | `flex flex-col` |
| Push one item right | `ms-auto` (or `ml-auto`) |
| Equal columns | `grid grid-cols-3 gap-4` |
| Custom column track | `grid-cols-[24rem_1fr]` |
| Responsive auto-fit grid | `grid grid-cols-[repeat(auto-fit,minmax(16rem,1fr))]` |
| Square element | `size-8` |
| Centre absolutely | `absolute inset-0 m-auto` or `grid place-items-center` |
| Prevent flex child shrinking | `shrink-0` (**not** `flex-shrink-0`) |
| Space between children | `gap-4` (**not** `space-y-4`) |
| RTL-safe margin | `ms-4` / `me-4` |
| RTL-safe block margin | `mbs-4` / `mbe-4` (v4.2) |
| RTL-safe inset | `inset-s-4` (**not** `start-4`, deprecated v4.2) |
| Logical width/height | `inline-8` / `block-6` (v4.2) |
| Negative | `-mt-8` |

[↑ Back to top](#table-of-contents)

---

# Part 5 — Responsive Design & Container Queries

<a id="p5-beginner"></a>
## Beginner

Breakpoint variants make a utility apply from a given width **upward**:

```html
<img class="w-16 md:w-32 lg:w-48" src="…" />
```

16 by default, 32 from 768px, 48 from 1024px. The defaults ([responsive design docs](https://tailwindcss.com/docs/responsive-design)):

| Prefix | Min width | Generated |
|---|---|---|
| `sm` | 40rem (640px) | `@media (width >= 40rem)` |
| `md` | 48rem (768px) | `@media (width >= 48rem)` |
| `lg` | 64rem (1024px) | `@media (width >= 64rem)` |
| `xl` | 80rem (1280px) | `@media (width >= 80rem)` |
| `2xl` | 96rem (1536px) | `@media (width >= 96rem)` |

Note the modern range syntax (`width >= 40rem`) rather than `min-width:` — see [css-responsive-typography.md](../css/css-responsive-typography.md).

And don't forget the viewport meta tag, or none of this does anything:

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
```

### The one mistake everybody makes

`sm:` does not mean "on small screens". It means "from the small breakpoint **upward**". Tailwind is mobile-first, so the unprefixed utility is your mobile style.

```html
<!-- WRONG: only centres on screens ≥640px. Mobile is left-aligned. -->
<div class="sm:text-center">

<!-- RIGHT: centred everywhere, left-aligned from 640px up -->
<div class="text-center sm:text-left">
```

If you take one thing from Part 5, take this. It is the single most common Tailwind error and it produces a layout that looks fine on your desktop and wrong on every phone.

<a id="p5-working"></a>
## Working Knowledge

### Targeting a range

Stack a `max-*` variant to close the top end:

```html
<div class="md:max-lg:flex">   <!-- only between md and lg -->
<div class="md:max-xl:flex">   <!-- md through xl -->
<div class="max-sm:hidden">    <!-- hidden below sm -->
```

Available: `max-sm`, `max-md`, `max-lg`, `max-xl`, `max-2xl`. This composes exactly like the stacking rule from [Part 3](./tailwindcss-foundations.md) — `md:max-lg:` means "md AND below-lg".

### Custom breakpoints

Breakpoints are theme variables, so you add them the same way you add anything:

```css
@import "tailwindcss";

@theme {
  --breakpoint-xs: 30rem;
  --breakpoint-3xl: 120rem;
}
```

```html
<div class="grid xs:grid-cols-2 3xl:grid-cols-6">…</div>
```

Remove one, or replace the set entirely:

```css
@theme {
  --breakpoint-2xl: initial;        /* drop just 2xl */
}

@theme {
  --breakpoint-*: initial;          /* drop them all, then define your own */
  --breakpoint-tablet: 40rem;
  --breakpoint-laptop: 64rem;
  --breakpoint-desktop: 80rem;
}
```

For genuinely one-off widths, arbitrary breakpoints avoid polluting the theme:

```html
<div class="min-[320px]:text-center max-[600px]:bg-sky-300">…</div>
```

### Container queries — the ones you should usually reach for

Built into core since v4.0; **no plugin**, unlike v3. Mark an element as a container, then size its descendants against *it* rather than the viewport:

```html
<div class="@container">
  <div class="flex flex-col @md:flex-row">…</div>
</div>
```

Verified output:

```css
.@container { container-type: inline-size; }

@container (width >= 28rem) {
  .\@md\:flex-row { flex-direction: row; }
}
```

Note the container scale is **not** the breakpoint scale — `@md` is 28rem, while `md` is 48rem. They're different vocabularies for different jobs, and conflating them is a common source of confusion.

The full toolkit:

```html
<div class="@container">                     <!-- inline-size container -->
<div class="@container/main">                <!-- named -->
<div class="@md:flex-row">                   <!-- min-width -->
<div class="@max-lg:flex-col">               <!-- max-width -->
<div class="@sm/main:gap-4">                 <!-- against the named container -->
<div class="@min-[475px]:flex-row">          <!-- arbitrary -->
<div class="w-[50cqw]">                      <!-- container query units -->
```

Named containers generate `@container main (width >= 24rem)` — verified — which is how you skip past an intermediate container to query an outer one.

### `@container-size` — new in v4.3

`@container` creates an **inline-size** container: you can query its width only. Querying height requires a *size* container, which until v4.3 needed an arbitrary property.

```html
<div class="@container-size">…</div>
<div class="@container-size/main">…</div>
```

```css
.@container      { container-type: inline-size; }
.@container-size { container-type: size; }
```

**Use it sparingly, and know why.** `container-type: size` requires the element's block size to be independent of its contents, which means the browser will not let the container's height be determined by what's inside it. Applying it to an auto-height element will collapse it. That is a CSS constraint, not a Tailwind one — see [css-responsive-typography.md](../css/css-responsive-typography.md). If you only need width (you almost always do), stick with `@container`.

<a id="p5-advanced"></a>
## Advanced

### Media queries vs. container queries: the actual decision

The rule is short: **does this component's layout decision depend on the component's own width, or on the device?**

| Use a breakpoint (`md:`) | Use a container query (`@md:`) |
|---|---|
| Page-level layout — sidebar shows/hides | A card that appears in both a wide main column and a narrow sidebar |
| Navigation collapsing to a hamburger | A component in a design system, used at unknown widths |
| Anything genuinely about the device | Anything reusable |

The test that settles it: **if you could drop this component into a different-width slot and want it to adapt, it's a container query.** A media query gets that case wrong by definition, because the viewport is the same in both slots.

Historically everyone used media queries for everything, because container queries didn't exist. That's why so much Tailwind code you'll read uses `md:` for component-level decisions — it was the only option in v3 without a plugin. It is now a workaround, and the docs treat container queries as core.

### Wrong vs. right: a reusable card

```html
<!-- WRONG: goes horizontal at 768px viewport, even in a 300px sidebar -->
<article class="flex flex-col md:flex-row gap-4">
  <img class="w-full md:w-48" src="…">
  <div>…</div>
</article>
```

Drop that in a narrow sidebar on a desktop and it goes horizontal in 300px of space, and looks broken. The component is reacting to the wrong thing.

```html
<!-- RIGHT: reacts to its own width, correct in any slot -->
<div class="@container">
  <article class="flex flex-col @md:flex-row gap-4">
    <img class="w-full @md:w-48" src="…">
    <div>…</div>
  </article>
</div>
```

### Try It: prove the difference

Paste both versions into the playground, each wrapped in a deliberately narrow box on a wide window:

```html
<div class="w-[300px] border-2 border-red-500 p-2">
  <p class="text-xs mb-2">300px slot — media-query version</p>
  <article class="flex flex-col md:flex-row gap-2">
    <div class="bg-blue-200 p-4">A</div>
    <div class="bg-blue-300 p-4">B</div>
  </article>
</div>

<div class="w-[300px] border-2 border-green-500 p-2 mt-4 @container">
  <p class="text-xs mb-2">300px slot — container-query version</p>
  <article class="flex flex-col @md:flex-row gap-2">
    <div class="bg-blue-200 p-4">A</div>
    <div class="bg-blue-300 p-4">B</div>
  </article>
</div>
```

**Expected, with the browser window wider than 768px:** the red box's A and B sit **side by side**, cramped inside 300px — the media query fired because the *viewport* is wide. The green box's A and B stay **stacked**, because its 300px container is narrower than the 28rem `@md` threshold. That visual difference is the entire argument.

Now narrow the browser window below 768px. **Expected:** the red box stacks (viewport now narrow), the green box is unchanged (its container never changed). The container-query version is the one that was right in both cases.

<a id="p5-mastery"></a>
## Mastery

### Why the container scale differs from the breakpoint scale

`@sm` is 24rem, `@md` 28rem, `@lg` 32rem — verified from the generated `@container` rules. The viewport scale starts at 40rem and climbs to 96rem. They are deliberately different, and it's worth understanding why rather than assuming it's an inconsistency.

Viewport breakpoints are calibrated to **devices**: phone, tablet, laptop, desktop. Container breakpoints are calibrated to **component slots**: a narrow sidebar, a half-width column, a full-width region. A component almost never gets 96rem of space, and a device is almost never 24rem wide. Using one scale for both would leave most of the range useless in each direction.

Both live in the theme (`--breakpoint-*` and `--container-*`), so both are customisable — [Part 6](./tailwindcss-theming-design.md). Adding `--container-8xl: 96rem` gives you `@8xl:` variants.

### Container query units

`cqw`, `cqh`, `cqi`, `cqb`, `cqmin`, `cqmax` are proportional to the container instead of the viewport:

```html
<div class="@container">
  <div class="w-[50cqw]">half the container's width</div>
  <h2 class="text-[clamp(1rem,5cqi,2rem)]">fluid within the container</h2>
</div>
```

That second line is the genuinely powerful one: **fluid typography scoped to a component**. `vw`-based fluid type has always had the flaw that it responds to the window, so a heading in a narrow sidebar gets sized as though it had the whole screen. `cqi` fixes that properly, and it's the technique to reach for in a design system.

### The honest limitation

A container query cannot query its *own* element — only a descendant can query the container. This is why every example wraps the thing being styled in an extra `@container` div. That wrapper is not ceremony; it is structurally required, because an element whose size depends on a query about its own size is circular.

In practice this means container-query-driven components come in pairs: an outer element that declares the container, and an inner element that does the querying. Design your components with that shape from the start rather than discovering it halfway through.

<a id="p5-cheat"></a>
## Part 5 cheat sheet

| Syntax | Means |
|---|---|
| `md:flex` | Viewport ≥ 48rem |
| `max-md:flex` | Viewport < 48rem |
| `md:max-lg:flex` | 48rem ≤ viewport < 64rem |
| `min-[320px]:flex` | Arbitrary viewport min |
| `@container` | Declare an inline-size container |
| `@container/main` | …named `main` |
| `@container-size` | Declare a **size** container (v4.3) — width *and* height |
| `@md:flex-row` | Container ≥ 28rem |
| `@max-lg:flex-col` | Container < 32rem |
| `@sm/main:gap-4` | Named container `main` ≥ 24rem |
| `@min-[475px]:flex-row` | Arbitrary container min |
| `w-[50cqw]` | 50% of container width |
| `text-[clamp(1rem,5cqi,2rem)]` | Fluid type scoped to the container |

| Breakpoint | Viewport | Container |
|---|---|---|
| `sm` / `@sm` | 40rem | 24rem |
| `md` / `@md` | 48rem | 28rem |
| `lg` / `@lg` | 64rem | 32rem |
| `xl` | 80rem | — |
| `2xl` | 96rem | — |

**The three rules that matter**

1. Unprefixed = mobile. `sm:` means "640px **and up**", never "on small screens".
2. Reusable component → container query. Page layout → media query.
3. `@container` for width (almost always). `@container-size` only when you truly need height, and only on an element whose height doesn't depend on its contents.

**Customising**

```css
@theme {
  --breakpoint-xs: 30rem;       /* adds xs: */
  --breakpoint-2xl: initial;    /* removes 2xl: */
  --container-8xl: 96rem;       /* adds @8xl: */
}
```

[↑ Back to top](#table-of-contents)

---

*Parts 4–5 of the [Tailwind CSS set](./tailwindcss-mastery-guide.md). Previous: [Foundations (Parts 1–3)](./tailwindcss-foundations.md) · Next: [Theming & Design (Parts 6–7)](./tailwindcss-theming-design.md).*
