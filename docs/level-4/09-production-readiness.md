---
description: "Make a Vue app observable in production: a tested error reporter with route and release context, sampling and dedupe, source maps, web vitals, feature flags and a launch checklist."
---

# 09 · Observability & Production Readiness

In development you see every error in the console. In production, errors happen in
thousands of browsers you'll never look at, on devices and networks you don't have, in
versions of your code a user loaded three days ago. **Observability** is what lets you
answer "is it broken, for whom, since when, and why?" without asking users to send
screenshots.

This lesson builds a small error-reporting module, tests it with Vitest 4.1.11 and jsdom
on Vue 3.5.43, and then covers the rest of a launch checklist. Hosted services (Sentry,
Datadog, Bugsnag, OpenTelemetry-based backends and others) do much of this for you; we
build the core by hand so you know what to look for in them and how to judge their
configuration. We didn't send data to any hosted service here.

## What a useful error report contains

A stack trace alone rarely explains an error. A report you can act on answers:

| Question | Field |
|----------|-------|
| What broke? | message, stack |
| Where in the app? | route, component, Vue's `info` (render, watcher, event handler…) |
| Which code? | **release** (version or commit SHA) — matches the source map to use |
| How often, for how many users? | a session identifier, timestamp — counted by the backend |
| What led up to it? | optional breadcrumbs: recent navigations, clicks, failed requests |

And it must **not** contain personal data you don't need: no form contents, tokens or
full user objects. Scrub before sending.

## Four ways errors escape

In a Vue app, errors reach you through four doors, and a reporter must watch all of them:

1. **`app.config.errorHandler`** — errors thrown in component render functions, setup,
   lifecycle hooks, watchers and event handlers called by Vue.
2. **`window` `error` event** — uncaught errors outside Vue: third-party scripts, code in
   `setTimeout`, top-level module errors.
3. **`unhandledrejection` event** — rejected promises nobody awaited or caught, the most
   common kind in async-heavy apps.
4. **`router.onError`** — failures during navigation, including lazy-loaded route chunks
   that fail to download (Lesson 08).

## The reporter

```ts title="src/observability/monitoring.ts"
import type { App } from 'vue'
import type { Router } from 'vue-router'

export interface ErrorReport {
  message: string
  stack?: string
  source: 'vue' | 'window' | 'promise' | 'router'
  info?: string
  component?: string
  route?: string
  release: string
  sessionId: string
  time: string
}

export interface MonitoringOptions {
  release: string
  send: (report: ErrorReport) => void
  sampleRate?: number   // 0..1, applied per session
  maxPerSession?: number
  random?: () => number
}

export function installMonitoring(app: App, router: Router, opts: MonitoringOptions) {
  const random = opts.random ?? Math.random
  const sessionId = random().toString(36).slice(2, 10)
  const sampled = random() < (opts.sampleRate ?? 1)
  const seen = new Set<string>()
  let sent = 0

  function report(err: unknown, source: ErrorReport['source'], extra: Partial<ErrorReport> = {}) {
    const e = err instanceof Error ? err : new Error(String(err))
    const fingerprint = `${source}:${e.message}:${extra.component ?? ''}`
    if (!sampled || seen.has(fingerprint) || sent >= (opts.maxPerSession ?? 20)) return
    seen.add(fingerprint)
    sent++
    opts.send({
      message: e.message,
      stack: e.stack,
      source,
      route: router.currentRoute.value.fullPath,
      release: opts.release,
      sessionId,
      time: new Date().toISOString(),
      ...extra,
    })
  }

  app.config.errorHandler = (err, instance, info) => {
    report(err, 'vue', { info, component: instance?.$options.name ?? instance?.$options.__name })
    if (import.meta.env.DEV) console.error(err)
  }
  window.addEventListener('error', (ev) => report(ev.error ?? ev.message, 'window'))
  window.addEventListener('unhandledrejection', (ev) => report(ev.reason, 'promise'))
  router.onError((err) => report(err, 'router'))

  return { report }
}
```

Design choices worth noticing:

- **`send` is injected.** The module doesn't know about HTTP, so it's trivial to test and
  to swap backends.
- **Dedupe by fingerprint.** A render error inside a list can fire once per item per
  update — hundreds of identical reports that tell you nothing new and can overload your
  endpoint.
- **A per-session cap** stops a broken page in a loop from flooding you.
- **Sampling per session, not per event**, so a sampled session gives you its complete
  story rather than random fragments.
- **`random` is injectable** so sampling can be tested deterministically.

Wire it up in `main.ts`, with the release injected at build time (Lesson 03 showed
`define`):

```ts title="src/main.ts"
const app = createApp(App).use(router)
installMonitoring(app, router, {
  release: `recipe-book@${__APP_VERSION__}`,
  send: beaconSender('/api/client-errors'),
  sampleRate: 1,
})
app.mount('#app')
```

