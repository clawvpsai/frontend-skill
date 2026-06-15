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
    "target": "ES2026",         // TS 6.0 defaults to ES2026 (was ES2022)
    "lib": ["ES2026", "DOM", "DOM.Iterable"],
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


### ES2026 Built-in Types (TypeScript 6.0)

TypeScript 6.0 includes types for ES2026 built-in methods:

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

These are enabled by setting `target: "ES2026"` (the TS 6.0 default) or adding `"lib": ["ES2026"]`.

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

TypeScript 7.0 Beta arrived April 2026 as the first release built on a **Go-based compiler** (`tsgo`), replacing the original TypeScript compiler written in TypeScript. The stable release targets mid-2026. Key motivation: 10x faster type-checking through native code and shared-memory parallelism.

**Current status:** Beta — not yet stable. Test in a branch before adopting in production.

### The Go Compiler (`tsgo`)

The new compiler binary is `tsgo` (vs the old `tsc`). It is shipped as the official `tsc` replacement when TypeScript 7 goes stable. Key architectural changes:

- **No V8 overhead** — runs as native code, not on the JavaScript engine
- **Automatic parallelization** — parses, type-checks, and emits across files simultaneously
- **Same API** — `tsgo` accepts the same CLI flags as `tsc`; most projects migrate without code changes
- **Same `tsconfig.json`** — no new config format; just swap the binary

```bash
# Install TS 7 beta
npm install -D typescript@beta

# Check version — should show 7.x with "G" indicator
npx tsc --version
# → 7.0.0-beta

# Run type-checking (no build output)
npx tsc --noEmit
```

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
grep -r '"moduleResolution": "Classic"' tsconfig*.json
# → Update to "Bundler" or "Node16"

# 5. Audit @types dependencies (ensure explicit)
npm ls @types/node @types/react  # check what's installed
# → Add any missing @types packages explicitly

# 6. Test with beta
npm install -D typescript@beta
npx tsc --noEmit
```

### What Stays the Same

The following continue to work as before — no migration needed:

- **`tsconfig.json` structure** — same format, same options (except removed ones above)
- **JavaScript/JSX support** — `jsx: "react-jsx"` still works
- **Path aliases** — `paths` in tsconfig unchanged
- **All existing type syntax** — generics, utility types, conditional types all work
- **`skipLibCheck`** — still recommended
- **`noUncheckedIndexedAccess`** — still available
- **IDE support** — VS Code, JetBrains IDEs support TS 7 beta via extensions

**Sources:**
- [TypeScript 7.0 Beta — Go-based foundation](https://visualstudiomagazine.com/articles/2026/04/21/typescript-7-0-beta-arrives-on-go-based-foundation-with-10x-speed-claim.aspx)
- [TypeScript 7 Progress — December 2025](https://devnewsletter.com/p/state-of-typescript-2026/)
- [TypeScript 7 Migration Guide](https://codingdunia.com/blog/typescript-7-migration-guide/)
## Common Mistakes

- **`any` type** — use `unknown` instead when the type is truly unknown, then narrow
- **Type assertions (`as`)** — avoid them; use type guards instead
- **Over-engineering generics** — simple types are better until complexity demands generics
- **Not using `z.infer<>`** — define types with Zod, not twice
- **`noUncheckedIndexedAccess` off** — turn it on, handle undefined
- **`object` type** — use `Record<string, unknown>` or specific shapes instead
- **Legacy TypeScript targets** — `target: "ES3"` or `"ES5"` is deprecated in 6.0; use ES2025 minimum
- **Missing `stableTypeOrdering`** — set it now to prepare for TS 7.0
- **`import defer` with named export in JSX** — use `import defer * as mod` then destructure: `const { Foo } = await mod`
- **`import defer` in Server Components for data** — `import defer` is a bundle-splitting tool, not a data-fetching tool; use React's `use()` hook with Promises for streaming server-side data
