# Performance — Streaming, Suspense, Images, Caching

## Streaming with Suspense

Streaming allows the server to send HTML progressively — the shell renders immediately, content streams in as it resolves.

### Basic Streaming

```tsx
// app/blog/page.tsx
import { Suspense } from 'react'
import { BlogHeader } from '@/components/blog-header'
import { PostList, PostListSkeleton } from '@/components/post-list'

export default function BlogPage() {
  return (
    <main>
      {/* This renders immediately */}
      <BlogHeader />
      
      {/* This streams in when the data is ready */}
      <Suspense fallback={<PostListSkeleton />}>
        <PostList />
      </Suspense>
    </main>
  )
}
```

### Multiple Streaming Sections

```tsx
export default function DashboardPage() {
  return (
    <DashboardLayout>
      <Suspense fallback={<StatsSkeleton />}>
        <Stats />
      </Suspense>
      
      <Suspense fallback={<RecentActivitySkeleton />}>
        <RecentActivity />
      </Suspense>
      
      <Suspense fallback={<NotificationsSkeleton />}>
        <Notifications />
      </Suspense>
    </DashboardLayout>
  )
}
```

### Skeleton Loading Components

```tsx
// components/post-list-skeleton.tsx
export function PostListSkeleton() {
  return (
    <div className="space-y-4">
      {[...Array(5)].map((_, i) => (
        <div key={i} className="border rounded-lg p-4 space-y-2">
          <Skeleton className="h-4 w-1/3" />
          <Skeleton className="h-6 w-2/3" />
          <Skeleton className="h-4 w-full" />
          <Skeleton className="h-4 w-4/5" />
        </div>
      ))}
    </div>
  )
}
```

## Image Optimization

### `<Image>` vs `<img>`

Always use Next.js `<Image>`:

```tsx
import Image from 'next/image'

// ❌ Wrong — no optimization
<img src="/photo.jpg" alt="photo" />

// ✅ Right — automatic optimization, lazy loading, WebP conversion
<Image 
  src="/photo.jpg" 
  alt="photo" 
  width={800} 
  height={600}
  className="object-cover"
/>
```

### Remote Images

```tsx
// next.config.ts — allow remote patterns
const nextConfig = {
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'images.unsplash.com' },
      { protocol: 'https', hostname: '*.amazonaws.com' },
      { protocol: 'https', hostname: 'picsum.photos' },
    ],
  },
}
```

### Responsive Images

```tsx
// srcset for different viewport sizes
<Image
  src={post.thumbnail}
  alt={post.title}
  fill
  sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
  className="object-cover"
/>
```

### Priority Loading for LCP

```tsx
<Image 
  src={heroImage} 
  alt="Hero"
  width={1200} 
  height={600}
  priority  // Preload this image, boosts LCP score
/>
```

### Image Caching — `minimumCacheTTL` Default Changed (Next.js 16)

**Next.js 16 changed the default `images.minimumCacheTTL`** from 1 minute to **4 hours**. This significantly reduces origin requests for sites with many images.

```ts
// next.config.ts
const nextConfig: NextConfig = {
  images: {
    // Next.js 16 default is now 4 hours (14400 seconds)
    // If you need the old 1-minute behavior:
    minimumCacheTTL: 60,
  },
}
```

**When to change it:** If your images change frequently (e.g., user-generated content with new uploads), set `minimumCacheTTL: 60` to revert to 1-minute caching. For most sites, the 4-hour default is ideal.

## Caching Strategies

### `use cache` + `cacheTag` (Next.js 16)

The `use cache` directive is the primary way to cache data in Next.js 16:

```tsx
// lib/data.ts
import { cacheTag } from 'next/cache'

export async function getPosts() {
  'use cache'
  cacheTag('posts')
  return db.post.findMany()
}
```

On-demand revalidation:

```tsx
import { revalidateTag, updateTag } from 'next/cache'

// After mutation — revalidateTag schedules background refresh (fast, serves stale briefly)
// Use for non-critical data, high-traffic pages
revalidateTag('posts')

// updateTag immediately expires the cache (strong consistency, slightly slower)
// Use for critical data: inventory, auth, personalization
updateTag('posts')
```

### Route Handler Caching (fetch-based)

```ts
// app/api/posts/route.ts
export async function GET() {
  const posts = await fetch('https://api.example.com/posts', {
    next: { revalidate: 3600 },  // Revalidate every hour
  })
  return NextResponse.json(posts)
}
```

