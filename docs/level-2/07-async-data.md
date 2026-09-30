---
description: "Load server data in Vue: where to fetch, loading/error/empty states, typed HTTP errors, request dedupe and caching in a store, Suspense, and Pinia Colada."
---

# 07 · Async Data & API Calls

Almost every Vue app talks to an API. Fetching is easy; fetching *well* means handling
loading and error states, not showing stale responses, not firing the same request five
times, and not refetching data you already have. This lesson builds up those pieces, then
shows where a data-fetching library takes over.

## Where to fetch

In `<script setup>`, you can start a request directly in setup — you don't need
`onMounted`:

```vue
<script setup lang="ts">
import { ref } from 'vue'

const posts = ref<Post[]>([])
fetch('/api/posts').then((r) => r.json()).then((data) => (posts.value = data))
</script>
```

Starting in setup begins the request slightly earlier than `onMounted` would. Use
`onMounted` only when the request depends on the DOM (a size, a scroll position) or must
not run during server-side rendering.

That snippet ignores errors and loading state, though. Every real fetch has **four**
states: idle/loading, success with data, success with *no* data, and error. The UI should
show each one deliberately.

## A typed HTTP helper

`fetch` only rejects on network failure — a 404 or 500 **resolves** successfully with
`res.ok === false`. Wrap it once so every call site gets consistent errors:

```ts title="src/api/http.ts"
export class HttpError extends Error {
  constructor(
    public status: number,
    public url: string,
    public body: unknown,
  ) {
    super(`HTTP ${status} for ${url}`)
    this.name = 'HttpError'
  }
}

export async function getJSON<T>(url: string, init?: RequestInit): Promise<T> {
  const res = await fetch(url, { headers: { Accept: 'application/json' }, ...init })
  const text = await res.text()
  const body: unknown = text ? JSON.parse(text) : null
  if (!res.ok) throw new HttpError(res.status, url, body)
  return body as T
}
```

The error keeps the status and the parsed body, so a component can distinguish "not
found" (show a friendly page) from "server error" (show retry).

## Rendering the four states

```vue title="src/views/ProductView.vue"
<script setup lang="ts">
import { ref, watch } from 'vue'
import { HttpError } from '@/api/http'
import { useProductsStore, type Product } from '@/stores/products'

const { id } = defineProps<{ id: number }>()
const store = useProductsStore()

const product = ref<Product | null>(null)
const error = ref<unknown>(null)
const loading = ref(false)

watch(
  () => id,
  async (current) => {
    loading.value = true
    error.value = null
    try {
      const result = await store.load(current)
      if (current === id) product.value = result   // ignore a late response for an old id
    } catch (e) {
      if (current === id) error.value = e
    } finally {
      if (current === id) loading.value = false
    }
  },
  { immediate: true },
)
</script>

<template>
  <p v-if="loading && !product" aria-busy="true">Loading product…</p>
  <section v-else-if="error" role="alert">
    <template v-if="error instanceof HttpError && error.status === 404">
      <h1>Product not found</h1>
      <RouterLink to="/products">Back to all products</RouterLink>
    </template>
    <template v-else>
      <h1>Something went wrong</h1>
      <button type="button" @click="store.load(id, { force: true })">Try again</button>
    </template>
  </section>
  <article v-else-if="product" :aria-busy="loading">
    <h1>{{ product.title }}</h1>
    <p>{{ product.price }} USD</p>
  </article>
</template>
```

Two choices worth noticing:

- **`loading && !product`** shows the spinner only on the *first* load. When switching to
  another product, the old content stays visible (with `aria-busy`) until the new data
  arrives, which feels faster than flashing a spinner.
- **`current === id` checks** guard against the race from Lesson 01: if the user switches
  from product 1 to 2 and product 1's response arrives last, it is ignored. (The
  `AbortController` approach from Lesson 01 is better when you can use it; this check
  works even when the request is shared through a cache and mustn't be aborted.)

## Dedupe and cache in a store

When a product appears in a list, a detail page and a "recently viewed" widget, you don't
want three requests for it, and you don't want to refetch it on every navigation. A store
is a natural home for that cache:

