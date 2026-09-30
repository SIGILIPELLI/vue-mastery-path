---
description: "Level 3 project: an accessible Vue Kanban board with drag and drop, a keyboard alternative, WIP limits, animated moves, a native dialog editor and tests."
---

# 10 · Project — A Kanban Board

The Level 3 project is a Kanban board: three columns, cards you can drag between them *or*
move with the keyboard, a work-in-progress limit on the middle column, animated
reordering, and a dialog for editing card details. It pulls together reactivity choices,
`TransitionGroup`, custom keyboard interaction, live-region announcements, error handling
for rule violations, and a test suite that had to work around three real jsdom/test-utils
limitations. All ten tests pass, `vue-tsc` is clean and `vite build` succeeds.

| Concept | Where | Lesson |
|---------|-------|--------|
| Derived state with stable ordering | `byColumn` computed | 01 |
| `TransitionGroup` move animations | `KanbanColumn` | 04 |
| Native drag and drop + keyboard alternative | `KanbanCard`, `KanbanColumn` | 08 |
| Live-region announcements, focus restoration | `App.vue` | 08 |
| Rule violations as errors, surfaced to the user | store + `run()` in `App.vue` | 09 |
| Native `<dialog>` editing a copy | `CardDialog` | 04, 08 |
| Store and full-app tests | `__tests__/` | 07 |

## 1. Scaffold

```bash
npm create vue@latest kanban -- --ts --pinia --vitest --bare
cd kanban
npm install
```

Delete `src/stores/counter.ts` and `src/__tests__/App.spec.ts`, and add the Node 25+
`execArgv` line to `vitest.config.ts` if needed (Level 1 · 10).

## 2. The store

```ts title="src/stores/board.ts"
import { ref, computed, watch } from 'vue'
import { defineStore } from 'pinia'

export type ColumnId = 'todo' | 'doing' | 'done'
export interface Card { id: string; title: string; notes: string; column: ColumnId; order: number }

export const COLUMNS: { id: ColumnId; title: string; limit?: number }[] = [
  { id: 'todo', title: 'To do' },
  { id: 'doing', title: 'In progress', limit: 3 },
  { id: 'done', title: 'Done' },
]

const STORAGE_KEY = 'kanban:v1'

function load(): Card[] {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    return raw ? (JSON.parse(raw) as Card[]) : []
  } catch {
    return []
  }
}

export const useBoardStore = defineStore('board', () => {
  const cards = ref<Card[]>(load())

  watch(cards, (value) => localStorage.setItem(STORAGE_KEY, JSON.stringify(value)), { deep: true })

  const byColumn = computed(() => {
    const out: Record<ColumnId, Card[]> = { todo: [], doing: [], done: [] }
    for (const card of cards.value) out[card.column].push(card)
    for (const list of Object.values(out)) list.sort((a, b) => a.order - b.order)
    return out
  })

  function isFull(column: ColumnId, ignoringId?: string): boolean {
    const limit = COLUMNS.find((c) => c.id === column)?.limit
    if (limit === undefined) return false
    return byColumn.value[column].filter((c) => c.id !== ignoringId).length >= limit
  }

  function add(title: string, column: ColumnId = 'todo'): Card {
    const trimmed = title.trim()
    if (!trimmed) throw new Error('A card needs a title')
    if (isFull(column)) throw new Error(`“${COLUMNS.find((c) => c.id === column)!.title}” is full`)
    const card: Card = { id: crypto.randomUUID(), title: trimmed, notes: '', column, order: byColumn.value[column].length }
    cards.value.push(card)
    return card
  }

  /** Move a card to `column` at position `index` (0-based), renumbering both columns. */
  function move(id: string, column: ColumnId, index: number) {
    const card = cards.value.find((c) => c.id === id)
    if (!card) return
    if (card.column !== column && isFull(column, id)) {
      throw new Error(`“${COLUMNS.find((c) => c.id === column)!.title}” is full`)
    }
    const from = card.column
    const target = byColumn.value[column].filter((c) => c.id !== id)
    const clamped = Math.max(0, Math.min(index, target.length))
    target.splice(clamped, 0, card)
    card.column = column
    target.forEach((c, i) => (c.order = i))
    if (from !== column) byColumn.value[from].filter((c) => c.id !== id).forEach((c, i) => (c.order = i))
  }

  function update(id: string, patch: Partial<Pick<Card, 'title' | 'notes'>>) {
    const card = cards.value.find((c) => c.id === id)
    if (card) Object.assign(card, patch)
  }

  function remove(id: string) {
    const card = cards.value.find((c) => c.id === id)
    if (!card) return
    cards.value = cards.value.filter((c) => c.id !== id)
    byColumn.value[card.column].forEach((c, i) => (c.order = i))
  }

  return { cards, byColumn, isFull, add, move, update, remove }
})
```

