# Server Components — RSC Mental Model

> **React 19.2.7 patch (June 1, 2026):** v19.2.7 fixes a regression in Server Actions where `FormData` entries were not being passed correctly — affecting forms that submit multiple values or file uploads via Server Actions. If you use Server Actions for form submission, upgrade: `npm install react@latest react-dom@latest`. All v19.2.x versions now have this fix.
>
> **Sources:** [React 19.2.7 release](https://github.com/facebook/react/releases/tag/v19.2.7)

## The Core Distinction

| Server Components | Client Components |
|---|---|
| Run on the server only | Run on both server + client |
| Can access DB, filesystem, secrets directly | Can use hooks, browser APIs |
| Cannot use hooks (`useState`, `useEffect`) | Can use all React hooks |
| Cannot use event handlers | Can use event handlers |
| Default in Next.js App Router | Explicit `'use client'` required |
| Zero JS sent to client | JS bundle includes component |

## Decision Tree: Server or Client?

Ask these questions in order:

```
1. Does it need user interaction (click, hover, form)?
   → YES → Client Component ('use client')
   → NO  → Go to 2

2. Does it need browser APIs (localStorage, window, geolocation)?
   → YES → Client Component
   → NO  → Go to 3

3. Does it need React hooks (useState, useEffect, useContext)?
   → YES → Client Component
   → NO  → Server Component (default)

4. Does it fetch data or render UI from server data?
   → → Server Component
```

## Data Fetching Patterns

### Server Component: Direct DB Access

```tsx
// app/posts/page.tsx — server component
import { db } from '@/lib/db'
import { PostCard } from '@/components/post-card'

export default async function PostsPage() {
  // Parallel data fetching — don't await sequentially
  const [posts, authors] = await Promise.all([
    db.post.findMany({ 
      where: { published: true },
      orderBy: { createdAt: 'desc' },
      take: 20,
    }),
    db.author.findMany({ select: { id: true, name: true } }),
  ])

  const authorMap = new Map(authors.map(a => [a.id, a.name]))

  return (
    <main className="container mx-auto py-8">
      <h1 className="text-3xl font-bold mb-6">Latest Posts</h1>
      <div className="grid gap-6">
        {posts.map(post => (
          <PostCard 
            key={post.id} 
            post={post} 
            authorName={authorMap.get(post.authorId) ?? 'Unknown'}
          />
        ))}
      </div>
    </main>
  )
}
```

### Server Component: Parallel Fetching with Promise.all

```tsx
// ❌ Wrong — sequential awaits block each other
const user = await getUser(id)
const posts = await getUserPosts(id)

// ✅ Right — parallel
const [user, posts] = await Promise.all([getUser(id), getUserPosts(id)])
```

## Next.js 16 Caching — `use cache` Directive

**Next.js 16 removed implicit caching entirely.** In Next.js 15, `fetch` had implicit caching behavior. In Next.js 16, everything runs dynamically by default — you must explicitly opt into caching with the `use cache` directive.

This is a fundamental shift: caching is now opt-in, not opt-out.

### `use cache` — The Primary Caching API (Next.js 16)

`use cache` is the new first-class way to cache data in Next.js 16. It replaces both `unstable_cache` and the old `fetch` with `next` options pattern. The compiler handles memoization automatically.

```tsx
// lib/data.ts
import { cacheTag } from 'next/cache'

// Cache this function's return value for 1 hour
// The compiler automatically memoizes repeated calls with the same arguments
export async function getTopPosts(limit: number) {
  'use cache'

  const posts = await db.post.findMany({
    where: { published: true },
    orderBy: { views: 'desc' },
    take: limit,
  })

  // Tag this cache entry for on-demand invalidation
  cacheTag('posts')

  return posts
}
```

```tsx
// In a server component:
export default async function PopularPosts() {
  const posts = await getTopPosts(10)
  return <PostList posts={posts} />
}
```

**How `use cache` works:**
- The compiler analyzes the function and generates a cached version automatically
- Cache keys are derived from function arguments — same args = same cached result
- `cacheTag()` registers a tag for on-demand invalidation
- `cacheLife()` sets the TTL (defaults to a sensible framework-determined value)

```tsx
// With explicit TTL using cacheLife()
export async function getPopularPosts() {
  'use cache'
  cacheLife({ ttl: 3600 }) // cache for 1 hour

  return db.post.findMany({ where: { featured: true } })
}
```

**When to use `use cache`:**
- Any function that fetches data (DB, external API, filesystem)
- Use on the data function itself, not the page component
- Works on exported functions from shared modules and inline in components

**`use cache` persistence across deploys:** In Next.js 16, cached data persists across deployments — the deployment timestamp does not automatically invalidate the cache. This is intentional: stale-while-revalidate behavior means users may briefly see old data after a deploy. To invalidate cache on deploy, call `revalidateTag` or `updateTag` in a post-deploy hook or CI pipeline, or set a short `cacheLife({ ttl: ... })` TTL.

### On-Demand Revalidation: `revalidateTag` vs `updateTag`

Next.js 16 provides two distinct cache invalidation functions. Understanding when to use each matters for data freshness:

#### `revalidateTag` — Background Refresh (Eventual Consistency)

`revalidateTag(tag, profile)` schedules the cached entry to be refreshed on the **next request** — the stale data is served while revalidation happens in the background. The second `profile` argument is **required as of Next.js 16.2** — the single-argument form is deprecated. Pass `'max'` (recommended, stale-while-revalidate), any other [`cacheLife` profile name](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheLife), or `{ expire: number }` for a custom expiration:

```tsx
// app/actions.ts — revalidate after mutation
'use server'

import { revalidateTag } from 'next/cache'

export async function createPost(formData: FormData) {
  const parsed = createPostSchema.parse(Object.fromEntries(formData))
  await db.post.create({ data: parsed })

  // Schedule revalidation — stale data served until next request refetches
  // 'max' = framework's longest cacheLife profile (recommended for most cases)
  revalidateTag('posts', 'max')
}
```

**`profile` argument options** (Next.js 16.2+):

| Profile | Behavior | When to use |
|---|---|---|
| `'max'` (recommended) | Stale-while-revalidate — serves stale, refreshes in background | Most cases: product listings, blog posts, dashboards |
| `'hours'` / `'days'` / `'minutes'` (default profiles) | Same stale-while-revalidate semantics with the matching time window | When you want profile-aligned freshness |
| `{ expire: 60 }` | Custom: stale served for 60s, then blocking revalidate on next request | Specific SLA-bound data |
| (no profile) — deprecated | Blocking immediate revalidate on next request | Migrate to `updateTag` or `'max'` |

**Use when:** Slight staleness is acceptable, and you want to avoid latency spikes from synchronous cache rebuilds.

#### `updateTag` — Immediate Expiration (Strong Consistency)

`updateTag` **immediately expires** the cached entry — the next request gets fresh data, no stale serving:

```tsx
// app/actions.ts — update after mutation
'use server'

import { updateTag } from 'next/cache'

export async function createPost(formData: FormData) {
  const parsed = createPostSchema.parse(Object.fromEntries(formData))
  await db.post.create({ data: parsed })

  // Immediately expire — next request gets fresh data
  updateTag('posts')
}
```

 **Use when:** Data must be immediately consistent (e.g., inventory counts, financial data, user-generated content that affects authorization). Note: `updateTag` expires the entry immediately and revalidates in the background — the current request gets the stale value while the next gets fresh data.

#### Which to Use

| Function | Behavior | Latency on Next Request | Use Case |
|---|---|---|---|
| `revalidateTag(tag, 'max')` | Background refresh (stale-while-revalidate) | Fast (serves stale) | Non-critical data, high-traffic pages |
| `updateTag` | Immediate expiration | Slower (refetches on next request) | Critical data, personalization, auth |

**Important:** Both `revalidateTag` and `updateTag` can only be called from within **Server Actions**. For Route Handlers or other contexts, use `revalidateTag`.

```tsx
// lib/data.ts — tag the cache entry (used by both revalidateTag and updateTag)
import { cacheTag } from 'next/cache'

export async function getPosts() {
  'use cache'
  cacheTag('posts')  // Tag this cache for invalidation
  return db.post.findMany()
}
```

### `use cache` for External APIs

For external HTTP APIs, pass the fetch result directly:

```tsx
// lib/github.ts
import { cacheTag } from 'next/cache'

export async function getGitHubRepos(username: string) {
  'use cache'

  cacheTag(`github-${username}`)

  const res = await fetch(`https://api.github.com/users/${username}/repos`, {
    next: { revalidate: 3600 }, // still uses fetch's revalidate option
  })

  return res.json()
}
```

### When to Use `use cache` vs `fetch` Directly

| Pattern | Use When |
|---|---|
| `use cache` + `cacheTag` | Any server-side data function (DB, file I/O, external API wrapped in a function) |
| `fetch` directly in page | Quick, one-off external API fetches where fetch's `next` options are sufficient |

### `use cache: private` — Private Cached Data (Compliance Use)

`use cache: private` is for compliance requirements where data cannot leave the server but needs to be cached. Unlike standard `use cache` (which can be stored in shared caches), `use cache: private` keeps the cached data private to that specific server instance.

**When to use `use cache: private`:**
- Regulatory/compliance requirements that mandate data stays on the same server (e.g., GDPR data residency)
- When you cannot refactor code to pass runtime data (cookies, headers) as function arguments

```tsx
// lib/compliance-data.ts

// ❌ Standard use cache — data may be stored in shared/distributed cache
export async function getDashboardMetrics() {
  'use cache'
  // This cached data could be stored in a platform's distributed cache
  return db.metrics.findMany()
}

// ✅ use cache: private — data stays on this server only
export async function getDashboardMetrics() {
  'use cache: private'
  // This cached data is private to this server instance
  return db.metrics.findMany()
}
```

**Constraints:**
- `use cache: private` still requires reading cookies/headers **outside** the cached scope and passing values as arguments
- It does not bypass the constraint that cached functions must be deterministic — same args always return same result
- Only use when compliance requires it; standard `use cache` is preferred for performance

### `use cache: remote` — Distributed Cache (Platform Handler)

`use cache: remote` allows platforms to provide a dedicated external cache handler (Redis, KV, etc.) instead of the default in-memory cache. This reduces load on your database for high-traffic cached data.

**When to use `use cache: remote`:**
- High-traffic cached data where in-memory cache isn't sufficient
- When you want cache persistence across server restarts
- Platform supports a distributed cache (Vercel KV, Cloudflare KV, etc.)

```tsx
// next.config.ts — configure the remote cache handler
const nextConfig: NextConfig = {
  cacheHandlers: {
    remote: {
      // Platform-specific configuration
      // e.g., for Vercel: uses Vercel KV automatically
    },
  },
}
```

**Trade-offs:**
- Requires a network roundtrip to check the remote cache (higher latency than in-memory)
- Typically incurs platform fees for the cache service
- Not suitable for latency-sensitive paths; best for data that changes infrequently

```tsx
// lib/popular-posts.ts

// Standard use cache — in-memory only
export async function getTopPosts() {
  'use cache'  // Fast, in-memory, server-local only
  cacheLife({ ttl: 3600 })
  return db.post.findMany({ orderBy: { views: 'desc' }, take: 10 })
}

// use cache: remote — uses platform's distributed cache
// Lower latency spikes under high load, but higher baseline latency
export async function getTopPosts() {
  'use cache: remote'  // Network roundtrip to remote cache
  cacheLife({ ttl: 3600 })
  return db.post.findMany({ orderBy: { views: 'desc' }, take: 10 })
}
```

**Quick reference — which cache variant:**

| Directive | Storage | Latency | Use When |
|---|---|---|---|
| `use cache` | In-memory (server-local) | Lowest | Default — most cases |
| `use cache: private` | In-memory (server-local, private) | Lowest | Compliance — data must not leave server |
| `use cache: remote` | Platform cache (Redis/KV/etc.) | Higher | High traffic, persistent cache across restarts |

## `export const instant = false` — The "Block" Path (Next.js 16.3)

Next.js 16.3's Instant Navigations feature presents three explicit data-fetching choices at every server component render point. This is the third path — **"Block"** — for data that genuinely cannot be cached or streamed:

### The Stream / Cache / Block Decision Framework

Every `async` server component render in Next.js 16.3 must choose one of three paths:

| Path | Mechanism | When to use |
|---|---|---|
| **Stream** | Wrap in `<Suspense>` | Data arrives quickly but unpredictably |
| **Cache** | `'use cache'` | Data is safe to cache with a known TTL |
| **Block** | `export const instant = false` | Data cannot be prefetched; render must wait |

**The mental model:** `instant = false` tells Next.js "do not attempt to render this segment during the instant (shell-only) phase of navigation — wait for the real data." The full route renders normally on navigation, the same as it would have in Next.js 15. This is the **escape hatch**, not the default.

```tsx
// app/dashboard/page.tsx
// This route reads request-time data that can't be cached or predicted.
// Block it from instant nav so the navigation waits for real data.
export const instant = false

export default async function DashboardPage() {
  const user = await getUser() // request-time, uncacheable
  const alerts = await getAlerts(user.id)
  return <Dashboard user={user} alerts={alerts} />
}
```

**Key behaviors of `instant = false`:**
- **Highest-wins resolution.** Resolution is top-down, first-explicit-config-wins — the **highest** `instant = false` in a route tree decides the whole subtree. Removing a leaf's opt-out does nothing while an ancestor still holds one.
- **Doesn't clear sync-IO errors.** `new Date()`, `Date.now()`, `Math.random()`, `crypto.randomUUID()` called at render time still fail the prerender even with `instant = false` — fix those explicitly.
- **Client Components don't get an opt-out.** `instant` is a Server Component segment config; exporting it from a `"use client"` module is a build error (`E1344`).
- **Framework routes (`/-not-found`, etc.) have no user file.** Don't try to add `instant = false` to `app/not-found.tsx` — add it to `app/layout.tsx` instead (the root layout covers all framework routes).

**Source:** [Next.js 16.3 — Instant Navigations blog post](https://nextjs.org/blog/next-16-3-instant-navigations) · [`export const instant` docs](https://nextjs.org/docs/app/api-reference/next-config-js/instant) · [`cache-components-instant-false` codemod](https://nextjs.org/docs/app/guides/migrating-to-cache-components)

**Sources:**
- [Next.js `use cache: private` docs](https://nextjs.org/docs/app/api-reference/directives/use-cache-private)
- [Next.js `use cache: remote` docs](https://nextjs.org/docs/app/api-reference/directives/use-cache-remote)
- [Next.js `cacheHandlers` config](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheHandlers)

### `unstable_cache` — Legacy Pattern (Still Works, Prefer `use cache`)

`unstable_cache` still works in Next.js 16 for backward compatibility, but `use cache` is the preferred approach going forward:

```tsx
// ⚠️ Legacy — prefer 'use cache' in Next.js 16
import { unstable_cache } from 'next/cache'

export const getTopPosts = unstable_cache(
  async (limit: number) => {
    return db.post.findMany({
      where: { published: true },
      orderBy: { views: 'desc' },
      take: limit,
    })
  },
  ['top-posts'],
  { tags: ['posts'], revalidate: 3600 }
)
```

### Dynamic Rendering (Default in Next.js 16)

Every route is dynamic by default. Use `use cache` to opt into caching:

```tsx
// Dynamic (default) — every request fetches fresh data
export default async function Dashboard() {
  const stats = await getDashboardStats() // runs on every request
  return <DashboardUI stats={stats} />
}
```

### Route Segment Config (Still Valid)

```tsx
// Force static rendering — still uses 'use cache' implicitly for data
export const dynamic = 'force-static'

// Force dynamic — always runs at request time
export const dynamic = 'force-dynamic'
```

## Server → Client Data Passing

### Passing Serializable Props

Server components can pass data to client components via props — but props must be **serializable**:

```tsx
// ✅ Valid — plain objects, arrays, primitives
<ClientComponent 
  user={{ id: '1', name: 'Alice', role: 'admin' }}
  tags={['typescript', 'nextjs']}
  count={42}
/>
```

```tsx
// ❌ Invalid — functions, class instances, Promises from client code
<ClientComponent 
  onClick={handleClick}       // ❌ functions not serializable
  ref={inputRef}              // ❌ refs not serializable
/>
```

### Passing Promises (React 19 `use()` hook)

In React 19 / Next.js 15+, you can pass Promises from server to client:

```tsx
// Server Component — passes a Promise, NOT the resolved data
import { Suspense } from 'react'

export default async function Page() {
  const userPromise = getUser('123') // returns Promise<User>
  
  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  )
}
```

```tsx
// Client Component — resolves the Promise with use()
'use client'

import { use } from 'react'

interface UserProfileProps {
  userPromise: Promise<User>
}

export function UserProfile({ userPromise }: UserProfileProps) {
  const user = use(userPromise) // suspends until resolved
  return <div>{user.name}</div>
}
```

**Why not just await?** Because `await` blocks the entire render, while `use()` with Suspense allows streaming and partial rendering.

**Error handling with `use()`:** If the Promise rejects, `use()` **throws** the error (it doesn't return normally). This means:
- The **Suspense boundary** handles the "pending" state (shows fallback while loading)
- The **Error Boundary** handles the "rejected" state (shows error UI if the fetch fails)

```tsx
// Both boundaries are needed for complete handling
export default function UserPage() {
  const userPromise = getUser('123')
  
  return (
    <ErrorBoundary fallback={<UserError />}>
      <Suspense fallback={<UserSkeleton />}>
        <UserProfile userPromise={userPromise} />
      </Suspense>
    </ErrorBoundary>
  )
}
```

### `use()` + Error Boundary — Complete Pattern (React 19)

When a Promise passed to `use()` **rejects**, React throws the error which propagates to the nearest Error Boundary. Here's a complete, production-ready pattern combining both:

**Step 1 — A reusable Error Boundary component:**

```tsx
// components/error-boundary.tsx
'use client'

import { Component, type ReactNode } from 'react'

interface ErrorBoundaryProps {
  children: ReactNode
  fallback: ReactNode // Rendered when an error is caught
}

interface ErrorBoundaryState {
  hasError: boolean
  error: Error | null
}

export class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props)
    this.state = { hasError: false, error: null }
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error }
  }

  override render() {
    if (this.state.hasError) {
      return this.props.fallback
    }
    return this.props.children
  }
}
```

**Step 2 — A safer fallback that doesn't re-throw:**

```tsx
// components/error-fallback.tsx
'use client'

interface ErrorFallbackProps {
  error: Error
  reset: () => void
}

export function ErrorFallback({ error, reset }: ErrorFallbackProps) {
  return (
    <div className="flex flex-col items-center gap-4 p-6 border border-destructive rounded-lg bg-destructive/5">
      <div className="text-center">
        <p className="text-sm font-medium text-destructive">Something went wrong</p>
        <p className="text-xs text-muted-foreground mt-1">{error.message}</p>
      </div>
      <button
        onClick={reset}
        className="text-sm px-4 py-2 bg-primary text-primary-foreground rounded-md hover:bg-primary/90"
      >
        Try again
      </button>
    </div>
  )
}
```

**Step 3 — Using `use()` with Error Boundary and Suspense:**

```tsx
// app/users/[id]/page.tsx — Server Component
import { Suspense } from 'react'
import { ErrorBoundary } from '@/components/error-boundary'
import { ErrorFallback } from '@/components/error-fallback'
import { UserProfile } from '@/components/user-profile'
import { UserSkeleton } from '@/components/user-skeleton'
import { getUser } from '@/lib/data'

export default async function UserPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const userPromise = getUser(id) // Can throw — wrapped in ErrorBoundary

  return (
    <ErrorBoundary
      fallback={<ErrorFallback error={new Error('Failed to load user')} reset={() => {}} />}
    >
      <Suspense fallback={<UserSkeleton />}>
        <UserProfile userPromise={userPromise} />
      </Suspense>
    </ErrorBoundary>
  )
}
```

**Key points:**
- **Suspense** handles the **loading** state (Promise pending)
- **ErrorBoundary** handles the **rejected** state (Promise threw)
- Without both, a rejected Promise would crash the entire component tree
- The ErrorBoundary must be **outside** the Suspense — if it's inside, the Suspense catches the error first

**Resetting the Error Boundary after recovery:**

```tsx
// app/users/[id]/page.tsx — with reset capability
'use client'

import { useState } from 'react'
import { ErrorBoundary } from '@/components/error-boundary'
import { ErrorFallback } from '@/components/error-fallback'

function UserSection({ userId }: { userId: string }) {
  const [key, setKey] = useState(0)

  function handleReset() {
    setKey(k => k + 1) // Remount children by changing key
  }

  return (
    <ErrorBoundary
      fallback={<ErrorFallback error={new Error('Failed to load user')} reset={handleReset} />}
    >
      <UserProfileWithKey userId={userId} key={key} />
    </ErrorBoundary>
  )
}
```

**Using `react-error-boundary` library (recommended for complex apps):**

```tsx
import { ErrorBoundary } from 'react-error-boundary'

function UserPage({ userId }: { userId: string }) {
  return (
    <ErrorBoundary
      fallbackRender={({ error, resetErrorBoundary }) => (
        <div>
          <p>Failed: {error.message}</p>
          <button onClick={resetErrorBoundary}>Retry</button>
        </div>
      )}
    >
      <Suspense fallback={<Skeleton />}>
        <UserProfile userPromise={getUser(userId)} />
      </Suspense>
    </ErrorBoundary>
  )
}
```

**`use()` rejection vs synchronous errors:**
- `use(promise)` **throws** if the promise rejects — caught by Error Boundary
- Synchronous errors thrown during render are also caught by Error Boundary
- Both paths lead to the same Error Boundary `fallback` UI

**Sources:**
- [React `use()` hook reference](https://react.dev/reference/react/use)
- [React error boundaries](https://react.dev/reference/react/Component#catching-rendering-errors-with-error-boundaries)
- [React 19 `use()` error handling](https://react.dev/learn/error-bounds)

### Consuming Context with `use()` (React 19)

React 19's `use()` hook can consume Context, even in components that can't use hooks at the top level:

```tsx
// components/user-badge.tsx
'use client'

import { use } from 'react'
import { ThemeContext } from '@/contexts/theme-context'

