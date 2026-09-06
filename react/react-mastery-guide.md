# React — Mastery Guide (Overview)

The entry point for this repo's React material. This document holds the shared spec, the setup you need before running anything, the map of the focused sub-docs, the React 18 → 19 delta, and the reference apparatus (learning order, self-check, cross-doc cheat sheet).

The actual teaching lives in the sub-docs. Start here, then follow the [Suggested Learning Order](#7-suggested-learning-order).

---

## About This Document

This section is the canonical spec for **every doc in this set** — the sub-docs each carry a short pointer back here rather than repeating it.

- **Framing: curiosity-driven.** No upcoming task is driving this. The goal is durable, hands-on understanding of React itself. The docs are built as a *guided tutorial* — a sequence of runnable examples and **Try It** exercises you are meant to actually type into a running app, each stating what correct output looks like. Theory ("why is it designed this way") appears only where it is needed to explain a mechanism — the render/commit cycle, referential identity, the fiber tree — never as a standalone history section.
- **Confirmed scope: core client-side React only.** JSX and components, lists and conditional rendering, state and the render cycle, events and forms, effects and synchronisation, refs, lifting state / Context / reducers, data fetching and async UI, performance and referential identity (including React Compiler), concurrent features, custom hooks, and internals.
- **Dual-version: React 19 is the spine, React 18 is annotated.** Every example is written for React 19. Wherever the code you would *write* differs on React 18, a `> **On React 18**` callout sits directly under the example showing the 18 form — because that is the code you will meet in existing repos. The complete difference list lives in [Part 14: React 18 → 19](#5-part-14-react-18--19) below, and doubles as a migration reference. Everything not annotated behaves identically in both.
- **Deliberately excluded** (with pointers): React Router, TanStack Query, React Server Components / Server Actions / streaming SSR, styling, build-tool internals, testing, and React Native. See [Deliberately Not Covered](#6-deliberately-not-covered-and-where-to-look-instead).
- **Depth tiers.** Each Part runs **Beginner → Working Knowledge → Advanced → Mastery**, so you can stop at whichever level you need today. Tiers are collapsed where one would be filler, and each collapse is stated inline. Collapsing is expected behaviour, not a defect to fix on a later pass. The weight of this set sits in Working Knowledge and Advanced; Mastery concentrates in [Performance & Concurrency](./react-performance-concurrency.md) and [Internals & Custom Hooks](./react-internals-hooks.md).
- **Assumed background: modern JavaScript.** You are comfortable with closures, referential equality (`Object.is`), immutable updates, `map`/`filter`, destructuring, spread, and promises. If any of that is shaky, read [`javascript/javascript-functions-closures.md`](../javascript/javascript-functions-closures.md) and [`javascript/javascript-arrays-iteration.md`](../javascript/javascript-arrays-iteration.md) first — this set does not re-teach the language. No prior React is assumed.
- **Examples are TypeScript, with light types.** Types are kept minimal and idiomatic: enough to be realistic, never so much that the type noise obscures the React concept. For the type system itself see [`typescript/typescript-mastery-guide.md`](../typescript/typescript-mastery-guide.md).
- **Per-section ingredients**, used where they earn their place rather than forced into every subsection:
  - a short plain-language explanation of what a thing is and why it matters;
  - a **wrong vs. right** snippet pair wherever a gotcha has a common wrong form;
  - a **Try It** — a runnable exercise with the expected observable result — since this is a curiosity-driven set (production "Real Scenario" incidents are the goal-driven equivalent, used only where a bug genuinely only bites in production);
  - an **ASCII diagram** where the concept is spatial or structural — the render/commit pipeline, the UI tree and state positions, the fiber double buffer, the hook linked list.
- **Hooks-only teaching.** Class components and lifecycle methods appear exactly **once**, as a translation table in the [Cross-Doc Cheat Sheet](#9-cross-doc-cheat-sheet), so you can read older code and older Stack Overflow answers without being misled by them.
- **React Compiler stance: teach both, compiler-forward.** `memo` / `useMemo` / `useCallback` are taught properly, because you must read them in existing code and because referential identity is a real mechanism you need in your head regardless. [Performance & Concurrency](./react-performance-concurrency.md) then shows what the compiler auto-memoises and which cases still need you.
- **Sourcing rule.** [react.dev](https://react.dev) is the primary source and is cited inline at the specific claim it supports. Stack Overflow is used for real-world gotchas and how practitioners actually resolve them. GitHub and other reputable sites are supplementary only. Where a popular-but-outdated convention disagrees with the official docs, the official docs win and the doc says so explicitly.
- **Written against.** **React 19.2.7** (released June 1, 2026 — [React versions](https://react.dev/versions)), **React Compiler 1.x**, **Vite 8.2.2**, **`eslint-plugin-react-hooks` 7.1.1**, TypeScript 5.9+. React 18 annotations target **18.3.1**. Verified **September 6, 2026**.
- **To update this doc later.** *Preserve*: the curiosity-driven framing, the numbered Part structure with explicit tier subheadings, the wrong-vs-right pairs, the Try It exercises with expected results, the `> **On React 18**` callout convention, the TOC/back-to-top/anchor apparatus, and the per-Part cheat sheets. *Re-verify against current docs first*: the React patch version and anything in [Part 14](#5-part-14-react-18--19) (deprecations move — `forwardRef` and `<Context.Provider>` are deprecated but not yet removed, and that will change), the React Compiler version and whether it is still opt-in, the `eslint-plugin-react-hooks` major (it went 6 → 7 in 2026 and the flat-config shape moves with it), the Vite major, and the status of **View Transitions**, which had *not* shipped as of 19.2 and is the most likely thing in this set to become stale.

---

## Table of Contents

1. [How This Set Is Organised](#1-how-this-set-is-organised)
2. [The Sub-Docs](#2-the-sub-docs)
3. [Orientation & Setup](#3-orientation--setup)
4. [The One-Paragraph Mental Model](#4-the-one-paragraph-mental-model)
5. [Part 14: React 18 → 19](#5-part-14-react-18--19)
6. [Deliberately Not Covered (and where to look instead)](#6-deliberately-not-covered-and-where-to-look-instead)
7. [Suggested Learning Order](#7-suggested-learning-order)
8. [Quick Self-Check](#8-quick-self-check)
9. [Cross-Doc Cheat Sheet](#9-cross-doc-cheat-sheet)

---

## 1. How This Set Is Organised

React is 13 Parts of teaching material spread over 6 sub-docs, plus this overview. The split is not arbitrary — it follows the order in which the concepts actually become necessary:

```
  react-mastery-guide.md   ← you are here: spec, setup, map, 18→19 delta, reference
        │
        ├── react-foundations.md             Parts 1–3    what React renders
        ├── react-state-events.md            Parts 4–5    how it changes
        ├── react-effects-refs.md            Parts 6–7    how it reaches outside itself
        ├── react-shared-state.md            Parts 8–9    how components share
        ├── react-performance-concurrency.md Parts 10–11  how it stays fast
        └── react-internals-hooks.md         Parts 12–13  how it actually works
```

Each Part is self-contained enough to reopen on its own, and each ends with a quick-reference table. Within a Part, the tiers mean:

| Tier | What you get |
|---|---|
| **Beginner** | What it is, the minimal correct usage, nothing else |
| **Working Knowledge** | Everyday patterns, the API surface you actually reach for |
| **Advanced** | Edge cases, gotchas, performance and correctness implications |
| **Mastery** | Internals, powerful-but-rare capabilities, expert tradeoffs |

Where a tier would be filler, it is collapsed and the doc says so — e.g. "*Lists gets Beginner → Advanced only; its Mastery material is reconciliation, which belongs to Part 13.*"

[↑ Back to top](#table-of-contents)

---

## 2. The Sub-Docs

### [`react-foundations.md`](./react-foundations.md) — Parts 1–3

| Part | Covers |
|---|---|
| 1. Setup & Your First Render | `createRoot`, the root element, Strict Mode, what a component *is* |
| 2. JSX & Components | JSX as `createElement` sugar, props, children, composition, conditional rendering |
| 3. Lists & Keys | `map` over data, the rules of keys, why index keys bite, Fragments with keys |

**Read this if:** you have never written React, or you want the precise mental model of what JSX compiles to and why keys exist.

### [`react-state-events.md`](./react-state-events.md) — Parts 4–5

| Part | Covers |
|---|---|
| 4. State & the Render Cycle | `useState`, state as a snapshot, batching, updater functions, immutable updates, state structure, preserving/resetting state by position and `key` |
| 5. Events & Forms | Event handlers, the synthetic event system, controlled vs. uncontrolled inputs, forms, React 19 Actions (`useActionState`, `useOptimistic`, `useFormStatus`) |

**Read this if:** your state updates aren't taking effect, or you can't decide between controlled and uncontrolled inputs.

### [`react-effects-refs.md`](./react-effects-refs.md) — Parts 6–7

| Part | Covers |
|---|---|
| 6. Effects & Synchronisation | `useEffect` as synchronisation not lifecycle, dependencies, cleanup, Strict Mode double-invocation, *You Might Not Need an Effect*, `useEffectEvent`, `useLayoutEffect`, `useSyncExternalStore` |
| 7. Refs & the Escape Hatch | `useRef`, DOM refs, `ref` as a prop (19) vs `forwardRef` (18), ref callbacks and cleanup, `useImperativeHandle`, when a ref beats state |

**Read this if:** your effect runs twice, runs in a loop, or you are fighting the dependency array. This is where most React bugs live.

### [`react-shared-state.md`](./react-shared-state.md) — Parts 8–9

| Part | Covers |
|---|---|
| 8. Lifting, Context & Reducers | Lifting state up, `useReducer`, `createContext`/`useContext`, the reducer+context pattern, context performance, when to reach for a store |
| 9. Data Fetching & Async UI | Fetching in effects and its race conditions, `use()` + `<Suspense>`, error boundaries, `startTransition` for navigations, why you eventually want a library |

**Read this if:** you're prop-drilling, or your fetches are landing out of order.

### [`react-performance-concurrency.md`](./react-performance-concurrency.md) — Parts 10–11

| Part | Covers |
|---|---|
| 10. Performance & Referential Identity | Why re-renders happen, `memo`, `useMemo`, `useCallback`, the composition fixes that beat memoisation, React Compiler 1.x, measuring with Profiler and Performance Tracks |
| 11. Concurrent React | Transitions, `useDeferredValue`, `<Suspense>` mechanics, `<Activity>`, priority and interruption |

**Read this if:** something feels slow, or you want to know what the Compiler actually does to your code.

### [`react-internals-hooks.md`](./react-internals-hooks.md) — Parts 12–13

| Part | Covers |
|---|---|
| 12. Custom Hooks | Extracting logic, the rules of hooks and *why* they exist, composing hooks, what a custom hook does and doesn't share |
| 13. Under the Hood | Elements vs. fibers, the double-buffered tree, render vs. commit phases, the hook linked list, reconciliation and bailouts, the scheduler and lanes |

**Read this if:** you want the model that makes every earlier rule obvious rather than memorised.

[↑ Back to top](#table-of-contents)

---

## 3. Orientation & Setup

Every example in this set runs in one Vite + React + TypeScript app. Set it up once.

```bash
npm create vite@latest react-lab -- --template react-ts
cd react-lab
npm install
npm run dev
```

React's own docs list Vite first among build tools for [building a React app from scratch](https://react.dev/learn/build-a-react-app-from-scratch), with exactly this command. Note that react.dev's headline recommendation for *production* apps is a full-stack framework — we are deliberately not using one, because a framework would pull Server Components into scope, and this set is client-side React. See [Deliberately Not Covered](#6-deliberately-not-covered-and-where-to-look-instead).

> **Outdated convention to recognise:** `npx create-react-app my-app`. Create React App is no longer a recommended path and has not been for years — if a tutorial opens with it, its React advice is likely to be several years stale in other ways too. Vite, Parcel and Rsbuild are the build tools [react.dev currently documents](https://react.dev/learn/build-a-react-app-from-scratch).

Verify what you actually installed:

```bash
npm ls react react-dom vite
```

```
react-lab@0.0.0
├── react@19.2.7
├── react-dom@19.2.7
└── vite@8.2.2
```

### Enabling the React Compiler (optional, used in Part 10)

The Compiler is opt-in. You do not need it until [Part 10](./react-performance-concurrency.md), and you should do that Part's exercises **both with it off and on** to see what changes.

```bash
npm install --save-dev babel-plugin-react-compiler@latest
```

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react({
      babel: { plugins: [['babel-plugin-react-compiler', {}]] },
    }),
  ],
});
```

> **On React 18** — the Compiler supports React 17 and 18, but needs a runtime shim and an explicit target. Per [the target reference](https://react.dev/reference/react-compiler/target): install `react-compiler-runtime@latest` as a **dependency** (not a devDependency — it is needed at runtime), and set `target` as a *string*, matching your React major:
> ```bash
> npm install react-compiler-runtime@latest
> ```
> ```js
> // babel plugin options
> { target: '18' }   // or '17'
> ```

### The linter is not optional

```bash
npm install --save-dev eslint-plugin-react-hooks@latest
```

`eslint-plugin-react-hooks` (7.1.1 at time of writing) enforces the Rules of Hooks and the exhaustive-dependencies rule. In React, the linter is not a style tool — it catches a class of bug that is otherwise invisible until runtime. Several Parts in this set explain rules whose *only* practical enforcement is this plugin; [`useEffectEvent`'s call-site restriction](https://react.dev/reference/react/useEffectEvent) is explicitly one of them.

### The three files you will edit

```
react-lab/
├── index.html          ← contains <div id="root"></div>
├── src/
│   ├── main.tsx        ← createRoot(...).render(...)  — Part 1
│   ├── App.tsx         ← where you'll write nearly every exercise
│   └── components/     ← create this; extracted components go here
└── vite.config.ts
```

[↑ Back to top](#table-of-contents)

---

## 4. The One-Paragraph Mental Model

If you internalise one thing before reading anything else, make it this:

> **A React component is a pure function from props and state to a description of UI. You never update the screen; you produce a new description and React figures out the minimal DOM changes to match it.** State updates *request* a re-render, they do not perform one. Rendering means React *calls your components* — that is all it means. Committing is the separate, later step where the DOM is actually touched, and [React only changes the DOM nodes if there's a difference between renders](https://react.dev/learn/render-and-commit). Everything else in React — keys, the dependency array, referential identity, memoisation, concurrency — is a consequence of this one design, plus the fact that React needs to know *which* description belongs to *which* piece of state.

```
   ┌──────────┐   setState / initial mount
   │ TRIGGER  │──────────────────────────────┐
   └──────────┘                              │
                                             ▼
                                     ┌───────────────┐
                                     │    RENDER     │  React calls your components.
                                     │  (pure, can   │  Must be pure. Can be thrown
                                     │  be discarded)│  away, paused, or restarted.
                                     └───────┬───────┘
                                             │  produces a tree of elements
                                             ▼
                                     ┌───────────────┐
                                     │    COMMIT     │  React mutates the DOM —
                                     │ (synchronous, │  minimal ops only. Refs are
                                     │  never paused)│  attached. Layout effects run.
                                     └───────┬───────┘
                                             │
                                             ▼
                                     ┌───────────────┐
                                     │    PAINT      │  Browser draws.
                                     └───────┬───────┘
                                             │
                                             ▼
                                       useEffect runs
```

Three consequences worth stating now, because they explain most beginner confusion:

1. **Render can run more than once for one visible update**, and can be thrown away entirely. That is why [rendering must always be a pure calculation](https://react.dev/learn/render-and-commit), and why Strict Mode double-invokes your components in development to catch impurity.
2. **The state value inside a render is a frozen snapshot.** `count` in an event handler is the value from the render that created that handler — it never changes mid-render. This is the entire explanation for the classic "why did calling `setCount(count + 1)` three times only add one?" ([Part 4](./react-state-events.md)).
3. **Effects run after paint**, not during render. Anything you do in an effect is by definition a *reaction* to the DOM already existing.

[↑ Back to top](#table-of-contents)

---

## 5. Part 14: React 18 → 19

<a id="part-14"></a>


React 19.0 shipped on **December 5, 2024**; 19.2.7 is current as of June 2026 ([versions](https://react.dev/versions)). Adoption is real but not universal — the State of React 2025 survey (fielded Nov 2025 – Jan 2026) put roughly half of daily React users on 19 — so you will read both in the wild. This Part is the complete list of what differs. **Anything not on this list behaves identically in 18 and 19**, which is most of React.

### 14.1 New in 19 (nothing in 18 to compare)

| Feature | What it does | Covered in |
|---|---|---|
| `use(promise)` | Read a promise during render; the component suspends | [Part 9](./react-shared-state.md) |
| `use(context)` | Read context — legally inside a condition or loop | [Part 8](./react-shared-state.md) |
| Actions | `async` functions passed to `<form action>`; pending/error state handled for you | [Part 5](./react-state-events.md) |
| `useActionState` | State + pending flag + dispatch for an async action | [Part 5](./react-state-events.md) |
| `useOptimistic` | Show a provisional value while a mutation is in flight | [Part 5](./react-state-events.md) |
| `useFormStatus` | Read the enclosing `<form>`'s pending state from a child | [Part 5](./react-state-events.md) |
| Document metadata | `<title>`, `<meta>`, `<link>` rendered anywhere are hoisted into `<head>` | [Part 2](./react-foundations.md) |
| Stylesheet `precedence` | `<link rel="stylesheet" precedence="...">` ordering handled by React | — (noted only) |
| Preloading APIs | `preload`, `preinit`, `preconnect`, `prefetchDNS` from `react-dom` | — (noted only) |
| `useEffectEvent` | Non-reactive escape hatch inside Effects (stable in **19.2**) | [Part 6](./react-effects-refs.md) |
| `<Activity>` | Hide a subtree, keep its state, destroy its Effects (stable in **19.2**) | [Part 11](./react-performance-concurrency.md) |
| Performance Tracks | React-specific tracks in Chrome DevTools profiles (**19.2**) | [Part 10](./react-performance-concurrency.md) |

Sources: [React v19](https://react.dev/blog/2024/12/05/react-19) and [React 19.2](https://react.dev/blog/2025/10/01/react-19-2).

### 14.2 Same job, different code

This is the table that matters when *reading* an 18 codebase.

| Task | React 18 | React 19 |
|---|---|---|
| Forward a ref to a child | `forwardRef((props, ref) => …)` | `function Child({ ref })` — `ref` is a normal prop |
| Provide context | `<ThemeContext.Provider value={t}>` | `<ThemeContext value={t}>` |
| Clean up a ref callback | Callback is re-invoked with `null` on unmount | Return a cleanup function from the callback |
| Deferred value on first render | `useDeferredValue(v)` — no initial value | `useDeferredValue(v, initialValue)` |
| Report uncaught render errors | Errors re-thrown to `window` | `onUncaughtError` / `onCaughtError` on `createRoot` |
| Default prop values | `Component.defaultProps = {…}` | Destructuring defaults: `({ x = 1 })` |
| Runtime prop validation | `Component.propTypes = {…}` | TypeScript (propTypes **removed**) |

Both `forwardRef` and `<Context.Provider>` are **deprecated in 19, not removed** — 18-style code keeps working. Re-verify that when you next update this doc.

### 14.3 Removed in 19

Per the [upgrade guide](https://react.dev/blog/2024/04/25/react-19-upgrade-guide): `propTypes` and `defaultProps` on function components, string refs, legacy context (`contextTypes` / `getChildContext`), module pattern factories, `React.createFactory`, `ReactDOM.render`, `ReactDOM.hydrate`, `ReactDOM.unmountComponentAtNode`, `ReactDOM.findDOMNode`, `react-dom/test-utils` (`act` moved into the `react` package), and UMD builds.

### 14.4 If you have to do the upgrade

The documented path goes **through 18.3**, which adds deprecation warnings without changing behaviour:

```bash
npm install --save-exact react@18.3 react-dom@18.3
# fix every warning, then:
npm install --save-exact react@^19.0.0 react-dom@^19.0.0
```

Codemods, verbatim from the upgrade guide:

```bash
npx codemod@latest react/19/migration-recipe        # runs the whole set
npx types-react-codemod@latest preset-19 ./src      # TypeScript types
```

TypeScript-specific changes to expect: `useRef` now requires an argument, ref callbacks may not have implicit returns (use a block body), `ReactElement["props"]` defaults to `unknown` rather than `any`, and the `JSX` namespace moved from global to `React.JSX`.

### 14.5 What *doesn't* differ

Worth stating explicitly, because it is the bulk of what you are learning: JSX semantics, keys and reconciliation, `useState` and batching, state-as-snapshot, `useEffect` and its dependency rules, Strict Mode double-invocation, `useRef`, `useReducer`, `useContext` (the hook — only the provider *syntax* changed), `memo` / `useMemo` / `useCallback`, `useTransition`, `useDeferredValue`, `useSyncExternalStore`, `useId`, `<Suspense>`, error boundaries, custom hooks, and the entire fiber architecture in [Part 13](./react-internals-hooks.md). Learn those once; they are the same in both.

[↑ Back to top](#table-of-contents)

---

## 6. Deliberately Not Covered (and where to look instead)

Each of these is genuinely important. Each is omitted because half-covering it would be worse than pointing you at the real thing.

| Topic | Why it's out | Where to look |
|---|---|---|
| **React Server Components, Server Actions, streaming SSR** | Needs a framework to demonstrate at all; it is a different execution model, not an extra API | [react.dev: Server Components](https://react.dev/reference/rsc/server-components), then a Next.js doc |
| **React Router / TanStack Router** | Routing is a library decision, not a React feature; ages faster than React itself | React Router docs |
| **TanStack Query / SWR** | [Part 9](./react-shared-state.md) teaches the *primitives* and shows exactly which problems these libraries exist to solve, then stops | TanStack Query docs |
| **Styling** | Orthogonal to React | [`legacy/tailwindcss/tailwindcss-mastery-guide.md`](../legacy/tailwindcss/tailwindcss-mastery-guide.md) |
| **Build tooling internals** | Vite is setup here, not subject matter | [`legacy/vite/vite-mastery-guide.md`](../legacy/vite/vite-mastery-guide.md) |
| **Testing** | Deserves its own doc; React Testing Library has its own model | React Testing Library docs |
| **React Native** | Different renderer, different platform concerns | react.dev / Expo |
| **The JavaScript itself** | Assumed background | [`javascript/`](../javascript/javascript-mastery-guide.md) |
| **The TypeScript itself** | Types here are deliberately thin | [`typescript/`](../typescript/typescript-mastery-guide.md) |

[↑ Back to top](#table-of-contents)

---

## 7. Suggested Learning Order

**If you are new to React** — straight through, doing every Try It:

1. [Foundations](./react-foundations.md) Parts 1–3
2. [State & Events](./react-state-events.md) Parts 4–5 — stop at Working Knowledge on your first pass
3. [Effects & Refs](./react-effects-refs.md) Part 6 through Working Knowledge, then Part 7 Beginner
4. Build something small. Genuinely. Come back when it annoys you.
5. [Effects & Refs](./react-effects-refs.md) Advanced tiers — especially *You Might Not Need an Effect*
6. [Shared State](./react-shared-state.md) Parts 8–9
7. [Performance & Concurrency](./react-performance-concurrency.md) Parts 10–11
8. [Internals & Custom Hooks](./react-internals-hooks.md) Parts 12–13

**If you already write React and want the gaps filled:**

1. [Part 4 Advanced/Mastery](./react-state-events.md) — state as a snapshot, preserving/resetting by position
2. [Part 6 Advanced](./react-effects-refs.md) — the whole *You Might Not Need an Effect* section; this is where most React codebases are wrong
3. [Part 13](./react-internals-hooks.md) — then re-read Parts 4, 6 and 10, which will now feel obvious
4. [Part 10](./react-performance-concurrency.md) — the Compiler section
5. [Part 14](#5-part-14-react-18--19) above, if you maintain an 18 codebase

**If you are debugging something right now:**

| Symptom | Go to |
|---|---|
| State update "didn't work" | [Part 4 — state as a snapshot](./react-state-events.md) |
| Effect runs twice on mount | [Part 6 — Strict Mode](./react-effects-refs.md) |
| Effect runs forever | [Part 6 — dependencies](./react-effects-refs.md) |
| Input loses focus / component state resets | [Part 4 — preserving and resetting state](./react-state-events.md) |
| Stale value inside a closure | [Part 4 — snapshots](./react-state-events.md), [Part 6 — `useEffectEvent`](./react-effects-refs.md) |
| Fetch results arrive out of order | [Part 9 — race conditions](./react-shared-state.md) |
| Everything re-renders on every keystroke | [Part 10](./react-performance-concurrency.md) |
| List items behave weirdly when reordered | [Part 3 — keys](./react-foundations.md) |

[↑ Back to top](#table-of-contents)

---

## 8. Quick Self-Check

If you can answer these without looking, you have the set.

**Rendering**
1. What does "rendering" literally mean in React, and what does *not* happen during it?
2. Why must render be pure, and what does Strict Mode do to enforce it?
3. When does React skip touching a DOM node entirely?

**State**
4. Why does calling `setCount(count + 1)` three times in one handler add 1, not 3 — and what changes with `setCount(c => c + 1)`?
5. State is tied to what, exactly? (Not the component, not the JSX.)
6. Name two ways to reset a child's state when a prop changes, and say which one the docs prefer.

**Effects**
7. Give three cases from *You Might Not Need an Effect* and their replacements.
8. Why does the cleanup function exist, and what does the Strict Mode extra setup+cleanup cycle test?
9. What problem does `useEffectEvent` solve that a `useRef` workaround solves worse?

**Refs**
10. Why must you not read `ref.current` during render?
11. On React 19, how do you forward a ref? On 18?

**Sharing state**
12. What re-renders when a context value changes, and does `memo` stop it?
13. When is `useReducer` genuinely better than `useState`?

**Async**
14. Write the four-line cleanup pattern that fixes fetch race conditions.
15. What must be true of a promise passed to `use()`, and why?

**Performance**
16. Name three fixes that beat `memo` for the same problem.
17. What does React Compiler actually do to your component, and what does it still not handle?
18. What's the difference between `useTransition` and `useDeferredValue`?

**Internals**
19. What is a fiber, and how does it differ from an element?
20. Why must hooks be called unconditionally, in terms of the actual data structure?

[↑ Back to top](#table-of-contents)

---

## 9. Cross-Doc Cheat Sheet

### Every hook in this set

| Hook | One line | Part |
|---|---|---|
| `useState` | Component-local value that triggers a re-render when set | 4 |
| `useReducer` | Same, but transitions are described as actions in one function | 8 |
| `useContext` | Read the nearest provider's value | 8 |
| `use` | Read a promise (suspends) or context (conditionally legal) — **19+** | 8, 9 |
| `useEffect` | Synchronise with something outside React, after paint | 6 |
| `useLayoutEffect` | Same, but before paint — for measurement only | 6 |
| `useEffectEvent` | Non-reactive callback readable from an Effect — **19.2+** | 6 |
| `useRef` | A mutable box that does not trigger re-renders | 7 |
| `useImperativeHandle` | Choose what a parent's ref exposes | 7 |
| `useMemo` | Cache a value between renders | 10 |
| `useCallback` | Cache a function identity between renders | 10 |
| `useTransition` | Mark updates non-urgent, get an `isPending` flag | 11 |
| `useDeferredValue` | Let a value lag behind, to keep input responsive | 11 |
| `useSyncExternalStore` | Subscribe to a non-React store, tear-free | 6 |
| `useId` | Stable unique id, SSR-safe — for `htmlFor`/`aria-*` | 5 |
| `useActionState` | Async action + pending + result state — **19+** | 5 |
| `useOptimistic` | Provisional UI during a mutation — **19+** | 5 |
| `useFormStatus` | Enclosing form's pending state — **19+** | 5 |
| `useDebugValue` | Label a custom hook in DevTools | 12 |

### Class-component translation table

You will not write these. You will read them. This table appears **once** in the set.

| Class API | Hooks equivalent | Notes |
|---|---|---|
| `this.state` / `this.setState` | `useState` / `useReducer` | `setState` merged objects; `useState` **replaces** — this catches people |
| `componentDidMount` | `useEffect(fn, [])` | *Not* equivalent: the effect also re-runs if deps change, and Strict Mode runs it twice |
| `componentDidUpdate` | `useEffect(fn, [deps])` | Effects are per-value, not per-lifecycle |
| `componentWillUnmount` | cleanup return from `useEffect` | |
| `shouldComponentUpdate` | `memo(Component, arePropsEqual)` | |
| `PureComponent` | `memo(Component)` | |
| `getDerivedStateFromProps` | Adjust state during render, or use `key` | See [Part 4](./react-state-events.md) |
| `componentDidCatch` / `getDerivedStateFromError` | **Still a class.** No hook equivalent | Error boundaries remain class-only — [Part 9](./react-shared-state.md) |
| `this.refs.foo` (string ref) | `useRef` | String refs **removed in 19** |
| `createRef` | `useRef` | `createRef` is the class version |

The single most misleading habit carried over from class components is thinking of `useEffect` as `componentDidMount`. [Part 6](./react-effects-refs.md) is largely about unlearning that.

### The rules, in one place

1. **Render must be pure** — no mutation of props/state/module state, no side effects, no `ref.current` reads or writes.
2. **Hooks are called unconditionally**, at the top level of a component or another hook, in the same order every render.
3. **State is immutable** — always produce a new object/array, never mutate the existing one.
4. **Every reactive value used inside an Effect must be in its dependency array** — if that's painful, the Effect is usually wrong ([Part 6](./react-effects-refs.md)).
5. **Keys must be stable, unique among siblings, and derived from data** — not the index, never `Math.random()`.
6. **Don't call hooks from event handlers**, conditions, loops, or outside components.

### Version markers used in this set

| Marker | Meaning |
|---|---|
| `> **On React 18**` | The code above differs on 18; the callout shows the 18 form |
| **19+** | Does not exist in React 18 at all |
| **19.2+** | Added in a React 19 minor — check your patch version |

[↑ Back to top](#table-of-contents)

---

*React 19.2.7 · React Compiler 1.x · Vite 8.2.2 · eslint-plugin-react-hooks 7.1.1 · verified September 6, 2026*
