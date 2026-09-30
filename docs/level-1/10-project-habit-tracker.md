---
description: "Level 1 project: build a Vue habit tracker with a weekly grid, streaks, localStorage persistence, v-model components, transitions and component tests."
---

# 10 · Project — A Habit Tracker

Time to put Level 1 together. You'll build a habit tracker: add habits, tick them off for
each of the last seven days, see your current streak, and keep everything across page
reloads. Every file on this page was compiled with `vue-tsc`, and the tests at the end were
run with Vitest — all six pass.

What you'll use from this level:

| Feature | Where |
|---------|-------|
| `ref`, `computed`, deep `watch` | `App.vue` state, summary, persistence |
| Props, emits, `defineModel` | `HabitRow`, `DayCell`, `HabitForm` |
| `v-for` with stable keys, `v-if`/`v-else` | the habit list and empty state |
| Form handling with `v-model` and `.prevent` | `HabitForm` |
| Scoped CSS, `v-bind()` in CSS | per-habit colours in `DayCell` |
| `<TransitionGroup>` | rows animate in and out |

## 1. Create the project

```bash
npm create vue@latest habit-tracker -- --ts --vitest
cd habit-tracker
npm install
```

No router or Pinia this time — one screen, and state that lives in the root component.
Delete the example files in `src/components/` and `src/components/__tests__/`, and empty
`src/assets/main.css` (or keep its reset — your choice).

### A Vitest fix for Node 25 and later

If you are on Node.js 25 or newer, add one line to `vitest.config.ts` before writing any
test that touches `localStorage`:

```ts title="vitest.config.ts (test section)"
test: {
  environment: 'jsdom',
  // Node 25+ ships its own localStorage global, which hides jsdom's in tests
  execArgv: ['--no-experimental-webstorage'],
  exclude: [...configDefaults.exclude, 'e2e/**'],
  root: fileURLToPath(new URL('./', import.meta.url)),
},
```

We found this the hard way. On Node 26 our tests first failed with:

```text
TypeError: Cannot read properties of undefined (reading 'clear')
```

Newer Node versions define a built-in `localStorage` global that is `undefined` unless
Node is started with `--localstorage-file`, and it shadows the one jsdom provides. Plain
`node -e "console.log(typeof localStorage)"` prints `undefined` with an
`ExperimentalWarning`. Turning Node's web-storage off for the test workers restores jsdom's
implementation. On Node 22 or 24 the flag is harmless.

## 2. Pure logic first

Keep date and streak logic out of components. Plain functions are easier to test, and
components stay focused on rendering.

```ts title="src/habits.ts"
export interface Habit {
  id: string
  name: string
  color: string
  /** ISO dates (YYYY-MM-DD) on which the habit was completed */
  done: string[]
}

export function toISODate(d: Date): string {
  const y = d.getFullYear()
  const m = String(d.getMonth() + 1).padStart(2, '0')
  const day = String(d.getDate()).padStart(2, '0')
  return `${y}-${m}-${day}`
}

/** The last `n` days, oldest first, ending with `today`. */
export function lastDays(n: number, today = new Date()): string[] {
  return Array.from({ length: n }, (_, i) => {
    const d = new Date(today)
    d.setDate(today.getDate() - (n - 1 - i))
    return toISODate(d)
  })
}

/** Consecutive completed days ending today (or yesterday, if today isn't done yet). */
export function currentStreak(done: string[], today = new Date()): number {
  const set = new Set(done)
  const cursor = new Date(today)
  if (!set.has(toISODate(cursor))) cursor.setDate(cursor.getDate() - 1)
  let streak = 0
  while (set.has(toISODate(cursor))) {
    streak++
    cursor.setDate(cursor.getDate() - 1)
  }
  return streak
}

const STORAGE_KEY = 'habit-tracker:v1'

export function loadHabits(): Habit[] {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    const parsed: unknown = raw ? JSON.parse(raw) : []
    return Array.isArray(parsed) ? (parsed as Habit[]) : []
  } catch {
    return []
  }
}

export function saveHabits(habits: Habit[]): void {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(habits))
}
```

Design decisions:

- **Dates are stored as `YYYY-MM-DD` strings in local time**, built by hand rather than
  with `toISOString()`. `toISOString()` converts to UTC, so for someone in India at 2 a.m.
  it would report *yesterday's* date.
- **`done` is an array** rather than a `Set` because it has to go through `JSON.stringify`.
- **The streak forgives "not yet today"**: if today isn't ticked, it counts back from
  yesterday, so your streak doesn't show 0 every morning.
