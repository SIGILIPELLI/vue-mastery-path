---
description: "Style Vue components: class and style bindings, scoped CSS and :deep, CSS modules, v-bind() in CSS, and enter/leave animations with Transition."
---

# 09 · Styling & Transitions

Vue doesn't impose a styling approach — plain CSS, Sass, Tailwind and CSS-in-JS all work.
What it adds are three conveniences worth knowing well: **class and style bindings**,
**scoped styles** in single-file components, and the **`<Transition>`** component for
enter/leave animations.

## Binding `class`

`:class` accepts a string, an object or an array, and merges with a static `class`:

```vue-html
<!-- object: key is the class, value decides if it's applied -->
<li class="task" :class="{ done: task.done, overdue: isOverdue(task) }">

<!-- array: mix strings and objects -->
<button :class="['btn', `btn-${variant}`, { loading: isSaving }]">

<!-- computed: when the logic grows -->
<div :class="alertClasses">
```

```ts
const alertClasses = computed(() => ({
  alert: true,
  [`alert-${level.value}`]: true,   // alert-info, alert-error, ...
  dismissible: canDismiss.value,
}))
```

With `task.done = true`, the first line renders `class="task done"`. Falsy values are
dropped, so `:class="[isActive && 'active']"` is safe.

As you saw in Lesson 06, a `class` passed to a component merges onto the component's root
element.

## Binding `style`

`:style` takes an object of CSS properties (camelCase or quoted kebab-case) or an array
of objects:

```vue-html
<div class="bar" :style="{ width: `${percent}%`, backgroundColor: color }" />
<p :style="[baseStyles, { '--accent': accent }]">
```

Vue adds units only where you tell it — `width: 50` is not valid CSS; write `'50px'` or a
template string. Custom properties (`--accent`) work, which is often the cleanest way to
pass a dynamic value into your stylesheet. Vue also adds vendor prefixes automatically when
a property needs one in the current browser.

Prefer classes for anything that isn't truly dynamic; inline styles can't express hover
states, media queries or transitions nicely.

## Scoped styles

```vue title="src/components/AlertBox.vue"
<script setup lang="ts">
defineProps<{ level: 'info' | 'error' }>()
</script>

<template>
  <div class="alert" :class="`alert-${level}`" role="status">
    <slot />
  </div>
</template>

<style scoped>
.alert {
  padding: 0.75rem 1rem;
  border-left: 4px solid;
  border-radius: 4px;
}
.alert-info { border-color: #2563eb; background: #eff6ff; }
.alert-error { border-color: #dc2626; background: #fef2f2; }
</style>
```

With `scoped`, these rules affect only this component's own elements. Another component
can have its own `.alert` class without collisions.

### Reaching into children: `:deep()` and `:slotted()`

Scoped styles stop at child component boundaries — except for the child's root element,
which gets both the parent's and the child's scope attribute (so the parent can position
it). To style inside a child, use `:deep()`; to style slot content passed *in*, use
`:slotted()`:

```css
.card :deep(.title) { font-weight: 700; }   /* any .title inside .card, even in children */
:slotted(p) { margin: 0; }                   /* <p>s the parent put into our slots */
```

Use `:deep()` sparingly: it couples you to the child's internal markup.

### Global rules in an SFC

`<style>` without `scoped` is global. You can have both blocks in one file, or wrap a
single selector in `:global(...)` inside a scoped block.

## CSS modules

`<style module>` compiles class names to unique hashed names and exposes them as `$style`:

```vue
<template>
  <p :class="$style.warning">Low disk space</p>
</template>

<style module>
.warning { color: #b45309; font-weight: 600; }
</style>
```

In script, `useCssModule()` returns the same mapping. CSS modules are a good fit when you
need to pass class names to non-Vue code or want styles that *can't* leak by selector.

## `v-bind()` in CSS

You can reference component state directly in a `<style>` block:

```vue title="src/components/ProgressBar.vue"
<script setup lang="ts">
import { computed } from 'vue'

const { value, color = '#16a34a' } = defineProps<{ value: number; color?: string }>()
const width = computed(() => `${Math.min(100, Math.max(0, value))}%`)
</script>

<template>
  <div class="track" role="progressbar" :aria-valuenow="value" aria-valuemin="0" aria-valuemax="100">
    <div class="fill" />
  </div>
</template>

<style scoped>
.track { height: 0.5rem; background: #e5e7eb; border-radius: 999px; overflow: hidden; }
.fill {
  height: 100%;
  width: v-bind(width);
  background: v-bind(color);
  transition: width 300ms ease;
}
</style>
```

Because `width` is animated with a CSS transition, the bar glides between values.

## Transitions

`<Transition>` animates a single element or component as it enters or leaves via `v-if`,
`v-show`, a dynamic component or a changing `key`:

