---
description: "Level 3 of Vue.js: reactivity and rendering internals, directives and plugins, built-ins, code splitting, performance, testing, accessibility, security and a Kanban project."
---

# Level 3 · Advanced <span class="level-badge">Under the hood</span>

Goal: understand Vue well enough to predict its behaviour instead of discovering it — and
use that understanding to make apps faster, more robust, accessible and secure.

Levels 1 and 2 gave you rules: don't destructure reactive objects, keep keys stable, pass
getters to composables. Level 3 shows **why** each rule exists by opening the reactivity
system and the renderer, then applies that knowledge to the concerns that separate a
working app from a production-quality one.

## Modules

1. [Reactivity in Depth](01-reactivity-in-depth.md) — track/trigger, a mini implementation, shallow and raw state, `readonly`, `customRef`, computed stability, effect scopes
2. [The Rendering Mechanism & Render Functions](02-rendering-mechanism.md) — virtual DOM, patch flags, blocks, keyed diffing, `h()` and functional components
3. [Custom Directives & Plugins](03-directives-plugins.md) — `v-autofocus`, `v-click-outside`, and a typed toast plugin
4. [Teleport, KeepAlive, Transition & Suspense](04-built-in-components.md) — observed behaviour of each built-in and how the renderer implements them
5. [Async Components & Code Splitting](05-async-components-splitting.md) — `defineAsyncComponent`, retry, preloading, lazy hydration
6. [Performance](06-performance.md) — measured re-render counts, props stability, `v-memo`, virtual scrolling, load performance
7. [Testing with Vitest & Vue Test Utils](07-testing.md) — behaviour-focused tests, async control, mocks, stubs, coverage — and two traps we hit
8. [Accessibility in Vue Apps](08-accessibility.md) — route announcements and focus, accessible widgets, `useId`, axe and its limits
9. [Error Handling & Security](09-errors-security.md) — error routing and boundaries, XSS sinks, `v-html`, URL validation, CSP
10. [Project — A Kanban Board](10-project-kanban.md) — drag and drop with a keyboard alternative, WIP limits, animations, dialog editing, tests

## Before you start

You should have built the Level 2 project or something similar: a routed app with a Pinia
store and tests. This level assumes you're comfortable reading TypeScript generics and
writing Vitest tests.

Some lessons print compiler output or runtime internals. You don't need to memorise them;
the point is to see that the "magic" is ordinary JavaScript — proxies, functions and
objects — that you can reason about.

## How this level was verified

Everything with observable output was run against **Vue 3.5.43** in Vitest 4.1.11 with
jsdom: reactivity experiments, compiler output from `@vue/compiler-dom` and
`@vue/compiler-sfc`, re-render counts, the directives, the plugin, the error boundary,
axe-core checks and the Kanban project. Several lessons document failures we hit and how
we diagnosed them, because the diagnosis is the transferable skill.

Where jsdom can't answer the question — real layout, real drag gestures, screen reader
output, timing on real hardware — the lesson says so, and the exercises ask you to check it
in a browser. Performance numbers are deliberately given as **counts of work done** (renders,
DOM rows) rather than milliseconds, because timings depend on your machine.
