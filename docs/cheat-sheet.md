---
title: "Vue.js Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Vue.js cheat sheet: 19 quick-reference tables from every level of the Vue.js Mastery Path, each linked to its full lesson."
---

# Vue.js Cheat Sheet

Quick-reference tables from **19 lessons** of the free Vue.js Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · What Vue Is](level-1/01-what-vue-is.md)

*What ships in "Vue" and what is separate*

| Need | Package | Covered in |
|------|---------|-----------|
| Client-side routing | `vue-router` | Level 2 · 03–04 |
| Shared state | `pinia` | Level 2 · 05–06 |
| Build tooling & dev server | `vite` + `@vitejs/plugin-vue` | Level 1 · 02, Level 4 · 03 |
| Component testing | `@vue/test-utils` (with Vitest) | Level 3 · 07 |
| Type checking `.vue` files | `vue-tsc` | Level 2 · 09 |
| Full-stack meta-framework | `nuxt` (community-led, closely aligned) | Level 4 · 02 |

### [02 · create-vue, Vite &amp; Project Structure](level-1/02-tooling-project-structure.md)

*The four `tsconfig` files*

| File | Covers | Why separate |
|------|--------|--------------|
| `tsconfig.app.json` | `src/**`, `env.d.ts` | Browser code: DOM types, `@/*` paths |
| `tsconfig.node.json` | `vite.config.*`, `vitest.config.*` | Runs in Node: Node types, no DOM |
| `tsconfig.vitest.json` | `src/**/__tests__/*` | Tests run in jsdom with Node globals |

### [03 · Template Syntax &amp; Directives](level-1/03-template-syntax.md)

*Conditional rendering: `v-if` and `v-show`*

| | `v-if` | `v-show` |
|---|---|---|
| When false | element is **not in the DOM** | element is in the DOM with `display: none` |
| Toggle cost | creates/destroys elements and child components | flips one CSS property |
| Initial cost when false | nothing rendered | still rendered |
| Child component state | lost when removed | kept |
| Works on `<template>` | yes | no |

### [05 · Watchers &amp; Lifecycle Hooks](level-1/05-watchers-lifecycle.md)

*Batching: many changes, one callback*

| `flush` | Runs | Typical use |
|---------|------|-------------|
| `'pre'` (default) | before the component re-renders | update other state |
| `'post'` | after the DOM has updated | read the updated DOM, measure elements |
| `'sync'` | immediately on every write | rarely: debugging, integration code |

### [07 · Forms &amp; v-model](level-1/07-forms-v-model.md)

*Every input type*

| Element | Bound state | Event listened to |
|---------|-------------|-------------------|
| text input, textarea | string | `input` |
| single checkbox | boolean (or custom `true-value`/`false-value`) | `change` |
| checkboxes sharing an array | array of the checked `value`s | `change` |
| radio group | the checked radio's `value` | `change` |
| `<select>` | the selected option's `value` | `change` |
| `<select multiple>` | array of selected values | `change` |

### [09 · Styling &amp; Transitions](level-1/09-styling-transitions.md)

*Transitions*

| Class | Applied |
|-------|---------|
| `toast-enter-from` | first frame of enter, removed one frame later |
| `toast-enter-active` | the whole enter phase — put `transition` here |
| `toast-enter-to` | from the second frame until the enter finishes |
| `toast-leave-from` | first frame of leave |
| `toast-leave-active` | the whole leave phase |
| `toast-leave-to` | from the second frame until the leave finishes |

### [10 · Project — A Habit Tracker](level-1/10-project-habit-tracker.md)

| Feature | Where |
|---------|-------|
| `ref`, `computed`, deep `watch` | `App.vue` state, summary, persistence |
| Props, emits, `defineModel` | `HabitRow`, `DayCell`, `HabitForm` |
| `v-for` with stable keys, `v-if`/`v-else` | the habit list and empty state |
| Form handling with `v-model` and `.prevent` | `HabitForm` |
| Scoped CSS, `v-bind()` in CSS | per-habit colours in `DayCell` |
| `<TransitionGroup>` | rows animate in and out |

## Level 2 · Intermediate

### [03 · Vue Router Basics](level-2/03-vue-router-basics.md)

*History modes*

| Function | URLs look like | Server needs |
|----------|----------------|--------------|
| `createWebHistory()` | `/users/42` | to serve `index.html` for every unknown path (Level 4 · 08) |
| `createWebHashHistory()` | `/#/users/42` | nothing — works on any static host |
| `createMemoryHistory()` | (not reflected in the address bar) | used for SSR and tests |

### [04 · Router in Depth: Guards, Meta, File Routes](level-2/04-router-in-depth.md)

*Navigation guards*

| Return | Effect |
|--------|--------|
| `undefined` or `true` | continue |
| `false` | cancel; stay on the current URL |
| a route location (`'/login'`, `{ name: 'login', query: {...} }`) | redirect |
| throw an error | cancel and call `router.onError` handlers |

### [05 · Pinia Fundamentals](level-2/05-pinia-fundamentals.md)

*When to use a store (and when not)*

| Situation | Use |
|-----------|-----|
| State used by one component | `ref` in that component |
| Logic reused across components, state separate each time | a composable |
| State shared by a component subtree (a form and its fields) | provide/inject |
| State shared across unrelated parts of the app, or across routes | a Pinia store |
| Server data (lists, details, caching, refetching) | a store, or a data-fetching library such as Pinia Colada (Lesson 07) |

