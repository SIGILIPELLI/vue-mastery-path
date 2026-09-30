---
description: "Vue's built-in components in depth: Teleport for modals, KeepAlive caching with include and max, Transition modes, TransitionGroup list moves, and Suspense."
---

# 04 · Teleport, KeepAlive, Transition & Suspense

Vue ships five built-in components that solve problems you can't solve cleanly with normal
components: rendering somewhere else in the DOM (`Teleport`), keeping components alive
while hidden (`KeepAlive`), animating enter/leave (`Transition`, `TransitionGroup`) and
coordinating async loading (`Suspense`). You've met some briefly; this lesson covers how
they behave, with observed output from Vue 3.5.43.

## `<Teleport>`

A modal logically belongs to the component that opens it — it uses that component's state
and emits to it. But visually it must escape its ancestors: a parent with
`overflow: hidden`, `transform` or a low `z-index` stacking context will clip or bury it.
`<Teleport>` renders its content into another DOM node while keeping it part of the
component tree.

```vue-html
<section class="card" style="overflow: hidden">
  <Teleport to="#modals" :disabled="!open">
    <div class="dialog">Hi</div>
  </Teleport>
</section>
```

With `index.html` containing `<div id="app"></div><div id="modals"></div>`, we rendered it
and printed both containers:

```text
app: <div data-v-app=""><section class="card" style="overflow: hidden;"><!--teleport start--><!--teleport end--></section></div>
modals: <div class="dialog">Hi</div>
```

