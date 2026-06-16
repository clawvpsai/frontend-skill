# Forms — React Hook Form + Zod
> **React 19.2.7 patch (June 1, 2026):** If your Server Action forms submit multiple fields or files and values are missing on the server, upgrade to React 19.2.7: `npm install react@latest react-dom@latest`. v19.2.7 fixes a regression where `FormData` entries were not being passed correctly to Server Actions.


## The Mental Model

**React Hook Form** manages form state with uncontrolled inputs — it reads values directly from the DOM, not from React state. This means:
- Fewer re-renders (only on validation errors)
- Better performance for large forms
- Native browser validation works naturally

**Zod** validates the data and infers TypeScript types — no separate type definitions needed.

**Together:** RHF handles the form, Zod validates, TypeScript types flow through automatically.

## React Hook Form 8.0 Beta — Coming Soon

React Hook Form 8.0 is in beta (8.0.0-beta.2) with one major API change:
`createForm()` replaces `useForm()` as the recommended entry point. The v7 API (`useForm`, `register`, `Controller`) still works in v8 — it is not removed — but `createForm` is the new recommended pattern for new projects.

**Key changes in RHF 8.0 beta:**

```tsx
// ❌ v7 style — still works in v8 but createForm is preferred
import { useForm } from 'react-hook-form'

function MyForm() {
  const { register, handleSubmit } = useForm<FormData>()
  // ...
}

// ✅ v8 style — createForm (recommended)
import { createForm } from 'react-hook-form'

const useMyForm = createForm({
  defaultValues: { name: '', email: '' },
  validate: zodResolver(formSchema),
})

function MyForm() {
  const { register, handleSubmit } = useMyForm()
  // ...
}
```

**Why `createForm`?** It creates a typed hook factory upfront, making TypeScript inference cleaner across deeply nested form structures. The returned hook is reusable across multiple form instances.

**Migration:** Upgrade from v7 to v8 with `npm install react-hook-form@beta`.

> ⚠️ **RHF 8.0 is beta — not yet stable for production.** The claim that `useForm` is "fully backward-compatible" is incorrect. There are 6 significant breaking changes listed below. Test in a branch before upgrading production apps.

## React Hook Form 8.0 — Breaking Changes (Beta)

RHF 8.0 beta introduces **6 breaking changes** that affect existing `useForm` users — not just new `createForm` users:

### 1. `register` — Ref Now Passed Directly

`register` now passes the actual input `ref` instead of a partial ref object. Most components work without changes, but refs tied to register may need updating:

```tsx
// Most inputs work without changes
<input {...form.register('name')} />

// If you manually handle ref: the ref is now the full input ref, not partial
// Update any ref handler that expected { name, value, checked } partial
```

### 2. `useFieldArray` — `id` Renamed to `key`

The internal render identifier changed from `id` to `key`:

```tsx
// v7
const { fields } = useFieldArray({ name: 'items' })
fields[0].id  // ← was the render key

// v8
const { fields } = useFieldArray({ name: 'items' })
fields[0].key  // ← is now the render key

// You can still store id/key as data in append() — only the render identifier changed
```

### 3. `useFieldArray` — `keyName` Prop Removed

The `keyName` option is removed. The render key is always `key`:

```tsx
// v7
const { fields } = useFieldArray({ name: 'items', keyName: 'myKey' })

// v8 — keyName removed; use 'key' to access the render identifier
const { fields } = useFieldArray({ name: 'items' })
fields[0].key
```

### 4. `<Watch>` — `names` Prop Renamed to `name`

The Watch component prop changed from `names` to `name`:

```tsx
// v7
<Watch names={['firstName', 'lastName']} />

// v8
<Watch name={['firstName', 'lastName']} />
```

### 5. `watch` Callback API Removed — Use `subscribe`

The `watch` subscription callback was removed. Use `subscribe` instead:

```tsx
// v7
watch((value, { name, type }) => console.log(value))

// v8 — use subscribe
subscribe({ formValues: true }, ({ values }) => console.log(values))
```

### 6. `setValue` No Longer Updates `useFieldArray` Fields

