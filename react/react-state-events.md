# React — State & Events (Parts 4–5)

> **Spec:** this doc follows the shared specification in [`react-mastery-guide.md`](./react-mastery-guide.md#about-this-document) — curiosity-driven framing, Beginner → Working Knowledge → Advanced → Mastery tiers, wrong-vs-right pairs, runnable **Try It** exercises, React 19 spine with `> **On React 18**` callouts.

[Part 1–3](./react-foundations.md) produced a static description of UI. This doc is about making it change: how state works, why it behaves like a snapshot rather than a variable, and how user input gets in.

Part 4 is the most important Part in this set. Nearly every "React is confusing" moment traces back to something in it.

---

## Table of Contents

- [Part 4: State & the Render Cycle](#part-4-state--the-render-cycle)
  - [Beginner — useState](#p4-beginner)
  - [Working Knowledge — objects, arrays, and structuring state](#p4-working)
  - [Advanced — state as a snapshot, batching, and updaters](#p4-advanced)
  - [Advanced — preserving and resetting state](#p4-advanced-2)
  - [Mastery — the update queue](#p4-mastery)
  - [Part 4 cheat sheet](#part-4-cheat-sheet)
- [Part 5: Events & Forms](#part-5-events--forms)
  - [Beginner — handlers](#p5-beginner)
  - [Working Knowledge — the event object, controlled inputs, useId](#p5-working)
  - [Advanced — propagation, delegation, and uncontrolled inputs](#p5-advanced)
  - [Mastery — Actions (19+)](#p5-mastery)
  - [Part 5 cheat sheet](#part-5-cheat-sheet)

---

## Part 4: State & the Render Cycle

<a id="p4-beginner"></a>

### Beginner — useState

A plain variable doesn't work, for two independent reasons: it doesn't survive a re-render, and changing it doesn't cause one.

```tsx
// ❌ Wrong — nothing happens on screen
function Counter() {
  let count = 0;
  return <button onClick={() => { count++; }}>{count}</button>;
}

// ✅ Right
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

`useState(initial)` returns a pair: the value for *this render*, and a setter that asks React to re-render with a new value. The array destructuring is convention, not magic — `useState` returns a two-element array and you name both halves.

The initial value is used **only on the first render**. On every render after that, `useState` hands back the current value and ignores the argument.

```tsx
// If computing the initial value is expensive, pass a function — it's called once.
const [rows, setRows] = useState(() => parseHugeCSV(raw));   // ✅ lazy
const [rows, setRows] = useState(parseHugeCSV(raw));          // 🚩 runs every render
```

<a id="p4-working"></a>

### Working Knowledge — objects, arrays, and structuring state

#### State is immutable

React compares old and new state with `Object.is`. Mutating an object leaves it `===` to itself, so React sees no change and skips the re-render.

```tsx
// ❌ Wrong — mutation; React sees the same object
function addTodo(text: string) {
  todos.push({ id: crypto.randomUUID(), text });
  setTodos(todos);
}

// ✅ Right — a new array
function addTodo(text: string) {
  setTodos([...todos, { id: crypto.randomUUID(), text }]);
}
```

The everyday array operations, in their immutable forms:

| Operation | Mutating (❌) | Immutable (✅) |
|---|---|---|
| Add to end | `arr.push(x)` | `[...arr, x]` |
| Add to start | `arr.unshift(x)` | `[x, ...arr]` |
| Remove by id | `arr.splice(i, 1)` | `arr.filter(a => a.id !== id)` |
| Replace one | `arr[i] = x` | `arr.map(a => a.id === id ? x : a)` |
| Sort | `arr.sort(f)` | `arr.toSorted(f)` or `[...arr].sort(f)` |
| Reverse | `arr.reverse()` | `arr.toReversed()` |
| Insert at i | `arr.splice(i, 0, x)` | `[...arr.slice(0, i), x, ...arr.slice(i)]` |

`toSorted`/`toReversed`/`with`/`toSpliced` are the modern non-mutating array methods — see [`javascript/javascript-arrays-iteration.md`](../javascript/javascript-arrays-iteration.md).

For nested objects, spread at every level you change:

```tsx
setUser({
  ...user,
  address: { ...user.address, city: 'Berlin' },
});
```

If that gets deep enough to hurt, the nesting is usually the problem — see below — but [Immer](https://immerjs.github.io/immer/) (via `useImmer`) is the standard escape hatch.

#### Structuring state well

Four principles, each of which removes a category of bug:

1. **Group state that changes together.** Two values always updated in the same handler should be one object (or one reducer — [Part 8](./react-shared-state.md)).
2. **Avoid contradictions.** `isLoading` + `isError` + `isSuccess` as three booleans allows five impossible combinations. One `status: 'idle' | 'loading' | 'error' | 'success'` allows none.
3. **Avoid redundancy — don't put derived values in state.** If it can be computed during render, compute it during render.
4. **Avoid duplication.** Store an id and look the item up, rather than storing a second copy of the item.

```tsx
// ❌ Wrong — fullName is redundant and will drift out of sync
const [first, setFirst] = useState('');
const [last, setLast] = useState('');
const [fullName, setFullName] = useState('');

// ✅ Right — derive it
const [first, setFirst] = useState('');
const [last, setLast] = useState('');
const fullName = `${first} ${last}`;
```

This is the single most common unnecessary-state mistake, and it usually comes packaged with an unnecessary Effect ([Part 6](./react-effects-refs.md) covers why).

```tsx
// ❌ Wrong — duplication: selectedItem is a stale copy
const [items, setItems] = useState(initial);
const [selectedItem, setSelectedItem] = useState(null);
// edit an item → selectedItem still holds the old version

// ✅ Right — store the id, derive the item
const [selectedId, setSelectedId] = useState<string | null>(null);
const selectedItem = items.find(i => i.id === selectedId) ?? null;
```

<a id="p4-advanced"></a>

### Advanced — state as a snapshot, batching, and updaters

This section is the load-bearing one.

**A state variable is not a variable. It is a value baked into one particular render.** When React calls your component, `count` is a `const` holding whatever the state was at the time of that call. Nothing that happens later changes it — not a setter call, not an await, not a timeout. A new value only appears in a *new render*, which is a new call to your function with a new `count`.

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
    console.log(count);        // logs 0 — always
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

Click once: the count becomes **1**, not 3, and the log says **0**. Two mechanisms combine to produce that:

**Batching.** [React waits until all code in the event handler has run before processing your state updates](https://react.dev/learn/queueing-a-series-of-state-updates) — one re-render for the whole handler, not one per setter.

**Snapshots.** All three calls read `count` from *this* render, where it is `0`. So all three enqueue "replace with 1".

```
  handler runs (count === 0 for its entire lifetime)
    setCount(count + 1)  →  queue: [replace with 1]
    setCount(count + 1)  →  queue: [replace with 1, replace with 1]
    setCount(count + 1)  →  queue: [replace with 1, replace with 1, replace with 1]
  handler ends → React processes queue → final state 1 → one re-render
```

The fix is the **updater function**, which receives the pending state rather than the render's snapshot:

```tsx
setCount(c => c + 1);
setCount(c => c + 1);
setCount(c => c + 1);   // → 3
```

```
  queue: [c=>c+1, c=>c+1, c=>c+1]
    c = 0 → 1
    c = 1 → 2
    c = 2 → 3
```

The rule, verbatim from the docs:

> - **An updater function** (e.g. `n => n + 1`) gets added to the queue.
> - **Any other value** (e.g. number `5`) adds "replace with `5`" to the queue, ignoring what's already queued.

Naming convention: the first letters of the state variable — `setEnabled(e => !e)`, `setFriendCount(fc => fc * 2)`.

**When to reach for the updater form:** whenever the next state depends on the previous one. Two setter calls in one handler, an update inside a `setInterval` or `setTimeout`, an update after an `await`, or an update inside an Effect. If it's a plain replacement from a fresh value (`setName(e.target.value)`), the direct form is fine and clearer.

**Try It.** Snapshots outlive the handler:

```tsx
function Delayed() {
  const [count, setCount] = useState(0);
  return (
    <button
      onClick={() => {
        setCount(count + 1);
        setTimeout(() => alert(`count is ${count}`), 3000);
      }}
    >
      {count}
    </button>
  );
}
```

Click once, then click three more times before the alert fires. Expected result: the button reads **4**, and the alert says **count is 0** — the closure captured the first render's snapshot and three seconds of subsequent renders could not change it. Swap `alert(count)` for a `setCount(c => …)` and you see the pending-value behaviour instead.

> **On React 18** — batching works the same way. React 18 is where **automatic batching** was extended to updates outside event handlers (in promises, timeouts, native handlers). On React 17 and earlier, `setTimeout(() => { setA(1); setB(2); })` caused *two* renders. If you read an old blog post claiming updates in async code aren't batched, it predates React 18.

<a id="p4-advanced-2"></a>

### Advanced — preserving and resetting state

Where does state actually live? Not in the component, and not in the JSX. From [Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state):

> "React keeps state for as long as the same component is rendered at the same position. State is not kept in JSX tags. It's associated with the tree position in which you put that JSX."

So the rules are:

| Situation | Result |
|---|---|
| Same component type, same position | State **preserved** (even if props changed) |
| Different component type, same position | State **destroyed**, subtree rebuilt |
| Same position, different `key` | State **destroyed** — you asked for a different identity |
| Component removed from the tree | State destroyed |

The surprising case is the first one:

```tsx
// Both branches put <Counter> as the first child of <div>.
// React sees the SAME component at the SAME position → the count survives the toggle.
{isFancy ? <Counter isFancy={true} /> : <Counter isFancy={false} />}
```

And the counterpart:

```tsx
// 🚩 Different types at the same position → Counter's state is destroyed
{isPaused ? <p>See you later!</p> : <Counter />}
```

```
   position: <div> child #1
   ┌────────────────────────────────────────┐
   │ render A        render B      state?   │
   ├────────────────────────────────────────┤
   │ <Counter/>  →   <Counter/>    kept     │
   │ <Counter/>  →   <p/>          destroyed│
   │ <Counter k=1/> → <Counter k=2/> destroyed│
   └────────────────────────────────────────┘
```

#### The `key` reset — the pattern that replaces a whole family of bugs

The classic wrong version is an Effect that clears state when a prop changes:

```tsx
// ❌ Wrong — an Effect to reset state
function Profile({ userId }: { userId: string }) {
  const [comment, setComment] = useState('');
  useEffect(() => {
    setComment('');
  }, [userId]);
  …
}
```

That renders once with the *stale* comment before the Effect clears it, and it only resets the state you remembered to list. The documented fix:

```tsx
// ✅ Right — reset the whole component by identity
<Profile userId={userId} key={userId} />
```

> "By passing `userId` as a `key` to the `Profile` component, you're asking React to treat two `Profile` components with different `userId` as two different components that should not share any state."

The `key` here is not in a list — keys work anywhere, not just inside `map`.

#### Adjusting *some* state when props change

Occasionally you need to reset one piece of state rather than the whole component. The documented pattern updates state **during render**, which React handles specially — it re-runs the component immediately, before touching the DOM, with no intermediate paint:

```tsx
function List({ items }: { items: Item[] }) {
  const [prevItems, setPrevItems] = useState(items);
  const [selection, setSelection] = useState<Item | null>(null);

  if (items !== prevItems) {        // ✅ compare, then set — during render
    setPrevItems(items);
    setSelection(null);
  }
  …
}
```

The docs immediately hedge this, and so should you:

> "Although this pattern is more efficient than an Effect, most components shouldn't need it either… Always check whether you can reset all state with a key or calculate everything during rendering instead."

Use it only when the `key` reset is too coarse. And note that only a *conditional* set during render is legal — an unconditional one is an infinite loop.

<a id="p4-mastery"></a>

### Mastery — the update queue

Each `useState` maintains a queue of pending updates attached to its hook, and processes them in order when the component re-renders. That is what makes the two forms differ: a value **replaces** the accumulator; a function **transforms** it. Mixing them is well-defined but rarely what you want:

```tsx
setCount(5);            // queue: [replace 5]
setCount(c => c + 1);   // queue: [replace 5, c=>c+1]   → 6
setCount(42);           // queue: [replace 5, c=>c+1, replace 42] → 42
```

Two further behaviours worth knowing:

**Bailouts.** If the new state is `Object.is`-equal to the current state, React may skip re-rendering the subtree. It may still re-render the component *once* before bailing out — so don't rely on "setting the same value never renders" for correctness.

```tsx
const [n, setN] = useState(0);
setN(0);   // may render once, then bail out; children are not re-rendered
```

**Updater functions must be pure.** React can call them twice in development (Strict Mode) to check. Do not put logging, mutation, or anything else with a side effect inside `setX(prev => …)`.

The actual data structure — where the queue lives, how it's keyed to the component instance, and why call *order* is what identifies a hook — is [Part 13](./react-internals-hooks.md).

<a id="part-4-cheat-sheet"></a>

### Part 4 cheat sheet

| Task | Code |
|---|---|
| Declare state | `const [x, setX] = useState(init)` |
| Expensive initial value | `useState(() => compute())` |
| Depends on previous state | `setX(prev => next)` |
| Replace outright | `setX(value)` |
| Toggle | `setOpen(o => !o)` |
| Add to array | `setArr([...arr, item])` |
| Remove from array | `setArr(arr.filter(a => a.id !== id))` |
| Update one item | `setArr(arr.map(a => a.id === id ? {...a, done: true} : a))` |
| Update nested object | `setU({...u, addr: {...u.addr, city}})` |
| Reset a child's state | Change its `key` |

| Question | Answer |
|---|---|
| Why didn't my state update immediately? | It's a snapshot; the new value appears in the next render |
| Why did three `+1`s add one? | All read the same snapshot — use the updater form |
| Are updates batched? | Yes, everywhere, since React 18 |
| Where does state live? | At a position in the UI tree, identified by type and key |
| Does changing props reset state? | No. Changing `key` or component *type* does |
| Can I set state during render? | Only conditionally, only to adjust state from props, and prefer `key` |

[↑ Back to top](#table-of-contents)

---

## Part 5: Events & Forms

<a id="p5-beginner"></a>

### Beginner — handlers

Pass a function; don't call one.

```tsx
// ❌ Wrong — calls handleClick during render, every render
<button onClick={handleClick()}>Click</button>

// ✅ Right — passes the function
<button onClick={handleClick}>Click</button>

// ✅ Right — needs an argument, so wrap in an arrow
<button onClick={() => handleDelete(id)}>Delete</button>
```

The `onClick()` mistake produces a very characteristic symptom: something fires immediately on page load and possibly loops forever (if the handler sets state, which re-renders, which calls it again).

Handler naming convention: `handleX` for the function, `onX` for the prop that receives it.

```tsx
function Button({ onClick, children }: { onClick: () => void; children: React.ReactNode }) {
  return <button onClick={onClick}>{children}</button>;
}

function Toolbar() {
  function handleSave() { … }
  return <Button onClick={handleSave}>Save</Button>;
}
```

<a id="p5-working"></a>

### Working Knowledge — the event object, controlled inputs, useId

#### The event object

Handlers receive a React **synthetic event**, which wraps the native one with a cross-browser-consistent interface.

```tsx
function Form() {
  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();          // stop the browser's full-page reload
    …
  }
  return <form onSubmit={handleSubmit}>…</form>;
}
```

The TypeScript types you will actually reach for:

| Event | Type |
|---|---|
| Input/textarea/select change | `React.ChangeEvent<HTMLInputElement>` |
| Form submit | `React.FormEvent<HTMLFormElement>` |
| Click | `React.MouseEvent<HTMLButtonElement>` |
| Key press | `React.KeyboardEvent<HTMLInputElement>` |
| Any handler prop | `React.ChangeEventHandler<HTMLInputElement>` etc. |

Reach the underlying DOM event with `e.nativeEvent` when you need something React doesn't wrap.

#### Controlled inputs

A controlled input has its value driven by state. Every keystroke is a state update; every render sets the value back.

```tsx
function NameField() {
  const [name, setName] = useState('');
  return (
    <input
      value={name}
      onChange={e => setName(e.target.value)}
    />
  );
}
```

The single most common beginner bug here:

```tsx
// ❌ Wrong — value with no onChange: a permanently frozen, read-only input
<input value={name} />

// ✅ Right — either control it…
<input value={name} onChange={e => setName(e.target.value)} />

// ✅ …or don't
<input defaultValue={name} />
```

React warns about the first form explicitly. `defaultValue` (and `defaultChecked`) is the uncontrolled equivalent: it seeds the DOM once and then leaves it alone.

Checkboxes use `checked`/`onChange`, not `value`:

```tsx
<input type="checkbox" checked={agreed} onChange={e => setAgreed(e.target.checked)} />
```

#### `useId` for accessible labels

Never hand-roll ids for label association — they collide the moment a component renders twice.

```tsx
import { useId } from 'react';

function EmailField() {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>Email</label>
      <input id={id} type="email" />
    </>
  );
}
```

`useId` is SSR-safe (server and client agree), which is why it exists rather than a counter. It is **not** for list keys.

<a id="p5-advanced"></a>

### Advanced — propagation, delegation, and uncontrolled inputs

#### Propagation

React events bubble exactly like DOM events, and every handler except `onScroll` bubbles.

```tsx
<div onClick={() => console.log('outer')}>
  <button onClick={e => {
    e.stopPropagation();          // outer handler will NOT fire
    console.log('inner');
  }}>
    Click
  </button>
</div>
```

- `e.stopPropagation()` — stop the event travelling further up.
- `e.preventDefault()` — stop the browser's default behaviour (form submit, link navigation, checkbox toggle).

They are unrelated and frequently confused. A "submit button does nothing but the page reloads" bug is a missing `preventDefault`; a "clicking the delete button also opens the row" bug is a missing `stopPropagation`.

Capture-phase handlers exist for the rare case where a parent must see the event first: `onClickCapture`.

#### How the synthetic system attaches listeners

React does not attach a listener to every element. It attaches one listener per event type at the **root container** and dispatches from there.

```
  document
    └─ #root                ← React attaches its listeners HERE
         └─ <div>
              └─ <button>   ← your onClick lives on the element in the tree,
                              not on the DOM node
```

> **Outdated convention to recognise:** older articles say React attaches its listeners to `document`. That was true through React 16; **React 17 moved delegation to the root container**, which is what makes it safe to run two React versions on one page or embed React inside another app. If you are integrating with non-React code and reading pre-2020 advice about `document`-level handlers fighting React, check the date.

One more thing changed and is now settled: in React 16 and earlier, the synthetic event object was *pooled* and reused, so reading `e.target` after an `await` gave you `null` unless you called `e.persist()`. **Event pooling was removed in React 17.** `e.persist()` still exists as a no-op. Any tutorial that tells you to call it is pre-17.

#### Uncontrolled inputs, and when to prefer them

An uncontrolled input keeps its value in the DOM. You read it when you need it.

```tsx
function Search() {
  const inputRef = useRef<HTMLInputElement>(null);

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    console.log(inputRef.current?.value);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input ref={inputRef} defaultValue="" />
    </form>
  );
}
```

Or, for a whole form, skip refs entirely and use the platform:

```tsx
function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
  e.preventDefault();
  const data = new FormData(e.currentTarget);
  console.log(Object.fromEntries(data));    // { email: '…', password: '…' }
}
```

`FormData` reads every field with a `name` attribute. For a large form where you only care about the values at submit time, this is dramatically less code than twenty `useState`s — and it re-renders zero times while typing.

| Use controlled when | Use uncontrolled when |
|---|---|
| You need to validate or format as the user types | You only need values at submit |
| One field's value affects another | The form is large and simple |
| You disable submit based on contents | Performance matters and fields are numerous |
| The value comes from elsewhere (reset, prefill from server) | You're integrating a non-React widget |

<a id="p5-mastery"></a>

### Mastery — Actions (19+)

React 19 added a layer on top of forms that handles the pending/error/optimistic bookkeeping you would otherwise write by hand. All of this Mastery tier is **19-only**; on React 18 you write the manual version shown in each "before".

#### `<form action>` and `useActionState`

```tsx
// Before (works on 18 and 19) — manual pending + error state
function UpdateName() {
  const [name, setName] = useState('');
  const [error, setError] = useState<string | null>(null);
  const [isPending, setIsPending] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setIsPending(true);
    const err = await updateName(name);
    setIsPending(false);
    setError(err);
  }
  …
}
```

```tsx
// After (19+) — useActionState
import { useActionState } from 'react';

function UpdateName() {
  const [error, submitAction, isPending] = useActionState(
    async (previousState: string | null, formData: FormData) => {
      const err = await updateName(formData.get('name') as string);
      return err;                       // becomes the new state
    },
    null,                               // initial state
  );

  return (
    <form action={submitAction}>
      <input name="name" />
      <button disabled={isPending}>Update</button>
      {error && <p>{error}</p>}
    </form>
  );
}
```

Passing a function to `<form action>` makes React call it with the `FormData`, mark the update as a Transition ([Part 11](./react-performance-concurrency.md)), track pending state, and **reset the form on success**.

> **On React 18** — `<form action={fn}>` sets the HTML `action` attribute to a stringified function and does not work. `useActionState` does not exist. The `useFormState` you may find in React 18 canary/Next.js code was **renamed** to `useActionState` in 19; there is a codemod: `npx codemod@latest react/19/replace-use-form-state`.

#### `useFormStatus`

Lets a *child* component read the enclosing form's pending state without prop-drilling:

```tsx
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? 'Saving…' : 'Save'}</button>;
}
```

It reads the nearest `<form>` **above** it — so `SubmitButton` must be rendered inside the form, not be the component that renders the form.

#### `useOptimistic`

Show the result before the server confirms it, and roll back automatically if it fails.

```tsx
import { useOptimistic } from 'react';

function Thread({ messages, sendMessage }: Props) {
  const [optimisticMessages, addOptimistic] = useOptimistic(
    messages,
    (state: Message[], text: string) => [...state, { text, pending: true }],
  );

  async function formAction(formData: FormData) {
    const text = formData.get('message') as string;
    addOptimistic(text);              // appears instantly
    await sendMessage(text);          // real state arrives; optimistic entry drops
  }

  return (
    <>
      {optimisticMessages.map(m => (
        <p key={m.text}>{m.text}{m.pending && ' (sending…)'}</p>
      ))}
      <form action={formAction}>
        <input name="message" />
      </form>
    </>
  );
}
```

The optimistic value exists only while the action is in flight. When it settles — success or failure — React discards it and you are back to real state. That automatic revert is the part that's genuinely hard to write yourself.

**Try It.** Build the thread above with a `sendMessage` that is `new Promise(r => setTimeout(r, 2000))` and then throws every other call. Expected result: the message appears immediately with "(sending…)", and on the throwing calls it vanishes two seconds later — you did not write a single line of rollback logic.

<a id="part-5-cheat-sheet"></a>

### Part 5 cheat sheet

| Task | Code |
|---|---|
| Attach handler | `onClick={handleClick}` (no parentheses) |
| Handler with argument | `onClick={() => handle(id)}` |
| Stop default browser behaviour | `e.preventDefault()` |
| Stop bubbling | `e.stopPropagation()` |
| Parent sees event first | `onClickCapture` |
| Controlled text input | `value={x} onChange={e => setX(e.target.value)}` |
| Controlled checkbox | `checked={x} onChange={e => setX(e.target.checked)}` |
| Uncontrolled input | `defaultValue` / `defaultChecked` + `ref` |
| Read a whole form | `new FormData(e.currentTarget)` |
| Accessible id | `const id = useId()` |
| Native event | `e.nativeEvent` |

| React 19 form APIs | Purpose |
|---|---|
| `<form action={fn}>` | Call `fn(formData)`; auto pending, auto reset on success |
| `useActionState(fn, initial)` | `[state, dispatch, isPending]` for an async action |
| `useFormStatus()` | `{ pending }` of the nearest form above — from `react-dom` |
| `useOptimistic(real, reducer)` | Provisional value during the action, auto-reverted |

| Symptom | Cause |
|---|---|
| Handler fires on page load | You called it: `onClick={fn()}` |
| Page reloads on submit | Missing `e.preventDefault()` |
| Input won't accept typing | `value` without `onChange` |
| Clicking a child also triggers the parent | Missing `e.stopPropagation()` |
| `e.target` is null after `await` | Pre-React-17 event pooling advice; not a real issue on 17+ |

[↑ Back to top](#table-of-contents)

---

**Previous:** [Parts 1–3 — Foundations](./react-foundations.md) · **Next:** [Parts 6–7 — Effects & Refs](./react-effects-refs.md) · **Up:** [Overview](./react-mastery-guide.md)

*React 19.2.7 · verified September 6, 2026*
