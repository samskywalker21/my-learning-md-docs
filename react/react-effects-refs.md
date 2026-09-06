# React — Effects & Refs (Parts 6–7)

> **Spec:** this doc follows the shared specification in [`react-mastery-guide.md`](./react-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Working Knowledge → Advanced → Mastery tiers, wrong-vs-right pairs, runnable **Try It** exercises, React 19 spine with `> **On React 18**` callouts.

Parts 1–5 stayed inside React. This doc is about the two escape hatches that reach outside it: **Effects**, for synchronising with systems React doesn't control, and **refs**, for values and DOM nodes React shouldn't manage.

Both are escape hatches. That framing matters — reaching for either when you didn't need to is the most common source of bugs in real React codebases, and [Advanced: You Might Not Need an Effect](#p6-advanced-2) is the most valuable section in this whole set.

---

## Table of Contents

- [Part 6: Effects & Synchronisation](#part-6-effects--synchronisation)
  - [Beginner — what an Effect is](#p6-beginner)
  - [Working Knowledge — dependencies and cleanup](#p6-working)
  - [Advanced — Strict Mode, and why your Effect runs twice](#p6-advanced)
  - [Advanced — You Might Not Need an Effect](#p6-advanced-2)
  - [Advanced — fixing the dependency array properly](#p6-advanced-3)
  - [Mastery — useLayoutEffect, useSyncExternalStore, and timing](#p6-mastery)
  - [Part 6 cheat sheet](#part-6-cheat-sheet)
- [Part 7: Refs & the Escape Hatch](#part-7-refs--the-escape-hatch)
  - [Beginner — a box that doesn't re-render](#p7-beginner)
  - [Working Knowledge — DOM refs and forwarding](#p7-working)
  - [Advanced — ref callbacks, cleanup, and the render rule](#p7-advanced)
  - [Mastery — useImperativeHandle and flushSync](#p7-mastery)
  - [Part 7 cheat sheet](#part-7-cheat-sheet)

---

## Part 6: Effects & Synchronisation

<a id="p6-beginner"></a>

### Beginner — what an Effect is

An Effect lets you run code **after** React has committed to the DOM, to synchronise your component with something outside React: a network connection, a subscription, a timer, a browser API, a non-React widget.

```tsx
import { useEffect, useState } from 'react';

function WindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    function handleResize() { setWidth(window.innerWidth); }
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return <p>{width}px</p>;
}
```

Three parts, every time:

1. **The setup function** — what to do.
2. **The cleanup function** it returns — how to undo it.
3. **The dependency array** — when to redo it.

The mental model that will serve you best is **not** "run this on mount". It is: *this Effect describes a synchronisation that must hold whenever these values are current. React will set it up, tear it down, and set it up again as those values change.* Everything else in this Part follows from that framing.

<a id="p6-working"></a>

### Working Knowledge — dependencies and cleanup

#### The dependency array

| You write | React runs the Effect |
|---|---|
| `useEffect(fn)` | After **every** commit |
| `useEffect(fn, [])` | After the first commit only |
| `useEffect(fn, [a, b])` | After the first commit, and whenever `a` or `b` changes |

Dependencies are compared with `Object.is`. "Changes" therefore means *referentially* changes — a new object or array literal counts as changed even if its contents are identical. That single fact causes most infinite-loop Effects.

The array is not a filter you choose. **Every reactive value your Effect reads must be in it** — props, state, and anything derived from them inside the component body. `eslint-plugin-react-hooks` computes the correct list for you; when the linter and your intuition disagree, the linter is right and your Effect is structured wrong ([see below](#p6-advanced-3)).

#### Cleanup

The cleanup function runs before every re-run, and on unmount:

```
  mount        deps change        deps change        unmount
    │               │                  │                │
  setup(a) ──────► cleanup(a)        cleanup(b)      cleanup(c)
                   setup(b) ───────► setup(c) ──────►
```

Cleanup must **mirror** setup. Connect → disconnect. Subscribe → unsubscribe. `setInterval` → `clearInterval`. `addEventListener` → `removeEventListener` (with the same function reference).

```tsx
// ✅ Symmetric: whatever setup started, cleanup stops
useEffect(() => {
  const connection = createConnection(serverUrl, roomId);
  connection.connect();
  return () => connection.disconnect();
}, [serverUrl, roomId]);
```

If you can't write the cleanup, that's usually a sign the thing in the Effect isn't a synchronisation at all.

<a id="p6-advanced"></a>

### Advanced — Strict Mode, and why your Effect runs twice

In development with `<StrictMode>`, [React runs an extra setup+cleanup cycle before the first real setup](https://react.dev/reference/react/useEffect):

> "When Strict Mode is on, React will **run one extra development-only setup+cleanup cycle** before the first real setup. This is a stress-test that ensures that your cleanup logic 'mirrors' your setup logic and that it stops or undoes whatever the setup is doing."

So on mount you see: `setup → cleanup → setup`. This is not a bug, it does not happen in production, and **the fix is never to suppress it**.

```tsx
// ❌ Wrong — the classic "fix": a ref guard to stop the second run
const didRun = useRef(false);
useEffect(() => {
  if (didRun.current) return;
  didRun.current = true;
  connection.connect();          // never disconnected — leaks on unmount too
}, []);

// ✅ Right — write the cleanup the double-run was asking for
useEffect(() => {
  const connection = createConnection(roomId);
  connection.connect();
  return () => connection.disconnect();
}, [roomId]);
```

The ref guard silences the symptom and keeps the actual bug: this component leaks a connection every time it unmounts, which in production shows up as a slowly degrading app, not as a doubled console log.

**Try It.** Add `console.log('connect', roomId)` and `console.log('disconnect', roomId)` to a chat-connection Effect. Expected result on mount: `connect A`, `disconnect A`, `connect A`. Change `roomId` to B: `disconnect A`, `connect B`. If your log shows `connect A` then `connect B` with no disconnect between, your cleanup is missing and you have a leak.

There is a legitimate exception — genuine once-per-app-load work, which the docs address directly:

```tsx
// ✅ For true app initialisation, run it outside React entirely
if (typeof window !== 'undefined') {
  checkAuthToken();
  loadDataFromLocalStorage();
}
```

<a id="p6-advanced-2"></a>

### Advanced — You Might Not Need an Effect

This is the section to actually read twice. [The docs' own page](https://react.dev/learn/you-might-not-need-an-effect) enumerates the cases; the underlying test is one question:

> "When you're not sure whether some code should be in an Effect or in an event handler, ask yourself *why* this code needs to run. Use Effects only for code that should run *because* the component was displayed to the user."

If it should run because *the user did something*, it belongs in the handler. If it should run because *some value changed*, it usually belongs in render. Effects are only for *"because this is now on screen and must stay in sync with something external."*

#### 1. Transforming data for rendering → just calculate it

```tsx
// ❌ Wrong — an extra render pass, and a chance to be stale
const [fullName, setFullName] = useState('');
useEffect(() => {
  setFullName(firstName + ' ' + lastName);
}, [firstName, lastName]);

// ✅ Right
const fullName = firstName + ' ' + lastName;
```

#### 2. Expensive calculations → `useMemo`, not state + Effect

```tsx
// ❌ Wrong
const [visibleTodos, setVisibleTodos] = useState([]);
useEffect(() => {
  setVisibleTodos(getFilteredTodos(todos, filter));
}, [todos, filter]);

// ✅ Right
const visibleTodos = useMemo(() => getFilteredTodos(todos, filter), [todos, filter]);
```

And note the docs' own footnote: [React Compiler](https://react.dev/learn/react-compiler) can memoise this for you, removing the `useMemo` too ([Part 10](./react-performance-concurrency.md)).

#### 3. Responding to a user event → put it in the handler

```tsx
// ❌ Wrong — fires whenever `product` changes for any reason
useEffect(() => {
  if (product.isInCart) {
    showNotification(`Added ${product.name} to the shopping cart!`);
  }
}, [product]);

// ✅ Right
function handleBuyClick() {
  addToCart(product);
  showNotification(`Added ${product.name} to the shopping cart!`);
}
```

The docs' reasoning is worth internalising: *"You know exactly what happened [in the event handler]. By the time an Effect runs, you don't know what the user did."*

#### 4. Resetting state when a prop changes → `key`

```tsx
// ❌ Wrong
useEffect(() => { setComment(''); }, [userId]);

// ✅ Right
<Profile userId={userId} key={userId} />
```

Covered in full in [Part 4](./react-state-events.md#p4-advanced-2).

#### 5. Chains of Effects that each set state → do it in one handler

```tsx
// ❌ Wrong — three extra render passes, fragile ordering
useEffect(() => { setGoldCardCount(c => c + 1); }, [card]);
useEffect(() => { setRound(r => r + 1); }, [goldCardCount]);
useEffect(() => { setIsGameOver(true); }, [round]);

// ✅ Right — one handler computes the whole transition
function handlePlaceCard(nextCard) {
  setCard(nextCard);
  if (nextCard.gold) {
    if (goldCardCount < 3) {
      setGoldCardCount(goldCardCount + 1);
    } else {
      setGoldCardCount(0);
      setRound(round + 1);
    }
  }
}
```

#### 6. Notifying a parent → call the callback in the same handler

```tsx
// ❌ Wrong — the parent learns one render late
useEffect(() => { onChange(isOn); }, [isOn, onChange]);

// ✅ Right — React batches both updates into one render pass
function updateToggle(nextIsOn: boolean) {
  setIsOn(nextIsOn);
  onChange(nextIsOn);
}
```

#### 7. Subscribing to an external store → `useSyncExternalStore`

```tsx
// ❌ Wrong — misses changes between render and effect; tears under concurrency
useEffect(() => {
  window.addEventListener('online', updateState);
  return () => window.removeEventListener('online', updateState);
}, []);

// ✅ Right
const isOnline = useSyncExternalStore(
  subscribe,
  () => navigator.onLine,    // client snapshot
  () => true,                // server snapshot
);
```

#### 8. Fetching data → an Effect is *allowed*, but needs cleanup

Fetching is the one case on this list where an Effect is legitimately the primitive. It just isn't sufficient on its own — see [Part 9](./react-shared-state.md), and note the docs' own caveat that *"modern frameworks provide more efficient built-in data fetching mechanisms than writing Effects directly in your components."*

#### The summary table

| Instead of an Effect that… | Do this |
|---|---|
| Sets state from other state/props | Calculate during render |
| Sets state from an expensive calculation | `useMemo` (or let the Compiler do it) |
| Reacts to a click/submit | Put the code in the handler |
| Clears state when a prop changes | `key` |
| Adjusts one piece of state on prop change | Conditional `setState` during render ([Part 4](./react-state-events.md#p4-advanced-2)) |
| Triggers another Effect | Compute the whole transition in one place |
| Tells the parent about a state change | Call the prop in the same handler |
| Subscribes to an external store | `useSyncExternalStore` |
| Runs once at app startup | Module-level code outside the component |

What legitimately remains: connecting to a chat server, `setInterval`, non-React widget lifecycles (a map, a chart, a video player), imperative DOM APIs like `dialog.showModal()`, analytics page views, and data fetching.

<a id="p6-advanced-3"></a>

### Advanced — fixing the dependency array properly

You cannot lie to the dependency array. You can only change what the Effect reads.

```tsx
// ❌ Wrong — suppressing the linter
useEffect(() => {
  connect(options);
}, []);  // eslint-disable-line react-hooks/exhaustive-deps
```

That comment is a promise that `options` will never change, and it will be broken by someone later. Four honest fixes:

**Move the value inside the Effect.** If the object is only used by the Effect, create it there:

```tsx
// ❌ Wrong — a new object every render → the Effect re-runs every render
const options = { serverUrl, roomId };
useEffect(() => {
  const connection = createConnection(options);
  connection.connect();
  return () => connection.disconnect();
}, [options]);

// ✅ Right
useEffect(() => {
  const options = { serverUrl, roomId };
  const connection = createConnection(options);
  connection.connect();
  return () => connection.disconnect();
}, [serverUrl, roomId]);
```

**Use the updater form** so the Effect doesn't need to read the state it's setting:

```tsx
// ❌ Wrong — re-creates the interval on every tick
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, [count]);

// ✅ Right — no dependency on count at all
useEffect(() => {
  const id = setInterval(() => setCount(c => c + 1), 1000);
  return () => clearInterval(id);
}, []);
```

**Move a function inside**, or out of the component entirely if it doesn't read props/state.

**Use `useEffectEvent`** (19.2+) for the genuinely non-reactive part.

#### `useEffectEvent` — 19.2+

Some values are read by an Effect without being reasons to *re-run* it. The canonical case: reconnect when the room changes, but read the current theme when showing the notification.

```tsx
// ❌ Wrong — reconnects the chat every time the theme toggles
function ChatRoom({ roomId, theme }: Props) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.on('connected', () => showNotification('Connected!', theme));
    connection.connect();
    return () => connection.disconnect();
  }, [roomId, theme]);   // theme is a real dependency — and that's the problem
}

// ✅ Right — theme is read, but is not reactive
import { useEffectEvent, useEffect } from 'react';

function ChatRoom({ roomId, theme }: Props) {
  const onConnected = useEffectEvent(() => {
    showNotification('Connected!', theme);     // always the latest theme
  });

  useEffect(() => {
    const connection = createConnection(roomId);
    connection.on('connected', () => onConnected());
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);          // onConnected is deliberately NOT a dependency
}
```

The [caveats](https://react.dev/reference/react/useEffectEvent) are strict, and the linter enforces them:

> - **Effect Events can only be called from inside Effects or other Effect Events.** Do not call them during rendering or pass them to other components or Hooks.
> - **Do not use `useEffectEvent` to avoid specifying dependencies** in your Effect's dependency array. This hides bugs and makes your code harder to understand. Only use it for logic that is genuinely an event fired from Effects.
> - **Effect Event functions do not have a stable identity.** Their identity intentionally changes on every render.

That last one is a deliberate trap: if you wrongly pass one around as a dependency, your Effect will re-run every render and the mistake becomes loud immediately.

> **On React 18** — `useEffectEvent` does not exist. The pre-19.2 workaround is a ref updated in an Effect:
> ```tsx
> const themeRef = useRef(theme);
> useEffect(() => { themeRef.current = theme; });
> // then read themeRef.current inside the Effect
> ```
> It works, and it is worse: nothing stops you reading the ref during render, nothing stops you passing it around, and the linter cannot tell that you meant this. If you're on 19.2+, replace these.

<a id="p6-mastery"></a>

### Mastery — useLayoutEffect, useSyncExternalStore, and timing

#### The full commit ordering

```
  render (your components run)
      │
      ▼
  DOM mutations applied
      │
      ▼
  useLayoutEffect cleanup + setup     ← synchronous, blocks paint
      │                                  refs are attached by now
      ▼
  browser PAINTS
      │
      ▼
  useEffect cleanup + setup           ← asynchronous, after paint
```

Children's effects run before parents' — React works bottom-up, so a parent's Effect can assume its children have already synchronised.

#### `useLayoutEffect`

Identical API to `useEffect`, but runs **before the browser paints**. Use it only when you must measure the DOM and change something based on the measurement, and a flash of the wrong position would be visible.

```tsx
function Tooltip({ children }: { children: React.ReactNode }) {
  const ref = useRef<HTMLDivElement>(null);
  const [height, setHeight] = useState(0);

  useLayoutEffect(() => {
    setHeight(ref.current!.getBoundingClientRect().height);   // measure, then flip
  }, []);

  return <div ref={ref} style={{ top: height > 100 ? 0 : -height }}>{children}</div>;
}
```

Because it blocks paint, it is a performance cost by definition. The rule of thumb: `useEffect` unless you can *see* the problem.

#### `useSyncExternalStore`

The correct way to read from any mutable source outside React — a global store, `navigator.onLine`, `matchMedia`, a third-party observable.

```tsx
function subscribe(callback: () => void) {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function useOnlineStatus() {
  return useSyncExternalStore(
    subscribe,
    () => navigator.onLine,   // getSnapshot (client)
    () => true,               // getServerSnapshot (SSR)
  );
}
```

Its reason to exist is **tearing**: under concurrent rendering, React can pause mid-render, and an external value read at two different moments in one render could differ, producing a UI showing two versions of the same data. `useSyncExternalStore` forces a synchronous, consistent read. Every serious state library (Zustand, Redux, Jotai) is built on it.

One caveat that bites: `getSnapshot` must return a **cached** value, not a fresh object. Returning `{ x: store.x }` creates a new object each call, React sees an infinite change, and you get "The result of getSnapshot should be cached to avoid an infinite loop."

<a id="part-6-cheat-sheet"></a>

### Part 6 cheat sheet

| Task | Code |
|---|---|
| Run after every render | `useEffect(fn)` |
| Run once after mount | `useEffect(fn, [])` |
| Re-run when values change | `useEffect(fn, [a, b])` |
| Undo the Effect | `return () => …` from setup |
| Before paint (measurement only) | `useLayoutEffect` |
| Read a non-React store | `useSyncExternalStore(subscribe, get, getServer)` |
| Read latest value without re-running | `useEffectEvent` (19.2+) |

| Symptom | Cause | Fix |
|---|---|---|
| Runs twice on mount | Strict Mode | Write the cleanup — don't guard |
| Runs on every render | Missing deps array, or an object/function dep | Move the value inside, or memoise |
| Infinite loop | Sets state that is also a dependency | Updater form, or move out of the Effect |
| Stale value inside | Suppressed linter | Add the dep, or `useEffectEvent` |
| Leaks connections/timers | Missing cleanup | Mirror setup exactly |
| "getSnapshot should be cached" | New object from `getSnapshot` | Return a cached reference |

| Ask before writing an Effect | If yes → |
|---|---|
| Can this be calculated during render? | Calculate it |
| Does this happen because the user did something? | Event handler |
| Am I resetting state because a prop changed? | `key` |
| Am I reading an external mutable store? | `useSyncExternalStore` |
| Am I connecting to something outside React? | ✅ Effect is right |

[↑ Back to top](#table-of-contents)

---

## Part 7: Refs & the Escape Hatch

<a id="p7-beginner"></a>

### Beginner — a box that doesn't re-render

`useRef` gives you a mutable box that survives re-renders and, crucially, **does not trigger one when it changes**.

```tsx
const ref = useRef(0);
ref.current = 5;        // no re-render
```

State versus ref, as a decision table:

| | `useState` | `useRef` |
|---|---|---|
| Triggers re-render on change | Yes | No |
| Value during render | Snapshot, immutable | Whatever it currently is |
| Safe to mutate | No | Yes |
| Safe to read during render | Yes | **No** |
| Use for | Anything the UI displays | Timer ids, DOM nodes, previous values, "has this fired" flags |

The question to ask: *does the screen need to change when this changes?* If yes, state. If no, ref.

```tsx
// ✅ A timer id is a perfect ref: the UI doesn't display it
function Stopwatch() {
  const [now, setNow] = useState(0);
  const intervalRef = useRef<number | null>(null);

  function start() {
    intervalRef.current = window.setInterval(() => setNow(Date.now()), 10);
  }
  function stop() {
    if (intervalRef.current !== null) clearInterval(intervalRef.current);
  }
  …
}
```

<a id="p7-working"></a>

### Working Knowledge — DOM refs and forwarding

Pass a ref to a DOM element and React puts the node in `.current` at commit time.

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

Typed as `useRef<HTMLInputElement>(null)`, `.current` is `HTMLInputElement | null` — hence the `?.`. It is `null` before the first commit and after unmount.

The legitimate uses are the imperative browser APIs React has no declarative equivalent for: `focus()`, `scrollIntoView()`, `select()`, `showModal()`, `play()`/`pause()`, `getBoundingClientRect()`, and canvas contexts.

#### Forwarding a ref to your own component

Refs only attach to DOM elements. To let a parent reach into a child component, the child must pass the ref down.

```tsx
// ✅ React 19 — ref is an ordinary prop
function TextField({ ref, ...props }: React.ComponentProps<'input'>) {
  return <input ref={ref} {...props} />;
}

// parent
const ref = useRef<HTMLInputElement>(null);
<TextField ref={ref} />
```

> **On React 18** — `ref` is not a prop; it is intercepted by React and never reaches your function. You need `forwardRef`:
> ```tsx
> import { forwardRef } from 'react';
>
> const TextField = forwardRef<HTMLInputElement, Props>((props, ref) => (
>   <input ref={ref} {...props} />
> ));
> TextField.displayName = 'TextField';
> ```
> `forwardRef` still works in 19 but is **deprecated**, and there's a codemod in the [migration recipe](https://react.dev/blog/2024/04/25/react-19-upgrade-guide). Expect to read a great deal of `forwardRef` in existing component libraries.

#### A list of refs

You can't call `useRef` in a loop. Use one ref holding a `Map`:

```tsx
function ItemList({ items }: { items: Item[] }) {
  const nodesRef = useRef<Map<string, HTMLLIElement>>(new Map());

  function scrollTo(id: string) {
    nodesRef.current.get(id)?.scrollIntoView({ behavior: 'smooth' });
  }

  return (
    <ul>
      {items.map(item => (
        <li
          key={item.id}
          ref={node => {
            const map = nodesRef.current;
            map.set(item.id, node!);
            return () => map.delete(item.id);   // 19: cleanup function
          }}
        >
          {item.text}
        </li>
      ))}
    </ul>
  );
}
```

<a id="p7-advanced"></a>

### Advanced — ref callbacks, cleanup, and the render rule

#### Ref callbacks

Instead of a ref object, you can pass a function. React calls it with the node when attaching. In React 19, it may **return a cleanup function**:

```tsx
<input ref={node => {
  node?.focus();
  return () => { /* runs on detach */ };
}} />
```

> **On React 18** — returning a cleanup is not supported. Instead React calls the same callback again with `null` on unmount, which is why 18-era code is full of `if (node) { … } else { … }` branches. React 19 deprecates the `null` call in favour of the cleanup return.
>
> There is a TypeScript trap here on 19: a ref callback may no longer have an **implicit return**, because the return value now means something.
> ```tsx
> // ❌ Type error on 19 — implicitly returns the assignment's value
> <div ref={node => (myRef.current = node)} />
>
> // ✅ Block body
> <div ref={node => { myRef.current = node; }} />
> ```
> The `npx types-react-codemod@latest preset-19` codemod fixes these.

#### Never read or write a ref during render

From the [useRef caveats](https://react.dev/reference/react/useRef):

> - Do not write *or read* `ref.current` during rendering, except for initialization. This makes your component's behavior unpredictable.
> - When you change `ref.current`, React does not re-render your component. React is not aware of when you change it because a ref is a plain JavaScript object.
> - In Strict Mode, React will call your component function twice… Each ref object will be created twice, but one version will be discarded.

```tsx
// ❌ Wrong — reading during render
function Bad({ x }: { x: number }) {
  return <p>{countRef.current}</p>;   // renders a value React doesn't track
}

// ✅ Right — read in a handler or an Effect
function Good() {
  useEffect(() => { countRef.current += 1; });
  return <button onClick={() => alert(countRef.current)}>Show</button>;
}
```

The reason is concurrency, not style: React may render your component and discard the result. If that render mutated a ref, the mutation happened but the UI it was meant to accompany never existed.

#### Don't fight React for the DOM

A ref lets you *read* and call methods on nodes React owns. It does not let you restructure them.

```tsx
// ❌ Wrong — React doesn't know this node is gone; it will crash on the next update
<div ref={r => r?.remove()} />

// ✅ Right — let React own its own nodes
{visible && <div />}
```

Manipulating DOM that React manages is safe only for nodes React has no children in — a container you hand entirely to a third-party library, for example.

<a id="p7-mastery"></a>

### Mastery — useImperativeHandle and flushSync

#### `useImperativeHandle`

Sometimes you want a parent to have *some* imperative control without exposing the raw DOM node. `useImperativeHandle` lets you choose what `.current` is:

```tsx
type VideoHandle = { play: () => void; pause: () => void };

function Video({ ref, src }: { ref: React.Ref<VideoHandle>; src: string }) {
  const videoRef = useRef<HTMLVideoElement>(null);

  useImperativeHandle(ref, () => ({
    play: () => videoRef.current?.play(),
    pause: () => videoRef.current?.pause(),
  }), []);

  return <video ref={videoRef} src={src} />;
}
```

The parent now gets exactly `play` and `pause` — it cannot reach in and change `style` or remove the node. Use this sparingly; a component with an imperative API is harder to compose than one driven by props.

> **On React 18** — same hook, but wrapped in `forwardRef`: `forwardRef<VideoHandle, Props>((props, ref) => { useImperativeHandle(ref, …) })`.

#### `flushSync`

State updates are asynchronous with respect to the DOM. If you need the DOM updated *before* the next line runs — typically to scroll to an item you just added — `flushSync` forces the commit:

```tsx
import { flushSync } from 'react-dom';

function handleAdd(text: string) {
  flushSync(() => {
    setTodos([...todos, { id: crypto.randomUUID(), text }]);
  });
  // by here, the new <li> exists in the DOM
  listRef.current?.lastElementChild?.scrollIntoView();
}
```

It defeats batching and concurrency for that update, so it is a real performance cost. It is the correct tool for exactly this "act on the just-committed DOM" case and close to nothing else.

#### When a ref genuinely beats state

| Situation | Why a ref |
|---|---|
| Storing a timer/animation-frame id | Nothing displays it |
| Caching a value across renders (a previous prop, a WebSocket) | Changing it shouldn't paint |
| "Has this already run" flags in an integration boundary | Not UI |
| High-frequency values you sample rather than render (mouse position during a drag) | State would render on every mousemove |
| Holding a third-party instance (map, chart, editor) | The library owns its own rendering |

And the anti-pattern that will get you: using a ref to hold something the UI *does* display, then wondering why the screen doesn't update. If you catch yourself calling `forceUpdate`-style tricks to make a ref show up, it was state all along.

<a id="part-7-cheat-sheet"></a>

### Part 7 cheat sheet

| Task | Code |
|---|---|
| Mutable non-rendering value | `const r = useRef(initial)` |
| DOM node | `const r = useRef<HTMLInputElement>(null)` + `<input ref={r} />` |
| Use the node | `r.current?.focus()` |
| Forward to your component (19) | `function C({ ref })` — plain prop |
| Forward to your component (18) | `forwardRef((props, ref) => …)` |
| Many nodes | one ref holding a `Map` |
| Run code on attach/detach (19) | ref callback returning a cleanup |
| Expose a limited API | `useImperativeHandle(ref, () => ({ … }), [])` |
| Force a synchronous commit | `flushSync(() => setX(…))` |

| Rule | |
|---|---|
| Read/write during render | ❌ Never (except lazy init) |
| Read/write in handlers and Effects | ✅ Always fine |
| `.current` before first commit | `null` |
| Does changing it re-render? | No |
| Restructuring React-owned DOM | ❌ |

[↑ Back to top](#table-of-contents)

---

**Previous:** [Parts 4–5 — State & Events](./react-state-events.md) · **Next:** [Parts 8–9 — Shared State & Data](./react-shared-state.md) · **Up:** [Overview](./react-mastery-guide.md)

*React 19.2.7 · verified September 6, 2026*
