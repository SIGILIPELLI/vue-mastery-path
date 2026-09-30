---
description: "Level 2 project: a Vue Recipe Book with Vue Router, a Pinia store over a swappable API layer, URL-driven filters, validated forms, leave guards and tests."
---

# 10 · Project — A Recipe Book

This project combines everything from Level 2 into a small but complete multi-page app:
browse recipes, search and filter them (with the filters in the URL), view one, create and
edit recipes with validation, and get warned before leaving a half-filled form. The full
source is on this page. We ran it: `vue-tsc` passes, `vite build` succeeds, and all nine
tests pass.

| Feature | Lesson |
|---------|--------|
| Routes with params as props, lazy views, 404, titles, scroll | 03, 04 |
| `onBeforeRouteLeave` for unsaved changes | 04 |
| Setup store with a reactive `Map` cache | 05, 07 |
| API layer that can be swapped for real `fetch` calls | 07 |
| Race-safe loading keyed on a prop | 01, 07 |
| Validated form with accessible errors, `defineExpose` | 08, 09 |
| Store, component and whole-app tests | 06 |

## 1. Scaffold

```bash
npm create vue@latest recipe-book -- --ts --router --pinia --vitest --bare
cd recipe-book
npm install
```

`--bare` skips the example components. Delete `src/stores/counter.ts` and
`src/__tests__/App.spec.ts`, and — if you're on Node.js 25 or newer — add
`execArgv: ['--no-experimental-webstorage']` to the `test` section of `vitest.config.ts`
(see Level 1 · 10 for why).

## 2. The API layer

There's no backend in this course, so the "API" is a module of async functions with
artificial latency, storing data in `localStorage`. Everything else in the app talks to
it only through these functions, so replacing them with `fetch` calls later changes one
file.

```ts title="src/api/recipes.ts"
// A tiny fake backend: async functions with latency, persisted in localStorage.
// Swap this file for real fetch() calls without touching the store or components.
export interface Recipe {
  id: string
  title: string
  minutes: number
  tags: string[]
  ingredients: string[]
  steps: string
}

export type RecipeInput = Omit<Recipe, 'id'>

const KEY = 'recipe-book:recipes'
const delay = (ms = 150) => new Promise((r) => setTimeout(r, ms))

const seed: Recipe[] = [
  {
    id: 'dal-tadka',
    title: 'Dal Tadka',
    minutes: 35,
    tags: ['vegetarian', 'indian'],
    ingredients: ['1 cup toor dal', '1 onion', '2 tomatoes', '1 tsp cumin', '2 cloves garlic', 'ghee'],
    steps: 'Pressure-cook the dal until soft.\nFry cumin, garlic, onion and tomato in ghee.\nPour the tempering over the dal and simmer 5 minutes.',
  },
  {
    id: 'shakshuka',
    title: 'Shakshuka',
    minutes: 25,
    tags: ['vegetarian', 'breakfast'],
    ingredients: ['4 eggs', '1 can tomatoes', '1 red pepper', '1 tsp paprika', 'olive oil'],
    steps: 'Soften pepper in oil.\nAdd tomatoes and paprika; simmer 10 minutes.\nMake wells, crack in the eggs, cover until set.',
  },
]

function read(): Recipe[] {
  try {
    const raw = localStorage.getItem(KEY)
    return raw ? (JSON.parse(raw) as Recipe[]) : structuredClone(seed)
  } catch {
    return structuredClone(seed)
  }
}
function write(recipes: Recipe[]) {
  localStorage.setItem(KEY, JSON.stringify(recipes))
}

export class NotFoundError extends Error {
  constructor(id: string) {
    super(`Recipe "${id}" not found`)
    this.name = 'NotFoundError'
  }
}

export async function listRecipes(): Promise<Recipe[]> {
  await delay()
  return read()
}

export async function getRecipe(id: string): Promise<Recipe> {
  await delay()
  const recipe = read().find((r) => r.id === id)
  if (!recipe) throw new NotFoundError(id)
  return recipe
}

function slugify(title: string): string {
  return title.toLowerCase().trim().replace(/[^a-z0-9]+/g, '-').replace(/^-|-$/g, '') || 'recipe'
}

export async function createRecipe(input: RecipeInput): Promise<Recipe> {
  await delay()
  const all = read()
  let id = slugify(input.title)
  for (let n = 2; all.some((r) => r.id === id); n++) id = `${slugify(input.title)}-${n}`
  const recipe = { ...input, id }
  write([...all, recipe])
  return recipe
}

export async function updateRecipe(id: string, input: RecipeInput): Promise<Recipe> {
  await delay()
  const all = read()
  const i = all.findIndex((r) => r.id === id)
  if (i === -1) throw new NotFoundError(id)
  const recipe = { ...input, id }
  all[i] = recipe
  write(all)
  return recipe
}

export async function deleteRecipe(id: string): Promise<void> {
  await delay()
  write(read().filter((r) => r.id !== id))
}
```

