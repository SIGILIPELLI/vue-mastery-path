---
description: "Share data down a Vue component tree without prop drilling: typed provide/inject with InjectionKey, readonly state, app-level provides and a compound Tabs component."
---

# 02 · Provide & Inject

Props are perfect for passing data one level down. They get painful when a value is needed
five levels deep and every component in between has to accept and forward it — **prop
drilling**. `provide` and `inject` let an ancestor make a value available to *any*
descendant, however deep, without the components in between knowing about it.

```text
App                    provide('theme', theme)
└── Layout
    └── Sidebar
        └── NavList
            └── NavItem     inject('theme')
```

## The basics

```vue title="src/App.vue"
<script setup lang="ts">
import { provide, ref } from 'vue'

const theme = ref<'light' | 'dark'>('light')
provide('theme', theme)
</script>
```

```vue title="src/components/NavItem.vue"
<script setup lang="ts">
import { inject, type Ref } from 'vue'

const theme = inject<Ref<'light' | 'dark'>>('theme')
</script>

<template>
  <a :class="theme">…</a>
</template>
```

Provide a **ref** (not `theme.value`) so injectors stay reactive: when the ancestor changes
`theme.value`, every injector that reads it updates.

### Defaults and missing providers

If nothing above provides the key, `inject` returns `undefined` and Vue warns in
development. We injected a key that nobody provided:

```text
[Vue warn]: injection "absent" not found.
```

Pass a default as the second argument to make a provider optional:

```ts
const size = inject('density', 'comfortable')
const config = inject('config', () => createDefaultConfig(), true) // factory: 3rd arg = true
```

In a test where an ancestor provided `{ theme: 'dark' }` under a symbol key and nobody
provided `'nope'`, a leaf component that did `inject(key)` and `inject('nope', 'fallback')`
rendered `<p>dark fallback</p>`.

## Type-safe keys with `InjectionKey`

String keys can collide and give you no type information. Use a `Symbol` typed with
`InjectionKey<T>`, exported from a shared module:

```ts title="src/keys.ts"
import type { InjectionKey, Ref } from 'vue'

export interface CurrentUser { id: number; name: string; role: 'admin' | 'member' }

export const CurrentUserKey: InjectionKey<Readonly<Ref<CurrentUser | null>>> = Symbol('CurrentUser')
```

```ts
provide(CurrentUserKey, readonly(user))   // TypeScript checks the provided value's type
const user = inject(CurrentUserKey)       // typed as Readonly<Ref<CurrentUser | null>> | undefined
```

The `| undefined` is honest — there might be no provider. Wrap the inject in a small
function that throws a helpful error, and callers get a non-optional type:

```ts
export function useCurrentUser() {
  const user = inject(CurrentUserKey)
  if (!user) throw new Error('useCurrentUser() called outside <UserProvider>')
  return user
}
```

## Keep mutations in the provider

If descendants can write to injected state, you've recreated global mutable state with
extra steps. Provide state as `readonly()` and provide **functions** for the allowed
changes:

```ts
const user = ref<CurrentUser | null>(null)

provide(CurrentUserKey, readonly(user))
provide(AuthActionsKey, {
  login: async (email: string, password: string) => { /* ...sets user.value */ },
  logout: () => { user.value = null },
})
```

Now there's exactly one place where `user` changes.

## App-level provide

`app.provide()` makes a value available to every component in the app. Plugins use this
(the router and Pinia both do):

```ts title="src/main.ts"
const app = createApp(App)
app.provide(ApiBaseKey, import.meta.env.VITE_API_BASE ?? '/api')
app.mount('#app')
```

It's also a clean way to inject services you want to swap in tests:

```ts
mount(UserCard, { global: { provide: { [ApiBaseKey as symbol]: 'http://localhost:9999' } } })
```

## Worked example: a compound Tabs component

Provide/inject shines in **compound components** — a parent and children designed to be
used together, where the parent coordinates and the children register themselves. The
usage we want:

```vue-html
<TabGroup v-model="tab">
  <TabPanel id="profile" label="Profile"><ProfileForm /></TabPanel>
  <TabPanel id="billing" label="Billing"><BillingForm /></TabPanel>
  <TabPanel id="security" label="Security"><SecurityForm /></TabPanel>
</TabGroup>
```

The shared context and a guarded inject helper:

```ts title="src/components/tabs/tabs-context.ts"
import { inject, type InjectionKey, type Ref } from 'vue'

export interface TabsContext {
  active: Readonly<Ref<string>>
  select: (id: string) => void
  register: (id: string, label: string) => void
  unregister: (id: string) => void
  baseId: string
}

export const TabsKey: InjectionKey<TabsContext> = Symbol('Tabs')

export function useTabs(): TabsContext {
  const ctx = inject(TabsKey)
  if (!ctx) throw new Error('<TabPanel> must be used inside <TabGroup>')
  return ctx
}
```

The group owns the state, renders the tab buttons and handles arrow-key navigation:

```vue title="src/components/tabs/TabGroup.vue"
<script setup lang="ts">
import { provide, readonly, ref, useId } from 'vue'
import { TabsKey } from './tabs-context'

const active = defineModel<string>({ default: '' })
const tabs = ref<{ id: string; label: string }[]>([])
const baseId = useId()

provide(TabsKey, {
  active: readonly(active),
  select: (id) => (active.value = id),
  register(id, label) {
    tabs.value.push({ id, label })
    if (!active.value) active.value = id
  },
  unregister(id) {
    tabs.value = tabs.value.filter((t) => t.id !== id)
  },
  baseId,
})

function onKeydown(e: KeyboardEvent) {
  const i = tabs.value.findIndex((t) => t.id === active.value)
  const n = tabs.value.length
  const next = e.key === 'ArrowRight' ? (i + 1) % n : e.key === 'ArrowLeft' ? (i - 1 + n) % n : -1
  if (next < 0) return
  active.value = tabs.value[next]!.id
  document.getElementById(`${baseId}-tab-${active.value}`)?.focus()
}
</script>

<template>
  <div class="tabs">
    <div role="tablist" @keydown="onKeydown">
      <button
        v-for="t in tabs"
        :id="`${baseId}-tab-${t.id}`"
        :key="t.id"
        type="button"
        role="tab"
        :aria-selected="t.id === active"
        :aria-controls="`${baseId}-panel-${t.id}`"
        :tabindex="t.id === active ? 0 : -1"
        @click="active = t.id"
      >
        {{ t.label }}
      </button>
    </div>
    <slot />
  </div>
</template>
```

Each panel registers itself and shows only when active:

```vue title="src/components/tabs/TabPanel.vue"
<script setup lang="ts">
import { onBeforeUnmount } from 'vue'
import { useTabs } from './tabs-context'

const { id, label } = defineProps<{ id: string; label: string }>()
const tabs = useTabs()
tabs.register(id, label)
onBeforeUnmount(() => tabs.unregister(id))
</script>

<template>
  <div
    v-show="tabs.active.value === id"
    :id="`${tabs.baseId}-panel-${id}`"
    role="tabpanel"
    :aria-labelledby="`${tabs.baseId}-tab-${id}`"
    tabindex="0"
  >
    <slot />
  </div>
</template>
```

`useId()` (Vue 3.5+) generates an id that's unique within the app and stable between
server and client rendering, so two `TabGroup`s on a page don't produce clashing
`aria-controls` ids. We mounted two panels and inspected the output **immediately** after
mounting, then again after one `nextTick()`:

```text
sync: <div role="tablist"></div>
after tick: <div role="tablist"><button id="v-0-tab-profile" type="button" role="tab" aria-selected="true" aria-controls="v-0-panel-profile" tabindex="0">Profile</button><button id="v-0-tab-billing" type="button" role="tab" aria-selected="false" aria-controls="v-0-panel-billing" tabindex="-1">Billing</button></div>
```

The tab buttons appear one tick late. That's inherent to this pattern and worth
understanding (see below). After pressing → the second tab was selected and focused:

```text
active after →: Billing focused: <button id="v-0-tab-billing" type="button" role="tab" aria-selected="true" aria-controls="v-0-panel-billing" tabindex="0">Billing</button>
```

Using a `TabPanel` on its own threw our guard's message —
`<TabPanel> must be used inside <TabGroup>` — right after Vue's own
`injection "Symbol(Tabs)" not found` warning.

## How It Actually Works

Every component instance has a `provides` object. When a component is created, its
`provides` is simply a **reference to its parent's** `provides`. The first time a component
calls `provide()`, Vue replaces its `provides` with `Object.create(parentProvides)` — a new
object whose *prototype* is the parent's — and sets the key on it. The root's `provides`
has the app-level provides as its prototype.

So `inject(key)` is just a property lookup on the parent's `provides` object (`key in
provides`), and JavaScript's prototype chain does the walk up the tree. A nearer provider
shadows a farther one for the same key, exactly like variable scoping. Nothing is copied,
and components that don't provide anything add no cost.

`inject` looks at the **parent's** provides, not the component's own — so a component
can't inject what it provided itself, and it can provide a new value under the same key
for its own descendants.

The one-tick delay in the tabs: `TabGroup` renders first, with `tabs` empty. The panels'
`setup` runs during that same render pass (as children are created) and `push`es into
`tabs`. That mutation happens while `TabGroup` is already rendering, so Vue schedules a
second render of `TabGroup`, which runs in the next flush. Registration patterns always
cost one extra render; a component library that needs the buttons in the very first
render (for SSR, say) has the parent read its children's props from the slot instead.

## Common mistakes

- **Providing `ref.value` instead of the ref** — injectors get a frozen snapshot.
- **Using provide/inject for truly global app state** (logged-in user used on every page,
  a shopping cart). It works, but Pinia (Lesson 05) gives you devtools, SSR safety and
  testing helpers. Provide/inject is best for *subtree-scoped* context: a form and its
  fields, a tabs group and its panels, a theme for one section.
- **Calling `inject` outside `setup`** (in an event handler, after `await`). It needs the
  current instance. Inject during setup and keep the result.
- **String keys in shared libraries** — use symbols.
- **Mutating injected state from descendants.** Provide readonly state and actions.

## Exercise

1. Build `TabGroup`/`TabPanel` and use them with `v-model`. Add Home and End key support
   (first and last tab).
2. Add a `disabled` prop to `TabPanel` that makes its tab button unfocusable and skipped
   by arrow keys.
3. Build a `FormGroup` component that provides `{ disabled, size }` to any `FormInput`
   inside it, with `FormInput` falling back to its own props when there's no group.
4. Write a Vitest test that mounts `TabPanel` inside a tiny test component that provides a
   fake `TabsKey` context, and asserts that it calls `register` with its id and label.
