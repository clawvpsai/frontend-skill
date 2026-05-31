# Patterns — Composite Recipes for Common Flows

## Turbopack (Next.js 16)

Turbopack is Next.js's Rust-based bundler. In Next.js 16, Turbopack is stable for both development and production builds. It replaces Webpack for development, offering significantly faster hot module replacement (HMR) and cold start times.

```bash
# Use Turbopack in development (Next.js 16 default)
npm run dev

# Force Turbopack explicitly
next dev --turbopack

# Force Webpack if needed
next dev --webpack
```

**Current status (Next.js 16.2.6):**
- ✅ Stable for development — hot reload, fast refresh, error overlay
- ✅ Stable for production builds (`next build --turbopack`) — 2x–5x faster builds than Webpack; used in production on vercel.com and nextjs.org serving 1.2B+ requests
- ✅ App Router and Pages Router both supported
- ⚠️ Some edge-case features may still differ from Webpack; check the [migration guide](https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopack)

**Enable Turbopack production builds:**
```bash
# In next.config.ts — enable Turbopack for production builds
# next build --turbo  (or set environment variable NEXT_BUILD_USE_TURBOPACK=1)
```

**When using Turbopack:**
```ts
// next.config.ts — Turbopack config options
const nextConfig: NextConfig = {
  turbopack: {
    // Enable experimental features if needed
    // experimental: { ... }
  },
}
```

## Search with Debounce + URL Sync + React Query

```tsx
// components/post-search.tsx
'use client'

import { useCallback, useEffect } from 'react'
import { useRouter, useSearchParams } from 'next/navigation'
import { useQuery } from '@tanstack/react-query'
import { useDebouncedValue } from '@/hooks/use-debounced-value'

export function PostSearch() {
  const router = useRouter()
  const searchParams = useSearchParams()
  const query = searchParams.get('q') ?? ''
  
  // Debounce input to avoid excessive API calls
  const [debouncedQuery, isDebouncing] = useDebouncedValue(query, 300)
  
  // React Query for caching + loading
  const { data: results, isLoading } = useQuery({
    queryKey: ['posts', 'search', debouncedQuery],
    queryFn: () => searchPosts(debouncedQuery),
    enabled: debouncedQuery.length > 2,  // Don't search until 3+ chars
  })
  
  // Sync input to URL
  function handleSearch(e: React.ChangeEvent<HTMLInputElement>) {
    const value = e.target.value
    const params = new URLSearchParams(searchParams)
    if (value) {
      params.set('q', value)
    } else {
      params.delete('q')
    }
    router.push(`?${params.toString()}`, { scroll: false })
  }
  
  return (
    <div>
      <input 
        value={query} 
        onChange={handleSearch}
        placeholder="Search posts..."
      />
      {isLoading && <Spinner />}
      {results && <PostList posts={results} />}
    </div>
  )
}
```

### Debounce Hook

```ts
// hooks/use-debounced-value.ts
import { useState, useEffect } from 'react'

export function useDebouncedValue<T>(value: T, delay: number): [T, boolean] {
  const [debouncedValue, setDebouncedValue] = useState(value)
  const [isDebouncing, setIsDebouncing] = useState(false)
  
  useEffect(() => {
    setIsDebouncing(true)
    const timer = setTimeout(() => {
      setDebouncedValue(value)
      setIsDebouncing(false)
    }, delay)
    
    return () => clearTimeout(timer)
  }, [value, delay])
  
  return [debouncedValue, isDebouncing]
}
```

## Multi-Step Form with React Hook Form + Zod

