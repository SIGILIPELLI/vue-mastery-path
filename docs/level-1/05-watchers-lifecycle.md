---
description: "Run side effects in Vue with watch and watchEffect, understand batching and flush timing, clean up with onWatcherCleanup, and use lifecycle hooks."
---

# 05 · Watchers & Lifecycle Hooks

`computed` answers "what is this value, given the state?" Watchers answer a different
question: "when this state changes, what should *happen*?" — save to `localStorage`, fetch
new data, focus an input, sync a third-party chart. Lifecycle hooks answer "what should
happen when this component appears or disappears?" This lesson covers both, with output
captured from Vue 3.5.43.

## `watch()`

`watch(source, callback, options?)` runs `callback(newValue, oldValue)` when `source`
changes:

```ts
import { ref, watch } from 'vue'

const query = ref('')

watch(query, (newQuery, oldQuery) => {
  console.log(`search changed from "${oldQuery}" to "${newQuery}"`)
})
```

The **source** can be:

- a ref (`query`);
- a getter function (`() => user.value.id`, `() => props.page`);
- a reactive object (watched deeply);
- an array of any of those (`[page, pageSize]` — the callback receives arrays).

### Watching a property of a reactive object needs a getter

```ts
const state = reactive({ q: '', filters: { tag: 'a' } })

watch(state.q, () => {})          // ✗ passes the string '' — not reactive
watch(() => state.q, () => {})    // ✓ getter re-reads the property
```

The first form printed this warning in our run:

```text
[Vue warn]: Invalid watch source:  vue A watch source can only be a getter/effect function,
a ref, a reactive object, or an array of these types.
```

(The empty-looking value before "vue" is the string it received.)

### Deep or shallow?

A getter that returns an object is watched **shallowly**: the callback fires only if the
getter returns a *different object*. Mutating a nested property doesn't count. We set up
four watchers on the object above, then ran `state.q = 'vue'; state.filters.tag = 'b'`:

```ts
watch(() => state.q, (v) => hits.push('q:' + v))
watch(() => state.filters, () => hits.push('filters shallow'))
watch(() => state.filters, () => hits.push('filters deep'), { deep: true })
watch(state, () => hits.push('whole reactive (implicitly deep)'))
```

```text
q:vue | whole reactive (implicitly deep) | filters deep
```

The shallow `filters` watcher never fired: the getter returned the same object. Passing
`deep: true` (or a number, such as `deep: 1`, to limit the depth) makes it traverse
nested properties. Passing a reactive object directly is implicitly deep. Deep watching
walks the entire object on every check, so prefer specific getters on large data.

### Batching: many changes, one callback

Watchers don't run on every assignment. By default they are **batched** and run before
the next component update:

```ts
const n = ref(0)
watch(n, (v, old) => log.push(`watch ${old}->${v}`))
n.value++; n.value++; n.value++
log.push('sync end')
await nextTick()
```

```text
sync end | watch 0->3
```

Three increments produced **one** callback, after the synchronous code finished, with the
old value from before the first change. If you truly need a callback per assignment,
`{ flush: 'sync' }` does that (our run printed `sync 4 | sync 5` for two increments) — but
sync watchers defeat batching and are easy to make slow. Use them rarely.

The `flush` option controls timing:

| `flush` | Runs | Typical use |
|---------|------|-------------|
| `'pre'` (default) | before the component re-renders | update other state |
| `'post'` | after the DOM has updated | read the updated DOM, measure elements |
| `'sync'` | immediately on every write | rarely: debugging, integration code |

`watchPostEffect()` and `watchSyncEffect()` are shorthands for `watchEffect` with those
flush modes.

### `immediate`, `once` and cleanup

```ts
import { ref, watch, onWatcherCleanup } from 'vue'

const userId = ref(1)

watch(userId, (id) => {
  const controller = new AbortController()
  fetch(`/api/users/${id}`, { signal: controller.signal })
    .then((r) => r.json())
    .then((data) => { /* ... */ })
    .catch(() => {})
  onWatcherCleanup(() => controller.abort())
}, { immediate: true })
```