`NotFoundError` is a real class, so views can use `instanceof` to tell "this recipe doesn't
exist" from any other failure. `structuredClone(seed)` ensures nobody mutates the seed
array by accident.

## 3. The store

```ts title="src/stores/recipes.ts"
import { ref, computed } from 'vue'
import { defineStore, acceptHMRUpdate } from 'pinia'
import * as api from '@/api/recipes'
import type { Recipe, RecipeInput } from '@/api/recipes'

export const useRecipesStore = defineStore('recipes', () => {
  const byId = ref(new Map<string, Recipe>())
  const listLoaded = ref(false)
  const loading = ref(false)
  const error = ref<Error | null>(null)

  const all = computed(() => [...byId.value.values()].sort((a, b) => a.title.localeCompare(b.title)))
  const allTags = computed(() => [...new Set(all.value.flatMap((r) => r.tags))].sort())

  async function loadAll() {
    if (listLoaded.value) return
    loading.value = true
    error.value = null
    try {
      for (const r of await api.listRecipes()) byId.value.set(r.id, r)
      listLoaded.value = true
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }

  async function load(id: string): Promise<Recipe> {
    const cached = byId.value.get(id)
    if (cached) return cached
    const recipe = await api.getRecipe(id)
    byId.value.set(id, recipe)
    return recipe
  }

  async function save(input: RecipeInput, id?: string): Promise<Recipe> {
    const recipe = id ? await api.updateRecipe(id, input) : await api.createRecipe(input)
    byId.value.set(recipe.id, recipe)
    return recipe
  }

  async function remove(id: string) {
    await api.deleteRecipe(id)
    byId.value.delete(id)
  }

  return { byId, all, allTags, loading, error, loadAll, load, save, remove }
})

if (import.meta.hot) import.meta.hot.accept(acceptHMRUpdate(useRecipesStore, import.meta.hot))
```

The store is a cache in front of the API: `loadAll` runs once per session; `load(id)`
answers from the cache when it can (so opening a recipe from the list is instant); `save`
and `remove` update the cache after the API succeeds. `all` and `allTags` are getters
derived from the cache, so creating a recipe with a new tag updates the tag filter
automatically.

## 4. Routes

```ts title="src/router/index.ts"
import { createRouter, createWebHistory } from 'vue-router'
import RecipeListView from '@/views/RecipeListView.vue'

declare module 'vue-router' {
  interface RouteMeta {
    title?: string
  }
}

export const routes = [
  { path: '/', name: 'recipes', component: RecipeListView, meta: { title: 'Recipes' } },
  {
    path: '/recipes/new',
    name: 'recipe-new',
    component: () => import('@/views/RecipeEditView.vue'),
    meta: { title: 'New recipe' },
  },
  {
    path: '/recipes/:id',
    name: 'recipe',
    component: () => import('@/views/RecipeDetailView.vue'),
    props: true,
  },
  {
    path: '/recipes/:id/edit',
    name: 'recipe-edit',
    component: () => import('@/views/RecipeEditView.vue'),
    props: true,
    meta: { title: 'Edit recipe' },
  },
  { path: '/:pathMatch(.*)*', name: 'not-found', component: () => import('@/views/NotFoundView.vue') },
]

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
  scrollBehavior: (to, from, saved) => saved ?? (to.path !== from.path ? { top: 0 } : undefined),
})

router.afterEach((to) => {
  document.title = to.meta.title ? `${to.meta.title} · Recipe Book` : 'Recipe Book'
})

export default router
```