```vue title="src/components/ToastMessage.vue"
<script setup lang="ts">
defineProps<{ message: string | null }>()
</script>

<template>
  <Transition name="toast">
    <p v-if="message" class="toast" role="status">{{ message }}</p>
  </Transition>
</template>

<style scoped>
.toast { position: fixed; bottom: 1rem; right: 1rem; padding: 0.75rem 1rem; background: #111827; color: white; border-radius: 6px; }

.toast-enter-active,
.toast-leave-active { transition: opacity 200ms ease, transform 200ms ease; }

.toast-enter-from,
.toast-leave-to { opacity: 0; transform: translateY(0.5rem); }
</style>
```

Vue adds six classes over the life of the animation, prefixed with the `name`:

| Class | Applied |
|-------|---------|
| `toast-enter-from` | first frame of enter, removed one frame later |
| `toast-enter-active` | the whole enter phase — put `transition` here |
| `toast-enter-to` | from the second frame until the enter finishes |
| `toast-leave-from` | first frame of leave |
| `toast-leave-active` | the whole leave phase |
| `toast-leave-to` | from the second frame until the leave finishes |

Useful props: `mode="out-in"` (when swapping elements, finish the leave before starting the
enter), `appear` (animate on first render), and `:duration` if Vue can't infer when an
animation ends. For lists, `<TransitionGroup>` animates insertions, removals and moves
(Level 3 · 04).

Respect users who ask for less motion:

```css
@media (prefers-reduced-motion: reduce) {
  .toast-enter-active, .toast-leave-active { transition: none; }
}
```

## How It Actually Works

**Scoped CSS** is two coordinated rewrites. The SFC compiler assigns the component an id
(a short hash derived from the file — `@vitejs/plugin-vue` uses its path, and in
production builds its contents too) and adds a `data-v-<id>` attribute to every element in its
template. It then rewrites each selector in the `<style scoped>` block to require that
attribute on its last compound selector. Compiling a sample block with
`@vue/compiler-sfc` gave:

```css
.card[data-v-7ba5bd90] { color: var(--7ba5bd90-color);
}
.card[data-v-7ba5bd90] .title { font-weight: 700;
}
p[data-v-7ba5bd90-s] { margin: 0
}
.card[data-v-7ba5bd90]:hover { outline: 1px solid;
}
```

You can see each feature here: `:deep(.title)` became a descendant selector with the
attribute applied to `.card` only, so it matches any `.title` below it; `:slotted(p)` uses a
separate `-s` attribute that Vue adds to slot content; and `v-bind(color)` became a CSS
custom property `var(--7ba5bd90-color)`. Scoping is only an attribute selector — it
raises specificity slightly and does nothing to stop inherited properties like `color` or
`font-family` from flowing into children.

**`v-bind()` in CSS** works through the custom property: the compiler injects a
`useCssVars()` call into the component's setup that sets `--7ba5bd90-color` as an inline
style on the component's root element(s), inside a watcher. When `color` changes, only that
custom property is updated; the stylesheet itself is static.

**`<Transition>`** is a component that wraps its child's vnode with enter/leave hooks. On
insert it adds `*-enter-from` and `*-enter-active`, waits one animation frame, swaps
`*-enter-from` for `*-enter-to`, then listens for `transitionend`/`animationend` (or reads
the computed `transition-duration` to know how long to wait) before removing the classes.
On leave, it delays the actual DOM removal until the leave animation has finished — that
delay is the one thing you couldn't do with CSS alone, because a `v-if` would otherwise
remove the element instantly.

## Common mistakes

- **Expecting scoped styles to reach child components.** They reach only the child's root
  element. Use `:deep()` or give the child a prop/class API.
- **Styling a component's root from both sides** and then being surprised by which wins —
  both scope attributes apply, so specificity decides.
- **Wrapping more than one element in `<Transition>`.** It animates a single child. Use
  `<TransitionGroup>` for lists.
- **Transitions that never "finish"**, because the element has no `transition` or
  `animation` in `*-active`. Vue then removes it immediately.
- **Using `v-bind()` in CSS for values that change every frame** (mouse position, scroll).
  It works, but setting the inline custom property on each change can be costlier than a
  `transform` on a single element.

## Exercise

1. Build `AlertBox.vue` and render an info and an error alert. Inspect them in devtools and
   find the `data-v-…` attribute and the rewritten selectors.
2. Build `ProgressBar.vue` with buttons that add and subtract 10. Pass a `color` that turns
   red below 20 using a `computed`.
3. Build `ToastMessage.vue`. In `App.vue`, show a toast for 3 seconds when a button is
   clicked (use `setTimeout` and clear it if clicked again). Then enable
   "Emulate CSS prefers-reduced-motion" in devtools' Rendering panel and confirm the
   animation is disabled.
4. Convert `AlertBox`'s styles to a CSS module and compare the rendered class names.
