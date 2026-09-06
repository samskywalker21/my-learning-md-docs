# Vue 3 — Slots, Provide/Inject, Watchers & Lifecycle (Parts 7–9)

> **Spec:** this doc follows the shared spec in [`vue-mastery-guide.md` → About This Document](./vue-mastery-guide.md#about-this-document) — curiosity-driven, `<script setup lang="ts">` only, Vue **3.5.42**, verified **September 6, 2026**.
>
> **Assumes** [Parts 1–2 — Reactivity](./vue-reactivity.md) and [Parts 5–6 — Components](./vue-components-script-setup.md).

Three ways components relate to things outside themselves: **slots** (content passed down as markup), **provide/inject** (values passed down across any depth), and **watchers** (reacting to change over time). Part 9 also covers lifecycle and error handling, because both are about the same question — *when* does my code run relative to the DOM.

---

## Table of Contents

**Part 7 — Slots & Content Distribution**
1. [Beginner: the default slot and fallback content](#1-beginner-the-default-slot-and-fallback-content)
2. [Working Knowledge: named slots](#2-working-knowledge-named-slots)
3. [Working Knowledge: scoped slots](#3-working-knowledge-scoped-slots)
4. [Advanced: conditional slots, dynamic names, `defineSlots`](#4-advanced-conditional-slots-dynamic-names-defineslots)
5. [Mastery: renderless components, and when a composable is better](#5-mastery-renderless-components-and-when-a-composable-is-better)

**Part 8 — Provide / Inject**

6. [Beginner: `provide` and `inject`](#6-beginner-provide-and-inject)
7. [Working Knowledge: typing with `InjectionKey`, defaults, app-level provide](#7-working-knowledge-typing-with-injectionkey-defaults-app-level-provide)
8. [Advanced: keeping mutations with the provider](#8-advanced-keeping-mutations-with-the-provider)
9. [Advanced: when this stops being enough](#9-advanced-when-this-stops-being-enough)

**Part 9 — Watchers, Lifecycle & Errors**

10. [Beginner: `watch` and `watchEffect`](#10-beginner-watch-and-watcheffect)
11. [Working Knowledge: sources, and the ones that silently don't work](#11-working-knowledge-sources-and-the-ones-that-silently-dont-work)
12. [Working Knowledge: `deep`, `immediate`, `once`, `flush`](#12-working-knowledge-deep-immediate-once-flush)
13. [Advanced: cleanup, race conditions, and stopping watchers](#13-advanced-cleanup-race-conditions-and-stopping-watchers)
14. [Advanced: the lifecycle hooks](#14-advanced-the-lifecycle-hooks)
15. [Advanced: error handling](#15-advanced-error-handling)
16. [Cheat Sheet](#cheat-sheet)

---

# Part 7 — Slots & Content Distribution

## 1. Beginner: the default slot and fallback content

A slot is a hole in a child's template that the parent fills with markup ([Slots](https://vuejs.org/guide/components/slots.html)).

```vue
<!-- FancyButton.vue -->
<template>
  <button class="fancy-btn">
    <slot>Submit</slot>          <!-- "Submit" is fallback content -->
  </button>
</template>
```

```vue
<FancyButton>Click me!</FancyButton>   <!-- → <button class="fancy-btn">Click me!</button> -->
<FancyButton />                        <!-- → <button class="fancy-btn">Submit</button> -->
```

**Slot content is compiled in the parent's scope.** Whatever you write between the tags can read the *parent's* data and nothing of the child's:

```vue
<!-- ✗ wrong — `internalCount` belongs to FancyButton, not to us -->
<FancyButton>{{ internalCount }}</FancyButton>

<!-- ✓ right — parent data is available -->
<FancyButton>{{ user.name }}</FancyButton>
```

That lexical-scoping rule is exactly why scoped slots (§3) exist.

[↑ Back to top](#table-of-contents)

---

## 2. Working Knowledge: named slots

Multiple holes need names:

```vue
<!-- BaseLayout.vue -->
<template>
  <div class="container">
    <header><slot name="header" /></header>
    <main><slot /></main>                     <!-- implicitly name="default" -->
    <footer><slot name="footer" /></footer>
  </div>
</template>
```

```vue
<BaseLayout>
  <template #header><h1>Page title</h1></template>

  <p>Body content goes to the default slot.</p>

  <template #footer><p>© 2026</p></template>
</BaseLayout>
```

`#header` is shorthand for `v-slot:header`. Content not inside a named `<template>` goes to the default slot; you can also be explicit with `<template #default>`, which the docs require whenever you mix named slots with a *scoped* default slot.

**The design payoff.** Named slots turn "a component with 14 props for every possible variation" into "a component that defines a shape and lets you fill it". A `<Card>` with `#header`/`#footer`/default beats a `<Card :title :subtitle :icon :action-label :action-icon …>` — because the slot version accommodates the case you didn't anticipate without a code change.

[↑ Back to top](#table-of-contents)

---

## 3. Working Knowledge: scoped slots

The child has data the parent's markup needs — the row being rendered, the item's index, whether the panel is open. The child passes it *out* through the slot outlet as attributes:

```vue
<!-- DataTable.vue -->
<template>
  <table>
    <tr v-for="(row, index) in rows" :key="row.id">
      <slot name="row" :row="row" :index="index" :selected="row.id === selectedId" />
    </tr>
  </table>
</template>
```

```vue
<DataTable :rows="users">
  <template #row="{ row, index, selected }">
    <td>{{ index + 1 }}</td>
    <td :class="{ 'is-selected': selected }">{{ row.email }}</td>
  </template>
</DataTable>
```

For a default slot only, you can put `v-slot` straight on the component tag:

```vue
<MouseTracker v-slot="{ x, y }">Mouse is at {{ x }}, {{ y }}</MouseTracker>
```

This is the mechanism behind every good list/table/select component: the child owns *behaviour* (iteration, selection, keyboard handling, virtualisation) and the parent owns *appearance*. Neither has to anticipate the other.

```
   parent scope                       child scope
  ┌──────────────────────┐          ┌────────────────────────┐
  │ <DataTable :rows>    │  rows →  │ v-for over rows        │
  │   #row="{ row }" ────┼──────────┤ <slot :row="row" />    │
  │     markup using row │  ← row   │ (child data flows OUT) │
  └──────────────────────┘          └────────────────────────┘
   markup lives here,                behaviour lives here
   compiled in parent scope
```

[↑ Back to top](#table-of-contents)

---

## 4. Advanced: conditional slots, dynamic names, `defineSlots`

**Don't render an empty wrapper.** `$slots.x` is truthy only when the parent supplied that slot:

```vue
<template>
  <div class="card">
    <div v-if="$slots.header" class="card__header"><slot name="header" /></div>
    <div class="card__body"><slot /></div>
    <div v-if="$slots.footer" class="card__footer"><slot name="footer" /></div>
  </div>
</template>
```

Without the `v-if` you ship an empty `.card__header` whose padding and border are visible — a small bug that looks like a CSS problem and isn't.

**Dynamic slot names:**

```vue
<template v-for="col in columns" #[col.key]="{ row }" :key="col.key">
  {{ format(row[col.key]) }}
</template>
```

**`defineSlots` (3.3+)** types the contract, so a wrong destructure in the parent is a type error rather than `undefined` at runtime:

```vue
<script setup lang="ts">
defineSlots<{
  default(props: { msg: string }): any
  header(props: { title: string }): any
}>()
</script>
```

It is a **typing-only** macro — it generates no runtime code and doesn't declare the slots' existence to Vue.

[↑ Back to top](#table-of-contents)

---

## 5. Mastery: renderless components, and when a composable is better

A **renderless component** has no markup of its own — it computes state and hands it entirely to a scoped slot ([Slots](https://vuejs.org/guide/components/slots.html)):

```vue
<!-- MouseTracker.vue -->
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
const x = ref(0), y = ref(0)
const update = (e: MouseEvent) => { x.value = e.pageX; y.value = e.pageY }
onMounted(() => window.addEventListener('mousemove', update))
onUnmounted(() => window.removeEventListener('mousemove', update))
</script>

<template><slot :x="x" :y="y" /></template>
```

This is a genuinely useful pattern in React, where it (and higher-order components) was for years the *only* way to share stateful logic. **In Vue it usually isn't the right tool**, and the docs say so: composables achieve the same logic reuse without the overhead of an extra component instance.

```ts
// the same thing as a composable — no component instance, no slot indirection
const { x, y } = useMouse()
```

Keep the renderless form only when the reusable thing genuinely involves *rendering*: it needs a lifecycle tied to markup being present, it needs to wrap or transform the slot's vnodes, or the whole point is that the consumer supplies the markup (a headless `<Combobox>`). Reuse of plain logic → [Part 10](./vue-composables-builtins.md).

> **vs. React.** Render props and HOCs largely disappeared in React once hooks arrived, for exactly this reason. The Vue equivalent of that transition is renderless components → composables. If you learned render props first, notice that Vue's scoped slots remain excellent for *layout* composition even after they stop being the tool for *logic* reuse.

[↑ Back to top](#table-of-contents)

---

# Part 8 — Provide / Inject

## 6. Beginner: `provide` and `inject`

Prop drilling — passing a value through five components that don't use it — is the problem this solves ([Provide / Inject](https://vuejs.org/guide/components/provide-inject.html)).

```vue
<!-- ancestor -->
<script setup lang="ts">
import { provide, ref } from 'vue'
const theme = ref<'light' | 'dark'>('light')
provide('theme', theme)
</script>
```

```vue
<!-- any descendant, at any depth -->
<script setup lang="ts">
import { inject } from 'vue'
const theme = inject('theme')     // the same ref — still reactive
</script>
```

```
  App                provide('theme', themeRef)
   └── Layout                 │
        └── Sidebar           │  (Layout and Sidebar know nothing about it)
             └── ThemeToggle ─┘  inject('theme')
```

Provide the **ref**, not `theme.value` — passing the unwrapped value hands over a dead copy, exactly as in [Part 1](./vue-reactivity.md).

[↑ Back to top](#table-of-contents)

---

## 7. Working Knowledge: typing with `InjectionKey`, defaults, app-level provide

**String keys collide** across a large app and give you no types. Use a `Symbol` typed as an `InjectionKey` ([TypeScript with Composition API](https://vuejs.org/guide/typescript/composition-api.html)):

```ts
// keys.ts
import type { InjectionKey, Ref } from 'vue'

export interface ThemeContext {
  theme: Readonly<Ref<'light' | 'dark'>>
  toggle: () => void
}
export const themeKey = Symbol('theme') as InjectionKey<ThemeContext>
```

```ts
// provider
provide(themeKey, { theme: readonly(theme), toggle })

// consumer — typed as ThemeContext | undefined
const ctx = inject(themeKey)
```

**Defaults.** `inject(key)` returns `undefined` if nothing provided it — which is why the type includes `| undefined`. Three ways to deal with that:

```ts
const theme = inject(themeKey, defaultCtx)                    // a default value
const store = inject(storeKey, () => new Store(), true)       // factory (3rd arg = treat as factory)
const ctx = inject(themeKey)
if (!ctx) throw new Error('useTheme() must be used inside <ThemeProvider>')  // fail loudly
```

The third is usually the best for a required context: a clear error at the boundary beats `undefined` propagating three components deep. Wrapping it in a composable is the idiomatic packaging:

```ts
// useTheme.ts
export function useTheme(): ThemeContext {
  const ctx = inject(themeKey)
  if (!ctx) throw new Error('useTheme() must be used within a ThemeProvider')
  return ctx
}
```

**App-level provide** for things every component may need — an API client, a config object, an i18n instance:

```ts
const app = createApp(App)
app.provide(apiKey, createApiClient())
```

[↑ Back to top](#table-of-contents)

---

## 8. Advanced: keeping mutations with the provider

The failure mode of provide/inject is action at a distance: any of forty descendants can write to the injected state, and when it goes wrong you have forty suspects. The documented pattern is to **provide `readonly` state plus explicit mutation functions**, keeping every write in the provider:

```ts
// ✗ wrong — anyone can write, from anywhere
const location = ref('North Pole')
provide('location', location)

// ✓ right — reads are open, writes go through named functions
const location = ref('North Pole')
function updateLocation(next: string) {
  location.value = next
}
provide(locationKey, { location: readonly(location), updateLocation })
```

`readonly()` makes the boundary enforced rather than merely documented: a descendant that tries `location.value = 'x'` gets a development warning and no change ([Provide / Inject](https://vuejs.org/guide/components/provide-inject.html)).

This is the same shape a store has — state plus actions — which is the honest way to think about provide/inject: it is a store with a scope, and that scoping is its advantage. A `<Form>` providing form context serves *its own subtree*; two forms on a page don't collide. A global store would need instance juggling to do that.

[↑ Back to top](#table-of-contents)

---

## 9. Advanced: when this stops being enough

Provide/inject genuinely covers more than people expect: theming, form context, a table's selection model, an API client, i18n, modal stacks. Reach past it when you need something it structurally does not do:

| You need | provide/inject | Why |
|---|---|---|
| Subtree-scoped context | ✅ ideal | That's the design |
| Dependency injection of services | ✅ | App-level provide |
| State shared by *unrelated* trees | ❌ | There's no common ancestor to provide from |
| State that outlives the provider | ❌ | Dies when the provider unmounts |
| Devtools timeline / time-travel | ❌ | No instrumentation |
| SSR-safe singleton state | ⚠️ | Module-level singletons leak between requests; app-level provide is the fix |
| Server-state caching, dedup, retries | ❌ | Different problem entirely |

The first four rows are where a store (Pinia) earns its place, and the last is where a query library does. Both are out of scope here — see [Deliberately Not Covered](./vue-mastery-guide.md#6-deliberately-not-covered-and-where-to-look-instead). The reason to know the boundary precisely is that reaching for a store on day one is the more common mistake: a lot of apps ship a global store that exists solely to avoid passing two props.

[↑ Back to top](#table-of-contents)

---

# Part 9 — Watchers, Lifecycle & Errors

## 10. Beginner: `watch` and `watchEffect`

A watcher runs **side effects** in response to change — the things a `computed` must never do: fetching, logging, writing to storage, imperative DOM work ([Watchers](https://vuejs.org/guide/essentials/watchers.html)).

```ts
import { ref, watch, watchEffect } from 'vue'

const question = ref('')

// watch: explicit source, callback gets (new, old)
watch(question, async (newQ, oldQ) => {
  if (newQ.includes('?')) answer.value = await ask(newQ)
})

// watchEffect: runs immediately, tracks whatever it reads
watchEffect(() => {
  document.title = `${unread.value} unread — ${folder.value}`
})
```

| | `watch` | `watchEffect` |
|---|---|---|
| Dependencies | Explicit source | Auto-tracked from the body |
| First run | Not until the source changes (unless `immediate`) | Immediately |
| Old value | ✅ available | ❌ |
| Best when | You care about *which* thing changed, or need the old value | The effect reads several values and you want it kept in sync |

**The first rule of watchers is not to use one.** Derived state belongs in a `computed`; syncing two refs with a watcher is nearly always a design smell:

```ts
// ✗ wrong — a watcher maintaining derived state
const fullName = ref('')
watch([first, last], () => { fullName.value = `${first.value} ${last.value}` })

// ✓ right
const fullName = computed(() => `${first.value} ${last.value}`)
```

[↑ Back to top](#table-of-contents)

---

## 11. Working Knowledge: sources, and the ones that silently don't work

`watch` accepts a ref, a **getter function**, a reactive object, or an array of those ([Watchers](https://vuejs.org/guide/essentials/watchers.html)):

```ts
watch(x, cb)                          // a ref
watch(() => x.value + 1, cb)          // a getter
watch(obj, cb)                        // a reactive object → implicitly deep
watch([x, () => obj.count], cb)       // an array; callback gets arrays
```

The one that catches everybody:

```ts
// ✗ wrong — this passes a NUMBER, not a reactive source. Silent no-op.
watch(obj.count, cb)

// ✓ right
watch(() => obj.count, cb)
```

Why: arguments are evaluated before `watch` runs, so `obj.count` is a plain `0` by the time `watch` sees it. There is nothing there to subscribe to. A getter defers the read so Vue can run it inside a tracking context — the same reason `toValue` and `MaybeRefOrGetter` exist in [Part 10](./vue-composables-builtins.md).

Same trap with props, and it's more common because props *look* like plain objects:

```ts
// ✗ wrong                    // ✓ right
watch(props.userId, cb)       watch(() => props.userId, cb)
```

**`watchEffect` and `await`.** `watchEffect` only tracks what it reads **synchronously**. Anything after an `await` is invisible to it:

```ts
// ✗ wrong — `id` is tracked; `filter` is not, so filter changes are ignored
watchEffect(async () => {
  const data = await fetch(`/api/${id.value}`)
  apply(data, filter.value)
})

// ✓ right — read your dependencies up front…
watchEffect(async () => {
  const currentId = id.value
  const currentFilter = filter.value
  apply(await fetch(`/api/${currentId}`), currentFilter)
})

// ✓ …or use watch, where the sources are explicit
watch([id, filter], async ([i, f]) => apply(await fetch(`/api/${i}`), f))
```

[↑ Back to top](#table-of-contents)

---

## 12. Working Knowledge: `deep`, `immediate`, `once`, `flush`

```ts
watch(source, cb, { deep: true, immediate: true, once: true, flush: 'post' })
```

**`deep`** — watchers on a getter returning an object are shallow: they fire only when the *reference* changes. `deep: true` traverses. Since 3.5, `deep` accepts a **number** for bounded traversal, which is the escape from "deep-watching a large structure is expensive":

```ts
watch(() => state.settings, cb, { deep: true })   // any nested change
watch(() => state.settings, cb, { deep: 2 })      // 3.5+: only two levels down
```

Passing a `reactive` object directly (`watch(obj, cb)`) creates an implicit deep watcher — worth knowing, because it's easy to do accidentally and wonder why the callback fires so often. Note also that with a deep watcher on a mutated object, `newValue` and `oldValue` are **the same object** — Vue has nothing to compare against, since it was mutated in place. If you need the previous state, snapshot it yourself.

**`immediate: true`** — run once on creation. This is the standard "fetch on mount and on every change" shape:

```ts
watch(() => props.userId, async id => { user.value = await fetchUser(id) }, { immediate: true })
```

**`once: true`** (3.4+) — fire at most once.

**`flush`** — *when* the callback runs relative to the DOM update:

| `flush` | Runs | Use it for |
|---|---|---|
| `'pre'` (default) | Before the component's DOM update | Almost everything |
| `'post'` | After the DOM has been updated | Reading measurements, scrolling, third-party DOM libraries |
| `'sync'` | Synchronously, before any update, unbatched | Rare — integrating with something that demands it |

```ts
// ✗ wrong — the list DOM hasn't been updated yet; you measure the old height
watch(items, () => { height.value = listEl.value!.offsetHeight })

// ✓ right
watch(items, () => { height.value = listEl.value!.offsetHeight }, { flush: 'post' })
```

`watchPostEffect()` and `watchSyncEffect()` are aliases for `watchEffect` with those flushes. The docs warn that sync watchers have **no batching** — a mutation in a loop fires them once per iteration, so avoid them on frequently-mutated data.

`await nextTick()` inside a `'pre'` watcher is the ad-hoc equivalent of `flush: 'post'`, and is fine for a one-off.

[↑ Back to top](#table-of-contents)

---

## 13. Advanced: cleanup, race conditions, and stopping watchers

### The race condition, and `onWatcherCleanup`

Type fast in a search box and responses arrive out of order; the slowest request wins and the results don't match the query. `onWatcherCleanup()` (3.5+) registers a callback that runs **before the next invocation and on stop**:

```ts
import { watch, onWatcherCleanup } from 'vue'

watch(query, async q => {
  const controller = new AbortController()
  onWatcherCleanup(() => controller.abort())

  const res = await fetch(`/api/search?q=${encodeURIComponent(q)}`, {
    signal: controller.signal,
  })
  results.value = await res.json()
})
```

Each new keystroke aborts the in-flight request before starting the next, so a late response can never overwrite a newer one. The pre-3.5 form is the `onCleanup` argument passed to the callback (`watch(src, (val, old, onCleanup) => …)`), which still works; `onWatcherCleanup` is nicer because it can be called from a helper function rather than only where the argument is in scope.

Register cleanup **before** the first `await` — after one, the synchronous context is gone and it can't be attached.

### Stopping

Watchers created **synchronously** in `setup()`/`<script setup>` are bound to the component and stop on unmount. Watchers created **asynchronously** are not, and leak ([Watchers](https://vuejs.org/guide/essentials/watchers.html)):

```ts
// ✗ wrong — this watcher outlives the component
setTimeout(() => { watchEffect(() => { /* … */ }) }, 100)

// ✓ right — keep the handle and stop it
const stop = watchEffect(() => { /* … */ })
onUnmounted(stop)
```

Best practice is simply to create watchers synchronously. When you can't — a watcher whose lifetime is a modal, or one created inside a store — own the disposal explicitly, with the handle or with an [`effectScope`](./vue-reactivity.md#10-mastery-effectscope-and-owning-your-own-effects).

[↑ Back to top](#table-of-contents)

---

## 14. Advanced: the lifecycle hooks

```
  setup() body runs
        │
        ├─ onBeforeMount      no DOM yet
        ├─ onMounted          DOM exists; refs populated; browser APIs safe
        │     ⟳ on each update:
        │        ├─ onBeforeUpdate    DOM about to be patched
        │        └─ onUpdated         DOM patched (all children too)
        ├─ onBeforeUnmount    still fully functional — good place to save
        └─ onUnmounted        listeners/timers must be gone by now
```

Plus `onActivated`/`onDeactivated` for `<KeepAlive>` ([Part 11](./vue-composables-builtins.md)), `onErrorCaptured` (§15), and the dev-only `onRenderTracked`/`onRenderTriggered` ([Part 2](./vue-reactivity.md#11-mastery-debugging-reactivity)).

**The registration rule:** hooks must be registered **synchronously during setup** ([Lifecycle Hooks](https://vuejs.org/guide/essentials/lifecycle.html)). They don't have to be lexically inside `setup()` — a composable can register them — but the call stack must be synchronous, because Vue associates the callback with the *currently active instance*:

```ts
// ✗ wrong — no active instance by the time this runs
setTimeout(() => { onMounted(() => {}) }, 100)

// ✓ right — a composable called synchronously from setup registers fine
function useLogger() { onMounted(() => console.log('mounted')) }
useLogger()
```

**`onMounted` does not run during SSR.** That makes it the correct home for anything touching `window`, `document`, `IntersectionObserver`, or measurements — and it's why the composable convention in [Part 10](./vue-composables-builtins.md) puts DOM side effects there.

**Pair every subscription with its teardown, adjacently:**

```ts
onMounted(() => window.addEventListener('resize', onResize))
onUnmounted(() => window.removeEventListener('resize', onResize))
```

Writing them next to each other is the whole trick; the leaks happen when the two halves drift apart. Better still, extract the pair into a composable once and stop thinking about it.

**`onUpdated` is a trap.** It fires after *any* re-render of the component and all its children. Mutating state inside it re-renders, which fires it again — an infinite loop. If you want "when X changed, do Y", that is a `watch` on X, which is both precise and loop-free.

[↑ Back to top](#table-of-contents)

---

## 15. Advanced: error handling

**`onErrorCaptured`** catches errors thrown from any descendant — render errors, lifecycle hooks, watchers, event handlers:

```vue
<!-- ErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const error = ref<Error | null>(null)

onErrorCaptured((err, instance, info) => {
  error.value = err instanceof Error ? err : new Error(String(err))
  report(err, { info })     // `info` names the Vue-specific source, e.g. "render function"
  return false              // stop propagation; return true (or nothing) to let it bubble
})
</script>

<template>
  <slot v-if="!error" />
  <div v-else role="alert">
    <p>Something went wrong.</p>
    <button @click="error = null">Try again</button>
  </div>
</template>
```

Wrap the risky subtree: `<ErrorBoundary><Dashboard /></ErrorBoundary>`.

**Returning `false` stops propagation** — the error does not reach ancestors or the global handler. That is right for a boundary that genuinely handles the failure and wrong for one that only wants to *display* it while still reporting; in that case report inside the handler, or let it propagate.

**`app.config.errorHandler`** is the global net — the place to wire your error tracker:

```ts
app.config.errorHandler = (err, instance, info) => {
  Sentry.captureException(err, { extra: { info } })
}
```

**What it does not catch:** errors from an `async` callback that already returned (an unhandled promise rejection with no Vue frame on the stack) and errors outside Vue entirely. Add `window.addEventListener('unhandledrejection', …)` alongside it.

**`<Suspense>` has no error handling of its own** — the docs point you to `onErrorCaptured` in a parent for a failed async setup ([Suspense](https://vuejs.org/guide/built-ins/suspense.html)). Boundary above, suspense inside.

[↑ Back to top](#table-of-contents)

---

## Cheat Sheet

### Slots

| Task | Syntax |
|---|---|
| Declare a slot | `<slot />`, `<slot name="header" />` |
| Fallback content | `<slot>Default text</slot>` |
| Fill a named slot | `<template #header>…</template>` |
| Pass data out (child) | `<slot :row="row" :index="i" />` |
| Receive it (parent) | `<template #row="{ row, index }">` |
| Default scoped slot on the tag | `<Comp v-slot="{ x }">` |
| Only render a wrapper if filled | `v-if="$slots.header"` |
| Dynamic name | `<template #[key]>` |
| Type the contract | `defineSlots<{ … }>()` (3.3+, typing only) |

### Provide / inject

| Task | Code |
|---|---|
| Provide | `provide(key, value)` — pass the **ref**, not `.value` |
| Inject | `const v = inject(key)` |
| Typed key | `Symbol() as InjectionKey<T>` |
| Default value | `inject(key, fallback)` |
| Factory default | `inject(key, () => new X(), true)` |
| Required, fail loudly | `if (!v) throw new Error(…)` — wrap in a `useX()` composable |
| App-wide | `app.provide(key, value)` |
| Read-only boundary | `provide(key, { state: readonly(s), update })` |

### Watchers

| Need | Code |
|---|---|
| Watch a ref | `watch(x, cb)` |
| Watch a property | `watch(() => obj.count, cb)` — **never** `watch(obj.count, cb)` |
| Watch a prop | `watch(() => props.id, cb)` |
| Several sources | `watch([a, () => b.c], ([a, c]) => …)` |
| Run immediately | `{ immediate: true }` |
| Deep, bounded | `{ deep: 2 }` (3.5+) |
| Once | `{ once: true }` (3.4+) |
| After DOM update | `{ flush: 'post' }` or `watchPostEffect` |
| Cancel in-flight work | `onWatcherCleanup(() => controller.abort())` |
| Stop manually | `const stop = watch(…); onUnmounted(stop)` |
| Auto-tracked effect | `watchEffect(fn)` — sync reads only |

### Lifecycle

| Hook | Fires | Typical use |
|---|---|---|
| `onBeforeMount` | Before first DOM | Rarely needed |
| `onMounted` | DOM ready, refs populated; **not on SSR** | Listeners, measurements, browser APIs |
| `onBeforeUpdate` | Before a patch | Capture pre-update measurements |
| `onUpdated` | After a patch (children included) | Almost never — use `watch` |
| `onBeforeUnmount` | Still functional | Persist drafts |
| `onUnmounted` | Gone | Remove listeners/timers |
| `onActivated` / `onDeactivated` | `<KeepAlive>` in/out | Refresh on return, pause on leave |
| `onErrorCaptured` | Descendant threw | Error boundary |
| `onRenderTracked` / `onRenderTriggered` | Dev only | Debug re-renders |

### Gotcha index

| Symptom | Cause | Fix |
|---|---|---|
| Watcher never fires | `watch(obj.count, cb)` | `watch(() => obj.count, cb)` |
| Watcher misses one dependency | Read after `await` in `watchEffect` | Read up front, or use `watch` |
| Measurement is one update stale | Default `flush: 'pre'` | `flush: 'post'` |
| Stale search results | Out-of-order responses | `onWatcherCleanup` + `AbortController` |
| Watcher runs after unmount | Created asynchronously | Create synchronously, or stop it |
| `onMounted` "doesn't run" | Registered asynchronously | Register synchronously in setup |
| Infinite update loop | State mutated in `onUpdated` | Use `watch` |
| Empty styled wrapper renders | No `v-if="$slots.x"` | Guard the wrapper |
| Injected value is `undefined` | No provider above, or key mismatch | Typed `InjectionKey` + explicit error |
| Injected state changes from nowhere | Consumers mutating it | `readonly()` + provided actions |

**Next:** [Parts 10–11 — Composables & Built-ins](./vue-composables-builtins.md) · **Up:** [Overview](./vue-mastery-guide.md)

[↑ Back to top](#table-of-contents)