- The list view is imported eagerly (it's where most people land); the rest are lazy.
- `routes` is exported so tests can build their own router with memory history.
- The same `RecipeEditView` serves both `/recipes/new` (no `id` prop) and
  `/recipes/:id/edit`.
- `RouteMeta` is augmented so `meta.title` is typed.

## 5. The shell and the list

```vue title="src/App.vue"
<template>
  <header class="site-header">
    <RouterLink to="/" class="brand">🍲 Recipe Book</RouterLink>
  </header>
  <main class="container">
    <RouterView />
  </main>
</template>

<style>
body { margin: 0; font-family: system-ui, sans-serif; line-height: 1.5; }
.site-header { padding: 0.75rem 1rem; border-bottom: 1px solid #e5e7eb; }
.brand { font-weight: 700; text-decoration: none; color: inherit; }
.container { max-width: 60rem; margin: 0 auto; padding: 1rem; }
</style>
```

```vue title="src/components/RecipeCard.vue"
<script setup lang="ts">
import type { Recipe } from '@/api/recipes'
defineProps<{ recipe: Recipe }>()
</script>

<template>
  <article class="card">
    <h2>
      <RouterLink :to="{ name: 'recipe', params: { id: recipe.id } }">{{ recipe.title }}</RouterLink>
    </h2>
    <p class="meta">{{ recipe.minutes }} min · {{ recipe.ingredients.length }} ingredients</p>
    <ul class="tags" aria-label="Tags">
      <li v-for="tag in recipe.tags" :key="tag">#{{ tag }}</li>
    </ul>
  </article>
</template>

<style scoped>
.card { border: 1px solid #e5e7eb; border-radius: 8px; padding: 1rem; }
.card h2 { margin: 0 0 0.25rem; font-size: 1.1rem; }
.meta { color: #6b7280; margin: 0; }
.tags { display: flex; gap: 0.5rem; list-style: none; padding: 0; margin: 0.5rem 0 0; font-size: 0.85rem; }
</style>
```

```vue title="src/views/RecipeListView.vue"
<script setup lang="ts">
import { computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { storeToRefs } from 'pinia'
import { useRecipesStore } from '@/stores/recipes'
import RecipeCard from '@/components/RecipeCard.vue'

const store = useRecipesStore()
const { all, allTags, loading, error } = storeToRefs(store)
store.loadAll()

const route = useRoute()
const router = useRouter()

// search text and tag live in the URL, so filtered views can be shared and survive reloads
const q = computed({
  get: () => (typeof route.query.q === 'string' ? route.query.q : ''),
  set: (value) => router.replace({ query: { ...route.query, q: value || undefined } }),
})
const tag = computed({
  get: () => (typeof route.query.tag === 'string' ? route.query.tag : ''),
  set: (value) => router.replace({ query: { ...route.query, tag: value || undefined } }),
})

const visible = computed(() => {
  const needle = q.value.trim().toLowerCase()
  return all.value.filter(
    (r) =>
      (!tag.value || r.tags.includes(tag.value)) &&
      (!needle ||
        r.title.toLowerCase().includes(needle) ||
        r.ingredients.some((i) => i.toLowerCase().includes(needle))),
  )
})
</script>

<template>
  <section>
    <header class="toolbar">
      <h1>Recipes</h1>
      <RouterLink :to="{ name: 'recipe-new' }" class="button">New recipe</RouterLink>
    </header>

    <div class="filters">
      <label>Search <input v-model="q" type="search" placeholder="Title or ingredient" /></label>
      <label>
        Tag
        <select v-model="tag">
          <option value="">All</option>
          <option v-for="t in allTags" :key="t" :value="t">{{ t }}</option>
        </select>
      </label>
    </div>

    <p v-if="loading && !all.length" aria-busy="true">Loading recipes…</p>
    <p v-else-if="error" role="alert">Couldn't load recipes: {{ error.message }}</p>
    <p v-else-if="!visible.length" class="empty">No recipes match your filters.</p>
    <div v-else class="grid">
      <RecipeCard v-for="r in visible" :key="r.id" :recipe="r" />
    </div>
  </section>
</template>

<style scoped>
.toolbar { display: flex; justify-content: space-between; align-items: center; }
.filters { display: flex; gap: 1rem; margin-bottom: 1rem; }
.grid { display: grid; gap: 1rem; grid-template-columns: repeat(auto-fill, minmax(16rem, 1fr)); }
</style>
```

The search box and tag select are bound to **writable computeds backed by the URL**.
Reading `q` reads `route.query.q`; writing it calls `router.replace` with the new query.
The result: filters survive reloads, can be shared as links, and the browser's Back button
doesn't step through every keystroke (because it's `replace`, not `push`). Setting a
value to `undefined` removes it from the query, keeping URLs clean.

