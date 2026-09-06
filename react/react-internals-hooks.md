# React — Custom Hooks & Internals (Parts 12–13)

> **Spec:** this doc follows the shared specification in [`react-mastery-guide.md`](./react-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Working Knowledge → Advanced → Mastery tiers, wrong-vs-right pairs, runnable **Try It** exercises, React 19 spine with `> **On React 18**` callouts.

Part 12 is how you package the patterns from Parts 4–11 for reuse. Part 13 is the machinery underneath all of it — read after the rest, and every earlier rule stops being a rule you memorised and becomes something you could have derived.

Nothing in this doc differs between React 18 and 19. The fiber architecture, the hook storage model, and reconciliation are identical in both; the differences listed in [Part 14](./react-mastery-guide.md#5-part-14-react-18--19) are all surface API.

---

## Table of Contents

- [Part 12: Custom Hooks](#part-12-custom-hooks)
  - [Beginner — extracting logic](#p12-beginner)
  - [Working Knowledge — the Rules of Hooks](#p12-working)
  - [Advanced — composing, and what a hook does not share](#p12-advanced)
  - [Part 12 cheat sheet](#part-12-cheat-sheet)
- [Part 13: Under the Hood](#part-13-under-the-hood)
  - [Working Knowledge — elements, fibers, and the two trees](#p13-working)
  - [Advanced — the render and commit phases](#p13-advanced)
  - [Advanced — reconciliation and bailouts](#p13-advanced-2)
  - [Mastery — the hook linked list](#p13-mastery)
  - [Mastery — lanes and the scheduler](#p13-mastery-2)
  - [Part 13 cheat sheet](#part-13-cheat-sheet)

---

## Part 12: Custom Hooks

<a id="p12-beginner"></a>

### Beginner — extracting logic

A custom hook is a function whose name starts with `use` and which calls other hooks. That's the entire definition — there's no registration, no API, no special import.

```tsx
function useOnlineStatus() {
  return useSyncExternalStore(
    subscribe,
    () => navigator.onLine,
    () => true,
  );
}

// usage — reads like a built-in
function StatusBar() {
  const isOnline = useOnlineStatus();
  return <p>{isOnline ? '✅ Online' : '❌ Disconnected'}</p>;
}
```

The `use` prefix is not decoration. It's how the linter knows this function may call hooks and must obey the Rules of Hooks, and it's how *you* know at a call site that this line participates in the component's state.

```tsx
// ❌ Wrong — a function that calls hooks but isn't named like one
function getOnlineStatus() { return useSyncExternalStore(…); }

// ❌ Wrong — named like a hook but calls none; misleads readers and the linter
function useFormatDate(d: Date) { return d.toISOString(); }   // just a function

// ✅ Right on both counts
function useOnlineStatus() { return useSyncExternalStore(…); }
function formatDate(d: Date) { return d.toISOString(); }
```

<a id="p12-working"></a>

### Working Knowledge — the Rules of Hooks

Two rules, and both have real teeth ([the mechanism is in Part 13](#p13-mastery)):

**1. Only call hooks at the top level.** Not in conditions, loops, nested functions, or after an early return.

```tsx
// ❌ Every one of these is a violation
if (isLoggedIn) { const [x, setX] = useState(0); }
for (const item of items) { useEffect(…); }
function handleClick() { const [y] = useState(0); }
if (!user) return null;
const [z, setZ] = useState(0);        // unreachable on some renders
```

```tsx
// ✅ Hooks first, conditions after
const [z, setZ] = useState(0);
useEffect(() => { … }, []);
if (!user) return null;
```

If you need conditional state, extract a component — a component that isn't rendered runs no hooks at all, which is legal.

**2. Only call hooks from React functions** — a component, or another custom hook. Not from a plain function, an event handler, or a class.

The one documented exception is [`use`](https://react.dev/reference/react/use), which *may* be called conditionally. It is not a hook in the same sense; it doesn't allocate a slot.

#### What a custom hook gives you and what it doesn't

This is the point people get wrong most often:

> **Custom hooks share stateful logic, not state itself.**

Every call gets its own independent state. Two components calling `useCounter()` have two counters. The same component calling it twice has two counters.

```tsx
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const increment = () => setCount(c => c + 1);
  return { count, increment };
}

function Panel() {
  const a = useCounter();     // independent
  const b = useCounter();     // independent
}
```

To share the *state*, you need lifting, context, or a store ([Part 8](./react-shared-state.md)). A custom hook is a copy-paste of the logic, not a shared instance.

<a id="p12-advanced"></a>

### Advanced — composing, and what a hook does not share

#### Composition

Hooks compose like ordinary functions, and this is where they earn their keep:

```tsx
function useDebounced<T>(value: T, ms: number): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), ms);
    return () => clearTimeout(id);          // cancels the previous timer on each change
  }, [value, ms]);
  return debounced;
}

function useSearch(query: string) {
  const debouncedQuery = useDebounced(query, 300);
  return useFetch<Result[]>(`/api/search?q=${encodeURIComponent(debouncedQuery)}`);
}

// the component is now boring, which is the goal
function Search() {
  const [query, setQuery] = useState('');
  const state = useSearch(query);
  …
}
```

Note the cleanup in `useDebounced` — the debounce *is* the cleanup. Each change to `value` cancels the pending timer and starts a new one. That's a synchronisation, correctly expressed ([Part 6](./react-effects-refs.md)).

#### What to extract, and what not to

| Extract | Don't extract |
|---|---|
| A concrete synchronisation (`useChatRoom`, `useInterval`) | A wrapper that just renames `useState` |
| Logic repeated in three or more places | Logic used once, "for cleanliness" |
| Something with a good, specific name | Something you'd name `useLogic` or `useHelpers` |
| An Effect you'd rather not read again | An Effect you shouldn't have written ([Part 6](./react-effects-refs.md#p6-advanced-2)) |

The last row is the trap: extracting a bad Effect into `useSomething` doesn't fix it, it hides it. Delete first, extract second.

#### `useDebugValue`

Labels a custom hook's value in React DevTools:

```tsx
function useOnlineStatus() {
  const isOnline = useSyncExternalStore(…);
  useDebugValue(isOnline ? 'Online' : 'Offline');
  return isOnline;
}
```

Worth adding to hooks in a shared library; noise in application code.

#### Hooks and the Compiler

Custom hooks are compiled like components — the Compiler memoises inside them too. The same bail-out condition applies: a hook that breaks the rules or can't be proven pure is skipped silently. This is another reason to keep the ESLint plugin on ([setup](./react-mastery-guide.md#3-orientation--setup)).

<a id="p12-mastery"></a>

#### Mastery — collapsed

*Part 12's Mastery material is "why the Rules of Hooks exist", which is a fact about the data structure hooks are stored in. It is [Part 13's Mastery tier](#p13-mastery), immediately below.*

<a id="part-12-cheat-sheet"></a>

### Part 12 cheat sheet

| Rule | |
|---|---|
| Naming | Must start with `use` |
| Call site | Top level of a component or another hook only |
| Conditions/loops/handlers | ❌ Never (except `use`) |
| Early return before a hook | ❌ Never |
| What's shared between callers | The **logic**. Never the state |
| Sharing actual state | Lift, Context, or a store ([Part 8](./react-shared-state.md)) |
| Label in DevTools | `useDebugValue(label)` |
| Enforcement | `eslint-plugin-react-hooks` |

| Common custom hooks worth writing once | |
|---|---|
| `useDebounced(value, ms)` | Timer + cleanup |
| `useLocalStorage(key, initial)` | `useSyncExternalStore` over the `storage` event |
| `useOnlineStatus()` | `useSyncExternalStore` over `online`/`offline` |
| `useInterval(fn, delay)` | `useEffectEvent` + `setInterval` |
| `useMediaQuery(q)` | `useSyncExternalStore` over `matchMedia` |
| `usePrevious(value)` | A ref updated in an Effect |

[↑ Back to top](#table-of-contents)

---

## Part 13: Under the Hood

<a id="p13-beginner"></a>

#### Beginner — collapsed

*There is no useful Beginner tier for internals: the Beginner-level answer is [the one-paragraph mental model](./react-mastery-guide.md#4-the-one-paragraph-mental-model), and everything below assumes Parts 1–11.*

<a id="p13-working"></a>

### Working Knowledge — elements, fibers, and the two trees

Three things get casually called "the virtual DOM", and separating them is most of the insight.

**An element** is what your JSX evaluates to ([Part 2](./react-foundations.md#p2-advanced)): a plain, immutable object describing *what* should exist. Created fresh on every render, thrown away immediately after.

```js
{ $$typeof: Symbol(react.transitional.element), type: Counter, key: null, props: { n: 1 } }
```

**A fiber** is React's internal, mutable, long-lived record of a component *instance* — where the actual state lives. One fiber per position in the tree, surviving across renders. Roughly:

```js
{
  type: Counter,            // the function
  key: null,
  stateNode: null,          // the DOM node, for host components
  memoizedProps: {…},       // props from the last committed render
  memoizedState: {…},       // ← the head of the hook linked list
  return: parentFiber,      // parent
  child: firstChildFiber,   // first child only
  sibling: nextFiber,       // next sibling
  alternate: otherFiber,    // the same position in the other tree
  flags: …,                 // what to do at commit (Placement, Update, Deletion…)
  lanes: …,                 // priority of pending work
}
```

Note the shape: not an array of children, but **child / sibling / return** pointers — a singly-linked tree. That's what lets React walk it iteratively with an explicit cursor instead of recursively, and therefore what lets it **pause mid-traversal**, which is the whole basis of concurrent rendering ([Part 11](./react-performance-concurrency.md)).

```
        ┌──────────┐
        │   App    │
        └────┬─────┘
             │ child
        ┌────▼─────┐  sibling   ┌──────────┐  sibling  ┌──────────┐
        │  Header  ├───────────►│   List   ├──────────►│  Footer  │
        └──────────┘            └────┬─────┘           └──────────┘
             ▲                       │ child
             │ return           ┌────▼─────┐ sibling ┌──────────┐
             └──────────────────┤  Item 1  ├────────►│  Item 2  │
                                └──────────┘         └──────────┘
```

**The two trees (double buffering).** React keeps two fiber trees: `current` (what's on screen) and `workInProgress` (what's being built). Each fiber points to its counterpart via `alternate`. Rendering builds the work-in-progress tree; committing swaps the pointer so it becomes `current`.

```
    before commit                    after commit
    ┌─────────┐                      ┌─────────┐
    │ current │──► tree A            │ current │──► tree B
    └─────────┘    ▲                 └─────────┘    ▲
                   │ alternate                      │ alternate
    ┌─────────┐    ▼                 ┌─────────┐    ▼
    │   WIP   │──► tree B            │   WIP   │──► tree A  (reused next time)
    └─────────┘                      └─────────┘
```

That's why abandoning a render is free: nothing on screen was touched, and the discarded work-in-progress tree is just garbage. It's also why React allocates so little — the two trees are recycled, not rebuilt.

<a id="p13-advanced"></a>

### Advanced — the render and commit phases

**Render phase.** React walks the tree building work-in-progress fibers, calling your components as it goes. Interruptible, restartable, discardable — and therefore required to be pure. Nothing is visible yet.

**Commit phase.** React applies the accumulated changes. Synchronous, uninterruptible, atomic. Broken into three sub-phases, whose order explains several timing rules from earlier Parts:

```
  ┌─ BEFORE MUTATION ──────────────────────────────────────┐
  │  read the DOM as it still is (getSnapshotBeforeUpdate)  │
  └────────────────────────┬───────────────────────────────┘
                           ▼
  ┌─ MUTATION ─────────────────────────────────────────────┐
  │  insert / update / delete DOM nodes                     │
  │  detach old refs                                        │
  │  run useLayoutEffect CLEANUPS                           │
  └────────────────────────┬───────────────────────────────┘
                           ▼
              current ← workInProgress   (the pointer swap)
                           ▼
  ┌─ LAYOUT ───────────────────────────────────────────────┐
  │  attach new refs                                        │
  │  run useLayoutEffect SETUPS  ← ref.current is available │
  └────────────────────────┬───────────────────────────────┘
                           ▼
                    browser PAINTS
                           ▼
              useEffect cleanups, then setups
```

Which is exactly why:

- `ref.current` is `null` during render but populated in effects — refs are attached in the layout sub-phase, after your function has returned;
- `useLayoutEffect` can measure the DOM without a flash, and blocks paint doing it;
- `useEffect` runs after paint and so can never cause a visible pre-paint flicker — but can cause a visible *post*-paint one if it sets state (which is what `useLayoutEffect` exists to avoid);
- the pointer swap happens between the two, so during layout effects `current` already describes the new screen.

<a id="p13-advanced-2"></a>

### Advanced — reconciliation and bailouts

Diffing two arbitrary trees is O(n³). React gets O(n) by making two assumptions, and every rule about keys and component identity falls out of them:

**1. Different types produce different trees.** If `type` changed at a position, React doesn't diff further — it destroys the old subtree (including all state) and builds a new one. This is why [defining a component inside another component](./react-foundations.md#p2-mastery) is catastrophic: the `type` is a new function object every render.

**2. Keys mark stable identity across renders.** Within a list, React matches children by key rather than position, which is the [Part 3](./react-foundations.md#part-3-lists--keys) material seen from the other side.

The children-diff is a single pass, roughly:

```
  first pass:  walk old and new children in parallel while keys match
               → update in place (the fast path)
  then:        if new children remain → insert them
               if old children remain → delete them
               if keys matched out of order → build a key→fiber map,
                                              move rather than recreate
```

Two consequences worth internalising: appending to the end of a list is cheap because the first pass matches everything and then inserts; prepending with **index keys** is maximally expensive because *every* position's key changed, so nothing matches and React updates every row.

#### Bailouts

React can skip work entirely. It bails out of re-rendering a subtree when, at a given fiber:

- props are referentially equal to the previous props (this includes the same `children` element object — the mechanism behind [passing JSX as children](./react-performance-concurrency.md#p10-advanced)), **and**
- there is no pending state update on that fiber, **and**
- context it consumes hasn't changed, **and**
- `type` is unchanged.

`memo` doesn't invent a new mechanism; it makes an existing bailout reachable by comparing props shallowly instead of referentially. And a `setState` that produces an `Object.is`-equal value can bail out too — though as noted in [Part 4](./react-state-events.md#p4-mastery), React may still re-render that one component once before it does.

<a id="p13-mastery"></a>

### Mastery — the hook linked list

Here's the payoff. **A fiber's `memoizedState` is the head of a singly-linked list of hook records**, one per hook call, in call order:

```
  function Form() {
    const [name, setName] = useState('');      ─┐
    const [age, setAge] = useState(0);          │
    useEffect(…);                               │
  }                                             │
                                                ▼
  fiber.memoizedState
        │
        ▼
  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
  │ hook #1       │    │ hook #2       │    │ hook #3       │
  │ state: ''     │───►│ state: 0      │───►│ effect record │───► null
  │ queue: […]    │next│ queue: […]    │next│ deps: […]     │
  └───────────────┘    └───────────────┘    └───────────────┘
```

There is no name, no key, no identifier anywhere in that structure. On a re-render React walks the list with a cursor, handing out the next record each time a hook is called. **Position in the call sequence is the only identity a hook has.**

So now the rules are not rules:

```tsx
// render 1: showName = true          render 2: showName = false
if (showName) {
  const [name] = useState('');        // hook #1 ← skipped on render 2
}
const [age] = useState(0);            // hook #2 ← now reads hook #1's record
```

On render 2 the cursor hands `age` the record that belongs to `name`. Your number is now a string, your setters are crossed, and React's warning ("Rendered fewer hooks than expected") is a best-effort detection of a corruption that has already happened. The linter exists because *this is not generally detectable at runtime*.

The same structure explains the rest of the set:

| Rule from earlier | Because |
|---|---|
| Hooks must be unconditional and in order | The list is positional; there are no names |
| Custom hooks share logic, not state | Their hooks are appended to the *calling* fiber's list — each call site gets its own records |
| Two `useCounter()` calls are independent | Two disjoint runs of records in the same list |
| State is tied to tree position | State lives on the fiber, and the fiber *is* the position |
| Changing `key` resets state | Different key → different fiber → a fresh, empty hook list |
| `useState` initial value is ignored after mount | On update React reads the existing record and never looks at the argument |
| `useRef` returns the same object every time | It's one record, mutated in place, never recreated |
| `dispatch` from `useReducer` is stable | It's stored on the record and bound to the fiber, not recreated per render |

**Try It.** Deliberately break it, once, so the failure is familiar:

```tsx
export default function App() {
  const [show, setShow] = useState(true);
  if (show) {
    const [a] = useState('hello');    // 🚩 conditional hook
  }
  const [n, setN] = useState(42);
  return <button onClick={() => setShow(s => !s)}>{String(n)}</button>;
}
```

Expected result: it renders `42`, and clicking the button throws *"Rendered fewer hooks than expected. This may be caused by an accidental early return statement."* Your ESLint plugin should have flagged the line before you ever ran it.

<a id="p13-mastery-2"></a>

### Mastery — lanes and the scheduler

React needs to represent "this update is more urgent than that one", and to combine and check sets of pending priorities cheaply. It does this with **lanes**: a 31-bit bitmask where each bit is a priority band.

```
   lane bit    band                    produced by
   ───────────────────────────────────────────────────────────────
   0           SyncLane                discrete input: click, keydown, flushSync
   ...         InputContinuousLane     drag, scroll, mousemove
   ...         DefaultLane             everything else (timeouts, network callbacks)
   ...         TransitionLanes         startTransition / useTransition / useDeferredValue
   ...         RetryLanes              Suspense retries
   30          IdleLane                offscreen / hidden Activity work
```

Bitmasks make the operations React needs constant-time: merging pending work is `a | b`, "is any of this work in the set I'm rendering?" is `a & b`, and picking the most urgent is finding the lowest set bit. A fiber's `lanes` field records pending work on that fiber; a parent's `childLanes` records pending work *anywhere* beneath it — which is how React skips whole subtrees during a render without visiting them at all.

The rendering loop is then, in outline:

```
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);       // one fiber at a time
  }
  // yielded? hand control back to the browser via a scheduler callback,
  // let it paint and handle input, then resume — or throw this work away
  // and restart if something more urgent arrived.
```

`shouldYield()` is what makes rendering interruptible: React checks whether it has run long enough that the browser should get a turn. Synchronous-lane work skips that check entirely and runs to completion — a click must not be deferred.

This is also the mechanism behind behaviours from earlier Parts. A transition renders in a transition lane, so a keystroke in a sync lane preempts it and React discards the partial work ([Part 11](./react-performance-concurrency.md)). `<Activity mode="hidden">` renders in an idle lane, so pre-rendering never competes with anything visible. And "React batches updates" is really "updates in the same lane, scheduled in the same tick, are processed in one render pass".

If you want to go further, the source is more readable than its reputation: `ReactFiberWorkLoop.js`, `ReactFiberHooks.js` and `ReactFiberLane.js` in [facebook/react](https://github.com/facebook/react) are the three files that contain almost everything above.

<a id="part-13-cheat-sheet"></a>

### Part 13 cheat sheet

| Term | What it is |
|---|---|
| **Element** | Immutable plain object from JSX — a *description*. Recreated every render |
| **Fiber** | Mutable long-lived record of a component instance — holds state. Survives renders |
| **`current` tree** | The fiber tree currently on screen |
| **`workInProgress` tree** | The one being built; becomes `current` at commit |
| **`alternate`** | A fiber's counterpart in the other tree |
| **Render phase** | Calls your components, builds WIP. Interruptible, discardable, must be pure |
| **Commit phase** | Applies DOM changes. Synchronous, atomic |
| **Reconciliation** | Matching new elements to existing fibers, by `type` and `key` |
| **Bailout** | Skipping a subtree because nothing relevant changed |
| **Lane** | One bit in a priority bitmask |
| **`shouldYield()`** | The check that makes render interruptible |

| Fiber field | Meaning |
|---|---|
| `type` | Your function, or a tag string like `'div'` |
| `stateNode` | The real DOM node (host components) |
| `memoizedState` | Head of the hook linked list |
| `memoizedProps` | Props from the last committed render |
| `child` / `sibling` / `return` | Tree links — a linked tree, not an array |
| `flags` | Commit work: Placement, Update, Deletion, … |
| `lanes` / `childLanes` | Pending work here / anywhere below |

| Earlier rule | Internals reason |
|---|---|
| Hooks unconditional, same order | Positional linked list, no names |
| Render must be pure | Renders get discarded, restarted, doubled |
| State tied to tree position | State lives on the fiber; the fiber is the position |
| Changing `key` resets state | New fiber, empty hook list |
| Don't nest component definitions | New `type` every render → forced remount |
| Index keys corrupt list state | Reconciliation matches by key, not content |
| Passing `children` avoids re-renders | Same element object → props referentially equal → bailout |
| `ref.current` is null during render | Refs attach in the layout sub-phase |
| `useLayoutEffect` blocks paint | It runs in the layout sub-phase, before paint |

[↑ Back to top](#table-of-contents)

---

**Previous:** [Parts 10–11 — Performance & Concurrency](./react-performance-concurrency.md) · **Up:** [Overview](./react-mastery-guide.md) → [Suggested Learning Order](./react-mastery-guide.md#7-suggested-learning-order) · [Quick Self-Check](./react-mastery-guide.md#8-quick-self-check)

*React 19.2.7 · verified September 6, 2026*
