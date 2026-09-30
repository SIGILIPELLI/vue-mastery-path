---
description: "Load Vue code on demand: defineAsyncComponent with loading, error, delay and retry, route-level splitting, preloading, lazy hydration strategies, and reading chunk output."
---

# 05 · Async Components & Code Splitting

Every kilobyte of JavaScript in your main bundle is downloaded, parsed and executed before
the app becomes interactive. Most apps contain large pieces that most visits never use: an
admin area, a rich-text editor, a charting library, a settings wizard. **Code splitting**
moves those pieces into separate files that load only when needed. In Vue that means lazy
routes (Level 2 · 04) and **async components**.

## `defineAsyncComponent`

Wrap a dynamic `import()` and you get a component that loads its real implementation the
first time it renders:

```vue title="src/split/Page.vue"
<script setup lang="ts">
import { ref, defineAsyncComponent } from 'vue'
const Heavy = defineAsyncComponent(() => import('./Heavy.vue'))
const show = ref(false)
</script>
<template><button @click="show = true">Show report</button><Heavy v-if="show" /></template>
```

Building that tiny app with Vite 8 produced a separate chunk for `Heavy.vue`:

```text
dist-split/split.html                 0.13 kB │ gzip:  0.13 kB
dist-split/assets/Heavy-C44uvFJG.js   0.33 kB │ gzip:  0.25 kB
dist-split/assets/split-CSlDgDsG.js  65.18 kB │ gzip: 25.78 kB
```

Searching the minified main chunk for the file name shows how it's referenced — through a
dynamic import wrapped in Vite's preload helper:

```text
...setup(e){let t=Gn(()=>so(()=>import(`./Heavy-C44uvFJG.js`),[]))...
```

`Gn` is the minified `defineAsyncComponent`; `so` is Vite's helper that also preloads any
CSS or shared chunks the lazy module depends on (the empty array means `Heavy` had none).
Nothing is downloaded until someone clicks "Show report" and `<Heavy>` renders.

## Loading, error, delay and retry

The options form handles the states a network load can be in:

```ts
const Chart = defineAsyncComponent({
  loader: () => import('./RevenueChart.vue'),
  loadingComponent: ChartSkeleton,     // shown while loading…
  delay: 200,                          // …but only if loading takes longer than 200 ms
  errorComponent: ChartLoadError,      // receives an `error` prop
  timeout: 10_000,                     // treat as failed after 10 s
  onError(error, retry, fail, attempts) {
    // chunk loads fail on flaky networks and right after a deploy (old chunk names)
    if (attempts <= 3) retry()
    else fail()
  },
})
```

We tested these options with a loader that rejects on its first attempt and succeeds on the
second, each attempt taking 300 ms, using fake timers:

```text
t=0 <div>
  <!---->
</div>
t=200 <div>
  <p>loading chart…</p>
</div>
t=600 attempts 2 <div>
  <p>chart</p>
</div>
```

- At 0 ms nothing is shown — the `delay` avoids a skeleton flash for fast loads.
- At 200 ms the loading component appears.
- The first attempt failed at 300 ms, `onError` called `retry()`, and the second attempt
  succeeded at 600 ms. The user never saw an error.

**Deploy-time chunk errors** deserve special mention. After a new deploy, a user with the
old app open still has the old main bundle, which references old chunk file names that no
longer exist on the server. Their next lazy load fails. Retrying doesn't help if the file is
gone; a common strategy is to catch this case (Vite dispatches a `vite:preloadError` event
on `window`) and reload the page once. Level 4 · 08 returns to this with caching headers.

## Async components inside `<Suspense>`

An async component inside a `<Suspense>` boundary is treated as an async dependency: the
boundary's fallback is shown instead of the component's own `loadingComponent`, and errors
propagate to the boundary's parent. Pass `suspensible: false` to opt out and use the
component's own options.

## What to split

Good candidates:

- **Routes** — everything except the landing route (Level 2 · 04). The biggest, easiest win.
- **Heavy widgets that aren't visible initially**: modals, drawers, tabs other than the
  first, "advanced" panels.
- **Large third-party dependencies used in one place**: charting, maps, rich-text editors,
  PDF generation, syntax highlighting. Import them *inside* the lazily-loaded component
  (or with `await import()` inside the function that needs them), so the library lands in
  the lazy chunk.

Poor candidates: small components used on most pages. Each chunk is a separate request
with its own overhead; splitting a 2 KB component costs more than it saves.

## Preloading: hiding the latency

Lazy loading trades bundle size for a delay at the moment of use. You can often hide the
delay by starting the load *before* it's needed:

