# TanStack Router — Search Params (Part 5)

> The chapter that makes this router different from every other one: `?page=2&sort=newest` as validated, typed, structurally-shared application state.

**Written against:** `@tanstack/react-router` **1.170.x**, `@tanstack/zod-adapter` 1.170.x, Zod 4, TypeScript 5.9+
**Last updated:** September 14, 2026
**Part of:** [TanStack Router Mastery Guide](./tanstack-router-mastery-guide.md) — read its [About This Document](./tanstack-router-mastery-guide.md#about-this-document) before editing this file.

---

## Table of Contents

- [Why this is a whole part](#why)
- [Working Knowledge: `validateSearch` and `useSearch`](#working)
- [Working Knowledge: writing search params](#writing)
- [Advanced: schema libraries and the input/output split](#advanced)
- [Advanced: search middlewares](#middlewares)
- [Mastery: structural sharing, and why it is load-bearing](#mastery)
- [Part 5 cheat sheet](#cheat)
- [Where to go next](#next)

*Tier note: this part **skips the Beginner tier**. There is no meaningful "minimal correct usage" below `validateSearch` — a route either declares a schema or has no typed search params at all, and everything interesting starts one step past that.*

---

<a id="why"></a>
## Why this is a whole part

In most React apps, the query string is a second-class citizen. You reach for `useSearchParams`, get back a `URLSearchParams` of strings, hand-parse `page` into a number, forget the `NaN` case, and end up duplicating that parsing in three components. Meanwhile the *real* state lives in `useState` — which means refreshing the page loses the user's filters, and nobody can share a link to what they are looking at.

TanStack Router inverts this. The [overview](https://tanstack.com/router/latest/docs/framework/react/overview) describes search params as first-class state, "like having `useState` right in the URL!" Concretely, you get four things you do not get elsewhere:

```
  ?page=2&filter=react&tags=%5B%22a%22%2C%22b%22%5D
        │
        ▼
  ① JSON-first parsing        arrays and objects survive the round trip,
                              not just strings
        ▼
  ② validateSearch            one schema per route; the output type is
                              the source of truth everywhere downstream
        ▼
  ③ structural sharing        unchanged sub-objects keep their identity,
                              so React does not re-render the world
        ▼
  ④ typed reads AND writes    Route.useSearch() is typed;
                              <Link search={prev => …}> is typed too
```

Points ② and ④ together are the payoff: the query string becomes as safe to work with as a `useState` value, while remaining shareable, bookmarkable, and restored on refresh for free.

[↑ Back to top](#table-of-contents)

---

<a id="working"></a>
## Working Knowledge: `validateSearch` and `useSearch`

`validateSearch` is a route option. It receives JSON-parsed but unvalidated search params typed as `Record<string, unknown>` and returns your typed object ([search params guide](https://tanstack.com/router/latest/docs/framework/react/guide/search-params)):

```tsx
type ProductSearchSortOptions = 'newest' | 'oldest' | 'price'

type ProductSearch = {
  page: number
  filter: string
  sort: ProductSearchSortOptions
}

export const Route = createFileRoute('/shop/products')({
  validateSearch: (search: Record<string, unknown>): ProductSearch => {
    return {
      page: Number(search?.page ?? 1),
      filter: (search.filter as string) || '',
      sort: (search.sort as ProductSearchSortOptions) || 'newest',
    }
  },
})
```

Reading is then one hook, fully typed:

```tsx
function Products() {
  const { page, filter, sort } = Route.useSearch()
  //      ^number  ^string  ^'newest' | 'oldest' | 'price'
}
```

From outside the route file, the same two options as with params: `getRouteApi('/shop/products').useSearch()` for strict typing, or `useSearch({ strict: false })` for genuinely route-agnostic components, which returns the properties as optional.

**The return type is the contract.** `ProductSearch` is not just what `useSearch` returns — it is also what `<Link search={...}>` demands, what `loaderDeps` receives, and what the router serialises back into the URL. Getting the schema right once fixes all of them.

> **Wrong vs. right — the thing you are replacing.**
>
> ```tsx
> // ❌ WRONG — the pre-TanStack habit, repeated in every component
> const [sp] = useSearchParams()
> const page = parseInt(sp.get('page') ?? '1', 10)  // NaN if someone types ?page=abc
> const sort = sp.get('sort') ?? 'newest'           // typed as string, not the union
>
> // ✅ RIGHT — declared once on the route, typed everywhere
> const { page, sort } = Route.useSearch()
> ```

**🧪 Try It — see JSON-first parsing.**

Give a route this schema:

```tsx
validateSearch: (s: Record<string, unknown>) => ({
  tags: (s.tags as string[]) ?? [],
  page: Number(s.page ?? 1),
}),
```

…then render `<Link to="/x" search={{ tags: ['react', 'ts'], page: 2 }}>Go</Link>` and click it.

**Expected output:** the address bar shows the array JSON-encoded (something like `?tags=%5B%22react%22%2C%22ts%22%5D&page=2`), and `Route.useSearch().tags` is a real `string[]` of length 2 — not `"react,ts"`, and not a string that you have to split. Try the same with `URLSearchParams` and you will be writing a delimiter convention and a parser.

[↑ Back to top](#table-of-contents)

---

<a id="writing"></a>
## Working Knowledge: writing search params

Search params are updated **functionally**, like `setState`. The `search` prop on `<Link>` takes either an object or an updater receiving the previous search:

```tsx
<Link from={Route.fullPath} search={(prev) => ({ page: prev.page + 1 })}>
  Next Page
</Link>
```

`navigate()`, `router.navigate()` and `<Navigate>` all accept the identical `search` option — the docs are explicit that these are the same interface.

```tsx
const navigate = useNavigate({ from: Route.fullPath })

navigate({ search: (prev) => ({ ...prev, sort: 'price' }) })
```

**The `from` prop matters more here than anywhere else.** It is what tells TypeScript which schema `prev` conforms to. Without it, the router assumes you are navigating from `/`, and `prev` has no useful type.

> **Wrong vs. right — the dropped-params bug.**
>
> ```tsx
> // ❌ WRONG — sets page, silently wipes filter and sort
> <Link from={Route.fullPath} search={{ page: 2 }}>Page 2</Link>
>
> // ✅ RIGHT — spread what you are not changing
> <Link from={Route.fullPath} search={(prev) => ({ ...prev, page: 2 })}>Page 2</Link>
> ```
>
> The object form **replaces** the search state; it does not merge. This is consistent (it mirrors `setState` with an object vs. an updater), but it catches nearly everyone once. If a route's schema has required fields, TypeScript will catch the first form for you — which is a good argument for *not* making every field optional. If your schema is entirely optional fields, nothing will warn you, and the filters just vanish when the user paginates.
>
> The structural fix for cross-route cases is [search middleware](#middlewares) — `retainSearchParams` — rather than remembering to spread at every call site.

[↑ Back to top](#table-of-contents)

---

<a id="advanced"></a>
## Advanced: schema libraries and the input/output split

Hand-written validators get unpleasant fast. The router ships adapters for the standard schema libraries; the Zod one is `@tanstack/zod-adapter`.

```tsx
import { z } from 'zod'
import { createFileRoute } from '@tanstack/react-router'

const productSearchSchema = z.object({
  page: z.number().catch(1),
  filter: z.string().catch(''),
  sort: z.enum(['newest', 'oldest', 'price']).catch('newest'),
})

export const Route = createFileRoute('/shop/products')({
  validateSearch: productSearchSchema,
})
```

**Zod version note.** Per the search params guide: for **Zod v3** you wrap the schema with `zodValidator()` from `@tanstack/zod-adapter`; **Zod v4** schemas work directly in `validateSearch`. If you are on Zod 4 and reading a tutorial that wraps everything in `zodValidator`, it is targeting v3 — harmless, but unnecessary.

**`.catch()` over `.default()` — this is the important recommendation.** The docs state it directly: *"if a search parameter is malformed, you probably don't want to halt the user's experience."*

```tsx
// ❌ WRONG for most cases — a malformed ?page=abc throws, and the user
//    gets an error boundary instead of the page they asked for
page: z.number().default(1)

// ✅ RIGHT — a malformed value falls back and the page renders
page: z.number().catch(1)
```

The difference: `.default()` supplies a value when the key is **missing**, but still *throws* when the key is present and invalid. `.catch()` supplies a value in **both** cases. Since search params are user-editable text in an address bar — and, worse, are pasted, truncated by chat clients, and mangled by email link rewriters — assume malformed input is normal traffic.

Use `.default()` deliberately and only when a bad value genuinely *should* block navigation. A `?impersonateUserId=` that fails validation is a case where erroring out is correct.

**The input/output split.** A schema can transform, which means the type you *write* and the type you *read* differ:

```tsx
const schema = z.object({
  page: z.number().catch(1),
  createdAfter: z.string().transform((s) => new Date(s)).catch(new Date(0)),
})
```

Here `useSearch()` gives you a `Date`, but a `<Link search={...}>` should accept a `string`. The adapter exists precisely for this: the docs describe it as ensuring "correct input/output type inference for navigation versus reading." Without an adapter, the router would infer one type for both and you would end up casting at every link.

> **The practical rule:** use the adapter whenever your schema does anything beyond straight validation — transforms, coercion, `.pipe()`. For plain validating schemas the distinction is invisible and you will not notice either way.

[↑ Back to top](#table-of-contents)

---

<a id="middlewares"></a>
## Advanced: search middlewares

Middlewares transform search params **when links are built and when navigation happens** — not when they are read. They are the structural answer to "every link in the app needs to carry this param" and "stop putting defaults in the URL."

```tsx
import { retainSearchParams, stripSearchParams } from '@tanstack/react-router'

const defaultValues = { arrayWithDefaults: ['foo', 'bar'] }

export const Route = createFileRoute('/x')({
  validateSearch: schema,
  search: {
    middlewares: [
      retainSearchParams(['retainMe']),
      stripSearchParams({ arrayWithDefaults: defaultValues.arrayWithDefaults }),
    ],
  },
})
```

The two built-ins solve the two recurring problems:

**`retainSearchParams(keys)`** — per the docs, it makes sure "that for every link that is being built, the `rootValue` search param is added if it is part of the current search params." This is the real fix for global, cross-cutting params: a `?theme=dark`, a `?tenant=acme`, a `?debug=1`. Declare it once on the root or a layout route and every link inside that subtree carries it automatically.

**`stripSearchParams(defaults)`** — removes params whose value equals the default. This keeps `/products` clean instead of `/products?page=1&filter=&sort=newest`, while `useSearch()` still hands you `{ page: 1, filter: '', sort: 'newest' }` because `validateSearch` fills them back in. Users share shorter links; you lose nothing.

Middlewares chain in array order.

> **Wrong vs. right — the manual version of `retainSearchParams`.**
>
> ```tsx
> // ❌ WRONG — every link in the app has to remember this, forever
> <Link to="/posts" search={(prev) => ({ theme: prev.theme })}>Posts</Link>
> <Link to="/about" search={(prev) => ({ theme: prev.theme })}>About</Link>
>
> // ✅ RIGHT — declare it once on the route that owns the param
> // __root.tsx
> search: { middlewares: [retainSearchParams(['theme'])] }
> // then, everywhere:
> <Link to="/posts">Posts</Link>
> ```

**🧪 Try It — strip and retain, together.**

On a route with `validateSearch` giving `{ page: number; theme: string }` and:

```tsx
search: {
  middlewares: [retainSearchParams(['theme']), stripSearchParams({ page: 1 })],
}
```

Navigate to `?page=3&theme=dark`, then click a `<Link search={(p) => ({ ...p, page: 1 })}>First page</Link>`.

**Expected output:** the URL becomes `?theme=dark` — `page=1` is stripped because it equals the default, `theme` is retained without you passing it. And `Route.useSearch()` still returns `{ page: 1, theme: 'dark' }`. The URL is minimal; the state is complete.

[↑ Back to top](#table-of-contents)

---

<a id="mastery"></a>
## Mastery: structural sharing, and why it is load-bearing

Here is a problem that only shows up once search params are real state.

Every time the URL changes, the router re-parses the query string. The search params guide names the consequence: *"each new parse creates a brand new data structure with a unique memory reference."* Naively, that means:

```
  URL change: ?page=1&filters={"tags":["a"],"author":"sam"}
           →  ?page=2&filters={"tags":["a"],"author":"sam"}

  Naive parse produces:
     search        → NEW object reference
     search.filters→ NEW object reference   ← even though it is deeply equal
     search.filters.tags → NEW array reference

  In React, every component subscribed to `filters` re-renders,
  every useEffect([filters]) fires, every useMemo([filters]) recomputes.
  Paginating a list re-runs your entire filter pipeline for no reason.
```

**Structural sharing** is the fix: after parsing, the router reconciles the new object against the previous one and **reuses the old reference for any sub-tree that is deeply equal**. So in the example above, `search` is a new reference (it genuinely changed) but `search.filters` and `search.filters.tags` are *the same references as before*.

```
  With structural sharing:

     search        → NEW       (page changed — correct)
     search.filters→ SAME ref  (deeply equal — reused)
     search.filters.tags → SAME ref
```

This is why the JSON-first design and the validation requirement are not separable features. You can only reconcile structures if you *have* structures — which requires JSON-first parsing — and you can only trust the comparison if the shape is stable, which is what `validateSearch` guarantees. The docs tie these together directly: the JSON-first approach "with proper validation preserves referential integrity across serialization cycles, critical for frameworks like React that track changes via immutability."

**What this buys you in practice:**

- `useEffect(..., [search.filters])` fires when filters actually change, not on every navigation.
- A memoised expensive child subscribed to one slice of search does not re-render when a different slice changes.
- `loaderDeps` comparison (see [Part 6](./tanstack-router-data-loading.md#part-6)) is meaningful — this is the mechanism that lets the router decide a loader does *not* need to re-run.

**The connection to `loaderDeps`.** This is the piece people miss. `loaderDeps` selects which search params a loader depends on, and the router compares the selected object to decide whether to re-run. Structural sharing is what makes that comparison stable:

```tsx
// ✅ Select narrowly. Structural sharing then guarantees this object is
//    reference-stable across any navigation that does not change offset/limit.
loaderDeps: ({ search: { offset, limit } }) => ({ offset, limit }),
```

Selecting the whole search object defeats it — covered as a full wrong/right pair in [Part 6](./tanstack-router-data-loading.md#part-6), because it manifests as a data-fetching bug rather than a search-params one.

[↑ Back to top](#table-of-contents)

---

<a id="cheat"></a>
## Part 5 cheat sheet

| Task | API |
|---|---|
| Declare the schema | `validateSearch: (search: Record<string, unknown>) => T` |
| Zod 4 | pass the schema directly to `validateSearch` |
| Zod 3 | wrap with `zodValidator()` from `@tanstack/zod-adapter` |
| Schema with transforms | use the adapter, so navigate-input and read-output types differ correctly |
| Tolerant fallback | `z.number().catch(1)` — **preferred** for user-facing params |
| Strict fallback | `z.number().default(1)` — throws on a present-but-invalid value |
| Read (strict) | `Route.useSearch()` |
| Read (other file) | `getRouteApi('/path').useSearch()` |
| Read (route-agnostic) | `useSearch({ strict: false })` — all optional |
| Write (replace) | `<Link search={{ page: 2 }}>` |
| Write (merge) | `<Link search={(prev) => ({ ...prev, page: 2 })}>` |
| Carry a param everywhere | `search: { middlewares: [retainSearchParams(['theme'])] }` |
| Keep defaults out of the URL | `search: { middlewares: [stripSearchParams({ page: 1 })] }` |
| Connect search to data | `loaderDeps: ({ search }) => ({ /* narrow selection */ })` |

**Three rules to carry forward**

1. `.catch()` not `.default()` for anything a user can type or paste.
2. The object form of `search` replaces; the updater form merges. Prefer the updater.
3. Middlewares belong on the route that *owns* the param, not on every link that passes through it.

[↑ Back to top](#table-of-contents)

---

<a id="next"></a>
## Where to go next

- **Next in sequence:** [Data Loading (Parts 6–7)](./tanstack-router-data-loading.md) — `loaderDeps` picks up exactly where this part ends.
- **Related:** `<Link search>` typing depends on `from`, covered in [Part 4](./tanstack-router-routing-navigation.md#p4-working).
- **Back to:** [Overview](./tanstack-router-mastery-guide.md)

### Quick self-check for Part 5

1. Why does the guide recommend `.catch()` over `.default()`, and when is `.default()` the right call?
2. `<Link search={{ page: 2 }}>` loses the user's filters. Why, and what are the two fixes (one local, one structural)?
3. What does `from` do for the *types* of a `search` updater?
4. `?page=1&sort=newest` is ugly but `useSearch()` needs those values. Which middleware solves this, and why does reading still work?
5. Explain structural sharing in terms of object references, and name one bug it prevents that you would otherwise debug as "my effect fires too often".
6. Your schema uses `.transform()` to produce a `Date`. Why do you now need the adapter?

[↑ Back to top](#table-of-contents)
