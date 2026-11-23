---
title: "Vue.js Interview Questions and Answers (179 Questions)"
description: "179 Vue.js interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Vue.js Mastery Path."
---

# Vue.js Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Vue.js Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Vue Is](level-1/01-what-vue-is.md)

**Q1.** What kind of framework is Vue?

**Answer:** A progressive JavaScript framework for building user interfaces with a reactive, component-based model

You can adopt it incrementally.

**Q2.** What does 'progressive' mean for Vue?

**Answer:** You can use it for a small widget or scale up to a full single-page app with routing and state management

Core is small; the ecosystem is optional.

**Q3.** Which style does the course use for components?

**Answer:** Single-file components (.vue) with &lt;script setup&gt; and the Composition API

SFCs keep template, logic and style together.

**Q4.** What does the template syntax render here?

```js
const app = createSSRApp({ setup() { const n = ref(3); return { n } }, template: '<p>{{ n * 2 }}</p>' })
console.log(await renderToString(app))
```

**Answer:** &lt;p&gt;6&lt;/p&gt;

Mustache interpolation evaluates a JavaScript expression. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

### [02 · create-vue, Vite &amp; Project Structure](level-1/02-tooling-project-structure.md)

**Q5.** Which tool is the standard way to scaffold a Vue 3 project?

**Answer:** create-vue (npm create vue@latest), which sets up Vite

Vite provides the dev server and build.

**Q6.** What does the Vite dev server give you?

**Answer:** Fast startup and hot module replacement using native ES modules

Edits appear instantly.

**Q7.** Where does a Vue SFC put its markup?

**Answer:** In the &lt;template&gt; block (logic in &lt;script&gt;, styles in &lt;style&gt;)

Blocks compile to a render function and component options.

**Q8.** What file imports and mounts the app in a create-vue project?

**Answer:** main.js (or main.ts), which calls createApp(App).mount('#app')

The root component is App.vue.

### [03 · Template Syntax &amp; Directives](level-1/03-template-syntax.md)

**Q9.** What does this template render?

```js
const app = createSSRApp({ data: () => ({ items: ['a','b'], on: true, html: '<b>x</b>' }), template: '<ul><li v-for="(i, idx) in items" :key="i">{{ idx }}:{{ i }}</li></ul><p v-if="on" :class="{ hot: on }">yes</p><p v-else>no</p><span v-html="html"></span>' })
console.log(await renderToString(app))
```

**Answer:** &lt;ul&gt;&lt;li&gt;0:a&lt;/li&gt;&lt;li&gt;1:b&lt;/li&gt;&lt;/ul&gt;&lt;p class="hot"&gt;yes&lt;/p&gt;&lt;span&gt;&lt;b&gt;x&lt;/b&gt;&lt;/span&gt;

v-for, v-if, class binding and v-html in one template. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q10.** What does :class="{ active: isActive }" do?

**Answer:** Adds the class active when isActive is truthy

The colon is shorthand for v-bind.

**Q11.** What is the difference between v-if and v-show?

**Answer:** v-if adds or removes the element; v-show toggles display: none

Use v-show for frequent toggling.

**Q12.** Why is :key needed on v-for lists?

**Answer:** It gives Vue a stable identity per item so it can update, reorder and reuse elements correctly

Use a unique id, not the index, when items can move.

### [04 · Reactivity: ref, reactive, computed](level-1/04-reactivity-ref-reactive-computed.md)

**Q13.** What does this reactive code print?

```js
const count = ref(1)
const double = computed(() => count.value * 2)
const state = reactive({ n: 1, nested: { x: 1 } })
count.value++
state.nested.x = 5
const { n } = state
state.n = 9
console.log(count.value, double.value, state.nested.x, n, state.n)
```

**Answer:** 2 4 5 1 9

Destructuring a reactive object copies the primitive value, losing reactivity; use toRefs to keep it. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q14.** What does computed caching show here?

```js
let runs = 0
const a = ref(2)
const sq = computed(() => { runs++; return a.value ** 2 })
sq.value; sq.value; sq.value
a.value = 3
sq.value
console.log(runs, sq.value)
```

**Answer:** 2 9

A computed re-runs only when a dependency changes. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q15.** What does toRefs let you do?

```js
const s = reactive({ a: 1, b: 2 })
const { a } = toRefs(s)
a.value = 10
console.log(s.a, isRef(a))
```

**Answer:** 10 true

toRefs creates refs linked to the reactive object's properties. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q16.** Why do refs need .value in script but not in templates?

**Answer:** Templates automatically unwrap top-level refs

Nested refs inside plain objects are not unwrapped.

### [05 · Watchers &amp; Lifecycle Hooks](level-1/05-watchers-lifecycle.md)

**Q17.** In what order do watch and watchEffect callbacks run on change?

```js
const n = ref(0)
const log = []
watch(n, (v, old) => log.push(`watch ${old}->${v}`))
watchEffect(() => log.push(`effect ${n.value}`))
n.value = 1
await nextTick()
console.log(log.join(' | '))
```

**Answer:** effect 0 | watch 0-&gt;1 | effect 1

watchEffect runs immediately; callbacks flush before render, in creation order. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q18.** What is the difference between watch and computed?

**Answer:** computed derives a cached value; watch runs side effects when a source changes

Prefer computed for derived state.

**Q19.** Which hook is the first where the component's DOM is available?

**Answer:** onMounted

Fetching often starts in setup or onMounted.

**Q20.** Why clean up in onUnmounted or onWatcherCleanup?

**Answer:** Timers, listeners and subscriptions would otherwise leak after the component is gone

Return a cleanup from watchers too.

### [06 · Components: Props &amp; Emits](level-1/06-props-emits.md)

**Q21.** How does a child component send data up to its parent?

**Answer:** By emitting an event with defineEmits that the parent listens to

Props down, events up.

**Q22.** Why should a child not mutate a prop?

**Answer:** Props are one-way; mutating hides data flow and Vue warns about it

Copy to local state or emit an update.

**Q23.** What does this component render with props and an emit?

