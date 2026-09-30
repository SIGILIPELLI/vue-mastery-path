---
description: "Level 4 capstone: a production-shaped Nuxt meetup app with SSR, validated server API, RSVP form, error reporting, runtime release config, unit tests and a smoke test."
---

# 10 · Capstone — A Production Vue App

The final project is small in features and large in production concerns: a **meetup site**
where people browse events and RSVP. It's server-rendered with Nuxt, has its own
validated API, handles capacity and duplicate RSVPs, reports client errors with the
release attached, returns real status codes, and ships with unit tests plus a post-deploy
smoke test.

Everything below was built with Nuxt 4.5.2 (Vue 3.5.43) and run as a production build
(`node .output/server/index.mjs`). The API was exercised with `curl`, the smoke test ran
against the server, and the RSVP form was used in a real browser. Two of the checks failed
the first time; both failures are shown, because finding them is what the smoke test is
for.

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

## 1. Scaffold

```bash
npm create nuxt@latest meetup
cd meetup
```

Final layout:

```text
meetup/
├─ nuxt.config.ts
├─ app/
│  ├─ app.vue
│  ├─ error.vue
│  ├─ components/RsvpForm.vue
│  ├─ pages/index.vue
│  ├─ pages/about.vue
│  ├─ pages/events/[slug].vue
│  ├─ plugins/monitoring.client.ts
│  └─ utils/format.ts
├─ server/
│  ├─ api/events/index.get.ts
│  ├─ api/events/[slug].get.ts
│  ├─ api/events/[slug]/rsvp.post.ts
│  ├─ api/client-errors.post.ts
│  └─ utils/{events,rsvp}.ts
├─ tests/rsvp.test.ts
└─ scripts/smoke.mjs
```

## 2. Configuration

```ts title="nuxt.config.ts"
export default defineNuxtConfig({
  compatibilityDate: '2025-07-15',
  devtools: { enabled: false },
  app: { head: { htmlAttrs: { lang: 'en' } } },
  runtimeConfig: {
    public: { release: 'dev' },          // overridden by NUXT_PUBLIC_RELEASE at runtime
  },
  routeRules: {
    '/about': { prerender: true },
    '/api/**': { headers: { 'cache-control': 'no-store' } },
  },
})
```

`runtimeConfig.public.release` is a default. Nuxt overrides runtime config from
environment variables named after the key — `NUXT_PUBLIC_RELEASE` here — when the server
starts, so one build can be deployed with different values. Keep that sentence in mind;
section 8 shows where it stops being true.

## 3. Validation as a pure function

The most important server rule — what counts as a valid RSVP — lives in a module with no
framework imports, so it can be tested with nothing but Node:

```ts title="server/utils/rsvp.ts"
// Pure validation: no Nuxt/Nitro imports, so it can be unit-tested with plain Node.
export interface RsvpInput { name: string; email: string; guests: number }
export type RsvpResult =
  | { ok: true; value: RsvpInput }
  | { ok: false; errors: Partial<Record<keyof RsvpInput, string>> }

const EMAIL = /^[^\s@]+@[^\s@]+\.[^\s@]+$/

export function validateRsvp(body: unknown): RsvpResult {
  const b = (body ?? {}) as Record<string, unknown>
  const errors: Partial<Record<keyof RsvpInput, string>> = {}
  const name = typeof b.name === 'string' ? b.name.trim() : ''
  const email = typeof b.email === 'string' ? b.email.trim().toLowerCase() : ''
  const guests = b.guests === undefined ? 0 : Number(b.guests)

  if (name.length < 2) errors.name = 'Please enter your name.'
  else if (name.length > 80) errors.name = 'Name is too long.'
  if (!EMAIL.test(email)) errors.email = 'Please enter a valid email address.'
  if (!Number.isInteger(guests) || guests < 0 || guests > 3) errors.guests = 'Guests must be 0–3.'

  return Object.keys(errors).length ? { ok: false, errors } : { ok: true, value: { name, email, guests } }
}
```

It normalises (trim, lowercase email, numeric guests) and returns **all** field errors at
once, so the form can show them together. It never throws on odd input: the body comes
from the network and can be anything.

