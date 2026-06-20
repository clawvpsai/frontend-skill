# State — Zustand + React Query

## State Architecture

Two distinct types of state, two different tools:

| Type | Tool | Examples |
|---|---|---|
| **Server State** (async, cached, shared) | React Query / TanStack Query | API data, lists, user profile |
| **Client State** (sync, local) | Zustand | UI state, modals, theme, cart |

**Never use Zustand for server data.** Never use React Query for client state.

## React Query Setup

### `lib/api.ts` (TanStack Query v5)

```tsx
import { QueryClient } from '@tanstack/react-query'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,  // 5 minutes before refetch
      gcTime: 1000 * 60 * 10,   // 10 minutes in cache (was cacheTime in v4)
      retry: 1,                   // retry failed requests once
      refetchOnWindowFocus: true, // refetch when tab regains focus
    },
  },
})
```

### `app/providers.tsx`

```tsx
'use client'
import { QueryClientProvider } from '@tanstack/react-query'
import { queryClient } from '@/lib/api'

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}
```

```tsx
// app/layout.tsx
import { Providers } from './providers'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  )
}
```

## React Query Data Fetching

### Basic Query

```tsx
'use client'
import { useQuery } from '@tanstack/react-query'
import { fetchUser } from '@/lib/api'

export function UserProfile({ userId }: { userId: string }) {
  const { data, isLoading, error, isError } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  })

  if (isLoading) return <Skeleton />
  if (isError) return <p>Error: {error.message}</p>

  return <div>{data.name}</div>
}
```

### Query with Dependent Data

```tsx
const { data: user } = useQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
  enabled: !!userId, // won't run until userId is truthy
})
```

### Parallel Queries

```tsx
const { data: [user, posts] } = useQueries({
  queries: [
    { queryKey: ['user', id], queryFn: () => fetchUser(id) },
    { queryKey: ['posts', id], queryFn: () => fetchPosts(id) },
  ],
})
```

## Mutations

### Basic Mutation

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query'

export function CreatePostButton() {
  const queryClient = useQueryClient()

  const mutation = useMutation({
    mutationFn: (newPost: CreatePostInput) => createPost(newPost),
    onSuccess: () => {
      // Invalidate and refetch the posts list
      queryClient.invalidateQueries({ queryKey: ['posts'] })
    },
    onError: (error) => {
      console.error('Failed to create post:', error)
    },
  })

  return (
    <button 
      onClick={() => mutation.mutate({ title: 'New Post', content: '...' })}
      disabled={mutation.isPending}
    >
      {mutation.isPending ? 'Creating...' : 'Create Post'}
    </button>
  )
}
```

### Optimistic Updates (useMutation pattern)

```tsx
const mutation = useMutation({
  mutationFn: (updatedPost: Post) => updatePost(updatedPost),
  
  onMutate: async (newPost) => {
    // Cancel any outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['post', newPost.id] })
    
    // Snapshot previous value
    const previous = queryClient.getQueryData(['post', newPost.id])
    
    // Optimistically update
    queryClient.setQueryData(['post', newPost.id], newPost)
    
    return { previous }
  },
  
  onError: (err, newPost, context) => {
    // Roll back on error
    queryClient.setQueryData(['post', newPost.id], context?.previous)
  },
  
  onSettled: (data) => {
    // Always refetch after error or success
    queryClient.invalidateQueries({ queryKey: ['post', data.id] })
  },
})
```

### Optimistic Updates — React 19 `useOptimistic` (Recommended)

React 19 introduces `useOptimistic` — a declarative way to show immediate UI feedback while a mutation is pending. Unlike the `useMutation` pattern above, this doesn't require manual snapshot/rollback:

```tsx
'use client'

import { useOptimistic, useState } from 'react'
import { updatePost } from '@/app/actions'

interface Post {
  id: string
  content: string
  likes: number
}

