# Vue 3 — Composables, Built-in Components, Directives & Styling (Parts 10–11)

> **Spec:** this doc follows the shared spec in [`vue-mastery-guide.md` → About This Document](./vue-mastery-guide.md#about-this-document) — curiosity-driven, `<script setup lang="ts">` only, Vue **3.5.42**, verified **September 6, 2026**.
>
> **Assumes** [Parts 1–2 — Reactivity](./vue-reactivity.md) and [Part 9 — Watchers & Lifecycle](./vue-slots-injection-watchers.md).

Part 10 is the payoff of the Composition API: reusable stateful logic that is just functions. Part 11 is the box of things Vue gives you that you'd otherwise write badly yourself — transitions, portals, async boundaries, instance caching — plus the two ways to attach behaviour and styles to elements.

---

## Table of Contents

**Part 10 — Composables**
1. [Beginner: your first composable](#1-beginner-your-first-composable)
2. [Working Knowledge: the conventions that make one reusable](#2-working-knowledge-the-conventions-that-make-one-reusable)
3. [Advanced: the call-site rules (and why they exist)](#3-advanced-the-call-site-rules-and-why-they-exist)
4. [Advanced: composables are not React hooks](#4-advanced-composables-are-not-react-hooks)
5. [Mastery: cleanup, scopes, and SSR safety](#5-mastery-cleanup-scopes-and-ssr-safety)

**Part 11 — Built-in Components, Directives & Styling**

6. [Working Knowledge: `<Transition>`](#6-working-knowledge-transition)
7. [Working Knowledge: `<TransitionGroup>`](#7-working-knowledge-transitiongroup)
8. [Working Knowledge: `<Teleport>`](#8-working-knowledge-teleport)
9. [Advanced: `<KeepAlive>`](#9-advanced-keepalive)
10. [Advanced: `<Suspense>` (experimental)](#10-advanced-suspense-experimental)
11. [Advanced: custom directives](#11-advanced-custom-directives)
12. [Working Knowledge: SFC styling](#12-working-knowledge-sfc-styling)
13. [Cheat Sheet](#cheat-sheet)

*Part 11 gets Working Knowledge → Advanced. There's no Beginner tier — you need components first — and its Mastery material is rendering internals, which is [Part 12](./vue-internals-performance.md).*

---

# Part 10 — Composables

## 1. Beginner: your first composable

A composable is a function that uses Vue's reactivity APIs to encapsulate **stateful logic** ([Composables](https://vuejs.org/guide/reusability/composables.html)). That's the entire definition — there is no special registration, no framework machinery.

```ts
// composables/useMouse.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  function update(event: MouseEvent) {
    x.value = event.pageX
    y.value = event.pageY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return { x, y }
}
```

```vue
<script setup lang="ts">
import { useMouse } from './composables/useMouse'
const { x, y } = useMouse()
</script>

<template>Mouse at {{ x }}, {{ y }}</template>
```

**Each call creates its own state.** Two components calling `useMouse()` get two independent pairs of refs and two independent listeners. That is the opposite of a store, and it's why composables compose without coordination.

Composables can call other composables, which is where the leverage comes from:

```ts
export function useMouseInElement(el: Ref<HTMLElement | null>) {
  const { x, y } = useMouse()                       // built on the one above
  const isInside = computed(() => { /* … */ })
  return { x, y, isInside }
}
```

[↑ Back to top](#table-of-contents)

---

## 2. Working Knowledge: the conventions that make one reusable

Four conventions, all from the official guide, that separate a composable you can drop into any component from one that only works where you first wrote it.

**1. Name it `useSomething`, camelCase.** Not decoration — tooling, lint rules, and readers all key on it.

**2. Return a plain object of refs, not a `reactive`.** This is what lets the caller destructure without losing reactivity:

```ts
// ✓ recommended — destructuring keeps the refs alive
return { x, y }
const { x, y } = useMouse()

// ✗ avoid — reactive(useMouse()) loses reactivity on destructure (Part 1)
```

**3. Accept refs, getters, *or* plain values.** Normalise with `toValue()`:

```ts
import { toValue, watchEffect, ref, type MaybeRefOrGetter } from 'vue'

export function useFetch<T>(url: MaybeRefOrGetter<string>) {
  const data = ref<T | null>(null)
  const error = ref<Error | null>(null)
  const loading = ref(false)

  watchEffect(async () => {
    const u = toValue(url)          // tracked: re-runs when a ref/getter source changes
    loading.value = true
    data.value = null
    error.value = null
    try {
      data.value = await (await fetch(u)).json()
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  })

  return { data, error, loading }
}
```

All three call styles now work, and the reactive ones re-fetch automatically:

```ts
useFetch('/api/todos')                       // static
useFetch(userUrl)                            // ref
useFetch(() => `/api/users/${id.value}`)     // getter — the most common
```

**4. Perform DOM side effects in `onMounted` and clean them up.** The docs' canonical example is worth memorising as a shape:

```ts
export function useEventListener(
  target: EventTarget,
  event: string,
  callback: EventListener,
) {
  onMounted(() => target.addEventListener(event, callback))
  onUnmounted(() => target.removeEventListener(event, callback))
}
```

Note that the `useFetch` above still has a race condition — see [Part 9 §13](./vue-slots-injection-watchers.md#13-advanced-cleanup-race-conditions-and-stopping-watchers). Adding `onWatcherCleanup(() => controller.abort())` is a good five-minute exercise; a production version would also want caching and retries, at which point you want a library.

[↑ Back to top](#table-of-contents)

---

## 3. Advanced: the call-site rules (and why they exist)

Composables that register lifecycle hooks or rely on `getCurrentInstance` must be called **synchronously during setup** — from `<script setup>`, from `setup()`, or transitively from a function those call synchronously ([Composables](https://vuejs.org/guide/reusability/composables.html)).

```ts
// ✓ fine
const { x } = useMouse()

// ✗ no active instance — onMounted inside would silently do nothing
onMounted(() => { const { x } = useMouse() })
watch(id, () => { const { x } = useMouse() })
```

The reason is [Part 1 §5](./vue-reactivity.md#5-mastery-what-track-and-trigger-actually-do): Vue tracks a "currently active instance" (and a currently active effect) in a module-level variable, set for the duration of the synchronous setup call. Once setup returns or you `await`, that variable is cleared and `onMounted` has nothing to attach to.

**The documented exception:** `<script setup>` is the one place a composable may be called *after* `await`, because the compiler restores the instance context around each `await` boundary:

```vue
<script setup lang="ts">
const data = await fetchSomething()   // requires <Suspense> — §10
const { x, y } = useMouse()           // ✓ works here, and only here
</script>
```

A composable that only creates refs and computeds and registers no hooks can technically be called anywhere. Don't rely on that distinction at call sites — a later refactor that adds an `onMounted` inside the composable would break every caller that broke the rule.

[↑ Back to top](#table-of-contents)

---

## 4. Advanced: composables are not React hooks

*Skip this section if React isn't in your head; nothing later depends on it.*

They look alike and solve the same problem, and the docs are explicit that the execution models differ: Vue composables are built on **fine-grained reactivity**, not on re-executing the component ([Composables](https://vuejs.org/guide/reusability/composables.html)).

What that changes, concretely:

| React hooks | Vue composables |
|---|---|
| Re-run on **every** render | Run **once**, when setup runs |
| Order must be identical each render → no conditionals, no loops | Order is irrelevant; call in an `if` if you like (as long as it's synchronous during setup) |
| `useEffect` needs a dependency array | `watch`/`watchEffect` track automatically |
| Stale closures are a constant hazard | Rare — the function body ran once and refs are live boxes |
| `useMemo`/`useCallback` to stabilise identity | Identity is not load-bearing |

```ts
// ✗ Rules of Hooks thinking, imported into Vue — this is fine in Vue!
if (props.trackMouse) {
  const { x } = useMouse()      // legal: no ordering requirement
}

// ✗ React reflex that does harm in Vue: over-parameterising to avoid "stale closures"
// ✓ Vue: the composable body ran once; `count` is a live ref. Just read count.value.
```

The trap worth naming: a React developer will reach for a dependency array or memoisation, find neither exists, and conclude something is missing. Nothing is missing — the tracking is automatic and per-property. The genuinely new discipline in the other direction is `.value` and *not breaking the tracking link* (Part 1), which has no React analogue.

[↑ Back to top](#table-of-contents)

---

## 5. Mastery: cleanup, scopes, and SSR safety

**Prefer `onScopeDispose` to `onUnmounted` inside composables.** `onUnmounted` requires a component instance; `onScopeDispose` attaches to the current *effect scope*, which exists both inside a component (disposed on unmount) and inside a standalone `effectScope()` ([Part 2 §10](./vue-reactivity.md#10-mastery-effectscope-and-owning-your-own-effects)). That makes the composable usable in a store or service too:

```ts
import { getCurrentScope, onScopeDispose } from 'vue'

export function useInterval(fn: () => void, ms: number) {
  const id = setInterval(fn, ms)
  const stop = () => clearInterval(id)
  if (getCurrentScope()) onScopeDispose(stop)   // auto-clean where possible
  return stop                                   // and always return manual control
}
```

**SSR safety.** The server runs your setup body; it never runs `onMounted`. So:

```ts
// ✗ wrong — ReferenceError: window is not defined, during SSR
const width = ref(window.innerWidth)

// ✓ right — a sane server default, real value after mount
const width = ref(0)
onMounted(() => {
  width.value = window.innerWidth
  useEventListener(window, 'resize', () => { width.value = window.innerWidth })
})
```

Two related traps:

- **Module-level state leaks across requests on the server.** A `const cache = ref({})` at module scope is shared by every user of a Node server. Per-request state belongs in a component or behind app-level `provide` ([Part 8](./vue-slots-injection-watchers.md#7-working-knowledge-typing-with-injectionkey-defaults-app-level-provide)).
- **Hydration mismatches from non-deterministic values.** `Math.random()`, `Date.now()`, and hand-rolled id counters produce different markup on server and client. `useId()` (3.5+) exists precisely for this: application-unique ids that are stable across server and client renders ([Vue 3.5](https://blog.vuejs.org/posts/vue-3-5)).

```ts
const id = useId()   // stable across SSR/CSR — use for label/input pairing, aria-describedby
```

**Try It.** Prove the scope rule:

```ts
import { effectScope } from 'vue'
const scope = effectScope()
const stop = scope.run(() => useInterval(() => console.log('tick'), 500))
setTimeout(() => scope.stop(), 2000)   // logs ~4 ticks, then silence
```

Ticking stops without any component being involved — that's what `onScopeDispose` bought you over `onUnmounted`.

[↑ Back to top](#table-of-contents)

---

# Part 11 — Built-in Components, Directives & Styling

## 6. Working Knowledge: `<Transition>`

`<Transition>` animates a **single element or component** entering and leaving, driven by `v-if`, `v-show`, or a dynamic component swap ([Transition](https://vuejs.org/guide/built-ins/transition.html)).

```vue
<template>
  <Transition>
    <p v-if="show">hello</p>
  </Transition>
</template>

<style scoped>
.v-enter-active, .v-leave-active { transition: opacity 0.5s ease; }
.v-enter-from,   .v-leave-to     { opacity: 0; }
</style>
```

Six classes, applied on this timeline:

```
  ENTER                                  LEAVE
  ├ v-enter-from    (1 frame)            ├ v-leave-from    (1 frame)
  ├ v-enter-active  (whole phase) ───┐   ├ v-leave-active  (whole phase) ───┐
  └ v-enter-to      (after 1 frame) ─┘   └ v-leave-to      (until done)  ───┘
      put the transition/duration on *-active
      put the start/end state on *-from / *-to
```

`name="fade"` swaps the `v-` prefix for `fade-`. `enter-active-class` and friends override individual class names entirely — that's how you drive an external library like Animate.css.

Four options you'll actually use:

```vue
<Transition mode="out-in">…</Transition>   <!-- leave finishes before enter starts -->
<Transition appear>…</Transition>          <!-- also animate the initial render -->
<Transition :duration="{ enter: 500, leave: 800 }">…</Transition>
<Transition :css="false" @enter="onEnter" @leave="onLeave">…</Transition>
```

**`mode="out-in"` is nearly always what you want when swapping two things.** Without it the outgoing and incoming elements are in the DOM simultaneously and the layout jumps:

```vue
<!-- ✗ both buttons present during the crossfade — the row visibly grows -->
<Transition>
  <button v-if="saved">Edit</button>
  <button v-else>Save</button>
</Transition>

<!-- ✓ -->
<Transition mode="out-in"> … </Transition>
```

**JavaScript hooks** for physics-based or library-driven animation. With `:css="false"` Vue skips class detection entirely and relies on you calling `done()`:

```ts
function onEnter(el: Element, done: () => void) {
  gsap.to(el, { opacity: 1, onComplete: done })   // forget done() and it hangs mid-transition
}
```

**Reusable transitions** are ordinary components wrapping `<Transition>` around a `<slot />`. One caveat the docs flag: `scoped` styles don't reach slotted content, so a reusable transition component's classes must be global (or `:slotted()`, §12).

[↑ Back to top](#table-of-contents)

---

## 7. Working Knowledge: `<TransitionGroup>`

For lists: multiple elements, entering, leaving, **and moving**.

```vue
<TransitionGroup name="list" tag="ul">
  <li v-for="item in items" :key="item.id">{{ item.text }}</li>
</TransitionGroup>
```

```css
.list-enter-active, .list-leave-active, .list-move { transition: all 0.4s ease; }
.list-enter-from, .list-leave-to { opacity: 0; transform: translateX(30px); }
.list-leave-active { position: absolute; }   /* take it out of flow so others can move */
```

Differences from `<Transition>`: it renders no element by default (pass `tag` if you want one), it does **not** support `mode`, and every child **must have a unique key** — the key is what tells it which element moved where.

The `*-move` class is the part people miss: it animates the FLIP-style repositioning of items displaced by an insertion or removal, which is the difference between a list that shuffles pleasantly and one that snaps. The `position: absolute` on `*-leave-active` is what lets the survivors slide into place while the departing item fades.

[↑ Back to top](#table-of-contents)

---

## 8. Working Knowledge: `<Teleport>`

Render markup somewhere else in the DOM while keeping it *logically* where it is ([Teleport](https://vuejs.org/guide/built-ins/teleport.html)).

```vue
<Teleport to="body">
  <div v-if="open" class="modal">…</div>
</Teleport>
```

The problem it solves: a modal nested six components deep inherits ancestors' `overflow: hidden`, `transform`, and z-index stacking contexts, and no amount of `position: fixed; z-index: 9999` reliably escapes them. Teleporting to `body` removes the constraint at the source.

**Logical hierarchy is preserved** — props, emits, and `inject` all still work as if the content were where you wrote it, and DevTools shows it in its original position.

```vue
<Teleport to="#modals" :disabled="isMobile">…</Teleport>   <!-- inline on mobile -->
<Teleport defer to="#late-target">…</Teleport>             <!-- 3.5+: target renders later -->
```

Multiple teleports to the same target append in order. `defer` (3.5+) postpones target resolution until after the current render cycle, which is the clean fix for "target element doesn't exist yet" — previously a `nextTick` dance.

[↑ Back to top](#table-of-contents)

---

## 9. Advanced: `<KeepAlive>`

Caches component **instances** instead of destroying them, so state and scroll position survive a switch ([KeepAlive](https://vuejs.org/guide/built-ins/keep-alive.html)).

```vue
<KeepAlive :max="10" :include="['TabA', 'TabB']">
  <component :is="currentTab" />
</KeepAlive>
```

`include`/`exclude` match the component's `name` — inferred from the filename for `<script setup>` SFCs (3.2.34+), or set explicitly with `defineOptions({ name: 'TabA' })`. `max` makes it an **LRU cache**: past the limit, the least recently used instance is destroyed.

Cached components don't unmount, so `onMounted`/`onUnmounted` stop firing on switch. Use the pair that does:

```ts
onActivated(() => refreshIfStale())   // also fires on initial mount
onDeactivated(() => pauseVideo())     // also fires on final unmount
```

Both hooks fire for the whole cached subtree, not just the root — so a deeply nested video player can pause itself without the tab component knowing it exists.

**The cost is memory and liveness.** A kept-alive component keeps its watchers, timers, and subscriptions running while invisible. `onDeactivated` is where you pause them — a polling interval that keeps firing for ten cached tabs is a real bug, and an easy one to ship because nothing looks wrong on screen.

[↑ Back to top](#table-of-contents)

---

## 10. Advanced: `<Suspense>` (experimental)

**Still marked experimental** in the current docs — API may change ([Suspense](https://vuejs.org/guide/built-ins/suspense.html)). Worth understanding; think twice before making it load-bearing in production.

It coordinates **async dependencies** in a subtree and shows one fallback until they all resolve:

```vue
<Suspense>
  <Dashboard />                                  <!-- #default, one root node -->
  <template #fallback>Loading dashboard…</template>
</Suspense>
```

Two things count as async dependencies: a component with an `async setup()`, and `<script setup>` with **top-level `await`**:

```vue
<!-- Dashboard.vue -->
<script setup lang="ts">
const stats = await fetchStats()    // this component now has an async dependency
</script>
```

Async components are suspensible by default, so a `defineAsyncComponent` inside the boundary participates too.

Behaviour worth knowing: after the initial resolve, it re-enters pending only when the root `#default` node is *replaced*; `timeout` controls how long the old content stays up before the fallback appears on such a replacement (`timeout="0"` shows it immediately). Nesting order with other built-ins is `<Transition>` → `<KeepAlive>` → `<Suspense>` → component.

**It has no error handling.** A rejected `await` in setup needs `onErrorCaptured` in a parent ([Part 9 §15](./vue-slots-injection-watchers.md#15-advanced-error-handling)) — so the honest shape is:

```vue
<ErrorBoundary>
  <Suspense>
    <Dashboard />
    <template #fallback><Spinner /></template>
  </Suspense>
</ErrorBoundary>
```

Given the experimental label plus the missing error story, the conservative default is still per-component `loading`/`error` state (the `useFetch` shape in §2), with `<Suspense>` reserved for cases where coordinating several async children into one loading state is genuinely worth it.

[↑ Back to top](#table-of-contents)

---

## 11. Advanced: custom directives

A custom directive is reusable **low-level DOM manipulation** attached to an element. In `<script setup>`, any camelCase variable named `vSomething` is automatically available as `v-something` ([Custom Directives](https://vuejs.org/guide/reusability/custom-directives.html)):

```vue
<script setup lang="ts">
const vAutofocus = {
  mounted: (el: HTMLElement) => el.focus(),
}
</script>

<template><input v-autofocus /></template>
```

The hooks mirror the component lifecycle: `created`, `beforeMount`, `mounted`, `beforeUpdate`, `updated`, `beforeUnmount`, `unmounted`. Each receives `(el, binding, vnode, prevVnode)`, where `binding` carries `value`, `oldValue`, `arg`, `modifiers`, and `instance`. So `v-example:foo.bar="baz"` gives `{ arg: 'foo', modifiers: { bar: true }, value: baz }`.

A realistic one — click-outside, with cleanup:

```ts
const vClickOutside = {
  mounted(el: HTMLElement & { _handler?: (e: Event) => void }, binding: { value: () => void }) {
    el._handler = (e: Event) => {
      if (!el.contains(e.target as Node)) binding.value()
    }
    document.addEventListener('click', el._handler)
  },
  unmounted(el: HTMLElement & { _handler?: (e: Event) => void }) {
    if (el._handler) document.removeEventListener('click', el._handler)
  },
}
```

The function shorthand (`app.directive('color', (el, binding) => …)`) registers the same callback for `mounted` and `updated` — fine for a pure "write this value to the element" directive, wrong the moment you need teardown.

**The official guidance is to prefer components.** Directives are for direct DOM manipulation that a component or a built-in directive can't express; they are harder to test, they carry an SSR caveat (they only run their DOM hooks on the client), and using one on a multi-root component causes warnings and unpredictable placement. Focus management, tooltips-via-a-library, intersection observation, and text selection are good uses. "Rendering something" never is.

[↑ Back to top](#table-of-contents)

---

## 12. Working Knowledge: SFC styling

`<style scoped>` rewrites selectors to include a per-component data attribute ([SFC CSS Features](https://vuejs.org/api/sfc-css-features.html)):

```css
/* .example { color: red } becomes: */
.example[data-v-f3f3eg9] { color: red; }
```

Four things the docs are explicit about, and all four surprise people:

**1. A child component's root element is affected by both the parent's and the child's scoped CSS.** This is deliberate, so the parent can lay the child out. It also means "scoped" doesn't mean "sealed".

**2. `:deep()` is required to reach further inside a child**, which you'll need constantly with third-party components:

```css
.wrapper :deep(.some-library-inner) { padding: 0; }
```

**3. Slotted content belongs to the parent**, so a child's scoped styles don't touch it — use `:slotted()`:

```css
:slotted(p) { margin: 0; }
```

**4. Descendant selectors are slower when scoped**, and recursive components make `.a .b` match further than you intended. Prefer a single class per rule.

`:global(.class)` escapes scoping for one rule. **CSS Modules** are the alternative when you want hashed names rather than attribute scoping:

```vue
<template><p :class="$style.red">red</p></template>
<style module>.red { color: red; }</style>
```

`v-bind()` in CSS links a style value to reactive state, compiled to a hashed CSS custom property that updates reactively:

```vue
<script setup lang="ts">
const theme = ref({ color: 'red' })
</script>

<style scoped>
p { color: v-bind('theme.color'); }
</style>
```

That last one is genuinely useful for runtime theming — one reactive object drives CSS without a single class toggle, and it keeps the values in the stylesheet where they belong rather than in inline `:style` bindings.

[↑ Back to top](#table-of-contents)

---

## Cheat Sheet

### Composable conventions

| Convention | Why |
|---|---|
| `useThing()`, camelCase | Recognisability, tooling |
| Return a plain object of refs | Destructuring keeps reactivity |
| Accept `MaybeRefOrGetter`, read with `toValue()` | Works with refs, getters and plain values |
| DOM side effects in `onMounted` | Doesn't run on the server |
| Clean up with `onScopeDispose` | Works in components *and* standalone scopes |
| Call synchronously during setup | Vue needs an active instance to attach hooks |
| Return a manual `stop()` too | Callers outside a scope can still clean up |

### Built-in components

| Component | Use for | Key props |
|---|---|---|
| `<Transition>` | One element/component entering/leaving | `name`, `mode="out-in"`, `appear`, `:css="false"`, `:duration` |
| `<TransitionGroup>` | Lists — enter, leave and **move** | `tag`, `name`; every child needs `:key` |
| `<Teleport>` | Render elsewhere in the DOM | `to`, `:disabled`, `defer` (3.5+) |
| `<KeepAlive>` | Cache instances across switches | `include`, `exclude`, `max` (LRU) |
| `<Suspense>` | One fallback for async children (**experimental**) | `timeout`, `#fallback`; no error handling |

### Transition classes

| Class | When |
|---|---|
| `v-enter-from` | Start of enter, removed after one frame |
| `v-enter-active` | Whole enter phase — put `transition` here |
| `v-enter-to` | Added after one frame, until done |
| `v-leave-from` / `v-leave-active` / `v-leave-to` | Same, leaving |
| `v-move` | `<TransitionGroup>` only — repositioning |

### Styling

| Feature | Syntax |
|---|---|
| Scoped | `<style scoped>` |
| Reach into a child | `.a :deep(.b)` |
| Style slotted content | `:slotted(p)` |
| One global rule | `:global(.red)` |
| CSS Modules | `<style module>` + `$style.red`, or `useCssModule()` |
| Reactive CSS value | `color: v-bind('theme.color')` |

### Gotcha index

| Symptom | Cause | Fix |
|---|---|---|
| `onMounted` in a composable never runs | Called async / outside setup | Call it synchronously during setup |
| `window is not defined` on the server | Browser API read in the setup body | Move into `onMounted` |
| Hydration mismatch on ids | Random/counter ids | `useId()` (3.5+) |
| Users see each other's data (SSR) | Module-level shared state | App-level `provide`, or per-component state |
| Transition does nothing | Missing `*-active` class or wrong `name` prefix | Check the six class names |
| Layout jumps mid-swap | Both elements present | `mode="out-in"` |
| JS transition hangs | `done()` never called with `:css="false"` | Call `done()` |
| List items teleport rather than slide | No `*-move` class, or leaving item still in flow | Add `*-move`; `position: absolute` on `*-leave-active` |
| Modal clipped by an ancestor | Stacking/overflow context | `<Teleport to="body">` |
| Teleport target not found | Target renders later | `<Teleport defer>` (3.5+) |
| Timers keep firing on a hidden tab | `<KeepAlive>` keeps instances live | Pause in `onDeactivated` |
| Can't style a third-party component's insides | Scoped CSS | `:deep()` |
| Reusable transition's classes ignored | Scoped styles don't reach slotted content | Global CSS or `:slotted()` |

**Next:** [Parts 12–13 — Internals & Performance](./vue-internals-performance.md) · **Up:** [Overview](./vue-mastery-guide.md)

[↑ Back to top](#table-of-contents)
