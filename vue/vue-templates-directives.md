# Vue 3 — Templates, Directives, Events & Forms (Parts 3–4)

> **Spec:** this doc follows the shared spec in [`vue-mastery-guide.md` → About This Document](./vue-mastery-guide.md#about-this-document) — curiosity-driven, `<script setup lang="ts">` only, Vue **3.5.42**, verified **September 6, 2026**.
>
> **Assumes** [Parts 1–2 — Reactivity](./vue-reactivity.md). Templates are where reactive state becomes DOM; if tracking isn't in your head yet, half of this doc will look arbitrary.

A Vue template is not a string that gets evaluated at runtime. It is compiled — ahead of time, by `@vue/compiler-sfc` — into a render function, and the compiler uses what it can prove about your markup to make updates cheap ([Rendering Mechanism](https://vuejs.org/guide/extras/rendering-mechanism.html)). That is why templates are the default rather than JSX in Vue, and it is worth keeping in mind as you read: most directives exist so the compiler can *see* your intent.

---

## Table of Contents

**Part 3 — Template Syntax & Directives**
1. [Beginner: interpolation, `v-bind`, `v-on`](#1-beginner-interpolation-v-bind-v-on)
2. [Working Knowledge: `v-if` vs. `v-show`](#2-working-knowledge-v-if-vs-v-show)
3. [Working Knowledge: `v-for` and the rules of keys](#3-working-knowledge-v-for-and-the-rules-of-keys)
4. [Advanced: `v-for` + `v-if`, and why it's a mistake](#4-advanced-v-for--v-if-and-why-its-a-mistake)
5. [Advanced: class and style bindings](#5-advanced-class-and-style-bindings)
6. [Advanced: `v-html`, `v-text`, `v-pre`, `v-once`, `v-cloak`](#6-advanced-v-html-v-text-v-pre-v-once-v-cloak)
7. [Mastery: what the compiler is allowed to assume](#7-mastery-what-the-compiler-is-allowed-to-assume)

**Part 4 — Events & Forms**

8. [Beginner: `v-on` and handler shapes](#8-beginner-v-on-and-handler-shapes)
9. [Working Knowledge: event and key modifiers](#9-working-knowledge-event-and-key-modifiers)
10. [Working Knowledge: `v-model` on every input type](#10-working-knowledge-v-model-on-every-input-type)
11. [Advanced: `v-model` modifiers, IME, and when to drop to `:value` + `@input`](#11-advanced-v-model-modifiers-ime-and-when-to-drop-to-value--input)
12. [Advanced: a form that actually validates](#12-advanced-a-form-that-actually-validates)
13. [Cheat Sheet](#cheat-sheet)

*Part 4 gets Beginner → Advanced only. Its Mastery material — how `v-model` compiles on components, and `defineModel` — belongs with components and lives in [Part 5](./vue-components-script-setup.md).*

---

# Part 3 — Template Syntax & Directives

## 1. Beginner: interpolation, `v-bind`, `v-on`

Three things cover most of a template.

**Text interpolation** — `{{ }}` takes any single JavaScript *expression* (not a statement):

```vue
<template>
  <p>{{ message }}</p>
  <p>{{ message.toUpperCase() }}</p>
  <p>{{ ok ? 'yes' : 'no' }}</p>
  <!-- ✗ statements don't work: {{ if (ok) {…} }}, {{ const x = 1 }} -->
</template>
```

**`v-bind`** binds an attribute or DOM property to an expression. The shorthand `:` is what you'll write and read everywhere:

```vue
<img v-bind:src="imageUrl" :alt="caption" />
<button :disabled="isLoading">Save</button>

<!-- dynamic attribute name -->
<a :[attrName]="value">…</a>

<!-- same-name shorthand (3.4+) — :id="id" -->
<div :id />

<!-- spread an object of attributes -->
<div v-bind="{ id: 'x', class: 'y' }" />
```

Two behaviours worth knowing on day one: a boolean attribute like `disabled` is *removed* entirely when the value is falsy (`null`, `undefined`, `false`), and `:class`/`:style` get special merge treatment (§5).

**`v-on`** attaches an event listener; shorthand `@`:

```vue
<button @click="count++">inline</button>
<button @click="increment">method</button>
<button @click="increment($event)">method with the event</button>
```

Part 4 covers events properly.

**Try It.** All three at once:

```vue
<script setup lang="ts">
import { ref } from 'vue'
const busy = ref(false)
const label = ref('Save')
</script>

<template>
  <button :disabled="busy" @click="busy = !busy">
    {{ busy ? 'Saving…' : label }}
  </button>
</template>
```

Click it: the label changes and the button disables. Inspect the element — the `disabled` attribute *appears and disappears* rather than toggling `disabled="false"`.

[↑ Back to top](#table-of-contents)

---

## 2. Working Knowledge: `v-if` vs. `v-show`

Both hide things. They are not interchangeable.

| | `v-if` | `v-show` |
|---|---|---|
| Mechanism | Creates/destroys the element and its component | Toggles `style="display: none"` |
| Initial cost | Nothing rendered if false | Always rendered |
| Toggle cost | Mount/unmount, lifecycle hooks fire | One style write |
| Works on `<template>` | ✅ | ❌ |
| Supports `v-else` / `v-else-if` | ✅ | ❌ |

```vue
<template>
  <p v-if="status === 'loading'">Loading…</p>
  <p v-else-if="status === 'error'">Something broke.</p>
  <p v-else>Done.</p>

  <!-- group several elements without a wrapper div -->
  <template v-if="user">
    <h2>{{ user.name }}</h2>
    <p>{{ user.email }}</p>
  </template>

  <!-- toggled often, cheap to keep around -->
  <aside v-show="panelOpen">…</aside>
</template>
```

**Rule of thumb:** toggles rarely, or the content is expensive → `v-if`. Toggles often (a dropdown, a tab panel) → `v-show`.

**The gotcha `v-if` hides:** destroying a component destroys its state. A form behind `v-if="expanded"` loses everything the user typed when it collapses; behind `v-show` it doesn't. That's sometimes exactly what you want (a modal that should reset) and sometimes a bug report.

[↑ Back to top](#table-of-contents)

---

## 3. Working Knowledge: `v-for` and the rules of keys

```vue
<li v-for="item in items" :key="item.id">{{ item.title }}</li>
<li v-for="(item, index) in items" :key="item.id">{{ index }}: {{ item.title }}</li>

<!-- objects: value, key, index -->
<li v-for="(value, key, index) in myObject" :key="key">{{ key }}: {{ value }}</li>

<!-- a range, 1..10 -->
<span v-for="n in 10" :key="n">{{ n }}</span>

<!-- `of` works too, if you prefer it -->
<li v-for="item of items" :key="item.id">…</li>
```

### Keys

Always supply a `:key`. Without one, Vue uses an "in-place patch" strategy that **reuses DOM nodes by position**, which the docs warn "can cause issues with component state or form values" ([List Rendering](https://vuejs.org/guide/essentials/list.html)).

The key must be **stable, unique among siblings, and tied to the item's identity** — a database id, a UUID, a natural key. Not the array index, and not `Math.random()` (which destroys and recreates every node on every render).

**Why an index key corrupts state.** Say you render three rows, each with an `<input>` the user has typed into, and then delete the first row:

```
before (key = index)          after deleting row 0
  0 → Ada    [input: "aa"]      0 → Grace  [input: "aa"]  ← wrong text!
  1 → Grace  [input: "bb"]      1 → Linus  [input: "bb"]  ← wrong text!
  2 → Linus  [input: "cc"]      (node 2 removed)

before (key = item.id)        after deleting row 0
  a → Ada    [input: "aa"]      b → Grace  [input: "bb"]  ✓
  b → Grace  [input: "bb"]      c → Linus  [input: "cc"]  ✓
  c → Linus  [input: "cc"]      (node a removed)
```

With index keys, Vue sees "key 0 still exists, its text changed" and patches the *text* while keeping the DOM node — including the uncontrolled `<input>` value and any component state inside it. With identity keys, Vue sees "key `a` is gone" and removes that node.

```vue
<!-- ✗ wrong -->
<TodoRow v-for="(todo, i) in todos" :key="i" :todo="todo" />

<!-- ✓ right -->
<TodoRow v-for="todo in todos" :key="todo.id" :todo="todo" />
```

Index keys are *safe* only when the list is append-only, never reordered, never filtered, and the items hold no state. That is rarer than it sounds, and the day someone adds a sort control it silently becomes a bug — so just use identity keys.

### Arrays: mutate or replace, both work

Mutation methods (`push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`) are intercepted and tracked. Non-mutating methods return new arrays, so assign the result ([List Rendering](https://vuejs.org/guide/essentials/list.html)):

```ts
items.value.push(newItem)                            // ✓
items.value = items.value.filter(i => i.active)      // ✓
```

**Try It.** Reproduce the index-key bug in 30 seconds:

```vue
<script setup lang="ts">
import { ref } from 'vue'
const rows = ref([{ id: 'a', n: 'Ada' }, { id: 'b', n: 'Grace' }, { id: 'c', n: 'Linus' }])
</script>

<template>
  <div v-for="(r, i) in rows" :key="i">
    {{ r.n }} <input placeholder="type here" />
  </div>
  <button @click="rows.shift()">delete first</button>
</template>
```

Type a different word into each input, then click **delete first**. The typed text stays put while the names shift up — the inputs are now attached to the wrong rows. Change `:key="i"` to `:key="r.id"` and repeat: the first input's text is removed along with its row, and the rest stay correctly paired.

[↑ Back to top](#table-of-contents)

---

## 4. Advanced: `v-for` + `v-if`, and why it's a mistake

```vue
<!-- ✗ wrong — v-if has higher priority, so `todo` is not in scope yet -->
<li v-for="todo in todos" v-if="!todo.isComplete" :key="todo.id">
  {{ todo.name }}
</li>
```

Vue 3 gives `v-if` **higher priority** than `v-for` on the same element, so the condition is evaluated before the loop variable exists — you get a compile-time error about `todo` being undefined ([List Rendering](https://vuejs.org/guide/essentials/list.html)). (In Vue 2 the priority was reversed and this "worked", which is why old blog posts show it. Official docs are explicit that it is not the pattern to use.)

Two sanctioned fixes:

```vue
<!-- ✓ best — filter in a computed. The template stays declarative and the
     filter is cached (Part 1 §3) -->
<script setup lang="ts">
const activeTodos = computed(() => todos.value.filter(t => !t.isComplete))
</script>
<template>
  <li v-for="todo in activeTodos" :key="todo.id">{{ todo.name }}</li>
</template>

<!-- ✓ also fine — wrap, so the two directives are on different elements -->
<template v-for="todo in todos" :key="todo.id">
  <li v-if="!todo.isComplete">{{ todo.name }}</li>
</template>
```

Note where `:key` goes in the second form: on the `<template>` carrying `v-for`, not on the `<li>`.

For nested loops where a computed per level is awkward, a plain function is fine — it isn't cached, but the cost is a filter over a small inner array:

```vue
<li v-for="n in even(set.numbers)" :key="n">{{ n }}</li>
```

[↑ Back to top](#table-of-contents)

---

## 5. Advanced: class and style bindings

`:class` and `:style` are special-cased: Vue merges them with any static `class`/`style` on the same element, and accepts objects and arrays.

```vue
<!-- object: key = class name, value = truthiness -->
<div class="card" :class="{ active: isActive, 'is-error': hasError }" />
<!-- → class="card active" -->

<!-- array -->
<div :class="[baseClass, isActive ? 'active' : '']" />
<div :class="[{ active: isActive }, errorClass]" />

<!-- style: camelCase or kebab-case keys -->
<div :style="{ color: activeColor, fontSize: size + 'px' }" />
<div :style="[baseStyles, overrideStyles]" />

<!-- multiple values: the browser picks the last it supports -->
<div :style="{ display: ['-webkit-box', 'flex'] }" />
```

Two habits worth forming:

**Compute the class object when it grows.** More than two or three conditions in the template is unreadable and untestable:

```vue
<script setup lang="ts">
const classes = computed(() => ({
  'btn': true,
  'btn--primary': variant === 'primary',
  'btn--disabled': disabled || loading,
}))
</script>
<template><button :class="classes" /></template>
```

**Class bindings pass through to child components.** `<MyButton class="mt-4" />` lands on the child's root element via fallthrough attributes (Part 5) — which is why `:class` on a component usually just works, and why a multi-root component needs `:class="$attrs.class"` placed explicitly.

[↑ Back to top](#table-of-contents)

---

## 6. Advanced: `v-html`, `v-text`, `v-pre`, `v-once`, `v-cloak`

The rest of the built-in directives, in decreasing order of how often you'll need them.

**`v-html`** sets `innerHTML`. It **bypasses all escaping**:

```vue
<!-- ✗ dangerous — XSS if `comment` contains user-supplied HTML -->
<div v-html="comment" />

<!-- ✓ if you must render HTML, sanitise first, at the boundary -->
<div v-html="sanitize(comment)" />
```

Use it only for HTML you produced or have sanitised. Note also that `v-html` content is *not* compiled as a Vue template (no directives, no interpolation inside it) and is not affected by `scoped` styles without `:deep()` (Part 11).

**`v-text`** — `<span v-text="msg" />` equals `<span>{{ msg }}</span>`. Occasionally useful to avoid a flash of un-interpolated braces before hydration; `v-cloak` is the more targeted fix for that in no-build setups.

**`v-once`** renders once and never updates that subtree again. **`v-memo="[deps]"`** skips updates unless a dependency changed. Both are performance tools — see [Part 13](./vue-internals-performance.md), where the cost model makes sense.

**`v-pre`** skips compilation of an element and its children entirely — handy for displaying literal `{{ mustache }}` syntax in documentation.

[↑ Back to top](#table-of-contents)

---

## 7. Mastery: what the compiler is allowed to assume

The reason Vue templates are more constrained than JSX is that the constraints buy optimisations. The compiler analyses your template and marks up the render function it emits ([Rendering Mechanism](https://vuejs.org/guide/extras/rendering-mechanism.html)):

- **Static parts get cached.** Markup with no bindings is hoisted out of the render function and reused by reference. Enough consecutive static elements are condensed into a single static vnode holding a plain HTML string.
- **Dynamic parts get a patch flag.** A vnode created with a dynamic class only is emitted as `createElementVNode("div", { class: … }, null, 2 /* CLASS */)`; at patch time the runtime tests the flag bitwise and updates only the class.
- **Stable structures become blocks.** A template region whose structure can't change is a "block" that tracks only its dynamic descendants, in a flat array — so diffing walks a handful of nodes instead of a tree.

Which is why the directives are shaped the way they are: `v-if`/`v-for` are the *only* ways structure can change, so the compiler treats them as block boundaries and can assume everything else is stable. Writing the same logic with a render function and a ternary gives the runtime no such promise, and you fall back to a full diff.

[Part 12](./vue-internals-performance.md) covers this properly, including how to read the compiled output. The practical takeaway for template-writing today: **prefer directives to hand-rolled logic**, because the compiler can only optimise what it can see.

[↑ Back to top](#table-of-contents)

---

# Part 4 — Events & Forms

## 8. Beginner: `v-on` and handler shapes

```vue
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)

function onClick(event: MouseEvent) {
  console.log(event.currentTarget)
  count.value++
}
</script>

<template>
  <!-- inline handler: an expression, evaluated on each event -->
  <button @click="count++">inline</button>

  <!-- method handler: gets the native event as its argument -->
  <button @click="onClick">method</button>

  <!-- method with arguments — use $event to also get the event -->
  <button @click="add(5, $event)">add 5</button>

  <!-- multiple handlers, comma-separated -->
  <button @click="one(), two()">both</button>

  <!-- dynamic event name -->
  <button @[eventName]="onClick">dynamic</button>
</template>
```

Typing note: with `lang="ts"`, `event` in an inline handler is typed by Vue, and in a method handler you type the parameter yourself (`MouseEvent`, `KeyboardEvent`, `Event`). For `event.target.value` you need a cast, because the DOM types can't know which element fired:

```ts
function onInput(event: Event) {
  const value = (event.target as HTMLInputElement).value
}
```

[↑ Back to top](#table-of-contents)

---

## 9. Working Knowledge: event and key modifiers

Modifiers are suffixes that handle the DOM boilerplate you would otherwise write inside every handler.

```vue
<!-- event.preventDefault() -->
<form @submit.prevent="onSubmit">…</form>

<!-- event.stopPropagation() -->
<div @click.stop="…" />

<!-- only if event.target IS this element (not a descendant) -->
<div class="overlay" @click.self="close" />

<!-- capture phase; once; passive (never calls preventDefault) -->
<div @click.capture="…" @click.once="…" @scroll.passive="onScroll" />
```

```vue
<!-- key modifiers -->
<input @keyup.enter="submit" />
<input @keyup.esc="cancel" />
<input @keyup.page-down="next" />   <!-- kebab-case of KeyboardEvent.key -->

<!-- system modifiers -->
<div @click.ctrl="…" />         <!-- ctrl held (plus possibly others) -->
<div @click.ctrl.exact="…" />   <!-- ctrl and ONLY ctrl -->
<div @click.exact="…" />        <!-- no modifier keys at all -->

<!-- mouse buttons -->
<div @click.middle="…" @click.right.prevent="contextMenu" />
```

Two things worth internalising:

**`.prevent` on `<form @submit>` is not optional.** Without it the browser navigates away and your SPA reloads. This is the single most common "my form does nothing / the page flashes" bug.

**`.self` vs. `.stop` on overlays.** For a modal backdrop that should close only when the backdrop itself is clicked, `.self` is correct; `.stop` on the inner panel also works but scatters the logic:

```vue
<!-- ✓ intent is local and readable -->
<div class="backdrop" @click.self="close">
  <div class="panel"><!-- clicks in here don't close --></div>
</div>
```

**Order matters.** Modifiers are applied left to right, so `@click.prevent.self` prevents on *all* clicks and only calls the handler for self-clicks, while `@click.self.prevent` prevents only for self-clicks.

[↑ Back to top](#table-of-contents)

---

## 10. Working Knowledge: `v-model` on every input type

`v-model` is sugar for a value binding plus an event listener. On a text input:

```vue
<!-- these two are equivalent -->
<input :value="text" @input="event => text = (event.target as HTMLInputElement).value" />
<input v-model="text" />
```

The property/event pair it expands to depends on the element ([Form Input Bindings](https://vuejs.org/guide/essentials/forms.html)):

| Element | Property | Event |
|---|---|---|
| `<input type="text">`, `<textarea>` | `value` | `input` |
| `<input type="checkbox">`, `<input type="radio">` | `checked` | `change` |
| `<select>` | `value` | `change` |

```vue
<script setup lang="ts">
import { ref } from 'vue'
const text = ref('')
const agreed = ref(false)
const picked = ref('')
const checkedNames = ref<string[]>([])
const selected = ref('')
</script>

<template>
  <input v-model="text" placeholder="text" />
  <textarea v-model="text" />

  <!-- single checkbox → boolean -->
  <input type="checkbox" v-model="agreed" />

  <!-- multiple checkboxes bound to one array → array of `value`s -->
  <input type="checkbox" value="Jack" v-model="checkedNames" />
  <input type="checkbox" value="John" v-model="checkedNames" />

  <!-- radios → the checked value -->
  <input type="radio" value="One" v-model="picked" />
  <input type="radio" value="Two" v-model="picked" />

  <select v-model="selected">
    <option disabled value="">Please select one</option>
    <option>A</option>
    <option>B</option>
  </select>
</template>
```

**Non-string values.** `true-value`/`false-value` on a checkbox, and `:value` on radios and options, let you bind anything — including objects:

```vue
<input type="checkbox" v-model="toggle" true-value="yes" false-value="no" />
<input type="radio" v-model="pick" :value="firstOption" />
<option :value="{ number: 123 }">123</option>
```

Note that `true-value`/`false-value` only apply to `v-model`, not to the underlying `checked` attribute.

**Two documented traps:**

1. **`v-model` ignores initial `value`/`checked`/`selected` attributes.** The JavaScript state is the single source of truth — declare initial values there, not in the markup. `<input v-model="text" value="hello">` will not put "hello" in `text`.
2. **A `<select>` whose bound value matches no option renders unselected**, which behaves badly on iOS. Always provide a `disabled` placeholder option with an empty value, as in the snippet above.

**Try It.** Bind five inputs to one object and print it:

```vue
<script setup lang="ts">
import { reactive } from 'vue'
const form = reactive({ name: '', bio: '', agreed: false, plan: 'free', tags: [] as string[] })
</script>

<template>
  <input v-model="form.name" placeholder="name" />
  <textarea v-model="form.bio" />
  <label><input type="checkbox" v-model="form.agreed" /> agree</label>
  <select v-model="form.plan"><option>free</option><option>pro</option></select>
  <label><input type="checkbox" value="vue" v-model="form.tags" /> vue</label>
  <label><input type="checkbox" value="ts" v-model="form.tags" /> ts</label>
  <pre>{{ form }}</pre>
</template>
```

The `<pre>` updates live as you type and click. Note `form.tags` is an array of the checked `value`s — this is the one place a `reactive` object earns its keep over five separate refs.

[↑ Back to top](#table-of-contents)

---

## 11. Advanced: `v-model` modifiers, IME, and when to drop to `:value` + `@input`

Three built-in modifiers ([Form Input Bindings](https://vuejs.org/guide/essentials/forms.html)):

```vue
<input v-model.lazy="msg" />    <!-- sync on `change`, not `input` -->
<input v-model.number="age" />  <!-- cast to number -->
<input v-model.trim="msg" />    <!-- trim whitespace -->
```

`.number` is applied automatically when the input has `type="number"`, and if the cast fails (`parseFloat` returns `NaN`), the raw value is kept — so `.number` alone is not input validation.

### The IME gotcha

For languages requiring an **IME** (Chinese, Japanese, Korean), `v-model` deliberately does **not** update while the user is composing a character. The docs are explicit about this and about the escape hatch: if you need to respond to updates *during* composition, use `:value` and `@input` directly.

```vue
<!-- ✗ live search that "doesn't fire" for Japanese users -->
<input v-model="query" />

<!-- ✓ raw binding — fires on every keystroke, composition included -->
<input :value="query" @input="query = ($event.target as HTMLInputElement).value" />
```

This is a real, reportable bug class: a search-as-you-type box that appears broken only for CJK users. If your product has them, test with an IME on.

### When else to drop `v-model`

- You need the raw event (`inputType`, `isComposing`, selection range).
- The value needs asymmetric transformation (display formatted, store raw) — a writable `computed` (Part 1 §3) between the ref and `v-model` is the clean version of this.
- You're building a component's own `v-model` — that's `defineModel()`, [Part 5](./vue-components-script-setup.md).

```ts
// display a currency string, store cents
const amountCents = ref(0)
const amountDisplay = computed({
  get: () => (amountCents.value / 100).toFixed(2),
  set: v => { amountCents.value = Math.round(parseFloat(v) * 100) || 0 },
})
```

[↑ Back to top](#table-of-contents)

---

## 12. Advanced: a form that actually validates

Nothing new API-wise — this is Parts 1, 3 and 4 assembled, because forms are where beginners most often produce something that technically works and is unpleasant to use.

```vue
<script setup lang="ts">
import { ref, computed, reactive } from 'vue'

const form = reactive({ email: '', password: '' })
const touched = reactive({ email: false, password: false })
const submitting = ref(false)
const serverError = ref<string | null>(null)

const errors = computed(() => ({
  email: !form.email
    ? 'Email is required'
    : !/^\S+@\S+\.\S+$/.test(form.email) ? 'That does not look like an email' : null,
  password: form.password.length < 8 ? 'At least 8 characters' : null,
}))

const isValid = computed(() => Object.values(errors.value).every(e => e === null))

async function onSubmit() {
  touched.email = touched.password = true      // reveal all errors on submit
  if (!isValid.value) return
  submitting.value = true
  serverError.value = null
  try {
    await api.signIn({ ...form })
  } catch (e) {
    serverError.value = e instanceof Error ? e.message : 'Sign-in failed'
  } finally {
    submitting.value = false
  }
}
</script>

<template>
  <form @submit.prevent="onSubmit" novalidate>
    <label>
      Email
      <input v-model.trim="form.email" type="email" @blur="touched.email = true"
             :aria-invalid="!!(touched.email && errors.email)" />
    </label>
    <p v-if="touched.email && errors.email" role="alert">{{ errors.email }}</p>

    <label>
      Password
      <input v-model="form.password" type="password" @blur="touched.password = true" />
    </label>
    <p v-if="touched.password && errors.password" role="alert">{{ errors.password }}</p>

    <p v-if="serverError" role="alert">{{ serverError }}</p>
    <button type="submit" :disabled="submitting">
      {{ submitting ? 'Signing in…' : 'Sign in' }}
    </button>
  </form>
</template>
```

Four decisions in there worth naming:

1. **Validation is a `computed`, not a watcher.** It's derived state; deriving it is cheaper and cannot drift out of sync.
2. **`touched` gates *display*, not validity.** Showing "Email is required" before the user has typed anything is the classic beginner form.
3. **The submit button is disabled by `submitting`, not by `!isValid`.** Disabling on invalid hides *why* the form won't submit; letting the click through and revealing the errors is kinder and more accessible.
4. **`@submit.prevent` on the `<form>`, not `@click` on the button** — so Enter-to-submit works, which button click handlers alone break.

[↑ Back to top](#table-of-contents)

---

## Cheat Sheet

### Directives

| Directive | Shorthand | Does |
|---|---|---|
| `v-bind:attr` | `:attr` | Bind attribute/prop; `v-bind="obj"` spreads; `:id` is same-name shorthand (3.4+) |
| `v-on:evt` | `@evt` | Attach listener |
| `v-model` | — | Two-way binding on inputs and components |
| `v-if` / `v-else-if` / `v-else` | — | Conditionally create/destroy |
| `v-show` | — | Toggle `display` |
| `v-for` | — | Render a list; **always `:key`** |
| `v-slot` | `#name` | Slot content ([Part 7](./vue-slots-injection-watchers.md)) |
| `v-html` / `v-text` | — | Set `innerHTML` (XSS risk) / `textContent` |
| `v-once` / `v-memo` | — | Render once / skip unless deps change ([Part 13](./vue-internals-performance.md)) |
| `v-pre` / `v-cloak` | — | Skip compilation / hide until compiled |

### Event modifiers

| Modifier | Effect |
|---|---|
| `.stop` | `stopPropagation()` |
| `.prevent` | `preventDefault()` |
| `.self` | Only if `event.target` is this element |
| `.capture` | Capture phase |
| `.once` | Remove after first fire |
| `.passive` | Never calls `preventDefault` (scroll perf) |
| `.exact` | Exactly these system modifiers |
| `.enter` `.esc` `.tab` `.space` `.up` … | `KeyboardEvent.key`, kebab-cased |
| `.ctrl` `.alt` `.shift` `.meta` | System modifier held |
| `.left` `.right` `.middle` | Mouse button |

### `v-model` quick reference

| Target | Binds | Notes |
|---|---|---|
| text / textarea | `value` + `input` | `.lazy` switches to `change` |
| checkbox (single) | `checked` + `change` | Boolean, or `true-value`/`false-value` |
| checkbox (array) | `checked` + `change` | Collects `value`s into the array |
| radio | `checked` + `change` | `:value` for non-strings |
| select | `value` + `change` | Add a disabled empty placeholder option |
| component | `modelValue` + `update:modelValue` | `defineModel()` — [Part 5](./vue-components-script-setup.md) |

### Gotcha index

| Symptom | Cause | Fix |
|---|---|---|
| Page reloads on form submit | Missing `.prevent` | `@submit.prevent` |
| Wrong row's input text after delete | Index keys | `:key="item.id"` |
| Compile error: `todo` is not defined | `v-if` + `v-for` on one element | Filter in a `computed` |
| `[object Object]1` in output | Ref not unwrapped in a nested expression | [Part 1 §4](./vue-reactivity.md#4-advanced-unwrapping-rules-toref-torefs-tovalue) |
| Search box dead for CJK users | IME composition + `v-model` | `:value` + `@input` |
| Initial `value=""` attribute ignored | `v-model` state is the source of truth | Set the initial value in JS |
| Select renders blank on iOS | Bound value matches no option | Disabled placeholder option |

**Next:** [Parts 5–6 — Components & `<script setup>`](./vue-components-script-setup.md) · **Up:** [Overview](./vue-mastery-guide.md)

[↑ Back to top](#table-of-contents)
