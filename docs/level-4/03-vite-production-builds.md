---
description: "Configure Vite 8 production builds for Vue: env files and modes, what gets exposed to the client, base paths, define, source maps, browser targets and chunk splitting."
---

# 03 · Vite & Production Builds

`npm run build` has worked since Level 1 without any configuration. Production apps need a
few deliberate decisions on top: different settings per environment, a base path when the
app isn't served from `/`, version information baked in, source maps for error reports,
and sensible chunking. This lesson covers each one with builds we ran on **Vite 8.3.1**
(which bundles with **Rolldown** 1.2) and shows what ended up in the output.

## Environment variables and modes

Vite loads variables from `.env` files in the project root:

| File | Loaded when |
|------|-------------|
| `.env` | always |
| `.env.local` | always, git-ignored — for your machine only |
| `.env.[mode]` | only in that mode (`.env.production`, `.env.staging`) |
| `.env.[mode].local` | only in that mode, git-ignored |

More specific files win. The **mode** is `development` for `vite` (dev server) and
`production` for `vite build`, unless you pass `--mode`.

We set up a test project with:

```ini title=".env"
VITE_API_BASE=/api
VITE_APP_TITLE=Recipe Book
DATABASE_PASSWORD=do-not-ship-me
```

```ini title=".env.staging"
VITE_API_BASE=https://staging-api.example.com
```

```ini title=".env.local"
VITE_APP_TITLE=Recipe Book (local)
```

and a `main.ts` that logs what it can see:

```ts title="src/main.ts"
console.log(JSON.stringify({
  mode: import.meta.env.MODE,
  prod: import.meta.env.PROD,
  base: import.meta.env.BASE_URL,
  api: import.meta.env.VITE_API_BASE,
  title: import.meta.env.VITE_APP_TITLE,
  secret: (import.meta.env as Record<string, unknown>).DATABASE_PASSWORD ?? null,
  version: __APP_VERSION__,
}))
if (import.meta.env.DEV) console.log('dev-only branch')
```

After `vite build`, the end of the output file (Vite's small modulepreload polyfill comes
first) was:

```js
console.log(JSON.stringify({mode:`production`,prod:!0,base:`/recipes/`,api:`/api`,title:`Recipe Book (local)`,secret:null,version:`1.4.0`}));
```

After `vite build --mode staging`:

```js
console.log(JSON.stringify({mode:`staging`,prod:!0,base:`/recipes/`,api:`https://staging-api.example.com`,title:`Recipe Book (local)`,secret:null,version:`1.4.0`}));
```

What this shows:

- **Only `VITE_`-prefixed variables reach client code.** `DATABASE_PASSWORD` came out as
  `null`, and searching the bundle for its value found nothing (`contains password: false`).
  The prefix is a safety net, not a secrets manager: **anything with `VITE_` is public** —
  it's pasted into JavaScript anyone can read. API keys that must stay secret belong on a
  server.
- **Values are replaced at build time**, as literal strings. There's no runtime lookup;
  changing `.env` means rebuilding. If you need one build deployed to several environments,
  load a `/config.json` at startup instead.
- **`--mode staging` is still a production build** (`prod: true`, minified) with staging
  values. Mode and "production-ness" are separate concepts.
- **`.env.local` overrode `.env`** in both builds — which is why it must never be
  committed, or your local values end up in CI builds.
- **Dead code is removed.** The `if (import.meta.env.DEV)` branch doesn't appear at all:
  `DEV` became `false` and the minifier dropped the unreachable code.

Env variables also work in `index.html` with `%VITE_APP_TITLE%`; the built page had
`<title>Recipe Book (local)</title>`.

For TypeScript, declare your variables so typos are caught:

```ts title="env.d.ts"
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_BASE: string
  readonly VITE_APP_TITLE: string
}
```

## The config for that build

```ts title="vite.config.ts"
import { defineConfig } from 'vite'

export default defineConfig({
  base: '/recipes/',
  define: { __APP_VERSION__: JSON.stringify('1.4.0') },
  build: { sourcemap: 'hidden' },
})
```

### `base`

Set `base` when the app is served from a sub-path — GitHub Pages project sites
(`/<repo>/`), an app mounted under `/admin/` behind a reverse proxy. Every asset URL is
rewritten: the built HTML referenced `/recipes/assets/index-D2estjeq.js`. Use
`import.meta.env.BASE_URL` for anything you construct at runtime, including
`createWebHistory(import.meta.env.BASE_URL)` so the router knows its prefix.

### `define`

`define` replaces global identifiers with constants at build time. Always wrap values in
`JSON.stringify` — `define` inserts code, not strings. A common use is the app version
(read `package.json`'s version in the config) so error reports can say which release
failed. Declare it for TypeScript with `declare const __APP_VERSION__: string`.

### Source maps

Minified code is unreadable in stack traces. Source maps map it back to your source.
Options:

| `build.sourcemap` | Map file | `//# sourceMappingURL` comment | Use |
|-------------------|----------|-------------------------------|-----|
| `false` (default) | no | no | no maps at all |
| `true` | yes | yes | browsers load maps automatically; your source is public |
| `'hidden'` | yes | no | upload maps to your error tracker; don't serve them publicly |
| `'inline'` | embedded | — | development only; bloats bundles |

