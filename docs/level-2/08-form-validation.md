---
description: "Validate Vue forms properly: a tested useForm composable with rules, touched state, submit handling and server errors, accessible error messages, and Zod schemas."
---

# 08 · Form Validation

Level 1 bound inputs with `v-model`. Real forms also have to answer: *Is this value
acceptable? When should the user be told it isn't? What happens while submitting? What if
the server rejects it anyway?* This lesson builds a small, fully tested validation
composable so you understand every moving part, then shows how schema libraries such as
Zod and form libraries such as VeeValidate fit in.

## What good validation UX looks like

Validation is mostly about **timing**:

1. **Don't shout at people who haven't finished typing.** An empty required field
   shouldn't be red the moment the page loads, and an email shouldn't be "invalid" after
   the first character.
2. **Show a field's error once the user leaves it** (on `blur`) — they've told you they're
   done with it.
3. **On submit, show every error**, move focus to the first invalid field, and don't send
   the request.
4. **Once an error is showing, update it live** as they fix it, so it disappears the moment
   the value becomes valid.
5. **Server errors map to fields** when they're about a field ("email already
   registered"), and disappear when that field is edited.
6. **Disable double submits**, and show that something is happening.

Also: keep the browser's built-in checks meaningful (`type="email"`, `required`,
`autocomplete`) for autofill and mobile keyboards, but add `novalidate` to the form so the
browser's own bubbles don't compete with your messages. **Client-side validation is for
UX only** — the server must validate everything again.

## A `useForm` composable

```ts title="src/composables/useForm.ts"
import { reactive, computed, ref, toRaw } from 'vue'

export type Rule<V, F> = (value: V, form: F) => string | true
export type Rules<F> = { [K in keyof F]?: Rule<F[K], F>[] }

export function useForm<F extends Record<string, unknown>>(initial: F, rules: Rules<F>) {
  const values = reactive({ ...initial }) as F
  // cast: TypeScript can't index reactive()'s mapped type with a generic key
  const touched = reactive({}) as Partial<Record<keyof F, boolean>>
  const serverErrors = reactive({}) as Partial<Record<keyof F, string>>
  const submitting = ref(false)
  const submitted = ref(false)

  // every field's first failing rule, recomputed whenever any value changes
  const errors = computed(() => {
    const out: Partial<Record<keyof F, string>> = {}
    for (const key of Object.keys(rules) as (keyof F)[]) {
      for (const rule of rules[key] ?? []) {
        const result = rule(values[key], values)
        if (result !== true) {
          out[key] = result
          break
        }
      }
      if (!out[key] && serverErrors[key]) out[key] = serverErrors[key]
    }
    return out
  })

  const isValid = computed(() => Object.keys(errors.value).length === 0)

  /** The error to *show*: only after the field was touched or a submit was attempted. */
  function visibleError(key: keyof F): string | undefined {
    return touched[key] || submitted.value ? errors.value[key] : undefined
  }

  function touch(key: keyof F) {
    touched[key] = true
  }

  function setField<K extends keyof F>(key: K, value: F[K]) {
    values[key] = value
    delete serverErrors[key] // editing a field clears the server's complaint about it
  }

  function handleSubmit(
    onValid: (values: F) => Promise<void | Partial<Record<keyof F, string>>>,
  ) {
    return async () => {
      submitted.value = true
      if (!isValid.value || submitting.value) return
      submitting.value = true
      try {
        const fieldErrors = await onValid(structuredClone(toRaw(values)) as F)
        if (fieldErrors) Object.assign(serverErrors, fieldErrors)
      } finally {
        submitting.value = false
      }
    }
  }

  return { values, errors, isValid, touched, submitting, visibleError, touch, setField, handleSubmit }
}

// small, reusable rules
export const required =
  (message = 'This field is required') =>
  (v: unknown) =>
    (typeof v === 'string' ? v.trim().length > 0 : v != null) || message

export const minLength = (n: number) => (v: string) =>
  v.length >= n || `Must be at least ${n} characters`

export const email = () => (v: string) =>
  /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v) || 'Enter a valid email address'
```

