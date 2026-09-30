---
description: "Make Vue apps fast with evidence: measure first, keep props stable, use v-memo and v-once, shallow state for big data, virtual scrolling, and bundle and load performance."
---

# 06 · Performance

Vue is fast by default — the compiler optimisations and fine-grained reactivity from
Lessons 01–02 mean most apps never need tuning. When one does, the problem is almost
always one of a handful of patterns: too many components re-rendering, too much data made
deeply reactive, too many DOM nodes, or too much JavaScript on load. This lesson shows how
to find which one you have and fix it, using **re-render counts we measured** rather than
timing claims that depend on your machine.

## Measure first

Two kinds of performance matter, and they're measured differently:

- **Load performance** — how quickly the page becomes usable. Measured with Lighthouse /
  PageSpeed Insights and field data (Core Web Vitals: LCP, INP, CLS). Usually dominated by
  bundle size, network and images.
- **Update performance** — how quickly the UI responds to changes. Measured with the
  browser Performance panel and the Vue devtools' timeline and component render tracking.

In development you can also enable Vue's own performance marks:

```ts title="src/main.ts (excerpt)"
if (import.meta.env.DEV) app.config.performance = true
```

Component init, render and patch then appear as named entries in the browser Performance
panel's "Timings" track. Always profile a **production build** (`vite build && vite
preview`) before drawing conclusions: dev builds include warnings and devtools hooks that
make everything slower.

## Update performance

### 1. Keep props stable

A child component re-renders when any prop it receives changes. The classic mistake is
passing every row the **same changing value** and letting each row compute whether it
applies:

```vue title="src/perf/RowA.vue"
<script setup lang="ts">
const { item, activeId } = defineProps<{ item: { id: number; label: string }; activeId: number }>()
</script>
<template><li :class="{ active: item.id === activeId }">{{ item.label }}</li></template>
```

versus computing it in the parent and passing a boolean:

```vue title="src/perf/RowB.vue"
<script setup lang="ts">
const { item, active } = defineProps<{ item: { id: number; label: string }; active: boolean }>()
</script>
<template><li :class="{ active }">{{ item.label }}</li></template>
```

```vue-html
<RowA v-for="i in items" :key="i.id" :item="i" :active-id="activeId" />
<RowB v-for="i in items" :key="i.id" :item="i" :active="i.id === activeId" />
```

We rendered 100 rows of each, changed the active row once, and counted `onUpdated` calls:

```text
child updates after changing active row: activeId prop = 100 | boolean prop = 2
```

With `activeId`, every row's props changed, so all 100 re-rendered. With `active`, only
the old and new active rows received a different value. Same visual result, 50× fewer
component updates. The general rule: **pass children exactly the data they need, in the
most specific form.**

The same logic applies to objects created in templates — `:options="{ dense: true }"`
creates a new object on every parent render, so the child sees a changed prop every
time. Hoist constant objects into script.

### 2. `v-memo` for large lists

When you can't restructure (or the row is a plain element rather than a component),
`v-memo` tells Vue to skip re-rendering a subtree unless listed values changed:

```vue-html
<li
  v-for="i in items"
  :key="i.id"
  v-memo="[i.id === selected]"
  :class="{ sel: i.id === selected }"
>
  {{ i.label }}
</li>
```

Counting how many list items re-rendered when `selected` changed, with and without it:

```text
v-memo: initial item renders 100 | re-rendered items after selection change 2
no v-memo: re-rendered items after selection change 100
```

The memo array must include **every** value the subtree depends on that can change —
leave one out and that item shows stale content. `v-memo` is a sharp tool: reach for it in
lists of hundreds or thousands of items, after measuring, not by default.

`v-once` is the extreme version: render once, never update. Use it for content that is
truly static but uses data (a legal notice rendered from config, say).

### 3. Shallow state for big data

From Lesson 01: deep reactivity wraps every nested object in a proxy on access and tracks
every property read. For a large dataset you replace wholesale — search results, a table,
map features — use `shallowRef` and update immutably:

```ts
const rows = shallowRef<Row[]>([])
rows.value = await fetchRows()                               // tracked
rows.value = rows.value.map((r) => (r.id === id ? { ...r, done: true } : r))  // tracked
```

### 4. Stable computeds

Computeds that return new objects or arrays notify their dependants even when the content
is identical. Use the `prev` argument (Lesson 01) for computeds feeding expensive watchers
or big subtrees.

### 5. Don't over-componentise hot paths

Each component instance has a cost: an instance object, a reactive props object, a render
effect. In a 10,000-cell grid, wrapping each cell in three layers of tiny components
multiplies that cost. In hot paths, inline markup in the parent's template is cheaper.
(Vapor Mode in Vue 3.6 is aimed partly at this cost; see Level 4 · 06.)

## Too many DOM nodes: virtual scrolling

Rendering 10,000 rows means 10,000+ DOM nodes, however clever the diffing. **Virtual
scrolling** renders only the rows in (and just around) the viewport, and pads the rest with
empty space so the scrollbar is correct:

```vue title="src/components/VirtualList.vue"
<script setup lang="ts" generic="T extends { id: string | number }">
import { computed, ref } from 'vue'

