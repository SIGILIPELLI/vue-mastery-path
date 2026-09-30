---
description: "Level 1 of Vue.js: tooling, templates, ref/reactive/computed, watchers, props and emits, v-model, slots, styling, and a habit-tracker project."
---

# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: build a small but complete Vue application from an empty folder, and be able to
explain what Vue is doing at every step — why the screen updated when you assigned to a
ref, why a destructured value stopped updating, where a child's event went, and what the
compiler did with your template.

Vue's surface area is small at this level, and that's the point. Three ideas carry you
through all ten modules:

1. **State is reactive, and Vue tracks what reads it.** Wrap values in `ref()` or
   `reactive()`, derive values with `computed()`, and Vue re-renders exactly the components
   that read something that changed. Most "it doesn't update" bugs come from breaking that
   tracking (destructuring, replacing proxies, mutating copies).
2. **Data flows down, events flow up.** Parents pass props; children emit events.
   `v-model` is just those two halves bundled together.
3. **Templates are compiled.** Directives like `v-if`, `v-for` and `@click` are
   instructions to a compiler, which produces optimised render functions. Knowing roughly
   what it produces explains keys, scoped CSS and slot behaviour.

## Modules

1. [What Vue Is](01-what-vue-is.md) — the progressive framework, SFCs, Composition vs Options API, and versions
2. [create-vue, Vite & Project Structure](02-tooling-project-structure.md) — every generated file, dev server, build output and type checking
3. [Template Syntax & Directives](03-template-syntax.md) — interpolation, `v-bind`, `v-on` and modifiers, `v-if`/`v-show`, `v-for` and keys
4. [Reactivity: ref, reactive, computed](04-reactivity-ref-reactive-computed.md) — reactive values, unwrapping, destructuring pitfalls, cached derived state
5. [Watchers & Lifecycle Hooks](05-watchers-lifecycle.md) — `watch`, `watchEffect`, batching, flush timing, cleanup and the component lifecycle
6. [Components: Props & Emits](06-props-emits.md) — typed props, reactive destructure, events, fallthrough attributes
7. [Forms & v-model](07-forms-v-model.md) — every input type, modifiers, and `defineModel` for custom components
8. [Slots](08-slots.md) — default, named and scoped slots, typed with `defineSlots`
9. [Styling & Transitions](09-styling-transitions.md) — class/style bindings, scoped CSS, CSS modules, `v-bind()` in CSS, `<Transition>`
10. [Project — A Habit Tracker](10-project-habit-tracker.md) — a tested app with streaks, persistence and animated lists

## What you need before starting

- **Node.js** 22.18+ or 24.12+ (the range `create-vue` declares). Check with `node -v`.
- **JavaScript fundamentals**: modules, arrow functions, array methods (`map`, `filter`,
  `reduce`), destructuring, promises. If any of those are shaky, work through
  [JavaScript Mastery Path](https://sigilipelli.github.io/javascript-mastery-path/) first.
- **Basic TypeScript** helps: this course uses `lang="ts"` everywhere, but mostly for
  simple annotations like `ref<string | null>(null)` and prop types. If types are new,
  keep [TypeScript Mastery Path](https://sigilipelli.github.io/typescript-mastery-path/)
  open alongside.
- **HTML and CSS basics** — forms, labels, selectors, flexbox.
- A terminal and an editor. VS Code with the official **Vue (Official)** extension gives
  you template type checking and autocompletion.

No accounts, cloud services or paid tools are needed anywhere in Levels 1–3.

## How the code is organized

Code blocks are titled with a path relative to the project root, such as
`src/components/StarRating.vue`. Blocks labelled `text` are terminal or console output
captured from real runs on **Vue 3.5.43, Vite 8.3.1 and Vitest 4.1.11 with Node.js 26**
while this course was written. Where our first attempt failed — a type error, a test
that couldn't find `localStorage` — the lesson shows that too, because those failures are
what you'll hit in practice.

!!! note "About versions"
    Vue 3 has kept its core API stable since 2020, and minor releases add features rather
    than remove them. The newest APIs used in this level are marked with the version that
    introduced them: `defineModel` (3.4), reactive props destructure, `useTemplateRef` and
    `onWatcherCleanup` (3.5). Run `npm ls vue` to see your version. Vue 3.6, which
    introduces the opt-in Vapor Mode, was in release-candidate stage at the time of
    writing; it doesn't change anything taught here.
