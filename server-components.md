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

`revalidateTag` schedules the cached entry to be refreshed on the **next request** — the stale data is served while revalidation happens in the background:

```tsx
// app/actions.ts — revalidate after mutation
'use server'

import { revalidateTag } from 'next/cache'

export async function createPost(formData: FormData) {
  const parsed = createPostSchema.parse(Object.fromEntries(formData))
  await db.post.create({ data: parsed })

  // Schedule revalidation — stale data served until next request refetches
  revalidateTag('posts')
}
```

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
| `revalidateTag` | Background refresh | Fast (serves stale) | Non-critical data, high-traffic pages |
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
  revalidateTag('posts')
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
- **Relying on implicit caching** — in Next.js 16, everything is dynamic by default; use `use cache` to opt into caching
- **Using `unstable_cache` in new code** — use `use cache` + `cacheTag` instead in Next.js 16
- **`use()` without an Error Boundary** — if the Promise rejects, `use()` throws; you need an Error Boundary to catch it
- **Reading cookies/headers inside `use cache`** — read them outside the cached scope and pass as arguments
- **`use()` for Context without `'use client'`** — this only works in Client Components; always add `'use client'` when consuming Context with `use()`
- **`use cache` surviving deploys without explicit invalidation** — cache persists across deployments; add deploy-time invalidation if fresh data is needed immediately after deploy

**Sources:**
- [Next.js `use cache` directive](https://nextjs.org/docs/app/api-reference/directives/use-cache)
- [Next.js 16 release notes](https://nextjs.org/blog/next-16)
- [Next.js `cacheTag`](https://nextjs.org/docs/app/api-reference/functions/cacheTag)
- [Next.js `revalidateTag`](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)
- [Next.js `updateTag`](https://nextjs.org/docs/app/api-reference/functions/updateTag)
- [React 19.2.7 release](https://github.com/facebook/react/releases/tag/v19.2.7)