export function LikeButton({ post }: { post: Post }) {
  const [liked, setLiked] = useState(false)

  // Optimistic state: shows updated likes immediately while action runs
  const [optimisticPost, addOptimisticPost] = useOptimistic(
    post,
    (state, newLikes: number) => ({ ...state, likes: newLikes })
  )

  async function handleLike() {
    const newLikes = optimisticPost.likes + 1
    // Apply optimistic update immediately
    addOptimisticPost(newLikes)
    setLiked(true)

    try {
      await updatePost(post.id, { likes: newLikes })
    } catch {
      // If the action fails, React reverts to the actual server state
      setLiked(false)
    }
  }

  return (
    <button onClick={handleLike}>
      {optimisticPost.likes} likes
    </button>
  )
}
```

**When to use which:**

| Pattern | Best for | Why |
|---|---|---|
| **`useOptimistic`** | Form actions, simple toggles (like/subscribe/delete), list operations driven by Server Actions | React auto-reverts on error; works seamlessly with Server Actions; no manual cache plumbing; **recommended in Next.js 16 + React 19.2** |
| **`useMutation` + `onMutate` / `onError` / `onSettled`** | Shared REST/RPC mutations that affect the React Query cache, cross-component invalidation, complex rollback logic | Full control over the React Query cache; can update multiple queries atomically; works with non-action HTTP endpoints |

**Rule of thumb:** If the mutation is a Server Action called from a form button, use `useOptimistic`. If the mutation is called from a `fetch`-based client API and you need to keep the React Query cache in sync, use `useMutation`.

**Common mistake — `useOptimistic` requires a transition (or async action) to take effect.** The `addOptimistic(...)` call only "sticks" when wrapped in a `startTransition` (or when called from inside an async function passed to a form action). Calling it from a synchronous event handler without a transition causes the optimistic state to flash and revert immediately.

```tsx
'use client'

import { useOptimistic, useState, startTransition } from 'react'
import { updatePost } from '@/app/actions'

// ❌ Optimistic state may not show — no transition wrapping the action
function handleLike() {
  addOptimisticPost(newLikes)
  void updatePost(post.id, { likes: newLikes })
}

// ✅ Wrap the action in a transition so the optimistic state actually shows
function handleLike() {
  startTransition(async () => {
    addOptimisticPost(newLikes)
    await updatePost(post.id, { likes: newLikes })
  })
}
```

### Optimistic List Operations with `useOptimistic`

`useOptimistic` works for list operations too — not just single-item updates. The key is the reducer pattern: provide the current state and the action, return the new optimistic state:

```tsx
'use client'

import { useOptimistic, useState, useRef } from 'react'
import { addComment, deleteComment } from '@/app/actions'

interface Comment {
  id: string
  text: string
  author: string
  createdAt: Date
}

type OptimisticAction =
  | { type: 'add'; comment: Comment }
  | { type: 'delete'; id: string }

// Optimistic state + dispatcher
const [optimisticComments, dispatchOptimistic] = useOptimistic(
  comments,  // current state
  (state: Comment[], action: OptimisticAction): Comment[] => {
    switch (action.type) {
      case 'add':
        return [...state, action.comment]
      case 'delete':
        return state.filter(c => c.id !== action.id)
      default:
        return state
    }
  }
)

// Add optimistically
async function handleAddComment(text: string) {
  const tempId = `temp-${Date.now()}`
  const optimisticComment: Comment = {
    id: tempId,
    text,
    author: 'You',
    createdAt: new Date(),
  }

  // Apply optimistic update immediately
  dispatchOptimistic({ type: 'add', comment: optimisticComment })

  try {
    const result = await addComment({ text })
    // On success, the server responds — the temp item stays until the list re-fetches
  } catch {
    // React automatically reverts to the last committed state on error
  }
}

// Delete optimistically
async function handleDeleteComment(id: string) {
  dispatchOptimistic({ type: 'delete', id })

  try {
    await deleteComment(id)
  } catch {
    // React reverts — deleted item reappears
  }
}

