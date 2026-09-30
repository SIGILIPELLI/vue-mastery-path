---
description: "Build accessible Vue apps: semantic elements first, focus management and announcements on route change, accessible custom widgets, useId, and automated axe checks."
---

# 08 · Accessibility in Vue Apps

Accessibility isn't a Vue feature — it's HTML, ARIA and keyboard behaviour. But single-page
apps introduce specific problems that server-rendered sites don't have (the page never
"loads", so screen readers aren't told it changed), and component frameworks make it easy
to build `<div>`-based widgets that only work with a mouse. This lesson covers the
Vue-specific parts: route changes, focus, reusable accessible components, and automated
checks — including what those checks miss.

## Semantic HTML first

Most accessibility comes free with the right element:

| Instead of | Use | You get |
|------------|-----|---------|
| `<div @click>` | `<button type="button">` | focus, Enter/Space activation, "button" role |
| `<span @click="router.push(...)">` | `<RouterLink>` (an `<a href>`) | open in new tab, "link" role, visited state |
| placeholder-only inputs | `<label for>` + `<input id>` | a name that doesn't vanish when typing |
| a custom modal `<div>` | `<dialog>` with `showModal()` | focus trap, Escape, inert background |
| `<div class="h1">` | real `<h1>`–`<h6>` in order | navigable document outline |

Components are where this pays off: fix `BaseButton` once and every button in the app is
right.

## Automated checks, and their limits

**axe-core** is the engine behind most accessibility checkers. It runs in jsdom, so you can
use it in component tests. We ran it (axe-core 4.13.0, colour-contrast disabled because
jsdom has no layout) against this deliberately bad component:

```vue title="src/a11y/BadCard.vue"
<template>
  <div class="card">
    <img src="/avatar.png" />
    <div class="button" @click="$emit('follow')">Follow</div>
    <input type="text" placeholder="Leave a note" />
    <a href="#"></a>
  </div>
</template>
```

```ts
import axe from 'axe-core'

const wrapper = mount(BadCard, { attachTo: document.body })
const results = await axe.run(wrapper.element as HTMLElement, {
  rules: { 'color-contrast': { enabled: false } },
})
```

```text
image-alt (critical): Images must have alternative text — 1 node(s)
link-name (serious): Links must have discernible text — 1 node(s)
```

It found two real problems. Now look at what it **didn't** report:

- the `<div class="button" @click>` — unreachable by keyboard and not announced as a
  button. To axe it's just a div; it can't see Vue's click listener;
- the input with only a `placeholder` — axe accepts a placeholder as an accessible name,
  but it disappears when the user starts typing and often has poor contrast.

Automated tools catch a meaningful minority of issues. They're worth running in CI, but
they don't replace keyboard testing (unplug the mouse: can you do everything with Tab,
Shift+Tab, Enter, Space, Escape and arrow keys?) and a quick screen-reader pass (VoiceOver
on macOS/iOS, NVDA on Windows, TalkBack on Android).

## Route changes in a single-page app

When a traditional site navigates, the browser loads a new document: focus resets to the
top, and screen readers announce the new page title. In an SPA, the router swaps
components and **nothing is announced**. A screen reader user clicks "Pricing", hears
nothing, and focus stays on a link that may no longer exist.

Fix both halves — announce and move focus:

```vue title="src/a11y/RouteAnnouncer.vue"
<script setup lang="ts">
import { ref, nextTick } from 'vue'
import { useRouter } from 'vue-router'

const message = ref('')
const router = useRouter()

router.afterEach(async (to, from, failure) => {
  if (failure || to.path === from.path) return      // query-only changes: stay put
  await nextTick()                                   // wait for the new view to render
  const heading = document.querySelector<HTMLElement>('main h1')
  message.value = `${heading?.textContent ?? document.title}, page loaded`
  if (heading) {
    heading.tabIndex = -1                            // focusable by script, not by Tab
    heading.focus()
  }
})
</script>

<template>
  <p class="visually-hidden" aria-live="polite" aria-atomic="true">{{ message }}</p>
</template>

<style>
.visually-hidden {
  position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px;
  overflow: hidden; clip: rect(0, 0, 0, 0); white-space: nowrap; border: 0;
}
</style>
```

Render it once, outside the `<RouterView>`:

```vue title="src/App.vue (template)"
<template>
  <a href="#main" class="skip-link">Skip to content</a>
  <RouteAnnouncer />
  <AppHeader />
  <main id="main"><RouterView /></main>
</template>
```

We mounted it with a two-route router and navigated to `/pricing`:

```text
announced: "Pricing, page loaded" | focused: <h1 tabindex="-1">Pricing</h1>
```

Decisions in this component:

- **Focus the main heading** rather than the top of the page, so keyboard users land at the
  new content (and screen readers read the heading). `tabindex="-1"` makes it focusable by
  script without adding it to the Tab order.
- **Skip query-only changes** — updating a filter shouldn't yank focus away from the filter
  input (the Recipe Book from Level 2 would have been unusable otherwise).
- **`aria-live="polite"`** waits until the screen reader finishes its current sentence.
- The "skip link" lets keyboard users jump past the header on every page.

