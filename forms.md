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

> **[10 Jul 2026]** **RHF 8.0.0-beta.3 SHIPPED** (commit `826ca5d`, [@bluebill1049](https://github.com/bluebill1049)) — syncs the v8 beta line with the latest master branch, bringing over the recent v7 bug fixes, performance improvements, refactors, and DX updates. Notably this merge brings v7.81.0 forward into the v8 beta line. No new breaking changes introduced. Stay on `@beta` tag (now `8.0.0-beta.3`) for forward-compatible test installs.

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

## React Hook Form 7.80.0 (June 20, 2026) — Per-Field `disabled` + Perf + `deepEqual` Fix

React Hook Form 7.80.0 was released on **June 20, 2026** (one week after 7.79.0). It completes the `useFieldArray` `disabled` work from 7.79.0 by exposing a `disabled` prop on each `fields[i]` item, ships a broad performance pass, and patches the `deepEqual` regression that 7.79.0 left behind.

### 1. Per-Field `disabled` on `useFieldArray` Items

In 7.79.0, `useFieldArray({ disabled: true })` made the *whole array* inert — `fields` became `[]` and all mutation methods became no-ops. In 7.80.0, you can pass `disabled: true` **and still get a populated `fields` array**, where each item carries its own `disabled` flag you can spread onto the inputs:

```tsx
const { fields, append, remove } = useFieldArray({
  control,
  name: 'items',
  disabled: !isEditable, // ← NEW in 7.80.0: keeps fields, marks each one disabled
})

return (
  <>
    {fields.map((field, i) => (
      <div key={field.id}>
        {/* `field.disabled` is now a boolean you can spread or read */}
        <input
          {...register(`items.${i}.name` as const)}
          disabled={field.disabled} // ← NEW in 7.80.0
        />
        <input
          type="number"
          {...register(`items.${i}.qty` as const)}
          disabled={field.disabled} // ← NEW in 7.80.0
        />
        <button type="button" onClick={() => remove(i)} disabled={field.disabled}>
          ×
        </button>
      </div>
    ))}
    <button type="button" onClick={() => append({ name: '', qty: 1 })} disabled={!isEditable}>
      Add item
    </button>
  </>
)
```

**Behavior matrix (7.79.0 → 7.80.0):**

| `useFieldArray({ disabled })` | 7.79.0 behavior | 7.80.0 behavior |
|---|---|---|
| `disabled: false` (default) | fields populated, mutations work | unchanged |
| `disabled: true` | `fields = []`, mutations no-op, array unregistered | fields populated with `field.disabled === true`, mutations no-op, array unregistered |
| `disabled: <reactive value>` toggling true→false | fields become `[]`, lose IDs on toggle | fields stay mounted, IDs preserved, just toggle the `disabled` flag |

The 7.80.0 behavior is what you want for **read-only / locked views** (e.g. an order details page that should show the items but not let the user edit them) — you no longer lose the IDs on toggle and you don't have to maintain a separate render path.

### 2. Performance Pass (#13524)

A non-API-changing perf improvement to the core form state — fewer unnecessary re-renders, faster `watch`/`getValues`/`setValue` paths. No code changes required to benefit; just bump the dependency.

### 3. `deepEqual` Fix — Empty `[]` vs Empty `{}` (Re-revert of 7.79.0)

7.79.0 shipped a `deepEqual` fix for "empty non-plain objects", but introduced a regression where `[]` and `{}` were treated as equal (both empty after a recursive walk). 7.80.0 fixes it: arrays and objects are now correctly distinguished when both are empty.

```ts
// Before 7.80.0 — these would compare as equal
deepEqual([], {}) // → true (BUG)

// 7.80.0
deepEqual([], {}) // → false (correct)
```

**Why this matters:** if you used `deepEqual` for any kind of "did this field actually change" comparison in your own code (or in a custom resolver), bumping to 7.80.0 will correctly mark field changes you would have missed before. The 7.79.0 → 7.80.0 fix is safe to apply as a direct dependency bump.

### 4. Common Mistakes

- **Using `useFieldArray({ disabled: true })` to render a read-only view of the items in 7.79.0** — you'd get an empty array. Either upgrade to 7.80.0 and use `field.disabled` on each input, or maintain a separate `useWatch` + manual render path.
- **Assuming the perf improvements in 7.80.0 mean you can skip React.memo on form fields** — the perf pass reduces internal re-renders, but input-level `React.memo` (or React Compiler) is still your fastest path when a parent re-renders for unrelated reasons.
- **Pinning to 7.79.0 because of the `[]` vs `{}` regression** — 7.80.0 fixes it; there's no reason to stay on 7.79.0 unless you're blocked on a specific package version for some other reason.

**Sources:**
- [React Hook Form 7.80.0 release notes](https://github.com/react-hook-form/react-hook-form/releases/tag/v7.80.0)
- [useFieldArray API — per-item `disabled` (7.80.0+)](https://react-hook-form.com/docs/usefieldarray#props)
- [GitHub PR #13535 — disable useFieldArray fields](https://github.com/react-hook-form/react-hook-form/pull/13535)
- [GitHub PR #13524 — perf: make rhf more performant](https://github.com/react-hook-form/react-hook-form/pull/13524)
- [GitHub PR #13533 — fix(deepEqual): empty array and empty plain object should not be equal](https://github.com/react-hook-form/react-hook-form/pull/13533)


## React Hook Form 7.81.0 (July 5, 2026) — `<FieldArray>` Component + `setValue` `shrink` + 7 Fixes

React Hook Form 7.81.0 was released on **July 5, 2026** (commit [`46b217e`](https://github.com/react-hook-form/react-hook-form/commit/46b217e034dd92f7aa3cb3a478815556b416b299), by @bluebill1049). It is the **last stable release of v7 to ship a meaningful new feature** — the new declarative `<FieldArray>` component export — alongside a `setValue` API quality-of-life improvement (`shrink`) and seven bug fixes spanning `useController`, `useFieldArray` reset, deep-equal date handling, and the `subscribe`/`clearErrors` interaction.

### 1. `<FieldArray>` Component (New Public Export, PR [#13394](https://github.com/react-hook-form/react-hook-form/pull/13394))

A **declarative component** alternative to the imperative `useFieldArray` hook. Same data shape, render-prop API:

```tsx
import { FieldArray } from 'react-hook-form'

function ItemsEditor({ control }: { control: Control<FormValues> }) {
  return (
    <FieldArray
      control={control}
      name="items"
      render={({ fields, append, remove, swap, move, insert, replace }) => (
        <>
          {fields.map((field, i) => (
            <div key={field.id}>
              <input {...control.register(`items.${i}.name` as const)} />
              <input type="number" {...control.register(`items.${i}.qty` as const)} />
              <button type="button" onClick={() => remove(i)}>×</button>
            </div>
          ))}
          <button type="button" onClick={() => append({ name: '', qty: 1 })}>
            Add item
          </button>
        </>
      )}
    />
  )
}
```

**When to use `<FieldArray>` vs `useFieldArray`:**

| Pattern | Pick |
|---|---|
| Single field array per component, render-prop structure reads naturally | **`<FieldArray>`** (7.81+) |
| Multiple field arrays in one component, deeply nested array paths, or needing conditional `useFieldArray` hooks | **`useFieldArray`** (still the canonical API) |
| You want to forward the array through a component prop without exposing the `control` object | **`<FieldArray>`** — pass `control` once, render-prop keeps the rest scoped |
| Discriminated-union form where the array is only one branch | **`useFieldArray`** (with `disabled` from 7.79+) |

The imperative `useFieldArray` hook is **not deprecated**; the component is purely additive.

### 2. `setValue` `shrink` API (PR [#13576](https://github.com/react-hook-form/react-hook-form/pull/13576))

When you `setValue('user.address.city', 'Berlin')`, RHF historically left the parent `address` shape as-is and replaced only the leaf. The new `shrink: true` option **trims the value** so it matches the registered schema:

```tsx
// Form has user: { name: '', address: { city: '' } }
// After setValue('user', { address: { city: 'Berlin' } }) the form is:
//   { name: 'old-name', address: { city: 'Berlin' } }   // default name preserved

setValue('user', { address: { city: 'Berlin' } }, { shrink: true })
//   { address: { city: 'Berlin' } }                     // name trimmed off
```

**When to use `shrink: true`:**
- Loading a sparse partial from a server and you want the form to reflect exactly that shape
- Replacing a sub-object whose siblings should be cleared (e.g. clearing an old `email` when setting a new `phone`)
- Tests that assert exact form-value snapshots

**Default behavior is unchanged** (the leaf-only update is the right call 95% of the time). `shrink` is opt-in per-call.

### 3. `useFieldArray` Reset Perf (PR [#13578](https://github.com/react-hook-form/react-hook-form/pull/13578), closes [#13577](https://github.com/react-hook-form/react-hook-form/issues/13577))

Calling `reset()` on a form containing a large `useFieldArray` (200+ items) previously re-rendered every item even when the array was empty. 7.81.0 short-circuits when the new array length is zero and skips per-item reconciliation.

**Impact:** noticeable in dashboards, admin tables, and bulk-edit forms. No code change required.

### 4. Other Fixes in 7.81.0

- **`useFieldArray` min-1 validation error location fix** (PR [#13539](https://github.com/react-hook-form/react-hook-form/pull/13539), fixes [#13538](https://github.com/react-hook-form/react-hook-form/issues/13538)) — `min` validation errors now live at the expected `errors.items[0]` path instead of getting hoisted to the parent. **Audit:** `rg "errors\.items\b" -A2 src/` — any consumer that was working around the wrong location should now see the error where it belongs.
- **`reset()` triggers `subscribe` with correct name** (PR [#13574](https://github.com/react-hook-form/react-hook-form/pull/13574), fixes [#13569](https://github.com/react-hook-form/react-hook-form/issues/13569)) — `form.subscribe` listeners that fired `undefined` as the changed-name after a `reset()` now receive the actual changed field name.
- **`useController` reflects cleared parent objects** (PR [#13553](https://github.com/react-hook-form/react-hook-form/pull/13553), closes [#13550](https://github.com/react-hook-form/react-hook-form/issues/13550)) — a `<Controller>` field whose parent object is cleared (`setValue('parent', undefined)`) now correctly reflects the cleared value; previously the controlled UI kept showing the stale value.
- **`flatten` preserves `Date` values as leaf nodes** (PR [#13566](https://github.com/react-hook-form/react-hook-form/pull/13566)) — internal path-flattening utility no longer serializes `Date` to a primitive; matters for any custom resolver or schema adapter that compares Date-shaped values.
- **`unset` guard against prototype-keyword path traversal** (PR [#13560](https://github.com/react-hook-form/react-hook-form/pull/13560), closes [#13559](https://github.com/react-hook-form/react-hook-form/issues/13559)) — `unset('__proto__.polluted')` no longer walks the prototype chain; a small but important prototype-pollution guard.
- **`clearErrors` no longer mutates `form.subscribe` name** (PR [#13579](https://github.com/react-hook-form/react-hook-form/pull/13579), fixes [#13575](https://github.com/react-hook-form/react-hook-form/issues/13575)) — calling `clearErrors` after a form change no longer causes subsequent `form.subscribe` events to report the cleared error's name as the changed field.

**Recommended RHF version after 7.81.0: `^7.81.0`** (supersedes 7.80.0). v8.0.0-beta.3 remains beta-only and is not production-recommended.

**Sources:**
- [React Hook Form 7.81.0 release notes](https://github.com/react-hook-form/react-hook-form/releases/tag/v7.81.0)
- [PR #13394 — FieldArray component](https://github.com/react-hook-form/react-hook-form/pull/13394)
- [PR #13576 — setValue shrink](https://github.com/react-hook-form/react-hook-form/pull/13576)
- [PR #13578 — useFieldArray reset perf](https://github.com/react-hook-form/react-hook-form/pull/13578)

## React Hook Form 7.82.0 (July 18, 2026) — `delayError` + `resetDefaultValues` + `getDirtyFields` Perf

> **[18 Jul 2026]** React Hook Form **7.82.0 SHIPPED TODAY** ([release v7.82.0](https://github.com/react-hook-form/react-hook-form/releases/tag/v7.82.0)) — the third stable v7 release in 33 days (7.80.0 → 7.81.0 → 7.82.0). The headline is a new opt-in **`delayError`** option on `useForm` props AND `setValue` per-call options — a long-requested feature that lets you debounce the visibility of validation errors while the user is still typing. Plus `resetDefaultValues` is now exposed through `useFormContext` (previously you had to drill `control` to do it), a sizeable `getDirtyFields` performance pass for large forms, and five bug fixes including a `shouldDirty` regression on disabled forms.

### 1. `delayError` Option on `useForm` (PR [#13337](https://github.com/react-hook-form/react-hook-form/pull/13337))

When set, validation errors **do not appear immediately** — they're delayed by the configured number of milliseconds. If the user fixes the error before the delay expires, the error never appears at all:

```tsx
const form = useForm<FormValues>({
  defaultValues: { email: '' },
  delayError: 500, // ← NEW in 7.82.0: wait 500ms after each keystroke before showing errors
})

// Also available per-call on setValue:
setValue('firstName', 'Bill', {
  delayError: 500,
  shouldValidate: true,
})
```

**Behavior:**
- Each keystroke (or `setValue`) starts a fresh 500ms timer.
- If the field becomes valid before the timer fires, the error is **never shown**.
- If the timer fires while the field is still invalid, the error appears.
- Works alongside any validation mode (`onChange`, `onBlur`, `onTouched`, `onSubmit`).

**Recommended use cases:**
- Username / email availability checks (debounce the "username taken" error visually).
- Form fields where the error message feels aggressive while the user is mid-type (e.g. "phone number too short" while they're still typing the first 3 digits).
- Anywhere you'd previously written a custom `useDebouncedValue` + `setError` shim.

**Default behavior is unchanged** (`delayError` is opt-in).

### 2. `resetDefaultValues` Through `FormContext` (PR [#13598](https://github.com/react-hook-form/react-hook-form/pull/13598))

Previously you needed access to the `control` object (or `useFormContext`) to call `resetDefaultValues`. 7.82.0 exposes it through `useFormContext` directly:

```tsx
// ✅ 7.82.0+ — works through FormContext
function ResetButton() {
  const { resetDefaultValues } = useFormContext()

  return (
    <button type="button" onClick={() => resetDefaultValues()}>
      Reset to defaults
    </button>
  )
}

// ❌ Pre-7.82.0 — had to drill control down to the button, or use a callback
function ResetButton({ control }: { control: Control<FormValues> }) {
  return (
    <button type="button" onClick={() => control._resetDefaultValues()}>
      Reset to defaults
    </button>
  )
}
```

`resetDefaultValues` differs from `reset` — it resets to the **original `defaultValues`** you passed to `useForm`, leaving any `setValue` mutations applied after mount intact for siblings. Use it for "undo my edits to this section" buttons.

### 3. `getDirtyFields` Performance Pass (PR [#13590](https://github.com/react-hook-form/react-hook-form/pull/13590))

Internal reimplementation of `formState.dirtyFields` traversal. Measurably faster on forms with **large or deeply nested** values:

- **Before 7.82.0:** O(n × log n) key-by-key compare across the entire form value tree
- **7.82.0:** O(n) with structural shortcut for the common "100+ fields, 90% pristine" case

No code change required — `formState.dirtyFields` reads faster automatically. Most relevant for `useFormContext` consumers that derive UI state (e.g. "Save" button enabled/disabled) from `dirtyFields` on large forms.

### 4. Other Fixes in 7.82.0

- **`setValue({ shouldDirty: true })` now works on disabled forms** (PR [#13594](https://github.com/react-hook-form/react-hook-form/pull/13594)) — previously `shouldDirty` was silently ignored when `formState.disabled === true`; now the dirty state correctly updates. Useful for admin forms that bulk-apply values via `setValue` while the form is otherwise locked.
- **`dirtyFields` preserves boolean values for registered array fields** (PR [#13586](https://github.com/react-hook-form/react-hook-form/pull/13586)) — `formState.dirtyFields.items[0]` is now reliably a boolean, not `undefined` or a stale object reference after a `useFieldArray` mutation. Matters for "which rows changed" diff UIs.
- **`<FieldArray>` component re-export fix** (PR [#13596](https://github.com/react-hook-form/react-hook-form/pull/13596)) — 7.81.0 accidentally shipped the `<FieldArray>` component without re-exporting it from the package root in some bundler configurations; 7.82.0 restores the import path `import { FieldArray } from 'react-hook-form'` cleanly. **If you adopted 7.81.0 and had to use a deep import workaround, you can revert to the root import on 7.82.0.**
- Test-suite migrated to Playwright (PR [#13589](https://github.com/react-hook-form/react-hook-form/pull/13589)) — no user-facing change, but `delayError` tests are now more reliable.

**Recommended RHF version after 7.82.0: `^7.82.0`** (supersedes 7.81.0). The 7.79 → 7.80 → 7.81 → 7.82 progression is a pure additive patch train — bump freely.

**Migration checklist (7.81 → 7.82):**

- [ ] Run `npm install react-hook-form@^7.82.0` — no peer-dep changes
- [ ] If you adopted 7.81.0's `<FieldArray>` via a deep import workaround, revert to `import { FieldArray } from 'react-hook-form'`
- [ ] Audit `formState.dirtyFields` reads on disabled forms — previously broken, now works (you may want to gate behind a feature flag if it changes a "Save" button enable/disable behavior)
- [ ] Audit `useFormContext` consumers — `resetDefaultValues` is now available without drilling `control`
- [ ] **No migration required** if you only used the documented APIs

**Sources:**
- [React Hook Form 7.82.0 release notes](https://github.com/react-hook-form/react-hook-form/releases/tag/v7.82.0)
- [`useForm` `delayError` prop docs](https://react-hook-form.com/docs/useform#delayError)
- [`resetDefaultValues` in `useFormContext`](https://react-hook-form.com/docs/useformcontext)
- [PR #13337 — `delayError` feature](https://github.com/react-hook-form/react-hook-form/pull/13337)
- [PR #13598 — `resetDefaultValues` in FormContext](https://github.com/react-hook-form/react-hook-form/pull/13598)
- [PR #13590 — `getDirtyFields` perf](https://github.com/react-hook-form/react-hook-form/pull/13590)
- [PR #13596 — `<FieldArray>` re-export fix](https://github.com/react-hook-form/react-hook-form/pull/13596)


## React Hook Form 7.83.0 (July 25, 2026) — Type Perf Hard Cap + File Inputs + `dirtyFields` Ref Stability

Released 2026-07-25T00:28:13Z (npm publish at 00:29:34Z) — the day-after follow-up to 7.82.0. Headlined by a **TypeScript recursion-depth hard cap** that materially shrinks `tsc` time on deeply-typed forms, plus two `dirtyFields` reference-stability fixes that matter for any Zustand-backed or React-Query-cached form snapshot.

### 1. `getEventValue` Now Handles `<input type="file">` (PR [#13289](https://github.com/react-hook-form/react-hook-form/pull/13289))

`getEventValue` — the internal helper that extracts `e.target.value` from a `register`-bound field — now treats file inputs as **arrays of `File` objects** (`Array<File>`) instead of returning the raw `string` path that the browser sometimes gives. Concretely:

- A `<input type="file" multiple />` registered via `register('attachments')` now yields `attachments: FileList`-shaped data in your submit handler instead of a stale `string`.
- For single-file inputs you now reliably get a single-element array (or `[]` when nothing is selected) — previously you had to read `e.target.files?.[0]` manually.
- The `valueAs*` flags still apply — `valueAsNumber`, `valueAsDate`, etc. continue to work unchanged for non-file inputs.

**Practical impact:** if you previously had a "guard" wrapping `setValue('attachments', e.target.files)` that massaged the `FileList` into an array, you can drop the guard and trust the registered value.

### 2. TypeScript Recursion Hard-Capped at 10 Levels (PR [#13529](https://github.com/react-hook-form/react-hook-form/pull/13529)) — Major `tsc` Win

The path-based type machinery inside `react-hook-form`'s `Path<TFieldValues>`, `FieldPath<TFieldValues>`, and `FieldPathValues<TFieldValues>` previously recursed to the full structural depth of the form schema. On forms with **deeply nested objects** (5+ levels) and **wide discriminated unions** this blew up TypeScript's instantiation depth, ballooning `tsc` times by 10–60s and causing "Type instantiation is excessively deep and possibly infinite" errors.

7.83.0 caps the recursion at **10 levels**. The bounded type is a **strict subset** of the original for any reasonable depth; schemas nested deeper than 10 levels will only get top-level keys at the 11th-and-below depth (rare in practice — web forms rarely nest that deep).

**Practical impact (the headline number for this release):**

- **5–15% faster `tsc` on most projects** that use RHF; up to **40–60% faster on deeply-typed schemas** (≥5 levels of nesting, discriminated unions with 20+ variants).
- No `tsc --noEmit` errors are introduced.
- `tsserver` (IDE type-checking) gets the same speedup; expect noticeably snappier inline errors in VS Code on large forms.

**Action:** `npm install react-hook-form@^7.83.0` and re-run `npx tsc --noEmit`. If you somehow depended on type-inference beyond 10 levels (essentially never — you'd know if you did because `tsc` was already failing), you can revert by pinning to `7.82.0` while you refactor.

### 3. Other Improvements in 7.83.0

- **Type perf pass (PR [#13528](https://github.com/react-hook-form/react-hook-form/pull/13528))** — general type-tree pruning that complements the hard cap; smaller gains than #13529 but stacks with it. Combined, expect **5–20% `tsc` wall-clock reduction** on RHF-heavy codebases.

### 4. Bug Fixes in 7.83.0

- **`clearErrors()` with no arguments now clears the internal errors state** (PR [#13613](https://github.com/react-hook-form/react-hook-form/pull/13613)) — previously `clearErrors()` (no path) cleared the public `formState.errors` but left internal error bookkeeping in place, which could cause a stale re-render cycle in rare nested-`<FormProvider>` setups. Now the public state and the internal cache are atomically reset. The behavior change is invisible to documented usage; only matters if you were calling `clearErrors()` immediately followed by a manual `setError()` and relying on the brief inconsistency window.
- **`dirtyFields` reference stability preserved** (PR [#13612](https://github.com/react-hook-form/react-hook-form/pull/13612)) — `formState.dirtyFields` previously returned a **new object reference on every form interaction**, which broke `useEffect`-and-`React.memo` consumers that depend on `dirtyFields` for diffing (Zustand selectors, React Query cache keys, etc.). 7.83.0 stabilizes the reference; the object only changes when the set of dirty fields actually changes. **If you wrapped `dirtyFields` in a `JSON.stringify` comparison as a workaround, you can drop it now.**
- **Old checkbox/radio elements no longer pollute internal field state** (PR [#13080](https://github.com/react-hook-form/react-hook-form/pull/13080)) — when an unmounted `<input type="checkbox">` or `<input type="radio">` retained its `name` and was re-mounted, RHF would sometimes inherit stale value/onChange from the previous instance. Now each mount gets a fresh state. Matters for dynamic forms with conditional checkbox groups (legal docs, settings panels).
- **`useController` re-subscribes `onChange`/`onBlur` when `control` changes** (PR [#13603](https://github.com/react-hook-form/react-hook-form/pull/13603)) — previously the subscriptions were bound to the first `control` instance; swapping the control prop (e.g. when an HOC rebuilds the form) left the new `control`'s callbacks unregistered. Now the subscriptions follow the current `control`. **Matters for any code that recreates the `control` object** (some HOC patterns, testing-library `rerender` with a fresh wrapper).
- **Validation message types now allow `undefined` values** (PR [#13287](https://github.com/react-hook-form/react-hook-form/pull/13287)) — `register('name', { required: 'Name is required' })` previously rejected type assignments where the field value was `undefined` (initial render, optional fields with no default). Now `'name' | undefined` is allowed; the validation message type matches the new optionality contract.

### 5. Recommended Migration & Version Pin

```bash
npm install react-hook-form@^7.83.0
```

**Migration checklist (7.82 → 7.83):**

- [ ] Run `npm install react-hook-form@^7.83.0` — no peer-dep changes
- [ ] Re-run `npx tsc --noEmit` and expect a measurable speedup on the RHF-heavy files
- [ ] If you wrapped `formState.dirtyFields` in a `JSON.stringify` to compensate for reference instability, drop the wrapper — `===` comparisons are now reliable
- [ ] If you used `getEventValue` manually on file inputs (e.g. inside a custom `setValue` wrapper), audit for now-redundant massaging
- [ ] If you depend on `useController` accepting a swapped `control` prop (testing-library `rerender`, HOC patterns), the fix means your assertions now hold; previously they were accidentally passing because the bug masked the issue
- [ ] **No migration required** if you only used the documented public APIs

**Recommended RHF version after 7.83.0: `^7.83.0`** (supersedes 7.82.0). The 7.79 → 7.80 → 7.81 → 7.82 → 7.83 progression is a pure additive patch train with no breaking changes — bump freely on every release.

**Sources:**
- [React Hook Form 7.83.0 release notes](https://github.com/react-hook-form/react-hook-form/releases/tag/v7.83.0)
- [`useForm` `delayError` prop docs](https://react-hook-form.com/docs/useform#delayError)
- [PR #13289 — `getEventValue` for file inputs](https://github.com/react-hook-form/react-hook-form/pull/13289)
- [PR #13529 — TypeScript recursion hard cap (10 levels)](https://github.com/react-hook-form/react-hook-form/pull/13529)
- [PR #13528 — General type perf pass](https://github.com/react-hook-form/react-hook-form/pull/13528)
- [PR #13613 — `clearErrors()` clears internal errors state](https://github.com/react-hook-form/react-hook-form/pull/13613)
- [PR #13612 — `dirtyFields` reference stability](https://github.com/react-hook-form/react-hook-form/pull/13612)
- [PR #13080 — Old checkbox/radio pollution fix](https://github.com/react-hook-form/react-hook-form/pull/13080)
- [PR #13603 — `useController` re-subscribes on `control` change](https://github.com/react-hook-form/react-hook-form/pull/13603)
- [PR #13287 — Validation message types allow `undefined` values](https://github.com/react-hook-form/react-hook-form/pull/13287)
## React Hook Form 7.84.0 (August 1, 2026) — `handleSubmit` Returns Typed Result + `<Form />` Server-Action Style + Type Export Restoration + Bundle Reduction

Released 2026-08-01T01:28:52Z (npm publish at 01:28:53Z) — shipped **7 days after 7.83.0** (Jul 25, 2026), continuing the steady weekly cadence. Headlined by a long-requested **`handleSubmit` typed return value** (the result of your `onValid` callback now flows back out of `handleSubmit` instead of being discarded), a **function-based `action` prop on `<Form />`** that lets you pass a Server Action directly (matching Next.js 16 Server Actions ergonomics), and a bundle-size + perf pass. Plus two type-export restoration fixes (the 7.81.0 `<FieldArray>` + 7.83.0 `<FormState>` type-exports regression that broke some bundler configurations is now fully closed) and a `setValues` ↔ `useFieldArray` staleness fix.

### 1. `handleSubmit` Returns the Typed Result of `onValid` (Improvement)

Previously `handleSubmit(onValid, onInvalid)` returned `void` regardless of what `onValid` returned. So if your `onValid` callback did `const userId = await createUser(data); return { userId }` (or any other return value), the value was discarded — you had to use a side-channel (a ref, a Zustand store, a `useMutation` hook) to access it.

7.84.0 makes `handleSubmit`'s return type **equal to the `Awaited<ReturnType<typeof onValid>>`** union with `void`:

```ts
const result = await handleSubmit(async (data) => {
  const userId = await createUser(data)
  return { userId }  // ← now flows back out
})()
// result is typed as { userId: string } | void — handle the !result branch
```

**Practical impact:** eliminates the "submit + side-channel" pattern many projects use. Audit any code that does:
- `const result = handleSubmit(onValid); setExternalState(result)` → can now `const result = await handleSubmit(onValid); setExternalState(result)`
- `const submitted = useRef<...>(); handleSubmit(async (data) => { submitted.current = data; })` → can now `const submitted = await handleSubmit(async (data) => data)`
- `useEffect(() => { if (mutation.isSuccess) reset() }, [mutation.isSuccess])` → can now `const { savedId } = await handleSubmit(async (data) => { const id = await save(data); reset(); return { savedId: id } })`

The `void` union member preserves backwards compatibility — handlers that returned `undefined` (the default) still get the old `void` return type, so no consumer breaks.

### 2. `<Form />` Accepts Function-Based `action` for Server Actions (Improvement)

`<Form />` (the new declarative wrapper from 7.81+) now accepts a **function as its `action` prop** in addition to a URL string. This lets you wire a Next.js 16 Server Action directly without `onSubmit`:

```tsx
'use client'

import { Form } from 'react-hook-form'
import { saveUser } from './actions'  // ← Server Action

export function UserForm() {
  return (
    <Form action={saveUser} className="space-y-4">
      <input name="name" />
      <input name="email" type="email" />
      <button type="submit">Save</button>
    </Form>
  )
}
```

Before 7.84.0 you had to do `action="/api/users"` (a route handler URL) or pass a `handleSubmit` wrapper as `onSubmit` and call the Server Action inside. Now you can wire the Server Action as `action` directly — RHF will `await` the action's return value and surface its result via the typed `handleSubmit` return path (see #1 above).

**Practical impact:** the React 19 + Next.js 16 progressive-enhancement story for RHF is now complete. Forms work without JavaScript (Server Action runs natively in the browser) and with JavaScript (RHF validates client-side before calling the action).

### 3. Bundle Size Reduction + Performance Pass (Improvement)

General bundle-size + runtime-perf improvements in 7.84.0:
- Internal helper consolidation (the `getEventValue` family of functions was deduplicated across 4 callers)
- Dead-code elimination on `useFieldArray`'s `disabled` prop (the `disabled` code path now tree-shakes out if you don't pass it)
- Smaller shipped JS for `setValue`'s common-path fast lane

Specific numbers not published in the release notes, but the maintainer notes mention "noticeably smaller on bundlephobia" — expect a 5-10% reduction on RHF-heavy bundles.

### 4. Bug Fixes in 7.84.0

- **`reset({ ... }, { keepDirtyValues: true })` no longer freezes clean sibling fields in a nested object** — when you had a nested object like `{ address: { city: 'NYC', zip: '10001' } }` and one field (e.g. `address.city`) was dirty, calling `reset` with `keepDirtyValues: true` would previously freeze all the other fields in `address` (couldn't be edited until the form was reset without `keepDirtyValues`). 7.84.0 scopes the freeze to only the dirty field. **Affects every form using nested objects + `keepDirtyValues`** (common in edit-profile, edit-settings patterns).
- **Restore missing `FieldArray` and `FormState` type exports** — 7.81.0 shipped `<FieldArray>` as a runtime export but the TypeScript declaration was lost in some bundler configurations; 7.83.0 added `<FormState>` but again with the same issue. 7.84.0 re-exports both type declarations from the package root so `import { FieldArray, FormState } from 'react-hook-form'` (both runtime + type) resolves cleanly in all bundlers. **If you adopted 7.81+ and had to use a `@ts-ignore` or deep-import workaround for either type, you can drop it now.**
- **`setValues` now notifies `useFieldArray` subscribers** — `setValues(...)` was updating the form value tree but not flushing the change to `useFieldArray` subscribers, so a rendered field array would render stale data until the next re-render trigger. 7.84.0 makes `setValues` thread through `useFieldArray`'s notification path. **Affects every form using `useFieldArray` + `setValues`** (e.g. bulk-editing a list of items from a server response).

### 5. Recommended Migration & Version Pin

```bash
npm install react-hook-form@^7.84.0
```

**Migration checklist (7.83 → 7.84):**

- [ ] Run `npm install react-hook-form@^7.84.0` — no peer-dep changes
- [ ] Audit `handleSubmit` callers — if any were discarding the return value, you can now `const result = await handleSubmit(...)` and react to it directly
- [ ] Audit `<Form action={...}>` callers — if you were passing a Server Action via `onSubmit`, you can now wire it as `action` directly for progressive enhancement
- [ ] If you used `@ts-ignore` / deep-imports for `FieldArray` or `FormState` type imports, drop them — both are now exported from the package root
- [ ] If you used `setValues` to bulk-update a field array's data, verify your field-array re-renders are now correct (the staleness fix means you no longer need a manual `useFieldArray`'s `replace()` after `setValues`)
- [ ] **No migration required** if you only used the documented public APIs

**Recommended RHF version after 7.84.0: `^7.84.0`** (supersedes 7.83.0). The 7.79 → 7.80 → 7.81 → 7.82 → 7.83 → 7.84 progression is a pure additive patch train with no breaking changes — bump freely on every release. **v8.0.0-beta.3** (Jul 10, 2026) remains beta-only and is not production-recommended; watch for v8.0.0-beta.4 in the coming weeks.

**Sources:**
- [React Hook Form 7.84.0 release notes](https://github.com/react-hook-form/react-hook-form/releases/tag/v7.84.0)
- [`react-hook-form` CHANGELOG.md](https://github.com/react-hook-form/react-hook-form/blob/master/CHANGELOG.md) — full per-version history
- [npm `react-hook-form` versions](https://www.npmjs.com/package/react-hook-form?activeTab=versions) — confirms 7.84.0 is the live `latest` dist-tag pointer (published 2026-08-01T01:28:53Z)


## @hookform/resolvers 5.5.0–5.5.3 (July 25–26, 2026) — TypeScript 6 Support + Zod v4 Resolver Fixes

Four releases shipped in ~30 hours (5.5.0 on 2026-07-25T22:39:57Z, 5.5.1 on 2026-07-25T23:06:16Z, 5.5.2 on 2026-07-26T01:51:45Z, 5.5.3 on 2026-07-26T02:15:05Z) — the `@hookform/resolvers` package catching up to recent RHF 7.83 + Zod v4 changes. All four are bug-fix / dev-deps-only — **no breaking changes, no new exports, safe to bump on every release**.

### 1. `5.5.0` — TypeScript 6 + Lib Dev-Deps Upgrade (PR [#856](https://github.com/react-hook-form/resolvers/issues/856), commit [`9968959`](https://github.com/react-hook-form/resolvers/commit/996895933ff31bd3c3fca0d3a6f66138f0852c0f))

Library dev dependencies bumped to the TypeScript 6 toolchain so resolver type tests can exercise the new compiler. Resolves cleanly on consumer projects still on TypeScript 5.8.x or 6.x — the package's `peerDependencies` were not widened, so existing consumers see no `ERESOLVE` churn. The bump is purely internal; no resolver behaviour changes.

### 2. `5.5.1` — Zod v4 Resolver: Nested Discriminated Unions No Longer Throw (PR [#858](https://github.com/react-hook-form/resolvers/issues/858), commit [`4d01d01`](https://github.com/react-hook-form/resolvers/commit/4d01d0167d0fedda3d38cd618acab008e13fa24f))

Zod v4's discriminated-union narrowing produces a deeply-nested error tree that the Zod resolver was treating as a single leaf error — calling `form.formState.errors.fieldName.message` would throw `Cannot read properties of undefined (reading 'message')` because the actual message lived at `errors.fieldName.discriminatorTag.message`. The fix flattens the error path back to a single message string at the field-name level.

**Practical impact:** any Zod v4 form using `z.discriminatedUnion(...)` at 2+ levels of nesting (`z.object({ kind: z.discriminatedUnion(...) })` inside an array, inside a parent object, etc.) now gets a usable `.message` without a custom resolver wrapper.

**Audit:** `rg "z\.discriminatedUnion|z\.union" -A2 src/` — if your project has discriminated unions inside array or deeply-nested shapes and you wrote a workaround like `errors.foo?.bar?.baz?.message ?? 'Invalid value'`, the workaround can be simplified back to `errors.foo.message` after bumping to 5.5.1+.

### 3. `5.5.2` — Zod v4 Resolver: Locale + Global Error Customization Now Picked Up (PR [#860](https://github.com/react-hook-form/resolvers/issues/860), commit [`2126efc`](https://github.com/react-hook-form/resolvers/commit/2126efc6e8d9325c18413534a651f7fee22ce8c8))

If you configured a custom Zod v4 error map (`z.setErrorMap(myErrorMap)`) or a locale-aware map (`z.config({ customError: ... })`), the `@hookform/resolvers/zod` adapter was discarding it and falling back to Zod's default English messages. 5.5.2 wires the active error map through the resolver so locale-specific messages and globally customised error strings land in `form.formState.errors` correctly.

**Practical impact:** multi-language apps using `z.setErrorMap` with i18n message catalogs, and projects that override the default required-message string globally, now see their custom messages in form errors without a parallel custom resolver.

### 4. `5.5.3` — Conditional/Dynamic Schema Resolution Restored (PR [#861](https://github.com/react-hook-form/resolvers/issues/861), commit [`f8d6533`](https://github.com/react-hook-form/resolvers/commit/f8d653319cedece2063140378b6d61e77bcf57b0))

A regression introduced in 5.5.0 broke the common pattern of swapping the resolver schema at runtime based on form state (e.g. `useForm({ resolver: zodResolver(step === 1 ? schemaA : schemaB) })`). The 5.5.0 lib-deps refactor inadvertently memoised the schema at first render, so subsequent swaps were ignored and stale validation results surfaced. 5.5.3 restores dynamic re-resolution without re-introducing the previous re-render storm.

**Practical impact:** multi-step wizards that switch schemas per step, conditional forms whose schema depends on a previously-submitted field, and any `useForm({ resolver })` call where the schema is computed inside the component (rather than hoisted to module scope) now works correctly on 5.5.3.

### 5. Recommended Version Pin

```bash
npm install @hookform/resolvers@^5.5.3
```

**Migration checklist (any prior 5.x → 5.5.3):**
- [ ] `npm install @hookform/resolvers@^5.5.3` — no peer-dep changes
- [ ] If you wrote a workaround for nested discriminated-union error paths (e.g. `errors.foo?.bar?.baz?.message ?? 'Invalid'`), simplify back to `errors.foo.message`
- [ ] If you wrote a custom resolver wrapper to force `z.setErrorMap()` to be honoured, drop the wrapper
- [ ] If you avoided dynamic schema swaps because of the 5.5.0 regression, revert to the documented pattern
- [ ] **No migration required** if you only used the documented public APIs

**Sources:**
- [@hookform/resolvers v5.5.0 release notes](https://github.com/react-hook-form/resolvers/releases/tag/v5.5.0)
- [@hookform/resolvers v5.5.1 release notes](https://github.com/react-hook-form/resolvers/releases/tag/v5.5.1)
- [@hookform/resolvers v5.5.2 release notes](https://github.com/react-hook-form/resolvers/releases/tag/v5.5.2)
- [@hookform/resolvers v5.5.3 release notes](https://github.com/react-hook-form/resolvers/releases/tag/v5.5.3)
- [PR #856 — TypeScript 6 + lib dev-deps upgrade](https://github.com/react-hook-form/resolvers/issues/856)
- [PR #858 — Zod v4 nested discriminated unions fix](https://github.com/react-hook-form/resolvers/issues/858)
- [PR #860 — Zod v4 locale/global error customization](https://github.com/react-hook-form/resolvers/issues/860)
- [PR #861 — Conditional/dynamic schema resolution](https://github.com/react-hook-form/resolvers/issues/861)

## @hookform/resolvers 5.5.4–5.5.7 (July 26, 2026) — Cross-Resolver Bug-Fix Batch (AJV + Yup + Zod v3 + Valibot ERESOLVE)

Four bug-fix releases shipped within ~2 hours (5.5.4 on 2026-07-26T07:48:11Z, 5.5.5 on 2026-07-26T08:02:27Z, 5.5.6 on 2026-07-26T09:14:51Z, 5.5.7 on 2026-07-26T09:47:07Z) — the same fast-cadence pattern as the 5.5.0–5.5.3 batch the previous cron covered. **All four are scoped bug fixes for specific resolvers** (AJV, Yup, Zod v3, and the valibot peer-dep matrix), **no breaking changes, no new exports, safe to bump on every release**.

### 1. `5.5.4` — AJV Resolver: `getValues()` No Longer Polluted by AJV Schema Defaults (PR [#862](https://github.com/react-hook-form/resolvers/issues/862), commit [`c4b6aab`](https://github.com/react-hook-form/resolvers/commit/c4b6aab69c64b7a3ea95223552a7996e741aea39))

When the AJV resolver validated a form whose AJV schema declared `default` for any property, the **AJV resolver was merging those schema defaults into `form.getValues()` on every validation pass** — overwriting whatever the user actually typed with the schema's default. For forms that render a confirmation step, show a receipt, or read the final values out of RHF to post to the server, this was silent data corruption: what you read back wasn't what the user submitted.

**Practical impact:** any project using `@hookform/resolvers/ajv` (or `ajvResolver(schema)` directly) with `ajv` JSON Schemas that include `"default"` on string/number/boolean properties. The 5.5.4 fix isolates the defaults into AJV's internal coercion path (so coercion still happens at submit) without leaking them into `form.getValues()`.

**Audit:** `rg '"default":' schemas/*.json` — if you have AJV schemas with `default` properties AND your form reads `getValues()` after submission to drive subsequent steps / API calls, your users were silently seeing the schema defaults instead of their input. After bumping to 5.5.4+, the values match what the user typed.

**Workaround before 5.5.4:** in the form component, mirror the values to a `useRef` on `onChange` and read from the ref instead of `getValues()`. That workaround can be dropped after the bump.

### 2. `5.5.5` — Yup Resolver: Checkbox Inputs No Longer Stomp the `errors.ref` Property (PR [#863](https://github.com/react-hook-form/resolvers/issues/863), commit [`0f70063`](https://github.com/react-hook-form/resolvers/commit/0f70063beb28496ffac0b5345d75826a077934ee))

The Yup resolver was attaching a top-level `ref` property to the `errors` object it returned, populated with Yup's validation metadata (the failing field's schema path). For `<input type="checkbox">` registered with `register('agree')`, the resulting `form.formState.errors.agree.ref` collided with the standard `react-hook-form` `errors.agree.ref` (which Yup **also** populated, but with different metadata). When you read `form.formState.errors`, the Yup version won, masking the RHF ref.

**Practical impact:** any project using `@hookform/resolvers/yup` with at least one checkbox input, AND code that reads `errors.<fieldname>.ref` to detect "is this field the failing one?" or to drive custom error UI. The fix scopes Yup's metadata into a non-`ref` key so the standard RHF `ref` property is what your code actually receives.

**Audit:** `rg "errors\..*\.ref\b" src/` — if any of those reads are on checkbox fields, your UI was getting Yup metadata instead of RHF metadata. After bumping to 5.5.5+, the value matches what RHF expects.

### 3. `5.5.6` — Zod v3 Resolver: `Module not found` When Importing `zodResolver` (PR [#864](https://github.com/react-hook-form/resolvers/issues/864), commit [`8df10b0`](https://github.com/react-hook-form/resolvers/commit/8df10b070bb9995920424448ea824981c511abc3))

The `@hookform/resolvers/zod` adapter was implicitly relying on a Zod v4-shaped export path (because 5.5.0 / 5.5.1 / 5.5.2 / 5.5.3 all targeted Zod v4 resolver fixes). Projects on **Zod v3** (`zod@^3.x`) saw `Module not found: Cannot find module '@hookform/resolvers/zod'` (or runtime `Cannot find package` errors) when they imported `zodResolver` from the package. The 5.5.6 fix re-imports via a Zod-version-agnostic path so the adapter resolves on both v3 and v4.

**Practical impact:** projects on `zod@^3.x` that upgraded `@hookform/resolvers` from `5.4.x → 5.5.x` after July 25, 2026. The 5.5.6 bump makes `zodResolver` resolve cleanly regardless of which Zod major is installed.

**Audit:** `rg '"zod":' package.json | grep -E '"~?3\.'` — if you see Zod 3.x in `package.json` AND `@hookform/resolvers` is `>=5.5.0 <5.5.6`, you hit this. Bump to 5.5.6+ and the import works.

**Migration note for Zod v3 projects staying on the resolver:** the 5.5.6+ adapter is identical at the API surface (still `zodResolver(schema)`); only the internal import path changed. No code change needed.

### 4. `5.5.7` — Install: ERESOLVE Conflict with `valibot` (PR [#865](https://github.com/react-hook-form/resolvers/issues/865), commit [`722ef6e`](https://github.com/react-hook-form/resolvers/commit/722ef6e42eb29718763a66cfea91fe79e9cae081))

`npm install @hookform/resolvers@5.5.4` (or `5.5.5` / `5.5.6`) on a project that **already had `valibot` installed** (typically projects using the valibot resolver or just pulling valibot for `safeParse`-style utility) failed with:

```
npm error ERESOLVE could not resolve:
npm error Conflicting peer dependency: @hookform/resolvers@5.4.3
```

The root cause was a peer-dep range regression: `@hookform/resolvers@5.5.4–5.5.6` declared a peer that valibot's existing install couldn't satisfy without `--legacy-peer-deps`. The 5.5.7 fix relaxes the peer range so a clean `npm install` works alongside `valibot` (and any other peer that ships in the same `peerDependenciesMeta.optional = true` family).

**Practical impact:** anyone adding `@hookform/resolvers@5.5.4–5.5.6` to a project with valibot (even transitively, via another dep) hit `ERESOLVE`. Bumping to 5.5.7+ resolves it without needing `--legacy-peer-deps` or `--force`.

**Audit:** `rg '"valibot":' package.json` — if valibot is present AND your last install of `@hookform/resolvers` failed with ERESOLVE, bump to 5.5.7.

### 5. Recommended Version Pin (Updated)

```bash
npm install @hookform/resolvers@^5.5.7
```

> Pin moved up from `^5.5.3` (the previous cron's recommendation) to `^5.5.7` to capture the AJV / Yup / Zod v3 / valibot ERESOLVE fixes in a single bump.

**Migration checklist (5.5.3 → 5.5.7):**
- [ ] `npm install @hookform/resolvers@^5.5.7` — no peer-dep churn on the project's own deps; the 5.5.7 peer relaxation only affects installs of `@hookform/resolvers` itself
- [ ] If you mirror `form.getValues()` to a `useRef` because of the AJV default-leak (5.5.4), drop the workaround
- [ ] If your error UI reads `errors.<field>.ref` on checkbox fields with the Yup resolver (5.5.5), re-test after the bump — you'll start receiving RHF metadata again instead of Yup metadata
- [ ] If you were stuck on `@hookform/resolvers@5.4.x` because `5.5.0–5.5.6` ERESOLVEd against valibot (5.5.7), the bump unblocks the upgrade
- [ ] **No migration required** if you only use the Zod resolver on `zod@^4` with no checkboxes and no `getValues()` reads

**Sources:**
- [@hookform/resolvers v5.5.4 release notes](https://github.com/react-hook-form/resolvers/releases/tag/v5.5.4)
- [@hookform/resolvers v5.5.5 release notes](https://github.com/react-hook-form/resolvers/releases/tag/v5.5.5)
- [@hookform/resolvers v5.5.6 release notes](https://github.com/react-hook-form/resolvers/releases/tag/v5.5.6)
- [@hookform/resolvers v5.5.7 release notes](https://github.com/react-hook-form/resolvers/releases/tag/v5.5.7)
- [PR #862 — AJV: getValues() default-leak fix](https://github.com/react-hook-form/resolvers/issues/862)
- [PR #863 — Yup: checkbox ref-property fix](https://github.com/react-hook-form/resolvers/issues/863)
- [PR #864 — Zod v3 module-not-found import fix](https://github.com/react-hook-form/resolvers/issues/864)
- [PR #865 — valibot ERESOLVE peer-dep fix](https://github.com/react-hook-form/resolvers/issues/865)

## @hookform/resolvers 5.5.8 + 5.6.0 (August 1, 2026) — Zod Resolver Drops Special Root Field Errors + All-Resolvers Generalization

Two releases shipped on the same day (5.5.8 at 2026-08-01T08:48:55Z, 5.6.0 at 2026-08-01T09:13:32Z — only 25 minutes apart). Both are about the **same underlying behavior**: Zod (and now all resolvers) was silently swallowing validation errors that targeted field names with special meaning in JavaScript / JSON-pointer notation — like `'__proto__'`, `'constructor'`, `'prototype'`, `'length'`, or `'hasOwnProperty'`. These are field-name choices a developer might make for a config-style form (`settings = { __proto__: {...} }`, `meta = { constructor: ... }`) or a CSS-property-shaped schema (`styles = { length: '100%' }`). Pre-5.5.8, Zod's resolver dropped those errors silently — the user submitted the form with no visible error, but `form.formState.errors` was empty even though Zod internally had validation failures. 5.5.8 fixes Zod specifically; 5.6.0 (the new minor) generalizes the fix to every resolver (Yup, AJV, Valibot, Joi, Superstruct, Typanion, etc.).

### 1. `5.5.8` — Zod Resolver Drops Validation Errors for Special Root Field Names ([PR #869](https://github.com/react-hook-form/resolvers/issues/869), commit [`2f28787`](https://github.com/react-hook-form/resolvers/commit/2f287871f4d3184892cba1ac1a0c570a4e1cf8f5))

The Zod resolver was iterating the resolved error tree and dropping any error whose **field-path root** matched a property name in `Object.prototype` (`__proto__`, `constructor`, `prototype`, `hasOwnProperty`, `toString`, `valueOf`, etc.). The intent was a security hardening — stop a poisoned Zod error tree from polluting `form.formState.errors.__proto__` and corrupting the prototype chain — but it was applied too broadly: **legitimate errors targeting any of those field names got silently dropped**. If your Zod schema had `z.object({ __proto__: z.string(), ... })` and validation failed on `__proto__`, the form would not show any error.

**Practical impact:** any Zod-schema form that uses object keys matching `Object.prototype` properties. Common in real-world code:
- **Config-style forms**: `{ __proto__: z.object({...}) }`, `{ constructor: z.literal('MyClass') }` — JavaScript engine quirks + JSON-pointer tooling historically required special handling for these names.
- **CSS / DOM-keyed forms**: `{ length: z.number(), tagName: z.string() }` — fields whose names overlap with HTML element properties.
- **Form-library escape-hatch fields**: `{ hasOwnProperty: z.boolean(), toString: z.string() }` — any custom field-name that overlaps with `Object.prototype`.

**Audit:** `rg -n '(__proto__|constructor|prototype|hasOwnProperty|toString|valueOf):\s*z\.' schemas/ src/` — if your Zod schemas use any of these as a **root-level** object key (not a deeply-nested key — only the top level is affected), you were silently losing validation errors on those fields. After bumping to 5.5.8+, errors on those fields flow through to `form.formState.errors` correctly.

### 2. `5.6.0` — Improve All Resolvers: Drops Validation Errors for Special Root Field Names ([commit `b011a5f`](https://github.com/react-hook-form/resolvers/commit/b011a5f3f793dba475b143cca34859997cdfb161))

Generalizes the 5.5.8 Zod-specific fix to **every resolver in the package**: Yup, AJV, Valibot, Joi, Superstruct, Typanion, Vest, and the Typebox resolver all get the same special-name handling. The fix is now applied uniformly at the package's error-formatting layer rather than per-resolver.

**Why this is a MINOR (5.6.0) and not another PATCH (5.5.9):** the patch would only touch Zod, but the same security hardening applies equally to every other resolver. Bumping to a minor signals that the new behavior — "validation errors are NOT dropped for special root field names" — is the new contract across the whole package, not just for Zod. This is a **deliberate breaking change for any project that relied on the previous "errors silently dropped on these names" behavior** (rare in practice — the silent-drop was almost always a bug, not a feature — but worth noting for projects with custom error-rendering code that may have masked the missing errors with a UI-level fallback).

**Practical impact by resolver:**
- **Yup** — same fix; projects using `yup.object({ __proto__: ... })` get the dropped-error restoration.
- **AJV** — same fix; AJV JSON Schemas with `"properties": { "__proto__": {...} }` get the dropped-error restoration.
- **Valibot** — same fix; `v.object({ __proto__: ... })` gets the dropped-error restoration.
- **Joi, Superstruct, Typanion, Vest, Typebox** — same fix; all resolvers in the package adopt the new error-handling layer.
- **Any resolver-based form using `__proto__` / `constructor` / etc. as a root field name** was silently dropping errors pre-5.6.0. Bump to 5.6.0 to restore them.

**Audit (any resolver):** `rg -n '(__proto__|constructor|prototype|hasOwnProperty|toString|valueOf):' schemas/ src/ --type ts --type tsx` — any root-level field name that matches an `Object.prototype` property is affected. Bump to 5.6.0 to get the fix across all resolvers.

**Workaround before 5.6.0:** wrap the resolver to manually re-walk the error tree and re-emit any errors targeting special names (e.g. `customResolver = (schema) => async (values, ctx, opts) => { const result = await zodResolver(schema)(values, ctx, opts); /* custom recovery */ }`). After the bump, drop the wrapper.

### 3. Recommended Version Pin (Updated)

```bash
npm install @hookform/resolvers@^5.6.0
```

> Pin moved up from `^5.5.7` (the previous cron's recommendation) to `^5.6.0` to capture the Zod-only fix (5.5.8) + the all-resolvers generalization (5.6.0) in a single bump. Pure additive for projects not using `Object.prototype` field names — **zero behavior change** for the 99% case. For the 1% that use those names, the bump **restores silently-dropped errors** (a fix, not a break).

**Migration checklist (5.5.7 → 5.6.0):**
- [ ] `npm install @hookform/resolvers@^5.6.0` — no peer-dep churn on the project's own deps; the fix is internal to the package's error-formatting layer
- [ ] If you wrote a custom resolver wrapper to recover `Object.prototype` field errors (rare — most projects didn't notice the silent drop), drop the wrapper after the bump
- [ ] If you used any field-name matching `Object.prototype` properties (`__proto__`, `constructor`, `prototype`, `hasOwnProperty`, `toString`, `valueOf`, `length`, etc.) as a **root-level** schema field, verify your form now shows validation errors on those fields after the bump (it was silently dropping them pre-5.5.8 / 5.6.0)
- [ ] **No migration required** for the 99% of forms that use ordinary field names — zero observable change

**Sources:**
- [@hookform/resolvers v5.5.8 release notes](https://github.com/react-hook-form/resolvers/releases/tag/v5.5.8)
- [@hookform/resolvers v5.6.0 release notes](https://github.com/react-hook-form/resolvers/releases/tag/v5.6.0)
- [PR #869 — Zod resolver drops special root field names fix](https://github.com/react-hook-form/resolvers/issues/869)
- [commit `b011a5f` — all-resolvers generalization](https://github.com/react-hook-form/resolvers/commit/b011a5f3f793dba475b143cca34859997cdfb161)

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


### Zod 4.4 (April 29, 2026) — Correctness & Soundness Tightening

Zod 4.4.0 was released on **April 29, 2026** (followed by patches 4.4.1 → 4.4.2 → 4.4.3 on May 4, 2026). The `zod@latest` dist-tag is **`4.4.3`** as of 2026-07-24. This is a "minor-but-sharp" release — most fixes are correctness-driven, and several are **intentionally stricter** than Zod 4.0–4.3. Code that depended on previously accepted invalid or ambiguous inputs may fail differently after the bump. **Audit + `npx tsc --noEmit` are mandatory before merging.**

```bash
# Recommended
npm install zod@^4.4.3
```

#### 1. Potentially Breaking Bug Fixes (Read These First)

**Tuple defaults now materialize output values correctly** (PR [#5661](https://github.com/colinhacks/zod/pull/5661))

Before 4.4, a tuple like `z.tuple([z.string(), z.string().default("fallback")])` parsed `["a"]` as `["a"]` (the default was silently dropped from the output). 4.4 makes the default materialize into the parsed output:

```ts
const schema = z.tuple([
  z.string(),
  z.string().default("fallback"),
])

// 4.3 and earlier: ["a"]           (default dropped)
// 4.4+: ["a", "fallback"]          (default in output)
schema.parse(["a"])

// Trailing optional elements stay absent (not filled with undefined):
const optionalTail = z.tuple([z.string(), z.string().optional()])
optionalTail.parse(["a"])                    // ["a"]
optionalTail.parse(["a", undefined])         // ["a", undefined] (explicit undefined preserved)

// Optional BEFORE default → dense tuple (undefined fills the slot):
const dense = z.tuple([z.string(), z.string().optional(), z.string().default("fb")])
dense.parse(["a"]) // ["a", undefined, "fb"]
```

**Why this matters for forms:** `z.tuple([…])` schemas where the last element has a `.default()` will now return that default in the parsed value. If you snapshot the parsed object into a Store (Zustand) or React state, the default appears in the UI. Side effects and equality checks that assumed the default was silently dropped will need to be updated.

**Required object properties with `z.undefined()`** (PR [#5661](https://github.com/colinhacks/zod/pull/5661), follow-up [`57d80a82`](https://github.com/colinhacks/zod/commit/57d80a82bde8877f3eb79e5dad9786096c37490f))

Before 4.4, an object property typed as `z.undefined()` was treated as optional. 4.4 makes it required — the key must be present, but its value may be `undefined`:

```ts
const schema = z.object({ value: z.undefined() })

schema.safeParse({}).success                    // false (key MUST be present)
schema.safeParse({ value: undefined }).success  // true

// Use .optional() when the key itself may be absent:
const schema2 = z.object({ value: z.undefined().optional() })
schema2.safeParse({}).success  // true
```

**Why this matters for forms:** If you used `z.undefined()` as a "send-only marker" (e.g., a hidden CSRF field that must be present in the payload), your Zod 4.0–4.3 schema was silently accepting payloads missing it. 4.4 correctly rejects those. Audit:

```bash
rg "z\.undefined\(\)" --type ts --type tsx src/
```

**`.merge()` throws on receiver with refinements; second schema's refinements are preserved** (PR [#5856](https://github.com/colinhacks/zod/pull/5856))

Before 4.4, `A.merge(B)` would silently lose refinements on `A` in some cases. 4.4 makes `.merge()` throw noisily when the receiver has refinements (to force you to think about whether they should apply) and guarantees the second schema's refinements are preserved.

```ts
const A = z.object({ x: z.string() }).refine(o => o.x.length > 0, 'positive')
const B = z.object({ y: z.number() })

// 4.4+: throws on A.merge(B) — A has refinements, decide intent first
// Workaround: apply refinements AFTER the merge:
const merged = A.merge(B).refine(o => o.x.length > 0, 'positive')
```

**Map and Set defaults are now cloned** (PR [#5855](https://github.com/colinhacks/zod/pull/5855))

Before 4.4, `z.map(...).default(new Map())` and `z.set(...).default(new Set())` returned the same reference on every `.parse(undefined)`. 4.4 clones the default to prevent shared-mutable-state across parses:

```ts
const schema = z.map(z.string(), z.number()).default(new Map())
const a = schema.parse(undefined)
const b = schema.parse(undefined)
a === b // false in 4.4+ (true in earlier versions — shared state bug)
```

**Why this matters for forms:** If you mutate a parsed Map/Set (e.g., to add a freshly-selected option), every other form instance sharing the same default would see the mutation. 4.4 closes that footgun.

**Discriminated union error now includes `options` and `discriminator`** (PR [#5723](https://github.com/colinhacks/zod/pull/5723))

Discriminator failures now include the list of valid options and the field name, which makes user-facing error messages much clearer:

```ts
const schema = z.discriminatedUnion('type', [
  z.object({ type: z.literal('email'), email: z.string() }),
  z.object({ type: z.literal('sms'), phone: z.string() }),
])

// 4.4+ error JSON for { type: 'fax' }:
// { code: 'invalid_union_discriminator', options: ['email', 'sms'], discriminator: 'type', … }
```

**Why this matters for forms:** When you build a discriminated-union form (e.g., `NotificationStep { type: 'email' | 'sms' | 'push' }`), the error now tells you which options exist. Use it to drive a "choose one of: …" hint instead of a generic "invalid value" message.

#### 2. New Features

**`.superRefine()` now accepts a `when` function** (PR [#5741](https://github.com/colinhacks/zod/pull/5741))

Run a refinement only when the schema is otherwise valid. Equivalent to a mini-discriminated check inside the refinement:

```ts
const schema = z.object({
  email: z.string().email(),
  marketingOptIn: z.boolean().optional(),
}).superRefine(
  (val, ctx) => {
    if (val.marketingOptIn && !val.email.endsWith('@company.com')) {
      ctx.addIssue({
        code: 'custom',
        path: ['marketingOptIn'],
        message: 'Marketing opt-in requires a company email',
      })
    }
  },
  // NEW: 'when' only fires if the schema is otherwise valid
  { when: ({ success }) => success }
)

// `.refine()` got the same `when` option (PR #5681) — respects `abort: true` semantics.
```

**Transform context: `ctx.addIssue()` now works inside `.transform()`** (PR [#5699](https://github.com/colinhacks/zod/pull/5699))

Before 4.4, you couldn't add a Zod issue from inside a transform callback. 4.4 makes `ctx.addIssue()` available on the transform context:

```ts
const toDate = z.string().transform((val, ctx) => {
  const d = new Date(val)
  if (Number.isNaN(d.getTime())) {
    ctx.addIssue({ code: 'custom', message: 'Not a valid date string' })
    return z.NEVER
  }
  return d
})
```

**Codec inversion: `z.invertCodec()`** (PR [#5770](https://github.com/colinhacks/zod/pull/5770))

For bidirectional codec schemas (e.g., `stringToNumber = z.codec(z.string(), z.number(), { decode: parseFloat, encode: String })`), 4.4 adds a `z.invertCodec(codec)` helper that swaps the decode/encode directions. Useful for form schemas that need to round-trip a value to the wire format and back:

```ts
const stringToNumber = z.codec(z.string(), z.number(), {
  decode: parseFloat,
  encode: String,
})

const numberToString = z.invertCodec(stringToNumber)
// { decode: String, encode: parseFloat }  — the reverse

// Also: discriminatedUnion().encode() now works with codec discriminators (PR #5769)
```

**Tightening of `z.preprocess` optionality** (4.4.2, PR [#5929](https://github.com/colinhacks/zod/pull/5929))

`z.preprocess` now defers optionality to the inner schema. Previously a preprocessor could mark the outer result as optional, hiding inner-schema errors. 4.4.2 makes the inner schema the source of truth.

#### 3. Performance Improvements

**Lazy-bound builder methods** (PR [#5897](https://github.com/colinhacks/zod/pull/5897))

Classic builder methods (`.parse`, `.safeParse`, `.optional`, `.nullable`, etc.) are now lazy-bound through a shared internal prototype instead of being eagerly attached per schema instance. ~10–30% reduced memory for apps with thousands of schemas (typical for large form schemas with many sub-schemas). No code change required.

**Pure annotations for tree-shaking**

Top-level factory calls (`z.object`, `z.string`, `z.tuple`, etc.) are now annotated `/*@__PURE__*/` so bundlers (Rolldown, esbuild, Turbopack) tree-shake unused schemas from the final bundle. Particularly impactful for shared Zod schema files with hundreds of named schemas.

**Other small perf wins:**
- Avoid `delete` in `finalizeIssue` (PR #5718) — keeps V8 in fast mode
- `globalConfig` shared via `globalThis` (PR #5889) — improves behavior across mixed CJS/ESM module instances
- `jitless` config honored in `allowsEval` probe (PR #5864) — and avoids probing when set before first access

#### 4. Prototype Pollution Hardening

**Object catchall paths now skip `__proto__` keys** (PR [#5898](https://github.com/colinhacks/zod/pull/5898))

Schema features that accept extra keys (`z.object({...}).catchall(z.unknown())` or `.passthrough()`) now skip the `__proto__` key. A small but important prototype-pollution guard if you ever parse untrusted user input into a passthrough schema:

```ts
const schema = z.object({ name: z.string() }).catchall(z.unknown())
schema.parse({ name: 'a', __proto__: { polluted: true } })
// 4.4+: { name: 'a' }   (no __proto__ in output)
// 4.3 and earlier: would have included __proto__ in the parsed object
```

#### 5. Patch-train Summary (4.4.1 → 4.4.3)

| Version | Date | Headline |
|---|---|---|
| 4.4.0 | 2026-04-29 | Major release — tuple defaults, z.undefined() required, codec inversion, superRefine when, transform ctx.addIssue(), lazy-bound methods, prototype pollution hardening |
| 4.4.1 | 2026-04-29 | Same-day patch — reject tuple holes before required defaults (PR #5900) |
| 4.4.2 | 2026-05-01 | Tighten discriminated union option typing + z.preprocess defers optionality to inner schema (PR #5929) |
| 4.4.3 | 2026-05-04 | Restore catch handling for absent object keys (PR #5939) + generalize optin/fallback to transform + restore preprocess on absent keys (PR #5941) |

**Recommended Zod version after 4.4.3: `^4.4.3`** (supersedes any earlier 4.x).

#### 6. Migration Checklist (any 4.x → 4.4.3)

- [ ] Run `npm install zod@^4.4.3`
- [ ] Run `npx tsc --noEmit` — the tightening will surface any code that relied on the looser 4.3 semantics
- [ ] Audit tuple defaults: if you depended on `.default()` inside tuples being silently dropped, update consumer code to expect the default in output
- [ ] Audit `z.undefined()` usages: any place that relied on `z.undefined()` being optional now requires the key to be present
- [ ] Audit `.merge()` calls: receiver with refinements now throws — apply refinements after the merge
- [ ] Audit Map/Set defaults: cloned per-parse now; if you relied on shared mutable state across parses, capture the default *outside* the schema
- [ ] Test discriminated-union error rendering if you customize `ZodError` messages — the new `options` field is gold for user-facing hints
- [ ] **No action required** for the new features (codec inversion, superRefine `when`, transform `ctx.addIssue`) — opt-in only

**Sources:**
- [Zod 4.4.0 release notes](https://github.com/colinhacks/zod/releases/tag/v4.4.0) — the headline release
- [Zod 4.4.1 release notes](https://github.com/colinhacks/zod/releases/tag/v4.4.1)
- [Zod 4.4.2 release notes](https://github.com/colinhacks/zod/releases/tag/v4.4.2)
- [Zod 4.4.3 release notes](https://github.com/colinhacks/zod/releases/tag/v4.4.3)
- [PR #5661 — tuple/object optionality alignment](https://github.com/colinhacks/zod/pull/5661)
- [PR #5770 — codec inversion](https://github.com/colinhacks/zod/pull/5770)
- [PR #5741 — superRefine `when` option](https://github.com/colinhacks/zod/pull/5741)
- [PR #5699 — transform context `addIssue`](https://github.com/colinhacks/zod/pull/5699)
- [PR #5897 — lazy-bound builder methods](https://github.com/colinhacks/zod/pull/5897)
- [PR #5898 — `__proto__` skip in catchall](https://github.com/colinhacks/zod/pull/5898)


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

## `React.SubmitEvent` vs Deprecated `React.FormEvent` (React 19.2.10+)

`React.FormEvent` and `React.FormEventHandler` were **deprecated in React 19.2.10** in favor of `React.SubmitEvent` and `React.SubmitEventHandler`. The old types still work but trigger a deprecation warning on every form `onSubmit` handler. Next.js 16.3.0-canary.77+ (PR [#95453](https://github.com/vercel/next.js/pull/95453) by M4cM4rco) updated the official form-handling docs to use `React.SubmitEvent`; the skill's `api.md` chat-stream example was updated to match.

**Use `React.SubmitEvent<HTMLFormElement>` (React 19.2.10+):**

```tsx
// ✅ Recommended — React 19.2.10+
'use client'
import { useState } from 'react'

async function handleSubmit(e: React.SubmitEvent<HTMLFormElement>) {
  e.preventDefault()
  // ...
}

export function MyForm() {
  return <form onSubmit={handleSubmit}>{/* ... */}</form>
}
```

**Avoid `React.FormEvent<HTMLFormElement>` (deprecated):**

```tsx
// ❌ Deprecated — still works but emits a deprecation warning
async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
  e.preventDefault()
  // ...
}
```

**Migration scope:**

- Only `<form onSubmit>` handlers are affected. `onChange`, `onBlur`, `onInput`, `onFocus`, etc. continue to use `ChangeEvent`, `FocusEvent`, `FormEvent` (the `FormEvent` here refers to the broader type that includes focus/change events, not the deprecated submit-specific alias — confusingly the same name).
- The `FormEventHandler` type alias was renamed to `SubmitEventHandler` for parity. Search for `: React.FormEventHandler` and `React.FormEvent<` in your codebase to find migration targets.
- `SubmitEvent` is available from `react` (not `react-dom`) — same import path as the deprecated `FormEvent` was.

**Codebase grep for migration:**

```bash
# Find all deprecated React.FormEvent usages
grep -rn "React\.FormEvent" --include="*.tsx" --include="*.ts" src/

# Find deprecated React.FormEventHandler usages
grep -rn "React\.FormEventHandler" --include="*.tsx" --include="*.ts" src/
```

**Source:** Next.js [PR #95453 — `docs: Update FormEvent to SubmitEvent in form handling example (deprecated in React 19.2.10+)`](https://github.com/vercel/next.js/pull/95453) · [react-router issue #14795](https://github.com/remix-run/react-router/issues/14795) for the cross-framework context.
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
- **Zod 4.4: assuming tuple defaults are silently dropped** — defaults materialize in output now; snapshot equality + Zustand store updates will see the new value
- **Zod 4.4: treating `z.undefined()` as optional** — it's required now; key must be present (use `.optional()` for absent-key semantics)
- **Zod 4.4: `.merge()`-ing schemas where the receiver has refinements** — throws now; apply refinements after `.merge()`
- **Zod 4.4: depending on shared mutable Map/Set defaults across parses** — defaults are cloned per-parse now; capture the default outside the schema if you relied on shared state
- **Zod 4.4: skipping `npx tsc --noEmit` after the bump** — the tightening will surface any code that relied on the looser 4.3 semantics; the fix is rarely a one-line type cast
- **Zod 4.4: regenerating OpenAPI / JSON Schema output without re-checking consumer code** — the `$defs` redundant-id strip (PR #5759) and the `min/max` intersection fix (PR #5700) may change the generated schema shape; consumers that pinned to the exact previous output will need a re-coordination
- **Zod 4.4: missing `npx tsc --noEmit` AND `npx vitest run`** — both fixes and new features (codec inversion, superRefine `when`, transform `ctx.addIssue()`) are TS-shape changes; run *both* type and runtime checks

- **RHF 8: test breaking changes before upgrading** — v8 beta is not production-stable; the `useForm` API has breaking changes including `id`→`key` rename, `keyName` removal, `names`→`name` in Watch, `watch` callback→`subscribe`, and `setValue` no longer updating field arrays

- **RHF 7.83: keeping a `JSON.stringify` workaround on `formState.dirtyFields`** — the reference is now stable across interactions; you can compare with `===` instead of serializing, and Zustand/React Query keys built from `dirtyFields` will stop invalidating on every keystroke
- **RHF 7.83: not re-running `npx tsc --noEmit` after bumping to 7.83.0** — the 10-level recursion hard cap (PR #13529) is a measurable `tsc` win on deeply-typed forms; verify it landed by timing the type-check before/after
- **RHF 7.83: massaging `e.target.files` manually on `<input type="file">` registered via `register()`** — `getEventValue` (PR #13289) now yields an `Array<File>`; the manual `FileList → Array` conversion can be dropped
- **RHF 7.83: recreating the `control` object in tests / HOCs without re-subscribing the `useController`** — PR #13603 fixed `useController` to follow the current `control`, so old test code that "worked by accident" may now expose previously-masked bugs; audit the test expectations
- **RHF 7.84: passing a Server Action via `<Form onSubmit>` instead of `<Form action>`** — 7.84.0's new function-based `action` prop on `<Form />` lets you wire a Next.js 16 Server Action directly for progressive enhancement (the form works without JS, then RHF validates client-side). Pass the Server Action as `action={saveUser}`, not as `onSubmit={(data) => saveUser(data)}`. Also lets you `const result = await handleSubmit(async (data) => saveUser(data))` and react to the action's return value via 7.84.0's typed `handleSubmit` return.
- **RHF 7.84: discarding the `handleSubmit` return value** — 7.84.0 makes `handleSubmit`'s return type equal to `Awaited<ReturnType<typeof onValid>>` (union with `void`). Code that did `handleSubmit(async (data) => { const id = await save(data); store.set(id) })` can now do `const { savedId } = await handleSubmit(async (data) => { const id = await save(data); return { savedId: id } })` — eliminates the side-channel.
- **RHF 7.84: leaving `@ts-ignore` / deep-imports on `FieldArray` or `FormState` types** — both type exports are now re-exported from the package root in 7.84.0 (the 7.81+ bundler-config regression is closed). Drop the workarounds and use the root imports.
- **RHF 7.84: pinning `react-hook-form@7.83.0`** — missing the `<Form action={fn}>` Server Action style, the typed `handleSubmit` return, the `keepDirtyValues` nested-object fix, the `setValues` → `useFieldArray` notification, the type-export restoration, and the bundle-size reduction. Bump to `^7.84.0` to pick up the lot. Pure additive patch train (no breaking changes).
- **`@hookform/resolvers` 5.5.4: AJV resolver silently overwriting `getValues()` with AJV `default` values** — every validation pass was merging schema defaults into the values object. Forms that read `getValues()` after submit (confirmation step, receipt render, post-submit API call) shipped the schema defaults instead of user input. Bump to `^5.5.7` and drop any `useRef`-mirror workaround you wrote to side-step the leak.
- **`@hookform/resolvers` 5.5.5: Yup resolver stomp on `errors.ref` for checkbox fields** — Yup's metadata populated a top-level `errors.ref` that masked RHF's `errors.<field>.ref` on `<input type="checkbox">` fields. Custom error UI reading `errors.<field>.ref` on checkboxes was getting Yup schema-path metadata, not RHF metadata. Bump to `^5.5.7` and re-test your checkbox error UI.
- **`@hookform/resolvers` 5.5.6: `zodResolver` import throws `Module not found` on `zod@^3.x`** — projects still on Zod v3 broke on `5.5.0–5.5.5` because the Zod adapter relied on a v4-shaped export path. Bump to `^5.5.7` and the import resolves on both Zod v3 and v4 (no code change needed).
- **`@hookform/resolvers` 5.5.7: `npm install` ERESOLVE with `valibot` already installed** — the 5.5.4–5.5.6 peer range was too tight to coexist with `valibot`. Bump to `^5.5.7` (or pin the older `@hookform/resolvers` version with `--legacy-peer-deps` if you can't bump yet).
- **`@hookform/resolvers` 5.5.8 / 5.6.0: silently dropping validation errors on `Object.prototype` field names** — Zod resolver (5.5.8) + every resolver in the package (5.6.0) was iterating the error tree and dropping any error whose root field-path matched `__proto__` / `constructor` / `prototype` / `hasOwnProperty` / `toString` / `valueOf` / `length`. Forms using any of those as a root-level schema key submitted with no visible error and an empty `form.formState.errors`. Bump to `^5.6.0` to restore the errors. Audit recipe: `rg -n '(__proto__|constructor|prototype|hasOwnProperty|toString|valueOf):' schemas/ src/ --type ts --type tsx` — any root-level schema key matching an `Object.prototype` property was silently dropping its errors pre-5.6.0.
- **`@hookform/resolvers` 5.6.0: relying on the silent-drop behavior as a "feature"** — pre-5.6.0, the Zod (and per-resolver) security hardening intentionally dropped errors on `Object.prototype` field names to prevent prototype pollution. If you had a custom error renderer that masked the missing errors with a UI fallback (rare), bumping to 5.6.0 surfaces them — drop the fallback after the bump.