The dialog lives in `#modals`; the card keeps two comment anchors marking where the
teleport sits logically. Setting `disabled` to `true` renders the content in place instead
(useful for, say, a panel that's inline on desktop and a full-screen overlay on mobile):

```text
disabled -> app: <div data-v-app=""><section class="card" style="overflow: hidden;"><!--teleport start--><div class="dialog">Hi</div><!--teleport end--></section></div>
```

What stays "logical":

- props, events, provide/inject and slots work exactly as if the content were in place;
- scoped styles still apply (the content carries the component's scope attribute);
- DOM events **bubble through the real DOM**, so a click inside the teleported dialog does
  *not* bubble to the `section` — but a Vue event emitted from a component inside it still
  reaches the parent component.

The `to` target must exist when the Teleport mounts. If the target is rendered by the same
app later in the same render pass, add `defer` (Vue 3.5+), which waits until the rest of the
app has mounted. A deferred teleport targeting a `#later` element that appears *after* it
in the same template worked: `defer target content: <p>deferred</p>`.

Level 1 · 08's `BaseModal` used the native `<dialog>` with `showModal()`, which renders in
the browser's **top layer** — above everything, regardless of stacking contexts. For
dialogs, `showModal()` is often all you need; `Teleport` remains useful for non-modal
overlays (toasts, popovers, dropdowns) and for older browser support.

## `<KeepAlive>`

Switching between components with `v-if` or `<component :is>` destroys the old one — its
state, scroll position and form inputs are lost. `<KeepAlive>` caches deactivated component
instances instead:

```vue-html
<KeepAlive :include="['InboxView', 'SearchView']" :max="5">
  <component :is="currentView" />
</KeepAlive>
```

With the router:

```vue-html
<RouterView v-slot="{ Component }">
  <KeepAlive include="SearchView">
    <component :is="Component" />
  </KeepAlive>
</RouterView>
```

Cached components get two extra hooks. In Level 1 we switched A → B → A (having clicked A's
counter twice first):

```text
A 2 | A activated, A deactivated, B activated, B deactivated, A activated
```

A's counter was still 2 on return. `onActivated` runs on first mount *and* every time the
component comes back from the cache; `onDeactivated` runs when it goes into the cache.
Refresh stale data in `onActivated`; pause timers and subscriptions in `onDeactivated`.

- **`include` / `exclude`** match the component's **name**. With `<script setup>`, the name
  is inferred from the file name (`SearchView.vue` → `SearchView`), or set it with
  `defineOptions({ name: 'SearchView' })`.
- **`max`** limits the cache; the least recently used instance is destroyed when the limit
  is exceeded. With `max: 2`, visiting A, B, C, then A again:

```text
mount A, mount B, unmount A, mount C, unmount B, mount A
```

  Visiting C evicted A (least recently used); returning to A mounted it fresh and evicted B.

Keep caches small and targeted: every cached instance holds its DOM and state in memory.

## `<Transition>` beyond basics

Level 1 · 09 covered the CSS classes. A few more capabilities:

**Transition between elements or components** — only one child renders at a time, so key
it or use `v-if`/`v-else`:

```vue-html
<Transition name="fade" mode="out-in">
  <component :is="step" :key="stepIndex" />
</Transition>
```

Without `mode`, the leaving and entering elements animate simultaneously and briefly both
occupy space. `out-in` waits for the leave to finish first — usually what you want for
content swaps. `in-out` is rarely useful.

**JavaScript hooks** — for animations driven by a library such as the Web Animations API:

```vue-html
<Transition :css="false" @enter="onEnter" @leave="onLeave">
  <div v-if="show" class="panel">…</div>
</Transition>
```

```ts
function onEnter(el: Element, done: () => void) {
  el.animate([{ opacity: 0, transform: 'scale(0.95)' }, { opacity: 1, transform: 'none' }], {
    duration: 180,
    easing: 'ease-out',
  }).onfinish = done
}
function onLeave(el: Element, done: () => void) {
  el.animate([{ opacity: 1 }, { opacity: 0 }], { duration: 120 }).onfinish = done
}
```

`:css="false"` tells Vue not to look for CSS transitions, and you **must** call `done`.

**Route transitions:**

```vue-html
<RouterView v-slot="{ Component, route }">
  <Transition name="fade" mode="out-in">
    <component :is="Component" :key="route.path" />
  </Transition>
</RouterView>
```

## `<TransitionGroup>`

For lists, `TransitionGroup` animates each item's enter and leave, and — with a
`*-move` class — animates items **moving** to new positions when the list is reordered:

```vue-html
<TransitionGroup name="list" tag="ul">
  <li v-for="item in items" :key="item.id">{{ item.title }}</li>
</TransitionGroup>
```

```css
.list-move,
.list-enter-active,
.list-leave-active { transition: all 250ms ease; }
.list-enter-from,
.list-leave-to { opacity: 0; transform: translateX(1rem); }
.list-leave-active { position: absolute; }   /* take leaving items out of the flow so others can move */
```

`tag` sets the wrapper element (none by default). Every child needs a unique `key`.
`mode` is not supported, because items enter and leave independently.

## `<Suspense>`

`<Suspense>` waits for **async dependencies** in its subtree — components with top-level
`await` in `<script setup>`, and `defineAsyncComponent` components (next lesson) — and shows
a fallback until all of them resolve. From Level 2 · 07:

```text
<Suspense> is an experimental feature and its API will likely change.
before: <p>loading...</p>
after: <p>loaded</p>
```

Useful properties:

- **One fallback for many async children**: a dashboard with five widgets that each await
  their data shows one loading state instead of five spinners popping in.
- **`timeout`**: when new content is loading and the old content is still showing, the
  fallback appears only after this many milliseconds, avoiding flicker for fast loads.
- **Events**: `@pending`, `@resolve`, `@fallback`.
- **Errors** are not handled by Suspense; catch them with `onErrorCaptured` in a parent
  (Lesson 09).

Nesting order with the router, `KeepAlive` and `Transition` matters. The documented order
is:

```vue-html
<RouterView v-slot="{ Component }">
  <template v-if="Component">
    <Transition mode="out-in">
      <KeepAlive>
        <Suspense>
          <component :is="Component" />
          <template #fallback>Loading…</template>
        </Suspense>
      </KeepAlive>
    </Transition>
  </template>
</RouterView>
```

Suspense still carries its experimental label in Vue 3.5; it's widely used (Nuxt depends
on it), but test edge cases in your own app rather than assuming.

## How It Actually Works

These aren't ordinary components; the renderer gives each special treatment.

- **Teleport** has its own `process` function in the renderer. On mount it inserts the two
  anchor comments at the logical position, resolves the target (`querySelector` when `to`
  is a string), and mounts its children into the target instead of the parent element.
  Because the vnode tree is unchanged, patching, unmounting, provide/inject and component
  events work as usual; only the physical `insert` calls use a different container.
  Toggling `disabled` simply *moves* the existing DOM nodes between the two containers.
- **KeepAlive** keeps a `Map` from key (component type or vnode key) to cached vnode and an
  LRU `Set` of keys. When its child is switched away, instead of unmounting it calls a
  `deactivate` that moves the child's DOM into an off-document `<div>` "storage container"
  and runs `deactivated` hooks. When the same key comes back, it moves the DOM back and
  runs `activated` hooks — no re-render needed. Exceeding `max` prunes the oldest key with
  a real unmount, which is the `unmount A` in the log above.
- **Transition** wraps its child vnode's `transition` property with enter/leave hooks that
  the renderer calls around insertion and removal. For leave, the renderer calls the
  leave hook and only removes the element in the `done` callback, which fires on
  `transitionend`/`animationend` or after the computed duration.
- **TransitionGroup**'s move animation uses the FLIP technique: before an update it records
  each child's position (`getBoundingClientRect`), after the update it records the new
  position, applies an inverse `transform` so each item *appears* not to have moved, then
  removes the transform with the `*-move` class active so the browser animates it into
  place.
- **Suspense** renders its default slot into an off-DOM container. Async setups register
  their promises with the nearest Suspense boundary (`suspense.registerDep`); when the
  pending count hits zero, Suspense moves the resolved subtree into the real DOM and
  replaces the fallback.

## Common mistakes

- **Teleporting to a target that doesn't exist yet** — Vue warns and renders nothing. Put
  the target in `index.html` or use `defer`.
- **Expecting DOM events to bubble through a Teleport** to the logical parent.
- **KeepAlive `include` not matching** because the component has no name you expected —
  check the component name in devtools.
- **Unbounded KeepAlive caches** in apps with many views — set `max`.
- **Forgetting `done()` in JS transition hooks** — the element never finishes entering or
  is never removed.
- **Leaving items without `position: absolute`** in `TransitionGroup` — other items can't
  move until the leave animation ends.

## Exercise

1. Build a `PopoverMenu` that teleports its panel to `body`, positions it under its trigger
   button using `getBoundingClientRect()`, and closes with `v-click-outside` from Lesson 03.
2. Wrap your Recipe Book's list view in `KeepAlive` (by name, via the router) so search text
   and scroll position survive visiting a recipe and pressing Back. Refresh the list in
   `onActivated` if it's older than a minute.
3. Add a sortable list with `TransitionGroup` and buttons for "sort by name" and "shuffle".
   Remove the `position: absolute` rule and describe the difference.
4. Build a dashboard with three widgets that each `await` a fake request of a different
   duration inside a single `<Suspense>` with a `timeout` of 200 ms. Observe when the
   fallback appears on first load versus when switching widgets.
