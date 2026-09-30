---
description: "Test Vue apps with Vitest and Vue Test Utils: behaviour-focused component tests, async flushing, module mocks, stubs, testing composables and stores, coverage and E2E."
---

# 07 · Testing with Vitest & Vue Test Utils

You've written tests since Level 1. This lesson steps back and makes the approach
explicit: what to test, which tools do what, how to handle async code and dependencies,
and the traps we fell into along the way (two of them while writing this very lesson).
Tests ran on Vitest 4.1.11 with `@vue/test-utils` 2.5.1 and jsdom.

## What to test

Test components the way users use them: **render, interact, assert on what's visible**.
A good component test reads like a user story — "type a city, submit, see the forecast" —
and doesn't care whether the component uses one ref or three, a `computed` or a method.
Tests written against internals (`wrapper.vm.someRef`, calling private functions) break
every time you refactor, and pass even when the UI is broken.

A practical split:

| Layer | Tool | What it catches |
|-------|------|-----------------|
| Pure functions, composables, stores | Vitest (no DOM needed for most) | logic errors, edge cases |
| Components | Vitest + Vue Test Utils + jsdom | rendering, events, props/emits, states |
| Whole flows in a real browser | Playwright (or Cypress) | routing, real CSS/layout, integration with a real backend |

Most tests should be in the first two rows: they're fast and precise. Keep a smaller set
of end-to-end tests for critical paths (sign-in, checkout). The
[Playwright Mastery Path](https://sigilipelli.github.io/playwright-mastery-path/) covers E2E
in depth.

## The component under test

```ts title="src/testing/weather.ts"
export interface Forecast { city: string; tempC: number; summary: string }

export async function fetchForecast(city: string): Promise<Forecast> {
  const res = await fetch(`/api/forecast?city=${encodeURIComponent(city)}`)
  if (!res.ok) throw new Error(`Forecast unavailable (${res.status})`)
  return res.json()
}
```

```vue title="src/testing/WeatherWidget.vue"
<script setup lang="ts">
import { ref } from 'vue'
import { fetchForecast, type Forecast } from './weather'

const city = ref('')
const forecast = ref<Forecast | null>(null)
const error = ref('')
const loading = ref(false)

async function lookup() {
  if (!city.value.trim()) return
  loading.value = true
  error.value = ''
  try {
    forecast.value = await fetchForecast(city.value.trim())
  } catch (e) {
    forecast.value = null
    error.value = (e as Error).message
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <form @submit.prevent="lookup">
    <label for="city">City</label>
    <input id="city" v-model="city" />
    <button type="submit" :disabled="loading">{{ loading ? 'Loading…' : 'Get forecast' }}</button>
  </form>
  <p v-if="error" role="alert">{{ error }}</p>
  <section v-else-if="forecast" aria-label="Forecast">
    <h2>{{ forecast.city }}</h2>
    <p>{{ Math.round(forecast.tempC) }}°C · {{ forecast.summary }}</p>
    <RouterLink :to="`/cities/${forecast.city}`">Details</RouterLink>
  </section>
</template>
```

## The tests

```ts title="src/testing/__tests__/WeatherWidget.spec.ts"
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mount, flushPromises, RouterLinkStub } from '@vue/test-utils'
import WeatherWidget from '../WeatherWidget.vue'
import { fetchForecast, type Forecast } from '../weather'

// replace the whole module; the component imports the mocked function
vi.mock('../weather', () => ({ fetchForecast: vi.fn() }))

function setup() {
  return mount(WeatherWidget, {
    global: { stubs: { RouterLink: RouterLinkStub } }, // no real router needed
  })
}

describe('WeatherWidget', () => {
  beforeEach(() => {
    // braces matter: a function *returned* from beforeEach is run as teardown
    vi.mocked(fetchForecast).mockReset()
  })

  it('shows the forecast for the typed city', async () => {
    // a promise we resolve by hand, so we can look at the in-between state
    let resolve!: (f: Forecast) => void
    vi.mocked(fetchForecast).mockReturnValue(new Promise((r) => (resolve = r)))
    const wrapper = setup()

    await wrapper.get('#city').setValue('  Hyderabad ')
    await wrapper.get('form').trigger('submit')
    expect(wrapper.get('button').text()).toBe('Loading…')

    resolve({ city: 'Hyderabad', tempC: 31.6, summary: 'Humid' })
    await flushPromises()
    expect(fetchForecast).toHaveBeenCalledWith('Hyderabad')
    expect(wrapper.get('section').text()).toContain('32°C · Humid')
    expect(wrapper.getComponent(RouterLinkStub).props('to')).toBe('/cities/Hyderabad')
  })

  it('shows the error message when the lookup fails', async () => {
    vi.mocked(fetchForecast).mockRejectedValue(new Error('Forecast unavailable (503)'))
    const wrapper = setup()

    await wrapper.get('#city').setValue('Pune')
    await wrapper.get('form').trigger('submit')
    await flushPromises()

    expect(wrapper.get('[role="alert"]').text()).toBe('Forecast unavailable (503)')
    expect(wrapper.find('section').exists()).toBe(false)
  })

  it('does nothing for a blank city', async () => {
    const wrapper = setup()
    await wrapper.get('form').trigger('submit')
    expect(fetchForecast).not.toHaveBeenCalled()
  })
})
```

```text
 ✓ src/testing/__tests__/WeatherWidget.spec.ts > WeatherWidget > shows the forecast for the typed city 23ms
 ✓ src/testing/__tests__/WeatherWidget.spec.ts > WeatherWidget > shows the error message when the lookup fails 3ms
 ✓ src/testing/__tests__/WeatherWidget.spec.ts > WeatherWidget > does nothing for a blank city 1ms
      Tests  3 passed (3)
```

Let's go through the techniques in it.

### Querying: `get` vs `find`

`wrapper.get(selector)` throws with a helpful message if nothing matches — use it when the
element must exist. `wrapper.find(selector)` returns an empty wrapper instead — use it with
`.exists()` to assert absence. `findAll` returns an array. `getComponent`/`findComponent`
locate child components (by imported definition, name or ref).

Prefer selectors a user would recognise: labels, roles, visible text, ids tied to labels.
`[role="alert"]` survives a restyle; `.text-red-600` doesn't.

### Interacting and waiting

`setValue()` and `trigger()` return promises that resolve after Vue's next DOM update
(`nextTick`). **Awaiting them is required** — without `await`, you assert against the DOM
before it re-rendered.

`nextTick` covers Vue's updates, but not your own promises. `flushPromises()` resolves all
pending promise callbacks, so async code (a mocked API call and the state updates after it)
completes before you assert.

### Controlling async timing

Our first version of the first test used `mockResolvedValue` and asserted that the button
said "Loading…" right after submitting. It failed:

```text
AssertionError: expected 'Get forecast' to be 'Loading…' // Object.is equality
```

An already-resolved promise settles in a microtask — and `await trigger()` waits for a
microtask too, so by the time we looked, loading had finished. To observe the in-between
state, the test returns a promise it resolves **by hand**. The same technique tests "cancel
while loading", optimistic updates and race conditions.

### Mocking modules with `vi.mock`

`vi.mock('../weather', factory)` replaces the module for every importer in this test file —
including the component. Vitest **hoists** `vi.mock` calls above the imports, which is why
it can appear after them in the file. `vi.mocked(fn)` is just a typed view of the mock.

Mock at the **boundary you own** — here, the API module — rather than mocking Vue internals
or `fetch` globally in component tests. (`fetch` stubbing, as in Level 2 · 07's store tests,
is right when you're testing the API layer itself.)

### A `beforeEach` trap

The second test failed with its own mocked error:

```text
FAIL  src/testing/__tests__/WeatherWidget.spec.ts > WeatherWidget > shows the error message when the lookup fails
Error: Forecast unavailable (503)
```

The component caught the error correctly — rendering the component alone showed
`<p role="alert">Forecast unavailable (503)</p>`. The culprit was the hook:

```ts
beforeEach(() => vi.mocked(fetchForecast).mockReset())   // ✗
```

`mockReset()` returns the mock itself, the arrow function returns that, and **Vitest treats
a function returned from `beforeEach` as a teardown function** — so after the test it called
`fetchForecast()`, which returned the rejected promise we'd configured, failing the test.
Braces fix it. It's the kind of bug that makes you distrust your tests; knowing the rule
makes it a two-second fix.

### Stubbing child components

`global: { stubs: { RouterLink: RouterLinkStub } }` replaces `RouterLink` with a stub that
renders its slot and records its props, so we don't need a router at all.
`getComponent(RouterLinkStub).props('to')` asserts the link target. You can stub any child:
`stubs: { HeavyChart: true }` renders `<heavy-chart-stub>`. `shallowMount` stubs *all*
children — occasionally useful, but it tests less of what users see, so prefer `mount` and
stub selectively.

## Testing composables

A composable that only uses reactivity can be called directly. If it uses lifecycle hooks
or `inject`, it needs a component context — mount a tiny host:

```ts
import { mount } from '@vue/test-utils'
import { defineComponent, h } from 'vue'

function withSetup<T>(composable: () => T) {
  let result!: T
  const wrapper = mount(defineComponent({ setup() { result = composable(); return () => h('div') } }))
  return { result, wrapper }   // wrapper.unmount() to test cleanup
}

const { result, wrapper } = withSetup(() => useWindowSize())
```

For composables that only need an effect scope (watchers, `onScopeDispose`), `effectScope`
is enough — Level 2 · 01 tested `useEventListener` that way.

## Testing stores

Stores are covered in Level 2 · 06: a fresh Pinia per test with `setActivePinia`, a
throwaway app when plugins are involved, and `createTestingPinia` for components that use
stores.

## Coverage

```bash
npm install -D @vitest/coverage-v8
npx vitest run --coverage
```

Install the coverage package matching your Vitest version. For this lesson's folder:

```text
 % Coverage report from v8
-------------------|---------|----------|---------|---------|-------------------
File               | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
-------------------|---------|----------|---------|---------|-------------------
All files          |   78.94 |    86.66 |      75 |   82.35 |
 weather.ts        |       0 |        0 |       0 |       0 | 4-6
-------------------|---------|----------|---------|---------|-------------------
```

The text table listed only the file that wasn't fully covered; the JSON summary showed
`WeatherWidget.vue` at 100% of lines and branches. `weather.ts` is at 0% — because we
mocked it. That's the honest reading of coverage: it tells you what *ran*, not what's
*verified*. The API module needs its own tests (with `fetch` stubbed). Use coverage to find
untested branches, not as a target to hit.

## Browser-based component tests

jsdom is not a browser: no layout, no real CSS, no `IntersectionObserver`, no real focus
rules in every case. When a component depends on those, run component tests in a real
browser — Vitest's **browser mode** (with Playwright as the provider) runs the same test
files in Chromium, Firefox or WebKit, and Cypress has a component-testing mode too. We
didn't set up browser mode for this course; check the Vitest docs for the current setup
steps, which changed across Vitest 3 and 4.

## How It Actually Works

`mount(Component, options)` creates a real Vue app (`createApp`) with a wrapper root
component, applies `global` options (plugins, stubs, provides, directives) to that app,
mounts it into a detached `<div>` (or `attachTo`), and returns a `VueWrapper` around the
component instance. Nothing is simulated — it's the real renderer writing real (jsdom) DOM.

`trigger('click')` dispatches a real DOM `Event` on the element and then returns
`nextTick()`. Stubs work through the app's component resolution: VTU installs a
transformation that swaps the component definition for a stub before the renderer creates
the vnode's instance.

`vi.mock` works at the module-loader level. Vitest runs test files through Vite's
transform pipeline; its transformer rewrites `vi.mock` calls to the top of the file and
registers the factory, and the module runner returns the factory's result whenever that
path is imported — which is why the component, importing `./weather` normally, receives the
mock.

## Common mistakes

- **Not awaiting `trigger`/`setValue`** — asserting before the re-render.
- **Using `nextTick` when you need `flushPromises`** (or vice versa).
- **Asserting on `wrapper.vm` internals** instead of the rendered output.
- **Expression-bodied `beforeEach`/`afterEach` returning a function** — it becomes a
  teardown.
- **Shared state between tests** — module-level refs, one Pinia for all tests, mocks not
  reset.
- **Over-mocking** until the test only proves that mocks return what you told them.
- **Snapshot tests of whole components** — large snapshots get approved without reading;
  assert on specific text and attributes instead.

## Exercise

1. Write tests for `fetchForecast` itself with `fetch` stubbed: success, a 503, and a city
   name containing spaces and `&` (check the encoded URL).
2. Add a "Use my location" button to `WeatherWidget` that calls
   `navigator.geolocation.getCurrentPosition`. Stub it with `vi.stubGlobal` and test the
   success and permission-denied paths.
3. Write a `withSetup` helper as shown and use it to test that `useWindowSize` removes its
   `resize` listener on unmount (spy on `window.removeEventListener`).
4. Break `WeatherWidget` on purpose (swap the `v-if`/`v-else-if` order) and check that at
   least one test fails. If none does, your tests have a gap — add one.