```tsx
// components/multi-step-form.tsx
'use client'

import { useState } from 'react'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { Button } from '@/components/ui/button'
import { Form } from '@/components/ui/form'

const step1Schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Min 8 characters'),
})

const step2Schema = z.object({
  firstName: z.string().min(1),
  lastName: z.string().min(1),
  age: z.coerce.number().int().min(18),
})

const fullSchema = step1Schema.merge(step2Schema)
type FormData = z.infer<typeof fullSchema>

export function MultiStepForm() {
  const [step, setStep] = useState(1)
  const form = useForm<FormData>({
    resolver: zodResolver(step === 1 ? step1Schema : fullSchema),
    mode: 'onChange',
  })
  
  async function handleNext() {
    const fields = step === 1 ? ['email', 'password'] : ['firstName', 'lastName', 'age']
    const valid = await form.trigger(fields as any)
    if (valid) setStep(s => s + 1)
  }
  
  async function handleSubmit(data: FormData) {
    await fetch('/api/register', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    })
  }
  
  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(handleSubmit)}>
        {step === 1 && (
          <fieldset className="space-y-4">
            <Form.Field control={form.control} name="email">
              <Form.Item>
                <Form.Label>Email</Form.Label>
                <Form.Control><input {...form.register('email')} /></Form.Control>
                <Form.Message />
              </Form.Item>
            </Form.Field>
            <Form.Field control={form.control} name="password">
              <Form.Item>
                <Form.Label>Password</Form.Label>
                <Form.Control><input type="password" {...form.register('password')} /></Form.Control>
                <Form.Message />
              </Form.Item>
            </Form.Field>
            <Button type="button" onClick={handleNext}>Next</Button>
          </fieldset>
        )}
        
        {step === 2 && (
          <fieldset className="space-y-4">
            <Form.Field control={form.control} name="firstName">
              <Form.Item><Form.Label>First Name</Form.Label>
                <Form.Control><input {...form.register('firstName')} /></Form.Control>
                <Form.Message />
              </Form.Item>
            </Form.Field>
            <Form.Field control={form.control} name="lastName">
              <Form.Item><Form.Label>Last Name</Form.Label>
                <Form.Control><input {...form.register('lastName')} /></Form.Control>
                <Form.Message />
              </Form.Item>
            </Form.Field>
            <Form.Field control={form.control} name="age">
              <Form.Item><Form.Label>Age</Form.Label>
                <Form.Control><input type="number" {...form.register('age')} /></Form.Control>
                <Form.Message />
              </Form.Item>
            </Form.Field>
            <div className="flex gap-2">
              <Button type="button" variant="outline" onClick={() => setStep(1)}>Back</Button>
              <Button type="submit">Register</Button>
            </div>
          </fieldset>
        )}
      </form>
    </Form>
  )
}
```

## Server Component → Client Component Data Passing

### Pattern: Pass Serializable Data

```tsx
// app/users/[id]/page.tsx — server component
import { db } from '@/lib/db'
import { UserProfileClient } from '@/components/user-profile-client'

export default async function UserPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const user = await db.user.findUnique({ where: { id } })
  
  if (!user) notFound()
  
  // Pass plain serializable data to client component
  return (
    <UserProfileClient 
      user={{
        id: user.id,
        name: user.name,
        email: user.email,
        role: user.role,
        joinedAt: user.createdAt.toISOString(),  // Convert Date to string
      }}
    />
  )
}
```

```tsx
// components/user-profile-client.tsx
'use client'

import { useState } from 'react'

interface UserProfileProps {
  user: {
    id: string
    name: string
    email: string
    role: string
    joinedAt: string
  }
}

export function UserProfileClient({ user }: UserProfileProps) {
  const [following, setFollowing] = useState(false)
  // ...
}
```

### Pattern: Pass a Promise (React 19 `use()`)

```tsx
// app/users/[id]/page.tsx — server component
import { getUser } from '@/lib/api'

export default async function UserPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  
  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserProfile userPromise={getUser(id)} />
    </Suspense>
  )
}
```

```tsx
// components/user-profile.tsx — client component
'use client'

import { use } from 'react'

export function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise)  // Suspends until resolved
  return <div>{user.name}</div>
}
```

## Infinite Scroll with React Query

