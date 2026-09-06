# React — Performance & Concurrency (Parts 10–11)

> **Spec:** this doc follows the shared specification in [`react-mastery-guide.md`](./react-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Working Knowledge → Advanced → Mastery tiers, wrong-vs-right pairs, runnable **Try It** exercises, React 19 spine with `> **On React 18**` callouts.

Part 10 is about doing **less work**. Part 11 is about doing the work **in a better order**. They are genuinely different tools and people reach for the wrong one constantly: memoisation cannot help a render that is unavoidably slow, and a transition cannot help a render that shouldn't be happening at all.

---

## Table of Contents

- [Part 10: Performance & Referential Identity](#part-10-performance--referential-identity)
  - [Beginner — measure before you memoise](#p10-beginner)
  - [Working Knowledge — memo, useMemo, useCallback](#p10-working)
  - [Advanced — the fixes that beat memoisation](#p10-advanced)
  - [Mastery — React Compiler](#p10-mastery)
  - [Part 10 cheat sheet](#part-10-cheat-sheet)
- [Part 11: Concurrent React](#part-11-concurrent-react)
  - [Beginner — urgent vs. non-urgent](#p11-beginner)
  - [Working Knowledge — useTransition and useDeferredValue](#p11-working)
  - [Advanced — Suspense mechanics and Activity](#p11-advanced)
  - [Mastery — what "interruptible" actually means](#p11-mastery)
  - [Part 11 cheat sheet](#part-11-cheat-sheet)

---

## Part 10: Performance & Referential Identity

<a id="p10-beginner"></a>

### Beginner — measure before you memoise

Start from the right default: **a re-render is not a bug.** React re-rendering a component means calling a function and diffing its output. For most components that is microseconds, and the DOM is only touched where output actually differs ([Part 1](./react-foundations.md#p1-advanced)).

What causes a component to re-render:

1. Its own state changed.
2. Its parent re-rendered (regardless of whether its props changed).
3. A context it consumes changed.

Point 2 surprises people, and it's the reason "my component re-renders even though its props didn't change" is a top React search. By default React re-renders the whole subtree; that is cheap and correct.

The workflow is always the same, and the first two steps are non-negotiable:

```
  1. Notice something is actually slow        (not "might be")
  2. Profile it in a PRODUCTION build         (dev builds are 2–10× slower and lie)
  3. Find the specific expensive thing
  4. Fix the structure if you can
  5. Memoise only if you can't
  6. Re-measure
```

Two tools:

- **React DevTools Profiler** — record an interaction, see which components rendered, how long each took, and *why* it rendered (enable "Record why each component rendered" in the profiler settings).
- **Performance Tracks** (19.2+) — React-specific tracks in a Chrome DevTools performance profile: a **Scheduler** track showing work and its priority, and a **Components** track showing render and effect timing, interleaved with everything else the browser was doing. This is the better tool when the question is "is React even the problem?"

<a id="p10-working"></a>

### Working Knowledge — memo, useMemo, useCallback

Three APIs, three different jobs. They are constantly confused.

| API | Caches | Prevents |
|---|---|---|
| `memo(Component)` | The component's rendered output | Re-rendering when props are shallow-equal |
| `useMemo(fn, deps)` | A **value** | Recomputing an expensive value |
| `useCallback(fn, deps)` | A **function's identity** | Breaking a `memo`'d child or an Effect dep |

#### `memo`

```tsx
const ExpensiveList = memo(function ExpensiveList({ items }: { items: Item[] }) {
  …
});
```

Props are compared shallowly with `Object.is`. Which is why `memo` alone so often does nothing:

```tsx
// ❌ memo is defeated — a new array and a new function every parent render
function Parent() {
  const [text, setText] = useState('');
  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <ExpensiveList items={data.filter(d => d.active)} onSelect={id => pick(id)} />
    </>
  );
}

// ✅ memo can work — both props are referentially stable
function Parent() {
  const [text, setText] = useState('');
  const items = useMemo(() => data.filter(d => d.active), []);
  const onSelect = useCallback((id: string) => pick(id), []);
  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <ExpensiveList items={items} onSelect={onSelect} />
    </>
  );
}
```

**`memo` is all-or-nothing across every prop.** One unstable prop and the whole optimisation is dead — which is why memoisation tends to spread virally through a codebase once you start.

And two things `memo` explicitly does **not** prevent:

- re-renders caused by the component's own state changing;
- re-renders caused by a context it consumes changing ([Part 8](./react-shared-state.md#p8-advanced)).

#### `useMemo`

For genuinely expensive calculations. The docs give a concrete threshold for "expensive":

```tsx
console.time('filter array');
const visibleTodos = useMemo(() => filterTodos(todos, tab), [todos, tab]);
console.timeEnd('filter array');
```

> **General guideline:** If logging time adds up to 1ms or more, memoization may help.

Below that, `useMemo` costs more than it saves — it allocates, stores, and compares dependencies on every render.

Critically, `useMemo` is a **performance hint, not a guarantee**:

> React may throw away the cached value if there's a reason (component suspends, file edited in development, future virtualization features).

Never rely on it for correctness — don't cache something whose recreation would be a *bug*.

#### `useCallback`

`useMemo` for functions. `useCallback(fn, deps)` is exactly `useMemo(() => fn, deps)`.

It is useful in exactly three situations:
1. The function is a prop of a `memo`'d component.
2. The function is a dependency of an Effect.
3. The function is part of a context value.

Everywhere else it is noise. A `useCallback` on a handler passed to a plain `<button>` accomplishes nothing at all — DOM elements don't care about function identity.

<a id="p10-advanced"></a>

### Advanced — the fixes that beat memoisation

The docs list these *before* recommending `memo`, and they're right to: each removes the problem instead of caching around it.

#### 1. Move state down

If only part of a component uses a fast-changing value, extract that part.

```tsx
// ❌ Wrong — every keystroke re-renders the whole page including <HugeTree/>
function Page() {
  const [text, setText] = useState('');
  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <HugeTree />
    </>
  );
}

// ✅ Right — the state now lives in a leaf; HugeTree never re-renders
function SearchInput() {
  const [text, setText] = useState('');
  return <input value={text} onChange={e => setText(e.target.value)} />;
}

function Page() {
  return (
    <>
      <SearchInput />
      <HugeTree />
    </>
  );
}
```

#### 2. Pass JSX as children ("lift content up")

This is the one worth really understanding, because it looks like it shouldn't work.

```tsx
// ❌ Wrong — ColorPicker's state re-renders everything it renders
function ColorPicker() {
  const [color, setColor] = useState('lightcoral');
  return (
    <div style={{ background: color }}>
      <input value={color} onChange={e => setColor(e.target.value)} />
      <ExpensiveTree />
    </div>
  );
}

// ✅ Right — ExpensiveTree is created by App and passed through
function ColorPicker({ children }: { children: React.ReactNode }) {
  const [color, setColor] = useState('lightcoral');
  return (
    <div style={{ background: color }}>
      <input value={color} onChange={e => setColor(e.target.value)} />
      {children}
    </div>
  );
}

function App() {
  return (
    <ColorPicker>
      <ExpensiveTree />
    </ColorPicker>
  );
}
```

Why it works: `children` is a prop, and `App` didn't re-render, so it's the **same element object** as last time ([Part 2](./react-foundations.md#p2-advanced) — elements are plain values). React sees an identical element at the same position and bails out of re-rendering that subtree. No `memo`, no `useMemo`, no dependency arrays. This is the mechanism behind every "slot"-style layout component.

#### 3. Pass primitives instead of objects

```tsx
// ❌ person is a new object → memo defeated
<Profile person={{ name, age }} />

// ✅ primitives compare by value
<Profile name={name} age={age} />

// ✅ or compute the derived boolean in the parent
<CallToAction hasGroups={person.groups !== null} />
```

#### 4. Remove unnecessary Effects

An Effect that sets state causes a second render pass by definition. [Part 6](./react-effects-refs.md#p6-advanced-2) is, among other things, a performance chapter.

#### 5. Virtualise long lists

Ten thousand rows is not a memoisation problem. Render only what's visible (`@tanstack/react-virtual` or similar). No amount of `memo` beats not creating 9,950 elements.

#### The custom comparison escape hatch

```tsx
const Chart = memo(function Chart({ dataPoints }: Props) { … }, arePropsEqual);

function arePropsEqual(oldProps: Props, newProps: Props) {
  return oldProps.dataPoints.length === newProps.dataPoints.length &&
    oldProps.dataPoints.every((p, i) => p.x === newProps.dataPoints[i].x);
}
```

Deep equality here is usually a trap: comparing is work too, and you must compare *every* prop including functions, which you can't meaningfully compare at all.

<a id="p10-mastery"></a>

### Mastery — React Compiler

React Compiler is a build-time tool that inserts memoisation for you, based on an actual analysis of what your component reads and produces. Version 1.0 shipped in October 2025; it is **opt-in** and works with React 19, and with 17/18 via `target` ([setup](./react-mastery-guide.md#3-orientation--setup)).

#### What it does

Roughly: it rewrites your component to allocate a memo cache and reuse values whose inputs didn't change. Conceptually,

```tsx
function TodoList({ todos, tab, theme }) {
  const visible = filterTodos(todos, tab);
  return <div className={theme}><List items={visible} /></div>;
}
```

becomes something equivalent to hand-written memoisation of `visible`, of the `<List>` element, and of the returned `<div>` element — each keyed on exactly the values it depends on. The docs are explicit about the two headline consequences:

> "React Compiler automatically memoizes values and functions, reducing the need for manual `useMemo` calls."

> With React Compiler enabled, `memo` is **automatically applied** to all components. You typically **don't need manual `memo` anymore**.

So the practical answer to "do I still write `useMemo`/`useCallback`/`memo`?" is: **not in new code compiled by the compiler.** You still need to *read* them fluently, because they are everywhere in existing code and in every library.

#### What it does not do

- **It doesn't fix bad structure.** A component that re-renders because state is in the wrong place still re-renders; the fixes in [Advanced](#p10-advanced) remain yours to make.
- **It doesn't make a slow algorithm fast.** `filterTodos` over 100k items is still O(n) — the compiler only avoids re-running it needlessly.
- **It won't compile impure components.** It bails out on code it can't prove safe, silently leaving that component unoptimised. Rule violations are the usual cause, which is why the ESLint plugin matters.
- **It doesn't remove the need to understand referential identity.** Effect dependencies, context values, and external subscriptions are all still identity-driven.

#### Verifying it's working

The ESLint rules ship in `eslint-plugin-react-hooks` and flag code the compiler will refuse. Beyond that, check the build output for the runtime import the compiled code uses, or look for the ✨ badge React DevTools shows on compiled components.

**Try It.** Build the "❌ memo is defeated" example above, profile a keystroke in the React DevTools Profiler, and note `ExpensiveList` re-rendering. Now enable the compiler and re-profile without changing a line of the component. Expected result: `ExpensiveList` no longer re-renders on each keystroke — the compiler stabilised the `items` array and the `onSelect` function that you would otherwise have wrapped by hand.

<a id="part-10-cheat-sheet"></a>

### Part 10 cheat sheet

| Task | Code |
|---|---|
| Skip re-render on equal props | `memo(Component)` |
| Custom comparison | `memo(Component, arePropsEqual)` |
| Cache a value | `useMemo(() => compute(a), [a])` |
| Cache a function identity | `useCallback(fn, [deps])` |
| Time a calculation | `console.time` / `console.timeEnd` around it |
| See why a component rendered | DevTools Profiler → "Record why each component rendered" |
| See React work in a browser profile | Performance Tracks (19.2+) |

| Try this before `memo` | |
|---|---|
| Move state down into a leaf | Removes the re-render entirely |
| Pass JSX as `children` | Same element object → automatic bailout |
| Pass primitives, not objects | Shallow comparison actually succeeds |
| Delete an unnecessary Effect | Removes a whole render pass |
| Virtualise the list | Beats memoising 10,000 rows |

| Fact | |
|---|---|
| Does `memo` stop own-state re-renders? | No |
| Does `memo` stop context re-renders? | No |
| Is `useMemo` a correctness guarantee? | **No** — React may discard the cache |
| "Expensive" threshold | ≥ 1ms measured |
| Does the Compiler replace `memo`/`useMemo`/`useCallback`? | In compiled code, yes — but read them anyway |

[↑ Back to top](#table-of-contents)

---

## Part 11: Concurrent React

<a id="p11-beginner"></a>

### Beginner — urgent vs. non-urgent

Some updates must feel instant: typing, clicking, dragging. Others can wait a moment: filtering a big list, rendering a new tab, loading a route. Before concurrent rendering, React couldn't tell the difference — every update was processed synchronously and to completion, so one slow render blocked the keystroke behind it.

Concurrent React lets you say which is which. The renderer can then start a low-priority render, **pause it** when something urgent arrives, and discard or resume it later.

```
  Blocking (pre-18 model)
  ├─ keystroke ─┤ ████████████████ slow render ████████████████ │ next keystroke waits
                                                                  ↑ input feels frozen

  Concurrent
  ├─ keystroke ─┤ ███ transition ███ ⏸  ├─ keystroke ─┤ ███ restart ███ ✓
                                       ↑ interrupted; input stays responsive
```

<a id="p11-working"></a>

### Working Knowledge — useTransition and useDeferredValue

#### `useTransition`

```tsx
const [isPending, startTransition] = useTransition();

function selectTab(nextTab: string) {
  startTransition(() => {
    setTab(nextTab);            // non-urgent
  });
}
```

`isPending` is true while the transition is rendering, giving you somewhere to hang a subtle affordance. Two caveats matter, both from the [reference](https://react.dev/reference/react/useTransition):

> **Important**: State updates must happen **synchronously within** the `startTransition` callback to be marked as Transitions.

So `startTransition(() => setTimeout(() => setX(v), 0))` marks nothing. And for async work in React 19, updates after an `await` need re-wrapping:

```tsx
startTransition(async () => {
  const result = await fetchData();
  startTransition(() => {
    setState(result);          // required — the outer scope is gone after await
  });
});
```

> ⚠️ **Cannot control text inputs** — Transitions are non-blocking; input updates must be synchronous.

```tsx
// ❌ Wrong — the input lags behind the user's typing
startTransition(() => setText(e.target.value));

// ✅ Right — the input's own state is urgent; the expensive consumer is deferred
const [text, setText] = useState('');
const deferredText = useDeferredValue(text);
```

#### `useDeferredValue`

Same idea, different shape. Use `useTransition` when you own the `setState` call; use `useDeferredValue` when you only have the value — typically a prop, or state you must also render urgently.

```tsx
function Search() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <div style={{ opacity: isStale ? 0.5 : 1 }}>
        <SlowResults query={deferredQuery} />
      </div>
    </>
  );
}
```

React renders once immediately with the old `deferredQuery` (so the input updates instantly), then re-renders in the background with the new one, interrupting if you type again. `query !== deferredQuery` is the idiomatic "results are stale" check.

For this to help at all, the expensive child must be memoised or compiled — otherwise it re-renders on the urgent pass too and you've deferred nothing.

React 19 added an `initialValue` for the first render:

```tsx
const deferred = useDeferredValue(value, '');   // 19+
```

> **On React 18** — `useTransition` and `useDeferredValue` both exist and work as described; only the `initialValue` second argument is 19+. React 18 is where concurrent rendering shipped at all, via `createRoot`. If a post says "React doesn't have concurrent features", it predates March 2022.

| Reach for | When |
|---|---|
| `useTransition` | You control the state update and want an `isPending` flag |
| `useDeferredValue` | You only have a value (a prop), or the same state must render urgently too |
| `startTransition` (standalone import) | Same as `useTransition` but outside a component — no `isPending` |

<a id="p11-advanced"></a>

### Advanced — Suspense mechanics and Activity

#### What a Suspense boundary actually does

A boundary shows `fallback` when any component below it suspends. The nuance is *which* boundary: the **nearest one above the suspending component**. That gives you control over granularity — one boundary around the page means the whole page flashes; boundaries around each panel mean they resolve independently.

```tsx
<Suspense fallback={<PageSkeleton />}>
  <Header />
  <Suspense fallback={<ListSkeleton />}>
    <Albums />              {/* only this region falls back */}
  </Suspense>
</Suspense>
```

The interaction with transitions is the important part, and it's the fix for the most common Suspense complaint:

- Update **not** in a transition → React shows the fallback, replacing existing content.
- Update **in** a transition → React keeps the current content on screen and swaps only when ready.

```tsx
// ❌ Navigating blanks the page to a skeleton every time
setAlbumId(id);

// ✅ The current album stays until the next one is ready
startTransition(() => setAlbumId(id));
```

#### `<Activity>` — 19.2+

`<Activity>` hides a subtree **without unmounting it**:

```tsx
<Activity mode={isShowingSidebar ? 'visible' : 'hidden'}>
  <Sidebar />
</Activity>
```

What `hidden` does, per the [reference](https://react.dev/reference/react/Activity):

- applies `display: none`;
- **destroys Effects** and cleans up subscriptions;
- **preserves state and DOM**;
- keeps re-rendering children at lower priority as props change.

That combination is what makes it different from both alternatives:

| Approach | State | DOM (scroll, video position, text selection) | Effects | Cost while hidden |
|---|---|---|---|---|
| `{show && <Panel/>}` | Lost | Lost | Cleaned up | None |
| CSS `display:none` | Kept | Kept | **Still running** — timers, subscriptions, fetches | Full |
| `<Activity mode="hidden">` | Kept | Kept | Cleaned up | Low-priority renders only |

Two uses: preserving a tab's state and scroll position across switches, and **pre-rendering** content you expect to need soon so it's instant when revealed.

The caveat worth knowing: because the DOM survives, side effects owned by DOM nodes don't automatically stop. A `<video>` keeps playing. Add the cleanup yourself:

```tsx
useLayoutEffect(() => {
  return () => { videoRef.current?.pause(); };
}, []);
```

> **On React 18** — no `Activity`. The 18-era approximations are conditional rendering (loses state) or CSS hiding (keeps effects running, which is the leak this API exists to fix).

<a id="p11-mastery"></a>

### Mastery — what "interruptible" actually means

The property that makes all of this possible was established back in [Part 1](./react-foundations.md#p1-advanced): **render is pure and its output can be discarded.** Because calling your components has no observable side effects, React is free to:

- start rendering a transition, get partway, and abandon it when a keystroke arrives;
- render a hidden `<Activity>` subtree in leftover time;
- render the same component twice and keep one result;
- render a Suspense fallback and the real content in parallel, committing whichever is ready.

None of that is safe if your render mutates something. This is the concrete reason the purity rules are rules and not style preferences — and the concrete reason Strict Mode's double-invocation exists to catch violations before they become Heisenbugs under concurrency.

Two practical consequences:

**Rendering ≠ committing.** A component may render several times for one visible change. Counting renders is not measuring performance; measure commits and their durations.

**Nothing is torn.** Because commit is still synchronous and atomic, you never see a half-updated screen. Concurrency splits *rendering* into interruptible chunks, not the DOM update.

The scheduling machinery underneath — lanes, priority levels, `shouldYield`, and how React decides what to work on next — is [Part 13: Under the Hood](./react-internals-hooks.md).

<a id="part-11-cheat-sheet"></a>

### Part 11 cheat sheet

| Task | Code |
|---|---|
| Mark an update non-urgent | `startTransition(() => setX(v))` |
| …with a pending flag | `const [isPending, startTransition] = useTransition()` |
| Defer a value you don't set | `const d = useDeferredValue(v)` |
| Detect staleness | `v !== d` |
| Deferred initial value (19+) | `useDeferredValue(v, initial)` |
| Loading fallback | `<Suspense fallback={…}>` |
| Avoid the fallback on navigation | Wrap the update in a transition |
| Hide but keep state (19.2+) | `<Activity mode="hidden">` |
| Async transition (19) | Re-wrap updates after `await` in `startTransition` |

| Symptom | Fix |
|---|---|
| Input lags while a big list filters | `useDeferredValue` on the value the list consumes |
| Page blanks to a skeleton on every tab switch | Wrap the update in `startTransition` |
| Transition seems to do nothing | Update wasn't synchronous inside the callback, or the child isn't memoised/compiled |
| Tab loses scroll position when switching | `<Activity>` instead of conditional rendering |
| Hidden panel still polling the network | You used CSS hiding; use `<Activity>` |

[↑ Back to top](#table-of-contents)

---

**Previous:** [Parts 8–9 — Shared State & Data](./react-shared-state.md) · **Next:** [Parts 12–13 — Custom Hooks & Internals](./react-internals-hooks.md) · **Up:** [Overview](./react-mastery-guide.md)

*React 19.2.7 · React Compiler 1.x · verified September 6, 2026*
