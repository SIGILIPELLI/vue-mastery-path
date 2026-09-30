---
description: "Advanced Pinia: $subscribe and $onAction, writing a typed persistence plugin, hot module replacement, and unit-testing stores and components with @pinia/testing."
---

# 06 · Pinia in Depth

Lesson 05 covered defining and using stores. This lesson covers what you need once stores
are central to an app: reacting to store changes, extending every store with a **plugin**,
keeping state during hot reloads, and **testing** — both the stores themselves and the
components that use them. Every test on this page ran green with Pinia 4.0.3 and
`@pinia/testing` 2.0.1.

## Subscribing to state changes: `$subscribe`

`store.$subscribe(callback)` runs after state changes, with a description of the mutation
and the new state. Unlike a `watch` on the whole store, a `$patch` produces **one**
notification however many properties it changes.

We subscribed to a store with fields `a` and `b`, made two direct writes in the same tick,
then a `$patch`, then added a second subscription with `flush: 'sync'` and wrote twice
more:

```ts
s.$subscribe((m, state) => ev.push(`${m.type} a=${state.a} b=${state.b}`))
s.a = 1; s.b = 1; await nextTick()
s.$patch({ a: 2 }); await nextTick()
s.$subscribe((m) => ev.push('sync ' + m.type), { flush: 'sync' })
s.a = 3; s.b = 3; await nextTick()
```

```text
direct a=1 b=1
patch object a=2 b=1
sync direct
sync direct
direct a=3 b=3
```

- Direct writes are **batched** like watchers: two writes, one `direct` event.
- `$patch` reports its own type (`patch object` or `patch function`).
- `flush: 'sync'` fires per write — useful for debugging, rarely otherwise.

Subscriptions made in a component are removed when it unmounts. Pass
`{ detached: true }` to keep one alive (for example in a logging plugin).

One subtlety we observed: while a `$patch` is being applied, Pinia pauses its `direct`
notifications until the next tick, so a direct write made in the **same tick** right after
a `$patch` doesn't produce a separate `direct` event.

## Observing actions: `$onAction`

`$onAction` runs before each action, with hooks for success and failure:

```ts
const unsubscribe = auth.$onAction(({ name, args, after, onError }) => {
  const start = performance.now()
  after(() => console.debug(`${name} took ${Math.round(performance.now() - start)} ms`))
  onError((err) => reportError(err, { action: name, args }))
})
```

In our test, calling `auth.login('ada')` recorded `[ 'login(ada)', 'after login' ]`.
`after` waits for async actions to resolve. This is the natural place for analytics,
performance timing and centralised error reporting.

## Plugins

A Pinia plugin is a function called once **for every store** when the store is created. It
receives the store and the options it was defined with, and can:

- add properties to every store (return an object, or assign to `store`);
- subscribe to state or actions;
- read custom options you add to `defineStore`.

Here's a small persistence plugin that saves any store opted in with `persist: true`:

```ts title="src/stores/plugins/persist.ts"
import type { PiniaPluginContext } from 'pinia'

declare module 'pinia' {
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  export interface DefineStoreOptionsBase<S, Store> {
    /** Persist this store's state to localStorage under `pinia:<id>`. */
    persist?: boolean
  }
}

export function persistPlugin({ store, options }: PiniaPluginContext) {
  if (!options.persist) return

  const key = `pinia:${store.$id}`
  const saved = localStorage.getItem(key)
  if (saved) {
    try {
      store.$patch(JSON.parse(saved))
    } catch {
      localStorage.removeItem(key)
    }
  }

  store.$subscribe((_mutation, state) => {
    localStorage.setItem(key, JSON.stringify(state))
  })
}
```

The `declare module 'pinia'` block teaches TypeScript that `defineStore` accepts a
`persist` option. Register the plugin once:

```ts title="src/main.ts (excerpt)"
import { createPinia } from 'pinia'
import { persistPlugin } from './stores/plugins/persist'

app.use(createPinia().use(persistPlugin))
```

And a store that opts in, passing options as the third argument of a setup store:

```ts title="src/stores/todos.ts"
import { ref, computed } from 'vue'
import { defineStore, acceptHMRUpdate } from 'pinia'

export interface Todo { id: number; title: string; done: boolean }

export const useTodosStore = defineStore(
  'todos',
  () => {
    const items = ref<Todo[]>([])
    const filter = ref<'all' | 'open' | 'done'>('all')

    const visible = computed(() =>
      filter.value === 'all' ? items.value : items.value.filter((t) => t.done === (filter.value === 'done')),
    )

    function add(title: string) {
      const trimmed = title.trim()
      if (!trimmed) throw new Error('Title is required')
      // derive the id from state: a module counter would restart at 1 after a reload
      const id = Math.max(0, ...items.value.map((t) => t.id)) + 1
      items.value.push({ id, title: trimmed, done: false })
    }
    function toggle(id: number) {
      const t = items.value.find((t) => t.id === id)
      if (t) t.done = !t.done
    }
    function $reset() {
      items.value = []
      filter.value = 'all'
    }

    return { items, filter, visible, add, toggle, $reset }
  },
  { persist: true },
)

if (import.meta.hot) import.meta.hot.accept(acceptHMRUpdate(useTodosStore, import.meta.hot))
```

Two details in this store came from bugs we hit while testing it:

- The first version used a module-level `let nextId = 1`. After a "reload" the restored
  todos had ids 1..n but the counter started again at 1, producing duplicate ids — and
  duplicate `:key`s in any list rendering them. Deriving the id from state fixed it; the
  test below checks for it.
- The last line enables **hot module replacement** for the store: when you edit the file
  during `npm run dev`, Pinia swaps in the new actions and getters while keeping the
  current state, instead of reloading the page.

For production apps, well-maintained community plugins exist for persistence (for example
`pinia-plugin-persistedstate`); writing one yourself, as above, is the best way to
understand what they do and what to configure.

## Testing stores

A store is plain TypeScript — test it without mounting anything. Give each test a fresh
Pinia so state doesn't leak between tests:

```ts title="src/stores/__tests__/todos.spec.ts"
import { describe, it, expect, beforeEach } from 'vitest'
import { createPinia, setActivePinia } from 'pinia'
import { createApp, nextTick } from 'vue'
import { useTodosStore } from '../todos'
import { persistPlugin } from '../plugins/persist'

// Pinia only runs plugins once it is installed in an app, so tests that
// exercise a plugin need a (never mounted) app.
function freshPinia() {
  const pinia = createPinia().use(persistPlugin)
  createApp({}).use(pinia)
  setActivePinia(pinia)
}

describe('todos store', () => {
  beforeEach(() => {
    localStorage.clear()
    freshPinia()
  })

  it('adds trimmed todos and rejects empty titles', () => {
    const todos = useTodosStore()
    todos.add('  Write tests ')
    expect(todos.items).toEqual([{ id: 1, title: 'Write tests', done: false }])
    expect(() => todos.add('   ')).toThrow('Title is required')
  })

  it('filters by status', () => {
    const todos = useTodosStore()
    todos.add('a'); todos.add('b'); todos.toggle(1)
    todos.filter = 'done'
    expect(todos.visible.map((t) => t.title)).toEqual(['a'])
  })

  it('persists state and restores it in a new pinia', async () => {
    useTodosStore().add('survive a reload')
    await nextTick()                     // $subscribe callbacks are flushed like watchers

    freshPinia()   // simulate a fresh page load
    const restored = useTodosStore()
    expect(restored.items[0]?.title).toBe('survive a reload')
    restored.add('after reload')
    expect(restored.items.map((t) => t.id)).toEqual([1, 2])   // no duplicate ids
  })
})
```

Our first attempt used the more common `setActivePinia(createPinia().use(persistPlugin))`
and the persistence test **failed** — the plugin never ran. Pinia queues plugins added
with `.use()` until the Pinia instance is installed in an app with `app.use(pinia)`. For
store tests that don't involve plugins, `setActivePinia(createPinia())` alone is fine; to
exercise a plugin, install Pinia into a throwaway `createApp({})` as shown.

## Testing components that use stores

For component tests you usually don't want real actions to run (they might call APIs).
`@pinia/testing` provides `createTestingPinia()`, which stubs every action with a spy and
lets you seed initial state:

```bash
npm install -D @pinia/testing
```

```vue title="src/components/TodoCount.vue"
<script setup lang="ts">
import { useTodosStore } from '@/stores/todos'
const todos = useTodosStore()
</script>
<template><button type="button" @click="todos.add('From component')">{{ todos.items.length }} todos</button></template>
```

```ts title="src/components/__tests__/TodoCount.spec.ts"
import { it, expect, vi } from 'vitest'
import { mount } from '@vue/test-utils'
import { createTestingPinia } from '@pinia/testing'
import TodoCount from '../TodoCount.vue'
import { useTodosStore } from '@/stores/todos'

it('renders the count and calls add on click', async () => {
  const wrapper = mount(TodoCount, {
    global: {
      plugins: [
        createTestingPinia({
          createSpy: vi.fn,
          initialState: { todos: { items: [{ id: 1, title: 'Seeded', done: false }] } },
        }),
      ],
    },
  })
  const todos = useTodosStore()

  expect(wrapper.text()).toBe('1 todos')
  await wrapper.find('button').trigger('click')
  expect(todos.add).toHaveBeenCalledWith('From component')
  expect(todos.items).toHaveLength(1)   // actions are stubbed by default
})
```