How it's put together:

- **`errors` is a `computed`**, not state you update by hand. Every rule for every field is
  re-evaluated when any value changes, so cross-field rules (confirm password) just work.
  For forms with dozens of fields and expensive rules, you'd split this into a computed per
  field; for typical forms, one computed is simple and fast.
- **Rules are plain functions** returning `true` or a message. The second argument is the
  whole form, for cross-field checks.
- **`visibleError`** implements the timing rules: errors exist all the time, but are only
  *shown* after `touch` (blur) or a submit attempt.
- **`handleSubmit`** returns an event handler. It marks the form as submitted (revealing all
  errors), refuses to run while invalid or already submitting, and lets the submit
  function **return field errors** from the server.
- **`structuredClone(toRaw(values))`** hands the submit function a plain copy, so async
  code can't be confused by the user editing fields mid-request. `toRaw` is needed because
  `structuredClone` can't clone a Proxy.
- The TypeScript cast on `touched` and `serverErrors` works around a limitation: TypeScript
  can't index `reactive()`'s mapped return type with a generic key. We hit six `TS2536`
  errors from `vue-tsc` before adding it.

## Using it: an accessible signup form

```vue title="src/components/SignupCard.vue"
<script setup lang="ts">
import { useForm, required, minLength, email } from '@/composables/useForm'

const emit = defineEmits<{ registered: [email: string] }>()

const { values, visibleError, touch, setField, submitting, handleSubmit } = useForm(
  { email: '', password: '', confirm: '' },
  {
    email: [required(), email()],
    password: [required(), minLength(8)],
    confirm: [required('Please repeat the password'), (v, f) => v === f.password || 'Passwords do not match'],
  },
)

const onSubmit = handleSubmit(async (data) => {
  const res = await fetch('/api/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email: data.email, password: data.password }),
  })
  if (res.status === 409) return { email: 'An account with this email already exists' }
  if (!res.ok) throw new Error(`Registration failed (${res.status})`)
  emit('registered', data.email)
})
</script>

<template>
  <form novalidate @submit.prevent="onSubmit">
    <div class="field">
      <label for="su-email">Email</label>
      <input
        id="su-email"
        type="email"
        autocomplete="email"
        :value="values.email"
        :aria-invalid="!!visibleError('email')"
        aria-describedby="su-email-error"
        @input="setField('email', ($event.target as HTMLInputElement).value)"
        @blur="touch('email')"
      />
      <p id="su-email-error" class="error">{{ visibleError('email') }}</p>
    </div>

    <div class="field">
      <label for="su-password">Password</label>
      <input
        id="su-password"
        v-model="values.password"
        type="password"
        autocomplete="new-password"
        :aria-invalid="!!visibleError('password')"
        aria-describedby="su-password-error"
        @blur="touch('password')"
      />
      <p id="su-password-error" class="error">{{ visibleError('password') }}</p>
    </div>

    <div class="field">
      <label for="su-confirm">Repeat password</label>
      <input
        id="su-confirm"
        v-model="values.confirm"
        type="password"
        autocomplete="new-password"
        :aria-invalid="!!visibleError('confirm')"
        aria-describedby="su-confirm-error"
        @blur="touch('confirm')"
      />
      <p id="su-confirm-error" class="error">{{ visibleError('confirm') }}</p>
    </div>

    <button type="submit" :disabled="submitting">{{ submitting ? 'Creating account…' : 'Create account' }}</button>
  </form>
</template>
```

Accessibility details that matter here:

- Each input has a real `<label for>`.
- **`aria-invalid`** marks the field as invalid for screen readers only when an error is
  visible.
