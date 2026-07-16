# Routing — Next.js App Router

## File-Based Routing

Next.js App Router uses filenames as routes:

| File | Route |
|---|---|
| `app/page.tsx` | `/` |
| `app/about/page.tsx` | `/about` |
| `app/blog/[slug]/page.tsx` | `/blog/:slug` |
| `app/(marketing)/layout.tsx` | Route groups (no URL impact) |
| `app/api/users/route.ts` | `/api/users` |
| `app/@modal/(.)photo/[id]/page.tsx` | Parallel routes |

## Route Groups

Use `(group)` to organize routes without affecting the URL:

```
app/
├── (marketing)/
│   ├── layout.tsx        # Shared layout for marketing pages
│   ├── about/page.tsx   # /about
│   └── pricing/page.tsx # /pricing
├── (app)/
│   ├── layout.tsx        # Shared layout for app pages (with auth)
│   ├── dashboard/page.tsx # /dashboard
│   └── settings/page.tsx # /settings
```

## Dynamic Routes

### Single Segment: `[slug]`

```tsx
// app/blog/[slug]/page.tsx
interface BlogPostPageProps {
  params: Promise<{ slug: string }>
}

export default async function BlogPostPage({ params }: BlogPostPageProps) {
  const { slug } = await params
  const post = await getPostBySlug(slug)
  
  if (!post) notFound()
  return <article>{post.content}</article>
}
```

**Note:** In Next.js 15, `params` is a `Promise` — always `await` it.

### Multiple Segments: `[...slug]`

```tsx
// app/docs/[...slug]/page.tsx
// Matches /docs/intro, /docs/api/auth, /docs/api/auth/login, etc.

export default async function DocPage({ params }: { params: Promise<{ slug: string[] }> }) {
  const { slug } = await params
  const path = slug.join('/')
  return <DocContent path={path} />
}
```

### Catch-All: `[[...slug]]`

```tsx
// Optional catch-all — matches /, /a, /a/b, etc.
// [[...slug]] vs [...slug]: the latter requires at least one segment
```

### `generateStaticParams` — Static Generation (Build Time)

For dynamic routes with known paths at build time, use `generateStaticParams` to pre-render those pages. This is faster than rendering on-demand:

```tsx
// app/blog/[slug]/page.tsx
interface Props {
  params: Promise<{ slug: string }>
}

// Tell Next.js which slugs to pre-render at build time
export async function generateStaticParams() {
  const posts = await getAllPostSlugs()  // Returns [{ slug: 'hello-world' }, { slug: '...' }]
  return posts
}

export default async function BlogPostPage({ params }: Props) {
  const { slug } = await params
  const post = await getPostBySlug(slug)
  
  if (!post) notFound()
  return <article>{post.content}</article>
}
```

**When to use `generateStaticParams`:**
- Blog posts, product pages, documentation — any content with a known set of URLs
- Export from a dynamic route segment (`[slug]`, `[...slug]`, etc.)
- Return value is an array of params objects: `[{ slug: 'value' }]` or `[{ slug: ['a', 'b'] }]` for catch-all routes

**Without `generateStaticParams`:**
- Dynamic pages render on first request (slower first visit)
- With `revalidate`, they get ISR — cached but still generated on demand

**With `generateStaticParams` + `revalidate`:**
```tsx
// Pre-rendered at build time, then revalidated every hour
export const revalidate = 3600

export async function generateStaticParams() {
  return (await getAllPostSlugs())
}
```