## 6. The detail view

```vue title="src/views/RecipeDetailView.vue"
<script setup lang="ts">
import { ref, watch } from 'vue'
import { useRouter } from 'vue-router'
import { useRecipesStore } from '@/stores/recipes'
import { NotFoundError, type Recipe } from '@/api/recipes'

const { id } = defineProps<{ id: string }>()
const store = useRecipesStore()
const router = useRouter()

const recipe = ref<Recipe | null>(null)
const notFound = ref(false)

watch(
  () => id,
  async (current) => {
    notFound.value = false
    try {
      const result = await store.load(current)
      if (current === id) {
        recipe.value = result
        document.title = `${result.title} · Recipe Book`
      }
    } catch (e) {
      if (e instanceof NotFoundError) notFound.value = true
      else throw e
    }
  },
  { immediate: true },
)

async function onDelete() {
  if (!recipe.value || !window.confirm(`Delete “${recipe.value.title}”?`)) return
  await store.remove(recipe.value.id)
  await router.push({ name: 'recipes' })
}
</script>

<template>
  <section v-if="notFound">
    <h1>Recipe not found</h1>
    <RouterLink :to="{ name: 'recipes' }">Back to all recipes</RouterLink>
  </section>
  <article v-else-if="recipe">
    <h1>{{ recipe.title }}</h1>
    <p>{{ recipe.minutes }} minutes · {{ recipe.tags.join(', ') }}</p>
    <h2>Ingredients</h2>
    <ul>
      <li v-for="(item, i) in recipe.ingredients" :key="i">{{ item }}</li>
    </ul>
    <h2>Method</h2>
    <ol>
      <li v-for="(step, i) in recipe.steps.split('\n').filter(Boolean)" :key="i">{{ step }}</li>
    </ol>
    <nav class="actions">
      <RouterLink :to="{ name: 'recipe-edit', params: { id: recipe.id } }">Edit</RouterLink>
      <button type="button" @click="onDelete">Delete</button>
    </nav>
  </article>
  <p v-else aria-busy="true">Loading…</p>
</template>
```

The `current === id` check makes quick navigation between recipes safe — the Level 2 · 07
pattern. Anything other than `NotFoundError` is rethrown, to be handled by a global error
handler (Level 3 · 09).

## 7. The form and the edit view

