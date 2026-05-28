# Server Components — RSC Mental Model

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

### On-Demand Revalidation with `cacheTag` + `revalidateTag`

```tsx
// lib/data.ts — tag the cache entry
import { cacheTag } from 'next/cache'

export async function getPosts() {
  'use cache'
  cacheTag('posts')  // Tag this cache for revalidation
  return db.post.findMany()
}
```

```tsx
// app/actions.ts — revalidate after mutation
'use server'

import { revalidateTag } from 'next/cache'
import { createPostSchema } from '@/lib/validations'

export async function createPost(formData: FormData) {
  const parsed = createPostSchema.parse(Object.fromEntries(formData))
  await db.post.create({ data: parsed })

  // Invalidate all cached data tagged with 'posts'
  revalidateTag('posts')
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

### Consuming Context with `use()` (React 19)

React 19's `use()` hook can consume Context, even in components that can't use hooks at the top level:

```tsx
// components/user-badge.tsx
'use client'

import { use } from 'react'
import { ThemeContext } from '@/contexts/theme-context'

export function UserBadge() {
  // Unlike useContext(), use() works inside conditional statements
  // and during render (though with caveats — only for Suspense-compatible data)
  const theme = use(ThemeContext)
  return <span className={theme === 'dark' ? 'text-white' : 'text-black'}>{theme}</span>
}
```

```tsx
// Conditional consumption with use() — only in client components
function Dashboard(props: { showStats: boolean }) {
  if (props.showStats) {
    const stats = use(StatsContext) // ✅ allowed in React 19
  }
  return <div>...</div>
}
```

**Note:** `use()` for Context is only available in Client Components. The component itself must have `'use client'`.

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
- **Forgetting `revalidateTag`** — after mutations with Server Actions, invalidate cached data
- **Relying on implicit caching** — in Next.js 16, everything is dynamic by default; use `use cache` to opt into caching
- **Using `unstable_cache` in new code** — use `use cache` + `cacheTag` instead in Next.js 16
- **`use()` without an Error Boundary** — if the Promise rejects, `use()` throws; you need an Error Boundary to catch it
- **Reading cookies/headers inside `use cache`** — read them outside the cached scope and pass as arguments

**Sources:**
- [Next.js `use cache` directive](https://nextjs.org/docs/app/api-reference/directives/use-cache)
- [Next.js 16 release notes](https://nextjs.org/blog/next-16)
- [Next.js `cacheTag`](https://nextjs.org/docs/app/api-reference/functions/cacheTag)
- [Next.js `revalidateTag`](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)