```js
const Child = defineComponent({ props: { label: String, count: { type: Number, default: 0 } }, emits: ['bump'], template: '<button @click="$emit(\'bump\')">{{ label }}: {{ count }}</button>' })
const app = createSSRApp({ components: { Child }, template: '<Child label="Clicks" :count="2" />' })
console.log(await renderToString(app))
```

**Answer:** &lt;button&gt;Clicks: 2&lt;/button&gt;

Props are declared with types and defaults; events are not rendered into HTML. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q24.** What does defineProps with TypeScript give you?

**Answer:** Compile-time prop types inferred from an interface or generic

defineProps&lt;{ title: string }&gt;().

### [07 · Forms &amp; v-model](level-1/07-forms-v-model.md)

**Q25.** What does v-model on a text input and a checkbox produce here?

```js
const app = createSSRApp({ data: () => ({ name: 'Ana', agree: true }), template: '<input v-model="name"><input type="checkbox" v-model="agree">' })
console.log(await renderToString(app))
```

**Answer:** &lt;input value="Ana"&gt;&lt;input type="checkbox" checked&gt;

v-model binds the value (or checked state) both ways. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q26.** What does v-model.trim do?

**Answer:** Removes leading and trailing whitespace from the input value

Also .number and .lazy modifiers.

**Q27.** What does v-model on a component do under the hood?

**Answer:** Passes a modelValue prop and listens for update:modelValue

defineModel() simplifies it.

**Q28.** Why validate on the server as well as in the form?

**Answer:** Client validation can be bypassed

Treat client checks as UX only.

### [08 · Slots](level-1/08-slots.md)

**Q29.** What do default and named slots render?

```js
const Card = defineComponent({ template: '<div><h2><slot name="title">Untitled</slot></h2><slot>No content</slot></div>' })
const app = createSSRApp({ components: { Card }, template: '<Card><template #title>Hi</template><p>Body</p></Card><Card />' })
console.log(await renderToString(app))
```

**Answer:** &lt;div&gt;&lt;h2&gt;Hi&lt;/h2&gt;&lt;p&gt;Body&lt;/p&gt;&lt;/div&gt;&lt;div&gt;&lt;h2&gt;Untitled&lt;/h2&gt;No content&lt;/div&gt;

Slots with fallback content render the fallback when nothing is provided. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q30.** What are scoped slots for?

**Answer:** The child exposes data to the slot content provided by the parent

&lt;slot :item="item"&gt; with v-slot="{ item }".

**Q31.** What is the shorthand for v-slot:header?

**Answer:** #header

The hash shorthand.

**Q32.** Why use slots rather than many props for layout?

**Answer:** The parent controls the markup while the child controls structure and behaviour

Flexible composition.

### [09 · Styling &amp; Transitions](level-1/09-styling-transitions.md)

**Q33.** What does &lt;style scoped&gt; do?

**Answer:** Limits CSS to the current component by adding data attributes to its elements

Use :deep() to reach child internals.

**Q34.** What does &lt;Transition&gt; do?

**Answer:** Applies enter and leave CSS classes or hooks when an element is added or removed

Name drives class names.

**Q35.** What does v-bind() in CSS allow?

**Answer:** Using reactive component state inside style rules

Compiled to CSS variables.

**Q36.** What does &lt;TransitionGroup&gt; add?

**Answer:** Animations for lists that are added, removed or reordered

Needs keys.

### [10 · Project — A Habit Tracker](level-1/10-project-habit-tracker.md)

**Q37.** What does this habit-tracker logic print?

```js
const habits = ref([{ id: 1, name: 'Read', done: false }, { id: 2, name: 'Run', done: true }])
const remaining = computed(() => habits.value.filter(h => !h.done).length)
function toggle(id) { const h = habits.value.find(h => h.id === id); h.done = !h.done }
toggle(1); toggle(2)
console.log(remaining.value, habits.value.map(h => `${h.name}:${h.done}`).join(','))
```

**Answer:** 1 Read:true,Run:false

Mutating items inside a ref array is reactive and computed values update. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q38.** Where should the habits be persisted between page loads?

**Answer:** localStorage (or a backend), saved from a watcher and read on startup

Guard against bad stored JSON.

**Q39.** How do you structure the project's components?

**Answer:** A list component, an item component and a form, with state in a composable

Small units are easier to test.

**Q40.** How do you test the logic without the UI?

**Answer:** Extract it into plain functions or composables and unit test them

Fast tests.

## Level 2 · Intermediate

### [01 · Composables](level-2/01-composables.md)

**Q41.** What does this composable print?

```js
function useCounter(start = 0) { const n = ref(start); const inc = () => n.value++; const double = computed(() => n.value * 2); return { n, inc, double } }
const a = useCounter(); const b = useCounter(10)
a.inc(); a.inc(); b.inc()
console.log(a.n.value, a.double.value, b.n.value)
```

**Answer:** 2 4 11

Each call creates independent reactive state. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q42.** What is a composable?

**Answer:** A function that encapsulates reusable stateful logic using the Composition API

By convention named useSomething.

**Q43.** What should composables return?

**Answer:** Refs and functions (not destructured plain values) so reactivity is preserved

Return an object of refs.

**Q44.** Where can composables be called?

**Answer:** Synchronously inside setup or other composables so lifecycle hooks attach to the component

Context matters.

### [02 · Provide &amp; Inject](level-2/02-provide-inject.md)

**Q45.** What does provide/inject give a deep child?

```js
const Leaf = defineComponent({ setup() { const theme = inject('theme', 'light'); return { theme } }, template: '<i>{{ theme }}</i>' })
const Mid = defineComponent({ components: { Leaf }, template: '<Leaf />' })
const app = createSSRApp({ components: { Mid }, setup() { provide('theme', 'dark') }, template: '<Mid />' })
console.log(await renderToString(app))
```

**Answer:** &lt;i&gt;dark&lt;/i&gt;

Values flow down to any descendant without prop drilling; the default applies only when nothing is provided. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q46.** What is a downside of overusing provide/inject?

**Answer:** Implicit dependencies are harder to trace than explicit props

Use for cross-cutting context such as theme.