### Cache Tags

```ts
// Tag-based invalidation — use with 'use cache' functions
const posts = await getPosts() // getPosts uses cacheTag('posts')

// Invalidate anywhere
revalidateTag('posts')
```

**Note:** The `next: { tags: [...] }` pattern on fetch still works, but `use cache` + `cacheTag` is the Next.js 16 preferred approach for server-side data functions.

### `use cache` vs `cacheComponents` — Two Levels of Caching

Next.js 16 has two distinct caching mechanisms that serve different purposes:

| Concern | Mechanism | Level |
|---|---|---|
| **Data fetching** | `use cache` + `cacheTag` | Data layer — caches function return values |
| **Render output** | `cacheComponents: true` (PPR) | Render layer — caches rendered HTML segments |

**`use cache`** is for **data** — wrap any data-fetching function with `'use cache'` and the compiler memoizes its result:

```tsx
// Data — cached at function level
export async function getUserPosts(userId: string) {
  'use cache'
  cacheTag(`user-posts-${userId}`)
  return db.post.findMany({ where: { authorId: userId } })
}
```

**`cacheComponents: true`** is for **rendering** — it enables PPR, which caches the rendered output of Suspense boundaries:

```ts
// next.config.ts — enables PPR (render-level caching via Suspense boundaries)
const nextConfig: NextConfig = {
  cacheComponents: true,
}
```

**They work together:** `use cache` caches the data, `cacheComponents` (PPR) caches the rendered component shell around that data. A common pattern:

```tsx
// With PPR + use cache:
// 1. The static shell (<Header>, <Footer>) renders once and is cached
// 2. Dynamic Suspense boundaries stream in
// 3. Inside those boundaries, data functions with 'use cache' are cached separately

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <Header />  {/* Static — cached by PPR */}
        <Suspense fallback={<FeedSkeleton />}>
          <Feed />  {/* Dynamic — streams; Feed uses 'use cache' internally */}
        </Suspense>
        <Footer />  {/* Static — cached by PPR */}
      </body>
    </html>
  )
}
```

**When to use each:**
- Use `use cache` on **any data function** — DB queries, API calls, file reads
- Use `cacheComponents: true` (PPR) for **pages with mixed static/dynamic content** — marketing pages, dashboards with personalized widgets

**Rule:** Always use `use cache` for data. Enable `cacheComponents` (PPR) at the route level when you want the framework to cache rendered Suspense shells too.

## Bundle Optimization

### Dynamic Imports (Code Splitting)

```tsx
import dynamic from 'next/dynamic'

const HeavyChart = dynamic(() => import('@/components/heavy-chart'), {
  loading: () => <Skeleton className="h-80 w-full" />,
  ssr: true,  // or false for client-only
})

// Usage
export default function DashboardPage() {
  return (
    <div>
      <HeavyChart data={chartData} />
    </div>
  )
}
```

### Client Component Boundary Optimization

