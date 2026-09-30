---
description: "Tour Nuxt 4 by building a small shop: file-based pages, server API routes, useFetch and payload transfer, useState, SEO meta, route rules and error pages."
---

# 02 · Nuxt Overview

The previous lesson built SSR by hand: an app factory per request, two entry points, a
server that calls `renderToString`, and a script tag that carries state to the browser.
**Nuxt** is the framework that packages all of that — plus routing, a server for API
endpoints, data fetching that doesn't double-fetch, head management and deployment
presets — behind conventions.

This lesson is an overview, not a full Nuxt course. The goal is that you can read a Nuxt
project, know which Vue concept each convention maps to, and decide whether you need it.
Everything below was built with **Nuxt 4.5.2** (Vue 3.5.43, Nitro 2.13.4), run with
`node .output/server/index.mjs`, and inspected with `curl`. Nuxt's APIs have been stable
through the 3.x → 4.x transition, but check the docs for your version before copying
configuration.

## When Nuxt is worth it

Reach for Nuxt when you want at least one of:

- **Server-rendered or pre-rendered HTML** — content sites, marketing pages, e-commerce,
  anything where first paint and crawlable HTML matter.
- **A backend in the same project** — small API endpoints, form handlers, proxies to
  services whose keys must stay secret.
- **Per-route rendering strategies** — some pages static, some server-rendered, some
  client-only.

If you're building a dashboard behind a login, where SEO is irrelevant and a separate API
already exists, a plain Vite + Vue Router + Pinia app (everything in Levels 1–3) is
simpler and entirely reasonable. Nuxt adds build steps and concepts; buy them only when
you use them.

## Creating a project

```bash
npm create nuxt@latest nuxt-shop
cd nuxt-shop
npm run dev
```

In Nuxt 4 the application code lives in `app/`, and server code lives in `server/`. The
shop we built has this shape:

```text
nuxt-shop/
├─ nuxt.config.ts
├─ app/
│  ├─ app.vue                  # root component
│  ├─ composables/useCart.ts   # auto-imported
│  └─ pages/
│     ├─ index.vue             # → /
│     ├─ about.vue             # → /about
│     └─ products/[id].vue     # → /products/:id
└─ server/
   └─ api/products/
      ├─ index.get.ts          # GET /api/products
      └─ [id].get.ts           # GET /api/products/:id
```

Two conventions do most of the work: **file names become routes** and **folders like
`composables/` and `components/` are auto-imported**. Neither is magic — the next sections
show what each one generates.

## Pages and layout

