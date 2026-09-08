# Modern CSS — Layout: Positioning, Flexbox & Grid (Parts 4–6)

Where boxes actually go. Positioning and the containing block, the stacking model that decides what covers what, and the two layout algorithms — flexbox for one dimension, grid for two — that replaced twenty years of workarounds.

> **Spec:** this doc follows the canonical spec in [`css-mastery-guide.md`](./css-mastery-guide.md#about-this-document). Assumes [Foundations Parts 1–3](./css-foundations.md) — the cascade, the box model, and formatting contexts. Set up [the playground file](./css-mastery-guide.md#3-setup-your-playground-file) before you start.
>
> **Reading order note:** if you are learning from scratch, do **Part 5 → Part 6 → Part 4**. Positioning makes far more sense after you have built something with flex and grid and hit your first `z-index` that "doesn't work".

---

## Table of Contents

- [Part 4 — Positioning & Stacking](#part-4--positioning--stacking)
  - [Beginner: the five `position` values](#p4-beginner)
  - [Working Knowledge: the containing block, and `inset`](#p4-working)
  - [Advanced: `sticky`, and the things that break it](#p4-advanced)
  - [Mastery: stacking contexts and `z-index`](#p4-mastery)
  - [Part 4 cheat sheet](#part-4-cheat-sheet)
- [Part 5 — Flexbox](#part-5--flexbox)
  - [Beginner: axes and the first flex container](#p5-beginner)
  - [Working Knowledge: alignment, `gap`, wrapping](#p5-working)
  - [Advanced: the `flex` shorthand decoded, and `min-width: auto`](#p5-advanced)
  - [Mastery: baseline alignment, `order`, and the limits of one dimension](#p5-mastery)
  - [Part 5 cheat sheet](#part-5-cheat-sheet)
- [Part 6 — Grid & Subgrid](#part-6--grid--subgrid)
  - [Beginner: tracks, `fr`, and `gap`](#p6-beginner)
  - [Working Knowledge: placement, named areas, `repeat()`](#p6-working)
  - [Advanced: `minmax()`, `auto-fill` vs. `auto-fit`, implicit tracks](#p6-advanced)
  - [Mastery: subgrid, and grid as a stacking tool](#p6-mastery)
  - [Part 6 cheat sheet](#part-6-cheat-sheet)

---

## Part 4 — Positioning & Stacking

<a id="p4-beginner"></a>

### Beginner: the five `position` values

`position` decides which box an element is positioned *relative to*, and whether it still takes up space in the flow.

| Value | Takes up space? | Positioned relative to | `inset` properties apply? |
|---|---|---|---|
| `static` (default) | Yes | Nothing — normal flow | No |
| `relative` | Yes (its original slot is kept) | Its own normal-flow position | Yes |
| `absolute` | **No** — removed from flow | Nearest positioned ancestor | Yes |
| `fixed` | **No** | The viewport | Yes |
| `sticky` | Yes | Scroll position, within its parent | Yes (as thresholds) |

The `inset` properties are `top`, `right`, `bottom`, `left` — or their logical equivalents `inset-block-start`, `inset-inline-end`, etc., plus the shorthands `inset`, `inset-block`, `inset-inline`. **They do nothing on a `static` element**, which is the single most common "my positioning isn't working" cause.

```css
/* WRONG - top has no effect, position is still static */
.badge { top: -8px; right: -8px; }

/* RIGHT */
.card   { position: relative; }   /* becomes the reference box */
.badge  { position: absolute; top: -8px; right: -8px; }
```

`position: relative` on the parent and `absolute` on the child is *the* fundamental pairing. The parent barely changes — it just volunteers as the reference box.

**Try It.**

```html
<style>
  .card { position: relative; width: 200px; height: 120px; background: #cfe; }
  .badge { position: absolute; inset-block-start: -8px; inset-inline-end: -8px;
           background: crimson; color: white; border-radius: 999px; padding: 2px 8px; }
</style>
<div class="card"><span class="badge">3</span></div>
```

**Expected:** the red badge overhangs the card's top-right corner. Delete `position: relative` from `.card` and reload — the badge jumps to the top-right of the *page*, because it fell back to the next positioned ancestor (here, the initial containing block). That jump is the containing block made visible.

<a id="p4-working"></a>

### Working Knowledge: the containing block, and `inset`

The **containing block** is the rectangle an element's percentages and offsets resolve against. Which rectangle it is depends on `position` ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_display/Containing_block)):

| Element's `position` | Containing block |
|---|---|
| `static`, `relative`, `sticky` | The **content box** of the nearest block-container ancestor (or the nearest ancestor establishing a formatting context) |
| `absolute` | The **padding box** of the nearest ancestor whose `position` is not `static` |
| `fixed` | The **viewport** |

Note the content-box/padding-box difference: an absolutely positioned child with `inset: 0` fills its positioned ancestor *including* that ancestor's padding.

The root of the chain is the **initial containing block** — a viewport-sized rectangle anchored at the document origin. An `absolute` element with no positioned ancestor lands there, which is what the Try It above demonstrated.

Handy `inset` combinations:

```css
.overlay   { position: absolute; inset: 0; }              /* fill the parent */
.centred   { position: absolute; inset: 0; margin: auto; width: 200px; height: 100px; }
.pinned    { position: fixed; inset-block-end: 1rem; inset-inline-end: 1rem; }
```

The `inset: 0` + `margin: auto` + explicit size trick centres an absolutely positioned box in both axes. It still works and is worth knowing, but for anything not absolutely positioned, `display: grid; place-items: center` ([Part 6](#p6-working)) is the modern answer.

**Outdated convention:** the `position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%)` centring hack is still everywhere in blog posts. It works, but it is only necessary when you cannot know the element's size. For a known size use `inset` + `margin: auto`; for anything in normal flow use grid or flex centring. Reaching for the transform hack by default also silently creates a stacking context and a containing block (see [Mastery](#p4-mastery)), which causes bugs elsewhere.

<a id="p4-advanced"></a>

### Advanced: `sticky`, and the things that break it

`position: sticky` is `relative` until the element hits a scroll threshold you define, then behaves like `fixed` — but only within its parent's box.

```css
.section-heading {
  position: sticky;
  inset-block-start: 0;   /* the threshold: stick when it reaches the top */
  background: white;      /* essential - otherwise content scrolls through it */
}
```

Three rules govern it, and violating any of them makes sticky do nothing at all — silently, with no error:

1. **You must specify a threshold.** `position: sticky` with no `top`/`bottom`/`inset-*` never sticks. There is no default.
2. **The parent must be taller than the sticky element.** A sticky element sticks *within its parent*; if the parent is exactly as tall as the element, there is no travel distance and nothing appears to happen. This is the most common cause, and the hardest to see.
3. **No ancestor may have `overflow` other than `visible`.** As [Foundations Part 3](./css-foundations.md#p3-advanced) explains, `overflow: hidden`, `auto`, `scroll` or `clip` on *any* ancestor creates a scroll container, and the sticky element then sticks relative to that container instead of the viewport — usually meaning it scrolls away as normal.

```css
/* WRONG - a common "fix" elsewhere in the stylesheet that kills sticky */
.page-wrapper { overflow-x: hidden; }   /* often added to stop horizontal scroll */

/* RIGHT - clip only the axis you mean, without creating a scroll container */
.page-wrapper { overflow-x: clip; }
```

**Real Scenario.** A sticky table header stops working after someone adds `overflow: hidden` to a distant wrapper to contain a float. Nothing errors, nothing looks broken in the Styles pane, and the sticky rule is still listed as applied. The diagnostic: in DevTools, walk up the ancestor chain in the Computed pane checking `overflow` on each one. Chrome DevTools also badges sticky elements in the Elements panel and will tell you when a sticky element has no ancestor it can stick within.

<a id="p4-mastery"></a>

### Mastery: stacking contexts and `z-index`

When boxes overlap, paint order decides what you see. Within a **stacking context**, elements paint in this order: the context's own background and borders, then negative-`z-index` children, then in-flow block boxes, then floats, then inline content, then `z-index: 0`/`auto` positioned boxes, then positive-`z-index` children in ascending order.

The critical property: **a stacking context is atomic**. Once an element creates one, its children's `z-index` values are only compared *with each other*. A child with `z-index: 99999` inside a parent with `z-index: 1` still paints below a sibling of the parent with `z-index: 2`.

```
  root stacking context
   |
   +-- .header   z-index: 10   [creates a context]
   |     |
   |     +-- .dropdown  z-index: 99999   <- can NEVER escape .header's slot
   |
   +-- .modal    z-index: 20   [creates a context]

  Result: .modal paints over .dropdown, whatever number you write.
```

That is the answer to "`z-index: 9999` doesn't work". The fix is never a bigger number — it is to find the ancestor creating the trapping context and either raise *it*, or move the element out (a `<dialog>` in the top layer, or a portal).

**What creates a stacking context** — the list is longer than most people expect ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_positioned_layout/Stacking_context)):

- The root `<html>` element.
- `position: relative` or `absolute` **with a `z-index` other than `auto`**.
- `position: fixed` or `sticky` — **always**, with or without `z-index`.
- A flex or grid **item** with a `z-index` other than `auto`.
- `opacity` less than `1`.
- `transform`, `scale`, `rotate`, `translate`, `perspective`, `filter`, `backdrop-filter`, `clip-path`, `mask` — any of these set to something other than `none`.
- `mix-blend-mode` other than `normal`.
- `isolation: isolate`.
- `will-change` naming any of the above properties.
- `contain: layout`, `contain: paint`, or a composite value including them — **which includes `container-type: size` and `inline-size`**.
- Top-layer elements (`<dialog>` shown modally, popovers, fullscreen).

Two of these bite regularly. **`opacity: 0.99`** is a genuine, deliberate hack you will see in old code to force a stacking context — `isolation: isolate` is the modern, explicit way to do the same thing with no visual side effect. And **`container-type: inline-size`** ([Part 7](./css-responsive-typography.md)) creates one as a side effect, which is why adding container queries to a component can move a tooltip behind something.

The same list overlaps heavily with the list of things that make an element a **containing block for `fixed` descendants** ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_display/Containing_block)): `transform`, `filter`, `backdrop-filter`, `perspective`, `contain: layout|paint|strict|content`, `content-visibility: auto`, and `will-change` naming those. This produces the most disorienting bug in CSS:

**Real Scenario.** A `position: fixed` modal works perfectly until someone adds a CSS animation to a wrapping element. The animation uses `transform`, the transform makes the wrapper a containing block for fixed descendants, and the modal — still `position: fixed` — now scrolls with the page. The Styles pane shows `position: fixed` applied and unstruck. Nothing looks wrong. The diagnostic is to search ancestors for a non-`none` `transform`, `filter`, or `will-change`.

**Try It — trap and escape.**

```html
<style>
  .row { display: flex; gap: 2rem; }
  .parent { width: 120px; height: 80px; background: #cfe; }
  .a { position: relative; z-index: 1; }
  .b { position: relative; z-index: 2; }
  .child { position: absolute; inset-block-start: 40px; inset-inline-start: 40px;
           width: 120px; height: 80px; background: crimson; z-index: 99999; }
</style>
<div class="row">
  <div class="parent a"><div class="child">z-index 99999</div></div>
  <div class="parent b">z-index 2</div>
</div>
```

**Expected:** the blue `z-index: 2` box paints **over** the crimson `z-index: 99999` box. Now change `.a` to `z-index: 3` and reload — the crimson box wins, without touching its own `z-index` at all. Then remove `z-index` from `.a` entirely: with no stacking context created, the child's 99999 finally applies against the root.

### Part 4 cheat sheet

| Symptom | Cause | Fix |
|---|---|---|
| `top`/`left` do nothing | Element is `position: static` | Set `relative`/`absolute`/`fixed` |
| Absolute child positions off the page | No positioned ancestor | `position: relative` on the intended parent |
| `position: fixed` scrolls with the page | Ancestor has `transform`/`filter`/`will-change`/`contain` | Remove it, or move the element to the top layer |
| `position: sticky` does nothing | No threshold set | Add `inset-block-start: 0` |
| `position: sticky` does nothing | Parent is the same height as the element | Give the parent real height |
| `position: sticky` does nothing | Ancestor has `overflow` != `visible` | Use `overflow-x: clip`, or remove it |
| `z-index: 9999` ignored | Trapped in an ancestor's stacking context | Raise the ancestor, or escape the context |
| Need a stacking context, no visual change | — | `isolation: isolate` (not `opacity: .99`) |

| Containing block | Is... |
|---|---|
| `static`/`relative`/`sticky` | Nearest block container's **content box** |
| `absolute` | Nearest non-static ancestor's **padding box** |
| `fixed` | The viewport — unless an ancestor has `transform`/`filter`/`contain`/`will-change` |

[↑ Back to top](#table-of-contents)

---

## Part 5 — Flexbox

<a id="p5-beginner"></a>

### Beginner: axes and the first flex container

Flexbox lays children out along **one** axis and gives you fine control over how leftover space is distributed. `display: flex` on a parent turns its direct children into flex items.

```
  flex-direction: row (default)

   main-start                                    main-end
      |                                             |
      v                                             v
      +--------+  +--------+  +--------+            |   ^ cross-start
      | item 1 |  | item 2 |  | item 3 |            |   |
      +--------+  +--------+  +--------+            |   v cross-end
      <------------ main axis ------------------------>

  flex-direction: column  ->  the two axes swap roles.
```

Everything in flexbox is described in terms of **main axis** and **cross axis**, never left/right/top/bottom — which is exactly why it works unchanged in right-to-left languages.

```css
.toolbar {
  display: flex;          /* children become flex items */
  gap: 1rem;              /* space between them - use this, not margins */
  align-items: center;    /* centre on the cross axis */
}
```

Three default behaviours to internalise, because they explain most first-time surprises:

1. Items line up in a **row** and **do not wrap**.
2. Items **do not grow** to fill spare space, but they **do shrink** when there isn't enough.
3. Items **stretch** to fill the cross axis — which is why flex children in a row all end up the same height for free.

**Try It.**

```html
<style>
  .bar { display: flex; gap: 1rem; background: #eee; padding: 1rem; }
  .bar > * { background: #cfe; padding: .5rem 1rem; }
</style>
<div class="bar"><div>one</div><div>two<br>lines</div><div>three</div></div>
```

**Expected:** three boxes in a row, all the same height as the tallest (the two-line one) — that is `align-items: stretch` by default. Add `align-items: start` and they shrink to their own content height.

<a id="p5-working"></a>

### Working Knowledge: alignment, `gap`, wrapping

The alignment properties are the everyday surface. The naming is consistent once you see it: **`justify-*` acts on the main axis, `align-*` on the cross axis.**

| Property | Axis | Applies to |
|---|---|---|
| `justify-content` | Main | Distributing spare space between items |
| `align-items` | Cross | All items, as a default |
| `align-self` | Cross | One item, overriding `align-items` |
| `align-content` | Cross | The *lines*, only when wrapped |

```css
.header {
  display: flex;
  justify-content: space-between;   /* logo left, nav right */
  align-items: center;
}
```

`justify-content` values: `flex-start` (default), `flex-end`, `center`, `space-between` (first and last flush to the edges), `space-around` (half-size gaps at the ends), `space-evenly` (equal gaps everywhere). `align-items` values: `stretch` (default), `flex-start`, `flex-end`, `center`, `baseline`.

**`gap` for spacing, always.** Flexbox `gap` is widely available and removes the old margin dance entirely:

```css
/* Outdated convention - still all over older tutorials */
.list > * { margin-right: 1rem; }
.list > *:last-child { margin-right: 0; }

/* Modern */
.list { display: flex; gap: 1rem; }
```

**Wrapping.** `flex-wrap: wrap` lets items flow onto new lines when they run out of room. Combine with `flex-basis` for a responsive row that needs no media queries at all:

```css
.cards {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
}
.cards > * {
  flex: 1 1 18rem;   /* grow, shrink, and prefer to be 18rem wide */
}
```

That single rule gives you 3-up on a wide screen, 2-up on a tablet, 1-up on a phone. It is the reason "flexbox with a sensible `flex-basis`" is often more responsive than a grid with explicit breakpoints.

One consequence to know: `align-items` centres items *within their line*; `align-content` positions the lines themselves. If your wrapped rows look oddly spread out, you want `align-content: flex-start`.

**Try It.**

```html
<style>
  .cards { display: flex; flex-wrap: wrap; gap: 1rem; }
  .cards > * { flex: 1 1 18rem; background: #cfe; padding: 1rem; }
</style>
<div class="cards"><div>A</div><div>B</div><div>C</div><div>D</div><div>E</div></div>
```

**Expected:** resize the window slowly. The cards reflow from 5-across to 4, 3, 2, 1 — and the last row's items stretch to fill the width rather than leaving a ragged gap. That stretching is `flex-grow: 1` at work, and it is the visible difference from a grid, which would keep the tracks aligned.

<a id="p5-advanced"></a>

### Advanced: the `flex` shorthand decoded, and `min-width: auto`

`flex` is a shorthand for three longhands, and the shorthand's *defaults for omitted values differ from the longhand defaults* — which is why `flex: 1` and `flex-grow: 1` are not the same thing.

```
  flex: <flex-grow> <flex-shrink> <flex-basis>
          how much     how much     starting size
          of spare     to shrink    before growing
          space         when         or shrinking
          to take       cramped
```

| You write | Expands to | Behaviour |
|---|---|---|
| `flex: 1` | `1 1 0%` | Grow, shrink, **ignore content size** — all items end up equal width |
| `flex: auto` | `1 1 auto` | Grow, shrink, **start from content size** — bigger content gets more room |
| `flex: initial` (the default) | `0 1 auto` | Don't grow, do shrink |
| `flex: none` | `0 0 auto` | Rigid — never grows, never shrinks |
| `flex: 1 1 18rem` | as written | Grow and shrink around an 18rem preference |

The `flex: 1` vs. `flex: auto` distinction is worth a Try It on its own, because choosing the wrong one is behind "why are my columns uneven?".

```html
<style>
  .row { display: flex; gap: 1rem; margin-block-end: 1rem; }
  .row > * { background: #cfe; padding: .5rem; }
  .equal > * { flex: 1; }
  .content > * { flex: auto; }
</style>
<div class="row equal"><div>short</div><div>a much much longer item</div></div>
<div class="row content"><div>short</div><div>a much much longer item</div></div>
```

**Expected:** the first row has two exactly equal columns (basis `0`, so content length is ignored). The second row gives the longer text more space (basis `auto`, so content length is the starting point). Same flex container, opposite result.

**`flex-basis` vs. `width`.** On the main axis, `flex-basis` wins over `width`. Setting both is not an error, but the `width` is only a fallback for when `flex-basis: auto`. Prefer `flex-basis` (usually via the `flex` shorthand) inside flex containers.

**The `min-width: auto` gotcha — the most important thing in this Part.** Flex items have an *automatic minimum size*: they refuse to shrink below their `min-content` size. This is deliberate (it stops text disappearing), but it means a flex item containing a long unbroken string, a `<pre>` block, or a nested overflow container will blow out of the container instead of shrinking:

```css
/* WRONG - .content overflows the row when it holds a long URL or a wide table */
.row { display: flex; }
.content { flex: 1; }

/* RIGHT - opt out of the automatic minimum size */
.content { flex: 1; min-width: 0; }
```

`min-width: 0` (or `overflow: hidden`, which has the same effect via a different mechanism) is the fix. In a `column` flex container, the equivalent is `min-height: 0`. Memorise this one — it will save you hours.

**Try It.**

```html
<style>
  .row { display: flex; gap: 1rem; width: 400px; border: 2px solid crimson; }
  .row > * { background: #cfe; flex: 1; }
  .fix { min-width: 0; }
  .fix p { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
</style>
<div class="row"><div><p>https://example.com/a/very/long/url/that/never/breaks/anywhere</p></div><div>sidebar</div></div>
<div class="row"><div class="fix"><p>https://example.com/a/very/long/url/that/never/breaks/anywhere</p></div><div>sidebar</div></div>
```

**Expected:** the first row bursts through its red border. The second stays inside it and truncates with an ellipsis. The only difference is `min-width: 0`.

<a id="p5-mastery"></a>

### Mastery: baseline alignment, `order`, and the limits of one dimension

**Baseline alignment** aligns items by their first line of text rather than their box edges — the right choice for a row of labels with different font sizes:

```css
.meta { display: flex; align-items: baseline; gap: .5rem; }
```

`align-items: last baseline` aligns by the *last* line instead, which is what you want when boxes have differing numbers of lines and their bottoms should line up.

**`order`** re-sequences items visually without touching the DOM:

```css
.item--featured { order: -1; }   /* jump to the front */
```

It works, but treat it as a hazard. **`order` and `flex-direction: row-reverse` change visual order only — keyboard tab order and screen-reader reading order still follow the DOM.** A user tabbing through a visually-reordered toolbar jumps around unpredictably. Use `order` for genuinely presentational reordering at a breakpoint; if the *meaning* of the order changed, change the DOM.

**`margin: auto` inside flex** absorbs free space in that direction — a neat trick that predates `justify-content` having enough values, and still the cleanest way to push one item away from the rest:

```css
.nav { display: flex; gap: 1rem; }
.nav .profile { margin-inline-start: auto; }   /* everything else left, this one right */
```

**The honest limit of flexbox: it is one-dimensional.** Items know about their line; they do not know about items on other lines. Consequences:

- A wrapped flex layout **cannot align items into columns across rows** — each line sizes independently. If a wrapped "grid" of cards looks ragged, you want [Grid](#part-6--grid--subgrid), not more flex properties.
- There is no flex equivalent of `subgrid`; making three cards' titles, bodies and buttons line up across the row needs grid ([Part 6 Mastery](#p6-mastery)).

The practical rule: **flexbox when content should determine the sizes along one axis (toolbars, button rows, a label-and-value pair, anything that should wrap naturally); grid when the layout determines the sizes in two axes (page skeletons, card decks that must align, forms).** Nesting them is normal and correct — a grid page skeleton with a flex toolbar inside one cell is a completely ordinary structure.

### Part 5 cheat sheet

| Shorthand | Longhand | Use for |
|---|---|---|
| `flex: 1` | `1 1 0%` | Equal columns regardless of content |
| `flex: auto` | `1 1 auto` | Columns sized by content, sharing spare space |
| `flex: initial` | `0 1 auto` | Default: shrink but don't grow |
| `flex: none` | `0 0 auto` | Rigid item (icons, fixed sidebars) |
| `flex: 1 1 18rem` | as written | Responsive wrap without media queries |

| Property | Axis | Note |
|---|---|---|
| `justify-content` | Main | `space-between` for header layouts |
| `align-items` | Cross | Default `stretch` gives equal-height rows |
| `align-self` | Cross | Per-item override |
| `align-content` | Cross | Only does something when wrapped |
| `gap` | Both | Replaces margin hacks entirely |

| Symptom | Fix |
|---|---|
| Item overflows instead of shrinking | `min-width: 0` (or `min-height: 0` in a column) |
| Columns uneven when they should match | `flex: 1`, not `flex: auto` |
| Wrapped rows won't align into columns | Use grid — flex can't do this |
| One item to the far end | `margin-inline-start: auto` |
| Different font sizes look misaligned | `align-items: baseline` |
| Tab order jumps around | You used `order` or `row-reverse` — reorder the DOM instead |

[↑ Back to top](#table-of-contents)

---

## Part 6 — Grid & Subgrid

<a id="p6-beginner"></a>

### Beginner: tracks, `fr`, and `gap`

Grid lays out in **two** dimensions at once. You define **tracks** (columns and rows), and items land in the **cells** they form.

```css
.layout {
  display: grid;
  grid-template-columns: 200px 1fr;   /* two columns: fixed, then flexible */
  gap: 1rem;
}
```

The **`fr` unit** is grid's headline feature: one fraction of the *leftover* space after fixed sizes and gaps are subtracted. `1fr 1fr` is two equal columns; `1fr 2fr` gives the second twice the leftover. Because it is computed after gaps, `fr` does not have the "percentages plus gaps overflow the container" problem that the old float and percentage layouts had.

```
  grid-template-columns: 200px 1fr 1fr;   gap: 16px

   line 1        line 2        line 3        line 4
     |             |             |             |
     v             v             v             v
     +-----------+ +-----------+ +-----------+
     |   200px   | |    1fr    | |    1fr    |
     +-----------+ +-----------+ +-----------+
                 ^             ^
               16px gap     16px gap
```

Note the numbering: **grid lines** are numbered from 1, and there is always one more line than there are tracks. Placement is expressed in lines, not tracks — this is the piece that clicks late for most people.

**Try It.**

```html
<style>
  .layout { display: grid; grid-template-columns: 200px 1fr; gap: 1rem; }
  .layout > * { background: #cfe; padding: 1rem; }
</style>
<div class="layout"><aside>sidebar</aside><main>main content</main></div>
```

**Expected:** a fixed 200px sidebar and a main area that absorbs everything else. Resize the window: the sidebar stays put, the main column flexes. Change `1fr` to `2fr` — nothing changes, because there is only one flexible track and it already takes all the leftover space. That is `fr` being a *ratio*, not a size.

<a id="p6-working"></a>

### Working Knowledge: placement, named areas, `repeat()`

**Line-based placement** puts an item into specific tracks:

```css
.hero {
  grid-column: 1 / 3;    /* from line 1 to line 3 - spans two columns */
  grid-row: 1 / 2;
}
.wide { grid-column: 1 / -1; }   /* -1 is the last line: "span everything" */
.next { grid-column: span 2; }   /* span two tracks from wherever I land */
```

`grid-column: 1 / -1` is the single most useful placement idiom — "full bleed across the grid, whatever the track count".

**Named areas** are the most readable way to express a page skeleton, because the CSS looks like the layout:

```css
.page {
  display: grid;
  grid-template-columns: 200px 1fr;
  grid-template-rows: auto 1fr auto;
  grid-template-areas:
    "header  header"
    "sidebar main"
    "footer  footer";
  min-block-size: 100dvh;
  gap: 1rem;
}
header { grid-area: header; }
aside  { grid-area: sidebar; }
main   { grid-area: main; }
footer { grid-area: footer; }
```

Rearranging the layout at a breakpoint then means rewriting only the ASCII art:

```css
@media (width < 40em) {
  .page {
    grid-template-columns: 1fr;
    grid-template-areas: "header" "main" "sidebar" "footer";
  }
}
```

Note that this *does* change visual order relative to the DOM — the same accessibility caution as flexbox `order` applies. Here the DOM order (header, sidebar, main, footer) and the mobile visual order (header, main, sidebar, footer) differ, so decide deliberately which one is correct for a keyboard user, and put the DOM in that order.

**`repeat()`** removes the repetition:

```css
grid-template-columns: repeat(3, 1fr);              /* 1fr 1fr 1fr */
grid-template-columns: 20px repeat(6, 1fr) 20px;    /* patterns are fine */
```

**Centring, finally solved.** `place-items` is the shorthand for `align-items` + `justify-items`, and on a grid container it centres children in both axes:

```css
.centre { display: grid; place-items: center; min-block-size: 100dvh; }
```

Two declarations, no absolute positioning, no transform hack, works at any content size.

<a id="p6-advanced"></a>

### Advanced: `minmax()`, `auto-fill` vs. `auto-fit`, implicit tracks

**`minmax(min, max)`** sizes a track between two bounds — the building block of responsive grids:

```css
grid-auto-rows: minmax(100px, auto);   /* at least 100px, taller if content needs it */
grid-template-columns: minmax(0, 1fr) 300px;
```

`minmax(0, 1fr)` deserves its own note. A bare `1fr` is actually shorthand for `minmax(auto, 1fr)`, and that `auto` minimum is the same automatic-minimum-size behaviour that bites in flexbox ([Part 5 Advanced](#p5-advanced)): a track containing a long unbreakable string or a wide table will refuse to shrink and will blow out the grid. `minmax(0, 1fr)` opts out.

```css
/* WRONG - a long URL or wide <pre> in the main column overflows the page */
.layout { grid-template-columns: 200px 1fr; }

/* RIGHT */
.layout { grid-template-columns: 200px minmax(0, 1fr); }
```

**`auto-fill` vs. `auto-fit`** — the responsive-grid idiom, and the one difference people never remember:

```css
.cards {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(16rem, 1fr));
  gap: 1rem;
}
```

Both create as many tracks as fit. The difference appears **only when there are fewer items than tracks**:

- **`auto-fill`** keeps the empty tracks. Items stay at their natural size; you get trailing empty columns.
- **`auto-fit`** collapses the empty tracks to zero width, so the remaining items stretch to fill the row.

```
  Container fits 4 tracks, but there are only 2 items:

  auto-fill:  [ item ][ item ][ empty ][ empty ]
  auto-fit:   [    item      ][      item      ]
```

Choose `auto-fit` when a half-empty row should still look filled (card decks), and `auto-fill` when the column rhythm must stay stable regardless of item count (a calendar, a product grid where alignment across pages matters).

**Implicit tracks.** Items placed outside your explicit grid — or auto-placed after the explicit tracks run out — create *implicit* tracks. Size them with `grid-auto-rows` / `grid-auto-columns`, and control the placement direction with `grid-auto-flow`:

```css
.gallery {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(12rem, 1fr));
  grid-auto-rows: 12rem;      /* every implicit row is 12rem tall */
  grid-auto-flow: row dense;  /* backfill holes left by spanning items */
}
```

`dense` packing is genuinely useful for masonry-ish galleries, with the same warning as `order`: it decouples visual order from DOM order.

**Try It.**

```html
<style>
  .cards { display: grid; gap: 1rem; }
  .fill { grid-template-columns: repeat(auto-fill, minmax(10rem, 1fr)); }
  .fit  { grid-template-columns: repeat(auto-fit,  minmax(10rem, 1fr)); }
  .cards > * { background: #cfe; padding: 1rem; }
</style>
<div class="cards fill"><div>A</div><div>B</div></div>
<hr>
<div class="cards fit"><div>A</div><div>B</div></div>
```

**Expected:** on a wide window, the first pair stays narrow with empty space to the right; the second pair stretches to fill the whole width. Narrow the window until only two tracks fit and the two rows become identical — proving the difference only exists when tracks would otherwise be empty.

<a id="p6-mastery"></a>

### Mastery: subgrid, and grid as a stacking tool

**Subgrid** solves the problem flexbox structurally cannot: making the *internals* of sibling components line up with each other. Widely available since September 2023 ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_grid_layout/Subgrid)).

Normally a nested grid creates its own independent tracks, so three cards each size their own title/body/button rows — and if one title wraps to two lines, that card's body starts lower than its neighbours'. `grid-template-rows: subgrid` makes the child adopt the *parent's* tracks instead:

```css
.card-deck {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(16rem, 1fr));
  grid-template-rows: auto 1fr auto;   /* title / body / footer, defined once */
  gap: 1rem;
}
.card {
  display: grid;
  grid-row: span 3;              /* the card occupies all three parent rows */
  grid-template-rows: subgrid;   /* and its children use the parent's row lines */
}
```

Now every card's title occupies the deck's row 1, every body row 2, every footer row 3 — so titles, bodies and buttons align across the whole deck no matter how the content varies.

Details worth knowing: gaps are inherited from the parent but can be overridden on the subgrid (`row-gap: 0`); the parent's **named grid lines pass through** into the subgrid; line numbering restarts at 1 inside the subgrid; and a subgridded axis creates **no implicit tracks**, so an item that doesn't fit has nowhere to go.

**Grid as a stacking tool.** Two or more items placed in the *same* cell overlap — which makes grid a cleaner alternative to absolute positioning for overlays:

```css
.hero { display: grid; }
.hero > * { grid-area: 1 / 1; }    /* everything in the same cell */
.hero .caption { place-self: end start; padding: 1rem; }
```

No `position: absolute`, no containing-block side effects, and — critically — the grid still sizes itself to the *tallest* item, which absolute positioning cannot do because absolutely positioned elements are out of flow.

**Try It — subgrid alignment.**

```html
<style>
  .deck { display: grid; grid-template-columns: repeat(3, 1fr);
          grid-template-rows: auto 1fr auto; gap: 1rem; }
  .card { display: grid; grid-row: span 3; grid-template-rows: subgrid;
          background: #cfe; padding: 1rem; }
  .card > footer { background: gold; }
</style>
<div class="deck">
  <article class="card"><h3>Short</h3><p>Body.</p><footer>Read</footer></article>
  <article class="card"><h3>A considerably longer title that wraps</h3><p>Body.</p><footer>Read</footer></article>
  <article class="card"><h3>Mid length title</h3><p>Longer body text here that pushes down.</p><footer>Read</footer></article>
</div>
```

**Expected:** all three gold footers sit at exactly the same height, and all three bodies start at the same height, despite the wrapping title. Delete `grid-template-rows: subgrid` and reload — the footers still align (because the cards stretch), but the *bodies* no longer do. That inner alignment is what subgrid buys you and what no amount of flexbox can.

### Part 6 cheat sheet

| Task | Declaration |
|---|---|
| Two columns, fixed + flexible | `grid-template-columns: 200px minmax(0, 1fr)` |
| N equal columns | `grid-template-columns: repeat(3, 1fr)` |
| Responsive card deck, fills the row | `repeat(auto-fit, minmax(16rem, 1fr))` |
| Responsive deck, stable columns | `repeat(auto-fill, minmax(16rem, 1fr))` |
| Full-bleed row | `grid-column: 1 / -1` |
| Span two tracks | `grid-column: span 2` |
| Page skeleton | `grid-template-areas` + `grid-area` |
| Centre anything | `display: grid; place-items: center` |
| Size auto-created rows | `grid-auto-rows: minmax(100px, auto)` |
| Backfill gaps | `grid-auto-flow: row dense` |
| Align card internals across a deck | `grid-template-rows: subgrid` |
| Overlay elements | `.parent > * { grid-area: 1 / 1 }` |

| Gotcha | Fix |
|---|---|
| Long content overflows a `1fr` track | `minmax(0, 1fr)` — a bare `1fr` is `minmax(auto, 1fr)` |
| Empty trailing columns | `auto-fit` instead of `auto-fill` |
| Nested card internals don't align | `subgrid` on the row axis |
| Visual order != DOM order | `grid-template-areas` and `dense` both do this — check tab order |

### Flex or grid?

| Situation | Use |
|---|---|
| Toolbar, button row, tag list | Flex |
| Content should determine sizes along one axis | Flex |
| Should wrap naturally with no breakpoints | Flex (`flex: 1 1 <basis>`) |
| Page skeleton, form layout | Grid |
| Items must align in both axes | Grid |
| Card deck whose internals must align | Grid + subgrid |
| Overlapping layers sized by the tallest | Grid (same cell) |
| Both, nested | Extremely normal |

[↑ Back to top](#table-of-contents)

---

**Next:** [Responsive Design & Typography (Parts 7–8)](./css-responsive-typography.md) — units, container queries, and making all of this adapt.