```ts title="tests/rsvp.test.ts"
import { test } from 'node:test'
import assert from 'node:assert/strict'
import { validateRsvp } from '../server/utils/rsvp.ts'

test('accepts and normalises a valid RSVP', () => {
  assert.deepEqual(validateRsvp({ name: '  Ada ', email: 'ADA@Example.org', guests: '1' }),
    { ok: true, value: { name: 'Ada', email: 'ada@example.org', guests: 1 } })
})

test('defaults guests to 0', () => {
  const r = validateRsvp({ name: 'Grace', email: 'g@h.io' })
  assert.equal(r.ok && r.value.guests, 0)
})

test('reports every invalid field at once', () => {
  assert.deepEqual(validateRsvp({ name: 'A', email: 'nope', guests: 7 }), {
    ok: false,
    errors: { name: 'Please enter your name.', email: 'Please enter a valid email address.', guests: 'Guests must be 0–3.' },
  })
})

test('rejects non-object bodies without throwing', () => {
  for (const body of [null, undefined, 'x', 42, []]) assert.equal(validateRsvp(body).ok, false)
})

test('rejects fractional guests', () => {
  assert.equal(validateRsvp({ name: 'Al', email: 'a@b.co', guests: 1.5 }).ok, false)
})
```

Node 26 runs TypeScript files directly by stripping types, so no test framework or build
step is needed:

```text
$ node --test 'tests/*.test.ts'
✔ accepts and normalises a valid RSVP
✔ defaults guests to 0
✔ reports every invalid field at once
✔ rejects non-object bodies without throwing
✔ rejects fractional guests
ℹ tests 5
ℹ pass 5
ℹ fail 0
```

(A small trap on the way: `node --test tests/` treats the directory as a single file and
fails; pass a glob instead.)

## 4. Data and business rules

```ts title="server/utils/events.ts"
import type { RsvpInput } from './rsvp'

export interface MeetupEvent {
  slug: string
  title: string
  startsAt: string   // ISO, UTC
  capacity: number
  summary: string
}

const seed: MeetupEvent[] = [
  { slug: 'vue-internals', title: 'Vue internals night', startsAt: '2026-10-14T17:30:00Z', capacity: 3, summary: 'Reactivity, the compiler and the renderer, live-coded.' },
  { slug: 'testing-clinic', title: 'Testing clinic', startsAt: '2026-10-28T17:30:00Z', capacity: 40, summary: 'Bring a component that is hard to test.' },
]

// Nitro storage: in-memory by default; mount a real driver (Redis, filesystem, KV) in production.
const storage = () => useStorage('data')

export async function listEvents() {
  return Promise.all(seed.map(async (e) => ({ ...e, going: await countGoing(e.slug) })))
}

export async function getEvent(slug: string) {
  const e = seed.find((x) => x.slug === slug)
  return e ? { ...e, going: await countGoing(slug) } : null
}

async function rsvps(slug: string) {
  return (await storage().getItem<RsvpInput[]>(`rsvps:${slug}`)) ?? []
}

async function countGoing(slug: string) {
  return (await rsvps(slug)).reduce((n, r) => n + 1 + r.guests, 0)
}

export async function addRsvp(slug: string, input: RsvpInput) {
  const list = await rsvps(slug)
  if (list.some((r) => r.email === input.email)) return { error: 'duplicate' as const }
  const event = seed.find((x) => x.slug === slug)!
  const going = list.reduce((n, r) => n + 1 + r.guests, 0)
  if (going + 1 + input.guests > event.capacity) return { error: 'full' as const, remaining: event.capacity - going }
  list.push(input)
  await storage().setItem(`rsvps:${slug}`, list)
  return { going: going + 1 + input.guests, remaining: event.capacity - going - 1 - input.guests }
}
```

`useStorage` is Nitro's key-value storage layer. With no configuration it's **in memory**:
data disappears on restart and isn't shared between server instances. That's fine for the
project; production needs a real driver or a database (see the exercise). Capacity counts
the person plus their guests.

## 5. API routes

```ts title="server/api/events/index.get.ts"
export default defineEventHandler(() => listEvents())
```

```ts title="server/api/events/[slug].get.ts"
export default defineEventHandler(async (event) => {
  const found = await getEvent(getRouterParam(event, 'slug')!)
  if (!found) throw createError({ statusCode: 404, statusMessage: 'Event not found' })
  return found
})
```

