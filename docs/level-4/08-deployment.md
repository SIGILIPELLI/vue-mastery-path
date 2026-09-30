---
description: "Deploy Vue apps correctly: SPA history fallback, cache headers for hashed assets, base paths, stale-chunk recovery after releases, containers, SSR hosting and CI pipelines."
---

# 08 · Deployment

A Vite build is a folder of static files. Deploying it looks trivial — copy `dist/`
somewhere — and it mostly is, except for four things that break real apps: deep links
that 404, caches that serve yesterday's code, users stranded on an old version after a
release, and configuration that differs between environments. This lesson handles each
one, then covers containers, SSR hosting and a CI pipeline.

The demonstrations use the Recipe Book from Level 2 built with Vite 8.3.1 and served
locally with two servers: Python's `http.server` (a naive static server) and a small Node
server written below. Hosting-provider configs (Netlify, nginx, Docker) are shown as
configuration but were **not** run here — no container runtime or nginx was available on
the machine — so verify them against your provider's current docs.

## What the build contains

```text
dist/index.html                             0.42 kB │ gzip:  0.28 kB
dist/assets/RecipeEditView-lzIqcY5t.css     0.13 kB │ gzip:  0.13 kB
dist/assets/index-kfREwQ26.css              0.79 kB │ gzip:  0.42 kB
dist/assets/NotFoundView-CD-aGKFm.js        0.39 kB │ gzip:  0.29 kB
dist/assets/RecipeDetailView-DZtA2VHT.js    1.48 kB │ gzip:  0.83 kB
dist/assets/RecipeEditView-BW-Z2ntj.js      3.68 kB │ gzip:  1.60 kB
dist/assets/index-KmR4qrb9.js             103.36 kB │ gzip: 40.26 kB
```

Two kinds of file, which need opposite treatment:

- **`index.html`** — a fixed name, different content every release. It must never be
  cached without revalidation, or users keep loading old JavaScript.
- **`assets/*-[hash].*`** — the hash is derived from the file's content, so a given URL's
  content never changes. These can be cached "forever".

## Problem 1: deep links 404

The Recipe Book uses `createWebHistory()`, so URLs look like `/recipes/2`. There is no
`recipes/2` file on disk — the router handles that path in the browser, *after*
`index.html` has loaded. A naive static server doesn't know that:

```text
naive /recipes/2: 404
naive /         : 200
```

Clicking around from `/` works; refreshing or opening a shared link to `/recipes/2` fails.
The fix is a **history fallback**: serve `index.html` for any path that isn't a real file.

Here's a minimal production-shaped server that does it correctly, including caching:

```js title="serve.mjs"
import { createServer } from 'node:http'
import { readFile, stat } from 'node:fs/promises'
import { extname, join, normalize } from 'node:path'

const root = join(import.meta.dirname, 'dist')
const types = { '.html': 'text/html; charset=utf-8', '.js': 'text/javascript',
  '.css': 'text/css', '.ico': 'image/x-icon', '.svg': 'image/svg+xml' }

async function fileFor(urlPath) {
  const safe = normalize(decodeURIComponent(urlPath)).replace(/^(\.\.[/\\])+/, '')
  const p = join(root, safe)
  if (!p.startsWith(root)) return null           // no path traversal
  try { if ((await stat(p)).isFile()) return p } catch {}
  return null
}

createServer(async (req, res) => {
  const { pathname } = new URL(req.url, 'http://x')
  let file = await fileFor(pathname)
  if (!file) {
    // Missing asset → real 404. Anything else → the SPA shell.
    if (pathname.startsWith('/assets/') || extname(pathname)) {
      res.writeHead(404).end('Not found'); return
    }
    file = join(root, 'index.html')
  }
  const hashed = file.includes('/assets/')
  res.writeHead(200, {
    'content-type': types[extname(file)] ?? 'application/octet-stream',
    'cache-control': hashed ? 'public, max-age=31536000, immutable' : 'no-cache',
  })
  res.end(await readFile(file))
}).listen(process.env.PORT ?? 4173)
```

Observed status and `cache-control` for a range of paths:

