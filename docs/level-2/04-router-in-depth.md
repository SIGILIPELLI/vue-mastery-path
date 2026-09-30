---
description: "Vue Router in depth: navigation guards and their order, typed route meta, auth redirects, lazy-loaded routes, scroll behaviour and Vue Router 5 file-based routing."
---

# 04 · Router in Depth: Guards, Meta, File Routes

The previous lesson mapped URLs to components. Real apps also need to **decide whether a
navigation may happen** (auth, unsaved changes), **attach information to routes** (titles,
required roles), **load route code on demand**, and **restore scroll position**. Vue
Router 5 also brings **file-based routing** into the core package. All logs here come from
Vue Router 5.3.1.

## Navigation guards

A guard is a function that runs during a navigation and can let it continue, cancel it or
redirect it. Its return value decides:

| Return | Effect |
|--------|--------|
| `undefined` or `true` | continue |
| `false` | cancel; stay on the current URL |
| a route location (`'/login'`, `{ name: 'login', query: {...} }`) | redirect |
| throw an error | cancel and call `router.onError` handlers |

Guards may be `async`; the router waits for the promise.

### Global guards

```ts title="src/router/index.ts (excerpt)"
router.beforeEach((to, from) => {
  const auth = useAuthStore()               // Pinia store — Lesson 05
  if (to.meta.requiresAuth && !auth.isLoggedIn) {
    return { name: 'login', query: { redirect: to.fullPath } }
  }
})

router.afterEach((to, from, failure) => {
  if (!failure) document.title = to.meta.title ? `${to.meta.title} · Acme` : 'Acme'
})
```

- `beforeEach` — runs before every navigation. Auth checks go here.
- `beforeResolve` — runs after all in-component guards and async components are resolved,
  just before the navigation is confirmed. Good for "last chance" checks such as asking for
  camera permission.
- `afterEach` — runs after the navigation; can't change it. Analytics, titles, focus
  management. Its third argument is the failure, if there was one.

### Per-route and in-component guards

```ts
{
  path: '/admin',
  component: AdminView,
  meta: { requiresAuth: true, roles: ['admin'] },
  beforeEnter: (to) => {
    if (!useAuthStore().hasRole('admin')) return { name: 'forbidden' }
  },
}
```

`beforeEnter` runs only when *entering* the route, not when params, query or hash change
within it.

Inside a component, use the composition guards:

```ts title="src/views/EditPostView.vue (excerpt)"
import { onBeforeRouteLeave, onBeforeRouteUpdate } from 'vue-router'

const dirty = ref(false)

onBeforeRouteLeave(() => {
  if (dirty.value && !window.confirm('Discard unsaved changes?')) return false
})

onBeforeRouteUpdate(async (to) => {
  // same component, new params — e.g. /posts/1/edit → /posts/2/edit
  await loadPost(to.params.id as string)
})
```

### The order, observed

We registered a global `beforeEach` that redirects unauthenticated users away from
`/admin` (a route with its own `beforeEnter`), plus `beforeResolve` and `afterEach` loggers.
Navigating to `/admin` while logged out, then logging in and navigating again, logged:

```text
beforeEach / -> /admin
beforeEach / -> /login?redirect=/admin
beforeResolve /login?redirect=/admin
afterEach /login?redirect=/admin
beforeEach /login?redirect=/admin -> /admin
beforeEnter /admin
beforeResolve /admin
afterEach /admin
```

Read it carefully:

1. The first navigation reached `beforeEach` and was redirected. The redirect is a **new
   navigation** — `beforeEach` runs again, and `from` is still `/`, because the user never
   actually arrived anywhere. No `afterEach` fires for the abandoned `/admin` attempt
   (redirecting inside a guard replaces the navigation instead of failing it).
2. `/admin`'s `beforeEnter` never ran the first time — the global guard stopped the
   navigation before per-route guards.
3. Once authenticated, the full chain runs: global `beforeEach` → route `beforeEnter` →
   `beforeResolve` → `afterEach`.

