# Modern CSS — Animation & Performance (Parts 13–14)

The fun Part and the serious Part, in that order. Transitions and keyframes from scratch, then the modern additions that removed whole JavaScript libraries — entry animations for elements that start hidden, view transitions, scroll-driven animation, anchor positioning. Then the rendering model underneath all of it, so you can tell why something is janky rather than guessing.

> **Spec:** this doc follows the canonical spec in [`css-mastery-guide.md`](./css-mastery-guide.md#about-this-document). Assumes [Foundations](./css-foundations.md) and [Layout Part 4](./css-layout-flex-grid.md#part-4--positioning--stacking) (stacking contexts and containing blocks — several features here create both).

---

## Table of Contents

- [Part 13 — Transitions, Animations & the Frontier](#part-13--transitions-animations--the-frontier)
  - [Beginner: `transition`](#p13-beginner)
  - [Working Knowledge: `@keyframes`, `transform`, easing](#p13-working)
  - [Advanced: entry/exit animation, `@starting-style`, `interpolate-size`](#p13-advanced)
  - [Mastery: view transitions, scroll-driven animation, anchor positioning](#p13-mastery)
  - [Part 13 cheat sheet](#part-13-cheat-sheet)
- [Part 14 — Rendering & Performance](#part-14--rendering--performance)
  - [Beginner: the pixel pipeline](#p14-beginner)
  - [Working Knowledge: animating the cheap properties](#p14-working)
  - [Advanced: `contain`, `content-visibility`, `will-change`](#p14-advanced)
  - [Mastery: reading a performance trace, and selector cost](#p14-mastery)
  - [Part 14 cheat sheet](#part-14-cheat-sheet)

---

## Part 13 — Transitions, Animations & the Frontier

<a id="p13-beginner"></a>

### Beginner: `transition`

A transition animates a property when its value *changes* — you declare the animation once, on the base state, and every state change uses it.

```css
.button {
  background: var(--brand);
  transition: background 200ms ease;
}
.button:hover { background: var(--brand-hover); }
```

The shorthand is `property duration timing-function delay`:

```css
transition: opacity 300ms ease-out 100ms;
transition: opacity 300ms, transform 300ms 50ms;   /* comma-separated list */
```

**Put the transition on the base rule, not the state rule.** On the base rule it applies in both directions; on `:hover` it only applies going *in*, and the element snaps back instantly on mouse-out:

```css
/* WRONG - fades in over 200ms, snaps out instantly */
.card:hover { opacity: 1; transition: opacity 200ms; }

/* RIGHT */
.card { opacity: .8; transition: opacity 200ms; }
.card:hover { opacity: 1; }
```

**Not everything is animatable.** A property is animatable if the browser knows how to compute values between the two endpoints. `opacity` (a number), `color` (a colour), `transform` (a matrix) — yes. `display`, `position`, `font-family` — no; these are *discrete*, and by default they simply flip at the end. [Advanced](#p13-advanced) covers the modern way to transition the discrete ones.

**`transition: all` is a mistake**, even though it looks convenient. It animates properties you didn't intend (including ones set by third-party CSS and by user-agent defaults), it makes the browser check every property on every change, and it produces bizarre effects when a class toggle changes twelve things at once. Name the properties.

<a id="p13-working"></a>

### Working Knowledge: `@keyframes`, `transform`, easing

**`@keyframes`** describes an animation independent of any state change — it runs on its own timeline:

```css
@keyframes fade-up {
  from { opacity: 0; transform: translateY(1rem); }
  to   { opacity: 1; transform: none; }
}

.toast {
  animation: fade-up 300ms ease-out both;
}
```

The `animation` shorthand is `name duration timing-function delay iteration-count direction fill-mode play-state`. Two of those need explanation:

- **`fill-mode`**: `forwards` keeps the final frame after the animation ends; `backwards` applies the first frame during the delay; `both` does both. Without it, the element snaps back to its unanimated state the instant the animation finishes — the cause of "my animation flickers at the end".
- **`iteration-count`**: a number or `infinite`. Paired with `direction: alternate` for ping-pong effects.

Percentage keyframes give you more than two waypoints, and multiple selectors can share a frame:

```css
@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50%      { transform: scale(1.06); }
}
```

**`transform`** is the workhorse, because — as [Part 14](#part-14--rendering--performance) explains — it is one of the two properties the browser can animate without recalculating layout:

```css
transform: translate(10px, 20px) rotate(3deg) scale(1.1);   /* applied right to left */
```

There are also **individual transform properties**, which are usually the better choice because they cascade independently:

```css
.card { translate: 0 0; rotate: 0deg; scale: 1; }
.card:hover { translate: 0 -4px; scale: 1.02; }   /* rotate is untouched */
```

With the `transform` shorthand, a `:hover` rule setting `transform: scale(1.02)` would wipe an existing `translate` — the same shorthand-reset trap from [Foundations Part 1](./css-foundations.md#p1-working). The individual properties avoid it entirely.

`transform-origin` moves the pivot (`center` by default), and `perspective` on a *parent* plus `transform-style: preserve-3d` unlocks real 3D.

**Easing** ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/easing-function)):

| Function | Feel | Use for |
|---|---|---|
| `linear` | Mechanical, constant | Progress bars, spinners |
| `ease-out` | Fast start, gentle stop | **Things entering** — the default you want most often |
| `ease-in` | Gentle start, fast end | Things leaving |
| `ease-in-out` | Gentle both ends | Movement between two on-screen positions |
| `cubic-bezier(x1,y1,x2,y2)` | Custom curve | Overshoot: `cubic-bezier(.34,1.56,.64,1)` |
| `steps(n, jump-end)` | Discrete jumps | Sprite sheets, typewriter effects |
| `linear(0, 0.25 75%, 1)` | Arbitrary multi-point curve | Approximating springs and bounces |

`linear()` is the newer one worth knowing: it takes a list of progress stops and interpolates linearly between them, which lets you approximate a spring or bounce curve that a single cubic-bézier cannot express. MDN notes it is part of Easing Functions Level 2 and support varies — feature-detect with `@supports (transition-timing-function: linear(0, 1))` before relying on it.

**The reduced-motion rule.** Some users get nauseated by movement. Honour their setting — but *replace* the motion rather than deleting all feedback, which is what MDN's own guidance recommends ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion)):

```css
/* WRONG - removes the feedback entirely, so state changes become invisible */
@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; transition: none !important; }
}

/* RIGHT - swap movement for a fade, keep the feedback */
.toast { animation: fade-up 300ms ease-out both; }
@media (prefers-reduced-motion: reduce) {
  .toast { animation: fade-in 150ms ease-out both; }   /* opacity only, no translate */
}
```

Test it with DevTools → Rendering → "Emulate prefers-reduced-motion".

<a id="p13-advanced"></a>

### Advanced: entry/exit animation, `@starting-style`, `interpolate-size`

Two things CSS could not do until recently, and that every JS animation library existed partly to work around.

**Problem 1: an element that starts hidden cannot transition in.** A transition needs a *from* value, and an element that has just been inserted, or has just gone from `display: none` to `display: block`, has no previous value to transition from. The browser skips the transition entirely.

**`@starting-style`** supplies that missing "from" state ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@starting-style), Baseline since August 2024):

```css
.toast {
  opacity: 1;
  transform: translateY(0);
  transition: opacity 300ms, transform 300ms;

  @starting-style {
    opacity: 0;
    transform: translateY(1rem);
  }
}
```

MDN is explicit about one gotcha: `@starting-style` and the rule it applies to have **equal specificity**, so the `@starting-style` block must come **after** the base rule (or be nested inside it, as above) or it loses on source order and does nothing.

**Problem 2: `display` is discrete.** Transitioning `opacity` while also setting `display: none` fails, because `display` flips immediately and the element vanishes before the fade runs. **`transition-behavior: allow-discrete`** fixes the timing ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/transition-behavior)): the browser flips a discrete property at 0% when entering and at 100% when exiting, so the element stays visible for the whole animation.

Together, these give a fully declarative dialog or popover animation with no JavaScript beyond opening it:

```css
dialog {
  opacity: 0;
  transform: scale(.96);
  transition:
    opacity 200ms ease,
    transform 200ms ease,
    display 200ms allow-discrete,
    overlay 200ms allow-discrete;
}

dialog[open] {
  opacity: 1;
  transform: scale(1);
}

@starting-style {
  dialog[open] { opacity: 0; transform: scale(.96); }
}

dialog::backdrop {
  background: rgb(0 0 0 / 0);
  transition: background 200ms, display 200ms allow-discrete, overlay 200ms allow-discrete;
}
dialog[open]::backdrop { background: rgb(0 0 0 / 40%); }
@starting-style { dialog[open]::backdrop { background: rgb(0 0 0 / 0); } }
```

The `overlay` property in that transition list is the piece people miss: a modal `<dialog>` is promoted to the browser's **top layer**, and `overlay` is what controls when it is removed from it. Without `overlay … allow-discrete`, the dialog drops out of the top layer immediately on close and the exit animation plays behind the rest of the page.

**Problem 3: `height: auto` doesn't animate.** The classic accordion problem, historically solved with `max-height` tricks that were always slightly wrong (guess too low and content clips; too high and the timing is off). **`interpolate-size: allow-keywords`** makes intrinsic sizes interpolable ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/interpolate-size)):

```css
:root { interpolate-size: allow-keywords; }

.accordion-panel {
  block-size: 0;
  overflow: hidden;
  transition: block-size 250ms ease;
}
.accordion-panel[data-open] { block-size: auto; }
```

`calc-size(auto)` does the same thing for a single value without the document-wide opt-in. **Both are experimental and limited in availability** — MDN flags this explicitly — so keep the `max-height` fallback and gate the enhancement:

```css
.accordion-panel { max-block-size: 0; transition: max-block-size 250ms ease; }
.accordion-panel[data-open] { max-block-size: 50rem; }

@supports (interpolate-size: allow-keywords) {
  :root { interpolate-size: allow-keywords; }
  .accordion-panel { max-block-size: none; block-size: 0; transition: block-size 250ms ease; }
  .accordion-panel[data-open] { block-size: auto; }
}
```

**Try It — a popover that animates both ways.**

```html
<style>
  [popover] {
    opacity: 0; transform: translateY(8px); border: 1px solid; padding: 1rem;
    transition: opacity 250ms, transform 250ms, display 250ms allow-discrete, overlay 250ms allow-discrete;
  }
  [popover]:popover-open { opacity: 1; transform: translateY(0); }
  @starting-style { [popover]:popover-open { opacity: 0; transform: translateY(8px); } }
</style>
<button popovertarget="p1">Toggle</button>
<div id="p1" popover>Hello. I fade in and out.</div>
```

**Expected:** the popover fades and slides *both* on open and on close. Delete the `@starting-style` block: it now appears instantly but still fades out. Delete `display … allow-discrete` as well: it appears instantly and *disappears* instantly. Each piece is doing exactly one job.

<a id="p13-mastery"></a>

### Mastery: view transitions, scroll-driven animation, anchor positioning

Three features that each replaced a category of JavaScript. All three are newer than the rest of this set — **check the [baseline table](./css-mastery-guide.md#5-the-baseline-table) and feature-detect.**

**View transitions** animate between two DOM states by snapshotting the old and new renderings and cross-fading (or morphing) them ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API)).

Same-document, one line of JavaScript:

```js
document.startViewTransition(() => {
  // make any DOM change here - the browser animates from old to new
  list.prepend(newItem);
});
```

Give an element a `view-transition-name` and it animates *independently* — the morphing hero-image effect:

```css
.hero-image { view-transition-name: hero; }

::view-transition-old(hero),
::view-transition-new(hero) {
  animation-duration: 400ms;
}
```

The pseudo-element tree the browser builds is worth knowing, because it is what you style:

```
  ::view-transition                      (the overlay root)
    +-- ::view-transition-group(name)    (animates position and size)
          +-- ::view-transition-image-pair(name)
                +-- ::view-transition-old(name)   (a snapshot of the old state)
                +-- ::view-transition-new(name)   (a live view of the new state)
```

Cross-document transitions — actual page-to-page navigation in a multi-page app — need no JavaScript at all, just an opt-in on both pages:

```css
@view-transition { navigation: auto; }
```

Two rules: a `view-transition-name` must be **unique on the page** at the time of the transition (two elements sharing one breaks it), and always feature-detect the JS entry point:

```js
if (!document.startViewTransition) { update(); return; }
document.startViewTransition(update);
```

**Scroll-driven animations** run a `@keyframes` animation against scroll position instead of time ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_scroll-driven_animations)). The whole scroll-progress-bar / reveal-on-scroll genre, with no scroll listener and no `IntersectionObserver`:

```css
@keyframes grow { from { transform: scaleX(0); } to { transform: scaleX(1); } }

.progress-bar {
  transform-origin: left;
  animation: grow linear;
  animation-timeline: scroll(root block);   /* tied to the document's scroll */
}
```

Two timeline types:

- **`scroll()`** — progress through a scroll container. Arguments are a scroll container (`nearest`, `root`, `self`) and an axis (`block`, `inline`, `x`, `y`).
- **`view()`** — progress of *this element* through the scrollport. This is the reveal-on-scroll one:

```css
.reveal {
  animation: fade-up linear both;
  animation-timeline: view();
  animation-range: entry 0% cover 40%;   /* start as it enters, finish 40% in */
}
```

Named timelines (`scroll-timeline: --main`, `view-timeline: --img`) let one element drive another's animation, with `timeline-scope` widening where the name is visible.

Because support is still rolling out, gate it and keep a static fallback:

```css
@supports (animation-timeline: scroll()) {
  .reveal { animation: fade-up linear both; animation-timeline: view(); }
}
```

And note the accessibility angle: scroll-driven animation is *motion tied to the user's own input*, which is usually less problematic than autoplaying motion — but it should still respect `prefers-reduced-motion`.

**Anchor positioning** tethers one element to another in pure CSS, which is precisely the job Floating UI and Popper.js exist to do ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_anchor_positioning/Using)):

```css
.tooltip-trigger { anchor-name: --trigger; }

.tooltip {
  position: absolute;
  position-anchor: --trigger;
  position-area: block-start center;         /* place it above, centred */
  margin-block-end: .5rem;

  position-try-fallbacks: block-end center;  /* flip below if it would overflow */
}
```

The pieces: `anchor-name` marks the anchor; `position-anchor` binds the positioned element to it; `anchor()` reads an edge (`top: anchor(bottom)`); `anchor-size()` reads a dimension (`inline-size: anchor-size(width)`); `position-area` places the element in a 3×3 grid around the anchor; `position-try-fallbacks` lists alternative placements to try when the first would overflow the viewport; and `position-visibility` hides the element when the anchor scrolls away.

That last group — automatic flipping and hiding — is the part that was genuinely hard in JavaScript, and it is why this feature matters more than it first appears. Verify current support before shipping it as the only mechanism.

### Part 13 cheat sheet

| Task | Declaration |
|---|---|
| Animate a state change | `transition: background 200ms ease` on the **base** rule |
| Named animation | `animation: fade-up 300ms ease-out both` |
| Keep the final frame | `animation-fill-mode: forwards` (or `both`) |
| Move something cheaply | `translate: 0 -4px` (individual property, not the `transform` shorthand) |
| Entering element | `ease-out` |
| Overshoot | `cubic-bezier(.34, 1.56, .64, 1)` |
| Spring-ish curve | `linear(0, .25 75%, 1)` — verify support |
| Animate in from hidden | `@starting-style { … }` **after** the base rule |
| Transition `display`/`overlay` | `transition: display 200ms allow-discrete` |
| Animate `height: auto` | `interpolate-size: allow-keywords` — experimental, keep a fallback |
| Respect motion preference | Swap translate for opacity under `prefers-reduced-motion: reduce` |
| Morph between DOM states | `document.startViewTransition(fn)` + `view-transition-name` |
| Page-to-page transition | `@view-transition { navigation: auto }` on both pages |
| Scroll progress bar | `animation-timeline: scroll(root block)` |
| Reveal on scroll | `animation-timeline: view(); animation-range: entry 0% cover 40%` |
| Tooltip tethered to a button | `anchor-name` + `position-anchor` + `position-area` |
| Auto-flip when it would overflow | `position-try-fallbacks` |

| Trap | Reality |
|---|---|
| `transition` on the `:hover` rule | Only animates one direction |
| `transition: all` | Animates properties you didn't mean; name them |
| `transform` shorthand in a hover rule | Wipes other transform components — use `translate`/`scale`/`rotate` |
| `@starting-style` before the base rule | Loses on source order — put it after |
| Dialog exit animation plays behind the page | Add `overlay … allow-discrete` |
| Two elements with the same `view-transition-name` | Breaks the transition |
| Disabling all motion for reduced-motion users | Removes feedback — replace it instead |

[↑ Back to top](#table-of-contents)

---

## Part 14 — Rendering & Performance

<a id="p14-beginner"></a>

### Beginner: the pixel pipeline

Every visual change goes through up to five stages ([web.dev](https://web.dev/articles/rendering-performance)):

```
   JavaScript  -->  Style  -->  Layout  -->  Paint  -->  Composite
   (or CSS         (which     (geometry:   (fill in    (assemble
    animation)      rules      where and    pixels,     layers in
                    match)     how big)     colours,    order and
                                            shadows)    draw)
```

Not every change runs the whole pipeline, and that is the entire performance story:

| You change | Pipeline runs |
|---|---|
| `width`, `height`, `margin`, `padding`, `top`, `left`, `font-size`, `display` | Style → **Layout** → Paint → Composite |
| `background-color`, `color`, `box-shadow`, `border-radius`, `visibility` | Style → Paint → Composite |
| **`transform`, `opacity`** | Style → **Composite only** |

The budget: for 60fps you have about 16ms per frame, and the browser needs some of that for itself — web.dev's guidance is to finish your work inside roughly **10ms**. Layout on a large tree does not fit in 10ms. Compositing a transform does.

This yields the single most repeated rule in CSS performance, and it is genuinely correct:

```css
/* WRONG - animates layout on every frame */
.card { transition: left 200ms, height 200ms; }

/* RIGHT - composite-only */
.card { transition: translate 200ms, opacity 200ms; }
```

<a id="p14-working"></a>

### Working Knowledge: animating the cheap properties

Most "make it smooth" work is translating a layout-property animation into a transform one.

| Instead of | Use |
|---|---|
| `left` / `top` | `translate` |
| `width` / `height` | `scale` (with `transform-origin`) |
| `margin-top` | `translate` |
| `display: none` ↔ `block` | `opacity` + `transition-behavior: allow-discrete` ([Part 13](#p13-advanced)) |
| `background-position` | `translate` on an inner layer |
| `box-shadow` (on hover) | A pseudo-element with a shadow, animating its `opacity` |

That last one is worth spelling out, because animating `box-shadow` is a common source of paint jank — blurred shadows are expensive to rasterise, and re-rasterising on every frame is what makes a hover feel gluey:

```css
/* WRONG - repaints a large blurred shadow every frame */
.card { transition: box-shadow 200ms; }
.card:hover { box-shadow: 0 12px 32px rgb(0 0 0 / 20%); }

/* RIGHT - paint the shadow once, animate only its opacity */
.card { position: relative; }
.card::after {
  content: "";
  position: absolute;
  inset: 0;
  border-radius: inherit;
  box-shadow: 0 12px 32px rgb(0 0 0 / 20%);
  opacity: 0;
  transition: opacity 200ms;
  pointer-events: none;
}
.card:hover::after { opacity: 1; }
```

**Scaling has a caveat:** `scale` stretches everything including borders and text, so an element scaled from 1 to 1.5 has visibly blurry text mid-animation and a 1.5×-thick border at the end. For size changes where fidelity matters, either scale a background-only layer, or accept a layout animation on a small, isolated element.

<a id="p14-advanced"></a>

### Advanced: `contain`, `content-visibility`, `will-change`

**`contain`** promises the browser that an element's internals cannot affect anything outside it, which lets it skip work ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_containment/Using_CSS_containment)):

| Value | Promise |
|---|---|
| `layout` | Nothing inside affects layout outside, and vice versa |
| `paint` | Descendants are clipped to the padding box and never painted outside it |
| `style` | Counters and quotes inside don't leak out |
| `size` | The element's size does not depend on its children — **requires `contain-intrinsic-size`** or it collapses to zero |
| `content` | Shorthand for `layout paint style` — safe to apply broadly |
| `strict` | Shorthand for `size layout paint style` — needs explicit sizing |

`contain: content` on repeated, independent blocks (feed items, comments, table rows) is a cheap win: appending one item no longer forces the browser to reconsider layout for the whole page.

```css
.feed-item { contain: content; }
```

**`content-visibility: auto`** goes further and skips rendering entirely for content that isn't near the viewport:

```css
.feed-item {
  content-visibility: auto;
  contain-intrinsic-size: auto 400px;   /* the size to assume while skipped */
}
```

`contain-intrinsic-size` is not optional in practice. Without it, skipped elements report zero height, the scrollbar jumps as you scroll, and the page feels broken. The `auto` prefix tells the browser to remember the real size once it has been rendered once and use that thereafter — the best of both.

Two important properties of `content-visibility: auto`, both from MDN:

- Skipped content is **still findable** — in-page find, focus, and selection all force it to render. That is what makes it safe for accessibility, and it is the key difference from `content-visibility: hidden`, which removes content from the accessibility tree entirely.
- It applies `layout paint style` containment, which — like everything in that family — **creates a containing block and a stacking context** ([Layout Part 4](./css-layout-flex-grid.md#p4-mastery)).

**When it makes things worse:** on short pages, or on elements that are all in the viewport anyway, you pay the containment cost for no skipping benefit. And an element whose real height differs wildly from `contain-intrinsic-size` produces scroll-position jumps as it renders. Measure before and after; this is not a decoration.

**`will-change`** tells the browser to prepare for a change — usually by promoting the element to its own compositor layer ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/will-change)):

```css
.modal { will-change: transform, opacity; }
```

MDN's warnings are unusually strong, and they are correct: **use it sparingly, as a last resort for a measured problem, never as a blanket optimisation.** Each promoted layer costs memory, and a page with hundreds of promoted layers is slower, not faster. The browser also holds the optimisation longer than you expect.

```css
/* WRONG - promotes every card permanently */
.card { will-change: transform; }

/* RIGHT - add it just before the change, remove it after */
```

```js
el.addEventListener("pointerenter", () => { el.style.willChange = "transform"; });
el.addEventListener("transitionend", () => { el.style.willChange = "auto"; });
```

And note: **elements being animated by CSS animations are already treated as `will-change`** — you do not need to declare it for a running `@keyframes`. Also remember it creates a stacking context, so adding it can change what paints over what.

**Outdated convention:** `transform: translateZ(0)` and `backface-visibility: hidden` as "GPU acceleration hacks". They worked by forcing layer promotion as a side effect. `will-change` is the explicit, supported way to express the same intent — and both were always overused.

<a id="p14-mastery"></a>

### Mastery: reading a performance trace, and selector cost

**The workflow.** Open DevTools → Performance, click record, interact with the janky thing for a few seconds, stop. Then:

1. **Look at the Frames track first.** Long frames are red. Click one.
2. **Read the flame chart for that frame.** The colours are the pipeline: yellow is scripting, purple is *Recalculate Style* and *Layout*, green is *Paint* and *Composite*.
3. **A wide purple "Layout" block** means something forced geometry recalculation. Click it — DevTools names the element and, for forced synchronous layouts, links to the JavaScript line that triggered it.
4. **A wide green "Paint" block** means expensive rasterisation — large blurred shadows, `filter`, `backdrop-filter`, big gradients. Use the **Layers** panel and the "Paint flashing" toggle under Rendering to see what is repainting.
5. **No layout, no paint, still janky** — usually too many compositor layers (check the Layers panel) or long-running JavaScript blocking the main thread.

**Layout thrashing** is the classic JavaScript-side cause and shows up as a sawtooth of tiny purple blocks. It happens when you interleave reads and writes:

```js
// WRONG - each read forces a synchronous layout because of the previous write
for (const el of items) {
  el.style.height = el.offsetHeight + 10 + "px";
}

// RIGHT - batch reads, then batch writes
const heights = items.map(el => el.offsetHeight);
items.forEach((el, i) => { el.style.height = heights[i] + 10 + "px"; });
```

DevTools flags these explicitly as "Forced reflow" warnings — they are one of the few performance problems the tooling names for you.

**Selector cost.** For most pages, selector matching is not your bottleneck; the browser matches right-to-left and is extremely good at it. Three cases where it genuinely matters:

1. **`:has()` on a broad anchor.** MDN's own performance note says to avoid anchoring on `body`, `:root` or `*`, and to constrain the inner selector with `>` or `+`. `body:has(.x .y .z)` can force re-evaluation of a very large subtree on every DOM mutation.
2. **Very large DOMs** (10,000+ nodes) with many rules — style recalculation time scales with elements × rules.
3. **Frequent class toggling high in the tree.** Toggling a class on `<html>` invalidates styles for everything below it. Scope theme toggles as tightly as the design allows, and prefer changing a custom property — a custom-property change on a registered, **non-inheriting** property (`@property … inherits: false`, [Part 11](./css-architecture.md#p11-mastery)) invalidates far less than a class toggle that could match anything.

**The honest ordering of effort.** Before micro-optimising CSS, check that you have done the things with an order of magnitude more impact: images sized and in modern formats, fonts preloaded with `font-display` set ([Part 8](./css-responsive-typography.md#p8-working)), `width`/`height` on images to prevent layout shift ([Part 7](./css-responsive-typography.md#p7-advanced)), and no render-blocking CSS you don't need. CSS performance work is worth doing *after* those, not instead of them.

**Try It — see the pipeline difference.**

```html
<style>
  .box { position: absolute; inline-size: 100px; block-size: 100px; background: crimson; }
  .layout { animation: move-left 2s infinite alternate linear; inset-block-start: 0; }
  .composite { animation: move-transform 2s infinite alternate linear; inset-block-start: 120px; }
  @keyframes move-left { to { inset-inline-start: 300px; } }
  @keyframes move-transform { to { translate: 300px 0; } }
</style>
<div class="box layout"></div>
<div class="box composite"></div>
```

**Expected:** both squares move identically. Record a Performance trace: the first square produces a purple *Layout* block on **every frame**; the second produces none. Then open Rendering → "Paint flashing" — the first square's area flashes green continuously, the second barely at all. Two visually identical animations, completely different cost.

### Part 14 cheat sheet

| Change | Pipeline cost |
|---|---|
| `transform`, `opacity` | Composite only — cheapest |
| `color`, `background-color`, `box-shadow`, `border-radius` | Paint + composite |
| `width`, `height`, `top`, `left`, `margin`, `padding`, `font-size` | **Layout** + paint + composite |

| Task | Declaration |
|---|---|
| Move something | `translate`, not `left`/`top` |
| Resize something (cheaply) | `scale` — watch for blurry text |
| Animate a shadow | Fade a pseudo-element's `opacity` |
| Isolate a repeated block | `contain: content` |
| Skip offscreen rendering | `content-visibility: auto` + `contain-intrinsic-size: auto 400px` |
| Prepare for an imminent change | `will-change`, added and removed via JS |
| Hide from render *and* a11y | `content-visibility: hidden` |

| Symptom in a trace | Likely cause |
|---|---|
| Wide purple *Layout* every frame | Animating a layout property |
| Sawtooth of tiny Layout blocks | Layout thrashing — batch reads then writes |
| Wide green *Paint* | Big blurred shadow, `filter`, `backdrop-filter`, large gradient |
| Many layers in the Layers panel | `will-change` overuse |
| Slow *Recalculate Style* | Broad `:has()`, huge DOM, class toggle on `<html>` |
| Scrollbar jumps while scrolling | `content-visibility` without an accurate `contain-intrinsic-size` |

| Outdated convention | Modern form |
|---|---|
| `transform: translateZ(0)` for "GPU acceleration" | `will-change`, applied narrowly and temporarily |
| `backface-visibility: hidden` as a rendering hack | Same |
| `will-change` on everything | Add before the change, remove after |

[↑ Back to top](#table-of-contents)

---

**You have reached the end of the set.** Go back to the [overview](./css-mastery-guide.md) for the [Quick Self-Check](./css-mastery-guide.md#9-quick-self-check) and the [cross-doc symptom index](./css-mastery-guide.md#10-cross-doc-cheat-sheet).