```tsx
// components/post-feed.tsx
'use client'

import { useInfiniteQuery } from '@tanstack/react-query'
import { useCallback } from 'react'
import { useInView } from 'react-intersection-observer'

export function PostFeed() {
  const { ref, inView } = useInView()
  
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam = 0 }) => fetchPosts({ cursor: pageParam }),
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    initialPageParam: 0,
  })
  
  useEffect(() => {
    if (inView && hasNextPage) {
      fetchNextPage()
    }
  }, [inView, hasNextPage, fetchNextPage])
  
  return (
    <div>
      {data?.pages.flatMap(page => page.posts).map(post => (
        <PostCard key={post.id} post={post} />
      ))}
      
      <div ref={ref}>
        {isFetchingNextPage && <Spinner />}
        {!hasNextPage && <p>No more posts</p>}
      </div>
    </div>
  )
}
```

## Optimistic UI with `useOptimistic` (React 19)

React 19's `useOptimistic` provides a declarative way to show immediate UI feedback while a mutation is pending:

```tsx
'use client'

import { useOptimistic } from 'react'
import { updatePost } from '@/app/actions'

interface Post {
  id: string
  content: string
  likes: number
}

export function LikeButton({ post }: { post: Post }) {
  const [optimisticPost, addOptimistic] = useOptimistic(
    post,
    (state, newLikes: number) => ({ ...state, likes: newLikes })
  )

  async function handleLike() {
    addOptimistic(post.likes + 1)
    try {
      await updatePost(post.id, { likes: optimisticPost.likes + 1 })
    } catch {
      // React reverts on error automatically
    }
  }

  return (
    <button onClick={handleLike}>
      {optimisticPost.likes} likes
    </button>
  )
}
```

**When to use `useOptimistic` vs React Query's optimistic update:**
- `useOptimistic` — simpler, for single-item optimistic updates (likes, follows, toggles)
- React Query `onMutate` — for complex list mutations (add/remove from lists), full rollback control

## File Upload with Preview

```tsx
// components/image-upload.tsx
'use client'

import { useState } from 'react'
import { useForm } from 'react-hook-form'
import Image from 'next/image'

export function ImageUpload() {
  const [preview, setPreview] = useState<string | null>(null)
  const [uploading, setUploading] = useState(false)
  
  function handleFileChange(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0]
    if (!file) return
    
    // Create preview
    const url = URL.createObjectURL(file)
    setPreview(url)
  }
  
  async function handleUpload(e: React.ChangeEvent<HTMLFormElement>) {
    e.preventDefault()
    const formData = new FormData(e.currentTarget)
    const file = formData.get('file') as File
    
    setUploading(true)
    try {
      const res = await fetch('/api/upload', {
        method: 'POST',
        body: formData,
      })
      const { url } = await res.json()
      console.log('Uploaded to:', url)
    } finally {
      setUploading(false)
    }
  }
  
  return (
    <form onSubmit={handleUpload}>
      <input type="file" name="file" accept="image/*" onChange={handleFileChange} />
      {preview && (
        <div className="mt-4 relative w-64 h-64">
          <Image src={preview} alt="Preview" fill className="object-cover" />
        </div>
      )}
      <button type="submit" disabled={uploading}>
        {uploading ? 'Uploading...' : 'Upload'}
      </button>
    </form>
  )
}
```


## Deferred Operations with `after()` (Next.js 16)

Next.js 15+ introduces `after()` from `next/server` — a way to run code **after** the response is sent to the user. This is ideal for non-critical operations that shouldn't block the response.

```tsx
// app/dashboard/page.tsx
import { after } from 'next/server'
import { logAnalytics, sendSlackNotification } from '@/lib/analytics'

export default async function DashboardPage() {
  const data = await getDashboardData()
  
  // Run AFTER the response is sent — doesn't delay the page
  after(async () => {
    await logAnalytics({ 
      page: '/dashboard', 
      userId: data.user.id,
      timestamp: new Date().toISOString()
    })
  })
  
  after(async () => {
    if (data.user.isNew) {
      await sendSlackNotification(`New user signed up: ${data.user.email}`)
    }
  })
  
  return <Dashboard data={data} />
}
```