`createSpy: vi.fn` tells it to use Vitest's spies (with `globals: true` in the Vitest config
you can omit it). Because actions are stubbed, clicking the button records the call but
doesn't change state — the test asserts both. Pass `stubActions: false` when you want real
actions but still want seeded state.

Our run of all four tests:

```text
 ✓ src/stores/__tests__/todos.spec.ts > todos store > adds trimmed todos and rejects empty titles 5ms
 ✓ src/stores/__tests__/todos.spec.ts > todos store > filters by status 1ms
 ✓ src/stores/__tests__/todos.spec.ts > todos store > persists state and restores it in a new pinia 1ms
 ✓ src/components/__tests__/TodoCount.spec.ts > renders the count and calls add on click 16ms
 Test Files  2 passed (2)
      Tests  4 passed (4)
```

## Pinia and SSR (preview)

With server-side rendering, each request must get its own Pinia (`createPinia()` inside
the per-request app factory). After rendering, the server serialises `pinia.state.value`
into the HTML, and the client sets `pinia.state.value = window.__PINIA_STATE__` before
mounting, so stores start with the server's data instead of refetching. Nuxt does all of
this for you. Level 4 · 01 builds it by hand — including why you must escape that
serialised JSON.

## How It Actually Works

`pinia.use(plugin)` pushes the plugin into a list — but if Pinia hasn't been installed in an
app yet, it goes into a *to-be-installed* list that's only moved over inside Pinia's
`install(app)` method. That's the behaviour our first test tripped over. When a store is
created, Pinia loops through the installed plugins and calls each one inside the store's
own effect scope (`runWithContext`), merging any returned properties into the store.
Because the plugin runs in the store's scope, a `watch` or `$subscribe` created in the
plugin lives exactly as long as the store.

`$subscribe` is implemented with a `watch` on the store's slice of `pinia.state` with
`deep: true`, plus a separate list of subscribers that `$patch` notifies directly. During a
`$patch`, Pinia sets an `isListening` flag to `false` so the deep watcher doesn't *also*
report the same changes as `direct`; it re-enables listening after `nextTick()`. That's the
source of the same-tick subtlety noted above.

`$onAction` works because every action returned from a setup store is wrapped at creation
time. The wrapper calls each `$onAction` listener with `after`/`onError` registration
functions, then calls your action, and if the result is a promise, chains the `after` or
`onError` callbacks onto it.

`createTestingPinia` creates a normal Pinia and installs a plugin of its own that, for each
store, replaces every action with `createSpy(originalAction)` (or a spy that doesn't call
through when `stubActions` is true), wraps `$patch` in a spy too, and deep-merges
`initialState` into `store.$state` — no magic, just the plugin API you used above. It also
has a `fakeApp: true` option that does the throwaway `createApp({}).use(pinia)` for you,
for tests that use the testing Pinia without mounting a component.

## Common mistakes

- **Expecting plugins to run in tests without installing Pinia** in an app.
- **Module-level counters or caches in store files** — they don't reset with the store and
  don't survive (or are shared, in SSR) the way state does.
- **Persisting everything.** Persist user preferences and drafts, not server data that can
  be refetched and may be stale or user-specific. Never persist tokens in `localStorage`
  if you can avoid it — any XSS can read them (Level 3 · 09).
- **Reusing one Pinia across tests** — state leaks and tests pass or fail depending on
  order. Create a fresh one in `beforeEach`.
- **Forgetting `acceptHMRUpdate`** and then wondering why editing a store reloads the page
  and loses state.

## Exercise

1. Add the persistence plugin to your habit store from Lesson 05's exercise. Extend the
   plugin so `persist` can also be an object `{ key?: string; pick?: string[] }` that
   persists only some state keys. Update the `declare module` types.
2. Write a plugin that adds a `$loading` ref to every store and automatically sets it to
   `true` while any async action is running (use `$onAction` with `after`/`onError`).
   Test it with a store whose action awaits a timer.
3. Write a component test with `createTestingPinia` for a component that shows an error
   message when `store.error` is set. Seed `error` via `initialState`.
4. Temporarily move `createPinia()` in your store test to module scope (outside
   `beforeEach`) and watch which test fails when you run them in a different order with
   `npx vitest run --sequence.shuffle`.
