---
description: "Build Vue server-side rendering by hand with Vite: per-request app factories, renderToString, onServerPrefetch, safe state transfer, preload links, status codes and hydration."
---

# 01 · Server-Side Rendering from Scratch

A client-rendered Vue app sends an almost empty HTML page and builds everything in the
browser. **Server-side rendering (SSR)** runs the same components on the server, sends
real HTML, and then **hydrates** it in the browser — attaching event listeners and state
to the existing DOM instead of rebuilding it. You get content on screen before JavaScript
runs, crawlable HTML, and correct status codes.

Most teams use Nuxt for SSR (next lesson). Building it once by hand is the best way to
understand what Nuxt does for you and to debug it when something goes wrong. Everything
below was built with Vite 8.3.1, run with Node.js 26, fetched with `curl`, and hydrated in
a real browser.

## The architecture

```text
Request /products/lamp
  │
  ▼
server.js ──► render(url)                      (dist/server/entry-server.js)
               ├─ createApp(memory history)    fresh app, router, Pinia
               ├─ router.push(url); isReady()
               ├─ renderToString(app)          runs setup + onServerPrefetch
               └─ serialise Pinia state
  ◄── HTML with <div id="app">…rendered…</div> + <script>window.__PINIA__=…</script>
  │
Browser ──► entry-client.ts
               ├─ createApp(web history)
               ├─ pinia.state = window.__PINIA__
               └─ app.mount('#app')            → hydrate, not re-render
```

Two builds come out of one codebase: a **client** bundle (like any Vite app) and a
**server** bundle that exports a `render` function.

## Project files

```json title="package.json"
{
  "name": "vue-ssr-from-scratch",
  "private": true,
  "type": "module",
  "scripts": {
    "build:client": "vite build --outDir dist/client --ssrManifest",
    "build:server": "vite build --outDir dist/server --ssr src/entry-server.ts",
    "build": "npm run build:client && npm run build:server",
    "start": "node server.js"
  },
  "dependencies": {
    "devalue": "^6.0.2",
    "pinia": "^4.0.3",
    "vue": "^3.5.43",
    "vue-router": "^5.3.1"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^6.0.9",
    "typescript": "^7.0.2",
    "vite": "^8.3.1"
  }
}
```

Dependencies: `vue`, `vue-router`, `pinia`, `devalue` (for safe serialisation), plus
`vite`, `@vitejs/plugin-vue` and `typescript` as dev dependencies. The `--ssrManifest` flag
makes the client build record which chunks each module needs; the server uses it to add
preload links.

```html title="index.html"
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <!--app-head-->
  </head>
  <body>
    <div id="app"><!--app-html--></div>
    <!--app-state-->
    <script type="module" src="/src/entry-client.ts"></script>
  </body>
</html>
```

The three comments are placeholders the server fills in per request.

### The app factory

```ts title="src/main.ts"
import { createSSRApp } from 'vue'
import { createPinia } from 'pinia'
import { createRouter } from 'vue-router'
import App from './App.vue'
import { routes } from './routes'
import type { RouterHistory } from 'vue-router'

// a factory: every request (and the browser) gets fresh app, router and store instances
export function createApp(history: RouterHistory) {
  const app = createSSRApp(App)
  const pinia = createPinia()
  const router = createRouter({ history, routes })
  app.use(pinia).use(router)
  return { app, pinia, router }
}
```

This is the most important rule of SSR: **no shared mutable state across requests.** A
Node.js server handles many users in one process. A module-level `const app = createApp()`
or a module-level store would leak one user's data into another's page. Every request calls
the factory and gets its own app, router and Pinia. `createSSRApp` (instead of `createApp`)
tells the client build to hydrate rather than mount fresh.

```ts title="src/routes.ts"
import type { RouteRecordRaw } from 'vue-router'

export const routes: RouteRecordRaw[] = [
  { path: '/', component: () => import('./pages/HomePage.vue') },
  { path: '/products/:id', component: () => import('./pages/ProductPage.vue'), props: true },
  { path: '/:pathMatch(.*)*', component: () => import('./pages/NotFound.vue'), meta: { status: 404 } },
]
```

`meta.status` lets a route declare its HTTP status; the server reads it.

