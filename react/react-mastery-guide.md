# React Mastery Guide

> 🔄 **Needs re-write** — predates the current `prompt.md` template. Will be regenerated/updated to match it; treat as possibly stale on structure/format until then.

## About This Document

- **Framing:** **Curiosity-driven** — no deadline, no shipping task. Structured as a *guided, hands-on tutorial*: a sequence of runnable examples and exercises that build technical fluency. Theory ("why is it designed this way") appears only at the exact points where it is needed to explain a mechanism — there is deliberately **no history/evolution section**.
- **Depth tiers requested:** **Beginner → Working Knowledge → Advanced → Mastery**, applied per Part. Tiers that would be filler for a given concept are collapsed rather than padded. The weight of this doc sits in Working Knowledge and Advanced; Part 13 is where Mastery concentrates.
- **Confirmed scope — core client-side React only:**
  1. Setup & Your First Render
  2. JSX & Components
  3. Lists & Conditional Rendering
  4. State & the Render Cycle
  5. Events & Forms
  6. Effects & Synchronisation
  7. Refs & the Escape Hatch
  8. Sharing State: Lifting, Context, Reducers
  9. Performance & Referential Identity
  10. Concurrent React
  11. Data Fetching & Async UI
  12. Custom Hooks
  13. Under the Hood
  14. Cheat Sheets · 15. Suggested Learning / Reference Order · 16. Quick Self-Check
