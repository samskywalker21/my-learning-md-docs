# Vue 3 — Reactivity (Parts 1–2)

> **Spec:** this doc follows the shared spec in [`vue-mastery-guide.md` → About This Document](./vue-mastery-guide.md#about-this-document) — curiosity-driven, `<script setup lang="ts">` only, Vue **3.5.42**, verified **September 6, 2026**. Read that first if you haven't.

This is the doc to read properly. Vue's reactivity system is not one feature among many — it is the thing every other API in Vue is built out of. A component's render is a reactive effect. `computed` is a lazy reactive effect. `watch` is a reactive effect with a callback. If you understand tracking, the rest of Vue stops having surprises in it.

---

## Table of Contents

**Part 1 — Refs, Reactive & Computed**
1. [Beginner: `ref()` and the one rule](#1-beginner-ref-and-the-one-rule)
2. [Working Knowledge: `reactive()`, and why `ref` is still the default](#2-working-knowledge-reactive-and-why-ref-is-still-the-default)
3. [Working Knowledge: `computed()`](#3-working-knowledge-computed)
4. [Advanced: unwrapping rules, `toRef`, `toRefs`, `toValue`](#4-advanced-unwrapping-rules-toref-torefs-tovalue)
5. [Mastery: what `track` and `trigger` actually do](#5-mastery-what-track-and-trigger-actually-do)

**Part 2 — The Shape of Reactivity**

6. [Beginner: deep reactivity, and what it costs](#6-beginner-deep-reactivity-and-what-it-costs)
7. [Working Knowledge: `shallowRef` and `shallowReactive`](#7-working-knowledge-shallowref-and-shallowreactive)
8. [Advanced: `triggerRef`, `customRef`, `toRaw`, `markRaw`](#8-advanced-triggerref-customref-toraw-markraw)
9. [Advanced: arrays, `Map`, `Set`, and class instances](#9-advanced-arrays-map-set-and-class-instances)
10. [Mastery: `effectScope` and owning your own effects](#10-mastery-effectscope-and-owning-your-own-effects)
11. [Mastery: debugging reactivity](#11-mastery-debugging-reactivity)
12. [Cheat Sheet](#cheat-sheet)

---

# Part 1 — Refs, Reactive & Computed

## 1. Beginner: `ref()` and the one rule

A **ref** is a box holding a value. Vue can tell when you open the box (read) and when you put something new in it (write) — that is the entire trick. You read and write through `.value` in JavaScript; in templates, Vue unwraps it for you.

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)

function increment() {
  count.value++          // .value in JS — always
}
</script>

<template>
  <button @click="increment">{{ count }}</button>
  <!-- no .value here — templates auto-unwrap top-level refs -->
</template>
```

`ref()` works with **any** value type — numbers, strings, booleans, objects, arrays, `Map`s ([Reactivity Fundamentals](https://vuejs.org/guide/essentials/reactivity-fundamentals.html)). That single fact is why the Vue docs say plainly: *"we recommend using `ref()` as the primary API for declaring reactive state."*

**The one rule to internalise now:** `.value` in script, nothing in template. Forgetting `.value` is the single most common Vue beginner bug, and it usually fails *silently* — you get the ref object itself, which is truthy, so no error is thrown.

```ts
// ✗ wrong — comparing a Ref object to a number is always false
const count = ref(0)
if (count === 0) { /* never runs */ }

// ✓ right
if (count.value === 0) { /* runs */ }
```

TypeScript catches this one for you (`Ref<number>` is not `number`), which is a good reason to use `lang="ts"` even for a scratchpad.

**Try It.** Put this in `App.vue` and open the console:

```vue
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
console.log(count)              // RefImpl { … }  ← the box
console.log(count.value)        // 0              ← the contents
</script>

<template>
  <p>{{ count }}</p>            <!-- 0     — unwrapped -->
  <p>{{ count.value }}</p>      <!-- 0     — also works, but redundant -->
  <p>{{ typeof count }}</p>     <!-- number — because it was unwrapped first -->
</template>
```

The third line is the one worth pausing on: unwrapping happens *before* your expression evaluates, so inside the template `count` genuinely is a number.

[↑ Back to top](#table-of-contents)

---

## 2. Working Knowledge: `reactive()`, and why `ref` is still the default

`reactive()` takes an object and returns a **Proxy** of it. You mutate properties directly — no `.value`:

```ts
import { reactive } from 'vue'

const state = reactive({ count: 0, user: { name: 'Ada' } })
state.count++            // reactive
state.user.name = 'Grace' // also reactive — reactive() is deep
```

That looks nicer than `.value`. It is still not the default, because of three limitations the docs call out explicitly ([Reactivity Fundamentals](https://vuejs.org/guide/essentials/reactivity-fundamentals.html)):

**1. Object types only.** `reactive(0)` and `reactive('hi')` do nothing useful. Refs hold anything.

**2. You cannot replace the object.** Reactivity is attached to *that proxy*, so reassigning the variable severs every subscriber:

```ts
// ✗ wrong — the template still watches the OLD proxy; the UI freezes
let state = reactive({ items: [] })
async function load() {
  state = reactive({ items: await fetchItems() })   // new proxy, old subscribers
}

// ✓ right — mutate in place…
const state = reactive({ items: [] as Item[] })
async function load() {
  state.items = await fetchItems()
}

// ✓ …or just use a ref, where replacement is the normal operation
const items = ref<Item[]>([])
async function load() {
  items.value = await fetchItems()
}
```

**3. Destructuring breaks it.** This is the big one:

```ts
const state = reactive({ count: 0 })

// ✗ wrong — `count` is a plain number copied out of the proxy
let { count } = state
count++                  // state.count is still 0; nothing re-renders

// ✗ wrong for the same reason — the proxy's getter is never called again
function useCount(n: number) { /* … */ }
useCount(state.count)
```

Why: tracking happens **when a property is read through the proxy**. Destructuring reads once, at destructure time, and hands you a disconnected copy. There is no mechanism by which a bare `number` could notify anyone. This is not a Vue quirk you can work around by trying harder — it is a property of JavaScript values.

The fixes are `toRefs()` (§4) or, far more often, *don't use `reactive` here*.

**So when is `reactive` right?** When you have a cohesive group of fields that you always mutate in place and never replace — a form model, a canvas viewport, a wizard's step state:

```ts
const form = reactive({ email: '', password: '', remember: false })
// form.email = … everywhere; you never do form = {…}
```

Even then, a `ref` holding an object is a defensible choice, and it's the one this set uses by default. Picking `ref` everywhere costs you `.value`; picking `reactive` costs you three rules to remember at every call site.

> **vs. React.** There is no "you must not mutate state" rule here. Mutating a reactive object *is* the update mechanism — the proxy's `set` trap is what notifies subscribers. Immutability is an option for performance (see [`shallowRef`](#7-working-knowledge-shallowref-and-shallowreactive)), not a correctness requirement.

**Try It.** Prove limitation 3 to yourself:

```vue
<script setup lang="ts">
import { reactive } from 'vue'
const state = reactive({ count: 0 })
let { count } = state
</script>

<template>
  <button @click="count++">detached: {{ count }}</button>
  <button @click="state.count++">attached: {{ state.count }}</button>
</template>
```

Click the first button: **nothing happens** — not even the label. (`count` is a module-scope `let`; the render effect never tracked it, so nothing re-renders.) Click the second: **both labels update**, because the re-render re-reads the stale `count` binding too. That second effect is what makes this bug so confusing in the wild — the broken value *appears* to work whenever something else triggers a render.

[↑ Back to top](#table-of-contents)

---

## 3. Working Knowledge: `computed()`

A computed is a derived value that caches. It re-evaluates only when one of the reactive values it read has changed, and it is **lazy** — it doesn't run at all until something reads it.

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const items = ref([
  { id: 1, title: 'Write docs', done: true },
  { id: 2, title: 'Read docs', done: false },
])

const remaining = computed(() => items.value.filter(i => !i.done).length)
</script>

<template>
  <p>{{ remaining }} left</p>
</template>
```

**Computed vs. a method.** Both produce the right number. `{{ countRemaining() }}` re-runs the filter on *every* render of this component; `{{ remaining }}` re-runs it only when `items` actually changes. For a cheap expression the difference is noise; for a filter over 10,000 rows inside a component that re-renders on mouse-move, it is the whole performance story.

**Computeds must be pure.** A getter is for deriving a value, not for doing things:

```ts
// ✗ wrong — side effect in a getter. Fires unpredictably (lazily, cached),
//   and mutating a dependency inside a computed can loop.
const total = computed(() => {
  logAnalytics('total viewed')      // side effect
  cart.lastViewed = Date.now()      // mutating a dependency
  return cart.items.reduce((s, i) => s + i.price, 0)
})

// ✓ right — derive here, act in a watcher (Part 9)
const total = computed(() => cart.items.reduce((s, i) => s + i.price, 0))
watch(total, t => logAnalytics('total changed', t))
```

**Writable computeds** exist for the genuine two-way case:

```ts
const firstName = ref('Ada')
const lastName = ref('Lovelace')

const fullName = computed({
  get: () => `${firstName.value} ${lastName.value}`,
  set(value: string) {
    [firstName.value, lastName.value] = value.split(' ')
  },
})

fullName.value = 'Grace Hopper'   // updates both refs
```

Reach for this sparingly — it's ideal for adapting a value to `v-model` (Part 4) and a smell almost everywhere else.

**Don't mutate inside a computed, and don't create async computeds.** A computed getter runs synchronously; `async` returns a Promise, so your template gets a `Promise`, not a value. Fetch in a watcher or a composable instead (Parts 9–10).

**Try It.** Watch the cache work:

```ts
const n = ref(1)
const double = computed(() => { console.log('computing'); return n.value * 2 })

console.log(double.value)  // logs "computing", then 2
console.log(double.value)  // logs 2 only — cached
n.value = 5
console.log(double.value)  // logs "computing", then 10
```

Note that setting `n.value = 5` alone logs nothing — laziness means the recompute waits for a reader.

[↑ Back to top](#table-of-contents)

---

## 4. Advanced: unwrapping rules, `toRef`, `toRefs`, `toValue`

Auto-unwrapping is a convenience, and like most conveniences it has edges. The official rules ([Reactivity Fundamentals](https://vuejs.org/guide/essentials/reactivity-fundamentals.html)):

| Where | Unwrapped? |
|---|---|
| **Top-level** property in a template | ✅ yes — `{{ count }}` |
| Property of a **`reactive` object** | ✅ yes — `state.count` where `count` is a ref |
| Element of a **plain array or `Map`** inside `reactive` | ❌ **no** — `books[0].value` |
| **Non-top-level** path in a template expression | ⚠️ only as a final render convenience |

That last row is the one that bites:

```vue
<script setup lang="ts">
import { ref } from 'vue'
const object = { id: ref(1) }   // a PLAIN object holding a ref — not reactive()
</script>

<template>
  {{ object.id }}       <!-- 1 — convenience unwrap at render time -->
  {{ object.id + 1 }}   <!-- "[object Object]1" — no unwrap inside an expression -->
</template>
```

The fix is not to memorise the rule but to avoid the shape: either make `object` a `reactive()` (then `.id` unwraps properly), or destructure the ref out into a top-level binding.

### `toRef` and `toRefs`

`toRefs(obj)` converts every property of a reactive object into a ref that stays linked to the source. It is the sanctioned answer to "I want to destructure a reactive object":

```ts
const state = reactive({ count: 0, name: 'Ada' })

// ✗ wrong — disconnected copies
const { count } = state

// ✓ right — refs that write through to state
const { count, name } = toRefs(state)
count.value++          // state.count === 1
```

`toRef(state, 'count')` does the same for a single property. In 3.3+, `toRef` also accepts a getter — `toRef(() => props.foo)` — which is the neatest way to hand a read-only reactive value into a composable.

The price is that you're back to `.value`. That is not a coincidence: `.value` is what a live link *looks like* in JavaScript. Anything that gives you a bare value has, by definition, cut the link.

### `toValue` — the composable-author's tool

`toValue(x)` normalises **a ref, a getter function, or a plain value** down to a plain value, tracking it if it is reactive ([Composables](https://vuejs.org/guide/reusability/composables.html)):

```ts
import { toValue, type MaybeRefOrGetter } from 'vue'

function useDoubled(input: MaybeRefOrGetter<number>) {
  return computed(() => toValue(input) * 2)
}

useDoubled(2)               // plain
useDoubled(count)           // ref
useDoubled(() => a.value + b.value)  // getter
```

Accepting `MaybeRefOrGetter` and normalising with `toValue` is the modern convention for composable arguments; Part 10 leans on it heavily.

### `readonly`

`readonly(x)` returns a deeply read-only proxy that still tracks. It is how you hand state to a consumer who should observe but not mutate — the standard boundary for provide/inject (Part 8):

```ts
const count = ref(0)
const publicCount = readonly(count)
publicCount.value++    // dev warning; no change
```

[↑ Back to top](#table-of-contents)

---

## 5. Mastery: what `track` and `trigger` actually do

The docs give near-real pseudo-code, and it is worth reading once because every gotcha above falls out of it ([Reactivity in Depth](https://vuejs.org/guide/extras/reactivity-in-depth.html)):

```js
function reactive(obj) {
  return new Proxy(obj, {
    get(target, key)  { track(target, key);  return target[key] },
    set(target, key, value) { target[key] = value; trigger(target, key) },
  })
}

function ref(value) {
  const refObject = {
    get value() { track(refObject, 'value'); return value },
    set value(newValue) { value = newValue; trigger(refObject, 'value') },
  }
  return refObject
}
```

Vue keeps one global `activeEffect` and a `WeakMap<target, Map<key, Set<effect>>>` of subscribers:

```js
let activeEffect

function track(target, key) {
  if (activeEffect) {
    getSubscribersForProperty(target, key).add(activeEffect)
  }
}

function trigger(target, key) {
  getSubscribersForProperty(target, key).forEach(effect => effect())
}
```

And an effect sets itself as active while it runs:

```js
function whenDepsChange(update) {
  const effect = () => {
    activeEffect = effect
    update()
    activeEffect = null
  }
  effect()
}
```

Read the consequences off that code directly:

```
  ┌─ render effect of <TodoList> ─────────────┐
  │  runs template                            │
  │    reads items.value      ── track ──►  Set{ renderEffect }
  │    reads remaining.value  ── track ──►  Set{ renderEffect }   (computed)
  └───────────────────────────────────────────┘
                                    ▲
   items.value = [...]  ── trigger ─┘   re-runs ONLY that set
```

- **Tracking is per-property, on read.** `state.count` subscribes you to *`count` of that object*, not to `state`. This is why fine-grained updates are free, and why replacing the whole proxy strands your subscribers.
- **Tracking needs an active effect.** Reading a ref at module top level, outside any effect, subscribes nobody. This is also why lifecycle hooks and composables must be called synchronously during setup (Parts 9–10) — after an `await`, the "currently active" context is gone.
- **Only synchronous reads are tracked.** In `watchEffect(async () => { await x; y.value })`, `y` is read after the await and never tracked. Part 9 returns to this.
- **`computed` is a lazy effect** that caches its result and only re-runs when a dependency triggers *and* something reads it.
- **A component's render is an effect** — Vue conceptually does `watchEffect(() => updateDOM())` per component instance.

One nuance the pseudo-code elides: `reactive()` returns a *different object identity* than its target, so `toRaw(proxy) === original` but `proxy !== original`. Mixing raw and proxied references to the same object is a real source of "two things that should be equal aren't" bugs — §8 covers `toRaw`/`markRaw`.

[↑ Back to top](#table-of-contents)

---

# Part 2 — The Shape of Reactivity

*This Part is Working Knowledge → Mastery only. There is no honest Beginner tier for "which flavour of reactivity" — you need Part 1 in your head first. §6 is the bridge.*

## 6. Beginner: deep reactivity, and what it costs

Both `ref()` and `reactive()` are **deep** by default: nested objects and arrays are reactive too ([Reactivity Fundamentals](https://vuejs.org/guide/essentials/reactivity-fundamentals.html)).

```ts
const obj = ref({ nested: { count: 0 }, arr: ['foo'] })
obj.value.nested.count++     // works
obj.value.arr.push('bar')    // works
```

The mechanism: when you read a nested object through the proxy, Vue wraps *that* object in a proxy too, lazily, on access. So the cost is proportional to what you actually touch, not to the size of the object graph — but for a 10,000-row table that a template iterates in full, you are creating 10,000 proxies.

3.5's reactivity refactor cut memory by **56%** and made large deeply-reactive arrays up to **10× faster** ([Vue 3.5 announcement](https://blog.vuejs.org/posts/vue-3-5)), so this is much less often a problem than older blog posts suggest. Reach for shallow reactivity when you have measured, or when the data is genuinely owned by something else.

[↑ Back to top](#table-of-contents)

---

## 7. Working Knowledge: `shallowRef` and `shallowReactive`

`shallowRef(v)`: **only `.value` assignment is reactive**; the contents are left alone ([Reactivity API: Advanced](https://vuejs.org/api/reactivity-advanced.html)).

```ts
const state = shallowRef({ count: 1 })
state.value.count = 2      // ✗ no update — inner value is not reactive
state.value = { count: 2 } // ✓ update — .value was assigned
```

`shallowReactive(obj)`: only root-level properties are reactive; `state.nested.bar++` does nothing.

The trade the docs name explicitly: **you must treat the inner value as immutable and replace at the root** ([Performance](https://vuejs.org/guide/best-practices/performance.html)).

```ts
// ✗ wrong with shallowRef — mutation is invisible
rows.value.push(newRow)

// ✓ right — replace the root
rows.value = [...rows.value, newRow]
```

Three cases where this is the right call:

1. **Large immutable datasets** — a fetched 50k-row table you only ever replace wholesale.
2. **External state you don't own** — a Leaflet map instance, a WebGL scene, a third-party store's snapshot. Proxying these is at best wasted work and at worst breaks them (§9).
3. **High-frequency values** where you want exactly one notification per frame.

> **vs. React.** `shallowRef` + replace-the-array is the React mental model, opted into locally. If immutable updates feel more natural to you than mutation, note that Vue lets you work that way — you just don't get the extra performance unless you say `shallowRef`, because plain `ref` is paying for deep tracking you aren't using.

[↑ Back to top](#table-of-contents)

---

## 8. Advanced: `triggerRef`, `customRef`, `toRaw`, `markRaw`

### `triggerRef` — I mutated a shallow ref on purpose

```ts
const shallow = shallowRef({ greet: 'Hello, world' })
watchEffect(() => console.log(shallow.value.greet))

shallow.value.greet = 'Hello, universe'  // nothing logged
triggerRef(shallow)                      // logs "Hello, universe"
```

Legitimate when you're batching many mutations into one notification — mutate freely, `triggerRef` once at the end. Illegitimate as a routine patch for "my shallow ref isn't updating"; that usually means you wanted a deep `ref`.

### `customRef` — control track and trigger yourself

`customRef` hands you `track` and `trigger` and lets you decide when to call them. The canonical use is a debounced input, straight from the API docs ([Reactivity API: Advanced](https://vuejs.org/api/reactivity-advanced.html)):

```ts
import { customRef } from 'vue'

export function useDebouncedRef<T>(value: T, delay = 200) {
  let timeout: ReturnType<typeof setTimeout>
  return customRef<T>((track, trigger) => ({
    get() {
      track()
      return value
    },
    set(newValue) {
      clearTimeout(timeout)
      timeout = setTimeout(() => {
        value = newValue
        trigger()
      }, delay)
    },
  }))
}
```

Now `v-model="query"` on a `useDebouncedRef('')` debounces at the *state* layer — every consumer, template and watcher alike, sees the debounced value with no per-consumer wiring.

### `toRaw` and `markRaw` — leaving the proxy

`toRaw(proxy)` returns the original object; `markRaw(obj)` permanently opts an object out of ever being proxied.

```ts
const foo = {}
toRaw(reactive(foo)) === foo      // true

const inert = markRaw({ big: 'data' })
isReactive(reactive(inert))       // false
const bar = reactive({ inert })
isReactive(bar.inert)             // false — markRaw survives nesting
```

**Real Scenario — the identity bug.** A selection model stores `selected.value = row` where `row` came out of a `reactive` array, then a later comparison against a raw object from a fresh fetch fails: `selected.value === rows.find(...)` is `false` even though both "are" the same row. The cause is proxy identity — one side is the proxy, the other the raw object. Fixes, in order of preference: compare by `id` rather than identity; or normalise with `toRaw()` on both sides at the comparison. Reaching for `toRaw` to *write* through and skip reactivity is an escape hatch that hides updates from the UI — use it to read, not to mutate.

`markRaw` is the right tool for **class instances and third-party objects**: a `Chart.js` chart, a `MediaRecorder`, a map instance. Proxying these can break them outright, because their internals often rely on identity or on private fields that proxies interfere with.

[↑ Back to top](#table-of-contents)

---

## 9. Advanced: arrays, `Map`, `Set`, and class instances

**Arrays** are fully reactive in Vue 3 — index writes and `length` included, because a Proxy intercepts them (this is a real Vue 2 → Vue 3 improvement; if you find advice about `Vue.set` or `this.$set`, it is Vue 2 material and does not apply). Both mutation methods and replacement work ([List Rendering](https://vuejs.org/guide/essentials/list.html)):

```ts
const items = ref([1, 2, 3])
items.value.push(4)                          // ✓ tracked
items.value[0] = 99                          // ✓ tracked in Vue 3
items.value = items.value.filter(n => n > 1) // ✓ tracked
```

One trap that is not about reactivity but bites here: `sort()` and `reverse()` mutate in place. Inside a `computed` that means you are mutating a dependency from within a getter — copy first:

```ts
// ✗ wrong — mutates the source array during a getter
const sorted = computed(() => items.value.sort())

// ✓ right
const sorted = computed(() => [...items.value].sort())
```

**`Map` and `Set`** are supported as collection types by `reactive()`. Note the unwrapping gap from §4: refs stored *inside* arrays and collections are **not** unwrapped — `books[0].value`.

**Class instances** are proxied by default, which is usually not what you want: private fields (`#x`) and internal identity checks can misbehave through a proxy. `markRaw` them, or keep the instance in a plain `let` outside reactivity and hold reactive *snapshots* of the bits your template needs.

```ts
// ✓ a typical shape for wrapping a third-party object
const chart = markRaw(new Chart(canvas, config))
const chartTitle = ref('Sales')     // reactive UI state, separately
watch(chartTitle, t => { chart.options.plugins.title.text = t; chart.update() })
```

[↑ Back to top](#table-of-contents)

---

## 10. Mastery: `effectScope` and owning your own effects

Effects created synchronously inside `setup()` are auto-disposed when the component unmounts. Effects created **anywhere else** are not — the docs are explicit that async-created watchers must be stopped by hand ([Watchers](https://vuejs.org/guide/essentials/watchers.html)).

`effectScope()` gives you a container that owns a group of effects and can stop them all at once ([Reactivity API: Advanced](https://vuejs.org/api/reactivity-advanced.html)):

```ts
import { effectScope, computed, watch } from 'vue'

const scope = effectScope()

scope.run(() => {
  const doubled = computed(() => counter.value * 2)
  watch(doubled, v => console.log(v))
  watchEffect(() => console.log('count', counter.value))
})

scope.stop()    // every effect created inside run() is disposed
```

Where this actually matters:

- **A store or service outside any component** — a singleton that owns watchers needs a disposal story, and `effectScope` is it.
- **A composable that must clean up regardless of who called it** — `onScopeDispose(fn)` registers cleanup on the *current* scope, which works inside a component (fires on unmount) *and* inside a detached scope. That makes it strictly more reusable than `onUnmounted`, which is why Part 10 prefers it in composables.
- **Per-item effect lifetimes** — a scope per row, stopped when the row leaves.

```
  component instance
  └── effect scope (implicit)
        ├── render effect
        ├── computed from useThing()
        └── nested scope (yours, via effectScope)
              └── watch  ← scope.stop() kills these,
                            unmount kills the parent, which kills these too
```

Nested scopes are collected by the parent unless you pass `effectScope(true)` for a **detached** scope — detached means "I take full responsibility for stopping this."

[↑ Back to top](#table-of-contents)

---

## 11. Mastery: debugging reactivity

When something re-renders too often, or not at all, stop guessing. Vue ships hooks that tell you exactly which dependency did it ([Reactivity in Depth](https://vuejs.org/guide/extras/reactivity-in-depth.html)). These are **development-mode only**.

**Why did this component re-render?**

```ts
import { onRenderTracked, onRenderTriggered } from 'vue'

onRenderTracked(e => console.log('tracked', e.key, e))
onRenderTriggered(e => console.log('triggered by', e.key, e.oldValue, '→', e.newValue))
```

`onRenderTracked` fires once per dependency *collected* during render; `onRenderTriggered` fires when one of them causes a re-render. The `DebuggerEvent` carries `target`, `key`, `type` (`'get'`, `'set'`, …), `oldValue`, `newValue`.

**Why did this computed/watcher run?** Both accept `onTrack`/`onTrigger` options:

```ts
const plusOne = computed(() => count.value + 1, {
  onTrack(e) { debugger },
  onTrigger(e) { debugger },
})

watch(source, cb, { onTrigger(e) { debugger } })
```

A `debugger` statement in `onTrigger` is the fastest route to the *stack* that performed the offending write — far more useful than the key alone, because the question is usually "who set this?"

**Vue DevTools** gives you the same information visually — the component inspector shows each instance's reactive state, and the timeline records component renders. Enable `app.config.performance = true` to get Vue's own markers in the Chrome Performance panel ([Performance](https://vuejs.org/guide/best-practices/performance.html)); Part 13 uses that.

**Try It.** Find a phantom dependency:

```vue
<script setup lang="ts">
import { ref, onRenderTriggered } from 'vue'
const a = ref(0)
const b = ref(0)
onRenderTriggered(e => console.log('re-render caused by', e.key))
</script>

<template>
  <p>{{ a }}</p>
  <button @click="a++">a</button>
  <button @click="b++">b</button>
</template>
```

Clicking **a** logs `re-render caused by value`. Clicking **b** logs **nothing** — `b` is never read by the template, so the render effect never tracked it. That silence is the whole point of fine-grained reactivity, and it is also how you prove a suspected dependency is not one.

[↑ Back to top](#table-of-contents)

---

## Cheat Sheet

### Choosing a reactive primitive

| Situation | Use |
|---|---|
| Anything, by default | `ref()` |
| Primitive value | `ref()` (only option) |
| Cohesive object mutated in place, never replaced | `reactive()` |
| Derived value | `computed()` |
| Large dataset replaced wholesale | `shallowRef()` |
| Third-party / class instance | `markRaw()`, kept outside reactivity |
| Value handed out for reading only | `readonly()` |
| Debounced / validated write | `customRef()` |

### `reactive()`'s three limitations

| Limitation | Consequence | Fix |
|---|---|---|
| Object types only | `reactive(0)` is pointless | `ref()` |
| Can't replace the object | Reassignment strands subscribers | Mutate in place, or `ref()` |
| Not destructure-friendly | `const {x} = state` is a dead copy | `toRefs(state)`, or `ref()` |

### Unwrapping

| Context | Unwrapped? |
|---|---|
| `{{ count }}` (top-level, template) | ✅ |
| `state.count` where `state` is `reactive` | ✅ |
| `arr[0]` / `map.get(k)` inside `reactive` | ❌ `.value` needed |
| `{{ obj.id + 1 }}` where `obj` is a plain object | ❌ `[object Object]1` |
| Anywhere in `<script setup>` | ❌ `.value` always |

### Full API index

| API | One-line purpose |
|---|---|
| `ref` / `reactive` | Create reactive state |
| `computed` | Cached, lazy derived value (getter, or get/set pair) |
| `readonly` / `shallowReadonly` | Non-mutable view that still tracks |
| `toRef` / `toRefs` | Preserve the reactive link across destructuring |
| `toValue` | Normalise `MaybeRefOrGetter` → value, tracking it |
| `shallowRef` / `shallowReactive` | Track only the root level |
| `triggerRef` | Force notification after mutating a shallow ref |
| `customRef` | Hand-written `track`/`trigger` (debounce, validation) |
| `toRaw` / `markRaw` | Get the raw object / never proxy this object |
| `isRef` / `isReactive` / `isReadonly` / `isProxy` | Type predicates |
| `unref` | `isRef(x) ? x.value : x` (`toValue` is the newer superset) |
| `effectScope` / `getCurrentScope` / `onScopeDispose` | Group and dispose effects |
| `onRenderTracked` / `onRenderTriggered` | Dev-only: why did this render? |

**Next:** [Parts 3–4 — Templates, Directives, Events & Forms](./vue-templates-directives.md) · **Up:** [Overview](./vue-mastery-guide.md)

[↑ Back to top](#table-of-contents)