Design notes:

- **Cards are one flat array** with a `column` and an `order`. Columns are *derived*
  (`byColumn`), so there's exactly one copy of each card — moving a card is changing two
  fields, not removing from one array and inserting into another.
- **`order` is renumbered 0..n** on every move, which keeps sorting trivial and makes the
  tests readable. For boards with thousands of cards synced to a server, you'd use
  fractional or lexicographic ordering to avoid rewriting every card; that's an
  optimisation this app doesn't need.
- **WIP limits throw.** A limit violation isn't a bug, but it's an action that can't
  happen; throwing keeps the store's rules in the store, and the UI decides how to present
  the message. Reordering *within* a full column is still allowed (`ignoringId`).

## 3. A card

```vue title="src/components/KanbanCard.vue"
<script setup lang="ts">
import { COLUMNS, type Card } from '@/stores/board'

const { card, index, total } = defineProps<{ card: Card; index: number; total: number }>()
const emit = defineEmits<{
  open: [id: string]
  move: [id: string, column: Card['column'], index: number]
}>()

const columnIndex = COLUMNS.findIndex((c) => c.id === card.column)

function onDragStart(e: DragEvent) {
  e.dataTransfer?.setData('text/plain', card.id)
  if (e.dataTransfer) e.dataTransfer.effectAllowed = 'move'
}

// keyboard alternative to drag and drop: Alt + arrow keys
function onKeydown(e: KeyboardEvent) {
  if (!e.altKey) return
  const moves: Record<string, [number, number] | undefined> = {
    ArrowUp: [0, index - 1],
    ArrowDown: [0, index + 1],
    ArrowLeft: [-1, 0],
    ArrowRight: [1, 0],
  }
  const m = moves[e.key]
  if (!m) return
  const target = COLUMNS[columnIndex + m[0]]
  if (!target || (m[0] === 0 && (m[1] < 0 || m[1] >= total))) return
  e.preventDefault()
  emit('move', card.id, target.id, m[1])
}
</script>

<template>
  <li
    class="card"
    draggable="true"
    :data-id="card.id"
    @dragstart="onDragStart"
  >
    <button
      type="button"
      class="card-button"
      :aria-describedby="'kanban-help'"
      @click="emit('open', card.id)"
      @keydown="onKeydown"
    >
      {{ card.title }}
      <span v-if="card.notes" class="has-notes" aria-label="has notes">📝</span>
    </button>
  </li>
</template>

<style scoped>
.card { list-style: none; }
.card-button {
  width: 100%; text-align: left; padding: 0.6rem 0.75rem; border: 1px solid #d1d5db;
  border-radius: 6px; background: white; font: inherit; cursor: grab;
}
.card-button:focus-visible { outline: 3px solid #2563eb; outline-offset: 2px; }
</style>
```

Two ways to move a card:

- **Drag and drop** with the native HTML Drag and Drop API: `draggable="true"` and the card
  id in `dataTransfer`.
- **Keyboard**: with the card's button focused, **Alt + arrows** move it up/down within the
  column or left/right between columns. Drag and drop alone is inaccessible to keyboard and
  many assistive-technology users, so an alternative isn't optional. (Alt avoids clashing
  with plain arrow keys, which screen readers use for reading.)

The card is a real `<button>` inside the `<li>`, so it's focusable and opens the editor with
Enter or Space.

## 4. A column

