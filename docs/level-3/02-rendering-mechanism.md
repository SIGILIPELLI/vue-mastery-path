---
description: "How Vue renders: virtual DOM, compiler-informed patch flags, blocks and cached static nodes, keyed diffing, and writing render functions and functional components."
---

# 02 · The Rendering Mechanism & Render Functions

Lesson 01 explained how Vue knows *that* a component must update. This lesson explains
*how* it updates the DOM: virtual nodes, the patch algorithm, and the compiler hints that
let Vue skip most of the work. You'll also write components directly as **render
functions**, which is occasionally the right tool and always a good way to understand what
templates compile into.

## Virtual DOM in one paragraph

A component's render function returns a tree of **vnodes** — plain objects describing
elements (`{ type: 'button', props: { onClick }, children: 'Clicked 3' }`), text,
comments, fragments and child components. On first render, Vue walks the tree and creates
real DOM nodes. On update, it calls the render function again to get a new tree and
**patches**: it compares new vnodes with old ones and applies only the differences to the
DOM. Vnodes are cheap JavaScript objects; DOM operations are comparatively expensive, so
diffing in JavaScript and touching the DOM minimally is a good trade.

A naive virtual DOM diffs *everything* on every update, including parts of the template
that can never change. Vue's compiler makes that unnecessary.

## Compiler-informed virtual DOM

Because Vue compiles templates, the compiler can see which parts are static and which
bindings are dynamic, and pass that knowledge to the runtime. We compiled this template
with `@vue/compiler-dom` 3.5.43:

```vue-html
<ul class="list">
  <li v-for="item in items" :key="item.id">{{ item.name }}</li>
</ul>
<p v-if="!items.length">Empty</p>
```

```js
export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock(_Fragment, null, [
    _createElementVNode("ul", { class: "list" }, [
      (_openBlock(true), _createElementBlock(_Fragment, null, _renderList(_ctx.items, (item) => {
        return (_openBlock(), _createElementBlock("li", {
          key: item.id
        }, _toDisplayString(item.name), 1 /* TEXT */))
      }), 128 /* KEYED_FRAGMENT */))
    ]),
    (!_ctx.items.length)
      ? (_openBlock(), _createElementBlock("p", { key: 0 }, "Empty"))
      : _createCommentVNode("v-if", true)
  ], 64 /* STABLE_FRAGMENT */))
}
```

Three optimisations are visible:

### 1. Patch flags

The number after the children is a **patch flag**, a bitmask of what can change on that
node. `1 /* TEXT */` on the `<li>` says "only the text content is dynamic". When patching,
the runtime checks the flag and compares only text — not props, not class, not style.
Common flags: `TEXT` (1), `CLASS` (2), `STYLE` (4), `PROPS` (8, with a list of the dynamic
prop names), `FULL_PROPS` (16, dynamic keys like `v-bind="obj"`), `NEED_HYDRATION` (32),
`STABLE_FRAGMENT` (64), `KEYED_FRAGMENT` (128), `UNKEYED_FRAGMENT` (256). Negative values
mark special cases, like `-1 /* CACHED */` for static content.

The `<ul class="list">` has **no** flag: nothing about it is dynamic except its children.

### 2. Blocks and tree flattening

`openBlock()` + `createElementBlock()` make a **block**: a vnode that collects every
*dynamic* descendant into a flat array, `dynamicChildren`. When the block re-renders, the
runtime walks only that array instead of the whole subtree. A card with fifty static
elements and one interpolation patches one node.

Structural directives start new blocks, because they can change the shape of the tree:
each `v-if` branch is its own block (with a `key`, so switching branches replaces rather
than diffs), and each `v-for` produces a fragment block whose children are diffed as a
list. `128 /* KEYED_FRAGMENT */` tells the runtime to use the keyed algorithm.

### 3. Cached static content

Lesson 01 of Level 1 showed a static `<h2>` compiled to
`_cache[0] || (_cache[0] = _createElementVNode(...), -1 /* CACHED */)`. Static subtrees are
created once and reused; the patcher skips them entirely. Large static chunks may be
stringified into a single `innerHTML` insertion.

## Keyed list diffing

When a `KEYED_FRAGMENT` updates, Vue:

1. Syncs matching keys from the **start** of both lists, then from the **end** (this makes
   appends, prepends and single removals fast).
2. For the unknown middle section, builds a map from key to new index, patches every old
   node that still exists, and unmounts the ones that don't.
3. Computes the **longest increasing subsequence** of the kept nodes' new positions. Nodes
   in that subsequence are already in relative order and stay put; everything else is
   moved. This minimises DOM moves.

This is why stable keys matter so much (Level 1 · 03): the key is the only way the
algorithm knows two vnodes represent the same item.

## Render functions