```ts title="server/api/events/[slug]/rsvp.post.ts"
export default defineEventHandler(async (event) => {
  const slug = getRouterParam(event, 'slug')!
  if (!(await getEvent(slug))) throw createError({ statusCode: 404, statusMessage: 'Event not found' })

  const result = validateRsvp(await readBody(event).catch(() => null))
  if (!result.ok) {
    throw createError({ statusCode: 422, statusMessage: 'Invalid RSVP', data: { errors: result.errors } })
  }
  const outcome = await addRsvp(slug, result.value)
  if ('error' in outcome) {
    const message = outcome.error === 'full'
      ? `Not enough seats left (${outcome.remaining} remaining).`
      : 'This email has already RSVPed.'
    throw createError({ statusCode: 409, statusMessage: 'Conflict', data: { message } })
  }
  setResponseStatus(event, 201)
  return outcome
})
```

Each outcome has its own status: **201** created, **404** unknown event, **422** invalid
input (with per-field errors in `data`), **409** conflicts with existing state. The client
can branch on the status instead of parsing messages.

Exercising it against the production build, on the event with capacity 3:

```text
POST {"name":"Ada","email":"ada@example.org","guests":1}      → 201 {"going":2,"remaining":1}
POST {"name":"Ada again","email":"ADA@example.org"}           → 409 "This email has already RSVPed."
POST {"name":"Grace","email":"grace@example.org","guests":1}  → 409 "Not enough seats left (1 remaining)."
POST {"name":"Linus","email":"linus@example.org"}             → 201 {"going":3,"remaining":0}
POST {"name":"Evan","email":"evan@example.org"}               → 409 "Not enough seats left (0 remaining)."
POST {"name":"A","email":"bad","guests":9}                    → 422
```

The duplicate check caught `ADA@example.org` because validation lowercases emails. The
422 response body was:

```json
{
  "error": true,
  "url": "http://localhost:3920/api/events/vue-internals/rsvp",
  "statusCode": 422,
  "statusMessage": "Invalid RSVP",
  "message": "Invalid RSVP",
  "data": {
    "errors": {
      "name": "Please enter your name.",
      "email": "Please enter a valid email address.",
      "guests": "Guests must be 0–3."
    }
  }
}
```

Client errors get their own endpoint, which validates and truncates what it logs:

```ts title="server/api/client-errors.post.ts"
export default defineEventHandler(async (event) => {
  const report = await readBody(event).catch(() => null)
  if (!report || typeof report.message !== 'string') {
    throw createError({ statusCode: 400, statusMessage: 'Bad report' })
  }
  // In production, forward to your log pipeline or error service instead.
  console.warn('[client-error]', JSON.stringify({
    message: report.message.slice(0, 500),
    source: report.source,
    route: report.route,
    release: report.release,
    ua: getRequestHeader(event, 'user-agent')?.slice(0, 120),
  }))
  setResponseStatus(event, 204)
  return null
})
```

## 6. Pages

```vue title="app/app.vue"
<script setup lang="ts">
useHead({ titleTemplate: (t) => (t ? `${t} · Vue Meetup` : 'Vue Meetup') })
</script>

<template>
  <NuxtRouteAnnouncer />
  <header>
    <NuxtLink to="/">Vue Meetup</NuxtLink>
    <nav aria-label="Main"><NuxtLink to="/about">About</NuxtLink></nav>
  </header>
  <NuxtPage />
</template>
```

```ts title="app/utils/format.ts"
// Always pass an explicit time zone: the server's zone and the visitor's zone differ,
// and a date rendered differently on each side is a hydration mismatch.
const fmt = new Intl.DateTimeFormat('en-GB', {
  dateStyle: 'medium', timeStyle: 'short', timeZone: 'Europe/Berlin',
})
export const formatEventTime = (iso: string) => `${fmt.format(new Date(iso))} (Berlin time)`
```

The explicit `timeZone` is not a nicety. Without it, the server formats in *its* zone
(often UTC in containers) and the browser in the visitor's, so the text differs and
hydration reports a mismatch (Lesson 01). With it, both sides agree. The rendered output
also shows `Intl` handling daylight saving time for us: the 14 October event (17:30 UTC)
showed **19:30**, the 28 October one (also 17:30 UTC) showed **18:30** — Berlin had
switched from summer to winter time in between.

```vue title="app/pages/index.vue"
<script setup lang="ts">
const { data: events } = await useFetch('/api/events')
useSeoMeta({ title: 'Upcoming events', description: 'Free evening meetups about Vue.' })
</script>

<template>
  <main>
    <h1>Upcoming events</h1>
    <ul>
      <li v-for="e in events" :key="e.slug">
        <NuxtLink :to="`/events/${e.slug}`">{{ e.title }}</NuxtLink>
        — {{ formatEventTime(e.startsAt) }} · {{ e.capacity - e.going }} seats left
      </li>
    </ul>
  </main>
</template>
```

