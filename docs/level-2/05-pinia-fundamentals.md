---
description: "Share state across a Vue app with Pinia: setup and option stores, getters and actions, storeToRefs for safe destructuring, and when a store is the right tool."
---

# 05 · Pinia Fundamentals

Some state belongs to the whole app rather than to one component subtree: the signed-in
user, a shopping cart, feature flags, a cache of entities several pages display. **Pinia**
is Vue's official state-management library. A Pinia **store** is a reactive object that any
component can use, with devtools support, TypeScript inference, SSR safety and simple
testing. Output on this page was captured with Pinia 4.0.3.

## Installing

`create-vue --pinia` sets it up. Manually:

```bash
npm install pinia
```

```ts title="src/main.ts (excerpt)"
import { createPinia } from 'pinia'

app.use(createPinia())
```

Pinia 4 is ESM-only and requires `@vue/devtools-api` v8 alongside it; npm installs peer
dependencies automatically, so in practice nothing changes from Pinia 3 for most apps.

## Two ways to define a store

### Setup stores (recommended in this course)

A setup store is written exactly like a composable: refs are **state**, computeds are
**getters**, functions are **actions**. Return everything you want to expose.

```ts title="src/stores/auth.ts"
import { ref, computed } from 'vue'
import { defineStore } from 'pinia'

export interface User { id: number; name: string; roles: string[] }

export const useAuthStore = defineStore('auth', () => {
  const user = ref<User | null>(null)

  const isLoggedIn = computed(() => user.value !== null)
  const hasRole = (role: string) => user.value?.roles.includes(role) ?? false

  async function login(email: string, password: string) {
    const res = await fetch('/api/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    })
    if (!res.ok) throw new Error('Invalid email or password')
    user.value = (await res.json()) as User
  }

  function logout() {
    user.value = null
  }

  return { user, isLoggedIn, hasRole, login, logout }
})
```

The first argument, `'auth'`, is the store's **unique id** — used by devtools, SSR state
serialisation and plugins.

### Option stores

The object syntax mirrors the Options API:

```ts title="src/stores/cart.ts"
import { defineStore } from 'pinia'

interface CartItem { sku: string; qty: number }

export const useCartStore = defineStore('cart', {
  state: () => ({
    items: [] as CartItem[],
    coupon: '',
  }),
  getters: {
    count: (state) => state.items.reduce((n, i) => n + i.qty, 0),
  },
  actions: {
    add(sku: string) {
      const item = this.items.find((i) => i.sku === sku)
      if (item) item.qty++
      else this.items.push({ sku, qty: 1 })
    },
  },
})
```

Both styles produce the same kind of store. Setup stores are more flexible — they can use
`watch`, other composables and injected values — so they're what this course uses. Option
stores have one convenience setup stores lack, shown below: a built-in `$reset()`.

## Using a store

```vue title="src/components/CartBadge.vue"
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useCartStore } from '@/stores/cart'

const cart = useCartStore()
const { count } = storeToRefs(cart)   // reactive refs for state & getters
const { add } = cart                  // actions can be destructured directly
</script>

<template>
  <button type="button" @click="add('sku-123')">Add sample</button>
  <span class="badge">{{ count }}</span>
</template>
```

Calling `useCartStore()` in any component returns **the same instance** — our test printed
`same instance true` for two calls. The store is created lazily on first use.

### Destructuring: `storeToRefs`

A store is a `reactive()` object, so plain destructuring has the Lesson 04 problem. We added
three items, destructured `count` directly and via `storeToRefs`, then added a fourth:

```text
count 3 same instance true
storeToRefs isRef true true actions in storeToRefs? false
plain destructure stale 3 ref 4
```

The plain `count` stayed at 3; the ref updated to 4. `storeToRefs` returns refs only for
state and getters — actions aren't included (`actions in storeToRefs? false`), which is why
you destructure actions from the store itself. Alternatively, don't destructure: using
`cart.count` in templates and scripts is always reactive.

## Changing state

Inside actions, just assign. From components you can also write state directly
(`cart.coupon = 'SAVE10'`) — Pinia allows it, and devtools records it. For several
changes at once, `$patch` groups them into one devtools entry and one subscription event:

```ts
cart.$patch({ coupon: 'SAVE10' })                      // object form: merge
cart.$patch((state) => { state.items = []; state.coupon = '' })   // function form
```

Use the function form for arrays (the object form would replace them).

A team convention worth adopting: **components call actions; only actions change state.**
Direct writes are fine for prototypes, but actions give you one place to add validation,
logging and API calls later.

### Resetting