Templates are the right choice almost always. Render functions are useful when the
structure is highly dynamic — a heading whose level is a prop, a component that renders
a tree of arbitrary depth from data, library components that wrap arbitrary slot content.

`h(type, props?, children?)` creates a vnode:

```ts title="src/components/Counter.ts"
import { h, ref, defineComponent, type FunctionalComponent } from 'vue'

// a functional component: no state, no instance — just props in, vnodes out
export const Heading: FunctionalComponent<{ level: 1 | 2 | 3 | 4 | 5 | 6 }> = (props, { slots }) =>
  h(`h${props.level}`, slots.default?.())
Heading.props = ['level']

export const Counter = defineComponent({
  props: { start: { type: Number, default: 0 } },
  emits: ['change'],
  setup(props, { emit, slots }) {
    const n = ref(props.start)
    return () =>
      h('div', { class: 'counter' }, [
        h(Heading, { level: 2 }, () => slots.title?.() ?? 'Counter'),
        h('button', { type: 'button', onClick: () => { n.value++; emit('change', n.value) } }, `Clicked ${n.value}`),
      ])
  },
})
```

Mounted with `start: 2` and a `title` slot of `'Votes'`, then clicked once:

```html
<div class="counter">
  <h2>Votes</h2><button type="button">Clicked 3</button>
</div>
```

It emitted `change` with `[[3]]`. Rules to remember:

- `setup` **returns a function** that returns vnodes. That function is the render effect;
  refs read inside it are tracked. Returning the vnodes directly (not a function) would
  render once and never update.
- Events are props named `onXxx`.
- Slots are **functions**: pass `() => children` for a component's default slot, or an
  object `{ default: () => ..., title: () => ... }` for several. Call `slots.x?.()` to
  render slots you received.
- `v-if` is a ternary, `v-for` is `.map()` (with `key` in props), `v-model` is a
  `modelValue` prop plus an `'onUpdate:modelValue'` handler.
- Hand-written render functions get **no patch flags or blocks** — the runtime falls back
  to full diffing for them. That's fine for small dynamic pieces, and another reason
  templates are the default.

### JSX

With `@vitejs/plugin-vue-jsx` (added by `create-vue --jsx`), you can write the same thing
as JSX: `<button onClick={() => n.value++}>Clicked {n.value}</button>`. Vue's JSX transform
is not React's: it uses `class` rather than `className`, and supports directives such as
`v-show`. We didn't set up JSX for this course; if your team prefers it, the concepts in
this section transfer directly.

## How It Actually Works

The renderer is a set of mutually recursive functions — `patch`, `processElement`,
`processComponent`, `patchChildren`, `patchKeyedChildren` — that take an old vnode and a new
one. `patch` first checks whether they're the **same type and key**; if not, it unmounts
the old subtree and mounts the new one (which is why changing a `key` forces a remount).
If they are, it dispatches by type: elements patch props and children; components decide
whether the child needs to re-render by comparing props (`shouldUpdateComponent`) and, if
so, trigger the child's own render effect.

The renderer never touches the DOM directly. It calls a small **host API** — `createElement`,
`insert`, `remove`, `setElementText`, `patchProp` — passed in by `@vue/runtime-dom`. That
split is why the same component system can render to the DOM, to a string on the server
(Level 4 · 01), or to custom targets (canvas, terminal, native) via `createRenderer`.

Vue 3.6's **Vapor Mode** (opt-in, release-candidate at the time of writing) goes one step
further: for components compiled in Vapor mode, the compiler emits code that creates DOM
nodes directly and binds each dynamic spot to its own reactive effect — no vnodes, no
diffing. The fine-grained reactivity from Lesson 01 is what makes that possible. It's
covered as an evaluation topic in Level 4 · 06.

## Common mistakes

- **Returning vnodes instead of a render function from `setup`** — the component never
  updates.
- **Reusing the same vnode object twice in a tree** — each position needs its own vnode.
  Call `h()` again or use a function.
- **Passing slot content as a plain array to a component** instead of a function — Vue
  warns about non-function default slots, and you lose lazy slot rendering.
- **Choosing render functions for performance.** Templates are usually *faster*, because
  of patch flags and blocks.
- **Forgetting keys in `.map()`** inside render functions.

## Exercise

1. Paste one of your own components' templates into the Vue SFC Playground
   (play.vuejs.org) and open the "JS" output tab. Identify every patch flag and every
   block.
2. Write a recursive `TreeView` render function that renders nested `<ul>`s from
   `{ label: string; children?: Node[] }[]`, with a scoped slot to customise each label.
3. Rewrite `Heading` as a template-based SFC using `<component :is="`h${level}`">`. Which
   version is clearer?
4. Using `createRenderer` from `@vue/runtime-core`, write a renderer that "renders" to a
   plain JavaScript object tree (implement `createElement`, `insert`, `setElementText`,
   `patchProp` and friends as object operations) and mount a tiny counter into it.
