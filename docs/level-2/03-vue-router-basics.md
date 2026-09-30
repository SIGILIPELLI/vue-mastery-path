---
description: "Add client-side routing with Vue Router: route records, RouterLink and RouterView, dynamic params as props, nested routes, programmatic navigation and 404s."
---

# 03 · Vue Router Basics

A single-page app still needs URLs: people bookmark pages, share links and press the Back
button. **Vue Router** maps URLs to components, keeps the address bar in sync with what's
on screen, and does it without full page reloads. This lesson covers the core API; the
next one covers guards, metadata, lazy loading and the file-based routing that's built into
Vue Router 5.

## Setup

If you chose `--router` when scaffolding, this is already in place:

```ts title="src/router/index.ts"
import { createRouter, createWebHistory } from 'vue-router'
import HomeView from '@/views/HomeView.vue'

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes: [
    { path: '/', name: 'home', component: HomeView },
    { path: '/about', name: 'about', component: () => import('@/views/AboutView.vue') },
  ],
})

export default router
```

```ts title="src/main.ts (excerpt)"
app.use(router)
```

Otherwise, `npm install vue-router` and add the two files yourself.

### History modes

| Function | URLs look like | Server needs |
|----------|----------------|--------------|
| `createWebHistory()` | `/users/42` | to serve `index.html` for every unknown path (Level 4 · 08) |
| `createWebHashHistory()` | `/#/users/42` | nothing — works on any static host |
| `createMemoryHistory()` | (not reflected in the address bar) | used for SSR and tests |

Use web history for real apps. Hash history is a fallback for hosts where you can't
configure a catch-all rewrite; hash URLs are also less friendly for SEO.

## `RouterView` and `RouterLink`

```vue title="src/App.vue"
<template>
  <header>
    <nav>
      <RouterLink to="/">Home</RouterLink>
      <RouterLink :to="{ name: 'users' }">Users</RouterLink>
      <RouterLink to="/about">About</RouterLink>
    </nav>
  </header>
  <main>
    <RouterView />
  </main>
</template>
```

`<RouterView />` renders the component matched by the current URL. `<RouterLink>` renders
an `<a>` with a real `href` (so middle-click, "copy link" and screen readers work) and
intercepts normal clicks to navigate without a reload.

The active link gets classes and ARIA for free. From a test with two links where the
current URL was `/users/1`:

```html
<a aria-current="page" href="/users/1" class="router-link-active router-link-exact-active">u1</a>
<a href="/users/1/posts" class="">posts</a>
```

`router-link-active` applies when the link's route is part of the current match (including
parents); `router-link-exact-active` only when it's the exact route. Style those classes to
highlight the current section.

## Dynamic routes

A segment starting with `:` is a **param**:

```ts
{ path: '/users/:id', name: 'user', component: UserView }
```

`/users/42` matches, with `route.params.id === '42'` — params are always **strings** (or
arrays of strings for repeatable params). Convert them yourself.

### Params as props

Reading `useRoute().params` inside a component ties the component to the router. Prefer
passing params as props:

```ts
{ path: '/users/:id', name: 'user', component: UserView, props: true }
```

```vue title="src/views/UserView.vue"
<script setup lang="ts">
import { computed } from 'vue'
import { useFetch } from '@/composables/useFetch'

const { id } = defineProps<{ id: string }>()
const { data: user, loading } = useFetch<{ name: string }>(() => `/api/users/${id}`)
const numericId = computed(() => Number(id))
</script>
```

Now the component is reusable and trivially testable with `mount(UserView, { props: { id: '1' } })`.
`props` can also be an object (static props) or a function `(route) => ({ ... })` for
converting and combining params and query values:

```ts
props: (route) => ({ id: Number(route.params.id), tab: route.query.tab ?? 'overview' })
```

### The same component is reused

This is the #1 routing surprise. Navigating from `/users/1` to `/users/2` does **not**
create a new `UserView` — both URLs match the same route, so Vue Router keeps the instance
and just updates its props. We logged `setup` and `onMounted` in a user component and
navigated from user 1 to user 2:

```text
user 2 (route 2) |  / 
setup 1, mounted
```

The heading updated to user 2, but `setup` ran only once. Any code in `setup` that read the
id **once** (a non-reactive fetch, a `ref(props.id)` copy) is now stale. That's why the
`useFetch` above takes a **getter** — it re-runs when `id` changes. If you truly need a
fresh component per param, put a key on the view: `<RouterView :key="$route.fullPath" />`
— at the cost of destroying and recreating it on every navigation.

## Query strings and hashes

```ts
router.push({ name: 'users', query: { page: 2, sort: 'name' } })   // /users?page=2&sort=name
```

In a component, `useRoute().query.page` is `'2'` (a string), or an array if repeated. The
query is a great place for UI state you want shareable: filters, search, pagination, the
selected tab.

## Nested routes