```vue title="src/components/KanbanColumn.vue"
<script setup lang="ts">
import { ref } from 'vue'
import KanbanCard from './KanbanCard.vue'
import type { Card, ColumnId } from '@/stores/board'

const { id, title, cards, limit } = defineProps<{ id: ColumnId; title: string; cards: Card[]; limit?: number }>()
const emit = defineEmits<{
  open: [id: string]
  move: [id: string, column: ColumnId, index: number]
  add: [title: string, column: ColumnId]
}>()

const dragOver = ref(false)
const newTitle = ref('')

/** Where in the list was the card dropped? Count cards whose middle is above the pointer. */
function dropIndex(e: DragEvent): number {
  const items = [...(e.currentTarget as HTMLElement).querySelectorAll<HTMLElement>('li.card')]
  return items.filter((el) => {
    const box = el.getBoundingClientRect()
    return box.top + box.height / 2 < e.clientY
  }).length
}

function onDrop(e: DragEvent) {
  dragOver.value = false
  const cardId = e.dataTransfer?.getData('text/plain')
  if (cardId) emit('move', cardId, id, dropIndex(e))
}

function submit() {
  if (!newTitle.value.trim()) return
  emit('add', newTitle.value, id)
  newTitle.value = ''
}
</script>

<template>
  <section class="column" :class="{ 'drag-over': dragOver }" :aria-labelledby="`col-${id}`">
    <h2 :id="`col-${id}`">
      {{ title }}
      <span class="count">{{ cards.length }}<template v-if="limit"> / {{ limit }}</template></span>
    </h2>
    <TransitionGroup
      tag="ul"
      name="cards"
      class="cards"
      @dragover.prevent="dragOver = true"
      @dragleave="dragOver = false"
      @drop.prevent="onDrop"
    >
      <KanbanCard
        v-for="(card, i) in cards"
        :key="card.id"
        :card="card"
        :index="i"
        :total="cards.length"
        @open="emit('open', $event)"
        @move="(cardId, col, idx) => emit('move', cardId, col, idx)"
      />
    </TransitionGroup>
    <form class="add" @submit.prevent="submit">
      <label :for="`add-${id}`" class="visually-hidden">New card in {{ title }}</label>
      <input :id="`add-${id}`" v-model="newTitle" placeholder="Add a card…" />
    </form>
  </section>
</template>

<style scoped>
.column { background: #f3f4f6; border-radius: 8px; padding: 0.75rem; min-height: 12rem; display: flex; flex-direction: column; }
.column.drag-over { outline: 2px dashed #2563eb; }
.column h2 { font-size: 1rem; display: flex; justify-content: space-between; margin: 0 0 0.5rem; }
.count { color: #6b7280; font-weight: 400; }
.cards { display: grid; gap: 0.5rem; padding: 0; margin: 0; flex: 1; align-content: start; min-height: 3rem; }
.cards-move, .cards-enter-active, .cards-leave-active { transition: all 200ms ease; }
.cards-enter-from, .cards-leave-to { opacity: 0; transform: scale(0.97); }
.cards-leave-active { position: absolute; }
@media (prefers-reduced-motion: reduce) {
  .cards-move, .cards-enter-active, .cards-leave-active { transition: none; }
}
.add input { width: 100%; margin-top: 0.5rem; padding: 0.4rem; box-sizing: border-box; }
</style>
```

`dropIndex` works out where in the list the card was dropped by counting cards whose
vertical midpoint is above the pointer. The `TransitionGroup` with a `cards-move` class
animates the other cards sliding into place, and `.cards-leave-active { position: absolute }`
lets them move while the leaving card fades.

## 5. The dialog

