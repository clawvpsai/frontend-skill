# Components — React Patterns + shadcn/ui

## The shadcn/ui Mental Model

**shadcn/ui is not a component library — it's a copy-and-paste component source.**

Unlike traditional UI libraries (MUI, Ant Design), shadcn/ui components:
- Live **in your codebase** (`src/components/ui/`)
- You **own and customize** every line
- Add via CLI: `npx shadcn@latest add button`
- No `import { Button } from 'shadcn'` — you import from `@/components/ui/button`

This means zero runtime dependencies for the UI layer, no version lock-in, full customization.

## Component Composition Patterns

### Composition Over Inheritance

```tsx
// ❌ Wrong — prop drilling, tight coupling
function UserCard({ name, email, avatar, role }: UserProps) {
  return <div>{/* deeply nested with all props */}</div>
}

// ✅ Right — compose smaller components
function UserCard({ user }: { user: User }) {
  return (
    <Card>
      <CardHeader>
        <UserAvatar src={user.avatar} alt={user.name} />
        <CardTitle>{user.name}</CardTitle>
        <CardDescription>{user.email}</CardDescription>
      </CardHeader>
      <CardContent>
        <Badge variant={user.role === 'admin' ? 'destructive' : 'secondary'}>
          {user.role}
        </Badge>
      </CardContent>
    </Card>
  )
}
```

### Compound Components

```tsx
// A parent that manages state, children that consume it
function Form({ children, onSubmit }: { onSubmit: (data: unknown) => void }) {
  const [submitting, setSubmitting] = useState(false)
  
  return (
    <form onSubmit={(e) => { setSubmitting(true); onSubmit(e) }}>
      {children}
    </form>
  )
}

// Usage
<Form onSubmit={handleSubmit}>
  <Form.Field name="email">
    <Form.Label>Email</Form.Label>
    <Form.Input type="email" />
    <Form.Error />
  </Form.Field>
</Form>
```

## Server Components vs Client Components

### Server Component (default)

```tsx
// app/users/page.tsx — server component (no directive = server)
import { db } from '@/lib/db'
import { UserCard } from '@/components/user-card'

export default async function UsersPage() {
  const users = await db.user.findMany() // direct DB call, no API needed
  
  return (
    <main className="grid gap-4 grid-cols-1 md:grid-cols-3">
      {users.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </main>
  )
}
```

### Client Component ('use client')

```tsx
// components/like-button.tsx
'use client'

import { useState } from 'react'
import { Button } from '@/components/ui/button'
import { Heart } from 'lucide-react'

interface LikeButtonProps {
  postId: string
  initialCount: number
}

export function LikeButton({ postId, initialCount }: LikeButtonProps) {
  const [liked, setLiked] = useState(false)
  const [count, setCount] = useState(initialCount)
  
  async function toggleLike() {
    setLiked(!liked)
    setCount(c => liked ? c - 1 : c + 1)
    await fetch(`/api/posts/${postId}/like`, { method: 'POST' })
  }
  
  return (
    <Button onClick={toggleLike} variant={liked ? 'destructive' : 'outline'} size="sm">
      <Heart className={liked ? 'fill-current' : ''} className="mr-1 h-4 w-4" />
      {count}
    </Button>
  )
}
```

**Rule:** Keep server components as the default. Add `'use client'` only when you need hooks, browser APIs, or event handlers.

## Component Props Patterns

### Extracting Props from shadcn/ui

```tsx
// Extend shadcn/ui components cleanly
import { Button, type ButtonProps } from '@/components/ui/button'

type PrimaryButtonProps = ButtonProps & {
  loading?: boolean
}

export function PrimaryButton({ loading, children, disabled, ...props }: PrimaryButtonProps) {
  return (
    <Button disabled={loading || disabled} {...props}>
      {loading ? <Spinner className="mr-2 h-4 w-4" /> : null}
      {children}
    </Button>
  )
}
```

### Polymorphic Components

```tsx
import { forwardRef, type ElementType, type ComponentPropsWithRef } from 'react'

type AsProp<C extends ElementType> = {
  as?: C
}

type PropsToOmit<C extends ElementType, P> = P & AsProp<C>

type PolymorphicProps<C extends ElementType, P = object> = 
  PropsToOmit<C, P> & Omit<ComponentPropsWithRef<C>, PropsToOmit<C, P>>

export function Card<C extends ElementType = 'div'>(
  props: PolymorphicProps<C, { className?: string }>
) {
  const { as: Component = 'div', className, ...rest } = props
  return <Component className={cn('rounded-xl border bg-card', className)} {...rest} />
}
```

