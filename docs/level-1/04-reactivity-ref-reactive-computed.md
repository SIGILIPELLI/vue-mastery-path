---
description: "Vue's core reactivity: ref versus reactive, .value and template unwrapping, why destructuring breaks reactivity, and cached computed properties."
---

# 04 · Reactivity: ref, reactive, computed

Every interactive Vue component is built on three functions: `ref()` for reactive values,
`reactive()` for reactive objects, and `computed()` for values derived from them. This
lesson explains each one, when to reach for which, and the rules that make them behave
predictably. All console output shown was captured from Vue 3.5.43 in a Vitest run.

## `ref()`

`ref()` wraps any value — number, string, object, array, `null` — in an object with a
single reactive property, `.value`:

```ts
import { ref } from 'vue'

const count = ref(0)          // Ref<number>
count.value++                 // write
console.log(count.value)      // read: 1

const user = ref<{ name: string } | null>(null)
user.value = { name: 'Ada' }
```

In the template, top-level refs are **unwrapped** automatically, so you write `{{ count }}`
and `@click="count++"`, not `.value`.

If you pass an object to `ref()`, its `.value` is made deeply reactive:

```ts
const cart = ref({ items: [1, 2] })
cart.value.items.push(3)      // tracked — components reading cart.value.items update
```

We verified that `isReactive(cart.value)` is `true`: `ref` hands objects to `reactive()`
internally.

## `reactive()`

`reactive()` takes an object and returns a reactive **proxy** of it. No `.value` needed:

```ts
import { reactive } from 'vue'

const form = reactive({
  email: '',
  password: '',
  remember: false,
})

form.email = 'ada@example.com'   // tracked
```

Nested objects are reactive too — `isReactive(state.nested)` came back `true` in our run.

The proxy is **not** the original object:

```text
proxy===raw false same proxy twice true toRaw true
```

That line came from:

```ts
const raw = { n: 1 }
const r = reactive(raw)
console.log('proxy===raw', r === raw,
  'same proxy twice', reactive(raw) === r,
  'toRaw', toRaw(r) === raw)
```

Only changes made **through the proxy** are tracked. If some other code keeps a reference
to `raw` and mutates it, Vue never finds out.

### The limits of `reactive()`

1. **It only works on objects** (including arrays, `Map`, `Set`). `reactive(0)` gives a
   warning and a plain number.
2. **You can't replace the whole thing.** `let state = reactive({...}); state = reactive({...})`
   disconnects everything that was reading the first proxy.
3. **Destructuring breaks the connection.** This is the big one.

```ts
const state = reactive({ count: 0 })
let { count } = state
count++
console.log(state.count)          // 0  — 'count' was a plain number copy
```

Our run printed `destructured count++ -> state.count 0`. When you destructure, you read
the *current value* out of the proxy into a local variable. To destructure and keep
reactivity, convert to refs first:

```ts
import { toRefs } from 'vue'

const { count } = toRefs(state)   // count is a Ref linked to state.count
count.value++
console.log(state.count)          // 1
```

## Which one should I use?

The official recommendation — and this course's convention — is **`ref()` as the default**.
Reasons:

- it works for every kind of value, including primitives and `null`;
- you can replace the whole value (`items.value = await fetchItems()`);
- the `.value` makes reactive reads visible when you scan the code;
- refs survive being passed into functions and returned from composables (Level 2).

`reactive()` is pleasant for a cohesive group of fields that you always use together
through the object, such as form state. Just never destructure it.

### Ref unwrapping rules

Refs are unwrapped in a few places and not others. From our run:

```ts
const inner = ref(5)
const box = reactive({ inner })
box.inner                // 5 — unwrapped when accessed as a property of a reactive object

const arr = reactive([ref(1)])
isRef(arr[0])            // true — NOT unwrapped inside reactive arrays (or Map/Set)
```

In templates, unwrapping applies only to **top-level** bindings. If you have
`const obj = { id: ref(1) }` (a plain object holding a ref), then `{{ obj.id + 1 }}` gives
`[object Object]1`. Destructure `id` to the top level or use `obj.id.value`.

## `computed()`

`computed()` derives a value from other reactive state:

```ts
import { ref, computed } from 'vue'

const items = ref([
  { name: 'Coffee', price: 4, qty: 2 },
  { name: 'Bagel', price: 3, qty: 1 },
])
const total = computed(() =>
  items.value.reduce((sum, i) => sum + i.price * i.qty, 0),
)

total.value   // 11
```

You never list dependencies — whatever reactive values the getter **reads while running**
become its dependencies. Computed values are **lazy** and **cached**. We counted getter
runs:

```ts
const a = ref(2)
let runs = 0
const doubled = computed(() => { runs++; return a.value * 2 })
```

```text
after create 0
after 2 reads 1
after set 1
after read 2
same value set + read 2
```

- Creating a computed runs nothing.
- Two reads ran the getter once — the second read used the cache.
- Setting `a` did **not** run the getter; it only marked the cache as stale.
- The next read recomputed.
- Setting `a` to the value it already had (`3` → `3`) triggered nothing at all.

That makes `computed` the right tool for filtering, sorting and totals. A method called
from the template, by contrast, runs on every render of the component.

### Writable computed

Occasionally you want a derived value that can also be assigned. Provide `get` and `set`:

```ts
const first = ref('Ada')
const last = ref('Lovelace')
const fullName = computed({
  get: () => `${first.value} ${last.value}`,
  set: (value: string) => {
    const [f = '', l = ''] = value.split(' ')
    first.value = f
    last.value = l
  },
})

fullName.value = 'Grace Hopper'   // first = 'Grace', last = 'Hopper'
```

Our run confirmed `first.value, last.value` became `Grace Hopper`. Use this sparingly —
most of the time, a computed should be read-only.

## Worked example: a filtered, sorted list

```vue title="src/components/TaskList.vue"
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Task { id: number; title: string; done: boolean; priority: 1 | 2 | 3 }

const tasks = ref<Task[]>([
  { id: 1, title: 'Write tests', done: false, priority: 2 },
  { id: 2, title: 'Fix login bug', done: false, priority: 1 },
  { id: 3, title: 'Update README', done: true, priority: 3 },
])
const hideDone = ref(false)
const query = ref('')

const visible = computed(() =>
  tasks.value
    .filter((t) => !hideDone.value || !t.done)
    .filter((t) => t.title.toLowerCase().includes(query.value.toLowerCase()))
    .sort((a, b) => a.priority - b.priority), // safe: filter() already made a new array
)
const remaining = computed(() => tasks.value.filter((t) => !t.done).length)

let nextId = 4
function add(title: string) {
  tasks.value.push({ id: nextId++, title, done: false, priority: 2 })
}
</script>

<template>
  <input v-model="query" placeholder="Filter tasks" />
  <label><input type="checkbox" v-model="hideDone" /> Hide done</label>
  <p>{{ remaining }} remaining</p>
  <ul>
    <li v-for="t in visible" :key="t.id">
      <label><input type="checkbox" v-model="t.done" /> {{ t.title }}</label>
    </li>
  </ul>
  <button type="button" @click="add('New task')">Add</button>
</template>
```

Checking a box writes `t.done` on an object inside `tasks.value`. That object is reactive
(it came through `ref`), so both `visible` and `remaining` are invalidated and the list
updates. Note that `.sort()` is only safe here because `filter()` has already produced a
new array. **Never mutate reactive state inside a computed getter** — calling
`tasks.value.sort()` directly would reorder the source array itself, triggering every
effect that depends on it (including the computed that is currently running). The
non-mutating `toSorted()` also works at runtime in current browsers, but the generated
`tsconfig` doesn't include the ES2023 library types, so `vue-tsc` rejects it unless you
raise `lib`; we hit exactly that error when type-checking this example.

## How It Actually Works

`reactive()` returns a JavaScript `Proxy`. Its `get` trap calls `track(target, key)`,
which records "the currently running effect depends on `target.key`". Its `set` trap
calls `trigger(target, key)`, which finds every effect that depends on that key and
schedules it. A `ref` is a small object whose `value` getter and setter do the same
tracking and triggering on a single property — which is why refs can hold primitives
(you can't proxy a number) and why you need `.value` in script (the getter *is* the
interception point).

"The currently running effect" is the key idea. When a component renders, Vue runs its
render function inside a *reactive effect*; every `track` call during that run adds the
component to a dependency list. A `computed` is also an effect, but a lazy one: when a
dependency triggers, it just flips a `dirty` flag and notifies *its own* subscribers.
Recomputation happens on the next `.value` read. In Vue 3.5 each computed also remembers
a version number for every dependency, so it can tell cheaply whether anything it read
really changed — which is why setting `a` to the same value caused zero reruns above
(and `ref` writes compare with `Object.is` first, so that write didn't even trigger).

Destructuring breaks things because a plain variable has no getter. `let { count } = state`
executes one `get` on the proxy and copies the number out; subsequent reads of `count`
never hit the proxy, so nothing is tracked. `toRefs` creates refs whose getters read
`state.count` on every access, restoring the interception.

## Common mistakes

- **Destructuring `reactive()` or props-like objects** and expecting updates. Use
  `toRefs`, or just use `ref` everywhere.
- **Reassigning a `reactive` variable.** Mutate its properties or use `Object.assign`,
  or switch to `ref` and assign `.value`.
- **Side effects in `computed`** — fetching data, mutating state, writing to storage.
  Computed getters must be pure; side effects belong in watchers (Lesson 05).
- **Mutating the result of a computed.** `visible.value.push(x)` modifies a derived array
  that will be thrown away on the next recompute. Change the source instead.
- **Comparing proxies to raw objects with `===`.** After `list.push(item)` on a reactive
  array, `list[list.length - 1] === item` is `false` — the array hands back a proxy of
  `item`. (Vue does patch `includes`, `indexOf` and `lastIndexOf` to also match the raw
  object; we checked, and `list.includes(item)` returned `true`.) Comparing by id avoids
  the question entirely.

## Exercise

Build `TaskList.vue` above, then extend it:

1. Add a `computed` called `progress` that returns the percentage of tasks done, rounded
   to a whole number, and show it in a `<progress>` element.
2. Add a "Sort by" `<select>` (priority / title) stored in a `ref`, and make `visible`
   respect it.
3. Add a `console.log('recomputing')` inside `visible`'s getter. Type in the filter box
   and toggle "Hide done" — then click a button that changes something unrelated (add a
   `const clicks = ref(0)` and a button that increments it and displays it). Confirm the
   log does **not** fire for the unrelated change. Remove the log when done.
4. Refactor `tasks` from `ref<Task[]>` to `reactive<Task[]>` and see which lines must
   change. Which version do you prefer, and why?