**When to use `after()`:**

| Use Case | Example |
|---|---|
| Analytics / logging | `logAnalytics()`, `trackEvent()` |
| Notifications | Send Slack/email after user action |
| Cache warming | Pre-fetch related data after page load |
| Background jobs | Trigger async jobs without blocking response |

**Rules:**
- `after()` runs **after** the response is streamed — the user sees the page immediately
- It still runs on the server — it has access to server-only resources (DB, filesystem)
- If `after()` throws, the page still renders normally — errors don't affect the user
- Multiple `after()` calls run in parallel

```tsx
// Real-world: log page view without slowing down response
import { after } from 'next/server'

export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  const post = await getPost(slug)
  
  after(async () => {
    // Non-blocking — doesn't affect TTFB
    await fetch('/api/analytics', {
      method: 'POST',
      body: JSON.stringify({ path: `/blog/${slug}`, referrer: headers().get('referer') }),
    })
  })
  
  return <Article post={post} />
}
```

**`after()` vs alternatives:**

| Pattern | Use When |
|---|---|
| `after()` (Next.js 15) | Server-side ops after response, inside Next.js |
| `queue` (Redis/Bull) | Background jobs that need durability across restarts |
| `useEffect` (client) | Client-side ops like analytics after hydration |

## React 19 Compiler (React Compiler)

The React Compiler (stable in React 19.2) is a build-time tool that automatically optimizes component rendering — eliminating the need for manual `useMemo` and `useCallback` in most cases. It analyzes your code and generates memoized versions of components and hooks.

**How it works:** The compiler looks at your component code and determines which values are "expensive" and which are stable. It wraps those values in `useMemo`/`useCallback` automatically, so you don't have to.

### Enable in Next.js

Next.js uses an SWC-based implementation that's more efficient than the standalone Babel plugin — it only runs on files that actually use JSX or React hooks.

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  reactCompiler: true,  // ✅ Auto-compile all eligible components
}
```

**With granular control** (opt-in mode — only compile explicitly annotated components):

```ts
const nextConfig: NextConfig = {
  reactCompiler: {
    mode: 'annotation',  // Only compile components/hooks with "use memo" directive
  },
}
```

### Usage in Components

With the compiler enabled, you can write components naturally without manual memoization:

```tsx
// ❌ Before React Compiler — manual memoization required
function ExpensiveList({ items, filter }: Props) {
  const filtered = useMemo(
    () => items.filter(i => i.category === filter),
    [items, filter]
  )
  const sorted = useMemo(
    () => [...filtered].sort((a, b) => a.name.localeCompare(b.name)),
    [filtered]
  )
  return <div>{sorted.map(item => <Item key={item.id} {...item} />)}</div>
}

function Parent() {
  const [count, setCount] = useState(0)
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveList items={data} filter="active" />
    </div>
  )
}
```

```tsx
// ✅ With React Compiler — write naturally, compiler handles memoization
function ExpensiveList({ items, filter }: Props) {
  const filtered = items.filter(i => i.category === filter)
  const sorted = [...filtered].sort((a, b) => a.name.localeCompare(b.name))
  return <div>{sorted.map(item => <Item key={item.id} {...item} />)}</div>
}
```

### ESLint Plugin (Recommended Alongside Compiler)

The `eslint-plugin-react-compiler` catches code patterns that violate compiler rules:

```bash
npm install -D eslint-plugin-react-compiler
```

```js
// eslint.config.mjs
import reactCompiler from 'eslint-plugin-react-compiler'

