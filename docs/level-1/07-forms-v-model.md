---
description: "Two-way binding in Vue: v-model on inputs, checkboxes, radios and selects, the .lazy/.number/.trim modifiers, and defineModel for custom components."
---

# 07 · Forms & v-model

Forms are where UI state gets written by the user instead of by your code. `v-model` is
Vue's shorthand for "keep this input and this piece of state in sync in both directions".
This lesson covers `v-model` on every kind of native input, its modifiers, and how to
support `v-model` on your own components with `defineModel()`.

## `v-model` on a text input

```vue
<script setup lang="ts">
import { ref } from 'vue'
const name = ref('')
</script>

<template>
  <label for="name">Name</label>
  <input id="name" v-model="name" />
  <p>Hello, {{ name || 'stranger' }}</p>
</template>
```

`v-model` on an `<input>` is equivalent to:

```vue-html
<input :value="name" @input="name = ($event.target as HTMLInputElement).value" />
```

So typing updates `name`, and assigning `name.value = 'Ada'` in code updates the input.

`v-model` ignores the `value`, `checked` and `selected` attributes you write in the
template — the bound state is the single source of truth. Set initial values in script.

## Every input type

```vue title="src/components/SignupForm.vue"
<script setup lang="ts">
import { reactive } from 'vue'

const form = reactive({
  email: '',
  bio: '',
  age: null as number | null,
  plan: 'free' as 'free' | 'pro',
  interests: [] as string[],
  newsletter: false,
  country: '',
  languages: [] as string[],
})
</script>

<template>
  <form @submit.prevent="console.log({ ...form })">
    <label>Email <input type="email" v-model.trim="form.email" required /></label>

    <label>Bio <textarea v-model="form.bio" rows="3" /></label>

    <label>Age <input type="number" v-model.number="form.age" min="13" /></label>

    <fieldset>
      <legend>Plan</legend>
      <label><input type="radio" value="free" v-model="form.plan" /> Free</label>
      <label><input type="radio" value="pro" v-model="form.plan" /> Pro</label>
    </fieldset>

    <fieldset>
      <legend>Interests</legend>
      <label><input type="checkbox" value="frontend" v-model="form.interests" /> Frontend</label>
      <label><input type="checkbox" value="backend" v-model="form.interests" /> Backend</label>
      <label><input type="checkbox" value="design" v-model="form.interests" /> Design</label>
    </fieldset>

    <label><input type="checkbox" v-model="form.newsletter" /> Send me the newsletter</label>

    <label>
      Country
      <select v-model="form.country">
        <option disabled value="">Please select</option>
        <option value="IN">India</option>
        <option value="DE">Germany</option>
        <option value="BR">Brazil</option>
      </select>
    </label>

    <label>
      Languages
      <select v-model="form.languages" multiple>
        <option>English</option>
        <option>Hindi</option>
        <option>Telugu</option>
      </select>
    </label>

    <button type="submit">Sign up</button>
    <pre>{{ form }}</pre>
  </form>
</template>
```

What each binds to:

| Element | Bound state | Event listened to |
|---------|-------------|-------------------|
| text input, textarea | string | `input` |
| single checkbox | boolean (or custom `true-value`/`false-value`) | `change` |
| checkboxes sharing an array | array of the checked `value`s | `change` |
| radio group | the checked radio's `value` | `change` |
| `<select>` | the selected option's `value` | `change` |
| `<select multiple>` | array of selected values | `change` |

Two details from that table:

- An `<option>` without a `value` attribute uses its text (`'English'`).
- The disabled empty `<option value="">` is the standard way to show a placeholder. If the
  bound value doesn't match any option, some browsers (notably iOS Safari) show the first
  option as selected without firing `change`, so the user can never pick it. The empty
  disabled option avoids that.

### Binding non-string values

`value` attributes are strings, but `:value` can bind anything — numbers, objects:

```vue-html
<select v-model="selectedUser">
  <option v-for="u in users" :key="u.id" :value="u">{{ u.name }}</option>
</select>
```

Now `selectedUser` holds the actual user object. Vue compares options by identity (or
with loose equality for primitives) to decide which one is selected.

## Modifiers

- **`.lazy`** — sync on `change` (when the field loses focus or Enter is pressed) instead of
  every keystroke. Useful when every update triggers something expensive.
- **`.number`** — convert the input to a number with `parseFloat`. If it can't be parsed,
  the original string is kept. On `<input type="number">`, Vue applies this automatically.
  Note that clearing the field gives `''`, not `null` — handle that case.
- **`.trim`** — strip surrounding whitespace.

Modifiers combine: `v-model.lazy.trim="title"`.

## `v-model` on your own components

Use `v-model` on a component and Vue passes a `modelValue` prop and listens for an
`update:modelValue` event. Since Vue 3.4, `defineModel()` generates both for you and gives
you a ref you can read and write:

```vue title="src/components/QuantityInput.vue"
<script setup lang="ts">
const quantity = defineModel<number>({ required: true })
const { min = 1, max = 99 } = defineProps<{ min?: number; max?: number }>()

function step(delta: number) {
  quantity.value = Math.min(max, Math.max(min, quantity.value + delta))
}
</script>

<template>
  <div class="qty">
    <button type="button" :disabled="quantity <= min" @click="step(-1)" aria-label="Decrease">−</button>
    <input type="number" v-model="quantity" :min="min" :max="max" aria-label="Quantity" />
    <button type="button" :disabled="quantity >= max" @click="step(1)" aria-label="Increase">+</button>
  </div>
</template>
```

