---
description: "Scaffold a Vue 3 app with create-vue, understand every generated file, and use Vite's dev server, production build and type checking."
---

# 02 · create-vue, Vite & Project Structure

The official way to start a Vue application is **create-vue**, a scaffolding tool that
generates a Vite-based project with the features you pick. This lesson walks through
creating a project, what each generated file is for, and what actually happens when you run
the dev server and the production build.

## Creating a project

You need Node.js. The generated `package.json` declares
`"node": "^22.18.0 || >=24.12.0"`, so check with `node -v` first.

Interactive mode asks you a series of questions:

```bash
npm create vue@latest
```

You can also pass feature flags to skip the prompts. This is the setup used for the
rest of the course — TypeScript, Vue Router, Pinia and Vitest:

```bash
npm create vue@latest habit-tracker -- --ts --router --pinia --vitest
cd habit-tracker
npm install
npm run dev
```

The `--` separates npm's own arguments from the ones passed to create-vue. Other flags
include `--eslint`, `--prettier`, `--playwright`, `--cypress` and `--bare` (skip the
example components). `npm create vue@latest -- --help` lists them all.

## What gets generated

With the flags above, create-vue 3.24 produced this tree (icons and CSS trimmed):

```text
habit-tracker/
├── index.html
├── package.json
├── vite.config.ts
├── vitest.config.ts
├── env.d.ts
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
├── tsconfig.vitest.json
├── public/
│   └── favicon.ico
└── src/
    ├── main.ts
    ├── App.vue
    ├── assets/            base.css, main.css, logo.svg
    ├── components/        HelloWorld.vue, TheWelcome.vue, ...
    │   └── __tests__/HelloWorld.spec.ts
    ├── router/index.ts
    ├── stores/counter.ts
    └── views/             HomeView.vue, AboutView.vue
```

### `index.html` — the real entry point

With Vite, `index.html` lives at the project root, not in `public/`. It contains a
`<div id="app">` and a module script:

```html
<script type="module" src="/src/main.ts"></script>
```

Vite treats that script tag as the entry to your dependency graph. Everything else is
reached by following imports from `main.ts`.

### `src/main.ts` — creating the app

```ts title="src/main.ts"
import './assets/main.css'

import { createApp } from 'vue'
import { createPinia } from 'pinia'

import App from './App.vue'
import router from './router'

const app = createApp(App)

app.use(createPinia())
app.use(router)

app.mount('#app')
```

`createApp(App)` creates an **application instance** with `App` as the root component.
`app.use()` installs plugins — here the Pinia store and the router, which each register
things the rest of the app can reach (Level 2). `mount('#app')` renders `App` into the
element with that id. Nothing appears on screen until `mount` runs.

You can create more than one app instance on a page; each has its own plugins, global
components and config. That is how Vue can control several independent widgets on a
server-rendered page.

### `src/App.vue` — the root component

The generated `App.vue` renders a header with `<RouterLink>`s and a `<RouterView />`,
which is where the component for the current URL appears. You'll replace its content in
Lesson 10.

### `vite.config.ts`

```ts title="vite.config.ts"
import { fileURLToPath, URL } from 'node:url'

import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import vueDevTools from 'vite-plugin-vue-devtools'

export default defineConfig({
  plugins: [
    vue(),
    vueDevTools(),
  ],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
})
```

- `@vitejs/plugin-vue` is what teaches Vite to compile `.vue` files.
- `vite-plugin-vue-devtools` adds an in-page devtools panel during `npm run dev`
  (toggle it with the floating button at the bottom of the page). It is not included in
  production builds.
- The `@` alias lets you write `import X from '@/components/X.vue'` instead of long
  relative paths. The same alias is mirrored in `tsconfig.app.json` under `paths` so the
  type checker agrees with the bundler.

### The four `tsconfig` files

`tsconfig.json` has no settings of its own; it just references three projects:

| File | Covers | Why separate |
|------|--------|--------------|
| `tsconfig.app.json` | `src/**`, `env.d.ts` | Browser code: DOM types, `@/*` paths |
| `tsconfig.node.json` | `vite.config.*`, `vitest.config.*` | Runs in Node: Node types, no DOM |
| `tsconfig.vitest.json` | `src/**/__tests__/*` | Tests run in jsdom with Node globals |

The generated `tsconfig.app.json` also turns on `noUncheckedIndexedAccess`, which makes
`items[0]` typed as `Item | undefined`. It is stricter than many tutorials assume, and it
catches real bugs; keep it.

### `package.json` scripts

```json title="package.json (scripts)"
{
  "dev": "vite",
  "build": "run-p type-check \"build-only {@}\" --",
  "preview": "vite preview",
  "test:unit": "vitest",
  "build-only": "vite build",
  "type-check": "vue-tsc --build"
}
```