```vue title="src/components/CardDialog.vue"
<script setup lang="ts">
import { ref, watch, useTemplateRef } from 'vue'
import type { Card } from '@/stores/board'

const { card } = defineProps<{ card: Card | null }>()
const emit = defineEmits<{
  save: [id: string, patch: { title: string; notes: string }]
  remove: [id: string]
  close: []
}>()

const dialog = useTemplateRef<HTMLDialogElement>('dialog')
const title = ref('')
const notes = ref('')

// edit a copy, so Cancel really cancels
watch(
  () => card,
  (c) => {
    if (c) {
      title.value = c.title
      notes.value = c.notes
      if (!dialog.value?.open) dialog.value?.showModal()
    } else {
      dialog.value?.close()
    }
  },
  { flush: 'post' },
)

function save() {
  if (!card || !title.value.trim()) return
  emit('save', card.id, { title: title.value.trim(), notes: notes.value })
}
</script>

<template>
  <dialog ref="dialog" aria-labelledby="card-dialog-title" @close="emit('close')">
    <form v-if="card" method="dialog" @submit.prevent="save">
      <h2 id="card-dialog-title">Edit card</h2>
      <label for="cd-title">Title</label>
      <input id="cd-title" v-model="title" required />
      <label for="cd-notes">Notes</label>
      <textarea id="cd-notes" v-model="notes" rows="5" />
      <div class="actions">
        <button type="button" class="danger" @click="emit('remove', card.id)">Delete</button>
        <button type="button" @click="emit('close')">Cancel</button>
        <button type="submit">Save</button>
      </div>
    </form>
  </dialog>
</template>

<style scoped>
dialog { width: min(32rem, 90vw); border: none; border-radius: 10px; padding: 1.25rem; }
dialog::backdrop { background: rgb(0 0 0 / 0.4); }
form { display: grid; gap: 0.4rem; }
.actions { display: flex; gap: 0.5rem; justify-content: flex-end; margin-top: 0.75rem; }
.danger { margin-right: auto; color: #b91c1c; }
</style>
```

The dialog edits **local copies** of the title and notes, so Cancel genuinely discards
changes; only Save emits. The watcher runs with `flush: 'post'` so the `<dialog>` element
exists before `showModal()`. The native `<dialog>` gives focus trapping, Escape to close
and an inert background for free.

## 6. The app

```vue title="src/App.vue"
<script setup lang="ts">
import { ref, computed } from 'vue'
import { useBoardStore, COLUMNS, type ColumnId } from '@/stores/board'
import KanbanColumn from '@/components/KanbanColumn.vue'
import CardDialog from '@/components/CardDialog.vue'

const board = useBoardStore()
const openId = ref<string | null>(null)
const openCard = computed(() => board.cards.find((c) => c.id === openId.value) ?? null)
const status = ref('')

function run(action: () => void, success?: string) {
  try {
    action()
    if (success) status.value = success
  } catch (e) {
    status.value = (e as Error).message
  }
}

function onMove(id: string, column: ColumnId, index: number) {
  const card = board.cards.find((c) => c.id === id)
  const label = COLUMNS.find((c) => c.id === column)!.title
  run(() => board.move(id, column, index), card && `Moved “${card.title}” to ${label}, position ${index + 1}`)
  // keep keyboard focus on the moved card after it re-renders in its new place
  requestAnimationFrame(() =>
    document.querySelector<HTMLButtonElement>(`li[data-id="${id}"] button`)?.focus(),
  )
}
</script>

<template>
  <main class="board-page">
    <h1>Board</h1>
    <p id="kanban-help" class="hint">Drag cards between columns, or focus a card and use Alt + arrow keys.</p>
    <p class="visually-hidden" aria-live="polite">{{ status }}</p>
    <p v-if="status.endsWith('full')" class="warning" role="alert">{{ status }}</p>

    <div class="board">
      <KanbanColumn
        v-for="col in COLUMNS"
        :id="col.id"
        :key="col.id"
        :title="col.title"
        :limit="col.limit"
        :cards="board.byColumn[col.id]"
        @open="openId = $event"
        @move="onMove"
        @add="(title, column) => run(() => board.add(title, column), `Added “${title.trim()}”`)"
      />
    </div>

    <CardDialog
      :card="openCard"
      @save="(id, patch) => { board.update(id, patch); openId = null }"
      @remove="(id) => { board.remove(id); openId = null }"
      @close="openId = null"
    />
  </main>
</template>

<style>
body { margin: 0; font-family: system-ui, sans-serif; }
.board-page { padding: 1rem; }
.board { display: grid; grid-template-columns: repeat(3, minmax(14rem, 1fr)); gap: 1rem; }
.hint { color: #4b5563; }
.warning { color: #92400e; background: #fef3c7; padding: 0.5rem; border-radius: 6px; }
.visually-hidden { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0 0 0 0); white-space: nowrap; }
</style>
```