// Render uses optimisticComments, not comments
return (
  <div>
    {optimisticComments.map(c => (
      <CommentItem
        key={c.id}
        comment={c}
        onDelete={() => handleDeleteComment(c.id)}
      />
    ))}
  </div>
)
```

**Why use `useOptimistic` for lists:**
- User sees the change instantly — no flicker, no loading spinner
- If the server action fails, React reverts to the last committed state automatically
- Works with React Query's `invalidateQueries` — after `onSettled`, the list refetches and replaces the optimistic item with the real one

**Caveat:** Temporary IDs (`temp-${Date.now()}`) prevent key conflicts during the brief window before the server responds. This prevents React from reusing the wrong DOM node when the real item arrives.

## Suspense-Based Data Fetching (Next.js 16 + React 19.2)

React 19.2 + TanStack Query v5 + Next.js 16 enable a **Suspense-first** data-fetching pattern. Instead of `useQuery` + manual `isLoading` checks, you use `useSuspenseQuery` (or `useQuery` with the experimental `use(query.promise)` hook) and let the nearest `<Suspense>` boundary render the loading state. This pairs naturally with `cacheComponents: true` in `next.config.ts` (Next.js 16) and PPR streaming.

### `useSuspenseQuery` — Replace `useQuery` for Static Layouts

```tsx
// app/posts/page.tsx — Server Component shell
import { Suspense } from 'react'
import { PostsList } from './posts-list'
import { PostsSkeleton } from './posts-skeleton'

export default function PostsPage() {
  return (
    <Suspense fallback={<PostsSkeleton />}>
      <PostsList />
    </Suspense>
  )
}
```

```tsx
// app/posts/posts-list.tsx — Client Component, no isLoading branch needed
'use client'

import { useSuspenseQuery } from '@tanstack/react-query'
import { fetchPosts } from '@/lib/api'
import { queryKeys } from '@/lib/query-keys'

export function PostsList() {
  // data is ALWAYS defined inside a Suspense boundary — no isLoading / isError needed
  const { data } = useSuspenseQuery({
    queryKey: queryKeys.posts.list({}),
    queryFn: fetchPosts,
  })

  return (
    <ul>
      {data.map(p => <li key={p.id}>{p.title}</li>)}
    </ul>
  )
}
```

**Why this is the recommended Next.js 16 pattern:**
- `data` is typed as the resolved value (no `data | undefined`) — better DX, fewer type guards
- The loading skeleton is colocated with the boundary, not inside the component
- Errors bubble to the nearest `<ErrorBoundary>` — no `isError` check needed
- Works seamlessly with PPR — the Suspense boundary becomes a streaming hole in the static shell

**Caveats:**
- `useSuspenseQuery` does **not** support `enabled: false` — dependent queries are handled by serial fetching inside the same component instead
- `placeholderData` is also unavailable — wrap your key changes in `startTransition` to avoid unwanted fallbacks
- `useSuspenseQuery` is a hook, so it must be called from a Client Component or inside another Client Component

### `useSuspenseInfiniteQuery` — Paginated Lists

```tsx
'use client'

import { useSuspenseInfiniteQuery } from '@tanstack/react-query'
import { fetchPostsPage } from '@/lib/api'
import { queryKeys } from '@/lib/query-keys'

export function PostsList() {
  const { data, fetchNextPage, hasNextPage } = useSuspenseInfiniteQuery({
    queryKey: queryKeys.posts.list({}),
    queryFn: ({ pageParam }) => fetchPostsPage(pageParam),
    initialPageParam: 0,
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
  })

  return (
    <div>
      {data.pages.flatMap(page => page.items).map(p => (
        <PostCard key={p.id} post={p} />
      ))}
      {hasNextPage && (
        <button onClick={() => fetchNextPage()}>Load more</button>
      )}
    </div>
  )
}
```

### `useSuspenseQueries` — Parallel Dependent Fetches

When you need to fetch N resources in parallel and the component cannot render until all are ready:

```tsx
'use client'