**Q47.** How do you keep injected state reactive?

**Answer:** Provide a ref or reactive object, not a plain value

Prefer readonly for consumers.

**Q48.** What does a Symbol key avoid?

**Answer:** Name collisions between provided values

Use an InjectionKey in TypeScript.

### [03 · Vue Router Basics](level-2/03-vue-router-basics.md)

**Q49.** What does Vue Router map?

**Answer:** URL paths to components (routes)

createRouter({ history, routes }).

**Q50.** Which component renders the matched route?

**Answer:** &lt;RouterView&gt;

&lt;RouterLink&gt; creates navigation links.

**Q51.** How do you read route params in a component?

**Answer:** useRoute().params (and useRouter() to navigate)

Params are strings.

**Q52.** What does history mode (createWebHistory) require from the server?

**Answer:** A fallback to index.html for unknown paths so deep links work

Otherwise refresh gives 404.

### [04 · Router in Depth: Guards, Meta, File Routes](level-2/04-router-in-depth.md)

**Q53.** What are navigation guards for?

**Answer:** Controlling navigation, for example redirecting unauthenticated users

beforeEach and per-route guards.

**Q54.** How do you lazy-load a route component?

**Answer:** component: () =&gt; import('./views/About.vue')

Code splitting per route.

**Q55.** What are nested routes?

**Answer:** Child routes rendered inside a parent's &lt;RouterView&gt;

Layouts.

**Q56.** Why use named routes?

**Answer:** Navigate by name and params without hard-coded paths

Refactor-safe.

### [05 · Pinia Fundamentals](level-2/05-pinia-fundamentals.md)

**Q57.** What does this Pinia setup store print?

```js
import { createPinia, defineStore, setActivePinia } from 'pinia'
setActivePinia(createPinia())
const useCart = defineStore('cart', () => { const items = ref([]); const total = computed(() => items.value.reduce((s, i) => s + i.price * i.qty, 0)); function add(i) { items.value.push(i) }; return { items, total, add } })
const cart = useCart()
cart.add({ price: 5, qty: 2 }); cart.add({ price: 3, qty: 1 })
console.log(cart.total, useCart() === cart)
```

**Answer:** 13 true

A store is a singleton per Pinia instance; computed getters derive from state. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q58.** What is Pinia?

**Answer:** The official state management library for Vue

Replaces Vuex.

**Q59.** What belongs in a store?

**Answer:** State shared across components, with actions that change it

Keep component-local state local.

**Q60.** Why not mutate store state from many places?

**Answer:** Centralising changes in actions makes behaviour easier to trace and test

Use devtools to inspect.

### [06 · Pinia in Depth](level-2/06-pinia-in-depth.md)

**Q61.** What does $patch do on a options-style store?

```js
import { createPinia, defineStore, setActivePinia } from 'pinia'
setActivePinia(createPinia())
const useS = defineStore('s', { state: () => ({ a: 1, b: 2 }), getters: { sum: s => s.a + s.b }, actions: { bump() { this.a++ } } })
const s = useS()
s.$patch({ b: 10 }); s.bump()
console.log(s.sum, JSON.stringify(s.$state))
s.$reset()
console.log(s.sum)
```

**Answer:** 12 {"a":2,"b":10}
3

$patch batches updates; $reset restores the initial state (options stores). Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q62.** How do you keep destructured store state reactive?

**Answer:** Use storeToRefs

Actions can be destructured directly.

**Q63.** How do you persist a store?

**Answer:** A plugin or manual subscribe that saves to localStorage and rehydrates on start

Mind sensitive data.

**Q64.** How do you test a store?

**Answer:** Create a fresh Pinia per test and call actions directly

Isolated and fast.

### [07 · Async Data &amp; API Calls](level-2/07-async-data.md)

**Q65.** Which pattern fetches data in a composable?

**Answer:** Expose data, error and loading refs and update them around an async call

Handle all three states.

**Q66.** What should you do when a new request starts before an old one finishes?

**Answer:** Ignore or abort the stale response (AbortController or a request token)

Avoid race conditions.

**Q67.** What is Suspense for?

**Answer:** Showing fallback content while async components or setup() with await resolve

Still experimental in some versions.

**Q68.** Why show loading and error states?

**Answer:** Users need feedback; blank screens look broken

Include retry.

### [08 · Form Validation](level-2/08-form-validation.md)

**Q69.** What does this validation composable return?

```js
const rules = { required: v => !!v.trim() || 'Required', min: n => v => v.length >= n || `Min ${n}` }
function validate(value, fns) { const errs = fns.map(f => f(value)).filter(r => r !== true); return errs }
console.log(validate('', [rules.required, rules.min(3)]), validate('abcd', [rules.required, rules.min(3)]))
```

**Answer:** [ 'Required', 'Min 3' ] []

Validation rules return true or a message. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q70.** When should errors be shown?

**Answer:** After the field is touched or on submit, not while the user is still typing the first character

Avoid noisy forms.

**Q71.** What libraries help with forms in Vue?

**Answer:** VeeValidate or FormKit with schema validators like Zod or Yup

Share schemas with the server.

**Q72.** How should errors be announced?

**Answer:** Linked to inputs with aria-describedby and aria-invalid for assistive tech

Accessibility.

### [09 · TypeScript with Vue](level-2/09-typescript-with-vue.md)

**Q73.** How do you type props with &lt;script setup lang="ts"&gt;?

**Answer:** defineProps&lt;{ title: string; count?: number }&gt;()

Compile-time types.

**Q74.** What type is a template ref to a component?

**Answer:** InstanceType&lt;typeof Component&gt; or a helper such as useTemplateRef

Use null as the initial value.

**Q75.** What tool type-checks .vue files?

**Answer:** vue-tsc

Run it in CI.

**Q76.** What do generics in components allow?

**Answer:** Reusable typed components such as a typed list or table

&lt;script setup generic="T"&gt;.

### [10 · Project — A Recipe Book](level-2/10-project-recipe-book.md)

**Q77.** What does the recipe book project combine?

**Answer:** Router, Pinia, async data, forms and composables in a multi-view app