```vue title="src/components/RecipeForm.vue"
<script setup lang="ts">
import { reactive, computed, ref } from 'vue'
import type { RecipeInput } from '@/api/recipes'

const { initial } = defineProps<{ initial?: RecipeInput }>()
const emit = defineEmits<{ submit: [value: RecipeInput] }>()

// the form edits text versions of the list fields; they're parsed on submit
const form = reactive({
  title: initial?.title ?? '',
  minutes: initial?.minutes ?? 30,
  tags: initial?.tags.join(', ') ?? '',
  ingredients: initial?.ingredients.join('\n') ?? '',
  steps: initial?.steps ?? '',
})
const snapshot = JSON.stringify(form)
const submitted = ref(false)

const lines = (s: string) => s.split('\n').map((l) => l.trim()).filter(Boolean)

const errors = computed(() => {
  const e: Partial<Record<keyof typeof form, string>> = {}
  if (!form.title.trim()) e.title = 'Give the recipe a title'
  if (!Number.isInteger(form.minutes) || form.minutes < 1) e.minutes = 'Enter a whole number of minutes'
  if (lines(form.ingredients).length === 0) e.ingredients = 'Add at least one ingredient'
  if (!form.steps.trim()) e.steps = 'Describe how to make it'
  return e
})

const isDirty = computed(() => JSON.stringify(form) !== snapshot)
defineExpose({ isDirty })

function onSubmit() {
  submitted.value = true
  if (Object.keys(errors.value).length) return
  emit('submit', {
    title: form.title.trim(),
    minutes: form.minutes,
    tags: form.tags.split(',').map((t) => t.trim().toLowerCase()).filter(Boolean),
    ingredients: lines(form.ingredients),
    steps: form.steps.trim(),
  })
}
const show = (k: keyof typeof form) => (submitted.value ? errors.value[k] : undefined)
</script>

<template>
  <form novalidate @submit.prevent="onSubmit">
    <label for="rf-title">Title</label>
    <input id="rf-title" v-model="form.title" :aria-invalid="!!show('title')" aria-describedby="rf-title-err" />
    <p id="rf-title-err" class="error">{{ show('title') }}</p>

    <label for="rf-minutes">Minutes</label>
    <input id="rf-minutes" v-model.number="form.minutes" type="number" min="1" :aria-invalid="!!show('minutes')" aria-describedby="rf-minutes-err" />
    <p id="rf-minutes-err" class="error">{{ show('minutes') }}</p>

    <label for="rf-tags">Tags <small>(comma separated)</small></label>
    <input id="rf-tags" v-model="form.tags" />

    <label for="rf-ingredients">Ingredients <small>(one per line)</small></label>
    <textarea id="rf-ingredients" v-model="form.ingredients" rows="6" :aria-invalid="!!show('ingredients')" aria-describedby="rf-ingredients-err" />
    <p id="rf-ingredients-err" class="error">{{ show('ingredients') }}</p>

    <label for="rf-steps">Method <small>(one step per line)</small></label>
    <textarea id="rf-steps" v-model="form.steps" rows="6" :aria-invalid="!!show('steps')" aria-describedby="rf-steps-err" />
    <p id="rf-steps-err" class="error">{{ show('steps') }}</p>

    <button type="submit">Save recipe</button>
  </form>
</template>

<style scoped>
form { display: grid; gap: 0.25rem; max-width: 36rem; }
.error { color: #b91c1c; min-height: 1.25em; margin: 0 0 0.5rem; }
</style>
```

The form edits **text** versions of the list fields (comma-separated tags, one ingredient
per line) and parses them on submit — simpler for users than dynamic lists of inputs, and
the parsing is covered by a test. It shows errors only after the first submit attempt, and
exposes `isDirty` so its parent can ask about unsaved changes without owning the form's
state.

```vue title="src/views/RecipeEditView.vue"
<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'
import { useRouter, onBeforeRouteLeave } from 'vue-router'
import { useRecipesStore } from '@/stores/recipes'
import RecipeForm from '@/components/RecipeForm.vue'
import type { RecipeInput } from '@/api/recipes'

const { id } = defineProps<{ id?: string }>()
const store = useRecipesStore()
const router = useRouter()
const formRef = useTemplateRef('form')

// editing: load first, then render the form with the loaded values
const initial = ref<RecipeInput | undefined>()
const ready = ref(!id)
if (id) {
  store.load(id).then((r) => {
    const { id: _ignored, ...rest } = r
    initial.value = rest
    ready.value = true
  })
}

const saving = ref(false)
const saveError = ref<string | null>(null)
let saved = false

async function onSubmit(value: RecipeInput) {
  saving.value = true
  saveError.value = null
  try {
    const recipe = await store.save(value, id)
    saved = true
    await router.push({ name: 'recipe', params: { id: recipe.id } })
  } catch (e) {
    saveError.value = (e as Error).message
  } finally {
    saving.value = false
  }
}

onBeforeRouteLeave(() => {
  if (!saved && formRef.value?.isDirty && !window.confirm('Discard unsaved changes?')) return false
})
</script>

<template>
  <section>
    <h1>{{ id ? 'Edit recipe' : 'New recipe' }}</h1>
    <p v-if="saveError" role="alert">Couldn't save: {{ saveError }}</p>
    <RecipeForm v-if="ready" ref="form" :initial="initial" @submit="onSubmit" />
    <p v-else aria-busy="true">Loading…</p>
    <p v-if="saving" aria-live="polite">Saving…</p>
  </section>
</template>
```