`setValue` no longer automatically updates field array items. Use `replace()` from `useFieldArray` instead:

```tsx
// v7
setValue('items', newItems)

// v8 — use replace from useFieldArray
const { replace } = useFieldArray({ name: 'items' })
replace(newItems)
```

### RHF 8.0 — New Features (Non-Breaking)

Beyond breaking changes, v8 adds:
- **React Compiler support** — works out of the box with the React Compiler; no extra config
- **Flat field arrays** — simpler data structures for `useFieldArray`

### RHF 8.0 Migration Checklist

Before upgrading to v8 beta:
- [ ] Audit `fields[].id` → `fields[].key` in all `useFieldArray` usages
- [ ] Remove any `keyName` options in `useFieldArray` configs
- [ ] Check `<Watch names={...}>` → `<Watch name={...}>`
- [ ] Replace `watch(callback)` with `subscribe({ formValues: true }, callback)`
- [ ] Replace `setValue('fieldArray', newArray)` with `replace()` from `useFieldArray`
- [ ] Test form submission, validation, and field array operations extensively
- [ ] Run `npm install react-hook-form@beta` in a test branch first

**Sources:**
- [React Hook Form 8.0 beta announcement](https://react-hook-form.com/)
- [Migrate V7 to V8 guide](https://react-hook-form.com/migrate-v7-to-v8)
- [createForm RFC discussion](https://github.com/react-hook-form/react-hook-form/discussions)

## React Hook Form 7.79.0 (June 13, 2026) — `useFieldArray` `disabled` Option

React Hook Form 7.79.0 was released on **June 13, 2026** — the last stable v7 release before v8 ships (v8 is still in beta as `8.0.0-beta.2`). The headline change is a new `disabled` option on `useFieldArray` for **conditionally enabling entire field arrays** — useful for discriminated-union form shapes where one branch of a union has a list and another doesn't.

### The New `disabled` Prop

```tsx
import { useForm, useFieldArray } from 'react-hook-form'

type FormValues =
  | { type: 'simple'; name: string }
  | { type: 'items'; items: { name: string; qty: number }[] }

function DynamicForm() {
  const { control, watch, register } = useForm<FormValues>({
    defaultValues: { type: 'simple', name: '' },
  })
  const type = watch('type')

  // The `disabled` prop turns the entire field array off in one prop:
  //  - `fields` becomes `[]` so iteration is a no-op
  //  - All mutation methods (append, prepend, insert, remove, swap, move, update, replace) become no-ops
  //  - The array is not registered with the form (no validation, no submission value)
  const { fields, append, remove } = useFieldArray({
    control,
    name: 'items',
    disabled: type !== 'items', // ← NEW in 7.79.0
  })

  return (
    <>
      <select {...register('type')}>
        <option value="simple">Simple</option>
        <option value="items">Items</option>
      </select>

      {type === 'items' && (
        <>
          {fields.map((field, i) => (
            <div key={field.id}>
              <input {...register(`items.${i}.name` as const)} />
              <input type="number" {...register(`items.${i}.qty` as const)} />
              <button type="button" onClick={() => remove(i)}>×</button>
            </div>
          ))}
          <button type="button" onClick={() => append({ name: '', qty: 1 })}>
            Add item
          </button>
        </>
      )}
    </>
  )
}
```

**Before 7.79.0** you had to conditionally render the `useFieldArray` hook itself (only call it when `type === 'items'`), or wrap the array in a parent field path. That worked but lost the field IDs and re-mounted the whole array on toggle. The `disabled` prop keeps the hook mounted, preserves field identity, and lets the same component handle both branches of the union.

### Other Fixes in 7.79.0

- **`<Controller>` `onChange` promise** — `onChange` callbacks that return a `Promise` are no longer treated as sync (the return type is now correctly widened).
- **`deepEqual` false positives** — fields that share a common object reference no longer incorrectly appear "changed" when comparing default values against current values.
- **`shouldUseNativeValidation` for radio groups** — radio groups with native validation now behave natively (previously the browser tooltip fired on every field, not just the first invalid one).
- **`createFormControl` + React Fast Refresh** — the form state is no longer wiped when a file containing `createFormControl` is hot-reloaded.
- **StrictMode double-mount** — field values are preserved across the unmount/remount cycle in React 18 StrictMode (previously values were lost).
- **`formState.errors` reactivity with React Compiler** — error changes now correctly trigger re-renders in child components when the React Compiler is enabled (fixes the issue where errors updated but the UI didn't reflect them).

**Sources:**
- [React Hook Form 7.79.0 release notes](https://github.com/react-hook-form/react-hook-form/releases/tag/v7.79.0)
- [useFieldArray API — `disabled` prop](https://react-hook-form.com/docs/usefieldarray#props)

## Basic Setup

```bash
npm install react-hook-form zod @hookform/resolvers
```

```tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const formSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Invalid email address'),
  age: z.coerce.number().int().positive().min(18),
})

type FormData = z.infer<typeof formSchema>

export function MyForm() {
  const form = useForm<FormData>({
    resolver: zodResolver(formSchema),
    defaultValues: { name: '', email: '', age: 18 },
  })

  function onSubmit(values: FormData) {
    console.log(values)
  }

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      <input {...form.register('name')} />
      {form.formState.errors.name && <p>{form.formState.errors.name.message}</p>}
      <button type="submit">Submit</button>
    </form>
  )
}
```

## React 19 Native Forms with Server Actions

React 19 introduces native `<form>` support for Server Actions — forms work even before JavaScript loads (progressive enhancement). This is the recommended approach for forms that submit to Server Actions.

### Server Action (in `app/actions.ts`)

```tsx
'use server'

import { z } from 'zod'
import { revalidatePath } from 'next/cache'
import { redirect } from 'next/navigation'

const ContactSchema = z.object({
  name: z.string().min(1, 'Name is required'),
  email: z.string().email('Invalid email address'),
  message: z.string().min(10, 'Message must be at least 10 characters'),
})

// Return type: null on success, Record<string, string[]> on validation error
export async function submitContact(prevState: unknown, formData: FormData) {
  const parsed = ContactSchema.safeParse(Object.fromEntries(formData))
  
  if (!parsed.success) {
    // Return field errors for useActionState to display
    return { errors: parsed.error.flatten().fieldErrors }
  }
  
  // await sendEmail(parsed.data)
  revalidatePath('/contact')
  redirect('/contact/success')
}
```

### Form Component with `useActionState` (from `react`, not `react-dom`)

```tsx
// components/contact-form.tsx
'use client'

import { useActionState } from 'react'  // React 19: from 'react'
import { submitContact } from '@/app/actions'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Textarea } from '@/components/ui/textarea'

// Initial state for the form
const initialState = { errors: {} as Record<string, string[]> }

export function ContactForm() {
  const [state, formAction, isPending] = useActionState(submitContact, initialState)

  return (
    <form action={formAction} className="space-y-4">
      <div>
        <Input name="name" placeholder="Your name" />
        {state.errors?.name && (
          <p className="text-sm text-destructive">{state.errors.name[0]}</p>
        )}
      </div>
      
      <div>
        <Input name="email" type="email" placeholder="your@email.com" />
        {state.errors?.email && (
          <p className="text-sm text-destructive">{state.errors.email[0]}</p>
        )}
      </div>
      
      <div>
        <Textarea name="message" placeholder="Your message..." />
        {state.errors?.message && (
          <p className="text-sm text-destructive">{state.errors.message[0]}</p>
        )}
      </div>
      
      <Button type="submit" disabled={isPending}>
        {isPending ? 'Sending...' : 'Send Message'}
      </Button>
    </form>
  )
}
```

**Why `useActionState` from `react`?**
- In React 19, `useActionState` (formerly `useFormState` in canary) is exported from `react`, not `react-dom`
- `useFormStatus` is also from `react` in React 19 (not `react-dom`)
- Both work with native `<form action={...}>` elements

**Progressive enhancement:** Because this uses `action={formAction}` on a `<form>` element (not `onSubmit`), the form works without JavaScript — the Server Action handles the submission directly.

## Zod Schema Patterns

### Zod 4 — What's New

Zod 4 is a major release with significant performance improvements (14x faster string parsing, 7x faster array parsing, 2.3x smaller bundle) and some breaking changes. The key migration items:

```ts
// ✅ Still works — z.infer<typeof schema> is still supported (aliased to z.output)
type User = z.infer<typeof UserSchema>

// ✅ New in Zod 4 — z.file() for file validation (replaces z.instanceof(File))
const avatarSchema = z.file({
  accept: ['image/jpeg', 'image/png'],
  maxSize: 5 * 1024 * 1024,  // 5MB
})

// ✅ New in Zod 4 — z.templateLiteral() for template literal types
const slugSchema = z.templateLiteral({ pattern: /^[a-z0-9-]+$/ })

// ✅ New in Zod 4 — strict/loose object modes
const strictSchema = z.object({
  name: z.string(),
  age: z.number(),
}).strict()  // errors on extra keys

const looseSchema = z.object({
  name: z.string(),
}).loose()   // allows extra keys, strips them in output
```

**Zod 4 breaking changes to watch:**
- `z.instanceof(File)` and `z.instanceof(FileList)` are replaced by `z.file()`
- String validators `.email()` and `.url()` are stricter by default
- `z.union()` is deprecated in favor of `z.discriminatedUnion()` for unions with a common key
- Some internal type inference behavior changed — run `npx tsc --noEmit` after upgrading

### String Validation

```ts
z.string()
  .min(1, 'Required')
  .max(100)
  .email()
  .url()
  .regex(/^[a-z]+$/)
  .trim()          // always trim whitespace
  .toLowerCase()   // normalize
```

### Number Validation

```ts
z.number()
  .int()
  .positive()
  .min(0)
  .max(100)
  .finite()

// Coerce from string input
z.coerce.number()   // "42" → 42
```

### Object Schemas

```ts
const AddressSchema = z.object({
  street: z.string().min(1),
  city: z.string().min(1),
  zip: z.string().regex(/^\d{5}(-\d{4})?$/),
})

const UserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  address: AddressSchema,
  tags: z.array(z.string()).nonempty().max(5),
  role: z.enum(['admin', 'user', 'guest']),
  active: z.boolean().default(true),
})
```

### Optional and Nullable

```ts
z.string().optional()     // string | undefined
z.string().nullable()    // string | null
z.string().nullish()     // string | null | undefined
```

### Discriminated Unions

```ts
const NotificationSchema = z.discriminatedUnion('type', [
  z.object({ type: z.literal('email'), email: z.string().email() }),
  z.object({ type: z.literal('sms'), phone: z.string() }),
  z.object({ type: z.literal('push'), token: z.string() }),
])

// TypeScript infers the exact shape based on 'type' field
```

## Integrating with shadcn/ui Form

```tsx
// components/my-form.tsx
'use client'

import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { Button } from '@/components/ui/button'
import {
  Form, FormControl, FormDescription, FormField, FormItem, FormLabel, FormMessage,
} from '@/components/ui/form'
import { Input } from '@/components/ui/input'

const formSchema = z.object({
  username: z.string()
    .min(2, 'Username must be at least 2 characters')
    .max(20)
    .regex(/^[a-zA-Z0-9_]+$/, 'Only letters, numbers, and underscores'),
  email: z.string().email('Invalid email'),
  bio: z.string().max(160).optional(),
})

export function ProfileForm() {
  const form = useForm<z.infer<typeof formSchema>>({
    resolver: zodResolver(formSchema),
    defaultValues: { username: '', email: '', bio: '' },
  })

  async function onSubmit(values: z.infer<typeof formSchema>) {
    const res = await fetch('/api/profile', {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(values),
    })
    if (!res.ok) {
      const error = await res.json()
      form.setError('root', { message: error.message })
    }
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-8">
        <FormField
          control={form.control}
          name="username"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Username</Form.Label>
              <FormControl><Input {...field} /></FormControl>
              <FormDescription>Your unique identifier.</FormDescription>
              <FormMessage />
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</Form.Label>
              <FormControl><Input type="email" {...field} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        {form.formState.errors.root && (
          <p className="text-destructive">{form.formState.errors.root.message}</p>
        )}
        <Button type="submit" disabled={form.formState.isSubmitting}>
          {form.formState.isSubmitting ? 'Saving...' : 'Save'}
        </Button>
      </form>
    </Form>
  )
}
```

## Multi-Step Forms (Wizard)

```tsx
// Use a step key in the form to track progress
const wizardSchema = z.object({
  step1: z.object({
    email: z.string().email(),
  }),
  step2: z.object({
    name: z.string().min(1),
  }),
})

type WizardData = z.infer<typeof wizardSchema>

export function WizardForm() {
  const [step, setStep] = useState(1)
  const form = useForm<WizardData>({
    resolver: zodResolver(wizardSchema),
    mode: 'onChange',
  })

  const step1Valid = form.trigger('step1')
  const step2Valid = form.trigger('step2')

  async function nextStep() {
    const valid = step === 1 ? await step1Valid : await step2Valid
    if (valid) setStep(s => s + 1)
  }

  return (
    <div>
      {step === 1 && <Step1 form={form} />}
      {step === 2 && <Step2 form={form} />}
      <button onClick={nextStep}>
        {step < 2 ? 'Next' : 'Submit'}
      </button>
    </div>
  )
}
```

## File Upload Forms

```tsx
// Zod 4 — use z.file() for file validation
const uploadSchema = z.object({
  file: z.file({
    accept: ['image/jpeg', 'image/png', 'image/webp'],
    maxSize: 5 * 1024 * 1024,  // 5MB
  }),
  description: z.string().optional(),
})

const form = useForm<z.infer<typeof uploadSchema>>({
  resolver: zodResolver(uploadSchema),
})

async function onSubmit(values: z.infer<typeof uploadSchema>) {
  const file = values.file  // Zod 4: File object directly
  const formData = new FormData()
  formData.append('file', file)
  
  await fetch('/api/upload', { method: 'POST', body: formData })
}
```

```tsx
// Template ref for file input
<input 
  type="file" 
  accept="image/*"
  onChange={(e) => {
    const file = e.target.files?.[0]
    if (file) form.setValue('file', file)
  }}
/>
```

## Form Status with `useFormStatus`

```tsx
// components/submit-button.tsx
'use client'
import { useFormStatus } from 'react' // React 19: from 'react', not react-dom
import { Button } from '@/components/ui/button'
import { Loader2 } from 'lucide-react'

export function SubmitButton() {
  const { pending } = useFormStatus()
  return (
    <Button type="submit" disabled={pending}>
      {pending && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
      Submit
    </Button>
  )
}
```

```tsx
// Parent form using Server Action — useFormStatus reads this form's state
<form action={serverAction}>
  <input name="email" />
  <SubmitButton />
</form>
```

## Validation Triggers

```tsx
const form = useForm({
  mode: 'onSubmit',       // validate on submit only (default)
  // mode: 'onBlur',      // validate on field blur
  // mode: 'onChange',    // validate on every change (expensive)
  // mode: 'onTouched',   // validate after blur, then on change
  reValidateMode: 'onChange', // re-validate after submit error
})
```

## Common Mistakes

- **Using `any` for form data** — always use Zod `z.infer<typeof schema>`
- **Not calling `form.reset()` after successful submission** — prevents stale state
- **Setting initial values incorrectly** — use `defaultValues`, not `value` prop on fields
- **Missing `type="submit"` on submit button** — triggers form's `onSubmit`
- **Not handling server-side validation errors** — map API errors to form errors with `form.setError()`
- **`useActionState` from `react-dom`** — in React 19, import from `react`, not `react-dom`
- **Using `onSubmit` with Server Actions** — prefer native `action={serverAction}` for progressive enhancement
- **Zod 4: using `z.instanceof(File)`** — migrate to `z.file()` which has better types and size validation
- **Zod 4: not running type check after upgrade** — `npx tsc --noEmit` to catch type inference changes

- **RHF 8: test breaking changes before upgrading** — v8 beta is not production-stable; the `useForm` API has breaking changes including `id`→`key` rename, `keyName` removal, `names`→`name` in Watch, `watch` callback→`subscribe`, and `setValue` no longer updating field arrays