```tsx
// ❌ Bad — whole page is a client component
'use client'
export default function Page() {
  const [count, setCount] = useState(0)
  // Heavy data processing
  const processed = data.map(d => expensiveTransform(d))
  return <div>{processed.map(d => <Item key={d.id} {...d} />)}</div>
}

// ✅ Good — only the interactive part is client
// app/page.tsx — server component
export default function Page() {
  const processed = data.map(d => expensiveTransform(d))
  return (
    <div>
      {processed.map(d => <Item key={d.id} {...d} />)}
      <Counter /> {/* Only this small island is client */}
    </div>
  )
}

// components/counter.tsx
'use client'
export function Counter() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

## Partial Prerendering (PPR) — Stable in Next.js 16

Partial Prerendering (PPR) combines static rendering and dynamic rendering at the route segment level. Static parts are pre-rendered at build time (or revalidated), while dynamic parts stream in on request.

**Status:** PPR is **stable** in Next.js 16. Enable it with `cacheComponents: true`:

```ts
// next.config.ts
const nextConfig: NextConfig = {
  cacheComponents: true,  // ✅ Stable in Next.js 16
}
```

### How PPR Works

Wrap dynamic segments in `Suspense` boundaries — Next.js automatically identifies which parts are static (rendered at build time) and which are dynamic (streamed on request):

```tsx
// app/layout.tsx — static shell + dynamic content
import { Suspense } from 'react'

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head />
      <body>
        <Header />  {/* Static — rendered at build time */}
        <Suspense fallback={<Spinner />}>
          <UserSpecificContent />  {/* Dynamic — streams on request */}
        </Suspense>
        <Footer />  {/* Static — rendered at build time */}
      </body>
    </html>
  )
}
```

**The static shell loads instantly from CDN** while dynamic content streams in — giving you sub-100ms TTFB for static pages with personalized content.

### When to Use PPR

| Scenario | Use PPR? |
|---|---|
| Mostly static page with small dynamic parts | ✅ Yes — great fit |
| Fully dynamic page (personalized per user) | ❌ No — use `force-dynamic` instead |
| Mixed static/dynamic content | ✅ Yes — best fit |

**PPR vs traditional ISR:**
- **ISR**: Entire page is static or revalidated together
- **PPR**: Granular control — individual Suspense boundaries can have different caching strategies

**Sources:**
- [Next.js PPR Platform Guide](https://nextjs.org/docs/app/guides/ppr-platform-guide)
- [Next.js 16 release notes](https://nextjs.org/blog/next-16)

## App Shells (Next.js 16.3 canary)

**Status:** App Shells is an emerging feature in Next.js 16.3 canary — not yet stable. This section is for forward-looking awareness.

App Shells extend PPR by enabling **prefetched, cached shell renders** that are served from edge CDN before any dynamic content streams in. Unlike standard PPR where the static shell is rendered per-request, App Shells are rendered once and cached at the CDN edge — giving you sub-50ms TTFB for fully-personalized pages.

**How it differs from PPR:**

| Aspect | PPR | App Shells |
|---|---|---|
| Shell rendering | Per-request (after cache) | Prefetched and cached at edge |
| TTFB for static parts | ~100ms | ~10-50ms |
| Cache location | Origin cache | Edge CDN |
| Maturity | Stable (Next.js 16.2+) | Canary (16.3+) |

**When it becomes stable**, App Shells will be the recommended approach for pages with personalized content (user name, avatar, notification count) because the shell itself is cacheable while only the dynamic Suspense boundaries stream in per-user.

**Enable in next.config.ts (when stable):**
```ts
const nextConfig: NextConfig = {
  cacheComponents: true,       // Enables PPR
  appShells: true,             // Enables App Shells (Next.js 16.3+)
}
```

**Note:** App Shells require all static content to be wrapped in Suspense boundaries — if a component doesn't have a Suspense boundary, it's considered part of the shell and can't stream independently.

See: [Next.js 16.3 canary release notes](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.26)
## Turbopack — Fast Development Bundler

Next.js 16 ships Turbopack (Rust-based bundler) as the default development bundler:

```bash
# Development uses Turbopack automatically in Next.js 16
npm run dev

# Force Webpack if you hit Turbopack bugs
next dev --webpack
```

**Benefits:**
- ~10x faster cold start vs Webpack
- 10x faster HMR (hot module replacement) for large apps
- Same behavior as Webpack for most Next.js features

**Production builds** use Turbopack by default in Next.js 16 (`next build` uses Turbopack automatically).

## Font Optimization

```tsx
// app/layout.tsx
import { Inter } from 'next/font/google'

const inter = Inter({ 
  subsets: ['latin'],
  display: 'swap',  // Prevents FOIT (flash of invisible text)
  preload: true,
  variable: '--font-inter',
})

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html className={inter.variable}>
      <body>{children}</body>
    </html>
  )
}
```

**Next.js 15 font improvements:**
- `next/font` now handles font subsetting automatically
- No layout shift from font swaps with `display: 'swap'`
- Self-hosted fonts with zero external requests

## Prefetching

Next.js 16 introduced `<Link prefetch>` prop options for fine-grained control over when and how prefetching occurs. This became critical after the **Next.js 16 Prefetch Traffic Issue** — teams upgrading from Next.js 15 to 16 reported doubled origin request volume and significantly higher tail latency because Next.js 16 began aggressively prefetching all visible links by default.

### Prefetch Prop Options

```tsx
import Link from 'next/link'

// prefetch="full" (default in Next.js 16) — prefetches full page on hover
// ⚠️ This caused the traffic issue: every visible link triggers a full prefetch
<Link href="/dashboard" prefetch="full">
  Dashboard
</Link>

// prefetch="none" — no prefetching at all
// Use for: navigation links that rarely get clicked (footer links, secondary nav)
<Link href="/privacy" prefetch="none">
  Privacy Policy
