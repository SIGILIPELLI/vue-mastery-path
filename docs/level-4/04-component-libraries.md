---
description: "Build and publish a Vue component library: API design, Vite library mode with Vue external, type declarations with vue-tsc, package exports, CSS and theming tokens."
---

# 04 · Component Libraries & Design Systems

Once several apps (or several teams) share buttons, form fields and dialogs, copying
components between repositories stops scaling. A **component library** packages them once,
with a stable API, types and styles, and apps install it like any dependency. This lesson
builds a small library, `@acme/ui`, packs it, installs it into a separate app and checks
that types and styles arrive intact.

## Designing the component API

A library component's props, events and slots are a public API — changing them breaks
consumers. Some principles:

- **Small, predictable props with union types** (`variant: 'primary' | 'secondary' |
  'danger'`) rather than booleans that can conflict (`primary`, `danger` both true?).
- **Pass through native attributes.** A `Button` should accept `type`, `aria-*`, `form`,
  `@click` without declaring each — fallthrough attrs (Level 1 · 06) do this for free.
- **Slots for content, props for configuration.** Labels and icons go in slots; the library
  doesn't guess what apps will put there.
- **Accessibility built in** — labels wired up, `aria-invalid`, focus styles, reduced
  motion. Every app that uses the library inherits it.
- **Theming through CSS custom properties**, not props: apps change `--acme-color-primary`
  once instead of passing colours everywhere.

## The library

Two components.

```vue title="src/components/AcmeButton.vue"
<script setup lang="ts">
export interface AcmeButtonProps {
  variant?: 'primary' | 'secondary' | 'danger'
  size?: 'sm' | 'md'
  loading?: boolean
}
const { variant = 'primary', size = 'md', loading = false } = defineProps<AcmeButtonProps>()
</script>

<template>
  <button
    class="acme-btn"
    :class="[`acme-btn--${variant}`, `acme-btn--${size}`]"
    :disabled="loading || undefined"
    :aria-busy="loading || undefined"
  >
    <span v-if="loading" class="acme-btn__spinner" aria-hidden="true" />
    <slot />
  </button>
</template>

<style>
.acme-btn {
  --acme-btn-bg: var(--acme-color-primary, #2563eb);
  display: inline-flex; align-items: center; gap: 0.4rem;
  border: 0; border-radius: var(--acme-radius, 6px);
  background: var(--acme-btn-bg); color: white; font: inherit; cursor: pointer;
}
.acme-btn--secondary { --acme-btn-bg: var(--acme-color-secondary, #4b5563); }
.acme-btn--danger { --acme-btn-bg: var(--acme-color-danger, #b91c1c); }
.acme-btn--sm { padding: 0.25rem 0.6rem; font-size: 0.875rem; }
.acme-btn--md { padding: 0.5rem 1rem; }
.acme-btn:disabled { opacity: 0.6; cursor: not-allowed; }
.acme-btn__spinner {
  width: 0.9em; height: 0.9em; border: 2px solid currentColor; border-right-color: transparent;
  border-radius: 50%; animation: acme-spin 0.7s linear infinite;
}
@keyframes acme-spin { to { transform: rotate(360deg); } }
@media (prefers-reduced-motion: reduce) { .acme-btn__spinner { animation-duration: 2s; } }
</style>
```

The button exposes `loading` as both a visual spinner and `aria-busy`, disables itself
while loading, and reads every colour from a custom property with a fallback. Styles use
BEM-like `acme-` prefixes and **no** `scoped`: library CSS must be overridable by apps, and
scoped attribute selectors make overrides awkward. The prefix prevents collisions instead.

```vue title="src/components/AcmeField.vue"
<script setup lang="ts">
import { useId } from 'vue'

defineOptions({ inheritAttrs: false })
const { label, error, hint } = defineProps<{ label: string; error?: string; hint?: string }>()
const model = defineModel<string>({ default: '' })
const id = useId()
</script>

<template>
  <div class="acme-field" :class="{ 'acme-field--invalid': error }">
    <label :for="id">{{ label }}</label>
    <input
      :id="id"
      v-model="model"
      v-bind="$attrs"
      :aria-invalid="!!error || undefined"
      :aria-describedby="error || hint ? `${id}-msg` : undefined"
    />
    <p v-if="error || hint" :id="`${id}-msg`" class="acme-field__msg">{{ error ?? hint }}</p>
  </div>
</template>

<style>
.acme-field { display: grid; gap: 0.25rem; }
.acme-field input { padding: 0.45rem 0.6rem; border: 1px solid #9ca3af; border-radius: var(--acme-radius, 6px); font: inherit; }
.acme-field--invalid input { border-color: var(--acme-color-danger, #b91c1c); }
.acme-field__msg { margin: 0; font-size: 0.875rem; color: #4b5563; }
.acme-field--invalid .acme-field__msg { color: var(--acme-color-danger, #b91c1c); }
</style>
```

