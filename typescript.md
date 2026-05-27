# TypeScript — Strict Patterns, Generics, Utilities

## Strict Configuration

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

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

## Common Mistakes

- **`any` type** — use `unknown` instead when the type is truly unknown, then narrow
- **Type assertions (`as`)** — avoid them; use type guards instead
- **Over-engineering generics** — simple types are better until complexity demands generics
- **Not using `z.infer<>`** — define types with Zod, not twice
- **`noUncheckedIndexedAccess` off** — turn it on, handle undefined
- **`object` type** — use `Record<string, unknown>` or specific shapes instead
