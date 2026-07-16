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

### Durable `use cache` Code Hash — `experimental.durableUseCacheEntries` (16.3.0-canary.72+, [PR #94234](https://github.com/vercel/next.js/pull/94234) by mischnic, merged 2026-06-30T12:50:55Z)

Each `use cache` function now gets an implementation hash that lets a remote cache (Vercel Data Cache, S3, Redis, etc.) survive across deployments without serving stale entries. Without a stable hash, a deploy that changes a transitive import of a `use cache` function would either (a) keep returning the old cached value or (b) invalidate everything on every deploy — both bad.

**What the hash covers:**

- The module containing the `use cache` function, plus all transitive imports of that module (including externals via NFT)
- For bundled code: the generated code (so it includes any AST transforms, inlined env vars, etc.)
- For NFT'd files: the file content on disk
- Equivalent to hashing the generated JS chunks, **scoped to only what the given actions module uses**

**Configuration (Turbopack-only, dev disabled):**

```ts
// next.config.ts
const nextConfig = {
  experimental: {
    // Default false. When true, Turbopack writes a per-`use cache`-function
    // `codeHash` field into .next/server/server-reference-manifest.json so a
    // remote cache key can include it. Survives across deployments.
    durableUseCacheEntries: true,
  },
}

export default nextConfig
```

**Surface — the new `codeHash` field in `.next/server/server-reference-manifest.json`:**

```jsonc
{
  "node": {
    "806f4954cfbb75404a19d6d405065ed9059cc0cab2": {
      "workers": {
        "app/rsc/page": {
          "moduleId": "[project]/bench/app-router-server/.next-internal/server/app/rsc/page/actions.js { ACTIONS_MODULE0 => "[project]/bench/app-router-server/app/rsc/logic.js [app-rsc] (ecmascript)" } [app-rsc] (server actions loader, ecmascript)",
          "async": false,
          "exportedName": "$$RSC_SERVER_CACHE_0",
          "filename": "bench/app-router-server/app/rsc/logic.js",
          "codeHash": "413e394d9597b35df12828a6a7fd6363" // <-- new
        }
      },
      "filename": "bench/app-router-server/app/rsc/logic.js",
      "exportedName": "$$RSC_SERVER_CACHE_0"
    }
  },
  "edge": {},
  "encryptionKey": "jOF2TbpNLjp2oOhl3VPBwGE7luodnei9clJPq/gaYQo="
}
```

**Implementation details from the PR description:**

- **Gated behind the experimental flag** — opt-in via `experimental.durableUseCacheEntries`. No behavior change when off.
- **Only computed for `use cache` functions** — not all server actions. The PR deliberately scopes the hash to the cache path.
- **Disabled in dev** — the hash is computed at build time only.
- **Multiple `use cache` functions in a single file share the same hash** — since the hash covers the module + imports, two `use cache` exports from `lib/data.ts` will share one hash. That's intentional: any change to `lib/data.ts` invalidates both, which is the conservative thing to do.
- **Known limitation:** `final_read_hint` is a problem — some chunk items get codegen'd twice, so the AST is recomputed. The author flagged this in the PR description; the workaround is in the manifest writer and does not affect correctness.
- **Turbopack-only.** Webpack doesn't implement this; no-op if you build with webpack.

**Audit commands:**

```bash
# See which use cache functions have a code hash
rg '"codeHash"' .next/server/server-reference-manifest.json

# Or pretty-print
node -e 'console.log(JSON.stringify(require("./.next/server/server-reference-manifest.json"), null, 2))' | rg -A 2 'codeHash'

# Verify the flag is on
rg 'durableUseCacheEntries' next.config.ts
```

**Files touched** (21 files, +281/-12, mostly test scaffolding):
- `crates/next-api/src/server_actions.rs` (+256/-12) — Turbopack `use_cache` loader writes `codeHash` to manifest entries
- `crates/next-api/src/app.rs` (+2/-4) — wires through
- `crates/next-core/src/next_config.rs` (+12/-0) — config key plumbing
- `crates/next-core/src/next_manifests/mod.rs` (+2/-0) — `ServerReferenceManifest` struct gains `codeHash`
- `packages/next/src/server/config-schema.ts` (+1/-0) — `durableUseCacheEntries: z.boolean().optional()`
- `packages/next/src/server/config-shared.ts` (+6/-0) — `ExperimentalConfig.durableUseCacheEntries?: boolean` with JSDoc "Enables durable "use cache" remote cache entries across deployments. Only implemented for Turbopack."
- 14 new test files under `test/production/app-dir/use-cache-code-hash/`