const { items, rowHeight, height } = defineProps<{ items: T[]; rowHeight: number; height: number }>()
defineSlots<{ default(props: { item: T }): any }>()

const scrollTop = ref(0)
const overscan = 5
const start = computed(() => Math.max(0, Math.floor(scrollTop.value / rowHeight) - overscan))
const end = computed(() => Math.min(items.length, Math.ceil((scrollTop.value + height) / rowHeight) + overscan))
const visible = computed(() => items.slice(start.value, end.value))
</script>

<template>
  <div
    class="viewport"
    :style="{ height: `${height}px`, overflowY: 'auto' }"
    @scroll.passive="scrollTop = ($event.target as HTMLElement).scrollTop"
  >
    <div :style="{ height: `${items.length * rowHeight}px`, position: 'relative' }">
      <div :style="{ transform: `translateY(${start * rowHeight}px)` }">
        <div v-for="item in visible" :key="item.id" :style="{ height: `${rowHeight}px` }">
          <slot :item="item" />
        </div>
      </div>
    </div>
  </div>
</template>
```

We mounted it with 50,000 items, 30 px rows and a 300 px viewport, then set `scrollTop` to
30,000 px and fired a scroll event:

```text
rendered rows at top: 15 first: Row 0
after scroll to 30000px: 20 first: Row 995 last: Row 1014
```

Ten visible rows plus five overscan rows at the top (there's nothing above row 0 to
overscan), then ten visible plus five on each side mid-list — twenty DOM rows instead of
fifty thousand. This fixed-row-height version is about 30 lines and handles the common case. Variable row
heights, sticky headers, keyboard navigation and accessibility (screen readers can't see
rows that aren't rendered — use `aria-rowcount`/`aria-rowindex` in grids) are where
libraries such as `vue-virtual-scroller` or TanStack Virtual earn their place.

Also consider the simplest option: pagination or "load more". Most people never scroll
past the first hundred results.

## Load performance

The biggest levers, roughly in order:

1. **Split routes and heavy components** (Lesson 05).
2. **Audit dependencies.** One date library, charting package or icon set can outweigh
   your whole app. Check the build output; prefer packages that support tree-shaking (ES
   modules, named imports).
3. **Images**: correct sizes, modern formats, `loading="lazy"` below the fold, explicit
   `width`/`height` to avoid layout shift.
4. **SSR or prerendering** for content pages, so the first paint doesn't wait for
   JavaScript (Level 4 · 01–02).
5. **Cache hashed assets forever** and the HTML briefly (Level 4 · 08).

## How It Actually Works

Why did 98 `RowB` components skip their update? When the parent re-renders, the renderer
reaches each child component vnode and calls `shouldUpdateComponent(prevVNode, nextVNode)`.
With compiled templates, the child vnode carries a patch flag and the list of dynamic prop
names (`["item", "active"]`), so the check is a loop over just those keys comparing old
and new values with `hasChanged` (an `Object.is` check). `item` is the same object and
`active` is the same boolean for 98 rows, so the function returns `false` and the child's
render effect never runs. For `RowA`, `activeId` differs for every row, so every child
re-renders — and then each produces the same `<li>` for 98 of them, which the DOM patch
finds unchanged. The DOM work was nearly the same; the wasted work was 98 render calls and
vnode diffs.

`v-memo` compiles to a `withMemo(memoArray, () => vnode, _cache, index)` call. It stores the
previous memo array and vnode in the render cache; if every element of the new array is
the same (`Object.is`), it returns the **cached vnode** and the patcher sees an identical
vnode and skips the whole subtree. Inside `v-for`, the compiler caches per item.

## Common mistakes

- **Optimising without measuring** — and measuring a dev build.
- **Passing "global" changing values to every list item** instead of item-specific
  derived props.
- **Inline object/array/function literals as props** to heavy children.
- **An incomplete `v-memo` dependency array** — stale UI that only shows up sometimes.
- **Deep reactivity on large immutable data.**
- **Rendering thousands of rows** when pagination or virtualisation would do.
- **Blaming Vue for bundle size** that comes from dependencies.

## Exercise

1. Reproduce the `RowA`/`RowB` experiment in your project with 1,000 rows. Use the Vue
   devtools' component render highlighting to see which rows update, and the Performance
   panel (production build) to compare the time for one active-row change. Write down
   what *you* measure.
2. Use `VirtualList` to render 50,000 generated rows. Check the number of `<div>` rows in
   the DOM while scrolling. Then add keyboard navigation (↑/↓ move a selected index and
   scroll it into view).
3. Run `vite build` on your Recipe Book, pick the largest chunk, and identify its biggest
   dependency with a bundle visualiser. Could it be lazy-loaded or replaced?
4. Find a component in one of your projects that receives an inline object literal prop.
   Measure its updates before and after hoisting the object.