The complete resolution flow is: leave guards in deactivated components → global
`beforeEach` → `beforeRouteUpdate` in reused components → `beforeEnter` in route configs →
resolve async route components → `beforeRouteEnter` in activated components → global
`beforeResolve` → navigation confirmed → `afterEach` → DOM updated.

### A redirect loop to avoid

```ts
router.beforeEach((to) => {
  if (!auth.isLoggedIn) return '/login'   // ✗ also runs for /login itself → infinite redirect
})
```

Always check the target: `if (!auth.isLoggedIn && to.name !== 'login')`, or better, only
guard routes that declare `meta.requiresAuth`.

## Typed route meta

`meta` is an arbitrary object on each route record. `to.meta` merges the meta of **all
matched records**, parent to child, so a parent's `requiresAuth: true` covers its children.
Declare its shape once for type safety:

```ts title="src/router/meta.d.ts"
import 'vue-router'

export {}

declare module 'vue-router' {
  interface RouteMeta {
    title?: string
    requiresAuth?: boolean
    roles?: Array<'admin' | 'editor'>
  }
}
```

Now `to.meta.requiresAuth` is typed as `boolean | undefined`, and a typo such as
`meta: { requireAuth: true }` is a compile error.

## Lazy-loaded routes

```ts
{ path: '/reports', component: () => import('@/views/ReportsView.vue') }
```

A function returning a dynamic `import()` tells both the router and the bundler to split
this view into its own chunk, downloaded on first navigation to it. The router waits for
the chunk before confirming the navigation (the "resolve async route components" step
above). Lazy-load every route except the one most users land on.

Group related routes into one chunk if they are always used together — Vite follows
Rollup-style naming, so check your build output rather than guessing chunk names.

## Scroll behaviour

By default, navigations keep the scroll position of the previous page, which is rarely what
you want. `scrollBehavior` runs after each navigation:

```ts
const router = createRouter({
  history: createWebHistory(),
  routes,
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) return savedPosition            // Back/Forward: restore
    if (to.hash) return { el: to.hash, top: 80 }       // anchor, below a fixed header
    if (to.path !== from.path) return { top: 0 }       // new page: top
    // same page, only query changed (filters, pagination): keep position
  },
})
```

## File-based routing (Vue Router 5)

Writing route records by hand gets repetitive. Vue Router 5 merged the former
`unplugin-vue-router` project into the core package: routes are generated from files in
`src/pages`, with full type safety for names and params. The classic API above is
unchanged; this is opt-in.

We set this up in a fresh `create-vue` project:

```ts title="vite.config.ts"
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import VueRouter from 'vue-router/vite'

export default defineConfig({
  plugins: [
    VueRouter({ routesFolder: 'src/pages', dts: 'src/typed-router.d.ts' }), // must come before vue()
    vue(),
  ],
  resolve: { alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) } },
})
```

```ts title="src/router.ts"
import { createRouter, createWebHistory } from 'vue-router'
import { routes, handleHotUpdate } from 'vue-router/auto-routes'

export const router = createRouter({ history: createWebHistory(import.meta.env.BASE_URL), routes })
if (import.meta.hot) handleHotUpdate(router)
```

With these files:

```text
src/pages/
├── index.vue          →  /
├── about.vue          →  /about
├── users/
│   ├── index.vue      →  /users
│   └── [id].vue       →  /users/:id
└── [...path].vue      →  catch-all (not found)
```

`vite build` split each page into its own chunk automatically:

```text
dist/assets/pages-DinSk_ts.js       0.15 kB │ gzip:  0.15 kB
dist/assets/about-DYbTG2Kj.js       0.15 kB │ gzip:  0.15 kB
dist/assets/users-CnDiOmAl.js       0.15 kB │ gzip:  0.15 kB
dist/assets/_...path_-CASNU4M8.js   0.16 kB │ gzip:  0.15 kB
dist/assets/_id_-V2Kop0lO.js        0.20 kB │ gzip:  0.18 kB
dist/assets/index-BQg10P-D.js      86.87 kB │ gzip: 33.79 kB
```