export default [
  {
    plugins: {
      'react-compiler': reactCompiler,
      // Or via flat config rules:
    },
    rules: {
      'react-compiler/react-compiler': 'warn',  // or 'error' for strict
    },
  },
]
```

**Note:** Even with the React Compiler, keep the ESLint plugin — it catches cases where the compiler can't optimize and tells you why.

### What the Compiler Handles

| Pattern | Compiler Action |
|---|---|
| `useMemo(() => expr)` | Included automatically |
| `useCallback(fn)` | Included automatically |
| `React.memo(Component)` | Often unnecessary with compiler |
| `[] deps that never change` | Detected and preserved |
| Mutations of props/state | Flagged as errors |

### What the Compiler Requires

The compiler works when your code follows React's rules:
- **No mutation of props or state** — treat all values as immutable
- **Stable hook signatures** — don't call hooks conditionally
- **Functions called during render must be pure** — same inputs → same outputs

```tsx
// ❌ Compiler skips this component — it mutates a prop
function BadComponent({ items }: Props) {
  items.push({ id: 'new' })  // Mutation — compiler skips entire component
  return <List items={items} />
}

// ✅ Compiler optimizes this component — pure render
function GoodComponent({ items }: Props) {
  const newItems = [...items, { id: 'new' }]  // Creates new array
  return <List items={newItems} />
}
```

### When to Use the Compiler

| Scenario | Recommendation |
|---|---|
| New project | ✅ Enable by default |
| Existing codebase with `useMemo`/`useCallback` | ✅ Enable, then gradually remove manual memoization |
| Codebase with frequent prop/state mutations | ⚠️ Fix mutation patterns first |
| Strict performance requirements | ✅ Enable + keep ESLint plugin |

**Sources:**
- [React Compiler docs](https://react.dev/reference/react-compiler)
- [Next.js React Compiler config](https://nextjs.org/docs/app/api-reference/config/next-config-js/reactCompiler)
- [React Compiler configuration](https://react.dev/reference/react-compiler/configuration)



## React 19.2 New Primitives (October 2025)

React 19.2 stabilizes several previously-experimental APIs and introduces new primitives for fine-grained reactivity and loading states.

### `<Activity />` — Declarative Loading States

`<Activity>` is React 19.2's new component for declaratively showing loading/activity states. It replaces the need for manual `isPending` state tracking when you want to detect any pending state in a subtree.

**Key props:**
- `detection` — `'subtree'` (default) tracks any state change in the entire subtree; `'elements'` tracks only mounted interactive elements that are pending
- `children` — render prop receiving `{ isActivity: boolean, activity: ActivityDetails }`

**`detection` variants explained:**

| Mode | Behavior | Use When |
|---|---|---|
| `'subtree'` | Detects any pending state anywhere in the rendered subtree | General-purpose — default for most cases |
| `'elements'` | Only tracks interactive elements that are actively pending | Fine-grained — button spinner on submit, toggle loading |

**Practical example — Server Action with `<Activity>`:**

```tsx
// app/actions.ts
'use server'

import { revalidateTag } from 'next/cache'

export async function publishPost(postId: string) {
  await db.post.update({ where: { id: postId }, data: { published: true } })
  revalidateTag('posts')
}
```

```tsx
// components/post-actions.tsx
'use client'

import { Activity } from 'react'
import { publishPost } from '@/app/actions'
import { useActionState } from 'react'

const initialState = { error: null as string | null }

