---
description: "Extract reusable stateful logic into Vue composables: conventions, MaybeRefOrGetter inputs with toValue, cleanup with onScopeDispose, and a race-safe useFetch."
---

# 01 · Composables

In Level 1, every piece of logic lived inside a component. As soon as two components need
the same stateful behaviour — tracking the window size, syncing with `localStorage`,
fetching a resource — you want to share it. In Vue, the tool for that is a **composable**:
a plain function that uses the Composition API and returns reactive state.

```ts
const { data, error, loading } = useFetch<User>(() => `/api/users/${id.value}`)
const theme = useLocalStorage('theme', 'light')
const { width } = useWindowSize()
```

There's no special API. A composable is a function whose name starts with `use`, which
calls `ref`, `computed`, `watch`, lifecycle hooks and other composables, and returns what
the caller needs.

## Your first composable

Start from code you'd write in a component:

```vue
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const width = ref(window.innerWidth)
function update() { width.value = window.innerWidth }
onMounted(() => window.addEventListener('resize', update))
onUnmounted(() => window.removeEventListener('resize', update))
</script>
```

Move it into a function and return the state:

```ts title="src/composables/useWindowSize.ts"
import { ref, onMounted, onUnmounted, readonly } from 'vue'

export function useWindowSize() {
  const width = ref(window.innerWidth)
  const height = ref(window.innerHeight)

  function update() {
    width.value = window.innerWidth
    height.value = window.innerHeight
  }

  onMounted(() => window.addEventListener('resize', update, { passive: true }))
  onUnmounted(() => window.removeEventListener('resize', update))

  return { width: readonly(width), height: readonly(height) }
}
```

```vue
<script setup lang="ts">
import { computed } from 'vue'
import { useWindowSize } from '@/composables/useWindowSize'

const { width } = useWindowSize()
const isMobile = computed(() => width.value < 640)
</script>
```

Each component that calls `useWindowSize()` gets **its own** refs and its own listener —
composables share *logic*, not *state*. (For shared state, see Lesson 05 on Pinia, or the
module-level pattern in the How It Actually Works section.)

## Conventions that make composables pleasant

1. **Name them `useXxx`** and put them in `src/composables/`.
2. **Return a plain object of refs**, not a `reactive()`. The caller can then destructure
   without losing reactivity: `const { data, error } = useFetch(...)`.
3. **Return `readonly()` refs for state the caller shouldn't write**, and expose functions
   for the allowed changes.
4. **Accept refs *and* getters *and* plain values as inputs**, and normalise them with
   `toValue()` (below).
5. **Call them synchronously at the top of `setup`**, like lifecycle hooks — they usually
   register hooks or watchers that must attach to the current component.
6. **Clean up everything** you start: listeners, timers, subscriptions, requests.

## Flexible inputs: `MaybeRefOrGetter` and `toValue`

A composable that takes an input should react when that input changes. Callers have the
input in different forms — a literal, a ref, a prop — so accept all of them:

```ts
import { toValue, type MaybeRefOrGetter } from 'vue'

function useTitle(title: MaybeRefOrGetter<string>) {
  watchEffect(() => {
    document.title = toValue(title)   // unwraps a ref, calls a getter, returns a plain value as-is
  })
}

useTitle('Settings')                         // static
useTitle(pageTitle)                          // a ref
useTitle(() => `${unread.value} unread`)     // a getter
useTitle(() => props.title)                  // a prop — must be a getter
```

Because `toValue` runs *inside* `watchEffect`, whatever it reads is tracked. This is why
Level 1 said to pass `() => props.x` (not `props.x` or a destructured prop) into
composables: a getter can be re-read; a plain value can't.

## Worked example: a race-safe `useFetch`

Data fetching is the composable everyone writes, and most first versions have a race
condition: if the URL changes while a request is in flight, the older response can arrive
last and overwrite the newer one.

```ts title="src/composables/useFetch.ts"
import { ref, shallowRef, toValue, watchEffect, onWatcherCleanup, type MaybeRefOrGetter } from 'vue'

export function useFetch<T>(url: MaybeRefOrGetter<string | null>) {
  const data = shallowRef<T | null>(null)
  const error = shallowRef<Error | null>(null)
  const loading = ref(false)

  watchEffect(async () => {
    const target = toValue(url) // reading it here makes the effect track it
    if (!target) return
    const controller = new AbortController()
    onWatcherCleanup(() => controller.abort())

    loading.value = true
    error.value = null
    try {
      const res = await fetch(target, { signal: controller.signal })
      if (!res.ok) throw new Error(`HTTP ${res.status} for ${target}`)
      data.value = (await res.json()) as T
    } catch (e) {
      if ((e as Error).name !== 'AbortError') error.value = e as Error
    } finally {
      if (!controller.signal.aborted) loading.value = false
    }
  })

  return { data, error, loading }
}
```

Using it:

```vue title="src/components/UserCard.vue"
<script setup lang="ts">
import { useFetch } from '@/composables/useFetch'

interface User { id: number; name: string; email: string }
const { id } = defineProps<{ id: number }>()

const { data: user, error, loading } = useFetch<User>(() => `/api/users/${id}`)
</script>

<template>
  <p v-if="loading">Loading…</p>
  <p v-else-if="error" role="alert">{{ error.message }}</p>
  <article v-else-if="user">
    <h2>{{ user.name }}</h2>
    <a :href="`mailto:${user.email}`">{{ user.email }}</a>
  </article>
</template>
```

We tested the race with a fake `fetch` where user 1 takes 50 ms and user 2 takes 10 ms,
switching the id from 1 to 2 immediately. The recorded calls and final state:

```text
[ '/api/users/1', 'aborted /api/users/1', '/api/users/2' ] {"url":"/api/users/2"} false
```

