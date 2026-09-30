---
description: "Level 2 of Vue.js: composables, provide/inject, Vue Router, Pinia, async data, form validation, TypeScript and a routed recipe-book project."
---

# Level 2 · Intermediate <span class="level-badge">Building apps</span>

Goal: go from single-screen components to a real multi-page application — shared logic,
shared state, URLs, server data, validated forms and types you can trust — and know which
tool fits which problem.

Level 1 was about how one component works. Level 2 is about how many components *share*
things, and it gives you a decision table you'll use for the rest of your Vue career:

| Need to share… | Reach for | Lesson |
|----------------|-----------|--------|
| Logic, with separate state per use | a composable | 01 |
| State within one component subtree | provide / inject | 02 |
| "Where am I?" — the current page and its parameters | the router (and the URL) | 03, 04 |
| State across unrelated parts of the app | a Pinia store | 05, 06 |
| Server data, cached and deduplicated | a store or a data-fetching library | 07 |

## Modules

1. [Composables](01-composables.md) — reusable stateful logic, flexible inputs with `toValue`, cleanup, and a race-safe `useFetch`
2. [Provide & Inject](02-provide-inject.md) — typed injection keys, readonly state, and a compound Tabs component
3. [Vue Router Basics](03-vue-router-basics.md) — routes, links, params as props, component reuse, nested routes, 404s
4. [Router in Depth: Guards, Meta, File Routes](04-router-in-depth.md) — guard order (observed), typed meta, lazy routes, scroll, Vue Router 5 file-based routing
5. [Pinia Fundamentals](05-pinia-fundamentals.md) — setup vs option stores, `storeToRefs`, `$patch`, `$reset`, when to use a store
6. [Pinia in Depth](06-pinia-in-depth.md) — `$subscribe`, `$onAction`, a persistence plugin, HMR, testing stores and components
7. [Async Data & API Calls](07-async-data.md) — typed HTTP errors, the four UI states, caching and dedupe, Suspense, Pinia Colada
8. [Form Validation](08-form-validation.md) — a tested `useForm`, accessible errors, server errors, Zod schemas
9. [TypeScript with Vue](09-typescript-with-vue.md) — typing macros and refs, generic components, and what `vue-tsc` catches
10. [Project — A Recipe Book](10-project-recipe-book.md) — router + Pinia + validation + tests in one app

## Before you start

You should be comfortable with everything in [Level 1](../level-1/index.md), especially
`ref`/`computed`/`watch`, props and emits, and `v-model` on components. The projects use
`async`/`await` and `fetch` throughout; the
[JavaScript Mastery Path](https://sigilipelli.github.io/javascript-mastery-path/) covers
promises if you need a refresher, and the
[REST API Mastery Path](https://sigilipelli.github.io/rest-api-mastery-path/) explains the
HTTP status codes and API design this level assumes.

## What was verified, and how

Every store, composable and component in this level with a test was run with Vitest 4.1
against Vue 3.5.43, Vue Router 5.3.1 and Pinia 4.0.3, and type-checked with `vue-tsc`
3.3.11. Several lessons show a test that **failed first** — Pinia plugins not running
without an app, a lazy route not loaded yet, a type error in a generic composable — because
you'll meet the same failures. Third-party libraries mentioned but not installed (for
example VeeValidate) are marked as such; check their current docs.

!!! note "Vue Router 5 and Pinia 4"
    Both were released in 2026 and are deliberately low-drama majors: Vue Router 5 merged
    file-based routing into the core without breaking the classic API, and Pinia 4's
    breaking changes are packaging (ESM-only, devtools API v8). Tutorials written for
    Vue Router 4 and Pinia 2/3 still apply almost unchanged.