### Data: a store and `onServerPrefetch`

```ts title="src/stores/products.ts"
import { ref } from 'vue'
import { defineStore } from 'pinia'

export interface Product { id: string; name: string; price: number; description: string }

// stands in for a database or API call on the server
const DB: Record<string, Product> = {
  lamp: { id: 'lamp', name: 'Desk Lamp', price: 39, description: 'Warm light. Contains </script> in its description on purpose.' },
  chair: { id: 'chair', name: 'Oak Chair', price: 149, description: 'Solid oak, <b>hand-finished</b>.' },
}

export const useProductsStore = defineStore('products', () => {
  const byId = ref<Record<string, Product>>({})

  async function load(id: string): Promise<Product | null> {
    if (byId.value[id]) return byId.value[id]
    await new Promise((r) => setTimeout(r, 20))   // pretend latency
    const product = DB[id] ?? null
    if (product) byId.value[id] = product
    return product
  }

  return { byId, load, all: () => Object.values(DB) }
})
```

The fake database deliberately contains `</script>` and `<b>` in descriptions, to test
escaping.

```vue title="src/pages/ProductPage.vue"
<script setup lang="ts">
import { ref, onServerPrefetch, onMounted } from 'vue'
import { useProductsStore, type Product } from '../stores/products'

const { id } = defineProps<{ id: string }>()
const store = useProductsStore()
const product = ref<Product | null>(store.byId[id] ?? null)   // already there after hydration
const count = ref(0)

async function load() {
  product.value = await store.load(id)
}
onServerPrefetch(load)              // server: awaited before the HTML is produced
onMounted(() => {                   // client: only fetch if the server didn't
  if (!product.value) load()
})
</script>

<template>
  <article v-if="product">
    <h1>{{ product.name }}</h1>
    <p>{{ product.price }} USD</p>
    <p>{{ product.description }}</p>
    <button type="button" @click="count++">Add to cart ({{ count }})</button>
  </article>
  <p v-else>Loading…</p>
</template>
```

`onServerPrefetch` registers an async hook that `renderToString` **awaits** before producing
this component's HTML. It runs only on the server. On the client, the store already holds
the product (from the transferred state), so `onMounted` skips the fetch — unless this page
was reached by client-side navigation, in which case it loads normally.

### The two entries

```ts title="src/entry-server.ts"
import { createMemoryHistory } from 'vue-router'
import { renderToString, type SSRContext } from 'vue/server-renderer'
import { uneval } from 'devalue'
import { createApp } from './main'

export async function render(url: string, manifest: Record<string, string[]>) {
  const { app, pinia, router } = createApp(createMemoryHistory())
  await router.push(url)
  await router.isReady()

  const ctx: SSRContext = {}
  const html = await renderToString(app, ctx)          // awaits onServerPrefetch hooks
  const status = (router.currentRoute.value.meta.status as number | undefined) ?? 200

  // uneval escapes </script> and other characters that would break out of the tag
  const state = `<script>window.__PINIA__=${uneval(pinia.state.value)}</script>`

  // preload the JS chunks of the components this page actually rendered
  const files = new Set<string>()
  for (const id of ctx.modules ?? []) for (const f of manifest[id] ?? []) files.add(f)
  const head = [...files]
    .filter((f) => f.endsWith('.js'))
    .map((f) => `<link rel="modulepreload" href="${f}">`)
    .join('\n    ')

  return { html, state, head, status }
}
```

```ts title="src/entry-client.ts"
import { createWebHistory } from 'vue-router'
import { createApp } from './main'

const { app, pinia, router } = createApp(createWebHistory())

// reuse the server's store state instead of refetching
const state = (window as unknown as { __PINIA__?: Record<string, unknown> }).__PINIA__
if (state) pinia.state.value = state

router.isReady().then(() => app.mount('#app'))   // mount = hydrate, for a createSSRApp app
```

- **State transfer**: the server serialises `pinia.state.value` and the client assigns it
  back *before* mounting, so stores start with the server's data. Pinia was designed for
  exactly this: all store state lives in that one object (Level 2 · 05).
- **Escaping**: `JSON.stringify` is **not** safe inside a `<script>` tag — a string
  containing `</script>` ends the tag and whatever follows is parsed as HTML, a classic XSS.
  `devalue`'s `uneval` escapes it (and also supports `Date`, `Map`, `Set` and repeated
  references).