`app.vue` is the root component. `<NuxtPage />` is Nuxt's wrapper around Vue Router's
`<RouterView>`, and `<NuxtLink>` wraps `<RouterLink>` (adding prefetching of the target
page's code when the link scrolls into view).

```vue title="app/app.vue"
<template>
  <NuxtRouteAnnouncer />
  <header>
    <NuxtLink to="/">Shop</NuxtLink> · <NuxtLink to="/about">About</NuxtLink>
    · Cart: {{ cart.length }}
  </header>
  <NuxtPage />
</template>

<script setup lang="ts">
const cart = useCart()
</script>
```

`<NuxtRouteAnnouncer />` renders a polite live region that announces the page title after
navigation — the same technique Level 3's accessibility lesson built by hand.

Nothing imports `useCart`. Nuxt scans `app/composables/` and generates type declarations
and import statements for you at build time, so the compiled code contains an ordinary
import. The same goes for Vue's own APIs (`ref`, `computed`, …) and Nuxt's composables
(`useFetch`, `useRoute`, …).

## Server routes

Files in `server/api/` become HTTP endpoints, served by **Nitro**, Nuxt's server engine
(built on the **h3** HTTP library). The method suffix (`.get.ts`) restricts the handler to
that verb; `[id]` is a route parameter.

```ts title="server/api/products/index.get.ts"
const products = [
  { id: 1, name: 'Notebook', price: 4.5 },
  { id: 2, name: 'Pencil set', price: 3 },
  { id: 3, name: 'Desk lamp', price: 24 },
]

export default defineEventHandler((event) => {
  const { max } = getQuery(event)
  const limit = max ? Number(max) : Infinity
  return products.filter((p) => p.price <= limit)
})
```

```ts title="server/api/products/[id].get.ts"
export default defineEventHandler(async (event) => {
  const id = Number(getRouterParam(event, 'id'))
  const all = await $fetch('/api/products')
  const product = all.find((p) => p.id === id)
  if (!product) {
    throw createError({ statusCode: 404, statusMessage: 'Product not found' })
  }
  return product
})
```

Returning an object sends JSON. Against the production build:

```text
$ curl -i "http://localhost:3917/api/products?max=5"
HTTP/1.1 200 OK
cache-control: public, max-age=60
content-type: application/json

[{"id":1,"name":"Notebook","price":4.5},{"id":2,"name":"Pencil set","price":3}]
```

(The `cache-control` header comes from a route rule, below.) A thrown `createError`
becomes a proper error response:

```text
$ curl http://localhost:3917/api/products/9
{
  "error": true,
  "url": "http://localhost:3917/api/products/9",
  "statusCode": 404,
  "statusMessage": "Product not found",
  "message": "Product not found"
}
```

Note the `$fetch('/api/products')` inside the second handler. On the server, Nitro
recognises a request to one of its own routes and calls the handler **directly in
memory** instead of opening an HTTP connection. This is also what makes `useFetch` cheap
during SSR.

Server code never ships to the browser, so this is where secrets belong: database clients,
API keys (read via `useRuntimeConfig()` from environment variables), and anything you
wouldn't paste into a public JavaScript file.

## Fetching data: `useFetch`

```vue title="app/pages/index.vue"
<script setup lang="ts">
const { data: products, status } = await useFetch('/api/products')
useSeoMeta({ title: 'All products', description: 'Everything in the demo shop' })
const renderedOn = import.meta.server ? 'server' : 'client'
</script>

<template>
  <main>
    <h1>Products</h1>
    <p>First rendered on: {{ renderedOn }}</p>
    <p v-if="status === 'pending'">Loading…</p>
    <ul v-else>
      <li v-for="p in products" :key="p.id">
        <NuxtLink :to="`/products/${p.id}`">{{ p.name }}</NuxtLink> — ${{ p.price }}
      </li>
    </ul>
  </main>
</template>
```

`useFetch` solves the problem Lesson 01 solved with `onServerPrefetch` and a hand-written
state script:

1. During SSR, the `await` pauses setup until the data arrives, so the HTML contains the
   list.
2. The result is stored in the **payload** under a key (derived from the URL and call
   site), serialised into the page.
3. During hydration in the browser, `useFetch` finds that key in the payload and **does
   not fetch again**.

Here is what the server sent for `/` (trimmed):

```html
<title>All products</title>
<meta name="description" content="Everything in the demo shop">
...
<p>First rendered on: server</p>
<li><a href="/products/1" class="">Notebook</a> — $4.5</li><li><a href="/products/2" class="">Pencil set</a> — $3</li>...
<script type="application/json" data-nuxt-data="nuxt-app" data-ssr="true" id="__NUXT_DATA__">[["ShallowReactive",1],{"data":2,"state":16,"once":19,"_errors":20,"serverRendered":22,"path":23},["ShallowReactive",3],{"$fm2j8a7sf5k2s":4},[5,9,13],{"id":6,"name":7,"price":8},1,"Notebook",4.5,{"id":10,"name":11,"price":12},2,"Pencil set",3,{"id":12,"name":14,"price":15},"Desk lamp",24,["Reactive",17],{"$scart":18},[],...]</script>
```

The payload is not plain JSON of your data. It's a flat array where objects refer to
other entries by index (the format comes from the `devalue` library). That buys three
things JSON can't do:

- **Types survive**: markers like `"Reactive"`, `"ShallowReactive"` and `"Set"` let the
  client rebuild reactive objects, `Set`s, `Map`s and `Date`s.
- **Shared references and cycles survive**: an object used twice is stored once.
- **Values are de-duplicated**: look at the Desk lamp, `{"id":12,...}`. Index 12 is the
  number `3` — the same entry used for the Pencil set's price. The serialiser stored `3`
  once and pointed to it twice.

It also escapes characters like `<` so data can't close the `<script>` tag — the XSS
problem Lesson 01 handled with a custom escape function.

`$fetch` is the plain function; `useFetch` is the SSR-aware composable. Use `useFetch` (or
`useAsyncData` for non-HTTP sources) in setup to load page data. Use `$fetch` in event
handlers — a button click that posts a form has nothing to hydrate.

## Shared state: `useState`

```ts title="app/composables/useCart.ts"
export const useCart = () => useState<string[]>('cart', () => [])
```

A module-level `ref([])` would be a bug under SSR: that module is loaded once per server
process, so every visitor would share one cart. `useState` stores the value on the
**per-request** Nuxt app instead and serialises it into the payload — you can see
`{"$scart":18}` with an empty array in the output above. For anything larger than a
couple of values, Pinia works in Nuxt too (via the `@pinia/nuxt` module) and gets the same
per-request isolation.

## Dynamic routes, SEO and errors

```vue title="app/pages/products/[id].vue"
<script setup lang="ts">
const route = useRoute()
const cart = useCart()
const { data: product, error } = await useFetch(`/api/products/${route.params.id}`)
if (error.value) {
  throw createError({ statusCode: 404, statusMessage: 'Product not found', fatal: true })
}
useSeoMeta({ title: () => product.value?.name ?? 'Product' })
</script>

<template>
  <main v-if="product">
    <h1>{{ product.name }}</h1>
    <p>${{ product.price }}</p>
    <button @click="cart.push(product.name)">Add to cart</button>
  </main>
</template>
```

Results against the production server:

```text
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3917/products/2
200            # <title>Pencil set</title> ... <h1>Pencil set</h1>
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3917/products/99
404
```

The page for a missing product returns a **real 404 status**, not a 200 with "not found"
text — important for search engines and monitoring. The server also logged the error
(`[request error] [fatal] [GET] http://localhost:3917/products/99`). A custom
`app/error.vue` replaces Nuxt's default error page; call `clearError({ redirect: '/' })`
from it to recover.

`useSeoMeta` accepts getters, so the title updates when the data changes during client
navigation. It's built on **Unhead**, the same head manager you could use in a plain Vue
app.

## Route rules: rendering per route

```ts title="nuxt.config.ts"
export default defineNuxtConfig({
  compatibilityDate: '2025-07-15',
  routeRules: {
    '/about': { prerender: true },
    '/api/products': { headers: { 'cache-control': 'public, max-age=60' } },
  },
})
```

`routeRules` set behaviour per path pattern. During `nuxt build` the log showed the
prerenderer running (`Prerendering 1 routes` … `Prerendered 2 routes`), and
`.output/public/about/` contained both `index.html` and `_payload.json` — the page and its
data as static files, served without running Vue at request time. Other rule options
include `ssr: false` (client-only rendering for that path), `redirect`, `cors`, and
caching options such as `swr`; check the current docs for the exact set your version
supports.

## Building and deploying

```bash
npm run build           # → .output/
node .output/server/index.mjs
```

`.output/` is self-contained: `server/` holds the bundled Nitro server (including your API
routes and the SSR renderer), `public/` holds client assets and prerendered pages. Nitro
**presets** retarget the same app at other hosts (Node, serverless functions, edge
runtimes, static hosting); `nuxt generate` prerenders the entire site for purely static
hosting such as GitHub Pages. We only ran the Node output; the other presets depend on the
provider's platform, so test there before relying on them.

## How It Actually Works

- **Build-time code generation.** Before bundling, Nuxt scans your directories and writes
  generated files into `.nuxt/`: a route table built from `pages/` (the same route records
  you wrote for Vue Router in Level 2, with `component: () => import(...)` for code
  splitting), a list of server handlers for Nitro, and type declarations for auto-imports.
  An unplugin transform then adds real `import` statements to any file that uses an
  auto-imported name. What runs is ordinary Vue and Vue Router.
- **One Vite build, two targets.** Like Lesson 01's `entry-client` / `entry-server`, Nuxt
  builds a client bundle and a server bundle. The server bundle exports a render function;
  Nitro's `renderer` route calls it per request with a fresh app, router and payload.
- **Payload keys make hydration line up.** `useFetch` computes a key per call site. On the
  server it writes `payload.data[key]`; on the client it reads the same key before deciding
  whether to fetch. If the key or the rendered output differs between the two sides, you
  get a refetch or a hydration mismatch — the same failure mode Lesson 01 showed.
- **In-process fetch.** Nitro gives `$fetch` a local handler: during SSR, a request to
  `/api/...` is dispatched straight to the matching h3 handler, so a page that calls its
  own API doesn't pay for a network round trip.

## Common mistakes

- **Module-level state in composables** (`const cart = ref([])` outside the function) —
  shared between all users on the server. Use `useState` or Pinia.
- **`$fetch` in setup for page data** — runs on the server, then again on the client,
  because nothing records the result in the payload. Use `useFetch` / `useAsyncData`.
- **Browser APIs during SSR** — `window`, `localStorage` and `document` don't exist on the
  server. Guard with `import.meta.client`, use `onMounted`, or wrap parts in
  `<ClientOnly>`.
- **Secrets in public runtime config** — only keys under `runtimeConfig.public` reach the
  browser; keep API keys in the non-public part and use them in `server/`.
- **Returning 200 for missing content** — throw `createError({ statusCode: 404 })` so the
  status is correct.
- **Assuming a preset works because the Node build did** — serverless and edge runtimes
  have different limits (no filesystem, execution-time caps, restricted Node APIs).

## Exercise

1. Build the shop above. Run `npm run build`, start the server, and use `curl` to confirm
   the product list is in the HTML and that `/products/99` returns 404.
2. Open the page source and decode the `__NUXT_DATA__` array by hand for the product list:
   which index holds the Desk lamp's `id`?
3. Change the index page to use `$fetch` inside `onMounted` instead of `useFetch`. Compare
   the server HTML and the Network panel. What did you lose?
4. Add `server/api/cart.post.ts` that validates a body with `readBody` and returns the new
   cart count, and call it with `$fetch` from the "Add to cart" button.
5. Add a route rule `'/products/**': { ssr: false }` and explain the difference in the
   response for `/products/2`. Then remove it — which choice is right for a shop?