Note that `build` runs **two things in parallel**: `vue-tsc --build` (type checking) and
`vite build` (bundling). Vite itself does *not* type-check — it strips types and moves on.
If you only run `vite build`, type errors will not stop the build.

## Running it

`npm run dev` starts the dev server (by default on `http://localhost:5173/`). Edit a
`.vue` file and the change appears without a page reload, keeping component state where
possible — that is **Hot Module Replacement** (HMR).

`npm run build` produced this on our machine (Vite 8.3.1, the untouched scaffold):

```text
vite v8.3.1 building client environment for production...
transforming...
✓ 44 modules transformed.
rendering chunks...
computing gzip size...
dist/index.html                       0.42 kB │ gzip:  0.28 kB
dist/assets/AboutView-CXtZgaLf.css    0.08 kB │ gzip:  0.10 kB
dist/assets/index-DEm2-gV0.css        4.05 kB │ gzip:  1.27 kB
dist/assets/AboutView-Cpcj6SP6.js     0.22 kB │ gzip:  0.20 kB
dist/assets/index-D9-VCYkA.js       100.24 kB │ gzip: 38.76 kB

✓ built in 264ms
```

Two JavaScript files came out: the main bundle, and a tiny separate `AboutView` chunk.
That is because the router config imports `AboutView` lazily with
`() => import('../views/AboutView.vue')` — Vite turns every dynamic `import()` into its own
chunk that loads only when that route is visited. The hashes in the file names change
whenever the content changes, which lets you cache them forever (Level 4 · 08).

`npm run preview` serves the built `dist/` folder locally so you can check the production
output before deploying. It is not a production server.

`npm run test:unit` runs Vitest in watch mode; `npx vitest run` runs once and exits:

```text
 Test Files  1 passed (1)
      Tests  1 passed (1)
```

## How It Actually Works

In **development**, Vite does not bundle your app. The browser requests `/src/main.ts`
directly; Vite intercepts the request, transforms the file (TypeScript → JavaScript,
`.vue` → JavaScript module) and serves it as a native ES module. The browser then follows
`import` statements and requests each dependency the same way. Only the files the current
page actually imports get transformed, which is why startup is fast regardless of project
size.

Third-party packages in `node_modules` are handled differently: Vite **pre-bundles** them
once (the `node_modules/.vite` folder) so that a package made of hundreds of small files,
or published as CommonJS, becomes a single ES module the browser can load efficiently.

A `.vue` file becomes several virtual modules. `@vitejs/plugin-vue` parses the file with
`@vue/compiler-sfc`, compiles `<script setup>` into a normal component definition,
compiles the template into a render function, and turns each `<style>` block into a CSS
import (with scoped-style attribute rewriting applied). When you edit only the template,
the plugin sends an HMR update that swaps the render function and re-renders the component
while keeping its state — which is why HMR feels instantaneous.

In **production**, `vite build` uses a real bundler (Rolldown, in Vite 8) to produce
minified, tree-shaken, content-hashed files. Vue's production build flags are applied too:
dev-only warnings and devtools hooks are stripped, which is part of why the dev and prod
bundles behave slightly differently (for example, `[Vue warn]` messages only appear in dev).

## Common mistakes

- **Putting `index.html` in `public/`.** Files in `public/` are copied verbatim and never
  processed. `index.html` must be at the root so Vite can transform it.
- **Relying on `vite build` for type safety.** It doesn't check types. Use `npm run build`
  (which includes `vue-tsc`) locally and in CI.
- **Importing from `public/` with a path.** Reference public files by absolute URL
  (`/favicon.ico`), never `import x from '../public/x.png'`. Assets you import from `src/`
  get hashed and optimised; public ones don't.
- **Editing `node_modules/.vite`.** It's a cache. If dependencies behave strangely after an
  upgrade, delete it and restart the dev server.
- **Opening `dist/index.html` from the file system.** Module scripts don't load over
  `file://` and absolute asset paths break. Use `npm run preview`.

## Exercise

1. Create a project with `npm create vue@latest my-first-vue -- --ts --router --pinia --vitest`
   and run it.
2. Change the text in `src/components/HelloWorld.vue` while the dev server is running and
   watch it update without a reload. Then click the counter-free parts of the page and
   confirm the browser tab didn't refresh (the devtools Network tab shows no new document
   request).
3. Run `npm run build` and write down how many JS chunks you get. Change the `about` route
   in `src/router/index.ts` to import `AboutView` statically at the top of the file and
   build again. How many chunks now, and why?
4. Introduce a deliberate type error (`const n: number = 'x'`) in `App.vue`. Compare what
   `npx vite build` and `npm run build` do.