Level 2 capstone.

**Q78.** How do you search recipes without hammering the API?

**Answer:** Debounce input and cancel stale requests

Cache results.

**Q79.** Where should favourites live?

**Answer:** In a Pinia store persisted to localStorage

Rehydrate on load.

**Q80.** How should the project be tested?

**Answer:** Unit tests for composables and stores and component tests for key views

Cover the logic.

## Level 3 · Advanced

### [01 · Reactivity in Depth](level-3/01-reactivity-in-depth.md)

**Q81.** What does shallowRef do compared to ref?

```js
const deep = ref({ n: 1 }); const shallow = shallowRef({ n: 1 })
const seen = []
watch(deep, () => seen.push('deep'), { deep: true })
watch(shallow, () => seen.push('shallow'), { deep: true })
deep.value.n++; shallow.value.n++
await nextTick()
console.log(seen.join(','))
shallow.value = { n: 5 }
await nextTick()
console.log(seen.join(','))
```

**Answer:** deep
deep,shallow

A shallowRef tracks only replacing .value, not mutations inside the object. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q82.** What does effectScope let you do?

```js
const scope = effectScope()
const n = ref(0); const log = []
scope.run(() => { watchEffect(() => log.push(n.value)) })
await nextTick()
n.value = 1; await nextTick()
scope.stop()
n.value = 2; await nextTick()
console.log(log.join(','))
```

**Answer:** 0,1

Stopping the scope disposes all effects created inside it. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q83.** What does readonly guard against?

```js
const state = reactive({ a: 1 }); const ro = readonly(state)
ro.a = 99
state.a = 2
console.log(ro.a, state.a)
```

**Answer:** 2 2

Writes through a readonly proxy are ignored (with a warning in dev), while the source stays writable. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q84.** How does Vue 3 track reactivity?

**Answer:** With Proxies that record which effects read each property and trigger them on change

Dependencies are tracked automatically.

### [02 · The Rendering Mechanism &amp; Render Functions](level-3/02-rendering-mechanism.md)

**Q85.** What is the virtual DOM?

**Answer:** A lightweight JavaScript tree of vnodes that Vue diffs to compute minimal DOM updates

Templates compile to render functions.

**Q86.** What does the compiler optimise with patch flags?

**Answer:** It marks dynamic parts at compile time so updates skip static content

Static hoisting.

**Q87.** What does a render function with h() produce?

```js
const app = createSSRApp({ render() { return h('ul', { class: 'list' }, [1,2].map(n => h('li', { key: n }, `Item ${n}`))) } })
console.log(await renderToString(app))
```

**Answer:** &lt;ul class="list"&gt;&lt;li&gt;Item 1&lt;/li&gt;&lt;li&gt;Item 2&lt;/li&gt;&lt;/ul&gt;

Templates compile to calls like this. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q88.** Why prefer templates over render functions?

**Answer:** The compiler can apply optimisations that hand-written render functions cannot

Use h() for dynamic components.

### [03 · Custom Directives &amp; Plugins](level-3/03-directives-plugins.md)

**Q89.** What is a custom directive for?

**Answer:** Low-level direct DOM access reused across components (for example v-focus)

Use composables for logic.

**Q90.** What does app.use(plugin) do?

**Answer:** Installs a plugin that can add global components, directives or provide values

Plugins have an install function.

**Q91.** When should you avoid a plugin?

**Answer:** When a composable or component would do, since globals make dependencies implicit

Keep scope small.

**Q92.** What lifecycle hooks do directives have?

**Answer:** created, beforeMount, mounted, beforeUpdate, updated, beforeUnmount, unmounted

Receive the element and binding.

### [04 · Teleport, KeepAlive, Transition &amp; Suspense](level-3/04-built-in-components.md)

**Q93.** What does &lt;Teleport&gt; do?

**Answer:** Renders its content to another place in the DOM, such as &lt;body&gt;, while keeping logical component hierarchy

Modals and tooltips.

**Q94.** What does &lt;KeepAlive&gt; do?

**Answer:** Caches component instances when switching so state is preserved

Use include and max.

**Q95.** What does &lt;Suspense&gt; coordinate?

**Answer:** Async dependencies in a subtree, showing a fallback until they resolve

Handle errors separately.

**Q96.** What does &lt;component :is="x"&gt; do?

**Answer:** Renders a dynamic component chosen at runtime

Use markRaw for component objects in reactive state.

### [05 · Async Components &amp; Code Splitting](level-3/05-async-components-splitting.md)

**Q97.** What does defineAsyncComponent do?

**Answer:** Loads a component lazily when it is first rendered

Add loading and error components.

**Q98.** How does code splitting help?

**Answer:** Users download only the code needed for the current view

Dynamic import() creates chunks.

**Q99.** Where are good split points?

**Answer:** Routes and heavy rarely used components (charts, editors)

Measure bundle size.

**Q100.** What should you show while an async component loads?

**Answer:** A skeleton or spinner with a delay to avoid flicker

loadingComponent and delay.

### [06 · Performance](level-3/06-performance.md)

**Q101.** What does v-memo / computed memoisation save here?

```js
let runs = 0
const list = ref([1,2,3])
const evens = computed(() => { runs++; return list.value.filter(n => n % 2 === 0) })
evens.value; evens.value
list.value = [...list.value, 4]
evens.value; evens.value
console.log(runs, evens.value)
```

**Answer:** 2 [ 2, 4 ]

Computed values cache until a dependency changes. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q102.** What reduces unnecessary re-renders of a big list?

**Answer:** Stable keys, splitting rows into components and avoiding reactive wrappers for huge static data (shallowRef/markRaw)

Measure first.

**Q103.** When use virtual scrolling?

**Answer:** For very long lists where rendering every row is too slow

Libraries such as vue-virtual-scroller.

**Q104.** What tool shows what is slow?

**Answer:** Vue Devtools performance tab and the browser Performance panel

Measure before optimising.

### [07 · Testing with Vitest &amp; Vue Test Utils](level-3/07-testing.md)

**Q105.** What is the recommended test runner for Vue projects?

**Answer:** Vitest, with Vue Test Utils for mounting components

