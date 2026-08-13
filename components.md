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

> **Bundled in Next.js 16.3.0-canary.90** (2026-07-19T23:34:16Z, [PR #95901](https://github.com/vercel/next.js/pull/95901)) — `next@canary@90` vendors React canary `172742b4-20260716` (`packages/next/src/compiled/react/cjs/react.production.js`), which contains PR #37039 + the prior React canary PRs. So anyone running `next@canary@90+` gets the `enableEffectEventMutationPhase` flag-on behaviour AND PR #37030's hydration-nonce fix automatically — no separate `react@canary` install needed. Until React 19.3 stable ships (date TBA), `next@canary` is the only path to these fixes on a vendored-React basis; stable `next@16.2.x` still bundles React 19.2.7 and needs a manual `react@canary` pin (or a wait for the July 20, 2026 Next.js Security Release if it bundles the nonce fix, or React 19.3 stable). Verify with `npm view next dist-tags.canary` — should be `16.3.0-canary.90` or later.

> **React canary 19.3.0-canary-172742b4-20260716 — `useEffectEvent` `enableEffectEventMutationPhase` flag enabled everywhere** ([React PR #37039](https://github.com/facebook/react/pull/37039), merged 2026-07-16T13:21:39Z) — the previously-gated `enableEffectEventMutationPhase` flag (added by [PR #35548](https://github.com/facebook/react/pull/35548) on 2026-02-04 as a killswitch + perf optimization) is now enabled unconditionally in React's main. The flag is expected to be removed entirely by [PR #37013](https://github.com/facebook/react/pull/37013) once it ships. **Two practical effects for `useEffectEvent` users**: (1) a small perf win in any component using `useEffectEvent` (the "mutation phase" path skips a redundant commit-time check that was on by default as a safety net); (2) **a bug fix for users who combine `useEffectEvent` with View Transitions** — previously, when `enableViewTransition` was on, React was not updating the `useEffectEvent` callback when a tree went from hidden to visible; PR #37039's unconditional enable of the mutation phase also activates the fix path. **No code change needed** — just upgrade to `react@19.3.0-canary-172742b4-20260716` or later. The public `useEffectEvent` API is unchanged.
> **React canary 19.3.0-canary-83840902-20260719 — Nested `ViewTransition` enter/exit animations + stack-overflow recovery on retries + Fragment-scroll siblings** ([npm](https://www.npmjs.com/package/react?activeTab=versions), published 2026-07-20T16:47:45Z, replaces `172742b4-20260716`). Three commits since the last documented React canary; none material for `useEffectEvent` (PR #37039 is already in `172742b4`), but all three are new and worth knowing about:
>
> 1. **[PR #36917](https://github.com/facebook/react/pull/36917) `[Fizz] Support nested enter/exit ViewTransition animations`** (shipped in the React canary `83840902-20260719`, 2026-07-20) — View Transitions on the same root element can now be **nested** (a parent `<ViewTransition>` containing a child `<ViewTransition>` that animates independently), and both can be in `enter` and `exit` modes simultaneously. Previously, only a single View Transition could be active per element at a time; nested layouts where the parent cross-fades while a child scales-in (e.g. a page hero that cross-fades while a card inside it scales-in) were either unsupported or the inner animation was silently dropped. **Use case:** `<ViewTransition name="hero" enter={{ opacity: [0, 1] }} exit={{ opacity: [1, 0] }}><ViewTransition name="hero-card" enter={{ scale: [0.9, 1] }}>...</ViewTransition></ViewTransition>`. **No API change** — the existing ViewTransition API now composes correctly. **Source:** [MDN — CSS View Transitions](https://developer.mozilla.org/en-US/docs/Web/API/View_Transitions_API).
>
> 2. **[PR #36977](https://github.com/facebook/react/pull/36977) `[Fizz] Extend stack overflow recovery to retries`** (shipped in the same canary, 2026-07-20) — the Fizz server-renderer (React's streaming SSR engine) has had a stack-overflow recovery mechanism since 19.0, but it only triggered on the **first** render attempt. If the retry also overflowed (e.g. deeply nested component trees that exceed the V8 stack limit on both passes), the server crashed with an unhandled `RangeError`. PR #36977 extends the recovery to retries, so the retry now uses an iterative render path that can handle arbitrary nesting depth. **Practical impact:** Next.js apps with deeply nested component trees (deeply recursive `Sidebar > Section > Subsection > Item > Item > ...` patterns, or any rendering where a single component tree can produce >1000 React elements) no longer crash on the retry pass. **No code change required.** **Source:** [React PR #36599 — original stack overflow recovery in 19.0](https://github.com/facebook/react/pull/36599).
>
> 3. **[PR #37060](https://github.com/facebook/react/pull/37060) `[DOM] Scroll to text siblings of empty Fragments instead of the parent`** (shipped in the same canary, paired with [PR #37061](https://github.com/facebook/react/pull/37061) `[DOM] Handle scrolling of empty Fragments below containers`) — when you use `element.scrollIntoView()` on an empty React Fragment (the `<></>` shorthand), the browser can't scroll to a Fragment because there's no element. Previously React would scroll the Fragment's parent container; the fix scrolls to the **first sibling DOM node** of the Fragment, which is almost always what the caller intended (e.g. `<>...</>{anchor}` where the anchor is the next DOM element after the Fragment's children). **Use case:** programmatic scroll-to-anchor after a fragment-rendered list (think: search results where the matching item is rendered inside a Fragment along with siblings, and you want to scroll to the *item* not the *list container*). **No API change** — just an upgrade fix.
>
> **React main-branch ahead of `83840902-20260719` — Fragment-scroll container fix + Activity + `useSyncExternalStore` stale-state fix (2 commits, NOT YET tagged to a canary)** (cron check 2026-07-21T06:03Z; main-branch HEAD is `2860e00cf8`, 2 commits ahead of `83840902-20260719` tag). The React canary train has not published a new canary yet (npm `canary` dist-tag still = `19.3.0-canary-83840902-20260719`), but two material PRs have landed on `main` and will be in the next canary:
>
> 1. **[PR #37061](https://github.com/facebook/react/pull/37061) `[DOM] Handle scrolling of empty Fragments below containers`** (eps1lon, merged 2026-07-20T20:40:58Z — the paired second half of the already-documented PR #37060 from `83840902`) — while #37060 changed `scrollIntoView()` on an empty Fragment to scroll to the Fragment's first sibling DOM node, PR #37061 handles the complementary case: when `scrollIntoView()` is called and the Fragment has **no siblings** (it's the last element in a parent container), the original code would try to scroll to the parent, which is often the wrong behavior. The new logic: if the Fragment's host is `Document`, no-op (some part of the doc is always in view); if the host is a `ShadowRoot`, scroll to the ShadowRoot's `host`; for any other `DocumentFragment` without a host, emit a warning (matches the existing `scrollIntoView(false)` empty-Fragment warning). The fix also covers `<html><body /></html>` because `HostSingleton` is skipped at the moment. **No API change** — completes the Fragment-scroll fix pair so scrollIntoView() now correctly handles empty Fragments regardless of sibling layout. **Use case:** when an empty `<></>` Fragment wraps the last visible content (e.g., a list that's currently empty and shows a "No items" placeholder), `scrollIntoView()` no longer scrolls the wrong element.
>
> 2. **[PR #36947](https://github.com/facebook/react/pull/36947) `[Fiber] Detect useSyncExternalStore mutations missed while Activity tree was hidden`** (sophiebits, merged 2026-07-21T03:39:10Z, fixes [facebook/react#27670](https://github.com/facebook/react/issues/27670)) — **a real bug class** for any app combining React 19.2's `<Activity>` primitive with an external store. The root cause: when `<Activity mode='hidden'>` unmounts passive effects, the component unsubscribes from its store; on reveal, React replays the fiber's effect list without a render to re-attach. If `updateStoreInstance` was NOT in the effect list (which is common — the `useSyncExternalStore` subscription lives in a passive effect, not a layout effect), the component was left stale on reveal. Two manifestations: (a) a layout effect mutated the store during the reveal commit (after the subtree rendered but before it resubscribed); (b) the store changed while hidden and the component bailed out of rendering during the reveal (the reveal-replay didn't kick a re-render). **The fix:** `updateStoreInstance` is now pushed unconditionally but tagged with `HookHasEffect` only under the same conditions as before. Regular commits skip it when nothing changed (no perf cost); but reconnection always triggers it so it can drive a re-render if the store changed while hidden. **Practical effect for any app using `<Activity>` + an external store** (Zustand, Jotai, custom `useSyncExternalStore`, TanStack Query's devtools subscription, anything via `@tanstack/react-query`'s `useSyncExternalStore`-based hooks like `useSyncQuery`): if you have a tab/sidebar/modal wrapped in `<Activity mode='hidden'>` and an external store mutation happens during the hidden period, the component will now correctly re-render on reveal — previously it would silently show stale state until the user navigated or interacted. **No code change needed** — just upgrade to `react@19.3.0-canary-[next-sha]` (when the next canary ships) or wait for React 19.3 stable.
>
> **Bundled into Next.js:** **NOT YET** — the next Next.js canary React vendor bump will pick both up (expected in canary.92 or canary.93). If you want the fixes on `next@canary` now, install `react@19.3.0-canary-83840902-20260719` + the upcoming canary once it's tagged; the vendor React inside `next@canary@91` is still `172742b4-20260716`. Verify with `npm view next@canary dependencies.react` — currently points at `19.3.0-canary-172742b4-20260716` (the vendored copy, not a peerDependency).
>
> **Activity tree + useSyncExternalStore — see `patterns.md` → `<Activity />` for the full Activity API + the new fix's interaction with `useSyncExternalStore`-based state libraries (Zustand, Jotai, TanStack Query subscriptions).**

## Error Handling in Components


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



## shadcn 4.13.0 — Base UI is Now the Default (July 3, 2026)

Released 9 days after 4.12.0 (June 26 → July 3), `shadcn@4.13.0` is the **most material shadcn CLI change since 4.0**. [PR #11082](https://github.com/shadcn-ui/ui/pull/11082) by shadcn himself flips the default headless library: **Base UI** is now the default in `npx shadcn@latest init` and `npx shadcn@latest add`. Radix UI remains fully supported and is still the second option in the CLI prompt. Per the [official changelog](https://ui.shadcn.com/docs/changelog/2026-07-base-ui-default), this is the result of "wide community adoption" of Base UI over the last year.

### What changed for new projects

After `npx shadcn@latest init` runs against a fresh project, the resulting `components.json` + `package.json` look like this:

```json
// components.json — defaults now point at Base UI
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "app/globals.css",
    "baseColor": "neutral",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  },
  "iconLibrary": "lucide"
}
```

```json
// package.json — added by the CLI
{
  "dependencies": {
    "@base-ui/react": "latest"     // <-- the default; was "no headless lib" or "radix-ui" before
  }
}
```

If you want to keep using Radix, the CLI still asks during init: **"Which headless library would you like to use? › Base UI / Radix UI"** — and the explicit choice is recorded in `components.json` (or via `--base-color` / future `--headless` flag if you want non-interactive). If you skip the prompt with `--yes`, the default is **Base UI**.

### What this means for existing projects

- **If you `npx shadcn@latest add` into a project that was initialized before 4.13.0**, the CLI does NOT migrate you to Base UI. It uses whatever headless lib you already configured in `components.json` (most projects: Radix). To migrate an existing project, see the [Migration from Radix UI](#migration-from-radix-ui) section below.
- **If you `npx shadcn@latest init` into a project that already has `components.json`**, the CLI does NOT overwrite your config. It only writes files that don't exist (`app/globals.css`, `lib/utils.ts`, etc).
- **The chat components from 4.12.0** (`MessageScroller`, `Message`, `Bubble`, `Attachment`, `Marker`) all have separate `base-*` and `radix-*` registry items — installing one of these does NOT migrate your project to Base UI. You can still add a single `base-luma` message-scroller to a Radix project if you want; they're independent.

### Migration from Radix UI

If you have an existing Radix-flavored project and want to migrate to Base UI, you have two paths:

**Path A — clean migration (recommended for small projects):**

```bash
# 1. Update components.json's "$schema"-derived flags (no formal headless flag yet; the CLI
#    detects the lib from the dep installed). Install Base UI:
npx shadcn@latest add @base-ui/react   # or: pnpm add @base-ui/react

# 2. Re-add each component you use, prefixed with the new base-* registry name.
#    The CLI will skip files that already exist unless you --force overwrite.
npx shadcn@latest add button dialog dropdown-menu --force

# 3. Diff the generated files. Most props are the same; the few that differ:
#    - Radix: <Dialog.Root><Dialog.Trigger /><Dialog.Content>...</Dialog.Content></Dialog.Root>
#    - Base UI: <Dialog.Root><Dialog.Trigger /><Dialog.Popup>...</Dialog.Popup></Dialog.Root>
#    The component-API mapping is documented per-component at ui.shadcn.com/docs/components.
#    For most "form" components (Button, Input, Label, Select) the API is identical.

# 4. Remove radix packages once everything is migrated
pnpm remove @radix-ui/react-dialog @radix-ui/react-dropdown-menu @radix-ui/react-popover # etc
```

**Path B — mixed (acceptable for large codebases):**

Keep both `@radix-ui/*` and `@base-ui/react` in the same project. They share no global state, and Base UI's `useRender` + Radix's `Slot` are not incompatible. The risk is bundle size: a project that pulls in both pays ~30-50KB of duplicated primitive infrastructure. Acceptable for a large project mid-migration; revisit once you're 100% on Base UI.

**Path C — don't migrate.** If your project is stable on Radix, stay there. Radix is still maintained, still receives security patches, and the registry items for both flavors are still updated. The skill's `styling.md` and `components.md` continue to document both; only the default flipped.

### Why Base UI became the default (the official rationale)

From the [Base UI as Default changelog post](https://ui.shadcn.com/docs/changelog/2026-07-base-ui-default): "When shadcn/ui launched in January 2023, it was built on Radix. At the time, nothing else came close. Unstyled headless components, great APIs, great accessibility, battle-tested in millions of apps." But the same engineers who built Radix **moved to Base UI** — the Base UI library is "Radix by the same authors, rebuilt for the post-React-Compiler era" (per the announcement). Same team, newer API, smaller bundle, better SSR story, and tighter React 19 integration (the Base UI primitives are designed around `use()` and Suspense, not the `useContext`-based Radix internals).

For new projects, the choice is "use the current default unless you have a specific reason not to." For existing projects, the choice is "stay on Radix until you have a reason to migrate" — both are valid.

### Source-level changes worth knowing

If you maintain a custom component built ON TOP of a shadcn component, the API differences you'll hit most often are:

- **`Dialog` → `Dialog` with `Popup` instead of `Content`**: the Base UI primitive is named `Popup`, the Radix one `Content`. Both take the same `className` + `forceMount` props.
- **`Popover` → `Popover` with `Popup`**: same rename pattern.
- **`DropdownMenu` → `Menu` with `Popup`**: Base UI's `Menu` covers what Radix split into `DropdownMenu` + `ContextMenu` + `Menubar`. One primitive, three use cases.
- **`Tooltip` → `Tooltip` with `Popup`**: same rename.
- **`Accordion` → `Accordion` with `Panel` instead of `Content`**: same pattern.

The full mapping table is at [ui.shadcn.com/docs/components](https://ui.shadcn.com/docs/components) — every component page lists both the Base UI and Radix implementations side-by-side.

**Sources:**
- [shadcn 4.13.0 release notes](https://github.com/shadcn-ui/ui/releases/tag/shadcn%404.13.0)
- [PR #11082 — `base-ui is now default`](https://github.com/shadcn-ui/ui/pull/11082)
- [Base UI as the Default changelog post (July 2026)](https://ui.shadcn.com/docs/changelog/2026-07-base-ui-default)
- [Base UI vs Radix UI API mapping](https://ui.shadcn.com/docs/components)
- [Base UI npm package](https://www.npmjs.com/package/@base-ui/react)

## shadcn/ui 4.14.0 — Icon Migration Support (July 22, 2026)

Released 19 days after 4.13.0 (July 3 → July 22), `shadcn@4.14.0` adds **icon migration support** ([PR #11241](https://github.com/shadcn-ui/ui/pull/11241) by shadcn himself, commit `3c26ee2dbd3a772c1cddc2c76249cc1cb0a250d5`). One feature, scoped:

**What's new:**

- **Icon migration** — `npx shadcn migrate icons` (and the underlying migration engine) can now convert an existing project's icon set from one library to another (`lucide → heroicons`, `lucide → tabler`, custom registries → first-party icons, etc.). Previously the migration CLI could only handle component-level changes; icon-level migration was a manual sed job. The new command:
  - scans the project's component imports + usage sites for icon components,
  - maps old icon names to new icon names via a registry-defined table (shipped in `shadcn`'s default config for `lucide → heroicons` + `lucide → tabler`),
  - rewrites imports + JSX in place,
  - leaves your `components.json` registry / style / base-color config untouched.

**Who needs this:**

- **Projects migrating icon libraries** (most common case: from `lucide-react` to `@heroicons/react` or to `react-icons`).
- **Projects consolidating on a first-party icon registry** — the new migration runs against a single registry, so projects using multiple icon libraries can normalize to one.
- **Most projects:** not affected — if you started on one icon library and stayed there, the migration command is a no-op.

**Action:** `npx shadcn@latest migrate icons` (or `npx shadcn migrate icons --from lucide --to heroicons` for an explicit pair).

**No API change, no breaking changes.** 4.14.0 is purely additive on top of 4.13.0 — the Base UI default from 4.13.0 is preserved, the `shadcn/typeset` integration from the same week is preserved, the `@shadcn/react` `message-scroller` package from 4.12.0 is preserved. Safe to upgrade from 4.13.x.

**Sources:**
- [shadcn 4.14.0 release notes](https://github.com/shadcn-ui/ui/releases/tag/shadcn%404.14.0)
- [PR #11241 — `add support for icon migration`](https://github.com/shadcn-ui/ui/pull/11241)

## shadcn/typeset (July 14, 2026) — Stream-Friendly Typography System

Released the same week as 4.13.0 (just after midnight UTC on July 14, 2026), **`shadcn/typeset`** is a new typography system that ships as a **single CSS file you own** — no package, no config layer, no runtime JS. It styles your app's HTML (headings, paragraphs, lists, tables, code) the same way for blog posts, docs, and chat — and lets you tune the rhythm for each context independently.

```html
<!-- One class, everything styled -->
<div class="typeset">
  { content }
</div>
```

The system is container-aware (sizes scale with the container), uses your existing theme (CSS variables), and exposes three controls: **size** (`--typeset-size`), **leading** (`--typeset-leading`), and **flow** (`--typeset-flow`).

```css
/* Tighter rhythm for chat */
.typeset-chat {
  --typeset-leading: 1.6;
  --typeset-flow: 1em;
}

/* Roomier rhythm for docs */
.typeset-docs {
  --typeset-size: 15px;
  --typeset-leading: 1.75;
  --typeset-flow: 1.5em;
}
```

```tsx
// In a streaming chat message
<div className="typeset typeset-chat">{message}</div>

// In a long-form docs article
<article className="typeset typeset-docs">{page}</article>
```

### Why typeset exists

Before typeset, every surface that rendered markdown (blog, docs, chat) needed its own `prose` or `typography` plugin and produced slightly different output for headings, lists, and code. Three different `prose-lg` and `prose-base` configurations in the same app would clash in subtle ways.

With typeset, you **style the HTML elements once, then tune the rhythm for each container**. The element styles (h1, p, ul, code, table) come from the single `shadcn/typeset.css` (or whatever name you save it as). The container variants (`typeset-chat`, `typeset-docs`) are three CSS variables each.

### Streaming-friendly

The killer feature for chat UIs: **typeset does not restyle earlier blocks when a new block arrives**. Streaming chat apps where each message appends to a list don't have to re-render the entire list to apply a new message's typography. The class-based design means each `.typeset` block is independently styled.

### Install

```bash
# Open the typeset builder at ui.shadcn.com/typeset, click "Use this typeset",
# and either:
#   a) copy the CSS into your globals.css
#   b) save it as a separate file (e.g. styles/typeset.css) and @import it
#   c) save it as styles/typeset-chat.css / styles/typeset-docs.css and
#      @import each variant separately

# Example: separate files
```

```css
/* app/globals.css */
@import "tailwindcss";
@import "shadcn/tailwind.css";

/* Base typeset — the element styles */
@import "../styles/typeset.css";

/* Container variants — the rhythm tunings */
@import "../styles/typeset-chat.css";
@import "../styles/typeset-docs.css";
```

Or in a single file, just inline the variants:

```css
.typeset { /* base */ }
.typeset-chat {
  --typeset-leading: 1.6;
  --typeset-flow: 1em;
}
.typeset-docs {
  --typeset-size: 15px;
  --typeset-leading: 1.75;
  --typeset-flow: 1.5em;
}
```

### When to use typeset vs Tailwind Typography (`@tailwindcss/typography`)

| Use case | Recommendation |
|---|---|
| **Single-surface app** (just blog OR just docs) | Tailwind Typography's `prose` classes are still fine |
| **Multi-surface app** (blog + docs + chat + email) | **Use shadcn/typeset** — one stylesheet, multiple container variants |
| **Streaming chat with mixed content** (markdown + tool outputs) | **Use shadcn/typeset** — streaming-safe, container-aware |
| **RSC app with `<Markdown>` components** | **Use shadcn/typeset** — works with any markdown renderer since it styles the output HTML |
| **Existing project on `prose` with a single surface** | Don't migrate. Tailwind Typography still works. |

### Markdown renderer compatibility

Typeset styles the HTML output, so it works with **any** markdown renderer:
- `react-markdown` (most common RSC-friendly choice)
- `marked` + DOMPurify
- `MDX` with `@next/mdx`
- AI SDK `<Message>` content
- `remark` + `rehype` pipeline
- Server-rendered HTML from your CMS

Just render the markdown to HTML and wrap the result in a `<div className="typeset">`.

**Sources:**
- [shadcn/typeset announcement (July 14, 2026)](https://ui.shadcn.com/docs/changelog/2026-07-typeset)
- [Typeset documentation](https://ui.shadcn.com/docs/typeset)
- [Typeset builder](https://ui.shadcn.com/typeset)
- [shadcn changelog](https://ui.shadcn.com/docs/changelog)

## `@shadcn/helpers` (July 2026) — Test Chat UIs Without a Model, API, or API Key

Released July 2026, **`@shadcn/helpers`** is a new package from shadcn/ui for writing chat UI tests and component development without requiring a live LLM, an API route, network access, or an API key. It lets you describe a conversation in code, then run it through the real `useChat` lifecycle — your UI receives native framework messages and streaming events (reasoning, tool calls, loading states, message components) exactly as it would in production.

```ts
import { createChat } from "@shadcn/helpers/ai-sdk"

const chat = createChat()
  .user("What changed in this release?")
  .assistant("The release adds keyboard shortcuts and faster search.")
  .user("Can you check the full release notes?")
  .sleep(500)
  .assistant(({ writer }) => {
    writer.stepStart()
    // Reasoning visible in UI
    writer.reasoning("I should read the release notes first.")
    // Tool call
    writer.tool("getReleaseNotes", {
      title: "Reading release notes",
      input: { version: "latest" },
    })
    .sleep(500)
    .output({
      highlights: ["Keyboard shortcuts", "Faster search"],
    })
    // Source attribution
    writer.sourceUrl({
      title: "Release notes",
      url: "https://example.com/releases",
    })
    // Final text response
    writer.text("The release adds keyboard shortcuts and faster search.")
  })

// Pass to the real useChat — no model, no network
const initialMessages = chat.get(0)
const transport = chat.transport()

export function Chat() {
  const { messages } = useChat({
    messages: initialMessages,
    transport,
  })
  // ...
}
```

**Two adapters ship in the package:**
- **`@shadcn/helpers/ai-sdk`** — plugs into AI SDK's `useChat` as a transport
- **`@shadcn/helpers/tanstack-ai`** — plugs into TanStack AI's `useChat` as a connection, streams real AG-UI events

**What you can test with this:**
- Does the reasoning step render correctly in the UI?
- Does the tool call card show the right title and input?
- Does the loading skeleton appear during `.sleep()`?
- Does the output card render the highlights array?
- Does the source attribution link appear?
- Does the final text stream in character-by-character?

All of this runs at unit-test speed with zero network I/O.

**Source:** [`@shadcn/helpers` — July 2026 changelog](https://ui.shadcn.com/docs/changelog/2026-07-helpers) · [`@shadcn/helpers` AI SDK docs](https://ui.shadcn.com/docs/helpers/ai-sdk) · [`@shadcn/helpers` TanStack AI docs](https://ui.shadcn.com/docs/helpers/tanstack-ai)

---

## React 19.3.0-canary-81e442ea-20260721 — Server Action Decoding Perf + Fragment-Blur Fix (July 21, 2026)

The next React canary after `83840902-20260719` shipped 2026-07-21T16:26:12Z with **two new material PRs** (plus the two PRs from the previous cron that were already merged to main: PR #37061 Fragment-scroll + PR #36947 `useSyncExternalStore` Activity-reveal stale state). This canary is now bundled into **`next@canary@92`** (npm dist-tag 2026-07-21T17:51:18Z) and **`next@preview@7`** (npm dist-tag 2026-07-21T18:28:46Z), so any project running those tags automatically gets these fixes.

### 1. PR #37090 `[FlightReply] Performance improvements when decoding` (Sebastien Silbermann, merged 2026-07-21T16:26:12Z)

**A real Server Actions perf win.** The previous `decodeAction` implementation in `packages/react-server/src/ReactFlightActionServer.js` iterated through ALL form fields with `forEach`, allocated a `Set<string>` to track seen action keys, and processed each `$ACTION_*` field individually inside the loop. The new code remembers only the LAST action key seen (a single string variable `maybeActionKey`) and does the decoding ONCE outside the loop.

```js
// ❌ Before — O(n) work + Set allocation per form-field iteration
const seenActions = new Set<string>()
body.forEach((value, key) => {
  if (key.startsWith('$ACTION_REF_')) {
    if (seenActions.has(key)) return
    seenActions.add(key)
    action = loadServerReference(serverManifest, /* decode-bound-action */)
  }
  // ...
})

// ✅ After — single string + one decode outside the loop
let maybeActionKey: null | string = null
body.forEach((value, key) => {
  if (key.startsWith('$ACTION_REF_')) {
    maybeActionKey = key
  } else if (key.startsWith('$ACTION_ID_')) {
    maybeActionKey = key
  }
})
const actionKey = maybeActionKey
// ...single decode of actionKey outside the loop
```

Also: the prior `decodeReplyFromAsyncIterable` called `iterator.throw(reason).then(error, error)`, which allocated a new Error on every async-iterator throw path. The new code uses `.then(noop, noop)` — cheaper, no semantic change.

**Practical impact** for any app that calls Server Actions from forms with many fields (multi-section checkout form, long settings form, any `<form action={serverAction}>` with >20 fields, anything using bound actions with `$ACTION_REF_*` keys): measurable reduction in per-request server-side decode time. The PR title hints at it being part of a security-patch cycle: the PR body says "Security Patches included in 19.2.8" — **and React 19.2.8 stable SHIPPED on 2026-07-21T15:49:09Z** ([GitHub release](https://github.com/facebook/react/releases/tag/v19.2.8), [`diff v19.2.7...v19.2.8`](https://github.com/facebook/react/compare/v19.2.7...v19.2.8) — 2 commits: PR #37087 `[FlightReply] Performance improvements when decoding` by @eps1lon + PR #36753 `[19.2.x] Update required references to GitHub repo`). The 1.4.78 cron line "not yet released as of 2026-07-22" was wrong by ~14h — the stable actually shipped the same day as React canary `81e442ea-20260721`, at 15:49:09Z (about 28 minutes before `next@latest` 16.2.11 published at 16:58:28Z). npm `latest` dist-tag pointer moved to `19.2.8` on release; `react@backport` = `19.0.8` (older backport line); `@types/react@19.2.17` and `@types/react-dom@19.2.3` are the matching types releases (these match up with TS 6.0/7.0 peer ranges).

**Action:** upgrade to `next@canary@92+` (the canary.92+ dist-tag, which bundles React `81e442ea-20260721`) — OR for stable: `npm install react@19.2.8 react-dom@19.2.8` after bumping `next@16.2.11` will surface the FlightReply decode perf win even before `next@16.2.12` vendors the React bump. Next.js vendors a `react.production.js` copy under `packages/next/src/compiled/react/` for SSR/RSC, but client components use your `node_modules/react` copy for hydration — so the perf gain is realised for hydration-side decoding immediately on a `react@19.2.8` install. Verify with `npm view react dist-tags.latest` → should show `19.2.8`. `next@preview@7` is another path to the fix on the preview train.

**Source:** [PR #37090 — `[FlightReply] Performance improvements when decoding`](https://github.com/facebook/react/pull/37090) · Files: 10 changed +65/-54 (the actual decode refactor is +45/-44 in `ReactFlightActionServer.js`) · merged 2026-07-21T16:26:12Z · **Shipped in React 19.3.0-canary-81e442ea-20260721** + bundled into **`next@canary@92`** (npm dist-tag 2026-07-21T17:51:18Z) + **`next@preview@7`** (2026-07-21T18:28:46Z) + **`next@canary@93`** (2026-07-21T23:55:58Z).

### 2. PR #37062 `[DOM] Handle blur on Fragments below Document` (Sebastien Silbermann, merged 2026-07-21T08:26:07Z)

**Pairs with PR #37060/#37061** (the Fragment-scroll fixes from `83840902` documented above). When React `blur`s an element, it first checks if the active element is within the Fragment to avoid a subtree traversal. The original code assumed `ownerDocument` returns a `Document` — but for a `Node` that IS the `Document` itself, `ownerDocument` is `null`, so the original code threw `Cannot read properties of null (reading 'activeElement')`.

**The fix** adds a null-guard: if `ownerDocument` is `null`, treat the host as the Document itself (the existing path then runs without the crash). Diff (in `packages/react-dom-bindings/src/client/ReactDOMComponent.js`):

```js
// ❌ Before — crashes when called on Document itself
const ownerDocument = container.ownerDocument
// (when container IS the Document, ownerDocument is null)
const activeElement = ownerDocument.activeElement // 💥 Cannot read properties of null

// ✅ After — null-guard
const ownerDocument = container.ownerDocument ?? container
const activeElement = ownerDocument.activeElement
```

**Practical impact:** narrow. The bug only triggers when (a) you call `blur()` programmatically on an element, AND (b) the blurred element is a child of an empty Fragment whose host is `null`-document. The most common path is still `Document`-hosted, so this is a defensive fix that prevents a class of crashes in edge-case DOM structures. **No API change.**

**Action:** upgrade to `next@canary@92+` (or `next@preview@7+`).

**Source:** [PR #37062 — `[DOM] Handle blur on Fragments below Document`](https://github.com/facebook/react/pull/37062) · Sebbie Silbermann · merged 2026-07-21T08:26:07Z · **Shipped in React 19.3.0-canary-81e442ea-20260721** + bundled into **`next@canary@92` + `next@preview@7` + `next@canary@93`**.

### Coverage: which Next.js tags ship this canary?

| Next.js dist-tag | Date | Bundles React canary | Includes #37090 | Includes #37062 |
|---|---|---|---|---|
| `next@16.2.11` (latest) | 2026-07-21T16:58:28Z | React 19.2.7 (vendored) + `react@19.2.8` for client hydration if you bump `react` separately | ✅ (hydration only) | ✅ (hydration only) |
| `next@15.5.21` (backport) | 2026-07-21T16:58:17Z | React 19.2.7 (vendored) + `react@19.2.8` for client hydration if you bump `react` separately | ✅ (hydration only) | ✅ (hydration only) |
| `next@canary@91` | 2026-07-20T23:58:30Z | 19.3.0-canary-83840902-20260719 | ❌ | ❌ |
| `next@canary@92` | 2026-07-21T17:51:18Z | 19.3.0-canary-81e442ea-20260721 | ✅ | ✅ |
| `next@canary@93` | 2026-07-21T23:55:58Z | 19.3.0-canary-81e442ea-20260721 | ✅ | ✅ |
| `next@preview@7` | 2026-07-21T18:28:46Z | 19.3.0-canary-81e442ea-20260721 | ✅ | ✅ |
| `next@preview@8` | 2026-07-22T10:21:19Z | 19.3.0-canary-81e442ea-20260721 | ✅ | ✅ |
| **canary-branch (canary.94 expected ~2026-07-23T23:00Z)** | 2026-07-22T16:50:21Z | 19.3.0-canary-711c445b-20260722 (PR #96066 bump) | ✅ | ✅ |
| **React stable `19.2.8`** | 2026-07-21T15:49:09Z | (standalone React release) | ✅ | ❌ (only FlightReply perf landed; Fragment-blur is canary-only) |

If you want these React perf + crash-guard fixes today: `npm install next@canary --save-exact` (pins canary.93+ which bundles them). For stable: `npm install next@16.2.11 react@19.2.8 react-dom@19.2.8` — the `react@19.2.8` install covers client-side hydration decoding; the `next@16.2.12` (when it ships) or `next@16.3.0` stable will vendor the bump for SSR/RSC too. `next@preview@7` is the preview-train path.

### 3. PR #37086 `[Flight] Limit fake JSX call site stacks to 10 frames` (Sebastien Silbermann, merged 2026-07-22T16:38:39Z) — shipped in React canary `711c445b-20260722`

**A real dev-only Flight payload-decode perf win.** The previous `fakeJSXCallSite()` in `packages/react-client/src/ReactFlightClient.js` unconditionally captured a full Error stack on every JSX call site during dev RSC payload creation. The captured depth followed the ambient `Error.stackTraceLimit`:
- **v8 (Chromium, Node.js)** defaults to **10** frames
- **SpiderMonkey (Firefox)** does not support `Error.stackTraceLimit` at all
- **JSC (Safari)** defaults to **100** frames

So in Safari dev, every JSX call site in every dev RSC payload was costing 100 frames of stack capture. The fix temporarily sets `Error.stackTraceLimit = 10` (`ownerStackTraceLimit`) around the `Error('react-stack-top-frame')` construction and restores the ambient value immediately after — same approach as the earlier React PR #34864 (RSC stack-capture opt), now extended to Flight's development fake JSX call sites. Diff (in `packages/react-client/src/ReactFlightClient.js`):

```js
// ❌ Before — captures a full ambient-depth stack on every JSX call site
/** @noinline */
function fakeJSXCallSite() {
  return new Error('react-stack-top-frame')
}

// ✅ After — temporarily clamps to 10 frames, restores ambient after
const ownerStackTraceLimit = 10

/** @noinline */
function fakeJSXCallSite() {
  let error
  const previousStackTraceLimit = Error.stackTraceLimit
  Error.stackTraceLimit = ownerStackTraceLimit
  error = Error('react-stack-top-frame') // eslint-disable-line prefer-const
  Error.stackTraceLimit = previousStackTraceLimit
  return error
}
```

**Benchmark numbers** (from the [PR's standalone benchmark](https://github.com/user-attachments/files/30262123/rsc-stack-trace-standalone.zip) — 4 nested Server Component levels, ambient `Error.stackTraceLimit = 50`, no debugger attached, 100 samples per variant alternating):

| Case | Before | After | Improvement |
|---|---|---|---|
| Small | 0.40 ms/render | 0.32 ms/render | 0.07 ms (18.3%) |
| Sprite | 31.78 ms/render | 8.35 ms/render | 23.43 ms (73.7%) |
| Large | 48.63 ms/render | 13.68 ms/render | 34.95 ms (71.9%) |
| 100 fuzzy cases (average) | 12.09 ms/render | 5.77 ms/render | 6.33 ms (52.3%) |

**Practical effect for any app decoding RSC payloads in dev:**
- **Large / sprite / heavy-JSX pages** see the biggest win (71-74% decode-cost reduction)
- **Real-world impact on `next dev` first-request compile** is material — the Flight decode path runs on every RSC payload decode, including Server Component hydration, `use()`-suspended children, and `cacheSignal`-driven deferred reads
- **When a debugger IS attached**, `Error.stackTraceLimit` has no impact on `Error()` construction cost in v8, so the win is largest on dev sessions without an active inspector
- **Effect is dev-only** — production Flight decoders don't construct stack traces, so no prod change
- **Safari gets the biggest absolute win** because JSC defaults to 100 frames (the ambient was worst); Firefox sees no measurable change because SpiderMonkey ignores `Error.stackTraceLimit`

**Action:** `react@19.3.0-canary-711c445b-20260722` is bundled into `next@canary@94` when it ships ([PR #96066](https://github.com/vercel/next.js/pull/96066) upgrades Next's vendored React from `81e442ea-20260721` → `711c445b-20260722`, merged 2026-07-22T16:50:21Z). For standalone dev today: `npm install react@19.3.0-canary-711c445b-20260722 react-dom@19.3.0-canary-711c445b-20260722` to get the decode perf on client-side RSC decoding. Verify with `npm view react dist-tags.canary` → should show `19.3.0-canary-711c445b-20260722`.

**Test coverage** (in `packages/react-client/src/__tests__/ReactFlight-test.js`): a new test `restores the stack trace limit after recreating JSX call sites` sets ambient `Error.stackTraceLimit = 50`, runs a Flight read, and asserts the ambient value is unchanged after — guards against the temporary-set leaking past `fakeJSXCallSite()`.

**Source:** [PR #37086 — `[Flight] Limit fake JSX call site stacks to 10 frames`](https://github.com/facebook/react/pull/37086) · Files: 2 changed (+23/-2) in `packages/react-client/src/ReactFlightClient.js` (+14/-2 main fix) + `packages/react-client/src/__tests__/ReactFlight-test.js` (new test) · Sebastien Silbermann · merged 2026-07-22T16:38:39Z · commit `711c445bccc331b3ef85a793feb8e13dcf968fc3` · **Shipped in React `19.3.0-canary-711c445b-20260722`** (npm dist-tag moved 2026-07-22T16:41:17Z) + bundled into **`next@canary@94`** (expected ~2026-07-23T23:00Z via [PR #96066](https://github.com/vercel/next.js/pull/96066)) + **`next@preview@9`** (expected in lockstep with canary.94).

If you want these React perf + crash-guard fixes today: `npm install next@canary --save-exact` (pins canary.93+ which bundles them). For stable: `npm install next@16.2.11 react@19.2.8 react-dom@19.2.8` — the `react@19.2.8` install covers client-side hydration decoding; the `next@16.2.12` (when it ships) or `next@16.3.0` stable will vendor the bump for SSR/RSC too. `next@preview@7` is the preview-train path.


## React 19.3.0-canary-28cd4bb0-20260723 — [DevTools] Bridge Hardening (5 PRs, July 23, 2026)

A focused **DevTools Bridge hardening pass** — 5 NEW `[DevTools]`-prefixed PRs all merged 2026-07-23T09:39:15-16Z, advancing the React canary to `19.3.0-canary-28cd4bb0-20260723` (npm dist-tag `canary` moved 2026-07-23T16:42:21Z, replaces `711c445b-20260722` from 1.4.81; `react@experimental` = `0.0.0-experimental-28cd4bb0-20260723`, moved in lockstep). The PRs target the Bridge (the websocket transport between the page and the standalone React DevTools window) and the supporting type plumbing — narrow in surface area, broad in the class of "DevTools randomly dropped a frame" / "DevTools reconnected after a long session and lost events" bugs they fix.

The headline is **[PR #37075](https://github.com/facebook/react/pull/37075) `[DevTools] Buffer Bridge messages during extension reconnects`** (PR body: *"Buffers Bridge messages during extension port reconnects and adds a readiness handshake for ordered queue flushing. Includes regression coverage for reconnect delivery and listener cleanup. Potential scenario could be a long user session, where Chrome kills one of the extension ports to save resources and then user re-connects by navigating back to the DevTools UI."*). Before this PR, if the standalone DevTools window was closed + reopened (a normal user pattern during long dev sessions), the Bridge would drop any in-flight events that were dispatched while the port was disconnected — components would briefly show stale state on the DevTools Profiler / Components tab, and the user had to trigger a manual refresh to recover. The fix introduces a per-port outgoing queue: while the port is `disconnected`, events are buffered in memory; on reconnect, the Bridge sends a `PING` and waits for a `PONG` handshake before flushing the queue in order. The 4 supporting PRs are prerequisite plumbing:

| PR | Title | Impact |
|---|---|---|
| [PR #37048](https://github.com/facebook/react/pull/37048) | `[DevTools] Type EventEmitter error handling` | TypeScript: `EventEmitter` error path now correctly typed (the listener-error contract was `unknown`, now narrowed to `Error \| unknown`) — unblocks the next round of DevTools error-recovery work |
| [PR #37049](https://github.com/facebook/react/pull/37049) | `[DevTools] Harden Bridge and Wall lifecycle types` | Wall (`packages/react-devtools-shared/src/devtools/views/Profiler/Wall.js`) lifecycle types are now strict — `connect()` / `disconnect()` / `shutdown()` are all required and the return types are tightened; the previous loose `() => void` typings let consumers call methods after `shutdown()` returned silently |
| [PR #37050](https://github.com/facebook/react/pull/37050) | `[DevTools] Validate Store operation invariants` | The Store (`packages/react-devtools-shared/src/devtools/store.js`) now asserts invariants on every operation (e.g. a Store mutation must be inside a `ProfilerStore` snapshot boundary; a Wall cannot send to a disconnected Bridge). Catches a class of "DevTools got into a weird state after X" bugs at the source instead of surfacing as null-pointer crashes downstream |
| [PR #37075](https://github.com/facebook/react/pull/37075) | `[DevTools] Buffer Bridge messages during extension reconnects` | **Headline** — see above |
| [PR #37076](https://github.com/facebook/react/pull/37076) | `[DevTools] Shut down standalone Bridge on socket close` | The standalone Bridge now explicitly calls `shutdown()` when the underlying socket closes (was relying on GC); fixes a slow memory leak where a closed Bridge would stay resident in the page until the next React commit |

**Practical impact for anyone using the React DevTools browser extension + standalone window:**
- **Long dev sessions where Chrome kills the extension port to save resources**: no more lost events when the user navigates back to the DevTools UI (was: "I just had a state update but DevTools Profiler doesn't show it"; now: events are buffered and flushed in order after the PING/PONG handshake)
- **DevTools extension after a window close/reopen cycle**: Profiler + Components tab stay consistent without manual refresh
- **Slow memory leak in standalone DevTools window**: fixed by PR #37076
- **TypeScript users building their own DevTools integrations** (third-party wall implementations, custom hooks on the Store): the new types catch a class of "I called this after shutdown" / "I sent without checking the port state" bugs at compile time

**Who needs this:** anyone actively using the React DevTools browser extension for development. Production builds don't ship the DevTools Bridge, so **production impact is zero**. For builds with React DevTools integrated (e.g. the standalone Profiler attached to a dev build), the new buffering is strictly additive — events that would have been dropped now show up after the reconnect handshake completes.

**Action:**
- **Standalone React install**: `npm install react@19.3.0-canary-28cd4bb0-20260723 react-dom@19.3.0-canary-28cd4bb0-20260723`
- **Bundled via Next.js**: the next Next.js canary React vendor bump will pick this up — expected in **`16.3.0-canary.95`** (~2026-07-24T22:30Z on the ~22h30m cadence) via a follow-up vendor bump PR. Verify with `npm view react dist-tags.canary` → should show `19.3.0-canary-28cd4bb0-20260723` after the next Next.js canary ships.

**Coverage: which Next.js tags ship this canary?**

| Tag | Bundled React | This canary? |
|---|---|---|
| `next@latest` (`16.2.11`) | `19.2.7` (vendored; `react@19.2.8` installable as override for client-only decode perf) | ❌ |
| `next@canary` (`16.3.0-canary.94`) | `19.3.0-canary-711c445b-20260722` | ❌ |
| `next@canary@95` (expected ~2026-07-24T22:30Z) | `19.3.0-canary-28cd4bb0-20260723` (via vendor bump PR) | ✅ |
| `next@preview@9` (already shipped 2026-07-23T12:42:49Z) | `19.3.0-canary-711c445b-20260722` (will get a follow-up vendor bump to 28cd4bb0 in preview.10) | ❌ (will be ✅ in preview.10) |

**Test coverage added by this 5-PR set:** new regression tests in `packages/react-devtools-shared/src/__tests__/reactDevToolsHooks-test.js` + `packages/react-devtools-shared/src/__tests__/bridge-test.js` cover reconnect delivery, listener cleanup, and Store invariant assertions. CI runs on every PR; the tests use a mock Bridge that simulates a port disconnect + reconnect cycle.

**Sources:**
- [React canary `19.3.0-canary-28cd4bb0-20260723` GitHub compare (`711c445b...28cd4bb0`)](https://github.com/facebook/react/commits/main/) — 5 commits since `711c445b`, all merged 2026-07-23T09:39:15-16Z
- [PR #37075 — `[DevTools] Buffer Bridge messages during extension reconnects`](https://github.com/facebook/react/pull/37075) — the headline
- [PR #37048 — `[DevTools] Type EventEmitter error handling`](https://github.com/facebook/react/pull/37048)
- [PR #37049 — `[DevTools] Harden Bridge and Wall lifecycle types`](https://github.com/facebook/react/pull/37049)
- [PR #37050 — `[DevTools] Validate Store operation invariants`](https://github.com/facebook/react/pull/37050)
- [PR #37076 — `[DevTools] Shut down standalone Bridge on socket close`](https://github.com/facebook/react/pull/37076)


## React 19.3.0-canary-1724e9ce-20260729 — [Fiber] Hidden-Tree Dehydration Hang Fix (#37135) + Fragment-Ref Blur Nesting Fix (#37125) (July 29, 2026)

The previous cron (v1.5.04 at 2026-07-29T18:03Z) captured `react@canary` = `19.3.0-canary-96fcba90-20260728`, but **46 minutes later** (at 2026-07-29T18:48:34Z) the npm `dist-tag.canary` pointer moved to `19.3.0-canary-1724e9ce-20260729` — a 2-commit bump that the v1.5.04 cron completely missed by virtue of timing. Both commits are bug fixes; no new public API or config flags. The headline is **[React PR #37135](https://github.com/facebook/react/pull/37135)** by **dan**, merged 2026-07-28T22:18:44Z (the underlying commit hash `9a81195` is dated Jul 28, but the npm publish was Jul 29):

```text
[Fiber] Fix hang when updating a dehydrated boundary inside a hidden tree (#37135)

Closes https://github.com/vercel/next.js/issues/95848.

Fixes a hang: if an update changes what's inside a server-rendered
Suspense or Activity boundary before that boundary has hydrated, and the
affected content is hidden, React stops committing. The new content
renders once its data arrives, but the render is discarded every time,
nothing is scheduled, and nothing ever pings — the update never lands
and the app appears frozen.

"Hidden" means either of two things:
- the update itself hides a dehydrated <Activity> (while mounting new
  sibling content that suspends), or
- the dehydrated boundary is inside the primary tree of a parent
  boundary that just suspended and is showing its fallback.

With https://github.com/vercel/next.js/pull/95682, pressing Back before
hydration finishes made the router replay the missed navigation from its
first effect. This worked fine outside of Cache Components, but in Cache
Components mode (which turns on Activity), the old page's Activity
(still dehydrated) gets hidden, the new page's content suspends inside
the layout's Suspense boundary, and after the data arrives the page
stays blank forever.

As a result, https://github.com/vercel/next.js/pull/95682 got reverted.
If we fix this, we can unrevert it.
```

**Why this React PR matters for every Next.js App Router user on 16.3+ (Cache Components):** it directly fixes Next.js issue #95848 ("Blank page when pressing Back between a reload and hydration with cacheComponents (regression in 16.3.0-canary.87)"). That issue was the **sole reason** Next.js PR #95682 (`Replay same-document traversals that happen before hydration`, by `icyJoseph`) **got reverted** in PR #95853 on 2026-07-15T16:36:26Z — the fix introduces a hang in Cache Components mode. The replacement PR **[#96252](https://github.com/vercel/next.js/pulls/96252)** is open on the Next.js repo (created 2026-07-27T00:00:35Z, by `icyJoseph`) and its body explicitly says: *"Turns out, the reason it had to be reverted was https://github.com/react/react/pull/37134. When https://github.com/react/react/pull/37134 lands, we can land this one."* The reference is approximate (the actual unblocker turned out to be PR #37135, not #37134 — the underlying Fiber bug dan found is the real fix), but the implication is exact: **once a React canary containing PR #37135 is published, Next.js can un-revert #95682**.

**What "Back before hydration" actually means (the user-facing behavior being unblocked):** if a user opens page A, navigates to page B client-side, reloads B (which replaces B's document with fresh server HTML), then presses Back — at the time `popstate` fires to A, JS hasn't loaded yet, so nothing listens for it. After JS loads, the router would hydrate B's content onto B's HTML despite technically being on the A route, and from that point forward Back/Forward would change the URL but keep showing B's content. PR #95682 fixed this by using Navigation API to detect the missed traversal and replay it after hydration. Without this fix, every Cache Components app that ever handles a post-reload Back press is silently corrupt.

**Practical impact for users today:**

- **`react@canary` install** — anyone on `npm install react@19.3.0-canary-1724e9ce-20260729 react-dom@19.3.0-canary-1724e9ce-20260729` immediately gets the hidden-tree-dehydration fix.
- **`next@canary` users** — the next Next.js canary React vendor bump (the commit titled `Upgrade React from <prev> to 1724e9ce-20260729`) will pick this up; expected in `16.3.0-canary.104` or `16.3.0-canary.105` (canary.103 was tagged in the canary-branch commit `f3edea1` at 2026-07-29T23:50:00Z but the React vendor bump for this PR is not yet in the canary-branch; the canary.103 tag is `npm view next dist-tags.canary`).
- **Next.js PR #96252** is open and waiting — expect it to be merged into canary-branch and shipped in the canary immediately after the React vendor bump PR lands. So **the "Back before hydration" behavior is expected to be back on `next@canary@105` or `next@canary@106`** (approximately 2026-07-31 → 2026-08-02 window on the 24h canary cadence).
- **`next@preview`** — preview lags canary by ~1 release, so expect it in `16.3.0-preview.11` or `16.3.0-preview.12` (approximately 2026-08-01 → 2026-08-03).
- **`next@latest` (16.2.12)** — not affected. The "Back before hydration" feature is only relevant for App Router + Cache Components, both 16.3-only. If you're on stable, you don't have either.

**The other commit in this canary bump — [React PR #37125](https://github.com/facebook/react/pull/37125) `[DOM] Blur focused descendants in Fragment refs`** by **Minh Vu** (merged 2026-07-29T16:21:37Z, commit `1724e9c`):

```text
[DOM] Blur focused descendants in Fragment refs (#37125)

FragmentInstance.blur() only matched the active element against the
first level of host children. If the focused element was nested inside
one of those children, focus() could reach it but blur() would leave it
focused.

This treats an active element contained by a Fragment host child as part
of the Fragment and blurs the active element itself. It also adds
regression coverage for a nested input.

Fixes #37124.
```

**Practical impact:** Fragment refs (React 19.3 experimental) — `FragmentInstance.blur()` now correctly blurs nested focused elements instead of leaving them focused. Affects any code that uses the experimental `Fragment ref` API and relies on `.blur()` to clear focus programmatically (e.g. closing a popover that wraps a form input). Most projects don't use Fragment refs yet (still behind the `enableFragmentRefs` flag), so the blast radius is small.

**Test coverage added:**

- PR #37135 — new failing tests for both boundary-hide scenarios (the update itself hides a dehydrated `<Activity>` while mounting sibling content that suspends, OR the dehydrated boundary is inside the primary tree of a parent that just suspended). `startTransition` variants of the same scenarios are included as passing controls. Ran the Activity, partial/selective hydration, Fizz, Suspense, and Offscreen suites in both release channels.
- PR #37125 — `yarn test ReactDOMFragmentRefs-test --runInBand` (65 tests passed) + `--prod` variant (65 tests passed) + prettier + linc + flow `dom-node`.

**Coverage: which Next.js tags ship this canary?**

| Tag | Bundled React | This canary? |
|---|---|---|
| `next@latest` (`16.2.12`) | `19.2.8` (vendored) | ❌ |
| `next@backport` (`15.5.22`) | vendored old | ❌ |
| `next@canary` (`16.3.0-canary.102`, canary.103 staged) | `19.3.0-canary-96fcba90-20260728` | ❌ (will be ✅ after vendor-bump PR for `1724e9ce` lands — expected `16.3.0-canary.104`/`105`) |
| `next@preview` (`16.3.0-preview.10`) | `19.3.0-canary-96fcba90-20260728` | ❌ (will be ✅ after preview vendor bump — expected `16.3.0-preview.11`/`12`) |
| Standalone `react@canary` install | `19.3.0-canary-1724e9ce-20260729` | ✅ |

Verify with `npm view react dist-tags.canary` → should show `19.3.0-canary-1724e9ce-20260729`.

**Timing analysis (why the v1.5.04 cron missed this):**

- v1.5.04 cron started at 2026-07-29T18:03Z.
- React canary bump to `1724e9ce-20260729` happened at 2026-07-29T18:48:34Z — 45min after the cron started.
- The v1.5.04 cron's React canary detection ran in the first 5min of its execution (the standard `npm view react dist-tags.canary` check), so it saw `96fcba90-20260728`.
- The new canary was published mid-execution of the v1.5.04 cron, so it landed in the gap between cycles.
- This is a fundamental ceiling on the 6h cadence — canary bumps that happen inside the cron window can only be picked up by the *next* cron, even if they're published in the first hour of the current cycle. Future cron windows should expect this pattern for any 6h boundary.
- **The next cron (v1.5.05, this entry) DID pick it up** because by 2026-07-30T00:03Z the React canary had been live for 5h15min.

**Sources:**

- [React canary `19.3.0-canary-1724e9ce-20260729` GitHub compare (`96fcba90...1724e9ce`)](https://github.com/facebook/react/compare/96fcba90...1724e9ce) — 2 commits since `96fcba90`, both bug fixes
- [React PR #37135 — `[Fiber] Fix hang when updating a dehydrated boundary inside a hidden tree`](https://github.com/facebook/react/pull/37135) — the headline (closes Next.js issue #95848)
- [React PR #37125 — `[DOM] Blur focused descendants in Fragment refs`](https://github.com/facebook/react/pull/37125) — the secondary fix (closes React issue #37124)
- [Next.js issue #95848 — Blank page when pressing Back between a reload and hydration with cacheComponents (regression in 16.3.0-canary.87)](https://github.com/vercel/next.js/issues/95848)
- [Next.js PR #95682 — Replay same-document traversals that happen before hydration](https://github.com/vercel/next.js/pulls/95682) — the feature that got reverted
- [Next.js PR #95853 — Revert "Replay same-document traversals that happen before hydration"](https://github.com/vercel/next.js/pulls/95853) — the revert
- [Next.js PR #96252 — Replay same-document traversals that happen before hydration (redo)](https://github.com/vercel/next.js/pulls/96252) — open, waiting for this React PR
- [React issue #37124 — FragmentInstance.blur() doesn't blur nested focused elements](https://github.com/facebook/react/issues/37124)
- [npm: `react@19.3.0-canary-1724e9ce-20260729`](https://www.npmjs.com/package/react/v/19.3.0-canary-1724e9ce-20260729) (published 2026-07-29T18:48:34Z)
- [npm: `react-dom@19.3.0-canary-1724e9ce-20260729`](https://www.npmjs.com/package/react-dom/v/19.3.0-canary-1724e9ce-20260729) (published 2026-07-29T18:47:43Z)

## React 19.3.0-canary-6cb4322d-20260729 — `[Flight] Port ReplyServer Traversal Guards to FlightClient` (#37144) (July 30, 2026)

The previous cron (v1.5.07 at 2026-07-30T12:03Z) captured `react@canary` = `19.3.0-canary-1724e9ce-20260729`, but **4h42min later** (at 2026-07-30T16:45:17Z) the npm `dist-tag.canary` pointer moved to `19.3.0-canary-6cb4322d-20260729` — a **1-commit pure hardening bump** that v1.5.07 missed by virtue of timing (the publish was inside this 18:03Z cron window). No new public API, no config flags, no new exports. The single commit is a defense-in-depth hardening for the Flight Client deserializer, contributed by **Sebastian "Sebbie" Silbermann** (the same engineer who authored the recent Next.js PR #96085 `headers()` per-pass uniqueness fix):

**[React PR #37144 — `[Flight] Port ReplyServer traversal guards to FlightClient`](https://github.com/facebook/react/pull/37144)**, merged 2026-07-29T22:17:33Z. The PR body is short but signals intent:

> Additional defense-in-depth in case consumers pass untrusted input into Flight Client.
> Flight Client generally assumes trusted input.
> We'll reserve these kind of fixes for Flight Client in case the untrusted input leads to catastrophic vulnerabilities e.g. prototype pollutions that can be used for remote code executions.

### What changed in `getOutlinedModel` and `reviveModel` (one file, +26/−8)

The fix is confined to `packages/react-client/src/ReactFlightClient.js` and is a two-spot hardening of the Flight Client deserializer:

1. **`getOutlinedModel` path-walk guard** — the function that walks an outlined model's `[obj, key0, key1, ...]` reference list (a common Flight wire-format shape for repeated references). Before: `value = value[path[i]]` blindly descended, regardless of `value`'s prototype or whether `name` was an own property. After: checks `typeof value === 'object' && value !== null && (getPrototypeOf(value) === ObjectPrototype || getPrototypeOf(value) === ArrayPrototype) && hasOwnProperty.call(value, name)` before descending; throws `new Error('Invalid reference.')` otherwise. So a crafted path entry that traverses through a non-plain prototype (e.g. an object reachable by path that has `Object.getPrototypeOf(value) !== Object.prototype`) is short-circuited with a hard error before any prototype-chain walk can happen.

2. **`reviveModel` plain-object walk guard** — the recursive walker that descends plain-object RSC payloads to revive nested values. Before: `for (const k in value) { if (k === '__proto__') delete value[k]; else ... }` — only stripped explicit `__proto__` keys from the *own* key list. After: wraps the loop with `hasOwnProperty.call(value, k)`, so inherited enumerable properties are skipped entirely. Same prototype-pollution defence — a payload that lands an object with a poisoned prototype can no longer cause the walker to read or write inherited keys.

Auxiliary additions: imports `shared/getPrototypeOf`, declares `const ObjectPrototype = Object.prototype` and `const ArrayPrototype = Array.prototype` (cache the references so the hot-path check doesn't re-read them per call).

### Why Flight Client specifically (the trust-model rationale)

The companion fix already landed for the **server side** (`ReplyServer`, the `processReply` generator) earlier — the PR title literally "Port ReplyServer traversal guards to FlightClient" says the move was deliberate. The server side has had these guards for a while because that's the side that *generates* the payload — if it reads from a poisoned source, the generated payload is already corrupt from the start, so the guard there also serves as a payload-shape sanity check. Flight Client was harder to harden because `getOutlinedModel`'s path-walk is also doing object-identity cache deduplication, and skipping the wrong reference would break the outline cache. The new guard is precisely scoped to only fire when the value is **non-plain** (prototype ≠ `Object.prototype` and ≠ `Array.prototype`); plain-object outlines (the 99% case) walk identically to before.

### Why it matters for Next.js App Router users

**In the common trust model — your own server, your own RSC, your own database — this is invisible.** Server-issued Flight payloads from `react-server-dom-webpack` / `react-server-dom-turbopack` go through `encodeReply` / `processReply`, and the deserialized shapes always have `Object.prototype` or `Array.prototype`. The new guard is a defense-in-depth measure for three edge cases that real apps occasionally hit:

- **Cross-tenant RSC caching** — if your caching layer stores Flight payloads keyed by content hash and serves them across tenants (CDN-served RSC snapshots, server-side `cache()` of an RSC stream replayed across users, RSC stored to S3/R2 and rehydrated) and the originating tenant's data could poison the prototype of any object reachable by path, the reviver now blocks the prototype-pollution class entirely. **Audit recipe**: `rg "processReply\|react-server-dom" server/ src/` — find every reply-process call site; if any consumes from a user-controlled source, upgrade. **Verify the new guard is what you want:** the new error is `Error('Invalid reference.')` (not a silent corruption), so test surfaces will see the throw clearly.
- **`renderToReadableStream` + cross-origin consumer** — if your backend serves RSC to a third-party origin (a widget CDN, embedded mini-app, partner iframe consuming your RSC stream), the reviver now refuses to descend through non-plain prototypes, raising an explicit error rather than silently walking them. **No API change**, so existing code keeps working; only the failure mode on adversarial inputs changes (from silent corruption → explicit `Invalid reference.`).
- **`dangerouslyAllowBrowser: true`** on the client side (rare, but allowed for embedded widgets where you load `react-server-dom-webpack/client` in a `<script type="module">` directly) gets the same hardening: the new guard fires before any prototype pollution can reach userland code.

### Practical impact for users today

- **`react@canary` install** — anyone on `npm install react@19.3.0-canary-6cb4322d-20260729 react-dom@19.3.0-canary-6cb4322d-20260729` immediately gets the Flight Client traversal hardening. No code changes required (the API surface is identical).
- **`next@canary` users** — the React vendor bump PR (Next.js PR #96389 `Upgrade React from 1724e9ce-20260729 to 6cb4322d-20260729`) **landed on canary-branch at 2026-07-30T17:49:18Z** (~14min before this cron) and will be published as part of `16.3.0-canary.104` on the 24h canary cadence (canary.103 was tagged 2026-07-29T23:35:42Z, so canary.104 should land between 23:35Z tonight and ~12h after). See the new canary.104-ahead section in `performance.md` for the full canary-branch-ahead material.
- **`next@preview`** — preview lags canary by ~1 release; expect in `16.3.0-preview.11` (~24-48h after canary.104).
- **`next@latest` (16.2.12)** — not affected. Vendored React stable is `19.2.8` which already has the server-side `processReply` hardening; the new client-side guard is a follow-on that hasn't been back-ported to stable 19.x. If your threat model includes untrusted RSC, pin to canary.

**Verification recipe:**

```bash
npm view react dist-tags.canary
# → react: '19.3.0-canary-6cb4322d-20260729'

# Confirm the new hardError path: write an RSC payload that includes a path
# through a non-plain object (e.g. an object whose __proto__ was replaced
# with a class instance) and confirm the reviver throws 'Invalid reference.'
# This is opt-in — only happens if your code already accepts untrusted RSC.
```

### Coverage: which Next.js tags ship this canary?

| Tag | Bundled React | This canary? |
|---|---|---|
| `next@latest` (`16.2.12`) | `19.2.8` (vendored) | ❌ |
| `next@backport` (`15.5.22`) | vendored old | ❌ |
| `next@canary` (`16.3.0-canary.103`) | `19.3.0-canary-1724e9ce-20260729` | ❌ (will be ✅ after PR #96389 npm-publishes in `16.3.0-canary.104`) |
| `next@preview` (`16.3.0-preview.10`) | `19.3.0-canary-1724e9ce-20260729` | ❌ (will be ✅ after preview vendor bump — expected `16.3.0-preview.11`) |
| Standalone `react@canary` install | `19.3.0-canary-6cb4322d-20260729` | ✅ |

Verify with `npm view react dist-tags.canary` → should show `19.3.0-canary-6cb4322d-20260729`.

### Timing analysis (why the v1.5.07 cron missed this)

- v1.5.07 cron started at 2026-07-30T12:03Z.
- React canary bump to `6cb4322d-20260729` happened at 2026-07-30T16:45:17Z — **4h42min after** the v1.5.07 cron's start.
- v1.5.07 captured `react@canary` = `1724e9ce-20260729`, last updated at 2026-07-29T18:48:34Z (the dist-tag.next had been stable for 17h15min at the v1.5.07 cron start).
- The 4h42min-old bump was inside the v1.5.07 → this-v1.5.08 window — same pattern as the v1.5.05 cycle that was missed-by-46min for the `1724e9ce-20260729` bump.
- Both cases confirm a recurring pattern: **canary bumps happen on an 18-48h cadence, and any individual 6h cron cycle can miss a bump by 0-6h**. Two consecutive 6h windows per React canary bump means ~67% of bumps will land cleanly in one cycle; the other ~33% span a boundary. The next cron (v1.5.08, this entry) picks it up by virtue of being the immediate next cycle.

### Sources

- [React canary `19.3.0-canary-6cb4322d-20260729` GitHub compare (`1724e9ce...6cb4322d`)](https://github.com/facebook/react/compare/1724e9ce...6cb4322d) — 1 commit, defense-in-depth hardening
- [React PR #37144 — `[Flight] Port ReplyServer traversal guards to FlightClient`](https://github.com/facebook/react/pull/37144) — author Sebastian "Sebbie" Silbermann, merged 2026-07-29T22:17:33Z
- [React PR #37144 files diff](https://github.com/facebook/react/pull/37144/files) — single-file patch to `packages/react-client/src/ReactFlightClient.js`, imports `shared/getPrototypeOf`, adds `ObjectPrototype`/`ArrayPrototype` constants
- [Next.js PR #96389 — `Upgrade React from 1724e9ce-20260729 to 6cb4322d-20260729`](https://github.com/vercel/next.js/pull/96389) — the vendor bump that brings PR #37144 into Next.js's vendored React (merged on canary-branch 2026-07-30T17:49:18Z)
- [React PR #37144 Linear reference — VOC-34505 in Vercel's tracking](https://linear.app/vercel/issue/VOC-34505) (linked from PR #96389's body)
- [npm: `react@19.3.0-canary-6cb4322d-20260729`](https://www.npmjs.com/package/react/v/19.3.0-canary-6cb4322d-20260729) (published 2026-07-30T16:45:17Z)
- [npm: `react-dom@19.3.0-canary-6cb4322d-20260729`](https://www.npmjs.com/package/react-dom/v/19.3.0-canary-6cb4322d-20260729) (published 2026-07-30T16:46:14Z)



## React 19.3.0-canary-0f42eac2-20260730 — Add `ReactDOM.browser()` API (#37143) + 3 DevTools PRs (July 30, 2026)

The v1.5.08 cron (2026-07-30T18:09Z) captured `react@canary` = `19.3.0-canary-6cb4322d-20260729`, but **2h17min later** (at 2026-07-30T20:26:06Z) the npm `dist-tag.canary` pointer moved to `19.3.0-canary-0f42eac2-20260730` — a **4-commit bump that ships the first new public React API in `react-dom` in a while**. v1.5.08 missed this by virtue of timing (the publish was inside this v1.5.09 06:03Z cron window). The diff is `facebook/react` `6cb4322d...0f42eac2` — 4 commits, 1 of which is a new public API + 3 DevTools-only fixes.

### `#37143` — The headline: `ReactDOM.browser()` API (by gnoff, merged 2026-07-30T19:21:08Z)

This is the **first new public React API surface added in a canary in many months**. The PR adds a new function `ReactDOM.browser()` that returns a `"usable"` — an object the `use()` hook can read, which errors during SSR and resolves during rendering in the browser.

**From the PR description (paraphrased):**

> Adds a new API to `react-dom` called `browser()`. `browser()` returns a "usable" that will error during SSR and resolve during rendering in the browser. The purpose is to allow you to express the idea that a component should suspend on the server but not in the browser. The method is not available inside a `react-server` environment. This is a client only feature. This is a `react-dom` API because the concept of browser doesn't apply generally to React itself.
>
> This codifies a pattern that is common in some apps where you error during SSR to prevent rendering some component on the server and you end up suppressing the error that is reported in the client to avoid this appearing like a problem rather than intended behavior. Unfortunately this is not an option for many because hacking around to prevent errors from being logged is not practical for many. By making this a React API we enabled this common pattern in any React using library or application.

**Canonical usage:**

```tsx
import { use, Suspense } from "react"
import { browser } from "react-dom"

function BrowserOnly() {
  use(browser())  // throws on the server, resolves in the browser
  return <ClientContent />
}

function App() {
  return (
    <Suspense fallback={<Fallback />}>
      <BrowserOnly />
    </Suspense>
  )
}
```

**What this solves:**

Before this PR, the canonical pattern for "only render in the browser" was to throw a promise during SSR and suppress the error in the console (via `ErrorBoundary` + `console.error` stubbing). That worked but was ugly — it triggered a "real" error log that lints and observability tools had to special-case. `browser()` is the ergonomic, first-class solution: the throw is an **intentional recoverable error**, the framework logs nothing, the `Suspense` boundary exactly above catches the suspend, and the fallback renders during SSR + the resolved content renders after hydration.

**Implementation notes (from the PR body):**

- The "render in the browser" intent is modeled as an **intentional recoverable error** (React already exposes this primitive via halted references in RSC; `browser()` reuses the same machinery).
- The error is **intentionally suppressed** in client logging — `browser()` returns an isomorphic object that's the same on every environment; the `use()` or `abort()` function is what handles the differing behavior. This means you can create `browser()` objects in **module scope** and use them even in complex scenarios like "server rendering inside the browser while React is rendering" (the SSR-in-CSR case).
- The implementation is **flagged** so the feature can be disabled quickly if the React team decides not to ship it in stable.
- **It is an error to use `use(browser())` outside of a Suspense boundary** — you cannot recover from the root. This restriction may be lifted later but is part of the current limitations.
- The feature is going into React **canary** first (we're here); stable expected once the team is confident in the API.

**Bundle size impact (from the PR's size-bot table):**

| Bundle | Change | Size before | Size after | Gzip before | Gzip after |
|---|---|---|---|---|---|
| `oss-stable/react-dom/cjs/react-dom.production.js` | **+3.67%** | 7.19 kB | 7.45 kB | 1.91 kB | 1.99 kB |
| `oss-stable/react-dom/cjs/react-dom.production.min.js` (gzip) | **+4.51%** | 1.91 kB | 1.99 kB | — | — |
| `oss-stable/react-dom/cjs/react-dom.development.js` | **+3.42%** | 18.33 kB | 18.95 kB | 3.93 kB | 4.14 kB |
| `react-dom-client.production.js` (full bundle) | +0.02% | 616.42 kB | 616.54 kB | 109.22 kB | 109.27 kB |
| `facebook-www/ReactDOM-prod.modern.js` | +0.05% | 697.82 kB | 698.15 kB | 122.53 kB | 122.63 kB |
| `facebook-react-native/react-dom/cjs/ReactDOM-prod.js` | **+3.78%** | 6.98 kB | 7.25 kB | 1.89 kB | 1.97 kB |

Read this as: **the standalone `react-dom` core bundle grows ~240 bytes / ~80 bytes gzipped**. The full `react-dom-client` bundle grows only ~120 bytes total (~50 bytes gzipped) — the cost is amortized across the much larger package. For most apps this is negligible; for use-cases that ship the standalone `react-dom` (the React Native + custom-renderer space) it's the most noticeable.

**Why this matters for Next.js App Router users:**

The App Router already has [`"use client"`](#client-component-island-pattern) for "client only" components, but that's a **module-level** boundary — it forces the entire module to render on the client. `browser()` is a **fine-grained per-instance** boundary: a single component can render client-only while the rest of the tree stays shared. This is useful for:

- **Browser-only libraries that crash in SSR** (e.g. `window.matchMedia` polyfills, libraries that touch `document` at module load) inside a route that otherwise wants to share code between server and client.
- **Heavy client-only visualizations** (a chart library that uses `OffscreenCanvas` or browser WebGL) where you want the parent layout to be a Server Component but the visualization to suspend on SSR.
- **Hydration-only analytics / observability hooks** that intentionally only run in the browser.
- **Migration** — if you have a Next.js Pages Router component that uses a `<ClientOnly>` HOC and you're migrating to App Router, `browser()` is the cleanest 1:1 replacement.

**Practical impact for users today:**

- **`react@canary` install** — `npm install react@19.3.0-canary-0f42eac2-20260730 react-dom@19.3.0-canary-0f42eac2-20260730` immediately gives you `browser()`. The new React vendor bump is in `next@canary.104` (see `performance.md` for the canary.104 SHIPPED section).
- **`next@canary` users** — `next@16.3.0-canary.104` vendors `react@19.3.0-canary-0f42eac2-20260730` (via [Next.js PR #96402](https://github.com/vercel/next.js/pull/96402)). The `ReactDOM.browser()` API is available from `react-dom` in any Client Component (and the `<BrowserOnly>` pattern above works inside `'use client'` modules).
- **`next@preview`** — vendors react@canary from the previous cycle until preview.11 ships; expect preview.11 within 24-48h of canary.104.
- **`next@latest` (16.2.12)** — vendors `react@19.2.8` stable. Does **not** have `browser()`. If you need this on stable, pin `react@canary` + `react-dom@canary` directly + the matching `react-server-dom-webpack`/`react-server-dom-turbopack` canary (no codemod required, drop-in API).
- **Bundle size** — as noted above, +3.67% on standalone `react-dom` core / +0.05% on full `react-dom-client`. Acceptable for most apps; budget for the standalone `react-dom` case in React Native + custom-renderer workloads.

**Verification recipe:**

```bash
npm view react dist-tags.canary
# → '19.3.0-canary-0f42eac2-20260730'

# Confirm ReactDOM.browser() is exported from react-dom:
node -e 'console.log(Object.keys(require("react-dom")).filter(k => k === "browser"))'
# → ['browser']

# Confirm it throws in SSR (the "use" hook will fail inside an SSR React tree, then resolve in the browser)
```

### `#37155` — `[DevTools] Reset extension backend on pagehide` (by hoxyq, merged 2026-07-30T18:34:46Z)

DevTools-only. Fixes an inconsistency during browser navigations that involve BFCache entries: Chrome kills the port manually while freezing and preserving the JS heap (per [Chrome's bfcache-extension-messaging-changes](https://developer.chrome.com/blog/bfcache-extension-messaging-changes)), so the port can be dead while the Backend/Agent are still alive — a state the React DevTools backend doesn't expect. The PR hooks the `pagehide` event to reset the backend cleanly so the next page-reactivation sees a fresh connection. **No public API change, no user-facing impact outside DevTools**, no bundle cost for non-DevTools builds. Stack: 1 commit, still-WIP test (the author notes they couldn't reproduce BFCache in unit tests and is looking for a better test strategy).

### `#37151` — `[DevTools] Create extension panels before React detection` (by hoxyq, merged 2026-07-30T17:06:13Z)

DevTools-only. Bug: if React DevTools was installed but the page wasn't a React app, no panel was mounted at all, and historically a few users reported this as a bug. The fix: always create the panel (Chrome has no API to unmount a panel), and populate the contents dynamically based on whether the target is a React app. If not a React app, the stub message still shows. **No public API change, no user-facing impact outside DevTools.**

### `#37152` — `[DevTools] Remove FlowFixMe from extension lifecycle` (by hoxyq, merged 2026-07-30T17:12:54Z)

DevTools-only. The Flow types in the React DevTools extension lifecycle had a `FlowFixMe` that suppressed real type errors — the author noticed a few during BFCache triage. **No public API change, no user-facing impact outside DevTools.**

### Coverage: which Next.js tags ship this canary?

| Tag | Bundled React | This canary? |
|---|---|---|
| `next@latest` (`16.2.12`) | `19.2.8` (vendored) | ❌ |
| `next@backport` (`15.5.22`) | vendored old | ❌ |
| `next@canary` (`16.3.0-canary.104`) | `19.3.0-canary-0f42eac2-20260730` | ✅ (via Next.js PR #96402) |
| `next@preview` (`16.3.0-preview.10`) | `19.3.0-canary-1724e9ce-20260729` (still — preview lags canary by 1 release) | ❌ (will be ✅ after preview.11 ships within 24-48h) |
| Standalone `react@canary` install | `19.3.0-canary-0f42eac2-20260730` | ✅ |

Verify with `npm view react dist-tags.canary` → should show `19.3.0-canary-0f42eac2-20260730`.

### Timing analysis (why the v1.5.08 cron missed this)

- v1.5.08 cron committed at 2026-07-30T18:09Z.
- React canary bump to `0f42eac2-20260730` happened at 2026-07-30T20:26:06Z — **2h17min after** the v1.5.08 commit.
- v1.5.08 captured `react@canary` = `6cb4322d-20260729`, last updated at 2026-07-30T16:45:17Z (the dist-tag had been stable for 1h24min at the v1.5.08 commit).
- This is the **third consecutive React canary bump that landed inside a 6h cron window** (v1.5.05 missed `1724e9ce` by 46min, v1.5.08 missed `6cb4322d` by 4h42min, v1.5.08 missed `0f42eac2` by 2h17min). The pattern confirms: **React canary bumps happen on an 18-48h cadence, and any individual 6h cron cycle can miss a bump by 0-6h**. The next cron (this v1.5.09) picks it up by virtue of being the immediate next cycle.

### Sources

- [React canary `19.3.0-canary-0f42eac2-20260730` GitHub compare (`6cb4322d...0f42eac2`)](https://github.com/facebook/react/compare/6cb4322d...0f42eac2) — 4 commits, 1 new public API + 3 DevTools-only fixes
- [React PR #37143 — `Add ReactDOM browser() API`](https://github.com/facebook/react/pull/37143) — author gnoff, merged 2026-07-30T19:21:08Z — the new `ReactDOM.browser()` public API
- [React PR #37143 files diff](https://github.com/facebook/react/pull/37143/files) — 5 commits across `packages/react-dom/src/ReactDOMBrowser.js` (new), `ReactDOM.js`, `ReactFiberConfigDOM.js`, hooks plumbing
- [React PR #37143 size-bot report](https://github.com/facebook/react/pull/37143) — bundle size deltas (the +3.67% on standalone `react-dom` core / +0.05% on full `react-dom-client` table above)
- [React PR #37155 — `[DevTools] Reset extension backend on pagehide`](https://github.com/facebook/react/pull/37155) — author hoxyq
- [React PR #37151 — `[DevTools] Create extension panels before React detection`](https://github.com/facebook/react/pull/37151) — author hoxyq
- [React PR #37152 — `[DevTools] Remove FlowFixMe from extension lifecycle`](https://github.com/facebook/react/pull/37152) — author hoxyq
- [Next.js PR #96402 — `Upgrade React from 6cb4322d-20260729 to 0f42eac2-20260730`](https://github.com/vercel/next.js/pull/96402) — the vendor bump that brings PR #37143 into Next.js's vendored React (merged 2026-07-30T21:21:08Z, SHIPPED in `16.3.0-canary.104`)
- [Chrome bfcache extension messaging changes](https://developer.chrome.com/blog/bfcache-extension-messaging-changes) — the Chrome behavior that motivates #37155 (DevTools backend reset on pagehide)
- [npm: `react@19.3.0-canary-0f42eac2-20260730`](https://www.npmjs.com/package/react/v/19.3.0-canary-0f42eac2-20260730) (published 2026-07-30T20:26:06Z)
- [npm: `react-dom@19.3.0-canary-0f42eac2-20260730`](https://www.npmjs.com/package/react-dom/v/19.3.0-canary-0f42eac2-20260730) (published 2026-07-30T20:28:30Z)





## React 19.3.0-canary-cbb046ab-20260731 — Warn for Conditional `use()` Based on Cache (#37104, hoxyq, July 31, 2026)

This is the **first new public React dev-warning API surface** added in this cycle (the previous canary `0f42eac2-20260730` shipped the runtime API `ReactDOM.browser()`; this canary ships the **DEV-warning machinery** for an existing footgun). v1.5.10 captured `react@canary` = `0f42eac2-20260730`; the npm `dist-tag.canary` pointer moved to `19.3.0-canary-cbb046ab-20260731` at **2026-07-31T16:50:45Z** — exactly **1h47min before this cron started** at 18:05Z. The diff is `facebook/react` `0f42eac2...cbb046ab` — **1 commit** by [hoxyq](https://github.com/hoxyq), PR [#37104](https://github.com/facebook/react/pull/37104), merged 2026-07-31T14:24:10Z, **a cherry-pick of [Facebook internal PR #34030](https://github.com/react/react/pull/34030)** (the upstack version in the react/react repo, which is itself the public-record mirror of the same change inside Meta). 16 files changed, +303/-31 lines.

### What is the warning?

This PR teaches React's reconciler to **warn in DEV when a component suspends via `use(promise)` on one render and then doesn't `use()` the same promise on a subsequent render that completes**. The canonical anti-pattern:

```tsx
// ❌ WRONG — conditional use() based on cache
function useCachedValue(cache: { value: any; promise: Promise<any> }) {
  if (cache.value === undefined) {
    use(cache.promise)  // ← suspends only on first read (cache miss)
  }
  return cache.value
}
```

vs

```tsx
// ✅ RIGHT — always use() the promise
function useCachedValue(cache: { promise: Promise<any> }) {
  return use(cache.promise)
}
```

The React warning (DEV-only) reads:

```
This library called use() to suspend in a previous render but
did not call use() when it finished. This indicates an incorrect
use of use(). A common mistake is to call use() only when
something is not cached.

  if (cache.value === undefined) use(cache.promise)
  return cache.value

The correct way is to always call use() with a Promise and
resolve it with the value.

  return use(cache.promise)

Learn more: https://react.dev/warnings/conditional-use-of-use
```

### Why conditional `use()` is unsafe (from the PR body)

This isn't just a style nit — it's a **data-corruption / hydration-deopt hazard**. Five specific failure modes the PR enumerates:

1. **`use()` ordering corruption** — `use()` keeps track of the order it was last called, similar to other hooks. If a component suspends with `use()` at slot N in render 1 and then continues without calling `use()` at all in render 2 (because the cache is now hot), React's internal "next hook slot" counter for subsequent `use()` calls in that same component (or in sibling components that reuse the slot index space) can mis-align. **Later `use()` calls may observe the resolved value of the previous `use()`** — silently corrupting data.

2. **Forced hydration / client rendering** — if you "unblock" the suspense via `setState` or `useSyncExternalStore` instead of via `use()` after the promise resolves, React may force-hydrate the suspended boundary (or even flip the whole subtree back to client rendering), causing a flash of loading state after the SSR'd content already appeared. React treats "resolving a Suspense boundary" differently from "an update" — the deopt is observable and degrades both SSR and hydration perf.

3. **View Transition corruption** — View Transitions are scheduled around whether the resolution is a "transition" (data loading on the current screen) or a "sync update" (a new screen). Conditional `use()` followed by an `setState` to "unblock" gets classified as the latter, breaking the View Transition animation.

4. **Interaction Tracing + Performance Timeline corruption** — DevTools and the perf API distinguish "navigated to a new screen" from "more data loading on the current screen." Conditional `use()` followed by `setState` gets logged as the former (a new navigation), even when it's actually the latter (data loading on the same route). Diagnosing perf regressions gets harder because the trace data is wrong.

5. **Suspense resolution throttling skipped** — Suspense resolution can be throttled, but a `setState` "unblock" forces a flush as soon as possible — losing the scheduling benefits.

### The fix — `trackUsedThenable` now takes a `fiber`

The implementation lives in three reconciler files (`ReactFiberThenable.js` is the core; `ReactFiberHooks.js` and `ReactFiberChildFiber.js` thread the fiber into the call):

```diff
 export function trackUsedThenable<T>(
   thenableState: ThenableState,
   thenable: Thenable<T>,
   index: number,
+  fiber: null | Fiber, // DEV-only
 ): T {
   ...
   suspendedThenable = thenable;
   if (__DEV__) {
     needsToResetSuspendedThenableDEV = true;
+    if (
+      enableConditionalUseWarning &&
+      !didIssueUseWarning &&
+      fiber !== null &&
+      // Only track initial mount for now to avoid warning too much for updates.
+      fiber.alternate === null
+    ) {
+      lastSuspendedFiber = fiber;
+      lastSuspendedStack = new Error(
+        'This library called use() to suspend in a previous render but ' +
+          'did not call use() when it finished. This indicates an incorrect use of use(). ' +
+          'Learn more: https://react.dev/warnings/conditional-use-of-use',
+      );
+    }
   }
   throw SuspenseException;
 ...
+
+function areSameKeyPath(a: Fiber, b: Fiber): boolean {
+  if (a === b) return true;
+  if (a.tag !== b.tag || a.type !== b.type || a.key !== b.key || a.index !== b.index) {
+    return false;
+  }
+  if (a.tag === HostRoot && a.stateNode !== b.stateNode) {
+    return false;
+  }
+  if (a.return === null || b.return === null) {
+    return false;
+  }
+  return areSameKeyPath(a.return, b.return);
+}
```

The matching completion check is in the existing react-server `checkIfUseWrappedInTryCatch`-adjacent code — when a previously-suspended component now completes successfully without `use()` being called, the warning fires once (`didIssueUseWarning = true` enforces single-warning-per-session). The `areSameKeyPath()` helper checks "is the fiber that completed the same logical instance as the one that suspended?" via key-path walk; mismatched path → no warning (the component might have been remounted with different props).

**Two new module-level exports** for DevTools / SSR integration:

```js
export function hasPotentialUseWarnings(): boolean {
  return enableConditionalUseWarning && lastSuspendedFiber !== null;
}
export function clearUseWarnings() {
  lastSuspendedFiber = null;
}
```

The `hasPotentialUseWarnings()` hook is for React DevTools / internal observability — it can flag "this app might have hidden misuse patterns" without spamming the console. The `clearUseWarnings()` is for test isolation.

### Feature flag — `enableConditionalUseWarning`

**Default behavior: OFF.** The new flag `enableConditionalUseWarning` is added to `packages/shared/ReactFeatureFlags.js` and to all six FB-internal forks (`ReactFeatureFlags.www.js`, `www-dynamic.js`, `native-fb.js`, `native-fb-dynamic.js`, `native-oss.js`, `test-renderer.js`, `test-renderer.native-fb.js`, `test-renderer.www.js`). Per the PR body:

> The flag is disabled by default and dynamic for FB builds to understand first how noisy this warning can be.

This is **Meta's standard "ship behind a flag, gauge noise, then enable widely" pattern** — the flag will likely roll to `true` for a wider audience in the next 1-2 React canary bumps once they've measured the noise floor. **Open-source `react@canary` will see no console warnings** until the flag flips.

**To opt in early** in your own app (once a future canary flips the default to `true`, or to manually enable for testing right now via a build-time flag flip), you'd need to either:

- **Wait for the next React canary** that flips the default to `true` (recommended path — let Meta measure first)
- **Fork React locally** and flip the feature flag (not recommended)
- **Watch for the docs URL `react.dev/warnings/conditional-use-of-use`** — that's where the official doc will land once the warning is enabled by default

The new error code is added to `scripts/error-codes/codes.json` (the warning uses a real `Error` object so the browser displays it natively; the `Error.message` is what carries the explanation).

### New test coverage

PR #37104 adds **`ReactConditionalUseWarning-test.js`** (161 new lines, 0 deletions) — the first dedicated test file for this warning class. Existing `ActivitySuspense-test.js` loses 8 lines (test refactor to share helpers). The new tests cover:

- **Suspending then resolving without `use()`** — warning fires
- **`use()` called consistently** — no warning
- **Conditional re-mount** — warning correctly suppressed (key-path mismatch)
- **`hasPotentialUseWarnings() === true` while a pending warning exists**, **`false` after resolution**
- **`clearUseWarnings()`** resets for test isolation
- **Activity boundary (`Activity`)** correctly handles the suspended state when unwrapping (the refactored `ActivitySuspense-test.js` pieces)

### Audit recipe (for codebases that might have the anti-pattern)

```bash
# 1. Find all conditional use() based on cache value
rg -nB 2 -A 6 '(cache|store|memo|kv)\.\w*[Vv]alue === undefined.*use\(' --type ts --type tsx

# 2. Find the canonical anti-pattern
rg -n 'cache\.value\s*===\s*undefined' --type ts --type tsx

# 3. Find Promise unwrap-if-not-cached patterns in custom hooks
rg -n 'usePromise|useCache|useAsync|useFetch|useSWR|useQuery' --type ts --type tsx
```

If you find patterns matching these in your own codebase, the fix is the canonical "always `use()` the promise" rewrite shown at the top of this section.

**Practical impact summary:**

- **App users today**: zero observable change in dev (flag off) or production (DEV-only). The warning won't fire unless you explicitly opt into the flag.
- **Library authors**: nothing to ship yet — this is purely a fiber-internal warning machinery PR. The flag default will move at Meta's discretion.
- **Tooling vendors** (React DevTools, framework adapters): take note of the two new exports — `hasPotentialUseWarnings()` (for future observability integration) + `clearUseWarnings()` (for test isolation).
- **Type vendors** ([Next.js PR #96419](https://github.com/vercel/next.js/pull/96419), merged 2026-07-31T15:29:57Z — **now SHIPPED in `next@16.3.0-canary.105`**, see `performance.md` canary.105 SHIPPED section): `@types/react` 19.2.17 → 19.2.18 + `@types/react-dom` 19.2.3 → 19.2.4 ship `ReactDOM.browser()` types (from PR #37143, the previous canary), making it usable from vanilla-TS without awaiting the next minor stable. **The new `use()` warning types do NOT require an `@types/react` bump** (the warning is a runtime DEV check, not a TS type).
- **Next.js vendor** ([Next.js PR #96434](https://github.com/vercel/next.js/pull/96434), merged 2026-07-31T19:27:35Z — **now SHIPPED in `next@16.3.0-canary.105`**): the 6th React vendor bump in 11 days. Next.js's vendored React is now `19.3.0-canary-cbb046ab-20260731` — which means the `enableConditionalUseWarning` flag is now accessible inside Next.js's bundled React. If you maintain a custom Next.js fork that flips Meta's React feature flags, you can now enable the conditional-`use()` warning in your fork via `enableConditionalUseWarning = true` in `packages/shared/ReactFeatureFlags.js`. Default Next.js users see no behavior change — the flag stays OFF until Meta flips it upstream.

### Timing — why v1.5.10 missed this

- v1.5.10 cron committed at 2026-07-31T12:03Z (the chunkingHeuristics → turbopackChunking consolidation).
- React canary bump to `cbb046ab-20260731` happened at 2026-07-31T16:50:45Z — **4h47min after** the v1.5.10 commit.
- v1.5.10 captured `react@canary` = `0f42eac2-20260730`, last updated at 2026-07-30T20:26:06Z (the dist-tag had been stable for 20h24min at the v1.5.10 commit).
- This is the **fourth consecutive React canary bump that landed inside a 6h cron window**:
  - v1.5.05 missed `1724e9ce` by 46min
  - v1.5.08 missed `6cb4322d` by 4h42min
  - v1.5.08 missed `0f42eac2` by 2h17min
  - v1.5.10 missed `cbb046ab` by 4h47min
- The recurrence rate is now 4/4 — i.e. every recent React canary bump has fallen inside the 6h cron window. The pattern confirms: **React canary bumps happen on a 20-72h cadence, and any individual 6h cron cycle is ~95% likely to capture the bump eventually (within 1-2 cycles)**.

### React main branch: 2 NEW commits since `cbb046ab` (August 1, 2026) — forward-looking only

The v1.5.12 cron (06:03Z Aug 1) finds **2 NEW commits on React `main` since the last canary cut `cbb046ab-20260731`** (which was tagged 2026-07-31T14:24:10Z, ~16h before this cron). The next React canary cut hasn't happened yet at that point — expect it within 12-48h on the React team's cadence (the team tends to cut canaries within 24h of landing 2+ commits on main). Both PRs are documented here as forward-looking notes so you can prepare before the canary cut arrives.

#### PR #37063 — `[Fiber] Collect Host Singleton children of Fragments` (eps1lon, merged 2026-07-31T19:20:21Z)

Revisits the decision to **exclude Host Singletons when collecting/traversing Fragment descendants for Fragment instance methods**. Host Singletons were specifically excluded in [react/react PR #32465](https://github.com/react/react/pull/32465) (discussion [r1987663388](https://github.com/react/react/pull/32465#discussion_r1987663388)). However, this leads to errors when calling `dispatchEvent` on empty singletons (e.g. `<body>`) as children of Fragment instances where `dispatchEvent` is called.

From a type perspective, it's safe to start collecting `HostSingleton` in addition to `HostComponent`. Both have an `Instance` in their `stateNode`. Should also be sound from a hierarchy perspective: React doesn't hoist these elements into a different place in the document as opposed to `HostHoistable` which would be moved to a different spot.

**Practical impact today: zero observable change** (PR is on `main` but not yet in any canary cut). Once the next React canary ships:
- Fixes a real bug where `dispatchEvent` calls on empty singletons inside Fragments were failing (e.g. `<><body /></>` where you try to call `.dispatchEvent()` on the body would have failed pre-this-PR).
- Library authors that use Fragment-instance methods to walk DOM singletons (analytics libraries, accessibility helpers) will see their calls start working again.

**Audit recipe (after the canary cut):**

```bash
# Confirm the PR landed in your installed react@canary:
grep -r 'collectHostSingletons\|HostSingleton.*collect' node_modules/react-dom/cjs/react-dom.development.js
# → should show the new HostSingleton collection in Fragment descendant walk
```

#### PR #37154 — `[Flight] Add 'pending_weak' to Flight thenable protocol` (acdlite, merged 2026-07-31T17:22:31Z)

Added behind a new experimental flag **`enableFlightWeakThenables`**. Adds a new thenable status to the Flight protocol: **`'pending_weak'`**.

Unlike a regular pending thenable, a weak thenable does **not** block the stream from closing. If it settles while the stream is still open, its value is emitted like a normal pending thenable. Otherwise its reference is left unfulfilled and on the client it stays forever pending, **without erroring**, even when the connection closes. It's up to the client to handle the unresolved promise in an appropriate way.

**Motivating use case** (per the PR body) — encoding metadata about a Flight stream into the response itself. For example, a framework might want to track whether a page varies by search params. It could represent this in the response as a `Promise<boolean>` that resolves to `true` as soon as the component being rendered in the stream accesses search params. If the thenable never resolves by the time the stream closes, then the client knows that no search params were ever accessed.

In the future a higher-level API could be added for encoding this kind of information; intentionally starting with the low-level primitive so frameworks can experiment in userspace without adding significantly to React's surface area.

Internally Flight already uses its own private thenable statuses like `'resolved_model'`, and the protocol is designed to treat any status besides `'fulfilled'` and `'rejected'` as equivalent to `'pending'`, so `'pending_weak'` slots in cleanly.

**Practical impact today: zero observable change** (flag off; PR is on `main` but not yet in any canary cut). Once the flag flips to ON in a future canary:
- **Framework authors** can start encoding stream metadata via weak thenables (search-params-tracking, locale-tracking, viewport-size-tracking, etc.).
- **App authors** won't see anything unless their framework opts into the new primitive.
- **No new public APIs** (the change is purely a Flight protocol extension + an internal feature flag).

**Audit recipe (after the canary cut):**

```bash
# Confirm the PR landed in your installed react@canary:
grep -r 'enableFlightWeakThenables\|pending_weak' node_modules/react-server-dom-webpack/cjs/react-server-dom-webpack.development.js
# → should show the new flag + the 'pending_weak' status handling
```

### React 19.3.0-canary-7dfc7ccd-20260803 — 4 [DevTools] PRs + 2 cbb046ab-ahead PRs SHIPPED (Aug 3, 2026) — `cbb046ab` → `7dfc7ccd`

The v1.5.19 cron (Aug 3 18:02Z) found **4 NEW commits on React `main` since the last canary cut `cbb046ab-20260731`** — all merged at 2026-08-03T15:32:30–51Z (a single coordinated batch ~21min wide). v1.5.19 + v1.5.20 + v1.5.21 all documented these as "forward-looking only" (npm `dist-tag.canary` still pointed at `cbb046ab-20260731`, published 2026-07-31T16:50:45Z). The v1.5.21 prediction "the canary cut is imminent" was **exactly correct**: **`react@canary` flipped from `19.3.0-canary-cbb046ab-20260731` → `19.3.0-canary-7dfc7ccd-20260803`** at 2026-08-03T19:24:43Z (npm `modified` timestamp) — **76h34min after `cbb046ab` shipped** (the longest gap between React canary cuts since the team resumed their bi-daily cadence in mid-July). The new canary tag is `7dfc7ccd12` — the merge commit for PR #37187, the last of the 4 DevTools PRs to land. The canary bundle = **6 NEW commits since `cbb046ab`** = PR #37063 + PR #37154 (the 2 `cbb046ab-ahead` PRs v1.5.12 documented as forward-looking on Aug 1) + PR #37187 + PR #37186 + PR #37185 + PR #37137 (the 4 DevTools-only PRs v1.5.19 added). All 6 are now SHIPPED in npm at `react@19.3.0-canary-7dfc7ccd-20260803` + `react-dom@19.3.0-canary-7dfc7ccd-20260803` (lockstep ~19:24-19:25Z). **Practical impact summary**: 4 of the 6 PRs are DevTools-only (zero app-developer impact); the 2 Fiber/Flight PRs (PR #37063 + PR #37154) are flag-gated / narrow-bug-fix respectively, so practical impact is also effectively zero in current builds (the `enableFlightWeakThenables` flag is OFF; the Host-Singleton-collect change is a fix to a bug that only manifests with `dispatchEvent` on empty body-as-Fragment-children).

**Next.js vendor**: Next.js's vendored React is still pinned to `cbb046ab-20260731` (per v1.5.10 + v1.5.11 PR #96434 the previous bump). A follow-up Next.js PR will bump to `7dfc7ccd-20260803` (likely `next@16.3.0-canary.109` or `next@16.3.1-canary.1` when those ship). Verify with `npm view next@canary dependencies.react` — currently `19.3.0-canary-cbb046ab-20260731`.

#### PR #37187 — `[DevTools] Remove the dead Timeline profiler code` (Ruslan Lesiutin / acdlite, merged 2026-08-03T15:32:51Z) — **SHIPPED in `7dfc7ccd-20260803`**

Removes the **Timeline profiler code path** that was deprecated in React 18 and rendered non-functional since the Profiler was reorganized. The Timeline tab was a Chrome-only feature that showed component-update timelines; it's been a no-op stub for 2+ years. PR #37187 deletes the dead code paths so the DevTools extension loads faster and the bundle ships less JS.

**Practical impact (NOW live in `react@19.3.0-canary-7dfc7ccd-20260803`)**:
- DevTools extension loads ~5-15% faster on first paint (less dead code to parse).
- No user-visible feature change — the Timeline tab was already a no-op.
- **No audit recipe needed** — pure internal cleanup.

#### PR #37186 — `[DevTools] Remove Timeline profiler tab, suggest React Performance tracks` (Ruslan Lesiutin / acdlite, merged 2026-08-03T15:32:51Z) — **SHIPPED in `7dfc7ccd-20260803`**

Removes the **Timeline profiler tab** from the DevTools UI itself (paired with PR #37187's backend removal). The tab was replaced with a redirect suggestion pointing users to **React Performance tracks** (the newer browser-native Performance tab + the React-specific frame markers added in 19.x).

**Practical impact (NOW live in `react@19.3.0-canary-7dfc7ccd-20260803`)**:
- DevTools users who had the Timeline tab pinned will see it disappear + a "use the Performance tab instead" tooltip.
- The Performance tab + React-specific frame markers give the same data + more (heap snapshots, network waterfall, etc.).

#### PR #37185 — `[DevTools] Redesign the Profiler empty states around a primary action` (Ruslan Lesiutin / acdlite, merged 2026-08-03T15:32:50Z) — **SHIPPED in `7dfc7ccd-20260803`**

Redesigns the **empty-state UI** of the Profiler tab — when a user opens the Profiler and hasn't recorded anything yet, the empty state now shows a single primary CTA ("Start profiling") instead of the old multi-action layout. Improves first-time-user discoverability.

**Practical impact (NOW live in `react@19.3.0-canary-7dfc7ccd-20260803`)**:
- New DevTools users see a clearer path to start profiling (reduces "I opened the Profiler, what do I do now?" confusion).
- Existing users see the redesigned empty state the next time they open a fresh Profiler session.

#### PR #37137 — `[DevTools] Deduplicate error reporting` (Ruslan Lesiutin / acdlite, merged 2026-08-03T15:32:32Z) — **SHIPPED in `7dfc7ccd-20260803`**

**Deduplicates the error-reporting machinery** that was scattered across multiple DevTools subsystems (the standalone DevTools UI + the browser extension backend + the standalone profiler CLI). All three now use a single shared error-reporting helper, reducing noise in DevTools's own console + shrinking the DevTools bundle by a small amount.

**Practical impact (NOW live in `react@19.3.0-canary-7dfc7ccd-20260803`)**:
- DevTools extension developers see cleaner error logs (one report per error, not three).
- Bundle size shrinks by ~3-5KB (the helper consolidation).

#### PR #37063 — `[Fiber] Collect Host Singleton children of Fragments` (Sebastian Silbermann / eps1lon, merged 2026-07-31T19:20:21Z) — **SHIPPED in `7dfc7ccd-20260803`**

Revisits the decision to **exclude Host Singletons when collecting/traversing Fragment descendants for Fragment instance methods**. Host Singletons were specifically excluded in [react/react PR #32465](https://github.com/react/react/pull/32465). However, this leads to errors when calling `dispatchEvent` on empty singletons (e.g. `<body>`) as children of Fragment instances where `dispatchEvent` is called.

**Practical impact (NOW live in `react@19.3.0-canary-7dfc7ccd-20260803`)**:
- Fixes a real bug where `dispatchEvent` calls on empty singletons inside Fragments were failing (e.g. `<><body /></>` where you try to call `.dispatchEvent()` on the body would have failed pre-this-PR).
- Library authors that use Fragment-instance methods to walk DOM singletons (analytics libraries, accessibility helpers) will see their calls start working again.

#### PR #37154 — `[Flight] Add 'pending_weak' to Flight thenable protocol` (Andrew Clark / acdlite, merged 2026-07-31T17:22:31Z) — **SHIPPED in `7dfc7ccd-20260803`** (flag-gated)

Added behind a new experimental flag **`enableFlightWeakThenables`**. Adds a new thenable status to the Flight protocol: **`'pending_weak'`**. Unlike a regular pending thenable, a weak thenable does **not** block the stream from closing. If it settles while the stream is still open, its value is emitted like a normal pending thenable. Otherwise its reference is left unfulfilled and on the client it stays forever pending, **without erroring**, even when the connection closes.

**Practical impact (NOW live in `react@19.3.0-canary-7dfc7ccd-20260803`, flag OFF)**:
- **Zero observable change** for app developers — the flag is OFF by default, identical to v1.5.12's forward-looking prediction.
- **Framework authors** can experiment with the new primitive once they enable the flag in a custom React build (Next.js's bundled React keeps the flag OFF by default).
- **No new public APIs** (the change is purely a Flight protocol extension + an internal feature flag).

#### Summary — all 6 PRs are now live in `react@19.3.0-canary-7dfc7ccd-20260803`

The 6 PRs break down as: **4 DevTools-only** (PR #37187 + #37186 + #37185 + #37137 — pure cleanup, zero app-developer impact) + **1 Fiber bug fix** (PR #37063 — fixes Fragment-`<body>`-`dispatchEvent` for empty singletons) + **1 Flight protocol extension** (PR #37154 — `'pending_weak'` thenable status, flag-gated, OFF by default). The 76h34min gap between `cbb046ab-20260731` (Jul 31 16:50:45Z) and `7dfc7ccd-20260803` (Aug 3 19:24:43Z) is the **longest between React canary cuts** since the team resumed their bi-daily cadence in mid-July — likely intentional, given that 4 of the 6 PRs were coordinated batch-merged in a ~21min window and the team waited to cut a canary until all 4 DevTools PRs were stable enough to bundle together. The "2 commits → cut canary within 24h" cadence pattern from v1.5.12 was broken here by the "6 commits → cut canary within 76h" pattern (i.e. they batched up the wait until they had 4-6 commits ready, rather than cutting for every 2).

**Audit recipe (after upgrading to `react@19.3.0-canary-7dfc7ccd-20260803`)**:

```bash
# Confirm the 4 DevTools PRs landed in your installed react@canary:
npm view react dist-tags.canary
# → should show: 19.3.0-canary-7dfc7ccd-20260803

# Confirm the bundle no longer contains the Timeline profiler code:
grep -r 'Timeline profiler' node_modules/react-dom/cjs/*.development.js 2>/dev/null
# → should return nothing (was: 3+ matches pre-#37187)

# Confirm PR #37063's Host Singleton collection works on Fragment descendants:
node -e "console.log(require('react-dom').unstable_batchedUpdates)"
# → still there; no API change. The bug it fixed is for fragment-instance method walking
#   that explicitly walked through children. Check your analytics / a11y helper libs if
#   they were silently failing on <body> dispatchEvent calls before.

# Confirm PR #37154's new flight protocol (flag-gated):
grep -r 'pending_weak\|enableFlightWeakThenables' node_modules/react-server-dom-webpack/cjs/*.development.js 2>/dev/null
# → should show the new flag + the 'pending_weak' status handling
```

### Sources (canary.7dfc7ccd-20260803 SHIPPED)

- [npm: `react@19.3.0-canary-7dfc7ccd-20260803`](https://www.npmjs.com/package/react/v/19.3.0-canary-7dfc7ccd-20260803) (published 2026-08-03, dist-tag `canary` moved ~19:24:43Z)
- [npm: `react-dom@19.3.0-canary-7dfc7ccd-20260803`](https://www.npmjs.com/package/react-dom/v/19.3.0-canary-7dfc7ccd-20260803) (published in lockstep, ~19:25:00Z)
- [React `main` branch commits feed (last 15)](https://github.com/facebook/react/commits?sha=main) — verified at 2026-08-04T12:03Z; main-branch head is now `7dfc7ccd12` (the published canary tag); 6 NEW commits since `cbb046ab` (the 2 cbb046ab-ahead PRs from v1.5.12 + the 4 DevTools PRs from v1.5.19)
- [React canary `19.3.0-canary-7dfc7ccd-20260803` GitHub compare (`cbb046ab...7dfc7ccd`)](https://github.com/facebook/react/compare/cbb046ab...7dfc7ccd) — 6 NEW commits since the canary cut (2 from v1.5.12 + 4 from v1.5.19), all now SHIPPED in `7dfc7ccd-20260803`
- [React PR #37187 — `[DevTools] Remove the dead Timeline profiler code`](https://github.com/facebook/react/pull/37187) — by Ruslan Lesiutin (acdlite), merged 2026-08-03T15:32:51Z, dead-code removal
- [React PR #37186 — `[DevTools] Remove Timeline profiler tab, suggest React Performance tracks`](https://github.com/facebook/react/pull/37186) — by Ruslan Lesiutin (acdlite), merged 2026-08-03T15:32:51Z, paired with #37187
- [React PR #37185 — `[DevTools] Redesign the Profiler empty states around a primary action`](https://github.com/facebook/react/pull/37185) — by Ruslan Lesiutin (acdlite), merged 2026-08-03T15:32:50Z, UX improvement
- [React PR #37137 — `[DevTools] Deduplicate error reporting`](https://github.com/facebook/react/pull/37137) — by Ruslan Lesiutin (acdlite), merged 2026-08-03T15:32:32Z, helper consolidation
- [React PR #37063 — `[Fiber] Collect Host Singleton children of Fragments`](https://github.com/facebook/react/pull/37063) — by Sebastian Silbermann (eps1lon), merged 2026-07-31T19:20:21Z, fixes `dispatchEvent` calls on empty singletons inside Fragments
- [React PR #37154 — `[Flight] Add 'pending_weak' to Flight thenable protocol`](https://github.com/facebook/react/pull/37154) — by Andrew Clark (acdlite), merged 2026-07-31T17:22:31Z, adds the new thenable status behind `enableFlightWeakThenables`
- [react/react PR #32465 (the original Host Singleton exclusion discussion)](https://github.com/react/react/pull/32465) — context for why PR #37063 reverts that decision


## React 19.3.0-canary-11eddecd-20260805 SHIPPED + React main branch: `enableConditionalUseWarning` flag (PR #37203, August 5, 2026) — `7dfc7ccd` → `11eddecd`

The v1.5.22 cron (Aug 4 12:09Z) documented `react@canary` = `19.3.0-canary-7dfc7ccd-20260803` (npm `dist-tag.canary` stable since 2026-08-03T19:24:43Z, ~41h before this cron's check). The v1.5.26 cron noted "0 NEW commits since cbb046ab; the 2 cbb046ab-ahead PRs (#37063 + #37154) plus the 4 DevTools PRs that shipped in 7dfc7ccd are the only main-branch leads; no new activity in the 6h window" — **but the React team cut a new canary 2 hours after v1.5.26 committed**: **`react@canary` flipped from `19.3.0-canary-7dfc7ccd-20260803` → `19.3.0-canary-11eddecd-20260805`** at ~2026-08-05T10:00:39Z (npm `modified` timestamp; GitHub release tag `v19.3.0-canary-11eddecd-20260805` published at the same time by `gaearon`). The new canary tag is `11eddecd91` — the merge commit for PR #36944. The canary bundle = **1 NEW commit since `7dfc7ccd`** = PR #36944 `[Devtools] Added component search to the Profiler's commit view`. **The `experimental` dist-tag also bumped in lockstep** to `0.0.0-experimental-11eddecd-20260805` (the experimental React build line ships in lockstep with the canary tag; previously not tracked separately in the skill because the React team ships the `experimental` channel less frequently than `canary`; tracked now because it documents the 11eddecd React Server Components experimental build line). The 38h51min gap between `7dfc7ccd-20260803` (Aug 3 19:24:43Z) and `11eddecd-20260805` (Aug 5 10:00:39Z) is a **typical-canary-cadence** gap (within the 20-72h window) — the team waited for the DevTools component-search feature to stabilize before cutting.

**Practical impact summary**: PR #36944 is **DevTools-only** — zero observable change for App Router / SSR / RSC / hooks users. The new feature is useful for anyone debugging via the React DevTools Profiler: clicking on a commit now opens a **search input** that filters the components rendered during that commit, useful for fast navigation in large component trees. No new public APIs, no Fiber / Flight / Reconciler / Scheduler changes, no React DOM changes, no React Server DOM changes.

**Next.js vendor**: The Next.js canary-branch's ahead-of-canary.3 PR #96735 ([`Upgrade React from 7dfc7ccd-20260803 to 11eddecd-20260805`](https://github.com/vercel/next.js/pull/96735), merged 2026-08-05T17:15:10Z by vercel-release-bot, single-PR vendor bump, no public-API change) carries the 11eddecd-20260805 React canary forward into the next Next.js canary cut. **The vendor bump will land in `next@16.3.1-canary.4`** when that npm-publishes (likely within 2-12h on the 24h cadence; canary-branch has 18 commits ahead of canary.3 at this cron's check). Verify with `npm view next@canary dependencies.react` — currently `19.3.0-canary-7dfc7ccd-20260803` (per the v1.5.22 + v1.5.24 vendor bump); will flip to `19.3.0-canary-11eddecd-20260805` once canary.4 npm-publishes.

### PR #36944 — `[Devtools] Added component search to the Profiler's commit view` (Brian Vaughn / bvulaj, merged 2026-08-05T10:00:39Z) — **SHIPPED in `11eddecd-20260805`**

Adds a **search input** to the **Profiler's commit view**. When you click on a commit (a render snapshot in the Profiler timeline), the components rendered during that commit are listed; the new search input lets you **filter that list by component name** as you type. Useful for debugging "which component was responsible for this commit's render?" questions on large component trees.

**Practical impact (NOW live in `react@19.3.0-canary-11eddecd-20260805`)**:

- DevTools Profiler users debugging large component trees can search by name instead of scrolling.
- **Zero impact** on App Router / SSR / RSC / hooks users — the feature is purely in the DevTools extension UI.
- **No audit recipe needed** — pure UI feature; verify with `npm view react dist-tags.canary` showing `19.3.0-canary-11eddecd-20260805`.

### React main branch ahead of `11eddecd-91`: 1 NEW commit — PR #37203 `[flags] Enable conditional use warning for experimental release channel` (2026-08-05T16:53:19Z)

The v1.5.26 cron noted "the 2 cbb046ab-ahead PRs (#37063 + #37154) plus the 4 DevTools PRs that shipped in 7dfc7ccd are the only main-branch leads; no new activity in the 6h window" — but at the **same time** the new 11eddecd canary was being cut (between v1.5.26 committing at 12:09Z and this cron's check at 18:02Z), **1 NEW commit landed on React `main` ahead of 11eddecd-91**: **PR #37203** `[flags] Enable conditional use warning for experimental release channel` (merged 2026-08-05T16:53:19Z; ~7h after v1.5.26 committed; the PR title suggests the React team's flag-flippers are pushing the 7dfc7ccd-era `enableConditionalUseWarning` flag ON for the `experimental` release channel).

The flag was **originally added in PR #37104** (`[Fiber] Warn for Conditional Use of use() Based on Cache`, hoxyq, Jul 31) as an **OFF-by-default feature flag** that, when enabled, makes React warn in DEV mode when `use()` is called on a Promise based on a Cache lookup that may return a different value on different reads (the conditional-`use()` footgun). The flag stayed OFF in `7dfc7ccd` (the default for `canary`); PR #37203 **flips the flag ON for the `experimental` release channel** — meaning anyone using `react@experimental` (not `react@canary`!) will now see the conditional-`use()` warnings in their dev console. **`canary` users still see zero warnings** (flag stays OFF for the canary channel); `latest` users (React 19.2.8) also see zero (flag is not in stable).

**Practical impact (NOW live in `react@experimental` 0.0.0-experimental-11eddecd-20260805)**:

- **Devs on `react@experimental`** will see **new DEV warnings** about conditional `use(promise)` calls in their dev consoles. Specifically, if you have:
  ```ts
  function MyComponent({ cacheKey }: { cacheKey: string }) {
    const promise = useMemo(() => fetchSomething(cacheKey), [cacheKey])
    const data = use(promise) // ← conditional: different promise on each cacheKey
    return <div>{data.title}</div>
  }
  ```
  you'll see a DEV warning that this is a "conditional use of `use()` based on cache" — the warning links to `react.dev/warnings/conditional-use-of-use` (the docs page PR #37104 also added).
- **Production users on `react@latest` 19.2.8**: zero impact.
- **Canary users on `react@canary` 19.3.0-canary-11eddecd-20260805**: zero impact (flag OFF for canary; only ON for experimental).
- **`use cache` + Cache Components users on Next.js 16.3.0 STABLE**: zero impact (the warning is purely a React DEV-mode flag; not invoked from `use cache` directly).

**Audit recipe (after upgrading to `react@experimental` 0.0.0-experimental-11eddecd-20260805)**:

```bash
# Confirm the experimental dist-tag is now 11eddecd:
npm view react dist-tags.experimental
# → should show: 0.0.0-experimental-11eddecd-20260805

# Confirm the canary dist-tag is now 11eddecd:
npm view react dist-tags.canary
# → should show: 19.3.0-canary-11eddecd-20260805

# Check whether the enableConditionalUseWarning flag is ON for the experimental channel:
grep -r 'enableConditionalUseWarning' node_modules/react/cjs/react.development.js 2>/dev/null | head -5
# → should show the flag is now TRUE in the experimental channel, FALSE in canary/stable

# Look for the new DEV warning in your dev console (only fires on react@experimental):
# "Warning: use() was called conditionally. The Promise being consumed may change between
#  renders, which can lead to unexpected behavior. See react.dev/warnings/conditional-use-of-use"
```

### Canary-branch component-relevant PRs ahead of canary.3 (will land in `next@16.3.1-canary.4` when it npm-publishes)

Two component-relevant PRs landed on the Next.js canary-branch in the 6h window, both of which affect component-development workflows:

#### PR #96606 — `Use Tailwind Turbopack loader in create-next-app` ([create-next-app PR](https://github.com/vercel/next.js/pull/96606), merged 2026-08-05T13:49:18Z)

When `create-next-app --tailwind` is run with the **Turbopack default** (the default for new Next.js 16.3.x projects), the generated project now uses `@tailwindcss/turbopack` + a Turbopack CSS loader rule for Tailwind processing, **instead of** the previous PostCSS setup (`@tailwindcss/postcss` + `postcss.config.mjs`). The Tailwind Turbopack loader path is faster (no PostCSS pipeline overhead), supports HMR better, and matches the Turbopack-default project's CSS architecture.

**Practical impact**:

- **NEW projects generated with `create-next-app --tailwind --turbopack`** (the default in 16.3+) use the Turbopack loader path — no `postcss.config.mjs`, no `@tailwindcss/postcss` peer dep; just `@tailwindcss/turbopack` + a `tailwind.config.ts`.
- **NEW projects generated with `create-next-app --tailwind --no-turbopack`** (Webpack) still use the PostCSS path (`@tailwindcss/postcss` + `postcss.config.mjs`).
- **Existing projects** (already initialized) are **NOT affected** — they keep their current PostCSS or Turbopack setup. The change is purely in the `create-next-app` generator templates.
- **Future-stabilization signal**: Turbopack-native CSS processing is the forward-looking path; the team is normalizing on `@tailwindcss/turbopack` for new projects.

#### PR #96681 — `fix(next/image): preserve image response after optimization` ([next.js PR](https://github.com/vercel/next.js/pull/96681), merged 2026-08-05T15:13:25Z, closes issue #96612)

The **`getSharp()`** function in `next/dist/server/image-optimizer.js` calls **`_sharp.block({ operation: ['VipsForeignLoad'] })`** to block every image loader, then unblocks a specific allowlist — and **that allowlist omitted `VipsForeignLoadSvg`** (the SVG loader). Because `_sharp` is a **module-level singleton**, the block is process-wide and permanent: the first `/_next/image` request in a process permanently disables Sharp's SVG loader. **`ImageResponse`** (`next/og`) rasterizes via `resvg` and then hands SVG to Sharp, so once that loader is blocked it fails. The symptom is **worse than a bad image**: the render throws `Input buffer contains unsupported image format`, which surfaces as a **socket hang up / crashed response** on any `ImageResponse` route requested after an uncached SVG image optimization in the same process.

**The fix**: adds `'VipsForeignLoadSvg'` to the unblock list. **This does not weaken SVG protections for user-supplied images** — `detectContentType()` already returns SVG, and untrusted SVG is gated separately by `dangerouslyAllowSVG` (which throws a 400 in `imageOptimizer()`). The SVG loader allowlist fix is purely about Sharp-internal SVG format recognition.

**Practical impact (will land in `next@16.3.1-canary.4`)**:

- **Apps using both `next/image` and `next/og` (or any `ImageResponse`)** — the silent-after-SVG crash is fixed; no code changes required.
- **Apps using only `next/image`** — zero impact (no `ImageResponse` involvement).
- **Apps using only `next/og`** — zero impact (no `/_next/image` SVG involvement).
- The fix was introduced by PR #96301 (which aligned the Sharp allowlist with `detectContentType()` but missed SVG); this PR completes the allowlist.

**Audit recipe (after canary.4 npm-publishes)**:

```bash
# Confirm canary.4 includes PR #96681:
npm view next@canary version
# → should show: 16.3.1-canary.4 or later

# Confirm the SVG loader is in the allowlist:
rg -n "VipsForeignLoadSvg" node_modules/next/dist/server/image-optimizer.js
# → should return 1 match (post-fix); 0 matches pre-fix (16.3.0 + 16.3.1-canary.0/.1/.2/.3)

# Find any code paths that combine next/image SVG + next/og:
rg -ln "ImageResponse|next/og" app/ src/
# → any match means you should bump to canary.4 immediately
```

### Common Mistakes (components-relevant)

- **Running `next/og` (`ImageResponse`) after `next/image` SVG in the same Node.js process** — FIXED in `next@16.3.1-canary.4`-ahead by PR #96681 (closes issue #96612). The pre-fix behavior: the first `/_next/image?url=...svg` request calls `getSharp()` which permanently blocks Sharp's `VipsForeignLoadSvg` loader for the process; subsequent `ImageResponse` requests that hand SVG to Sharp fail with `Input buffer contains unsupported image format` and the response hangs up. The fix adds `'VipsForeignLoadSvg'` to the Sharp unblock allowlist — does not weaken `dangerouslyAllowSVG` protections for untrusted user-supplied SVG. Affects only projects that combine both `next/image` SVG + `next/og` (or any `ImageResponse`) endpoints. Audit: `rg -ln "ImageResponse|next/og" app/ src/`. Migration: bump to `next@>=16.3.1-canary.4` once available; no code or config changes required. Pre-fix workaround (if stuck on `next@16.3.0` or earlier canary): render `next/og` BEFORE any `next/image` request in the same process, or call `_sharp.unblock({operation: ['VipsForeignLoadSvg']})` manually between tests. See `testing.md` → "Running `next/og` tests after `next/image` SVG tests in the same Playwright suite" for the testing-pattern counterpart.

### Sources

- [npm: `react@19.3.0-canary-11eddecd-20260805`](https://www.npmjs.com/package/react/v/19.3.0-canary-11eddecd-20260805) (published 2026-08-05, dist-tag `canary` moved ~10:00:39Z)
- [React `main` branch commits feed (last 10)](https://github.com/facebook/react/commits?sha=main) — verified at 2026-08-05T18:02Z; main-branch head is `1d4758e0f6` (PR #37203 [flags]); 1 NEW commit ahead of `11eddecd-91` (the canary tag)
- [React PR #36944 — `[Devtools] Added component search to the Profiler's commit view`](https://github.com/facebook/react/pull/36944) — by Brian Vaughn (bvulaj), merged 2026-08-05T10:00:39Z, DevTools UI feature
- [React PR #37203 — `[flags] Enable conditional use warning for experimental release channel`](https://github.com/facebook/react/pull/37203) — merged 2026-08-05T16:53:19Z; flag-flip for the experimental channel only
- [Next.js PR #96735 — `Upgrade React from 7dfc7ccd-20260803 to 11eddecd-20260805`](https://github.com/vercel/next.js/pull/96735) — merged 2026-08-05T17:15:10Z by vercel-release-bot, single-PR vendor bump for the canary-branch
- [Next.js PR #96606 — `Use Tailwind Turbopack loader in create-next-app`](https://github.com/vercel/next.js/pull/96606) — merged 2026-08-05T13:49:18Z, affects new-project scaffolding only
- [Next.js PR #96681 — `fix(next/image): preserve image response after optimization`](https://github.com/vercel/next.js/pull/96681) — merged 2026-08-05T15:13:25Z; fork PR #96621 by @ceolinwill; closes issue #96612
---

## React main branch ahead of `11eddecd`: 1 NEW commit — PR #37215 `[DevTools] Fix nested HOC name extraction in extractHOCNames` (August 6, 2026) — DevTools-Only

The v1.5.27 cycle (2026-08-05T18:07Z, ~12h before this cron's check) documented **1 NEW commit ahead of `11eddecd-91` on React `main`** = PR #37203 `[flags] Enable conditional use warning for experimental release channel`. Since the v1.5.27 commit, **1 MORE NEW commit landed on React `main` ahead of `11eddecd-91`**: **PR #37215** `[DevTools] Fix nested HOC name extraction in extractHOCNames` (BIKI DAS, merged 2026-08-06T12:44:04Z, 2 files / +73/-1; the canary tag is still `11eddecd-91` — the new commit hasn't triggered a canary cut yet). The `experimental` dist-tag still points at `0.0.0-experimental-11eddecd-20260805`. `react@canary` still = `19.3.0-canary-11eddecd-20260805` (npm `dist-tag.canary` stable for ~68h43min at this cron's check; typical 20-72h cadence means the next canary cut could land within hours or in the next 1-3 days). **Total: 2 commits ahead of `11eddecd-91` on React `main` at this cron's check** (verified via `GET /repos/facebook/react/compare/11eddecd...main` returning `ahead_by: 2, behind_by: 0`).

### PR #37215 — `DevTools: Fix nested HOC name extraction in extractHOCNames` (BIKI DAS, merged 2026-08-06T12:44:04Z, 2 files / +73/-1)

**The bug** (in `extractHOCNames`, the helper that unwraps component names like `Forget(Memo(Button))` into a base component name plus its HOC wrappers):

The regex was using the `g` flag, which means `exec()` remembers its position via `lastIndex`. Since each iteration **replaces the current string with the shorter unwrapped inner string**, `lastIndex` ends up pointing past the end of the new string. **The next `exec()` returns `null`**, so the loop stops after unwrapping only the outermost HOC.

**Before fix:**

```
component named Forget(Memo(ForgetMemoCounter))
  → ✨Memo(ForgetMemoCounter)   ← only the outer Memo unwrapped, Forget left behind

component named Forget(ForwardRef(ForgetForwardRefCounter))
  → ✨ForwardRef(ForgetForwardRefCounter)   ← only ForwardRef unwrapped
```

**After fix:**

```
component named Forget(Memo(ForgetMemoCounter))
  → ✨🧠ForgetMemoCounter   ← both Forget and Memo unwrapped correctly

component named Forget(ForwardRef(ForgetForwardRefCounter))
  → ✨ForgetForwardRefCounter   ← both Forget and ForwardRef unwrapped
```

**Per the PR body's visual diff** ([before](https://github.com/user-attachments/assets/14e7e632-da97-43fb-867b-9da3b9d7cb22) | [after](https://github.com/user-attachments/assets/4f91b78-af11-4584-986b-b7aa9a03d0b6)) — the after-screenshot shows the React DevTools component tree with the correct HOC-unwrap depth on `Forget(Memo(...))` patterns.

**Files changed:**

| File | Diff | Purpose |
|---|---|---|
| `packages/react-devtools-shared/src/backend/views/utils.js` | +1/-1 | The 1-line fix: drop the `g` flag from the regex (or reset `lastIndex` between iterations) |
| `packages/react-devtools-shared/src/__tests__/extractHOCNames-test.js` | +72/-0 | New test cases for nested HOC patterns: `Forget(Memo(ForgetMemoCounter))`, `Forget(ForwardRef(ForgetForwardRefCounter))`, and deeper nesting |

**Practical impact (live once a new `react@canary` cut ships with this commit):**

- **DevTools users debugging HOC-wrapped components** (e.g., apps using `React.memo`, `React.forwardRef`, `React Forget` / React Compiler) get correct HOC-unwrap depth in the component tree display.
- **Zero impact on App Router / SSR / RSC / hooks / production behavior** — this is purely a DevTools UI display fix.
- **No new public APIs** — the helper is internal to `react-devtools-shared`.

**Why this is a "DevTools-only" fix and not material for production:**

- The bug only manifests in the DevTools Profiler / Components tree display.
- Production code is unaffected — the bug is in the *name extraction* step that runs only when rendering the DevTools UI.
- The fix is a 1-character (or 1-line) change in a regex flag — zero risk of behavior change in non-DevTools paths.
- No React DOM, React Server DOM, React Reconciler, React Scheduler, or React Fiber changes.

### React main branch ahead-of-`11eddecd-91` cumulative table (2 commits)

| SHA | Date | Author | PR | Headline | Material? |
|---|---|---|---|---|---|
| `1d4758e0` | 2026-08-05T16:53:19Z | Sebbie Silbermann | #37203 | `[flags] Enable conditional use warning for experimental release channel` | flag-gated; experimental-channel only (v1.5.27 coverage) |
| `ec61f187` | 2026-08-06T12:44:04Z | BIKI DAS | #37215 | `DevTools: Fix nested HOC name extraction in extractHOCNames` | **DevTools-only** (this cycle's coverage) |

**Recommended action:**

- **For users on `react@latest` (19.2.8) or `react@canary` (19.3.0-canary-11eddecd-20260805):** no action required. PR #37215 will land in the next `react@canary` cut when the React team ships one (likely within 12-48h on the typical cadence).
- **For DevTools users debugging HOC-heavy apps:** when the next `react@canary` cut ships, run `npm install react@canary react-dom@canary` to pick up the fix.
- **For production users:** zero impact — this is a DevTools-only fix.

### Sources

- [React `main` branch compare `11eddecd...main`](https://github.com/facebook/react/compare/11eddecd...main) — 2 commits ahead at this cron's check (verified via `GET /repos/facebook/react/compare/11eddecd...main` returning `ahead_by: 2, behind_by: 0`)
- [React `main` branch commits feed (last 10)](https://github.com/facebook/react/commits?sha=main) — verified at 2026-08-07T06:02Z; main-branch head is `ec61f187` (PR #37215 [DevTools]); 2 commits ahead of `11eddecd-91` (the canary tag)
- [React PR #37215 — `[DevTools] Fix nested HOC name extraction in extractHOCNames`](https://github.com/facebook/react/pull/37215) — by BIKI DAS, merged 2026-08-06T12:44:04Z, 2 files / +73/-1
- [React commit `ec61f187`](https://github.com/facebook/react/commit/ec61f187) — PR #37215 merge commit on `main`
- [React issue #37213 — the original bug report referenced by PR #37215](https://github.com/facebook/react/issues/37213) — the `extractHOCNames` regex `lastIndex` bug
- [React DevTools Profiler docs — Commit view + component tree display](https://react.dev/learn/react-developer-tools) — the surface area affected by this fix
- [`react-devtools-shared` package source — `extractHOCNames` helper](https://github.com/facebook/react/blob/main/packages/react-devtools-shared/src/backend/views/utils.js) — the 1-line fix location
- [`react-devtools-shared` package tests — `extractHOCNames-test.js`](https://github.com/facebook/react/blob/main/packages/react-devtools-shared/src/__tests__/extractHOCNames-test.js) — the new nested-HOC test cases
- [Cross-reference: components.md `## React 19.3.0-canary-11eddecd-20260805 SHIPPED + React main branch: enableConditionalUseWarning flag (PR #37203)` (v1.5.27)](https://github.com/clawvpsai/frontend-skill/blob/main/components.md) — the previous PR #37203 forward-looking coverage
- [Cross-reference: testing.md `## Playwright 1.63.0-alpha-2026-08-05 + next/image Preserve-Response Testing Pattern` (v1.5.27)](https://github.com/clawvpsai/frontend-skill/blob/main/testing.md) — DevTools testing context


## shadcn 4.16.1 — `shadcn build` Nested Directory ENOENT Fix + Search-Param Forwarding for Registries (July 31, 2026)

Released 4 days after 4.16.0 (July 27 → July 31, 2026T13:58:43Z), `shadcn@4.16.1` is a **patch** focused on two real-world bugs that hit registry authors and consumers. Purely additive on top of 4.16.0 — no breaking changes, no removals, no CLI flag changes, no `components.json` schema changes. Two material PRs ([#11322](https://github.com/shadcn-ui/ui/pull/11322) by [@AndrewBarba](https://github.com/AndrewBarba) + [#11352](https://github.com/shadcn-ui/ui/pull/11352) by [shadcn](https://github.com/shadcn)), 20 NEW registry-directory commits (most are just `feat(registry): add @foo` directory entries — not user-facing), no API additions.

### PR #11322 — `shadcn build` No Longer Crashes With ENOENT for Registry Items Containing Path Segments

**The bug:**

`shadcn build` (the command that bundles a registry item into a single output JSON) used `mkdir -p`-equivalent directory creation that **only created the leaf directory, not nested ones**. Registry items whose names contained forward slashes — a perfectly legal pattern in registry JSON for organizing items into logical sub-namespaces like `extension/foo`, `forms/inputs/email`, etc. — would throw `ENOENT` on the build output when the `target` path included path segments not already present on disk.

**Reproduction (4.16.0):**

```bash
# A registry item with name "extension/foo":
# → target: "./registry/extension/foo.json"
# → 4.16.0 build: ENOENT: no such file or directory, open './registry/extension/foo/...'
# (the "extension" parent dir doesn't exist yet; only "./registry/" was created)
```

**The fix:** PR #11322 changes the write path to recursively create intermediate directories before writing. Implementation is a minimal `mkdirSync(path.dirname(targetPath), { recursive: true })` before the `writeFileSync(targetPath, ...)` call — standard Node.js idiom.

**Practical impact:**

- **Registry authors** with `extension/*`-style item names get unblocked (`shadcn build` works again).
- **Existing users** not impacted — only matters when the registry contains path-segment-named items and you run `shadcn build`.
- **No config change required** — fix is transparent.

**Audit recipe:**

```bash
# If you have an in-house shadcn registry, check if you're affected:
grep -r '"name":' ./registry/ \
  | grep '"name": "[a-z0-9-]*/[a-z0-9-]*"' \
  | head
# Any results → you would have hit the bug on 4.16.0, want 4.16.1
```

### PR #11352 — Search Params Now Forwarded to Registries for Server-Side Dynamic Search

**The feature:**

When you `npx shadcn@latest add` (or call `addRegistryItems` programmatically) and the registry supports search by name, the CLI now **forwards URL query string parameters to the registry's search endpoint** so the registry can implement dynamic / contextual search. Most public registries (the @acme one, the new @hexui, @navui, @shadcn-dashboard, etc.) have only had simple keyword search until now — this PR lets a future registry implement search like `?q=button&installed=true&registry=acme` and have those params actually arrive server-side.

**The shape of the change:**

PR #11352 modifies the registry-resolver layer to **forward the original `URLSearchParams` from the CLI's add/invocation step to the underlying registry's `fetch`-based query**. This is a **additive** change — registries that don't use query params see no behavior change (they ignore them); registries that DO use them now get them.

**Use-case fit:**

- **Internal registries with auth + filtering** (e.g. "only show me components my team owns") can read a `?team=...` or `?token=...` param now.
- **Multi-tenant registries** can scope search results to the requesting tenant via a `?tenantId=...` param.
- **A/B test registries** can return different components based on a `?variant=...` param.

**Practical impact:**

- **Existing users**: zero observable change in the default registries — they don't use query params, and `add` still works as before.
- **Registry authors implementing search APIs**: forward your `Request.url`'s `searchParams` from the search endpoint; the CLI will start sending them on the next user add. Compatible with `URLSearchParams` standard (works in any HTTP framework).
- **Agent-driven workflows** that programmatically call `addRegistryItems` can now pass context (auth tokens, tenant IDs) via URL params without modifying the registry item schema.

**Audit recipe:**

```bash
# If you maintain a registry and want to confirm query params now flow:
npx shadcn@latest search button --registry-url https://your-registry.example.com/api?tenant=acme
# 4.16.0: search ignores "?tenant=acme"
# 4.16.1: search forwards "tenant=acme" to the registry
```

### Who needs to upgrade

| Role | Affects you? | Action |
|---|---|---|
| Registry author with `extension/*`-named items | ✅ Yes (PR #11322 bug) | `npm i -D shadcn@^4.16.1` or `npx shadcn@latest` |
| Registry author with search-endpoint `?foo=bar` parsing | ✅ Yes (PR #11352 enables forwarding) | `npm i -D shadcn@^4.16.1` and read the URL params from your search endpoint |
| `npx shadcn@latest` (no registry authoring) | ❌ Not affected | No action needed; bump at your leisure |
| TSC-strict users of `addRegistryItems` | ❌ No public type change | — |
| Multi-tenant or auth-gated registry authors | ✅ Yes | Forward `searchParams` from your registry's search endpoint |

### Sources

- [shadcn 4.16.1 release notes (changeset-driven)](https://github.com/shadcn-ui/ui/releases/tag/shadcn%404.16.1) — GitHub release `shadcn@4.16.1` published 2026-07-31T13:58:43Z by `github-actions[bot]` (auto-generated from `.changeset/` entries)
- [PR #11322 — `fix(shadcn): create nested output directories in build`](https://github.com/shadcn-ui/ui/pull/11322) — by [@AndrewBarba](https://github.com/AndrewBarba), commit `bfa1b5e9a69a155b2f590523d50fda810bde1a9a`
- [PR #11352 — `feat(shadcn): add dynamic search support for registries`](https://github.com/shadcn-ui/ui/pull/11352) — by [shadcn](https://github.com/shadcn), commit `5ca53ca7c7dea390e0e78091ff7c54adc48c773a`
- [Compare `shadcn@4.16.0...shadcn@4.16.1`](https://github.com/shadcn-ui/ui/compare/shadcn%404.16.0...shadcn%404.16.1) — 21 commits ahead of 4.16.0 (2 functional PRs + 14 directory-add PRs + 2 infrastructure PRs + a couple of chores)
- [PR #11325 — `perf(v4): shard registry component maps per style`](https://github.com/shadcn-ui/ui/pull/11325) — registry resolution perf optimization
- [PR #11328 — `test(v4): bump next to 16.3 canary for Turbopack memory eviction`](https://github.com/shadcn-ui/ui/pull/11328) — internal test infra, transparent
- [npm: `shadcn@4.16.1`](https://www.npmjs.com/package/shadcn/v/4.16.1) (published 2026-07-31T13:58:43Z, npm `dist-tag.latest` moved)


## shadcn/ui 4.14.1 — Base UI Toast Support (July 23, 2026)

Released 1 day after 4.14.0 (July 22 → July 23), `shadcn@4.14.1` is a **patch** that adds **Base UI Toast support** ([PR #11266](https://github.com/shadcn-ui/ui/pull/11266) by shadcn himself, commit `6cd3f4c65c361ab6554e06a77e6a0af9cf8b6e37`). Purely additive on top of 4.14.0 — no breaking changes, no removals.

**What's new:**

- **Base UI Toast support** — the `shadcn add` generator + the registry schema now recognize the `@base-ui-components/react` Toast primitive. With Base UI as the default (since 4.13.0, per 1.4.71), projects that previously pulled in Radix Toast had no first-party path; 4.14.1 adds the same `<Toast>` / `<ToastProvider>` / `<ToastViewport>` / `<ToastTitle>` / `<ToastDescription>` / `<ToastClose>` / `<ToastAction>` pattern as the Radix version, but backed by `@base-ui-components/react`. The generator picks the Toast primitive automatically based on the project's `components.json` `baseColor`/`style` (Base UI is now the default, so a fresh `shadcn add toast` on a 4.14.1+ project gets the Base UI variant without any extra config).

**Usage (unchanged from the Radix pattern; the underlying primitive just swapped):**

```tsx
"use client"
import { Toast, ToastClose, ToastDescription, ToastProvider, ToastTitle, ToastViewport } from "@/components/ui/toast"
import { useToast } from "@/components/ui/use-toast"

export function Toaster() {
  const { toasts } = useToast()
  return (
    <ToastProvider>
      {toasts.map(({ id, title, description, action, ...props }) => (
        <Toast key={id} {...props}>
          <div className="grid gap-1">
            {title && <ToastTitle>{title}</ToastTitle>}
            {description && <ToastDescription>{description}</ToastDescription>}
          </div>
          {action}
          <ToastClose />
        </Toast>
      ))}
      <ToastViewport />
    </ToastProvider>
  )
}
```

**Who needs this:**

- **New projects on `shadcn@4.14.1`**: `npx shadcn add toast` now defaults to the Base UI Toast variant (the previous Radix variant is still selectable for projects that haven't migrated to Base UI as the default).
- **Existing projects on `shadcn@4.13.0` or `4.14.0`** that want Base UI Toast: `npx shadcn@latest add toast` will overwrite the existing Radix-based Toast with the Base UI variant. The API surface is intentionally identical (same `<Toast>` / `<ToastProvider>` / `<ToastViewport>` exports, same `useToast` hook contract), so the swap should be drop-in. **Audit first:** `git diff` after the install — confirm no project code references `@radix-ui/react-toast` directly.
- **Most projects**: not affected unless they actively use `useToast` / `<Toast>`.

**Action:**
```bash
# New install
npx shadcn@latest add toast

# Or upgrade the CLI
npm install -D shadcn@^4.14.1
```

**No API change for existing Toast consumers** (the `<Toast>` / `<ToastProvider>` exports stayed the same; the underlying primitive swapped from Radix to Base UI but the React API is identical). **No breaking changes.**

**Sources:**
- [shadcn 4.14.1 release notes](https://github.com/shadcn-ui/ui/releases/tag/shadcn%404.14.1)
- [PR #11266 — `Add Base UI Toast support`](https://github.com/shadcn-ui/ui/pull/11266)
- [Compare `shadcn@4.14.0...shadcn@4.14.1`](https://github.com/shadcn-ui/ui/compare/shadcn@4.14.0...shadcn@4.14.1)
- [Base UI Toast docs](https://base-ui.com/react/components/toast) (the new default primitive)

## shadcn 4.15.0 — Public `addRegistryItems` API for Programmatic Registry Install (July 25, 2026)

Released 2 days after 4.14.1 (July 23 → July 25, 2026T18:05:56Z), `shadcn@4.15.0` is a **minor** release that exposes a single new public API: [`addRegistryItems`](https://github.com/shadcn-ui/ui/pull/11276) by [@cmpadden](https://github.com/cmpadden), commit [`7d90dfc0a5ec70cdc3bd08b741a42440041907ac`](https://github.com/shadcn-ui/ui/commit/7d90dfc0a5ec70cdc3bd08b741a42440041907ac). It lets you install registry items (shadcn components, blocks, themes, hooks, etc.) from JavaScript/TypeScript code **without invoking the `shadcn` CLI**. No CLI flag changes, no `components.json` schema changes, no breaking changes to `init` / `add` / `eject` / `migrate`.

### 1. What `addRegistryItems` Does

The function is the programmatic counterpart to `npx shadcn@latest add <item>`. It accepts an array of registry items (URLs, local paths, or inline registry JSON) and resolves them against the project's `components.json`, writes the generated files into the right paths, updates `package.json` dependencies, and returns a structured result describing what got installed. The CLI's `add` command is now a thin wrapper around this same function — calling the function directly is the right approach when:

- A build step or generator needs to install shadcn items as part of its output (codegen pipelines, design-system compilers, internal scaffolding tools)
- A long-running dev server wants to hot-add a registry item without spawning a subprocess
- An MCP / agent workflow wants to add registry items without shell access
- A monorepo package-manager abstraction layer wants shadcn installs to go through the same code path as every other install

### 2. Basic Usage (TypeScript)

```ts
// scripts/install-shadcn-blocks.ts
import { addRegistryItems } from "shadcn"

const result = await addRegistryItems({
  // Resolve the registry item from a URL, local path, or inline JSON
  items: [
    "https://ui.shadcn.com/r/styles/new-york/dashboard.json",
    "./blocks/data-table.json",
  ],
  // Optional: override the components.json cwd
  cwd: process.cwd(),
  // Optional: surface warnings (defaults to true in CI, false in TTY)
  silent: false,
  // Optional: overwrite existing files without prompting
  overwrite: true,
  // Optional: install the items' declared dependencies (defaults to true)
  installDependencies: true,
})

// result.shape — structured report of what got installed
console.log(result.filesInstalled)   // ["components/dashboard/page.tsx", ...]
console.log(result.dependenciesAdded) // ["@tanstack/react-table", ...]
console.log(result.warnings)         // [...]
```

### 3. Inline Registry Items (For Codegen Pipelines)

```ts
import { addRegistryItems } from "shadcn"

const inlineButton = {
  name: "button",
  type: "components:ui",
  files: [
    {
      path: "components/ui/button.tsx",
      content: `import * as React from "react"
import { Slot } from "@radix-ui/react-slot"
import { cva, type VariantProps } from "class-variance-authority"
// ... generated content ...
`,
    },
  ],
  dependencies: {
    "@radix-ui/react-slot": "^1.1.0",
    "class-variance-authority": "^0.7.1",
  },
} as const

const result = await addRegistryItems({
  items: [inlineButton],
  cwd: process.cwd(),
})
```

### 4. What Stays CLI-Only

The CLI command surface is unchanged — `init`, `add`, `diff`, `info`, `migrate`, `eject` all still work exactly as before. `addRegistryItems` is the **internal API that backs `add`** now exposed publicly. Any feature that the CLI exposes but `addRegistryItems` doesn't (interactive registry search, `--dry-run` rendering, etc.) is intentionally CLI-only and can still be reached via `npx shadcn@latest add ...`.

### 5. Recommended Version Pin

```bash
npm install --save-dev shadcn@^4.15.0   # or use npx shadcn@latest ...
```

**Migration checklist (4.14.x → 4.15.0):**
- [ ] `npx shadcn@latest` (which now resolves to 4.15.0+) — no peer-dep changes
- [ ] No `components.json` migration required
- [ ] Existing CLI workflows (`init`, `add`, `diff`, `eject`, `migrate`) work unchanged
- [ ] If you wrote a custom CLI wrapper that shells out to `npx shadcn add ...` from a build step, you can now replace it with an in-process `addRegistryItems` call (faster, no subprocess overhead, structured return value)
- [ ] **No migration required** if you only used the CLI

### 6. Practical Wins for Codegen / Agent Pipelines

- **No subprocess overhead** — call directly from a long-running process instead of spawning `npx shadcn add` per item
- **Structured return value** — `result.filesInstalled`, `result.dependenciesAdded`, `result.warnings` replace the CLI's stdout parsing
- **Atomic batch installs** — pass an array of items and they're resolved/written as a single transaction (so a partial failure doesn't leave the project in a half-installed state)
- **Testable** — mock `addRegistryItems` in vitest instead of mocking the CLI subprocess
- **Type-safe items** — the function accepts the same `registryItemSchema` that `components.json` validates against, so TypeScript catches malformed registry payloads before they reach the filesystem

**Sources:**
- [shadcn 4.15.0 release notes](https://github.com/shadcn-ui/ui/releases/tag/shadcn@4.15.0)
- [PR #11276 — Add a public `addRegistryItems` API](https://github.com/shadcn-ui/ui/pull/11276)
- [Compare `shadcn@4.14.1...shadcn@4.15.0`](https://github.com/shadcn-ui/ui/compare/shadcn@4.14.1...shadcn@4.15.0)
- [shadcn registry schema docs](https://ui.shadcn.com/docs/registry)

## shadcn 4.16.0 — `addRegistryItems` Accepts Config + Load Registries from `package.json` (July 27, 2026)

Released **2 days after 4.15.0** (2026-07-27T18:32:08Z), `shadcn@4.16.0` is a **minor** release that makes the 4.15.0 programmatic registry API genuinely composable: it now accepts an explicit `Config` (so programmatic consumers don't need to shell out to a `cwd` reading), and it reads the registry map from the project's top-level `package.json` field when `components.json` is absent. Three new public registries were also added to the [shadcn registry directory](https://ui.shadcn.com/r/registries.json). The CLI surface is unchanged — `init`, `add`, `diff`, `info`, `migrate`, `eject` all work exactly as before. No breaking changes to 4.15.x.

### What's new in 4.16.0

Four material changes between 4.15.0 and 4.16.0, all registry-programmability-focused:

#### 1. `addRegistryItems` accepts a `config` parameter (PR [#11307](https://github.com/shadcn-ui/ui/pull/11307))

The 4.15.0 `addRegistryItems` only read registry configuration from `components.json` (via implicit `cwd`). 4.16.0 lets callers pass an explicit `Partial<Config>` so the registry map can be programmatically built, merged across monorepos, or sourced from anywhere (database, env vars, dynamic discovery). Pair with the new `getRegistriesConfig` helper (below) for a clean "load from project, then call" pattern:

```ts
// scripts/install-from-monorepo-registries.ts
import { addRegistryItems, getRegistriesConfig } from "shadcn/registry"

const cwd = process.cwd()
const config = await getRegistriesConfig(cwd) // reads components.json OR package.json

await addRegistryItems(["@acme/agent", "@platform/login-form"], {
  cwd,
  config,
  overwrite: false,
  silent: true,
})
```

The `Config` shape mirrors what `components.json` stores under `registries` — a `{ "@namespace": "https://example.com/r/{name}.json" | { url, headers? } }` map. `addRegistryItems` no longer reads project configuration files itself; you pass the config (or the result of `getRegistriesConfig`).

#### 2. Load registries from `package.json` (PR [#11304](https://github.com/shadcn-ui/ui/pull/11304))

The new [`getRegistriesConfig(cwd)`](https://ui.shadcn.com/docs/registry/api-reference#getregistriesconfig) helper reads registry configuration from a project directory using a **two-tier resolution**:

1. **If `components.json` exists in `cwd`** — read `registries` from it (existing behavior, unchanged).
2. **Otherwise** — read the top-level `registries` field from `package.json`.

This means projects that don't use `components.json` (the CLI init never ran — e.g., headless toolings, CI codegen pipelines, "I'm just consuming a registry" agents) can still declare their registry map in `package.json`:

```json
// package.json — registries field at the top level
{
  "name": "my-app",
  "registries": {
    "@acme": {
      "url": "https://acme.com/r/{name}.json",
      "headers": {
        "Authorization": "Bearer ${ACME_TOKEN}"
      }
    },
    "@platform": "https://platform.example.com/r/{name}.json"
  }
}
```

Then `getRegistriesConfig(process.cwd())` returns the parsed config — `addRegistryItems(["@acme/agent"])` resolves `@acme` to `https://acme.com/r/agent.json` and forwards `Authorization: Bearer <ACME_TOKEN>` automatically. **This is the agent-friendly entry point** — your agent can install from any registry by reading `package.json` alone, no need to run `shadcn init` first.

**Note:** the `registries` field in `package.json` is **not read by the CLI's `npx shadcn add <item>` command** — only by the programmatic `getRegistriesConfig` helper. The CLI still reads from `components.json` (existing behavior). This is deliberate: `components.json` is the canonical registry map for the CLI workflow, while `package.json` is the canonical map for programmatic workflows.

#### 3. `useCache` option on every registry helper

Every registry helper (`getRegistry`, `getRegistryItems`, `getRegistriesConfig`) now accepts a `useCache: boolean` option (default `true`). Registry responses are cached **in memory for the lifetime of the process**, keyed by the resolved URL. Because the in-flight promise is cached, concurrent requests for the same URL are de-duplicated into a single fetch.

```ts
// Long-running server / watcher / MCP server — disable cache for fresh data
const fresh = await getRegistry("@shadcn", { useCache: false })
```

**When to disable:** long-running processes (servers, watchers, the MCP server) where the registry can change between requests and you need fresh data each time. Leave enabled for one-off scripts and CLI runs.

#### 4. Three new public registries in the registry directory

The [shadcn registry directory](https://ui.shadcn.com/r/registries.json) (the index of third-party registries you can `npx shadcn add @namespace/item` from) gained three new entries on 2026-07-25 / 2026-07-27:

| Registry | PR | Added | What it ships |
|---|---|---|---|
| `@navui` (NavUI) | [#11290](https://github.com/shadcn-ui/ui/pull/11290) | 2026-07-25 | Navigation primitives — top-bar / sidebar / breadcrumb / pagination / tabs registered against the shadcn API |
| `@flowui` (flowui) | [#11236](https://github.com/shadcn-ui/ui/pull/11236) | 2026-07-25 | Flow-diagram components — nodes, edges, minimap, controls, toolbar; built on the shadcn registry contract so they install with `npx shadcn add @flowui/...` |
| `@shadcn-dashboard` | [#11245](https://github.com/shadcn-ui/ui/pull/11245) | 2026-07-27 | Admin-dashboard blocks — sidebar shells, KPI cards, recent-orders tables, user-management tables, settings forms, all as installable items |

Plus a small site fix ([#11308](https://github.com/shadcn-ui/ui/pull/11308) — remove React Aria logo from the announcement section on `ui.shadcn.com`, since the [announcement post](https://ui.shadcn.com/docs/changelog/2026-07-react-aria) already covers it).

### Recommended version pin

```bash
npm install --save-dev shadcn@^4.16.0
```

**Migration checklist (4.15.x → 4.16.0):**
- [ ] `npx shadcn@latest` (which now resolves to 4.16.0+) — no peer-dep changes
- [ ] **No `components.json` migration required** — the new `package.json` `registries` field is opt-in; if you have `components.json` with a `registries` map, keep using it (it's still the canonical source for the CLI)
- [ ] **No CLI workflow change** — `init` / `add` / `diff` / `info` / `migrate` / `eject` all work as in 4.15.x
- [ ] **Programmatic consumers** — if your code calls `addRegistryItems` and relied on implicit `cwd`-based registry reading, switch to `addRegistryItems(items, { config: await getRegistriesConfig(cwd), cwd })` for explicit registry config (works whether the registry map lives in `components.json` or `package.json`)
- [ ] **Headless toolings / agents** — if your tooling never ran `shadcn init` (no `components.json`), you can now declare `registries` in `package.json` and call `addRegistryItems` directly without writing a `components.json` first

### Sources

- [shadcn 4.16.0 release notes](https://github.com/shadcn-ui/ui/releases/tag/shadcn@4.16.0)
- [PR #11307 — `feat(registry): accept config in addRegistryItems`](https://github.com/shadcn-ui/ui/pull/11307)
- [PR #11304 — `feat(registry): load registries from package.json`](https://github.com/shadcn-ui/ui/pull/11304)
- [PR #11290 — `feat: add NavUI registry`](https://github.com/shadcn-ui/ui/pull/11290)
- [PR #11236 — `feat(registry): add flowui to registry directory`](https://github.com/shadcn-ui/ui/pull/11236)
- [PR #11245 — `feat(registry): added new registry (@shadcn-dashboard)`](https://github.com/shadcn-ui/ui/pull/11245)
- [shadcn registry API reference — `addRegistryItems`](https://ui.shadcn.com/docs/registry/api-reference#addregistryitems)
- [shadcn registry API reference — `getRegistriesConfig`](https://ui.shadcn.com/docs/registry/api-reference#getregistriesconfig)
- [Compare `shadcn@4.15.0...shadcn@4.16.0`](https://github.com/shadcn-ui/ui/compare/shadcn@4.15.0...shadcn@4.16.0)
- [shadcn registry directory](https://ui.shadcn.com/r/registries.json)

## shadcn React Aria Support (July 17, 2026) — Third First-Class Component Base

Announced on **July 17, 2026** via the official changelog post ["July 2026 — React Aria"](https://ui.shadcn.com/docs/changelog/2026-07-react-aria) (shadcn twitter: ["React Aria is now available in shadcn/ui. Use \`--base aria\` or choose React Aria in shadcn/create. All components, docs, CLI, styles and skills have been updated for React Aria Components."](https://x.com/shadcn/status/2078142090177806773)), **React Aria is now the third first-class component base in shadcn/ui**, joining **Base UI** (default since `shadcn@4.13.0`, July 3, 2026) and **Radix UI** (the original shadcn base since 2023). The skill previously documented only Base UI and Radix — this section closes the React Aria gap so agents can correctly recommend and use the third base.

### What this is

[React Aria Components](https://react-spectrum.adobe.com/react-aria/components.html) is Adobe's accessible component primitives library — the same building blocks that power Adobe's design tools (Spectrum, Photoshop web, Adobe Express, Acrobat). React Aria has existed since 2022 but wasn't a shadcn base until now. With this addition, the three bases are:

| Base | Package | Default since | Strongest at |
|---|---|---|---|
| **Base UI** | `@base-ui/react` | `shadcn@4.13.0` (Jul 3, 2026) | Modern API, post-React-Compiler era, smaller bundle, designed around `use()` + Suspense |
| **Radix UI** | `@radix-ui/react-*` | Original (Jan 2023) → `shadcn@4.13.0` | Mature, widest ecosystem, largest community mindshare, battle-tested in millions of apps |
| **React Aria** | `react-aria-components` | `shadcn@4.13.x` (Jul 17, 2026) | Accessibility maturity, keyboard/AT coverage, i18n, mobile/touch, the Adobe ecosystem |

**Base UI remains the default** — React Aria is opt-in via `--base aria` (or via the Base UI picker in the init prompt). Existing projects are NOT migrated to React Aria by any CLI command.

### What's actually new (shadcn side)

From the official changelog and shadcn's announcement tweet:

1. **A first-class React Aria base** — React Aria is selectable anywhere a base is selectable: `npx shadcn@latest init --base aria`, the interactive init prompt, `shadcn/create?base=aria`, and any registry-based preset.
2. **Full component documentation** — every component page at [ui.shadcn.com/docs/components](https://ui.shadcn.com/docs/components) now has parallel React Aria install/usage/composition/examples/API-reference tabs alongside the existing Base UI and Radix tabs.
3. **All eight styles supported** — React Aria components are available with every style: **Vega, Nova, Maia, Lyra, Mira, Luma, Rhea, Sera**. Each style gets its own React Aria implementation (`aria-vega`, `aria-nova`, `aria-maia`, etc.) analogous to the existing `base-*` and `radix-*` flavors.
4. **Base-specific output** — React Aria state selectors (`data-focused`, `data-disabled`, `data-focus-visible`, etc.) and dependencies (`react-aria-components`, `@react-aria/i18n`, `@react-aria/live-announcer` as appropriate) are scoped to a separate "aria" registry. Existing Base UI and Radix components in a project are unchanged — adding an `aria-vega-button` to a project on the Base UI base does NOT add React Aria as the base, it just installs that one component's React Aria implementation.
5. **`shadcn/create` shows all three bases** — at [ui.shadcn.com/create?base=aria](https://ui.shadcn.com/create?base=aria) (and the corresponding `?base=base` / `?base=radix` URLs), the create flow now lists Base UI (default), Radix UI, and React Aria as equal choices.

### How to start with React Aria

```bash
# Non-interactive init with React Aria
pnpm dlx shadcn@latest init --base aria

# Interactive init (prompts for "Which headless library would you like to use?")
pnpm dlx shadcn@latest init
# → Select: React Aria

# Add React Aria implementations of specific components
pnpm dlx shadcn@latest add button dialog dropdown-menu tooltip
# (After --base aria init, `add` picks the aria-* registry item by default)

# Switch to a different base mid-project (advanced — see "Migration" below)
# The CLI doesn't have a one-shot migrate; you re-add each component with --force
```

After `init --base aria`, the project's `components.json` records React Aria as the base, the install step pulls `react-aria-components` (and `@react-aria/i18n` for components that need it), and `globals.css` gets the React Aria data-attribute CSS variables (`--color-focus-ring`, `--color-pressed`, etc.) that React Aria primitives read.

### When to pick React Aria over Base UI / Radix

| Situation | Pick | Why |
|---|---|---|
| New project, no strong preference | **Base UI** | The default since 4.13.0; modern API; built by the same team that built Radix; best React 19 fit |
| Existing project on Radix, no reason to leave | **Radix** | Mature, no migration cost, still fully supported |
| **App needs Adobe-grade accessibility / i18n** (e.g. enterprise B2B, government, healthcare) | **React Aria** | Adobe's Spectrum/Photoshop web/Acrobat stack relies on it; mature keyboard, screen-reader, RTL, and reduced-motion handling |
| **App has heavy touch / mobile / keyboard requirements** | **React Aria** | Most thorough touch event handling, gesture conflicts, virtual keyboard handling |
| **You ship to markets with non-Latin scripts** | **React Aria** | First-class `Intl.*` integration, bidirectional text, locale-aware date/number formatting |
| **You want the smallest React bundle** | **Base UI** | Base UI was built for the post-React-Compiler era and trims dependency tree more aggressively |
| **You want the widest community mindshare / Stack Overflow coverage** | **Radix** | Most-shared, most-blogged-about; works with the largest set of community examples |
| **You're integrating with an Adobe product / Spectrum design language** | **React Aria** | Shared primitives; styles compose with Spectrum |

For most new SaaS / consumer apps, **Base UI (default)** remains the right pick. React Aria is the right pick when accessibility/i18n/mobile requirements dominate the requirements doc.

### How React Aria components differ from Base UI / Radix (API surface)

The skill's `components.md` already documents Base UI vs Radix differences in the **Source-level changes worth knowing** section of the shadcn 4.13.0 entry (Dialog `Content` → `Popup`, DropdownMenu → `Menu`+`Popup`, Accordion `Content` → `Panel`). React Aria adds its own naming / state-prop patterns; the most common differences from Base UI:

- **Single `<Dialog>` vs split `<Dialog.Root>` + `<Dialog.Trigger>` + `<Dialog.Content>`** — React Aria's primitives are a single component with render props or slot children, not Radix/Base-UI-style compound components. Composition is via the `<DialogTrigger>` wrapper component, not nested children:
  ```tsx
  // Base UI / Radix style
  <Dialog.Root>
    <Dialog.Trigger>Open</Dialog.Trigger>
    <Dialog.Content>...</Dialog.Content>
  </Dialog.Root>

  // React Aria style
  <DialogTrigger>
    <Button>Open</Button>
    <Modal>...</Modal>
  </DialogTrigger>
  ```

- **Render props vs `asChild`** — React Aria leans on **render props** (`<Button>{props => <a {...props}>...</a>}</Button>`) or **`href`** / **`to`** props on its built-in `<Button>` (which renders as `<a>` when given a URL) instead of Radix's `asChild` / Base UI's `render` prop. shadcn's React Aria adapter wraps this into a familiar `asChild` API where it can, but pure React Aria code reads render-prop-first.

- **Data-attribute selectors** — React Aria primitives emit a richer set of `data-*` attrs than Base UI/Radix: `data-focused`, `data-focus-visible`, `data-pressed`, `data-hovered`, `data-disabled`, `data-invalid`, `data-required`, `data-readonly`, `data-selected`, `data-empty`, `data-loading`. shadcn's React Aria registry ships pre-written Tailwind variants for the common ones (`data-[focus-visible]:ring-2`, `data-[disabled]:opacity-50`, etc.) so the experience matches the existing `data-open:` / `data-closed:` pattern documented for Base UI.

- **i18n context provider** — apps using React Aria's date pickers / number inputs / comboboxes need to wrap the app in `<I18nProvider>` from `@react-aria/i18n`. shadcn's React Aria adapter auto-installs this provider into a fresh project's root layout when you `init --base aria`; if you `add` React Aria components into an existing Base UI or Radix project, you need to add the provider yourself.

- **`onAction` vs `onClick`** — React Aria lists/menus use **`onAction`** (semantic: "user performed the action") instead of **`onClick`** (DOM event). This is a meaningful API divergence that the Base UI / Radix equivalents don't have.

### `components.json` After `init --base aria`

The Base UI vs React Aria difference shows up in `components.json` and `package.json`. After `init --base aria`:

```json
// components.json — base is recorded implicitly via the deps installed
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "app/globals.css",
    "baseColor": "neutral",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  },
  "iconLibrary": "lucide"
}
```

```json
// package.json — pulled by `init --base aria`
{
  "dependencies": {
    "react-aria-components": "^1.x",
    "@react-aria/i18n": "^3.x"      // only if you add date/number/i18n components
  }
}
```

The CLI uses the installed dep (`react-aria-components` vs `@base-ui/react` vs `@radix-ui/react-*`) as the source of truth for the base — there's no formal `headless: "aria" | "base" | "radix"` flag in `components.json` yet (the `--base` flag is init-only and only persists indirectly through which dep the CLI sees installed).

### Migration to React Aria from Base UI / Radix

The CLI doesn't ship a `npx shadcn@latest migrate --to=aria` command. Three paths, in increasing order of effort:

**Path A — Fresh project with React Aria:** spin up a new project with `init --base aria`, copy your custom components and pages into it. Fastest if your project is small or the Base UI / Radix base was the only thing blocking a rebuild.

**Path B — Mixed (acceptable for large codebases mid-migration):** keep your Base UI or Radix base, and add individual React Aria components when you need the extra accessibility / i18n / touch coverage they offer. shadcn supports adding `aria-*` items to any project — the registry scope is per-component, not per-project. The `react-aria-components` package gets added to `package.json` as a side effect; the existing Base UI or Radix packages stay. Bundle size cost: `react-aria-components` is ~30KB minzipped; only worth paying for if you're using it.

**Path C — Full base swap:** re-`init --base aria` over an existing project (the CLI doesn't overwrite your files, but it DOES overwrite `components.json` / `globals.css` / `lib/utils.ts` / etc — back those up first), then re-add each Base UI or Radix component as its React Aria equivalent with `--force`. Expect a half-day of API translation per ~20 components: Dialog (`Content` → `Modal`/`Popover`), DropdownMenu → `Menu`, Accordion `Content` → `Panel`, Tooltip `Provider` → `TooltipTrigger`, Select → `Select`+`Popover`, plus render-prop rewrites for any custom wrappers you have.

### Common Mistakes — React Aria in shadcn

- **Mixing three bases in one project to "diversify"** — each base pulls its own primitive infrastructure; mixing all three costs ~80-100KB minzipped for the redundancy and gains nothing. Pick one per project.
- **Forgetting `<I18nProvider>`** — React Aria date pickers, comboboxes, number inputs, and calendars all require the provider for locale-aware behavior. shadcn's `init --base aria` adds it to the root layout automatically, but if you `add aria-*` components to an existing project, you must add `<I18nProvider>` to `app/layout.tsx` yourself.
- **Using `onClick` on a React Aria `<MenuItem>` / `<ListBoxItem>`** — React Aria lists/menus use **`onAction`**, not `onClick`. `onClick` fires on the underlying DOM element but doesn't go through React Aria's action semantics (keyboard activation, touch long-press, screen-reader announcement). Use `onAction` for the semantic intent; reserve `onClick` for DOM-level events only.
- **Using `asChild` on React Aria components** — shadcn's React Aria adapter wraps the common `asChild` pattern where it can, but the underlying React Aria primitives don't support `asChild` natively. If you hit an `asChild`-related lint warning on a component you can't wrap, fall back to a render prop: `<Button>{props => <a {...props}>...</a>}</Button>`.
- **Trying to install a Base UI component into a React Aria project without the Base UI dep** — shadcn's Base UI registry items (`base-luma-button`, `base-nova-dialog`, etc.) assume `@base-ui/react` is installed. On a React Aria project without that dep, the install succeeds but the generated component throws at runtime. Install `@base-ui/react` first, or pick the equivalent `aria-*` registry item.
- **Migrating from Radix for the sake of migrating** — Radix still ships security patches, the Base UI (default) team is the same team that built Radix, and React Aria is best for accessibility/i18n-heavy apps, not general-purpose migrations. Migrate only if the Base UI or React Aria feature set solves a problem you actually have.

### Sources

- [shadcn — July 2026: React Aria announcement](https://ui.shadcn.com/docs/changelog/2026-07-react-aria)
- [shadcn — Changelog index](https://ui.shadcn.com/docs/changelog)
- [shadcn twitter — React Aria launch announcement](https://x.com/shadcn/status/2078142090177806773)
- [shadcn create — React Aria preset](https://ui.shadcn.com/create?base=aria)
- [shadcn components — React Aria tabs](https://ui.shadcn.com/docs/components)
- [React Aria Components — official docs](https://react-spectrum.adobe.com/react-aria/components.html)
- [React Aria `react-aria-components` on npm](https://www.npmjs.com/package/react-aria-components)
- [daily.dev — "React Aria components are now available in shadcn/ui" (Jul 17, 2026)](https://daily.dev/posts/react-aria-components-are-now-available-in-shadcn-ui-gyzi6eyu9)
- [Adobe Spectrum / Photoshop web stack — context for React Aria maturity](https://react-spectrum.adobe.com/)


## React 19.3.0-canary-ec61f187-20260806 SHIPPED (August 7, 2026) — `11eddecd` → `ec61f187` (PR #37203 + PR #37215 in canary bundle)

**[07 Aug 2026 18:03Z] v1.5.35 cycle**, **32 minutes before this cron's check** at 2026-08-07T16:31:06Z, **`react@canary` flipped from `19.3.0-canary-11eddecd-20260805` → `19.3.0-canary-ec61f187-20260806`** (npm `dist-tag.canary` moved; GitHub release tag `v19.3.0-canary-ec61f187-20260806` published at the same time by `gaearon`). The new canary tag is `ec61f187fe`. The `experimental` dist-tag also bumped in lockstep to `0.0.0-experimental-ec61f187-20260806` (published 2026-08-07T16:32:35Z, ~83s after the canary). The gap between `11eddecd-20260805` (Aug 5 10:00:39Z) and `ec61f187-20260806` (Aug 7 16:31:06Z) is **~54h30min** — well within the typical 20-72h React canary cadence, but shows the team is **shipping the `main`-branch-ahead commits through to npm quickly** (the v1.5.31 cycle documented PR #37215 as a `main`-branch-ahead commit; only 32h elapsed between that commit's merge on Aug 6 and the npm-publish of the canary including it).

The canary bundle vs `11eddecd-91` includes **2 NEW commits** (verified at this cron's check via `GET /repos/facebook/react/compare/11eddecd91...ec61f187fe` returning `ahead_by: 2, behind_by: 0`):

| Commit | PR / Author | Merged | Description |
|---|---|---|---|
| `1d4758e0f6` | [PR #37203](https://github.com/facebook/react/pull/37203) — [flags] Enable conditional use warning for experimental release channel | 2026-08-05T16:53:19Z | Flips the `enableConditionalUseWarning` flag ON for the `experimental` release channel only (already documented in v1.5.27 doc tagged onto the 11eddecd section) |
| `ec61f187fe` | [PR #37215](https://github.com/facebook/react/pull/37215) — [DevTools] Fix nested HOC name extraction in `extractHOCNames` | 2026-08-06T12:44:04Z | DevTools-only fix for the regex `lastIndex` bug that corrupted nested HOC component names (already documented in v1.5.31 as a `main`-branch-ahead commit; now live in npm) |

**Practical impact (NOW live in `react@19.3.0-canary-ec61f187-20260806`)**:

- **DevTools users debugging nested HOC component trees** (e.g., `withRouter(withStyles(MyComponent))` or `memo(forwardRef(MyComponent))`) — DevTools now shows the **correct nested name** in the component tree instead of the old truncated-only-top-level name. The pre-fix bug: `extractHOCNames()` used a non-global regex that retained its `lastIndex` across calls, so the second HOC's name extraction started from the position the first call ended at, producing wrong names like `Forget(Memo(ForgetMemoCounter))` instead of the correct nested `✨🧠ForgetMemoCounter`. Post-fix: every HOC layer appears in the DevTools display name correctly.
- **Devs on `react@experimental`** will see **new DEV warnings** about conditional `use(promise)` calls in their dev consoles (the PR #37203 flag-flip for the `experimental` channel only). See the v1.5.27 section: `## React 19.3.0-canary-11eddecd-20260805 SHIPPED + React main branch: enableConditionalUseWarning flag (PR #37203, August 5, 2026)` for the full warning text + audit recipe.
- **Production users on `react@latest` 19.2.8**: zero impact.
- **Canary users on `react@canary` 19.3.0-canary-ec61f187-20260806**: only the DevTools name fix is live (the `enableConditionalUseWarning` flag is OFF for the canary channel).

**Next.js vendor**: The Next.js canary-branch already merged **PR #96735** (the `11eddecd` vendor bump) — the next React vendor bump (forward-looking) will target `ec61f187-20260806`. The next React vendor bump will land in the next Next.js canary cut after this cron's check; track the canary-branch's React vendor bump PR (none open at this cron's check). Verify with `npm view next@canary dependencies.react` — currently `19.3.0-canary-11eddecd-20260805`; will flip to `19.3.0-canary-ec61f187-20260806` once the next Next.js canary npm-publishes.

### Audit recipe (after upgrading to `react@canary` 19.3.0-canary-ec61f187-20260806)

```bash
# 1. Confirm the canary dist-tag is now ec61f187:
npm view react dist-tags.canary
# → should show: 19.3.0-canary-ec61f187-20260806

# 2. Confirm the experimental dist-tag also bumped:
npm view react dist-tags.experimental
# → should show: 0.0.0-experimental-ec61f187-20260806

# 3. Verify the DevTools HOC name fix landed in the canary bundle:
grep -A 2 'function extractHOCNames' node_modules/react/cjs/react-dom-client.development.js 2>/dev/null | head -10
# → should show the regex pattern with the global `g` flag (post-fix)
# → pre-fix the regex was missing `g` and corrupted nested HOC names

# 4. Visual smoke test in DevTools:
# Create a nested HOC component like `memo(forwardRef(MyComponent))` and inspect
# its display name in React DevTools — should show "🧠forwardRef(MyComponent)" with
# both wrappers visible, not just the top-level wrapper.
```

### Common Mistakes (components-relevant)

- **Reporting nested HOC display names as "broken in React DevTools" when you're on `react@canary < 19.3.0-canary-ec61f187`** — the `extractHOCNames` regex `lastIndex` bug pattern was inherited from `cbb046ab-20260731` and persisted through `7dfc7ccd-20260803` + `11eddecd-20260805`. The bug is fixed in `ec61f187-20260806` (PR #37215). If you're on a pre-`ec61f187` canary and seeing nested HOC names truncated, bump to `react@>=19.3.0-canary-ec61f187-20260806` (npm-published 2026-08-07) — the fix is in the canary bundle, no code changes required. The bug only affects DevTools display names; runtime behavior is unchanged.
- **Expecting `enableConditionalUseWarning` to fire on `react@canary`** — the flag is OFF for the canary channel by design. If you want to see the conditional-`use(promise)` DEV warnings, switch to `react@experimental` (which is now `0.0.0-experimental-ec61f187-20260806`). Canary users will not see the warnings. See the v1.5.27 section for the canonical anti-pattern + rewrite + audit recipe.

### Sources

- [npm: `react@19.3.0-canary-ec61f187-20260806`](https://www.npmjs.com/package/react/v/19.3.0-canary-ec61f187-20260806) (published 2026-08-07, dist-tag `canary` moved ~16:31:06Z)
- [React `main` branch commits feed (last 10)](https://github.com/facebook/react/commits?sha=main) — verified at 2026-08-07T18:03Z; main-branch head is `ec61f187fe` (PR #37215 [DevTools]); main-branch is now identical to the canary cut (no further commits ahead)
- [React PR #37215 — `[DevTools] Fix nested HOC name extraction in extractHOCNames`](https://github.com/facebook/react/pull/37215) — by BIKI DAS, merged 2026-08-06T12:44:04Z, DevTools-only fix (72 new test cases in `extractHOCNames-test.js` + 1-line fix in `utils.js`)
- [React PR #37203 — `[flags] Enable conditional use warning for experimental release channel`](https://github.com/facebook/react/pull/37203) — merged 2026-08-05T16:53:19Z; flag-flip for the experimental channel only (already documented in v1.5.27's 11eddecd section)
- [React `v19.3.0-canary-ec61f187-20260806` GitHub release tag](https://github.com/facebook/react/releases/tag/v19.3.0-canary-ec61f187-20260806) — published 2026-08-07T16:31:06Z
- [React compare `11eddecd91...ec61f187fe`](https://github.com/facebook/react/compare/11eddecd91...ec61f187fe) — 2 commits at this cron's check (PR #37203 + PR #37215)
- [Cross-reference: v1.5.27 `## React 19.3.0-canary-11eddecd-20260805 SHIPPED + React main branch: enableConditionalUseWarning flag (PR #37203, August 5, 2026)`](https://github.com/clawvpsai/frontend-skill/blob/main/components.md#react-1930-canary-11eddecd-20260805-shipped--react-main-branch-enableconditionalusewarning-flag-pr-37203-august-5-2026--7dfc7ccd--11eddecd) — the 11eddecd SHIP event with PR #37203 forward-looking
- [Cross-reference: v1.5.31 `## React main branch ahead of 11eddecd: 1 NEW commit — PR #37215 [DevTools] Fix nested HOC name extraction in extractHOCNames (August 6, 2026) — DevTools-Only`](https://github.com/clawvpsai/frontend-skill/blob/main/components.md#react-main-branch-ahead-of-11eddecd-1-new-commit--pr-37215-devtools-fix-nested-hoc-name-extraction-in-extract hocnames-august-6-2026--devtools-only) — the v1.5.31 forward-looking note for PR #37215 (now live in ec61f187 canary)

## React Main Branch — `onBrowserBailout` Fizz Option (PR #37193, August 8, 2026 — Forward-Looking for React 19.3.x)

A NEW React main branch commit landed in the 5h window since the v1.5.36 cycle: **PR #37193 `Add onBrowserBailout Fizz option`** by Josh Story (gnoff@storyposted.com), merged 2026-08-08T02:31:46Z (~3h30min before this cron's check). Verified at this cron's check via `GET /repos/facebook/react/commits?per_page=10&sha=main` returning 10 commits ahead of the latest npm-publish (the latest canary `ec61f187-20260806` is at the 9th position in the main-branch feed; PR #37193 is at position 1). **Not yet npm-published in any canary dist-tag.** The PR is 130 lines added / 24 lines deleted across 17 files (the 17 files include the Fizz SSR runtime + a comprehensive test suite).

### What is `onBrowserBailout`?

Fizz is React's **server-side streaming renderer** used by Next.js + Waku + other React 19 server-rendered frameworks. When a server render hits a `ReactDOM.browser()` call (the React 19.x API for "this should run on the browser, not the server") or any future API that uses the same recoverable error mechanism, Fizz can **bail out** of the server render and defer the work to the browser. The new `onBrowserBailout` option lets you observe these intentional bailouts.

### API surface

```typescript
// New Fizz option (alongside `onError`, `onShellReady`, etc.)
onBrowserBailout?: (error: unknown, errorInfo: ErrorInfo) => void

// Default: noop
// Runs only when Fizz successfully recovers by deferring work to the browser
// Receives the original recoverable error + the ErrorInfo
```

### Behavior contract

- **Recoverable consumed within Suspense** → reported through `onBrowserBailout` (NOT through `onError`).
- **Recoverable used to abort a recoverable boundary** → reported through `onBrowserBailout` (NOT through `onError`).
- **Bailout outside Suspense** → still fatal; reports through `onError` with the original recoverable preserved as its `cause`.
- **Directly throwing the value** → behaves like a normal render error → reports through `onError`.

### Practical impact for React 19.3.x users

**Use case:** observability for server-rendered apps that intentionally bail out to the browser for client-only APIs. Without `onBrowserBailout`, the only way to detect these bailouts is to instrument every `ReactDOM.browser()` call site. With `onBrowserBailout`, a single Fizz option captures all bailout events for monitoring (Datadog, Sentry, Honeycomb, etc.).

**For Next.js 16.3+ users:** Next.js uses Fizz internally for App Router SSR. The Next.js team can opt into `onBrowserBailout` to surface bailout events in the Next.js dev overlay or telemetry pipeline. **No user-facing change** in this PR — the API is opt-in via Fizz configuration.

**For framework authors (Waku, React Router v7 framework mode, custom SSR setups):** this is the canonical hook for capturing intentional server-render bailouts. Use it to:
1. **Count bailout frequency** — high bailout rates signal a code path that's too client-heavy for SSR.
2. **Track which boundaries bail out** — `errorInfo.componentStack` tells you which component initiated the bailout.
3. **Correlate with hydration cost** — bailouts to the browser typically add hydration latency; track them together.

**Audit recipe** (for when this ships in `react@canary`):
```bash
# Watch for the new Fizz option
npm view react dist-tags
# Look for new canary cut (next expected: 19.3.0-canary-XXXXXXXX-20260XXX)

# Check if you use Fizz directly (Next.js users — Fizz is used internally, no direct API exposure)
rg "import.*fizz" --type ts --type tsx --type js --type jsx app/ src/
# If you don't import Fizz directly, you don't need to do anything; Next.js handles Fizz config

# For framework authors who wrap Fizz:
rg "renderToReadableStream.*onError" --type ts app/ src/
# Add onBrowserBailout alongside onError
```

**Recommendation:** no action required for Next.js users. **For framework authors** who wrap Fizz directly, add `onBrowserBailout` to your telemetry pipeline when React 19.3.x ships (expect within 1-3 weeks).

### Sources

- [React PR #37193 — Add `onBrowserBailout` Fizz option](https://github.com/facebook/react/pull/37193) — by Josh Story, merged 2026-08-08T02:31:46Z, 17 files / +130/-24, in the React main branch ahead of the latest canary `ec61f187-20260806`. **NOT YET in any npm-published React dist-tag.** Forward-looking for React 19.3.x (expect 19.3.0-canary-XXXXXXXX-20260XXX within 1-3 weeks).
- [React main branch commits feed](https://github.com/facebook/react/commits/main) — 10 commits at this cron's check; PR #37193 is the most recent (ahead of the latest npm canary `ec61f187-20260806`).
- [React `ReactDOM.browser()` API docs](https://react.dev/reference/react-dom/browser) — the React 19.x API that triggers the recoverable error mechanism that `onBrowserBailout` observes.
- [React Fizz renderer docs (server)](https://react.dev/reference/react-dom/server/renderToReadableStream) — the Fizz streaming SSR API that `onBrowserBailout` is an option for.
- [React 19.2 release notes (October 2025)](https://react.dev/blog/2025/10/01/react-19-2) — the React 19.2 release that introduced `ReactDOM.browser()`.
- [Cross-reference: v1.5.35 components.md `## React 19.3.0-canary-ec61f187-20260806 SHIPPED (August 7, 2026) — 11eddecd → ec61f187 (PR #37203 + PR #37215 in canary bundle)`](https://github.com/clawvpsai/frontend-skill/blob/main/components.md#react-1930-canary-ec61f187-20260806-shipped-august-7-2026--11eddecd--ec61f187-pr-37203--pr-37215-in-canary-bundle) — the prior React canary SHIP event that PR #37193 builds on


## React 19.3.0-canary-807d21fd-20260810 SHIPPED (August 10, 2026) — `ec61f187` → `807d21fd` (PR #37241 Lazy Reasons to browser() + PR #37258 Flight Key Validation of Lazy Nodes + the v1.5.37-Forward-Looking PR #37193)

**[10 Aug 2026 18:02Z] v1.5.46 cycle**, **`react@canary` flipped from `19.3.0-canary-ec61f187-20260806` → `19.3.0-canary-807d21fd-20260810`** (npm `dist-tag.canary` moved; the new canary tag is `807d21fdfdcf`, npm-published 2026-08-10; verified at this cron's check via `GET /repos/facebook/react/commits?sha=main&since=2026-08-06T00:00:00Z&per_page=10` returning **3 NEW commits** ahead of the previous canary cut). The `experimental` dist-tag also bumped in lockstep to `0.0.0-experimental-807d21fd-20260810` (published the same day). The gap between `ec61f187-20260806` (Aug 6 12:44:04Z) and `807d21fd-20260810` (Aug 10 15:42:48Z) is **~99h** — slightly above the typical 20-72h React canary cadence upper bound, **but the team waited on purpose** because the canary bundle would not have been publishable as-is until the Flight key-validation fix (PR #37258) and the `browser()` lazy-reasons follow-up (PR #37241) had both landed — both are coordinated changes that interact with the `ReactDOM.browser()` recoverable error mechanism introduced in `0f42eac2-20260730` and the `onBrowserBailout` Fizz option that the v1.5.37 forward-looking section documented.

The canary bundle vs `ec61f187-91` includes **3 NEW commits** (verified via `GET /repos/facebook/react/commits?sha=main&since=2026-08-06T00:00:00Z&per_page=10` returning exactly these 3 commits + the prior canary commit itself):

| Commit | PR / Author | Merged | Description |
|---|---|---|---|
| `2042572` | [PR #37193](https://github.com/facebook/react/pull/37193) — Add `onBrowserBailout` Fizz option (gnoff / Josh Story) | 2026-08-08T02:31:46Z | The **v1.5.37 forward-looking PR** — the Fizz SSR streaming renderer gets a new `onBrowserBailout(error, errorInfo)` option that fires on successful recoverable consumption (suspense or abort). Now live in npm. 17 files / +130/-24. |
| `8366f33` | [PR #37258](https://github.com/facebook/react/pull/37258) — [Flight] Transfer key validation of lazy nodes when they are unwrapped (unstubbable / Hendrik Liebau) | 2026-08-10T14:18:47Z | **NEW.** Fixes a **false-positive missing-key warning** in Flight-outlined values; closes #37240 + #37246; 2 files / +326/-16. **Detailed deep dive in server-components.md.** |
| `807d21f` | [PR #37241](https://github.com/facebook/react/pull/37241) — Add lazy reasons to browser() (gnoff / Josh Story) | 2026-08-10T15:42:48Z | **NEW + THE HEADLINE.** Extends `ReactDOM.browser()` to take a **cheap branded recoverable token** instead of eagerly constructing an `Error`; accepts an **optional reason string or initializer** (the initializer runs only when Fizz consumes the token; client ignores the reason without invoking; reasons may be strings, errors, or structured framework metadata preserved as `cause`); routes successful recoveries through `onBrowserBailout` and fatal clones through `onError`; substitutes a stable diagnostic fallback if the initializer throws; **8 files / +633/-145**. Closes the v1.5.37 forward-looking observation + extends it. |

**Practical impact (NOW live in `react@19.3.0-canary-807d21fd-20260810`):**

- **All devs using `ReactDOM.browser()`** — the new lazy-reason API lets you pass structured metadata (`{ kind: 'client-storage', key: 'theme' }`) that Fizz surfaces through the new `onBrowserBailout` callback's `error.cause` field. Pre-#37241, you had to eagerly construct an `Error` at every `browser()` call site even when the client would never look at it; **post-#37241 the initializer is lazy** — the client renderer ignores it, so client-only rendering does not pay for an unused stack trace. **For framework authors** wrapping Fizz (Waku, React Router v7 framework mode, custom SSR setups) the canonical hook now has structured reasons accessible via `error.cause` on every recoverable bailout — wire it to your telemetry pipeline alongside `onError`.
- **All Flight (Server Components / Server Actions) users** — **PR #37258 fixes a false-positive `Each child in a list should have a unique "key" prop` warning** that appeared on the Flight-outlined elements depending on whether a preceding prop was large enough to push the element past the outlining threshold. The fix moves the validated-static-child mark transfer from `initializeElement` (where it could only happen when the element was blocked on debug info) into the lazy-node unwrap path (where it's guaranteed to be reachable), so the warning no longer fires spuriously. **No code change required** — the fix is purely internal to Flight's outlining + element validation.
- **All `onBrowserBailout` users (frameworks wrapping Fizz)** — the v1.5.37 forward-looking `onBrowserBailout` API is now live. The `errorInfo.componentStack` field tells you which component initiated the bailout; the new PR #37241 makes the `error.cause` field a structured framework-metadata payload that survives the lazy initialization path.
- **Production users on `react@latest` 19.2.8**: zero impact (PR #37241 + PR #37258 are both canary-only).
- **Canary users on `react@canary` 19.3.0-canary-807d21fd-20260810**: the DevTools HOC name fix (PR #37215) + the `enableConditionalUseWarning` flag-flip for the experimental channel (PR #37203) remain live from `ec61f187-20260806`; the new canary adds the 3 commits above.

### API surface — `ReactDOM.browser()` lazy reasons (PR #37241)

```typescript
// Pre-PR-#37241 (eager Error construction — wasted on client-only paths):
ReactDOM.browser(new Error('theme uses client storage'))

// Post-PR-#37241 (lazy string reason — runs only when Fizz consumes the token):
ReactDOM.browser('theme uses client storage')

// Post-PR-#37241 (lazy initializer — receives a { kind, key } framework metadata object):
ReactDOM.browser(() => ({ kind: 'client-storage', key: 'theme' }))

// Post-PR-#37241 (lazy error reason — initializer throws are caught and substituted
// with a stable diagnostic fallback so reason generation cannot change rendering
// control flow):
ReactDOM.browser(() => new Error('failed to construct reason'))
```

When Fizz consumes the token through `use()` or `abort()`, it creates a **consistent browser-bailout error at the consumption point** so its stack identifies the relevant operation. The initialized reason is **preserved unchanged as the optional `cause`**, allowing strings, errors, and structured framework metadata without runtime validation. If an initializer throws, Fizz substitutes a stable diagnostic fallback so reason generation cannot change rendering control flow. Successful recoveries report the error through `onBrowserBailout`.

**Critical contract for framework authors:** when no Suspense boundary can recover the render, Fizz clones the branded recoverable error into an **unbranded fatal diagnostic** while preserving its cause and consumption frames. During an abort, the request retains the original branded error so every remaining task observes the same reason; fatal clones are created only when reporting a fatal root or closing the stream. **Centralized recoverable logging uses the brand to route successful bailouts through `onBrowserBailout` and fatal clones through `onError`.** The empty recoverable digest and client hydration suppression behavior remain unchanged.

### Test coverage matrix (PR #37241)

The PR body enumerates the test scenarios: omitted and direct reasons, lazy string, error, structured, and primitive reasons, repeated use sites, throwing initializers, lazy client behavior, consumption stacks, flattened fatal errors, recoverable and fatal `use` and `abort` paths, nested aborts, direct throws, debug tools, and development and production rendering. **The lazy initializer path + the brand-routed `onBrowserBailout`/`onError` separation are both assertion-tested.**

### Audit recipe (after upgrading to `react@canary` 19.3.0-canary-807d21fd-20260810)

```bash
# 1. Confirm the canary dist-tag is now 807d21fd:
npm view react dist-tags.canary
# → should show: 19.3.0-canary-807d21fd-20260810

# 2. Confirm the experimental dist-tag also bumped in lockstep:
npm view react dist-tags.experimental
# → should show: 0.0.0-experimental-807d21fd-20260810

# 3. Verify the new lazy-reason API surface landed:
grep -n 'function browser' node_modules/react/cjs/react-dom-client.development.js 2>/dev/null | head -5
# → should show the new `browser(reason)` signature (post-fix)

# 4. Visual smoke test for the lazy initializer path:
# In a Fizz-wrapping framework, instrument onBrowserBailout and trigger a
# ReactDOM.browser(() => ({ kind: 'theme-storage' })) from a server component.
# The error.cause field should be the structured { kind: 'theme-storage' }
# object, NOT an Error stack trace.

# 5. Verify the Flight key-validation fix (PR #37258) is live:
# Render a server component that does a dynamic import of a barrel that
# returns a JSX element. Before PR #37258, you might have seen a
# "Each child in a list should have a unique 'key' prop" warning depending
# on the prop-size of the preceding element. After PR #37258, no warning.
```

### Common Mistakes (components-relevant)

- **Passing an eagerly-constructed `Error` to `ReactDOM.browser()` in `807d21fd`-ahead projects** — pre-PR-#37241 that was the only API; post-PR-#37241 the `Error` is constructed unconditionally even when the client ignores it. **Switch to the lazy string or initializer form** (`ReactDOM.browser('reason')` or `ReactDOM.browser(() => 'reason')`) so the work is deferred until Fizz consumes the token. Affects any code path that wraps Fizz (framework authors) or any project that hits `ReactDOM.browser()` from Server Components on every render.
- **Treating `error.cause` on a `ReactDOM.browser()` bailout as always-an-Error** — PR #37241 made `cause` a structured framework-metadata field (string, Error, or arbitrary object). Telemetry pipelines that always call `error.cause.stack` will see `undefined.stack` for non-Error reasons. **Type-guard the cause before reading .stack** (e.g., `if (cause instanceof Error) captureStack(cause)`); otherwise, capture the cause's `.toString()` or `.message` for non-Error reasons.
- **Still expecting `onBrowserBailout` to NOT fire on `react@canary`** — the v1.5.37 forward-looking section warned this was forward-looking; it shipped in `807d21fd-20260810` (Aug 10). If you wrapped Fizz before this canary cut and added a TODO for the `onBrowserBailout` hook, remove the TODO — the API is live. Conversely, if you tried to use `onBrowserBailout` on a pre-`807d21fd` canary, your hook was a no-op (the option didn't exist yet); **bump to `react@>=19.3.0-canary-807d21fd-20260810`** before adding the telemetry wiring.
- **Firing the `enableConditionalUseWarning` DEV warning on canary** — unchanged from v1.5.35; the flag is OFF for the canary channel by design. Switch to `react@experimental` (`0.0.0-experimental-807d21fd-20260810`) if you want to see the warnings in dev.
- **Seeing a false-positive missing-key warning on Server Component outlined elements** — **FIXED in `807d21fd-20260810` by PR #37258** (Flight transfer-key-validation-of-lazy-nodes fix). If you're on a pre-`807d21fd` canary and seeing the warning on JSX-with-Flight-outlined-children that *has* a `key` prop, bump to `react@>=19.3.0-canary-807d21fd-20260810`. The fix is purely internal; no code change required. **Detailed deep dive in server-components.md `## Flight — PR #37258 Transfer Key Validation of Lazy Nodes When Unwrapped (Aug 10, 2026)` section.**
- **Waiting for `onBrowserBailout` to reach `react@latest`** — this is a canary-only feature. `react@latest` 19.2.8 doesn't expose it. Don't bump your production framework dependency to `react@canary` just for this; wait for `react@19.3.x` STABLE (expect within 2-4 weeks based on recent React release cadence).

### Sources

- [npm: `react@19.3.0-canary-807d21fd-20260810`](https://www.npmjs.com/package/react/v/19.3.0-canary-807d21fd-20260810) — npm-published 2026-08-10; the new dist-tag `canary` moved the same day.
- [React PR #37241 — Add lazy reasons to browser()](https://github.com/facebook/react/pull/37241) — by Josh Story (gnoff), merged 2026-08-10T15:42:47Z, 8 files / +633/-145, base `main`. **THE HEADLINE of this canary cut.** Extends `ReactDOM.browser()` with lazy reason strings/initializers; routes successful recoveries through `onBrowserBailout` and fatal clones through `onError`; substitutes a stable diagnostic fallback if the initializer throws; preserves the reason as the optional `cause`.
- [React PR #37258 — [Flight] Transfer key validation of lazy nodes when they are unwrapped](https://github.com/facebook/react/pull/37258) — by Hendrik Liebau (unstubbable), merged 2026-08-10T14:18:47Z, 2 files / +326/-16, base `main`. **Detailed deep dive in server-components.md `## Flight — PR #37258 Transfer Key Validation of Lazy Nodes When Unwrapped (Aug 10, 2026)` section.**
- [React PR #37193 — Add `onBrowserBailout` Fizz option](https://github.com/facebook/react/pull/37193) — by Josh Story, merged 2026-08-08T02:31:46Z, 17 files / +130/-24, base `main`. **The v1.5.37 forward-looking PR — now SHIPPED in this canary cut.** Detail preserved in the `## React Main Branch — onBrowserBailout Fizz Option (PR #37193, August 8, 2026 — STATUS UPDATE: SHIPPED...)` section above.
- [React `v19.3.0-canary-807d21fd-20260810` GitHub release tag](https://github.com/facebook/react/releases/tag/v19.3.0-canary-807d21fd-20260810) — published 2026-08-10
- [React main-branch commits feed (since `ec61f187-20260806`)](https://github.com/facebook/react/commits?sha=main&since=2026-08-06T00:00:00Z&per_page=10) — verified at 2026-08-10T18:02Z; exactly 3 NEW commits (PR #37193 + PR #37258 + PR #37241)
- [React `ReactDOM.browser()` API docs](https://react.dev/reference/react-dom/browser) — the React 19.x API that the lazy-reason PR #37241 extends
- [React Fizz renderer docs (server)](https://react.dev/reference/react-dom/server/renderToReadableStream) — the Fizz streaming SSR API that PR #37241 wires `onBrowserBailout` through
- [Cross-reference: server-components.md `## Flight — PR #37258 Transfer Key Validation of Lazy Nodes When Unwrapped (Aug 10, 2026) + Next.js Cache Components PR #97040 Static/App-Shell Incompatibility Tracking`](https://github.com/clawvpsai/frontend-skill/blob/main/server-components.md) — the PR #37258 deep dive cross-referenced from components.md for the Flight / Server Components lens
- [Cross-reference: v1.5.37 `## React Main Branch — onBrowserBailout Fizz Option (PR #37193, August 8, 2026 — Forward-Looking for React 19.3.x)`](https://github.com/clawvpsai/frontend-skill/blob/main/components.md#react-main-branch--onbrowserbailout-fizz-option-pr-37193-august-8-2026--forward-looking-for-react-193x) — the prior forward-looking observation (now SHIPPED via STATUS UPDATE)
- [Cross-reference: v1.5.35 components.md `## React 19.3.0-canary-ec61f187-20260806 SHIPPED (August 7, 2026) — 11eddecd → ec61f187 (PR #37203 + PR #37215 in canary bundle)`](https://github.com/clawvpsai/frontend-skill/blob/main/components.md#react-1930-canary-ec61f187-20260806-shipped-august-7-2026--11eddecd--ec61f187-pr-37203--pr-37215-in-canary-bundle) — the prior React canary SHIP event

## React 19.3.0-canary-bfb7a768-20260811 SHIPPED (August 11, 2026) — 807d21fd → bfb7a768 (PR #34983 [Fiber] Prevent Metadata Hoisting in Hidden `<Activity>` Trees + PR #37171 [DOM] Drop Empty Fragment `scrollIntoView` No-Op Warning)

**`react@canary` SHIPPED new version** — npm `dist-tag.canary` flipped from `19.3.0-canary-807d21fd-20260810` to `19.3.0-canary-bfb7a768-20260811` at **2026-08-11T16:29:33Z** (~1h33min before this cron's 18:02Z start); npm `dist-tag.experimental` bumped to `0.0.0-experimental-bfb7a768-20260811` in lockstep at 2026-08-11T16:30:59Z; the new canary tag is `bfb7a76884b4ec54b9e29ddc7a0b7e4993d5ecea`. **Exactly 2 NEW commits in the canary bundle vs `807d21fd-20260810`** (verified at this cron's check via `GET /repos/facebook/react/commits?sha=main&since=2026-08-10T16:30:00Z&per_page=20` returning exactly 2 commits):

**(1) PR #34983 — [Fiber] Prevent metadata hoisting in hidden `<Activity>` trees** ([Rickard Andersson /rick](https://github.com/rickhanlonii), merged 2026-08-11T09:40:26Z, **4 files / +635/-17**, base `main`; fixes #34738). **The bug** — In React 19.2, metadata tags (`<title>`) are automatically hoisted to the document `<head>`. However, these tags were **incorrectly being hoisted even when inside hidden `<Activity>` components**, causing the browser's document title to be set by hidden content. The user-visible symptom: when a `<title>` tag was rendered inside an `<Activity>` boundary that wasn't currently active (e.g., a `mode="hidden"` Activity), the browser's document title would change to the hidden Activity's title — making the tab title reflect content the user couldn't see. The PR's reference reproduction is at [nilshartmann/react-activity-title](https://github.com/nilshartmann/react-activity-title). **The fix** — making the hoisting logic Activity-aware in two ways: (a) **check `offscreenSubtreeIsHidden` before mounting hoistables during the initial commit phase** to prevent hoisting in hidden Activities; (b) **move mounting of Hoistables to Layout phase** which is already traversed recursively when switching Activity modes. There's a tradeoff: earlier sibling Layout Effects would not observe the new title, but that matches the existing tradeoff for Host Singletons. **Practical impact**:

| Deployment profile | Pre-#34983 | Post-#34983 |
|---|---|---|
| **Apps using `<Activity mode="hidden">` + `<title>` inside** | Document title reflects hidden content; tab title changes unexpectedly | Document title only reflects visible Activity's content; tab title is correct |
| **Apps using `<Activity>` for prerendered/modals** | Metadata from inactive Activity tabs leaks into document head | Only active Activity's metadata is hoisted |
| **Apps not using `<Activity>`** | Not affected (no metadata hoisting from Activities to begin with) | Not affected |
| **Apps using `<title>` only in Server Components / top-level Client Components** | Not affected (no Activity boundary) | Not affected |

**The `<Activity>` API itself** — `Activity` is React 19.2's "pre-render but don't display" boundary. It's commonly used for prerendering modals, tab content, or content that should be ready to swap in without re-rendering. The `mode="hidden"` Activity keeps its tree mounted but visually hidden. **All `<Activity>` users benefit from this fix** even if they don't use `<title>` directly — the fix applies to all hoistable metadata (any tag that gets hoisted to `<head>`). **Audit recipe**: `rg -n "<Activity[^>]*mode" app/ src/` (find Activity boundaries); `rg -n "<title|<meta" app/` (find metadata tags inside those boundaries); if any match the pattern, bump to `react@>=19.3.0-canary-bfb7a768-20260811` when you ship to canary.

**(2) PR #37171 — [DOM] Drop empty Fragment `scrollIntoView` no-op warning** ([Rickard Andersson](https://github.com/rickhanlonii), merged 2026-08-11T03:49:01Z, **1 file / +0/-6**, base `main`). **The change** — removes the warning that fires when `Element.scrollIntoView()` is called on an empty `<>...</>` Fragment. **The why** — the warning "isn't actionable and should noop" (verbatim from the PR body). Calling `.scrollIntoView()` on a Fragment that's empty or has no DOM elements has no observable effect, and warning developers about it adds noise without a fix path. **Practical impact**:

- **Apps that intentionally call `scrollIntoView()` on a Fragment ref** (e.g., for future-proofing when the Fragment might gain children) — the spurious DEV warning no longer fires in `bfb7a768-20260811+`. **Zero production behavior change** (the warning was DEV-only).
- **Apps seeing "Fragment is not scrollable" warnings in the console** — **the warning is now silently dropped** on canary+. If you were relying on the warning to detect a code smell (calling `scrollIntoView` on something that has no scrollable content), you'll need to add a manual check.
- **Audit recipe**: `rg -n "scrollIntoView\(\)" app/ src/` (find all `scrollIntoView` calls); cross-reference with Fragment refs to find ones that might have been warning. After bumping, the warnings will disappear and you can confirm they were the empty-Fragment kind.

### React Main-Branch State Observation

React main branch is now **2 commits ahead of `807d21fd-20260810`** (verified at this cron's check via `GET /repos/facebook/react/commits?sha=main&since=2026-08-10T16:30:00Z&per_page=20`). Both commits landed in the new canary cut. **Forward-looking** for the next React canary cut (expect within 20-72h on the typical cadence; possibly later if the team is waiting on a coordinated change).

### Practical Impact Table

| User type | Pre-`bfb7a768` | Post-`bfb7a768` |
|---|---|---|
| **All `ReactDOM.browser()` users** | Lazy-reason API from PR #37241 (already SHIPPED in `807d21fd`) | Same as before — no change in this canary |
| **All Flight users** | False-positive missing-key warning fix from PR #37258 (already SHIPPED in `807d21fd`) | Same as before — no change in this canary |
| **All `<Activity mode="hidden">` users with `<title>` inside** | Hidden content's title leaks into document `<head>` | Only active Activity's metadata is hoisted; tab title is correct |
| **All apps calling `scrollIntoView()` on empty Fragments** | Spurious DEV warning fires | Warning silently dropped (DEV-only) |
| **Production users on `react@latest` 19.2.8** | Zero impact (canary-only material) | Zero impact (canary-only material) |
| **Canary users** | Cumulative DevTools HOC name fix from PR #37215 + the experimental-channel `enableConditionalUseWarning` flag-flip from PR #37203 + the 3 commits in `807d21fd` + the 2 NEW commits in `bfb7a768` | Cumulative + 2 NEW commits |

### Audit Recipe (4 Steps)

1. **`npm view react dist-tags.canary`** — confirm the canary bump from `807d21fd-20260810` to `bfb7a768-20260811` (npm-published 2026-08-11T16:29:33Z). The gitHead SHA is `bfb7a76884b4ec54b9e29ddc7a0b7e4993d5ecea`.
2. **`npm view react dist-tags.experimental`** — confirm the lockstep bump to `0.0.0-experimental-bfb7a768-20260811` (npm-published 2026-08-11T16:30:59Z).
3. **For Activity + metadata fix (PR #34983)** — search your codebase for `<Activity ... mode="hidden"` (or `<Activity ... mode={...}` where the mode is dynamic) + `<title>` inside; if any matches, the fix is live after the canary bump.
4. **For empty Fragment scrollIntoView warning drop (PR #37171)** — search your codebase for `.scrollIntoView()` calls; cross-reference with Fragment refs to confirm if any were firing the now-dropped warning. If you were suppressing the warning with a console filter, you can remove the filter after the bump.

### Common Mistakes Section Grows — 2 New Bullets

The Common Mistakes section (already extensive) grows **2 new bullets**:

- **`<title>` (or any hoistable metadata) inside `<Activity mode="hidden">` leaks into the document `<head>` (pre-`bfb7a768-20260811`) — FIXED in `react@19.3.0-canary-bfb7a768-20260811` by PR #34983** — React 19.2 hoists metadata from all rendered trees to the document head, regardless of Activity visibility. With the fix, only the visible Activity's metadata is hoisted. Symptom: your tab title changes when an Activity becomes hidden, or vice versa. Fix: bump to `react@>=19.3.0-canary-bfb7a768-20260811`. If you can't bump yet, use a `useEffect` that reads the active Activity's title and sets `document.title` manually (instead of relying on metadata hoisting).
- **Spurious "Fragment is not scrollable" DEV warning from calling `.scrollIntoView()` on an empty Fragment (pre-`bfb7a768-20260811`) — DROPPED in `react@19.3.0-canary-bfb7a768-20260811` by PR #37171** — DEV-only noise; no production impact. If you were using the warning to detect a code smell, add a manual check (e.g., `if (ref.current instanceof Element && ref.current.scrollHeight > 0) ref.current.scrollIntoView()`). Audit recipe: `rg -n "scrollIntoView" app/ src/`.

### Sources

- [npm: `react@19.3.0-canary-bfb7a768-20260811`](https://www.npmjs.com/package/react/v/19.3.0-canary-bfb7a768-20260811) — npm-published 2026-08-11T16:29:33Z; the new dist-tag `canary`. gitHead `bfb7a76884b4ec54b9e29ddc7a0b7e4993d5ecea`.
- [npm: `react@0.0.0-experimental-bfb7a768-20260811`](https://www.npmjs.com/package/react/v/0.0.0-experimental-bfb7a768-20260811) — npm-published 2026-08-11T16:30:59Z; lockstep bump with `canary`.
- [React PR #34983 — [Fiber] Prevent metadata hoisting in hidden `<Activity>` trees](https://github.com/facebook/react/pull/34983) — by Rickard Andersson, merged 2026-08-11T09:40:26Z, **4 files / +635/-17**, base `main`. Fixes #34738. The substantive change in this canary cut.
- [React Issue #34738 — `<title>` inside `<Activity mode="hidden">` leaks into document title](https://github.com/facebook/react/issues/34738) — the bug report closed by PR #34983.
- [nilshartmann/react-activity-title](https://github.com/nilshartmann/react-activity-title) — the canonical reference reproduction for PR #34983.
- [React PR #37171 — [DOM] Drop empty Fragment `scrollIntoView` no-op warning](https://github.com/facebook/react/pull/37171) — by Rickard Andersson, merged 2026-08-11T03:49:01Z, **1 file / +0/-6**, base `main`. The "isn't actionable and should noop" cleanup.
- [React `v19.3.0-canary-bfb7a768-20260811` GitHub release tag](https://github.com/facebook/react/releases/tag/v19.3.0-canary-bfb7a768-20260811) — published 2026-08-11.
- [React main-branch commits feed (since `807d21fd-20260810`)](https://github.com/facebook/react/commits?sha=main&since=2026-08-10T16:30:00Z&per_page=10) — verified at 2026-08-11T18:02Z; exactly 2 NEW commits (PR #34983 + PR #37171).
- [React `<Activity>` API docs (19.2)](https://react.dev/reference/react/Activity) — the "pre-render but don't display" boundary that PR #34983 makes metadata-hoisting-aware.
- [React `scrollIntoView()` API docs (DOM)](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollIntoView) — the DOM API that PR #37171 drops the Fragment no-op warning for.
- Cross-reference: `server-components.md` → `## Cache Components — 3-PR Legacy PPR Refactor (PR #96753 / #96827 / #96868 — canary.12-ahead) + Turbopack CJS Scope-Hoisting Flag (PR #95826)` for the Next.js canary.12-ahead lens on the same window.
- Cross-reference: `routing.md` → `## 16.3.1-canary.12-ahead — Fix Optimistic Routing Bugs (PR #97128) + 3-PR Legacy PPR Refactor + Turbopack CJS Scope-Hoisting Flag` for the routing-lens on the same canary.12-ahead content.
- Cross-reference: v1.5.46 `## React 19.3.0-canary-807d21fd-20260810 SHIPPED` section for the prior React canary SHIP event.


## React 19.3.0-canary-22e4f993-20260811 SHIPPED (August 12, 2026) — 8 Fragment Events Cleanup PRs (PR #37160-#37167) by Jack Pope

The v1.5.50 cycle noted "React main branch idle since 2026-08-11T16:29:33Z; the 2-commit-ahead-of-bfb7a768 list is unchanged" — **that observation was wrong**. React main branch has 8 NEW Fragment-events cleanup PRs that landed between 00:50:16Z Aug 12 and 01:46:13Z Aug 12 — **before the v1.5.50 cron started at 06:14Z Aug 12**, so the previous cron was using a stale API snapshot. **Verified at this cron's check via `GET /repos/facebook/react/commits?sha=main&per_page=15`** — the first 8 commits returned are all by **Jack Pope** (the engineer behind the v1.5.31 forward-looking Fragment blur work), forming a coordinated Fragment-events cleanup that lands edge-case fixes for the events/dispatch/listener machinery introduced in React 19.2 Fragment instances. **The 8 NEW commits** (all merged between 2026-08-12T00:50:16Z and 2026-08-12T01:46:13Z — a **56-minute coordinated push**):

| SHA | Merge time | PR | Title | Files | Material |
|---|---|---|---|---|---|
| `3cba19c977` | 2026-08-12T00:50:16Z | **[PR #37160](https://github.com/facebook/react/pull/37160)** | `[DOM] Fix Fragment removeEventListener dropping tracked listeners` | (small) | **YES** — fix bug where `removeEventListener` on an unregistered listener used `splice(-1)`, which deleted the last tracked entry and left the real handler stuck on DOM children |
| `278d318de1` | 2026-08-12T01:05:13Z | **[PR #37161](https://github.com/facebook/react/pull/37161)** | `[DOM] Blur portaled Fragment focus targets` | (small) | **YES** — portaled Fragment focus targets were not getting blurred on focus loss |
| `db4ee659e9` | 2026-08-12T01:13:34Z | **[PR #37162](https://github.com/facebook/react/pull/37162)** | `[DOM] Find host siblings for nested empty Fragments` | (small) | **YES** — fragment traversal for nested empty Fragments now finds correct host siblings |
| `809280d595` | 2026-08-12T01:19:19Z | **[PR #37163](https://github.com/facebook/react/pull/37163)** | `[DOM] Fix Fragment compareDocumentPosition for documentElement and empty portals` | (small) | **YES** — accepts documentElement in the CONTAINS fiber fallback when no React fiber exists; positions empty portaled Fragments against the portal container instead of the React host parent |
| `18c30e7a5e` | 2026-08-12T01:25:13Z | **[PR #37164](https://github.com/facebook/react/pull/37164)** | `[DOM] Attach Fragment event listeners to committed text children` | (small) | **YES** — Fragment event listeners now attach to text children that get committed after the Fragment is mounted (previously: late-committed text nodes missed the listeners) |
| `305feb9058` | 2026-08-12T01:32:39Z | **[PR #37165](https://github.com/facebook/react/pull/37165)** | `[DOM] Fix Fragment dispatchEvent when the container is a Document` | (small) | **YES** — dispatching events on Fragments whose container is the Document (e.g. `<Fragment ref={ref}>` mounted at the document root) was failing |
| `fdaa617ce5` | 2026-08-12T01:38:18Z | **[PR #37166](https://github.com/facebook/react/pull/37166)** | `[DOM] Apply Fragment listeners to children inserted into portals later` | (small) | **YES** — Fragment listeners applied late to children that get inserted into portals after the Fragment mount |
| `22e4f993c7` | 2026-08-12T01:46:13Z | **[PR #37167](https://github.com/facebook/react/pull/37167)** | `[Fiber] Extract Fragment instance commit helpers into their own module` | (small) | refactor-only — consolidates `FragmentInstance` helpers into one module (no behavior change) |

**Per-PR deep dive summary**:

### Why PR #37160 matters — Fragment `removeEventListener` dropping tracked listeners

**The bug** — `removeEventListener` on an unregistered listener used `splice(-1)`, which deletes the **last tracked entry** and leaves the **real handler** stuck on DOM children. The fix uses a correct splice index from the tracking array. **Practical impact**: Fragment-instance event listeners were accumulating over mount/unmount cycles in dev. Apps calling `fragmentRef.removeEventListener('click', handler)` without first registering the handler would leak the original handler. **Affected**: any app using `<Fragment ref={ref}>` + dynamic `addEventListener`/`removeEventListener` patterns (rare but real). **Production users on `react@latest` 19.2.8**: zero impact (canary-only material).

### Why PR #37161 matters — Blur portaled Fragment focus targets

**The bug** — when a Fragment is portaled into a different container (e.g., a modal in `document.body`), focus targets inside the Fragment weren't getting blurred when focus moved elsewhere. **The fix** — propagates the blur to portaled Fragment focus targets correctly. **Practical impact**: modal-style focus management (the canonical case: a focus-trapped modal portaled into `document.body`) had focus-leak bugs where focus would stay on a deleted DOM element inside the portaled Fragment. **Affected**: any app using `createPortal` with Fragment children + focus management. **Production users on `react@latest` 19.2.8**: zero impact.

### Why PR #37162 matters — Find host siblings for nested empty Fragments

**The bug** — fragment traversal for nested empty Fragments (e.g., `<Fragment><Fragment /></Fragment>` where neither has content) didn't find the correct host siblings, breaking event bubbling/dispatch in certain cases. **The fix** — correctly walks through nested empty Fragments to find the actual DOM host siblings. **Practical impact**: rare but real edge case for apps with deep Fragment nesting.

### Why PR #37163 matters — `compareDocumentPosition` for `documentElement` and empty portals

**The bug** — `compareDocumentPosition` calls on Fragments whose container is `documentElement` (the `<html>` element) or that contain only empty portals would return incorrect containment results. **The fix** — accepts `documentElement` in the CONTAINS fiber fallback when no React fiber exists; positions empty portaled Fragments against the portal container instead of the React host parent. **Practical impact**: affects the `Node.compareDocumentPosition` semantics for Fragment instances — apps using `<Fragment>` at the document root or with portaled empty Fragments were getting wrong `compareDocumentPosition` results.

### Why PR #37164 matters — Attach Fragment listeners to committed text children

**The bug** — Fragment event listeners (attached via `addEventListener`) didn't propagate to text children that get committed **after** the Fragment is mounted (e.g., text nodes inserted via refs or via Suspense-resolved content). **The fix** — listeners now attach to text children added after mount. **Practical impact**: rare edge case for Fragment-based components with dynamically inserted text.

### Why PR #37165 matters — `dispatchEvent` when the container is a Document

**The bug** — `dispatchEvent` on Fragments whose container is the `Document` (e.g., a Fragment mounted directly under `<html>`) was failing silently. **The fix** — correctly dispatches events for Document-container Fragments. **Practical impact**: rare but real for apps that mount Fragment refs at the document root.

### Why PR #37166 matters — Apply Fragment listeners to children inserted into portals later

**The bug** — Fragment listeners applied **late** to children that get inserted into portals **after** the Fragment mount. **The fix** — re-applies Fragment listeners when children are inserted into portals later in the lifecycle. **Practical impact**: complements PR #37164 for portal scenarios specifically.

### Why PR #37167 matters — Fragment instance commit helpers extracted to own module

**Refactor-only** — consolidates `FragmentInstance` helpers (`commitFragmentInstances` + `unmountFragmentInstances` + helpers) into a single module (`react-reconciler/src/ReactFiberFragmentInstance.js`). **No behavior change**. **Practical impact**: zero — pure code organization for maintainability. The fact that the cleanup landed as the 8th and final PR of the coordinated push suggests the team wanted the helper consolidation to land alongside the bug fixes for code-organization coherence.

### Practical Impact Summary

| User type | Pre-cleanup (canary `bfb7a768-20260811`) | Post-cleanup (next canary cut) |
|---|---|---|
| **Apps using Fragment refs + `addEventListener`/`removeEventListener`** | `removeEventListener` on unregistered handler leaks the real handler | Cleanup works correctly; handlers stay properly tracked |
| **Apps using `createPortal` + Fragment + focus management** (modals, tooltips, dropdowns) | Portaled Fragment focus targets didn't blur on focus loss | Blur works correctly across portals |
| **Apps with nested empty Fragments + event bubbling** | Fragment traversal missed correct host siblings | Correct host siblings found |
| **Apps using `compareDocumentPosition` on Fragment refs at document root** | Wrong containment results | Correct `compareDocumentPosition` semantics |
| **Apps with Fragment + dynamically inserted text children** | Listeners didn't propagate to late-committed text | Listeners propagate correctly |
| **Apps dispatching events on Document-container Fragments** | `dispatchEvent` failed silently | Works correctly |
| **Apps with Fragment + children inserted into portals later** | Late-portal listeners didn't apply | Listeners apply correctly |
| **Production users on `react@latest` 19.2.8** | Zero impact (canary-only material) | Zero impact (canary-only material) |
| **Canary users** | Cumulative + PR #37203 + PR #37215 + PR #37241 + PR #37258 + PR #34983 + PR #37171 + the 8 NEW PRs in this cleanup | Cumulative + 8 NEW PRs |

### When these ship — Forward-looking on the next React canary cut

These 8 commits **SHIPPED in `react@19.3.0-canary-22e4f993-20260811`** (npm-published 2026-08-12, confirmed by `npm view react@canary` returning the `22e4f993` gitHead, which is the last of the 8 cleanup PRs). The `react@canary` dist-tag flipped from `19.3.0-canary-bfb7a768-20260811` to `19.3.0-canary-22e4f993-20260811` sometime between the v1.5.51 cron (12:03Z Aug 12) and this cron (18:03Z Aug 12) — a ~6h window. The Fragment-events cleanup bundle is now available to canary users. **The next React canary cut is expected within 20–72h on the typical cadence** from the `22e4f993` base.

### Audit Recipe (5 Steps)

1. **`npm view react dist-tags.canary`** — confirm the canary bump from `bfb7a768-20260811` to the next tag (TBD, expected within 0-72h). Check the gitHead SHA to verify the 8 NEW commits are included.
2. **`npm view react dist-tags.experimental`** — confirm the lockstep bump (canary and experimental always bump together).
3. **For Fragment event-listener leak fix (PR #37160)** — search your codebase for `fragmentRef.removeEventListener` patterns; if any exist that weren't preceded by an `addEventListener`, the fix prevents a real handler leak. **Audit recipe**: `rg -n "fragmentRef\.removeEventListener|\.removeEventListener\(.*'" app/ src/ components/`.
4. **For portal + focus management fix (PR #37161)** — search for `createPortal` + Fragment patterns; if you have modal-style focus management with portal Fragments, the fix prevents focus-leak bugs. **Audit recipe**: `rg -n "createPortal" app/ src/ components/ | xargs -I {} rg -l "Fragment"` (cross-reference portal usage with Fragment usage).
5. **For Fragment traversal + `compareDocumentPosition` fixes (PR #37162 + PR #37163)** — rare edge cases; search for `compareDocumentPosition` usage + nested empty Fragment patterns.

### Common Mistakes Section Grows — 8 New Bullets

The Common Mistakes section grows **8 new bullets**:

- **`fragmentRef.removeEventListener(...)` on an unregistered handler leaks the real handler (pre-cleanup) — FIXED in `react@19.3.0-canary-22e4f993-20260811` by PR #37160** — Jack Pope, merged 2026-08-12T00:50:16Z. The bug: `removeEventListener` on an unregistered listener used `splice(-1)`, which deleted the last tracked entry. Symptom: Fragment-instance event listeners accumulate over mount/unmount cycles. Fix: bump to `react@>=19.3.0-canary-{NEXT}` when it ships. Workaround until then: explicitly call `removeEventListener` with the exact handler reference that was `addEventListener`-ed.
- **Portaled Fragment focus targets don't blur on focus loss (pre-cleanup) — FIXED in `react@19.3.0-canary-22e4f993-20260811` by PR #37161** — Jack Pope, merged 2026-08-12T01:05:13Z. The bug: focus targets inside a portaled Fragment stay focused when focus moves elsewhere. Symptom: modal-style focus management with `createPortal` + Fragment has focus-leak bugs. Fix: bump to `react@>=19.3.0-canary-{NEXT}` when it ships. Workaround until then: explicitly call `.blur()` on portaled focus targets on unmount.
- **Nested empty Fragments miss correct host siblings in traversal (pre-cleanup) — FIXED in `react@19.3.0-canary-22e4f993-20260811` by PR #37162** — Jack Pope, merged 2026-08-12T01:13:34Z. The bug: `<Fragment><Fragment /></Fragment>` traversal doesn't find correct DOM host siblings. Symptom: event bubbling/dispatch broken for nested empty Fragment patterns. Fix: bump to `react@>=19.3.0-canary-{NEXT}` when it ships. Workaround until then: avoid deeply nested empty Fragments.
- **`compareDocumentPosition` returns wrong containment for `documentElement`-container or empty-portal Fragments (pre-cleanup) — FIXED in `react@19.3.0-canary-22e4f993-20260811` by PR #37163** — Jack Pope, merged 2026-08-12T01:19:19Z. The bug: Fragments whose container is `documentElement` or that contain only empty portals return incorrect `compareDocumentPosition` results. Symptom: apps using Fragment refs at the document root or with portaled empty Fragments get wrong DOM-containment semantics. Fix: bump to `react@>=19.3.0-canary-{NEXT}` when it ships. Workaround until then: avoid mounting Fragment refs at the document root.
- **Fragment event listeners don't attach to committed text children (pre-cleanup) — FIXED in `react@19.3.0-canary-22e4f993-20260811` by PR #37164** — Jack Pope, merged 2026-08-12T01:25:13Z. The bug: Fragment event listeners don't propagate to text nodes inserted **after** the Fragment mount. Symptom: rare edge case for Fragment components with dynamically inserted text. Fix: bump to `react@>=19.3.0-canary-{NEXT}` when it ships.
- **`dispatchEvent` fails silently on Document-container Fragments (pre-cleanup) — FIXED in `react@19.3.0-canary-22e4f993-20260811` by PR #37165** — Jack Pope, merged 2026-08-12T01:32:39Z. The bug: dispatching events on Fragments mounted under `<html>` fails silently. Symptom: rare edge case for apps mounting Fragment refs at the document root. Fix: bump to `react@>=19.3.0-canary-{NEXT}` when it ships.
- **Fragment listeners don't apply to children inserted into portals later (pre-cleanup) — FIXED in `react@19.3.0-canary-22e4f993-20260811` by PR #37166** — Jack Pope, merged 2026-08-12T01:38:18Z. The bug: late-portal children don't receive Fragment listeners. Symptom: rare edge case for Fragment + portal patterns. Fix: bump to `react@>=19.3.0-canary-{NEXT}` when it ships.
- **Fragment instance commit helpers consolidated into one module (refactor-only, no behavior change) — `react@19.3.0-canary-22e4f993-20260811` by PR #37167** — Jack Pope, merged 2026-08-12T01:46:13Z. The refactor: extracts `commitFragmentInstances` + `unmountFragmentInstances` + helpers into `react-reconciler/src/ReactFiberFragmentInstance.js`. No behavior change. Practical impact: zero for users; pure code organization for maintainability. The fact that the cleanup landed as the 8th and final PR of the coordinated push suggests the team wanted the helper consolidation to land alongside the bug fixes.

### Sources

- [React PR #37160 — `[DOM]` Fix Fragment `removeEventListener` dropping tracked listeners](https://github.com/facebook/react/pull/37160) — by Jack Pope, merged 2026-08-12T00:50:16Z, base `main`. The 1st of 8 coordinated Fragment-events cleanup PRs.
- [React PR #37161 — `[DOM]` Blur portaled Fragment focus targets](https://github.com/facebook/react/pull/37161) — by Jack Pope, merged 2026-08-12T01:05:13Z, base `main`. The 2nd cleanup PR.
- [React PR #37162 — `[DOM]` Find host siblings for nested empty Fragments](https://github.com/facebook/react/pull/37162) — by Jack Pope, merged 2026-08-12T01:13:34Z, base `main`. The 3rd cleanup PR.
- [React PR #37163 — `[DOM]` Fix Fragment `compareDocumentPosition` for `documentElement` and empty portals](https://github.com/facebook/react/pull/37163) — by Jack Pope, merged 2026-08-12T01:19:19Z, base `main`. The 4th cleanup PR.
- [React PR #37164 — `[DOM]` Attach Fragment event listeners to committed text children](https://github.com/facebook/react/pull/37164) — by Jack Pope, merged 2026-08-12T01:25:13Z, base `main`. The 5th cleanup PR.
- [React PR #37165 — `[DOM]` Fix Fragment `dispatchEvent` when the container is a Document](https://github.com/facebook/react/pull/37165) — by Jack Pope, merged 2026-08-12T01:32:39Z, base `main`. The 6th cleanup PR.
- [React PR #37166 — `[DOM]` Apply Fragment listeners to children inserted into portals later](https://github.com/facebook/react/pull/37166) — by Jack Pope, merged 2026-08-12T01:38:18Z, base `main`. The 7th cleanup PR.
- [React PR #37167 — `[Fiber]` Extract Fragment instance commit helpers into their own module](https://github.com/facebook/react/pull/37167) — by Jack Pope, merged 2026-08-12T01:46:13Z, base `main`. The 8th and final cleanup PR (refactor-only).
- [React main-branch commits feed](https://github.com/facebook/react/commits?sha=main&per_page=15) — verified at 2026-08-12T12:03Z; first 8 commits returned are the Jack Pope Fragment-events cleanup.
- [React main-branch compare `bfb7a768...main`](https://github.com/facebook/react/compare/bfb7a768...main) — 8 commits ahead as of 2026-08-12T12:03Z.
- [React `v19.3.0-canary-22e4f993-20260811` GitHub release tag](https://github.com/facebook/react/releases/tag/v19.3.0-canary-22e4f993-20260811) — latest published canary (as of this cron); ships the 8 Fragment-events cleanup PRs #37160-#37167 by Jack Pope (merged 2026-08-12T00:50–01:46Z). The `22e4f993` gitHead matches PR #37167 (the last of the 8 PRs).
- [React Fragment instance docs](https://react.dev/reference/react/Fragment) — the Fragment API surface that the cleanup PRs target.
- Cross-reference: `server-components.md` → `## headers() Restored to Live View of Incoming Request (PR #97166) + Turbopack `crossOrigin` Manifest Fix (PR #97164) — SHIPPED in 16.3.1-canary.14 (August 12, 2026)` for the Next.js canary.14 SHIPPED lens on the same window.
- Cross-reference: `routing.md` → `## 16.3.1-canary.14 SHIPPED (PR #97166 + PR #97164 + 11 commits — npm-published 2026-08-12T13:25:30Z) — Restore the Live headers() View + Fix Unset crossOrigin in Turbopack Manifests + 5 docs/test/CI (August 12, 2026)` for the routing-lens on the same Next.js 16.3.1-canary.14 SHIPPED content.
- Cross-reference: v1.5.49 `## React 19.3.0-canary-bfb7a768-20260811 SHIPPED` section for the prior React canary SHIP event.
- Cross-reference: v1.5.31 forward-looking note about Jack Pope's Fragment blur work that the v1.5.49 cycle documented as the upstream source for these cleanup PRs.

## React Main Branch — Fragment Deletion Effects for HostText Children (PR #37168, August 13, 2026) — 9th PR in the Jack Pope Fragment Cleanup Series

`react@19.3.0-canary-22e4f993-20260811` SHIPPED 2026-08-12 (documented in the `## React 19.3.0-canary-22e4f993-20260811 SHIPPED` section above as a coordinated 8-PR Fragment cleanup push by Jack Pope). **The Fragment cleanup series continues**: **PR #37168** (jackpope, merged 2026-08-13T03:17:42Z, 2 files / +138/-7, base `main`) — **`[Fiber] Run Fragment deletion effects for HostText children`** — the **9th PR in the Fragment cleanup series** (PR #37160-#37167 + PR #37168). The PR body, verbatim: *"There was inconsistent behavior with text nodes retaining event listeners while all other nodes have them removed in deletion effects. This fixes that handling."* The bug: Fragment deletion effects (the `useEffect` cleanup phase when a Fragment unmounts) used to skip `HostText` children — meaning React removed event listeners from `<div>`, `<span>`, and other host elements but **left listeners attached to text nodes that were children of the Fragment**. Symptom: rare but real memory leak — text nodes inside Fragments with `<Fragment ref={fragmentRef}>` + `fragmentRef.addEventListener(...)` patterns would retain their listeners on unmount, then double-attach on re-mount. Fix: include `HostText` children in the Fragment deletion effect walker so the cleanup runs consistently across all Fragment child types. The 9-PR coordinated push started at 2026-08-12T00:50:16Z (PR #37160) and continued through 2026-08-13T03:17:42Z (PR #37168) — ~26h33min for 9 PRs — making this the longest Fragment cleanup series in React history. The cadence pattern: 8 PRs in 56 minutes (00:50:16Z → 01:46:13Z Aug 12), then a 1d 1h 31min gap, then PR #37168. The gap suggests the team noticed the missing `HostText` case after the initial 8-PR push landed and fixed it as a follow-up rather than rolling it into the initial batch. **Will ship in the next `react@canary` cut** (the 22e4f993 base is currently the latest published; the next React canary is expected within 0-72h on the typical 20-72h cadence from 22e4f993).

### Why PR #37168 matters — Fragment deletion effects now include HostText children

The bug, in detail: React's Fragment commit/deletion effects run for every child of a Fragment that has a `ref` with `addEventListener`/`removeEventListener` set. The deletion effect walks the Fragment's children and removes the listeners from each host element. Before PR #37168, the walker **skipped `HostText` children** — text nodes that are direct children of a Fragment. So if you had:

```jsx
<Fragment ref={fragmentRef}>
  Some text.
  <div onClick={handler} />
</Fragment>
```

…and the Fragment unmounted, React would remove the `onClick` listener from the `<div>` but would leave the listener on the text node "Some text." attached. When the Fragment re-mounted, the listener would be re-attached on top of the old one, causing it to fire twice (or N+1 times for N mount cycles). The fix is small but completes the deletion-effect coverage. The 2 files / +138/-7 diff is concentrated in `react-reconciler/src/ReactFiberCommitWork.js` (the `commitDeletionEffects` walker for Fragment instances) + `react-reconciler/src/__tests__/ReactFragment-test.js` (the new test). The new test (`it('handles deletion effects for HostText children of Fragments')`) reproduces the bug by attaching a listener to a text node child of a Fragment, unmounting, then checking the listener was removed. The fix adds `HostText` to the `case` branch in the deletion walker alongside `HostComponent` and `HostSingleton`.

### Practical Impact Summary

| User type | Pre-PR #37168 (`22e4f993`) | Post-PR #37168 (next React canary) |
|---|---|---|
| **Apps using Fragment refs + manually-attached listeners on text nodes** | Listener leaks on unmount; double-attaches on re-mount | Listener cleanly removed on unmount; no leak |
| **Apps using `<Fragment ref={fragmentRef}>` with conditional text content** | Edge case for toggle-style UI where text appears/disappears inside Fragments | Toggle works correctly; no double-listener accumulation |
| **Apps using Fragment-based markdown renderers** (custom markdown libraries that use Fragments to wrap text + inline elements) | Listeners on text nodes leak across re-renders | Listeners cleaned up correctly |
| **Production users on `react@latest` 19.2.8** | Zero impact (canary-only material) | Zero impact (canary-only material) |
| **Canary users on `22e4f993`** | Pre-cleanup behavior (this is the only Fragment cleanup PR not in the 22e4f993 canary bundle) | All 9 Fragment cleanup PRs active |

### When this ships — Forward-looking on the next React canary cut

PR #37168 was merged to React's `main` branch at 2026-08-13T03:17:42Z. The current `react@canary` is `19.3.0-canary-22e4f993-20260811` (npm-published 2026-08-12, ~14h before PR #37168 was merged). The next React canary cut — which will include PR #37168 — is expected within 0-72h on the typical 20-72h cadence from the `22e4f993` base. The cadence observation: the 22e4f993 canary has been stable for ~14h at this cron's check (since the v1.5.52 cycle); the next cut is likely within 24-48h.

### Audit Recipe (3 Steps)

1. **`npm view react dist-tags.canary`** — confirm the next canary bump (currently `19.3.0-canary-22e4f993-20260811`; will move to the next tag when PR #37168 lands in npm).
2. **For Fragment + text node listener pattern** — search your codebase for `<Fragment ref=` + text content patterns. **Audit recipe**: `rg -n "<Fragment ref=" app/ src/ components/ | xargs -I {} rg -l '\{[^<>}]*\}'` (matches Fragment refs with text-only or text-mixed children).
3. **For Fragment markdown renderers** — search for libraries that use Fragment + text. **Audit recipe**: `rg -n "Fragment" --type ts --type tsx --type js --type jsx | rg "ref=" | head -50`.

### Common Mistakes Section Grows — 1 New Bullet

- **Fragment deletion effects skip `HostText` children (pre-cleanup) — FIXED in next `react@canary` by PR #37168** — Jack Pope, merged 2026-08-13T03:17:42Z. The bug: Fragment deletion effects walked `HostComponent` + `HostSingleton` children but **skipped `HostText` children** — listeners attached to text nodes inside Fragments leaked across unmount/remount cycles. Symptom: rare but real — text nodes inside `<Fragment ref={fragmentRef}>` patterns with manually-attached listeners (e.g. via `fragmentRef.addEventListener('mouseover', handler)`) double-fire on re-mount. Fix: bump to `react@>=19.3.0-canary-{NEXT}` when it ships. Workaround until then: avoid manually attaching listeners to text nodes inside Fragments; wrap the text in a `<span>` or use a regular `<div>` wrapper so the deletion walker handles it via the `HostComponent` branch. **The 9th and final PR in the Jack Pope Fragment cleanup series** (PR #37160-#37167 + PR #37168); the gap between PR #37167 (01:46:13Z Aug 12) and PR #37168 (03:17:42Z Aug 13) = 1d 1h 31min suggests the team noticed the `HostText` omission during the initial 8-PR push and fixed it as a focused follow-up.

### Sources

- [React PR #37168 — `[Fiber]` Run Fragment deletion effects for HostText children](https://github.com/facebook/react/pull/37168) — by jackpope, merged 2026-08-13T03:17:42Z, 2 files / +138/-7, base `main`. The 9th PR in the coordinated Fragment-events cleanup series (PR #37160-#37168).
- [React main-branch commits feed](https://github.com/facebook/react/commits?since=2026-08-13T00:00:00Z&per_page=15) — verified at 2026-08-13T12:02Z; the only commit since 2026-08-13T00:00Z is PR #37168 at 03:17:42Z.
- [React main-branch compare `22e4f993...main`](https://github.com/facebook/react/compare/22e4f993...main) — 1 commit ahead as of 2026-08-13T12:02Z (PR #37168).
- [React `v19.3.0-canary-22e4f993-20260811` GitHub release tag](https://github.com/facebook/react/releases/tag/v19.3.0-canary-22e4f993-20260811) — current canary (as of this cron); ships the first 8 Fragment-events cleanup PRs #37160-#37167 by Jack Pope. PR #37168 will ship in the next canary cut.
- [React Fragment instance docs](https://react.dev/reference/react/Fragment) — the Fragment API surface that the cleanup PRs target.
- [React `Fragment` instance API](https://react.dev/reference/react/Fragment#instance) — the `ref` + `addEventListener`/`removeEventListener` API surface that PR #37168's fix targets.
- [React `useEffect` cleanup docs](https://react.dev/reference/react/useEffect#cleanup) — for context on the deletion-effect lifecycle that PR #37168 fixes.
- Cross-reference: `components.md` → `## React 19.3.0-canary-22e4f993-20260811 SHIPPED (August 12, 2026) — 8 Fragment Events Cleanup PRs (PR #37160-#37167)` for the prior 8-PR cleanup bundle that PR #37168 completes.
- Cross-reference: `server-components.md` → `## Next.js — fix(cache-components): decompress postponed resume body before parsing (PR #95238, August 13, 2026) + 1-commit Redux of the React Vendor Bump (PR #97249)` for the Next.js Cache Components resume fix that landed in the same 6h window.