- `immediate: true` runs the callback once right away (with `oldValue` `undefined`).
- `onWatcherCleanup()` (Vue 3.5+) registers a function that runs **before the next
  callback** and when the watcher stops. Here, if the user id changes again before the
  first request finishes, the stale request is aborted — which prevents an old response
  arriving late and overwriting the new one.
- `once: true` (Vue 3.4+) fires at most once and then stops.

Our trace, changing an id from 1 → 2 → 3 with both an immediate watcher and a `once`
watcher:

```text
start 1 | cleanup 1 | start 2 | once 2 | cleanup 2 | start 3
```

Before Vue 3.5, cleanup was registered through the callback's third parameter,
`(value, old, onCleanup) => { onCleanup(() => ...) }`. That form still works. One
restriction on `onWatcherCleanup`: call it synchronously inside the callback, not after an
`await`, because it relies on knowing which watcher is currently running.

## `watchEffect()`

`watchEffect(fn)` runs `fn` immediately and re-runs it whenever anything it **read** changes
— the same automatic tracking as `computed`, but for side effects:

```ts
const a = ref(1), b = ref(10), useA = ref(true)

watchEffect(() => {
  log.push(String(useA.value ? a.value : b.value))
})
```

We changed `b` (not being read), then switched the flag, then changed `b` again, then
stopped the watcher and changed `b` once more:

```text
1 | (b changed while unused) | 11 | 12
```

Dependencies are re-collected on every run. While the branch read `a`, changes to `b`
were ignored. After the switch, `b` was tracked. After `stop()`, nothing ran.

`watch` vs `watchEffect`: use `watch` when you want explicit sources, access to the old
value, or lazy behaviour (not running immediately). Use `watchEffect` when the effect
naturally reads everything it depends on.

### Stopping watchers

Watchers created synchronously in `<script setup>` are stopped automatically when the
component unmounts. Both functions return a stop handle for when you need to stop one
earlier:

```ts
const stop = watch(source, cb)
stop()
```

A watcher created **asynchronously** (for example after an `await` or inside a
`setTimeout`) isn't bound to the component and will leak unless you stop it yourself.

## Lifecycle hooks

Each component goes through creation, mounting, updates and unmounting. You register
code for those moments with `on*` functions called during `setup`:

| Hook | Runs when | Typical use |
|------|-----------|-------------|
| (setup body) | component instance is being created | initialise state |
| `onBeforeMount` | just before first render is inserted | rarely needed |
| `onMounted` | component's DOM is in the document | DOM access, third-party widgets, start timers |
| `onBeforeUpdate` | before a re-render patches the DOM | read DOM state before change |
| `onUpdated` | after a re-render patched the DOM | avoid — prefer `watch` with `flush: 'post'` |
| `onBeforeUnmount` | component is about to be removed | — |
| `onUnmounted` | component removed | tear down listeners, timers, widgets |

There are also `onActivated`/`onDeactivated` (for `<KeepAlive>`, Level 3 · 04),
`onErrorCaptured` (Level 3 · 09) and `onServerPrefetch` (Level 4 · 01).

### The order, observed

We mounted a parent that renders a child with a `count` prop, clicked a button in the
child that makes the parent increase `count`, then hid the child with `v-if`:

```text
parent setup
parent beforeMount
child sees count=0
child beforeMount
child mounted
parent mounted
--- click
child sees count=2
child updated
parent updated
--- hide
child beforeUnmount
child unmounted
parent updated
```

Children finish mounting **before** their parent's `onMounted` runs — so in a parent's
`onMounted`, the whole subtree's DOM exists. Updates also complete bottom-up.

## Worked example: autosave with debounce

