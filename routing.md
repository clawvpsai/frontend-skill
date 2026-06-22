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
  },
}

export default nextConfig
```

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

## Common Mistakes — Routing Edition

- **Missing `default.tsx` in parallel route slots** — Next.js 16 will fail the build. Add `default.tsx` to every `@slot` that can be unmatched.
- **Using `middleware.ts` instead of `proxy.ts`** — works but deprecated in Next.js 16; the export must be renamed to `proxy` and made `async`.
- **`matcher` regex missing the root path** — `['/dashboard/:path*']` does not match `/dashboard` itself. Always include both.
- **Putting auth checks only in `proxy.ts`** — proxy can be bypassed; always re-validate `await auth()` in the page/route handler too.
- **Catching all search params with `useSearchParams()` without `<Suspense>`** — forces the entire route segment to be dynamic. Wrap the client subtree in `<Suspense>` to keep the static shell cacheable.
- **Inlining heavy layouts** with `experimental.prefetchInlining` — defeats layout dedup. Only enable when most routes have small segments.
- **Enabling `experimental.cachedNavigations`** for real-time data — you'll show stale data. Skip for trading/chat/monitoring.
- **Catching `<Error>` from a Server Component in a Client `error.tsx`** — works, but the error boundary must be a Client Component. The boundary also re-renders the static shell, so use it sparingly.