```vue-html
<QuantityInput v-model="cart.qty" :max="10" />
```

Writing `quantity.value` emits `update:modelValue`; the parent's `v-model` assigns the new
value to `cart.qty`; the new value comes back down as the prop. The child never owns the
state — it only asks the parent to change it. Notice the inner `<input v-model="quantity">`:
you can pass a model ref straight through to a native input.

### Multiple and named models

A component can support several `v-model`s by naming them:

```vue title="src/components/NameFields.vue"
<script setup lang="ts">
const first = defineModel<string>('first', { default: '' })
const last = defineModel<string>('last', { default: '' })
</script>

<template>
  <input v-model="first" placeholder="First name" />
  <input v-model="last" placeholder="Last name" />
</template>
```

```vue-html
<NameFields v-model:first="user.first" v-model:last="user.last" />
```

### Custom modifiers

Destructure the second element returned by `defineModel` to read modifiers, and use `set`
to transform the value on the way out:

```ts
const [title, modifiers] = defineModel<string, 'capitalize'>('title', {
  set(value) {
    return modifiers.capitalize ? value.charAt(0).toUpperCase() + value.slice(1) : value
  },
})
```

```vue-html
<TitleInput v-model:title.capitalize="post.title" />
```

In our test, a component mounted with `v-model:title.upper` received the modifiers object
`{"upper":true}` — modifiers arrive as a `titleModifiers` prop (`modelModifiers` for the
default model).

## Worked example: a controlled search box

A common pattern: the parent owns the query, the child adds a clear button and keyboard
shortcuts.

```vue title="src/components/SearchBox.vue"
<script setup lang="ts">
import { useTemplateRef } from 'vue'

const query = defineModel<string>({ default: '' })
const input = useTemplateRef<HTMLInputElement>('input')

function clear() {
  query.value = ''
  input.value?.focus()
}
</script>

<template>
  <div role="search">
    <input
      ref="input"
      v-model.trim="query"
      type="search"
      placeholder="Search…"
      aria-label="Search"
      @keydown.esc="clear"
    />
    <button v-if="query" type="button" @click="clear" aria-label="Clear search">×</button>
  </div>
</template>
```

## How It Actually Works

`v-model` is a compile-time transform, and it compiles differently depending on the
element:

- On native inputs, the compiler emits a runtime directive — `vModelText`,
  `vModelCheckbox`, `vModelRadio`, `vModelSelect` or `vModelDynamic` (when `type` is bound)
  — plus an `onUpdate:modelValue` prop that assigns to your state. The directive attaches
  the right DOM listener (`input`, or `change` for `.lazy`, checkboxes, radios and selects),
  applies `.trim`/`.number` casting, and on each render writes the state back into
  `el.value`/`el.checked` only if it differs — so the cursor doesn't jump while typing.
  The text directive also ignores `input` events fired during IME composition (Chinese,
  Japanese, Korean input) until composition ends.
- On components, `v-model="x"` compiles to two props:
  `modelValue: x` and `"onUpdate:modelValue": $event => (x = $event)`. That's all.
  `defineModel()` in the child compiles to a `modelValue` prop, an `update:modelValue` emit,
  and a call to the runtime helper `useModel()`. That helper returns a custom ref whose
  getter returns the prop and whose setter emits the event. It also keeps a local value, so
  the model still works if the parent **didn't** bind `v-model` at all — the component
  behaves as uncontrolled and holds the value itself.

## Common mistakes

- **Mutating a prop instead of using a model.** If a child writes `props.modelValue`, it's
  ignored with a readonly warning. Use `defineModel`.
- **Setting `value="..."` or `checked` in the template** alongside `v-model` — it's ignored.
- **Assuming `.number` always yields a number.** Empty and unparsable inputs stay strings.
  Validate or coerce before using the value (Level 2 · 08).
- **Binding `v-model` to a destructured prop or computed without a setter** — you can't
  assign to it. Bind to state you own.
- **Checkbox groups bound to a non-array** — a single boolean flips instead of collecting
  values.
- **Using `v-model` on a `<select>` with no matching option** — see the iOS note above.

## Exercise

1. Build `SignupForm.vue` and watch the `<pre>` update as you interact. Clear the Age field
   and note the type of `form.age` in the output.
2. Add a "Confirm email" field and a `computed` `emailsMatch`. Disable the submit button
   until the emails match and at least one interest is chosen.
3. Build `QuantityInput.vue` and use it in `App.vue` with `v-model`. Then render it
   **without** `v-model`. Because the model is declared `required: true`, Vue warns
   `Missing required prop: "modelValue"` and the input starts empty. Change the declaration
   to `defineModel<number>({ default: 1 })` and try again: the buttons now work with no
   parent state at all (in our test, two clicks took the input from 1 to 3 while still
   emitting `update:modelValue` each time). Explain why, using the How It Actually Works
   section.
4. Add a custom `.round` modifier to `QuantityInput` that rounds typed decimals to the
   nearest whole number before they reach the parent.