import { useSuspenseQueries } from '@tanstack/react-query'
import { fetchUser, fetchPosts, fetchFriends } from '@/lib/api'

export function ProfilePage({ userId }: { userId: string }) {
  // All three queries fetch in parallel; the component only renders after all resolve
  const [{ data: user }, { data: posts }, { data: friends }] = useSuspenseQueries({
    queries: [
      { queryKey: ['user', userId], queryFn: () => fetchUser(userId) },
      { queryKey: ['posts', userId], queryFn: () => fetchPosts(userId) },
      { queryKey: ['friends', userId], queryFn: () => fetchFriends(userId) },
    ],
  })

  return (
    <div>
      <h1>{user.name}</h1>
      <PostsList posts={posts} />
      <FriendsList friends={friends} />
    </div>
  )
}
```

**Caveat:** `useSuspenseQueries` re-mounts the component if **any** query re-fetches in the background (cancellations don't apply to suspense). Set a high `staleTime` to avoid that.

### Experimental: `use(query.promise)` + `React.use()` (React 19.2 + TanStack v5)

React 19.2 + TanStack Query v5 also support an **experimental** pattern where you call `useQuery` in a parent, then unwrap its `.promise` in a child with `React.use()` — letting you decide per-subtree whether to suspense or not:

```tsx
// app/dashboard/page.tsx — Server Component
import { QueryClient } from '@tanstack/react-query'
import { Dashboard } from './dashboard'

const queryClient = new QueryClient({
  defaultOptions: { queries: { experimental_prefetchInRender: true } },
})

export default async function DashboardPage() {
  // Prefetch on the server (PPR cacheable)
  await queryClient.prefetchQuery({
    queryKey: ['dashboard'],
    queryFn: fetchDashboard,
  })

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <Dashboard />
    </HydrationBoundary>
  )
}
```

```tsx
// app/dashboard/dashboard.tsx
'use client'

import { useQuery } from '@tanstack/react-query'
import { fetchDashboard } from '@/lib/api'
import { Skeleton } from '@/components/skeleton'

function DashboardCharts({ query }: { query: UseQueryResult<Dashboard> }) {
  // Only this subtree suspends — the rest of the page renders immediately
  const data = React.use(query.promise)
  return <Charts data={data} />
}

export function Dashboard() {
  const query = useQuery({ queryKey: ['dashboard'], queryFn: fetchDashboard })

  return (
    <div>
      <h1>Welcome back</h1>
      <React.Suspense fallback={<Skeleton variant="charts" />}>
        <DashboardCharts query={query} />
      </React.Suspense>
    </div>
  )
}
```

**When to use this pattern:**
- The page has both fast-to-load and slow-to-load sections
- You want the slow sections to suspend, but the fast sections to render immediately
- You're on React 19.2+ with TanStack Query v5 — older React versions don't support `React.use()` with promises

**Note:** This pattern is **experimental** in TanStack Query v5 and requires the `experimental_prefetchInRender: true` client config. It is not recommended for production-critical paths yet.

**Sources:**
- [TanStack Query v5 — Suspense guide](https://tanstack.com/query/v5/docs/framework/react/guides/suspense)
- [TanStack Query v5 — `useSuspenseQuery` reference](https://tanstack.com/query/v5/docs/framework/react/reference/useSuspenseQuery)
- [TanStack Query v5 — `useSuspenseInfiniteQuery` reference](https://tanstack.com/query/v5/docs/framework/react/reference/useSuspenseInfiniteQuery)
- [TanStack Query v5 — `useSuspenseQueries` reference](https://tanstack.com/query/v5/docs/framework/react/reference/useSuspenseQueries)
- [React 19.2 — `use()` hook with promises](https://react.dev/reference/react/use)

## Zustand Setup

```bash
npm install zustand
```

### Basic Store

```tsx
// stores/cart-store.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface CartItem {
  id: string
  name: string
  price: number
  quantity: number
}