The plugin also generated `src/typed-router.d.ts`, a map of every route name to its path
and param types. Route names default to the file path (`'/users/[id]'`). In a page,
`useRoute('/users/[id]')` returns a route whose `params.id` is typed. And a typo in a link
is caught by `vue-tsc` — we changed `'/users/[id]'` to `'/user/[id]'` in a `RouterLink`:

```text
Type '"/user/[id]"' is not assignable to type '"/" | "/[...path]" | "/about" | "/users/" | "/users/[id]" | undefined'. Did you mean '"/users/[id]"'?
```

Nested layouts work by pairing a file and a folder with the same name (`users.vue` renders
a `<RouterView>` for everything in `users/`). Per-page `meta` can be set with
`definePage({ meta: { requiresAuth: true } })` inside the page's `<script setup>`. Check
the Vue Router docs' file-based routing section for the full conventions — groups `(name)`,
optional params `[[id]]`, and the experimental data loaders, which were still marked
experimental in 5.3.

Whether to use file-based routing is a team choice: it removes boilerplate and adds type
safety; hand-written routes keep everything explicit in one file. Nuxt (Level 4 · 02) uses
file-based routing by default.

## How It Actually Works

A navigation is a pipeline of promises. `router.push(to)` resolves the target location,
then builds a queue: extract leave guards from components that are leaving (matched records
in `from` but not `to`), global `beforeEach`, update guards from reused records,
`beforeEnter` from entering records, then lazy components are loaded
and their `beforeRouteEnter` extracted, then `beforeResolve`. Each group runs in sequence;
every guard is wrapped in a function that turns its return value (or thrown error) into
"continue", "abort" or "redirect". A redirect abandons the pipeline and calls `push`
recursively with `replace` semantics — which is why the log shows `beforeEach` twice and
`from` unchanged.

Before running each step the router also checks whether a **newer navigation** has started
(each navigation remembers the "pending location"). If you click two links quickly, the
first navigation's remaining guards notice they are stale and it resolves with a
`cancelled` failure — the router never commits a navigation that's no longer wanted.

Only when the whole queue succeeds does it call `history.pushState` and assign
`currentRoute.value`, which triggers the `RouterView`s to re-render. The file-based plugin
changes none of this: at build time it scans `src/pages`, generates a `routes` array of
ordinary route records (with `() => import()` components), and serves it as the virtual
module `vue-router/auto-routes`.

## Common mistakes

- **Redirect loops** — a guard that redirects to a route it also guards.
- **Using the old `next()` callback style incorrectly** — calling it twice or not at all
  hangs or breaks navigation. It still works but is discouraged; return values instead.
- **Reading a Pinia store at module top level in the router file.** Call `useXxxStore()`
  *inside* the guard, after Pinia is installed.
- **Putting authorization only in the client.** Guards are UX. The API must enforce
  permissions too; anyone can edit client code.
- **Heavy work in `beforeEach`** — it delays every navigation. Cache results and keep it fast.
- **Forgetting `meta` merging** — you don't need to repeat `requiresAuth` on every child.

## Exercise

1. Add `/login`, `/account` (requires auth) and `/admin` (requires the `admin` role) to
   your router. Use a simple `ref` for the current user for now. After login, redirect to
   the `redirect` query parameter if present.
2. Declare the `RouteMeta` interface and set `document.title` in `afterEach`.
3. Add an edit page with `onBeforeRouteLeave` that confirms when there are unsaved changes.
   Test it with both a `RouterLink` click and the browser Back button.
4. In a separate scratch project, set up file-based routing as shown. Add a
   `users/[id]/posts.vue` page and link to it with a typed `RouterLink`. Rename the file and
   confirm `vue-tsc` flags the now-broken link.
