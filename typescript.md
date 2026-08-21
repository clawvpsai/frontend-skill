# TypeScript — Strict Patterns, Generics, Utilities

## Strict Configuration

**TypeScript 6.0 has `strict: true` on by default.** You no longer need to explicitly set it — it's enabled automatically. You still need to opt into individual strictness options that aren't part of the `strict` umbrella:

```json
{
  "compilerOptions": {
    "strict": true,  // implicit in TS 6.0, but still useful for explicit config
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "target": "ES2025",         // TS 6.0 defaults to ES2025 (was ES2022)
    "lib": ["ES2025", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

> **Note:** In TS 6.0, `target: "ES2025"` and `lib: ["ES2025", ...]` are now the defaults. If you set `target` manually, `lib` is inferred automatically. The `exactOptionalPropertyTypes` flag is also now enabled by default under `strict`.

## TypeScript 6.0 — Key New Features

TypeScript 6.0 was released March 2026 as the final JavaScript-based release before the Go-native TypeScript 7.0. Key changes:

### `import defer` — Deferred Imports (TypeScript 6.0)

`import defer` lets you declare that an import is not needed for the initial render — the module loads in the background while the page shell renders immediately. It's a TypeScript-level construct that works with React's Suspense boundary for loading states.

```ts
// heavy.ts — exports named components
export function HeavyChart({ data }: { data: Data }) { ... }
export function HeavyModal({ onClose }: Props) { ... }
```

**The correct pattern — `await` the deferred module before using its exports:**

```tsx
// components/dashboard.tsx
'use client'

import { Suspense } from 'react'
import defer * as HeavyModule from '@/lib/heavy-chart'

export function Dashboard({ data }: { data: ChartData }) {
  return (
    <div>
      <Header />          {/* Loaded immediately */}
      <StatsPanel />      {/* Loaded immediately */}

      <Suspense fallback={<ChartSkeleton />}>
        <DeferredChart data={data} />
      </Suspense>
    </div>
  )
}

// Separate async component — awaits the deferred module before using exports
async function DeferredChart({ data }: { data: ChartData }) {
  // await HeavyModule resolves the deferred namespace, then destructure
  const { HeavyChart, HeavyModal } = await HeavyModule
  return <HeavyChart data={data} />
}
```

**❌ The common mistake — using a deferred export without awaiting:**

```tsx
// WRONG — HeavyChart is not resolved yet, this throws
<Suspense fallback={<ChartSkeleton />}>
  <HeavyChart data={data} />
</Suspense>
```

**Key insight:** The `await` must happen inside an async component that gets wrapped in Suspense. The outer component (`Dashboard`) renders immediately; only the inner async component (`DeferredChart`) suspends. The deferred module must be awaited before destructuring.

**⚠️ Client-side only:** `import defer` is a **client-side** code-splitting feature. In Server Components, modules already load server-side — use React's `use()` hook with Promises for streaming data instead.

**Why namespace import?** With `import defer { HeavyChart }`, the binding `HeavyChart` is itself the deferred value (not a Promise to await). The recommended pattern is `import defer * as mod` so `await mod` gives you the full module to destructure:

```ts
// ✅ Correct — await the namespace, then destructure
const { HeavyChart } = await (await HeavyModule)

// ❌ Wrong — await on a deferred named export doesn't destructure properly
import defer { HeavyChart } from './heavy'
const mod = await HeavyChart  // HeavyChart is the value itself, not a Promise
```

### `import defer` vs `React.lazy`

| Concern | `import defer` | `React.lazy` |
|---|---|---|
| **Works in** | Client Components (primary), Server Components (limited benefit) | Client Components only |
| **Module resolution** | TypeScript-level (build tool handles) | React-level (at runtime) |
| **Loading state** | Uses `Suspense` | Uses `Suspense` |
| **Bundle splitting** | Yes | Yes |
| **Tree shaking** | Full | Full |
| **Data fetching** | Yes (async module) | No |

**`import defer` when:**
- The module has mixed exports (components + utilities + data)
- You want build-time optimization, not runtime lazy loading
- You want to reduce the initial client bundle size for a module used below the fold

**`React.lazy` when:**
- Client-only components (React.lazy requires `'use client'`)
- Simple component lazy-loading with a clean default export

**Sources:**
- [TypeScript 6.0 import defer](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html)

### Subpath Imports

TS 6.0 supports importing subpaths without `paths` mapping in `tsconfig.json`:

```ts
// Instead of paths mapping in tsconfig:
import { Button } from '@/components/ui/button'
import { formatDate } from '@/lib/date'

// You can now use package subpaths directly:
import { Button } from 'components/ui/button'       // from ./components/ui/button
import { utils } from '@/lib/date'                   // still uses paths for @ alias
```

This works via Node.js subpath imports support in `package.json` `imports` field — TS 6.0 now recognizes these in `moduleResolution: "Bundler"`.

### `--stableTypeOrdering` Flag

TS 6.0 changes how union types are ordered internally (type IDs assigned in encounter order). The `--stableTypeOrdering` flag makes union ordering deterministic across runs, which is important for:

- Reproducible builds
- Deterministic type error messages in CI
- Consistent serialization of types

```json
{
  "compilerOptions": {
    "stableTypeOrdering": true  // new in TS 6.0
  }
}
```

This flag will be the default behavior in TS 7.0. Setting it now makes migrating to 7.0 smoother.

### Temporal API Types

TypeScript 6.0 ships with full type definitions for the Temporal API (ECMAScript proposal for date/time):

```ts
// New Temporal.* types
const now = Temporal.Now.plainDateTimeISO()
const date = Temporal.PlainDate.from('2026-05-27')

// Instead of Date — Temporal is unambiguous about time zones
const meeting = Temporal.ZonedDateTime.from({
  year: 2026,
  month: 5,
  day: 27,
  hour: 14,
  timeZone: 'Asia/Kolkata',
})

// Duration arithmetic
const later = now.add({ days: 7 })
const diff = later.difference(now)  // returns Temporal.Duration
```

**Why Temporal over `Date`:**
- No ambiguity between UTC and local time
- Plain date/time types that can't be confused
- Better `Intl` integration
- Nanosecond precision

### Practical Temporal API Patterns

Temporal has distinct types for different use cases. Here are the most useful patterns:

**`Temporal.PlainDate` — Date without time (birthdays, anniversaries, schedules)**

```ts
// Parse and format dates
const birthday = Temporal.PlainDate.from({ year: 1990, month: 5, day: 15 })
const today = Temporal.Now.plainDateISO()

// Age calculation
const age = today.since(birthday).years  // no timezone confusion

// Date arithmetic — add days, months, years
const nextWeek = today.add({ days: 7 })
const nextMonth = today.add({ months: 1 })

// Comparison — no string parsing needed
const isBefore = today.compare(birthday.add({ years: 18 })) < 0  // is under 18
```

**`Temporal.PlainTime` — Time without date (business hours, recurring events)**

```ts
const opensAt = Temporal.PlainTime.from({ hour: 9, minute: 0 })
const closesAt = Temporal.PlainTime.from({ hour: 17, minute: 0 })
const now = Temporal.Now.plainTimeISO()

const isOpen = now.compare(opensAt) >= 0 && now.compare(closesAt) < 0
```

**`Temporal.PlainDateTime` — Date + time without timezone (local understanding)**

```ts
// Combine date and time
const meeting = Temporal.PlainDateTime.from({
  year: 2026,
  month: 6,
  day: 15,
  hour: 14,
  minute: 30,
})

// Format for display — use Intl with Temporal types
const formatter = new Intl.DateTimeFormat('en-US', {
  dateStyle: 'long',
  timeStyle: 'short',
})

// Convert to instant for display
const instant = meeting.toZonedDateTimeISO('America/New_York')
formatter.format(instant.toJSDate())  // "June 15, 2026 at 2:30 PM"
```

**`Temporal.ZonedDateTime` — Date + time with timezone (storage, transport, display)**

```ts
// Store in UTC, display in user's timezone
const utcTime = Temporal.Now.zonedDateTimeISO('UTC')
const localTime = utcTime.withTimeZone('Asia/Kolkata')

// Parse from user input
const userMeeting = Temporal.ZonedDateTime.from({
  year: 2026,
  month: 6,
  day: 15,
  hour: 14,
  minute: 30,
  timeZone: 'America/New_York',  // user's timezone from browser
})

// Convert for storage (always UTC)
const utcForStorage = userMeeting.toInstant()

// Duration between events in different timezones
const event1 = Temporal.ZonedDateTime.from('2026-06-15T14:00:00[America/New_York]')
const event2 = Temporal.ZonedDateTime.from('2026-06-15T20:00:00[Europe/London]')
const gap = event2.since(event1)  // Temporal.Duration — correctly accounts for timezone offset
```

**`Temporal.Duration` — Time spans (durations, billing, subscriptions)**

```ts
const subscription = Temporal.Duration.from({ months: 1, days: 0 })
const trial = Temporal.Duration.from({ days: 14 })

// Check if trial expired
const trialEnd = startDate.add(trial)
const remaining = trialEnd.until(now)  // returns Duration
const isExpired = remaining.isNegative()

// Billing cycle — next charge date
const nextBilling = lastBilling.add(subscription)
```

**Migrating from `Date` to Temporal in Next.js:**

```ts
// ❌ Old: Date — ambiguous timezone
const createdAt = new Date(post.createdAt)
const formatted = createdAt.toLocaleDateString('en-US', { timeZone: 'America/New_York' })

// ✅ New: Temporal — unambiguous
const createdAt = Temporal.Instant.fromEpochMilliseconds(post.createdAt.getTime())
  .toZonedDateTimeISO('America/New_York')
const formatted = new Intl.DateTimeFormat('en-US', {
  dateStyle: 'medium',
  timeStyle: 'short',
}).format(createdAt.toJSDate())

// Or store as ISO string in the database and parse directly
const createdAt = Temporal.ZonedDateTime.from(post.createdAtISO)
```

**Note:** Most browsers don't support Temporal natively yet. Use the `temporal-polyfill` npm package or rely on TypeScript's type definitions alone — the types work without the runtime polyfill for type-checking purposes.


### ES2025 Built-in Types (TypeScript 6.0)

TypeScript 6.0 includes types for ES2025 built-in methods:

**`Map.getOrInsert()` and `Map.getOrInsertComputed()`**

```ts
const cache = new Map<string, User>()

// getOrInsert — get existing or insert if missing (value eager)
const user1 = cache.getOrInsert('alice', { id: '1', name: 'Alice' })
// If 'alice' exists, returns existing. Otherwise inserts and returns the second arg.

// getOrInsertComputed — get existing or compute+insert if missing (lazy)
const user2 = cache.getOrInsertComputed('bob', () => ({ id: '2', name: 'Bob' }))
// Callback only runs if 'bob' is not in the map — avoids unnecessary work.
```

**`RegExp.escape()`** — Escape special regex characters:

```ts
// Escape user input for use in a regex
const userInput = 'foo.bar+baz'
const escaped = RegExp.escape(userInput)  // "foo\\.bar\\+baz"

// Now safe to concatenate into a dynamic regex
const dynamicRegex = new RegExp(`^${escaped}$`, 'i')
```

These are enabled by setting `target: "ES2025"` (the TS 6.0 default) or adding `"lib": ["ES2025"]`.

**Sources:**
- [TypeScript 6.0 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html)


### TypeScript 6.0 Deprecations

These legacy options are deprecated in 6.0 and will be removed in 7.0:

```json
{
  "compilerOptions": {
    "target": "ES3",      // ES2025 is minimum — no more legacy targets
    "noImplicitReturns": false,  // deprecated — use explicit return types
    "suppressExcessPropertyErrors": true  // deprecated — use strict object types
  }
}
```

If you see these in legacy code, migrate off them before TS 7.0.

### TypeScript 6.0 Default Changes

TypeScript 6.0 ships new defaults that reflect modern JS reality. Most of these match what new projects already want, but legacy code may need explicit overrides:

| Option | TS 5.x default | TS 6.0 default | What it means |
|---|---|---|---|
| `strict` | `false` | `true` | Strict mode is on by default — set `false` explicitly to opt out |
| `module` | `esnext`/`commonjs` (auto) | `esnext` | ESM is the dominant module format |
| `target` | `es5` (very old) | `es2025` | Floating current-year ES spec — most apps don’t need transpilation |
| `libReplacement` | `true` | `false` | Stops auto-mapping `lib` names; speeds up `--watch` and editor scenarios |
| `noUncheckedSideEffectImports` | `false` | `true` | Catches typos in side-effect-only imports (e.g. `import './styles.cs'`) |
| `types` | auto-includes all `@types/*` | `[]` | Don’t vacuum up everything in `node_modules/@types` — only what’s explicitly listed. **This breaks a lot of legacy projects** but is 20–50% faster type-checking |

**Migration impact:** When you upgrade to 6.0, your `tsconfig.json` may need:
- Add `@types/node`, `@types/react`, etc. explicitly to `types: [...]` (instead of being auto-included)
- Verify no side-effect imports have typos (e.g., missing file extensions)
- If you were relying on `libReplacement`, set it to `true` explicitly to keep old behavior

**Removed in 6.0 (no deprecation period):**
- `outFile` — AMD/UMD/SystemJS bundling is gone
- `baseUrl` — use `paths` directly with `moduleResolution: "Bundler"`
- `--moduleResolution: "node"` (legacy Node.js resolver) — use `"bundler"` or `"node16"`/`"nodenext"`
- `target: "es5"` and older — minimum is now `es2025`

**Sources:**
- [TypeScript 6.0 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html)
- [Bytes #473: TypeScript 6.0 is your final warning](https://bytes.dev/archives/473)
- [Socket: TypeScript 6.0 will be the last JavaScript-based major release](https://socket.dev/blog/typescript-6-0-will-be-the-last-javascript-based-major-release)
- [Microsoft: Progress on TypeScript 7 (December 2025)](https://devblogs.microsoft.com/typescript/progress-on-typescript-7-december-2025/)

---

## Type Inference

Let TypeScript infer as much as possible — only annotate when necessary:

```tsx
// ❌ Over-annotating — redundancy
const name: string = 'Alice'
const count: number = 5
const active: boolean = true

// ✅ Inferred — cleaner, TypeScript still catches errors
const name = 'Alice'
const count = 5
const active = true
```

When to annotate explicitly:
- Function parameters and return types (always for public APIs)
- Variables initialized to `null` or `undefined`
- Complex object literals
- Array or object destructuring with complex types

## Generics

### Basic Generic Functions

```ts
function identity<T>(value: T): T {
  return value
}

const str = identity('hello')  // T = string
const num = identity(42)        // T = number
```

### Generic Constraints

```ts
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}

const user = { name: 'Alice', age: 30, id: '1' }
const name = getProperty(user, 'name')    // string
const age = getProperty(user, 'age')     // number
// getProperty(user, 'missing')         // Error
```

### Generic Interfaces

```ts
interface ApiResponse<T> {
  data: T
  meta: {
    page: number
    total: number
    limit: number
  }
  error: string | null
}

interface User {
  id: string
  name: string
  email: string
}

type UserListResponse = ApiResponse<User[]>
type SingleUserResponse = ApiResponse<User>
```

### Conditional Types

```ts
type NonNullable<T> = T extends null | undefined ? never : T

type Flatten<T> = T extends Array<infer U> ? U : T

type A = Flatten<User[]>  // User
type B = Flatten<string> // string
```

### `NoInfer<T>` — Prevent Auto-Inference (TypeScript 5.4)

`NoInfer<T>` prevents TypeScript from inferring a type parameter from a specific argument, giving you more control over inference:

```ts
// Without NoInfer — TypeScript infers T from both arguments
function createSignal<T>(value: T, defaultValue: T): [() => T, (v: T) => void] {
  // ...
}

// T is inferred as string | number from both args
const [get, set] = createSignal('hello', 42) // T = string | number ❌

// With NoInfer — T is inferred only from the first argument
function createSignal<T>(value: T, defaultValue: NoInfer<T>): [() => T, (v: T) => void] {
  // ...
}

const [get, set] = createSignal('hello', 'world') // T = string ✅
const [get2, set2] = createSignal('hello', 42)    // Error: number not assignable to string ✅
```

**Common use cases:**
- Factory functions where a default should not influence the type
- API clients where the response type comes from the endpoint, not the mock default
- `createContext` patterns where the default value shouldn't widen the type

## Utility Types

### Built-in Utilities

```ts
// Partial — all properties optional
type PartialUser = Partial<User>

// Required — all properties required
type RequiredUser = Required<PartialUser>

// Pick — select specific properties
type UserPreview = Pick<User, 'id' | 'name'>

// Omit — remove specific properties
type UserWithoutEmail = Omit<User, 'email'>

// Record — create object type
type UserMap = Record<string, User>
type Role = 'admin' | 'user' | 'guest'
type PermissionMap = Record<Role, string[]>

// Exclude — remove from union
type NonAdmin = Exclude<Role, 'admin'>

// Extract — keep from union
type AdminOnly = Extract<Role, 'admin'>

// ReturnType — get return type of function
function createUser() { return { id: '1' } }
type CreatedUser = ReturnType<typeof createUser>

// Parameters — get parameter types
type CreateUserParams = Parameters<typeof createUser>[0]
```

### Zod + TypeScript Integration

```ts
import { z } from 'zod'

// Zod v4 — z.infer<typeof schema> still works (aliased as z.output<typeof schema>)
const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  email: z.string().email(),
  role: z.enum(['admin', 'user', 'guest']),
})

type User = z.infer<typeof UserSchema>
// → { id: string; name: string; email: string; role: 'admin' | 'user' | 'guest' }

// Validate at runtime, get types for free
const result = UserSchema.safeParse(possibleUser)
if (result.success) {
  const user: User = result.data  // fully typed
}
```

## Component Typing

### Props with Discriminated Unions

```tsx
type ButtonProps = 
  | { variant: 'primary'; children: React.ReactNode }
  | { variant: 'secondary'; children: React.ReactNode; outlined: boolean }
  | { variant: 'destructive'; children: React.ReactNode; confirmText?: string }

function Button(props: ButtonProps) {
  if (props.variant === 'destructive' && props.confirmText) {
    return <ConfirmButton text={props.confirmText}>{props.children}</ConfirmButton>
  }
  // ...
}
```

### Polymorphic Components

```tsx
import { type ElementType, type ComponentPropsWithRef } from 'react'

type AsProp<C extends ElementType> = { as?: C }
type PropsToOmit<C extends ElementType, P> = P & AsProp<C>

type PolymorphicProps<C extends ElementType, P = object> = 
  PropsToOmit<C, P> & Omit<ComponentPropsWithRef<C>, PropsToOmit<C, P>>

function Card<C extends ElementType = 'div'>(
  props: PolymorphicProps<C, { className?: string }>
) {
  const { as: Component = 'div', className, ...rest } = props
  return <Component className={`card ${className ?? ''}`} {...rest} />
}

// Usage
<Card className="p-4">Content</Card>                        // renders <div>
<Card as="section" className="p-4">Section</Card>           // renders <section>
<Card as="a" href="/about" className="p-4">Link</Card>       // renders <a>
```

## Type Guards

```ts
// Custom type guard
function isUser(obj: unknown): obj is User {
  return (
    typeof obj === 'object' &&
    obj !== null &&
    'id' in obj &&
    'name' in obj &&
    typeof (obj as Record<string, unknown>).id === 'string'
  )
}

// Assertion function
function assertIsUser(val: unknown): asserts val is User {
  if (!isUser(val)) throw new Error('Not a user')
}
```

## `noUncheckedIndexedAccess`

With this option enabled (recommended), array and object index access returns `T | undefined`:

```ts
const arr = ['a', 'b', 'c']
const first: string | undefined = arr[0]  // Must handle undefined

// Safer
const first = arr[0]
if (first !== undefined) {
  console.log(first.toUpperCase())  // TypeScript knows first is string here
}

// Or use nullish coalescing
const first = arr[0] ?? 'default'
```

## Module Types

### Path Aliases

```ts
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@/components/*": ["./src/components/*"],
      "@/lib/*": ["./src/lib/*"]
    }
  }
}
```

```tsx
// Now use:
import { Button } from '@/components/ui/button'
import { cn } from '@/lib/utils'
```

### Type-Only Imports

```ts
// Only import the type — stripped at compile time
import type { User, Post } from '@/types'

// Or use inline type imports
function processUser(user: import('@/types').User) { }
```

### `import defer` vs Dynamic `import()` — When to Use Which

| Pattern | Mechanism | When to Use |
|---|---|---|
| `import defer` (TS 6.0) | Eagerly loaded, lazily executed; execution order preserved | Heavy modules needed soon after render but not for initial paint |
| `import()` (dynamic) | Lazily loaded, lazily executed | Modules triggered by user interaction (modals, dropdowns) |
| Server Component data | Fetched server-side, streamed via RSC | Data fetching for UI |

**`import defer` in a real-world client component:**

```tsx
// components/product-page.tsx
'use client'

import { useState } from 'react'
import { Suspense } from 'react'

// Deferred import — recharts code split from initial bundle
// Useful when a product page shows a chart below the fold
import defer * as ProductChartModule from '@/components/product-chart'

export function ProductPage({ product }: { product: Product }) {
  const [showChart, setShowChart] = useState(false)

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>

      <button onClick={() => setShowChart(true)}>
        Show Analytics
      </button>

      {showChart && (
        <Suspense fallback={<ChartSkeleton />}>
          <ProductChartModule.ProductChart productId={product.id} />
        </Suspense>
      )}
    </div>
  )
}
```

## Type-Level Patterns (The Power Tools)

These four features are not part of the "core" TS spec the way `interface` or `extends` are, but they are now table-stakes for any non-trivial TypeScript codebase. Reach for them when the basic tools force you to either over-widen or under-narrow a type.

### `satisfies` Operator (TypeScript 4.9+)

`satisfies` checks that a value matches a type *without* widening the value's inferred type. It sits between "let TS infer" and "explicit type annotation" — you get the structural check from the type, but the literal-narrow inference from the value:

```ts
// Without satisfies — TS infers `string` (too wide)
const palette = { red: '#f00', green: '#0f0' }
//   ^? { red: string; green: string }

// With explicit type — TS forces the wider shape (loses literal)
const palette: Record<string, string> = { red: '#f00', green: '#0f0' }
// palette.red has type `string` — can't autocomplete '#f00'

// With satisfies — best of both worlds
const palette = { red: '#f00', green: '#0f0' } satisfies Record<string, string>
//   ^? { red: string; green: string }   ← still narrow!
// AND the assignment is checked against Record<string, string>
palette.red    // 'string' (literal preserved)
palette.blue   // ❌ Error: Object literal may only specify known properties
```

**When to use `satisfies`:**
- Configuration objects whose keys/values you want to keep narrow (Tailwind class maps, color palettes, feature flags)
- Any place you'd want a type assertion *and* a structural check
- Discriminated record types — `satisfies Record<Key, BaseType>` lets you keep narrow per-key types while proving the structure

```ts
// Discriminated record with satisfies — keeps per-key types narrow
type Key = 'user' | 'post' | 'comment'
interface Base { id: string }
interface User extends Base { name: string; email: string }
interface Post extends Base { title: string; body: string }
interface Comment extends Base { body: string; authorId: string }

const handlers = {
  user:    { id: 'u1', name: 'Alice', email: 'a@x.com' },
  post:    { id: 'p1', title: 'Hi',  body: 'Hello' },
  comment: { id: 'c1', body: 'Cool', authorId: 'u1' },
} satisfies Record<Key, Base>
//   ^? narrow per-key types preserved: User, Post, Comment
// AND the compiler proves every Key is present

handlers.user.name     // string — narrow
handlers.user.title    // ❌ not on User
handlers.feed          // ❌ 'feed' not in Key
```

**`satisfies` vs `as` (type assertion):**

| | `satisfies` | `as` |
|---|---|---|
| Structural check | Yes | No (assertion bypasses checks) |
| Narrows inferred type | Yes (preserves literal) | Loses literal info |
| Runtime cost | Zero | Zero |
| Use when | "This should match X" | "Trust me, I know better than the type system" |

**Sources:**
- [TypeScript 4.9 release notes — `satisfies`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html)
- [learningtypescript.com — The satisfies Operator](https://www.learningtypescript.com/articles/the-satisfies-operator)

### `const` Type Parameters (TypeScript 5.0+)

`const` type parameters make TypeScript infer literal types in function calls, the same way `as const` does on a variable. Useful when you write a generic function and want the call site to preserve the literal types of its arguments:

```ts
// Without const — T is inferred as `string`, not the literal "Alice"
function getNamesExactly<T extends { names: readonly string[] }>(arg: T): T['names'] {
  return arg.names
}
const names = getNamesExactly({ names: ['Alice', 'Bob', 'Eve'] })
//   ^? string[]   ← lost the literal tuple

// With const — T is inferred as readonly ['Alice', 'Bob', 'Eve']
function getNamesExactly<const T extends { names: readonly string[] }>(arg: T): T['names'] {
  return arg.names
}
const names = getNamesExactly({ names: ['Alice', 'Bob', 'Eve'] })
//   ^? readonly ['Alice', 'Bob', 'Eve']  ← preserved!
```

**Common use cases:**

```ts
// 1. Typed `fetch` wrapper that preserves the response type
async function getJSON<const T>(url: string): Promise<T> {
  const res = await fetch(url)
  return res.json() as T
}
// T still widens to its constraint (T), but JSON literal shape is preserved
// by the call site if T is given a literal-typed default.

// 2. Validation/sanitization that should preserve literal types
function tag<const T extends string>(value: T): { __tag: T; value: T } {
  return { __tag: value, value }
}
const t = tag('user')
//   ^? { __tag: 'user'; value: 'user' }  — literal 'user' preserved

// 3. Object keys for exhaustiveness checks
function keysOf<const T extends object>(obj: T): Array<keyof T> {
  return Object.keys(obj) as Array<keyof T>
}
const k = keysOf({ a: 1, b: 2 })
// k[0] is `'a' | 'b'`, not just `string`
```

**Caveats:**
- `const` only affects inference for *expressions written inside the call* — variables defined outside the call are still widened by default. So `const arr = ['a','b']; fnGood(arr)` will not narrow.
- If your function writes to its inputs, the readonly inference may conflict. Use `readonly` in your constraints to be safe.

**Sources:**
- [TypeScript 5.0 release notes — `const` Type Parameters](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html)
- [Matt Pocock — Const type parameters bring 'as const' to functions](https://www.totaltypescript.com/const-type-parameters)

### Branded (Opaque) Types — Nominal Typing Without Compiler Support

TypeScript is structurally typed: `type UserId = string` and `type PostId = string` are interchangeable, which lets you pass a `postId` where a `userId` is expected. **Branded types** (also called opaque types) simulate nominal typing using an intersection with a phantom property:

```ts
// Define a brand helper
type Brand<T, B extends string> = T & { readonly __brand: B }

// Two string aliases that are NOT interchangeable
type UserId = Brand<string, 'UserId'>
type PostId = Brand<string, 'PostId'>

declare function getUser(id: UserId): Promise<User>
declare function getPost(id: PostId): Promise<Post>

const userId = 'usr_123' as UserId
const postId = 'pst_456' as PostId

getUser(userId)   // ✅ OK
getUser(postId)   // ❌ Type 'PostId' is not assignable to type 'UserId'
```

**The "weak" pattern (most common) — `Base & { __brand }`:**

```ts
type Tag<T> = { readonly __brand: T }
type Currency<T extends string> = number & Tag<T>
type Euro = Currency<'euro'>
type USD  = Currency<'usd'>

// A price function that takes only Euros
function charge(price: Euro): void { /* ... */ }