Shares Vite config.

**Q106.** What should component tests assert on?

**Answer:** What the user sees and does (text, roles, events), not implementation details

Testing Library encourages this.

**Q107.** Why mock the network in tests?

**Answer:** Tests stay fast and deterministic

MSW is popular.

**Q108.** When use end-to-end tests?

**Answer:** For a few critical user journeys across the real app (Playwright, Cypress)

They are slower.

### [08 · Accessibility in Vue Apps](level-3/08-accessibility.md)

**Q109.** How do you make custom components keyboard accessible?

**Answer:** Use native elements (button, a) and manage focus and ARIA roles for custom widgets

Test with keyboard only.

**Q110.** What should route changes do for screen readers?

**Answer:** Move focus or announce the new page title

SPA navigation doesn't reload.

**Q111.** What does aria-live do?

**Answer:** Announces dynamic content updates to assistive technology

Use sparingly.

**Q112.** How do you check accessibility continuously?

**Answer:** Automated checks (axe) in CI plus manual keyboard and screen-reader testing

Both needed.

### [09 · Error Handling &amp; Security](level-3/09-errors-security.md)

**Q113.** What does app.config.errorHandler catch?

**Answer:** Errors thrown in component render, lifecycle and event handlers

Send to monitoring.

**Q114.** Why is v-html dangerous with user input?

**Answer:** It inserts raw HTML, enabling XSS; sanitise or avoid

Prefer text interpolation.

**Q115.** What does text interpolation do with HTML in user input?

```js
const app = createSSRApp({ data: () => ({ input: '<img src=x onerror=alert(1)>' }), template: '<p>{{ input }}</p>' })
console.log(await renderToString(app))
```

**Answer:** &lt;p&gt;&amp;lt;img src=x onerror=alert(1)&amp;gt;&lt;/p&gt;

Mustache escapes HTML, which is why it is safe by default. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q116.** Why not store tokens in localStorage if avoidable?

**Answer:** Any XSS can read them; prefer httpOnly cookies

Defence in depth.

### [10 · Project — A Kanban Board](level-3/10-project-kanban.md)

**Q117.** What does the kanban project practise?

**Answer:** Drag and drop, derived state, persistence and component composition

Level 3 capstone.

**Q118.** How do you make drag and drop accessible?

**Answer:** Provide keyboard alternatives (move buttons) and announce changes

Test with keyboard.

**Q119.** Where should the board state live?

**Answer:** In a store with actions that move cards immutably and persist

Single source of truth.

**Q120.** How do you keep ordering stable?

**Answer:** Store order explicitly (position or ids array) rather than relying on DOM order

Reorder in state.

## Level 4 · Master

### [01 · Server-Side Rendering from Scratch](level-4/01-ssr-from-scratch.md)

**Q121.** What does server-side rendering produce for a component with async state?

```js
const app = createSSRApp({ data: () => ({ items: ['x','y'] }), template: '<ul><li v-for="i in items">{{ i }}</li></ul>' })
const html = await renderToString(app)
console.log(html)
console.log(html.startsWith('<ul>'))
```

**Answer:** &lt;ul&gt;&lt;li&gt;x&lt;/li&gt;&lt;li&gt;y&lt;/li&gt;&lt;/ul&gt;
true

SSR returns complete HTML that the client then hydrates. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q122.** What is hydration?

**Answer:** The client attaching event listeners and state to server-rendered HTML

Mismatches cause warnings.

**Q123.** What causes hydration mismatches?

**Answer:** Server and client rendering different output, for example using Date.now() or window during render

Defer client-only work to onMounted.

**Q124.** Why share state carefully on the server?

**Answer:** Module-level singletons leak between requests; create a new app and store per request

Isolation.

### [02 · Nuxt Overview](level-4/02-nuxt-overview.md)

**Q125.** What does Nuxt add to Vue?

**Answer:** Conventions and tooling for routing by file structure, SSR, data fetching and deployment

File-based routing.

**Q126.** What does useFetch do in Nuxt?

**Answer:** Fetches data on the server and reuses it on the client during hydration

Avoids double fetch.

**Q127.** What are Nuxt's rendering modes?

**Answer:** SSR, static generation (SSG), SPA and hybrid per route

Choose per page.

**Q128.** When might plain Vue be enough?

**Answer:** For client-only apps behind login with no SEO needs

Choose the lightest tool.

### [03 · Vite &amp; Production Builds](level-4/03-vite-production-builds.md)

**Q129.** What does vite build produce?

**Answer:** Optimised, minified, code-split static assets with hashed filenames

Use for production.

**Q130.** Why hashed filenames?

**Answer:** Browsers can cache assets forever and updates change the name

Cache busting.

**Q131.** What does rollup-plugin-visualizer show?

**Answer:** What contributes to bundle size

Find heavy dependencies.

**Q132.** How are environment variables exposed?

**Answer:** Only those prefixed with VITE_ reach client code, and they are public

Never put secrets in them.

### [04 · Component Libraries &amp; Design Systems](level-4/04-component-libraries.md)

**Q133.** What is a good way to build a component library?

**Answer:** Vite library mode with typed props, documentation (Storybook) and tests

Publish ESM and types.

**Q134.** What belongs in a library's dependencies?

**Answer:** Vue as a peer dependency, not bundled

Avoid duplicate Vue copies.

**Q135.** How should styles be shipped?

**Answer:** Scoped or CSS variables with documented theming hooks

Allow overrides.

**Q136.** How do you version a library?

**Answer:** Semantic versioning with a changelog

Communicate breaking changes.

### [05 · Large-App Architecture](level-4/05-large-app-architecture.md)

**Q137.** How do you structure a large Vue app?

**Answer:** Feature-based modules with their own components, composables, stores and routes

Scale by feature.

**Q138.** What keeps coupling low?

**Answer:** Clear public APIs per feature and no deep imports across features

Lint boundaries.

**Q139.** Where should API calls live?

**Answer:** In a service layer or composables, not scattered through components

Easier to mock and change.

**Q140.** What do architecture decision records capture?

