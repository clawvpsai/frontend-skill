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

// After mutation — revalidateTag with 'max' profile = stale-while-revalidate
// (single-arg form is deprecated; use the profile parameter)
// Use for non-critical data, high-traffic pages
revalidateTag('posts', 'max')

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
revalidateTag('posts', 'max')
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

**Legacy PPR codepaths — REMOVED in 16.3.0-canary.61, then REVERTED in 16.3.0-canary.66 ([#94955](https://github.com/vercel/next.js/pull/94955) + [#95113](https://github.com/vercel/next.js/pull/95113), June 22–24, 2026):** Canary.61 removed the `experimental.ppr` route-segment config and the `isAppPPREnabled` codepath — the `__NEXT_PPR` define-env was deleted alongside — and added error code #1375 (`"Route segment config \"dynamic\" is not compatible with \`nextConfig.cacheComponents\`. Please remove it."`) for `dynamic` + `cacheComponents` collisions. The 16.0 codemod `remove-experimental-ppr` ([#90948](https://github.com/vercel/next.js/pull/90948)) was also restored. **Canary.66 reverts that whole change** — the revert message is *"Builder isn't ready. Need to figure out what config it is actually reading from."* So in canary.66+ the legacy PPR config is back, `isAppPPREnabled` is back, `__NEXT_PPR` is back, error #1375 is gone, and the codemod is reverted. The skill's "use `cacheComponents: true` for all PPR behavior" guidance still holds — that's the forward-looking API — but the flip-flop matters in two ways: (a) if you pin to a specific canary, check whether the version you target has the removal in place (canary.61–canary.65 = removed; canary.66+ = re-enabled), and (b) error #1375 only fires on canary.61–canary.65. Expect the removal to be re-attempted in a later canary once the config-reader issue is sorted.

**Metadata image routes now statically prerender under Cache Components (16.3, [#94957](https://github.com/vercel/next.js/pull/94957), canary.61, June 22, 2026):** Before this fix, `opengraph-image`, `icon`, and other metadata image routes that returned an `ImageResponse` were always classified as `ƒ` (Dynamic) when `cacheComponents` was on, even if the route was fully deterministic. The fix serializes the `ImageResponse` arguments with React Flight's `prerenderToNodeStream` inside the prerender work-unit store, runs the user's component tree once in the correct scope, and resolves `React.lazy` references so async Server Components (including `use cache` ones) can be passed as the `ImageResponse` element. Result: a route that resolves entirely from static data or `use cache` finishes serializing within the prerender's cache-sourced input budget and renders to a real image at build time — the route comes out `○` (Static) in the build's route table instead of `ƒ`. Routes that still hit uncached `fetch` or `cookies()` fall back to `ƒ`, same as before.

**Local fonts in statically-prerendered `ImageResponse` metadata routes — Buffer-corruption follow-up (16.3.0-canary.67, [#95121](https://github.com/vercel/next.js/pull/95121), June 24, 2026):** The #94957 change serialized the `ImageResponse` constructor arguments through React Flight to build a cache key + run the element's async Server Components once — but a custom font passed as a Node `Buffer` in the options was corrupted by that round-trip. Flight applies the `toJSON` method that `Buffer` carries, so the font reached satori as a `{ type: 'Buffer', data: [...] }` plain object rather than binary, and satori's font parser threw `TypeError: First argument to DataView constructor must be an ArrayBuffer`. The crash happened even without a user `'use cache'` directive because the serialization runs unconditionally during the prerender. The fix serializes **only the element** through Flight (the only part that needs async resolution + dynamic-input detection) and pairs the resolved element with the **original in-memory options** when handing the tree to satori — so the font `Buffer` never gets serialized. The cache key is now a SHA-256 hash of the serialized element combined with a content hash of the options (binary font data hashed by its bytes, everything else by a sorted, type-tagged walk). This also keeps font bytes out of the cache key and removes the "Uint8Array objects are not supported" warning that the dev React runtime emitted during `next build --debug-prerender`. The custom-fonts docs example now loads the font **once at module scope** — reading it inside the component would be uncached I/O that Cache Components treats as dynamic, and wrapping the read in `'use cache'` would route the `Buffer` through use-cache's own Flight serialization and reintroduce the same corruption upstream of `ImageResponse`, where this fix cannot reach it.

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

### ISR + Prefetch + App Shells (16.3.0-canary.63)

PR [#94534](https://github.com/vercel/next.js/pull/94534) (June 24, 2026) closes a long-standing cold-cache gap: previously, a static segment prefetch that hit a route with an unresolved ISR entry couldn't be served a fallback shell — so the prefetch cache was left cold until the entry regenerated, and the user saw the spinner on the first navigation after revalidation. Canary.63 lets the prefetch request be served the ISR **fallback shell immediately**, warming the cache right away.

Because the shell is only a partial response, the client retries the prefetch a bounded number of times so it eventually upgrades to the full concrete response once the server finishes regenerating in the background. **Only shells that can actually be upgraded are retried** — a route with no `generateStaticParams` never upgrades, so its shell isn't flagged and the client doesn't waste retries on it.

The new serving behavior is gated behind the existing `experimental.appShells` flag, so existing behavior is unchanged when it's off. Pair this with `experimental.cachedNavigations` to get the full "instant repeat visits + warm prefetch on cold cache" combo.

### Dev Insights — Cleaner Fix Cards (16.3.0-canary.63)

PR [#94926](https://github.com/vercel/next.js/pull/94926) cleans up two misleading Insight fix cards in the dev overlay:

- **Drop the `generateStaticParams` card** from runtime + client-hook Insight sets. It was being shown on every error involving `cookies()` / `headers()` / `params` / `searchParams`, but `generateStaticParams` only applies to `params` — for the other three it was noise, and even for `params` it nudged devs to make the route static instead of fixing the immediate error.
- **Filter the `"use cache"` card** when the cause is `connection()`. Caching `connection()` is contradictory, so suggesting it was actively wrong.

Both manifests on initial load and in-navigation (the fix-card sets are shared). The card set is now `01-cookies-body`, `03-params-body`, `90-client-use-params`, `41-subnav-cookies`, `42-subnav-fetch` (drop) plus `05-connection-body`, `08-connection-body-dynamic`, `31-connection-in-metadata`, `33-connection-in-viewport` (filtered).

See: [Next.js 16.3 canary release notes](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.26) • [PR #94534 — Serve ISR fallback shells to prefetch requests (canary.63)](https://github.com/vercel/next.js/pull/94534) • [PR #94926 — Drop irrelevant fix cards from instant errors (canary.63)](https://github.com/vercel/next.js/pull/94926) • [Next.js 16.3.0-canary.63 release notes](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.63) • [PR #95080 — `TURBOPACK_DEBUG_CSS_CHUNKING` env var (canary.64)](https://github.com/vercel/next.js/pull/95080) • [PR #95100 — `cacheMaxMemorySize: 0` no longer forces dynamic cache life in dev (canary.65)](https://github.com/vercel/next.js/pull/95100) • [PR #95116 — `getHeaders` no longer mutates `req.headers` (canary.65)](https://github.com/vercel/next.js/pull/95116)

### Turbopack Single-Entry Chunks + Chunk Merging (16.3.0-canary.64)

PRs [#94727](https://github.com/vercel/next.js/pull/94727) and [#95102](https://github.com/vercel/next.js/pull/95102) (June 24, 2026) land two complementary improvements to the chunk graph:

- **Single-entry chunks** — the chunker can now emit chunks that contain exactly one entry module, in addition to multi-entry chunks. This is the building block for finer-grained code splitting when a route has heavy peer-imports (e.g. a charting library imported only by a single route) that don't naturally cluster with the rest of the page.
- **Merge chunks when `overlap == 1`** — when two adjacent chunks would emit a `1`-overlap (one shared dependency in the same segment), they're now merged into a single chunk. This trims the request count and avoids a class of awkward `ChunkLoadError`s where a shared helper was reachable only through a chunk that the next-page navigation had already evicted.

For most apps the practical effect is neutral or slightly positive — a small reduction in chunk count and a small improvement in navigation reliability. If you watch the build's chunk graph, you'll see fewer "tiny" chunks in the report; the new rules are safe defaults and don't need opt-in.

### Debugging the Graph-Based CSS Chunker (16.3.0-canary.64)

PR [#95080](https://github.com/vercel/next.js/pull/95080) (June 24, 2026) ships a new debug side-channel for the `experimental.cssChunking: "graph"` pipeline (the new graph-based CSS chunker in Turbopack-core):

```bash
TURBOPACK_DEBUG_CSS_CHUNKING=1 pnpm next build
```

On every invocation of `compute_style_groups_graph` the chucker writes a timestamped, pretty-printed JSON snapshot — `turbopack-css-chunking-debug-<unix_ms>-<seq>.json` — into the current working directory. Each dump contains:

- `chunk_groups: string[][]` — the module idents per chunk group, in the order the algorithm saw them
- `global_order: string[]` — the flat global order from `linearize`
- `global_order_chunks: string[][]` — the same modules grouped by the merged segments produced by `split_into_chunks`
- `modules: [{ ident, size, style_type }]` — per-module size and style-type metadata

The env var is read exactly once per process (cached in a `static LazyLock<bool>`); truthy values are anything other than unset, empty, `0`, or `false` (case-insensitive). **Dump failures are swallowed** — the toggle must never fail a build. Use it when filing a Vercel issue about bad CSS chunking decisions, attach the JSON, and a maintainer can replay the chunking graph locally without instrumenting Rust on your machine.

### `cacheMaxMemorySize: 0` Dev Hot-Reload Fix (16.3.0-canary.65)

PR [#95100](https://github.com/vercel/next.js/pull/95100) (June 24, 2026) fixes a real-world dev hot-reload regression that bit fully-cached routes. Previously the `'use cache'` wrapper forced every entry in a `cacheMaxMemorySize: 0` cache to a **dynamic cache life** (`revalidate: 0`, a five-minute `expire`) so that warm reads would serve stale + regenerate in the background. That forcing leaked into the *stored* entry. A fully-cached route with no `<Suspense>` above the `'use cache'` would, on a warm reload, read the entry, see `revalidate: 0`, and the dev validation prerender would throw a false-positive: *"`'use cache'` with zero `revalidate` is nested inside another `'use cache'` that has no explicit `cacheLife`"*. The route was then reported as a blocking route even though nothing was actually nested. The **cold load was unaffected** because the entry didn't exist yet — the bug only fired on hot reload.

With canary.65 the size-0 path keeps the entry's *resolved* cache life (the explicit `cacheLife()` value when the user set one, otherwise the default 15-min `revalidate` / infinite `expire`). That default sits well above the dynamic thresholds, so a fully-cached route can prerender again. An explicitly-short `cacheLife()` still makes the route legitimately dynamic without a spurious throw, because the explicit-revalidate and explicit-expire flags are then set. Private caches are unchanged and remain forced-dynamic.

To keep reloads showing a fresh value, the background revalidation in the cache-hit path also fires for size-0 built-in entries regardless of the stored `revalidate`, but **only in the dynamic dev request render** — not in the dev validation prerender — so the prerender never regenerates in parallel with itself.

### `getHeaders` Request-Header Mutation Fix (16.3.0-canary.65)

PR [#95116](https://github.com/vercel/next.js/pull/95116) (June 24, 2026) fixes a regression in dev instant navigation that left the React dev overlay stuck in *"Rendering..."* after a server-action redirect.

**Root cause:** `getHeaders` in the request store strips internal flight and dev request-id headers before sealing the object exposed to userland `headers()`. It built the sealed object with `HeadersAdapter.from(req.headers)` — which **wraps `IncomingHttpHeaders` without copying** — and then called `.delete()` on it. Those deletes mutated the shared `req.headers` as a side effect. For flight headers the absence is a handled state, so it was harmless. But for the dev request-id headers (`x-nextjs-request-id` and `x-nextjs-html-request-id`) added in PR #94703, the action's `headers()` access deleted them from `req.headers`. The dev instant-navigation render of the redirect target reuses the same request, then read a *missing* request id and fabricated a *mismatched* HTML request id. The React dev debug channel was registered under that wrong id, so the client overlay stayed in "Rendering..." and never applied the navigation.

**Fix:** copy the headers before stripping so only the sealed userland view is affected, leaving the shared request headers intact. An e2e test now reproduces the regression.

**Practical impact:** if a teammate reports the dev overlay hanging after a server action redirect, the first check is `next dev` version — anything before canary.65 with `cacheComponents` enabled will still show this. Either upgrade to 16.3.0-canary.65+ or temporarily toggle `cacheComponents` off in `next.config.ts` to unblock.

### Navigation Inspector — Blocking-Route Empty-Shell Error (16.3.0-canary.67)

PR [#95139](https://github.com/vercel/next.js/pull/95139) (June 24, 2026) closes a footgun in the dev-only Navigation Inspector (Instant Navigation Testing API).

**The bug:** when the Navigation Inspector is active, the server renders only a route's static shell. A **blocking** route — one that reads a dynamic value such as `await cookies()` at the root with no Suspense boundary above it, or that opts into blocking via `export const instant = false` — produces an *empty* static shell. Canary.66 and earlier served that empty shell directly, so the browser showed a blank document **with no DevTools**. With no DevTools, there was no way to release the inspector lock from the UI — every reload stayed blank, the user was stuck, and a misread of the situation could be mistaken for a dev server crash.

**The fix:** detect the empty static shell at serve time and **throw to a normal error page** instead, and **clear the instant-navigation cookie** so the next reload renders the route normally (the inspector is no longer active, so the route renders its full tree). The team plans to render this error message inline in the Navigation Inspector panel rather than as a full-page error in a future canary.

**Practical impact:** if you see a full-page error during a Navigation Inspector session — particularly the "blank document, no DevTools, can't reload out" failure mode — upgrade to 16.3.0-canary.67+ first. If you can't upgrade, the manual escape hatch is `await fetch('/__nextjs_original-stack-frame?file=…')` to clear the inspector cookie via DevTools, or a hard restart of `next dev` (the cookie is process-scoped in dev).
### `instant()` Renders Shell Only Unless `prefetch` Prop Is Set (16.3.0-preview.5)

PR [#95150](https://github.com/vercel/next.js/pull/95150) (June 25, 2026) tightens the `instant()` behavior so the shell and the prefetched payload stay cleanly separated. Before this change, an `instant()` call could render a full payload into the shell response — wasting bytes on data that was only useful on actual navigation. After this change, **`instant()` only renders the route's static shell by default**. To render the prefetched payload (the data the route would produce on navigation), pass the `prefetch` prop explicitly:

```ts
// Renders only the shell — fast, small payload
// Use this when serving the response for the "instant nav" cookie path
instant()  // now equivalent to: instant({ prefetch: false })

// Renders the shell + the prefetched payload
// Use this when warming the prefetch cache for a future navigation
instant({ prefetch: true })
```

**Practical impact:**

- If you have custom integration code that calls `instant()` from a Server Action or a route handler and expects the full payload, you'll now get just the shell. Pass `{ prefetch: true }` to restore the old behavior.
- The `<Link prefetch>` path is unaffected — it always passes `prefetch: true` internally, so prefetched payloads continue to flow as before.
- Combined with the dev-parity change below, this means dev now mirrors production: prefetch requests serve the same shell-only response they would in prod, which makes Dev Insights and the Navigation Inspector behave consistently.

### Production Prefetch Shells Now Replicated in Dev (16.3.0-preview.5)

PR [#95067](https://github.com/vercel/next.js/pull/95067) (June 25, 2026) closes a long-standing dev/prod discrepancy: previously, `next dev` rendered a fully-hydrated tree for prefetch requests, while `next start` / production served the static shell only. That difference made it impossible to catch shell-only correctness issues (missing Suspense boundaries, blocking data reads, layout-vs-page mismatches) until the app shipped. After this change, dev serves the **same shell-only response** that production would, so prefetch issues surface in the dev overlay rather than in customer logs.

**Practical impact:**

- If a route previously "worked fine" in dev but threw "Empty static shell" or similar errors in prod, you can now reproduce the failure locally without deploying.
- The Dev Insights fix-card set is unchanged, but the new shell-only path means more routes will trigger the `connection()` / `cookies()` / `headers()` fix cards in dev than before — that's the intended behavior, since those routes would have failed in prod anyway.
- Combine with `experimental.cachedNavigations` + `experimental.prefetchInlining` for the full production-equivalent instant-navigation pipeline.

### `next-dev-loop` Papercut Fixes (16.3.0-preview.5)

PR [#95153](https://github.com/vercel/next.js/pull/95153) (June 25, 2026) tightens the **`next-dev-loop` skill** (the agent-skill that drives Next.js dev through iterative edits). Several papercut-class fixes — clearer error messages on a failed dev server start, better cache invalidation between agent iterations, and a more deterministic restart path. The agent skill itself is the right way to drive a Next.js project through a coding agent loop (see `setup.md` for the recommended agent-driven dev workflow).

**Sources for preview.5:**
- [Next.js 16.3.0-preview.5 release notes (June 25, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0-preview.5)
- [PR #95150 — `instant()` only renders shell unless `prefetch` prop is set](https://github.com/vercel/next.js/pull/95150)
- [PR #95067 — Replicate production prefetch shells in dev](https://github.com/vercel/next.js/pull/95067)
- [PR #95153 — `next-dev-loop` papercut fixes](https://github.com/vercel/next.js/pull/95153)
- [PR #95147 — docs: expand `io()` reference](https://github.com/vercel/next.js/pull/95147)

### Dev Cold-Cache Badge Now Behind `experimental.coldCacheBadge` (16.3.0-canary.68)

PR [#95169](https://github.com/vercel/next.js/pull/95169) (June 25, 2026) gates the **persistent "Cold cache" badge** that the dev overlay shows after a navigation filled an empty cache. The badge was added in canary.57 / [#94611](https://github.com/vercel/next.js/pull/94611) and was on by default in every dev session. After this change it's **off by default** — too loud and visually disruptive in its current form, the team wants to iterate on the UI/UX before re-enabling it for everyone.

**What's now gated:**
- The **persistent** "Cold cache" badge in the dev-overlay corner (left behind after a load settles) — **now off by default**.
- The **transient** "Rendering (cold cache)" pill shown *during* a navigation is **unchanged** — it clears itself once the navigation commits, so it stays valuable without being disruptive.
- The pre-existing "Cache disabled" (bypass) badge is also unchanged.
- The DevTools menu's cold-cache entry is also unchanged.

**How to enable / disable:**

```ts
// next.config.ts — opt in to the persistent badge for local dev
const nextConfig: NextConfig = {
  experimental: {
    coldCacheBadge: true,   // default: false (canary.68+)
  },
}

// Or via the env var the badge plumbing actually reads:
process.env.__NEXT_EXPERIMENTAL_COLD_CACHE_BADGE = '1'
```

The flag is plumbed to the dev overlay via the define plugin (`computeIntent` resolves to a no-badge path when the flag is off). **Storybook forces the flag on** through its `env` hook so the badge stories remain the surface for iterating on the design — every test suite that asserts on the badge also opts in, so none of them regress while it's disabled by default.

**Practical impact for agents:**
- If you were relying on the persistent badge as a "your build is doing cold-cache work" signal — and you see it disappear after upgrading to canary.68+ — set `experimental.coldCacheBadge: true` in `next.config.ts` to bring it back.
- If you were *not* relying on it and found it noisy, just upgrade. The transient pill is unchanged.
- The canary.57 guidance ("the cold cache indicator is scoped to shell cache misses only") is still accurate for the transient pill; only the persistent badge is gated.

### `partialPrefetching` Shell Prefetch Simulation — Reveal after ShellRuntime (16.3.0-canary.68)

PR [#95149](https://github.com/vercel/next.js/pull/95149) (June 25, 2026) refines the **Shell Prefetch** simulation in dev when `partialPrefetching` is on. Before this change, the dev overlay's cold-cache indicator and the `revealAfterStage` machinery used the `RenderStage.Runtime` boundary as the "what counts as cold-cache work" cutoff, which over-counted for shell-prefetch routes (the user only cares about cache misses up to the App Shell, not the runtime tree that runs after the shell commits).

**What changed:**
- When `partialPrefetching` is on, the shell we care about is the **App Shell**, represented by the `ShellRuntime` stage. The dev overlay now counts cold-cache work up to **and including `ShellRuntime`**, and releases the client-side promise at that boundary.
- This produces a much more accurate dev-only cold-cache signal for the common case (a shell prefetch) — warm-shell navigations no longer get flagged just because the runtime tree behind the shell needed to fetch data.
- **Limitation:** if you're navigating via `<Link prefetch={true}>` or using `partialPrefetching: 'unstable_eager'`, you might want the `Runtime` stage instead — that's a follow-up PR.
- Internal refactor: `revealAfterStage` and `holdStreamUntilRevealed` were merged into a single `DevNavigationKind` object that represents either an initial load or a client navigation. This removes the invalid `holdStreamUntilRevealed = true` + `revealAfterStage = RenderStage.Runtime/ShellRuntime` combination and brings the logic closer to where the stream-blocking tricks actually happen. Drive-by refactor of `streamStagedRenderInDev` deduped some repetitive code, and all `revealAfter.resolve()` calls were moved into separate tasks (previously inconsistent).

**Practical impact:**
- If you use `partialPrefetching: true` and watch the dev overlay for cold-cache signals, expect fewer false-positive "cold cache" warnings after upgrading to canary.68+. Warm-shell navigations on shell-prefetch routes will no longer show the badge.
- This is a **dev-only** change — production prefetch behavior is unchanged. Canary.65+ (#95067) already aligned dev with prod on shell-only prefetch responses; this change further aligns the dev-overlay *signal* with prod behavior.
- If you're hacking on the dev-overlay itself, the new `DevNavigationKind` type is the right entry point — the old `revealAfterStage`/`holdStreamUntilRevealed` combo is gone.

**Sources for canary.68:**
- [Next.js 16.3.0-canary.68 release notes (June 25, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.68)
- [PR #95169 — Gate the dev Cold cache badge behind an experimental flag](https://github.com/vercel/next.js/pull/95169)
- [PR #95149 — `[PP]` Reveal after ShellRuntime when simulating a Shell Prefetch in dev](https://github.com/vercel/next.js/pull/95149)
- [PR #94611 — Dev Cold cache indicator (canary.57, now gated by `experimental.coldCacheBadge`)](https://github.com/vercel/next.js/pull/94611)
- [PR #94911 — Scope the Cold cache indicator to shell cache misses (canary.57)](https://github.com/vercel/next.js/pull/94911)

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

### Turbopack Configuration Reference (Next.js 16.2+)

All Turbopack options live under the top-level `turbopack` key in `next.config.ts` (not `experimental.turbopack` anymore):

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  turbopack: {
    // File system cache — persists compiler artifacts to .next/ for faster restarts
    // Dev: default true. Build: default false (turn on for large apps).
    fileSystemCacheForDev: true,
    fileSystemCacheForBuild: true,

    // Tree shaking modes
    treeShaking: false,              // module-fragments mode (advanced); default reexports-only
    removeUnusedImports: false,      // default true in build
    removeUnusedExports: false,      // default true in build; requires removeUnusedImports
    inferModuleSideEffects: true,    // local analysis for better tree shaking

    // Other build options
    minify: true,                    // default true in build
    sourceMaps: true,                // default true in dev, productionBrowserSourceMaps in build
    inputSourceMaps: true,           // extract source maps from input files
    scopeHoisting: true,             // default true in build, always off in dev
  },
}
```

| Option | Dev default | Build default | What it does |
|---|---|---|---|
| `fileSystemCacheForDev` | `true` | n/a | Cache compiler artifacts to `.next/` — restart is much faster |
| `fileSystemCacheForBuild` | n/a | `false` | Same for production builds — turn on for large apps |
| `minify` | `false` | `true` | Minify output |
| `sourceMaps` | `true` | `productionBrowserSourceMaps` | Emit source maps |
| `treeShaking` | `false` | `false` | Advanced module-fragments mode (more aggressive than reexports-only) |
| `removeUnusedImports` | `false` | `true` | Strip unused imports |
| `removeUnusedExports` | `false` | `true` | Strip unused exports |
| `inferModuleSideEffects` | `true` | `true` | Local analysis to detect side-effect-free modules |
| `scopeHoisting` | `false` | `true` | Combine module scopes for smaller output |

**When to override:**
- Slow restart in dev? `fileSystemCacheForDev: true` (already default)
- Slow build on large app? `fileSystemCacheForBuild: true` (turn on)
- Aggressive bundle size? `treeShaking: true` + `removeUnusedImports: true`
- Stale source maps? `sourceMaps: true` explicitly

**Sources:**
- [Turbopack API reference (Next.js docs)](https://nextjs.org/docs/app/api-reference/turbopack)
- [Turbopack 16.2 improvements](https://nextjs.org/blog/next-16-2-turbopack)

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