```vue title="app/pages/events/[slug].vue"
<script setup lang="ts">
const route = useRoute()
const { data: event, error, refresh } = await useFetch(`/api/events/${route.params.slug}`)
if (error.value) {
  throw createError({ statusCode: error.value.statusCode ?? 500, statusMessage: error.value.statusMessage, fatal: true })
}
useSeoMeta({
  title: () => event.value?.title,
  description: () => event.value?.summary,
})
</script>

<template>
  <main v-if="event">
    <h1>{{ event.title }}</h1>
    <p>{{ event.summary }}</p>
    <p>{{ formatEventTime(event.startsAt) }}</p>
    <p data-test="seats">{{ event.capacity - event.going }} of {{ event.capacity }} seats left</p>
    <RsvpForm :slug="event.slug" @done="refresh()" />
  </main>
</template>
```

Throwing `createError` with `fatal: true` during setup makes the **server response itself**
a 404, rendered by `error.vue`:

```vue title="app/error.vue"
<script setup lang="ts">
import type { NuxtError } from '#app'
const props = defineProps<{ error: NuxtError }>()
useSeoMeta({ title: props.error.statusCode === 404 ? 'Not found' : 'Error', robots: 'noindex' })
</script>

<template>
  <main>
    <h1>{{ error.statusCode === 404 ? 'We couldn’t find that page' : 'Something went wrong' }}</h1>
    <p>{{ error.statusMessage }}</p>
    <button @click="clearError({ redirect: '/' })">Back to events</button>
  </main>
</template>
```

`robots: 'noindex'` keeps error pages out of search results.

```vue title="app/pages/about.vue"
<script setup lang="ts">
useSeoMeta({ title: 'About', description: 'Who runs the Vue Meetup and how RSVPs work.' })
const release = useRuntimeConfig().public.release
</script>

<template>
  <main>
    <h1>About</h1>
    <p>A small community meetup. Seats are limited; RSVP on each event page.</p>
    <p><small>Build: {{ release }}</small></p>
  </main>
</template>
```

## 7. The RSVP form

```vue title="app/components/RsvpForm.vue"
<script setup lang="ts">
import { FetchError } from 'ofetch'

const props = defineProps<{ slug: string }>()
const emit = defineEmits<{ done: [] }>()

const form = reactive({ name: '', email: '', guests: 0 })
const errors = ref<Record<string, string>>({})
const status = ref('')
const pending = ref(false)

async function submit() {
  pending.value = true
  errors.value = {}
  status.value = ''
  try {
    const res = await $fetch(`/api/events/${props.slug}/rsvp`, { method: 'POST', body: form })
    status.value = `You're in! ${res.remaining} seats left.`
    Object.assign(form, { name: '', email: '', guests: 0 })
    emit('done')
  } catch (e) {
    const data = e instanceof FetchError ? e.data?.data : undefined
    if (data?.errors) errors.value = data.errors
    status.value = data?.message ?? (data?.errors ? 'Please fix the highlighted fields.' : 'Something went wrong. Please try again.')
  } finally {
    pending.value = false
  }
}
</script>

<template>
  <form novalidate @submit.prevent="submit">
    <h2>RSVP</h2>
    <label>Name <input v-model="form.name" name="name" autocomplete="name"
      :aria-invalid="!!errors.name" aria-describedby="name-err"></label>
    <p id="name-err">{{ errors.name }}</p>
    <label>Email <input v-model="form.email" name="email" type="email" autocomplete="email"
      :aria-invalid="!!errors.email" aria-describedby="email-err"></label>
    <p id="email-err">{{ errors.email }}</p>
    <label>Guests <input v-model.number="form.guests" name="guests" type="number" min="0" max="3"
      :aria-invalid="!!errors.guests" aria-describedby="guests-err"></label>
    <p id="guests-err">{{ errors.guests }}</p>
    <button :disabled="pending">{{ pending ? 'Sending…' : 'RSVP' }}</button>
    <p role="status">{{ status }}</p>
  </form>