### React 19 `ref` as a Prop

In React 19, you can pass `ref` as a regular prop — no `forwardRef` needed. This is the preferred pattern for new code:

```tsx
// React 19 — ref as a regular prop (no forwardRef)
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  ref?: React.Ref<HTMLInputElement>
}

function MyInput({ ref, className, ...props }: InputProps) {
  return <input ref={ref} className={cn('border rounded px-3 py-2', className)} {...props} />
}

// Usage with ref
function Form() {
  const inputRef = useRef<HTMLInputElement>(null)
  
  function handleFocus() {
    inputRef.current?.focus()
  }
  
  return (
    <div>
      <MyInput ref={inputRef} placeholder="Type here..." />
      <button onClick={handleFocus}>Focus input</button>
    </div>
  )
}
```

**Why this matters:** `forwardRef` was previously the only way to forward refs. React 19 deprecates `forwardRef` in favor of this native prop pattern. The old `forwardRef` still works, but prefer the new style for new components.

```tsx
// Legacy approach (still works, prefer new style in React 19)
const MyInput = forwardRef<HTMLInputElement, React.InputHTMLAttributes<HTMLInputElement>>(
  ({ className, ...props }, ref) => <input ref={ref} className={className} {...props} />
)
```

## Error Handling in Components

### Error Boundaries (Client Components)

```tsx
// components/error-boundary.tsx
'use client'

import { Component, type ReactNode } from 'react'
import { Button } from '@/components/ui/button'

interface ErrorBoundaryProps {
  children: ReactNode
  fallback: ReactNode
}

interface ErrorBoundaryState {
  hasError: boolean
  error?: Error
}

export class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props)
    this.state = { hasError: false }
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error }
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="flex flex-col items-center gap-4 p-8">
          {this.props.fallback}
          <Button onClick={() => this.setState({ hasError: false })}>
            Try again
          </Button>
        </div>
      )
    }
    return this.props.children
  }
}
```

### Loading States

```tsx
// Always handle loading states — never show blank space
function UserProfile({ userId }: { userId: string }) {
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  })

  if (isLoading) return <Skeleton className="h-32 w-full" />
  if (error) return <p className="text-destructive">Failed to load user</p>
  if (!user) return null

  return <UserCard user={user} />
}
```

## Component Patterns from shadcn/ui

### The `cn()` Utility

Every shadcn/ui project includes this — it merges Tailwind classes intelligently:

```ts
// lib/utils.ts
import { type ClassValue, clsx } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

Usage: `className={cn('base-classes', condition && 'conditional-class')}`

### Form Component Pattern

```tsx
// Using shadcn/ui Form
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { Button } from '@/components/ui/button'
import {
  Form, FormControl, FormField, FormItem, FormLabel, FormMessage,
} from '@/components/ui/form'
import { Input } from '@/components/ui/input'

const formSchema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Min 8 characters'),
})

export function LoginForm() {
  const form = useForm<z.infer<typeof formSchema>>({
    resolver: zodResolver(formSchema),
    defaultValues: { email: '', password: '' },
  })

  async function onSubmit(values: z.infer<typeof formSchema>) {
    await fetch('/api/login', { method: 'POST', body: JSON.stringify(values) })
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl><Input {...field} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit">Submit</Button>
      </form>
    </Form>
  )
}
```

## Performance Rules

- **Colocate components** — keep related components in the same file or directory
- **Lazy load heavy components** — `const Heavy = lazy(() => import('./heavy'))`
- **Memoize expensive computations** — `useMemo(() => expensive(data), [data])`
- **Don't over-abstract** — extract when you have 3+ usages, not 2
- **Keep server components lean** — no business logic, just rendering

## React 19.2 `<Activity>` — Hide UI Without Losing State

`<Activity>` is a new component primitive for hiding part of the UI while preserving its state and DOM. The full API is just two props: `children` and `mode` (`'visible' | 'hidden'`). There is no `detection` prop, no `isActivity` render prop, no `visible={true}` boolean.

```tsx
import { Activity } from 'react'

