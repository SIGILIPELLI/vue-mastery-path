---
description: "How Vue reactivity really works: proxies, track and trigger, effects and the scheduler, shallowRef, markRaw, readonly, customRef, effectScope and computed stability."
---

# 01 · Reactivity in Depth

Levels 1 and 2 used reactivity as a black box with a few rules ("don't destructure",
"getters for props"). This lesson opens the box. Once you can picture what `track` and
`trigger` do, every one of those rules becomes obvious, and you can make deliberate
performance choices — shallow state, raw objects, stable computeds — instead of guessing.
All outputs are from Vue 3.5.43.

## The three pieces

Vue's reactivity system (`@vue/reactivity`, usable on its own) has three parts:

1. **Reactive data** — objects that know when they're read and written: `reactive()`
   proxies, `ref` objects, computeds.
2. **Effects** — functions that run and re-run: component render functions, `watch` and
   `watchEffect` callbacks, computed getters.
3. **A dependency graph** connecting them: which effects read which properties.

When an effect runs, Vue remembers it as the "active subscriber". Every reactive read
during that run calls **`track`**, adding an edge "this effect depends on this property".
Every reactive write calls **`trigger`**, which finds the effects depending on that
property and notifies them. That's the whole idea; everything else is detail and
optimisation.

A minimal version, stripped of every optimisation, looks like this:

```ts
let activeEffect: (() => void) | null = null
const deps = new WeakMap<object, Map<PropertyKey, Set<() => void>>>()

function track(target: object, key: PropertyKey) {
  if (!activeEffect) return
  let byKey = deps.get(target)
  if (!byKey) deps.set(target, (byKey = new Map()))
  let effects = byKey.get(key)
  if (!effects) byKey.set(key, (effects = new Set()))
  effects.add(activeEffect)
}

function trigger(target: object, key: PropertyKey) {
  deps.get(target)?.get(key)?.forEach((effect) => effect())
}

function reactive<T extends object>(obj: T): T {
  return new Proxy(obj, {
    get(target, key, receiver) {
      track(target, key)
      const value = Reflect.get(target, key, receiver)
      return typeof value === 'object' && value !== null ? reactive(value) : value
    },
    set(target, key, value, receiver) {
      const old = Reflect.get(target, key, receiver)
      const ok = Reflect.set(target, key, value, receiver)
      if (!Object.is(old, value)) trigger(target, key)
      return ok
    },
  })
}

function effect(fn: () => void) {
  const run = () => { activeEffect = run; try { fn() } finally { activeEffect = null } }
  run()
}
```

The real implementation adds: a cache so `reactive(obj)` always returns the same proxy;
cleanup of stale dependencies between runs; batching through a scheduler; special handlers
for arrays, `Map` and `Set`; `has`/`deleteProperty`/`ownKeys` traps; and — since 3.5 — a
doubly-linked-list dependency structure with version counters that makes tracking cheaper
and lets computeds skip work when nothing actually changed.

## Precise tracking, observed

Tracking is per **property**, not per object. An effect that reads only `obj.a`:

```ts
const obj = reactive({ a: 1, b: 2 })
watchEffect(() => { runs++; obj.a })
obj.b = 3   // → runs stays 1
obj.a = 5   // → runs becomes 2
```

```text
effect reading only a, after b changed, runs = 1
after a changed, runs = 2
```

Arrays track indexes and `length` separately. An effect that read only `arr.length` did
**not** re-run when we wrote `arr[0] = 9`, but did when we `push`ed:

```text
reading length, index write -> runs 1
push -> runs 2
```

This precision is why Vue rarely needs manual memoisation: an effect re-runs only if
something it actually read changed.

## Identity: proxies are not the original

`reactive(raw)` returns a proxy; nested objects are wrapped lazily, on access. Putting a raw
object into a reactive array and reading it back gives you a proxy:

```ts
const list = reactive([{ id: 1 }])
const item = { id: 2 }
list.push(item)
list[1] === item        // false
list.includes(item)     // true  — Vue patches includes/indexOf to also search raw values
list.indexOf(item)      // 1
```

```text
stored as proxy: false includes(raw): true indexOf(raw): 1
```

`toRaw(proxy)` gives back the original. Use it when handing data to code that must not
trigger tracking or can't handle proxies (`structuredClone`, some charting libraries,
`postMessage`), and treat the result as read-only.

## Opting out: shallow and raw

Deep reactivity has a cost: every nested object accessed becomes a proxy, and every
property read goes through a trap. For large data that you only ever **replace** — API
responses, a 10,000-row table, a GeoJSON blob — that cost buys nothing.

### `shallowRef`

`shallowRef` tracks only `.value` itself:

```ts
const rows = shallowRef([{ done: false }])
watch(rows, () => log.push('shallow fired'))

rows.value[0].done = true      // not tracked — nothing happens
triggerRef(rows)               // force a notification
rows.value = [...rows.value]   // replacing .value is tracked
```

```text
after mutate | shallow fired | shallow fired
```

The mutation alone did nothing; `triggerRef` and replacement each fired. The idiomatic
pattern is **immutable updates**: `rows.value = rows.value.map(...)`.

`shallowReactive` is the object equivalent: only top-level properties are reactive
(`isReactive(sr.nested)` was `false`).

### `markRaw`

`markRaw(obj)` flags an object so Vue **never** proxies it, even inside reactive state:

```ts
const map = markRaw(new maplibregl.Map({ container: 'map' }))
const state = reactive({ map })   // state.map is the real instance
```