Our `'hidden'` build produced `index-D2estjeq.js.map` next to the bundle, and the bundle
had no `sourceMappingURL` comment. Upload the map files to your error-reporting service
during deployment (Level 4 · 09), and don't deploy them to the public web server if you
don't want your source readable.

### Browser targets

Vite 8's default `build.target` is `'baseline-widely-available'` — browsers that support
features the Baseline project has marked "widely available" (roughly, features supported
across major browsers for at least 30 months). Modern syntax is left as-is for those
browsers. If you must support older browsers, set a lower target and consider
`@vitejs/plugin-legacy`; check your real audience's browsers first.

## Chunk splitting

Vite splits automatically at every dynamic `import()` and extracts shared code. Sometimes
you want explicit groups — for example a `vendor` chunk for third-party code, which
changes less often than your app code and can stay cached across deploys.

In Vite 8, bundling options live under `build.rolldownOptions` (`rollupOptions` is kept as a
deprecated alias). Rolldown's `manualChunks` is also deprecated in favour of
`codeSplitting.groups`:

```ts title="vite.config.ts (excerpt)"
build: {
  rolldownOptions: {
    output: {
      codeSplitting: {
        groups: [{ name: 'vendor', test: /node_modules/ }],
      },
    },
  },
},
```

Applied to the Level 2 Recipe Book, the 103 kB main chunk split into app and vendor code:

```text
dist-split/assets/NotFoundView-C1Ek359c.js       0.43 kB │ gzip:  0.31 kB
dist-split/assets/RecipeDetailView-CRr09nK-.js   1.52 kB │ gzip:  0.85 kB
dist-split/assets/RecipeEditView-cW8B1cGN.js     3.72 kB │ gzip:  1.62 kB
dist-split/assets/index-BJRUrnso.js              7.94 kB │ gzip:  3.62 kB
dist-split/assets/vendor-2B0q-k03.js            95.68 kB │ gzip: 37.07 kB
```

Now a change to the app's own code only changes the 8 kB `index` chunk; returning visitors
keep the cached 96 kB vendor chunk. Whether that's worth an extra request depends on how
often you deploy — measure rather than assume. Don't create dozens of tiny groups.

If you're upgrading a Vite 5–7 project, `build.rollupOptions.output.manualChunks` still
works in Vite 8 but is marked deprecated in the type definitions; plan the move to
`codeSplitting`.

## Previewing and checking a build

- `vite preview` serves `dist/` locally with the right `base`.
- Read the size table after every dependency change.
- Search the bundle for things that must not be there: `grep -r "do-not-ship" dist/` is a
  cheap CI check for leaked secrets or debug code.

## How It Actually Works

`vite build` runs your Vite plugins through Rolldown, a bundler written in Rust with a
Rollup-compatible plugin API. Environment handling happens before bundling: Vite reads the
`.env` files for the mode (using the `dotenv` format, with `dotenv-expand` for `${VAR}`
references), filters to variables starting with `envPrefix` (default `VITE_`), and builds a
`define`-style replacement table where `import.meta.env.VITE_API_BASE` becomes the literal
`"/api"`. Those replacements are plain text substitution at the AST level, which is why the
minifier can then evaluate `if (false)` and drop the branch.

Chunking works on the module graph: every entry and every dynamic import starts a chunk,
modules reachable from only one chunk go into it, and modules shared by several are
extracted into shared chunks. `codeSplitting.groups` adds your own rules on top: modules
whose id matches `test` are pulled into the named group. Content hashes in file names are
computed from each chunk's final content, so an unchanged chunk keeps its name across
builds — the property that makes long-term caching (Level 4 · 08) work.

## Common mistakes

- **Secrets in `VITE_` variables** — they're public.
- **Expecting to change env values after the build.** They're compiled in.
- **Committing `.env.local`.**
- **Forgetting `base`** for sub-path deployments — the page loads, but every asset 404s.
- **`sourcemap: true` in production** when you didn't intend to publish your source.
- **Many hand-made chunk groups** that defeat caching and add requests.

## Exercise

1. Add `.env.development`, `.env.staging` and `.env.production` to your Recipe Book with a
   different `VITE_API_BASE` in each, typed in `env.d.ts`. Build each mode and confirm the
   value with `grep` on `dist/`.
2. Inject the version from `package.json` with `define` and show it in the footer.
3. Deploy (or `vite preview`) with `base: '/recipes/'`, and fix everything that breaks —
   router history, links to `public/` files, anything with a hard-coded `/`.
4. Add a CI step that fails if `dist/` contains the string `localhost` or any value from
   your non-`VITE_` variables.