Option stores have `$reset()`, which calls `state()` again:

```text
after $reset 0 {"items":[],"coupon":""}
```

Setup stores don't — Pinia can't know how to recreate refs you created by hand. Calling it
on our setup store threw:

```text
🍍: Store "auth" is built using the setup syntax and does not implement $reset().
```

Write your own `reset` action in setup stores:

```ts
function $reset() {
  user.value = null
}
return { user, /* ... */ $reset }
```

## Stores using other stores

Call the other store's `useXxxStore()` inside your store's setup (or inside an action):

```ts title="src/stores/checkout.ts"
import { defineStore } from 'pinia'
import { computed } from 'vue'
import { useCartStore } from './cart'
import { useAuthStore } from './auth'

export const useCheckoutStore = defineStore('checkout', () => {
  const cart = useCartStore()
  const auth = useAuthStore()

  const canCheckout = computed(() => auth.isLoggedIn && cart.count > 0)

  return { canCheckout }
})
```

Avoid two stores that **both** read each other during setup — a circular dependency at
creation time. If you need that, one of them should access the other inside an action or
getter only.

## When to use a store (and when not)

| Situation | Use |
|-----------|-----|
| State used by one component | `ref` in that component |
| Logic reused across components, state separate each time | a composable |
| State shared by a component subtree (a form and its fields) | provide/inject |
| State shared across unrelated parts of the app, or across routes | a Pinia store |
| Server data (lists, details, caching, refetching) | a store, or a data-fetching library such as Pinia Colada (Lesson 07) |

Don't put everything in stores "just in case". Global state is harder to reason about than
local state; move state up only when something actually needs it.

## How It Actually Works

`createPinia()` creates a root object holding an `effectScope` and a `state` ref (a map of
store id → state). `app.use(pinia)` provides it to the app with `app.provide(piniaSymbol, pinia)`
and marks it as the *active* Pinia.

`defineStore(id, setup)` doesn't create anything; it returns the `useXxxStore` function.
The first time that function is called, it finds the Pinia instance (via `inject` if
called in a component, otherwise the active Pinia), and if no store with that id exists
yet, it:

1. runs your setup function inside a child `effectScope` of Pinia's scope — so the refs,
   computeds and watchers you create belong to Pinia, **not** to the component that
   happened to call it first; that's why a store survives the component unmounting;
2. finds the refs in the returned object and connects them to `pinia.state.value[id]`, so
   all store state lives in one serialisable tree (used for SSR hydration and devtools);
3. wraps each function in an action wrapper that powers `$onAction` subscriptions;
4. creates a `reactive()` object containing everything plus the `$patch`, `$subscribe`,
   `$onAction`, `$dispose` and (for option stores) `$reset` methods, and caches it.

Subsequent calls return the cached reactive object — hence `same instance true`. Because
the store object is `reactive`, refs inside it are unwrapped (`cart.count`, not
`cart.count.value`), and destructuring breaks reactivity just like any reactive object.
`storeToRefs` walks the store, skips functions, and uses `toRef` for everything else.

On the server, you create a fresh `createPinia()` per request, so each request gets its own
map of stores — solving the module-level shared-state leak described in Lesson 01.

## Common mistakes

- **Destructuring state without `storeToRefs`** — values stop updating.
- **Calling `useXxxStore()` at module top level** (for example at the top of
  `router/index.ts`) before `app.use(pinia)`. Pinia 4.0.3 threw this in our test:
  `[🍍]: "getActivePinia()" was called but there was no active Pinia. Are you trying to use a store before calling "app.use(pinia)"?`
  Call it inside functions that run later (guards, actions, setup).
- **Forgetting to return state from a setup store.** Unreturned refs aren't part of the
  store's state: devtools won't show them and SSR won't serialise them.
- **Expecting `$reset()` on setup stores.** Write your own.
- **Storing derived data as state** (for example a `total` ref you keep in sync by hand).
  Use a getter (`computed`).
- **Putting non-serialisable things in state** (class instances with methods, DOM nodes,
  sockets). Keep them outside state or mark them with `markRaw`.

## Exercise

1. Move the habit list from the Level 1 project into a `useHabitsStore` setup store with
   actions `add`, `toggle` and `remove`, and a `doneToday` getter. `App.vue` should become
   mostly template.
2. Add a `HabitsSummary` component somewhere far from the list (for example in a header)
   that shows the same count, using `storeToRefs`.
3. Write your own `$reset` action. Wire a "Clear all" button to it with a confirmation.
4. Convert the store to option-store syntax and compare which parts got longer and which
   shorter. Then convert it back.
