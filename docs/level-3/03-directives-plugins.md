---
description: "Extend Vue with custom directives for low-level DOM behaviour (click-outside, autofocus) and with plugins that install components, provides and global APIs."
---

# 03 · Custom Directives & Plugins

Components and composables cover almost everything. Two extension points remain for the
rest: **custom directives**, for reusable low-level behaviour attached to a DOM element,
and **plugins**, for installing app-wide functionality in one `app.use()` call. This lesson
builds two directives and a toast-notification plugin, all tested.

## When a directive is the right tool

A custom directive gets direct access to an element and hooks into its lifecycle. Use one
when the behaviour is **about the element itself** and would be awkward as a component:
focusing, measuring, observing intersection or resize, detecting clicks outside, adding a
third-party behaviour (a tooltip library, an input mask) to an existing element.

If the behaviour needs its own markup or state that renders, it's a component. If it's
logic without an element, it's a composable.

## Directive hooks

A directive is an object with optional hooks, each called with `(el, binding, vnode,
prevVnode)`:

| Hook | Called |
|------|--------|
| `created` | before the element's attributes and listeners are applied |
| `beforeMount` | before the element is inserted |
| `mounted` | after the element and its children are in the DOM |
| `beforeUpdate` / `updated` | before / after the containing component updates |
| `beforeUnmount` / `unmounted` | before / after the element is removed |

`binding` contains `value` (the expression's result), `oldValue` (in update hooks), `arg`
(`v-my:arg`), `modifiers` (`v-my.a.b` → `{ a: true, b: true }`) and `instance`.

## Example 1: `v-autofocus`

```ts title="src/directives/autofocus.ts"
import type { Directive } from 'vue'

/** v-autofocus — focus on mount; v-autofocus="false" to disable; v-autofocus.select to also select text */
export const vAutofocus: Directive<HTMLInputElement | HTMLTextAreaElement, boolean | undefined, 'select'> = {
  mounted(el, { value, modifiers }) {
    if (value === false) return
    el.focus()
    if (modifiers.select) el.select()
  },
}
```

The type parameters of `Directive` are the element type, the value type and the allowed
modifiers, so `v-autofocus.selct` (typo) is a type error in templates checked by
`vue-tsc`.

Why not the HTML `autofocus` attribute? It only works on page load. Elements that appear
later — a field inside a modal or an inline editor — need programmatic focus when they
mount. In our test the input was focused on mount and, with `.select`, its text was
selected: `focused: true selection: 0 5`.

## Example 2: `v-click-outside`

Closing a menu or popover when the user clicks elsewhere is a classic directive:

```ts title="src/directives/clickOutside.ts"
import type { Directive } from 'vue'

type Handler = (event: PointerEvent) => void

interface State {
  handler: Handler           // latest bound function
  listener: (e: Event) => void
}
const state = new WeakMap<HTMLElement, State>()

export const vClickOutside: Directive<HTMLElement, Handler> = {
  mounted(el, binding) {
    const s: State = {
      handler: binding.value,
      listener: (e) => {
        if (!el.contains(e.target as Node)) s.handler(e as PointerEvent)
      },
    }
    state.set(el, s)
    // capture phase: still fires if something inside the page stops propagation
    document.addEventListener('pointerdown', s.listener, true)
  },
  updated(el, binding) {
    // inline handlers are new functions on every render; keep the latest one
    const s = state.get(el)
    if (s) s.handler = binding.value
  },
  unmounted(el) {
    const s = state.get(el)
    if (s) document.removeEventListener('pointerdown', s.listener, true)
    state.delete(el)
  },
}
```

Design points:

- **Per-element state lives in a `WeakMap`** keyed by the element, so it's garbage
  collected with the element and doesn't pollute the DOM node with custom properties.
- **The listener is registered once** and always calls the *latest* handler. Templates
  often pass inline arrow functions (`v-click-outside="() => open = false"`), which are
  new functions on every render; the `updated` hook swaps in the new one. Our first version
  removed and re-added the document listener in `updated` — it worked, but did DOM work on
  every render of the parent for no reason.
- **`pointerdown` in the capture phase** fires before any click handler inside the page can
  call `stopPropagation()`, and covers mouse, touch and pen.
- **`unmounted` removes the listener** — forgetting that leaks one listener per mount.

Usage in `<script setup>`: any imported variable whose name starts with `v` and is
camelCase can be used as a directive directly:

```vue title="src/components/DropdownMenu.vue"
<script setup lang="ts">
import { ref } from 'vue'
import { vClickOutside } from '@/directives/clickOutside'

const open = ref(false)
</script>

<template>
  <div class="dropdown">
    <button type="button" :aria-expanded="open" @click="open = !open">Options</button>
    <ul v-if="open" v-click-outside="() => (open = false)" role="menu">
      <li role="menuitem">Rename</li>
      <li role="menuitem">Delete</li>
    </ul>
  </div>
</template>
```

In our test, a `pointerdown` inside the menu left it open; one on a button outside closed
it and the menu was removed from the DOM:

```text
after inside click open = true
after outside click open = false menu exists false
```

(For a real dropdown menu, also close on Escape and manage focus — Lesson 08 covers that.)

To register a directive globally instead: `app.directive('click-outside', vClickOutside)`.

### Directives on components

A directive on a component applies to the component's **root element**. If the component
has several root nodes, Vue warns and ignores it. Prefer putting directives on elements.

### SSR

Directive hooks don't run during server rendering (there's no DOM). If a directive must
affect the server HTML — say, adding an attribute — implement the `getSSRProps` hook.