// Sidebar stays mounted but its Effects are torn down when hidden.
// When toggled back to 'visible', state and scroll position are intact.
function App() {
  const [open, setOpen] = useState(true)
  return (
    <Activity mode={open ? 'visible' : 'hidden'}>
      <Sidebar />
    </Activity>
  )
}
```

**When to use `<Activity>`:** tab/panel/modal/drawer state preservation; pre-rendering heavy content at lower priority. **When NOT to use it:** as a loading-state detector (use `useFormStatus` or `useActionState`'s `isPending`).

See `patterns.md` for the full API, troubleshooting, and production patterns.

**Sources:** [React Activity reference](https://react.dev/reference/react/Activity) · [React 19.2 release notes](https://react.dev/blog/2025/10/01/react-19-2)

## React Compiler — Stop Memoizing by Hand

With `reactCompiler: true` in `next.config.ts` (Next.js 16+), the React Compiler automatically inserts memoization at the expression level. This means **most manual `useMemo`, `useCallback`, and `React.memo` calls become unnecessary**:

```tsx
// ❌ Pre-React-Compiler: manual memoization everywhere
const filteredItems = useMemo(
  () => items.filter(i => i.category === filter),
  [items, filter]
)
const handleClick = useCallback(
  (id: string) => console.log(id),
  []
)
const MemoCard = React.memo(Card)

// ✅ With React Compiler enabled: plain code, identical perf
const filteredItems = items.filter(i => i.category === filter)
const handleClick = (id: string) => console.log(id)
function Card({ item }: { item: Item }) { /* ... */ }
```

The compiler memoizes at a finer granularity than humans do (it can memoize across early returns and conditional branches) and never gets a dependency array wrong.

**Still use `useMemo` / `useCallback` when:**
- **Crossing component boundaries the compiler can't see** — e.g. functions passed into third-party libraries that aren't compiled (e.g. `chart.setOptions(opts)` in an Effect)
- **Referential identity matters for non-React APIs** — WebSocket subscriptions, event emitter listeners, imperative DOM libraries
- **The component is opted out of compilation** — file-level `'use no memo'` directive, or the compiler emits a "skipped" warning for that component

**Don't use them when:**
- "Just in case" — the compiler will handle it
- Stable callbacks for memoized children — the compiler memoizes the children too
- Inside a fully compiled tree — let the compiler decide

**Sources:** [React Compiler docs](https://react.dev/reference/react-compiler) · [Why React 19's Compiler Changes Everything for Senior Devs (SitePoint)](https://www.sitepoint.com/why-react-19-s-compiler-changes-everything-for-senior-devs/)

## shadcn/ui CLI v4 (March 2026)

shadcn CLI v4 is built for coding agents and includes several new commands worth knowing about.

```bash
# Add a component with diff preview (good for upgrades)
npx shadcn@latest add button --diff

# Get a snapshot of your project (framework, version, installed components) — great for AI agents
npx shadcn@latest info

# Dry-run a registry install (see what would change without writing files)
npx shadcn@latest add dialog --dry-run

# Use a preset when scaffolding
npx shadcn@latest init --preset adtk27v

# Install shadcn/ui skills for your coding agent (one command, knows the registry + CLI)
npx skills add shadcn/ui
```

**New in CLI v4:**
- **shadcn/skills** — drop-in agent skill that teaches coding agents how to use the shadcn CLI and registry correctly
- **Presets** — scaffold a project or switch design systems with `--preset <id>`
- **Dry-run mode** — preview what `add` would do
- **`--diff` flag** — see changes before merging registry updates
- **`shadcn info`** — full project context for agents
- **Package imports** (shadcn@4.7.0+) — use `package.json` `#...` aliases in `components.json` instead of `tsconfig.json` `compilerOptions.paths`
- **Registry types**: `registry:base` ships an entire design system as a single payload; `registry:font` is now a first-class type
- **`shadcn eject`** (May 2026) — pulls the registry's CSS into your codebase so you can own it fully

**Sources:** [shadcn CLI v4 changelog (March 2026)](https://ui.shadcn.com/docs/changelog/2026-03-cli-v4) · [shadcn changelog](https://ui.shadcn.com/docs/changelog) · [shadcn package imports (May 2026)](https://ui.shadcn.com/docs/changelog/2026-05-package-imports-target-aliases)