- `run()` turns store errors (a full column, an empty title) into a status message. The
  message goes to a polite live region; "is full" messages are *also* shown visibly as an
  alert.
- After a move, focus is put back on the moved card, found by `data-id` after it has
  re-rendered in its new column — without that, keyboard users lose their place after every
  move, because the old button element was removed.
- The description in `#kanban-help` is linked from every card with `aria-describedby`, so
  screen readers announce how to move cards.

## 7. Tests

The store tests are straightforward:

```ts title="src/stores/__tests__/board.spec.ts"
import { describe, it, expect, beforeEach } from 'vitest'
import { createPinia, setActivePinia } from 'pinia'
import { useBoardStore } from '../board'

const titles = (cards: { title: string }[]) => cards.map((c) => c.title)

describe('board store', () => {
  beforeEach(() => {
    localStorage.clear()
    setActivePinia(createPinia())
  })

  it('adds cards to the end of a column', () => {
    const board = useBoardStore()
    board.add('A'); board.add('B'); board.add('C', 'done')
    expect(titles(board.byColumn.todo)).toEqual(['A', 'B'])
    expect(titles(board.byColumn.done)).toEqual(['C'])
  })

  it('reorders within a column and renumbers', () => {
    const board = useBoardStore()
    const a = board.add('A'); board.add('B'); board.add('C')
    board.move(a.id, 'todo', 2)
    expect(titles(board.byColumn.todo)).toEqual(['B', 'C', 'A'])
    expect(board.byColumn.todo.map((c) => c.order)).toEqual([0, 1, 2])
  })

  it('moves across columns and closes the gap behind it', () => {
    const board = useBoardStore()
    board.add('A'); const b = board.add('B'); board.add('C')
    board.move(b.id, 'doing', 0)
    expect(titles(board.byColumn.todo)).toEqual(['A', 'C'])
    expect(board.byColumn.todo.map((c) => c.order)).toEqual([0, 1])
    expect(titles(board.byColumn.doing)).toEqual(['B'])
  })

  it('enforces the WIP limit on "In progress"', () => {
    const board = useBoardStore()
    for (const t of ['1', '2', '3']) board.add(t, 'doing')
    const extra = board.add('4')
    expect(() => board.move(extra.id, 'doing', 0)).toThrow('“In progress” is full')
    // reordering inside a full column is still allowed
    board.move(board.byColumn.doing[0]!.id, 'doing', 2)
    expect(titles(board.byColumn.doing)).toEqual(['2', '3', '1'])
  })

  it('persists to localStorage', async () => {
    useBoardStore().add('Persist me')
    await Promise.resolve()   // the deep watcher flushes before the next render
    await new Promise((r) => setTimeout(r))
    expect(localStorage.getItem('kanban:v1')).toContain('Persist me')
  })
})
```

The app tests drive the real components — and our first run had **five of ten** failing.
Each failure taught something about the test environment rather than the app:

1. **`TypeError: dialog.value?.showModal is not a function`** — jsdom implements the
   `<dialog>` element but not `showModal()`/`close()`. The tests add minimal versions in
   `beforeAll`.
2. **Queries found nothing** (`expected [] to deeply equal [ 'Fix bug' ]`) — Vue Test Utils
   **stubs `<Transition>` and `<TransitionGroup>` by default**, rendering a
   `<transition-group-stub>` element instead of our `<ul>`. Printing `wrapper.html()` showed
   it. `stubs: { 'transition-group': false }` renders the real thing.
3. With the real `TransitionGroup`, the keyboard-move test then saw the moved card in
   **both** columns: it appeared in "In progress" immediately, but the old `<li>` stayed in
   "To do" until its leave transition finished a few frames later. We logged the columns
   immediately and 50 ms later to confirm, then used `vi.waitFor`.