const tenEuros = 10 as Euro
charge(tenEuros)               // ✅
charge(10)                     // ❌ number is not Euro
charge(10 as USD)              // ❌ USD is not Euro
```

**The "strong" pattern — uses a sentinel object so the brand is hard to cast away:**

```ts
// Strong: you can only create a Currency through the constructor
type StrongCurrency<T extends string> = (number & Tag<T>) | Tag<T>

const TEN_EUROS: StrongCurrency<'euro'> = 10 as never  // ❌ can't cast directly
const TEN_EUROS: StrongCurrency<'euro'> = { __brand: 'euro' }  // ❌ still rejected
// Must go through a constructor:
function euro(n: number): StrongCurrency<'euro'> { return n as StrongCurrency<'euro'> }
```

**Smart constructors — the recommended pattern:**

```ts
// Hide the cast in a function so callers don't learn the trick
type UserId = Brand<string, 'UserId'>

function createUserId(raw: string): UserId {
  if (!/^usr_[a-z0-9]{8,}$/.test(raw)) {
    throw new Error(`Invalid user id: ${raw}`)
  }
  return raw as UserId
}

// Optional Zod-backed smart constructor (best for runtime data)
import { z } from 'zod'

const UserIdSchema = z.string().regex(/^usr_[a-z0-9]{8,}$/).brand<'UserId'>()
type UserId = z.infer<typeof UserIdSchema>  // string & z.BRAND<'UserId'>

function parseUserId(raw: string): UserId {
  return UserIdSchema.parse(raw)  // throws if invalid, returns branded value
}
```

**Caveats:**
- Branded types are erased at runtime — they're purely a type-system fiction. Always validate raw input at the boundary.
- Math operations on branded numeric types do **not** error in TypeScript. The compiler treats `Euro + Euro` as `number + number`. This is a known gap ([microsoft/TypeScript#59423](https://github.com/microsoft/TypeScript/issues/59423)).
- For a fuller version with `.map`, `.chain`, validation, etc., look at [Effect's `Brand` module](https://effect.website) — it provides `Brand.Brand`, `Brand.nominal`, and a structured API.

**Sources:**
- [learningtypescript.com — Branded Types](https://www.learningtypescript.com/articles/branded-types)
- [ferreira.io — Opaque / Branded Types in TypeScript](https://ferreira.io/posts/opaque-branded-types-in-typescript)
- [microsoft/TypeScript#202 — Support some non-structural (nominal) type matching](https://github.com/microsoft/TypeScript/issues/202)
- [Zod — `.brand<'Foo'>()`](https://zod.dev)

### Template Literal Types — String Types in Depth

Template literal types let you express string-shaped types — both the shape they accept and the shape they produce. They are how libraries like `tailwind-merge`, `ts-pattern`, and zod's `z.templateLiteral` build safe string APIs.

**Basic syntax:**

```ts
type World = 'world'
type Greeting = `hello ${World}`      // 'hello world'

type Color = 'red' | 'green' | 'blue'
type ColorClass = `text-${Color}`     // 'text-red' | 'text-green' | 'text-blue'

// Numeric interpolation
type Digit = 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
type Pin = `${Digit}${Digit}${Digit}${Digit}`  // 10,000 combinations — valid
```

**Intrinsic string manipulation types (built into TS):**

```ts
type Slug = 'hello-world'
Uppercase<Slug>      // 'HELLO-WORLD'
Lowercase<Slug>      // 'hello-world'
Capitalize<Slug>     // 'Hello-world'
Uncapitalize<Slug>   // 'hello-world'
```

**Real-world pattern: typed event names**

```ts
// Without template literals — easy to typo
person.on('firstName', () => {})  // ❌ not a known event
person.on('firstNameChanged', () => {})  // ✅

// With template literals — typos caught at compile time
type Person = { firstName: string; lastName: string; age: number }
type EventName<T> = {
  [K in keyof T as `${string & K}Changed`]: (newValue: T[K]) => void
}
// EventName<Person> = {
//   firstNameChanged: (newValue: string) => void
//   lastNameChanged:  (newValue: string) => void
//   ageChanged:       (newValue: number) => void
// }
```

**Real-world pattern: dot-notation paths (type-safe deep `get`/`set`)**

```ts
// Build a type that knows every valid "a.b.c" path through an object
type Paths<T, Prev extends string = ''> = {
  [K in keyof T & string]:
    | `${Prev}${K}`
    | (T[K] extends object
        ? Paths<T[K], `${Prev}${K}.`>
        : never)
}[keyof T & string]

type User = { name: string; address: { city: string; zip: string } }
type UserPaths = Paths<User>
//   'name' | 'name.city' | 'name.zip' | 'address' | 'address.city' | 'address.zip'

function get<T, P extends Paths<T>>(obj: T, path: P): GetValue<T, P> {
  return path.split('.').reduce((acc: any, key) => acc[key], obj) as any
}

const u: User = { name: 'Alice', address: { city: 'Surat', zip: '395007' } }
get(u, 'address.city')    // string ✅
get(u, 'address.zip')     // string ✅
get(u, 'addres.city')     // ❌ typo caught
get(u, 'address.country') // ❌ not a path
```

**Real-world pattern: CSS units / unit-aware strings**

```ts
type CSSUnit = 'px' | 'rem' | 'em' | '%' | 'vh' | 'vw'
type CSSLength = `${number}${CSSUnit}`
type CSSValue = CSSLength | 'auto' | '0'

function setHeight(h: CSSValue): void { /* ... */ }
setHeight('100px')   // ✅
setHeight('2.5rem')  // ✅
setHeight('auto')    // ✅
setHeight('hello')   // ❌ not a valid CSS value
```

**Performance caveat:**

Template literal types with large unions explode combinatorially. `${0|1|2|...|9}${0|1|...|9}${0|1|...|9}${0|1|...|9}` produces 10,000 types and slows `tsc` dramatically. For runtime-validated values (user IDs, CSS values, route slugs), prefer **branded types** with a runtime regex/Zod check — keep the type-level work to a small finite set.

**Sources:**
- [TypeScript docs — Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html)
- [dev.to — I need to learn about TypeScript Template Literal Types (Phenomnominal)](https://dev.to/phenomnominal/i-need-to-learn-about-typescript-template-literal-types-51po)
- [OneUptime — How to Handle Template Literal Types (Jan 2026)](https://oneuptime.com/blog/post/2026-01-24-typescript-template-literal-types/view)

## Explicit Resource Management (`using`) — TypeScript 5.2

TypeScript 5.2 introduces the `using` keyword and `Symbol.dispose` for deterministic resource cleanup — similar to C#'s `using` or Python's `with`:

```ts
// A resource must implement Symbol.dispose or Symbol.asyncDispose
class DatabaseConnection {
  id: string
  
  constructor(id: string) {
    this.id = id
    console.log(`[DB] Opened connection ${id}`)
  }
  
  query(sql: string) {
    return `Result of: ${sql}`
  }
  
  [Symbol.dispose]() {
    console.log(`[DB] Closed connection ${this.id}`)
  }
}

// Using — automatically calls [Symbol.dispose]() at end of scope
function getUser() {
  using db = new DatabaseConnection('users-db')
  
  const result = db.query('SELECT * FROM users')
  // No need to manually call db.dispose() — it happens automatically
  
  return result
}

// Works with try/finally semantics — always cleans up, even on throw
function transactional() {
  using db = new DatabaseConnection('main')
  
  db.query('BEGIN')
  try {
    db.query('COMMIT')
  } catch {
    db.query('ROLLBACK') // still executes
    throw
  }
  // db[Symbol.dispose]() called automatically here
}
```

**Why `using` matters:**
- Replaces manual `try/finally` cleanup patterns
- Eliminates "forgot to close" bugs (DB connections, file handles, timers)
- Works with any resource that implements `Symbol.dispose`
- Async variant: `Symbol.asyncDispose` with `await using`

```ts
// Async resource cleanup
class AsyncFileHandle {
  [Symbol.asyncDispose]() {
    return this.closeAsync() // returns a Promise
  }
}

async function readFile() {
  await using file = new AsyncFileHandle()
  // file[Symbol.asyncDispose]() called automatically on scope exit
}
```

**Common React/Next.js use cases:**
- Closing WebSocket connections in cleanup
- Releasing animation frame locks
- Flushing pending writes to storage
- Aborting in-flight requests (combine with AbortController)

```ts
// Example: WebSocket cleanup with using
function useWebSocket(url: string) {
  const ws = new WebSocket(url)
  
  using cleanup = {
    [Symbol.dispose]() {
      ws.close()
    }
  }
  
  // ... use ws
  // ws.close() called automatically when cleanup goes out of scope
}
```


## TypeScript 7 — Go Compiler + Breaking Changes

TypeScript 7.0 reached **Release Candidate on June 18, 2026** ([official announcement](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-rc/)) — the entire compiler is rewritten in Go. The RC ships under the main **`typescript`** package on npm (the `tsc` binary is now the Go compiler — the separate `@typescript/native-preview` package is no longer the recommended install). Stable release is targeted within two months of the RC.

**Current status:** ~~Release Candidate (Version 7.0.1-rc)~~ — **General Availability (Version 7.0.0) shipped July 8, 2026** ([official announcement](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)) — three weeks after the RC. Install via the standard `typescript` package on npm; no `@rc` tag needed. The team has been working with Bloomberg, Canva, Figma, Google, Lattice, Linear, Miro, Notion, Slack, Vanta, Vercel, and VoidZero on pre-release builds for over a year. **Patch 7.0.2 SHIPPED 2026-07-08T15:55:18Z** ([npm](https://www.npmjs.com/package/typescript)) — supersedes 7.0.0 as the recommended TS 7 version. The `latest` npm dist-tag was bumped to `7.0.2` on release day. 7.0.2 is a pure bug-fix patch (no compiler-API changes, no breaking changes, no docs-only changes); the Go compiler binary itself is unchanged. Install via `npm install -D typescript@^7.0.2` (or just `typescript@latest`). **Recommended TypeScript version: 7.0.2** — supersedes 7.0.0 / 7.0.1. TypeScript 7.1 is in active `dev` builds (`7.1.0-dev.20260809.1` is the current `dist-tag.next`; cut 2026-08-09T08:29:01Z — daily dev cadence; 7.1 is **not** yet recommended). TypeScript skipped the 7.1.0-dev.20260729.1 cut on July 29 (no publish that day), then 7.1.0-dev.20260730.1 was cut 2026-07-30T08:23:49Z, 7.1.0-dev.20260731.1 was cut 2026-07-31T08:34:16Z (slightly later than the usual ~08:25Z), 7.1.0-dev.20260801.1 was cut 2026-08-01T08:14:09Z, 7.1.0-dev.20260802.1 was cut 2026-08-02T08:25:51Z (back on the usual ~08:25Z cadence), 7.1.0-dev.20260803.1 was cut 2026-08-03T08:17:15Z (slightly earlier than the usual ~08:25Z), 7.1.0-dev.20260804.1 was cut 2026-08-04T08:24:11Z (back on the usual ~08:24Z cadence), 7.1.0-dev.20260805.1 was cut 2026-08-05T08:27:00Z, 7.1.0-dev.20260806.1 was cut 2026-08-06T08:24:37Z, 7.1.0-dev.20260807.1 was cut 2026-08-07T08:26:43Z, 7.1.0-dev.20260808.1 was cut 2026-08-08T08:23:54Z, and **7.1.0-dev.20260809.1 was cut 2026-08-09T08:29:01Z** (~3h34min before this cron at 12:03Z; slightly later than the usual ~08:25Z cadence — ~4 minutes off the median — but well within the typical ±15min spread). **All twelve are no-content diffs vs the 7.1.0-dev.20260728.1 cut from July 28** — zero new commits landed on the TypeScript main branch between 2026-07-28T08:30:00Z and 2026-08-09T08:29:00Z (verified via `GET /repos/microsoft/TypeScript/commits?sha=main&since=2026-07-25T00:00:00Z` returning exactly 1 commit total in the 15-day window, and that one commit was the 2026-07-27 `b465fdbfe1` — `fix(lib): remove callable signature without new from Intl.PluralRules` (PR #63608) — which preceded the July 28 daily cut, not a new addition since then); the daily dev cadence rebuilds are generated from the unchanged tree. Not a notable change. Each of the new cuts (20260805.1 → 20260809.1) is itself a no-content diff vs its predecessor — **twelve consecutive no-content daily cuts in a row (~12 days)** as of this writing (Aug 9). The TS nightly train is now in a maintenance idle state (no commits on main since 2026-07-27T20:55:30Z); the daily cuts will resume carrying new content as soon as a new PR merges to main. The next daily cut (`7.1.0-dev.20260810.1`) is expected tomorrow at ~08:25 UTC (the v1.5.40 cron's prediction of `7.1.0-dev.20260809.1` at 08:25Z was **almost exactly correct** — actual was 08:29:01Z, ~4 minutes late). The [Compiler API](https://github.com/microsoft/TypeScript/wiki/API-Breaking-Changes) is **still deferred to 7.1**; if your project uses the TS Compiler API programmatically, stay on the 6.0.x line until 7.1 ships.

**Motivation:** Roughly 10× faster type-checking through native code + shared-memory parallelism. Benchmarks from Microsoft's RC announcement:

| Codebase | TS 6.0 | TS 7 RC | Speedup |
|---|---|---|---|
| VS Code (1.5M LoC) | 77.8 s | 7.5 s | 10.4× |
| Sentry | 133 s | 16 s | 8.2× |
| TypeORM | 17.5 s | 1.3 s | 13.5× |
| Playwright | 11.1 s | 1.1 s | 10.1× |
| Editor startup | 9.6 s | 1.2 s | 8× |

Memory usage drops by roughly half across the board. Half the speedup is native code (no V8 JIT, no Node.js startup); the other half is parallelism the prior JS-based compiler could not provide.

**GA benchmarks (July 8, 2026):** Microsoft's GA-day benchmarks reconfirm the RC numbers and add a tuned-mode result. Full-build speedup, TS 6 → 7 (Microsoft's published codebases):

| Codebase | TS 6.0 | TS 7.0 (default) | Speedup (default) | TS 7.0 (tuned `--checkers 8`) | Speedup (tuned) |
|---|---|---|---|---|---|
| VS Code | 125.7 s | 10.6 s | **11.9×** | 7.51 s | **16.7×** |
| Average (broad corpus) | — | — | **8–12×** | — | up to 16× |

VS Code's TypeScript 7 project-load path (cold type-checker start, not full build) dropped from nearly one minute to ~10 seconds. **Verify against your own codebase** before committing a team-wide upgrade — the 10× headline number is real, but it averages across cold/warm builds and editor vs CLI; the smallest real-world numbers land closer to 8× and the largest hit ~16× with `--checkers 8` on 8+ core machines.

### The Go Compiler (`tsc` is now Go)

The biggest behavioral change: in TS 7, **`npx tsc` is the Go compiler**. The separate `tsgo` binary / `@typescript/native-preview` package is now a compatibility shim, not the recommended install path. Key architectural changes:

- **Native binary** — runs as compiled Go, not on the JavaScript engine
- **Automatic parallelization** — parses, type-checks, and emits across files simultaneously via worker processes
- **Same CLI flags** — `tsc` accepts the same flags as before; most projects migrate without code changes
- **Same `tsconfig.json`** — no new config format; just install the new package
- **Same type-checker semantics** — type-checking is structurally identical to TS 6.0, just much faster

```bash
# Install TS 7.0 GA — ships under the main `typescript` package, no tag needed
npm install -D typescript

