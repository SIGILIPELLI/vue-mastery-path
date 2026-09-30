---
description: "Vue template syntax: interpolation, v-bind, v-on with modifiers, v-if versus v-show, v-for with keys, and what the compiler does with each directive."
---

# 03 · Template Syntax & Directives

A Vue template is valid HTML plus a small, fixed vocabulary of special attributes called
**directives** (they all start with `v-`). This lesson covers the ones you'll use every
day, the shorthands, and the rules that trip people up.

We'll build a small product card as we go:

```vue title="src/components/ProductCard.vue"
<script setup lang="ts">
import { ref } from 'vue'

const product = ref({
  name: 'Trail Running Shoe',
  price: 129,
  imageUrl: '/shoe.jpg',
  inStock: true,
  tags: ['running', 'outdoor', 'waterproof'],
})
const quantity = ref(1)
const showDetails = ref(false)
</script>
```

## Text interpolation

Double curly braces render a value as **text**:

```vue-html
<h2>{{ product.name }}</h2>
<p>{{ product.price * quantity }} USD</p>
<p>{{ product.inStock ? 'In stock' : 'Sold out' }}</p>
```

The content is a single JavaScript **expression** — anything that produces a value. You
can call functions and use ternaries, but not statements: `{{ if (x) {...} }}` and
`{{ const y = 1 }}` are compile errors.

Interpolated text is always escaped. If `product.name` were `<b>hi</b>`, the page would
show the literal characters, not bold text. That is a security feature (Level 3 · 09).

Expressions only see what your component exposes plus a short allow-list of globals
(`Math`, `Date`, `JSON`, `Number`, `parseInt` and a few others). `window` and `document`
aren't reachable from a template.

## Attribute binding: `v-bind` / `:`

Curly braces don't work inside attributes. To bind an attribute to a value, use `v-bind`
or its shorthand `:`:

```vue-html
<img v-bind:src="product.imageUrl" :alt="product.name" />
<button :disabled="!product.inStock">Add to cart</button>
```

A few binding behaviours worth knowing:

- **Booleans**: `:disabled="false"` removes the attribute entirely; `true` adds it.
- **`null` / `undefined`** remove the attribute too.
- **Same-name shorthand** (Vue 3.4+): `:alt` on its own means `:alt="alt"`.
- **Bind many at once**: `v-bind="attrs"` spreads an object's keys as attributes.
- **Dynamic attribute names**: `:[attrName]="value"`.

`class` and `style` accept objects and arrays, covered in Lesson 09.

## Event handling: `v-on` / `@`

```vue-html
<button @click="quantity++">+</button>
<button @click="quantity > 1 && quantity--">−</button>
<button @click="showDetails = !showDetails">Details</button>
```

A handler can be:

- **inline code** — `@click="quantity++"`;
- **a method name** — `@click="addToCart"` (Vue calls it with the native event);
- **a method call** — `@click="addToCart(product, $event)"`, where `$event` is the
  native event object.

### Modifiers

Modifiers are dot-suffixes that handle common event chores declaratively:

```vue-html
<form @submit.prevent="save">          <!-- calls event.preventDefault() -->
<div @click.self="close">              <!-- only if the click target is the div itself -->
<a @click.stop="track">                <!-- stopPropagation() -->
<button @click.once="subscribe">       <!-- fires at most once -->
<input @keyup.enter="search" />        <!-- only on the Enter key -->
<input @keydown.ctrl.s.prevent="save" /> <!-- Ctrl+S -->
<div @scroll.passive="onScroll">       <!-- passive listener, better scrolling -->
```

Key modifiers use the kebab-case of `KeyboardEvent.key`: `.enter`, `.esc`, `.page-down`.
Add `.exact` to require that *no other* modifier keys are held.

## Conditional rendering: `v-if` and `v-show`

```vue-html
<p v-if="product.inStock">Ships in 2 days</p>
<p v-else-if="quantity > 5">Back-order large quantities</p>
<p v-else>Currently unavailable</p>

<section v-show="showDetails">
  <h3>Details</h3>
  ...
</section>
```

They look similar but behave very differently:

| | `v-if` | `v-show` |
|---|---|---|
| When false | element is **not in the DOM** | element is in the DOM with `display: none` |
| Toggle cost | creates/destroys elements and child components | flips one CSS property |
| Initial cost when false | nothing rendered | still rendered |
| Child component state | lost when removed | kept |
| Works on `<template>` | yes | no |

Use `v-show` for things toggled often (tabs, dropdowns); use `v-if` for things that are
rarely shown or expensive to render.

To conditionally render a group without a wrapper element, put `v-if` on a `<template>`:

```vue-html
<template v-if="product.inStock">
  <label for="qty">Quantity</label>
  <input id="qty" type="number" min="1" v-model="quantity" />
</template>
```

## List rendering: `v-for`

```vue-html
<ul>
  <li v-for="tag in product.tags" :key="tag">#{{ tag }}</li>
</ul>

<ol>
  <li v-for="(tag, index) in product.tags" :key="tag">{{ index + 1 }}. {{ tag }}</li>
</ol>
```

`v-for` also iterates objects (`(value, key, index) in obj`) and ranges (`n in 5` gives
1 to 5).

### Always provide a stable `:key`

The key tells Vue which DOM element belongs to which item. Use an ID that belongs to the
item — never the index when the list can be reordered, filtered or have items inserted.

Here is why. Imagine a to-do list rendered with `:key="index"` where each row has a text
input:

```vue-html
<li v-for="(todo, index) in todos" :key="index">
  {{ todo.title }} <input placeholder="notes" />
</li>
```

