# Vue 3 — Rendering Internals & Performance (Parts 12–13)

> **Spec:** this doc follows the shared spec in [`vue-mastery-guide.md` → About This Document](./vue-mastery-guide.md#about-this-document) — curiosity-driven, `<script setup lang="ts">` only, Vue **3.5.42**, verified **September 6, 2026**.
>
> **Assumes** the whole set, especially [Parts 1–2 — Reactivity](./vue-reactivity.md) and [Part 3 — Templates](./vue-templates-directives.md).

This is the pair of Parts that explains *why*. Part 12 is the machine: how a template becomes DOM, and what the compiler knows that a runtime-only library cannot. Part 13 is what to do with that knowledge when something is slow — and what is coming next in Vapor Mode.

---

## Table of Contents

**Part 12 — The Rendering Mechanism**
1. [Working Knowledge: compile → mount → patch](#1-working-knowledge-compile--mount--patch)
2. [Working Knowledge: the render effect, and what triggers a re-render](#2-working-knowledge-the-render-effect-and-what-triggers-a-re-render)
3. [Advanced: the compiler-informed virtual DOM](#3-advanced-the-compiler-informed-virtual-dom)
4. [Mastery: patch flags, static caching, and block trees](#4-mastery-patch-flags-static-caching-and-block-trees)
5. [Mastery: reading the compiled output yourself](#5-mastery-reading-the-compiled-output-yourself)

**Part 13 — Performance & What's Next**

6. [Working Knowledge: measure before you optimise](#6-working-knowledge-measure-before-you-optimise)
7. [Working Knowledge: page-load performance](#7-working-knowledge-page-load-performance)
8. [Advanced: update performance](#8-advanced-update-performance)
9. [Advanced: `v-once`, `v-memo`, and computed stability](#9-advanced-v-once-v-memo-and-computed-stability)
10. [Advanced: large data — shallow reactivity and virtualisation](#10-advanced-large-data--shallow-reactivity-and-virtualisation)
11. [Mastery: Vapor Mode](#11-mastery-vapor-mode)
12. [Cheat Sheet](#cheat-sheet)

*Part 12 has no Beginner tier: nothing here is needed to write correct Vue, and reading it before Parts 1–11 inverts the actual learning order.*

---

# Part 12 — The Rendering Mechanism

## 1. Working Knowledge: compile → mount → patch

Vue's rendering runs in three stages ([Rendering Mechanism](https://vuejs.org/guide/extras/rendering-mechanism.html)):

```
   ┌─────────────┐   build time    ┌──────────────────┐
   │  template   │ ──────────────► │ render function  │   returns a vnode tree
   └─────────────┘   (compile)     └────────┬─────────┘
                                            │  runtime
                        ┌───────────────────▼───────────────────┐
                        │ MOUNT: walk the vnode tree, create     │
                        │ real DOM. Done as a reactive effect,   │
                        │ so every dep read is tracked.          │
                        └───────────────────┬───────────────────┘
                                            │  a dep changes
                        ┌───────────────────▼───────────────────┐
                        │ PATCH: re-run render → new vnode tree, │
                        │ diff against the old, apply the        │
                        │ minimum DOM updates.                   │
                        └────────────────────────────────────────┘
```

A **vnode** is a plain object describing what the DOM should be:

```js
const vnode = { type: 'div', props: { id: 'hello' }, children: [ /* more vnodes */ ] }
```

The two operations on that tree are **mount** (build real DOM from it) and **patch** (diff two trees, apply the differences).

The important line in the official description is that mounting *"is performed as a reactive effect, so it keeps track of all reactive dependencies that were used."* That single sentence connects this Part to [Part 1](./vue-reactivity.md): the render is an effect, the template's property reads are its dependencies, and a write to any of them re-runs it.

[↑ Back to top](#table-of-contents)

---

## 2. Working Knowledge: the render effect, and what triggers a re-render

Each component instance owns one render effect. It re-runs when a reactive value **that its own template read** changes.

Three consequences that answer most "why did/didn't this update" questions:

**A parent re-rendering does not force a child to re-render.** The child re-renders only if its props actually changed, or its own state did. There is no `React.memo` equivalent because it is the default.

**Reading a value in `<script setup>` but not in the template does not create a render dependency.** The render effect tracks what the *render* reads. A ref used only inside an event handler never re-renders anything.

**Updates are batched.** Multiple mutations in the same tick queue one re-render, flushed on the microtask queue — which is what `nextTick()` waits for:

```ts
count.value++
count.value++
count.value++          // one re-render, not three
await nextTick()       // now the DOM reflects count === 3
```

> **vs. React.** The default blast radius is inverted. React: parent renders → children render, unless you memoise. Vue: only components that read the changed value render, always. The corollary is that Vue's performance work is about *what a component reads*, not about *stabilising identities* — and that a huge component reading many values is Vue's version of the too-coarse-grained problem.

[↑ Back to top](#table-of-contents)

---

## 3. Advanced: the compiler-informed virtual DOM

A pure runtime VDOM has to assume anything in the tree might have changed, so patching means walking the whole tree and comparing. Vue controls **both the compiler and the runtime**, and the docs name the resulting hybrid explicitly: a **"Compiler-Informed Virtual DOM"** ([Rendering Mechanism](https://vuejs.org/guide/extras/rendering-mechanism.html)).

The compiler can prove things about your template that a runtime never learns:

- this `<div class="card">` has no bindings at all — it can never change;
- this element's *only* dynamic part is its class;
- between this `v-if` and that `v-for`, the structure of this region is fixed.

Each of those facts becomes an annotation in the generated render function, and the runtime uses the annotation to skip work. This is why templates are the default in Vue while JSX is the escape hatch: **JSX is opaque to the compiler**, so a hand-written render function gets a full generic diff. Nothing is broken about that — it's just that you gave up the optimisations by hiding your intent.

[↑ Back to top](#table-of-contents)

---

## 4. Mastery: patch flags, static caching, and block trees

The three optimisations, in the docs' own terminology.

### Cache static

Markup with no bindings is hoisted out of the render function and reused by reference — it is created once and the same vnode is returned on every render. Enough consecutive static elements are condensed into a **single static vnode holding a plain HTML string**, which the runtime can insert with one `innerHTML` write instead of building nodes individually.

### Patch flags

Each dynamic vnode is created with a bitmask describing *what kind* of update it can need:

```js
createElementVNode("div", {
  class: _normalizeClass({ active: _ctx.active })
}, null, 2 /* CLASS */)
```

At patch time the runtime tests the flag with **bitwise operations** and does only the corresponding work — here, compare and update `class`, and nothing else. No prop enumeration, no children walk.

### Tree flattening (block trees)

A **block** is *"a part of the template that has stable inner structure"*. Each block tracks all its dynamic descendants — not just direct children — in a **flat array**:

```
  template                          block's dynamic children (flat)
  <div>                 ← block root
    <div>static</div>               ┌─────────────────┐
    <div>static</div>               │ 0: <span :class>│  ← patch flag CLASS
    <div>                           │ 1: {{ msg }}    │  ← patch flag TEXT
      <span :class="c" />           └─────────────────┘
      <p>{{ msg }}</p>
    </div>                          patching walks 2 nodes, not 6
  </div>
```

The result, as the docs put it, *"greatly reduces the number of nodes that need to be traversed during virtual DOM reconciliation."* Structure can only change at `v-if`/`v-for` boundaries, so those create nested blocks — everything else is provably stable and can be flattened.

Both patch flags and tree flattening also improve **SSR hydration** performance, for the same reason: the hydrator can skip static regions instead of walking them.

**What this means for how you write templates.** Not much day to day, which is the point — you get it for free. But it does explain two pieces of practical advice elsewhere in this set: prefer directives to hand-rolled render logic ([Part 3 §7](./vue-templates-directives.md#7-mastery-what-the-compiler-is-allowed-to-assume)), and don't be afraid of static markup inside a frequently-updating component, because static markup costs nothing per update.

[↑ Back to top](#table-of-contents)

---

## 5. Mastery: reading the compiled output yourself

The fastest way to make the previous section concrete is to look at what your own template compiles to. The **Vue SFC Playground** (linked from [vuejs.org](https://vuejs.org)) shows compiled JS beside the source; the same output appears in your bundler's dev output.

Paste in a small component and read the render function. Things to look for:

| In the output | Means |
|---|---|
| `_hoisted_1`, `_cache[0]` at module scope | Static content cached outside the render |
| `, 2 /* CLASS */` as the 4th arg | Patch flag — only the class can change |
| `, 1 /* TEXT */` | Only the text content can change |
| `, 8 /* PROPS */, ["id"]` | Only these named props can change |
| `_openBlock()`, `_createElementBlock` | A block boundary — usually a `v-if`/`v-for` root |
| `_createElementVNode` with no flag | Fully static, or the compiler couldn't prove anything |
| `-1 /* HOISTED */` / `_normalizeClass` | Hoisted node / runtime class normalisation |

**Try It.** Compile these two templates and compare:

```vue
<!-- A -->
<div class="card"><h2>Title</h2><p>{{ msg }}</p></div>

<!-- B -->
<div :class="cls"><h2>{{ title }}</h2><p>{{ msg }}</p></div>
```

A hoists the `<h2>` (it can never change) and gives the `<p>` a `TEXT` patch flag. B has no hoisting and three dynamic bindings, with flags on each. Nothing about B is *wrong* — but you can see the runtime cost difference in the output, and you can see that the win came from the compiler being able to prove that the `<h2>` was static.

[↑ Back to top](#table-of-contents)

---

# Part 13 — Performance & What's Next

## 6. Working Knowledge: measure before you optimise

The official guide splits performance into **page load** (LCP, INP) and **update** performance, and names the tools ([Performance](https://vuejs.org/guide/best-practices/performance.html)):

- **Production:** PageSpeed Insights, WebPageTest.
- **Development:** the Chrome DevTools Performance panel and the Vue DevTools extension.
- Set `app.config.performance = true` to emit Vue's own component-level markers into the Performance panel timeline.

Two cautions worth stating loudly. **Development builds are much slower than production** — dev-mode warnings, reactivity debug info and no minification. Never conclude anything from `npm run dev` timings; measure a production build. And an optimisation without a before-and-after number is a guess: the [`v-memo`](#9-advanced-v-once-v-memo-and-computed-stability) and `shallowRef` sections below both describe tools that can make code *slower* and more fragile when applied to the wrong thing.

[↑ Back to top](#table-of-contents)

---

## 7. Working Knowledge: page-load performance

The docs' guidance, in order of typical impact:

**Choose the right architecture.** If load performance is critical, a pure client-side SPA is the wrong shape — SSR or SSG. For a marketing page, static HTML with minimal JS.

**Ship less JavaScript:**

- Use a build step so the runtime is tree-shaken — unused built-ins like `<Transition>` are dropped.
- Pre-compile templates so you don't ship the compiler: **~14 kB min+gzip** saved. (This is automatic with SFCs and a build step; you only pay it in a no-build/in-DOM-template setup.)
- Prefer tree-shakeable ES-module dependencies — `lodash-es` over `lodash`.
- For progressive enhancement of mostly-static pages, `petite-vue` is **~6 kB**.

**Code-split.** Dynamic `import()`, and `defineAsyncComponent` for components ([Part 6](./vue-components-script-setup.md#9-working-knowledge-defineasynccomponent)):

```ts
const SettingsDialog = defineAsyncComponent(() => import('./SettingsDialog.vue'))
```

**Hydrate lazily** in SSR apps — `hydrateOnVisible()` and friends ([Part 6 §10](./vue-components-script-setup.md#10-advanced-lazy-hydration-35)).

[↑ Back to top](#table-of-contents)

---

## 8. Advanced: update performance

### Prop stability

The most Vue-specific optimisation in the official guide, and the least obvious. Keep the props you pass to children as stable as possible, so a change to one thing doesn't touch every child:

```vue
<!-- ✗ every ListItem re-renders when activeId changes: each one's prop changed -->
<ListItem v-for="item in items" :key="item.id" :active-id="activeId" />

<!-- ✓ only the two items whose boolean flipped re-render -->
<ListItem v-for="item in items" :key="item.id" :active="item.id === activeId" />
```

Both compute `active` from the same data. The difference is *where*: in the first, every child receives a value that changed, so every child re-renders. In the second, the comparison happens in the parent's render and only two children see a different prop.

### Avoid unnecessary component abstractions

The docs are blunt that a component instance is much more expensive than a DOM node, and that this matters most in large lists — removing one layer of abstraction there can eliminate hundreds of instances. This cuts against normal decomposition instincts, so scope it precisely: it applies to the inner rows of a big list, not to your app's general structure.

### Keep components' reads narrow

A component whose template reads twenty reactive values re-renders when any of the twenty changes. Splitting it means each piece re-renders only for its own data. This is the Vue-shaped version of "component granularity is a performance decision."

[↑ Back to top](#table-of-contents)

---

## 9. Advanced: `v-once`, `v-memo`, and computed stability

**`v-once`** renders a subtree once and skips it for the lifetime of the component:

```vue
<footer v-once>
  <p>© {{ year }} — rendered once, never patched again</p>
</footer>
```

**`v-memo="[deps]"`** skips updating a subtree unless one of the dependencies changed. It's a scalpel, and its one genuinely great use is a long `v-for` where each row is expensive and only a couple of fields matter:

```vue
<div v-for="item in list" :key="item.id" v-memo="[item.id === selected, item.updatedAt]">
  <!-- expensive row markup; skipped entirely unless selection or timestamp changed -->
</div>
```

Get the dependency array wrong and you get a **stale UI that looks like a data bug** — visibly wrong content with no error. Treat `v-memo` as a last resort after measuring, and keep the array exhaustive.

**Computed stability (3.4+)** — computeds only trigger effects when the value *actually changes*, compared with `Object.is`. For a computed returning a fresh object each time, that comparison always fails; you can opt into returning the previous value ([Performance](https://vuejs.org/guide/best-practices/performance.html)):

```ts
const computedObj = computed(oldValue => {
  const newValue = { isEven: count.value % 2 === 0 }
  if (oldValue && oldValue.isEven === newValue.isEven) return oldValue   // no downstream update
  return newValue
})
```

The getter receiving its previous value is the piece people don't know exists. Reach for it when a derived object drives an expensive subtree and its *meaning* changes far less often than its identity.

[↑ Back to top](#table-of-contents)

---

## 10. Advanced: large data — shallow reactivity and virtualisation

**Virtualise long lists.** Rendering 10,000 rows is a DOM problem before it's a Vue problem — render only what's near the viewport. The docs point to `vue-virtual-scroller`, `vue-virtual-scroll-grid`, and `vueuc/VVirtualList`. This is nearly always the biggest win available on a slow table, and it's worth doing before any of the micro-optimisations above.

**Reduce reactivity overhead on large immutable structures** with `shallowRef`/`shallowReactive` ([Part 2](./vue-reactivity.md#7-working-knowledge-shallowref-and-shallowreactive)):

```ts
const rows = shallowRef<Row[]>([])

rows.value.push(newRow)                    // ✗ invisible — inner value isn't reactive
rows.value = [...rows.value, newRow]       // ✓ replace at the root
```

The trade the docs name: you must treat nested objects as immutable and only replace at the root. Note the 3.5 context — the reactivity refactor cut memory by **56%** and made large deeply-reactive arrays up to **10× faster** ([Vue 3.5](https://blog.vuejs.org/posts/vue-3-5)) — so on 3.5+ this is an optimisation you should reach for after measuring, not by default.

**Keep third-party instances out of reactivity entirely** with `markRaw` ([Part 2 §8](./vue-reactivity.md#8-advanced-triggerref-customref-toraw-markraw)) — a chart, a map, an editor. Proxying them is wasted work at best.

**Real Scenario — the table that got slower after "optimising".** A 5,000-row admin table is janky on filter. The team adds `v-memo` to the rows and `shallowRef` to the data. It gets slightly better, then a bug arrives: edits to a row don't show until the page is refreshed — `shallowRef` plus in-place mutation, silently. Rolling both back and adding virtualisation instead takes the render from thousands of rows to about forty, is faster than either patch, and reintroduces no bugs. The order to try things in is: **reduce how much you render**, then **reduce how often**, then **reduce how much reactivity costs** — the last one being where the correctness risk lives.

[↑ Back to top](#table-of-contents)

---

## 11. Mastery: Vapor Mode

**Status, September 6, 2026:** Vue **3.6 is in RC** (v3.6.0-rc.7); stable remains **3.5.42** ([vuejs/core releases](https://github.com/vuejs/core/releases)). Vapor Mode is feature-complete in the RC apart from `<Suspense>` support. Treat this section as forward-looking, and re-verify before relying on it.

**What it is.** A second compilation strategy. A Vapor component's template compiles **straight to DOM operations** — no vnodes, no virtual tree, no diffing. Reactive effects are wired directly to the specific DOM nodes they update, in the manner of Solid and Svelte.

```
  Today (VDOM)                     Vapor
  state → render fn → vnode tree   state → effect bound to a text node
       → diff vs. previous              → node.data = newValue
       → patch DOM
```

**Why it's faster.** Everything Part 12 describes — patch flags, block trees, static hoisting — exists to make diffing cheap. Vapor doesn't diff at all, so the cost disappears rather than being minimised, and the vnode allocations go with it. Bundle size drops too, since a Vapor-only app doesn't ship the VDOM runtime.

**What it costs.** Vapor supports a *subset* of existing APIs, so it is opt-in per component and coexists with VDOM components in the same app. Anything relying on vnodes — hand-written render functions, JSX, some libraries that manipulate vnodes — doesn't translate directly.

**What to do about it today:** nothing structural. Everything in this set is standard Composition API and templates, which is exactly what Vapor compiles best. The practical preparation is to avoid hand-written render functions and vnode manipulation where a template will do — which is also the advice for getting the most out of the current compiler.

**For the next revision of this doc:** if 3.6 is stable, verify whether Vapor is still opt-in, whether `<Suspense>` support landed, and whether the API subset restrictions changed. Also re-check `<Suspense>`'s experimental label ([Part 11](./vue-composables-builtins.md#10-advanced-suspense-experimental)), since its Vapor support was the last blocker named before stable.

[↑ Back to top](#table-of-contents)

---

## Cheat Sheet

### The pipeline

| Stage | What happens |
|---|---|
| **Compile** | Template → render function, annotated with patch flags, hoists, blocks |
| **Mount** | Render fn runs *as a reactive effect*, tracking every dep it reads; vnodes → DOM |
| **Patch** | A dep changes → effect re-runs → new vnode tree diffed against old → minimal DOM writes |

### Compiler optimisations

| Optimisation | Effect |
|---|---|
| Cache static | Binding-free markup hoisted and reused; runs of static nodes become one string vnode |
| Patch flags | Bitmask per dynamic vnode; runtime updates only the flagged aspect |
| Tree flattening (blocks) | Each stable region tracks dynamic descendants in a flat array |
| SSR hydration | Both of the above let the hydrator skip static regions |

### Performance decision order

| Ask | If yes |
|---|---|
| Am I measuring a **production** build? | Otherwise stop and rebuild |
| Is it **page load**? | SSR/SSG, code-split, tree-shake, lazy hydration |
| Am I rendering **too many things**? | Virtualise; remove component layers in big lists |
| Am I rendering **too often**? | Prop stability, split components, `v-once`/`v-memo` |
| Is **reactivity** itself the cost? | `shallowRef`/`shallowReactive`, `markRaw`, computed stability |

### Tools

| Tool | For |
|---|---|
| `app.config.performance = true` | Vue markers in the Chrome Performance panel |
| Vue DevTools | Component inspector, timeline |
| `onRenderTracked` / `onRenderTriggered` | Why *this* component re-rendered ([Part 2](./vue-reactivity.md#11-mastery-debugging-reactivity)) |
| SFC Playground | See your compiled render function |
| PageSpeed Insights / WebPageTest | Field and lab load metrics |

### Gotcha index

| Symptom | Cause | Fix |
|---|---|---|
| "Vue is slow" in dev | Dev build overhead | Measure a production build |
| Whole list re-renders on selection | A changing prop passed to every child | Pass a derived boolean per child |
| Stale content, no error | Wrong `v-memo` deps | Complete the deps array, or remove `v-memo` |
| Mutations invisible after "optimising" | `shallowRef` + in-place mutation | Replace at the root |
| Third-party instance misbehaves | It got proxied | `markRaw` |
| Downstream effects fire constantly | Computed returns a new object each run | Return `oldValue` when meaning is unchanged (3.4+) |
| Huge table still janky after micro-optimising | Rendering too many nodes | Virtualise |

**Up:** [Overview](./vue-mastery-guide.md) · **Start of set:** [Parts 1–2 — Reactivity](./vue-reactivity.md)

[↑ Back to top](#table-of-contents)
