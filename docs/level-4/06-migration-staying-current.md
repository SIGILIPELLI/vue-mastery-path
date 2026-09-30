---
description: "Migrate Vue 2 code to Vue 3 with observed behaviour changes, plan an incremental upgrade, and evaluate new releases like Vue 3.6 Vapor Mode without chasing hype."
---

# 06 · Migrating from Vue 2 & Staying Current

Two related jobs land on senior Vue developers sooner or later. The first is **migrating
a Vue 2 codebase** — Vue 2 reached end of life on 31 December 2023, so remaining Vue 2
apps get no security fixes from the core team. The second is **keeping a Vue 3 app
current** without breaking it every few months or chasing every new feature.

Both come down to the same skill: knowing precisely what changed, verifying behaviour
instead of trusting memory, and upgrading in small, reversible steps. The behaviour below
was observed on **Vue 3.5.43** (in Node, via `vue/server-renderer` and jsdom), and the
compiler output at the end came from **Vue 3.6.0-rc.9**, the `rc` tag on npm at the time
of writing.

## The changes that actually bite

The official migration guide lists dozens of changes. In real codebases, a handful cause
most of the work. Each one below was run on Vue 3.5 so you can see the failure rather than
take it on faith.

### 1. `v-if` now runs before `v-for`

Vue 2 code often filtered inline:

```vue
<li v-for="t in todos" v-if="!t.done" :key="t.id">{{ t.text }}</li>
```

In Vue 2, `v-for` had higher precedence, so this worked. In Vue 3, `v-if` is evaluated
first, when `t` doesn't exist yet. Rendering it produced:

```text
[Vue warn]: Property "t" was accessed during render but is not defined on instance.
[Vue warn]: Unhandled error during execution of render function
THREW: Cannot read properties of undefined (reading 'done')
```

**Fix:** filter in a computed (`const openTodos = computed(() => todos.value.filter(t => !t.done))`)
or move `v-if` to an inner element or a wrapping `<template v-for>`.

### 2. `v-bind="obj"` order now matters

```vue
<p id="red" v-bind="{ id: 'blue' }"></p>
<p v-bind="{ id: 'blue' }" id="red"></p>
```

Output on Vue 3.5:

```html
<p id="blue"></p><p id="red"></p>
```

Vue 3 merges in source order: **whichever comes last wins**. In Vue 2 the individual
attribute always won regardless of position. Look for components that spread `$attrs` or a
props object and also set explicit attributes.

### 3. Filters are gone — and the old syntax still compiles

```vue
<p>{{ msg | upper }}</p>
```

Vue 3 removed filters, but `|` is JavaScript's bitwise OR, so this is a valid expression.
With `msg: 'hi'` and a component property named `upper` equal to `2`, it rendered
`<p>2</p>` — no error, just wrong output (`'hi'` becomes `NaN`, which bitwise-ORs as `0`).
If `upper` doesn't exist you get a warning; if it's a method, you get a garbage number.
**Search the codebase for `| ` inside mustaches and `v-bind` values** rather than relying on
errors. Replace filters with functions or computed properties.

### 4. Instance event and reactivity helpers are gone

A template probing the old instance APIs rendered:

```text
[Vue warn]: Property "$on" was accessed during render but is not defined on instance.
[Vue warn]: Property "$set" was accessed during render but is not defined on instance.
[Vue warn]: Property "$listeners" was accessed during render but is not defined on instance.
<p>undefined undefined undefined function</p>
```

- `$on` / `$off` / `$once` — the "event bus" pattern (`new Vue()` as an emitter) must be
  replaced by a store, `provide`/`inject`, or a tiny emitter library.
- `$set` / `Vue.set` / `$delete` — unnecessary: Vue 3's proxies track added and deleted
  properties. Delete the calls.
- `$listeners` — merged into `$attrs` (next point). `$emit` still exists.

### 5. `$attrs` now contains `class`, `style` and listeners

A child with `inheritAttrs: false` rendering `Object.keys($attrs)`, used as
`<Child class="x" style="color:red" data-a="1" @close="f" />`, printed:

```text
class,style,data-a,onClose
```

In Vue 2, `class` and `style` were never in `$attrs` and were always applied to the root,
and listeners lived in `$listeners`. Components that used `inheritAttrs: false` and
`v-bind="$attrs"` on an inner input will now **also move `class` and `style` onto that
input** — a visible layout change that tests may not catch. Also declare emitted events in
`emits`; undeclared ones fall through as `on*` attributes and can fire twice (once from
the listener on the root element, once from `$emit`).

### 6. Watching an array no longer sees mutations

```js
watch: { list() { fired++ } }        // plain watcher
watch: { list: { handler() { deep++ }, deep: true } }
```