## Plugins

A plugin is an object with an `install(app, options)` method, or a function with the same
signature. Inside `install` you can do anything you'd otherwise do in `main.ts`:

- `app.component()` — register global components;
- `app.directive()` — register global directives;
- `app.provide()` — make a service injectable everywhere;
- `app.config.globalProperties.x = ...` — add `$x` to every component instance (for
  templates and the Options API);
- `app.mixin()` — possible, but avoid in new code.

Vue Router and Pinia are plugins; so are most UI libraries.

### Example: a toast plugin

```ts title="src/plugins/toast/index.ts"
import { reactive, readonly, inject, type App, type InjectionKey } from 'vue'
import ToastHost from './ToastHost.vue'

export interface Toast { id: number; message: string; kind: 'info' | 'success' | 'error' }
export interface ToastApi {
  toasts: readonly Toast[]
  show: (message: string, kind?: Toast['kind'], ms?: number) => void
  dismiss: (id: number) => void
}

export const ToastKey: InjectionKey<ToastApi> = Symbol('toast')

export function createToast(options: { duration?: number } = {}) {
  const duration = options.duration ?? 4000
  const toasts = reactive<Toast[]>([])
  let nextId = 1

  const api: ToastApi = {
    toasts: readonly(toasts) as readonly Toast[],
    show(message, kind = 'info', ms = duration) {
      const id = nextId++
      toasts.push({ id, message, kind })
      if (ms > 0) setTimeout(() => api.dismiss(id), ms)
    },
    dismiss(id) {
      const i = toasts.findIndex((t) => t.id === id)
      if (i !== -1) toasts.splice(i, 1)
    },
  }

  return {
    install(app: App) {
      app.provide(ToastKey, api)
      app.component('ToastHost', ToastHost)
      app.config.globalProperties.$toast = api
    },
  }
}

export function useToast(): ToastApi {
  const api = inject(ToastKey)
  if (!api) throw new Error('useToast() requires app.use(createToast())')
  return api
}

declare module 'vue' {
  interface ComponentCustomProperties {
    $toast: ToastApi
  }
  interface GlobalComponents {
    ToastHost: typeof ToastHost
  }
}
```