### [10 · Project — A Recipe Book](level-2/10-project-recipe-book.md)

| Feature | Lesson |
|---------|--------|
| Routes with params as props, lazy views, 404, titles, scroll | 03, 04 |
| `onBeforeRouteLeave` for unsaved changes | 04 |
| Setup store with a reactive `Map` cache | 05, 07 |
| API layer that can be swapped for real `fetch` calls | 07 |
| Race-safe loading keyed on a prop | 01, 07 |
| Validated form with accessible errors, `defineExpose` | 08, 09 |
| Store, component and whole-app tests | 06 |

## Level 3 · Advanced

### [03 · Custom Directives &amp; Plugins](level-3/03-directives-plugins.md)

*Directive hooks*

| Hook | Called |
|------|--------|
| `created` | before the element's attributes and listeners are applied |
| `beforeMount` | before the element is inserted |
| `mounted` | after the element and its children are in the DOM |
| `beforeUpdate` / `updated` | before / after the containing component updates |
| `beforeUnmount` / `unmounted` | before / after the element is removed |

### [07 · Testing with Vitest &amp; Vue Test Utils](level-3/07-testing.md)

*What to test*

| Layer | Tool | What it catches |
|-------|------|-----------------|
| Pure functions, composables, stores | Vitest (no DOM needed for most) | logic errors, edge cases |
| Components | Vitest + Vue Test Utils + jsdom | rendering, events, props/emits, states |
| Whole flows in a real browser | Playwright (or Cypress) | routing, real CSS/layout, integration with a real backend |

### [08 · Accessibility in Vue Apps](level-3/08-accessibility.md)

*Semantic HTML first*

| Instead of | Use | You get |
|------------|-----|---------|
| `<div @click>` | `<button type="button">` | focus, Enter/Space activation, "button" role |
| `<span @click="router.push(...)">` | `<RouterLink>` (an `<a href>`) | open in new tab, "link" role, visited state |
| placeholder-only inputs | `<label for>` + `<input id>` | a name that doesn't vanish when typing |
| a custom modal `<div>` | `<dialog>` with `showModal()` | focus trap, Escape, inert background |
| `<div class="h1">` | real `<h1>`–`<h6>` in order | navigable document outline |

### [10 · Project — A Kanban Board](level-3/10-project-kanban.md)

| Concept | Where | Lesson |
|---------|-------|--------|
| Derived state with stable ordering | `byColumn` computed | 01 |
| `TransitionGroup` move animations | `KanbanColumn` | 04 |
| Native drag and drop + keyboard alternative | `KanbanCard`, `KanbanColumn` | 08 |
| Live-region announcements, focus restoration | `App.vue` | 08 |
| Rule violations as errors, surfaced to the user | store + `run()` in `App.vue` | 09 |
| Native `<dialog>` editing a copy | `CardDialog` | 04, 08 |
| Store and full-app tests | `__tests__/` | 07 |

## Level 4 · Master

### [03 · Vite &amp; Production Builds](level-4/03-vite-production-builds.md)

*Environment variables and modes*

| File | Loaded when |
|------|-------------|
| `.env` | always |
| `.env.local` | always, git-ignored — for your machine only |
| `.env.[mode]` | only in that mode (`.env.production`, `.env.staging`) |
| `.env.[mode].local` | only in that mode, git-ignored |

### [05 · Large-App Architecture](level-4/05-large-app-architecture.md)

*Layering state*

| Layer | Holds | Lives in |
|-------|-------|----------|
| Component state | open/closed, input drafts, hover | `ref` in the component |
| URL state | filters, pagination, selected tab, ids | the route (Level 2 · 10) |
| Server cache | entities fetched from APIs | a data-fetching layer: feature stores or Pinia Colada |
| Client app state | auth session, preferences, feature flags | a few small Pinia stores |

### [09 · Observability &amp; Production Readiness](level-4/09-production-readiness.md)

*What a useful error report contains*

| Question | Field |
|----------|-------|
| What broke? | message, stack |
| Where in the app? | route, component, Vue's `info` (render, watcher, event handler…) |
| Which code? | **release** (version or commit SHA) — matches the source map to use |
| How often, for how many users? | a session identifier, timestamp — counted by the backend |
| What led up to it? | optional breadcrumbs: recent navigations, clicks, failed requests |

### [10 · Capstone — A Production Vue App](level-4/10-capstone.md)

| Concern | Where | Lesson |
|---------|-------|--------|
| SSR, payload, per-route rendering | pages, `routeRules` | 01, 02 |
| Validated server API with correct status codes | `server/api/`, `server/utils/rsvp.ts` | 02, Level 3 · 09 |
| Accessible form with field errors and a live status | `RsvpForm.vue` | Level 2 · 08, Level 3 · 08 |
| Hydration-safe date formatting | `app/utils/format.ts` | 01, 07 |
| Release-tagged client error reporting | `monitoring.client.ts`, `client-errors.post.ts` | 09 |
| Runtime vs build-time configuration | `runtimeConfig`, `/about` | 03, 08 |
| No-cache API headers, custom error page | `routeRules`, `error.vue` | 08 |
| Unit tests and a post-deploy smoke test | `tests/`, `scripts/smoke.mjs` | Level 3 · 07, 08 |

---

Go deeper with the full [Vue.js Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
