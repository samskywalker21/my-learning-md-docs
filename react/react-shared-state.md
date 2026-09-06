# React — Shared State & Data (Parts 8–9)

> **Spec:** this doc follows the shared specification in [`react-mastery-guide.md`](./react-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Working Knowledge → Advanced → Mastery tiers, wrong-vs-right pairs, runnable **Try It** exercises, React 19 spine with `> **On React 18**` callouts.

Everything so far has been one component at a time. This doc is about state that more than one component needs — first from a parent (Part 8), then from a server (Part 9).

---

## Table of Contents

- [Part 8: Lifting, Context & Reducers](#part-8-lifting-context--reducers)
  - [Beginner — lifting state up](#p8-beginner)
  - [Working Knowledge — useReducer](#p8-working)
  - [Working Knowledge — Context](#p8-working-2)
  - [Advanced — reducer + context, and context performance](#p8-advanced)
  - [Mastery — what Context is not](#p8-mastery)
  - [Part 8 cheat sheet](#part-8-cheat-sheet)
- [Part 9: Data Fetching & Async UI](#part-9-data-fetching--async-ui)
  - [Beginner — fetching in an Effect](#p9-beginner)
  - [Working Knowledge — the four states, and the race condition](#p9-working)
  - [Advanced — Suspense, use(), and error boundaries](#p9-advanced)
  - [Mastery — transitions, caching, and why you'll want a library](#p9-mastery)
  - [Part 9 cheat sheet](#part-9-cheat-sheet)

---

## Part 8: Lifting, Context & Reducers

<a id="p8-beginner"></a>

### Beginner — lifting state up

When two components need the same value, move the state to their closest common parent and pass it down. That's the whole technique, and it's the right answer far more often than people expect.

```tsx
function Accordion() {
  const [activeIndex, setActiveIndex] = useState(0);   // ← lifted
  return (
    <>
      <Panel isActive={activeIndex === 0} onShow={() => setActiveIndex(0)} title="About" />
      <Panel isActive={activeIndex === 1} onShow={() => setActiveIndex(1)} title="Etymology" />
    </>
  );
}

function Panel({ title, isActive, onShow }: PanelProps) {
  return isActive ? <p>{title} content</p> : <button onClick={onShow}>Show</button>;
}
```

The child is now **controlled** — it has no state of its own and reports events upward. This is the same controlled/uncontrolled distinction as form inputs in [Part 5](./react-state-events.md#p5-working), applied to your own components.

Lifting has a cost: the parent re-renders, so all its children re-render. That is usually fine ([Part 10](./react-performance-concurrency.md) covers when it isn't), and it is much better than the two failure modes it replaces — duplicated state that drifts apart, or an Effect syncing one copy to the other.

<a id="p8-working"></a>

### Working Knowledge — useReducer

When a component's state updates are spread across many handlers that each do a bit of the same thing, consolidate them into a reducer: one function that takes the current state and an action, and returns the next state.

```tsx
type Action =
  | { type: 'added'; id: string; text: string }
  | { type: 'changed'; todo: Todo }
  | { type: 'deleted'; id: string };

function todosReducer(todos: Todo[], action: Action): Todo[] {
  switch (action.type) {
    case 'added':
      return [...todos, { id: action.id, text: action.text, done: false }];
    case 'changed':
      return todos.map(t => (t.id === action.todo.id ? action.todo : t));
    case 'deleted':
      return todos.filter(t => t.id !== action.id);
    default: {
      const _exhaustive: never = action;    // TS catches an unhandled action type
      throw new Error(`Unknown action: ${JSON.stringify(action)}`);
    }
  }
}

function TaskApp() {
  const [todos, dispatch] = useReducer(todosReducer, initialTodos);

  function handleAdd(text: string) {
    dispatch({ type: 'added', id: crypto.randomUUID(), text });
  }
  …
}
```

Three properties make this worth the ceremony:

1. **All transitions live in one place**, readable as a list of what can happen.
2. **The reducer is a pure function of `(state, action)`**, so it's trivially unit-testable with no React at all.
3. **The action describes intent, not mechanism** — `{ type: 'deleted', id }` rather than "here is the new array".

The discriminated-union action type plus the `never` exhaustiveness check is the idiomatic TypeScript pairing; see [`typescript/typescript-narrowing-functions.md`](../typescript/typescript-narrowing-functions.md).

| Prefer `useState` when | Prefer `useReducer` when |
|---|---|
| One or two independent values | Several values that change together |
| Updates are simple replacements | The same update logic repeats across handlers |
| Handlers are short | Handlers are long and mostly state-shuffling |
| | You want to log/test every transition |
| | A bug is "state got into an impossible combination" |

The reducer must be **pure** — no fetching, no timers, no mutation. Actions are described in the past tense by convention (`added`, not `add`), because they report something that happened rather than command something to happen.

> **On React 18** — identical. React 19 only changed the *TypeScript* typings (better contextual inference; don't annotate the whole reducer type).

<a id="p8-working-2"></a>

### Working Knowledge — Context

Context passes a value to any depth without threading props through every layer.

```tsx
// ThemeContext.ts
export const ThemeContext = createContext<'light' | 'dark'>('light');

// App.tsx — React 19
function App() {
  const [theme, setTheme] = useState<'light' | 'dark'>('dark');
  return (
    <ThemeContext value={theme}>
      <Page />
    </ThemeContext>
  );
}

// any descendant
function Button() {
  const theme = useContext(ThemeContext);
  return <button className={theme}>Click</button>;
}
```

> **On React 18** — the provider is `<ThemeContext.Provider value={theme}>`. React 19 lets you render the context object directly; `.Provider` still works but is deprecated. The *consumer* side (`useContext`) is unchanged in both.

Two things people get wrong:

```tsx
// ❌ Wrong — the prop must be called `value`
<ThemeContext theme={theme}>

// ✅ Right
<ThemeContext value={theme}>
```

And the default value is a fallback for *no provider at all*, not an initial value:

> The default value is used **only when there is no matching provider above**. It never changes during component lifetime.

A frequent debugging trap follows from the search direction — [`useContext()` "always looks for the closest provider *above* the component that calls it. It searches upwards and does not consider providers in the component from which you're calling `useContext()`"](https://react.dev/reference/react/useContext). A component cannot read the context it itself provides.

#### `use(context)` — 19+

`use` can read context too, and unlike `useContext` it is legal inside a condition or a loop:

```tsx
function HorizontalRule({ show }: { show: boolean }) {
  if (show) {
    const theme = use(ThemeContext);      // ✅ legal — use is not bound by hook order
    return <hr className={theme} />;
  }
  return null;
}
```

Use `useContext` by default; reach for `use(context)` when the conditional read genuinely simplifies the code.

<a id="p8-advanced"></a>

### Advanced — reducer + context, and context performance

#### The combination

Reducer for the logic, context for the distribution. This is the standard "app-level state without a library" pattern, and the piece worth copying is the **two-context split**:

```tsx
const TasksContext = createContext<Todo[]>([]);
const TasksDispatchContext = createContext<React.Dispatch<Action>>(() => {});

export function TasksProvider({ children }: { children: React.ReactNode }) {
  const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);
  return (
    <TasksContext value={tasks}>
      <TasksDispatchContext value={dispatch}>
        {children}
      </TasksDispatchContext>
    </TasksContext>
  );
}

export const useTasks = () => useContext(TasksContext);
export const useTasksDispatch = () => useContext(TasksDispatchContext);
```

Splitting matters because `dispatch` is **guaranteed stable** — React never changes its identity. A component that only dispatches (a "Add task" button) subscribes to the dispatch context alone and therefore never re-renders when the task list changes. Put both in one object and you have thrown that away.

#### The performance model

This is the caveat to have memorised:

> React **automatically re-renders** all the children that use a particular context starting from the provider that receives a different `value`. The previous and next values are compared with `Object.is()`. **Skipping re-renders with `memo` does not prevent children from receiving fresh context values.**

So `memo` does not shield a consumer from a context change. That combines badly with the most common context mistake:

```tsx
// ❌ Wrong — a new object every render, so every consumer re-renders every render
function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  return (
    <AuthContext value={{ user, login, logout }}>{children}</AuthContext>
  );
}

// ✅ Right — stable identity unless something actually changed
function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const login = useCallback((res) => setUser(res.user), []);
  const logout = useCallback(() => setUser(null), []);
  const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);
  return <AuthContext value={value}>{children}</AuthContext>;
}
```

With React Compiler enabled this particular memoisation is done for you ([Part 10](./react-performance-concurrency.md)) — but the *splitting* decision is architectural and the Compiler will not make it for you.

**Try It.** Put a `console.log('render', name)` in three components: one that reads a combined `{user, login}` context, one that reads a split value-only context, and one that reads a split dispatch-only context. Trigger an unrelated re-render of the provider. Expected result: the combined consumer logs every time; the dispatch-only consumer never logs.

#### Where to place a provider

Wrap the smallest subtree that needs it. A provider at the root means every context change re-renders candidates across the whole app; a provider around one route means it doesn't. Multiple small contexts beat one large one, for exactly the reason above.

<a id="p8-mastery"></a>

### Mastery — what Context is not

Context is a **dependency-injection mechanism, not a state manager**. It solves "how does this value get here", not "how is this value stored, updated efficiently, or shared with fine granularity". The distinction becomes practical when:

| Symptom | Context's limit | What to reach for |
|---|---|---|
| Updates are frequent (per-keystroke, per-frame) and many components consume | Every consumer re-renders on every change | An external store with selectors |
| Consumers need *part* of a large object | Context has no selector — you get the whole value or nothing | Zustand / Jotai / Redux Toolkit |
| State must be read outside React | `useContext` only works in components | An external store you can read directly |
| Server data with caching, revalidation, dedup | Context stores it; it doesn't fetch, cache, or invalidate it | TanStack Query ([Part 9](#p9-mastery)) |

All the mainstream stores are built on `useSyncExternalStore` ([Part 6](./react-effects-refs.md#p6-mastery)), which is the primitive that makes fine-grained subscriptions possible: a component subscribes to a *slice*, and only re-renders when that slice changes. Context fundamentally cannot do this, because its subscription unit is the whole context value.

The honest heuristic: reach for a store when you have measured a context-driven re-render problem, or when you need to read/write state from outside the React tree. Reaching for one on day one, for a form and a theme, is how apps end up with three state libraries and no clear owner for anything.

<a id="part-8-cheat-sheet"></a>

### Part 8 cheat sheet

| Task | Code |
|---|---|
| Share state between siblings | Lift it to the common parent |
| Consolidate complex transitions | `const [s, dispatch] = useReducer(reducer, init)` |
| Lazy reducer init | `useReducer(reducer, arg, init)` |
| Create context | `const C = createContext(defaultValue)` |
| Provide (19) | `<C value={v}>` |
| Provide (18) | `<C.Provider value={v}>` |
| Consume | `useContext(C)` |
| Consume conditionally (19) | `use(C)` |
| Stable context value | `useMemo` the object, `useCallback` the functions |
| Avoid re-rendering dispatch-only consumers | Split into value + dispatch contexts |

| Question | Answer |
|---|---|
| Does `memo` stop context re-renders? | **No** |
| How is the value compared? | `Object.is` |
| Can a component read the context it provides? | No — search goes strictly upward |
| Is `dispatch` stable? | Yes, always |
| Is Context a state manager? | No — it's dependency injection |

[↑ Back to top](#table-of-contents)

---

## Part 9: Data Fetching & Async UI

<a id="p9-beginner"></a>

### Beginner — fetching in an Effect

The minimal correct version, which is already more code than people expect:

```tsx
function Profile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(setUser);
  }, [userId]);

  if (!user) return <p>Loading…</p>;
  return <h1>{user.name}</h1>;
}
```

This works, and it has at least three bugs. The rest of this Part is those bugs.

<a id="p9-working"></a>

### Working Knowledge — the four states, and the race condition

#### Model the states properly

A fetch has four states, not two, and representing them as separate booleans invites impossible combinations ([Part 4](./react-state-events.md#p4-working)):

```tsx
type State<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'error'; error: Error }
  | { status: 'success'; data: T };
```

#### The race condition

This is the bug that makes hand-rolled fetching genuinely dangerous. Change `userId` from `1` to `2` quickly: two requests are in flight, and **the responses can arrive in either order**. If user 1's response is slower, it lands last and overwrites user 2 — you are now showing the wrong person's data with the right person's id in the URL.

```
   userId: 1 ──► request A ──────────────────────────► response A  (slow)
   userId: 2 ──────────► request B ──► response B                  (fast)
                                       setUser(B)      setUser(A)  ← WRONG
```

The documented fix is an ignore flag in the cleanup:

```tsx
useEffect(() => {
  let ignore = false;
  fetchResults(query, page).then(json => {
    if (!ignore) setResults(json);
  });
  return () => { ignore = true; };
}, [query, page]);
```

When the dependencies change, cleanup runs *before* the new setup, flipping the old closure's `ignore` to `true` — so the stale response, whenever it lands, is discarded. This four-line pattern is worth memorising verbatim.

`AbortController` is the stronger version, which also cancels the request:

```tsx
useEffect(() => {
  const controller = new AbortController();
  fetch(url, { signal: controller.signal })
    .then(r => r.json())
    .then(setData)
    .catch(err => { if (err.name !== 'AbortError') setError(err); });
  return () => controller.abort();
}, [url]);
```

Note the `catch` must ignore `AbortError`, or every navigation logs a spurious error.

#### The full careful version

```tsx
function useFetch<T>(url: string) {
  const [state, setState] = useState<State<T>>({ status: 'loading' });

  useEffect(() => {
    let ignore = false;
    setState({ status: 'loading' });

    fetch(url)
      .then(r => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);   // fetch does NOT throw on 404/500
        return r.json();
      })
      .then(data => { if (!ignore) setState({ status: 'success', data }); })
      .catch(error => { if (!ignore) setState({ status: 'error', error }); });

    return () => { ignore = true; };
  }, [url]);

  return state;
}
```

The `r.ok` check catches the third common bug: `fetch` only rejects on network failure, so a 500 response resolves happily and you try to render an error page's JSON.

**Try It.** Reproduce the race deliberately. Make your fetch `await new Promise(r => setTimeout(r, id === '1' ? 3000 : 300))` before resolving, then click from user 1 to user 2 quickly. Expected result without the ignore flag: user 2 appears, then is replaced by user 1 after three seconds. Add the flag; user 2 stays.

<a id="p9-advanced"></a>

### Advanced — Suspense, use(), and error boundaries

#### `<Suspense>`

A Suspense boundary shows a fallback while anything beneath it is "not ready", and it moves the loading state out of the component that loads.

```tsx
<Suspense fallback={<Skeleton />}>
  <Albums albumsPromise={albumsPromise} />
</Suspense>
```

#### `use(promise)` — 19+

```tsx
function Albums({ albumsPromise }: { albumsPromise: Promise<Album[]> }) {
  const albums = use(albumsPromise);        // suspends until resolved
  return <ul>{albums.map(a => <li key={a.id}>{a.title}</li>)}</ul>;
}
```

No `useState`, no `useEffect`, no loading branch. The component reads the data as if it were synchronous; the boundary above handles "not yet".

The critical constraint is promise identity:

> **Promises passed to `use` must be cached so the same Promise instance is reused across re-renders.** Creating new Promises during render causes the Suspense fallback to display repeatedly.

```tsx
// ❌ Wrong — a new promise every render, so it suspends forever
function Albums() {
  const albums = use(fetch('/albums').then(r => r.json()));
}

// ✅ Right — a cache keyed by URL, outside render
const cache = new Map<string, Promise<unknown>>();

export function fetchData(url: string) {
  if (!cache.has(url)) cache.set(url, fetch(url).then(r => r.json()));
  return cache.get(url)!;
}
```

That cache is deliberately naive — it never invalidates, never refetches, and never evicts. Which is precisely the argument in [Mastery](#p9-mastery) below.

Second constraint:

> **Do not call `use` inside try-catch blocks.** `use` throws internally to integrate with Suspense.

Rejected promises go to an error boundary instead.

> **On React 18** — `use` does not exist, and Suspense for client-side data fetching is not supported without a framework/library implementing it. On 18, Suspense is for `React.lazy` code-splitting and for framework-driven data. Your data-fetching option on 18 is the Effect pattern above, or a library. Do not try to hand-roll a throw-a-promise "Suspense-compatible" fetcher; it was never a supported API and the failure modes are ugly.

#### Error boundaries — still a class

React has no hook for catching render errors. An error boundary is the one component you will still write as a class:

```tsx
class ErrorBoundary extends React.Component<
  { fallback: React.ReactNode; children: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    logToService(error, info.componentStack);
  }

  render() {
    return this.state.hasError ? this.props.fallback : this.props.children;
  }
}
```

Most people use [`react-error-boundary`](https://github.com/bvaughn/react-error-boundary) rather than writing it, mainly for its reset support. Composed with Suspense, the two boundaries cover the two failure modes:

```tsx
<ErrorBoundary fallback={<p>Something went wrong.</p>}>
  <Suspense fallback={<Skeleton />}>
    <Albums albumsPromise={albumsPromise} />
  </Suspense>
</ErrorBoundary>
```

What error boundaries do **not** catch: errors in event handlers, in `setTimeout`, in async code after an await, and in the boundary itself. Those are ordinary JavaScript errors — use `try/catch`.

> **On React 18** — the same class API. What changed in 19 is *uncaught* errors: they are no longer re-thrown to `window`, and you configure reporting on the root instead:
> ```tsx
> createRoot(el, {
>   onCaughtError: (error, info) => log(error, info),
>   onUncaughtError: (error, info) => report(error, info),
> });
> ```

<a id="p9-mastery"></a>

### Mastery — transitions, caching, and why you'll want a library

#### Keeping the old UI while the new one loads

A plain Suspense boundary replaces content with the fallback on every navigation, which makes the app flash. Wrapping the update in a Transition tells React to keep showing the current content until the new content is ready:

```tsx
const [isPending, startTransition] = useTransition();

function selectAlbum(id: string) {
  startTransition(() => {
    setAlbumId(id);       // suspends — but the old album stays visible
  });
}
```

`isPending` gives you the subtle "loading" affordance (a dimmed panel, a spinner in the tab) instead of a full skeleton. This is the single biggest perceived-quality difference between a hand-rolled fetch UI and a good one — [Part 11](./react-performance-concurrency.md) covers the mechanics.

For React 19 async transitions, remember the caveat from [`useTransition`](https://react.dev/reference/react/useTransition): state updates **after an `await`** need their own `startTransition`:

```tsx
startTransition(async () => {
  const result = await fetchData();
  startTransition(() => { setData(result); });     // required
});
```

#### The problems a library exists to solve

Write the careful `useFetch` above and you have solved: loading state, error state, race conditions, and HTTP error detection. You have not solved:

| Problem | What it looks like |
|---|---|
| **Caching** | Navigating back refetches everything from scratch |
| **Deduplication** | Three components needing the same user fire three requests |
| **Invalidation** | After a mutation, which queries are now stale? |
| **Refetch on focus/reconnect** | Data is silently stale after the laptop wakes |
| **Pagination / infinite scroll** | Merging pages, keeping previous data while loading the next |
| **Retry with backoff** | One flaky request breaks the screen |
| **Garbage collection** | The naive `Map` cache above grows forever |
| **Optimistic mutation + rollback** | Partly solved by `useOptimistic` ([Part 5](./react-state-events.md#p5-mastery)) |

That list is the product spec for [TanStack Query](https://tanstack.com/query) and SWR, and it's why "just use `useEffect` and `fetch`" is honest advice for a demo and bad advice for an app. The reason to learn the primitives first is that every one of those libraries is built from them, and their failure modes are the ones in this Part.

React's own docs say the same thing more gently — that [modern frameworks provide more efficient built-in data fetching mechanisms than writing Effects directly in your components](https://react.dev/learn/you-might-not-need-an-effect).

Per the scope in [Deliberately Not Covered](./react-mastery-guide.md#6-deliberately-not-covered-and-where-to-look-instead), this set stops here rather than teaching a library.

<a id="part-9-cheat-sheet"></a>

### Part 9 cheat sheet

| Task | Code |
|---|---|
| Fetch in an Effect | `useEffect(() => { … }, [url])` with cleanup |
| Prevent stale responses | `let ignore = false; … return () => { ignore = true; }` |
| Cancel the request too | `AbortController` + `controller.abort()` in cleanup |
| Detect HTTP errors | `if (!r.ok) throw new Error(...)` — fetch doesn't reject on 4xx/5xx |
| Read a promise in render (19) | `const data = use(promise)` — promise must be cached |
| Show a loading fallback | `<Suspense fallback={…}>` |
| Catch render errors | Class error boundary / `react-error-boundary` |
| Avoid flashing the fallback | `startTransition(() => setId(next))` |
| Report uncaught errors (19) | `createRoot(el, { onUncaughtError, onCaughtError })` |

| Bug | Cause |
|---|---|
| Wrong user's data after fast navigation | Missing ignore flag / abort |
| Suspense fallback flashes forever | New promise created during render |
| "Error: Objects are not valid as a React child" | Rendering the error/JSON object directly |
| 500 response renders as success | Missing `r.ok` check |
| Console error on every navigation | Not filtering `AbortError` |
| Fallback replaces the whole page on navigation | Update not wrapped in a Transition |

[↑ Back to top](#table-of-contents)

---

**Previous:** [Parts 6–7 — Effects & Refs](./react-effects-refs.md) · **Next:** [Parts 10–11 — Performance & Concurrency](./react-performance-concurrency.md) · **Up:** [Overview](./react-mastery-guide.md)

*React 19.2.7 · verified September 6, 2026*