```ts title="src/components/__tests__/App.spec.ts"
import { describe, it, expect, beforeEach, beforeAll, vi } from 'vitest'
import { mount, flushPromises } from '@vue/test-utils'
import { createPinia, setActivePinia } from 'pinia'
import App from '@/App.vue'
import { useBoardStore } from '@/stores/board'

function setup() {
  const pinia = createPinia()
  setActivePinia(pinia)
  const board = useBoardStore()
  board.add('Write tests'); board.add('Fix bug')
  const wrapper = mount(App, {
    global: {
      plugins: [pinia],
      // VTU replaces <TransitionGroup> with a stub by default; we need the real <ul>
      stubs: { 'transition-group': false },
    },
    attachTo: document.body,
  })
  return { wrapper, board }
}

const list = (id: string) => `section[aria-labelledby="col-${id}"] ul`
const columnTitles = (wrapper: ReturnType<typeof mount>, id: string) =>
  wrapper.findAll(`${list(id)} li button`).map((b) => b.text())

// jsdom has <dialog> but not showModal()/close() — add minimal versions
beforeAll(() => {
  HTMLDialogElement.prototype.showModal ??= function (this: HTMLDialogElement) {
    this.open = true
  }
  HTMLDialogElement.prototype.close ??= function (this: HTMLDialogElement) {
    this.open = false
    this.dispatchEvent(new Event('close'))
  }
})

describe('Kanban board', () => {
  beforeEach(() => {
    localStorage.clear()
    document.body.innerHTML = ''
  })

  it('moves a card to the next column with Alt+ArrowRight and announces it', async () => {
    const { wrapper } = setup()
    await wrapper.find('li.card button').trigger('keydown', { key: 'ArrowRight', altKey: true })
    // the card appears in its new column at once, but the old <li> stays in the DOM
    // until its leave transition finishes a few frames later
    expect(columnTitles(wrapper, 'doing')).toEqual(['Write tests'])
    await vi.waitFor(() => expect(columnTitles(wrapper, 'todo')).toEqual(['Fix bug']))
    expect(wrapper.find('[aria-live]').text()).toBe('Moved “Write tests” to In progress, position 1')
  })

  it('ignores arrow keys without Alt', async () => {
    const { wrapper } = setup()
    await wrapper.find('li.card button').trigger('keydown', { key: 'ArrowRight' })
    expect(columnTitles(wrapper, 'todo')).toEqual(['Write tests', 'Fix bug'])
  })

  it('moves a card with drag and drop', async () => {
    const { wrapper, board } = setup()
    const id = board.byColumn.todo[0]!.id
    const data = new Map<string, string>()
    const dataTransfer = { setData: (k: string, v: string) => data.set(k, v), getData: (k: string) => data.get(k) ?? '' }

    await wrapper.find(`li[data-id="${id}"]`).trigger('dragstart', { dataTransfer })
    await wrapper.find(list('done')).trigger('drop', { dataTransfer, clientY: 0 })
    expect(columnTitles(wrapper, 'done')).toEqual(['Write tests'])
  })

  it('adds a card from the column input', async () => {
    const { wrapper } = setup()
    await wrapper.find('#add-done').setValue('Ship it')
    await wrapper.find('#add-done').trigger('submit')
    expect(columnTitles(wrapper, 'done')).toEqual(['Ship it'])
  })

  it('edits a card in the dialog and cancel discards changes', async () => {
    const { wrapper, board } = setup()
    await wrapper.find('li.card button').trigger('click')
    await flushPromises()
    await wrapper.find('#cd-title').setValue('Changed title')
    await wrapper.findAll('.actions button').find((b) => b.text() === 'Cancel')!.trigger('click')
    expect(board.byColumn.todo[0]!.title).toBe('Write tests')

    await wrapper.find('li.card button').trigger('click')
    await flushPromises()
    await wrapper.find('#cd-notes').setValue('Cover the store first')
    await wrapper.find('dialog form').trigger('submit')
    expect(board.byColumn.todo[0]!.notes).toBe('Cover the store first')
  })
})
```