**Sources:**
- [Next.js generateStaticParams](https://nextjs.org/docs/app/api-reference/functions/generate-static-params)



## Layouts

### Root Layout

```tsx
// app/layout.tsx — required, wraps every page
import type { Metadata } from 'next'
import './globals.css'

export const metadata: Metadata = {
  title: 'My App',
  description: 'Description of my app',
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
```

### Nested Layouts

```tsx
// app/dashboard/layout.tsx — wraps all /dashboard/* pages
// Persists across navigation within dashboard

export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex h-screen">
      <Sidebar />
      <main className="flex-1">{children}</main>
    </div>
  )
}
```

## Dynamic Metadata (`generateMetadata`)

For metadata that depends on dynamic data (e.g., blog post titles, OG images), use `generateMetadata`:

```tsx
// app/blog/[slug]/page.tsx
import type { Metadata } from 'next'

interface Props {
  params: Promise<{ slug: string }>
}

// Generate metadata at build time or request time
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params
  const post = await getPostBySlug(slug)
  
  if (!post) return { title: 'Post not found' }
  
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [{ url: post.ogImage, width: 1200, height: 630 }],
    },
    twitter: {
      card: 'summary_large_image',
      title: post.title,
      description: post.excerpt,
      images: [post.ogImage],
    },
  }
}

export default async function BlogPostPage({ params }: Props) {
  const { slug } = await params
  const post = await getPostBySlug(slug)
  if (!post) notFound()
  return <article>{post.content}</article>
}
```

**Key pattern:** `params` in `generateMetadata` is also a `Promise` in Next.js 15 — always `await` it.

### `generateViewport` (Mobile Responsiveness)

```tsx
import type { Metadata, Viewport } from 'next'

export const viewport: Viewport = {
  themeColor: [
    { media: '(prefers-color-scheme: light)', color: '#ffffff' },
    { media: '(prefers-color-scheme: dark)', color: '#0a0a0a' },
  ],
  width: 'device-width',
  initialScale: 1,
}

// Or use generateViewport for dynamic viewport config
export function generateViewport({ params }: Props): Viewport {
  return {
    themeColor: '#4f46e5',
  }
}
```

## Route Segment Config

Control how a route is rendered and cached with export config:

```tsx
// app/api/health/route.ts
export const dynamic = 'force-dynamic'     // Always render at request time (no cache)
export const revalidate = 0                 // Revalidate every request (same as above)
```

```tsx
// app/blog/[slug]/page.tsx
export const revalidate = 3600  // Revalidate this page every hour (ISR)
```

**Options for `dynamic`:**
| Value | Behavior |
|---|---|
| `'auto'` | Default — Next.js decides based on data fetching |
| `'force-dynamic'` | Always render at request time (no cache) |
| `'force-static'` | Force static rendering (throws if you try to use dynamic data) |
| `'error'` | Error if dynamic data is accessed without explicitly opting out |

**Note:** `revalidate = 0` is equivalent to `dynamic = 'force-dynamic'`.

## Loading UI

```tsx
// app/dashboard/loading.tsx — shown while dashboard pages load
export default function Loading() {
  return <Skeleton className="h-32 w-full" />
}
```

This works with `<Suspense>` boundaries — the loading.tsx is the Suspense fallback for the content inside the layout.

## Error Handling

### Not Found

```tsx
// 404 — app/blog/[slug]/page.tsx
import { notFound } from 'next/navigation'

if (!post) notFound()
```

```tsx
// Custom not-found page — app/not-found.tsx
export default function NotFound() {
  return (
    <div className="flex flex-col items-center justify-center h-screen">
      <h1 className="text-4xl font-bold">404</h1>
      <p>This page doesn't exist.</p>
    </div>
  )
}
```

### Error Boundary

```tsx
// app/dashboard/error.tsx — client component for error handling
'use client'

import { useEffect } from 'react'

export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  useEffect(() => {
    // Log to error tracking service
    console.error(error)
  }, [error])

  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  )
}
```

## Parallel Routes

Render multiple routes simultaneously in the same layout using named **slots** (the `@folder` convention). Slots are passed as props to the shared parent layout:

```
app/
├── layout.tsx              # Receives { children, team, analytics } as props
├── @team/
│   ├── page.tsx            # /team (renders inside the @team slot)
│   └── settings/page.tsx   # /team/settings
├── @analytics/
│   ├── page.tsx            # /analytics (renders inside the @analytics slot)
│   └── default.tsx         # ⚠️ REQUIRED in Next.js 16 — see below
```

```tsx
// app/layout.tsx
export default function Layout({
  children,
  team,
  analytics,
}: {
  children: React.ReactNode
  team: React.ReactNode
  analytics: React.ReactNode
}) {
  return (
    <>
      {children}
      {team}
      {analytics}
    </>
  )
}
```

### `default.tsx` — Required in Next.js 16 (Breaking Change)

**Next.js 16 requires every parallel route slot to have an explicit `default.tsx` file** — without one, the build fails. This applies to all slots including ones that never need a fallback, because Next.js needs to know what to render when the user navigates to a URL that doesn't match the slot's current child.

```tsx
// app/@analytics/default.tsx
import { notFound } from 'next/navigation'

// Option A: 404 when the slot has no matching child
export default function Default() {
  notFound()
}

// Option B: render nothing (preserves the pre-Next.js 16 default behavior)
export default function Default() {
  return null
}
```

**Why this changed:** Without `default.tsx`, navigating to a URL that doesn't match a slot would either error or use stale slot content from a previous route — confusing users. The `default.tsx` makes the slot's fallback explicit and predictable.

**Rule of thumb:**
- Use `notFound()` for slots that should 404 when unmatched
- Use `return null` for slots that should disappear when unmatched (e.g., a modal slot)
- Use a loading skeleton for slots that should stream independently

### Independent Loading and Error States

Each parallel route can have its own `loading.tsx` and `error.tsx` — they stream independently:

```tsx
// app/@analytics/loading.tsx
export default function Loading() {
  return <AnalyticsSkeleton />
}

// app/@analytics/error.tsx
'use client'
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <p>Analytics failed to load</p>
      <button onClick={reset}>Retry</button>
    </div>
  )
}
```

## Intercepting Routes

Intercept a route to show a modal instead of navigating:

```
app/
├── (feed)/
│   ├── photo/[id]/page.tsx      # Full page: /photo/123
│   └── @modal/(.)photo/[id]/page.tsx  # Intercepted modal overlay
```

When visiting `/photo/123` from within the app, the modal version is shown. Direct navigation loads the full page.

## Navigation API

### Same-Document Traversals Before Hydration — REVERTED in canary.88 (PR [#95682](https://github.com/vercel/next.js/pull/95682) shipped in canary.87, reverted by PR [#95853](https://github.com/vercel/next.js/pull/95853) on 2026-07-16T12:19:58Z, canary-branch commit `3baea3576`)

**Heads up: this fix is in a flux state.** The original fix (PR #95682) shipped in canary.87 (2026-07-15T23:34:23Z) and was reverted on the canary branch on 2026-07-16T12:19:58Z by PR #95853. It is therefore **not** in canary.88. The reverting author (gaearon) noted: "This breaks CC ([#95848](https://github.com/vercel/next.js/issues/95848)) due to PPR codepath doing something different. Although Claude says [da4d63d7](https://github.com/vercel/next.js/commit/da4d63d759a9bffd3e8c586bac1e4dabffd5b35d) can work around it, I think it's safest to revert this given that I don't know what the right full fix is. I also think we need more test coverage because the navigation test suite is off in CC mode. I'll send a follow-up with all the failing tests we know of (from #95682 and new) so we can fix this for good after a closer look." The skill records both the original fix and the revert so anyone on canary.87–canary.88+ can reason about which behaviour they're seeing.

**The race** (pre-canary.87, and again post-#95853 on canary.88+ for `cacheComponents: true`): there is a class of silent navigation race that hits when you navigate to a route, hard-reload, and then hit Back before the new page hydrates. The repro is:

1. Open page A.
2. Navigate to page B (client-side `<Link>`).
3. Cmd+Shift+R on page B (hard reload — replaces the document with B's server-rendered HTML).
4. Hit Back super fast.

The router would then get confused and show page B's content with page A's URL. From then on, going Back and Forward changed the URL (and the tab title) but kept showing B's content. No error overlay, no console warning, no DevTools signal — the URL bar and the rendered content just stopped agreeing.

**Why it happens.** Step 2 creates B's history entry via `pushState`. Step 3 replaces the document with B's fresh server-rendered HTML (the browser's reload discards the JS heap). Step 4 fires `popstate` to A — but at that point nothing in the JS has loaded yet, so nothing listens. Then the JS loads and hydrates B's content onto B's HTML despite technically being on the A route.

**The original fix (canary.87 only).** canary.87 used the [Navigation API](https://developer.mozilla.org/en-US/docs/Web/API/Navigation_API) to detect that the current entry is not the entry the page was started with. Once the router mounted, it replayed the missed traversal the same way the existing `onPopState` handler would have handled it. The author (gaearon) noted at the time: "I originally tried to avoid using Navigation API but wasn't sure how to detect this reliably."

**Why the fix was reverted (issue #95848).** With `cacheComponents: true` AND a `<Suspense>` boundary around `children` in the root layout, the same scenario that the fix was meant to repair *regressed* into a **blank page** (regression introduced by #95682 itself). The reproducer is at [franky47/next-repro-back-before-hydration](https://github.com/franky47/next-repro-back-before-hydration); without CC the fix works correctly, with CC the traversal replay loses the deferred RSC payload because React's `Activity`-based segment cache retries don't fire when the Suspense boundary hadn't yet hydrated at the time of the replay. The reverted PR is currently in `[reverted]` state and gaearon has promised a follow-up with a full fix and proper test coverage. **If you are on `cacheComponents: true` and you do not need the original fix, you can downgrade to canary.86 to avoid both the URL/content desync AND the new blank-page regression — but you will lose the rest of canary.87's material fixes (Request Insights, debug-build-paths metadata, Asset Icons, etc.).**

**A standalone workaround commit exists** at [da4d63d759a9bffd3e8c586bac1e4dabffd5b35d](https://github.com/vercel/next.js/commit/da4d63d759a9bffd3e8c586bac1e4dabffd5b35d) (referenced by gaearon in the revert PR body as "Claude says this can work around it"). The change re-renders after a history traversal's dynamic data lands under cacheComponents: a history traversal commits its tree immediately and streams the dynamic data in afterwards, relying on React to retry the suspended render once the data resolves; under CC that retry is lost when the traversal rendered into a Suspense boundary that had not hydrated yet. The fix dispatches a restore of the current entry when a history-traversal task finishes with all data cached — a no-op when nothing was lost, and convergent because a fully cached restore spawns no dynamic requests. **Gated to `cacheComponents`**, where the `Activity`-based segment cache is in play. As of 2026-07-16T18:00Z this commit is not on the canary branch and is therefore not in any canary tag.

**What you need to know:**

- **If you are on canary.87 and you do NOT have `cacheComponents: true`** — the original #95682 fix is in effect; the URL/content desync after a hard-reload-then-Back is fixed. You can keep using canary.87 with confidence.
- **If you are on canary.87 and you DO have `cacheComponents: true`** — downgrade to canary.86 to avoid the new blank-page regression. If you must stay on canary.87, the only known mitigation is to remove the `<Suspense>` boundary around `children` in the root layout, but that defeats most of the cacheComponents benefits.
- **If you are on canary.88+** — the #95682 fix is reverted; you are back to the pre-canary.87 race. Watch for the follow-up PR(s) from gaearon and the da4d63d7 workaround landing on the canary branch.
- **If you maintain a custom router wrapper or a `<Link>` shim that swallows popstate events** — there is currently no replay behaviour to worry about. Once a follow-up fix lands, custom wrappers that observe popstate will get the replay for free; wrappers that intercept popstate and swallow it before the reducer sees it would also miss the replay (and historically this was the main reason wrapper authors hit the race).
- **If you use the Browser History directly via `window.history.pushState` / `replaceState`** — until the follow-up fix lands, the only safe options are `<Link>` / `router.push()` / `router.back()`. Direct `history` calls are not replayed.
- **If you're on 16.2.x or 16.3.0-canary.86 or earlier** — the race exists in those versions. Workaround before canary.87 was to add a one-line artificial delay before the Back click (a few hundred ms), which made the test non-deterministic enough to not reproduce reliably in practice. There was no real fix prior to canary.87, and there is no real fix in canary.88 either.

**Audit:** as of canary.88+, the #95682 fix is reverted and there is no shipped mitigation. If you are on `cacheComponents: true` and you care about this race, the cleanest interim answer is to stay on canary.86 and wait for gaearon's follow-up.

**Sources:**
- [PR #95682 — `Replay same-document traversals that happen before hydration`](https://github.com/vercel/next.js/pull/95682) · Commit `2ce362f312` · gaearon · merged 2026-07-15T17:35:16Z · **canary.87, REVERTED**
- [PR #95853 — `Revert "Replay same-document traversals that happen before hydration"`](https://github.com/vercel/next.js/pull/95853) · Commit `3baea3576` · gaearon · merged 2026-07-16T12:19:58Z · **canary.88+**
- [Issue #95848 — `Blank page when pressing Back between a reload and hydration with cacheComponents (regression in 16.3.0-canary.87)`](https://github.com/vercel/next.js/issues/95848) — reproducer at [franky47/next-repro-back-before-hydration](https://github.com/franky47/next-repro-back-before-hydration)
- [Commit `da4d63d7` — `Re-render after a history traversal's dynamic data lands under cacheComponents`](https://github.com/vercel/next.js/commit/da4d63d759a9bffd3e8c586bac1e4dabffd5b35d) — the Claude-authored workaround referenced in PR #95853's body; not yet on the canary branch as of 2026-07-16T18:00Z
- Original tests added in PR #95682 (red before the fix, red again with CC after the revert): `packages/next/src/client/components/segment-cache/tests/navigation-api-replay.test.ts` (route shape)

### `useRouter` (Client Component)

### `useRouter` (Client Component)

```tsx
'use client'
import { useRouter } from 'next/navigation'

export function BackButton() {
  const router = useRouter()
  return <button onClick={() => router.back()}>Go back</button>
}
```

### `redirect` (Server Component)

```tsx
import { redirect } from 'next/navigation'
import { auth } from '@/lib/auth'

export default async function AdminPage() {
  const session = await auth()
  if (!session?.user) redirect('/login')
  // ...
}
```

### `Link` Component

```tsx
import Link from 'next/link'

// Prefer <Link> over <a> for client-side navigation
<Link href="/blog/nextjs-15" className="text-primary hover:underline">
  Read more
</Link>
```

**Never use `<a href="...">` for internal links** — it causes a full page reload.

### `useSearchParams()` — Reading URL Query Params

`useSearchParams()` gives you access to the current URL's query string in a Client Component. It must be wrapped in a Suspense boundary if the page uses it:

```tsx
// app/search/page.tsx — Server Component wrapper
import { Suspense } from 'react'
import { SearchResults } from '@/components/search-results'

export default function SearchPage() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <SearchResults />
    </Suspense>
  )
}

// components/search-results.tsx
'use client'

import { useSearchParams } from 'next/navigation'

export function SearchResults() {
  const searchParams = useSearchParams()
  const query = searchParams.get('q') ?? ''
  const page = searchParams.get('page') ?? '1'
  
  return (
    <div>
      <p>Searching for: {query}</p>
      <p>Page: {page}</p>
    </div>
  )
}
```

**Why Suspense is required:**
- In Next.js 16, `useSearchParams()` in a Client Component opts the entire route segment into dynamic rendering
- Without a Suspense boundary, this causes the whole page to be dynamic — potentially hurting TTFB
- Wrapping in Suspense isolates the dynamic portion, letting the static shell render at build time

**Reading search params in Server Components:**
```tsx
// Server Components can read searchParams directly from page props
interface Props {
  searchParams: Promise<{ q?: string; page?: string }>
}

export default async function SearchPage({ searchParams }: Props) {
  const { q = '', page = '1' } = await searchParams
  // No Suspense needed — Server Components handle this natively
  return <SearchResults query={q} page={page} />
}
```

**Common pattern — URL-synced search:**
```tsx
'use client'

import { useRouter, useSearchParams } from 'next/navigation'

export function SearchInput() {
  const router = useRouter()
  const searchParams = useSearchParams()
  
  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    const params = new URLSearchParams(searchParams)
    if (e.target.value) {
      params.set('q', e.target.value)
    } else {
      params.delete('q')
    }
    router.push(`?${params.toString()}`, { scroll: false })
  }
  
  return (
    <input
      defaultValue={searchParams.get('q') ?? ''}
      onChange={handleChange}
      placeholder="Search..."
    />
  )
}
```

**Sources:**
- [Next.js useSearchParams](https://nextjs.org/docs/app/api-reference/hooks/use-search-params)

## Programmatic Navigation

```tsx
// Client-side navigation
router.push('/dashboard')
router.replace('/login') // replaces history entry

// With search params
router.push('/search?q=nextjs&page=2')

// Prefetch a route (preload on hover)
router.prefetch('/dashboard')
```

## Route Handlers vs Pages

| Route Handler | Page |
|---|---|
| `app/api/*/route.ts` | `app/*/page.tsx` |
| Handles API requests (REST/GraphQL) | Renders UI |
| Can use all HTTP methods | One export (default) |
| No UI rendering | Server or Client Component |

## Prefetch & Caching Optimizations (Next.js 16.2)

Next.js 16.2 added two experimental flags that significantly change how navigation data is fetched and cached. Both are **opt-in** and stable enough to use in production with `cacheComponents: true` enabled.

### `experimental.prefetchInlining` — One Request Per Link

By default, Next.js 16's per-segment prefetcher issues **separate requests** for each segment in the route tree (one for the leaf page, one for each slot, one for the layout). This keeps shared layouts cache-efficient but increases request volume. `experimental.prefetchInlining` bundles all segments for a route into **a single response**, reducing the number of prefetch requests to one per `<Link>`.

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  experimental: {
    prefetchInlining: true,
  },
}

export default nextConfig
```

**Trade-off:** Shared layout data is duplicated across inlined responses rather than being cached and reused. Use this flag when your app has many leaf pages with small segments — the per-link request reduction wins. Skip it when most of your routes share a heavy layout that benefits from dedup.

### `experimental.cachedNavigations` — Instant Repeat Visits

This flag independently controls the Cached Navigations behavior, which caches **static and dynamic** Server Component data from navigations and the initial HTML loads so that repeat visits to a previously-visited route serve instantly. Requires `cacheComponents: true`.

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  cacheComponents: true,
  experimental: {
    cachedNavigations: true,
    // 16.3.0-canary.61+ — also accepts 'allow-runtime' to widen the runtime stage
    // to every segment regardless of its per-segment prefetch config:
    //   cachedNavigations: 'allow-runtime',
  },
}

export default nextConfig
```

**`cachedNavigations` widened type (16.3.0-canary.61, [#95064](https://github.com/vercel/next.js/pull/95064), June 22, 2026):** The config is now `boolean | 'allow-runtime'`. `true` keeps the current behavior (static stage cached, runtime stage only for segments with `export const prefetch = 'allow-runtime'`). `'allow-runtime'` additionally treats every segment as runtime-cached regardless of its per-segment `prefetch` config — useful for dogfooding the CPU/payload overhead of the runtime stage across a whole app. The client-facing env var `__NEXT_EXPERIMENTAL_CACHED_NAVIGATIONS` stays a plain boolean; the runtime prerender is entirely server-driven, so the widening is only consumed in the two production spawn gates (`app-render.tsx` — the client-navigation Flight render and the initial HTML render). Prefetch hints, instant validation, and component-tree staging are intentionally still keyed off the per-segment config — the global flag is a cache-side override, not a prefetch-side one.

**`partialPrefetching` + per-segment `prefetch: 'partial'` / `'unstable_eager'` also opt into the runtime stage (16.3.0-canary.63, [#95097](https://github.com/vercel/next.js/pull/95097), June 24, 2026):** The runtime-cache path for cached navigations used to fire only when a segment exported `prefetch = 'allow-runtime'`. Canary.63 broadens that gate via a new `anySegmentHasPartialPrefetchingEnabled` helper — the two runtime-prefetch spawn gates in `app-render.tsx` now also match when:

- The new global `experimental.partialPrefetching: true` flag is set (opts every route in), or
- Any segment in the matched route exports `prefetch = 'partial'` / `'unstable_eager'` (PPR-style partial prefetching).

Tests cover `prefetch = 'partial'` and `prefetch = 'unstable_eager'` fixtures and a new `partial-prefetching` fixture for the global flag. Each asserts the route runtime-caches its request-derived content on a second navigation without a per-segment `prefetch = 'allow-runtime'` export. The narrower callers that gate metadata staging and the dev reveal stage keep using the existing `anySegmentHasRuntimePrefetchEnabled` because they specifically concern `'allow-runtime'` (which the client runtime-prefetches), not partial prefetching. Apps that do **not** set `experimental.partialPrefetching` and have no partial-prefetch segments are unaffected — `partialPrefetching` has no default and is not auto-enabled by `cacheComponents`.

**Dev fallback `params` computed from the most-specific matching route (16.3.0-canary.63, [#95066](https://github.com/vercel/next.js/pull/95066), June 24, 2026):** When Cache Components is enabled in dev, the server threads a `fallbackParams` request meta so the staged render knows which params are not statically known. The previous computation walked the prerendered routes from `getStaticPaths` and picked the one with the fewest fallback params, **without verifying the route actually matched the requested URL**. For a route like `/mixed/[lang]/[id]` where `generateStaticParams` covers `lang: 'en'` but not `id`, the covered `/mixed/en/[id]` route deferred only `[id]`. For a request to `/mixed/fr/123`, that covered route had fewer fallback params, but `en` didn't match `fr` — so applying its `[id]` set left `lang` out of the fallback set and `fr` was treated as a statically known value. The fix matches the requested URL against each prerendered route with the canonical `getRouteRegex` and picks the most-specific *matching* one. Concrete effect: in dev you'll see `params` correctly resolve to the runtime stage for routes with partial `generateStaticParams` coverage. The cold prod behavior is unaffected — this only changes which params the dev server marks as "not statically known" during the staged render. If a dynamic param was previously being read in the prerender stage (and surfacing sync-IO errors as a result), it'll now be deferred to the runtime stage and the errors should move with it.

**Suppress `prefetch={true}` warning when route opts out via `instant = false` (16.3.0-canary.67, [#95099](https://github.com/vercel/next.js/pull/95099), June 24, 2026):** In dev with Cache Components, navigating via a `<Link prefetch={true}>` to a route that hasn't opted into Partial Prefetching logs a warning, because the full prefetch pulls in dynamic data and defeats the static/dynamic split. `instant = false` is the explicit API for opting a route out of this validation, so it should silence the warning — but previously it didn't, because an `instant = false` segment set no prefetch hint at all and the warning still fired. The fix adds a new `SubtreeHasInstantFalse` `PrefetchHint`, set on any segment that exports `instant = false` and **propagated up to the root** (mirroring `SubtreeHasPartialPrefetching`). The dev warning now skips routes where this bit is present anywhere on the target subtree. This is **warning-only** and kept separate from the existing `PrefetchDisabled` bit (which feeds `StaticPrefetchDisabled` and would change actual prefetch behavior). Practical effect: a route like `/admin` that has `export const instant = false` at its layout no longer triggers the `prefetch={true}` dev warning when linked from the public marketing pages with `<Link href="/admin" prefetch={true}>`.

**`generateStaticParams` required for root parameters when `cacheComponents: true` (16.3.0-canary.67, [#95073](https://github.com/vercel/next.js/pull/95073), June 24, 2026):** Root parameters (from `next/root-params`) are available as soon as you create the routes that define them — a `generateStaticParams` is only required **with `cacheComponents: true`**, where each root parameter must have at least one value or the build fails. Without CC, `generateStaticParams` for root params is optional. Code shape:

```ts
// app/[lang]/layout.tsx — with cacheComponents, every root param needs a value
import { lang } from 'next/root-params'

export default async function RootLayout(props: LayoutProps<'/[lang]'>) {
  return (
    <html lang={await lang()}>
      <body>{props.children}</body>
    </html>
  )
}

export async function generateStaticParams() {
  return [{ lang: 'en' }, { lang: 'fr' }]
}
```

For multiple root parameters, return a value for each combination:

```ts
// app/[lang]/[locale]/layout.tsx
export async function generateStaticParams() {
  return [
    { lang: 'en', locale: 'us' },
    { lang: 'en', locale: 'uk' },
  ]
}
```

A nested segment's `generateStaticParams` can read the parent's root params (it gets the resolved values, not the raw request) — the docs now document this explicitly. The CC requirement is **fail-closed**: omit a value for a root param and the build errors out with the standard `generateStaticParams` "must return at least one value" message, naming the missing param. The companion note in the docs (TypeScript types for `next/root-params` exports) clarifies that types are generated during `next dev`, `next build`, or `next typegen` — same as `PageProps` and `LayoutProps`.

**When to enable:**
- Multi-page apps with many repeat visits (dashboards, admin panels, e-commerce back-office)
- Apps where TTI on the second visit matters more than fresh data
- Combine with `revalidateTag(tag, profile)` to control how stale the cache is allowed to get

**When to skip:**
- Apps that depend on real-time data (live trading, chat, monitoring)
- Apps that render fully-dynamic content per request (no cacheable parts)

### Layout Deduplication + Incremental Prefetching (Next.js 16)

Even without the experimental flags, Next.js 16 ships with two prefetch improvements on by default:

- **Layout deduplication** — When prefetching multiple URLs that share a layout (e.g. 50 product pages under `/shop/[id]`), the shared layout is downloaded **once**, not 50 times. Dramatically reduces network transfer.
- **Incremental prefetching** — Next.js only prefetches the parts of a route not already in cache, and cancels requests when the link leaves the viewport. The prefetch cache prioritizes links on hover or re-entering the viewport, and re-prefetches links when their data is invalidated.

### `experimental.prefetchInlining` — Now Documented (16.3.0-canary.52, June 16, 2026)

`experimental.prefetchInlining` now has a canonical config reference page ([#94650](https://github.com/vercel/next.js/pull/94650), [docs page](https://nextjs.org/docs/app/api-reference/config/next-config-js/prefetchInlining)). The reference documents the default behavior and the `false` / object forms, including:

- **`maxSize`** threshold — when an inlined bundle exceeds this size, Next.js stops inlining and falls back to per-segment prefetch (prevents over-bundling)
- **`maxBundleSize`** threshold — per-segment soft cap when inlining is on
- **Request-count vs dedup trade-off** — turning inlining on reduces the number of prefetch requests but loses layout dedup across shared routes

Previously you could only discover the option through code, PRs, and release notes — now there's an authoritative reference.

### `experimental.useExperimentalReact` — Opt Into React's Experimental Channel (16.3.0-canary.53, June 17, 2026)

New opt-in flag for selecting React's experimental build (which emits `<link rel="expect">` to hold first paint until the streamed shell is coherent — avoids flicker from partially-streamed HTML):

```ts
// next.config.ts
const nextConfig: NextConfig = {
  experimental: {
    useExperimentalReact: true,
  },
}
```

Feeds the existing `needsExperimentalReact` aggregation and the matching Turbopack `react_channel` switch. Previously you had to enable `experimental.taint` (or `transitionIndicator` / `gestureTransition`) as a side effect to get the experimental channel — confusing because the taint APIs only exist in the experimental build. Setting `useExperimentalReact: false` is a no-op when one of those still requires it, and `assignDefaults` warns on that contradiction.

**Sources:**
- [Next.js 16.2 release notes — `experimental.prefetchInlining` + `experimental.cachedNavigations`](https://nextjs.org/blog/next-16-2)
- [Next.js 16 release notes — Layout deduplication + Incremental prefetching](https://nextjs.org/blog/next-16)
- [Next.js docs — Prefetching guide](https://nextjs.org/docs/app/guides/prefetching)
- [Next.js docs — Parallel Routes file convention](https://nextjs.org/docs/app/api-reference/file-conventions/parallel-routes)
- [Next.js 16 upgrade guide — `default.tsx` requirement for parallel routes](https://nextjs.org/docs/app/guides/upgrading/version-16)
- [Next.js `prefetchInlining` config reference (canary.52, [#94650](https://github.com/vercel/next.js/pull/94650))](https://nextjs.org/docs/app/api-reference/config/next-config-js/prefetchInlining)
- [PR #94861 — Add `experimental.useExperimentalReact` (canary.53)](https://github.com/vercel/next.js/pull/94861)
- [PR #95064 — Widen `cachedNavigations` to `boolean | 'allow-runtime'` (canary.61)](https://github.com/vercel/next.js/pull/95064)
- [PR #95097 — Partial-prefetching routes opt into runtime stage of Cached Navs (canary.63)](https://github.com/vercel/next.js/pull/95097)
- [PR #95099 — Suppress `prefetch={true}` warning when route opts out via `instant = false` (canary.67)](https://github.com/vercel/next.js/pull/95099)
- [PR #95073 — `generateStaticParams` required for root params with `cacheComponents: true` (canary.67)](https://github.com/vercel/next.js/pull/95073)
- [Next.js docs — `next-root-params` reference (canary.67 + #95073)](https://nextjs.org/docs/app/api-reference/functions/next-root-params)
- [Next.js 16.3.0-canary.61 release notes](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.61)
- [Next.js 16.3.0-canary.63 release notes](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.63)
- [Next.js 16.3.0-canary.66 release notes (June 24, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.66)
- [Next.js 16.3.0-canary.67 release notes (June 24–25, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.67)
- [Next.js 16.3.0-preview.4 release notes (June 24, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0-preview.4)
- [Next.js 16.3.0-preview.5 release notes (June 25, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0-preview.5)

## `proxy.ts` — Next.js 16 Renamed from `middleware.ts`

**In Next.js 16, `middleware.ts` is deprecated in favor of `proxy.ts`.** The rename reflects that this file intercepts and proxies requests — not just adds middleware headers. Both files work during the deprecation period, but `proxy.ts` is the forward-looking name.

### Migration: `middleware.ts` → `proxy.ts`

Rename your file and update the export:

```ts
// middleware.ts (Next.js 15 and below)
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const token = request.cookies.get('token')
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  return NextResponse.next()
}
```

```ts
// proxy.ts (Next.js 16+)
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export const proxy = async (request: NextRequest) => {
  const token = request.cookies.get('token')
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  return NextResponse.next()
}
```

**Key changes:**
- File renamed: `middleware.ts` → `proxy.ts`
- Export renamed: `middleware` → `proxy`
- Function must be `async` (required in Next.js 16)

### `matcher` Config — Same Pattern, Different Place

In Next.js 15, `matcher` was exported from `middleware.ts`. In Next.js 16, move it to `next.config.ts`:

```ts
// next.config.ts
const nextConfig: NextConfig = {
  matcher: ['/dashboard/:path*', '/admin/:path*'],
}
```

Or keep it in `proxy.ts` as a named export:

```ts
// proxy.ts
export const matcher = ['/dashboard/:path*', '/admin/:path*']

export const proxy = async (request: NextRequest) => {
  // ...
}
```

### What Goes in `proxy.ts`

`proxy.ts` intercepts requests **before** they hit your app. Common use cases:

- **Authentication redirects** — check auth token, redirect to login
- **Geolocation routing** — route users to region-specific content
- **A/B testing** — assign variant, set cookie
- **Rate limiting headers** — add headers for downstream rate limiters
- **CORS** — handle cross-origin requests

```ts
// proxy.ts — auth check pattern
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

const PUBLIC_PATHS = ['/login', '/register', '/api/health']

export const proxy = async (request: NextRequest) => {
  const { pathname } = request.nextUrl
  
  // Allow public paths
  if (PUBLIC_PATHS.some(p => pathname.startsWith(p))) {
    return NextResponse.next()
  }
  
  // Check auth
  const session = request.cookies.get('session')
  if (!session) {
    const loginUrl = new URL('/login', request.url)
    loginUrl.searchParams.set('redirect', pathname)
    return NextResponse.redirect(loginUrl)
  }
  
  return NextResponse.next()
}
```

### `matcher` Pattern Mistakes to Avoid

The most common `matcher` mistake is using a regex that doesn't cover all variations:

```ts
// ❌ Wrong — doesn't match /dashboard or /dashboard/
export const matcher = ['/dashboard/:path*']

// ✅ Correct — covers all cases
export const matcher = ['/dashboard/:path*', '/dashboard']
```

**Sources:**
- [Next.js 16 upgrade guide — proxy](https://nextjs.org/docs/app/guides/upgrading/version-16)
- [Next.js proxy.ts migration](https://krishna-adhikari.com.np/blogs/next16-middleware-to-proxy)


## Pages + App Router Coexistence — Hard-Navigate Fix (16.3, June 26, 2026)

PR [#95185](https://github.com/vercel/next.js/pulls/95185) "Hard-navigate to app routes shadowed by a pages dynamic route" fixes a real, easy-to-miss client-side navigation bug in projects migrating from Pages Router to App Router.

### The bug (now fixed) — soft nav to `app/[locale]/about` from a Pages page could land on `pages/[locale]/[category]`

If you have **both** a Pages Router route **and** an App Router route whose path starts with a dynamic segment — for example `pages/[locale]/[category].tsx` and `app/[locale]/about/page.tsx` — a client-side navigation from the Pages side to `/en/about` resolved **against Pages Router routes only**. The Pages Router matched `pages/[locale]/[category]` (less specific, but Pages wins the soft-nav resolution) and rendered the wrong route. A hard reload always rendered the correct App Router page (the server pools and ranks app + pages routes by specificity), so the bug was **soft-nav only** — invisible until a user clicked a link instead of typing the URL.

The fix lives in the client router filter (`createClientRouterFilter`). Previously it only recorded the **static prefix** of a dynamic App Router route — so App routes whose first segment is dynamic contributed nothing to the filter. The Pages Router never learned to hand them off. The new code also stores a **normalized pattern** for those routes, with dynamic segments replaced by a placeholder token — `/[locale]/about` becomes `/[]/about`. After the Pages Router resolves a navigation to a dynamic route, `hasDynamicFilterCandidate` reconstructs the candidate app-route patterns from that route and the concrete path, then **forces a hard navigation** when any candidate is present in the dynamic filter. This also covers catch-all and optional catch-all Pages routes (whose final parameter absorbs a variable number of segments).

### What you actually need to do

If your project is **Pages Router only**, or **App Router only**, or has static-prefix App routes (e.g. `app/dashboard/[id]`) under a Pages project — **nothing changes**. The fix only activates when an App route starts with a dynamic segment AND a Pages dynamic route could match the resolved path.

If you're migrating Pages → App with `app/[locale]/*` routes co-resident with `pages/[locale]/*`, you get the fix automatically on the next canary cut. No config change, no opt-in flag — `createClientRouterFilter` always populates the dynamic filter now.

### Audit checklist for affected projects

Run this to find candidate routes in your project:

```bash
# List all App Router routes whose first segment is dynamic
find app -mindepth 2 -maxdepth 3 -path 'app/\[*/*' -name 'page.tsx' | head -20

# List all Pages Router dynamic routes (will shadow soft navs if they share a prefix)
find pages -name '*.tsx' | grep -E '\[' | head -20
```

If you have routes that match both lists with overlapping prefixes, your users were hitting the wrong route on soft navigation before this fix. **The fix shipped in [16.3.0-canary.69](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.69) on June 26, 2026** — upgrade to that canary (or any later one) to get it.

### Why this is a real bug, not a doc cleanup

The original PR title makes it sound like a routing-table edge case, but the underlying mechanics are a **silent misroute**: users on a Pages page who click a link to an App route would land on the wrong page, with no error, no console warning, no DevTools signal. The Pages Router's resolution is the only place where the wrong route got picked, so dev-mode tests that reload-then-assert missed it (server resolution is correct). The `pages-to-app-routing` end-to-end suite was extended to cover this — the fix is verified by a real test that does a client-side navigation from a Pages page to an App dynamic-segment route and asserts the destination.

**Source:** [PR #95185 — Hard-navigate to app routes shadowed by a pages dynamic route](https://github.com/vercel/next.js/pulls/95185)


## Middleware Rewrite Query/Params Lost in Pages API Routes — #94905 (16.3.0-canary.69, June 26, 2026)

PR [#94905](https://github.com/vercel/next.js/pulls/94905) "fix: preserve middleware rewrite query in Pages API routes" fixes a real, easy-to-miss **silent data loss** in middleware rewrites that target Pages API routes. Same bug family as #95185 above (silent corruption in shared server routing), different symptom — instead of the user landing on the wrong page, the API handler reads the wrong `req.query` and silently returns wrong data.

### The bug (now fixed) — rewrite-added query params + destination `slug` disappeared from `req.query` in Pages API handlers

If you rewrite from `/foo/bar?key=value` to `/api/proxy/bar?added=1&extra=2` in middleware, with an optional catch-all Pages API route at `pages/api/proxy/[[...slug]].ts`, the bundled Pages API handler in Next.js 15+ received **only** `{ key: "value" }` — the `added`, `extra`, and catch-all `slug` values were silently dropped. The Node `NextNodeServer.runApi` path restored `req.url` to the original browser-visible URL and forwarded only `waitUntil`, so the handler re-parsed the original URL and produced a truncated `req.query`. This regressed from Next.js 14.2.35 (which had the rewrite query and dynamic params) during the Pages API handler-interface migration.

Concrete repro (from issue [#94647](https://github.com/vercel/next.js/issues/94647), opened June 10, 2026 against Next.js 15.5.19, [reproduction repo](https://github.com/lionralfs/repro-rewrite-bug)):

```http
# Middleware
# Rewrite /foo/bar?key=value → /api/proxy/bar?added=1&extra=2
```

```ts
// pages/api/proxy/[[...slug]].ts
import type { NextApiRequest, NextApiResponse } from 'next'

export default function handler(req: NextApiRequest, res: NextApiResponse) {
  // Next.js 14.2.35: { key: 'value', added: '1', extra: '2', slug: ['bar'] }
  // Next.js 15.5.19 → 16.3.0-canary.68: { key: 'value' }   ← silent data loss
  res.json({ url: req.url, query: req.query })
}
```

```json
// GET /foo/bar?key=value  →  200 OK
// Next.js 14.2.35 (correct):
{ "url": "/foo/bar?key=value", "query": { "key": "value", "added": "1", "extra": "2", "slug": ["bar"] } }
// Next.js 15.5.19 → 16.3.0-canary.68 (regression):
{ "url": "/foo/bar?key=value", "query": { "key": "value" } }
```

### How the fix works

`NextNodeServer.runApi` now forwards the existing request metadata **together with** the resolved `query` and `match.params` through the bundled handler context. The Pages API entrypoint already installs the supplied request metadata before `routeModule.prepare()`, which then uses these routed values while retaining the original `req.url` for browser-visible compatibility. Two invariants are preserved:

1. `req.url` stays `/foo/bar?key=value` (the browser-visible URL, unchanged from the regressed behavior).
2. `req.query` contains the **original** query (`key=value`), the **rewrite-added** query (`added=1&extra=2`), AND the **catch-all `slug`** (`['bar']`).

Affects both webpack and Turbopack because the data was lost in **shared server routing after bundling** — neither bundler is at fault, the data path is in `next-server.ts`. The regression test added in #94905 asserts both invariants under `test/e2e/middleware-rewrites/` with `pages/api/proxy/[[...slug]].js`.

### What you actually need to do

**If you use middleware rewrites that target a Pages API route and read `req.query` from the handler** — you were hitting this regression. Audit your rewrites:

```bash
# Find all middleware/proxy files that rewrite to /api/* destinations
grep -rn "rewrite.*api\|rewrite.*\x27/api" proxy.ts middleware.ts 2>/dev/null

# Find all Pages API handlers that read req.query (candidates for the bug)
find pages/api -name '*.ts' -o -name '*.js' | xargs grep -l 'req.query' 2>/dev/null
```

If both lists are non-empty in your project, **upgrade to [16.3.0-canary.69](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.69)+** to get the fix. No config change, no opt-in flag — `runApi` always forwards the resolved `query` and `match.params` now.

If your middleware rewrites only target **App Router route handlers** (`app/api/*/route.ts` reading the `request.nextUrl.searchParams`) — you're unaffected. The bug is specific to the Pages API `runApi` path.

### Why this is a real bug, not a doc cleanup

Three properties make this a classic silent-corruption issue:

1. **No error, no warning, no DevTools signal.** The handler ran cleanly with truncated `req.query`. No `console.error`, no Next.js warning, no HMR overlay. The wrong data just returned to whoever called the API.
2. **Reload-based dev tests miss it.** If your test does `page.goto('/foo/bar?key=value')` and asserts the response body, the **server resolution** is correct in both versions — the bug only surfaces when a **middleware rewrite** is involved. Many tests don't include middleware in their setup.
3. **Production-only symptom.** Most dev middleware doesn't rewrite to API routes; the failure surfaces when the team's middleware starts proxying user-facing paths to internal `/api/*` endpoints (a common pattern for AB-testing, geo-redirects, A/B routing, multi-tenant path normalization).

This is the **third** silent-corruption fix the skill has documented in the past week — after #95185 (silent misroute to wrong page, June 26), and the shadcn 4.11.1 specifier-swap (silent `package.json` mutation, June 26). All three share the same shape: deterministic, silent, surfaces only in production or in specific configurations. Treat any "middleware + Pages API" rewrite path in your codebase as a candidate to audit against this fix.

## Repeated Search Params Collapsed in Client Page Segment Cache — #94863 (16.3.0-canary.71, June 29, 2026)

PR [#94863](https://github.com/vercel/next.js/pulls/94863) "fix: preserve repeated search params in client page segment cache keys" (icyJoseph, merged June 29, 2026) fixes a silent **stale-UI** bug in the client page segment cache that affects any URL whose query string has a key with multiple values (`?color=red&color=blue`, `?tag=foo&tag=bar`, `?k=A&k=B`, etc.). It is the **fourth** silent-corruption fix the skill has documented in the past week, after #95185 (silent misroute to wrong page, June 26), #94905 (silent `req.query` truncation, June 26), and the shadcn 4.11.1 specifier-swap (silent `package.json` mutation, June 26). Different bug family from those three — no Pages API or middleware involved — but the same shape: deterministic, silent, surfaces only in specific configurations.

### The bug (now fixed) — `Object.fromEntries(new URLSearchParams(search))` collapsed repeated keys in the cache key

The client-side segment-cache scheduler and PPR navigation reducer built their cache keys with `Object.fromEntries(new URLSearchParams(search))`. `Object.fromEntries` **keeps only the last value per key**:

```js
const s = '?foo=bar&foo=baz'
Object.fromEntries(new URLSearchParams(s)) // { foo: 'baz' }   ← drops 'bar'
```

Two URLs that should have been **different cache entries** therefore hashed to the **same key**:

```js
'?color=red&color=blue'  → { color: 'blue' }   // value is the last one
'?color=blue'            → { color: 'blue' }   // same key
// → cache HIT on the second navigation, no re-render — stale UI
```

The correct form was already available in the codebase as [`urlSearchParamsToParsedUrlQuery`](https://github.com/vercel/next.js/blob/canary/packages/next/src/client/route-params.ts#L235) (`packages/next/src/client/route-params.ts`), which preserves repeated params as an array:

```js
urlSearchParamsToParsedUrlQuery(new URLSearchParams(s)) // { foo: ['bar', 'baz'] }
```

The fix swaps the `Object.fromEntries` calls for `urlSearchParamsToParsedUrlQuery` in the three client-side cache sites:

- `packages/next/src/client/components/router-reducer/ppr-navigations.ts` — PPR navigation cache key
- `packages/next/src/client/components/segment-cache/scheduler.ts` — segment cache scheduler key
- `packages/next/src/client/route-params.ts` — shared helper export

A new e2e regression test lands at `test/e2e/app-dir/multi-value-search-params-stale/` (44-line `multi-value-search-params-stale.test.ts` + a `page.tsx` that renders the URL's `searchParams` so the test can assert the new render after the multi→single transition).

### Concrete symptom — multi→single search-param unset does not commit RSC, toggle "works the second time", form submissions don't update the client

Three independent user reports resolved by this single fix. All three were filed against **stable** versions of Next.js — not just canary:

**Issue [#92787](https://github.com/vercel/next.js/issues/92787) "Object.fromEntries(URLSearchParams) drops duplicate keys → stale UI on navigation"** (krivcikfilip, opened April 14, 2026 against `next@16.2.1-canary.38`):
> Click "+ red"  → `/?color=red`            (2 items) ✅
> Click "+ blue" → `/?color=red&color=blue` (4 items) ✅
> Click "✕ red"  → `/?color=blue`           still 4 items ❌ (should be 2)

**Issue [#93104](https://github.com/vercel/next.js/issues/93104) "multi→single search-param unset does not commit RSC"** (dave-wwg, opened April 21, 2026 against stable): `router.replace(url, { scroll: false })` that changes a repeated search-param from multiple values (`?k=A&k=B`) to a single value (`?k=B`) fetches a new RSC payload, the server renders the correct `searchParams`, but the client tree **never commits the update**. Empty→single and single→multi transitions commit correctly — only multi→single is broken. The reporter also notes "It also seems to work if the toggle is done a second time?", which is the cache-hit signature: the second toggle to the same key is a real re-render, the first is served from the stale cache.

**Issue [#94821](https://github.com/vercel/next.js/issues/94821) "Submitting Query Parameters via a Form does not always update the client with the newly server rendered content"** (JanBayer, opened June 15, 2026 against `next@16.3.0-canary.51`): A `next/form` `<form>` submission that adds a repeated checkbox query param and then removes one — the URL updates, the server logs the new value, the **client never receives the new render**. Same root cause: the cache key for the new URL collides with the cache key for the previous URL after `Object.fromEntries` collapses the repeated keys.

### How the fix works

`urlSearchParamsToParsedUrlQuery` was already imported in `ppr-navigations.ts` and `scheduler.ts` for a related path; the fix consolidates all three cache-key sites to use it. The returned object has arrays for repeated keys (`{ foo: ['bar', 'baz'] }`) and scalars for single keys (`{ foo: 'bar' }`) — the existing cache-key serializer already handles both shapes (it stringifies arrays), so no further plumbing is required.

Two invariants are preserved:

1. **Single-value URLs are unaffected.** `urlSearchParamsToParsedUrlQuery(new URLSearchParams('?foo=bar'))` returns `{ foo: 'bar' }` — same as before. The cache-key hash is identical for any URL that has no repeated params, so no cache misses are introduced.
2. **Server-rendered `searchParams` is unchanged.** This is a *client cache key* fix only. The server already parses search params correctly via Next.js's `parsedUrlQuery` pipeline, which has always used `urlSearchParamsToParsedUrlQuery`. The bug was strictly on the client-side cache-key computation.

### What you actually need to do

**If your app has any URL with a repeated query-string key** — and the client-side RSC tree or `<form>`-driven URL state derives from those params — you were hitting this regression. Auditable via:

```bash
# Client code that reads search params from the URL
grep -rn "useSearchParams\|searchParams.get\|searchParams\.getAll" --include="*.tsx" --include="*.ts" app/ components/ 2>/dev/null

# Server or client code that mutates the URL with repeated keys (likely via URLSearchParams or qs.stringify)
grep -rn "URLSearchParams\|searchParams\.append\|qs\.stringify\|qs\.parse" --include="*.tsx" --include="*.ts" app/ components/ lib/ 2>/dev/null

# Forms that produce repeated keys (checkbox groups, multi-select filters, etc.)
grep -rn "name=\"[a-zA-Z_]*\[\]*\"\|useFormState" --include="*.tsx" app/ 2>/dev/null
```

If any of these exist in your project, **upgrade to [16.3.0-canary.71](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.71) or later** (or the eventual 16.3.0 stable once it ships) to get the fix. No config change, no opt-in flag — the cache key always uses `urlSearchParamsToParsedUrlQuery` now.

If your app only uses single-value search params, you're unaffected.

### Why this is a real bug, not a doc cleanup

Three properties make this a classic silent-corruption issue:

1. **No error, no warning, no DevTools signal.** The cache HIT is invisible. No `console.error`, no Next.js warning, no HMR overlay. The wrong (stale) data just stays on screen until the user does a hard reload, navigates away and back via the browser back button, or — depending on the URL pattern — makes a second mutation that bypasses the cache.
2. **Reload-based tests miss it.** If your test does `page.goto('/shop?color=red&color=blue')` and asserts the response body, the **server render** is correct in all versions — the bug only surfaces on **soft navigation** (via `<Link>`, `router.replace`, or `<form>` submission) where the client cache key gets consulted. Many integration tests don't exercise soft nav with repeated keys.
3. **Masked by the browser cache.** A browser hard reload reads the URL fresh; the bug only manifests on in-app navigation. End users who reload to "fix" the UI never report it — they assume the page just needs a refresh — and the same URL keeps misbehaving for users who navigate within the app.

This is the **fourth** silent-corruption fix the skill has documented in the past week. All four share the same shape: deterministic, silent, surfaces only in production or in specific configurations. Treat any feature in your codebase that produces repeated query-string keys (filter facets, tag chips, multi-select dropdowns, checkbox groups in `<form>`) as a candidate to audit against this fix.

**Source:** [PR #94863 — fix: preserve repeated search params in client page segment cache keys](https://github.com/vercel/next.js/pulls/94863) · [Issue #92787 — Object.fromEntries(URLSearchParams) drops duplicate keys → stale UI on navigation](https://github.com/vercel/next.js/issues/92787) · [Issue #93104 — multi→single search-param unset does not commit RSC](https://github.com/vercel/next.js/issues/93104) · [Issue #94821 — Submitting Query Parameters via a Form does not always update the client](https://github.com/vercel/next.js/issues/94821)


## Client Navigation URL Bar Stuck After Middleware Redirect — #95207 (16.3.0-canary.71, June 29, 2026)

PR [#95207](https://github.com/vercel/next.js/pulls/95207) "Fix: Update the URL when a client navigation redirects to a rewritten route" (Andrew Clark / acdlite, merged June 29, 2026 at 17:00:29Z) fixes a silent **stale-URL** bug where a client-side navigation through a proxy/middleware redirect chain left the browser address bar on the *original* URL, while the page content rendered correctly. It is the **fifth** silent-corruption fix the skill has documented in the past week, after #95185 (silent misroute, June 26), #94905 (silent `req.query` truncation, June 26), the shadcn 4.11.1 specifier-swap (silent `package.json` mutation, June 26), and #94863 (silent cache-key collapse, June 29). Different bug family from the other four — only happens on soft-nav, only on the client, only with a specific three-leg path — but the same shape: deterministic, silent, no error or warning, surfaces only when all three legs are present.

### The bug (now fixed) — click `<Link href="/a">`, middleware redirects to `/`, address bar stays at `/a` even though `/` rendered

If you have a proxy/middleware that **redirects** `/a` → `/` AND **rewrites** `/` → `/a` (where `/a` is a dynamic route that reads its `params`), clicking `<Link href="/a">` (or calling `router.push('/a')`) renders the right page content (the dynamic route's `slug = 'a'` shows up correctly), but the browser address bar stays on `/a`. Hard navigation (typing the URL, browser reload, `curl`) is unaffected — those always redirect correctly to `/` because the server's resolver runs the full chain. The bug is **soft-nav only** — invisible until a user actually clicks a link.

Concrete repro from issue [#95195](https://github.com/vercel/next.js/issues/95195) (amannn, opened June 26, 2026 against [16.3.0-preview.5](https://github.com/vercel/next.js/releases/tag/v16.3.0-preview.5)) — full automated Playwright test in the [reproduction repo](https://github.com/amannn/nextjs-16-3-bug-repro-redirect):

```ts
// proxy.ts (formerly middleware.ts)
import { NextResponse, type NextRequest } from 'next/server'

export function proxy(request: NextRequest) {
  // /a → redirect to /
  if (request.nextUrl.pathname === '/a') {
    return NextResponse.redirect(new URL('/', request.url))
  }
  // / → rewrite to /a (a dynamic, force-dynamic route)
  if (request.nextUrl.pathname === '/') {
    return NextResponse.rewrite(new URL('/a', request.url))
  }
}
```

```tsx
// app/page.tsx
import Link from 'next/link'

export default function Page() {
  return <Link href="/a">Go to /a</Link>
}
```

```tsx
// app/[slug]/page.tsx
export const dynamic = 'force-dynamic'

export default async function Page({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  return <p>slug: {slug}</p>  // → "slug: a"
}
```

```bash
# Hard navigation (works in all versions):
# curl http://localhost:3000/a
# → 307 → curl http://localhost:3000/
# → rewrite → app/[slug] with slug='a' → "slug: a" ✅

# Soft navigation via <Link> (broken on 16.3.0-preview.5 + canary.66..canary.70):
# Click "Go to /a"
# → page renders: "slug: a" ✅
# → window.location.href: "/a" ❌   (should be "/")
```

All three of the following are required to trigger the bug:

1. **A client-side navigation** (hard navigation redirects correctly via the server resolver).
2. **The middleware redirect target is itself rewritten to another route** (here `/` → `/a`). A bare redirect without a rewrite does not exhibit the bug.
3. **The rewrite target is dynamically rendered and reads route params** (here `app/[slug]/page.tsx` with `force-dynamic` reading `await params`). The full-page App Router rendering pipeline must walk the dynamic param to resolve the slug.

This matches the dynamic-route shell changes in 16.3 ("Instant Navigations"): the router appears to commit the destination URL (`/a`) to the address bar optimistically before the RSC response resolves, and does not correct it after the server follows the redirect.

### How the fix works

The fix lives in the client-side navigation reducer (`packages/next/src/client/components/router-reducer/reducers/server-patch-reducer.ts` + the `ppr-navigations.ts` path that PR #94863 also touched, with new `PrefetchHint`-shape plumbing in `router-reducer-types.ts`). The previous code path committed the **predicted** URL to the address bar when the navigation started, then treated the resolved RSC payload as confirmation of the prediction. When the redirect chain resolved to a route with the same shape as the prediction (dynamic segment + same concrete path), the reducer treated the navigation as already correct and never reconciled the address bar.

The new code path treats a redirect discovered during a navigation as an **invalidator of the prediction**: the optimistic URL commit is rolled back, the route is re-resolved with the redirect target as the new key, and the address bar reflects the redirect destination before the page commits. The fix is scoped to the navigation reducer — the server-side resolver was already correct (which is why hard navigation always worked).

A new end-to-end test lands at `test/e2e/app-dir/redirect-rewrite-dynamic/`. The test app has `app/[slug]/page.tsx`, a `proxy.ts` with the same `/a → /` redirect + `/ → /a` rewrite chain, and a `<Link>` that triggers the soft nav. The test asserts both the rendered content (`slug: a`) **and** the address bar (`/`) after the click — the second assertion is what would fail on canary.66..canary.70.

### What you actually need to do

**If you have a `proxy.ts` (or `middleware.ts`) that redirects to a path that is itself rewritten to a dynamic route** — you were hitting this regression on soft-nav. Auditable via:

```bash
# Find redirects in your middleware/proxy that target paths rewritten elsewhere
grep -rn "NextResponse.redirect" proxy.ts middleware.ts 2>/dev/null

# Find rewrites that target dynamic routes (params, slug, id, etc.)
grep -rn "NextResponse.rewrite" proxy.ts middleware.ts 2>/dev/null

# Find routes that read params dynamically (force-dynamic + awaits params)
grep -rn "force-dynamic\|export const dynamic" --include="*.tsx" --include="*.ts" app/ 2>/dev/null
```

If any combination matches the three-leg pattern above (redirect + rewrite-to-different-path + rewrite target reads params), **upgrade to [16.3.0-canary.71](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.71) or later (or the eventual 16.3.0 stable)** to get the fix. No config change, no opt-in flag — the reducer always invalidates the prediction on a redirect now.

If your middleware/proxy only does one of: redirects without a rewrite, or rewrites without a follow-up redirect, or rewrites to fully-static routes — you're unaffected. The bug requires all three legs.

### Why this is a real bug, not a doc cleanup

Four properties make this a classic silent-corruption issue specific to 16.3:

1. **No error, no warning, no DevTools signal.** The page renders the correct content. The browser address bar just shows the wrong URL. No `console.error`, no Next.js warning, no HMR overlay. A user who copies the URL from the address bar and shares it gives a recipient a 307 to `/` — the share link silently does the wrong thing.
2. **Hard-navigation tests miss it.** If your Playwright test does `page.goto('/a')` and asserts the response body, the **server resolution** is correct in all versions (canary.66..canary.70 + canary.71+) — the bug only surfaces on **soft navigation** via `<Link>` or `router.push`. Many e2e suites use `page.goto` exclusively and miss this entire class of soft-nav bugs.
3. **Only present in 16.3 Preview + canary.** Next.js 15.x and 16.0–16.2 don't have the Instant Navigations optimistic-URL-commit path, so they don't exhibit the bug. The bug is **specific to the 16.3 series** where the router started committing the predicted URL before the server confirms. Stable Next.js users are unaffected until they upgrade.
4. **Masked by browser back/forward.** A user who hits the browser back button after the broken soft-nav gets the correct URL on the way back, because the browser's history stores the URL the user originally clicked — not the URL the address bar shows. End users who "fix" the issue by going back and clicking again never report it; they assume the page just needs a refresh — and the same path keeps misbehaving for users who don't know to back-out.

This is the **fifth** silent-corruption fix the skill has documented in the past week. All five share the same shape: deterministic, silent, surfaces only when specific configurations line up. Treat any combination of `(redirect → rewrite → dynamic route)` in your `proxy.ts` as a candidate to audit against this fix.

**Source:** [PR #95207 — Fix: Update the URL when a client navigation redirects to a rewritten route](https://github.com/vercel/next.js/pulls/95207) · [Issue #95195 — Middleware redirect not reflected in URL on client-side navigation when target is rewritten to a dynamic route](https://github.com/vercel/next.js/issues/95195) · [Reproduction repo (amannn/nextjs-16-3-bug-repro-redirect)](https://github.com/amannn/nextjs-16-3-bug-repro-redirect)

### Client Pages with `await params`/`await searchParams` Crash Dev Instant Validation — [#95289](https://github.com/vercel/next.js/pull/95289) (16.3.0-canary.72+, June 30, 2026)

A client component (`'use client'`) reading `await params` or `await searchParams` on a **Cache Components** dynamic route crashes dev instant validation with a misleading "This is a bug in Next.js" redbox. This is the **sixth** silent-corruption fix the skill has documented in the past week, and the most impactful for early adopters — it crashes the entire route in dev, with no work-around other than upgrading to canary.72+ or refactoring the param read to a server component.

**The crash:** a route like `app/store/[slug]/page.tsx` where the page component is a client component:

```tsx
'use client'
import { use } from 'react'

export default function Page(props: PageProps<'/store/[slug]'>) {
  const { slug } = use(props.params)  // crashes dev validation
  return (
    <>
      <h1>Store {slug}</h1>
      ...
    </>
  )
}
```

Running `next dev` against this route in any of `16.3.0-preview.5`, `16.3.0-canary.66` through `16.3.0-canary.71` produces a dev overlay:

```
## Error Message
Route "/store/[slug]": Could not validate `instant` because an error prevented the target segment from rendering.

Next.js version: 16.3.0-preview.5 (Turbopack)

  [cause]: Error [InvariantError]: Invariant: Expected to have a workUnitStore that provides validationSampleTracking. This is a bug in Next.js.
      at Page (app/store/[slug]/page.tsx:6:11)
    4 |
    5 | export default function Page(props: PageProps<"/store/[slug]">) {
  > 6 |   const { slug } = use(props.params);
      |           ^
    7 |   return (
    8 |     <>
```

The route is unusable in dev. Build production works (the code path that crashes is dev-instant-validation only), so many CI suites with `next build && next start` pass while `next dev` crashes — making it easy to miss until a developer hits it locally.

**Root cause:** `createParamsFromClient` in `packages/next/src/server/request/params.ts` (and the parallel `createSearchParamsFromClient` in `search-params.ts`) had a `'validation-client'` case that wrapped `params`/`searchParams` in a "validation-sample-tracking proxy" — a proxy that records property accesses so dev validation can confirm the page actually consumed each request-derived input. That proxy is **only valid in the build-time prerender path** where `workUnitStore.validationSamples` is set up on the store. In the dev-instant-validation path, the store has no `validationSamples`, so the proxy crashed with `Invariant: Expected to have a workUnitStore that provides validationSampleTracking`.

The original `'validation-client'` case was added so dev validation could audit client-page access to `await params`. But the proxy needs the store to be in "validation-sample-recording" mode, and dev-instant-validation puts the store in "validation-sample-passive" mode (it reads samples but doesn't record new ones). The case was gated correctly in some call paths but not in others — specifically, any client component reading `params`/`searchParams` at runtime would land on the wrong path.

**Fix ([PR #95289](https://github.com/vercel/next.js/pull/95289) by Janka Uryga, merged 2026-06-30T14:38:02Z, will be in canary.72):**

Two changes, one per file:

1. **`packages/next/src/server/request/params.ts`** — the `'validation-client'` case in `createParamsFromClient`'s switch is **removed entirely**. The switch falls through to the existing `'prerender'` case (which is the correct code path for the dev-instant-validation workUnitStore). The change is -7/+11 lines.

2. **`packages/next/src/server/request/search-params.ts`** — the `'validation-client'` case is **gated on `workUnitStore.validationSamples`**. When samples are present (build-time validation path), the proxy is created as before; when samples are absent (dev-instant-validation path), the function returns `makeUntrackedSearchParams(underlyingSearchParams)` instead — the same fallback used by the other "no samples" cases. The change is -7/+10 lines.

The asymmetry between params and searchParams is intentional: `params` only need the proxy when samples are recorded, so removing the case unconditionally is equivalent to the gated path. `searchParams` might want to behave differently in the two cases (the proxy adds a level of introspection that `makeUntrackedSearchParams` skips), so the gate preserves the option. **9 new e2e tests** at `test/e2e/app-dir/instant-validation/static/valid-client-params/[slug]/`, `valid-client-search-params/`, plus a `suspense-in-root/page.tsx` regression test.

**Who is affected (audit your codebase):**

- Any client component (`'use client'`) reading `await props.params` or `await props.searchParams` via `use()` on a dynamic route, **and** Cache Components is enabled (`cacheComponents: true` in `next.config.ts`), **and** `next dev` is being used.

Concrete search patterns to audit:

```bash
# Find client components that consume params via use()
rg -l "'use client'" app/ -g '*.tsx' | xargs rg -l 'use\(props\.(params|searchParams)\)|await props\.(params|searchParams)'

# Find any await/use of params.searchParams in client components
rg "use\(.*\.searchParams" app/ -g '*.tsx' --type tsx

# Find pages that look like 'use client' + PageProps<'/[slug]'>
rg "'use client'" app/ -g '*page.tsx'
```

**Migration to canary.72+:**

```bash
# Pin to canary.72 once cut (current canary.71 still crashes)
npm install next@canary
# Verify:
npx next --version  # should show 16.3.0-canary.72+
```

Until canary.72 is cut, **two workarounds**:

1. **Move the param read to a server component.** Server components reading `await props.params` don't crash dev validation because they don't go through `createParamsFromClient`. Extract the data the client component needs (e.g. `slug`) into props passed down from the server page, and keep the client component purely presentational:

   ```tsx
   // app/store/[slug]/page.tsx (server, no 'use client')
   export default async function Page(props: PageProps<'/store/[slug]'>) {
     const { slug } = await props.params
     return <StorePage slug={slug} />
   }

   // app/store/StorePage.tsx ('use client')
   'use client'
   export function StorePage({ slug }: { slug: string }) {
     // ... use slug freely, no use(props.params)
   }
   ```

2. **Disable Cache Components temporarily.** Set `cacheComponents: false` in `next.config.ts` while developing, then flip back on once canary.72 is out. You lose the static-shell validation benefits but the route renders.

**Why this matters even if you're not on canary yet:** any project planning to enable Cache Components and that has client components reading `await params`/`searchParams` (a common pattern — e.g. `<ProductCard>` client components that get `slug` from the URL) **must upgrade to canary.72+ before enabling Cache Components in dev**. Otherwise they'll hit the redbox on first dev start. The skill's existing Cache Components guidance didn't flag this because the bug surfaced after the guidance was written.

**Properties of this fix:**

1. **Crashes with a misleading "This is a bug in Next.js" message.** The error names `validationSampleTracking` and tells you to file a Next.js issue, but the bug is in the case-routing in `createParamsFromClient`, not in user code. A developer following the error's instructions would waste time searching for a non-existent bug in their own code.
2. **Dev-only.** `next build` is unaffected; only `next dev` instant validation hits the wrong case. CI suites with build+start pass; developers hitting `next dev` crash.
3. **Specific to client components reading params/searchParams.** Server components have their own code path (`createParamsFromServer`/`createSearchParamsFromServer`) that's not affected.
4. **Specific to Cache Components dynamic routes.** Without `cacheComponents: true`, the `'validation-client'` case in `createParamsFromClient` isn't reached, so the crash doesn't fire. The skill recommends running `cacheComponents: true` for all new 16.3 projects; combined with the common pattern of client components reading `await params`, this fix is **broadly applicable to anyone starting a new 16.3 project today**.

**Source:** [PR #95289 — fix: params/searchParams in client page crashing dev instant validation](https://github.com/vercel/next.js/pull/95289) · [Commit af066a2fe5849cf2397603381600f50ab66c4467](https://github.com/vercel/next.js/commit/af066a2fe5849cf2397603381600f50ab66c4467) · Files: `packages/next/src/server/request/params.ts` + `packages/next/src/server/request/search-params.ts` + 7 e2e tests in `test/e2e/app-dir/instant-validation/`

### "Link Data" Validation Errors for `params`/`searchParams` Outside `<Suspense>` — PR [#95151](https://github.com/vercel/next.js/pull/95151) + [#94595](https://github.com/vercel/next.js/pull/94595) (16.3.0-canary.73, SHIPPED 2026-07-01T16:33:19Z)

When `cacheComponents: true` + `partialPrefetching: true` (or per-segment `prefetch = 'partial'` / `'unstable_eager'`) is enabled, the Instant Navigation validation pipeline now treats `params` and `searchParams` accessed outside a `<Suspense>` boundary as a **distinct error category called "link data"** — separate from "runtime data" and "dynamic data". Three new blocking-route error builders land in `packages/next/src/server/app-render/blocking-route-messages.ts` and three new entries (1390–1392) in `packages/next/errors.json`. This is the new "shell can never have link data" enforcement for partial-prefetching routes.

**What's new (PR #95151, Janka Uryga, merged 2026-07-01T04:36:41Z, shipped in 16.3.0-canary.73):**

The Instant Navigation pipeline previously only distinguished Runtime vs Dynamic hole kinds. PR #95151 adds a third kind — **Link** (`DynamicHoleKind.Link = 1` in `packages/next/src/server/app-render/dynamic-rendering.ts`, with the existing `Runtime` and `Dynamic` renumbered to 2 and 3) — for holes caused by `params`/`searchParams` accessed outside `<Suspense>`. The detection logic re-runs each new segment under `ShellRuntime` (a new stage added to `instant-validation.tsx`'s `SEGMENT_STAGE_ORDER`, between `Static` and `Runtime`) and "forces them into `Runtime` for the purposes of discriminating dynamic holes": if a hole is present in `ShellRuntime` but **disappears** in `Runtime`, it's a *link data* hole (link data cannot resolve in App Shell, but it can resolve in `Runtime` since the runtime render can call `prefetch({ params })`). The three new error builders:

| Builder | Surfaces in error 1390/1391/1392 | When it fires |
|---|---|---|
| `createLinkBodyErrorInNavigation(route)` | `1390` | `params` or `searchParams` accessed in the **page body** outside `<Suspense>` |
| `createLinkMetadataError(route)` | `1391` | `params` or `searchParams` accessed in **`generateMetadata()`** |
| `createLinkViewportError(route)` | `1392` | `params` or `searchParams` accessed in **`generateViewport()`** |

The redbox (dev validation) and build error (static shell validation) point to `https://nextjs.org/docs/messages/blocking-prerender-{runtime,metadata-runtime,viewport-runtime}#{fix-card}` for the three fixes — same doc-anchor pattern as the runtime/dynamic variants.

**The body's redbox (16.3.0-canary.75+ wording, PR [#95187](https://github.com/vercel/next.js/pull/95187)):**

```
Route "/store/[slug]": Next.js encountered link data during prerendering or a navigation.

`params` or `searchParams` accessed outside of `<Suspense>` prevents the navigation from being instant, leading to a slower user experience.

Ways to fix this:
  - [stream] Provide a placeholder with `<Suspense fallback={...}>` around the data access
    https://nextjs.org/docs/messages/blocking-prerender-runtime#wrap-in-or-move-into-suspense
  - [block] Set `export const instant = false` to allow a blocking route
    https://nextjs.org/docs/messages/blocking-prerender-runtime#allow-blocking-route
```

> **canary.73–canary.74 wording** (still seen if you're not on canary.75+ yet): the `[block]` line read `Set \`export const instant = false\` to silence this warning and allow a blocking route`. PR #95187 drops "silence this warning and" because (a) the line was logged as `Error:` but the word "warning" was wrong at warning/manual-warning levels (or when the check only surfaces in dev), and (b) `instant = false` doesn't *silence* a problem that still exists — it declares blocking is acceptable for the route so validation stops treating it as one. The new wording matches the dev-overlay cards ("Allow blocking route", "Disable validation on this route") and the docs sections the lines link to. Two `[ignore]` variants in `instant-messages.ts` were updated the same way (`opt the route out of instant-navigation validation` and `opt the dropped segment out of instant-navigation validation`).


The metadata variant points to `blocking-prerender-metadata-runtime` (fixes: `[static]` use static metadata export OR `[dynamic]` mark route as dynamic via `await connection()` inside `<Suspense>`). The viewport variant points to `blocking-prerender-viewport-runtime` (fixes: `[static]` use static viewport export OR `[block]` `export const instant = false`).

**The follow-up PR #94595 (Janka Uryga, merged 2026-07-01T06:59:43Z, shipped in 16.3.0-canary.73) — extending link-data detection to `generateStaticParams`:**

Static params from `generateStaticParams` are special: they resolve at build time, but the dev/render pipeline needs to choose *when* to resolve them — and the answer depends on whether the route is rendering for an App Shell or a Static HTML Shell:

1. If we resolve in the `Static` stage, we get an accurate *static HTML shell* (used for initial HTML navigation) but an **incorrect App Shell** (link data is included when it shouldn't be, masking violations).
2. If we resolve in the `ShellRuntime` stage, we get an accurate App Shell but cannot validate a static HTML shell (which would contain static params).

PR #94595 picks **(2)** whenever client-navigating with `partialPrefetching` enabled (`requestStore.needsSessionShell === true`), and **(1)** for initial HTML renders. The static-params stage selection is now `requestStore.needsSessionShell ? RENDER_STAGES_BY_DATA_KIND.runtimeLinkData : RENDER_STAGES_BY_DATA_KIND.staticLinkData` in `createStagedRenderParamsImpl` (`packages/next/src/server/request/params.ts`). Three params utilities move from inline definitions into a new `packages/next/src/server/lib/params-utils.ts` (`isEmptyParams`, `hasFallbackRouteParams`, `allParamsAreRootParams`, plus a new `hasNonRootStaticParams`) so `app-render.tsx` can call them at the second-render-planning call site.

**Concretely, the new validation behaviour:**

| Page shape | Dev validation outcome on `next dev` after PR #95151 + #94595 |
|---|---|
| `<Page>` reads `await params` inside `<Suspense fallback={...}>` | ✅ PASSES (link data is suspended — shell renders the fallback) |
| `<Page>` reads `await params` **outside** `<Suspense>` | ❌ REDBOX — `createLinkBodyErrorInNavigation` → error 1390 |
| `generateMetadata()` reads `await params` | ❌ REDBOX — `createLinkMetadataError` → error 1391 |
| `generateViewport()` reads `await params` | ❌ REDBOX — `createLinkViewportError` → error 1392 |
| `<Page>` reads `await cookies()` (session data) outside `<Suspense>` | ✅ PASSES (session data is allowed in App Shell — see `valid-session-only` and `valid-session-with-dynamic` test fixtures) |
| `<Page>` reads `await io()` / dynamic data outside `<Suspense>` | ❌ REDBOX — existing dynamic/runtime error (different code path) |

The pattern: **session data (`cookies()` / `headers()` / `'use cache: private'`) is allowed outside Suspense in an App Shell** (since each user has their own shell). **Link data (`params` / `searchParams`) is not** (since link data is per-URL, the App Shell can't render it). **Dynamic data (`io()` / uncached fetches) is also not** (the shell can't render it; it streams in after).

**Who is affected (audit your codebase):**

- Any route with `cacheComponents: true` (default-on intent in 16.3) **and** `experimental.partialPrefetching: true` **and** the page reads `params`/`searchParams` outside `<Suspense>`. Specifically watch for:
  - Pages that read `(await params).slug` directly in JSX (move into `<Suspense>`)
  - `generateMetadata({ params })` that does `const { slug } = await params` for OG image URLs (move to static metadata, or wrap in `<Suspense>`+`await connection()`)
  - `generateViewport({ params })` reading `params` (use static viewport export)

Audit command:

```bash
# Find page.tsx files that read params/searchParams unguarded
rg -l 'await params\b|await searchParams\b' app/ -g '*page.tsx' | \
  xargs -I{} sh -c 'echo "=== {} ==="; grep -c "<Suspense" "{}" || true'

# Find generateMetadata that await's params or searchParams
rg -A2 "generateMetadata" app/ -g '*.tsx' | rg 'await (params|searchParams)'

# Same for generateViewport
rg -A2 "generateViewport" app/ -g '*.tsx' | rg 'await (params|searchParams)'
```

A non-zero count of `await params` lines in a `page.tsx` that has no `<Suspense` is the trigger pattern.

**Known limitations (as of these PRs, both noted by the author):**

1. **The PR #95151 implementation is intentionally incomplete.** The implementation uses "chunks from the dev render, which resolves static params in the `Static` stage. We use `ShellRuntime` for validating the App Shell, so as a result, static params are incorrectly included in it and don't trigger link data errors. This will be implemented in a follow-up." The follow-up is PR #94595 — but only handles the ShellRuntime-vs-Static stage order choice, not the chunk-source issue. Caching builds may show false negatives for `generateStaticParams` routes.
2. **Pre-existing `fallbackParams` bug in build validation.** Two tests marked `// TODO(app-shells): missing fallback params in build validation` — fallback params don't resolve correctly during static prerender, so a route reading `params` as fallback resolves statically when it shouldn't, and the validation error is silently missed. This is documented as a separate fix in a follow-up.
3. **The new error kind has no new test coverage for the deprecated `prefetch = 'allow-runtime'` path.** The fixtures (`invalid-runtime-params/[slug]`, `invalid-runtime-searchparams`, `invalid-runtime-with-valid-static-parent`) all use `prefetch = 'partial'` per PR #95151's purpose. The skill's existing guidance for `allow-runtime` (read params outside `<Suspense>` blocks the shell) still holds — but the validation message is now a "link data" one, not a "runtime" one.

**Migration to canary.73:**

canary.73 SHIPPED on 2026-07-01T16:33:19Z (npm `canary` dist-tag pointer moved 2026-07-01T16:35Z, 6–7 hours earlier than the 24h cadence from canary.70 predicted). The audit command above flags every page that needs a `<Suspense>` wrap on canary.73. For each match, two fixes:

1. **Stream the data access (preferred):** wrap the data access in `<Suspense fallback={<Skeleton/>}>`. The shell renders the skeleton; the data resolves and replaces it in place. Compatible with `prefetch = 'partial'` and the runtime stage.

   ```tsx
   import { Suspense } from 'react'
   export default async function Page({ params }: { params: Promise<{ slug: string }> }) {
     return (
       <main>
         <h1>Store</h1>
         <Suspense fallback={<StoreSkeleton/>}>
           <StoreCard params={params} />
         </Suspense>
       </main>
     )
   }
   async function StoreCard({ params }: { params: Promise<{ slug: string }> }) {
     const { slug } = await params   // resolves inside Suspense → OK
     return <article>{slug}</article>
   }
   ```

2. **Block the route from being instant (workaround for metadata/viewport where you can't easily Suspense the data):** set `export const instant = false` at the page or layout level. The link data fix card points at this — it tells the router "this route is a blocking route; don't try to prefetch an App Shell for it." Use only when the metadata/viewport genuinely needs the param (e.g. dynamic OG images per slug).

```bash
# canary.73 is the current canary — install from the canary dist-tag
npm install next@canary
npx next --version  # should show 16.3.0-canary.73
```

If you're on `16.3.0-canary.72` or earlier, `next dev` will not produce the new redbox — instead `params`/`searchParams` outside `<Suspense>` will silently fall into the `Runtime` hole kind (existing "Runtime data" or "Dynamic data" redbox from #95289-era behaviour). The same fixes (wrap in `<Suspense>` or set `instant = false`) apply. **Upgrade to canary.73+ for the new redbox** — and use `npx @next/codemod upgrade canary --yes` (auto-enabled on non-TTY, per PR #95312; also bumps `eslint` to match `eslint-config-next`'s peer dep, per PR #95314) when running from an agent or CI.

**Why this matters even if you're on the stable 16.2 branch:** `partialPrefetching` is the 16.3 default-on-intent flag, and `cacheComponents` is the foundation. Once 16.3 ships as stable, the "link data" error becomes a **first-class redbox** you'll see on day one of any Cache Components adoption. The skill pre-loads the audit command + the three error numbers so it can diagnose the redbox in 30 seconds instead of 30 minutes.

**Source:** [PR #95151 — [PP] Validate Shell prefetches (except gSP)](https://github.com/vercel/next.js/pull/95151) · [Commit c131314bcf0cf21c5d5624b2cf8c6dbf07375abd](https://github.com/vercel/next.js/commit/c131314bcf0cf21c5d5624b2cf8c6dbf07375abd) · [PR #94595 — [PP] Instant validation - error for unguarded static params](https://github.com/vercel/next.js/pull/94595) · [Commit 576d3a3397bf2818369636dd26d8ca4a9303bd98](https://github.com/vercel/next.js/commit/576d3a3397bf2818369636dd26d8ca4a9303bd98) · Files: `packages/next/src/server/app-render/{blocking-route-messages,dynamic-rendering}.ts` + `packages/next/src/server/app-render/instant-validation/instant-validation.tsx` + `packages/next/src/server/request/params.ts` + new `packages/next/src/server/lib/params-utils.ts` + `packages/next/errors.json` + 21 test fixtures in `test/e2e/app-dir/instant-validation/app/shells/` and `test/e2e/app-dir/instant-validation/app/invalid-runtime-{params,searchparams}/`.

**Cross-reference — `use cache` constants that gate the prerender vs runtime stages (canary.74 rename):** the `ShellRuntime` stage above works alongside the prerender vs runtime gates defined in `packages/next/src/server/use-cache/constants.ts`. In canary.74 those gates were renamed from `DYNAMIC_EXPIRE` / `DYNAMIC_STALE` → `MIN_PRERENDERABLE_EXPIRE` (300s) / `MIN_PREFETCHABLE_STALE` (30s) for clarity at comparison sites. See patterns.md → `MIN_PRERENDERABLE_EXPIRE` / `MIN_PREFETCHABLE_STALE` Constant Rename for the threshold semantics, the constant table, and the [PR #95361](https://github.com/vercel/next.js/pull/95361) source.



## Server Action + Navigation Interaction Bugs — #95391 + #95392 (16.3.0-canary.76, July 2, 2026)

Two silent-corruption fixes shipped in the same canary. Both involve Server Actions interacting with client-side navigation in ways that drop user state or trap the user in the wrong URL.

### Navigation Reverted After a Server Action Settles — #95391 (gaearon/dan, merged 2026-07-02T22:52:24Z)

When a navigation started while a Server Action was still in flight, the router would discard the action and the navigation would take its place at the head of the action queue. The discarded action still ran to completion. Since PR #82674 (shipped in 15.5.1), its completion also called `runRemainingActions`, which advances the queue — but the discarded action is no longer at the head of the queue, the navigation is. This started the next queued action while the navigation was still running, computed against router state that didn't include the navigation yet. Those actions send their requests with the previous URL, and applying their responses reverts the navigation.

**Symptom:** Click a link or call `router.push()`, the destination page renders, then the router snaps back to the previous URL. Reported in #86151 with a minimal repro. There was a related stuck-forever bug (#84299) that the same fix closes.

**The promise rejection path had the same problem even before #82674:** a discarded action that rejected would also advance the queue.

**Fix:** Only the settlement of the action at the head of the queue may advance the queue. A discarded action's settlement (resolve or reject) now only records whether the action revalidated, and the deferred refresh is flushed once the queue goes idle. This keeps the behavior #82674 was after: no refresh for discarded actions that didn't revalidate, and a refresh for those that did — including when they settle mid-navigation, a case that could previously lose the refresh entirely.

**Properties:**

- **Regression from 15.5.1** (PR #82674). Verified broken in 16.2; was working in 16.1.
- **Alternative to PR #95188** which was opened from a fork; #95391 was the same change on an upstream branch so the deploy-test jobs could run with credentials.
- **Fixes two bugs:** #86151 (navigation reverted) + #84299 (navigation stuck forever).
- **New e2e tests** cover both the broken case AND the case that the regressing PR #82674 was originally trying to fix. The tests are "a bit wonky and do some timing assertions" per the PR description — agent-loop reruns may need retries.

**Who needs to audit:** any app that does `router.push()` / `<Link>` navigation immediately after (or during) a Server Action call. The "immediately after" case is the most common — a form submit that ends with a `router.push('/dashboard')` after the action succeeds.

**Source:** [PR #95391 — `Fix navigation getting reverted when a Server Action is in flight`](https://github.com/vercel/next.js/pull/95391) · Commit `db4e6231e1` (2026-07-02T22:52:24Z) · gaearon · Fixes [#86151](https://github.com/vercel/next.js/issues/86151) (navigation reverted) + [#84299](https://github.com/vercel/next.js/issues/84299) (navigation stuck forever).

### `router.push('/x'); router.refresh()` Drops the Push — #95392 (canary.76, merged 2026-07-02T19:06:41Z)

**Symptom:** A `router.push('/target')` followed by `router.refresh()` ended up at `/target`, but the previous page's history entry was replaced instead of a new one being pushed. Pressing Back skipped the previous page (yanked past it) because that history entry no longer existed.

```tsx
// ❌ Dropped: previous page replaced, Back skips it
router.push('/target')   // this entry was getting lost
router.refresh()
```

**Why:** React commits a state whose `pushRef.pendingPush` is true (`HistoryUpdater`). Refresh-type actions (`refresh`, `hmr-refresh`, `server-patch`) create their state with a fresh `pushRef` where `pendingPush` is false. React can skip committing the superseded navigation state entirely, so `pendingPush: true` is never observed and the push never happens — `HistoryUpdater` replaces the previous page's history entry with the new URL instead of pushing a new one. Going back after that does nothing: the entry the browser returns to has no state, and the popstate handler ignores entries without state.

**Regression from #88046 in 16.1.** The old router carried the previous state's `pendingPush` forward unless a reducer set it explicitly (the `handleMutable` reducer), and the refresh reducer never set it. That guard was lost in #88046, which deleted `handleMutable` and moved the final state assembly into `completeSoftNavigation`, where `pushRef` is created fresh with `pendingPush: navigateType === 'push'`. PR #95392 restores the old behavior there. There's no double push: if the navigation's state did commit, the push already happened and consumed the flag (`HistoryUpdater` mutates the `pushRef`), and its same-URL check turns a leftover flag into a replace anyway.

In dev, an `hmr-refresh` dispatched while compiling the destination route on demand can supersede the navigation the same way. That's what's behind the recent `browser back to a revalidated page` flakes in `navigation.test.ts` (traced in #95385).

**Who needs to audit:** any app that pairs `router.push` with `router.refresh()` in close succession. Common pattern: a `<form action={serverAction}>` whose `useFormStatus().pending` triggers a manual `router.push('/list')` followed by `router.refresh()` to clear stale cached data. Pre-canary.76 the user could not press Back to return to the form.

**Source:** [PR #95392 — `Fix history push getting treated like replace when followed by refresh`](https://github.com/vercel/next.js/pull/95392) · Merged 2026-07-02T19:06:41Z · Regression from PR #88046.

## Metadata Title Dropped on Soft Navigation with Cache Components — #95315 (16.3.0-canary.75, July 2, 2026)

With `cacheComponents: true` + `partialPrefetching`, navigating to an already-prefetched route with a dynamic `generateMetadata` left `document.title` permanently empty until a hard reload.

**The bug:** the prefetch cached the head as complete using the server's `isHeadPartial` flag, which is unreliable under Cache Components. Derive the head's partiality from `isResponsePartial` instead, as segment data already does, so a dynamic head is fetched and applied on navigation while a fully static head stays complete.

Fixing the flag exposed a latent issue in the prefetch scheduler: during a speculative prefetch, the head was unconditionally runtime-prefetched, which previously went unnoticed because the head was mismarked as complete. The head is now runtime-prefetched only when a segment in the new part of the tree is a candidate for runtime prefetching — it rides along with that request rather than spawning a standalone one. If nothing in the new part of the tree is a runtime prefetch candidate, the head is fetched during the navigation instead. This makes the scheduler's metadata-only request path dead code, so it's removed.

**Symptom:**
- Soft navigation (clicking a `<Link>`) lands on the new route but `<title>` is empty.
- Hard reload restores the title.
- Only triggers with `cacheComponents: true` AND `partialPrefetching: true` (or per-segment `prefetch = 'partial'` / `'unstable_eager'`).
- Only triggers when the destination route has a dynamic `generateMetadata`.

**Audit:**
```bash
rg -l "cacheComponents:\s*true" next.config.{ts,js,mjs} app/
rg -l "partialPrefetching" next.config.{ts,js,mjs} app/
rg -l "export async function generateMetadata" app/ -g '*.{ts,tsx}'
# Intersection of all three → affected
```

**Who needs to upgrade:** any project on `cacheComponents: true` with dynamic metadata. The fix is in canary.75+ and the bug is silent (no error, no warning, just an empty title) so it can go unnoticed for a long time in production.

**Fixes:** [#95268](https://github.com/vercel/next.js/issues/95268).

**Source:** [PR #95315 — `Fix metadata title dropped on soft navigation with Cache Components`](https://github.com/vercel/next.js/pull/95315) · Commit `27e225f468` (2026-07-02T12:16:45Z) · acdlite/Andrew Clark.

## Prefetch Priority — Top-of-Document Links First — #95393 (16.3.0-canary.75, July 2, 2026)

Most viewport prefetches are scheduled via a shared `IntersectionObserver`. When several links entered the viewport at once (e.g. on initial load), the observer reported them in document order and we scheduled them in that order. Because the prefetch scheduler gives the highest priority to the *most recently scheduled* task, this meant the link **lowest** in the document ended up with the highest priority — the opposite of a sensible default.

**Fix:** iterate each observer batch in reverse, so the link nearest the **top** of the document is scheduled last and therefore prioritized. The topmost link isn't guaranteed to be the most important, but as a default heuristic it's more reasonable than favoring whichever link happens to be lowest in the document.

**Test impact:** updates the two hover tests in `prefetch-scheduling` whose expected ordering was coupled to the old bottom-first default (the set of route trees prefetched within the concurrency limit flips from pages 7,6,5,4 to 1,2,3,4). The behaviors they assert are unchanged, and a new test covers the top-of-document prioritization.

**Why this matters:** the topmost link in the viewport is almost always the most likely next click — that's why viewport-driven prefetch exists. Prioritizing the lowest link meant prefetch budget was wasted on elements the user was unlikely to click next. After #95393, prefetch bandwidth tracks expected click probability more closely.

**Supersedes #94902** which was opened from a fork. The "Test new and changed tests when deployed" jobs cannot pass on fork PRs — `vercel link` fails with "No existing credentials found" because `secrets.VERCEL_ADAPTER_TEST_TOKEN` is not available to `pull_request` runs from forks. PR #95393 is the same change (rebased onto latest canary) on an upstream branch so the deploy jobs get credentials.

**Source:** [PR #95393 — `Prefetch links nearest the top of the document first`](https://github.com/vercel/next.js/pull/95393) · Commit `c099530eb7` (2026-07-02T12:17:05Z) · acdlite/Andrew Clark.

## Common Mistakes — Routing Edition

- **Missing `default.tsx` in parallel route slots** — Next.js 16 will fail the build. Add `default.tsx` to every `@slot` that can be unmatched.
- **Using `middleware.ts` instead of `proxy.ts`** — works but deprecated in Next.js 16; the export must be renamed to `proxy` and made `async`.
- **`matcher` regex missing the root path** — `['/dashboard/:path*']` does not match `/dashboard` itself. Always include both.
- **Putting auth checks only in `proxy.ts`** — proxy can be bypassed; always re-validate `await auth()` in the page/route handler too.
- **Catching all search params with `useSearchParams()` without `<Suspense>`** — forces the entire route segment to be dynamic. Wrap the client subtree in `<Suspense>` to keep the static shell cacheable.
- **Inlining heavy layouts** with `experimental.prefetchInlining` — defeats layout dedup. Only enable when most routes have small segments.
- **Enabling `experimental.cachedNavigations`** for real-time data — you'll show stale data. Skip for trading/chat/monitoring.
- **Catching `<Error>` from a Server Component in a Client `error.tsx`** — works, but the error boundary must be a Client Component. The boundary also re-renders the static shell, so use it sparingly.