interface CartStore {
  items: CartItem[]
  addItem: (item: Omit<CartItem, 'quantity'>) => void
  removeItem: (id: string) => void
  clearCart: () => void
  total: () => number
}

export const useCartStore = create<CartStore>()(
  persist(
    (set, get) => ({
      items: [],
      
      addItem: (item) => set((state) => {
        const existing = state.items.find(i => i.id === item.id)
        if (existing) {
          return {
            items: state.items.map(i => 
              i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i
            ),
          }
        }
        return { items: [...state.items, { ...item, quantity: 1 }] }
      }),
      
      removeItem: (id) => set((state) => ({
        items: state.items.filter(i => i.id !== id),
      })),
      
      clearCart: () => set({ items: [] }),
      
      total: () => get().items.reduce((sum, i) => sum + i.price * i.quantity, 0),
    }),
    { name: 'cart-storage' }  // localStorage key
  )
)
```

### Usage in Component

```tsx
import { useCartStore } from '@/stores/cart-store'

export function CartButton() {
  const items = useCartStore(s => s.items)
  const addItem = useCartStore(s => s.addItem)
  
  return (
    <button onClick={() => addItem({ id: '1', name: 'Widget', price: 9.99 })}>
      Add to Cart ({items.length})
    </button>
  )
}
```

**Selector pattern:** Always select specific slices, not the whole store — prevents unnecessary re-renders.

```tsx
// ❌ Bad — re-renders on any store change
const { items, addItem } = useCartStore()

// ✅ Good — re-renders only when items changes
const items = useCartStore(s => s.items)
```

### Zustand with Middleware

```tsx
import { create } from 'zustand'
import { persist } from 'zustand/middleware'
import { immer } from 'zustand/middleware/immer'

interface UIStore {
  theme: 'light' | 'dark' | 'system'
  sidebarOpen: boolean
  setTheme: (t: 'light' | 'dark' | 'system') => void
  toggleSidebar: () => void
}

export const useUIStore = create<UIStore>()(
  persist(
    immer((set) => ({
      theme: 'system',
      sidebarOpen: false,
      setTheme: (theme) => set(s => { s.theme = theme }),
      toggleSidebar: () => set(s => { s.sidebarOpen = !s.sidebarOpen }),
    })),
    { name: 'ui-storage' }
  )
)
```

## Combining React Query + Zustand

```tsx
// Zustand for UI state, React Query for server data
export function PostWithComments({ postId }: { postId: string }) {
  const [showComments, setShowComments] = useState(false)  // Zustand for this
  const { data: post } = useQuery({ queryKey: ['post', postId], queryFn: () => fetchPost(postId) })
  const { data: comments } = useQuery({ 
    queryKey: ['comments', postId], 
    queryFn: () => fetchComments(postId),
    enabled: showComments,  // Only fetch when panel is open
  })
  
  return (
    <div>
      <PostContent post={post} />
      <button onClick={() => setShowComments(!showComments)}>
        {showComments ? 'Hide' : 'Show'} Comments
      </button>
      {showComments && comments?.map(c => <Comment key={c.id} {...c} />)}
    </div>
  )
}
```

## API Client Pattern

```tsx
// lib/api-client.ts — typed fetch wrapper
async function apiClient<T>(
  endpoint: string,
  options?: RequestInit
): Promise<T> {
  const res = await fetch(endpoint, {
    headers: { 'Content-Type': 'application/json' },
    ...options,
  })
  
  if (!res.ok) {
    const error = await res.json().catch(() => ({ message: res.statusText }))
    throw new Error(error.message ?? `HTTP ${res.status}`)
  }
  
  return res.json()
}

// Usage
export const fetchUser = (id: string) => 
  apiClient<User>(`/api/users/${id}`)

export const createPost = (data: CreatePostInput) =>
  apiClient<Post>('/api/posts', { method: 'POST', body: JSON.stringify(data) })