**Answer:** Why key decisions were made, and their trade-offs

Help future maintainers.

### [06 · Migrating from Vue 2 &amp; Staying Current](level-4/06-migration-staying-current.md)

**Q141.** What changed from Vue 2 to Vue 3?

**Answer:** Composition API, Proxy-based reactivity, fragments, Teleport and better TypeScript support

Vue 2 is end of life.

**Q142.** How do you stay current safely?

**Answer:** Read release notes, upgrade regularly in small steps and keep tests green

Small upgrades are easier.

**Q143.** What does the migration build help with?

**Answer:** Running Vue 2 code on Vue 3 with compatibility warnings

A transitional tool.

**Q144.** Why track deprecations?

**Answer:** They signal future removals; fix them early

Treat warnings as work items.

### [07 · Internationalization](level-4/07-i18n.md)

**Q145.** What does vue-i18n manage?

**Answer:** Translations, locale switching, pluralisation and number/date formatting

Lazy-load locale files.

**Q146.** Why not concatenate translated strings?

**Answer:** Word order differs by language; use placeholders and plural rules

ICU-style messages.

**Q147.** What else should i18n consider?

**Answer:** Right-to-left layouts, date and number formats and text expansion

Test with long strings.

**Q148.** How do you handle missing translations?

**Answer:** Fall back to a default locale and log the missing keys

Track coverage.

### [08 · Deployment](level-4/08-deployment.md)

**Q149.** What does an SPA deployment need?

**Answer:** Static hosting with a fallback to index.html for client-side routes

Cache hashed assets, not index.html.

**Q150.** What cache headers suit hashed assets?

**Answer:** Long-lived immutable caching; keep index.html short-lived

Safe updates.

**Q151.** What does a CDN add?

**Answer:** Edge caching close to users for faster loads

Purge on deploy.

**Q152.** What belongs in a deploy pipeline?

**Answer:** Install, lint, test, build, deploy and a smoke test with easy rollback

Automate everything.

### [09 · Observability &amp; Production Readiness](level-4/09-production-readiness.md)

**Q153.** What does production readiness include?

**Answer:** Error monitoring, performance budgets, accessibility checks, security headers and runbooks

Beyond working code.

**Q154.** How do you track real-user errors?

**Answer:** Send errors from app.config.errorHandler to a monitoring service with source maps

Release tagging.

**Q155.** What are performance budgets?

**Answer:** Limits on bundle size and load metrics enforced in CI

Prevent regressions.

**Q156.** What is a security header example?

**Answer:** Content-Security-Policy restricting where scripts load from

Mitigates XSS.

### [10 · Capstone — A Production Vue App](level-4/10-capstone.md)

**Q157.** What does the capstone combine?

**Answer:** Routing, state, async data, testing, accessibility, performance and deployment

Show it all.

**Q158.** What proves quality?

**Answer:** Passing CI with tests, accessibility audit, budgets and documentation

Evidence.

**Q159.** What belongs in the README?

**Answer:** Setup, scripts, architecture, decisions and deployment steps

Handover.

**Q160.** How do you handle residual risks?

**Answer:** Document them and accept, mitigate or transfer

Risk register.

## Mixed questions across all levels

**Q161.** What does this reactive code print?

```js
const a = ref(1)
const b = computed(() => a.value + 1)
const log = []
watch(b, (v, o) => log.push(`${o}->${v}`))
a.value = 2; a.value = 3
await nextTick()
console.log(log.join(','), b.value)
```

**Answer:** 2-&gt;4 4

Watchers are batched: two synchronous changes produce one callback with the final value. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q162.** What does destructuring a reactive object do?

```js
const s = reactive({ n: 1 })
let { n } = s
s.n = 2
const { n: r } = toRefs(s)
s.n = 3
console.log(n, r.value)
```

**Answer:** 1 3

Plain destructuring copies the value; toRefs keeps a live link. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q163.** What does this component render with a scoped default?

```js
const Item = defineComponent({ props: { label: { type: String, default: 'none' } }, template: '<i>{{ label }}</i>' })
const app = createSSRApp({ components: { Item }, template: '<Item /><Item label="x" />' })
console.log(await renderToString(app))
```

**Answer:** &lt;i&gt;none&lt;/i&gt;&lt;i&gt;x&lt;/i&gt;

Props can declare defaults. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q164.** What does Vue's text interpolation do with markup?

```js
const app = createSSRApp({ data: () => ({ s: '<b>hi</b>' }), template: '<p>{{ s }}</p><p v-html="s"></p>' })
console.log(await renderToString(app))
```

**Answer:** &lt;p&gt;&amp;lt;b&amp;gt;hi&amp;lt;/b&amp;gt;&lt;/p&gt;&lt;p&gt;&lt;b&gt;hi&lt;/b&gt;&lt;/p&gt;

Mustache escapes; v-html inserts raw HTML, so never use it with untrusted input. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q165.** What does shallowRef track?

```js
const s = shallowRef({ n: 1 })
const seen = []
watch(s, () => seen.push('changed'))
s.value.n = 2
await nextTick()
seen.push(seen.length)
s.value = { n: 3 }
await nextTick()
console.log(seen.join(','))
```

**Answer:** 0,changed

Only replacing .value triggers a shallowRef. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability).

**Q166.** A parent passes an object as a prop and the child edits a field. What is wrong?

**Answer:** Props are one-way; mutating them hides data flow, so emit an event or copy to local state

Mutating nested props even works silently, which is why it is discouraged.

**Q167.** Why do v-for lists need a stable :key?

**Answer:** So Vue can reuse and reorder elements correctly instead of patching them in place

Use unique ids, not the array index, if items can move.

**Q168.** When is computed better than watch?

**Answer:** When you derive a value from other state, because it caches and stays declarative

Use watch for side effects.

**Q169.** What does &lt;script setup&gt; compile to?

**Answer:** Component setup code where top-level bindings are exposed to the template automatically

Less boilerplate.

**Q170.** What is the right place to fetch data for a view?

**Answer:** A composable returning data, error and loading refs, called from setup

Handle all three states.