```vue title="src/plugins/toast/ToastHost.vue"
<script setup lang="ts">
import { useToast } from './index'
const toast = useToast()
</script>

<template>
  <div class="toast-host" role="status" aria-live="polite">
    <TransitionGroup name="toast">
      <p v-for="t in toast.toasts" :key="t.id" class="toast" :class="t.kind">
        {{ t.message }}
        <button type="button" aria-label="Dismiss" @click="toast.dismiss(t.id)">×</button>
      </p>
    </TransitionGroup>
  </div>
</template>

<style scoped>
.toast-host { position: fixed; bottom: 1rem; right: 1rem; display: grid; gap: 0.5rem; }
.toast { margin: 0; padding: 0.5rem 0.75rem; border-radius: 6px; background: #1f2937; color: #fff; }
.toast.error { background: #b91c1c; }
.toast.success { background: #15803d; }
.toast-enter-active, .toast-leave-active { transition: opacity 150ms; }
.toast-enter-from, .toast-leave-to { opacity: 0; }
</style>
```

Install and use it:

```ts title="src/main.ts (excerpt)"
import { createToast } from './plugins/toast'

app.use(createToast({ duration: 3000 }))
```

```vue-html title="src/App.vue (template excerpt)"
<RouterView />
<ToastHost />
```

```ts
// in any component
const toast = useToast()
toast.show('Recipe saved', 'success')
```

Notes on the design:

- A **factory** (`createToast(options)`) returns the plugin, the same pattern as
  `createPinia()` and `createRouter()`. Each app gets its own state — important for SSR and
  tests.
- The API is **provided** with a typed key and consumed with a `useToast()` composable —
  the Composition API way. `$toast` in `globalProperties` is there for templates and Options
  API components; the `declare module 'vue'` block types both it and the global
  `<ToastHost>` component.
- The host uses `role="status"` and `aria-live="polite"`, so screen readers announce new
  toasts without interrupting.

In our test, one toast created through `useToast()` and one through `$toast`, then after the
duration elapsed (fake timers):

```text
[ 'toast success:Saved ×', 'toast info:Global ×' ]
after timeout 0
```

## How It Actually Works

**Directives** are compiled into the render function as `withDirectives(vnode, [[dir,
value, arg, modifiers]])`. That helper stores the directive bindings on the vnode
(`vnode.dirs`). The renderer calls each hook at the matching point of its own element
lifecycle — `invokeDirectiveHook(vnode, prevVNode, instance, 'mounted')` right after
inserting the element, `beforeUpdate`/`updated` around patching it, and so on. The
`updated` hook runs whenever the **containing component** re-renders, even if the
directive's value didn't change — which is why the click-outside directive keeps that hook
cheap. Built-ins such as `v-show` and `v-model` are implemented exactly the same way.

**`app.use(plugin, options)`** checks that the plugin hasn't been installed on this app
already (Vue keeps a `Set` of installed plugins and warns on a second install), then calls
`plugin.install(app, options)`, or `plugin(app, options)` if it's a function. There's no
further magic. `app.provide` writes into the app context's `provides` object, which is the
root of the prototype chain every component's `inject` walks (Level 2 · 02).
`globalProperties` are consulted by the component proxy's `get` trap as a last resort,
after setup state, data, props and `$` built-ins.

## Common mistakes

- **Using a directive where a component or composable fits better** — directives can't
  render markup and are harder to test in isolation.
- **Forgetting cleanup in `unmounted`** — listeners, observers and third-party instances
  leak.
- **Reading `binding.value` only in `mounted`** — if the value can change, handle
  `updated` too.
- **Mutating the element in ways Vue will overwrite** — setting `el.className` in a
  directive fights `:class` bindings. Use `classList.add/remove`.
- **Singleton plugin state at module level** — shared across SSR requests and across test
  apps. Create state inside the factory.
- **Relying on `globalProperties` in `<script setup>`** — they're not in scope there. Use
  provide/inject.

## Exercise

1. Write `v-intersect` that calls a handler when the element enters the viewport, using
   `IntersectionObserver`, with a `.once` modifier. Use it to lazy-load images.
2. Add Escape-to-close to `v-click-outside` as an option (`v-click-outside="{ handler,
   escape: true }"`), keeping the type definitions accurate.
3. Extend the toast plugin with a `promise(p, { loading, success, error })` method that
   shows a loading toast and replaces it with the outcome.
4. Write a Vitest test that installs the plugin twice on the same app and assert on the
   warning Vue prints. Then test `useToast()` outside an app and assert on your error.
