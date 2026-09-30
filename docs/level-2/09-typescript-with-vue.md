---
description: "TypeScript in Vue SFCs: typing props, emits, models and slots, typed template refs, generic components, and catching template errors with vue-tsc."
---

# 09 · TypeScript with Vue

You've been writing `lang="ts"` since Level 1. This lesson makes the typing deliberate:
what each macro accepts, how types flow from a child into its parent's template, how to
write **generic components**, and what `vue-tsc` catches that `tsc` alone can't. All errors
shown are real `vue-tsc` output (vue-tsc 3.3.11, TypeScript 6.0).

If TypeScript itself is new, the
[TypeScript Mastery Path](https://sigilipelli.github.io/typescript-mastery-path/) covers
generics, unions and utility types in depth.

## Why `vue-tsc` and not just `tsc`

`tsc` doesn't understand `.vue` files. `vue-tsc` (part of the Vue language tools, the same
engine behind the "Vue (Official)" VS Code extension) converts each SFC — **including its
template** — into virtual TypeScript and type-checks it. Your template expressions, props
passed to child components and event handlers are all checked.

We wrote a deliberately broken component that uses a typed `UserBadge`:

```vue title="src/tsdemo/UserBadge.vue"
<script setup lang="ts">
export interface User { id: number; name: string; avatarUrl?: string }
const { user, size = 'md' } = defineProps<{ user: User; size?: 'sm' | 'md' | 'lg' }>()
const emit = defineEmits<{ select: [id: number] }>()
function focus() { /* ... */ }
defineExpose({ focus })
</script>
<template><button type="button" :class="size" @click="emit('select', user.id)">{{ user.name.toUpperCase() }}</button></template>
```

```vue title="src/tsdemo/Broken.vue"
<script setup lang="ts">
import { ref, useTemplateRef, onMounted } from 'vue'
import UserBadge from './UserBadge.vue'
const current = ref({ id: 1, name: 'Ada' })
const selected = ref<string | null>(null)
const badge = useTemplateRef('badge')
onMounted(() => { badge.value?.focus(); badge.value?.blur() })
function onSelect(id: string) { selected.value = id }
</script>
<template>
  <UserBadge ref="badge" :user="current" size="xl" @select="onSelect" />
  <UserBadge />
  <p>{{ current.email }}</p>
</template>
```

`npx vue-tsc --build` reported five errors (long type names trimmed):

```text
src/tsdemo/Broken.vue(7,54): error TS2339: Property 'blur' does not exist on type 'CreateComponentPublicInstanceWithMixins<...{ focus: () => void; }...>'.
src/tsdemo/Broken.vue(11,42): error TS2322: Type '"xl"' is not assignable to type '"sm" | "md" | "lg" | undefined'.
src/tsdemo/Broken.vue(11,53): error TS2322: Type '(id: string) => void' is not assignable to type '(id: number) => any'.
  Types of parameters 'id' and 'id' are incompatible.
    Type 'number' is not assignable to type 'string'.
src/tsdemo/Broken.vue(12,4): error TS2345: Argument of type '{}' is not assignable to parameter of type '{ readonly user: User; ... }'.
  Property 'user' is missing in type '{}' but required in type '{ readonly user: User; ... }'.
src/tsdemo/Broken.vue(13,17): error TS2339: Property 'email' does not exist on type '{ id: number; name: string; }'.
```

Every one of these is a runtime bug that the Vite dev server would happily build: an
invalid prop value, a handler with the wrong parameter type, a missing required prop, a
typo in a template expression and a call to a method the child never exposed.

Run `vue-tsc` in CI (`npm run type-check` in the scaffold) and let the editor extension show
the same errors as you type.

## Typing props

The type argument to `defineProps` can be an inline literal, an interface, or a type
imported from another file:

```ts
import type { User } from '@/types'

interface Props {
  user: User
  size?: 'sm' | 'md' | 'lg'
  onDismiss?: () => void       // function props are allowed
}

const { user, size = 'md' } = defineProps<Props>()
```