Real layouts nest: a user page with "Profile" and "Posts" tabs, each with its own URL.
Child routes render inside the parent's own `<RouterView>`:

```ts
{
  path: '/users/:id',
  component: UserLayout,
  props: true,
  children: [
    { path: '', name: 'user-profile', component: UserProfile },     // /users/42
    { path: 'posts', name: 'user-posts', component: UserPosts },    // /users/42/posts
  ],
}
```

```vue title="src/views/UserLayout.vue (template)"
<template>
  <h1>User {{ id }}</h1>
  <nav>
    <RouterLink :to="{ name: 'user-profile', params: { id } }">Profile</RouterLink>
    <RouterLink :to="{ name: 'user-posts', params: { id } }">Posts</RouterLink>
  </nav>
  <RouterView />
</template>
```

Child paths without a leading `/` are relative to the parent. The empty-path child renders
at the parent's own URL.

## Programmatic navigation

```ts
import { useRouter } from 'vue-router'

const router = useRouter()

await router.push({ name: 'user', params: { id: 42 } })   // adds a history entry
await router.replace('/login')                           // replaces the current entry
router.back()                                            // same as history.back()
```

`push` returns a promise that resolves when navigation finishes. It resolves to
`undefined` on success, or to a **navigation failure** if the navigation was aborted,
cancelled or redundant. Pushing the current URL again resolved to a failure whose string
form was:

```text
Error: Avoided redundant navigation to current location: "/admin".
```

It's resolved, not thrown — so `await router.push()` doesn't need a `try/catch` for these.
Use `isNavigationFailure(result)` when you need to know.

Named routes are more robust than string paths: rename a URL in one place and every
`{ name: 'user' }` link keeps working. `router.resolve()` shows what a location becomes:

```ts
router.resolve({ name: 'user', params: { id: 42 }, query: { tab: 'posts' } }).fullPath
// '/users/42?tab=posts'
```

## Not-found pages

A catch-all route with a custom regex matches anything the other routes don't:

```ts
{ path: '/:pathMatch(.*)*', name: 'not-found', component: NotFoundView }
```

Visiting `/nope/deep` matched it, with params `{"pathMatch":["nope","deep"]}` — the
trailing `*` makes the param repeatable, so it's split into segments. Order doesn't matter
in Vue Router 4+: routes are ranked by specificity, so the catch-all loses to any more
specific route.

## How It Actually Works

When you create the router, it compiles every route path into a **matcher**: a regular
expression plus a score. Static segments score higher than params, params higher than
custom regexes and wildcards. When a URL comes in, the router tries matchers in score
order and takes the first match — which is why route order in the array rarely matters.

The current location is stored in `router.currentRoute`, a `shallowRef`. `useRoute()`
returns a reactive view of it; `<RouterView>` reads it inside its render function, so
every navigation re-renders the `RouterView`s. Each `RouterView` knows its **depth** (a
nested `RouterView` injects its parent's depth and adds one) and renders
`currentRoute.matched[depth]` — the matched record at its level. A nested URL like
`/users/42/posts` has `matched = [userLayoutRecord, userPostsRecord]`: depth 0 renders the
layout, depth 1 renders the posts.

When the new match at a depth has the **same record** as before, the `RouterView` produces a
vnode for the same component type, and Vue's renderer patches it in place instead of
remounting — that's the reuse you saw above. With `props: true`, the router passes
`route.params` as props when rendering, so the reused instance gets new props.

`createWebHistory` wraps the browser's History API: `router.push` validates the navigation
(running guards, next lesson), then calls `history.pushState`, then updates `currentRoute`.
Pressing Back fires `popstate`, which the router turns into a navigation through the same
pipeline.

## Common mistakes

- **Using `<a href>` for internal links** — it triggers a full page reload and loses
  in-memory state. Use `<RouterLink>`.
- **Reading params once in `setup`** and missing param changes on the same route.
- **Forgetting params are strings.** `params.id === 42` is always false.
- **Using `useRoute()` outside `setup`** — like other composables, call it at the top level
  and keep the result.
- **Deploying web history without a server fallback** — refreshing `/users/42` returns 404
  from the server. Level 4 · 08 covers the fix for common hosts.
- **Using `router.push` for external URLs** — use `window.location.href` or a plain `<a>`.

## Exercise

Create a small "people directory" with the JSONPlaceholder API
(`https://jsonplaceholder.typicode.com/users`) or a local JSON file:

1. Routes: `/` (home), `/users` (list), `/users/:id` with children `''` (profile) and
   `posts` (that user's posts from `/posts?userId=:id`), and a not-found page.
2. Pass `id` as a prop converted to a number with a props function.
3. Add "Previous" / "Next" links on the profile that go to `id - 1` / `id + 1`. Verify
   the data changes when you click them — and then break it deliberately by fetching with
   a non-reactive value, to see the reuse problem for yourself.
4. Add a `?sort=name|email` query to the list page, bound to a `<select>`, so that the sort
   survives a reload and a shared link.
