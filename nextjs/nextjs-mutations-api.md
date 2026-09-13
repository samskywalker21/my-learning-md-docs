# Next.js — Mutations & API Layer (Parts 6–7)

Writing data with Server Actions and forms, and knowing when you need a real HTTP endpoint instead.

> **Spec:** this doc follows the shared spec in [`nextjs-mastery-guide.md`](./nextjs-mastery-guide.md#about-this-document). Written against **Next.js 16.3.4**, verified **September 11, 2026**.

The single most important idea in this doc: **a Server Action is a public POST endpoint.** It happens to be written next to your UI, but anyone who can find the action ID can invoke it without ever loading your page. Everything in [§6.4](#64-mastery--the-security-model) follows from that.

---

## Table of Contents

- [Part 6: Mutations — Server Actions and Forms](#part-6-mutations-server-actions-and-forms)
  - [6.1 Beginner — a form that writes to the database](#61-beginner--a-form-that-writes-to-the-database)
  - [6.2 Working Knowledge — validation, pending states, errors](#62-working-knowledge--validation-pending-states-errors)
  - [6.3 Advanced — optimistic UI, cookies, event handlers](#63-advanced--optimistic-ui-cookies-event-handlers)
  - [6.4 Mastery — the security model](#64-mastery--the-security-model)
  - [6.5 Cheat sheet: Part 6](#65-cheat-sheet-part-6)
- [Part 7: Route Handlers](#part-7-route-handlers)
  - [7.1 Beginner — `route.ts` and the Web APIs](#71-beginner--routets-and-the-web-apis)
  - [7.2 Working Knowledge — params, cookies, caching](#72-working-knowledge--params-cookies-caching)
  - [7.3 Advanced — webhooks, streaming, uploads](#73-advanced--webhooks-streaming-uploads)
  - [7.4 Cheat sheet: Part 7](#74-cheat-sheet-part-7)

---

## Part 6: Mutations — Server Actions and Forms

<a id="part-6-mutations-server-actions-and-forms"></a>

### 6.1 Beginner — a form that writes to the database

A **Server Function** is an async function marked `'use server'` that runs on the server and can be called from the client over the network. Used for a form submission or a mutation, it's called a **Server Action** ([Mutating data](https://nextjs.org/docs/app/getting-started/mutating-data)).

```ts
// app/lib/actions.ts
'use server'

import { db, posts } from '@/lib/db'
import { revalidatePath } from 'next/cache'
import { redirect } from 'next/navigation'
import { verifySession } from '@/lib/dal'

export async function createPost(formData: FormData) {
  const session = await verifySession()        // ← always. see §6.4

  await db.insert(posts).values({
    title: String(formData.get('title')),
    content: String(formData.get('content')),
    authorId: session.userId,
  })

  revalidatePath('/posts')
  redirect('/posts')
}
```

```tsx
// app/posts/new/page.tsx — a Server Component. No 'use client' anywhere.
import { createPost } from '@/app/lib/actions'

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" required />
      <textarea name="content" required />
      <button type="submit">Create</button>
    </form>
  )
}
```

That is a complete write path: no API route, no `fetch`, no client JavaScript. React extends `<form>` so that `action` accepts a function; the framework POSTs to the current route, runs the action, and returns the updated UI.

Three things are happening that are worth naming:

1. **Progressive enhancement.** A form in a Server Component submits even if JavaScript hasn't loaded or is disabled ([Mutating data](https://nextjs.org/docs/app/getting-started/mutating-data#server-components)).
2. **One roundtrip.** `revalidatePath` invalidates the cache *and* the response carries a freshly rendered RSC payload for the current route ([Server Actions](https://nextjs.org/docs/app/guides/server-actions#a-single-response-carries-data-and-ui)). No follow-up fetch.
3. **`redirect` throws.** It raises a framework control-flow exception, so nothing after it runs — and revalidation calls must come *before* it ([Redirect after a mutation](https://nextjs.org/docs/app/getting-started/mutating-data#redirect-after-a-mutation)).

**Two placements of the directive:**

```ts
// File-level: every export in this file is a Server Function.
'use server'
export async function createPost(formData: FormData) { /* … */ }
export async function deletePost(id: string) { /* … */ }
```

```tsx
// Function-level: inline inside a Server Component.
export default function Page() {
  async function createPost(formData: FormData) {
    'use server'
    // …
  }
  return <form action={createPost}>{/* … */}</form>
}
```

You **cannot** define a Server Function inside a Client Component. You import it from a `'use server'` file, or receive it as a prop ([Client Components](https://nextjs.org/docs/app/getting-started/mutating-data#client-components)).

### 6.2 Working Knowledge — validation, pending states, errors

Real forms need to show errors without losing the user's input. The tool is `useActionState`, which threads a state value through the action.

The action takes `(prevState, formData)` and **returns** errors rather than throwing them:

```ts
// app/lib/actions.ts
'use server'

import * as z from 'zod'
import { revalidatePath } from 'next/cache'
import { verifySession } from '@/lib/dal'

const PostSchema = z.object({
  title: z.string().min(3, { error: 'Title must be at least 3 characters.' }).trim(),
  content: z.string().min(1, { error: 'Content is required.' }).trim(),
})

export type PostFormState = {
  errors?: { title?: string[]; content?: string[] }
  message?: string
} | undefined

export async function createPost(
  _prev: PostFormState,
  formData: FormData
): Promise<PostFormState> {
  const session = await verifySession()

  const parsed = PostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
  })

  if (!parsed.success) {
    return { errors: parsed.error.flatten().fieldErrors }
  }

  try {
    await db.insert(posts).values({ ...parsed.data, authorId: session.userId })
  } catch {
    return { message: 'Could not save the post. Please try again.' }
  }

  revalidatePath('/posts')
  return { message: 'Saved.' }
}
```

```tsx
// app/posts/new/form.tsx
'use client'

import { useActionState } from 'react'
import { createPost, type PostFormState } from '@/app/lib/actions'

const initialState: PostFormState = undefined

export function PostForm() {
  const [state, action, pending] = useActionState(createPost, initialState)

  return (
    <form action={action}>
      <label htmlFor="title">Title</label>
      <input id="title" name="title" />
      {state?.errors?.title && <p aria-live="polite">{state.errors.title[0]}</p>}

      <label htmlFor="content">Content</label>
      <textarea id="content" name="content" />
      {state?.errors?.content && <p aria-live="polite">{state.errors.content[0]}</p>}

      {state?.message && <p aria-live="polite">{state.message}</p>}
      <button disabled={pending}>{pending ? 'Saving…' : 'Create Post'}</button>
    </form>
  )
}
```

**Wrong vs. right — expected errors:**

```ts
// ✗ Throwing a validation error hits the error boundary and blows away the form.
if (!parsed.success) throw new Error('Invalid input')

// ✓ Expected errors are return values. Throwing is for bugs.
if (!parsed.success) return { errors: parsed.error.flatten().fieldErrors }
```

The official framing: *"For these errors, avoid using `try`/`catch` blocks and throw errors. Instead, model expected errors as return values"* ([Error handling](https://nextjs.org/docs/app/getting-started/error-handling#server-functions)). Uncaught exceptions are for genuine failures, and in production their messages are redacted to a `digest` hash anyway — useless as user-facing text.

**`useActionState` outside a form.** You can dispatch it from a button, but it must go through a transition:

```tsx
'use client'
import { useActionState, startTransition } from 'react'

export function PublishButton({ id }: { id: string }) {
  const [state, action, pending] = useActionState(publishPost, null)
  return (
    <button onClick={() => startTransition(() => action(id))} disabled={pending}>
      {pending ? 'Publishing…' : 'Publish'}
    </button>
  )
}
```

**Passing extra arguments.** `FormData` only carries form fields, so bind anything else:

```tsx
// ✗ Hidden inputs are user-editable. Never trust an ID from one.
<input type="hidden" name="postId" value={post.id} />

// ✓ bind() — the value is encrypted into the action reference, and the
//   action must STILL verify the caller owns it (see §6.4).
const deleteWithId = deletePost.bind(null, post.id)
;<form action={deleteWithId}><button>Delete</button></form>
```

### 6.3 Advanced — optimistic UI, cookies, event handlers

**Optimistic updates** with `useOptimistic` make a mutation feel instant:

```tsx
'use client'

import { useOptimistic, startTransition } from 'react'
import { addTodo } from '@/app/lib/actions'

export function TodoList({ todos }: { todos: Todo[] }) {
  const [optimisticTodos, addOptimistic] = useOptimistic(
    todos,
    (state: Todo[], newTitle: string) => [
      ...state,
      { id: `temp-${Date.now()}`, title: newTitle, done: false, pending: true },
    ]
  )

  async function handleSubmit(formData: FormData) {
    const title = String(formData.get('title'))
    startTransition(() => addOptimistic(title))
    await addTodo(title)
  }

  return (
    <>
      <ul>
        {optimisticTodos.map((t) => (
          <li key={t.id} style={{ opacity: t.pending ? 0.5 : 1 }}>{t.title}</li>
        ))}
      </ul>
      <form action={handleSubmit}>
        <input name="title" />
        <button>Add</button>
      </form>
    </>
  )
}
```

The optimistic state reverts automatically when the action settles and the real data arrives. The one rule: the optimistic update must go inside a transition, or React throws.

**Cookies in actions.** Setting or deleting a cookie **automatically re-renders the current page** so the UI reflects the new value ([Cookies](https://nextjs.org/docs/app/getting-started/mutating-data#cookies)):

```ts
'use server'
import { cookies } from 'next/headers'

export async function setTheme(theme: 'light' | 'dark') {
  const store = await cookies()
  store.set('theme', theme, { httpOnly: false, sameSite: 'lax', path: '/' })
  // No revalidate call needed — the cookie write triggers the re-render.
}
```

Client state is preserved across that re-render for components that stay mounted.

**Calling actions from event handlers and effects.** Legitimate, and useful for things that aren't form submissions:

```tsx
'use client'
import { incrementViews } from './actions'
import { useEffect, useTransition, useState } from 'react'

export function ViewCount({ initial }: { initial: number }) {
  const [views, setViews] = useState(initial)
  const [, startTransition] = useTransition()

  useEffect(() => {
    startTransition(async () => setViews(await incrementViews()))
  }, [])

  return <p>{views} views</p>
}
```

**The parallelism limit.** Next.js dispatches Server Actions **one at a time per client** ([Sequential dispatch](https://nextjs.org/docs/app/guides/server-actions#sequential-dispatch-on-the-client)):

```ts
// ✗ These do NOT run in parallel. The client queues them; each waits
//   for the previous to complete AND for its route re-render.
await Promise.all([saveA(), saveB(), saveC()])

// ✓ Do the parallel work inside one action.
export async function saveAll(data: Payload) {
  'use server'
  await Promise.all([saveA(data.a), saveB(data.b), saveC(data.c)])
}
```

This is a property of the client dispatcher — it keeps the re-rendered server tree consistent with the action that produced it. Server-side, an action is a normal async function and can do whatever it likes.

> **Real Scenario — the bulk-delete that took 90 seconds.**
> An admin UI had a "delete selected" button that mapped over checked rows and called `deleteItem(id)` for each. With 40 rows selected it took a minute and a half, and the UI flickered the whole time.
> **Cause:** 40 sequentially dispatched actions, each of which also triggered `revalidatePath('/admin/items')` — so the framework re-rendered and re-streamed the entire admin table 40 times.
> **Fix:** one `deleteItems(ids: string[])` action doing a single `DELETE … WHERE id = ANY($1)` and one `revalidatePath`. 90 seconds became 300ms.
> **The lesson:** an action's granularity should match the *user's* intent, not the data model's. "Delete these 40 things" is one intent.

### 6.4 Mastery — the security model

> Server Functions are reachable via direct POST requests, not just through your application's UI. Always verify authentication and authorization inside every Server Function. — [Mutating data](https://nextjs.org/docs/app/getting-started/mutating-data)

Here is what actually happens at build time. The `'use server'` directive tells the compiler to replace the function's body, in client bundles, with a **reference**: an action ID plus a dispatcher that POSTs back to the server. The implementation never ships. But the endpoint exists, and it answers to anyone with the ID ([Server Actions — security](https://nextjs.org/docs/app/guides/server-actions#security)).

**What the framework gives you:**

| Protection | Detail |
|---|---|
| CSRF check | `Origin` is compared to `Host`/`X-Forwarded-Host`; mismatches rejected. Configure `serverActions.allowedOrigins` behind a proxy or CDN |
| Body size limit | 1 MB by default; raise with `serverActions.bodySizeLimit` |
| Encrypted action IDs | Rotated at most every 14 days, even if the source is unchanged |
| Dead code elimination | Unused Server Functions are stripped, so they have no public endpoint |
| Closure encryption | Variables captured by an inline action are encrypted before reaching the client |

**What is entirely yours:** authentication, authorisation, input validation, and constraining what you return.

**Wrong vs. right — the ownership check that isn't:**

```ts
// ✗ The client supplies the whole record, including its id. Anyone who can
//   POST here can mark ANY item complete. Zod validates the shape, not the
//   right to touch the row.
'use server'
export async function completeItemUnsafe(item: Item) {
  await db.update(items).set({ completed: true }).where(eq(items.id, item.id))
}

// ✓ Take only the reference. Derive identity from the session.
//   Look the row up BY OWNERSHIP.
'use server'
export async function completeItem(itemId: string) {
  const session = await verifySession()
  if (!session) return

  const item = await db.query.items.findFirst({
    where: and(eq(items.id, itemId), eq(items.ownerId, session.userId)),
  })
  if (!item) return   // not found, or not yours — same answer either way

  await db.update(items).set({ completed: true }).where(eq(items.id, item.id))
}
```

Source: [Server Actions — security](https://nextjs.org/docs/app/guides/server-actions#security). The principle: **the client tells you *which* thing; the session tells you *who*; the database tells you whether those two go together.**

**Rendering is not authorisation.** "This form only renders on an authenticated page" is not a boundary — the POST doesn't go through your page. Similarly, a layout that conditionally hides a route does not stop that route from rendering or from appearing in the RSC payload ([Authentication](https://nextjs.org/docs/app/guides/authentication#layouts-and-auth-checks)).

**Return values are serialised to the client.** Returning a raw database row publishes every column, password hash included. Shape returns to what the UI renders — the DTO discipline in [Part 9](./nextjs-database-auth.md#part-9-authentication-and-sessions).

**Destructive operations deserve more.** Deletes and permission changes warrant an elevated session check or re-authentication, and should fail loudly if a check is missed. If you enable the experimental [`authInterrupts`](https://nextjs.org/docs/app/api-reference/config/next-config-js/authInterrupts) flag you can `throw unauthorized()` / `forbidden()` from `next/navigation` and let Next.js render `unauthorized.tsx` / `forbidden.tsx`.

**Multi-instance deployments need a stable encryption key.** Closure variables are encrypted with a key generated per build; with several replicas, an action encrypted by one instance can't be decrypted by another, producing *"Failed to find Server Action"* ([Self-hosting](https://nextjs.org/docs/app/guides/self-hosting#server-functions-encryption-key)):

```bash
NEXT_SERVER_ACTIONS_ENCRYPTION_KEY=<base64, 32 bytes> next build
```

Full deployment treatment in [Part 13](./nextjs-production-deployment.md#part-13-deployment-and-internals).

> **Real Scenario — the deploy that broke every open tab.**
> A team deployed mid-afternoon. Within minutes, support filled with "the save button does nothing." Reloading fixed it. The logs showed *Failed to find Server Action*.
> **Cause:** action IDs are part of the build artifacts and change on deploy. Every browser tab still running the previous build was POSTing IDs the new server had never heard of.
> **Fix, in three parts.** `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` was pinned so references stayed decryptable across instances; `deploymentId` was set so the router detects version skew and does a hard navigation instead of failing; and the global error handler was taught to recognise this specific error and prompt a refresh rather than showing a generic failure ([Deployment considerations](https://nextjs.org/docs/app/guides/server-actions#deployment-considerations)).

### 6.5 Cheat sheet: Part 6

| Goal | Code |
|---|---|
| Define an action | `'use server'` at the top of the file or the function body |
| Call from a form | `<form action={createPost}>` |
| Call from a Client Component | import it from a `'use server'` file |
| Bind extra args | `action.bind(null, id)` |
| Pending + error state | `const [state, action, pending] = useActionState(fn, init)` |
| Optimistic UI | `useOptimistic(current, reducer)` + `startTransition` |
| Call from `onClick` | `startTransition(() => action(arg))` |
| Redirect after | `redirect('/posts')` — after revalidation, never before |
| User must see their change | `updateTag(tag)` |
| Others can see it eventually | `revalidateTag(tag, 'max')` |
| Refresh uncached view data | `refresh()` |

| Config | Default | Where |
|---|---|---|
| `serverActions.bodySizeLimit` | `'1mb'` | `next.config.ts` → `experimental` |
| `serverActions.allowedOrigins` | `[]` | `next.config.ts` → `experimental` |
| `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` | generated per build | build environment |

| Symptom | Cause |
|---|---|
| "Failed to find Server Action" | Stale client after deploy, or mismatched encryption keys |
| Form clears on validation error | Throwing instead of returning the error |
| Actions run one after another | Sequential dispatch — combine into one action |
| Body too large | 1 MB default; use a Route Handler for uploads ([Part 7](#part-7-route-handlers)) |
| Works locally, CSRF-rejected in prod | Add your proxy domain to `allowedOrigins` |

[↑ Back to top](#table-of-contents)

---

## Part 7: Route Handlers

<a id="part-7-route-handlers"></a>

Route Handlers get **Beginner → Advanced** only. Their "Mastery" material is really the security and caching content already covered in Parts 5, 6 and 9, applied to a different entry point.

### 7.1 Beginner — `route.ts` and the Web APIs

A Route Handler is a custom HTTP endpoint defined by a `route.ts` file, built on the Web `Request` and `Response` APIs ([Route Handlers](https://nextjs.org/docs/app/getting-started/route-handlers)):

```ts
// app/api/health/route.ts  →  GET /api/health
export async function GET() {
  return Response.json({ ok: true })
}
```

Supported methods: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`. Anything else gets a 405.

**Two hard rules:**

1. `route.ts` and `page.tsx` cannot live in the same segment — each file takes over every HTTP verb for its route.
2. Route Handlers don't participate in layouts or client-side navigation. They are the lowest-level routing primitive.

**When do you actually need one?** This is the question people get wrong, so:

| Use a **Server Action** | Use a **Route Handler** |
|---|---|
| Form submissions | Incoming webhooks (Stripe, GitHub, Clerk) |
| Mutations triggered by your own UI | Requests from a mobile app or third-party client |
| Anything that should re-render the current page | File uploads over 1 MB |
| Optimistic updates | Streaming responses (LLM tokens, SSE) |
| | Cron/scheduled job targets |
| | OAuth callbacks |
| | Anything that must be a specific URL with a specific shape |

The heuristic: **if the caller is your own React tree, use an Action. If the caller is something else on the internet, use a Handler.** A new Next.js app can go a long way with zero Route Handlers — which surprises people coming from `pages/api`.

### 7.2 Working Knowledge — params, cookies, caching

**Dynamic segments** work the same as pages, and there's a generated type for the context object:

```ts
// app/api/users/[id]/route.ts
import type { NextRequest } from 'next/server'

export async function GET(_req: NextRequest, ctx: RouteContext<'/api/users/[id]'>) {
  const { id } = await ctx.params        // async, like everywhere in 16
  const user = await db.query.users.findFirst({ where: eq(users.id, id) })
  if (!user) return new Response(null, { status: 404 })
  return Response.json({ id: user.id, name: user.name })   // DTO, not the raw row
}
```

**Query params, body, cookies, headers:**

```ts
import { cookies, headers } from 'next/headers'
import { NextResponse, type NextRequest } from 'next/server'

export async function POST(request: NextRequest) {
  const query = request.nextUrl.searchParams.get('q')   // ?q=…
  const body = await request.json()                     // Web Request: it's a stream
  const session = (await cookies()).get('session')?.value
  const agent = (await headers()).get('user-agent')

  const res = NextResponse.json({ ok: true })
  res.cookies.set('last-seen', new Date().toISOString(), { httpOnly: true })
  return res
}
```

Note the two ways to touch cookies: `cookies()` from `next/headers` reads the request; `NextResponse.cookies.set` writes on the response.

**Caching.** Route Handlers are **not cached by default**. `GET` can opt in; other methods never cache, even alongside a cached `GET` in the same file ([Route Handlers — caching](https://nextjs.org/docs/app/getting-started/route-handlers#caching)):

```ts
export const dynamic = 'force-static'

export async function GET() {
  const res = await fetch('https://api.example.com/items')
  return Response.json({ data: await res.json() })
}
```

Under Cache Components, `GET` handlers follow the same prerendering model as pages: they run at request time by default, prerender when they touch nothing runtime, and can use `use cache` — **but not directly in the handler body**; extract a helper ([With Cache Components](https://nextjs.org/docs/app/getting-started/route-handlers#with-cache-components)):

```ts
import { cacheLife } from 'next/cache'

export async function GET() {
  return Response.json(await getProducts())
}

async function getProducts() {
  'use cache'
  cacheLife('hours')
  return db.select().from(products)
}
```

**Authorisation is identical to Server Actions** — treat a handler as a public endpoint ([Authentication — Route Handlers](https://nextjs.org/docs/app/guides/authentication#route-handlers)):

```ts
export async function GET() {
  const session = await verifySession()
  if (!session) return new Response(null, { status: 401 })
  if (session.role !== 'admin') return new Response(null, { status: 403 })
  // …
}
```

### 7.3 Advanced — webhooks, streaming, uploads

**Webhooks** are the canonical Route Handler. The non-obvious part is that signature verification needs the **raw body**, so you cannot parse it first:

```ts
// app/api/webhooks/stripe/route.ts
import { headers } from 'next/headers'
import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!)

export async function POST(request: Request) {
  const body = await request.text()   // ← text(), not json()
  const signature = (await headers()).get('stripe-signature')

  let event: Stripe.Event
  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature!,
      process.env.STRIPE_WEBHOOK_SECRET!
    )
  } catch (err) {
    return new Response('Invalid signature', { status: 400 })
  }

  switch (event.type) {
    case 'checkout.session.completed':
      await fulfilOrder(event.data.object)
      break
  }

  return new Response(null, { status: 200 })
}
```

Three things to get right, and they're the same three every time:

1. **`await request.text()` before verifying.** Calling `.json()` first consumes the stream and you lose the exact bytes the signature covers.
2. **Return 200 fast.** Providers retry on timeout; slow handlers cause duplicate deliveries. Push slow work to a queue, or to `after()`:
   ```ts
   import { after } from 'next/server'
   after(async () => { await sendReceiptEmail(order) })   // runs post-response
   ```
   `after` is fully supported self-hosted with `next start` — but your shutdown must drain properly, or pending callbacks are lost ([Self-hosting — `after`](https://nextjs.org/docs/app/guides/self-hosting#after)).
3. **Make it idempotent.** You *will* receive the same event twice. Store the provider's event ID and no-op on repeats.

**Streaming responses** — for LLM output, progress, or server-sent events:

```ts
export async function GET() {
  const encoder = new TextEncoder()
  const stream = new ReadableStream({
    async start(controller) {
      for (const chunk of await generateChunks()) {
        controller.enqueue(encoder.encode(`data: ${chunk}\n\n`))
      }
      controller.close()
    },
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache, no-transform',
      Connection: 'keep-alive',
    },
  })
}
```

Self-hosted, this needs the same end-to-end no-buffering setup as RSC streaming — nginx, load balancers, everything in the path. See [Part 13](./nextjs-production-deployment.md#part-13-deployment-and-internals).

**File uploads.** Server Actions cap request bodies at 1 MB by default. You can raise `bodySizeLimit`, but for anything genuinely large the right answer is a Route Handler issuing a presigned URL so the file goes straight to object storage and never touches your Node process:

```ts
// app/api/uploads/presign/route.ts
export async function POST(request: Request) {
  const session = await verifySession()
  if (!session) return new Response(null, { status: 401 })

  const { filename, contentType } = await request.json()
  const key = `users/${session.userId}/${crypto.randomUUID()}-${filename}`
  const url = await createPresignedPutUrl({ key, contentType, expiresIn: 60 })

  return Response.json({ url, key })
}
```

> **Real Scenario — the webhook that charged customers twice.**
> A Stripe webhook created an order row on `checkout.session.completed`. Under load, some customers got two orders and two emails.
> **Cause, compounding.** The handler did its work — DB write, email, analytics — *before* returning 200, taking ~11 seconds under load. Stripe's timeout is shorter, so it retried, and the second delivery raced the first. Nothing was idempotent.
> **Fix.** Verify signature → upsert on the Stripe event ID (a unique constraint, so a duplicate is a no-op) → return 200 → do the email and analytics in `after()`. Response time dropped to 40ms and the duplicates stopped.
> **The lesson:** a webhook handler's job is to acknowledge, durably and fast. Everything else is someone else's job.

### 7.4 Cheat sheet: Part 7

| Need | Code |
|---|---|
| Define an endpoint | `export async function GET(request: Request) {}` in `route.ts` |
| Typed dynamic params | `ctx: RouteContext<'/api/users/[id]'>`, then `await ctx.params` |
| Query string | `request.nextUrl.searchParams.get('q')` |
| JSON body | `await request.json()` |
| Raw body (webhooks) | `await request.text()` |
| Read cookies/headers | `await cookies()` / `await headers()` from `next/headers` |
| Set a cookie | `NextResponse.json(…)` then `res.cookies.set(…)` |
| Cache a `GET` | `export const dynamic = 'force-static'` |
| Cache under Cache Components | `'use cache'` in a helper, not the handler body |
| Work after responding | `after(async () => {…})` from `next/server` |
| Status-only response | `new Response(null, { status: 401 })` |

| Rule | Why |
|---|---|
| `route.ts` can't share a segment with `page.tsx` | Each owns all verbs for the route |
| `GET` is the only cacheable method | By design |
| `request.body` is a stream | Web `Request`, not Node's `req` |
| Auth-check every handler | It's a public URL |
| Webhooks must be idempotent | Retries are guaranteed, not hypothetical |

[↑ Back to top](#table-of-contents)

---

**Next:** [Database & Auth (Parts 8–9)](./nextjs-database-auth.md) — Postgres with Drizzle, sessions, and the Data Access Layer.
