---
description: "Structure large Vue codebases: feature-based folders, public feature APIs, layered state and data access, composable conventions, and enforcing import boundaries in CI."
---

# 05 · Large-App Architecture

A 20-component app can be organised any way you like. A 500-component app worked on by
several teams for years cannot: without structure, every change touches everything, nobody
knows where code belongs, and the build and test suite slow to a crawl. This lesson is
about the structural decisions that keep a large Vue codebase changeable — with one small
tool you can run in CI to keep those decisions from eroding.

There's no single correct architecture. What follows is a set of defaults that work well
for most Vue SPAs, with the reasoning, so you can adapt them.

## Organise by feature, not by file type

The `create-vue` scaffold groups by **type**: `components/`, `views/`, `stores/`,
`composables/`. That's fine until each folder holds 150 unrelated files and a single
feature ("shopping list") is spread across six folders.

Group by **feature** instead, with a small shared layer:

```text
src/
├── app/                    app shell: main.ts, App.vue, router, global plugins
├── features/
│   ├── recipes/
│   │   ├── components/     RecipeCard.vue, RecipeForm.vue
│   │   ├── views/          RecipeListView.vue, RecipeDetailView.vue
│   │   ├── stores/         recipes.ts
│   │   ├── api/            recipes.ts (HTTP calls for this feature)
│   │   ├── routes.ts       this feature's route records
│   │   └── index.ts        the feature's PUBLIC API
│   └── shopping-list/
│       └── ...
└── shared/
    ├── ui/                 design-system components (or an @acme/ui package)
    ├── composables/        useDebouncedRef, useEventListener, ...
    ├── api/                http client, error types
    └── utils/
```

Benefits: a feature can be understood (and deleted) as a unit; teams can own features;
lazy-loading follows feature boundaries naturally (each `routes.ts` uses dynamic imports).

## Public APIs between features

The key rule: **other code may import a feature only through its `index.ts`.** Everything
else inside the folder is private.

```ts title="src/features/recipes/index.ts"
export { default as RecipeCard } from './components/RecipeCard.vue'
export { useRecipesStore } from './stores/recipes'
export { recipeRoutes } from './routes'
```

This gives the feature's owners freedom to restructure internals without breaking other
teams, and makes dependencies between features explicit and reviewable.

Two more dependency rules keep the graph healthy:

1. **`shared/` never imports from `features/`.** Shared code is below features in the
   layering; if it needs feature knowledge, it isn't shared.
2. **Features don't form cycles.** If `recipes` needs `shopping-list` and vice versa, extract
   the shared part into a third feature or into `shared/`.

## Enforcing the rules

Architecture rules that live only in a wiki erode within months. Check them in CI. Lint
plugins exist for this (for example `eslint-plugin-boundaries` or `import/no-restricted-paths`
rules), but the check is simple enough to write yourself — and writing it makes the rules
concrete:

```js title="scripts/check-boundaries.mjs"
// Enforces two rules on src/:
//  1. code outside a feature may import only that feature's public index
//  2. src/shared/ must not import from src/features/
import { readdir, readFile } from 'node:fs/promises'
import { join, relative, sep } from 'node:path'

const SRC = 'src'
const importRe = /(?:import|export)\s[^'"]*?from\s+['"](@\/[^'"]+)['"]/g

async function* files(dir) {
  for (const entry of await readdir(dir, { withFileTypes: true })) {
    const path = join(dir, entry.name)
    if (entry.isDirectory()) yield* files(path)
    else if (/\.(ts|vue)$/.test(entry.name)) yield path
  }
}

const featureOf = (path) => path.split(sep)[2] // src/features/<name>/...
const problems = []

for await (const file of files(SRC)) {
  const rel = relative(SRC, file)
  const code = await readFile(file, 'utf8')
  for (const [, spec] of code.matchAll(importRe)) {
    const target = spec.slice(2) // strip '@/'
    const [area, feature, ...rest] = target.split('/')
    if (area !== 'features') continue
    if (rel.startsWith(`shared${sep}`)) {
      problems.push(`${file}: shared code imports a feature (${spec})`)
    } else if (rest.length > 0 && featureOf(file) !== feature) {
      problems.push(`${file}: deep import into feature "${feature}" (${spec}) — use '@/features/${feature}'`)
    }
  }
}

if (problems.length) {
  console.error(problems.join('\n'))
  console.error(`\n${problems.length} boundary violation(s)`)
  process.exit(1)
}
console.log('boundaries OK')
```

We ran it on a sample tree with one legitimate import and two violations:

```ts title="src/features/shopping-list/stores/shoppingList.ts"
import { useRecipesStore } from '@/features/recipes'                 // ✓ public API
import RecipeCard from '@/features/recipes/components/RecipeCard.vue' // ✗ deep import
```

```ts title="src/shared/ui/BaseButton.ts"
import { useRecipesStore } from '@/features/recipes'   // ✗ shared must not depend on features
```

