# Modern CSS — Colour & Visual Effects (Parts 9–10)

Colour is the part of CSS that changed most in the last three years and that most stylesheets have not caught up with. This doc covers why `hsl()` lies to you, what the perceptual colour spaces fix, how to build a palette that survives dark mode — and then everything you can paint into a box: backgrounds, gradients, shadows, filters, blend modes, clipping and masking.

> **Spec:** this doc follows the canonical spec in [`css-mastery-guide.md`](./css-mastery-guide.md#about-this-document). Assumes [Foundations](./css-foundations.md).

---

## Table of Contents

- [Part 9 — Colour in Modern CSS](#part-9--colour-in-modern-css)
  - [Beginner: the colour notations](#p9-beginner)
  - [Working Knowledge: dark mode with `color-scheme` and `light-dark()`](#p9-working)
  - [Advanced: why `hsl()` lightness lies, and `oklch()`](#p9-advanced)
  - [Mastery: `color-mix()`, relative colour, and generated palettes](#p9-mastery)
  - [Part 9 cheat sheet](#part-9-cheat-sheet)
- [Part 10 — Backgrounds, Borders & Effects](#part-10--backgrounds-borders--effects)
  - [Beginner: backgrounds and borders](#p10-beginner)
  - [Working Knowledge: gradients and shadows](#p10-working)
  - [Advanced: `filter`, `backdrop-filter`, blend modes](#p10-advanced)
  - [Mastery: clipping, masking, and forced-colors mode](#p10-mastery)
  - [Part 10 cheat sheet](#part-10-cheat-sheet)

---

## Part 9 — Colour in Modern CSS

<a id="p9-beginner"></a>

### Beginner: the colour notations

Every one of these is the same red:

```css
color: red;                    /* named colour - 148 of them exist */
color: #ff0000;                /* hex */
color: #f00;                   /* short hex */
color: #ff0000cc;              /* hex with alpha */
color: rgb(255 0 0);           /* modern space-separated syntax */
color: rgb(255 0 0 / 80%);     /* alpha after a slash */
color: hsl(0 100% 50%);        /* hue, saturation, lightness */
```

**Note the modern syntax:** spaces between channels, and a `/` before alpha. The old comma form (`rgba(255, 0, 0, 0.8)`) still works and always will, but `rgb()` and `rgba()` are now the same function, and the space form is what every new colour function uses — `oklch()`, `color()`, `lab()` — so switching now means one syntax to remember rather than two.

Two special keywords worth knowing early:

- **`currentColor`** resolves to the element's own `color` value. It is the single most useful colour keyword: an icon or border declared `border-color: currentColor` follows the text colour automatically through every theme, state and override.
- **`transparent`** is `rgb(0 0 0 / 0)` — fully transparent *black*, which matters in gradients (see [Part 10](#p10-working)).

```css
/* An icon button that just works in every context */
.icon-button { color: inherit; }
.icon-button svg { fill: currentColor; }
```

<a id="p9-working"></a>

### Working Knowledge: dark mode with `color-scheme` and `light-dark()`

Dark mode has two halves, and most implementations only do the first.

**Half one: your own colours.** The traditional approach is a media query flipping custom properties:

```css
:root {
  --bg: #ffffff;
  --fg: #1a1a1a;
}
@media (prefers-color-scheme: dark) {
  :root { --bg: #131313; --fg: #ececec; }
}
```

**Half two: the browser's colours.** Form controls, scrollbars, the canvas behind your page, `<input>` spinners, default focus rings and `::selection` are painted by the browser, and it only makes them dark if you tell it the page supports dark mode:

```css
:root { color-scheme: light dark; }
```

Skip that one line and you get a dark page with bright white form fields and a white scrollbar. It is the most commonly missed declaration in dark-mode implementations.

**`light-dark()`** then collapses the two-branch pattern into one value ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/light-dark), Baseline since May 2024). It **requires** `color-scheme` to be set:

```css
:root {
  color-scheme: light dark;
  --bg: light-dark(#ffffff, #131313);
  --fg: light-dark(#1a1a1a, #ececec);
  --accent: light-dark(oklch(55% 0.18 265), oklch(72% 0.14 265));
}
body { background: var(--bg); color: var(--fg); }
```

No media query, no duplicated block, and the values sit next to each other where you can compare them. Because `color-scheme` can be set per-element, you can also force a subtree light or dark:

```css
.always-dark { color-scheme: dark; }   /* light-dark() inside now picks the dark value */
```

**Try It.**

```html
<style>
  :root { color-scheme: light dark; }
  body { background: light-dark(#fff, #131313); color: light-dark(#1a1a1a, #ececec); }
  input { font: inherit; }
</style>
<p>Some text.</p><input placeholder="type here">
```

**Expected:** in DevTools → Rendering → "Emulate prefers-color-scheme: dark", both the page **and** the input flip to dark, and the scrollbar with them. Now delete `color-scheme: light dark` and repeat: the page goes dark but the input stays bright white. That contrast is exactly what the property is for.

<a id="p9-advanced"></a>

### Advanced: why `hsl()` lightness lies, and `oklch()`

`hsl()` looks like it should be perceptual — hue, saturation, lightness — but it is a trivial mathematical transform of sRGB, and **its "lightness" has almost nothing to do with how light a colour looks.**

```css
.yellow { background: hsl(60 100% 50%); }   /* blindingly bright */
.blue   { background: hsl(240 100% 50%); }  /* very dark */
```

Both are "50% lightness". One is nearly white, the other nearly black. That is why a palette built by stepping `hsl()` lightness produces a scale where some hues look washed out and others look muddy, and why a `hsl()`-based dark theme requires per-colour hand-tuning.

**`oklch()`** fixes this ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/oklch), widely available since May 2023). It expresses colour in the Oklab space, which is *perceptually uniform*: equal numeric changes look like equal visual changes.

```css
color: oklch(L C H);
color: oklch(70% 0.15 250);
color: oklch(70% 0.15 250 / 50%);
```

| Channel | Range | Meaning |
|---|---|---|
| **L** — lightness | `0`–`1`, or `0%`–`100%` | *Perceived* brightness. `0` is black, `1` is white. |
| **C** — chroma | `0` to about `0.4` | Colourfulness. `0` is grey. There is no fixed maximum — it depends on hue and lightness. |
| **H** — hue | `0`–`360` | Angle. Note that hues **do not match `hsl()`**: red is around 29°, not 0°. |

The payoff is that `oklch(70% 0.15 H)` produces colours of genuinely equal apparent lightness at every hue. That is what makes a systematic palette possible:

```css
:root {
  --blue-500:   oklch(60% 0.16 255);
  --green-500:  oklch(60% 0.16 150);
  --red-500:    oklch(60% 0.16 25);
  /* All three read as the same weight. Try that with hsl(). */
}
```

**Chroma is the one that needs care.** Unlike `hsl()` saturation, chroma has no natural ceiling, and high values fall outside the sRGB gamut — the browser will clamp them, which can flatten several nominally different colours into the same rendered one. Keep chroma below roughly `0.25` for sRGB-safe colours unless you are deliberately targeting wide-gamut displays via `color(display-p3 …)` and gating it with `@media (color-gamut: p3)`.

**Try It — see the lie.**

```html
<style>
  div { padding: 1rem; font: 1rem system-ui; }
  .h1 { background: hsl(60 100% 50%); }
  .h2 { background: hsl(240 100% 50%); }
  .o1 { background: oklch(60% 0.16 100); }
  .o2 { background: oklch(60% 0.16 265); }
</style>
<div class="h1">hsl 60, L 50%</div>
<div class="h2">hsl 240, L 50%</div>
<div class="o1">oklch L 60%, hue 100</div>
<div class="o2">oklch L 60%, hue 265</div>
```

**Expected:** the two `hsl()` bands look wildly different in weight — one nearly white, one nearly black. The two `oklch()` bands look like the same colour in two hues. Squint at the screen: the `oklch()` pair blurs into a single grey value, the `hsl()` pair does not.

<a id="p9-mastery"></a>

### Mastery: `color-mix()`, relative colour, and generated palettes

Two functions let you *derive* colours in CSS instead of hard-coding a ramp — the job people used Sass for.

**`color-mix()`** blends two colours in a named space ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/color-mix), widely available since May 2023):

```css
color-mix(in oklab, var(--accent) 70%, white)     /* a tint */
color-mix(in oklab, var(--accent) 70%, black)     /* a shade */
color-mix(in srgb, var(--accent) 20%, transparent) /* alpha, without touching the hex */
```

The interpolation space matters. `in oklab` (the default) mixes perceptually and keeps midpoints vivid; `in srgb` desaturates through the middle. Mixing with `transparent` is the cleanest way to make a translucent version of a custom property whose value you don't control.

Percentage rules from MDN: omit both and each gets 50%; give one and the other becomes `100% - p`; give two that don't sum to 100 and they are normalised.

**Relative colour syntax** derives a colour from another by decomposing it into channels you can do arithmetic on ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_colors/Relative_colors)):

```css
oklch(from var(--brand) calc(l + 0.1) c h)        /* 10% lighter */
oklch(from var(--brand) l calc(c * 0.5) h)        /* half the chroma */
oklch(from var(--brand) l c calc(h + 180))        /* the complement */
oklch(from var(--brand) l c h / 40%)              /* alpha only */
```

The `from` keyword binds `l`, `c`, `h` (or `r`, `g`, `b`, or `h`, `s`, `l`) as numbers you can use in `calc()`. This is genuinely a preprocessor function's worth of power, running live in the browser against a runtime custom-property value — something Sass fundamentally cannot do.

Feature-detect it, since it is newer than `color-mix()`:

```css
@supports (color: oklch(from white l c h)) {
  /* safe */
}
```

**A complete generated palette.** Putting it together, an entire themed system from two authored values:

```css
:root {
  color-scheme: light dark;
  --brand-h: 265;
  --brand: oklch(60% 0.16 var(--brand-h));

  --brand-hover:  oklch(from var(--brand) calc(l - 0.06) c h);
  --brand-subtle: color-mix(in oklab, var(--brand) 12%, canvas);
  --brand-ring:   color-mix(in oklab, var(--brand) 45%, transparent);

  --surface: light-dark(oklch(99% 0.005 var(--brand-h)),
                        oklch(20% 0.02  var(--brand-h)));
  --text:    light-dark(oklch(25% 0.02 var(--brand-h)),
                        oklch(93% 0.01 var(--brand-h)));
}
```

Change `--brand-h` and the whole system re-tints, in both themes, with no build step. Note `canvas` in the `color-mix()` — it is a **system colour keyword** that resolves to the browser's current canvas colour, so `--brand-subtle` automatically becomes a dark tint in dark mode rather than a washed-out light one.

**Contrast, honestly.** `oklch()` lightness correlates far better with perceived contrast than `hsl()` did, but it is **not** the WCAG contrast formula, which is defined on sRGB relative luminance. Equal `L` does not guarantee a passing contrast ratio. Use `oklch()` to make a palette *coherent*, and check the pairs you actually ship with a contrast checker (DevTools shows the ratio in the colour picker, with the AA/AAA thresholds marked).

### Part 9 cheat sheet

| Task | Declaration |
|---|---|
| Modern colour syntax | `rgb(255 0 0 / 80%)` — spaces, slash for alpha |
| Follow the text colour | `currentColor` |
| Enable browser dark UI | `:root { color-scheme: light dark }` |
| Theme value without a media query | `light-dark(#fff, #131313)` |
| Force a subtree's scheme | `color-scheme: dark` on that element |
| Perceptually even palette | `oklch(60% 0.16 <hue>)` at fixed L and C |
| Tint | `color-mix(in oklab, var(--c) 70%, white)` |
| Shade | `color-mix(in oklab, var(--c) 70%, black)` |
| Translucent version of a variable | `color-mix(in srgb, var(--c) 20%, transparent)` |
| Lighter, derived | `oklch(from var(--c) calc(l + 0.1) c h)` |
| Complement | `oklch(from var(--c) l c calc(h + 180))` |
| Theme-aware neutral | `canvas`, `canvastext` system keywords |
| Feature-detect relative colour | `@supports (color: oklch(from white l c h))` |

| Trap | Reality |
|---|---|
| `hsl()` lightness = perceived lightness | No — yellow and blue at `50%` look nothing alike |
| oklch hue = hsl hue | No — red is ~29°, not 0° |
| Higher chroma is always more vivid | Above ~0.25 it clamps to the sRGB gamut |
| Equal oklch `L` passes WCAG | No — check the ratio separately |
| Dark mode is just colours | Set `color-scheme` or form controls stay light |

[↑ Back to top](#table-of-contents)

---

## Part 10 — Backgrounds, Borders & Effects

<a id="p10-beginner"></a>

### Beginner: backgrounds and borders

`background` is a shorthand for eight longhands. As [Foundations Part 1](./css-foundations.md#p1-working) warned, using the shorthand resets all of them — so know the pieces:

```css
.panel {
  background-color: #eee;
  background-image: url("texture.png");
  background-repeat: no-repeat;
  background-position: center;
  background-size: cover;      /* or contain, or explicit lengths */
  background-origin: padding-box;
  background-clip: border-box;
  background-attachment: scroll;
}
```

`cover` fills the box and crops; `contain` fits the whole image and may letterbox. Those two cover ninety percent of real usage.

You can stack multiple backgrounds by comma-separating — **first in the list paints on top**:

```css
.hero {
  background:
    linear-gradient(rgb(0 0 0 / 60%), rgb(0 0 0 / 20%)),   /* on top */
    url("photo.jpg") center / cover no-repeat;             /* underneath */
}
```

That is the standard "darken a photo so white text is readable" pattern, and it needs no extra element.

**Borders** are `width style color`, and `border-style` is mandatory — `border: 1px red` renders nothing because the default style is `none`:

```css
/* WRONG - invisible */
.box { border: 1px crimson; }

/* RIGHT */
.box { border: 1px solid crimson; }
```

`outline` looks similar but behaves differently in the one way that matters: **outlines do not take up space**, so adding one never shifts the layout. That is why `outline` is the correct property for focus rings ([Foundations Part 2](./css-foundations.md#p2-working)) and for temporary debugging. Modern outlines follow `border-radius` automatically, and `outline-offset` pushes them away from the edge.

`border-radius` takes up to four corner values, and a `/` separates horizontal from vertical radii for elliptical corners:

```css
.pill   { border-radius: 999px; }             /* any height, always a pill */
.blob   { border-radius: 60% 40% 30% 70% / 60% 30% 70% 40%; }
```

<a id="p10-working"></a>

### Working Knowledge: gradients and shadows

**Gradients are images**, so they go in `background-image` (or anywhere an `<image>` is accepted) and can be stacked and blended ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_images/Using_CSS_gradients)).

```css
linear-gradient(to right, blue, pink)
linear-gradient(70deg, blue, pink)            /* 0deg = upward, 90deg = rightward */
radial-gradient(circle at 30% 20%, gold, crimson)
conic-gradient(from 45deg, red, orange 50%, yellow 85%, green)
repeating-linear-gradient(-45deg, #eee 0 10px, #ddd 10px 20px)   /* stripes */
```

Colour stops take positions, and giving one colour **two** positions creates a hard band — the trick behind stripes, progress bars, and fake borders:

```css
.progress {
  background: linear-gradient(to right,
    var(--brand) 0 var(--pct),
    #e5e5e5 var(--pct) 100%);
}
```

A bare number between two colours is a **colour hint** — it moves the midpoint without adding a stop: `linear-gradient(to top, lime, 20%, cyan)`.

**The two gradient bugs everyone hits:**

**1. The grey band.** `transparent` is transparent *black*, so fading a colour to `transparent` interpolates through dark grey:

```css
/* WRONG - a dirty grey band through the middle */
background: linear-gradient(to right, crimson, transparent);

/* RIGHT - fade to a transparent version of the same colour */
background: linear-gradient(to right, crimson, rgb(220 20 60 / 0));
/* or, without knowing the value: */
background: linear-gradient(to right, var(--c), color-mix(in srgb, var(--c) 0%, transparent));
```

**2. Banding on subtle gradients.** Long, low-contrast gradients show visible steps in 8-bit sRGB. Interpolating in a perceptual space smooths them:

```css
background: linear-gradient(in oklab, to right, #003049, #669bbc);
```

The `in <space>` syntax works on every gradient function, and `in oklch longer hue` will take the long way around the colour wheel — the easy way to make a rainbow from two stops.

**Shadows.** `box-shadow` is `x y blur spread color`, with an optional `inset`:

```css
.card { box-shadow: 0 1px 2px rgb(0 0 0 / 8%), 0 8px 24px rgb(0 0 0 / 12%); }
.well { box-shadow: inset 0 2px 4px rgb(0 0 0 / 15%); }
```

Layering two or three shadows with increasing blur and decreasing opacity reads as far more natural depth than one big shadow — that is how every good design system does elevation. Note the difference from `text-shadow`, which takes no spread and no inset. And note that `box-shadow` follows `border-radius`, while `filter: drop-shadow()` follows the *alpha channel* — see [Advanced](#p10-advanced).

<a id="p10-advanced"></a>

### Advanced: `filter`, `backdrop-filter`, blend modes

**`filter`** applies graphical effects to an element and everything inside it:

```css
filter: blur(4px);
filter: grayscale(1);
filter: brightness(1.2) contrast(0.9) saturate(1.4);   /* they compose, in order */
filter: drop-shadow(0 4px 8px rgb(0 0 0 / 30%));
filter: url(#my-svg-filter);
```

`drop-shadow()` vs. `box-shadow` is the distinction worth knowing: `box-shadow` traces the element's border box (respecting `border-radius`), while `drop-shadow()` traces the *rendered alpha* — so it correctly shadows a transparent PNG, an SVG icon, or a `clip-path` shape, which `box-shadow` cannot.

```css
/* WRONG - a rectangular shadow behind a transparent logo */
.logo { box-shadow: 0 4px 8px rgb(0 0 0 / 30%); }

/* RIGHT - follows the actual artwork */
.logo { filter: drop-shadow(0 4px 8px rgb(0 0 0 / 30%)); }
```

Remember from [Layout Part 4](./css-layout-flex-grid.md#p4-mastery): **any non-`none` `filter` creates a stacking context and a containing block for fixed descendants.** A `filter: grayscale(0)` added "harmlessly" for a hover transition changes the positioning behaviour of everything inside.

**`backdrop-filter`** applies the effect to what is *behind* the element — the frosted-glass effect:

```css
.glass {
  background: rgb(255 255 255 / 12%);
  backdrop-filter: blur(12px) saturate(1.6);
}
```

It needs a semi-transparent background to show anything through, and it is genuinely expensive to render — see [Part 14](./css-animation-performance.md). Use it on a small number of elements, and never animate the blur radius.

**Blend modes** control how layers combine. `mix-blend-mode` blends an element with what is behind it; `background-blend-mode` blends an element's own stacked backgrounds with each other:

```css
.duotone {
  background: url("photo.jpg") center/cover, linear-gradient(oklch(45% .2 265), oklch(75% .18 40));
  background-blend-mode: luminosity;
}
.caption { mix-blend-mode: difference; color: white; }   /* legible on any photo */
```

`mix-blend-mode` other than `normal` also creates a stacking context, and blending is confined to the nearest stacking context — which is exactly what `isolation: isolate` is for: put it on a wrapper to stop a blended child from blending with the whole page.

<a id="p10-mastery"></a>

### Mastery: clipping, masking, and forced-colors mode

**`clip-path`** cuts an element to a shape. Everything outside is not painted and not hit-testable:

```css
clip-path: inset(10px 20px round 8px);
clip-path: circle(50% at center);
clip-path: ellipse(40% 30% at 50% 40%);
clip-path: polygon(0 0, 100% 0, 100% 85%, 0 100%);   /* angled section divider */
clip-path: url(#svg-clip);
```

`clip-path` is animatable between shapes with the same function and the same number of points, which makes reveal animations easy. The `shape()` function — Baseline 2026, newer than the rest of this list — expresses responsive curved paths using CSS units and percentages rather than the fixed SVG coordinates that `path()` requires; verify support before relying on it.

**`mask`** is the general form: an image (usually a gradient) whose alpha channel decides what shows:

```css
/* Fade the bottom of a scrolling list instead of a hard cut */
.fade-out {
  mask-image: linear-gradient(to bottom, black 80%, transparent);
}
```

That is the cleanest "text fades out at the edge" technique in CSS, and it works over any background because it removes pixels rather than painting over them.

**Forced-colors mode** is where visual effects meet accessibility. When a user runs Windows High Contrast or an equivalent, the browser **overrides your colours with a system palette** ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/forced-colors), widely available since September 2022). Specifically, it forces:

- `color`, `background-color`, `border-color`, `outline-color`, `text-decoration-color`, SVG `fill` and `stroke` → system colours
- `box-shadow` → `none`
- `text-shadow` → `none`
- `background-image` → `none` for non-`url()` values, **which includes every gradient**

So a button whose only visual boundary is a `box-shadow` or a gradient becomes an invisible rectangle. The fix is to re-add a real border, using the system colour keywords:

```css
.button {
  border: 0;
  background: linear-gradient(var(--brand), var(--brand-hover));
  box-shadow: 0 2px 6px rgb(0 0 0 / 25%);
}

@media (forced-colors: active) {
  .button {
    border: 2px solid ButtonText;   /* system keyword, always contrasting */
  }
}
```

System colour keywords available include `Canvas`, `CanvasText`, `ButtonFace`, `ButtonText`, `LinkText`, `Highlight`, `HighlightText`, `Field`, `FieldText`. They also work outside forced-colors mode, where they follow `color-scheme` — which is why `canvas` appeared in the [Part 9 palette](#p9-mastery).

`forced-color-adjust: none` opts an element out entirely. MDN is explicit that this should be used for small, deliberate tweaks — a colour swatch whose colour *is* the content, a flag, a chart legend — never to preserve your design wholesale, because that defeats the user's setting.

**Try It.**

```html
<style>
  .btn { border: 0; padding: .6rem 1.2rem; color: white; border-radius: 6px;
         background: linear-gradient(oklch(55% .18 265), oklch(45% .18 265));
         box-shadow: 0 2px 6px rgb(0 0 0 / 30%); }
  @media (forced-colors: active) {
    .btn { border: 2px solid ButtonText; }
  }
</style>
<button class="btn">Continue</button>
```

**Expected:** in DevTools → Rendering → "Emulate CSS media feature forced-colors: active", the gradient and shadow vanish and the button is drawn with a system-coloured border. Comment out the `@media` block and repeat — the button becomes a bare label with no visible boundary at all.

### Part 10 cheat sheet

| Task | Declaration |
|---|---|
| Darken a photo behind text | `background: linear-gradient(rgb(0 0 0/60%), rgb(0 0 0/20%)), url(…) center/cover` |
| Stripes | `repeating-linear-gradient(-45deg, #eee 0 10px, #ddd 10px 20px)` |
| Progress bar with no extra element | Two-position colour stops |
| Fade to nothing without a grey band | Fade to `rgb(R G B / 0)`, not `transparent` |
| Smooth subtle gradient | `linear-gradient(in oklab, …)` |
| Natural elevation | Two or three layered `box-shadow`s |
| Shadow that follows artwork | `filter: drop-shadow(…)`, not `box-shadow` |
| Frosted glass | `backdrop-filter: blur(12px)` + translucent background |
| Duotone photo | `background-blend-mode: luminosity` |
| Contain a blend | `isolation: isolate` on a wrapper |
| Angled section edge | `clip-path: polygon(…)` |
| Fade the edge of a scroll area | `mask-image: linear-gradient(to bottom, black 80%, transparent)` |
| Border visible in high contrast | `@media (forced-colors: active) { border: 2px solid ButtonText }` |

| Gotcha | Note |
|---|---|
| `border: 1px red` invisible | `border-style` defaults to `none` |
| `background` shorthand wiped my image | Use `background-color` when overriding one aspect |
| Stacked backgrounds in the wrong order | First in the list paints **on top** |
| `filter` broke my fixed positioning | Non-`none` filter = stacking context + containing block |
| Gradients and shadows disappear for some users | Forced-colors mode — add a border |
| `backdrop-filter` shows nothing | The element needs a translucent background |

[↑ Back to top](#table-of-contents)

---

**Next:** [Architecture — Custom Properties, Layers & Scope (Parts 11–12)](./css-architecture.md) — how all of this stays maintainable past 200 lines.