- `loadHabits` never throws — corrupted storage just gives an empty list.

## 3. The form

```vue title="src/components/HabitForm.vue"
<script setup lang="ts">
import { ref, computed } from 'vue'

const { existingNames } = defineProps<{ existingNames: string[] }>()
const emit = defineEmits<{ add: [name: string, color: string] }>()

const name = ref('')
const color = ref('#16a34a')

const trimmed = computed(() => name.value.trim())
const duplicate = computed(() =>
  existingNames.some((n) => n.toLowerCase() === trimmed.value.toLowerCase()),
)
const canAdd = computed(() => trimmed.value.length > 0 && !duplicate.value)

function submit() {
  if (!canAdd.value) return
  emit('add', trimmed.value, color.value)
  name.value = ''
}
</script>

<template>
  <form class="habit-form" @submit.prevent="submit">
    <label for="habit-name">New habit</label>
    <input id="habit-name" v-model="name" maxlength="40" placeholder="e.g. Read 20 pages" />
    <input v-model="color" type="color" aria-label="Habit colour" />
    <button type="submit" :disabled="!canAdd">Add</button>
    <p v-if="duplicate" class="error" role="alert">You already track “{{ trimmed }}”.</p>
  </form>
</template>

<style scoped>
.habit-form { display: flex; flex-wrap: wrap; gap: 0.5rem; align-items: center; }
.habit-form label { width: 100%; font-weight: 600; }
.error { width: 100%; color: #b91c1c; margin: 0; }
</style>
```

The form doesn't know about the habit list. It receives the existing names (so it can
reject duplicates, case-insensitively and ignoring surrounding spaces) and emits `add`.
The parent decides what adding means. The error uses `role="alert"`, so screen readers
announce it as soon as it appears.

## 4. One day cell

```vue title="src/components/DayCell.vue"
<script setup lang="ts">
const { date, color } = defineProps<{ date: string; color: string }>()
const done = defineModel<boolean>({ required: true })

const label = new Date(`${date}T00:00:00`).toLocaleDateString(undefined, {
  weekday: 'short',
  day: 'numeric',
})
</script>

<template>
  <button
    type="button"
    class="cell"
    :class="{ done }"
    :aria-pressed="done"
    :aria-label="`${label}: ${done ? 'done' : 'not done'}`"
    @click="done = !done"
  >
    {{ done ? '✓' : '' }}
  </button>
</template>

<style scoped>
.cell {
  width: 2rem; height: 2rem; border: 2px solid v-bind(color); border-radius: 6px;
  background: transparent; color: white; font-weight: 700; cursor: pointer;
}
.cell.done { background: v-bind(color); }
</style>
```

`DayCell` supports `v-model` through `defineModel`. The habit colour flows into CSS with
`v-bind(color)`, so each row gets its own colour without generating class names. The
visible content is only a checkmark, so the button has an `aria-label` with the day and
state, plus `aria-pressed` to expose it as a toggle.

## 5. One habit row

```vue title="src/components/HabitRow.vue"
<script setup lang="ts">
import { computed } from 'vue'
import DayCell from './DayCell.vue'
import { currentStreak, type Habit } from '../habits'

const { habit, days } = defineProps<{ habit: Habit; days: string[] }>()
const emit = defineEmits<{
  toggle: [id: string, date: string]
  remove: [id: string]
}>()

const streak = computed(() => currentStreak(habit.done))
</script>

<template>
  <li class="row">
    <span class="name">{{ habit.name }}</span>
    <DayCell
      v-for="day in days"
      :key="day"
      :date="day"
      :color="habit.color"
      :model-value="habit.done.includes(day)"
      @update:model-value="emit('toggle', habit.id, day)"
    />
    <span class="streak" :title="`${streak}-day streak`">🔥 {{ streak }}</span>
    <button type="button" class="remove" :aria-label="`Remove ${habit.name}`" @click="emit('remove', habit.id)">×</button>
  </li>
</template>

<style scoped>
.row { display: grid; grid-template-columns: 10rem repeat(7, 2rem) 4rem 2rem; gap: 0.4rem; align-items: center; }
.name { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.remove { border: none; background: none; font-size: 1.25rem; cursor: pointer; }
</style>
```

Look at how `HabitRow` uses `DayCell`. It *doesn't* use `v-model`, because a `DayCell`'s
value is derived (`habit.done.includes(day)`) rather than a variable the row owns. So it
binds the two halves of `v-model` by hand: `:model-value` down, `@update:model-value` up —
and turns the update into a `toggle` event for the parent. The row never mutates
`habit`, which is a prop.

