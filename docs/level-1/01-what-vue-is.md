---
description: "What Vue.js is: a progressive framework built on a reactive state system and a template compiler, the Composition API, and when Vue is a good fit."
---

# 01 · What Vue Is

Vue is a JavaScript framework for building user interfaces. You describe what the screen
should look like for a given piece of state, and Vue keeps the DOM in sync with that state
as it changes. That sentence describes most modern UI frameworks, so it is more useful to
look at what makes Vue *Vue*:

- **A reactive state system at the core.** You wrap values in `ref()` or `reactive()`, and
  Vue records which parts of the UI read them. When a value changes, Vue knows exactly
  which components need to re-render — you never call `setState` or tell it "something
  changed".
- **HTML-based templates, compiled ahead of time.** A Vue template looks like HTML with a
  few extra attributes (`v-if`, `v-for`, `:href`, `@click`). A compiler turns it into
  optimised JavaScript render functions at build time, and uses what it can see in the
  template (which parts are static, which bindings are dynamic) to make updates cheaper.
- **Single-File Components.** A `.vue` file keeps a component's logic, template and styles
  together in one file with three blocks: `<script setup>`, `<template>` and `<style>`.
- **Progressive adoption.** Vue can be dropped into a single page with a `<script>` tag to
  add interactivity to server-rendered HTML, or it can power a full single-page app with a
  router, a store, server-side rendering and a build pipeline. You add the pieces you need.

## A first look

Here is a complete Vue component. Don't worry about every detail yet — each piece gets its
own lesson.

```vue title="src/components/ClickCounter.vue"
<script setup lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const label = computed(() => (count.value === 1 ? 'time' : 'times'))

function increment() {
  count.value++
}
</script>

<template>
  <button type="button" @click="increment">
    Clicked {{ count }} {{ label }}
  </button>
</template>

<style scoped>
button {
  font: inherit;
  padding: 0.5rem 1rem;
}
</style>
```

Read it top to bottom:

1. `ref(0)` creates a reactive value. In script you read and write it through `.value`.
2. `computed()` derives another value from it. It re-evaluates only when `count` changes.
3. The template uses `count` and `label` **without** `.value` — refs are unwrapped
   automatically in templates.
4. `@click="increment"` attaches a click listener.
5. `scoped` styles apply only to this component's elements.

Everything declared at the top level of `<script setup>` — variables, functions, imported
components — is available to the template. There is no `return`, no `export default`, no
registration step.

## The two APIs: Composition and Options

Vue 3 supports two ways of writing a component.

The **Composition API** (above) uses imported functions — `ref`, `computed`, `watch`,
`onMounted` — inside `<script setup>`. Related logic sits together and can be pulled out
into reusable functions called *composables* (Level 2).

The **Options API** describes a component as an object with named sections:

```vue title="src/components/ClickCounterOptions.vue"
<script lang="ts">
import { defineComponent } from 'vue'

export default defineComponent({
  data() {
    return { count: 0 }
  },
  computed: {
    label(): string {
      return this.count === 1 ? 'time' : 'times'
    },
  },
  methods: {
    increment() {
      this.count++
    },
  },
})
</script>

<template>
  <button type="button" @click="increment">Clicked {{ count }} {{ label }}</button>
</template>
```

Both compile to the same runtime and can live side by side in one app. The Options API
is still fully supported and you will meet it in older codebases (and in Vue 2 migration
work — Level 4). This course uses the Composition API with `<script setup>` and
TypeScript throughout, because that is what `create-vue` scaffolds, what the official docs
lead with, and what scales best as components grow.

## What ships in "Vue" and what is separate

The `vue` package gives you the reactivity system, the component model, the template
compiler and the built-in components (`Transition`, `KeepAlive`, `Teleport`, `Suspense`).
The rest of the official ecosystem is separate packages maintained by the core team:

| Need | Package | Covered in |
|------|---------|-----------|
| Client-side routing | `vue-router` | Level 2 · 03–04 |
| Shared state | `pinia` | Level 2 · 05–06 |
| Build tooling & dev server | `vite` + `@vitejs/plugin-vue` | Level 1 · 02, Level 4 · 03 |
| Component testing | `@vue/test-utils` (with Vitest) | Level 3 · 07 |
| Type checking `.vue` files | `vue-tsc` | Level 2 · 09 |
| Full-stack meta-framework | `nuxt` (community-led, closely aligned) | Level 4 · 02 |

This is a real difference from Angular (one package set, one release train) and from React
(a view library where routing and state are entirely community choices). Vue sits in the
middle: small core, official recommendations for the rest.

## When Vue is a good fit

Vue works well when:

- you want a gentle on-ramp — templates are HTML, and the reactivity model is small enough
  to learn in an afternoon;