Type "buy milk" into the notes input of the first row, then remove the first to-do. With
index keys, the old row 2 now has key `0`. Vue sees "key 0 still exists, its text changed"
and **reuses the first `<li>`**, updating its text but keeping the `<input>` that still
contains "buy milk". The note appears to jump to the wrong row. With `:key="todo.id"`,
Vue removes the correct `<li>` and the input goes with it.

### `v-if` and `v-for` on the same element

Don't. In Vue 3, `v-if` is evaluated **before** `v-for`, so it can't see the loop
variable:

```vue-html
<!-- ✗ 'tag' is not defined when v-if runs -->
<li v-for="tag in tags" v-if="tag !== 'outdoor'" :key="tag">{{ tag }}</li>
```

Filter in a `computed` instead (Lesson 04), or move `v-for` to a wrapping `<template>`:

```vue-html
<template v-for="tag in tags" :key="tag">
  <li v-if="tag !== 'outdoor'">{{ tag }}</li>
</template>
```

(This precedence is the reverse of Vue 2, a common source of migration bugs.)

## Other built-in directives

| Directive | What it does |
|-----------|--------------|
| `v-model` | Two-way binding for form inputs and components (Lesson 07) |
| `v-html` | Sets `innerHTML` — **never** use with untrusted content (Level 3 · 09) |
| `v-text` | Sets `textContent`; equivalent to `{{ }}` for the whole element |
| `v-once` | Render once, then treat as static |
| `v-memo` | Skip re-rendering a subtree unless listed values change (Level 3 · 06) |
| `v-pre` | Skip compilation — shows `{{ }}` literally |
| `v-cloak` | Stays on the element until the app mounts; used with CSS to hide raw templates in no-build setups |
| `v-slot` / `#` | Pass content into a component's slots (Lesson 08) |

## The finished template

```vue-html title="src/components/ProductCard.vue (template)"
<template>
  <article class="card">
    <img :src="product.imageUrl" :alt="product.name" width="240" height="160" />
    <h2>{{ product.name }}</h2>
    <p class="price">{{ product.price * quantity }} USD</p>

    <template v-if="product.inStock">
      <button type="button" @click="quantity > 1 && quantity--" aria-label="Decrease">−</button>
      <span>{{ quantity }}</span>
      <button type="button" @click="quantity++" aria-label="Increase">+</button>
    </template>
    <p v-else>Currently unavailable</p>

    <button type="button" @click="showDetails = !showDetails" :aria-expanded="showDetails">
      {{ showDetails ? 'Hide' : 'Show' }} details
    </button>
    <ul v-show="showDetails">
      <li v-for="tag in product.tags" :key="tag">#{{ tag }}</li>
    </ul>
  </article>
</template>
```

## How It Actually Works

Directives are **compile-time** instructions, not runtime attributes. The compiler rewrites
each one into plain JavaScript in the render function:

- `{{ product.name }}` becomes `_toDisplayString(_ctx.product.name)`. `toDisplayString`
  turns `null`/`undefined` into `''` and objects into formatted JSON — which is why
  `{{ someObject }}` prints JSON rather than `[object Object]`.
- `:disabled="x"` becomes a `disabled: x` entry in the element's props object, plus a
  patch flag marking `disabled` as dynamic.
- `@click="quantity++"` becomes an `onClick` function. Inline handlers are wrapped in an
  arrow function (`$event => (quantity.value++)`), and the compiler caches the function in
  `_cache` so the same function object is reused across renders — the runtime never has to
  remove and re-add the listener.
- `@submit.prevent` becomes `withModifiers(handler, ["prevent"])`, a tiny wrapper that
  calls `preventDefault()` first.
- `v-if`/`v-else` become a ternary in the render function that returns one vnode or the
  other (or a comment placeholder `<!--v-if-->` when nothing renders, so Vue remembers
  where to insert content later).
- `v-show` becomes a runtime directive that sets `el.style.display`, preserving any
  display value you had set yourself.
- `v-for` becomes a call to `renderList(source, callback)` that returns an array of vnodes.
  During updates, the renderer matches old and new children **by key** and computes the
  minimal set of moves (using a longest-increasing-subsequence algorithm) to reorder
  existing DOM nodes rather than recreate them.

Because all of this happens at build time, a production app ships only the runtime — not
the template compiler — which saves roughly the size of the compiler in your bundle.

## Common mistakes

- **Using `{{ }}` inside attributes.** `<img src="{{ url }}">` sets the literal string.
  Use `:src="url"`.
- **Index keys on mutable lists** — state attached to DOM (inputs, focus, animations,
  child component state) ends up on the wrong item.
- **`v-if` + `v-for` on one element** — `v-if` runs first and can't see the loop variable.
- **Calling expensive functions in the template.** `{{ formatAll(items) }}` runs on every
  re-render of the component. Use a `computed`, which caches.
- **Forgetting `type="button"`** on buttons inside a `<form>` — the default type is
  `submit`, so clicking "+" submits the form.
- **`v-show` on a `<template>`** — it has no element to hide, so it does nothing.

## Exercise

Build `ProductCard.vue` from this lesson in your project and render it from `App.vue`.
Then:

1. Add a `discount` field (for example `0.15`) and show both the original price
   (struck through with `<s>`) and the discounted price, only when `discount > 0`.
2. Add a text input bound with `@keyup.enter` that pushes a new tag into `product.tags`.
   Prevent duplicate tags.
3. Render the tags with `:key="index"` and add an input next to each tag. Type into the
   second input, then remove the first tag with a button. Observe the bug described above,
   then fix it with `:key="tag"`.
4. Replace `v-show` with `v-if` on the details list, and use the Vue devtools (or the
   Elements panel) to confirm that the `<ul>` disappears from the DOM when hidden.
