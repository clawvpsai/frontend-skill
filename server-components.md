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