```text
 ✓ src/stores/__tests__/board.spec.ts > board store > adds cards to the end of a column 4ms
 ✓ src/stores/__tests__/board.spec.ts > board store > reorders within a column and renumbers 1ms
 ✓ src/stores/__tests__/board.spec.ts > board store > moves across columns and closes the gap behind it 1ms
 ✓ src/stores/__tests__/board.spec.ts > board store > enforces the WIP limit on "In progress" 1ms
 ✓ src/stores/__tests__/board.spec.ts > board store > persists to localStorage 2ms
 ✓ src/components/__tests__/App.spec.ts > Kanban board > moves a card to the next column with Alt+ArrowRight and announces it 127ms
 ✓ src/components/__tests__/App.spec.ts > Kanban board > ignores arrow keys without Alt 5ms
 ✓ src/components/__tests__/App.spec.ts > Kanban board > moves a card with drag and drop 7ms
 ✓ src/components/__tests__/App.spec.ts > Kanban board > adds a card from the column input 6ms
 ✓ src/components/__tests__/App.spec.ts > Kanban board > edits a card in the dialog and cancel discards changes 8ms
      Tests  10 passed (10)
```

**What these tests don't prove.** jsdom has no layout, so `getBoundingClientRect()` returns
zeros: the drag-and-drop test drops at `clientY: 0` and can only check that the card
arrives *first* in the target column, not that mid-list positioning works. Real
drag-and-drop gestures, animation smoothness and focus visibility need a real browser —
exercise them by hand, or with Playwright
([Playwright Mastery Path](https://sigilipelli.github.io/playwright-mastery-path/)), whose
`dragTo()` performs real pointer events. We verified the build and the jsdom tests; we did
not record a browser session for this page.

## How It Actually Works

Follow a keyboard move — Alt+→ on "Write tests":

1. `KanbanCard` emits `move(id, 'doing', 0)`; `KanbanColumn` re-emits it; `App.onMove` calls
   `board.move`.
2. The store sets `card.column = 'doing'` and renumbers `order` in both columns. Those are
   writes to properties of objects inside `cards.value` — each triggers the effects that
   read them: `byColumn` (it read every card's `column` and `order`) and the deep
   persistence watcher.
3. On the next flush, the persistence watcher (a `pre` job) saves to `localStorage`, then
   `App` re-renders (it read `board.byColumn[...]` for each column's `cards` prop).
   `byColumn` recomputes once for that render.
4. Each `KanbanColumn` receives a new `cards` array. "To do" and "In progress" re-render;
   "Done" gets a new array too (the computed builds fresh arrays), so it re-renders as well,
   but its `TransitionGroup` finds the same keys and does no DOM work. (This is where the
   computed-stability technique from Lesson 01 could save a render, if profiling ever showed
   it mattered.)
5. `TransitionGroup` in "To do" sees the key disappear and runs the leave transition, keeping
   the `<li>` until it completes; in "In progress" it inserts the new `<li>` with an enter
   transition; siblings that changed position get the FLIP `move` animation.
6. `requestAnimationFrame` runs after that render, finds the new button by `data-id`, and
   focuses it; the live region's text changed, so the screen reader announces the move.

## Common mistakes

- **Drag and drop with no keyboard alternative.**
- **Losing focus after a move** because the focused element was destroyed and recreated.
- **Moving data between arrays in two steps** (remove, then insert) — an error between the
  steps loses the card. Derive columns from one list, or do both in one synchronous action.
- **Editing the store's card object directly in the dialog** — Cancel can't undo it.
- **Trusting jsdom for layout-dependent behaviour.**

## Exercise

1. **Undo.** Keep a history of the last 20 board states (`structuredClone` of `cards` before
   each action) and add Ctrl/Cmd+Z. Where should the history live, and what should *not* be
   undoable?
2. **Column WIP limits from settings.** Let users set each column's limit in a settings
   dialog. Moving a card into an over-limit column should show the alert and leave the card
   where it was.
3. **Playwright test.** Write one end-to-end test that drags a card from "To do" to the
   middle of "Done" with `dragTo()` and asserts its position.
4. **Multiple boards.** Add Vue Router with `/boards/:id`, keeping each board's cards under
   its own storage key, and reuse the same store with a `boardId` state (or one store per
   board — compare the trade-offs).