</Link>

// prefetch="viewport" — prefetches when the link enters the viewport
// Useful for: below-the-fold links that become visible on scroll
<Link href="/blog" prefetch="viewport">
  Read our blog
</Link>
```

**When to use each:**

| Option | Trigger | Bandwidth Cost | Use When |
|---|---|---|---|
| `prefetch="full"` (default) | Hover | High | Primary navigation — users almost always click |
| `prefetch="viewport"` | Scroll into view | Medium | Below-the-fold links that users scroll to |
| `prefetch="none"` | None | None | Rarely-clicked links, external links, footer |

### Diagnosing Prefetch Traffic Issues

If your Next.js 16 app has higher origin request volume after upgrading, check these signs:

```bash
# Symptoms to look for:
# - Origin requests doubled after Next.js 15 → 16 upgrade
# - High tail latency (p99) despite low average latency
# - Prefetch requests hitting your origin for rarely-visited pages

# Check in your analytics:
# 1. Compare request graphs before/after upgrade
# 2. Look for requests to pages that aren't in the top navigation
# 3. Prefetch requests look like GET /[route] with a Next.js prefetch header
```

### Fixing Prefetch Traffic

**Step 1: Audit your links** — identify which `<Link>` components don't need prefetch:

```tsx
// ❌ Prefetching footer links, legal pages — wastes bandwidth
<footer>
  <Link href="/privacy">Privacy</Link>
  <Link href="/terms">Terms</Link>
  <Link href="/sitemap">Sitemap</Link>
</footer>

// ✅ Disable prefetch for low-priority links
<footer>
  <Link href="/privacy" prefetch="none">Privacy</Link>
  <Link href="/terms" prefetch="none">Terms</Link>
  <Link href="/sitemap" prefetch="none">Sitemap</Link>
</footer>
```

**Step 2: Use `viewport` prefetch for below-the-fold content** — prefetches only when scrolled into view:

```tsx
// ✅ "Related articles" at the bottom of a blog post — prefetch when visible
// Only triggers prefetch when the user scrolls down to that section
<Link href="/blog/related-article" prefetch="viewport">
  Related Article
</Link>
```

**Step 3: Global default via next.config.ts** (Next.js 16.3+):

```ts
// next.config.ts — set a less aggressive default
const nextConfig: NextConfig = {
  // Change the default prefetch behavior for all <Link> components
  // "full" = default (prefetch on hover), "none" = opt-in per link
  // Note: this is a Next.js 16.3+ option
}
```

**Step 4: Measure** — after changes, compare your origin request graph to before the upgrade. The goal is returning to pre-upgrade request volumes.

### `router.prefetch()` — Programmatic Prefetch

```tsx
'use client'
import { useRouter } from 'next/navigation'

export function PrefetchOnHover({ href }: { href: string }) {
  const router = useRouter()

  return (
    <button
      onMouseEnter={() => router.prefetch(href)}
      onClick={() => router.push(href)}
    >
      Go somewhere
    </button>
  )
}
```

### Prefetch Cache Deduplication (Next.js 16.3 canary)

Next.js 16.3 canary introduced **deduplication improvements** for the `use cache` directive — multiple components fetching the same cached data now share a single origin request instead of each triggering their own. This directly addresses the doubled-request issue when `use cache` is used extensively:

```tsx
// Next.js 16.3+ — dedup means this data fetch is shared across components
// Instead of 3 components each triggering a fetch, Next.js coalesces into 1
export async function getUserData() {
  'use cache'
  cacheTag('user-data')
  return db.user.findFirst()
}
```

**Note:** Prefetch cache dedup is available in **Next.js 16.3 canary** and later. For stable Next.js 16.2.x, use explicit prefetch controls on `<Link>` to manage traffic.

**Sources:**
- [Next.js Prefetching Guide](https://nextjs.org/docs/app/guides/prefetching)
- [Next.js 16.3 canary prefetch controls](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.26)
- [Next.js 16 prefetch traffic issue](https://blog.path-finder.jp/troubleshooting/next-js-16-prefetch-traffic-guide-2026-en/)


## React 19.2 — PPR Resume APIs + DevTools Performance Tracks

React 19.2 (October 2025) introduced two significant additions for streaming SSR and debugging.

### PPR Resume APIs — Streaming HTML Recovery

React 19.2 adds `resume` and `prerender` APIs that enable **Partial Pre-Rendering with Web Streams** — you can resume a prerender that was postponed, streaming the completed HTML while the rest renders. This is the foundation Next.js 16's PPR feature is built on.

These APIs are for framework authors and advanced use cases — Next.js handles PPR automatically. But understanding them helps when debugging streaming behavior.

#### `prerender` + `resume` (Web Streams — Browsers)

```tsx
import { prerender } from 'react-dom/server'
import { resume } from 'react-dom/server'