</template>
```

Notes:

- `novalidate` turns off the browser's own bubbles so the server's messages are the
  single source of truth — the server must validate anyway, and duplicating rules in two
  places lets them drift. (For instant feedback you can *also* run `validateRsvp` on the
  client; it has no server dependencies.)
- Each input points to its error with `aria-describedby`, and `aria-invalid` marks it.
  The `role="status"` paragraph is a polite live region, so screen readers announce the
  result.
- `$fetch` (not `useFetch`) because this runs in an event handler; there's nothing to
  hydrate.
- On success the page's `refresh()` re-fetches the event so the seat count updates.

In the browser, submitting with an empty name and `not-an-email` showed "Please enter
your name.", "Please enter a valid email address." and "Please fix the highlighted
fields." Filling in valid values and submitting again showed "You're in! 39 seats left."
and the seat line changed from "40 of 40" to "39 of 40". The console's only error was the
browser logging the expected 422 response; there were no hydration warnings.

## 8. Error reporting

```ts title="app/plugins/monitoring.client.ts"
export default defineNuxtPlugin((nuxtApp) => {
  const release = useRuntimeConfig().public.release
  const router = useRouter()
  const seen = new Set<string>()

  function report(err: unknown, source: string) {
    const e = err instanceof Error ? err : new Error(String(err))
    const key = `${source}:${e.message}`
    if (seen.has(key) || seen.size >= 20) return
    seen.add(key)
    const body = JSON.stringify({ message: e.message, stack: e.stack, source,
      route: router.currentRoute.value.path, release })
    navigator.sendBeacon?.('/api/client-errors', new Blob([body], { type: 'application/json' }))
  }

  nuxtApp.hook('vue:error', (err) => report(err, 'vue'))
  nuxtApp.hook('app:error', (err) => report(err, 'app'))
  window.addEventListener('unhandledrejection', (ev) => report(ev.reason, 'promise'))
})
```

The `.client.ts` suffix makes Nuxt load the plugin only in the browser. `vue:error` and
`app:error` are Nuxt hooks that fire for component errors and fatal app errors. A report
posted to the endpoint appeared in the server log as:

```text
[client-error] {"message":"Test error","source":"vue","route":"/","release":"meetup@1.0.0+abc123","ua":"curl/8.7.1"}
```

## 9. The smoke test

Unit tests prove the rules; a **smoke test** proves the *deployed* thing works — right
routes, right statuses, right headers, rendered HTML. Run it against every environment
after each deploy:

```js title="scripts/smoke.mjs"
// Post-deploy smoke test: node scripts/smoke.mjs https://your-host
const base = process.argv[2] ?? 'http://localhost:3000'
const checks = []
const check = async (name, fn) => {
  try { await fn(); checks.push(['PASS', name]) } catch (e) { checks.push(['FAIL', `${name}: ${e.message}`]) }
}
const expect = (cond, msg) => { if (!cond) throw new Error(msg) }
const get = (p, init) => fetch(base + p, init)
const post = (p, body) => get(p, { method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify(body) })

await check('home is server-rendered with events', async () => {
  const res = await get('/'); const html = await res.text()
  expect(res.status === 200, `status ${res.status}`)
  expect(html.includes('Vue internals night'), 'event title missing from HTML')
  expect(html.includes('<title>Upcoming events · Vue Meetup</title>'), 'title missing')
  expect(/<html[^>]*\slang="en"/.test(html), 'lang missing')
})
await check('event page renders and has a description', async () => {
  const html = await (await get('/events/testing-clinic')).text()
  expect(html.includes('<meta name="description" content="Bring a component that is hard to test.">'), 'description missing')
})
await check('unknown event is a real 404', async () => {
  expect((await get('/events/nope')).status === 404, 'expected 404')
})
await check('about page is prerendered', async () => {
  const res = await get('/about'); expect(res.status === 200, `status ${res.status}`)
})
await check('API responses are not cached', async () => {
  expect((await get('/api/events')).headers.get('cache-control') === 'no-store', 'cache-control')
})
await check('invalid RSVP → 422 with field errors', async () => {
  const res = await post('/api/events/testing-clinic/rsvp', { name: 'A', email: 'x' })
  const body = await res.json()
  expect(res.status === 422, `status ${res.status}`)
  expect(body.data?.errors?.email, 'no email error')
})
await check('bad client-error report → 400', async () => {
  expect((await post('/api/client-errors', { nope: 1 })).status === 400, 'expected 400')
})