The field generates its own ids with `useId()`, connects the message with
`aria-describedby`, and forwards attributes such as `type`, `autocomplete` and `required`
to the `<input>` (not the wrapper) with `inheritAttrs: false`.

```ts title="src/index.ts"
import type { App } from 'vue'
import AcmeButton from './components/AcmeButton.vue'
import AcmeField from './components/AcmeField.vue'

export { AcmeButton, AcmeField }
export type { AcmeButtonProps } from './components/AcmeButton.vue'

/** Optional: register every component globally with app.use(AcmeUI). */
export default {
  install(app: App) {
    app.component('AcmeButton', AcmeButton)
    app.component('AcmeField', AcmeField)
  },
}
```

Named exports let apps import only what they use (tree-shaking); the default export is an
optional plugin for apps that prefer global registration.

## Building in library mode

```ts title="vite.config.ts"
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  build: {
    lib: {
      entry: fileURLToPath(new URL('./src/index.ts', import.meta.url)),
      formats: ['es'],
      fileName: 'acme-ui',
    },
    rolldownOptions: {
      external: ['vue'],            // the app provides Vue — never bundle a second copy
    },
  },
})
```

- **`build.lib`** switches Vite from "build an app with an `index.html`" to "build a
  package from an entry file".
- **`external: ['vue']`** is essential. Without it, Vue's runtime is bundled into the
  library, and the app ends up with **two** copies of Vue — double the size, and subtle
  breakage because reactivity and `provide`/`inject` don't work across copies.
- **ES modules only** (`formats: ['es']`) is enough for modern bundlers.

Types come from `vue-tsc` in declaration-only mode:

```json title="tsconfig.build.json"
{
  "extends": "@vue/tsconfig/tsconfig.dom.json",
  "include": [
    "src/**/*.ts",
    "src/**/*.vue"
  ],
  "compilerOptions": {
    "noEmit": false,
    "declaration": true,
    "emitDeclarationOnly": true,
    "outDir": "dist",
    "rootDir": "src"
  }
}
```

Our first attempt produced **no `.d.ts` files at all**, with exit code 0. The shared
`@vue/tsconfig` base sets `"noEmit": true` (right for apps, which only type-check), and it
overrode our `emitDeclarationOnly`. Adding `"noEmit": false` fixed it.

```json title="package.json"
{
  "name": "@acme/ui",
  "version": "0.1.0",
  "type": "module",
  "files": ["dist"],
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/acme-ui.js"
    },
    "./style.css": "./dist/acme-ui.css"
  },
  "sideEffects": ["**/*.css"],
  "peerDependencies": {
    "vue": "^3.5.0"
  },
  "scripts": {
    "build": "vite build && vue-tsc -p tsconfig.build.json"
  }
}
```

- **`exports`** defines exactly what consumers can import: the JS entry with its types, and
  the stylesheet as `@acme/ui/style.css`. Anything not listed is private.
- **`peerDependencies`** declares Vue without installing a copy — the app's Vue is used.
- **`sideEffects`** tells bundlers the JS is side-effect-free (safe to tree-shake) but CSS
  imports must be kept.
- **`files`** limits the published package to `dist`.

The build:

```text
dist/acme-ui.css  1.13 kB │ gzip: 0.50 kB
dist/acme-ui.js   2.02 kB │ gzip: 1.00 kB
```

2 kB of JavaScript for two components — because Vue isn't in it. The first line of the
bundle imports from `vue` rather than containing it:

```js
import { createCommentVNode as e, createElementBlock as t, createElementVNode as n, defineComponent as r, mergeModels as i, mergeProps as a, normalizeClass as o, openBlock as s, renderSlot as c, toDisplayString as l, unref as u, useId as d, useModel as f, vModelDynamic as p, withDirectives as m } from "vue";
```

