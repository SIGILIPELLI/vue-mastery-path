---
description: "Pass data into Vue components with typed defineProps, send events out with defineEmits, use reactive props destructure and understand fallthrough attributes."
---

# 06 · Components: Props & Emits

Components become useful when you can reuse them with different data. Data flows **down**
through **props**; notifications flow **up** through **emitted events**. This one-way flow
is the rule that keeps a Vue app understandable: a child never reaches into its parent's
state, and a parent never pokes at a child's internals.

## Using a component

In `<script setup>`, importing a component is all it takes to use it:

```vue title="src/App.vue"
<script setup lang="ts">
import { ref } from 'vue'
import StarRating from './components/StarRating.vue'

const score = ref(3)
</script>

<template>
  <StarRating :value="score" :max="5" @change="(n) => (score = n)" />
  <p>You rated {{ score }} / 5</p>
</template>
```

Use PascalCase tags (`<StarRating>`) in SFC templates; it makes components easy to tell
apart from native elements. Vue also accepts `<star-rating>`.

## Declaring props with `defineProps`

`defineProps` is a **compiler macro**: you don't import it, and it only works at the top
level of `<script setup>`. With TypeScript, you declare props as a type:

```vue title="src/components/StarRating.vue"
<script setup lang="ts">
const { value, max = 5, readonly = false } = defineProps<{
  value: number
  max?: number
  readonly?: boolean
}>()

const emit = defineEmits<{
  change: [value: number]
}>()

function select(n: number) {
  if (!readonly) emit('change', n)
}
</script>

<template>
  <div class="rating" role="radiogroup" aria-label="Rating">
    <button
      v-for="n in max"
      :key="n"
      type="button"
      role="radio"
      :aria-checked="n === value"
      :aria-label="`${n} of ${max}`"
      :disabled="readonly"
      @click="select(n)"
    >
      {{ n <= value ? '★' : '☆' }}
    </button>
  </div>
</template>
```

- `value: number` is **required**; `max?: number` is optional.
- Default values are given with ordinary destructuring defaults: `max = 5`.
- Boolean props get special casting: `<StarRating readonly />` means `readonly: true`,
  and an absent boolean prop is `false`, not `undefined`.
- In the parent template, use kebab-case or camelCase interchangeably:
  `:max-value` and `:maxValue` bind the same prop.

### Reactive props destructure (Vue 3.5+)

Destructuring `defineProps()` looks like it would break reactivity (Lesson 04), but the
compiler handles it specially: every use of `value` in the script is rewritten to
`props.value`. We verified it — a child did `watchEffect(() => log(count))` on a
destructured prop, and when the parent changed the value the effect re-ran:

```text
child sees count=0
...
child sees count=2
```

There's one exception. Passing a destructured prop **into a function** passes its current
value, not something reactive:

```ts
watch(value, ...)          // ✗ compile error: pass a getter
watch(() => value, ...)    // ✓
useSomething(() => value)  // ✓ composables should accept getters (Level 2 · 01)
```

Before Vue 3.5 (or if you prefer), keep the props object and use `withDefaults`:

```ts
const props = withDefaults(defineProps<{ value: number; max?: number }>(), { max: 5 })
props.max
```

### Props are read-only

A child must not assign to a prop. We tried it:

```text
[Vue warn] Set operation on key "value" failed: target is readonly. Proxy({ value: 2, max: undefined })
```

The assignment is ignored. If the child needs to change a value, it has two options:

1. **Local copy** — for a prop that is just an initial value:
   `const count = ref(initialCount)`.
2. **Emit an event** so the owner updates its own state — the pattern above, and the basis
   of `v-model` (next lesson).

Objects and arrays are passed by reference, so a child *can* mutate `props.item.name`
without a warning. Don't: it makes the data flow invisible. Emit instead.

### Runtime validation

Type-only props are checked by TypeScript at build time. Vue also generates a runtime type
check in development. Passing a string where a number was expected produced:

```text
[Vue warn]: Invalid prop: type check failed for prop "value". Expected Number with value NaN, got String with value "x".
```

For extra validation you can use the object syntax, which supports `validator`:

```ts
defineProps({
  size: {
    type: String,
    default: 'md',
    validator: (v: string) => ['sm', 'md', 'lg'].includes(v),
  },
})
```

## Emitting events with `defineEmits`

The type-based form lists each event name with a tuple of its arguments:

```ts
const emit = defineEmits<{
  change: [value: number]
  submit: [payload: { title: string; tags: string[] }]
  close: []
}>()

emit('change', 4)
emit('close')
```

TypeScript then checks every `emit` call *and* every listener in parent templates.
In the parent, listen with `@change="handler"` or `@change="(n) => ..."`. In our test,
clicking the fourth star recorded `{"change":[[4]]}` as the emitted events.

Component events **do not bubble**. If a grandparent needs to know, the parent must
re-emit, or you should reach for `provide/inject` or a store (Level 2).

In templates you can also emit without the script: `@click="$emit('close')"`.

## Fallthrough attributes

Attributes and listeners that a parent passes but the child doesn't declare as props or
emits "fall through" to the child's **root element**:

```vue title="src/components/BaseButton.vue"
<script setup lang="ts">
defineProps<{ variant?: 'primary' | 'ghost' }>()
</script>

<template>
  <button class="btn" :class="variant ?? 'primary'"><slot /></button>
</template>
```

```vue-html
<BaseButton variant="ghost" class="wide" id="save" @click="save">Save</BaseButton>
```

Rendered (from our run):

```html
<button class="btn ghost wide" id="save"></button>
```

`class` and `style` are **merged** with the root's own values; other attributes are added
(or override); the undeclared `click` listener is attached to the native `<button>` — our
handler ran when the button was clicked. This is why wrapper components "just work" with
`id`, `aria-*`, `data-*` and native events.

If a component has multiple root nodes, or you want the attributes on an inner element,
turn off automatic fallthrough and bind them yourself:

```vue
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
</script>

<template>
  <label class="field">
    <span><slot name="label" /></span>
    <input v-bind="$attrs" />
  </label>
</template>
```

Now `<TextField placeholder="Email" type="email" required />` puts those attributes on the
`<input>`, not the `<label>`. In script, `useAttrs()` gives you the same object.

## How It Actually Works

`defineProps`, `defineEmits` and `withDefaults` don't exist at runtime. The SFC compiler
reads the TypeScript type and generates a runtime `props` option — for the `StarRating`
example, roughly:

```js
props: {
  value: { type: Number, required: true },
  max: { type: Number, required: false, default: 5 },
  readonly: { type: Boolean, required: false, default: false },
},
emits: ['change'],
setup(__props, { emit: __emit }) { /* uses __props.value, __props.max ... */ }
```

This is why the type must be something the compiler can analyse — a literal type, an
interface or type alias in the same file, or one imported from a relative file (Vue 3.3+
resolves those). Very complex types (conditional types, types from some libraries) can't be
turned into runtime options, and the compiler tells you so.

When the parent re-renders, it creates a new vnode for the child with new props. The
renderer compares the incoming props with the old ones, writes changed values into the
child's `props` object — which is a `shallowReactive` proxy — and that write triggers the
child's effects that read those props. The child re-renders only if a prop it actually
uses changed. The same props object is exposed to the child as `readonly`, which is where
the warning above comes from.

Declared emits matter beyond types: an event name listed in `emits` is **removed** from
fallthrough attrs. Without the declaration, a parent's `@change` listener would also be
attached as a native `change` listener on the child's root element — and could fire twice.

## Common mistakes

- **Mutating props**, including nested mutation of object props. Emit an event.
- **Copying a prop into a ref and expecting it to follow the prop.** `ref(props.value)`
  takes a one-time snapshot. If you need a derived value, use `computed(() => ...)`.
- **Passing a destructured prop to `watch` or a composable directly.** Wrap it in a getter.
- **Not declaring emitted events**, which causes duplicate native listeners and loses type
  checking.
- **Static vs dynamic binding**: `max="5"` passes the *string* `"5"`; `:max="5"` passes the
  number.
- **Relying on fallthrough in multi-root components.** Vue warns because it doesn't know
  which root should receive the attributes.

## Exercise

1. Build `StarRating.vue` and use it twice in `App.vue`: one interactive, one `readonly`
   showing the average of an array of ratings (a `computed`).
2. Add a `size?: 'sm' | 'md' | 'lg'` prop that sets a class, with `'md'` as the default.
3. Emit a second event, `hover`, with the star number being hovered (or `null` on
   `mouseleave`), and use it in the parent to show a preview label such as "4 — Very good".
4. Build a `TextField.vue` wrapper with `inheritAttrs: false` as shown and use it with
   `type="email"`, `required` and an `@input` listener. Confirm in the Elements panel that
   everything landed on the `<input>`.
