---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Vue.js Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this reactive code print?\n```js\nconst a = ref(1)\nconst b = computed(() => a.value + 1)\nconst log = []\nwatch(b, (v, o) => log.push(`${o}->${v}`))\na.value = 2; a.value = 3\nawait nextTick()\nconsole.log(log.join(','), b.value)\n```\n",
    "options": [
      "2->3 3",
      "2->3,3->4 4",
      "2->4 4",
      "4 4"
    ],
    "correctIndex": 2,
    "explanation": "Watchers are batched: two synchronous changes produce one callback with the final value. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability)."
  },
  {
    "question": "What does destructuring a reactive object do?\n```js\nconst s = reactive({ n: 1 })\nlet { n } = s\ns.n = 2\nconst { n: r } = toRefs(s)\ns.n = 3\nconsole.log(n, r.value)\n```\n",
    "options": [
      "2 3",
      "3 3",
      "1 3",
      "1 1"
    ],
    "correctIndex": 2,
    "explanation": "Plain destructuring copies the value; toRefs keeps a live link. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability)."
  },
  {
    "question": "What does this component render with a scoped default?\n```js\nconst Item = defineComponent({ props: { label: { type: String, default: 'none' } }, template: '<i>{{ label }}<\/i>' })\nconst app = createSSRApp({ components: { Item }, template: '<Item /><Item label=\"x\" />' })\nconsole.log(await renderToString(app))\n```\n",
    "options": [
      "<i>none<\/i><i>none<\/i>",
      "<i><\/i><i>x<\/i>",
      "<i>none<\/i><i>x<\/i>",
      "<i>label<\/i><i>x<\/i>"
    ],
    "correctIndex": 2,
    "explanation": "Props can declare defaults. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability)."
  },
  {
    "question": "What does Vue's text interpolation do with markup?\n```js\nconst app = createSSRApp({ data: () => ({ s: '<b>hi<\/b>' }), template: '<p>{{ s }}<\/p><p v-html=\"s\"><\/p>' })\nconsole.log(await renderToString(app))\n```\n",
    "options": [
      "<p>&lt;b&gt;hi&lt;/b&gt;<\/p><p>&lt;b&gt;hi&lt;/b&gt;<\/p>",
      "<p><b>hi<\/b><\/p><p><b>hi<\/b><\/p>",
      "<p>&lt;b&gt;hi&lt;/b&gt;<\/p><p><b>hi<\/b><\/p>",
      "<p>hi<\/p><p>hi<\/p>"
    ],
    "correctIndex": 2,
    "explanation": "Mustache escapes; v-html inserts raw HTML, so never use it with untrusted input. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability)."
  },
  {
    "question": "What does shallowRef track?\n```js\nconst s = shallowRef({ n: 1 })\nconst seen = []\nwatch(s, () => seen.push('changed'))\ns.value.n = 2\nawait nextTick()\nseen.push(seen.length)\ns.value = { n: 3 }\nawait nextTick()\nconsole.log(seen.join(','))\n```\n",
    "options": [
      "changed,1",
      "0",
      "changed,changed",
      "0,changed"
    ],
    "correctIndex": 3,
    "explanation": "Only replacing .value triggers a shallowRef. Verified by running it with Vue 3.5 in Node (server renderer for templates; the SSR hydration comment markers are stripped for readability)."
  },
  {
    "question": "A parent passes an object as a prop and the child edits a field. What is wrong?",
    "options": [
      "It is faster",
      "Props are one-way; mutating them hides data flow, so emit an event or copy to local state",
      "Vue forbids objects",
      "Nothing"
    ],
    "correctIndex": 1,
    "explanation": "Mutating nested props even works silently, which is why it is discouraged."
  },
  {
    "question": "Why do v-for lists need a stable :key?",
    "options": [
      "Law",
      "Speed only",
      "So Vue can reuse and reorder elements correctly instead of patching them in place",
      "Styling"
    ],
    "correctIndex": 2,
    "explanation": "Use unique ids, not the array index, if items can move."
  },
  {
    "question": "When is computed better than watch?",
    "options": [
      "Logging",
      "When you derive a value from other state, because it caches and stays declarative",
      "Fetching",
      "Side effects"
    ],
    "correctIndex": 1,
    "explanation": "Use watch for side effects."
  },
  {
    "question": "What does <script setup> compile to?",
    "options": [
      "Component setup code where top-level bindings are exposed to the template automatically",
      "Options API",
      "A module system",
      "A class"
    ],
    "correctIndex": 0,
    "explanation": "Less boilerplate."
  },
  {
    "question": "What is the right place to fetch data for a view?",
    "options": [
      "CSS",
      "The template",
      "A directive",
      "A composable returning data, error and loading refs, called from setup"
    ],
    "correctIndex": 3,
    "explanation": "Handle all three states."
  },
  {
    "question": "A route works when navigated to but 404s on refresh in production. Why?",
    "options": [
      "Cache",
      "A router bug",
      "The server has no fallback to index.html for history-mode routes",
      "Vue bug"
    ],
    "correctIndex": 2,
    "explanation": "Configure the host fallback."
  },
  {
    "question": "What is Pinia's role?",
    "options": [
      "Build",
      "Testing",
      "Shared state outside components, with actions and getters",
      "Routing"
    ],
    "correctIndex": 2,
    "explanation": "Keep local state local."
  },
  {
    "question": "How do you test a Pinia store in isolation?",
    "options": [
      "Create a fresh Pinia per test and call actions directly",
      "Skip",
      "Mock Vue",
      "Mount the app"
    ],
    "correctIndex": 0,
    "explanation": "Fast, deterministic."
  },
  {
    "question": "What does hydration mean in SSR?",
    "options": [
      "Styling",
      "Caching",
      "The client attaching reactivity and listeners to server-rendered HTML",
      "Fetching"
    ],
    "correctIndex": 2,
    "explanation": "Mismatches produce warnings."
  },
  {
    "question": "What commonly causes hydration mismatches?",
    "options": [
      "Router",
      "CSS",
      "Rendering different output on server and client, for example Date.now() or window during render",
      "Props"
    ],
    "correctIndex": 2,
    "explanation": "Defer client-only work to onMounted."
  },
  {
    "question": "Why is a module-level store dangerous in SSR?",
    "options": [
      "Colour",
      "Speed",
      "State leaks between requests; create a new app and store per request",
      "Law"
    ],
    "correctIndex": 2,
    "explanation": "Isolation per request."
  },
  {
    "question": "Which assets can be cached forever in production?",
    "options": [
      "Nothing",
      "Hashed build assets (the file name changes with content)",
      "index.html",
      "Everything"
    ],
    "correctIndex": 1,
    "explanation": "Keep index.html short-lived."
  },
  {
    "question": "How do you secure a Vue app against XSS?",
    "options": [
      "Minify",
      "Avoid v-html with user input, sanitise when needed, and set a Content-Security-Policy",
      "Hide it",
      "Use v-show"
    ],
    "correctIndex": 1,
    "explanation": "Mustache escaping helps by default."
  },
  {
    "question": "What proves a Vue app is production-ready?",
    "options": [
      "It runs",
      "Few lines",
      "CI with tests and accessibility checks, performance budgets, error monitoring and a rollback plan",
      "Looks good"
    ],
    "correctIndex": 2,
    "explanation": "Evidence."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A habit-tracker composable with persistence

Write useHabits, a composable that restores habits from a storage object, adds (with trimming and duplicate checks), toggles days immutably, summarises today's progress, computes streaks and saves changes with a deep watcher. A fake storage object stands in for localStorage so it runs in Node; only Vue's reactivity package is used, no DOM or components.

**Starter code**

```javascript
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

**Expected output**

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

<details><summary>Show solution and explanation</summary>

<pre><code>import { ref, computed, watch, nextTick } from &#x27;vue&#x27;

// a fake localStorage so the script runs anywhere
const storage = { data: {}, writes: 0, getItem(k) { return this.data[k] ?? null }, setItem(k, v) { this.data[k] = v; this.writes++ } }

export function useHabits(store, key = &#x27;habits&#x27;) {
  let initial = []
  try { initial = JSON.parse(store.getItem(key)) ?? [] } catch { initial = [] }
  const habits = ref(initial)
  let nextId = habits.value.reduce((m, h) =&gt; Math.max(m, h.id), 0) + 1

  const doneCount = computed(() =&gt; habits.value.filter(h =&gt; h.days.includes(&#x27;2026-03-05&#x27;)).length)
  const summary = computed(() =&gt; `${doneCount.value}/${habits.value.length} done today`)
  const streak = (h, today) =&gt; {
    let n = 0
    const d = new Date(today + &#x27;T00:00:00Z&#x27;)
    while (h.days.includes(d.toISOString().slice(0, 10))) { n++; d.setUTCDate(d.getUTCDate() - 1) }
    return n
  }

  function add(name) {
    const trimmed = name.trim()
    if (!trimmed || habits.value.some(h =&gt; h.name.toLowerCase() === trimmed.toLowerCase())) return false
    habits.value.push({ id: nextId++, name: trimmed, days: [] })
    return true
  }
  function toggle(id, day) {
    const h = habits.value.find(h =&gt; h.id === id)
    if (!h) return false
    h.days = h.days.includes(day) ? h.days.filter(d =&gt; d !== day) : [...h.days, day]
    return true
  }
  function deleteHabit(id) { habits.value = habits.value.filter(h =&gt; h.id !== id) }

  watch(habits, v =&gt; store.setItem(key, JSON.stringify(v)), { deep: true })
  return { habits, summary, streak, add, toggle, deleteHabit }
}

const log = []
const app1 = useHabits(storage)
log.push([&#x27;add Read&#x27;, app1.add(&#x27;Read&#x27;)], [&#x27;add read (dup)&#x27;, app1.add(&#x27;read&#x27;)], [&#x27;add Run&#x27;, app1.add(&#x27;  Run &#x27;)], [&#x27;add empty&#x27;, app1.add(&#x27;   &#x27;)])
for (const d of [&#x27;2026-03-03&#x27;, &#x27;2026-03-04&#x27;, &#x27;2026-03-05&#x27;]) app1.toggle(1, d)
app1.toggle(2, &#x27;2026-03-05&#x27;)
app1.toggle(2, &#x27;2026-03-05&#x27;)
await nextTick()
console.log(log.map(([k, v]) =&gt; `${k}: ${v}`).join(&#x27;\n&#x27;))
console.log(app1.summary.value, &#x27;| streak(Read) =&#x27;, app1.streak(app1.habits.value[0], &#x27;2026-03-05&#x27;), &#x27;| streak(Run) =&#x27;, app1.streak(app1.habits.value[1], &#x27;2026-03-05&#x27;))
console.log(&#x27;storage writes (batched by the watcher):&#x27;, storage.writes)

const app2 = useHabits(storage)
console.log(&#x27;restored:&#x27;, app2.habits.value.map(h =&gt; `${h.name}(${h.days.length})`).join(&#x27;, &#x27;))
app2.deleteHabit(2)
await nextTick()
console.log(&#x27;after delete:&#x27;, JSON.parse(storage.getItem(&#x27;habits&#x27;)).map(h =&gt; h.name).join(&#x27;,&#x27;))
const broken = { getItem: () =&gt; &#x27;{not json&#x27;, setItem() {} }
console.log(&#x27;corrupt storage -&gt;&#x27;, useHabits(broken).habits.value.length, &#x27;habits&#x27;)
</code></pre>

<p>Because the watcher is flushed asynchronously, the five synchronous edits cause a single storage write. Restoring runs through try/catch so corrupt data falls back to an empty list. Verified by running it with Vue 3.5 in Node.</p>
</details>

### Server-render a product list with props, slots and provide/inject

Build ProductList, ProductCard and PriceView components and render the page to HTML on the server with createSSRApp and renderToString. Use props with defaults, named slots with fallback content, a computed filter and sort, and provide/inject for a money formatter. The hydration comment markers are stripped from the printed HTML; note the id attribute, which falls through from v-bind because it is not a declared prop.

**Starter code**

```javascript
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

**Expected output**

```text
<ul><li class="" id="1">Notebook <b class="price">$4.50</b><!----></li><li class="" id="3">Notepad <b class="price">$3.00</b><!----></li><li class="sold" id="2">Pen <b class="price">$1.50</b><span class="badge">gone</span></li></ul>
<ul><li class="" id="3">Notepad <b class="price">$3.00</b><!----></li><li class="" id="1">Notebook <b class="price">$4.50</b><!----></li></ul>
<p class="empty">No products match &quot;zzz&quot;</p>
```

<details><summary>Show solution and explanation</summary>

<pre><code>import { createSSRApp, defineComponent, computed, provide, inject } from &#x27;vue&#x27;
import { renderToString as _render } from &#x27;vue/server-renderer&#x27;
const renderToString = async app =&gt; (await _render(app)).replace(/&lt;!--[\[\]]--&gt;/g, &#x27;&#x27;)

const PriceView = defineComponent({
  props: { cents: { type: Number, required: true } },
  setup(props) {
    const fmt = inject(&#x27;money&#x27;, c =&gt; String(c))
    const text = computed(() =&gt; fmt(props.cents))
    return { text }
  },
  template: &#x27;&lt;b class=&quot;price&quot;&gt;{{ text }}&lt;/b&gt;&#x27;
})

const ProductCard = defineComponent({
  components: { PriceView },
  props: { name: String, cents: Number, stock: { type: Number, default: 0 } },
  template: `&lt;li :class=&quot;{ sold: stock === 0 }&quot;&gt;&lt;slot name=&quot;title&quot;&gt;{{ name }}&lt;/slot&gt; &lt;PriceView :cents=&quot;cents&quot; /&gt;&lt;slot v-if=&quot;stock === 0&quot; name=&quot;badge&quot;&gt;&lt;em&gt;sold out&lt;/em&gt;&lt;/slot&gt;&lt;/li&gt;`
})

const ProductList = defineComponent({
  components: { ProductCard },
  props: { products: Array, query: { type: String, default: &#x27;&#x27; }, sort: { type: String, default: &#x27;name&#x27; } },
  setup(props) {
    const visible = computed(() =&gt; {
      const q = props.query.toLowerCase()
      const list = props.products.filter(p =&gt; p.name.toLowerCase().includes(q))
      return [...list].sort((a, b) =&gt; (props.sort === &#x27;price&#x27; ? a.cents - b.cents : a.name.localeCompare(b.name)))
    })
    return { visible }
  },
  template: `&lt;ul v-if=&quot;visible.length&quot;&gt;&lt;ProductCard v-for=&quot;p in visible&quot; :key=&quot;p.id&quot; v-bind=&quot;p&quot;&gt;&lt;template #badge&gt;&lt;span class=&quot;badge&quot;&gt;gone&lt;/span&gt;&lt;/template&gt;&lt;/ProductCard&gt;&lt;/ul&gt;&lt;p v-else class=&quot;empty&quot;&gt;No products match &quot;{{ query }}&quot;&lt;/p&gt;`
})

const products = [
  { id: 1, name: &#x27;Notebook&#x27;, cents: 450, stock: 12 },
  { id: 2, name: &#x27;Pen&#x27;, cents: 150, stock: 0 },
  { id: 3, name: &#x27;Notepad&#x27;, cents: 300, stock: 3 },
]
const money = c =&gt; `$${(c / 100).toFixed(2)}`

async function page(props) {
  const app = createSSRApp({
    components: { ProductList },
    setup() { provide(&#x27;money&#x27;, money); return { props } },
    template: &#x27;&lt;ProductList v-bind=&quot;props&quot; /&gt;&#x27;,
  })
  return renderToString(app)
}

console.log(await page({ products }))
console.log(await page({ products, query: &#x27;note&#x27;, sort: &#x27;price&#x27; }))
console.log(await page({ products, query: &#x27;zzz&#x27; }))
</code></pre>

<p>v-bind=&quot;p&quot; passes every product field; declared props are consumed and the undeclared id becomes an attribute on the card&#x27;s root element (attribute fallthrough). The scoped badge slot replaces the fallback only for sold-out items, and an empty filter shows the empty state. Verified by running it with Vue 3.5 in Node.</p>
</details>
