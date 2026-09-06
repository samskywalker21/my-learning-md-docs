# Vue 3 — Mastery Guide (Overview)

The entry point for this repo's Vue material. This document holds the shared spec, the setup you need before running anything, the map of the focused sub-docs, the 3.5 → 3.6/Vapor delta, and the reference apparatus (learning order, self-check, cross-doc cheat sheet).

The actual teaching lives in the sub-docs. Start here, then follow the [Suggested Learning Order](#7-suggested-learning-order).

---

## About This Document

This section is the canonical spec for **every doc in this set** — the sub-docs each carry a short pointer back here rather than repeating it.

- **Framing: curiosity-driven.** No upcoming task is driving this. The goal is durable, hands-on understanding of Vue itself. The docs are built as a *guided tutorial* — a sequence of runnable examples and **Try It** exercises you are meant to actually type into a running app, each stating what correct output looks like. Theory ("why is it designed this way") appears only where it explains a mechanism — dependency tracking, patch flags, the injection chain — never as a standalone history section.
- **Confirmed scope: core Vue 3 + Composition API only.** Reactivity, template syntax and directives, SFCs and `<script setup>` macros, slots, provide/inject, watchers and lifecycle, composables, built-in components (`<Transition>`/`<Teleport>`/`<Suspense>`/`<KeepAlive>`), custom directives, SFC styling, and enough rendering internals to explain *why* the API behaves as it does.
- **Deliberately excluded** (with pointers): Vue Router, Pinia, Nuxt, testing, and the Vite/build layer. SSR appears only where it leaks into core API design (`useId`, lazy hydration, `onMounted` not running on the server). See [Deliberately Not Covered](#6-deliberately-not-covered-and-where-to-look-instead).
- **Composition API only.** `<script setup lang="ts">` is the only style taught. The Options API appears exactly once, as a translation table in the [Cross-Doc Cheat Sheet](#9-cross-doc-cheat-sheet), so you can read older code and older answers without being misled by them.
- **Depth tiers.** Each Part runs **Beginner → Working Knowledge → Advanced → Mastery**, so you can stop at whichever level you need today. Tiers are collapsed where one would be filler, and each collapse is stated inline. Collapsing is expected behaviour, not a defect to fix on a later pass. The weight of this set sits in Working Knowledge and Advanced; Mastery concentrates in [Internals & Performance](./vue-internals-performance.md).
- **Assumed background: modern JavaScript and TypeScript.** You are comfortable with ES modules, destructuring, `async`/`await`, closures, and basic generics. If any of that is shaky, read [`javascript/javascript-functions-closures.md`](../javascript/javascript-functions-closures.md) and [`typescript/typescript-foundations.md`](../typescript/typescript-foundations.md) first — this set does not re-teach the language. No prior Vue is assumed.
- **React contrast callouts.** Because [`react/react-mastery-guide.md`](../react/react-mastery-guide.md) is already in this repo, short `> **vs. React**` callouts appear *only* where the mental models genuinely diverge (fine-grained reactivity vs. re-render-and-diff, composables vs. Rules of Hooks, `ref()` vs. `useRef`). They are a shortcut, never a prerequisite — skip them freely if React is not in your head.
- **Per-section ingredients**, used where they earn their place rather than forced into every subsection:
  - a short plain-language explanation of what a thing is and why it matters;
  - a **wrong vs. right** snippet pair wherever a gotcha has a common wrong form;
  - a **Try It** — a runnable exercise with the expected observable result — since this is a curiosity-driven set (production "Real Scenario" blocks appear only where a bug genuinely only bites in production);
  - an **ASCII diagram** where the concept is spatial or structural — the effect graph, the injection chain, the block tree, the `<Transition>` class timeline.
- **Sourcing rule.** [vuejs.org](https://vuejs.org) is the primary source and is cited inline at the specific claim it supports. Where a popular-but-outdated convention disagrees with the official docs, the official docs win and the doc says so explicitly. **Deviation from the repo template, stated openly:** Stack Overflow was not reachable from the authoring environment on the date below, so no SO links appear in this set. Gotchas are instead sourced to the caveats the official docs state themselves. If you extend these docs from an environment that *can* reach SO, adding real-world SO citations at the gotchas is a welcome improvement — just don't construct SO URLs from memory.
- **Written against.** **Vue 3.5.42** — the latest stable release on npm, published August 27, 2026 ([vuejs/core releases](https://github.com/vuejs/core/releases)). **Vue 3.6 is still RC** (v3.6.0-rc.7, September 4, 2026); its headline feature, **Vapor Mode**, is covered at Mastery level in [Internals & Performance](./vue-internals-performance.md) as an experimental opt-in, not taught as the default. Node `^22.18.0 || >=24.12.0` ([Quick Start](https://vuejs.org/guide/quick-start.html)). TypeScript 5.9+. Verified **September 6, 2026**.
- **To update this doc later.** *Preserve*: the curiosity-driven framing, the numbered Part structure with explicit tier subheadings, the wrong-vs-right pairs, the Try It exercises with expected results, the `> **vs. React**` callout convention, the TOC/back-to-top/anchor apparatus, and the per-Part cheat sheets. *Re-verify against current docs first*: the stable version and Node requirement; **everything in [§5](#5-the-35--36-vapor-delta)** — if 3.6 has gone stable, promote Vapor out of "experimental", re-check whether it changes the `shallowRef`/custom-directive guidance (Vapor supports only a subset of existing APIs), and check whether `<Suspense>` lost its experimental label; and the 3.5-introduced APIs (`useTemplateRef`, `useId`, lazy hydration, `deep: number`, `onWatcherCleanup`), which are new enough that blog posts still show the pre-3.5 forms.

---

## Table of Contents

1. [How This Set Is Organised](#1-how-this-set-is-organised)
2. [The Sub-Docs](#2-the-sub-docs)
3. [Orientation & Setup](#3-orientation--setup)
4. [The One-Paragraph Mental Model](#4-the-one-paragraph-mental-model)
5. [The 3.5 → 3.6 / Vapor Delta](#5-the-35--36-vapor-delta)
6. [Deliberately Not Covered (and where to look instead)](#6-deliberately-not-covered-and-where-to-look-instead)
7. [Suggested Learning Order](#7-suggested-learning-order)
8. [Quick Self-Check](#8-quick-self-check)
9. [Cross-Doc Cheat Sheet](#9-cross-doc-cheat-sheet)

---

## 1. How This Set Is Organised

Vue is 13 Parts of teaching material spread over 6 sub-docs, plus this overview. The split follows the order in which the concepts actually become necessary — reactivity first, because every other API in Vue is a consequence of it:

```
  vue-mastery-guide.md   ← you are here: spec, setup, map, 3.5→3.6 delta, reference
        │
        ├── vue-reactivity.md               Parts 1–2    what changes, and how Vue knows
        ├── vue-templates-directives.md     Parts 3–4    how state becomes DOM
        ├── vue-components-script-setup.md  Parts 5–6    how you split it into pieces
        ├── vue-slots-injection-watchers.md Parts 7–9    how the pieces talk to each other
        ├── vue-composables-builtins.md     Parts 10–11  how you reuse logic and behaviour
        └── vue-internals-performance.md    Parts 12–13  how it actually works, and fast
```

Each Part is self-contained enough to reopen on its own, and each ends with a quick-reference table. Within a Part, the tiers mean:

| Tier | What you get |
|---|---|
| **Beginner** | What it is, the minimal correct usage, nothing else |
| **Working Knowledge** | Everyday patterns, the API surface you actually reach for |
| **Advanced** | Edge cases, gotchas, performance and correctness implications |
| **Mastery** | Internals, powerful-but-rare capabilities, expert tradeoffs |

Where a tier would be filler, it is collapsed and the doc says so — e.g. "*SFC styling gets Beginner → Advanced only; there is no honest Mastery tier beyond reading the PostCSS output, which Part 12 covers.*"

[↑ Back to top](#table-of-contents)

---

## 2. The Sub-Docs

### [`vue-reactivity.md`](./vue-reactivity.md) — Parts 1–2

| Part | Covers |
|---|---|
| 1. Refs, Reactive & Computed | `ref()` as the default, `reactive()` and its three limitations, `computed()`, unwrapping rules, `toRef`/`toRefs`/`toValue`, `readonly` |
| 2. The Shape of Reactivity | Deep vs. shallow, `shallowRef`/`shallowReactive`/`triggerRef`/`customRef`, `toRaw`/`markRaw`, `effectScope`, arrays and collections, reactivity debugging |

**Read this if:** something "isn't updating", or you can't decide between `ref` and `reactive`. Nearly every Vue bug that isn't a template typo starts here.

### [`vue-templates-directives.md`](./vue-templates-directives.md) — Parts 3–4

| Part | Covers |
|---|---|
| 3. Template Syntax & Directives | Interpolation, `v-bind` and its shorthands, `v-if` vs. `v-show`, `v-for` and the rules of keys, class/style binding, `v-html` and its risk |
| 4. Events & Forms | `v-on`, event and key modifiers, `v-model` on every input type, `.lazy`/`.number`/`.trim`, IME composition, validation patterns |

**Read this if:** your list re-uses the wrong DOM node, or `v-model` is fighting you.

### [`vue-components-script-setup.md`](./vue-components-script-setup.md) — Parts 5–6

| Part | Covers |
|---|---|
| 5. Components & `<script setup>` | SFC anatomy, registration, `defineProps`/`defineEmits`, reactive props destructure (3.5), `defineModel`, `defineExpose`/`defineOptions`, fallthrough attributes, generic components |
| 6. Dynamic & Async Components | `<component :is>`, `defineAsyncComponent`, loading/error states, lazy hydration (3.5), `v-model` on components |

**Read this if:** you're writing your third component and want to stop guessing which macro does what.

### [`vue-slots-injection-watchers.md`](./vue-slots-injection-watchers.md) — Parts 7–9

| Part | Covers |
|---|---|
| 7. Slots & Content Distribution | Default/named/scoped slots, fallback content, `$slots` and conditional wrappers, dynamic slot names, `defineSlots`, renderless components |
| 8. Provide / Inject | `provide`/`inject`, `InjectionKey` typing, app-level provide, defaults and factories, `readonly()` boundaries, when this beats a store |
| 9. Watchers, Lifecycle & Errors | `watch` vs. `watchEffect`, source types, `deep`/`immediate`/`once`/`flush`, `onWatcherCleanup`, the lifecycle hooks, `onErrorCaptured` and `app.config.errorHandler` |

**Read this if:** you're prop-drilling, or a watcher fires too often, too late, or never.

### [`vue-composables-builtins.md`](./vue-composables-builtins.md) — Parts 10–11

| Part | Covers |
|---|---|
| 10. Composables | Naming and call-site rules, returning refs, accepting `MaybeRefOrGetter` + `toValue`, cleanup with `onScopeDispose`, SSR-safe patterns, why the Rules of Hooks don't apply |
| 11. Built-in Components, Directives & Styling | `<Transition>`/`<TransitionGroup>`, `<Teleport>`, `<Suspense>` (experimental), `<KeepAlive>`, custom directives, `scoped`/`:deep()`/CSS Modules/`v-bind()` in CSS |

**Read this if:** you're extracting reusable logic, or animating/portalling/caching a component.

### [`vue-internals-performance.md`](./vue-internals-performance.md) — Parts 12–13

| Part | Covers |
|---|---|
| 12. The Rendering Mechanism | Compile → mount → patch, the render effect, compiler-informed VDOM, patch flags, static caching, tree flattening and block trees |
| 13. Performance & What's Next | Prop stability, `v-once`/`v-memo`, computed stability (3.4+), shallow reactivity for big data, virtualisation, profiling, and Vapor Mode |

**Read this if:** you want to know *why* Vue is fast, or why yours isn't.

[↑ Back to top](#table-of-contents)

---

## 3. Orientation & Setup

You need a running project to do the **Try It** exercises. Everything in this set works in a default scaffold.

**Prerequisite:** Node `^22.18.0 || >=24.12.0` ([Quick Start](https://vuejs.org/guide/quick-start.html)).

```bash
npm create vue@latest
```

Answer the prompts as follows for this set:

| Prompt | Answer | Why |
|---|---|---|
| Add TypeScript? | **Yes** | Every example here is `<script setup lang="ts">` |
| Add JSX Support? | No | Templates are the teaching medium; JSX is out of scope |
| Add Vue Router? | No | Out of scope — see [§6](#6-deliberately-not-covered-and-where-to-look-instead) |
| Add Pinia? | No | Out of scope; Part 8 shows how far provide/inject gets you first |
| Add Vitest / E2E? | No | Testing is out of scope |
| Add ESLint + Prettier? | Yes | Harmless, and the Vue ESLint rules catch two of the gotchas in Part 3 |
| Add Vue DevTools 7? | **Yes** | You will use the component and timeline panels in Parts 9 and 13 |

```bash
cd <your-project>
npm install
npm run dev
```

Also install the **Vue - Official** VS Code extension — the officially recommended IDE setup, and what type-checks `defineProps<{...}>()` in your editor ([Quick Start](https://vuejs.org/guide/quick-start.html)).

**The file to scribble in.** Replace `src/App.vue` with this and you have a scratchpad every Try It in this set can build on:

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
</script>

<template>
  <main>
    <button @click="count++">count is {{ count }}</button>
  </main>
</template>
```

Click the button; the label increments. If it doesn't, stop here and fix the setup — every later exercise assumes this works.

> **No-build option.** For a throwaway experiment you can skip the toolchain entirely with the global build from a CDN ([Quick Start](https://vuejs.org/guide/quick-start.html)). It is genuinely useful for isolating a reactivity question, but you lose SFCs, `<script setup>`, and every compile-time optimisation in [Part 12](./vue-internals-performance.md) — so it is not the environment this set is written for.

[↑ Back to top](#table-of-contents)

---

## 4. The One-Paragraph Mental Model

Vue keeps two things in sync: **reactive state** and **the DOM**. You declare state with `ref()`; you declare a template that reads it. When Vue renders a component it runs the template as a *reactive effect*, and every reactive property the template touches during that run is recorded as a dependency of that effect ([Rendering Mechanism](https://vuejs.org/guide/extras/rendering-mechanism.html), [Reactivity in Depth](https://vuejs.org/guide/extras/reactivity-in-depth.html)). When one of those properties is written to, Vue re-runs exactly the effects that depend on it — nothing else. That is the whole system. Components, computed values, watchers, and props are all the same mechanism wearing different clothes: a component's render is an effect, a `computed` is a *lazy* effect, a `watch` is an effect with a callback. Everything that surprises you about Vue — why destructuring `reactive()` breaks, why `watch(obj.count, ...)` doesn't fire, why `shallowRef` is faster — is a consequence of one fact: tracking happens **on property access, during a synchronous run**.

```
   state                  effects                          DOM
  ┌───────┐   read       ┌─────────────────────┐  patch   ┌──────┐
  │ ref   │◄─────────────│ render effect       │─────────►│ node │
  │ .value│   (track)    │ computed (lazy)     │          └──────┘
  └───┬───┘              │ watch / watchEffect │
      │   write          └──────────▲──────────┘
      └────── trigger ──────────────┘
          (only the effects that read it)
```

> **vs. React.** React re-runs the whole component function on every state change and reconciles the result; you control the blast radius with `memo`/`useMemo` and referential identity. Vue records *which* properties each effect read and re-runs only those effects; the identity of a callback or object is almost never the issue. This is why Vue has no dependency arrays and (mostly) no memoisation ceremony — and why "it's not updating" in Vue usually means *you broke the tracking link*, not *you forgot to memoise*.

[↑ Back to top](#table-of-contents)

---

<a id="5-the-35--36-vapor-delta"></a>

## 5. The 3.5 → 3.6 / Vapor Delta

**Status as of September 6, 2026:** stable is **3.5.42**; **3.6 is in RC** (v3.6.0-rc.7) ([vuejs/core releases](https://github.com/vuejs/core/releases)). Everything taught in this set is 3.5 API and continues to work in 3.6.

### What 3.5 changed that older material still gets wrong

These landed in 3.5 ([Vue 3.5 announcement](https://blog.vuejs.org/posts/vue-3-5)) and are new enough that a lot of tutorials still show the older form. Recognise both:

| Feature | Pre-3.5 form you'll see in the wild | 3.5+ form taught here |
|---|---|---|
| **Reactive props destructure** | `const props = defineProps<Props>()` then always `props.foo`; or `toRefs(props)` | `const { foo } = defineProps<Props>()` — destructured bindings stay reactive, compiled to `props.foo` access |
| **Props defaults** | `withDefaults(defineProps<Props>(), { msg: 'hi' })` | `const { msg = 'hi' } = defineProps<Props>()` — plain JS defaults. `withDefaults` still works |
| **Template refs** | `const input = ref(null)` whose *variable name must match* the `ref="input"` attribute | `const input = useTemplateRef('input')` — string ID, works with dynamic names ([Template Refs](https://vuejs.org/guide/essentials/template-refs.html)) |
| **Deep watch depth** | `deep: true` (all or nothing) | `deep: <number>` for bounded traversal ([Watchers](https://vuejs.org/guide/essentials/watchers.html)) |
| **Watcher cleanup** | the `onCleanup` argument passed into the callback | `onWatcherCleanup()`, callable anywhere in the callback |
| **SSR-safe IDs** | hand-rolled counters that caused hydration mismatches | `useId()` |
| **Lazy hydration** | not available | `hydrateOnVisible()` etc. on `defineAsyncComponent` ([Async Components](https://vuejs.org/guide/components/async.html)) |
| **Late teleport targets** | `nextTick` gymnastics | `<Teleport defer>` |

3.5 also refactored the reactivity system internally: **−56% memory** and up to **10× faster** on large deeply-reactive arrays, with **no behaviour change** ([Vue 3.5 announcement](https://blog.vuejs.org/posts/vue-3-5)). Nothing you write differs — but if you read a blog post claiming deep reactivity is unusably slow on big arrays, check its date.

### Already removed — name it so you recognise it

**Reactivity Transform** (`$ref()`, `$computed()`, `$()`, `$$()`) was **removed in Vue 3.4** ([Reactivity Transform](https://vuejs.org/guide/extras/reactivity-transform.html)). If you find a tutorial using `let count = $ref(0)`, it is dead in Vue core; the feature moved to the third-party Vue Macros plugin. Use plain `ref()` and `.value`.

### Vue 3.6 and Vapor Mode

**Vapor Mode** is a second compilation strategy: components compiled in Vapor mode skip the virtual DOM entirely and compile templates straight to DOM operations — no vnodes, no diffing, smaller runtime. It is opt-in and coexists with VDOM components in the same app. As of the RC it is feature-complete apart from `<Suspense>` support.

**Stance taken by this set:** 3.5 VDOM behaviour is the spine, because that is what stable ships and what every existing codebase runs. Vapor is taught in the Mastery tier of [Part 13](./vue-internals-performance.md) at a "know what it is, know what changes, know why it's fast" level. When 3.6 goes stable, that is the first section to rewrite — see the update note in [About This Document](#about-this-document).

[↑ Back to top](#table-of-contents)

---

## 6. Deliberately Not Covered (and where to look instead)

Per the repo's depth-over-breadth rule, these are genuinely important and deliberately absent rather than half-covered:

| Topic | Why it's out | Where to look |
|---|---|---|
| **Vue Router** | Its own topic; a routing doc that isn't 40% Vue would serve you better | [router.vuejs.org](https://router.vuejs.org) |
| **Pinia** | Own topic. Part 8 shows how far provide/inject genuinely gets you first | [pinia.vuejs.org](https://pinia.vuejs.org) |
| **Nuxt** | A framework on top of Vue, with its own routing/data/SSR model | [nuxt.com](https://nuxt.com) |
| **Testing** | Vitest + Vue Test Utils is a doc-sized topic on its own | [test-utils.vuejs.org](https://test-utils.vuejs.org) |
| **Vite / build config** | Adjacent, not Vue | [`legacy/vite/vite-mastery-guide.md`](../legacy/vite/vite-mastery-guide.md) (pending re-write) |
| **Full SSR / hydration** | Only the parts that shape the core API (`useId`, lazy hydration, `onMounted`) appear here | [SSR guide](https://vuejs.org/guide/scaling-up/ssr.html) |
| **Options API** | Superseded style; taught only as a reading aid in [§9](#9-cross-doc-cheat-sheet) | [Options API reference](https://vuejs.org/api/options-state.html) |
| **Render functions / JSX** | A legitimate escape hatch, but templates are what the compiler optimises and what you'll read | [Render Function APIs](https://vuejs.org/api/render-function.html) |
| **Custom elements / Web Components** | Narrow use case | [Vue and Web Components](https://vuejs.org/guide/extras/web-components.html) |
| **Vue 2 → 3 migration** | Historical; this set assumes Vue 3 from scratch | [v3-migration.vuejs.org](https://v3-migration.vuejs.org) |

[↑ Back to top](#table-of-contents)

---

## 7. Suggested Learning Order

**If you have never written Vue** — go in file order; the dependencies are real:

1. [§3 Setup](#3-orientation--setup) and [§4 mental model](#4-the-one-paragraph-mental-model) (15 min)
2. [Part 1 — Refs, Reactive & Computed](./vue-reactivity.md) — Beginner + Working Knowledge only
3. [Parts 3–4 — Templates, Directives, Events & Forms](./vue-templates-directives.md) — you can now build a page
4. [Part 5 — Components & `<script setup>`](./vue-components-script-setup.md) — you can now build an app
5. [Part 7 — Slots](./vue-slots-injection-watchers.md) and [Part 9 — Watchers & Lifecycle](./vue-slots-injection-watchers.md)
6. [Part 10 — Composables](./vue-composables-builtins.md) — the payoff of the Composition API
7. Then return to [Part 2](./vue-reactivity.md) and [Part 8](./vue-slots-injection-watchers.md) at Advanced tier, and take [Parts 12–13](./vue-internals-performance.md) whole

**If you know React** — the ordering changes, because your instincts will fight you in one specific place:

1. [§4 mental model](#4-the-one-paragraph-mental-model), then **all of [Parts 1 and 2](./vue-reactivity.md)**, including Mastery. Do not skim reactivity. Nearly every React-developer Vue bug is a tracking bug.
2. [Part 10 — Composables](./vue-composables-builtins.md), specifically the "these are not hooks" section — the Rules of Hooks do not apply, and assuming they do produces strange code.
3. [Part 5](./vue-components-script-setup.md) for the macro surface, then [Part 3](./vue-templates-directives.md) skimmed for directive syntax.
4. [Part 12](./vue-internals-performance.md) to see what the compiler does that a runtime-only library cannot.

**As a reference** — jump straight to the Part, then the tier. Every Part ends with a cheat-sheet table; [§9](#9-cross-doc-cheat-sheet) below is the cross-doc index.

[↑ Back to top](#table-of-contents)

---

## 8. Quick Self-Check

You understand this set if you can answer these without looking. Each links to where it's answered.

1. Why does `const { count } = reactive({ count: 0 })` break, while `const { count } = defineProps<{count: number}>()` works in 3.5? → [Part 1](./vue-reactivity.md), [Part 5](./vue-components-script-setup.md)
2. What exactly does ref unwrapping do in a template, and where does it *not* happen? → [Part 1](./vue-reactivity.md)
3. Why does `watch(obj.count, cb)` never fire, and what are the two correct forms? → [Part 9](./vue-slots-injection-watchers.md)
4. What's the difference between `flush: 'pre'`, `'post'` and `'sync'`, and when do you need `'post'`? → [Part 9](./vue-slots-injection-watchers.md)
5. Why is `v-if` on the same element as `v-for` a mistake, and what are the two fixes? → [Part 3](./vue-templates-directives.md)
6. What does `key` actually do during patching, and why does an index key corrupt input state? → [Part 3](./vue-templates-directives.md), [Part 12](./vue-internals-performance.md)
7. What does `defineModel()` compile to? → [Part 5](./vue-components-script-setup.md)
8. Why can a `<script setup>` component not be poked at through a template ref by default? → [Part 5](./vue-components-script-setup.md)
9. What makes a composable a composable, and which React hook rules do *not* apply? → [Part 10](./vue-composables-builtins.md)
10. Why is a scoped slot the only way for parent-written markup to see child state? → [Part 7](./vue-slots-injection-watchers.md)
11. What is a patch flag, and what is a block? → [Part 12](./vue-internals-performance.md)
12. When does `shallowRef` beat `ref`, and what must you promise in exchange? → [Part 2](./vue-reactivity.md), [Part 13](./vue-internals-performance.md)
13. Why does scoped CSS still reach a child component's root element? → [Part 11](./vue-composables-builtins.md)
14. What does Vapor Mode remove, and what does it cost? → [Part 13](./vue-internals-performance.md)

[↑ Back to top](#table-of-contents)

---

## 9. Cross-Doc Cheat Sheet

### Core reactivity API

| API | Use it for | Doc |
|---|---|---|
| `ref(v)` | **Default** for all state, primitive or object | [Part 1](./vue-reactivity.md) |
| `reactive(obj)` | Object-only state you never replace wholesale | [Part 1](./vue-reactivity.md) |
| `computed(fn)` | Derived value — cached and lazy | [Part 1](./vue-reactivity.md) |
| `readonly(x)` | Handing state down without letting it be mutated | [Part 8](./vue-slots-injection-watchers.md) |
| `toRef` / `toRefs` | Keeping a reactive link across destructuring | [Part 1](./vue-reactivity.md) |
| `toValue(x)` | Normalising ref \| getter \| raw inside a composable | [Part 10](./vue-composables-builtins.md) |
| `shallowRef` / `shallowReactive` | Big or externally-owned data | [Part 2](./vue-reactivity.md) |
| `triggerRef` / `customRef` | Manual trigger; debounced or validated refs | [Part 2](./vue-reactivity.md) |
| `toRaw` / `markRaw` | Escaping the proxy — class instances, node maps | [Part 2](./vue-reactivity.md) |
| `effectScope` / `onScopeDispose` | Grouped effect disposal outside components | [Part 2](./vue-reactivity.md), [Part 10](./vue-composables-builtins.md) |

### `<script setup>` macros

| Macro | Since | Doc |
|---|---|---|
| `defineProps` | 3.0 (reactive destructure 3.5) | [Part 5](./vue-components-script-setup.md) |
| `defineEmits` | 3.0 (tuple syntax 3.3) | [Part 5](./vue-components-script-setup.md) |
| `defineModel` | 3.4 | [Part 5](./vue-components-script-setup.md) |
| `defineExpose` | 3.2 | [Part 5](./vue-components-script-setup.md) |
| `defineOptions` | 3.3 | [Part 5](./vue-components-script-setup.md) |
| `defineSlots` | 3.3 | [Part 7](./vue-slots-injection-watchers.md) |
| `withDefaults` | 3.0 (largely superseded by 3.5 destructure defaults) | [Part 5](./vue-components-script-setup.md) |

### Options API → Composition API translation

You will meet the left column in older codebases and older answers. This table exists so you can *read* it — this set does not teach it.

| Options API | Composition API (`<script setup>`) |
|---|---|
| `data() { return { x: 1 } }` | `const x = ref(1)` |
| `computed: { y() {} }` | `const y = computed(() => …)` |
| `methods: { f() {} }` | `function f() {}` |
| `watch: { x(n, o) {} }` | `watch(x, (n, o) => {})` |
| `created()` | the body of `<script setup>` |
| `mounted()` | `onMounted(() => {})` |
| `beforeUnmount()` | `onBeforeUnmount(() => {})` |
| `errorCaptured()` | `onErrorCaptured(() => {})` |
| `props: { … }` | `defineProps<{ … }>()` |
| `emits: [ … ]` | `defineEmits<{ … }>()` |
| `provide` / `inject` options | `provide(key, v)` / `inject(key)` |
| `this.$refs.x` | `useTemplateRef('x')` |
| `mixins: [ … ]` | composables — [Part 10](./vue-composables-builtins.md) |
| `activated()` / `deactivated()` | `onActivated` / `onDeactivated` |

### Where each Part's cheat sheet lives

| Parts | Cheat sheet |
|---|---|
| 1–2 Reactivity | [end of `vue-reactivity.md`](./vue-reactivity.md#cheat-sheet) |
| 3–4 Templates & Forms | [end of `vue-templates-directives.md`](./vue-templates-directives.md#cheat-sheet) |
| 5–6 Components | [end of `vue-components-script-setup.md`](./vue-components-script-setup.md#cheat-sheet) |
| 7–9 Slots, Inject, Watchers | [end of `vue-slots-injection-watchers.md`](./vue-slots-injection-watchers.md#cheat-sheet) |
| 10–11 Composables & Built-ins | [end of `vue-composables-builtins.md`](./vue-composables-builtins.md#cheat-sheet) |
| 12–13 Internals & Performance | [end of `vue-internals-performance.md`](./vue-internals-performance.md#cheat-sheet) |

[↑ Back to top](#table-of-contents)
