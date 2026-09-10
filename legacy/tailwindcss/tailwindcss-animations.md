# Building Animations with Tailwind CSS — A Goal-Driven Guide

> ⤴️ **Partly superseded by [`tailwindcss/`](../../tailwindcss/tailwindcss-mastery-guide.md)** — the new set covers Tailwind's effect and transition utilities in [Part 7](../../tailwindcss/tailwindcss-theming-design.md), and the underlying CSS animation model is now covered properly in [`css/css-animation-performance.md`](../../css/css-animation-performance.md). This doc's deeper animation-recipe material has no direct replacement, so it is kept for that; treat anything version-specific in it as predating v4.2/v4.3.

## About This Document

- **Framing:** Goal-driven. Written to support building real UI animations with Tailwind — depth and examples are prioritized toward the utilities, config syntax, and plugins you'll actually touch (transitions, keyframes, enter/exit states, scroll-driven motion), not toward exhaustive coverage of Tailwind as a whole.
- **Companion doc:** [`tailwindcss-mastery-guide.md`](./tailwindcss-mastery-guide.md) is the curiosity-driven core reference (setup, utility fundamentals, variants, theming, JIT internals, plugins). This doc assumes that material (or equivalent Tailwind familiarity) and does **not** re-cover setup, the build pipeline, or general utility/variant syntax — it goes straight to animation.
- **Scope:** Transition utilities and their timing/duration/easing controls; built-in `animate-*` keyframe utilities; authoring custom keyframe animations via the v4 `@theme` directive; arbitrary-value one-offs; the community `tw-animate-css` plugin (the v4-compatible successor to `tailwindcss-animate`, common in shadcn/ui-based projects); enter/exit animation with `@starting-style` and `transition-discrete`; and where Tailwind stops and raw CSS (`animation-timeline`, View Transitions API) takes over for scroll-driven and page-transition animation. Framework-specific animation libraries (Framer Motion, GSAP, Vue `<Transition>`) are out of scope — this is Tailwind-native, CSS-only animation.
- **Depth tiers:** Beginner → Working Knowledge → Advanced → Mastery, collapsed where a section doesn't have four honest tiers (e.g. the built-in `animate-*` keyframes stop at Working Knowledge — there's no deep "Mastery" layer to five fixed presets; enter/exit animation and scroll-driven motion lean Advanced/Mastery since that's where the real complexity and browser-support tradeoffs live).
- **Version basis:** Tailwind CSS v4 (CSS-first `@theme` config, `--animate-*` / `--ease-*` / `--duration-*` theme namespaces, `starting:` variant, `transition-discrete`). Checked against official docs as of September 2026. Flagged explicitly wherever v3's `tailwind.config.js`-based animation config differs.
- **Assumed background:** Comfortable with core Tailwind (utility classes, variants, the `@theme` directive, arbitrary-value syntax) per the mastery guide. Not assumed: prior experience with CSS `@starting-style`, `animation-timeline`, or the View Transitions API — all introduced here from first principles since they're recent CSS additions.
- **Sourcing standard:** [tailwindcss.com](https://tailwindcss.com/docs) official docs are the primary source, cited inline. MDN is used for the underlying CSS mechanisms (`@starting-style`, scroll-driven animation, View Transitions) since those are web-platform features Tailwind wraps rather than invents. Stack Overflow and GitHub issues used for real-world gotchas (JIT missing dynamically-built class names, `display:none` transition timing bugs). When official docs and an outdated (v3-era or pre-`@starting-style`) convention disagree, the docs/spec win and the difference is called out explicitly.
- **Update instructions for future-me:** Match this structure (numbered Parts, tier subheadings, wrong-vs-right pairs for gotchas, "Real Scenario" production incidents, TOC + back-to-top links). Re-check Part 5 (`tw-animate-css`) first on any revisit — it's a young community package that has already had one breaking major version and is the piece most likely to drift. Re-check Part 6 for browser support changes — scroll-driven animations and cross-document View Transitions were still stabilizing as of this writing.

---

## Table of Contents

1. [Orientation — Tailwind's Animation Toolkit](#1-orientation--tailwinds-animation-toolkit)
2. [Part 1 — Transition Utilities](#2-part-1--transition-utilities)
3. [Part 2 — Built-in Keyframe Animations](#3-part-2--built-in-keyframe-animations)
4. [Part 3 — Custom Keyframes via `@theme`](#4-part-3--custom-keyframes-via-theme)
5. [Part 4 — Arbitrary Values & One-off Animations](#5-part-4--arbitrary-values--one-off-animations)
6. [Part 5 — The `tw-animate-css` Plugin](#6-part-5--the-tw-animate-css-plugin)
7. [Part 6 — Scroll-Driven & View Transitions](#7-part-6--scroll-driven--view-transitions)
8. [Part 7 — Production Concerns](#8-part-7--production-concerns)
9. [Cheat Sheets](#9-cheat-sheets)
10. [Suggested Learning Order](#10-suggested-learning-order)
11. [Quick Self-Check](#11-quick-self-check)

---

## 1. Orientation — Tailwind's Animation Toolkit

Tailwind doesn't have one "animation system" — it gives you four layers, and most real animation work in an app uses two or three of them together:

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer                  │  Use for                               │
├──────────────────────────┼─────────────────────────────────────┤
│  transition-*            │  Property A → Property B on state     │
│  duration / ease / delay │  change (hover, focus, class toggle)  │
├──────────────────────────┼─────────────────────────────────────┤
│  animate-*                │  Self-running, looping motion         │
│  (built-in or @theme)     │  (spinners, pulses, custom keyframes) │
├──────────────────────────┼─────────────────────────────────────┤
│  starting: / discrete     │  Enter/exit animation — animating     │
│                            │  something appearing or disappearing  │
├──────────────────────────┼─────────────────────────────────────┤
│  raw CSS + arbitrary vals │  Scroll-driven motion, page/element   │
│  (animation-timeline,     │  view transitions — platform features │
│  view-transition-name)    │  Tailwind doesn't wrap with utilities │
└──────────────────────────┴─────────────────────────────────────┘
```

A concrete example that uses three of the four layers: a toast notification that **enters** with a fade+slide (`starting:` + `transition-discrete`), sits idle with a **looping** pulse on its icon (`animate-pulse`), and **transitions** its background color on hover (`transition-colors`). Keeping these layers distinct in your head is the single biggest thing that makes Tailwind animation code readable — mixing "one-shot state transition" and "looping keyframe animation" into the same utility is the most common source of confusion for people new to this.

[Back to top](#building-animations-with-tailwind-css--a-goal-driven-guide)

---

## 2. Part 1 — Transition Utilities

### Beginner

A transition animates a CSS property smoothly between two values when that value changes — on `:hover`, `:focus`, or any state/variant toggle. Tailwind's `transition` utilities are shorthand for `transition-property` + sensible default `transition-duration` (150ms) and `transition-timing-function` (`cubic-bezier(0.4, 0, 0.2, 1)`, i.e. `ease-in-out`-ish).

```html
<button class="bg-blue-500 transition hover:bg-blue-700">
  Hover me
</button>
```

`transition` (no suffix) transitions a broad, curated list of properties Tailwind considers "commonly animated" — colors, opacity, box-shadow, transform/translate/scale/rotate, filter, and a few others — deliberately **not** `all`, because transitioning `all` forces the browser to watch every property for changes, which is wasteful and can cause janky animation on unrelated property changes (e.g. `height` snapping during a layout shift you didn't intend to animate).

```html
<!-- Wrong: transitions everything, including properties you don't want animated -->
<div class="transition-all hover:shadow-lg hover:h-64">...</div>

<!-- Right: scope the transition to what you actually intend to animate -->
<div class="transition-[box-shadow,height] hover:shadow-lg hover:h-64">...</div>
```

[Docs: transition-property](https://tailwindcss.com/docs/transition-property)

### Working Knowledge

The full toolkit for tuning a transition:

| Utility | Controls | Example |
|---|---|---|
| `transition-{property}` | Which properties animate | `transition-colors`, `transition-opacity`, `transition-transform`, `transition-shadow`, `transition-none` |
| `duration-{n}` | How long, in ms | `duration-150`, `duration-300`, `duration-700` |
| `ease-{name}` | The easing curve | `ease-linear`, `ease-in`, `ease-out`, `ease-in-out` |
| `delay-{n}` | Delay before starting, in ms | `delay-150` |

```html
<button class="bg-blue-500 transition delay-150 duration-300 ease-in-out hover:-translate-y-1 hover:scale-110 hover:bg-indigo-500">
  Save Changes
</button>
```

Arbitrary and CSS-variable values work the same way they do everywhere else in Tailwind v4:

```html
<button class="duration-[1200ms] ease-[cubic-bezier(0.95,0.05,0.795,0.035)]">...</button>
<button class="duration-(--my-duration)">...</button>
```

[Docs: transition-duration](https://tailwindcss.com/docs/transition-duration) · [Docs: transition-timing-function](https://tailwindcss.com/docs/transition-timing-function)

### Advanced

**Custom durations and easings belong in `@theme`, not scattered arbitrary values.** If `duration-300` and a custom "snappy" ease show up on ten different components, define them once:

```css
@theme {
  --duration-snappy: 180ms;
  --ease-snappy: cubic-bezier(0.34, 1.56, 0.64, 1);
}
```

```html
<button class="transition duration-snappy ease-snappy">...</button>
```

This is the v4-idiomatic replacement for v3's `theme.extend.transitionDuration` / `theme.extend.transitionTimingFunction` in `tailwind.config.js` — the values live in CSS now, not JS.

**`motion-safe:` / `motion-reduce:` are not optional polish.** Any transition that involves `transform` or that runs longer than ~a couple hundred ms should be gated:

```html
<button class="motion-safe:transition motion-safe:hover:-translate-x-0.5">
  Save changes
</button>
```

Preferring `motion-safe:` over sprinkling `motion-reduce:transition-none` everywhere avoids having to "undo" animation for every property — you opt *into* motion only when the user hasn't asked for less of it.

[Docs: prefers-reduced-motion variants](https://tailwindcss.com/docs/hover-focus-and-other-states)

[Back to top](#building-animations-with-tailwind-css--a-goal-driven-guide)

---

## 3. Part 2 — Built-in Keyframe Animations

### Beginner

Tailwind ships five ready-made, self-running `animate-*` utilities — no keyframe authoring required:

| Class | Effect | Typical use |
|---|---|---|
| `animate-spin` | Linear 360° rotation, 1s, infinite | Loading spinners |
| `animate-ping` | Scale + fade out, 1s, infinite | Notification "radar" badges |
| `animate-pulse` | Gentle opacity fade in/out, 2s, infinite | Skeleton loaders |
| `animate-bounce` | Vertical bounce, 1s, infinite | Scroll-down indicators |
| `animate-none` | Removes any animation | Overriding a variant |

```html
<button type="button" class="bg-indigo-500 ..." disabled>
  <svg class="animate-spin motion-reduce:hidden ..." viewBox="0 0 24 24">...</svg>
  Processing...
</button>
```

Note the `motion-reduce:hidden` — for a spinner specifically, hiding it under reduced motion (rather than freezing it mid-frame) is usually the better call, since "Processing..." text already conveys the state.

[Docs: animation](https://tailwindcss.com/docs/animation)

### Working Knowledge

These five are it — there's no sixth built-in preset hiding in config. Anything beyond spin/ping/pulse/bounce means writing your own keyframes (Part 3) or reaching for a plugin (Part 5). Don't spend time searching for a built-in "fade" or "slide" utility; it doesn't exist, by design — Tailwind keeps the built-in set small and pushes custom motion into the same `@theme` mechanism used for every other design token.

[Back to top](#building-animations-with-tailwind-css--a-goal-driven-guide)

---

## 4. Part 3 — Custom Keyframes via `@theme`

### Beginner

v4's CSS-first config lets you define a `@keyframes` block and register it as a reusable utility with `--animate-*` in the same `@theme` block:

```css
@theme {
  --animate-wiggle: wiggle 1s ease-in-out infinite;

  @keyframes wiggle {
    0%, 100% { transform: rotate(-3deg); }
    50%      { transform: rotate(3deg); }
  }
}
```

```html
<div class="animate-wiggle">🔔</div>
```

The `--animate-wiggle` value is exactly what you'd write for the CSS `animation` shorthand (name, duration, easing, iteration count) — Tailwind just turns it into a class.

**This replaces v3's JS-based config entirely.** In v3 you'd write:

```js
// v3 — tailwind.config.js (superseded, shown for contrast only)
module.exports = {
  theme: {
    extend: {
      keyframes: { wiggle: { '0%, 100%': { transform: 'rotate(-3deg)' }, '50%': { transform: 'rotate(3deg)' } } },
      animation: { wiggle: 'wiggle 1s ease-in-out infinite' },
    },
  },
}
```

If you find a blog post or Stack Overflow answer showing the `tailwind.config.js` form for a v4 project, it still *works* if you're running v4 in compatibility mode with a config file, but the `@theme` CSS form above is the current, official, and simpler path — prefer it for anything new.

[Docs: animation — theme customization](https://tailwindcss.com/docs/animation)

### Working Knowledge

A one-shot entrance animation (no `infinite`) composes naturally with a variant, so it only plays when the state you care about is active:

```css
@theme {
  --animate-fade-in-up: fade-in-up 0.4s ease-out;

  @keyframes fade-in-up {
    from { opacity: 0; transform: translateY(8px); }
    to   { opacity: 1; transform: translateY(0); }
  }
}
```

```html
<div class="opacity-0 data-[state=open]:animate-fade-in-up">...</div>
```

**Real Scenario:** A dropdown menu re-plays its entrance animation every time you scroll the page, because it's implemented as `animate-fade-in-up` unconditionally rather than gated behind an `open` state — the keyframe animation restarts on every re-render/reflow that touches the element's class list. The fix is exactly the pattern above: the animation utility should only be present in the DOM while the state that should trigger it is true, driven by a data attribute or conditional class, not applied unconditionally and left to "just play once."

### Advanced

Keyframes referencing custom properties let one `@theme` animation adapt per-instance without duplicating the whole block:

```css
@theme {
  --animate-slide-in: slide-in 0.3s ease-out;
  @keyframes slide-in {
    from { transform: translateX(var(--slide-from, -100%)); }
    to   { transform: translateX(0); }
  }
}
```

```html
<div class="animate-slide-in [--slide-from:100%]">Slides in from the right</div>
<div class="animate-slide-in">Slides in from the left (uses the default)</div>
```

[Back to top](#building-animations-with-tailwind-css--a-goal-driven-guide)

---

## 5. Part 4 — Arbitrary Values & One-off Animations

### Beginner

For a truly one-off animation not worth registering in `@theme`, use bracket syntax directly:

```html
<div class="animate-[wiggle_1s_ease-in-out_infinite]">...</div>
```

This only works if `wiggle` is defined *somewhere* reachable as a `@keyframes` rule (e.g. in a plain `<style>` block or another CSS file) — Tailwind's arbitrary-value syntax lets you *reference* a keyframe name, it doesn't let you define the keyframes inline in a class name.

```html
<!-- Wrong: there is no @keyframes "wiggle" anywhere — this utility generates CSS
     that references a name that doesn't exist, so nothing visibly animates -->
<div class="animate-[wiggle_1s_ease-in-out_infinite]">...</div>

<!-- Right: define it once via @theme (Part 3) and reuse the generated animate-wiggle,
     or if it's genuinely one-off, define @keyframes wiggle in a <style> block first -->
```

### Working Knowledge

The CSS-variable shorthand (`animate-(--foo)`) is useful when the animation itself needs to be computed or swapped at runtime — e.g. toggled via a CSS custom property rather than a class:

```html
<div class="animate-(--my-animation)" style="--my-animation: wiggle 1s ease-in-out infinite">...</div>
```

In practice, reach for `@theme` (Part 3) by default — it's cacheable, reusable, and shows up in your design system. Arbitrary values are for genuine one-offs (a single decorative element on a single landing page) or values generated dynamically at runtime that can't be known at build time.

[Back to top](#building-animations-with-tailwind-css--a-goal-driven-guide)

---

## 6. Part 5 — The `tw-animate-css` Plugin

### Working Knowledge

`tailwindcss-animate` — the plugin many React/shadcn-ui tutorials reference — is a v3-era JS plugin and is **not v4-compatible** (v4 dropped the old JS plugin API those tutorials rely on). Its v4-compatible successor is [`tw-animate-css`](https://github.com/Wombosvideo/tw-animate-css), a CSS-first package built for v4's architecture, and it's what current shadcn/ui scaffolds install.

```bash
npm i tw-animate-css
```

```css
/* app.css */
@import "tailwindcss";
@import "tw-animate-css";
```

```html
<div data-state="open" class="animate-in fade-in slide-in-from-bottom-4">...</div>
<div data-state="closed" class="animate-out fade-out slide-out-to-bottom-4">...</div>
```

It gives you composable `animate-in`/`animate-out` + `fade-*`/`slide-*`/`zoom-*` building blocks (the same convention popularized by `tailwindcss-animate`), plus a couple of ready-made keyframes (`accordion-down`, `accordion-up`, `caret-blink`) commonly needed for Radix-style accordion/collapsible components that Tailwind's built-in five don't cover.

### Advanced

**Version churn is the real risk here**, not the API itself. The package documents an upcoming `2.0.0` with breaking changes and a migration script — if you install it into a project, pin the version and re-check the changelog before bumping majors, rather than assuming `animate-in`/`fade-in`-style class names are stable across all future versions. This is the one piece of this doc most likely to be outdated by the time you read it — [check the README](https://github.com/Wombosvideo/tw-animate-css) for current class names before relying on this section verbatim.

Before reaching for the plugin at all, weigh it against Part 3: if you only need two or three enter/exit effects, hand-writing them via `@theme` keeps you dependency-free and fully in control of the keyframes. The plugin earns its place when you want the whole composable `fade-*`/`slide-*`/`zoom-*` vocabulary (common in component libraries with many independently-animated pieces: dialogs, dropdowns, tooltips, accordions) rather than writing a bespoke keyframe per component.

[Back to top](#building-animations-with-tailwind-css--a-goal-driven-guide)

---

## 7. Part 6 — Scroll-Driven & View Transitions

### Advanced

**Enter/exit animation for elements that toggle `display: none`** is where transitions historically broke down: a CSS transition needs two states to interpolate between, and an element that's `display: none` has no rendered "from" state to animate from. Tailwind v4 exposes the CSS platform's fix directly:

- `starting:` — wraps a utility in `@starting-style`, defining what the element looks like the instant it first renders or flips from `display: none`.
- `transition-discrete` (`transition-behavior: allow-discrete`) — lets `display` and other normally-instant properties participate in the transition's timing instead of flipping immediately.

```html
<!-- Wrong: no starting state defined — the element just pops in at full opacity,
     because the browser has nothing to transition *from* -->
<div popover class="opacity-100 transition-opacity duration-300">...</div>

<!-- Right: starting: defines opacity-0 as the pre-render state, transition-discrete
     keeps display:none → display:block from cutting the fade short -->
<div popover class="opacity-0 starting:open:opacity-0 open:opacity-100 transition-discrete transition-opacity duration-300">
  ...
</div>
```

[Docs: transition-behavior](https://tailwindcss.com/docs/transition-behavior) · [MDN: @starting-style](https://developer.mozilla.org/en-US/docs/Web/CSS/@starting-style)

### Mastery

**Scroll-driven animations and the View Transitions API are CSS/browser platform features Tailwind does not wrap with dedicated utilities.** As of this writing, browser support for scroll-driven animations (`animation-timeline: scroll()` / `view()`) has reached all major engines, but Tailwind's official utility set has no `scroll-timeline-*` or `animation-timeline-*` classes — you reach these through arbitrary properties and plain CSS:

```html
<div class="animate-[reveal_linear] [animation-timeline:view()] [animation-range:entry_25%_cover_50%]">
  ...
</div>
```

```css
@theme {
  @keyframes reveal {
    from { opacity: 0; transform: translateY(24px); }
    to   { opacity: 1; transform: translateY(0); }
  }
}
```

Same story for the View Transitions API — cross-document and same-document page transitions are configured with `view-transition-name` in plain CSS, then optionally combined with Tailwind's `transition-*`/`animate-*` utilities on the elements being transitioned:

```css
.card { view-transition-name: card-detail; }
```

**Why Tailwind stays out of this layer:** both features are still evolving fast enough (cross-document View Transitions and scroll-timeline browser support were still stabilizing across engines as of 2026) that baking dedicated utilities in now would risk locking in an API shape the platform itself hasn't finished settling. Treat these as "write the CSS directly, use Tailwind's arbitrary-value/property syntax as the bridge" rather than waiting for first-class utilities — and check [caniuse](https://caniuse.com) / [Baseline](https://developer.mozilla.org/en-US/docs/Glossary/Baseline/Compatibility) before shipping either to production without a fallback.

[Back to top](#building-animations-with-tailwind-css--a-goal-driven-guide)

---

## 8. Part 7 — Production Concerns

### Advanced

**Animate GPU-cheap properties, not layout-triggering ones.** `transform` and `opacity` are composited on the GPU and don't trigger layout/reflow; `height`, `width`, `top`, `left`, and `margin` do, and animating them on lower-end devices is where "janky" animation complaints usually come from.

```html
<!-- Wrong: animating height forces layout recalculation on every frame -->
<div class="h-0 transition-[height] hover:h-48">...</div>

<!-- Right: animate transform/opacity, and if content needs to reveal, pair
     scale/opacity with overflow-hidden rather than animating height directly -->
<div class="scale-y-0 opacity-0 origin-top transition [transform,opacity] hover:scale-y-100 hover:opacity-100">
  ...
</div>
```

When you genuinely need to animate an intrinsic-size property (an accordion revealing variable-height content), prefer the CSS `grid-template-rows: 0fr → 1fr` trick or `interpolate-size: allow-keywords` (with `height: auto` as the target) over animating raw pixel heights computed in JS — both keep the animation on the compositor-friendlier path.

**Content-visibility JIT gotcha:** Tailwind's JIT engine scans your source files as plain text for class names — it can't see class names built via string concatenation at runtime.

```jsx
// Wrong: JIT never sees "animate-fade-in" as a literal string, so it never
// generates the CSS for it, and the element silently doesn't animate
<div className={`animate-${isOpen ? 'fade-in' : 'fade-out'}`}>

// Right: write every complete class name literally somewhere in source
<div className={isOpen ? 'animate-fade-in' : 'animate-fade-out'}>
```

This is the single most common "my animation isn't working" report on Stack Overflow for Tailwind projects, and it applies identically to custom `@theme` animations and built-in ones. [See discussion](https://stackoverflow.com/questions/tagged/tailwind-css+dynamic-classnames).

**Real Scenario:** A production skeleton-loader component builds its animation class as `` `animate-${loading ? 'pulse' : 'none'}` `` inside a component library that gets tree-shaken and bundled before Tailwind's build step ever scans it — meaning even a full re-scan of the *app's* source wouldn't help, since the string is assembled inside a pre-built dependency, not visible source text at all. The fix: the library ships pre-built CSS itself, or the class names are passed in as complete literal props from the consuming app (which *is* scanned) rather than assembled inside the library.

**Always gate meaningful motion behind `motion-safe:`** in shipped code — see Part 1 — and additionally consider whether an animation communicates state (a spinner) or is purely decorative (a hero section's floating shapes); decorative motion is the first thing to strip for `motion-reduce:` and the first thing worth `motion-reduce:hidden`-ing rather than merely slowing down.

[Back to top](#building-animations-with-tailwind-css--a-goal-driven-guide)

---

## 9. Cheat Sheets

### Transition utilities

| Class | Effect |
|---|---|
| `transition` | Colors, opacity, shadow, transform, filter (curated list, not `all`) |
| `transition-all` | Every animatable property (avoid — see Part 1) |
| `transition-colors` / `-opacity` / `-shadow` / `-transform` | Scoped subsets |
| `transition-none` | Disables transitions |
| `duration-{n}` | `n` in ms: 75, 100, 150, 200, 300, 500, 700, 1000, or arbitrary `duration-[Nms]` |
| `ease-linear` / `-in` / `-out` / `-in-out` | Built-in easing curves |
| `delay-{n}` | Delay before starting, in ms |
| `transition-discrete` | Let `display`/discrete properties participate in timing |
| `starting:` | `@starting-style` — pre-render/pre-visible state |

### Animation utilities

| Class | Effect |
|---|---|
| `animate-spin` / `-ping` / `-pulse` / `-bounce` / `-none` | Built-in presets |
| `animate-{name}` | Custom, from `@theme { --animate-name: ... }` |
| `animate-[keyframename_1s_ease_infinite]` | Arbitrary value (keyframes must exist elsewhere) |
| `animate-(--var)` | Shorthand for `animate-[var(--var)]` |
| `motion-safe:` / `motion-reduce:` | Gate on `prefers-reduced-motion` |

### `@theme` animation authoring

```css
@theme {
  --animate-name: keyframe-name duration timing-function iteration-count;
  --duration-name: value;
  --ease-name: cubic-bezier(...);

  @keyframes keyframe-name {
    from { /* ... */ }
    to   { /* ... */ }
  }
}
```

[Back to top](#building-animations-with-tailwind-css--a-goal-driven-guide)

---

## 10. Suggested Learning Order

1. **Part 1 (Transitions)** — this is 80% of real-world animation work: hover states, focus states, toggled classes. Get comfortable with `transition-{property}` + `duration` + `ease` before anything else.
2. **Part 2 (Built-in `animate-*`)** — five presets, quick to learn, immediately useful for loaders/spinners.
3. **Part 3 (`@theme` keyframes)** — the moment you need any custom looping or one-shot keyframe animation, this is the v4-idiomatic way to define it.
4. **Part 4 (Arbitrary values)** — know it exists for true one-offs, but default to Part 3.
5. **Part 7 (Production concerns)** — read this before shipping *any* animation, not just at the end; the GPU-property and JIT-visibility gotchas apply from day one.
6. **Part 6 (`starting:`/`transition-discrete`)** — reach for this specifically when building anything that toggles visibility (modals, popovers, toasts, dropdowns).
7. **Part 5 (`tw-animate-css`)** — only if you're pulling in a component pattern (shadcn/ui-style dialogs/accordions) that expects its `animate-in`/`fade-*` vocabulary; otherwise Part 3 covers the same ground with less dependency risk.
8. **Part 6, Mastery (scroll-driven/view transitions)** — last, and only once you need scroll-linked or page-transition motion specifically; this is platform CSS with Tailwind as a thin bridge, not a Tailwind feature to "master" per se.

[Back to top](#building-animations-with-tailwind-css--a-goal-driven-guide)

---

## 11. Quick Self-Check

1. Why does Tailwind's plain `transition` utility deliberately exclude `all`, and when (if ever) is `transition-all` actually the right choice?
2. You need a custom `wiggle` animation used in three components. Where does it belong — an arbitrary `animate-[...]` value, or `@theme`? Why?
3. What's wrong with `` className={`animate-${state}`} `` in a JIT-built project, and what would you write instead?
4. A modal fades out but the underlying content is still clickable/visible for a frame before `display: none` fully applies — which two utilities fix this, and what does each one do?
5. Why is `transform`/`opacity` preferred over `height`/`width` for animated properties, in terms of what the browser actually has to do each frame?
6. `tailwindcss-animate` is referenced in a shadcn/ui tutorial you're following on a v4 project — what's the problem, and what should you install instead?
7. You want an element to animate in based on scroll position, with no JavaScript. Does Tailwind have a dedicated utility for this? What do you write instead?
8. What's the difference in intent between gating an animation behind `motion-safe:` versus `motion-reduce:`, and which should you default to?

[Back to top](#building-animations-with-tailwind-css--a-goal-driven-guide)