```text
src/features/shopping-list/stores/shoppingList.ts: deep import into feature "recipes" (@/features/recipes/components/RecipeCard.vue) — use '@/features/recipes'
src/shared/ui/BaseButton.ts: shared code imports a feature (@/features/recipes)

2 boundary violation(s)
exit code 1
```

Add it as `"lint:boundaries": "node scripts/check-boundaries.mjs"` and run it in CI. The
regex-based import scan is deliberately simple — it handles `import ... from '@/...'` and
`export ... from`, not dynamic `import()` or relative paths that escape a feature
(`../../other-feature`). A lint plugin handles those; this script is a starting point.

## Layering state

Large apps get into trouble by putting everything in global stores. Think in layers, from
most local to most global — and keep state at the lowest layer that works (the table in
Level 2 · 05):

| Layer | Holds | Lives in |
|-------|-------|----------|
| Component state | open/closed, input drafts, hover | `ref` in the component |
| URL state | filters, pagination, selected tab, ids | the route (Level 2 · 10) |
| Server cache | entities fetched from APIs | a data-fetching layer: feature stores or Pinia Colada |
| Client app state | auth session, preferences, feature flags | a few small Pinia stores |

Most "state management problems" in large Vue apps are server data kept in hand-rolled
global stores that drift out of sync with the server. Treat server data as a **cache** with
explicit invalidation (Level 2 · 07), not as app state you own.

## Data access

Keep HTTP details out of components and stores:

```text
component ──► store / query composable ──► feature api module ──► shared http client
```

- The **shared http client** handles base URL, auth headers, JSON parsing, typed errors,
  retries and timeouts — once.
- **Feature api modules** are thin typed functions: `getRecipe(id): Promise<Recipe>`.
  They're the natural place for response validation (Zod, Level 2 · 08) at the boundary,
  because the API is outside your type system.
- **Stores/composables** handle caching and state; **components** handle presentation.

The Recipe Book already followed this shape; its swappable `api/recipes.ts` is the
payoff.

## Conventions that scale

- **Composables return plain objects of refs and functions**, accept `MaybeRefOrGetter`
  inputs, and clean up with `onScopeDispose` (Level 2 · 01).
- **Components under ~200 lines of template+script.** When a component grows past that,
  it's usually doing two jobs; split presentation from data wiring.
- **Name files after their default export** (`RecipeCard.vue` exports `RecipeCard`), and
  use multi-word component names to avoid clashing with HTML elements.
- **TypeScript strictness** (`strict`, `noUncheckedIndexedAccess`) from day one — adding it
  later to a large codebase is a project in itself.
- **Type-checked templates** (`vue-tsc`) and tests in CI on every pull request.

## When one app isn't enough

Beyond a certain size, splitting into separately built units helps: a **monorepo** with
several apps and shared packages (the `@acme/ui` library from the last lesson, a shared API
client), managed with npm/pnpm workspaces and a task runner such as Turborepo or Nx.
**Micro-frontends** — separately deployed apps composed at runtime — solve an
*organisational* problem (independent team deploys) at a real technical cost (duplicated
dependencies, cross-app state, consistency). Reach for them only when independent
deployment is a hard requirement.

## How It Actually Works

Why do boundaries matter technically, not just organisationally?

- **Bundling**: Vite/Rolldown split chunks along the import graph. When a shared module
  imports a feature, that feature's code gets pulled into every chunk that uses the shared
  module — often into the main bundle. Clean layering produces clean chunks.
- **HMR**: when you edit a file, Vite invalidates it and walks *up* the importer graph until
  it finds modules that can accept the update (Vue SFCs can). A tangled graph makes that walk
  wider, and occasionally reaches the entry, forcing a full reload.
- **Type-checking and tests**: `vue-tsc --build` with project references and Vitest's
  related-test detection both follow imports. Fewer cross-feature edges mean less work per
  change.

## Common mistakes

- **One global store per API resource**, reimplementing a cache badly.
- **Deep imports into other features' internals.**
- **A `shared/` folder that imports features** — it becomes a second, hidden app.
- **Architecture rules without enforcement.**
- **Premature micro-frontends.**
- **Generic "utils" dumping grounds** — name modules after what they do.

## Exercise

1. Restructure your Recipe Book into `app/`, `features/recipes/` and `shared/`, with a
   public `index.ts` for the feature and route records contributed by the feature.
2. Add `check-boundaries.mjs` and wire it into `npm run lint`. Introduce a deep import on
   purpose and confirm CI would fail.
3. Extend the script to catch relative imports that leave a feature folder
   (`../../shopping-list/...`), with a test fixture for each rule.
4. Add a second feature (`shopping-list`: add a recipe's ingredients to a list) that uses
   only the recipes feature's public API. Which new exports did `recipes/index.ts` need?
