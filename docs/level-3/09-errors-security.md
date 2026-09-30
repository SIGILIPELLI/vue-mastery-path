---
description: "Handle Vue errors with app.config.errorHandler and error boundaries, and defend against XSS: escaping, v-html sanitising, unsafe URLs, template injection and CSP."
---

# 09 · Error Handling & Security

Two topics that share a theme: things going wrong at runtime. The first half covers
**errors** — where Vue catches them, how to contain a failure to one part of the page, and
what Vue can't catch. The second covers **security** — which attacks Vue's templates
prevent automatically and which ones you must handle yourself.

## Where errors go

Vue wraps the code it calls for you — setup functions, render functions, lifecycle hooks,
watcher callbacks, event handlers bound in templates — and routes any error to a chain of
handlers. We registered a global handler and triggered errors from three places:

```ts
app.config.errorHandler = (err, instance, info) => {
  seen.push(`${(err as Error).message} | info: ${info}`)
}
```

```text
in sync handler | info: native event handler
in watcher | info: watcher callback
in async handler | info: native event handler
```

Note the third line: an `async` click handler that threw **after an `await`** was still
caught, because Vue checks whether a handler returns a promise and attaches a rejection
handler to it. The `info` string tells you where the error came from (in production builds
it's a short code instead, to save bytes).

What Vue **doesn't** see: errors in code it didn't call — a `setTimeout` callback, a
`fetch().then()` chain you didn't return, a third-party library's event callback. Those go
to the browser's `window.onerror` / `unhandledrejection`. A production app needs both
(Level 4 · 09 wires them to an error-reporting service).

## Error boundaries with `onErrorCaptured`

Before reaching the global handler, an error travels **up the component tree**. Any
ancestor can intercept it with `onErrorCaptured`, and returning `false` stops it going
further. That's enough to build an error boundary — a component that shows a fallback
instead of its broken children, so one failing widget doesn't take down the page:

```vue title="src/security/ErrorBoundary.vue"
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const { label = 'This section' } = defineProps<{ label?: string }>()
const error = ref<Error | null>(null)

onErrorCaptured((err, _instance, info) => {
  error.value = err instanceof Error ? err : new Error(String(err))
  console.error(`[${label}] ${info}:`, err)   // a real app reports this (Level 4 · 09)
  return false                                 // stop propagation: we handled it
})

function retry() {
  error.value = null   // re-renders the slot, remounting the failed subtree
}
</script>

<template>
  <div v-if="error" role="alert" class="error-boundary">
    <p>{{ label }} couldn't be displayed.</p>
    <button type="button" @click="retry">Try again</button>
  </div>
  <slot v-else />
</template>
```

```vue-html
<ErrorBoundary label="Revenue chart">
  <RevenueChart :data="report" />
</ErrorBoundary>
```

We wrapped a component whose render threw, next to other page content, then fixed the
cause and pressed "Try again":

```text
sync: rest of page
after tick: Revenue chart couldn't be displayed.Try againrest of page
after retry: chart okrest of page
```

Two observations:

- Immediately after mounting, the boundary's slot rendered nothing and the fallback hadn't
  appeared yet. The error was captured *during* the render; setting `error.value` schedules
  the boundary's re-render for the next tick. In tests, `await nextTick()` before asserting.
- The rest of the page rendered normally throughout.

Where to put boundaries: around independent widgets (charts, feeds, third-party
embeds) and around each route's view, so a crash in one page still leaves navigation
working. Remember that `<Suspense>` async errors surface through `onErrorCaptured` too
(Lesson 04).

`onErrorCaptured` runs for errors from **descendants**, not for errors in the component
itself.

## Security: what Vue protects you from

### Interpolation is escaped

We rendered an attacker-controlled string three ways:

```ts
const evil = '<img src=x onerror="alert(1)">'
const url = 'javascript:alert(document.cookie)'
```

```vue-html
<p>{{ evil }}</p>
<div v-html="evil"></div>
<a :href="url">profile</a>
```

```html
<p>&lt;img src=x onerror="alert(1)"&gt;</p>
<div><img src="x" onerror="alert(1)"></div>
<a href="javascript:alert(document.cookie)">profile</a>
```

- `{{ }}` (and `:textContent`, attribute bindings) are escaped — the markup shows as text.
  This is the default for everything you render, and it's why Vue apps are safe from the
  most common XSS by default.
- `v-html` inserts **raw HTML**. The `onerror` handler is live; in a browser, it would run.
- A bound `href` is **not validated**. Vue sets exactly what you give it; clicking that
  link runs the script in your page's origin.

### Sanitise HTML you must render

If you need to render user-provided or CMS HTML (a comment, a rich-text field), sanitise it
first with a well-maintained sanitiser such as DOMPurify (3.4.16 here):

```ts
import DOMPurify from 'dompurify'
const safe = DOMPurify.sanitize(userHtml)
```

```text
<p>Hi <b>there</b><img src="x"><a>x</a></p>
```

from the input
`<p>Hi <b>there</b><img src=x onerror="alert(1)"><script>alert(2)</script><a href="javascript:alert(3)">x</a></p>`:
the event handler, the script and the `javascript:` URL were removed; the harmless markup
stayed. Better still, store Markdown and render it with a Markdown library configured to
disallow raw HTML — then sanitise the output anyway.

### Validate URLs by protocol

For user-supplied links (profile websites, "return to" parameters), allow-list protocols:

```ts title="src/security/safeUrl.ts"
const ALLOWED = new Set(['http:', 'https:', 'mailto:'])

/** Returns the URL if it uses an allowed protocol, otherwise a harmless fallback. */
export function safeUrl(input: string, fallback = '#'): string {
  try {
    const url = new URL(input, window.location.origin)   // relative URLs resolve against us
    return ALLOWED.has(url.protocol) ? url.href : fallback
  } catch {
    return fallback
  }
}
```

```text
"https://example.com/a?b=1" -> https://example.com/a?b=1
"/profile/42" -> http://localhost:3000/profile/42
"javascript:alert(1)" -> #
" JaVaScRiPt:alert(1)" -> #
"data:text/html,<script>alert(1)</script>" -> #
"mailto:a@b.co" -> mailto:a@b.co
```

Parsing with `URL` rather than checking `startsWith('javascript:')` handles case tricks and
leading whitespace, which the second line shows. (The relative path resolved against our
test environment's origin, `http://localhost:3000`.) For redirects after login — like the
`?redirect=` from Level 2 · 04 — go further and only accept **same-origin paths**, or an
attacker can send users from your login page to a phishing site.

### Never compile user content as a template

Vue's runtime compiler (in builds that include it) will execute any expression in a
template string. `createApp({ template: userInput })` or rendering user content inside a
server-rendered template that Vue then mounts over lets an attacker run arbitrary
JavaScript: `{{ constructor.constructor('alert(1)')() }}`-style payloads are the classic
example. Templates must come from your source code, never from users. If you server-render
HTML with another language (PHP, Django, Rails) and mount Vue on it, make sure user content
in that HTML can't contain `{{ }}` that Vue will compile — or use `v-pre` on those regions.

### Styles and other sinks

- `:style` bindings with user input can be abused for UI redressing (overlaying fake
  buttons). Allow-list values.
- Don't bind user input to `:is` (component name) or to attributes like `srcdoc`.
- Server-rendered state serialised into HTML (`window.__STATE__ = ...`) must be escaped so a
  `</script>` inside data can't break out — Level 4 · 01 shows how.

### Content Security Policy

A CSP header such as `script-src 'self'` blocks inline scripts and `eval`, turning many XSS
bugs into non-events. SFC-based builds (everything in this course) work under a strict CSP
because templates are precompiled. The full build with the runtime template compiler needs
`'unsafe-eval'` — one more reason to precompile.

## How It Actually Works

Vue's error routing is `handleError(err, instance, type)`. Every place the runtime calls
user code goes through `callWithErrorHandling` (sync) or `callWithAsyncErrorHandling`
(which also attaches `.catch` to returned promises — hence the async handler result).
`handleError` walks `instance.parent` upward, calling each ancestor's `errorCaptured` hooks;
if one returns `false`, it stops. Otherwise it calls `app.config.errorHandler`, and if there
isn't one, falls back to logging the error to the console.

Escaping is equally mechanical. Interpolations compile to text children set with
`textContent`/text nodes, which the browser never parses as HTML. Attribute bindings use
`setAttribute`, which doesn't execute anything — but a `javascript:` URL is *valid data*
for `href`, and it's the browser that executes it when clicked. `v-html` compiles to setting
`innerHTML`, the one place parsing happens. So the rule is: HTML parsing and URL navigation
are the sinks; Vue guards the first by default except in `v-html`, and never guards the
second.

## Common mistakes

- **No global error handler** in production — errors vanish into users' consoles.
- **`onErrorCaptured` without returning `false`** — the error keeps propagating, so it's
  handled twice: once by your boundary and again by ancestors and the global handler.
- **Trusting `v-html` content because "it comes from our API"** — the API stored what a user
  typed.
- **Validating URLs with string prefixes.**
- **Open redirects** via `?redirect=` parameters.
- **Rendering user content inside Vue-mounted server templates.**
- **Tokens in `localStorage`** — any XSS can read them. Prefer `HttpOnly` cookies set by the
  server, with CSRF protection.

## Exercise

1. Add a global `errorHandler` to your Recipe Book that shows a toast (Lesson 03) and logs
   the `info` string. Throw deliberately from a watcher, a click handler and a
   `setTimeout`; note which ones reach it, and add a `window` `unhandledrejection` listener
   for the rest.
2. Wrap each route view in `ErrorBoundary` via the `RouterView` slot, keyed on
   `route.fullPath` so navigating away resets the boundary.
3. Add a "notes" field to recipes rendered as Markdown. Render it with a Markdown library
   plus DOMPurify, and write a test proving that `<img onerror>` and `javascript:` links
   are stripped.
4. Make the login redirect safe: accept only paths that start with a single `/`, and write
   tests for `//evil.example`, `https://evil.example` and `/\evil.example`.
