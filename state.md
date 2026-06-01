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
- `useOptimistic` — best for simple like/subscribe/toggle patterns where you want instant feedback
- `useMutation` optimistic pattern — better for complex updates that need full rollback control, or when you need to optimistically add/remove items from a list

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

## Common Mistakes

- **Using useState for server data** — use React Query instead
- **Not invalidating queries after mutations** — always `queryClient.invalidateQueries` after writes
- **Zustand selectors returning large objects** — select only what you need
- **Not handling loading/error states** — always check `isLoading`/`isError`/`isPending`
- **Overfetching with `enabled: false`** — remember to enable queries when conditions change
- **Mutations inside render** — mutations are side effects, always in event handlers or `useEffect`
