# Mastering Vue 3 — A Hands-On Reference Guide

## About This Document

- **Framing:** Curiosity-driven. No upcoming task is driving this — the goal is durable, hands-on understanding of Vue itself. Structured as a guided tutorial with runnable examples/exercises meant to actually be tried in a scaffolded project, with theory ("why it's designed this way") sprinkled in only where it explains a mechanism — e.g. *why* `reactive()` loses reactivity on destructure — not as standalone history.
- **Scope:** **Core Vue 3 + Composition API only** — reactivity, `<script setup>` and its compiler macros, watchers/lifecycle, composables, built-in components (Transition/Teleport/Suspense/KeepAlive), directives, and enough rendering-internals to explain *why* the API behaves the way it does. Explicitly **out of scope**: Vue Router, Pinia, Nuxt, and the Vite/build-tool layer — those get their own docs later, consistent with how this repo splits topics (see `vite/vite-mastery-guide.md`). The Options API is not taught — Composition API with `<script setup lang="ts">` is the only style used, per current Vue team guidance and this repo owner's stated preference.
- **Depth tiers:** **Beginner → Working Knowledge → Advanced → Mastery**, collapsed where a topic doesn't have four honest tiers (e.g. `defineOptions`/`defineSlots` stay at Working Knowledge — there's no deeper tier worth forcing).
- **Version basis:** **Vue 3.5** (stable line, e.g. [v3.5.42](https://github.com/vuejs/core/releases) on npm as of September 2026) — [Vue 3.5 announcement](https://blog.vuejs.org/posts/vue-3-5). **Vue 3.6** is in RC as of this writing, shipping **Vapor Mode** (a virtual-DOM-free compilation strategy) as an opt-in, experimental feature — covered at a "know it exists and why" level in the Mastery tier of Part 7, not taught as the default since it isn't stable yet. Anywhere 3.4-and-earlier behaved differently (e.g. `defineProps` destructure defaults needing `withDefaults`), it's flagged explicitly.
- **Assumed background:** Comfortable with modern JS/TS (ES modules, destructuring, `async`/`await`), and has used *some* frontend framework or at least vanilla DOM APIs before — this is not a "what is a browser" doc. No prior Vue 2 / Options API experience assumed or required.
- **Sourcing standard:** [vuejs.org](https://vuejs.org) official docs are the primary source, cited inline. Stack Overflow used for real-world gotchas. Checked as of September 2026.
- **Update instructions for future-me:** Match this structure (numbered Parts, tier subheadings, wrong-vs-right pairs for gotchas, TOC + back-to-top links). If Vapor Mode reaches stable, promote Part 7's Vapor coverage out of "Mastery/experimental" and re-check whether it changes any Working Knowledge guidance (e.g. whether `shallowRef` advice still applies the same way).

---

## Table of Contents

1. [Orientation — What Vue Actually Is](#1-orientation--what-vue-actually-is)
2. [Part 1 — The Reactivity System](#2-part-1--the-reactivity-system)
3. [Part 2 — Components & `<script setup>`](#3-part-2--components--script-setup)
4. [Part 3 — Watchers & Lifecycle](#4-part-3--watchers--lifecycle)
5. [Part 4 — Composables](#5-part-4--composables)
6. [Part 5 — Built-in Components](#6-part-5--built-in-components)
7. [Part 6 — Directives](#7-part-6--directives)
8. [Part 7 — Rendering Internals & What's Next](#8-part-7--rendering-internals--whats-next)
9. [Cheat Sheets](#9-cheat-sheets)
10. [Suggested Learning Order](#10-suggested-learning-order)
11. [Quick Self-Check](#11-quick-self-check)

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

Open the printed `localhost` URL and open `src/App.vue`. Everything in Parts 1–4 can be tried directly inside this file.
([Quick Start — Vue.js](https://vuejs.org/guide/quick-start))

[⬆ back to top](#table-of-contents)

---

## 2. Part 1 — The Reactivity System

Covers: `ref`, `reactive`, `computed`, `readonly`, `shallowRef`, and *why* each exists.

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

**Real Scenario.** This is one of the most common Vue 3 bugs reported on Stack Overflow: a composable returns `reactive({ x, y })`, a component destructures it for convenience (`const { x, y } = useMouse()`), and the values silently stop updating with no error thrown. This is exactly why the [composables guide](https://vuejs.org/guide/reusability/composables#return-values) recommends composables always return a plain object of `ref`s, never a `reactive()` object — refs survive destructuring because the reactive *connection* lives in the ref object itself, not in a Proxy wrapper you can strip away.
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
  <!-- Wrong usage calls the function on every re-render for any reason -->
  <div v-for="item in expensiveFilter()" :key="item.id">{{ item.name }}</div>
  <!-- Right usage reads the cached computed -->
  <div v-for="item in filtered" :key="item.id">{{ item.name }}</div>
</template>
```
([Computed Properties — Vue.js](https://vuejs.org/guide/essentials/computed))

### Advanced

**`shallowRef()`.** A `ref()` makes its `.value` *deeply* reactive — if `.value` is an object, Vue recursively wraps every nested property too. For large objects (big arrays, deeply nested API responses) that recursive wrapping has a real cost. `shallowRef()` only tracks reassignment of `.value` itself — mutating nested properties does nothing reactively.

```ts
import { shallowRef, triggerRef } from 'vue'

const data = shallowRef({ items: [] as string[] })

data.value.items.push('new') // does NOT trigger reactivity — mutated a nested property
data.value = { ...data.value, items: [...data.value.items, 'new'] } // DOES trigger — reassigned .value

// Escape hatch: force a re-render after a deliberate in-place mutation
data.value.items.push('another')
triggerRef(data)
```

**Real Scenario.** A dashboard renders a table from a 5,000-row API response stored in a `ref()`. Every incoming WebSocket update mutates one row deep inside that structure, and profiling shows the reactivity system re-wrapping the *entire* dataset's Proxy graph on every tick, even though only one row changed. Switching the top-level store to `shallowRef()` and replacing `.value` wholesale on each update (rather than mutating in place) removes the recursive-wrap cost entirely — this is the exact tradeoff the Vue team calls out in the performance guide.
([`shallowRef()` — Vue.js API](https://vuejs.org/api/reactivity-advanced#shallowref) · [Reactivity in Depth — Vue.js](https://vuejs.org/guide/extras/reactivity-in-depth))

**`readonly()`.** Wraps a reactive object so mutations are blocked (with a dev-mode warning), while reads stay reactive. Commonly used to expose store state to components that shouldn't be able to mutate it directly.

```ts
import { reactive, readonly } from 'vue'

const state = reactive({ count: 0 })
const publicState = readonly(state)

publicState.count++ // warning in dev, mutation blocked — read state.count instead, mutate via an action
```
([`readonly()` — Vue.js API](https://vuejs.org/api/reactivity-core#readonly))

### Mastery — how it actually works

Vue 3's reactivity is built on **ES `Proxy`** objects, not `Object.defineProperty` getters/setters like Vue 2. Each reactive object is wrapped in a Proxy whose `get` trap calls `track()` (registers "the currently-running effect depends on this property") and whose `set` trap calls `trigger()` (re-runs every effect that was tracked against that property). A `ref` is just a small object whose `.value` getter/setter does the same track/trigger dance for a single value — which is *why* refs need `.value` (a plain variable has no getter/setter to hook into) while `reactive()` properties don't (the Proxy intercepts property access directly).

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

As of **Vue 3.5**, the reactivity core was rewritten for a ~56% memory reduction with no API changes ([Vue 3.5 announcement](https://blog.vuejs.org/posts/vue-3-5)), and **Vue 3.6** replaces the dependency-tracking engine with [`alien-signals`](https://github.com/vuejs/core/releases) — again with no public API change, since this machinery sits entirely behind `ref`/`reactive`/`computed`. This is a useful pattern to internalize: Vue has repeatedly rewritten reactivity internals across versions while keeping the surface API stable, which is exactly why understanding the *mental model* (track/trigger, not the specific data structure) is more durable than memorizing implementation details.

[⬆ back to top](#table-of-contents)

---

## 3. Part 2 — Components & `<script setup>`

Covers: SFC anatomy, `defineProps`, `defineEmits`, `defineModel`, `defineExpose`, generics.

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
([`<script setup>` — Vue.js](https://vuejs.org/api/sfc-script-setup))

### Working Knowledge

**`defineProps`.** Type-based declaration is preferred over the runtime-object form — it gives full IDE inference and requires no separate `PropType` import.

```ts
// Type-based (recommended)
const props = defineProps<{
  title: string
  count?: number
}>()

// Vue 3.5+ — destructure directly with inline defaults
const { title, count = 0 } = defineProps<{
  title: string
  count?: number
}>()
```

**Wrong vs. right — defaults before Vue 3.5:**

```ts
// Wrong (pre-3.5 mental model applied incorrectly) — plain destructure loses reactivity
// AND silently drops the "count" default the type suggests
const { title, count = 0 } = defineProps<{ title: string; count?: number }>() // only correct on 3.4-, this actually works differently

// Right for Vue 3.4 and earlier — withDefaults wrapper
const props = withDefaults(defineProps<{
  title: string
  items?: string[]
}>(), {
  items: () => [] // factory function required for array/object defaults
})
```

The reason this is worth calling out explicitly: reactive-props-destructure (destructuring `defineProps()` directly) was *disallowed* pre-3.5 because it broke reactivity the same way plain `reactive()` destructuring does — Vue 3.5 added compiler magic that makes `const { title } = defineProps()` specifically transform into reactive access under the hood. It only works for `defineProps` itself, not for `reactive()` in general — don't generalize the pattern.
([`defineProps` — Vue.js](https://vuejs.org/api/sfc-script-setup#defineprops--defineemits) · [Vue 3.5 Reactive Props Destructure](https://blog.vuejs.org/posts/vue-3-5#reactive-props-destructure))

**`defineEmits`.** Named-tuple syntax gives typed payloads:

```ts
const emit = defineEmits<{
  update: [value: string]
  change: [id: number, name: string]
}>()

emit('update', 'new value')
```

**`defineModel`** (Vue 3.4+). Replaces the old `props: ['modelValue'] + emit('update:modelValue')` boilerplate with a single reactive ref that reads *and* writes back to the parent.

```ts
// Child.vue
const model = defineModel<string>() // creates "modelValue" prop + "update:modelValue" emit
model.value = 'hello' // writes propagate to the parent's v-model binding

// Named model
const count = defineModel<number>('count', { default: 0 })
```

```vue
<!-- Parent.vue -->
<Child v-model="name" />
<Child v-model:count="total" />
```
([`defineModel` — Vue.js](https://vuejs.org/api/sfc-script-setup#definemodel))

**`defineExpose`.** Components using `<script setup>` are **closed by default** — a parent holding a template ref to a child can't reach into its internals unless the child explicitly exposes them.

```ts
// Child.vue
const count = ref(0)
const reset = () => { count.value = 0 }
defineExpose({ count, reset })
```

```vue
<!-- Parent.vue -->
<script setup lang="ts">
const childRef = ref<{ count: number; reset: () => void }>()
</script>
<template>
  <Child ref="childRef" />
  <button @click="childRef?.reset()">Reset child</button>
</template>
```
([`defineExpose` — Vue.js](https://vuejs.org/api/sfc-script-setup#defineexpose))

### Advanced

**Generic components.** A component can declare a type parameter the same way a TS generic function would, useful for things like a typed `<Select>` or `<List>` that should preserve the caller's item type.

```vue
<script setup lang="ts" generic="T extends { id: string | number }">
defineProps<{
  items: T[]
  selected: T
}>()
</script>
```
([Generics — Vue.js](https://vuejs.org/api/sfc-script-setup#generics))

**Real Scenario.** A `<DataTable>` component is written once as `items: any[]`, then reused across a dozen pages with different row shapes — losing all autocomplete on `row.` inside the row-template slot. Converting it to `generic="T"` and typing `items: T[]` plus a scoped slot as `#row="{ row }: { row: T }"` restores full type inference at every call site, without duplicating the component. This is the single highest-leverage use of generics in real Vue codebases: reusable list/table/select components.

### Working Knowledge (continued) — lower-ceremony macros

`defineOptions` (3.3+) sets component options (like `name` or `inheritAttrs`) without a second `<script>` block; `defineSlots` (3.3+) types slot props for IDE checking. Both are one-liners with no deeper tier worth documenting:

```ts
defineOptions({ inheritAttrs: false, name: 'UserCard' })

const slots = defineSlots<{
  default(props: { item: string }): any
}>()
```
([`defineOptions`](https://vuejs.org/api/sfc-script-setup#defineoptions) · [`defineSlots`](https://vuejs.org/api/sfc-script-setup#defineslots))

[⬆ back to top](#table-of-contents)

---

## 4. Part 3 — Watchers & Lifecycle

Covers: `watch`, `watchEffect`, flush timing, lifecycle hooks, `effectScope`.

### Beginner

`watch` observes a specific, explicit source and runs a callback with old/new values. Use it when you need to know *what changed* or need to run side effects only in response to specific state.

```ts
import { ref, watch } from 'vue'

const count = ref(0)
watch(count, (newVal, oldVal) => {
  console.log(`Changed from ${oldVal} to ${newVal}`)
})
```

`watchEffect` runs immediately, auto-tracking whatever reactive values it reads inside — use it when the *logic* naturally determines its own dependencies, and you don't need the old value.

```ts
import { ref, watchEffect } from 'vue'

const id = ref(1)
watchEffect(async () => {
  const res = await fetch(`/api/users/${id.value}`) // id is auto-tracked as a dependency
  console.log(await res.json())
})
```
([Watchers — Vue.js](https://vuejs.org/guide/essentials/watchers))

### Working Knowledge

**Watch a getter, not just a ref** — needed to watch a single prop or a nested/derived value without deep-watching the whole object:

```ts
watch(
  () => props.id,
  (id) => fetchData(id),
  { immediate: true } // also run once on setup, not just on change
)

// Multiple sources
watch([firstName, lastName], ([first, last]) => {
  fullName.value = `${first} ${last}`
})
```

**Wrong vs. right — watching an object when you mean one property:**

```ts
// Wrong — deep-watches the entire user object; fires (and does expensive work)
// even when an unrelated field like user.lastLoginAt changes
watch(user, () => saveEmailPreference(user.value.email), { deep: true })

// Right — narrow the source to exactly what should trigger the effect
watch(() => user.value.email, (email) => saveEmailPreference(email))
```

**Cleanup with `onWatcherCleanup`** (3.5+) — replaces the older pattern of manually tracking an `AbortController` across re-runs:

```ts
import { ref, watchEffect, onWatcherCleanup } from 'vue'

watchEffect(async () => {
  const controller = new AbortController()
  onWatcherCleanup(() => controller.abort()) // runs before the next re-run, and on unmount

  const res = await fetch(`/api/${id.value}`, { signal: controller.signal })
  data.value = await res.json()
})
```

**Real Scenario.** A search-as-you-type input fires a `watchEffect` on every keystroke to hit an API. Without cleanup, a slow response to keystroke #1 can resolve *after* a fast response to keystroke #3, overwriting the correct result with stale data — a classic race condition. Wiring `onWatcherCleanup` to abort the in-flight request whenever the effect re-runs eliminates the race entirely, and is the officially recommended pattern as of Vue 3.5.
([Watchers: Side Effect Cleanup — Vue.js](https://vuejs.org/guide/essentials/watchers#side-effect-cleanup) · [Stack Overflow: Vue 3 watchEffect race condition on fast input](https://stackoverflow.com/questions/tagged/vuejs3+watch))

**Flush timing.** By default (`flush: 'pre'`), watcher callbacks run *before* the component re-renders for that same tick — so DOM reads inside a watcher see stale DOM. `flush: 'post'` (or `watchPostEffect`) waits until after the DOM updates.

```ts
watch(source, callback, { flush: 'post' }) // needed when the callback reads the updated DOM
```

### Advanced

**Lifecycle hooks** mirror the component's mount/update/unmount cycle. The two used almost universally are `onMounted` (DOM is ready — safe to measure elements, attach non-Vue listeners, start timers) and `onUnmounted` (mandatory cleanup point — remove anything registered in `onMounted`).

```ts
import { onMounted, onUnmounted } from 'vue'

const handleResize = () => { /* ... */ }
onMounted(() => window.addEventListener('resize', handleResize))
onUnmounted(() => window.removeEventListener('resize', handleResize))
```

**Wrong vs. right — the #1 memory-leak pattern in Vue apps:**

```ts
// Wrong — listener attached in onMounted, never removed. Every time this
// component mounts/unmounts (e.g. inside a v-if or router view), another
// listener stacks up, each firing on every resize forever.
onMounted(() => window.addEventListener('resize', handleResize))

// Right — always pair mount-time registration with matching unmount-time cleanup
onMounted(() => window.addEventListener('resize', handleResize))
onUnmounted(() => window.removeEventListener('resize', handleResize))
```
([Lifecycle Hooks — Vue.js](https://vuejs.org/guide/essentials/lifecycle))

**`effectScope()`** — groups multiple reactive effects (refs, computeds, watchers) so they can all be disposed together, without needing a component instance. This is what powers things like Pinia stores, which need watchers that outlive any single component.

```ts
import { effectScope, onScopeDispose } from 'vue'

const scope = effectScope()
scope.run(() => {
  const count = ref(0)
  watch(count, () => console.log(count.value))
  onScopeDispose(() => console.log('cleaned up'))
})

scope.stop() // disposes every effect created inside .run(), all at once
```
([`effectScope()` — Vue.js API](https://vuejs.org/api/reactivity-advanced#effectscope))

[⬆ back to top](#table-of-contents)

---

## 5. Part 4 — Composables

Covers: the `use*` pattern, accepting reactive input, returning refs.

### Working Knowledge

A composable is a plain function that uses Composition API functions (`ref`, `watch`, lifecycle hooks) to encapsulate reusable stateful logic — Vue's answer to React hooks, but without the rules-of-hooks restrictions (composables can be called conditionally, in loops, etc., since they don't rely on call-order).

```ts
// composables/useMouse.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  const update = (e: MouseEvent) => { x.value = e.pageX; y.value = e.pageY }
  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return { x, y } // plain object of refs — survives destructuring, see Part 1
}
```

```vue
<script setup lang="ts">
import { useMouse } from './composables/useMouse'
const { x, y } = useMouse()
</script>
<template><p>{{ x }}, {{ y }}</p></template>
```
([Composables — Vue.js](https://vuejs.org/guide/reusability/composables))

### Advanced

**Accepting flexible input with `toValue()`** (3.3+) — lets a composable's caller pass a plain value, a ref, *or* a getter function, and the composable normalizes all three the same way.

```ts
import { ref, watchEffect, toValue, type MaybeRefOrGetter } from 'vue'

export function useFetch(url: MaybeRefOrGetter<string>) {
  const data = ref(null)
  watchEffect(async () => {
    const res = await fetch(toValue(url)) // works whether url is a string, a ref, or () => string
    data.value = await res.json()
  })
  return { data }
}

useFetch('/api/users')                       // static string
useFetch(urlRef)                             // ref
useFetch(() => `/api/users/${props.id}`)     // reactive getter — re-fetches when props.id changes
```

**Real Scenario.** A `useFetch(url)` composable is written to accept only a plain string, then reused in a component that needs the URL to update reactively based on a route param. Without `toValue`, the composable captures the string once and never re-fetches when the param changes — the fix isn't a special case, it's accepting `MaybeRefOrGetter<string>` from the start so every caller shape "just works."
([`toValue()` — Vue.js API](https://vuejs.org/api/reactivity-utilities#tovalue))

[⬆ back to top](#table-of-contents)

---

## 6. Part 5 — Built-in Components

Covers: `Transition`, `TransitionGroup`, `Teleport`, `Suspense`, `KeepAlive`.

### Working Knowledge

**`Transition`** animates a single element/component's enter/leave by toggling CSS classes at each phase.

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

**`TransitionGroup`** does the same for a `v-for` list; each child needs a unique `:key`, and it adds a `.{name}-move` class for reordering animations.

**`Teleport`** renders its content into a different part of the real DOM (typically `body`) while keeping it logically inside the current component tree — the standard fix for modals/tooltips that would otherwise be clipped by an ancestor's `overflow: hidden` or fight with `z-index` stacking contexts.

```vue
<template>
  <button @click="open = true">Open Modal</button>
  <Teleport to="body">
    <div v-if="open" class="modal">Modal content, rendered at &lt;body&gt;</div>
  </Teleport>
</template>
```
([Transition](https://vuejs.org/guide/built-ins/transition) · [TransitionGroup](https://vuejs.org/guide/built-ins/transition-group) · [Teleport](https://vuejs.org/guide/built-ins/teleport))

### Advanced

**`Suspense`** coordinates loading state for async dependencies — components using top-level `await` in `<script setup>`, or `defineAsyncComponent` — showing a fallback until they resolve. **Still marked experimental** in the official docs; treat its API as more likely to change than the rest of this guide.

```vue
<template>
  <Suspense>
    <template #default><AsyncUserProfile :id="userId" /></template>
    <template #fallback><div>Loading…</div></template>
  </Suspense>
</template>
```

```vue
<!-- AsyncUserProfile.vue -->
<script setup lang="ts">
const props = defineProps<{ id: string }>()
const user = await fetch(`/api/users/${props.id}`).then(r => r.json()) // makes this an async component
</script>
```
([Suspense — Vue.js](https://vuejs.org/guide/built-ins/suspense) — flagged experimental in the docs themselves)

**`KeepAlive`** caches inactive component instances (instead of destroying them) when toggled via `v-if`/dynamic `<component :is>`, preserving their state and DOM. `onActivated`/`onDeactivated` replace `onMounted`/`onUnmounted` as the relevant lifecycle hooks for cached components.

```vue
<template>
  <KeepAlive :max="10">
    <component :is="currentTab" />
  </KeepAlive>
</template>
```

**Wrong vs. right — using `onMounted` inside a `KeepAlive`d component:**

```ts
// Wrong — this only fires the FIRST time the component is created, not
// every time it's toggled back into view via KeepAlive
onMounted(() => fetchLatestData())

// Right — onActivated fires every time a cached instance is reinserted
onActivated(() => fetchLatestData())
onDeactivated(() => pauseTimers())
```
([KeepAlive — Vue.js](https://vuejs.org/guide/built-ins/keep-alive))

[⬆ back to top](#table-of-contents)

---

## 7. Part 6 — Directives

Covers: `v-memo`, `v-once`, custom directives.

### Working Knowledge

**Custom directives** hook into an element's lifecycle directly (below the component/reactivity layer) — the right tool when you need imperative DOM access Vue's template syntax doesn't expose, like autofocus-on-mount.

```ts
// Local, via the vNameOfDirective naming convention
const vFocus: Directive<HTMLElement> = {
  mounted: (el) => el.focus()
}
```

```vue
<template><input v-focus /></template>
```

Full hook set, plus argument/modifier access:

```ts
const vColor: Directive<HTMLElement, string> = {
  mounted(el, binding) {
    // binding.arg = 'background', binding.modifiers = { bold: true }, binding.value = 'red'
    el.style[binding.arg || 'color'] = binding.value
    if (binding.modifiers.bold) el.style.fontWeight = 'bold'
  }
}
```

```vue
<div v-color:background.bold="'red'">
```
([Custom Directives — Vue.js](https://vuejs.org/guide/reusability/custom-directives))

### Advanced

**`v-memo`** skips re-rendering a subtree unless the given dependency array changes — a targeted performance escape hatch, not a default habit.

```vue
<div v-for="item in list" :key="item.id" v-memo="[item.selected]">
  <!-- Only re-renders this subtree when item.selected changes,
       even if other fields on item (or the parent) change -->
  <ExpensiveComponent :item="item" />
</div>
```

**Real Scenario.** A list of 2,000 rows re-renders visibly janky whenever *any* row's `selected` state toggles, because every row's expensive child re-renders on every parent update even though only one row's visible state actually changed. `v-memo="[item.selected]"` on each row scopes re-renders to exactly the rows whose memoized dependency changed. The Vue docs are explicit that this is a niche optimization for measured hot paths — reaching for it by default adds complexity without benefit in the common case, since Vue's compiler-driven re-render granularity is already fairly precise (see Part 7).
([`v-memo` — Vue.js API](https://vuejs.org/api/built-in-directives#v-memo))

[⬆ back to top](#table-of-contents)

---

## 8. Part 7 — Rendering Internals & What's Next

Covers: why Vue's re-renders are cheaper than "re-render the whole component," and Vapor Mode.

### Mastery

Vue's template compiler doesn't produce a naive virtual-DOM diff of the whole component on every update. It statically analyzes the template at *compile time* and marks which parts are dynamic vs. static, attaching **PatchFlags** to dynamic vnodes so the runtime knows exactly what kind of change to check for (text content, a specific prop, `class`, etc.) without diffing the entire subtree. Static subtrees are hoisted out and reused across renders entirely. This is why "Vue re-renders the whole component when a ref changes" is a common but inaccurate mental model — in practice, the compiler has already narrowed the update to specific, tagged vnodes before the runtime does anything.

```
Compile-time template analysis (simplified)

  <div>
    <span>Static label</span>          ◀── hoisted, never re-created
    <p>{{ count }}</p>                 ◀── PatchFlag: TEXT — only textContent diffed
    <MyComp :active="isActive" />      ◀── PatchFlag: PROPS, tracked=['active']
  </div>
```

**Vapor Mode (Vue 3.6, opt-in, RC as of writing).** Everything above still produces and diffs *some* vnode tree, even if a narrow one. Vapor Mode is a separate compilation strategy that skips the virtual DOM entirely for components that opt in — the compiler emits direct DOM-manipulation instructions tied straight to reactive updates, closer to how Svelte or Solid compile. It's **100% opt-in per-component**, coexists with standard vdom components in the same app, and reported third-party benchmarks put it in the same performance class as Solid/Svelte 5.
([Vapor Mode — Vue.js Vapor docs](https://vuejs.org/guide/extras/vapor) · [`vuejs/core` releases](https://github.com/vuejs/core/releases))

Why this belongs at Mastery tier and not earlier in this doc: it's not yet stable, its API surface is still settling in RC, and none of the guidance above (props, watchers, composables) changes to use it — Vapor Mode is a compilation *backend* swap, not a new API to learn. Treat it as "know it exists, and know it doesn't obsolete anything you just learned" for now; revisit this section once it ships stable.

[⬆ back to top](#table-of-contents)

---

## 9. Cheat Sheets

### Reactivity — which primitive to reach for

| Need | Use |
|---|---|
| A single reactive value (primitive or object) | `ref()` |
| A reactive object where you'll never destructure it out of scope | `reactive()` |
| A large object/array where deep tracking is wasted cost | `shallowRef()` |
| Derived, cached value from other reactive state | `computed()` |
| Expose state to a consumer that shouldn't mutate it | `readonly()` |
| A composable's return value | Plain object of `ref()`s — never `reactive()` |

### Watching

| Need | Use |
|---|---|
| React to one specific known source, need old/new values | `watch(source, cb)` |
| Effect's dependencies are implicit in the logic itself | `watchEffect(fn)` |
| Watch a single prop/nested field, not a whole object | `watch(() => x.y, cb)` |
| Read updated DOM inside the callback | `{ flush: 'post' }` / `watchPostEffect` |
| Cancel in-flight async work on re-run | `onWatcherCleanup()` (3.5+) |

### `<script setup>` macros

| Macro | Purpose | Since |
|---|---|---|
| `defineProps<T>()` | Typed props | 3.0 |
| `defineEmits<T>()` | Typed emitted events | 3.0 |
| `defineModel<T>()` | Two-way `v-model` binding | 3.4 |
| `defineExpose({...})` | Expose internals to parent template refs | 3.0 |
| `defineOptions({...})` | Component options w/o extra `<script>` block | 3.3 |
| `defineSlots<T>()` | Typed slot props | 3.3 |
| `withDefaults(defineProps<T>(), {...})` | Prop defaults, needed pre-3.5 | 3.0 |

### Lifecycle ↔ KeepAlive equivalents

| Normal | Inside `<KeepAlive>` |
|---|---|
| `onMounted` | `onActivated` (fires on *every* reinsertion, not just the first) |
| `onUnmounted` | `onDeactivated` |

[⬆ back to top](#table-of-contents)

---

## 10. Suggested Learning Order

1. **Part 1 (Reactivity)** first, always — every later API is a consequence of `ref`/`reactive`/`computed`, so shortcuts here compound into confusion everywhere else.
2. **Part 2 (Components)** — build a few components with props/emits/`defineModel` until the parent↔child data flow feels automatic.
3. **Part 3 (Watchers & Lifecycle)** — once you're writing components with real side effects (fetches, subscriptions), this is where cleanup discipline gets built.
4. **Part 4 (Composables)** — revisit once you notice yourself copy-pasting the same `ref` + `watchEffect` logic across two components; that's the signal to extract it.
5. **Part 5 & 6 (Built-ins, Directives)** — reach for these as concrete needs arise (a modal → Teleport, a list toggle → Transition) rather than studying them in the abstract.
6. **Part 7 (Internals)** — read once you're comfortable with everything above and want the "why," or when you hit a real performance question `v-memo`/`shallowRef` might solve.

[⬆ back to top](#table-of-contents)

---

## 11. Quick Self-Check

- Why does `const { count } = reactive({ count: 0 })` break reactivity, but `const { count } = defineProps<{ count: number }>()` (on Vue 3.5+) doesn't?
- When would you reach for `shallowRef()` instead of `ref()`, and what specifically stops being reactive?
- What's the difference in *when* a `watch` callback runs vs. a `watchEffect`, in terms of how each determines its dependencies?
- Why must a composable return `{ x, y }` (refs) rather than `reactive({ x, y })` if you want callers to be able to destructure it safely?
- A component inside `<KeepAlive>` needs to refresh its data every time it's shown again, not just on first creation — which lifecycle hook is that, and why doesn't `onMounted` cover it?
- What problem does `onWatcherCleanup` solve in a search-as-you-type `watchEffect`, and what would happen without it?
- Why is Vue's real-world re-render cost usually much cheaper than "diff the whole component," even without touching `v-memo`?

[⬆ back to top](#table-of-contents)
