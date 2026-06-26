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

### Ref callback cleanup (React 19)

In React 19, a **callback ref** can return a cleanup function. React calls it when the ref is detached (component unmounts) — and the ref is *not* called with `null` anymore. This is the cleanest way to attach and tear down observers, subscriptions, or imperative DOM APIs without `useEffect`:

```tsx
// React 19 — ref callback returns a cleanup function
'use client'

import { useState } from 'react'

function MeasureExample() {
  const [height, setHeight] = useState(0)

  // ✅ The returned function is the cleanup — no useEffect needed
  const measuredRef = (node: HTMLHeadingElement | null) => {
    if (!node) return  // safety for the null case

    const observer = new ResizeObserver(([entry]) => {
      setHeight(entry.contentRect.height)
    })
    observer.observe(node)

    // Return cleanup — React calls it when the node is detached
    return () => {
      observer.disconnect()
    }
  }

  return (
    <>
      <h1 ref={measuredRef}>Hello, world</h1>
      <h2>The above header is {Math.round(height)}px tall</h2>
    </>
  )
}
```

**What changed vs React 18:**
- React 18: callback ref was called with `null` on unmount; you had to detect that to clean up.
- React 19: return a cleanup function — React calls it on detach, and skips the `null` call. Cleaner mental model, fewer re-runs.

**Caveats:**
- The returned cleanup runs on unmount, not on every re-render. The callback itself is still re-invoked on every render unless you wrap it in `useCallback` or enable the **React Compiler** (then the compiler memoizes it for you).
- Pair with the React Compiler: it removes the need for `useCallback` around the ref callback, so the cleanup pattern is fully self-contained.
- Cleanup return is not yet supported in async Server Components.