`vue-tsc` generated `dist/index.d.ts` and a `.vue.d.ts` per component, including the props
interface:

```ts
export interface AcmeButtonProps {
    variant?: 'primary' | 'secondary' | 'danger';
    size?: 'sm' | 'md';
    loading?: boolean;
}
```

## Consuming it

We packed the library (`npm pack` → `acme-ui-0.1.0.tgz`, 2.4 kB, 6 files) and installed the
tarball into a separate Vite app — the same thing that happens with a registry install:

```vue title="consumer: src/App.vue"
<script setup lang="ts">
import { ref } from 'vue'
import { AcmeButton, AcmeField } from '@acme/ui'
import '@acme/ui/style.css'

const email = ref('')
const saving = ref(false)
</script>

<template>
  <AcmeField v-model="email" label="Email" type="email" hint="We never share it." />
  <AcmeButton variant="danger" :loading="saving" @click="saving = true">Delete account</AcmeButton>
  <AcmeButton variant="ghost">Oops</AcmeButton>
</template>
```

`vue-tsc` in the **consumer** caught the invalid variant using the library's published
types:

```text
src/App.vue(13,15): error TS2322: Type '"ghost"' is not assignable to type '"danger" | "primary" | "secondary" | undefined'.
```

`npm ls vue` in the consumer showed a single `vue@3.5.43`, with the library's usage
`deduped` — one copy, as intended. Theming is plain CSS in the app:

```css
:root {
  --acme-color-primary: #0f766e;
  --acme-radius: 999px;
}
```

## Development workflow

- **A playground or Storybook.** Develop components in isolation with every variant
  visible. Storybook and Histoire are common choices; a plain Vite app that imports from
  `src/` works too.
- **Component tests** for behaviour and accessibility (Level 3 · 07–08) — the library is
  where they pay off most.
- **Versioning.** Follow semantic versioning strictly: removing a prop or renaming a slot
  is a major version. Tools such as Changesets automate changelogs and version bumps.
- **Monorepos.** When the library lives next to its apps, npm/pnpm workspaces let apps
  depend on the local package (`"@acme/ui": "workspace:*"` with pnpm) without publishing.

## Buy vs build

Before building a full library, look at existing ones. PrimeVue, Vuetify, Quasar, Element
Plus and Naive UI are full-featured; Reka UI (formerly Radix Vue) and Headless UI provide
unstyled, accessible behaviour you style yourself. A common pattern is building a thin
design-system layer (your tokens, your API) on top of headless primitives. We haven't
evaluated these libraries' current versions for this course; check their accessibility
documentation and Vue 3.5 support before choosing.

## How It Actually Works

In library mode, Vite uses your entry file instead of HTML as the bundle input, emits
modules in the requested format, and extracts all CSS imported by the components into one
stylesheet (unless you configure otherwise). `external` tells Rolldown to leave matching
imports as `import ... from "vue"` in the output rather than following them. When the app
bundles the library, its own bundler resolves `"vue"` to the app's copy — which is exactly
why the peer dependency matters: if the library listed Vue as a regular dependency with a
different version range, npm could install a nested second copy that the library's
`import "vue"` would resolve to.

`vue-tsc` in declaration mode runs the same virtual-file transformation described in
Level 2 · 09 and asks TypeScript to emit `.d.ts` files for the generated code. That's why
the declarations contain names such as `__VLS_WithSlots` — they're the helper types
`vue-tsc` uses to describe props, emits and slots.

## Common mistakes

- **Bundling Vue into the library.**
- **Vue as a `dependency` instead of a `peerDependency`.**
- **Scoped styles in library components** that apps can't override.
- **No `exports` map**, letting apps deep-import private files you'll later move.
- **Breaking prop or slot names in a minor release.**
- **Forgetting the CSS import** in the app — components render unstyled.
- **Trusting a clean exit code** — check that `dist/` actually contains the files you
  expect (we got none the first time).

## Exercise

1. Build `@acme/ui` with the two components and add a third: `AcmeDialog`, wrapping
   `<dialog>` with `v-model:open`, a title slot and focus return.
2. Add a `playground/` Vite app that imports from `../src` and shows every variant of every
   component.
3. Pack the library and install it into your Recipe Book. Replace the Recipe Book's buttons
   and fields with library components, and theme them with custom properties.
4. Make a breaking change (rename `variant` to `tone`) and write the changelog entry and
   migration note a consumer would need.