- **Preloading**: `ctx.modules` is filled by `@vitejs/plugin-vue` during rendering with the
  ids of components that rendered; the SSR manifest maps them to chunk files.

### The server

```js title="server.js"
import { createServer } from 'node:http'
import { readFile } from 'node:fs/promises'
import { extname, join } from 'node:path'

const template = await readFile('./dist/client/index.html', 'utf8')
const manifest = JSON.parse(await readFile('./dist/client/.vite/ssr-manifest.json', 'utf8'))
const { render } = await import('./dist/server/entry-server.js')
const types = { '.js': 'text/javascript', '.css': 'text/css', '.svg': 'image/svg+xml' }

createServer(async (req, res) => {
  const url = req.url ?? '/'
  try {
    if (url.startsWith('/assets/')) {
      const file = await readFile(join('./dist/client', url))
      res.writeHead(200, {
        'Content-Type': types[extname(url)] ?? 'application/octet-stream',
        'Cache-Control': 'public, max-age=31536000, immutable',   // hashed file names
      })
      return res.end(file)
    }
    const { html, state, head, status } = await render(url, manifest)
    const page = template
      .replace('<!--app-head-->', head)
      .replace('<!--app-html-->', html)
      .replace('<!--app-state-->', state)
    res.writeHead(status, { 'Content-Type': 'text/html; charset=utf-8', 'Cache-Control': 'no-cache' })
    res.end(page)
  } catch (err) {
    console.error(err)
    res.writeHead(500, { 'Content-Type': 'text/plain' }).end('Internal Server Error')
  }
}).listen(3000, () => console.log('SSR server on http://localhost:3000'))
```

A plain `node:http` server keeps the moving parts visible; Express, Fastify, Hono or H3 work
the same way. Hashed assets get a one-year immutable cache; HTML gets `no-cache`
(Level 4 · 08).

## What it produced

`npm run build` created both bundles:

```text
dist/client/index.html                       0.34 kB │ gzip:  0.24 kB
dist/client/.vite/ssr-manifest.json          1.22 kB │ gzip:  0.40 kB
dist/client/assets/NotFound-D_DIumQw.js      0.16 kB │ gzip:  0.15 kB
dist/client/assets/HomePage-C1zw3XFg.js      0.48 kB │ gzip:  0.34 kB
dist/client/assets/products-C0SMTFOz.js      0.49 kB │ gzip:  0.35 kB
dist/client/assets/ProductPage-BaMQZ8-n.js   0.61 kB │ gzip:  0.41 kB
dist/client/assets/index-BZlrWVGJ.js        98.89 kB │ gzip: 38.69 kB
...
dist/server/entry-server.js                 2.58 kB │ gzip: 1.09 kB
```

`curl -i http://localhost:3000/products/lamp` (headers trimmed):

```html
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Cache-Control: no-cache

  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link rel="modulepreload" href="/assets/ProductPage-BaMQZ8-n.js">
    <script type="module" crossorigin src="/assets/index-BZlrWVGJ.js"></script>
  </head>
  <body>
    <div id="app"><!--[--><header><a href="/" class="">Shop</a></header><main><article><h1>Desk Lamp</h1><p>39 USD</p><p>Warm light. Contains &lt;/script&gt; in its description on purpose.</p><button type="button">Add to cart (0)</button></article></main><!--]--></div>
    <script>window.__PINIA__={products:{byId:{lamp:{id:"lamp",name:"Desk Lamp",price:39,description:"Warm light. Contains </script> in its description on purpose."}}}}</script>
  </body>
```

Check the details:

- The product data is **in the HTML** — the server awaited `onServerPrefetch`.
- The `</script>` in the description is escaped twice, correctly for each context: as
  `&lt;/script&gt;` in the HTML text, and as `</script>` inside the state script.
- `<!--[-->`/`<!--]-->` are fragment markers (the `App` template has two root nodes);
  hydration uses them to line up DOM with vnodes.
- Only `ProductPage`'s chunk was preloaded — the page that rendered.

`/nope` returned status **404** (from `meta.status`), and an asset URL returned 200.

