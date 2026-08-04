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