**Q171.** A route works when navigated to but 404s on refresh in production. Why?

**Answer:** The server has no fallback to index.html for history-mode routes

Configure the host fallback.

**Q172.** What is Pinia's role?

**Answer:** Shared state outside components, with actions and getters

Keep local state local.

**Q173.** How do you test a Pinia store in isolation?

**Answer:** Create a fresh Pinia per test and call actions directly

Fast, deterministic.

**Q174.** What does hydration mean in SSR?

**Answer:** The client attaching reactivity and listeners to server-rendered HTML

Mismatches produce warnings.

**Q175.** What commonly causes hydration mismatches?

**Answer:** Rendering different output on server and client, for example Date.now() or window during render

Defer client-only work to onMounted.

**Q176.** Why is a module-level store dangerous in SSR?

**Answer:** State leaks between requests; create a new app and store per request

Isolation per request.

**Q177.** Which assets can be cached forever in production?

**Answer:** Hashed build assets (the file name changes with content)

Keep index.html short-lived.

**Q178.** How do you secure a Vue app against XSS?

**Answer:** Avoid v-html with user input, sanitise when needed, and set a Content-Security-Policy

Mustache escaping helps by default.

**Q179.** What proves a Vue app is production-ready?

**Answer:** CI with tests and accessibility checks, performance budgets, error monitoring and a rollback plan

Evidence.

## Practical coding questions

### A habit-tracker composable with persistence

Write useHabits, a composable that restores habits from a storage object, adds (with trimming and duplicate checks), toggles days immutably, summarises today's progress, computes streaks and saves changes with a deep watcher. A fake storage object stands in for localStorage so it runs in Node; only Vue's reactivity package is used, no DOM or components.

Starter code:

```
import { ref, computed, watch, nextTick } from 'vue'

// a fake localStorage so the script runs anywhere
const storage = { data: {}, writes: 0, getItem(k) { return this.data[k] ?? null }, setItem(k, v) { this.data[k] = v; this.writes++ } }

// TODO: export function useHabits(store, key = 'habits')
//   restore habits from store.getItem(key) (JSON; fall back to [] on bad data), keep them in a ref, compute nextId
//   add(name): trim, reject empty or case-insensitive duplicates (return false/true), push { id, name, days: [] }
//   toggle(id, day): add or remove the day immutably (return false for an unknown id)
//   deleteHabit(id)
//   summary (computed): "<done>/<total> done today" counting habits whose days include '2026-03-05'
//   streak(habit, today): consecutive days ending today (use UTC dates)
//   a deep watcher that saves JSON to the store (the watcher batches synchronous changes into one write)

const log = []
const app1 = useHabits(storage)
log.push(['add Read', app1.add('Read')], ['add read (dup)', app1.add('read')], ['add Run', app1.add('  Run ')], ['add empty', app1.add('   ')])
for (const d of ['2026-03-03', '2026-03-04', '2026-03-05']) app1.toggle(1, d)
app1.toggle(2, '2026-03-05')
app1.toggle(2, '2026-03-05')
await nextTick()
console.log(log.map(([k, v]) => `${k}: ${v}`).join('\n'))
console.log(app1.summary.value, '| streak(Read) =', app1.streak(app1.habits.value[0], '2026-03-05'), '| streak(Run) =', app1.streak(app1.habits.value[1], '2026-03-05'))
console.log('storage writes (batched by the watcher):', storage.writes)

const app2 = useHabits(storage)
console.log('restored:', app2.habits.value.map(h => `${h.name}(${h.days.length})`).join(', '))
app2.deleteHabit(2)
await nextTick()
console.log('after delete:', JSON.parse(storage.getItem('habits')).map(h => h.name).join(','))
const broken = { getItem: () => '{not json', setItem() {} }
console.log('corrupt storage ->', useHabits(broken).habits.value.length, 'habits')
```

Expected output:

```text
add Read: true
add read (dup): false
add Run: true
add empty: false
1/2 done today | streak(Read) = 3 | streak(Run) = 0
storage writes (batched by the watcher): 1
restored: Read(3), Run(0)
after delete: Read
corrupt storage -> 0 habits
```

??? success "Show a solution"

    ```
    import { ref, computed, watch, nextTick } from 'vue'

    // a fake localStorage so the script runs anywhere
    const storage = { data: {}, writes: 0, getItem(k) { return this.data[k] ?? null }, setItem(k, v) { this.data[k] = v; this.writes++ } }

    export function useHabits(store, key = 'habits') {
      let initial = []
      try { initial = JSON.parse(store.getItem(key)) ?? [] } catch { initial = [] }
      const habits = ref(initial)
      let nextId = habits.value.reduce((m, h) => Math.max(m, h.id), 0) + 1

      const doneCount = computed(() => habits.value.filter(h => h.days.includes('2026-03-05')).length)
      const summary = computed(() => `${doneCount.value}/${habits.value.length} done today`)
      const streak = (h, today) => {
        let n = 0
        const d = new Date(today + 'T00:00:00Z')
        while (h.days.includes(d.toISOString().slice(0, 10))) { n++; d.setUTCDate(d.getUTCDate() - 1) }
        return n
      }

      function add(name) {
        const trimmed = name.trim()
        if (!trimmed || habits.value.some(h => h.name.toLowerCase() === trimmed.toLowerCase())) return false
        habits.value.push({ id: nextId++, name: trimmed, days: [] })
        return true
      }
      function toggle(id, day) {
        const h = habits.value.find(h => h.id === id)
        if (!h) return false
        h.days = h.days.includes(day) ? h.days.filter(d => d !== day) : [...h.days, day]
        return true
      }
      function deleteHabit(id) { habits.value = habits.value.filter(h => h.id !== id) }

      watch(habits, v => store.setItem(key, JSON.stringify(v)), { deep: true })
      return { habits, summary, streak, add, toggle, deleteHabit }
    }

    const log = []
    const app1 = useHabits(storage)
    log.push(['add Read', app1.add('Read')], ['add read (dup)', app1.add('read')], ['add Run', app1.add('  Run ')], ['add empty', app1.add('   ')])
    for (const d of ['2026-03-03', '2026-03-04', '2026-03-05']) app1.toggle(1, d)
    app1.toggle(2, '2026-03-05')
    app1.toggle(2, '2026-03-05')
    await nextTick()
    console.log(log.map(([k, v]) => `${k}: ${v}`).join('\n'))
    console.log(app1.summary.value, '| streak(Read) =', app1.streak(app1.habits.value[0], '2026-03-05'), '| streak(Run) =', app1.streak(app1.habits.value[1], '2026-03-05'))
    console.log('storage writes (batched by the watcher):', storage.writes)

    const app2 = useHabits(storage)
    console.log('restored:', app2.habits.value.map(h => `${h.name}(${h.days.length})`).join(', '))
    app2.deleteHabit(2)
    await nextTick()
    console.log('after delete:', JSON.parse(storage.getItem('habits')).map(h => h.name).join(','))
    const broken = { getItem: () => '{not json', setItem() {} }
    console.log('corrupt storage ->', useHabits(broken).habits.value.length, 'habits')
    ```

    Because the watcher is flushed asynchronously, the five synchronous edits cause a single storage write. Restoring runs through try/catch so corrupt data falls back to an empty list. Verified by running it with Vue 3.5 in Node.