```ts title="src/stores/products.ts"
import { ref } from 'vue'
import { defineStore } from 'pinia'
import { getJSON } from '@/api/http'

export interface Product { id: number; title: string; price: number }

const STALE_AFTER_MS = 60_000

export const useProductsStore = defineStore('products', () => {
  const byId = ref(new Map<number, Product>())
  const fetchedAt = new Map<number, number>()
  const inFlight = new Map<number, Promise<Product>>()

  function get(id: number): Product | undefined {
    return byId.value.get(id)
  }

  async function load(id: number, { force = false } = {}): Promise<Product> {
    const cached = byId.value.get(id)
    const fresh = Date.now() - (fetchedAt.get(id) ?? 0) < STALE_AFTER_MS
    if (cached && fresh && !force) return cached

    // two components asking for the same product at once share one request
    const pending = inFlight.get(id)
    if (pending) return pending

    const request = getJSON<Product>(`/api/products/${id}`)
      .then((product) => {
        byId.value.set(id, product)
        fetchedAt.set(id, Date.now())
        return product
      })
      .finally(() => inFlight.delete(id))
    inFlight.set(id, request)
    return request
  }

  return { byId, get, load }
})
```

- `byId` is a reactive `Map`. Vue's reactivity supports `Map` and `Set` — `get`, `set`,
  `has` and iteration are all tracked.
- `inFlight` and `fetchedAt` are **plain** Maps: they're bookkeeping, not UI state, so
  there's no reason to make them reactive or expose them.
- A cached value younger than `STALE_AFTER_MS` is returned without a request; `force`
  bypasses the cache (for "Try again" and pull-to-refresh).

The tests use fake timers to move time forward without waiting:

```ts title="src/stores/__tests__/products.spec.ts"
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { createPinia, setActivePinia } from 'pinia'
import { useProductsStore } from '../products'

describe('products store', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    vi.useFakeTimers()
    vi.stubGlobal('fetch', vi.fn(async (url: string) =>
      new Response(JSON.stringify({ id: Number(url.split('/').pop()), title: 'Lamp', price: 30 }), { status: 200 })))
  })

  it('dedupes concurrent requests and caches the result', async () => {
    const store = useProductsStore()
    const [a, b] = await Promise.all([store.load(1), store.load(1)])
    expect(a).toBe(b)
    await store.load(1)
    expect(fetch).toHaveBeenCalledTimes(1)
    expect(store.get(1)?.title).toBe('Lamp')
  })

  it('refetches once the cache is stale', async () => {
    const store = useProductsStore()
    await store.load(1)
    vi.advanceTimersByTime(61_000)
    await store.load(1)
    expect(fetch).toHaveBeenCalledTimes(2)
  })

  it('surfaces HTTP errors with the status', async () => {
    vi.mocked(fetch).mockResolvedValueOnce(new Response('{"message":"nope"}', { status: 404 }))
    await expect(useProductsStore().load(9)).rejects.toMatchObject({ status: 404, name: 'HttpError' })
  })
})
```

```text
 ✓ src/stores/__tests__/products.spec.ts > products store > dedupes concurrent requests and caches the result 5ms
 ✓ src/stores/__tests__/products.spec.ts > products store > refetches once the cache is stale 1ms
 ✓ src/stores/__tests__/products.spec.ts > products store > surfaces HTTP errors with the status 1ms
```

## Async setup and `<Suspense>`

`<script setup>` may use top-level `await`. The component then becomes **async**, and it
must be rendered inside a `<Suspense>` boundary somewhere above it, which shows fallback
content until all async descendants resolve:

```vue title="src/components/ProductSummary.vue"
<script setup lang="ts">
import { getJSON } from '@/api/http'
const { id } = defineProps<{ id: number }>()
const product = await getJSON<{ title: string }>(`/api/products/${id}`)
</script>

<template><h2>{{ product.title }}</h2></template>
```

```vue-html
<Suspense>
  <ProductSummary :id="1" />
  <template #fallback><p>Loading…</p></template>
</Suspense>
```

In our test, the boundary rendered `<p>loading...</p>` first and `<p>loaded</p>` after the
async setup finished. Vue also logged:

```text
<Suspense> is an experimental feature and its API will likely change.
```

That warning has been there since Vue 3.0, and Suspense is used heavily by Nuxt, but take it
at face value: errors must be caught with `onErrorCaptured` (Level 3 · 09), and a
component with top-level `await` fetches only once — it doesn't react to prop changes.
Use it for "load once, then render" cases, not for data that changes with props.

## Letting a library do it: Pinia Colada

Caching, deduplication, stale times, refetch-on-focus, invalidation after mutations,
optimistic updates — these are the same problems in every app, and data-fetching libraries
solve them. In the Vue ecosystem, **Pinia Colada** (`@pinia/colada`, by the Pinia author;
1.4.6 at the time of writing) builds on Pinia. Vue Router 5 even lists it as an optional
peer dependency for its experimental data loaders. TanStack Query's Vue adapter
(`@tanstack/vue-query`) is a popular alternative.