```text
spa /:                              200 no-cache
spa /recipes/2:                     200 no-cache
spa /recipes/2/edit:                200 no-cache
spa /assets/index-KmR4qrb9.js:      200 public, max-age=31536000, immutable
spa /assets/old-chunk-abc123.js:    404
spa /favicon.ico:                   200 no-cache
spa /missing.png:                   404
```

The **missing-asset rule** matters. A fallback that returns `index.html` for *everything*
answers a request for a deleted JavaScript chunk with HTML and status 200; the browser then
fails with a confusing MIME-type or syntax error instead of a clean 404 you can detect.

The same logic in common hosts (not run here — check current syntax):

```text title="Netlify: public/_redirects"
/*    /index.html   200
```

```nginx title="nginx"
location /assets/ {
  add_header Cache-Control "public, max-age=31536000, immutable";
  try_files $uri =404;
}
location / {
  add_header Cache-Control "no-cache";
  try_files $uri $uri/ /index.html;
}
```

GitHub Pages has no rewrite rules; the usual workaround is copying `index.html` to
`404.html`, which works for users but serves deep links with a 404 status (bad for SEO). If
that matters, use hash history or prerender the routes.

Note that with a fallback, **every** unknown path returns 200 — your in-app "not found"
route is a soft 404. If search engines index the site, prerender or server-render instead
(Lessons 01–02), or at least add `<meta name="robots" content="noindex">` on the not-found
view.

## Problem 2: caches

`no-cache` does **not** mean "don't cache"; it means "revalidate before using". The
browser keeps `index.html` but asks the server each time (cheaply, with `ETag` /
`If-None-Match`), so a new release is picked up on the next navigation. `immutable` with
a one-year `max-age` tells the browser not to even revalidate hashed assets.

A CDN in front adds a second cache. Configure it to respect origin headers, or set the same
rules there, and purge (or let expire quickly) `index.html` on each deploy. Never let a
CDN cache `index.html` for hours.

## Problem 3: users on the old version

A user opens the app at 10:00. You deploy at 10:05; the old chunks are deleted. At 10:10
they click "Edit", and the router lazy-loads `RecipeEditView-BW-Z2ntj.js` — which now
404s. The navigation fails.

Two complementary fixes:

1. **Keep the previous release's assets** for a while (deploy new files alongside old
   ones, delete after some days). Hashes make that safe: filenames never collide.
2. **Recover in the app.** Vite dispatches a `vite:preloadError` event when a dynamic
   import fails:

   ```ts title="src/main.ts"
   window.addEventListener('vite:preloadError', (event) => {
     event.preventDefault()               // we handle it
     const key = 'reloaded-for-new-version'
     if (!sessionStorage.getItem(key)) {  // avoid reload loops
       sessionStorage.setItem(key, '1')
       window.location.reload()
     }
   })
   ```

   Vue Router also reports the failure through `router.onError`, where you can do a full
   navigation to the target URL (`window.location.assign(to.fullPath)`) so the user lands
   where they wanted, on the new version. We haven't simulated a live release race in a
   browser here; test it by deploying, keeping a tab open, deploying again, and clicking.

## Problem 4: configuration per environment

Level 4 · 03 showed that `import.meta.env.VITE_*` values are **baked in at build time**.
That forces one build per environment — and "test the build, then promote the same build
to production" becomes impossible.

For values that differ per environment (API URL, feature flags, error-reporting DSN), load
them at runtime:

```ts title="src/config.ts"
export interface AppConfig { apiBase: string; sentryDsn?: string }

export async function loadConfig(): Promise<AppConfig> {
  const res = await fetch('/config.json', { cache: 'no-cache' })
  if (!res.ok) throw new Error(`config.json: HTTP ${res.status}`)
  return res.json()
}
```

```ts title="src/main.ts"
const config = await loadConfig()
createApp(App).provide('config', config).use(router).mount('#app')
```

Each environment serves its own `config.json` (written by the deploy script or mounted into
the container). Remember: it's public. Secrets never go in the frontend, runtime or not.

## Containers