## 6. The root component

```vue title="src/App.vue"
<script setup lang="ts">
import { ref, computed, watch } from 'vue'
import HabitForm from './components/HabitForm.vue'
import HabitRow from './components/HabitRow.vue'
import { lastDays, loadHabits, saveHabits, toISODate, type Habit } from './habits'

const habits = ref<Habit[]>(loadHabits())
const days = lastDays(7)
const today = toISODate(new Date())

watch(habits, (value) => saveHabits(value), { deep: true })

const doneToday = computed(() => habits.value.filter((h) => h.done.includes(today)).length)
const names = computed(() => habits.value.map((h) => h.name))

function addHabit(name: string, color: string) {
  habits.value.push({ id: crypto.randomUUID(), name, color, done: [] })
}

function toggle(id: string, date: string) {
  const habit = habits.value.find((h) => h.id === id)
  if (!habit) return
  const i = habit.done.indexOf(date)
  if (i === -1) habit.done.push(date)
  else habit.done.splice(i, 1)
}

function remove(id: string) {
  habits.value = habits.value.filter((h) => h.id !== id)
}
</script>

<template>
  <main class="app">
    <h1>Habits</h1>
    <p class="summary" aria-live="polite">
      {{ doneToday }} of {{ habits.length }} done today
    </p>

    <HabitForm :existing-names="names" @add="addHabit" />

    <TransitionGroup v-if="habits.length" name="list" tag="ul" class="list">
      <HabitRow
        v-for="habit in habits"
        :key="habit.id"
        :habit="habit"
        :days="days"
        @toggle="toggle"
        @remove="remove"
      />
    </TransitionGroup>
    <p v-else class="empty">No habits yet — add your first one above.</p>
  </main>
</template>

<style scoped>
.app { max-width: 42rem; margin: 2rem auto; font-family: system-ui, sans-serif; }
.list { list-style: none; padding: 0; display: grid; gap: 0.5rem; }
.list-enter-active, .list-leave-active { transition: opacity 200ms, transform 200ms; }
.list-enter-from, .list-leave-to { opacity: 0; transform: translateX(-1rem); }
@media (prefers-reduced-motion: reduce) {
  .list-enter-active, .list-leave-active { transition: none; }
}
</style>
```

All the state lives here: `habits` is the single source of truth, and every change goes
through one of three functions. Because `toggle` mutates an array *inside* a habit, the
persistence watcher needs `{ deep: true }` — a shallow watcher on `habits` would only
notice when the whole array is replaced (as `remove` does).

`<TransitionGroup tag="ul">` renders a real `<ul>` and animates `HabitRow` items as they're
added or removed, keyed by `habit.id`.

Replace `src/main.ts`'s contents with the three-line version:

```ts title="src/main.ts"
import { createApp } from 'vue'
import App from './App.vue'

createApp(App).mount('#app')
```

Run `npm run dev`, add a couple of habits and tick some days, then reload the page — your
data should still be there.

## 7. Tests

```ts title="src/__tests__/habits.spec.ts"
import { describe, it, expect, beforeEach } from 'vitest'
import { mount } from '@vue/test-utils'
import { nextTick } from 'vue'
import App from '../App.vue'
import { currentStreak, lastDays } from '../habits'

describe('streaks', () => {
  const today = new Date('2026-09-29T12:00:00')
  it('counts consecutive days ending today', () => {
    expect(currentStreak(['2026-09-27', '2026-09-28', '2026-09-29'], today)).toBe(3)
  })
  it('still counts yesterday when today is not done yet', () => {
    expect(currentStreak(['2026-09-27', '2026-09-28'], today)).toBe(2)
  })
  it('breaks on a gap', () => {
    expect(currentStreak(['2026-09-25', '2026-09-28', '2026-09-29'], today)).toBe(2)
  })
  it('lists the last 7 days oldest first', () => {
    expect(lastDays(7, today)).toEqual([
      '2026-09-23', '2026-09-24', '2026-09-25', '2026-09-26', '2026-09-27', '2026-09-28', '2026-09-29',
    ])
  })
})

describe('App', () => {
  beforeEach(() => localStorage.clear())

  it('adds a habit, toggles today and persists', async () => {
    const wrapper = mount(App)
    await wrapper.find('#habit-name').setValue('Read')
    await wrapper.find('form').trigger('submit')
    expect(wrapper.findAll('li.row')).toHaveLength(1)

    const cells = wrapper.findAll('button.cell')
    await cells[cells.length - 1]!.trigger('click')   // today is the last cell
    expect(wrapper.find('.summary').text()).toBe('1 of 1 done today')

    await nextTick()
    const saved = JSON.parse(localStorage.getItem('habit-tracker:v1')!)
    expect(saved[0].name).toBe('Read')
    expect(saved[0].done).toHaveLength(1)
  })

  it('rejects duplicate names', async () => {
    const wrapper = mount(App)
    await wrapper.find('#habit-name').setValue('Read')
    await wrapper.find('form').trigger('submit')
    await wrapper.find('#habit-name').setValue('read ')
    expect(wrapper.find('[role="alert"]').text()).toContain('You already track')
    expect(wrapper.find('button[type="submit"]').attributes('disabled')).toBeDefined()
  })
})
```