There's no single standard for SPA route announcements; this pattern (announce + focus
heading) is a widely used, conservative choice. Test it with a real screen reader in your
app.

## Accessible custom components

When no native element fits, follow the corresponding **ARIA Authoring Practices** pattern
(published by the W3C) for roles, states and keyboard behaviour. You've built two already:

- **Tabs** (Level 2 · 02): `role="tablist"`/`tab`/`tabpanel`, `aria-selected`,
  `aria-controls`, roving `tabindex`, arrow-key navigation.
- **Toggle buttons** (Level 1 · 10's day cells): a real `<button>` with `aria-pressed`.

The rules that make them work in Vue:

1. **Bind ARIA state to the same reactive state as the visuals.** `:aria-expanded="open"`
   next to `v-if="open"` can't drift apart.
2. **Generate ids with `useId()`** (Vue 3.5+) for `aria-controls`, `aria-labelledby` and
   `aria-describedby`. It gives ids unique within the app and stable between server and
   client rendering; `Math.random()` ids cause hydration mismatches.
3. **Manage focus with template refs** after DOM updates (`await nextTick()` or a
   `flush: 'post'` watcher) — focusing an element that hasn't rendered yet silently fails.
4. **Return focus** when a transient UI closes: a menu or dialog should give focus back to
   the button that opened it.

A disclosure (show/hide) component showing all four:

```vue title="src/components/DisclosurePanel.vue"
<script setup lang="ts">
import { ref, useId, useTemplateRef, nextTick } from 'vue'

defineProps<{ summary: string }>()
const open = ref(false)
const id = useId()
const panel = useTemplateRef<HTMLElement>('panel')
const trigger = useTemplateRef<HTMLButtonElement>('trigger')

async function toggle() {
  open.value = !open.value
  if (open.value) {
    await nextTick()
    panel.value?.focus()
  }
}
function close() {
  open.value = false
  trigger.value?.focus()   // give focus back
}
</script>

<template>
  <button ref="trigger" type="button" :aria-expanded="open" :aria-controls="`${id}-panel`" @click="toggle">
    {{ summary }}
  </button>
  <div v-if="open" :id="`${id}-panel`" ref="panel" tabindex="-1" @keydown.esc="close">
    <slot />
  </div>
</template>
```

(For simple cases, the native `<details>`/`<summary>` element does this with no JavaScript.)

## Motion, colour and forms

- **Respect `prefers-reduced-motion`** in transitions (Level 1 · 09 showed the media query).
- **Don't encode meaning in colour alone** — the error states in Level 2 · 08 use text and
  `aria-invalid`, not just red borders.
- **Form errors** must be connected (`aria-describedby`) and announced; see Level 2 · 08.
- **Loading states**: `aria-busy="true"` on the region being updated, and a polite live
  region for "Saved" style confirmations (the toast host in Lesson 03).

## How It Actually Works

Screen readers don't read your Vue components; they read the browser's **accessibility
tree**, which the browser derives from the DOM: each element's role (from the tag or a
`role` attribute), its accessible name (from content, `<label>`, `alt`, `aria-label`,
`aria-labelledby`), its states (`aria-expanded`, `aria-pressed`, `disabled`) and its
relationships. Vue's job ends at producing the right DOM — which is why a `<div @click>`
is invisible to assistive technology even though the listener works.

Live regions work by the browser watching a subtree marked `aria-live` for changes and
queueing its new text for announcement. Two consequences shape the announcer: the region
must **already exist** in the DOM before its text changes (so it's rendered once in `App`,
not created per page), and the text must actually change — announcing the same page twice
needs the message cleared first.

`useId()` generates ids from an app-level prefix (`app.config.idPrefix`, `v` by default —
the disclosure above got `v-0` in our test) plus counters that follow the order in which
components are created, with extra bookkeeping around async components so their ids don't
shift. Because server and client create components in the same order, the ids match during
hydration — the property that `Math.random()` lacks.

## Common mistakes

- **Clickable `div`s and `span`s** instead of buttons and links.
- **No focus management on route change** — keyboard and screen-reader users get lost.
- **Removing focus outlines** (`outline: none`) without a visible replacement. Use
  `:focus-visible` styles.
- **`aria-label` everywhere** — it overrides visible text and drifts from it. Prefer
  visible labels; use ARIA to fill gaps, not to replace HTML.
- **Live regions created on the fly** — the first message is often not announced.
- **Treating a clean axe report as "accessible"** — it's a floor, not a ceiling.

## Exercise

1. Add `RouteAnnouncer` and a skip link to your Recipe Book. Navigate using only the
   keyboard and confirm focus lands on each page's `<h1>`.
2. Add an axe check to your component test setup: a helper `expectNoAxeViolations(wrapper)`
   that fails with the list of violations. Run it against three of your components and fix
   what it finds.
3. Fix `BadCard.vue` completely — including the two problems axe didn't report. Then turn
   on VoiceOver or NVDA and tab through it.
4. Rebuild the dropdown from Lesson 03 following the ARIA menu button pattern: arrow keys
   move between items, Escape closes and returns focus to the button, and Tab closes the
   menu.
