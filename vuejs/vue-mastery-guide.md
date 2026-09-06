# Mastering Vue 3 — A Hands-On Reference Guide

## About This Document

- **Framing:** Curiosity-driven. No upcoming task is driving this — the goal is durable, hands-on understanding of Vue itself. Structured as a guided tutorial with runnable examples/exercises meant to actually be tried in a scaffolded project, with theory ("why it's designed this way") sprinkled in only where it explains a mechanism — e.g. *why* `reactive()` loses reactivity on destructure — not as standalone history.
- **Scope:** **Core Vue 3 + Composition API only** — reactivity, template syntax and directives, `<script setup>` and its compiler macros, slots, provide/inject, watchers/lifecycle, composables, built-in components (Transition/Teleport/Suspense/KeepAlive), SFC styling, and enough rendering-internals to explain *why* the API behaves the way it does. Explicitly **out of scope**: Vue Router, Pinia, Nuxt, testing, and the Vite/build-tool layer — those get their own docs, consistent with how this repo splits topics (see [`vite/vite-mastery-guide.md`](../vite/vite-mastery-guide.md)). SSR appears only where it leaks into core API design (`useId`, lazy hydration, `onMounted`-vs-server) — full SSR is its own topic. The Options API is not taught — Composition API with `<script setup lang="ts">` is the only style used, per current Vue team guidance and this repo owner's stated preference.
- **Depth tiers:** **Beginner → Working Knowledge → Advanced → Mastery**, collapsed where a topic doesn't have four honest tiers (e.g. `defineOptions`/`defineSlots` stay at Working Knowledge — there's no deeper tier worth forcing; SFC styling is Working Knowledge only).
- **Version basis:** **Vue 3.5** (stable line — [v3.5.42](https://github.com/vuejs/core/releases) is the latest stable release on npm as of September 2026) — [Vue 3.5 announcement](https://blog.vuejs.org/posts/vue-3-5). **Vue 3.6** is still in **RC** as of this writing (v3.6.0-rc.7), shipping **Vapor Mode** (a virtual-DOM-free compilation strategy) as an opt-in, experimental feature — covered at a "know it exists and why" level in the Mastery tier of Part 10, not taught as the default since it isn't stable yet. Anywhere 3.4-and-earlier behaved differently (e.g. `defineProps` destructure needing `withDefaults`), it's flagged explicitly, with a **Since** column in the macro cheat sheet.
- **Assumed background:** Comfortable with modern JS/TS (ES modules, destructuring, `async`/`await`), and has used *some* frontend framework or at least vanilla DOM APIs before — this is not a "what is a browser" doc. No prior Vue 2 / Options API experience assumed or required.
- **Per-section ingredients:** plain-language explanation everywhere; **wrong-vs-right pairs** only where a real gotcha exists; **Try It** exercises (curiosity framing) and occasional **Real Scenario** blocks where a production-shaped story explains the API better than an abstract one; ASCII diagrams only where the concept is structural (effect tracking, the injection chain, patch flags).
- **Sourcing standard:** [vuejs.org](https://vuejs.org) official docs are the primary source, cited inline. Stack Overflow used for real-world gotchas. Where a popular convention disagrees with current docs (e.g. "always use `reactive()` for objects", "composables follow React's rules of hooks"), the official position is stated and the folk convention named as wrong. Checked as of September 2026.
- **Update instructions for future-me:** Match this structure (numbered Parts, tier subheadings, wrong-vs-right pairs for gotchas, TOC + back-to-top links, cheat sheets at the end). If Vapor Mode reaches stable, promote Part 10's Vapor coverage out of "Mastery/experimental", re-check whether it changes any Working Knowledge guidance (`shallowRef` advice, custom directives — Vapor supports a subset of existing APIs), and update the Version basis bullet.

---

## Table of Contents

1. [Orientation — What Vue Actually Is](#1-orientation--what-vue-actually-is)
2. [Part 1 — The Reactivity System](#2-part-1--the-reactivity-system)
3. [Part 2 — Template Syntax & Rendering Directives](#3-part-2--template-syntax--rendering-directives)
4. [Part 3 — Components & `<script setup>`](#4-part-3--components--script-setup)
5. [Part 4 — Slots & Content Distribution](#5-part-4--slots--content-distribution)
6. [Part 5 — Provide / Inject](#6-part-5--provide--inject)
7. [Part 6 — Watchers, Lifecycle & Error Handling](#7-part-6--watchers-lifecycle--error-handling)
8. [Part 7 — Composables](#8-part-7--composables)
9. [Part 8 — Built-in Components](#9-part-8--built-in-components)
10. [Part 9 — Directives & SFC Styling](#10-part-9--directives--sfc-styling)
11. [Part 10 — Rendering Internals, Performance & What's Next](#11-part-10--rendering-internals-performance--whats-next)
12. [Cheat Sheets](#12-cheat-sheets)
13. [Suggested Learning Order](#13-suggested-learning-order)
14. [Quick Self-Check](#14-quick-self-check)

---

## 1. Orientation — What Vue Actually Is

**What it is.** Vue is a component framework built around one core idea: you declare state, you declare a template that reads that state, and Vue keeps the DOM in sync automatically when the state changes. The "how" of that sync — a reactivity system that tracks *which* pieces of the DOM depend on *which* pieces of state, so only the necessary parts re-render — is the single mechanism that everything else in this doc builds on.

**Why it matters.** Contrast with manual DOM manipulation: without a framework, *you* track what changed and *you* write the code that updates the right element. Vue's reactivity system does that tracking for you, automatically, at the granularity of individual reactive values — which is why understanding reactivity first makes every other Vue API (computed, watch, props, components) look like a natural consequence rather than a pile of unrelated rules to memorize.

**Try it now — scaffold something to follow along with:**

```bash
npm create vue@latest my-vue-app
# When prompted: TypeScript = Yes, everything else = No (keep it minimal for now)
cd my-vue-app
npm install
npm run dev
```

Open the printed `localhost` URL and open `src/App.vue`. Everything in Parts 1–7 can be tried directly inside this file.
([Quick Start — Vue.js](https://vuejs.org/guide/quick-start))

### The application root — the one non-component piece

Everything else in this guide happens *inside* components. The entry file is the exception, and it's worth reading once so the global knobs aren't a mystery later:

```ts
// src/main.ts
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

app.config.errorHandler = (err, instance, info) => {
  // Last-resort catch for errors thrown in any component, watcher, or lifecycle hook.
  // Wire this to your error reporter — see Part 6 for the per-component counterpart.
  reportToErrorTracker(err, { info })
}

app.provide('apiBaseUrl', '/api/v2') // app-level provide — see Part 5

app.mount('#app')
```

An "app instance" is a scope: `app.config`, `app.provide()`, `app.component()` (global registration), and `app.use()` (plugins) all apply to that one tree. You can create more than one app instance on the same page, each with its own config — which is how Vue gets embedded piecemeal into a non-Vue page.
([Application API — Vue.js](https://vuejs.org/api/application))

[⬆ back to top](#table-of-contents)

---

## 2. Part 1 — The Reactivity System

Covers: `ref`, `reactive`, `computed`, `readonly`, the shallow family, ref-unwrapping rules, and *why* each exists.

### Beginner

A `ref` wraps a value in an object with a `.value` property. Reading `.value` inside a template or a `computed`/`watchEffect` registers that spot as a *dependency*; writing `.value` triggers everything that depends on it to re-run.

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
const increment = () => count.value++
</script>

<template>
  <!-- No .value needed in the template — Vue unwraps top-level refs automatically -->
  <button @click="increment">Count: {{ count }}</button>
</template>
```

**Try it:** add a second `<p>{{ count * 2 }}</p>` in the template. Click the button — both update, because the template re-evaluates its whole expression tree whenever any ref it reads changes.
([Reactivity Fundamentals — Vue.js](https://vuejs.org/guide/essentials/reactivity-fundamentals))

**The unwrapping rules, stated once so they stop being surprising:**

| Where | `.value` needed? |
|---|---|
| Anywhere in `<script setup>` | **Yes**, always |
| Top-level binding in `<template>` | No — auto-unwrapped |
| Ref nested in a plain object read from a template (`wrapper.count`) | **Yes** — only *top-level* bindings unwrap |
| Ref as a property of a `reactive()` object | No — `reactive` unwraps refs on property access |
| Ref inside a plain array, `Map`, or `Set` | **Yes** — collections never unwrap |

```vue
<script setup lang="ts">
const count = ref(0)
const wrapper = { count }               // plain object holding a ref
const state = reactive({ count })       // reactive object holding a ref
const list = reactive([ref(1)])         // array — no unwrapping
</script>

<template>
  {{ count }}          <!-- 0   — top-level ref, unwrapped -->
  {{ wrapper.count }}  <!-- [object Object] — needs wrapper.count.value -->
  {{ state.count }}    <!-- 0   — reactive() unwraps its ref properties -->
  {{ list[0] }}        <!-- [object Object] — array elements never unwrap -->
</template>
```

That table is the source of about half of all "why is my template printing `[object Object]`" questions. The rule underneath it: unwrapping is a *convenience applied at specific boundaries* (template top-level bindings, `reactive` property access), not a property of refs themselves.
([Caveat when unwrapping in templates — Vue.js](https://vuejs.org/guide/essentials/reactivity-fundamentals#caveat-when-unwrapping-in-templates))

### Working Knowledge

**`reactive()` vs `ref()`.** `reactive()` wraps an *object* (not primitives) in a Proxy, so every property access is reactive without `.value`. The catch: destructuring a `reactive()` object gives you plain, disconnected values — the reactivity lives on the Proxy, not on the values pulled out of it.

**Wrong vs. right — losing reactivity on destructure:**

```ts
// Wrong — count is now a plain number, frozen at whatever it was at destructure time
const state = reactive({ count: 0 })
const { count } = state
count++ // does nothing to state.count, and doesn't trigger anything

// Right — use toRefs() to destructure while keeping each property reactive
import { toRefs } from 'vue'
const state = reactive({ count: 0 })
const { count } = toRefs(state)
count.value++ // updates state.count, template re-renders
```

`reactive()` has three further limitations worth knowing before reaching for it:

1. **Primitives don't work** — `reactive(0)` returns the number unchanged (with a dev warning).
2. **Reassignment breaks the connection** — `state = reactive({...})` points the local variable at a *new* proxy; anything that captured the old one still watches the old one.
3. **Proxy identity ≠ raw identity** — `reactive(obj) !== obj`, so `someReactiveArray.includes(rawObj)` can return `false` unexpectedly. Use `toRaw()` when you must compare against the original.

The guidance the docs now give, and the one this guide follows: **default to `ref()` for everything**, and treat `reactive()` as a niche tool for a locally-scoped object you never destructure or reassign. The older folk convention "`ref` for primitives, `reactive` for objects" predates that and causes more bugs than it prevents.

**Real Scenario.** One of the most common Vue 3 bugs on Stack Overflow: a composable returns `reactive({ x, y })`, a component destructures it for convenience (`const { x, y } = useMouse()`), and the values silently stop updating with no error thrown. This is exactly why the [composables guide](https://vuejs.org/guide/reusability/composables#return-values) says composables should return a plain object of `ref`s, never a `reactive()` object — refs survive destructuring because the reactive *connection* lives in the ref object itself, not in a Proxy wrapper you can strip away.
([Reactivity: `reactive()` — Vue.js](https://vuejs.org/guide/essentials/reactivity-fundamentals#reactive) · [Stack Overflow: Vue 3 reactive object loses reactivity on destructure](https://stackoverflow.com/questions/64891659))

**`computed()`.** Derives a value from other reactive state, and — critically — **caches** it. A computed only re-evaluates when one of its dependencies changes; reading it repeatedly between changes returns the cached value instantly.

```ts
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2) // read-only by default

// Writable computed — rare, but useful for two-way derived state
const plusOne = computed({
  get: () => count.value + 1,
  set: (val) => { count.value = val - 1 }
})
```

**Wrong vs. right — computed vs. a method:**

```vue
<script setup lang="ts">
const list = ref([/* thousands of items */])

// Wrong — recalculates on every single render, even if list hasn't changed
function expensiveFilter() {
  return list.value.filter(x => x.active)
}

// Right — cached; only re-runs when list.value actually changes
const filtered = computed(() => list.value.filter(x => x.active))
</script>

<template>
  <!-- Wrong usage calls the function on every re-render, for any reason -->
  <div v-for="item in expensiveFilter()" :key="item.id">{{ item.name }}</div>
  <!-- Right usage reads the cached computed -->
  <div v-for="item in filtered" :key="item.id">{{ item.name }}</div>
</template>
```

**Wrong vs. right — side effects inside a computed getter:**

```ts
// Wrong — a getter must be pure. This fires a request during render, re-runs on
// any dependency change, and can loop forever if the fetch writes back into a dep.
const user = computed(() => {
  fetchAnalytics(id.value)      // side effect
  return users.value[id.value]
})

// Right — derive in computed, act in watch
const user = computed(() => users.value[id.value])
watch(user, (u) => fetchAnalytics(u.id))
```

Two more computed rules that bite people: **don't mutate the returned value** (mutating a computed's result mutates the underlying source, bypassing the setter), and remember dependencies are tracked *only along the path actually taken* — an early `return` that skips reading `b.value` means changes to `b` won't retrigger the computed until some run reads it.
([Computed Properties — Vue.js](https://vuejs.org/guide/essentials/computed))

### Advanced

**The shallow family and the escape hatches.**

| API | Tracks | Use when |
|---|---|---|
| `shallowRef(v)` | Reassignment of `.value` only | Large payloads you replace wholesale |
| `shallowReactive(o)` | Root-level properties only | Object whose nested values are big and never individually rendered |
| `triggerRef(r)` | — (manual trigger) | You deliberately mutated inside a `shallowRef` |
| `markRaw(o)` | Nothing, ever | Third-party class instances (a chart, a map, a socket) that must not be proxied |
| `toRaw(p)` | — (unwraps) | Identity comparison, or handing an object to a non-Vue library |

```ts
import { shallowRef, triggerRef, markRaw, ref } from 'vue'

const data = shallowRef({ items: [] as string[] })

data.value.items.push('new') // does NOT trigger reactivity — mutated a nested property
data.value = { ...data.value, items: [...data.value.items, 'new'] } // DOES trigger — reassigned .value

// Escape hatch: force a re-render after a deliberate in-place mutation
data.value.items.push('another')
triggerRef(data)

// markRaw — a Leaflet map or Chart.js instance stored in a plain ref() gets deeply
// proxied, which breaks the library's internal identity checks and tanks performance
const map = ref(markRaw(new LeafletMap(el)))
```

**Real Scenario.** A dashboard renders a table from a 5,000-row API response stored in a `ref()`. Every incoming WebSocket update mutates one row deep inside that structure, and profiling shows the reactivity system re-wrapping the *entire* dataset's Proxy graph on every tick, even though only one row changed. Switching the top-level store to `shallowRef()` and replacing `.value` wholesale on each update (rather than mutating in place) removes the recursive-wrap cost entirely — the exact tradeoff the Vue team calls out in the performance guide. Caveat worth measuring first: Vue 3.5's reactivity rewrite made deeply reactive *arrays* up to 10× faster, so on current versions `ref()` is less often the bottleneck than blog posts from the 3.2 era suggest.
([`shallowRef()` — Vue.js API](https://vuejs.org/api/reactivity-advanced#shallowref) · [Reactivity in Depth — Vue.js](https://vuejs.org/guide/extras/reactivity-in-depth))

**`readonly()`.** Wraps a reactive object so mutations are blocked (with a dev-mode warning) while reads stay reactive. Commonly used to expose store state to components that shouldn't mutate it directly.

```ts
import { reactive, readonly } from 'vue'

const state = reactive({ count: 0 })
const publicState = readonly(state)

publicState.count++ // dev warning, mutation blocked — mutate via an exposed action instead
```

It's a *deep* readonly proxy (`shallowReadonly()` covers one level), and the block is a dev-time warning rather than a production throw — a design guardrail, not a security boundary.
([`readonly()` — Vue.js API](https://vuejs.org/api/reactivity-core#readonly))

**`toRef` / `toValue` / `customRef`.** The three utilities worth knowing by name:

```ts
import { toRef, toValue, customRef } from 'vue'

// toRef — a writable ref bound to one property of a reactive source
const emailRef = toRef(state, 'email')       // emailRef.value = 'x' writes state.email
// 3.3+ normalizing form: value | ref | getter -> ref
const idRef = toRef(() => props.id)          // read-only ref that follows props.id

// toValue — the inverse: value | ref | getter -> plain value (see Part 7)
toValue(idRef)  // number

// customRef — full control over track/trigger; the canonical use is debouncing
function useDebouncedRef<T>(value: T, delay = 300) {
  let timeout: ReturnType<typeof setTimeout>
  return customRef<T>((track, trigger) => ({
    get() { track(); return value },
    set(newValue) {
      clearTimeout(timeout)
      timeout = setTimeout(() => { value = newValue; trigger() }, delay)
    }
  }))
}
```

`customRef` is the clearest window into the whole system: you're handed `track` and `trigger` directly and decide when to call them. Everything else in this Part is a preset arrangement of those two calls.
([Reactivity Utilities — Vue.js API](https://vuejs.org/api/reactivity-utilities) · [`customRef()`](https://vuejs.org/api/reactivity-advanced#customref))

### Mastery — how it actually works

Vue 3's reactivity is built on **ES `Proxy`** objects, not `Object.defineProperty` getters/setters like Vue 2. Each reactive object is wrapped in a Proxy whose `get` trap calls `track()` ("the currently-running effect depends on this property") and whose `set` trap calls `trigger()` (re-run every effect tracked against that property). A `ref` is just a small object whose `.value` getter/setter does the same track/trigger dance for a single value — which is *why* refs need `.value` (a plain variable has no getter/setter to hook into) while `reactive()` properties don't (the Proxy intercepts property access directly).

```
Effect Tracking (simplified)

  computed(() => count.value * 2)
       │
       │ 1. runs the getter, reads count.value
       ▼
  count's dep set  ──tracks──▶  [ this computed's effect ]

  count.value = 5
       │
       │ 2. setter fires trigger()
       ▼
  every tracked effect for count re-runs  ──▶  computed recalculates, template re-renders
```

Three consequences fall out of this design, and they explain most "my reactivity broke" bugs:

1. **Tracking is synchronous, and only during the effect's own run.** A dependency read *after* an `await` inside a `watchEffect` is not tracked — the effect stack has already unwound. Read what you want tracked before the first `await`.
2. **Only property *access* is intercepted.** Adding or deleting a property on a `reactive()` object works fine (Proxy has `set`/`deleteProperty` traps, unlike Vue 2's `Vue.set` era), but a destructured plain value has no access to intercept — hence `toRefs`.
3. **Effects are batched.** Multiple synchronous writes queue a single re-render on the next microtask. That's what `nextTick()` is for: waiting until the DOM reflects the writes you just made.

```ts
import { nextTick } from 'vue'
count.value++
count.value++
console.log(el.textContent)   // still the old value — the render hasn't flushed yet
await nextTick()
console.log(el.textContent)   // updated
```

As of **Vue 3.5**, the reactivity core was rewritten for a ~56% memory reduction with no API changes ([Vue 3.5 announcement](https://blog.vuejs.org/posts/vue-3-5)), and **Vue 3.6** replaces the dependency-tracking engine with [`alien-signals`](https://github.com/vuejs/core/releases) — again with no public API change, since this machinery sits entirely behind `ref`/`reactive`/`computed`. A useful pattern to internalize: Vue has repeatedly rewritten reactivity internals across versions while keeping the surface API stable, which is exactly why understanding the *mental model* (track/trigger, not the specific data structure) is more durable than memorizing implementation details.

[⬆ back to top](#table-of-contents)

---

## 3. Part 2 — Template Syntax & Rendering Directives

Covers: interpolation, `v-bind`/`v-on` and their shorthands, `v-if` vs `v-show`, `v-for` and `key`, class/style bindings, `v-model` on native inputs, and the escape hatches (`v-html`, `v-pre`, `v-cloak`).

### Beginner

Templates are valid HTML with two additions: **interpolation** (`{{ }}`) for text, and **directives** (`v-` prefixed attributes) for everything else. Any *single JavaScript expression* is allowed inside a binding — statements, `if`, and variable declarations are not.

```vue
<template>
  <!-- Text interpolation — expression, not statement -->
  <p>{{ user.name.toUpperCase() }}</p>
  <p>{{ ok ? 'yes' : 'no' }}</p>
  <!-- Wrong: {{ const x = 1 }} and {{ if (ok) {...} }} are both compile errors -->

  <!-- v-bind: attribute binding, shorthand `:` -->
  <img :src="imageUrl" :alt="caption" :disabled="isDisabled" />

  <!-- v-on: event handling, shorthand `@` -->
  <button @click="count++">Inline statement</button>
  <button @click="handleClick">Method reference</button>
  <button @click="handleClick($event, item.id)">With arguments</button>
</template>
```

Two conveniences worth knowing early:

```vue
<!-- Same-name shorthand (3.4+): :id="id" can be written :id -->
<MyComponent :id :title />

<!-- Dynamic argument: the attribute/event name itself comes from state -->
<button :[attrName]="value" @[eventName]="handler">Dynamic</button>

<!-- Binding an object spreads it as multiple attributes -->
<div v-bind="{ id: 'x', class: 'y' }"></div>
```

A `null` or `undefined` bound value *removes* the attribute; a boolean attribute (`disabled`, `checked`) is present for truthy values and absent for falsy ones. That's why `:disabled="isDisabled"` behaves correctly without ternaries.
([Template Syntax — Vue.js](https://vuejs.org/guide/essentials/template-syntax))

### Working Knowledge

**Event modifiers** replace the boilerplate lines you'd otherwise write inside every handler:

```vue
<template>
  <form @submit.prevent="onSubmit">        <!-- event.preventDefault() -->
    <div @click.stop="onClick" />          <!-- event.stopPropagation() -->
    <div @click.self="onClick" />          <!-- only if event.target IS this element -->
    <div @click.once="onClick" />          <!-- fires at most once -->
    <div @scroll.passive="onScroll" />     <!-- { passive: true } — never blocks scrolling -->
    <div @click.capture="onClick" />       <!-- capture phase -->
    <input @keyup.enter="submit" @keyup.esc="cancel" />
    <input @keyup.ctrl.s.prevent="save" /> <!-- key + system modifiers, chained -->
  </form>
</template>
```

Order matters when chaining: `@click.prevent.self` prevents *all* clicks then filters, while `@click.self.prevent` only prevents clicks on the element itself.

**`v-if` vs `v-show`.** `v-if` truly adds and removes the element (and destroys/recreates the component, running lifecycle hooks); `v-show` always renders it and toggles `display: none`.

| | `v-if` | `v-show` |
|---|---|---|
| Initial cost when false | Nothing rendered | Rendered, then hidden |
| Toggle cost | Mount/unmount + hooks | One CSS property |
| Works on `<template>` groups | Yes | No |
| Right for | Rarely-toggled, expensive, or conditional-by-permission content | Frequently toggled UI (tabs, dropdowns) |

```vue
<template>
  <!-- v-if / v-else-if / v-else must be siblings with no elements between them -->
  <div v-if="status === 'loading'">Loading…</div>
  <div v-else-if="status === 'error'">Failed</div>
  <div v-else>{{ data }}</div>

  <!-- <template> as an invisible wrapper so v-if can guard a group -->
  <template v-if="isAdmin">
    <AdminBar />
    <AuditLog />
  </template>
</template>
```

**`v-for` and the `key` rule.** Always supply a stable, unique `:key` — Vue uses it to match old and new vnodes across a re-render. Without one (or with the index as key), Vue's default in-place patch reuses DOM nodes positionally, which silently corrupts any *DOM-local* state: input values, focus, scroll position, CSS transition state.

```vue
<template>
  <li v-for="(item, index) in items" :key="item.id">{{ index }}: {{ item.name }}</li>

  <!-- Objects: value, key, index — in that order -->
  <li v-for="(value, key, index) in myObject" :key="key">{{ key }}: {{ value }}</li>

  <!-- A number range starts at 1, not 0 -->
  <span v-for="n in 5" :key="n">{{ n }}</span>
</template>
```

**Wrong vs. right — `v-if` and `v-for` on the same element:**

```vue
<!-- Wrong — in Vue 3 v-if has HIGHER priority than v-for, so `item` isn't
     even defined yet when the condition is evaluated. (In Vue 2 the priority
     was reversed, which is why old blog posts show this pattern "working".) -->
<li v-for="item in items" v-if="item.active" :key="item.id">{{ item.name }}</li>

<!-- Right (preferred) — filter in a computed; the list is filtered once and cached -->
<li v-for="item in activeItems" :key="item.id">{{ item.name }}</li>

<!-- Right (when you need the raw list for other reasons) — move v-if inward -->
<template v-for="item in items" :key="item.id">
  <li v-if="item.active">{{ item.name }}</li>
</template>
```

**Wrong vs. right — index as key:**

```vue
<!-- Wrong — deleting the first row shifts every key; Vue reuses the DOM node
     positionally, so the text updates but the typed <input> value stays put
     and now belongs to the wrong row. -->
<li v-for="(todo, i) in todos" :key="i"><input v-model="todo.text" /></li>

<!-- Right — key by identity, so the removed node is the one that's removed -->
<li v-for="todo in todos" :key="todo.id"><input v-model="todo.text" /></li>
```

Index keys are only safe when the list is append-only and never reordered, filtered, or containing stateful children — which is rarely worth reasoning about, so key by ID as a habit.
([List Rendering — Vue.js](https://vuejs.org/guide/essentials/list) · [Conditional Rendering — Vue.js](https://vuejs.org/guide/essentials/conditional))

**Class and style bindings** get special treatment from `v-bind`: objects and arrays are supported, and a bound `class` is *merged* with the static `class` attribute rather than replacing it.

```vue
<template>
  <div class="base" :class="{ active: isActive, 'text-danger': hasError }" />
  <div :class="[activeClass, errorClass]" />
  <div :class="[isActive ? 'active' : '', 'always']" />

  <div :style="{ color: activeColor, fontSize: size + 'px' }" />
  <div :style="[baseStyles, overrideStyles]" />
  <!-- Auto-prefixing is applied; an array of values picks the last supported one -->
  <div :style="{ display: ['-webkit-box', 'flex'] }" />
</template>
```

Note that `class`/`style` bound on a *component* land on its root element via attribute fallthrough (Part 3), which is why `<MyButton class="mt-4" />` usually just works.
([Class and Style Bindings — Vue.js](https://vuejs.org/guide/essentials/class-and-style))

**`v-model` on native inputs** is sugar for a value binding plus an input listener, with the right pair chosen per element type (`value`/`input` for text, `checked`/`change` for checkboxes, `value`/`change` for `<select>`).

```vue
<template>
  <input v-model="text" />
  <input v-model.lazy="text" />      <!-- sync on `change`, not every keystroke -->
  <input v-model.number="age" />     <!-- parse to a number (falls back to the string) -->
  <input v-model.trim="name" />      <!-- trim whitespace -->

  <input type="checkbox" v-model="checked" />                 <!-- boolean -->
  <input type="checkbox" value="vue" v-model="frameworks" />  <!-- array membership -->
  <input type="radio" value="a" v-model="choice" />
  <select v-model="selected"><option value="a">A</option></select>

  <!-- Bind non-string values on checkboxes/radios -->
  <input type="checkbox" v-model="answer" :true-value="'yes'" :false-value="'no'" />
</template>
```

**Wrong vs. right — `v-model` on a prop:**

```vue
<!-- Wrong — props are read-only; this mutates the parent's data through the back
     door and logs "Set operation on key 'x' failed: target is readonly" -->
<input v-model="props.title" />

<!-- Right — defineModel gives you a writable ref wired to the parent (Part 3) -->
<script setup lang="ts">
const title = defineModel<string>()
</script>
<template><input v-model="title" /></template>
```
([Form Input Bindings — Vue.js](https://vuejs.org/guide/essentials/forms))

### Advanced

**The rendering escape hatches**, in decreasing order of how often you should use them:

```vue
<template>
  <!-- v-html: renders raw HTML. NEVER on user-supplied content — this is an
       XSS injection point by definition. Sanitize server-side, or don't use it. -->
  <div v-html="trustedMarkdownHtml" />

  <!-- v-text: equivalent to {{ }} but as an attribute; useful when the mustache
       would flash before hydration in an SSR page -->
  <span v-text="msg" />

  <!-- v-pre: skip compilation for this subtree — for displaying literal mustaches -->
  <span v-pre>{{ this is printed as-is }}</span>

  <!-- v-cloak: with `[v-cloak] { display: none }`, hides un-compiled templates.
       Only relevant for in-DOM templates on a CDN build — irrelevant in SFCs. -->
  <div v-cloak>{{ msg }}</div>
</template>
```

**Why templates, and not JSX?** Vue supports [render functions and JSX](https://vuejs.org/guide/extras/render-function) fully, and they're the right tool when a component's structure is genuinely programmatic (a renderer driven by a schema, a component that transforms its own slot vnodes). The reason templates stay the default: because a template's structure is statically analyzable, the compiler can hoist static subtrees and tag dynamic ones with patch flags before the app ever runs — an optimization that's impossible for arbitrary JSX, where any expression could produce any tree. Part 10 shows what that compiled output looks like.

**Try it:** paste a template into the [Vue SFC Playground](https://play.vuejs.org) and open the **Compiled Output → template** tab. Add a static `<p>hello</p>` next to a `<p>{{ count }}</p>` and watch which one gets hoisted out of the render function.

[⬆ back to top](#table-of-contents)
---

## 4. Part 3 — Components & `<script setup>`

Covers: SFC anatomy, `defineProps`, `defineEmits`, `defineModel`, `defineExpose`, attribute fallthrough, template refs, dynamic and async components, generics.

### Beginner

A Single-File Component (`.vue`) has up to three blocks: `<script setup>` (logic), `<template>` (markup), `<style>` (CSS, optionally `scoped`). `<script setup>` is compile-time sugar — every top-level binding is automatically exposed to the template, with no `return` statement needed.

```vue
<script setup lang="ts">
import { ref } from 'vue'
const message = ref('Hello Vue')
</script>

<template>
  <p>{{ message }}</p>
</template>

<style scoped>
p { color: teal; }
</style>
```

Imports work the same way: importing a `.vue` file makes it usable in the template with no registration step. Component names are matched PascalCase-to-kebab-case both ways, so `<MyButton />` and `<my-button />` both resolve — **prefer PascalCase in SFCs**, since it distinguishes components from native elements at a glance.

When you need code that runs *once at module scope* rather than per component instance, add a plain `<script>` block alongside — both compile into the same module:

```vue
<script lang="ts">
// Runs once when the module is imported, not per instance
const registry = new Map()
</script>

<script setup lang="ts">
// Runs per component instance
registry.set(useId(), true)
</script>
```
([`<script setup>` — Vue.js](https://vuejs.org/api/sfc-script-setup))

### Working Knowledge

**`defineProps`.** Type-based declaration is preferred over the runtime-object form — full IDE inference, no separate `PropType` import, and the compiler generates the runtime declaration for you.

```ts
// Type-based (recommended)
const props = defineProps<{
  title: string
  count?: number
}>()

// Vue 3.5+ — destructure directly, with plain JS defaults, and stay reactive
const { title, count = 0 } = defineProps<{
  title: string
  count?: number
}>()
```

**Wrong vs. right — prop defaults across versions:**

```ts
// Wrong on Vue 3.4 and earlier — plain destructure of defineProps() silently
// snapshots the value at setup time; the component never updates when the parent
// changes `count`. (Behind a flag in 3.3/3.4, stable and reactive from 3.5.)
const { title, count = 0 } = defineProps<{ title: string; count?: number }>()

// Right for Vue 3.4 and earlier — withDefaults wrapper, props accessed via `props.`
const props = withDefaults(defineProps<{
  title: string
  items?: string[]
}>(), {
  items: () => []   // factory function required for array/object defaults
})

// Right for Vue 3.5+ — destructure with inline defaults; no factory needed
const { title, items = [] } = defineProps<{ title: string; items?: string[] }>()
```

Why this is worth calling out: destructuring `defineProps()` *looks* like the `reactive()` destructure that breaks reactivity, and pre-3.5 it broke for the same reason. Vue 3.5 stabilized **reactive props destructure**, where the compiler rewrites every reference to a destructured prop back into `props.x` access at build time. It's compiler magic scoped specifically to `defineProps` — **don't generalize the pattern to `reactive()`**, which still breaks.

Two consequences of that compile-time rewrite:

```ts
const { count } = defineProps<{ count: number }>()

// Wrong — passing a destructured prop to a watcher passes a plain number,
// so the watcher never fires. Same trap as any getter-vs-value confusion.
watch(count, () => {})

// Right — wrap in a getter so the compiler's `props.count` access happens inside it
watch(() => count, () => {})
```

**Props are one-way and read-only.** Mutating `props.x` warns; when a child needs a locally editable copy, either derive it (`const local = computed(() => props.x)`) or, for a genuinely two-way binding, use `defineModel` below.
([`defineProps` — Vue.js](https://vuejs.org/api/sfc-script-setup#defineprops--defineemits) · [Reactive Props Destructure — Vue 3.5](https://blog.vuejs.org/posts/vue-3-5#reactive-props-destructure))

**`defineEmits`.** Named-tuple syntax gives typed payloads and doubles as documentation of the component's outward API:

```ts
const emit = defineEmits<{
  update: [value: string]
  change: [id: number, name: string]
  close: []
}>()

emit('update', 'new value')
```

Declaring an event also *removes* it from `$attrs`, so a declared `close` won't also fire a native listener bound on the root element — a subtle but real source of double-firing when it's left undeclared.

**`defineModel`** (3.4+). Replaces the `props: ['modelValue'] + emit('update:modelValue')` boilerplate with a single ref that reads *and* writes back to the parent.

```ts
// Child.vue
const model = defineModel<string>()                       // "modelValue" prop + "update:modelValue" emit
const count = defineModel<number>('count', { default: 0 }) // named model, v-model:count
const required = defineModel<string>({ required: true })

// Transform on the way in/out
const trimmed = defineModel<string>({
  set: (v) => v.trim()
})

// Modifiers: destructure the tuple form to read what the parent asked for
const [text, modifiers] = defineModel<string, 'capitalize'>({
  set(value) {
    return modifiers.capitalize ? value[0].toUpperCase() + value.slice(1) : value
  }
})
```

```vue
<!-- Parent.vue -->
<Child v-model="name" />
<Child v-model:count="total" />
<Child v-model.capitalize="name" />
```

One caveat: `defineModel` returns a **local ref that syncs**, not a direct window into the parent. If the parent doesn't bind `v-model` at all, the ref still works as plain local state — convenient, and occasionally surprising when a missing binding fails silently instead of erroring.
([`defineModel` — Vue.js](https://vuejs.org/api/sfc-script-setup#definemodel))

**`defineExpose`.** Components using `<script setup>` are **closed by default** — a parent holding a template ref to a child can't reach into its internals unless the child explicitly exposes them.

```ts
// Child.vue
const count = ref(0)
const reset = () => { count.value = 0 }
defineExpose({ count, reset })   // refs are unwrapped for the consumer
```

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import Child from './Child.vue'

// 3.5+: useTemplateRef('name') matches ref="name" in the template
const child = useTemplateRef<InstanceType<typeof Child>>('child')
</script>
<template>
  <Child ref="child" />
  <button @click="child?.reset()">Reset child</button>
</template>
```
([`defineExpose`](https://vuejs.org/api/sfc-script-setup#defineexpose) · [`useTemplateRef()`](https://vuejs.org/api/composition-api-helpers#usetemplateref))

**Template refs.** `ref="name"` on a plain element gives you the DOM node; on a component it gives the exposed instance. Since 3.5, `useTemplateRef(name)` is the preferred accessor — it works with dynamic ref names and doesn't depend on the variable name matching the attribute.

```vue
<script setup lang="ts">
import { ref, useTemplateRef, onMounted } from 'vue'

const input = useTemplateRef<HTMLInputElement>('input')   // 3.5+
const legacyInput = ref<HTMLInputElement | null>(null)     // pre-3.5: variable name must match ref=""

// A ref inside v-for collects an array — order is NOT guaranteed to match the source
const items = useTemplateRef<HTMLLIElement[]>('items')

onMounted(() => input.value?.focus())   // null before mount, null again after unmount
</script>

<template>
  <input ref="input" />
  <input ref="legacyInput" />
  <li v-for="i in list" :key="i.id" ref="items">{{ i.name }}</li>
</template>
```

Template refs are `null` until mount and are populated during the render flush, which is why reads belong in `onMounted` or a `flush: 'post'` watcher (Part 6) — never at the top level of `<script setup>`.
([Template Refs — Vue.js](https://vuejs.org/guide/essentials/template-refs))

**Attribute fallthrough and `$attrs`.** Any attribute a parent passes that isn't a declared prop or emit — `class`, `style`, `id`, `data-*`, native listeners — falls through onto the child's single root element automatically.

```vue
<!-- MyButton.vue: <button class="btn">…</button> -->
<!-- Parent: <MyButton class="mt-4" @click="save" /> -->
<!-- Result: <button class="btn mt-4"> with a real click listener. class merges, it doesn't replace. -->
```

Fallthrough silently *stops* when the component has multiple root nodes (fragments) — Vue warns and drops the attributes unless you bind them explicitly. That's the fix for the most common "why is my `class` not applying" case:

```vue
<script setup lang="ts">
defineOptions({ inheritAttrs: false })   // opt out of automatic root-element fallthrough
</script>

<template>
  <!-- Multi-root, or you want attrs on an inner element instead of the root -->
  <label>{{ label }}</label>
  <input v-bind="$attrs" />
</template>
```

In `<script setup>`, `$attrs` is available in the template directly; `useAttrs()` gives the same object in script. Note `$attrs` is **not reactive per-property** — you can't `watch()` an individual attr; it's re-evaluated on each update.
([Fallthrough Attributes — Vue.js](https://vuejs.org/guide/components/attrs))

**Lower-ceremony macros.** `defineOptions` (3.3+) sets component options without a second `<script>` block; `defineSlots` (3.3+) types slot props for IDE checking. Both are one-liners with no deeper tier worth documenting:

```ts
defineOptions({ inheritAttrs: false, name: 'UserCard' })

const slots = defineSlots<{
  default(props: { item: string }): any
  header?(): any
}>()
```

`useId()` (3.5+) rounds out the small helpers — it generates an app-unique ID that's stable across server and client renders, which is exactly what accessible form markup needs without risking a hydration mismatch:

```vue
<script setup lang="ts">
import { useId } from 'vue'
const id = useId()   // call at setup top level — never inside a computed()
</script>
<template>
  <label :for="id">Name</label>
  <input :id="id" />
</template>
```
([`defineOptions`](https://vuejs.org/api/sfc-script-setup#defineoptions) · [`defineSlots`](https://vuejs.org/api/sfc-script-setup#defineslots) · [`useId()`](https://vuejs.org/api/composition-api-helpers#useid))

### Advanced

**Dynamic components.** `<component :is>` renders whichever component (or HTML tag name) the expression resolves to — the basis of tab interfaces and schema-driven renderers. In `<script setup>`, pass the imported component itself, not a string name, since there's no component registry to look a string up in.

```vue
<script setup lang="ts">
import TabA from './TabA.vue'
import TabB from './TabB.vue'
const tabs = { a: TabA, b: TabB }
const current = ref<'a' | 'b'>('a')
</script>

<template>
  <component :is="tabs[current]" />
  <component :is="someCondition ? 'a' : 'span'" />  <!-- HTML tags work as strings -->
</template>
```

**Async components.** `defineAsyncComponent` turns a dynamic import into a component, so its code splits into its own chunk and loads on first render — the standard way to keep a heavy editor or chart library out of the initial bundle.

```ts
import { defineAsyncComponent, hydrateOnVisible } from 'vue'

const Chart = defineAsyncComponent(() => import('./HeavyChart.vue'))

// Full form, with loading/error UI
const Editor = defineAsyncComponent({
  loader: () => import('./RichTextEditor.vue'),
  loadingComponent: Spinner,
  delay: 200,          // ms before showing the spinner — avoids a flash on fast loads
  errorComponent: LoadFailed,
  timeout: 3000,       // after this, render errorComponent
  // 3.5+ lazy hydration (SSR only): keep the markup static until it's actually needed
  hydrate: hydrateOnVisible({ rootMargin: '100px' })
})
```

The 3.5 hydration strategies — `hydrateOnVisible()`, `hydrateOnIdle()`, `hydrateOnInteraction('click')`, `hydrateOnMediaQuery('(max-width: 500px)')`, `hydrateNever()` — are the core-Vue primitive behind "islands"-style SSR: the server sends HTML, and the JS for a given component only boots when its strategy says so. They're inert in a client-only SPA.

**Wrong vs. right — defining an async component inside setup:**

```ts
// Wrong — a new component definition on every render means Vue sees a different
// component type each time, unmounting and remounting (and re-fetching) constantly
const Comp = computed(() => defineAsyncComponent(() => import(`./${name.value}.vue`)))

// Right — define at module scope, or memoize per key
const registry = {
  chart: defineAsyncComponent(() => import('./Chart.vue')),
  table: defineAsyncComponent(() => import('./Table.vue'))
}
const Comp = computed(() => registry[name.value])
```
([Async Components — Vue.js](https://vuejs.org/guide/components/async))

**Generic components.** A component can declare a type parameter the same way a TS generic function would, useful for a typed `<Select>` or `<List>` that should preserve the caller's item type.

```vue
<script setup lang="ts" generic="T extends { id: string | number }">
defineProps<{
  items: T[]
  selected: T
}>()

defineEmits<{ select: [item: T] }>()

defineSlots<{
  row(props: { row: T }): any
}>()
</script>
```

**Real Scenario.** A `<DataTable>` is written once as `items: any[]`, then reused across a dozen pages with different row shapes — losing all autocomplete on `row.` inside the row-template slot, and with it every compile-time guarantee that the column definitions match the data. Converting it to `generic="T"` and typing `items: T[]` plus a scoped slot as `row(props: { row: T })` restores full inference at every call site with zero duplication. This is the single highest-leverage use of generics in real Vue codebases: reusable list/table/select components.
([Generics — Vue.js](https://vuejs.org/api/sfc-script-setup#generics))

[⬆ back to top](#table-of-contents)

---

## 5. Part 4 — Slots & Content Distribution

Covers: default and named slots, fallback content, scoped slots, conditional slots, and the renderless-component pattern.

### Beginner

A slot is a hole in a child's template that the parent fills. Props pass *data* down; slots pass *markup* down.

```vue
<!-- FancyCard.vue -->
<template>
  <div class="card">
    <slot>Nothing here yet</slot>   <!-- fallback content, used when the parent passes none -->
  </div>
</template>
```

```vue
<!-- Parent -->
<FancyCard>
  <p>Anything I want, including <OtherComponent /></p>
</FancyCard>
```

Slot content is compiled in the **parent's** scope, so it can read the parent's state and not the child's. That single rule explains why scoped slots (below) have to exist at all.
([Slots — Vue.js](https://vuejs.org/guide/components/slots))

### Working Knowledge

**Named slots** let a component define multiple insertion points, filled with `v-slot` (shorthand `#`):

```vue
<!-- BaseLayout.vue -->
<template>
  <header><slot name="header" /></header>
  <main><slot /></main>              <!-- name="default" implicitly -->
  <footer><slot name="footer">© 2026</slot></footer>
</template>
```

```vue
<BaseLayout>
  <template #header><h1>Page title</h1></template>
  <p>Body content goes to the default slot without a template wrapper.</p>
  <template #footer>Custom footer</template>

  <!-- Dynamic slot name -->
  <template #[dynamicName]>…</template>
</BaseLayout>
```

**Scoped slots** flip the direction: the child passes data *up* into the parent-authored markup, by binding props on the `<slot>` element.

```vue
<!-- ItemList.vue -->
<template>
  <li v-for="item in items" :key="item.id">
    <slot :item="item" :index="items.indexOf(item)" />
  </li>
</template>
```

```vue
<ItemList :items="users">
  <template #default="{ item, index }">
    {{ index }} — {{ item.name }}
  </template>
</ItemList>

<!-- Shorthand when default is the only slot -->
<ItemList :items="users" #default="{ item }">{{ item.name }}</ItemList>
```

**Conditional slots** — render the wrapper markup only when the parent actually supplied content, so an empty `<header>` doesn't leave a stray border on the page:

```vue
<template>
  <div class="card">
    <header v-if="$slots.header"><slot name="header" /></header>
    <slot />
  </div>
</template>
```

**Wrong vs. right — reaching for the child's scope in slot content:**

```vue
<!-- Wrong — `item` is defined in the CHILD, but slot content compiles in the
     PARENT's scope, so this is a "property is not defined" error at runtime -->
<ItemList :items="users">
  <p>{{ item.name }}</p>
</ItemList>

<!-- Right — the child must explicitly pass it out via slot props -->
<ItemList :items="users">
  <template #default="{ item }"><p>{{ item.name }}</p></template>
</ItemList>
```
([Scoped Slots — Vue.js](https://vuejs.org/guide/components/slots#scoped-slots))

### Advanced

**Renderless components.** A component whose template is nothing but a scoped slot: it owns logic and state, and hands both to the caller to render however they like.

```vue
<!-- MouseTracker.vue — renders nothing of its own -->
<script setup lang="ts">
const { x, y } = useMouse()
</script>
<template><slot :x="x" :y="y" /></template>
```

```vue
<MouseTracker #default="{ x, y }">Mouse is at {{ x }}, {{ y }}</MouseTracker>
```

This was the dominant reuse pattern in Vue 2 (and React's "render props"). **In Vue 3, prefer a composable** for pure logic — `useMouse()` has no component-instance overhead and composes freely, whereas nesting three renderless components creates a template pyramid. The pattern still earns its place when the reusable piece is *both* logic and markup structure: a headless `<Combobox>` that owns keyboard navigation and ARIA wiring while letting the caller supply every element is the canonical example.
([Renderless Components — Vue.js](https://vuejs.org/guide/components/slots#renderless-components))

**Typing slots** with `defineSlots` gives call sites real checking on slot props — the missing half of type-safe generic components:

```ts
defineSlots<{
  default(props: { item: T; index: number }): any
  empty?(): any
}>()
```

[⬆ back to top](#table-of-contents)

---

## 6. Part 5 — Provide / Inject

Covers: escaping prop drilling, injection keys, reactivity discipline, and where this stops being the right tool.

### Working Knowledge

`provide` / `inject` lets an ancestor supply a value to *any* descendant, at any depth, without threading props through every intermediate component.

```
   <App>            provide('theme', theme)
      │
   <Layout>         (knows nothing about theme)
      │
   <Sidebar>        (knows nothing about theme)
      │
   <ThemeToggle>    inject('theme')  ◀── resolves up the chain to the nearest provider
```

```ts
// Ancestor
import { provide, ref, readonly } from 'vue'

const theme = ref<'light' | 'dark'>('light')
const toggleTheme = () => { theme.value = theme.value === 'light' ? 'dark' : 'light' }

provide('theme', { theme: readonly(theme), toggleTheme })
```

```ts
// Any descendant
import { inject } from 'vue'

const { theme, toggleTheme } = inject('theme')!
const fallback = inject('optional', 'default value')       // default when unprovided
const expensive = inject('svc', () => new Service(), true) // 3rd arg: treat default as a factory
```

**Typed injection keys.** A plain string key gives you `unknown` on the way out and collides silently across a large app. `InjectionKey<T>` fixes both:

```ts
// keys.ts — shared module, so the provider and injector agree by import, not by spelling
import type { InjectionKey, Ref } from 'vue'

export interface ThemeContext {
  theme: Readonly<Ref<'light' | 'dark'>>
  toggleTheme: () => void
}
export const themeKey = Symbol('theme') as InjectionKey<ThemeContext>
```

```ts
provide(themeKey, { theme: readonly(theme), toggleTheme })
const ctx = inject(themeKey)   // typed as ThemeContext | undefined
```

**Wrong vs. right — providing a snapshot instead of reactive state:**

```ts
// Wrong — provides a plain number, captured once. The descendant renders 0 forever.
const count = ref(0)
provide('count', count.value)

// Wrong in a different way — mutable everywhere. Any descendant can write to it,
// so "who changed this?" becomes unanswerable in a large tree.
provide('count', count)

// Right — provide the reactive object, readonly, plus explicit mutation functions
provide('count', {
  count: readonly(count),
  increment: () => count.value++
})
```

That last shape — **readonly state + named actions, provided together** — is the discipline that keeps provide/inject maintainable. It's also, not coincidentally, the shape of a Pinia store; when the injected context starts needing devtools, persistence, or cross-page lifetime, that's the signal to graduate to a real store library (its own doc).

**App-level provide** covers values every component may need, and is how plugins expose themselves:

```ts
app.provide('apiBaseUrl', '/api/v2')
```

**Injection is resolved at component-instantiation time, walking *up the component tree***, not the DOM tree and not the route hierarchy. Two practical consequences: `inject()` must be called synchronously during `setup()` (not after an `await`, not in a click handler), and a nearer provider of the same key **shadows** a farther one — which is a feature (per-subtree theme overrides) as often as it is a surprise.
([Provide / Inject — Vue.js](https://vuejs.org/guide/components/provide-inject))

### Advanced — when *not* to reach for it

Provide/inject makes a component's dependencies invisible at the call site: nothing in `<ThemeToggle />` says it will explode without an ancestor providing `themeKey`. That's an acceptable trade for genuinely ambient concerns (theme, locale, form context, a design-system's density setting) and a bad one for ordinary data flow.

A short decision rule:

| Situation | Reach for |
|---|---|
| Parent → direct child | Props / emits |
| Parent → grandchild, 2–3 levels | Props — the explicitness is worth the typing |
| Ambient context consumed by unknown descendants | `provide` / `inject` |
| Sibling components, or state outliving the tree | A store (Pinia) |
| Reusable logic with no shared instance | A composable (Part 7) |

**Real Scenario.** A form library provides its validation context so that a `<FormField>` nested inside three layers of layout components can register itself without the app author wiring anything up. This works precisely because a field's dependency on "some enclosing form" is *conceptually* ambient. The same team then used provide/inject to pass the current user object down two levels of a settings page — and spent an afternoon debugging a page that rendered blank when someone reused the leaf component outside that page's provider. The distinction isn't depth, it's whether the dependency is structural or incidental.

[⬆ back to top](#table-of-contents)
---

## 7. Part 6 — Watchers, Lifecycle & Error Handling

Covers: `watch`, `watchEffect`, flush timing, cleanup, the lifecycle hooks, `onErrorCaptured`, and `effectScope`.

### Beginner

`watch` observes a specific, explicit source and runs a callback with old/new values. Use it when you need to know *what changed*, or to run side effects only in response to specific state.

```ts
import { ref, watch } from 'vue'

const count = ref(0)
watch(count, (newVal, oldVal) => {
  console.log(`Changed from ${oldVal} to ${newVal}`)
})
```

`watchEffect` runs immediately, auto-tracking whatever reactive values it reads inside — use it when the *logic* naturally determines its own dependencies and you don't need the old value.

```ts
import { ref, watchEffect } from 'vue'

const id = ref(1)
watchEffect(async () => {
  const res = await fetch(`/api/users/${id.value}`) // id is read before the await, so it IS tracked
  console.log(await res.json())
})
```

**The first rule of `watchEffect`:** only dependencies read **before the first `await`** are tracked. Reading `other.value` after an `await` registers nothing, and the effect will never re-run for it — a silent failure with no warning. When in doubt, use `watch` with explicit sources.
([Watchers — Vue.js](https://vuejs.org/guide/essentials/watchers))

### Working Knowledge

**Watch a getter, not just a ref** — needed to watch a single prop or a nested/derived value without deep-watching a whole object:

```ts
watch(
  () => props.id,
  (id) => fetchData(id),
  { immediate: true }   // also run once on setup, not only on change
)

// Multiple sources — the callback receives arrays of new and old values
watch([firstName, lastName], ([first, last], [prevFirst, prevLast]) => { /* … */ })
```

**Wrong vs. right — watching an object when you mean one property:**

```ts
// Wrong — deep-watches the entire user object; fires (and does expensive work)
// even when an unrelated field like user.lastLoginAt changes
watch(user, () => saveEmailPreference(user.value.email), { deep: true })

// Right — narrow the source to exactly what should trigger the effect
watch(() => user.value.email, (email) => saveEmailPreference(email))
```

**Wrong vs. right — the mutation-vs-replacement trap:**

```ts
// Wrong — watching a reactive object's property with a getter, but the object is
// mutated in place: `oldValue` and `newValue` are the SAME object reference,
// so any `if (oldVal.x !== newVal.x)` comparison inside is always false.
watch(() => state.profile, (nv, ov) => diff(ov, nv), { deep: true })

// Right — either watch primitives, or snapshot for comparison
watch(() => structuredClone(toRaw(state.profile)), (nv, ov) => diff(ov, nv), { deep: true })
```

**The full option set**, since these are where most watcher tuning lives:

| Option | Effect | Since |
|---|---|---|
| `immediate: true` | Also run on creation (`oldValue` is `undefined`) | 3.0 |
| `deep: true` | Traverse nested properties of an object source | 3.0 |
| `deep: 2` | Traverse to a **maximum depth** — the cheap middle ground | 3.5 |
| `once: true` | Run at most once, then stop automatically | 3.4 |
| `flush: 'post'` | Run after the DOM updates | 3.0 |
| `flush: 'sync'` | Run synchronously on every write — rarely correct | 3.0 |
| `onTrack` / `onTrigger` | Debug hooks; log what a watcher depends on and why it fired | 3.0 |

**Flush timing.** By default (`flush: 'pre'`), watcher callbacks run *before* the component re-renders in that tick — so DOM reads inside a watcher see the **old** DOM. `flush: 'post'` (or `watchPostEffect`) waits until after the DOM updates. `flush: 'sync'` skips batching entirely and fires on every single write, which turns one loop of ten mutations into ten callback runs.

```ts
watch(list, () => {
  console.log(el.value.scrollHeight)   // stale under the default 'pre'
}, { flush: 'post' })                  // correct: DOM has been patched
```

**Stopping a watcher.** Watchers created synchronously in `setup()` are bound to the component and stop on unmount. Ones created *asynchronously* are not — they leak unless you stop them by hand:

```ts
const stop = watch(source, cb)
stop()   // manual teardown

// Wrong — created after an await, so it is NOT bound to the component instance
onMounted(async () => {
  await loadConfig()
  watch(source, cb)   // survives unmount, keeps firing, holds the whole scope in memory
})
```

**Cleanup with `onWatcherCleanup`** (3.5+) — the modern replacement for manually threading an `AbortController` across re-runs:

```ts
import { ref, watchEffect, onWatcherCleanup } from 'vue'

watchEffect(async () => {
  const controller = new AbortController()
  onWatcherCleanup(() => controller.abort())   // runs before the next re-run, and on unmount

  const res = await fetch(`/api/${id.value}`, { signal: controller.signal })
  data.value = await res.json()
})
```

**Real Scenario.** A search-as-you-type input fires a `watchEffect` on every keystroke to hit an API. Without cleanup, a slow response to keystroke #1 can resolve *after* a fast response to keystroke #3, overwriting the correct result with stale data — a classic race condition that only shows up on bad networks, which is why it survives QA and lands in production. Wiring `onWatcherCleanup` to abort the in-flight request whenever the effect re-runs eliminates the race entirely, and is the officially recommended pattern as of Vue 3.5. Note `onWatcherCleanup` must be called **synchronously** during the effect — registering it after an `await` throws.
([Side Effect Cleanup — Vue.js](https://vuejs.org/guide/essentials/watchers#side-effect-cleanup))

### Advanced

**Lifecycle hooks** mirror the component's mount/update/unmount cycle. All of them must be called **synchronously during `setup()`** — registering a hook inside a `setTimeout` or after an `await` silently attaches to nothing (or worse, to whatever component happens to be instantiating at that moment).

| Hook | Fires | Typical use |
|---|---|---|
| `onBeforeMount` | Before first render | Rare |
| `onMounted` | DOM is in the document | Measure elements, attach non-Vue listeners, init third-party libs |
| `onBeforeUpdate` | Before a re-render patch | Read pre-update DOM (e.g. scroll position) |
| `onUpdated` | After a re-render patch | Read post-update DOM — do **not** mutate state here (infinite loop risk) |
| `onBeforeUnmount` | Instance still fully functional | Save state, notify a parent |
| `onUnmounted` | Teardown complete | **Mandatory cleanup point** — undo everything from `onMounted` |
| `onActivated` / `onDeactivated` | KeepAlive insertion/removal | See Part 8 |
| `onErrorCaptured` | A descendant threw | Error boundaries |
| `onRenderTracked` / `onRenderTriggered` | Dev only | Debug *why* a component re-rendered |

Two notes that matter in practice: `onMounted` **does not run on the server** during SSR (nothing is mounted), which makes it the correct home for browser-only code; and `onUnmounted` does not run for a component cached by `<KeepAlive>` — it's deactivated, not destroyed.

```ts
import { onMounted, onUnmounted } from 'vue'

const handleResize = () => { /* … */ }
onMounted(() => window.addEventListener('resize', handleResize))
onUnmounted(() => window.removeEventListener('resize', handleResize))
```

**Wrong vs. right — the #1 memory-leak pattern in Vue apps:**

```ts
// Wrong — listener attached in onMounted, never removed. Every time this
// component mounts/unmounts (inside a v-if or a router view), another listener
// stacks up, each firing on every resize forever, each pinning the whole
// component scope in memory.
onMounted(() => window.addEventListener('resize', handleResize))

// Right — always pair mount-time registration with matching unmount-time cleanup
onMounted(() => window.addEventListener('resize', handleResize))
onUnmounted(() => window.removeEventListener('resize', handleResize))
```

The same pairing applies to `setInterval`, `IntersectionObserver`/`ResizeObserver`, WebSocket subscriptions, and any third-party widget with a `.destroy()`. Extracting the pair into a composable (Part 7) is how you stop having to remember.
([Lifecycle Hooks — Vue.js](https://vuejs.org/guide/essentials/lifecycle) · [Lifecycle API](https://vuejs.org/api/composition-api-lifecycle))

**Error handling.** `onErrorCaptured` turns a component into an error boundary: it catches errors thrown by any descendant's render, lifecycle hook, watcher, or event handler, and can stop them propagating by returning `false`.

```vue
<script setup lang="ts">
import { onErrorCaptured, ref } from 'vue'

const error = ref<Error | null>(null)

onErrorCaptured((err, instance, info) => {
  error.value = err as Error
  report(err, { info })   // `info` names the Vue-specific source, e.g. "render function"
  return false            // stop propagation — omit (or return true) to let it bubble to app.config.errorHandler
})
</script>

<template>
  <FallbackUI v-if="error" :error="error" @retry="error = null" />
  <slot v-else />
</template>
```

The one thing it doesn't catch: **errors inside your own async code that never touch Vue's call stack.** An unhandled rejection from a bare `fetch()` in an event handler goes to `window.onunhandledrejection`, not here — `try`/`catch` it yourself. Pair a boundary component like this with `app.config.errorHandler` (Part 1's orientation) so nothing escapes silently.
([Error Handling — Vue.js](https://vuejs.org/error-reference/) · [`onErrorCaptured`](https://vuejs.org/api/composition-api-lifecycle#onerrorcaptured))

**`effectScope()`** groups multiple reactive effects (computeds, watchers, even lifecycle-less composables) so they can all be disposed together, without needing a component instance. This is what powers Pinia stores, which need watchers that outlive any single component.

```ts
import { effectScope, onScopeDispose, ref, watch } from 'vue'

const scope = effectScope()
scope.run(() => {
  const count = ref(0)
  watch(count, () => console.log(count.value))
  onScopeDispose(() => console.log('cleaned up'))   // also works inside composables
})

scope.stop()   // disposes every effect created inside .run(), all at once
```

`onScopeDispose()` is the composable-friendly counterpart to `onUnmounted`: it fires whether the enclosing scope is a component or a manual `effectScope`, which is what makes a composable usable outside a component.
([`effectScope()` — Vue.js API](https://vuejs.org/api/reactivity-advanced#effectscope))

[⬆ back to top](#table-of-contents)

---

## 8. Part 7 — Composables

Covers: the `use*` pattern, accepting reactive input, return-value conventions, shared vs. per-instance state.

### Working Knowledge

A composable is a plain function that uses Composition API functions (`ref`, `computed`, `watch`, lifecycle hooks) to encapsulate reusable *stateful* logic. It's Vue's answer to React hooks — but with an important difference: composables don't rely on call order, because Vue's `setup()` runs **once per instance** rather than on every render. There is no "rules of hooks" here; a composable can be called conditionally or in a loop. The only real constraint is that a composable which registers lifecycle hooks or reads `inject()` must be **called synchronously during `setup()`**, since that's when a component instance is the active context.

```ts
// composables/useMouse.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  const update = (e: MouseEvent) => { x.value = e.pageX; y.value = e.pageY }
  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return { x, y }   // plain object of refs — survives destructuring, see Part 1
}
```

```vue
<script setup lang="ts">
import { useMouse } from './composables/useMouse'
const { x, y } = useMouse()
</script>
<template><p>{{ x }}, {{ y }}</p></template>
```

**The conventions, in one list:**

- Name it `useSomething` — tooling and readers both rely on the prefix.
- Return **a plain object of refs** (destructurable), or a single ref for a one-value composable. Never a `reactive()` object.
- Accept **`MaybeRefOrGetter<T>`** for inputs that might be reactive (below), and normalize with `toValue()`.
- Register cleanup with `onScopeDispose()` (or `onUnmounted()`) *inside* the composable, so callers can't forget.
- Expose state as `readonly()` when callers should mutate only through returned actions.
- Keep it side-effect-free at module scope: state created *inside* the function is per-caller; state created *outside* it is shared by every caller in the app — see below.

### Advanced

**Accepting flexible input with `toValue()`** (3.3+) lets a caller pass a plain value, a ref, *or* a getter, all normalized identically:

```ts
import { ref, watchEffect, toValue, onWatcherCleanup, type MaybeRefOrGetter } from 'vue'

export function useFetch<T>(url: MaybeRefOrGetter<string>) {
  const data = ref<T | null>(null)
  const error = ref<Error | null>(null)
  const isLoading = ref(false)

  watchEffect(async () => {
    isLoading.value = true
    const controller = new AbortController()
    onWatcherCleanup(() => controller.abort())
    try {
      const res = await fetch(toValue(url), { signal: controller.signal })
      data.value = await res.json()
    } catch (e) {
      if ((e as Error).name !== 'AbortError') error.value = e as Error
    } finally {
      isLoading.value = false
    }
  })

  return { data, error, isLoading }
}

useFetch('/api/users')                       // static string
useFetch(urlRef)                             // ref
useFetch(() => `/api/users/${props.id}`)     // reactive getter — re-fetches when props.id changes
```

**Real Scenario.** A `useFetch(url)` composable is written to accept only a plain string, then reused in a component that needs the URL to follow a route param. Without `toValue`, the composable reads the string once and never re-fetches — and because there's no error, the bug reads as "the API is caching." The fix isn't a special case for that call site; it's accepting `MaybeRefOrGetter<string>` from the start so every caller shape works. Treat `MaybeRefOrGetter` as the default parameter type for anything a composable will read reactively.
([`toValue()` — Vue.js API](https://vuejs.org/api/reactivity-utilities#tovalue))

**Per-instance vs. shared state — where the state is created decides everything:**

```ts
// Per-instance: every caller gets its own independent count
export function useCounter() {
  const count = ref(0)
  return { count, increment: () => count.value++ }
}

// Shared (module-scoped singleton): every caller in the app reads the SAME count
const count = ref(0)
export function useSharedCounter() {
  return { count, increment: () => count.value++ }
}
```

Module-scoped shared state is a legitimate, tiny state-management pattern — and a real hazard in SSR, where the module is evaluated once per *server process*, not once per request, so one user's state leaks into another's response. For anything shared and server-rendered, use a store library with per-request instantiation, or scope the state with `effectScope()` created per app instance.

**Wrong vs. right — losing the component context:**

```ts
// Wrong — the composable is called after an await, so `onMounted` and `inject`
// have no active instance to attach to. Vue warns; the hook simply never fires.
onMounted(async () => {
  await something()
  const { x } = useMouse()
})

// Right — call composables synchronously at the top level of setup
const { x } = useMouse()
```
([Composables — Vue.js](https://vuejs.org/guide/reusability/composables))

**When *not* to write one.** If the function has no reactive state and no lifecycle — a date formatter, a currency parser — it's just a utility function. Don't give it a `use` prefix or import `ref` to look the part; plain functions are cheaper, testable without a component, and composable everywhere.

[⬆ back to top](#table-of-contents)

---

## 9. Part 8 — Built-in Components

Covers: `Transition`, `TransitionGroup`, `Teleport`, `Suspense`, `KeepAlive`.

### Working Knowledge

**`Transition`** animates a single element/component's enter/leave by toggling CSS classes at each phase. It only ever takes **one** child, triggered by `v-if`, `v-show`, or a `<component :is>` swap.

```vue
<template>
  <Transition name="fade">
    <div v-if="show">Content</div>
  </Transition>
</template>

<style>
.fade-enter-active, .fade-leave-active { transition: opacity 0.3s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>
```

| Class | When |
|---|---|
| `{name}-enter-from` / `-to` | Start / end state for entering |
| `{name}-enter-active` | Applied for the whole enter transition — put the `transition` CSS property here |
| `{name}-leave-from` / `-to` | Start / end state for leaving |
| `{name}-leave-active` | Applied for the whole leave transition |

Useful props: `appear` (animate on initial render), `mode="out-in"` (finish the leave before the enter starts — almost always what you want when swapping two elements), and `:duration` to override Vue's automatic detection. For JS-driven animation (GSAP, Web Animations API), the hook events `@before-enter`, `@enter="(el, done) => …"`, `@leave`, and `@after-leave` give you full control — pair them with `:css="false"` so Vue skips class detection entirely.

**`TransitionGroup`** does the same for a list. Differences from `Transition`: it renders a real element by default (`tag="ul"`), children **require** a unique `:key`, and it adds a `.{name}-move` class so reordering animates via FLIP.

```vue
<TransitionGroup name="list" tag="ul">
  <li v-for="item in items" :key="item.id">{{ item.name }}</li>
</TransitionGroup>

<style>
.list-move, .list-enter-active, .list-leave-active { transition: all 0.4s ease; }
.list-enter-from, .list-leave-to { opacity: 0; transform: translateY(30px); }
/* Required for smooth `move` animation — take leaving items out of layout flow */
.list-leave-active { position: absolute; }
</style>
```

**`Teleport`** renders its content into a different part of the real DOM (typically `body`) while keeping it logically inside the current component tree — so props, injections, and events still work as if it hadn't moved. This is the standard fix for modals and tooltips that would otherwise be clipped by an ancestor's `overflow: hidden` or lose a `z-index` fight inside a stacking context.

```vue
<template>
  <button @click="open = true">Open Modal</button>
  <Teleport to="body" :disabled="isMobile">
    <div v-if="open" class="modal">Rendered at &lt;body&gt;, still owned by this component</div>
  </Teleport>

  <!-- 3.5+: `defer` waits until after the current render cycle, so the target
       can be an element rendered later by Vue itself -->
  <Teleport defer to="#late-container">…</Teleport>
</template>
```

Multiple teleports to the same target append in order rather than replacing each other.
([Transition](https://vuejs.org/guide/built-ins/transition) · [TransitionGroup](https://vuejs.org/guide/built-ins/transition-group) · [Teleport](https://vuejs.org/guide/built-ins/teleport))

### Advanced

**`KeepAlive`** caches inactive component instances instead of destroying them when toggled via `v-if` or a dynamic `<component :is>`, preserving their state and DOM.

```vue
<template>
  <KeepAlive :max="10" :include="['TabA', 'TabB']">
    <component :is="currentTab" />
  </KeepAlive>
</template>
```

`include`/`exclude` match against the component's **name** — which under `<script setup>` is inferred from the filename, or set explicitly with `defineOptions({ name: 'TabA' })`. `:max` turns it into an LRU cache, which is the difference between a fast tab UI and an unbounded memory leak in a long-lived app.

**Wrong vs. right — using `onMounted` inside a `KeepAlive`d component:**

```ts
// Wrong — fires only the FIRST time the component is created, never again when
// it's toggled back into view. The panel shows stale data with no obvious cause.
onMounted(() => fetchLatestData())

// Right — onActivated fires on every reinsertion of a cached instance
onActivated(() => fetchLatestData())
onDeactivated(() => pauseTimers())
```

The same applies to teardown: `onUnmounted` does **not** run when a cached component is hidden, so timers and subscriptions keep running unless you stop them in `onDeactivated`.
([KeepAlive — Vue.js](https://vuejs.org/guide/built-ins/keep-alive))

**`Suspense`** coordinates loading state for async dependencies — components with a top-level `await` in `<script setup>`, or `defineAsyncComponent` — showing a fallback until every one of them resolves. **Still marked experimental** in the official docs; treat its API as more likely to change than anything else in this guide.

```vue
<template>
  <Suspense @pending="isLoading = true" @resolve="isLoading = false" @fallback="onFallback">
    <template #default><AsyncUserProfile :id="userId" /></template>
    <template #fallback><div>Loading…</div></template>
  </Suspense>
</template>
```

```vue
<!-- AsyncUserProfile.vue -->
<script setup lang="ts">
const props = defineProps<{ id: string }>()
const user = await fetch(`/api/users/${props.id}`).then(r => r.json()) // top-level await = async component
</script>
```

Two things to know before using it. First, a top-level `await` splits `setup()` at that point — code after it runs outside the active-instance context, so **register lifecycle hooks and call `inject()` before the first `await`**. Second, `Suspense` has no built-in error handling; pair it with an `onErrorCaptured` boundary (Part 6) or a rejected fetch will take the whole subtree down. For most data-fetching in an app, an explicit `isLoading` ref from a composable is the more predictable choice today.
([Suspense — Vue.js](https://vuejs.org/guide/built-ins/suspense) — flagged experimental in the docs themselves)

[⬆ back to top](#table-of-contents)
---

## 10. Part 9 — Directives & SFC Styling

Covers: custom directives, `v-once`, `v-memo`, and the `<style>` block's Vue-specific features.

### Working Knowledge

**Custom directives** hook into an element's lifecycle directly, below the component and reactivity layers — the right tool when you need imperative DOM access the template syntax doesn't expose, and the wrong tool for anything a component or composable could express.

```ts
import type { Directive } from 'vue'

// Local registration: the vNameOfDirective naming convention makes it available
// in this component's template as v-focus, with no registration call
const vFocus: Directive<HTMLElement> = {
  mounted: (el) => el.focus()
}
```

```vue
<template><input v-focus /></template>
```

Full hook set, plus argument/modifier/value access:

```ts
const vColor: Directive<HTMLElement, string> = {
  created(el, binding) {},        // before attributes/listeners are applied
  beforeMount(el, binding) {},    // before insertion into the DOM
  mounted(el, binding) {          // element is in the document
    // binding.arg = 'background', binding.modifiers = { bold: true }, binding.value = 'red'
    el.style[binding.arg || 'color'] = binding.value
    if (binding.modifiers.bold) el.style.fontWeight = 'bold'
  },
  beforeUpdate(el, binding) {},
  updated(el, binding) {          // binding.oldValue is available here
    if (binding.value !== binding.oldValue) el.style.color = binding.value
  },
  beforeUnmount(el, binding) {},  // ← clean up listeners/observers HERE
  unmounted(el, binding) {}
}
```

```vue
<div v-color:background.bold="'red'">…</div>
```

For the common case where the same logic belongs in `mounted` and `updated`, a bare function is shorthand for exactly those two hooks:

```ts
const vColor: Directive<HTMLElement, string> = (el, binding) => {
  el.style.color = binding.value
}
```

Global registration (`app.directive('focus', vFocus)`) is available for genuinely app-wide directives, at the cost of the same discoverability problem as any global.

**Wrong vs. right — a directive that leaks:**

```ts
// Wrong — an observer created per element, never disconnected. Every mount adds one.
const vInView: Directive<HTMLElement> = {
  mounted(el, binding) {
    new IntersectionObserver(binding.value).observe(el)
  }
}

// Right — stash it on the element and tear it down on unmount
const vInView: Directive<HTMLElement & { _obs?: IntersectionObserver }> = {
  mounted(el, binding) {
    el._obs = new IntersectionObserver(binding.value)
    el._obs.observe(el)
  },
  beforeUnmount(el) {
    el._obs?.disconnect()
    delete el._obs
  }
}
```

One caveat: a directive **used on a component** applies to that component's root element and is silently ignored (with a warning) if the component has multiple root nodes. Directives can't be targeted at an inner element the way `v-bind="$attrs"` can.
([Custom Directives — Vue.js](https://vuejs.org/guide/reusability/custom-directives))

### Advanced

**`v-once`** renders an element or subtree a single time and skips it on every subsequent update — cheap, blunt, and correct only for content that genuinely never changes.

```vue
<span v-once>{{ initialRenderTimestamp }}</span>
```

**`v-memo`** is the tunable version: it skips re-rendering a subtree unless a value in its dependency array changes. It's a targeted performance escape hatch, not a default habit.

```vue
<div v-for="item in list" :key="item.id" v-memo="[item.selected]">
  <!-- Only re-renders this subtree when item.selected changes,
       even if other fields on item (or the parent) change -->
  <ExpensiveComponent :item="item" />
</div>
```

Rules worth memorizing before reaching for it: the dependency array must have a **constant length** across renders (like React's hook deps), `v-memo` must sit on the **same element as `v-for`** when combined with it, and an incomplete dependency list means stale UI with no warning — the failure mode is a component that visibly doesn't update, and it's genuinely hard to diagnose.

**Real Scenario.** A list of 2,000 rows re-renders visibly janky whenever *any* row's `selected` state toggles, because every row's expensive child re-renders on each parent update even though only one row's visible state changed. `v-memo="[item.selected]"` scopes re-renders to exactly the rows whose memoized dependency changed. The docs are explicit that this is a niche optimization for measured hot paths: Vue's compiler-driven update granularity (Part 10) is already precise enough that reaching for `v-memo` by default adds a correctness hazard for no measurable gain. Profile first; and for lists that large, virtual scrolling usually beats memoization outright.
([`v-memo` — Vue.js API](https://vuejs.org/api/built-in-directives#v-memo) · [`v-once`](https://vuejs.org/api/built-in-directives#v-once))

### Working Knowledge — the `<style>` block

Vue's SFC compiler gives the style block four features that have no plain-CSS equivalent:

```vue
<!-- 1. scoped — adds a data-v-xxxx attribute to this component's elements and to
     every selector, so styles can't leak out -->
<style scoped>
.title { color: teal; }

/* 2. :deep() — deliberately reach into a CHILD component's markup.
   Without it, `.child-class` never matches, because the child's elements
   don't carry this component's scope attribute. */
:deep(.child-class) { padding: 0; }

/* :slotted() — style content the PARENT passed into this component's slots */
:slotted(li) { margin: 0; }

/* :global() — one escape to the document scope, without a second style block */
:global(body) { margin: 0; }
</style>

<!-- 3. CSS Modules — hashed class names exposed to the template as `$style` -->
<style module>
.red { color: red; }
</style>
<template><p :class="$style.red">…</p></template>

<!-- 4. v-bind() in CSS — a live link from component state to a CSS custom property -->
<script setup lang="ts">
const themeColor = ref('#42b883')
</script>
<style scoped>
.title { color: v-bind(themeColor); }        /* updates reactively */
.box { border-color: v-bind('theme.border'); } /* quote non-identifier expressions */
</style>
```

Two things to know about `scoped`. It applies to the component's **root element** too, which is what lets a parent style a child's root via fallthrough `class` — occasionally surprising when a parent's `.card` rule "leaks" one level down by design. And `v-bind()` in CSS compiles to an inline CSS custom property on the element, so it works with `scoped`, updates without re-rendering the component, and cannot be used for property *names* or at-rules — only values.
([SFC CSS Features — Vue.js](https://vuejs.org/api/sfc-css-features))

[⬆ back to top](#table-of-contents)

---

## 11. Part 10 — Rendering Internals, Performance & What's Next

Covers: why Vue's re-renders are cheaper than "re-render the whole component," a practical performance checklist, and Vapor Mode.

### Advanced — the performance checklist

Before any of the internals below matter, this is the order to actually work through when something is slow. Measure with the Vue DevTools timeline and `onRenderTriggered` first — every item here is wasted effort applied to the wrong bottleneck.

| Symptom | First thing to try |
|---|---|
| Slow initial load | `defineAsyncComponent` + route-level code splitting; check bundle composition |
| Slow list rendering | Virtual scrolling (render only visible rows) — before `v-memo`, before anything else |
| Expensive recomputation on every render | Move it into a `computed` (Part 1) |
| Whole app janky on updates to one big object | `shallowRef` + wholesale replacement (Part 1) |
| Third-party instance (map/chart/editor) sluggish | `markRaw` so Vue never proxies it |
| Tab/panel state lost and refetched on every toggle | `<KeepAlive :max="n">` (Part 8) |
| A specific expensive subtree re-rendering needlessly | `v-memo` / `v-once` (Part 9) — last resort |
| Slow hydration on an SSR page | Lazy hydration strategies (Part 3) |

Note what's *not* on this list: manually splitting components to "reduce render scope," which is React advice that transfers poorly. Vue's update granularity is already per-binding, as the next section explains.
([Performance — Vue.js](https://vuejs.org/guide/best-practices/performance))

### Mastery — what the compiler actually emits

Vue's template compiler doesn't produce a naive virtual-DOM diff of the whole component on every update. It statically analyzes the template at *compile time* and marks which parts are dynamic vs. static, attaching **PatchFlags** to dynamic vnodes so the runtime knows exactly what kind of change to check for (text content, one specific prop, `class`, …) without walking the entire subtree. Static subtrees are hoisted out of the render function and reused by reference across renders.

```
Compile-time template analysis (simplified)

  <div>
    <span>Static label</span>          ◀── hoisted, created once, never re-created
    <p>{{ count }}</p>                 ◀── PatchFlag: TEXT — only textContent diffed
    <MyComp :active="isActive" />      ◀── PatchFlag: PROPS, dynamicProps=['active']
  </div>
```

This is why "Vue re-renders the whole component when a ref changes" is a common but inaccurate mental model. The render function *does* re-run, but it re-creates almost nothing: hoisted nodes are reused as-is, and flagged nodes tell the patcher precisely which one attribute to compare. The practical upshot is the one from the checklist above — component-splitting for performance is rarely necessary in Vue, because the compiler has already narrowed the update before the runtime does anything.

**Try it:** in the [SFC Playground](https://play.vuejs.org), write a template with one static element and one interpolation, then read the **Compiled Output**. You'll see the static node hoisted to module scope as a `_hoisted_1` constant, and the dynamic one created with a numeric patch flag argument (`1 /* TEXT */`).

**Vapor Mode (Vue 3.6, opt-in, RC as of writing).** Everything above still produces and diffs *some* vnode tree, even a narrow one, and still ships the vdom runtime to the browser. Vapor Mode is a separate compilation strategy that skips the virtual DOM entirely for components that opt in — the compiler emits direct DOM-manipulation instructions wired straight to reactive effects, closer to how Solid or Svelte 5 compile. It is **100% opt-in per component**, coexists with standard vdom components in the same app, supports a subset of existing APIs with otherwise identical behavior, and cuts baseline bundle size for apps that go all-in. The team's current recommendation is partial adoption: a performance-sensitive page rendered in Vapor inside an otherwise conventional app.

Why this sits at Mastery tier and not earlier: it isn't stable, its API surface is still settling in RC, and **none** of the guidance above changes to use it — Vapor is a compilation *backend* swap, not a new API to learn. Treat it as "know it exists, and know it doesn't obsolete anything you just learned"; revisit once it ships stable, at which point the `shallowRef`/`v-memo` performance advice is worth re-reading against it, since both exist to work around vdom costs Vapor doesn't have.
([Rendering Mechanism — Vue.js](https://vuejs.org/guide/extras/rendering-mechanism) · [`vuejs/core` releases](https://github.com/vuejs/core/releases))

[⬆ back to top](#table-of-contents)

---

## 12. Cheat Sheets

### Reactivity — which primitive to reach for

| Need | Use |
|---|---|
| Any reactive value (the default choice) | `ref()` |
| A reactive object you'll never destructure or reassign | `reactive()` |
| A large object/array where deep tracking is wasted cost | `shallowRef()` |
| Derived, cached value from other reactive state | `computed()` |
| Expose state to a consumer that shouldn't mutate it | `readonly()` |
| A third-party class instance Vue must not proxy | `markRaw()` |
| One property of a reactive object, as a ref | `toRef(obj, 'key')` |
| Destructure a `reactive()` object safely | `toRefs(obj)` |
| Normalize a value, ref, or getter → plain value | `toValue(x)` |
| A composable's return value | Plain object of `ref()`s — never `reactive()` |
| Wait for the DOM to reflect state changes | `await nextTick()` |

### Watching

| Need | Use |
|---|---|
| React to one known source, need old/new values | `watch(source, cb)` |
| Dependencies are implicit in the logic itself | `watchEffect(fn)` |
| Watch a single prop or nested field, not a whole object | `watch(() => x.y, cb)` |
| Run once on setup as well as on change | `{ immediate: true }` |
| Run at most once, ever | `{ once: true }` (3.4+) |
| Deep-watch, but bound the traversal cost | `{ deep: 2 }` (3.5+) |
| Read updated DOM inside the callback | `{ flush: 'post' }` / `watchPostEffect` |
| Cancel in-flight async work on re-run | `onWatcherCleanup()` (3.5+) |
| Debug why a watcher fired | `{ onTrigger: console.log }` |
| Dispose a watcher created outside `setup()` | Keep the returned `stop()` and call it |

### `<script setup>` macros and helpers

| API | Purpose | Since |
|---|---|---|
| `defineProps<T>()` | Typed props | 3.0 |
| `withDefaults(defineProps<T>(), {})` | Prop defaults — needed pre-3.5 | 3.0 |
| `const { a = 1 } = defineProps<T>()` | Reactive props destructure with defaults | 3.5 |
| `defineEmits<T>()` | Typed emitted events | 3.0 |
| `defineModel<T>()` | Two-way `v-model` binding | 3.4 |
| `defineExpose({…})` | Expose internals to parent template refs | 3.0 |
| `defineOptions({…})` | Component options w/o a second `<script>` | 3.3 |
| `defineSlots<T>()` | Typed slot props | 3.3 |
| `generic="T"` attribute | Generic component type parameter | 3.3 |
| `useTemplateRef('name')` | Template ref by string name | 3.5 |
| `useId()` | SSR-stable unique ID for a11y attributes | 3.5 |
| `useAttrs()` / `useSlots()` | `$attrs` / `$slots` in script | 3.0 |

### Communication — picking the mechanism

| Direction / need | Mechanism |
|---|---|
| Parent → child data | Props |
| Child → parent event | `defineEmits` |
| Two-way value | `defineModel` (3.4+) |
| Parent → child markup | Slots |
| Child → parent data, inside parent-authored markup | Scoped slots |
| Ancestor → any descendant (ambient) | `provide` / `inject` |
| Parent calls a child method | `defineExpose` + template ref |
| Anything cross-tree or long-lived | A store (own doc) |

### Lifecycle ↔ KeepAlive equivalents

| Normal | Inside `<KeepAlive>` |
|---|---|
| `onMounted` | `onActivated` (fires on *every* reinsertion, not just the first) |
| `onUnmounted` | `onDeactivated` (`onUnmounted` does not fire while cached) |

### Directive quick reference

| Directive | Purpose |
|---|---|
| `v-bind` / `:` | Bind an attribute, prop, class, style, or spread an object |
| `v-on` / `@` | Bind an event, with `.prevent`/`.stop`/`.once`/`.self`/key modifiers |
| `v-model` | Two-way binding, with `.lazy`/`.number`/`.trim` |
| `v-if` / `v-else-if` / `v-else` | Conditionally create/destroy |
| `v-show` | Toggle `display` — keeps the element mounted |
| `v-for` | Iterate; always with a stable `:key` |
| `v-slot` / `#` | Fill a named or scoped slot |
| `v-html` / `v-text` | Raw HTML (XSS risk) / text content |
| `v-once` / `v-memo` | Render-skipping performance escape hatches |
| `v-pre` / `v-cloak` | Skip compilation / hide un-compiled template |

[⬆ back to top](#table-of-contents)

---

## 13. Suggested Learning Order

1. **Part 1 (Reactivity)** first, always — every later API is a consequence of `ref`/`reactive`/`computed`, so shortcuts here compound into confusion everywhere else.
2. **Part 2 (Template syntax)** — short, and it's the vocabulary the rest of the guide is written in. Pay particular attention to `key` and the `v-if`/`v-for` precedence rule; both cause bugs that don't look like bugs.
3. **Part 3 (Components)** — build a few components with props/emits/`defineModel` until parent↔child data flow feels automatic. Attribute fallthrough is worth understanding *before* you spend an hour on a `class` that won't apply.
4. **Part 4 (Slots)** — reach for this once you've written the same wrapper markup three times with slightly different insides.
5. **Part 6 (Watchers & Lifecycle)** — once your components have real side effects (fetches, subscriptions, observers), this is where cleanup discipline gets built. Skipping ahead of it produces leaks you'll find months later.
6. **Part 7 (Composables)** — revisit when you notice yourself copy-pasting the same `ref` + `watch` logic across two components; that's the signal to extract.
7. **Part 5 (Provide/Inject)** — deliberately *after* composables, so you learn the explicit tool before the implicit one and don't reach for injection where a composable or a prop would do.
8. **Parts 8 & 9 (Built-ins, Directives, Styling)** — as concrete needs arise (a modal → Teleport, a list toggle → Transition, a stubborn child style → `:deep()`) rather than studied in the abstract.
9. **Part 10 (Internals & Performance)** — once you're comfortable with everything above and want the "why," or when you hit a real, *measured* performance question.

[⬆ back to top](#table-of-contents)

---

## 14. Quick Self-Check

- Why does `const { count } = reactive({ count: 0 })` break reactivity, but `const { count } = defineProps<{ count: number }>()` (on Vue 3.5+) doesn't — and what does that imply about passing a destructured prop to `watch`?
- A template prints `[object Object]` for `{{ wrapper.count }}` but `0` for `{{ count }}`. What rule explains the difference?
- When would you reach for `shallowRef()` instead of `ref()`, and what specifically stops being reactive? What does `markRaw()` solve that `shallowRef()` doesn't?
- Why is `<li v-for="i in items" v-if="i.active">` wrong in Vue 3 but common in Vue 2 code, and what are the two correct rewrites?
- A `watchEffect` reads `a.value`, then `await`s, then reads `b.value`. Which of the two re-triggers it, and why?
- What's the difference between `flush: 'pre'` and `flush: 'post'`, and which one do you need to measure an element's height after a list changes?
- Why must a composable be called synchronously at the top level of `setup()` if it uses `onMounted` or `inject`, even though composables have no "rules of hooks"?
- Why must a composable return `{ x, y }` (refs) rather than `reactive({ x, y })` if callers should be able to destructure it safely?
- Slot content that references `item` from the child throws "not defined". What scoping rule causes this, and what's the fix?
- A component inside `<KeepAlive>` needs to refresh its data every time it's shown again, not just on first creation — which lifecycle hook, and why doesn't `onMounted` cover it? What's the matching teardown hook?
- What problem does `onWatcherCleanup` solve in a search-as-you-type `watchEffect`, and what does the bug look like to a user without it?
- You want a child component's inner `<input>` to receive the parent's `class` and listeners, not its root `<label>`. What two pieces of API do you need?
- Why is Vue's real-world re-render cost usually much cheaper than "diff the whole component," even without touching `v-memo` — and what does that imply about splitting components for performance?

[⬆ back to top](#table-of-contents)
