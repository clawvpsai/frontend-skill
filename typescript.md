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
    "baseUrl": ".",
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

**Current status:** ~~Release Candidate (Version 7.0.1-rc)~~ — **General Availability (Version 7.0.0) shipped July 8, 2026** ([official announcement](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)) — three weeks after the RC. Install via the standard `typescript` package on npm; no `@rc` tag needed. The team has been working with Bloomberg, Canva, Figma, Google, Lattice, Linear, Miro, Notion, Slack, Vanta, Vercel, and VoidZero on pre-release builds for over a year. **Patch 7.0.2 SHIPPED 2026-07-08T15:55:18Z** ([npm](https://www.npmjs.com/package/typescript)) — supersedes 7.0.0 as the recommended TS 7 version. The `latest` npm dist-tag was bumped to `7.0.2` on release day. 7.0.2 is a pure bug-fix patch (no compiler-API changes, no breaking changes, no docs-only changes); the Go compiler binary itself is unchanged. Install via `npm install -D typescript@^7.0.2` (or just `typescript@latest`). **Recommended TypeScript version: 7.0.2** — supersedes 7.0.0 / 7.0.1. TypeScript 7.1 is in active `dev` builds (`7.1.0-dev.20260723.1` was cut 2026-07-23T08:10:25Z — daily dev cadence; 7.1 is **not** yet recommended). The [Compiler API](https://github.com/microsoft/TypeScript/wiki/API-Breaking-Changes) is **still deferred to 7.1**; if your project uses the TS Compiler API programmatically, stay on the 6.0.x line until 7.1 ships.

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