```


## TanStack Query v5 — Query Key Factory

**Query key organization matters** as your app grows. The Query Key Factory pattern provides type-safe, centralized query key management — no more magic strings or inconsistent key shapes across your codebase.

### Install

```bash
npm install @lukemorales/query-key-factory
```

### Define Query Keys

```ts
// lib/query-keys.ts
import { createQueryKeyStore } from '@lukemorales/query-key-factory'

export const queryKeys = createQueryKeyStore({
  users: {
    all: null,
    list: (filters: { role?: string; search?: string }) => filters,
    detail: (id: string) => ({ id }),
  },
  posts: {
    all: null,
    list: (filters: { authorId?: string; published?: boolean }) => filters,
    detail: (id: string) => ({ id }),
    popular: (limit: number) => ({ limit }),
  },
  comments: {
    all: null,
    byPost: (postId: string) => ({ postId }),
  },
})
```

**What you get:**
- Full type inference — `queryKeys.users.list({ role: 'admin' })` returns `{ role: 'admin' }` not `Record<string, unknown>`
- Auto-complete in IDE — `queryKeys.` shows all defined key factories
- Single source of truth — change a key structure in one place

### Use in Queries

```tsx
// ❌ Magic strings — error-prone
const { data } = useQuery({ queryKey: ['users', 'detail', userId], queryFn: fetchUser })

// ✅ Query Key Factory — typed and discoverable
const { data } = useQuery({
  queryKey: queryKeys.users.detail(userId),
  queryFn: () => fetchUser(userId),
})
```

### Use in Mutations (Invalidation)

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { queryKeys } from '@/lib/query-keys'

export function useCreatePost() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: createPost,
    onSuccess: () => {
      // Invalidate all posts queries — any list with any filters will refetch
      queryClient.invalidateQueries({ queryKey: queryKeys.posts.all() })
      // Or invalidate just the list, keeping detail queries intact
      queryClient.invalidateQueries({ queryKey: queryKeys.posts.list({}) })
    },
  })
}

export function useUpdatePost() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: updatePost,
    onSuccess: (updated) => {
      // Invalidate the specific post detail
      queryClient.invalidateQueries({ queryKey: queryKeys.posts.detail(updated.id) })
      // Also refetch lists — they'll update to show the new data
      queryClient.invalidateQueries({ queryKey: queryKeys.posts.all() })
    },
  })
}
```

### Prefetching with Query Keys

```tsx
// In a server component or loader
import { queryKeys } from '@/lib/query-keys'

export default async function UserPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  
  // Prefetch for client navigation
  await queryClient.prefetchQuery({
    queryKey: queryKeys.users.detail(id),
    queryFn: () => fetchUser(id),
  })

  return <UserProfile userId={id} />
}
```

### Query Key Structure Convention

Follow this pattern for consistent, cache-friendly keys:

```
[entity]             → all items (e.g., queryKeys.users.all())
[entity].list(f)    → filtered list (e.g., queryKeys.posts.list({ published: true }))
[entity].detail(id) → single item (e.g., queryKeys.posts.detail('123'))
[entity].popular(n) → derived query (e.g., queryKeys.posts.popular(10))
```

**Invalidation strategy:**
- `queryClient.invalidateQueries({ queryKey: queryKeys.users.all() })` — refetches all user queries (list + detail)
- `queryClient.invalidateQueries({ queryKey: queryKeys.users.list({}) })` — refetches only list queries
- `queryClient.invalidateQueries({ queryKey: queryKeys.users.detail(userId) })` — refetches specific detail