After `list.push(2)`: **plain watcher 0, deep watcher 1**. After `list = [3]`: plain
watcher 1. In Vue 3 a watcher on an array only fires when the array is replaced, unless
it's `deep`. Vue 2 fired on mutation. Audit array watchers — this bug is silent.

### 7. Component `v-model` renamed its prop and event

Vue 3's `v-model` on a component uses the `modelValue` prop and `update:modelValue` event
(Vue 2 used `value` and `input`, or the `model` option). A component following the new
contract updated the parent from `Ada` to `Grace` when we dispatched an `input` event.
In new code, `defineModel()` (Vue 3.4+) generates both for you. Multiple `v-model:title`
bindings replace Vue 2's `.sync` modifier.

### The rest, briefly

- **Global API → app instance**: `new Vue({...})` becomes `createApp(App)`;
  `Vue.use/component/directive/mixin` become `app.use/...`. Global config no longer leaks
  between apps (or between tests).
- **Async components** need `defineAsyncComponent(() => import(...))`; plain
  `() => import()` is now only valid for routes.
- **Transition classes**: `v-enter` → `v-enter-from`, `v-leave` → `v-leave-from`.
- **Functional components** are plain functions; `functional: true` and
  `<template functional>` are gone.
- **Ecosystem**: Vue Router 3 → 4+ (`createRouter`, `createWebHistory`), Vuex → Pinia
  (recommended; Vuex 4 exists but isn't where new work happens), and every UI library you
  depend on must have a Vue 3 version. **This last item often decides the timeline.**

## An incremental migration plan

Big-bang rewrites of a large app rarely finish. The path that works:

1. **Upgrade to Vue 2.7** first. It backported `<script setup>`, the Composition API and
   `defineProps`/`defineEmits`, so new code can be written in Vue 3 style while still
   running on Vue 2. Move from Vue CLI/webpack to Vite during this phase if you can
   (Vue 2.7 is supported by `@vitejs/plugin-vue2`).
2. **Remove Vue-2-only patterns while still on 2.7**: filters, event buses, `$listeners`,
   `.sync`, `v-if` with `v-for`, `Vue.set`. Each is a small, testable PR. Lint rules from
   `eslint-plugin-vue`'s Vue 3 configs flag many of them.
3. **Inventory dependencies**: for each Vue-specific library, find its Vue 3 version or a
   replacement. Anything without one is a blocker to plan around.
4. **Switch to Vue 3 with the migration build** (`@vue/compat`). It is Vue 3 with a
   compatibility layer that re-enables most Vue 2 behaviours and **logs a deprecation
   warning for each one it encounters** (with an ID like `INSTANCE_LISTENERS`). You fix
   warnings category by category, turning each compat flag off as you go.
5. **Remove `@vue/compat`** once the warnings are gone, and run the full test suite and a
   manual pass on the highest-traffic pages.

The migration build has documented limits — some libraries that depend on Vue 2 internals
won't work even with it — so read its current list before committing to it. Throughout,
characterisation tests (Level 3 · 07) on the most important components are what let you
refactor with confidence: write them against the Vue 2 behaviour, then keep them green.

## Staying current on Vue 3

Vue 3 minors have added features without removing APIs since 2020 (`defineModel` in 3.4,
`useTemplateRef` and reactive props destructure in 3.5). Staying current is mostly
routine hygiene:

- **Pin and automate.** Keep a lockfile, and let Renovate or Dependabot open one PR per
  update group (Vue core packages together; `vue`, `@vue/*` and `vue-tsc` in step). CI must
  run type-check, unit tests and a build on each.
- **Read the changelog before merging**, not after an incident. Vue's `CHANGELOG.md` and
  GitHub releases list fixes and features per version; majors of Router, Pinia and Vite
  come with migration notes.
- **Know the release channels.** `npm view vue dist-tags` showed, at the time of writing:

  ```text
  { csp: '1.0.28-csp', legacy: '2.7.16', 'v2-latest': '2.7.16',
    alpha: '3.6.0-alpha.7', beta: '3.6.0-beta.17', rc: '3.6.0-rc.9', latest: '3.5.43' }
  ```

  `latest` is what `npm install vue` gives you. `rc`, `beta` and `alpha` are for trying
  things out on a branch, not for production.
- **Follow design discussions where they happen**: the `vuejs/rfcs` repository for
  proposals and the core repo's issues and releases. Blog posts and videos lag behind, and
  some describe features that changed before release.

## Evaluating Vapor Mode (Vue 3.6)

Vapor Mode is the headline of Vue 3.6: an opt-in compilation mode that skips the virtual
DOM. It's enabled per component. We compiled the same counter both ways with
`@vue/compiler-sfc` from 3.6.0-rc.9:

```vue title="Counter.vue"
<script setup vapor>   <!-- remove "vapor" for the classic mode -->
import { ref } from 'vue'
const count = ref(0)
</script>
<template>
  <button @click="count++">Clicked {{ count }} times</button>
</template>
```

Classic (virtual DOM) output — a render function that builds a vnode each update:

```js
return (_ctx, _cache) => {
  return (_openBlock(), _createElementBlock("button", {
    onClick: _cache[0] || (_cache[0] = $event => (count.value++))
  }, "Clicked " + _toDisplayString(count.value) + " times", 1 /* TEXT */))
}
```

Vapor output — DOM created once from a template, then one effect per dynamic spot:

```js
const t0 = _template("<button> ", 1)
// inside setup:
const n0 = t0()
const x0 = _txt(n0)
_on(n0, "click", () => (count.value++))
_renderEffect(() => _setText(x0, "Clicked " + _toDisplayString(count.value) + " times"))
return n0
```

No vnodes, no diff: when `count` changes, only the `_renderEffect` that reads it runs, and
it writes one text node. That's the fine-grained reactivity from Level 3 · 01 applied
directly to the DOM.

How to evaluate it for your app, rather than adopting it because it's new:

1. **Wait for a stable release** (it was RC when this was written) and read its list of
   supported and unsupported features. Vapor components are designed to interoperate with
   classic ones, but some APIs and libraries may not be supported in Vapor components.
2. **Measure first.** Use the profiling approach from Level 3 · 06 to find components that
   are actually expensive — large lists, frequently updating dashboards. If rendering isn't
   your bottleneck, Vapor won't change what users feel.
3. **Convert a leaf component on a branch**, keep its tests green, and compare measured
   work (and bundle size — Vapor has its own runtime) before and after.
4. **Keep the change reversible.** It's one attribute on `<script setup>`; that makes the
   experiment cheap to undo.

We haven't included benchmark numbers because we didn't measure Vapor against a real
app; anything you read should be checked against your own measurements.

## How It Actually Works

- **Why v-if/v-for changed:** the compiler turns directives into nested JavaScript. In Vue
  3, a `v-if` on an element wraps the whole element *including* its `v-for` loop in a
  conditional, so the condition runs in the outer scope where the loop variable doesn't
  exist. That's the `Property "t" was accessed` warning.
- **Why `$set` disappeared:** Vue 2 made objects reactive with `Object.defineProperty`
  getters/setters, created once per existing key, so new keys (and array index writes)
  were invisible without `Vue.set`. Vue 3 wraps objects in a `Proxy`, whose `set` and
  `deleteProperty` traps see every key, including new ones.
- **Why array watchers changed:** a non-deep watcher tracks only the reactive reads its
  getter makes — reading `this.list` tracks the property, not the array's contents. Vue 2
  special-cased array mutation methods to notify the parent property's watchers; Vue 3's
  consistent model needs `deep` (or a getter that reads the contents) for that.
- **How `@vue/compat` works:** it's a build of Vue 3 whose code paths check per-feature
  compat flags. When a flag is on and your code uses the old behaviour, it runs the Vue 2
  semantics and records a deprecation warning; switching the flag off gives you pure
  Vue 3 behaviour for that feature, so you can migrate one category at a time.

## Common mistakes

- **Migrating without tests** — several changes above are silent (filters, array
  watchers, `$attrs` class moves). Only tests and careful review catch them.
- **Big-bang rewrite** of a large app instead of 2.7 → compat → Vue 3 steps.
- **Ignoring dependencies until the end** — a UI library with no Vue 3 version is the most
  common blocker.
- **Blindly search-and-replacing `.sync` and `value`/`input`** without updating the child's
  props and emits.
- **Running prerelease versions in production** because a blog post said a feature was
  "out".
- **Batching a year of upgrades into one PR** — small, frequent updates are easier to
  review and to revert.

## Exercise

1. Paste each of the seven "changes that bite" snippets into a Vue 3 project and reproduce
   the observed output. Then write the fixed version of each.
2. Write a grep (or ESLint config using `eslint-plugin-vue`'s Vue 3 rules) that finds
   filters, `$listeners`, `.sync`, `Vue.set` and `v-if` next to `v-for` in a codebase.
   Run it on any Vue 2 project you can find on GitHub and estimate the migration effort.
3. Set up Renovate or Dependabot on your Level 3 project, grouping `vue`, `@vue/*` and
   `vue-tsc`. What does CI need to run for you to trust an automated update PR?
4. In a throwaway branch, install `vue@rc`, add `vapor` to one leaf component, and compile
   it. Compare the generated code with the classic output. Do its tests still pass?
