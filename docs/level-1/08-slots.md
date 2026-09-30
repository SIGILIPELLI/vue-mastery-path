---
description: "Compose Vue components with slots: default and named slots, fallback content, scoped slots that pass data back to the parent, and typed defineSlots."
---

# 08 · Slots

Props pass **data** into a component. Slots pass **markup**. Whenever a component is a
container — a card, a modal, a layout, a list, a table — slots let the parent decide what
goes inside while the component decides the structure and behaviour around it.

## The default slot

```vue title="src/components/BaseCard.vue"
<template>
  <article class="card">
    <slot />
  </article>
</template>
```

```vue-html
<BaseCard>
  <h2>Monthly report</h2>
  <p>Revenue is up 4% on last month.</p>
</BaseCard>
```

Everything between `<BaseCard>` and `</BaseCard>` renders where `<slot />` is.

### Fallback content

Content inside `<slot>` renders only when the parent provides nothing:

```vue-html
<button class="btn" type="button">
  <slot>Submit</slot>
</button>
```

`<SubmitButton />` renders "Submit"; `<SubmitButton>Save draft</SubmitButton>` renders
"Save draft".

### Render scope

Slot content is compiled in the **parent's** scope. It can use the parent's variables, but
not the child's:

```vue-html
<!-- in the parent -->
<BaseCard>
  <p>{{ parentMessage }}</p>   <!-- ✓ parent state -->
  <p>{{ cardInternalState }}</p> <!-- ✗ not visible here -->
</BaseCard>
```

This is deliberate: the parent wrote the content, so the parent owns it. If the child needs
to share data with the slot content, it uses a *scoped slot* (below).

## Named slots

A component can expose several insertion points by naming them:

```vue title="src/components/PageLayout.vue"
<template>
  <div class="layout">
    <header><slot name="header" /></header>
    <main><slot /></main>
    <footer><slot name="footer">© {{ new Date().getFullYear() }}</slot></footer>
  </div>
</template>
```

The parent targets them with `<template #name>` (short for `v-slot:name`):

```vue-html
<PageLayout>
  <template #header>
    <h1>Settings</h1>
  </template>

  <p>Everything not inside a named template goes to the default slot.</p>

  <template #footer>
    <a href="/help">Need help?</a>
  </template>
</PageLayout>
```

Slot names can be dynamic: `<template #[activeTab]>`.

### Rendering wrappers only when a slot is used

`$slots` (or `useSlots()` in script) tells you which slots the parent provided. Use it to
avoid rendering empty wrappers:

```vue-html
<footer v-if="$slots.footer" class="card-footer">
  <slot name="footer" />
</footer>
```

## Scoped slots

Sometimes the child has data the parent's slot content needs. The classic case is a list
component: the child owns iteration, the parent decides how each item looks. The child
passes data to the slot as attributes on `<slot>` — called **slot props**:

```vue title="src/components/DataList.vue"
<script setup lang="ts" generic="T extends { id: string | number }">
defineProps<{ items: T[] }>()

defineSlots<{
  default(props: { item: T; index: number }): any
  empty(): any
}>()
</script>

<template>
  <ul v-if="items.length">
    <li v-for="(item, index) in items" :key="item.id">
      <slot :item="item" :index="index" />
    </li>
  </ul>
  <p v-else>
    <slot name="empty">Nothing here.</slot>
  </p>
</template>
```

The parent receives the props object in `v-slot` / `#default`, usually destructured:

```vue-html
<DataList :items="users">
  <template #default="{ item, index }">
    {{ index }}: <strong>{{ item.name }}</strong> — {{ item.email }}
  </template>
  <template #empty>No users match your search.</template>
</DataList>
```

We mounted this component with two items and a slot of `{{ index }}:{{ item.name }}`, then
again with an empty array and no slot content:

```html
<ul>
  <li>0:a</li>
  <li>1:b</li>
</ul>
<p>Nothing here.</p>
```

Because `DataList` is declared with `generic="T ..."` and typed slots, `item` in the
parent's slot is typed as the actual element type of `users` — autocompletion and type
checking work on `item.email`. Generic components are covered properly in Level 2 · 09.

If a component has **only** a default scoped slot, you can put `v-slot` on the component
tag itself: `<DataList :items="users" v-slot="{ item }">{{ item.name }}</DataList>`.
Don't mix that form with named `<template>`s — use explicit templates for all slots.

## Worked example: a reusable modal

Slots, props and events together:

```vue title="src/components/BaseModal.vue"
<script setup lang="ts">
import { watch, useTemplateRef } from 'vue'

const open = defineModel<boolean>('open', { default: false })
defineProps<{ title: string }>()
const dialog = useTemplateRef<HTMLDialogElement>('dialog')

watch(open, (isOpen) => {
  if (isOpen) dialog.value?.showModal()
  else dialog.value?.close()
}, { flush: 'post' })
</script>

<template>
  <dialog ref="dialog" aria-labelledby="modal-title" @close="open = false">
    <header>
      <h2 id="modal-title">
        <slot name="title">{{ title }}</slot>
      </h2>
    </header>
    <div class="body"><slot :close="() => (open = false)" /></div>
    <footer v-if="$slots.actions">
      <slot name="actions" :close="() => (open = false)" />
    </footer>
  </dialog>
</template>
```

```vue-html
<button type="button" @click="showDelete = true">Delete project</button>

<BaseModal v-model:open="showDelete" title="Delete project?">
  <p>This removes the project and all its tasks. It can't be undone.</p>
  <template #actions="{ close }">
    <button type="button" @click="close">Cancel</button>
    <button type="button" class="danger" @click="deleteProject(); close()">Delete</button>
  </template>
</BaseModal>
```

The modal uses the native `<dialog>` element, which gives you focus trapping, the Escape
key and the backdrop for free. The `close` event fires when the user presses Escape, so the
component writes `open = false` back to the parent. The watcher uses `flush: 'post'` so the
`<dialog>` element exists before `showModal()` is called. The `actions` slot receives a
`close` function, so buttons inside it can close the modal without knowing how.

(The id `modal-title` would clash if two modals were in the DOM at once; Level 3 · 08 shows
`useId()` for generating unique ids.)

## How It Actually Works

The parent's slot content is not rendered by the parent. The compiler turns each slot into
a **function** and passes an object of those functions to the child as its "children":

```js
_createVNode(BaseCard, null, {
  default: _withCtx(() => [ _createElementVNode("h2", null, "Monthly report") ]),
  footer:  _withCtx(() => [ /* ... */ ]),
  _: 1 /* STABLE */
})
```

Inside the child, `<slot name="footer" :x="y" />` compiles to
`_renderSlot(_ctx.$slots, "footer", { x: _ctx.y })`, which **calls** the parent's function
with the slot props. That's the entire mechanism behind scoped slots: slot props are
function arguments.

Two consequences:

1. **Slot content is rendered lazily by the child.** If the child never renders
   `<slot name="footer">`, the parent's footer content is never created at all.
2. **Dependencies are tracked in the child's render.** Because the slot function runs
   during the child's render effect, reactive values the slot content reads become
   dependencies of the *child*. When the parent's data used inside a slot changes, Vue can
   update just the child. `withCtx` makes sure that while the function runs, the
   "current rendering instance" is the parent — so the content still resolves the parent's
   components and scoped CSS ids correctly. The `_: 1 /* STABLE */` hint tells the runtime
   the set of slots can't change between renders, so it can skip comparing them.

## Common mistakes

- **Expecting slot content to see the child's state.** It can't; expose it through slot
  props.
- **Mixing `v-slot` on the component tag with named templates.** Use `<template #default>`
  explicitly when there are named slots.
- **Using `v-if` on `<slot>` to check for content.** Check `$slots.name` instead —
  `<slot>` itself is not an element.
- **Duplicating a list component for each "look".** If two components differ only in how
  an item renders, you want one component with a scoped slot.
- **Expecting an empty slot template to blank out the fallback.** Vue treats a slot that
  renders nothing (or only comments) as empty and shows the fallback instead — we passed
  `<template #footer></template>` to `PageLayout` and still got `<footer>© 2026</footer>`.
  To hide a section, add a prop (or `v-if` on the wrapper) rather than an empty slot.

## Exercise

1. Build `PageLayout.vue` and use it as the root of `App.vue` with header, default and
   footer content. Remove the footer template and confirm the fallback year renders.
2. Build `DataList.vue` and use it to render a list of books with titles in bold and
   authors in italics. Add an input that filters the list, and provide a custom `#empty`
   message that includes the search text.
3. Build `BaseModal.vue`. Open it from a button, then close it with each of: the Cancel
   button, the Delete button and the Escape key. Confirm `showDelete` is `false` every time.
4. Add a `#title` slot usage that includes an icon, and check the `title` prop fallback still
   works when the slot isn't provided.
