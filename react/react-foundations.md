# React — Foundations (Parts 1–3)

> **Spec:** this doc follows the shared specification in [`react-mastery-guide.md`](./react-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Working Knowledge → Advanced → Mastery tiers, wrong-vs-right pairs, runnable **Try It** exercises, React 19 spine with `> **On React 18**` callouts. Set up the sandbox app from [Orientation & Setup](./react-mastery-guide.md#3-orientation--setup) before starting.

This doc covers **what React renders**: how a React app boots, what JSX actually is, and how React identifies items in a list. Nothing here changes anything on screen after the first paint — that's [Part 4](./react-state-events.md).

---

## Table of Contents

- [Part 1: Setup & Your First Render](#part-1-setup--your-first-render)
  - [Beginner — the smallest working app](#p1-beginner)
  - [Working Knowledge — components, props, and Strict Mode](#p1-working)
  - [Advanced — purity, and what "render" actually means](#p1-advanced)
  - [Part 1 cheat sheet](#part-1-cheat-sheet)
- [Part 2: JSX & Components](#part-2-jsx--components)
  - [Beginner — the syntax rules](#p2-beginner)
  - [Working Knowledge — props, children, composition, conditionals](#p2-working)
  - [Advanced — what JSX compiles to](#p2-advanced)
  - [Mastery — elements as values, and component identity](#p2-mastery)
  - [Part 2 cheat sheet](#part-2-cheat-sheet)
- [Part 3: Lists & Keys](#part-3-lists--keys)
  - [Beginner — rendering an array](#p3-beginner)
  - [Working Knowledge — the rules of keys](#p3-working)
  - [Advanced — the index-key bug, and keys as identity](#p3-advanced)
  - [Part 3 cheat sheet](#part-3-cheat-sheet)

---

## Part 1: Setup & Your First Render

<a id="p1-beginner"></a>

### Beginner — the smallest working app

A React app is one DOM node that React owns, plus a tree of your functions that describe what goes in it.

```html
<!-- index.html -->
<div id="root"></div>
```

```tsx
// src/main.tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';

function App() {
  return <h1>Hello</h1>;
}

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

That's the whole boot sequence. `createRoot` claims the DOM node; `.render()` hands React a description of what should be inside it.

> **On React 18** — identical. `createRoot` is the React 18 API too. What you must *not* see is `ReactDOM.render(<App />, container)`: that is the React 17 API, **removed in 19**, and its presence is a reliable sign a tutorial predates 2022.

Three rules that are true from this point on and never stop being true:

1. **A component is a function that returns JSX.** Nothing more.
2. **Component names start with a capital letter.** `<App />` renders your component; `<app />` renders a DOM tag called `app`. This is not a style convention — it's how the JSX transform tells them apart.
3. **A component returns exactly one root node.** Wrap siblings in a `<>…</>` fragment.

```tsx
// ❌ Wrong — two roots
function Header() {
  return (
    <h1>Title</h1>
    <p>Subtitle</p>
  );
}

// ✅ Right — one root (a fragment adds no DOM node)
function Header() {
  return (
    <>
      <h1>Title</h1>
      <p>Subtitle</p>
    </>
  );
}
```

**Try It.** Replace the contents of `src/main.tsx` with the snippet above and run `npm run dev`. Now lowercase the component to `app` and render `<app />`. Expected result: the heading disappears and the console warns `<app> is using incorrect casing`, because React treated it as an unknown HTML tag rather than your function.

<a id="p1-working"></a>

### Working Knowledge — components, props, and Strict Mode

Components take a single argument — the **props** object — and are composed by nesting.

```tsx
// src/App.tsx
type GreetingProps = { name: string; excited?: boolean };

function Greeting({ name, excited = false }: GreetingProps) {
  return <p>Hello, {name}{excited ? '!' : '.'}</p>;
}

export default function App() {
  return (
    <main>
      <Greeting name="Sam" excited />
      <Greeting name="Alex" />
    </main>
  );
}
```

Note `excited` with no value: a bare attribute is `true`. And note the default comes from **destructuring**, not a `defaultProps` assignment.

> **On React 18** — `Greeting.defaultProps = { excited: false }` still worked on function components. It is **removed in React 19**; destructuring defaults are the replacement. If you see `defaultProps` on a function component, it is dead code on 19 and React will warn.

#### Strict Mode

`<StrictMode>` is a development-only wrapper that makes React deliberately do things twice, to surface bugs that would otherwise appear intermittently in production:

- It **calls your component functions twice** per render. If your component is impure — mutating something outside itself, pushing to a module array, incrementing a counter — you will see doubled output.
- It **runs each Effect's setup and cleanup an extra time** on mount ([Part 6](./react-effects-refs.md)).

It has no effect in production builds. It is not a bug and you should not remove it — the double-invocation is the point.

**Try It.** Add an impurity and watch Strict Mode catch it:

```tsx
let renders = 0;

export default function App() {
  renders++;                       // 🚩 impure: mutating module state during render
  return <p>Rendered {renders} times</p>;
}
```

Expected result: the page shows **2**, not 1, on first load. Remove `<StrictMode>` from `main.tsx` and it shows 1. Put it back — the doubling is telling you this component would misbehave the moment React re-rendered it for any reason.

<a id="p1-advanced"></a>

### Advanced — purity, and what "render" actually means

The docs are unusually blunt here: **"Rendering must always be a pure calculation."** ([Render and Commit](https://react.dev/learn/render-and-commit)) In practice that means, during the body of your component function:

| Don't | Why |
|---|---|
| Mutate props, state, or objects created in earlier renders | React may reuse or discard renders; mutation makes the result depend on how many times it ran |
| Write to module-level or global variables | Same output must follow from same input |
| Read or write `ref.current` | See [Part 7](./react-effects-refs.md) — refs are for after commit |
| Perform I/O, start timers, subscribe | That is what Effects are for ([Part 6](./react-effects-refs.md)) |
| Call `Math.random()` or `Date.now()` for anything React sees | Two renders would disagree; also breaks SSR hydration |

Local mutation is fine — creating an array inside the function and pushing to it is pure, because nothing outside the render observes it.

```tsx
// ✅ Pure: the array is created during this render and never escaped
function Rows({ items }: { items: string[] }) {
  const cells = [];
  for (const item of items) cells.push(<td key={item}>{item}</td>);
  return <tr>{cells}</tr>;
}
```

The payoff for purity is the whole pipeline from [the mental model](./react-mastery-guide.md#4-the-one-paragraph-mental-model): React can call your component, throw the result away, and call it again — which is exactly what concurrent rendering does ([Part 11](./react-performance-concurrency.md)).

And the commit step is stingy on purpose: **"React only changes the DOM nodes if there's a difference between renders."** The docs illustrate this with a clock that re-renders every second around an `<input>` — the text you typed survives, because the input appears in the same place in the JSX each time, so React never touches it.

```
render #1                render #2                 committed DOM ops
┌──────────────┐         ┌──────────────┐
│ <h1>12:00</h1│         │ <h1>12:01</h1│   →      set textContent on the h1
│ <input/>     │         │ <input/>     │   →      (nothing — identical)
└──────────────┘         └──────────────┘
```

<a id="p1-mastery"></a>

#### Mastery — collapsed

*Part 1's Mastery material is the fiber tree and the commit phases, which only make sense once you have seen state, effects and reconciliation. It lives in [Part 13: Under the Hood](./react-internals-hooks.md).*

<a id="part-1-cheat-sheet"></a>

### Part 1 cheat sheet

| Thing | Form |
|---|---|
| Mount an app | `createRoot(el).render(<App />)` |
| Component | Capitalised function returning JSX |
| Multiple roots | Wrap in `<>…</>` |
| Prop default | `function C({ x = 1 })` — **not** `C.defaultProps` |
| Boolean prop shorthand | `<C excited />` means `excited={true}` |
| Development-only checks | `<StrictMode>` — double render, double effect setup |
| Forbidden in render | mutation, I/O, `ref.current`, `Math.random()` |

[↑ Back to top](#table-of-contents)

---

## Part 2: JSX & Components

<a id="p2-beginner"></a>

### Beginner — the syntax rules

JSX is HTML-shaped syntax that compiles to JavaScript function calls. The rules that differ from HTML:

```tsx
function Card() {
  return (
    <div className="card" style={{ padding: 8, borderRadius: 4 }}>
      {/* a comment, in braces */}
      <label htmlFor="name">Name</label>
      <input id="name" defaultValue="Sam" />
      <br />
    </div>
  );
}
```

| HTML | JSX | Why |
|---|---|---|
| `class="x"` | `className="x"` | `class` is a reserved word |
| `for="x"` | `htmlFor="x"` | `for` is a reserved word |
| `style="padding: 8px"` | `style={{ padding: 8 }}` | An object, camelCased; numbers get `px` |
| `<br>` | `<br />` | Every tag must close |
| `onclick="…"` | `onClick={fn}` | camelCase, and a function not a string |
| `tabindex` | `tabIndex` | camelCase attributes generally |

Anything in `{braces}` is a JavaScript expression — a value, not a statement. `{items.map(…)}` works; `{if (x) …}` does not.

<a id="p2-working"></a>

### Working Knowledge — props, children, composition, conditionals

#### Children

Whatever you nest inside a component arrives as the `children` prop. This is the single most useful compositional tool in React.

```tsx
function Panel({ title, children }: { title: string; children: React.ReactNode }) {
  return (
    <section className="panel">
      <h2>{title}</h2>
      {children}
    </section>
  );
}

// usage
<Panel title="Settings">
  <Toggle />
  <p>Anything at all goes here.</p>
</Panel>
```

`React.ReactNode` is the type for "anything React can render" — elements, strings, numbers, arrays, `null`, `undefined`, booleans.

#### Conditional rendering

Three forms, in ascending order of how often they bite you:

```tsx
// 1. Ternary — the default. Always safe.
{isLoggedIn ? <Dashboard /> : <Login />}

// 2. && — fine when the left side is a real boolean
{hasError && <ErrorBanner />}

// 3. Early return — best when the whole component differs
if (!user) return <Login />;
return <Dashboard user={user} />;
```

The `&&` form has a specific trap:

```tsx
// ❌ Wrong — renders a literal "0" when the list is empty
{items.length && <List items={items} />}

// ✅ Right — force a boolean
{items.length > 0 && <List items={items} />}
```

`0` is falsy, so `&&` returns `0` — and React renders the number `0`, unlike `false`/`null`/`undefined` which render nothing. This is one of the most common bugs in real React code.

**Try It.** Render `{0 && <p>hi</p>}`, `{false && <p>hi</p>}` and `{'' && <p>hi</p>}` side by side. Expected result: you see a lone `0` on the page from the first one, and nothing from the other two.

#### Document metadata (19+)

In React 19 you can render `<title>`, `<meta>` and `<link>` anywhere in your tree and React hoists them into `<head>`:

```tsx
function ArticlePage({ article }: { article: Article }) {
  return (
    <article>
      <title>{article.title}</title>
      <meta name="description" content={article.summary} />
      <h1>{article.title}</h1>
    </article>
  );
}
```

> **On React 18** — this does nothing useful; a `<title>` inside a `<div>` is just an inert element. Eighteen-era code reaches for `react-helmet` or an equivalent. If you are on 19, you can delete that dependency.

<a id="p2-advanced"></a>

### Advanced — what JSX compiles to

JSX is not special to React; it's a syntax the compiler rewrites. With the modern JSX transform, this:

```tsx
<Panel title="Settings">
  <Toggle on />
</Panel>
```

becomes roughly:

```js
import { jsx as _jsx } from 'react/jsx-runtime';

_jsx(Panel, {
  title: 'Settings',
  children: _jsx(Toggle, { on: true })
});
```

And the return value of `_jsx(...)` is **a plain object**, not a DOM node and not a rendered component:

```js
{
  $$typeof: Symbol(react.transitional.element),
  type: Panel,          // the function itself, or the string "div"
  key: null,
  props: { title: 'Settings', children: {…} }
}
```

Four consequences that explain a lot of React's behaviour:

1. **Creating an element is free and does nothing.** `<Expensive />` in your JSX does not call `Expensive` — it records that React *should* call it. This is why passing JSX as a prop (`<Layout sidebar={<Expensive />} />`) is a legitimate performance technique ([Part 10](./react-performance-concurrency.md)).
2. **`type` is the identity React compares.** Same `type` at the same position → React updates in place and keeps state. Different `type` → teardown and rebuild ([Part 4](./react-state-events.md)).
3. **`key` is lifted out of props.** It sits on the element object, not in `props` — which is why components genuinely cannot read their own key ([Part 3](#part-3-lists--keys)).
4. **Elements are immutable.** You cannot change one after creating it; you produce a new one.

> **Outdated convention to recognise:** `import React from 'react'` at the top of every file purely to make JSX work. That was required by the *old* JSX transform, which compiled to `React.createElement`. The modern transform imports from `react/jsx-runtime` automatically — Vite's `react-ts` template uses it, and you only need to import `React` if you reference the namespace directly (`React.ReactNode`, `React.useState`). Old code and old tutorials are full of the redundant import.

<a id="p2-mastery"></a>

### Mastery — elements as values, and component identity

Because elements are values, you can store them, pass them, and choose between them at runtime:

```tsx
const views: Record<string, React.ReactNode> = {
  list: <ListView />,
  grid: <GridView />,
};

function Content({ mode }: { mode: keyof typeof views }) {
  return <>{views[mode]}</>;
}
```

You can also make the *type* dynamic, as long as it's capitalised at the point of use:

```tsx
// ❌ Wrong — lowercase means "HTML tag named tag"
function Heading({ level, children }) {
  const tag = `h${level}`;
  return <tag>{children}</tag>;   // renders <tag>, not <h2>
}

// ✅ Right — capitalised variable
function Heading({ level, children }: { level: 1 | 2 | 3; children: React.ReactNode }) {
  const Tag = `h${level}` as 'h1' | 'h2' | 'h3';
  return <Tag>{children}</Tag>;
}
```

#### Never define a component inside another component

```tsx
// ❌ Wrong
function Parent() {
  function Child() { … }        // new function identity on every Parent render
  return <Child />;
}

// ✅ Right — top level
function Child() { … }
function Parent() { return <Child />; }
```

Because `type` holds the function itself, redefining `Child` each render gives React a *different* `type` every time. React concludes the old component was removed and a new one added, so it **unmounts the subtree and destroys its state** on every parent render. The docs put it plainly: ["This is why you should not nest component function definitions."](https://react.dev/learn/preserving-and-resetting-state)

**Try It.** Put a `<input />` inside a nested-definition `Child`, type into it, then trigger a re-render of `Parent` (any state change will do — borrow one from [Part 4](./react-state-events.md)). Expected result: the input clears every time. Move `Child` to the top level; the text now survives.

<a id="part-2-cheat-sheet"></a>

### Part 2 cheat sheet

| Task | JSX |
|---|---|
| CSS class | `className="x"` |
| Inline style | `style={{ marginTop: 8 }}` |
| Label association | `htmlFor="id"` |
| Comment | `{/* … */}` |
| Conditional (either/or) | `{cond ? <A /> : <B />}` |
| Conditional (maybe) | `{cond && <A />}` — ensure `cond` is boolean |
| Nothing | `null` (not `false` in a variable you later render as a number) |
| Slot content | `children` prop |
| Type of "renderable" | `React.ReactNode` |
| Type of a component | `React.ElementType` / `React.ComponentType<P>` |
| Dynamic tag | assign to a **capitalised** variable first |
| Page title (19+) | render `<title>` anywhere |

| Element internals | Value |
|---|---|
| What `<A x={1} />` evaluates to | `{ type: A, key: null, props: { x: 1 } }` |
| Cost of creating it | Essentially zero — `A` is not called |
| Can a component read its own key? | No — `key` is not in `props` |

[↑ Back to top](#table-of-contents)

---

## Part 3: Lists & Keys

<a id="p3-beginner"></a>

### Beginner — rendering an array

React renders arrays of elements directly. `map` is the whole technique.

```tsx
type Todo = { id: string; text: string };

function TodoList({ todos }: { todos: Todo[] }) {
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>{todo.text}</li>
      ))}
    </ul>
  );
}
```

Omit `key` and React warns: *"Each child in a list should have a unique key prop."* It still renders — which is why the warning gets ignored, and why [Advanced](#p3-advanced) below exists.

<a id="p3-working"></a>

### Working Knowledge — the rules of keys

Straight from [Rendering Lists](https://react.dev/learn/rendering-lists):

> - **Keys must be unique among siblings.** However, it's okay to use the same keys for JSX nodes in *different* arrays.
> - **Keys must not change** or that defeats their purpose! Don't generate them while rendering.

And:

> Note that **your components won't receive `key` as a prop. It's only used as a hint by React itself.** If your component needs an ID, you have to pass it as a separate prop: `<Profile key={id} userId={id} />`.

Where the key goes matters. It belongs on the outermost element **produced by the map callback**:

```tsx
// ❌ Wrong — key is inside the component, React never sees it
{todos.map(todo => <TodoItem todo={todo} />)}
function TodoItem({ todo }) { return <li key={todo.id}>{todo.text}</li>; }

// ✅ Right — key on the element in the array
{todos.map(todo => <TodoItem key={todo.id} todo={todo} />)}
```

If each item needs multiple sibling nodes, use the long-form Fragment, which is the only form that accepts a key:

```tsx
import { Fragment } from 'react';

{people.map(person => (
  <Fragment key={person.id}>
    <dt>{person.name}</dt>
    <dd>{person.role}</dd>
  </Fragment>
))}
```

`<>…</>` cannot take a key — that's the one thing the shorthand can't do.

<a id="p3-advanced"></a>

### Advanced — the index-key bug, and keys as identity

The docs name both anti-patterns explicitly:

> You might be tempted to use an item's index in the array as its key. In fact, that's what React will use if you don't specify a `key` at all. But the order in which you render items will change over time if an item is inserted, deleted, or if the array gets reordered. **Index as a key often leads to subtle and confusing bugs.**
>
> Similarly, **do not generate keys on the fly, e.g. with `key={Math.random()}`. This will cause keys to never match up between renders, leading to all your components and DOM being recreated every time.**

`Math.random()` is the more obviously wrong one — it silently destroys and recreates the entire list on every render, losing focus, scroll position and any state inside. The index key is more insidious, because it is *correct until the list reorders*.

Here is exactly how it breaks. Keys tell React which old item corresponds to which new item. With index keys, that mapping is positional:

```
Delete "Bread" from [Apples, Bread, Cherries]

  key=index                            key=id
  ─────────                            ──────
  before  0:Apples  1:Bread  2:Cherry  a:Apples  b:Bread  c:Cherry
  after   0:Apples  1:Cherry           a:Apples  c:Cherry

  React sees:                          React sees:
   0 stays, text 1 changes,             a stays untouched,
   2 removed                            b removed,
                                        c stays untouched
   → the DOM node that held             → the DOM node that held
     "Bread" is REUSED for                "Cherries" is reused for
     "Cherries", keeping any              "Cherries". Correct.
     state it had (checkbox,
     focus, edit-in-progress)
```

**Try It.** This is worth actually running — it's the clearest demonstration of what keys do.

```tsx
import { useState } from 'react';

const initial = ['Apples', 'Bread', 'Cherries'];

export default function App() {
  const [items, setItems] = useState(initial);
  return (
    <>
      <button onClick={() => setItems(items.slice(1))}>Remove first</button>
      <ul>
        {items.map((item, i) => (
          <li key={i}>                              {/* 🚩 index key */}
            <input type="checkbox" /> {item}
          </li>
        ))}
      </ul>
    </>
  );
}
```

Tick the checkbox next to **Cherries**, then click *Remove first*. Expected result: the list becomes `Bread, Cherries` but the tick is now next to **Bread** — the checkbox state stayed with position 2, not with the item. Change `key={i}` to `key={item}` and repeat: the tick correctly follows Cherries.

The checkbox is uncontrolled DOM state, so React never re-renders it away — exactly the class of bug that makes index keys "subtle and confusing" rather than obviously broken.

#### When an index key is actually fine

The docs' objection is about reordering. If **all** of these hold, an index key is defensible:

- the list is never reordered, filtered, or spliced (append-only or fully static), **and**
- items have no state, no refs, no uncontrolled inputs, and no focus to lose, **and**
- you genuinely have no stable id.

If you're generating ids because the data has none, generate them **when the item is created**, not during render:

```tsx
// ✅ Right — id assigned once, at creation time
setItems([...items, { id: crypto.randomUUID(), text }]);
```

#### Keys as a deliberate reset

The flip side of "same key → keep state" is "changed key → destroy state", and that is a feature. Changing a key is the documented way to reset a subtree:

```tsx
<Chat key={recipient.id} recipient={recipient} />
```

Switching recipients gives the `Chat` a new key, so React unmounts the old one and mounts a fresh one — clearing the half-typed draft rather than sending it to the wrong person. This technique gets its full treatment in [Part 4: preserving and resetting state](./react-state-events.md), where it replaces a whole family of buggy Effects.

<a id="p3-mastery"></a>

#### Mastery — collapsed

*Part 3's Mastery material is the reconciliation algorithm itself — how React diffs two children arrays, why it does a single pass rather than a full tree diff, and what a "bailout" is. It belongs with the fiber model in [Part 13: Under the Hood](./react-internals-hooks.md).*

<a id="part-3-cheat-sheet"></a>

### Part 3 cheat sheet

| Rule | Detail |
|---|---|
| Where the key goes | On the outermost element returned by the `map` callback |
| Uniqueness | Among siblings only — different arrays may reuse keys |
| Stability | Must not change between renders |
| Never | `key={Math.random()}` — recreates everything, every render |
| Avoid | `key={index}` unless the list is append-only *and* stateless |
| Good source | A database id, or an id generated at item-creation time |
| Multiple siblings per item | `<Fragment key={id}>` — the `<>` shorthand can't take a key |
| Reading your own key | Impossible; pass it again as a normal prop |
| Deliberate reset | Change the key |

| Symptom | Likely key problem |
|---|---|
| Checkbox / input state attaches to the wrong row after a delete | Index keys |
| Whole list flashes, loses focus, loses scroll on every render | `Math.random()` or otherwise unstable keys |
| Stale row content after a reorder | Index keys |
| Console: "Each child in a list should have a unique key" | Missing key |
| Console: "Encountered two children with the same key" | Duplicate ids in the data |

[↑ Back to top](#table-of-contents)

---

**Next:** [Part 4–5 — State & Events](./react-state-events.md) · **Up:** [Overview](./react-mastery-guide.md)

*React 19.2.7 · verified September 6, 2026*
