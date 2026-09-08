# Modern Responsive Layouts with Tailwind CSS — A Goal-Driven Guide

> ⤴️ **Superseded by [`tailwindcss/tailwindcss-layout-responsive.md`](../../tailwindcss/tailwindcss-layout-responsive.md)** (Parts 4–5), written against **v4.3.3**. This doc predates the v4.2 logical-property utilities (`mbs-*`, `inline-*`, `inset-s-*`, and the `start-*`/`end-*` deprecation) and v4.3's `@container-size`. Kept for reference, not for extending.

## About This Document

- **Framing:** Goal-driven. Written to support building modern, responsive layouts for a real web application — depth and examples are prioritized toward the utilities, patterns, and config you'll actually touch while laying out app UI (dashboards, nav shells, cards, forms, tables), not toward exhaustive coverage of Tailwind as a whole.
- **Companion doc:** [`tailwindcss-mastery-guide.md`](./tailwindcss-mastery-guide.md) is the curiosity-driven core reference (setup, utility fundamentals, JIT/engine internals, theming, plugins). This doc assumes that material (or equivalent Tailwind familiarity) and does **not** re-cover setup, the build pipeline, or general utility syntax — it goes straight to layout.
- **Scope:** Responsive layout only — breakpoints, container queries, flexbox/grid layout utilities, and the concrete app-shell/component patterns built from them (sidebars, dashboards, card grids, forms, tables). Covers systems-level concepts (mobile-first strategy, breakpoints vs. container queries) briefly as foundation, then spends most of the doc on app-shell-first concrete patterns, per confirmed scope.
- **Depth tiers:** Beginner → Working Knowledge → Advanced → Mastery, collapsed where a section doesn't have four honest tiers (e.g. breakpoint foundations stop at Working Knowledge here since Advanced/Mastery breakpoint internals are out of scope for a layout-patterns doc; layout pattern sections lean Working Knowledge/Advanced since that's where production layout work actually lives).
- **Version basis:** Tailwind CSS v4 (`@theme` CSS-first config, container queries as first-class variants, `@container` utility). Checked against official docs as of September 2026.
- **Assumed background:** Comfortable with core Tailwind (utility classes, responsive prefixes like `md:`, the `@theme` directive) and CSS flexbox/grid fundamentals. Not assumed: prior experience with Tailwind's container query variants or subgrid, both covered here from Working Knowledge up.
- **Sourcing standard:** [tailwindcss.com](https://tailwindcss.com/docs) official docs are the primary source, cited inline. Stack Overflow and real-world blog postmortems used for production gotchas (container-query setup failures, dynamic class name JIT misses). When official docs and an outdated (v3-era) convention disagree, the docs win and the difference is called out explicitly.
- **Update instructions for future-me:** Match this structure (numbered Parts, tier subheadings, wrong-vs-right pairs for gotchas, "Real Scenario" production incidents, TOC + back-to-top links). If Tailwind ships a new major version, re-check the default breakpoint/container-size tables in Parts 1–2 first — those are official-docs-sourced tables that drift across majors.

---

## Table of Contents

1. [Orientation — Two Ways to Be Responsive](#1-orientation--two-ways-to-be-responsive)
2. [Part 1 — Breakpoint Foundations](#2-part-1--breakpoint-foundations)
3. [Part 2 — Container Queries in Practice](#3-part-2--container-queries-in-practice)
4. [Part 3 — Flexbox Patterns for App UI](#4-part-3--flexbox-patterns-for-app-ui)
5. [Part 4 — Grid Patterns for Dashboards & Card Layouts](#5-part-4--grid-patterns-for-dashboards--card-layouts)
6. [Part 5 — The App Shell Pattern](#6-part-5--the-app-shell-pattern)
7. [Part 6 — Responsive Forms & Data Tables](#7-part-6--responsive-forms--data-tables)
8. [Part 7 — Production Concerns](#8-part-7--production-concerns)
9. [Cheat Sheets](#9-cheat-sheets)
10. [Suggested Learning Order](#10-suggested-learning-order)
11. [Quick Self-Check](#11-quick-self-check)

---

## 1. Orientation — Two Ways to Be Responsive

Tailwind gives you two independent responsive systems, and most layout bugs in real apps come from reaching for the wrong one:

- **Breakpoint variants** (`sm:`, `md:`, `lg:`...) respond to the **viewport** — the browser window size. Use these for page-level layout: does the whole app switch from a mobile stack to a desktop sidebar layout?
- **Container query variants** (`@sm:`, `@md:`...) respond to the size of a **containing element**, regardless of viewport. Use these for components that need to adapt based on *where they're placed* — a card that's full-width in a single-column feed but cramped inside a 3-up dashboard widget, on the exact same screen size.

```
Viewport-based (breakpoint)          Container-based (@container)
┌─────────── 1440px window ───────────┐   ┌── 320px sidebar ──┐  ┌── 900px main ──┐
│  lg:flex-row kicks in because        │   │ card@sm:flex-col  │  │ card@md:flex-row│
│  the WINDOW crossed 1024px           │   │ (container narrow)│  │ (container wide)│
└───────────────────────────────────────┘   └────────────────────┘  └─────────────────┘
```

This orientation matters immediately: in Part 5 (App Shell) you'll place the *same* card component in a wide main content area and a narrow sidebar, and only container queries make it look right in both places at once. ([Responsive Design — Tailwind CSS](https://tailwindcss.com/docs/responsive-design))

[⬆ back to top](#table-of-contents)

---

## 2. Part 1 — Breakpoint Foundations

### Working Knowledge

**Default breakpoints** — memorize these five, they're the ones you'll type constantly:

| Prefix | Min width | Media query |
|---|---|---|
| `sm` | 40rem (640px) | `@media (width >= 40rem)` |
| `md` | 48rem (768px) | `@media (width >= 48rem)` |
| `lg` | 64rem (1024px) | `@media (width >= 64rem)` |
| `xl` | 80rem (1280px) | `@media (width >= 80rem)` |
| `2xl` | 96rem (1536px) | `@media (width >= 96rem)` |

Every breakpoint variant is **mobile-first**: it applies at that width *and up*, never below it.

**Wrong vs. right — the #1 mobile-first mistake:**

```html
<!-- Wrong: only centers text on screens 640px and WIDER — mobile gets left-align -->
<div class="sm:text-center">Welcome</div>

<!-- Right: mobile default is centered; sm:+ overrides to left-align -->
<div class="text-center sm:text-left">Welcome</div>
```

The unprefixed class is your mobile design. Every prefixed class is an addition for *larger* screens, never a mobile-only override. ([Responsive Design — Tailwind CSS](https://tailwindcss.com/docs/responsive-design))

**Targeting a range** (e.g. "only between md and lg, collapse elsewhere"):

```html
<div class="md:max-lg:flex">
  <!-- Applies only when width is >= 768px AND < 1024px -->
</div>
```

**Custom breakpoints for app-specific layout thresholds** — say your sidebar needs to collapse at a width the default scale doesn't hit:

```css
/* app.css */
@import "tailwindcss";

@theme {
  --breakpoint-tablet: 48rem;   /* replaces reaching for md when "tablet" is the real intent */
  --breakpoint-3xl: 120rem;     /* add a breakpoint above the defaults for ultra-wide dashboards */
}
```

```html
<div class="grid grid-cols-1 tablet:grid-cols-2 3xl:grid-cols-6">...</div>
```
([Customizing your theme — Tailwind CSS](https://tailwindcss.com/docs/responsive-design))

**Real Scenario:** A dashboard's sidebar nav was built with `hidden lg:block` (show sidebar only at 1024px+, mobile gets a hamburger menu). QA files a bug: on a 1080px-wide laptop with the browser at 70% zoom, the sidebar and the collapsed mobile nav *both* render, because zoom changes the effective CSS pixel viewport and the breakpoint boundary was crossed mid-transition, but a stale class from a cached component wasn't rebuilt. The fix wasn't the breakpoint — it was verifying the build actually picked up the new class (see Part 7's dynamic-class-name gotcha for the general failure mode). The lesson that stuck: breakpoint math is reliable; what breaks in practice is usually the *build not generating the class you're relying on*.

[⬆ back to top](#table-of-contents)

---

## 3. Part 2 — Container Queries in Practice

This is the single biggest shift in "modern" Tailwind layout work versus the viewport-only era — components that adapt to their placement, not just the screen.

### Working Knowledge

**Mark a container, then query it:**

```html
<div class="@container">
  <div class="flex flex-col @md:flex-row">
    <img class="@md:w-48" src="..." />
    <div class="p-4">Card content</div>
  </div>
</div>
```

The outer `@container` div sets `container-type: inline-size` on itself. The inner element then responds to *that container's* width via `@md:`, not the viewport's. ([Responsive Design — Container queries](https://tailwindcss.com/docs/responsive-design))

**Container size scale** (distinct from the breakpoint scale — smaller increments, since components are usually narrower than viewports):

| Variant | Min width | Variant | Min width |
|---|---|---|---|
| `@3xs` | 16rem (256px) | `@2xl` | 42rem (672px) |
| `@2xs` | 18rem (288px) | `@3xl` | 48rem (768px) |
| `@xs` | 20rem (320px) | `@4xl` | 56rem (896px) |
| `@sm` | 24rem (384px) | `@5xl` | 64rem (1024px) |
| `@md` | 28rem (448px) | `@6xl` | 72rem (1152px) |
| `@lg` | 32rem (512px) | `@7xl` | 80rem (1280px) |
| `@xl` | 36rem (576px) | | |

**Wrong vs. right — the #1 container query bug:**

```html
<!-- Wrong: @md:flex-row does nothing — no ancestor is a container -->
<div class="flex flex-col @md:flex-row">
  <div>...</div>
</div>

<!-- Right: the parent must carry @container, or the variant has nothing to measure -->
<div class="@container">
  <div class="flex flex-col @md:flex-row">
    <div>...</div>
  </div>
</div>
```

This is consistently reported as the top cause of "my container query just doesn't apply" bugs — a missing `container-type` on the ancestor, not a cascade or specificity issue. ([Fixing CSS Container Queries and Layout Bugs in Tailwind v4 — Loke.dev](https://loke.dev/blog/fixing-container-queries-tailwind-v4-layout-bugs))

### Advanced

**Named containers** — when a component needs to respond to a *distant* ancestor's size, not just its immediate parent (e.g. a deeply nested badge that should react to the overall card width, skipping an intermediate flex wrapper):

```html
<div class="@container/card">
  <div class="flex">
    <div class="flex-1">
      <span class="hidden @sm/card:inline">Full label</span>
    </div>
  </div>
</div>
```

**Max-width and range container queries** — mirror the breakpoint range syntax:

```html
<div class="@container">
  <!-- Stack below @md, row between @sm and @md -->
  <div class="flex-col @sm:@max-md:flex-row @md:flex-row">...</div>
</div>
```

**Real Scenario — the sidebar-aware card:** A product card component renders identically whether it's in the main 900px-wide content grid or a 280px dashboard sidebar widget, because it was built with viewport breakpoints (`md:flex-row`). At 1440px viewport width, both placements pass the `md:` threshold, so the sidebar card renders as a wide horizontal card squeezed into 280px — broken layout, correct viewport. Swapping to `@container`/`@md:flex-row` on each placement's wrapper fixes it permanently: the card now asks "how much room do *I* have," not "how big is the window." This is exactly the class of bug container queries exist to eliminate. ([Container queries — Tailwind CSS](https://tailwindcss.com/docs/responsive-design))

**Custom container sizes** via `@theme`, same mechanism as custom breakpoints:

```css
@theme {
  --container-8xl: 96rem;
}
```

[⬆ back to top](#table-of-contents)

---

## 4. Part 3 — Flexbox Patterns for App UI

### Working Knowledge

**Responsive nav bar — stack on mobile, row on desktop:**

```html
<nav class="flex flex-col gap-2 sm:flex-row sm:items-center sm:justify-between">
  <div class="font-bold text-lg">Logo</div>
  <div class="flex flex-col gap-2 sm:flex-row sm:gap-6">
    <a href="#">Dashboard</a>
    <a href="#">Settings</a>
  </div>
</nav>
```

**Toolbar with wrapping** — buttons that reflow instead of overflowing on narrow screens:

```html
<div class="flex flex-wrap gap-2">
  <button class="px-3 py-1.5 rounded-md bg-slate-100">Filter</button>
  <button class="px-3 py-1.5 rounded-md bg-slate-100">Sort</button>
  <button class="px-3 py-1.5 rounded-md bg-slate-100">Export</button>
</div>
```

`flex-wrap` is doing the responsive work here, not a breakpoint — the toolbar adapts to *however much horizontal space it actually has*, which is often the better default over hand-tuning breakpoints for every toolbar variant.

### Advanced

**Wrong vs. right — flex-basis vs. width for equal-ish columns:**

```html
<!-- Wrong: width doesn't account for gap/padding, columns drift uneven -->
<div class="flex gap-4">
  <div class="w-1/3">A</div>
  <div class="w-1/3">B</div>
  <div class="w-1/3">C</div>
</div>

<!-- Right: flex-1 lets flexbox handle distribution correctly with gap -->
<div class="flex gap-4">
  <div class="flex-1">A</div>
  <div class="flex-1">B</div>
  <div class="flex-1">C</div>
</div>
```

`w-1/3` computes against the parent's full width, ignoring the `gap-4` already consuming space between items — the third column overflows. `flex-1` (`flex: 1 1 0%`) has flexbox distribute the *remaining* space after gaps, which is almost always what "three equal columns" actually means in a flex context. This is a recurring Stack Overflow gotcha for anyone porting fixed-width column code into a flex layout. ([Stack Overflow — Tailwind flex columns overflow with gap](https://stackoverflow.com/questions/tagged/tailwind-css+flexbox))

**Sticky footer action bar pattern** (flex column, content grows, actions pin to bottom):

```html
<div class="flex flex-col h-screen">
  <header class="shrink-0 border-b p-4">Header</header>
  <main class="flex-1 overflow-y-auto p-4">Scrollable content</main>
  <footer class="shrink-0 border-t p-4 flex justify-end gap-2">
    <button>Cancel</button>
    <button>Save</button>
  </footer>
</div>
```

`flex-1` on `<main>` claims all leftover vertical space; `shrink-0` on header/footer stops them from being compressed as content grows — this pairing is the backbone of most app-shell scroll regions.

[⬆ back to top](#table-of-contents)

---

## 5. Part 4 — Grid Patterns for Dashboards & Card Layouts

### Working Knowledge

**Responsive dashboard grid — column count grows with viewport:**

```html
<div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
  <div class="rounded-lg border p-4">Widget</div>
  <!-- ... -->
</div>
```

**Auto-fit card grid** — no breakpoints at all; cards reflow based on available width and a minimum card size, which scales better than hand-picking column counts per breakpoint:

```html
<div class="grid gap-4 grid-cols-[repeat(auto-fit,minmax(240px,1fr))]">
  <div class="rounded-lg border p-4">Card</div>
  <!-- ... -->
</div>
```

Arbitrary-value grid templates (`grid-cols-[...]`) are how you reach native CSS grid features Tailwind doesn't name a utility for — `auto-fit`/`minmax` being the most common. ([grid-template-columns — Tailwind CSS](https://tailwindcss.com/docs/grid-template-columns))

### Advanced

**Wrong vs. right — fixed columns vs. auto-fit for card grids:**

```html
<!-- Wrong: 4 columns look fine at 1440px, but at 1800px each card
     stretches to an awkward ~440px width; at 900px it's cramped -->
<div class="grid grid-cols-4 gap-4">...</div>

<!-- Right: cards stay near their ideal width at any viewport,
     the browser decides the column count -->
<div class="grid gap-4 grid-cols-[repeat(auto-fit,minmax(220px,1fr))]">...</div>
```

Fixed `grid-cols-N` requires a breakpoint per meaningful width range to stay looking intentional. `auto-fit`/`minmax` collapses that maintenance burden into one declaration — reach for it by default for card/tile grids, and reserve fixed `grid-cols-N` + breakpoints for layouts where the column count itself is meaningful (e.g. a 12-column page grid).

**Subgrid — aligning nested grid items to a parent's tracks:**

```html
<div class="grid grid-cols-4 gap-4">
  <div class="col-span-3 grid grid-cols-subgrid gap-4">
    <div class="col-start-2">Aligned to parent's 2nd column</div>
  </div>
</div>
```

Use this when a card's internal label/value columns need to line up across *sibling cards* in a grid — subgrid inherits the parent's actual track sizes instead of redefining its own, which is what makes columns align pixel-perfect across independently-sized cards. ([grid-template-columns — subgrid](https://tailwindcss.com/docs/grid-template-columns))

**Real Scenario:** A metrics dashboard has cards of varying content length, each internally laid out as a 2-column label/value grid. Without subgrid, each card computes its own column widths independently, so the label column width drifts card-to-card — "Revenue" and "Conversion Rate" don't line up. Switching each card's inner grid to `grid-cols-subgrid` against a shared parent track definition fixes the misalignment without hand-tuning widths per card.

[⬆ back to top](#table-of-contents)

---

## 6. Part 5 — The App Shell Pattern

### Working Knowledge

The canonical app shell — collapsible sidebar, sticky header, scrollable main:

```
┌─────────────────────────────────────────┐
│  Header (sticky top-0)                   │
├───────────┬───────────────────────────────┤
│           │                               │
│  Sidebar  │  Main (flex-1, overflow-y)    │
│  (hidden  │                               │
│  lg:block)│                               │
│           │                               │
└───────────┴───────────────────────────────┘
```

```html
<div class="flex h-screen flex-col">
  <header class="sticky top-0 z-10 shrink-0 border-b bg-white p-4">Header</header>
  <div class="flex flex-1 overflow-hidden">
    <aside class="hidden w-64 shrink-0 border-r p-4 lg:block">Sidebar</aside>
    <main class="flex-1 overflow-y-auto p-4">
      <div class="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">...</div>
    </main>
  </div>
</div>
```

Below `lg`, the sidebar is `hidden` entirely and typically replaced by a toggleable off-canvas panel (below) rather than shown at reduced width — a squeezed sidebar nav is rarely usable on mobile.

### Advanced

**Off-canvas mobile sidebar** (toggled via a small amount of JS/framework state, e.g. `x-data`/React state driving a class):

```html
<!-- Backdrop -->
<div class="fixed inset-0 z-20 bg-black/50 lg:hidden" data-open="true"></div>

<!-- Panel: translates on/off screen, always in flow for lg+ -->
<aside class="fixed inset-y-0 left-0 z-30 w-64 -translate-x-full bg-white p-4
              transition-transform duration-200 data-[open=true]:translate-x-0
              lg:static lg:translate-x-0 lg:z-auto">
  Sidebar content
</aside>
```

The pattern: `fixed` + `-translate-x-full` off-screen by default on mobile, slid in via a data-attribute/class toggle, then `lg:static lg:translate-x-0` neutralizes all of that at the desktop breakpoint where the sidebar is permanently visible in-flow. ([Tailwind data attribute variants](https://tailwindcss.com/docs/hover-focus-and-other-states#data-attributes))

**Container-query cards inside the shell** — this is where Part 2 pays off directly. The *same* card component:

```html
<div class="@container">
  <div class="flex flex-col gap-2 @sm:flex-row @sm:items-center @sm:justify-between">
    <h3 class="font-medium">Revenue</h3>
    <span class="text-2xl font-bold">$12,400</span>
  </div>
</div>
```

...stacks vertically at 260px in the sidebar and goes horizontal at 500px in the main grid — automatically, with zero viewport-breakpoint logic, because each placement's `@container` wrapper reports its own actual width.

**Real Scenario:** An app shell was first built with `lg:` breakpoints on the card internals directly (skipping container queries for expedience). It shipped fine — until a "compact sidebar widgets" feature reused the same card component inside the 280px sidebar. At any viewport above 1024px, `lg:flex-row` fired regardless of the sidebar's actual width, breaking the compact widgets. The fix was exactly the swap shown above: move the responsive variant from `lg:` (viewport) to `@sm:` (container) and wrap each placement in `@container`. No component code changed beyond variant prefixes — which is the general lesson: build shell-nested components container-query-first from the start rather than retrofitting later.

[⬆ back to top](#table-of-contents)

---

## 7. Part 6 — Responsive Forms & Data Tables

### Working Knowledge

**Responsive form grid** — single column on mobile, paired fields on desktop:

```html
<form class="grid grid-cols-1 gap-4 sm:grid-cols-2">
  <label class="flex flex-col gap-1">
    First name
    <input class="rounded-md border px-3 py-2" />
  </label>
  <label class="flex flex-col gap-1">
    Last name
    <input class="rounded-md border px-3 py-2" />
  </label>
  <label class="flex flex-col gap-1 sm:col-span-2">
    Email
    <input class="rounded-md border px-3 py-2" type="email" />
  </label>
</form>
```

### Advanced

**Wrong vs. right — data tables on mobile:**

```html
<!-- Wrong: table forces horizontal scroll on mobile with no affordance,
     and users often don't realize the table extends off-screen -->
<table class="w-full">...</table>

<!-- Right: explicit scroll container with a visible boundary, so the cut-off
     is obviously intentional rather than a broken layout -->
<div class="overflow-x-auto rounded-lg border">
  <table class="w-full min-w-[640px]">...</table>
</div>
```

Wrapping in `overflow-x-auto` on a bounded container (not the whole page) contains the scroll to the table itself, and `min-w-[640px]` (or similar) stops columns from crushing illegibly — this is the standard, low-effort fix over building a separate card-based mobile table view, which is only worth the extra work for tables with very few columns.

**Card-based responsive table** (for tables with few enough columns that a mobile card view is worth building):

```html
<!-- Desktop: real table -->
<table class="hidden w-full sm:table">...</table>

<!-- Mobile: each row becomes a card -->
<div class="flex flex-col gap-3 sm:hidden">
  <div class="rounded-lg border p-3" role="row">
    <div class="flex justify-between"><span class="text-sm text-slate-500">Name</span><span>Ada Lovelace</span></div>
    <div class="flex justify-between"><span class="text-sm text-slate-500">Role</span><span>Engineer</span></div>
  </div>
</div>
```

Rendering both and toggling visibility with `hidden`/`sm:table`/`sm:hidden` is simpler than conditional markup in most component frameworks, at the cost of shipping both DOM trees — acceptable for small-to-medium tables, worth reconsidering (server-side or JS-driven single tree) for large ones.

[⬆ back to top](#table-of-contents)

---

## 8. Part 7 — Production Concerns

### Advanced

**Wrong vs. right — the dynamic class name JIT miss:**

```jsx
// Wrong: Tailwind's scanner looks for complete class strings in your source
// files as plain text. `bg-${brand}-600` never appears literally, so no
// bg-*-600 variant gets generated — this renders unstyled in production.
<div className={`bg-${brand}-600`} />

// Right: every class the scanner needs to find must appear complete,
// somewhere in source, even if selected conditionally at runtime.
const brandClass = {
  blue: 'bg-blue-600',
  green: 'bg-green-600',
}[brand]
<div className={brandClass} />
```

This is one of the most common production incidents with Tailwind: it works in dev (where a previous build's CSS may still be cached) and silently fails after a clean production build, because the content scanner never saw the literal string. If class names must be genuinely dynamic, either enumerate every variant as a literal string somewhere in source, or use a [safelist](https://tailwindcss.com/docs/detecting-classes-in-source-files#safelisting-specific-utilities) as a last resort. ([Detecting classes in source files — Tailwind CSS](https://tailwindcss.com/docs/detecting-classes-in-source-files))

**Arbitrary-value overuse becomes design debt.** `top-[13.5px]` and `w-[287px]` are fine as one-offs, but a codebase full of them means every "spacing decision" was made ad hoc, with no shared vocabulary. If the same arbitrary value shows up more than twice, promote it to a theme token:

```css
@theme {
  --spacing-sidebar: 17.5rem;
}
```

```html
<aside class="w-sidebar">...</aside>
```

**Testing responsiveness before shipping:**

- Resize the actual browser viewport through every breakpoint boundary (639→640px, 767→768px, etc.) — devtools device presets often skip the exact pixel where your layout breaks.
- For container-query components, test them in *every placement* they're used, not just the one you built them in — a card that only gets tested at its original 900px placement can silently break the moment it's reused in a 280px sidebar (see Part 5's Real Scenario).
- Check the collapse/expand transition itself for off-canvas sidebars (Part 5), not just the two end states — a `translate-x` transition with the wrong `z-index` stacking is a common source of "sidebar flashes over content" bugs during the animation.

**Dark mode contrast in layout chrome.** Structural elements (borders, dividers, sidebar backgrounds) need distinct dark-mode values, not just inverted text/background — pure black (`dark:bg-black`) removes the depth cues (`border`, subtle background shifts) that make a shell's regions readable as separate regions. Prefer `dark:bg-slate-900` / `dark:border-slate-800` over `dark:bg-black` / `dark:border-black` for shell chrome specifically. ([Dark Mode — Tailwind CSS](https://tailwindcss.com/docs/dark-mode))

[⬆ back to top](#table-of-contents)

---

## 9. Cheat Sheets

**Breakpoint vs. container query — which to reach for:**

| Situation | Use |
|---|---|
| Whole-page layout shifts (sidebar shows/hides) | Breakpoint (`lg:`) |
| A component reused in differently-sized containers | Container query (`@md:`) |
| One-off pixel threshold, used once | Arbitrary value (`min-[500px]:`, `@min-[475px]:`) |
| Threshold reused across the app | Named custom breakpoint/container via `@theme` |

**Layout primitive quick reference:**

| Goal | Utility pattern |
|---|---|
| Equal flex columns with gap | `flex gap-4` + `flex-1` per child (not `w-1/3`) |
| Responsive dashboard grid | `grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3` |
| Auto-reflowing card grid | `grid grid-cols-[repeat(auto-fit,minmax(240px,1fr))]` |
| Columns aligned across sibling cards | `grid-cols-subgrid` on the nested grid |
| Sticky header + scrollable content | `flex flex-col h-screen` parent, `flex-1 overflow-y-auto` on content |
| Off-canvas mobile sidebar | `fixed -translate-x-full` → `lg:static lg:translate-x-0` |
| Component that adapts to placement, not viewport | `@container` wrapper + `@sm:`/`@md:` on children |

**Container query size scale:** `@3xs`(256px) `@2xs`(288px) `@xs`(320px) `@sm`(384px) `@md`(448px) `@lg`(512px) `@xl`(576px) `@2xl`(672px) `@3xl`(768px) `@4xl`(896px) `@5xl`(1024px) `@6xl`(1152px) `@7xl`(1280px).

[⬆ back to top](#table-of-contents)

---

## 10. Suggested Learning Order

1. **Part 1 (Breakpoint Foundations)** — even if familiar with Tailwind generally, confirm the mobile-first mental model is solid; it's the base every later pattern assumes.
2. **Part 3 (Flexbox) → Part 4 (Grid)** — build the layout primitives before combining them into shells.
3. **Part 2 (Container Queries)** — read this before Part 5, not after; the app shell pattern only makes sense once you know why container queries exist.
4. **Part 5 (App Shell)** — the payoff section; builds directly on 1–4.
5. **Part 6 (Forms & Tables)** — narrower, apply as needed when building those specific UI pieces.
6. **Part 7 (Production Concerns)** — read once fully, then keep as a reference to revisit when something breaks in production specifically (dynamic classes, dark mode contrast, cross-placement component bugs).

[⬆ back to top](#table-of-contents)

---

## 11. Quick Self-Check

- Why does `sm:text-center` alone *not* center text on mobile — what does the unprefixed class need to be doing instead?
- A component looks right at its original placement but wrong when reused in a narrower container at the same viewport width — what's the fix, and why does a viewport breakpoint fail here?
- What's the one thing every `@sm:`/`@md:` container variant requires on an ancestor for it to work at all?
- When would you reach for `grid-cols-[repeat(auto-fit,minmax(...))]` instead of `grid-cols-4 lg:grid-cols-6`?
- Why does `flex-1` produce more reliable equal-width columns than `w-1/3` when a `gap` is involved?
- A `className={`bg-${color}-600`}` renders unstyled only in the production build, never locally — what's actually going wrong, and how do you fix it without disabling the content scanner?
- Why use `grid-cols-subgrid` instead of just picking consistent column widths by hand across sibling cards?

[⬆ back to top](#table-of-contents)