- **Excluded by design:** Routing and data libraries (React Router, TanStack Query) — mentioned only as "where this leads" pointers. React Server Components, Server Actions, and streaming SSR — they need a framework and belong in a separate Next.js doc. Styling — see [`tailwindcss/tailwindcss-mastery-guide.md`](../tailwindcss/tailwindcss-mastery-guide.md). Build tooling internals — see [`vite/vite-mastery-guide.md`](../vite/vite-mastery-guide.md). React Native.
- **Legacy coverage:** Hooks-only teaching. Class components and lifecycle methods appear **once**, as a translation table in the cheat sheets ([Part 14](#14-cheat-sheets)), so you can read older code and older Stack Overflow answers without being misled by them.
- **Setup for examples:** **Vite 8 + React 19 + TypeScript** (`npm create vite@latest` → `react-compiler-ts` template). Types are kept minimal and idiomatic — enough to be realistic, never so much that the type noise obscures the React concept. For the type system itself see [`typescript/typescript-mastery-guide.md`](../typescript/typescript-mastery-guide.md).
- **React Compiler stance:** **Teach both, compiler-forward.** `memo` / `useMemo` / `useCallback` are taught properly, because you must read them in existing code and because referential identity is a real mechanism. [Part 9](#9-performance--referential-identity) then shows what the compiler auto-memoises and which cases still need you.
- **Assumed background:** You are comfortable with modern JavaScript — closures, referential equality, immutable updates, `map`/`filter`, destructuring, promises. If any of that is shaky, read [`javascript/javascript-for-react-mastery.md`](../javascript/javascript-for-react-mastery.md) first; this doc is its direct sequel and does not re-teach it.
- **Versions verified:** September 2026, against **React 19.2.8** ([React versions](https://react.dev/versions)), **React Compiler 1.0** ([announcement](https://react.dev/blog/2025/10/07/react-compiler-1)), **Vite 8.x**, **`eslint-plugin-react-hooks` v6**.
- **Sourcing standard:** [react.dev](https://react.dev) first, Stack Overflow for real-world gotchas, GitHub/other sites only as supplement. Where a popular convention contradicts the current docs, the docs win and the disagreement is stated explicitly.
- **If you ask me to update this doc later:** keep the numbered Parts with tier subheadings inside, keep the wrong-vs-right snippet pairs, keep the "Try It" exercises (this is the curiosity-driven doc's version of a "Real Scenario"), keep the TOC and back-to-top links, and prefer adding Parts or tiers over reverting to freeform prose. A collapsed tier is a deliberate choice, not a gap to fill.

---

## Table of Contents

1. [Setup & Your First Render](#1-setup--your-first-render)
2. [JSX & Components](#2-jsx--components)
3. [Lists & Conditional Rendering](#3-lists--conditional-rendering)
4. [State & the Render Cycle](#4-state--the-render-cycle)
5. [Events & Forms](#5-events--forms)
6. [Effects & Synchronisation](#6-effects--synchronisation)
7. [Refs & the Escape Hatch](#7-refs--the-escape-hatch)
8. [Sharing State: Lifting, Context, Reducers](#8-sharing-state-lifting-context-reducers)
9. [Performance & Referential Identity](#9-performance--referential-identity)
10. [Concurrent React](#10-concurrent-react)
11. [Data Fetching & Async UI](#11-data-fetching--async-ui)
12. [Custom Hooks](#12-custom-hooks)
13. [Under the Hood](#13-under-the-hood)
14. [Cheat Sheets](#14-cheat-sheets)
15. [Suggested Learning / Reference Order](#15-suggested-learning--reference-order)
16. [Quick Self-Check](#16-quick-self-check)

---

## 1. Setup & Your First Render

### Beginner — scaffold the sandbox you will use for every exercise

```bash
npm create vite@latest react-lab -- --template react-compiler-ts
cd react-lab
npm install
npm run dev
```

Vite requires **Node.js 20.19+ or 22.12+** ([Vite guide](https://vite.dev/guide/)). The `react-compiler-ts` template is new — it wires up the React Compiler for you. The plain `react-ts` template does not; [Part 9](#9-performance--referential-identity) shows the manual wiring if you scaffolded with that instead.

Three files matter:

```text
react-lab/
├── index.html          ← one <div id="root"></div>, one <script type="module">
├── src/
│   ├── main.tsx        ← createRoot(...).render(<App />)   ← React attaches here
│   └── App.tsx         ← your component tree starts here
└── vite.config.ts
```

`src/main.tsx`:

```tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App.tsx';

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

Replace `src/App.tsx` entirely with:

```tsx
export default function App() {
  return <h1>Hello, React</h1>;
}
```

**Why two packages?** `react` is the renderer-agnostic core — components, hooks, element creation. `react-dom` is the *host renderer* that knows how to turn the result into DOM nodes. React Native swaps `react-dom` for a native renderer and keeps `react` identical. That split is why `useState` imports from `react` but `createRoot` and `useFormStatus` import from `react-dom`.

> **Deprecation to know:** `ReactDOM.render()` was **removed** in React 19. Any tutorial still showing it predates React 18 (2022) and should be treated as stale in every other respect too — see [React 19 removals](https://react.dev/blog/2024/12/05/react-19#removed-deprecated-react-apis).

### Working Knowledge — what `createRoot().render()` actually does

```text
  <App />                    createRoot(container)
  (a plain JS object,   ──►  ┌──────────────────────────┐
   not DOM, not HTML)        │ 1. build fiber tree      │  "render phase"
                             │ 2. diff vs previous tree │   (pure, interruptible)
                             │ 3. produce effect list   │
                             ├──────────────────────────┤
                             │ 4. mutate real DOM       │  "commit phase"
                             │ 5. run layout effects    │   (synchronous, never
                             │ 6. run passive effects   │    interrupted)
                             └──────────────────────────┘
                                        ↓
                             <div id="root"> …real DOM… </div>
```

The single most useful thing to internalise on day one: **`<App />` is a value, not an action.** Writing it does not render anything. It creates a small object describing *what* to render. React decides *when*, and calls your function itself. Everything confusing about React later — stale closures, batching, double invocation — follows from your component being a function React calls, rather than code you run.

Prove it:

```tsx
function App() { return <h1>Hi</h1>; }

console.log(<App />);
// { $$typeof: Symbol(react.transitional.element), type: App, props: {}, key: null, ... }
```

`type` is the *function itself*, uncalled. React will call it later.

### Advanced — StrictMode double invocation, and why your console logs twice

`<StrictMode>` in development deliberately:

- calls your component function **twice** per render,
- runs `useState` initialisers and `useMemo` factories **twice**,
- **mounts, unmounts, and remounts** every component once, running effects → cleanup → effects.

This is not a bug, and you should not remove `StrictMode` to silence it. It is a conformance test: pure components and correctly-cleaned-up effects are invisible to double invocation. Anything that breaks under it has a latent bug that will surface under concurrent rendering, Fast Refresh, or `<Activity>` ([Strict Mode docs](https://react.dev/reference/react/StrictMode)).

```tsx
// ❌ Wrong — breaks under StrictMode, and it is the *code* that is wrong.
let idCounter = 0;
function Row() {
  const [id] = useState(() => ++idCounter);   // mutates module scope during render
  return <li>{id}</li>;                       // dev renders 2, 4, 6… prod renders 1, 2, 3
}

// ✅ Right — the initialiser is pure; calling it twice yields the same tree.
function Row({ id }: { id: number }) {
  return <li>{id}</li>;                       // identity comes from data, not a side effect
}
```

Double invocation only happens in development builds; `npm run build && npm run preview` renders once. Never "fix" a StrictMode symptom with a `useRef` guard like `if (didRun.current) return;` — that hides the bug in dev while shipping it to prod, and it is the most common bad advice in older Stack Overflow answers about ["useEffect running twice"](https://stackoverflow.com/questions/72238175/why-useeffect-running-twice-and-how-to-handle-it-well-in-react).

**Try It.** Add `console.log('render')` to `App`, load the page, and count the logs. Temporarily remove `<StrictMode>` from `main.tsx` and count again. Then put it back.

[⬆ back to top](#table-of-contents)

---

## 2. JSX & Components

### Beginner — a component is a function that returns JSX

```tsx
type GreetingProps = { name: string };

function Greeting({ name }: GreetingProps) {
  return <p>Hello, {name}</p>;
}

export default function App() {
  return (
    <main>
      <Greeting name="Ada" />
      <Greeting name="Grace" />
    </main>
  );
}
```

Rules that are actually enforced:

- **Capitalise component names.** `<greeting />` compiles to the string `"greeting"` — an unknown HTML tag. `<Greeting />` compiles to the variable `Greeting`. Lowercase means host element; uppercase means your component.
- **Return exactly one root node.** Use a fragment `<>…</>` when you don't want a wrapper element.
- **`className`, not `class`;** `htmlFor`, not `for`. Attributes are camelCase (`onClick`, `tabIndex`, `strokeWidth`) because JSX attributes become JavaScript object keys.

### Working Knowledge — props, children, and composition

Props are a read-only object. You never assign to them.

```tsx
// ❌ Wrong — props are frozen input, not local variables.
function Badge({ label }: { label: string }) {
  label = label.toUpperCase();   // mutating your input makes the component unpredictable
  return <span>{label}</span>;
}

// ✅ Right — derive a new value during render.
function Badge({ label }: { label: string }) {
  const upper = label.toUpperCase();
  return <span>{upper}</span>;
}
```

`children` is just a prop with special JSX syntax — which is what makes layout composition work:

```tsx
type CardProps = { title: string; children: React.ReactNode };

function Card({ title, children }: CardProps) {
  return (
    <section className="card">
      <h2>{title}</h2>
      {children}
    </section>
  );
}

// Everything between the tags becomes `children`:
<Card title="Profile">
  <Avatar />
  <p>Member since 2019</p>
</Card>
```

Any prop can hold JSX, not just `children` — this is the "slots" pattern, and it needs no library:

```tsx
type PanelProps = {
  header: React.ReactNode;
  footer?: React.ReactNode;
  children: React.ReactNode;
};

function Panel({ header, footer, children }: PanelProps) {
  return (
    <div>
      <header>{header}</header>
      {children}
      {footer && <footer>{footer}</footer>}
    </div>
  );
}

<Panel header={<h2>Inbox</h2>} footer={<Pagination page={2} />}>
  <MessageList />
</Panel>
```

**Prop spreading** forwards an unknown set of props, typically for wrapper components:

```tsx
type ButtonProps = React.ComponentProps<'button'> & { variant?: 'primary' | 'ghost' };

function Button({ variant = 'primary', ...rest }: ButtonProps) {
  return <button className={`btn btn-${variant}`} {...rest} />;
}
```

`React.ComponentProps<'button'>` gives you every valid `<button>` attribute — including `ref`, since React 19 treats `ref` as an ordinary prop. Use spreading for wrappers like this; avoid it for domain components, where it hides which props are actually used.

### Advanced — elements vs components vs instances

Three different things, routinely conflated:

```text
Component   function Greeting(props) { … }        a function you wrote
Element     <Greeting name="Ada" />               a plain object: { type: Greeting, props: {…} }
Instance    the fiber node React keeps for it     internal; holds state, effects, DOM ref
```

The consequence you will actually hit: **a component's identity is its position in the tree plus its `type`.** Change either and React unmounts the old instance and mounts a new one — destroying its state.

```tsx
// ❌ Wrong — defining a component inside another component.
function Parent() {
  const [text, setText] = useState('');
  function Child() {                       // NEW function object on every Parent render
    return <input value={text} onChange={e => setText(e.target.value)} />;
  }
  return <Child />;                        // type changed → unmount + remount → input loses focus
}

// ✅ Right — define components at module scope.
function Child({ text, onChange }: { text: string; onChange: (v: string) => void }) {
  return <input value={text} onChange={e => onChange(e.target.value)} />;
}
function Parent() {
  const [text, setText] = useState('');
  return <Child text={text} onChange={setText} />;
}
```

**Try It.** Paste the wrong version into `App.tsx` and type into the input. It accepts exactly one character before losing focus — because after each keystroke `Child` is a brand-new function, so React tears down the DOM node and builds a fresh one. This is the highest-value gotcha in Part 2; [Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state) explains the general principle.

### Advanced — purity is a contract, not a style preference

A component must be a pure function of its props, state, and context: same inputs → same JSX, no side effects during render.

```tsx
// ❌ Wrong — side effects during render.
let renderCount = 0;
function Counter() {
  renderCount++;                              // mutation of outer scope
  document.title = `Count ${renderCount}`;    // touching the DOM during render
  return <p>{renderCount}</p>;
}

// ✅ Right — render computes, effects act.
function Counter({ count }: { count: number }) {
  useEffect(() => { document.title = `Count ${count}`; }, [count]);
  return <p>{count}</p>;
}
```

Mutating something you created *during this render* is fine — that is "local mutation" and is invisible to everyone else:

```tsx
function List({ items }: { items: string[] }) {
  const rows = [];                       // created in this render → safe to mutate
  for (const item of items) rows.push(<li key={item}>{item}</li>);
  return <ul>{rows}</ul>;
}
```

Purity is what buys React the right to call your component twice, abandon a half-finished render, or re-run it at a different priority ([Keeping Components Pure](https://react.dev/learn/keeping-components-pure)).

### Mastery — what JSX compiles to now

Since React 17 the "automatic runtime" is the default: you do **not** need `import React from 'react'` in files that only use JSX. Vite's React plugin configures this for you.

```tsx
// You write:
<Greeting name="Ada" />

// The compiler emits (automatic runtime):
import { jsx as _jsx } from 'react/jsx-runtime';
_jsx(Greeting, { name: 'Ada' });
```

Two practical consequences:

1. `import React from 'react'` at the top of every file is a *legacy* convention. Harmless, but a tutorial insisting on it predates 2020 ([Introducing the New JSX Transform](https://legacy.reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html)).
2. `key` and `ref` are **not** in `props`. The compiler pulls them out — they are instructions to React, not data for your component. React 19 moved `ref` back into props for function components you write, but `key` stays reserved.

[⬆ back to top](#table-of-contents)

---

## 3. Lists & Conditional Rendering

### Beginner — `map` produces elements, `key` identifies them

```tsx
const users = [
  { id: 'u1', name: 'Ada' },
  { id: 'u2', name: 'Grace' },
];

function UserList() {
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

Conditionals use expressions, because a JSX slot holds a value:

```tsx
{isLoggedIn ? <Dashboard /> : <LoginForm />}   // either/or
{error && <p role="alert">{error}</p>}         // render-or-nothing
{items.length === 0 && <Empty />}
```

### Working Knowledge — the `&&` numeric trap

```tsx
// ❌ Wrong — when items.length is 0, React renders the number 0 on the page.
{items.length && <ItemTable items={items} />}

// ✅ Right — force a boolean, or use a ternary.
{items.length > 0 && <ItemTable items={items} />}
{items.length ? <ItemTable items={items} /> : null}
```

React skips `null`, `undefined`, `false`, and `true` when rendering, but renders `0` and `NaN` because they are legitimate values ([conditional rendering docs](https://react.dev/learn/conditional-rendering#logical-and-operator-)). This reaches production constantly: an empty cart renders a stray `0` above the "your cart is empty" message.

### Advanced — `key` is identity, and the index is usually the wrong identity

`key` tells React which element in the *new* list corresponds to which element in the *old* list. Get it wrong and React reuses the wrong instance — keeping the DOM node, its state, its scroll position, and its focus, while swapping the data underneath.

```text
Keyed by stable id — item removed from the middle:

  before:  [a:"Alpha"]  [b:"Beta"]  [c:"Gamma"]
  after:   [a:"Alpha"]              [c:"Gamma"]
  React:    reuse a      DESTROY b   reuse c          ← correct

Keyed by index — same removal:

  before:  [0:"Alpha"]  [1:"Beta"]  [2:"Gamma"]
  after:   [0:"Alpha"]  [1:"Gamma"]
  React:    reuse 0      reuse 1, change text only    DESTROY 2
                             ↑
             Beta's DOM node, its checkbox state, and its
             focus are now attached to Gamma's data.
```

```tsx
// ❌ Wrong — index keys with a reorderable/removable list.
{todos.map((todo, i) => <TodoRow key={i} todo={todo} />)}

// ✅ Right — a key that belongs to the data.
{todos.map(todo => <TodoRow key={todo.id} todo={todo} />)}
```

Index keys are acceptable only when the list is **static, never reordered, never filtered, and items hold no internal state**. Since you rarely know that will stay true, default to a data key. If your data has no id, generate one *when the item is created* (`crypto.randomUUID()`), never during render.

**Try It.** Build this, type different text into each input, then delete the middle row:

```tsx
function KeyDemo() {
  const [rows, setRows] = useState([
    { id: 'a', label: 'Alpha' },
    { id: 'b', label: 'Beta' },
    { id: 'c', label: 'Gamma' },
  ]);
  return (
    <ul>
      {rows.map((row, i) => (
        <li key={i}>                            {/* ← swap to key={row.id} after testing */}
          <input defaultValue={row.label} />    {/* uncontrolled: the DOM holds the value */}
          <button onClick={() => setRows(rs => rs.filter(r => r.id !== row.id))}>×</button>
        </li>
      ))}
    </ul>
  );
}
```

With `key={i}` the surviving inputs keep the *wrong* text. With `key={row.id}` they behave correctly. That is the whole argument, demonstrated in ten seconds.

### Advanced — using `key` deliberately, to reset state

Because a changed `key` forces unmount + remount, `key` is also the idiomatic way to *intentionally* discard state:

```tsx
// A form that must fully reset when you switch to a different user.
<ProfileForm key={selectedUserId} user={selectedUser} />
```

Without the `key`, `ProfileForm`'s internal draft state persists across user switches — you would edit Bob's profile with Alice's half-typed changes still in the fields. This is the documented alternative to syncing props into state with an effect ([Resetting state with a key](https://react.dev/learn/preserving-and-resetting-state#resetting-state-at-the-same-position)).

### Mastery — fragments need keys too

When each list item renders multiple siblings, you need the long-form fragment, because `<>` cannot take a key:

```tsx
// ❌ Wrong — shorthand fragments accept no key (syntax error).
{rows.map(r => <key={r.id}>…</>)}

// ✅ Right
import { Fragment } from 'react';

{rows.map(r => (
  <Fragment key={r.id}>
    <dt>{r.term}</dt>
    <dd>{r.definition}</dd>
  </Fragment>
))}
```

This matters most inside CSS Grid and `<dl>` / `<table>` structures, where a wrapper `<div>` would break the layout or the HTML validity.

[⬆ back to top](#table-of-contents)

---

## 4. State & the Render Cycle

### Beginner — `useState` gives a component memory

```tsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Clicked {count} times</button>;
}
```

`useState` returns a pair: the current value for *this* render, and a setter that asks React to re-render with a new value. Calling the setter does not change `count` in the current scope — it schedules a new render in which `useState` returns the new value.

Two rules with teeth:

- **Call hooks unconditionally, at the top level** of a component or another hook. Never inside `if`, loops, or callbacks. React matches hooks to their stored state *by call order* — [Part 13](#13-under-the-hood) shows the linked list that makes this true.
- **Never mutate state.** Replace it. `Object.is(prev, next)` is how React decides whether anything changed.

### Working Knowledge — state is a snapshot

This is the mechanism that explains most "why is my state one step behind" questions.

```tsx
// ❌ Wrong — expects 3, gets 1.
<button onClick={() => {
  setCount(count + 1);   // count is 0 in this closure → schedules 1
  setCount(count + 1);   // count is STILL 0 → schedules 1
  setCount(count + 1);   // count is STILL 0 → schedules 1
}}>+3</button>

// ✅ Right — updater functions queue against each other.
<button onClick={() => {
  setCount(c => c + 1);  // 0 → 1
  setCount(c => c + 1);  // 1 → 2
  setCount(c => c + 1);  // 2 → 3
}}>+3</button>
```

```text
Render #1: count = 0
  handler closes over count = 0  ──┐
                                   │  setCount(count + 1) → queue: [replace with 1]
  clicking runs the handler ───────┤  setCount(count + 1) → queue: [replace 1, replace 1]
                                   │  setCount(count + 1) → queue: [replace 1, ×1, ×1]
                                   ▼
Render #2: count = 1        ← the queue's last "replace" wins

With updaters, the queue holds *functions*, each fed the previous result:
  [c=>c+1, c=>c+1, c=>c+1]  applied to 0  →  1 → 2 → 3
```

The variable `count` is a **constant within one render**. It never changes mid-render — a property that makes rendering predictable, and that is why `setCount(c => c + 1)` exists at all ([State as a Snapshot](https://react.dev/learn/state-as-a-snapshot)).

**Rule of thumb:** if the next state depends on the previous state, use the updater form. Otherwise either is fine.

### Working Knowledge — batching

React batches all `setState` calls made during the same event into a single re-render. Since React 18 this includes calls in promises, `setTimeout`, and native event handlers ("automatic batching") — previously only React event handlers batched, which is why older articles claim `setTimeout` causes extra renders ([React 18 automatic batching](https://react.dev/blog/2022/03/29/react-v18#new-feature-automatic-batching)).

```tsx
function Demo() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);
  console.log('render', a, b);

  return (
    <button onClick={async () => {
      setA(x => x + 1);
      setB(x => x + 1);          // ONE render, not two

      await fetch('/api/ping');
      setA(x => x + 1);
      setB(x => x + 1);          // also ONE render (React 18+; React 17 gave two)
    }}>go</button>
  );
}
```

To opt out for a specific update — rare, mostly for measuring the DOM between updates — use `flushSync` from `react-dom` ([reference](https://react.dev/reference/react-dom/flushSync)).

### Working Knowledge — structuring state objects and arrays

Updates must produce new references down to the level you changed:

```tsx
type Profile = { name: string; address: { city: string; zip: string } };

// ❌ Wrong — same object reference; React sees no change and does not re-render.
setProfile(p => { p.address.city = 'Berlin'; return p; });

// ✅ Right — new outer object AND new nested object.
setProfile(p => ({ ...p, address: { ...p.address, city: 'Berlin' } }));
```

Arrays follow the same rule — use the non-mutating methods:

```tsx
setItems(xs => [...xs, next]);                            // add
setItems(xs => xs.filter(x => x.id !== id));              // remove
setItems(xs => xs.map(x => x.id === id ? { ...x, done: true } : x));  // update one
setItems(xs => xs.toSorted((a, b) => a.n - b.n));         // sort (copies; ES2023)
```

`sort`, `reverse`, `splice`, `push`, and `pop` mutate. `toSorted`, `toReversed`, `toSpliced`, and `with` copy — they are available in every browser Vite targets and in Node 20+.

When nesting gets deep enough that spread pyramids hurt, that is a signal to **flatten the state shape**, not to reach for a library. If you still want one, [Immer](https://immerjs.github.io/immer/) via `useImmerReducer` is the conventional answer; React's docs endorse it for deeply nested state ([Updating Objects in State](https://react.dev/learn/updating-objects-in-state#write-concise-update-logic-with-immer)).

### Advanced — don't put derived values in state

The most common structural mistake in React code:

```tsx
// ❌ Wrong — two sources of truth that must be manually kept in sync.
const [items, setItems] = useState<Item[]>([]);
const [total, setTotal] = useState(0);
useEffect(() => { setTotal(items.reduce((s, i) => s + i.price, 0)); }, [items]);

// ✅ Right — derive during render. Always correct, one render pass, no effect.
const [items, setItems] = useState<Item[]>([]);
const total = items.reduce((s, i) => s + i.price, 0);
```

The wrong version renders twice per change (once with the stale total), and drifts permanently the first time someone updates `items` down a code path that skips the effect. React's own guidance is explicit: ["You Might Not Need an Effect"](https://react.dev/learn/you-might-not-need-an-effect#updating-state-based-on-props-or-state).

Ask three questions before adding a `useState`:

1. Can I compute it from existing props/state during render? → derive it.
2. Does it stay the same across renders and never need to trigger one? → `useRef` ([Part 7](#7-refs--the-escape-hatch)).
3. Do several components need it? → lift it, or reach for context ([Part 8](#8-sharing-state-lifting-context-reducers)).

### Advanced — lazy initialisers

`useState(expensiveInit())` calls `expensiveInit` on **every render** and throws the result away after the first. Pass the function instead:

```tsx
// ❌ Wrong — parses localStorage on every keystroke elsewhere in the component.
const [draft, setDraft] = useState(JSON.parse(localStorage.getItem('draft') ?? 'null'));

// ✅ Right — React calls this only on the initial mount.
const [draft, setDraft] = useState(() => JSON.parse(localStorage.getItem('draft') ?? 'null'));
```

**Try It.** Put `console.log('init')` inside both forms and click a sibling button that changes unrelated state. The first logs on every render; the second logs once (twice under StrictMode — which is the point of Part 1's warning).

### Mastery — where state actually lives

State is stored on the **fiber node**, not in your function and not keyed by variable name. Two consequences that surprise people:

- **Same component, different positions = different state.** `<Counter />` rendered twice holds two independent counts.
- **Same position, different `type` = state destroyed.** Rendering `<Counter />` where `<Timer />` used to be discards everything.

```tsx
// State survives: same position, same type, only props differ.
{isFancy ? <Counter isFancy={true} /> : <Counter isFancy={false} />}

// State is destroyed: React sees different types at that position.
{isFancy ? <FancyCounter /> : <PlainCounter />}
```

This is also why the "component defined inside a component" bug in [Part 2](#2-jsx--components) is fatal rather than merely wasteful: a new function identity each render means a new `type` each render.

[⬆ back to top](#table-of-contents)

---

## 5. Events & Forms

### Beginner — handlers are props holding functions

```tsx
// ❌ Wrong — calls handleClick during render and passes its return value (undefined).
<button onClick={handleClick()}>Save</button>

// ✅ Right — pass the function itself.
<button onClick={handleClick}>Save</button>

// ✅ Also right — an inline arrow, when you need to pass arguments.
<button onClick={() => handleDelete(id)}>Delete</button>
```

React attaches one listener at the root and dispatches through a **synthetic event** — a normalised wrapper over the native event with a consistent API across browsers. `e.preventDefault()`, `e.stopPropagation()`, and `e.target` all work as expected; `e.nativeEvent` gets you the underlying event when you need it.

> **Outdated advice alert:** older articles tell you to call `e.persist()` before using an event asynchronously. React 17 removed event pooling; `e.persist()` is now a no-op ([React 17 changes](https://legacy.reactjs.org/blog/2020/08/10/react-v17-rc.html#no-event-pooling)).

### Working Knowledge — controlled inputs

A controlled input's value comes from state; every keystroke round-trips through React.

```tsx
function NameField() {
  const [name, setName] = useState('');
  return (
    <input
      value={name}
      onChange={e => setName(e.target.value)}
      placeholder="Your name"
    />
  );
}
```

The classic mistakes, both of which log a console warning:

```tsx
// ❌ Wrong — value with no onChange: React makes the field read-only.
<input value={name} />

// ❌ Wrong — undefined initial value flips the input from uncontrolled to controlled
//    mid-life ("A component is changing an uncontrolled input to be controlled").
const [name, setName] = useState<string | undefined>(undefined);

// ✅ Right — always initialise to '' (never null/undefined) for controlled inputs.
const [name, setName] = useState('');
```

Uncontrolled inputs (`defaultValue`, read via a ref or `FormData`) are perfectly legitimate — the [React docs](https://react.dev/reference/react-dom/components/input#controlling-an-input-with-a-state-variable) recommend controlled only when you need to react to every keystroke (live validation, filtering, formatting). For a plain submit-and-forget form, uncontrolled is less code and fewer renders.

### Working Knowledge — one handler for many fields

```tsx
type Form = { email: string; password: string };

function SignUp() {
  const [form, setForm] = useState<Form>({ email: '', password: '' });

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    const { name, value } = e.target;
    setForm(f => ({ ...f, [name]: value }));   // computed key + spread
  }

  return (
    <>
      <input name="email" value={form.email} onChange={handleChange} />
      <input name="password" type="password" value={form.password} onChange={handleChange} />
    </>
  );
}
```

### Advanced — form Actions (`<form action>` + `useActionState`), React 19

React 19 made this the recommended shape for form submission. Passing a **function** to `action` means React handles the `preventDefault`, resets the form on success, and wraps your async work in a transition so pending state is automatic ([React 19 release](https://react.dev/blog/2024/12/05/react-19#actions)).

```tsx
import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';

type State = { error: string | null };

function SubmitButton() {
  const { pending } = useFormStatus();          // reads the ENCLOSING <form>'s status
  return <button type="submit" disabled={pending}>{pending ? 'Saving…' : 'Save'}</button>;
}

export default function NameForm() {
  const [state, formAction, isPending] = useActionState<State, FormData>(
    async (_prev, formData) => {
      const name = formData.get('name') as string;
      if (!name.trim()) return { error: 'Name is required' };
      await fetch('/api/name', { method: 'POST', body: formData });
      return { error: null };
    },
    { error: null },
  );

  return (
    <form action={formAction}>
      <input name="name" defaultValue="" />
      <SubmitButton />
      {state.error && <p role="alert">{state.error}</p>}
      {isPending && <span aria-live="polite">working…</span>}
    </form>
  );
}
```

Two details worth remembering:

- `useFormStatus` **must be in a child component**. Called inside the component that renders the `<form>`, it returns `pending: false` forever, because it reads the status of the form *above* it. This is the most-reported gotcha with the hook.
- `useActionState` was called `useFormState` and lived in `react-dom` during the React 19 canary period. It is now `useActionState` from `react`. Code samples using the old name are pre-19.0.

Compare with the manual equivalent you would have written before:

```tsx
// The pre-19 shape — still valid, just more moving parts you maintain yourself.
const [isPending, setIsPending] = useState(false);
const [error, setError] = useState<string | null>(null);

async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
  e.preventDefault();
  setIsPending(true);
  setError(null);
  try { await save(new FormData(e.currentTarget)); }
  catch (err) { setError(String(err)); }
  finally { setIsPending(false); }
}
```

### Advanced — `useOptimistic` for instant feedback

Show the result before the server confirms it; React reverts automatically if the action throws or when the real state arrives.

```tsx
import { useOptimistic } from 'react';

function Likes({ count, onLike }: { count: number; onLike: () => Promise<void> }) {
  const [optimisticCount, addOptimistic] = useOptimistic(count, (c, delta: number) => c + delta);

  return (
    <form action={async () => { addOptimistic(1); await onLike(); }}>
      <button type="submit">👍 {optimisticCount}</button>
    </form>
  );
}
```

`useOptimistic` only holds its optimistic value **during a transition/action**. Once the action settles, the value snaps back to the real prop — which is exactly what you want, and also why calling `addOptimistic` outside an action appears to do nothing.

**Try It.** Give `onLike` a `await new Promise(r => setTimeout(r, 1500))` and click rapidly. The count increments immediately per click, then reconciles to the server value.

### Mastery — event ordering and delegation

React 17+ attaches its listeners to the **root container** you passed to `createRoot`, not to `document`. This matters when React coexists with other code:

```text
click on <button> inside React root
        │
        ├─ native capture listeners on document ─────► fire first
        ├─ native listeners on the button itself ────► fire
        ├─ bubbles up to the React root container
        │        └─ React dispatches synthetic events down/up your component tree
        └─ continues bubbling to document ───────────► your document listeners fire LAST
```

The practical consequence: `e.stopPropagation()` inside a React handler does **not** stop a listener attached directly to `document` — by the time React runs your handler, the native event has already reached the root, and a document-level listener will still fire on the way up. This is the root cause of the classic "my click-outside-to-close dropdown closes immediately when I click the button that opens it" bug. The fix is to listen in the capture phase, or to check `containerRef.current?.contains(e.target as Node)` before closing.

[⬆ back to top](#table-of-contents)

---

## 6. Effects & Synchronisation

### Beginner — what an Effect is for

An Effect synchronises your component with something **outside** React: a subscription, a browser API, a chart library, a WebSocket, the document title. It runs *after* the commit, when the DOM already matches your JSX.

```tsx
useEffect(() => {
  // setup — runs after every commit where a dependency changed
  const id = setInterval(() => console.log('tick'), 1000);
  return () => clearInterval(id);   // cleanup — runs before the next setup, and on unmount
}, []);                             // dependencies
```

The dependency array is not a "run when" trigger list. It is a **correctness declaration**: "this Effect reads exactly these reactive values." Getting it wrong causes stale data, not a missing feature.

### Working Knowledge — the three dependency shapes

```tsx
useEffect(() => { … });              // after EVERY render — almost always a mistake
useEffect(() => { … }, []);          // once on mount, cleanup on unmount
useEffect(() => { … }, [roomId]);    // on mount, and whenever roomId changes
```

Think in terms of **setup/cleanup pairs**, not mount/unmount. With `[roomId]`, changing rooms runs: cleanup(old room) → setup(new room). Writing it that way makes StrictMode's extra mount a non-event.

```tsx
// ✅ The canonical shape — everything the Effect creates, it destroys.
useEffect(() => {
  const conn = createConnection(serverUrl, roomId);
  conn.connect();
  return () => conn.disconnect();
}, [serverUrl, roomId]);
```

### Working Knowledge — you probably don't need this Effect

Most Effects in real codebases should not exist. The four cases, straight from ["You Might Not Need an Effect"](https://react.dev/learn/you-might-not-need-an-effect):

| You are doing this in an Effect | Do this instead |
|---|---|
| Computing a value from props/state | Calculate it during render |
| Resetting state when a prop changes | Pass a `key` to the component |
| Adjusting some state when a prop changes | Adjust during render, or restructure state |
| Running code in response to a user action | Put it in the event handler |

```tsx
// ❌ Wrong — an Effect chasing a user action. Fires on every render where cart changes,
//    including when the cart is loaded from storage on mount.
useEffect(() => {
  if (items.length > 0) showToast('Added to cart');
}, [items]);

// ✅ Right — the toast belongs to the click, not to the state.
function handleAdd(item: Item) {
  setItems(xs => [...xs, item]);
  showToast('Added to cart');
}
```

The distinction: **Effects are for synchronising with external systems; event handlers are for reacting to what the user did.** If you cannot name the external system, you probably want a handler.

### Advanced — the dependency lint rule is right and you should not silence it

```tsx
// ❌ Wrong — the classic "just make it run once" hack.
useEffect(() => {
  fetchUser(userId).then(setUser);
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, []);                       // stale: switching users never refetches

// ✅ Right — declare the dependency and let the Effect re-run.
useEffect(() => {
  let cancelled = false;
  fetchUser(userId).then(u => { if (!cancelled) setUser(u); });
  return () => { cancelled = true; };
}, [userId]);
```

When the lint rule demands a dependency you "don't want", the real fix is almost never to suppress it. It is one of:

- move the value **inside** the Effect (if it is only used there),
- move a function **outside** the component (if it uses nothing reactive),
- wrap a value in `useMemo`/`useCallback` so its identity is stable,
- extract the non-reactive part into `useEffectEvent`,
- or delete the Effect, because it was one of the four table rows above.

### Advanced — `useEffectEvent` (React 19.2, stable)

The problem: your Effect should re-run when `roomId` changes, but *not* when `theme` changes — yet it reads `theme`.

```tsx
// ❌ Wrong — reconnects the socket every time the user toggles dark mode.
useEffect(() => {
  const conn = createConnection(roomId);
  conn.on('connected', () => showNotification('Connected!', theme));
  conn.connect();
  return () => conn.disconnect();
}, [roomId, theme]);

// ✅ Right — the notification is an "event", not a reactive dependency.
import { useEffectEvent } from 'react';

const onConnected = useEffectEvent(() => {
  showNotification('Connected!', theme);   // always reads the LATEST theme
});

useEffect(() => {
  const conn = createConnection(roomId);
  conn.on('connected', () => onConnected());
  conn.connect();
  return () => conn.disconnect();
}, [roomId]);                               // Effect Events are never dependencies
```

`useEffectEvent` shipped as stable in **React 19.2** ([release notes](https://react.dev/blog/2025/10/01/react-19-2)); before that it was `experimental_useEffectEvent` behind a canary flag, so pre-2025 examples will not compile. The linter enforces that you never list it in a dependency array and never call it outside an Effect. This is the sanctioned replacement for the `useRef`-holding-the-latest-callback pattern that older Stack Overflow answers recommend.

### Advanced — cleanup is what makes Effects safe

```text
mount        →  setup
deps change  →  cleanup(previous)  →  setup(next)
unmount      →  cleanup
StrictMode   →  setup → cleanup → setup     (dev only, on mount)
```

Every Effect that starts something must stop it. The pattern per resource type:

| Effect starts | Cleanup |
|---|---|
| `setInterval` / `setTimeout` | `clearInterval` / `clearTimeout` |
| `addEventListener` | `removeEventListener` (the *same* function reference) |
| `fetch` | `AbortController.abort()`, or an `ignore` flag |
| subscription / socket | `unsubscribe()` / `close()` |
| third-party widget | its `.destroy()` / `.dispose()` |

```tsx
// ✅ Window listener with correct cleanup.
useEffect(() => {
  const onResize = () => setWidth(window.innerWidth);
  window.addEventListener('resize', onResize);
  return () => window.removeEventListener('resize', onResize);
}, []);
```

Passing an inline arrow to `removeEventListener` — a different function object — removes nothing and leaks a listener per mount. That leak is invisible in a small demo and lethal in a route users navigate through fifty times.

### Mastery — `useLayoutEffect` and when the difference bites

```text
commit DOM  →  useLayoutEffect (sync, blocks paint)  →  browser paints  →  useEffect (async)
```

Use `useLayoutEffect` only when you must measure the DOM and change it *before* the user sees anything — tooltip positioning, scroll restoration, measuring a row height to set a container size. Everything else should be `useEffect`, because layout effects block painting.

```tsx
// ✅ Measure-then-position: with useEffect the tooltip visibly jumps.
const ref = useRef<HTMLDivElement>(null);
const [height, setHeight] = useState(0);

useLayoutEffect(() => {
  setHeight(ref.current!.getBoundingClientRect().height);
}, []);
```

In SSR, `useLayoutEffect` does not run on the server and logs a warning. React 18 added [`useInsertionEffect`](https://react.dev/reference/react/useInsertionEffect) for CSS-in-JS libraries injecting styles before layout reads — you will use it only if you write such a library.

**Try It.** Build a tooltip that measures its own height and offsets itself upward. Implement it with `useEffect`, watch the one-frame flicker, then switch to `useLayoutEffect` and watch it disappear. That flicker *is* the difference between the two hooks, made visible.

[⬆ back to top](#table-of-contents)

---

## 7. Refs & the Escape Hatch

### Beginner — a ref is a box React does not watch

```tsx
const ref = useRef<HTMLInputElement>(null);
// ref.current — mutable, persists across renders, changing it triggers NO re-render
```

Two distinct uses, same hook:

1. **Hold a DOM node** — `<input ref={ref} />` then `ref.current.focus()`.
2. **Hold a mutable value that must survive renders but is not rendered** — a timeout id, a previous value, a WebSocket, a scroll position.

```tsx
function SearchBox() {
  const inputRef = useRef<HTMLInputElement>(null);
  return (
    <>
      <input ref={inputRef} />
      <button onClick={() => inputRef.current?.focus()}>Focus</button>
    </>
  );
}
```

**state vs ref**, the whole distinction:

| | `useState` | `useRef` |
|---|---|---|
| Changing it re-renders | Yes | No |
| Readable during render | Yes | Don't (it may be stale/inconsistent) |
| Mutate directly | Never | Always (`ref.current = x`) |
| Use for | Anything the UI displays | Anything the UI does not display |

### Working Knowledge — refs as instance variables

```tsx
// ✅ A debounce timer stored in a ref, cleaned up on unmount.
function Search({ onSearch }: { onSearch: (q: string) => void }) {
  const timer = useRef<ReturnType<typeof setTimeout> | null>(null);

  useEffect(() => () => { if (timer.current) clearTimeout(timer.current); }, []);

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    const q = e.target.value;
    if (timer.current) clearTimeout(timer.current);
    timer.current = setTimeout(() => onSearch(q), 300);
  }

  return <input onChange={handleChange} />;
}
```

The failure this prevents: storing the timer in a plain local variable resets it every render; storing it in state causes a re-render per keystroke *and* the same stale-closure problems.

```tsx
// ❌ Wrong — reading/writing a ref during render breaks purity and concurrent rendering.
function Bad() {
  const renders = useRef(0);
  renders.current++;                 // mutation during render; wrong under StrictMode too
  return <p>{renders.current}</p>;   // reading during render → inconsistent output
}

// ✅ Right — mutate refs in effects and event handlers only.
function Good({ value }: { value: string }) {
  const previous = useRef(value);
  useEffect(() => { previous.current = value; }, [value]);
  return <p>now: {value}</p>;
}
```

### Advanced — `ref` is a prop now (React 19)

`forwardRef` is deprecated. Function components receive `ref` in props directly ([React 19](https://react.dev/blog/2024/12/05/react-19#ref-as-a-prop)):

```tsx
// ❌ Legacy (still works, will warn in a future major).
const Input = forwardRef<HTMLInputElement, Props>((props, ref) => <input ref={ref} {...props} />);

// ✅ React 19.
type InputProps = React.ComponentProps<'input'>;
function Input({ ref, ...props }: InputProps) {
  return <input ref={ref} {...props} />;
}
```

Ref **callbacks can now return a cleanup function**, which replaces the old "called with null on unmount" convention:

```tsx
<div
  ref={node => {
    const observer = new ResizeObserver(() => { /* … */ });
    observer.observe(node);
    return () => observer.disconnect();   // React 19: cleanup on unmount/ref change
  }}
/>
```

Note the trap this introduces: an *arrow body* like `ref={node => (nodeRef.current = node)}` now implicitly **returns** the assignment value, which React 19 rejects as an invalid cleanup. Use a block body: `ref={node => { nodeRef.current = node; }}`.

### Advanced — what refs are legitimately for

React's guidance is that refs are an escape hatch for things React does not model: focus, text selection, scrolling, media playback, measuring, and imperative third-party libraries.

```tsx
// ✅ Integrating a non-React chart library.
function Chart({ data }: { data: number[] }) {
  const el = useRef<HTMLDivElement>(null);
  const chart = useRef<ChartInstance | null>(null);

  useEffect(() => {
    chart.current = createChart(el.current!);          // create once
    return () => chart.current?.destroy();
  }, []);

  useEffect(() => { chart.current?.setData(data); }, [data]);  // sync on change

  return <div ref={el} />;
}
```

What refs are **not** for: reaching into a child to change its DOM when a prop would do, or working around a re-render you did not want. If you find yourself calling `ref.current.style.display = 'none'` on a node React also renders, React will overwrite you on the next render — the state belongs in React.

`useImperativeHandle` exists for the rare case where a component must expose a *narrow* imperative API (`{ focus, scrollIntoView }`) rather than its raw DOM node ([reference](https://react.dev/reference/react/useImperativeHandle)). Reach for it deliberately; it is not a general escape route.

[⬆ back to top](#table-of-contents)

---

## 8. Sharing State: Lifting, Context, Reducers

### Beginner — lifting state up

When two components need the same state, move it to their nearest common parent and pass it down.

```text
        App  ← state lives here (filter, results)
        ├── SearchInput   (filter, onFilterChange)
        └── ResultList    (results)
```

```tsx
function App() {
  const [filter, setFilter] = useState('');
  const results = ALL.filter(r => r.name.includes(filter));   // derived, not state
  return (
    <>
      <SearchInput value={filter} onChange={setFilter} />
      <ResultList items={results} />
    </>
  );
}
```

This is the default answer, and it is correct far more often than people expect. Reach for the tools below only when lifting stops working.

### Working Knowledge — `useReducer` when updates get complicated

`useReducer` is `useState` with the update logic named and moved outside the component. Use it when several fields change together, when the next state depends on the action type, or when you want to unit-test the transitions.

```tsx
type State = { items: Item[]; filter: string; selectedId: string | null };
type Action =
  | { type: 'added'; item: Item }
  | { type: 'removed'; id: string }
  | { type: 'filtered'; text: string };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'added':
      return { ...state, items: [...state.items, action.item] };
    case 'removed':
      return {
        ...state,
        items: state.items.filter(i => i.id !== action.id),
        selectedId: state.selectedId === action.id ? null : state.selectedId,  // invariant kept here
      };
    case 'filtered':
      return { ...state, filter: action.text };
    default: {
      const _exhaustive: never = action;   // TS catches an unhandled action at compile time
      return state;
    }
  }
}

function List() {
  const [state, dispatch] = useReducer(reducer, { items: [], filter: '', selectedId: null });
  return <button onClick={() => dispatch({ type: 'removed', id: 'x' })}>Remove</button>;
}
```

The real payoff is the `'removed'` case: "deleting the selected item must clear the selection" is an invariant that lives in *one* place instead of being re-remembered at every call site. Reducers must be **pure** — no fetching, no timers, no `Math.random()` ([Extracting State Logic into a Reducer](https://react.dev/learn/extracting-state-logic-into-a-reducer)).

### Working Knowledge — Context, to skip the prop drilling

```tsx
import { createContext, useContext, useState } from 'react';

type Theme = 'light' | 'dark';
const ThemeContext = createContext<Theme>('light');   // default is used only with NO provider above

export function App() {
  const [theme, setTheme] = useState<Theme>('dark');
  return (
    <ThemeContext value={theme}>          {/* React 19: <Context> IS the provider */}
      <Page />
    </ThemeContext>
  );
}

function DeepButton() {
  const theme = useContext(ThemeContext);
  return <button className={theme}>Save</button>;
}
```

`<ThemeContext.Provider>` still works but is deprecated in React 19 in favour of `<ThemeContext>` ([React 19](https://react.dev/blog/2024/12/05/react-19#context-as-a-provider)).

A safer typed pattern — make the missing-provider case loud instead of silently falling back to a default:

```tsx
const ThemeContext = createContext<Theme | null>(null);

export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (ctx === null) throw new Error('useTheme must be used inside <ThemeProvider>');
  return ctx;   // narrowed to Theme
}
```

### Advanced — the two Context performance traps

**Trap 1: a new object value on every render** re-renders every consumer, even if nothing they read changed.

```tsx
// ❌ Wrong — new object identity per render of the provider.
<AuthContext value={{ user, login, logout }}>

// ✅ Right — stable identity (or let the React Compiler do this; see Part 9).
const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);
<AuthContext value={value}>
```

**Trap 2: one fat context for unrelated state.** Every consumer re-renders when *any* field changes, because context has no selector API.

```tsx
// ❌ Wrong — a component that only reads `theme` re-renders on every cart change.
<AppContext value={{ user, theme, cart, notifications }}>

// ✅ Right — split by change frequency.
<ThemeContext value={theme}>
  <UserContext value={user}>
    <CartContext value={cart}>
```

A widely-used refinement is splitting **state** and **dispatch** into two contexts: `dispatch` is referentially stable forever, so components that only dispatch never re-render at all.

```tsx
const TodosContext = createContext<Todo[] | null>(null);
const TodosDispatchContext = createContext<React.Dispatch<Action> | null>(null);

export function TodosProvider({ children }: { children: React.ReactNode }) {
  const [todos, dispatch] = useReducer(reducer, []);
  return (
    <TodosContext value={todos}>
      <TodosDispatchContext value={dispatch}>{children}</TodosDispatchContext>
    </TodosContext>
  );
}
```

This reducer + context combination is React's documented ["scaling up with reducer and context"](https://react.dev/learn/scaling-up-with-reducer-and-context) pattern, and it covers most app state that people reach for Redux to handle.

**Try It.** Put `console.log` in three consumers, then update one field of a fat context. All three log. Split the context and update again — only the relevant one logs.

### Advanced — where Context is the wrong tool

Context is a **dependency injection** mechanism, not a state manager. It has no selectors, no equality checks, no middleware, no devtools time travel.

| Situation | Better fit |
|---|---|
| Server data (caching, refetch, staleness) | [TanStack Query](https://tanstack.com/query/latest) — outside this doc's scope |
| High-frequency updates (cursor position, animation) | A ref + subscription, or a store with selectors |
| Large global state read by many components selectively | [Zustand](https://zustand.docs.pmnd.rs/) / [Jotai](https://jotai.org/) / Redux Toolkit |
| Theme, locale, current user, feature flags | Context — this is exactly what it is for |

The signal that you have outgrown context: you start memoising consumers to stop unrelated re-renders. That is a selector-shaped problem, and context has no selectors.

### Mastery — `useSyncExternalStore`

When state lives *outside* React — a custom store, `localStorage`, `matchMedia`, an event emitter — this is the correct subscription hook. It is tear-free under concurrent rendering, which a naive `useState` + `useEffect` subscription is not.

```tsx
import { useSyncExternalStore } from 'react';

function useOnlineStatus() {
  return useSyncExternalStore(
    callback => {                                        // subscribe
      window.addEventListener('online', callback);
      window.addEventListener('offline', callback);
      return () => {
        window.removeEventListener('online', callback);
        window.removeEventListener('offline', callback);
      };
    },
    () => navigator.onLine,                              // client snapshot
    () => true,                                          // server snapshot (SSR)
  );
}
```

The snapshot function must return a **cached, referentially stable** value for unchanged state — returning a fresh object each call causes an infinite render loop, which is the number-one reported issue with this hook ([reference](https://react.dev/reference/react/useSyncExternalStore)). Every modern external store library (Zustand, Redux) uses this hook internally; that is the main reason to understand it.

[⬆ back to top](#table-of-contents)

---

## 9. Performance & Referential Identity

### Beginner — measure before you optimise

React re-renders a component when its state changes, its context changes, or its parent re-renders. A re-render is *not* a DOM update: React re-runs your function, diffs, and usually touches nothing. Most re-renders are cheap and not worth a single line of memoisation.

Tools, in order of usefulness:

1. **React DevTools Profiler** — record an interaction, read the flamegraph, sort by "Ranked" to find the actual cost.
2. **"Highlight updates when components render"** in DevTools settings — instant visual feedback on what re-renders.
3. **Chrome DevTools performance tracks** — React 19.2 adds dedicated *Scheduler* and *Components* tracks showing render/effect timing by priority ([19.2 release](https://react.dev/blog/2025/10/01/react-19-2)).

### Working Knowledge — the three memo APIs

```tsx
const Memo = memo(Component);            // skip re-render if props are shallow-equal
const value = useMemo(() => f(a, b), [a, b]);   // cache a VALUE between renders
const fn = useCallback(() => g(a), [a]);        // cache a FUNCTION identity between renders
```

`useCallback(fn, deps)` is exactly `useMemo(() => fn, deps)`. They exist as separate hooks only for ergonomics.

The critical point: **`memo` is useless if the props are new objects every render.** These three APIs are one system.

```tsx
// ❌ Wrong — memo does nothing: `style` and `onSelect` are new objects/functions each render.
const Row = memo(function Row({ item, style, onSelect }: RowProps) { … });

function List({ items }: { items: Item[] }) {
  return items.map(item => (
    <Row key={item.id} item={item} style={{ padding: 8 }} onSelect={() => pick(item.id)} />
  ));
}

// ✅ Right — stable identities, so memo can actually bail out.
const ROW_STYLE = { padding: 8 };                       // hoisted: same object forever

function List({ items }: { items: Item[] }) {
  const onSelect = useCallback((id: string) => pick(id), []);
  return items.map(item => (
    <Row key={item.id} item={item} style={ROW_STYLE} onSelect={onSelect} />
  ));
}
```

Note the second fix: passing `onSelect(id)` instead of a per-item closure is what makes one stable callback serve every row.

### Working Knowledge — the free optimisation: children as props

Before memoising anything, try restructuring. A component passed as `children` is created by the *parent*, so a state change in the wrapper does not re-render it:

```tsx
// ❌ Wrong — <SlowTree /> re-renders on every colour change.
function App() {
  const [color, setColor] = useState('red');
  return (
    <div style={{ color }}>
      <input value={color} onChange={e => setColor(e.target.value)} />
      <SlowTree />
    </div>
  );
}

// ✅ Right — the state moves down into a wrapper; SlowTree is a prop, so it is not re-created.
function ColorLayout({ children }: { children: React.ReactNode }) {
  const [color, setColor] = useState('red');
  return (
    <div style={{ color }}>
      <input value={color} onChange={e => setColor(e.target.value)} />
      {children}
    </div>
  );
}

function App() {
  return <ColorLayout><SlowTree /></ColorLayout>;
}
```

No `memo`, no dependencies to maintain, and it cannot go stale. Prefer this shape whenever it applies.

### Advanced — the React Compiler, and what it does to all of the above

The React Compiler reached **1.0 stable in October 2025** ([announcement](https://react.dev/blog/2025/10/07/react-compiler-1)). It is a build-time tool that analyses your components and inserts memoisation automatically — roughly, it does what a perfectly disciplined developer would do with `useMemo`/`useCallback`/`memo`, on every value, without you writing any of it.

If you scaffolded with `react-compiler-ts` it is already on. Manual wiring for an existing Vite project:

```bash
npm install -D babel-plugin-react-compiler@latest @rolldown/plugin-babel
npm install -D eslint-plugin-react-hooks@latest
```

```ts
// vite.config.ts — for Vite 6+/8 with @vitejs/plugin-react v6+
import { defineConfig } from 'vite';
import react, { reactCompilerPreset } from '@vitejs/plugin-react';
import babel from '@rolldown/plugin-babel';

export default defineConfig({
  plugins: [
    react(),
    babel({ presets: [reactCompilerPreset()] }),
  ],
});
```

> **Version trap worth knowing about:** `@vitejs/plugin-react` v6 replaced Babel with **oxc** for the JSX transform and Fast Refresh, and *removed inline Babel options*. The compiler still needs Babel, hence the separate `@rolldown/plugin-babel`. Any blog post showing `react({ babel: { plugins: [['babel-plugin-react-compiler', {}]] } })` is targeting plugin-react v5 or earlier ([current install docs](https://react.dev/learn/react-compiler/installation)).

What changes in practice:

| | Without the compiler | With the compiler |
|---|---|---|
| `useMemo` / `useCallback` | You write them by hand where profiling says so | Mostly unnecessary; the compiler memoises automatically |
| `memo()` | Needed for expensive children | Mostly unnecessary |
| Rules of React violations | Cause subtle bugs | Cause the compiler to **skip** that component (it bails out safely) |
| The ESLint plugin | Nice to have | Essential — it reports exactly what the compiler will refuse to optimise |

Two things the compiler does **not** do, so you still need to understand memoisation:

1. It **bails out of impure components** rather than fixing them. A component that mutates props or reads a ref during render is silently left unoptimised. `eslint-plugin-react-hooks` v6's compiler-powered rules tell you which and why.
2. It does not memoise across a **non-React boundary** — values you hand to an external library, a `useSyncExternalStore` snapshot, or the `useEffect` dependency of a value produced outside the component are still yours to stabilise.

The honest stance for 2026: **write clean, pure components and let the compiler handle memoisation**; keep manual `useMemo` only for genuinely expensive computations you have profiled, and expect to read plenty of hand-memoised code in existing codebases for years.

### Advanced — when `useMemo` is still the right call

```tsx
// ✅ Genuinely expensive: a large sort/filter you can measure.
const visible = useMemo(
  () => rows.filter(r => r.tags.includes(tag)).toSorted(byDate),
  [rows, tag],
);

// ✅ Referential stability required by a non-React consumer.
const options = useMemo(() => ({ threshold: 0.5 }), []);
useEffect(() => {
  const io = new IntersectionObserver(cb, options);
  return () => io.disconnect();
}, [options]);
```

And when it is not:

```tsx
// ❌ Wrong — the memo costs more than the work it saves.
const doubled = useMemo(() => count * 2, [count]);
```

`useMemo` is not free: it allocates, stores dependencies, and compares them on every render. For arithmetic and short string work, computing directly is faster.

### Mastery — the render-cost hierarchy

When the profiler says a render is slow, the cause is almost always one of these, in descending order of frequency:

```text
1. Rendering too many nodes        → virtualise the list (react-window / TanStack Virtual)
2. Re-rendering a large subtree    → move state down, or pass children as props
3. Expensive work per render       → useMemo (measured, not guessed)
4. Layout thrash in effects        → batch DOM reads then writes; prefer useEffect
5. Context value churn             → split contexts / stabilise value
```

Reaching for `memo` first, without profiling, is the most common wasted effort in React codebases: it adds dependency-array maintenance and a comparison cost to every render in exchange for skipping work that was already cheap.

**Try It.** Render a 5,000-row table and profile it. Then wrap the row in `memo` — the profile barely moves, because the cost is the *number of nodes*, not re-render overhead. Then virtualise it and watch the render time collapse. That experiment permanently changes where you look first.

[⬆ back to top](#table-of-contents)

---

## 10. Concurrent React

### Beginner — the one idea

React can work on a render **without committing it**, pause it, and let a more urgent update jump the queue. Two priorities, informally:

```text
urgent      typing, clicking, hovering        must feel instant
transition  filtering a list, switching tabs  may take a moment, must not block typing
```

Nothing here is automatic magic: you opt in by marking updates as transitions.

### Working Knowledge — `useTransition` and `useDeferredValue`

```tsx
// useTransition: you control the state update, and get a pending flag.
const [isPending, startTransition] = useTransition();

function onTabClick(tab: Tab) {
  startTransition(() => setTab(tab));    // rendering the new tab won't block the click
}
return <div style={{ opacity: isPending ? 0.6 : 1 }}>…</div>;
```

```tsx
// useDeferredValue: you don't control the update (it's a prop, or comes from an input).
function SearchResults({ query }: { query: string }) {
  const deferredQuery = useDeferredValue(query);
  const stale = query !== deferredQuery;
  const results = useMemo(() => search(deferredQuery), [deferredQuery]);
  return <ul style={{ opacity: stale ? 0.6 : 1 }}>{results.map(…)}</ul>;
}
```

Choosing between them: **`useTransition` when you own the setter; `useDeferredValue` when you only receive the value.** Both keep the old UI on screen while the new one renders in the background.

```text
Typing "reactjs" into a filter over 10,000 rows:

Without transitions              With useDeferredValue
──────────────────               ─────────────────────
key "r" → filter 10k → paint     key "r" → input updates instantly (urgent)
key "e" → BLOCKED…               list re-renders in background (transition)
input stutters, dropped frames   later keystrokes ABANDON the in-flight render
                                 input stays at 60fps; list catches up
```

**Try It.** Render 10,000 `<li>`s filtered by an input's value with no transition, and type fast — the input visibly lags. Wrap the filter value in `useDeferredValue` and type again. Nothing about the filtering got faster; the *input* just stopped waiting for it.

Caveat worth internalising: transitions do not make slow renders fast. If a single render of the list takes 400ms, it still takes 400ms — you have only stopped it from blocking input. Deferring is not a substitute for virtualising.

### Advanced — Suspense

`<Suspense>` shows a fallback while a child is "not ready". It is triggered by a component *suspending*, which happens when it reads a resource that is not yet available — a lazily-loaded component, or a promise passed to `use()`.

```tsx
import { lazy, Suspense } from 'react';

const Settings = lazy(() => import('./Settings'));

<Suspense fallback={<Spinner />}>
  <Settings />
</Suspense>
```

Rules that trip people up:

- **A Suspense boundary shows its fallback only for content that has not yet rendered.** If an *already-visible* subtree suspends because of a state update, React replaces it with the fallback — unless the update is inside a transition, in which case the old content stays on screen. This is why `startTransition` and Suspense are usually used together.
- **Boundary placement is a design decision.** One boundary at the root means the whole page flashes; boundaries per section give independent loading states.
- Nested boundaries reveal outside-in, so the user sees a stable shell filling in rather than a full-page spinner.

React 19.2 additionally **batches Suspense reveals** for a short window during SSR, so multiple boundaries that resolve nearly together appear at once instead of popcorning ([19.2 release](https://react.dev/blog/2025/10/01/react-19-2)).

### Advanced — `use()`

`use()` reads a resource during render. Unlike every other hook, it **can** be called conditionally and inside loops.

```tsx
import { use, Suspense } from 'react';

function Comments({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  const comments = use(commentsPromise);      // suspends until resolved
  return <ul>{comments.map(c => <li key={c.id}>{c.text}</li>)}</ul>;
}

<Suspense fallback={<p>Loading comments…</p>}>
  <Comments commentsPromise={fetchComments()} />   {/* promise created by the PARENT */}
</Suspense>
```

```tsx
// ❌ Wrong — a new promise every render: suspend, re-render, new promise, suspend… infinite loop.
function Comments() {
  const comments = use(fetch('/api/comments').then(r => r.json()));
  …
}
```

This is the crucial constraint on client-side `use()`: the promise must be **cached or created outside the render**. In a client-only app there is no framework-level cache to do that for you, which is precisely why data libraries exist ([Part 11](#11-data-fetching--async-ui)). `use()` also reads context — `use(ThemeContext)` — and *that* usage is immediately practical, because it can sit inside an `if`.

### Mastery — `<Activity>` (React 19.2)

`<Activity>` renders a subtree in `hidden` mode: the DOM is created, effects are unmounted, updates are deferred until React has nothing better to do — but **state is preserved**.

```tsx
import { Activity } from 'react';

<Activity mode={tab === 'search' ? 'visible' : 'hidden'}>
  <SearchPanel />        {/* keeps scroll position and inputs when you tab away */}
</Activity>
```

Two uses: preserving the state of a section the user navigated away from, and pre-rendering a section they are likely to visit next, at low priority. Because hidden mode unmounts effects, your effects must be correctly paired setup/cleanup — the same discipline StrictMode has been rehearsing since [Part 1](#1-setup--your-first-render). This is a genuinely new capability in the 19.2 line; treat any pre-2026 article describing it as experimental as out of date.

[⬆ back to top](#table-of-contents)

---

## 11. Data Fetching & Async UI

### Beginner — the naive fetch, and why it is a teaching example only

```tsx
function User({ id }: { id: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    setLoading(true);
    fetch(`/api/users/${id}`)
      .then(r => r.json())
      .then(setUser)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [id]);

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <Profile user={user!} />;
}
```

This works in a demo and is wrong in production. It has at least four defects, covered below.

### Working Knowledge — defect 1: the race condition

Change `id` from `1` to `2` quickly. Two requests are in flight. If request 1 resolves *after* request 2, you render user 1's data on user 2's page — permanently, with no error anywhere.

```text
t0  id=1  ──── fetch(1) ───────────────────────────────► resolves  setUser(user1)  ← WRONG, wins
t1  id=2  ──── fetch(2) ──────────► resolves  setUser(user2)
                                                          screen now shows user 1
```

```tsx
// ✅ Right — the "ignore" flag. Cleanup marks the previous request's result as unwanted.
useEffect(() => {
  let ignore = false;
  setUser(null);
  fetch(`/api/users/${id}`)
    .then(r => r.json())
    .then(data => { if (!ignore) setUser(data); });
  return () => { ignore = true; };
}, [id]);
```

```tsx
// ✅ Better — actually cancel the request, not just its result.
useEffect(() => {
  const controller = new AbortController();
  fetch(`/api/users/${id}`, { signal: controller.signal })
    .then(r => r.json())
    .then(setUser)
    .catch(err => { if (err.name !== 'AbortError') setError(err); });   // ignore our own aborts
  return () => controller.abort();
}, [id]);
```

Note the `AbortError` filter: without it, every navigation logs a spurious error. React's own [fetching guidance](https://react.dev/reference/react/useEffect#fetching-data-with-effects) uses the ignore flag; `AbortController` additionally frees the network connection.

### Working Knowledge — defect 2: `fetch` does not reject on HTTP errors

```tsx
// ❌ Wrong — a 500 with an HTML error page parses as… whatever, and lands in setUser.
const data = await fetch(url).then(r => r.json());

// ✅ Right — check `ok` yourself.
const res = await fetch(url);
if (!res.ok) throw new Error(`${res.status} ${res.statusText}`);
const data = await res.json();
```

`fetch` rejects only on network failure. This is standard `fetch` behaviour, not a React thing, but it hides inside every hand-rolled React data hook ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Window/fetch)).

### Advanced — defects 3 and 4: waterfalls and no cache

**Waterfalls.** A parent that fetches, then renders a child that fetches, serialises two round trips that could have been parallel:

```text
❌ Waterfall                          ✅ Parallel
parent fetch ──400ms──┐               parent fetch ──400ms──┐
                      └─ child fetch ──400ms──►             ├─ both start together
total: 800ms                          child fetch ──400ms──┘
                                      total: 400ms
```

Fix by hoisting the requests (`Promise.all` in one place) or by rendering both children under one Suspense boundary with their promises created by the parent.

**No cache.** Navigate away and back and you refetch everything. Nothing dedupes two components requesting the same endpoint. Nothing revalidates on window focus. Nothing retries.

This is the honest conclusion of Part 11: **hand-rolled fetching in `useEffect` is correct only for the simplest cases**, and React's docs say so directly, recommending a framework's data layer or a library like [TanStack Query](https://tanstack.com/query/latest) or [SWR](https://swr.vercel.app/) ([Effects: fetching data](https://react.dev/reference/react/useEffect#fetching-data-with-effects)). The value of writing it by hand once is that you can then read a library's docs and recognise *which* of these four defects each feature exists to fix.

### Advanced — a reusable fetch hook, done properly

If you do hand-roll it, this is the shape worth keeping:

```tsx
type Result<T> =
  | { status: 'loading' }
  | { status: 'error'; error: Error }
  | { status: 'success'; data: T };

function useFetch<T>(url: string): Result<T> {
  const [result, setResult] = useState<Result<T>>({ status: 'loading' });

  useEffect(() => {
    const controller = new AbortController();
    setResult({ status: 'loading' });

    (async () => {
      try {
        const res = await fetch(url, { signal: controller.signal });
        if (!res.ok) throw new Error(`${res.status} ${res.statusText}`);
        setResult({ status: 'success', data: (await res.json()) as T });
      } catch (error) {
        if ((error as Error).name !== 'AbortError') {
          setResult({ status: 'error', error: error as Error });
        }
      }
    })();

    return () => controller.abort();
  }, [url]);

  return result;
}
```

The discriminated union is doing real work here: it makes `loading && error && data` states unrepresentable, so you cannot render a spinner and stale data simultaneously — a bug three separate `useState` booleans invite.

### Mastery — Error Boundaries

React has no hook for catching render errors. Error boundaries are the **one remaining reason to write a class component**, unless you use [`react-error-boundary`](https://github.com/bvaughn/react-error-boundary), which is the conventional answer.

```tsx
class ErrorBoundary extends React.Component<
  { fallback: React.ReactNode; children: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  componentDidCatch(error: Error, info: React.ErrorInfo) { logToService(error, info); }
  render() { return this.state.hasError ? this.props.fallback : this.props.children; }
}
```

What boundaries **do not** catch: errors in event handlers, in `setTimeout`, in async code after an `await`, and in the boundary itself. Those need ordinary `try/catch`. Pair a boundary with a Suspense boundary to cover both axes:

```tsx
<ErrorBoundary fallback={<p>Could not load comments.</p>}>
  <Suspense fallback={<Spinner />}>
    <Comments commentsPromise={promise} />
  </Suspense>
</ErrorBoundary>
```

React 19 also added root-level error hooks — `onCaughtError`, `onUncaughtError`, `onRecoverableError` on `createRoot` — which is where you wire your error reporting service ([React 19](https://react.dev/blog/2024/12/05/react-19#error-handling)).

[⬆ back to top](#table-of-contents)

---

## 12. Custom Hooks

### Beginner — a custom hook is a function that calls hooks

```tsx
function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn(o => !o), []);
  return [on, toggle] as const;
}

// Usage
const [isOpen, toggleOpen] = useToggle();
```

The `use` prefix is not decoration — the linter uses it to decide where to enforce the Rules of Hooks. A function named `getToggle` that calls `useState` will not be checked, and will break silently.

### Working Knowledge — custom hooks share *logic*, never state

This is the most misunderstood point about them:

```tsx
function useCounter() {
  const [count, setCount] = useState(0);
  return { count, increment: () => setCount(c => c + 1) };
}

function A() { const { count } = useCounter(); }   // A has its own count
function B() { const { count } = useCounter(); }   // B has a SEPARATE count
```

Each call site gets a fresh, independent instance of every hook inside. To *share* state you still need lifting or context ([Part 8](#8-sharing-state-lifting-context-reducers)) — a custom hook only shares the code.

### Working Knowledge — the everyday ones, worth writing yourself once

```tsx
// Debounce a value.
function useDebounced<T>(value: T, delay = 300): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);      // cancel the pending update on every change
  }, [value, delay]);
  return debounced;
}

// Persist state to localStorage.
function useLocalStorage<T>(key: string, initial: T) {
  const [value, setValue] = useState<T>(() => {              // lazy init: read storage once
    try { const raw = localStorage.getItem(key); return raw ? (JSON.parse(raw) as T) : initial; }
    catch { return initial; }
  });
  useEffect(() => { localStorage.setItem(key, JSON.stringify(value)); }, [key, value]);
  return [value, setValue] as const;
}

// A media query, via the concurrent-safe subscription hook from Part 8.
function useMediaQuery(query: string) {
  const mql = useMemo(() => window.matchMedia(query), [query]);
  return useSyncExternalStore(
    cb => { mql.addEventListener('change', cb); return () => mql.removeEventListener('change', cb); },
    () => mql.matches,
    () => false,
  );
}
```

`useDebounced` is the one to study: the cleanup clearing the previous timeout is the entire debounce mechanism. Without it you get a burst of updates rather than one.

### Advanced — designing the interface

Return whatever shape reads best at the call site, and be consistent:

- **Tuple** (`[value, setValue] as const`) when the caller will want to rename both — mirrors `useState`.
- **Object** (`{ data, error, isLoading }`) when there are more than two values, or some are optional.
- `as const` matters in TypeScript: without it, `[string, () => void]` widens to `(string | (() => void))[]` and destructuring loses the types.

Two design rules that keep hooks composable:

```tsx
// ❌ Wrong — a hook that owns a fetch AND a modal AND a form. Untestable, unreusable.
function useUserPageEverything(id: string) { … }

// ✅ Right — small hooks composed at the call site.
function UserPage({ id }: { id: string }) {
  const user = useFetch<User>(`/api/users/${id}`);
  const [isEditing, toggleEditing] = useToggle();
  …
}
```

And: **hooks must be called unconditionally**, so a custom hook may not be called inside an `if` either. If a hook should sometimes do nothing, pass that in as an argument (`useFetch(shouldLoad ? url : null)`) and branch *inside* the hook.

### Mastery — testing hooks, and the effect-vs-render boundary

Custom hooks are tested through a component, or with `renderHook` from [React Testing Library](https://testing-library.com/docs/react-testing-library/api#renderhook):

```tsx
import { renderHook, act } from '@testing-library/react';

test('useToggle flips', () => {
  const { result } = renderHook(() => useToggle(false));
  expect(result.current[0]).toBe(false);
  act(() => { result.current[1](); });    // act() flushes React's work before you assert
  expect(result.current[0]).toBe(true);
});
```

`act()` exists because state updates are asynchronous with respect to your test: without it you assert on the pre-update render and get a warning telling you exactly that.

The deeper design lesson: a hook whose logic lives in **render** (derivation, computation) is trivial to test and always correct. A hook whose logic lives in **effects** must be tested with timers, mocks, and `act`. When you can move work from an effect into render — the [Part 6](#6-effects--synchronisation) table — you get testability for free as a side benefit.

[⬆ back to top](#table-of-contents)

---

## 13. Under the Hood

This Part is Mastery-tier throughout. None of it is required to write good React; all of it makes the earlier Parts stop feeling arbitrary.

### Fibers: the unit of work

A **fiber** is a plain JavaScript object representing one node of work — one component instance, one host element. React maintains a tree of them, linked as `child` / `sibling` / `return`, which lets it traverse the tree with a loop instead of recursion — and therefore *pause* mid-traversal.

```text
        App(fiber)
         │ child
         ▼
      Header ──sibling──► Main ──sibling──► Footer
         │ return                │ child
         └──► App                ▼
                              ItemList
```

Each fiber holds: `type`, `props` (pending and memoised), `stateNode` (the DOM node or class instance), the hooks linked list, an effect flag, and a pointer to its **alternate** — the other copy in the double-buffered pair.

### Double buffering: `current` vs `workInProgress`

```text
       ┌─────────── current tree ───────────┐   ← what's on screen
       │  (committed, matches the DOM)      │
       └────────────┬───────────────────────┘
                    │ alternate pointers
       ┌────────────┴───────────────────────┐
       │  workInProgress tree               │   ← being built; may be thrown away
       └────────────────────────────────────┘

  commit = swap the pointer.  root.current = workInProgress
```

Two trees, alternating. This is *why* an interrupted render costs nothing: React abandons the work-in-progress tree and the on-screen tree was never touched. It is also why your component must be pure — React may build a work-in-progress tree it never commits, and any side effect you performed during that render already escaped.

### Render phase vs commit phase

```text
RENDER PHASE  (asynchronous, interruptible, may be abandoned or restarted)
  · call your component functions
  · compute new JSX
  · diff against the current tree, tag fibers with effect flags
  · ← anything you do here may happen twice, or never be committed

COMMIT PHASE  (synchronous, uninterruptible)
  · mutate the DOM
  · run useLayoutEffect + ref attachment  → then the browser paints
  · run useEffect (passive, after paint)
```

Every rule in this document derives from this split: purity requirements, why refs must not be read during render, why `useLayoutEffect` blocks paint, why StrictMode can double-invoke safely.

### Reconciliation: the two heuristics

React's diff is O(n) rather than O(n³) because it assumes two things ([Reconciliation](https://legacy.reactjs.org/docs/reconciliation.html) — still the clearest write-up, though it lives in the legacy docs):

1. **Different element types produce different trees.** `<div>` → `<span>` means destroy and rebuild the whole subtree, no matter how similar the contents.
2. **Keys tell React which children are stable across renders.** Without keys, React compares children positionally — which is exactly the [Part 3](#3-lists--conditional-rendering) bug.

Neither is universally true; both are true often enough that the algorithm is fast and predictable, and predictability is why you can reason about state preservation at all.

### Hooks as a linked list — why call order is the rule

Hooks are stored on the fiber as an ordered linked list. `useState` does not know its own name; it takes the *next* node in the list and advances a cursor.

```text
Fiber.memoizedState
  → hook1 { state: "", queue }        ← useState('')      1st call
  → hook2 { state: 0, queue }         ← useState(0)       2nd call
  → hook3 { deps: [id], destroy }     ← useEffect(…)      3rd call
```

```tsx
// ❌ Wrong — a conditional hook shifts the whole list on the render where it is skipped.
function Bad({ show }: { show: boolean }) {
  if (show) {
    const [a, setA] = useState('a');   // present sometimes
  }
  const [b, setB] = useState('b');     // now reads slot 1 on one render, slot 2 on another:
  …                                     // b gets a's value, effects mismatch, or React throws
}

// ✅ Right — always call, branch on the value instead.
function Good({ show }: { show: boolean }) {
  const [a, setA] = useState('a');
  const [b, setB] = useState('b');
  return show ? <p>{a}</p> : <p>{b}</p>;
}
```

This is not a stylistic preference — it is the storage mechanism. And it explains `use()`'s exemption in [Part 10](#10-concurrent-react): `use()` does not allocate a hook slot, so it can be called conditionally.

### Lanes: how priority is represented

React encodes update priority as a **bitmask of lanes** (SyncLane, InputContinuousLane, DefaultLane, TransitionLanes, IdleLane…). Every update is tagged with a lane; the scheduler picks the highest-priority set of lanes with pending work and renders those. Because lanes are bits, "is there any pending work at or above this priority" is a single bitwise operation.

Practically, this is what makes `startTransition` real: your transition update lands in a transition lane, a subsequent keystroke lands in a sync lane, and React renders the keystroke first, discarding the in-progress transition render if needed.

### Bailouts: when React skips your component

Even without `memo`, React bails out of re-rendering a subtree when:

- you call `setState` with a value `Object.is`-equal to the current one (though React may still re-render the component *once* before bailing — hence "eager state" comparison), **and**
- no other update targets that fiber, **and**
- the element object is referentially identical to last time (which is what makes the "children as props" trick in [Part 9](#9-performance--referential-identity) work — the parent's re-render produces the *same* child element object).

That last bullet is the mechanism behind an optimisation that looks like magic when you first meet it.

### Reading the source, if you want to go further

The React repository is more approachable than its reputation. Three files worth an afternoon: `ReactFiberBeginWork.js` (the render phase, per fiber type), `ReactFiberCommitWork.js` (the commit phase), and `ReactFiberHooks.js` (every hook, in one file, mount and update implementations side by side). Start at [`packages/react-reconciler`](https://github.com/facebook/react/tree/main/packages/react-reconciler).

[⬆ back to top](#table-of-contents)

---

## 14. Cheat Sheets

### Hooks at a glance

| Hook | Import from | Use it for | Watch out for |
|---|---|---|---|
| `useState` | `react` | Values the UI displays | Snapshot semantics; use the updater form for derived-from-previous |
| `useReducer` | `react` | Multi-field / invariant-heavy updates | Reducer must be pure |
| `useContext` | `react` | Reading injected values | Re-renders all consumers on any value change |
| `useRef` | `react` | DOM nodes, mutable non-rendered values | Never read/write during render |
| `useEffect` | `react` | Syncing with external systems | Most Effects should not exist — see Part 6 |
| `useLayoutEffect` | `react` | Measure + mutate before paint | Blocks painting; no-op on the server |
| `useEffectEvent` | `react` (19.2+) | Non-reactive logic inside an Effect | Never a dependency; only callable from an Effect |
| `useMemo` | `react` | Caching an expensive value | Often unnecessary with the compiler |
| `useCallback` | `react` | Stable function identity | Same |
| `memo` | `react` | Skip re-render on equal props | Useless if props are new objects each render |
| `useTransition` | `react` | Marking your own updates non-urgent | Doesn't make renders faster |
| `useDeferredValue` | `react` | Lagging a value you don't control | Renders twice per change by design |
| `use` | `react` | Reading a promise or context during render | Promise must be cached outside render |
| `useOptimistic` | `react` | Instant UI before the server confirms | Only holds during a transition/action |
| `useActionState` | `react` | Form action + pending + returned state | Renamed from `useFormState` |
| `useFormStatus` | `react-dom` | Pending state of the enclosing form | Must be in a **child** of the `<form>` |
| `useId` | `react` | SSR-safe unique ids for a11y attributes | Not for list keys |
| `useSyncExternalStore` | `react` | Subscribing to an external store | Snapshot must be referentially stable |
| `useImperativeHandle` | `react` | Exposing a narrow imperative API | Rare; prefer props |

### Rules of React, condensed

| Rule | Why |
|---|---|
| Components and hooks must be pure | React may call them twice or abandon the render |
| Never mutate props, state, or context | Identity comparison is how React detects change |
| Call hooks unconditionally, top level | Hooks are stored by call order in a linked list |
| Call hooks only from components/custom hooks | Otherwise there is no fiber to store them on |
| Don't call component functions yourself | `Comp()` gets no fiber, no state, no hooks — render `<Comp />` |
| Clean up everything an Effect starts | Effects re-run; unclean setups leak |

### Common bugs → cause → fix

| Symptom | Cause | Fix |
|---|---|---|
| State "one step behind" | Snapshot semantics | `setX(prev => …)` |
| A stray `0` on the page | `{items.length && …}` | `{items.length > 0 && …}` |
| Input loses focus after one keystroke | Component defined inside a component | Move it to module scope |
| Wrong row keeps checkbox/focus after delete | Index keys | Key by a stable data id |
| Effect runs twice on mount | StrictMode (dev only) | Nothing — write correct cleanup |
| Infinite render loop | `setState` during render, or an object/array dependency recreated each render | Move to a handler/effect; stabilise the dependency |
| Effect never re-runs on prop change | Suppressed lint rule / `[]` | Declare real dependencies |
| Data from a previous request wins | Race condition | `ignore` flag or `AbortController` |
| `useFormStatus` always `pending: false` | Called in the same component as `<form>` | Move it into a child |
| "Uncontrolled → controlled" warning | Initial value `undefined` | Initialise to `''` |
| Whole app re-renders on one context field | Fat context | Split contexts; split state/dispatch |
| Ref callback error in React 19 | Arrow implicitly returns the assignment | Use a block body `{ }` |

### React 19 / 19.2 migration notes

| Old | New | Status |
|---|---|---|
| `ReactDOM.render` | `createRoot().render` | **Removed** in 19 |
| `forwardRef` | `ref` as a normal prop | Deprecated |
| `<Context.Provider>` | `<Context>` | Deprecated |
| `useFormState` (`react-dom`) | `useActionState` (`react`) | Renamed |
| `propTypes`, `defaultProps` (functions) | TypeScript, default parameters | **Removed** |
| `e.persist()` | — | No-op since React 17 |
| `experimental_useEffectEvent` | `useEffectEvent` | Stable in 19.2 |
| Manual `useMemo`/`useCallback` everywhere | React Compiler 1.0 | Stable since Oct 2025 |

### Class → hooks translation (for reading legacy code only)

| Class API | Hooks equivalent |
|---|---|
| `this.state` / `this.setState` | `useState` / `useReducer` |
| `constructor` initialisation | `useState(() => initial)` |
| `componentDidMount` | `useEffect(fn, [])` |
| `componentDidUpdate(prevProps)` | `useEffect(fn, [deps])` |
| `componentWillUnmount` | the cleanup returned from `useEffect` |
| `getDerivedStateFromProps` | Derive during render, or reset with a `key` |
| `shouldComponentUpdate` | `memo(Component)` |
| `getSnapshotBeforeUpdate` | `useLayoutEffect` |
| `componentDidCatch` / `getDerivedStateFromError` | **No hook equivalent** — still requires a class, or `react-error-boundary` |
| `this.myField = x` (instance variable) | `useRef` |
| HOCs (`withRouter`, `connect`) | Custom hooks |
| Render props | Custom hooks, or `children` as a function |

### Project setup quick reference

```bash
npm create vite@latest my-app -- --template react-compiler-ts   # React + TS + compiler
npm create vite@latest my-app -- --template react-ts            # React + TS, no compiler
npm run dev            # dev server with Fast Refresh
npm run build          # production build (no StrictMode double-invoke, no dev warnings)
npm run preview        # serve the production build locally
```

[⬆ back to top](#table-of-contents)

---

## 15. Suggested Learning / Reference Order

**First pass — build a working mental model (Parts 1–6).** Do these in order and actually run the Try It exercises; each one demonstrates something that reads as arbitrary until you have seen it fail.

1. Part 1 — scaffold the lab, and see that `<App />` is a value.
2. Part 2 — components, props, composition. Run the "component inside a component" bug.
3. Part 3 — lists and keys. Run the index-key deletion demo. This is the single highest-value 10 minutes in the doc.
4. Part 4 — state as a snapshot; the `setCount(c => c + 1)` experiment.
5. Part 5 — controlled inputs, then form Actions.
6. Part 6 — Effects, cleanup, and the "you might not need an Effect" table. Re-read that table after you have written a few Effects; it lands differently the second time.

**Second pass — structure (Parts 7, 8, 12).** Refs, then lifting → reducer → context, then extract your own custom hooks from what you have built. These three are the ones that decide whether an app stays maintainable.

**Third pass — production concerns (Parts 9, 10, 11).** Performance and the compiler, then concurrent features, then data fetching. Do Part 11 last of the three: its lesson is partly "use a library", and that only lands once you have felt the four defects yourself.

**Fourth pass — Part 13, whenever you want.** Read it when something in Parts 1–12 still feels like a rule you memorised rather than a consequence you understand. Fibers explain purity; the hooks linked list explains the Rules of Hooks; lanes explain transitions.

**As a reference afterwards:** the Part 14 cheat sheets are the intended entry point. The "Common bugs → cause → fix" table is the one you will use most.

**Where to go next, beyond this doc's scope:** routing ([React Router](https://reactrouter.com/) or [TanStack Router](https://tanstack.com/router)), server state ([TanStack Query](https://tanstack.com/query/latest)), testing ([React Testing Library](https://testing-library.com/docs/react-testing-library/intro/) + [Vitest](https://vitest.dev/)), then Server Components via [Next.js](https://nextjs.org/docs) — which is where the [React docs' own framework guidance](https://react.dev/learn/creating-a-react-app) points.

[⬆ back to top](#table-of-contents)

---

## 16. Quick Self-Check

If you can answer these without looking, you have the material. Each maps to a Part.

**Fundamentals**

1. What does `<App />` evaluate to, and when does your component function actually run? *(Part 1)*
2. Why does removing `<StrictMode>` to stop double-logging make your app worse, not better? *(Part 1)*
3. What breaks if you define a component inside another component, and why is it the *type* that matters? *(Part 2)*
4. Name three things you may not do during render, and one mutation that is always safe. *(Part 2)*
5. Why does `{items.length && <List />}` render a `0`, but `{items.length > 0 && <List />}` render nothing? *(Part 3)*
6. Given a list keyed by index, describe exactly what goes wrong when the middle item is deleted. *(Part 3)*
7. What is the one situation where changing a `key` is the *goal* rather than a bug? *(Part 3)*

**State and effects**

8. Why does calling `setCount(count + 1)` three times in a handler increment by one? *(Part 4)*
9. Give two cases where a `useState` should have been a derived value or a `useRef`. *(Parts 4, 7)*
10. When does `useState(expensiveInit())` bite you, and what is the fix? *(Part 4)*
11. What is the difference between an Effect and an event handler, stated as a rule? *(Part 6)*
12. The lint rule wants a dependency you "don't want". Name three legitimate fixes that are not `eslint-disable`. *(Part 6)*
13. What problem does `useEffectEvent` solve that a dependency array cannot? *(Part 6)*
14. When is `useLayoutEffect` correct, and what does it cost? *(Part 6)*

**Structure and performance**

15. Why does a custom hook share logic but never state? *(Part 12)*
16. What are the two ways a context value causes unnecessary re-renders, and the fix for each? *(Part 8)*
17. What signal tells you that you have outgrown context for a piece of state? *(Part 8)*
18. Why is `memo` useless on a component receiving `style={{ padding: 8 }}`? *(Part 9)*
19. Describe the "children as props" optimisation and why it requires no memoisation. *(Part 9)*
20. With React Compiler 1.0 enabled, what memoisation still needs writing by hand, and why? *(Part 9)*

**Concurrency, data, internals**

21. When do you reach for `useTransition` versus `useDeferredValue`? *(Part 10)*
22. Why does `use(fetch(url))` inline in a component cause an infinite loop? *(Part 10)*
23. Sketch the race condition from switching a fetched `id` twice, and both fixes. *(Part 11)*
24. Why does a 500 response not reject a `fetch` promise? *(Part 11)*
25. Which four kinds of error do Error Boundaries *not* catch? *(Part 11)*
26. Why must hooks be called unconditionally — what is the actual storage mechanism? *(Part 13)*
27. What is the difference between the render phase and the commit phase, and which of your code runs in each? *(Part 13)*
28. Why is an interrupted render free? *(Part 13)*

[⬆ back to top](#table-of-contents)