Sending uses `navigator.sendBeacon` where possible, because it survives the page being
closed — which is exactly when many errors are reported:

```ts title="src/observability/monitoring.ts (continued)"
export function beaconSender(url: string) {
  return (report: ErrorReport) => {
    const body = JSON.stringify(report)
    if (!navigator.sendBeacon?.(url, new Blob([body], { type: 'application/json' }))) {
      fetch(url, { method: 'POST', body, keepalive: true,
        headers: { 'content-type': 'application/json' } }).catch(() => {})
    }
  }
}
```

Note the empty `catch`: the error reporter must never throw, or it becomes a source of the
errors it reports.

## Testing it

```ts title="src/observability/monitoring.spec.ts"
import { describe, it, expect } from 'vitest'
import { createApp, defineComponent, h, nextTick } from 'vue'
import { createRouter, createMemoryHistory, RouterView } from 'vue-router'
import { installMonitoring, type ErrorReport } from './monitoring'

const Broken = defineComponent({
  name: 'PriceTag',
  props: { price: Object },
  setup(props) {
    return () => h('span', (props.price as any).amount.toFixed(2))
  },
})

async function setup(opts: Partial<Parameters<typeof installMonitoring>[2]> = {}) {
  const reports: ErrorReport[] = []
  const router = createRouter({ history: createMemoryHistory(), routes: [
    { path: '/', component: { render: () => h('p', 'home') } },
    { path: '/product/:id', component: { render: () => h(Broken, { price: undefined }) } },
  ] })
  const app = createApp({ render: () => h('div', [h(RouterView)]) })
  app.use(router)
  const api = installMonitoring(app, router,
    { release: 'shop@1.4.0', send: (r) => reports.push(r), ...opts })
  await router.push('/'); app.mount(document.createElement('div'))
  return { reports, router, api }
}

describe('monitoring', () => {
  it('reports a render error with component, route and release', async () => {
    const { reports, router } = await setup()
    await router.push('/product/7'); await nextTick()
    const { stack, time, sessionId, ...rest } = reports[0]
    console.log(JSON.stringify(rest, null, 2))
    expect(rest).toMatchObject({ source: 'vue', component: 'PriceTag',
      route: '/product/7', release: 'shop@1.4.0' })
  })

  it('reports unhandled promise rejections', async () => {
    const { reports } = await setup()
    const ev = new Event('unhandledrejection') as any
    ev.reason = new Error('checkout API timed out')
    window.dispatchEvent(ev)
    expect(reports.map((r) => [r.source, r.message]))
      .toContainEqual(['promise', 'checkout API timed out'])
  })

  it('deduplicates repeats and caps per session', async () => {
    const { reports, api } = await setup({ maxPerSession: 3 })
    for (let i = 0; i < 50; i++) api.report(new Error('same'), 'window')
    for (let i = 0; i < 10; i++) api.report(new Error(`different ${i}`), 'window')
    console.log('sent:', reports.map((r) => r.message))
    expect(reports).toHaveLength(3)
  })

  it('drops everything for an unsampled session', async () => {
    const { reports, api } = await setup({ sampleRate: 0.1, random: () => 0.5 })
    api.report(new Error('x'), 'window')
    expect(reports).toHaveLength(0)
  })
})
```

All four tests passed. The logged report for the render error:

```json
{
  "message": "Cannot read properties of undefined (reading 'amount')",
  "source": "vue",
  "route": "/product/7",
  "release": "shop@1.4.0",
  "info": "render function",
  "component": "PriceTag"
}
```

and the dedupe test printed `sent: [ 'same', 'different 0', 'different 1' ]` — 60 calls,
three reports.

The first test did **not** pass at first: we had written `h('router-view')`, which
renders a literal `<router-view>` element rather than the component, so the product page
never mounted and `reports[0]` was undefined. Passing the imported `RouterView` fixed it.
It's a reminder that a test for an error reporter must prove the error actually happened.

`info` read `"render function"` in this development-mode test. In production builds Vue
passes a short code instead of the readable text to save bytes; Vue's documentation has an
error-reference page that maps codes back to descriptions, so decode them in your backend.

## Making stack traces readable

Production stacks point into minified bundles: `index-KmR4qrb9.js:1:48213`. To map them
back, build source maps and **upload them to your error service instead of serving them**:

```ts title="vite.config.ts"
export default defineConfig({
  build: { sourcemap: 'hidden' },   // .map files emitted, no sourceMappingURL comment
})
```

`hidden` generates `.map` files without the comment that tells browsers where to find
them. Upload the maps from CI tagged with the same **release** string the app reports,
then delete them from the deploy artifact (or block `*.map` at the server) if you don't want
your original source public.

## Performance in the field

Lab tools (Lighthouse, your own profiling from Level 3 · 06) measure one device on one
network. Field data — **Real User Monitoring** — measures actual users. The Core Web
Vitals are the common baseline:

- **LCP** (Largest Contentful Paint): how quickly the main content appears.
- **INP** (Interaction to Next Paint): how quickly the page responds to input.
- **CLS** (Cumulative Layout Shift): how much the layout jumps.

The browser exposes the raw entries through `PerformanceObserver`; Google's small
`web-vitals` library turns them into the official metrics, handling many edge cases. Report
them with the same `sendBeacon` pattern, tagged with route and release, and look at
**percentiles** (the 75th is the usual target), never averages. We haven't included field
numbers because we don't have a production deployment to measure; your own data is the
only data that matters for your app.

For single-page apps, remember that LCP and CLS are measured for the initial page load;
client-side navigations need custom timings (for example, from `router.beforeEach` to the
next paint after `afterEach`).

## Beyond errors

- **Logs from the server side** (API, SSR) should carry a **request ID** that the client can
  include in error reports, so one incident can be followed across both.
- **Feature flags** let you ship code dark and turn it on gradually, and — more
  importantly — turn it *off* in seconds without a deploy. Keep flag checks in one
  composable (`useFlag('new-checkout')`) so removing a flag later is a small change.
- **Uptime and synthetic checks**: an external probe that loads the home page and one
  critical route every few minutes catches "the whole site is down" faster than error
  reports, because a site that doesn't load reports nothing.
- **Alerts on rates, not counts**: "errors per 1,000 sessions doubled since release X"
  is actionable; "500 errors today" isn't.

## A launch checklist

Before a Vue app goes to real users:

- [ ] Error reporting on all four doors, with release, route and scrubbing (this lesson)
- [ ] Hidden source maps uploaded per release
- [ ] Web vitals reported from the field
- [ ] History fallback and cache headers verified with `curl` (Lesson 08)
- [ ] Stale-chunk recovery (`vite:preloadError`) in place
- [ ] Runtime config, no secrets in the bundle (Lessons 03 and 08)
- [ ] CSP and `v-html` review (Level 3 · 09)
- [ ] Accessibility checks, including keyboard-only and a screen reader pass (Level 3 · 08)
- [ ] 404 and error pages that help the user continue
- [ ] Rollback tested: you have actually redeployed a previous artifact once
- [ ] Someone is on call to read the alerts during the launch window

## How It Actually Works

- **How Vue routes errors:** Vue wraps every call into user code — setup, render, hooks,
  watcher callbacks, event handlers bound in templates — in `callWithErrorHandling` (or its
  async variant, which also attaches a `.catch` to returned promises). A thrown error walks
  up the component parent chain calling `onErrorCaptured` hooks; if none stops it, it
  reaches `app.config.errorHandler` with the component instance and an `info` describing
  where it happened.
- **Why `unhandledrejection` is separate:** a promise rejected inside your own `fetch`
  chain that nobody awaits never throws synchronously into a Vue-wrapped call, so Vue never
  sees it. The browser detects "rejected with no handler attached by the end of the
  microtask checkpoint" and fires the window event.
- **Why `sendBeacon` survives unload:** the browser queues the request and completes it
  independently of the page's lifetime, instead of cancelling in-flight requests when the
  document is torn down. `fetch` with `keepalive: true` does the same, with a size limit.
- **How source maps are applied:** a map is JSON that encodes, for positions in the
  generated file, the original file, line and column. The error service parses each stack
  frame's `file:line:column`, finds the map uploaded for that release and file, and looks
  up the original position — which is why the release string must match exactly.

## Common mistakes

- **Only setting `errorHandler`** and missing promise rejections and router errors.
- **No release tag** — you can't tell whether an error is fixed or just from old tabs.
- **Sending personal data** (form values, tokens, full user objects) in reports.
- **No dedupe or cap** — one bug in a loop floods your quota.
- **A reporter that can throw** — wrap sends in `try`/`catch` and swallow failures.
- **Public source maps by accident**, or none at all.
- **Averages instead of percentiles** for performance.
- **Alerting on everything** until people stop reading alerts.

## Exercise

1. Add the monitoring module to your Recipe Book and write the four tests. Then add a
   fifth: a lazy route whose import rejects should produce a `router` report.
2. Add a `scrub(report)` step that removes query strings from `route` and any string that
   looks like an email address from `message`. Test it.
3. Write a tiny endpoint (Nuxt server route, or a Node script) that receives reports and
   prints them. Build the app with `sourcemap: 'hidden'`, trigger an error, and use the
   `source-map` npm package to map the top stack frame back to your `.vue` file.
4. Record LCP with `PerformanceObserver` (or `web-vitals`) and send it with
   `sendBeacon`. Check in the Network panel that it's sent when you close the tab.
5. Write your own launch checklist for the Level 4 capstone, starting from the one above.
   Which items can CI verify automatically?