- you are adding interactivity to an existing server-rendered site (Laravel, Rails,
  Django) one widget at a time;
- you are building a single-page app or dashboard and want official, well-integrated
  answers for routing and state;
- you want SSR or static generation later — Nuxt gives you that without changing how you
  write components.

It is a less obvious fit when your team is already deep in another framework's ecosystem
(component libraries, internal tooling), or when you need a native mobile UI from the same
code — Vue has no first-party equivalent of React Native.

## How It Actually Works

When you run `npm run dev` or `npm run build`, the Vite plugin for Vue splits each `.vue`
file into its blocks. The `<template>` goes through `@vue/compiler-dom`, which produces a
**render function**. For a template like:

```vue-html
<div class="card">
  <h2>Static title</h2>
  <p :class="{ active: isActive }">{{ message }}</p>
</div>
```

the compiler emits (we ran `@vue/compiler-dom` 3.5.43 on it; trimmed):

```js
const _hoisted_1 = { class: "card" }

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock("div", _hoisted_1, [
    _cache[0] || (_cache[0] = _createElementVNode("h2", null, "Static title", -1 /* CACHED */)),
    _createElementVNode("p", {
      class: _normalizeClass({ active: _ctx.isActive })
    }, _toDisplayString(_ctx.message), 3 /* TEXT, CLASS */)
  ]))
}
```

Three things to notice already:

1. The static `<h2>` is created once and **cached** — it is never re-created or diffed.
2. The `<p>` carries a **patch flag** (`3 /* TEXT, CLASS */`) telling the runtime that
   only its text and class can ever change, so an update skips checking everything else.
3. The render function reads `_ctx.isActive` and `_ctx.message`. When it runs inside
   Vue's rendering *effect*, those reads are recorded as dependencies of this component.

That last point is the whole engine. `ref` and `reactive` values notify their subscribers
when written. A component's render runs inside an effect that subscribes to whatever it
reads. So changing `message` schedules a re-render of exactly the components that read
`message` — batched to run once per tick, not once per assignment. Level 3 opens up both
halves (the reactivity system and the renderer) in detail.

## Versions in this course

This course targets **Vue 3.5** (3.5.43 was the `latest` tag on npm while it was written),
with **Vue Router 5.3**, **Pinia 4.0** and **Vite 8.3**, exactly as scaffolded by
`create-vue` 3.24. Two things to know about the near future:

- **Vue 3.6** was at release-candidate stage (`3.6.0-rc.x` on npm's `rc` tag) at the time
  of writing. Its headline feature is **Vapor Mode**, an opt-in compilation strategy that
  skips the virtual DOM for components that use it. It is opt-in and component-level, so
  everything in this course still applies; Level 4 · 06 covers how to evaluate it.
- **Vue Router 5** merged file-based routing (previously `unplugin-vue-router`) into the
  core package without breaking the classic API. **Pinia 4** is technically breaking only
  in packaging (ESM-only, newer devtools API). Code written for Vue Router 4 / Pinia 3
  generally runs unchanged.

Run `npm ls vue vue-router pinia` in your project to see exactly what you have.

## Common mistakes

- **Forgetting `.value` in script.** `count++` on a ref does nothing useful — it tries to
  increment the ref object. In `<script>` it is always `count.value++`; in the template it
  is `count`.
- **Mixing the two APIs in one component without a reason.** It works, but it forces
  readers to hold two mental models. Pick one per component.
- **Assuming Vue re-renders "everything" on each change.** It re-renders only components
  whose reactive dependencies changed. If a component doesn't update, the usual cause is
  that it never *read* a reactive value (Lesson 04).
- **Treating old tutorials as current.** Vue 2 reached end of life on 31 December 2023.
  Search results still contain Vue 2 idioms (`this.$set`, filters, `Vue.component`,
  event buses with `$on`) that do not exist in Vue 3.

## Exercise

1. Without installing anything, create an `index.html` that loads Vue from a CDN and mounts
   an app. Use the ES-module build:

    ```html
    <div id="app">
      <button @click="count++">Count: {{ count }}</button>
    </div>
    <script type="module">
      import { createApp, ref } from 'https://unpkg.com/vue@3/dist/vue.esm-browser.js'
      createApp({ setup: () => ({ count: ref(0) }) }).mount('#app')
    </script>
    ```

    Open it in a browser. This is the "progressive" end of Vue: no build step at all.
2. Add a second button that resets the count, and a paragraph that shows "even" or "odd"
   using a `computed`.
3. Open the browser devtools console and type `document.querySelector('button').outerHTML`.
   Note that the `@click` attribute is gone — Vue compiled the template in the browser
   and attached a real listener. Write one sentence on why a build step (next lesson)
   moves that compile work out of the browser.