```ts
const loadEditor = () => import('./RichEditor.vue')
const RichEditor = defineAsyncComponent(loadEditor)

// start downloading when the user shows intent, not when they click
function onEditHover() {
  loadEditor()   // the module is cached; the later render reuses the same promise
}
```

Idle time is another good moment: `requestIdleCallback(() => import('./SettingsView.vue'))`
after the first page renders. For routes, a `RouterLink` wrapper that calls the route
component's loader on `mouseenter`/`focus` gives most of the benefit of eager loading with
none of the upfront cost.

## Lazy hydration (SSR)

With server-side rendering (Level 4), the HTML for a component arrives with the page, but it
isn't interactive until its JavaScript loads and **hydrates** it. Vue 3.5 added hydration
strategies for async components, so below-the-fold or rarely-used parts of a
server-rendered page don't compete for the main thread on load:

```ts
import { defineAsyncComponent, hydrateOnVisible, hydrateOnIdle, hydrateOnInteraction } from 'vue'

const Comments = defineAsyncComponent({
  loader: () => import('./Comments.vue'),
  hydrate: hydrateOnVisible({ rootMargin: '200px' }),  // hydrate when scrolled near
})

const Footer = defineAsyncComponent({
  loader: () => import('./SiteFooter.vue'),
  hydrate: hydrateOnIdle(),                            // hydrate when the browser is idle
})

const ShareMenu = defineAsyncComponent({
  loader: () => import('./ShareMenu.vue'),
  hydrate: hydrateOnInteraction('click'),              // hydrate on first click, then replay it
})
```

The four built-in strategies exported by Vue 3.5.43 are `hydrateOnIdle`, `hydrateOnVisible`,
`hydrateOnMediaQuery` and `hydrateOnInteraction`. They only have an effect during
hydration of server-rendered HTML; in a client-only app the component just loads when it
renders.

## Reading your bundle

Vite prints every output file with its raw and gzip size after `vite build`. Make a habit
of reading it after adding a dependency. For a visual breakdown of what's inside each
chunk, bundle-visualiser plugins such as `rollup-plugin-visualizer` generate an
interactive treemap. Vite also warns when a chunk exceeds `build.chunkSizeWarningLimit`
(500 kB by default) — treat that warning as a prompt to look, not a rule.

## How It Actually Works

`defineAsyncComponent` returns a normal component (named `AsyncComponentWrapper` in
devtools) with a `setup` that:

1. calls the loader once and caches the resulting promise — every instance shares it;
2. if the component has already resolved, renders it immediately (so the second
   `<Heavy>` on the page never shows a loading state);
3. otherwise tracks `loaded`, `error` and `delayed` refs, starts the `delay` and `timeout`
   timers, and returns a render function that renders the loading component, error
   component or resolved component depending on those refs;
4. inside a Suspense boundary, returns the promise from `setup` instead, so the boundary
   waits for it.

`import('./Heavy.vue')` is where the bundler splits: every dynamic import with a static
string becomes a chunk boundary. Anything imported only by that chunk goes into it;
anything imported by several chunks is hoisted into a shared chunk. Vite's preload helper
then emits `<link rel="modulepreload">` for the chunk's dependencies when the import
starts, so shared chunks download in parallel instead of in a waterfall.

For hydration strategies, the wrapper renders nothing interactive at first. During
hydration, Vue calls the strategy with a `hydrate` callback; `hydrateOnVisible` sets up an
`IntersectionObserver` on the server-rendered DOM, and when it fires, the loader runs and
the component hydrates the existing DOM.

## Common mistakes

- **Statically importing the same module elsewhere** — if any eagerly-loaded file imports
  `Heavy.vue` normally, it lands in the main chunk and your async wrapper saves nothing.
  Check the build output.
- **Splitting everything** — dozens of tiny chunks add request overhead.
- **No error state** — lazy loads fail on real networks. Always handle errors.
- **Flashing loading states** — use `delay`, or preload on intent.
- **Calling `defineAsyncComponent` inside a component's `setup` or render** — it creates a
  new component type each render, causing remounts. Define it at module level.

## Exercise

1. In your Recipe Book, split the edit form into an async component with a skeleton loading
   component and an error component that has a "Retry" button. Check the build output for
   the new chunk.
2. Add a Markdown preview to the recipe steps using a Markdown library, imported only inside
   the preview component. Confirm with the build output that the library is not in the main
   chunk.
3. Throttle your network to "Slow 4G" in devtools and tune `delay` so the skeleton doesn't
   flash on fast loads but appears on slow ones.
4. Implement "preload on hover" for the edit link, and verify in the Network tab that the
   chunk starts downloading on hover rather than click.