export function UserBadge() {
  // Unlike useContext(), use() works inside conditional statements
  // and during Render (though with caveats — only for Suspense-compatible data)
  const theme = use(ThemeContext)
  return <span className={theme === 'dark' ? 'text-white' : 'text-black'}>{theme}</span>
}
```

```tsx
// components/dashboard-stats.tsx
'use client'

// Conditional consumption with use() — only in client components
function Dashboard(props: { showStats: boolean }) {
  if (props.showStats) {
    const stats = use(StatsContext) // ✅ allowed in React 19
    return <StatsPanel stats={stats} />
  }
  return <div>...</div>
}
```

**Note:** `use()` for Context is only available in Client Components. Both components above have `'use client'` — without it, the code will fail because `use()` for Context requires a client component.

### Production Pattern: Server-Authed Context with `use()` (React 19)

A common real-world pattern: fetch user/auth data in a Server Component, pass it through Context, consume it in deeply nested Client Components without prop drilling:

```tsx
// contexts/user-context.tsx
'use client'

import { createContext, use } from 'react'

interface User {
  id: string
  name: string
  email: string
  avatarUrl: string | null
  role: 'admin' | 'user' | 'guest'
}

// 1. Create context — value is the User or null
export const UserContext = createContext<User | null>(null)

// 2. Provider component wraps children with the context value
// This is a Client Component that receives server data as props
interface UserProviderProps {
  children: React.ReactNode
  user: User | null
}

export function UserProvider({ children, user }: UserProviderProps) {
  return (
    <UserContext.Provider value={user}>
      {children}
    </UserContext.Provider>
  )
}

// 3. Consumer hook — uses use() to read context
// Works anywhere in the tree without prop drilling
export function useUser() {
  const user = use(UserContext)
  if (user === null) {
    throw new Error('useUser must be used within a UserProvider with a resolved user')
  }
  return user
}

export function useOptionalUser() {
  return use(UserContext)  // Can return null — no error thrown
}
```

```tsx
// app/layout.tsx — Server Component
import { getServerUser } from '@/lib/auth'
import { UserProvider } from '@/contexts/user-context'

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  // 1. Fetch on the server — direct DB access, no API needed
  const user = await getServerUser()

  return (
    <html lang="en">
      <body>
        {/* 2. Pass serializable user data to Client Provider */}
        <UserProvider user={user}>
          <Header />
          {children}
        </UserProvider>
      </body>
    </html>
  )
}
```

```tsx
// components/header.tsx — Client Component (no props needed!)
'use client'

import { useUser, useOptionalUser } from '@/contexts/user-context'

export function Header() {
  // No prop drilling — read directly from context
  const user = useUser()
  const optional = useOptionalUser()

  return (
    <header className="flex items-center gap-4 p-4 border-b">
      <Logo />
      <nav className="flex-1" />
      {user.role === 'admin' && <AdminLink href="/admin" />}
      <UserAvatar
        src={user.avatarUrl}
        alt={user.name}
        fallback={user.name.slice(0, 2).toUpperCase()}
      />
    </header>
  )
}
```

**Why this pattern:**
- **No prop drilling** — deeply nested components read auth data directly
- **Server fetches, client consumes** — the Server Component fetches user data, Client Components read from context
- **`use()` for conditional reads** — unlike `useContext()`, `use(Context)` can be called inside conditionals (inside the component body, not at the top level)
- **Type-safe** — `useUser()` throws if called outside provider, enforcing correct usage

**When NOT to use this pattern:**
- For frequently changing data (use React Query instead)
- For UI state only (use Zustand instead)
- When the data is only needed 1–2 levels deep (prop drilling is fine for shallow trees)

**Alternative: Zod + tRPC for type-safe auth** — For larger apps, consider tRPC with a typed `context` that infers the session user, which gives you end-to-end type safety without manual context wiring.

## React 19 Document Metadata

React 19 introduces native support for rendering metadata elements (`<title>`, `<meta>`, `<link>`) directly in component JSX — no framework-specific API needed. This works in both server and client components.

```tsx
// app/about/page.tsx
import { title, meta, link } from 'react'

export default function AboutPage() {
  return (
    <>
      {title('About Us — My App')}
      {meta({ name: 'description', content: 'Learn about our company and team.' })}
      {link({ rel: 'canonical', href: 'https://myapp.com/about' })}
      
      <main>
        <h1>About Us</h1>
        {/* Page content */}
      </main>
    </>
  )
}
```

**Why this matters:** Previously, metadata required either Next.js `generateMetadata()` or a library like `react-helmet`. Now it's just React. Next.js's metadata API is still recommended for complex cases (OG images, i18n), but simple metadata is now portable across React frameworks.

**With Next.js metadata:** Next.js's `generateMetadata()` still takes precedence. React 19 metadata functions are a lower-level primitive that works everywhere React does.

**Common metadata patterns:**

```tsx
import { title, meta, link } from 'react'

// Set page title
{title('Page Title')}

// Meta description
{meta({ name: 'description', content: 'Page description' })}

// Viewport (React 19 handles this too)
{meta({ name: 'viewport', content: 'width=device-width, initial-scale=1' })}

// Open Graph
{meta({ property: 'og:title', content: 'My Article' })}
{meta({ property: 'og:image', content: 'https://myapp.com/og.jpg' })}

// Canonical link
{link({ rel: 'canonical', href: 'https://myapp.com/articles/my-post' })}
```

## The Client Component Island Pattern

Keep the `'use client'` boundary as small as possible. Wrap interactive areas, not entire pages:

```tsx
// app/posts/page.tsx — server component (default)
import { PostsList } from './posts-list'
import { FilterBar } from '@/components/filter-bar' // 'use client'

export default async function PostsPage() {
  return (
    <main>
      {/* Small interactive island */}
      <FilterBar /> 
      {/* Rest is server-rendered */}
      <PostsList />
    </main>
  )
}
```

## Context in Server Components

Server components **cannot use React Context** directly. Solutions:

### Pass Data as Props

```tsx
// Server component provides, client component consumes
export default async function Layout({ children }: { children: React.ReactNode }) {
  const user = await getCurrentUser()
  return <ThemeProvider userTheme={user.preferredTheme}>{children}</ThemeProvider>
}
```

### Create a Client Provider

```tsx
// components/theme-provider.tsx
'use client'

import { createContext, useContext, useState } from 'react'

const ThemeContext = createContext<ThemeContextType | null>(null)

export function ThemeProvider({ children, theme }: { children: React.ReactNode; theme: Theme }) {
  return (
    <ThemeContext.Provider value={{ theme, setTheme: () => {} }}>
      {children}
    </ThemeContext.Provider>
  )
}
```



## `headers()` Returns a Unique Object Per Render Pass — PR #96085 (**SHIPPED in `16.3.0-canary.103`**, [Sebastian "Sebbie" Silbermann](https://github.com/eps1lon), merged 2026-07-29T22:16:35Z, npm-published 2026-07-30T00:11:44Z)

**The change:** every render pass within a single HTTP request now resolves `await headers()` (from `next/headers`) to a **distinct `Headers` object** over the **same underlying data**. Previously, all passes within the same request resolved to the **same sealed `Headers` object** created lazily by the request store.

**Why "render pass" matters:** a single HTTP request can render the same React tree in **multiple passes** with different semantics for `connection()`:

1. **Prospective prerender + final prerender of a runtime prefetch** — the runtime-prefetch path first does a prospective prerender, then a final prerender. Both are server-side, both can read `headers()`, but they have different runtime semantics (e.g. the prospective render can be canceled/discarded).
2. **Dynamic render + the runtime prerender spawned from it to refresh the client's prefetch cache** — when a user navigates, the page is dynamically rendered; that dynamic render can spawn a runtime prerender to update the prefetch cache for future navigations. Both renders can read `headers()`, but the user's request only goes through the dynamic pass — the runtime prerender is for the cache.

Before PR #96085, both passes shared the same sealed `Headers` object — so any userland cache keyed on the identity of `await headers()` (e.g. `WeakMap<Headers, Promise<MyData>>` to memoize expensive request-derived data) **leaked promises across passes**. A discarded prospective prerender could leave a resolved Promise attached to a Headers object that's also visible from the final prerender; a runtime prefetch cache could pick up data from the dynamic render's Headers object that wasn't supposed to be cached for the next request.

After PR #96085, every render pass gets a distinct Headers object, even though the underlying data (the request's actual headers) is the same. So:

```ts
// ❌ WRONG (this is what was happening before)
const headersA = await headers();
const promise = memoizePerHeaders(headersA);  // keyed on identity
// ...later in the same request, a different render pass runs...
const headersB = await headers();
const samePromise = memoizePerHeaders(headersB);  // returned the SAME promise, even though it should be a fresh per-pass memo
```

```ts
// ✅ RIGHT (this is what happens now)
const headersA = await headers();
const promise = memoizePerHeaders(headersA);  // keyed on identity
// ...later in the same request, a different render pass runs...
const headersB = await headers();
const samePromise = memoizePerHeaders(headersB);  // returns a DIFFERENT promise — headersB is a different object
```

**The new contract:** *"the headers object identifies the request within one render pass"*. This is now a real, enforceable contract. Identity checks (`===`) on `headers()` are reliable for "are we in the same pass" but **not** for "is this the same request".

**Practical impact for users today:**

- **Anyone using `WeakMap<Headers, T>` or `Map<Headers, T>` as a request-scoped memoization cache** — this PR fixes the leak. If you saw memoized data from a discarded prospective prerender appearing in the final prerender, that's now fixed.
- **Anyone using `headers()` as a `useMemo` / `use cache` cache key** — if your key was the `Headers` object identity, you need to switch to a stable derived key (e.g. `JSON.stringify(Object.fromEntries(headers.entries()))` or a specific header like `headers.get('cookie')`). The `Headers` object identity is now per-pass, not per-request, so it's not a stable cache key across passes.
- **Anyone reading `headers()` inside `use cache`** — this PR does not change `use cache` semantics; the cached scope is still a separate request from the headers reading scope. But if your cache was keyed on `headers` identity, see above.
- **Anyone reading `headers()` inside Server Actions** — Server Actions are their own render pass, so `headers()` in a Server Action is the request's headers (not a separate request's headers). This PR doesn't change that semantic.

**Migration recipe** — find any code that uses `headers()` as a Map/WeakMap key:

```bash
# Find request-scoped memos keyed on headers identity
rg -n 'WeakMap.*[Hh]eaders|new Map\(\[\[headers' --type ts --type tsx --type js --type jsx

# Or simpler: find any WeakMap that's keyed on something derived from headers()
rg -n 'WeakMap' --type ts --type tsx | head -20

# Find any userland cache that uses headers() as a key
rg -n 'cache.*headers\(\)|headers\(\).*cache' --type ts --type tsx
```

For each match, either (a) use a stable derived key (specific header values, not the object identity), or (b) scope the cache to a single pass via React's `cache()` from `react` (which is already per-render-pass scoped) instead of a long-lived `WeakMap`.

**Source:** [PR #96085 — `Ensure unique resolved headers() value between render passes`](https://github.com/vercel/next.js/pull/96085) · Sebastian "Sebbie" Silbermann · merged 2026-07-29T22:16:35Z · **SHIPPED in `16.3.0-canary.103`** (npm-published 2026-07-30T00:11:44Z).


## App Router Execution Mode Refactor — 9-PR Coordinated Set Ahead of `16.3.1-canary.3` (August 5, 2026)

The v1.5.21 cycle (Aug 4 06:14Z) added the `## headers() Returns a Unique Object Per Render Pass — PR #96085` section (SHIPPED in `next@16.3.0-canary.103`). The v1.5.21 cycle also touched `server-components.md` for the `## revalidateTag vs updateTag — The Canonical Decision Matrix (Next.js 16.3 STABLE, August 3, 2026)` section. Since then, **server-components.md has been silent through the v1.5.22 → v1.5.26 cycles (35h48min stale at this cron's check, tied with `testing.md` for the most-stale topic file)**. The material change in the 6h window for server-components is the **coordinated 9-PR executionMode refactor** that landed on the Next.js canary-branch in the 6h window — a significant App Router pipeline refactor that moves `WorkStore.isStaticGeneration` → explicit `'prerender' | 'request'` execution mode, lifts the decision to entrypoints, separates render/prerender pipelines, and removes the downstream mode reads. All 9 PRs are on the canary-branch ahead of `next@16.3.1-canary.3` (which was published 2026-08-05T06:27:06Z); the canary-branch has 18 commits ahead of canary.3 at this cron's check; the 9-PR executionMode refactor + PR #96735 React vendor bump are the bulk of what will ship in `next@16.3.1-canary.4` when that npm-publishes (expected within 2-12h on the 24h cadence).

**All 9 PRs are pure refactors — zero observable behavior change for App Router users on `next@16.3.0` STABLE or on `next@16.3.1-canary.3`.** The only user-visible change when canary.4 npm-publishes: the WorkStore internal field for execution mode is gone, but no public API contract is touched. **Do not plan a migration for this refactor** — it's pure internal cleanup for maintainability.

### The 9-PR executionMode refactor set (in chronological order)

The 9 PRs are a coordinated refactor that moves the "is this a prerender or a request?" decision through 4 phases:

**Phase 1 — Foundation: replace `WorkStore.isStaticGeneration` with `executionMode`** ([PR #96570](https://github.com/vercel/next.js/pull/96570), merged 2026-08-05T12:51:07Z)

Replaces the boolean `WorkStore.isStaticGeneration` with an explicit `'prerender' | 'request'` execution mode. Preserves the existing derivation and behavior while making the render/prerender distinction explicit. Documents that prerendering includes both **build-time generation** (the static prerender pass during `next build`) AND **runtime revalidation** (the per-request revalidation pass when a stale cache entry is detected and re-rendered before being served). Establishes groundwork for moving the render decision higher in the stack and replacing downstream mode checks with purpose-specific state. Stacked on PR #96564.

**Phase 2 — Threading: pass explicit prefetch hint policy + render capabilities through App Render** (PR #96572 + PR #96576, both merged 2026-08-05T12:51:04Z)

- [PR #96572](https://github.com/vercel/next.js/pull/96572) — Pass explicit **prefetch hint policy** through App Render (the policy that determines which segments to prefetch and how aggressively — used by the partial-prefetching logic).
- [PR #96576](https://github.com/vercel/next.js/pull/96576) — Pass explicit **render capabilities** through App Render (the capability flags that determine what operations are valid in this render pass — e.g. "can spawn runtime prefetches", "can throw redirect exceptions", "can read request headers").

Both PRs thread the values as explicit parameters rather than relying on the App Render code to read them from WorkStore (which is the pattern the refactor is moving away from).

**Phase 3 — The headline: Move App Router execution intent to entrypoints** ([PR #96640](https://github.com/vercel/next.js/pull/96640), merged 2026-08-05T12:51:11Z, closes issue #96519)

App Router execution intent was **previously inferred during WorkStore construction** from response capabilities and request state. App Page also carried that intent through its handler context and lazy render API before selecting the rendering operation inside app-render. **PR #96640 moves those decisions to entrypoints where the work is known:**

- **Reusable output** — including build-time generation, fallback shells, and runtime revalidation — selects **prerendering**.
- **Request-specific and resumed rendering** selects **request rendering**.
- **App Page** exposes distinct **render** and **prerender** operations, so execution mode no longer travels through `AppPageRouteHandlerContext` or the lazy render API.
- **App Route** execution derives its intent from **whether the response has an ISR cache key** (responses without an ISR cache key are request-time, responses with one can be prerendered).

**WorkStore still consumes the explicit mode internally as a transitional seam.** Follow-up work can migrate downstream behavior to the active `WorkUnitStore` and remove the mode from WorkStore entirely (PR #96674 below is that follow-up).

**Verification** per the PR description: `pnpm --filter=next types` + `pnpm test-start-turbo test/e2e/app-dir/segment-cache/static-shell-vary-params-regression/static-shell-vary-params-regression.test.ts` + `pnpm test-start-webpack test/e2e/app-dir/segment-cache/static-shell-vary-params-regression/static-shell-vary-params-regression.test.ts`.

**Phase 4 — Pipeline separation: split App Page and App Route render/prerender pipelines** (PR #96659 + PR #96662, both merged 2026-08-05T12:51:12Z / 16:52:57Z)

- [PR #96659](https://github.com/vercel/next.js/pull/96659) — **Separate App Page render and prerender pipelines**. App Page rendering previously shared a single implementation that used the Work Store execution mode to choose between the request and prerender pipelines. **The fix**: (a) extracts mode-neutral App Page preparation into a shared step, (b) gives request rendering and prerendering separate top-level implementations, (c) moves Work Store creation into the corresponding entrypoint, (d) makes each path call `renderToStream` or `prerenderToStream` directly, removing the execution-mode dispatch between them. This moves the rendering decision to the App Page boundary and creates a clearer path toward removing execution mode from the Work Store entirely.
- [PR #96662](https://github.com/vercel/next.js/pull/96662) — **Separate App Route render and prerender pipelines**. The App Route counterpart to PR #96659; same pattern — separate top-level implementations, Work Store creation moved to entrypoint.

**Phase 5 — Downstream cleanup: remove execution-mode reads** (PR #96660 + PR #96670, both merged 2026-08-05T16:52:56–57Z)

- [PR #96660](https://github.com/vercel/next.js/pull/96660) — **Remove App Page execution mode reads**. Downstream consumers no longer read execution mode from WorkStore / handler context; they receive the intent as an explicit parameter. Matches the threading done in Phase 2.
- [PR #96670](https://github.com/vercel/next.js/pull/96670) — **Remove cache revalidation execution mode reads**. Downstream cleanup for the cache revalidation path (revalidateTag / updateTag / revalidatePath handlers).

**Phase 6 — Final removal: WorkStore no longer carries execution mode** ([PR #96674](https://github.com/vercel/next.js/pull/96674), merged 2026-08-05T16:52:57Z)

The **final removal PR** — WorkStore no longer carries an execution mode field; the transitional seam from PR #96570 is gone. After PR #96674, all rendering decisions are made at entrypoints; WorkStore is purely a work-context object (carrying params, request state, cache state) without the "am I a prerender or a request?" flag.

### Companion PR: React vendor bump 7dfc7ccd-20260803 → 11eddecd-20260805

[PR #96735](https://github.com/vercel/next.js/pull/96735) — `Upgrade React from 7dfc7ccd-20260803 to 11eddecd-20260805`, merged 2026-08-05T17:15:10Z by vercel-release-bot. Single-PR vendor bump; no public API change. The PR carries the React 11eddecd-20260805 canary cut (which ships PR #36944 [Devtools] component search in Profiler's commit view — the same 1-commit bundle documented in `components.md` → "React 19.3.0-canary-11eddecd-20260805 SHIPPED"). After this PR merges, `npm view next@canary dependencies.react` returns `19.3.0-canary-11eddecd-20260805` instead of the previous `19.3.0-canary-7dfc7ccd-20260803`. **Zero user-visible impact** from the vendor bump — both `7dfc7ccd` and `11eddecd` are DevTools-only on the React side.

### Per-PR practical-impact table

| PR | Phase | Files | User-visible change |
|---|---|---|---|
| [#96570](https://github.com/vercel/next.js/pull/96570) | 1 — Foundation | multiple WorkStore callers | None — internal field rename |
| [#96572](https://github.com/vercel/next.js/pull/96572) | 2 — Threading | App Render | None — explicit parameter |
| [#96576](https://github.com/vercel/next.js/pull/96576) | 2 — Threading | App Render | None — explicit parameter |
| [#96640](https://github.com/vercel/next.js/pull/96640) | 3 — Headline | App Page + Work Store | None — decisions move to entrypoints |
| [#96659](https://github.com/vercel/next.js/pull/96659) | 4 — Pipelines | App Page render + prerender | None — separate top-level impls |
| [#96660](https://github.com/vercel/next.js/pull/96660) | 5 — Cleanup | App Page downstream | None — reads replaced with params |
| [#96662](https://github.com/vercel/next.js/pull/96662) | 4 — Pipelines | App Route render + prerender | None — separate top-level impls |
| [#96670](https://github.com/vercel/next.js/pull/96670) | 5 — Cleanup | Cache revalidation | None — reads replaced with params |
| [#96674](https://github.com/vercel/next.js/pull/96674) | 6 — Final | WorkStore field | None — WorkStore field removed |
| [#96735](https://github.com/vercel/next.js/pull/96735) | Vendor | next packages/react | None — 11eddecd React vendor bump |

**Zero user-visible change across all 10 PRs.** Pure refactor.

### Architectural rationale

The `WorkStore.isStaticGeneration` boolean was an **inferable-but-not-explicit** signal that traveled through the rendering layers. Three problems with the old design:

1. **Implicit decision-making** — the boolean was set during WorkStore construction based on response capabilities + request state, but the rendering layer had to read it later to know what to do. This made the rendering decision implicit + spread across multiple files.
2. **"I forgot to set the flag" bugs** — any code path that needed to know the rendering mode had to remember to read the WorkStore field. If a new code path was added that needed the mode, the developer had to thread the read through the appropriate layers.
3. **Two-source-of-truth risk** — the WorkStore field could drift from the actual response/request state if either was modified after WorkStore construction (e.g. runtime revalidation could change the semantics).

The refactor **lifts the decision to entrypoints where the work is known** — by the time the request hits App Render, the entrypoint has already classified the work as "reusable output" (prerender) or "request-specific" (request render). The decision travels through as an explicit parameter, not a hidden WorkStore read. This is the same pattern that Vercel has been hinting at for several releases as part of the **`WorkUnitStore` migration** — PR #96674 (the WorkStore field removal) is a stepping stone toward fully migrating the rendering layer to `WorkUnitStore`-based work contexts.

### 5-step audit recipe (after `next@16.3.1-canary.4` npm-publishes)

```bash
# 1. Confirm canary.4 includes the executionMode refactor + React 11eddecd vendor bump:
npm view next@canary version
# → should show: 16.3.1-canary.4 or later

# 2. Confirm the WorkStore.isStaticGeneration field is gone:
rg -n "isStaticGeneration" packages/next/src/server/  # would only work in the next.js source; for users, skip
# (this is a source-level audit only; users see no API change)

# 3. Find any code paths that read WorkStore.executionMode directly (should be NONE post-PR-#96674):
# (source-level audit only)

# 4. Confirm React 11eddecd-20260805 is the bundled React:
npm view next@canary dependencies.react
# → should show: 19.3.0-canary-11eddecd-20260805 (post-#96735 vendor bump)

# 5. Confirm canary-branch ahead count is back to 0:
curl -s https://api.github.com/repos/vercel/next.js/compare/v16.3.1-canary.4...canary | python3 -c "import sys,json; print('ahead:', json.load(sys.stdin).get('ahead_by'))"
# → should show: 0 (or low single digits if new PRs landed in the gap)
```

### What's NOT in the refactor (forward-looking)

- **No public API change** — no exports added, none removed, none renamed.
- **No new config option** — `next.config.ts` keys are unchanged.
- **No codemod needed** — the refactor is purely internal.
- **No behavior change for production users** — pre-fix and post-fix produce identical rendered output for App Router routes.
- **Pure internal refactor for maintainability** — the goal is to make future rendering-layer changes easier + to set up the `WorkUnitStore` migration.
- **Will ship in `next@16.3.1-canary.4`** (and eventually `next@16.3.1` STABLE) when the canary-branch version-tag commit lands + npm publishes.

### Sources

- [Next.js PR #96570 — `Replace WorkStore isStaticGeneration with executionMode`](https://github.com/vercel/next.js/pull/96570) — merged 2026-08-05T12:51:07Z, foundation phase
- [Next.js PR #96640 — `Move App Router execution intent to entrypoints`](https://github.com/vercel/next.js/pull/96640) — merged 2026-08-05T12:51:11Z, **the headline PR**; closes issue #96519
- [Next.js PR #96659 — `Separate App Page render and prerender pipelines`](https://github.com/vercel/next.js/pull/96659) — merged 2026-08-05T12:51:12Z, App Page pipeline split
- [Next.js PR #96662 — `Separate App Route render and prerender pipelines`](https://github.com/vercel/next.js/pull/96662) — merged 2026-08-05T16:52:57Z, App Route pipeline split
- [Next.js PR #96674 — `Remove WorkStore execution mode`](https://github.com/vercel/next.js/pull/96674) — merged 2026-08-05T16:52:57Z, **the final removal PR**
- [Next.js PR #96735 — `Upgrade React from 7dfc7ccd-20260803 to 11eddecd-20260805`](https://github.com/vercel/next.js/pull/96735) — merged 2026-08-05T17:15:10Z, vendor bump
**Test coverage added:** the PR adds a test that demonstrates the previous incorrect behavior (the same `Headers` object returned across passes), and confirms the post-fix behavior (distinct objects per pass).


## `ReactDOM.browser()` — Per-Instance "Client-Only" Boundary (React canary `0f42eac2-20260730`, **SHIPPED in `next@16.3.0-canary.104`**)

The App Router's primary mechanism for "only render in the browser" is the `"use client"` directive — but it's a **module-level** boundary: every export in the module becomes a client component, and the entire module's transitive imports are forced into the client bundle. For some use cases that's overkill.

**The new public API** [React PR #37143](https://github.com/facebook/react/pull/37143) (shipped in `react@19.3.0-canary-0f42eac2-20260730`, vendored into `next@16.3.0-canary.104` via [Next.js PR #96402](https://github.com/vercel/next.js/pull/96402)) adds `ReactDOM.browser()` — a function that returns a **"usable"** object that errors during SSR and resolves during rendering in the browser. When combined with the `use()` hook inside a `<Suspense>` boundary, this gives you a **per-instance** client-only boundary — a single component can render client-only while the rest of the tree stays shared.

### Canonical usage

```tsx
// components/browser-only-chart.tsx — 'use client' (needed for `use()`)
'use client'
import { use, Suspense } from 'react'
import { browser } from 'react-dom'
import { ChartCanvas } from './chart-canvas' // browser-only WebGL

function Chart() {
  use(browser())  // throws on the server, resolves in the browser
  return <ChartCanvas />
}

export function BrowserOnlyChart({ data }: { data: ChartData }) {
  return (
    <Suspense fallback={<ChartSkeleton />}>
      <Chart data={data} />
    </Suspense>
  )
}
```

```tsx
// app/dashboard/page.tsx — server component (default)
import { BrowserOnlyChart } from '@/components/browser-only-chart'

export default async function DashboardPage() {
  const data = await getDashboardData()  // runs on the server
  return (
    <main>
      <h1>Dashboard</h1>
      {/* Server-rendered shell, but the chart suspends on SSR and renders in the browser */}
      <BrowserOnlyChart data={data} />
    </main>
  )
}
```

### Why not just `"use client"`?

If you move the entire `BrowserOnlyChart` into a `'use client'` module, then:

- The whole module (and its imports) goes into the client bundle.
- The chart's `data` prop gets serialized across the network (RSC payload) — costs bandwidth.
- The chart's parent layout cannot share code between server and client.

With `browser()` instead:

- The parent `app/dashboard/page.tsx` stays a **Server Component** (gets all the data on the server, no serialization).
- Only the `Chart` component inside the `Suspense` boundary is "client-only" — and only at the per-instance level.
- The skeleton's `ChartSkeleton` renders during SSR (no client JS for the skeleton).
- The chart's `data` prop is passed by closure from the Server Component — no network serialization.

### Rules & limitations

- **`use(browser())` must be inside a `<Suspense>` boundary.** It's an error to use it at the root — you can't recover from the root. The `use()` hook throws, and the `Suspense` boundary above catches the throw.
- **`react-dom` only.** `browser()` is intentionally a `react-dom` API, not a `react` API, because the concept of "browser" doesn't apply generally to React (think React Native, custom renderers, server-only contexts).
- **Module-scope safe.** You can create `browser()` objects at module scope and use them in complex scenarios like "server rendering inside the browser while React is rendering" (the SSR-in-CSR case) — the API is isomorphic.
- **Currently canary-only.** The implementation is **flagged** so the React team can disable it quickly if they decide not to ship it in stable. As of `react@19.3.0-canary-0f42eac2-20260730`, the API is exported from `react-dom` and works in production usage. Track [`react@canary`](https://www.npmjs.com/package/react?activeTab=versions) for the stable promotion.

### When to use `browser()` vs `"use client"` vs `<ClientOnly>` HOC

| Pattern | Granularity | Server-rendered shell? | Network serialization? | Use when |
|---|---|---|---|---|
| `"use client"` (module-level) | Module | No (whole module is client) | Yes (RSC payload) | The component is genuinely interactive — state, effects, event handlers |
| `ReactDOM.browser()` + `use()` + `<Suspense>` | Per-instance | Yes (shell renders on server) | No (data passed by closure) | The component is "display-only" but needs `window` / WebGL / browser-only APIs |
| `<ClientOnly>` HOC (manual) | Per-instance | Yes (shell renders on server) | No (data passed by closure) | You can't use canary React yet — manual alternative to `browser()` |

**Recommendation:** for any new app on `next@16.3.0-canary.104`+ (or a standalone `react@canary`), prefer `browser()` over the manual `<ClientOnly>` HOC. It's the canonical, first-class API and it correctly suppresses the "real error" log that the HOC pattern triggers.

### Practical impact + bundle size

- **Bundle size:** `react-dom` core grows +3.67% (~240 bytes / ~80 bytes gzipped); full `react-dom-client` grows +0.05% (~120 bytes / ~50 bytes gzipped). See the `## React 19.3.0-canary-0f42eac2-20260730` section in `components.md` for the full size-bot table.
- **No new config flags, no breaking changes** — pure API addition.
- **Available on:** `next@16.3.0-canary.104+` (vendored via PR #96402), standalone `react@19.3.0-canary-0f42eac2-20260730+`, and `next@16.3.0-preview.11` once it ships (preview lags canary by 1 release). **Not in `next@latest` (16.2.12)** — that's stable React 19.2.8.

### Sources

- [React PR #37143 — `Add ReactDOM browser() API`](https://github.com/facebook/react/pull/37143) · gnoff · merged 2026-07-30T19:21:08Z
- [Next.js PR #96402 — `Upgrade React from 6cb4322d-20260729 to 0f42eac2-20260730`](https://github.com/vercel/next.js/pull/96402) · vercel-release-bot · merged 2026-07-30T21:21:08Z · **SHIPPED in `16.3.0-canary.104`**
- [React canary `19.3.0-canary-0f42eac2-20260730` on npm](https://www.npmjs.com/package/react/v/19.3.0-canary-0f42eac2-20260730) (published 2026-07-30T20:26:06Z)
- [`## React 19.3.0-canary-0f42eac2-20260730` in `components.md`](../components.md#react-1930-canary-0f42eac2-20260730--add-reactdombrowser-api-37143--3-devtools-prs-july-30-2026) — full API doc + bundle impact + 3 DevTools PRs

## Server Actions

Server Actions are functions that run on the server but can be called from client components — like an API endpoint you call directly:

### Defining a Server Action

```tsx
// app/actions.ts
'use server'

import { z } from 'zod'
import { revalidateTag } from 'next/cache'

const CreatePostSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1),
})

export async function createPost(formData: FormData) {
  const parsed = CreatePostSchema.parse({
    title: formData.get('title'),
    content: formData.get('content'),
  })

  await db.post.create({ data: parsed })
  // Use revalidateTag — Next.js 16 preferred revalidation method
  revalidateTag('posts', 'max')
}
```

### Calling from a Form

```tsx
// components/create-post-form.tsx
'use client'

import { createPost } from '@/app/actions'
import { useFormStatus } from 'react' // React 19: from 'react', not 'react-dom'
import { useActionState } from 'react' // React 19: useActionState for form state

function SubmitButton() {
  const { pending } = useFormStatus()
  return <button type="submit" disabled={pending}>{pending ? 'Saving...' : 'Create Post'}</button>
}

export function CreatePostForm() {
  // React 19 useActionState — replaces useFormState from react-dom
  const [state, formAction, isPending] = useActionState(createPost, null)
  
  return (
    <form action={formAction}>
      <input name="title" placeholder="Post title" />
      <textarea name="content" placeholder="Post content" />
      <SubmitButton />
    </form>
  )
}
```

## Common Mistakes

- **Putting `'use client'` on parent that wraps server children** — moves ALL children to client bundle
- **Trying to use `useState` in a server component** — add `'use client'` or lift state to parent
- **Passing non-serializable props** — functions and refs can't cross the RSC boundary
- **Sequential awaits when parallel is possible** — use `Promise.all` for independent fetches
- **Forgetting cache invalidation** — after mutations, use `revalidateTag` (background refresh) or `updateTag` (immediate expiration) to keep data fresh
- **`revalidateTag('posts')` without a profile** — single-arg form is deprecated as of Next.js 16.2. Use `revalidateTag('posts', 'max')` for stale-while-revalidate (recommended), or another `cacheLife` profile name, or `{ expire: number }`. Only use `updateTag` when you need immediate expiration for strong-consistency data.
- **Relying on implicit caching** — in Next.js 16, everything is dynamic by default; use `use cache` to opt into caching
- **Using `unstable_cache` in new code** — use `use cache` + `cacheTag` instead in Next.js 16
- **`use()` without an Error Boundary** — if the Promise rejects, `use()` throws; you need an Error Boundary to catch it
- **Reading cookies/headers inside `use cache`** — read them outside the cached scope and pass as arguments
- **`use()` for Context without `'use client'`** — this only works in Client Components; always add `'use client'` when consuming Context with `use()`
- **`use cache` surviving deploys without explicit invalidation** — cache persists across deployments; add deploy-time invalidation if fresh data is needed immediately after deploy

## `revalidateTag` vs `updateTag` — The Canonical Decision Matrix (Next.js 16.3 STABLE, August 3, 2026)

`next@16.3.0` STABLE ships `updateTag` from `next/cache` as a first-class API alongside the now-stable two-argument `revalidateTag(tag, profile)`. The two functions invalidate cached data by tag but **serve different invalidation models** — picking the wrong one causes stale UI (background refresh) or unnecessary blocking (immediate refresh). This is the canonical decision matrix; premature `updateTag` everywhere is a common mistake, and so is using `revalidateTag('x', 'max')` from a Server Action that needs read-your-own-writes.

**The rule of thumb:**

| Location | Use | Why |
| --- | --- | --- |
| **Server Action** (mutated from a `<form action>` / `<button formAction>` after a user gesture) | **`updateTag(tag)`** | Server Actions are event-driven — the user just clicked something and expects the UI to show the change. `updateTag` is **read-your-own-writes**: it expires the tag *and* blocks the next render until fresh data is fetched. The route that called the action immediately re-renders with the new data; no stale-data flash. |
| **Route Handler** (POST/DELETE/PATCH from a fetcher, a webhook, a cron, an external API call) | **`revalidateTag(tag, profile)`** | Route Handlers are external — the caller doesn't see the action's response. `revalidateTag` is **stale-while-revalidate**: the tag is marked stale, the next request that reads via `'use cache'` blocks the fresh fetch, and the cached entry is replaced in the background. Recommended profile: `'max'` (the largest `cacheLife` profile you have). |
| **Server Action that should NOT refresh the current route's UI** (e.g., background cleanup, audit log write) | **`revalidateTag(tag, 'max')`** | `updateTag` would force a re-render of the current route even though the user shouldn't see the side effect. Use `revalidateTag` with the lightweight SWR profile — the data is fresh on the next navigation, but the current page doesn't re-render pointlessly. |

**The two-argument `revalidateTag` signature is now stable** (was experimental in 16.2; promoted to stable in 16.3.0 STABLE). The single-argument `revalidateTag('posts')` form is **deprecated** — TypeScript enforces the two-argument form (the signature `revalidateTag(tag: string, profile: string | { expire?: number }): void` throws if the second argument is missing under `strict` mode in 16.3+). The `profile` argument accepts either a string (the name of a configured `cacheLife` profile like `'max'`, `'hours'`, `'minutes'`, `'days'`, `'seconds'`) or an inline `{ expire: number }` object that overrides the expiry directly.

**`updateTag` signature is intentionally minimal** — `updateTag(tag: string): void`. It takes a single tag (no profile — the semantics are "expire NOW", profile is irrelevant). It can only be called **inside a Server Action** (Next.js 16.3 throws at runtime if you call it from a Route Handler, Client Component, or Proxy). The function invalidates cache entries for the tag AND invalidates the affected routes in the full route cache as read-your-own-writes — but only the route the Server Action was called from immediately re-renders with fresh data. Other routes that reference the same tag re-render on their next navigation.

**The canonical patterns:**

```tsx
// app/actions.ts — Server Action with read-your-own-writes
'use server'

import { updateTag } from 'next/cache'

export async function createPost(formData: FormData) {
  await db.post.create({ data: parse(formData) })
  updateTag('posts') // ✅ Server Action → updateTag
  // The page that called this action re-renders with the new post visible immediately.
}
```

```tsx
// app/api/posts/route.ts — Route Handler with stale-while-revalidate
import { revalidateTag } from 'next/cache'

export async function POST(req: Request) {
  await db.post.create({ data: await req.json() })
  revalidateTag('posts', 'max') // ✅ Route Handler → revalidateTag with profile
  // Next request to any route that uses 'use cache' with cacheTag('posts')
  // triggers a SWR refresh — stale is served while fresh is fetched in background.
}
```

```tsx
// app/actions.ts — Server Action that does NOT need to re-render the UI
'use server'

import { revalidateTag } from 'next/cache'

export async function auditLogWrite(entry: AuditEntry) {
  await db.audit.insert(entry)
  revalidateTag('audit', 'max') // ✅ Server Action → revalidateTag (not updateTag)
  // The 'audit' tag is marked stale, but we don't want to re-render the user's
  // current page just because the audit log got a new entry. The next page
  // navigation that reads 'audit' will see fresh data.
}
```

**The antipatterns (with concrete audit recipes):**

```tsx
// ❌ ANTI-PATTERN 1: revalidateTag in Server Action (loses read-your-own-writes)
'use server'
export async function createPost(formData: FormData) {
  await db.post.create({ data: parse(formData) })
  revalidateTag('posts', 'max') // ❌ Stale data is served until next navigation
  // The user just clicked "Create Post" and expects to see it. With revalidateTag,
  // the current page still shows the old list (the cache is stale, not expired).
}
```

```tsx
// ❌ ANTI-PATTERN 2: updateTag in Route Handler (throws at runtime)
import { updateTag } from 'next/cache'

export async function POST(req: Request) {
  await db.post.create({ data: await req.json() })
  updateTag('posts') // ❌ Throws: updateTag can only be called from within a Server Action
}
```

```tsx
// ❌ ANTI-PATTERN 3: Single-argument revalidateTag (deprecated, throws under strict TS)
export async function createPost() {
  revalidateTag('posts') // ❌ Deprecated. Add a profile: revalidateTag('posts', 'max')
}
```

**Audit recipe — find Server Actions that should use `updateTag` but are using `revalidateTag`:**

```bash
# Find all Server Actions in the app that mutate and invalidate
rg -l "use server" --type ts --type tsx app/ | \
  xargs rg -l "revalidateTag" | \
  xargs rg -B1 -A1 "revalidateTag" | \
  rg -v "max|expire|profile"  # Flags any revalidateTag call without a profile arg
```

For each match, decide: if the action is invoked from a `<form action>` after a user gesture and the user expects to see the change, switch to `updateTag`. If the action is a background audit / cleanup / out-of-band write, keep `revalidateTag` with `'max'`.

**Why the split exists (architectural rationale):** `updateTag` was introduced in Next.js 16.2 because the old model conflated "invalidate the cache" with "force the current route to re-render right now" — and those two operations have different semantics. The router-level re-render in `updateTag` is what enables read-your-own-writes; `revalidateTag` only invalidates the cache entry, so the current route (if it has the tag's data already in props or a cached component) continues to render the stale value until the next signal (navigation, refresh, or revalidatePath). Splitting them lets the framework optimize: `revalidateTag(tag, 'max')` doesn't need to touch the router at all — it's a pure cache-control operation. `updateTag` triggers a router-level update that resolves the promise, re-renders the current route, and replaces the affected cache entries.

**Practical impact by Next.js tag:**

- **Apps on `next@16.3.0` STABLE (August 3, 2026):** `revalidateTag(tag, profile)` is the **only** supported form (the single-argument form throws under `strict` and is deprecated regardless). `updateTag` is first-class in `next/cache`. Codemod available: `npx @next/codemod@latest update-tag-2-arg` (if you have a large existing codebase, run this to convert single-arg `revalidateTag` calls to `revalidateTag(tag, 'max')`).
- **Apps on `next@16.2.x`:** `revalidateTag(tag, profile)` is **experimental** (works but logs a warning). `updateTag` is **not available** — fall back to `revalidateTag(tag, 'max')` for SWR or `revalidateTag(tag, { expire: 0 })` for read-your-own-writes (the inline object form is the v16.2 equivalent of `updateTag`).
- **Apps on `next@15.x`:** `revalidateTag(tag)` is the only supported form. Upgrade to 16.3+ to get the better split.

**When to use `revalidatePath` instead of either:** if the cache key is **a path** (not a tag), use `revalidatePath('/blog')` — it's a different function in the same `next/cache` module, and the two-arg form is the same. The `revalidatePath` / `revalidateTag` / `updatePath` / `updateTag` quartet is the canonical Next.js 16.3 invalidation API.

**Sources:**

- [Next.js `updateTag` API reference](https://nextjs.org/docs/app/api-reference/functions/updateTag) — single-arg signature, Server Actions only, read-your-own-writes semantics
- [Next.js `revalidateTag` API reference](https://nextjs.org/docs/app/api-reference/functions/revalidateTag) — two-arg required signature, profile-stamped SWR, Server Actions + Route Handlers
- [Next.js Caching — `use cache` directive](https://nextjs.org/docs/app/api-reference/directives/use-cache) — `cacheTag` + `cacheLife` + the `use cache` boundary
- [Next.js 16 release notes — Cache Components](https://nextjs.org/blog/next-16) — `updateTag` introduced as the canonical read-your-own-writes invalidation API
- [GitHub Discussion #84805 — `updateTag` vs `revalidateTag`](https://github.com/vercel/next.js/discussions/84805) — community-curated decision matrix, the source of the "server action → updateTag, route handler → revalidateTag" heuristic
- [Dev.to — `revalidateTag` & `updateTag` in Next.js (12-part series)](https://dev.to/peterlidee/revalidatetag-updatetag-in-nextjs-4j8b) — practical walkthroughs of the SWR vs RYOW trade-off
- [Next.js Cache Components Migration Guide](https://nextjs.org/docs/app/guides/migrating-to-cache-components) — `updateTag` + `revalidateTag` + `use cache` + `cacheTag` patterns

## Cache Components — 16.3 Canary Hardening (canary.72–78, June 30–July 4, 2026)

Eight material PRs landed in 16.3 canary.72 → canary.78 that refine how `'use cache'`, `cacheLife()`, and instant validation behave. They are not breaking changes for normal usage, but they remove silent footguns and tighten the type surface:

### 1. `'use client'` `await params`/`await searchParams` no longer crash dev instant validation — canary.72

Client components that read `await props.params` or `await props.searchParams` via `use()` on a Cache Components (`cacheComponents: true`) dynamic route used to crash in dev instant validation with `Invariant: Expected to have a workUnitStore that provides validationSampleTracking. This is a bug in Next.js.` Dev-only — build passes, so CI suites miss it. Fixed in [PR #95289](https://github.com/vercel/next.js/pull/95289) by Janka Uryga (canary.72, 2026-06-30T14:38:02Z). The fix removes the validation-tracking case from `createParamsFromClient` and gates it on `workUnitStore.validationSamples` for search-params.

**Practical impact:** if you read `await params` in a `'use client'` component on a dynamic route, you no longer see the redbox. This was the 6th silent-corruption fix in the past week.

### 2. Constants renamed: `DYNAMIC_EXPIRE` → `MIN_PRERENDERABLE_EXPIRE` (300s) and `DYNAMIC_STALE` → `MIN_PREFETCHABLE_STALE` (30s) — canary.74

The internal thresholds that gate when a `'use cache'` entry is treated as dynamic (300s) or as eligible for prefetching (30s) were renamed for clarity. **No behavior change** — only the symbol names changed. Source: [PR #95361](https://github.com/vercel/next.js/pull/95361) by Hendrik Liebau, merged 2026-07-01T18:33:02Z.

```ts
// Before (canary.73 and earlier) — these names no longer exist:
// import { DYNAMIC_EXPIRE, DYNAMIC_STALE } from '...' // ❌ won't compile

// After (canary.74+) — same numeric values, clearer names:
// MIN_PRERENDERABLE_EXPIRE = 300  (5 minutes — gate for prerenderability)
// MIN_PREFETCHABLE_STALE  = 30   (30 seconds  — gate for prefetch eligibility)
```

**Practical impact:** if you import these constants anywhere (advanced — most apps don't), update the import. If you've pinned to canary.73, also bump.

### 3. New "Link Data" validation errors for `params`/`searchParams` accessed outside `<Suspense>` under `partialPrefetching` — canary.73

On `partialPrefetching: true` (or per-segment `prefetch = 'partial'` / `'unstable_eager'`) routes, `<Link>` prefetches an App Shell that cannot access link data. The instant validation system in canary.73 adds three new blocking-route errors to catch this:

| Error | Builder | Trigger |
|---|---|---|
| 1390 | `createLinkBodyErrorInNavigation` | `params`/`searchParams` accessed in a body Suspense boundary that doesn't include them |
| 1391 | `createLinkMetadataError` | `params`/`searchParams` accessed in `generateMetadata` outside `<Suspense>` |
| 1392 | `createLinkViewportError` | `params`/`searchParams` accessed in viewport boundary code |

New `DynamicHoleKind.Link = 1` in `packages/next/src/server/app-render/dynamic-rendering.ts` (renumbers `Runtime` 1→2 and `Dynamic` 2→3); new `ShellRuntime` stage between `Static` and `Runtime` in `instant-validation.tsx`'s `SEGMENT_STAGE_ORDER`. Detection: if a hole is present in `ShellRuntime` but disappears in `Runtime`, it's link data.

```tsx
// ❌ Triggers error 1390 on partialPrefetching routes
async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params  // link data, not in Suspense
  return <PostBody id={id} />
}

// ✅ Wrap in Suspense
async function Page({ params }: { params: Promise<{ id: string }> }) {
  return (
    <Suspense fallback={<Skeleton />}>
      <PostBody params={params} />
    </Suspense>
  )
}
```

Sources: [PR #95151](https://github.com/vercel/next.js/pull/95151) (Janka Uryga, 2026-07-01T04:36:41Z) + [PR #94595](https://github.com/vercel/next.js/pull/94595) (follow-up for `generateStaticParams`, 2026-07-01T06:59:43Z). Only fires with `cacheComponents: true` and `partialPrefetching: true` (or per-segment `prefetch = 'partial'`).

#### 4a. App Shell cache-miss fix for `generateStaticParams` — PR #95665 (canary-branch ahead of canary.93, ships in canary.94 ~2026-07-23T23:00Z)

[PR #95665](https://github.com/vercel/next.js/pull/95665) (Janka Uryga, merged 2026-07-22T15:18:51Z, closes `NAR-883`) fixes a **silent cache-miss footgun** in the same code path. The bug: when an App Shell is being prerendered, the `ShellRuntime` stage is the ceiling (URL data is excluded; the prerender doesn't advance beyond `ShellRuntime`). But the prospective runtime prerender — the one that decides which inputs are "hanging" — was letting `params` / `searchParams` resolve normally instead of marking them as hanging. When `params` ended up being inputs to a cached page, those inputs weren't hanging in the prospective render and then became a cache miss in the final render (which IS past `ShellRuntime`).

**The fix** adds `readonly isSessionShell: boolean` to `PrerenderStoreModernRuntime` and threads it through `createRuntimePrerenderParams`, which now calls `makeHangingParams(underlyingParams, workStore, workUnitStore)` when in `ShellRuntime` mode (was: `makeUntrackedParams(userspaceParams)`). For route handlers and other code that reads `searchParams` during prerender, new error code #1449 `"Accessed \`searchParams\` during prerendering."` enforces the new contract.

**Practical effect:** if you use `cacheComponents: true` + `generateStaticParams` + `partialPrefetching: true` (or per-segment `prefetch = 'partial'` / `'unstable_eager'`) and noticed duplicate data fetches / shell-then-runtime cache misses after adding `generateStaticParams`, **upgrading to `next@canary@94` fixes it without code changes**. No public API change.

Audit: `rg 'isSessionShell' packages/next/src/server/` should return 5 hits across `app-render.tsx`, `work-unit-async-storage.external.ts`, `request/params.ts` after canary.94 lands. Full impact analysis in `performance.md` → "Cache-Miss Fix in App Shell for Cached Pages with `generateStaticParams`".

**Source:** [PR #95665](https://github.com/vercel/next.js/pull/95665) · 5 files +33/-11 · Janka Uryga · merged 2026-07-22T15:18:51Z · commit `63f14c6c90` · closes `NAR-883` · **Ships in `16.3.0-canary.94`**.

### 4. Short-`expire` `'use cache'` values now retain across dev reloads — canary.75

Companion to #5: in dev, the built-in default handler now keeps short-`expire` entries for at least `MIN_PRERENDERABLE_EXPIRE` (300s) and re-warms them in the background on every dynamic-request render. Before this PR, a short `expire` meant the next reload regenerated the value from scratch.

```ts
'use cache'
cacheLife({ expire: 30 })  // 30 seconds
export async function getDashboardStats() {
  // dev: served from cache for 5 minutes (re-warmed in background)
  // prod: regenerated every 30s as expected
  return db.stats.findFirst()
}
```

Source: [PR #95362](https://github.com/vercel/next.js/pull/95362) by Hendrik Liebau, merged 2026-07-02T09:26:09Z.

### 5. `expire: 0` no longer persists to the default cache handler in production — canary.76

`cacheLife({ expire: 0 })` produces a value that is expired the moment it is produced — the `'use cache'` wrapper regenerates it on every read rather than serving the stored copy. The built-in default handler used to write the never-to-be-served entry anyway; now it skips the `set()` call in production.

```ts
// lib/feature-flags.ts
'use cache'
export async function getFeatureFlags(userId: string) {
  // expire: 0 keeps the value out of the server-side cache handler
  // — useful for client-only caching via Cached Navigations / Runtime Prefetches,
  //   or for opt-out inside a function that has an otherwise normal cacheLife.
  cacheLife({ expire: 0 })
  // ...
}
```

**What this means:**
- **Production** (no `__NEXT_DEV_SERVER`): default handler skips the `set()`. Zero round-trip to remote handlers, no stored payload.
- **Development**: still stores the entry, because the default handler's minimum retention serves the previously cached value across reloads while the entry re-warms in the background.
- A custom cache handler can opt into the same skip or not — it's per-handler.

Source: [PR #95363](https://github.com/vercel/next.js/pull/95363) by Hendrik Liebau, merged 2026-07-02T11:36:41Z.

### 6. False-positive nested-cache error fixed for a short default profile — canary.76

Overriding the `default` `cacheLife` profile with a short cache life previously made the nested-`'use cache'` error fire in two cases where it should not have:

1. A single non-nested `'use cache'` with no inline `cacheLife()` — the error fired with no nesting at all (killed the build in prod, threw in dev).
2. A genuinely nested cache where the developer already opted into a short default — the warning was pointless because the default profile already makes every cache a dynamic hole.

The error now requires (a) a dynamic nested cache that propagated its short life upward (`dynamicNestedCacheError` is set) AND (b) the `default` profile is itself prerenderable. Otherwise the short-lived entry is omitted as a dynamic hole instead — exactly as an inline short `cacheLife()` already was.

**Practical impact:** if you set a short `default` cacheLife profile globally, nested caches no longer fail builds. You can use `'use cache'` freely under a short default profile.

Source: [PR #95373](https://github.com/vercel/next.js/pull/95373) by Hendrik Liebau, merged 2026-07-02T11:49:53Z.

### 7. `ResolvedCacheLifeProfiles` — `cacheLife` is now non-optional and the `default` profile is `Required<CacheLife>` — canary.77

The runtime used to assert the default `cacheLife` profile on every `'use cache'` call because the type allowed partial profiles. Config normalization (`assignDefaultsAndValidate`) already guarantees a resolved `default` profile at config-load time, so the type system now reflects that guarantee.

```ts
// Before (canary.76 and earlier) — runtime asserts and optional chaining everywhere:
// cacheLifeProfiles?.default?.stale  // runtime checks at every read
// assertDefaultCacheLife(cacheLifeProfiles) // throws if missing

// After (canary.77+) — type-level guarantee, no runtime asserts:
// cacheLifeProfiles.default.stale  // direct read, no nullish check needed
```

**What changed:**
- New `ResolvedCacheLifeProfiles` type in `packages/next/src/server/config-shared.ts` types the `default` profile as `Required<CacheLife>`.
- The type is threaded (non-optional) through `NextConfigComplete.cacheLife`, render options, work store, and the build/export/dev workers.
- `assertDefaultCacheLife` and the per-`cacheLife()` presence `InvariantError` are **deleted** from the runtime.
- The proxy/middleware work store gets a sentinel whose `default` getter throws if ever read (proxy never reads `cacheLife`).

**Practical impact:** your TS configs compile slightly faster (no extra type narrowing), and you can rely on `cacheLife.default.stale` / `cacheLife.default.expire` existing at runtime. The `cacheLife()` helper no longer throws when the default profile is missing — the type system prevents that path at compile time.

Source: [PR #95428](https://github.com/vercel/next.js/pull/95428) by unstubbable, merged 2026-07-03T19:35:11Z.

### 8. `experimental.serverComponentsHmrCancellation` flag (inert) — canary.78

A new `experimental.serverComponentsHmrCancellation?: boolean` flag was added in canary.78 that **does nothing on its own**. The plumbing is in place; a follow-up PR will use it to cancel a Server Components HMR refresh once a newer refresh supersedes it. Default `false`. Behind `__NEXT_EXPERIMENTAL_SERVER_COMPONENTS_HMR_CANCELLATION` env var for CI. Production SSR hardcodes `false` (edge rendering doesn't expose the Node response-close signal the cancellation relies on).

**Practical impact:** none yet — opt in only if you want to test the upcoming cancellation behavior in dev. Source: [PR #95462](https://github.com/vercel/next.js/pull/95462) by unstubbable, merged 2026-07-04T12:23:35Z.


### 9. `experimental.devValidationWorker` — Dev Validation on a Worker Thread (canary.97 ahead — PR [#96150](https://github.com/vercel/next.js/pull/96150) + #96153, July 25, 2026)

A new `experimental.devValidationWorker?: boolean` flag landed on the canary branch immediately after the canary.96 tag (PR #96150, merged 2026-07-25T05:21:02Z, by the Next.js team). It defaults to **`true`** — meaning dev-mode Cache Components validation now runs on a worker thread instead of the dev server's main thread. Set it to `false` only as an escape hatch if the worker misbehaves.

**The problem:** with `cacheComponents: true`, the dev server validates every Cache Components render and ships any errors to the browser overlay. On rapid back-to-back navigation (e.g. clicking links quickly across deeply-nested route trees), the validation render previously **monopolized the dev server's main event loop**, causing the next navigation to wait for the prior validation to complete before its request handler could even start. The synthetic `pnpm bench:dev-validation` benchmark measured worst-case TTFB inflation up to ~7.7 seconds on a single client-rendered route.

**What changed:** PR #96153 moves the entire Cache Components dev validation render (the client prerender + Flight re-encode + validation lifecycle) off the main thread onto a dedicated worker (`dev-validation-worker.ts` + a single-worker pool). The worker loads the app-page bundle exactly once via `ComponentMod.routeModule.runValidationInDev`, rebuilds the render context / work store / request store from a serializable snapshot, runs the whole validation there, and forwards the resulting errors back to the main thread as Flight bytes — the main thread only delivers them to the overlay via `sendErrorsToBrowser`. A one-slot `SharedArrayBuffer` propagates a "supersede" abort into the worker so a newer navigation cancels an in-flight validation. The worker is installed at server boot when `experimental.devValidationWorker !== false`, and `runDevValidationInBackground` uses it when present, falling back to the in-process path otherwise.

**Benchmark results** (synthetic `pnpm bench:dev-validation`, back-to-back clicks — worst-case scenario for main-thread contention, browser-observed navigation TTFB):

| Route family | Worker (p50 / p95 / max) | In-process (p50 / p95 / max) |
| --- | --- | --- |
| `client` (client-component-heavy) | **19 / 24 / 27 ms** | 40 / 66 / **7,762 ms** |
| `server` (server-component-heavy) | **42 / 45 / 46 ms** | 122 / 158 / 252 ms |

The `client` route's `max` is the headline — the worker removes a ~7.7-second main-thread stall that appeared as the worst-case tail on deep app pages during fast navigation.

**The companion refactor stack (PRs #96148 / #96149 / #96151 / #96152 / #96175)** prepared the worker landing without behavior changes:

- **#96148 — Forward dev invalid dynamic usage errors from the render, not validation**: moves the error forwarding (and the `cacheReady()` wait that finalizes the verdict) out of `runDevValidationInBackground` into the render's completion handler. Validation becomes a pure consumer of the settled render — it no longer reads or forwards the error, takes the settled render result rather than its promise, and is simply not scheduled when the render recorded such an error (nothing to validate). Consolidates two duplicate forwarding paths into one place at the render level.
- **#96149 — Model dev validation render outputs as a discriminated union**: splits `DevValidationInputs` into `ResolvedValidationInputs | SyncInterruptedStagedDevRender`, discriminated by the presence of `syncInterruptReason`. A render's settled output becomes `StagedDevRenderResult` whose `outcome` is the same union, with `hadCacheMiss` lifted onto the result (orthogonal to how the render ended). Pure refactor — behavior-preserving.
- **#96151 — Prepare dev validation for running on a worker thread**: narrows the context `runValidationInDev` and its helpers require to a `ValidationRenderContext` (a `Pick` of `AppRenderContext` plus the two `renderOpts` fields and the debug-channel flag). Splits "compute errors" from "deliver errors" so the worker can compute and the main thread can deliver (delivery needs the response object).
- **#96152 — Add a benchmark for dev Cache Components validation on a worker thread**: new `bench/dev-validation/`, wired as `pnpm bench:dev-validation`, A/B-tests the flag against the in-process path using browser-observed TTFB from Playwright. Prints absolute p50/p95/max side-by-side (no ratio) because the worker's win is bounded by the validation render's CPU — not by total request time.
- **#96175 — [test] Unflake the `enabled-features-trace` test suite**: test-only fix; the suite was flaky on the validation lifecycle markers.

**Practical impact (for agents building apps):**

- **Massive dev-time wins on `cacheComponents: true` projects with deep routes and rapid navigation.** If you previously saw "validation stalls" or unexplainable long TTFBs in dev when clicking `<Link>`s quickly, they should disappear once canary.97 ships.
- **No code change required** — flag defaults to `true` on the canary branch. Just upgrade and the win lands.
- **Escape hatch:** `experimental.devValidationWorker: false` falls back to the in-process path. Use only if you observe worker misbehavior (errors mentioning `dev-validation-worker`); not a perf tuning knob.
- **Benchmarking your own app:** once canary.97 ships, run `pnpm bench:dev-validation` against your real app routes (or adapt the fixture) to quantify the win on your specific shapes.
- **Build-mode validation is unaffected.** The worker only handles `next dev` validation. `next build` validation (which catches the same `cacheComponents` violations earlier in the pipeline) still runs in-process.

**Will land in `16.3.0-canary.97` — expected ~2026-07-26T22:30Z** (24h after the canary.96 tag was cut at 2026-07-25T00:00:34Z). Until then, install with `npm install next@canary` to pick up the canary-branch ahead of canary.96.
### 9a. Dev Validation Worker — Follow-up Fixes (4 PRs, 13:15–14:30Z, 2026-07-25)

Right after the worker landed, four tightly-scoped PRs arrived on the canary branch to repair regressions exposed by the move. None change the headline design — all keep `experimental.devValidationWorker: true` as the default — but each closes a real footgun that operators would have hit on the first canary.97 build.

**PR [#96218](https://github.com/vercel/next.js/pull/96218) — Read chunk source maps from disk in the dev validation worker** (unstubbable, merged 2026-07-25T14:30:40Z). Node.js caches source maps per isolate, so the dev validation worker only has maps for the chunks it has already loaded on its own thread. Anything split into a chunk of its own — most visibly a Turbopack dynamic `import()` — never appears in that cache, and any stack frame pointing into it was logged without a source location. In-process validation resolves these frames by asking Turbopack through the `Project` handle; that handle cannot cross the worker thread boundary, so the worker instead reads the `.map` file that Turbopack already writes next to the chunk. Lookups are restricted to `distDir` so a stack frame cannot point the reader at an arbitrary file, and both hits and misses are memoised. Implementation note: this required hoisting `bundlerFindSourceMapPayload` to a `globalThis` symbol, because `patch-error-inspect` is bundled into several runtimes that each get their own copy, and the copy that registers the implementation (the worker bundle) is not the copy that symbolicates the frame (the app-page bundle). The code-frame renderer in the same file is already shared this way for the same reason.

**PR [#96219](https://github.com/vercel/next.js/pull/96219) — Run dev validation in process when using Webpack** (unstubbable, merged 2026-07-25T14:30:41Z). The same source-map-isolation problem affects Webpack differently: Webpack keeps its dev source maps in the compiler rather than writing them next to the chunks, so once PR #96218's disk-read fallback is in place there is still nothing for the worker to read when running under Webpack. The fix is to detect the bundler and turn the worker off — `experimental.devValidationWorker` now has no effect under Webpack, and validation runs in process again. Dev performance under Webpack goes back to what it was before the worker landed, which is the right trade: the worker exists to keep the event loop responsive during rapid navigation, and that is not worth losing the source location on validation errors. If you are on Webpack (the default if you have not opted into Turbopack) and you set `experimental.devValidationWorker: true`, that flag is now silently ignored — add `turbopack: {}` to `next.config.ts` to actually get the worker.

**PR [#96215](https://github.com/vercel/next.js/pull/96215) — Retry the source map lookup with a plain path** (unstubbable, merged 2026-07-25T13:15:50Z). Node.js 22+ escapes the brackets in Turbopack's `[root-of-the-server]` chunk names when computing `pathToFileURL`, so a cache lookup by the file URL misses against a chunk that was inserted by its plain path. Node.js 20 leaves brackets alone (which is why the original CI was green); the local repro on Node.js 24 fails in `test/e2e/app-dir/instant-validation/suspense-boundaries.test.ts` for the two "missing suspense around search params" cases — stacks that should read `app/…/page.tsx:40:18` are logged as raw `about://React/Prefetch/file:///…/%5Broot-of-the-server%5D…` URLs. PR #95946 (the `file:` source maps switch) anticipated the mismatch and left it as a follow-up, because on the main thread the miss is only wasteful: Turbopack hands back the chunk's map instead. The worker has no Turbopack project handle, so a miss is final. The fix retries the lookup with the plain path before giving up — Node.js accepts that key unambiguously for both CommonJS absolute paths and ESM `file:` URLs. Long-term, [React PR #37105](https://github.com/facebook/react/pull/37105) would make React's fake frame URLs reversible so the first lookup succeeds on its own and retire the retry. The new test snapshots the worker's coverage shape: it caches maps per chunk file in its own isolate and never renders server components, so statically imported modules sit in a chunk it has already loaded and resolve, while dynamically imported modules get a chunk of their own that is never loaded and do not. The snapshots for those second routes record broken output on purpose — in-process validation resolves the frame, so losing it is a regression from moving validation onto a worker, and it affects Turbopack (no location) and Webpack (compiled positions) alike. The snapshots are recorded with the retry applied so they are stable across Node.js versions.

**PR [#96210](https://github.com/vercel/next.js/pull/96210) — Pass fallback params to the dev validation worker as maps** (unstubbable, merged 2026-07-25T13:15:35Z). Internal cleanup of the same area. The dev validation snapshot flattened both fallback param sets into arrays of entries before handing them to the worker, and the worker rebuilt a `Map` from each one on the other side. The structured clone that carries the snapshot across the `worker_threads` boundary supports `Map` natively, so neither step is needed. Dropping the conversion on both ends lets the two snapshot fields be typed as `OpaqueFallbackRouteParams` directly, and it makes them consistent with `rootParams` and `implicitTags`, which the snapshot already passes through untouched. The neighbouring `headers` spread stays as it is because `Headers` is not a structured-cloneable type and those entries genuinely do have to be materialised before the snapshot is sent.

**Net effect for operators on canary.97:**

- **Turbopack users (default for `next dev` with `turbopack: {}`):** the worker is fully active, and the four new PRs close the source-map regressions exposed by the move. Stack frames pointing into dynamic-import chunks now resolve; Node.js 22+ users no longer see raw `about://React/...` URLs in their terminal. The `p50/p95/max` TTFB numbers in the table above are realised.
- **Webpack users:** validation runs in process again. `experimental.devValidationWorker` is silently ignored under Webpack. If you want the worker, opt into Turbopack. If you are stuck on Webpack, the worst-case `client` TTFB inflation is back, but the validation errors you see carry correct source locations.
- **Anyone on Node.js 22 or 24:** the bracket-escaping fix means CI failures on Node.js 24 that did not show up on Node.js 20 are now caught before merge.
- **Escape hatch:** `experimental.devValidationWorker: false` still works under Turbopack — falls back to the in-process path, with the same source-map coverage as before canary.97.

**Sources:**
- [PR #96218 — Read chunk source maps from disk in the dev validation worker](https://github.com/vercel/next.js/pull/96218) · merged 2026-07-25T14:30:40Z · **canary-branch ahead of canary.96**
- [PR #96219 — Run dev validation in process when using Webpack](https://github.com/vercel/next.js/pull/96219) · merged 2026-07-25T14:30:41Z · **canary-branch ahead of canary.96**
- [PR #96215 — Retry the source map lookup with a plain path](https://github.com/vercel/next.js/pull/96215) · merged 2026-07-25T13:15:50Z · **canary-branch ahead of canary.96**
- [PR #96210 — Pass fallback params to the dev validation worker as maps](https://github.com/vercel/next.js/pull/96210) · merged 2026-07-25T13:15:35Z · **canary-branch ahead of canary.96** (internal cleanup)



**Sources:**
- [PR #96148 — Forward dev invalid dynamic usage errors from the render, not validation](https://github.com/vercel/next.js/pull/96148) · merged 2026-07-25T05:21:01Z · **canary-branch ahead of canary.96**
- [PR #96149 — Model dev validation render outputs as a discriminated union](https://github.com/vercel/next.js/pull/96149) · merged 2026-07-25T05:21:01Z · **canary-branch ahead of canary.96** (refactor, no behavioral change)
- [PR #96150 — Add the `experimental.devValidationWorker` config flag](https://github.com/vercel/next.js/pull/96150) · merged 2026-07-25T05:21:02Z · **canary-branch ahead of canary.96**
- [PR #96151 — Prepare dev validation for running on a worker thread](https://github.com/vercel/next.js/pull/96151) · merged 2026-07-25T05:21:02Z · **canary-branch ahead of canary.96** (behavior-preserving refactor)
- [PR #96152 — Add a benchmark for dev Cache Components validation on a worker thread](https://github.com/vercel/next.js/pull/96152) · merged 2026-07-25T05:21:03Z · **canary-branch ahead of canary.96**
- [PR #96153 — Run Cache Components dev validation on a worker thread](https://github.com/vercel/next.js/pull/96153) · merged 2026-07-25T05:21:03Z · **canary-branch ahead of canary.96**
- [PR #96175 — [test] Unflake the enabled-features-trace test suite](https://github.com/vercel/next.js/pull/96175) · merged 2026-07-25T05:21:02Z · **canary-branch ahead of canary.96** (test-only)
- [Next.js `cacheComponents` config reference](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents)
- [Next.js Migration to Cache Components guide](https://nextjs.org/docs/app/guides/migrating-to-cache-components)

**Sources:**
- [Next.js `use cache` directive](https://nextjs.org/docs/app/api-reference/directives/use-cache)
- [Next.js 16 release notes](https://nextjs.org/blog/next-16)
- [Next.js `cacheTag`](https://nextjs.org/docs/app/api-reference/functions/cacheTag)
- [Next.js `revalidateTag`](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)
- [Next.js `updateTag`](https://nextjs.org/docs/app/api-reference/functions/updateTag)
- [React 19.2.7 release](https://github.com/facebook/react/releases/tag/v19.2.7)
- [Next.js PR #95289 — client page params/searchParams dev validation fix](https://github.com/vercel/next.js/pull/95289)
- [Next.js PR #95361 — DYNAMIC_EXPIRE→MIN_PRERENDERABLE_EXPIRE rename](https://github.com/vercel/next.js/pull/95361)
- [Next.js PR #95151 — Validate Shell prefetches (Link Data errors 1390-1392)](https://github.com/vercel/next.js/pull/95151)
- [Next.js PR #94595 — Link Data errors for generateStaticParams](https://github.com/vercel/next.js/pull/94595)
- [Next.js PR #95362 — short-expire dev reload retention](https://github.com/vercel/next.js/pull/95362)
- [Next.js PR #95363 — skip expire:0 set() in prod](https://github.com/vercel/next.js/pull/95363)
- [Next.js PR #95373 — false-positive nested-cache error fix](https://github.com/vercel/next.js/pull/95373)
- [Next.js PR #95428 — ResolvedCacheLifeProfiles typing](https://github.com/vercel/next.js/pull/95428)
- [Next.js PR #95462 — serverComponentsHmrCancellation flag](https://github.com/vercel/next.js/pull/95462)
- [Next.js PR #96148 — Forward dev invalid dynamic usage errors from the render](https://github.com/vercel/next.js/pull/96148) · **canary-branch ahead of canary.96**
- [Next.js PR #96149 — Model dev validation render outputs as a discriminated union](https://github.com/vercel/next.js/pull/96149) · **canary-branch ahead of canary.96** (refactor)
- [Next.js PR #96150 — Add the `experimental.devValidationWorker` config flag](https://github.com/vercel/next.js/pull/96150) · **canary-branch ahead of canary.96**
- [Next.js PR #96151 — Prepare dev validation for running on a worker thread](https://github.com/vercel/next.js/pull/96151) · **canary-branch ahead of canary.96** (behavior-preserving refactor)
- [Next.js PR #96152 — Add a benchmark for dev Cache Components validation on a worker thread](https://github.com/vercel/next.js/pull/96152) · **canary-branch ahead of canary.96**
- [Next.js PR #96153 — Run Cache Components dev validation on a worker thread](https://github.com/vercel/next.js/pull/96153) · **canary-branch ahead of canary.96**
- [Next.js PR #96175 — Unflake the `enabled-features-trace` test suite](https://github.com/vercel/next.js/pull/96175) · **canary-branch ahead of canary.96** (test-only)
- [PR #96218 — Read chunk source maps from disk in the dev validation worker](https://github.com/vercel/next.js/pull/96218) · **canary-branch ahead of canary.96**
- [PR #96219 — Run dev validation in process when using Webpack](https://github.com/vercel/next.js/pull/96219) · **canary-branch ahead of canary.96**
- [PR #96215 — Retry the source map lookup with a plain path](https://github.com/vercel/next.js/pull/96215) · **canary-branch ahead of canary.96**
- [PR #96210 — Pass fallback params to the dev validation worker as maps](https://github.com/vercel/next.js/pull/96210) · **canary-branch ahead of canary.96** (internal cleanup)

## Flight — PR #37258 Transfer Key Validation of Lazy Nodes When Unwrapped (Aug 10, 2026) + Next.js Cache Components PR #97040 Static/App-Shell Incompatibility Tracking

**[10 Aug 2026 18:02Z] v1.5.46 cycle**, **2 NEW server-component-relevant commits** since the v1.5.43 / v1.5.45 cycles:

1. **React Flight — PR #37258 (unstubbable / Hendrik Liebau, merged 2026-08-10T14:18:47Z, 2 files / +326/-16, base `main`)** — **SHIPPED in `react@19.3.0-canary-807d21fd-20260810`** (the new React canary cut, npm-published 2026-08-10). Fixes a **false-positive `Each child in a list should have a unique "key" prop` warning** in Flight-outlined values. **Detailed components.md coverage of the SHIP event + the full per-PR deep dive of PR #37258 below.**

2. **Next.js Cache Components — PR #97040 (lubieowoce, merged 2026-08-10T16:29:50Z, 7 files / +91/-47, base `canary`)** — **will ship in `next@16.3.1-canary.11`** (npm-published expected ~2026-08-11T07:41Z ± a few hours on the 24h cadence; canary-branch currently 11 commits ahead of canary.10). Adds **dynamic tracking of API usage** that causes incompatible static vs app shells — replaces the previous static `params`-only detection with a mutable `workUnitStore.hasIncompatibleShellContent` field that flips to `true` if certain runtime APIs (`navigation()` + `prefetch()` — upcoming — plus the existing static `params` promise) are invoked on a route. **Detailed deep dive below.**

### PR #37258 — Flight transfer-key-validation-of-lazy-nodes-when-unwrapped

**The bug:** When Flight outlines a value into its own row, the **client wraps the reference in a lazy node at the reference site** while the value itself is created later, when the target row is initialized. The JSX runtime can therefore only mark the **lazy node** as a validated static child — and that mark never reached the element. **Whether the warning appeared depended on whether a preceding prop was large enough to push the element past the outlining threshold**, so it was non-deterministic from a developer's perspective.

**The fix:** The transfer now happens when the **lazy node is unwrapped**, which is the first point at which both the mark and the value exist. Unwrapping is also the only way any consumer reaches the value, so **a single transfer covers all of them**, and a value that is itself a lazy node forwards the validation one step further (the nested case shows up when an outlined row contains an element that is blocked on debug info). **This subsumes the transfer that `initializeElement` performed for blocked elements** — those are read through the same lazy node, so that special case is gone. **Because the lazy nodes created for chunks no longer share a single `_init` function, `getTaskName` can no longer use it to recognize them** — it checks that the payload is a chunk instead, which describes the same set of lazy nodes without depending on which function unwraps them.

**Why fix it in Flight instead of Fiber (the alternative #37246 proposed):** Fiber-side fix would only cover the reconciler's own `warnOnInvalidKey`. Anything else that unwraps a lazy node and then consults `_store.validated` — e.g., `React.Children.map` — would need its own copy of the same logic, as would the transfer that Flight already performed in `initializeElement`. It would also leave the **nested lazy node case** warning. Since **Flight is what turns a plain element into a lazy node to begin with, it should also be what hides that again.**

**Closes:** #37240 (the false-positive missing-key warning) + #37246 (the Fiber-side fix proposal that was rejected in favor of this Flight-side fix).

**Practical impact for production:**

- **All Server Component + Server Action users on `react@>=19.3.0-canary-807d21fd-20260810`**: the false-positive warning no longer fires on Flight-outlined elements. **No code change required** — the fix is purely internal to Flight's outlining + element validation. If you previously added `key={...}` props as a workaround, you can leave them in place (they're harmless); you don't need to remove them either.
- **All users on pre-`807d21fd` canary** (or `react@latest` 19.2.8 STABLE): if you've seen the "Each child in a list should have a unique 'key' prop" warning on a Server Component output that *clearly has* a `key` prop, and the warning only fires for some elements but not others (typically for elements whose preceding sibling pushes them past the outlining threshold), **bump to `react@>=19.3.0-canary-807d21fd-20260810`** to get the fix.
- **All framework authors wrapping React.Children.map or unwrapping lazy nodes manually**: the fix is internal to Flight's unwrap path; you don't need to change your code.

**Audit recipe:**

```bash
# 1. Confirm the canary cut includes the PR #37258 fix:
npm view react dist-tags.canary
# → should show: 19.3.0-canary-807d21fd-20260810 (or later)

# 2. Render a Server Component that uses dynamic import of a barrel that
# returns a JSX element. With a pre-fix canary, you might see a false-positive
# missing-key warning depending on prop sizes. Post-fix: no warning.

# 3. For framework authors: instrument React.Children.map to log when _store.validated
# is consulted; pre-fix you had to re-implement the transfer; post-fix Flight does
# the transfer at the unwrap site.
```

### PR #97040 — [CC] Track APIs that cause incompatible static/app shells

**Context:** Certain APIs resolve in different stages depending on whether they're in a **static prerender** or a **runtime prerender**:
- `static params` (already exists)
- `navigation()` and `prefetch()` (upcoming)

When a page uses one of these, **separate renders are required for Static and Instant validation.** Previously, static `params` was the only instance of this, and the team could detect those for a route statically. However, **there is no way to statically know if the upcoming `navigation()` or `prefetch()` are going to be called on a given route**, so the approach has to switch to **dynamically tracking API usage** instead.

**The fix:** A **mutable field `workUnitStore.hasIncompatibleShellContent`** (per the PR body), which starts out as `false` and may get set to `true` if one of the APIs is used. Conceptually, the field works **in tandem with `workUnitStore.needsAppShell`**¹ — `needsAppShell` controls **when** things resolve, and `hasIncompatibleShellContent` tracks whether the result would've been equivalent if `needsAppShell` was set to the opposite value (i.e., **are the static and runtime shells equivalent?**).

¹ `needsSessionShell` was renamed to `needsAppShell`, because "session shell" is not really a term being used anywhere else right now — it's basically a leftover.

**This PR moves static `params` to use this new method.** The resulting `params` promise is instrumented, and `hasIncompatibleShellContent` is set when it's `then()`ed, which seems close enough to tracking use.

**No new tests are added**, because the existing tests that check if unguarded `generateStaticParams` trigger instant validation errors in `partialPrefetching` already exercise this codepath.

**Practical impact for production:**

- **Anyone using `cacheComponents: true` + `partialPrefetching: true` + unguarded `generateStaticParams`** — the new tracking field `workUnitStore.hasIncompatibleShellContent` ensures your route is correctly flagged as needing **separate Static vs Instant validation** when the `params` promise is consumed (regardless of whether you're on the static-prerender path or the runtime-prerender path). Behavior change is internal; no public API impact.
- **Anyone using `navigation()` or `prefetch()` (forward-looking for the Aug-Sep window)** — once those APIs ship, they'll also flip `hasIncompatibleShellContent` to `true` automatically. The static detection that worked for `params` doesn't apply to those (because the framework can't statically know whether a route will call them), so the dynamic tracking mechanism is what makes the multi-shell validation work.
- **Audit recipe:**

```bash
# 1. Confirm the canary cut includes the PR #97040 fix:
npm view next dist-tags.canary
# → should show: 16.3.1-canary.11 or later (when it npm-publishes ~2026-08-11)

# 2. For cacheComponents:true projects with unguarded generateStaticParams,
# verify the new tracking field is consulted by adding a console.log:
grep -n 'hasIncompatibleShellContent' node_modules/next/dist/server/...
# Post-fix: the field appears in 7 source files per PR #97040's 7-file diff

# 3. After upgrade, run your existing partialPrefetching tests to confirm
# the unguarded generateStaticParams instant-validation behavior is unchanged.
```

### Canary-branch state observation

At this cron's check, the Next.js canary-branch is **11 commits ahead of `v16.3.1-canary.10`** (verified via `GET /repos/vercel/next.js/compare/v16.3.1-canary.10...canary` returning `ahead_by: 11`):

| Commit | Merged | Author | Description | Material? |
|---|---|---|---|---|
| `da8fc4fe` | 2026-08-10T06:58:33Z | Tobias Koppers | [fix(turbopack): point at the glob that matched a file with no module type](https://github.com/vercel/next.js/pull/96561) | low |
| `ea05267d` | 2026-08-10T08:23:13Z | Niklas Mischkulnig | [Remove unused htmlLimitedBots from renderOpts](https://github.com/vercel/next.js/pull/96701) | low |
| `5f23fb6a` | 2026-08-10T08:43:36Z | Niklas Mischkulnig | test: cleanup Turbopack snapshot config (PR #97013) | low (test-only) |
| `2966db44` | 2026-08-10T10:25:22Z | David Alexandru Ilie | [Trace development route preparation](https://github.com/vercel/next.js/pull/96453) | low (observability) |
| `17f6f135` | 2026-08-10T11:26:51Z | Sebastian Silbermann | [[fragment-scroll] Rename `ScrollAndFocusHandler` to `ScrollHandler`](https://github.com/vercel/next.js/pull/96828) | low (rename) |
| `a7bd5531` | 2026-08-10T11:28:55Z | Hendrik Liebau | [Revert "[turbopack] Follow re-exports for side-effect free async modules" (#97009)](https://github.com/vercel/next.js/pull/97009) | **MATERIAL** (revert; covered in v1.5.45 performance.md / patterns.md) |
| `259abbba` | 2026-08-10T11:28:55Z | Hendrik Liebau | [Revert "[turbopack] Enable CJS tree shaking by default (#96779)" (#97018)](https://github.com/vercel/next.js/pull/97018) | **MATERIAL** (revert; covered in v1.5.45 performance.md / patterns.md) |
| `f0166228` | 2026-08-10T15:15:10Z | Sebastian Silbermann | [Prefix `'use cache'` debug logs with the full directive](https://github.com/vercel/next.js/pull/97037) | **MEDIUM** (Cache Components debug-log clarity) |
| `ec8b5435` | 2026-08-10T15:15:56Z | David Alexandru Ilie | [Trace development route compilation](https://github.com/vercel/next.js/pull/96454) | low (observability; companion to #96453) |
| `1722e45c` | 2026-08-10T15:15:57Z | David Alexandru Ilie | [Fix client component loading span timing](https://github.com/vercel/next.js/pull/96455) | low (observability/tracing fix) |
| `90747a9e` | 2026-08-10T16:29:49Z | Janka Uryga | **[CC] Track APIs that cause incompatible static/app shells (PR #97040)** | **MATERIAL** (Cache Components static-vs-runtime validation) |

The 4 NEW commits since the v1.5.45 cycle's snapshot (the v1.5.45 cycle documented the first 7; the 4 new ones are bolded):
- **PR #97037** (`f0166228`) — `'use cache'` debug log prefix (1 file / +104/-33; affects anyone running `NEXT_PRIVATE_DEBUG_CACHE=1` to debug Cache Components). With `NEXT_PRIVATE_DEBUG_CACHE` enabled, every log line from the "use cache" wrapper used the same generic `use-cache:` prefix, so output from `"use cache: remote"` or `"use cache: private"` functions was indistinguishable from plain `"use cache"`. Each wrapper line is now prefixed with the full quoted directive derived from the handler kind. **Material for anyone debugging Cache Components**; low for typical users.
- **PR #96454** (`ec8b5435`) — Trace dev route compilation (companion to PR #96453).
- **PR #96455** (`1722e45c`) — Fix client component loading span timing (observability/tracing fix).
- **PR #97040** (`90747a9e`) — [CC] Track APIs that cause incompatible static/app shells — **the Cache Components material change documented above**.

**Forward-looking:** canary.11 npm-publish window expected ~2026-08-11T07:41Z ± a few hours on the 24h cadence (closes the v1.5.44 anomaly-prediction cycle; the v1.5.45 cycle's "canary.11 npm-publish expected ~2026-08-11T07:41Z" prediction is on track).

### Sources

- [React PR #37258 — [Flight] Transfer key validation of lazy nodes when they are unwrapped](https://github.com/facebook/react/pull/37258) — by Hendrik Liebau (unstubbable), merged 2026-08-10T14:18:47Z, 2 files / +326/-16, base `main`. **SHIPPED in `react@19.3.0-canary-807d21fd-20260810`** (npm-published 2026-08-10). Closes #37240 + #37246.
- [React Issue #37240 — False-positive missing-key warning on Flight-outlined values](https://github.com/facebook/react/issues/37240) — the bug report PR #37258 closes.
- [React Issue #37246 — Fiber-side fix proposal (rejected in favor of the Flight-side fix in PR #37258)](https://github.com/facebook/react/issues/37246) — explains why fixing in Flight instead of Fiber was the right call.
- [npm: `react@19.3.0-canary-807d21fd-20260810`](https://www.npmjs.com/package/react/v/19.3.0-canary-807d21fd-20260810) — the new React canary cut that ships PR #37258.
- [Cross-reference: components.md `## React 19.3.0-canary-807d21fd-20260810 SHIPPED (August 10, 2026) — ec61f187 → 807d21fd (PR #37241 Lazy Reasons to browser() + PR #37258 Flight Key Validation of Lazy Nodes + the v1.5.37-Forward-Looking PR #37193)`](https://github.com/clawvpsai/frontend-skill/blob/main/components.md#react-1930-canary-807d21fd-20260810-shipped-august-10-2026--ec61f187--807d21fd-pr-37241-lazy-reasons-to-browser--pr-37258-flight-key-validation-of-lazy-nodes--the-v1537-forward-looking-pr-37193) — the components.md coverage of the SHIP event + PR #37258 + PR #37241 + the now-SHIPPED v1.5.37 PR #37193.
- [Next.js PR #97040 — [CC] Track APIs that cause incompatible static/app shells](https://github.com/vercel/next.js/pull/97040) — by Janka Uryga (lubieowoce), merged 2026-08-10T16:29:50Z, 7 files / +91/-47, base `canary`. **Forward-looking for `next@16.3.1-canary.11`** (npm-publish expected ~2026-08-11T07:41Z).
- [Next.js PR #97037 — Prefix `'use cache'` debug logs with the full directive](https://github.com/vercel/next.js/pull/97037) — by Sebastian Silbermann, merged 2026-08-10T15:15:11Z, 1 file / +104/-33, base `canary`. Cache Components debug-log clarity fix.
- [Next.js PR #96453 — Trace development route preparation](https://github.com/vercel/next.js/pull/96453) + [PR #96454](https://github.com/vercel/next.js/pull/96454) + [PR #96455](https://github.com/vercel/next.js/pull/96455) — observability/tracing trio.
- [Next.js canary-branch compare `v16.3.1-canary.10...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.10...canary) — verified at 2026-08-10T18:02Z; 11 commits ahead.
- [Next.js `cacheComponents` config reference](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents) — the Cache Components config that PR #97040's tracking field interacts with.
- [Next.js `generateStaticParams` API docs](https://nextjs.org/docs/app/api-reference/functions/generate-static-params) — the API that the static `params` tracking now uses via the new `hasIncompatibleShellContent` field.
- [Next.js `partialPrefetching` config reference](https://nextjs.org/docs/app/api-reference/config/next-config-js/partialPrefetching) — the validation mode that exercises the new tracking field.
- [Cross-reference: v1.5.45 components.md `## React 19.3.0-canary-ec61f187-20260806 SHIPPED` section](https://github.com/clawvpsai/frontend-skill/blob/main/components.md) — the prior React canary SHIP event; v1.5.46 ships the next canary cut.
- [Cross-reference: v1.5.45 performance.md `## next@16.3.1-canary.10 SHIPPED` section](https://github.com/clawvpsai/frontend-skill/blob/main/performance.md) — the prior Next.js canary SHIP event; v1.5.46 documents the canary-branch state ahead of canary.11.

## Cache Components — 3-PR Legacy PPR Refactor (PR #96753 / #96827 / #96868 — canary.12-ahead) + Turbopack CJS Scope-Hoisting Flag (PR #95826) + `experimental.ppr` Now Hard-Deprecated + Cache Components as Sole Internal PPR Signal (August 11, 2026)

**The v1.5.45/v1.5.46 forward-looking "3-PR legacy PPR refactor" prediction came true.** All three PRs merged 2026-08-11T01:39:14Z in a coordinated push: **(1) PR #96753** (`d1123c9`) — `Make legacy PPR paths explicit` — renames the remaining legacy PPR render capability / prerender store / component / tracking helper from their old generic names to their `Legacy*` equivalents (e.g. `ppr_renderer_capability` → `legacy_ppr_renderer_capability`, `PprTracker` → `LegacyPprTracker`, `PprComponent` → `LegacyPprComponent`, etc.). The names are now uniform across the four legacy identifiers. **(2) PR #96827** (`d18e789`) — `Use Cache Components as the internal PPR signal` — migrates the runtime so enabling Cache Components is the sole path for partial prerendering. Before this PR, both `experimental.ppr: true` and `experimental.cacheComponents: true` could independently enable PPR; after this PR, only `cacheComponents` does. `experimental.ppr` remains a hard error at config-eval time (it has been since the deprecation PR landed in an earlier cycle). **(3) PR #96868** (`268ae98`) — `Remove legacy PPR code paths` — deletes the now-renamed legacy PPR code paths entirely. Only the Cache-Components-driven path remains. **Plus PR #95826** (`5327653`, merged 2026-08-11T07:06:04Z) — `[turbopack] Do the CJS analysis needed for scope hoisting` — adds the new `turbopackCjsScopeHoisting` config flag plumbed through `next.config.ts` + the CJS-analysis machinery in `turbopack/crates/turbopack-ecmascript/src/analyzer/graph/visitor.rs`. **All four PRs land in `next@16.3.1-canary.12`** (tagged 2026-08-11T17:23:13Z; npm `dist-tag.canary` still resolves to `16.3.1-canary.11` at this cron's check — canary.12 npm-publish expected ~2026-08-12T07:23Z ± a few hours on the 24h cadence). Detailed routing-lens coverage in `routing.md`.

### Per-PR Deep Dive

**PR #96753 — Make legacy PPR paths explicit** (Janka Uryga / lubieowoce, merged 2026-08-11T01:39:14Z, base `canary`): The PR renames the four remaining legacy PPR symbols from their old generic names to their `Legacy*` equivalents:

| Old name | New name |
|---|---|
| `ppr_renderer_capability` (or similar) | `legacy_ppr_renderer_capability` |
| `ppr_prerender_store` | `legacy_ppr_prerender_store` |
| `PprTracker` | `LegacyPprTracker` |
| `PprComponent` (or `ppr_component`) | `LegacyPprComponent` |

**The why** — making the names explicit lets the next two PRs in the coordinated set (PR #96827 + PR #96868) delete the legacy code without ambiguity. Anything else named "ppr" in the codebase is now Cache Components (the non-legacy code path). **Practical impact** — zero behavior change. If you import any of these by name from `next/dist/server/...`, the import paths are unchanged but the exported identifiers are renamed. Audit recipe: `rg -n "ppr_renderer_capability|PprTracker|PprComponent" packages/next/src/server/` (will return 0 lines for canary.12+ users; renames are in place).

**PR #96827 — Use Cache Components as the internal PPR signal** (Janka Uryga / lubieowoce, merged 2026-08-11T01:39:14Z, base `canary`): This PR makes Cache Components the **sole** internal signal for "this is a PPR render". Before this PR, both `experimental.ppr: true` AND `experimental.cacheComponents: true` could independently enable partial prerendering. After this PR, only `cacheComponents` does. `experimental.ppr` was already hard-deprecated (config-eval error if truthy); this PR makes the runtime truth match the config truth. **The change**:

```ts
// next.config.ts — pre-canary.12 (still works for now):
export default {
  experimental: {
    ppr: true,  // ❌ deprecated; will error at config-eval time
    cacheComponents: true,  // ✅ the new public API
  },
};

// next.config.ts — canary.12+:
export default {
  experimental: {
    // Don't add `ppr: true` here — it'll error.
    cacheComponents: true,  // ✅ the only way to enable PPR
  },
};
```

**Practical impact**:

| Deployment profile | Pre-canary.12 | Post-canary.12 |
|---|---|---|
| **Using `experimental.ppr: true`** | Config-eval error (already deprecated) | Same — error continues to fire |
| **Using `experimental.cacheComponents: true`** | Enables PPR via Cache Components | Enables PPR via Cache Components (unchanged externally; the internal signal source moved) |
| **Using neither** | No PPR | No PPR |
| **Framework authors wrapping Next.js internals** | Could rely on either config option | Must use Cache Components — `ppr` is now an error |

**Audit recipe** — `rg -n "experimental.ppr|experimental:.*ppr" next.config.ts next.config.js` (should return 0 lines for canary.12+ users). `rg -n "ppr_renderer_capability|PprTracker|PprComponent" packages/` (will find Cache-Components-only references after canary.12).

**PR #96868 — Remove legacy PPR code paths** (Janka Uryga / lubieowoce, merged 2026-08-11T01:39:14Z, base `canary`): Pure cleanup. Deletes the now-renamed legacy PPR code paths so only the Cache-Components-driven path remains. **Zero behavior change** for users — the legacy paths were already inaccessible (PR #96827 made them so). For framework authors wrapping Next.js internals: anything you imported from the `Legacy*` paths is now gone. Audit recipe: `rg -n "legacy_ppr|LegacyPpr" packages/next/src/server/` (returns 0 lines; the paths are deleted in canary.12+).

**PR #95826 — [turbopack] Do the CJS analysis needed for scope hoisting** (Tobias Koppers / sokra, merged 2026-08-11T07:06:04Z, base `canary`): Adds the new `turbopackCjsScopeHoisting` config flag plumbed through `next.config.ts` + the CJS-analysis machinery in `turbopack/crates/turbopack-ecmascript/src/analyzer/graph/visitor.rs`. The PR body explains: this PR does two things: adds a flag `turbopackCjsScopeHoisting` and plumbs it through and adds the extra stuff we need in `turbopack/crates/turbopack-ecmascript/src/analyzer/graph/visitor.rs`. **Practical impact**:

```ts
// next.config.ts — canary.12+ opt-in to CJS scope hoisting
export default {
  experimental: {
    turbopackCjsScopeHoisting: true,  // NEW in canary.12; default OFF
  },
};
```

**Zero behavior change if not opted-in.** If you opt-in, you get additional Turbopack tree-shaking on CJS modules (compile-time only; bundle-size savings). Audit recipe: `rg -n "turbopackCjsScopeHoisting" next.config.ts` (verify your setting; default is `false`).

### Practical Impact Summary

| User type | Pre-canary.12 | Post-canary.12 |
|---|---|---|
| **Anyone using `experimental.ppr: true`** | Config-eval error (already deprecated) | Same — error continues to fire |
| **Anyone using `experimental.cacheComponents: true`** | PPR enabled via Cache Components | PPR enabled via Cache Components (no behavior change) |
| **Framework authors wrapping Next.js internals** | Could rely on either config option | Must use Cache Components — `ppr` is now an error |
| **Anyone wanting Turbopack CJS tree-shaking** | Only via `turbopackCjsScopeHoisting` opt-in (PR #95826) | Same — opt-in flag for the CJS scope-hoisting analysis |
| **Anyone using `@mixmark-io/domino` or other self-referential CJS modules** | PR #97018's CJS tree-shaking revert means no tree-shaking | Can opt-in to CJS scope hoisting (PR #95826) + bail on self-referential patterns (PR #97130) |

### Audit Recipe (3 Steps)

1. **`rg -n "experimental.ppr|experimental:.*ppr" next.config.ts next.config.js`** — should return 0 lines. If you see any matches, the build will error at config-eval time on canary.12+. Remove the line.
2. **`rg -n "cacheComponents" next.config.ts`** — should return 1 line with `true`. This is the new public API for enabling PPR.
3. **`npm ls next`** — verify you're on `next@>=16.3.1-canary.12` (when it npm-publishes ~2026-08-12T07:23Z ± a few hours). Until then, you're still on canary.11 with the legacy PPR paths still in place (they're benign because PR #96827 hasn't yet enforced Cache-Components-only at the runtime signal level).

### Forward-Looking

- **Next.js `v16.3.2-canary` (forward-looking, late August 2026)** — once the 3-PR legacy PPR refactor lands in canary.12 npm-publish, the next canary cuts will likely start exploring `experimental.ppr` config-eval-error messages with a migration hint to `cacheComponents`. Plan to migrate before 16.4 STABLE ships (likely Oct 2026).
- **`experimental.cacheComponents` documentation** — Next.js docs will be updated to clarify that enabling Cache Components is the only way to enable PPR in 16.4+. Watch for the docs refresh around the 16.4 alpha window (late September 2026).
- **`generateStaticParams` + the new `hasIncompatibleShellContent` field (PR #97040)** — already shipped in canary.11. Combined with the 3-PR legacy PPR refactor in canary.12, the `generateStaticParams` instrumentation is now the primary mechanism for static-vs-runtime shell tracking (the field flips when `params.then()` is called inside the route's render).

### Sources

- [PR #96753 — Make legacy PPR paths explicit](https://github.com/vercel/next.js/pull/96753) — by Janka Uryga (lubieowoce), merged 2026-08-11T01:39:14Z, base `canary`. The naming pass.
- [PR #96827 — Use Cache Components as the internal PPR signal](https://github.com/vercel/next.js/pull/96827) — by Janka Uryga (lubieowoce), merged 2026-08-11T01:39:14Z, base `canary`. The substantive change.
- [PR #96868 — Remove legacy PPR code paths](https://github.com/vercel/next.js/pull/96868) — by Janka Uryga (lubieowoce), merged 2026-08-11T01:39:14Z, base `canary`. The cleanup.
- [PR #95826 — [turbopack] Do the CJS analysis needed for scope hoisting](https://github.com/vercel/next.js/pull/95826) — by Tobias Koppers (sokra), merged 2026-08-11T07:06:04Z, base `canary`. The `turbopackCjsScopeHoisting` flag.
- [Next.js `cacheComponents` config reference](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents) — the new public API for enabling PPR.
- [Next.js `experimental.ppr` config reference](https://nextjs.org/docs/app/api-reference/config/next-config-js/ppr) — the deprecated API; will error at config-eval time on canary.12+.
- [Next.js canary-branch compare `v16.3.1-canary.11...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.11...canary) — 15 commits ahead as of 2026-08-11T18:02Z.
- [Next.js `v16.3.1-canary.12` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.12) — published 2026-08-11T17:23:13Z; npm-publish expected ~2026-08-12T07:23Z ± a few hours.
- [Next.js `v16.3.1-canary.11` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.11) — npm-published 2026-08-11T00:03:41Z; latest npm-available canary.
- Cross-reference: `routing.md` → `## 16.3.1-canary.12-ahead — Fix Optimistic Routing Bugs (PR #97128) + 3-PR Legacy PPR Refactor (PR #96753 / #96827 / #96868) + Turbopack CJS Scope-Hoisting Flag (PR #95826)` for the routing lens on the same PRs.
- Cross-reference: `components.md` → `## React 19.3.0-canary-bfb7a768-20260811 SHIPPED (August 11, 2026) — 807d21fd → bfb7a768 (PR #34983 Metadata Hoisting in Hidden Activity Trees + PR #37171 Drop Empty Fragment scrollIntoView No-Op Warning)` for the React canary SHIP event.
- Cross-reference: v1.5.46 components.md `## React 19.3.0-canary-807d21fd-20260810 SHIPPED` section for the prior React canary SHIP event.


## `headers()` Restored to Live View of Incoming Request (PR #97166) + Turbopack `crossOrigin` Manifest Fix (PR #97164) — SHIPPED in 16.3.1-canary.14 (npm-published 2026-08-12T13:25:30Z, August 12, 2026)

The single most material Server-Components-relevant Next.js change in the 5h49min window since v1.5.50 — both fixes landed in canary-branch ahead of `16.3.1-canary.13` and **SHIPPED at 2026-08-12T13:25:30Z** â ~4h38min before this cron's 18:03Z start. **PR #97166** (Hendrik Liebau / unstubbable, merged 2026-08-12T11:36:12Z — 29min before this cron) restores the live `headers()` view; **PR #97164** (Tim Neutkens, merged 2026-08-12T07:19:23Z) closes issue #96831 (the Turbopack `crossOrigin: "none"` cross-origin CDN regression that v1.5.33 documented as one of the 3 NEW material open issues on `next@16.3.0` STABLE). **canary-branch is now 4 commits ahead of `16.3.1-canary.14`** (was 11 in v1.5.51 at 12:03Z Aug 12); 4 NEW commits since canary.14 tagged at 13:25:30Z, 2 MATERIAL (PR #97166 + PR #97164) + 5 docs/test/CI (PR #93529, PR #97191, PR #97187, PR #97163, PR #97190).

### Why PR #97166 matters for Server Components — Live `headers()` View

`headers()` in Server Components is the canonical mechanism for reading the request's headers in RSCs. The bug introduced in `next@16.3.0` STABLE (and persisting through `canary.0`–`canary.13`) broke this contract: **a Proxy that mutated `request.headers.set(...)` was invisible to subsequent `headers()` reads**, while the same mutation was visible via direct `request.headers.get(...)` access. This meant any Server Component reading `headers()` after a Proxy mutation saw a **stale pre-mutation view** — breaking trace-id propagation, tenant-id injection, feature-flag injection, A/B-test cookie mirroring, and any other "Proxy injects, RSC reads" pattern. **Server-Components-lens practical impact**:

| Server Component pattern | Symptom pre-#97166 | Post-#97166 |
|---|---|---|
| **`await headers()` in a Server Component to read `x-trace-id` set by Proxy** | Returns the pre-mutation value; tracing breaks end-to-end | Live view returns the post-mutation value; tracing works |
| **`await headers()` in a Server Component to read `x-tenant-id` set by Proxy** | Returns the pre-mutation value; tenant-aware RSCs render the wrong tenant's data | Live view; correct tenant data |
| **`await headers()` in a Server Component to read `accept-language` set by Proxy** (e.g., for i18n) | Returns the pre-mutation value; i18n RSC falls back to default | Live view; i18n RSC reads the correct locale |
| **`await headers()` in a Server Action** (the `server-action-headers-redirect` e2e that #95116 was protecting) | Worked (PR #95116's copy protected this specific path) | Still works |
| **`await cookies()` in a Server Component** | Snapshot semantics preserved (predates 16.3) | **Unaffected** — `RequestCookies` parses on construct, left alone |
| **RSC + Cache Components + `await headers()` for cache-key derivation** | Cache key derived from stale headers; stale cache hits across requests | Live headers; correct cache key derivation per request |
| **RSC + Proxy + `request.headers.forEach(...)` for custom logging** | The `forEach` callback could escape the seal and write to the underlying headers — defeating the security model | `forEach` now passes the sealed proxy; can't mutate |
| **No Proxy, just `await headers()` in RSC for `accept-language` / `user-agent`** | Worked (no mutation involved) | Works the same |
| **Client Components reading via `useHeaders()` or `headers()` in route handlers** | Worked | Works the same |

**Why this matters for Cache Components** — Cache Components derives cache keys from a stable view of the request. If `headers()` returns stale data, the cache key may be derived from the pre-mutation view, which means **stale cache hits across requests**: two different tenants may hit the same cached RSC because the cache key was derived from headers that hadn't yet been mutated by the Proxy. PR #97166 closes this gap.

**Audit recipe (4 steps)**:
1. **`rg -n "await headers\(\)" app/ src/`** — find every Server Component or Server Action that reads headers. If you have a Proxy that mutates `request.headers`, these are the sites that need the fix.
2. **`rg -n "request\.headers\.(set|delete|append)" proxy.ts middleware.ts app/proxy.ts`** — find the Proxy mutation sites. Cross-reference with step 1 to identify the Server Components affected.
3. **`rg -n "forEach.*headers|headers.*forEach" app/ src/`** — find any `forEach` over headers. Review to ensure no callback was relying on the mutability (security-tightening side fix in PR #97166).
4. **`npm ls next`** — you're on `next@>=16.3.1-canary.14` (now that it has shipped at 2026-08-12T13:25:30Z). Until then, work around by reading via `request.headers.get(name)` directly in the Proxy, or by stashing the header on a custom `request.cookies` set before the mutation.

**Workaround (for versions pre-canary.14)** — canary.14 shipped at 2026-08-12T13:25:30Z; bump to `next@>=16.3.1-canary.14` to fix:

```tsx
// app/api/route.ts — Server Component reading x-tenant-id set by Proxy
// WORKAROUND (works on all versions, but no-op after canary.14):
import { headers } from 'next/headers';
import { NextResponse, type NextRequest } from 'next/server';

export async function proxy(request: NextRequest) {
  request.headers.set('x-tenant-id', await resolveTenantFromAuth(request));
  return NextResponse.next();
}

export default async function TenantPage() {
  // Pre-#97166: returns stale pre-mutation x-tenant-id
  // Post-#97166: returns the live post-mutation value
  const headersList = await headers();
  const tenantId = headersList.get('x-tenant-id');
  // ...render with tenantId
}
```

The proxy-headers-live-view test added by PR #97166 (`test/e2e/app-dir/proxy-headers-live-view/`) is the canonical verification target.

### Why PR #97164 matters for Server Components — Turbopack `crossOrigin` Manifest Fix

PR #97164 closes issue **#96831** (documented in v1.5.33 `deployment.md` as one of the 3 NEW material open issues on `next@16.3.0` STABLE). The bug: Turbopack's client reference manifest serialized the `CrossOrigin` enum's `None` variant as the string `"none"`, so React saw a present string and emitted `crossorigin=""` on preinited `<script>` tags even when `crossOrigin` was unset. **Server-Components-lens impact**: the client reference manifest is the bridge between Server Components and their Client Component counterparts in the browser. Every Server Component that imports a Client Component gets a manifest entry pointing at the chunk(s) that contain the client reference. If those chunks carry `crossorigin=""` on cross-origin `assetPrefix` CDN deployments, the chunks fail to load (CORS-mode loads poisoned by no-cors cache entries). **The fix is unconditional** (no opt-in flag needed) — skip serializing `CrossOrigin::None` in the Turbopack client reference manifest, so an unset option remains absent while preserving explicit anonymous and credentialed modes.

**Audit recipe (3 steps)**:
1. **`curl -sL https://your-app.example.com | grep -i 'crossorigin'`** — confirm `crossorigin` attribute is absent on script tags post-#97164.
2. **`rg -n "assetPrefix" next.config.ts`** — find any project with cross-origin CDN assetPrefix on Turbopack + App Router + `next@16.3.0` STABLE.
3. **`npm ls next`** — bump to `next@>=16.3.1-canary.14` (shipped at 2026-08-12T13:25:30Z).

### Practical Impact Summary

| User type | Pre-canary.14 | Post-canary.14 |
|---|---|---|
| **Anyone using `headers()` in RSC after a Proxy mutation** | Stale pre-mutation view | Live view; mutations observed |
| **Anyone using `headers()` in Cache Components for cache-key derivation** | Stale cache keys; stale hits | Live cache keys; correct hits |
| **Anyone using `headers().forEach(...)` with a mutating callback** | Callback could escape the seal (security gap) | Callback can't mutate; seal preserved |
| **Anyone using `cookies()` in RSC** | Snapshot (predates 16.3, left alone) | Same |
| **Anyone with Turbopack + cross-origin `assetPrefix` CDN + App Router** | `crossorigin=""` on preinited chunks; CORS-mode loads poisoned | Unset remains absent; CORS-mode loads succeed |
| **Anyone using `crossOrigin: 'anonymous'` / `'use-credentials'` explicitly** | Worked | Still works |
| **Anyone with same-origin or no `assetPrefix`** | Not affected | Not affected |
| **Anyone on Webpack** | Not affected (Webpack was already omitting `undefined` correctly) | Not affected |
| **Framework authors wrapping Next.js request-store internals** | Could rely on either live or snapshot semantics | Must handle live semantics — code that assumed snapshot needs review |

### Sources

- [PR #97166 — Restore the live `headers()` view of the incoming request](https://github.com/vercel/next.js/pull/97166) — by Hendrik Liebau (unstubbable), merged 2026-08-12T11:36:12Z, 8 files / +382/-23, base `canary`. **THE HEADLINE** for the Server Components lens.
- [PR #97164 — `[turbopack]` Fix unset `crossOrigin` in Turbopack manifests](https://github.com/vercel/next.js/pull/97164) — by Tim Neutkens, merged 2026-08-12T07:19:23Z, 7 files / +164/-31, base `canary`. Closes issue #96831.
- [Issue #96831 — Turbopack `crossOrigin: "none"` string serialization breaks cross-origin assetPrefix CDNs](https://github.com/vercel/next.js/issues/96831) — closed by PR #97164.
- [Issue #97145 — Fix `headers()` stale snapshot after `NextRequest` mutation in Proxy (the @tachsin fork PR)](https://github.com/vercel/next.js/pull/97145) — the original @tachsin fork PR; recreated by Hendrik Liebau as PR #97166.
- [Next.js `HeadersAdapter.seal` source](https://github.com/vercel/next.js/blob/canary/packages/next/src/server/web/spec-extension/adapters/headers.ts) — the function PR #97166 restructures to hide-on-read instead of copy-and-delete.
- [Next.js `request-store.ts`](https://github.com/vercel/next.js/blob/canary/packages/next/src/server/async-storage/request-store.ts) — the Server Component request-scoped storage where `headers()` lives.
- [Next.js `client_reference_manifest.rs` source](https://github.com/vercel/next.js/blob/canary/crates/next-core/src/next_manifests/client_reference_manifest.rs) — the 5-line Turbopack change in PR #97164 that skips serializing `CrossOrigin::None`.
- [Next.js canary-branch compare `v16.3.1-canary.13...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.13...canary) — 11 commits ahead as of 2026-08-12T12:03Z; 7 NEW since v1.5.50.
- [Next.js `proxy-headers-live-view` e2e test](https://github.com/vercel/next.js/tree/canary/test/e2e/app-dir/proxy-headers-live-view) — the canonical verification target added by PR #97166 (proxy.ts + page.tsx + layout.tsx + next.config.js + test).
- [Next.js `app-config-crossorigin` e2e test](https://github.com/vercel/next.js/tree/canary/test/e2e/app-dir/app-config-crossorigin) — the production coverage added by PR #97164 for static prerendering, dynamic rendering, and `output: 'export'`.
- Cross-reference: `routing.md` → `## 16.3.1-canary.13-ahead — Restore the Live headers() View (PR #97166) + Fix Unset crossOrigin in Turbopack Manifests (PR #97164)` for the routing lens on the same PRs.
- Cross-reference: `deployment.md` → `## Next.js 16.3.0 STABLE — 3 NEW Open Issues Affecting Production Deployments Today` for the issue #96831 closure confirmation (already documented in v1.5.33).

## Next.js — `fix(cache-components): decompress postponed resume body before parsing` (PR #95238, August 13, 2026) + 1-commit Redux of the React Vendor Bump (PR #97249) (Server Components Lens)

`next@16.3.1-canary.15` SHIPPED at 2026-08-12T23:26:21Z with 15 commits ahead of canary.14 (documented in v1.5.54). **canary-branch is now 2 NEW commits ahead of canary.15 (verified at 2026-08-13T12:02Z via `GET /repos/vercel/next.js/compare/v16.3.1-canary.15...canary` returning `ahead_by: 2, behind_by: 0`; the v1.5.55 cycle captured the first 2 [PR #97247 + PR #96525] and the canary.15 SHIP event was the cutoff point)**. Wait — that's the v1.5.55 cycle's lens. Let me re-state cleanly: **the only commits ahead of canary.15 in the 6h window since v1.5.55 (06:13Z Aug 13 → 12:02Z Aug 13) are PR #97249** (next-js-bot, merged 2026-08-13T10:19:25Z, base `canary`) — **Upgrade React from `11eddecd-20260805` to `22e4f993-20260811` (PR #97249)** — a routine vendor bump, low material. The headline Server Components / Cache Components material this cycle is **PR #95238** (lazypool, merged 2026-08-13T02:42:51Z, 10 files / +195/-24, base `canary`) — **`fix(cache-components): decompress postponed resume body before parsing`** — a **Cache Components / PPR resume bug fix** that addresses a production-blocker where the resume body arrives gzip-compressed behind Vercel's infrastructure but the `next/dist/server/base-server.js` resume branch reads it as raw UTF-8 without checking `Content-Encoding`. The PR body, verbatim: *"In the PPR (Cache Components) resume flow, `base-server.js` reads the POST request body and decodes it using `body.toString('utf8')` without respecting the `Content-Encoding` header. When the resume body arrives gzip-compressed (which happens behind Vercel's infrastructure), the gzip binary bytes (starting with `1f 8b 08`) are misinterpreted as a UTF-8 string. This causes `parsePostponedState` to fail the `^([0-9]*):` format check, throwing an `Invariant: invalid postponed state` error (E314). The error is caught, degraded to `type:1`, and results in a logged server error with an HTTP 200 fallback — meaning the route cannot resume its prerendered HTML shell. **Note: This is not a 16.3 regression. The resume-body read logic in `base-server.js` is byte-for-byte identical in 16.2.x and 16.3.**"* The fix adds a `decompressBody()` utility in `next/dist/server/lib/postponed-request-body.ts` that handles decompression **after reading the body** and **before calling `toString('utf8')`**: **(1) Explicit `Content-Encoding`** — when the header is present, `decompressBody` dispatches to the matching `zlib` sync method (`inflateSync` for `deflate`, `gunzipSync` for `gzip`, `brotliDecompressSync` for `br`). **(2) Magic header detection (no `Content-Encoding`)** — the PPR resume chain contract does not carry `Content-Encoding`, but Vercel's infrastructure may gzip the body without setting the header. `decompressBody` auto-detects the gzip magic number (`0x1f 0x8b`) when the header is absent. This is safe because a valid postponed state always starts with a length prefix (ASCII digits `0x30`–`0x3a`), so there is no overlap. **(3) Zip bomb protection** — each decompression step is bounded by `experimental.maxPostponedStateSize` (the existing limit), preventing a malicious or corrupted gzip from exhausting memory. The 10 files / +195/-24 diff is concentrated in `next/dist/server/lib/postponed-request-body.ts` (the new `decompressBody` utility + the `readBodyWithSizeLimit` wiring) + `next/dist/server/base-server.js` (the resume branch change to call `decompressBody` before `body.toString('utf8')`) + 8 new tests covering gzip-with-Content-Encoding, gzip-without-Content-Encoding (magic-number detection), deflate, brotli, zip-bomb-attempt, valid-plain-text-passthrough, oversized-body-rejection, and corrupted-payload-rejection. **Will ship in `next@16.3.1-canary.16`** npm-published within 8-12h on the accelerated 24h cadence (canary.15→canary.16 cadence follows the canary.14→canary.15 cadence of 9h59min from v1.5.54). **The 1-commit addendum since v1.5.55 is PR #97249** (the React vendor bump 11eddecd → 22e4f993, low material, no behavior change — just syncs Next.js's vendored React to the latest canary bundle that includes the 8-PR Jack Pope Fragment cleanup PR #37160-#37167).

### Why PR #95238 matters for Server Components — Cache Components resume body decompression

The bug, in depth: Cache Components (the PPR successor, now the default for `cacheComponents: true` in 16.3.x) generates a **postponed state** on the server during prerendering — a serialized blob of the React Server Components render tree that gets stored as the response payload. When the client navigates back to the prerendered route, it POSTs the serialized state back to the server in the resume body so the server can re-hydrate the tree without re-running the full RSC render. The format is `^([0-9]*):<serialized-json>` — a length-prefixed JSON payload. The bug: **Vercel's infrastructure gzips the POST request body** to save wire bytes (a normal optimization for large RSC payloads that can be 50-500KB), but the `next/dist/server/base-server.js` resume branch **does not check `Content-Encoding`** before passing the body to `parsePostponedState`. So the gzip binary bytes (`1f 8b 08...`) get decoded as UTF-8 → garbage string → `parsePostponedState` fails the `^([0-9]*):` regex → E314 `Invariant: invalid postponed state` error. The error is caught and degraded to `type:1`, which means **the route resumes with no prerendered HTML shell** — the user gets a full server-side render instead of the fast Cache Components path. The HTTP 200 is preserved (the request "succeeds" but the response is slow), so the failure is invisible to error monitoring that only watches 5xx. The fix is small but complete: a `decompressBody()` utility that handles the 3 cases (explicit Content-Encoding, magic-number detection, zip-bomb protection). The magic-number detection is the clever part — it auto-detects gzip even when the header is missing, so the fix works **without coordinating infrastructure changes** (the Vercel edge can keep gzipping the body without setting the header). The other clever part: the magic-number detection is safe because a valid postponed state always starts with `0x30`–`0x3a` (ASCII digits for the length prefix), which never overlaps with `0x1f 0x8b` (the gzip magic number). So a valid plain-text postponed state is never mistakenly decompressed.

### Why PR #97249 matters for Server Components — Routine React vendor bump

The diff is 1 file (`package.json` + `pnpm-lock.yaml`) bumping the React vendor entry from `11eddecd-20260805` to `22e4f993-20260811`. The 22e4f993 commit is the last of the 8 Jack Pope Fragment cleanup PRs (PR #37167, merged 2026-08-12T01:46:13Z). Practical impact: Next.js users on `next@>=16.3.1-canary.16` get the 8-PR Fragment cleanup bundle automatically through the React vendor bump. No code changes required. The bump is non-breaking — the API surface is unchanged.

### Audit Recipe (3 Steps)

1. **Check if you're hitting this bug** — `rg -n "Invariant: invalid postponed state\|E314\|invalid postponed state" .next/server-logs/ logs/ ~/.pm2/logs/ 2>/dev/null | head -20`. If you see this error pattern in production logs since the Cache Components rollout, PR #95238 fixes it.
2. **Check your `experimental.maxPostponedStateSize`** — the zip-bomb protection in `decompressBody()` uses the existing limit. **Audit recipe**: `rg -n "maxPostponedStateSize" next.config.*` and confirm the value is reasonable for your app (default 1MB; raise to 5MB for large RSC payloads).
3. **Verify the fix is active** — `npm ls next` should show `>=16.3.1-canary.16` after the npm-publish.

### Practical Impact Summary

| User type | Pre-PR #95238 (canary.15 and earlier) | Post-PR #95238 (canary.16+) |
|---|---|---|
| **Deployments on Vercel with `experimental.cacheComponents: true`** | Route resumes fail with E314; HTTP 200 fallback to full RSC render; users see slow navigation back to prerendered routes | Route resumes successfully; full Cache Components path preserved |
| **Self-hosted deployments with gzip-on-the-wire (Cloudflare, custom proxies)** | Same E314 failure pattern if the proxy gzips the POST body | Body decompressed correctly; resume works |
| **Deployments where the wire is plain (no compression)** | Works correctly (Content-Encoding absent + no gzip magic) | Works correctly (magic-number detection path skipped) |
| **Apps with `experimental.maxPostponedStateSize` set too low** | E314 with "body too large" error | Same; the limit is still enforced post-decompression |
| **Apps with `experimental.cacheComponents: false`** | Not affected (no resume flow) | Not affected |
| **Framework authors wrapping `next/dist/server/base-server.js`** | Not affected (the bug is in the internal `isAppPPREnabled && minimalMode` branch) | Not affected; the fix is internal to the resume branch |

### Sources

- [PR #95238 — `fix(cache-components): decompress postponed resume body before parsing`](https://github.com/vercel/next.js/pull/95238) — by lazypool, merged 2026-08-13T02:42:51Z, 10 files / +195/-24, base `canary`. The PR body documents the gzip-binary-as-UTF-8 bug + the 3-layer fix (explicit Content-Encoding + magic-number detection + zip bomb protection) + the byte-for-byte identity with 16.2.x.
- [PR #97249 — `Upgrade React from 11eddecd-20260805 to 22e4f993-20260811`](https://github.com/vercel/next.js/pull/97249) — by next-js-bot, merged 2026-08-13T10:19:25Z, base `canary`. Routine vendor bump; the 8 Fragment cleanup PRs (PR #37160-#37167) by Jack Pope are now in Next.js's bundled React.
- [Next.js `base-server.js` source](https://github.com/vercel/next.js/blob/canary/packages/next/src/server/base-server.ts) — the `isAppPPREnabled && minimalMode` resume branch that PR #95238 modifies.
- [Next.js `postponed-request-body.ts` source](https://github.com/vercel/next.js/blob/canary/packages/next/src/server/lib/postponed-request-body.ts) — the new `decompressBody()` utility lives here.
- [Next.js `experimental.maxPostponedStateSize` config reference](https://nextjs.org/docs/app/api-reference/config/next-config-js/maxPostponedStateSize) — the existing limit that PR #95238's zip-bomb protection uses.
- [Next.js canary-branch compare `v16.3.1-canary.15...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.15...canary) — 1 commit ahead as of 2026-08-13T12:02Z (PR #97249, the routine vendor bump).
- [Next.js canary-branch compare `v16.3.1-canary.15...main`](https://github.com/vercel/next.js/compare/v16.3.1-canary.15...main) — 1 commit ahead as of 2026-08-13T12:02Z (PR #97249).
- [Next.js `parsePostponedState` source](https://github.com/vercel/next.js/blob/canary/packages/next/src/server/lib/router-utils/postponed-state.ts) — the function that throws E314 `Invariant: invalid postponed state` when the gzip binary is mistakenly decoded as UTF-8.
- [Next.js 16.3.0 Cache Components docs](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents) — the recommended setup for `experimental.cacheComponents: true`.
- [Next.js `experimental.ppr` deprecation note](https://nextjs.org/docs/app/api-reference/config/next-config-js/ppr) — the Cache Components migration path that makes the resume-body decompression fix relevant.
- Cross-reference: `deployment.md` → `## Next.js — next@16.3.1-canary.15-ahead — Resume Data Cache (RDC) Compression Rollout Controls (PR #97247, August 12-13, 2026) + Testmode Passthrough Fetch Infinite-Recursion Fix (PR #96525, August 13, 2026)` (Deployment Impact Lens) for the same canary.15-ahead window from the deployment-angle view.
- Cross-reference: `components.md` → `## React Main Branch — Fragment Deletion Effects for HostText Children (PR #37168, August 13, 2026) — 9th PR in the Jack Pope Fragment Cleanup Series` for the React-side Fragment cleanup PR #37168 that ships via PR #97249's vendor bump.
- Cross-reference: v1.5.54 `## Next.js — canary.15 SHIPPED (August 12, 2026) — PR #95157 Turbopack clusters + PR #97213 HMR fix + PR #97268 mtime fallback + PR #97205 Webpack deferred entry` for the canary.15 SHIP event that PR #95238 is canary-branch-ahead of.

## Next.js 16.3.1-canary.17 SHIPPED (August 14, 2026) — 15 Commits Ahead of canary.16 — HEADLINE PR #97287 NFT Fix + PR #96819 Pages API Runtime + PR #97350 App-Entry Scoping (Server Components Lens — npm-published 2026-08-14T17:20:01Z)

`next@16.3.1-canary.17` SHIPPED at 2026-08-14T17:20:01Z (npm-published 40 minutes before this cron). **Comparing canary.16 → canary.17** (`GET /repos/vercel/next.js/compare/v16.3.1-canary.16...v16.3.1-canary.17` returning `ahead_by: 15, behind_by: 0`): 15 NEW commits landed in this canary cycle. **The v1.5.59 cycle (Aug 14 12:08Z) marked server-components.md as "23h 49min stale WITHOUT new material"** — that was a documentation miss because the canary.17 SHIP event was already 5+ hours from happening but the v1.5.59 cron ran before the npm-publish. **This cycle corrects the miss**. **The current canary-branch is 0 commits ahead of canary.17** (`GET /repos/vercel/next.js/compare/v16.3.1-canary.17...canary` returns `ahead_by: 0, behind_by: 0`) — the team just shipped. **The 15 commits break down as: 4 MATERIAL PRs + 4 trace instrumentation PRs + 3 Turbopack improvements + 4 docs/test/CI/version-tag PRs**. The 4 MATERIAL PRs are the headline for this cycle:

### 4 MATERIAL PRs in canary.17

1. **PR #97287 — `Emit whole-app server NFTs when output: 'standalone' is used with an adapter`** (bryan-ferry, merged 2026-08-14T09:09:02Z, 2 files / +44/-2, base `canary`) — **fixes a production-blocker build crash on 16.3.0 STABLE for any app combining a deployment adapter with `output: 'standalone'`**. The PR body, verbatim: *"Since v16.3.0, `next build` crashes for any app that combines a deployment adapter with `output: 'standalone'`: `Error: ENOENT: no such file or directory, open '<distDir>/next-server.js.nft.json'`"*. The bug: PR #93684 stopped emitting the whole-app server NFTs whenever an adapter is configured (the rationale: "Adapters don't read these files"). But the whole-app NFT has a SECOND, adapter-independent consumer: the `output: 'standalone'` finalizer. `copyTracedFiles` reads `distDir/next-server.js.nft.json` unconditionally whenever standalone output is requested — unlike its sibling per-page trace reads, it has no `.catch`. The build runs adapter finalize and standalone finalize in sequence, so with both configured the Rust side skips the writer while the JS side still runs the reader → raw ENOENT. **This breaks every Vercel deployment of a standalone-configured app on 16.3** (the builder injects its adapter via `NEXT_ADAPTER_PATH` under the `NEXT_ENABLE_ADAPTER` rollout), and — per the reports on #96646 — also self-hosted/AWS `cdk-nextjs` users, for whom **there is no config-level escape**: that adapter force-sets `output: 'standalone'` and the construct requires `.next/standalone`, so both sides of the conflict are mandatory. The combination was never covered by CI: `test/production/next-server-nft/next-server-nft.test.ts` has a `with output:standalone` suite and a `with adapters` suite, but **no combined suite** — the only state where the writer gate and the reader disagree. **The fix**: re-scope PR #93684's gate instead of removing it — skip the whole-app server NFTs only when an adapter is configured **AND** standalone output is not. `is_standalone` was already computed in this exact function two lines below the gate, so the change is a reorder plus one condition. **Additionally**, `copyTracedFiles` gets the same `.catch` + `Log.warn` treatment its sibling per-page reads already have, so any future writer/reader drift degrades into one actionable warning instead of a raw ENOENT. The 2-file +44/-2 diff is concentrated in `packages/next/src/build/utils.ts` (the `is_standalone` gate reorder) + `packages/next/src/build/index.ts` (the `copyTracedFiles` `.catch` addition). **Practical impact**: HIGH — every Next.js 16.3.0 STABLE deployment that uses both an adapter and `output: 'standalone'` is currently broken. **Migration**: bump to `next@>=16.3.1-canary.17` (or the next 16.3.1 STABLE).

2. **PR #96819 — `Fix missing Pages runtime in adapter Pages API outputs`** (timneutkens, merged 2026-08-14T08:38:29Z, 11 files / +191/-5, base `canary`) — **fixes a Pages API route initialization error on 16.3.0 STABLE for apps using deployment adapters**. The PR body, verbatim: *"Pages API functions produced through a build adapter can fail during function initialization, before the customer handler executes: `Cannot find module 'next/dist/compiled/next-server/pages-turbo.runtime.prod.js'`"*. The bug: a Pages API trace naturally contains `pages-api[-turbo].runtime.prod.js` (the runtime for the API route module) but NOT `pages[-turbo].runtime.prod.js` (the Pages renderer reached through the external `next/head` import). The Next.js require hook redirects the shared-runtime import to the Pages vendored context, and `pages/module.compiled.js` selects the bundler-specific production renderer. As a result, a Pages API entry trace does not discover the regular Pages renderer. **The failure is specific to Pages API routes** — a regular Pages SSR entry already references and traces the Pages renderer. App Router entries use different route modules and traces. **The fix**: trace the hidden Pages renderer dependency using the mechanism appropriate to each bundler. For **Turbopack**: add `pages-turbo.runtime.prod.js` as an explicit entry in `Project::pages_traced_modules`. The existing native Turbopack module graph then traces its full runtime closure. This does not run Node File Trace and remains scoped to Pages endpoints. For **Webpack**: run the existing Node File Trace path on `pages.runtime.prod.js` and merge that closure into the Pages shared assets. The 11-file +191/-5 diff is concentrated in `packages/next/src/build/webpack-config.ts` (the Webpack Pages API trace) + `packages/next/src/build/templates/app-middleware.ts` (the Pages API middleware wiring) + 9 new tests covering Pages API + adapter combinations across both bundlers. **Practical impact**: HIGH — every Next.js 16.3.0 STABLE deployment that uses a deployment adapter + Pages API routes is currently broken. **Migration**: bump to `next@>=16.3.1-canary.17`.

3. **PR #97350 — `Scope app-entry export validation to files inside the app directory`** (unstubbable, merged 2026-08-14T09:53:25Z, 47 files / +79/-18, base `canary`) — **fixes a pages-router build regression introduced in 16.3.0 STABLE for pages named `sitemap` or `robots`**. The PR body, verbatim: *"Adopts #97150. Fixes #96859. Since 16.3.0, builds fail for pages-router files named `sitemap` or `robots` that export `getStaticProps`/`getServerSideProps`: `Error: "getStaticProps" is not supported in app/.`"*. The bug: PR #94962 added the metadata conventions (`sitemap`, `robots`, `manifest`, `icon`, …) to the app-entry filename regex in `ReactServerComponentValidator::assert_invalid_api`. The regex only looks at the filename, so a file like `pages/sitemap.js` is now mistaken for an app entry and rejected for exporting `getStaticProps`. **The fix**: a file is only treated as an app entry when it's inside `appDir`, reusing the gate `assert_server_filename` already applies to `error.js`. The pages compilation context has no `appDir`, so pages-router files are never validated as app entries. The 47-file +79/-18 diff is concentrated in `packages/next/src/server/app-render/ReactServerComponentValidator.ts` (the `assert_invalid_api` + `assert_server_filename` gates) + 46 fixture/test moves (`app-dir/` subdirectory relocation + new fixtures + new e2e suite `test/e2e/pages-metadata-filenames`). The new e2e test covers `pages/sitemap.js` with `getStaticProps` + `pages/robots.js` with `getServerSideProps` — it fails without the fix under Turbopack (both build and dev) and passes with it under both bundlers. **Also closes**: #96967 (duplicate issue) + supersedes #96873 (same approach, closed by its author). **Practical impact**: MEDIUM — affects any Next.js 16.3.0 STABLE app that uses a Pages Router file named `sitemap`, `robots`, `manifest`, `icon`, or any other metadata-file basename. **Migration**: bump to `next@>=16.3.1-canary.17`.

4. **PR #97276 — `fix(deps): bump satori and @vercel/og`** (styfle, merged 2026-08-14T11:36:11Z, 7 files / +5747/-6607, base `canary`) — **bump satori 0.26.0 → 0.29.0 + @vercel/og 0.7.x → 0.10.x**. The 7-file diff is the dep version bump in `package.json` + `pnpm-lock.yaml` + 5 generated/bundled files. satori 0.27.0 brought better emoji rendering; 0.28.0 added Tailwind CSS v4 token compatibility; 0.29.0 fixed a regression in 0.27.0. **Practical impact**: low-medium — affects apps using `next/og` (the Open Graph image generator) + satori directly. The biggest user-facing improvement is better OG image rendering for apps with emoji-heavy content. **Migration**: bump to `next@>=16.3.1-canary.17`; or independently `npm install @vercel/og@latest`.

### 4 Trace Instrumentation PRs in canary.17

- **PR #97295 — `Trace route module preparation`** (basement, low-material observability)
- **PR #97296 — `Trace instrumentation startup`** (basement, low-material observability)
- **PR #97318 — `Trace route module loading`** (basement, low-material observability)
- **PR #97342 — `Update blob version`** (next-js-bot, dep version bump)

These 4 PRs add new trace instrumentation hooks to the Next.js dev server for the agent-first observability story (the `next-dev-loop` skill). The hooks let the agent observe route module preparation + instrumentation startup + route module loading separately. **Practical impact**: low — only affects agents / observability tools that subscribe to the new trace hooks. **Migration**: no action required; the hooks are additive.

### 3 Turbopack Improvements in canary.17

- **PR #96898 — `test: Reenable i18n-api-support deploy test for Turbopack with adapters`** (test-only, re-enables a Turbopack-with-adapters test that was disabled)
- **PR #97353 — `Turbopack: don't trace embedded WASM loader helpers`** (dx improvement, reduces WASM loader trace noise)
- **PR #97370 — `Turbopack: improve chunk_item_content docs`** (docs-only)
- **PR #97373 — `Turbopack: add imports_wildcard NFT unit test`** (test-only)

These 4 PRs are the Turbopack bundle. **Practical impact**: low — only affects Turbopack bundle-size profiling + adapter-with-Turbopack deployments. **Migration**: no action required.

### 4 Docs/Test/CI PRs in canary.17

- **PR #97349 — `[test] Cover 'use cache: private' in route handlers`** (test-only, adds coverage for the `'use cache: private'` directive in route handlers)
- **PR #97341 — `docs: align SPA guide reducer example with React conventions`** (docs-only, aligns the SPA guide reducer example with React's official patterns)
- **PR #97342 — `Update blob version`** (version-tag, already counted in instrumentation)
- **PR #97296 — `Trace instrumentation startup`** (already counted in instrumentation)

These 4 PRs are docs/test/CI improvements. **Practical impact**: zero — no user-facing behavior change. **Migration**: no action required.

### Practical Impact Summary — `next@16.3.1-canary.17`

| User type | Pre-canary.17 (16.3.1 STABLE / 16.3.0 STABLE) | Post-canary.17 (16.3.1-canary.17+) |
|---|---|---|
| **Apps using deployment adapter + `output: 'standalone'`** | `Error: ENOENT: no such file or directory, open '<distDir>/next-server.js.nft.json'` (FIX #97287) | Build succeeds; NFTs emitted correctly |
| **Apps using deployment adapter + Pages API routes** | `Cannot find module 'next/dist/compiled/next-server/pages-turbo.runtime.prod.js'` (FIX #96819) | Pages API route initialization succeeds |
| **Apps using Pages Router with `sitemap.js` / `robots.js` (or any metadata-filename)** | `Error: "getStaticProps" is not supported in app/.` (FIX #97350) | Build succeeds; pages-router files compile correctly |
| **Apps using `next/og` / `@vercel/og`** | Older satori version with emoji rendering issues (FIX #97276) | Bumped satori 0.29.0 + @vercel/og 0.10.x with better emoji + Tailwind v4 token support |
| **Vercel deployments on 16.3.0 with adapters** | **3 BLOCKERS** (NFT + Pages API + Pages Router) | All 3 blockers resolved |
| **Self-hosted deployments on 16.3.0 with `cdk-nextjs` adapter** | **2 BLOCKERS** (NFT + Pages API) | Both blockers resolved |
| **Apps using Cache Components + RDC** | No change (the v1.5.55-PR #95238 fix is already shipped) | No change |
| **Apps using `experimental.testProxy`** | No change (the v1.5.55-PR #96525 fix is already shipped) | No change |
| **Apps not using `output: 'standalone'` or adapters** | Not affected | Not affected |

### When this ships — Forward-looking on the next Next.js canary cut

PR #97287 + PR #96819 + PR #97350 + PR #97276 + the 11 trace/Turbopack/docs/test PRs all merged to Next.js's `canary` branch before the `16.3.1-canary.17` npm-publish (PR #97287 at 2026-08-14T09:09:02Z = ~8h before npm-publish; PR #97342 at 2026-08-14T06:41:33Z = ~10h before npm-publish). All are **currently SHIPPED in this canary cut**. The next React canary cut is expected within 0-72h on the typical 24h cadence. **The next stable cut**: `next@16.3.2 STABLE` is expected within 1-2 weeks (the team typically waits 1-2 weeks of canary stability before STABLE). The 4 MATERIAL PRs in canary.17 (especially PR #97287 + PR #96819 + PR #97350) are the **PRIORITY** for the 16.3.2 STABLE cut — every deployed app on 16.3.0 STABLE that uses adapters needs these fixes.

### Audit Recipe (5 Steps)

1. **`npm ls next`** — confirm the canary version (`>=16.3.1-canary.17` is the target).
2. **Audit adapter + `output: 'standalone'` combination** — `rg -n "output: ['\"]standalone['\"]|NEXT_ADAPTER_PATH|NEXT_ENABLE_ADAPTER" next.config.* package.json`. **Action**: if you see both, you're affected by PR #97287. Bump to canary.17 or higher.
3. **Audit Pages API routes + adapter combination** — `rg -n "export const|export async function" pages/api/ -l | head -20` + `rg -n "NEXTAUTH_URL|NEXT_ADAPTER_PATH" next.config.*`. **Action**: if you see both, you're affected by PR #96819.
4. **Audit Pages Router with `sitemap.js` / `robots.js`** — `ls pages/sitemap.js pages/robots.js 2>/dev/null`. **Action**: if either exists, you're affected by PR #97350 (and MUST be on canary.17+).
5. **Audit `next/og` / `@vercel/og` usage** — `rg -n "next/og|@vercel/og" app/ src/ --type ts --type tsx --type js --type jsx`. **Action**: if you use them, the satori 0.29.0 bump in PR #97276 brings better emoji rendering for free.

### Common Mistakes Section Grows — 3 New Bullets

- **Adapter + `output: 'standalone'` build crashes on 16.3.0 STABLE (ENOENT next-server.js.nft.json) — FIXED in 16.3.1-canary.17+ (PR #97287)** — bryan-ferry, merged 2026-08-14T09:09:02Z. The bug: PR #93684 stopped emitting the whole-app server NFTs whenever an adapter is configured, but the `output: 'standalone'` finalizer reads the whole-app NFT unconditionally. The combination was never covered by CI; the only state where the writer gate and the reader disagree. **Symptom**: `next build` crashes with `Error: ENOENT: no such file or directory, open '<distDir>/next-server.js.nft.json'`. **The fix**: PR #97287 re-scopes PR #93684's gate to skip the whole-app server NFTs only when an adapter is configured AND standalone output is not. **Action**: bump to canary.17 or higher; if stuck on 16.3.0 STABLE, **DO NOT use `output: 'standalone'` with an adapter** until you can upgrade. **Audit recipe**: `rg -n "output: ['\"]standalone['\"]|NEXT_ADAPTER_PATH|NEXT_ENABLE_ADAPTER" next.config.* package.json`.
- **Pages API route initialization fails on 16.3.0 STABLE with adapter (Cannot find module pages-turbo.runtime.prod.js) — FIXED in 16.3.1-canary.17+ (PR #96819)** — timneutkens, merged 2026-08-14T08:38:29Z. The bug: Pages API entry traces don't discover the regular Pages renderer because `pages/module.compiled.js` selects the bundler-specific production renderer via a require-hook chain. The Pages API trace only contains `pages-api[-turbo].runtime.prod.js`, not `pages[-turbo].runtime.prod.js`. **The fix**: PR #96819 traces the hidden Pages renderer dependency using the mechanism appropriate to each bundler (Turbopack: add to `pages_traced_modules`; Webpack: run Node File Trace on `pages.runtime.prod.js`). **Symptom**: Pages API routes fail with `Cannot find module 'next/dist/compiled/next-server/pages-turbo.runtime.prod.js'` at function initialization. **Action**: bump to canary.17 or higher. **Audit recipe**: `rg -n "export async function|export const" pages/api/ -l | head -20` + `rg -n "NEXT_ADAPTER_PATH" next.config.*`.
- **Pages Router `sitemap.js` / `robots.js` build fails on 16.3.0 STABLE (getStaticProps is not supported in app/) — FIXED in 16.3.1-canary.17+ (PR #97350)** — unstubbable, merged 2026-08-14T09:53:25Z. The bug: PR #94962 added the metadata conventions to the app-entry filename regex in `ReactServerComponentValidator::assert_invalid_api`. The regex only looks at the filename, so a file like `pages/sitemap.js` is now mistaken for an app entry. **The fix**: PR #97350 scopes app-entry validation to files inside `appDir` (the gate `assert_server_filename` already applies to `error.js`). **Symptom**: `Error: "getStaticProps" is not supported in app/.` for pages-router files named `sitemap`, `robots`, `manifest`, `icon`, or any other metadata-filename basename. **Action**: bump to canary.17 or higher. **Audit recipe**: `ls pages/sitemap.js pages/robots.js pages/manifest.js pages/icon.js 2>/dev/null`.

### Sources

- [Next.js `v16.3.1-canary.17` GitHub release notes (August 14, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.17) — npm-published 2026-08-14T17:20:01Z; the 15-commit canary.16 → canary.17 diff
- [Next.js canary-branch compare `v16.3.1-canary.16...v16.3.1-canary.17`](https://github.com/vercel/next.js/compare/v16.3.1-canary.16...v16.3.1-canary.17) — 15 commits ahead as of 2026-08-14T17:33Z (the 4 MATERIAL PRs + 4 trace instrumentation PRs + 3 Turbopack PRs + 4 docs/test/CI PRs)
- [Next.js canary-branch compare `v16.3.1-canary.17...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.17...canary) — 0 commits ahead as of 2026-08-14T17:33Z (the team just shipped; no new canary-branch activity since the canary.17 tag)
- [PR #97287 — `Emit whole-app server NFTs when output: 'standalone' is used with an adapter`](https://github.com/vercel/next.js/pull/97287) — by bryan-ferry, merged 2026-08-14T09:09:02Z, 2 files / +44/-2. **THE HEADLINE PR** for adapter + standalone deployments; fixes the ENOENT crash on 16.3.0 STABLE.
- [PR #96819 — `Fix missing Pages runtime in adapter Pages API outputs`](https://github.com/vercel/next.js/pull/96819) — by timneutkens, merged 2026-08-14T08:38:29Z, 11 files / +191/-5. **THE HEADLINE PR** for Pages API + adapter deployments; fixes the Pages renderer trace miss.
- [PR #97350 — `Scope app-entry export validation to files inside the app directory`](https://github.com/vercel/next.js/pull/97350) — by unstubbable, merged 2026-08-14T09:53:25Z, 47 files / +79/-18. **THE HEADLINE PR** for Pages Router + metadata-filename; fixes the getStaticProps-is-not-supported-in-app error.
- [PR #97276 — `fix(deps): bump satori and @vercel/og`](https://github.com/vercel/next.js/pull/97276) — by styfle, merged 2026-08-14T11:36:11Z, 7 files / +5747/-6607. Bumps satori 0.26.0 → 0.29.0 + @vercel/og 0.7.x → 0.10.x.
- [PR #97295 — `Trace route module preparation`](https://github.com/vercel/next.js/pull/97295) — by basement, low-material observability.
- [PR #97296 — `Trace instrumentation startup`](https://github.com/vercel/next.js/pull/97296) — by basement, low-material observability.
- [PR #97318 — `Trace route module loading`](https://github.com/vercel/next.js/pull/97318) — by basement, low-material observability.
- [PR #97342 — `Update blob version`](https://github.com/vercel/next.js/pull/97342) — by next-js-bot, dep version bump.
- [PR #96898 — `test: Reenable i18n-api-support deploy test for Turbopack with adapters`](https://github.com/vercel/next.js/pull/96898) — test-only.
- [PR #97353 — `Turbopack: don't trace embedded WASM loader helpers`](https://github.com/vercel/next.js/pull/97353) — by mischnic, dx improvement.
- [PR #97370 — `Turbopack: improve chunk_item_content docs`](https://github.com/vercel/next.js/pull/97370) — docs-only.
- [PR #97373 — `Turbopack: add imports_wildcard NFT unit test`](https://github.com/vercel/next.js/pull/97373) — test-only.
- [PR #97349 — `[test] Cover 'use cache: private' in route handlers`](https://github.com/vercel/next.js/pull/97349) — test-only.
- [PR #97341 — `docs: align SPA guide reducer example with React conventions`](https://github.com/vercel/next.js/pull/97341) — docs-only.
- [Issue #96646 — `next build` crashes with ENOENT next-server.js.nft.json for adapter + standalone apps](https://github.com/vercel/next.js/issues/96646) — closed by PR #97287.
- [Issue #96657 — root-cause chain repro for adapter + standalone](https://github.com/vercel/next.js/issues/96657) — closed by repro-link bot; tracked under #96646.
- [Issue #96859 — `getStaticProps is not supported in app/` for pages-router sitemap.js](https://github.com/vercel/next.js/issues/96859) — closed by PR #97350.
- [PR #93684 — the PR that gated the whole-app server NFTs behind `is_standalone == false`](https://github.com/vercel/next.js/pull/93684) — the PR that PR #97287 re-scopes.
- [PR #94962 — the PR that added metadata conventions to the app-entry filename regex](https://github.com/vercel/next.js/pull/94962) — the PR that PR #97350 narrows with `appDir` scoping.
- [Next.js `next.config.ts` adapter docs](https://nextjs.org/docs/app/api-reference/config/next-config-js/output) — the `output: 'standalone'` reference.
- [cdk-nextjs README — the adapter that force-sets `output: 'standalone'`](https://github.com/cdklabs/cdk-nextjs) — the AWS CDK construct that PR #97287 unblocks.
- [Next.js `next@16.3.1-canary.17` npm publish time](https://registry.npmjs.org/next) — `2026-08-14T17:20:01.370Z`.
- Cross-reference: `deployment.md` → `## Next.js — next@16.3.1-canary.15-ahead — Resume Data Cache (RDC) Compression Rollout Controls (PR #97247, August 12-13, 2026) + Testmode Passthrough Fetch Infinite-Recursion Fix (PR #96525, August 13, 2026)` for the v1.5.55 lens that now precedes this canary.17 cycle.
- Cross-reference: `components.md` → `## React 19.3.0-canary-eb8feb71-20260814 SHIPPED (August 14, 2026) — PR #37169 Fragment Once Listeners + PR #37290 HEADLINE enableParallelTransitions Flag Flipped ON` for the React canary.eb8feb71 SHIP event that shared the same 2026-08-14T17:33Z window.
- Cross-reference: `routing.md` → `## 16.3.1-canary.13-ahead — Restore the Live headers() View of the Incoming Request (PR #97166) + Fix Unset crossOrigin in Turbopack Manifests (PR #97164)` for the routing-impact angle on PR #96819 (Pages API route initialization).
- Cross-reference: `setup.md` → `## Next.js 16.3.1-canary.12 SHIPPED (August 11, 2026) — Setup Recipe Lens` for the next.config.ts setup recipe that PR #97287's `is_standalone` gate interacts with.
- Cross-reference: v1.5.59 SKILL.md observation that **server-components.md was last touched by v1.5.56 with the PR #95238 + PR #97249 lens; the v1.5.58 cycle's "23h 49min stale WITHOUT new material" evaluation was a documentation miss — Next.js canary.17 SHIPPED 2026-08-14T17:20:01Z with 15 NEW commits (4 MATERIAL PRs + 4 trace instrumentation PRs + 3 Turbopack PRs + 4 docs/test/CI PRs), clearly material for server-components.md; this cycle corrects the miss**.

## Next.js 16.3.1-canary.20 SHIPPED (August 16, 2026) — 5 Commits + **Extract Metadata Resolution Primitives (PR #97388)** — Server Components / RSC Lens (npm-published 2026-08-16T00:02:44Z)

The v1.5.60 cycle documented canary.17 with 15 NEW commits (PR #97287 + PR #96819 + PR #97350 + PR #97276 from a server-components/RSC lens). The v1.5.62 cycle deferred the routing.md lens to canary.19 PRs. The v1.5.63 cycle did NOT touch server-components.md. The v1.5.64 cycle did NOT touch server-components.md.

canary.20 was published ~6h before this cron at 2026-08-16T00:02:44.804Z. **PR #97388** is the only canary.20 commit that materially affects the Server Components / RSC lens — it's a behavioral-preserving refactor that splits the metadata resolution pipeline into a reusable primitives module. The other 4 canary.20 commits (PR #94157 + PR #97372 + PR #97321 + PR #97415) are covered by routing.md (PR #94157 server route matcher stack) and performance.md (PR #97372 Turbopack retain conditions) and cross-referenced at the bottom of this section.

### PR #97388 — Extract metadata resolution primitives — Server Components / RSC deep dive

The metadata resolver (`resolve-metadata.ts`) previously combined two responsibilities:
1. **Walking the loader tree** to collect + schedule route exports
2. **Interpreting** the metadata or viewport value produced by an individual route layer

This PR moves the route-layer operations into `metadata-resolution-primitives.ts`. The extracted module owns:

- Wrapping `generateMetadata` and `generateViewport` (preserving their async semantics + parent-promise behavior)
- Loading file-based metadata (`favicon.ico`, `icon.{ico,png,jpg,svg}`, `apple-icon.{jpg,png}`, `opengraph-image.*`, `twitter-image.*`, `sitemap.{js,ts}`, `robots.{js,ts}`, `manifest.{json,js,ts}`)
- Merging values with their resolved parents (the cascade rule that turns nested `app/blog/[slug]/page.tsx` + `app/page.tsx` + root metadata into the final `<head>` payload)
- Post-processing metadata (URL resolution, asset prefix application, Robots / Sitemap / Manifest shape normalization)
- Producing `SelectedMetadata` for rendering (the wrapper that decides which tags to emit per platform — meta-name / meta-property / link / etc.)

`resolve-metadata.ts` keeps the existing traversal + scheduling flow, imports the extracted operations back into the same call sites, and preserves its existing exports. **No new public-API surface**.

**Behavior-preserving** by design — verified via `pnpm build-all` in the PR's verification step:

- Traversal order — unchanged
- Eager generator invocation — `generateMetadata` + `generateViewport` invoked at the same point in the request lifecycle
- Parent-promise behavior — parent metadata is awaited before the merge step exactly as before
- Warnings — `app-dir-metadata-static-export-warnings`, `Sitemap-robots-static-export-warnings`, etc. fire at the same points with the same messages
- Rendered metadata tags + viewport tags — byte-identical to pre-canary.20

### Why this matters for the RSC lens

1. **Future traversal strategies**. The reusable boundary means alternative traversal paths can be plugged in without touching `resolve-metadata.ts`. Concrete candidates: parallel-route metadata composition (per-slot metadata with `<ParallelRoutePlaceholder>` injection), edge-cache-aware metadata (RDC-aware metadata that participates in the resume-data-cache compression rollout from PR #97247), request-metadata-coupled metadata (alongside PR #94157's route-definitions living in request metadata).

2. **Test surface expansion**. With the operations extracted, future refactors of the **traversal** (resolve-metadata) and the **operations** (metadata-resolution-primitives) can be tested independently. The PR doesn't add tests in this batch, but the next metadata-system PR will.

3. **`SelectedMetadata` is now self-contained**. PR #97387 (canary.19) adopted `SelectedMetadata` for metadata rendering. PR #97388 extracts the **production** of `SelectedMetadata` into the primitives. Combined, the two PRs move Next.js toward a single ownership: `metadata-resolution-primitives.ts` produces + `next/dist/server/render` consumes. Future canvas / ImageResponse / non-HTML rendering paths can adopt the same primitive without forking the metadata resolver.

### Migration guide

```ts
// No code change required. Verify post-upgrade:
import { resolveMetadata } from 'next/dist/lib/metadata-resolve-metadata'
// The public surface is preserved. resolveMetadata() + resolveViewport() return the same shapes.

// If you depend on @internal paths (anti-pattern, but documented for completeness):
// The internal modules moved: src/server/lib/router-utils/resolve-metadata.ts → split into
//   resolve-metadata.ts + metadata-resolution-primitives.ts
// Imports like `'next/dist/server/lib/router-utils/resolve-metadata'` still resolve
// (re-exports), but new code should import from the primitives module directly.

import {
  accumulateMetadata,
  accumulateViewport,
  resolveMetadataForRoute,
  resolveViewportForRoute
} from 'next/dist/server/lib/router-utils/metadata-resolution-primitives'
// These are the new functions extracted from resolve-metadata.ts.
```

### Recommended version pin

```bash
# Production — stay on @latest (16.3.1 STABLE) unless you specifically need canary.20
npm install next@latest  # → 16.3.1

# Canary evaluator — upgrade
npm install next@canary  # → 16.3.1-canary.20

# Frame authors + RSC power users on canary.19 → upgrade to canary.20 to confirm
# the behavior-preserving property (compare rendered <head> in production for
# the same routes + the same metadata cascades)
```

### Audit recipe (verify the behavior-preserving property)

```
1. Choose 5 representative routes with metadata cascades (root metadata + nested metadata + page-level generateMetadata + viewport + file-based metadata)
2. Capture the rendered HTML <head> on canary.19 (snapshot the bytes)
3. Upgrade to canary.20 (npm install next@canary)
4. Rebuild + restart production
5. Re-capture the rendered HTML <head>
6. Diff the two captures — they MUST be byte-identical (or differ only in path resolution orderings unrelated to <head> content)
7. If any <head> tag differs in content or attribute order, OPEN AN ISSUE — the PR promises behavior-preserving
```

### Why canary.20 is significant for Server Components

Next.js's metadata pipeline has been a contributor to RSC + SSR + Streaming SSR + Cache Components + RDC-compression correctness. The pipeline's two-responsibilities-in-one-module design was a constraint on every metadata-adjacent PR going back to 15.x. **PR #97388 is the first refactor that splits the module into reusable + traversable parts** without altering behavior. Expect follow-up PRs in canary.21+ to:

1. Lift the metadata primitives into request-metadata alongside PR #94157's route-definitions
2. Parallel-route per-slot metadata composition using the primitives (probably 16.4 if not before)
3. RDC-cacheable metadata (the resume-data-cache compression rollout from PR #97247 could apply to metadata snapshots)

### Sources

- [Next.js `v16.3.1-canary.20` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.20) — released by `@next-js-bot` 2026-08-15T23:52Z; npm-published 2026-08-16T00:02:44Z; 5 commits; 0 CVE-class
- [PR #97388 — Extract metadata resolution primitives](https://github.com/vercel/next.js/pull/97388) — by @byebyers; the metadata primitives split; behavior-preserving refactor
- [Next.js `next@16.3.1-canary.20` npm publish time](https://registry.npmjs.org/next) — `2026-08-16T00:02:44.804Z`
- [Cross-reference: `routing.md` → `## Next.js 16.3.1-canary.20 SHIPPED — 5 Commits: Remove Server Route Matcher Stack + Extract Metadata Resolution Primitives + ... — Routing-Lens` — PR #94157 + PR #97388 from the routing-system lens
- [Cross-reference: `performance.md` → `## Next.js 16.3.1-canary.20 SHIPPED — PR #97372 Turbopack Retain Conditions for Resolve Request Keys — Performance-Lens` — PR #97372 from the Turbopack production-blocker lens
- [Cross-reference: `components.md` → `## React 19.3.0-canary-eb8feb71 SHIPPED (August 14, 2026) — PR #37169 Fragment Once Listeners + PR #37290 enableParallelTransitions Flag Flipped ON` — the React canary SHIP event that canary.20 shares the bundling window with (PR #97249 vendor bump 22e4f993 followed by PR #97388 metadata pipeline refactor — neither is React-bundled; they ship with the next/react vendor bump 24e4f993 → beef6d60 → eb8feb71 ahead of itself)
- [Cross-reference: v1.5.60 server-components.md `## Next.js 16.3.1-canary.17 SHIPPED — 4 MATERIAL PRs + 15 Commits` — the previous canary-batch to touch server-components.md (canary.17 was 1d 22h 42min before canary.20)
- [PR #97387 — Adopt SelectedMetadata for metadata rendering](https://github.com/vercel/next.js/pull/97387) — the canary.19 consumer of `SelectedMetadata`; pairs with PR #97388 as producer