The streak tests pass a fixed "today" so they don't depend on when you run them. The
component tests drive the app the way a user would: type into the input, submit the form,
click a cell, and check what's on screen and in storage. Our run:

```text
 ✓ src/habit/__tests__/habits.spec.ts > streaks > counts consecutive days ending today 1ms
 ✓ src/habit/__tests__/habits.spec.ts > streaks > still counts yesterday when today is not done yet 0ms
 ✓ src/habit/__tests__/habits.spec.ts > streaks > breaks on a gap 0ms
 ✓ src/habit/__tests__/habits.spec.ts > streaks > lists the last 7 days oldest first 0ms
 ✓ src/habit/__tests__/habits.spec.ts > HabitApp > adds a habit, toggles today and persists 58ms
 ✓ src/habit/__tests__/habits.spec.ts > HabitApp > rejects duplicate names 5ms

 Test Files  1 passed (1)
      Tests  6 passed (6)
```

(We built the project inside a larger scratch app, which is why the paths and the
`HabitApp` name differ slightly from yours.) Run yours with `npx vitest run`, and
type-check with `npx vue-tsc --build`.

## How It Actually Works

Follow a single click on today's cell through the system:

1. The native `click` runs `done = !done` inside `DayCell`. `done` is the `defineModel`
   ref; its setter emits `update:modelValue` with `true`.
2. `HabitRow` listens with `@update:model-value` and emits `toggle(id, date)`.
3. `App.vue`'s `toggle` finds the habit and `push`es the date into `habit.done`. That
   array lives inside `habits.value`, so it is a reactive proxy: `push` triggers every
   effect that read `habit.done` — which includes the `doneToday` computed, `HabitRow`'s
   render and `streak` computed (they read `habit.done` through the prop), and the deep
   watcher.
4. Vue batches all of those. On the next microtask the scheduler runs the `pre` jobs
   (the deep watcher saves to storage), then re-renders the components whose render
   effects were triggered — `App` (the summary text) and that one `HabitRow`. Other rows
   were not triggered, so they don't re-render at all.
5. The `HabitRow` re-render produces a new `modelValue` of `true` for that `DayCell`,
   which re-renders with the `done` class and a checkmark.

Notice that step 3 mutates a habit object that is also a *prop* of `HabitRow`. That's fine,
because the mutation happens in the owner (`App`), not in the child — the one-way data flow
rule is about who writes, not about object identity.

## Common mistakes

- **Using `toISOString().slice(0, 10)` for "today"** — it's UTC, not the user's day.
- **A shallow watcher for persistence** — ticking a day would not save.
- **`:key="index"` on the rows** — removing a habit would animate the wrong row out and
  shift everyone's colours for a frame.
- **Letting child components mutate `habit.done`** directly — it works, but you lose the
  single place where state changes, which is exactly what you'll want when this moves into
  a Pinia store in Level 2.
- **Testing with the real date** — tests that pass today and fail at midnight.

## Exercise

Extend the tracker:

1. **Rename in place.** Double-clicking a habit name turns it into an input (use `v-if`
   and `useTemplateRef` to focus it). Enter saves, Escape cancels. Reject duplicates using
   the same rule as `HabitForm`.
2. **Longest streak.** Add a `longestStreak(done)` function to `habits.ts`, with tests for
   an empty list, a single day, and two separate runs.
3. **Weekly goal.** Add an optional `goal` (1–7 days per week) to each habit and show
   "3/5 this week" with a `ProgressBar` from Lesson 09.
4. **Export/import.** Add buttons to download the habits as JSON and to import a JSON file,
   validating that the file has the right shape before replacing state.
5. **Storage versioning.** The storage key ends in `:v1`. Add a `goal` field in a way that
   still loads v1 data correctly, and write a test for it.