function PublishButton({ postId }: { postId: string }) {
  // useActionState — React 19 hook for Server Action state
  const [state, formAction, isPending] = useActionState(
    async (prev: typeof initialState, formData: FormData) => {
      try {
        await publishPost(formData.get('postId') as string)
        return { error: null }
      } catch {
        return { error: 'Failed to publish' }
      }
    },
    initialState
  )

  return (
    <form action={formAction}>
      <input type="hidden" name="postId" value={postId} />
      {/* Wrap in Activity to detect isPending from useActionState */}
      <Activity detection="elements">
        {({ isActivity }) => (
          <button
            type="submit"
            disabled={isActivity}
            className={isActivity ? 'opacity-50' : ''}
          >
            {isActivity ? 'Publishing...' : 'Publish'}
          </button>
        )}
      </Activity>
      {state.error && <p className="text-sm text-destructive">{state.error}</p>}
    </form>
  )
}
```

**When to use `<Activity>` vs alternatives:**

| Pattern | Best Use |
|---|---|
| `<Activity detection="subtree">` | Detect any loading in a large section (entire sidebar, full form) |
| `<Activity detection="elements">` | Single interactive element — button loading state |
| `useFormStatus` | Form-level pending state — submit button only |
| `isPending` (React Query) | Query/mutation-specific loading — for one particular async operation |
| Manual `isLoading` state | When you need precise control or multiple distinct loading states |

**Common `<Activity>` mistakes:**
- Wrapping too large a tree — `isActivity` becomes true for any pending child, causing cascading loading states
- Using `'elements'` when you need to detect background loading — `'elements'` only tracks mounted pending elements
- Confusing `<Activity>` with a spinner — it's a detection mechanism, not a UI component; use it to conditionally render your own loading UI

### `useEffectEvent` — Stable Event Handlers in Effects (React 19.2)

`useEffectEvent` was previously experimental (`useEffectEvent` from `react experimental`) and is now **stable** in React 19.2. It solves the "stale closure" problem in `useEffect` — you can define event handlers inside `useEffect` that always see the latest values without needing them in the dependency array:

```tsx
'use client'

import { useEffect, useEffectEvent } from 'react'

function ChatRoom({ roomId }: { roomId: string }) {
  const [status, setStatus] = useState<'connected' | 'disconnected'>('disconnected')

  // ❌ Problem: messageHandler captures stale roomId if not in deps
  // useEffect(() => {
  //   const ws = new WebSocket(`wss://chat/${roomId}`)
  //   ws.onmessage = (event) => {
  //     // roomId is stale here if deps aren't managed carefully
  //     handleMessage(roomId, event.data)
  //   }
  // }, [roomId])

  // ✅ useEffectEvent — event handler that always sees current roomId
  const handleMessage = useEffectEvent((message: string) => {
    // This function can access current props/state without them being deps
    console.log(`[Room ${roomId}]:`, message)
  })

  useEffect(() => {
    const ws = new WebSocket(`wss://chat/${roomId}`)
    setStatus('connected')

    ws.onmessage = (event) => {
      handleMessage(event.data) // Always sees current roomId
    }

    return () => {
      ws.close()
      setStatus('disconnected')
    }
  }, [roomId])

  return <div className={status === 'connected' ? 'text-green-500' : 'text-gray-400'}>{status}</div>
}
```

**Why `useEffectEvent` instead of just putting the function inside the effect?**
- Putting logic inside `useEffect` makes it harder to test and reason about
- `useEffectEvent` lets you keep the effect focused on subscription lifecycle while extracting event handlers that need current values

**Rules:**
- `useEffectEvent` can only be called inside `useEffect`
- The event handler it returns can reference any current value without causing the effect to re-run
- Unlike `useCallback`, there's no dependency array — it captures values at call time, not render time

### `cacheSignal` — Reactive Cached Values

`cacheSignal` creates a signal whose value is cached and recomputed only when accessed, not on every render:

```tsx
import { cacheSignal } from 'react'

// Creates a cached computation — only recomputes when deps change
const useUserData = cacheSignal((userId: string) => {
  console.log('Fetching user:', userId) // Runs once per unique userId
  return fetchUser(userId)
})