A multi-stage Dockerfile builds with Node and serves with a small static server, so the
final image contains no toolchain (not built here — verify with your registry and base
images):

```dockerfile title="Dockerfile"
FROM node:24-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
```

Copying the lockfile before the source lets Docker cache the `npm ci` layer when only
code changes. Pin base images to a version (and ideally a digest) you've tested.

## SSR deployments

Nuxt (or the hand-built SSR server from Lesson 01) needs a **running process**, not just
files: `node .output/server/index.mjs` behind a reverse proxy, a serverless function, or an
edge runtime via a Nitro preset. Extra concerns compared with static hosting:

- **Health checks and graceful shutdown** so the platform can replace instances.
- **Memory per request** — SSR renders on your server's CPU; load-test it.
- **Per-request state** — any accidental module-level state leaks between users (Lesson
  02).
- **Caching rendered HTML** — only for pages that aren't personalised.

## A CI pipeline

```yaml title=".github/workflows/deploy.yml"
name: deploy
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24, cache: npm }
      - run: npm ci
      - run: npm run type-check
      - run: npm run test:unit -- --run
      - run: npm run build-only
      - uses: actions/upload-artifact@v4
        with: { name: dist, path: dist }
  # deploy job: download the artifact and upload it to your host
```

The important properties: **the artifact that passed tests is the one that ships**, the
deploy is automatic from `main`, and rolling back means redeploying the previous artifact.
Action versions change over time; check the current major for each.

## How It Actually Works

- **Why fallback is needed:** with `createWebHistory`, the router calls
  `history.pushState` to change the URL without a request. Only on refresh or a new tab
  does the browser actually request `/recipes/2` from the server — which has to answer with
  the app shell so the router can take over.
- **Why hashes make caching safe:** Rolldown computes each chunk's filename from its
  content, and a chunk's content includes the filenames of the chunks it imports. We
  changed one string in `RecipeEditView.vue` and rebuilt: `RecipeEditView-*.js` got a new
  hash, so did `index-*.js` (it references the edit view's filename), and so did
  `NotFoundView` and `RecipeDetailView` — they import shared code from the renamed
  `index` chunk. Both CSS files kept their names. The cascade costs some cache hits, but
  it's what guarantees consistency: the browser's cache key is the URL, so an old chunk can
  never be paired with a new one.
- **Revalidation:** with `no-cache`, the browser sends `If-None-Match` with the stored
  `ETag`; an unchanged file gets `304 Not Modified` with no body. That's why `no-cache` on
  `index.html` costs almost nothing.
- **Why old chunks 404:** a running tab keeps the old `index-*.js` in memory, which
  contains the old chunk names for lazy routes. Nothing reloads it until the user does.

## Common mistakes

- **No history fallback** — refreshes and shared links 404.
- **Fallback for everything, including `/assets/`** — missing chunks return HTML with 200.
- **Long-caching `index.html`** at the browser or CDN.
- **Deleting old assets on every deploy** without in-app recovery.
- **Building per environment** because config is baked in — prefer a runtime `config.json`.
- **Wrong `base`** when hosting under a sub-path (`/app/`): assets load from `/` and 404.
  Set `base` in `vite.config.ts` and pass `import.meta.env.BASE_URL` to the router.
- **Secrets in `VITE_*` or `config.json`** — both are public.

## Exercise

1. Build your Recipe Book and serve it with `python3 -m http.server`. Reproduce the deep
   link 404. Then serve it with `serve.mjs` and confirm the status codes and headers above.
2. Change one view, rebuild, and compare `dist/assets` before and after. Which filenames
   changed? Explain each one using the import graph (in our run, every JS chunk changed
   and both CSS files didn't).
3. Simulate the stale-version problem: open the app, rebuild with a changed lazy route,
   delete the old chunk, then navigate to that route in the still-open tab. Add the
   `vite:preloadError` handler and repeat.
4. Replace one `VITE_` variable with `config.json` loaded at startup, and write a test for
   `loadConfig` that covers a non-OK response.
5. Write a GitHub Actions workflow that builds once, uploads the artifact, and deploys it
   to GitHub Pages with the `404.html` workaround. What status code does a deep link get?