In the browser, we clicked "Add to cart" and checked that the button element was the
**same DOM node** the server sent (`sameNode: true`, text `Add to cart (1)`), with no
console warnings. Clicking "Shop" and then another product navigated client-side — a
marker we set on `window` survived, so there was no page reload.

The Network panel showed one thing to improve:

```text
GET http://localhost:3000/assets/ProductPage-BaMQZ8-n.js → 200 OK
GET http://localhost:3000/assets/index-BZlrWVGJ.js → 200 OK
GET http://localhost:3000/assets/products-C0SMTFOz.js → 200 OK
```

`products-*.js` (the store, shared by two pages) wasn't in the preload list, so it was
discovered only after `ProductPage` loaded — a small request waterfall. Our preload logic
only includes the rendered components' own files; production setups (and Nuxt) also walk
each chunk's imports from the client build manifest. That's Exercise 2.

## Hydration mismatches

Hydration assumes the client's first render produces **exactly** the server's HTML. When it
doesn't, Vue patches the difference and warns in development. From a Level 1 experiment
that rendered "Rendered on server" on the server and "Rendered on client" in the browser:

```text
[Vue warn]: Hydration text content mismatch on HTMLParagraphElement {}
  - rendered on server: Rendered on server
  - expected on client: Rendered on client
Hydration completed but contains mismatches.
```

Common causes: `Date.now()` or `Math.random()` in templates, reading `window`/`localStorage`
during setup (the server has neither — it would crash or take another branch), locale or
time-zone differences in formatting, invalid HTML nesting that the browser "fixes"
(`<div>` inside `<p>`), and data that changed between server render and hydration. Fixes:
compute such values in `onMounted` (so the first client render matches), render a
client-only placeholder, or pass the value from the server in state. For unavoidable cases
(a timestamp), Vue 3.5 supports `data-allow-mismatch` on the element.

## How It Actually Works

`renderToString` runs components through a **separate SSR compiler output**. For the server
build, `@vitejs/plugin-vue` compiles templates to `ssrRender` functions that push strings into
a buffer (`_push(`<h1>${_ssrInterpolate(product.name)}</h1>`)`) instead of creating vnodes —
much faster, because there's no DOM to diff. Reactivity is effectively off on the server:
effects aren't kept, watchers don't re-run, `onMounted` never fires. Each component's
`setup` runs, `onServerPrefetch` promises are awaited, and the string is produced once.

On the client, `createSSRApp(...).mount(el)` calls `hydrate` instead of `render`. The
hydration walker traverses the new vnode tree and the existing DOM **in parallel**: for each
vnode it checks that the DOM node has the expected type (and text, for text nodes), adopts
it as the vnode's `el`, attaches event listeners and continues into children. Fragment
comment markers tell it where multi-node fragments start and end. When a node doesn't
match, it logs a mismatch and falls back to creating the correct DOM for that subtree. After
hydration, the app behaves like any client-rendered Vue app — which is why the button node
was reused and later updates went through the normal renderer.

## Common mistakes

- **Module-level app, router or store instances** — cross-request data leaks.
- **`JSON.stringify` into a `<script>` tag** — XSS via `</script>`.
- **Browser APIs in `setup`** on SSR-rendered components — guard with `onMounted` or
  `import.meta.env.SSR`.
- **Fetching in `onMounted` only** — the server renders "Loading…", and crawlers see that.
- **Always returning 200** — send 404s and 500s from the server so crawlers and monitors
  see the truth.
- **Non-deterministic rendering** (random ids, current time) — use `useId()` and
  client-only rendering for such values.

## Exercise

1. Rebuild this project, run it, and view the page source (not devtools Elements) to see
   the server HTML. Disable JavaScript in devtools and confirm the page still shows the
   product.
2. Fix the waterfall: read the client build manifest (`build.manifest: true` produces
   `.vite/manifest.json`) and include each preloaded chunk's `imports` recursively.
3. Add a `<title>` per page: set it in a `route.meta.title` or on the SSR context during
   rendering, and inject it into `<!--app-head-->`.
4. Deliberately render `new Date().toLocaleTimeString()` in a template. Reproduce the
   mismatch warning in a development build (`vite build --mode development`), then fix it
   two different ways.