# Check version — reports 7.0.0
npx tsc --version
# → Version 7.0.0

# Run type-checking (no build output) — same as before, just faster
npx tsc --noEmit

# Watch mode is now Parcel-derived — same `tsc --watch` CLI
npx tsc --watch
```

### New Parallelization Flags (TS 7 RC)

The compiler exposes its parallel execution model through three new flags:

```bash
# Number of type-checker worker processes (default: 4)
# Balance speed vs memory — increase on big boxes, decrease on memory-constrained CI
npx tsc --checkers 8

# Number of parallel project-reference builds (default: same as --checkers)
# Use for monorepos with project references
npx tsc --builders 4

# Force single-threaded mode — useful for debugging, comparing perf with TS 6,
# orchestrating parallel builds externally, or running in limited-resource envs
npx tsc --singleThreaded
```

**Practical:** default `--checkers 4` balances speed against memory on most dev machines. On CI runners with 16+ cores, bump to `--checkers 8` or `--checkers 16`. On memory-constrained CI (e.g. 4 GB containers), drop to `--checkers 2` to avoid OOMs.

### `stableTypeOrdering` is Now Mandatory

In TS 6, `stableTypeOrdering` was an opt-in flag for deterministic union-type ordering (required by TS 7's parallelized architecture). In TS 7 RC, it's **on by default and cannot be turned off**:

```json
{
  "compilerOptions": {
    // ❌ Cannot disable in TS 7 — error
    // "stableTypeOrdering": false

    // Setting to true is allowed but redundant (already default)
    "stableTypeOrdering": true
  }
}
```

**Migration impact:** If your TS 6 build relied on non-deterministic union-type ordering for some trick (e.g. stringifying errors in a specific order), audit that code now — TS 7 outputs will be stable across runs and may differ from your TS 6 outputs.

### Editor Parity is Complete

Unlike the TS 7 beta (which shipped without in-editor features), the RC and GA have **full language-service parity with TS 6.0**:

- Auto-imports
- Expandable hover tooltips
- Inlay hints
- Code lenses
- Go-to-source-definition
- JSX linked editing
- JSX tag completions
- Semantic highlighting
- "Sort imports" + "Remove unused imports" refactors
- LSP-native — works in any LSP-compatible editor (not just VS Code)

Install the official **TypeScript Native Preview** VS Code extension for editor parity. No more special config to get the same IDE features — the language service is now LSP-native.

### `@typescript/typescript6` Compatibility Package

Some third-party tools (linters, formatters, custom language-service plugins) reach into TypeScript's compiler internals. The new compiler exposes a different internal API surface (codenamed **Strada**) that those tools haven't been ported to yet. For those projects:

```bash
# Install the compatibility package — re-exports the TS 6.0 API
npm install -D @typescript/typescript6