`useTemplateRef('form')` is automatically typed as the `RecipeForm` instance, so
`formRef.value?.isDirty` type-checks. (Exposed refs are unwrapped on the instance, which is
why it's `isDirty`, not `isDirty.value`.) The `saved` flag lets the guard skip the
confirmation when the navigation *is* the successful save.

```vue title="src/views/NotFoundView.vue"
<template>
  <section>
    <h1>Page not found</h1>
    <p>That page doesn't exist. <RouterLink to="/">See all recipes</RouterLink>.</p>
  </section>
</template>
```

## 8. Tests

Store tests use fake timers with `shouldAdvanceTime`, so the fake API's 150 ms delays pass
without slowing the suite much:

```ts title="src/stores/__tests__/recipes.spec.ts"
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { createPinia, setActivePinia } from 'pinia'
import { useRecipesStore } from '../recipes'
import * as api from '@/api/recipes'

describe('recipes store', () => {
  beforeEach(() => {
    localStorage.clear()
    setActivePinia(createPinia())
    vi.useFakeTimers({ shouldAdvanceTime: true })   // the fake API waits 150 ms per call
  })

  it('loads the seeded recipes once, sorted by title', async () => {
    const spy = vi.spyOn(api, 'listRecipes')
    const store = useRecipesStore()
    await store.loadAll()
    await store.loadAll()
    expect(spy).toHaveBeenCalledTimes(1)
    expect(store.all.map((r) => r.title)).toEqual(['Dal Tadka', 'Shakshuka'])
    expect(store.allTags).toEqual(['breakfast', 'indian', 'vegetarian'])
  })

  it('creates a recipe with a unique slug id', async () => {
    const store = useRecipesStore()
    const input = { title: 'Dal Tadka', minutes: 30, tags: [], ingredients: ['dal'], steps: 'cook' }
    const created = await store.save(input)
    expect(created.id).toBe('dal-tadka-2')
    expect(store.byId.get('dal-tadka-2')?.minutes).toBe(30)
  })

  it('throws NotFoundError for an unknown id', async () => {
    await expect(useRecipesStore().load('nope')).rejects.toBeInstanceOf(api.NotFoundError)
  })
})
```

The form test checks both the timing of errors and the parsing:

```ts title="src/components/__tests__/RecipeForm.spec.ts"
import { it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import RecipeForm from '../RecipeForm.vue'

it('shows errors only after submit, then emits parsed values', async () => {
  const wrapper = mount(RecipeForm)
  expect(wrapper.find('#rf-title-err').text()).toBe('')

  await wrapper.find('form').trigger('submit')
  expect(wrapper.find('#rf-title-err').text()).toBe('Give the recipe a title')
  expect(wrapper.emitted('submit')).toBeUndefined()

  await wrapper.find('#rf-title').setValue('  Pancakes ')
  await wrapper.find('#rf-tags').setValue('Breakfast, sweet,')
  await wrapper.find('#rf-ingredients').setValue('flour\n\n milk \neggs')
  await wrapper.find('#rf-steps').setValue('Mix.\nFry.')
  await wrapper.find('form').trigger('submit')

  expect(wrapper.emitted('submit')![0]).toEqual([
    {
      title: 'Pancakes',
      minutes: 30,
      tags: ['breakfast', 'sweet'],
      ingredients: ['flour', 'milk', 'eggs'],
      steps: 'Mix.\nFry.',
    },
  ])
})

it('tracks whether the form is dirty', async () => {
  const wrapper = mount(RecipeForm, {
    props: { initial: { title: 'Soup', minutes: 20, tags: [], ingredients: ['water'], steps: 'boil' } },
  })
  expect(wrapper.vm.isDirty).toBe(false)
  await wrapper.find('#rf-title').setValue('Better soup')
  expect(wrapper.vm.isDirty).toBe(true)
})
```

The app tests mount `App` with a real router (memory history) and a real Pinia, and drive
whole flows:

```ts title="src/router/__tests__/app.spec.ts"
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mount, flushPromises } from '@vue/test-utils'
import { createRouter, createMemoryHistory } from 'vue-router'
import { createPinia } from 'pinia'
import App from '@/App.vue'
import { routes } from '@/router'

async function mountAt(path: string) {
  const router = createRouter({ history: createMemoryHistory(), routes })
  router.push(path)
  await router.isReady()
  const wrapper = mount(App, { global: { plugins: [createPinia(), router] } })
  return { wrapper, router }
}

// the fake API sleeps; let timers run instantly
async function settle() {
  await vi.runAllTimersAsync()
  await flushPromises()
}

describe('Recipe Book app', () => {
  beforeEach(() => {
    localStorage.clear()
    vi.useFakeTimers()
  })

  it('lists recipes and filters by the ?q= query', async () => {
    const { wrapper, router } = await mountAt('/?q=egg')
    await settle()
    expect(wrapper.findAll('article h2').map((h) => h.text())).toEqual(['Shakshuka'])

    await router.push('/')
    await settle()
    expect(wrapper.findAll('article h2')).toHaveLength(2)
  })

  it('creates a recipe and lands on its page', async () => {
    const { wrapper, router } = await mountAt('/recipes/new')
    await settle()
    await wrapper.find('#rf-title').setValue('Masala Chai')
    await wrapper.find('#rf-ingredients').setValue('tea\nmilk\ncardamom')
    await wrapper.find('#rf-steps').setValue('Boil everything.')
    await wrapper.find('form').trigger('submit')

    // saving waits on the fake API, then navigation waits on the lazily imported
    // detail view — a real module load, not a timer — so poll until we arrive
    await vi.waitFor(() => expect(router.currentRoute.value.fullPath).toBe('/recipes/masala-chai'))
    await settle()
    expect(wrapper.find('h1').text()).toBe('Masala Chai')
  })

  it('asks before leaving a dirty form', async () => {
    const confirm = vi.spyOn(window, 'confirm').mockReturnValue(false)
    const { wrapper, router } = await mountAt('/recipes/new')
    await settle()
    await wrapper.find('#rf-title').setValue('Half-typed')
    await router.push('/')
    expect(confirm).toHaveBeenCalledWith('Discard unsaved changes?')
    expect(router.currentRoute.value.fullPath).toBe('/recipes/new')
  })

  it('shows a not-found page for unknown recipes', async () => {
    const { wrapper } = await mountAt('/recipes/does-not-exist')
    await settle()
    expect(wrapper.find('h1').text()).toBe('Recipe not found')
  })
})
```

Our first version of the "creates a recipe" test failed:

```text
AssertionError: expected '/recipes/new' to be '/recipes/masala-chai' // Object.is equality
```

`settle()` flushed timers and promises, but navigating to the detail page also waits for
the **lazy import** of `RecipeDetailView` — real module loading, not a timer — which hadn't
finished. `vi.waitFor` polls the assertion until it passes (and, with fake timers,
advances them between attempts), which fixed it. The final run:

```text
 ✓ src/components/__tests__/RecipeForm.spec.ts > shows errors only after submit, then emits parsed values 24ms
 ✓ src/components/__tests__/RecipeForm.spec.ts > tracks whether the form is dirty 2ms
 ✓ src/router/__tests__/app.spec.ts > Recipe Book app > lists recipes and filters by the ?q= query 31ms
 ✓ src/router/__tests__/app.spec.ts > Recipe Book app > creates a recipe and lands on its page 173ms
 ✓ src/router/__tests__/app.spec.ts > Recipe Book app > asks before leaving a dirty form 6ms
 ✓ src/router/__tests__/app.spec.ts > Recipe Book app > shows a not-found page for unknown recipes 3ms
 ✓ src/stores/__tests__/recipes.spec.ts > recipes store > loads the seeded recipes once, sorted by title 191ms
 ✓ src/stores/__tests__/recipes.spec.ts > recipes store > creates a recipe with a unique slug id 177ms
 ✓ src/stores/__tests__/recipes.spec.ts > recipes store > throws NotFoundError for an unknown id 173ms
 Test Files  3 passed (3)
      Tests  9 passed (9)
```

And `npm run build`:

```text
dist/index.html                             0.42 kB │ gzip:  0.28 kB
dist/assets/RecipeEditView-lzIqcY5t.css     0.13 kB │ gzip:  0.13 kB
dist/assets/index-kfREwQ26.css              0.79 kB │ gzip:  0.42 kB
dist/assets/NotFoundView-CD-aGKFm.js        0.39 kB │ gzip:  0.29 kB
dist/assets/RecipeDetailView-DZtA2VHT.js    1.48 kB │ gzip:  0.83 kB
dist/assets/RecipeEditView-BW-Z2ntj.js      3.68 kB │ gzip:  1.60 kB
dist/assets/index-KmR4qrb9.js             103.36 kB │ gzip: 40.26 kB
```

Each lazy view became its own chunk; the edit view's chunk even carries its own CSS file,
loaded only when someone edits.

## How It Actually Works

Trace "type `egg` into the search box" through the app:

1. `v-model="q"` calls the computed's setter with `'egg'`, which calls
   `router.replace({ query: { q: 'egg' } })`.
2. The router runs its (short) guard pipeline, calls `history.replaceState`, and assigns
   the new location to `currentRoute.value`.
3. `route.query` is reactive over `currentRoute`, so every effect that read `route.query.q`
   is triggered — the `q` computed, and through it the `visible` computed and the list's
   render effect.
4. `visible` recomputes on the next render, reading `all` from the store — which *didn't*
   change, so the store getter returns its cached array — and filters it.
5. The list re-renders; `RecipeCard`s are matched by `:key="r.id"`, so the kept cards are
   reused and only removed ones are unmounted.

The URL is now the single source of truth for the filters, and the store is the single
source of truth for the data. Neither component nor store holds a copy of the other's
state — which is what makes features like "share this filtered view" free.

## Common mistakes

- **Duplicating URL state in a ref** and trying to keep the two in sync with watchers.
  Derive from the route instead, as above.
- **`router.push` for filter changes** — every keystroke becomes a history entry.
- **Letting the edit form write directly into the store's cached object** — cancel would
  be impossible and other views would show unsaved edits. Edit a copy; save explicitly.
- **Forgetting the "saved" case in a leave guard** — users get asked to discard changes
  they just saved.
- **Tests that pass locally but depend on timing** — use fake timers and `waitFor` rather
  than real sleeps.

## Exercise

1. **Favourites.** Add a `favourite: boolean` field, a heart toggle on each card, and a
   "Favourites only" checkbox filter stored as `?fav=1`.
2. **Scaling.** On the detail page, add servings controls (½×, 1×, 2×) that scale any
   leading number in each ingredient line (`2 tomatoes` → `4 tomatoes`). Write unit tests
   for the scaling function, including fractions like `1/2 cup`.
3. **Real API.** Run `npx json-server` (or any small REST server) with a `recipes`
   collection and rewrite `src/api/recipes.ts` to use `fetch` and the `getJSON` helper from
   Lesson 07. No other file should need to change — if one does, find out why.
4. **Pinia Colada.** Replace the store's `loadAll`/`load` with `useQuery` in the views and
   `useMutation` for saving, invalidating the list after a save.