for (const [s, n] of checks) console.log(s, n)
process.exitCode = checks.some(([s]) => s === 'FAIL') ? 1 : 0
```

### What it caught

The first run printed:

```text
FAIL home is server-rendered with events: lang missing
PASS event page renders and has a description
PASS unknown event is a real 404
...
```

The page was fine — Nuxt rendered `<html  lang="en">` with **two spaces**, and the original
check looked for the exact string `<html lang="en"`. The fix was a looser regular
expression (`/<html[^>]*\slang="en"/`). Smoke tests should assert on meaning, not on
incidental formatting.

The second problem wasn't in the test output at all. The server was started with
`NUXT_PUBLIC_RELEASE=meetup@1.0.0+abc123`, the error report carried that value — but the
About page said **`Build: dev`**. The About page is **prerendered at build time**
(`routeRules`), so it captured the runtime config as it was *during the build*. Environment
variables set when starting the server can't change HTML that was already written to disk.
Building with the variable set as well fixed it:

```text
$ NUXT_PUBLIC_RELEASE=meetup@1.0.0+abc123 npx nuxt build
$ NUXT_PUBLIC_RELEASE=meetup@1.0.0+abc123 node .output/server/index.mjs
$ node scripts/smoke.mjs http://localhost:3920
PASS home is server-rendered with events
PASS event page renders and has a description
PASS unknown event is a real 404
PASS about page is prerendered
PASS API responses are not cached
PASS invalid RSVP → 422 with field errors
PASS bad client-error report → 400
```

and the About page then showed `Build: meetup@1.0.0+abc123`. The general rule: **anything
prerendered is build-time**, whatever API produced it. Either supply the value at build
time or don't prerender pages that display it. Add an assertion for it to the smoke test
(exercise 1).

## 10. Shipping it

In CI: install, run `node --test`, type-check, `nuxt build` with the release set, deploy
`.output/`, then run `scripts/smoke.mjs` against the new deployment and roll back on
failure. The container, caching and monitoring advice from Lessons 08 and 09 applies
unchanged. We ran all of this locally, not on a hosting provider.

## How It Actually Works

- **One request, end to end:** a GET for `/events/testing-clinic` reaches Nitro, which
  matches no API route and calls the Vue renderer. The page's `useFetch` calls
  `/api/events/testing-clinic` **in-process**, stores the result in the payload, renders
  HTML, and Unhead writes the title and description into `<head>`. In the browser, the
  client bundle hydrates using the payload without refetching; the form becomes
  interactive; submitting posts to the API over real HTTP.
- **Status codes cross the SSR boundary:** `createError` thrown in page setup is caught by
  Nuxt's renderer, which renders `error.vue` and uses the error's `statusCode` for the HTTP
  response — why `/events/nope` is a real 404 and not a 200 page saying "not found".
- **Runtime config resolution:** at server start, Nitro walks the config keys and applies
  matching `NUXT_*` environment variables; the public part is serialised into each
  server-rendered page for the client. Prerendered pages were rendered during the build,
  so they carry whatever values existed then.
- **Where concurrency bites:** `addRsvp` reads the list, checks capacity, then writes —
  with `await`s in between. Two requests for the last seat can both read "1 remaining"
  before either writes, and both succeed. A single Node process with in-memory storage
  makes that window small, not zero; with several instances and a shared store it's a real
  overbooking bug. Databases solve it with transactions or a conditional update
  (`UPDATE ... WHERE going + n <= capacity`).

## Common mistakes

- **Validating only on the client** — the API is public; anyone can POST to it.
- **One status code for every failure** — clients can't tell "fix your input" from "event
  full" from "server broken".
- **Formatting dates without a time zone** in server-rendered pages.
- **Assuming runtime config reaches prerendered pages.**
- **In-memory storage in production** — lost on restart, not shared across instances.
- **Check-then-write without atomicity** for limited resources like seats.
- **Brittle smoke tests** that match exact HTML formatting.

## Exercise

1. Build the app and run the unit tests and smoke test. Add a smoke check that the About
   page shows the expected release (pass it as a second argument), and confirm it fails when
   you build without `NUXT_PUBLIC_RELEASE`.
2. Write a script that sends 20 concurrent RSVPs for the last seats of an event. Does
   overbooking happen with in-memory storage? Then fix `addRsvp` so it can't happen — for
   example with a per-event promise queue in-process, or with SQLite and a conditional
   `UPDATE`.
3. Replace in-memory storage with a persistent Nitro storage driver or SQLite, and make
   the data survive a server restart.
4. Add an RSVP cancellation link: `DELETE /api/events/:slug/rsvp` with a signed token
   emailed to the user (log the link instead of sending mail). What status codes do you
   need?
5. Translate the app into a second language using Lesson 07. Which strings come from the
   server, and how will you translate them?
6. Write the launch checklist for this app from Lesson 09, and tick off each item with
   evidence (a test, a `curl` command, or a screenshot).