**Sources:**
- [PR #94234 — `Turbopack: compute code hash per "use cache" function`](https://github.com/vercel/next.js/pull/94234)
- [PR #94234 commit `84457f4e6c`](https://github.com/vercel/next.js/commit/84457f4e6c73bd44cf65736d75075e962f7ef30c)
- [`config-shared.ts` at v16.3.0-canary.72](https://raw.githubusercontent.com/vercel/next.js/v16.3.0-canary.72/packages/next/src/server/config-shared.ts) — `durableUseCacheEntries` declaration
- [`server-reference-manifest` schema in `next_manifests/mod.rs`](https://github.com/vercel/next.js/blob/v16.3.0-canary.72/crates/next-core/src/next_manifests/mod.rs)

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

### `experimental.cssChunking: 'graph'` Config Schema Breaking Change — `moduleFactorCost` → `weightDistribution` (16.3.0-canary.71, June 29, 2026)

PR [#95088](https://github.com/vercel/next.js/pull/95088) (Tobias Koppers / sokra, merged June 29, 2026 at 15:53:24Z) **reworks the cost model** of the experimental graph-based CSS chunker (`experimental.cssChunking: 'graph'`, Turbopack only) and **changes the config schema**. This is the first breaking config change in the 16.3 series for the CSS chunker.

**What changed:**

- **`moduleFactorCost` option REMOVED.** Replaced by `weightDistribution`. The old `moduleFactorCost` term penalized a chunk purely by `chunk_size / group_total_size`, which charged a chunk group even for CSS it fully needs and never actually measured overshipping.
- **New per-group cost formula:** `chunk_group_weight * (chunk_size + request_cost)`, summed over the chunk groups that load a chunk, where `chunk_group_weight = group_total_size ^ (-weightDistribution)` is precomputed once per group. `weightDistribution = 0` weights every chunk group equally; higher values give smaller chunk groups a larger weight, so the algorithm overships less to small pages at the cost of more requests. The size weighting subsumes the explicit overship penalty, so the metric stays chunk-local and the greedy merger is unchanged.
- **Defaults retuned:** `requestCost` `20000` → `100000` (bytes) and `weightDistribution` default `0.1`.
- **New config shape (object form):** `{ type: 'graph', requestCost?, weightDistribution? }`. The previous shape `{ type: 'graph', requestCost?, moduleFactorCost? }` is invalid as of canary.71.
- **String-form (`'graph'`) unchanged.** If you don't override the object options, you keep the defaults automatically — no migration needed.

**Migration guide:**

```ts
// next.config.ts

// ✅ String form — no migration, defaults just got retuned
const nextConfig: NextConfig = {
  experimental: {
    turbopack: {
      // ... other turbopack config ...
    },
    cssChunking: 'graph',
  },
}

// ❌ Old object form — will throw a config-schema error on canary.71+ (and later)
const nextConfig: NextConfig = {
  experimental: {
    turbopack: { /* ... */ },
    cssChunking: { type: 'graph', requestCost: 20000, moduleFactorCost: 0.1 },
  },
}

// ✅ New object form
const nextConfig: NextConfig = {
  experimental: {
    turbopack: { /* ... */ },
    cssChunking: { type: 'graph', requestCost: 100000, weightDistribution: 0.1 },
  },
}
```

If you were using the old object form with custom `moduleFactorCost`, you'll need to pick a `weightDistribution` value that produces roughly equivalent chunking decisions. The new defaults are tuned by Vercel — if you were relying on `moduleFactorCost = 0.1` (the previous default), `weightDistribution: 0.1` is the equivalent new default. The new `requestCost` default is 5× higher (100,000 vs 20,000) — if you had `requestCost` set lower than the previous default, you may see fewer requests but more bytes shipped per chunk; if you had it set higher than the new default, you may see more requests but smaller chunks.

**Verification:** `cargo test -p turbopack-core --lib style_groups_graph` (54 tests pass, including a new test asserting `weightDistribution` keeps an unneeded module out of a small group's chunk).

**Practical impact:**

- **If you set `experimental.cssChunking: 'graph'`** (string form) with no object options — **nothing changes**, the new defaults apply automatically.
- **If you set `experimental.cssChunking: { type: 'graph', ... }`** with `moduleFactorCost` or a custom `requestCost` — your build will **fail** with a config-schema error on canary.71+ (and later). Migrate to the new shape using the guide above.
- **If you set `experimental.cssChunking: 'merge'` (default)** or don't set the flag at all — **unaffected**, this PR only touches the `'graph'` algorithm.
- **If you use webpack** (not Turbopack) — **unaffected**, this PR only touches `turbopack-core`'s CSS chunker.

The change is wired end-to-end through the config schema/types, `StyleGroupsAlgorithm::Graph`, and the chunking algorithm. It is not exposed via a codemod — the schema-error is the migration signal, and the fix is a one-line edit in `next.config.ts`.

### Turbopack `experimental.turbopack.chunkingHeuristics` Config Flag (16.3.0-canary.71, [PR #95019](https://github.com/vercel/next.js/pull/95019) by sampoder, June 29, 2026)

A 4-PR stack landed in canary.71 that gives you **fine-grained control over Turbopack's production chunking heuristics**: [#95019](https://github.com/vercel/next.js/pull/95019) (add the config knob), [#95020](https://github.com/vercel/next.js/pull/95020) (thread `chunkingHeuristics` through to chunk groups), [#95021](https://github.com/vercel/next.js/pull/95021) (set `chunking_heuristics` on each `ChunkGroupInfo`), [#95026](https://github.com/vercel/next.js/pull/95026) (use experimental chunking heuristics in the chunker). This **directly complements the canary.71 `cssChunking: 'graph'` rework** — both are part of Vercel's broader "make Turbopack chunking match real-world SPA traffic patterns" effort.

**What's new** — 4 new experimental options under `experimental.turbopack`:

```ts
// next.config.ts
const nextConfig: NextConfig = {
  experimental: {
    turbopack: {
      chunkingHeuristics: {
        requestCost: 20000,                    // bytes; default 20000 — per-request overhead
        clusters: ['/admin', '/dashboard'],   // string[] — route prefixes where chunk merging is preferred (Z+Z case in `make_production_chunks` is more likely → larger merged chunks are better)
        entryPoints: ['/', '/products'],       // string[] — route prefixes where larger merged chunks are prioritized for fast initial page loads (achieved by increasing P(N=1))
        bounceRate: 0.65,                     // number 0..1 — configures P(N=1) and P(N=2) based on site data (e.g. 0.65 = 65% of visits are single-page)
      },
    },
  },
}
```

**What each option does:**

- **`requestCost`** (default `20000` bytes) — same knob used by `cssChunking: 'graph'`'s new formula; the byte-cost of an extra HTTP request, used when computing the trade-off between chunk size and request count.
- **`clusters`** (default `[]`) — route-prefix array that says "these routes tend to navigate to each other frequently; merge their chunks." Models the Z+Z case in Turbopack's `make_production_chunks` (the probability that a user on page Z navigates to page Z'). Higher Z+Z → bigger merged chunks are worth the extra bytes (one request instead of two).
- **`entryPoints`** (default `[]`) — route-prefix array for "fast initial page load" priority. For entry points, the algorithm increases P(N=1) — the probability that the next navigation stays on the same chunk. Larger merged chunks at entry points = fewer total bytes the entry route needs to load.
- **`bounceRate`** (default `0` = disabled) — your site's empirical single-page-visit rate (0..1). Higher bounceRate → P(N=1) higher (chunks merge more aggressively), P(N=2) lower (we don't expect second-page nav). Set to 0.6-0.7 for typical content sites, 0.3-0.4 for SPAs, 0.1-0.2 for document-heavy apps.

**Why this matters:**

- **Without this config**, Turbopack uses a one-size-fits-all chunking heuristic that doesn't know about your app's specific traffic shape. A dashboard app where users click between 5 routes constantly, and a blog where most visitors read one post and leave, get the same chunks — suboptimally.
- **With this config**, you can tell Turbopack "users navigate from `/admin/*` to `/admin/users` to `/admin/users/123` a lot — merge those chunks," and "users land on `/` and often bounce — don't merge chunks at `/`, prioritize initial-load size instead."
- **Symmetric with `cssChunking: 'graph'`** — same author (Sam Poder), same week, same theme: expose the cost function so apps can tune it. If you're already tuning `cssChunking`, you should also be tuning `chunkingHeuristics` if your app has a non-uniform traffic shape.

**When to use:**

- **Single-page apps** with rich navigation between a small set of routes (dashboards, admin panels, IDE-like UIs) — set `clusters` to the top-3 navigation prefixes, `bounceRate` to ~0.3.
- **Content sites** with high bounce (most visitors read 1 page) — set `bounceRate` to your analytics-derived value (0.5-0.7 typical).
- **E-commerce** with predictable funnel paths (home → category → product → cart) — set `clusters` to the category slug pattern, `entryPoints` to `/`.
- **If you don't tune it** — the defaults give the same chunks as before, no behavior change.

**Verification:** the chunker integrates `chunkingHeuristics` via `ChunkGroupInfo` propagation — chunks inherit heuristics from the entry chunk groups that reference them (for entry-points, if a chunk group is referenced by any entry point, it inherits `true`; for clusters, chunk groups inherit all the clusters that the groups that reference them are a part of). Touches `packages/next/src/server/{config-schema.ts,config-shared.ts}` (+33) and `packages/next/src/build/swc/index.ts` (+19). 60 lines of config-schema and config-shared changes, no behavior change for apps that don't set the flag.

**Sources:**
- [PR #95019 — `[turbopack]` Add an experimental `chunkingHeuristics` to `next.config.js`](https://github.com/vercel/next.js/pull/95019)
- [PR #95020 — `[turbopack]` Thread `chunkingHeuristics` through to chunk groups](https://github.com/vercel/next.js/pull/95020)
- [PR #95021 — `[turbopack]` Set `chunking_heuristics` on each `ChunkGroupInfo`](https://github.com/vercel/next.js/pull/95021)
- [PR #95026 — `[turbopack]` Use experimental chunking heuristics](https://github.com/vercel/next.js/pull/95026)

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

### Navigation Inspector Now Works in Safari (16.3.0-canary.73, [PR #95329](https://github.com/vercel/next.js/pull/95329) by Sam Selikoff, merged 2026-07-01T14:34:25Z)

The dev-only Navigation Inspector (the Instant Navigation Testing API that pauses navigations and shows the predicted RSC payload) was Chrome-only before 16.3.0-canary.73: Safari WebKit didn't render the inspector overlay because the inspector's `focus()`-then-screenshot sequence (used to capture the predicted payload's visual state) didn't trigger Safari's compositor the same way it triggered Chrome's. The inspector panel would either render blank or, in some Safari versions, throw `TypeError: undefined is not an object (evaluating 'element.focus')` because Safari's focus model for the inspector's hidden iframe was different.

**The fix** (PR #95329) is in `packages/next/src/client/components/segment-cache/navigation-inspector/dom.ts` (5 lines, +12/-3): the inspector now uses `element.scrollIntoView({ block: 'center' })` followed by a single `requestAnimationFrame` before the screenshot, which both Chrome and Safari WebKit handle identically. The `focus()` call was removed; the inspector's interaction model doesn't actually need focus — it just needs the element to be in the viewport and rendered.

**Practical impact:**

- **If you develop on Safari (or you have Mac teammates on Safari):** upgrade to 16.3.0-canary.73+ to get the inspector working. Before this PR, the inspector panel was effectively Safari-invisible; the instant-navigation cookie was still set/cleared correctly, but the visual overlay never showed.
- **If you develop on Chrome:** no impact. The inspector still works the same way; the screenshot sequence was just made portable.
- **If you develop on Firefox:** not tested. The PR description only mentions Safari; Firefox support may still be partial (the inspector uses Chrome DevTools-specific rendering hooks that Firefox doesn't have).

**Source:** [PR #95329 — `Fix Navigation Inspector in Safari`](https://github.com/vercel/next.js/pull/95329) · [Commit `d6f4cd33`](https://github.com/vercel/next.js/commit/d6f4cd33) · Files: `packages/next/src/client/components/segment-cache/navigation-inspector/dom.ts` (+12/-3, 1 file)

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

### Build Errors Get Code Frames + Real Stacks (canary.71+, June 30, 2026)

Two companion PRs landed on the canary branch on June 30, 2026 that make Cache Components build errors **actually debuggable** instead of opaque framework-code errors. Together they close the most common "my `generateStaticParams` failed the build but I don't know why" footgun. The two PRs are designed to be stacked: #95269 gives you a real stack anchored at your code; #95270 makes sure the build prints the code frame (with file + line numbers + the offending source line), not just the stack.

#### `generateStaticParams` empty-array redbox with user-code stack ([#95269](https://github.com/vercel/next.js/pull/95269), Hendrik Liebau, merged 2026-06-30T09:18:55Z)

**Before this PR (the footgun):** Under `cacheComponents: true`, a dynamic route whose `generateStaticParams` returns `[]` is intentionally an error — CC requires at least one set of params to prerender the route. Previously the failure mode was terrible:

- **Dev:** the request threw an uncaught error, leaving a **blank screen + 500**, no redbox, no stack trace — just a dead page.
- **Prod:** `next build` printed the error message without a stack.

Root cause: `throwEmptyGenerateStaticParamsError` deliberately discarded the stack because the throw happens in framework code (`buildAppStaticPaths`) **after** the user's function has already returned, so the original `Error` had no user-code frame to point at.

**What #95269 does:** adds an SWC transform that emits a `__next_create_empty_gsp_error` factory in any `page`/`layout`/`default` file that exports `generateStaticParams`. The factory's `new Error` is **span-mapped back to the source** — so when it ultimately throws from framework code, the stack trace points at the user's `return []` line (or the most specific frame available). The factory is attached to the segment as `createEmptyParamsError` in `collectSegments`, and `buildAppStaticPaths` calls it when it detects an empty result.

**Anchor logic (most-specific wins):**

| Case | Anchor |
|---|---|
| `export function gsp() { return [] }` — single, literal `return []` | the `return []` line |
| `export function gsp() { if (...) return [...]; return [] }` — computed empty | the function declaration |
| `export { gsp } from './other'` — body in another module | the `export` statement |
| `export { x as generateStaticParams }` — aliased re-export | covered (key is the export, not the declaration name) |
| `export * from './other'` — wildcard re-export | **not covered** — see "Known limitations" below |
| A same-named local helper that is never exported | ignored — only the exported name triggers the transform |

**Transform gating:** registered for both bundlers, gated on `cacheComponents: true`, excluded from the edge runtime (mirrors the existing `debug_instant_stack` wiring). Idempotent — re-running the transform over an already-transformed file is a no-op.

**User-visible result:**

- **Dev:** a proper redbox with a meaningful stack trace pointing at the offending line in your `generateStaticParams`. The custom `EmptyGenerateStaticParamsError` class name was dropped (it only added noise to the logs); now you just see a regular `Error` with the right stack.
- **Prod:** the build fails with a stacked CLI error pointing at the same user-code line, instead of a framework-only message.

**Known limitations:**

- **Wildcard re-exports (`export * from './other'`)** are not covered — the SWC transform keys off the export statement and doesn't follow wildcard re-exports. If you re-export `generateStaticParams` via `export *`, you won't get the user-code stack; you'll get the old framework-only error. Workaround: use a named re-export (`export { generateStaticParams } from './other'`), which IS covered.
- The transform is gated on `cacheComponents: true` — apps not using CC won't get the improved error UX. That's intentional: outside CC, `generateStaticParams` returning `[]` is just "no static params" and isn't an error.

#### Build errors now print code frames for static-worker errors ([#95270](https://github.com/vercel/next.js/pull/95270), Hendrik Liebau, merged 2026-06-30T09:18:56Z)

**The companion fix.** Previously, errors thrown **while collecting page data** during the build (including empty `generateStaticParams` under CC, but also any other throw from `isPageStatic`) printed a source-mapped stack **but no code frame**. Prerender errors from the export worker already showed the offending source lines — but the static worker did not, even though the stack was source-mapped.

**Root cause:** the static worker (`packages/next/src/build/worker.ts`) exposes both `isPageStatic` and `exportPages`, but only `exportPages` called `installCodeFrameSupport` to register the code-frame renderer. When `isPageStatic` ran, the renderer was never registered, so the frame was dropped from the output even though the underlying source map was intact.

**What #95270 does:** installs the native bindings and code-frame support in the static worker entry by wrapping the exposed `isPageStatic`, mirroring what `exportPages` already does. Both installs are idempotent. **The implementation deliberately does NOT do this in `build/utils.ts`**: that module is reachable from `next-server` through the dev server, so importing the code-frame installer there pulls `next-devtools/server/shared` into the production server's file trace, which the `next-server-nft` test rightly rejects (NFT = Node File Trace, the build artifact Vercel uses to keep dev-only modules out of the production bundle). The worker entry is only loaded by build workers, so the renderer stays out of the runtime server.

**User-visible result:** a build error thrown from `generateStaticParams` (or any other static-worker error) now prints the same code frame as a prerender error — file, line number, column, and the offending source line. The frame is **only rendered when the source map carries source content**, which happens under `--debug-prerender` (or `serverSourceMaps: true` in `next.config.ts`); normal minified production builds are unaffected. The `empty-generate-static-params` e2e test is updated to assert the code frame that now appears in the build error for both the literal and the computed empty array cases.

**Combined practical impact of #95269 + #95270:**

- The single most confusing Cache Components failure mode — "my route failed the build with a meaningless framework-only error, and I don't know which line of my `generateStaticParams` is the problem" — is gone on canary.71+. You now get a redbox in dev (real stack anchored at your code) and a code-framed CLI error in prod build.
- If you're running into this failure on a canary.71+ project and you DON'T see a code frame, double-check `serverSourceMaps: true` or pass `--debug-prerender`. The code-frame renderer is registered unconditionally; it just can't render anything if the source map doesn't carry source content.
- If your `generateStaticParams` re-exports via `export * from '...'`, you'll still get the old behavior — file an issue and reference this PR; named re-exports work, wildcard re-exports don't.

**Sources:**
- [PR #95269 — `Surface empty generateStaticParams as a redbox with a real stack` (Hendrik Liebau, canary.71+, June 30, 2026)](https://github.com/vercel/next.js/pull/95269)
- [PR #95270 — `Render a code frame for build errors thrown collecting page data` (Hendrik Liebau, canary.71+, June 30, 2026)](https://github.com/vercel/next.js/pull/95270)

### Turbopack SWC: Constant-Fold `x in y` (#95286, canary.71+, June 30, 2026)

PR [#95286](https://github.com/vercel/next.js/pull/95286) (Niklas Mischkulnig, merged 2026-06-30T09:04:56Z) teaches Turbopack's SWC transform to **constant-evaluate the `in` operator** when `x` is a literal. Pattern:

```ts
const hasFoo = 'foo' in obj   // → const hasFoo = true / false at compile time, when 'obj' is a literal
```

This is part of the ongoing constant-folding pass in Turbopack's SWC integration. The practical effect is small but real:

- **Smaller bundles** for code that does dictionary-shaped checks against literal keys (common in feature-flag libs, dependency-injection containers, enum-shaped maps).
- **Cheaper startup** — the check runs once at compile time instead of once per call site.
- **No code change required** — the optimization is applied transparently by the SWC transformer. If `x` is anything but a literal (`'foo' in someVariable`, `'foo' in obj[key]`), the check stays as runtime code.

Caveat: this is a compile-time transformation, not a TypeScript narrowing — `'foo' in obj` becoming `true` does NOT change the inferred type of `obj`. If you're relying on `in` narrowing for type-guarding (e.g. `if ('x' in val) { val.x }`), keep using the runtime form.

### Next.js config-evaluation time is logged (#94811, canary.71+, June 30, 2026)

PR [#94811](https://github.com/vercel/next.js/pull/94811) (Luke Sandberg, merged 2026-06-30T06:36:48Z) adds timing instrumentation around the `next.config.ts` evaluation step. The duration now appears in dev startup and `next build` startup logs:

```
▲ Next.js 16.3.0-canary.71
- Local: http://localhost:3000
- Network: use --hostname to expose
✓ Ready in 1.2s
✓ Compiled / in 412ms
✓ next.config.ts evaluated in 187ms   ← NEW
```

Use this when diagnosing slow dev startup or slow CI builds: if the config-evaluation line dominates your "Ready in Xs", the bottleneck is in your config (custom plugins, large env-var lookups, expensive module imports in the config file). Move heavy work out of `next.config.ts` or behind a `process.env` gate.

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
- [Turbopack: What's New in Next.js 16.3 (June 29, 2026 — Andrew Imm; dev memory eviction, persistent build cache, Rust React Compiler, import.meta.glob, HMR perf, runtime lazy-load, local PostCSS)](https://nextjs.org/blog/next-16-3-turbopack)
- [`experimental.turbopackMemoryEviction` config reference](https://preview.nextjs.org/docs/app/api-reference/config/next-config-js/turbopackMemoryEviction)
- [`turbopackFileSystemCache` config reference (dev + build)](https://preview.nextjs.org/docs/app/api-reference/config/next-config-js/turbopackFileSystemCache)
- [`turbopackRustReactCompiler` config reference](https://preview.nextjs.org/docs/app/api-reference/config/next-config-js/turbopackRustReactCompiler)
- [`import.meta.glob` turbopack reference](https://preview.nextjs.org/docs/app/api-reference/turbopack#importmetaglob)

### Dev Memory Eviction — `experimental.turbopackMemoryEviction` (16.3 Preview, June 29, 2026 — defaults on)

Turbopack's incremental-compilation model has always traded **memory** for **CPU** — caching more results in memory to avoid recompilation. After three months of work, the Turbopack team is moving that trade-off the other direction, led by an in-memory **eviction** mechanism that flushes cached results back to disk when they're no longer needed. On the Vercel-owned benchmark apps:

| Project | Before | After | Reduction |
|---|---|---|---|
| vercel.com (dashboard) | 21.5 GB | 2 GB | **~90% smaller** |
| nextjs.org | 4,600 MB | 840 MB | **~82% smaller** |

Both numbers are **after compiling 50 routes**. There is no single reduction percentage that applies to every application — your result depends on the size of the route graph, how much of it was touched during the dev session, and how long the session was running. The biggest wins come from evicting the in-memory cache: the dev filesystem cache (introduced in 16.1 as `experimental.turbopackFileSystemCacheForDev`, default on) means evicted entries are still served from disk on the next request.

```ts
// next.config.ts — both flags are on by default in 16.3
const nextConfig: NextConfig = {
  experimental: {
    // dev filesystem cache → required for memory eviction to evict safely
    turbopackFileSystemCacheForDev: true,
    // memory eviction strategy: 'full' (default) | 'single' | false (disable)
    turbopackMemoryEviction: 'full',
  },
}
```

Values:
- **`'full'`** (default in 16.3) — evict entries as soon as they're inactive. The recommended setting; gives the largest memory reduction at the cost of a slightly higher chance of re-reading from disk.
- **`'single'`** — evict one entry at a time on a stricter threshold. Lower memory pressure but less aggressive.
- **`false`** — disable memory eviction entirely. Useful **only** when investigating cache or dev performance regressions where you need Turbopack to behave identically to 16.2.

**Why this matters:** coding agents, IDEs, typecheckers, and linters all consume dev-time memory. Long-running agentic dev sessions (the default mode for an AI agent on a Next.js project) accumulate cached routes indefinitely in 16.2; with `turbopackMemoryEviction: 'full'`, the working set stays bounded even after hours of editing. **If you see OOMs on a long-lived `next dev` against a large app, upgrade to canary.71+ / 16.3 stable before doing anything else.**

### Persistent File-System Cache for Builds — `experimental.turbopackFileSystemCacheForBuild` (16.3 Preview, June 29, 2026 — was already opt-in, now GA)

The dev filesystem cache became stable in 16.1 ([post](https://nextjs.org/blog/next-16-1#turbopack-file-system-caching-for-next-dev)). After months of hardening in production with Vercel-owned sites, **the same persistence is now available for `next build`**. On Vercel's own apps:

| Project | Cold `next build` | Cached `next build` | Speedup |
|---|---|---|---|
| nextjs.org | 21s | 9.2s | **~2.3× faster** |
| vercel.com/home | 66s | 46s | **~1.4× faster** |
| vercel.com/geist | 30s | 5.5s | **~5.5× faster** |

CI benefits the most: **copy `.next` between runs**, and Turbopack will read the cache from disk before compiling any new changes on the next run. The on-by-default behavior in 16.3.0-preview.3 (PR [#94616](https://github.com/vercel/next.js/pull/94616)) is now the documented, stable default.

```ts
// next.config.ts — also reachable via the top-level `turbopack.fileSystemCacheForBuild`
const nextConfig: NextConfig = {
  experimental: {
    turbopackFileSystemCacheForBuild: true,
  },
}

// CI workflow (GitHub Actions, Vercel build cache, buildkite, etc.) —
// restore the .next directory before running next build
- run: next build
```

**Caveats:**

- The cache is keyed by Turbopack's internal version + the resolved module graph. Schema changes invalidate the cache automatically.
- The cache file lives under `.next/` — exclude it from git but include it in CI caches (`actions/cache@v4` on `path: .next/cache` keyed by `package-lock.json` hash, or the equivalent on your CI).
- Webpack builds don't get this benefit (Webpack uses its own disk cache; Turbopack is the path to fast `next build`).

### Experimental Rust React Compiler — `experimental.turbopackRustReactCompiler` (16.3 Preview, June 29, 2026 — added docs page)

The React Compiler stable-on-Turbopack integration (added as `experimental.rustReactCompiler` in [canary.52 — June 16, 2026](#react-compiler-on-turbopack--experimental-rust-port-1630-canary52-june-16-2026) above) is now documented at `preview.nextjs.org/docs/.../turbopackRustReactCompiler.mdx` (PR [#95280](https://github.com/vercel/next.js/pull/95280), canary.71, June 29). The early benchmarks Vercel published against [v0.app](https://v0.app) saw **20–50% faster builds** compared to the Babel transform. To use it:

```ts
// next.config.ts
const nextConfig: NextConfig = {
  reactCompiler: true,           // turn the compiler on (otherwise Turbopack's Rust pass is skipped)
  experimental: {
    rustReactCompiler: true,     // legacy key (canary.52); still works
    turbopackRustReactCompiler: true, // new key in 16.3; both keys are aliases
  },
}
```

The 16.3 Preview post calls out that the Rust compiler is now being released as **experimental** to drive more adoption against large React apps — your real-world build times will vary. The Babel transform (the OG implementation) remains the fallback; the Rust version is opt-in.

### `import.meta.glob` API on Turbopack (16.3 Preview, June 29, 2026 — Vite-compatible)

Turbopack now supports the Vite-compatible **`import.meta.glob`** API, letting you import all modules that match a glob without hardcoding their names:

```ts
// All .mdx files under ./posts — returns an object keyed by matching file paths
const posts = import.meta.glob('./posts/*.mdx')

// Each value is an async function that loads the module by default
for (const path in posts) {
  const post = await posts[path]()  // → MDX module
}
```

Use `eager: true` to import each match immediately (same as static imports — they become part of the bundle):

```ts
const posts = import.meta.glob('./posts/*.mdx', { eager: true })
// posts['/posts/hello.mdx'] is the imported module, not a function
```

The implementation also supports: named imports (`import.meta.glob('./posts/*.mdx', { import: 'default' })`), multiple patterns (`['./posts/*.mdx', './drafts/*.mdx']`), negative patterns (`'!./posts/draft-*.mdx'`), custom search path (`{ root: './src' }`), query strings to pick a loader (`'./styles/*.css?raw'`), and TypeScript type generation for the result.

**Powered by Turbopack's file watcher** — when a file is added to or removed from the match set, it triggers a recompilation in dev mode, so your site always reflects the latest files.

**`--webpack` not supported.** This is a Turbopack-only feature. Webpack-based builds will throw at compile time.

```ts
// Real-world: blog index page
import type { MDXModule } from '*.mdx'

// type-safe: posts: Record<string, () => Promise<MDXModule>>
const posts = import.meta.glob('./posts/*.mdx')

export async function getStaticPaths() {
  return {
    paths: Object.keys(posts).map((path) => ({
      params: { slug: path.replace(/^\/posts\//, '').replace(/\.mdx$/, '') },
    })),
    fallback: false,
  }
}
```

### HMR Cold-Start Win — Single-Subscription Chunk Tracking (16.3 Preview, June 29, 2026)

By analyzing the performance of Turbopack in large Next.js apps at Vercel, the team identified a number of improvements that benefit all Turbopack users. The most significant one **streamlines the tracking of chunks that are loaded on a page** — by collapsing multiple subscriptions to a single one, **dev-server cold start was reduced by over 15% on complex apps**. No config required; it's an HMR-internal optimization.

### Smaller Runtime Size — Lazy-WASM / Lazy-Workers / Lazy-Async-Modules (16.3 Preview, June 29, 2026)

Turbopack ships runtime code to every route that allows it to resolve modules and dynamically fetch new chunks — including code for loading WebAssembly, workers, and top-level async modules. Not every Next.js application uses that functionality. **In 16.3, Turbopack only ships those features when they're needed** and avoids shipping extra runtime code the rest of the time. Reduced client bundle size on apps that don't use WASM/workers.

### Local PostCSS Configuration — `experimental.turbopackLocalPostcssConfig` (16.3 Preview, June 29, 2026)

Monorepos may need different PostCSS transforms for different packages. The new experimental flag lets Turbopack resolve the **closest** PostCSS config to each CSS file before falling back to the project root:

```ts
// next.config.ts
const nextConfig: NextConfig = {
  experimental: {
    turbopackLocalPostcssConfig: true,
  },
}
```

With this enabled, package-level CSS files (e.g. `packages/ui/src/button.css`) pick up `packages/ui/postcss.config.cjs`; application-level CSS picks up the root config. Without the flag, Turbopack only reads `postcss.config.*` at the project root. Use this when your monorepo's design tokens require plugins (mixins, `postcss-import` chains, custom syntaxes) that the consuming app doesn't need.

### Turbopack Compatibility and Reliability (16.3 Preview, June 29, 2026 — final section of the 16.3 Preview Turbopack post)

Next.js 16.3 rolls up all of the fixes from the 16.2 patch line and adds more improvements across **module resolution**, **tracing**, and **HMR**:

- **Correct `import.meta.url` file URLs on Windows** — previously, `import.meta.url` resolved to a `file:///C:/...` URL on Windows but Turbopack passed through the raw forward-slashed path some users wrote, breaking dedupe against the runtime. Now consistent across OSes.
- **Retry chunk fetching on failure** — transient chunk-fetch failures (network hiccup, build process restart) now retry automatically with the same incremental cache key.
- **Better support for `createRequire(new URL(..., import.meta.url))`** — Turbopack now correctly resolves the relative path within `createRequire`, including when the URL has a query string or fragment.
- **Correct `worker_threads` URL resolution** — Turbopack-emitted `new Worker(new URL('./worker.js', import.meta.url))` now resolves the same way at runtime as it does in build, including in production.
- **Support for the `module-sync` export condition** — packages that publish dual ESM/CJS via the `module-sync` exports condition (a Node 22+ feature) now bundle through Turbopack without falling back to the slower sync path.
- **Better errors when webpack loaders crash** — clearer stack traces when a custom loader throws, including the loader name and the file path it was processing.
- **CSS HMR fixes in Safari** — Safari was failing to apply HMR updates when a CSS file was changed via edit-while-replacing (an editor save-then-replace sequence, common with `sed -i`). The fix uses a different update strategy on Safari only; Chrome/Firefox are unaffected.

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

## 16.3 canary.72–86 Performance & Diagnostics Updates (July 1–14, 2026)

Performance-specific surface that landed after this file's last full pass on July 1, 2026. The biggest addition is the **Request Insights** dev-only diagnostics stack (canary.84–86) — a brand-new way to inspect what a route is actually doing in dev. There's also a Turbopack micro-fix cluster (canary.81 + canary.85) that improves chunking determinism, dev-memory stability, and signal handling.

### `experimental.requestInsights` — Dev-Only Request Diagnostics Stack (16.3.0-canary.84+ — full 5-PR stack SHIPPED in canary.86 on 2026-07-14T23:31:39Z)

When diagnosing "why is this route slow?", the standard toolchain is the browser DevTools Network tab + a hand-rolled `console.log` ladder. **Request Insights** is a new dev-only diagnostics stack (5 PRs by `@feedthejim`, shipped across canary.84 → canary.85 → canary.86) that records framework OTEL spans into a bounded in-memory store and exposes them to AI agents + humans via three surfaces: an MCP tool, a CLI, and a DevTools panel. **Dev-only — never enabled in `next build` / `next start`, no telemetry, no PII leak** (sanitizes sensitive attribute keys + URL query strings before snapshotting).

```ts
// next.config.ts — opt in for the dev session
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  experimental: {
    requestInsights: true, // default false; dev-only; no production exposure
  },
}

export default nextConfig
```

**What it records (per request):**

- **OTEL-compatible spans** mirrored from `next/dist/compiled/@opentelemetry/api` when no user OTEL provider is installed (a `LocalSpanRecorder` + `LocalRecordingSpan` satisfy the `Span` interface — existing user-installed OTEL providers are unaffected). Only *framework* spans are mirrored; user spans go through the user provider as usual.
- **Fetch records** deduplicated across completed `AppRender.fetch` spans vs direct fetch metrics (one record per fetch URL per request, not two).
- **HTTP metadata** (method / route / status / URL) per request.
- **Next-specific metadata** (`next.route`, `next.rsc`, `next.segment`, `next.span_name`, `next.span_type`, `next.fetch.cache_reason`, `next.fetch.cache_status`).

**Bounded store:** `MAX_REQUEST_INSIGHTS = 100` — circular buffer of the last 100 requests keyed by `requestId`. Older requests fall off; no I/O cost. The store lives on `globalThis[Symbol.for('@next/local-span-recorder')]` so HMR reloads preserve it.

**Sanitization** (the part that makes this safe to expose):

- **Sensitive attribute keys** (`*token*` / `*secret*` / `*key*` / `*password*` / `*auth*` / `*signature*` / `*jwt*` / etc, case-insensitive) are replaced with `'redacted'` before snapshotting. Only keys in a `SAFE_SPAN_ATTRIBUTE_KEYS` allowlist (`http.method` / `http.route` / `http.status_code` / `http.url` / `next.*` / etc) pass through unchanged.
- **URL query strings** are sanitized via `sanitizeUrl()` before snapshotting (sensitive query params removed).
- **No raw header values**, **no auth headers**, **no request/response bodies**.

### Three ways to consume the snapshot

#### 1. DevTools overlay panel (humans, canary.86+) — `packages/next/src/next-devtools/dev-overlay/components/request-insights/`

A new **"Request Insights"** tab in the dev-overlay DevTools (the same overlay that holds the existing Navigation Inspector + Instant Insights tabs). Shows:

- Sortable table of the last 100 requests with method / route / status / total duration
- Click any row to drill into the span tree (parent-child span relationships, per-span duration, per-span attributes for the safe keys)
- Fetch-record list per request (URL + cache reason + cache status)
- Copy-as-JSON / Copy-as-prompt for sharing with an agent

#### 2. `next experimental-request-insights` CLI (agents / CI, canary.85+)

```bash
# Human-readable summary of the last 100 requests
next experimental-request-insights

# Raw JSON for piping into jq / a script
next experimental-request-insights --json | jq '.requests[] | select(.route == "/api/checkout")'
```

Perfect for shell-only agents and CI scripts that don't have an MCP client.

#### 3. MCP tool (canary.85+) — `packages/next/src/server/mcp/tools/get-request-insights.ts`

```ts
// From the agent side (via the MCP client):
const result = await mcp.call('get_request_insights', {
  requestId: 'req-abc123',       // optional filter
  htmlRequestId: 'html-def456',  // optional filter
})
// Returns: { requests: [{ requestId, spans: [...], fetches: [...], ... }] }
```

Records its own telemetry call (`mcp/get_request_insights`) via the existing `mcpTelemetryTracker`. If `experimental.requestInsights` is off, returns a friendly JSON error rather than throwing. **The originally-proposed `subscribe_to_request_insights` streaming tool was deferred** — for live updates, re-invoke the snapshot tool on a short interval (live updates ARE available over the HMR transport `HMR_MESSAGE_SENT_TO_BROWSER.REQUEST_INSIGHTS_UPDATE` for browser subscribers, but no MCP streaming tool ships in the initial cut).

### Private dev endpoint (canary.84+) — `GET /_next/development/request-insights`

```bash
# Fetch the snapshot directly
curl -sS http://localhost:3000/_next/development/request-insights | jq '.requests[0]'

# Returns 404 with helpful body when the flag is off:
# { "error": "Request Insights is not enabled. Set experimental.requestInsights = true and restart next dev." }
```

Dev-only (`opts.dev` gate) + additionally gated on `blockCrossSiteDEV(req, res, development.config.allowedDevOrigins, opts.hostname)` — same CSRF guard the existing dev-only Next.js internal endpoints use. Same-origin requests from `http://localhost:3000` work; cross-origin requests get a CSRF error.

### Practical agent loop (canary.86+)

1. Agent triggers a slow render → user reports "this is slow"
2. Agent calls `mcp__next-devtools-mcp__get_request_insights` with the suspect `requestId`
3. Reads the span tree, identifies the longest span (e.g. an uncached `fetch` to `/api/users`)
4. Suggests `'use cache'` + `cacheTag('users')` on the data function, or memoization in a Server Component
5. User edits the file → HMR re-renders → Request Insights reflects the new spans immediately
6. Agent re-calls the tool, confirms the span duration dropped

**Full feature breakdown** (with code samples, schema details, file paths, sources) is in `setup.md → experimental.requestInsights`. The 5-PR attribution: [PR #93974](https://github.com/vercel/next.js/pull/93974) (1/5 spans) + [PR #93975](https://github.com/vercel/next.js/pull/93975) (2/5 store) + [PR #93976](https://github.com/vercel/next.js/pull/93976) (3/5 transport) all in canary.84 (2026-07-12) + [PR #93977](https://github.com/vercel/next.js/pull/93977) (4/5 CLI + MCP tool) in canary.85 (2026-07-13) + [PR #93978](https://github.com/vercel/next.js/pull/93978) (5/5 DevTools panel) in canary.86 (2026-07-14).

### `experimental.serverComponentsHmrCancellation` — HMR Perf Win (16.3.0-canary.78 flag → canary.79+.80 activation)

The canary.78 flag (`serverComponentsHmrCancellation`) shipped **inert** — plumbing only, no observable behavior. The actual cancellation logic landed in two follow-up PRs:

- **canary.79 (July 7, 2026)** — [PR #95463](https://github.com/vercel/next.js/pull/95463) `Abort superseded Server Components HMR requests on the client` (andrewimm): when the flag is on, the client now aborts an in-flight Server Components HMR fetch when a newer edit lands. Uses an `AbortController` per refresh + a version-stamped HMR route param so the dev server can detect stale work.
- **canary.80 (July 8, 2026)** — [PR #95486](https://github.com/vercel/next.js/pull/95486) `Cancel a superseded Server Components HMR refresh's server-side work` (unstubbable): server-side counterpart. The dev render pipeline now respects the client abort signal and stops rendering work for a superseded refresh; previously the server kept rendering until completion and discarded the result.

**Perf impact for agents doing rapid iterative edits:** every edit during an active render used to start a new render while the old one ran to completion — measurable CPU + memory waste on large apps with slow Server Components. canary.79+.80 aborts the superseded renders on both client and server, so the dev server spends its time on the *latest* edit instead of finishing abandoned ones. **Opt in:** `experimental.serverComponentsHmrCancellation: true` in `next.config.ts`. Production SSR hardcodes `false` because edge rendering doesn't expose the Node response-close signal the cancellation relies on. **Edge-rendered server components** don't benefit (production-safe cancellation requires the Node response-close signal).

### Turbopack Chunking & Memory Stability — canary.81 + canary.85 Perf Wins

Four small but measurable Turbopack improvements:

- **canary.81 (July 8, 2026)** — [PR #95213](https://github.com/vercel/next.js/pull/95213) + [#95137](https://github.com/vercel/next.js/pull/95137) `Turbopack: don't evict when little memory` — previously, on memory-constrained systems Turbopack could evict cached compilation results even when system memory was abundant (conservative threshold). The new threshold check accounts for actual free memory before triggering eviction; on low-RAM CI runners and containerized dev envs, this means fewer unnecessary evictions and faster rebuilds after edit cycles.
- **canary.81 (July 8, 2026)** — [PR #95606](https://github.com/vercel/next.js/pull/95606) `Turbopack: default ON for 'import with {type: "text"}'` — `import x from './x.txt' with { type: 'text' }` was previously gated behind an opt-in flag; now on by default in Turbopack. Same semantics as the spec, no user code change. Affects any app importing text-as-string assets (i18n JSON files loaded as text, template literals read from disk).
- **canary.85 (July 13, 2026)** — [PR #95579](https://github.com/vercel/next.js/pull/95579) `Turbopack: order CSS modules by chunk-group co-occurrence in linearize` — deterministic CSS module ordering when chunk graphs co-occur (CSS modules imported by multiple chunks now linearize in a stable order based on chunk-group co-occurrence). Eliminates CSS source-order flakiness on rebuilds that could cause minor visual diffs in dev.
- **canary.85 (July 13, 2026)** — [PR #95749](https://github.com/vercel/next.js/pull/95749) `Turbopack: switch make_production_chunks to use floats` — chunk-size math now uses `f64` instead of integer arithmetic, eliminating integer-overflow edge cases that could produce `-1`-byte chunks or weird chunk-size warnings on builds with very large assets.

### Turbopack Clean SIGTERM Termination — canary.85 PR [#95692](https://github.com/vercel/next.js/pull/95692) (July 13, 2026, merged 2026-07-13T21:10:43Z)

Previously, when `next dev` received SIGTERM (Ctrl+C in the terminal, Kubernetes pod shutdown, CI runner timeout), the Turbopack worker threads sometimes left zombie processes or hung cleanup. canary.85 propagates SIGTERM cleanly through the Turbopack process tree — workers exit within ~100ms of the parent receiving the signal, no zombies. **CI impact:** build runners that hit the timeout-and-kill pattern now exit cleanly, so the next build run starts from a clean process state (no leftover dev-server ports in use, no stuck file watchers).

### `experimental.useTscCli` — ~10× Faster Type-Checks (16.3.0-canary.83+, PR [#95639](https://github.com/vercel/next.js/pull/95639) — TypeScript 7 Support)

Adds an experimental flag that routes Next.js's type-check step through the new Go-based TypeScript 7.0 `tsc` subprocess (TS 7.0 GA shipped July 8, 2026) instead of the legacy JS-based compiler. **~10× faster type-check times on large monorepos** (the Go-based `tsc` does type checking in parallel batches that the JS-based one cannot).

```ts
// next.config.ts
const nextConfig: NextConfig = {
  experimental: {
    useTscCli: true, // default false in canary.83
  },
}
```

**canary.86 PR [#95753](https://github.com/vercel/next.js/pull/95753) fixes the spinner UX** — previously the `next build` / `next dev` spinner rendered `Running Typescript ...` and never cleared (the normal pause/resume pattern doesn't work with a subprocess); canary.86+ stops the spinner as soon as the TSC subprocess produces output. **Compiler API is still deferred to TypeScript 7.1** — `useTscCli: true` only affects the type-check step, not the build pipeline. Full TypeScript 7 details in `typescript.md`.

### Next.js 16.2.10 Stable (July 1, 2026) — Current Recommended Stable

Next.js 16.2.10 stable shipped on July 1, 2026 (npm `dist-tag.latest` pointer moved 2026-07-01T20:13:14Z). 4 commits since 16.2.9 — all CI/release/docs backports, no functional changes, safe to upgrade. **For production, use 16.2.10** (the latest stable line); canary.86 is for testing 16.3 features ahead of stable. The skill's existing perf advice (`use cache` + `cacheComponents: true` + `dynamicIO: true` defaults) is unchanged on 16.2.10.

**Sources for this section:**
- [Next.js 16.3.0-canary.84 release](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.84) (Request Insights 1/5–3/5)
- [Next.js 16.3.0-canary.85 release](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.85) (Request Insights 4/5 + Turbopack perf PRs #95579, #95749, #95692)
- [Next.js 16.3.0-canary.86 release](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.86) (Request Insights 5/5 + `useTscCli` spinner fix)
- [PR #95694 — `[turbopack] Optimize the implementation of AutoMap/AutoSet`](https://github.com/vercel/next.js/pull/95694) (canary.87)
- [PR #95261 — `[turbopack] Generate component chunks for each merged group to increase cache hits`](https://github.com/vercel/next.js/pull/95261) (canary.87)
- [PR #95813 — `chore: Remove stale build warning`](https://github.com/vercel/next.js/pull/95813) (canary.87)

### Turbopack AutoMap/AutoSet Backed by Inline `TinyVec` (16.3.0-canary.87, PR [#95694](https://github.com/vercel/next.js/pull/95694) by lukesandberg, merged 2026-07-14T23:35:01Z)

The internal Turbo Tasks storage `AutoMap` and `AutoSet` previously used a `List` variant backed by `SmallVec` (24-byte header: `len` + `cap` + `ptr` all `usize`) even though the list variant never held more than 32 elements — the size was dominated by the header. canary.87 replaces this with the existing `TinyVec` (now enhanced with an `inline` array variant) using a `NonZeroU8` length that reserves a niche the `AutoMap` enum folds its discriminant into. The result: **`AutoMap`'s minimum size drops from 24 → 16 bytes**, which in turn shrinks `TaskStorage` and the `LazyField` enum. Some inline structs in `TaskStorage` are grown to maintain the 128-byte cache-line footprint. **No user-facing change** — purely an internal perf optimization. Expect marginally lower dev-memory pressure in very large monorepos (the existing `experimental.turbopackMemoryEviction: 'full'` flag will work better in combination); no config changes required.

### Turbopack Component Chunks for Each Merged Group (16.3.0-canary.87, PR [#95261](https://github.com/vercel/next.js/pull/95261) by sampoder, merged 2026-07-15T22:57:11Z — WIP, not yet recommended for production)

To increase client-side cache hit rates (currently many chunks are invalidated together because they're emitted as a single merged group), Turbopack now emits per-module chunks **in addition to** the merged-group chunk, reusing the existing `OutputChunkRuntimeInfo.module_chunks` field. The browser can then selectively load just the modules it needs from a chunk without loading the whole chunk. The trade-off is more chunks and more bytes on initial build:

| App | Client chunk files (baseline → PR) | Δ | Client chunk bytes (baseline → PR) | Δ |
|---|---:|---:|---:|---:|
| vercel-site | 3651 → 5245 | +1594 (+43.7%) | 344.5 MiB → 418.4 MiB | +73.9 MiB (+21.5%) |
| vercel-marketing | 2242 → 2634 | +392 (+17.5%) | 163.5 MiB → 180.7 MiB | +17.2 MiB (+10.5%) |
| **Total** | **5893 → 7879** | **+1986 (+33.7%)** | **508.0 MiB → 599.1 MiB** | **+91.1 MiB (+17.9%)** |

**Not yet recommended for production** — the build-size increase is significant, and the cache-hit data hasn't been published yet. Watch the canary channel for the `experimental.turbopackComponentChunks` flag (or whatever it ships as) and the production-scale cache-hit results before enabling. If you have a very large monorepo and your cache-hit rate is the dominant pain point, this is the PR to track; otherwise wait for the analysis.

### `cssChunking: 'graph'` Docs Landed (16.3.0-canary.87, PR [#95693](https://github.com/vercel/next.js/pull/95693) by icyJoseph, merged 2026-07-15T15:47:41Z)

The `experimental.cssChunking: 'graph'` config option (introduced in canary.71) was missing user-facing docs until canary.87. The new docs section ("Balancing requests and grouping") explains how the graph algorithm decides which CSS modules to bundle together, the `weightDistribution` parameter (controls how aggressively the algorithm groups modules vs splitting for cache locality), and includes a "Choosing a strategy" + "Debugging what a route actually uses" walkthrough. The `requestCost` default was retuned from `20000` → `100000` in canary.71; the docs now explain why. Closes Linear DOC-6481 and partially DOC-3447.

**Sources for this section:**
- [PR #93978 — `request insights: add DevTools request panel (5/5)`](https://github.com/vercel/next.js/pull/93978)
- [PR #95463 — `Abort superseded Server Components HMR requests on the client`](https://github.com/vercel/next.js/pull/95463)
- [PR #95486 — `Cancel a superseded Server Components HMR refresh's server-side work`](https://github.com/vercel/next.js/pull/95486)
- [PR #95213 — `Turbopack: don't evict when little memory`](https://github.com/vercel/next.js/pull/95213)
- [PR #95606 — `Turbopack: default ON for 'import with {type: "text"}'`](https://github.com/vercel/next.js/pull/95606)
- [PR #95579 — `Turbopack: order CSS modules by chunk-group co-occurrence in linearize`](https://github.com/vercel/next.js/pull/95579)
- [PR #95749 — `Turbopack: switch make_production_chunks to use floats`](https://github.com/vercel/next.js/pull/95749)
- [PR #95692 — `Turbopack: clean SIGTERM termination`](https://github.com/vercel/next.js/pull/95692)
- [PR #95639 — `experimental TypeScript CLI backend (TS 7 support)`](https://github.com/vercel/next.js/pull/95639)
- [PR #95753 — `Better support the CLI spinner when running the TSC CLI`](https://github.com/vercel/next.js/pull/95753)
- [Next.js 16.2.10 release notes](https://github.com/vercel/next.js/releases/tag/v16.2.10)
- [TypeScript 7.0 GA blog post](https://devblogs.microsoft.com/typescript/typescript-7-0/) (referenced from setup.md / typescript.md)

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
- **Diagnosing slow routes without `experimental.requestInsights`** — if you find yourself hand-rolling `console.log` ladders to figure out "what is this route doing?", enable `experimental.requestInsights: true` in dev (`next.config.ts`) and use the MCP tool / CLI / DevTools panel instead. Much faster diagnosis, agent-friendly output, no production exposure. See the new "16.3 canary.72–86 Performance & Diagnostics Updates" section above for the full feature breakdown.