- **`aria-describedby`** points at the error paragraph, so the message is read along with
  the field. The paragraph always exists (empty when there's no error) so the reference is
  never broken.
- `autocomplete="new-password"` lets password managers offer to generate one.

The email field uses `:value` + `@input` with `setField` instead of `v-model`, because
editing the email must also clear a server error about it. The password fields use plain
`v-model` on `values` — both styles work because `values` is reactive.

## Testing it

These tests drive the component like a user and cover each timing rule:

```ts title="src/components/__tests__/SignupCard.spec.ts"
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mount, flushPromises } from '@vue/test-utils'
import SignupCard from '../SignupCard.vue'

describe('SignupCard', () => {
  beforeEach(() => {
    vi.stubGlobal('fetch', vi.fn(async () => new Response(null, { status: 201 })))
  })

  it('shows no errors until a field is touched', async () => {
    const wrapper = mount(SignupCard)
    await wrapper.find('#su-email').setValue('not-an-email')
    expect(wrapper.find('#su-email-error').text()).toBe('')
    await wrapper.find('#su-email').trigger('blur')
    expect(wrapper.find('#su-email-error').text()).toBe('Enter a valid email address')
  })

  it('shows every error on submit and does not call the API', async () => {
    const wrapper = mount(SignupCard)
    await wrapper.find('form').trigger('submit')
    expect(wrapper.findAll('.error').map((e) => e.text())).toEqual([
      'This field is required',
      'This field is required',
      'Please repeat the password',
    ])
    expect(fetch).not.toHaveBeenCalled()
  })

  it('validates the confirmation against the password', async () => {
    const wrapper = mount(SignupCard)
    await wrapper.find('#su-password').setValue('correct horse')
    await wrapper.find('#su-confirm').setValue('correct hose')
    await wrapper.find('#su-confirm').trigger('blur')
    expect(wrapper.find('#su-confirm-error').text()).toBe('Passwords do not match')
  })

  it('maps a 409 from the server onto the email field, then clears it on edit', async () => {
    vi.mocked(fetch).mockResolvedValueOnce(new Response(null, { status: 409 }))
    const wrapper = mount(SignupCard)
    await wrapper.find('#su-email').setValue('ada@example.com')
    await wrapper.find('#su-password').setValue('correct horse')
    await wrapper.find('#su-confirm').setValue('correct horse')
    await wrapper.find('form').trigger('submit')
    await flushPromises()
    expect(wrapper.find('#su-email-error').text()).toBe('An account with this email already exists')
    expect(wrapper.emitted('registered')).toBeUndefined()

    await wrapper.find('#su-email').setValue('ada2@example.com')
    expect(wrapper.find('#su-email-error').text()).toBe('')
  })

  it('emits registered on success', async () => {
    const wrapper = mount(SignupCard)
    await wrapper.find('#su-email').setValue('ada@example.com')
    await wrapper.find('#su-password').setValue('correct horse')
    await wrapper.find('#su-confirm').setValue('correct horse')
    await wrapper.find('form').trigger('submit')
    await flushPromises()
    expect(wrapper.emitted('registered')).toEqual([['ada@example.com']])
  })
})
```

```text
 ✓ src/components/__tests__/SignupCard.spec.ts > SignupCard > shows no errors until a field is touched 18ms
 ✓ src/components/__tests__/SignupCard.spec.ts > SignupCard > shows every error on submit and does not call the API 3ms
 ✓ src/components/__tests__/SignupCard.spec.ts > SignupCard > validates the confirmation against the password 2ms
 ✓ src/components/__tests__/SignupCard.spec.ts > SignupCard > maps a 409 from the server onto the email field, then clears it on edit 4ms
 ✓ src/components/__tests__/SignupCard.spec.ts > SignupCard > emits registered on success 2ms
 Test Files  1 passed (1)
      Tests  5 passed (5)
```

## Schemas: validating with Zod

Hand-written rules are fine for one form. When the same data shape is validated in several
places — a form, an API client, maybe a Node.js backend — a **schema library** lets you
declare it once and derive both the validator and the TypeScript type. Zod is the most
widely used (4.6.5 at the time of writing):

```bash
npm install zod
```

```ts title="src/composables/zodErrors.ts"
import { z } from 'zod'

export const signupSchema = z
  .object({
    email: z.email('Enter a valid email address'),
    password: z.string().min(8, 'Must be at least 8 characters'),
    confirm: z.string(),
  })
  .refine((f) => f.password === f.confirm, { message: 'Passwords do not match', path: ['confirm'] })

export type Signup = z.infer<typeof signupSchema>

/** First message per top-level field, in the shape our form expects. */
export function fieldErrors(input: unknown): Partial<Record<keyof Signup, string>> {
  const result = signupSchema.safeParse(input)
  if (result.success) return {}
  const out: Partial<Record<keyof Signup, string>> = {}
  for (const issue of result.error.issues) {
    const key = issue.path[0] as keyof Signup
    out[key] ??= issue.message
  }
  return out
}
```

```ts
fieldErrors({ email: 'nope', password: 'short', confirm: 'x' })
```

```text
{
  email: 'Enter a valid email address',
  password: 'Must be at least 8 characters',
  confirm: 'Passwords do not match'
}
```

With valid input it returned `{}`. You could plug `fieldErrors` into `useForm` as a
single rule source — replace the rule loop with a call to it — and `z.infer` gives you the
form's type for free. Note Zod 4's top-level `z.email()`; code written for Zod 3 used
`z.string().email()`, which Zod 4 still accepts but marks as deprecated.

## Form libraries

For large or highly dynamic forms (field arrays, wizards, conditional sections), a form
library saves a lot of code. **VeeValidate** (4.15 at the time of writing) is the most
established Vue option: `useForm`, `useField` and `defineField` composables with
touched/dirty tracking, field arrays, and adapters for Zod and other schema libraries.
FormKit is another option that also generates inputs. Because you've now built the core
yourself, their APIs should read as familiar ideas: values, errors, touched, submit
handling. We didn't install and test either library for this lesson, so check their docs
for current APIs.

## How It Actually Works

Follow what happens when the user types a character into the confirm field:

1. `v-model` writes `values.confirm`. `values` is a reactive proxy, so the `set` trap
   triggers every effect that read `values.confirm`.
2. The `errors` computed read `values.confirm` (through `rule(values[key], values)`) during
   its last evaluation, so it's marked dirty. Nothing is recalculated yet — computeds are
   lazy.
3. The component re-renders because its template read `visibleError('confirm')`, which
   read `errors.value` (only if the field was touched) — so the render effect depends on the
   computed. During render, reading `errors.value` recomputes it once, and every
   `visibleError` call in the template shares that single result.
4. If the field hasn't been touched, `visibleError` returned early without reading
   `errors.value` at all — so before the first blur, typing into a field doesn't even
   evaluate the rules for rendering purposes. Dependency tracking follows the code path
   that actually ran.

The cross-field rule `v === f.password` reads `values.password` too, so editing the
password re-validates the confirmation — no extra wiring, because dependencies are
whatever the code read.

## Common mistakes

- **Validating on every keystroke from the first keystroke** — noisy and hostile.
- **Only validating on the client.** Anyone can bypass it; the server is the authority.
- **Disabling the submit button while the form is invalid.** Users can't discover *why*
  it's disabled, and screen reader users may not find out at all. Keep it enabled; show
  errors on submit.
- **Error text not connected to the input** — use `aria-describedby` and `aria-invalid`.
- **Not clearing server errors on edit**, so a fixed field still shows "already taken".
- **Passing the reactive form object to async code** and reading it after `await` — the
  user may have changed it. Copy it at submit time.
- **Duplicating the data shape** in a TypeScript interface, client rules and server
  validation — share a schema where you can.

## Exercise

1. Add a `username` field with rules: required, 3–20 characters, letters/numbers/underscores
   only. Write the rule functions and tests.
2. After submit with errors, move focus to the first invalid input. (Hint: after
   `await nextTick()`, `document.querySelector('[aria-invalid="true"]')`.)
3. Add an **async** availability check for `username` that runs 400 ms after typing stops
   (reuse your `useDebouncedRef` from Lesson 01) and shows "Checking…" while pending.
   Make sure a slow response for an old value can't overwrite the result for the new one.
4. Replace the three rule arrays with a Zod schema and a single call to `fieldErrors`,
   keeping all five tests passing.