### Server-render a product list with props, slots and provide/inject

Build ProductList, ProductCard and PriceView components and render the page to HTML on the server with createSSRApp and renderToString. Use props with defaults, named slots with fallback content, a computed filter and sort, and provide/inject for a money formatter. The hydration comment markers are stripped from the printed HTML; note the id attribute, which falls through from v-bind because it is not a declared prop.

Starter code:

```
import { createSSRApp, defineComponent, computed, provide, inject } from 'vue'
import { renderToString as _render } from 'vue/server-renderer'
const renderToString = async app => (await _render(app)).replace(/<!--[\[\]]-->/g, '')

// TODO: PriceView (prop cents, injects 'money' with a default formatter, renders <b class="price">)
// TODO: ProductCard (props name, cents, stock default 0; slots "title" and "badge" with fallbacks; class sold when stock is 0)
// TODO: ProductList (props products, query, sort; computed filtered + sorted list; empty-state paragraph)
// Hint: v-bind="p" on the card passes every product field, so non-prop fields such as id fall through as attributes

const products = [
  { id: 1, name: 'Notebook', cents: 450, stock: 12 },
  { id: 2, name: 'Pen', cents: 150, stock: 0 },
  { id: 3, name: 'Notepad', cents: 300, stock: 3 },
]
const money = c => `$${(c / 100).toFixed(2)}`

async function page(props) {
  const app = createSSRApp({
    components: { ProductList },
    setup() { provide('money', money); return { props } },
    template: '<ProductList v-bind="props" />',
  })
  return renderToString(app)
}

console.log(await page({ products }))
console.log(await page({ products, query: 'note', sort: 'price' }))
console.log(await page({ products, query: 'zzz' }))
```

Expected output:

```text
<ul><li class="" id="1">Notebook <b class="price">$4.50</b><!----></li><li class="" id="3">Notepad <b class="price">$3.00</b><!----></li><li class="sold" id="2">Pen <b class="price">$1.50</b><span class="badge">gone</span></li></ul>
<ul><li class="" id="3">Notepad <b class="price">$3.00</b><!----></li><li class="" id="1">Notebook <b class="price">$4.50</b><!----></li></ul>
<p class="empty">No products match &quot;zzz&quot;</p>
```

??? success "Show a solution"

    ```
    import { createSSRApp, defineComponent, computed, provide, inject } from 'vue'
    import { renderToString as _render } from 'vue/server-renderer'
    const renderToString = async app => (await _render(app)).replace(/<!--[\[\]]-->/g, '')

    const PriceView = defineComponent({
      props: { cents: { type: Number, required: true } },
      setup(props) {
        const fmt = inject('money', c => String(c))
        const text = computed(() => fmt(props.cents))
        return { text }
      },
      template: '<b class="price">{{ text }}</b>'
    })

    const ProductCard = defineComponent({
      components: { PriceView },
      props: { name: String, cents: Number, stock: { type: Number, default: 0 } },
      template: `<li :class="{ sold: stock === 0 }"><slot name="title">{{ name }}</slot> <PriceView :cents="cents" /><slot v-if="stock === 0" name="badge"><em>sold out</em></slot></li>`
    })

    const ProductList = defineComponent({
      components: { ProductCard },
      props: { products: Array, query: { type: String, default: '' }, sort: { type: String, default: 'name' } },
      setup(props) {
        const visible = computed(() => {
          const q = props.query.toLowerCase()
          const list = props.products.filter(p => p.name.toLowerCase().includes(q))
          return [...list].sort((a, b) => (props.sort === 'price' ? a.cents - b.cents : a.name.localeCompare(b.name)))
        })
        return { visible }
      },
      template: `<ul v-if="visible.length"><ProductCard v-for="p in visible" :key="p.id" v-bind="p"><template #badge><span class="badge">gone</span></template></ProductCard></ul><p v-else class="empty">No products match "{{ query }}"</p>`
    })

    const products = [
      { id: 1, name: 'Notebook', cents: 450, stock: 12 },
      { id: 2, name: 'Pen', cents: 150, stock: 0 },
      { id: 3, name: 'Notepad', cents: 300, stock: 3 },
    ]
    const money = c => `$${(c / 100).toFixed(2)}`

    async function page(props) {
      const app = createSSRApp({
        components: { ProductList },
        setup() { provide('money', money); return { props } },
        template: '<ProductList v-bind="props" />',
      })
      return renderToString(app)
    }

    console.log(await page({ products }))
    console.log(await page({ products, query: 'note', sort: 'price' }))
    console.log(await page({ products, query: 'zzz' }))
    ```

    v-bind="p" passes every product field; declared props are consumed and the undeclared id becomes an attribute on the card's root element (attribute fallthrough). The scoped badge slot replaces the fallback only for sold-out items, and an empty filter shows the empty state. Verified by running it with Vue 3.5 in Node.

---

More free practice: the [Vue.js Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
