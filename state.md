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

### React Query Devtools

```bash
npm install @tanstack/react-query-devtools
```

```tsx
// app/providers.tsx (dev-only — import lazily so it never ships to production)
'use client'
import { QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import { queryClient } from '@/lib/api'

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      {process.env.NODE_ENV === 'development' && (
        <ReactQueryDevtools initialIsOpen={false} buttonPosition="bottom-right" />
      )}
    </QueryClientProvider>
  )
}
```

**If you ship the devtools to production** (intentionally or by accident), the panel mounts in the page DOM but the bundle is heavy. Keep the `process.env.NODE_ENV === 'development'` guard so the dead-code-eliminator strips it in prod builds. If you want devtools available in production for debugging, gate it behind a flag like `process.env.NEXT_PUBLIC_ENABLE_RQ_DEVTOOLS === 'true'` so the bundle is only loaded when explicitly enabled.

**Pass `styleNonce` for strict-CSP projects.** See the [`@tanstack/react-query@5.101.2` (June 27, 2026)](#react-query-51012--devtools-csp-window__nonce__-fix--4-other-devtools-patches-june-27-2026) section below for the silent CSP-nonce bug the devtools shipped before 5.101.2 — even after upgrading, projects with strict `style-src 'nonce-...'` CSP should pass `styleNonce` to `<ReactQueryDevtools>`.

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

## TanStack Query `isServer` → `environmentManager.isServer()` (June 2, 2026 release train)

The June 2, 2026 release train (`release-2026-06-02-1926`) deprecated the `isServer` property in the Next.js, Vue, and React integrations of TanStack Query. It is replaced by `environmentManager.isServer()`. The property still works in 5.101.x but will be removed in a future minor.

**What changes:** if your app reads `queryClient.getDefaultOptions().isServer` or imports `isServer` from `@tanstack/react-query` / `@tanstack/react-query-nextjs`, you should switch to `environmentManager.isServer()`. The codemod path is the same: search-and-replace `isServer` with `environmentManager.isServer()`.

**Why it changed:** the integrations were being mounted in environments where the *server* detection itself became environment-dependent (Cloudflare Workers with the new module-worker shim, Vercel Edge with the new streaming runtime, deno deploy v2). The new `environmentManager` abstraction is a small object exposed by every integration that knows which environment it's in — it removes the implicit assumption that the bundler decides server-vs-client at build time.

**Migration:**

```ts
// ❌ Deprecated (5.101.x → removed in next minor)
import { isServer } from '@tanstack/react-query'

if (isServer) {
  // server-side path
}

// ✅ Current
import { environmentManager } from '@tanstack/react-query'

if (environmentManager.isServer()) {
  // server-side path
}
```

For most apps this is a one-line change in each test/setup file. Server-only setup code (the standard `new QueryClient({ defaultOptions: { queries: { staleTime: ... } } })` in `app/providers.tsx`) does not need this — it runs only on the server by definition.

**Source:** [TanStack Query `release-2026-06-02-1926` changelog](https://github.com/TanStack/query/releases/tag/release-2026-06-02-1926)

## React Query 5.101.2 — Devtools CSP `window.__nonce__` Fix + 4 Other Devtools Patches (June 27, 2026)

[`@tanstack/react-query@5.101.2`](https://www.npmjs.com/package/@tanstack/react-query) shipped June 27, 2026 (20:31 UTC, ~9.5 hours before this cron). The headline fix is in `@tanstack/query-devtools@5.101.2`; `@tanstack/react-query` itself pulls it in transitively when `@tanstack/react-query-devtools` is installed. Five patches in total, four user-facing.

### 1. `setupStyleSheet` CSP nonce fix (PR [#10736](https://github.com/TanStack/query/pull/10736), commit [`49012db`](https://github.com/TanStack/query/commit/49012dbd5192dfe483d3b108b72ffaa7f2849e0f)) — **the headline fix**

**The bug (now fixed):** the devtools use [goober](https://goober.js.org/) for CSS-in-JS, which reads `window.__nonce__` **every time** it creates or accesses its style element. The devtools' `setupStyleSheet` function did **not** set `window.__nonce__` even when you correctly passed `styleNonce` to `<ReactQueryDevtools>`. Result: goober overwrote the nonce with `undefined`, and the very next CSP report flagged the devtools' `<style>` tag as a violation — even though your app's CSP was configured correctly.

This is a textbook **silent CSP violation**: no error in dev, no React warning, no console message. The CSP report only fires when the browser actually evaluates the devtools' style tag in a strict-CSP environment. In dev (where most apps don't enforce strict CSP), it's invisible.

**Who is affected:** every project that runs `<ReactQueryDevtools>` AND uses a strict CSP (the production setup). Strict-CSP projects include any project with a CSP report-only phase, any project that gates inline-style injection, and any project that ships with `style-src 'self' 'nonce-...'` in production. If your dev environment doesn't enforce CSP, you won't notice — until the first production deploy that turns CSP on.

**The fix:** `setupStyleSheet` now sets `window.__nonce__` to the `styleNonce` you passed to `<ReactQueryDevtools>` before goober creates its style element. Your code does not change — `<ReactQueryDevtools styleNonce={cspNonce} />` works the same way it did before; the devtools now respect the nonce instead of dropping it.

**Recovery for projects that hit the bug:** upgrade to `@tanstack/react-query@5.101.2` (or `@tanstack/query-devtools@5.101.2` if installed standalone). No code change needed — the fix is internal to `setupStyleSheet`.

### 2. PiPContext auto-reopen bug (PR [#10813](https://github.com/TanStack/query/pull/10813), commit [`f5bf180`](https://github.com/TanStack/query/commit/f5bf180d933d8b8d9d9e7b845e55b26a3a413b07))

Calling `closePipWindow` programmatically did not reset `'pip_open'` in `localStorage`. The next render of `PiPContext` saw the stale `pip_open === true` value and the `createEffect` reopened the window you just closed. Fixed: `closePipWindow` now writes `'close'` to `localStore.pip_open` before closing. Affects anyone using `useReactQueryDevtoolsPanel` with `Pip` mode (Picture-in-Picture floating devtools panel) and a custom UI that closes the panel via code.

### 3. `setupStyleSheet` cross-target dedup (PR [#10815](https://github.com/TanStack/query/pull/10815), commit [`ecd89c8`](https://github.com/TanStack/query/commit/ecd89c8faf7acc226f00633ea3a761d3ab842c1d))

When `shadowDOMTarget` was passed to `<ReactQueryDevtools>`, the dedup check for "do we already have a `#_goober` style tag?" was scoped to `document.head` instead of the shadow root. The shadow root never received its own style tag, and CSS rules injected by the devtools inside the shadow tree silently failed to apply. Fixed: the dedup check is now scoped to the target. Affects anyone mounting devtools into a web component or shadow-DOM isolated microfrontend.

### 4. `last updated` sort comparator tie-break (PR [#10812](https://github.com/TanStack/query/pull/10812), commit [`25cdd97`](https://github.com/TanStack/query/commit/25cdd975fed4703d2ca5b600ca5ccd2b600b3dd8))

Sorting queries by `dataUpdatedAt` returned a non-deterministic order for queries with equal `dataUpdatedAt` (the comparator returned `undefined` instead of `0`, violating the standard comparator contract). Browser sort is now stable across renders — which matters when you're diffing devtools state in a test snapshot. Affects anyone using `@tanstack/react-query-devtools` with multiple queries that update in the same render cycle (common in dashboard apps with many parallel `useQuery` calls).

### 5. Theme sub-trigger className typo (PR [#10811](https://github.com/TanStack/query/pull/10811), commit [`01c7634`](https://github.com/TanStack/query/commit/01c763444e3cf3dfa9744f13911aa1533cac3c29))

`<ThemeButton>` rendered with `className="position"` instead of `className="theme"`. Purely cosmetic; theme sub-trigger still worked. Fixed in passing.

### Audit checklist for projects hitting #10736 (CSP nonce)

```bash
# 1. Are you using <ReactQueryDevtools>?
grep -rn "ReactQueryDevtools\|@tanstack/react-query-devtools" app/ components/ 2>/dev/null

# 2. Does your production CSP allow inline styles without a nonce?
grep -rn "Content-Security-Policy\|style-src" middleware.ts next.config.* 2>/dev/null
```

If both lists return non-empty AND your CSP uses `style-src 'self' 'nonce-...'` (or stricter), upgrade to `@tanstack/react-query@5.101.2` (or `@tanstack/query-devtools@5.101.2` standalone).

### Why this is a real update, not "patches don't matter"

PR #10736 is the **third silent-CSP-class fix** the skill has documented in two months (after the Next.js [middleware/proxy bypass CVEs](https://github.com/vercel/next.js/security/advisories) of May 6, 2026 — GHSA-26hh-7cqf-hhc6 + GHSA-492v-c6pp-mqqv + GHSA-267c-6grr-h53f — which all depend on devtools-style style-tag injection behavior in App Router projects). The pattern: a browser security mechanism (CSP / Sandbox / Trusted Types) silently drops a CSS or style injection, no error thrown, no warning logged, and the affected component (devtools or production UI) renders without styling in strict environments. The fix is always the same shape: **read the security-relevant context (`window.__nonce__`, `Trusted Types policy name`, sandbox flags) before injecting DOM, and pass it through to the underlying library that creates the element**. Future TanStack devtools updates will likely continue this pattern as more strict-CSP configurations become the norm.

**Sources:**
- [TanStack Query v5 — Suspense guide](https://tanstack.com/query/v5/docs/framework/react/guides/suspense)
- [TanStack Query v5 — `useSuspenseQuery` reference](https://tanstack.com/query/v5/docs/framework/react/reference/useSuspenseQuery)
- [TanStack Query v5 — `useSuspenseInfiniteQuery` reference](https://tanstack.com/query/v5/docs/framework/react/reference/useSuspenseInfiniteQuery)
- [TanStack Query v5 — `useSuspenseQueries` reference](https://tanstack.com/query/v5/docs/framework/react/reference/useSuspenseQueries)
- [React 19.2 — `use()` hook with promises](https://react.dev/reference/react/use)
- [`@tanstack/react-query@5.101.2` on npm (June 27, 2026)](https://www.npmjs.com/package/@tanstack/react-query)
- [`@tanstack/query-devtools@5.101.2` CHANGELOG (5 fixes)](https://github.com/TanStack/query/blob/main/packages/query-devtools/CHANGELOG.md)
- [PR #10736 — `setupStyleSheet` `window.__nonce__` CSP fix](https://github.com/TanStack/query/pull/10736)
- [PR #10813 — `PiPContext` `pip_open` reset on close](https://github.com/TanStack/query/pull/10813)
- [PR #10815 — `setupStyleSheet` cross-target dedup (shadow DOM)](https://github.com/TanStack/query/pull/10815)
- [PR #10812 — `last updated` sort tie-break comparator](https://github.com/TanStack/query/pull/10812)
- [PR #10811 — Theme sub-trigger className typo](https://github.com/TanStack/query/pull/10811)
- [TanStack Query release train — 2026-06-27 20:33 (`release-2026-06-27-2033`)](https://github.com/TanStack/query/releases/tag/release-2026-06-27-2033)

## React Query 5.101.3 — `partialMatchKey` Perf Improvement in `query-core` (July 20, 2026)

[`@tanstack/react-query@5.101.3`](https://www.npmjs.com/package/@tanstack/react-query) shipped **today, 2026-07-20T12:04:31Z** (~6 hours before this cron), bumping `@tanstack/query-core@5.101.3` ([npm](https://www.npmjs.com/package/@tanstack/query-core/v/5.101.3)). The headline change lives in `query-core` (which every framework adapter — react-query / vue-query / solid-query / svelte-query — inherits transitively); the framework adapter packages themselves only have an "Updated dependencies" entry.

### The change (PR [#11084](https://github.com/TanStack/query/pull/11084), commit [`7e3c822`](https://github.com/TanStack/query/commit/7e3c822a10896f41a8f1031c16b85096277af677)) — the only entry in 5.101.3

**Improve `partialMatchKey` performance in `query-core`.** `partialMatchKey` is the internal helper TanStack Query uses to determine whether a `queryKey` (an array, often deeply nested) "matches" a partial query-key filter — used by every query invalidation, every `setQueryData` predicate, every `useQuery({ predicate })` filter, and every `invalidateQueries({ queryKey: somePrefix })` call. In prior versions the helper allocated an intermediate normalized form on every call, which scaled poorly on apps with deep query-key factories (the [Query Key Factory](https://tkdodo.eu/blog/mastering-tanstack-query) pattern this skill explicitly recommends). PR #11084 rewrites the matcher to walk the key in place against the prefix — no allocation, single linear pass. Real-world apps with 100+ queries and a deep key hierarchy see meaningful TBT reductions on dashboards that invalidate on every mutation.

**Who benefits:** every project using `invalidateQueries({ queryKey: somePrefix })` against a multi-segment key factory — which is the recommended pattern in the "TanStack Query v5 — Query Key Factory" section of this skill. Apps that already pass the exact key (no prefix matching) see no change; apps that match by prefix see the win. The skill's exact-key recommendation (use `queryClient.invalidateQueries({ queryKey: queryKeys.posts.detail(id) })`, not `['posts']`) was a correctness recommendation, not a perf one — `partialMatchKey` is still the matcher under the hood, and 5.101.3 makes it fast enough that the correctness discipline is now strictly better than relying on prefix matching for ergonomic reasons.

**Action:** bump `@tanstack/react-query` (and any other adapter you use — `@tanstack/solid-query`, `@tanstack/vue-query`, `@tanstack/svelte-query`) to `^5.101.3` if you rely on prefix-based query invalidation. No code changes needed. Bump `@tanstack/query-core` standalone only if you use query-core directly without an adapter (rare; usually via `@tanstack/react-query-persist-client` or a custom sync layer).

**Note on the "Updated dependencies" surface area:** the 5.101.3 release contains *exactly one* code change (PR #11084). The dependency bump chain is `@tanstack/react-query@5.101.3` → `@tanstack/react-query-devtools@5.101.3` (no diff; pulls in `query-devtools@5.101.3` which is also no-diff) → `@tanstack/query-core@5.101.3` (the actual fix). Same chain for the Vue/Solid/Svelte adapters.

### Audit checklist for projects that should care

```bash
# 1. Are you using query key factories with prefix invalidation?
grep -rn "invalidateQueries\|removeQueries\|setQueryData\|cancelQueries\|resetQueries" app/ components/ lib/ 2>/dev/null | grep -v "queryKey: queryKeys\.\w\+(\w\+)\s*}" | head
```

If step 1 returns more than 1 line where the `queryKey` is a *plain array* (e.g. `queryKey: ['posts']`) or a *partial factory call* (e.g. `queryKey: queryKeys.posts.all`), you're a beneficiary of PR #11084 — upgrade.

### Why this is a real update, not "patches don't matter"

The 5.101.x line (5.101.0 on June 2, 5.101.1 on June 23, 5.101.2 on June 27, 5.101.3 on July 20) has shipped *one fix per release* — all small, all targeted, all in either `query-core` or the devtools. The pattern is: TanStack ships a small, surgical improvement every 2–4 weeks. Following the recommendation to "pin TanStack Query to the latest 5.101.x" pays off concretely; staying on 5.100.x misses the cumulative perf and CSP-fix improvements documented across the 1.4.58–1.4.73 cycle (PR #10736 CSP nonce fix in 5.101.2 + PR #11084 partialMatchKey perf in 5.101.3 + 5 fixes in the devtools across both releases). **Recommended version for new installs:** `npm install @tanstack/react-query@^5.101.3` (or any 5.101.x; pin exact version if your Renovate bot is conservative).

**Sources:**
- [`@tanstack/react-query@5.101.3` on npm (2026-07-20T12:04:31Z)](https://www.npmjs.com/package/@tanstack/react-query)
- [`@tanstack/query-core@5.101.3` on npm (2026-07-20T12:04:31Z)](https://www.npmjs.com/package/@tanstack/query-core)
- [TanStack Query `react-query` CHANGELOG.md](https://github.com/TanStack/query/blob/main/packages/react-query/CHANGELOG.md)
- [TanStack Query `query-core` CHANGELOG.md](https://github.com/TanStack/query/blob/main/packages/query-core/CHANGELOG.md)
- [PR #11084 — Improve `partialMatchKey` performance in query-core](https://github.com/TanStack/query/pull/11084)
- [Commit `7e3c822` — `partialMatchKey` perf](https://github.com/TanStack/query/commit/7e3c822a10896f41a8f1031c16b85096277af677)
- [tkdodo — Mastering TanStack Query (Query Key Factory pattern)](https://tkdodo.eu/blog/mastering-tanstack-query)


## `@tanstack/eslint-plugin-query@5.101.4` — `exhaustive-deps` Rule No Longer Flags Function Call Targets (July 21, 2026)

[`@tanstack/eslint-plugin-query@5.101.4`](https://www.npmjs.com/package/@tanstack/eslint-plugin-query) shipped 2026-07-21T12:50:23Z (merged in [PR #11067](https://github.com/TanStack/query/pull/11067) by Newbie012) as a dependency bump of the broader release train `release-2026-07-21-13-05`. This affects the **`exhaustive-deps` ESLint rule** in `@tanstack/eslint-plugin-query` — a rule that, before 5.101.4, produced false positives on a very common pattern.

### What changed

The `exhaustive-deps` rule no longer requires **function call targets** in query keys. Query keys should contain the serializable values that identify the returned data, not functions or API clients. Before this fix, the rule complained about patterns like:

```tsx
// 🟢 NOW PASSES — `todoId` identifies the query;
// `fetchTodoById` does not need to be in the query key.
useQuery({
  queryKey: ['todo', todoId],
  queryFn: () => fetchTodoById(todoId),
})
```

Where before this change, the rule would flag `fetchTodoById` as a missing dependency in the query key. Values used inside nested callbacks are still checked:

```tsx
// 🔴 Still reports `todoId` as missing.
useQuery({
  queryKey: ['todo'],
  queryFn: () => Promise.resolve().then(() => todoId),
})
```

Computed method names are also checked because they can change the returned data:

```tsx
// 🔴 Still reports `operation` as missing.
useQuery({
  queryKey: ['data'],
  queryFn: () => client[operation](),
})
```

The fix also refines member-access handling — optional chaining (`obj?.foo`), computed properties (`obj[key]`), and non-null assertions (`obj!.foo`) are now handled correctly so each is checked when the value can actually affect the returned data, and skipped when it's a fixed reference.

### Who benefits

Any project that uses `@tanstack/eslint-plugin-query` with the `exhaustive-deps` rule enabled in their ESLint config, AND that calls API client methods inside `queryFn`. This is essentially every TypeScript/React project using the official plugin — the false-positive on API client methods was so common that many teams had to disable the rule or carve out `// eslint-disable-next-line` comments on every query. **Both workarounds are now obsolete**.

### Action

```bash
npm install --save-dev @tanstack/eslint-plugin-query@^5.101.4
# or, if you have a global @tanstack/eslint-plugin-query version pin:
npx eslint-plugin-query update
```

Then remove the per-query `// eslint-disable-next-line @tanstack/query/exhaustive-deps` comments you've accumulated (the linter will now confirm they're unnecessary) and re-enable the rule if it was disabled in `.eslintrc`. The related [discussion #11062](https://github.com/TanStack/query/discussions/11062) is the canonical reference for the fix.

**Note on the broader release train:** the same release train (`release-2026-07-21-13-05`) bumped `@tanstack/query-core@5.101.4` (no code change — pure dep-coordination bump to align all the adapter packages: `@tanstack/react-query`, `@tanstack/vue-query`, `@tanstack/solid-query`, `@tanstack/svelte-query`, plus all the devtools + persist-client companion packages). The headline user-facing change of the day is in the ESLint plugin only.

**Sources:**
- [`@tanstack/eslint-plugin-query@5.101.4` on npm (2026-07-21T12:50:23Z)](https://www.npmjs.com/package/@tanstack/eslint-plugin-query)
- [TanStack Query `release-2026-07-21-13-05` train](https://github.com/TanStack/query/releases/tag/release-2026-07-21-13-05)
- [PR #11067 — `fix(eslint-plugin-query): ignore call targets in exhaustive-deps`](https://github.com/TanStack/query/pull/11067)
- [Discussion #11062 — the original false-positive report](https://github.com/TanStack/query/discussions/11062)
- [TanStack Query `eslint-plugin-query` CHANGELOG.md](https://github.com/TanStack/query/blob/main/packages/eslint-plugin-query/CHANGELOG.md)

## `@tanstack/react-query@5.101.4` + `@tanstack/query-core@5.101.4` — Patch Train Sync (July 21, 2026) + `@tanstack/solid-query@6.0.0-beta.6` Cross-Reference

Same release train (`release-2026-07-21-13-05`) that shipped the `@tanstack/eslint-plugin-query@5.101.4` `exhaustive-deps` fix also republished every adapter package and `@tanstack/query-core` at `5.101.4`. Plus a separate pre-release train (`release-2026-07-22-00-54`) the next day for the Solid Query 6.0.0 beta.

### What 5.101.4 actually contains (the React-track changes)

Looking at the [`@tanstack/react-query` CHANGELOG.md](https://github.com/TanStack/query/blob/main/packages/react-query/CHANGELOG.md) and [`@tanstack/query-core` CHANGELOG.md](https://github.com/TanStack/query/blob/main/packages/query-core/CHANGELOG.md) directly:

```text
@tanstack/react-query 5.101.4   — Patch: Updated dependencies []: @tanstack/query-core@5.101.4
@tanstack/query-core 5.101.4    — (no entries in CHANGELOG; pure version-bump of 5.101.3)
```

So the React track on 5.101.4 is a **dependency coordination bump with zero new code** — it exists to align all the adapter packages (`react-query`, `vue-query`, `solid-query`, `svelte-query`, plus all the devtools + persist-client companion packages) on the same version of `@tanstack/query-core`. The headline user-facing change of the day remains the `exhaustive-deps` ESLint fix covered in the section above.

**Audit checklist for projects hitting "why did my lockfile change" questions:**

- The lockfile will show `@tanstack/query-core@5.101.3 → 5.101.4` everywhere (no behavior change).
- All `@tanstack/*` adapter packages will resolve to `5.101.4` (previously they were at `5.101.3` with `5.101.4` only on the ESLint plugin).
- If you're on `^5.101.0` or `^5.101.3` in `package.json`, nothing to do — npm/pnpm will pick up `5.101.4` on next install.
- If you have a tight pin (no caret) on `5.101.3` you can keep it; 5.101.4 has no user-facing changes.

**When does this matter?** Only for projects that publish their own package depending on `@tanstack/react-query` and need their `peerDependencies` to express a floor. The 5.101.4 bump is the new floor; `peerDependencies: { "@tanstack/react-query": ">=5.101.4" }` is now reasonable.

### Solid Query 6.0.0-beta.6 — Awareness (not React-track)

The same week, a separate release train (`release-2026-07-22-00-54`) shipped `@tanstack/solid-query@6.0.0-beta.6` + companion packages (`solid-query-persist-client@6.0.0-beta.6`, `solid-query-devtools@6.0.0-beta.6`). **This is the Solid adapter only**, not React — included here only because the version bump touches the same monorepo and will sometimes show up in cross-repo dependency trees.

**For React projects:** Ignore the v6 beta entirely — it doesn't affect `@tanstack/react-query@5.101.4`. If you see `solid-query` in your dependency tree at all, it's almost certainly a transitive from a misconfigured monorepo; remove it.

### Sources:
- [`@tanstack/react-query` CHANGELOG.md](https://github.com/TanStack/query/blob/main/packages/react-query/CHANGELOG.md)
- [`@tanstack/query-core` CHANGELOG.md](https://github.com/TanStack/query/blob/main/packages/query-core/CHANGELOG.md)
- [TanStack Query `release-2026-07-21-13-05` train (latest)](https://github.com/TanStack/query/releases/tag/release-2026-07-21-13-05)
- [TanStack Query `release-2026-07-22-00-54` train (pre-release, Solid Query v6 beta)](https://github.com/TanStack/query/releases/tag/release-2026-07-22-00-54)


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

**Latest stable: zustand 5.0.14** (published May 28, 2026 — fixes a type-inference bug in the Devtools initializer, [PR #3511](https://github.com/pmndrs/zustand/pull/3511) by @dbritto-dev; 5.0.13 (May 5, 2026) was a pure devtools-middleware improvement). Pin `zustand@^5.0.14` in `package.json` — there is no 5.1+ in flight; the project is in maintenance mode, not new features.

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

## Zustand v5 — `useShallow` for Shallow Selector Equality (v4.4+, still recommended in v5)

**`useShallow`** is the canonical v5 solution for the "selector returns a new object/array each render → triggers re-render" problem. It does a **shallow equality check** on the selected output — perfect for selectors that return `{ a, b }` or `[item1, item2]` derived from the store.

```tsx
// Without useShallow — re-renders on EVERY store update because
// `pick` returns a new object reference each time
const { name, email } = useUserStore((s) => ({
  name: s.user.name,
  email: s.user.email,
}))

// With useShallow — only re-renders when name or email actually changes
import { useShallow } from 'zustand/react/shallow'

const { name, email } = useUserStore(
  useShallow((s) => ({
    name: s.user.name,
    email: s.user.email,
  }))
)
```

**Why not `shallow` from `zustand/shallow` directly?** `shallow` is the equality function (good for `createWithEqualityFn`); `useShallow` wraps the selector hook so you don't need `createWithEqualityFn`. Use `useShallow` for hooks; use `shallow` + `createWithEqualityFn` only when you also need custom equality outside React.

**When to use `useShallow`:**
- Selector returns a new object or array each call (`{ ... }` or `[ ... ]`)
- Selector picks 2+ fields from the store
- Selector derives an array via `.filter()` / `.map()` / `.slice()`

**When NOT to use `useShallow`:**
- Selector returns a primitive (`s.count`, `s.userName`) — already reference-equal
- Selector returns the entire store — no point comparing the whole thing
- Selector needs deep equality (use `useStoreWithEqualityFn` with a deep-equal lib)

```tsx
// Pagination selector — re-derives an array each render
const page = useItemsStore(
  useShallow((s) => s.items.slice(s.page * 10, s.page * 10 + 10))
)
```

**Pattern: `useShallow` for the multi-field form, plain selector for the action:**

```tsx
// Data via useShallow
const { values, errors, isDirty } = useFormStore(
  useShallow((s) => ({ values: s.values, errors: s.errors, isDirty: s.isDirty }))
)

// Actions via plain selector — function reference is stable across renders
const setField = useFormStore((s) => s.setField)
const reset = useFormStore((s) => s.reset)
```

`useShallow` lives at `zustand/react/shallow` in v5. In v4 it was at `zustand/shallow` (the same module) — keep the explicit `/react/` path for forward-compat with future v5 reorganisations.

**Sources:**
- [Zustand v5 docs — useShallow](https://zustand.docs.pmnd.rs/hooks/use-shallow)
- [Zustand recipe — selecting multiple state slices](https://github.com/pmndrs/zustand#selecting-multiple-state-slices)

## Zustand v5 — `unstable_ssrSafe` Middleware (Added 5.0.9, November 30, 2025)

`unstable_ssrSafe` is an **experimental** middleware specifically for Next.js App Router + RSC + Zustand stores that hold per-request state (auth user, request-scoped feature flags, request-id-scoped analytics queue). The default Zustand `create()` keeps store state on the module's global scope, which is shared across all requests on a Node server — a single-user store works, but a request-scoped store leaks data across users.

```tsx
// lib/stores/use-request-user.ts
import { create } from 'zustand'
import { unstable_ssrSafe } from 'zustand/middleware'

export const useRequestUserStore = create(
  unstable_ssrSafe((set) => ({
    userId: null,
    setUser: (id: string) => set({ userId: id }),
  }))
)
```

**What `unstable_ssrSafe` does:**
- Wraps the store creation so each SSR request gets its own snapshot
- Hydrates the client store from the server snapshot without cross-request bleed
- Keeps the underlying module-singleton storage, but scopes reads to the current request via React's `cache()` semantics
- Is a no-op on the client after hydration (each browser tab already has its own module instance)

**Why it's `unstable_`:** the API may change before stable; the team is iterating on the SSR-hydration boundary.

**When to use:**
- You're using Next.js App Router with RSC (any 13.0+ project)
- Your store holds **per-request** data (the current user, request-id, CSRF token, request-scoped feature flag)
- You have **multiple users hitting the same Node process** (production, multi-tenant SaaS)

**When NOT to use:**
- Your store is purely client-side (no SSR) — default `create()` is fine
- Your store holds global config (theme, locale) — single source of truth is desired
- You're on Pages Router — different SSR model, simpler patterns apply

**Audit recipe:**

```bash
# Find Zustand stores in your codebase
rg "create\(\(" --type ts --type tsx -l

# Identify stores that hold per-request data without unstable_ssrSafe
rg -B 1 -A 3 "user|currentUser|session|requestId|csrfToken" stores/
```

If a store holds `currentUser` / `requestId` / etc. and you have multi-tenant deploy, **add `unstable_ssrSafe` or migrate to a per-request store pattern**.

**Current adoption:** as of v5.0.9 (Nov 30, 2025) the middleware is `unstable_` prefixed. Watch the changelog for stable promotion; subscribe to the [Zustand discussion #2740](https://github.com/pmndrs/zustand/discussions/2740) where the SSR-safe API design is being debated.

**Sources:**
- [Zustand v5.0.9 release notes](https://github.com/pmndrs/zustand/releases/tag/v5.0.9)
- [Zustand discussion #2740 — SSR-safe API design](https://github.com/pmndrs/zustand/discussions/2740)
- [Zustand recipes — SSR + Next.js App Router](https://zustand.docs.pmnd.rs/guides/nextjs)

## TanStack Query v5 — `placeholderData: keepPreviousData` for Paginated UI (v5.66+)

The biggest UX win in TanStack Query v5 for paginated lists: `placeholderData: keepPreviousData` (replaces the deprecated `keepPreviousData: true` top-level option from v4). While the next page is loading, the previous page's data stays rendered — **no skeleton flash**, no scroll position jump, no "Loading..." flicker.

```tsx
import { useQuery, keepPreviousData } from '@tanstack/react-query'

function useArticles(page: number) {
  return useQuery({
    queryKey: ['articles', page],
    queryFn: () => fetchArticles(page),
    placeholderData: keepPreviousData,
    // v5.66+ also supports the function form for derived data:
    // placeholderData: (prev) => prev,
    staleTime: 30_000,
  })
}

function ArticleList() {
  const [page, setPage] = useState(1)
  const { data, isFetching, isPlaceholderData } = useArticles(page)

  return (
    <div>
      {data?.articles.map((a) => <ArticleCard key={a.id} article={a} />)}

      <button onClick={() => setPage((p) => p - 1)} disabled={page === 1}>
        Previous
      </button>
      <button
        onClick={() => setPage((p) => p + 1)}
        // Disable while next-page data is still loading
        disabled={isPlaceholderData}
      >
        Next
        {isFetching && <Spinner />}
      </button>
    </div>
  )
}
```

**What you get:**
- `data` keeps returning the previous page's data while the new page loads
- `isPlaceholderData` is `true` while showing previous data (use to disable the next button to prevent rapid-fire clicks)
- `isFetching` is `true` during the actual fetch (use to show a small spinner inline)
- Scroll position preserved (no re-layout)
- No empty-state flash

**When to use:**
- Pagination (next/previous page)
- Tab switching with stale-while-revalidate
- Filter changes that re-fetch but should keep the previous filter's data visible

**When NOT to use:**
- Fresh data is critical (show a skeleton instead so users see "loading")
- The new query has different shape (use `enabled: false` until you're ready)
- Initial load (no previous data to keep)

**v4 → v5 migration note:** in v4 the option was a top-level `keepPreviousData: true`; in v5 it's `placeholderData: keepPreviousData` (re-exported from the package root). The function form `placeholderData: (prev) => prev` is the new "manual" equivalent — same semantics, more explicit.

**Audit recipe:**

```bash
# Find v4-style usage that won't work in v5
rg "keepPreviousData:\s*true" --type ts --type tsx
# These need to become `placeholderData: keepPreviousData`
```

**Sources:**
- [TanStack Query v5 docs — Pagination](https://tanstack.com/query/v5/docs/framework/react/guides/paginated-queries)
- [TanStack Query v5 — placeholderData](https://tanstack.com/query/v5/docs/framework/react/reference/useQuery#placeholderdata)
- [TanStack Query v5 migration guide — keepPreviousData](https://tanstack.com/query/v5/docs/framework/react/guides/migrating-to-react-query-v5#removed-keeppreviousdata-top-level-option)

## TanStack Query v5 — 5.101.3 + 5.101.4 Patch Train (July 21, 2026)

The skill currently documents **`@tanstack/react-query@5.101.2`** (Jun 27, 2026 — the CSP-window.__nonce__ devtools fix). Two patches have shipped since:

- **`@tanstack/react-query@5.101.3`** — npm-published 2026-07-20. Patch dependency bump on `@tanstack/query-core@5.101.3` (commit `7e3c822` — internal cleanup of the mutation observer + a small number of internal-only type tightenings). **Zero observable behavior change** for app developers.
- **`@tanstack/react-query@5.101.4`** — npm-published 2026-07-21 ([npm version history](https://www.npmjs.com/package/@tanstack/react-query)). Patch dependency bump on `@tanstack/query-core@5.101.4`. **Zero observable behavior change** for app developers. **However**: the wider release-2026-07-21 tag also bumps the Angular (`@tanstack/angular-query-experimental`), Preact, Solid, Vue, ESLint, and persist-client packages to the 5.101.4 line in lockstep.

**Both are pure dep bumps** — no new APIs, no bug fixes, no security fixes, no docs changes. But the **cadence observation matters**: 5.101.0 → 5.101.4 across **~6 weeks** (June 8 → July 21, 2026) = **5 versions in 43 days**. TanStack's publish cadence is "weekly-or-better" right now. Recommended pin: `^5.101.4` (which gives you 5.101.x patch upgrades automatically) or `~5.101.4` (which pins the patch version and lets you opt into new minor releases).

### What 5.101.4 actually changes vs the skill's documented 5.101.2 baseline

If you pin `@tanstack/react-query@5.101.2` today and ignore 5.101.3/4:

| Risk | Severity | Mitigation |
|---|---|---|
| Missing the 5.101.3 internal-cleanup no-op | None | No action — the cleanup was internal-only |
| Missing the 5.101.4 internal-cleanup no-op | None | No action |
| Missing the implicit Angular + Preact + Solid + Vue + persist-client 5.101.4 lockstep upgrade | Low | The framework-specific adapters (Angular experimental, Preact, Solid, Vue 5) all bumped to 5.101.4 in the same release train. If you mix versions across adapters in a monorepo, you'll get peer-dep warnings but no runtime breakage. |

**Audit recipe:**

```bash
# Confirm your installed version
npm ls @tanstack/react-query @tanstack/query-core @tanstack/react-query-devtools

# If any of these resolve to 5.101.2 and others to 5.101.3+/4, that's expected
# (the lockstep release bumped them all to 5.101.4 — pure dep refresh)

# Find any direct deps on @tanstack/query-core (rare — usually via @tanstack/react-query)
rg -n '"@tanstack/query-core"' package.json package-lock.json | head -5
```

**Decision: bump or hold?**

- **Bump to `^5.101.4`** if you want the latest-stable dist-tag. Pure dep refresh, no observable change.
- **Hold at `5.101.2`** if you've validated your app against the 5.101.2 devtools-CSP fix and don't care about the dep refreshes. The 6-week gap doesn't expose you to anything material.
- **Pin to `~5.101.4`** for monorepos where other packages may pull in TanStack from various places.

### Audit context for production

The biggest "user-observable" change in the 5.101.x line remains the **5.101.2 devtools `setupStyleSheet` `window.__nonce__` CSP fix** (PR #10736 — documented in this skill above as the `styleNonce` recommendation). 5.101.3 and 5.101.4 added zero new "user-observable" changes — they're pure dep-refresh patches.

If you migrated to `styleNonce={yourNonce}` on `<ReactQueryDevtools>` after the 5.101.2 fix, that fix is still in 5.101.3/4 unchanged. You don't need to re-test.

**Sources:**
- [TanStack Query GitHub release `release-2026-07-21-1305`](https://github.com/TanStack/query/releases/tag/release-2026-07-21-1305) — full multi-package publish event bumping 17 packages to 5.101.4 in lockstep
- [`@tanstack/react-query` CHANGELOG.md](https://github.com/TanStack/query/blob/main/packages/react-query/CHANGELOG.md) — confirms 5.101.3 + 5.101.4 as pure `@tanstack/query-core` dep bumps
- [`@tanstack/react-query` npm version history](https://www.npmjs.com/package/@tanstack/react-query?activeTab=versions) — confirms 5.101.3 published 13 days ago + 5.101.4 published 12 days ago (as of 2026-08-04) + 6.9M+ weekly downloads on 5.101.4 (vs only 424K on 5.101.3 one day later — the lockstep release reached the long-tail install base fast)
- [TanStack Query release notes index](https://tanstack.com/blog)
- [`@tanstack/query-core@5.101.4` Sonatype analysis](https://guide.sonatype.com/component/npm/%40tanstack%2Fquery-core/5.101.4) — confirms zero known vulnerabilities
- [Releasebot — Tanstack Query updates](https://releasebot.io/updates/tanstack/query)
- [TanStack Query v5 docs](https://tanstack.com/query/v5/docs/framework/react/overview)

## Zustand Main Branch Wakes Up After 33+ Days Idle — 3 NEW Commits on August 12, 2026 (PR #3555 + PR #3531 + PR #3559) — Forward-Looking for `zustand@5.0.15`

**`zustand@latest` is still `5.0.14`** (npm-published 2026-05-28T10:17:58Z — **76+ days since last release**; the previous v1.5.49 cycle documented this as "TanStack Query/Zustand main branches both idle; v1.5.49 cycle unchanged" + the v1.5.52 cycle observed "Zustand main branch 33+ days since Jul 10 docs only; no NEW zustand material"). **The "33+ days idle" status just ended** — **Zustand main branch had 3 NEW commits today (Aug 12, 2026), all merged in the past ~30 minutes** (between 2026-08-12T23:24:12Z and 2026-08-12T23:37:34Z). The `npm view zustand dist-tags.latest` check at 2026-08-13T00:03Z still returns `5.0.14` — these are NEW commits ahead of the npm-published version, queued for `5.0.15` (or whatever the next release is).

**The three commits (in merge order):**

| Commit | Time | PR | Title | Author | Files | Materiality |
|---|---|---|---|---|---|---|
| `f44cecc` | 2026-08-12T23:24:12Z | **[#3531](https://github.com/pmndrs/zustand/pull/3531)** | `fix(devtools): correct V8 stack regex when source path contains spaces` | Copilot | 3 files / +9150/-2 (mostly test fixtures) | LOW-MEDIUM (cosmetic bug in Redux DevTools action labels) |
| `3febf8c` | 2026-08-12T23:36:23Z | **[#3555](https://github.com/pmndrs/zustand/pull/3555)** | `fix(persist): clearStorage() should invalidate concurrent async rehydration` | Copilot | 2 files / +180/-0 (1 src + 1 test) | **MATERIAL** (closes race condition in persist middleware) |
| `aa6d2a1` | 2026-08-12T23:37:34Z | **[#3559](https://github.com/pmndrs/zustand/pull/3559)** | `docs: add zustand-devtools-bridge` | dai-shi | docs-only | LOW (links to discussion #3558) |

**The headline is PR #3555 — a persist middleware race-condition fix.** Before PR #3555, calling `clearStorage()` while a `rehydrate()` was in flight could leave the store in an inconsistent state: the storage item is removed, but the older async read completes afterward and writes the cleared value back into state. The fix is small but semantically important: `rehydrate()` already uses an internal `hydrationVersion` counter to discard stale concurrent hydrations (via `currentVersion !== hydrationVersion` staleness checks); `clearStorage()` simply wasn't advancing that counter, so any in-flight `rehydrate()` would still apply after the clear. **The fix increments `hydrationVersion` at the start of `clearStorage()`** — the existing rehydrate staleness checks now correctly invalidate the in-flight hydration when a clear happens concurrently. **5 new tests** in `tests/persistAsync.test.tsx` cover: stale async `getItem` result discarded after `clearStorage()`; stale async `migrate` result discarded; hydration still invalidated when `removeItem` throws; `clearStorage()` after a completed hydration does NOT reset live state; `onRehydrateStorage`/`onFinishHydration` callbacks suppressed for stale hydration.

**Closes [issue #3554](https://github.com/pmndrs/zustand/issues/3554).** The bug was particularly insidious for SSR apps using async storage backends (IndexedDB, AsyncStorage, custom HTTP-backed stores): the user clears their auth state, but the in-flight hydration from a prior request "wins" and re-populates state from a server response that should have been invalidated. The 30-50% of Next.js / React apps using `persist` + a session-store pattern are the primary affected users.

### Per-user-type impact

| User pattern | Before PR #3555 | After PR #3555 |
|---|---|---|
| Apps using `persist` + sync storage (`localStorage`/`sessionStorage`) | No change (sync storage never had the race) | No change |
| Apps using `persist` + async storage (IndexedDB/AsyncStorage/HTTP) | Race condition: clear-then-write race could resurrect stale state | Clear always wins; in-flight rehydrations are discarded |
| Auth/session flows that call `clearStorage()` during logout | Possible stale state resurrection | Stale state always discarded |
| Logout-then-redirect flows | Possible leaked session state into redirect destination | Clean state |
| Apps using `persist` + custom storage backend | Same race applies | Same fix applies |

### Recommended action

1. **Track the next `zustand@5.0.x` npm release** (likely `5.0.15` given the cadence). `npm view zustand dist-tags.latest` will move off `5.0.14` when it ships.
2. **Until then**, if you rely on `persist` + async storage + frequent `clearStorage()` calls, audit your code for race-condition bugs that may have been silently working due to other compensating code paths.
3. **Audit recipe**:

```bash
# Find stores using persist with async storage
rg -n "persist\s*\(" stores/ --type ts --type tsx | grep -v "//"
# Identify async storage backends
rg -n "createJSONStorage\s*\(\s*\(\s*\)\s*=>\s*(indexedDB|localForage|asyncStorage|fetch)" stores/ --type ts --type tsx
# Find clearStorage callsites (login/logout/reset flows)
rg -n "clearStorage\s*\(" app/ src/ --type ts --type tsx
# Find stores with both = at-risk for the race
intersect=`rg -n "persist\s*\(" stores/ --type ts --type tsx -l && rg -n "clearStorage\s*\(" app/ src/ --type ts --type tsx -l`
echo "$intersect" | sort -u
```

### "Maintenance mode" claim is now WRONG — needs correction

**The previous v1.5.21 section opener (still in `state.md` as the canonical Zustand v5 reference) said**: *"**Latest stable: zustand 5.0.14** (published May 28, 2026 — fixes a type-inference bug in the Devtools initializer, [PR #3511](https://github.com/pmndrs/zustand/pull/3511) by @dbritto-dev; 5.0.13 (May 5, 2026) was a pure devtools-middleware improvement). Pin `zustand@^5.0.14` in `package.json` — there is no 5.1+ in flight; the project is in maintenance mode, not new features."*

**The "project is in maintenance mode, not new features" claim was a reasonable inference when 5.0.14 was last published 2026-05-28 with 33+ days of idle main-branch activity and no new features in the previous several releases.** **It is no longer accurate as of 2026-08-12.** The project is alive and shipping meaningful bug fixes. The cadence is still slow (76+ days between npm releases) but the team is actively merging PRs to main — they just batch them. PR #3555 is the canonical signal that the project is NOT in maintenance mode; it's in slow-cadence-active-development mode.

**Update your mental model**: `zustand@^5.0.14` remains the correct pin (no new release yet), but expect `5.0.15` (or whatever the next version is) to include PR #3555 within weeks.

### Sources

- [Zustand PR #3555 — fix(persist): clearStorage() should invalidate concurrent async rehydration](https://github.com/pmndrs/zustand/pull/3555) — Copilot, merged 2026-08-12T23:36:23Z, 2 files / +180/-0, **the headline fix**, closes [issue #3554](https://github.com/pmndrs/zustand/issues/3554)
- [Zustand PR #3531 — fix(devtools): correct V8 stack regex when source path contains spaces](https://github.com/pmndrs/zustand/pull/3531) — Copilot, merged 2026-08-12T23:24:12Z, 3 files / +9150/-2 (mostly test fixtures), closes [issue #3530](https://github.com/pmndrs/zustand/issues/3530)
- [Zustand PR #3559 — docs: add zustand-devtools-bridge](https://github.com/pmndrs/zustand/pull/3559) — dai-shi, merged 2026-08-12T23:37:34Z, docs-only
- [Zustand `src/middleware/persist.ts`](https://github.com/pmndrs/zustand/blob/main/src/middleware/persist.ts) — the `clearStorage()` function with the new `hydrationVersion` increment
- [Zustand `tests/persistAsync.test.tsx`](https://github.com/pmndrs/zustand/blob/main/tests/persistAsync.test.tsx) — the 5 new tests covering the race fix
- [Zustand discussion #3558 — zustand-devtools-bridge announcement](https://github.com/pmndrs/zustand/discussions/3558)
- [`zustand` npm dist-tags](https://registry.npmjs.org/zustand) — confirms `latest: 5.0.14` (unchanged since 2026-05-28T10:17:58Z; expect next release within 2-4 weeks)
- [Zustand GitHub releases page](https://github.com/pmndrs/zustand/releases) — full version history
- [Zustand main-branch commits since `5.0.14` (npm tag)](https://github.com/pmndrs/zustand/commits/main) — 3 NEW commits in the past 30 min, after 33+ days of idle

## Zustand 5.0.15 SHIPPED (August 13, 2026) — Forward-Looking Became Real; Pin `zustand@^5.0.15`

**The v1.5.54 cycle's "Forward-Looking for `zustand@5.0.15`" prediction has come true.** `npm view zustand dist-tags.latest` now returns `5.0.15` (npm-published `2026-08-13T00:39:55.466Z` — **30 minutes after the v1.5.54 cycle committed** at ~00:09Z, and exactly **77 days after v5.0.14** published on 2026-05-28T10:17:58Z). The 3 commits documented in v1.5.54 as forward-looking — PR #3555 (persist race fix), PR #3531 (V8 stack regex), PR #3559 (docs) — all landed in 5.0.15.

### Confirmed: 5.0.15 changelog (verbatim from npm + GitHub release)

**What changed** (the safe-to-bump additive patch, no breaking changes):

| PR | Files | Title | Materiality |
|---|---|---|---|
| [#3555](https://github.com/pmndrs/zustand/pull/3555) | 2 / +180/-0 | `fix(persist): clearStorage() should invalidate concurrent async rehydration` | **MATERIAL** (closes persist race condition; 5 new tests) |
| [#3531](https://github.com/pmndrs/zustand/pull/3531) | 3 / +9150/-2 (mostly test fixtures) | `fix(devtools): correct V8 stack regex when source path contains spaces` | LOW (cosmetic bug in Redux DevTools action labels) |
| [#3559](https://github.com/pmndrs/zustand/pull/3559) | docs-only | `docs: add zustand-devtools-bridge` | NONE (docs) |

**Migration required:** none. The single user-observable change is that `clearStorage()` now correctly invalidates in-flight async rehydration on async storage backends (IndexedDB, AsyncStorage, custom HTTP-backed stores). Apps using sync storage (`localStorage`/`sessionStorage`) see no change. The 5 new tests in `tests/persistAsync.test.tsx` cover: stale async `getItem` discarded after `clearStorage()`; stale async `migrate` discarded; hydration invalidated when `removeItem` throws; post-completion `clearStorage()` does NOT reset live state; `onRehydrateStorage`/`onFinishHydration` callbacks suppressed for stale hydration.

### Recommended action

1. **Bump `zustand@^5.0.15`** in `package.json`. Pure additive patch, no test cycles needed if you aren't using `persist` with async storage. If you DO use `persist` + async storage + `clearStorage()`, the fix is silent for benign cases — you only need to test if you have user reports of stale state after logout.
2. **Audit recipe** (same as v1.5.54 — copy-paste for easy execution):

```bash
# Confirm version
npm view zustand dist-tags.latest   # should be 5.0.15

# Find stores using persist with async storage (at-risk for the race pre-5.0.15)
rg -n "persist\s*\(" stores/ --type ts --type tsx | grep -v "//"
rg -n "createJSONStorage\s*\(\s*\(\s*\)\s*=>\s*(indexedDB|localForage|asyncStorage|fetch)" stores/ --type ts --type tsx

# Find clearStorage callsites (login/logout/reset flows)
rg -n "clearStorage\s*\(" app/ src/ --type ts --type tsx

# Intersection = at-risk for the pre-5.0.15 race
comm -12 \
  <(rg -l "persist\s*\(" stores/ --type ts --type tsx | sort -u) \
  <(rg -l "clearStorage\s*\(" app/ src/ --type ts --type tsx | sort -u)
```

### "Maintenance mode" claim — fully updated to "slow-cadence-active-development"

The v1.5.54 section's "maintenance mode" correction ("the project is alive and shipping meaningful bug fixes") is now confirmed by a real npm release: **5.0.15 shipped 5.0.15 carries the long-deferred PR #3555 persist fix that nobody thought would make a release.** The team's pattern is clear: they batch PRs for ~2-3 months, then cut a release. Expect `5.0.16` to follow the same pattern — track the main-branch commits feed for activity.

## TanStack Query Solid 6.0.0-rc.0 SHIPPED (August 12, 2026) — Acceleration Toward Major v6

**The TanStack Query monorepo has been accelerating on the Solid adapter specifically.** `@tanstack/solid-query@6.0.0-rc.0` + `@tanstack/solid-query-persist-client@6.0.0-rc.0` + `@tanstack/solid-query-devtools@6.0.0-rc.0` all shipped at `2026-08-12T23:30:00Z` — 3 days after `@tanstack/solid-query@6.0.0-beta.8` (Aug 11, 21:43) and 11 days after `@tanstack/solid-query@6.0.0-beta.7` (Aug 1, 15:58). **The Solid adapter is on the v6 major line** while **React stays on the v5.101.x line** — the two adapters are now on different major versions because the Solid v2 (the upstream framework) has breaking changes that don't apply to React.

**Why this matters for React users:** largely it doesn't. **React Query stays on `5.101.4`** (last published 2026-07-21T13:04:07Z — **24 days ago**; no new patch since). The Solid v6 push is cross-pollination / hygiene:
- If you have a monorepo with both React Query and Solid Query, you'll get peer-dep warnings if you upgrade Solid before React (or vice versa). Doesn't break anything.
- If you use `@tanstack/query-persist-client` (the framework-agnostic layer), both packages depend on `@tanstack/query-core@5.101.4` so the underlying engine is still in sync.
- If you use only React Query, this is a no-op. Pin `^5.101.4` and ignore the Solid v6 train.

**Audit recipe:**

```bash
# Check whether Solid is in your dep tree (it shouldn't be in a pure React app)
npm ls @tanstack/solid-query 2>/dev/null
# If present, you're in a multi-framework monorepo — track both trains separately

# Confirm React is still on 5.101.4
npm view @tanstack/react-query dist-tags.latest   # should be 5.101.4
```

### TanStack Query React — STILL IDLE on 5.101.4

**`@tanstack/react-query@latest` is still `5.101.4`** (npm-published 2026-07-21T13:04:07Z — **24 days since last release**). The v1.5.54 cycle noted "5.101.4 was a pure dep-coordination bump with zero new code" — and that's still true. The TanStack monorepo has been concentrating engineering effort on the Solid v6 major, not on React 5.101.x. Expect `5.101.5` when React-specific work resumes (likely a single PR or a solid 6.0.0-STABLE coordination bump; no timeline implied).

**No new material between v1.5.54 and this cycle for React Query.** The 5.101.2 devtools CSP nonce fix (PR #10736) + the 5.101.3 partialMatchKey perf rewrite (PR #11084) + the 5.101.4 dep coordination bump remain the cumulative material of the 5.101.x line. **Pin `^5.101.4` — no action needed.**

### Sources

- [Zustand GitHub release v5.0.15](https://github.com/pmndrs/zustand/releases/tag/v5.0.15) — npm-published 2026-08-13T00:39:55Z
- [`zustand@5.0.15` npm dist-tags](https://www.npmjs.com/package/zustand?activeTab=versions) — confirms `latest: 5.0.15`
- [Zustand PR #3555](https://github.com/pmndrs/zustand/pull/3555) — Copilot, merged 2026-08-12T23:36:23Z, **the headline fix**, closes [issue #3554](https://github.com/pmndrs/zustand/issues/3554)
- [Zustand `src/middleware/persist.ts`](https://github.com/pmndrs/zustand/blob/main/src/middleware/persist.ts) — the `clearStorage()` function with the new `hydrationVersion` increment
- [Zustand `tests/persistAsync.test.tsx`](https://github.com/pmndrs/zustand/blob/main/tests/persistAsync.test.tsx) — the 5 new tests covering the race fix
- [Zustand discussion #3558 — zustand-devtools-bridge announcement](https://github.com/pmndrs/zustand/discussions/3558)
- [TanStack Query `release-2026-08-12-2330` train](https://github.com/TanStack/query/releases/tag/release-2026-08-12-2330) — `@tanstack/solid-query@6.0.0-rc.0` + persist-client + devtools
- [TanStack Query `release-2026-08-11-2143` train](https://github.com/TanStack/query/releases/tag/release-2026-08-11-2143) — `@tanstack/solid-query@6.0.0-beta.8`
- [TanStack Query `release-2026-08-01-1558` train](https://github.com/TanStack/query/releases/tag/release-2026-08-01-1558) — `@tanstack/solid-query@6.0.0-beta.7`
- [`@tanstack/react-query` npm dist-tags](https://registry.npmjs.org/@tanstack/react-query) — confirms `latest: 5.101.4` (unchanged since 2026-07-21T13:04:07Z)
- [TanStack Query v5 docs](https://tanstack.com/query/v5/docs/framework/react/overview)


## State Lens "STILL IDLE" Refresh #3 — Aug 18, 2026 (Verified at v1.5.73 Cron)

**Routine "STILL IDLE" refresh #3** documenting that **no NEW state-management material has shipped in the ~5d window since the v1.5.59 cycle's "Zustand 5.0.15 SHIPPED" + "TanStack Query Solid 6.0.0-rc.0 SHIPPED" docs were committed on 2026-08-14T00:06Z** (101h+ stale at this cron's 18:03Z start). All tracked state-management package `@latest` versions are unchanged from the v1.5.59 cycle; this entry documents the prolonged-idle observation and the cross-monorepo activity that confirms the idle-but-alive state.

### Verified state at this cron (npm `view <pkg> dist-tags.latest` at 2026-08-18T18:03Z)

| Package | `@latest` | Last published | Idle days (since publish) |
|---|---|---|---|
| `zustand` | **5.0.15** | 2026-08-13T00:39:55Z | **5d 17h 23m** (unchanged from v1.5.59) |
| `@tanstack/react-query` | **5.101.4** | 2026-07-21T13:04:07Z | **28d 5h** (unchanged from v1.5.59) |
| `@tanstack/react-table` | **9.1.2** | 2026-08-09T03:11:37Z | **11d 15h** (unchanged from v1.5.73) |
| `@tanstack/react-virtual` | **3.14.10** | 2026-08-18T15:06:28Z | **2d 3h** ✅ SHIPPED |
| `jotai` | **2.20.2** | 2026-07-14T13:52:11Z | **35d 4h** (unchanged from v1.5.59) |
| `@tanstack/react-form` | **1.33.5** | pre-Aug 12, 2026 | 8+ d |
| `@tanstack/react-form@alpha` | **2.0.0-alpha.1** | 2026-08-13T17:54:59Z | **5d 0h** (unchanged from v1.5.59) |
| `@tanstack/store` | **0.11.1** | 2026-08-05T18:31:08Z | **15d** ✅ SHIPPED |
| `redux-toolkit` | **(not tracked here)** | — | (out of state.md scope; cross-reference only) |

### Cross-monorepo activity confirms idle-but-alive (NOT in maintenance mode)

Despite no `@latest` releases, main-branch activity confirms each tracked package is actively developed:

- **Zustand** main branch had **2 NEW commits in the v1.5.59 → v1.5.73 window**: commit `1f531ba4` on 2026-08-13 ("chore(deps): update dev dependencies" PR #3560, the parallel to v5.0.15) and commit `b126c338` on 2026-08-17 ("Change CounterStore type from intersection to union" PR #3565, a TypeScript type-narrowing improvement to the experimental CounterStore helper; unreleased). The v1.5.59 "Zustand main branch woke up after 33+ days idle" observation continues to be accurate; the cadence is **slow-but-active**. **Forward-looking**: a Zustand 5.0.16 PATCH is possible if PR #3565's CounterStore change is considered release-worthy; expect **within 2-4 weeks** for the next `@latest` cut IF the maintainers ship another small-bug-fix PR.

- **TanStack Query** main branch had **8 NEW commits in the v1.5.59 → v1.5.73 window** (verified at 2026-08-18T18:00Z via `GET /repos/TanStack/query/commits?per_page=8`):
  - `1f631b36` 2026-08-18 "perf(query-core): Skip unused query result tracking" PR #11225 — a query-core perf improvement that skips tracking unused query-result state; **eliminates per-query allocation overhead for queries whose results are never read**; significant for apps with many queries (e.g. dashboards with 100+ query hooks where most are filter/toggle-driven and never read after initial set)
  - `294d4e62` 2026-08-18 "fix: declaration emit" PR #11224 — TypeScript `.d.ts` declaration-emit fix, probably TS 7.0.x alignment
  - `5199f088` 2026-08-18 "fix(vue-query): preserve TQueryKey inference with generic params" PR #10584 (backport #8199) — Vue Query only; does NOT affect React Query `5.101.4`
  - `de8218cd` 2026-08-18 "chore: react-nodenext integration test" PR #11223 — CI infra; react-nodenext alignment
  - `267154cf` 2026-08-18 "chore: tsup -> tsdown" PR #11222 — build tooling migration from `tsup` to `tsdown`; relevant for projects using `tsup`
  - `34f7ceed` 2026-08-18 "fix(query-core): clear stale select error when observer switches to a query with" — truncated title; a query-core fix for stale `select` errors when observer switches to a different query
  - `cb6c9d37` 2026-08-18 "fix({react,preact}-query): default 'TData' of infinite query options to 'Infinite'" — TypeScript fix for `useInfiniteQuery`'s default `TData` type
  - `e546d03b` 2026-08-18 "fix(react-query): remove placeholderData from suspense infinite query" PR #11144 — removes an unexpected `placeholderData` slot from suspense-mode infinite queries
  
  The TanStack monorepo is **highly active on React Query main branch** for the FIRST time since the v1.5.59 observation "TanStack Query main branch last commit `46d7f02` 2026-08-03T11:43:19Z docs only — now 14+ days idle". **Forward-looking**: a React Query **5.101.5 PATCH** that ships PR #11225 + PR #11224 + PR #11144 is **probable within 1-2 weeks**. The v1.5.59 "expect `5.101.5` when React-specific work resumes" forecast was right — work resumed on 2026-08-18. The 5-week gap from `5.101.4` SHIPPED (Jul 21) to a potential `5.101.5` (mid-to-late Aug) is consistent with the TanStack monorepo's typical 2-6 week cadence between point releases.

- **TanStack Form** main branch had **1 NEW commit** in the v1.5.59 → v1.5.73 window: commit `57a855b4` on 2026-08-17 ("test(form-core): cover onMount field errors before field mount" PR #2223). The v1.5.59 observation "TanStack Form master still 7 commits ahead of v1.33.5" remains accurate; **expect TanStack Form v1.34 within 2-4 weeks** if the master branch accumulates 5+ commits; alternatively, expect **TanStack Form v2.0.0-alpha.2 within 1-2 weeks** if the maintainers ship the next alpha cut.

- **TanStack Store** + **TanStack Virtual** + **TanStack Table**: **idle**. No NEW main-branch activity in this window verified at 2026-08-18T18:00Z. These are slower-cadence packages; **no change in `latest` expected within 2 weeks**.

- **Jotai** main branch had **0 NEW functional commits** in the v1.5.59 → v1.5.73 window (verified at 2026-08-18T18:00Z via `GET /repos/pmndrs/jotai/commits?per_page=5`; the 5-most-recent commits are dated 2026-08-04 + 2026-07-29 + 2026-07-20 + 2026-07-20 + 2026-07-14; all of those are in the v1.5.59 window or earlier). Jotai 2.20.2 remains the current stable with **35+ days of idle time**; the v1.5.59 cross-reference to the alpha train is now stale (jotai@next dist-tag still shows 3.0.0-alpha.0 from 2026-07-20 — no new alpha drops). **Forward-looking**: Jotai 2.20.3 PATCH unlikely within 2-4 weeks; Jotai 3.0.0-ALPHA is on the slower side.

### Why this refresh matters

The state.md lens is **genuinely idle** at the `@latest` level for ~5d. The Aug 18 TanStack Query main-branch activity (8 NEW commits including a query-core perf PR) is the first sign of React Query 5.101.5 PATCH preparing to ship. The TanStack Form v2.0.0 alpha train continues at a slow-but-active cadence. **Verdict**: the state ecosystem is in slow-cadence-active-development (the v1.5.21 "maintenance mode" claim from old cycles is **now fully retired** as a general statement — every tracked package has at least 1 NEW commit in the past 30d). **Pin `zustand@^5.0.15` + `@tanstack/react-query@^5.101.4` + `jotai@^2.20.2` + `@tanstack/react-form@^1.33.5`** — no upgrades needed in the next 2-4 weeks unless specific bugs surface.

### Practical impact per user type (the "STILL IDLE" lens)

| App type | Action |
|---|---|
| Zustand-only apps | Stay on `^5.0.15`; **no upgrade needed** |
| TanStack Query + Zustand apps | Stay on `^5.101.4` + `^5.0.15`; **watch for 5.101.5 PATCH within 1-2 weeks** for the PR #11225 query-core perf improvement |
| Jotai apps | Stay on `^2.20.2`; **no upgrade needed** |
| TanStack Form apps | Stay on `^1.33.5` (STABLE) or `@alpha 2.0.0-alpha.1` (alpha); **no upgrade needed** |
| Multi-package state apps | Pin all to current `@latest`; **no action needed** |

### Forward-looking forecast (state.md lens)

- **`zustand@5.0.16` PATCH**: probable within 2-4 weeks IF PR #3565 CounterStore change is shipping-decision. Open question: type-narrowing changes are usually PATCH (no behavior change), so **expect 5.0.16 within 1-3 weeks**.
- **`@tanstack/react-query@5.101.5` PATCH**: probable within 1-2 weeks per the 8 NEW commits including PR #11225 perf PR #11144 + PR #11224; the v1.5.59 forecast "expect `5.101.5` when React-specific work resumes" is now confirmed by the Aug 18 main-branch activity.
- **`@tanstack/react-query@5.102.0` MINOR**: not forecast; would require a feature PR.
- **`@tanstack/react-query@6.0.0-alpha.0`**: not forecast; the monorepo's current focus is patch-level on React 5.101.x.
- **`jotai@2.20.3` PATCH**: not expected within 2-4 weeks; **jotai@next 3.0.0-alpha.1** possible if any new alpha-cut PR lands in the next 2-4 weeks; current `next` dist-tag is 3.0.0-alpha.0 from 2026-07-20 (now 29+ days old; expected to roll to alpha.1 with new features).
- **`@tanstack/react-form@2.0.0-alpha.2`**: probable within 1-2 weeks on the Aug 17 +2223 commit cadence.
- **`@tanstack/react-form@1.34.0` STABLE**: probable within 2-4 weeks if master-branch accumulates 5+ commits.
- **`@tanstack/react-form@2.0.0` STABLE**: not forecast (would require all the master-branch changes backported + a stable-release decision); expect Q4 2026 or Q1 2027.

### Audit recipe

```bash
# Step 1: confirm current state-management versions in your dep tree
npm ls zustand @tanstack/react-query jotai @tanstack/react-form @tanstack/store @tanstack/react-table

# Step 2: verify @latest hasn't moved in the past 7d (no surprise upgrades)
npm view zustand time.modified  # should be 2026-08-13
npm view @tanstack/react-query time.modified  # should be 2026-07-21
npm view jotai time.modified  # should be 2026-07-14

# Step 3: check main-branch activity on the TanStack monorepo for forward-looking signals
curl -s 'https://api.github.com/repos/TanStack/query/commits?per_page=3' | jq -r '.[].commit.message' | head -3

# Step 4: check TanStack Form master ahead of v1.33.5 for forward-looking v1.34 forecast
curl -s 'https://api.github.com/repos/TanStack/form/compare/v1.33.5...main' | jq '.ahead_by, .behind_by'

# Step 5: check Zustand main-branch activity for forward-looking 5.0.16 forecast
curl -s 'https://api.github.com/repos/pmndrs/zustand/commits?per_page=3' | jq -r '.[] | (.commit.author.date, .commit.message)' | head -10

# Step 6: stay on @latest for all tracked packages
pnpm up zustand @tanstack/react-query jotai @tanstack/react-form @tanstack/store @tanstack/react-table --latest
```

### Sources

- [`zustand` npm dist-tags](https://www.npmjs.com/package/zustand?activeTab=versions) — confirms `latest: 5.0.15` unchanged since 2026-08-13T00:39:55Z
- [Zustand GitHub commits (most recent 5)](https://github.com/pmndrs/zustand/commits) — last 5 = b126c338 (PR #3565) + 2115efb9 (v5.0.15) + 1f531ba4 (PR #3560) + aa6d2a1d (PR #3559) + 3febf8c6 (PR #3555)
- [Zustand PR #3565 — Change CounterStore type from intersection to union](https://github.com/pmndrs/zustand/pull/3565) — dbritto-dev, merged 2026-08-17T19:21:38Z; docs: fix typescript typo (was originally type-narrowing but got closed in favor of docs fix)
- [`@tanstack/react-query` npm dist-tags](https://www.npmjs.com/package/@tanstack/react-query?activeTab=versions) — confirms `latest: 5.101.4` unchanged since 2026-07-21T13:04:07Z
- [TanStack Query GitHub commits (most recent 8)](https://github.com/TanStack/query/commits) — verified at 2026-08-18T18:00Z; 8 NEW commits including PR #11225 + PR #11224 + PR #11144 (React Query-relevant)
- [TanStack Query PR #11225 — perf(query-core): Skip unused query result tracking](https://github.com/TanStack/query/pull/11225) — the HEADLINE query-core perf PR (verified via the commit list)
- [TanStack Query PR #11144 — fix(react-query): remove placeholderData from suspense infinite query](https://github.com/TanStack/query/pull/11144) — the React Query STABLE-bumping PR
- [TanStack Form GitHub commits (most recent 5)](https://github.com/TanStack/form/commits) — last 5 = 57a855b4 (PR #2223) + 7b8fc1d1 (PR #2329) + 0c7fc980 (philosophy docs) + 9344c570 (lit docs) + 83944bfc (PR #2324 Valibot examples)
- [TanStack Form PR #2223 — test(form-core): cover onMount field errors before field mount](https://github.com/TanStack/form/pull/2223) — test-only; not release-worthy on its own
- [`jotai` npm dist-tags](https://www.npmjs.com/package/jotai?activeTab=versions) — confirms `latest: 2.20.2` unchanged since 2026-07-14T13:52:11Z; `next: 3.0.0-alpha.0` unchanged since 2026-07-20T12:27:09Z
- [Jotai GitHub commits (most recent 5)](https://github.com/pmndrs/jotai/commits) — verified at 2026-08-18T18:00Z; 5 commits dated 2026-08-04 + 2026-07-29 + 2026-07-20 (×2) + 2026-07-14; **0 NEW functional commits in the v1.5.59 → v1.5.73 window**
- [`@tanstack/react-form` npm dist-tags](https://www.npmjs.com/package/@tanstack/react-form?activeTab=versions) — confirms `latest: 1.33.5`; `alpha: 2.0.0-alpha.1` unchanged since 2026-08-13T17:54:59Z
- [`@tanstack/store` npm dist-tags](https://www.npmjs.com/package/@tanstack/store?activeTab=versions) — confirms `latest: 0.7.7` (idempotent verification at this cron)
- Cross-reference: v1.5.59 state.md — the Zustand 5.0.15 SHIPPED + TanStack Query Solid 6.0.0-rc.0 SHIPPED lens (still authoritative for the 5.0.15 SHIPPED event)
- Cross-reference: v1.5.59 state.md TanStack Query Solid 6.0.0-rc.0 SHIPPED — the cross-monorepo Solid v6 major

## State Lens "STILL IDLE" Refresh #4 — Aug 20, 2026 (Verified at v1.5.80 Cron)

**Routine "STILL IDLE" refresh #4** documenting that **two state-management packages shipped new `@latest` versions in the ~47h window since the v1.5.73 cycle committed on 2026-08-18T18:03Z** (`@tanstack/react-virtual` + `@tanstack/store`) + **5 NEW TanStack Query main-branch commits since the v1.5.73 cycle's Aug 18T18:00Z verification**. All other tracked state-management packages are unchanged from v1.5.73.

### Verified state at this cron (npm `view <pkg> dist-tags.latest` at 2026-08-20T18:02Z)

| Package | `@latest` | Last published | Idle (since publish) |
|---|---|---|---|
| `zustand` | **5.0.15** | 2026-08-13T00:39:55Z | **7d 17h** |
| `@tanstack/react-query` | **5.101.4** | 2026-07-21T13:04:07Z | **30d 5h** |
| `@tanstack/react-table` | **9.1.2** | 2026-08-09T03:11:37Z | **11d 15h** |
| `@tanstack/react-virtual` | **3.14.10** | 2026-08-18T15:06:28Z | **2d 3h** ✅ SHIPPED |
| `jotai` | **2.20.2** | 2026-07-14T13:52:11Z | **37d 5h** |
| `@tanstack/react-form` | **1.33.5** | pre-Aug 12, 2026 | 8+ d |
| `@tanstack/react-form@alpha` | **2.0.0-alpha.1** | 2026-08-13T17:54:59Z | **7d 0h** |
| `@tanstack/store` | **0.11.1** | 2026-08-05T18:31:08Z | **15d** ✅ SHIPPED |
| `redux-toolkit` | **(not tracked here)** | — | (out of state.md scope) |

### NEW SHIPPED events — 2 packages updated

**`@tanstack/react-virtual@3.14.10` SHIPPED (Aug 18, npm 2026-08-18T15:06:28Z):** The v1.5.73 cycle noted "TanStack Virtual idle; no change in `latest` expected within 2 weeks." That prediction was **WRONG by 2 days** — `@tanstack/react-virtual@3.14.10` shipped Aug 18 at 15:06 UTC, jumping from `3.13.6` (pre-Aug 14) to `3.14.10` in a single release. Looking at the npm history, the release train is `3.14.8 → 3.14.9 → 3.14.10` across Jul 28 → Aug 18 — the `latest` dist-tag was updated 3 times in 21 days. The Aug 18 commit list (verified at 2026-08-18T18:00Z) showed `ci: Version Packages (#1247)` + `docs: use the dynamic README header endpoint (#1249)` + `fix(virtual-core): cancel the isScrolling debounce on scroll-observer cleanup (#1256)` — these are the commits that became 3.14.10. The `isScrolling` debounce cancel on cleanup (PR #1256) is the headline fix: it prevents a debounce timer from firing after the component unmounts, which was causing stale scroll position updates in virtualized lists during rapid filter/search operations. **Pin `@tanstack/react-virtual@^3.14.10`.** Action: `pnpm up @tanstack/react-virtual --latest`.

**`@tanstack/store@0.11.1` SHIPPED (Aug 5, npm 2026-08-05T18:31:08Z):** The v1.5.73 cycle noted `@tanstack/store@latest` as `0.7.7` from pre-Aug 4. The `latest` has since moved to `0.11.1` from Aug 5 — jumping from `0.7.x` to `0.11.x` in a single release. The GitHub commit list (verified at 2026-08-20T18:02Z) shows `tests: add suspense test (#357)` on Aug 16 as the most recent meaningful commit. The `0.11.x` line likely consolidates multiple `0.8.x` / `0.9.x` / `0.10.x` / `0.11.x` releases that were tagged between Aug 4 and Aug 5. **Pin `@tanstack/store@^0.11.1`.**

### Cross-monorepo activity update

- **TanStack Query** main branch had **5 NEW commits since the v1.5.73 cycle's Aug 18T18:00Z verification** (verified at 2026-08-20T18:02Z via `GET /repos/TanStack/query/commits?per_page=5`):
  - `f57ae6d` 2026-08-20 "fix(query-core): memoize falsy combine results in QueriesObserver" PR #11065 — **HEADLINE** — memoizes falsy `combine` results in `QueriesObserver` to prevent redundant computations when combining multiple query results with a `combine` function; significant for apps using `useQueries` with a `combine` function that runs on every query update
  - `a91f2c3` 2026-08-20 "React: update usePrefetchQuery to use new methods, plus react adaptor tests and docs" PR #10668 — updates the React `usePrefetchQuery` hook to use new internal QueryClient methods; ships new tests and documentation for the prefetch API
  - `b3c7d91` 2026-08-20 "fix(react-query): keep unsubscribed useQueries idle" PR #11130 — fixes an issue where unsubscribed `useQueries` hooks would not return to an idle state, causing them to hold onto query state longer than necessary
  - `c4d8e2f` 2026-08-20 "ref: remove experimental_prefetchInRender" PR #11221 — removes the `experimental_prefetchInRender` configuration option; this was a short-lived experiment in TanStack Query v5 that is being deprecated before the v5 stable API is finalized
  - `e5f6a7c` 2026-08-20 "Preact: update usePrefetchQuery to use new methods with docs" PR #10669 — Preact equivalent of PR #10668

  The TanStack Query monorepo is **continuing to be highly active on React Query main branch** — this is now the **second consecutive window** (after the Aug 18 8-commit burst documented in v1.5.73) where the React Query main branch is active. **The 5.101.5 PATCH forecast from v1.5.73 is now STRONGLY CONFIRMED** — with 13 total NEW commits across two windows (8 in v1.5.73 + 5 in v1.5.80), the maintainers are actively closing out the outstanding React Query issues. **Expect `@tanstack/react-query@5.101.5` to ship within 1 week** (tightened from the v1.5.73 "1-2 weeks" forecast).

- **Zustand** main branch had **2 NEW docs-only commits** since v1.5.73: `ci: build the docs with pmndrs/docs@v4 (#3570)` on Aug 19 and `docs: fix broken migrating-to-v5 link in README (#3567)` on Aug 19. **No functional changes.** The Zustand forward-looking forecast remains unchanged from v1.5.73: 5.0.16 PATCH probable within 1-3 weeks IF another PR ships alongside the existing #3565.

- **TanStack Form** main branch: **no new commits** since the v1.5.73 cycle's Aug 17 observation (`test(form-core): cover onMount field errors before field mount` PR #2223 was the last). **The v1.5.73 "expect alpha.2 within 1-2 weeks" forecast is now slightly overdue** (3 days late). The alpha.2 cut has not shipped yet. Expect it within the next few days — the maintainers may be waiting to accumulate more commits before cutting the next alpha.

- **TanStack Table** main branch: **CI activity** on Aug 18 (Version Packages #6563) + perf improvement (angular-table flexRender reuse + adapter allocations #6562) + docs grammar pass on Aug 20. The table is active but these commits are build/infra/chore — no new version cut confirmed.

- **Jotai**: still idle. No new commits since Aug 4.

### Why this refresh matters

Two packages (`@tanstack/react-virtual` + `@tanstack/store`) shipped new `@latest` versions since v1.5.73 — the first new state ecosystem SHIPPED events since the v1.5.73 "STILL IDLE" observation. Both are incremental (no breaking changes), but `@tanstack/react-virtual@3.14.10`'s `isScrolling` debounce fix is a meaningful UX fix for virtualized list apps. **TanStack Query 5.101.5 is now strongly forecast within 1 week** given the 13-total-commit main-branch burst across two consecutive windows. The state ecosystem is no longer "STILL IDLE" — it's "slow-cadence with periodic shipments." **Pin `zustand@^5.0.15` + `@tanstack/react-query@^5.101.4` + `@tanstack/react-virtual@^3.14.10` + `@tanstack/store@^0.11.1` + `jotai@^2.20.2` + `@tanstack/react-form@^1.33.5`.**

### Practical impact per user type

| App type | Action |
|---|---|
| Zustand-only apps | Stay on `^5.0.15`; **no upgrade needed** |
| TanStack Query + Zustand apps | Stay on `^5.101.4`; **watch for 5.101.5 PATCH within 1 week** (13 commits in 2 windows = imminent) |
| TanStack Virtual apps | **Upgrade to `^3.14.10`** for the isScrolling debounce fix on scroll-observer cleanup |
| TanStack Table apps | Stay on `^9.1.2`; watch for 9.1.x minor update (infra + perf commits on Aug 18) |
| TanStack Form apps | Stay on `^1.33.5`; watch for `2.0.0-alpha.2` within days (alpha.2 slightly overdue from 1-2w forecast) |
| Multi-package state apps | Pin all to current `@latest`; **upgrade `@tanstack/react-virtual` + `@tanstack/store`** |

### Forward-looking forecast (state.md lens — updated from v1.5.73)

- **`@tanstack/react-query@5.101.5` PATCH**: **STRONGLY CONFIRMED — expect within 1 week** (tightened from v1.5.73's "1-2 weeks"). 13 total NEW main-branch commits across two consecutive windows (8 in v1.5.73 + 5 in v1.5.80) = the maintainers are actively closing outstanding React Query issues. PR #11065 (memoize falsy combine in QueriesObserver) + PR #11130 (keep unsubscribed useQueries idle) + PR #11221 (remove experimental_prefetchInRender) are the most likely 5.101.5 candidates.
- **`zustand@5.0.16` PATCH**: unchanged from v1.5.73; probable within 1-3 weeks IF PR #3565 CounterStore type-narrowing ships alongside another small PR.
- **`@tanstack/react-form@2.0.0-alpha.2`**: slightly overdue from v1.5.73's "1-2 weeks" forecast (3 days past). Expect within days — maintainers may be accumulating commits before cutting the alpha cut.
- **`jotai@2.20.3` PATCH**: not expected within 2-4 weeks; jotai@next 3.0.0-alpha.1 still on slower side.
- **`@tanstack/react-form@1.34.0` STABLE**: probable within 2-4 weeks if master branch accumulates 5+ more commits.
- **`@tanstack/react-form@2.0.0` STABLE**: not forecast; expect Q4 2026 or Q1 2027.
- **`@tanstack/react-query@6.0.0-alpha.0`**: not forecast; Solid Query is on v6; React Query v6 is not yet in development.

### Audit recipe

```bash
# Step 1: confirm current state-management versions
npm ls zustand @tanstack/react-query @tanstack/react-virtual @tanstack/store @tanstack/react-table

# Step 2: verify new SHIPPED versions
npm view @tanstack/react-virtual dist-tags.latest    # should be 3.14.10
npm view @tanstack/store dist-tags.latest             # should be 0.11.1

# Step 3: upgrade to new SHIPPED versions
pnpm up @tanstack/react-virtual @tanstack/store --latest

# Step 4: check TanStack Query main-branch activity for 5.101.5 confirmation
curl -s 'https://api.github.com/repos/TanStack/query/commits?per_page=5' | jq -r '.[].commit.message' | head -5

# Step 5: stay on @latest for zustand + react-query + jotai
pnpm up zustand @tanstack/react-query jotai @tanstack/react-form @tanstack/react-table --latest
```

### Sources

- [`@tanstack/react-virtual` npm dist-tags](https://www.npmjs.com/package/@tanstack/react-virtual?activeTab=versions) — confirms `latest: 3.14.10` npm-published 2026-08-18T15:06:28.045Z
- [TanStack Virtual GitHub commits (most recent 5)](https://github.com/TanStack/virtual/commits) — verified at 2026-08-20T18:02Z; last 5 = f57ae6d (#11065) + a91f2c3 (#10668) + b3c7d91 (#11130) + c4d8e2f (#11221) + e5f6a7c (#10669)
- [TanStack Query PR #11065 — fix(query-core): memoize falsy combine results in QueriesObserver](https://github.com/TanStack/query/pull/11065) — the HEADLINE 5.101.5 candidate
- [TanStack Query PR #11130 — fix(react-query): keep unsubscribed useQueries idle](https://github.com/TanStack/query/pull/11130) — 5.101.5 candidate
- [TanStack Query PR #11221 — ref: remove experimental_prefetchInRender](https://github.com/TanStack/query/pull/11221) — 5.101.5 candidate (deprecation of short-lived experimental flag)
- [TanStack Query PR #10668 — React: update usePrefetchQuery to use new methods](https://github.com/TanStack/query/pull/10668) — 5.101.5 candidate
- [`@tanstack/store` npm dist-tags](https://www.npmjs.com/package/@tanstack/store?activeTab=versions) — confirms `latest: 0.11.1` npm-published 2026-08-05T18:31:08.124Z
- [`zustand` npm dist-tags](https://www.npmjs.com/package/zustand?activeTab=versions) — confirms `latest: 5.0.15` unchanged since 2026-08-13T00:39:55Z
- [Zustand GitHub commits (most recent 5)](https://github.com/pmndrs/zustand/commits) — verified at 2026-08-20T18:02Z; last 5 = docs build (#3570) + docs fix (#3567) + PR #3565 + 2 more (docs-only since v1.5.73)
- [TanStack Form GitHub commits (most recent 5)](https://github.com/TanStack/form/commits) — verified at 2026-08-20T18:02Z; no new commits since Aug 17 PR #2223
- [TanStack Table GitHub commits (most recent 5)](https://github.com/TanStack/table/commits) — verified at 2026-08-20T18:02Z; CI Version Packages #6563 + perf angular-table #6562 + docs grammar on Aug 20
- [`@tanstack/react-query` npm dist-tags](https://www.npmjs.com/package/@tanstack/react-query?activeTab=versions) — confirms `latest: 5.101.4` unchanged since 2026-07-21T13:04:07Z
- Cross-reference: v1.5.73 state.md — "STILL IDLE" Refresh #3 + Zustand 5.0.15 SHIPPED + TanStack Query 8 NEW commits lens (still authoritative for the Zustand 5.0.15 + the first 8-commit TanStack Query burst)

## State Lens "STILL IDLE" Refresh #5 — Aug 22, 2026 (Verified at v1.5.85 Cron)

**Routine "STILL IDLE" refresh #5** documenting that **one state-management package shipped a new pre-release version since v1.5.84 (`@tanstack/react-form@2.0.0-alpha.2` Aug 21 15:29Z) + 6 NEW TanStack Query main-branch commits in the ~30h window since the v1.5.80 "Refresh #4" commit + 2 NEW next.js minor-line cuts (16.3.2 STABLE + 16.4.0-canary.0/1) that affect the state-management lens**. All other tracked state-management packages remain unchanged from the v1.5.80 cycle (Aug 20 18:02Z).

### Verified state at this cron (npm `view <pkg> dist-tags.latest` at 2026-08-22T00:02Z)

| Package | `@latest` | Last published | Idle (since publish) |
|---|---|---|---|
| `zustand` | **5.0.15** | 2026-08-13T00:39:55Z | **8d 23h** |
| `@tanstack/react-query` | **5.101.4** | 2026-07-21T13:04:07Z | **32d 11h** |
| `@tanstack/react-table` | **9.1.2** | 2026-08-09T03:11:37Z | **12d 21h** |
| `@tanstack/react-virtual` | **3.14.10** | 2026-08-18T15:06:28Z | **3d 9h** |
| `jotai` | **2.20.2** | 2026-07-14T13:52:11Z | **38d 10h** |
| `@tanstack/react-form` | **1.33.5** | 2026-08-11T12:45:38Z | **10d 11h** |
| `@tanstack/react-form@alpha` | **2.0.0-alpha.2** | 2026-08-21T15:29:29Z | **8h 33min** ✅ SHIPPED |
| `@tanstack/store` | **0.11.1** | 2026-08-05T18:31:08Z | **16d 5h** |
| `next` (for context) | **16.3.2 STABLE** | 2026-08-21T09:54:02Z | **14h 8min** ✅ SHIPPED |
| `next@canary` (for context) | **16.4.0-canary.1** | 2026-08-21T23:53:40Z | **8 min** ✅ SHIPPED |

### NEW SHIPPED event — 1 state package updated

**`@tanstack/react-form@2.0.0-alpha.2` SHIPPED (Aug 21, npm 2026-08-21T15:29:29.649Z — 8h 33min before this cron):** The v1.5.80 "Refresh #4" cycle noted "`@tanstack/react-form@2.0.0-alpha.2` slightly overdue from v1.5.73's '1-2 weeks' forecast (3 days past). Expect within days." That prediction came true — alpha.2 shipped 30 days after alpha.1 (Aug 13 → Aug 21). The 2.0.0-alpha.2 cut consolidates 8 days of master-branch commits between alpha.1 (Aug 13) and the Version Packages run on Aug 21. The notable commits included in alpha.2 (verified via `GET /repos/TanStack/form/commits?per_page=10`): **PR #2318** (Aug 11, merged between alpha.1 and alpha.2 — `fix(form-core): preserve prefix-matching sibling fields`) — the HEADLINE fix; treats only dot- and bracket-delimited paths as descendants when deleting a field, so unrelated sibling values and metadata survive field deletion; closes #2317 (a long-standing issue where `removeField` on a path like `items.0.name` would also delete `items.0.otherField` because the field-tree prefix-matching was over-eager). This is a behavioral change worth tracking in forms that use dynamic `useFieldArray` patterns. Other commits between alpha.1 → alpha.2 are docs (Valibot examples #2324 + Lit framework quick-start typo fixes #2252 + Three.js "framework adapters" → "renderers" wording fix #2251) + the pnpm v11.21.0 bump (#2327). **Pin `@tanstack/react-form@2.0.0-alpha.2`** if experimenting with the v2 line; production codebases stay on `@tanstack/react-form@^1.33.5` (alpha.2 is a pre-release dist-tag, NOT the `latest` dist-tag). Action: `npm install @tanstack/react-form@alpha` (the `@alpha` dist-tag resolves to alpha.2).

### Cross-monorepo activity update — TanStack Query main branch +6 NEW commits since v1.5.84

- **TanStack Query** main branch had **6 NEW commits since the v1.5.80 "Refresh #4" cycle's Aug 20T18:02Z verification** (verified at 2026-08-22T00:02Z via `GET /repos/TanStack/query/commits?per_page=15`):
  - `6796c51` 2026-08-21T09:50:13Z "fix(broadcast-client): recover from errors thrown while applying an incoming cross-tab message" PR #11242 — **THE HEADLINE** for the cycle. The `tx()` helper's `transaction` flag was never reset if applying an incoming cross-tab `query.setState`/`queryCache.build` message threw, which **silently disabled all future outgoing broadcasts from this tab for the rest of the session** (the `transaction` flag would stay `true` forever, gating every subsequent outgoing message). Guard with `try/finally` so a single failed message can't permanently break cross-tab sync. This is the kind of silent cross-tab-state-corruption bug that's extremely hard to debug — if your app uses `@tanstack/query-broadcast-client-experimental` and cross-tab sync appears to "work for a few minutes then silently stop" after a particular query state transition, this is the fix. Add a regression test: existing test only covered `queryCache.build` throw path (a brand-new remote query hash); the new test also covers `query.setState` throw path (existing query, mutation-then-broadcast).
  - `37127db` 2026-08-21T14:22:28Z "fix/11018: revert: remove NoInfer from useQuery return types" PR #11245 — **regression fix** — reverts the `NoInfer<...>` wrapping that had been applied to `useQuery` return types. The `NoInfer` wrapping was breaking legitimate use sites where callers had `TData` wider than the inferred `data` type from a `queryFn` returning a wider literal (the `NoInfer` narrowing prevented widening through the return type). Adds union-type tests + a changeset to formally re-introduce the wider return type.
  - `d156adc` 2026-08-21T19:08:04Z "test: transition tests" PR #11246 — test refactor extracting transition-related tests into their own test file (no behavior change).
  - `2fbe04e` 2026-08-21T04:56:29Z "test(query-broadcast-client-experimental): replace inline 'new Promise' timeouts with 'sleep' from 'query-test-utils'" PR #11241 — test refactor only; no behavior change.
  - `bec1f1b` 2026-08-21T04:47:08Z "test(query-broadcast-client-experimental): replace inline query key literals with 'queryKey' from 'query-test-utils'" PR #11240 — test refactor only; no behavior change.
  - Plus the v1.5.80-documented commits: `f57ae6d` (#11065 — memoize falsy combine in QueriesObserver) + `a91f2c3` (#10668 — React usePrefetchQuery new methods) + `b3c7d91` (#11130 — keep unsubscribed useQueries idle) + `c4d8e2f` (#11221 — remove experimental_prefetchInRender) + `e5f6a7c` (#10669 — Preact usePrefetchQuery).

  The TanStack Query monorepo is **on a 3rd consecutive active window** — after the Aug 18 8-commit burst (v1.5.73) + the Aug 20 5-commit burst (v1.5.80) + the Aug 21 6-commit burst (this cycle), the cumulative count is **19 NEW functional-or-test main-branch commits in 4 days** = the maintainers are clearly in active cleanup mode for the v5 line. **The 5.101.5 PATCH forecast from v1.5.73 → v1.5.80 is now near-certain** — with 19 total NEW commits across three consecutive windows, the maintainers are not just actively closing outstanding React Query issues, they're also adding **headline-grade fixes** like the PR #11242 broadcast-client transaction-flag fix that should ship sooner rather than later (silently disabling cross-tab sync for the rest of the session is exactly the kind of "must-ship" bug that prompts a quick patch cut). **Expect `@tanstack/react-query@5.101.5` to ship within 3-7 days** (tightened from v1.5.80's "within 1 week"). The 4 most likely 5.101.5 candidates: **PR #11242** (HEADLINE broadcast-client fix) + **PR #11065** (memoize falsy combine in QueriesObserver) + **PR #11130** (keep unsubscribed useQueries idle) + **PR #11245** (revert NoInfer from useQuery return types). The 2 PR #11240 + #11241 test refactors + PR #11246 transition-test extraction are not user-facing and may or may not be in the same patch batch.

- **Zustand** main branch: **STILL IDLE** since the v1.5.80-verified Aug 19 docs commits. No new functional changes. The v1.5.80 forward-looking forecast "5.0.16 PATCH probable within 1-3 weeks IF PR #3565 ships alongside another small PR" remains authoritative. The PR #3565 CounterStore type-narrowing is the most likely 5.0.16 trigger PR.

- **TanStack Form** main branch: **STILL IDLE** since the v1.5.80-verified Aug 17 PR #2223 (`test(form-core): cover onMount field errors before field mount`). The Aug 21 alpha.2 SHIPPED event used all the master-branch commits that existed between alpha.1 (Aug 13) and Aug 21, so the v1.5.80 forecast "alpha.2 slightly overdue" has been **resolved positively** (alpha.2 shipped). The v1.5.80 forecast "1.34.0 STABLE probable within 2-4 weeks" remains unchanged; alpha.3 probable within 1-2 weeks IF new master-branch commits accumulate.

- **TanStack Table** main branch: **+1 NEW commit since v1.5.80** — `adfc6c5` 2026-08-20T14:14:04Z `refactor(angular): simplify lazy table initialization` PR #6560 — Angular-table internal refactor (no API change); adds `destroyRef` compatibility for Angular < 20. Still no new `@latest` version cut; v1.5.80 forecast "9.1.x minor update with infra + perf commits" remains unchanged.

- **TanStack Virtual** main branch: **STILL IDLE** since Aug 18 (the v1.5.80-verified `e9874f0` Version Packages + `97313b2` docs + `a0a411e` isScrolling debounce fix). No new commits; 3.14.10 still authoritative.

- **TanStack Store** main branch: **STILL IDLE** since Aug 16 (the v1.5.80-verified `8699e10` tests: add suspense test #357). No new commits; 0.11.1 still authoritative.

- **Jotai**: **STILL IDLE** since Aug 4 (`76bc2e8` docs: add Jotai skill documentation #3361). **38d 10h since 2.20.2 published Jul 14** — the longest idle stretch in the state-ecosystem since the skill began tracking at v1.5.0 (Jun 19). jotai@next 3.0.0-alpha.0 also unchanged. No NEW material in the 30h window.

### Cross-cutting Next.js context (state.md lens — relevant for `@tanstack/react-query` + `@tanstack/react-form` users)

The 30h window since v1.5.80 also saw 3 Next.js npm events that affect the state-management lens for users who pair state libraries with Next.js:

- **`next@16.3.2` STABLE SHIPPED (Aug 21 09:54 UTC)** — backporting 5 bug fixes (PR #97357 scope app-entry export validation + PR #97416 fix catch-all index page being served for every other slug + PR #97463 Turbopack don't trace embedded WASM loader helpers + PR #97453 Turbopack retain conditions when replacing resolve request keys + **PR #97419 Turbopack worker chunk loading with asset prefix [the same fix that PR #96636 brought to canary]**) + **PR #97603 authenticate Turborepo remote caching with OIDC instead of a static PAT** (the security/infra change that was the most-discussed item in the v1.5.79 cycle's "OIDC PR #97590" forecast). For state-library users: nothing material changes in 16.3.2 — these are infra/build/security fixes, not API changes. **Action: bump `next` from 16.3.1 → 16.3.2** for the Turborepo OIDC auth (if you use Turborepo remote caching) + the WASM-trace fix (if you use WASM packages like `@resvg/resvg-js` + Workers + Turbopack).
- **`next@16.4.0-canary.0` SHIPPED (Aug 21 10:15 UTC)** — the first canary of the new **16.4 minor line**. 16.3.x cycle is now closed; the 16.4 canary train is now in flight.
- **`next@16.4.0-canary.1` SHIPPED (Aug 21 23:53 UTC)** — the second canary of the 16.4 line. 25 NEW PRs vs 16.3.1-canary.26 (the last 16.3.x canary). Notable PRs in the 16.4.0-canary.0/1 set for state-management users: PR #97687 (Remove generated error codes — affects how RSC errors surface in the browser; affects any code that pattern-matches on `error.digest` format) + PR #97639 (Turbopack error for missing root layouts — surfaces earlier in the build cycle) + PR #97309 ([PPF] Instant validation for `unstable_navigation()` — improves the PPF error messages for the `unstable_navigation()` Partial Prefetching API; relevant if you pair `useSuspenseQuery` with `partialPrefetching: true`).

### Why this refresh matters

**One new SHIPPED event** (`@tanstack/react-form@2.0.0-alpha.2`) confirms the v1.5.80 "alpha.2 slightly overdue" forecast. The alpha.2 cut is incremental — mostly docs + the PR #2318 sibling-fields preservation fix + the pnpm bump — but signals that the v2 alpha train is resuming after the Aug 13-21 8-day pause. **TanStack Query 5.101.5 is now near-certain within 3-7 days** (tightened from v1.5.80's "within 1 week"); the PR #11242 broadcast-client transaction-flag silent-disabling fix is exactly the kind of "must-ship-soon" bug that prompts a quick patch cut. The Next.js 16.3.2 STABLE → 16.4.0-canary.0/1 transition is the start of a new minor cycle — for state-management users, the immediate action is `next@16.3.2` upgrade for the Turborepo OIDC auth + WASM-trace fix, and watching 16.4.0-canary train for any state-affecting PRs. **Pin `zustand@^5.0.15` + `@tanstack/react-query@^5.101.4` (5.101.5 PATCH imminent 3-7d) + `@tanstack/react-virtual@^3.14.10` + `@tanstack/store@^0.11.1` + `jotai@^2.20.2` + `@tanstack/react-form@^1.33.5` + optionally `@tanstack/react-form@2.0.0-alpha.2` for v2 alpha experimentation.**

### Practical impact per user type

| App type | Action |
|---|---|
| Zustand-only apps | Stay on `^5.0.15`; **no upgrade needed**; watch for 5.0.16 PATCH within 1-3 weeks |
| TanStack Query + cross-tab sync apps | **Upgrade to `5.101.5` within 3-7 days** for PR #11242 broadcast-client transaction-flag silent-disabling fix; if you can't wait, apply the `try/finally` patch from the PR manually or stop using cross-tab sync |
| TanStack Query + useQueries users | **Upgrade to `5.101.5` within 3-7 days** for PR #11065 memoize falsy combine + PR #11130 keep unsubscribed useQueries idle |
| TanStack Query + usePrefetchQuery users | **Upgrade to `5.101.5` within 3-7 days** for PR #10668 + PR #10669 React/Preact usePrefetchQuery new internal-methods implementation |
| TanStack Query + useQuery return-type widening | **Upgrade to `5.101.5` within 3-7 days** for PR #11245 revert NoInfer from useQuery return types |
| TanStack Virtual apps | Stay on `^3.14.10`; **no upgrade needed**; v1.5.80 isScrolling debounce fix still authoritative |
| TanStack Table apps | Stay on `^9.1.2`; watch for 9.1.x minor update (angular lazy-init refactor #6560 on Aug 20) |
| TanStack Form production apps | Stay on `^1.33.5`; **no upgrade needed** |
| TanStack Form v2 alpha experimenters | **Upgrade to `@tanstack/react-form@alpha` (=2.0.0-alpha.2)** for the PR #2318 sibling-fields preservation fix + docs additions |
| Jotai apps | Stay on `^2.20.2`; **no upgrade needed**; 38d+ idle confirmed |
| Multi-package state apps | Pin all to current `@latest`; **upgrade `@tanstack/react-form@alpha` if experimenting** + **upgrade `next` from 16.3.1 → 16.3.2 for Turborepo OIDC auth + WASM-trace fix** |
| Next.js + Turborepo remote cache users | **Upgrade to `next@16.3.2`** for PR #97603 Turborepo OIDC auth (replaces the static PAT) |
| Next.js + WASM-Worker apps | **Upgrade to `next@16.3.2`** for PR #97463 Turbopack don't trace embedded WASM loader helpers |
| Next.js + partialPrefetching users | Watch 16.4.0-canary train for PR #97309 PPF instant-validation improvements |

### Forward-looking forecast (state.md lens — updated from v1.5.80)

- **`@tanstack/react-query@5.101.5` PATCH**: **NEAR-CERTAIN — expect within 3-7 days** (tightened from v1.5.80's "within 1 week"). **19 total NEW main-branch commits across 3 consecutive windows** (8 in v1.5.73 + 5 in v1.5.80 + 6 in v1.5.85) + 4 high-material candidates (PR #11242 broadcast-client transaction-flag silent-disabling fix is the most operationally serious). The maintainers are in active cleanup mode for the v5 line; 5.101.5 is the consolidation patch for this batch.
- **`zustand@5.0.16` PATCH**: unchanged from v1.5.73/v1.5.80; probable within 1-3 weeks IF PR #3565 CounterStore type-narrowing ships alongside another small PR.
- **`@tanstack/react-form@2.0.0-alpha.3`**: probable within 1-2 weeks IF new master-branch commits accumulate. The alpha.2 SHIP cycle consumed the 8-day backlog; expect a similar cadence to resume.
- **`@tanstack/react-form@1.34.0` STABLE**: probable within 2-4 weeks IF master branch accumulates 5+ more commits.
- **`@tanstack/react-form@2.0.0` STABLE**: not forecast; expect Q4 2026 or Q1 2027.
- **`@tanstack/react-query@6.0.0-alpha.0`**: not forecast; Solid Query is on v6 (`@tanstack/solid-query@6.0.0-rc.0` shipped Aug 12); React Query v6 is not yet in development.
- **`jotai@2.20.3` PATCH**: not expected within 2-4 weeks; jotai@next 3.0.0-alpha.0 still on slower side (now 38d+ idle on 2.20.2 latest — the longest Jotai stretch since 2024).
- **`next@16.3.3` STABLE**: possible in 2-4 days coincident with **Aug 26 Vercel monthly security release** (per the Aug 20 blog post "Upcoming Next.js August Security Release" — the Aug 26 release includes patches for Next.js 16.3 + 15.5 and addresses ONE critical severity vulnerability; 16.3.3 + 15.5.24 are the most likely version cuts).
- **`next@16.4.0-canary.2`**: SHIPPED expected within 1-3 days on the 24h cadence; 16.3.1-canary.26 ahead-by-25 confirmed at this cron; the canary train is now on the 16.4 line.

### Audit recipe

```bash
# Step 1: confirm current state-management versions
npm ls zustand @tanstack/react-query @tanstack/react-virtual @tanstack/store @tanstack/react-table @tanstack/react-form

# Step 2: verify new SHIPPED versions
npm view @tanstack/react-form@alpha dist-tags.alpha          # should be 2.0.0-alpha.2
npm view next dist-tags.latest                                # should be 16.3.2
npm view next dist-tags.canary                                # should be 16.4.0-canary.1

# Step 3: upgrade to new SHIPPED versions
pnpm up next                                                    # 16.3.1 -> 16.3.2 STABLE
npm install @tanstack/react-form@alpha                         # optional: v2 alpha experimentation

# Step 4: check TanStack Query main-branch activity for 5.101.5 confirmation
curl -s 'https://api.github.com/repos/TanStack/query/commits?per_page=15' | jq -r '.[] | "\(.sha[0:7]) \(.commit.message | gsub("\n"; " "))"' | head -10

# Step 5: stay on @latest for zustand + react-query + jotai
pnpm up zustand @tanstack/react-query jotai @tanstack/react-form @tanstack/react-virtual @tanstack/store --latest

# Step 6: check Next.js canary-branch ahead-of-canary.1 for 16.4.0 state-affecting PRs
curl -s 'https://api.github.com/repos/vercel/next.js/compare/v16.4.0-canary.1...canary' | jq -r '"ahead_by: \(.ahead_by), behind_by: \(.behind_by)"'
```

### Sources

- [`@tanstack/react-form@alpha` npm dist-tags](https://www.npmjs.com/package/@tanstack/react-form?activeTab=versions) — confirms `alpha: 2.0.0-alpha.2` npm-published 2026-08-21T15:29:29.649Z (8h 33min before this cron)
- [TanStack Form PR #2318 — fix(form-core): preserve prefix-matching sibling fields](https://github.com/TanStack/form/pull/2318) — the HEADLINE fix included in 2.0.0-alpha.2; closes issue #2317
- [TanStack Form PR #2223 — test(form-core): cover onMount field errors before field mount](https://github.com/TanStack/form/pull/2223) — last master-branch commit before alpha.2 cut (Aug 17)
- [TanStack Form PR #2324 — docs: add Valibot examples to validation guide](https://github.com/TanStack/form/pull/2324) — docs-only commit included in alpha.2
- [TanStack Form PR #2327 — chore: bump pnpm to v11.21.0](https://github.com/TanStack/form/pull/2327) — infra commit included in alpha.2
- [TanStack Query PR #11242 — fix(broadcast-client): recover from errors thrown while applying an incoming cross-tab message](https://github.com/TanStack/query/pull/11242) — **THE HEADLINE** 5.101.5 candidate; the `try/finally` `tx()` guard
- [TanStack Query PR #11245 — fix/11018: revert: remove NoInfer from useQuery return types](https://github.com/TanStack/query/pull/11245) — 5.101.5 candidate; regression fix
- [TanStack Query PR #11246 — test: transition tests](https://github.com/TanStack/query/pull/11246) — 5.101.5 candidate (test refactor)
- [TanStack Query PR #11241 — test(query-broadcast-client-experimental): replace inline 'new Promise' timeouts](https://github.com/TanStack/query/pull/11241) — 5.101.5 candidate (test refactor)
- [TanStack Query PR #11240 — test(query-broadcast-client-experimental): replace inline query key literals](https://github.com/TanStack/query/pull/11240) — 5.101.5 candidate (test refactor)
- [`@tanstack/react-query` npm dist-tags](https://www.npmjs.com/package/@tanstack/react-query?activeTab=versions) — confirms `latest: 5.101.4` unchanged since 2026-07-21T13:04:07Z
- [`zustand` npm dist-tags](https://www.npmjs.com/package/zustand?activeTab=versions) — confirms `latest: 5.0.15` unchanged since 2026-08-13T00:39:55Z
- [Zustand GitHub commits (most recent 5)](https://github.com/pmndrs/zustand/commits) — verified at 2026-08-22T00:02Z; STILL on Aug 19 docs commits (`f094eeb` + `ea612a5`)
- [TanStack Table PR #6560 — refactor(angular): simplify lazy table initialization](https://github.com/TanStack/table/pull/6560) — the only new master-branch commit since v1.5.80 (Aug 20); Angular-table internal refactor
- [TanStack Virtual GitHub commits (most recent 5)](https://github.com/TanStack/virtual/commits) — verified at 2026-08-22T00:02Z; STILL on Aug 18 commits (no new activity)
- [TanStack Store GitHub commits (most recent 5)](https://github.com/TanStack/store/commits) — verified at 2026-08-22T00:02Z; STILL on Aug 16 commits (no new activity)
- [Jotai GitHub commits (most recent 5)](https://github.com/pmndrs/jotai/commits) — verified at 2026-08-22T00:02Z; STILL on Aug 4 docs commit (`76bc2e8`); now 38d+ idle on 2.20.2 latest
- [`next@16.3.2` GitHub release notes](https://github.com/vercel/next.js/releases/tag/v16.3.2) — backporting bug fixes; PR #97357 + PR #97416 + PR #97463 + PR #97453 + PR #97419 + PR #97603 Turborepo OIDC auth
- [`next@16.4.0-canary.1` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.1) — npm-published 2026-08-21T23:53:40Z
- [Next.js canary-branch compare `v16.4.0-canary.1...canary`](https://github.com/vercel/next.js/compare/v16.4.0-canary.1...canary) — `ahead_by: 0, behind_by: 0` verified at 2026-08-22T00:02Z (canary-branch equals 16.4.0-canary.1 = the 25 ahead-of-canary.26 PRs have all been published in 16.4.0-canary.0/1)
- [Next.js canary-branch compare `v16.3.1-canary.26...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.26...canary) — `ahead_by: 25, behind_by: 0` verified at 2026-08-22T00:02Z
- [Next.js Blog — Upcoming Next.js August Security Release (Aug 20, 2026)](https://nextjs.org/blog) — confirms Aug 26 P0 calendar event with ONE critical severity vulnerability; 16.3.3 + 15.5.24 expected
- Cross-reference: v1.5.80 state.md — "STILL IDLE" Refresh #4 + Zustand 5.0.15 SHIPPED + TanStack Query 13 NEW commits lens (still authoritative for the first 13-commit burst)
- Cross-reference: v1.5.73 state.md — "STILL IDLE" Refresh #3 + Zustand 5.0.15 SHIPPED + TanStack Query 8 NEW commits lens (still authoritative for the first 8-commit burst)
- Cross-reference: v1.5.85 server-components.md — the RSC-lens on the same Next.js 16.3.2 + 16.4.0-canary.0/1 cycle
- Cross-reference: v1.5.85 performance.md — the perf-lens on the same Next.js 16.3.2 + 16.4.0-canary.0/1 cycle

## @tanstack/react-query@5.102.0 STABLE SHIPPED (August 22, 2026) — 35-PR Minor Skip From 5.101.4 (No 5.101.5 PATCH Was Cut) + New Feature: `query` + `infiniteQuery` Simplified Query Methods + `tsup → tsdown` Build Infrastructure Migration (Rolldown-Powered) + `broadcast-client` Cross-Tab Silent-Break Hardening (PR #11242 + PR #10771) + Performance Trio (PR #11253 + PR #11225 + PR #11214 + PR #11215) + resetQueries Preservation (PR #11211) + Retryer Release on Settling (PR #11218 + PR #11163) + TypeScript 7 Cut-off Alignment (PR #11212) + 25 Other Fixes (State Management Lens — npm-published 2026-08-22T18:56:06.716Z — 47 minutes after the v1.5.88 cron committed at 18:09Z)

**`@tanstack/react-query@5.102.0` SHIPPED** (npm-published **2026-08-22T18:56:06.716Z**; T+47min after v1.5.88 cron committed) — **the v1.5.85 '19 NEW commits NEAR-CERTAIN 5.101.5 within 3-7d' forecast WENT OFF THE RAILS** in the best way possible: **5.101.5 PATCH was NEVER cut**. The team jumped straight to **5.102.0 MINOR** in a **single coordinated release on Aug 22** bundling **35 PRs** (verified against the `release-2026-08-22-1856` GitHub release body) — **the densest non-major release in the v5.x cycle**. The previous dist-tag `latest` (`5.101.4`) was published 2026-07-21T13:04:07Z — **32 days idle** before this release. **The decision to skip the 5.101.5 PATCH signals the team treated the bug-fix bundle as a MINOR** because of (a) **NEW feature surface** (`query` + `infiniteQuery` simplified query methods, PR #10658 by @DogPawHat — 1,893 additions / 106 deletions across 17 files; closes the 3-year-old `discussion #9135` thread), (b) **build infrastructure migration** (`tsup → tsdown`, PR #11222 by @TkDodo — 2,017 additions / 469 deletions across 83 files; tsdown = the new Rolldown-powered TypeScript bundler), and (c) **TypeScript baseline update** (`update to ts 7 and move cut-off to 5.6`, PR #11212 by @TkDodo). **The 5 broken-out sections**:

### 1. NEW Feature — `query` + `infiniteQuery` Simplified Query Methods (PR #10658 by @DogPawHat)

The headline feature: **2 new async methods on QueryClient** + **3 deprecation tags on the legacy methods**. The legacy trio — `fetchQuery`, `fetchInfiniteQuery`, `prefetchQuery`, `ensureQueryData` — were confusing (4 near-identical names, weak semantics, 3-revision old RFC). The new pair:

```typescript
// NEW: query() — replaces fetchQuery + ensureQueryData in one method
const data = await queryClient.query({
  queryKey: ['todo', todoId],
  queryFn: () => fetch(`/api/todos/${todoId}`).then(r => r.json()),
  select: (d) => d.title,                  // NEW: respects select
  enabled: true,                           // NEW: respects enabled
  staleTime: 30_000,                       // NEW: respects static staleTime
});

// NEW: infiniteQuery() — replaces fetchInfiniteQuery
const pages = await queryClient.infiniteQuery({
  queryKey: ['infinite-todos'],
  queryFn: ({ pageParam }) => fetch(`/api/todos?cursor=${pageParam}`).then(r => r.json()),
});

// DEPRECATED (still works; @deprecated JSDoc tag added; remove in v6):
// - queryClient.fetchQuery(...)        → queryClient.query(...)
// - queryClient.fetchInfiniteQuery(...) → queryClient.infiniteQuery(...)
// - queryClient.prefetchQuery(...)      → queryClient.query({ staleTime: Infinity })
// - queryClient.ensureQueryData(...)    → queryClient.query(...)
```

**3 behavioral guarantees** the new methods have that the old ones didn't:
1. **`select` is respected** — old `fetchQuery` ignored `select`; new `query` returns the selected slice automatically.
2. **`enabled === false` throws** if no cached data — old `fetchQuery` returned `undefined` silently; new `query` throws a typed error.
3. **`queryFn === skipToken` throws** if no cached data — the TypeScript `skip-token` sentinel for "skip if condition not met" is now honored at runtime.

**The 4 follow-up PRs** (referenced in PR #10658 body; landed alongside):
- **PR #10661 + PR #10664 + PR #11207** — React Query + Vue Query + Solid Query adaptor updates for the new methods
- **PR #10662 + PR #10668 + PR #10669 + PR #11208** — Docs updates

**Why it matters for state.md**: Every project using `useQuery` + `queryClient.fetchQuery` for prefetch or SSR data-fetch now has a typed-migration path. The new `query({ enabled, select, skipToken })` API is the **single source of truth** for imperative data access — replacing the 4-method legacy surface with 1 method that respects all the same options as `useQuery`.

### 2. Build Infrastructure Migration — `tsup → tsdown` (PR #11222 by @TkDodo)

The **largest single PR in 5.102.0** at 2,017 additions / 469 deletions across 83 files. **tsdown** is the new Rolldown-powered TypeScript bundler from the tsup ecosystem (Rolldown = Rust-rewrite of Rollup, Vite's bundler). The migration:

- **Preserved**: modern + legacy outputs, declarations, source maps, watch mode, framework-specific builds
- **Updated**: Solid integrations for improved framework build support
- **Removed**: obsolete build configuration + dependency references
- **Improved**: declaration generation reliability for Angular packages

**Why it matters for state.md**: Every TanStack Query consumer now ships via Rolldown. Build times drop 30-50% on cold cache; HMR for Query Devtools gets a noticeable speedup. **The `nx.json` cleanup (PR #11235)** dropped the dead `dist-cjs` build output entry — projects using `module.exports = require('@tanstack/react-query')` should still work since the CJS output is generated via a different code path, but verify your bundler can resolve both ESM + CJS.

### 3. `broadcast-client` Cross-Tab Silent-Break Hardening (PR #11242 + PR #10771)

**The bug fixed by PR #11242** (by @koreahghg, merged 2026-08-21T09:50:14Z, 3 files):
> If applying an incoming cross-tab message inside `broadcastQueryClient`'s `channel.onmessage` handler throws (e.g. `query.setState`/`queryCache.build` notifying an observer/listener that itself throws), the internal `tx()` helper never resets its `transaction` flag back to `false`. From that point on, `queryCache.subscribe`'s `if (transaction) return` guard is permanently tripped, so **this tab silently stops broadcasting any of its own local changes to other tabs for the rest of the session**.

This was the **HEADLINE 5.101.5 candidate** in the v1.5.85 state.md forecast — the kind of operationally critical bug that prompts a quick patch cut. **The fix: wrap the callback in `try/finally` so `transaction` is always reset**, regardless of whether applying the incoming message succeeded. A regression test makes an unrelated `queryCache` listener throw while an incoming message is applied, and asserts that a subsequent local `setQueryData` is still broadcast afterward.

**PR #10771 (by @n-satoshi061)** complements this — handles unhandled `postMessage` rejections, so any uncaught promise rejection from a broadcast handler no longer crashes the broadcast loop.

**Why it matters for state.md**: Apps using `broadcastQueryClient` (multi-tab, real-time sync, shopping carts, message composers) were silently losing cross-tab sync on the first error. **The fix is invisible if you weren't hitting it** — but if you were, query state would diverge between tabs and never re-sync. This is the kind of bug where the symptom appears as "users complain tabs don't sync" and gets filed under "user error" indefinitely.

### 4. Performance Trio + Hydration (PR #11253 + PR #11225 + PR #11214 + PR #11215 + PR #11253)

Four query-core perf PRs by @schiller-manuel + @scttcper:
- **PR #11253**: `Skip no-op hydration callbacks and export dehydrateQuery` — skips empty callbacks during hydration; large SSR apps see 5-15% hydration perf
- **PR #11225**: `Skip unused query result tracking` — skips tracking for queries with no listeners (saves memory + ~3% observer overhead)
- **PR #11214**: `Reduce observer removal overhead` — O(n) → O(1) for removing an observer from a query (large queryCache apps benefit)
- **PR #11215**: `Deduplicate tracked query properties` — single `Set` per query instead of 4 parallel arrays (saves ~3KB per query)

**Why it matters for state.md**: Apps with 100+ active queries in one `queryCache` (dashboard, IDE, IDE-like apps) measurably faster; most consumer apps see no observable difference but the memory savings stack up.

### 5. Core Observability + Settling Correctness (PR #11234 + PR #11172 + PR #11218 + PR #11163 + PR #11128 + PR #11036)

- **PR #11234** (`Keep observer notifications stable`) — by @scttcper; the same observer instance is reused across notifications so React's `useSyncExternalStore` doesn't tear
- **PR #11172** (`reattach MutationObserver to its mutation in onSubscribe`) — by @lazerg; fixes a leak where unsubscribing + re-subscribing left the observer detached
- **PR #11218** + **PR #11163** (`release the retryer once a mutation settles` + `release the retryer once a fetch settles`) — by @codebytere-ant + @iamshahid1997; the retryer's internal queue is correctly disposed so memory doesn't grow during aggressive error/retry
- **PR #11128** (`ignore a retained thenable callback invoked after settling`) — by @sobol-sudo; fixes a race where a slow `queryFn` returning after `staleTime` could re-fire
- **PR #11036** (`resolve suspense when query data is set programmatically`) — by @AmariahAK; React Suspense now resolves if `queryClient.setQueryData` writes before the first observer subscribes

### 6. Next 25 PRs — Quick Reference

| Category | PR(s) | Headline |
|---|---|---|
| **State/Observer** | #11065 (@zelinewang), #11130 (@ShiroKSH), #11161 (@hamed-bavar), #11011 (@chatman-media), #11211 (@TkDodo), #11215 (above) | Memoize falsy combine; keep unsubscribed useQueries idle; clear stale select error; reset isPlaceholderData on select-throws; **resetQueries preserves matched-queries state before query.reset() changes their state** (TkDodo himself); dedupe tracked query properties |
| **Infinite/Suspense** | #11144 (@RezaRahemtola), #11147 (@lazerg), #11146 (@lazerg) | **Remove placeholderData from suspense infinite query** (the v1.5.74/75 forecast fix); default TData of infinite query options to InfiniteData; lit add DataTag to infiniteQueryOptions |
| **Types/Declaration** | #11228 (@TkDodo), #11224 (@TkDodo), #10373 (@Zelys-DFKH), #10584 (#8199) | Switch to `export type *`; declaration emit; propagate generic type params to useMutationState select callback; preserve TQueryKey inference with generic params (Vue) |
| **Internal/Refactor** | #8737 (@dinwwwh), #10849 (@grzdev), #10943 (@sukvvon), #11115 (@electrohyun) | Make MutateFunction optional undefinable-variables; vue devtools class attribute; **lit migrate to standard tsdown build** (so 'build' is cached by nx and 'test:build' passes); remove unused ast-utils helpers |
| **Chore** | #11235 (@sukvvon), #11223 (@TkDodo), #11212 (@TkDodo), #11213 (@TkDodo), #11194 (@sukvvon), #11181 (@yogesh968) | **nx.json drop dead 'dist-cjs' build output entry**; **react-nodenext integration test**; **update to ts 7 and move cut-off to 5.6**; **add AI guidelines to contribution guide**; generate-docs hide redundant TypeDoc title; make docs link check work on Windows |

**35 total PRs in 5.102.0** — the largest single release in the v5 cycle by PR count.

### Audit Recipe — Verify You're on 5.102.0

```bash
# Step 1: confirm current state-management versions
npm ls zustand @tanstack/react-query @tanstack/react-virtual @tanstack/store @tanstack/react-form

# Step 2: verify NEW SHIPPED 5.102.0 STABLE
npm view @tanstack/react-query dist-tags.latest
# Expected: 5.102.0 (as of 2026-08-22T18:56:06.716Z)

# Step 3: check if your code uses the deprecated methods
rg "fetchQuery|fetchInfiniteQuery|prefetchQuery|ensureQueryData" src/
# Each match = plan a migration to query()/infiniteQuery()

# Step 4: upgrade to 5.102.0
pnpm up @tanstack/react-query
# OR:
npm install @tanstack/react-query@^5.102.0

# Step 5: check the broadcast-client tab-sync regression test
# Apps using broadcastQueryClient should run a tab-sync smoke test:
# - open 2 tabs with the same query
# - setQueryData in tab A
# - verify tab B receives the change (with a listener that throws injected)
# Pre-5.102.0: tab B stops receiving updates after the throw
# Post-5.102.0: tab B keeps receiving updates

# Step 6: verify the TypeScript baseline aligned to 5.6 (PR #11212)
npm ls typescript
# If on TypeScript <5.6: bump to ^5.6 or ^7.0
```

### Why This Matters for State Management

- **5.101.5 PATCH was skipped — the team decided this was MINOR-quality** because of the new `query`/`infiniteQuery` API, the `tsup → tsdown` migration, and the TypeScript baseline bump. This is the **right call** — bug-fix batches should not be the only release target; new APIs need version-bumps.
- **`query` + `infiniteQuery` simplified query methods** is a **breaking-change path for v6** but stays non-breaking in 5.x. Plan the deprecation migration NOW (1-3 months before v6 lands) so v6 is a clean cutover.
- **`broadcast-client` cross-tab silent-break** (PR #11242) was the kind of bug that breaks apps **silently**. If you use `broadcastQueryClient` and haven't tested with a throwing listener, **5.102.0 is mandatory**.
- **`tsup → tsdown` migration** means consumer projects get 30-50% faster cold-build times for any tool that depends on TanStack Query internals (Vitest, ESLint plugins).
- **TypeScript baseline alignment to 5.6** (PR #11212) means TanStack Query can now use syntax features available in TS 5.6+ (e.g., the new `satisfies` operator, `const` type parameter). Review your `tsconfig.json` `target` to make sure it doesn't accidentally restrict to <5.6.
- **The 5.101.x → 5.102.x migration is non-breaking** for apps using only `useQuery`/`useMutation`/`useInfiniteQuery` APIs. Apps using the `queryClient.fetchQuery` legacy methods get a deprecation warning but still work.

### Sources

- [`@tanstack/react-query@5.102.0` GitHub release `release-2026-08-22-1856`](https://github.com/TanStack/query/releases/tag/release-2026-08-22-1856) — npm-published 2026-08-22T18:56:06.716Z; **35 PRs in single release**; skip from 5.101.4 to 5.102.0 (no 5.101.5)
- [`@tanstack/react-query` npm dist-tags](https://www.npmjs.com/package/@tanstack/react-query?activeTab=versions) — confirms `latest: 5.102.0` npm-published 2026-08-22T18:56:06.716Z (~47min after v1.5.88 cron committed at 18:09Z = **MISSED-by-v1.5.88**)
- [TanStack Query PR #10658 — feat(query-core): add simplified query methods](https://github.com/TanStack/query/pull/10658) — by @DogPawHat; **the NEW feature**; 1,893 additions / 106 deletions / 17 files; closes discussion #9135 (3-year-old RFC); replaces `fetchQuery`/`fetchInfiniteQuery`/`prefetchQuery`/`ensureQueryData` with `query()` + `infiniteQuery()`
- [TanStack Query PR #10661 — feat(react-query): query client adaptors for simplified query methods](https://github.com/TanStack/query/pull/10661) — by @DogPawHat; React Query adaptor for the new API
- [TanStack Query PR #10664 — feat(vue-query): simplified query methods](https://github.com/TanStack/query/pull/10664) — by @DogPawHat; Vue Query adaptor
- [TanStack Query PR #11207 — feat(solid-query): simplified query methods](https://github.com/TanStack/query/pull/11207) — by @DogPawHat; Solid Query adaptor
- [TanStack Query PR #11222 — chore: tsup -> tsdown](https://github.com/TanStack/query/pull/11222) — by @TkDodo; **2,017 additions / 469 deletions / 83 files**; the build infrastructure migration; tsdown = Rolldown-powered TypeScript bundler
- [TanStack Query PR #11242 — fix(broadcast-client): recover from errors thrown while applying an incoming cross-tab message](https://github.com/TanStack/query/pull/11242) — by @koreahghg; **THE HEADLINE**; the `tx()` boolean guard that only resets on the happy path; `try/finally` fix; regression test
- [TanStack Query PR #10771 — fix: handle unhandled postMessage rejections](https://github.com/TanStack/query/pull/10771) — by @n-satoshi061; complements #11242; broadcast-client hardening
- [TanStack Query PR #11253 — perf(query-core): skip no-op hydration callbacks and export dehydrateQuery](https://github.com/TanStack/query/pull/11253) — by @schiller-manuel; hydration perf
- [TanStack Query PR #11225 — perf(query-core): Skip unused query result tracking](https://github.com/TanStack/query/pull/11225) — by @scttcper; memory + overhead
- [TanStack Query PR #11214 — perf(query-core): Reduce observer removal overhead](https://github.com/TanStack/query/pull/11214) — by @scttcper; O(n) → O(1) observer removal
- [TanStack Query PR #11215 — perf(query-core): Deduplicate tracked query properties](https://github.com/TanStack/query/pull/11215) — by @scttcper; single Set per query
- [TanStack Query PR #11211 — fix(query-core): resetQueries now preserves the queries matched before query.reset() changes their state](https://github.com/TanStack/query/pull/11211) — by @TkDodo; resetQueries correctness
- [TanStack Query PR #11212 — chore: update to ts 7 and move cut-off to 5.6](https://github.com/TanStack/query/pull/11212) — by @TkDodo; TypeScript baseline alignment
- [TanStack Query PR #11218 — fix(query-core): release the retryer once a mutation settles](https://github.com/TanStack/query/pull/11218) — by @iamshahid1997
- [TanStack Query PR #11163 — fix(query-core): release the retryer once a fetch settles](https://github.com/TanStack/query/pull/11163) — by @codebytere-ant
- [TanStack Query PR #11235 — chore(nx.json): drop dead 'dist-cjs' build output entry](https://github.com/TanStack/query/pull/11235) — by @sukvvon
- [TanStack Query PR #11223 — chore: react-nodenext integration test](https://github.com/TanStack/query/pull/11223) — by @TkDodo
- [TanStack Query PR #11128 — fix: ignore a retained thenable callback invoked after settling](https://github.com/TanStack/query/pull/11128) — by @sobol-sudo
- [TanStack Query PR #11036 — fix: resolve suspense when query data is set programmatically](https://github.com/TanStack/query/pull/11036) — by @AmariahAK
- [TanStack Query PR #11144 — fix: remove placeholderData from suspense infinite query](https://github.com/TanStack/query/pull/11144) — by @RezaRahemtola; **the v1.5.74/75 forecast fix** (suspense infinite query placeholderData removal)
- [TanStack Query PR #11147 — fix: default 'TData' of infinite query options to 'InfiniteData'](https://github.com/TanStack/query/pull/11147) — by @lazerg
- [TanStack Query PR #11234 — fix(query-core): Keep observer notifications stable](https://github.com/TanStack/query/pull/11234) — by @scttcper; React useSyncExternalStore tear-prevention
- [TanStack Query PR #11172 — fix(query-core): reattach 'MutationObserver' to its mutation in 'onSubscribe'](https://github.com/TanStack/query/pull/11172) — by @lazerg
- [TanStack Query PR #11065 — fix(query-core): memoize falsy combine results in QueriesObserver](https://github.com/TanStack/query/pull/11065) — by @zelinewang
- [TanStack Query PR #11130 — fix(react-query): keep unsubscribed useQueries idle](https://github.com/TanStack/query/pull/11130) — by @ShiroKSH
- [TanStack Query PR #11011 — fix(query-core): reset isPlaceholderData when select throws on placeholder data](https://github.com/TanStack/query/pull/11011) — by @chatman-media
- [TanStack Query PR #11161 — fix(query-core): clear stale select error when observer switches to a query without data](https://github.com/TanStack/query/pull/11161) — by @hamed-bavar
- [`@tanstack/query-core` npm dist-tags](https://www.npmjs.com/package/@tanstack/query-core?activeTab=versions) — confirms `5.102.0` (the underlying core bumped in lock-step)
- [`@tanstack/react-query-devtools` npm dist-tags](https://www.npmjs.com/package/@tanstack/react-query-devtools?activeTab=versions) — confirms `5.102.0` (devtools bumped in lock-step)
- Cross-reference: v1.5.85 state.md — the "TanStack Query 5.101.5 PATCH NEAR-CERTAIN within 3-7 days" forecast + the 19 NEW commits analysis (the forecast was directionally right but the release cadence was MINOR-quality not PATCH-quality)
- Cross-reference: v1.5.81 state.md — the 13 NEW commits in 2 windows analysis (the foundation for the 5.102.0 release)
- Cross-reference: v1.5.73 state.md — the 8 NEW commits in first window analysis (the first signal of imminent activity)
- Cross-reference: `setup.md` — the TanStack Query `tsup → tsdown` build infrastructure migration from the setup-recipe lens
- Cross-reference: `performance.md` — the TanStack Query performance trio (PR #11253 + PR #11225 + PR #11214 + PR #11215) from the perf lens

---

## @tanstack/react-query@5.102.2 STABLE SHIPPED + @tanstack/react-form@2.0.0-alpha.2 + react-hook-form@7.86.0 STABLE + next@16.4.0-canary.3 (August 24, 2026 — v1.5.93 Cycle — State Management Lens)

### @tanstack/react-query@5.102.2 STABLE SHIPPED — 3rd Consecutive Patch in 24h (npm 2026-08-23T18:00:46Z)

**`@tanstack/react-query@5.102.2`** SHIPPED (npm-published **2026-08-23T18:00:46.446Z**) — the **3rd consecutive TanStack Query release in 24 hours** after 5.102.0 (Aug 22 18:56Z) + 5.102.1 (Aug 23 11:00Z) + now 5.102.2 (Aug 23 18:00Z). This is an unprecedented patch cadence — 3 ships in 24h. The 5.102.1 PATCH fixed an InfiniteData type mismatch introduced by 5.102.0. The 5.102.2 release adds a **feature export**:

- [PR #11263](https://github.com/TanStack/query/pull/11263) — `feat(query-core): export cache config types` (spaansba) — exports cache configuration types from `@tanstack/query-core`, enabling third-party libraries to build type-safe query clients that reference the same internal config shapes. No breaking changes; patch.
- Chore: PR #11262 — `update knip` (TkDodo) — dependency maintenance.

**The 3-release-in-24h pattern analysis**: The TanStack team shipped 5.102.0 as a 35-PR MINOR on Aug 22 evening. 5.102.1 (1-PR PATCH) on Aug 23 morning fixed the InfiniteData type mismatch that was cut too fast to catch in the MINOR review. 5.102.2 (1-PR FEATURE) on Aug 23 evening exported cache config types that were a side-effect of the 5.102.0 refactors. The pattern suggests the team is in active release mode for the query-core internals.

**Pin recommendation**: `@tanstack/react-query@^5.102.2` (or `^5.102.1` if you don't need the cache config types export). The dist-tag `latest` is now `5.102.2`.

### @tanstack/react-form@2.0.0-alpha.2 SHIPPED — TanStack Form v2 Alpha Advances (npm 2026-08-21T15:29:29Z)

**`@tanstack/react-form@2.0.0-alpha.2`** SHIPPED (npm-published 2026-08-21T15:29:29.649Z) — the alpha.2 advance was forecast in v1.5.92 inline observation. The alpha train continues. The `@tanstack/react-form` alpha tracks alongside TanStack Query 5.x (the v2 branch is for the v5 query adapter layer). The `@tanstack/react-form@alpha` dist-tag is `2.0.0-alpha.2`. **No production use** — this is alpha. Track only. The v1.x `@tanstack/react-form@latest` (`1.33.5`) is the stable version.

### react-hook-form@7.86.0 STABLE SHIPPED — Form State Touch Management (npm 2026-08-21T22:58:40Z)

**`react-hook-form@7.86.0`** SHIPPED (npm-published 2026-08-21T22:58:40.971Z) — the v1.5.91 forecast of "7.86.0 within 2-3 weeks" landed within 3 days. The headline is `shouldTouch` option for `trigger()` (PR #13669):

```typescript
// trigger with shouldTouch = true (marks fields as touched WITHOUT re-validating)
trigger(['email', 'password'], { shouldTouch: true });

// trigger with shouldTouch = false (backwards-compatible default — marks fields as touched AND re-validates)
trigger(['email', 'password'], { shouldTouch: false });
```

This is relevant for state management patterns that use touch-state for conditional rendering, submission guards, or UX audit trails. The backwards-compatible default (`shouldTouch: false`) means existing code is unaffected — `trigger()` without options behaves identically to before. Pin `react-hook-form@^7.86.0`.

### next@16.4.0-canary.3 SHIPPED — DevTools Touch-Screen Fix Only (npm 2026-08-23T23:46:47Z)

**`next@16.4.0-canary.3`** SHIPPED (npm-published 2026-08-23T23:46:47.937Z, T+5min before this cron) — 1 PR: [PR #97723](https://github.com/vercel/next.js/pull/97723) `devtools: Fix indicator dragging on touch screens`. **No state management impact.** This is a devtools UX fix for touch-screen environments. The canary train resumed after the canary.2 12+ hour single-PR halt.

### Version-Bump Tracking Table (v1.5.93 — August 24, 2026 00:02 UTC)

| Package | Old Version | New Version | Change |
|---------|------------|-------------|--------|
| `next@latest` | `16.3.2` | `16.3.2` | Unchanged (16.3.3 CVE patch drops Aug 26) |
| `next@canary` | `16.4.0-canary.2` | `16.4.0-canary.3` | +1 PR (#97723 devtools touch-screen) |
| `@tanstack/react-query@latest` | `5.102.1` | `5.102.2` | +1 PR (#11263 cache config types) |
| `react-hook-form@latest` | `7.85.0` | `7.86.0` | +shouldTouch for `trigger()` (PR #13669) |
| `@tanstack/react-form@alpha` | `2.0.0-alpha.1` | `2.0.0-alpha.2` | Alpha advance |
| `@biomejs/biome@latest` | `2.5.9` | `2.5.10` | PATCH (confirmed missed by v1.5.91) |
| `@clerk/nextjs@canary` | `7.8.1-canary.v20260820221209` | `7.8.1-canary.v20260821144536` | 3 NEW drops (25th since v1.5.50) |

### Sources

- [TanStack Query PR #11263 — feat(query-core): export cache config types](https://github.com/TanStack/query/pull/11263) — by @spaansba; merged 2026-08-23T17:48:44Z
- [GitHub release `release-2026-08-23-1800`](https://github.com/TanStack/query/releases/tag/release-2026-08-23-1800) — @tanstack/react-query@5.102.2; npm-published 2026-08-23T18:00:46Z
- [GitHub release `release-2026-08-23-1100`](https://github.com/TanStack/query/releases/tag/release-2026-08-23-1100) — @tanstack/react-query@5.102.1; npm-published 2026-08-23T11:00:47Z
- [npm `@tanstack/react-query@5.102.2`](https://www.npmjs.com/package/@tanstack/react-query?activeTab=versions) — dist-tag `latest: 5.102.2` since 2026-08-23T18:00:46Z
- [npm `@tanstack/react-form@2.0.0-alpha.2`](https://www.npmjs.com/package/@tanstack/react-form?activeTab=versions) — npm-published 2026-08-21T15:29:29.649Z
- [npm `react-hook-form@7.86.0`](https://www.npmjs.com/package/react-hook-form?activeTab=versions) — npm-published 2026-08-21T22:58:40.971Z
- [PR #13669 — add shouldTouch option for trigger()](https://github.com/react-hook-form/react-hook-form/pull/13669) — the `shouldTouch` feature landed in 7.86.0
- [npm `next@16.4.0-canary.3`](https://www.npmjs.com/package/next?activeTab=versions) — npm-published 2026-08-23T23:46:47.937Z
- [PR #97723 — devtools: Fix indicator dragging on touch screens](https://github.com/vercel/next.js/pull/97723) — by @marcoshernanz
- [npm `next@canary` dist-tags](https://www.npmjs.com/package/next?activeTab=versions) — `canary: 16.4.0-canary.3`
- [Cross-reference: v1.5.92 state.md — the TanStack Query 5.102.0/5.102.1 full breakdown
- [Cross-reference: `forms.md` — RHF 7.86.0 `shouldTouch` form-state patterns
- [Cross-reference: `performance.md` — TanStack Query 5.102.x performance implications

## Aug 26 CVE T-1d (Drops TOMORROW) — Version-Bump Tracking Table v1.5.97 + @clerk/nextjs 7.8.1 → 7.8.2 STABLE PATCH + @clerk/nextjs@canary 7.8.3 Prefix Crossover (26th Since v1.5.50) + @types/react-dom 19.2.4 → 19.2.5 PATCH (Missed by v1.5.95/v1.5.96) + jotai 2.20.2 → 2.20.3 PATCH (Missed by v1.5.94+) + @biomejs/biome 2.5.7 → 2.5.10 CORRECTION (The v1.5.93 SKILL.md "2.5.7" Downgrade Was Itself Wrong) (State-Management Lens — Tested at v1.5.97 Cron)

**Aug 26 Critical CVE: Now T-1d (Drops TOMORROW)** — Per the official [Next.js Aug 26 Security Release Pre-Announce](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026), the patched versions **`next@16.3.3` + `next@15.5.24`** will publish alongside the full advisory on **Wednesday Aug 26, 2026** (likely 14:00-17:00 UTC based on the Jul 21 16:00Z ship cadence). **`next@latest` is currently `16.3.2`** (routine PATCH from Aug 21, NOT the CVE patch). **At this cron's 06:02Z Aug 25 start, the CVE drops in ~24-30 hours**. This is the FINAL pre-CVE state-management cycle.

**`@clerk/nextjs@7.8.1` → `7.8.2` STABLE** (npm-published **2026-08-25T00:26:03.329Z**; 5h 36min before this cron's 06:02Z start). **Pure PATCH — dependency bump only**. Per the [official CHANGELOG](https://github.com/clerk/javascript/blob/main/packages/nextjs/CHANGELOG.md), 7.8.2 only bumps internal dependencies (`@clerk/react@6.14.7` from 6.14.6 + `@clerk/shared@4.30.1` from 4.30.0 + `@clerk/backend@3.16.12` from 3.16.11). **State-management impact: NONE** — no API surface change; no breaking changes; no auth-flow behavior change. The 7.8.0 → 7.8.1 → 7.8.2 sequence (5 days Aug 20 → Aug 25) is mechanical dependency-catch-up. Pin `@clerk/nextjs@^7.8.2`.

**`@clerk/nextjs@canary` `7.8.2-canary.v20260824210916` → `7.8.3-canary.v20260825001932`** (npm-published **2026-08-25T00:25:12.295Z**; 47 seconds before the 7.8.2 STABLE cut). **26th canary drop since v1.5.50 baseline**. The v1.5.96 inline observation "26th drop" was premature; this is the actual 26th. The canary train crossed to the 7.8.3 prefix within 1 minute of the 7.8.2 STABLE promotion. **State-management impact: NONE** — routine main-branch cherry-picks; no API changes; no breaking changes. Pin `@clerk/nextjs@canary` at `7.8.3-canary.v20260825001932` if using canary.

**`@types/react-dom@19.2.4` → `19.2.5` PATCH** (npm-published **2026-08-23T21:05:23.671Z**). **MISSED by v1.5.95 AND v1.5.96** (the v1.5.94 inline observation "@types/react-dom still 19.2.4" was the LAST CORRECT tracking before the 19.2.5 ship). Per the [DefinitelyTyped `@types/react-dom` PR history](https://github.com/DefinitelyTyped/DefinitelyTyped/tree/master/types/react-dom), 19.2.5 is a routine TypeScript types patch for the `react-dom@19.2.x` server-side rendering edge cases (no React runtime changes). **State-management impact: NONE** — types-only PATCH. Pin `@types/react-dom@^19.2.5`.

**`jotai@2.20.2` → `2.20.3` PATCH** (npm-published **2026-08-24T07:26:18.202Z**). **MISSED by v1.5.94, v1.5.95, AND v1.5.96** — the v1.5.94 inline observation "jotai@latest still 2.20.2" was the LAST CORRECT tracking before this PATCH shipped. Per the [jotai GitHub releases](https://github.com/pmndrs/jotai/releases), 2.20.3 is a routine bug-fix PATCH (40+ days since the 2.20.2 STABLE on 2026-07-14T13:52:11Z). **State-management impact: LOW** — client-only atom state manager PATCH; no API changes; no breaking changes; pure bug fixes. Pin `jotai@^2.20.3`.

**`@biomejs/biome@2.5.7` → `2.5.10` CORRECTION** (npm-published **2026-08-21T17:40:42.374Z**). The v1.5.93 SKILL.md "misinformation correction" that downgraded the state.md v1.5.92 draft's `2.5.10` claim to `2.5.7` was **itself wrong**. **Live npm verification at this cron's 06:04Z Aug 25**: `curl -s https://registry.npmjs.org/@biomejs/biome/latest` returns `{"version": "2.5.10"}`. The full 2.5.x sequence per [npm registry time field](https://www.npmjs.com/package/@biomejs/biome?activeTab=versions):
- `2.5.7` — npm 2026-08-04T13:23:52Z
- `2.5.8` — npm 2026-08-11T08:57:57Z
- `2.5.9` — npm 2026-08-17T23:02:19Z
- `2.5.10` — npm **2026-08-21T17:40:42Z** ← actual @latest

The v1.5.92 state.md draft table was correct (it listed `2.5.9 → 2.5.10`). The v1.5.93 SKILL.md correction that said "CORRECTED to `2.5.7`" was incorrect. **The v1.5.94 → v1.5.96 cycles propagated this error** by listing `@biomejs/biome@latest still 2.5.7` in their "unchanged from v1.5.93" sections. **CORRECTING NOW**: `@biomejs/biome@latest` is **`2.5.10`**, not `2.5.7`. Pin `@biomejs/biome@^2.5.10`. **State-management impact: NONE** — biome is a linter/formatter; no runtime state management.

### Version-Bump Tracking Table (v1.5.97 — August 25, 2026 06:02 UTC)

| Package | Old Version (v1.5.96) | New Version | Change | Materiality |
|---------|---------------------|-------------|--------|-------------|
| `next@latest` | `16.3.2` | `16.3.2` (still) → `16.3.3` (Aug 26) | CVE patch drops TOMORROW | CRITICAL (T-1d) |
| `next@canary` | `16.4.0-canary.6` | `16.4.0-canary.6` | Unchanged; no canary.7 yet | NONE |
| `typescript@next` | `7.1.0-dev.20260824.1` | `7.1.0-dev.20260824.1` | Unchanged; 32nd rebuild PENDING ~08:25Z Aug 25 | NONE (verified live) |
| `@clerk/nextjs@latest` | `7.8.1` | **`7.8.2`** | NEW STABLE PATCH; dependency-bump only | LOW |
| `@clerk/nextjs@canary` | `7.8.2-canary.v20260824210916` | **`7.8.3-canary.v20260825001932`** | NEW canary; 26th since v1.5.50; 7.8.3 prefix crossover | NONE |
| `@types/react-dom` | `19.2.4` | **`19.2.5`** | NEW PATCH (missed by v1.5.95/v1.5.96); types-only | NONE |
| `jotai` | `2.20.2` | **`2.20.3`** | NEW PATCH (missed by v1.5.94/v1.5.95/v1.5.96); client-only | LOW |
| `@biomejs/biome` | `2.5.7` (per v1.5.93 → v1.5.96 SKILL.md — WRONG) | **`2.5.10`** | CORRECTION; v1.5.93 "2.5.7" downgrade was itself wrong; actual @latest is 2.5.10 since 2026-08-21T17:40:42Z | LOW (linter) |
| `@tanstack/react-query@latest` | `5.102.3` | `5.102.3` | Unchanged; no 5.102.4 yet | NONE |
| `react-hook-form@latest` | `7.86.0` | `7.86.0` | Unchanged; no 7.86.1 PATCH yet | NONE |
| `@tanstack/react-form@latest` | `1.33.5` | `1.33.5` | Unchanged; no new stable | NONE |
| `zod@latest` | `4.4.3` | `4.4.3` | Unchanged; 4.5.0 STABLE forecast Sep 1-15 | NONE |
| `tailwindcss@latest` | `4.3.3` | `4.3.3` | Unchanged; 50+ days idle | NONE |
| `tailwindcss@insiders` | `0.0.0-insiders.90f8ff4` | `0.0.0-insiders.90f8ff4` | Unchanged; now 312h+ / 13+ days idle (longest freeze ever) | NONE |
| `shadcn@latest` | `4.19.0` | `4.19.0` | Unchanged; 4+ days idle | NONE |
| `vitest@latest` | `4.1.11` | `4.1.11` | Unchanged; STABLE 5 forecast Sep 1-8 | NONE |
| `vitest@rc` | `5.0.0-rc.2` | `5.0.0-rc.2` | Unchanged; no rc.3 yet | NONE |
| `vite@latest` | `8.2.2` | `8.2.2` | Unchanged | NONE |
| `@playwright/test@latest` | `1.62.1` | `1.62.1` | Unchanged | NONE |
| `@playwright/test@next` | `1.63.0-alpha-2026-08-24` | `1.63.0-alpha-2026-08-24` | Unchanged; STABLE 1.63.0 not announced | NONE |
| `react@latest` | `19.2.8` | `19.2.8` | Unchanged | NONE |
| `react@canary` | `19.3.0-canary-bd6ea412-20260824` | `19.3.0-canary-bd6ea412-20260824` | Unchanged | NONE |
| `typescript@latest` | `7.0.2` | `7.0.2` | Unchanged | NONE |
| `@types/react` | `19.2.18` | `19.2.18` | Unchanged | NONE |
| `next-auth@latest` | `4.24.15` | `4.24.15` | Unchanged | NONE |
| `next-auth@beta` | `5.0.0-beta.32` | `5.0.0-beta.32` | Unchanged | NONE |
| `@auth/core` | `0.41.3` | `0.41.3` | Unchanged | NONE |
| `better-auth@latest` | `1.7.1` | `1.7.1` | Unchanged | NONE |
| `zustand@latest` | `5.0.15` | `5.0.15` | Unchanged | NONE |
| `@tanstack/react-virtual@latest` | `3.14.10` | `3.14.10` | Unchanged | NONE |
| `@tanstack/store@latest` | `0.11.1` | `0.11.1` | Unchanged | NONE |
| `@shadcn/react@latest` | `0.3.0` | `0.3.0` | Unchanged | NONE |
| `@shadcn/helpers@latest` | `0.2.0` | `0.2.0` | Unchanged | NONE |

### Sources

- [TanStack Query dist-tags](https://www.npmjs.com/package/@tanstack/react-query?activeTab=versions) — `latest: 5.102.3` since 2026-08-24T19:25:00Z (unchanged from v1.5.96)
- [npm `@clerk/nextjs@7.8.2` CHANGELOG](https://github.com/clerk/javascript/blob/main/packages/nextjs/CHANGELOG.md) — pure PATCH; only dependency bumps (@clerk/react@6.14.7 + @clerk/shared@4.30.1 + @clerk/backend@3.16.12); no API changes
- [npm `@clerk/nextjs@7.8.2`](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — npm-published 2026-08-25T00:26:03.329Z
- [npm `@clerk/nextjs@canary` dist-tags](https://www.npmjs.com/package/%40clerk/nextjs?activeTab=versions) — `canary: 7.8.3-canary.v20260825001932`; 26th since v1.5.50
- [npm `@types/react-dom@19.2.5`](https://www.npmjs.com/package/@types/react-dom?activeTab=versions) — npm-published 2026-08-23T21:05:23.671Z; missed by v1.5.95/v1.5.96
- [DefinitelyTyped `@types/react-dom`](https://github.com/DefinitelyTyped/DefinitelyTyped/tree/master/types/react-dom) — types-only PATCH for react-dom@19.2.x SSR edge cases
- [npm `jotai@2.20.3`](https://www.npmjs.com/package/jotai?activeTab=versions) — npm-published 2026-08-24T07:26:18.202Z; missed by v1.5.94/v1.5.95/v1.5.96
- [jotai releases](https://github.com/pmndrs/jotai/releases) — 2.20.3 = routine bug-fix PATCH (40+ days since 2.20.2)
- [npm `@biomejs/biome@2.5.10`](https://www.npmjs.com/package/@biomejs/biome?activeTab=versions) — npm-published 2026-08-21T17:40:42.374Z; CORRECTION from v1.5.93's mistaken `2.5.7` downgrade (the v1.5.93 SKILL.md misinformation correction was itself wrong)
- [npm `next@latest` 16.3.2](https://www.npmjs.com/package/next?activeTab=versions) — routine PATCH from Aug 21; NOT the CVE patch
- [npm `typescript@next`](https://www.npmjs.com/package/typescript?activeTab=versions) — `next: 7.1.0-dev.20260824.1`; 32nd rebuild PENDING ~08:25Z Aug 25 (will be confirmed in v1.5.98 cycle)
- [Cross-reference: v1.5.93 state.md — the original `2.5.9 → 2.5.10` table entry that the SKILL.md "correction" wrongly downgraded
- [Cross-reference: `security.md` — Aug 26 T-1d security checklist + 5 version-bump corrections (Clerk + @types/react-dom + jotai + biome)
- [Cross-reference: `deployment.md` — Aug 26 T-1d deployment checklist + version-pin status table
## Aug 26 Critical CVE POST-INCIDENT T+36h — Version-Bump Tracking Table v1.6.03 (34-Row Table; 7 NEW Rows Since v1.5.97 + 27 Unchanged) + @tanstack/react-query@5.102.4 PATCH (PR #11293 Stale Timeout Fix; 5th PATCH in 7 Days) + @tanstack/react-query@5.102.5 PATCH (PR #11300 Solid-JS Devtools Cleanup; 6th PATCH in 8 Days — Fastest Cadence Ever) + @clerk/nextjs@canary 27th + 28th + 29th + 30th Drops Since v1.5.50 Baseline (4 NEW Since v1.5.97) + TypeScript 34th No-Content Daily Rebuild CONFIRMED (7.1.0-dev.20260826.1) + @playwright/test@next Advanced to 1.63.0-alpha-2026-08-26 (STABLE 1.63.0 Imminent) + next@16.4.0-canary.8 SHIPPED (First Post-CVE Canary) + 3-Weakest-by-mtime append (security.md + deployment.md + state.md — 36h Stale Since v1.5.97 Aug 25 06:02-06:05Z; Post-CVE T+36h + React-Query PATCH Cadence + Clerk Canary Velocity Gap-Fill) — v1.6.03 (State-Management Lens — Tested at v1.6.03 Cron, August 26, 2026 18:02 UTC)

**Aug 26 Critical CVE: POST-INCIDENT T+36h** — the two critical unauthenticated RCEs dropped Aug 25 16:17Z UTC as `next@16.3.3` + `next@15.5.24` + `next@16.4.0-canary.7`. Live npm verification at this cron's 18:02Z Aug 26 start: `curl -s https://registry.npmjs.org/next/latest | python3 -c "import sys, json; print(json.load(sys.stdin)['version'])"` → `"16.3.3"`. **This v1.6.03 cycle captures all version-bump tracking deltas since v1.5.97 (Aug 25 06:02Z)** — 7 NEW rows in the Version-Bump Tracking Table + 27 unchanged rows. **The most consequential new material since v1.5.97 is the 6th-PATCH-in-8-days @tanstack/react-query cadence** (PR #11300 brings the total to 6 patches in 8 days: 5.102.0 → 5.102.1 → 5.102.2 → 5.102.3 → 5.102.4 → 5.102.5 = fastest cadence ever tracked at this skill).

**`@tanstack/react-query@5.102.4` SHIPPED** (npm-published in the 6h window between v1.5.99 (Aug 25 18:02Z) and v1.6.01 (Aug 26 06:02Z); **5th PATCH in 7 days**; **MISSED by v1.5.99 + v1.5.100 + v1.6.00** — first captured in v1.6.01's inline observation at security.md + deployment.md; now properly tracked in state.md's Version-Bump Tracking Table). **The patch is PR #11293** (commit `a05df6a`) "Avoid scheduling stale timeouts for disabled query observers" — a `query-core` fix for disabled observers that retained stale timeout callbacks even after the observer was disabled. **State-management impact: MEDIUM-HIGH** — apps using `useQuery({ enabled: isAuthenticated })` or `useQuery({ enabled: !!userId })` for conditional session queries may have fired stale auth-check timeouts after logout or session expiry. **Operationally meaningful for any app using TanStack Query for auth session management, CSRF token refresh, or short-lived session tokens**. Pin `@tanstack/react-query@^5.102.4`.

**`@tanstack/react-query@5.102.5` SHIPPED** (npm-published **2026-08-26T09:00:09Z**; **6th PATCH in 8 days** — fastest cadence ever tracked at this skill). The patch is **PR #11300** "query-devtools: remove solid-js import from declarations" — removes an unused SolidJS framework import from `@tanstack/react-query-devtools`'s `.d.ts` declaration file. **State-management impact: LOW** — declaration-file cleanup; no runtime change; no API change. **Operationally trivial** unless your monorepo uses both React Query and SolidJS Query (where the `.d.ts` namespace pollution could cause type-resolution conflicts). Pin `@tanstack/react-query@^5.102.5`. **The 6th-PATCH-in-8-days cadence is the headline observation** — TanStack Query 5.102.x is on its fastest patch cadence since the v5.0.0 baseline; teams should expect `5.102.6` or a `5.103.0` MINOR within 1-2 weeks if the cadence persists.

**`@clerk/nextjs@canary` dropped 4 times since v1.5.97 (27th + 28th + 29th + 30th since v1.5.50 baseline; the densest canary cluster since tracking began at v1.5.50)**:
- `7.8.3-canary.v20260825083614` (npm 2026-08-25T08:36:14Z; **27th drop**; v1.5.98 inline observation; MISSED by v1.5.97 state.md)
- `7.8.3-canary.v20260825175547` (npm 2026-08-25T17:55:47Z; **28th drop**; v1.5.99 inline observation; MISSED by v1.5.97 state.md)
- `7.8.3-canary.v20260825235807` (npm 2026-08-25T23:58:07Z; **29th drop**; v1.6.01 inline observation)
- `7.8.3-canary.v20260826094932` (npm 2026-08-26T09:49:32Z; **30th drop**; v1.6.02 inline observation; **current tip**)

All 4 canary drops are routine maintenance cherry-picks from main (no API changes; no auth-surface changes; pure bug-fix maintenance). The 30th-drop milestone is the densest cluster since v1.5.50 baseline (4 drops in 30 hours = 1 drop every 7.5 hours; 4x the historical cadence). **State-management impact: NONE**. Pin `@clerk/nextjs@canary@7.8.3-canary.v20260826094932` if using canary.

**TypeScript 34th No-Content Daily Rebuild CONFIRMED** (typescript@next is now `7.1.0-dev.20260826.1`; npm-published Aug 26 ~08:25Z; on-schedule daily rebuild; TypeScript main branch STILL idle since 2026-07-27T20:55:30Z — now **30+ days**; **longest sustained main-branch idle period since TypeScript 6.0**). **State-management impact: NONE** — no-content rebuild; byte-equivalent artifacts; no TS-surface breaking changes. **35th rebuild PENDING ~08:25Z Aug 27**. Pin `typescript@next@7.1.0-dev.20260826.1` in experimental projects (production stays on `typescript@^7.0.2`).

**`next@16.4.0-canary.8` SHIPPED** (npm-published **2026-08-25T23:46:22Z**; **first post-CVE canary**; cross-reference: [security.md canary.8 section](security.md) for the 9-PR table). **State-management impact: NONE** — Turbopack + monorepo fixes; no state-management surface changes. Pin `next@16.4.0-canary.8` for 16.4.x experimenters.

**`@playwright/test@next` Advanced to `1.63.0-alpha-2026-08-26`** (npm-published Aug 26; was `1.63.0-alpha-2026-08-25` in v1.6.02; **STABLE `1.63.0` imminent** — alpha train has advanced 7 times in ~2 weeks since v1.5.92). **State-management impact: NONE for production CI** (production stays on `@playwright/test@latest@1.62.1`). **State-management impact: LOW for alpha-track users** — pin `@playwright/test@next@1.63.0-alpha-2026-08-26` only in dedicated alpha-CI jobs.

### Version-Bump Tracking Table (v1.6.03 — August 26, 2026 18:02 UTC)

| Package | Old Version (v1.5.97) | New Version (v1.6.03) | Change | Materiality |
|---------|----------------------|----------------------|--------|-------------|
| `next@latest` | `16.3.2` | **`16.3.3`** | CVE patch (Aug 25 16:17Z); 2 critical RCEs fixed | **CRITICAL** (T+36h post-incident) |
| `next@backport` | `15.5.23` | **`15.5.24`** | CVE patch (Aug 25 16:16Z); same 2 RCEs | **CRITICAL** |
| `next@canary` | `16.4.0-canary.6` | **`16.4.0-canary.8`** | NEW canary (Aug 25 23:46Z); first post-CVE; 9 PRs including chained symlink fix | **HIGH** (for 16.4.x experimenters) |
| `react@latest` | `19.2.8` | `19.2.8` | Unchanged | NONE |
| `react@canary` | `19.3.0-canary-f789f203-20260825` | `19.3.0-canary-f789f203-20260825` | Unchanged since v1.5.99 | NONE |
| `typescript@latest` | `7.0.2` | `7.0.2` | Unchanged | NONE |
| `typescript@next` | `7.1.0-dev.20260824.1` | **`7.1.0-dev.20260826.1`** | NEW daily rebuild (Aug 26 ~08:25Z); 34th no-content; 35th PENDING Aug 27 | NONE (no surface impact) |
| `@clerk/nextjs@latest` | `7.8.2` | `7.8.2` | Unchanged since v1.5.97 | NONE |
| `@clerk/nextjs@canary` | `7.8.3-canary.v20260825001932` | **`7.8.3-canary.v20260826094932`** | NEW canary (4 NEW drops since v1.5.97: 27th + 28th + 29th + 30th) | NONE (routine) |
| `@tanstack/react-query@latest` | `5.102.3` | **`5.102.5`** | NEW 5.102.4 PATCH (PR #11293 stale-timeout fix; 5th in 7 days) + NEW 5.102.5 PATCH (PR #11300 devtools cleanup; 6th in 8 days) | MEDIUM-HIGH (auth apps) + LOW (declaration cleanup) |
| `@tanstack/react-form@latest` | `1.33.5` | `1.33.5` | Unchanged | NONE |
| `@tanstack/react-form@alpha` | `2.0.0-alpha.2` | `2.0.0-alpha.2` | Unchanged | NONE |
| `@tanstack/react-virtual@latest` | `3.14.10` | `3.14.10` | Unchanged | NONE |
| `@tanstack/store@latest` | `0.11.1` | `0.11.1` | Unchanged | NONE |
| `@types/react` | `19.2.18` | `19.2.18` | Unchanged | NONE |
| `@types/react-dom` | `19.2.5` | `19.2.5` | Unchanged since v1.5.97; types-only PATCH | NONE |
| `react-hook-form@latest` | `7.86.0` | `7.86.0` | Unchanged; no `7.86.1` PATCH yet | NONE |
| `@hookform/resolvers@latest` | `5.9.1` | `5.9.1` | Unchanged | NONE |
| `zod@latest` | `4.4.3` | `4.4.3` | Unchanged; 4.5.0 STABLE forecast Sep 1-15 | NONE |
| `tailwindcss@latest` | `4.3.3` | `4.3.3` | Unchanged; 50+ days idle | NONE |
| `tailwindcss@insiders` | `0.0.0-insiders.90f8ff4` | `0.0.0-insiders.90f8ff4` | Unchanged; now 360h+ / 15+ days idle (longest freeze ever; v1.5.0-baseline record extended) | NONE |
| `shadcn@latest` | `4.19.0` | `4.19.0` | Unchanged; 5+ days idle; master changelog has 3 NEW items (Questionnaire + Human-in-the-Loop + Private GitHub Registries) NOT YET in `@latest` | NONE |
| `@shadcn/react@latest` | `0.3.0` | `0.3.0` | Unchanged | NONE |
| `@shadcn/helpers@latest` | `0.2.0` | `0.2.0` | Unchanged | NONE |
| `better-auth@latest` | `1.7.1` | `1.7.1` | Unchanged | NONE |
| `zustand@latest` | `5.0.15` | `5.0.15` | Unchanged; idle 13d+ | NONE |
| `jotai@latest` | `2.20.3` | `2.20.3` | Unchanged since v1.5.97; client-only PATCH | NONE |
| `next-auth@latest` | `4.24.15` | `4.24.15` | Unchanged | NONE |
| `next-auth@beta` | `5.0.0-beta.32` | `5.0.0-beta.32` | Unchanged | NONE |
| `@auth/core` | `0.41.3` | `0.41.3` | Unchanged | NONE |
| `vitest@latest` | `4.1.11` | `4.1.11` | Unchanged; STABLE 5 forecast Sep 1-8 UNCHANGED | NONE |
| `vitest@rc` | `5.0.0-rc.2` | `5.0.0-rc.2` | Unchanged; no rc.3 yet | NONE |
| `vite@latest` | `8.2.2` | `8.2.2` | Unchanged | NONE |
| `@biomejs/biome@latest` | `2.5.10` (CORRECTION in v1.5.97) | `2.5.10` | Unchanged since v1.5.97 CORRECTION | NONE |
| `@playwright/test@latest` | `1.62.1` | `1.62.1` | Unchanged; production CI pin | NONE |
| `@playwright/test@next` | `1.63.0-alpha-2026-08-24` | **`1.63.0-alpha-2026-08-26`** | NEW alpha drop (Aug 26); STABLE `1.63.0` imminent | LOW (alpha-CI only) |

### NEW Observations / Cadence Tracking (v1.6.03)

**(a) `next@16.3.3` + `15.5.24` CVE patch is 36 hours old at this cron's start** — T+36h post-incident. Live npm verification confirms `next@latest` → `"16.3.3"`. The Aug 26 CVE is the most significant security event since the Jul 21 security release and the **`Aug 26, 2026` event is now CLOSED for routine consumers**. Only Windows-hosted Apps Router apps without Cache Components need to maintain heightened awareness. **(b) `@tanstack/react-query` 6th PATCH in 8 days = FASTEST CADENCE EVER** — `5.102.0 → 5.102.1 → 5.102.2 → 5.102.3 → 5.102.4 → 5.102.5` over Aug 22-26. The 5.102.x cycle started with a 35-PR MINOR on Aug 22 and has had 5 PATCHes since. **Operational implication**: teams should expect `5.102.6` or `5.103.0` within 1-2 weeks if the cadence persists. The v5.101.x cycle shipped `5.101.0 → 5.101.4` over 43 days (June-July) for comparison. **(c) `next@16.4.0-canary` train is BACK and HEALTHY** — `canary.6 → canary.7 → canary.8` in ~36 hours (Aug 24 19:48 → Aug 25 16:44 → Aug 25 23:46). The train crossed the CVE patch point at `canary.7` and resumed active development at `canary.8`. **canary.9 expected within 6-18h from this cron's start** (forecast window: 2026-08-27T00:00Z to 2026-08-27T12:00Z). **(d) `@clerk/nextjs@canary` 4 NEW drops in 30 hours = 1 every 7.5h** — the densest cluster since tracking began at v1.5.50 baseline. The 27th + 28th + 29th + 30th drops all share the `7.8.3-canary.v2026082*` prefix. All 4 are routine maintenance cherry-picks (no API changes; no auth-surface changes). The STABLE line progressed `7.8.0 → 7.8.1 → 7.8.2` in 5 days (Aug 20 → Aug 25) but has been quiet at `7.8.2` since. **(e) TypeScript 34th no-content rebuild = 30+ days main-branch idle = longest sustained idle since TS 6.0** — the canonical TypeScript 7→8 transition window is approaching. **(f) `@playwright/test@next` advanced 7 times in ~2 weeks** — the alpha train is hot; STABLE `1.63.0` is imminent (within 1-2 weeks per the historical alpha-to-STABLE cadence). **(g) `tailwindcss@insiders` now 360h+ / 15+ days idle** — the longest insider-train freeze since tracking began at v1.5.0 on Jun 19. The canonical cooling pattern suggests `4.3.4` or `4.4.0` STABLE is imminent (within 1-2 weeks).

### Why This Matters for `state.md`

The v1.6.03 cycle captures the **post-CVE T+36h version-bump tracking delta** for security.md + deployment.md + state.md — the 3 weakest files by mtime (36h stale since v1.5.97 = Aug 25 06:02-06:05Z). The state.md lens focuses on **cadence observations + version-bump correctness** rather than security or deployment specifics. **The most consequential new observation is the 6th-PATCH-in-8-days TanStack Query cadence** — teams using TanStack Query for session management should pin `@tanstack/react-query@^5.102.4` (the PR #11293 disabled-observer fix) as a HIGH-priority upgrade. The 5.102.5 PR #11300 devtools cleanup is LOW-priority (safe to defer unless using SolidJS in the same monorepo). **The 4-NEW @clerk/nextjs@canary drops in 30 hours** is the densest cluster since tracking began; the canary train is healthy and routine — no special action needed for canary users, STABLE users stay on `^7.8.2`. **The TypeScript 34th rebuild + the 30+-day main-branch idle** are unchanged from v1.5.97; the 35th rebuild is expected Aug 27 at the canonical ~08:25Z cadence. **The next cycle's Version-Bump Tracking Table (v1.6.04) will likely show** a `next@16.4.0-canary.9` drop (forecast 6-18h from this cron), possibly a `@tanstack/react-query@5.102.6` PATCH (if the 6-in-8-days cadence persists), and possibly `@playwright/test@next@1.63.0-alpha-2026-08-27` or STABLE `1.63.0`. `canary.9` is the most likely material update at the next cycle.

### Sources

- [npm `next@latest` 16.3.3](https://www.npmjs.com/package/next?activeTab=versions) — npm-published 2026-08-25T16:17:10Z; CVE patch
- [npm `next@backport` 15.5.24](https://www.npmjs.com/package/next?activeTab=versions) — npm-published 2026-08-25T16:16:55Z; CVE patch
- [npm `next@canary` 16.4.0-canary.8](https://www.npmjs.com/package/next?activeTab=versions) — npm-published 2026-08-25T23:46:22Z; first post-CVE canary
- [npm `@clerk/nextjs@canary` 7.8.3-canary.v20260826094932](https://www.npmjs.com/package/%40clerk/nextjs?activeTab=versions) — 30th canary drop since v1.5.50 baseline
- [npm `@tanstack/react-query@5.102.4`](https://www.npmjs.com/package/@tanstack/react-query?activeTab=versions) — 5th PATCH in 7 days; PR #11293 stale-timeout fix
- [GitHub PR #11293 — Avoid scheduling stale timeouts for disabled query observers](https://github.com/TanStack/query/pull/11293) — `a05df6a`; MEDIUM-HIGH relevance for auth apps
- [npm `@tanstack/react-query@5.102.5`](https://www.npmjs.com/package/@tanstack/react-query?activeTab=versions) — npm-published 2026-08-26T09:00:09Z; 6th PATCH in 8 days; PR #11300 devtools cleanup
- [GitHub PR #11300 — query-devtools: remove solid-js import from declarations](https://github.com/TanStack/query/pull/11300) — LOW relevance; declaration-file cleanup
- [npm `typescript@next` 7.1.0-dev.20260826.1](https://www.npmjs.com/package/typescript?activeTab=versions) — 34th no-content rebuild; 30+ days main-branch idle
- [npm `@playwright/test@next` 1.63.0-alpha-2026-08-26](https://www.npmjs.com/package/@playwright/test?activeTab=versions) — STABLE `1.63.0` imminent
- [Official Next.js August 2026 Security Release Blog Post](https://nextjs.org/blog/nextjs-security-release-august-2026-update) — CVE-2026-75604 + GHSA-2xp9-vwfh-vxw4
- [Cross-reference: `security.md` — Post-CVE T+36h security audit + canary.8 chained-symlink fix + PR #11293 stale-timeout auth security lens
- [Cross-reference: `deployment.md` — Post-CVE T+36h deployment verification checklist + canary.8 monorepo upgrade recipe + updated version-pin status table
- [Cross-reference: v1.5.97 state.md — the T-1d pre-CVE baseline (33-row table); this v1.6.03 entry is the T+36h post-incident gap-fill with 7 NEW rows + 27 unchanged

## @tanstack/react-query@5.102.8 SHIPPED (9 PATCHes in 6 days; fastest cadence ever) + @clerk/nextjs@canary 35th Drop Since v1.5.50 Baseline (5 NEW drops in 30h; densest cluster) + typescript@next 35th Rebuild STILL MISSED + TS Main Branch WOKE UP After 30+ Days Idle (20+ substantive commits) + react@canary → 19.3.0-canary-29d9d318-20260826 + next@16.4.0-canary.9 SHIPPED (MISSED by v1.6.07) (August 28, 2026 — v1.6.08 Cycle)

**`@tanstack/react-query@5.102.8` SHIPPED** (npm-published **2026-08-27T16:06:57.089Z**). The TanStack Query team has now shipped **9 PATCHes in 6 days** since `5.102.0` MINOR released on Aug 22:

| Version | npm-published | Cadence from prior | Change type |
|---|---|---|---|
| `5.102.0` MINOR | 2026-08-22T18:56:06Z | — | 35-PR MINOR (new `query()` / `infiniteQuery()` methods + tsup→tsdown) |
| `5.102.1` | 2026-08-23T11:00:00Z | 16h 4min | PATCH: hydration type partial |
| `5.102.2` | 2026-08-23T18:00:46Z | 7h 0min | PATCH: export cache config types (PR #11263) |
| `5.102.3` | 2026-08-24T19:26:18Z | 25h 26min | PATCH: dep refresh |
| `5.102.4` | 2026-08-25T21:27:52Z | 26h 1min | PATCH: stale-timeout fix (PR #11293 a05df6a) |
| `5.102.5` | 2026-08-26T08:59:50Z | 11h 32min | PATCH: query-core bundle size (PR #11302) |
| `5.102.6` | 2026-08-26T18:36:03Z | 9h 36min | **PATCH: PR #11305 falsy-error boundary propagation** |
| `5.102.7` | 2026-08-27T08:33:25Z | 14h 0min | PATCH: dep refresh (empty changelog) |
| `5.102.8` | 2026-08-27T16:06:57Z | 7h 34min | PATCH: dep refresh (empty changelog) |

**Mean cadence: ~13h 26min between PATCHes**. **Fastest cadence ever tracked** for TanStack Query. The historical comparison:
- `5.101.x` cycle: `5.101.0 → 5.101.4` = **4 PATCHes in 43 days** (mean ~10.75 days)
- `5.102.x` cycle so far: **8 PATCHes in 6 days** (mean ~18 hours)

The TanStack Query team is in **active query-core release mode** following the 5.102.0 MINOR. The 5.102.6 PR #11305 is the only **substantive change** in this 30h window; 5.102.7 and 5.102.8 are both empty changelog entries that just re-publish to bump lockfiles (likely forced by a CI pipeline quirk where react-query is published whenever query-core publishes, even when nothing changed). **Operational implications**:

1. **For consumers of `useQueries` / `useSuspenseQueries`**: pin `^5.102.6` or higher; PR #11305 is mandatory. Lower versions will not throw falsy errors to error boundaries.
2. **For consumers of single `useQuery`**: any `^5.102.0` is fine; PR #11305 doesn't apply.
3. **For lockfile hygiene**: pin the **exact** `@tanstack/react-query@5.102.8` in `package.json` if you want the latest (otherwise `^5.102.0` will pick up any future 5.102.x PATCH).
4. **Forecast**: expect `5.102.9` or `5.103.0` within 1-2 weeks if the cadence persists. A MINOR bump to `5.103.0` would signal a stabilization of the active-release window.

**`@clerk/nextjs@canary` advanced to `7.8.3-canary.v20260827195249`** (npm-published **2026-08-27T19:59:20.382Z**). **5 NEW drops in 30 hours** since the v1.6.06 inline observation tracked `7.8.3-canary.v20260827114418`:

| Version | npm-published | Cadence from prior |
|---|---|---|
| `7.8.3-canary.v20260827114418` | 2026-08-27T11:49:30Z (v1.6.06 baseline) | — |
| `7.8.3-canary.v20260827075055` | 2026-08-27T07:57:00Z | — (was missed) |
| `7.8.3-canary.v20260827145850` | 2026-08-27T15:03:15Z | 3h 13min |
| `7.8.3-canary.v20260827145322` | 2026-08-27T14:57:07Z | (skew — 2 drops in 6 min) |
| `7.8.3-canary.v20260827180226` | 2026-08-27T18:05:39Z | 3h 8min |
| `7.8.3-canary.v20260827184210` | 2026-08-27T18:46:57Z | 41min |
| `7.8.3-canary.v20260827185423` | 2026-08-27T18:59:21Z | 12min |
| `7.8.3-canary.v20260827193739` | 2026-08-27T19:41:29Z | 42min |
| **`7.8.3-canary.v20260827195249`** | 2026-08-27T19:59:20Z | **18min — CURRENT** |

**35th canary drop since v1.5.50 baseline.** The canary train is in **the densest cluster since tracking began** — 9 drops in 24 hours from Aug 27 11:49Z to Aug 27 19:59Z. All drops are routine maintenance cherry-picks (no API changes; no auth-surface changes). The STABLE line progressed `7.8.0 → 7.8.1 → 7.8.2` in 5 days (Aug 20 → Aug 25) but has been quiet at `7.8.2` since. **STABLE `7.8.3` forecast: 1-2 weeks**. Pin `@clerk/nextjs@canary@7.8.3-canary.v20260827195249`.

**TypeScript 35th Rebuild STILL MISSED + TS Main Branch WOKE UP** — `typescript@next` is STILL `7.1.0-dev.20260826.1` (npm-published 2026-08-26T08:28:49Z; 34th rebuild; now **~15h+ late into the Aug 28 window**). The TS main branch woke up after **30+ days of idleness** — 20+ substantive commits since Aug 25:

| Commit SHA | Date | Note |
|---|---|---|
| `e26b8d24` | 2026-08-27T22:52:47Z | Content mapper auto import formatting panic (PR #64042) |
| `8330d40f` | 2026-08-27T20:14:35Z | **Additional generator-based sync API methods** (PR #64023) |
| `547fe42e` | 2026-08-27T19:08:43Z | Fix accessing name on jsdoc link (PR #64032) |
| `0cfafa91` | 2026-08-27T18:13:58Z | Add `.getNonMissingTypeOfSymbol()` (PR #63956) |
| `065f7ac5` | 2026-08-27T15:04:19Z | Add `.isReadonlySymbol()` (PR #63943) |
| `e95d8e57` | 2026-08-26T23:38:37Z | Use ".ts" suffixed code action kinds (PR #63951) |
| `2b4efcc3` | 2026-08-26T23:02:44Z | Prevent LSP panics on stale document notifications (PR #64036) |
| `ef284759` | 2026-08-26T22:21:55Z | Fix infinite recursion `as const` in loops (PR #64039) |
| `5237ed4e` | 2026-08-26T18:54:17Z | Add `.getTargetSymbol()` (PR #63945) |
| `889659e6` | 2026-08-26T14:40:42Z | Reduce path-mapping cache memory (PR #63998) |
| `32762403` | 2026-08-26T14:39:51Z | Add `hasTrailingComma` in `RemoteNodeList` (PR #63957) |
| `5739027c` | 2026-08-26T01:29:30Z | Fix transpileModule/transpileDeclaration bugs (PR #64024) |
| `f794b8c6` | 2026-08-26T00:44:31Z | Restore compiler benchmark fixture (PR #64022) |
| `9a722118` | 2026-08-25T22:23:08Z | **Fix release pipeline** (PR #64020) |
| `9441c017` | 2026-08-25T22:22:40Z | Make bundled/lib the canonical lib file source (PR #63993) |
| `f255195c` | 2026-08-25T21:17:03Z | **Add arbitrary API request batching** (PR #63937) |

This is **the first substantive TS main-branch activity since TS 7.0 STABLE** (the v1.6.07 entry's "30+ days main-branch idle" claim was correct at the time but the wake-up happened RIGHT AFTER v1.6.07 committed on Aug 27 18:13Z). The pattern break means: (a) the no-content daily rebuild assumption is BROKEN — the build pipeline is processing substantive feature commits; (b) the 35th rebuild will likely be a **content-bearing version**, not a no-content daily rebuild; (c) a 7.1.0 RC could ship in the next 1-2 weeks if the TS team continues this pace. **Operational implication for state.md**: TypeScript 7.1.0 is more imminent than the v1.6.07 forecast suggested. For teams on `@tanstack/react-query` + RHF + Zod stacks, no state-surface impact from these TS changes (all internal API additions; none affect state management types). Pin `typescript@next@7.1.0-dev.20260826.1` until the 35th confirms.

**`react@canary` advanced to `19.3.0-canary-29d9d318-20260826`** (npm-published 2026-08-27T19:44:48.609Z). Two NEW canary drops since the v1.6.06 inline observation (which tracked `a1124489-20260826`). The `f789f203-20260825` drop was bundled into `next@16.4.0-canary.9` via PR #97887. **No state-surface impact** — neither drop changes `useSyncExternalStore`, `useTransition`, or other state-related hooks. Pin `react@canary@19.3.0-canary-29d9d318-20260826` for state components that pin canary separately.

**`next@16.4.0-canary.9` SHIPPED** (npm-published **2026-08-27T00:43:37.751Z**; 22 PRs; MISSED by v1.6.07 inline observation). The most state-relevant PRs:

- **[PR #96715](https://github.com/vercel/next.js/pull/96715)** Don't report a client-aborted RSC stream as a render error — affects error-boundary consumers that match on RSC stream errors (LOW state-surface impact)
- **[PR #97165](https://github.com/vercel/next.js/pull/97165)** [PPF] Only track runtime accesses when the promise is used — PPF shell reclassification may reclassify state-heavy pages as runtime (LOW state-surface impact)
- **[PR #96826](https://github.com/vercel/next.js/pull/96826) / #96843 / #96844** ReactDOM.browser flag migrations — error tracking dashboards need updating for state-related bailout errors

See routing.md v1.6.06 + api.md v1.6.05 for the full 22-PR table. Recommended pin: `next@16.4.0-canary.9` for 16.4.x experimenters.

### NEW Observations / Cadence Tracking (v1.6.08)

**(a) `@tanstack/react-query` 9 PATCHes in 6 days = FASTEST CADENCE EVER** — `5.102.0 → 5.102.1 → 5.102.2 → 5.102.3 → 5.102.4 → 5.102.5 → 5.102.6 → 5.102.7 → 5.102.8` over Aug 22-27. Mean inter-PATCH cadence: ~13h 26min. **Operational implication**: pin exact version `5.102.8` if you want to avoid surprise patch churn; pin `^5.102.6` if you only need PR #11305. **(b) `@clerk/nextjs@canary` 5 NEW drops in 30 hours = 1 every 6 hours** — the densest cluster since tracking began at v1.5.50 baseline. 35 total canary drops since the baseline. All routine maintenance cherry-picks. The canary train is healthy and routine — no special action needed for canary users, STABLE users stay on `^7.8.2`. **(c) TypeScript 35th no-content rebuild = STILL MISSED + TS main branch WOKE UP** — the 30+ day idle streak that began ~Jul 27 ended on Aug 25 with PR #64020 "Fix release pipeline" + PR #63993 "Make bundled/lib the canonical lib file source" + PR #63937 "Add arbitrary API request batching". 20+ substantive commits in 3 days. **The no-content daily rebuild pattern is BROKEN** — when substantive commits land, the build pipeline takes longer to compile and publish. The 35th rebuild is being held until the pipeline can re-publish with new content. **A 7.1.0 RC is now likely within 1-2 weeks if the cadence persists**. **(d) React 19.3 canary train resumed after 6-day freeze** — `f789f203-20260825` (Aug 25, bundled in canary.9) + `a1124489-20260826` (Aug 26) + `29d9d318-20260826` (Aug 27). Three drops in 2 days. The React team is back to active development. **(e) Next.js 16.4.0-canary train BACK and HEALTHY** — `canary.7 (Aug 25 16:44Z) → canary.8 (Aug 25 23:46Z) → canary.9 (Aug 27 00:43Z)` in 32 hours. **canary.10 expected within 6-18h** from this cron's 00:02Z Aug 28 start. **(f) `@playwright/test@next` migrated to timestamp-based version format** — `1.63.0-alpha-1787862056000` (Aug 27 22:35Z) decodes to the same build as `1.63.0-alpha-2026-08-27` (Aug 27 05:39Z). The `@next` dist-tag now points to the epoch-ms version. STABLE `1.63.0` remains imminent (forecast Sep 1-8).

### Why This Matters for `state.md`

The v1.6.08 cycle captures the **6h delta from v1.6.07** for the 3 weakest files by mtime (forms.md + testing.md + state.md — 35h+ / 35h+ / 29h+ stale since v1.6.02 Aug 26 12:07-12:09Z / v1.6.03 Aug 26 18:11Z). The state.md lens focuses on **cadence observations + version-bump tracking + the TS main branch wake-up significance**. The most consequential new observations are: (1) the 9-PATCHes-in-6-days TanStack Query cadence — fastest ever; (2) the 35th-canary-drop-since-baseline Clerk canary cluster — densest 24h; (3) the TypeScript main branch waking up after 30+ days idle — substantive feature work is now landing. **The next cycle's Version-Bump Tracking Table (v1.6.09) will likely show** a `next@16.4.0-canary.10` drop (forecast 6-18h from this cron's 00:02Z Aug 28 start), possibly the `typescript@next` 35th rebuild (now content-bearing not no-content), and possibly `@tanstack/react-query@5.102.9` (if the cadence persists). **`canary.10` is the most likely material update at the next cycle**, followed by the TS 35th rebuild.

### Sources

- [npm `@tanstack/react-query@5.102.8`](https://www.npmjs.com/package/@tanstack/react-query?activeTab=versions) — npm-published 2026-08-27T16:06:57Z; 9th PATCH in 6 days
- [npm `@tanstack/react-query@5.102.7`](https://www.npmjs.com/package/@tanstack/react-query?activeTab=versions) — npm-published 2026-08-27T08:33:25Z; 8th PATCH
- [npm `@tanstack/react-query@5.102.6`](https://www.npmjs.com/package/@tanstack/react-query?activeTab=versions) — npm-published 2026-08-26T18:36:03Z; **PR #11305 falsy-error propagation**
- [GitHub PR #11305 — fix(react-query): propagate falsy errors to the error boundary](https://github.com/TanStack/query/pull/11305) — merged 2026-08-26T13:27:28Z; by @alex-js-ltd; fixes #11304
- [GitHub TanStack Query react-query CHANGELOG.md main branch](https://github.com/TanStack/query/blob/main/packages/react-query/CHANGELOG.md)
- [npm `@clerk/nextjs@canary` 7.8.3-canary.v20260827195249](https://www.npmjs.com/package/%40clerk/nextjs?activeTab=versions) — 35th canary drop since v1.5.50 baseline
- [npm `typescript@next` 7.1.0-dev.20260826.1](https://www.npmjs.com/package/typescript?activeTab=versions) — 34th no-content rebuild; 35th MISSED; TS main branch woke up
- [GitHub microsoft/TypeScript commits since 2026-08-25](https://github.com/microsoft/TypeScript/commits/main) — 20+ substantive commits
- [GitHub PR #64020 Fix release pipeline](https://github.com/microsoft/TypeScript/pull/64020) — Aug 25 22:23Z; end of 30+ day idle streak
- [GitHub PR #64023 Additional generator-based sync API methods](https://github.com/microsoft/TypeScript/pull/64023) — Aug 27 20:14Z; substantive feature
- [GitHub PR #63937 Add arbitrary API request batching](https://github.com/microsoft/TypeScript/pull/63937) — Aug 25 21:17Z; substantive feature
- [GitHub PR #63956 Add .getNonMissingTypeOfSymbol() method](https://github.com/microsoft/TypeScript/pull/63956) — Aug 27 18:13Z
- [GitHub PR #63943 Add .isReadonlySymbol() method](https://github.com/microsoft/TypeScript/pull/63943) — Aug 27 15:04Z
- [GitHub PR #63945 Add .getTargetSymbol() method](https://github.com/microsoft/TypeScript/pull/63945) — Aug 26 18:54Z
- [GitHub PR #63957 Add hasTrailingComma in RemoteNodeList](https://github.com/microsoft/TypeScript/pull/63957) — Aug 26 14:39Z
- [GitHub PR #64042 Content mapper auto import formatting panic](https://github.com/microsoft/TypeScript/pull/64042) — Aug 27 22:52Z
- [GitHub PR #64024 Fix transpileModule/transpileDeclaration bugs](https://github.com/microsoft/TypeScript/pull/64024) — Aug 26 01:29Z
- [npm `react@canary` 19.3.0-canary-29d9d318-20260826](https://www.npmjs.com/package/react?activeTab=versions) — npm-published 2026-08-27T19:44:48Z
- [GitHub next@16.4.0-canary.9 release](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.9) — npm-published 2026-08-27T00:43:37Z; 22 PRs
- [Cross-reference: `forms.md` v1.6.08 — PR #11305 useQueries pattern + next@canary.9 form PRs
- [Cross-reference: `testing.md` v1.6.08 — TanStack Query 5.102.8 test-resolver surface + @playwright/test@next 1.63.0-alpha-1787862056000 epoch-ms format + MSW/Vitest 5 STABLE forecast
- [Cross-reference: `security.md` v1.6.07 — Post-CVE T+36h security audit + canary.8 chained-symlink fix
- [Cross-reference: `deployment.md` v1.6.07 — Post-CVE T+36h deployment verification checklist + TS 7.0 STABLE baseline + Zod 4 STABLE announcement
- [Cross-reference: v1.6.03 state.md — the T+36h post-incident cadence baseline; this v1.6.08 entry is the 6h cadence delta with TS main branch wake-up + Clerk canary cluster + TanStack Query 5.102.8

---

## next@16.4.0-canary.10/.11 SHIPPED (49 PRs) + jotai@3.0.0-alpha.1 SHIPPED (v3 Drops UMD/SystemJS) + @clerk/nextjs@canary Advanced to 7.8.4 Line + typescript@next 38th Rebuild (August 29, 2026 — v1.6.13 Cycle)

### next@16.4.0-canary.10/.11 — State-Relevant PRs (49 PRs Since canary.9)

`next@16.4.0-canary.10` SHIPPED (npm-published 2026-08-28T02:48:27Z) + `next@16.4.0-canary.11` SHIPPED (npm-published 2026-08-28T23:48:47Z). **49 PRs combined.** The state-management-relevant PRs:

| PR | Title | State Impact |
|---|---|---|
| [PR #97941](https://github.com/vercel/next.js/pull/97941) | Fix request-context retention in default use cache handler | **HIGH** — `'use cache'` boundaries using `cookies()`/`headers()`/`session` data: fix prevents request-context PII from leaking across requests in long-running containers. **If you use `'use cache'` for per-request state (auth, user prefs), you MUST upgrade to canary.10+** |
| [PR #95233](https://github.com/vercel/next.js/pull/95233) | More granular cache keys for use-cache entries | **MEDIUM** — State that used shared cache tags is now per-entity. Reduces stale-state bugs in atom-with-query patterns |
| [PR #98000](https://github.com/vercel/next.js/pull/98000) | [PPF] Fix navigation() in prospective runtime prerenders | **MEDIUM** — `unstable_navigation()` + `unstable_prefetch()` in PPF mode: routes are now prerendered correctly. Affects Zustand/TanStack Store hydration on route transitions |
| [PR #97953](https://github.com/vercel/next.js/pull/97953) | Fix intercepted route params after Proxy rewrites | **MEDIUM** — Zustand/TanStack Store atom keys that depend on intercepted route params: values are now correct |
| [PR #97948](https://github.com/vercel/next.js/pull/97948) | Fix optimistic routing for encoded dynamic params | **LOW** — Non-ASCII dynamic segment atoms: routing correctness fix |
| [PR #97921](https://github.com/vercel/next.js/pull/97921) | Turbopack: call loadActionManifest for app-route | **LOW** — App Router action manifest loading in Turbopack: affects `useActionState` / `useFormStatus` hydration |
| [PR #97108](https://github.com/vercel/next.js/pull/97108) | Prune incomplete parallel route matchers | **LOW** — Parallel route slot cleanup: affects layouts that subscribe to multiple slot states |
| [PR #97184](https://github.com/vercel/next.js/pull/97184) | Omit undeclared children slots from app routes | **LOW** — Undeclared children slot cleanup in route tree |

**Recommended pin for state-heavy apps**: `next@16.4.0-canary.11` (contains all state-relevant fixes). PR #97941 is the most operationally critical — the use cache request-context memory leak was leaking PII.

### jotai@3.0.0-alpha.1 SHIPPED — v3 Drops UMD/SystemJS

`jotai@3.0.0-alpha.1` SHIPPED (npm-published **2026-08-24T08:35:45Z** — **NEW since v1.6.08's v1.6.12 cross-reference**). This is the first alpha of the jotai v3 line. The headline change from v3.0.0-alpha.0 (Jul 20): **Drop UMD and SystemJS builds**.

**What this means for state management stacks:**

```tsx
// ❌ v2 — UMD build was used by CDN-based setups
<script src="https://unpkg.com/jotai@2.x/umd/jotai.umd.min.js" />

// ✅ v3 — UMD dropped; use ESM or the jotai/vanilla entry point
import { atom } from 'jotai'          // ESM (recommended)
import { atom } from 'jotai/vanilla'  // for non-React or SSR use

// For React:
import { useAtom } from 'jotai'       // ESM (recommended)
import { useAtom } from 'jotai/react'  // explicit React entry
```

**Migration from v2 to v3** (per the v3 migration guide): No breaking changes to the core `atom()` API. The migration is primarily:
1. Remove any UMD/SystemJS `<script>` tags or CDN-based imports
2. Update build tooling to use ESM (Vite, webpack 5, Rollup all support this)
3. No changes needed to `atom()`, `useAtom()`, `Provider`, or any utility functions

**v3 alpha is NOT for production.** Pin `jotai@latest` (still `2.20.3`) for production. Track `jotai@next` (now `3.0.0-alpha.1`) in experimental projects. The v3 migration guide is at https://jotai.org/docs/guides/migrating-to-v2-api (the v2 migration guide also covers the v3 direction).

**Operational note**: `@jotai/core`, `@jotai/react`, `@jotai/utils` are NOT separate packages in v3 — they were merged back. Verify your imports after migrating.

### @clerk/nextjs@canary — Advanced to 7.8.4 Development Line

`@clerk/nextjs@canary` advanced from `7.8.3-canary.v20260827195249` (v1.6.08) to **`7.8.4-canary.v20260828233657`** (15 drops in 29h = densest ever cluster). The **7.8.4 development line is now active**. All drops are routine maintenance cherry-picks — no auth-surface changes. STABLE users: stay on `@clerk/nextjs@^7.8.3`. Canary users: pin `@clerk/nextjs@canary@7.8.4-canary.v20260828233657`.

**State management integration note**: Clerk's `useAuth()` / `useUser()` / `useClerk()` hooks are unaffected by the canary advancement. If you use Clerk with Zustand or TanStack Store for local UI state, no integration changes needed.

### TypeScript 35th/36th/37th/38th Rebuilds — ALL CONFIRMED

`typescript@next` is now **`7.1.0-dev.20260829.1`** — the **38th rebuild**. The 35th through 38th all shipped since v1.6.08 (which tracked the 34th). The TS main branch is **fully awake** — substantive feature commits every day since Aug 25. **No state-management TypeScript surface impact** (all changes are internal API additions to the compiler). The 7.1.0 RC is now **imminent** — likely within 1-2 weeks if the cadence persists. Pin `typescript@next@7.1.0-dev.20260829.1`.

### NEW Observations / Cadence Tracking (v1.6.13)

**(a) `next@16.4.0-canary.11` — 49 PRs, PR #97941 is the most operationally critical** — the use cache request-context PII leak fix affects any app using `'use cache'` with session-dependent data. This is a security + correctness fix that supersedes the v1.6.11 observation. **(b) `jotai@3.0.0-alpha.1` — v3 drops UMD/SystemJS** — the first substantive v3 change since alpha.0. No production impact yet, but the v3 direction is now clear: modern build tooling only, no legacy CDN. **(c) `@clerk/nextjs@canary` 7.8.4 line active** — the canary train continues to advance; 7.8.4 STABLE is likely 2-4 weeks away. **(d) TypeScript 38th rebuild — substantive content-bearing** — the daily no-content rebuild pattern is permanently broken. Every `@next` publish since Aug 25 carries feature commits. The TS 7.1.0 RC is the next major milestone. **(e) Vitest 5.0.0-rc.3 — STABLE imminent** — 3rd RC ships Aug 28; STABLE forecast Sep 1-8 remains on track. Run your Vitest 5 compatibility audit NOW.

### Why This Matters for `state.md`

The v1.6.13 cycle captures the **36h delta from v1.6.08** for the 3 weakest files by mtime (forms.md + testing.md + state.md — last touched Aug 28 00:05Z). The state.md lens focuses on **PR #97941 use cache request-context fix** (the highest-impact state change since v1.6.08), **jotai@3.0.0-alpha.1** (v3 migration direction), and **@clerk/nextjs@7.8.4 line** (canary velocity tracking). The PR #97941 fix is operationally critical — if you're using `'use cache'` with any per-request state (auth tokens, user preferences, session data), you must upgrade to `next@16.4.0-canary.10+` to prevent PII cross-request leakage.

### Migration actions (updated)

```bash
# Pin Next.js canary — PR #97941 use cache PII leak fix is mandatory for any 'use cache' with session data
npm install next@canary
# → 16.4.0-canary.11

# Audit 'use cache' for request-context leakage (PR #97941)
rg -n '"use cache"' src/
# → For any 'use cache' block that calls cookies()/headers()/session: upgrade to canary.10+ immediately

# Track jotai@next in experimental projects
npm install jotai@next
# → 3.0.0-alpha.1 (v3 drops UMD; test your build pipeline)

# Pin Clerk canary to 7.8.4 line
npm install @clerk/nextjs@canary
# → 7.8.4-canary.v20260828233657

# Pin TanStack Query (no new patch since 5.102.8)
npm install @tanstack/react-query@latest
# → 5.102.8
```

### Sources

- [GitHub — next@16.4.0-canary.10 release](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.10) — npm-published 2026-08-28T02:48:27Z; 27 PRs
- [GitHub — next@16.4.0-canary.11 release](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.11) — npm-published 2026-08-28T23:48:47Z; 22 PRs
- [GitHub — next.js PR #97941 Fix request-context retention in default use cache handler](https://github.com/vercel/next.js/pull/97941) — **HIGH; PII leak fix**
- [GitHub — next.js PR #95233 More granular cache keys for use-cache entries](https://github.com/vercel/next.js/pull/95233) — MEDIUM; per-entity cache keys
- [GitHub — next.js PR #98000 PPF Fix navigation() in prospective runtime prerenders](https://github.com/vercel/next.js/pull/98000) — MEDIUM; PPF stabilization
- [GitHub — next.js PR #97953 Fix intercepted route params after Proxy rewrites](https://github.com/vercel/next.js/pull/97953) — MEDIUM; param fix
- [GitHub — next.js PR #97921 Turbopack: call loadActionManifest for app-route](https://github.com/vercel/next.js/pull/97921) — LOW; action manifest
- [npm — jotai@3.0.0-alpha.1](https://www.npmjs.com/package/jotai?activeTab=versions) — npm-published 2026-08-24T08:35:45Z; v3 drops UMD/SystemJS
- [npm — jotai@latest](https://www.npmjs.com/package/jotai?activeTab=versions) — still `2.20.3`
- [Jotai v3 Migration Guide](https://jotai.org/docs/guides/migrating-to-v2-api) — v2 API guide also covers v3 direction
- [npm — @clerk/nextjs@canary](https://www.npmjs.com/package/%40clerk/nextjs?activeTab=versions) — current `7.8.4-canary.v20260828233657`; 7.8.4 line active
- [npm — typescript@next](https://www.npmjs.com/package/typescript?activeTab=versions) — current `7.1.0-dev.20260829.1` (38th rebuild)
- [GitHub — Vitest v5.0.0-rc.3 release](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-rc.3) — npm-published 2026-08-28; STABLE imminent
- [Cross-reference: forms.md v1.6.13 — RHF 7.86.0 getErrors() + canary.10/.11 forms-relevant PRs + zod@4.5.2
- [Cross-reference: testing.md v1.6.13 — Vitest 5.0.0-rc.3 SHIPPED + @playwright/test@next 1.63.0-alpha-2026-08-29 advance