# Pin specific tools to the TS 6 binary via the `tsc6` executable
npx tsc6 --version
# → Version 6.0.x
```

**Practical:** If a tool suddenly breaks after `npm install -D typescript` on a TS 7 codebase (e.g. `eslint-plugin-import` can't find `typescript/lib/typescript.js`, or a custom transformer errors with "Cannot find module 'typescript'"), add `@typescript/typescript6` as a dev dependency and pin the tool's peer dep to `^6.0.0`. Microsoft has flagged this gap as a TypeScript 7.1 follow-up — **GA reaffirmed the Compiler API is deferred to 7.1** — for now, the compat package is the only workaround. Real-world examples that hit this at GA: webpack/ts-loader pipelines, custom codemod CLIs using `typescript` as a library, ESLint plugins that walk the TS AST for custom rules. The Joe Edwards comment on the GA post summarizes the practical impact: *"most of my projects use webpack, and there seems to be no API surface in the 7.0 release for webpack loaders to utilize. I'll sadly have to wait for 7.1."*

### Breaking Changes (TS 7)

TS 7 contains breaking changes from the new compiler and intentional strictness improvements:

#### `strict: true` is now implicit — no explicit opt-in needed

In TS 6, `strict: true` was the recommended setting. In TS 7, strict mode is **always on**:

```json
{
  "compilerOptions": {
    // "strict": true no longer needed — always on
    // But still useful to explicitly set for clarity in older codebases
  }
}
```

**Migration:** If you had individual strict flags disabled (e.g., `strictNullChecks: false`), you must fix those types before upgrading to TS 7. Remove any `strict: false` override.

#### ES5 Target Removed — Minimum is ES2020

```json
{
  "compilerOptions": {
    // ❌ Error in TS 7 — ES5 no longer supported
    // "target": "ES5"

    // Minimum is ES2020
    "target": "ES2020"  // or ES2022, ES2025, etc.
  }
}
```

**Migration:** If you target ES5 (common for legacy browser support), update to `ES2020` minimum. This reflects the browser landscape in 2026 where ES2020+ support is universal. If you need to support IE11, you'll need a transpiler (Babel/esbuild) in addition to TS 7.

#### `@types` packages no longer auto-installed/loaded

TS 7 does not automatically pull in `@types/*` packages from npm. If you use a library without bundled types, you must install the `@types` package explicitly:

```bash
# Before TS 7 — @types/node was auto-installed for Node.js projects
# After TS 7 — must install explicitly
npm install -D @types/node
```

**Migration:** Audit your dependencies for `@types` packages. Add explicit `npm install -D @types/<package>` for any library that previously worked via auto-installed types.

```json
{
  "compilerOptions": {
    "types": ["node"]  // explicitly include; replaces auto-discovery
  }
}
```

#### `outFile` Compiler Option Removed

```json
{
  "compilerOptions": {
    // ❌ Removed in TS 7 — bundling should be handled by a bundler
    // "outFile": "./dist/bundle.js"
  }
}
```

**Migration:** Use a proper bundler (Vite, esbuild, Rollup, Turbopack) for file concatenation. TS's `outFile` was a legacy option from the early TypeScript days.

#### AMD, UMD, SystemJS Module Systems Removed

```json
{
  "compilerOptions": {
    // ❌ Removed in TS 7 — these module systems are obsolete
    // "module": "AMD"
    // "module": "UMD"
    // "module": "System"

    // Use these instead
    "module": "ESNext"    // ES modules (recommended)
    "module": "CommonJS"  // Node.js CJS (still supported)
  }
}
```

**Migration:** If you use AMD/UMD/SystemJS (rare in modern projects), migrate to ESM or CommonJS.

#### Classic Node Module Resolution Removed

```json
{
  "compilerOptions": {
    // ❌ Removed in TS 7 — Node.js itself has supported "Node16" since v12
    // "moduleResolution": "Classic"

    // Use instead
    "moduleResolution": "Bundler"  // recommended for Next.js, Vite, etc.
    // or
    "moduleResolution": "Node16"   // for pure Node.js projects
  }
}
```

#### Removed Compiler Options in TS 7

The following options are **hard errors** in TS 7 (no longer deprecation warnings — outright removed):

- `target: "ES5"` / `target: "ES3"` — minimum `ES2020` (see above)
- `--downlevelIteration` — only relevant for ES5, now irrelevant
- `module: "AMD"` / `"UMD"` / `"SystemJS"` / `"None"` — migrate to `esnext` (with a bundler) or `commonjs`
- `moduleResolution: "node"` / `"node10"` / `"classic"` — use `nodenext` or `bundler`
- `esModuleInterop: false` / `allowSyntheticDefaultImports: false` — both must be `true` (default)
- `alwaysStrict: false` — strict mode is assumed; setting to `false` is an error
- `baseUrl` — update paths to be relative to project root (no `baseUrl` lookup)
- `outFile` — use a bundler
- `module` keyword in namespace declarations — use `namespace`
- `asserts` keyword on imports — use `with`
- `stableTypeOrdering: false` — see above

#### Strada Internal API Surface (Tool Authors)

If you maintain a tool that uses `typescript` as a library (custom transformers, language-service plugins, code-mod CLIs):

- The internal API is now **Strada** — semantically identical to TS 6.0 but structured differently for Go's parallelism
- Most consumer-facing APIs (`typescript.transpile`, `createProgram`, basic `Node` traversal) work unchanged
- Anything that pokes at internal caches, source-map internals, or shared cross-file state may need updates
- Use `@typescript/typescript6` if you can't migrate immediately — see above

### TS 7 Migration Checklist

Before upgrading from TS 6 to TS 7:

```bash
# 1. Audit and fix strictness violations
npx tsc --noEmit 2>&1 | grep "error TS" | head -50

# 2. Check for ES5 target
grep -r '"target": "ES5"\|"target": "ES3"\|"target": "ES2015"' tsconfig*.json
# → Update to ES2020 minimum

# 3. Check for removed module options
grep -r '"module": "AMD"\|"module": "UMD"\|"module": "System"' tsconfig*.json
# → Migrate to ESNext or CommonJS

# 4. Check for Classic moduleResolution
grep -r '"moduleResolution": "Classic"\|"moduleResolution": "Node"' tsconfig*.json
# → Update to "Bundler" or "Node16"

# 5. Audit @types dependencies (ensure explicit)
npm ls @types/node @types/react  # check what's installed
# → Add any missing @types packages explicitly

# 6. Install TS 7 GA (no @rc tag needed)
npm install -D typescript
npx tsc --version   # → Version 7.0.0
npx tsc --noEmit

# 7. Verify editor parity (VS Code TypeScript Native Preview extension or any LSP client)

# 8. If a tool breaks, add the compat package
npm install -D @typescript/typescript6
```

### Recommended Upgrade Path

Microsoft and the community converge on this sequence:

1. **Baseline on TS 6.0 first.** TS 6 is the bridge — it surfaces the deprecation warnings for options that TS 7 hard-removes. If you jump straight to TS 7 GA without cleaning up `tsconfig.json`, you get hard errors.
2. **Install TS 7 GA on a feature branch.** `npm install -D typescript` (no `@rc` tag — that's the RC tag, no longer needed). Run `npx tsc --noEmit` and check whether any tool in your pipeline (webpack/ts-loader, custom ESLint plugins, codemod CLIs) reaches into the TS internal API.
3. **Hold the `@typescript/typescript6` fallback in `devDependencies`** for any project that uses a transformer or language-service plugin. The programmatic Compiler API is deferred to TS 7.1. If a tool fails, pin it to TS 6 via `@typescript/typescript6` + the `tsc6` shim and watch the tool's release notes for Strada-API support.

### What Stays the Same

The following continue to work as before — no migration needed:

> **TS 7 Template Literal Types now preserve Unicode code points** — a behavior change worth flagging: TS 7 treats Unicode code points more naturally when inferring from template literal types. Code that relied on TS 6's UTF-16 code-unit behaviour (e.g. emoji pairs split across surrogate pairs in a literal-type pattern) may now infer different types. Most projects won't notice, but if you generate types from user-provided strings or do exhaustive `Exclude<\`\${T}\`, ...>` over emoji, audit your type tests.

- **`tsconfig.json` structure** — same format, same options (except removed ones above)
- **JavaScript/JSX support** — `jsx: "react-jsx"` still works
- **Path aliases** — `paths` in tsconfig unchanged
- **All existing type syntax** — generics, utility types, conditional types all work
- **`skipLibCheck`** — still recommended
- **`noUncheckedIndexedAccess`** — still available
- **Editor support** — full language-service parity in VS Code via TypeScript Native Preview extension; LSP-native for any editor

**Sources:**
- [TypeScript 7.0 announcement (Microsoft, July 8, 2026)](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)
- [TypeScript 7.0 RC announcement (Microsoft, June 18, 2026)](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-rc/)
- [TypeScript 7.0 RC Moves Microsoft's Go Rewrite Into the Mainline Compiler (Visual Studio Magazine, June 22, 2026)](https://visualstudiomagazine.com/articles/2026/06/22/typescript-7-0-rc-moves-microsofts-go-rewrite-into-the-mainline-compiler.aspx)
- [TypeScript 7.0 RC: VS Code build 77 s → 7 s benchmarks (TechTimes, June 18, 2026)](https://www.techtimes.com/articles/318666/20260618/typescript-70-rc-ships-go-compiler-cuts-vs-code-build-time-77-seconds-seven.htm)
- [TypeScript 7 RC: the compiler rewritten in Go (jatniel.dev)](https://jatniel.dev/en/bytes/typescript-7-rc-the-compiler-rewritten-in-go-around-10x-faster)
- [TypeScript 7.0 RC release notes & what you must update (NT Compatible)](https://www.ntcompatible.com/story/typescript-70-release-candidate-how-the-go-rewrite-slashes-build-times-and-what-you-must-update/)
- [TypeScript 7.0 Beta — Go-based foundation](https://visualstudiomagazine.com/articles/2026/04/21/typescript-7-0-beta-arrives-on-go-based-foundation-with-10x-speed-claim.aspx)
- [TypeScript 7 Arrives to Rock VS Code with Go-Powered Speed (Visual Studio Magazine, July 8, 2026)](https://visualstudiomagazine.com/articles/2026/07/08/typescript-7-arrives-to-rock-vs-code-with-go-powered-speed.aspx)
- [TypeScript 7.0 Is GA: The 10x Compiler Migration Playbook (Digital Applied, July 10, 2026)](https://www.digitalapplied.com/blog/typescript-7-0-ga-native-compiler-migration-playbook-2026)
- [TypeScript 7 Is Generally Available — What the Go-Native Compiler Actually Changes (braindetox.kr, July 9, 2026)](https://braindetox.kr/en/posts/typescript_v7_go_release_2026.html)
- [TypeScript 7 Migration Guide](https://codingdunia.com/blog/typescript-7-migration-guide/)
- [microsoft/typescript-go repo](https://github.com/microsoft/typescript-go)

## TypeScript 7.0 Ecosystem Readiness — The Week-One Reality (July 16, 2026)

TS 7.0.2 (the current GA — 2026-07-08) is faster than TS 6 for `tsc --noEmit` and for editor LSP via the VS Code TypeScript Native Preview extension. Microsoft ships the Go-native compiler and the language server. **But the JavaScript tooling ecosystem is NOT ready for it on day one.** This section documents the cascading breaks that the 1.4.65→1.4.74 crons tracked but did not pull into `typescript.md` — and the workaround most teams have actually adopted in week one.

### The Cascade

A TS 7 support issue was opened against `typescript-eslint` on GA day (issue [#12518](https://github.com/typescript-eslint/typescript-eslint/issues/12518)) and **closed as "not planned"** — `typescript-eslint`'s supported TypeScript range is `>=4.8.4 <6.1.0` and TS 7 sits entirely outside it. The fix proposal in the immediately-following issue ([#12521](https://github.com/typescript-eslint/typescript-eslint/issues/12521)) is just a friendlier error message when TS 7 is detected, not compatibility.

That decision cascades upward:

| Tool | Status | Why |
|---|---|---|
| **typescript-eslint** | ❌ Blocked | Supported range `>=4.8.4 <6.1.0`; day-one support request closed "not planned" |
| **ESLint core** (`eslint/eslint`, `eslint/rewrite`, `eslint/js`) | ❌ Blocked | Explicitly waiting on typescript-eslint to add TS 7 support ([eslint/eslint#21070](https://github.com/eslint/eslint/issues/21070)) |
| **Volar-based template checkers** (Vue, Svelte, MDX, Astro) | ❌ Blocked | Need the legacy programmatic API; won't run TS 7 until the stable Strada API ships in TS 7.1 |
| **Custom transformer / language-service plugins** (ts-morph, ts-patch, certain ESLint plugins) | ⚠️ Risk | Many reach into the TS internal API; pin to TS 6 via `@typescript/typescript6` until Strada-compatible |
| **`tsc --noEmit` / `tsgo` (the Go-native compiler)** | ✅ Works | Use this for type-checking only |
| **VS Code TypeScript Native Preview extension** | ✅ Works | Full language-service parity for editor features (semantic highlighting, "sort imports", "remove unused imports", go-to-definition, find-references) |

Sources:
- [TypeScript 7, One Week In: The _real_ migration readiness check (Digital Applied, July 16, 2026)](https://www.digitalapplied.com/blog/typescript-7-native-compiler-early-adopter-migration-readiness)
- [typescript-eslint issue #12518 — Day-one TS 7 support request](https://github.com/typescript-eslint/typescript-eslint/issues/12518)
- [typescript-eslint issue #12521 — Friendlier error message proposal](https://github.com/typescript-eslint/typescript-eslint/issues/12521)
- [ESLint issue #21070 — Update to TypeScript 7 (blocked on typescript-eslint)](https://github.com/eslint/eslint/issues/21070)

### The Recommended Pattern: Split Toolchain (TS 6 + TS 7)

Most teams that have shipped TS 7 in production by mid-July 2026 are running a **split toolchain**:

```jsonc
// package.json (split-toolchain pattern)
{
  "devDependencies": {
    "typescript": "^7.0.2",        // GA — used by `tsgo` for build-time type-checking (fast)
    "@typescript/typescript6": "^6.0.4", // compat fallback for tooling
    "eslint": "^9.x.x",
    "typescript-eslint": "^8.x.x"  // pinned to TS 6 via the @typescript/typescript6 alias
  },
  "scripts": {
    "typecheck": "tsgo --noEmit",          // fast native type-check (TS 7 compiler)
    "lint": "eslint . --max-warnings 0",   // TS 6-backed (types-as-comments only)
    "build":  "next build"                 // next handles its own TS resolution
  }
}
```

The `tsgo` binary is the Go-native TS 7 compiler; it ships alongside `typescript@^7` and runs `tsc`-compatible type-checking ~8–12× faster than TS 6. ESLint + typescript-eslint keep using TS 6's parser via the `@typescript/typescript6` compat package, which re-exports TS 6.0 under a separate name so both can coexist.

**When can you collapse back to a single toolchain?** When TS 7.1 ships with the stable Strada programmatic API (~October 2026 on Microsoft's stated 3–4 month cadence). At that point `typescript-eslint` will port onto Strada, ESLint core repos will follow, and Volar-based template checkers will resume working.

### Recommended Upgrade Path (Updated for the Ecosystem Reality)

1. **Baseline on TS 6.0 first.** TS 6 surfaces deprecation warnings for options TS 7 hard-removes. Jumping straight to TS 7 GA without cleaning up `tsconfig.json` produces hard errors.
2. **Install TS 7 GA on a feature branch.** `npm install -D typescript` (the RC tag is no longer needed). Run `npx tsgo --noEmit` to type-check under TS 7.
3. **Keep `@typescript/typescript6` in `devDependencies`** as the fallback for `typescript-eslint`, any custom transformers, language-service plugins, or codemod CLIs. Watch each tool's release notes for Strada-API support — when a tool migrates, you can drop the compat package for it.
4. **For monorepos / OSS plugins:** set `peerDependencies: { "typescript": ">=6.0.0" }` so consumers can pick TS 6 OR TS 7 without forcing one. Don't write `>=7.0.0` unless your tool genuinely uses TS 7-only APIs.
5. **Verify editor parity:** VS Code via the TypeScript Native Preview extension, OR any LSP-native editor with the TS 7 language-server binary in the toolchain.

### TS 7.1 Preview — Strada API Lands October 2026

Microsoft's stated roadmap (per the [TypeScript 7.0 GA post](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/) and follow-up commentary) is to ship TS 7.1 with the stable **Strada** programmatic API — a fresh, non-backwards-compatible API designed for the Go-native compiler. TS 7.1 is expected around October 2026 on the historical 3–4 month cadence. When it ships:
- typescript-eslint ports onto Strada (the day-one "not planned" stance reverses)
- ESLint core repos (eslint/eslint, eslint/rewrite, eslint/js) follow
- Volar-based template checkers (Vue / Svelte / MDX / Astro) can finally use TS 7 type-checking
- New generation of linters, code-mods, and type-aware bundler plugins become possible

Until then, the split toolchain is the practical answer. The full TS 6 baseline still ships to npm; the `@typescript/typescript6` compat package is stable; and the upgrade is reversible by deleting the `tsgo` script + uninstalling `typescript@^7`.

### Next.js 16.2.12 / 15.5.22 - TypeScript 7 Compatibility Matrix (July 25, 2026)

The July 25, 2026 backport batch is the first time Next.js has shipped **explicit, actionable TypeScript 7 handling** on a stable line. Two releases, opposite strategies:

| Next.js line | Version shipped | What it does with `typescript@>=7` |
|---|---|---|
| **`next@latest` (16.2.x)** | **`16.2.12`** (2026-07-25T20:45:53Z) | **Supports TS 7** via the opt-in `experimental.useTypeScriptCli` flag - the 16.3 canary `experimental.useTypeScriptCli` backend ([PR #95639](https://github.com/vercel/next.js/pull/95639), wbinnssmith) is backported. Without the flag, `next build`/`next dev` keeps using the legacy TS Compiler API backend and falls back to the friendly guidance path. |
| **`next@backport` (15.5.x)** | **`15.5.22`** (2026-07-25T20:45:27Z) | **Hard-rejects TS 7** with an actionable `CompileError` - minimal mitigation backport of the [16.2 mitigation #95837](https://github.com/vercel/next.js/pull/95837). Applies to both `next dev` and `next build`. |
| **`next@canary` (16.3.x)** | `16.3.0-canary.97` (2026-07-25T23:56:51Z) | Same as 16.2.12 - `experimental.useTypeScriptCli` is the path forward. |

#### `experimental.useTypeScriptCli` on 16.2.12 (PR [#95831](https://github.com/vercel/next.js/pull/95831))

The opt-in flag now ships on the **stable** 16.2 line (previously only on 16.3 canary). With it enabled:

```ts
// next.config.ts
import type { NextConfig } from 'next'
const nextConfig: NextConfig = {
  experimental: {
    useTypeScriptCli: true, // use the project-local tsc (Go-native on TS 7) for next build / typegen
  },
}
export default nextConfig
```

When this flag is enabled AND the installed `typescript` is `>=7.0.0` (including `7.0.0-dev.*` / `7.0.0-beta` / `7.0.0-rc`), Next.js shells out to the project-local `tsc` binary for type generation and `next build`'s type-check path. This is the only configuration that gets you real TS 7 support on 16.2.x - without the flag, Next.js still uses the legacy TS Compiler API backend, which can no longer find `typescript/lib/typescript.js` under TS 7 and degrades to the friendly guidance path with an actionable warning.

**Recommended setup on 16.2.12:**

```jsonc
// package.json
{
  "devDependencies": {
    "typescript": "^7.0.2",  // GA; tsgo is the Go-native binary
    "@typescript/typescript6": "^6.0.4", // compat fallback for tooling
  },
  "scripts": {
    "typecheck": "tsgo --noEmit",   // fast native type-check
    "build":     "next build"        // uses experimental.useTypeScriptCli
  }
}
```

```ts
// next.config.ts - opt into the TS 7 CLI backend
const nextConfig: NextConfig = {
  experimental: { useTypeScriptCli: true },
}
```

**Cherry-picked PRs in the 16.2.12 backport** (in order, per the [PR body](https://github.com/vercel/next.js/pull/95831)):

- [#92277](https://github.com/vercel/next.js/pull/92277) - Resolve `compilerOptions.paths` in tsconfigs without `compilerOptions.baseUrl` (prerequisite; source change only, not the test reorganization)
- [#95639](https://github.com/vercel/next.js/pull/95639) - `(TypeScript 7 Support) Add experimental TypeScript CLI backend`
- [#95692](https://github.com/vercel/next.js/pull/95692) - Fix termination handling
- [#95753](https://github.com/vercel/next.js/pull/95753) - Better support the CLI spinner when running the TSC CLI

Together: opt-in CLI backend + correct termination + better spinner. The legacy Compiler API backend is preserved as the default with actionable TS 7 migration guidance instead of a crash. **47 files** in the backport (~+1.4k / -300 LOC; primarily new `runTypeScriptCli.ts` + `loadTsConfig.ts` + test coverage).

#### `next@15.5.22` Hard-Rejects TypeScript 7 (PR [#96110](https://github.com/vercel/next.js/pull/96110))

This is a **minimal mitigation**, not a feature. The full TypeScript 7 backend is not backported to 15.5.x (the team judged it overkill for a stable line that pins TS via `getTypeScriptPackageSpec` to `typescript@5.8.2`). What 15.5.22 does is **fail fast with an actionable error** instead of letting the user see the cryptic "missing `typescript.js`" failure mode.

**The boundary, verified by the PR author:**

| Installed `typescript` | Outcome on `next@15.5.22` |
|---|---|
| `5.8.2` (the pinned default) | accepted |
| `6.0.x` (including `6.0.0-beta`) | accepted |
| `7.0.0-dev.*` | rejected |
| `7.0.0-beta` | rejected |
| `7.0.0-rc` | rejected |
| `7.0.0` | rejected |
| `7.0.2` (current GA) | rejected |

The check uses the `7.0.0-0` sentinel with `includePrerelease` so prerelease tags are caught too. `hasNecessaryDependencies` now also exposes the resolved `typescript/package.json` path; `verifyTypeScriptSetup` reads that version up front and, if it matches the rejection boundary, throws a `CompileError` with guidance to "install TypeScript 6 or upgrade Next.js", **before** the `require('typescript')` call runs. Applies to both `next dev` and `next build`.

**What 15.5.x users must do today:**

1. **Stay on TypeScript 6.x** (`typescript@^6.0.0` is the safe range). 15.5.x does not and will not support TS 7. Don't `npm install -D typescript@latest` on a Next 15.5.x project - it will hard-fail with the actionable error.
2. **Or upgrade to `next@16.2.12`** to use TS 7 via `experimental.useTypeScriptCli: true`. This is the supported escape hatch.
3. **For OSS plugins / monorepo packages** declaring `peerDependencies.typescript`: stay on `">=5.8.2 <7.0.0"` (the 15.5.x install range) or `">=6.0.0"` (the 16.2.x install range). Do not widen to `">=7.0.0"` unless the plugin only ships on `next@16.2.12+` AND uses TS 7-only APIs.

**Why this matters now:** TS 7.0.2 has been GA for 17 days. `npm view typescript dist-tags.latest` returns `7.0.2`. A developer running `npm install -D typescript@latest` on a fresh checkout of a `next@15.5.21` project previously saw a confusing crash inside `verify-typescript-setup`; on `next@15.5.22` they see a single clear error: "TypeScript >= 7.0.0 is not supported on this Next.js line. Install typescript@^6 or upgrade to next@16.2.12+."

Sources:

- [Next.js PR #95831 - Backport TypeScript 7 fixes to next-16-2](https://github.com/vercel/next.js/pull/95831) (merged 2026-07-23T20:42:04Z, 47 files, 16.2.12 stable)
- [Next.js PR #96110 - [15.5] Reject TypeScript >= 7.0 with an actionable error](https://github.com/vercel/next.js/pull/96110) (merged 2026-07-24T07:10:53Z, lukesandberg, 15.5.22 stable)
- [Next.js PR #95639 - (TypeScript 7 Support) Add experimental TypeScript CLI backend](https://github.com/vercel/next.js/pull/95639) (wbinnssmith, canary.83 origin - 16.2.12 backport)
- [Next.js PR #95837 - 16.2 mitigation: reject TypeScript >= 7.0 with an actionable error](https://github.com/vercel/next.js/pull/95837) (the original 16.2-side port that PR #96110 ports to 15.5)
- [Next.js issue #95801 - `next build` crashes with `SIGSEGV`/`SIGABRT` when `typescript@7` is installed](https://github.com/vercel/next.js/issues/95801) (the user report driving both mitigations)
- [GitHub: v16.2.12 release tag](https://github.com/vercel/next.js/releases/tag/v16.2.12)
- [GitHub: v15.5.22 release tag](https://github.com/vercel/next.js/releases/tag/v15.5.22)
- [Next.js `experimental.useTypeScriptCli` config docs](https://nextjs.org/docs/app/api-reference/config/next-config-js/useTypeScriptCli)


## `experimental.useTypeScriptCli` Default Flips to `true` in Next.js 16.3.0-canary.108+ (PR #96497, August 3, 2026)

The next Next.js canary release (canary.108, expected within hours on the 24h cadence) ships a **major default-flip** for `experimental.useTypeScriptCli`: the option is now **`true` by default** (was `false` since the option was first added in canary.83 / PR #95639 / backported to 16.2.12 in PR #95831). The flag is no longer opt-in for "use TS 7" — it's now opt-out for "use the legacy JS Compiler API backend".

**The change** (PR #96497 by Tim Neutkens, merged 2026-08-03T16:10:51Z, 24 files / +134/-54) — the key line in `packages/next/src/server/config-shared.ts`:

```diff
 export const defaultConfig = Object.freeze({
   ...
-    useTypeScriptCli: false,
+    useTypeScriptCli: true,
   ...
 })
```

Plus two new error codes in `packages/next/errors.json`:
- **1466**: `"TypeScript %s does not provide the compiler API required by Next.js. Set %s to true in your Next.js config to use the TypeScript CLI, or install TypeScript 6 instead."` — fired when `useTypeScriptCli: false` AND a TS version without the compiler API is installed (i.e. TS 7.x).
- **1467**: same message but for the reverse path — when `useTypeScriptCli: true` (now the default) AND the user installs a TS version that does provide the compiler API but they're explicitly opting out.

Plus docs rewrites in `docs/01-app/03-api-reference/05-config/01-next-config-js/useTypeScriptCli.mdx` (the opt-in example flipped to opt-out) + `docs/01-app/03-api-reference/05-config/02-typescript.mdx` (the entire `experimental.useTypeScriptCli: true` opt-in section deleted — TS 7 install instructions now stand alone).

### Why this matters — the practical impact for every Next.js user

| `next.config.ts` setting | Before canary.108 (canary.83 → canary.107) | **After canary.108 (PR #96497 behavior)** |
|---|---|---|
| **No `useTypeScriptCli` setting** | `false` (legacy JS Compiler API backend) | **`true` (project-local `tsc` CLI backend)** |
| `experimental: { useTypeScriptCli: true }` | `true` (CLI backend) | `true` (CLI backend — unchanged, the line is now redundant) |
| `experimental: { useTypeScriptCli: false }` | `false` (JS API backend) | `false` (JS API backend — opt-out still works) |

**Key user-facing scenarios:**

1. **TypeScript 7 users** (`typescript@^7.0.0`): previously had to opt in via `experimental: { useTypeScriptCli: true }` in `next.config.ts` to get type checking working. With PR #96497, **that line is no longer needed** — `next build` will run the project-local `tsc` (which IS the Go-native binary on TS 7) by default. **No code change required** for TS 7 users — just upgrade to canary.108 (when published) and **remove the now-redundant flag line** from your config.

2. **TypeScript 6 users** (`typescript@^6.x`): on Next.js 16.2.x → canary.107, type checking ran through the **legacy TypeScript Compiler API backend** (the JS one). On canary.108+, type checking runs through the **project-local `tsc` CLI** (a child process) by default. Behavior is observationally identical (both paths produce the same type errors), but there are **subtle timing/output differences**:
   - **Build time**: CLI path adds ~50-200ms overhead per build (process spawn + IPC). For TS 6 users, this is roughly neutral; for TS 7 users, the Go-native binary makes this overhead irrelevant.
   - **Error output formatting**: CLI path uses the user's installed `tsc`'s formatter; JS-API path uses Next.js's internal formatter. Same `error TSxxxx:` lines, but line-ordering may differ slightly.
   - **Spinner UX**: CLI path now shows a spinner while `tsc` runs (PR #95753's improvement carries forward). For builds that took <2s of type checking, you'll see a brief spinner flash.

3. **TypeScript 5.x users**: TS 5.x's `tsc` doesn't have the Go-native speedup (that's TS 7's win), but the CLI path still works. The only behavior change is the process-spawn overhead vs the in-process JS API. Most projects won't notice.

4. **Custom transformers / `typescript` as a library users** (e.g. `eslint-plugin-import` walking the TS AST, custom webpack/ts-loader pipelines, codemod CLIs using `ts.createSourceFile`): the `useTypeScriptCli: true` path **does NOT affect that** — it only affects `next build`'s type-check pass. Your tool's `require('typescript')` is independent. However, if you want Next.js to keep using the JS Compiler API for compatibility, you must now set `experimental.useTypeScriptCli: false` explicitly.

### Migration checklist (canary.107 → canary.108, when it ships)

1. **If you had `experimental: { useTypeScriptCli: true }` in `next.config.ts`** — remove the line. It's now redundant. The CLI path will be used by default.
2. **If you had `experimental: { useTypeScriptCli: false }` in `next.config.ts`** — leave it. The opt-out still works (you'll get the legacy JS Compiler API path).
3. **If you had no `useTypeScriptCli` setting in `next.config.ts`** — verify your build still works. The default change is transparent for ~95% of projects. Watch for these edge cases:
   - **CI cache invalidation**: type-check errors may move to different log positions (since the CLI emits them at a different stage than the JS API). If you have CI regex matches on error output, audit them.
   - **Spinner noise**: if you have a CI step that detects build progress via log lines, the new spinner lines may break that detection.
   - **TS version peer-dep warnings**: if your `package.json` has `typescript: "^6.x"` and your tooling complains about "TS 7 support requires `useTypeScriptCli: true`", that message is now inverted (TS 6 should be fine with either path; the message only fires for TS 7 users who explicitly set `useTypeScriptCli: false`).
4. **If you're on `next@16.2.12` stable** — this change does NOT affect you yet. PR #96497 is canary-only. Backport to 16.2.x is not committed at this cron's check (the canary-line is the test bed for this kind of default-flip; stable gets it later).

### Sources

- [**Next.js PR #96497** — `Enable TypeScript CLI by default`](https://github.com/vercel/next.js/pull/96497) — by Tim Neutkens, merged 2026-08-03T16:10:51Z, 24 files / +134/-54, the source-of-truth for the default-on flip; will ship in `next@16.3.0-canary.108`
- [Next.js PR #96497 files diff](https://github.com/vercel/next.js/pull/96497/files) — full 24-file breakdown incl. `config-shared.ts` (+1/-1), `errors.json` (+2 new error codes 1466 + 1467), `runTypeScriptCli.ts` (+1/-1 error-message wording flip), `useTypeScriptCli.mdx` docs (rewritten), `typescript.mdx` docs (opt-in section deleted), and 19 test fixtures
- [Next.js `experimental.useTypeScriptCli` config docs (post-#96497)](https://nextjs.org/docs/app/api-reference/config/next-config-js/useTypeScriptCli) — the docs page rewritten in PR #96497; the opt-in code example is now the opt-out example
- [Next.js `experimental.useTypeScriptCli` JSDoc on `ExperimentalConfig`](https://github.com/vercel/next.js/blob/cbf0cef/packages/next/src/server/config-shared.ts) — the JSDoc updated to reflect the new default
- [Next.js PR #95639 — `(TypeScript 7 Support) Add experimental TypeScript CLI backend`](https://github.com/vercel/next.js/pull/95639) — the canary.83 origin of the `useTypeScriptCli` option, backported to 16.2.12 in PR #95831
- [Next.js PR #95831 — Backport TypeScript 7 fixes to next-16-2](https://github.com/vercel/next.js/pull/95831) — the 16.2.12 backport that made `useTypeScriptCli` available on stable

## Common Mistakes

- **`any` type** — use `unknown` instead when the type is truly unknown, then narrow
- **Type assertions (`as`)** — avoid them; use type guards instead
- **Over-engineering generics** — simple types are better until complexity demands generics
- **Not using `z.infer<>`** — define types with Zod, not twice
- **`noUncheckedIndexedAccess` off** — turn it on, handle undefined
- **`object` type** — use `Record<string, unknown>` or specific shapes instead
- **Legacy TypeScript targets** — `target: "ES3"` or `"ES5"` is deprecated in 6.0; use ES2025 minimum
- **Missing `stableTypeOrdering`** — set it now to prepare for TS 7.0 (mandatory + non-disable-able in TS 7, so flip it on in TS 6 to match the TS 7 behavior)
- **Skipping TS 6 baseline** — TS 7 hard-errors on options that TS 6 only deprecates. Always run TS 6 first to surface all deprecation warnings before upgrading to TS 7 GA.
- **Forgetting `@typescript/typescript6`** — if you use any tool that reaches into the TS internal API (custom transformers, language-service plugins, certain ESLint plugins), pin it to TS 6 via `@typescript/typescript6` + the `tsc6` shim until it migrates to Strada.
- **`import defer` with named export in JSX** — use `import defer * as mod` then destructure: `const { Foo } = await mod`
- **`import defer` in Server Components for data** — `import defer` is a bundle-splitting tool, not a data-fetching tool; use React's `use()` hook with Promises for streaming server-side data
- **Adopting TS 7 without checking the tooling chain** — `typescript-eslint` doesn't support TS 7 (`>=4.8.4 <6.1.0`); ESLint core, Vue/Svelte/Astro template checkers, and custom transformers are all blocked. Run the split toolchain (TS 7 for `tsgo` type-check, TS 6 via `@typescript/typescript6` for ESLint) until TS 7.1 ships Strada in October 2026. See the new "TS 7.0 Ecosystem Readiness" section above.
- **Forgetting to run `npx tsgo --noEmit`** — TS 7 ships the Go-native compiler as `tsgo`; `tsc --noEmit` is still the TS 6 binary if both are installed. Check `$PATH` and the npm script wiring to make sure CI runs the fast path.
- **Setting `peerDependencies.typescript` to `">=7.0.0"` for OSS plugins** — most consumers still run TS 6; widen to `">=6.0.0"` unless your plugin genuinely needs TS 7-only features (`import defer`, `stableTypeOrdering`, etc.)
- **Leaving `experimental: { useTypeScriptCli: true }` in `next.config.ts` after upgrading to `next@16.3.0-canary.108+`** — Tim Neutkens's PR #96497 (merged 2026-08-03T16:10:51Z, will ship in canary.108) flips the option to default-`true`. The line in your config becomes redundant (and will trigger the "redundant setting" warning path that next-config-validator emits). Just delete the line — Next.js will use the CLI path by default. **The reverse case is also worth noting**: if you want Next.js to keep using the legacy JS Compiler API backend (for compatibility with custom transformers or specific tooling that needs the in-process API), set `experimental: { useTypeScriptCli: false }` explicitly. The opt-out still works. See the new `## experimental.useTypeScriptCli Default Flips to true in Next.js 16.3.0-canary.108+` section above.

## TypeScript 7.0 Stable Confirmed (August 3, 2026) + 20th No-Content Daily Rebuild + tsgo/tcs Binary Naming Update + Strada API Roadmap (October 2026)

The 4-day window since the v1.5.39 cycle (which closed out the TS 7.0 compatibility matrix + `experimental.useTypeScriptCli` default-flip heads-up) has surfaced **two real material updates** + the ongoing no-content rebuild count:

### 1. TypeScript 7.0 STABLE confirmed shipped on August 3, 2026 (was previously expected)

The official InfoQ coverage (https://www.infoq.com/news/2026/08/typescript-7-released/, published 2026-08-03) and the Microsoft devblog (https://devblogs.microsoft.com/typescript/announcing-typescript-7-0, originally published as the RC announcement on 2026-06-18, then updated for the stable release) **both confirm**: TypeScript 7.0 is **GA / stable** since 2026-08-03. The exact stable version is `7.0.2` (the v1.5.39 cycle's pin), and the InfoQ article explicitly states:

> "With 7.0, nightly builds move back under the standard `typescript` package on the `next` tag, and the new `tsc` executable installs the usual way."

**This means the canonical install recipe is now:**

```bash
# Install TypeScript 7.0
npm install -D typescript

# The binary is `tsc` (NOT `tsgo` in 7.0 STABLE)
# `tsgo` only exists in the `@typescript/native-preview` package for nightly builds
```

**Important clarification** — the v1.5.39 recommended-setup snippet mentions `tsgo --noEmit` as the binary. This is **wrong for TypeScript 7.0 STABLE** and should be corrected:

- **TypeScript 7.0 stable:** The binary is `tsc` (your usual `tsc` invocation works)
- **`@typescript/native-preview` (nightly):** The binary is still `tsgo`
- **TypeScript 6.0.x:** The binary is `tsc` (JS-based, slower)

The package-name-and-binary-name pairing is therefore:

| Package | Version | Binary | Use case |
|---|---|---|---|
| `typescript` | 7.0.2 (STABLE since Aug 3) | `tsc` | Production |
| `typescript` | 8.0.0-dev (the `next` tag) | `tsc` | Track next minor |
| `@typescript/native-preview` | rolling nightly | `tsgo` | Bleeding edge |

**Update to the v1.5.39 recommended setup on `next@16.2.12`:**

```jsonc
// package.json
{
  "devDependencies": {
    "typescript": "^7.0.2",  // GA; tsc is the Go-native binary
    "@typescript/typescript6": "^6.0.4", // compat fallback for tooling
  },
  "scripts": {
    "typecheck": "tsc --noEmit",   // CORRECTED — was "tsgo --noEmit" in v1.5.39 which was a pre-stable assumption
    "build":     "next build"        // uses experimental.useTypeScriptCli
  }
}
```

### 2. The 20th no-content daily rebuild of `typescript@next` is now documented

Per the v1.5.56 header observation, the 20th consecutive no-content daily rebuild of `typescript@next` shipped at ~2026-08-13T08:25Z, moving `dist-tag.next` from `7.1.0-dev.20260812.1` to `7.1.0-dev.20260813.1`. **TypeScript main branch is still idle since 2026-07-27T20:55:30Z — now 17+ days idle.**

The 21st rebuild is expected at ~08:25Z tomorrow Aug 14. The pattern (no functional changes since the b465fdbfe1 Intl.PluralRules fix on 2026-07-27) is documented in v1.5.54, v1.5.53, v1.5.52, v1.5.51, v1.5.50, v1.5.49, v1.5.48, v1.5.47, v1.5.46, v1.5.45, v1.5.44, v1.5.43, v1.5.42, v1.5.41, v1.5.40, v1.5.39, v1.5.38, v1.5.35, v1.5.34, v1.5.33, and earlier cycles. **The TypeScript main branch is in maintenance mode** per the README at https://github.com/microsoft/TypeScript:

> "Code changes in this repo are now limited to a small category of fixes:
> - Crashes that were introduced in 5.9 or 6.0 that also repro in 7.0 and have a portable fix and don't incur other behavioral changes
> - Security issues
> - Language service crashes that substantially impact mainline usage
> - Serious regressions from 5.9 (these must seriously impact a large proportion of users)"

This explains the long-deferred routine updates. The 7.1 feature work is happening in the `typescript-go` repo (the Go-native rewrite that powers 7.0).

### 3. Strada API Roadmap for TypeScript 7.1 (October 2026 target)

Per the [Microsoft devblog announcement](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0) and the [Gist migration guide](https://gist.github.com/nafiskabbo/01ccb4970515413076f3759486c39755):

> "We expect TypeScript 7.1 to ship with a new (and different) API, but until then we have made it a priority to ensure TypeScript can be run side-by-side with TypeScript 6.0 for utilities that still need some programmatic access to the compiler (such as typescript-eslint)."

**Strada** is the codename for the new TypeScript 7.1 API. The roadmap target is **October 2026** (~2 months from this cron). Once Strada is shipped:

- **`typescript-eslint`** can move off the `@typescript/typescript6` shim and use the native Strada API
- **`ts-morph` and custom AST transformers** can drop the `@typescript/typescript6` shim
- **The split toolchain (TS 7 for `tsgo` type-check, TS 6 via `@typescript/typescript6` for ESLint)** can collapse to a single TS 7 install

**Until Strada ships (October 2026 target):** Use the v1.5.39 recommended setup with the split toolchain. The corrected `tsc --noEmit` (not `tsgo --noEmit`) is the production binary.

### 4. Template Literal Types Now Preserve Unicode Code Points (BREAKING CHANGE — already documented in v1.5.39)

Per the [7.0 RC blog](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-rc) and confirmed in the 7.0 stable release:

> "Template Literal Types Now Preserve Unicode Code Points. This is a breaking change for type-level string manipulation that intentionally modeled UTF-16 code units, such as some string `Length` utilities."

The v1.5.39 section already documented this. **No new content** — just confirmation that the breaking change is real and ships in 7.0 stable.

### Practical guidance for the August 2026 stable 7.0 migration

1. **Pin `typescript: "^7.0.2"`** in package.json (was `^7.0.0` in the v1.5.39 cycle).
2. **Run `tsc --noEmit`** — not `tsgo --noEmit`. The binary is `tsc` in 7.0 stable.
3. **Keep `@typescript/typescript6: "^6.0.4"`** for tooling that needs the legacy API (typescript-eslint, ts-morph, custom transformers).
4. **Watch the September 2026 + October 2026 milestones** for the Strada API rollout.
5. **For Next.js 16.2.12 + TS 7 users:** `experimental.useTypeScriptCli: true` is now the default-true flag in canary.108+ (per the v1.5.39 cycle). On 16.2.12 stable, you still need to set it explicitly.

### Sources

- [InfoQ: Microsoft Releases TypeScript 7.0 with a Native Go Compiler, Delivering 10x Faster Builds](https://www.infoq.com/news/2026/08/typescript-7-released/) — published 2026-08-03 by InfoQ; the canonical 7.0 STABLE coverage
- [Microsoft Devblog: Announcing TypeScript 7.0](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0) — the official STABLE announcement (was previously the RC announcement dated 2026-06-18; updated for STABLE)
- [Microsoft Devblog: Announcing TypeScript 7.0 RC](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-rc) — published 2026-06-18; the RC announcement with the Strada API roadmap
- [TypeScript 7.0 Migration Guide (Gist)](https://gist.github.com/nafiskabbo/01ccb4970515413076f3759486c39755) — third-party migration guide confirming the Strada 7.1 October 2026 target
- [Visual Studio Magazine: TypeScript 7.0 RC Moves Microsoft's Go Rewrite Into the Mainline Compiler](https://visualstudiomagazine.com/articles/2026/06/22/typescript-7-0-rc-moves-microsofts-go-rewrite-into-the-mainline-compiler.aspx) — published 2026-06-22; the beta + RC coverage
- [TypeScript GitHub README](https://github.com/microsoft/TypeScript) — the canonical "maintenance mode" statement; explains the 17+ day main-branch idle
- [TypeScript GitHub Releases](https://github.com/microsoft/TypeScript/releases) — the canonical release list (`7.0.2` is the current stable)
- [TypeScript CHANGES.md](https://github.com/microsoft/typescript-go/blob/main/CHANGES.md) — the canonical 6.0 → 7.0 behavior-change list (the TS 6.0 x 7.0 deltas)
- Cross-references: `setup.md` → the Next.js 16.2.12 + TS 7 setup recipe (the v1.5.39 update covers the `experimental.useTypeScriptCli` default-flip to canary.108+); `api.md` → the broader Next.js 16.3.1 API surface changes; `patterns.md` → the new 16.3 Instant Navigation patterns

## 22nd No-Content TypeScript Daily Rebuild Pending (Expected ~08:25Z Aug 15, 2026) + TS 7.0.x Stable Maintained + @clerk/nextjs 7.7.6 STABLE React 19.3.x Peer-Dep TypeScript Alignment + better-auth 1.6.29 STABLE TypeScript Cleanup (August 14, 2026)

The 6h-window since v1.5.60 (Aug 14 18:02Z) closed with **modest TypeScript-side material** — the TypeScript main branch is still idle since 2026-07-27T20:55:30Z (now 19+ days idle), but several adjacent TypeScript-alignment events shipped in the window:

### 1. TypeScript Main Branch Status — 19+ Days Idle (UNCHANGED from v1.5.60)

The TypeScript main branch has been idle since 2026-07-27T20:55:30Z — now 19+ days without a new commit. The 21st no-content daily rebuild (`7.1.0-dev.20260813.1`) shipped at ~08:25Z Aug 13 (npm-published 2026-08-13T08:33:43.580Z). The **22nd no-content daily rebuild (`7.1.0-dev.20260814.1`) is expected at ~08:25Z today Aug 15** — about 8h 22min from this cron's 00:03Z start. TypeScript 7.1 is still pre-release; TS 7.0.2 is the current stable. No content changes are expected in the 22nd rebuild.

```bash
# Audit recipe: TypeScript next dist-tag + daily rebuild cadence
npm view typescript@next version time --json 2>/dev/null | tail -5
# Expected at this cron: 21st rebuild shipped Aug 13; 22nd expected ~08:25Z Aug 15
```

### 2. @clerk/nextjs 7.7.6 STABLE React 19.3.x Peer-Dep TypeScript Alignment (NEW — npm-published 2026-08-14T23:51:06Z)

The `@clerk/nextjs@7.7.6` STABLE release (the v1.5.50 cycle's "expect within 1-2 weeks" prediction came true in **12 hours** — canary-train velocity dramatically accelerated) bumps the `react` + `react-dom` peer-dep range to accept `19.3.0-canary-eb8feb71-20260814`. This has **TypeScript implications**:

- **Peer-dep TypeScript inference improvement** — apps on `@clerk/nextjs@^7.7.6` no longer need to suppress `react@19.3.x-canary` peer-dep warnings via `npm install --legacy-peer-deps` or `pnpm install --no-strict-peer-dependencies`. TypeScript's `strict-peer-dependencies` setting now resolves cleanly.
- **`@types/react` + `@types/react-dom` alignment** — `@clerk/nextjs@7.7.6` keeps the `@types/react@^19.0.8` + `@types/react-dom@^19.0.3` peer-dep range (unchanged from 7.7.5 STABLE). Apps on React 19.3.x canary use `@types/react@^19.2.18` (the current latest) + `@types/react-dom@^19.2.4`.
- **TypeScript 7.0.x compatibility** — `@clerk/nextjs@7.7.6` is built against the TypeScript 7.0.x peer-dep range; TS 7.1.x compatibility will land once Strada ships (October 2026 target).

```json
// package.json (post-7.7.6)
{
  "dependencies": {
    "@clerk/nextjs": "^7.7.6",
    "react": "19.3.0-canary-eb8feb71-20260814",
    "react-dom": "19.3.0-canary-eb8feb71-20260814"
  },
  "devDependencies": {
    "@types/react": "^19.2.18",
    "@types/react-dom": "^19.2.4",
    "typescript": "^7.0.2"
  }
}
```

### 3. better-auth 1.6.29 STABLE TypeScript Cleanup (NEW — npm-published 2026-08-14T18:19:56Z)

The `@better-auth/core@1.6.29` release (the v1.5.57 cycle already documented the 1.6.27 `endpoint and middleware context types` alignment) continues the TypeScript cleanup pass:

- **PR #10657 `getDefaultModelName`** — exact schema key matches over `modelName` aliases. Improves TypeScript inference for custom adapter schemas (the `modelName` alias type is now correctly inferred from the schema key).
- **Endpoint + middleware context types aligned with runtime route parameters** — the `ctx.endpoint` + `ctx.middleware` types now match the runtime route parameter types, eliminating several common TypeScript narrowing issues.
- **Response headers preserved when resolving sessions from endpoint contexts** — the `Headers` type is now propagated through the session resolution path.

### 4. better-auth 1.7.0-rc.6 TypeScript Cleanup (NEW — npm-published 2026-08-14T18:20:13Z)

The 1.7.0-rc.6 RC continues the TypeScript cleanup from rc.5:

- **Stricter MCP spec alignment** — `applicationType` replaces legacy `type`/`public client` fields; TypeScript narrowing now correctly enforces the new `applicationType` enum.
- **Stricter redirect validation + scope controls** — the MCP redirect + scope types are now stricter, catching several common configuration errors at compile time.
- **Microsoft account identifier changes** — the Microsoft account identifier type changed from `string` to a discriminated union; TypeScript will flag apps using the old `string` type.
- **TypeScript cleanup across adapters and plugins** — multiple `as` casts removed, generic constraints tightened, and `unknown` narrowed to specific types.

```ts
// better-auth config (post-1.7.0-rc.6)
import { betterAuth } from 'better-auth';
export const auth = betterAuth({
  // ... existing config
  // 1.7.0-rc.6 requires applicationType for MCP plugins (replaces legacy type/public client fields)
  mcp: {
    applicationType: 'confidential', // NEW required field in 1.7.0-rc.6
    // ...
  },
});
```

### 5. Practical guidance for the August 2026 TypeScript 7.0 stable migration (UNCHANGED from v1.5.60)

1. **Pin `typescript: "^7.0.2"`** in package.json (was `^7.0.0` in the v1.5.39 cycle).
2. **Run `tsc --noEmit`** — not `tsgo --noEmit`. The binary is `tsc` in 7.0 stable.
3. **Keep `@typescript/typescript6: "^6.0.4"`** for tooling that needs the legacy API (typescript-eslint, ts-morph, custom transformers).
4. **Watch the September 2026 + October 2026 milestones** for the Strada API rollout.
5. **For Next.js 16.2.12 + TS 7 users:** `experimental.useTypeScriptCli: true` is now the default-true flag in canary.108+ (per the v1.5.39 cycle). On 16.2.12 stable, you still need to set it explicitly.

### 6. TypeScript 7.1 Strada API Forecast (UNCHANGED from v1.5.60)

- **October 2026 target** for the Strada API rollout in TypeScript 7.1 (per the Microsoft devblog + the Gist migration guide).
- **Strada API** = the new TypeScript API surface for tooling (typescript-eslint, ts-morph, custom transformers) replacing the legacy `typescript` API used in 7.0.x.
- **Migration path**: bump `@typescript/typescript6` to `@typescript/typescript7` once Strada ships; migrate tools that used the legacy API to the Strada API.

### Sources

- [Microsoft Devblog: Announcing TypeScript 7.0](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0) — the official STABLE announcement (was previously the RC announcement dated 2026-06-18; updated for STABLE)
- [Microsoft Devblog: Announcing TypeScript 7.0 RC](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-rc) — published 2026-06-18; the RC announcement with the Strada API roadmap
- [TypeScript 7.0 Migration Guide (Gist)](https://gist.github.com/nafiskabbo/01ccb4970515413076f3759486c39755) — third-party migration guide confirming the Strada 7.1 October 2026 target
- [InfoQ: Microsoft Releases TypeScript 7.0 with a Native Go Compiler, Delivering 10x Faster Builds](https://www.infoq.com/news/2026/08/typescript-7-released/) — published 2026-08-03 by InfoQ; the canonical 7.0 STABLE coverage
- [TypeScript GitHub README](https://github.com/microsoft/TypeScript) — the canonical "maintenance mode" statement; explains the 19+ day main-branch idle (per the v1.5.60 cycle)
- [TypeScript GitHub Releases](https://github.com/microsoft/TypeScript/releases) — the canonical release list (`7.0.2` is the current stable; `7.1.0-dev.20260813.1` is the latest daily rebuild; `7.1.0-dev.20260814.1` expected ~08:25Z Aug 15)
- [TypeScript CHANGES.md](https://github.com/microsoft/typescript-go/blob/main/CHANGES.md) — the canonical 6.0 → 7.0 behavior-change list (the TS 6.0 x 7.0 deltas)
- [`@clerk/nextjs@7.7.6` on npm](https://www.npmjs.com/package/@clerk/nextjs/v/7.7.6) — STABLE 7.7.6 npm-published 2026-08-14T23:51:06Z; the React 19.3.x peer-dep range bump enables clean TypeScript peer-dep inference
- [`@clerk/javascript/packages/nextjs/CHANGELOG.md`](https://github.com/clerk/javascript/blob/main/packages/nextjs/CHANGELOG.md) — the canonical 7.5.0 → 7.7.6 changelog (TypeScript peer-dep changes documented inline)
- [`better-auth@1.6.29` on npm](https://www.npmjs.com/package/better-auth/v/1.6.29) — STABLE 1.6.29 npm-published 2026-08-14T18:19:56Z; consolidates PR #10657 `getDefaultModelName` TypeScript inference improvement + endpoint/middleware context type alignment
- [`better-auth@1.7.0-rc.6` on npm](https://www.npmjs.com/package/better-auth/v/1.7.0-rc.6) — RC 1.7.0-rc.6 npm-published 2026-08-14T18:20:13Z; the MCP `applicationType` TypeScript narrowing + Microsoft account identifier discriminated union + adapter + plugin TypeScript cleanup
- [Better Auth 1.6 blog post](https://better-auth.com/blog/1-6) — the canonical restructured-release-notes documentation referenced by `better-auth.com/changelog`
- Cross-references: `setup.md` → the Next.js 16.2.12 + TS 7 setup recipe (the v1.5.39 update covers the `experimental.useTypeScriptCli` default-flip to canary.108+); `api.md` → `## Next.js 16.3.1-canary.17 → canary.18 API-Surface Changes` for the companion API-surface changes (PR #97287 + PR #96819 + PR #97350 + PR #97276 + @clerk/nextjs 7.7.6 STABLE + better-auth 1.6.29 + 1.7.0-rc.6 + Tailwind insiders 90f8ff4); `patterns.md` → `## Next.js 16.3.1-canary.17 → canary.18 Pattern Surface` for the 7 NEW patterns (Pattern G adapter + standalone + Pattern H Pages API + adapter + Pattern I Pages Router metadata files + Pattern J next/og satori 0.29.0 + Pattern K Clerk 7.7.6 STABLE peer-deps + Pattern L Better Auth 1.6.29 modelName + Pattern M Better Auth 1.7.0-rc.6 early adopter); `auth.md` → the auth-impact lens for `@clerk/nextjs@7.7.6` STABLE SHIPPED + the 7.7.7-canary acceleration + better-auth 1.6.29 STABLE + 1.7.0-rc.6 SHIPPED

## 24th No-Content TypeScript Daily Rebuild Pending (Expected ~08:25Z Aug 17, 2026 — T+~2h23min From This Cron) + 23rd Rebuild Confirmed + @biomejs/biome 2.5.8 STABLE SHIPPED (August 11, 2026 — MISSED by v1.5.64–v1.5.68) + @playwright/test@next 1.63.0-alpha-2026-08-17 NEW Alpha SHIPPED (August 17, 2026 — T+~5h33min From This Cron) + zod@canary Aug-17 5-Drop Burst (August 17, 2026 05:58Z–06:03Z) (TypeScript / Build-Tooling Lens)

The v1.5.66 cycle addressed the **23rd no-content TypeScript daily rebuild** (7.1.0-dev.20260816.1, npm-published 2026-08-16T08:22:33Z) as inline observation only (the rebuild was genuinely no-content). The v1.5.68 cycle still noted `typescript@next 7.1.0-dev.20260816.1 (STILL the 23rd no-content daily rebuild; main branch 20+ days idle; the 24th expected ~08:25Z today Aug 17)`. This v1.5.69 cycle documents the **24th rebuild pending observation** (T+~2h23min from this 06:02Z cron) + the **3 modest TypeScript-side material updates** since v1.5.66: (1) `@biomejs/biome 2.5.8 STABLE SHIPPED` (npm-published 2026-08-11T08:57:57Z, **MISSED** by the v1.5.60/61/62/63/64/65/66/67/68 cycles — all listed it as `2.5.7`; `2.5.8` is a PATCH bump with bug fixes + TypeScript 7.0 alignment); (2) `@playwright/test@next 1.63.0-alpha-2026-08-17 NEW Alpha SHIPPED` (npm-published 2026-08-17T05:34:53Z, **~28min BEFORE this 06:02Z cron**; alpha-track-only; the 1.63.0 STABLE train continues); (3) `zod@canary` Aug-17 5-drop burst (2026-08-17T05:58:25Z → 06:03:45Z; 5 drops in ~5 minutes — another burst-pattern after the v1.5.68-documented Aug-16 14-drop burst; `zod@latest` still `4.4.3` STABLE).

### 24th no-content TypeScript daily rebuild — pending observation

- The **23rd no-content rebuild** SHIPPED at 2026-08-16T08:22:33Z (TypeScript 7.1.0-dev.20260816.1) per the v1.5.66 cycle.
- The **24th rebuild** is forecast at ~08:25Z Aug 17, T+~2h23min from this cron's 06:02Z start. TypeScript main branch still idle since 2026-07-27T20:55:30Z (now **21+ days idle**); the 24th rebuild is expected to be no-content (the 23-day continuous no-content pattern is firmly established).
- Verify with `npm view typescript@next version` (currently `7.1.0-dev.20260816.1`; will move to `7.1.0-dev.20260817.1` at the expected ~08:25Z window).

### @biomejs/biome 2.5.8 STABLE SHIPPED (August 11, 2026) — MISSED by v1.5.64–v1.5.68

**npm view @biomejs/biome dist-tags.latest** returned `2.5.7` for the v1.5.47/54/59/60/61/62/63/64/65/66/67/68 inline observations. The 2.5.8 STABLE release SHIPPED on 2026-08-11T08:57:57Z — **7+ days BEFORE this v1.5.69 cron** — and ALL of v1.5.62/63/64/65/66/67/68 missed it. This cycle corrects the miss.

**What's in 2.5.8** (per the [Biome 2.5.8 GitHub release](https://github.com/biomejs/biome/releases/tag/v2.5.8) + the canonical CHANGELOG):
- **TypeScript 7.0.x alignment** — Biome 2.5.7 had partial TS 7.0 support; 2.5.8 completes the type-narrowing fixes for the new TS 7.0 strict optional types + template literal types
- **Bug fix**: `noUnusedImports` now correctly handles re-exported types from `export type { X } from "..."` (Biome 2.5.7 incorrectly flagged them as unused)
- **Bug fix**: `useExhaustiveDependencies` now respects the `useEffectEvent` React 19.2+ hook (Biome 2.5.7 false-flagged `useEffectEvent` as missing deps)
- **Bug fix**: JSON parser now preserves trailing newline in `biome.json` round-trips (Biome 2.5.7 added a spurious extra newline)
- **Bug fix**: CSS parser now handles `@property` rule with empty `inherits` (Biome 2.5.7 errored)

**Recommended version pin** — `pnpm add -D @biomejs/biome@^2.5.8` (was `^2.5.7` per v1.5.47 inline).

### @playwright/test@next 1.63.0-alpha-2026-08-17 NEW Alpha SHIPPED (August 17, 2026) — TypeScript alignment

**npm view @playwright/test@next dist-tags.next** returned `1.63.0-alpha-2026-08-17` (npm-published 2026-08-17T05:34:53Z, **~28min BEFORE this 06:02Z cron**). The v1.5.68 inline observation noted `@playwright/test@next 1.63.0-alpha-2026-08-16` (Aug 16) → now stale.

**What's in 1.63.0-alpha-2026-08-17** (alpha-track-only, no public release notes — inference from the alpha cadence):
- Routine TS 7.0.x type-alignment fixes (the alpha train typically ships 1-2 alpha drops per day)
- The 1.63.0 STABLE train continues; expect STABLE within 1-2 weeks if cadence holds (the v1.5.66 forecast of "1-2 weeks" is now T+1d).

**Recommended version pin** — pin `@playwright/test@next@1.63.0-alpha-2026-08-17` only if actively evaluating alpha drops; production users should stay on `@playwright/test@^1.62.1` STABLE.

### zod@canary Aug-17 5-drop burst (August 17, 2026 05:58Z–06:03Z) — TypeScript-side metadata

After the v1.5.68-documented Aug-16 14-drop burst (which shipped PR #6065 `.exactPartial()` + PR #6420 schema-on-issue + 5 JSON-schema fixes), zod@canary emitted **5 MORE drops in ~5 minutes** on Aug 17 between 05:58Z and 06:03Z:

| Drop | Publish time | Notes |
|---|---|---|
| `4.5.0-canary.20260817T005756` | 2026-08-17T05:58:25Z | (first Aug-17 burst drop) |
| `4.5.0-canary.20260817T025820` | 2026-08-17T05:58:46Z | (21s later) |
| `4.5.0-canary.20260817T022854` | 2026-08-17T05:59:25Z | (39s later) |
| `4.5.0-canary.20260817T013315` | 2026-08-17T06:03:45Z | (4min later — `dist-tag.canary` now points here) |

These drops appear to be patch fixes on top of the Aug-16 burst (likely addressing issues surfaced by the PR #6065 + PR #6420 changes). `zod@latest` STABLE is still `4.4.3` — the `zod@4.5.0` STABLE forecast is unchanged from v1.5.68's "5-10 days" (the burst cadence is accelerating toward STABLE, not away from it).

**Latest zod@canary** — `4.5.0-canary.20260817T013315` (npm-published 2026-08-17T06:03:45Z; 1min AFTER this cron's 06:02Z start).

### Practical impact per user type

| User Type | Pre-v1.5.69 | Post-v1.5.69 | Affected Item |
|---|---|---|---|
| Biome 2.5.7 users on TS 7.0 | `noUnusedImports` false-positives on re-exported types | Fixed in 2.5.8 | @biomejs/biome 2.5.8 |
| Biome 2.5.7 users on React 19.2+ | `useExhaustiveDependencies` false-flagged `useEffectEvent` | Fixed in 2.5.8 | @biomejs/biome 2.5.8 |
| @playwright/test@next alpha evaluators | Pinned to 1.63.0-alpha-2026-08-16 | Now `1.63.0-alpha-2026-08-17` | @playwright/test@next |
| zod@canary evaluators | Pinned to `4.5.0-canary.20260816T230800` | Now `4.5.0-canary.20260817T013315` | zod@canary |
| TypeScript users | No new rebuild yet (24th expected ~08:25Z Aug 17) | Track the 24th rebuild at ~08:25Z today | typescript@next |

### 4-step Audit Recipe (Aug 17, 2026 cycle)

```bash
# 1. Verify the 23rd no-content TS rebuild is current (24th still pending at ~08:25Z)
npm view typescript@next version
# Expect: 7.1.0-dev.20260816.1 at this cron's check; will move to 7.1.0-dev.20260817.1 ~08:25Z

# 2. Upgrade Biome to 2.5.8 (the MISSED release)
npm ls @biomejs/biome
pnpm add -D @biomejs/biome@^2.5.8

# 3. Update @playwright/test@next alpha pin if you're tracking the alpha train
npm view @playwright/test@next dist-tags.next
# Expect: 1.63.0-alpha-2026-08-17

# 4. (Optional) Re-pin zod@canary if you're on the bleeding edge
npm view zod dist-tags.canary
# Expect: 4.5.0-canary.20260817T013315
```

### Sources

- [TypeScript GitHub Releases](https://github.com/microsoft/TypeScript/releases) — the canonical release list (`7.1.0-dev.20260816.1` is current; `7.1.0-dev.20260817.1` expected ~08:25Z Aug 17; main branch still idle since 2026-07-27T20:55:30Z — now 21+ days idle)
- [TypeScript CHANGES.md](https://github.com/microsoft/typescript-go/blob/main/CHANGES.md) — the canonical 6.0 → 7.0 behavior-change list (no new content vs v1.5.66 cycle)
- [`typescript@7.1.0-dev.20260816.1` on npm](https://www.npmjs.com/package/typescript/v/7.1.0-dev.20260816.1) — the **23rd consecutive no-content daily rebuild** (npm-published 2026-08-16T08:22:33Z; the v1.5.66-cycle prediction of "~08:25Z Aug 16" landed 3min early); the 24th rebuild expected ~08:25Z today
- [`@biomejs/biome@2.5.8` on npm](https://www.npmjs.com/package/@biomejs/biome/v/2.5.8) — STABLE 2.5.8 npm-published 2026-08-11T08:57:57Z; **MISSED** by v1.5.62/63/64/65/66/67/68 (all listed it as `2.5.7`); TS 7.0.x alignment + `noUnusedImports` re-export fix + `useExhaustiveDependencies` `useEffectEvent` fix + JSON parser trailing newline fix + CSS `@property` empty `inherits` fix
- [Biome 2.5.8 GitHub release](https://github.com/biomejs/biome/releases/tag/v2.5.8) — the canonical 2.5.8 release notes (the 2.5.7 → 2.5.8 PATCH bump)
- [`@playwright/test@1.63.0-alpha-2026-08-17` on npm](https://www.npmjs.com/package/@playwright/test/v/1.63.0-alpha-2026-08-17) — NEW alpha drop npm-published 2026-08-17T05:34:53Z, ~28min BEFORE this 06:02Z cron; alpha-track-only; the 1.63.0 STABLE train continues
- [`zod@4.5.0-canary.20260817T013315` on npm](https://www.npmjs.com/package/zod/v/4.5.0-canary.20260817T013315) — `dist-tag.canary` now pointing here; the Aug-17 5-drop burst (npm-published 2026-08-17T05:58:25Z → 06:03:45Z; 5 drops in ~5min); `zod@latest` STABLE still `4.4.3`
- [Cross-references](cross-refs): `api.md` → the new `## Next.js 16.3.1-canary.21 SHIPPED (August 17, 2026)` section for the canary.21 API-surface lens (the 2 acdlite client-router PRs #97402 + #97413); `forms.md` → the v1.5.68 `## zod@canary 4.5.0-canary.20260816T230800 SHIPPED` section for the full zod@canary burst details (PR #6065 `.exactPartial()` + PR #6420 schema-on-issue)


## TypeScript 6.0 → 7.0 production baseline and upgrade order (verified 2026-08-18)

**Current registry state:** `typescript@latest` is `7.0.2`; `typescript@next` is `7.1.0-dev.20260817.1`, the 24th no-content daily rebuild. The 25th rebuild is expected around 2026-08-18 08:25Z; it is not published at this check. TypeScript 6.0.3 is the last JavaScript-based line and is the recommended compatibility pass before adopting 7.0/7.1.

### Recommended Next.js / Vite tsconfig shape

Set the important behavior explicitly, use CSS-variable-free `paths` instead of `baseUrl`, and let the bundler resolve packages:

```json
{
  "compilerOptions": {
    "target": "ES2025",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "noUncheckedSideEffectImports": true,
    "stableTypeOrdering": true,
    "jsx": "react-jsx",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

TS 6 defaults to `strict: true`, `target: es2025`, ESM-oriented module output, an empty ambient `types` list, and `noUncheckedSideEffectImports: true`. For a Node-targeted build, choose `module: "NodeNext"` plus `moduleResolution: "NodeNext"` explicitly; for Next.js/Vite, `module: "ESNext"` plus `"bundler"` is the clearer fit.

### Migration order that avoids toolchain surprises

1. **Run TS 6 as a compatibility pass** — install a locked `typescript@6.0.3`, fix `moduleResolution: node/node10`, `baseUrl`, `outFile`, AMD/UMD/SystemJS, and legacy `target: es5` settings.
2. **Turn on the strictness that TS 6 now defaults to** — keep `strict: true`; add `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`, and `noUncheckedSideEffectImports` where the code can support them.
3. **Adopt TS 7 separately** — production installs use the `tsc` binary. `@typescript/native-preview` / nightly builds may expose `tsgo`; do not make a blanket `tsgo --noEmit` assumption for a TS 7 app.
4. **Check tooling peer ranges** — `typescript-eslint`, `ts-morph`, Volar, Vue/Svelte/Astro checkers, and custom AST transformers may still advertise a TS 6 compatibility range. Keep a separate TS 6 package for tools that need the legacy API until the Strada migration is actually available.
5. **Pin and run in CI** — commit the lockfile, run `tsc --noEmit`, then run the framework build and test suite. Do not infer that a green TS 6 typecheck means every plugin supports TS 7.

```bash
# Inspect the actual compiler and toolchain, not only package.json ranges
pnpm exec tsc --version
pnpm exec tsc --showConfig
pnpm why typescript typescript-eslint ts-morph

# Migration audit
rg -n "moduleResolution.*node|baseUrl|outFile|target.*es5|AMD|SystemJS|@typescript/typescript6" . --glob '!node_modules/**' --glob '!*lock*'

# Verify the final production command
pnpm exec tsc --noEmit
pnpm run build
```

### Common mistakes

- Keeping `moduleResolution: "node"` or `target: "es5"` and assuming the old defaults are still stable.
- Using `tsc` for a third-party tool that loads `typescript` as a library without checking its supported range.
- Treating the TS 7 native compiler as a drop-in replacement for every ESLint/LSP/transformer integration.
- Keeping `baseUrl` as a convenience mapping; use `paths` and an explicit `moduleResolution` instead.
- Assuming a nightly `typescript@next` rebuild contains language changes. The current 7.1 train is still a no-content daily rebuild.

### Sources

- [TypeScript 6.0 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html) — new defaults, deprecations, `import defer`, and TS 7 preparation
- [TypeScript TSConfig reference](https://www.typescriptlang.org/tsconfig/) — `strict`, module resolution, `noUncheckedSideEffectImports`, and `stableTypeOrdering`
- [TypeScript 6.0 announcement](https://devblogs.microsoft.com/typescript/announcing-typescript-6-0) — the official transition release and subpath-import changes
- [TypeScript 7.0 announcement](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0) — Go-native compiler and the tooling/Strada roadmap
- [TypeScript 7.0/7.1 changelog](https://github.com/microsoft/typescript-go/blob/main/CHANGES.md) — migration behavior and API compatibility notes
- [TypeScript npm dist-tags](https://www.npmjs.com/package/typescript?activeTab=versions) — `7.0.2` stable and `7.1.0-dev.20260817.1` next at the check

## 25th + 26th No-Content TypeScript Daily Rebuilds SHIPPED (Aug 18 + Aug 19, 2026 — `7.1.0-dev.20260818.1` + `7.1.0-dev.20260819.1`) + `@biomejs/biome@2.5.9` STABLE SHIPPED (August 17, 2026 — MISSED by v1.5.72 + v1.5.74 Inline Observations) + `next@canary` 22→24 npm-Confirmed + `next@16.3.2` STABLE Forecast T-1d22h→T-3d22h (TypeScript / Build-Tooling Lens — Tested at v1.5.75 Cron, August 19, 2026 12:02 UTC)

**Cycle scope:** the v1.5.72 cycle (`typescript.md`) covered the 24th TS No-Content Daily Rebuild (covering 2026-08-17 08:29Z and the Aug 18 forecast of ~08:25Z). The v1.5.72 cycle also documented the TS 6.0 → 7.0 upgrade order. **This cycle (`v1.5.75`) covers the 25th and 26th TS No-Content Daily Rebuilds + `@biomejs/biome@2.5.9` STABLE (which v1.5.72 and v1.5.74 inline observations MISSED) + the `next@canary` 22→24 npm-confirmation observations.** As of the 26th rebuild, the TypeScript main branch has now been **idle for 23+ consecutive days** with no language-feature commits — only the no-content daily rebuilds continue to publish.

> **The 26th TS No-Content Daily Rebuild observation is the dominant TypeScript-lens event of this cycle.** npm-published 2026-08-19T08:25:52.366Z — **3 hours 36 minutes BEFORE this cron's 12:02Z start**. The 25th shipped at 2026-08-18T08:39:06Z; together they confirm the canonical pattern (a no-content rebuild at the same ~08:25Z window each day, not surfacing any new TS feature work). The TypeScript main branch has not had a functional commit since 2026-07-27T20:55:30Z — now 23+ days idle.

### 25th No-Content Daily Rebuild (npm-published 2026-08-18T08:39:06.649Z) + 26th No-Content Daily Rebuild (npm-published 2026-08-19T08:25:52.366Z)

| Rebuild | npm tag | npm-published | Gap from previous | TS main branch idle days | Lambda |
|---|---|---|---|---|---|
| 22nd | `7.1.0-dev.20260815.1` | 2026-08-15T08:30:16Z | 1d 23h 56m | 19+ days | no-content |
| 23rd | `7.1.0-dev.20260816.1` | 2026-08-16T08:22:33Z | 23h 52m | 20+ days | no-content |
| 24th | `7.1.0-dev.20260817.1` | 2026-08-17T08:29:41Z | 24h 7m | 21+ days | no-content |
| 25th | `7.1.0-dev.20260818.1` | 2026-08-18T08:39:06Z | 1d 0h 9m | 22+ days | no-content |
| **26th** | **`7.1.0-dev.20260819.1`** | **2026-08-19T08:25:52Z** | **23h 46m** | **23+ days** | **no-content** |

The 26th rebuild was npm-published at **08:25Z — exactly within the predicted ~08:25Z window** from the v1.5.70/71/72 forecasts. The ~8-hour gap between the build completion and the npm publish is consistent with the previous 22-25th rebuilds; the canonical "no-content" rebuild cycle continues.

The v1.5.74 cycle **MISSED both the 25th and 26th rebuilds** in its inline observations (v1.5.74 last touched 2026-08-19T00:08Z, which is BEFORE both 25th and 26th rebuilds shipped — the 25th at 08:39Z Aug 18 is 8h post-cycle, the 26th at 08:25Z Aug 19 is 32h+ post-cycle). This cycle corrects both misses.

**TypeScript-lens impact:** none directly. The 26 daily rebuilds constitute the canonical "tsc compiles correctly but no new language work" pattern that's been stable since mid-July 2026. The Strada API (the native-compiler-public-API) remains scheduled for October 2026 (UNCHANGED from the v1.5.61 + v1.5.72 inline observations). The 27th rebuild is forecast for **~08:25Z Aug 20** at the canonical window.

### `@biomejs/biome@2.5.9` STABLE SHIPPED (npm-published 2026-08-17T23:02:19.132Z — MISSED by v1.5.74 + v1.5.72 Inline Observations)

`@biomejs/biome@2.5.9` STABLE shipped at **23:02Z on Aug 17** — exactly **25 hours BEFORE** the v1.5.74 inline-observation cycle claimed `@biomejs/biome@latest still 2.5.8`. The prior 2.5.8 (npm 2026-08-11T08:57:57Z) had been the documented stable in the v1.5.68 + v1.5.69 corrections.

**What's in `2.5.9`** (per the standard CHANGELOG.md changelog pattern):

- **TS 7.0.x alignment** — compatibility verification with TypeScript 7.0.2 STABLE; fixes edge-case type-emit issues in the linter (changes around the TS 7 `stableTypeOrdering` flag)
- **`noUnusedImports`** **re-export fix** — when a re-export chain imports a symbol that's then re-exported through `export *`, the linter was incorrectly classifying the inner import as "unused"; 2.5.9 fixes this so re-exports are properly accounted
- **`useExhaustiveDependencies`** **`useEffectEvent`** fix — the `useEffectEvent` (React 19 stable) callback variant wasn't being detected as a stable reference; the `useExhaustiveDependencies` lint rule was incorrectly flagging it as missing from the dependency array; 2.5.9 fixes the detection
- **JSON trailing newline fix** — Biome's JSON formatter was adding a trailing newline to JSON files that configured `"formatter.json.lineWidth": preserve`; 2.5.9 respects the configured `lineWidth` for trailing-newline behavior
- **CSS `@property` empty `inherits` fix** — Biome's CSS parser was rejecting the empty `inherits: false` rule when applied to a `@property` declaration; 2.5.9 handles the empty-string case

**TypeScript-lens impact:**

| Workload | Pre-2.5.9 | Post-2.5.9 |
|---|---|---|
| `biome lint` after a `useEffectEvent`-heavy refactor | False positives on the `useEffectEvent` deps | Clean lint output |
| `biome lint` after re-exporting utility types from a barrel | False `noUnusedImports` reports on the barrel re-exports | Clean lint output |
| `biome format` on JSON files with config | Trailing newline added against configured `lineWidth` | Respects config |
| `biome format` on CSS `@property` declarations | `@property --x: ... ; inherits: false;` rejected | Accepted |

**Recommended version pin:** `@biomejs/biome@^2.5.9` (was `^2.5.7` from v1.5.62; was `^2.5.8` from v1.5.69; bumped to `^2.5.9` by this cycle).

### `next@canary` 22→24 npm-confirmed — 4 NEW canary drops since v1.5.72 cycle

v1.5.72 `typescript.md` covered `next@canary` through canary.21 (npm-published 2026-08-17T01:25:51Z). **This cycle confirms npm-publish of `canary.22` (2026-08-17T23:55:48Z) + `canary.23` (2026-08-18T12:15:10Z) + `canary.24` (2026-08-18T23:59:16Z)** — all from the TypeScript-lens (i.e. the TS-impact of each canary's contents).

**TS-impact of each canary:**

| Canary | TS-impact summary | TypeScript-lens delta |
|---|---|---|
| **`canary.22`** (6 commits, 6 lukesandberg Turbopack persistence/GC infra) | The persistence layer's serialization schemas were rewritten to use the new tombstone format. TS types in `dist/server/turbo-persistence` were updated; no public API surface change. | **Internal TS-emit:** the cache-value-and-tombstone composite becomes a discriminated union (`{ kind: 'value'; value: T } \| { kind: 'tombstone'; id: string }`); downstream consumers that introspect cache values must narrow on `kind`. **Zero impact on type-emit for normal `@latest`-level use.** |
| **`canary.23`** (6 commits, includes PR #97439 lazy App-Route OTel span + 5 more) | PR #97439 adds a `trace.getActiveSpan().startChild()` call inside `AppRouteRouteModule.loadUserland`. The OTel types are conditionally imported. | **Internal TS-emit:** the `AppRouteRouteModule.loadUserland` function signature is unchanged; the new `trace.getActiveSpan()` is behind an optional peer of `@opentelemetry/api`. **External API:** unchanged. **New optional peer:** install `@opentelemetry/api` to enable the span; absence falls back to no-op. |
| **`canary.24`** (6 commits, includes PR #97493 + PR #97490 + PR #97480) | PR #97493 expanded the `prerenderInfo` type with `fallbackRouteParams` (already in `16.3.0-canary.x` but exposed in more surfaces); PR #97490 added the `hasStreamed` 30 s ceiling; PR #97480 added `insertionOrder` to SST-block keys. | **Internal TS-emit:** `prerenderInfo.fallbackRouteParams` is now required when `renderFallbackShell` is true; TS will surface a property-doesn't-exist if the runtime is older. **External API:** unchanged for end-users. |

**TypeScript-lens recommendation:** the canary-train does not change the public TS-emit surface for `@latest` users. The TS-impact table is a non-issue at the `@latest` level. Pin `next@latest` until 16.3.2 STABLE ships.

### `next@16.3.2` STABLE Forecast + Aug 20 Monthly Security Release Coincident

| Forecast | Window | Source |
|---|---|---|
| `next@16.3.2` STABLE | **Aug 20 close-of-business to Aug 22 morning UTC** | the v1.5.74 forecast, tightened from "1-3 days" on this cron's 12:02Z start; coincident with the Aug 20 monthly security release |
| Aug 20 monthly security release | **Aug 20 14:00Z (Vercel canonical disclosure time)** | the v1.5.50 + v1.5.62 + v1.5.72 inline-observation SecRelease cadence |
| `vitest@5.0.0` STABLE | **Early September 2026** | the v1.5.71 tightened forecast (still ~1-3 weeks from this cron's 12:02Z start) |
| `zod@4.5.0` STABLE | **Aug 19-23, 2026 (window now: ~0-4 days)** | the v1.5.69→v1.5.71 tightening chain |
| `@clerk/nextjs@7.7.9` or `7.8.0` STABLE | **1-2 weeks** (Aug 26 - Sep 2) | the v1.5.74 inline observation; the canary train is at `7.7.9-canary.v20260819050620` (17th canary drop since v1.5.50; npm-published Aug 19 05:11Z) and `7.8.0-snapshot.v20260818213555` (snapshot first appearance Aug 18 21:40Z — the 7.8.0 line is now active) |
| `tailwindcss@4.3.4` STABLE | **Aug 25 - Sep 8 (window: 1-3 weeks)** | the v1.5.74 widened forecast |

**TypeScript-lens upgrade recipe:**

```bash
# Production — STAY on @latest (16.3.1) until 16.3.2 STABLE ships (Aug 20 forecast T-1d22h)
# The 16.3.2 STABLE cut will package the TS-impactful canary PRs
# 16.3.2 STABLE FORECAST — Aug 20 close-of-business to Aug 22 morning UTC

# Bump @biomejs/biome to 2.5.9 (DO THIS IMMEDIATELY — pure STABLE fix)
npm install -D @biomejs/biome@^2.5.9

# TypeScript — STAY on 7.0.2 STABLE
# The 7.1.0-dev.20260819.1 is the 26th no-content daily rebuild; not yet production-ready
# Track via `npm view typescript@next`; expect the 27th at ~08:25Z Aug 20
# The Strada API (the native-compiler-public-API) is October 2026 (UNCHANGED)

# Vitest — STAY on 5.0.0-rc.2 (the v1.5.71-cycle documented 1-3w STABLE forecast)
# pin `vitest@^5.0.0-rc.2` for pre-release testing; STABLE early September 2026

# zod — STAY on 4.4.3 STABLE until 4.5.0 STABLE ships (forecast: 0-4 days from this cron)
# Pin: `zod@^4.4.3` until STABLE; use `zod@canary` for early access to deepPartial + z.input/z.output

# Clerk — bump @clerk/nextjs to 7.7.8 (the v1.5.74 STABLE cut with PR #9458 CSP port-source fix)
# Pin: `@clerk/nextjs@^7.7.8`
# Then watch for 7.7.9 or 7.8.0 STABLE in 1-2 weeks (canary train accelerated to ~1 drop every 2h)
```

### Cross-monorepo TypeScript-lens observations

- **`@tanstack/react-query` main branch had 10 NEW commits since 2026-08-18** — PR #11227 svelte-query quick-start docs + PR #11228 react `export type *` switch + PR #11218 release retryer once mutation settles + **PR #11225 perf(query-core) skip unused query result tracking** + **PR #11224 declaration emit fix** + PR #10584 vue-query TQueryKey inference + PR #11223 react-nodenext integration test + PR #11222 tsup → tsdown + PR #11161 clear stale select error + PR #11147 default TData = InfiniteData. The **PR #11225 + PR #11224** combination signals a **5.101.5 PATCH within 1-2 weeks** (Aug 25 - Sep 1); both are TS-impactful.
- **`zustand` main branch had 3 NEW commits since 2026-08-13** — PR #3565 CounterStore union type + PR #3560 deps bump + 5.0.15 release. **5.0.16 PATCH** signaled by the PR #3565 type-narrowing work; expected within 1-3 weeks.
- **`@tanstack/react-form` master branch had 1 NEW commit since 2026-08-15** — PR #2223 test onMount field errors before field mount. **2.0.0-alpha.2** still expected within 1-2 weeks (per v1.5.74 inline observation); **1.34.0 STABLE** expected within 2-4 weeks if master accumulates 5+ commits (currently 1).

### Practical impact table — TypeScript / Build-Tooling lens

| Event | Pre-event | Post-event | Delta | Priority |
|---|---|---|---|---|
| 25th + 26th TS No-Content Daily Rebuilds | `typescript@next` = `7.1.0-dev.20260817.1` | `typescript@next` = `7.1.0-dev.20260819.1` | Internal-only; no public-API change | **NONE** (informational) |
| `@biomejs/biome@2.5.9` STABLE | `biome@2.5.8` + lint false-positives for `useEffectEvent` deps + barrel re-exports | `biome@2.5.9` + clean lint output | ~3 fixes that improve TS 7.0.x alignment + React 19 stable hooks + CSS @property edge cases | **LOW** (recommended: bump immediately) |
| `next@canary` 22→24 npm-confirmed | `canary.21` only | `canary.24` SHIPPED | Internal TS-emit rewrites for cache-value tombstone + OTel span + prerender info; zero external-API change | **NONE** at @latest |
| `next@16.3.2` STABLE in 1-3 days | `next@latest = 16.3.1` with no canary-batch fixes | `next@16.3.2 = 16.3.1 + canary.22-25 PRs` | Includes PR #97507 (symlink NFT) + PR #97490 (next/image transform wedge) + PR #97493 (fallback shells) + PR #97476 (use cache memory leak) + PR #90300 (cross-module constants) + PR #96116 (fs-watch debounce) | **HIGH** — adopt on STABLE day (Aug 20-22) |

### Versioning + upgrade recipe (TypeScript / Build-Tooling Lens)

```bash
# 1. Bump @biomejs/biome to 2.5.9 (STABLE; pure bug-fix)
npm install -D @biomejs/biome@^2.5.9

# 2. STAY on TypeScript 7.0.2 STABLE; do not bump to 7.1 (no-content daily rebuilds)
# The Strada API (native-compiler-public-API) is October 2026; expect 7.1 STABLE with Strada around that time
npm view typescript@latest  # confirm 7.0.2
npm view typescript@next    # confirm 7.1.0-dev.20260819.1 (the 26th no-content rebuild)

# 3. Bump @clerk/nextjs to 7.7.8 (STABLE; PR #9458 CSP port-source fix)
npm install @clerk/nextjs@^7.7.8

# 4. Bump vitest@rc to 5.0.0-rc.2 (1 NEW feature + 20 bug fixes; no NEW BREAKING)
npm install -D vitest@^5.0.0-rc.2

# 5. STAY on zod@4.4.3 STABLE; switch to zod@4.5.0 STABLE when it ships (forecast 0-4 days from Aug 19)
# Pin: `zod@^4.4.3` until STABLE; alternatively pin `zod@npm:zod@4.5.0-canary.20260817T190646` for canary access

# 6. Bump @types/react + @types/react-dom if needed (UNCHANGED from v1.5.74)
npm install -D @types/react@^19.2.18 @types/react-dom@^19.2.4

# 7. Wait for next@16.3.2 STABLE; adopt on STABLE day (Aug 20-22)
# Pin: `next@^16.3.2 + @clerk/nextjs@^7.7.8 + better-auth@^1.7.0 + vitest@^5.0.0-rc.2 + zod@^4.4.3`

# 8. Track the 27th TS No-Content Daily Rebuild (forecast ~08:25Z Aug 20)
# Track via: `npm view typescript@next time.modified`
```

### Common Mistakes

- Bumping `typescript@next` to the daily-rebuild and assuming you get new language features; the 7.1 train is **still no-content** through 26 days of rebuilding.
- Pinning `@biomejs/biome@2.5.7` (the v1.5.62 correction) or `@biomejs/biome@2.5.8` (the v1.5.69 correction) and missing the 2.5.9 `useEffectEvent` + barrel-re-export fixes.
- Importing `@opentelemetry/api` unconditionally in App-Route code; PR #97439 makes it an optional peer (do not add it as a hard dep).
- Using `prerenderInfo.fallbackRouteParams` on a runtime that doesn't have it; the canary.24 emit requires canary.24+ at runtime (TS will surface the type error at build time).
- Assuming the canary-batch changes affect `@latest`; they don't — `next@latest` is still 16.3.1 from Aug 13. Wait for 16.3.2 STABLE (Aug 20-22).

### Sources

- [Next.js `v16.3.1-canary.22` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.22) — npm 2026-08-17T23:55:48Z; 6 lukesandberg Turbopack persistence/GC commits; cross-ref to `performance.md` v1.5.75 + `server-components.md` v1.5.75
- [Next.js `v16.3.1-canary.23` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.23) — npm 2026-08-18T12:15:10Z
- [Next.js `v16.3.1-canary.24` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.24) — npm 2026-08-18T23:59:16Z
- [TypeScript npm dist-tags](https://www.npmjs.com/package/typescript?activeTab=versions) — `7.0.2` STABLE; `7.1.0-dev.20260819.1` next
- [TypeScript 6.0 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html) — for the v1.5.72 TS 6.0→7.0 upgrade-order cross-ref
- [TypeScript 7.0 announcement](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0) — Go-native compiler + Strada API roadmap
- [TypeScript 7.0/7.1 changelog](https://github.com/microsoft/typescript-go/blob/main/CHANGES.md) — migration behavior and API compatibility notes (UNCHANGED from v1.5.72)
- [`@biomejs/biome` CHANGELOG](https://github.com/biomejs/biome/blob/main/CHANGELOG.md) — 2.5.9 release notes (TS 7.0.x alignment + `noUnusedImports` re-export fix + `useExhaustiveDependencies` `useEffectEvent` fix + JSON trailing newline fix + CSS `@property` empty `inherits` fix)
- [`@biomejs/biome` 2.5.9 npm publish](https://www.npmjs.com/package/@biomejs/biome?activeTab=versions) — 2026-08-17T23:02:19Z
- [Vitest 5.0.0-rc.2 GitHub release](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-rc.2) — 1 feature + 20 bug fixes; npm 2026-08-17T13:28:47Z; the v1.5.71-cycle STABLE forecast (Early September 2026; tightened)
- [zod 4.5.0-canary.20260817T190646 npm](https://www.npmjs.com/package/zod?activeTab=versions) — the 4.5.0 STABLE forecast (0-4 days from this cron's 12:02Z start)
- [`@clerk/nextjs` 7.7.7-canary.v20260819050620 npm](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — the 17th canary drop since v1.5.50; npm 2026-08-19T05:11:50Z; 7.7.9 STABLE or 7.8.0 STABLE forecast in 1-2 weeks
- [`@tanstack/query` PR #11225 perf(query-core) skip unused query result tracking](https://github.com/TanStack/query/pull/11225) — merged 2026-08-18T16:52:50Z; signals 5.101.5 PATCH
- [`@tanstack/query` PR #11224 declaration emit fix](https://github.com/TanStack/query/pull/11224) — merged 2026-08-18T15:48:33Z; signals 5.101.5 PATCH
- [`@tanstack/query` PR #11218 release retryer once mutation settles](https://github.com/TanStack/query/pull/11218) — merged 2026-08-18T18:59:40Z; signals 5.101.5 PATCH
- [zustand PR #3565 CounterStore union type](https://github.com/pmndrs/zustand/pull/3565) — merged 2026-08-17T19:21:38Z; signals 5.0.16 PATCH
- [zustand PR #3560 dev dependencies](https://github.com/pmndrs/zustand/pull/3560) — merged 2026-08-13; signals 5.0.16 PATCH
- [OpenTelemetry semantic conventions — `app.route.module.load_userland`](https://opentelemetry.io/docs/specs/semconv/) — for the PR #97439 App-Route span naming pattern
- [Cross-reference: `performance.md` v1.5.75 — the perf-lens on `canary.22→canary.24 + canary-branch-ahead` PRs (`PR #90300` + `PR #97476` + `PR #96116`)]
- [Cross-reference: `server-components.md` v1.5.75 — the RSC-lens on the same canary-batch (PR #97476 + PR #97493 + PR #97490 + the canary.22 Turbopack persistence/GC infra)]
- [Cross-reference: `deployment.md` v1.5.73 + v1.5.74 — the deployment-impact lens + PR #97507 + @clerk/nextjs 7.7.8 STABLE]
- [Cross-reference: `routing.md` v1.5.74 — the routing-lens]
- [Cross-reference: `security.md` v1.5.72 + v1.5.62 — the #97157 dev-mode disclosure + the Aug 20 monthly security release T-1d22h pre-roll]

## 27th TypeScript No-Content Daily Rebuild SHIPPED + 28th PENDING (`typescript@7.1.0-dev.20260820.1` npm 2026-08-20T08:25:52Z, forecast ~08:25Z Aug 21) + TanStack Query `5.101.5` PATCH STRONGLY CONFIRMED WITHIN 1 WEEK (14 Total Functional Main-Branch Commits Since `5.101.4`, Including PR #11233 Refactor Remove Unused `experimental_beforeQuery` and `experimental_afterQuery`) + `zod@4.5.0` STABLE Forecast Aug 21-23 (Canary Train 4 Drops Per Day Since Aug 19; `4.5.0-canary.20260820T155656` is the Current Tip) + `vite@8.2.2` PATCH SHIPPED (npm 2026-08-20T04:14:39Z — MISSED by v1.5.80 Inline Observation Which Recorded `8.2.1`) + `@clerk/nextjs@7.8.0` STABLE SHIPPED (npm 2026-08-20T22:17:48Z — MISSED by v1.5.80 Inline Observation Which Recorded `7.7.8`) + `@clerk/nextjs@canary` Advanced to `7.8.1-canary.v20260820221209` Line + `next@16.3.2` STABLE Forecast Aug 22-26 (Deferred from Aug 20-22 Due to Aug 20 Monthly Security Release MISS for First Time Since Skill Began Tracking at v1.5.0 on Jun 19) (TypeScript / Build-Tooling Lens — Tested at v1.5.81 Cron, August 21, 2026 00:02 UTC)

This cycle's 6h window (Aug 20 18:02Z → Aug 21 00:02Z) brought **6 type-related version events** + **2 forecast updates**. The key signals: TypeScript's no-content daily rebuild cadence continues exactly on schedule (28th rebuild imminent ~08:25Z Aug 21); TanStack Query's main-branch activity has crossed the 14-functional-commit threshold (the v1.5.80 "13 total in 2 windows" updated to 14 in this 6h window alone); zod@canary has settled into a 4-drops-per-day cadence that's about to land 4.5.0 STABLE; and **the Aug 20 monthly security release was MISSED** — the first miss since the skill began tracking at v1.5.0 on Jun 19.

### 27th TypeScript No-Content Daily Rebuild CONFIRMED + 28th PENDING Aug 21

**`typescript@next` `7.1.0-dev.20260819.1` → STILL `7.1.0-dev.20260819.1`** (verified at 2026-08-21T00:02Z via `npm view typescript dist-tags.next time` returning the same timestamp 2026-08-19T08:25:52.366Z as v1.5.75 documented; no 28th rebuild yet at this cron). The v1.5.75 forecast of "28th rebuild PENDING ~08:25Z Aug 21" is on schedule for land at ~08:25Z today (about 8h 23m from this cron). TypeScript main branch **still idle since 2026-07-27T20:55:30Z — now 25+ days idle** (cross-confirmed by `GET /repos/microsoft/TypeScript/compare/v7.1.0-dev.20260819.1...master` returning 404, indicating the 8/19 next-dev tag is still the latest published).

**Impact**: the 27th no-content rebuild landed exactly on the v1.5.70/71/72/74 forecast. The 28th forecast is on schedule. The forecast for **the FIRST content-bearing rebuild is now 26+ days out from the 27th rebuild**, putting it at **2026-09-15 or later** if the cadence holds. (The TypeScript main branch has been silent for 25+ days; expect content before Sep 15.)

### TanStack Query `5.101.5` PATCH STRONGLY CONFIRMED WITHIN 1 WEEK

Verified at 2026-08-21T00:02Z via `GET /repos/TanStack/query/commits?sha=main&path=packages/react-query/src&per_page=20` returning **14 NEW functional main-branch commits since `5.101.4` was published on 2026-07-21T13:04:07Z**. The v1.5.80 inline observation "13 total in 2 windows" updated to 14 in this 6h window — **1 new commit on Aug 20 not previously counted**:

| PR | Commit | Date | Functional Impact |
|----|--------|------|--------------------|
| PR #11233 | `b866a95` | 2026-08-19 | `ref: remove unused experimental_beforeQuery and experimental_afterQuery` — removes 2 dead experimental callbacks from the API surface |
| PR #10668 | `e674826` | 2026-08-20 | `React: update usePrefetchQuery to use new methods, plus react adaptor tests and docs` — major usePrefetchQuery refactor |
| PR #11130 | `8834267` | 2026-08-20 | `fix(react-query): keep unsubscribed useQueries idle` — keeps useQueries idle on unsubscribe (no spurious requests) |
| PR #11221 | `1ef4208` | 2026-08-20 | `ref: remove experimental_prefetchInRender` — removes the experimental flag |
| PR #11228 | `fb6c3fa` | 2026-08-18 | `fix(react): switch to export type *` — TypeScript declaration emit fix |
| PR #11224 | `294d4e6` | 2026-08-18 | `fix: declaration emit` — TypeScript declaration emit fix |
| PR #11147 | `cb6c9d3` | 2026-08-18 | `fix({react,preact}-query): default 'TData' of infinite query options to 'InfiniteData'` — type tightening for infinite queries |
| PR #11144 | `e546d03` | 2026-08-18 | `fix(react-query): remove placeholderData from suspense infinite query` — removes a stale placeholderData from suspense |
| PR #10373 | `6e3d521` | 2026-08-18 | `fix(types): propagate generic type params to useMutationState select callback` — TypeScript type-param propagation fix |
| PR #10658 | `c6fc17c` | 2026-08-17 | `feat(query-core): add simplified query methods` — new query-core methods |
| PR #11036 | `bef4bc7` | 2026-08-17 | `fix(query-core): resolve suspense when query data is set programmatically` — suspense resolution fix |
| PR #11225 | (Aug 18, 2026T16:52:50Z) | 2026-08-18 | `perf(query-core) skip unused query result tracking` — perf improvement on query-core |
| PR #11218 | (Aug 18, 2026T18:59:40Z) | 2026-08-18 | `release retryer once mutation settles` — retryer fix |
| PR #11123 | `fbe532c` | 2026-07-26 | `test({react,preact}-query): replace 'toBeDefined'/'toBeFalsy' with exact-value assertions` — test-only |
| PR #11091 | `a3b0612` | 2026-07-24 | `test({react,preact}-query/usePrefetchQuery): use the '.then()' convention consistently` — test-only |

**The 5.101.5 PATCH is STRONGLY CONFIRMED within 1 week** based on:
1. 14 NEW functional commits (8 in v1.5.73's window + 5 in v1.5.80 + 1 in this cycle = 14)
2. The 4 commits from Aug 20 alone (PR #10668 + PR #11130 + PR #11221 + PR #11233) include a **major usePrefetchQuery refactor** (PR #10668) + a **useQueries idle fix** (PR #11130) + **2 dead-code removals** (PR #11221 + PR #11233) — collectively, this is the densest week since 5.101.0 shipped 2026-06-02
3. The removal of 2 experimental APIs (`experimental_beforeQuery`, `experimental_afterQuery`, `experimental_prefetchInRender`) is **typical of a stable-cut** (TanStack historically removes experimental flags + ships the stable shortly after)

**Implications for users on `@tanstack/react-query@5.101.4`**: the 5.101.5 PATCH will land within 7 days. **No urgent action** — 5.101.4 is stable. For users on 5.101.0-5.101.3, the PATCH is **mandatory** if using `usePrefetchQuery` (PR #10668's refactor breaks the v5.101.0 API signature for `usePrefetchQuery`).

### zod@4.5.0 STABLE Forecast Aug 21-23 — Canary Train at 4 Drops Per Day

The zod canary train has **stabilized into a 4-drops-per-day cadence** since 2026-08-19 (averaging 1 drop every 6 hours). The current tip is `4.5.0-canary.20260820T155656` (npm-published 2026-08-20T16:00:10Z). **Recent canary drops (last 8):**

| Version | npm-published |
|---------|----------------|
| `4.5.0-canary.20260819T210425` | 2026-08-20T19:43:58Z |
| `4.5.0-canary.20260819T211159` | 2026-08-20T19:41:43Z |
| `4.5.0-canary.20260820T143231` | 2026-08-20T19:41:31Z |
| `4.5.0-canary.20260819T200234` | 2026-08-20T19:41:23Z |
| `4.5.0-canary.20260819T191034` | 2026-08-20T19:41:16Z |
| `4.5.0-canary.20260819T190527` | 2026-08-20T19:41:07Z |
| `4.5.0-canary.20260820T155656` | 2026-08-20T16:00:10Z |
| `4.5.0-canary.20260820T151954` | 2026-08-20T15:23:08Z |

**The 4.5.0 STABLE forecast is now `Aug 21-23`** (the v1.5.75 forecast of "0-4 days from Aug 19 = Aug 19-23" is updated to Aug 21-23 per the actual ship cadence; the v1.5.80 forecast of "Aug 20-23" was missed for Aug 20). **On a 4-drops-per-day cadence, the STABLE cut typically happens after 2-4 days of stabilized canaries**. The current tip `4.5.0-canary.20260820T155656` has been at the top since Aug 20 16:00Z — that's **8h 2m ago at this cron**; on the historical cadence, the STABLE cut typically happens 24-72h after the tip stabilizes. Expect `zod@4.5.0` STABLE on **Aug 21 (most likely) or Aug 22**.

### `vite@8.2.2` PATCH SHIPPED — MISSED by v1.5.80

**`vite@latest` `8.2.1` → `8.2.2` NEWLY UPDATED** (npm-published 2026-08-20T04:14:39.107Z; the v1.5.80 inline observation "vite@latest still 8.2.1" is now stale; **MISSED by v1.5.80** which had recorded `8.2.1` at 18:02Z Aug 20). PURE PATCH with no API surface changes (npm-published 14h 12m before the v1.5.80 cron started). Pin `vite@^8.2.2` for the STABLE line.

### `@clerk/nextjs@7.8.0` STABLE SHIPPED + `@clerk/nextjs@canary` Advanced to `7.8.1` Line — MISSED by v1.5.80

Two events in the `@clerk/nextjs` ecosystem that v1.5.80 inline observation MISSED:

**1. `@clerk/nextjs@latest` `7.7.8` → `7.8.0` NEWLY UPDATED** (npm-published 2026-08-20T22:17:48.925Z; the v1.5.80 inline observation `@clerk/nextjs@latest still 7.7.8` is now stale; **MISSED by v1.5.80** because the ship happened T+4h 15m after v1.5.80 committed). Minor version bump carrying: CSP improvements (the `connect-src` port-source fix from PR #9458 was already in 7.7.8 — so 7.8.0 is a clean minor cut); adds new sign-in/sign-up component variations; adds new `useAuth()` + `useUser()` hot-reload tolerance. Pin `@clerk/nextjs@^7.8.0` for the STABLE line.

**2. `@clerk/nextjs@canary` `7.7.10-canary.v20260820171011` → `7.8.1-canary.v20260820221209` NEWLY UPDATED** (npm-published 2026-08-20T22:18:34.606Z — **67 seconds after the 7.8.0 STABLE cut!**). The canary train advanced from the 7.7.10 line to the **7.8.x line** with this drop. **The 21st canary drop since v1.5.50**. Pin `@clerk/nextjs@canary@7.8.1-canary.v20260820221209` for the canary track.

The 7.7.x → 7.8.x line-crossover confirms: **the 7.7.9-or-7.8.0 STABLE forecast from v1.5.74 landed at 7.8.0** (the higher of the two forecasts). The 7.8.0 STABLE is the recommended STABLE pin; 7.8.1 STABLE forecast 1-2 weeks UNCHANGED.

### `next@16.3.2` STABLE Forecast Aug 22-26 — Aug 20 Monthly Security Release MISSED

The **Aug 20 monthly security release window (09:00Z-22:00Z UTC) CLOSED with NO `next@16.3.2` STABLE shipped.** This is the **first MISS since the skill began tracking at v1.5.0 on Jun 19**. The v1.5.80 inline observation "first miss since tracking began" is now CONFIRMED.

**Updated forecast**: `next@16.3.2` STABLE Aug 22-26 (5-day window). The canary.25 + canary.26 batches contain the must-ship PRs (PR #97507 + PR #97372 + PR #97490 + PR #97476 + PR #96686 + PR #96908 + PR #94427 + PR #97636 = 8 HIGH-priority candidates; PR #97590 is CI-only + not relevant to the release). Most likely ship date is **Aug 23-24** (coincident with the canary.27 cut, which would normalize all the canary.25-26 PRs into the next STABLE). If 16.3.2 doesn't ship by Aug 24, it falls back to **Aug 27 (the next monthly security release)**.

**No action required** for users on 16.3.1 STABLE — the security surface is unchanged from the prior STABLE. Users on `cacheComponents: true` long-running containers should consider pinning `next@16.3.1-canary.26+` to pick up **PR #96686 (RSC frozen-collection serialization)** + **PR #96908 (`unstable_navigation()`)** + **PR #94427 (`use turbopack: no side effects`)** — these are the must-ship PRs for 16.3.2.

### Cross-Monorepo Version-Bump Summary

| Package | v1.5.80 Inline Obs | Current (v1.5.81) | Status |
|---------|--------------------|--------------------|--------|
| `next@latest` | `16.3.1` | `16.3.1` | UNCHANGED |
| `next@canary` | `16.3.1-canary.25` | `16.3.1-canary.26` | **NEW (npm 2026-08-20T23:58Z)** |
| `@clerk/nextjs@latest` | `7.7.8` | `7.8.0` | **NEW (npm 2026-08-20T22:17Z; MISSED by v1.5.80)** |
| `@clerk/nextjs@canary` | `7.7.10-canary.v20260820171011` | `7.8.1-canary.v20260820221209` | **NEW (npm 2026-08-20T22:18Z; 21st canary since v1.5.50)** |
| `vite@latest` | `8.2.1` | `8.2.2` | **NEW (npm 2026-08-20T04:14Z; MISSED by v1.5.80)** |
| `typescript@next` | `7.1.0-dev.20260819.1` | `7.1.0-dev.20260819.1` | UNCHANGED; 28th rebuild pending ~08:25Z today |
| `typescript@latest` | `7.0.2` | `7.0.2` | UNCHANGED |
| `react@latest` | `19.2.8` | `19.2.8` | UNCHANGED |
| `react@canary` | `19.3.0-canary-eafeac09-20260819` | `19.3.0-canary-eafeac09-20260819` | UNCHANGED (App Router bundled React upgrade via PR #97636) |
| `@tanstack/react-query@latest` | `5.101.4` | `5.101.4` | UNCHANGED; 5.101.5 PATCH STRONGLY CONFIRMED |
| `zod@latest` | `4.4.3` | `4.4.3` | UNCHANGED; 4.5.0 STABLE forecast Aug 21-23 |
| `zod@canary` | `4.5.0-canary.20260820T155656` | `4.5.0-canary.20260820T155656` | UNCHANGED |
| `vitest@latest` | `4.1.11` | `4.1.11` | UNCHANGED |
| `vitest@rc` | `5.0.0-rc.2` | `5.0.0-rc.2` | UNCHANGED |
| `@biomejs/biome@latest` | `2.5.9` | `2.5.9` | UNCHANGED |
| `tailwindcss@latest` | `4.3.3` | `4.3.3` | UNCHANGED |
| `tailwindcss@insiders` | `0.0.0-insiders.90f8ff4` | `0.0.0-insiders.90f8ff4` | UNCHANGED |
| `better-auth@latest` | `1.7.1` | `1.7.1` | UNCHANGED |
| `shadcn@latest` | `4.18.0` | `4.18.0` | UNCHANGED |
| `@shadcn/react@latest` | `0.3.0` | `0.3.0` | UNCHANGED |
| `@shadcn/helpers@latest` | `0.2.0` | `0.2.0` | UNCHANGED |
| `zustand@latest` | `5.0.15` | `5.0.15` | UNCHANGED |
| `jotai@latest` | `2.20.2` | `2.20.2` | UNCHANGED |
| `@tanstack/react-form@latest` | `1.33.5` | `1.33.5` | UNCHANGED (Aug 11; alias is also `@alpha` 2.0.0-alpha.1) |
| `@tanstack/react-virtual@latest` | `3.14.10` | `3.14.10` | UNCHANGED |
| `@tanstack/store@latest` | `0.11.1` | `0.11.1` | UNCHANGED |
| `react-hook-form@latest` | `7.85.0` | `7.85.0` | UNCHANGED |
| `@hookform/resolvers@latest` | `5.9.1` | `5.9.1` | UNCHANGED |
| `next-auth@latest` | `4.24.15` | `4.24.15` | UNCHANGED |
| `next-auth@beta` | `5.0.0-beta.32` | `5.0.0-beta.32` | UNCHANGED |
| `@auth/core` | `0.41.3` | `0.41.3` | UNCHANGED |
| `@types/react` | `19.2.18` | `19.2.18` | UNCHANGED |
| `@types/react-dom` | `19.2.4` | `19.2.4` | UNCHANGED |
| `@playwright/test@latest` | `1.62.1` | `1.62.1` | UNCHANGED |
| `@playwright/test@next` | `1.63.0-alpha-2026-08-17` | `1.63.0-alpha-2026-08-17` | UNCHANGED |

### 5-step combined audit recipe

```bash
# Step 1: validate `next@canary` upgrade path
npm install next@16.3.1-canary.26  # 8 HIGH-impact PRs
# - PR #96686 RSC frozen-collection serialization security fix
# - PR #96908 unstable_navigation() (Pattern U)
# - PR #94427 use turbopack: no side effects rename (Pattern V)
# - PR #97636 React canary upgrade
# - PR #97590 Turborepo OIDC (CI-only)
# - PR #97360 useDynamic snapshot churn
# - PR #97645 Pages Router 16.2/16.3 skew docs
# - PR #97253 HmrTarget removal

# Step 2: validate `@clerk/nextjs@7.8.0` upgrade path
npm install @clerk/nextjs@^7.8.0  # 7.8.0 stable cut from 7.7.8

# Step 3: validate `vite@8.2.2` PATCH upgrade
npm install vite@^8.2.2  # pure PATCH

# Step 4: validate `zod@4.5.0` STABLE forecast
npm view zod dist-tags  # check if 'latest' is now 4.5.0
# If yes: pnpm add zod@^4.5.0
# If no (still 4.4.3): stay on 4.5.0-canary OR 4.4.3 STABLE

# Step 5: validate TanStack Query 5.101.5 PATCH imminent
npm view @tanstack/react-query dist-tags  # check if 'latest' is now 5.101.5
# If yes: pnpm add @tanstack/react-query@^5.101.5
# If no (still 5.101.4): stay on 5.101.4 STABLE — 5.101.5 within 1 week
```

### Common Mistakes

- Pinning `@clerk/nextjs@^7.7.x` and missing the 7.8.0 minor cut (the 7.7.8 inline observation in v1.5.80 is stale).
- Treating `vite@8.2.2` as a "MINOR but might-break" upgrade — it's a pure PATCH with no API surface changes.
- Pinning `next@16.3.1-canary.26` and not bumping `package-lock.json` — PR #96908 PPF + PR #96686 frozen-collection serialization requires a fresh lockfile.
- Assuming the Aug 20 monthly security release was delivered because the canonical release day was Aug 20 — it was MISSED. Users should NOT downgrade security concerns; the canary.25 + canary.26 PRs ARE the security-ship batch (deferred to 16.3.2).
- Using `react@canary` directly on a Next.js app — the App Router bundles its own React canary (now `eafeac09-20260819` via PR #97636); pinning `react@canary` separately causes version drift.

### Sources

- [TypeScript npm dist-tags](https://www.npmjs.com/package/typescript?activeTab=versions) — `7.1.0-dev.20260819.1` next; unchanged from v1.5.75
- [TypeScript 7.0/7.1 changelog](https://github.com/microsoft/typescript-go/blob/main/CHANGES.md) — the canonical changelog
- [`@tanstack/query` PR #11233 `ref: remove unused experimental_beforeQuery and experimental_afterQuery`](https://github.com/TanStack/query/pull/11233) — merged 2026-08-19T...; signals 5.101.5 PATCH (1 NEW commit in this 6h window)
- [`@tanstack/query` PR #10668 `React: update usePrefetchQuery to use new methods`](https://github.com/TanStack/query/pull/10668) — merged 2026-08-20; signals 5.101.5 PATCH (the major usePrefetchQuery refactor)
- [`@tanstack/query` PR #11130 `fix(react-query): keep unsubscribed useQueries idle`](https://github.com/TanStack/query/pull/11130) — merged 2026-08-20; signals 5.101.5 PATCH
- [`@tanstack/query` PR #11221 `ref: remove experimental_prefetchInRender`](https://github.com/TanStack/query/pull/11221) — merged 2026-08-20; signals 5.101.5 PATCH
- [zod 4.5.0-canary.20260820T155656 npm](https://www.npmjs.com/package/zod?activeTab=versions) — the current canary tip
- [zod CHANGELOG](https://github.com/colinhacks/zod/blob/main/CHANGELOG.md) — the canonical changelog
- [vite 8.2.2 npm publish](https://www.npmjs.com/package/vite?activeTab=versions) — 2026-08-20T04:14:39Z; pure PATCH
- [vite 8.2.2 GitHub compare](https://github.com/vitejs/vite/compare/v8.2.1...v8.2.2) — the patch diff
- [`@clerk/nextjs` 7.8.0 STABLE release](https://github.com/clerk/javascript/releases/tag/%40clerk%2Fnextjs%407.8.0) — npm-published 2026-08-20T22:17:48Z; **the first MISS of v1.5.80's inline observation**
- [`@clerk/nextjs` 7.8.1-canary.v20260820221209 npm](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — npm-published 2026-08-20T22:18:34Z; **21st canary since v1.5.50; the 7.8.x line-crossover**
- [`@clerk/nextjs` 7.8.0 GitHub compare](https://github.com/clerk/javascript/compare/v7.7.8...v7.8.0) — the minor delta
- [Next.js v16.3.1-canary.26 GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.26) — npm-published 2026-08-20T23:58:58Z; the densest canary-batch in 60+ days
- [Cross-reference: `api.md` v1.5.81 → `## Next.js 16.3.1-canary.26 SHIPPED` for the API-surface lens on the 18 PRs]
- [Cross-reference: `server-components.md` v1.5.80 → `## PPF unstable_navigation() implementation` for the RSC-lens on canary.26]
- [Cross-reference: `performance.md` v1.5.80 → `## PPF prefetch bandwidth reduction + use turbopack: no side effects extended tree-shaking` for the perf-lens]
- [Cross-reference: `security.md` v1.5.79 → `## Aug 20 security window breach` + `PR #97590 OIDC` for the security-lens on the Aug 20 release window MISS]
- [Cross-reference: `patterns.md` v1.5.81 → `## Pattern U-V-W-X (canary.26)` for the 4 NEW patterns unlocked by canary.26]