```bash
npm install @pinia/colada
```

```ts title="src/main.ts (excerpt)"
import { PiniaColada } from '@pinia/colada'

app.use(createPinia())
app.use(PiniaColada)   // after Pinia
```

```vue title="src/components/UserName.vue"
<script setup lang="ts">
import { useQuery } from '@pinia/colada'
import { getJSON } from '@/api/http'

const { id } = defineProps<{ id: number }>()

const { data, status, error, refetch } = useQuery({
  key: () => ['users', id],
  query: () => getJSON<{ name: string }>(`/api/users/${id}`),
})
</script>

<template>
  <span v-if="status === 'pending'">…</span>
  <span v-else-if="status === 'error'">Couldn't load ({{ error?.message }})</span>
  <span v-else>{{ data?.name }}</span>
</template>
```

We mounted a component like this twice for the same id, then switched one of them to a
different id. The rendered `status/asyncStatus: name` and our fake API's call count:

```text
initial: pending/idle: -
loaded: success/idle: User 1
second component: success/idle: User 1 fetch calls 1
switched: pending/loading: -
switched loaded: success/idle: User 2 calls 2
```

The second component got user 1 from the cache with no new request, and changing the
reactive `key` triggered a fetch for user 2. Colada separates **`status`** (do we have data:
`pending` / `success` / `error`) from **`asyncStatus`** (is a request running: `idle` /
`loading`) — the same "first load vs refresh" distinction we built by hand above. Check
its documentation for `useMutation`, cache invalidation and `staleTime` options; its API
has been stable since 1.0 but is younger than Pinia's.

## How It Actually Works

Everything here is ordinary reactivity plus promises; there's no special "async mode" in
Vue's renderer except `<Suspense>`.

- The `watch(() => id, ..., { immediate: true })` pattern works because `id` is a
  destructured prop that the compiler rewrites to `__props.id` inside the getter, so the
  watcher tracks it and reruns when the route reuses the component with a new id.
- A reactive `Map` is a `Proxy` around the Map with special handlers ("collection
  handlers"): `get(key)` tracks that key, `set(key)` triggers it, and `size`/iteration
  track a special iteration key. So a template reading `store.get(1)` re-renders when
  entry `1` is set, and not when entry `2` is.
- **Suspense**: when a component's `setup` returns a promise (which is what top-level
  `await` compiles to — the compiler wraps the awaits with `withAsyncContext` so the
  current instance is restored after each `await`), the renderer registers it as a
  dependency of the nearest `<Suspense>`. Suspense renders the default slot into an
  off-DOM container and shows the fallback; when all registered promises resolve, it moves
  the resolved tree into the document.
- **Pinia Colada** keeps a query cache inside a Pinia store, keyed by the serialised
  `key`. `useQuery` creates a computed key; a watcher on it looks up (or creates) the cache
  entry, reuses a pending promise if one exists (dedupe), and decides from `staleTime`
  whether to refetch. The `data`/`status` you get back are computeds over that shared entry,
  which is why two components stay in sync.

## Common mistakes

- **Treating `fetch` as throwing on HTTP errors** — check `res.ok`.
- **Showing only a spinner and data** — no error state, no empty state.
- **Flashing a spinner on every refetch** — distinguish first load from refresh.
- **Late responses overwriting newer ones** — abort, or check that the response still
  matches the current input.
- **Fetching in `onMounted` of every component that shows the same entity** — dedupe and
  cache in a store or library.
- **Top-level `await` without a `<Suspense>` boundary** — the component renders nothing
  and Vue warns.
- **Putting server data in `localStorage`** "for speed" — it goes stale and may leak
  between users on shared machines.

## Exercise

1. Build `ProductView.vue` against a local mock API (Vite serves anything in `public/`, so
   `public/api/products/1` — a file without extension containing JSON — works for GET
   requests; our Vite 8 dev server returned it with `200` and an empty `Content-Type`, which
   is fine because `getJSON` parses the body as text), and route to it with `props: (r) => ({ id: Number(r.params.id) })`.
2. Add a list page that preloads each product into the store, so opening a detail page is
   instant. Confirm in the Network tab that no second request is made.
3. Add an "empty" state to the list when the API returns `[]`.
4. Rebuild the detail page with Pinia Colada's `useQuery`. Compare the amount of code, and
   list which behaviours (dedupe, stale time, loading vs refreshing) you got for free.