// 1. Start a prerender (generates HTML shell + postponed state)
const { prelude, postponed } = await prerender(<App />, {
  signal,  // AbortSignal to cancel the prerender
})

// 2. Stream the shell to the client immediately
// prelude is a ReadableStream — pipe to the response
response.body = prelude

// 3. Later, when the suspended content is ready, resume:
const { resumePrelude } = await resume(<App />, { postponed, signal })
// resumePrelude is another ReadableStream — continue piping
```

**What this enables:** The browser receives the static shell instantly, while postponed (suspended) parts stream in as they resolve — no blocking on the full render.

#### `prerender` + `resumeAndPrerender` (Node Streams)

```tsx
import { prerender } from 'react-dom/server'
import { resumeAndPrerender } from 'react-dom/server'

// Node.js pattern — pipe through a writable stream
const { prelude } = await prerender(<App />, { signal })
const { resumePrelude } = await resumeAndPrerender(<App />, { postponed, signal })

// Pipe both to the response
passThrough.write(prelude)
for await (const chunk of resumePrelude) {
  passThrough.write(chunk)
}
```

#### `resumeToPipeableStream` + `resumeAndPrerenderToNodeStream`

For Node.js HTTP responses using `PipeableStream`:

```tsx
import { resumeToPipeableStream } from 'react-dom/server'

// Same as resumeAndPrerender but for pipeable streams
const { resumePrelude } = await resumeAndPrerenderToNodeStream(<App />, {
  postponed,
  signal,
  onError(error) { console.error(error) },
})

resumePrelude.pipe(res)
```

**When to use these directly:**
- You are building a framework like Next.js
- You need fine-grained control over streaming behavior beyond what Next.js provides
- Most production apps should use Next.js's built-in PPR (`cacheComponents: true`) instead

### React Performance Tracks (DevTools)

React 19.2 adds **React Performance Tracks** — React-specific timing data visible directly in the browser's Performance panel timeline. This lets you see React's render scheduling, Suspense boundaries, and effect timing alongside native browser events.

**What appears in the timeline:**
- React render phases (work in progress, commit)
- Suspense boundary shows (when content is loading vs revealed)
- Effect execution timing
- State updates and their propagation

**To use:**
1. Open Chrome DevTools → Performance tab
2. Start a recording and interact with your React app
3. Look for "React" tracks alongside "Frames", "Network", etc.

**This replaces the need for separate React DevTools profiling** for basic render debugging — you can now see React's impact on your app's performance directly in the browser's native Performance panel.

**Sources:**
- [React 19.2 release notes](https://react.dev/blog/2025/10/01/react-19-2)
- [React prerender/resume APIs](https://react.dev/reference/react-dom/server)

## Web Vitals

| Metric | Target | What to Fix |
|---|---|---|
| LCP (Largest Contentful Paint) | < 2.5s | Optimize hero images, use priority, preload fonts |
| FID / INP | < 100ms | Move heavy JS to Server Components, defer non-critical JS |
| CLS (Cumulative Layout Shift) | < 0.1 | Always set width/height on images, reserve space for ads |
| TTFB | < 800ms | Use edge caching (Vercel, Cloudflare), reduce server processing |

## Common Mistakes

- **Missing `priority` on above-the-fold images** — hurts LCP
- **No skeleton fallback** — streaming without Suspense fallback = layout shift
- **Client component bloat** — keeping too much in `'use client'` bundles all that JS
- **Forgetting cache invalidation** — after mutations with `use cache`, use `revalidateTag` (background) or `updateTag` (immediate)
- **Large `data` arrays passed as props** — paginate or virtualize long lists
- **`useEffect` for initial data** — use server components or React Query instead
- **Relying on implicit caching** — in Next.js 16, everything is dynamic by default; use `use cache` explicitly
- **All `<Link>` using default `prefetch="full"`** — causes doubled origin requests in Next.js 16; disable prefetch for footer links and low-priority routes