The first request was aborted the moment the URL changed, so its slow response could never
land. Then we switched to an id that returns 404:

```text
error: HTTP 404 for /api/users/404 data kept: {"url":"/api/users/2"}
```

Note the design choice this reveals: on error, `data` keeps the last good value. That's
sometimes what you want (show stale data plus an error banner) and sometimes not; if not,
set `data.value = null` at the start of each run.

Three details make this work:

- **Only reactive reads before the first `await` are tracked.** `toValue(url)` is called
  synchronously at the top of the effect. Anything read after an `await` runs outside
  the tracking context and won't re-trigger the effect.
- **`onWatcherCleanup` is also called before the `await`**, for the same reason.
- **`shallowRef` for `data`**: API responses are replaced wholesale, not mutated, so there's
  no need to make every nested object reactive (Level 3 · 01 explains the cost).

## Cleanup outside components: `onScopeDispose`

`onUnmounted` only works when the composable is called from a component. Composables can
also run inside a Pinia store, inside another composable called from a store, or inside a
manual `effectScope()`. `onScopeDispose` covers all of those — it runs when the enclosing
*effect scope* stops, and a component's scope stops when it unmounts:

```ts title="src/composables/useEventListener.ts"
import { onScopeDispose, toValue, watch, type MaybeRefOrGetter } from 'vue'

export function useEventListener<K extends keyof WindowEventMap>(
  target: MaybeRefOrGetter<EventTarget | null | undefined>,
  event: K,
  handler: (e: WindowEventMap[K]) => void,
) {
  const stop = watch(
    () => toValue(target),
    (el, _old, onCleanup) => {
      if (!el) return
      el.addEventListener(event, handler as EventListener)
      onCleanup(() => el.removeEventListener(event, handler as EventListener))
    },
    { immediate: true },
  )
  onScopeDispose(stop)
  return stop
}
```

Accepting a ref as the target means it works with template refs that are `null` until
mount and may change. Stopping the watcher runs its cleanup, which removes the listener.
We ran it inside an `effectScope`, dispatched `resize`, stopped the scope and dispatched
again: the handler ran once (`resize handled 1`).

Composables compose. Here's `useLocalStorage` built on it:

```ts title="src/composables/useLocalStorage.ts"
import { ref, watch, type Ref } from 'vue'
import { useEventListener } from './useEventListener'

export function useLocalStorage<T>(key: string, initial: T): Ref<T> {
  const read = (): T => {
    try {
      const raw = localStorage.getItem(key)
      return raw === null ? initial : (JSON.parse(raw) as T)
    } catch {
      return initial
    }
  }

  const value = ref(read()) as Ref<T>
  watch(value, (v) => localStorage.setItem(key, JSON.stringify(v)), { deep: true })

  // keep tabs in sync: 'storage' fires in *other* tabs when this key changes
  useEventListener(window, 'storage', (e) => {
    if (e.key === key) value.value = read()
  })

  return value
}
```

## How It Actually Works

Composables work because of two runtime mechanisms you've already met:

1. **Reactivity is not tied to components.** `ref()` and `computed()` are plain objects
   with tracking built in; they work anywhere. That's why a composable can create state
   and hand it back.
2. **The "current instance" and "active effect scope".** When a component's `setup` runs,
   Vue sets a module-level current instance and activates the component's `EffectScope`.
   Any `watch`, `watchEffect` or `computed` created during that synchronous run is
   registered in that scope; `onMounted` and friends register on that instance. A
   composable called during setup is simply more code running during that window. When
   the component unmounts, Vue calls `scope.stop()`, which stops every effect created in
   it and runs every `onScopeDispose` callback.

This explains the two rules. Composables must be called **synchronously** in setup,
because after an `await` the current instance and scope have been reset — a watcher
created there isn't collected and leaks. And each call creates **fresh** state, because
`ref()` runs again on each call.

If you *want* shared state, create it at module level, outside the function:

```ts
const count = ref(0)                   // one instance, shared by every caller
export function useSharedCounter() {
  return { count, increment: () => count.value++ }
}
```

That works in a single-page app, but in server-side rendering a module-level ref is shared
across **all requests from all users**, which leaks data between them. That's one reason
Pinia exists (Lesson 05).

## Common mistakes

- **Returning `reactive({ ... })`** — callers who destructure lose reactivity. Return refs.
- **Taking `props.x` as a plain argument.** Take `MaybeRefOrGetter` and call `toValue`
  inside a tracking context.
- **Reading reactive inputs after `await` in `watchEffect`** — they aren't tracked.
- **Calling a composable inside an event handler or after `await`** — hooks and watchers
  won't be attached to the component.
- **Forgetting cleanup**, especially for listeners on `window` or `document` and for
  in-flight requests.
- **Reinventing a well-tested utility.** For common browser APIs, the community library
  VueUse (`@vueuse/core`) has hundreds of composables. Reading its source is also one of
  the best ways to learn composable design; the patterns above mirror its approach.

## Exercise

1. Write `useWindowSize` and show "mobile" or "desktop" in a component. Then rewrite it on
   top of `useEventListener`.
2. Write `useDebouncedRef(source, delay)` that returns a ref which updates `delay` ms after
   `source` stops changing. Use it with `useFetch` to build a search box that fetches
   `https://jsonplaceholder.typicode.com/users?q=...` (or a local JSON file) only after
   typing pauses.
3. Add a `refresh()` function to `useFetch` that re-runs the request for the current URL.
   (Hint: add a `const trigger = ref(0)` that the effect reads.)
4. Write a test that calls `useLocalStorage` inside `effectScope().run()`, writes a value,
   awaits `nextTick()`, and asserts the JSON in `localStorage`. Remember the Node 25+ flag
   from Level 1 · 10.