Use it for class instances with internal state that proxying would break or slow down:
map and chart instances, editor instances, WebSocket objects, Three.js scenes. We verified
`isReactive(holder.lib)` was `false` for a marked object inside `reactive()`.

You've already seen Vue itself recommend this. Storing **component definitions** in a
`ref` produced this warning in Level 1's KeepAlive test:

```text
[Vue warn]: Vue received a Component that was made a reactive object. This can lead to unnecessary performance overhead and should be avoided by marking the component with `markRaw` or using `shallowRef` instead of `ref`.
```

For "which component to show" state, use `shallowRef(ComponentA)`.

### `readonly`

`readonly(source)` returns a proxy that rejects writes (with a warning in dev), deeply,
while still reflecting changes made to the source:

```text
[Vue warn] Set operation on key "name" failed: target is readonly. Proxy({ name: 'Ada' })
readonly nested write ignored: Ada true
readonly view sees source change: Grace
```

This is exactly what props are, and what you should expose from composables and
provide/inject for state consumers mustn't mutate.

## `customRef`: control track and trigger yourself

`customRef` gives you the `track` and `trigger` functions directly. A debounced ref:

```ts title="src/composables/useDebouncedRef.ts"
import { customRef } from 'vue'

export function useDebouncedRef<T>(value: T, delay = 200) {
  let timer: ReturnType<typeof setTimeout> | undefined
  return customRef<T>((track, trigger) => ({
    get() {
      track()
      return value
    },
    set(newValue) {
      clearTimeout(timer)
      timer = setTimeout(() => {
        value = newValue
        trigger()
      }, delay)
    },
  }))
}
```

Three quick writes, then 250 ms later (with fake timers):

```text
immediately: ""
after 250ms: vue watch saw [ 'vue' ]
```

The watcher saw a single change with the final value. Bind it with `v-model` for a search
box that only updates its dependants after typing pauses.

## Computed stability

A computed notifies its dependants when its value changes by `Object.is`. A getter that
returns a **new object** every time is "changed" every time it recomputes, even if the
contents are identical — so watchers and components that depend on it do extra work.

Since Vue 3.4, the getter receives the **previous value**, so you can return it when
nothing meaningful changed:

```ts
const summary = computed((prev?: { total: number }) => {
  const next = { total: obj.a > 5 ? 1 : 0 }
  return prev && prev.total === next.total ? prev : next
})
```

With that, changing `obj.a` from 6 to 7 to 8 (all "> 5") triggered the watcher on
`summary` **zero** times; the naive version (returning a fresh object) fires whenever its
input changes. Use this for computeds that produce objects or arrays consumed by expensive
effects.

## `effectScope`: grouping effects

An effect scope collects every effect created inside `scope.run()` so you can stop them all
at once:

```ts
const scope = effectScope()
scope.run(() => {
  watchEffect(() => log.push(n.value))
  computed(() => n.value)
})
n.value = 1   // logged
scope.stop()
n.value = 2   // not logged
```

```text
effectScope log [ 0, 1 ]
```

Every component has a scope (that's how its watchers are cleaned up on unmount), and so
does every Pinia store. You'll use `effectScope` directly in tests of composables and when
writing libraries that create reactive state outside components.

## How It Actually Works

Beyond the sketch above, three real-implementation details explain behaviours you'll see:

- **Dependency cleanup.** Each run of an effect re-collects dependencies. In 3.5, each
  dependency link carries a version; after a run, links not touched in that run are
  removed. That's why a `watchEffect` stopped reacting to `b` once a branch stopped reading
  it (Level 1 · 05).
- **Scheduling.** `trigger` doesn't run component renders or watchers immediately. Each
  effect has a *scheduler*; for components and default watchers, it queues a job. Jobs are
  deduplicated and flushed in a microtask, sorted so parents render before children and
  `pre` watchers before renders. Many writes → one render.
- **Computed versions.** A computed stores the global version counter from its last
  evaluation. When read, if no reactive value anywhere has changed since then, it returns
  the cache without checking anything; otherwise it checks whether each of *its*
  dependencies' versions changed, and only recomputes if one did. If the recomputed value
  is `Object.is`-equal to the old one, its own version doesn't bump, so its dependants skip
  work too. That cascade is what the "prev" trick above plugs into.

## Common mistakes

- **Deep `ref` for large, replace-only data** — use `shallowRef`.
- **Mutating inside a `shallowRef`** and expecting updates — replace, or `triggerRef`.
- **Putting third-party class instances in reactive state** — `markRaw` them.
- **Comparing reactive items with `===`** against raw objects.
- **Computeds returning fresh objects** feeding expensive watchers — reuse `prev`.
- **Using `toRaw` to "make things faster" and then mutating the raw object** — the UI
  won't update, because writes to the raw object bypass `trigger`.

## Exercise

1. Implement the mini reactivity system above in a plain `.ts` file and run it with
   `npx tsx`. Add a scheduler that batches effects with `queueMicrotask` and show that three
   writes cause one run.
2. Render a table of 5,000 generated rows twice: once from `ref(rows)`, once from
   `shallowRef(rows)`. Use the Vue devtools' Performance/Timeline tab (or
   `performance.now()` around an update) to compare the cost of replacing the data. Record
   what *you* observe on your machine — don't expect specific numbers.
3. Wrap a third-party object (for example `new Intl.NumberFormat()` or a chart instance) in
   `markRaw` and confirm with `isProxy` that it stays raw inside a store.
4. Write a `useThrottledRef` with `customRef` that updates at most once every N ms but
   always ends on the latest value.
