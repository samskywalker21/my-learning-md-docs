# Vue 3 — Components & `<script setup>` (Parts 5–6)

> **Spec:** this doc follows the shared spec in [`vue-mastery-guide.md` → About This Document](./vue-mastery-guide.md#about-this-document) — curiosity-driven, `<script setup lang="ts">` only, Vue **3.5.42**, verified **September 6, 2026**.
>
> **Assumes** [Parts 1–2 — Reactivity](./vue-reactivity.md) and [Parts 3–4 — Templates](./vue-templates-directives.md).

`<script setup>` is compile-time sugar over the Composition API, and its **compiler macros** — `defineProps`, `defineEmits`, `defineModel` and friends — are the API surface you'll touch every day. They are not imported and not runtime functions: they are instructions to the compiler, which is why they have rules ordinary functions don't ([`<script setup>`](https://vuejs.org/api/sfc-script-setup.html)).

---

## Table of Contents

**Part 5 — Components & `<script setup>`**
1. [Beginner: an SFC, and how components get used](#1-beginner-an-sfc-and-how-components-get-used)
2. [Working Knowledge: `defineProps`](#2-working-knowledge-defineprops)
3. [Working Knowledge: `defineEmits`](#3-working-knowledge-defineemits)
4. [Working Knowledge: `defineModel`](#4-working-knowledge-definemodel)
5. [Advanced: fallthrough attributes and `defineOptions`](#5-advanced-fallthrough-attributes-and-defineoptions)
6. [Advanced: `defineExpose` and template refs](#6-advanced-defineexpose-and-template-refs)
7. [Mastery: generic components, and what the macros compile to](#7-mastery-generic-components-and-what-the-macros-compile-to)

**Part 6 — Dynamic & Async Components**

8. [Working Knowledge: `<component :is>`](#8-working-knowledge-component-is)
9. [Working Knowledge: `defineAsyncComponent`](#9-working-knowledge-defineasynccomponent)
10. [Advanced: lazy hydration (3.5+)](#10-advanced-lazy-hydration-35)
11. [Cheat Sheet](#cheat-sheet)

*Part 6 gets Working Knowledge → Advanced only. There is no meaningful Beginner tier — dynamic components only make sense once you have several — and its Mastery material is bundle/hydration strategy, which belongs to [Part 13](./vue-internals-performance.md).*

---

# Part 5 — Components & `<script setup>`

## 1. Beginner: an SFC, and how components get used

A Single-File Component is three blocks in one `.vue` file:

```vue
<script setup lang="ts">
import { ref } from 'vue'
const open = ref(false)
</script>

<template>
  <section :class="{ 'is-open': open }">
    <button @click="open = !open">Toggle</button>
    <p v-if="open">Contents</p>
  </section>
</template>

<style scoped>
.is-open { border-color: currentColor; }
</style>
```

Three facts that make everything else make sense:

**Everything at the top level of `<script setup>` is available to the template.** Refs, functions, imports, types — no `return` statement, no `setup()` boilerplate. That's the whole point of the sugar.

**The script body runs once per component instance**, when that instance is created. It is the equivalent of the Options API's `created`. Anything you write there — a `console.log`, a `fetch` — happens per instance.

**Imported components need no registration.** Import it and use it:

```vue
<script setup lang="ts">
import TodoRow from './TodoRow.vue'
</script>

<template>
  <TodoRow v-for="t in todos" :key="t.id" :todo="t" @toggle="onToggle" />
</template>
```

Use `PascalCase` in templates. It distinguishes your components from native elements at a glance, and it's what the docs and tooling assume. `kebab-case` also works and is required in in-DOM templates (a no-build setup), which is a good reason to avoid in-DOM templates.

[↑ Back to top](#table-of-contents)

---

## 2. Working Knowledge: `defineProps`

Props are a component's inputs: read-only, one-way, parent to child.

### Type-based declaration (what to use with TypeScript)

```vue
<script setup lang="ts">
interface Props {
  todo: Todo
  compact?: boolean
  labels?: string[]
}

const { todo, compact = false, labels = [] } = defineProps<Props>()
</script>
```

That destructure is **reactive props destructure**, stable since 3.5 ([`<script setup>`](https://vuejs.org/api/sfc-script-setup.html)). The compiler rewrites every use of `compact` into `props.compact`, so the binding stays live — this is compile-time magic, and it only works on the destructure of a `defineProps` call.

Defaults are plain JavaScript defaults. Before 3.5 you needed `withDefaults`:

```ts
// pre-3.5 form you'll still see everywhere — still works
const props = withDefaults(defineProps<Props>(), {
  compact: false,
  labels: () => [],        // note: mutable defaults must be factory functions
})

// 3.5+ form used in this set
const { compact = false, labels = [] } = defineProps<Props>()
```

### Runtime declaration

Without TypeScript (or when you want runtime validators):

```ts
const props = defineProps({
  todo: { type: Object, required: true },
  compact: { type: Boolean, default: false },
  status: {
    type: String,
    validator: (v: string) => ['todo', 'doing', 'done'].includes(v),
  },
})
```

You cannot use runtime and type declarations together in one component ([`<script setup>`](https://vuejs.org/api/sfc-script-setup.html)). Type-based is the default for this set; runtime declaration earns its place when you want a `validator` to catch bad input at runtime in development.

### The rules that trip people

**Props are read-only.** Assigning to one is a development warning:

```ts
// ✗ wrong
props.count++

// ✓ right — emit and let the owner change it (§3), or derive locally
const localCount = ref(props.count)              // a snapshot, deliberately
const doubled = computed(() => props.count * 2)  // derived, stays in sync
```

That `ref(props.count)` line is worth pausing on: it copies *once*. If the parent later changes the prop, `localCount` does not follow. That is sometimes exactly what you want (an editable draft of a value) and sometimes a bug — if you want it to follow, use `computed`, or `watch` the prop.

**`camelCase` in script, `kebab-case` in templates.** `defineProps<{ maxLength: number }>()` is passed as `:max-length="10"`. Both spellings work in templates; kebab matches HTML convention and is what most codebases use.

**Boolean casting.** A declared `boolean` prop with no value present is `true` (`<Comp compact />`), and absent is `false` — DOM-attribute semantics, not JavaScript semantics.

**Passing objects doesn't clone them.** `:todo="todo"` passes the same reactive object. The child can mutate it, and it will work — and it is still a bad idea, because the ownership of that state becomes unclear. Emit instead.

[↑ Back to top](#table-of-contents)

---

## 3. Working Knowledge: `defineEmits`

Events are a component's outputs.

```vue
<script setup lang="ts">
// tuple syntax (3.3+) — the shortest correct form
const emit = defineEmits<{
  toggle: [id: string]
  update: [value: string, source: 'user' | 'sync']
  close: []
}>()

function onClick() {
  emit('toggle', props.todo.id)
}
</script>
```

The older call-signature form is equivalent and still valid:

```ts
const emit = defineEmits<{
  (e: 'toggle', id: string): void
  (e: 'close'): void
}>()
```

Runtime form, if you're not using TypeScript: `const emit = defineEmits(['toggle', 'close'])`.

The parent listens with `v-on`:

```vue
<TodoRow :todo="t" @toggle="onToggle" @update="(v, src) => …" />
```

**Why declare emits at all?** Three reasons, and the third is the one that bites:

1. Type-checking and editor autocomplete on both sides.
2. Documentation — the component's contract is visible in one place.
3. **Declared events are removed from fallthrough attributes.** An undeclared `@close` on a component whose root is a `<button>` ends up as a native listener on that button *as well as* your emit — so a single user click fires the handler twice.

```ts
// ✗ wrong — parent's @close fires twice: once from emit, once as fallthrough
// (no defineEmits at all)
// ✓ right
const emit = defineEmits<{ close: [] }>()
```

**Naming.** `camelCase` in `emit()`, `kebab-case` in templates: `emit('updateTitle')` is listened to as `@update-title`. Being consistent — this set uses lowercase single words or kebab-case throughout — avoids the class of bug where a listener silently never fires.

[↑ Back to top](#table-of-contents)

---

## 4. Working Knowledge: `defineModel`

A component that supports `v-model` needs a `modelValue` prop and an `update:modelValue` event. `defineModel()` (3.4+) declares both and hands you a writable ref ([`<script setup>`](https://vuejs.org/api/sfc-script-setup.html)):

```vue
<!-- CurrencyInput.vue -->
<script setup lang="ts">
const model = defineModel<number>()
</script>

<template>
  <input :value="model" @input="model = Number(($event.target as HTMLInputElement).value)" />
  <button @click="model = 0">clear</button>
</template>
```

```vue
<!-- parent -->
<CurrencyInput v-model="price" />
```

Writing `model.value = x` emits `update:modelValue`; reading it reads the prop. The pre-3.4 version of exactly this was:

```ts
// the old boilerplate defineModel replaces — you'll see it in existing code
const props = defineProps<{ modelValue: number }>()
const emit = defineEmits<{ 'update:modelValue': [value: number] }>()
const model = computed({
  get: () => props.modelValue,
  set: v => emit('update:modelValue', v),
})
```

**Named models** for multiple bindings:

```ts
const first = defineModel<string>('firstName')
const last = defineModel<string>('lastName')
```
```vue
<NameFields v-model:first-name="a" v-model:last-name="b" />
```

**Modifiers and transformers** — a custom `.capitalize` modifier:

```ts
const [model, modifiers] = defineModel<string>({
  set(value) {
    return modifiers.capitalize ? value[0].toUpperCase() + value.slice(1) : value
  },
})
```
```vue
<MyInput v-model.capitalize="title" />
```

**The documented default-value trap.** `defineModel({ default: 0 })` desynchronises parent and child when the parent passes no value: the child sees `0`, the parent's variable stays `undefined`, and they disagree until the first write. Prefer requiring the parent to provide a value; if you must have a default, make sure the parent initialises the bound ref.

[↑ Back to top](#table-of-contents)

---

## 5. Advanced: fallthrough attributes and `defineOptions`

An attribute passed to a component that isn't a declared prop or emit "falls through" onto the component's root element — `class`, `style`, `id`, `data-*`, `aria-*`, native listeners. This is why `<MyButton class="mt-4" @click="…" />` usually just works.

Three situations where you need to intervene:

**Multi-root components have no single root**, so Vue can't guess. You must place `$attrs` explicitly:

```vue
<template>
  <label :for="id">{{ label }}</label>
  <input v-bind="$attrs" :id="id" />   <!-- ✓ attrs land here, deliberately -->
</template>
```

**A wrapper whose root isn't the interesting element** — you want attributes on the inner `<input>`, not the outer `<div>`:

```vue
<script setup lang="ts">
defineOptions({ inheritAttrs: false })   // stop the automatic placement
</script>

<template>
  <div class="field">
    <input v-bind="$attrs" />
  </div>
</template>
```

`defineOptions()` (3.3+) is how you declare component options — `inheritAttrs`, `name`, custom options — that have no other macro. Its contents are hoisted to module scope, so it **cannot reference local setup variables**.

**You need to read the attrs in script:** `const attrs = useAttrs()`. Note `attrs` is *not* reactive-per-property — the docs' guidance is to prefer `$attrs` in the template.

> **vs. React.** This is Vue's answer to `{...rest}` spreading, except it happens automatically. The cost of the convenience is that "where did this class come from?" is occasionally non-obvious; `inheritAttrs: false` plus explicit `v-bind="$attrs"` is how you make it obvious again.

[↑ Back to top](#table-of-contents)

---

## 6. Advanced: `defineExpose` and template refs

**Components using `<script setup>` are closed by default.** A parent holding a template ref to one gets an instance that exposes nothing ([`<script setup>`](https://vuejs.org/api/sfc-script-setup.html)). This is deliberate: it keeps the component's contract to props/events/slots instead of "whatever happens to be in scope."

To open a hole in that wall on purpose:

```vue
<!-- VideoPlayer.vue -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
const el = useTemplateRef<HTMLVideoElement>('video')

function play() { el.value?.play() }
function seek(t: number) { if (el.value) el.value.currentTime = t }

defineExpose({ play, seek })
</script>

<template><video ref="video" /></template>
```

```vue
<!-- parent -->
<script setup lang="ts">
import VideoPlayer from './VideoPlayer.vue'
const player = useTemplateRef<InstanceType<typeof VideoPlayer>>('player')
function start() { player.value?.play() }
</script>

<template><VideoPlayer ref="player" /><button @click="start">Play</button></template>
```

### Template refs, 3.5 and before

`useTemplateRef('name')` (3.5+) takes the *string ID* from the `ref` attribute. Before 3.5, the convention was a `ref(null)` whose **variable name had to match** the attribute — brittle, and impossible with dynamic names ([Template Refs](https://vuejs.org/guide/essentials/template-refs.html)):

```ts
// pre-3.5, still works — variable name must equal ref="input"
const input = ref<HTMLInputElement | null>(null)

// 3.5+ — used in this set
const input = useTemplateRef('input')
```

**Refs are `null` until mounted**, and back to `null` after unmount. Anything touching one belongs in `onMounted` or a `watchEffect`:

```ts
// ✗ wrong — runs during setup, before the DOM exists
input.value?.focus()   // silently does nothing

// ✓ right
onMounted(() => input.value?.focus())
```

**Refs inside `v-for`** collect into an array (3.5+ populates it automatically). The array's order is **not guaranteed** to match the source array — if you need the pairing, use a function ref: `:ref="el => byId[item.id] = el"`.

**Use this sparingly.** Every `defineExpose` is a crack in the component boundary. Legitimate uses are imperative DOM operations a declarative prop can't express — focus, media playback, canvas, scroll position, `showModal()`. "It was easier than emitting an event" is not one.

[↑ Back to top](#table-of-contents)

---

## 7. Mastery: generic components, and what the macros compile to

### Generic components

`<script setup>` takes a `generic` attribute, giving you a genuinely type-safe reusable list, table, or select ([TypeScript with Composition API](https://vuejs.org/guide/typescript/composition-api.html)):

```vue
<!-- DataList.vue -->
<script setup lang="ts" generic="T extends { id: string }">
defineProps<{
  items: T[]
  getLabel: (item: T) => string
}>()

const emit = defineEmits<{ select: [item: T] }>()
</script>

<template>
  <li v-for="item in items" :key="item.id" @click="emit('select', item)">
    {{ getLabel(item) }}
  </li>
</template>
```

At the call site, `T` is inferred from `items`, so `getLabel` and the `@select` payload are both correctly typed — `<DataList :items="users" :get-label="u => u.email" @select="u => u.email" />` type-checks, and `u.emial` doesn't. Imported types work too (`generic="T extends Item"` with `import type { Item } from './types'`).

### What the macros actually are

None of `defineProps`/`defineEmits`/`defineModel`/`defineExpose`/`defineOptions`/`defineSlots` exists at runtime. The SFC compiler finds them, deletes the call, and emits an ordinary component object. That explains their rules:

- **They must be at the top level of `<script setup>`** — the compiler pattern-matches on the call site. You cannot call one conditionally, in a function, or in a composable.
- **They cannot reference local variables** — the generated options object is hoisted above your setup code, so anything you pass must be a literal or an import.
- **Type-only declarations work because the types are erased at compile time** — the compiler reads `defineProps<Props>()`'s type argument and generates the *runtime* props declaration from it, which is why the type has to be statically analysable (an inline literal, an interface in the same file, or an import — not a mapped type built from a value).

The rough shape of the output for a component with props, emits and a model:

```js
export default {
  props: { modelValue: { type: Number }, compact: { type: Boolean, default: false } },
  emits: ['update:modelValue', 'toggle'],
  setup(props, { emit, expose }) {
    // your <script setup> body, with `compact` rewritten to `props.compact`
    expose({ /* whatever defineExpose received, or nothing */ })
    return (/* compiled render function */)
  },
}
```

Two payoffs from knowing this. First, error messages like "`defineProps` is a compiler macro and no longer needs to be imported" or "cannot reference locally declared variables" stop being mysterious. Second, you understand why reactive props destructure is *safe* despite everything [Part 1](./vue-reactivity.md#2-working-knowledge-reactive-and-why-ref-is-still-the-default) says about destructuring reactive objects: nothing is destructured at runtime — the compiler rewrote every read into `props.x`.

[↑ Back to top](#table-of-contents)

---

# Part 6 — Dynamic & Async Components

## 8. Working Knowledge: `<component :is>`

`<component :is="X">` renders whichever component `X` names — a tab set, a wizard step, a form field chosen by type.

```vue
<script setup lang="ts">
import { shallowRef } from 'vue'
import TextField from './TextField.vue'
import SelectField from './SelectField.vue'

const map = { text: TextField, select: SelectField }
const current = shallowRef(TextField)
</script>

<template>
  <component :is="current" v-bind="fieldProps" @change="onChange" />
  <component :is="map[field.type]" />
  <component :is="'h' + level">Dynamic heading</component>  <!-- native element by name -->
</template>
```

**Use `shallowRef` for the component itself**, not `ref`. A component definition is a plain object; making it deeply reactive is wasted proxying and can produce warnings — this is the [`markRaw`/`shallowRef` case from Part 2](./vue-reactivity.md#7-working-knowledge-shallowref-and-shallowreactive) in its most common clothing.

Switching `:is` unmounts the old component and mounts the new one — state is lost. Wrap in `<KeepAlive>` ([Part 11](./vue-composables-builtins.md)) if that's wrong for your case.

[↑ Back to top](#table-of-contents)

---

## 9. Working Knowledge: `defineAsyncComponent`

Load a component's code only when it's first rendered — the route-level or modal-level code split ([Async Components](https://vuejs.org/guide/components/async.html)):

```ts
import { defineAsyncComponent } from 'vue'

const SettingsDialog = defineAsyncComponent(() => import('./SettingsDialog.vue'))
```

For anything user-facing, use the options form so a slow network doesn't show a blank hole:

```ts
const SettingsDialog = defineAsyncComponent({
  loader: () => import('./SettingsDialog.vue'),
  loadingComponent: Spinner,
  delay: 200,          // wait 200ms before showing the spinner (default 200)
  errorComponent: LoadFailed,
  timeout: 10_000,     // after this, render errorComponent (default Infinity)
})
```

`delay` exists to prevent a spinner flash on a fast connection; `timeout` defaults to `Infinity`, meaning a hung request shows the loading state forever — set it.

**Where the split boundary belongs:** around things that are large *and* not needed on first paint — dialogs, editors, charts, admin panels. Splitting a 2 KB component adds a network round-trip to save nothing.

**Async components are suspensible by default** — inside a `<Suspense>` boundary they participate in it rather than showing their own loading state, unless you set `suspensible: false` ([Part 11](./vue-composables-builtins.md)).

[↑ Back to top](#table-of-contents)

---

## 10. Advanced: lazy hydration (3.5+)

*SSR-only. Skip this if you render entirely on the client.*

In an SSR app, the server sends HTML and the client then **hydrates** it — walking the tree, attaching listeners, making it interactive. Hydration cost is paid up front for the entire page, including the footer nobody scrolls to.

Vue 3.5 added **lazy hydration strategies** for async components ([Async Components](https://vuejs.org/guide/components/async.html)):

```ts
import { defineAsyncComponent, hydrateOnVisible, hydrateOnIdle,
         hydrateOnInteraction, hydrateOnMediaQuery } from 'vue'

const Comments = defineAsyncComponent({
  loader: () => import('./Comments.vue'),
  hydrate: hydrateOnVisible({ rootMargin: '100px' }),   // IntersectionObserver
})
```

| Strategy | Hydrates when | Good for |
|---|---|---|
| `hydrateOnVisible(opts)` | It scrolls into view (`IntersectionObserver`) | Below-the-fold sections |
| `hydrateOnIdle(timeout?)` | `requestIdleCallback` | Nice-to-have widgets |
| `hydrateOnInteraction(events)` | On `'click'`, `['wheel','mouseover']`, … | Menus, accordions |
| `hydrateOnMediaQuery(q)` | The query matches | Desktop-only panels |
| `hydrateNever` | Never | Truly static server-rendered markup |

A custom strategy is a function receiving `hydrate` and `forEachElement`, returning a teardown.

The important half is **which components deserve it**: something above the fold that the user interacts with immediately should hydrate eagerly. Lazy-hydrating it trades a smaller initial cost for a stall at the exact moment of interaction — the worst place to put one.

[↑ Back to top](#table-of-contents)

---

## Cheat Sheet

### Compiler macros

| Macro | Since | Returns | Notes |
|---|---|---|---|
| `defineProps<T>()` | 3.0 | props object | Destructure stays reactive (3.5+); can't mix type + runtime forms |
| `withDefaults(defineProps<T>(), {…})` | 3.0 | props object | Superseded by 3.5 destructure defaults |
| `defineEmits<T>()` | 3.0 | `emit` fn | Tuple syntax 3.3+; declaring removes the event from `$attrs` |
| `defineModel<T>(name?, opts?)` | 3.4 | writable ref | `[model, modifiers]` form for modifiers; beware `default` desync |
| `defineExpose({…})` | 3.2 | — | Components are closed without it |
| `defineOptions({…})` | 3.3 | — | `inheritAttrs`, `name`; can't use local vars |
| `defineSlots<T>()` | 3.3 | slots object | Typing only — [Part 7](./vue-slots-injection-watchers.md) |

### Props and events

| Task | Code |
|---|---|
| Required prop | `defineProps<{ id: string }>()` |
| Optional with default | `const { size = 'md' } = defineProps<{ size?: Size }>()` |
| Runtime validator | `defineProps({ status: { type: String, validator: v => […].includes(v) } })` |
| Emit with payload | `defineEmits<{ change: [id: string] }>()` then `emit('change', id)` |
| Listen | `@change="onChange"` (kebab-case in templates) |
| Two-way | `defineModel<T>()` + `v-model` |
| Named two-way | `defineModel<T>('title')` + `v-model:title` |

### Component instance access

| Task | Code |
|---|---|
| DOM element ref | `const el = useTemplateRef('name')` + `ref="name"` |
| Component ref | same, typed `InstanceType<typeof Comp>` |
| Expose methods | `defineExpose({ play, seek })` |
| Read fallthrough attrs | `$attrs` in template, `useAttrs()` in script |
| Disable auto-fallthrough | `defineOptions({ inheritAttrs: false })` |

### Dynamic & async

| Task | Code |
|---|---|
| Switch component | `<component :is="Comp" />`, holding it in a `shallowRef` |
| Code-split | `defineAsyncComponent(() => import('./X.vue'))` |
| With loading/error UI | options form + `loadingComponent`/`errorComponent`/`delay`/`timeout` |
| Preserve state on switch | wrap in `<KeepAlive>` |
| SSR: hydrate on scroll | `hydrate: hydrateOnVisible()` |

### Gotcha index

| Symptom | Cause | Fix |
|---|---|---|
| Handler fires twice | Event not declared in `defineEmits` | Declare it |
| Warning: props are readonly | Assigning to a prop | Emit, or derive with `computed` |
| Prop change doesn't reach child state | `ref(props.x)` copies once | Use `computed`, or `watch` the prop |
| `player.value` is `null` | Touching a template ref during setup | Move into `onMounted` |
| Parent's ref exposes nothing | `<script setup>` is closed | `defineExpose({…})` |
| `class` lands on the wrong element | Automatic fallthrough | `inheritAttrs: false` + `v-bind="$attrs"` |
| "cannot reference locally declared variables" | Macro referencing a setup variable | Use a literal or an import |
| Loading spinner never goes away | `timeout` defaults to `Infinity` | Set `timeout` and an `errorComponent` |

**Next:** [Parts 7–9 — Slots, Provide/Inject, Watchers](./vue-slots-injection-watchers.md) · **Up:** [Overview](./vue-mastery-guide.md)

[↑ Back to top](#table-of-contents)
