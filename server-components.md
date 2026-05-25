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

## Next.js 15 Caching — Important Changes

In Next.js 15, the default caching behavior for `fetch` changed. **Dynamic is the new default** — data is fetched on every request unless explicitly cached.

### Cache Options in Next.js 15

```tsx
// Dynamic (no cache) — every request fetches fresh data
// This is NOW THE DEFAULT in Next.js 15
const stats = await fetch('https://api.example.com/stats', {
  cache: 'no-store',
})
```

```tsx
// Static (cached) — revalidate every hour
const posts = await fetch('https://api.example.com/posts', {
  cache: 'force-cache',
  next: { revalidate: 3600 },
})
```

```tsx
// Never cache (force dynamic)
const data = await fetch('https://api.example.com/real-time', {
  cache: 'no-store',
})
```

**Summary of changes in Next.js 15:**
- `cache: 'no-store'` is now the default behavior (previously needed explicit opt-out)
- `cache: 'force-cache'` (previously `cache: 'true'`) for static data
- `revalidateTag` and `revalidatePath` still work as before

### `unstable_cache` — Granular Function-Level Caching

For non-`fetch` data sources (direct DB calls, external APIs without `fetch`), use `unstable_cache`:

```tsx
// lib/data.ts
import { unstable_cache } from 'next/cache'

// Cache this DB query with tags
export const getTopPosts = unstable_cache(
  async (limit: number) => {
    return db.post.findMany({
      where: { published: true },
      orderBy: { views: 'desc' },
      take: limit,
    })
  },
  ['top-posts'],           // cache key parts
  { 
    tags: ['posts'],       // for revalidation
    revalidate: 3600,      // or use tags, not both
  }
)

// In a server component:
export default async function PopularPosts() {
  const posts = await getTopPosts(10)
  return <PostList posts={posts} />
}
```

**When to use `unstable_cache` vs `fetch` with `next` option:**
- Use `fetch` with `next: { tags: [...] }` when fetching from external HTTP APIs
- Use `unstable_cache` for direct database calls or other non-fetch data sources

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

In React 19 / Next.js 15, you can pass Promises from server to client:

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
import { revalidatePath } from 'next/cache'

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
  revalidatePath('/posts')
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
- **Forgetting `revalidatePath`** — after mutations with Server Actions, revalidate caches
- **Relying on old fetch caching defaults** — in Next.js 15, `cache: 'no-store'` is the default; use `cache: 'force-cache'` with `revalidate` for static data
- **Using `unstable_cache` for fetch-based data** — use `fetch` with `next: { tags: [...] }` instead