**Sources:**
- [React 19 release notes — ref cleanup](https://react.dev/blog/2024/12/05/react-19)
- [tkdodo — Ref Callbacks, React 19 and the Compiler](https://tkdodo.eu/blog/ref-callbacks-react-19-and-the-compiler)
- [React docs — `useRef` (callback ref API)](https://react.dev/reference/react/useRef#caveats)

## Custom Hooks (React 19 + 19.2)

React 19 (and 19.2) shipped several new hooks and stabilized others. These cover the patterns the React docs now recommend for forms, accessibility, and event separation.

### `useFormStatus` — Form pending state from a child

`useFormStatus()` reads the status of the parent `<form>`. It works with **Server Actions** and native form submissions — no extra wiring. The hook must be called from a component rendered *inside* the `<form>` (not the form component itself):

```tsx
// components/submit-button.tsx
'use client'

import { useFormStatus } from 'react-dom'
import { Button } from '@/components/ui/button'

export function SubmitButton({ children }: { children: React.ReactNode }) {
  const { pending, data, method, action } = useFormStatus()

  return (
    <Button type="submit" disabled={pending} aria-busy={pending}>
      {pending ? 'Submitting…' : children}
    </Button>
  )
}
```

**Returns:**
- `pending: boolean` — true while the form is submitting
- `data: FormData | null` — the data being submitted
- `method: 'get' | 'post' | 'dialog' | null`
- `action: string | null` — the resolved action URL

**Typical usage in a Server Action form:**

```tsx
// app/signup/page.tsx — server component
import { signup } from './actions'
import { SubmitButton } from '@/components/submit-button'

export default function SignupPage() {
  return (
    <form action={signup} className="space-y-4">
      <input name="email" type="email" required className="border rounded px-3 py-2" />
      <SubmitButton>Create account</SubmitButton>
    </form>
  )
}
```

**Caveats:**
- Must be inside a `<form>`. If `status.pending` is never `true`, the hook is being called from a component that isn't a child of the form (common bug).
- For React Hook Form, prefer `formState.isSubmitting` instead — `useFormStatus` only sees the parent native `<form>` it lives inside, not RHF-managed state.

**Sources:**
- [React docs — `useFormStatus`](https://react.dev/reference/react-dom/hooks/useFormStatus)
- [Telerik — Guide to New Hooks in React 19](https://www.telerik.com/blogs/guide-new-hooks-react-19)

### `useId` — SSR-safe, accessibility-grade unique IDs

`useId()` generates a unique, stable ID per component instance that matches between server and client. Use it for `<label htmlFor>`, `aria-describedby`, `aria-labelledby`, and any other DOM-ID relationship:

```tsx
'use client'

import { useId } from 'react'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'

export function TextField({ label, error, hint, ...props }: TextFieldProps) {
  // One call → derive every related ID deterministically
  const id = useId()
  const errorId = `${id}-error`
  const hintId = `${id}-hint`

  return (
    <div className="space-y-1">
      <Label htmlFor={id}>{label}</Label>
      <Input
        id={id}
        aria-describedby={[hintId, error ? errorId : null].filter(Boolean).join(' ') || undefined}
        aria-invalid={error ? true : undefined}
        {...props}
      />
      {hint ? <p id={hintId} className="text-sm text-muted-foreground">{hint}</p> : null}
      {error ? <p id={errorId} className="text-sm text-destructive">{error}</p> : null}
    </div>
  )
}
```

**Why `useId` (and not `Math.random()` or a global counter):**
- Stable across re-renders — screen readers don't get confused by changing IDs.
- Same value on server and client — no React hydration mismatch warnings.
- No shared mutable counter — safe inside lists, modals, portals, anywhere.

**Caveats (from the React docs):**
- **Don't use for React keys** in lists. Keys must come from your data, not from `useId`.
- **Don't use for `use()` cache keys** — IDs aren't stable enough as cache identifiers.
- **Not yet supported in async Server Components** — call it from a client child or hoist into a client component.
- The component tree rendered on server and client must be structurally identical, or the generated IDs won't match.

**Sources:**
- [React docs — `useId`](https://react.dev/reference/react/useId)
- [Epic React — Improving React Accessibility with `useId`](https://www.epicreact.dev/improving-react-accessibility-with-use-id-knljs)

### `useEffectEvent` (React 19.2) — Separate events from effects

`useEffectEvent` lets you extract a callback that reads the **latest** props/state without making those values reactive in your effect's dependency array. It's React's official answer to the "stale closure + lint suppression" problem:

```tsx
'use client'

import { useEffect, useContext, useEffectEvent } from 'react'
import { ShoppingCartContext } from './cart-context'

function Page({ url }: { url: string }) {
  const { items } = useContext(ShoppingCartContext)
  const numberOfItems = items.length

  // ✅ Effect event — reads latest `numberOfItems`, but does NOT re-run the effect when it changes
  const onNavigate = useEffectEvent((visitedUrl: string) => {
    logVisit(visitedUrl, numberOfItems)  // always the latest value
  })

  // ✅ Effect only re-runs when `url` changes
  useEffect(() => {
    onNavigate(url)
  }, [url])  // eslint react-hooks/exhaustive-deps won't complain about onNavigate
}
```

**The problem it solves:** Normally, if an effect reads a reactive value (a prop or state), that value must be in the dependency array — otherwise the effect sees a stale value. But adding it causes the effect to re-run unnecessarily. `useEffectEvent` lets you "mark" a function as non-reactive: it always sees the latest state but doesn't trigger the effect.

**More real-world examples:**

```tsx
// Auto-save on interval — useEffectEvent lets the save callback see fresh content
function DocumentEditor({ documentId }: { documentId: string }) {
  const [content, setContent] = useState('')
  const [meta, setMeta] = useState({})

  const doSave = useEffectEvent(() => {
    api.save(documentId, { content, meta, timestamp: Date.now() })
  })

  useEffect(() => {
    const id = setInterval(doSave, 30_000)
    return () => clearInterval(id)
  }, [documentId])  // doesn't re-run on every keystroke
}

// WebSocket notification — read latest `theme` without re-subscribing on theme change
function ChatRoom({ roomId, theme }: Props) {
  const onConnected = useEffectEvent(() => {
    showNotification('Connected!', theme)  // latest theme, no re-subscribe
  })

  useEffect(() => {
    const connection = createConnection(serverUrl, roomId)
    connection.on('connected', onConnected)
    connection.connect()
    return () => connection.disconnect()
  }, [roomId])  // re-subscribes only on roomId change
}
```

**Caveats:**
- `useEffectEvent` functions are meant to be called **inside Effects** (`useEffect`, `useLayoutEffect`, `useInsertionEffect`). They have an "Effect Event" identity, so they won't trigger re-renders and aren't reactive.
- Don't add the effect event to your effect's dependency array — the exhaustive-deps lint rule recognizes them and exempts them automatically.
- Pair with the React Compiler: it removes the manual `useCallback` you'd otherwise use, leaving you with a clean effect + event split.

**Sources:**
- [React docs — `useEffectEvent`](https://react.dev/reference/react/useEffectEvent)
- [React 19.2 release notes](https://react.dev/blog/2025/10/01/react-19-2)
- [React docs — Separating events from effects](https://react.dev/learn/separating-events-from-effects)

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
- **GitHub Registries** (shadcn@4.10+, June 2026) — any public GitHub repo with a `registry.json` at the root is now installable directly. No registry server, no JSON publishing step.

**Sources:** [shadcn CLI v4 changelog (March 2026)](https://ui.shadcn.com/docs/changelog/2026-03-cli-v4) · [shadcn changelog](https://ui.shadcn.com/docs/changelog) · [shadcn package imports (May 2026)](https://ui.shadcn.com/docs/changelog/2026-05-package-imports-target-aliases)

## shadcn eject (May 2026) — Take Ownership of the Registry CSS

When you run `npx shadcn@latest init`, the CLI adds `@import "shadcn/tailwind.css"` to your global stylesheet. This import provides shared Tailwind v4 utilities that both Radix and Base UI shadcn variants depend on — custom variants like `data-open:` and `data-closed:`, utilities like `no-scrollbar`, and accordion animations.

The `shadcn eject` command **inlines `shadcn/tailwind.css` into your global CSS file and removes the `shadcn` package from your `package.json` dependencies**. After ejecting, you own the CSS verbatim — future shadcn CLI updates to `shadcn/tailwind.css` will NOT apply automatically.

```bash
# Eject the shadcn/tailwind.css in your project
npx shadcn@latest eject

# In a monorepo — point at the workspace that contains components.json
npx shadcn@latest eject -c packages/ui

# Skip the confirmation prompt
npx shadcn@latest eject -y

# Mute output (useful in CI)
npx shadcn@latest eject -s
```

**Options:**

| Flag | Description |
|---|---|
| `-c, --cwd <cwd>` | Working directory (defaults to current). Use in monorepos. |
| `-y, --yes` | Skip confirmation prompt. |
| `-s, --silent` | Mute output. |
| `-h, --help` | Display help. |

**Before eject:**

```css
@import "tailwindcss";
@import "tw-animate-css";
@import "shadcn/tailwind.css";
```

**After eject:**

```css
@import "tailwindcss";
@import "tw-animate-css";
/* ejected from shadcn@4.8.3 */
@theme inline {
  @keyframes accordion-down {
    from { height: 0; }
    to {
      height: var(
        --radix-accordion-content-height,
        var(--accordion-panel-height, auto)
      );
    }
  }
}
@custom-variant data-open {
  &:where([data-state="open"]),
  &:where([data-open]:not([data-open="false"])) {
    @slot;
  }
}
@utility no-scrollbar {
  -ms-overflow-style: none;
  scrollbar-width: none;
  &::-webkit-scrollbar { display: none; }
}
```

> ⚠️ **This action is irreversible.** After ejecting, you will not get shadcn CLI improvements to the shared CSS automatically — you will have to either manually merge them in or re-init from scratch. Only eject if you are certain you want to fully own the CSS and have no plans to keep up with shadcn updates.

**When to eject:**
- You want zero runtime dependencies on shadcn internals
- You have internal CSS conventions that conflict with shadcn's shared utilities
- You are forking shadcn and need the source visible in your repo
- You are shipping to air-gapped / locked-down environments where pulling from npm at build time is restricted

**When NOT to eject:**
- You want to keep getting the latest shadcn improvements
- You use the standard Radix or Base UI shadcn variants (the shared CSS is what they need)

**Source:** [shadcn eject — May 2026 changelog](https://ui.shadcn.com/docs/changelog) · [shadcn CLI docs — eject command](https://ui.shadcn.com/docs/cli#eject)
## shadcn 4.11.1 — `package.json` Specifier Preservation Fix (June 26, 2026)

A real user-facing bug fix shipped in 4.11.1. If you use `shadcn add` to install registry items in a project with a curated `package.json` (pinned ranges, custom registries, monorepo workspace overrides), upgrade before your next `shadcn add`.

### The bug (now fixed) — `shadcn add` silently corrupted `package.json` specifiers

Tracked as [shadcn-ui/ui#10525](https://github.com/shadcn-ui/ui/issues/10525). When `shadcn add` ran against a `package.json` that already had dependencies, **specifier values got swapped between packages**. The CLI would write the new dep at the wrong key, leaving the lockfile and `package.json` in contradictory states. Repro (deterministic):

```json

// BEFORE shadcn add

"dependencies": {

  "@base-ui/react": "latest",

  "class-variance-authority": "^0.7.1"

}



// AFTER shadcn add (BROKEN pre-4.11.1)

"dependencies": {

  "@base-ui/react": "latest",

  "class-variance-authority": "^1.4.1"   // value that belonged to a different package

}

```

`pnpm-lock.yaml` would then show the correct resolved version but the wrong specifier — `pnpm install` against the corrupted file would silently upgrade or downgrade packages on next install.

### The fix — `shadcn add` now preserves existing specifiers

[PR #10967](https://github.com/shadcn-ui/ui/pull/10967) (shipped in 4.11.1) replaces the specifier-merge path so existing entries are read from `package.json` first, only the new dep is written, and no specifier shuffling happens. If you were bitten by this, diff `package.json` and your lockfile — the corrupted specifier is in `package.json` only, the lockfile holds the real resolved version. The fix is to:

1. Upgrade to `shadcn@4.11.1` or later (`npx shadcn@latest add ...`).

2. Manually restore the correct specifier values from the lockfile (`pnpm-lock.yaml` has the real resolved versions).

3. Re-run `pnpm install --lockfile-only` to verify the specifier/version pair reconciles cleanly.

### Also in 4.11.1 — `node-fetch` → native `fetch`

[PR #10905](https://github.com/shadcn-ui/ui/pull/10905) drops the `node-fetch` transitive dep in favor of the Node 18+ native `fetch`. No user-facing change, just a smaller install footprint and one fewer transitive supply-chain surface. Requires Node ≥ 18 (already a hard requirement for every tracked Next.js / Vite / TanStack project).

**Sources:** [shadcn-ui/ui#10525 — bug report](https://github.com/shadcn-ui/ui/issues/10525) · [shadcn-ui/ui#10967 — specifier preservation fix](https://github.com/shadcn-ui/ui/pull/10967) · [shadcn-ui/ui#10905 — node-fetch → native fetch](https://github.com/shadcn-ui/ui/pull/10905) · [shadcn changelog](https://ui.shadcn.com/docs/changelog)



## shadcn 4.12.0 — Chat Components + `scroll-fade`/`shimmer` Utilities + New `@shadcn/react` Package (June 26, 2026)

The first minor bump in 18 days (since 4.11.0 on June 8) and one of the largest single-PR feature drops in shadcn/ui history. PR [#11022](https://github.com/shadcn-ui/ui/pull/11022) "feat: @shadcn/react" landed **59,716 additions, 1,547 deletions, 569 files** — five new chat components, two new CSS utilities, and a brand-new npm package. If you build any kind of chat UI (AI assistants, support inboxes, team threads, group chats), this is the most material shadcn release since the CLI rewrite.

### The five new Chat Components

Announced as the "first phase of the chat components work" on the official [June 2026 — Components for Chat Interfaces](https://ui.shadcn.com/docs/changelog/2026-06-chat-components) post. All five ship as registry items you copy with the CLI, and each has a **Base UI flavor** (`base-luma`, `base-lyra`, `base-maia`, `base-mira`, `base-nova`, `base-rhea`, `base-sera`, `base-vega`) and a **Radix UI flavor** (`radix-luma`, `radix-lyra`, etc.) — choose the underlying primitive library per style.

```bash
npx shadcn@latest add message-scroller message bubble attachment marker
```

| Component | Role | Notes |
| --- | --- | --- |
| **`MessageScroller`** | Scroll container for a conversation. Owns anchored turns, streamed replies, saved-thread restore, prepended history, jump-to-message, scroll controls, and visibility tracking. **Does not own** messages, AI state, transport, persistence, or model state — you bring the content renderer. | The most-tested primitive in this batch: 1,701 lines of unit tests + 698 lines of browser tests + 841 lines of perf tests in `@shadcn/react`. The dev-mode story is anchoring, auto-follow, prepend preservation, scroll commands, and visibility tracking. |
| **`Message`** | A row in the conversation — avatar, alignment, header, content, footer, grouped messages. | Render-prop surface; bring your own content renderer. |
| **`Bubble`** | Message surface — variants, alignment, reactions, links, buttons, collapsible content. | Built on top of `Message`. Multiple example variants (alignment, collapsible, link-button, markdown, popover, reactions, tooltip, variants). |
| **`Attachment`** | Files and images — media, metadata, upload state, actions, full-card trigger that keeps actions separately clickable. | Group/image/sizes/states/trigger examples included. |
| **`Marker`** | Status updates, system notes, bordered rows, labeled separators — streaming state, tool activity, date breaks. | Border/icon/link-button/separator/shimmer/status/variants examples. |

The pieces compose: `MessageScroller` > `Message` > `Bubble` for the message surface; `Attachment` slots into `Message`; `Marker` slots into `MessageScroller.Content` between `Message.Item`s.

**Why this is a first-class release, not a "labs" preview:** these components are production-grade on day one — the registry items install into your codebase (same model as every other shadcn component), and the `@shadcn/react/message-scroller` primitive ships with a unit test suite, a Chromium browser test suite, and a perf test suite in CI. You do not adopt a "beta" namespace.

### The two new CSS utilities — `scroll-fade` and `shimmer`

Both utilities ship inside **`packages/shadcn/src/tailwind.css`** (the shared CSS bundle that `npx shadcn@latest init` writes into your project), so projects already initialized with the latest CLI get them for free. They use Tailwind v4 `@theme` directives and CSS custom properties — no runtime JS.

**`scroll-fade`** — adds scroll-aware edge fades to scroll containers. Use it on `MessageScroller`, `ScrollArea`, attachment rows, any long list where you want to hint at more content without adding overlays or scroll listeners.

```tsx
// Add the class to a scroll container — top + bottom edges fade when content is scrollable past them
<div className="scroll-fade scroll-fade-edge-both h-96 overflow-y-auto">
  {messages.map((m) => (
    <Message key={m.id} {...m} />
  ))}
</div>
```

Variants in the docs: `scroll-fade-edge-both`, `scroll-fade-edge-top`, `scroll-fade-edge-bottom`, `scroll-fade-edge-none`, `scroll-fade-horizontal`, `scroll-fade-rtl`, `scroll-fade-overflow`, `scroll-fade-size`. Pure CSS — no IntersectionObserver, no JS-driven opacity math.

**`shimmer`** — animated text shimmer for live status. Use it for "Thinking…", "Generating response…", running tools, streaming markers.

```tsx
<span className="shimmer shimmer-angle-90 shimmer-duration-1.5">
  Generating response…
</span>
```

Variants: `shimmer-angle-{0,45,90,135}`, `shimmer-color-{primary,muted,accent}`, `shimmer-duration-{0.5,1,1.5,2,3}`, `shimmer-marker`, `shimmer-none`, `shimmer-once`, `shimmer-rtl`, `shimmer-spread-{tight,normal,wide}`. Plays once on mount or loops, depending on `shimmer-once`. RTL is a first-class variant.

Both utilities are also exposed via the registry:

```bash
npx shadcn@latest add scroll-fade shimmer
```

### The new `@shadcn/react` npm package — headless React primitives

A brand-new npm package: **`@shadcn/react`** (currently `0.1.0` on the registry; the source `packages/react/package.json` is at `0.2.0` ahead of the next publish). Description: **"Unstyled components for React."** Keywords: `react, headless, unstyled, primitives, accessible, ui, shadcn`. License: MIT. React peer dep: `>=19`.

**The split that this enables:**

| What you want | Use |
| --- | --- |
| Styled copy-paste component in your codebase | `npx shadcn@latest add message-scroller` (registry item, fully styled) |
| Unstyled headless primitive + your own styles | `npm install @shadcn/react` + import `@shadcn/react/message-scroller` |

The first primitive is **`@shadcn/react/message-scroller`**. The registry component is a styled wrapper around the package — so you get the tested interaction logic from one source, and pick the visual style (Base UI or Radix) on top.

**API surface (`@shadcn/react/message-scroller`)** — compound component with hooks:

```tsx
import {
  MessageScroller,
  useMessageScroller,
  useMessageScrollerScrollable,
  useMessageScrollerVisibility,
} from "@shadcn/react/message-scroller"

;<MessageScroller.Provider autoScroll>
  <MessageScroller.Root>
    <MessageScroller.Viewport preserveScrollOnPrepend>
      <MessageScroller.Content>
        <MessageScroller.Item messageId="m1" scrollAnchor>
          {/* your message content */}
        </MessageScroller.Item>
      </MessageScroller.Content>
    </MessageScroller.Viewport>
    <MessageScroller.Button direction="end" />
  </MessageScroller.Root>
</MessageScroller.Provider>
```

**Parts** (props noted):
- `MessageScroller.Provider` — owns scroll state, anchoring, auto-follow, visibility. Props: `autoScroll`, `defaultScrollPosition` (`"start" | "end" | "last-anchor"`), `scrollPreviousItemPeek`, `scrollMargin`, `scrollEdgeThreshold`.
- `MessageScroller.Root` — styled frame.
- `MessageScroller.Viewport` — scrollable frame; prop `preserveScrollOnPrepend` (do not jump when history is loaded above).
- `MessageScroller.Content` — message list; defaults `role="log"` + `aria-relevant="additions"`.
- `MessageScroller.Item` — one message wrapper; props `messageId`, `scrollAnchor`.
- `MessageScroller.Button` — scroll-to-end/start affordance; auto-hides when caught up; prop `direction`.

**Hooks (flat siblings, not context consumers — call from anywhere in the tree):**
- `useMessageScroller()` → `{ scrollToMessage, scrollToStart, scrollToEnd }`
- `useMessageScrollerScrollable()` → `{ start, end }` (edges the viewport can scroll toward)
- `useMessageScrollerVisibility()` → `{ currentAnchorId, visibleMessageIds }`

**Types** — `MessageScrollerScrollOptions` (with `align: "start" | "center" | "end" | "nearest"`, `behavior`, `scrollMargin`), `MessageScrollerScrollable`, `MessageScrollerVisibilityState`.

**Why a separate npm package and not just another registry item?** Per the official post: *"This lets us ship behavior without locking it to a visual style. You still get copy-and-paste components that match your project, and the hard interaction logic stays tested in one place."* The package owns the interaction primitives (geometry math, scroll commands, visibility tracking, anchoring); the registry item owns the visual style. Consumers who want their own brand can pull `@shadcn/react/message-scroller` and wrap it in their own components without forking the tested behavior.

The package also includes a `useRender` hook — *"a poor man's version of base-ui `useRender`"* per the source comment — that will eventually be replaced by the upstream `base-ui` package when it ships. Currently shipped from `packages/react/src/use-render/`.

**Test surface in the package itself** (`pnpm test` / `pnpm test:browser`):
- `geometry.test.ts` — jsdom + stubbed rects, geometry math
- `message-scroller.browser.test.tsx` — chromium, native scroll anchoring / prepend / visibility
- `message-scroller.perf.browser.test.tsx` — chromium, performance benchmark + regression guard

**Caveats worth knowing:**
1. The release notes body is unusually terse (1 entry: *"add scroll-fade and shimmer utilities"*) — the real content is the PR diff and the changelog post, not the GitHub release body. Don't read the GitHub release body as the full changelog for this release.
2. `@shadcn/react@0.1.0` is on npm but the source is at `0.2.0` — a publish lag, not a version mismatch to worry about; the next release will reconcile.
3. The Base UI and Radix UI flavors are separate registry items — install the one your project uses; mixing both increases bundle size for no functional gain.
4. The two CSS utilities are included in the shared `tailwind.css` for projects initialized with the latest CLI. If you have an older init, re-run `npx shadcn@latest init` (or copy the `scroll-fade-*` and `shimmer-*` rules from `packages/shadcn/src/tailwind.css` into your own CSS).

**Source:** [shadcn changelog](https://ui.shadcn.com/docs/changelog) · [shadcn-ui/ui#11022 — feat: @shadcn/react](https://github.com/shadcn-ui/ui/pull/11022) · [June 2026 — Components for Chat Interfaces blog post](https://ui.shadcn.com/docs/changelog/2026-06-chat-components) · [@shadcn/react on npm](https://www.npmjs.com/package/@shadcn/react) · [@shadcn/react README](https://github.com/shadcn-ui/ui/tree/main/packages/react) · [MessageScroller package README](https://github.com/shadcn-ui/ui/tree/main/packages/react/src/message-scroller) · [shadcn-ui/ui commit `18fcf0f7` — feat: @shadcn/react](https://github.com/shadcn-ui/ui/commit/18fcf0f766857a7249cc0daac3c1609610edd158) · [shadcn-ui/ui commit `8055a12f` — chore(release): version packages](https://github.com/shadcn-ui/ui/commit/8055a12f)



## shadcn/ui GitHub Registries (June 2026)

As of shadcn CLI v4.10 (June 8, 2026), **any public GitHub repository can be a registry**. Add a `registry.json` file at the root describing what to distribute, commit, and users install items directly from GitHub — no server, no build step, no manual JSON publishing.

This is huge for teams that want to share design systems, conventions, agents skills, or internal component libraries without running a registry server.

### Installing From a GitHub Registry

```bash
# Install a single item from a GitHub registry
npx shadcn@latest add acme/toolkit/project-conventions

# View an item before installing
npx shadcn@latest view acme/toolkit/project-conventions

# Install an item whose name contains slashes
npx shadcn@latest add acme/toolkit/rules/agent

# Pin to a tag or commit SHA
npx shadcn@latest add acme/toolkit/project-conventions.0.0
```

The first two path segments are `<github-owner>/<github-repo>`. Any remaining segments are the registry item name (not a file path). Addresses ending in `.json` are treated as file paths.

### Creating Your Own GitHub Registry

**Step 1: Add `registry.json` at the repo root.**

```json
{
  "$schema": "https://ui.shadcn.com/schema/registry.json",
  "name": "acme",
  "homepage": "https://acme.com",
  "items": [
    {
      "name": "project-conventions",
      "type": "registry:file",
      "title": "Acme Project Conventions",
      "description": "Coding conventions, AGENTS.md, and PR templates for Acme projects.",
      "files": [
        {
          "path": "AGENTS.md",
          "type": "registry:file"
        },
        {
          "path": "docs/conventions.md",
          "type": "registry:file"
        }
      ]
    },
    {
      "name": "button",
      "type": "registry:ui",
      "title": "Acme Button",
      "description": "Acme-flavored button component",
      "files": [
        {
          "path": "components/ui/button.tsx",
          "type": "registry:ui"
        }
      ]
    }
  ]
}
```

**Step 2: Compose large registries with `include`.**

You can split a registry across multiple `registry.json` files (folder shorthand is NOT supported — must be explicit file paths):

```json
{
  "$schema": "https://ui.shadcn.com/schema/registry.json",
  "name": "acme",
  "homepage": "https://acme.com",
  "include": [
    "components/ui/registry.json",
    "hooks/registry.json",
    "skills/registry.json"
  ]
}
```

Included `registry.json` files may omit `name` and `homepage` (only required on the root).

**Step 3: Commit and push.**

```bash
git add registry.json
git commit -m "add registry"
git push
```

**Step 4: Users install from your repo.**

```bash
npx shadcn@latest add acme/toolkit/project-conventions
```

### Use Cases

- **Internal design systems** — share your button, dialog, table components across all your projects without running a CDN
- **Agent skills / AGENTS.md** — distribute `AGENTS.md`, `CLAUDE.md`, `.cursorrules` to every project from one GitHub repo
- **Coding conventions** — share `docs/conventions.md`, ESLint configs, Biome configs
- **Block libraries** — pre-built sections (hero, pricing, footer) installable as a unit
- **Framework adapters** — publish once, install anywhere

### Registry Item Types

| Type | Purpose |
|---|---|
| `registry:ui` | Single component (e.g. button, dialog) — installs to `components/ui/` |
| `registry:component` | Composite component, may include multiple files |
| `registry:block` | Multi-file block (e.g. hero section with multiple components) |
| `registry:file` | Raw file — AGENTS.md, configs, docs, anything |
| `registry:base` | Full design system as a single payload (colors, fonts, utilities, components) |
| `registry:font` | Font installation payload (first-class in v4) |
| `registry:hook` | Custom React hooks |
| `registry:theme` | Theme tokens (CSS variables) |
| `registry:style` | Global styles |
| `registry:lib` | Utility library file |

**Sources:** [shadcn GitHub Registries changelog (June 2026)](https://ui.shadcn.com/docs/changelog/2026-06-github-registries) · [shadcn GitHub Registry docs](https://ui.shadcn.com/docs/registry/github) · [shadcn registry.json schema](https://ui.shadcn.com/docs/registry/registry-json)