**Sources:**
- [TanStack Query v5 best practices](https://tanstackship.com/blog/tanstack-query-v5-best-practices)
- [Query Key Factory docs](https://tanstack.com/query/v5/docs/framework/react/community/lukemorales-query-key-factory)

## Zustand v5 — Key Migration Differences from v4

Zustand v5 (5.0.14+) has three breaking changes from v4 that commonly cause test failures and runtime errors:

### `subscribeWithSelector` — Separate Middleware Import

In v5, `subscribeWithSelector` is no longer built into `create` — import it separately:

```ts
// ❌ v4 pattern — broken in v5
import { create } from 'zustand'

const useStore = create((set) => ({
  count: 0,
  inc: () => set(s => ({ count: s.count + 1 })),
}))
useStore.subscribe((state) => { /* selector — broken: selector required */ })

// ✅ v5 pattern — import the middleware
import { create } from 'zustand'
import { subscribeWithSelector } from 'zustand/middleware'

const useStore = create(
  subscribeWithSelector((set) => ({
    count: 0,
    inc: () => set(s => ({ count: s.count + 1 })),
  }))
)
useStore.subscribe((state) => { /* selector — now works */ })
```

### Testing: No Auto-`act()` Wrapping

v4 auto-wrapped state updates in React Testing Library's `act()` during tests. v5 removes this — you must wrap updates manually:

```ts
// ❌ v4 pattern — act() was auto-wrapped (broken in v5 tests)
await user.click(button)
expect(screen.getByText('1')).toBeInTheDocument()

// ✅ v5 pattern — wrap updates in act() manually
import { act } from 'react-dom/test-utils'

await act(async () => { user.click(button) })
expect(screen.getByText('1')).toBeInTheDocument()
```

See `testing.md` for full Zustand v5 testing patterns.

### `combine` — Explicit Type Required

```ts
// ❌ v4 — implicit type (broken in v5)
import { combine } from 'zustand/middleware'
const useStore = create(
  combine({ count: 0 }, (set) => ({ inc: () => set(s => ({ count: s.count + 1 })) }))
)

// ✅ v5 — explicit type parameter
const useStore = create(
  combine({ count: 0 } as { count: number }, (set) => ({
    inc: () => set(s => ({ count: s.count + 1 })),
  }))
)
```

## Async Actions in Zustand — Loading/Error State

For async operations managed in Zustand (e.g., polling, WebSocket updates, local-only mutations):

```tsx
interface AsyncStore {
  user: User | null
  isLoading: boolean
  error: string | null
  fetchUser: (id: string) => Promise<void>
}

export const useUserStore = create<AsyncStore>((set) => ({
  user: null,
  isLoading: false,
  error: null,

  fetchUser: async (id) => {
    set({ isLoading: true, error: null })
    try {
      const user = await api.getUser(id)
      set({ user, isLoading: false })
    } catch (err) {
      set({ error: (err as Error).message, isLoading: false })
    }
  },
}))
```

**When to use Zustand async vs React Query:**
- **Zustand async** — data tied to a UI action; one subscriber owns the fetch lifecycle
- **React Query** — cached, shared server data with automatic refetching, background sync, pagination

## Resetting Zustand State

Reset to initial state — useful for auth logout, form clear, wizard restart:

```tsx
interface Store {
  count: number
  step: number
  data: Partial<FormData>
  reset: () => void
}

const initialState = {
  count: 0,
  step: 1,
  data: {},
}

export const useStore = create<Store>((set) => ({
  ...initialState,

  reset: () => set(initialState),
}))
```

**Use in logout flow:**

```tsx
// On sign-out, reset all client stores
async function signOut() {
  await nextAuthSignOut()
  useUIStore.getState().reset()
  useCartStore.getState().reset()
  router.push('/login')
}
```

## Common Mistakes

- **Using useState for server data** — use React Query instead
- **Not invalidating queries after mutations** — always `queryClient.invalidateQueries` after writes
- **Zustand selectors returning large objects** — select only what you need
- **Not handling loading/error states** — always check `isLoading`/`isError`/`isPending`
- **Overfetching with `enabled: false`** — remember to enable queries when conditions change
- **Mutations inside render** — mutations are side effects, always in event handlers or `useEffect`
- **v4 `subscribeWithSelector` in v5** — import from `zustand/middleware` explicitly in v5
- **v4 test auto-act in v5** — wrap state updates in `act()` manually; v5 removed auto-wrapping
- **Mutating state directly** — always use `set(state => ({ ... }))` or Immer middleware