```vue title="src/components/NoteEditor.vue"
<script setup lang="ts">
import { ref, watch, onWatcherCleanup, onMounted, onUnmounted, useTemplateRef } from 'vue'

const STORAGE_KEY = 'note-draft'
const text = ref(localStorage.getItem(STORAGE_KEY) ?? '')
const status = ref<'saved' | 'saving'>('saved')
const textarea = useTemplateRef<HTMLTextAreaElement>('editor')

watch(text, (value) => {
  status.value = 'saving'
  const timer = setTimeout(() => {
    localStorage.setItem(STORAGE_KEY, value)
    status.value = 'saved'
  }, 500)
  onWatcherCleanup(() => clearTimeout(timer))   // restart the countdown on each keystroke
})

function onKeydown(e: KeyboardEvent) {
  if (e.key === 'Escape') textarea.value?.blur()
}

onMounted(() => {
  textarea.value?.focus()
  window.addEventListener('keydown', onKeydown)
})
onUnmounted(() => window.removeEventListener('keydown', onKeydown))
</script>

<template>
  <textarea ref="editor" v-model="text" rows="8" cols="60" />
  <p aria-live="polite">{{ status === 'saved' ? 'All changes saved' : 'Saving…' }}</p>
</template>
```

Each keystroke re-runs the watcher; the
cleanup cancels the previous timer, so the save happens 500 ms after the user *stops*
typing. `useTemplateRef('editor')` (Vue 3.5+) returns a ref that points to the element
with `ref="editor"` once it is mounted — before that it is `null`, which is why the
element is only touched in `onMounted`.

## How It Actually Works

`watch` and `watchEffect` both create a `ReactiveEffect` — the same machinery that powers
component rendering. For `watch`, the effect's function is the **getter** (a ref source
becomes `() => source.value`; a reactive object becomes a function that traverses it).
When a dependency triggers, the effect's *scheduler* runs instead of the effect itself.
With the default `flush: 'pre'`, the scheduler pushes a job into Vue's scheduler queue;
the queue is flushed in a microtask (that's what `nextTick()` waits for). When the job
runs, it calls the getter, compares the new value with the old one (`Object.is`, or
"always changed" for deep watchers), and calls your callback only if it differs.

That design explains every behaviour above: three increments queue the same job three
times, but the queue de-duplicates jobs, so it runs once and sees only the final value;
`'pre'` jobs are sorted to run before component render jobs; `'post'` jobs run after the
DOM patch; `'sync'` bypasses the queue entirely.

Lifecycle hooks work because `setup` runs while Vue has set a module-level "current
instance" variable. `onMounted(fn)` simply pushes `fn` into that instance's list of
mounted hooks. That is also why hooks must be called synchronously during setup — after an
`await`, the current instance has already been reset, and Vue warns that there is no
active component instance to attach to.

## Common mistakes

- **Using a watcher to derive state.** `watch(a, v => b.value = v * 2)` should be
  `const b = computed(() => a.value * 2)`. Watchers are for side effects.
- **Passing `state.prop` instead of `() => state.prop`.**
- **Expecting a shallow getter to see nested mutations.** Watch the specific nested
  property, or use `deep`.
- **Forgetting cleanup** for timers, listeners and in-flight requests — they outlive the
  component and cause "setState on unmounted" style bugs and memory leaks.
- **Touching the DOM in `setup`** — it doesn't exist yet. Use `onMounted`, or a
  `flush: 'post'` watcher for DOM work after updates.
- **Registering hooks after `await`.** Register them before any `await` in setup.

## Exercise

Build `NoteEditor.vue` and render it in `App.vue`.

1. Add a character counter as a `computed`, and a second `watch` that logs a warning to the
   console once the note exceeds 280 characters — but only once per crossing (use the old
   value to detect the crossing).
2. Add a "Show editor" checkbox in `App.vue` that toggles `<NoteEditor v-if="...">`.
   Add `console.log`s to `onMounted` and `onUnmounted` and confirm the window listener is
   added and removed each time.
3. Remove the `onWatcherCleanup` line and type quickly. Using the log or the
   Application → Local Storage panel, describe what changed about how often the save
   happens.
4. Replace the `watch` with a `watchEffect`. What must the effect read for it to track
   `text`? Which version is clearer?
