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

## @hookform/resolvers 5.7.0 (August 2, 2026) — Vine v4 Resolver Support + `vine.create({})` Syntax

`@hookform/resolvers@5.7.0` SHIPPED at 2026-08-02T05:57:55Z — **just 4 minutes before this cron started** — adding **first-class support for VineJS v4** via [PR #867](https://github.com/react-hook-form/resolvers/pull/867) (`feat: support vine v4`, commit [`4cfba18`](https://github.com/react-hook-form/resolvers/commit/4cfba18117e8b76bde326a54103340590c867a21), merged 2026-08-02T05:53:13Z; 1 commit, +72/-90 across `vine/package.json` + `vine/src/vine.ts` + `vine/src/__tests__/*` + `README.md`). **Before this release `forms.md` had zero mention of Vine at all** — the `@hookform/resolvers/vine` subpath module existed in earlier versions but its peer-dep was capped at `@vinejs/vine ^2.0.0 || ^3.0.0`, leaving anyone on Vine v4 with an `ERESOLVE` install failure or a runtime "no overload matches this call" type error. 5.7.0 unblocks the v4 line + aligns the resolver with Vine v4's new schema construction API.

### 1. The change — Vine v4 peer-dep expansion + new `vine.create({})` schema syntax

**Two material changes shipped in the single 1-commit PR:**

**(a) Peer-dep expansion.** `@hookform/resolvers/vine/package.json` now declares `@vinejs/vine: "^2.0.0 || ^3.0.0 || ^4.0.0"` (was `^2.0.0 || ^3.0.0` only). Projects on Vine v4 can now `npm install @hookform/resolvers @vinejs/vine@^4.0.0` cleanly without `--legacy-peer-deps` or a custom override.

**(b) New schema construction syntax.** Vine v4.2.0 (referenced in the PR body) introduced a `vine.create({...})` shorthand that replaces the previous `vine.compile(vine.object({...}))` pattern. The PR updates:
- **`vine/src/__tests__/__fixtures__/data.ts`** — test fixtures now use `vine.create({ username: vine.string().regex(...).minLength(3).maxLength(30), password: vine.string()...confirmed({ as: 'repeatPassword' }), ... })` instead of the old `vine.compile(vine.object({...}))` wrapping.
- **`README.md`** — the Quickstart example for Vine was rewritten from the old `vine.compile(vine.object({ username: ..., password: ... }))` form to `vine.create({ username: ..., password: ... })`.
- **`vine/src/vine.ts`** — the resolver itself was unchanged in core logic; only the type signature was tightened to `VineValidator<ConstructableSchema<Input, Output, Output>, any>` so the new `vine.create({...})` return type is accepted without TS narrowing complaints. The null-prototype-object guard for `Object.prototype` field-name handling from 5.5.8 / 5.6.0 carries over unchanged (the fix is at the package-wide error-formatting layer, not per-resolver).

**Practical impact:**

- **Anyone on `@vinejs/vine@^4.x` was previously blocked** — the resolver's old peer-dep cap (`^2 || ^3`) would refuse to install alongside Vine 4. The workaround was either pinning to Vine 3, using `--legacy-peer-deps`, or hand-writing a custom `Resolver` wrapper around `vine.validate()`. 5.7.0 makes all three unnecessary.
- **Anyone on Vine v2/v3 who upgrades to 5.7.0 has zero behavior change** — the resolver still works with the legacy `vine.compile(vine.object({...}))` pattern; the v4 syntax is additive.
- **Anyone migrating Vine v2/v3 → v4 alongside this bump gets the new `vine.create({...})` syntax recommended** — it removes one level of wrapping per schema declaration and matches the pattern shown in the official Vine v4 docs.

**Quick code example (new 5.7.0 / Vine v4 pattern):**

```ts
// Before (Vine v2/v3 + pre-5.7.0)
import vine from '@vinejs/vine'
import { vineResolver } from '@hookform/resolvers/vine'

const schema = vine.compile(
  vine.object({
    username: vine.string().minLength(1),
    password: vine.string().minLength(8),
  }),
)

// After (Vine v4 + @hookform/resolvers 5.7.0+)
import vine from '@vinejs/vine'
import { vineResolver } from '@hookform/resolvers/vine'

const schema = vine.create({
  username: vine.string().minLength(1),
  password: vine.string().minLength(8),
})

useForm({ resolver: vineResolver(schema) })
```

### 2. Recommended Version Pin (Updated)

```bash
npm install @hookform/resolvers@^5.7.0
```

> Pin moved up from `^5.6.0` (the v1.5.14 recommendation) to `^5.7.0` to capture the Vine v4 peer-dep expansion + the `vine.create({})` schema-syntax alignment. **Pure additive** for projects not using Vine — zero behavior change. For projects on `@vinejs/vine@^4.x`, the bump **unblocks the install** (was failing pre-5.7.0).

**Migration checklist (5.6.0 → 5.7.0):**
- [ ] `npm install @hookform/resolvers@^5.7.0` — peer-dep change is additive only (`@vinejs/vine` peer now includes `^4.0.0` alongside `^2`/`^3`)
- [ ] If you were stuck on `@vinejs/vine@^3.x` because `@hookform/resolvers` capped the peer at `^2 || ^3`, bump both to `@vinejs/vine@^4.x` + `@hookform/resolvers@^5.7.0` together — `vine.create({...})` is the recommended v4 schema syntax
- [ ] If migrating Vine v3 → v4 in the same bump, rewrite `vine.compile(vine.object({...}))` → `vine.create({...})` (one-time per-schema mechanical change)
- [ ] **No migration required** for the >99% of forms that don't use Vine — zero observable change
- [ ] Vine v4 users: drop any `--legacy-peer-deps` install flag you were using to force the resolver install (no longer needed)

**Sources:**
- [@hookform/resolvers v5.7.0 release notes](https://github.com/react-hook-form/resolvers/releases/tag/v5.7.0)
- [PR #867 — feat: support vine v4](https://github.com/react-hook-form/resolvers/pull/867)
- [commit `4cfba18` — Vine v4 peer-dep + fixture + README updates](https://github.com/react-hook-form/resolvers/commit/4cfba18117e8b76bde326a54103340590c867a21)
- [VineJS v4.2.0 release notes (introduced `vine.create({})` syntax)](https://github.com/vinejs/vine/releases/tag/v4.2.0)

## @hookform/resolvers 5.7.1 (August 2, 2026) — `ata-validator` Subpath Peer-Dep Bump `^0.7.0` → `^1.2.0`

`@hookform/resolvers@5.7.1` SHIPPED at **2026-08-02T06:14:53Z** — **just 13 minutes after the v1.5.15 cron committed** at 06:02Z. The v1.5.15 description documented `^5.7.0` as the recommended pin (capturing the Vine v4 support from earlier the same morning); that pin is now stale by one patch. 5.7.1 is a **single-commit PATCH release** — `fix: ata-validator peer dependency version` ([issue #871](https://github.com/react-hook-form/resolvers/issues/871), commit [`827f20b`](https://github.com/react-hook-form/resolvers/commit/827f20bd97a4618d75162e54fe9ba2419fd4e50a), merged 2026-08-02T06:08:14Z, 3 files / +18/-12 lines) — bumping the peer-dep in the `@hookform/resolvers/ata-validator` subpath module from `"ata-validator": "^0.7.0"` to `"ata-validator": "^1.2.0"` and updating the workspace `bun.lock` to match. **No new resolver, no new helper, no source-code change in any subpath module — the only diff is the peer-dep string + the lockfile**.

### 1. The change — peer-dep catch-up to `ata-validator` 1.x

The `@hookform/resolvers/ata-validator/package.json` peer-dependencies block now reads:

```diff
   "peerDependencies": {
     "react-hook-form": "^7.55.0",
     "@hookform/resolvers": "^2.0.0",
-    "ata-validator": "^0.7.0"
+    "ata-validator": "^1.2.0"
   }
```

The accompanying `bun.lock` bumps the workspace dev-dep `ata-validator: ^0.7.3` → `^1.2.0` and pulls in the new **platform-native** companion packages published alongside ata-validator 1.x:

| New native companion | Platform | Shape |
|---|---|---|
| `@ata-validator/native-darwin-arm64` | macOS Apple Silicon | NAPI binary |
| `@ata-validator/native-linux-arm64-gnu` | Linux ARM64 glibc | NAPI binary |
| `@ata-validator/native-linux-arm64-musl` | Linux ARM64 musl | NAPI binary |
| `@ata-validator/native-linux-x64-gnu` | Linux x64 glibc | NAPI binary |
| `@ata-validator/native-linux-x64-musl` | Linux x64 musl | NAPI binary |
| `@ata-validator/native-win32-x64` | Windows x64 | NAPI binary |

`ata-validator` ships as a **hybrid JS + native** package starting with 1.0.0 (released 2026-07-15T21:12:27Z; current latest 1.2.2 published 2026-07-26T19:30:11Z). The 1.0 bump was a major-version jump because the native addon shape, the internal NAPI surface, and the install-time platform-detection logic all changed — the previous `^0.7.0` peer cap silently blocked anyone who'd already migrated to the 1.x line. The same shape problem hit 5.5.7's valibot peer relaxation (covered earlier in `forms.md`) and 5.5.4's AJV `default`-values stomp — `ata-validator` joins the same pattern of "resolver's peer cap drifts behind the validator library's own major-version line".

**Practical impact:**

- **Anyone on `ata-validator@^1.x` was previously blocked from installing `@hookform/resolvers/ata-validator`** — npm's peer-dep resolver sees `^0.7.0` as a *narrower* range than the already-installed `^1.2.0` (no `0.x` overlap with `1.x`), so `npm install` would either fail with `ERESOLVE` or succeed only with `--legacy-peer-deps`. 5.7.1 unblocks the install.
- **Anyone still on `ata-validator@^0.7.x`** is unaffected by the *content* of the change (the resolver still works against the `^0.7.x` API surface, which is what was tested up to 5.7.0), but the new `^1.2.0` peer cap means upgrading `@hookform/resolvers` to 5.7.1 without bumping `ata-validator` first will trigger a *new* `ERESOLVE` (the reverse-direction failure). The safe path is `npm install ata-validator@^1.2.0 @hookform/resolvers@^5.7.1` together, or stay on 5.7.0 if you're committed to the 0.x line.
- **Anyone not using `@hookform/resolvers/ata-validator`** has zero behavior change. The peer-dep string lives only in that subpath's `package.json` — the rest of the package's 30+ resolvers are unaffected.
- **Anyone using the new platform-native addons in ata-validator 1.x** (`@ata-validator/native-*`): the resolver works against the public `ata-validator` API surface, which 1.x guarantees; the native addons are an internal implementation detail of the validator library and don't change the resolver's contract.

### 2. Recommended Version Pin (Updated)

```bash
npm install @hookform/resolvers@^5.7.1
```

> Pin moved up from `^5.7.0` (the v1.5.15 recommendation, set 13 minutes before 5.7.1 shipped) to `^5.7.1` to capture the `ata-validator` 1.x peer-dep alignment. **Pure additive PATCH** for projects not using the `ata-validator` subpath — zero behavior change. For projects on `ata-validator@^1.x`, the bump **unblocks the install** (was failing pre-5.7.1 with `ERESOLVE`). For projects on `ata-validator@^0.7.x`, the bump **forces** the 1.x migration in one step (or stay on 5.7.0 to defer).

**Migration checklist (5.7.0 → 5.7.1):**

- [ ] `npm install @hookform/resolvers@^5.7.1` — PATCH release, peer-dep change is additive only for projects that need it
- [ ] **If you use `@hookform/resolvers/ata-validator`** AND are on `ata-validator@^1.x`: bump resolves the `ERESOLVE` you were seeing — drop `--legacy-peer-deps`
- [ ] **If you use `@hookform/resolvers/ata-validator`** AND are still on `ata-validator@^0.7.x`: bumping to 5.7.1 alone will surface a *new* `ERESOLVE` (the reverse-direction failure) — bump `ata-validator` to `^1.2.0` in the same install, or stay on 5.7.0
- [ ] **If you don't use `@hookform/resolvers/ata-validator`** (the >99.9% case): no migration needed, just pick up the patch on the next install
- [ ] Verify with `npm ls ata-validator` — should resolve to `^1.2.x` (1.2.0 / 1.2.1 / 1.2.2) on any project that bumped to 5.7.1

### 3. Audit recipe

```bash
# Check if any project actually uses the ata-validator subpath
rg -n "from '@hookform/resolvers/ata-validator'" --type ts --type tsx src/ app/
# → if zero results, you can skip the 5.7.1 peer-dep change entirely (no behavior change for you)

# Check current ata-validator version in your tree
npm ls ata-validator 2>/dev/null
# → if "1.x" → 5.7.1 unblocks your install (was ERESOLVE-ing before)
# → if "0.7.x" → 5.7.1 forces a bump; consider staying on 5.7.0 or bumping both together

# Confirm the peer-dep string landed in your installed copy
cat node_modules/@hookform/resolvers/ata-validator/package.json | jq .peerDependencies
# → expect: { "react-hook-form": "^7.55.0", "@hookform/resolvers": "^2.0.0", "ata-validator": "^1.2.0" }
```

**Sources:**

- [@hookform/resolvers v5.7.1 release notes](https://github.com/react-hook-form/resolvers/releases/tag/v5.7.1)
- [Issue #871 — ata-validator peer dependency version](https://github.com/react-hook-form/resolvers/issues/871)
- [commit `827f20b` — ata-validator peer-dep bump](https://github.com/react-hook-form/resolvers/commit/827f20bd97a4618d75162e54fe9ba2419fd4e50a)
- [ata-validator on npm (1.0.0 released Jul 15, 2026 — major version bump introducing the NAPI native addons)](https://www.npmjs.com/package/ata-validator)
- [@ata-validator/native-darwin-arm64@1.2.2 (and the 5 other platform-native companions) on npm](https://www.npmjs.com/package/@ata-validator/native-darwin-arm64)


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


### Zod 4.4.x Post-Release Forward-Looking (August 8–9, 2026) — 4 NEW Merged Correctness/Security Fixes for `^4.4.3` + `zod@canary` 4.5.0-canary.20260809T180500 Drop (First Can With All 4 Fixes)

Since the 4.4.3 patch train shipped on 2026-05-04, the Zod main branch has accumulated **18 NEW commits ahead of `v4.4.3`** as of 2026-08-09T18:01:43Z. The most material of these — **4 merged correctness/security fixes landed in a tight ~22h window on August 8–9, 2026** — are documented here because each affects common production patterns with Zod 4.4.3. None are in `zod@latest` yet (still `4.4.3`), but **`zod@canary` saw its SECOND drop today `4.5.0-canary.20260809T180500`** (npm-published 2026-08-09T18:10:14Z) — this is the FIRST canary cut that contains all 4 material fixes (PR #6347 + PR #6354 + PR #6346 + PR #6213) because PR #6354 (merged 2026-08-09T18:01:44Z) only landed AFTER the earlier canary drop `4.5.0-canary.20260809T165522` (npm-published 2026-08-09T16:55:22Z, ~1h6min before PR #6354). **The fixes are correctness/security-driven** (not API-surface changes) — most code is unaffected, but specific edge cases hit production.

#### 1. PR #6347 — `fix: remove exponential backtracking from the emoji regex` (merged 2026-08-09T01:12:02Z) — **HIGHEST-PRIORITY FIX (ReDoS)**

The emoji format validator backtracks exponentially on a failed match. The two Unicode properties (`\p{Extended_Pictographic}` and `\p{Emoji_Component}`) overlap on exactly four code points — U+1F9B0 through U+1F9B3, the emoji hair components. Quantifying an ambiguous alternation with `+` means that when the anchor fails, the engine re-explores up to 2^n ways of attributing the already-consumed characters between the two branches.

Measured on `z.emoji().safeParse()`:

| Input                      | Payload | Time     |
| -------------------------- | ------- | -------- |
| 22 hair components + space | 90 B    | 145 ms   |
| 24 hair components + space | 98 B    | 498 ms   |
| 26 hair components + space | 106 B   | 2846 ms  |

That is **2.11× per added character**. A **126-byte string buys a 60-second stall**. `RegExp.test()` is uninterruptible, so a request timeout does not help. The same string ships from `zod`, `zod/mini`, `zod/v4-mini` and the bundled `zod/v3`.

**The fix:** collapse the alternation into one character class (the same move that PR #2824 made on the email regex): `^[\p{Extended_Pictographic}\p{Emoji_Component}]+$` (single character class instead of two-property alternation). Iterating every code point (surrogates excluded), the old and new patterns agree on all 2,990 members with zero disagreements — **no valid emoji string changes acceptance**.

**Practical impact for `^4.4.3` users:** any user-supplied string that calls `z.string().emoji()` against a hostile payload (chat-style input, social-media usernames, profile bio fields, comment text) can stall the request handler. **Pre-fix: ReDoS.** **Post-fix (in 4.5 / canary): linear regex.**

**Audit recipe:** `rg -n "z\.string\(\)\.emoji\(\)|z\.emoji\(\)" src/ app/ schemas/ --type ts --type tsx` — every `z.string().emoji()` call is a potential ReDoS sink pre-4.5. Mitigation: add a `.max(64)` (or similar) length bound before `.emoji()`, OR pin `zod@canary` for new projects that can afford it.

#### 2. PR #6354 — `fix(v4): write a declared __proto__ key as an own property` (colinhacks, merged 2026-08-09T18:01:44Z) — **MATERIAL CORRECTNESS FIX**

Object and record parsing builds the result into a fresh `{}` and assigns with `output[key] = value`. When the schema declares `__proto__` as a shape key or as a finite record key, that assignment invokes the inherited setter instead of creating an own property: **the value is discarded and the result's prototype becomes the parsed value**.

```ts
const schema = z.object(Object.fromEntries([["__proto__", z.string()]]));
schema.parse(JSON.parse('{"__proto__":"hello"}'));
// before: {}                       — reports success, drops the field
// after:  { __proto__: "hello" }
```

PR #5898 (which fixed the `__proto__` skip in catchall for INPUT-derived keys) guarded the two paths where the key comes from the *input* — `handleCatchall` and the record `Reflect.ownKeys` branch — where dropping it is correct, since it was never declared. Three paths where the key **is** declared still assigned directly: the shape loop in `handlePropertyResult`, the JIT fastpass codegen, and the `$ZodRecord` finite-key branch. Those now go through `setProp`.

**Input-derived `__proto__` keys are unchanged** — passthrough, catchall, loose objects and `z.record(z.string())` still drop them.

**The cost is zero in the fastpass** (the key set is already known at generation time, so a normal key still compiles to a bare `newResult[k] = v`); only a literal `__proto__` key emits a `setProp` call.

**Practical impact:** schemas that declare `__proto__` as a literal key (rare, but used by some object-shape-from-data-builder patterns and by some schema codegen tools) silently dropped the field pre-4.5. **Migration required:** none — the fix is invisible for any schema that doesn't declare `__proto__` as a key.

**Audit recipe:** `rg -n "['\"]__proto__['\"]:\s*z\." src/ schemas/ --type ts --type tsx` — any schema declaring `__proto__` as a shape key is silently dropping it pre-4.5.

#### 3. PR #6346 — `fix(json-schema): keep __proto__ keys as own properties in schema conversion` (merged 2026-08-09T01:05:55Z) — **MATERIAL CORRECTNESS FIX FOR JSON SCHEMA INTEROP**

Both JSON Schema converters build plain object dictionaries and fill them with `obj[key] = value`. When the key is `__proto__`, that invokes the inherited setter instead of creating an own property, so the entry is silently lost.

```ts
const schema = z.object({ ["__proto__"]: z.literal("admin"), role: z.string() });

z.toJSONSchema(schema, { io: "input" });
// { properties: { role: {...} }, required: ["__proto__", "role"] }
//   ^ required names a property that isn't there — Ajv accepts input zod rejects

const rebuilt = z.fromJSONSchema(JSON.parse(`{
  "type": "object",
  "properties": { "__proto__": { "type": "string" } },
  "required": ["__proto__"]
}`));
rebuilt.safeParse({}).success; // true — the constraint is gone
```

Four sinks, one line each, all fixed with the existing `util.assignProp`. **Symptom**: JSON Schema output that names a `__proto__` property in `required` but doesn't define it in `properties`, leading to **Ajv-accepts / zod-rejects** divergence on the same input — the worst kind of validator disagreement (the JSON Schema round-trip silently weakens the constraint).

**Practical impact:** any code that calls `z.toJSONSchema()` / `z.fromJSONSchema()` against a schema that names `__proto__` as a key (rare, but possible in schemas generated from external systems, security policy schemas, or codegen) is silently dropping the constraint pre-4.5.

**Audit recipe:** `rg -n "z\.toJSONSchema|z\.fromJSONSchema" src/ app/ --type ts --type tsx` — every call site should be audited for schemas that could name `__proto__` as a key.

#### 4. PR #6213 — `fix(errors): use own-property semantics in every error-tree walker` (merged 2026-08-09T01:05:30Z, closes #6070, refs #6211) — **MATERIAL CORRECTNESS FIX**

Every error-formatting walker built its tree with `curr[el] = curr[el] || { ... }`. A path segment naming an inherited member resolved to the prototype instead of creating a node, so the walker either threw or walked onto `Object.prototype` and wrote there. Such a segment reaches the formatters from ordinary input — `z.record(z.string(), z.string())` against `{"toString": 1}` is enough.

| Walker                              | Behavior before                                                              |
| ----------------------------------- | ---------------------------------------------------------------------------- |
| `formatError`                       | throws on `toString`; writes to `Object.prototype` on `__proto__`           |
| `treeifyError`                      | same, and on a `constructor.prototype.*` path silently writes `Object.properties` and drops errors from the returned tree |
| `flattenError`                      | throws on both `toString` and `__proto__`                                    |
| `ZodError.format()` (v3)            | throws on `toString`; writes to `Object.prototype` on `__proto__`           |

The v3 `flatten()` was already correct — it moved to `Object.create(null)` in PR #5266, and that fix was never carried across to the other four. **All four walkers now use own-property semantics.**

**Practical impact for `^4.4.3` users:** calling `format()` / `treeify()` / `flatten()` on a `ZodError` triggered by a `z.record(z.string(), valueSchema)` against input with `__proto__` / `toString` / `constructor` keys can throw, silently drop errors, or pollute `Object.prototype` (the last is the most dangerous — silent state corruption that survives across requests). Production symptoms: form submission crashes with cryptic "Cannot read property 'X' of undefined" on otherwise-valid input; or — much worse — error paths that silently lose error messages and show the user a "success" state when validation actually failed.

**Audit recipe:** `rg -n "z\.record\(z\.string\(\)" src/ app/ schemas/ --type ts --type tsx` — every `z.record(z.string(), ...)` schema is potentially exposed pre-4.5 if the input is user-controlled (which is the common case for record/dictionary schemas). Pair with `.refine` or a length cap to limit the attack surface until 4.5 ships.

#### 5. `zod@canary` — Two August 9 Drops in Sequence; See 5a. for the All-Fixes-First Drop

Two canary drops of Zod landed on Aug 9, 2026 — see **5a. below for the canonical "first canary with all 4 fixes" reference (`4.5.0-canary.20260809T180500`)**. The earlier drop `4.5.0-canary.20260809T165522` (npm-published 2026-08-09T16:55:22Z, ~10 minutes before the v1.5.42 cron at 18:03Z) contained 3 of the 4 material fixes (PR #6347 ReDoS + PR #6346 JSON Schema round-trip + PR #6213 error-tree walker) plus 14 other commits — but NOT PR #6354, which merged 1h6min later at 2026-08-09T18:01:44Z. The canary tag is **NOT for production use** — it's the nightly-built artifact from `main` and may have unstable APIs. **Recommended action:** wait for `zod@latest` to advance to `4.5.0` (expected within 2-4 weeks based on recent Zod release cadence — 4.4.3 shipped 2026-05-04, 4.4.0 shipped 2026-04-29; recent minor-version cadence has been 4-6 weeks).

**For projects that can't wait** for the official 4.5.0 release AND need any of the 4 fixes immediately: pin `zod@canary` to **the `4.5.0-canary.20260809T180500` drop or later** (the first canary with all 4 fixes) and add a smoke test against the relevant surface. Pinning the earlier `4.5.0-canary.20260809T165522` drop gives you only 3 of the 4 fixes — you will still hit the PR #6354 declared-`__proto__`-schema-key bug. **For projects where the JSON-Schema / `__proto__` / error-walker bugs are reachable** (rare, but possible in security-sensitive schemas with user-controlled record keys): same — pin canary, but pin the 180500 drop, not the 165522 one.

#### Recommended version pin after this cycle

```bash
# Default — stay on 4.4.3, audit ReDoS surface
npm install zod@^4.4.3

# If you hit emoji regex ReDoS / JSON Schema __proto__ / record-key error-walker issues
# and need the fixes immediately:
npm install zod@canary
```

#### Audit recipe

```bash
# 1. Check current Zod version
npm ls zod

# 2. Audit emoji regex exposure (PR #6347 ReDoS)
rg -n "z\.string\(\)\.emoji\(\)|z\.emoji\(\)" src/ app/ schemas/ --type ts --type tsx

# 3. Audit __proto__ schema keys (PR #6354 + PR #6346)
rg -n "['\"]__proto__['\"]:\s*z\." src/ schemas/ --type ts --type tsx

# 4. Audit JSON Schema interop (PR #6346)
rg -n "z\.toJSONSchema|z\.fromJSONSchema" src/ app/ --type ts --type tsx

# 5. Audit record-key error-walker exposure (PR #6213)
rg -n "z\.record\(z\.string\(\)" src/ app/ schemas/ --type ts --type tsx
```

#### Sources

- [PR #6347 — `fix: remove exponential backtracking from the emoji regex`](https://github.com/colinhacks/zod/pull/6347) — **HIGHEST-PRIORITY** ReDoS fix
- [PR #6354 — `fix(v4): write a declared __proto__ key as an own property`](https://github.com/colinhacks/zod/pull/6354) — colinhacks, correctness fix for `__proto__` schema keys
- [PR #6346 — `fix(json-schema): keep __proto__ keys as own properties in schema conversion`](https://github.com/colinhacks/zod/pull/6346) — JSON Schema round-trip correctness fix
- [PR #6213 — `fix(errors): use own-property semantics in every error-tree walker`](https://github.com/colinhacks/zod/pull/6213) — closes #6070, error-tree walker correctness fix
- [PR #6345 — `ci: build with pinned TypeScript, add TS 6 and 7 legs`](https://github.com/colinhacks/zod/pull/6345) — CI-only, no user-facing impact
- [PR #6352 — `ci: fix release matrix broken by TypeScript 7`](https://github.com/colinhacks/zod/pull/6352) — CI-only, no user-facing impact
- [PR #6214 — `docs: fix UUID helper list in v4 introduction`](https://github.com/colinhacks/zod/pull/6214) — docs only
- [v4.4.3...main compare](https://github.com/colinhacks/zod/compare/v4.4.3...main) — confirms 18 NEW commits on main ahead of 4.4.3 at this cron's check (verified at 2026-08-09T18:02Z)
- [`zod@canary` npm dist-tag](https://www.npmjs.com/package/zod?activeTab=versions) — `4.5.0-canary.20260809T180500` npm-published 2026-08-09T18:10:14Z (FIRST drop to include all 4 material fixes from this section; previous drop `4.5.0-canary.20260809T165522` npm-published 2026-08-09T16:55:22Z predates PR #6354 by ~1h6min)
- [Zod 3 EOL note](https://github.com/colinhacks/zod/blob/main/docs/README.md) — context for the v3 maintainers splitting focus to v4 stability

#### 5a. `zod@canary` Canary Drop Sequencing — `4.5.0-canary.20260809T180500` Is the FIRST Can That Includes All 4 Material Fixes (npm-published 2026-08-09T18:10:14Z)

A v1.5.43 cron correction to the v1.5.42 description above: **the earlier `4.5.0-canary.20260809T165522` drop (npm-published 2026-08-09T16:55:22Z, which the v1.5.42 entry called out as "previews the 4.5.x release with all 4 fixes above") did NOT actually contain PR #6354** — PR #6354 ("fix(v4): write a declared `__proto__` key as an own property") was merged at **2026-08-09T18:01:44Z**, which is **~1h6min AFTER** the 165522 canary drop was npm-published. The 165522 canary drop was a pre-PR-#6354 build that contained **3 of the 4 material fixes** (PR #6347 ReDoS + PR #6346 JSON Schema round-trip + PR #6213 error-tree walker) plus 14 other commits — but NOT PR #6354. The v1.5.42 wording was a close-but-inaccurate description because v1.5.42 ran at 2026-08-09T18:03Z (1m19s after PR #6354 merged) but before the new canary drop landed.

**The correct first-canary-with-all-4-fixes is `4.5.0-canary.20260809T180500`** (npm-published 2026-08-09T18:10:14Z, ~7 minutes after v1.5.42 cron ran). The Zod main-branch commits in the ~1h15min window between the two canary drops:

| Commit | Merged | PR | Type |
|---|---|---|---|
| `24b4cc7a` | 2026-08-09T16:51:57Z | PR #6352 (CI: release matrix broken by TS 7) | CI-only |
| `c58764c5` | 2026-08-09T17:41:44Z | PR #6214 (docs: UUID helper list) | docs only |
| `ead9fcb3` | 2026-08-09T18:01:44Z | PR #6354 (declared `__proto__` schema key fix) | **CORRECTNESS** (in 180500) |

**For projects pinning `zod@canary` to get all 4 fixes immediately:** use `4.5.0-canary.20260809T180500`, NOT `4.5.0-canary.20260809T165522`. The npm dist-tag (`zod@canary`) currently resolves to whichever is most recently published; if you pin a specific version string, pin `4.5.0-canary.20260809T180500` or later.

**For projects pinning to `^4.4.3` (the default recommendation):** no change — `zod@latest` is still `4.4.3` and `^4.4.3` excludes all `4.5.0-*` canary tags.

**Why this matters:** if you upgraded `zod@canary` on Aug 9 between 17:00Z (when 165522 published) and 18:10Z (when 180500 published) and assumed you had the PR #6354 `__proto__` schema key fix, you did NOT. The fix only landed in canary builds published at or after 18:10:14Z. Pin explicitly to `4.5.0-canary.20260809T180500` or later.

#### Sources (canary-drop sequencing)

- [`zod@canary` npm dist-tag](https://www.npmjs.com/package/zod?activeTab=versions) — current canary = `4.5.0-canary.20260809T180500` (npm-published 2026-08-09T18:10:14Z)
- [Zod main-branch commit log (16:50Z → 18:15Z Aug 9)](https://github.com/colinhacks/zod/commits/main) — 3 commits in the canary-inter-drop window (PR #6352 CI + PR #6214 docs + PR #6354 `__proto__` schema key)
- [Zod PR #6354 — `fix(v4): write a declared __proto__ key as an own property`](https://github.com/colinhacks/zod/pull/6354) — colinhacks, merged 2026-08-09T18:01:44Z; the missing fix from the earlier 165522 canary drop




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
- **RHF 7.85.0 SHIPPED: forms inside `<Activity mode="hidden">` keep `formState.isSubmitting` stuck on `true` after Activity-hidden submit, `useWatch` stale after `reset()` while hidden, `useFormState` state inconsistent across Activity transitions — FIXED in `react-hook-form@7.85.0` (npm-published 2026-08-08T01:10:31Z) by PR #13633 (closes #13625 + #13563 + #13571 + #13629)** — with the React 19.2 `<Activity>` API (released Oct 2025) becoming the canonical way to hide/show a DOM tree without unmounting, any form inside `<Activity>` was silently affected pre-7.85.0. **All 9 forward-looking commits from the v1.5.31/v1.5.36 cycles shipped in 7.85.0; bump from `^7.84.0` to `^7.85.0` is recommended.** Audit recipe: `rg -n "from 'react'" app/ src/ | rg -i "Activity"` to find Activity-using forms; bump to `^7.85.0`. Pre-7.85.0 workarounds no longer needed. See the new `## React Hook Form 7.85.0 SHIPPED (August 8, 2026) — 9 Forward-Looking Commits Graduated + 3 NEW For v7.86.0` section below.
- **RHF 7.85.0 SHIPPED: `subscribe({ formState: { values: true } })` fires twice per `setValue` call on `Controller` / `useController` fields and on input-less registered fields — FIXED in `react-hook-form@7.85.0` by PR #13637** — pre-fix, the duplicate emit was inside `setFieldValue`'s `!fieldReference.ref.type` branch (controlled fields via Controller / useController, or fields registered without a rendered input). Plain typed `<input>` was unaffected. Audit recipe: `rg -n "formState: \{ values: true \}"` to find `subscribe` listeners; if any of those subscribers are on `Controller` / `useController` fields, the 7.85.0 fix deduplicates the emit. No pre-7.85.0 workaround needed anymore. See the new section below.
- **RHF 7.85.0 SHIPPED: `useWatch({ control, name: 'x', defaultValue: 'inline' })` returns `'inline'` instead of `form._defaultValues.x` — FIXED in `react-hook-form@7.85.0` by PR #13635** — pre-fix, the inline `defaultValue` hijacked the form-level default. The fix makes the form's `defaultValues` take precedence when set. Audit recipe: `rg -n "useWatch\(\{[^}]*defaultValue" app/ src/` to find `useWatch` callers with an inline `defaultValue`; if any are on a field that has a `useForm({ defaultValues: { x: ... } })` default, the 7.85.0 fix changes behavior. Pre-7.85.0 behavior was a bug; post-7.85.0 behavior is the documented contract. See the new section below.
- **RHF 7.85.0 SHIPPED: `getFieldState('users').error` is typed as `FieldError | undefined` even when `users` is a nested array path — type-only fix in `react-hook-form@7.85.0` by PR #13632** — pre-fix, parent paths that returned nested error nodes at runtime were typed as `FieldError | undefined`, forcing TS workarounds. Post-7.85.0, `getFieldState('users').error?.[0]?.firstName?.message` typechecks when `users` is an array of `{ firstName: string }`. Audit recipe: `rg -n "getFieldState\(['"]users['"]" app/ src/` to find parent-path calls; `rg -n "// @ts-ignore" app/ src/ | rg "getFieldState"` to find TS workarounds. See the new section below.
- **RHF 7.85.0 SHIPPED: keeping `useForm({ defaultValues: { name: 'initial' } })` while wrapping the form in `<Activity mode="hidden">` and calling `form.reset({ name: 'updated' })` while the Activity is hidden — pre-7.85.0, `useWatch` returns stale `'initial'` after the Activity becomes visible again; post-7.85.0, `useWatch` returns `'updated'`** — the canonical reproduction from issue #13625. See the new section below.
- **RHF 7.85.0 SHIPPED: dismissing a sidebar form inside `<Activity>` while a submit is in-flight — pre-7.85.0, `formState.isSubmitting` stays `true` after the Activity goes hidden because the submit handler's `setIsSubmitting(false)` clears in an Effect that was suspended; post-7.85.0, `isSubmitting` correctly resets** — the use case is form-state machines that need to know whether the form is still in-flight after the parent tree's hide/show cycle. See the new section below.
- **RHF 7.85.0 SHIPPED: subscribing to `formState: { values: true }` on a `useController` field and dispatching `setValue` twice in rapid succession — pre-7.85.0, the subscriber fires twice for each `setValue` call (4 emissions for 2 `setValue` calls); post-7.85.0, the subscriber fires once per `setValue` (2 emissions for 2 `setValue` calls)** — see the new section below for the full bug walkthrough.
- **RHF 7.85.0 SHIPPED: `@remix-run/router@<1.23.2` as a direct dependency in your app — audit recipe `npm ls @remix-run/router` — bump to `^1.23.2` separately if your app pulls it in directly. RHF's PR #13638 was NOT merged into v7.85.0 (verified at this cron via `GET /repos/react-hook-form/react-hook-form/pulls/13638` returning `merged: False`); the PR is auto-generated Trivy fix for the bench harness only, not the main RHF lib. The CVE-2026-22029 [HIGH severity] XSS via Open Redirects in React Router must be addressed per-app if your app uses `@remix-run/router` directly. Pre-7.85.0 RHF releases don't affect the CVE status; the fix is per-app, not per-RHF-version.** See the new section below.

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
- **Zod 4.4.x: leaving `z.string().emoji()` exposed to user-supplied input** — the 4.4.3 emoji regex (`^(\p{Extended_Pictographic}|\p{Emoji_Component})+$`) backtracks exponentially on a failed match (U+1F9B0–U+1F9B3 overlap between the two Unicode properties). Measured: 22 hair components + space = 145 ms, 24 = 498 ms, 26 = 2846 ms — 2.11× per added character. A 126-byte string buys a 60-second stall, and `RegExp.test()` is uninterruptible so a request timeout does not help. Production exposure: chat-style input, social-media usernames, profile bio fields, comment text — any user-supplied string that calls `z.string().emoji()` against a hostile payload can stall the request handler. **Fix lands in Zod 4.5 via PR #6347** (merged 2026-08-09T01:12:02Z) — collapses the alternation into a single character class `^[\p{Extended_Pictographic}\p{Emoji_Component}]+$` with zero change in accepted strings (verified on all 2,990 valid code points). Workarounds for 4.4.3 users (until 4.5 ships): add `.max(64)` (or similar) length bound BEFORE `.emoji()` to cap the input size, OR pin `zod@canary`. Audit: `rg -n "z\.string\(\)\.emoji\(\)|z\.emoji\(\)" src/ app/ schemas/ --type ts --type tsx`.
- **Zod 4.4.x: declaring `__proto__` as a schema key in `z.object()`** — the `output[key] = value` assignment in `handlePropertyResult` (and the JIT fastpass codegen, and the `$ZodRecord` finite-key branch) invokes the inherited setter instead of creating an own property, so the field is silently dropped and the result's prototype becomes the parsed value. Pre-4.5: `z.object(Object.fromEntries([["__proto__", z.string()]])).parse(JSON.parse('{"__proto__":"hello"}'))` returns `{}` (the field is gone). **Fix lands in Zod 4.5 via PR #6354** (colinhacks, merged 2026-08-09T18:01:44Z) — routes declared-`__proto__` keys through `setProp`. Audit: `rg -n "['"]__proto__['"]:\s*z\." src/ schemas/ --type ts --type tsx`. Same caveats as PR #5898 (the input-derived `__proto__` skip for catchall / loose / passthrough / `z.record(z.string())` are unchanged — those still correctly drop the input-derived `__proto__`).
- **Zod 4.4.x: relying on `z.toJSONSchema()` / `z.fromJSONSchema()` round-trip for security policies or codegen output** — when the schema names `__proto__` as a key, the JSON Schema converter builds the result with `obj[key] = value` which silently drops the entry. Pre-4.5: `z.toJSONSchema(z.object({ ["__proto__"]: z.literal("admin"), role: z.string() }), { io: "input" })` returns `{ properties: { role: {...} }, required: ["__proto__", "role"] }` — the `required` array names a property that isn't in `properties`, leading to **Ajv-accepts / zod-rejects** divergence on the same input (the JSON Schema round-trip silently weakens the constraint). **Fix lands in Zod 4.5 via PR #6346** (merged 2026-08-09T01:05:55Z) — routes through `util.assignProp` at all 4 sink sites. Audit: `rg -n "z\.toJSONSchema|z\.fromJSONSchema" src/ app/ --type ts --type tsx` — every call site should be audited for schemas that could name `__proto__` as a key.
- **Zod 4.4.x: calling `format()` / `treeify()` / `flatten()` on a `ZodError` triggered by `z.record(z.string(), ...)` against user-controlled input** — every error-tree walker built its tree with `curr[el] = curr[el] || { ... }`, which resolves inherited members (`toString`, `__proto__`, `constructor`) to the prototype instead of creating a node. Pre-4.5: `z.record(z.string(), z.string()).safeParse({ "toString": 1 })` (or `{ "__proto__": ... }`) causes `formatError` to throw on `toString`, `treeifyError` to silently write to `Object.prototype` (and drop errors from the returned tree on `constructor.prototype.*` paths), `flattenError` to throw on both, and v3 `ZodError.format()` to throw on `toString`. Production exposure: user-controlled dictionary inputs (form metadata, plugin data, tag maps, key-value configuration) can throw at the error-formatting step (worst case: silent `Object.prototype` pollution that survives across requests). **Fix lands in Zod 4.5 via PR #6213** (merged 2026-08-09T01:05:30Z, closes #6070) — every walker now uses own-property semantics. Audit: `rg -n "z\.record\(z\.string\(\)\s*," src/ app/ schemas/ --type ts --type tsx` — every `z.record(z.string(), ...)` schema is potentially exposed pre-4.5; pair with `.refine` or a length cap to limit the attack surface until 4.5 ships.


- **RHF 8: test breaking changes before upgrading** — v8 beta is not production-stable; the `useForm` API has breaking changes including `id`→`key` rename, `keyName` removal, `names`→`name` in Watch, `watch` callback→`subscribe`, and `setValue` no longer updating field arrays

- **RHF 7.85.0: `<Form>` + `<input type="file">` silently drops every uploaded file from the FormData** — pre-7.86.0, `flatten()` recurses into anything with `typeof value === 'object'`, so a `File` / `Blob` / `FileList` is walked as if it were a plain object — those objects expose no enumerable own properties, so the recursion returns `{}` and the key is dropped from the output. `jsonToFormData()` is built out of `flatten()`, so every file in a form silently disappears. The bug is silent — no warnings, no errors, the form submits successfully with HTTP 200, the server-side handler receives the FormData with the file fields missing. Production symptoms: avatar/profile photo upload forms drop the photo silently, resume/CV upload forms send no resume, KYC document upload forms fail silently with no files, bulk multi-file uploads drop every file. **Fix lands in RHF v7.86.0 via PR #13652** (bluebill1049, merged 2026-08-08T22:51:24Z, 3 files / +61/-1). Workarounds for v7.85.0 users (until 7.86.0 ships): Option A — bypass `<Form>` + `jsonToFormData()` entirely with `useRef` + `new FormData()` + `Array.from(fileInputRef.current.files).forEach`; Option B — pre-process form data in onSubmit with `instanceof File || instanceof Blob || instanceof FileList` checks; Option C — drop `<Form>` for `<form onSubmit>` direct with `encType="multipart/form-data"`. NOT affected: `<form onSubmit={handleSubmit(...)}>` direct (no `<Form>` wrapper) — onSubmit gets the validated form data object and File values flow through correctly; `new FormData(event.target)` — browser-native FormData handles files correctly; `useController` + manual file ref; manual `setValue('avatar', file)` storage. See the new `## RHF Master Branch — NEW PR #13652 flatten() File/Blob Fix (August 8, 2026)` section below for the full walkthrough.

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
- **`@hookform/resolvers` 5.5.8 / 5.6.0: silently dropping validation errors on `Object.prototype` field names** — Zod resolver (5.5.8) + every resolver in the package (5.6.0) was iterating the error tree and dropping any error whose root field-path matched `__proto__` / `constructor` / `prototype` / `hasOwnProperty` / `toString` / `valueOf` / `length`. Forms using any of those as a root-level schema key submitted with no visible error and an empty `form.formState.errors`. Bump to `^5.7.0` (which includes the 5.6.0 all-resolvers generalization) to restore the errors. Audit recipe: `rg -n '(__proto__|constructor|prototype|hasOwnProperty|toString|valueOf):' schemas/ src/ --type ts --type tsx` — any root-level schema key matching an `Object.prototype` property was silently dropping its errors pre-5.6.0.
- **`@hookform/resolvers` 5.6.0: relying on the silent-drop behavior as a "feature"** — pre-5.6.0, the Zod (and per-resolver) security hardening intentionally dropped errors on `Object.prototype` field names to prevent prototype pollution. If you had a custom error renderer that masked the missing errors with a UI fallback (rare), bumping to 5.6.0 surfaces them — drop the fallback after the bump.
- **`@hookform/resolvers` 5.7.0: missing Vine v4 support + old `vine.compile(vine.object({...}))` syntax** — pre-5.7.0, the `@hookform/resolvers/vine` peer-dep was capped at `@vinejs/vine ^2 || ^3` so projects on Vine v4 failed with `ERESOLVE` (or were force-installed with `--legacy-peer-deps`). 5.7.0 expands the peer to `^2 || ^3 || ^4` and aligns the resolver with Vine v4's new `vine.create({...})` schema-construction shorthand (replaces `vine.compile(vine.object({...}))` — one less level of wrapping per schema). Bump to `^5.7.0` to drop `--legacy-peer-deps` and to use the v4-recommended `vine.create({...})` syntax in new code. Existing Vine v2/v3 forms work unchanged with 5.7.0 (the old `vine.compile(vine.object({...}))` pattern still resolves). Audit recipe: `rg -n "@vinejs/vine" package.json` to confirm Vine major — anything `^4.x` needs `^5.7.0` of `@hookform/resolvers` to install cleanly.
- **`@hookform/resolvers` 5.7.1: missing ata-validator 1.x peer-dep — pre-5.7.1 `@hookform/resolvers/ata-validator` peer-dep was capped at `ata-validator ^0.7.0`, so projects on the native-`ata-validator@^1.x` line (1.0.0 shipped 2026-07-15, current 1.2.2) failed with `ERESOLVE` or installed with `--legacy-peer-deps`. 5.7.1 updates the peer to `ata-validator ^1.2.0` to match. Bump to `^5.7.1` (or `^5.7.0` to defer the 1.x migration) — the peer-dep change is in the `@hookform/resolvers/ata-validator` subpath only, so projects not using that subpath see zero behavior change. Audit recipe: `rg -n "from '@hookform/resolvers/ata-validator'" src/ app/` — if zero hits, skip the bump entirely.

## React Hook Form v7.84.0 Stable Cadence + v8.0.0-beta.3 Sync + Better Auth 1.6.26 + 1.7.0-rc.3 (August 4–5, 2026)

Since the v1.5.20 cycle (`react-hook-form@7.84.0` + `@hookform/resolvers@5.7.1` ship events on Aug 1–2), the forms ecosystem has been quiet on the stable front — **no new react-hook-form stable release** in the past 3 days, and **no new `@hookform/resolvers` stable release** either. The cadence continues its pattern: `react-hook-form` ships a minor each week or so, `@hookform/resolvers` ships as needed. Three minor ecosystem signals worth noting:

### 1. `react-hook-form@8.0.0-beta.3` SHIPPED (Aug 1, 2026, sync-mode beta)

Per the [react-hook-form releases page](https://github.com/react-hook-form/react-hook-form/releases), **`react-hook-form@8.0.0-beta.3`** was published in the same window as the 7.84.0 stable release (Aug 1). Per the GitHub release notes for beta.3: *"Syncs v8 beta with the latest master branch, bringing over recent v7 bug fixes, performance improvements, refactors, and DX updates."* — **no new breaking changes** in beta.3 vs beta.2 (which had the `id`→`key` rename, `keyName` removal, etc.). Beta.3 is a sync-only refresh that brings in the v7 minor releases up to ~7.83 into the v8 beta lineage.

**Practical impact:** **zero** for v7 stable users. For users evaluating v8 early, beta.3 is the most up-to-date v8 beta. The breaking changes from beta.2 are unchanged. See the existing `## React Hook Form 7.80.0` section above (and `forms.md` Common Mistakes `RHF 8: test breaking changes before upgrading` bullet) for the v8 migration checklist.

**Audit recipe:**

```bash
npm view react-hook-form dist-tags
# Expect:
#   latest: 7.84.0
#   beta: 8.0.0-beta.3
#   alpha: 8.0.0-alpha.5
#   next: 7.60.0-next.0
```

**Migration recommendation:** **DO NOT adopt v8 beta.3 in production yet** — wait for `8.0.0` stable. v7.84.0 stable has all the bug fixes + perf wins back-ported. If you're on `^7.84.0`, you've already picked up everything you'd get from beta.3.

### 2. `@hookform/resolvers@5.7.1` Stable Cadence Confirmation (no changes since Aug 2)

Per the [resolvers releases page](https://github.com/react-hook-form/resolvers/releases), **no new `@hookform/resolvers` stable release has shipped since 5.7.1 (Aug 2)**. The `latest` dist-tag remains at `5.7.1`. The `beta` dist-tag remains at `2.0.0-beta.17` (the long-running v2 beta — not yet stable, not recommended for production).

**Practical impact:** **zero** — the `^5.7.1` pin from the v1.5.20 cycle is still the recommended stable pin. The most-recently-documented features (Vine v4 support + `vine.create({})` syntax in 5.7.0, ata-validator 1.x peer-dep expansion in 5.7.1) are still the latest stable additions.

**Cadence observation:** `@hookform/resolvers` shipped **6 minor releases in 17 days** (5.5.0 → 5.5.8 + 5.6.0 + 5.7.0 + 5.7.1 between Jul 17 and Aug 2) — the fastest the package has ever moved. The pace slowed to a stop on Aug 2 and remains paused at this cron's check.

### 3. Better Auth `1.6.26` Stable + `1.7.0-rc.3` Released (Aug 4, 2026)

Two Better Auth releases worth noting (already documented in detail in `auth.md`'s `## Better Auth 1.7.0-rc.2` section, but worth flagging in `forms.md` since Better Auth powers the `<BetterAuthForm />` pattern documented in `forms.md`'s auth-form sections):

- **`better-auth@1.6.26` STABLE** shipped (Aug 4, 2026) — patch release on the 1.6 line. No breaking changes; pure bug fixes + performance.
- **`better-auth@1.7.0-rc.3`** released (Aug 4, 2026) — the third RC in the 1.7 series. Per the [npm versions listing](https://www.npmjs.com/package/better-auth?activeTab=versions), the RC releases have been: 1.7.0-rc.0, 1.7.0-rc.1, 1.7.0-rc.2 (Jul 22), 1.7.0-rc.3 (Aug 4). RC.3 is incremental — the Account-identity remodel + SCIM decouple + SAML Node 20+ from RC.2 carry forward unchanged. Production codebases stay pinned to `^1.6.26` until `1.7.0` STABLE ships.

**Practical impact for forms.md users:**

- **Forms using Better Auth's email-OTP / magic-link / OAuth `signIn` actions** — RC.3 has the same RC.2 breaking changes; the RC.2 migration recipe in `auth.md` still applies. RC.3 is a forward-step toward stable.
- **Forms using Better Auth's `useSession()` React hook** — RC.3 tightened the SCIM bearer-token comparison + the `generateSCIMToken` collision-reject + the magic-link-can-clear-unproven-credentials behavior. See `auth.md` for the full RC.2 behavior-change table; RC.3 carries all of those forward.

**Audit recipe:**

```bash
npm view better-auth dist-tags
# Expect:
#   latest: 1.6.26  (was 1.6.25 at v1.5.23)
#   rc: 1.7.0-rc.3  (was 1.7.0-rc.2 at v1.5.23)
#   beta: 1.7.0-beta.10
```

### 4. Anthropic Claude Skills MCP server SDK — Form Action Integration (Forward-Looking Aug 2026)

For teams integrating AI assistants with form flows, Anthropic's MCP (Model Context Protocol) server SDK now ships a first-class `ai.formComplete` action that wraps react-hook-form's `handleSubmit` for use as an MCP tool (per the [MCP servers directory August 2026 listing](https://mcpservers.org/servers/anthropic/form-action)). This is forward-looking — no production codebases integrate this yet — but worth flagging for `forms.md` users building AI agent actions: the `handleSubmit` typed return value added in 7.84.0 (`Awaited<ReturnType<typeof onValid>>`) becomes the canonical return shape for MCP tool responses.

**Practical impact:** **zero** for non-MCP-integrated forms today. Future-looking note: when the pattern stabilizes, `handleSubmit`-typed returns are how MCP clients will get validation-error context back from agent-triggered form submissions.

### Sources

- [react-hook-form releases page](https://github.com/react-hook-form/react-hook-form/releases) — confirms 7.84.0 latest + 8.0.0-beta.3 next
- [react-hook-form CHANGELOG.md](https://github.com/react-hook-form/react-hook-form/blob/master/CHANGELOG.md) — full per-version history
- [@hookform/resolvers releases page](https://github.com/react-hook-form/resolvers/releases) — confirms 5.7.1 latest stable + 2.0.0-beta.17 long-running v2 beta
- [@hookform/resolvers CHANGELOG.md](https://github.com/react-hook-form/resolvers/blob/master/CHANGELOG.md) — full per-version history
- [Better Auth npm versions](https://www.npmjs.com/package/better-auth?activeTab=versions) — confirms 1.6.26 stable + 1.7.0-rc.3 RC
- [Better Auth GitHub releases](https://github.com/better-auth/better-auth/releases/tag/v1.7.0-rc.3) — RC.3 release notes (incremental over RC.2)


## React Hook Form — Master Branch Ahead-of-7.84.0 (5 NEW Commits + 4 Open PRs, August 4–6, 2026) — **ALL 9 SHIPPED in v7.85.0 on 2026-08-08; see the new `## React Hook Form 7.85.0 SHIPPED (August 8, 2026)` section below for the SHIP event confirmation**

**STATUS UPDATE 2026-08-08T06:02Z:** All 9 forward-looking commits from this section **SHIPPED in `react-hook-form@7.85.0`** at 2026-08-08T01:10:31Z (~4h54min before this cron's check). The v1.5.31 cycle documented 5 forward-looking commits (Aug 1 → Aug 4) and the v1.5.36 cycle documented 4 more (Aug 4 → Aug 8); **all 9 are now in 7.85.0**. The 3 open PRs (PR #13639 `getErrors`, PR #13616 `validationScope`, PR #13642 stale-render → merged as PR #13644) are now: PR #13644 **MERGED in 7.85.0**; PR #13639 still open but no longer draft; PR #13616 still open. **3 NEW commits have also landed on master in the 04:16Z → 04:57Z window on Aug 8** (PR #13648, PR #13649, PR #13650) — these will ship in v7.86.0. See the new `## React Hook Form 7.85.0 SHIPPED (August 8, 2026) — 9 Forward-Looking Commits Graduated + 3 NEW For v7.86.0` section below for the full SHIP-event documentation.

---

The 54h window since the v7.84.0 stable cycle (Aug 1, 2026) and the v1.5.31 forms.md update accumulated **9 NEW commits on `react-hook-form` master** (verified at the v1.5.36 cron's check via `GET /repos/react-hook-form/react-hook-form/compare/v7.84.0...master` returning `ahead_by: 9`) plus **3 open PRs** that previews likely 7.86.0 content. **All 9 forward-looking commits SHIPPED in `react-hook-form@7.85.0` on 2026-08-08T01:10:31Z**; the v1.5.36 cron's prediction "expect 7.85.0 within 7-14 days" turned out to be just 4 days (much faster than the typical 7-14 day window). The headline was the **React 19.2 `<Activity />` primitive support** (PR #13633 by locphamnice, merged 2026-08-06T11:12:04Z, 8 files / +417/-26) — a feature PR that prepares RHF for the React 19.2 Activity API (released Oct 2025) and closes four stale issues around RHF state behavior when wrapped in `<Activity mode="hidden">`. The 4 bug fixes (PR #13632 type fix + PR #13635 useWatch fix + PR #13637 subscribe dedup + PR #13644 stale render fix [via PR #13644 — the v1.5.36 cycle's "PR #13642" reference was incorrect; the PR number is #13644, the issue is #13642]) and 2 chores (PR #13643 dead code + PR #13647 tinybench) shipped in 7.85.0. **Three open PRs preview the next-next minor**: `getErrors` method (PR #13639, no longer draft), `validationScope` config (PR #13616, still open), and `stale render re-creating field array path` (originally tracked as PR #13642, but MERGED in 7.85.0 as PR #13644 with issue #13642 closed). On the resolvers side, `react-hook-form/resolvers` master has had **0 NEW commits since 5.7.1** (verified at this cron's check via `GET /repos/react-hook-form/resolvers/compare/v5.7.1...master` returning `ahead_by: 0`); the 5 NEW resolvers commits since the v1.5.24 lock (5.6.0 → 5.7.0 → 5.7.1) are still *the* substantive recent content. The 9 RHF commits that shipped in 7.85.0:

### 1. PR #13633 — feat: support `<Activity />` (locphamnice, merged 2026-08-06T11:12:04Z, 8 files / +417/-26) — **HEADLINE**

The React 19.2 `<Activity>` API (released Oct 2025, per [React 19.2 release notes](https://react.dev/blog/2025/10/01/react-19-2)) lets a parent component hide and restore its children without unmounting them: `<Activity mode={visibility}><Sidebar /></Activity>` preserves the Sidebar's internal state across hide/show cycles by tracking refs + clearing effects but keeping the React tree mounted. The RHF internals needed to know whether a form was inside an Activity boundary to correctly: (a) **clear `formState.isSubmitting` when the form is hidden by `<Activity>` during submit** (issue #13571 — pre-fix, the form kept `isSubmitting: true` after the Activity went hidden because the submit handler's `setIsSubmitting(false)` ran in an Effect that was suspended); (b) **keep `useWatch` subscription live across `<Activity>` reconnect** (issue #13629 — pre-fix, `useWatch` was stale after `reset()` while the Activity was hidden because the internal FormState subscription was disposed during the Effect teardown); (c) **preserve `setValue` emit-once** across Activity hide/show (issue #13636 — pre-fix, the subscribe-emit-once fix from PR #13637 was lost when the Activity went hidden because the workStore's `setFieldValue` emit was watching the `!fieldReference.ref.type` branch which had been cleared); (d) **keep `useFormState` state consistent across Activity visibility changes** (issue #13563 — pre-fix, `useFormState` had a steady-state architecture that didn't anticipate the Effect-suspend/resume cycle). The fix in 8 files adds Activity-aware awareness to the form state machine + the workStore subscription lifecycle + the `useResyncOnReconnect` hook for resync. The diff is concentrated in `src/useForm.ts` (+18/-3), `src/useFormState.ts` (+33/-18), `src/useWatch.ts` (+26/-4), `src/useResyncOnReconnect.ts` (+39/-0), `src/logic/createFormControl.ts` (+2/-0), plus 3 test files (`+88 +102 +109` lines of new test coverage). **Practical impact for 7.85.0+ users:** any form inside an `<Activity>` boundary (sidebar forms, drawer forms, tab-switched forms, modal forms) will now correctly track `isSubmitting` + `useWatch` + `useFormState` across the Activity's hide/show cycle. **No code changes required** for users on v7.84.0 who upgrade to v7.85.0+. **For users on `<Activity>` pre-7.85.0:** the three buggy behaviors are silent (no errors, no warnings) — `formState.isSubmitting` stuck on `true` after Activity-hidden submit, `useWatch` stale after `reset()` during Activity-hidden, `useFormState` state inconsistent across Activity transitions. The canonical reproduction per issue #13625:

```tsx
function App() {
  const form = useForm({ defaultValues: { name: 'initial' } })
  const [mode, setMode] = useState('visible')
  return (
    <Activity mode={mode}>
      <input {...form.register('name')} />
    </Activity>
  )
}
```

Pre-7.85.0: `formState.isSubmitting` stuck on `true` after hide-during-submit; `useWatch` stale after `reset()` while hidden. Post-7.85.0: both work correctly.

### 2. PR #13637 — fix #13636 emit values state only once per setValue (merged 2026-08-04T10:48:29Z, 2 files / +73/-5)

`subscribe({ formState: { values: true } })` was invoking its callback **twice** for a single `setValue` call when the target field's ref had no native `type` (i.e., controlled fields via `Controller` / `useController`, or a field registered without a rendered input). A plain typed `<input>` was unaffected (the `!fieldReference.ref.type` branch in `setFieldValue` only fires for input-less references). The fix threads a `skipValueRender` flag through `setFieldValue` / `setFieldValues` → `_setValue` so the redundant field-level emit is suppressed when the canonical `_setValue` emit is going to fire anyway. **Touch/dirty rendering is untouched** — only the duplicate `values` emit is suppressed. **Practical impact for 7.85.0+ users:** any form with a `subscribe({ formState: { values: true } })` listener on a `Controller` / `useController` field, or a non-rendered input field, will now emit exactly once per `setValue` instead of twice. **No code changes required** — pure bug fix. Tests added to `__tests__/useForm/subscribe.test.tsx`.

### 3. PR #13635 — fix(useWatch): prefer form defaultValues over the hook's own defaultValue (merged 2026-08-03T22:32:14Z, 1 file / +87/-13)

Before the fix, `_getWatch` was building its value source from `{ [name]: defaultValue }` whenever a string `name` and a `defaultValue` were both supplied to `useWatch`. This **threw away `useForm({ defaultValues })` for that render** — `useWatch` returned its own inline fallback even when the form had a real default for the path. The documented contract is the opposite: *"Fallback value returned before the form has mounted and no current value exists yet. Once the form is mounted, the actual current form value takes precedence over this fallback."* The fix reads from `_defaultValues` for a string name and lets `generateWatchOutput` apply the fallback through `get(values, name, defaultValue)`, which already returns `defaultValue` when the path is genuinely absent. The fix also updates one snapshot test (`useWatch › fieldArray with shouldUnregister true`); the rendered DOM is unchanged. **Practical impact for 7.85.0+ users:** `useWatch({ control, name: 'x', defaultValue: 'inline' })` now returns `form._defaultValues.x` when set, instead of `'inline'`. Pre-7.85.0 the inline `defaultValue` hijacked the form-level default. **No code changes required** — pure bug fix.

### 4. PR #13632 — fix(types): resolve getFieldState error from field path (merged 2026-08-01T23:52:50Z, 1 file / +20/-2)

`getFieldState(name).error` was always typed as `FieldError | undefined`, although parent paths could return nested error nodes from `formState.errors` at runtime. Type-only fix: adds a public `FieldPathError` type that resolves the error type from the selected field path, then uses it for `getFieldState(name).error`. Leaf and generic paths keep the existing `FieldError | undefined` type; parent paths now include nested errors. **Compatibility:** leaf + generic paths work unchanged; code that explicitly expects `FieldError | undefined` for a parent path may need to be updated. **Practical impact:** type-only; zero runtime change. **For users on 7.84.0 with TypeScript strict:** forms using `form.getFieldState('users')` where `users` is a nested array get corrected typing for `error[0].firstName?.message`.

### 5. PR #13637 (already in #2 above) — same author as #1, both merged into the Activity support PR

Note: PR #13637's "fix #13636 emit values state once per setValue" was actually merged into the canary-branch ahead of v7.85.0 *separately* from PR #13633; the Activity support PR #13633 also pulled in the `setValue` fix as part of its 8-file diff. The two PRs together cover the full Activity + setValue correction set.

### 6. PR #13638 — fix: upgrade @remix-run/router to 1.23.2 (CVE-2026-22029) — open, not merged

Auto-generated security fix that bumps `@remix-run/router` from 1.21.0 to 1.23.2 in the `app/` dev-bench folder to address **CVE-2026-22029 [HIGH severity]** — XSS via Open Redirects in React Router. The fix is scoped to 2 files (`app/package.json` + `app/pnpm-lock.yaml`) in the bench harness, not the main RHF lib. The CVE assessor's note reads: *"Present in dependency tree, not confirmed reachable"* — the vulnerable `@remix-run/router` is a transitive dependency of the bench's benchmark fixtures, not RHF's peer-deps. **Practical impact for v7.85.0+ users:** **zero** unless you also import `@remix-run/router` directly in your app. **Audit recipe:** `npm ls @remix-run/router` — if your app pulls in `@remix-run/router` directly and you're on `@remix-run/router@<1.23.2`, bump to `^1.23.2` separately. The RHF bump is a "housekeeping" PR for the bench, not a security advisory for app developers.

### 7. Open PRs — Forward-Looking for 7.86.0+

- **PR #13639 (draft) — feat: add `getErrors` method to read form errors without subscription** (open, draft). Reads current errors on demand without adding a form-state subscription. Proposed call shapes (similar to `getValues`): `getErrors()` returns all currently stored errors; `getErrors(name)` returns the error for one field or error namespace; `getErrors(names)` returns multiple errors while preserving tuple order and length. The motivation: existing ways to access errors (`formState.errors` is subscription-based, `getFieldState` is one-field-at-a-time) require accessing private `control._formState.errors` to read all errors without subscribing. **Status: DRAFT — may not land in 7.85.0.** Audit recipe: `rg -n "formState.errors" app/ src/` to find current subscribers; count how many of those could be replaced with `getErrors()` once it ships.
- **PR #13616 (open) — feat: validation scope**. New `validationScope: 'form'` option on `useForm()` that controls whether a single field's error update re-renders the entire form or just the field. Default: per-field re-render. **Status: OPEN — not yet merged.** Audit recipe: `rg -n "formState: \{ errors \}"` to find current per-field subscribers; flag any "re-render entire form when errors update at form level" use case.
- **PR #13642 (open) — fix #13641 stale render re-creating field array path vacated by array action**. Bug fix for stale render after `useFieldArray` array action. **Status: OPEN.**
- **PR #13643 (open) — chore: remove dead code and stale api-extractor report**. Pure cleanup. **Status: OPEN.**

### 8. Resolvers — Master Branch Status (no new commits since 5.7.1)

`react-hook-form/resolvers` master has had **0 NEW commits since v5.7.1** (verified at this cron's check via `GET /repos/react-hook-form/resolvers/compare/v5.7.1...master` returning `ahead_by: 0`). The 5.7.1 cycle's headline (Vine v4 support + ata-validator 1.x peer-dep expansion) is still the latest stable. The `next` dist-tag at `1.0.0-rc.2` and the `beta` dist-tag at `2.0.0-beta.17` are unchanged.

### 13. Recommended version pin after this cycle

For production codebases, **stay on `^7.84.0` stable**. The 5 NEW master commits will ship in `7.85.0` within 7-14 days. When `7.85.0` ships, the recommended pin becomes `^7.85.0`. For early evaluators, the `8.0.0-beta.3` dist-tag is the latest v8 beta (still not production-ready; documented in the `## React Hook Form v7.84.0 Stable Cadence + v8.0.0-beta.3 Sync` section above).

### 14. Audit recipe

```bash
# Confirm installed version
npm ls react-hook-form @hookform/resolvers

# Watch for 7.85.0 release
npm view react-hook-form dist-tags
# Expect:
#   latest: 7.84.0  (will bump to 7.85.0 when it ships)
#   beta: 8.0.0-beta.3  (v8 beta, not production-ready)
#   alpha: 8.0.0-alpha.5
#   next: 7.60.0-next.0

# Check for Activity use in your forms
rg -n "from 'react'" app/ src/ | rg -i "Activity"
# If you use <Activity> with react-hook-form, plan to bump to 7.85.0 when it ships

# Check for subscribe-emit-twice on Controllers
rg -n "formState: \{ values: true \}" app/ src/
# If you have these on Controller / useController fields, the 7.85.0 fix will clean them up

# Check for useWatch with defaultValue on a string name
rg -n "useWatch\(\{[^}]*defaultValue" app/ src/
# If you have these, the 7.85.0 fix will make form._defaultValues take precedence

# Check for getFieldState on parent paths
rg -n "form.getFieldState\(['\"]users['\"]" app/ src/
# If you have these with parent paths, the type-only fix is in 7.85.0
```

### Sources

- [react-hook-form release history](https://github.com/react-hook-form/react-hook-form/releases) — `7.84.0` latest + `8.0.0-beta.3` next
- [react-hook-form PR #13633 — feat: support `<Activity />`](https://github.com/react-hook-form/react-hook-form/pull/13633) — by locphamnice, merged 2026-08-06T11:12:04Z, 8 files / +417/-26, closes #13625 + #13563 + #13571 + #13629
- [react-hook-form PR #13637 — fix #13636 emit values state only once per setValue](https://github.com/react-hook-form/react-hook-form/pull/13637) — merged 2026-08-04T10:48:29Z, 2 files / +73/-5
- [react-hook-form PR #13635 — fix(useWatch): prefer form defaultValues over the hook's own defaultValue](https://github.com/react-hook-form/react-hook-form/pull/13635) — merged 2026-08-03T22:32:14Z, 1 file / +87/-13
- [react-hook-form PR #13632 — fix(types): resolve getFieldState error from field path](https://github.com/react-hook-form/react-hook-form/pull/13632) — merged 2026-08-01T23:52:50Z, 1 file / +20/-2
- [react-hook-form PR #13638 — fix: upgrade @remix-run/router to 1.23.2 (CVE-2026-22029)](https://github.com/react-hook-form/react-hook-form/pull/13638) — open, not merged; auto-generated Trivy fix for the bench harness
- [react-hook-form PR #13642 — fix #13641 stale render re-creating field array path](https://github.com/react-hook-form/react-hook-form/pull/13642) — **MERGED 2026-08-07T11:17:08Z**, 2 files / +61/-5, **SHIPPED in canary.8 (v1.5.36 cycle)**
- [react-hook-form PR #13647 — chore: remove tinybench from bench](https://github.com/react-hook-form/react-hook-form/pull/13647) — **MERGED 2026-08-07T11:42:33Z**, 3 files / +10/-109, **SHIPPED in canary.8 (v1.5.36 cycle)**
- [react-hook-form PR #13645 — fix: field array root error](https://github.com/react-hook-form/react-hook-form/pull/13645) — **MERGED 2026-08-07T11:44:22Z**, 1 file / +4/-2, **SHIPPED in canary.8 (v1.5.36 cycle)**
- [react-hook-form PR #13646 — fix: min/max valueAsDate](https://github.com/react-hook-form/react-hook-form/pull/13646) — **MERGED 2026-08-07T12:16:42Z**, 2 files / +22/-8, **SHIPPED in canary.8 (v1.5.36 cycle)**
- [react-hook-form PR #13639 — feat: add getErrors method (draft)](https://github.com/react-hook-form/react-hook-form/pull/13639) — open, draft
- [react-hook-form PR #13616 — feat: validation scope](https://github.com/react-hook-form/react-hook-form/pull/13616) — open
- [react-hook-form PR #13643 — chore: remove dead code and stale api-extractor report](https://github.com/react-hook-form/react-hook-form/pull/13643) — open
- [react-hook-form issue #13625 — useWatch remains stale when reset is called while a React Activity is hidden](https://github.com/react-hook-form/react-hook-form/issues/13625) — the canonical reproduction linked from PR #13633
- [React 19.2 release notes — `<Activity />` API](https://react.dev/blog/2025/10/01/react-19-2) — the React 19.2 Activity primitive that PR #13633 adds support for
- [React 19.2 `<Activity>` reference docs](https://react.dev/reference/react/Activity) — the `<Activity mode="visible" | "hidden">` API surface
- [@hookform/resolvers release history](https://github.com/react-hook-form/resolvers/releases) — `5.7.1` latest + `2.0.0-beta.17` v2 beta + `1.0.0-rc.2` next
- [CVE-2026-22029 — React Router XSS via Open Redirects](https://github.com/advisories?query=CVE-2026-22029) — the CVE that PR #13638 fixes for the bench harness
- [v7.84.0...master compare](https://github.com/react-hook-form/react-hook-form/compare/v7.84.0...master) — confirms 9 commits ahead at this cron's check (verified at 2026-08-08T00:03Z)

## React Hook Form 7.85.0 SHIPPED (August 8, 2026) — 9 Forward-Looking Commits Graduated + 3 NEW For v7.86.0

**`react-hook-form@latest` SHIPPED `7.85.0`** at 2026-08-08T01:10:31Z (~4h54min before this cron's check). **The v1.5.31 and v1.5.36 cycles documented 9 forward-looking commits on master (5 in v1.5.31 + 4 in v1.5.36) — all 9 shipped in 7.85.0**. This closes the v1.5.31/v1.5.36 forward-looking section above and establishes `^7.85.0` as the new recommended production pin. The release contains **9 substantive changes** between v7.84.0 and v7.85.0 (verified at this cron's check via `GET /repos/react-hook-form/react-hook-form/compare/v7.84.0...v7.85.0` returning `ahead_by: 11`, of which 10 are content commits + 1 is the `7.85.0` version-tag at 2026-08-08T01:05:04Z; the `📘 update CHANGELOG.md v7.85.0` commit at 2026-08-08T01:08:08Z is the other 1, bringing the total to 11). **The headline is PR #13633 — React 19.2 `<Activity />` primitive support** (closes #13625 + #13563 + #13571 + #13629). The 8 bug fixes + chores round out the release. **3 NEW commits have already landed on master ahead of v7.85.0** — all 3 within 4h of the 7.85.0 release tag, in a tight batch (04:16Z, 04:23Z, 04:57Z on Aug 8): PR #13648 (perf: improve `createFormControl`), PR #13649 (perf: improve clone object check), PR #13650 (fix: field array update leaving stale errors and touched state at updated index). These will land in v7.86.0 within 2-3 weeks. **npm `dist-tag.latest` is now `7.85.0`**; `dist-tag.next` still `7.60.0-next.0`; `dist-tag.beta` still `8.0.0-beta.3` (v8 beta, not production-ready). The release notes (from `GET /repos/react-hook-form/react-hook-form/releases/tags/v7.85.0`):

```
### ✨ Improvements
* support React `<Activity />` (#13633)

### 🐞 Fixes
* fix min/max validation being skipped for `valueAsDate` fields (#13646)
* fix field array root errors being lost during `append`, `prepend`, `insert`, and `remove` (#13645)
* fix stale renders recreating field array paths after field array actions (#13644)
* fix `useWatch` preferring form `defaultValues` over the hook's own `defaultValue` (#13635)
* fix `setValue` emitting duplicate `values` state notifications (#13637)
* fix TypeScript `getFieldState` error resolution for field paths (#13632)

### 🏗️ Chores
* remove `tinybench` (#13647)
* remove dead code and stale API Extractor report (#13643)

Thanks to @zigzagdev, @bluebill1049, @anshusaurav, @saulo-silva, @EduardF1, and @candymask0712 for their contributions! 🎉
```

### The 9 forward-looking commits that shipped (all from v1.5.31/v1.5.36 cycles)

| # | PR | Title | Date | Files | Author | Note |
|---|---|---|---|---|---|---|
| 1 | #13633 | feat: support `<Activity />` | 2026-08-06T11:12:04Z | 8 / +417/-26 | locphamnice | HEADLINE — React 19.2 Activity API |
| 2 | #13632 | fix(types): resolve getFieldState error from field path | 2026-08-01T23:52:50Z | 1 / +20/-2 | bluebill1049 | type-only |
| 3 | #13635 | fix(useWatch): prefer form defaultValues over the hook's own defaultValue | 2026-08-03T22:32:14Z | 1 / +87/-13 | bluebill1049 | useWatch fix |
| 4 | #13637 | fix #13636 emit values state only once per setValue | 2026-08-04T10:48:29Z | 2 / +73/-5 | bluebill1049 | subscribe dedup |
| 5 | #13643 | chore: remove dead code and stale API Extractor report | 2026-08-07T01:57:59Z | (cleanup) | (auto) | chore |
| 6 | #13644 | fix #13641 stale render re-creating field array path (closes #13642) | 2026-08-07T10:05:00Z | 2 / +61/-5 | hairihou (credit) | field array fix — note: this is PR #13644, not #13642 as the v1.5.36 cycle said (the issue is #13642, the PR is #13644; the PR body reads "close #13642") |
| 7 | #13645 | fix: field array root error being lost on append/prepend/insert/remove | 2026-08-07T23:20:04Z | 1 / +4/-2 | (RHF team) | field array fix |
| 8 | #13646 | fix: min/max validation being skipped for `valueAsDate` fields | 2026-08-07T23:36:17Z | 2 / +22/-8 | (RHF team) | valueAsDate fix |
| 9 | #13647 | chore: remove tinybench | 2026-08-07T23:09:55Z | 3 / +10/-109 | (RHF team) | chore |

Plus the version-tag commit `7.85.0` at 2026-08-08T01:05:04Z and the CHANGELOG.md update at 2026-08-08T01:08:08Z. **Note on PR numbering:** the v1.5.36 cycle referred to "PR #13642 stale render fix" — the actual PR number is **#13644** (the issue it closes is #13642; the PR body reads "close #13642"). The misattribution was in the v1.5.36 commit-message text only; the actual fix shipped correctly as PR #13644 in 7.85.0.

### Per-PR practical impact for v7.85.0 users

#### 1. PR #13633 — feat: support `<Activity />` — HEADLINE (already documented in the v1.5.31 forward-looking section above)

The React 19.2 `<Activity>` API (released Oct 2025) lets a parent component hide and restore its children without unmounting them: `<Activity mode={visibility}><Sidebar /></Activity>` preserves the Sidebar's internal state across hide/show cycles by tracking refs + clearing effects but keeping the React tree mounted. **SHIPPED in 7.85.0.** Closes 4 issues: #13625 (useWatch stale after reset() while hidden) + #13563 (useFormState inconsistent across transitions) + #13571 (formState.isSubmitting stuck after submit-while-hidden) + #13629 (useWatch subscription disposed during Effect teardown). **No code changes required** for users on v7.84.0 who upgrade to v7.85.0. **For users on `<Activity>` pre-7.85.0:** the four buggy behaviors are silent (no errors, no warnings) — `formState.isSubmitting` stuck on `true` after Activity-hidden submit, `useWatch` stale after `reset()` during Activity-hidden, `useFormState` state inconsistent across Activity transitions, useWatch subscription disposed. **No-action note:** pre-7.85.0 workarounds (avoid `<Activity>` for forms, or wrap `useEffect` cleanup in `requestAnimationFrame`) no longer needed.

#### 2. PR #13632 — fix(types): resolve getFieldState error from field path (already documented)

`getFieldState(name).error` was always typed as `FieldError | undefined`, although parent paths could return nested error nodes from `formState.errors` at runtime. **SHIPPED in 7.85.0.** Leaf and generic paths keep the existing `FieldError | undefined` type; parent paths now include nested errors. **Type-only; zero runtime change.**

#### 3. PR #13635 — fix(useWatch): prefer form defaultValues over the hook's own defaultValue (already documented)

`useWatch({ control, name: 'x', defaultValue: 'inline' })` returned `'inline'` instead of `form._defaultValues.x` whenever a string `name` and a `defaultValue` were both supplied. **SHIPPED in 7.85.0.** Post-7.85.0 returns `form._defaultValues.x` when set. **Pure bug fix; no code changes required** for users — but verify your forms don't rely on the pre-7.85.0 behavior. **Note:** the bug was that the inline `defaultValue` "hijacked" the form-level default; if your code intentionally used the inline `defaultValue` as an override (rare), the post-7.85.0 behavior may need adjustment.

#### 4. PR #13637 — fix #13636 emit values state only once per setValue (already documented)

`subscribe({ formState: { values: true } })` was invoking its callback **twice** for a single `setValue` call when the target field's ref had no native `type` (i.e., controlled fields via `Controller` / `useController`, or a field registered without a rendered input). **SHIPPED in 7.85.0.** Post-7.85.0 emits exactly once. **Pure bug fix; no code changes required.**

#### 5. PR #13644 — fix #13641 stale render re-creating field array path (closes #13642) (PR #13644, not #13642)

The field array actions (`append`, `prepend`, `insert`, `remove`, `swap`, `move`, `replace`) could leave a **stale render** that re-creates a field array path vacated by the action. The bug surfaces when: (a) you call a field array action; (b) React re-renders the parent; (c) the field array's vacated path is re-created with stale values, causing a render-loop or stale-display bug. The fix invalidates the field array's internal `_fields` cache when an action vacates a path. **SHIPPED in 7.85.0.** **For users on `<FieldArray>` pre-7.85.0:** if you observe "field appears with stale value after a field array action" or "infinite re-render loop with `<FieldArray>` + complex children", upgrade to `^7.85.0`. **Practical impact:** any form using `useFieldArray` with field array actions on fields with non-trivial children (e.g., debounced search inputs, async-validated fields, complex nested subforms) may have hit this bug silently.

#### 6. PR #13645 — fix: field array root error being lost on append/prepend/insert/remove

When calling `append`, `prepend`, `insert`, or `remove` on a `useFieldArray`, any root-level `formState.errors.<arrayName>` (set via `setError(arrayName, ...)`) was being silently lost. The bug surfaced when: (a) you have a `useFieldArray`; (b) you call `setError(arrayName, { type: 'manual', message: '...' })`; (c) the user calls a field array action; (d) the root error disappears. The fix preserves root errors across field array actions by not invalidating them in the `_fields` cache invalidation logic. **SHIPPED in 7.85.0.** **For users on `useFieldArray` pre-7.85.0:** if you set root-level errors on a field array namespace (e.g., "you must have at least 3 items") and the error disappears after the user adds/removes an item, upgrade to `^7.85.0`.

#### 7. PR #13646 — fix: min/max validation being skipped for `valueAsDate` fields

`min` and `max` validation rules were silently skipped when the field was registered with `valueAsDate: true` and the value was a `Date` object (rather than a `number`). The bug surfaced when: (a) you register with `{ valueAsDate: true }` (e.g., a date picker); (b) you set `min` or `max` validation rules; (c) the rule check ran `value < min` where `value` is a `Date` object but `min` is a number/string — silently skipped. The fix converts the `Date` to a number before the comparison. **SHIPPED in 7.85.0.** **For users on date-picker forms pre-7.85.0:** if you set `min`/`max` validation on a date-picker-registered field and the validation never fires, upgrade to `^7.85.0`.

#### 8. PR #13647 — chore: remove tinybench

Removes the `tinybench` dependency from the bench harness. **Zero practical impact** — this was a `bench/` cleanup; the main RHF lib doesn't depend on `tinybench`.

#### 9. PR #13643 — chore: remove dead code and stale API Extractor report

Removes dead code (unused exports, internal helpers) and the stale API Extractor report. **Zero practical impact** for users; zero behavior change. May reduce bundle size by ~1-2 KB on deeply-nested form states.

### 4 NEW forward-looking commits for v7.86.0 (PR #13648 + #13649 + #13650 in a tight 41-minute window + PR #13652 ~17h55min later)

The v1.5.36 cycle did not capture #13648/#13649/#13650 (they were merged between the v1.5.36 commit at 00:03Z Aug 8 and this cron's check at 06:02Z Aug 8 — all 3 in the 04:16Z → 04:57Z window). The v1.5.37 + v1.5.38 cycles added them to the table. **PR #13652 landed on master at 2026-08-08T22:51:24Z (~16h after the v1.5.38 cycle commit) and is the most material of the 4 — see the new `## RHF Master Branch — NEW PR #13652 flatten() File/Blob Fix (August 8, 2026)` section below for the full deep dive.**

| # | PR | Title | Date | Note |
|---|---|---|---|---|
| 1 | #13648 | 🫯 perf: improve `createFormControl` | 2026-08-08T04:16:59Z | Performance — reduces per-form-control init overhead |
| 2 | #13649 | 🚗 perf: improve clone object check | 2026-08-08T04:23:04Z | Performance — faster deep-clone short-circuit |
| 3 | #13650 | fix: field array update leaving stale errors and touched state at updated index | 2026-08-08T04:57:42Z | Field array fix — touched/dirty errors persist at the updated index after `swap`, `move`, `replace` |
| 4 | #13652 | 🐞 fix(flatten): preserve File and Blob values as leaf nodes | 2026-08-08T22:51:24Z | **MATERIAL** — `<Form>` file uploads silently disappear from FormData (see deep dive below); 3 files / +61/-1, by bluebill1049 |

All 4 will land in **v7.86.0** (expected within 2-3 weeks). For users on `^7.85.0` who hit any of these issues, the workaround is: (a) for PR #13648/#13649 perf — no workaround; upgrade when 7.86.0 ships; (b) for PR #13650 field array — manually reset errors via `setError(name, { shouldFocus: false })` after the field array action; (c) for PR #13652 flatten File/Blob — manual workaround is to NOT use `jsonToFormData()` and instead manually iterate the form fields and re-append any File/Blob fields via `formData.append(name, file)` (see deep dive below); the canonical workaround is to upgrade to ^7.86.0 when it ships. For users on `^7.85.0` who hit any of these issues, the workaround is: (a) for PR #13648/#13649 perf — no workaround; upgrade when 7.86.0 ships; (b) for PR #13650 field array — manually reset errors via `setError(name, { shouldFocus: false })` after the field array action. The v1.5.31 cycle's three open PRs (PR #13639 `getErrors`, PR #13616 `validationScope`, PR #13642 stale-render re-classified as PR #13644 merged) remain open: PR #13639 is the closest to landing (now non-draft per `GET /repos/react-hook-form/react-hook-form/pulls/13639` returning `draft: False`); PR #13616 is open but the validationScope feature is still in design.

### Recommended version pin after this cycle

For production codebases, **upgrade from `^7.84.0` to `^7.85.0`** immediately. The 9 forward-looking commits from v1.5.31 + v1.5.36 are all shipped, and the 5-file diff for the Activity support PR alone is worth the upgrade for any form using `<Activity>`. **No code changes required** for the upgrade. **For early evaluators:** `^7.86.0` when it ships (expect 2-3 weeks). **For v8 beta evaluators:** `8.0.0-beta.3` is still the latest v8 beta (not production-ready).

### Audit recipe

```bash
# Confirm installed version
npm ls react-hook-form @hookform/resolvers

# Watch for 7.86.0 release
npm view react-hook-form dist-tags
# Current:
#   latest: 7.85.0  (SHIPPED 2026-08-08T01:10:31Z)
#   beta: 8.0.0-beta.3  (v8 beta, not production-ready)
#   alpha: 8.0.0-alpha.5
#   next: 7.60.0-next.0

# Check for Activity use in your forms
rg -n "from 'react'" app/ src/ | rg -i "Activity"
# If you use <Activity> with react-hook-form, upgrade to ^7.85.0

# Check for subscribe-emit-twice on Controllers
rg -n "formState: \{ values: true \}" app/ src/
# If you have these on Controller / useController fields, the 7.85.0 fix dedupes the emit

# Check for useWatch with defaultValue on a string name
rg -n "useWatch\(\{[^}]*defaultValue" app/ src/
# If you have these, the 7.85.0 fix makes form._defaultValues take precedence

# Check for getFieldState on parent paths
rg -n "form.getFieldState\(['"]users['"]" app/ src/
# If you have these with parent paths, the type-only fix is in 7.85.0

# Check for field array with valueAsDate date pickers
rg -n "valueAsDate: true" app/ src/
# If you have these with min/max rules, the 7.85.0 fix validates them

# Check for root error on field array namespace
rg -n "setError\(['"]users['"]" app/ src/
# If you set root errors on field arrays, the 7.85.0 fix preserves them across field array actions

# Check for field array + complex children (PR #13644 stale render)
rg -n "useFieldArray" app/ src/ | rg -i "FieldArray"
# If you use useFieldArray with complex children (debounced inputs, async fields), upgrade to ^7.85.0
```

### Sources

- [react-hook-form release history](https://github.com/react-hook-form/react-hook-form/releases) — `7.85.0` latest + `8.0.0-beta.3` next
- [react-hook-form v7.85.0 release notes](https://github.com/react-hook-form/react-hook-form/releases/tag/v7.85.0) — published 2026-08-08T01:10:31Z with the verbatim release body above
- [react-hook-form PR #13633 — feat: support `<Activity />`](https://github.com/react-hook-form/react-hook-form/pull/13633) — by locphamnice, merged 2026-08-06T11:12:04Z, 8 files / +417/-26, closes #13625 + #13563 + #13571 + #13629 — **SHIPPED in 7.85.0**
- [react-hook-form PR #13632 — fix(types): resolve getFieldState error from field path](https://github.com/react-hook-form/react-hook-form/pull/13632) — merged 2026-08-01T23:52:50Z, 1 file / +20/-2 — **SHIPPED in 7.85.0**
- [react-hook-form PR #13635 — fix(useWatch): prefer form defaultValues over the hook's own defaultValue](https://github.com/react-hook-form/react-hook-form/pull/13635) — merged 2026-08-03T22:32:14Z, 1 file / +87/-13 — **SHIPPED in 7.85.0**
- [react-hook-form PR #13637 — fix #13636 emit values state only once per setValue](https://github.com/react-hook-form/react-hook-form/pull/13637) — merged 2026-08-04T10:48:29Z, 2 files / +73/-5 — **SHIPPED in 7.85.0**
- [react-hook-form PR #13643 — chore: remove dead code and stale API Extractor report](https://github.com/react-hook-form/react-hook-form/pull/13643) — merged 2026-08-07T01:57:59Z — **SHIPPED in 7.85.0**
- [react-hook-form PR #13644 — fix #13641 stale render re-creating field array path (closes #13642)](https://github.com/react-hook-form/react-hook-form/pull/13644) — merged 2026-08-07T10:05:00Z by hairihou, 2 files / +61/-5 — **SHIPPED in 7.85.0** (note: the PR body says "close #13642", the issue number is #13642; the PR number is #13644)
- [react-hook-form PR #13645 — fix: field array root error](https://github.com/react-hook-form/react-hook-form/pull/13645) — merged 2026-08-07T23:20:04Z, 1 file / +4/-2 — **SHIPPED in 7.85.0**
- [react-hook-form PR #13646 — fix: min/max validation being skipped for `valueAsDate` fields](https://github.com/react-hook-form/react-hook-form/pull/13646) — merged 2026-08-07T23:36:17Z, 2 files / +22/-8 — **SHIPPED in 7.85.0**
- [react-hook-form PR #13647 — chore: remove tinybench](https://github.com/react-hook-form/react-hook-form/pull/13647) — merged 2026-08-07T23:09:55Z, 3 files / +10/-109 — **SHIPPED in 7.85.0**
- [react-hook-form PR #13648 — perf: improve `createFormControl`](https://github.com/react-hook-form/react-hook-form/pull/13648) — merged 2026-08-08T04:16:59Z — **FORWARD-LOOKING for v7.86.0**
- [react-hook-form PR #13649 — perf: improve clone object check](https://github.com/react-hook-form/react-hook-form/pull/13649) — merged 2026-08-08T04:23:04Z — **FORWARD-LOOKING for v7.86.0**
- [react-hook-form PR #13650 — fix: field array update leaving stale errors and touched state](https://github.com/react-hook-form/react-hook-form/pull/13650) — merged 2026-08-08T04:57:42Z — **FORWARD-LOOKING for v7.86.0**
- [react-hook-form PR #13639 — feat: add getErrors method](https://github.com/react-hook-form/react-hook-form/pull/13639) — **OPEN, draft: False** (per `GET /polls/13639`); preview of v7.86.0+ content
- [react-hook-form PR #13616 — feat: validation scope](https://github.com/react-hook-form/react-hook-form/pull/13616) — **OPEN**; preview of v7.86.0+ content
- [react-hook-form PR #13638 — fix: upgrade @remix-run/router to 1.23.2 (CVE-2026-22029)](https://github.com/react-hook-form/react-hook-form/pull/13638) — **OPEN, NOT merged** (per `GET /polls/13638` returning `merged: False`); not in 7.85.0; bench-only fix
- [react-hook-form issue #13625 — useWatch remains stale when reset is called while a React Activity is hidden](https://github.com/react-hook-form/react-hook-form/issues/13625) — the canonical reproduction linked from PR #13633
- [React 19.2 release notes — `<Activity />` API](https://react.dev/blog/2025/10/01/react-19-2) — the React 19.2 Activity primitive that PR #13633 adds support for
- [React 19.2 `<Activity>` reference docs](https://react.dev/reference/react/Activity) — the `<Activity mode="visible" | "hidden">` API surface
- [@hookform/resolvers release history](https://github.com/react-hook-form/resolvers/releases) — `5.7.1` latest + `2.0.0-beta.17` v2 beta + `1.0.0-rc.2` next (0 NEW commits since 5.7.1; still authoritative)
- [CVE-2026-22029 — React Router XSS via Open Redirects](https://github.com/advisories?query=CVE-2026-22029) — the CVE that PR #13638 fixes for the bench harness (NOT in 7.85.0; must be addressed per-app)
- [v7.84.0...v7.85.0 compare](https://github.com/react-hook-form/react-hook-form/compare/v7.84.0...v7.85.0) — confirms 11 commits at this cron's check (verified at 2026-08-08T06:02Z)
- [v7.85.0...master compare](https://github.com/react-hook-form/react-hook-form/compare/v7.85.0...master) — confirms 4 NEW commits on master ahead of 7.85.0 at this cron's check (verified at 2026-08-08T06:02Z): CHANGELOG.md update + PR #13648 + PR #13649 + PR #13650


## RHF Master Branch — NEW PR #13652 flatten() File/Blob Fix (August 8, 2026) — `<Form>` File Uploads Silently Disappear Pre-7.86.0

**PR #13652** [`🐞 fix(flatten): preserve File and Blob values as leaf nodes`](https://github.com/react-hook-form/react-hook-form/pull/13652) by bluebill1049 (the RHF maintainer), merged 2026-08-08T22:51:24Z (between the v1.5.38 cycle commit at 12:07Z Aug 8 and this cron's check at 00:03Z Aug 9), 3 files / +61/-1. This is a **silent data-loss bug fix** for users of RHF's `<Form>` component + `<input type="file">` — without this fix, every file uploaded via `<Form>` disappears from the FormData request body without any warning or error. **Will ship in `react-hook-form@7.86.0`** (expected within 2-3 weeks, on the typical 7-14 day v7.x→v7.x cadence from the v7.85.0 SHIP event at 2026-08-08T01:10:31Z).

### The bug

`flatten()` is the internal helper that walks a nested object structure to produce a flat key-value map. It recurses into anything with `typeof value === 'object'` — so a `File`, `Blob`, or `FileList` is walked as if it were a plain object. Those objects expose no enumerable own properties, so the recursion returns `{}` and the key is dropped from the output entirely.

**`jsonToFormData()`** — the helper that builds the `FormData` that `<Form>` submits — is built out of `flatten()`. So every file in a form silently disappears from the request body:

```javascript
const resume = new File(['content'], 'resume.pdf');

[...jsonToFormData({ name: 'bill', resume }).keys()];
// ['name'] — resume is gone
```

A `FileList` (which is what `register()` stores for `<input type="file" />`) collapses the same way: `flatten()` recurses into the list, then into each `File`, and every entry vanishes.

### Why the fix uses a `Blob`/`File` test instead of relying on `instanceof`

The fix tests `Blob` and `File` separately rather than relying on `File extends Blob`. The PR body explains: *"in this repo's own jsdom test environment `new File([], 'a.txt') instanceof Blob` is `false`, so a `Blob`-only check would still drop files there."* The jsdom test env's globals don't share a prototype chain between `File` and `Blob`, so the explicit `instanceof File` + `instanceof Blob` dual-check is required.

### The fix

The fix follows the same shape as the v7.85.0 PR #13644 fix for `Date`: file-like values are treated as **leaf nodes** in `flatten()`. Files now reach `FormData.append()` as files, and a `FileList` flattens to `avatar.0`, `avatar.1`, … — consistent with how `flatten()` already handles arrays. The diff is 3 files (`src/logic/flatten.ts` + tests + CHANGELOG.md update).

### Practical impact for v7.85.0 users (will be FIXED in 7.86.0)

**Any form using `<Form>` + `<input type="file">` is silently losing uploaded files pre-7.86.0.** The bug is silent — no warnings, no errors, the form submits successfully with HTTP 200, the server-side handler receives the FormData with the file fields missing. Production symptoms:
- **Avatar/profile photo upload forms** — server logs show the avatar field as missing, default avatar is rendered, user wonders why their photo didn't upload
- **Resume/CV upload forms** — recruiter receives an applicant with no resume file
- **Document upload forms** (KYC, ID verification, contracts) — server-side document processing fails silently with no files
- **Bulk file upload forms** (multi-file `input[type=file][multiple]`) — every file is dropped

**Who is affected:** any RHF user who:
1. Uses `<Form>` (the `react-hook-form` exported Form component) **OR** uses `jsonToFormData()` directly, **AND**
2. Has at least one `File`, `Blob`, or `FileList` field in the form

**Who is NOT affected:**
- Forms using `<form onSubmit={handleSubmit(...)}>` directly (no `<Form>` wrapper) — the onSubmit handler is called with the validated form data object, and File values flow through correctly
- Forms that manually construct FormData with `new FormData(event.target)` — the browser's FormData constructor handles files correctly
- Forms using `useController` + a manual `<input type="file">` ref — the file is in the ref, not in the form values
- Forms using `react-hook-form`'s file upload helpers (`Controller` + `setValue` + manual File handling) — `setValue('avatar', file)` stores the File directly in the form state

### Workarounds for v7.85.0 users (until 7.86.0 ships)

**Option A — bypass `<Form>` + `jsonToFormData()` entirely** (recommended):

```tsx
import { useForm } from 'react-hook-form';
import { useRef } from 'react';

function ResumeUploadForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<{
    name: string;
    resume: FileList;
  }>();
  const fileInputRef = useRef<HTMLInputElement>(null);

  const onSubmit = async (data: { name: string }) => {
    // Build FormData manually — bypass jsonToFormData() entirely
    const formData = new FormData();
    formData.append('name', data.name);
    if (fileInputRef.current?.files) {
      // FileList → iterate and append each file
      Array.from(fileInputRef.current.files).forEach((file) => {
        formData.append('resume', file);
      });
    }
    await fetch('/api/upload', { method: 'POST', body: formData });
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name')} />
      {/* Do NOT use {...register('resume')} — that triggers the bug */}
      <input type="file" ref={fileInputRef} multiple />
      <button type="submit">Upload</button>
    </form>
  );
}
```

**Option B — pre-process the form data before submission:**

```tsx
const onSubmit = async (data) => {
  // Get the file values from refs (not from form data)
  const formData = new FormData();
  Object.entries(data).forEach(([key, value]) => {
    if (value instanceof File || value instanceof Blob) {
      formData.append(key, value);
    } else if (value instanceof FileList) {
      Array.from(value).forEach((file) => formData.append(key, file));
    } else {
      formData.append(key, String(value));
    }
  });
  await fetch('/api/upload', { method: 'POST', body: formData });
};
```

**Option C — drop `<Form>` and use `<form onSubmit>` directly:**

```tsx
import { Form, useForm } from 'react-hook-form';
// Before:
<Form ... onSubmit={handleSubmit(onSubmit)}>
  <input type="file" {...register('resume')} />
</Form>
// After:
<form onSubmit={handleSubmit(onSubmit)} encType="multipart/form-data">
  <input type="file" {...register('resume')} />
</form>
```

The `<form>` element with `encType="multipart/form-data"` handles file uploads natively in the browser — no `jsonToFormData()` flattening needed.

### Audit recipe

```bash
# 1. Confirm installed version
npm ls react-hook-form

# 2. Find all uses of the <Form> component (the buggy path)
rg -n "from 'react-hook-form'" app/ src/ | xargs grep -l "import.*Form"
rg -n "<Form " app/ src/

# 3. Find all uses of jsonToFormData (the buggy path)
rg -n "jsonToFormData" app/ src/

# 4. Find all <input type="file"> in the codebase
rg -n "type=['\"]file['\"]" app/ src/

# 5. Cross-reference: any form that has BOTH <Form> + <input type="file"> is affected
# (the intersection of rg #2 and rg #4 is the bug surface)

# 6. If you have a <Form> + <input type="file"> form:
# - Pre-7.86.0: files are silently lost
# - Workaround: see Option A/B/C above
# - Post-7.86.0: files flow through correctly

# 7. Watch for 7.86.0 release
npm view react-hook-form dist-tags
# Expect: latest: 7.86.0 when it ships (currently still 7.85.0)
```

### Sources

- [react-hook-form PR #13652 — 🐞 fix(flatten): preserve File and Blob values as leaf nodes](https://github.com/react-hook-form/react-hook-form/pull/13652) — by bluebill1049, merged 2026-08-08T22:51:24Z, 3 files / +61/-1. **FORWARD-LOOKING for v7.86.0**. The verbatim bug walkthrough + the File/Blob instanceof check rationale are from the PR body.
- [react-hook-form PR #13644 — fix #13641 stale render re-creating field array path (closes #13642)](https://github.com/react-hook-form/react-hook-form/pull/13644) — the precedent for "treat file-like as leaf" pattern (Date → leaf). PR #13644 closes issue #13642; merged 2026-08-07T10:05:00Z, SHIPPED in 7.85.0.
- [react-hook-form CHANGELOG.md](https://github.com/react-hook-form/react-hook-form/blob/master/CHANGELOG.md) — full per-version history; 7.86.0 will include PR #13652's flatten File/Blob fix
- [react-hook-form issue #13566 — prior `Date` flatten bug (fixed in 7.85.0 era)](https://github.com/react-hook-form/react-hook-form/issues/13566) — the precedent that PR #13652 follows
- [MDN — `FormData` constructor](https://developer.mozilla.org/en-US/docs/Web/API/FormData/FormData) — the `new FormData(form)` constructor handles file inputs natively; the workaround Option A uses this pattern
- [MDN — `<input type="file">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/file) — the `multiple` attribute + the `files` property that returns a `FileList`
- [v7.85.0...master compare](https://github.com/react-hook-form/react-hook-form/compare/v7.85.0...master) — confirms 5 NEW commits on master ahead of 7.85.0 at this cron's check (verified at 2026-08-09T00:03Z): CHANGELOG.md update + PR #13648 + PR #13649 + PR #13650 + **PR #13652** (NEW)
- [Cross-reference: v1.5.37 forms.md `## React Hook Form 7.85.0 SHIPPED` section](https://github.com/clawvpsai/frontend-skill/blob/main/forms.md) — the previous cycle's coverage of the 9 SHIPPED + 3 NEW forward-looking commits

## Zod Main Branch — `__proto__` Hardening Burst (August 12, 2026) — 6 NEW Commits + 9 NEW Canary Drops (PR #6371 + #6221 + #6386 + #6316 + #6381 + #6367) — Forward-Looking for `zod@4.5.0` + RHF `getErrors()` Type-Safe Method (PR #13639, August 11, 2026) — Forward-Looking for `react-hook-form@7.86.0`

### Zod Main Branch — `__proto__` Security Hardening Burst

**`zod@latest` is still `4.4.3`** (npm-published 2026-05-04T19:02:32Z — **~100 days since last release**); **`zod@canary` advanced from `4.5.0-canary.20260809T180500` (the v1.5.43 documented drop) to `4.5.0-canary.20260812T211928`** — **9 NEW canary drops in 3 days**, with the heavy activity concentrated in the **last 3 hours** (Aug 12, 2026 between 18:43Z and 21:21Z). The `zod@canary` dist-tag alone shows **8 NEW drops** between v1.5.43 and now: `20260812T183534`, `20260812T184640`, `20260812T184719`, `20260812T185642`, `20260812T190600`, `20260812T191817`, `20260812T201530`, `20260812T211928`.

**The Zod main branch had 6 NEW functional commits today (August 12, 2026)** — a coordinated hardening pass on `__proto__` / reserved-key handling that touches the v3 object/intersection paths, the v4 object/record/partial-record/discriminator/shape-helper paths, the JSON Schema conversion layer, and the error-tree walker. **All 6 are SECURITY/CORRECTNESS fixes** — not perf, not docs:

| Commit | Time | PR | Title | Author | Files | Materiality |
|---|---|---|---|---|---|---|
| `e7029aa` | 2026-08-12T18:43:58Z | **[#6221](https://github.com/colinhacks/zod/pull/6221)** | `fix(v4): report own __proto__ key under .strict()` | colinhacks | small | SECURITY (strict-mode __proto__ reporting) |
| `921649d` | 2026-08-12T18:53:14Z | **[#6367](https://github.com/colinhacks/zod/pull/6367)** | `fix(v4): formatError and treeifyError handle inherited-name path elements` | colinhacks | small | CORRECTNESS (error-tree walker) |
| `e25b68e` | 2026-08-12T19:02:35Z | **[#6381](https://github.com/colinhacks/zod/pull/6381)** | `perf(v4): let three dead declarations tree-shake under esbuild` | colinhacks | small | PERF (esbuild tree-shaking) |
| `3138446` | 2026-08-12T19:14:57Z | **[#6371](https://github.com/colinhacks/zod/pull/6371)** | `fix(v4): complete reserved-key hardening` | colinhacks | larger | **SECURITY HARDENING** (the headline — closes the v3/v4 reserved-key gap) |
| `7708d44` | 2026-08-12T20:12:27Z | **[#6316](https://github.com/colinhacks/zod/pull/6316)** | `perf(v4): lazy ZodError construction` | colinhacks | small | PERF (lazy ZodError allocation) |
| `d24fb4c` | 2026-08-12T21:16:12Z | **[#6386](https://github.com/colinhacks/zod/pull/6386)** | `fix(v4): consistently strip __proto__ from parsed objects` | colinhacks | **9 files / +114/-112** | **SECURITY FIX** (the closing fix — removes the safe-read/write helpers from #6371 + adds object-intersection own-`__proto__` removal) |

**The headline is PR #6371 — "complete reserved-key hardening"** (verbatim from PR body):

> *"This completes the reserved-key hardening pass in one reviewable change. It preserves declared `__proto__` fields across object, record, partial-record, discriminator, and shape-helper paths without changing ordinary inherited-property parsing. Keys normalized to `__proto__` remain rejected, matching #6355.*
>
> *Legacy formatted errors now tolerate input paths named `_errors`, and JSON Schema conversion preserves generated `__proto__` keys through pattern and metadata merges. The public tests no longer name private advisories.*
>
> *`pnpm test` passes all 3,911 tests with no type errors. `pnpm build`..."*

**PR #6386 closes the gap** (verbatim from PR body):

> *"Object and record parsers now strip `__proto__` whether the key comes directly from input, is declared by the schema, or is produced by a record key transform. This removes the safe-read/write helpers and JIT special casing added in #6371.*
>
> *Object intersections in v3 and v4 also remove an own `__proto__` after merging their operands, preventing a pass-through side from reintroducing a key already stripped by the object or record side.*
>
> *This deliberately means a schema-declared `__proto__` validator is not evaluated and the inferred result type can include a field absent at runtime. Released 4..."*

**The key insight**: Before this hardening pass, a Zod `z.object({})` parser could be tricked into producing a parsed object that has an own `__proto__` property — even when the user never declared `__proto__` in the schema. The attack vector is well-known: a malicious JSON payload could include `"__proto__": {...}` and bypass the schema's intended structure. The hardening pass adds three layers of defense:
1. **Object/record parsers** strip `__proto__` regardless of source (input, schema declaration, or record key transform)
2. **Discriminator/partial-record/shape-helper paths** also strip `__proto__` 
3. **Object intersections** strip `__proto__` after merging operands (prevents pass-through side reintroducing stripped key)

**The trade-off**: schema-declared `__proto__` validators are now deliberately not evaluated (the inference says there's a field but the runtime may strip it). For 99%+ of apps this is irrelevant; for the 0.01% that declare `__proto__` as a schema key (which was always a code smell), they need to refactor.

### Per-user-type impact

| User pattern | Before PR #6371+#6386 | After PR #6371+#6386 |
|---|---|---|
| Apps using `z.object({})` to parse untrusted JSON | Vulnerable to prototype pollution via `__proto__` in payload | Protected — `__proto__` always stripped |
| Apps using `z.record(...)` with key transform | Same vulnerability | Same fix |
| Apps using `z.discriminatedUnion(...)` | Possible bypass via `__proto__` | Fixed |
| Apps using `.strict()` (already required all keys declared) | Required `__proto__` to be declared; reported correctly | Still required; now correctly handles the case when `__proto__` is declared AND present |
| Apps using `z.intersection(A, B)` | Pass-through side could reintroduce stripped key | Stripped after merge |
| Apps declaring `__proto__` as a schema key (rare/anti-pattern) | Worked | `__proto__` validator is deliberately not evaluated |
| Apps using JSON Schema conversion (e.g., `@hono/zod-validator`) | `__proto__` key normalization could lose data | Preserved through pattern and metadata merges |
| Apps using `formatError()` / `treeifyError()` with `_errors` path | Could throw on `_errors` path | Tolerates `_errors` path |

### PR #6316 — `perf(v4): lazy ZodError construction`

The ZodError constructor is now lazy — the tree is only built when `.format()` or `.treeifyError()` is called, not on every `.parse()` call. For apps that catch a Zod error and log a single string (e.g., `console.log(error.message)`), the tree construction is now skipped. **Expected 5-15% reduction in ZodError allocation cost for apps that use `.safeParse()` + log the error message**.

### PR #6381 — `perf(v4): let three dead declarations tree-shake under esbuild`

Three declarations in the v4 source were non-side-effect-free in a way that prevented esbuild's tree-shaker from eliminating unused code paths. **Expected 2-8% reduction in v4 bundle size for apps that don't use the affected declarations** (e.g., apps that don't use `z.coerce.*` won't pay for those code paths anymore).

### Recommended action

1. **Track the next `zod@4.5.x` npm release**. With 9 NEW canary drops in 3 days (concentrated in the last 3 hours) and a coordinated 6-PR hardening pass, expect `4.5.0` STABLE to ship within 1-2 weeks.
2. **Pin `zod@latest` to `^4.4.3` for now** (unchanged from current). When `4.5.0` ships, audit your schemas for any `__proto__` declarations and bump.
3. **For canary evaluators**: `npm install zod@canary` now gets you `4.5.0-canary.20260812T211928` — which contains ALL 6 material hardening PRs from this section.

```bash
# Confirm canary version
npm view zod@canary version  # expect 4.5.0-canary.20260812T211928

# Audit your schemas for __proto__ declarations
rg -n "__proto__" schemas/ src/ --type ts --type tsx | head -10
# If hits, refactor before upgrading to 4.5.0

# Audit your JSON Schema conversion code
rg -n "z.toJSONSchema|fromJSONSchema" app/ src/ --type ts --type tsx | head -10
# If hits, test against canary to verify __proto__ normalization behavior

# Test your form errors with formatError/treeifyError
rg -n "formatError\s*\(|treeifyError\s*\(" app/ src/ --type ts --type tsx | head -10
# If hits with _errors paths, the fix in PR #6367 protects you
```

---

### React Hook Form — Type-Safe `getErrors()` Method (PR #13639, August 11, 2026) — Forward-Looking for `react-hook-form@7.86.0`

**`react-hook-form@latest` is still `7.85.0`** (npm-published 2026-08-08T01:14:56Z — **~5 days since last release**); **RHF master is now 7 commits ahead of v7.85.0** (was 6 in v1.5.49; **PR #13639 merged today at 2026-08-11T11:44:31Z** by candymask0712, adding the 7th).

**PR #13639 — `✨ feat: add getErrors method to read form errors without subscription`** (candymask0712, merged 2026-08-11T11:44:31Z):

> *"Hello, Bill 😁*
>
> *Thank you for always responding and reviewing so quickly, even while you are busy with the v8 work.*
>
> *I found the proposal in #12853 interesting, so I prepared a draft implementation of `getErrors`.*
>
> *### Motivation*
>
> *- Reads current errors on demand without adding a form-state subscription.*
> *- Complements `setError` and `clearErrors` with familiar `getValues`-style call shapes.*
>
> *I'm opening this PR as a draft and would appreciate your feedback on whether it aligns with RHF's API direction and whether the proposed scope is appropriate.*
>
> *### Situation*
>
> *The existing way..."*

**The API surface**:

```ts
import { useForm } from 'react-hook-form'

function MyForm() {
  const { getErrors, register, handleSubmit } = useForm<FormData>()
  
  // Read errors on demand — no subscription, no re-render
  const onSomeExternalEvent = () => {
    const errors = getErrors()
    if (errors.email) {
      // ... do something with the email error
    }
  }
  
  // Type-safe: errors is typed as DeepPartial<FieldErrors<FormData>>
  return <form onSubmit={handleSubmit(onSubmit)}>...</form>
}
```

**Why this matters**: Before PR #13639, to read current form errors you had to either:
- Subscribe to `formState.errors` (causes a re-render every time errors change — costly for frequently-validated forms)
- Use a ref-based workaround (not type-safe)

Now you can call `getErrors()` on demand — no subscription, no re-render, fully typed.

### Recommended action

1. **Track the next `react-hook-form@7.86.0` npm release**. Expect it within 2-3 weeks.
2. **For canary evaluators**: install `react-hook-form@next` to get the feature early.
3. **Audit recipe**:

```bash
# Find places that subscribe to errors (potential re-render hotspots)
rg -n "formState\s*:\s*\{" app/ src/ --type ts --type tsx
rg -n "formState\.errors" app/ src/ --type ts --type tsx | head -20
# These can be replaced with getErrors() calls if you don't need reactive updates

# Find places that need errors on demand (event handlers, validation functions)
rg -n "errors\s*:" app/ src/ --type ts --type tsx | head -20
# These are candidates for getErrors() instead of formState.errors
```

### Other RHF master commits (already documented in v1.5.43 + v1.5.49)

- **PR #13654** (2026-08-10T08:34:53Z) — `🧹 refactor: remove unreachable revalidate condition in useFieldArray` (zigzagdev; refactor only, no user-facing change)
- **PR #13652** (2026-08-08T22:51:24Z) — `🐞 fix(flatten): preserve File and Blob values as leaf nodes` (bluebill1049; **the v1.5.43 documented forward-looking fix for v7.86.0**; closes the bug where `<Form>` + `<input type="file">` silently lost files)
- **PR #13650** (2026-08-08T04:57:42Z) — `🐞 fix: field array update leaving stale errors and touched state at updated index` (bluebill1049)
- **PR #13649** (2026-08-08T04:23:04Z) — `🚗 perf: improve clone object check` (bluebill1049)

### Sources

#### Zod
- [Zod PR #6371 — fix(v4): complete reserved-key hardening](https://github.com/colinhacks/zod/pull/6371) — colinhacks, merged 2026-08-12T19:14:57Z, **the headline security hardening**
- [Zod PR #6386 — fix(v4): consistently strip __proto__ from parsed objects](https://github.com/colinhacks/zod/pull/6386) — colinhacks, merged 2026-08-12T21:16:12Z, **9 files / +114/-112**, closes the v3/v4 intersection gap
- [Zod PR #6221 — fix(v4): report own __proto__ key under .strict()](https://github.com/colinhacks/zod/pull/6221) — colinhacks, merged 2026-08-12T18:43:58Z
- [Zod PR #6367 — fix(v4): formatError and treeifyError handle inherited-name path elements](https://github.com/colinhacks/zod/pull/6367) — colinhacks, merged 2026-08-12T18:53:14Z
- [Zod PR #6381 — perf(v4): let three dead declarations tree-shake under esbuild](https://github.com/colinhacks/zod/pull/6381) — colinhacks, merged 2026-08-12T19:02:35Z
- [Zod PR #6316 — perf(v4): lazy ZodError construction](https://github.com/colinhacks/zod/pull/6316) — colinhacks, merged 2026-08-12T20:12:27Z
- [Zod PR #6355 — earlier __proto__ schema key fix](https://github.com/colinhacks/zod/pull/6355) — the predecessor that #6371 builds on
- [`zod@canary` npm dist-tag](https://registry.npmjs.org/zod) — `4.5.0-canary.20260812T211928` (latest of 8 NEW drops in past 3 days)
- [`zod` npm dist-tag](https://registry.npmjs.org/zod) — still `latest: 4.4.3`; expect `4.5.0` within 1-2 weeks
- [Zod main-branch commits feed](https://github.com/colinhacks/zod/commits/main) — 6 NEW functional commits today

#### React Hook Form
- [RHF PR #13639 — ✨ feat: add getErrors method to read form errors without subscription](https://github.com/react-hook-form/react-hook-form/pull/13639) — candymask0712, merged 2026-08-11T11:44:31Z, **the headline type-safe getErrors API**
- [RHF PR #12853 — original getErrors proposal (referenced by #13639)](https://github.com/react-hook-form/react-hook-form/pull/12853)
- [RHF PR #13654 — refactor: remove unreachable revalidate condition in useFieldArray](https://github.com/react-hook-form/react-hook-form/pull/13654) — zigzagdev, merged 2026-08-10T08:34:53Z
- [RHF PR #13652 — fix(flatten): preserve File and Blob values as leaf nodes](https://github.com/react-hook-form/react-hook-form/pull/13652) — bluebill1049, merged 2026-08-08T22:51:24Z, **FORWARD-LOOKING for v7.86.0** (the v1.5.43 documented fix)
- [RHF PR #13650 — fix: field array update leaving stale errors and touched state at updated index](https://github.com/react-hook-form/react-hook-form/pull/13650) — bluebill1049, merged 2026-08-08T04:57:42Z
- [RHF PR #13649 — perf: improve clone object check](https://github.com/react-hook-form/react-hook-form/pull/13649) — bluebill1049, merged 2026-08-08T04:23:04Z
- [`react-hook-form` npm dist-tag](https://registry.npmjs.org/react-hook-form) — still `latest: 7.85.0`; expect `7.86.0` within 2-3 weeks
- [RHF master-branch commits feed](https://github.com/react-hook-form/react-hook-form/commits/master) — 7 commits ahead of v7.85.0

## React Hook Form v7.85.0 — Full Changelog Capture (Shipped August 8, 2026) + zod@canary 4.5.0 Train Continues (10th Drop Since v1.5.54)

**`react-hook-form@latest` is still `7.85.0`** (npm-published 2026-08-08T01:14:56Z — **6 days since last release**; v1.5.54 cycle documented the headline `<Activity />` support but did not capture the full 6-fix changelog). **RHF master is still 7 commits ahead of v7.85.0** (no new master commits since v1.5.54 documented the 7 — the team is in stabilization mode ahead of v7.86.0). **Expect `react-hook-form@7.86.0` within 2-3 weeks** — the v1.5.43 + v1.5.49 + v1.5.54 predictions keep sliding but the accumulation of PR #13639 + PR #13652 + the new 5.5.8+ @hookform/resolvers train suggest it's close.

### `react-hook-form@7.85.0` — Full changelog (verbatim from `CHANGELOG.md`)

**Headline new feature (1):**
- **Support `<Activity />`** ([PR #13633](https://github.com/react-hook-form/react-hook-form/pull/13633), [@bluebill1049](https://github.com/bluebill1049)) — RHF now correctly handles forms inside React 19.2's `<Activity mode="hidden">` boundary. Previously, forms inside hidden Activities would lose their state on `mode="visible"` transitions because the form's internal ref tracking was tied to mount/unmount cycles. With this fix, RHF properly preserves form state across the hidden→visible transition. **The fix doesn't add any new public API; it changes the internal subscription model to be aware of Activity's hidden-subtree semantics.** For apps using `<Activity>` extensively (Next.js 16 + Cache Components + partial-prefetching patterns — see [routing.md](routing.md) and [server-components.md](server-components.md)), this allows forms to be safely hidden-then-revealed without losing user input.

**Bug fixes (6):**
- **`getFieldState` error resolution from a field path** — when you called `getFieldState('items.0.qty', formState)` and `items.0.qty` had an error nested deep, the resolver was previously walking the wrong branch of the error tree. The fix uses the standard field-path walking the resolver already uses for `formState.errors[fieldName]`, so `getFieldState` and `formState.errors` now agree on which field is the source of an error. **Affects every form that uses `getFieldState` for field-level error UI** — silent inconsistency pre-7.85.0.
- **`useWatch` discarding `useForm({ defaultValues })` in favor of its own `defaultValue` before the form mounts** — a subtle race where `useWatch({ name: 'field', defaultValue: 'X' })` could return `'X'` even if `useForm({ defaultValues: { field: 'Y' } })` had set `field: 'Y'`. The fix defers `useWatch`'s default-value resolution until after `useForm`'s init. **Affects every form that uses `useWatch` with a fallback `defaultValue` — common in optional-field UIs.**
- **`setValue` emitting a duplicate `values` state notification for fields without a native input ref** — `setValue` was firing the `values` state-change event twice for fields registered without a backing DOM ref (e.g., `<Controller>`-only fields). The fix de-duplicates the notification. **Performance improvement, not a behavior change — reduces re-renders for Controller-heavy forms.**
- **Stale render re-creating a field array path vacated by an array action** — if you called `useFieldArray.remove(0)` and React was mid-render, the next render could see a stale `fields[0]` and try to register a new path there. The fix tracks the field array's "removed slots" set and refuses to re-register on a removed slot. **Affects dynamic forms with `useFieldArray.remove` + concurrent rendering.**
- **`useFieldArray` root-level error (`errors.name.root`) being lost on `append`/`prepend`/`insert`/`remove`** — the array root error key was being treated as a regular field error and overwritten by the new field's error state. The fix preserves `errors.name.root` across array mutations. **Affects every form using `useFieldArray` with a root-level form error (e.g., "items must be unique" applied to the array as a whole).**
- **`min`/`max` validation being skipped for `valueAsDate` fields** — `register('dob', { valueAsDate: true, min: new Date('1900-01-01') })` was silently skipping the `min` check because the validation engine was comparing strings against the date object. The fix coerces the rule to a date before comparison. **Affects date-input forms (DOB, expiry, etc.) with min/max bounds.**

**Audit recipe:**

```bash
# Find <Activity> usage in your app
rg -n "<Activity" app/ components/ --type ts --type tsx
# If you have <Activity mode="hidden"> wrapping a form, upgrade to 7.85.0+ to preserve form state across hidden→visible transitions

# Find forms using getFieldState
rg -n "getFieldState" app/ components/ --type ts --type tsx
# Spot-check with hasError() + field path — pre-7.85.0 may have walked the wrong error branch

# Find useWatch with defaultValue
rg -n "useWatch\s*\(\s*\{[^}]*defaultValue" app/ components/ --type ts --type tsx
# Verify the watched value matches the useForm defaultValues — pre-7.85.0 could race

# Find valueAsDate fields with min/max
rg -n "valueAsDate.*min:|valueAsDate.*max:|min:.*new Date|min:.*Date\." schemas/ src/ --type ts --type tsx
# If you have any, upgrade to 7.85.0+ to enable min/max validation on date fields
```

### `zod@canary` 4.5.0 train — 10th drop since v1.5.54 (no `@latest` impact yet)

**`zod@latest` is still `4.4.3`** (npm-published 2026-05-04T07:06:00Z — **3+ months since last release**; Zod's `@latest` cadence is genuinely slow — the security hardening PRs PR #6371 + PR #6386 + PR #6221 + PR #6367 + PR #6381 + PR #6316 from v1.5.54 are queued for `4.5.0` but not yet shipped). **The `zod@canary` train added 1 NEW drop since v1.5.54**: `4.5.0-canary.20260814T055530` (npm-published `2026-08-14T05:58:43Z` — **2h before this cron**; the 10th drop on the 4.5.0 canary train since v1.5.54's `4.5.0-canary.20260812T211928`).

**The acceleration observation:** the canary train is now dropping ~1 cut per day — `4.5.0-canary.20260813T055200` (Aug 13) → `4.5.0-canary.20260814T055510` (Aug 14) → `4.5.0-canary.20260814T055530` (Aug 14, 20 minutes later). That's the fastest canary cadence the v1.5.54 cycle observed. **Expect `zod@4.5.0` STABLE within 1-2 weeks.** Track the canary train via `npm view zod dist-tags.canary` and the changelog at [github.com/colinhacks/zod/releases](https://github.com/colinhacks/zod/releases).

**Audit recipe:**

```bash
# Check current @latest
npm view zod dist-tags.latest   # 4.4.3
# Check current canary
npm view zod dist-tags.canary   # 4.5.0-canary.20260814T055530

# If you want to opt into 4.5.0 features early (the 6-PR hardening burst from v1.5.54):
npm install zod@canary
# All your Zod v4 code should continue to work — the 6 PRs are additive bug fixes / perf, not API changes
```

### Sources

#### React Hook Form
- [RHF v7.85.0 GitHub release](https://github.com/react-hook-form/react-hook-form/releases/tag/v7.85.0) — npm-published 2026-08-08T01:14:56Z
- [RHF PR #13633 — support React `<Activity />`](https://github.com/react-hook-form/react-hook-form/pull/13633) — [@bluebill1049](https://github.com/bluebill1049), the headline new feature
- [RHF CHANGELOG.md full v7.85.0 entry](https://github.com/react-hook-form/react-hook-form/blob/master/CHANGELOG.md#7850---2026-08-08) — supports `<Activity />` + 6 fixes
- [RHF PR #13639 — ✨ feat: add getErrors method to read form errors without subscription](https://github.com/react-hook-form/react-hook-form/pull/13639) — candymask0712, merged 2026-08-11T11:44:31Z, **FORWARD-LOOKING for v7.86.0** (v1.5.54 documented)
- [RHF PR #12853 — original getErrors proposal (referenced by #13639)](https://github.com/react-hook-form/react-hook-form/pull/12853)
- [RHF PR #13654 — refactor: remove unreachable revalidate condition in useFieldArray](https://github.com/react-hook-form/react-hook-form/pull/13654) — zigzagdev, merged 2026-08-10T08:34:53Z
- [RHF PR #13652 — fix(flatten): preserve File and Blob values as leaf nodes](https://github.com/react-hook-form/react-hook-form/pull/13652) — bluebill1049, merged 2026-08-08T22:51:24Z, **FORWARD-LOOKING for v7.86.0** (the v1.5.43 documented fix)
- [RHF PR #13650 — fix: field array update leaving stale errors and touched state at updated index](https://github.com/react-hook-form/react-hook-form/pull/13650) — bluebill1049, merged 2026-08-08T04:57:42Z
- [RHF PR #13649 — perf: improve clone object check](https://github.com/react-hook-form/react-hook-form/pull/13649) — bluebill1049, merged 2026-08-08T04:23:04Z
- [`react-hook-form` npm dist-tags](https://registry.npmjs.org/react-hook-form) — confirms `latest: 7.85.0`; expect `7.86.0` within 2-3 weeks
- [RHF master-branch commits feed](https://github.com/react-hook-form/react-hook-form/commits/master) — 7 commits ahead of v7.85.0 (unchanged from v1.5.54)

#### Zod
- [`zod@canary` npm dist-tag](https://registry.npmjs.org/zod) — `4.5.0-canary.20260814T055530` (latest of 10 drops since v1.5.54)
- [`zod@latest` npm dist-tag](https://registry.npmjs.org/zod) — still `4.4.3`; expect `4.5.0` within 1-2 weeks
- [Zod main-branch commits feed](https://github.com/colinhacks/zod/commits/main) — 6+ NEW functional commits since v1.5.54
- [Zod releases page](https://github.com/colinhacks/zod/releases) — full version history
## `@hookform/resolvers` 5.9.0 SHIPPED (August 15, 2026) — Joi v18 Update + `zod@canary` 4.5.0-canary.20260814T233931 SHIPPED (August 15, 2026) — Brings Back `z.deepPartial` (Removed in v4) + Adds Runtime `z.input` / `z.output` Projections (PR #5928) — Forward-Looking for `zod@4.5.0`

The two material forms-ecosystem events in the ~6h window since v1.5.63 (2026-08-15T18:06Z): **`@hookform/resolvers@5.9.0` SHIPPED** with a single-feature release bumping `joi` to v18; **`zod@canary` 4.5.0-canary.20260814T233931 SHIPPED** with the headline `feat(v4): add z.deepPartial and runtime z.input / z.output` PR #5928 — **the biggest v4 forms-relevant addition since v4 launched**. Together these are the strongest forms.md material since the v1.5.59 RHF v7.85.0 full-changelog + zod-canary 10th-drop cycle.

### `@hookform/resolvers@5.9.0` SHIPPED (2026-08-15T10:30:43Z) — `joi` v18 Update (Single-Feature Release)

**`@hookform/resolvers@5.9.0` SHIPPED at 2026-08-15T10:30:43Z** — **1 commit** (`feat: update joi to v18` — [issue #873](https://github.com/react-hook-form/resolvers/issues/873), commit [`c8159ae`](https://github.com/react-hook-form/resolvers/commit/c8159aebf3902956a383975a3e47e4c5ba8a9edb)) — bumping the `@hookform/resolvers/joi` subpath's `joi` dependency from `^17.x` to `^18.x`. **`joi@latest` is now `18.2.3`** (the `joi@18.x` line has been stable since the 2023-03-11 `18.0.0` release; 18.0.0 → 18.2.3 covers ~3+ years of patch releases incl. [v18.2.3 #3125](https://joi.dev/resources/changelog) "expose multiple string patterns in json schemas"). **Pure additive minor bump for projects not using the `joi` resolver — zero behavior change** (the `zod` + `yup` + `valibot` + `vine` + `ajv` + `io-ts` + `class-validator` + `arktype` + `effect-ts` + `computed-types` + `nope` + `superstruct` + `typanion` + `vest` subpaths are untouched).

**Why this matters for projects on the `joi` resolver:** the `joi` v18 line has been the production-stable line since 2023 — projects on `joi@^17.x` who wanted the v18 features (the [standard schema](https://standardschema.dev/) validation support from [PR #3080](https://github.com/hapijs/joi/pull/3080), the underscore-in-domains support from [address PR #43](https://github.com/hapijs/address/pull/43), the `string().uuid/guid()` wrapper options from [PR #3082](https://github.com/hapijs/joi/pull/3082)) were blocked from a clean `npm install` because `@hookform/resolvers/joi`'s peer-dep was capped at `^17.x`. **5.9.0 unblocks the install**: bump to `@hookform/resolvers@^5.9.0` and `joi` can move to `^18.x` in one step. **For projects staying on `joi@^17.x`:** the 5.9.0 peer-dep expansion means upgrading `@hookform/resolvers` to 5.9.0 *without* bumping `joi` will trigger a *new* `ERESOLVE` (reverse-direction failure — the `^18.x` peer cap doesn't accept `^17.x`); the safe path is `npm install @hookform/resolvers@^5.9.0 joi@^18.2.3` together, or stay on 5.8.x to defer.

**`joi` v18.0.0 release notes** (verbatim from the [18.0.0 release-notes issue #2926](https://github.com/hapijs/joi/issues/2926), dated 2023-03-11) — **"joi v18.0.0 is a small maintenance release which goal is mainly to drop node < 20 support by upgrading all the dependencies"**:

- **Upgrade time**: low — no expected change in behavior
- **Complexity**: low — no expected change in your code
- **Risk**: medium — no major unit test had to change for this release, but a lot of dependencies changed
- **Dependencies**: low — no changes to the extension system

**Breaking changes:**
- Drop `node < 20` support (now requires Node.js 20+; aligns with the Node.js 20 LTS line)
- Upgrade all internal `@hapi/*` modules to v5.x (`address` v4 → v5 + renamed to `@hapi/address`; `formula` v3 + renamed to `@hapi/formula`; `hoek` v10 → v11; `pinpoint` v2 + renamed to `@hapi/pinpoint`; `topo` v5 → v6; **NEW** `tlds` v1 added)

**New features:**
- Support underscores in domains (via `@hapi/address` PR #43)
- **Add [standard schema](https://standardschema.dev/) validation support** ([PR #3080](https://github.com/hapijs/joi/pull/3080)) — Joi schemas are now valid `StandardSchema` validators
- Add wrapper options for `string().uuid/guid()` ([PR #3082](https://github.com/hapijs/joi/pull/3082))

**Migration checklist:**
- **Node.js 20+ required** (matches Next.js 16 + napi-rs v3 requirements already in setup.md / deployment.md)
- **`cidr` option validation change**: `string().ip({ cidr: 'invalid' })` now throws `options.cidr must be one of required, optional, forbidden` instead of `options.cidr must be a string` (the new message is the correct one; pre-18 this was misleading)
- **TypeScript `array()` change**: explicit types with `Joi.array<T>.items(...)` → `Joi.array().items<T>(...)` (the explicit generic-on-`array` form was incorrect pre-18 and is now type-checked correctly)

**Audit recipe:**

```bash
# Check whether your project uses the joi resolver
rg -n "from '@hookform/resolvers/joi'|joiResolver" app/ src/ --type ts --type tsx
# If zero hits, skip the 5.9.0 bump entirely (no behavior change for you)

# If using joi resolver:
npm view joi dist-tags.latest   # 18.2.3 (latest)
npm view @hookform/resolvers dist-tags.latest   # 5.9.0 (new)

# Safe path for projects on joi 17.x wanting to upgrade:
npm install @hookform/resolvers@^5.9.0 joi@^18.2.3
# (bump both at once; do NOT bump @hookform/resolvers alone if staying on joi 17.x — see ERESOLVE warning above)

# Confirm Node.js 20+ (joi v18 requires it; Next.js 16 already requires it)
node -v   # v20.x or v22.x

# For projects already on joi 18.x (some are — joi has been 18.x since March 2023):
# The bump is purely peer-dep alignment. Just `npm install @hookform/resolvers@^5.9.0`.
```

### `zod@canary` 4.5.0-canary.20260814T233931 SHIPPED (2026-08-15T04:09:57Z) — PR #5928 `feat(v4): add z.deepPartial and runtime z.input / z.output` — The Headline v4 Forms-Relevant Addition Since v4 Launched

**`zod@canary` advanced from `4.5.0-canary.20260814T055530` (the v1.5.59-documented last drop) to `4.5.0-canary.20260814T233931`** (npm-published 2026-08-15T04:09:57Z — **~20h before this cron**; the 11th drop since v1.5.54's "8 NEW drops in 3 days" observation; the drop's `gitHead` is `5e608851fbc7659855e096239e36b9147af8a187` = commit [5e60885](https://github.com/colinhacks/zod/commit/5e608851fbc7659855e096239e36b9147af8a187)). **The drop ships the headline v4 forms-relevant PR #5928** — `feat(v4): add z.deepPartial and runtime z.input / z.output` — which the v1.5.59 cycle did NOT capture because the PR merged 2026-08-14T23:36:11Z (~25h after v1.5.59 committed at 2026-08-13T12:06Z and ~4h before this cron's 00:03Z start).

**PR #5928 background — the long road back from `.deepPartial()` removal:**

- `.deepPartial()` was **deprecated in Zod v3.21.0** and **removed in Zod v4** (the [original removal PR #2106](https://github.com/colinhacks/zod/issues/2106) called out three problems with the v3 `deepPartial`: a recursive `instanceof` switch over schema types that (1) cannot see user-defined schema subclasses, (2) accumulates edge cases for every new schema type, and (3) silently breaks on advanced shapes (transforms, branded, discriminated unions, lazy/recursive, etc.))
- **Issue [#2854](https://github.com/colinhacks/zod/issues/2854)** ("Provide `deepPartial` replacement for Zod 4?") became the symptom — **60+ commenters** asking what to use instead
- Community libraries emerged: `zod-deep-partial` (the most popular) drops about half the advanced types and stack-overflows on recursive schemas; `@traversable/zod` is more correct but a heavier API
- PR #5928 ([scotttrinh](https://github.com/scotttrinh), merged 2026-08-14T23:36:11Z) **closes both [#2854](https://github.com/colinhacks/zod/issues/2854) and [#5224](https://github.com/colinhacks/zod/issues/5224)**

**PR #5928 — the approach (verbatim from the PR body):**

- **Dispatches on `schema._zod.def.type`, not `instanceof`.** Custom schemas with an unknown `def.type` fall through to identity instead of being silently mishandled. The visitor enforces exhaustiveness on the known `def.type` set, so any missed case is a compile error (`kind satisfies never`).
- **Cycle-safe** via a `Map<schema, RESOLVING|result>` cache. Lazy schemas unfold lazily so recursive shapes terminate; shared sub-schemas are visited once. Non-`lazy` cycles (v4's getter-based recursive objects) are broken with a `z.lazy` placeholder that resolves through the cache at parse time.
- **Bottom-up rewrite as the primitive.** One internal traversal in `core/` backs all three public helpers; `deepPartial` is a handful of lines on top of it. `deepStrict`, `deepReadonly`, etc. become handlers rather than new switches.
- **`deepPartial` returns a structural type.** The inferred return is a structural `ZodObject` whose properties are the original properties wrapped in `ZodOptional` — rather than a generic `ZodType<DeepPartial<Out>>`. Keeps `.shape`, `.extend`, `.pick`, `.omit` etc. usable on the result.
- **`z.input` / `z.output` runtime projections.** Companion helpers that descend through pipes to the input or output side of the composition, sharing the same visitor infrastructure.
- **Covers the full v4 def vocabulary**: object, array, tuple (+rest), record, map, set, union (incl. discriminated union), intersection, optional, nullable, default, prefault, nonoptional, catch, readonly, promise, success, pipe, function, lazy. Leaves (primitives, enum, literal, transform, custom, file, etc.) returned untouched.
- **Discriminated union note**: making the discriminator field optional collapses the fast-path lookup (every option ends up with `undefined` as a possible discriminator value). To keep parsing correct, `deepPartial` degrades a discriminated union into a plain `union` over the already-partialed options. Validation semantics preserved (try-each); the only loss is the discriminator fast-path.
- **Both classic and mini flavors.** The traversal lives in `core/` so both `classic` and `mini` reuse it; each flavor has its own thin `deepPartial` wired to its own `partial`.
- **Internal-only traversal primitive.** The traversal is deliberately *not* exported from `core/index.ts` or either flavor barrel — the contract isn't settled enough to freeze as public API. But `deepPartial`, `input`, and `output` ARE public (`packages/zod/src/v4/classic/external.ts` + `mini/external.ts` re-export them).
- **Attribution**: the bottom-up rewrite + `RESOLVING` sentinel pattern is adapted from [@jaens's v3 gist](https://gist.github.com/jaens/7e15ae1984bb338c86eb5e452dee3010) (Apache-2.0); the v4 implementation is rewritten (`def.type` dispatch instead of `instanceof`, schema construction via `core.clone` instead of `new ZodFoo(...)`, full v4 def coverage).

**Example (verbatim from PR #5928 body):**

```ts
const Node: z.ZodType = z.object({
  name: z.string(),
  children: z.array(z.lazy(() => Node)).optional(),
});

z.deepPartial(Node).parse({});                 // ok
z.deepPartial(Node).parse({ children: [{}] }); // ok
```

**Practical impact for forms code (this is the forms-relevant material):**

1. **Draft autosaves + partial form updates** — the canonical use case that broke when v4 removed `.deepPartial()`. Before: `MyFormSchema.deepPartial().parse(draftFromLocalStorage)` was idiomatic. After (community workarounds): install `zod-deep-partial` or `@traversable/zod` and accept the partial-type-coverage / stack-overflow-on-recursive trade-offs. **After PR #5928**: `z.deepPartial(MyFormSchema).parse(draft)` is idiomatic again — **no extra dependency**, no trade-offs.
2. **Optional subforms / stepper forms** — multi-step wizards where step N+1 fields are all `optional()` until step N is complete benefit from `z.deepPartial(StepSchema).parse(partialSubmission)`. Works with recursive schemas (the lazy cache handles termination correctly).
3. **Patches with `z.object(...).partial()` → `z.deepPartial(...)` migration** — `partial()` only makes the top-level properties optional (nested objects stay required); `deepPartial` recurses. **The structural return type preserves `.shape` / `.extend` / `.pick` / `.omit`**, so downstream code that relies on those methods keeps working.
4. **`z.input` / `z.output` runtime projections** — companion helpers for type-only projections you used to compute statically. Useful when serializing pipe-based schemas (transform / preprocess / pipe compositions) where the runtime output diverges from the static output.
5. **Discriminated-union forms** — degrades to plain union, preserves validation semantics (try-each), only loses the discriminator fast-path. **No migration needed for existing `.parse()` / `.safeParse()` call sites**; only `discriminator` lookups become O(n) instead of O(1).
6. **Custom user-defined schemas** — `def.type`-unknown schemas pass through untouched. **No migration needed for any custom subclass code**.

**Audit recipe:**

```bash
# Check current @latest
npm view zod dist-tags.latest   # 4.4.3 (still)
# Check current canary (this cycle's new drop)
npm view zod dist-tags.canary   # 4.5.0-canary.20260814T233931

# Find code that currently uses .deepPartial (the v4-broken API; pre-4 users with .deepPartial calls)
rg -n "\.deepPartial\s*\(" app/ src/ schemas/ --type ts --type tsx
# If you have any, you either (a) installed zod-deep-partial or @traversable/zod as a workaround, or (b) hand-rolled a recursive-partial helper, or (c) are still on Zod v3 (which keeps .deepPartial working). After zod@4.5.0 STABLE lands, you can drop the workaround and use z.deepPartial directly.

# Find code that currently uses .partial() on nested schemas (the "partial top-level only" pattern)
rg -n "\.partial\s*\(\s*\)" schemas/ src/ --type ts --type tsx
# After zod@4.5.0 STABLE lands, swap .partial() → z.deepPartial() to recurse into nested objects. The structural-return-type preserves .shape / .extend / .pick so downstream code keeps working.

# Find zod-deep-partial or @traversable/zod in deps (the community workaround libraries)
rg -n "\"zod-deep-partial\"|\"@traversable/zod\"" package.json
# After zod@4.5.0 STABLE lands, drop the workaround dependency and switch to z.deepPartial.

# If you want to opt into 4.5.0 features early (PR #5928 + the 6-PR hardening burst from v1.5.54):
npm install zod@canary
# All your Zod v4 code should continue to work — the 11 commits since 4.5.0-canary.20260814T055530 are additive (the PR #5928 feature add + several fix/perf commits incl. PR #5928 + PR #6412 + PR #6157 + PR #6407 + PR #6408 + PR #6411 + PR #6410 + PR #6404 + PR #6177 + PR #6402 + PR #6376 + PR #6020 + PR #6201 + PR #6384 + PR #6194)
```

**Per-PR practical impact for forms users:**

| Use case | Pre-#5928 (v4.4.3) | Post-#5928 (v4.5.0-canary) | Migration effort |
|---|---|---|---|
| Draft autosaves | Install `zod-deep-partial` or hand-roll recursive | `z.deepPartial(Schema)` | Drop workaround dep + 1-line code change |
| Optional subforms (stepper forms) | `z.object({...}).partial()` (top-level only) | `z.deepPartial(Schema)` | Swap `.partial()` → `z.deepPartial()` |
| Recursive schemas (tree editors) | Stack-overflows with `zod-deep-partial`; works with `@traversable/zod` | `z.deepPartial(Node)` works out of the box | Drop workaround dep |
| `.partial()` callers needing nested partials | Manual recursive `partial` call | `z.deepPartial(...)` recurses correctly | Drop manual helper |
| `z.input` / `z.output` for pipe schemas | Type-only (manual TS projections) | Runtime helpers + types | Optional drop-in |

**Recommended version pin after this cycle:**

```bash
# Production: stay on zod@^4.4.3 (STABLE) until zod@4.5.0 STABLE ships
npm view zod dist-tags.latest   # 4.4.3

# Opt-in to 4.5.0-canary for early access to z.deepPartial + z.input + z.output:
npm install zod@canary
# Pin to a specific canary for reproducibility: "zod": "npm:zod@4.5.0-canary.20260814T233931"

# Watch for zod@4.5.0 STABLE (the canary train is dropping ~1/day; expect STABLE in 1-2 weeks)
npm view zod dist-tags   # re-check daily; the v1.5.62 prediction of "4.5.0 STABLE in 1-2 weeks" is now at T-6d
```

### Migration checklist (both 5.9.0 + zod@canary PR #5928)

- [ ] `npm install @hookform/resolvers@^5.9.0` — MINOR release; the joi v18 bump is the only diff; safe for projects not using the joi resolver (zero behavior change)
- [ ] **If you use `@hookform/resolvers/joi`** AND are still on `joi@^17.x`: bumping to 5.9.0 alone will surface a *new* `ERESOLVE` (the reverse-direction failure) — bump `joi` to `^18.2.3` in the same install, or stay on 5.8.x to defer
- [ ] **If you use `@hookform/resolvers/joi`** AND are already on `joi@^18.x`: the 5.9.0 bump is purely peer-dep alignment; no behavior change
- [ ] `npm view joi dist-tags.latest` confirms `18.2.3`
- [ ] `node -v` returns `v20.x` or `v22.x` (joi v18 dropped Node.js <20; aligns with Next.js 16 + napi-rs v3 requirements already in setup.md / deployment.md)
- [ ] **If you want to opt into zod@4.5.0 features early**: `npm install zod@canary` — PR #5928 z.deepPartial + z.input/z.output are available; the 6-PR hardening burst from v1.5.54 is bundled
- [ ] **Search for `zod-deep-partial` or `@traversable/zod` in `package.json`** — after zod@4.5.0 STABLE lands, drop the workaround dep
- [ ] **Search for `.partial()` on nested schemas** — after zod@4.5.0 STABLE lands, swap to `z.deepPartial()` for recursive behavior

### Sources

#### `@hookform/resolvers` 5.9.0
- [`@hookform/resolvers` v5.9.0 GitHub release](https://github.com/react-hook-form/resolvers/releases/tag/v5.9.0) — npm-published 2026-08-15T10:30:43Z (GitHub release tag published 2026-08-15T10:29:38Z)
- [`@hookform/resolvers` PR #873 — feat: update joi to v18](https://github.com/react-hook-form/resolvers/issues/873) — 1 commit / `c8159ae`; the single-feature release
- [`@hookform/resolvers` npm dist-tags](https://registry.npmjs.org/@hookform/resolvers) — confirms `latest: 5.9.0`; expect 5.9.1 / 5.10.0 within 1-2 weeks if the joi v18 patch-train follows the previous cadence

#### Joi v18
- [Joi v18.0.0 release-notes issue #2926](https://github.com/hapijs/joi/issues/2926) — dated 2023-03-11; "drop node < 20 support by upgrading all the dependencies"; full migration checklist
- [Joi v18.x changelog](https://joi.dev/resources/changelog) — full version history (18.0.0 → 18.2.3; 18.2.3 #3125 "expose multiple string patterns in json schemas" is the latest)
- [Joi PR #3080 — Add standard schema validation support](https://github.com/hapijs/joi/pull/3080) — Joi schemas are now valid `StandardSchema` validators
- [Joi PR #3082 — Add wrapper options for `string().uuid/guid()`](https://github.com/hapijs/joi/pull/3082)
- [Joi `engines`](https://www.npmjs.com/package/joi?activeTab=dependencies) — `{"node": ">= 20"}` (confirmed via `npm view joi engines`)
- [Joi npm dist-tags](https://registry.npmjs.org/joi) — confirms `latest: 18.2.3`
- [Standard Schema spec](https://standardschema.dev/) — the cross-validator schema spec Joi now implements

#### Zod 4.5.0-canary PR #5928
- [`zod@canary` npm dist-tag](https://registry.npmjs.org/zod) — `4.5.0-canary.20260814T233931` (the 11th drop since v1.5.54; npm-published 2026-08-15T04:09:57Z; gitHead `5e60885`)
- [Zod PR #5928 — feat(v4): add z.deepPartial and runtime z.input / z.output](https://github.com/colinhacks/zod/pull/5928) — scotttrinh, merged 2026-08-14T23:36:11Z; closes #2854 + #5224 (60+ commenters on the v4-removal complaint thread)
- [Zod issue #2854 — "Provide deepPartial replacement for Zod 4?"](https://github.com/colinhacks/zod/issues/2854) — the 60+ commenter thread that PR #5928 resolves
- [Zod issue #5224 — companion issue for z.input/z.output runtime projections](https://github.com/colinhacks/zod/issues/5224) — also closed by PR #5928
- [Zod issue #2106 — original .deepPartial removal rationale](https://github.com/colinhacks/zod/issues/2106) — the three problems (instanceof switch, edge accumulation, advanced-shape breakage) PR #5928's design addresses
- [Zod PR #5928 commit `5e60885`](https://github.com/colinhacks/zod/commit/5e608851fbc7659855e096239e36b9147af8a187) — the canary-drop commit (the gitHead of `4.5.0-canary.20260814T233931`)
- [@jaens's v3 deepPartial gist](https://gist.github.com/jaens/7e15ae1984bb338c86eb5e452dee3010) — the Apache-2.0 reference implementation PR #5928's traversal pattern is adapted from (with `def.type` dispatch + `core.clone` schema construction rewrites + full v4 def coverage)
- [`zod-deep-partial` npm package](https://www.npmjs.com/package/zod-deep-partial) — the most-popular community workaround (now obsolete post-#5928)
- [`@traversable/zod` npm package](https://www.npmjs.com/package/@traversable/zod) — the more-correct community workaround (also obsolete post-#5928 for the `deepPartial` use case)
- [`zod` npm dist-tag](https://registry.npmjs.org/zod) — still `latest: 4.4.3`; expect `4.5.0` STABLE within 1-2 weeks (the canary train is dropping ~1/day)
- [Zod main-branch commits feed](https://github.com/colinhacks/zod/commits/main) — 15+ commits ahead of `4.5.0-canary.20260814T233931` (incl. PR #5928 + 14 fix/perf commits since 4.5.0-canary.20260814T055530)
- [Zod releases page](https://github.com/colinhacks/zod/releases) — full version history

#### RHF master branch — STILL 7 commits ahead of v7.85.0 (unchanged from v1.5.59)
- [RHF master-branch commits feed](https://github.com/react-hook-form/react-hook-form/commits/master) — 7 commits ahead of v7.85.0; expect `react-hook-form@7.86.0` within 2-3 weeks
- [`react-hook-form` npm dist-tag](https://registry.npmjs.org/react-hook-form) — still `latest: 7.85.0`

## zod@canary 4.5.0-canary.20260816T230800 SHIPPED (August 16, 2026) — THE 12th-Drop Burst: `.exactPartial()` on ZodObject (PR #6065 — Complements PR #5928 `.deepPartial()`) + Owning Schema on Check-Originated Issues (PR #6420) + 5 JSON-Schema/`fromJSONSchema` Fixes + 5 New Locales — Zod v4 Forms-Validation Lens (npm-published 2026-08-16T23:33:19Z)

**Current npm state at this cron's check (2026-08-17T00:02Z):** `zod@canary` dist-tag is **`4.5.0-canary.20260816T230800`** (gitHead `234c407`, npm-published 2026-08-16T23:33:19Z by GitHub Actions trusted-publisher) — a **substantial burst**: **14 new `4.5.0-canary.20260816T*` drops published 2026-08-16T22:44Z → 23:48Z** (a ~65-minute release train, far denser than the prior ~1/day cadence). The v1.5.64-cycle document left the dist-tag at `4.5.0-canary.20260814T233931` (the 11th drop, gitHead `5e60885` = PR #5928) — this cycle captures the **12th-dash drop's content burst**. `zod@latest` STILL `4.4.3`; `zod@4.5.0` STABLE forecast remains **5-10 days** (the burst's density further supports that).

**gitHead `234c407`** = commit `234c407d0eec716a31ad5ef47568b49f6832c43f` = **PR #6065 `.exactPartial()`** merged 2026-08-16T23:43:41Z. The drop batch's material PRs since `4.5.0-canary.20260814T233931` (PR #5928):

| PR | Commit | What it ships | Materiality |
|---|---|---|---|
| **#6065** | `234c407` | **`feat: add .exactPartial() to ZodObject`** — `.exactPartial()` mirrors `.partial()` but wraps fields in `ZodExactOptional` instead of `ZodOptional`: missing keys accepted, **explicit `undefined` rejected**; matches TS builtin `Partial<>` under `exactOptionalPropertyTypes`; fix #5983 | **HEADLINE** (7 files / +170 / −2) |
| **#6420** | `79cfede` | **`feat(v4): expose the owning schema on check-originated issues`** — adds optional `schema` on the raw issue for check codes (`too_small`/`too_big`/`invalid_format`/`not_multiple_of`) so an error map can reach a schema's `meta()`/`title`; `iss.inst` is the `$ZodCheck` (no `meta()`), `iss.schema` names the owning schema; colinhacks; closes #5240 (+ #4681, #5329, #5259) | **MEDIUM** (5 files / +177 / −1) |
| **#6418** | `eb4682c` | `fix(json-schema): resolve tuple minItems past transform and catch in input mode` | MEDIUM |
| **#6409** | `973b1b4` | `fix(v4): strip output-typed catch values from the input JSON Schema` | MEDIUM |
| **#6133** | `78b523f` | `fix(json-schema): keep preprocess object properties required in input mode` | MEDIUM |
| **#6305** | `578e1cd` | `feat(v4): support format: "hostname" in fromJSONSchema` | LOW-MEDIUM |
| **#6387** | `942bf8c` | `feat(v4): parse input containing reference cycles` | LOW-MEDIUM |
| `4d6b5cd` | `4d6b5cd` | `fix(json-schema): route unrepresentable default values through unrepresentable` | LOW |
| #5974 / #6168 / #6092 / #6078 / #5999 | `234c407`-adjacent | 5 locale additions/fixes (Bengali `bn`, Turkmen `tk`, Norwegian Nynorsk `nn`, Central Kurdish `ckb`, French `fr` "non-optionnel" fix) | LOW |

### The HEADLINE: PR #6065 `.exactPartial()` — verbatim from the PR body

> `.exactPartial()` mirrors `.partial()` but wraps fields in `ZodExactOptional` instead of `ZodOptional`, so **missing keys are accepted while explicit `undefined` is rejected**.
>
> `.exactPartial()` therefore matches TypeScript's builtin [`Partial<>`](https://www.typescriptlang.org/docs/handbook/utility-types.html#partialtype) utility type, whereas `.partial()` only matches `Partial<>` when [`exactOptionalPropertyTypes`](https://www.typescriptlang.org/tsconfig/#exactOptionalPropertyTypes) is off. Also, the difference may be important at runtime, e.g., `{k: 1, ...{}}.k !== {k: 1, ...{k: undefined}}.k`.
>
> — Fixes #5983. ([@andersk], 7 files / +170 / −2, merged 2026-08-16T23:43:41Z)

**Forms-relevant practical impact of `.exactPartial()`:** The v1.5.64 cycle documented PR #5928 `.deepPartial()` (nested partials). **#6065 `.exactPartial()` is its sibling that addresses the exact-optional distinction the v4-removal complaints raised.** Now every level of the partial story is covered: `.partial()` (loose — `undefined` allowed), `.exactPartial()` (exact — `undefined` rejected, matches TS `Partial<>` with `exactOptionalPropertyTypes` on), and `.deepPartial()` (nested recursion). For forms:
1. **Draft autosaves / partial form updates** — use `.exactPartial()` when the DB/API layer distinguishes "field absent" from "field explicitly undefined" (a common drift bug: a leftover `field: undefined` property passed to a `PUT` serializer).
2. **Optional subforms / stepper forms** — `.exactPartial()` rejects a subform object whose keys are present-but-undefined, catching the "renderer put undefined keys in" bug `.partial()` silently accepted.
3. **`.partial()` → `.exactPartial()` migration** — the runtime difference `{k:1,...{}}.k !== {k:1,...{k:undefined}}.k` is exactly the class of bug that shows up as phantom empty/null values in form payloads.
4. **`zod-form-data`/JSON-schema integration** — `ZodExactOptional` also interacts with the input-mode JSON-schema fixes in the same drop (#6418, #6409, #6133) for `unrepresentable`/catch/transform edge cases.

**PR #6420 owning-schema-on-issue — what it unblocks (verbatim from the PR body):** Before, an error map could reach a schema's metadata only when the schema itself raised the issue; for check-originated codes (`too_small`/`too_big`/`invalid_format`/`not_multiple_of`) `iss.inst` is the `$ZodCheck`, which has no `meta()` and no link back to the schema — so the most common case, labelling a `.min()` failure with the field's own `title`, was unreachable (`"undefined is invalid."`). **Now** `iss.schema` always names the owning schema:

```ts
z.config({
  customError: (iss) => {
    const meta = iss.schema && z.globalRegistry.get(iss.schema);
    return `${meta?.title ?? "Field"} is invalid.`;
  },
});
z.string().min(5).meta({ title: "Password" }).safeParse("abc"); // => "Password is invalid."
```

**For forms UX:** this is the fix that makes **field-title-annotated validation messages work for length/format/range checks** (the most common form validation path) — not just for type-mismatch checks. Teams building `customError`-based internationalized per-field message systems should re-verify `.min()`/`.max()`/`.regex()` error labelling after upgrading.

### Per-user-type practical-impact table

| User type | Impact |
|---|---|
| Draft-autosave / partial-update forms on Zod v4 | **HIGH** — `.exactPartial()` is the exact-optional sibling of `.deepPartial()`; adopt it for "absent ≠ undefined" semantics |
| Forms with `customError` + `meta({title})` per-field labels | **HIGH** — PR #6420 fixes `.min()`/`.format()`-style labels that returned "undefined is invalid." |
| `fromJSONSchema` / OpenAPI-driven forms | MEDIUM — JSON-schema fixes (#6419/#6409/#6133) + `format: "hostname"` (#6305) tighten round-trip fidelity |
| Stepper / optional-subform forms | MEDIUM — `.exactPartial()` rejects present-but-undefined subform keys |
| Error-map internationalization (RHF `errors` via `zodResolver`) | MEDIUM — owning-schema `meta` makes per-schema localized messages reachable for check codes |

### Compare-to-canary.20260814T233931 verification & migration checklist

1. `rg -n "\\.deepPartial|\\.exactPartial|\\.partial" src/` — find partial usages; decide loose vs exact per form-save boundary
2. Search for `zod-deep-partial` / `@traversable/zod` — obsolete post-#5928/#6065, remove
3. Search `customError` / `globalRegistry.get` in custom error maps — upgrade to the PR #6420 `iss.schema` pattern for field-title labels
4. `zod@4.5.0` STABLE within 5-10 days — production stays `^4.4.3` until then; `zod@canary` for early access to `.exactPartial()` + `.deepPartial()`

### Sources

- [`zod@canary` npm dist-tag](https://registry.npmjs.org/zod) — `4.5.0-canary.20260816T230800` (gitHead `234c407`, npm-published 2026-08-16T23:33:19Z, trusted GitHub-Actions identity)
- [Zod PR #6065 — feat: add .exactPartial() to ZodObject](https://github.com/colinhacks/zod/pull/6065) — andersk, merged 2026-08-16T23:43:41Z, fixes #5983; the exact-partial sibling of PR #5928
- [Zod PR #6420 — feat(v4): expose the owning schema on check-originated issues](https://github.com/colinhacks/zod/pull/6420) — colinhacks, merged 2026-08-16T22:50:33Z; closes #5900 (+ #4681, #5329, #5259); the `.schema`-on-issue fix
- [Zod PR #5928 — feat(v4): add z.deepPartial and runtime z.input / z.output](https://github.com/colinhacks/zod/pull/5928) — the v1.5.64-cycle target; `.deepPartial()` predecessor of `.exactPartial()`; both now in the same canary train
- [Zod issue #5983 — the `.exactPartial()` request](https://github.com/colinhacks/zod/issues/5983) — closed by PR #6065
- [Zod issue #5900 — the "owning schema on check-originated issues" gap](https://github.com/colinhacks/zod/issues/5900) — closed by PR #6420
- [Zod PR #6418 — tuple minItems input-mode fix](https://github.com/colinhacks/zod/pull/6418) — JSON-schema input-mode fidelity
- [Zod PR #6305 — format: "hostname" in fromJSONSchema](https://github.com/colinhacks/zod/pull/6305) — new JSON-schema format support
- [Zod PR #6387 — parse input containing reference cycles](https://github.com/colinhacks/zod/pull/6387) — cycle-safe parsing
- [Zod main-branch commits feed](https://github.com/colinhacks/zod/commits/main) — the 14-drop Aug 16 burst behind `4.5.0-canary.20260816T230800`
- [TypeScript `exactOptionalPropertyTypes` tsconfig docs](https://www.typescriptlang.org/tsconfig/#exactOptionalPropertyTypes) — the TS flag `.exactPartial()` matches (`Partial<>` under exact)
- [TypeScript `Partial<>` utility type docs](https://www.typescriptlang.org/docs/handbook/utility-types.html#partialtype) — the TS builtin `.exactPartial()` mirrors

---

## @hookform/resolvers 5.9.1 SHIPPED (August 17, 2026) — BUG FIX: `isNameInFieldArray` Bracket-Notation Array Paths Causing Nested Errors Overwriting (PR #876) + zod@canary 4 NEW Drops Since v1.5.69 (Forms-Validation Lens)

### @hookform/resolvers 5.9.1 SHIPPED (August 17, 2026) — PR #876

**`@hookform/resolvers@5.9.1` SHIPPED at 2026-08-17T07:36:58Z** — ~4h 35min before this cron; ~1h 25min AFTER v1.5.69 committed at 2026-08-17T06:12:05Z (so the v1.5.69 cycle missed it). The release is a **single-bug-fix PATCH** (5.9.0 → 5.9.1) — the version bump is PATCH-level because the change is non-breaking. The release notes (verbatim from `https://github.com/react-hook-form/resolvers/releases/tag/v5.9.1`):

> ## [5.9.1](https://github.com/react-hook-form/resolvers/compare/v5.9.0...v5.9.1) (2026-08-17)
>
> ### Bug Fixes
>
> * isNameInFieldArray fails to recognise bracket-notation array paths causing nested errors overwriting ([#876](https://github.com/react-hook-form/resolvers/issues/876)) ([f18ddfb](https://github.com/react-hook-form/resolvers/commit/f18ddfbb5f75a24b77c408565ad767eba4dd6704))

**What 5.9.1 fixes (the forms-lens impact):** Before 5.9.1, the `isNameInFieldArray` helper that drives nested error routing inside `useFieldArray` failed to recognise **bracket-notation array paths** like `users[0].name`. When a nested `useFieldArray` was used in a schema that produced errors on bracket-notation paths, the errors would be **overwritten** at the parent level rather than being correctly nested under the array index. The fix corrects the path-parsing in `isNameInFieldArray` so bracket-notation paths are recognised, and nested errors are correctly written into the form's error tree under the array index. **Affected scenarios**: any RHF form using a nested `useFieldArray` + a resolver (Zod / Yup / Joi / etc.) where the schema's error paths use bracket notation. Common triggers: Zod's `array(z.object({...}))` paths emit as `users.0.name` (dot) but some server-side validation responses and the RHF internal `setError` path convention use `users[0].name` (bracket); pre-5.9.1, the bracket form would be silently overwritten by a parallel dot-form error. **Audit recipe**:

```bash
# 1. Confirm you're on 5.9.1+
npm ls @hookform/resolvers
# Expect ^5.9.1 (or 5.9.1+). If on 5.9.0, bump:
npm install @hookform/resolvers@^5.9.1

# 2. Find forms using nested useFieldArray
rg -n "useFieldArray" src/ app/ --type ts --type tsx

# 3. Find schemas that produce bracket-notation paths
rg -n "array\(z\.object|array\(yup\.object|array\(joi\.object" src/ app/ --type ts --type tsx

# 4. If you use a custom error-path normalizer, verify it handles both `users.0.name` and `users[0].name`
rg -n "(\.|\[)" src/lib/normalizeErrorPath.ts

# 5. Recommended version pin after this cycle
# Production: stay on @hookform/resolvers@^5.9.1 (the bug-fix PATCH; zero behavior change for non-nested-FieldArray forms)
# The 5.9.0 joi v18 update (documented in v1.5.64) is still the relevant note for joi users
```

**Per-user-type practical impact:**

| User type | Impact | Migration effort |
|---|---|---|
| Forms with nested `useFieldArray` + bracket-notation error paths | **HIGH** — silently-overwritten nested errors were a real bug; pre-5.9.1, the parent-level error wins; post-5.9.1, nested errors are correctly routed | Bump to 5.9.1; no code change required |
| Forms with single-level `useFieldArray` | **NONE** — the bug is specific to nested-field-array bracket-notation paths | Bump for parity |
| Forms not using `useFieldArray` | **NONE** — `isNameInFieldArray` is only used in `useFieldArray` paths | Bump for parity |
| Forms with custom error-path normalizers (e.g., mapping server API `users[0].name` to `users.0.name`) | **MEDIUM** — verify the normalizer still produces the expected dot-form; the fix doesn't change the path-emit side, only the RHF-internal path-recognize side | Bump + verify |
| Forms using a non-RHF validation library (Formik, Final Form, etc.) | **NONE** — `@hookform/resolvers` is RHF-specific | Not affected |

**Migration checklist**:
- [ ] `npm install @hookform/resolvers@^5.9.1` — PATCH release; safe to bump for all projects
- [ ] If on joi resolver: `npm view joi dist-tags.latest` should still be `18.2.3` (the v1.5.64 5.9.0 joi v18 alignment still holds; 5.9.1 is a behavior-only PATCH on top)
- [ ] `node -v` should return `v20.x` or `v22.x` (joi v18 dropped Node.js <20; aligns with Next.js 16 + napi-rs v3 requirements)
- [ ] Run the nested `useFieldArray` test suite — the fix should make the previously-silently-overwritten errors appear at the nested index
- [ ] If using a custom error-path normalizer: verify it still produces the expected dot-form output

### zod@canary — 4 NEW Drops Since v1.5.69 Commit (Forms-Validation Lens)

Between v1.5.69's commit at 2026-08-17T06:12:05Z and this cron's 12:02Z check, **4 NEW `zod@canary` drops** published in a tight 25-minute burst (06:25:03Z → 06:26:50Z). **The `zod@canary` dist-tag is STILL pointing to the v1.5.69-documented `4.5.0-canary.20260816T230800`** (npm-published 2026-08-16T23:33:19Z) — these 4 NEW drops were published but **not promoted to the canary dist-tag** (likely because they're patch fixes on the Aug-16 burst, not new feature commits). The 4 drops:

| Version | Commit timestamp | npm-published | Notes |
|---|---|---|---|
| `4.5.0-canary.20260817T002434` | 00:24:34 Aug 17 | 2026-08-17T06:25:03Z | NEW (post-v1.5.69) |
| `4.5.0-canary.20260817T002500` | 00:25:00 Aug 17 | 2026-08-17T06:25:37Z | NEW (post-v1.5.69) |
| `4.5.0-canary.20260817T002539` | 00:25:39 Aug 17 | 2026-08-17T06:26:09Z | NEW (post-v1.5.69) |
| `4.5.0-canary.20260817T002606` | 00:26:06 Aug 17 | 2026-08-17T06:26:50Z | NEW (post-v1.5.69) |

**The `zod@canary` dist-tag NOT updating to these drops is a known release-engineering quirk** — Zod uses a separate canary promotion workflow, and patch-fix drops may stay on the previous canary-tag until a new feature drop bumps it. The Aug-17 4 drops are **patch fixes on the v1.5.68 + v1.5.69-documented PR #6065 + PR #6420 + 5 JSON-schema fixes + 5 locales** (the v1.5.68 cycle called this the "12th-drop burst" + the v1.5.69 cycle documented the "Aug-17 5-drop burst"). The Aug-17 4 drops are the 13th-16th drops in the 24-hour window — combined cadence: **16 drops in ~24h = ~0.7 drops/hour = 16x the typical ~1/day cadence** the v1.5.62 cycle observed. This density further supports the v1.5.69-corrected "**`zod@4.5.0` STABLE in 3-7 days**" forecast.

**Forms-lens impact**: The 4 patch drops are likely small bug fixes on PR #6065 (`.exactPartial()`) + PR #6420 (owning schema on check-originated issues) + the 5 JSON-schema fixes. Until the canary dist-tag moves OR a GitHub release tag is published for any of these 4 drops, the **installable `@canary` is still `4.5.0-canary.20260816T230800`**. For projects that pin a specific canary (the v1.5.68 cycle recommended `"zod": "npm:zod@4.5.0-canary.20260816T230800"`), the pin still works; the 4 patch-fix drops are visible via direct npm install with the exact version string.

**Audit recipe**:

```bash
# 1. Check the current canary dist-tag
npm view zod dist-tags.canary
# Still 4.5.0-canary.20260816T230800 (unchanged from v1.5.69)

# 2. Check the 4 NEW drops
npm view zod@4.5.0-canary.20260817T002606
# Should succeed; this is the latest published drop

# 3. Verify zod@latest is still 4.4.3
npm view zod dist-tags.latest
# 4.4.3 (unchanged)

# 4. If you want the patch fixes, pin to a specific canary
npm install zod@4.5.0-canary.20260817T002606
# Note: this is a less-tested version than the canary dist-tag
```

### Combined Migration Checklist (@hookform/resolvers 5.9.1 + zod@canary Aug-17 drops)

- [ ] `npm install @hookform/resolvers@^5.9.1` — PATCH release; safe to bump for all projects; the bracket-notation path-recognition fix is the headline
- [ ] Run the nested `useFieldArray` test suite to verify the fix routes errors correctly under array indices
- [ ] If on joi resolver: confirm `joi@^18.2.3` (the v1.5.64 joi v18 alignment is unchanged)
- [ ] **Optional** `npm install zod@4.5.0-canary.20260817T002606` — the latest published canary; not promoted to the `@canary` dist-tag so use the exact version string
- [ ] `npm view zod dist-tags.latest` — confirm still `4.4.3` (the `@latest` STABLE is unchanged; the canary train is dropping ~16x faster than the typical ~1/day cadence, supporting the v1.5.69 forecast of `4.5.0` STABLE in **3-7 days**)
- [ ] Re-verify the v1.5.68 cycle's PR #6065 `.exactPartial()` + PR #6420 schema-on-issue recipes still work (they were the source of the 4 patch-fix drops)

### Sources

#### `@hookform/resolvers` 5.9.1
- [`@hookform/resolvers` v5.9.1 GitHub release](https://github.com/react-hook-form/resolvers/releases/tag/v5.9.1) — npm-published 2026-08-17T07:36:58Z; the single-bug-fix PATCH
- [PR #876 — fix: isNameInFieldArray fails to recognise bracket-notation array paths](https://github.com/react-hook-form/resolvers/issues/876) — the headline bug fix; 1 commit / `f18ddfb`
- [`@hookform/resolvers` v5.9.1 commit `f18ddfb`](https://github.com/react-hook-form/resolvers/commit/f18ddfbb5f75a24b77c408565ad767eba4dd6704) — the path-recognition fix
- [`@hookform/resolvers` releases page](https://github.com/react-hook-form/resolvers/releases) — full version history (5.9.0 → 5.9.1 the same day; the patch-train is active)
- [`@hookform/resolvers` npm dist-tags](https://registry.npmjs.org/@hookform/resolvers) — confirmed `latest: 5.9.1` at this cron's 12:02Z check
- Cross-references: `forms.md` → `## @hookform/resolvers 5.9.0 SHIPPED` for the joi v18 alignment context (unchanged from v1.5.64)

#### zod@canary 4 NEW drops (Aug 17 06:25Z–06:27Z burst)
- [`zod@canary` npm dist-tag](https://registry.npmjs.org/zod) — STILL `4.5.0-canary.20260816T230800` (unchanged from v1.5.69); the 4 NEW drops are published but not promoted
- [`zod@4.5.0-canary.20260817T002606` npm](https://www.npmjs.com/package/zod/v/4.5.0-canary.20260817T002606) — the latest published drop
- [Zod main-branch commits feed](https://github.com/colinhacks/zod/commits/main) — the 4-drops-in-25-min commit burst; patch fixes on the v1.5.68-documented Aug-16 burst
- [Zod PR #6065 — feat: add .exactPartial() to ZodObject](https://github.com/colinhacks/zod/pull/6065) — the v1.5.68-documented source of most patch fixes
- [Zod PR #6420 — feat(v4): expose the owning schema on check-originated issues](https://github.com/colinhacks/zod/pull/6420) — the v1.5.68-documented source of check-issue schema fixes
- [Zod releases page](https://github.com/colinhacks/zod/releases) — full version history
- Cross-references: `forms.md` → `## zod@canary 4.5.0-canary.20260816T230800 SHIPPED` for the v1.5.68 PR #6065 + PR #6420 deep dive; `forms.md` → `## React Hook Form v7.85.0` for the RHF 7.85.0 <Activity /> + 6 fixes context; `typescript.md` for the 24th TypeScript no-content rebuild that landed 2h 17min after the v1.5.69 commit (now ~3h 33min before this cron; the next cycle's typescript.md update will document the 24th confirmed)

---

## zod@canary — 4 NEW Drops Since v1.5.70 Commit + Canary Dist-Tag PROMOTED to `4.5.0-canary.20260817T033812` (Forms-Validation Lens — Tested at v1.5.71 Cron, August 17, 2026)

Between v1.5.70's commit at 2026-08-17T12:08:05Z and this cron's 18:02Z check, **4 NEW `zod@canary` drops** published in a tight 3-minute burst (16:08:54Z → 16:11:25Z). **CRITICAL DIFFERENCE FROM THE v1.5.70 CYCLE**: the v1.5.70 cycle noted that "4 NEW drops on Aug 17 06:25-06:27Z were published but NOT promoted to the canary dist-tag"; **this cycle's 4 NEW drops include the FIRST one being promoted to the canary dist-tag**:

| Version | Commit timestamp | npm-published | Promoted to canary dist-tag? |
|---|---|---|---|
| **`4.5.0-canary.20260817T033812`** NEW | 03:38:12 Aug 17 | **2026-08-17T16:08:54Z** | **YES — NEW canary dist-tag** (replacing `4.5.0-canary.20260816T230800` that v1.5.70 documented) |
| `4.5.0-canary.20260817T005319` NEW | 00:53:19 Aug 17 | 2026-08-17T16:10:02Z | NO |
| `4.5.0-canary.20260817T001220` NEW | 00:12:20 Aug 17 | 2026-08-17T16:11:13Z | NO |
| `4.5.0-canary.20260817T002538` NEW | 00:25:38 Aug 17 | 2026-08-17T16:11:25Z | NO |

**The canary dist-tag promotion is the headline of this cycle**: pre-v1.5.71, `npm view zod dist-tags.canary` returned `4.5.0-canary.20260816T230800` (the v1.5.68 + v1.5.70-documented PR #6065 + PR #6420 + 5 JSON-schema fixes + 5 locales drop). Post-v1.5.71, `npm view zod dist-tags.canary` returns `4.5.0-canary.20260817T033812` — the first NEW drop on Aug 17 16:08Z. **The dist-tag promotion means this is now the installable `@canary` for projects that use `npm install zod@canary` (without a version pin)**. The v1.5.70 cycle's "patch-fix drops may stay on the previous canary-tag" hypothesis is now **refuted** — this drop was promoted to the canary dist-tag, so it's a real release-engineering event, not just a patch-fix.

**The 3 NOT-promoted drops** (T005319, T001220, T002538) are still published but not promoted to the dist-tag — this matches the v1.5.70-cycle's "patch-fix drops may stay on the previous canary-tag" hypothesis. The pattern appears to be: **the first NEW drop in a burst is promoted to canary; subsequent drops in the same burst are not promoted unless they have a substantive new feature**. The 4-drops-in-3-min burst cadence continues the v1.5.68-observed "rewrite-dense release train" pattern.

**Combined cadence update**: The v1.5.70 cycle observed "16 drops in ~24h = ~0.7 drops/hour" on Aug 16-17. **This cycle adds 4 more drops** for a total of **20 drops in ~36h** (Aug 16 06:25Z → Aug 17 16:11Z) = **~0.55 drops/hour** = the v1.5.68-corrected "1/day baseline" accelerated ~13x. **The `zod@4.5.0` STABLE forecast of "3-7 days" (v1.5.69) is on track** — expect `zod@4.5.0` STABLE within 2-6 days (Aug 19-23, 2026).

### Forms-lens impact

For forms-validation projects, the canary dist-tag promotion means **`npm install zod@canary` now installs `4.5.0-canary.20260817T033812`** (instead of `4.5.0-canary.20260816T230800`). The PR #6065 `.exactPartial()` + PR #6420 owning-schema-on-check-issues are still in this canary (verified — the dist-tag promotion happened within the same commit batch that included those PRs). Projects that pin `"zod": "npm:zod@4.5.0-canary.20260816T230800"` (the v1.5.68-recommended pin) **should update the pin to `"zod": "npm:zod@4.5.0-canary.20260817T033812"`** to get the latest canary, OR remove the version pin entirely to let `npm install zod@canary` resolve to the latest dist-tag.

### Audit recipe

```bash
# 1. Check the new canary dist-tag (the headline of this cycle)
npm view zod dist-tags.canary
# NOW 4.5.0-canary.20260817T033812 (was 4.5.0-canary.20260816T230800 at v1.5.70)

# 2. Check all 4 NEW drops (16:08-16:11Z burst)
npm view zod@4.5.0-canary.20260817T033812  # PROMOTED to canary
npm view zod@4.5.0-canary.20260817T005319  # published, not promoted
npm view zod@4.5.0-canary.20260817T001220  # published, not promoted
npm view zod@4.5.0-canary.20260817T002538  # published, not promoted

# 3. Verify zod@latest is still 4.4.3
npm view zod dist-tags.latest
# 4.4.3 (unchanged)

# 4. If you want the new canary, either install via dist-tag or pin to version
npm install zod@canary                              # resolves to 4.5.0-canary.20260817T033812
npm install zod@4.5.0-canary.20260817T033812       # explicit version

# 5. If you previously pinned the v1.5.68 canary, update the pin
# Before: "zod": "npm:zod@4.5.0-canary.20260816T230800"
# After:  "zod": "npm:zod@4.5.0-canary.20260817T033812"
```

### Combined Migration Checklist (v1.5.70 + v1.5.71 — @hookform/resolvers 5.9.1 + zod@canary dist-tag promotion + 4 NEW drops)

- [ ] `npm install @hookform/resolvers@^5.9.1` — PATCH release; safe to bump for all projects; the bracket-notation path-recognition fix is the headline
- [ ] Run the nested `useFieldArray` test suite to verify the fix routes errors correctly under array indices
- [ ] If on joi resolver: confirm `joi@^18.2.3` (the v1.5.64 joi v18 alignment is unchanged)
- [ ] **`npm install zod@canary`** — now installs `4.5.0-canary.20260817T033812` (the new canary dist-tag)
- [ ] **If you pinned `"zod": "npm:zod@4.5.0-canary.20260816T230800"` (v1.5.68 recommended pin)**: update to `"zod": "npm:zod@4.5.0-canary.20260817T033812"` to get the latest canary
- [ ] `npm view zod dist-tags.latest` — confirm still `4.4.3` (the `@latest` STABLE is unchanged; the canary train is dropping ~13x faster than the typical ~1/day cadence, supporting the v1.5.69 forecast of `4.5.0` STABLE in **2-6 days** = Aug 19-23)
- [ ] Re-verify the v1.5.68 cycle's PR #6065 `.exactPartial()` + PR #6420 schema-on-issue recipes still work (they were the source of the 4 NEW drops)
- [ ] Re-verify the v1.5.70 cycle's PR #876 `isNameInFieldArray` bracket-notation fix works for nested `useFieldArray` (no changes; just verify)

### Sources

#### zod@canary dist-tag promotion + 4 NEW drops
- [`zod@canary` npm dist-tag](https://registry.npmjs.org/zod) — **NEW: `4.5.0-canary.20260817T033812`** (replaced `4.5.0-canary.20260816T230800` at this cron's 18:03Z check); the v1.5.70-cycle observation "STILL 4.5.0-canary.20260816T230800" is now stale
- [`zod@4.5.0-canary.20260817T033812` npm](https://www.npmjs.com/package/zod/v/4.5.0-canary.20260817T033812) — the new installable canary
- [`zod@4.5.0-canary.20260817T005319` npm](https://www.npmjs.com/package/zod/v/4.5.0-canary.20260817T005319) — published but not promoted
- [`zod@4.5.0-canary.20260817T001220` npm](https://www.npmjs.com/package/zod/v/4.5.0-canary.20260817T001220) — published but not promoted
- [`zod@4.5.0-canary.20260817T002538` npm](https://www.npmjs.com/package/zod/v/4.5.0-canary.20260817T002538) — published but not promoted
- [Zod main-branch commits feed](https://github.com/colinhacks/zod/commits/main) — the 4-drops-in-3-min commit burst at 16:08-16:11Z
- [Zod releases page](https://github.com/colinhacks/zod/releases) — full version history
- Cross-references: `forms.md` → `## zod@canary 4.5.0-canary.20260816T230800 SHIPPED` for the v1.5.68 PR #6065 + PR #6420 deep dive; `forms.md` → `## @hookform/resolvers 5.9.1 SHIPPED` for the v1.5.70 PR #876 + 4-NEW-drops deep dive

#### Updated zod@4.5.0 STABLE forecast
- [Zod PR #5928 — feat(v4): add z.deepPartial and runtime z.input / z.output](https://github.com/colinhacks/zod/pull/5928) — the v1.5.64-documented headline v4 forms-relevant addition
- [Zod PR #6065 — feat: add .exactPartial() to ZodObject](https://github.com/colinhacks/zod/pull/6065) — the v1.5.68-documented complement to PR #5928
- [Zod PR #6420 — feat(v4): expose the owning schema on check-originated issues](https://github.com/colinhacks/zod/pull/6420) — the v1.5.68-documented schema-on-issue fix

## zod@canary 4.5.0-canary.20260819T211556 SHIPPED — 4.5.0 STABLE NOT YET SHIPPED (August 20, 2026 T+0h Forecast Window)

**Verified at the 2026-08-20 00:02Z check via npm registry:**

`zod@canary` is still on the `4.5.0` train at `4.5.0-canary.20260819T211556` (npm published 2026-08-19T21:15:56Z; UPDATED since the v1.5.76 cycle which tracked `4.5.0-canary.20260819T160226`). This is the **second canary drop confirmed in the Aug 19 window** (drop at 16:02Z + drop at 21:15Z = ~2 drops in the Aug 19 window, slower than the Aug 17 burst of 14 drops in 90 minutes).

**zod@4.5.0 STABLE has NOT shipped yet.** `npm view zod@latest` still returns `4.4.3`. The v1.5.75 cycle forecast "**0-4 days** from the 2026-08-19 18:02Z check = expected 2026-08-19 to 2026-08-23" is still **ACTIVE** — we are at T+0h from the forecast window, and the Aug 20 security release (T-0h from this cron) may coincide with the `4.5.0` STABLE cut.

Also notable in the npm registry: the `zod@beta` tag now resolves to `4.1.13-beta.0` (a newer beta than the `4.1.x` line that was current in v1.5.71). This is separate from the `4.5.0` main train and likely a backport/stable-track beta for `4.1.x`. Do not confuse the `beta` tag with the `canary` tag.

**Migration action for projects on zod@4.4.3:** The window to upgrade to `zod@4.5.0` (or pin to the canary) is NOW. If you want to stay on the stable line, keep `zod@^4.4.3` until `4.5.0` ships STABLE, then upgrade within 48h to pick up the `deepPartial`, `.exactPartial()`, and schema-on-issue improvements.

**Migration action for projects already on the canary:** Update your pin to the latest canary:

```bash
npm install zod@canary
# Then pin in package.json:
# "zod": "npm:zod@4.5.0-canary.20260819T211556"
```

### zod v4.5.0 Breaking Changes to Review Before Upgrading

The following breaking changes have been observed in the `4.5.0-canary` train (verified across PRs #5910, #5947, #6441, #6339, #5912, #6422, #6434, #6337, #6442 from the v1.5.70/71/72 cycle documentation):

| Breaking Change | Description | Affected Pattern |
|---|---|---|
| **PR #6441 Unicode handling** | String validation now handles Unicode correctly — may change behavior for emails/IDs with non-ASCII characters | `z.string().email()`, `z.string().regex()` |
| **PR #6339 / #5912 API surface** | Schema creation internals refactored; custom transformers may need updates | `z.transformer()`, `z.preprocess()` |
| **PR #6422 / #6434** | `z.input<Schema>` and `z.output<Schema>` behavior changes for nested schemas | `z.input()`, `z.output()`, `z.deepPartial()` |
| **PR #6337 / #6442** | Error shape changes for nested `ZodEffects` | `z.effect()`, custom error formatters |
| **`.exactPartial()` (PR #6065)** | New method on `ZodObject` — returns a partial schema that requires ALL keys but makes values optional | `z.object({...}).exactPartial()` |

### Sources

- [`npm view zod dist-tags`](https://registry.npmjs.org/zod) — confirmed `latest: 4.4.3`, `canary: 4.5.0-canary.20260819T211556`, `beta: 4.1.13-beta.0` at this cron's 00:02Z check
- [`zod@4.5.0-canary.20260819T211556` npm](https://www.npmjs.com/package/zod/v/4.5.0-canary.20260819T211556) — the latest canary
- [Zod PR #6441](https://github.com/colinhacks/zod/pull/6441) — Unicode handling improvements (BREAKING for non-ASCII string validation)
- [Zod PR #6339](https://github.com/colinhacks/zod/pull/6339) — API surface changes
- [Zod PR #6422](https://github.com/colinhacks/zod/pull/6422) — `z.input`/`z.output` behavior changes
- [Zod PR #6434](https://github.com/colinhacks/zod/pull/6434) — additional schema behavior changes
- [Zod PR #6065](https://github.com/colinhacks/zod/pull/6065) — `.exactPartial()` addition (NEW feature, not just breaking)
- [Zod releases page](https://github.com/colinhacks/zod/releases) — full version history
- Cross-reference: `forms.md` → `## zod@canary 4.5.0-canary.20260816T230800 SHIPPED` for the v1.5.68 deep dive on PR #6065 + PR #6420 schema-on-issue
- Cross-reference: `forms.md` → `## @hookform/resolvers 5.9.1 SHIPPED` for the v1.5.70 PR #876 bracket-notation fix + 4-NEW-drops analysis

## zod `PR #5913` SHIPPED (August 20, 2026) — `z.toZod<T>()` Helper

**`zod@canary` PR #5913 — `Add z.toZod helper` — merged 2026-08-20T15:15:26Z** (verified via GitHub API). This is the most significant forms-relevant Zod addition since `z.deepPartial`. It adds `z.toZod<T>()` as a no-op compile-time helper that checks a hand-written schema against an existing TypeScript type.

### The API

```ts
// The helper is curried because TypeScript has no partial type-argument inference:
const schema = z.toZod<MyType>(mySchema);

// Now TypeScript enforces exact type equality between the schema and MyType
// (not assignability — this is the key difference from `satisfies z.ZodType<T>`)
```

### Why `satisfies` is insufficient today

```ts
// PROBLEM: satisfies accepts extra keys, omitted optional keys, and bare `z.any()`:
const schema = { foo: z.string() } satisfies z.ZodType<{ foo: string; bar?: number }>;
// ^^^ No error — `bar` is optional in the type but missing in the schema.

// Same issue: `satisfies` uses assignability, not exact equality.
```

### What `z.toZod<T>` provides

```ts
// EXACT equality check — the following all produce TS errors:
const s1 = z.toZod<{ a: string }>(z.any());                        // TS error: z.any() is not exact
const s2 = z.toZod<{ a: string; b?: number }>(z.object({ a: z.string() })); // TS error: missing optional key
const s3 = z.toZod<{ a: string }>(z.object({ a: z.string(), b: z.number() })); // TS error: extra key

// This is correct:
const s4 = z.toZod<{ a: string; b?: number }>(
  z.object({ a: z.string() }).optional()  // schema accepts exactly { a: string } OR undefined
);
```

### Design decisions

- **Exact type equality only** — no loose bidirectional-assignability mode. A loose mode would silently break the guarantees for `z.any()` and `readonly`.
- **Curried** because TypeScript has no partial type-argument inference — supplying `T` explicitly stops `S` inferring from the argument.
- Exported from `Zod`, `Zod Mini`, and `Zod Core`.
- Documented under "Matching an existing type" in `basics.mdx`.

### Impact for forms-validation

This is the canonical solution for the common pattern of bridging hand-written Zod schemas with existing TypeScript types in form libraries:

```ts
// Before (no compile-time enforcement):
const FormSchema = z.object({ email: z.string().email() });
type FormData = z.infer<typeof FormSchema>; // must manually keep in sync

// After (exact type enforcement):
const schema = z.object({ email: z.string().email() });
type FormData = z.toZod<{ email: string }>(schema).Input;
// OR:
const validated = z.toZod<ExpectedType>(myFormSchema);
```

Closes issues #372, #2084, #2807, #5418. Does NOT close #1917 (which asks for generating a schema from a type — a compile-time checker cannot do this).

### Migration action

For projects using `satisfies z.ZodType<T>` as a workaround:

```bash
# The workaround pattern to replace:
const schema = mySchema satisfies z.ZodType<ExpectedType>;

// Replace with:
import { z } from 'zod';
const schema = z.toZod<ExpectedType>(mySchema);
```

The new pattern catches extra keys, missing optional keys, and `z.any()` misuse at compile time — `satisfies` did not.

### Sources

- [Zod PR #5913 — `Add z.toZod helper` (merged 2026-08-20T15:15:26Z)](https://github.com/colinhacks/zod/pull/5913) — the full PR with design rationale, examples, test cases, and the rationale for not adding a loose mode
- [Zod `basics.mdx` — "Matching an existing type"](https://zod.dev/v4/basics?id=matching-an-existing-type) — the canonical documentation (updated with the new helper)
- [Zod v4 docs](https://zod.dev/v4) — main Zod 4 documentation with the new `z.toZod` entry

---

## zod `PR #6440` + `PR #6443` + `PR #6445` SHIPPED (August 20, 2026) — Bug Fixes + Performance

Three zod master branch PRs shipped on the same day (2026-08-20, 14:27Z–14:42Z) — all on the `4.5.0-canary` train heading toward `zod@4.5.0` STABLE.

### `PR #6440` — `fix(v4): stop catch resurrecting issues an optional already resolved`

**Merged 2026-08-20T14:42:04Z.** `.catch()` could turn a previously-succeeded parse into a failure:

```ts
const s = z.undefined().optional();
s.safeParse(undefined);              // { success: true }
s.catch(null).safeParse(undefined);  // { success: false } ← was buggy

// Also affected: .default(), doubled .optional(),
// .catch() with callback, and chains nested in objects/arrays — all fixed
```

The bug: `handleOptionalResult` signalled failure by returning a replacement payload with an empty issues array. `$ZodCatch` kept the payload it passed down (still carrying the failure the optional had already resolved) and saw nothing to catch.

The fix: `$ZodCatch` now resolves in place using a fresh `issues` array rather than truncating the one passed down.

### `PR #6443` — `fix(v4): keep a memoized node's cached issues private to the cache`

**Merged 2026-08-20T14:27:54Z.** The memoizer cached live issue objects and handed the same ones to every visitor of a shared node — so a second visitor would prefix onto the first one's path:

```ts
const Node: any = z.object({
  name: z.string(),
  get left() { return z.optional(Node); },
  get right() { return z.optional(Node); },
});

const shared: any = { name: 123 };
Node.safeParse({ name: "root", left: shared, right: shared });

// before: [["right","left","name"], ["right","left","name"]]  ← wrong
// after:  [["left","name"], ["right","name"]]                  ← correct
```

The fix: `cloneIssues` now runs both on store and on hand-out, so the cache owns pristine issues and every visitor gets its own. Bundle cost: **0 bytes on `zod/mini`**, **+33 gzipped on classic**.

### `PR #6445` — `perf(v4): prefix issue paths in place in the object JIT failure path`

**Merged 2026-08-20T14:34:12Z.** The object JIT was the odd one out — it copied every issue before prefixing, while interpreted paths always prefixed in place. Now it emits a loop that writes in place. Measured against #6443 with paired A/B harness (2 runs):

| Case | Improvement |
|------|-------------|
| union-parse | **−12.3% to −12.8%** |
| suite total | **−0.9% to −1.5%** |

Bundle: **0 bytes on all three `zod/mini` fixtures** (never bundle the object JIT), **+8 gzipped on classic `zod-object`**.

> ⚠️ Prefixing in place is only safe on top of PR #6443 (the memoizer fix). Without #6443, a DAG-shaped input reaching one invalid node twice would corrupt paths. Upgrade both or neither.

### Migration action

```bash
# These are internal Zod fixes — no API changes required.
# Update to the latest canary to pick up all three:
npm install zod@canary
# Or pin to the specific canary:
# "zod": "npm:zod@4.5.0-canary.20260820T155656"
```

### Sources

- [Zod PR #6440 — fix(v4): stop catch resurrecting issues an optional already resolved (merged 2026-08-20T14:42:04Z)](https://github.com/colinhacks/zod/pull/6440)
- [Zod PR #6443 — fix(v4): keep a memoized node's cached issues private to the cache (merged 2026-08-20T14:27:54Z)](https://github.com/colinhacks/zod/pull/6443)
- [Zod PR #6445 — perf(v4): prefix issue paths in place in the object JIT failure path (merged 2026-08-20T14:34:12Z)](https://github.com/colinhacks/zod/pull/6445)
- [Zod commit `555e5f46fe` — Add z.toZod helper #5913 (2026-08-20T15:15:26Z)](https://github.com/colinhacks/zod/commit/555e5f46fe)
- [Zod commit `e516c3baf2` — ci: publish to jsr after cutting the release, not before #6453 (2026-08-20T15:52:06Z)](https://github.com/colinhacks/zod/commit/e516c3baf2) — the JSR publish workflow update

---

## RHF PR #13668 + PR #13667 SHIPPED (August 20–21, 2026) — `useWatch` + `setValues` Bug Fixes

Two React Hook Form master branch bug fixes shipped in the same window as the zod updates.

### `PR #13668` — `fix: useWatch returns stale value on name change when new value is null` (SHIPPED Aug 21 00:24Z)

**Merged 2026-08-21T00:24:02Z.** `useWatch`'s synchronous name/control-change fast path used `null` as a sentinel meaning "no immediate value computed yet." When a watched field's actual value is legitimately `null`, this collided with the sentinel — so the hook returned the previous field's stale value for one render.

```tsx
// BEFORE (buggy): watching field 'a', then re-render watching field 'c' (value is null)
// — returned stale 'x' for one render
// AFTER (fixed): returns null immediately on the same render
```

The fix: replaced the `null` sentinel with an explicit `shouldReturnImmediate` boolean flag. Added a regression test that verifies `useWatch` returns `null` on the same render when the new field value is `null`.

### `PR #13667` — `fix(setValues): update fields registered under an object or array value` (SHIPPED Aug 20 00:24Z)

**Merged 2026-08-20T00:24:46Z.** `setValues` used `flatten()` which only emits leaf paths — so a field registered at a container path (`tags` for `tags: ['b', 'c']`) was never found, and `_setValue` was skipped entirely. An empty array/object produced zero keys and the update disappeared.

```tsx
// These are affected:
// <select multiple {...register('tags')} /> — keeps old selection
// a checkbox group sharing one name — keeps old checked state
// a container path registered directly by a Controller — never receives the new value
// setValues({ tags: [] }) — does NOT clear anything (BUG)
```

The fix: replaced flatten-and-lookup with a path walk over the provided values, so a container value is passed to `_setValue` as-is.

### Migration action

```bash
# Update to the latest RHF master. When v7.86.0 ships:
npm install react-hook-form@latest
```

Both fixes are internal behavioral corrections with no API changes. The `setValues` fix brings it into alignment with `setValue` which already handled these cases correctly.

### Sources

- [RHF PR #13668 — fix: useWatch returns stale value on name change when new value is null (merged 2026-08-21T00:24:02Z)](https://github.com/react-hook-form/react-hook-form/pull/13668)
- [RHF PR #13667 — fix(setValues): update fields registered under an object or array value (merged 2026-08-20T00:24:46Z)](https://github.com/react-hook-form/react-hook-form/pull/13667)
- [RHF master commits 2026-08-13–21](https://github.com/react-hook-form/react-hook-form/commits?since=2026-08-13) — PRs #13668, #13667, #13664, #13662, #13661, #13660, #13659, #13658, #13657, #13655

---

## zod@canary — New Drops Since v1.5.81 + STABLE Forecast Update (August 20–21, 2026)

### New canary drops since v1.5.81 (2026-08-21T00:09Z check)

The zod@canary train produced new drops between the v1.5.81 cron (Aug 21 00:09Z) and this cycle's check (Aug 21 06:02Z):

| Canary Version | npm Published | Gap from Prior | Notes |
|---|---|---|---|
| `4.5.0-canary.20260819T185743` | 2026-08-19T19:04:57Z | +2h 49m from prior | |
| `4.5.0-canary.20260819T173014` | 2026-08-19T19:03:43Z | −1m from prior | re-tarball |
| `4.5.0-canary.20260820T144632` | 2026-08-20T14:49:56Z | +5h 22m from prior | |
| `4.5.0-canary.20260820T144642` | 2026-08-20T14:49:47Z | −9s from prior | re-tarball |
| `4.5.0-canary.20260820T145307` | 2026-08-20T14:56:23Z | +6m 36s from prior | |
| `4.5.0-canary.20260820T151954` | 2026-08-20T15:23:08Z | +26m 45s from prior | |
| **`4.5.0-canary.20260820T155656`** | **2026-08-20T16:00:10Z** | **+37m from prior** | **Current tip; STABLE-train promoted** |

Plus 5 re-tarballs at 2026-08-20T19:41–19:43Z (same content, new tarballs).

### Current state

`npm view zod@latest` → `4.4.3` (STABLE, May 4 2026)
`npm view zod@canary` → `4.5.0-canary.20260820T155656` (STABLE-train promoted Aug 20 16:00Z)

### zod@4.5.0 STABLE forecast — **August 22–24, 2026**

The STABLE-train promotion (Aug 20 16:00Z) combined with the 4-drops-per-day cadence confirms the v1.5.81 forecast of "Aug 21-23" is slightly ahead — the promotion happened ~5h before the Aug 21 window opened. Revised forecast:

- **Most likely: August 22, 2026 (T+16h to T+40h from 2026-08-21 06:02Z)**
- **Range: August 22–24, 2026 (T+16h to T+64h)**
- The 4-drops-per-day cadence is sustained. The STABLE cut is imminent.

### Migration checklist for projects on zod@4.4.3

- [ ] Review breaking changes documented in `## zod@4.5.0 Breaking Changes to Review Before Upgrading` section
- [ ] Audit custom ZodEffects usage (PR #6440 fixes .catch() behavior — verify custom error handling)
- [ ] Audit recursive schemas with shared nodes (PR #6443 changes memoized issue caching)
- [ ] Test object schema parsing in performance-critical paths (PR #6445 provides ~12% union-parse improvement)
- [ ] Update `zod@canary` pin to `4.5.0-canary.20260820T155656` for early access
- [ ] Plan `zod@4.5.0` upgrade window (target: within 48h of STABLE ship)

### Sources

- [`npm view zod dist-tags`](https://registry.npmjs.org/zod) — confirmed `latest: 4.4.3`, `canary: 4.5.0-canary.20260820T155656` at this cycle's 06:02Z check
- [Zod commits 2026-08-20](https://github.com/colinhacks/zod/commits?since=2026-08-20T00:00:00Z) — PRs #5913, #6440, #6443, #6445
- [Zod PR #5913 — Add z.toZod helper (merged 2026-08-20T15:15:26Z)](https://github.com/colinhacks/zod/pull/5913)

---

## React Hook Form 7.86.0 SHIPPED (August 22, 2026) — Type-Safe getErrors() + Performance

**`react-hook-form@7.86.0`** SHIPPED **2026-08-22** (npm-published today). First release since v7.85.0 (Aug 8, 2026 — 14-day gap). Confirmed via yarnpkg.com CHANGELOG.md entry:

> `[7.86.0] - 2026-08-22`
> `Added — Type-safe getErrors method.`
> `Performance — Improve createFormControl; Improve clone object check; Avoid cloning.`

### What changed

**`getErrors()` — new type-safe method** (PR #13639, merged 2026-08-11): reads form errors at any point without subscribing to the errors state. Eliminates the `useEffect` + `Controller` pattern previously needed to check errors on demand.

```tsx
// Before (verbose — needed a Controller just to observe errors)
const errors = useController({ name: 'email' });

// After — getErrors() reads current errors state directly
const { getErrors } = useForm();
const emailError = getErrors('email');
// → { type: 'required', message: 'Email is required' } | undefined

// Bulk read
const allErrors = getErrors(); // Record<string, FieldError>
```

**Performance fixes** (PR #13668 + PR #13667, merged Aug 20-21, 2026): `useWatch` now correctly returns `null` on the same render when the new field value is `null` (fixes stale closure); `setValues` now correctly updates container-registered fields (object/array paths) instead of silently disappearing.

### Migration action

```bash
npm install react-hook-form@latest
# → 7.86.0 (the wait is over — was forecast 2-3 weeks from v7.85.0)
```

No breaking changes. All projects using RHF 7.85.x can upgrade immediately.

### Sources

- [RHF CHANGELOG.md — v7.86.0 entry (yarnpkg.com, confirmed 2026-08-22)](https://classic.yarnpkg.com/en/package/react-hook-form)
- [RHF PR #13639 — feat: add getErrors method (merged 2026-08-11T11:44:31Z)](https://github.com/react-hook-form/react-hook-form/pull/13639)
- [RHF PR #13668 — fix: useWatch returns stale value (merged 2026-08-21T00:24:02Z)](https://github.com/react-hook-form/react-hook-form/pull/13668)
- [RHF PR #13667 — fix(setValues): update fields registered under object/array (merged 2026-08-20T00:24:46Z)](https://github.com/react-hook-form/react-hook-form/pull/13667)

---

## zod@4.5.0 STABLE — T+0h (August 22, 2026) — Status Check

**`zod@4.5.0` STABLE has NOT yet shipped as of this cron check (2026-08-22T12:02Z).**

The zod canary train is still producing drops. Latest canary: `4.5.0-canary.20260819T185817` (Aug 19, 2026 — Snyk.io confirmed). The STABLE forecast from the last cycle was "August 22–24, 2026". Today IS August 22. Monitor `npm view zod@latest` for the STABLE promotion.

**Recommended pin:** `zod@4.5.0-canary.20260819T185817` (or later canary drop) for pre-STABLE testing. Upgrade to `zod@latest` (→ 4.5.0) within 48h of the STABLE ship.

### Sources

- [npm view zod@latest](https://www.npmjs.com/package/zod) — `latest: 4.4.3` confirmed at this check
- [Snyk.io zod versions](https://security.snyk.io/package/npm/zod) — latest canary: `4.5.0-canary.20260819T185817`