function UserProfile({ userId }: { userId: string }) {
  // userData is cached — accessing multiple times doesn't trigger re-fetch
  const userData = useUserData(userId)
  const displayName = useUserData(userId) // Same cache, no re-fetch

  return <div>{userData.name}</div>
}
```

**When to use `cacheSignal`:**
- Expensive computations that depend on props but shouldn't re-run on every parent render
- Derived data that's shared across multiple component instances
- Similar to `useMemo` but the cache is tied to the signal's internal tracking rather than a dependency array

**vs `useMemo`:**
- `useMemo` — explicit dependency array; recomputes when deps change
- `cacheSignal` — lazy evaluation; only recomputes when `.get()` is called with a new combination of arguments

### `cache()` — Function Memoization (React 19.2)

`cache()` memoizes a function's return value based on its arguments — similar to `useMemo` but for standalone functions:

```tsx
import { cache } from 'react'

// Memoized function — same args return cached result
const getUserById = cache(async (id: string) => {
  console.log('API call for:', id) // Only runs once per unique id
  const res = await fetch(`/api/users/${id}`)
  return res.json()
})

// In server components:
async function UserCard({ id }: { id: string }) {
  const user = await getUserById(id) // First call: fetches
  // Next UserCard with same id: returns cached result
  return <div>{user.name}</div>
}

// In client components:
function UserCard({ id }: { id: string }) {
  const userPromise = getUserById(id)
  const user = use(userPromise) // Suspends until resolved
  return <div>{user.name}</div>
}
```

**When to use `cache()`:**
- Shared data fetching functions used across multiple components
- Server-side data access in RSC patterns where you want request-level memoization
- Replacing ad-hoc memoization patterns with a cleaner API

**Note:** `cache()` in React 19.2 is distinct from `use cache` in Next.js 16. React's `cache()` is a general-purpose memoization primitive; Next.js `use cache` is a framework-level caching directive with server-side persistence.

### Combined Example: Activity + Cache + EffectEvent

```tsx
'use client'

import { Activity, useEffectEvent, cache } from 'react'

const fetchNotifications = cache(async (userId: string) => {
  const res = await fetch(`/api/notifications?userId=${userId}`)
  return res.json()
})

export function NotificationBell({ userId }: { userId: string }) {
  const [unread, setUnread] = useState(0)

  const markAsRead = useEffectEvent(async () => {
    await fetch(`/api/notifications/read`, { method: 'POST' })
    setUnread(0)
  })

  useEffect(() => {
    const interval = setInterval(async () => {
      const data = await fetchNotifications(userId)
      setUnread(data.unread)
    }, 30000)
    return () => clearInterval(interval)
  }, [userId])

  return (
    <Activity detection="subtree">
      {({ isActivity }) => (
        <button onClick={markAsRead} disabled={isActivity}>
          🔔 {isActivity ? '...' : unread > 0 ? `(${unread})` : ''}
        </button>
      )}
    </Activity>
  )
}
```

**Sources:**
- [React 19.2 release notes](https://react.dev/blog/2025/10/01/react-19-2)
- [React Activity component](https://react.dev/reference/react/Activity)
- [React useEffectEvent](https://react.dev/reference/react/useEffectEvent)
- [React cache API](https://react.dev/reference/react/cache)

## Common Mistakes in Composite Patterns

- **Not aborting previous requests** — always use `AbortController` or React Query's built-in cancellation
- **Not handling loading states** — every async operation needs a loading state
- **Not handling errors** — always show error UI, not just console.log
- **Mutations in render** — never call mutation functions in render; always in event handlers
- **Not resetting forms after submit** — `form.reset()` after successful submission
- **Stale closures in useEffect** — use refs or proper dependency arrays
- **Passing server-side data to client without serialization** — Dates must be `.toISOString()`, non-serializable objects can't cross the RSC boundary
- **`use()` with non-Promise** — `use()` only accepts Promises; for regular values just use them directly
- **Mutating props/state with React Compiler** — the compiler skips components that mutate; fix mutations first
- **`cache()` vs `use cache` confusion** — React's `cache()` is client-side function memoization; Next.js `use cache` is server-side persistence; don't confuse the two
