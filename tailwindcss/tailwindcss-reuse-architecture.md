# Tailwind CSS — Reuse & Architecture (Parts 8–9)

The Parts about surviving a real codebase: what to do about repetition, and why the cascade behaves the way it does when you push back on it.

> **Spec:** this doc follows the shared spec in [`tailwindcss-mastery-guide.md`](./tailwindcss-mastery-guide.md#about-this-document) — curiosity-driven, tiered Beginner → Mastery, Tailwind **v4.3.3**, assuming the [`css/`](../css/css-mastery-guide.md) set. This is where the set's **Mastery** weight sits, alongside [Engine & Extensibility](./tailwindcss-engine-extensibility.md).

---

## Table of Contents

- [Part 8 — Reusability](#part-8--reusability)
  - [Beginner](#p8-beginner) · [Working Knowledge](#p8-working) · [Advanced](#p8-advanced) · [Mastery](#p8-mastery) · [Cheat sheet](#p8-cheat)
- [Part 9 — Architecture at Scale](#part-9--architecture-at-scale)
  - [Working Knowledge](#p9-working) · [Advanced](#p9-advanced) · [Mastery](#p9-mastery) · [Cheat sheet](#p9-cheat)

---

# Part 8 — Reusability

<a id="p8-beginner"></a>
## Beginner

You've written the same button classes four times:

```html
<button class="px-4 py-2 rounded-md bg-blue-500 text-white font-medium hover:bg-blue-600">Save</button>
<button class="px-4 py-2 rounded-md bg-blue-500 text-white font-medium hover:bg-blue-600">Submit</button>
<button class="px-4 py-2 rounded-md bg-blue-500 text-white font-medium hover:bg-blue-600">Confirm</button>
```

This is the moment every Tailwind user reaches, usually in the first hour. There are four answers, and **the one you'll instinctively reach for is the worst one.**

The four, in the order the [docs](https://tailwindcss.com/docs/styling-with-utility-classes) present them:

1. **A loop** — if the repetition is data, iterate.
2. **A component** — if you have a component system, make a component.
3. **`@layer components`** — plain CSS in Tailwind's component layer.
4. **`@apply`** — inline utilities into a custom class.

Reach for them in that order. `@apply` is last for good reasons, covered below.

<a id="p8-working"></a>
## Working Knowledge

### 1. Loops — for repetition that is really data

```jsx
{contributors.map((user) => (
  <img key={user.id} className="inline-block size-12 rounded-full ring-2 ring-white"
       src={user.avatarUrl} />
))}
```

The classes appear once in the source. There was never a duplication problem — just un-iterated data.

### 2. Components — the primary answer

If you have React, Vue, Svelte, or any template partial system, **that is your abstraction**:

```jsx
export function Button({ children, ...props }) {
  return (
    <button className="px-4 py-2 rounded-md bg-blue-500 text-white font-medium hover:bg-blue-600" {...props}>
      {children}
    </button>
  );
}
```

This is the answer the docs push hardest, and it's right. A component encapsulates markup *and* behaviour *and* styles together, where a CSS class can only encapsulate styles. If your button needs a loading spinner next month, the component can grow one; `.btn` cannot.

Even in a non-JS stack this usually exists — a Django `{% include %}`, a Rails partial, an Astro component, an 11ty shortcode. Use it.

### 3. `@layer components` — for when you have no component system

```css
@import "tailwindcss";

@layer components {
  .btn-primary {
    background-color: var(--color-violet-500);
    padding-inline: --spacing(5);
    padding-block: --spacing(2);
    font-weight: var(--font-weight-semibold);
    color: var(--color-white);
  }
}
```

Note this is **plain CSS using theme variables**, not `@apply`. You get the design system's values without the drawbacks of the next section. `--spacing(5)` is the function form of the spacing scale.

Being in `@layer components` means utilities still win against it, which is exactly what you want:

```html
<button class="btn-primary rounded-none">  <!-- the utility overrides -->
```

### 4. `@apply` — and why it's last

```css
@layer components {
  .btn { @apply px-4 py-2 rounded-md bg-blue-500 text-white; }
}
```

Verified output — note what happens:

```css
@layer components {
  .btn {
    border-radius: var(--radius-md);
    background-color: var(--color-blue-500);
    padding-inline: calc(var(--spacing) * 4);
    padding-block: calc(var(--spacing) * 2);
    color: var(--color-white);
  }
}
```

The utilities are **flattened into plain declarations**. `.btn` is now an ordinary CSS class with no memory of the utilities it came from. That flattening is the source of every problem with `@apply`:

**Three concrete reasons to avoid it:**

1. **You've reinvented the problem Tailwind solves.** You're back to naming things, maintaining a separate stylesheet, and wondering what uses `.btn` before you change it. If the answer is a CSS class either way, the utilities bought you nothing.
2. **It hides growth.** A `.btn` with `@apply` and twelve utilities looks tidy in CSS and is exactly as complex as twelve utilities in markup — but now the complexity is invisible from the call site.
3. **It doesn't compose.** Variants inside `@apply` work, but the result is still one flat rule. You cannot pass overrides in, and you cannot see from the markup what it does.

**When `@apply` *is* correct** — there is a real case, and it's narrow: styling markup you don't control and cannot add classes to.

```css
@layer base {
  /* a CMS or Markdown renderer emits bare <table> tags */
  .prose table { @apply w-full border-collapse text-sm; }
}
```

Here there is no element to put utilities on, so `@apply` is doing genuine work. That's the test: **can you reach the markup?** If yes, put the utilities there.

<a id="p8-advanced"></a>
## Advanced

### The override problem, and `tailwind-merge`

Recall the conflict rule from [Part 2](./tailwindcss-foundations.md): **class attribute order does not decide the winner** — source order in the generated stylesheet does. That has a sharp consequence for components.

```jsx
// WRONG — this does not do what it looks like
function Button({ className, children }) {
  return <button className={`px-4 py-2 bg-blue-500 ${className}`}>{children}</button>;
}

<Button className="bg-red-500">Delete</Button>
```

You get `class="px-4 py-2 bg-blue-500 bg-red-500"`. Both rules exist with equal specificity, and whichever Tailwind emitted later wins — which is **not** necessarily `bg-red-500`, and is not something you control. The button may well stay blue.

This is not a Tailwind bug; it's how CSS has always worked. But it makes the extremely common "component with an overridable className prop" pattern unreliable, and that's why `tailwind-merge` exists:

```jsx
// RIGHT
import { twMerge } from "tailwind-merge";

function Button({ className, children }) {
  return <button className={twMerge("px-4 py-2 bg-blue-500", className)}>{children}</button>;
}
```

`twMerge` understands that `bg-blue-500` and `bg-red-500` are the same *property group* and **removes the loser from the string entirely**, so only one class reaches the DOM and there's no conflict to resolve. The related `cva` (class-variance-authority) builds on the same idea for multi-variant components.

Both are third-party and out of scope for this set ([overview §8](./tailwindcss-mastery-guide.md#8-deliberately-not-covered-and-where-to-look-instead)). The point to carry away is the *problem*: string-concatenating Tailwind classes to override defaults doesn't work, and you need a merge step. Knowing why prevents an hour of confusion the first time it bites.

### `@reference` — and why you probably shouldn't need it

Scoped styles in Vue/Svelte/Astro, and CSS modules, are compiled as **separate stylesheets**. They don't see your `@theme`, so `@apply` in them fails — the utilities don't exist in that context.

`@reference` imports your main stylesheet **for reference only**, making its theme and utilities visible without duplicating any output:

```vue
<style scoped>
  @reference "../app.css";
  button { @apply bg-blue-500; }
</style>
```

The [Compatibility docs](https://tailwindcss.com/docs/compatibility) are unusually direct about this being a workaround, and recommend two things instead:

**Better — use the CSS variable directly.** No Tailwind processing needed at all:

```vue
<style scoped>
  button { background-color: var(--color-blue-500); }
</style>
```

**Best — don't use scoped styles.** Utility classes are already scoped, by virtue of being on the element. A scoped `<style>` block in a Tailwind project is usually solving a problem you no longer have.

The docs are equally blunt about CSS modules: **not recommended**, because they solve a scoping problem utilities don't have, and because Tailwind must run separately for each module, which is slow.

> **Performance note worth taking seriously.** "Tailwind runs separately for each CSS module" means a project with 200 CSS modules runs Tailwind 200 times per build. This is a genuine, measurable build-time problem, not a theoretical one.

<a id="p8-mastery"></a>
## Mastery

### The abstraction question, argued properly

The utility-first critique has a serious version, and it deserves a serious answer rather than dismissal.

**The critique:** utilities put presentation in markup, which is what CSS was invented to prevent. You lose the semantic layer, your HTML is unreadable, and you can't restyle without touching every template. This is the "you've reinvented inline styles" argument.

**What's right about it:** the readability cost is real. So is the coupling — you genuinely cannot restyle a Tailwind app by editing only CSS, and if that's a requirement (a themeable product where customers supply stylesheets, say), utility-first is the wrong choice and you should know that up front.

**What's wrong about it:** the semantic layer it defends mostly didn't exist in practice. Real-world "semantic" CSS drifts into `.card--featured-v2-alt` within a year, because visual variation doesn't decompose into semantic categories. The class name stops describing meaning and starts describing appearance, at which point it's a utility with worse ergonomics and no constraints. And the promised benefit — restyle by editing CSS alone — is rarely exercised; the last time most teams did a pure-CSS redesign was never.

**The synthesis, which is where v4 actually lands:** the useful semantic layer is not class names, it's **design tokens**. `--color-surface` is a semantic abstraction — it says what the colour is *for*, not what it *is* — and it lives in CSS where it can be swapped wholesale. That's how you get the genuine benefit the critique wants (restyle from one place) without the naming treadmill. The [Part 7 dark-mode example](./tailwindcss-theming-design.md) is exactly this: `bg-surface` re-themes an entire app from three lines of CSS, with no markup changes and no `dark:` variants.

So the strong position isn't "utilities instead of abstraction". It's **utilities for composition, tokens for abstraction, components for encapsulation** — three layers, each doing what it's good at. `@apply` is unpopular precisely because it's a fourth thing that does none of them well.

### When Tailwind is the wrong tool

Stated plainly, because a mastery-level understanding includes the boundary:

- **You need runtime-computed styles from data.** Build-time generation can't produce a class for a colour that arrives from an API. Custom properties handle it ([Part 10](./tailwindcss-engine-extensibility.md)), but if that's most of your styling, a runtime system fits better.
- **You must support pre-2023 browsers.** v4's floor is Chrome 111 / Safari 16.4 / Firefox 128, and it's not negotiable — cascade layers and `@property` can't be polyfilled. Use v3.4.
- **Third parties restyle your app via CSS.** Utilities have no stable hooks to target.
- **The project is genuinely tiny.** A landing page with 40 lines of CSS doesn't need a build step.

<a id="p8-cheat"></a>
## Part 8 cheat sheet

| Situation | Answer |
|---|---|
| Repeated markup driven by data | Loop |
| Repeated component, JS framework available | Component |
| Repeated pattern, no component system | `@layer components` with **plain CSS + theme vars** |
| Markup you can't reach (CMS, Markdown output) | `@apply` — the one good case |
| Component with an overridable `className` | `twMerge()` — concatenation does **not** work |
| Multi-variant component API | `cva` |
| Scoped `<style>` needs utilities | `@reference "…"` — but prefer `var(--color-…)` |
| CSS modules | Avoid. Slow (Tailwind runs per module) and unnecessary |

**Why `@apply` is discouraged**

1. Reinvents naming, the thing utilities removed
2. Hides complexity from the call site
3. Flattens to plain CSS — doesn't compose, can't be overridden from markup

[↑ Back to top](#table-of-contents)

---

# Part 9 — Architecture at Scale

**Tier note:** Part 9 skips **Beginner** — there is no beginner-level version of "how do cascade layers interact with a design system", and the material only becomes meaningful once you've hit the problems in Part 8.

<a id="p9-working"></a>
## Working Knowledge

### The layer order is the override order

From [Part 1](./tailwindcss-foundations.md), `@import "tailwindcss"` emits:

```css
@layer theme, base, components, utilities;
```

Native cascade layers, later wins, **specificity is not consulted between layers**. See [css-architecture.md](../css/css-architecture.md) for the underlying mechanism.

```
  theme       :root { --color-*: … }          design tokens
  base        Preflight reset                 element defaults
  components  YOUR component classes          loses to utilities  <-- the point
  utilities   .p-4 { … }                      always wins
  (unlayered) anything outside a layer        beats ALL layers    <-- the trap
```

This produces the behaviour you want by default:

```css
@layer components {
  #main .card { padding: 2rem; }   /* specificity (1,1,0) */
}
```

```html
<div class="card p-0">  <!-- p-0 wins, despite specificity (0,1,0) -->
```

A single utility beats a high-specificity component rule, because layer order is checked *before* specificity. In hand-written CSS you'd need `!important` for this. That's the main reason utility-first CSS doesn't develop specificity wars.

### The unlayered trap

**Anything outside a layer beats everything inside one.** This is the CSS rule, and it's the most common architectural surprise:

```css
@import "tailwindcss";

/* WRONG — unlayered, so it beats every utility */
.card { padding: 2rem; }
```

```html
<div class="card p-0">  <!-- p-0 does NOT win. The card stays at 2rem. -->
```

```css
/* RIGHT — in a layer, so utilities can override */
@layer components {
  .card { padding: 2rem; }
}
```

Same for third-party CSS you `@import` without wrapping. If a vendor stylesheet is fighting your utilities, wrap it:

```css
@layer base {
  @import "some-vendor/styles.css";
}
```

<a id="p9-advanced"></a>
## Advanced

### The two legacy-integration escape hatches

Both are for codebases you don't fully control. Both have real costs.

**`important`** — marks every utility `!important`:

```css
@import "tailwindcss" important;
```

Use when integrating into an app with high-specificity legacy CSS you cannot edit. The cost is that you can no longer override a utility with anything, including another utility — you've traded one specificity problem for a blunter one. Prefer wrapping the legacy CSS in `@layer base` if you can; only reach for `important` when you can't touch the vendor stylesheet at all.

**`prefix(tw)`** — namespaces everything:

```css
@import "tailwindcss" prefix(tw);
```

```html
<div class="tw:flex tw:bg-red-500 tw:hover:p-4">
```

> **Outdated convention.** In v3 the prefix was a *dash* separator (`tw-flex`). In v4 it is a **variant-style colon** (`tw:flex`), and it goes at the *front* of the whole chain — `tw:hover:p-4`, not `hover:tw-p-4` ([upgrade guide](https://tailwindcss.com/docs/upgrade-guide)).

Verified: the prefix renames **both** the classes and the theme variables:

```css
:root {
  --tw-color-red-500: oklch(63.7% 0.237 25.331);
  --tw-spacing: 0.25rem;
}
.tw\:bg-red-500 { background-color: var(--tw-color-red-500); }
```

That second part matters and is easy to miss: any hand-written CSS referencing `var(--color-red-500)` breaks in a prefixed build. Use `var(--tw-color-red-500)`.

### Design tokens across a monorepo

The pattern that works: **one package owns the tokens, every app imports them.**

```css
/* packages/design-tokens/tokens.css */
@theme {
  --color-brand-500: oklch(0.72 0.11 178);
  --color-surface: white;
  --radius-card: 0.75rem;
  --spacing: 0.25rem;
}
```

```css
/* apps/web/app.css */
@import "tailwindcss";
@import "@acme/design-tokens/tokens.css";
@source "../../packages/ui";   /* scan the shared component library */
```

Two things are load-bearing here:

1. **`@source` for the shared package.** Automatic detection respects `.gitignore` and skips `node_modules`, so utilities used only inside `packages/ui` are never generated unless you point at them. This is the number one monorepo Tailwind bug — the shared component library renders unstyled in consuming apps, and nothing errors. [Part 10](./tailwindcss-engine-extensibility.md) covers the scanner properly.
2. **Tokens are just a CSS file.** No build step, no JS config to share, no plugin. Versioning tokens is versioning a CSS file.

**Real Scenario — the monorepo that shipped unstyled buttons.** A team extracts `<Button>` into `packages/ui`. Storybook (which lives inside that package, with its own Tailwind entry) renders it perfectly. The consuming Next.js app renders an unstyled button. Nothing errors; `output.css` in the app simply has no `.bg-brand-500` rule, because the app's Tailwind never looked inside `packages/ui`. The fix is the one `@source` line above. The reason it's worth a scenario is the debugging path: the component is correct, the theme is correct, the classes are in the DOM — the only evidence is a rule *missing* from `output.css`, which is why the [Part 1 habit](./tailwindcss-foundations.md) of keeping that file open pays for itself.

### `source(none)` for explicit control

In a large monorepo, automatic detection may scan far more than you want:

```css
@import "tailwindcss" source(none);
@source "../admin";
@source "../shared";
```

Nothing is scanned except what you list. Slower to set up, dramatically faster to build, and it makes the dependency explicit. For a monorepo with several Tailwind entry points that each need different scopes, this is the correct configuration — [Part 10](./tailwindcss-engine-extensibility.md).

<a id="p9-mastery"></a>
## Mastery

### A layering strategy that scales

Tailwind gives you `base` and `components`. You can declare your own layers alongside them — this is just [CSS `@layer`](../css/css-architecture.md):

```css
@layer theme, base, vendor, components, app-overrides, utilities;

@import "tailwindcss";

@layer vendor {
  @import "flatpickr/dist/flatpickr.css";
}

@layer app-overrides {
  .flatpickr-day.selected { background: var(--color-brand); }
}
```

Now vendor CSS loses to your components, your overrides beat both, and utilities still beat everything. No `!important` anywhere, and the priority order is stated in one readable line at the top of the file rather than being emergent from specificity accidents.

**Redeclaring the layer order is safe** — later `@layer` statements listing the same names don't reset anything, they just establish position for new ones. Declare your full order once, at the top, before the import.

### Choosing a component boundary

The recurring scaling question is where to draw the line between "utilities in markup" and "a named thing". A workable heuristic:

| Signal | Suggests |
|---|---|
| Used 3+ times **and** always identical | Component |
| Used 3+ times with varying props | Component with `cva`/`twMerge` |
| Used twice, different contexts | Leave the utilities inline |
| It's a *value*, not a pattern (a colour, a radius) | `@theme` token |
| It's a *variant of appearance* (`elevated`, `muted`) | `@theme` token + utility, not a class |
| It's a CSS feature Tailwind lacks | `@utility` ([Part 11](./tailwindcss-engine-extensibility.md)) |
| Markup you can't reach | `@apply` in `@layer base` |

The row people get wrong is the fifth. "Elevated card" feels like a component. It's usually a *shadow token* — `--shadow-elevated` — applied as `shadow-elevated`. Tokenising it keeps composition intact; making it a class throws composition away.

### Knowing what you've bought

A final honest accounting, since this is the Mastery tier of the architecture Part:

Tailwind's structural wins are **bundle size that plateaus**, **no specificity wars** (via layers), **safe deletion**, and **a design system that's hard to escape by accident**. Those are real and they compound with codebase size.

Its structural costs are **markup density**, **noisy diffs**, **a build step**, **no runtime-computed styles**, and **a hard browser floor**. Those are also real and don't diminish.

What makes the trade favourable is not the utilities themselves — it's that v4 gave the abstraction layer (tokens) a proper home in CSS. Tailwind v3 forced you to choose between composition and abstraction. v4 doesn't, and that's the version worth learning.

<a id="p9-cheat"></a>
## Part 9 cheat sheet

| Layer | Contains | Beaten by |
|---|---|---|
| `theme` | `:root` design tokens | everything below |
| `base` | Preflight, element defaults | components, utilities, unlayered |
| `components` | Your component classes | utilities, unlayered |
| `utilities` | All utilities | unlayered CSS only |
| *(unlayered)* | Anything outside a layer | **nothing — it wins** |

| Task | Syntax |
|---|---|
| Make component CSS overridable | Put it in `@layer components` |
| Stop vendor CSS beating utilities | `@layer base { @import "vendor.css"; }` |
| Custom layer order | `@layer theme, base, vendor, components, app, utilities;` before the import |
| Force all utilities important | `@import "tailwindcss" important;` |
| Namespace everything | `@import "tailwindcss" prefix(tw);` → `tw:flex`, `var(--tw-color-*)` |
| Share tokens across a monorepo | A `@theme` CSS file, imported |
| Scan a shared package | `@source "../../packages/ui";` |
| Scan nothing automatically | `@import "tailwindcss" source(none);` |

**Two traps**

1. **Unlayered CSS beats every layer**, including utilities. Wrap it.
2. **`prefix()` renames theme variables too** — `var(--tw-color-red-500)`, not `var(--color-red-500)`.

[↑ Back to top](#table-of-contents)

---

*Parts 8–9 of the [Tailwind CSS set](./tailwindcss-mastery-guide.md). Previous: [Theming & Design (Parts 6–7)](./tailwindcss-theming-design.md) · Next: [Engine & Extensibility (Parts 10–12)](./tailwindcss-engine-extensibility.md).*