The compiler must turn the type into runtime props (Level 1 · 06), so it has to be
statically analysable. Imported types from relative paths and packages work (Vue 3.3+);
very dynamic types (conditional types that depend on generics) may not, and the compiler
tells you when it can't resolve something.

**Exporting types from an SFC**: `export interface User` inside `<script setup>` is
allowed, and other files can import it: `import type { User } from './UserBadge.vue'`.
For types used widely, prefer a `.ts` file.

## Typing emits, models and slots

```ts
const emit = defineEmits<{
  select: [id: number]
  'update:filters': [filters: Filters]
  close: []
}>()

const open = defineModel<boolean>('open', { default: false })   // Ref<boolean>
const value = defineModel<string>()                              // Ref<string | undefined>

defineSlots<{
  default(props: { item: Item; index: number }): any
  empty(): any
}>()
```

- The tuple syntax (`[id: number]`) labels parameters, which the editor shows in hints.
- `defineModel<string>()` without `required: true` or a `default` is typed
  `string | undefined` — because the parent might not bind it.
- `defineSlots` types the slot props the parent's template receives. The return type is
  ignored (use `any`).

## Typing refs and reactive state

Inference covers most cases:

```ts
const count = ref(0)                            // Ref<number>
const user = ref<User | null>(null)             // give it the full type when starting empty
const items = ref<Item[]>([])                   // otherwise [] infers never[]
const form = reactive({ email: '', age: 0 })    // { email: string; age: number }
const total = computed(() => items.value.length) // ComputedRef<number>
```

Don't annotate `reactive` with a generic that contains refs (`reactive<{ n: Ref<number> }>`);
reactive unwraps refs, so the declared and actual types disagree. Use an interface of
plain values.

### Template refs

Vue 3.5's `useTemplateRef('name')` is typed **automatically** by `vue-tsc` when the name is a
literal matching a `ref="name"` in the same template — which is how the error above knew
that `badge` was a `UserBadge` exposing only `focus`. For native elements:

```ts
const input = useTemplateRef<HTMLInputElement>('input')   // explicit type also works
onMounted(() => input.value?.focus())
```

For a component instance whose type you need elsewhere, use the helper types from
`vue-component-type-helpers` (`ComponentExposed<typeof UserBadge>`), or
`InstanceType<typeof UserBadge>` for non-generic components.

### Event handlers

```ts
function onInput(e: Event) {
  const value = (e.target as HTMLInputElement).value
}
```

DOM events are typed as `Event` by default; the target is cast because TypeScript can't
know which element fired it. For inline handlers in templates, `$event` is typed from the
element: on `@keydown`, `$event` is a `KeyboardEvent`.

## Generic components

The `generic` attribute on `<script setup>` declares type parameters for the whole
component, exactly like a generic function:

```vue title="src/tsdemo/SelectMenu.vue"
<script setup lang="ts" generic="T, K extends PropertyKey">
const { options, getKey, getLabel } = defineProps<{
  options: readonly T[]
  getKey: (option: T) => K
  getLabel: (option: T) => string
}>()
const selected = defineModel<T | null>({ default: null })

function onChange(e: Event) {
  const key = (e.target as HTMLSelectElement).value
  selected.value = options.find((o) => String(getKey(o)) === key) ?? null
}
</script>

<template>
  <select :value="selected === null ? '' : String(getKey(selected))" @change="onChange">
    <option value="" disabled>Choose…</option>
    <option v-for="o in options" :key="String(getKey(o))" :value="String(getKey(o))">
      {{ getLabel(o) }}
    </option>
  </select>
</template>
```

`T` is **inferred at each use site** from the props the parent passes:

```vue title="src/tsdemo/Use.vue"
<script setup lang="ts">
import { ref } from 'vue'
import SelectMenu from './SelectMenu.vue'
interface Country { code: string; name: string; population: number }
const countries: Country[] = [{ code: 'IN', name: 'India', population: 1 }, { code: 'BR', name: 'Brazil', population: 2 }]
const country = ref<Country | null>(null)
const wrong = ref<string | null>(null)
</script>
<template>
  <SelectMenu v-model="country" :options="countries" :get-key="(c) => c.code" :get-label="(c) => c.name" />
  <SelectMenu v-model="wrong" :options="countries" :get-key="(c) => c.code" :get-label="(c) => c.title" />
</template>
```

The first usage type-checks — `c` in both callbacks is inferred as `Country`. The second
produced:

```text
src/tsdemo/Use.vue(11,15): error TS2322: Type 'string | null' is not assignable to type 'Country | null | undefined'.
  Type 'string' is not assignable to type 'Country'.
src/tsdemo/Use.vue(11,98): error TS2339: Property 'title' does not exist on type 'Country'.
```

Once `T` was inferred as `Country` from `options`, a `v-model` bound to a string and a label
function reading a non-existent property were both rejected. That's the payoff of generic
components: a reusable table, list or select that is as type-safe as hand-written code for
each data type. (The `DataList` in Level 1 · 08 used the same mechanism for its scoped
slot.)

## Typing global things

Augment Vue's types when you add app-wide properties, custom route meta (Lesson 04) or
global components:

```ts title="src/types/vue.d.ts"
export {}

declare module 'vue' {
  interface ComponentCustomProperties {
    $formatCurrency: (amount: number) => string   // from app.config.globalProperties
  }
}
```

The `export {}` makes the file a module, so `declare module` *augments* Vue's types instead
of replacing them — a classic source of "all my Vue types disappeared" confusion.

## How It Actually Works

`vue-tsc` is a wrapper around the TypeScript compiler that plugs in a *language plugin*.
For each `.vue` file, the plugin generates a virtual `.ts` file:

- the `<script setup>` content, with the macros replaced by typed helper calls
  (`defineProps<Props>()` becomes a function whose return type is `Props` with defaults
  applied);
- a synthetic function containing the **template compiled to TypeScript**: each
  `{{ expr }}` becomes an expression statement, each component tag becomes a call to a
  helper with the passed props as an object literal, each `@event` handler is checked
  against the child's emit types, each `v-for` becomes a `for...of` loop so the loop
  variable is typed;
- a default export whose type describes the component's props, emits, slots and exposed
  members, so a parent's virtual code can check its usage.

TypeScript then type-checks those virtual files normally, and `vue-tsc` maps error
positions back to the original `.vue` lines — which is why the errors above point to
exact template columns. For generic components, the default export is a *generic function
type*, so the parent's usage becomes a generic call whose `T` TypeScript infers from the
arguments, just like calling `select<T>(options: T[])`.

None of this exists at runtime. Types are erased when Vite compiles the SFC; the only
runtime effect is the props/emits declarations the SFC compiler derives from your types.

## Common mistakes

- **Relying on Vite for type safety** — it strips types without checking them.
- **`ref([])` for a typed list** — infers `never[]`. Give it `ref<Item[]>([])`.
- **Declaring non-optional props that have defaults** — mark them `?` so the parent isn't
  forced to pass them.
- **`declare module 'vue'` in a file without imports/exports** — it replaces Vue's types
  instead of augmenting them.
- **Casting to silence template errors** (`as any`) — the error is almost always a real
  mismatch between parent and child.
- **Not restarting the editor's Vue language server** after changing `tsconfig` — stale
  errors are common. Use the "Restart Vue and TS servers" command.

## Exercise

1. Copy `Broken.vue` and `UserBadge.vue` into your project, run `npx vue-tsc --build`, and
   fix each error in the *right* place (sometimes that's the parent, sometimes the child).
2. Build `SelectMenu.vue` and use it for two different types (countries and users). Hover
   over the callbacks in your editor to see the inferred `T`.
3. Make `DataTable.vue` a generic component with a `columns` prop of type
   `{ key: keyof T; label: string }[]` and a scoped slot per column
   (`#cell-name="{ row }"`) — the hardest typing in this lesson; use `defineSlots` with a
   template-literal key type, or fall back to one generic `#cell` slot with `{ row, column }`.
4. Add `vue-tsc --build` to a pre-commit hook or CI step and confirm it fails the build on a
   template typo.
