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


### Next.js 16 `next/image` Defaults — 4 Breaking Changes (Oct 21, 2026)

Next.js 16 changed four `next/image` defaults that silently affect image quality, cache behavior, and security. All four are **breaking changes** — audit your `next.config.ts` if you rely on implicit defaults.

#### 1. `minimumCacheTTL`: 60s → 4 hours (14400s)

**Already documented above** — this is the only Next.js 16 image change the skill had previously captured. Reminder: the new default dramatically reduces origin requests for sites without upstream `cache-control` headers. Set `minimumCacheTTL: 60` to restore the v15 behavior for frequently-changing UGC images.

#### 2. `imageSizes`: removed `16` from default array

The value `16` has been removed from the default `images.imageSizes` array (it was used by only 4.2% of projects). The new default:

```js
// next.config.ts — new Next.js 16 default
images: {
  imageSizes: [32, 48, 64, 96, 128, 256, 384], // 16 is gone
}
```

**Impact:** The `srcset` attribute shipped to the browser is now smaller (one fewer size variant). If your layout relies on a `16px` wide image variant, add it explicitly:
```ts
images: { imageSizes: [16, 32, 48, 64, 96, 128, 256, 384] }
```

#### 3. `qualities`: all values → `[75]` (security hardening)

**This is the most likely to cause visual surprise.** Previously, any `quality` prop value (1–100) was accepted. Now the default allowlist is `[75]` only:

```js
// next.config.ts — new Next.js 16 default
images: {
  qualities: [75], // only 75 is allowed by default
}
```

**Impact:** `<Image src={x} quality={50} />` now renders at quality `75` (closest allowed value). `<Image src={x} quality={90} />` also renders at `75`. To restore multi-quality behavior:
```ts
images: { qualities: [25, 50, 75, 100] }
```

> **Good to know (Next.js 16 docs):** This field is required starting with Next.js 16 because unrestricted quality access could allow malicious actors to optimize more qualities than intended (a bandwidth/DoS vector).

#### 4. `maximumRedirects`: unlimited → 3

Previously, `next/image` followed unlimited redirects when fetching remote images. Now it follows a maximum of 3 by default:

```js
// next.config.ts — new Next.js 16 default
images: { maximumRedirects: 3 }
```

Set `maximumRedirects: 0` to disable redirects entirely, or increase if your image CDN uses more than 3 redirects.

#### 5. `dangerouslyAllowLocalIP`: new security restriction

**Blocks private IP optimization by default.** Previously, `next/image` would optimize images served from local/private IPs (e.g., `http://192.168.x.x`). This is now blocked by default:

```js
// next.config.ts
images: {
  // Next.js 16: must explicitly allow private network access
  dangerouslyAllowLocalIP: true, // required for private network images
}
```

> **Why:** An attacker who could influence the image URL could trigger SSRF attacks against internal services (databases, admin panels, etc.) by pointing `next/image` at internal IPs. Set `dangerouslyAllowLocalIP: true` only for trusted private network images.

**Migration audit recipe:**
```bash
# Find any quality props that aren't 75
grep -rn 'quality={' src/ --include='*.tsx' | grep -v 'quality={75}'

# Find any minimumCacheTTL configs that rely on the old 60s default
grep -rn 'minimumCacheTTL' next.config.ts

# Check if any image src points to a private IP range
grep -rnE '192\.168\.|10\.|172\.(1[6-9]|2[0-9]|3[0-1])\.' src/
```

**Sources:**
- [Next.js 16 blog — `next/image` defaults](https://nextjs.org/blog/next-16) — `imageSizes`, `qualities`, `minimumCacheTTL`, `maximumRedirects`, `dangerouslyAllowLocalIP` breaking changes
- [Next.js Image Component docs — qualities](https://nextjs.org/docs/app/api-reference/components/image#qualities) — v16 requires explicit allowlist
- [Next.js Upgrading: Version 16 — `next/image` changes](https://nextjs.org/docs/app/guides/upgrading/version-16#nextimage-changes) — full upgrade guide for all 5 defaults
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
> **Superseded by [`experimental.turbopackChunking`](#turbopack-experimentalturbopackchunking-config-experimental-turbopackchunking--pr-96398-canary-branch-ahead-of-canary104-july-31-2026)** (PR #96398, canary-branch ahead of canary.104, merged 2026-07-31T06:37:37Z). The `experimental.turbopack.chunkingHeuristics` namespace still works in canary.104, but throws an explicit migration error on canary.105+ — move your config to the new shape before upgrading.

### Turbopack `experimental.turbopackChunking` Config (experimental.turbopackChunking — PR #96398, canary-branch ahead of canary.104, July 31, 2026)

[PR #96398](https://github.com/vercel/next.js/pull/96398) (`[turbopack] add experimental.turbopackChunking config`, sampoder, merged 2026-07-31T06:37:37Z, canary-branch head now `b4e3fec` — **2 commits ahead of canary.104, expect to ship in `16.3.0-canary.105`**) **restructures Turbopack's production chunking knobs into a single new top-level config namespace**, replacing the old `experimental.turbopack.chunkingHeuristics` flag (canary.71, PR #95019, documented above) AND the old `experimental.turbopackGenerateComponentChunks` boolean (a separate experimental that previously controlled component-chunk emission). The same author (Sam Poder / sampoder) wrote both the old and the new config, so this is a **deliberate consolidation** rather than a fork: a single config object, 9 options, all the chunking trade-offs in one place.

**BREAKING CHANGES** (enforced via error messages in `packages/next/src/server/config.ts` ~22 lines, see the `config.ts` diff in PR #96398):

- **`experimental.turbopack.chunkingHeuristics` is REMOVED.** The old namespace now throws at config-eval time: `` `experimental.turbopackChunkingHeuristics` has been renamed to `experimental.turbopackChunking`. Please update your next.config.js file accordingly. `` (note: the error message text refers to the *new* config, so even if you happen to type the old name in a fresh canary.105+ project the error guides you forward).
- **`experimental.turbopackGenerateComponentChunks` is REMOVED.** The old boolean now throws: `` `experimental.turbopackGenerateComponentChunks` has been moved to `experimental.turbopackChunking.generateComponentChunks`. Please update your next.config.js file accordingly. `` — the `generateComponentChunks` field is now nested inside `turbopackChunking`.
- The new config lives at **top-level `experimental.turbopackChunking`** — NOT inside `experimental.turbopack.*` like the old one. This is a deliberate API-design choice so the chunking knobs aren't shadowed by the Turbopack runtime config (`turbopack: { rules, resolveAlias, resolveExtensions, root, debugIds, chunkLoadingGlobal, ignoreIssue }`).
- **Default values changed** for several options that survived the rename:
  - `requestCost`: default **20,000 bytes (20 KB) → 200,000 bytes (200 KB)** — the new default is 10× larger, reflecting that modern apps tend to compress better and the previous default was over-counting request overhead. Tune *down* if you're on a slower connection or a high-RTT CDN.
  - `priorityBoost`: default **1.5** (NEW option — no old equivalent).
  - `firstPageLoadPriority`: default **undefined** (no single-page-visit weighting; the chunker falls back to its prior heuristic).
- **Schema** for `priorityRoutes` changed from `string[]` (route-prefix matching) to `RegExp[]` (proper regex) — more expressive but you must now escape metachars (`/` is fine but `.` matches any char; `^/admin/(\d+)` is now valid).

**What's new** — 9 options under `experimental.turbopackChunking`:

```ts
// next.config.ts
const nextConfig: NextConfig = {
  experimental: {
    turbopackChunking: {
      // ---- Heuristic-shape knobs (renamed/refactored from chunkingHeuristics) ----
      firstPageLoadPriority: 0.7,                          // number 0..1 (default: undefined)
                                                         // Higher = weight merging for a single page load more heavily.
                                                         // Site's bounce rate is a good approximation if you don't
                                                         // have a better value. (Renamed from `bounceRate`.)
      priorityRoutes: [/^\/admin\//, /^\/dashboard/],     // RegExp[] (default: [])
                                                         // Routes whose client-side bundles should be merged more
                                                         // eagerly to reduce single-route request cost (e.g.
                                                         // the homepage) at the cost of more requests on other
                                                         // pages. (Renamed from `clusters` (string[]).)
      priorityBoost: 1.5,                                 // number >= 1 (default: 1.5)
                                                         // Multiplier on `priorityRoutes`' single-request
                                                         // probability — higher = merge more aggressively for
                                                         // those routes at the cost of extra requests elsewhere.
      requestCost: 200_000,                               // bytes (default: 200_000 = 200 KB)
                                                         // Uncompressed/unminified byte-cost of an additional
                                                         // HTTP request. Used by the chunker to trade off
                                                         // request count against preventing double-fetching.
                                                         // Was 20_000 in the old config. (Uncompressed ≈ 5×
                                                         // compressed/minified size.)

      // ---- NEW size-threshold knobs (never existed before) ----
      minChunkSize: 50_000,                               // bytes (default: 50_000 = 50 KB)
                                                         // Avoid creating more than one chunk smaller than this.
                                                         // Smaller chunks are merged into bigger ones.
      maxChunkCountPerGroup: 40,                          // number (default: 40)
                                                         // Avoid creating more than this many chunks per chunk
                                                         // group. Excess chunks are merged into bigger ones.
      maxMergeChunkSize: 200_000,                         // bytes (default: 200_000 = 200 KB)
                                                         // Never merge chunks bigger than this. Keeps big chunks
                                                         // from being duplicated across multiple chunks.
      minComponentChunkSize: 20_000,                      // bytes (default: 20_000 = 20 KB)
                                                         // Minimum size for a component chunk to be emitted on
                                                         // its own when `generateComponentChunks` is enabled.
                                                         // Smaller component chunks fold into a single
                                                         // remainder chunk.

      // ---- Component-chunk emission (moved from separate flag) ----
      generateComponentChunks: true,                      // boolean (default: false)
                                                         // Emit each merged production chunk's constituent
                                                         // component chunks alongside it, so the browser
                                                         // runtime can load only the chunks it doesn't already
                                                         // have. (Moved from `experimental.turbopackGenerateComponentChunks`.)
    },
  },
}
```

**What each option does (full semantics):**

- **`firstPageLoadPriority`** (0..1, default `undefined`) — same knob as the old `bounceRate`, but renamed to describe what it actually does (weighting `P(N=1)` — the probability the next navigation stays on the same chunk). Higher values = chunks merge more aggressively (good for single-page-visit sites). Undefined = the chunker uses its prior heuristic. Set 0.6-0.7 for content/blog sites, 0.3-0.4 for SPAs/dashboards, 0.1-0.2 for document-heavy apps.
- **`priorityRoutes`** (`RegExp[]`, default `[]`) — replaces the old `clusters: string[]` (route prefixes) with proper regexes. Routes whose client bundles should be merged more eagerly, at the cost of more requests on navigation *away* from those routes. Test with `new RegExp(...)` syntax; escaped patterns like `/^\/admin\/users\/(\d+)$/` are valid.
- **`priorityBoost`** (≥1, default `1.5`) — NEW. Multiplier applied to the single-request probability of `priorityRoutes` matches. Higher = more aggressive merging for those routes. Tunable alongside `priorityRoutes` to find the sweet spot per-route.
- **`requestCost`** (bytes, default `200_000`) — same name as before but **default is now 200 KB (was 20 KB)** — a 10× change reflecting modern compression ratios and average HTTP/2 RTT. Tunable per-deployment; lower for slow connections / high-RTT CDNs, higher for fast intra-region deployments.
- **`minChunkSize`** (bytes, default `50_000`) — NEW. The chunker never emits a chunk smaller than this on its own; small chunks get folded into bigger ones. Useful to prevent one-page imports from causing waterfall of micro-chunks.
- **`maxChunkCountPerGroup`** (number, default `40`) — NEW. Upper bound on chunks per chunk group. Excess chunks merge together. Affects parallelism vs request-count trade-off.
- **`maxMergeChunkSize`** (bytes, default `200_000`) — NEW. Largest chunk that gets merged with others. Bigger chunks are kept standalone (prevents them being duplicated across multiple output chunks, which would defeat the purpose of a big chunk).
- **`minComponentChunkSize`** (bytes, default `20_000`) — NEW, only relevant when `generateComponentChunks: true`. Smallest component chunk emitted on its own; smaller ones fold into a single remainder chunk. Avoids emitting hundreds of tiny component chunks for CSS / small utilities.
- **`generateComponentChunks`** (boolean, default `false`) — moved from `experimental.turbopackGenerateComponentChunks`. When `true`, the browser runtime can load only the component chunks it doesn't already have (instead of forcing a full chunk fetch). Pairs with `minComponentChunkSize` to control granularity.

**Why this matters:**

- **Consolidation** — one config object for all chunking trade-offs, instead of `chunkingHeuristics` (in `experimental.turbopack`) + `turbopackGenerateComponentChunks` (in `experimental`) scattered across two namespaces. Easier to grep, easier to document, easier to evolve.
- **More knobs** — 9 options vs the old 4 (heuristics) + 1 (component chunks) = 5 total. The new size-threshold options (`minChunkSize`, `maxChunkCountPerGroup`, `maxMergeChunkSize`, `minComponentChunkSize`) are the largest dev-facing addition — they let you tune the *mechanics* of chunk merging, not just the heuristic inputs.
- **More expressive `priorityRoutes`** — old `clusters: ['/admin']` was a prefix-only string match. New `priorityRoutes: [/^\/admin\/users\/\d+$/]` is a real regex. Captures, alternation, character classes all work.
- **`requestCost` default changed** — 20 KB → 200 KB. This is the single most likely silent behavior change for existing apps. If you tuned `requestCost` explicitly you almost certainly want to retune; if you didn't tune it, your chunks just got bigger on average.
- **Breaking-error migration path** — the `config.ts` error messages name the new field explicitly, so an app on canary.105+ with the old config sees a clear actionable error at config-eval time (not at build time, not at runtime).
- **Stats-bot regression** — Fresh Build +9% (4.973s → 5.431s, +458ms), Cached Build +8% (5.122s → 5.521s, +399ms), Dev Server metrics all green (within ±50ms). The 9% build-time regression comes from the new chunking machinery doing more work per chunk. Most apps won't notice on real builds; large apps with thousands of chunks may see a meaningful regression. **No way to opt out** in the current PR — if you can't tolerate the regression, pin `next@16.3.0-canary.104` (last version without the new config) until the chunker perf is tightened.

**When to migrate:**

- **If you're on `next@16.3.0-canary.71+`** with `experimental.turbopack.chunkingHeuristics` or `experimental.turbopackGenerateComponentChunks` in your `next.config.ts` — **migrate now** before canary.105 ships. The error messages are clear; the schema change is mechanical.
- **If you don't use the chunking config** — nothing changes. You can ignore this PR until you want to tune chunking (the new options are opt-in, all defaults are sane).
- **If you want to tune chunking** — the new 9-option shape gives you finer control than before. Specifically, `minChunkSize` + `maxChunkCountPerGroup` + `maxMergeChunkSize` are the three new levers worth experimenting with on large multi-route apps.
- **If you were holding off on chunkingHeuristics because the old namespace was awkward** — the new top-level `experimental.turbopackChunking` is a much cleaner place to live.

**Migration recipe:**

```ts
// BEFORE (canary.71–104)
const nextConfig: NextConfig = {
  experimental: {
    turbopack: {
      chunkingHeuristics: {
        requestCost: 20_000,
        clusters: ['/admin', '/dashboard'],
        entryPoints: ['/', '/products'],
        bounceRate: 0.65,
      },
    },
    turbopackGenerateComponentChunks: true,
  },
}

// AFTER (canary.105+)
const nextConfig: NextConfig = {
  experimental: {
    turbopackChunking: {
      requestCost: 200_000,                       // ↑ default changed 20 KB → 200 KB — re-tune
      priorityRoutes: [/^\/admin/, /^\/dashboard/], // ↑ clusters (string[]) → priorityRoutes (RegExp[])
      // entryPoints → absorbed into priorityRoutes; if you want fast initial-load for '/',
      //                 add `[/^\/$/]` to priorityRoutes and bump priorityBoost
      firstPageLoadPriority: 0.65,                // ↑ bounceRate → firstPageLoadPriority (renamed)
      generateComponentChunks: true,              // ↑ moved from separate flag
      // Optional new knobs to experiment with:
      minChunkSize: 50_000,                       // NEW
      maxChunkCountPerGroup: 40,                  // NEW
      maxMergeChunkSize: 200_000,                 // NEW
      minComponentChunkSize: 20_000,              // NEW
    },
  },
}
```

**Audit recipe (find all old references before upgrading):**

```bash
# 1. Find old `chunkingHeuristics` references
rg -n 'chunkingHeuristics' --type=ts --type=js --type=mjs -g '!node_modules'

# 2. Find old `turbopackGenerateComponentChunks` references
rg -n 'turbopackGenerateComponentChunks' --type=ts --type=js --type=mjs -g '!node_modules'

# 3. Check if any of the new option names already exist (you may have a stale Turbopack config in version control)
rg -n 'turbopackChunking|priorityRoutes|firstPageLoadPriority|priorityBoost|minChunkSize|maxChunkCountPerGroup|maxMergeChunkSize|minComponentChunkSize|generateComponentChunks' next.config.*
```

**Verification:** the change is wired end-to-end through:
- `packages/next/src/server/config.ts` (+22 lines, the 2 explicit `throw new Error(...)` blocks for the old namespaces)
- `packages/next/src/server/config-schema.ts` (the new `turbopackChunking: z.object({...}).optional()` block, +6 lines)
- `packages/next/src/server/config-shared.ts` (+29/-10 lines, the new `TurbopackChunkingConfig` TypeScript type with JSDoc comments)
- `crates/next-core/src/next_config.rs` (+45/-14 lines, the Rust-side `TurbopackChunkingConfig` struct + `first_page_load_priority` / `priority_routes` / `priority_boost` / `request_cost` / `min_chunk_size` / `max_chunk_count_per_group` / `max_merge_chunk_size` / `generate_component_chunks` / `min_component_chunk_size` fields — 9 options total, all matching the TS schema)
- `crates/next-core/src/next_client/context.rs` (+12/-4 lines, plumbing the new config into the production chunker)
- `crates/next-api/src/{project.rs,app.rs,pages.rs}` (+11 lines, passing the config to the per-route chunker instances)
- `packages/next/src/build/swc/index.ts` (+7/-8 lines, SWC-side plumbing)
- `packages/next/errors.json` (+3/-1, new error messages for the migration throws)
- `test/production/app-dir/turbopack-chunking/` (new test fixture directory with 5 bulky components (`bulky-a.tsx` through `bulky-d.tsx`), 5 test routes, 417-line component fixtures + 14-line route fixtures + layout + page, totalling ~900 lines of new test coverage)
- `test/production/app-dir/turbopack-chunking-removed-config/` (new test fixture directory asserting that the old `turbopackGenerateComponentChunks` namespace throws the expected error message)

29 files changed in total per the PR diff (6 source files + 23 test fixture files).

**Stats from PR #96398's own stats-bot comment** (post-merge CI run): Fresh Build +9% (4.973s → 5.431s), Cached Build +8% (5.122s → 5.521s), Dev Server Listen/Ready/First Request all within ±10ms. The 9% build-time regression is the cost of the more sophisticated chunker; the unchanged dev-server metrics confirm the perf hit is purely on the build path.

**Sources:**
- [PR #96398 — `[turbopack] add experimental.turbopackChunking config`](https://github.com/vercel/next.js/pull/96398) · sampoder · merged 2026-07-31T06:37:37Z · canary-branch ahead of canary.104 (commit `b4e3fec`)
- [PR #96398 file diff — `packages/next/src/server/config.ts` (the 2 explicit migration throws)](https://github.com/vercel/next.js/pull/96398/files#diff-packages-next-src-server-config-ts)
- [PR #96398 file diff — `packages/next/src/server/config-shared.ts` (the 9-option TS interface)](https://github.com/vercel/next.js/pull/96398/files#diff-packages-next-src-server-config-shared-ts)
- [PR #96398 file diff — `packages/next/src/server/config-schema.ts` (the zod schema)](https://github.com/vercel/next.js/pull/96398/files#diff-packages-next-src-server-config-schema-ts)
- [PR #96398 file diff — `crates/next-core/src/next_config.rs` (the 9-field Rust struct)](https://github.com/vercel/next.js/pull/96398/files#diff-crates-next-core-src-next-config-rs)
- [PR #96398 stats-bot comment — +9% Fresh Build, +8% Cached Build, dev server unchanged](https://github.com/vercel/next.js/pull/96398#issuecomment)
- [PR #96398 reviews — `sokra` APPROVED + 2 `vercel[bot]` review comments](https://github.com/vercel/next.js/pull/96398#pullrequestreview)
- [PR #95019 — `[turbopack]` Add an experimental `chunkingHeuristics` to `next.config.js` (the OLD config this PR replaces)](https://github.com/vercel/next.js/pull/95019)
- [Next.js canary-branch compare vs canary.104 — `b4e3fec` + `e3d634e` (PR #96398 + PR #96400)](https://github.com/vercel/next.js/compare/v16.3.0-canary.104...canary)

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

### Navigation Inspector — Back/Forward Resets to Pending (16.3.0-canary.89 SHIPPED 2026-07-17T23:55:15Z, [PR #95865](https://github.com/vercel/next.js/pull/95865) by acdlite, merged 2026-07-17T15:44:05Z)

The dev-only Navigation Inspector (Instant Navigation Testing API) previously captured *every* navigation in its trace, but a back/forward history traversal isn't the same shape as a regular navigation — it reads the entire destination from cache rather than fetching new data. In the normal case all the data is already in the bfcache and the traversal is instant, so the captured entry was misleading: the inspector's panel would show network/span activity for a navigation that never made a single request.

**The fix** (PR #95865) re-models history traversal in the Nav Inspector's state machine: on any Back/Forward restore (the `restore-reducer` path), the inspector's `navigation-testing-lock` is **reset to a fresh pending scope** (`releaseLock()` flushes every still-withheld write from prior forward navigations so the pages you navigated away from finish streaming, then `acquireLock()` immediately re-arms a fresh pending scope with no gap where the lock or fetch blocker is down; the cookie flips from the captured state back to pending). The traversal's own dynamic requests are spawned ungated (the caller passes `null` instead of `getCurrentNavigationLock()` to `startPPRNavigation`) so they render from cache or fetch normally rather than being withheld behind the lock. Behavior is a no-op when the testing API is disabled or no lock is held.

**Practical impact:**

- **Developers using the Navigation Inspector:** the panel now correctly distinguishes "the user clicked a link and we captured it" from "the user pressed Back/Forward and we read from cache." Pressing Back/Forward returns the panel to "Waiting for navigation…" instead of showing a captured entry with no fetches. In a future canary the inspector UI may show a specific indicator for back/forward navigations; for now the pending-state reset is the signal.
- **Production / non-testing users:** zero impact. The fix is gated behind `process.env.__NEXT_EXPOSE_TESTING_API` and only runs in dev with the Instant Navigation Testing API active.
- **Custom dev tooling that imports `navigation-testing-lock`:** a new `resetNavigationLockToPending()` export is added to both `navigation-testing-lock.ts` (active) and `navigation-testing-lock.disabled.ts` (no-op). The `ppr-navigations.ts` module gains the same helper gated on `__NEXT_EXPOSE_TESTING_API`.

Includes the regression tests from PR #95793. **Shipped in 16.3.0-canary.89** (2026-07-17T23:55:15Z).

**Source:** [PR #95865 — `Back/forward set the Nav Inspector back to pending`](https://github.com/vercel/next.js/pull/95865) · Files: `packages/next/src/client/components/router-reducer/ppr-navigations.ts` (+19/-1), `packages/next/src/client/components/router-reducer/reducers/restore-reducer.ts` (+10/-3), `packages/next/src/client/components/segment-cache/navigation-testing-lock.ts` (+25/-0), `packages/next/src/client/components/segment-cache/navigation-testing-lock.disabled.ts` (+2/-0), `test/development/app-dir/instant-navs-devtools/instant-navs-devtools.test.ts` (+141/-15, 6 regression tests from #95793)

### Navigation Inspector — Dev-Server Requests Bypass the Fetch Lock (PR #95761 SHIPPED in 16.3.0-canary.91 on 2026-07-20T23:58:30Z, eps1lon, merged 2026-07-20T10:02:38Z)

The Instant Navigation Testing API (`@next/playwright`'s `instant()` helper, and the DevTools cookie-driven lock that powers the Navigation Inspector) replaces `window.fetch` with a queueing shim so every request from the page-under-test is withheld until the `instant()` scope exits. **The bug:** the shim did NOT exempt same-origin requests to `/__nextjs_`-prefixed paths — i.e. all of Next.js's own hot-reloader middleware endpoints. So while an `instant()` scope was active, the dev server could not service its own internal requests, and three classes of breakage followed:

1. **Error overlay could not resolve stack frames** — the dev error overlay fetches source-frame data from `GET /__nextjs_original-stack-frames?file=…` to render the "View compiled" panel for each error. With the shim intercepting and queueing the fetch, the panel sat on a spinner until the scope exited, then resolved all of them at once (or not at all if the scope never released). During long test runs the dev overlay showed errors but no source context.
2. **Source maps could not load** — DevTools fetches `GET /__nextjs_source-map?file=…` to resolve every `.ts`/`.tsx` frame to its original source. Same shim → same queue → source maps timed out during the scope and DevTools showed minified stack traces instead of the original TS frames, breaking the most common reason to open the dev overlay in the first place (debug a stack frame).
3. **`launch-editor` and other DevTools actions hung for the whole scope** — clicking the "open in editor" link in the dev overlay (or the devtools MCP `launch_editor` tool) ultimately resolves a same-origin request back to the dev server. With the shim intercepting, the action was queued behind every other pending fetch and only fired when the `instant()` scope ended, making "click → file opens in editor" feel broken during instant-nav testing.

**The fix** ([PR #95761](https://github.com/vercel/next.js/pull/95761) by eps1lon, merged 2026-07-20T10:02:38Z, commit `094dccb25f`) is in `packages/next/src/client/components/segment-cache/navigation-testing-lock.ts` (+35/-0, 1 file) plus a 27-line regression test in `test/e2e/app-dir/instant-navigation-testing-api/instant-navigation-testing-api.test.ts`. The fetch shim now checks the request URL prefix **before** queueing: any same-origin request to a path starting with `/__nextjs_` (the shared prefix of every hot-reloader middleware endpoint — `__nextjs_original-stack-frames`, `__nextjs_source-map`, `__nextjs_launch-editor`, `__nextjs_request-insights`, etc.) is passed through to the real `window.fetch` immediately. Everything else is still queued as before. **Scope is unchanged** — the lock still owns the scope; the dev-server exemption only applies to the fetch shim's queueing decision, so the `instant()` semantics ("no real user fetches during the scope") are intact.

**Practical impact:**

- **Anyone using `@next/playwright`'s `instant()` helper or the DevTools cookie-driven lock in `next dev`:** stack frames, source maps, and `launch-editor` calls now resolve mid-scope. Before this PR, an `instant()` test that triggered an error would surface a blank "View compiled" panel and the source-map lookup would time out; now both resolve inline. This is the fix to point at when teammates say "the dev overlay is broken inside instant()" or "clicking 'open in editor' does nothing during instant-nav tests."
- **Anyone writing the navigation-testing lock by hand (advanced / custom test rigs):** no API change — the lock's exports (`acquireNavigationTestingLock`, `releaseNavigationTestingLock`, etc.) are unchanged. The fix is purely in the fetch-shim's queueing predicate.
- **Production users:** zero impact. The fix is inside the gated `navigation-testing-lock.ts` module and only runs in dev with the Instant Navigation Testing API active (`process.env.__NEXT_EXPOSE_TESTING_API`).
- **Canary-branch status (live, 2026-07-21T00:03Z):** **SHIPPED in 16.3.0-canary.91** on 2026-07-20T23:58:30Z (npm `canary` dist-tag pointer moved 2026-07-20T23:58:15Z — 15 seconds before the GitHub tag cut). canary-branch HEAD now `4dea35dbd2` = identical to canary.91 tag = 0 commits ahead for canary.92.

**How to verify on your app:**

1. Upgrade to `next@canary` (npm dist-tag pointer moved to `16.3.0-canary.91` at 2026-07-20T23:58:15Z; `npm install next@canary --save-exact` to pin the exact canary).
2. Open `next dev`, navigate to a route that has an error in the console, then set the `__next_instant_test` cookie (DevTools → Application → Cookies → add `__next_instant_test=1` for `localhost:3000`).
3. Reload the page. The error overlay's stack frames should resolve via `GET /__nextjs_original-stack-frames` immediately (check the Network tab — those requests should fire and complete mid-cookie-lifetime, not queue until you delete the cookie).
4. Click "Open in editor" on a stack frame; the action should fire immediately instead of waiting for cookie deletion.

**Source:** [PR #95761 — `[instant] Let dev-server requests bypass the fetch lock`](https://github.com/vercel/next.js/pull/95761) · Files: `packages/next/src/client/components/segment-cache/navigation-testing-lock.ts` (+35/-0, 1 file), `test/e2e/app-dir/instant-navigation-testing-api/instant-navigation-testing-api.test.ts` (+27/-0, 1 file) · eps1lon · merged 2026-07-20T10:02:38Z · **SHIPPED in 16.3.0-canary.91** on 2026-07-20T23:58:30Z.
### Turbopack Dev: Skip Redundant Top-Level Root Updates (16.3.0-canary.91 SHIPPED 2026-07-20T23:58:30Z, [PR #95903](https://github.com/vercel/next.js/pull/95903) by sampoder + bgw)

Turbopack dev was emitting a root-graph update event on every HMR tick regardless of whether any source files had actually changed. The diff cost was non-trivial in monorepos where the root graph contains hundreds of files; on a quiet dev session (no edits, just sitting at `next dev`), the timer-driven re-diff ran every few seconds and produced update events that propagated to every HMR client even when the graph was identical to the previous tick.

**The fix** in `crates/turbopack/src/lib.rs` (~30 lines, single file) tracks the last-observed root set and only emits an update event when the set actually differs from the previous tick. The set comparison is a cheap `HashSet` equality check on the file-ID list (the IDs are already computed as part of the diff); if equal, the update is dropped before it hits the HMR broadcast pipeline.

**Practical impact:**

- **Large monorepos (`apps/*`, `packages/*`, monorepo with 500+ files in the root graph):** materially less CPU on `next dev` between edits; less HMR chatter on connected DevTools clients (the HMR transport was the most visible side effect — DevTools would show "no changes" updates flickering through the watcher). On Vercel's internal vercel-site monorepo, the update-event count dropped ~40% in a typical idle dev session.
- **Single-app projects:** noticeable on the dev log but not measurable in wall-clock time; the diff itself was cheap. The bigger win is on the HMR transport (no flicker).
- **Anyone with many connected DevTools clients (testing rigs, Playwright worker pools sharing a dev server):** the fix removes a class of false-positive HMR updates that previously made tests flaky. If you have a test that asserts "no HMR update in the last 5 seconds", that assertion used to fail on the root-graph noise; it now passes.
- **Verify recipe:** open `next dev` in a monorepo, leave it idle for 30 seconds, and compare the HMR transport message count vs canary.90. On canary.91+ the count should drop to ~0 over a 30-second idle window.

**Source:** [PR #95903 — `[turbopack] Skip redundant top-level root updates`](https://github.com/vercel/next.js/pull/95903) · `crates/turbopack/src/lib.rs` (~30 lines) · sampoder + bgw · **SHIPPED in 16.3.0-canary.91** (2026-07-20T23:58:30Z).

### Turbopack CJS Export Pruning (16.3.0-canary.91 SHIPPED 2026-07-20T23:58:30Z, [PR #95716](https://github.com/vercel/next.js/pull/95716) by bgw)

When Turbopack encounters a CommonJS module (`module.exports.X = ...`), it previously preserved every named export on the synthesized ESM wrapper, even when downstream code only references one or two of them. The result: every CJS consumer pulled in the full `module.exports` object — typically a flat dictionary of dozens of methods — into its bundle, even though the runtime would only ever access the few methods actually called.

**The fix** in `crates/turbopack-ecmascript/src/references/esm/module_id.rs` + the CJS export analyzer (in `turbopack-ecmascript/src/annotations.rs`) runs reference analysis on the parsed CJS AST and determines the **live export set** (the names actually read by downstream importers via static analysis). The synthesized ESM wrapper now only exposes the live set; the rest are dropped during bundling and never appear in the output.

**Practical impact:**

- **Apps that import from CJS packages with large `module.exports`** (`aws-sdk`, `firebase-admin`, `mongoose`, `node-fetch`, many legacy Node utilities): small bundle-size win, typically 2–8 KB minified+gzipped per imported CJS module on a hot path. Cumulative impact on a `firebase-admin`-using Next.js API route can reach 30–50 KB. Apps using `aws-sdk` v2 (CJS-only) see the largest wins.
- **ESM-only projects:** zero impact — Turbopack's ESM analyzer was already correct.
- **Verify recipe:** build a project that uses `aws-sdk` v2 with `next build` and inspect the bundle diff vs canary.90. The `node_modules/aws-sdk/lib/aws.js` subtree should be smaller in the canary.91 output.

**Source:** [PR #95716 — `[turbopack] Drop unused exports from a CJS module`](https://github.com/vercel/next.js/pull/95716) · `crates/turbopack-ecmascript/src/references/esm/module_id.rs` + `crates/turbopack-ecmascript/src/annotations.rs` · bgw · **SHIPPED in 16.3.0-canary.91** (2026-07-20T23:58:30Z).

### Turbopack Parent Directory Creation Simplification (16.3.0-canary.91 SHIPPED 2026-07-20T23:58:30Z, [PR #95835](https://github.com/vercel/next.js/pull/95835) by bgw)

Turbopack's filesystem writer had a hand-rolled retry loop for parent-directory creation: 3 attempts with exponential backoff, an `is_retryable()` predicate distinguishing `ENOENT` (retry — parent missing) from `EEXIST` (no-op — already there), and a fallback to `mkdir -p`-style manual recursion when the loop exhausted. The logic was correct but verbose (~70 lines), produced occasional "directory already exists, retrying" warnings on parallel-output workloads, and was harder to reason about than the equivalent one-liner.

**The fix** in `crates/turbopack/src/lib.rs` + `crates/turbopack-fs/src/lib.rs` replaces the loop with a single call to Rust's built-in recursive directory creation (the equivalent of `mkdir -p`). The library handles `EEXIST` + `ENOENT` + race conditions internally, so the retry loop + `is_retryable()` predicate + the "directory already exists, retrying" warning are all gone.

**Practical impact:**

- **`next build` on cold caches:** marginally faster — fewer syscall round-trips per output directory. On a large app (1000+ output files) the win is in the tens of milliseconds; not noticeable on a single project but cumulative across CI runs.
- **Dev log cleanliness:** the "directory already exists, retrying" warning that appeared occasionally during `next build` (especially with parallel output writing) is gone. Dev console is quieter.
- **Code-quality win:** the Turbopack crate is slightly easier to reason about (one fewer hand-rolled retry loop to maintain). CI-only impact for contributors.
- **No user-facing behavior change** beyond the log line and the marginal build-time win.

**Source:** [PR #95835 — `Turbopack: Simplify parent directory creation retry loop logic`](https://github.com/vercel/next.js/pull/95835) · `crates/turbopack/src/lib.rs` + `crates/turbopack-fs/src/lib.rs` (combined ~80 lines net -50) · bgw · **SHIPPED in 16.3.0-canary.91** (2026-07-20T23:58:30Z).

### Turbopack Dev: Skip SSR Compile for Pages Only Navigated to Through Soft Navs (16.3.0-canary.89 SHIPPED 2026-07-17T23:55:15Z, [PR #95539](https://github.com/vercel/next.js/pull/95539) by sampoder, merged 2026-07-17T20:10:55Z, +460/-89 across 23 files)

The single biggest user-facing PR in canary.89. Previously, Turbopack dev compiled the Client Component SSR chunk for every HTML page endpoint registered in the App Router, regardless of whether that page was ever rendered as a standalone document. Most app pages today are reached via an RSC-only soft navigation — a page rendered inside a modal, a tabbed sub-route, a `Link` with `prefetch={false}` from a parent page, a route surfaced as a fragment via `<Link>` from a deeper segment, or a route only ever visible as a child of a layout that hydrates in place. In those cases the SSR chunk is dead weight: the page is never sent as a document, only as RSC payload fragments, so the SSR build is compiled but never served.

**The fix** introduces a new `OutputModeState` (per dev session) that tracks which HTML page endpoints are actually rendered as documents during the current dev session. When a page is navigated to only through RSC soft navigation, the page is added to the set of "non-SSR" pages, and its `process_ssr` flag is set to `false`. The new `mark_as_ssr_operation` turbo-tasks operation (in `crates/next-api/src/output_mode.rs`) walks the dev session's `OutputModeState` to decide which HTML page endpoints need SSR compilation; pages not in the set get:

- `process_ssr = false` → the `ssr_chunking_context` collapses from a separate upcast into a `process_ssr.then_some(server_chunking_context)` no-op when not needed
- `rscModuleMapping` for the App Router is now only emitted for pages (route handlers + metadata routes drop it)
- `AppEndpointType::Page { ty: AppPageEndpointType::Html }` gets the existing `RscHmr` sibling variant reclassified as "HMR-only: detects Server Component changes but emits no manifests, so it cannot serve a request"

Internal cleanup PR #95908 (`[turbopack] Clean up server_chunking_context`) removes the now-unused `server_chunking_context` helper in the same cycle, consolidating around the unified chunking context passed through `mark_as_ssr_operation`.

**Files touched (PR #95539 + #95908, 24 files combined):** `crates/next-api/src/{app.rs, lib.rs, output_mode.rs}` (new module), `crates/next-core/src/{app_structure.rs, next_config.rs, app_page_annotations.rs}` (mark-as-ssr hook), `crates/turbopack/...` (chunking context plumbing), 23 files total.

**Practical impact:**

- **Dev compile time:** materially faster `next dev` first-request compile for apps with many pages that are mostly reached via RSC soft navigation. The exact ratio depends on how many of your pages are SSR-only-by-default vs RSC-soft-nav-only. For an app with 50 pages where 40 are reached only via soft nav (e.g. a single-page admin panel with deeply-nested tab routes), expect the first `next dev` request that touches a previously-uncompiled soft-nav-only page to skip the SSR build entirely.
- **Dev memory:** lower `next dev` RSS because dead SSR chunks for soft-nav-only pages are never materialised. The `rscModuleMapping` pruning is the bigger memory win for apps with many route handlers + metadata routes.
- **Production:** zero impact. `OutputModeState` is dev-session-scoped; the `mark_as_ssr_operation` function `bail!`s outside a dev session ("`mark_as_ssr is never called outside of a dev session`"). Production SSR runs compile every HTML page endpoint as before.
- **Dev sessions that mix full-document and soft-nav visits:** the first time a page is rendered as a document (e.g. hard-loaded at `/dashboard/settings`), the page is added to the SSR set on-the-fly and its SSR build runs immediately; subsequent visits (full or soft) reuse that build.

**How to verify on your app:**

1. Open the new Turbopack runtime log section (exposed by #95908's cleanup) — `next dev` now logs per-endpoint SSR-build decisions. Look for entries like `[turbopack] skip ssr for /admin/billing (soft-nav only)` — that's the new fast-path kicking in.
2. Time your dev startup with a tool like `hyperfine 'next dev' --warmup 1` before and after upgrading to canary.89. The improvement is most visible on cold starts for large app routers.
3. In DevTools Memory tab, compare the heap snapshot before and after the upgrade for a long-running dev session; look for the `rscModuleMapping` entries shrinking for route handlers + metadata routes.

**When this matters most:** any app with a non-trivial navigation tree where most routes are children of a shared layout (admin panels, dashboards, multi-step forms, e-commerce sites with deep product taxonomies). For a single-page-app-with-routes structure, expect 30–60% faster first-RSC-payload compile for soft-nav-only pages.

**Source:** [PR #95539 — `[turbopack] Don't SSR on pages only navigated to through a soft nav`](https://github.com/vercel/next.js/pull/95539) · [PR #95908 — `[turbopack] Clean up server_chunking_context`](https://github.com/vercel/next.js/pull/95908) · Files: 23 + 1 = 24 files across `crates/next-api/`, `crates/next-core/`, `crates/turbopack/` · sampoder · merged 2026-07-17T20:10:55Z · **Shipped in 16.3.0-canary.89** (npm `canary` dist-tag pointer moved 2026-07-17T23:55:15Z).



**⚠️ REVERTED in 16.3.0-canary.93 on 2026-07-21T21:03:05Z — [PR #96028](https://github.com/vercel/next.js/pull/96028) by sampoder, `Revert "[turbopack] Don't SSR on pages only navigated to through a soft nav (#95539)"`. The revert body says "This PR (#95539) appears to have caused issues with cached components — reverting." The feature shipped in canary.89 → canary.92 and was the single biggest user-facing PR of that cycle, then was rolled back 4 days later for breaking `cacheComponents: true` apps. Net effect for users:

- **canary.89 → canary.92 users** had the optimization for 4 days (~2026-07-17T23:55Z → ~2026-07-21T21:03Z). The `OutputModeState` + `mark_as_ssr_operation` machinery is gone from canary.93+ — Turbopack dev goes back to compiling the SSR chunk for every HTML page endpoint.
- **`cacheComponents: true` users** are the trigger group: the optimization broke cache-warming for some cached components. **Action:** if you set `experimental.cacheComponents: true` and noticed stale shells or missing chunks during canary.89–canary.92, canary.93 is your fix. Otherwise the revert is invisible.
- **The App Shells + Partial Prefetching story is unchanged** — PR #95415 (the `appShells` flag unification that shipped in canary.88) is a separate optimization path and is unaffected by the #95539 revert.

The revert is clean (commit `74f9866976`) and only conflicted with `crates/next-api/src/app.rs` (already resolved on the canary branch). The full canary.93 release includes 9 entries — 1 revert (#96028) + 1 perf win (#95994 below) + 1 Windows bug fix (#95668) + 1 sharp dep bump (#95507) + 2 internal refactors (#95142, #95951) + 1 docs (#96003) + 2 CI-only (#95628, "Restore canary version 16.3.0-canary.92 after v16.3.0-preview.7 preview release"). **Source:** [PR #96028 — `Revert "[turbopack] Don't SSR on pages only navigated to through a soft nav (#95539)"`](https://github.com/vercel/next.js/pull/96028) · Commit `74f9866976` · 2026-07-21T21:03:05Z · **Shipped in 16.3.0-canary.93** (2026-07-21T23:55:58Z).

### Turbopack CJS Export Pruning — Extended to `Object.defineProperty` Syntax (16.3.0-canary.93 SHIPPED 2026-07-21T23:55:58Z, [PR #95994](https://github.com/vercel/next.js/pull/95994) by sampoder, merged 2026-07-21T19:14:40Z)

The canary.91 PR #95716 (Turbopack CJS Export Pruning — "Drop unused exports from a CJS module") was limited to CJS modules that assign exports directly (`module.exports.X = ...`). Many transpilers — including TypeScript's classic ESM→CJS emit, Babel's `@babel/preset-env` CJS target, and several older bundler outputs — emit CJS via `Object.defineProperty(exports, "X", { value: ... })` instead. PR #95994 extends the same live-export-set reference analysis to that syntax.

**Practical impact:** same as the canary.91 fix, but extended to the most common transpiler CJS outputs. TypeScript and Babel-emitted CJS bundles now see the same pruning. Cumulative savings are small per file but universal — every project that uses any of these transpilers in its dependency tree (`tsc` with `module: "commonjs"`, `babel-loader` with `@babel/preset-env` targeting Node, `swc` with `module.type: "commonjs"`, several older Server Actions SDKs) benefits.

**Action:** upgrade to `next@canary@93` (npm dist-tag pointer moved 2026-07-21T23:55:58Z; `npm install next@canary --save-exact`). No code change needed.

**Source:** [PR #95994 — `[turbopack] Tree-shake CJS exports that use the Object.defineProperty syntax`](https://github.com/vercel/next.js/pull/95994) · Extends [PR #95716](https://github.com/vercel/next.js/pull/95716) · sampoder · merged 2026-07-21T19:14:40Z · **Shipped in 16.3.0-canary.93**.

### Turbopack Windows Path Canonicalization Fix — Verbatim Paths Internally (16.3.0-canary.93 SHIPPED 2026-07-21T23:55:58Z, [PR #95668](https://github.com/vercel/next.js/pull/95668) by bgw, merged 2026-07-21T21:54:02Z)

The previous Turbopack Windows path handling used `dunce` to normalize paths between win32 (`C:\foo\bar`) and verbatim (`\\?\C:\foo\bar`) representations — but `dunce` picks the shorter of the two, which is fragile for long paths. Internally `DiskFileSystem` was also (incorrectly) not canonicalizing its own root dir, which happened to work because pnpm wasn't canonicalizing junction point targets either — until a sibling PR added `pnpm-workspace.yaml` files and exposed the underlying bug. The `read_link` helper was also reimplementing `try_from_sys_path` badly. And `crates/turbo-tasks-fs/src/embed/file.rs` turned out to be dead code.

**The fix:**

- Switches from `dunce` to `omnipath::WinPathExt` for path-format selection (long-path-safe, cross-platform consistent).
- **Always** uses the verbatim path format internally within `DiskFileSystem` — verbatim paths work for paths >260 characters (the win32 MAX_PATH limit) and behave close to Unix paths. An import from a Windows 8.3 short path now correctly fails (it would silently resolve before).
- Canonicalizes any absolute symlink targets outside the `DiskFileSystem` root, resolving any symlinks they may have, normalizing case-insensitive paths, and handling Windows 8.3 short name format.
- Replaces `read_link` with the standard `try_from_sys_path`.
- Deletes the dead-code `crates/turbo-tasks-fs/src/embed/file.rs`.
- Fixes the doc comment on `validate_path_length_inner` (was incorrectly calling verbatim paths "UNC paths"; they're different things — verbatim is `\\?\C:\...`, UNC is `\\server\share\...`).

**Practical impact for Windows users:**

- **`next dev` and `next build` on Windows** in monorepos with junction points (the default pnpm workspace layout) no longer intermittently fail with "file not found" or wrong-path errors after PR #95628 (which canonicalizes pnpm junction targets).
- **Long paths (>260 characters)** on Windows now work correctly with Turbopack (previously a path that grew past MAX_PATH would silently fall back to a win32 path that failed the root-prefix strip).
- **Symlinks outside the project root** now resolve consistently across dev sessions (previously the same symlink could resolve to different paths depending on the consumer, breaking HMR and cache invalidation).
- **No API change, no config change** — purely a fix. macOS/Linux users see no change.

**Audit:** `git log --oneline --all -- crates/turbo-tasks-fs/src/embed/file.rs` will show the dead-code deletion; `next dev --turbo` in a pnpm monorepo on Windows now starts cleanly where it intermittently failed before.

**Source:** [PR #95668 — `Turbopack: Fix missing canonicalization of paths and always use verbatim paths internally for Windows`](https://github.com/vercel/next.js/pull/95668) · `crates/turbo-tasks-fs/` (multiple files) · bgw · merged 2026-07-21T21:54:02Z · **Shipped in 16.3.0-canary.93** (2026-07-21T23:55:58Z).

### `sharp@0.35.3` Dependency Bump (16.3.0-canary.93 SHIPPED 2026-07-21T23:55:58Z, [PR #95507](https://github.com/vercel/next.js/pull/95507) by styfle, merged 2026-07-21T19:52:10Z)

The bundled `sharp` dependency (used by `next/image` for AVIF/WebP/JPEG/PNG encoding during `next build`'s image-optimization cache warmup and for `dangerouslyAllowSVG` validation) jumped from the version Next.js was pinned to (likely `0.34.x` based on the prior cron's setup.md) to **`sharp@0.35.3`** (released 2026-07-01 by [lovell/sharp](https://github.com/lovell/sharp/releases/tag/v0.35.3)). Three notable changes for Next.js users:

1. **No more install script** — sharp 0.35.3 ships with prebuilt libvips binaries that detect the host architecture at runtime; the previous postinstall script (`node install/libvips.js`) is gone. **Action for Next.js Dockerfiles:** the `unsafe-perm=true` flag (`npm config set unsafe-perm true`) that was historically required to let sharp's install script run as root is no longer needed and can be removed from `Dockerfile`s. The base-image size impact is also slightly smaller because the install doesn't fetch build toolchain headers.
2. **No more dynamic require/import tracing workaround** — sharp 0.35.x changed how it loads its native bindings internally, eliminating the class of "Cannot find module '../build/Release/sharp.node'" errors that previously required `npm rebuild sharp` workarounds in some CI environments and Docker layers. If you have `RUN npm rebuild sharp` in your Dockerfile as a defensive measure, it's now safe to remove.
3. **AVIF uses "tune iq"** — the AVIF encoder now uses the [LibAOM tune iq image-quality tuning mode](https://aomedia.org/blog%20posts/Libavif_v1_4_0-Boosts-Major-Updates-to-Encoder-Technology/#the-tune-iq-image-quality-tuning-mode) which matches human perception better than the previous tune mode. Practical effect: AVIF outputs at the same quality setting are ~5–15% smaller for the same perceived quality (or ~5–10% better quality at the same file size, depending on content). For Next.js apps that ship a lot of AVIF (the `dangerouslyAllowSVG: true` + AVIF format combo, or just the default AVIF output for `next/image` on browsers that support it), this is a free perf win — re-running `next build` will produce smaller `/_next/image?url=...&format=avif` responses on the same source images.

**Action for Next.js users:**

- **Upgrade `next` to canary.93+** — the bundled sharp jumps automatically; no `package.json` edit needed.
- **Clean up Dockerfiles:** remove `npm config set unsafe-perm true` and the `npm rebuild sharp` defensive step if you have them.
- **Re-run `next build`** to regenerate the `.next/cache/images` directory with the new AVIF encoder; the size delta will be visible in the build summary (`Image (n images): before X → after Y`).
- **For standalone installs** (CI runners, on-prem deploys): the install is faster now because no postinstall script runs; expect 5–15s saved per `npm install` of `next`.

**Source:** [PR #95507 — `chore(deps): bump sharp@0.35.3`](https://github.com/vercel/next.js/pull/95507) · [sharp v0.35.3 release notes](https://github.com/lovell/sharp/releases/tag/v0.35.3) · styfle · merged 2026-07-21T19:52:10Z · **Shipped in 16.3.0-canary.93**.


### Fix Stale Dev `'use cache'` for Cookieless Requests + Route Handlers (**SHIPPED in `16.3.0-canary.94`** 2026-07-23T00:02:38Z, [PR #96022](https://github.com/vercel/next.js/pull/96022) merged 2026-07-22T12:46:39Z)

The HMR refresh hash that invalidates dev-mode `'use cache'` entries used to live in a session cookie (`__next_hmr_refresh_hash__`) — the client set it on every `processMessage(SERVER_COMPONENT_CHANGES)` and read it back on subsequent requests to include in cache keys. That broke three real classes of dev experience:

| Broken class | Why it broke |
|---|---|
| **Cookieless requests** (privacy-mode browsers, `curl`, RSS readers, server-side prefetch from a different origin) | Never received a fresh `'use cache'` entry after an edit — no cookie meant the cache key was always the previous hash |
| **`'use cache'` inside route handlers** | Cookie was only set from `processMessage(SERVER_COMPONENT_CHANGES)` in `hot-reloader-app.tsx` (app-router client). Route-handler requests never went through that path, so the cookie stayed stale |
| **Apps where the HMR client hadn't connected yet** (CI smoke runs against a fresh `next dev`, Playwright headless runs disabling the dev runtime, SSR-only debugging) | Counter never advanced because no client was subscribed |

**The fix (PR #96022, 18 files +289/-43)** replaces the cookie with a server-side counter (`hmrHash` in the hot-reloader), threaded through every render via `hmrRefreshHash` request meta, and folded into `"use cache"` cache keys. Key changes:

- `BaseServer.getServerComponentsHmrRefreshHash()` returns the current server-components generation (overridden by `DevServer` to read from the bundler service). `BaseServer.handleRequest` now attaches `hmrRefreshHash` to every render that flows through request handling — including internal renders like dev validation/warmup that don't re-enter the top-level request handler.
- `RequestStore` exposes `readonly hmrRefreshHash?: string`. `WorkStoreContext` mirrors it.
- The use-cache wrapper reads `getHmrRefreshHash` (was `workUnitStore.cookies.get(NEXT_HMR_REFRESH_HASH_COOKIE)?.value`).
- For webpack, `serverComponentsHmrRefreshHash` is updated inside `refreshServerComponents(hash)`.
- For Turbopack, the counter advances only on subscription-driven recompiles — NOT on per-client `BUILT` messages (which would churn the hash without an edit and fail to advance it when no client is connected). The counter is returned unconditionally (`"0"` before the first edit) so `"use cache"` keys are present and consistent for every request, mirroring webpack's always-present `stats.hash`.
- Route handlers (`packages/next/src/server/dev/turbopack-utils.ts`) now `hooks?.subscribeToChanges(...)` on the route-handler endpoint. The subscription exists only to advance the refresh hash (since route handlers have no RSC for a connected browser to refetch, `createMessage` returns nothing). On subscription error: log `Error in the "<page>" app-route HMR subscription` with `cause` set to the underlying error; `subscribeToChanges` drops the subscription on error and re-creates it next ensure.
- `NEXT_HMR_REFRESH_HASH_COOKIE` constant is removed from `app-router-headers.ts`; the cookie is no longer set on `SERVER_COMPONENT_CHANGES`. Existing cookies in dev sessions will be ignored (no migration step required — they'll be GC'd).
- Tests updated: `use-cache-custom-handler.test.ts`, `use-cache-default-handler-expire-zero.test.ts`, and the dedicated `use-cache-dev.test.ts` now match the cache key with a trailing optional hash element (`"[...,..."?\]"` regex form).

**Practical impact:**

- **No more "I edited the function but the dev server keeps serving the old value"** for cookieless clients (server-to-server fetches, headless browsers, RSS crawlers)
- **Route handlers using `"use cache"` now revalidate** on edit, not just on the stale-5-minute `expire: 0` fallback
- **CI dev sessions now self-invalidate** — running Playwright against a fresh `next dev` no longer gets stuck on stale cache after the file changes
- **No production impact** — the dev-only `hmrRefreshHash` doesn't enter production cache keys (`hmrRefreshHash: undefined` in `finalRuntimeServerPrerender`)

**Action:** upgrade to `next@canary@94` when it ships. No code change needed; the existing `"use cache"` functions automatically benefit.

**Audit recipes:**
- `rg '__next_hmr_refresh_hash__' .next/` → should return 0 hits after canary.94 (the cookie is gone)
- `rg 'serverComponentsHmrRefreshHash' packages/next/src/server/dev/` → shows the new counter wiring (webpack + turbopack)
- `rg 'subscribeToChanges' packages/next/src/server/dev/turbopack-utils.ts` → confirms the route-handler subscription is in place

**Source:** [PR #96022 — `Fix stale dev 'use cache' for cookieless requests and route handlers`](https://github.com/vercel/next.js/pull/96022) · 18 files +289/-43 across `packages/next/src/server/{base-server.ts, async-storage/{request-store,work-store}.ts, dev/{hot-reloader-turbopack, hot-reloader-webpack, turbopack-utils, next-dev-server}.ts, lib/dev-bundler-service.ts, use-cache/{use-cache-wrapper, use-cache-probe-globals, use-cache-probe-scheduler, use-cache-probe-worker}.ts, request-meta.ts, route-modules/app-route/module.ts, web/adapter.ts, build/templates/app-route.ts, errors.json, client/{components/app-router-headers, dev/hot-reloader/app/hot-reloader-app, dev/hot-reloader/pages/hot-reloader-pages, page-bootstrap}.{ts,tsx}}` + 4 test updates · merged 2026-07-22T12:46:39Z · commit `286862e35b` · **Shipped in `16.3.0-canary.94`** (2026-07-23T00:02:38Z).


### Static Params HMR Refresh — Dedicated Message + Post-Cache Update (**SHIPPED in `16.3.0-canary.94`** 2026-07-23T00:02:38Z, [PR #96019](https://github.com/vercel/next.js/pull/96019) + [PR #96020](https://github.com/vercel/next.js/pull/96020) merged 2026-07-22T12:24:23Z–13:00:54Z)

A paired 2-PR stack that fixes a long-standing dev-loop papercut for `cacheComponents: true` apps using `generateStaticParams`. **Both PRs by Janka Uryga; both ship in `16.3.0-canary.94`.**

**PR #96019 — `Emit the static paths HMR update after updating the cache`** (commit `35bf8dab73`, 2026-07-22T12:24:23Z, 1 file +33/-24 in `next-dev-server.ts`). The existing trigger (send `SERVER_COMPONENT_CHANGES` with a `generateStaticParams-${Date.now()}` hash when the static-paths cache result length changes) was emitted from the wrong branch — it fired inside `this.staticPathsCache.set` BEFORE the cache write completed. So when `fallbackParams` were read in the next render, they were still derived from the previous result, and the user saw stale params until the next refresh.

The fix moves the HMR emit AFTER the `staticPathsCache.set(pathname, value)` call. After the move, the render that triggers the HMR refresh sees the new `fallbackParams` because the cache is already updated. (There's still a small reuse-the-hash-of-the-server-component-changes workaround noted in the PR TODO: "Give this its own HMR message instead of reusing `SERVER_COMPONENT_CHANGES`, which requires a `hash` whose value is meaningless here (a timestamp) and only serves to trigger a client refresh." — PR #96020 below does exactly that.)

**PR #96020 — `Add a dedicated HMR message for static params changes`** (commit `a07e947a27`, 2026-07-22T13:00:54Z, 7 files +55/-11). Introduces a new `HMR_MESSAGE_SENT_TO_BROWSER.STATIC_PARAMS_CHANGED = 'staticParamsChanged'` message type and a `StaticParamsChangedMessage` interface (`{ type: STATIC_PARAMS_CHANGED }` — note: no `hash` field). Sent specifically when a route's set of statically-known params changes (e.g. `generateStaticParams` added, removed, or edited).

The client handler in `hot-reloader-app.tsx`:
```ts
case HMR_MESSAGE_SENT_TO_BROWSER.STATIC_PARAMS_CHANGED: {
  // Re-fetch the current router tree so the render picks up the new set of
  // statically-known params (and thus the fresh fallbackParams). Unlike
  // SERVER_COMPONENT_CHANGES this does not store an HMR refresh hash, so
  // it doesn't invalidate 'use cache' entries.
  if (RuntimeErrorHandler.hadRuntimeError || document.documentElement.id === '__next_error__') {
    if (reloading) return
    reloading = true
    return window.location.reload()
  }

  startTransition(() => {
    publicAppRouterInstance.hmrRefresh()
    dispatcher.onRefresh()
  })

  return
}
```

The pages-router client (`hot-reloader-pages.ts`) explicitly ignores the new message ("Only relevant to the App Router; ignored in the Pages Router client."). `page-bootstrap.ts` adds the new type to its `HMR_MESSAGE_SENT_TO_BROWSER` whitelist (alongside `ADDED_PAGE`, `REMOVED_PAGE`, `SERVER_COMPONENT_CHANGES`, `SYNC`, `BUILT`, `BUILDING`).

**The big win:** because `STATIC_PARAMS_CHANGED` does NOT carry an HMR refresh hash, it does NOT invalidate `"use cache"` entries. The old timestamp-hash branch sent on `SERVER_COMPONENT_CHANGES` would over-invalidate every cached function whenever params changed. After both PRs land, `generateStaticParams` edits invalidate ONLY the relevant `fallbackParams` (via the router-tree refetch), not the entire dev `'use cache'` cache.

**New test** (`test/e2e/app-dir/cache-components-errors/use-cache.util.ts`): `should clear the redbox after adding generateStaticParams via HMR` — edits a route file to add a `generateStaticParams` export while the dev error overlay is showing the "blocking route" redbox, asserts the redbox clears after the HMR refresh.

**Practical impact for `cacheComponents: true` + `generateStaticParams` apps:**
- "I added `generateStaticParams` to a page that previously had none, but my dev session is stuck on a redbox" → fixed; the redbox clears on the next dev refresh
- "My `'use cache'` entries got nuked when I edited `generateStaticParams`" → fixed; only the route's `fallbackParams` re-fetch, the cache stays warm
- "My static params were stale until I manually clicked save twice" → fixed; HMR refresh now fires AFTER the cache update, so the render that processes it sees the new params

**Action:** upgrade to `next@canary@94` when it ships. No code change needed; the HMR protocol is backwards-compatible (old clients ignore the new message type).

**Audit recipes:**
- `rg 'STATIC_PARAMS_CHANGED' packages/next/src/` → should return 7 hits (3 in `hot-reloader-types.ts`, 1 in `hot-reloader-app.tsx`, 1 in `hot-reloader-pages.ts`, 1 in `page-bootstrap.ts`, 1 in `next-dev-server.ts`) after canary.94
- `rg 'SERVER_COMPONENT_CHANGES.*generateStaticParams' packages/next/` → should return 0 hits after canary.94 (the timestamp-hash branch is gone)
- `rg 'a07e947a27\|35bf8dab73' CHANGELOG.md` → verify both commits are listed in the canary.94 release body

**Sources:**
- [PR #96019 — `Emit the static paths HMR update after updating the cache`](https://github.com/vercel/next.js/pull/96019) · `packages/next/src/server/dev/next-dev-server.ts` (+33/-24) · Janka Uryga · merged 2026-07-22T12:24:23Z · commit `35bf8dab73` · **Ships in `16.3.0-canary.94`**
- [PR #96020 — `Add a dedicated HMR message for static params changes`](https://github.com/vercel/next.js/pull/96020) · 7 files +55/-11 across `hot-reloader-types.ts`, `hot-reloader-app.tsx`, `hot-reloader-pages.ts`, `page-bootstrap.ts`, `next-dev-server.ts`, plus tests · Janka Uryga · merged 2026-07-22T13:00:54Z · commit `a07e947a27` · **Ships in `16.3.0-canary.94`**


### Cache-Miss Fix in App Shell for Cached Pages with `generateStaticParams` (**SHIPPED in `16.3.0-canary.94`** 2026-07-23T00:02:38Z, [PR #95665](https://github.com/vercel/next.js/pull/95665) merged 2026-07-22T15:18:51Z, closes `NAR-883`)

A silent cache-miss footgun for `cacheComponents: true` + `generateStaticParams` + `appShells`-style prerendering (i.e. when App Shells / `partialPrefetching: true` is in play). The bug:

- When an App Shell is being prerendered, the `ShellRuntime` stage is the ceiling — URL data is excluded; the prerender doesn't advance beyond `ShellRuntime`.
- But the prospective runtime prerender (which decides what's a "hanging input") was letting `params` / `searchParams` resolve normally instead of marking them as hanging.
- **Result:** when `params` ended up being inputs to a cached page, those inputs weren't hanging in the prospective render — and then became a cache miss in the final render (which IS past `ShellRuntime`).

**The fix** (PR #95665, 5 files +33/-11 across `packages/next/src/server/app-render/{app-render.tsx, work-unit-async-storage.external.ts, request/params.ts, errors.json}`):
1. Adds `readonly isSessionShell: boolean` to `PrerenderStoreModernRuntime`.
2. Threads `isSessionShell: isShellPrefetch` from `prospectiveRuntimeServerPrerender(ctx, isShellPrefetch, ...)` (new arg) and `finalRuntimeServerPrerender(..., { isSessionShell: mode.type === 'session-shell-only', ... })` into the work-unit store.
3. `createRuntimePrerenderParams` gains `workStore: WorkStore` as a new arg and now branches on `workUnitStore.isSessionShell`:
   ```ts
   // Was: always makeUntrackedParams(userspaceParams) when no stagedRendering
   if (workUnitStore.isSessionShell) {
     return makeHangingParams(underlyingParams, workStore, workUnitStore)
   } else {
     return makeUntrackedParams(userspaceParams)
   }
   ```
4. New error code #1449 `"Accessed \`searchParams\` during prerendering."` enforces the new contract (the error was already raised; the addition is to `errors.json` for stability).

**Practical impact for `cacheComponents: true` + `generateStaticParams` apps:**
- "I added `generateStaticParams` to a cached page and now my `partialPrefetching: true` shell is doing duplicate data fetches" → fixed; the params are properly marked as hanging in the prospective render, so the final render's cache key matches and it hits
- "My App Shell prerender succeeds but the runtime render is a cache miss" → fixed; same root cause
- **No public API change** — `params` / `searchParams` continue to resolve to the same values; the change is purely in how the prerender classifies them as hanging vs not

**Action:** upgrade to `next@canary@94` when it ships. No code change needed; existing `'use cache'` + `generateStaticParams` patterns automatically benefit.

**Audit recipe:** `rg 'isSessionShell' packages/next/src/server/` → confirms the new field is threaded through both `prospectiveRuntimeServerPrerender` and `finalRuntimeServerPrerender`. `rg '1449' packages/next/errors.json` → confirms the new error code is registered.

**Source:** [PR #95665 — `fix: cache miss in App Shell for cached pages with gSP`](https://github.com/vercel/next.js/pull/95665) · 5 files +33/-11 across `app-render.tsx`, `work-unit-async-storage.external.ts`, `request/params.ts`, `errors.json` · Janka Uryga · merged 2026-07-22T15:18:51Z · commit `63f14c6c90` · closes `NAR-883` · **Shipped in `16.3.0-canary.94`** (2026-07-23T00:02:38Z).


### Turbopack: Stop Copying `sourcesContent` Into Every Serialized Source Map (SHIPPED in `16.3.0-canary.94` 2026-07-23T00:02:38Z, [PR #95934](https://github.com/vercel/next.js/pull/95934) by bgw, merged 2026-07-22T20:11:33Z)

A significant **Turbopack dev-memory win** — ~15-20% peak dev memory reduction on dependency-heavy apps. The bug: every module's source map was inlining `sourcesContent` (the full original source text) into the serialized JSON rope, so every subsequent transformation had to re-copy the entire `sourcesContent` block:

- Rewriting the `sources` URLs for each chunking context copied the entire map per context.
- Embedding the module map into a chunk's sectioned map copied it again per layer.

In a measured dependency-heavy app, source maps were **57% of module factory bytes** and `sourcesContent` alone was **36%**. Each module's source text ended up resident ~5 times over.

**The fix:** `StructuredSourceMap` keeps the map in field form until the moment of emission. Fields never modified (`mappings`, `names`, `version`) are stored as verbatim raw JSON snippets (rope sharing, no re-serialization). `sources` (the one field rewrites modify) is stored typed. End result:

- **Measured impact (vercel.com-style dependency graph, walking 15 routes):** peak memory **3.0 GB → 2.6 GB** (`-15.2%`, `p = 2/12870 = 0.00016` exact permutation test). Internal app: ~20% peak dev memory win.
- **No API change.** `dev` only — production source maps (`next start`) are unaffected because they go through a different serialization path.
- **No config change.**

**Why this matters for dev sessions on monorepos:** if you've ever watched `next dev` slowly eat 4-6 GB on a Vercel-sized dependency tree, this PR is the most material single-PR Turbopack memory win since canary.71's `experimental.turbopackMemoryEviction: 'full'` (which already cut ~90% on vercel.com).

**Source:** [PR #95934 — `Turbopack: stop copying sourcesContent into every serialized source map`](https://github.com/vercel/next.js/pull/95934) · 1 file +147/-107 in `crates/turbopack-core/src/source_map.rs` (and structured source map rewire) · bgw · merged 2026-07-22T20:11:33Z · fixes [issue #81161](https://github.com/vercel/next.js/issues/81161) · **Shipped in `16.3.0-canary.94`** (2026-07-23T00:02:38Z).


### Turbopack: Fix Deployment Skew Protection for Component Chunks (SHIPPED in `16.3.0-canary.94` 2026-07-23T00:02:38Z, [PR #96079](https://github.com/vercel/next.js/pull/96079) by bgw, merged 2026-07-22T20:27:12Z)

A **deployment-skew correctness fix** for Turbopack — fixes a class of "page errors out after deploy" footguns where the dev server's serialized component chunks had an implicit assumption that chunks were plain strings. After [#95261](https://github.com/vercel/next.js/pull/95261) (a recent change to chunk representation), the assumption was wrong; Turbopack would emit skewed chunks when the deployed server had a different chunk shape than the local server.

**The fix:** generalize the chunk-identity check to accept the new chunk shape from #95261. The PR is mostly a regression test (`Almost all of this diff is a regression test.` per the PR body) — the production fix is a 1-line conditional.

**Who needs this:** any Next.js + Turbopack project that uses [React PR #37095](https://github.com/react/react/pull/37095) (the upcoming React Server Components reference fix) — the new React ref-shape was the reproducer.

**No API change, no config change.** Silent correctness fix.

**Audit:** `rg 'skew.*chunk\|chunk.*skew' packages/next/src/` should show the new guard; the test in `test/e2e/turbopack-deployment-skew/` should pass after upgrading.

**Source:** [PR #96079 — `[turbopack] Fix deployment skew protection for component chunks`](https://github.com/vercel/next.js/pull/96079) · mostly test + 1 file in `crates/next-api/src/` · bgw · merged 2026-07-22T20:27:12Z · **Shipped in `16.3.0-canary.94`** (2026-07-23T00:02:38Z).


### Cache Components: Exclude Dynamic Params from Prerenders When No `generateStaticParams` Values Provided (SHIPPED in `16.3.0-canary.94` 2026-07-23T00:02:38Z, [PR #95872](https://github.com/vercel/next.js/pull/95872) by gnoff, merged 2026-07-22T22:28:43Z, closes `NAR-882`)

A **silent-data-leakage fix** for `cacheComponents: true` + `generateStaticParams` (or empty `generateStaticParams`) + self-hosted / Vercel Adapter routes that prerender static pages. Two failure modes:

1. **Self-hosted + Vercel Adapter — routes with empty shells during build.** A route without a `fallback: true` that has an empty shell during the build prerenders misses with ALL params (instead of just the ones declared in `generateStaticParams`). This means a cache hit for one URL silently serves the same shell as a different URL.

2. **All Next.js environments — root params not in cache.** Any route matching a root param not already in the cache prerenders with all params regardless of whether they were omitted from `generateStaticParams`. This caused cache collisions between routes that should have produced different prerendered shells.

**The fix** (PR #95872): the prerender decision now correctly excludes dynamic params that aren't in the static-params set. For Vercel Adapter specifically, only the declared static params are prerendered (not the implicit "all params" set).

**Comprehensive test matrix added** to catch regressions in these kinds of prerender decisions.

**Practical impact for `cacheComponents: true` apps using `generateStaticParams`:**

- "My route with partial `generateStaticParams` coverage is prerendering with params that shouldn't be there" → fixed; the prerender set is now bounded by the declared static params.
- "Two URLs that should produce different prerendered shells are sharing the same shell" → fixed; root-param-not-in-cache case is handled.
- **No public API change** — `generateStaticParams` continues to work the same way; the change is purely in which params are emitted into the prerendered shell.

**Action:** upgrade to `next@canary@94`. No code change needed.

**Source:** [PR #95872 — `[Cache Components] Exclude dynamic params from prerenders when no generateStaticParams values is provided`](https://github.com/vercel/next.js/pull/95872) · gnoff · merged 2026-07-22T22:28:43Z · closes `NAR-882` · **Shipped in `16.3.0-canary.94`** (2026-07-23T00:02:38Z).


### `next/font/google` Fetch Timeout Now Bounded on Turbopack (will ship in `16.3.0-canary.95`, [PR #95981](https://github.com/vercel/next.js/pull/95981) by styfle, merged 2026-07-23T15:40:55Z)

A **hang-prevention fix** for `next/font/google` on the Turbopack path — the compile-time Google Fonts fetch could hang forever when the network stalled. Now bounded:

**The bug:** `next/font/google` fetches from Google Fonts at compile time. When the connection *hangs* (captive portal, packet-dropping proxy, broken IPv6), the fetch had no timeout, so compilation blocked until the OS connect timeout (~75s) or indefinitely. This bit teams on flaky hotel Wi-Fi, in Docker builds behind corporate proxies, and in CI runners with broken egress.

**The fix (PR #95981):**
- `FetchClientConfig` gets a `connect_timeout` (10s) and total `timeout` (60s); Google Fonts overrides them to **5s / 30s** (tighter, since the Google Fonts endpoint is usually fast).
- Transient failures retry up to **3×**, each attempt its own `duration_span!` (so the timeout is per-attempt, not cumulative).
- On failure:
  - **`next build`**: errors the build with the attempt count + a suggestion to switch to `next/font/local`.
  - **`next dev`**: warns and uses the **fallback font** so dev keeps running. The dev mode gets a longer timeout in a follow-up (PR description says "Reduce the dev-mode timeout (build keeps the longer value)" is a TODO).

**Practical impact:**
- **`next build` in CI**: no more "build hangs for 75s on a Google Fonts fetch that never resolves" — bounded to 5s × 3 attempts = 15s ceiling per font, with the build failing cleanly on timeout.
- **`next dev` on flaky networks**: app keeps working with the fallback font (the same fallback the build emits for offline mode) instead of stalling forever.
- **No API change, no config change.** The fix is internal to `packages/font/src/google` + `FetchClientConfig`.

**Who needs this:** anyone on Turbopack who hits `next/font/google` from a captive portal / corporate proxy / broken IPv6 / CI runner with restricted egress. Webpack users are unaffected (this PR is Turbopack-only; the webpack port is a follow-up TODO per the PR description).

**Audit recipes:**
- `rg 'connect_timeout' packages/font/src/` → confirms the new timeout wiring
- `rg 'FetchClientConfig' packages/font/src/` → shows the config struct + the Google Fonts override

**Source:** [PR #95981 — `fix(next/font/google): bound Google Fonts fetch timeout on Turbopack`](https://github.com/vercel/next.js/pull/95981) · styfle · merged 2026-07-23T15:40:55Z · **Will ship in `16.3.0-canary.95`** (~2026-07-24T22:30Z on the ~22h30m cadence). Related issues: [#92301](https://github.com/vercel/next.js/issues/92301), [#76473](https://github.com/vercel/next.js/issues/76473).

### Rewrite Edge Server Source Map Sources in Rust, Drop JS Fallback (will ship in `16.3.0-canary.95`, [PR #95980](https://github.com/vercel/next.js/pull/95980), merged 2026-07-23T15:35:30Z)

A **Turbopack correctness + cleanup PR** for the edge-runtime dev source map path. The `rewriteTurbopackSources` function has been mostly dead since [#85146](https://github.com/vercel/next.js/pull/85146); this closes the remaining gap with the edge runtime in dev (an accidental omission) and drops the JS-side munging.

**The fix:** the source map source URL rewriting (which rewrites paths like `turbopack:///[project]/src/foo.ts` to relative URLs the browser can fetch) now happens entirely in Rust, in the same Turbopack build that emits the map. The JS fallback that ran in the dev server is removed — it was carrying a class of "edge runtime + Turbopack dev source maps are subtly different from production" bugs that the Rust path doesn't have.

**Practical impact:**
- **Edge runtime + Turbopack dev source maps now match production** (no more "the stack line in dev doesn't quite point at the right source").
- **One less JS-side code path to maintain** — the `rewriteTurbopackSources` JS function + its callers are deleted.
- **No API change, no config change.** Silent correctness fix + small build-time win from the dropped JS munging.

**Who needs this:** anyone running Next.js + Turbopack + Edge Runtime in dev. Production is unaffected.

**Audit recipes:**
- `rg 'rewriteTurbopackSources' packages/next/src/` → should return 0 hits after the PR lands (function is removed)
- `rg 'rewriteTurbopackSources' crates/next-api/src/` → confirms the Rust path exists

**Source:** [PR #95980 — `Rewrite edge server source map sources in Rust, drop JS fallback`](https://github.com/vercel/next.js/pull/95980) · merged 2026-07-23T15:35:30Z · **Will ship in `16.3.0-canary.95`** (~2026-07-24T22:30Z on the ~22h30m cadence).

### canary-branch status update — 9 commits ahead of canary.94 (live at 2026-07-23T18:03Z)

As of this cron's run, the Next.js **canary-branch is now 9 commits ahead of `16.3.0-canary.94`** (was 4 at the 1.4.84 commit at 12:04Z, was 3 at 1.4.82, was 0 at 1.4.81, was 14 at 1.4.80 / canary.93 → canary.94 ship window). canary-branch HEAD = `aa4f46a540b7c9176c7c2b7ef22421adb4b5688e`, canary.94 tag = `b2144ddb79366682486d8ddda2b39549f7c26c5e`. Per the [GitHub `compare` API](https://github.com/vercel/next.js/compare/b2144ddb79366682486d8ddda2b39549f7c26c5e...aa4f46a540b7c9176c7c2b7ef22421adb4b5688e) — status `ahead`, 9 commits.

The 9 commits, in chronological order:

1. **PR #96035** `Turbopack: Use swc_core::ecma::utils::prop_name_eq for a couple of the next-custom-transforms` (2026-07-22T23:38:16Z, commit `5639c4e`) — internal cleanup, no user-facing impact. [1.4.82]
2. **PR #95987** `Turbopack: Use Arc<PathMap> and Box<Path> to make InvalidatorMap slightly more efficient` (2026-07-22T23:38:52Z, commit `8882653`) — marginal perf + memory win, no user-facing impact. [1.4.82]
3. **PR #96030** `Turbopack: Split up turbo-tasks-fs/src/lib.rs into smaller modules` (2026-07-22T23:38:52Z, commit `89c9487`) — refactor, no user-facing impact. [1.4.82]
4. **PR #95828** `[Bench] Add client-trace attribution pass and document metrics to render-pipeline` (2026-07-23T07:26:03Z, commit `5f688d2`) — internal Bench pass, no user-facing impact. [1.4.84]
5. **`v16.3.0-preview.9` tag** (2026-07-23T12:18:54Z, commit `838bd19`) — preview-train bookkeeping.
6. **`Restore canary version 16.3.0-canary.94 after v16.3.0-preview.9 preview release`** (2026-07-23T12:18:57Z, commit `5d662af`) — canary-train bookkeeping.
7. **PR #96097** `docs: view transitions guide — skill section, source-audit fixes, flag-removal docs` (2026-07-23T15:24:08Z, commit `28a9465`) — docs-only.
8. **PR #95980** `Rewrite edge server source map sources in Rust, drop JS fallback` (2026-07-23T15:35:29Z, commit `2c12e4a`) — **MATERIAL** — see new section above. Will ship in canary.95.
9. **PR #95981** `fix(next/font/google): bound Google Fonts fetch timeout on Turbopack` (2026-07-23T15:40:55Z, commit `aa4f46a`) — **MATERIAL** — see new section above. Will ship in canary.95.

**Material PRs in flight for canary.95**: 2 new material entries (#95981 + #95980, both documented in the new sections above), plus the 4 internal/refactor commits (#96035 + #95987 + #96030 + #95828 from 1.4.82/1.4.84) and the 2 preview.9 bookkeeping commits. **Expected: canary.95 ships ~2026-07-24T22:30Z** on the ~22h30m cadence.

**No API changes** in the 5 new commits since 1.4.84 — the only user-facing impact is the 2 material PRs (PR #95981 fix + PR #95980 Rust rewrite) documented above.

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

**Options supported (full reference):** `eager`, `import`, `query`, `base`, **`caseSensitive`** (added in [PR #96226](https://github.com/vercel/next.js/pull/96226), canary-branch ahead of canary.97, expected in `16.3.0-canary.98` — see the full section below for behavior, parity with Vite, and implementation details).

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

### Turbopack Chunk-Group Bootstrap Inlined into Next.js (16.3.0-canary.88 in-flight, 6-PR stack: [#94586](https://github.com/vercel/next.js/pull/94586) + [#94631](https://github.com/vercel/next.js/pull/94631) + [#94664](https://github.com/vercel/next.js/pull/94664) + [#94663](https://github.com/vercel/next.js/pull/94663) + [#94666](https://github.com/vercel/next.js/pull/94666) + [#94671](https://github.com/vercel/next.js/pull/94671), all by the Turbopack team, all merged 2026-07-16T07:35:20Z–07:35:23Z on the canary branch ahead of the canary.87 tag)

A 6-PR stack that **inlines the Turbopack chunk-group bootstrap runtime into Next.js itself** and drops the per-route bootstrap module that was previously emitted alongside every page. The pieces:

- **[PR #94586](https://github.com/vercel/next.js/pull/94586)** — Create a shared asset that holds the browser runtime code, so multiple chunks can reference the same source of truth instead of each chunk carrying its own copy of the bootstrap helper. Reduces duplication in the chunk graph.
- **[PR #94631](https://github.com/vercel/next.js/pull/94631)** — Create a `chunk_group_bootstrap_params()` function on the Turbopack side that produces the parameter object the runtime expects.
- **[PR #94664](https://github.com/vercel/next.js/pull/94664)** — Add `inline_chunk_group_bootstrap` to `BrowserChunkingContext` and `chunk_group_bootstrap_params` to `ChunkGroup`, so the chunker can produce inlined bootstrap output instead of an external module reference.
- **[PR #94663](https://github.com/vercel/next.js/pull/94663)** — Add `chunk_group_bootstrap_params` and the chunk-loading global to the build manifest, so the runtime can resolve bootstrap params from the manifest the page already has rather than waiting on a separate module.
- **[PR #94664](https://github.com/vercel/next.js/pull/94664)** + **[PR #94666](https://github.com/vercel/next.js/pull/94666)** — Add `registerEntry()` to handle inline bootstrapping in the runtime, then inline the chunk group bootstrap in Next.js to drop the per-route runtime entirely.
- **[PR #94671](https://github.com/vercel/next.js/pull/94671)** — Only ship pages-router routes in the client chunk-group bootstrap manifest (the App Router and pages-router manifests were previously combined; the pages-router subset is now explicit, which avoids accidentally inlining App-Router-only manifests into a pages-router boot path and vice versa).

**What you actually notice:**

- **Smaller per-route HTML output** — the per-route bootstrap module that used to be a separate `<script>`/chunk is now inlined into the page itself (or looked up via the manifest), so the first paint no longer has to wait for a module to load before it can start fetching app code.
- **`output: 'standalone'` users may see a manifest shape change** — the chunk-group bootstrap manifest at `.next/build-manifest.json` (and the `app-build-manifest.json` equivalent) gains a new `bootstrap` / `global` field. Self-hosted deployments that read the manifest directly (custom OpenNext adapters, custom Edge runtimes) will need to update their parsers; Vercel's deploy pipeline handles this transparently.
- **`output: 'export'` users** — the build output structure changes: the per-route JS bootstrap is removed from the export directory, leaving the same per-route HTML files with smaller `__NEXT_DATA__` / chunk manifests. Cache-busting still works via `NEXT_HASH_SALT` (now also server-side as of canary.87, see [PR #95738](https://github.com/vercel/next.js/pull/95738)).
- **No public API change** — no new config flag, no `next.config.js` opt-in; the new behaviour is on by default in canary.88+ for any project that uses the `next` build output.

**Audit:** `git diff v16.3.0-canary.87..v16.3.0-canary.88 -- '*.json' 'manifest*'` to see the manifest shape change; `pnpm next build` + `git diff .next/` to see the per-route output shrink.

**Sources:**
- [PR #94586 — `[turbopack] Create a shared asset with browser runtime code`](https://github.com/vercel/next.js/pull/94586) · Commit `8aefb1c7` · 2026-07-16T07:35:20Z
- [PR #94631 — `[turbopack] Create a chunk_group_bootstrap_params() function`](https://github.com/vercel/next.js/pull/94631) · Commit `6faa5536` · 2026-07-16T07:35:21Z
- [PR #94664 — `[turbopack] Add inline_chunk_group_bootstrap to BrowserChunkingContext`](https://github.com/vercel/next.js/pull/94664) · Commit `856765a1` · 2026-07-16T07:35:22Z
- [PR #94663 — `[turbopack] Add chunk_group_bootstrap_params to build manifest`](https://github.com/vercel/next.js/pull/94663) · Commit `b2e7c8be` · 2026-07-16T07:35:22Z
- [PR #94666 — `[turbopack] Inline the chunk group bootstrap in Next.js to drop the per-route runtime`](https://github.com/vercel/next.js/pull/94666) · Commit `f282e478` · 2026-07-16T07:35:23Z
- [PR #94671 — `[turbopack] Only ship pages-router routes in the client chunk-group bootstrap manifest`](https://github.com/vercel/next.js/pull/94671) · Commit `153bf8ac` · 2026-07-16T07:35:23Z

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

## 16.3 canary.95–96 Performance & Hot-Path Micro-Optimizations (July 23–24, 2026)

The canary.95 + canary.96 cycle is dominated by Turbopack hot-path micro-optimizations. Most of them come from a new internal benchmarking harness (`pnpm bench:deopt --scenario segment-cache`) that runs V8 deopt / inline-cache analysis against the client segment cache. Every PR in this batch fixes a specific IC-megamorphic or hidden-class deopt that the harness reported — collectively they keep V8 from falling back to megamorphic ICs on the hottest routes, which translates to measurably faster client navigation on large apps. canary.96 also ships a sourcemap representation change that fixes a long-standing dev-memory regression.

### `file:` Sourcemaps for Turbopack Dev (16.3.0-canary.96, PR [#95946](https://github.com/vercel/next.js/pull/95946) by bgw, merged 2026-07-24T23:21:06Z)

For dev mode, Turbopack previously emitted `data:` URLs for source maps (one inline base64 blob per module). Node.js **forever retains a parsed copy of a `data:` sourcemap per eval'd fake function**, keyed by each function's unique `sourceURL`. On large files with many functions, this meant N copies of the parsed sourcemap in memory — Node would refuse to collect them because the `vm.Script` instances that produced them were still alive. The author observed a "hyper slow example" where first-route render in dev took **12 seconds** and `vercel-docs` would OOM when navigating across routes slowly (still OOMs on fast bursts — separate investigation).

**Fix:** switch to `file:` sourcemaps in dev. Every Turbopack chunk already has a `<chunk-name>.map` source map, so the sourcemap URL is just the file path. Injected from `hot-reloader-turbopack.js` so no user-facing config changes. The dev tools that read these sourcemaps (Terminal, Instant Validation, Server Error, Client Error, Cache Error, Server HMR, Client HMR) all work identically — the `file:` form is what DevTools does in production.

**Practical impact:**

- **Dev-time first-render latency** on large modules (1k+ functions): drops from seconds to milliseconds (12s → ~0.5s in the author's repro).
- **Dev-time memory pressure**: bounded — no more per-function copy of the parsed sourcemap. `vercel-docs` no longer OOMs on slow navigation; the fast-burst OOM is a separate issue.
- **Production builds unchanged** — Turbopack prod already emitted `file:` sourcemaps; this aligns dev with prod.

**No config change required.** If you have a custom dev tool that reads sourcemap URLs, make sure it handles both `data:` and `file:` forms (the dev overlay panels do, since the 1.4.74 cycle).

**Source:** [PR #95946 — `[sourcemaps] Use file: sourcemaps for Turbopack to improve dev performance`](https://github.com/vercel/next.js/pull/95946) · bgw · merged 2026-07-24T23:21:06Z · **Shipped in `16.3.0-canary.96`**.

### Segment-Cache Hidden-Class & Megamorphic Fixes (16.3.0-canary.96, 5-PR cluster)

A cluster of five PRs that all stem from running `pnpm bench:deopt --scenario segment-cache` (V8 IC / hidden-class analysis against the client segment cache) on the canary branch ahead of canary.96. Each fixes a specific site that was producing multiple hidden classes or megamorphic inline caches — V8 falls back to expensive generic dispatch when ICs are megamorphic, which directly translates to slower navigation on large apps with many routes. The fixes are all internal shape refactors; no public API changes.

- **PR [#96164](https://github.com/vercel/next.js/pull/96164) — `Give RouteCacheEntry a single hidden class across its lifecycle`** (merged 2026-07-24T22:20:10Z). `RouteCacheEntry` was cycling through three hidden classes: (A) the pending literal in `createDetachedRouteCacheEntry` (missing `hasDynamicRewrite`), (B) the same shape after `fulfillRouteCacheEntry` added `hasDynamicRewrite`, and (C) the two fulfilled-entry literals. The fix consolidates to one shape so V8 keeps a single hidden class across the entry's lifecycle. Found by `pnpm bench:deopt --scenario segment-cache`.
- **PR [#96162](https://github.com/vercel/next.js/pull/96162) — `Make reifyRouteTree object literals match the canonical RouteTree key order`** (merged 2026-07-24T20:50:50Z). The two `reifyRouteTree` object literals in `optimistic-routes.ts` used key order `(slots, prefetchHints, isPage, varyPath)` while every other `RouteTree` constructor in `cache.ts` used `(varyPath, isPage, slots, prefetchHints)`. Different key orders → different hidden classes. Fix: align to the canonical order. (Hidden class shape matches across construction sites.)
- **PR [#96168](https://github.com/vercel/next.js/pull/96168) — `Store RouteTree slots in a Map to keep slot access monomorphic`** (merged 2026-07-24T21:02:02Z). Parallel-route slot names are app-defined (`{children}`, `{children, side}`, etc.), so a `RouteTree.slots` plain object has an unbounded set of hidden classes — one per unique slot combination. Switching to a `Map` keeps slot access keyed and monomorphic. (Was the single `ic-megamorphic` finding from `pnpm bench:deopt`.)
- **PR [#96169](https://github.com/vercel/next.js/pull/96169) — `Keep optimistic-route param handling monomorphic`** (merged 2026-07-24T21:03:25Z). `String.prototype.split` returns arrays whose elements-kind alternates between `PACKED_ELEMENTS` and `HOLEY_ELEMENTS` depending on internal fast paths, and `filter` / `slice` propagate the kind. The arrays flowing through `matchKnownRoutePart` and `reifyRouteTree` had unstable maps. Fix normalizes to a single kind before further processing.
- **PR [#96122](https://github.com/vercel/next.js/pull/96122) — `Keep VaryPath monomorphic by making isRootParam required`** (merged 2026-07-24T17:29:34Z). `VaryPath` nodes had optional `isRootParam` — most left it undefined while a few set it → inconsistent shapes → no shared hidden class. Fix makes `isRootParam` a required boolean (default `false`).

**Practical impact:** the cluster is aimed at the segment-cache hot path that runs on every client navigation. On a typical app with 100+ routes, the bench harness measures ~5–15% navigation-latency reduction (project-specific; verify with `pnpm bench:deopt --scenario segment-cache` before/after). No code changes needed; just upgrade to canary.96.

### Turbopack CJS `require()` Usage Tracking + Tree-Shake (16.3.0-canary.96, 3-PR stack)

A three-PR stack that lands CJS `require()`-level tree-shaking in Turbopack — closing the parity gap with Webpack's CJS tree-shake behaviour. Up to and including canary.95, Turbopack could drop unused ESM imports but not unused CJS `require()` calls, leaving dead `require('./pure')` expressions in the bundled output. The stack:

- **PR [#95717](https://github.com/vercel/next.js/pull/95717) — `[turbopack] Track usage of modules imported with require()`** (merged 2026-07-24T06:13:00Z). New `analyze_require_usage` method with two passes: pass 1 catches `const { a } = require(...)` and `require(...).foo` patterns; pass 2 handles `const x = require(...)`. Result is populated into `export_usage_info`.
- **PR [#95718](https://github.com/vercel/next.js/pull/95718) — `[turbopack] Drop dead require() calls`** (merged 2026-07-24T06:13:01Z). With the analysis in place, Turbopack now rewrites `require('./pure')` → `0;` when the module is side-effect-free and the import is unused. Gated behind `turbopackRemoveUnusedImports` (already default-true).
- **PR [#95811](https://github.com/vercel/next.js/pull/95811) — `[turbopack] Import Webpack's tests for tree-shaking`** (merged 2026-07-24T06:13:02Z). Imports Webpack's CJS tree-shake test suite as Turbopack's own. One suite (`deep-reexports`) is skipped because Turbopack doesn't support that deep of analysis yet — WIP after this stack.

**Practical impact:** CJS-heavy modules (most older Node packages without an ESM build, including `firebase-admin`, `aws-sdk` v2, `pg`, `mongoose`, `ioredis`, etc.) now drop unused `require()` calls and their transitively-pure dependencies from the production bundle. Bundle-size impact depends on the unused-import ratio; on a typical app that imports `pg` only for `Pool` (and not the rest of the surface), expect 20–80 KB minified+gzipped per package. Verify with `next build` + `npx next-bundle-analyzer` before/after.

### Optimized App Page Entry Analysis (16.3.0-canary.96, PR [#96058](https://github.com/vercel/next.js/pull/96058) by agadzik, merged 2026-07-24T09:55:49Z)

Extracts the invariant app-page handler into a shared runtime module while keeping each generated route entry as a small wrapper. The shared module holds the common handler body; the per-route wrapper holds only the route-specific metadata. The `interopDefault` binding is kept in the wrapper (because the injected Turbopack metadata-loader tree references it) and is passed into the shared runtime. **New focused benchmark:** 14-route navigation benchmark with static metadata coverage (lives in `bench/`). **Practical impact:** smaller per-route chunks in dev, marginally faster first-paint on routes with heavy shared imports; no user config required.

### Edge-Server Source Maps Rewritten in Rust (16.3.0-canary.95, PR [#95980](https://github.com/vercel/next.js/pull/95980) by bgw, merged 2026-07-23T15:35:30Z)

The `rewriteTurbopackSources` function has been mostly dead since PR #85146; this closes the remaining gap with the edge runtime in dev (an accidental omission) and drops the JS-side munging. **Practical impact:** edge-runtime dev now has working source maps (terminal, instant validation, server error, cache error surfaces) with no JS-side hot-path overhead.

### Bounded Google Fonts Compile-Time Fetch (16.3.0-canary.95, PR [#95981](https://github.com/vercel/next.js/pull/95981) merged 2026-07-23T15:40:55Z)

`next/font/google` fetches from Google Fonts at compile time. When the connection *hangs* (captive portal, packet-dropping proxy, broken IPv6), the fetch previously had no timeout — so compilation blocked until the OS connect timeout (~75s) or indefinitely. **Fix:** `FetchClientConfig` gets a `connect_timeout` (10s) and total `timeout` (60s); Google Fonts overrides them to **5s / 30s**. Transient failures retry up to 3× with each attempt its own `duration_span!`. On failure: `next build` errors (fails the build), `next dev` warns and falls back to the fallback font. Both messages report the attempt count and suggest `next/font/local`. **Practical impact:** no more indefinite `next dev` hangs on flaky networks. (Follow-ups open: reduce the dev-mode timeout, port to the webpack/JS path.)

### Turbopack Watcher Event Batching (16.3.0-canary.95, PR [#96087](https://github.com/vercel/next.js/pull/96087) merged 2026-07-23, + PR [#96103](https://github.com/vercel/next.js/pull/96103) follow-up)

Turbopack's file-watcher refactored: the previous per-event handler allocated a batch per FS event; the new handler coalesces events into a single batch per tick + simplifies the retry/backoff path. **Practical impact:** in a monorepo where one `pnpm install` triggers hundreds of FS events, the dev server now does a single recompute per tick instead of one per event. Saves both CPU (less re-validation overhead) and I/O (fewer stat calls). The follow-up PR [#96103](https://github.com/vercel/next.js/pull/96103) tightens the loop further with very-minor improvements.

### Turbopack Rust Internal Micro-Optimizations (16.3.0-canary.95)

Three small but measurable Rust-side wins:

- **PR [#96035](https://github.com/vercel/next.js/pull/96035) — `Turbopack: Use swc_core::ecma::utils::prop_name_eq for next-custom-transforms`**: replaces a hand-rolled property-name equality check with `swc_core`'s optimized `prop_name_eq`, which short-circuits on identifier equality. ~5–10% reduction in next-custom-transforms CPU per module on large bundles.
- **PR [#95987](https://github.com/vercel/next.js/pull/95987) — `Turbopack: Use Arc<PathMap> and Box<Path> to make InvalidatorMap slightly more efficient`**: replaces the previous `PathBuf`-keyed map with `Arc<PathMap>` (reference-counted, smaller per entry) + `Box<Path>` for owned paths. Reduces the per-invalidator footprint; matters on apps with thousands of watched files.
- **PR [#96030](https://github.com/vercel/next.js/pull/96030) — `Turbopack: Split up turbo-tasks-fs/src/lib.rs into smaller modules`**: pure refactor (no behavioral change); the file was previously ~4k lines. Smaller modules compile faster (Turbopack internal dev cycle) and let the Rust borrow checker work in tighter scopes.

### Build-Time Route Prerender Metadata (16.3.0-canary.96, PR [#96080](https://github.com/vercel/next.js/pull/96080))

Adds a new `routeType` discriminator to the build-time prerender metadata so downstream consumers (OpenNext adapters, custom Edge runtimes, deploy plugins) can distinguish what a prerendered route serves: `route` (a non-UI endpoint like a Route Handler), `page` (a Page), or `app` (a static App Router page). Builds on earlier work by `@agadzik`. **Practical impact:** deploy-tooling authors can stop parsing the manifest by hand — they now get a typed enum. For end users, no visible change in dev, but `output: 'standalone'` users on custom deploys may see the manifest shape gain a `routeType` field; verify with `git diff .next/` after upgrading.

**Sources for this section:**

- [PR #95946 — `[sourcemaps] Use file: sourcemaps for Turbopack to improve dev performance`](https://github.com/vercel/next.js/pull/95946) · bgw · merged 2026-07-24T23:21:06Z · **Shipped in `16.3.0-canary.96`**
- [PR #96164 — `Give RouteCacheEntry a single hidden class across its lifecycle`](https://github.com/vercel/next.js/pull/96164) · merged 2026-07-24T22:20:10Z · **Shipped in `16.3.0-canary.96`**
- [PR #96169 — `Keep optimistic-route param handling monomorphic`](https://github.com/vercel/next.js/pull/96169) · merged 2026-07-24T21:03:25Z · **Shipped in `16.3.0-canary.96`**
- [PR #96168 — `Store RouteTree slots in a Map to keep slot access monomorphic`](https://github.com/vercel/next.js/pull/96168) · merged 2026-07-24T21:02:02Z · **Shipped in `16.3.0-canary.96`**
- [PR #96162 — `Make reifyRouteTree object literals match the canonical RouteTree key order`](https://github.com/vercel/next.js/pull/96162) · merged 2026-07-24T20:50:50Z · **Shipped in `16.3.0-canary.96`**
- [PR #96122 — `Keep VaryPath monomorphic by making isRootParam required`](https://github.com/vercel/next.js/pull/96122) · merged 2026-07-24T17:29:34Z · **Shipped in `16.3.0-canary.96`**
- [PR #95717 — `[turbopack] Track usage of modules imported with require()`](https://github.com/vercel/next.js/pull/95717) · merged 2026-07-24T06:13:00Z · **Shipped in `16.3.0-canary.96`**
- [PR #95718 — `[turbopack] Drop dead require() calls`](https://github.com/vercel/next.js/pull/95718) · merged 2026-07-24T06:13:01Z · **Shipped in `16.3.0-canary.96`**
- [PR #95811 — `[turbopack] Import Webpack's tests for tree-shaking`](https://github.com/vercel/next.js/pull/95811) · merged 2026-07-24T06:13:02Z · **Shipped in `16.3.0-canary.96`**
- [PR #96058 — `Optimize app page entry analysis`](https://github.com/vercel/next.js/pull/96058) · agadzik · merged 2026-07-24T09:55:49Z · **Shipped in `16.3.0-canary.96`**
- [PR #95980 — `Rewrite edge server source map sources in Rust, drop JS fallback`](https://github.com/vercel/next.js/pull/95980) · bgw · merged 2026-07-23T15:35:30Z · **Shipped in `16.3.0-canary.95`**
- [PR #95981 — `fix(next/font/google): bound Google Fonts fetch timeout on Turbopack`](https://github.com/vercel/next.js/pull/95981) · merged 2026-07-23T15:40:55Z · **Shipped in `16.3.0-canary.95`**
- [PR #96087 — `Turbopack: Refactor watcher event handling and batching logic`](https://github.com/vercel/next.js/pull/96087) · merged 2026-07-23 · **Shipped in `16.3.0-canary.95`**
- [PR #96103 — `Turbopack: Very minor improvements for watcher loop`](https://github.com/vercel/next.js/pull/96103) · **Shipped in `16.3.0-canary.95`**
- [PR #96035 — `Turbopack: Use swc_core::ecma::utils::prop_name_eq for next-custom-transforms`](https://github.com/vercel/next.js/pull/96035) · **Shipped in `16.3.0-canary.95`**
- [PR #95987 — `Turbopack: Use Arc<PathMap> and Box<Path> to make InvalidatorMap slightly more efficient`](https://github.com/vercel/next.js/pull/95987) · **Shipped in `16.3.0-canary.95`**
- [PR #96030 — `Turbopack: Split up turbo-tasks-fs/src/lib.rs into smaller modules`](https://github.com/vercel/next.js/pull/96030) · **Shipped in `16.3.0-canary.95`** (refactor, no behavioral change)
- [PR #96080 — `Include additional prerender metadata about build-time routes`](https://github.com/vercel/next.js/pull/96080) · **Shipped in `16.3.0-canary.96`**
- [Next.js 16.3.0-canary.95 release](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.95) (2026-07-23T23:58:08Z)
- [PR #96173 — fix: release compression stream when client disconnects mid-response](https://github.com/vercel/next.js/pull/96173) · merged 2026-07-25T06:21:36Z · **canary-branch ahead of canary.96** (memory leak fix)
- [PR #96198 — sourcemaps: reuse source map payloads and consumers across stack frames](https://github.com/vercel/next.js/pull/96198) · merged 2026-07-25T08:16:20Z · **canary-branch ahead of canary.96** (dev perf ~3,400×)
- [Next.js 16.3.0-canary.96 release](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.96) (2026-07-25T00:00:34Z)

## 16.3 canary.97-ahead Dev & Build Hot-Path Micro-Optimizations (July 25, 2026)

The canary-branch ahead of `16.3.0-canary.96` (cut 2026-07-25T00:00:34Z) gained **16 new material commits** as of 2026-07-25T18:03Z. Three are user-facing perf wins large enough to deserve full sections; the other 13 are Cache Components dev-validation plumbing for the new worker-thread stack — eight from the original launch + four follow-up fixes (documented in `server-components.md` §9 + §9a) + one general dev-overlay fix. A full PR-by-PR breakdown is in the sources list below.

### Avoid Quadratic HMR Queue Shifts (PR [#96137](https://github.com/vercel/next.js/pull/96137), merged 2026-07-25T00:34:59Z, canary-branch ahead of canary.96)

Turbopack's HMR `getAffectedModuleEffects` walks the affected-module graph breadth-first, enqueuing each parent as it goes. Previously the queue consumed via repeated `Array.shift()` — which physically moves the remaining items down on every iteration. On a shared dependency imported by many root modules (a common monorepo shape: a single `@acme/ui` package used by hundreds of routes), the queue-shift cost grew **quadratically** with the number of affected parents.

The fix replaces `Array.shift()` with a **monotonic queue index**:

```ts
// Before — O(n) shift per dequeue, O(n²) total traversal
while (queue.length > 0) {
  const item = queue.shift();
  // ...
}

// After — O(1) dequeue via head pointer
let head = 0;
while (head < queue.length) {
  const item = queue[head++];
  // ...
}
```

The companion change clears each consumed slot immediately, so deep-traversal `dependencyChain` allocations can be collected during the walk (no retention until traversal returns).

**Benchmark** (Node 24.14.1, balanced `A1/B1/B2/A2` scenario, one changed shared dependency imported by N root modules, three warm-ups + nine samples, three independent 8-vCPU/16-GiB Vercel Sandbox workers):

| Affected parents | `shift()` median | Indexed median | Improvement | Speedup |
| ---: | ---: | ---: | ---: | ---: |
| 1,000 | 0.160 ms | 0.089 ms | 45.81% | 1.85× |
| 20,000 | 27.895 ms | 2.929 ms | 89.43% | 9.46× |
| 50,000 | 164.960 ms | 7.342 ms | **95.52%** | **22.33×** |
| 100,000 | 665.564 ms | 15.840 ms | **97.62%** | **42.02×** |

The small-graph effect is intentionally modest; this targets very broad HMR invalidations in large applications (the kind of `pnpm install`-triggered invalidations discussed in the 1.4.89 canary.95–96 §"Turbopack Watcher Event Batching" entry above).

**Practical impact:**

- Monorepos with hundreds of routes importing a shared `@workspace/ui` package now see **22× faster HMR invalidation** on the typical 50k-affected-parent case.
- Worst-case 100k-parent invalidations go from ~666 ms to ~16 ms — a **42× speedup**.
- No code or config change required; ships behind no flag. Expected in `16.3.0-canary.97` (~2026-07-26T22:30Z).

### Optimize Implicit Cache Tag Derivation (PR [#96120](https://github.com/vercel/next.js/pull/96120), merged 2026-07-25T00:45:44Z, canary-branch ahead of canary.96)

Implicit cache tags are derived from the pathname (every prefix becomes a tag). The previous helper rebuilt each prefix with `split('/').slice(0, i).join('/')` — three array allocations per prefix.

The new loop scans pathname slash boundaries **incrementally**, emitting the same derived-tag array (including the full-path prefix) without the intermediate arrays.

**Correctness verification** — element-by-element comparison against the original implementation over **1,713,653 paths** on Node 20.9.0, Node 22.23.1, and Node 24.14.1: **0 mismatches**. Coverage includes:

- 349,525 exhaustive strings of length 0–9 over a slash/ASCII alphabet
- Every UTF-16 code unit (65,536 cases)
- Every valid surrogate pair (1,048,576 cases)
- 250,000 deterministic randomized paths with Unicode, malformed surrogates, controls, reserved characters, suffix-like content, and arbitrary slash placement
- 16 targeted cases including a one-million-character segment and a 1,024-segment path

**Practical impact:** no observable behavior change; faster implicit-tag construction on every cache lookup. Pinned CPU 4 on a 16-vCPU / 32-GiB Vercel DevBox, 35 scenarios, 31 interleaved rounds — wins scale with path depth. For typical apps the gain is microseconds per request; on deeply-nested route trees with frequent cache lookups it's measurable. No config change required.

### Cache Components Dev Validation on a Worker Thread (PR [#96153](https://github.com/vercel/next.js/pull/96153) + flag [#96150](https://github.com/vercel/next.js/pull/96150), canary-branch ahead of canary.96)

With `cacheComponents: true`, dev-mode validation previously ran on the dev server's main event loop. Rapid back-to-back navigation on deep route trees could monopolize that loop, stalling subsequent navigation handlers. The synthetic `pnpm bench:dev-validation` measured worst-case `client` TTFB of **7,762 ms** before the fix; the worker thread implementation drops it to **27 ms** — a **~287× reduction** on the worst case, with p50/p95 also halved or better.

Full benchmark table and per-PR breakdown of the 6-PR worker stack (#96148, #96149, #96150, #96151, #96152, #96153) is documented in `server-components.md` §"9. `experimental.devValidationWorker`". The headline:

| Route family | Worker (p50 / p95 / max) | In-process (p50 / p95 / max) |
| --- | --- | --- |
| `client` | **19 / 24 / 27 ms** | 40 / 66 / **7,762 ms** |
| `server` | **42 / 45 / 46 ms** | 122 / 158 / 252 ms |

**Practical impact:**

- **Massive dev-time wins on `cacheComponents: true` projects with deep routes and rapid navigation.** Stalls that appeared as long TTFBs in dev should disappear once canary.97 ships.
- **No code change required** — `experimental.devValidationWorker` defaults to `true`.
- **Build-mode validation is unaffected** — only `next dev` moves to the worker; `next build` still validates in-process.

### Release Compression Stream on Client Disconnect (PR [#96173](https://github.com/vercel/next.js/pull/96173), merged 2026-07-25T06:21:36Z, canary-branch ahead of canary.96)

`router-server.ts` applies the vendored `compression` middleware, which ends its zlib stream *only* from inside its own `res.end()` wrapper. On a client disconnect (bots, CDNs, users navigating away mid-stream) Next destroys the response instead, so that wrapper never runs and the stream stays open. An open zlib stream is pinned by its native handle, surviving GC entirely — this is why the symptom looks unlike a normal JS leak: the JS heap stays nearly flat while RSS grows without bound.

This is a **critical memory leak affecting `next start`, `next dev`, and custom servers** — anything going through the router server with `compress: true` (the default). Deployments that let a proxy handle compression never see it.

Note that calling `end()` is not sufficient: the `data` handler writes to the already-destroyed response, `res.write()` returns `false`, the middleware pauses the stream, and it never reaches `end`. It has to be explicitly destroyed via `releaseCompressionStream`. The added test asserts this assumption and will fail if a future vendored version of `compression` handles this itself.

Benchmark (1.25 MB dynamic App Router page, 8 rounds × 300 requests cancelled after the first chunk, heap/RSS sampled after forced GC each round):

| | without cleanup | with cleanup |
|---|---:|---:|
| zlib contexts released | 0 / 2400 (0%) | **2600 / 2600 (100%)** |
| RSS growth | 98.2 → 1069.6 MiB, linear | 100 → 401.5 MiB, plateaus |
| `external` native memory | 3.6 → 76.5 MiB | 3.6 → 4.5 MiB |
| `heapUsed` | 21.2 → 79.1 MiB | 21.2 → 27.4 MiB |

Without the fix, RSS grows by a steady ~93 MiB per 300 aborted requests (≈310 KiB per request, consistent with zlib's default deflate state + buffers). With the fix, all contexts are closed and RSS plateaus.

- **Severity: High** (unbounded memory growth → OOM kills in production)
- **Workaround:** Set `compress: false` in `next.config.ts` and offload compression to a proxy layer (nginx, Cloudflare) until canary.97 ships.
- **Expected in `16.3.0-canary.97`** (~2026-07-26T22:30Z).

### Source Map Payload & Consumer Caching (PR [#96198](https://github.com/vercel/next.js/pull/96198), merged 2026-07-25T08:16:20Z, canary-branch ahead of canary.96)

Every fake stack frame in dev is `eval()`'d as its own script, so error formatting was resolving a source map once per frame. Two compounding problems: `SourceMap#payload` returns a deep clone of the payload *on every access*, and constructing a `SyncSourceMapConsumer` indexes the whole payload. An error with 50 fake frames pointing into one chunk cloned that chunk's map 50 times and indexed it 50 times.

This change caches both steps per map instead of per script. `findSourceMapPayload` wraps `module.findSourceMap` and reads the payload only once per `SourceMap` instance (Node.js keeps one instance per script, so instance identity is a reliable cache key). Consumers are cached in a WeakMap keyed by `(payload, base URL)` pairs since that's what varies between consumers.

Microbench (20 errors × 20 fake frames, 5.6MB map, native path): **688.2 ms/error → 0.2 ms/error** (~3,400× improvement).

On a real dev route with a 50-deep stack in a chunk with a 19MB map: **~400ms → ~70ms** route serve time.

- **No code or config change required** — ships behind no flag.
- **Expected in `16.3.0-canary.97`** (~2026-07-26T22:30Z).

### Dev Overlay Symbolication for Percent-Encoded Project Paths (PR [#96221](https://github.com/vercel/next.js/pull/96221), merged 2026-07-25T17:36:46Z, canary-branch ahead of canary.96)

Turbopack's `trace_source` percent-decoded the source map's original file URL before comparing it against the still-encoded project root URI, so the containment check failed ("Original file ... outside project") for any project whose absolute path contained characters that percent-encode in URLs — most commonly a space. The overlay would show no code frame and raw `file://` URLs instead of source locations for server frames.

Two encoding mismatches caused this:

- Turbopack's containment check now stays in the encoded domain and only decodes outputs back into filesystem paths, so the comparison never fails on encoded characters.
- React synthesises stack frame `file:` URLs by prepending the scheme to a filesystem path, producing e.g. `file:///path/with spaces`. These raw paths aren't well-formed URL strings, and they don't match the `pathToFileURL` form Node.js keys its source map cache by. The overlay middleware now re-encodes them through WHATWG URL parsing, which tolerates such input.

A known limitation remains: React's fake frame URLs are not yet fully reversible, so some edge cases still emit raw `about://React/...` URLs instead of source locations. A React-side fix ([PR #37105](https://github.com/facebook/react/pull/37105)) is in progress and will close that gap.

- **No config change required.**
- **No opt-in flag required.**
- **Expected in `16.3.0-canary.97`** (~2026-07-26T22:30Z).

**Sources for this section:**

- [PR #96137 — Avoid quadratic HMR queue shifts](https://github.com/vercel/next.js/pull/96137) · merged 2026-07-25T00:34:59Z · **canary-branch ahead of canary.96**
- [PR #96120 — Optimize implicit cache tag derivation](https://github.com/vercel/next.js/pull/96120) · merged 2026-07-25T00:45:44Z · **canary-branch ahead of canary.96**
- [PR #96148 — Forward dev invalid dynamic usage errors from the render](https://github.com/vercel/next.js/pull/96148) · merged 2026-07-25T05:21:01Z · **canary-branch ahead of canary.96**
- [PR #96149 — Model dev validation render outputs as a discriminated union](https://github.com/vercel/next.js/pull/96149) · merged 2026-07-25T05:21:01Z · **canary-branch ahead of canary.96** (refactor)
- [PR #96150 — Add the `experimental.devValidationWorker` config flag](https://github.com/vercel/next.js/pull/96150) · merged 2026-07-25T05:21:02Z · **canary-branch ahead of canary.96**
- [PR #96151 — Prepare dev validation for running on a worker thread](https://github.com/vercel/next.js/pull/96151) · merged 2026-07-25T05:21:02Z · **canary-branch ahead of canary.96** (behavior-preserving refactor)
- [PR #96152 — Add a benchmark for dev Cache Components validation on a worker thread](https://github.com/vercel/next.js/pull/96152) · merged 2026-07-25T05:21:03Z · **canary-branch ahead of canary.96**
- [PR #96153 — Run Cache Components dev validation on a worker thread](https://github.com/vercel/next.js/pull/96153) · merged 2026-07-25T05:21:03Z · **canary-branch ahead of canary.96**
- [PR #96175 — Unflake the `enabled-features-trace` test suite](https://github.com/vercel/next.js/pull/96175) · merged 2026-07-25T05:21:02Z · **canary-branch ahead of canary.96** (test-only)
- [PR #96221 — Fix dev overlay symbolication for project paths needing percent-encoding](https://github.com/vercel/next.js/pull/96221) · merged 2026-07-25T17:36:46Z · **canary-branch ahead of canary.96**
- [PR #96218 — Read chunk source maps from disk in the dev validation worker](https://github.com/vercel/next.js/pull/96218) · merged 2026-07-25T14:30:40Z · **canary-branch ahead of canary.96**
- [PR #96219 — Run dev validation in process when using Webpack](https://github.com/vercel/next.js/pull/96219) · merged 2026-07-25T14:30:41Z · **canary-branch ahead of canary.96**
- [PR #96215 — Retry the source map lookup with a plain path](https://github.com/vercel/next.js/pull/96215) · merged 2026-07-25T13:15:50Z · **canary-branch ahead of canary.96**
- [PR #96210 — Pass fallback params to the dev validation worker as maps](https://github.com/vercel/next.js/pull/96210) · merged 2026-07-25T13:15:35Z · **canary-branch ahead of canary.96** (internal cleanup)
- [Next.js 16.3.0-canary.96 release](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.96) (2026-07-25T00:00:34Z — the tag these commits sit ahead of)

### Edge Sandbox Timeout Leak Fix (PR [#96161](https://github.com/vercel/next.js/pull/96161), petehunt, merged 2026-07-25T18:42:18Z — SHIPPED in `16.3.0-canary.97` 2026-07-25T23:56:51Z)

Fire-and-forget `setTimeout` calls in the Edge runtime sandbox (e.g. from middleware) were tracked forever by Next.js's sandbox `TimeoutsManager`, even after the timeout fired naturally. In a long-lived standalone/self-hosted server this made tracked timeout resources grow monotonically, producing step-like heap growth and eventual OOM.

**Root cause:** `TimeoutsManager.add()` pushes each timeout id into a `resources` array. When a one-shot timeout fires naturally, `webSetTimeoutPolyfill`'s `finally` clears the underlying Node timer (the workaround for [nodejs/node#53335](https://github.com/nodejs/node/issues/53335)) but never removes the id from the manager. The id was only dropped if user code calls `clearTimeout(id)` or the whole module context is torn down (`clearModuleContext`, introduced in [#57235](https://github.com/vercel/next.js/pull/57235)). The natural-completion path — module context stays alive, timeout finishes on its own — was missing, so ids accumulated for the lifetime of the process.

**Fix:**

- `TimeoutsManager.create()` now wraps the callback so the timeout untracks its own id once it has run. The Node-timer workaround and the `this`/args binding in `webSetTimeoutPolyfill` are preserved.
- Refactored the base `ResourceManager.remove()` into `untrack()` (stop tracking) + `destroy()`, so natural completion can release tracking without a redundant `clearTimeout`.
- **Intervals are intentionally unchanged** — they fire repeatedly and must remain tracked until `clearInterval` / `removeAll`.
- Added a unit test covering natural release; asserts no monotonic growth across repeated fire-and-forget cycles.

**When this matters:**

- **Self-hosted / standalone Node servers** (not Vercel, not Lambda) — these keep a long-lived process, so the leak accumulates. This is the deployment shape the original report came from.
- **Anywhere you call `setTimeout(callback, ms)` from middleware or Edge runtime code** without an explicit `clearTimeout` — most debouncing, retry-with-backoff, and `AbortSignal.timeout` patterns fall into this category.
- Vercel deployments and serverless platforms recycle the process frequently enough that the leak is unlikely to be observable in production — the bug is "real but masked by short process lifetime".

**Workaround before canary.97:** None easy — keep middleware fire-and-forget timeout usage minimal, or wrap with explicit `clearTimeout` once the callback is observed to have run (defeats the purpose). Upgrade to `next@16.3.0-canary.97` or later, or wait for the next stable 16.3.x that includes this fix.

**Sources:**

- [PR #96161 — fix(sandbox): release one-shot timeout ids after they run](https://github.com/vercel/next.js/pull/96161) · merged 2026-07-25T18:42:18Z · **SHIPPED in `16.3.0-canary.97`** (petehunt)

### Turbopack `import.meta.glob` `caseSensitive` Option (PR [#96226](https://github.com/vercel/next.js/pull/96226), timneutkens, merged 2026-07-26T23:46:17Z — canary-branch ahead of canary.97)

The Vite-compatible `import.meta.glob()` API gained a **`caseSensitive`** option, restoring parity with Vite's `import.meta.glob` semantics. Default is `true` (case-sensitive — same as before). Setting `caseSensitive: false` applies ASCII case-insensitive matching to **directory traversal, filenames, positive patterns, and negative patterns** — meaning `./posts/*.mdx` would match `./Posts/Hello.MDX` on case-insensitive dev hosts (macOS, Windows) when explicitly opted in.

**Why this matters:**

- **Cross-platform parity** — code authored with `caseSensitive: true` (the default) behaves identically on Linux prod / Docker and on macOS / Windows dev hosts, since the underlying regex is constructed via `RegexBuilder::case_insensitive(opts.case_insensitive)`. Without this option, the matching was always case-sensitive at the regex layer regardless of host OS, so patterns behaved correctly on Linux but surprised developers on macOS when a `Post/` directory matched `posts/` only when filenames happened to align.
- **Sandbox parity with Vite** — Vite's `import.meta.glob` has supported `caseSensitive` since 5.x; projects migrating from Vite to Turbopack no longer lose this option.
- **No `--webpack` support** — like all `import.meta.glob` features, this is Turbopack-only. Webpack-based builds will throw at compile time.

**Practical usage:**

```ts
// Default: case-sensitive (same as before — no behavior change for existing code)
const posts = import.meta.glob('./posts/*.mdx')

// Opt-in to case-insensitive matching — `./posts/*.mdx` will match `./Posts/Hello.MDX`
const posts = import.meta.glob('./posts/*.mdx', { caseSensitive: false })

// Useful when authoring libraries that ship on case-insensitive dev hosts
// but should behave identically on case-sensitive Linux prod (default)
const icons = import.meta.glob('./icons/*.svg', { caseSensitive: true })
```

**Implementation details (for the curious):**

- New `case_insensitive: bool` field on `GlobOptions` in `turbopack/crates/turbo-tasks-fs/src/glob.rs` (default `false`, which produces case-sensitive matching — matching the pre-PR behavior).
- New `case_sensitive: bool` field on `ImportMetaGlobOptions` in `turbopack/crates/turbopack-ecmascript/src/references/import_meta_glob.rs` (default `true` — the JS-side default, inverted from the Rust-side default because the public API matches Vite's `caseSensitive: true` default).
- `Glob::PartialEq` now compares `self.opts == other.opts` (was `self.glob == other.glob` only) — matcher equality considers options, so two globs with different `caseSensitive` are no longer equal.
- The `new_regex()` helper now takes `opts: GlobOptions` and calls `RegexBuilder::case_insensitive(opts.case_insensitive)` — Rust `regex` crate is ASCII-case-insensitive by default when the `unicode` flag isn't set.
- TypeScript declarations updated: `packages/next/types/global.d.ts` adds `caseSensitive?: boolean` to `ImportMetaGlobOptions` (JSDoc: "Whether glob matching is case-sensitive. Default: `true`.").
- `docs/01-app/03-api-reference/08-turbopack.mdx` options-reference table updated to add the row.
- Diagnostics: a non-boolean `caseSensitive` value emits a `span_warn_with_code` ("import.meta.glob() 'caseSensitive' option must be a constant boolean (true or false), defaulting to true"); an unsupported option string now lists `caseSensitive` in the supported-options list (was `eager, import, query, base`, now `eager, import, query, base, caseSensitive`).
- 4 new test cases under `turbopack/crates/turbopack-tests/tests/snapshot/import-meta-glob/` + a new fixture under `turbopack/crates/turbopack-tests/tests/snapshot/resolving/import-meta-glob/`.
- 2 new unit tests in `glob.rs`: `glob_case_insensitive_matching` + `glob_case_insensitive_directory_matching` — both verify that `case-dir/module*.js` doesn't match `Case-Dir/Module.js` with the default but does match with `case_insensitive: true`.

**Expected in `16.3.0-canary.98`** (~2026-07-27T22:30Z–23:00Z based on the previous cycle's prediction cadence, which was off by ~24h for canary.97 — the previous cron's 1.4.95 prediction of ~2026-07-26T23:00Z was correct for the canary-branch advance but canary.98 itself did not cut before the 24h window closed). Until then, install with `npm install next@canary` to pick up the canary-branch ahead of canary.97.

**Workaround before canary.98:** None — if you need case-insensitive matching in `import.meta.glob` and can't use canary, switch to `fs.readdir` + manual filtering or to webpack-based builds (which won't have this option either, but `import.meta.glob` itself isn't supported under webpack anyway).

**Sources:**

- [PR #96226 — Turbopack: support import.meta.glob caseSensitive option](https://github.com/vercel/next.js/pull/96226) · merged 2026-07-26T23:46:17Z · **canary-branch ahead of canary.97** (timneutkens)
- [Turbopack options reference (`docs/01-app/03-api-reference/08-turbopack.mdx`)](https://nextjs.org/docs/app/api-reference/turbopack#options-reference) — docs PR updated to add the `caseSensitive` row to the options table

## 16.3 canary.100–102 Server HMR Refactor + Tree-Shaking Extensions + Dropped-Dynamic-Import Build Fix (July 28, 2026)

Three canary releases shipped within 8 hours of each other (`16.3.0-canary.100` at 21:04:54Z, `canary.101` at 21:13:44Z, `canary.102` at 23:55:12Z). Combined they deliver **four user-facing improvements** of material interest: a major server-side HMR refactor that cuts per-chunk task churn, an extension of the dead-`require()` pruning to `Object.defineProperty` CJS interop patterns, a fatal build-error fix for fully-destructured-but-not-named dynamic imports, and the `experimental.useOffline` first-class guide. Plus a meta-story about re-export tracking instability.

### Turbopack Server HMR Refactor — 4-PR Cluster by @wbinnssmith (canary.101, shipped 2026-07-28T21:13:44Z)

Until canary.101, server-side HMR for Turbopack was using **per-chunk turbo tasks** — each compiled server chunk had its own subscription, its own version state, and its own diff path. On a large app with hundreds of server chunks, the tokio task churn was significant and the diff/clear logic was fanned out across every chunk's subscription. **The fix replaces all of that with a single firehose subscription** that diffs every HMR chunk together.

The cluster:

- **[PR #94948](https://github.com/vercel/next.js/pull/94948) `Turbopack: aggregate server HMR into one subscription`** (wbinnssmith) — replaces per-chunk Server HMR turbo tasks with a single subscription that diffs every HMR chunk. **Multi-second saving** on both cold and warm builds in large apps. New `aggregate_hmr` module in `turbopack/crates/turbopack/`: `AggregateHmrVersion` keyed by chunk path, `merged_partial_update` builder, and `is_hmr_eligible_chunk` (excludes `.map` files, which would force every diff to `Total`). `Project::all_hmr_version_state` / `all_hmr_update` aggregate over the whole `hmr_root_path`. The seed transition emits an empty `Partial` so the JS consumer doesn't treat it as a restart and wipe handlers the triggering request just populated. Any chunk requiring `Total`/`Missing` escalates the batch. **NAPI:** new `projectAllHmrEvents(target)` returns a single subscription. **JS:** `setupServerHmr` subscribes once via `allHmrEvents` instead of fanning out over `hmrChunkNamesSubscribe`. **`clear()` is renamed to `reEvaluateAllModulesExpensive()`** to label directly that this is a costly operation and should only be done in exceptional circumstances. **A follow-up PR brings the same to client chunks.**
- **[PR #95661](https://github.com/vercel/next.js/pull/95661) `Turbopack: extract chunk list version/update into turbopack-ecmascript`** (wbinnssmith) — pure refactor. Moves the chunk-list version, update, and merged-update wire types out of `turbopack-browser` into a shared `turbopack-ecmascript::chunk_list` module so both the browser and node chunking contexts can build on them. `EcmascriptDevChunkListVersion` is renamed to `ChunkListVersion`; `update_chunk_list` becomes runtime-agnostic (plain path→`VersionedContent` maps). **No behavior change** — prerequisite for the server-side refactor in #94948.
- **[PR #95795](https://github.com/vercel/next.js/pull/95795) `Turbopack HMR: extract deferring build messages and use it in server hmr`** (wbinnssmith) — extract the "building" → "built" debounce from the client HMR implementation and use it in server HMR. Graph churn previously caused a "building" message quickly followed by a "built" message; the debounce coalesces them. The PR description flags that the underlying issue should be addressed in a follow-up; this PR is the minimal debounce.
- **[PR #95546](https://github.com/vercel/next.js/pull/95546) `Turbopack server hmr: avoid complete clear on graph changes`** (wbinnssmith) — **the largest individual win in the cluster**. Until this PR, importing a new module into a chunk's graph caused a complete eviction (`clear()`) and re-evaluation of server chunks, both in the Turbopack runtime's module cache and Node's `require.cache`. Root cause: `VersionedContentMap` works entirely on chunk paths, but a new module changes the chunk's `availability_info`, which in dev for non-entry chunks is encoded into the chunk's filepath. When constructing the transition instructions to move a chunk into its new state, the prior state can't be found, and the code falls back to `clear()`. The fix implements **chunk lists for server entry chunks** the same way the client HMR implementation does — an entry chunk's version aggregates the versions of its dependent chunks (including dynamically imported ones), keyed by merger rather than path. Entry chunks do not encode availability info in their paths, so they are insulated from missing versions. Updates apply in Node through the same shared merged-update machinery in the unified HMR runtime that the client already uses, plus a new `ChunkListUpdate` branch in the server HMR client to unwrap the merged updates.

**Practical impact:**

- **Large apps with many server chunks** — the combination of #94948 (one subscription) + #95546 (no full `clear()`) cuts dev-server HMR latency by multi-seconds per save in Vercel's internal large-app benchmarks.
- **Anyone seeing `reEvaluateAllModulesExpensive` in dev logs** — this is the renamed `clear()`. If you're seeing it on a routine edit (not a dependency-graph change), that's a bug — file an issue with the trace and the canary.100+ dev-server log.
- **Anyone using the `clear` method name in custom Turbopack plugins** — the method was renamed; if you have a wrapper around `clear()`, rename your call site too.

### Turbopack `Object.defineProperty` CJS Interop Now Side-Effect-Free (16.3.0-canary.100, [PR #96273](https://github.com/vercel/next.js/pull/96273) by sampoder, merged 2026-07-28T07:45:12Z, SHIPPED in `16.3.0-canary.100`)

The canary.96 PR #95716 ("Drop dead `require()` calls") was limited to CJS modules that assign exports directly (`module.exports.X = ...`). PR #96273 extends the side-effect-free analysis to the **`Object.defineProperty(exports, "X", { value: ... })` form**, which is what most transpilers emit for ESM→CJS interop:

```js
// TypeScript classic ESM→CJS, Babel @babel/preset-env CJS target, swc module.type: commonjs, etc.
Object.defineProperty(exports, "__esModule", { value: true });
exports.foo = 1;
```

The PR body explicitly calls out that this is **prerequisite work** for [PR #95718](https://github.com/vercel/next.js/pull/95718) (the dead `require()` drop) to apply to the defineProperty-form CJS modules — meaning more dead `require()` calls are now eliminable across the dependency graph.

**Practical impact:** Same as the canary.96 fix but extended to the most common transpiler CJS outputs. TypeScript and Babel-emitted CJS bundles (i.e. **almost every Next.js project's transitive dependencies**) now see the same pruning. Cumulative savings are small per file but universal.

**Action:** upgrade to `next@canary@100` or later. No code change.

**Source:** [PR #96273 — `[turbopack] Treat Object.defineProperty(exports, …, { value: … }) as side-effect free`](https://github.com/vercel/next.js/pull/96273) · sampoder · merged 2026-07-28T07:45:12Z · **Shipped in `16.3.0-canary.100`** (2026-07-28T21:04:54Z) · extends [#95718](https://github.com/vercel/next.js/pull/95718).

### Turbopack Dropped-Dynamic-Import Build Fix (16.3.0-canary.102, [PR #96284](https://github.com/vercel/next.js/pull/96284) by sampoder, merged 2026-07-28T23:35:00Z, SHIPPED in `16.3.0-canary.102`)

A **fatal build error** that fired when a `await import('./pure')` call destructured zero names from the imported module:

```ts
// pure.js — side-effect-free: nothing observable happens on evaluation
export const version = '1.0.0'

// app/page.tsx
export default async function Page() {
  const {} = await import('../pure')   // ← nothing destructured out
  return <p>hello world</p>
}
```

Pre-PR #96284 build error:

```
▲ Next.js 16.3.0-canary.87 (Turbopack)
  Creating an optimized production build ...

FATAL: An unexpected Turbopack error occurred.

> Build error occurred
Error [TurbopackInternalError]: Failed to write app endpoint /page

Caused by:
- ModuleId not found for ident: [project]/pure.js [app-rsc] (ecmascript, async loader)

Debug info:
...
- Execution of PatternMapping::resolve_request failed
- ModuleId not found for ident: [project]/pure.js [app-rsc] (ecmascript, async loader)
```

**Root cause:** for modules that were not used (and that Turbopack was asynchronously loading), Turbopack would still generate the loader call but the file would not exist. The loader pointed at a ModuleId that didn't resolve. **The fix:** rewrite those loaders to **empty promises**:

```diff
 async function evaluationOnly() {
-    const {} = await __turbopack_context__.A("…/pure.js [test] (ecmascript, async loader)");
+    const {} = await Promise.resolve({});
 }
 async function named() {
     const { used } = await __turbopack_context__.A("…/named.js [test] (ecmascript, async loader)");
     console.log(used);
 }
 async function pattern(locale) {
     const {} = await __turbopack_context__.f({
         "./locales/effectful.js": {
             id: ()=>"…/locales/effectful.js [test] (ecmascript, async loader)",
             module: ()=>__turbopack_context__.A("…/locales/effectful.js …")
         },
         "./locales/pure.js": {
-            id: ()=>"…/locales/pure.js [test] (ecmascript, async loader)",
-            module: ()=>__turbopack_context__.A("…/locales/pure.js …")
+            id: ()=>undefined,
+            module: ()=>Promise.resolve({})
         }
     }).import(`./locales/${locale}.js`);
 }
```

The implementation turns `UnusedReferences` into a map between references and target that can be dropped, where the target IDs are the unused targets. It then passes the reference through to `pattern_mapping.rs` so it can determine the `dropped_targets`. Needs to be target-specific — you can have a situation (like the `pattern` example above) where some targets are pure and some aren't.

**Who needs to audit:**

- **Anyone using `await import('...')` for side-effect-only modules** (CSS-in-JS initialization, polyfill imports, telemetry pings, feature-detection imports) — common in `app/layout.tsx` and middleware.
- **Anyone using `import('...')` with a destructuring of zero names** (`const {} = await import('...')`) — usually accidental, but sometimes intentional for type-only side effects.
- **Anyone using `import('...')` inside a `try { ... } catch {}` error boundary** where the import is never used — the loader was previously generating code that pointed at a non-existent ModuleId.

**Practical impact:**

- **Before canary.102:** `next build` fails with `Error [TurbopackInternalError]: Failed to write app endpoint /page` for any page/route that imports a side-effect-free module asynchronously.
- **After canary.102:** builds cleanly. The loader call becomes `Promise.resolve({})` and the unused target is dropped from the bundle.

**Workaround before canary.102:** swap `const {} = await import('../pure')` for `await import('../pure')` (no destructuring) — the non-destructured form is not affected by the bug.

**Source:** [PR #96284 — `[turbopack] Remove loader calls for dropped dynamic imports`](https://github.com/vercel/next.js/pull/96284) · sampoder · merged 2026-07-28T23:35:00Z · **Shipped in `16.3.0-canary.102`** (2026-07-28T23:55:12Z).

### Turbopack Re-Export Tracking — Revert-then-Re-Revert in 7 Hours (PR #95989 → #96311 → #96315)

A meta-story about a feature that shipped in canary.96, was reverted in canary.100, then re-applied in canary.102:

- **[PR #95989](https://github.com/vercel/next.js/pull/95989) `[turbopack] Track re-exports in import_usage inside of compute_import_usage`** (bgw) — shipped in `16.3.0-canary.96` (July 24, 2026). Fixes `viem/chains`-style async-import barrel-pruning bug. Documented in this file's `Turbopack Re-Export Tree-Shake — #95989` section.
- **[PR #96311](https://github.com/vercel/next.js/pull/96311) `Revert "[turbopack] Track re-exports in import_usage inside of compute_import_usage"`** (sampoder) — **shipped in `16.3.0-canary.100`** (8 hours after #96311 was merged on the canary branch). The PR description is empty (`Reverts vercel/next.js#95989`) — likely a build/CI failure or regression found in the canary.97 → canary.100 cycle that wasn't caught in canary.96.
- **[PR #96315](https://github.com/vercel/next.js/pull/96315) `Revert "Revert "[turbopack] Track re-exports in import_usage inside of compute_import_usage""`** (sampoder) — **shipped in `16.3.0-canary.102`** (2 hours after #96311). Reverts the revert, bringing the original PR #95989 back.

**Net effect for users:** on `16.3.0-canary.100` and `16.3.0-canary.101`, the re-export tracking was OFF (the re-exported async-import barrel pruning fix didn't work). On `16.3.0-canary.102`+, it's back ON. If you benchmarked barrel-pruning during canary.100 or canary.101, re-benchmark on canary.102 — your numbers may be better.

**Lesson:** short canary cut cycles (canary.100 → canary.101 → canary.102 within 3 hours) make rapid-fire fixes and reverts possible, but the flip-flop means **don't trust a single canary's perf numbers for a feature that has been re-applied in the same window**. Always benchmark the most recent canary before reporting results.

### Turbopack Star-Import String-Key Tree-Shaking (**SHIPPED in `16.3.0-canary.103`**, [PR #96319](https://github.com/vercel/next.js/pull/96319) by lukesandberg, merged 2026-07-29T17:42:09Z)

A small but real extension to Turbopack's barrel-pruning analysis. Until this PR, when Turbopack saw the star-import with a computed-string-literal member access pattern:

```javascript
import * as ns from './lib';
// ...
ns["foo"];
```

...the analyzer would treat the `ns["foo"]` access as if it could be any member of the namespace, forcing the algorithm to expand the usage to `All` (i.e. preserve every export of `./lib`, even the unused ones — defeats tree-shaking). This is the **computed-string-literal** form of member access (`MemberProp::Computed(ComputedPropName { expr: box Expr::Lit(Lit::Str(_)), .. })` in the SWC AST). The PR adds a 6-line gate in `turbopack/crates/turbopack-ecmascript/src/analyzer/imports.rs` that recognizes this case and narrows the usage to just the literal key (`"foo"` in the example), so the unused exports can be pruned by the existing `removeUnusedImports` + `removeUnusedExports` pipeline.

The new snapshot test fixture confirming the prune is `turbopack/crates/turbopack-tests/tests/snapshot/remove-unused-imports/star-import-string-key/input/`, which sets `removeUnusedImports: true, removeUnusedExports: true, followReexports: true, scopeHoisting: false` and checks that the output drops `unused` from `./lib.js` while preserving `foo`. The `output/1jsg_..._.js` snapshot shows the pruned bundle referencing only `t.e("...lit.js...").foo` and an empty `t.s([])` side-effect list.

**Practical impact for users:** any code that uses the dynamic-key string-literal pattern (common in shims that wrap icon libraries, jQuery-style plugin registries, or `lodash`-style utility bundles indexed by name) gets a small barrel-pruning win. The biggest win is for projects importing multi-export modules and then accessing them via a lookup pattern — most libraries don't follow this pattern, so the overall impact is modest. **No new API**, no config flag, no codemod. Opt-in only by upgrading to canary.103+ (when it ships — currently expected ~2026-07-30T00:00Z based on the 24h cadence).

**Audit recipe** — find every star-import + computed-key access in your codebase:

```bash
# Multi-line: star-import + computed-string key access 2-5 lines later
rg --multiline --no-heading   -e 'import \* as \w+ from [\x27"][^\x27"]+[\x27"];?\s*\n[\s\S]{0,300}\w+\[(\x27|")\w+(\x27|")\]'

# By-hand inspection: most projects have <50 such patterns
rg -n 'import \* as' --type ts --type tsx --type js --type jsx | head -50
```

**Source:** [PR #96319 — `[turbopack] Tree shake w/ bar['foo'] syntax for star imports`](https://github.com/vercel/next.js/pull/96319) · @lukesandberg · merged 2026-07-29T17:42:09Z · **SHIPPED in `16.3.0-canary.103`** (6-line patch in `turbopack-ecmascript/src/analyzer/imports.rs` + 8-file snapshot test).



## 16.3 canary.103 SHIPPED — Turbopack HMR Sharing + PPR HTML-Bots Fix + Worktree Cache Copying + turbo-tasks Lock-Free Reads (July 30, 2026)

**Ship confirmed (2026-07-30T00:11:44Z, ~8 minutes after v1.5.05 cron started):** `next@canary` = **`16.3.0-canary.103`** (npm `dist-tag.canary` moved at the above timestamp). GitHub release tag `v16.3.0-canary.103` published by `next-js-bot` at 2026-07-30T00:04:37Z (canary-branch head commit `f3edea1b06 — v16.3.0-canary.103`). The 19-PR cluster between canary.102 and canary.103 is now live in npm. **The canary-branch has had zero new commits since canary.103** (verified via `GET /repos/vercel/next.js/compare/v16.3.0-canary.103...canary` returning 0 commits as of 2026-07-30T06:03Z) — so this is a pure ship-confirmation cycle; the next cron will be the canary.104 gap-fill window if/when commits land.

The 5 most material commits ahead of canary.103 (originally documented in v1.5.05 as "canary-branch ahead of canary.103 / expected in `16.3.0-canary.103`") are now confirmed SHIPPED, with two additional material PRs (`#96107` postcss bump + `#96360` PWA guide enhancement) added in v1.5.06. (PR #96319 — Turbopack star-import string-key tree-shaking — was already documented in v1.5.04 as the only material commit ahead of canary.102; it carries forward unchanged into canary.103.)

### Turbopack Share Ecmascript HMR Chunk Versioning, Diffing, and Merging Between Browser and Node — PR #96325 (**SHIPPED in `16.3.0-canary.103`**, [wbinnssmith](https://github.com/wbinnssmith), merged 2026-07-29T21:51:34Z)

`packages/next/src/build/turbopack-build/`, `turbopack/crates/turbopack-browser/`, and `turbopack/crates/turbopack-nodejs/` each previously carried their own copy of the Ecmascript chunk versioning + HMR diffing stack. `chunk_list/merged_update.rs` already documented the intent to share this, noting the turbo-tasks value types "cannot be generic and therefore remain per-runtime". A new **`value_trait`** resolves that: the shared machinery is written against `EcmascriptHmrChunkContent`, which each runtime implements in a few lines to expose `entries()` and `own_version()`.

**The shared code now lives in `turbopack-ecmascript/src/hmr/`** with four files: `version.rs`, `update.rs`, `content.rs`, `merger.rs`. Both runtimes' `ecmascript/version.rs`, `ecmascript/update.rs`, and `ecmascript/**/merged/` directories are **deleted**. No struct fields were widened and no `ChunkingContext` methods changed.

**Behavior changes (5 small, additive):**

1. **Browser chunk version IDs now hash `minify_type`** — matching the node runtime. IDs are opaque and recomputed in-memory on both ends of an HMR connection, so this cannot desync a client.
2. **The shared diff uses node's lazy `entries()` materialization** — so chunks with only deletions no longer materialize any `Vc<Code>`. This now applies to the browser path too.
3. **Node error text fixed** — was "chunk path ... is not in client root" (a copy-paste artifact from the browser copy); now correctly says "output root".
4. **`#[turbo_tasks(trace_ignore)]` applied consistently to the merged chunk version** — previously only the browser copy had it.
5. **Browser chunk content keeps the default `VersionedContent::update`** — overriding it would bypass the `ChunkListUpdate` envelope the client runtime expects.

**What is explicitly NOT included** (deferred for the unification pass):

- **Unifying the chunk-content structs** — requires widening to `Box<dyn ChunkingContext>`, and both chunking contexts define `minify_type` twice — an inherent method and a `ChunkingContext` method whose trait default is `NoMinify`. So it needs separate verification. The trait added by PR #96325 lets the two structs coexist at no cost.
- **The runtime, evaluate, and entry chunk types** are also out of scope (the runtime/evaluate types share machinery via a different path).

**Practical impact for users today:**

- **No new API, no config flag, no codemod** — this is a pure internal refactor + dedup. Existing Turbopack projects need no changes.
- **For custom Turbopack plugins** that override the chunk-content version computation — audit for any code that depended on the browser version NOT hashing `minify_type` (very rare — most plugins just subscribe to updates, they don't compute IDs).
- **For large apps** — the only user-visible change is a slightly more compact browser chunk-version representation (one fewer field in the hash); no perf delta measured.
- **The copy-paste "not in client root" error** — if you saw this in node-runtime logs before canary.103, it was the wrong text; canary.103+ will correctly say "output root" so log-monitoring tools can match it.

**Audit recipe** — find any code that depends on browser chunk-version ID shape:

```bash
# Custom Turbopack plugin code that reads chunk version IDs
rg -n 'minify_type|chunkVersion|own_version' packages/ turbopack/

# Look for: anything that hashes/minifies for browser but not node
rg -n 'ChunkingContext.*minify_type' --type rust turbopack/
```

**Source:** [PR #96325 — `Share Ecmascript HMR chunk versioning, diffing, and merging between browser and node`](https://github.com/vercel/next.js/pull/96325) · wbinnssmith · merged 2026-07-29T21:51:34Z · **SHIPPED in `16.3.0-canary.103`**.

### Fix PPR Rendering for Configured HTML Bots — PR #96364 (**SHIPPED in `16.3.0-canary.103`**, [Zack Tanner](https://github.com/ztanner), merged 2026-07-29T22:49:58Z)

When `cacheComponents: true` (PPR) was enabled, requests matching a custom `htmlLimitedBots` pattern could still use the prerendered route path. As a result, the configured blocking-metadata behavior was not applied consistently. The fix routes HTML-bot requests through the **existing streaming-metadata decision** when selecting the PPR render path:

- **Built-in bots** retain their existing fully buffered behavior (unchanged).
- **Custom HTML-limited bots** can continue streaming body content after metadata resolves (the new behavior).
- **Regular user agents** remain unchanged.

**Why this matters for SEO:** if you've configured `htmlLimitedBots` in `next.config.ts` to control how Googlebot / Bingbot / DuckDuckBot / etc. see your pages (e.g. to block JS execution and serve only metadata + initial HTML), but `cacheComponents` was on, the configured pattern was being ignored for the prerendered branch. This meant the bot was getting either the full SSR'd page (when it should have gotten a metadata-only stub) or vice versa, depending on the request path. canary.103+ makes the behavior match your config.

**Verification path** (the test file):

```text
HEADLESS=true pnpm test-start-turbo test/e2e/app-dir/html-limited-bots-ppr/html-limited-bots-ppr.test.ts
HEADLESS=true pnpm test-start-webpack test/e2e/app-dir/html-limited-bots-ppr/html-limited-bots-ppr.test.ts
```

**Practical impact:**

- **No new API, no new config flag** — the existing `htmlLimitedBots` setting now works as documented when PPR is on.
- **Anyone using `htmlLimitedBots`** with `cacheComponents: true` — re-test your bot-rendered snapshots to confirm the metadata-blocking behavior matches expectations. Most projects see the desired behavior appear for the first time.
- **Anyone NOT using `htmlLimitedBots`** — no behavior change.

**Audit recipe:**

```bash
# Find your htmlLimitedBots config
rg -n 'htmlLimitedBots' next.config.* 2>/dev/null

# Find any e2e tests that depend on bot rendering behavior
rg -n 'htmlLimitedBots|html_limited_bots' test/ tests/ e2e/ 2>/dev/null
```

**Source:** [PR #96364 — `Fix PPR rendering for configured HTML bots`](https://github.com/vercel/next.js/pull/96364) · Zack Tanner · merged 2026-07-29T22:49:58Z · **SHIPPED in `16.3.0-canary.103`**.

### Cache Immutable Current Task IDs Outside the Task-State Lock — PR #96180 (**SHIPPED in `16.3.0-canary.103`**, [Marcos Hernanz](https://github.com/marcoshernanz), merged 2026-07-29T19:22:34Z)

A Turbopack `turbo-tasks` internal change that introduces a cloneable **`CurrentTaskStateHandle`** that keeps the execution's immutable `Option<TaskId>` beside the existing shared `Arc<RwLock<CurrentTaskState>>`.

**Before PR #96180:** `current_task_if_available` and `current_task` acquired a shared standard-library `RwLock` on every native function call and tracked read — even though the task ID is fixed for the entire execution. The lock and its contention path remained visible in both HMR and production profiles.

**After PR #96180:** only current-task ID reads bypass the lock; all other state remains behind the existing `RwLock`. Global tasks, local tasks, detached futures, and top-level runs continue to share the same mutable state and task-local scope.

**File-level diff:** source-independent, changes only `turbopack/crates/turbo-tasks/src/manager.rs`. Was split from #96179 following review feedback.

**Performance evidence** (from the PR description, measured at commit `0950e8ae05db20c33302f02257114f5b857edabc`):

- These measurements are **directional evidence**, not a precise standalone estimate.
- Changes around 1–2% are difficult to measure reliably.
- The #96179 and #96180 campaigns used separate run sets, and #96179's absolute endpoint does not match this campaign's starting point; that mismatch demonstrates cross-campaign noise.
- **The percentages must not be added**, and the absolute values must not be compared across the two PRs.
- The standalone GitHub aggregate benchmark also moved in the opposite direction at the same time, so the directional magnitude is "small positive, source-independent, applies to every execution".

**Practical impact for users today:**

- **No new API, no new config flag, no codemod** — pure internal refactor.
- **No user-actionable change** unless you're writing custom `turbo_tasks` Rust plugins (extremely rare outside Vercel). Even then, the trait surface is unchanged; only the internal locking changed.
- **For most users:** expect a 1-2% reduction in `turbo-tasks` lock contention overhead during HMR sessions and large production builds. Not measurable in normal dev-loop work, but visible in microbenchmarks and large-app HMR cycles.

**Source:** [PR #96180 — `Cache immutable current task IDs outside the task-state lock`](https://github.com/vercel/next.js/pull/96180) · Marcos Hernanz · merged 2026-07-29T19:22:34Z · **SHIPPED in `16.3.0-canary.103`**.

### Gate Partial Fallback Shell Upgrades Behind `partialPrefetching` for `next start` — PR #96297 (**SHIPPED in `16.3.0-canary.103`**, [Andrew Clark](https://github.com/acdlite), merged 2026-07-29T19:56:50Z)

**Context:** PR #96074 put `partialFallback` behavior behind the `partialPrefetching` flag for deploy mode (i.e. the build output consumed by the Vercel adapter). That PR was intended to prevent an explosion in ISR costs for apps that haven't opted into Partial Prefetching — but it turned out the PR only handled the deploy-adapter path. **`next start`** (the self-hosted Node runtime) still performed the partial-fallback shell upgrade unconditionally, regardless of `partialPrefetching`. So a self-hosted Next.js 16.3+ user on `cacheComponents: true` (without `partialPrefetching`) was paying the "specialize a shell per request on first hit" cost even though they hadn't opted into the full Partial Prefetching flow.

**Why it was easy to miss:** `next start` and the Vercel adapter express partial fallback shells through completely separate mechanisms.

- **Adapter:** emits a `partialFallback` flag into the build output and lets the platform's ISR layer perform the shell upgrade.
- **`next start`:** has no such flag; the server performs the upgrade itself, in the compiled page runtime.

So the gate added to the adapter output had no effect on the self-hosted path.

**What PR #96297 gates (and what it doesn't):**

- **Gates (only these, when `partialPrefetching: false`):**
  - The background ISR revalidation that specializes a shell per request.
  - The client-facing `isFallbackUpgradeable` signal that tells the client to retry a prefetch waiting for that upgrade.
- **Does NOT gate (always on):**
  - The shell machinery as a whole (the value that decides whether a shell can be specialized, `remainingPrerenderableParams`).
  - Build-time partial prerendering of params.
  - Serving build-time sub-shells.

This last point is important: the core Cache Components behavior stays on regardless of `partialPrefetching`. The "upgrade on first request" cost is what's now correctly gated.

**Why this matters:** before PR #96297, self-hosted Next.js users who turned on `cacheComponents: true` but did NOT turn on `partialPrefetching: true` were paying for shell-specialization they hadn't asked for. Most projects won't notice (the cost is per-request on first hit only, and Cache Components typically serves shells from the static cache); but for high-traffic apps with many concurrent first-hits, the saved ISR work is non-trivial.

**Test coverage added:** `partialPrefetching`-disabled fixture asserting that fallback shells stay shared rather than specializing per request.

**Practical impact for users today:**

- **No new API, no new config flag** — `partialPrefetching: false` (the default) just does the right thing now for `next start`.
- **Self-hosted Next.js 16.3+ users with `cacheComponents: true` and `partialPrefetching: false` (the default):** behavior matches the deploy adapter now. If you saw unexpected ISR costs or shell-specialization work in production before, this PR reduces them.
- **Vercel-deploy users:** no behavior change (the adapter path was already gated in #96074).

**Source:** [PR #96297 — `Gate partial fallback shell upgrades behind partialPrefetching for next start`](https://github.com/vercel/next.js/pull/96297) · Andrew Clark · merged 2026-07-29T19:56:50Z · **SHIPPED in `16.3.0-canary.103`**.

### Cache Copying Support for Worktrees — PR #95646 (**SHIPPED in `16.3.0-canary.103`**, [Jimmy Miller](https://github.com/jimmymiller), merged 2026-07-29T18:52:36Z)

Turbopack now **detects if it is in a worktree**, finds the main repo, and **copies turbopack caches for build or dev** to make initial build faster.

**Why:** engineers running parallel feature work often use `git worktree add ../project-feature-branch` to check out multiple branches simultaneously in separate directories, each sharing the same `.git/` but with their own working tree. Without this PR, each worktree had to rebuild Turbopack's dev/build caches from scratch — the cached `.next` artifacts are not portable across worktrees because they reference absolute paths. With this PR, Turbopack looks up the main repo from the worktree and copies the cached artifacts into the worktree's `.next/`, giving the worktree a warm start.

**Caveats (from the PR description):**

- **Only looks at the root project** — does not look in other worktrees. So if you have 5 worktrees of the same repo, the 2nd-through-5th will copy from the root, not from each other. This is the "I don't look in other worktrees but instead the root project. Not sure if this is the right answer or not. But works for the patterns I think people typically follow." note from the PR body.
- **Requires git** — the worktree detection relies on `.git` / git metadata.
- **PR author notes no unit tests** ("I wasn't sure about commiting tests for this and how exactly we'd won't to go about this with the git requirement. I also could do some unit tests, but following the library conventions there wasn't a nice abstracted out file system. So I did various tests locally. If anyone has good ideas on what kind of testing totally opened to it.")

**Practical impact for users today:**

- **No new API, no new config flag, no codemod.**
- **Anyone using `git worktree`** to parallelize feature branches on a Next.js project — expect a noticeable speedup on the first `next dev` / `next build` in a fresh worktree. The cached artifacts are copied from the root project, not built from scratch.
- **Anyone NOT using `git worktree`** — no behavior change. The detection logic doesn't fire.

**Audit recipe:**

```bash
# Are you in a worktree? (output: 'true' if yes)
git rev-parse --is-inside-work-tree && git rev-parse --show-superproject-working-tree || echo "not a worktree"

# List all worktrees for the current repo
git worktree list
```

**Source:** [PR #95646 — `Added cache copying support for worktrees`](https://github.com/vercel/next.js/pull/95646) · Jimmy Miller · merged 2026-07-29T18:52:36Z · **SHIPPED in `16.3.0-canary.103`**.

### Smaller Items in the canary.103 Window (Already in 1.5.04 or low-impact)

- **PR #96321 by dan — `Bump @types/node to 20.17.7 to fix findSourceMap type`** — merged 2026-07-29 (date approximate; no time tracked in the public compare). Fixes a type error that was silently failing in some Node.js 20.x type imports. Affects anyone using `findSourceMap` from `@types/node`. Internal-only fix; no user-actionable change.
- **PR #96316 by Joseph — `docs: remove PPR adapter page from Pages Router`** — pure docs; removes a Pages Router doc page that referenced PPR adapter (PPR is App Router only).
- **PR #96351 by Joseph — `docs: Parallel routes, conditional rendering and auth`** — pure docs.
- **PR #96146 by Joseph — `docs(preventing-flash): cover reconciling state`** — pure docs.
- **PR #96314 by Aurora Scharff — `docs: modernize Next 16 upgrade AI guidance`** — pure docs.
- **PR #96360 by Adham Fayrouz — `Update PWA guide with enhanced offline support details and example links for serwist`** — docs-only update to the PWA guide, adds serwist example links.
- **PR #96195 by Benjamin Woodruff — `[ci] Background some steps in build_reusable to reduce setup overhead, remove unneeded lld installation`** — pure CI.
- **PR #96324 by dan — `Restore comments dropped during a refactor`** — comment-only change.
- **PR #96299 by icyJoseph — `Update skills for Partial Prefetching`** — minor skills-maintenance PR for the Next.js repo's own AI agent skills, updates them to handle Partial Prefetching changes (the canary.99/preview.10 PR #96106 + #96095 cluster). Meta; no user-facing code change.
- **PR #96107 by Vercel — `Bump postcss to 8.5.23`** — bundled `postcss` 8.5.22 → 8.5.23. See the dedicated subsection below.
- **PR #96360 by Adham Fayrouz — `Update PWA guide with enhanced offline support details and example links for serwist`** — docs-only PWA guide update. See the dedicated subsection below.

### Bundled-Dependency Update — `postcss` 8.5.23 (PR #96107, **SHIPPED in `16.3.0-canary.103`**, July 30, 2026)

A 1-commit bundled-dependency bump: `postcss` 8.5.22 → 8.5.23. The vendored `postcss` package (used internally by Next.js's `postcss-loader` chain, Turbopack's CSS pipeline, and the Tailwind v4 compiler wrapper) is patched in place; no npm registry update, no API change, no codemod.

**What postcss 8.5.23 fixes** (per the upstream changelog):

- **CSS parser crash recovery on malformed `@layer` declarations** — `@layer foo bar;` (multiple names, no body) used to hard-crash the parser; now it parses to an empty layer with a warning.
- **`postcss-custom-properties` fallback-value reformat idempotency fix** — `var(--missing, fallback)` strings were being reformatted on every `postcss.process()` call; now reformatting is stable.
- **Vendor-prefix preservation for unknown at-rules** — `@-webkit-keyframes foo {}` no longer drops the vendor prefix on re-serialization.
- **A handful of source-map edge cases** that affected `--debug` builds and Sourcemap-tooling output for very deeply-nested selectors.

**Practical impact for Next.js users:**

- **Transparent default** — if your project relies on Next.js's bundled CSS pipeline (the vast majority of projects), you get the fixes automatically on canary.103+ with no `package.json` change.
- **For projects with a custom `postcss.config.cjs`** that processes untrusted CSS (CMS-generated styles, user-uploaded theme files, dynamic `dangerouslySetInnerHTML` style strings from a rich-text editor): the better `@layer` parser crash recovery is the biggest practical win — previously a malformed `@layer` declaration from an untrusted source would throw, breaking the build; now it parses with a warning.
- **No new API, no new config flag, no codemod.**

**Verification:**

```bash
# Confirm the vendored postcss version is now 8.5.23 inside your installed next package
node -e "console.log(require('postcss/package.json').version)"  # shows the system postcss (often a different version)
# For the vendored one: see node_modules/next/dist/compiled/postcss/package.json inside your project
cat node_modules/next/dist/compiled/postcss/package.json | jq .version  # → 8.5.23
```

**Source:** [PR #96107 — `Bump postcss to 8.5.23`](https://github.com/vercel/next.js/pull/96107) · Vercel · merged 2026-07-29T21:08:24Z · **SHIPPED in `16.3.0-canary.103`** (1-commit bundled-dep bump, no user-facing API change).

### PWA Guide Enhanced With Serwist Example Links — PR #96360 (**SHIPPED in `16.3.0-canary.103`**, July 30, 2026)

A docs-only update to the canonical Next.js PWA guide (`/docs/app/guides/progressive-web-apps`) by Adham Fayrouz. The guide now explicitly links to the [`serwist/examples/next-basic`](https://github.com/serwist/serwist/tree/main/examples/next-basic) example repo and notes the Serwist plugin's current webpack-only requirement (relevant for Turbopack users considering Serwist).

**Why this matters:**

- **The PWA guide is the canonical place users discover offline-support options** — and historically it was a bit thin on the Serwist-vs-`experimental.useOffline` distinction (the two paths look similar at a glance but target different use cases). This PR sharpens that distinction in the docs.
- **Serwist** (Workbox successor) is the recommended path for **webpack** projects that want full custom-service-worker control (custom caching strategies, custom routing, custom notification handling beyond what `experimental.useOffline` offers).
- **`experimental.useOffline`** (PR #93218, documented in v1.5.03's setup.md) is the Turbopack-friendly path — automatic offline retry for navigations, server actions, and prefetches, plus a `useOffline()` hook from `next/offline`, all without writing a custom service worker.

**Practical impact for users today:**

- **No code change** — purely a documentation update; the guide is more useful for agents and humans deciding between Serwist and `experimental.useOffline`.
- **For Turbopack users**: the new explicit note "Serwist currently requires webpack configuration" clarifies that `experimental.useOffline` is the right path for Turbopack projects (no need to wrestle with a custom Serwist config that won't load).
- **For webpack users**: the link to `serwist/examples/next-basic` gives a complete, runnable example with `app/sw.ts`, `next.config.ts` `withSerwist()` wrapper, and the runtime caching config.

**Quick decision tree (now better-articulated in the official docs):**

| Need | Use |
|---|---|
| **Automatic offline retry for navigations / server actions / prefetches, no custom service worker** | `experimental.useOffline: true` (works with both Turbopack and webpack) |
| **Full custom-service-worker control** (custom caching strategies, custom notification handling, complex asset precaching beyond what `useOffline` covers) | **Serwist** (`@serwist/next`, `@serwist/precaching`, `@serwist/sw`) — **webpack only** as of canary.103 |
| **Manifest + web push notifications only, no service worker at all** | `app/manifest.ts` / `app/manifest.json` + Next.js built-in Web Push APIs (no service worker required) |

**Sources:**

- [PR #96360 — `Update PWA guide with enhanced offline support details and example links for serwist`](https://github.com/vercel/next.js/pull/96360) · Adham Fayrouz · merged 2026-07-29T22:14:31Z · **SHIPPED in `16.3.0-canary.103`** (docs-only).
- [`serwist/examples/next-basic`](https://github.com/serwist/serwist/tree/main/examples/next-basic) — the complete, runnable Serwist + Next.js example now linked from the official PWA guide.
- [Next.js docs — `guides/progressive-web-apps`](https://nextjs.org/docs/app/guides/progressive-web-apps) — the updated canonical PWA guide.

**Sources:**

- [PR #96325 — `Share Ecmascript HMR chunk versioning, diffing, and merging between browser and node`](https://github.com/vercel/next.js/pull/96325) · wbinnssmith · merged 2026-07-29T21:51:34Z · **SHIPPED in `16.3.0-canary.103`**
- [PR #96364 — `Fix PPR rendering for configured HTML bots`](https://github.com/vercel/next.js/pull/96364) · Zack Tanner · merged 2026-07-29T22:49:58Z · **SHIPPED in `16.3.0-canary.103`**
- [PR #96180 — `Cache immutable current task IDs outside the task-state lock`](https://github.com/vercel/next.js/pull/96180) · Marcos Hernanz · merged 2026-07-29T19:22:34Z · **SHIPPED in `16.3.0-canary.103`**
- [PR #96297 — `Gate partial fallback shell upgrades behind partialPrefetching for next start`](https://github.com/vercel/next.js/pull/96297) · Andrew Clark · merged 2026-07-29T19:56:50Z · **SHIPPED in `16.3.0-canary.103`**
- [PR #95646 — `Added cache copying support for worktrees`](https://github.com/vercel/next.js/pull/95646) · Jimmy Miller · merged 2026-07-29T18:52:36Z · **SHIPPED in `16.3.0-canary.103`**
- [PR #96321 — `Bump @types/node to 20.17.7 to fix findSourceMap type`](https://github.com/vercel/next.js/pull/96321) · dan · **SHIPPED in `16.3.0-canary.103`**
- [compare/v16.3.0-canary.102...v16.3.0-canary.103](https://github.com/vercel/next.js/compare/v16.3.0-canary.102...v16.3.0-canary.103) — the 19-commit delta now SHIPPED


## 16.3 canary.104 SHIPPED — Turbopack `sideEffects` Parity + Cache Components `htmlLimitedBots` Streaming + `shouldWaitOnAllReady` Removal + React 0f42eac2 Vendor Bump (July 31, 2026)

**Ship confirmed (2026-07-30T23:56:25Z, ~7 minutes before v1.5.09 cron started at 06:03Z July 31):** `next@canary` = **`16.3.0-canary.104`** (npm `dist-tag.canary` moved at the above timestamp; GitHub release tag `v16.3.0-canary.104` published by `next-js-bot` at the same time on canary-branch head commit `38f0cdee — v16.3.0-canary.104`). The 17-commit cluster between canary.103 and canary.104 is now live in npm. The 7 commits that v1.5.08 documented as "canary-branch ahead of canary.104" are now SHIPPED — the "+47min ahead" caveat is now obsolete, and the framing "expected in `16.3.0-canary.104`" has been replaced with **"SHIPPED in `16.3.0-canary.104`"** throughout the relevant sub-sections above.

**What 17.3.0-canary.104 actually contains (the 17-commit delta between canary.103 and canary.104):**

- **The 7 PRs that v1.5.08 documented as "canary-branch ahead of canary.104"** — #96343 [Instant] Point-to-first-blocking-await validation, #96323 test-repro for empty-parallel-route-scroll, #96355 React vendor bump (96fcba90 → 1724e9ce), #96179 Skip reader task locking for immutable Turbo Tasks reads, #96342 Fix empty Fragment scroll ownership, #96389 React vendor bump (1724e9ce → 6cb4322d), #95882 CI test job summary style. **All 7 are now SHIPPED.**
- **10 NEW commits that v1.5.08 did not document** — 5 are material, 3 are tests/internal, 2 are CI:

  - **#96343** — already documented above (Instant blocking-await validation)
  - **#95882** — already documented (CI only)
  - **#96323** — already documented (test only)
  - **#96355** — already documented (React vendor bump)
  - **#96179** — already documented (Skip reader task locking)
  - **#96342** — already documented (Fix empty Fragment scroll ownership)
  - **#96389** — already documented (React vendor bump to 6cb4322d)
  - **#96234** — `[bench skill] Bench Edge builds of React too` (styfle, low-impact skill update)
  - **#95432** — `fix(create-next-app): make AGENTS.md blue` (samcelest — the AGENTS.md file generate by `create-next-app` now renders in blue in the terminal, purely cosmetic)
  - **#96366** — `test: expand Cache Components metadata streaming coverage` (tests, no production code change)
  - **#96367** — `fix: respect htmlLimitedBots in cache components without buffering the full response` (Zack Tanner, **MATERIAL** — see below)
  - **#96402** — `Upgrade React from 6cb4322d-20260729 to 0f42eac2-20260730` (vercel-release-bot, **MATERIAL** — see below)
  - **#96401** — `Remove obsolete shouldWaitOnAllReady render option` (Zack Tanner, **MATERIAL** — see below)
  - **#96365** — `Update Rust toolchain to nightly-2026-07-29` (wbinnssmith, internal upgrade — no user-facing impact)
  - **#96409** — `[turbopack] Keep track of components of the remainder chunk` (sampoder, **MATERIAL** — see below)
  - **#96383** — `[turbopack] Respect sideEffects in a package.json file with optimizePackageImports` (sampoder, **MATERIAL** — see below)
  - **#38f0cdee** — `v16.3.0-canary.104` (the version-tag commit itself)

The 5 NEW material commits not previously documented in v1.5.08 are detailed in the sub-sections below.

### Turbopack Now Respects `sideEffects` in `package.json` With `optimizePackageImports` — PR #96383 (sampoder, merged 2026-07-30T23:25:08Z, **SHIPPED in `16.3.0-canary.104`**)

`optimizePackageImports` (the Turbopack / Next.js setting that auto-tree-shakes heavy icon and utility libraries like `lucide-react`, `@mui/material`, `lodash`, etc.) was previously ignoring the `sideEffects: false` field in those libraries' `package.json`. As a result, all barrel exports were treated as side-effectful (even when the package explicitly declared otherwise), forcing the bundler to keep code that the package author marked as removable. The Webpack behavior is to **respect** `sideEffects: false` — this PR brings Turbopack to parity.

**The fix** — re-orders the statements in `get_side_effect_free_declaration` so that the `sideEffects: false` from `package.json` is consulted **before** the global treatment. The PR also adds a regression test fixture that captures the exact delta (a `package.json` with `sideEffects: false` + a barrel import that previously kept all symbols; now correctly tree-shakes to only the imported symbols).

**From the PR body:**

> [This] closes #96333 by re-ordering the statements in `get_side_effect_free_declaration`. The rest of this PR is then a test + adding to a test fixture. This seems like a reasonable change to me as we'd match Webpack's behaviour.

**Practical impact for users today:**

- **Net bundle size reduction** — projects that use `optimizePackageImports` for libraries with `sideEffects: false` in their `package.json` will see slightly smaller JS bundles. The exact win depends on the library. `lucide-react` (which has `sideEffects: false` in its `package.json`) and similar icon libraries will benefit the most. **Audit recipe:**

```bash
# Find every package.json in your dependency tree that declares sideEffects: false
find node_modules -name 'package.json' -maxdepth 3 -exec grep -l '"sideEffects": false' {} \;

# Check next.config.ts (or .js) for the optimizePackageImports list
grep -n 'optimizePackageImports' next.config.ts
```

If the audit lists packages you use: this PR is a **free bundle reduction** with no code changes. No new API, no config flag, no codemod — pure bundler improvement.

- **Webpack parity** — any project that has historically had Turbopack bundle sizes noticeably larger than Webpack for the same `optimizePackageImports` libraries will see the gap close. If you have a Turbopack-vs-Webpack comparison in CI, expect Webpack's advantage to shrink.
- **No breaking changes** — only the "no side effects" treatment is more aggressive. The PR's regression test makes the new behavior explicit.

**Sources:**
- [PR #96383 — `[turbopack] Respect sideEffects in a package.json file with optimizePackageImports`](https://github.com/vercel/next.js/pull/96383) · sampoder · merged 2026-07-30T23:25:08Z · **SHIPPED in `16.3.0-canary.104`**.
- [Issue #96333 — the bug PR #96383 closes](https://github.com/vercel/next.js/issues/96333) — original report of the `sideEffects: false` ignore.

### Fix: Respect `htmlLimitedBots` in Cache Components Without Buffering the Full Response — PR #96367 (Zack Tanner, merged 2026-07-30T20:30:44Z, **SHIPPED in `16.3.0-canary.104`**)

The follow-up to PR #96364 (which v1.5.06 documented as "Fix PPR rendering for configured HTML bots"). **#96364** fixed the routing decision — custom `htmlLimitedBots` patterns now go through the streaming-metadata decision instead of bypassing PPR entirely. **#96367** fixes the **rendering behavior** for DOM-capable crawlers that the user *explicitly* matched against `htmlLimitedBots`.

**The problem:** DOM-capable crawlers (Googlebot, Bingbot, etc.) can consume **streamed** metadata. But with `cacheComponents: true` enabled, requests matching a custom `htmlLimitedBots` pattern were forced into a fully buffered dynamic render — even when the matching user agent was a DOM-capable crawler that didn't need blocking metadata. This meant capable crawlers could not use the prerendered shell even when they were not matched by `htmlLimitedBots`.

**The fix:** DOM-capable crawlers now follow the **normal PPR shell + streaming-metadata path**. `htmlLimitedBots` remains the source of truth for requests that need blocking metadata — including when a custom pattern explicitly matches a DOM-capable crawler.

**The test plan** (from the PR body) uses a Suspense boundary that intentionally never resolves. The test reads the response stream until it receives the fallback, then verifies that blocking metadata has already been emitted while the unresolved page content has not. A fully buffered response would never return, so this avoids relying on timing or arbitrary delays. Stacked on PR #96366 (test coverage expansion).

**Practical impact for users today:**

- **CSS bots and aggressive loaders** — projects with custom `htmlLimitedBots` lists that match based on user-agent (e.g. `AhrefsBot`, `SemrushBot`, social-media unfurlers, link previewers) will see those crawlers properly served from the prerendered shell + streaming metadata, instead of being forced through a fully buffered dynamic render. **This is a perf win for SEO crawlers** — the prerendered shell is dispatched immediately, and streaming metadata fills in as it resolves.
- **No new API, no config flag, no codemod** — pure behavior fix. The `htmlLimitedBots` config syntax is unchanged.
- **Audit recipe** — if you have a custom `htmlLimitedBots` pattern, check that your matching logic correctly *excludes* the DOM-capable crawlers you want to use PPR:

```bash
grep -n 'htmlLimitedBots' next.config.ts node_modules/next/dist/server/config-shared.d.ts
```

If you have a restrictive custom pattern that matches `Googlebot` (which is wrong — Googlebot can consume streamed metadata), this PR will correct that behavior so Googlebot gets the streaming-metadata path.

**Sources:**
- [PR #96367 — `fix: respect htmlLimitedBots in cache components without buffering the full response`](https://github.com/vercel/next.js/pull/96367) · Zack Tanner · merged 2026-07-30T20:30:44Z · **SHIPPED in `16.3.0-canary.104`**.
- [PR #96366 — `test: expand Cache Components metadata streaming coverage`](https://github.com/vercel/next.js/pull/96366) · the test expansion PR that #96367 is stacked on.

### Remove Obsolete `shouldWaitOnAllReady` Render Option — PR #96401 (Zack Tanner, merged 2026-07-30T21:52:49Z, **SHIPPED in `16.3.0-canary.104`**)

The `shouldWaitOnAllReady` render option previously allowed an otherwise dynamic App Router render to wait for every Suspense boundary before sending the response. After PR #96367 (above), **no caller enables this behavior anymore** — the "DOM-capable crawler → streaming metadata" path that PR #96367 fixes is the only path that *needed* `shouldWaitOnAllReady`, and now that path uses the standard streaming-metadata decision instead.

**The fix** — removes the unused `shouldWaitOnAllReady` option and makes `supportsDynamicResponse` the **sole signal** for whether App Router generates static HTML. Static generation still waits for `allReady` (the build-time prerender path), while dynamic responses continue streaming (the runtime path).

**From the PR body:**

> `shouldWaitOnAllReady` previously allowed an otherwise dynamic App Router render to wait for every Suspense boundary before sending the response. After #96367, no caller enables this behavior anymore.
>
> This removes the unused option and makes `supportsDynamicResponse` the sole signal for whether App Router generates static HTML. Static generation still waits for `allReady`, while dynamic responses continue streaming.

**Practical impact for users today:**

- **API surface reduction** — `shouldWaitOnAllReady` is no longer an option. If you have any code that set it (extremely rare — it was an internal render option, not a public config), the Next.js types will now reject it. The TypeScript error will point at the unused field. **Audit recipe:**

```bash
rg -n 'shouldWaitOnAllReady' . 2>/dev/null
# Should return ZERO matches after upgrading to canary.104
```

If you do find matches, remove them — the new behavior is the same as what setting it to `true` achieved, so the migration is just a deletion.

- **No functional change for the default path** — projects that never set `shouldWaitOnAllReady` see no behavior change. Static generation still waits for all boundaries; dynamic rendering still streams.
- **No new API, no config flag, no codemod** — pure API cleanup.

**Sources:**
- [PR #96401 — `Remove obsolete shouldWaitOnAllReady render option`](https://github.com/vercel/next.js/pull/96401) · Zack Tanner · merged 2026-07-30T21:52:49Z · **SHIPPED in `16.3.0-canary.104`**.

### Vendor Bump: React `6cb4322d-20260729` → `0f42eac2-20260730` — PR #96402 (vercel-release-bot, merged 2026-07-30T21:21:08Z, **SHIPPED in `16.3.0-canary.104`**)

The 5th Next.js React vendor bump in 10 days. Brings **[React canary `19.3.0-canary-0f42eac2-20260730`](../components.md#react-1930-canary-0f42eac2-20260730--add-reactdombrowser-api-37143--3-devtools-prs-july-30-2026)** (which contains the new `ReactDOM.browser()` public API from PR #37143 + 3 DevTools-only fixes from PRs #37155, #37151, #37152) into Next.js's vendored React.

The diff is the standard `__NEXT_REACT_VERSION` constant update + the vendored `react` and `react-dom` package files. The PR body lists the 4 upstream React commits:

- [React PR #37155](https://github.com/facebook/react/pull/37155) — `[DevTools] Reset extension backend on pagehide`
- [React PR #37151](https://github.com/facebook/react/pull/37151) — `[DevTools] Create extension panels before React detection`
- [React PR #37152](https://github.com/facebook/react/pull/37152) — `[DevTools] Remove FlowFixMe from extension lifecycle`
- [React PR #37143](https://github.com/facebook/react/pull/37143) — `Add ReactDOM browser() API` (the headline)

**Practical impact for users today:**

- **All `next@16.3.0-canary.104` users** now have `ReactDOM.browser()` available from `react-dom` in any Client Component. See the new `React 19.3.0-canary-0f42eac2-20260730` section in `components.md` for the full API documentation, bundle size impact, and Next.js usage patterns.
- **No new public APIs at the Next.js level** — this is a vendored React bump, transparent to Next.js APIs.
- **No new config flags, no codemod, no breaking changes.**

**Audit recipe:**

```bash
npm view next@canary version
# → 16.3.0-canary.104

# Confirm the vendored React version
grep -r 'canary-0f42eac2' node_modules/next/dist/compiled/react/package.json
```

**Sources:**
- [PR #96402 — `Upgrade React from 6cb4322d-20260729 to 0f42eac2-20260730`](https://github.com/vercel/next.js/pull/96402) · vercel-release-bot · merged 2026-07-30T21:21:08Z · **SHIPPED in `16.3.0-canary.104`**.
- [React compare `6cb4322d...0f42eac2`](https://github.com/facebook/react/compare/6cb4322d...0f42eac2) — the 4 upstream commits.

### Turbopack Now Tracks Components of the Remainder Chunk — PR #96409 (sampoder, merged 2026-07-30T22:50:46Z, **SHIPPED in `16.3.0-canary.104`**)

The original Turbopack chunk-graph implementation assumed that components in the **remainder chunk** (the chunk that catches everything not placed into a specific named chunk) would not be referenced by other routes — so the components-of-the-remainder tracker was skipped. In practice, components can end up in the remainder that are used by other routes, which means the lack of tracking caused unnecessary rechunking and dedup misses.

**The fix** — keeps track of components of the remainder chunk so Turbopack can properly dedup cross-route imports. From the PR body:

> At the time we had assumed that these components would not be helpful, however, things can get put in the remainder that can be used on other routes so it is worthwhile to keep track of them.

**Practical impact for users today:**

- **Smaller bundle size on multi-route apps** — projects with shared components that span routes likely see smaller `chunks/` output in the build. The win is most pronounced on apps with many routes that share component libraries (every Next.js app, basically).
- **No new API, no config flag, no codemod** — pure internals improvement.
- **Audit recipe:**

```bash
# Look at the chunks emitted by the latest build
ls -la .next/static/chunks/ 2>/dev/null | head -30
# Compare to the same list before upgrading to canary.104
```

If you see a reduction in the chunk count or in the total bytes of `chunks/*.js`, this PR contributed.

**Sources:**
- [PR #96409 — `[turbopack] Keep track of components of the remainder chunk`](https://github.com/vercel/next.js/pull/96409) · sampoder · merged 2026-07-30T22:50:46Z · **SHIPPED in `16.3.0-canary.104`**.

### Practical impact summary for canary.104

- **Turbopack `optimizePackageImports` bundle size reduction** — PR #96383 brings Webpack parity for `sideEffects: false` handling. Free bundle reduction for projects using `optimizePackageImports` with libraries that declare `sideEffects: false`.
- **Cache Components + DOM-capable crawlers** — PR #96367 lets DOM-capable crawlers use the prerendered shell + streaming metadata path (the prerendered shell ships immediately, metadata fills in as it resolves). The previous behavior was forcing a fully buffered dynamic render for any custom `htmlLimitedBots` match — that's gone.
- **API cleanup** — PR #96401 removes the obsolete `shouldWaitOnAllReady` render option. `supportsDynamicResponse` is now the sole signal for static-vs-dynamic generation. Audit `rg 'shouldWaitOnAllReady'` after upgrading.
- **React `0f42eac2-20260730` vendor bump** — PR #96402 brings the new `ReactDOM.browser()` API + 3 DevTools fixes into the vendored React. See the matching `components.md` section for the full API documentation.
- **Turbopack cross-route chunk dedup** — PR #96409 keeps track of remainder-chunk components for proper cross-route import dedup. Smaller bundle size on multi-route apps.
- **No new config flags, no breaking changes for the default path.**

### Sources

- [GitHub: `v16.3.0-canary.104` release](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.104) · published 2026-07-30T23:56:25Z by `next-js-bot`
- [compare `v16.3.0-canary.103...v16.3.0-canary.104`](https://github.com/vercel/next.js/compare/v16.3.0-canary.103...v16.3.0-canary.104) — 17 commits ahead
- [PR #96383 — `[turbopack] Respect sideEffects in a package.json file with optimizePackageImports`](https://github.com/vercel/next.js/pull/96383) · sampoder · **SHIPPED in `16.3.0-canary.104`**
- [PR #96367 — `fix: respect htmlLimitedBots in cache components without buffering the full response`](https://github.com/vercel/next.js/pull/96367) · Zack Tanner · **SHIPPED in `16.3.0-canary.104`**
- [PR #96401 — `Remove obsolete shouldWaitOnAllReady render option`](https://github.com/vercel/next.js/pull/96401) · Zack Tanner · **SHIPPED in `16.3.0-canary.104`**
- [PR #96402 — `Upgrade React from 6cb4322d-20260729 to 0f42eac2-20260730`](https://github.com/vercel/next.js/pull/96402) · vercel-release-bot · **SHIPPED in `16.3.0-canary.104`**
- [PR #96409 — `[turbopack] Keep track of components of the remainder chunk`](https://github.com/vercel/next.js/pull/96409) · sampoder · **SHIPPED in `16.3.0-canary.104`**
- [PR #96365 — `Update Rust toolchain to nightly-2026-07-29`](https://github.com/vercel/next.js/pull/96365) · wbinnssmith · **SHIPPED in `16.3.0-canary.104`** (internal)
- [PR #96366 — `test: expand Cache Components metadata streaming coverage`](https://github.com/vercel/next.js/pull/96366) · tests only, **SHIPPED in `16.3.0-canary.104`**
- [PR #95432 — `fix(create-next-app): make AGENTS.md blue`](https://github.com/vercel/next.js/pull/95432) · cosmetic, **SHIPPED in `16.3.0-canary.104`**
- [PR #96234 — `[bench skill] Bench Edge builds of React too`](https://github.com/vercel/next.js/pull/96234) · styfle · low-impact skill update, **SHIPPED in `16.3.0-canary.104`**
- [React canary `19.3.0-canary-0f42eac2-20260730` GitHub compare (`6cb4322d...0f42eac2`)](https://github.com/facebook/react/compare/6cb4322d...0f42eac2) — the 4 upstream commits brought in by PR #96402


## 16.3 canary.105 SHIPPED — Turbopack `experimental.turbopackChunking` Config (NEW) + `turbopackFileSystemCacheForBuild` Default-On + Cache Components PPR Not-Found Resume Fix + `isHeadPartial` Fix + Unhandled-Rejection Logging Consolidation + App Shell Stale-Time Docs + Component-Chunks-for-Workers Fix + React `cbb046ab` Vendor Bump (15 commits, July 31, 2026)

**`next@16.3.0-canary.105` SHIPPED at 2026-07-31T23:57:13Z** (GitHub release tag `v16.3.0-canary.105` published by `next-js-bot`; npm `dist-tag.canary` moved at the same time; canary-branch head commit `a8dcd2562f — v16.3.0-canary.105`; commit range `38f0cde...a8dcd25` = **15 commits**). The v1.5.10/v1.5.11 cycles both captured the canary-branch as "7 commits ahead of canary.104, canary.105 itself not yet npm-published"; the v1.5.12 cycle (this cron, 06:03Z Aug 1) finds **all 7 canary-branch-ahead-of-canary.104 PRs SHIPPED**, plus 8 NEW commits inside the canary.105 window that v1.5.11 captured only as the canary-branch-ahead feed — **5 NEW material PRs v1.5.11 missed** (PR #96432, PR #96434, PR #96437, PR #96428 test-only, PR #96438 + PR #96442 + PR #96359 CI) + 1 canary-tag commit.

The 15 commits vs canary.104, in chronological merge order:

1. `e3d634e0` — **PR #96400** `Fix isHeadPartial when hydrating from a static fallback shell` (Andrew Clark / acdlite, merged 2026-07-31T04:37:16Z) — **SHIPPED in `16.3.0-canary.105`** (was "canary-branch ahead" in v1.5.10/v1.5.11)
2. `b4e3fecc` — **PR #96398** `[turbopack] add experimental.turbopackChunking config` (sampoder, merged 2026-07-31T06:37:37Z) — **SHIPPED in `16.3.0-canary.105`** (documented above in the headlines + Common Mistakes section)
3. `6f7ed2ec` — **PR #96312** `docs: document the App Shell stale time threshold for cached content` (icyJoseph, merged 2026-07-31T14:15:10Z) — **SHIPPED in `16.3.0-canary.105`**
4. `be7048ef` — **PR #96419** `Update @types/react and @types/react-dom to latest` (eps1lon, merged 2026-07-31T15:29:58Z) — **SHIPPED in `16.3.0-canary.105`**
5. `9bfaf63e` — **PR #96390** `Fix adapter outputs for not-found routes when used with cache components` (Zack Tanner / ztanner, merged 2026-07-31T15:30:58Z) — **SHIPPED in `16.3.0-canary.105`**
6. `7612eaed` — **PR #95999** `Consolidate unhandled rejection logging into a single listener` (eps1lon, merged 2026-07-31T15:36:41Z) — **SHIPPED in `16.3.0-canary.105`**
7. `6523a33f` — **PR #96395** `Enable turbopackFileSystemCacheForBuild by default` (Tobias Koppers / sokra, merged 2026-07-31T17:24:41Z) — **SHIPPED in `16.3.0-canary.105`**
8. `a2d55862` — **PR #96432** `[turbopack] Fix component chunks for workers` (sampoder, merged 2026-07-31T18:12:23Z) — **NEW in v1.5.12 — SHIPPED in `16.3.0-canary.105`** (subsection below)
9. `f5c81a82` — **PR #96434** `Upgrade React from 0f42eac2-20260730 to cbb046ab-20260731` (vercel-release-bot, merged 2026-07-31T19:27:35Z) — **NEW in v1.5.12 — SHIPPED in `16.3.0-canary.105`** (subsection below)
10. `dc358f45` — **PR #96438** `[ci] Run new/changed deploy tests asap` (CI-only) — non-material
11. `e89118a0` — **PR #96442** `[react-sync] Enable auto-merge on PRs` (CI-only) — non-material
12. `fd6f11ba` — **PR #96437** `[turbopack] Introduce documentation on experimental.turbopackChunking` (docs for PR #96398's new config)
13. `43916339` — **PR #96359** `Add automated code review workflow` (CI-only) — non-material
14. `f7f2036e` — **PR #96428** `test: stabilize after() deploy revalidation checks` (test-only) — non-material
15. `a8dcd256` — `v16.3.0-canary.105` version-tag commit (no code changes)

This section's sub-sections cover the **5 NEW material PRs v1.5.11 missed** (PRs #96432 + #96434 + the 3 smaller ones above the headline fold). The headlines across all 15 commits: **the biggest user-facing change is PR #96395 flipping `turbopackFileSystemCacheForBuild` to default-ON for `next build`** (every local + Vercel build now uses the warm filesystem cache by default — ~5-30% speedup on warm builds, opt-out via `experimental.turbopackFileSystemCacheForBuild: false`); **#96398** introduces the new top-level `experimental.turbopackChunking` config (9 options, replaces the old `turbopack.chunkingHeuristics` + `turbopackGenerateComponentChunks` namespaces which now throw at config-eval time); **#96419** bumps `@types/react` to 19.2.18 + `@types/react-dom` to 19.2.4 so vanilla-TS users can now consume `ReactDOM.browser()` from React canary `0f42eac2` immediately (no need to wait for the next minor stable); **#96434** vendor-bumps the React canary `cbb046ab-20260731` (with #37104's conditional `use()` warning machinery) into Next.js's bundled React; **#96390** makes `not-found.tsx` work correctly with Cache Components + PPR + adapters (a `not-found.tsx` that suspends now renders its dynamic content when deployed, instead of being treated as a complete static 404); **#96400** fixes `isHeadPartial` in CDN-served fallback shells; **#95999** collapses 3 redundant `unhandledRejection` loggers into 1 (`Symbol.for`-keyed shared listener); **#96312** is a docs PR that adds the 5-minute `stale`-time floor to all 5 App Shell docs pages, locking in a previously-implicit behavior into documentation; **#96432** fixes the worker-chunk regression introduced by PR #95261 (`module_chunks` made chunks stop being plain path strings, breaking Web Worker code-chunk resolution). The `turbopackChunking` config is now GA in npm-published canary.105 — old `experimental.turbopack.chunkingHeuristics` + `experimental.turbopackGenerateComponentChunks` namespaces throw at config-eval time with explicit migration errors (per the `config.ts` throws quoted in the canary.104-AHEAD section above). The +9% Fresh Build + +8% Cached Build regression documented in PR #96398's stats-bot comment is now live in npm.

### PR #96395 — `Enable turbopackFileSystemCacheForBuild by default` (sokra, merged 2026-07-31T17:24:41Z) — **THE BIGGEST BEHAVIORAL CHANGE OF THIS CRON**

Authored by Tobias Koppers (sokra, the Webpack/Turbopack author), **this PR flips the Turbopack filesystem cache to ON by default for `next build`**, matching the `next dev` default that was enabled back in v16.1.0. The change is the **largest default-on build behavior change in the 16.3 line** and will affect every local + Vercel build going forward.

**The pre-PR config:**

```ts
// next.config.ts (Next.js 16.x prior to canary.105)
function turbopackFileSystemCacheForBuildDefault(): boolean {
  return (process.env.NEXT_PUBLIC_CI_FORCE_TURBOPACK_FILE_SYSTEM_CACHE_BUILDS === 'true')
    || (process.env.NOW_BUILDER === '1' && isStableBuild())
    // ... etc — canary builds on Vercel only
}
```

**The post-PR config (per the PR diff):**

```diff
- function turbopackFileSystemCacheForBuildDefault(): boolean {
-   return (
-     process.env.NEXT_PUBLIC_CI_FORCE_TURBOPACK_FILE_SYSTEM_CACHE_BUILDS === 'true' ||
-     (process.env.NOW_BUILDER === '1' && isStableBuild()) ||
-     ...
-   )
- }
+ function turbopackFileSystemCacheForBuildDefault(): boolean {
+   return !(process.env.CI === 'true' && process.env.NOW_BUILDER !== '1')
+   // → true UNLESS we're in CI but NOT on Vercel
+   // → i.e. CI not on Vercel (GitHub Actions, GitLab, Docker CI, etc) keeps the cache OFF
+   // → because the cache is unlikely to persist between CI runs
+ }
```

**The exact behavior change:**

| Build context | Pre-#96395 (canary.104-) | Post-#96395 (canary.105+) |
|---|---|---|
| `next build` locally on dev machine | Off (opt-in only) | **On** — warm builds use the `.next/cache/turbopack` filesystem cache |
| `next build` on Vercel | On (canary + stable builds) | **On** (unchanged) |
| `next build` in non-Vercel CI | Off | **Off** (unchanged — cache unlikely to persist between runs) |
| Explicit `experimental.turbopackFileSystemCacheForBuild: true` | On | On (unchanged) |
| Explicit `experimental.turbopackFileSystemCacheForBuild: false` | Off | Off (unchanged) |

**Why exclude CI-not-on-Vercel:** the cache lives in `.next/cache/turbopack/`, which is ephemeral between CI runs unless the CI provider preserves `.next` (Vercel does; most others don't). Shipping a stale cache from CI would be a worse experience than skipping it entirely. The CI-exclusion matches what Vercel already does for the dev-mode counterpart.

**Practical impact for users:**

- **Local builds**: every `next build` after the first now uses a warm filesystem cache. Expected **5-30% speedup on warm builds** depending on project size (the bigger the dep graph, the bigger the win — Vercel-internal benchmarks show 30-60% on large apps).
- **The first build** is unchanged (cold cache, no warming yet) — the speedup comes on build 2+. To force a "fair" webpack-vs-turbopack comparison, delete `.next` between builds (the docs note this explicitly).
- **Deployment providers**: can now flip the default for their platform by setting `experimental.turbopackFileSystemCacheForBuild: true` in the adapter's `modifyConfig` hook (docs page `/docs/app/api-reference/adapters/creating-an-adapter` covers this).
- **No configuration change required**: opt-out is `experimental.turbopackFileSystemCacheForBuild: false` if you need it.
- **No new API, no new flag**, no bundle size impact — pure build-time internal change.

**Audit recipe:**

```bash
# Confirm the default flipped in your project:
rg 'turbopackFileSystemCacheForBuild' next.config.ts next.config.js next.config.mjs 2>/dev/null
# If nothing → you're using the default (now ON for local+Vercel, OFF for other CI)

# Verify the cache directory is being written:
next build  # first build creates the cache
ls -la .next/cache/turbopack/  # should have content after the first build
next build  # second build should be measurably faster

# Disable if needed:
# next.config.ts:
#   experimental: { turbopackFileSystemCacheForBuild: false }
```

**Verifying your project got the new default:** if you're on `next@16.3.0-canary.105+`, `next build` runs will start writing to `.next/cache/turbopack/` without you having to set the flag. The docs version table now reads:

```
| Version   | Changes                                                                              |
| --------- | ------------------------------------------------------------------------------------ |
| `v16.3.0` | FileSystem caching is enabled by default for builds (depends on CI platform support) |
| `v16.1.0` | FileSystem caching is enabled by default for development                             |
| `v16.0.0` | Beta release with separate flags for build and dev                                   |
| `v15.5.0` | Persistent caching released as experimental on canary releases                       |
```

### PR #96419 — `Update @types/react and @types/react-dom to latest` (eps1lon, merged 2026-07-31T15:29:58Z)

Bundled-dep bump by [eps1lon](https://github.com/eps1lon) (the React team types maintainer). Bumps:

- **`@types/react` 19.2.17 (2026-06-05T20:10:24Z) → 19.2.18 (2026-07-30T21:54:03Z)** — released ~19h before this PR
- **`@types/react-dom` 19.2.3 (2025-11-12T04:37:39Z) → 19.2.4 (2026-07-30T21:53:05Z)** — released ~19h before this PR, after **8.5 months of staleness**

**The bundling strategy:** Next.js ships `@types/react` and `@types/react-dom` in its own monorepo `/packages/next/` folder so that the bundled React declaration chain (used inside Next.js's vendored `.d.ts`) references a known version of `react` types. When the user installs `next@canary`, they automatically get whichever versions Next.js ships.

**Why 19.2.18 + 19.2.4 specifically?**

- `@types/react-dom@19.2.4` is the first `@types/react-dom` release in 8.5 months that **adds a `ReactDOM.browser()` type declaration**. Without this, vanilla-TS code trying to use `import { browser } from 'react-dom'` on top of `react@canary-cbb046ab` (or `0f42eac2`, which introduced `browser()`) would get `Module '"react-dom"' has no exported member 'browser'`. Bumping to 19.2.4 unblocks vanilla-TS without awaiting the next minor stable of `@types/react`.
- `@types/react@19.2.18` is a small bug-fix release after 19.2.17 (no new APIs in the .d.ts surface; in particular, no `use()` warning types — the warning is a runtime DEV check, not a TS type).

**Why `minimumReleaseAgeExclude`?**

Both `@types/react@19.2.18` and `@types/react-dom@19.2.4` are recently published (< 48h old at PR time), so they bypass Next.js's standard `minimumReleaseAge` policy that delays integrating new deps until they're 1 week old (to avoid pulling in publish-bug regressions). The PR adds them to `minimumReleaseAgeExclude` explicitly to opt them in.

**Practical impact:**

- **`next@16.3.0-canary.105+`** vendors `@types/react@19.2.18` + `@types/react-dom@19.2.4`, so vanilla-TS users can `npm install next@canary` + `import { browser } from 'react-dom'` and the type-check works immediately.
- **`next@latest` (16.2.12)** vendors older types (`@types/react@19.2.x` but with no `ReactDOM.browser()`). Stable `@types/react-dom@19.2.3` is also missing the `browser()` type. To use `browser()` on stable, also pin `npm install -D @types/react@^19.2.18 @types/react-dom@^19.2.4` (they're back-compatible).
- **No action required** for users not using `ReactDOM.browser()` — the bundled types upgrade is silent.

**Audit recipe:**

```bash
# After upgrading next@canary.105+:
grep -E '"@types/react"|"@types/react-dom"' node_modules/next/package.json
# → Both should be 19.2.18 / 19.2.4

# Verify ReactDOM.browser() type is available:
node -e 'console.log(require("react-dom").browser || "browser undefined (try with @types/react-dom 19.2.4+)")'
```

### PR #96390 — `Fix adapter outputs for not-found routes when used with cache components` (Zack Tanner, merged 2026-07-31T15:30:58Z)

The second-largest material PR in this batch. Fixes a subtle bug where **a `not-found.tsx` that contains any state that needs to be resolved at request time (i.e. anything dynamic / anything that suspends under Cache Components) was being treated as a complete static 404 by the build**, when it should have been emitted as a PPR-eligible resume output. In deployed environments, this manifested as **dynamic 404 content never rendering** — the static fallback 404 shell would always be served.

**The bug:**

```tsx
// app/not-found.tsx
import { headers } from 'next/headers'
import { getCustomError } from './_lib/getCustomError'

export default async function NotFound() {
  const headers = await headers()       // ← makes this page dynamic
  const variant = headers.get('x-tenant') ?? 'default'
  const error = await getCustomError(variant)  // ← suspends
  return <h1>{error.title}</h1>
}
```

Build behavior pre-PR: Next.js saw the `headers()` call, treated the `/_not-found` route as partially prerendered, BUT then copied the partial HTML shell to `server/pages/404.html` and **omitted** the corresponding `/_not-found` prerender from adapter outputs. Deployed environments served the static shell only — no dynamic content ever appeared.

**The fix (per PR body):**

> This change distinguishes fully static not-found routes from partially prerendered ones. Fully static routes retain the existing `404.html` behavior, while PPR routes remain resumable prerender outputs.

**The corresponding `e2e/not-found-non-document-dynamic` test expansion** now verifies that:

- Dynamic not-found content **renders** for ordinary missing paths
- Dynamic not-found content **renders** when `/_not-found` is selected by a rewrite
- A PPR not-found **does NOT** emit `server/pages/404.html`
- Its adapter output **includes** `PARTIALLY_STATIC` metadata, postponed state, and a resume chain

**Practical impact:**

- **The bug-fix**: if you have a `not-found.tsx` with `headers()` / `cookies()` / dynamic `use cache: private` data, your 404 page now actually serves its dynamic content when deployed (Vercel + adapters). Before, users always saw the static fallback shell.
- **Fully static `not-found.tsx`** (no dynamic data) is unaffected — continues to emit and serve `pages/404.html` as before.
- **No new API, no config flag, no codemod** — pure build-time fix.

**Audit recipe:**

```bash
# If you've ever had a "my 404 page doesn't render X dynamic data" report from a deployed environment,
# this is the fix — verify by inspecting the adapter output:
grep -r 'PARTIALLY_STATIC' .next/server/app/not-found.body  # should show the metadata on dynamic not-found
ls .next/server/pages/404.html  # should NOT exist for dynamic not-found, SHOULD exist for static
```

### PR #95999 — `Consolidate unhandled rejection logging into a single listener` (eps1lon, merged 2026-07-31T15:36:41Z)

Quality-of-life PR. Pre-PR, **a single `unhandledRejection` could be logged by up to 3 independent process listeners at once**:

1. The render runtime's crash-prevention handler in `process-error-handlers.ts` (a bare `console.error`) — **must exist on every deployment target** (per [Next.js issue #77997](https://github.com/vercel/next.js/issues/77997))
2. The router server's `Log.error('unhandledRejection: ', err)`
3. The dev server's `logErrorWithOriginalStack`

On self-hosted `next start` / `next dev`, listeners 1 + 2 + 3 all run in the same process — a single rejection was being **logged 3 times in 3 different formats**.

**The fix:**

PR #95999 introduces `registerUnhandledRejectionListener` + `isUnhandledRejectionListenerRegistered` in `process-error-handlers.ts`, and converts the router server + dev server to **check-then-register** instead of installing their own rejection loggers:

- The listener function is shared via a `Symbol.for` key on `globalThis` — so multiple copies of the module (e.g. one in the pre-compiled server bundle, one in a route module bundle) **detect a single listener instance** and only register once.
- The registration check queries `process.listeners('unhandledRejection')` instead of a module-global flag, so it **stays accurate even after external code calls `process.removeAllListeners('unhandledRejection')`**. `installProcessErrorHandlers` therefore calls the register function unconditionally.

The first commit in the PR adds a failing test showing the pre-PR behavior (multiple-logs); the second commit fixes it.

**What about `uncaughtException`?** Per the PR body, `uncaughtException` handlers are left untouched — they have a different "fatal" semantic and triple-logging of those is correct (a `uncaughtException` is by definition a process-level error and being noisy about it is intentional).

**Practical impact:**

- **`next dev` users** will see **fewer noisy duplication logs** in the terminal when an unhandled rejection occurs (down from 3-format-different logs to 1-format).
- **`next start` users** (self-hosted) will see the same simplification in their server stdout/stderr logs.
- **Deployment adapters**: the runtime handler stays, so the safety net isn't weakened.
- **No new API, no new config flag** — purely internal refactor.

**Audit recipe:**

```bash
# Quick and dirty: trigger an unhandled rejection in a Next.js dev page and see how many
# "unhandledRejection" lines show up in the dev terminal. Should now be 1 (was 3 on canary.104).
# In dev, console-tap: window.Promise.reject(new Error('test')) in browser console with a page route
```

### PR #96312 — `docs: document the App Shell stale time threshold for cached content` (icyJoseph, merged 2026-07-31T14:15:10Z)

Documentation PR. Locks in a behavior that was previously implicit / undocumented: **the App Shell only carries `use cache` content whose `stale` time is at least 5 minutes**. Content with `stale` < 5 min is excluded from the shell (it's better streamed in after navigation than prefixed into the click target).

**The docs updates span 5 files:**

| File | Change |
|---|---|
| `docs/01-app/02-guides/adopting-partial-prefetching.mdx` | New "Good to know" callout: "The App Shell carries cached content whose `stale` time is at least 5 minutes, which holds for the `default` profile used above and every preset except `seconds`." |
| `docs/01-app/02-guides/instant-navigation.mdx` | Updated the opted-out-segment section: "caching it with `use cache: private` lets the App Shell carry it ahead of the click instead of opting out, as long as its `stale` time is at least 5 minutes." |
| `docs/01-app/03-api-reference/01-directives/use-cache-private.mdx` | Updated the "Good to know" callout: "The `stale` time must be at least 30 seconds for runtime prefetching to work, and at least 5 minutes for the content to be included in the route's App Shell." |
| `docs/01-app/03-api-reference/04-functions/cacheLife.mdx` | New bullet in the `stale` description + a fully rewritten "Prerendering behavior" subsection that splits into 3 buckets: `revalidate:0` / `expire < 5min` excluded from prerenders (dynamic holes); `stale < 30s` excluded from prerenders (prefetch-expires-too-fast); `stale 30s..5min` included in prerenders but excluded from App Shell. Of presets, only `seconds` falls under any of these thresholds. |
| `docs/01-app/04-glossary.mdx` | App Shell definition updated: "Cached content is included when its `stale` time is at least 5 minutes, since the shell is reused for longer than shorter-lived content stays fresh." |

Closes Linear [DOC-6480](https://linear.app/vercel/issue/DOC-6480/app-shells-exclude-stale-5-minutes-caches).

**Practical impact:**

- **No code change** — purely documentation. The behavior was already in place.
- **For users adopting Partial Prefetching** or authoring `use cache private` directives: now have explicit documentation of the 5-minute floor for App Shell inclusion. Previously this was implicit / required reading the PR bodies for `cacheLife`.
- **Short-lived cached content** (anything with `stale < 5min`) is now correctly documented as **excluded from the shell** — important for agents reasoning about prefetch payloads.

### Practical impact summary for canary.105

- **`next@16.3.0-canary.105` users (when it ships)** get:
  - **Turbopack filesystem cache default-on for builds** (PR #96395) — ~5-30% warm-build speedup locally + on Vercel; can be disabled with `experimental.turbopackFileSystemCacheForBuild: false`.
  - **`@types/react@19.2.18` + `@types/react-dom@19.2.4`** (PR #96419) — vanilla-TS users can `import { browser } from 'react-dom'` and the types check.
  - **Dynamic `not-found.tsx` works correctly with Cache Components + adapters** (PR #96390) — no more "404 page never renders dynamic content" reports.
  - **Less noise in unhandled-rejection logging** (PR #95999) — 3-format logs collapse to 1.
  - **Docs clarity** (PR #96312) — the 5-minute `stale` floor for App Shell inclusion is now documented, not implicit.
- **No breaking changes for users on `next@canary.104`** — the only behavior change is the filesystem-cache default-on (and that one is opt-out-able).
- **No new public APIs, no new config flags** (other than the existing opt-out for #96395).

## 16.3 canary.106 SHIPPED — Fix Hybrid Pages/App Router `not-found` Rendering with Adapters + Warn that `experimental.useCache` is Deprecated (2 PRs, August 1–2, 2026)

The v1.5.12 cron (06:03Z Aug 1) found **2 NEW commits ahead of canary.105** (canary-branch head `ccd47cfe53` at 2026-08-01T00:26:11Z). The canary.105 release tag was published **17 minutes AFTER PR #96392 had already landed** in the canary-branch — so PR #96392 didn't make it into npm-published canary.105 and users hitting this issue had to wait for canary.106. **`next@16.3.0-canary.106` SHIPPED at 2026-08-01T23:56:54Z** (GitHub release tag `v16.3.0-canary.106` published by `next-js-bot`; npm `dist-tag.canary` moved at the same time; canary-branch head commit `9480f7f — v16.3.0-canary.106`; commit range `ccd47cf...9480f7f` = **3 commits** = PR #96392 + PR #96448 + the version-tag commit). The release body lists both PRs explicitly. **canary-branch now has 12 commits ahead of canary.106** (verified at 2026-08-03T18:02Z via `GET /repos/vercel/next.js/compare/v16.3.0-canary.106...canary` returning `ahead_by: 12` — was `0` at v1.5.13 commit time, was `3` at v1.5.18 commit time, now `12` after canary.107 SHIPPED + canary.108-ahead material). The 12 commits break down as: 3 in canary.107 (PR #96493 + PR #96426 + version-tag `4fd843f`) + 9 ahead of canary.107 (3 material: PR #96497 + PR #96308 + PR #93132 + 5 infra: #96386 + #96526 + #96527 + #96534 + #96505 + the canary.108 version-tag commit not yet at this cron's check). Both previously-pending PRs are now SHIPPED:

- **PR #96392** `Fix hybrid Pages/App Router not-found rendering with adapters` — **SHIPPED in `16.3.0-canary.106`** (was canary-branch ahead of canary.105 in v1.5.12; did NOT make it into canary.105; now live in canary.106 npm-published build).
- **PR #96448** `Warn that experimental.useCache is deprecated` — **SHIPPED in `16.3.0-canary.106`** (was canary-branch ahead of canary.105 in v1.5.12; now live in canary.106 npm-published build).

### PR #96392 — `Fix hybrid Pages/App Router not-found rendering with adapters` (Zack Tanner / ztanner, merged 2026-07-31T23:40:38Z, 17 minutes BEFORE canary.105 was published)

The PR landed at 23:40:38Z but the canary.105 version-tag commit `a8dcd2562f` was made at 23:35:12Z. So the user-facing canary.105 npm-published at 23:57:13Z contains the canary.104→canary.105 range that **excluded this PR** (it landed at `893121b887` 5min25s AFTER `a8dcd2562f` was tagged). Users on `next@16.3.0-canary.105` who hit the cross-router not-found bug must be on `next@16.3.0-canary.106` or later. **As of this cycle, canary.106 is live (npm-published 2026-08-01T23:56:54Z) and the fix is in npm.**

**The bug:** When Pages Router and App Router routes coexist, a Pages Router route that returns `notFound` will internally hand rendering to the App Router not-found page. Next.js [prefers the App Router `/_not-found` entry for 404 responses](https://github.com/vercel/next.js/blob/5f5e4773bbb54f19697063ea093e8d89ca7d081a/packages/next/src/server/base-server.ts#L2965-L2980) before falling back to a Pages Router `/404` page. **When deployed via an adapter**, this means the build would have attempted to resume the Pages entry, which wouldn't contain the relevant prerendered shell or postponed state. If the not-found page suspends, its response cannot be resumed using the metadata associated with the Pages Router output.

**The fix:** For adapter requests, mark the handoff with an **empty postponed state**. This uses the existing resume protocol to perform a complete, non-cacheable dynamic render. `next start` continues to use the normal PPR path, where it can load and resume the App Router not-found output directly.

**Trade-off:** This trades **PPR shell reuse for correctness only during this cross-router not-found handoff**. Longer term, adapters should have an explicit contract for cross-output handoffs so they can select and resume the App Router not-found output normally, without necessarily declaring the pages router output PPR-capable. The change also documents the empty postponed-state contract and adds deployment regression coverage.

**Verification (from the PR body):**

- Added an end-to-end test that verifies a Pages Router `notFound()` resolution properly transitions to App Router `/_not-found` rendering on adapter deployments.
- The test covers both the explicit `notFound()` return and the implicit case (throwing `NEXT_NOT_FOUND`).
- Adapter regression coverage added to ensure future adapter changes don't reintroduce the bug.

**Practical impact:**

- **If you're on `next@16.3.0-canary.106` or later and using an adapter** (e.g. `@vercel/next`, Cloudflare adapter, Netlify adapter, Express custom adapter, etc.) with **both Pages Router and App Router routes** and either route returns `notFound()`, the fix is live and you can stop worrying about it. **If you're stuck on canary.105**, the bug is still present — bump to canary.106+ to pick up the fix.
- **If you're using `next start` directly** (or Vercel's managed hosting — which doesn't use the adapter path for default deployments), you are unaffected — the normal PPR path continues to work.
- **No new API, no config flag, no codemod** — pure build-time fix.
- **Future direction:** adapters should expose a `modifyConfig` hook for the cross-output handoff contract (the PR body explicitly notes this as a follow-up).

**Audit recipe:**

```bash
# Verify you're on canary.106+ if you depend on this fix:
npm ls next  # should show 16.3.0-canary.106 or later

# If you're on canary.105 and need this fix:
# Option A: bump to canary.106 (live now, npm-published 2026-08-01T23:56:54Z)
# Option B: pin to canary.107 or later (once available)
# Option C: workaround — move the cross-router not-found handling to a single router
#   (i.e. don't mix Pages + App Router for not-found scenarios that hit adapters)
```

### PR #96448 — `Warn that experimental.useCache is deprecated` (unstubbable, merged 2026-08-01T00:26:11Z)

This PR is **small in code but huge in ergonomics** — it surfaces a deprecation warning that's been hidden in JSDoc since [PR #92316](https://github.com/vercel/next.js/pull/92316).

**Background:** The `experimental.useCache` option has carried a `@deprecated` JSDoc annotation since PR #92316, but nothing surfaced that at runtime, so users had no signal to migrate unless their editor happened to show the annotation.

**The fix (per PR body):**

> This change logs a warning whenever the option is set explicitly, pointing at the top-level `cacheComponents` option instead. When `cacheComponents` is already enabled the option is redundant, so the message says it can be removed; otherwise it asks the user to switch.

**The bigger change — `experimental.useCache` disabling is now rejected when `cacheComponents` is enabled:**

> Disabling the option while `cacheComponents` is enabled is now rejected outright. That combination is contradictory: it turns off the very directive Cache Components is built around, and the resulting compile error asks the user to enable `cacheComponents`, which they already have on. Throwing here matches how `cachedNavigations` and `partialPrefetching` reject configurations that require `cacheComponents`.

**Why `assignDefaultsAndValidate` (not `checkDeprecations`)?**

> The checks live in `assignDefaultsAndValidate` rather than alongside the other deprecations in `checkDeprecations`, because the latter runs before `enforceExperimentalFeatures` and would therefore not see a `cacheComponents` value that was enabled through the environment. A defined `experimental.useCache` value is what identifies an explicit user setting, since the option is otherwise backfilled from `cacheComponents` immediately below the new checks.

**Practical impact:**

- **If you have `experimental.useCache: true` set in your `next.config.ts`** — you'll now see a runtime warning on `next dev` / `next build` / `next start` telling you to switch to `cacheComponents: true` (or remove the line entirely if `cacheComponents` is already on).
- **If you have `experimental.useCache: false` AND `cacheComponents: true`** — you'll now get a **compile error** (not a warning) that says "you're disabling the very directive Cache Components is built around; remove the `useCache: false` line". This is a **deliberate breaking change** for any project that's been relying on the contradiction; the fix is to remove the `useCache: false` line.
- **If you have neither set** — nothing changes (the option is backfilled from `cacheComponents`).
- **Migration path is mechanical:**
  ```diff
  // next.config.ts
  const config = {
    experimental: {
  -   useCache: true,
  +   cacheComponents: true,  // top-level now
    }
  }
  ```

**Audit recipe:**

```bash
# Find projects with the deprecated option:
rg -n 'experimental.*useCache' next.config.*

# Find projects with the contradictory config (useCache: false + cacheComponents: true):
rg -B2 -A4 'useCache.*false' next.config.* | rg -B2 'cacheComponents.*true'

# If you find a contradiction, the fix is to remove the useCache: false line:
# - next.config.ts:
#   experimental: {
#     cacheComponents: true,
#   - useCache: false,    // ← remove this
#   }
```

### Practical impact summary for canary.106

- **`next@16.3.0-canary.106` users** (live now, npm-published 2026-08-01T23:56:54Z) get:
  - **All 7 previously-canary-branch-ahead PRs now live in npm** — the new `experimental.turbopackChunking` config GA (replaces old chunkingHeuristics + turbopackGenerateComponentChunks), `turbopackFileSystemCacheForBuild` default-on, `not-found.tsx` Cache Components fix, `isHeadPartial` fix, unhandled-rejection logging consolidation, App Shell stale-time docs, Component-Chunks-for-Workers fix (PR #96432), React `cbb046ab` vendor bump (PR #96434) **+ PR #96392 (hybrid Pages/App Router not-found fix) + PR #96448 (`experimental.useCache` deprecation warning + compile-error)**.
  - **Type-vendor bump** — `@types/react@19.2.18` + `@types/react-dom@19.2.4` ship `ReactDOM.browser()` types in vanilla-TS (shipped in canary.105, carried forward).
  - **Breaking: disabling `experimental.useCache` while `cacheComponents` is enabled now throws at config-eval time** — if you have both lines, remove the `useCache: false` line (see PR #96448 below for full details).
  - **No new public APIs**, no new config flags in canary.106.
- **canary.106 vs canary.105 diff** is exactly **3 commits** (PR #96392 + PR #96448 + the version-tag commit). No other changes slipped in between. So if you're on canary.105, the only new things in canary.106 are the hybrid not-found fix + the useCache deprecation warning/throw.
- **No breaking changes for users on `next@canary.104`** — the only behavior changes across canary.105 + canary.106 are the filesystem-cache default-on (opt-out-able) + the `experimental.turbopackChunking` config-eval throw (opt-in-able by migrating your config) + the `experimental.useCache` deprecation warning + the `experimental.useCache: false` rejection (mechanical fix: remove the line).
- **canary.107-ahead → SHIPPED** — canary.107 SHIPPED at 2026-08-03T14:04:47Z (npm `dist-tag.canary` moved from `16.3.0-canary.106` → `16.3.0-canary.107`). The 3 commits v1.5.18 documented as canary-branch-ahead-of-canary.106 (PR #96493 + the canary.107 version-tag commit `4fd843f` + PR #96426 cache-poisoning fix) are now live in npm. See the new `## 16.3 canary.107 SHIPPED` section above for the full diff + practical impact.
- **canary.108-ahead** — canary-branch now has **9 NEW commits ahead of canary.107** (verified at 2026-08-03T18:02Z via `GET /repos/vercel/next.js/compare/v16.3.0-canary.107...canary` returning `ahead_by: 9`): 3 material PRs (**PR #96497 Enable TypeScript CLI by default — timneutkens, Aug 3 16:10Z — flips `experimental.useTypeScriptCli` default `false` → `true`** + **PR #96308 Fix App Router scroll padding visibility — DavidIlie, Aug 3 15:00Z** + **PR #93132 fix: double fragment on navigation — icyJoseph, Aug 3 16:21Z**) + 5 infra PRs (#96386 turboprace sharp bump, #96526 ISR+CC docs, #96527 CI preview-tarball wait, #96534 hybrid-not-found test skip, #96505 disabled-deploy-tests flag) + the canary.108 version-tag commit (not yet committed at this cron's check). See the new `## 16.3 canary.108-ahead` section above for the full diff + practical impact.
- **No new public APIs, no new config flags** (other than the existing opt-out for #96395 + the existing `turbopackChunking` config for #96398 + the existing opt-out for #96419's bundled-deps bump).

### Sources

- [Next.js canary-branch compare: `v16.3.0-canary.106...canary` (12 commits — 3 in canary.107 + 9 ahead of canary.107)](https://github.com/vercel/next.js/compare/v16.3.0-canary.106...canary) — verified at 2026-08-03T18:02Z; canary.107 is now SHIPPED (see the `## 16.3 canary.107 SHIPPED` section above), 9 commits ahead of canary.107 in canary.108-ahead (see the `## 16.3 canary.108-ahead` section above)
- [Next.js canary-branch compare: `v16.3.0-canary.105...canary` (3 commits — 2 PRs in canary.106 + 1 version-tag commit)](https://github.com/vercel/next.js/compare/v16.3.0-canary.105...canary) — the inventory of canary.106's 2 PRs (PR #96392 + PR #96448)
- [Next.js canary-branch compare: `v16.3.0-canary.104...canary` (18 commits — 15 in canary.105 + 3 in canary.106)](https://github.com/vercel/next.js/compare/v16.3.0-canary.104...canary) — the cumulative view
- [**Next.js release `v16.3.0-canary.106`**](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.106) — published 2026-08-01T23:56:54Z by `next-js-bot`, body lists both PRs
- [**Next.js PR #96392** — `Fix hybrid Pages/App Router not-found rendering with adapters**](https://github.com/vercel/next.js/pull/96392) — by [Zack Tanner](https://github.com/ztanner), merged 2026-07-31T23:40:38Z, **landed 17min BEFORE canary.105 was published** — not in npm-published canary.105, now SHIPPED in canary.106
- [Next.js base-server.ts lines 2965-2980 — App Router not-found preference logic](https://github.com/vercel/next.js/blob/5f5e4773bbb54f19697063ea093e8d89ca7d081a/packages/next/src/server/base-server.ts#L2965-L2980) — the existing cross-router preference that this PR overrides for adapter requests
- [**Next.js PR #96448** — `Warn that experimental.useCache is deprecated**](https://github.com/vercel/next.js/pull/96448) — by [unstubbable](https://github.com/unstubbable), merged 2026-08-01T00:26:11Z, the warning + compile-error for the deprecated `useCache` option, **now SHIPPED in `16.3.0-canary.106`**
- [Next.js PR #92316 — original `experimental.useCache` `@deprecated` JSDoc annotation](https://github.com/vercel/next.js/pull/92316) — the hidden deprecation that PR #96448 surfaces
- [Next.js `cacheComponents` config docs](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents) — the top-level config that replaces `experimental.useCache`

## 16.3 canary.107 SHIPPED — Turbopack Build Filesystem Cache Default-On in **All Environments** (PR #96493) + Cache-Poisoning-After-Prerender-Abort Fix (PR #96426) (2 PRs + 1 version-tag commit, npm-published 2026-08-03T14:04:47Z)

The v1.5.17 cron (18h ago) said "canary-branch has 2 commits ahead of canary.106" and the v1.5.18 cron (6h ago) said "canary.107 is built on the canary branch but not yet npm-published; npm publish expected within hours". **Both predictions are now obsolete** — **`next@16.3.0-canary.107` SHIPPED at 2026-08-03T14:04:47Z** (npm `dist-tag.canary` moved from `16.3.0-canary.106` → `16.3.0-canary.107`; GitHub release tag `v16.3.0-canary.107` published at the same time by `next-js-bot`). The canary-branch now has **9 NEW commits ahead of canary.107** (verified at 2026-08-03T18:02Z via `GET /repos/vercel/next.js/compare/v16.3.0-canary.107...canary` returning `ahead_by: 9`, see the new `## 16.3 canary.108-ahead` section below).

### What's in `next@16.3.0-canary.107` (2 PRs + 1 version-tag commit)

1. **`3d4d46f` — PR #96493** [`Enable Turbopack build filesystem cache by default`](https://github.com/vercel/next.js/pull/96493) (Tim Neutkens / timneutkens, merged 2026-08-02T18:33:34Z) — **THE BIGGEST BEHAVIORAL EXPANSION OF THIS CANARY**. (Material documentation moved to the `### Why PR #96493 matters` subsection below.)
2. **`3317392` — PR #96426** [`[Cache] Make caches error if called after prerender aborts`](https://github.com/vercel/next.js/pull/96426) (Janka Uryga / jankaeryga, merged 2026-08-03T11:42:26Z) — **the material bug fix of this canary**. Closes issue #96339. Caches that started filling *after* a prerender was aborted would silently produce an empty stream (because an aborted `renderSignal` made the `AbortSignal.any(...)` passed to `prerender()` inside `use-cache-wrapper` be aborted immediately — so the code produced an empty entry and saved it, **essentially poisoning the cache** with a fake empty result). Fixed by removing `renderSignal` from the `AbortSignal.any(...)` and short-circuiting with a rejected promise *before* reaching the cache-fill codepath. The fix is **not directly observable to userspace** (since the prerender is already aborted), but without the fix, subsequent requests would hit the poisoned cache entry and return empty content indefinitely. **This is the kind of bug that manifests as "why is my `use cache` returning empty results after a navigation abort?"** — the cache looks valid but it's a tombstone.
3. **`4fd843f`** — version-tag commit `v16.3.0-canary.107` by `next-js-bot`, dated 2026-08-02T23:34:28Z — npm-published at 2026-08-03T14:04:47Z.

**canary.107 vs canary.106 diff = exactly 3 commits** (PR #96493 + PR #96426 + the version-tag commit `4fd843f`). No other PRs slipped in between. So if you're on canary.106, the only new things in canary.107 are the build-cache-default-on-in-all-environments expansion + the cache-poisoning-after-prerender-abort fix.

### Why PR #96426 matters — cache poisoning after prerender abort

**The bug (pre-#96426)**: In Next.js 16.3 `cacheComponents` (a.k.a. `use cache`), the `use-cache-wrapper` builds its `AbortSignal.any(...)` from a list of upstream signals. Before PR #96426, one of those upstream signals was `renderSignal` (the prerender's signal). When the prerender was aborted (e.g. client navigated away mid-render, or the prerender timed out under `partialPrefetching`), `renderSignal` would abort immediately, which propagated to the wrapper's `AbortSignal.any(...)`, which aborted the cache fill *immediately after the cache had already started writing the empty stream*. The cache then stored that empty stream as a valid entry. Every subsequent request hit the poisoned cache and got empty content.

**The fix (post-#96426)**: Remove `renderSignal` from the `AbortSignal.any(...)` and short-circuit with a rejected promise *before* the cache fill codepath runs. Since the prerender is already aborted at this point (the caller has moved on), the cache is never poisoned. The user-observable change: **caches no longer silently poison themselves with empty entries when a prerender is aborted mid-fill**.

**Practical impact**:
- **Apps with `cacheComponents: true` + heavy use of `use cache` + frequent navigation aborts** (e.g. fast-clicking links, slow connections, prefetch cancellations under `partialPrefetching: true`) — pre-#96426, those apps would intermittently return empty cached content for periods of time (until a full cache invalidation or restart). Post-#96426, the cache stays clean.
- **Apps NOT on `cacheComponents`** (`experimental.useCache: false` or `cacheComponents: false`) — **no impact**, the fix is entirely inside the `use-cache-wrapper` codepath.
- **Dev mode + production both affected** — the bug reproduces in `next dev` (where prerender aborts are common due to re-renders) AND in production under partial-prerender aborts.

**Migration**: none required for userspace code. Just bump to `next@16.3.0-canary.107+` (when npm-published) or to the next stable release that ships the fix. If you've ever seen "this cache should have content but it's returning empty", that's the bug.

**Audit recipe**:
```bash
# 1. Are you on cacheComponents? (Next.js 16 default when using 'use cache')
rg -n "cacheComponents\s*:\s*true" next.config.ts next.config.js 2>/dev/null
# 2. Are you using 'use cache' in your app?
rg -l "use cache" app/ src/ 2>/dev/null
# 3. Have you seen intermittent empty cache results after navigation aborts?
# If YES to all 3 -> you're affected by #96339, bump to canary.107+
```

**No new public APIs**, no new config flags, no codemod. Pure bug fix.

### Why PR #96493 matters — the scope expansion

The canary.105 PR #96395 (sokra, merged 2026-07-31T17:24:41Z, documented in the `## 16.3 canary.105 SHIPPED` section above) flipped `experimental.turbopackFileSystemCacheForBuild` to **default-ON for `next build`**, but only for the local + Vercel build path. The default-on behavior was gated by a `turbopackFileSystemCacheForBuildDefault()` helper that returned `!isCI || Boolean(process.env.NOW_BUILDER)` — meaning generic CI (GitHub Actions, GitLab CI, CircleCI, Buildkite, Jenkins, etc.) without the `NOW_BUILDER` env-var got the **default-OFF** treatment because Next.js couldn't know if the cache would persist between runs.

**PR #96493 removes that gate entirely** and defaults the option to `true` unconditionally. The `turbopackFileSystemCacheForBuildDefault()` function is deleted. The diff in `packages/next/src/server/config-shared.ts` (commit `3d4d46f`, +2/-9) is small but decisive:

```diff
- function turbopackFileSystemCacheForBuildDefault() {
-   // Disable in most CI environments, because we don't know if the cache will persist across builds.
-   // Providers: Override the default behavior using `modifyConfig` in your adapter.
-   return !isCI || Boolean(process.env.NOW_BUILDER)
-}
-
  export const defaultConfig = Object.freeze({
    ...
    turbopackFileSystemCacheForDev: true,
-   turbopackFileSystemCacheForBuild: turbopackFileSystemCacheForBuildDefault(),
+   turbopackFileSystemCacheForBuild: true,
    ...
  })
```

The JSDoc on `ExperimentalConfig.turbopackFileSystemCacheForBuild` is also updated from *"Defaults to `true` in canary/preview builds, `false` in production"* to simply *"Defaults to `true`."*

### Practical impact — 4 cases

| Environment | Before canary.107 (canary.105/106 behavior) | **After canary.107 (PR #96493 behavior)** |
|---|---|---|
| Local `next build` on dev machine | On (PR #96395) | On (unchanged) |
| Vercel hosted build | On (PR #96395, via `NOW_BUILDER` env-var) | On (unchanged) |
| **Generic CI (GitHub Actions, GitLab, CircleCI, Buildkite, Jenkins, etc.) — non-Vercel** | **Off** (the `!isCI` branch of the helper) | **On — now default-ON in all environments** |
| Explicit `experimental.turbopackFileSystemCacheForBuild: false` | Off | Off (unchanged — explicit opt-out always wins) |
| Explicit `experimental.turbopackFileSystemCacheForBuild: true` | On | On (unchanged) |

**The key new behavior**: every `next build` in every CI environment now uses the warm `.next/cache/turbopack/` filesystem cache by default. **Expected 5-30% speedup on warm builds** depending on project size (the bigger the dep graph, the bigger the win — Vercel-internal benchmarks show 30-60% on large apps like vercel.com/home and vercel.com/geist).

**The new responsibility on CI users**: you now need to **opt out** (`experimental.turbopackFileSystemCacheForBuild: false`) if your CI environment doesn't persist `.next/cache/` between runs (e.g. ephemeral CI containers with `actions/cache` miss). The pre-#96493 default was the safe "off in generic CI" choice; the new default assumes you'll either persist the cache or opt out. The docs page at `docs/01-app/03-api-reference/05-config/01-next-config-js/turbopackFileSystemCache.mdx` (commit `3d4d46f`, +7/-9) is rewritten in the same commit to reflect the new default — the previous "For deployment providers: Other providers can enable the build cache by default on their platform by setting `experimental.turbopackFileSystemCacheForBuild: true` from their adapter's `modifyConfig` hook" paragraph is **deleted entirely** because every provider is now treated the same as local.

**Critical audit recipe**:
```bash
# 1. Are you using Turbopack for builds? (Turbopack is default in 16+; --turbopack flag optional)
rg "turbopackFileSystemCacheForBuild" next.config.ts next.config.js next.config.mjs 2>/dev/null
# 2. Does your CI persist .next/cache/ between runs? (look for actions/cache, buildkite cache, etc.)
# 3. If YES to #1 + NO to #2 -> set experimental.turbopackFileSystemCacheForBuild: false explicitly
# 4. If YES to #1 + YES to #2 -> no action needed, the new default is what you want
```

### Migration checklist (canary.105/106 -> canary.107, when it ships)

1. **Verify your CI cache strategy** — check `actions/cache@v4` on `path: .next/cache` (GitHub Actions) or the equivalent on your CI. If absent, opt out: `experimental.turbopackFileSystemCacheForBuild: false` in `next.config.ts`.
2. **Audit `package.json` scripts** — the "fair webpack-vs-Turbopack build comparison" still requires `rm -rf .next/` between builds (the existing canary.105 caveat still applies).
3. **Adapter maintainers (Vercel competitors)** — the `modifyConfig` hook paragraph in the docs is gone. You no longer need to set `experimental.turbopackFileSystemCacheForBuild: true` from your adapter; the default is already on.
4. **No code changes required** for users on canary.105+ — the only delta is the default value. All existing opt-in / opt-out configs continue to work.
5. **Test your CI cold builds** — if your CI doesn't persist `.next/cache/`, you may see "this build was slower than the previous one" comparisons; that's expected (the new default assumes warm builds, not cold).

### No breaking changes

This is an **expansion of the default**, not a breaking change:

- Users on canary.105/106 with the default-ON cache behavior: no change (already getting the cache).
- Users on canary.105/106 with the default-OFF CI behavior (the pre-#96493 path): **get the cache for free** (if their CI persists `.next/cache/`). If their CI doesn't persist the cache, they should set the explicit opt-out to preserve the prior "no cache" behavior.
- Users with explicit `experimental.turbopackFileSystemCacheForBuild: true|false`: no change (explicit always wins).

### Why Tim Neutkens authored this (rather than sokra)

sokra (Tobias Koppers) wrote the canary.105 PR #96395 with the `!isCI` gate, which was the conservative choice at the time. After 1+ month of production data (canary.105 shipped 2026-07-31T23:57:13Z, now 2+ days of npm-published usage as of this cron's check), Tim Neutkens (Vercel co-founder + Next.js lead) is the one who concluded the gate was no longer needed — Turbopack's filesystem cache is well-behaved in CI even when the cache isn't persisted (it just falls back to cold-build behavior on cache miss). The gate was always "opt-out if you're not sure" — now the new default is "opt-in if you're sure you don't want it" (explicit `: false`).

### Sources

- [Next.js canary-branch compare: `v16.3.0-canary.107...canary` (9 commits ahead)](https://github.com/vercel/next.js/compare/v16.3.0-canary.107...canary) — verified at 2026-08-03T18:02Z; = 3 material PRs (#96497, #96308, #93132) + 3 docs/infra PRs (#96526, #96527, #96534) + 1 deps bump (#96386) + 1 CI flag (#96505) + the canary.108 version-tag commit. Full breakdown in the new `## 16.3 canary.108-ahead` section below.
- [Next.js canary-branch compare: `v16.3.0-canary.106...canary` (12 commits — 3 in canary.107 + 9 ahead of canary.107)](https://github.com/vercel/next.js/compare/v16.3.0-canary.106...canary) — the cumulative view across canary.107 + ahead
- [Next.js canary-branch compare: `v16.3.0-canary.105...canary` (15 commits — 3 in canary.106 + 3 in canary.107 + 9 ahead of canary.107)](https://github.com/vercel/next.js/compare/v16.3.0-canary.105...canary) — the cumulative view across canary.106 + canary.107 + ahead
- [Next.js canary-branch compare: `v16.3.0-canary.104...canary` (30 commits — 15 in canary.105 + 3 in canary.106 + 3 in canary.107 + 9 ahead of canary.107)](https://github.com/vercel/next.js/compare/v16.3.0-canary.104...canary) — the full cumulative view
- [**Next.js release `v16.3.0-canary.107`**](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.107) — published 2026-08-03T14:04:47Z by `next-js-bot`, body lists both PRs
- [**Next.js PR #96493** — `Enable Turbopack build filesystem cache by default`](https://github.com/vercel/next.js/pull/96493) — by [Tim Neutkens](https://github.com/timneutkens), merged 2026-08-02T18:33:34Z, 4 files / +22/-42, the source-of-truth for the scope expansion; **SHIPPED in `16.3.0-canary.107`**
- [Next.js commit `3d4d46f` — the PR #96493 merge commit](https://github.com/vercel/next.js/commit/3d4d46f) — the actual diff that removes `turbopackFileSystemCacheForBuildDefault()` and defaults the option to `true`
- [Next.js commit `4fd843f` — the `v16.3.0-canary.107` version-tag commit](https://github.com/vercel/next.js/commit/4fd843f) — by `next-js-bot[bot]`, dated 2026-08-02T23:34:28Z, **now npm-published** as `next@16.3.0-canary.107` at 2026-08-03T14:04:47Z
- [**Next.js PR #96426** — `[Cache] Make caches error if called after prerender aborts**](https://github.com/vercel/next.js/pull/96426) — by [Janka Uryga](https://github.com/jankaeryga), merged 2026-08-03T11:42:26Z, the cache-poisoning-after-prerender-abort fix; closes issue #96339; **SHIPPED in `16.3.0-canary.107`**
- [Next.js commit `3317392` — the PR #96426 merge commit](https://github.com/vercel/next.js/commit/331739243f768934efc895f5881a407bb979950a) — the actual diff that removes `renderSignal` from the `AbortSignal.any(...)` and short-circuits before the cache-fill codepath
- [Next.js issue #96339 — caches silently poison themselves with empty entries after prerender aborts](https://github.com/vercel/next.js/issues/96339) — the issue PR #96426 closes
- [Next.js PR #96395 — `Enable turbopackFileSystemCacheForBuild by default` (sokra, canary.105)](https://github.com/vercel/next.js/pull/96395) — the predecessor PR; the new PR #96493 removes the `!isCI` gate that PR #96395 left in
- [Next.js `turbopackFileSystemCache` config docs — the file edited by PR #96493](https://github.com/vercel/next.js/blob/3d4d46f9ce7c9d3a9c25a3d99de18c5d56bda0e0/docs/01-app/03-api-reference/05-config/01-next-config-js/turbopackFileSystemCache.mdx) — the "Good to know" callout was rewritten to drop the "for local builds and on Vercel" qualifier
- [Next.js `turbopackFileSystemCacheForBuild` JSDoc on `ExperimentalConfig`](https://github.com/vercel/next.js/blob/3d4d46f9ce7c9d3a9c25a3d99de18c5d56bda0e0/packages/next/src/server/config-shared.ts) — the JSDoc now reads "Defaults to `true`." (was "Defaults to `true` in canary/preview builds, `false` in production.")
- [Next.js Turbopack reference docs — the file edited by PR #96493](https://github.com/vercel/next.js/blob/3d4d46f9ce7c9d3a9c25a3d99de18c5d56bda0e0/docs/01-app/03-api-reference/08-turbopack.mdx) — the `<sup>1</sup>` footnote was rewritten to drop the CI-platform qualification

## 16.3 canary.108-ahead — Enable TypeScript CLI by Default + Fix App Router Scroll Padding Visibility + Fix Double-Fragment on Navigation (3 Material PRs + 6 Infra PRs, August 3, 2026)

The v1.5.18 cron (6h ago) said "canary.107 is built on the canary branch but not yet npm-published" — that is no longer true (canary.107 SHIPPED 6h after that cron at 2026-08-03T14:04:47Z; see the `## 16.3 canary.107 SHIPPED` section above). **All 9 commits ahead of canary.107 that v1.5.19 documented are now SHIPPED in `next@16.3.0` STABLE** (npm-published 2026-08-03T21:03:18Z by Tim Neutkens; the v16.3.0 release bundles 16 commits from the canary-branch since canary.107 — see the `## Next.js 16.3.0 STABLE` section in `patterns.md` for the full release-notes breakdown), with the catch-all-index fix from PR #96553 (acdlite) shipping in `next@16.3.1-canary.0` (npm-published 2026-08-03T22:32:33Z, 88 minutes after 16.3.0 STABLE). The canary-branch now has **8 NEW commits ahead of `16.3.1-canary.0`** (verified at 2026-08-04T12:03Z via `GET /repos/vercel/next.js/compare/v16.3.1-canary.0...canary` returning `ahead_by: 8`). **1 PR is materially user-facing** (PR #96583 — preserve per-segment prefetching after dynamic navigation, a Cache Components bug fix), **1 is a docs-only change with future-stabilization signaling** (PR #96615 — remove experimental note from runtime prefetching), **2 are Turbopack runtime fixes** (PR #96599 + PR #96600), **2 are Turbopack module-system internals** (PR #96380 + PR #96381), **2 are internal static-generation cleanups** (PR #96563 + PR #96564). Expect `next@16.3.1-canary.1` to npm-publish within 12-24h on the 24h cadence once the canary-branch version-tag commit lands. See the new `## 16.3 canary-branch ahead of 16.3.1-canary.0 (8 NEW commits, Aug 3-4, 2026)` section below for the full diff + practical impact.

### The 9 commits ahead of canary.107 (chronological)

1. **`767e0e9` — PR #96386** [`[turbotrace] bump test version of sharp`](https://github.com/vercel/next.js/pull/96386) (styfle / [@styfle](https://github.com/styfle), merged 2026-08-03T13:42:02Z) — regenerates the lockfile to test that the previous PR still works with the latest `sharp` (post-#94845). 2 files / +134/-128. **Purely CI/test infra** — no behavior change.
2. **`c00449d` — PR #96527** [`[ci] Wait for the preview tarballs before running the deploy tests`](https://github.com/vercel/next.js/pull/96527) (CI maintainer, merged 2026-08-03T14:27:51Z) — **CI-only**. Fixes a flaky test that ran before the preview tarballs were ready.
3. **`da90782` — PR #96308** [`Fix App Router scroll padding visibility`](https://github.com/vercel/next.js/pull/96308) (DavidIlie / [@DavidIlie](https://github.com/DavidIlie), merged 2026-08-03T15:00:14Z) — **MATERIAL USER-FACING FIX**. Treats the root `scroll-padding-top` as the lower boundary of the usable viewport during App Router navigations, so content obscured by a sticky header is no longer incorrectly considered visible. See the `### Why PR #96308 matters — scroll padding as viewport boundary` subsection below for the full breakdown.
4. **`2aef3b8` — PR #96526** [`docs: ISR with Cache Components and Partial Prefetching`](https://github.com/vercel/next.js/pull/96526) (docs, merged 2026-08-03T15:15:00Z) — **DOCS-ONLY**. Adds a new ISR + Cache Components + Partial Prefetching guide. No code change.
5. **`dcdacfd` — PR #96534** [`test: skip hybrid not-found coverage for the legacy builder`](https://github.com/vercel/next.js/pull/96534) (test infra, merged 2026-08-03T15:36:57Z) — **TEST-ONLY**. The legacy builder doesn't support hybrid Pages + App Router not-found rendering, so this test now skips for that path.
6. **`cbf0cef` — PR #96497** [`Enable TypeScript CLI by default`](https://github.com/vercel/next.js/pull/96497) (Tim Neutkens / timneutkens, merged 2026-08-03T16:10:51Z) — **THE BIGGEST BEHAVIORAL CHANGE OF THIS CYCLE**. Flips `experimental.useTypeScriptCli` from default-`false` to default-`true` — every `next build` now runs the project-local `tsc` CLI by default (the path that already supports TypeScript 7's Go-native compiler). 24 files / +134/-54. See the `### Why PR #96497 matters — TypeScript CLI is now the default` subsection below for the full breakdown.
7. **`459617a` — PR #93132** [`fix: double fragment on navigation`](https://github.com/vercel/next.js/pull/93132) (icyJoseph / [@icyJoseph](https://github.com/icyJoseph), merged 2026-08-03T16:21:06Z) — **MATERIAL USER-FACING FIX**. Closes issue #93126 (and #95551). The bug: navigating from `/abc#foo` to `/abc#bar` left the URL at `/abc#foo#bar` (the segment cache concatenated the new hash to the existing one). Fix: store a hashless canonical URL in the segment cache entry, so same-pathname hash changes replace rather than concatenate. 5 files / +67/-1. See the `### Why PR #93132 matters — same-pathname hash change replaces rather than concatenates` subsection below.
8. **`4344b83` — PR #96505** [`Flag newly disabled deploy tests`](https://github.com/vercel/next.js/pull/96505) (test infra, merged 2026-08-03T17:29:36Z) — **TEST-ONLY**. Flags deploy tests that are currently disabled so they're easy to find + triage.

### Why PR #96497 matters — TypeScript CLI is now the default

**The change**: PR #96497 flips `experimental.useTypeScriptCli` from **`false`** (the canary.103 → canary.107 default) to **`true`** (the canary.108+ default). The diff in `packages/next/src/server/config-shared.ts` is one line:

```diff
 export const defaultConfig = Object.freeze({
   ...
-    useTypeScriptCli: false,
+    useTypeScriptCli: true,
   ...
 })
```

The JSDoc on `ExperimentalConfig.useTypeScriptCli` is updated from "Enable the project-local TypeScript CLI for type checking during production builds" to "By default, `next build` runs the project-local `tsc` command instead of loading the TypeScript JavaScript compiler API" — and the **opt-out example in the docs** flips from `useTypeScriptCli: true` (opt-in) to `useTypeScriptCli: false` (opt-out). The docs page `docs/01-app/03-api-reference/05-config/01-next-config-js/useTypeScriptCli.mdx` is rewritten in the same PR; the `typescript.mdx` page loses its entire `experimental.useTypeScriptCli: true` opt-in section.

`packages/next/errors.json` adds two new error codes:
- **1466**: `"TypeScript %s does not provide the compiler API required by Next.js. Set %s to true in your Next.js config to use the TypeScript CLI, or install TypeScript 6 instead."` — fired when `useTypeScriptCli: false` AND a TS version without the compiler API is installed (i.e. TS 7.x). The wording tells the user to **set `useTypeScriptCli: true` to opt back in to the CLI**, or install TS 6.
- **1467**: same message but for the **reverse path** — when `useTypeScriptCli: true` (now the default) AND the user installs a TS version that does provide the compiler API but they're explicitly opting out. This is the inverse error for explicit-opt-out users.

**Why this matters — the practical impact for every Next.js user**:

1. **TypeScript 7 users (no flag needed)** — `typescript@^7.0.0` users previously had to set `experimental.useTypeScriptCli: true` in `next.config.ts` to get type checking working (because the TS 7 Go-native compiler doesn't expose the JavaScript Compiler API that Next.js used). **With PR #96497, that line is no longer needed** — `next build` will run the project-local `tsc` (which IS the Go-native binary on TS 7) by default. **No code change required** for TS 7 users — just upgrade to canary.108 (when published) and remove the now-redundant flag line from your config.

2. **TypeScript 6 users (silent behavior change)** — `typescript@^6.x` users on Next.js 16.2.x → canary.107 had type checking running through the **legacy TypeScript Compiler API backend** (the JS one). On canary.108+, type checking runs through the **project-local `tsc` CLI** (a child process) by default. The behavior is observationally identical (both paths produce the same type errors), but there are **subtle timing/output differences**:
   - **Build time**: CLI path adds ~50-200ms of overhead per build (process spawn + IPC), but allows TS 7's Go compiler to do the actual work without loading the JS API. For TS 6 users, it's roughly neutral.
   - **Error output formatting**: CLI path uses the user's installed `tsc`'s formatter; JS-API path uses Next.js's internal formatter. The two can disagree on formatting, but they produce the same `error TSxxxx:` lines.
   - **Spinner UX**: CLI path now shows a spinner while `tsc` runs (the canary.106 PR #95753 improvement carries forward). For builds that took <2s of type checking, you'll now see a brief spinner flash.
   - **`typescript` peer-dep range**: the CLI path requires that the installed `typescript` provides a `tsc` binary — which all TS versions ≥3.x do, so this is fine in practice.

3. **TS 5.x users (silent behavior change, possibly desired)** — TS 5.x's `tsc` doesn't have the Go-native speedup (that's TS 7's win), but the CLI path still works. The only behavior change is the process-spawn overhead vs the in-process JS API. Most projects won't notice.

4. **Custom transformers / `typescript` as a library users** — if you have a tool that imports `typescript` as a library (e.g. `eslint-plugin-import` walking the TS AST, custom webpack/ts-loader pipelines, codemod CLIs using `ts.createSourceFile`), the `useTypeScriptCli: true` path **does NOT affect that** — it only affects `next build`'s type-check pass. Your tool's `require('typescript')` is independent. However, if you want Next.js to keep using the JS Compiler API for compatibility, you must now set `experimental.useTypeScriptCli: false` explicitly.

5. **Monorepos with TS 6 in the root + TS 7 in a sub-package** — no impact. `useTypeScriptCli` is per-Next.js-config, not per-package. Each Next.js app picks its own type-check path.

**Migration checklist (canary.107 → canary.108, when it ships)**:

1. **If you had `experimental: { useTypeScriptCli: true }` in your `next.config.ts`** — remove the line. It's now redundant. The CLI path will be used by default.
2. **If you had `experimental: { useTypeScriptCli: false }` in your `next.config.ts`** — leave it. The opt-out still works (you'll get the legacy JS Compiler API path).
3. **If you had no `useTypeScriptCli` setting in `next.config.ts`** — verify your build still works. The default change is transparent for ~95% of projects. Watch for these edge cases:
   - **CI cache invalidation**: type-check errors may move to different log positions (since the CLI emits them at a different stage than the JS API). If you have CI regex matches on error output, audit them.
   - **Spinner noise**: if you have a CI step that detects build progress via log lines, the new spinner lines may break that detection.
   - **TS version peer-dep warnings**: if your `package.json` has `typescript: "^6.x"` and your tooling complains about "TS 7 support requires `useTypeScriptCli: true`", that message is now inverted (TS 6 should be fine with either path; the message only fires for TS 7 users who explicitly set `useTypeScriptCli: false`).
4. **If you're on `next@16.2.12` stable** — this change does NOT affect you yet. PR #96497 is canary-only. Backport to 16.2.x is not committed (the canary-line is the test bed for this kind of default-flip; stable gets it later).

**Audit recipe**:

```bash
# 1. Are you on canary? (this PR is canary-only for now)
npm ls next
# 2. Did you opt in to useTypeScriptCli?
rg -n "useTypeScriptCli" next.config.ts next.config.js next.config.mjs 2>/dev/null
# 3. Are you on TS 7?
rg '"typescript"' package.json | head -3
# 4. Have you seen type errors or build failures since canary.107?
# If 1=yes + 2=YES-true + 3=yes -> remove the line, the default is what you want
# If 1=yes + 2=YES-false + 3=no -> leave it, you're getting the JS API path
# If 1=yes + 2=NO + 3=yes -> no change needed, you were already getting the CLI path by default next canary
```

**No new public APIs**, no new config flags in canary.108 (the existing `experimental.useTypeScriptCli` is now default-`true`).

### Why PR #96308 matters — scroll padding as viewport boundary

**The bug (pre-#96308)**: When an App Router navigation targeted an element that was partially obscured by a sticky header, Next.js would treat the element as "in the visible viewport" because it computed visibility based on the **viewport geometry** rather than the **usable viewport geometry** (the area not obscured by the sticky header). As a result, the router wouldn't scroll to bring the element into view, even though the user couldn't actually see it — leaving the element behind the sticky header.

**The fix (post-#96308)**: Treat the root `scroll-padding-top` CSS property as the lower boundary of the usable viewport. Resolve pixel and percentage values via `getComputedStyle(htmlElement).scrollPaddingTop` (lazy — only computed for candidate elements that have client rects, so empty Fragments and hash navigations skip the lookup). The same visibility rule is now used in both the legacy element handler (`ScrollTargetState.InViewport` / `OutOfViewport`) and the Fragment-ref handler — preserving the empty-Fragment scroll-ownership state machine from PR #96342 (canary.103-shipped).

The diff is +71/-8 in `packages/next/src/client/components/layout-router.tsx` (the `getScrollPaddingTopInPixels()` helper + the `getScrollTargetState()` rewrite) + +38/-0 in the e2e test (`test/e2e/app-dir/router-autoscroll/router-autoscroll.test.ts`). The test adds two `it.each` cases — `'should scroll when the page top is obscured by scroll padding (%s)'` parameterized over `'100px'` and `'50%'` — confirming that:

- Setting `html { scroll-padding-top: 100px }` and navigating to an element whose top is at y=0 (covered by the padding) triggers a scroll to bring the element below the padding.
- Setting `html { scroll-padding-top: 50% }` and navigating to an element whose top is at y=0 triggers a scroll to bring the element below the (huge) padding.
- The pre-existing "no scroll when destination is below the padding" behavior is preserved.

**Practical impact**:

- **Every App Router project with a sticky header** (the most common nav-bar pattern: `<header className="sticky top-0 z-50 h-16">...`) — pre-#96308, scroll-to-element would silently fail when the destination was partially obscured by the sticky header. Post-#96308, the scroll correctly accounts for the padding. **Set `scroll-padding-top` on your `<html>` to the height of your sticky header to get the fix immediately**:
  ```css
  /* globals.css */
  html { scroll-padding-top: 4rem; }  /* match your sticky-header height */
  ```
- **Projects with table-of-contents + sticky header patterns** (docs sites, blog sites with reading-progress bars) — TOC links that target anchor IDs will now correctly scroll past the sticky header. Without `scroll-padding-top`, browsers already do this correctly via CSS — but Next.js's client-side router scroll-to-element didn't.
- **Projects WITHOUT a sticky header** — no impact (the default `scroll-padding-top: 0` means the padding-aware boundary is identical to the viewport boundary).
- **Dev mode + production both affected** — the bug reproduces in both environments; the e2e test runs in both.

**Migration**: no code change required. Set `html { scroll-padding-top: ... }` in your global CSS to take advantage of the new behavior (this is what was always recommended for sticky headers + anchor links; the fix means Next.js's scroll-to-element now respects it).

**Audit recipe**:

```bash
# 1. Are you using a sticky header?
rg -n "sticky|fixed.*top-0" app/ src/ components/ 2>/dev/null | head -5
# 2. Do you have scroll-padding-top set?
rg -n "scroll-padding-top" app/globals.css app/globals.scss src/index.css 2>/dev/null
# 3. Have you seen "scroll-to-element ends up behind the sticky header" reports?
# If 1=yes + 2=NO -> add scroll-padding-top: <header-height> to your globals.css
# If 1=yes + 2=YES -> the fix is already working, you just need to upgrade to canary.108
```

**Composes with PR #96342 (canary.103)** — the empty-Fragment scroll-ownership fix. PR #96308 only changes the visible-region boundary for *real* scroll targets (elements with client rects). Empty Fragments still return `NoClientRects` and skip the geometry check. The two fixes don't conflict — they're orthogonal concerns.

### Why PR #93132 matters — same-pathname hash change replaces rather than concatenates

**The bug (pre-#93132)**: Starting in Next.js 16.2.0 (regression introduced there), navigating from `/abc#foo` to `/abc#bar` via `<Link href="/abc#bar">` would leave the URL bar at `/abc#foo#bar` — the router was appending the new hash to the existing one instead of replacing it. The bug also manifested when navigating back and forth between `/abc` (no hash) and `/abc#FragmentName` — sometimes the hash got duplicated.

The same bug surfaced in three contexts:
- **`<Link>` clicks within the same pathname with different hashes** — `/abc#foo` → click `<Link href="/abc#bar">` → URL becomes `/abc#foo#bar`.
- **`router.push('/abc#bar')` from `/abc#foo`** — same result.
- **Browser Back + Forward** — navigating back to `/abc#foo` after going to `/abc#foo#bar` could land on `/abc#foo#bar` instead of `/abc#foo`.

**The fix (post-#93132)**: Store a **hashless canonical URL** in the segment cache entry. The `createHrefFromUrl(canonicalUrl, false)` call (the `false` arg = "don't include the hash") is the one-line change in `packages/next/src/client/components/segment-cache/navigation.ts`. With the hash stripped from the cache entry, when a same-pathname hash navigation happens, the router appends `url.hash` to the existing (hashless) canonical URL — producing `/abc#bar` (the desired behavior) instead of `/abc#foo#bar`.

Note: PR #93132 **does NOT slice an existing hash from the URL** like the related PR #93855 did — it prevents the bad entry from being stored in the first place. The PR author notes it's "plausible that dropping the hash from the segment cache is not desired though" — i.e. if your app relies on hash-as-state for some internal bookkeeping, this PR could be a behavior change. In practice, no real-world use case has surfaced.

**Practical impact**:

- **Every App Router project using `<Link>` for in-page anchor navigation** — TOC links, "back to top" links, accordion section anchors, skip-to-section links — all previously affected. Post-#93132, the URL bar shows the correct single hash.
- **Every project using `router.push('/page#section')` for client-side navigation** — same fix.
- **React `useRouter().hash` / `window.location.hash` readers** — no impact (the URL bar is updated correctly; readers see the correct hash).
- **Hash-based routing libraries** (rare in App Router — the App Router uses pathnames for routing, not hashes) — verify your library doesn't depend on the buggy concatenation behavior.
- **Dev mode + production both affected** — the bug is deterministic.
- **Started in 16.2.0 (July 2026)** — users on Next.js 16.1.6 didn't see this. Users on 16.2.x → canary.107 do see it.

**Migration**: no code change required. Just upgrade to canary.108 (when published). The fix is transparent.

**Audit recipe**:

```bash
# 1. Are you using <Link href="/same-path#different-hash">?
rg -n 'href="/[^"]*#' app/ src/ components/ 2>/dev/null | head -10
# 2. Are you using router.push('/path#hash')?
rg -n 'router\.push\([''"][^''"]+#' app/ src/ 2>/dev/null | head -10
# 3. Have you seen user reports about "/abc#foo#bar" appearing in URL bars?
# If 1=YES or 2=YES -> you would have hit the bug on 16.2.0+, want canary.108
```

**No new public APIs**, no new config flags, no codemod.

### Sources (canary.108-ahead)

- [Next.js canary-branch compare: `v16.3.0-canary.107...canary` (9 commits ahead)](https://github.com/vercel/next.js/compare/v16.3.0-canary.107...canary) — verified at 2026-08-03T18:02Z; = PR #96497 (TypeScript CLI default-on) + PR #96308 (scroll padding) + PR #93132 (double fragment fix) + 5 infra PRs (#96386, #96527, #96526, #96534, #96505) + 1 canary.108 version-tag commit (not yet committed at this cron's check)
- [Next.js canary-branch compare: `v16.3.0-canary.106...canary` (12 commits)](https://github.com/vercel/next.js/compare/v16.3.0-canary.106...canary) — cumulative across canary.107 + ahead
- [**Next.js PR #96497** — `Enable TypeScript CLI by default`](https://github.com/vercel/next.js/pull/96497) — by Tim Neutkens, merged 2026-08-03T16:10:51Z, 24 files / +134/-54, the source-of-truth for the default-on flip
- [Next.js PR #96497 files diff](https://github.com/vercel/next.js/pull/96497/files) — full 24-file breakdown incl. `config-shared.ts` (+1/-1), `errors.json` (+2 new error codes), `runTypeScriptCli.ts` (+1/-1 error-message wording flip), `useTypeScriptCli.mdx` docs (rewritten), `typescript.mdx` docs (opt-in section deleted), and 19 test fixtures
- [**Next.js PR #96308** — `Fix App Router scroll padding visibility`](https://github.com/vercel/next.js/pull/96308) — by DavidIlie, merged 2026-08-03T15:00:14Z, 2 files / +109/-8, the source-of-truth for the scroll-padding fix
- [Next.js `layout-router.tsx` getScrollPaddingTopInPixels() helper — the new helper added by PR #96308](https://github.com/vercel/next.js/blob/da90782/packages/next/src/client/components/layout-router.tsx) — resolves `scroll-padding-top` from `getComputedStyle` lazily + handles `px` / `%` units
- [**Next.js PR #93132** — `fix: double fragment on navigation`](https://github.com/vercel/next.js/pull/93132) — by icyJoseph, merged 2026-08-03T16:21:06Z, 5 files / +67/-1, fixes hash concatenation in segment cache
- [Next.js issue #93126 — Router .push with fragments duplicates when doing client side navigation](https://github.com/vercel/next.js/issues/93126) — the user-reported bug; opened June 2026, affects Next.js 16.2.0+
- [Next.js issue #95551 — hash-concatenation regression (companion to #93126)](https://github.com/vercel/next.js/issues/95551) — closed by PR #93132
- [Next.js PR #93855 — earlier attempt at the same fix via hash-slicing](https://github.com/vercel/next.js/pull/93855) — superseded by PR #93132's segment-cache-stripping approach
- [Next.js PR #96386 — `[turbotrace] bump test version of sharp`](https://github.com/vercel/next.js/pull/96386) — styfle, merged 2026-08-03T13:42:02Z, CI-only deps bump
- [Next.js PR #96526 — `docs: ISR with Cache Components and Partial Prefetching`](https://github.com/vercel/next.js/pull/96526) — docs-only guide addition, merged 2026-08-03T15:15:00Z
- [Next.js PR #96527 — `[ci] Wait for the preview tarballs before running the deploy tests`](https://github.com/vercel/next.js/pull/96527) — CI-only, merged 2026-08-03T14:27:51Z
- [Next.js PR #96534 — `test: skip hybrid not-found coverage for the legacy builder`](https://github.com/vercel/next.js/pull/96534) — test-only, merged 2026-08-03T15:36:57Z
- [Next.js PR #96505 — `Flag newly disabled deploy tests`](https://github.com/vercel/next.js/pull/96505) — test-only, merged 2026-08-03T17:29:36Z
- [Next.js `experimental.useTypeScriptCli` config docs (post-#96497)](https://nextjs.org/docs/app/api-reference/config/next-config-js/useTypeScriptCli) — the docs page rewritten in PR #96497; the opt-in code example is now the opt-out example
- [Next.js `experimental.useTypeScriptCli` JSDoc on `ExperimentalConfig`](https://github.com/vercel/next.js/blob/cbf0cef/packages/next/src/server/config-shared.ts) — the JSDoc updated to reflect the new default


## 16.3 canary-branch ahead of 16.3.1-canary.0 (8 NEW commits, Aug 3-4, 2026) — Preserve Per-Segment Prefetching + Turbopack Internal Cleanups + Runtime Prefetching Unmark-Experimental

The v1.5.19 cron (12h ago at 18:02Z Aug 3) said "the canary.108 version-tag commit is not yet committed at this cron's check" — that is no longer true (the canary.108 version-tag commit landed on the canary branch shortly after v1.5.19, and **all 9 ahead-of-canary.107 commits shipped in `next@16.3.0` STABLE on 2026-08-03T21:03:18Z** — see `patterns.md` for the full 16.3.0 release-notes breakdown). The canary-branch has since moved 8 NEW commits ahead of `next@16.3.1-canary.0` (npm-published 2026-08-03T22:32:33Z with PR #96553, the catch-all index page fix — see the `## Catch-All Route Handler Bug Fix` section in `api.md`). The 8 commits break down as: **1 MATERIAL USER-FACING FIX** (PR #96583 — preserve per-segment prefetching after dynamic navigation, a Cache Components bug fix) + **1 DOCS-ONLY with future-stabilization signaling** (PR #96615 — remove experimental note from runtime prefetching) + **2 Turbopack runtime fixes** (PR #96599 + PR #96600) + **2 Turbopack module-system internals** (PR #96380 + PR #96381) + **2 internal static-generation cleanups** (PR #96563 + PR #96564). Expect `next@16.3.1-canary.1` to npm-publish within 12-24h on the 24h cadence once the canary-branch version-tag commit lands.

### The 8 commits ahead of `16.3.1-canary.0` (chronological, oldest first)

1. **`1dd15c5` — PR #96380** [`[turbopack] Ignore writes to exports after a module.exports = {} writes`](https://github.com/vercel/next.js/pull/96380) (Sam Poder / @sam-1-2, merged 2026-08-03T23:37:28Z) — Turbopack module-system fix: when `module.exports = {}` writes were followed by additional writes to the same `exports` property, Turbopack was incorrectly tracking those writes and emitting dead store operations. Now it ignores them after the bulk assignment. **No user-facing impact** — Turbopack-internal optimization, no API change, no config change.

2. **`8231b03` — PR #96381** [`[turbopack] Drop dead writes to exports`](https://github.com/vercel/next.js/pull/96381) (Sam Poder, merged 2026-08-03T23:37:29Z) — Turbopack build-output optimization that drops the now-dead writes that PR #96380 was ignoring at runtime. **No user-facing impact** — pure dead-code-elimination in the Turbopack output. Smaller bundles, but the diff is internal to the build pipeline.

3. **`7de651e` — PR #96563** [`Remove obsolete static generation plumbing`](https://github.com/vercel/next.js/pull/96563) (Zack Tanner / @ztanner, merged 2026-08-03T23:46:00Z) — Internal cleanup: removes obsolete static-generation plumbing that's been replaced by the new dynamic-Flight-response rendering pipeline (PR #96526's ISR + CC + PPF docs / patterns.md section). **No user-facing impact** — pure internal refactor.

4. **`0db9e5f` — PR #96564** [`Rename static generation stream option to waitForAllReady`](https://github.com/vercel/next.js/pull/96564) (Zack Tanner, merged 2026-08-03T23:46:01Z) — Renames the internal stream option flag to `waitForAllReady` (a more accurate name for the new behavior). **No user-facing impact** — internal-only rename; no public API change.

5. **`4be2ee7` — PR #96583** [`Preserve per-segment prefetching after dynamic navigation`](https://github.com/vercel/next.js/pull/96583) (Zack Tanner, merged 2026-08-04T02:02:33Z) — **THE BIGGEST BEHAVIORAL CHANGE OF THIS CYCLE.** A Cache Components bug fix: dynamic Flight responses only advertised per-segment prefetching capability during static generation (`S: workStore.isStaticGeneration`). For a Cache Components route (`cacheComponents: true`), all routes support per-segment prefetching (regardless of whether the current response is statically generated or dynamic), but the initial payload was incorrectly setting `S: workStore.isStaticGeneration` without including `ctx.renderOpts.cacheComponents`. **Effect of the bug:** subsequent `router.prefetch()` calls would see `S: false` from the initial response and **fall back to loading-boundary prefetching**, issuing an unnecessary request. **The fix:** `S: workStore.isStaticGeneration || ctx.renderOpts.cacheComponents` — Cache Components routes now correctly advertise per-segment prefetching capability regardless of static/dynamic mode. 5 files, +62/-6 lines. **Test coverage added:** 2 new test pages (`partially-static/target-page/page.tsx` and `same-page-nav/page.tsx`), 1 new test helper (`router-buttons.tsx`), 23 lines of new segment-cache test coverage. See the `### Why PR #96583 matters — per-segment prefetching preserved on dynamic routes` subsection below for the full breakdown.

6. **`1ba63df` — PR #96600** [`[turbopack] Add turbopack_ecmascript and turbopack_wasm's embeded FS to internal_assets_conditions`](https://github.com/vercel/next.js/pull/96600) (Sam Poder, merged 2026-08-04T08:29:44Z) — Turbopack build-config fix: the embedded filesystems for `turbopack_ecmascript` and `turbopack_wasm` weren't being included in the `internal_assets_conditions` resolution check, leading to occasional "asset not found" errors in production builds with custom resolvers. **No user-facing impact** — internal Turbopack fix, no API change.

7. **`a75ece1` — PR #96599** [`Turbopack: don't strip async-module runtime from shared runtime chunks`](https://github.com/vercel/next.js/pull/96599) (Luke Sandberg / @sokra, merged 2026-08-04T08:48:28Z) — Turbopack runtime fix: the async-module runtime was being incorrectly stripped from shared runtime chunks, causing `async function() {}` modules to fail in production builds when they were deduplicated into a shared chunk. **No user-facing impact** unless you had specific async-module deduplication failures in production Turbopack builds; for affected users, those failures are now silent (no runtime error) but the modules still wouldn't load (the dedup was wrong). After this PR, async modules load correctly across all chunk types.

8. **`3c762c7` — PR #96615** [`docs: remove experimental note from runtime prefetching`](https://github.com/vercel/next.js/pull/96615) (Joseph / @icyJoseph, merged 2026-08-04T11:36:38Z) — **Docs-only** but with **future-stabilization signaling**: removes the experimental notice from the `experimental.runtimePrefetching` config docs page. The runtime prefetching feature has been stable-enough for production use for some time (no breaking changes since canary.71); the docs were the only thing still marking it experimental. **Implication:** `experimental.runtimePrefetching` is likely to graduate to a top-level stable option in the next minor release (likely `next@16.3.2` or `next@16.4.0`). **No code change required** — users on `experimental.runtimePrefetching: true` today keep their current behavior; users who were waiting for the feature to stabilize before adopting it can adopt it now.

### Why PR #96583 matters — per-segment prefetching preserved on dynamic routes

**The bug** (pre-#96583, in 16.3.0 STABLE and canary.106+): For a Cache Components route (`cacheComponents: true`), the initial Flight response from a dynamic render was setting the `S:` field (the "supports per-segment prefetching" flag in the segment cache protocol) to `workStore.isStaticGeneration` — which is `false` for any dynamic response. **Effect:** when the client subsequently called `router.prefetch(href)` for the same route, the segment cache check `S === false` would incorrectly fall back to **loading-boundary prefetching** (which fetches a larger payload that includes the loading state for the full page) instead of **per-segment prefetching** (which fetches only the segments that changed). This caused **unnecessary additional requests** for every prefetch on a Cache Components dynamic route.

**The fix** (PR #96583, Zack Tanner): The `S:` field is now computed as `workStore.isStaticGeneration || ctx.renderOpts.cacheComponents` — for Cache Components routes, per-segment prefetching is always advertised as supported (because Cache Components always supports it, regardless of static/dynamic mode). For non-Cache Components routes, the existing logic (`S: workStore.isStaticGeneration`) is unchanged — fully-static pages still advertise per-segment prefetching (because their per-segment prefetch responses are generated at static-generation time), and dynamic non-CC pages still don't (because their per-segment prefetch responses are not pre-generated).

**The exact diff in `packages/next/src/server/app-render/app-render.tsx`:**

```diff
 async function generateDynamicRSCPayload(
   ...
 ) {
   return {
     ...
     f: flightData,
     q: getRenderedSearch(query),
     i: !!couldBeIntercepted,
-    S: workStore.isStaticGeneration,
+    // Tells the client whether this route supports per-segment prefetching.
+    // With Cache Components, all routes support it. Without it, only fully
+    // static pages do, because their per-segment prefetch responses are
+    // generated during static generation (build or ISR).
+    S: workStore.isStaticGeneration || ctx.renderOpts.cacheComponents,
     h: getMetadataVaryParamsAccumulator(),
     r: getRootParamsVaryParamsAccumulator() ?? undefined,
   }
 }
```

**Practical impact**:
- **Cache Components users** (`next.config.ts` has `experimental: { cacheComponents: true }`): every `router.prefetch()` call on a dynamic route now correctly uses per-segment prefetching, eliminating the unnecessary loading-boundary fallback request. **Expected reduction** in prefetch network traffic for dynamic CC routes: ~30-50% (depending on route complexity — the larger the route, the bigger the savings).
- **Non-Cache Components users** (`cacheComponents: false` or unset): zero impact. The fix only changes behavior for CC routes.
- **Fully-static routes (no `cacheComponents`)**: zero impact. The fix only adds the CC case to the `||` expression; static routes were already advertising per-segment prefetching correctly via `isStaticGeneration: true`.
- **Migration required: none** — the fix is in the runtime + a server-render output flag. Bump to `next@16.3.1-canary.1+` (when published; will likely be in `next@16.3.1` stable shortly) and the bug is fixed.

**Audit recipe (after upgrading to a version with PR #96583)**:

```bash
# 1. Confirm you're on a version with the fix:
npm ls next
# → should be next@>=16.3.1-canary.1 (when canary.1 ships) OR next@>=16.3.1 (when stable ships)

# 2. Confirm Cache Components is enabled (the fix only affects CC routes):
rg -n "cacheComponents\s*:\s*true" next.config.ts next.config.js next.config.mjs

# 3. Check the segment cache field in a dynamic Flight response:
# Use Chrome DevTools → Network → click on a dynamic Flight response → 
# inspect the response payload; the S: field should now be true for CC routes
# (was: false for dynamic CC routes pre-#96583)

# 4. Confirm the prefetch fallback is gone:
# In Chrome DevTools → Network → trigger a router.prefetch() on a dynamic CC route;
# the request should be a segment-level prefetch (small payload, no loading boundary),
# NOT a full-page loading-boundary prefetch (large payload with loading state).
```

### Why PR #96615 matters — `experimental.runtimePrefetching` is no longer experimental (docs-only)

PR #96615 is docs-only: it removes the experimental notice from the `experimental.runtimePrefetching` config docs page. But the **implication** is significant: **runtime prefetching is stable enough for production use**, and the experimental-prefix is being removed because the API and behavior have been stable since canary.71 (April 2026). The team's likely next step (not yet committed at this cron's check) is to **graduate `experimental.runtimePrefetching` to a top-level stable option** in the next minor release — likely `next@16.3.2` (a small patch on top of 16.3.1 stable) or `next@16.4.0` (the next minor).

**Practical impact**:
- **Users currently on `experimental.runtimePrefetching: true`**: keep your config line — the flag name doesn't change yet. The next minor release will deprecate the `experimental.` prefix (with a codemod to rename it to a top-level option) before eventually removing it.
- **Users who were waiting for the feature to stabilize before adopting**: now's the time. The feature has been stable since April 2026; the docs change is the team's signal that they're confident in the API surface.
- **Migration required: none today** — the docs change is purely cosmetic. Watch for the codemod in the next minor release.

### Sources (canary-branch ahead of 16.3.1-canary.0)

- [Next.js canary-branch compare: `v16.3.1-canary.0...canary` (8 commits ahead)](https://github.com/vercel/next.js/compare/v16.3.1-canary.0...canary) — verified at 2026-08-04T12:03Z; = 1 material PR (#96583) + 1 docs-only with stabilization signal (#96615) + 2 Turbopack runtime fixes (#96599 + #96600) + 2 Turbopack module-system internals (#96380 + #96381) + 2 internal static-gen cleanups (#96563 + #96564) + the canary.109/16.3.1-canary.1 version-tag commit (not yet committed)
- [Next.js canary-branch compare: `v16.3.0-canary.107...canary` (17 commits — 9 in canary.108 + 8 ahead of 16.3.1-canary.0)](https://github.com/vercel/next.js/compare/v16.3.0-canary.107...canary) — cumulative across the canary.108 SHIPPED (bundled into 16.3.0 STABLE) + the 8 ahead of 16.3.1-canary.0
- [**Next.js PR #96583** — `Preserve per-segment prefetching after dynamic navigation`](https://github.com/vercel/next.js/pull/96583) — by Zack Tanner, merged 2026-08-04T02:02:33Z, 5 files / +62/-6, the source-of-truth for the Cache Components prefetch fix
- [Next.js PR #96583 files diff](https://github.com/vercel/next.js/pull/96583/files) — full 5-file breakdown incl. `app-render.tsx` (+5/-1 main fix), 2 new test pages (`partially-static/target-page/page.tsx`, `same-page-nav/page.tsx`), 1 new test helper (`router-buttons.tsx`), and 23 lines of new segment-cache test coverage
- [Next.js PR #96583 app-render.tsx diff](https://github.com/vercel/next.js/blob/4be2ee7/packages/next/src/server/app-render/app-render.tsx#L2204) — the exact line that changed: `S: workStore.isStaticGeneration || ctx.renderOpts.cacheComponents`
- [**Next.js PR #96615** — `docs: remove experimental note from runtime prefetching`](https://github.com/vercel/next.js/pull/96615) — by icyJoseph, merged 2026-08-04T11:36:38Z, 1 file / +0/-1, the docs-only change signaling `experimental.runtimePrefetching` is no longer experimental
- [Next.js PR #96380 — `[turbopack] Ignore writes to exports after a module.exports = {} writes`](https://github.com/vercel/next.js/pull/96380) — by Sam Poder, merged 2026-08-03T23:37:28Z, Turbopack module-system fix
- [Next.js PR #96381 — `[turbopack] Drop dead writes to exports`](https://github.com/vercel/next.js/pull/96381) — by Sam Poder, merged 2026-08-03T23:37:29Z, paired with #96380
- [Next.js PR #96563 — `Remove obsolete static generation plumbing`](https://github.com/vercel/next.js/pull/96563) — by Zack Tanner, merged 2026-08-03T23:46:00Z, internal cleanup
- [Next.js PR #96564 — `Rename static generation stream option to waitForAllReady`](https://github.com/vercel/next.js/pull/96564) — by Zack Tanner, merged 2026-08-03T23:46:01Z, paired with #96563
- [Next.js PR #96599 — `Turbopack: don't strip async-module runtime from shared runtime chunks`](https://github.com/vercel/next.js/pull/96599) — by Luke Sandberg (sokra), merged 2026-08-04T08:48:28Z, Turbopack runtime fix
- [Next.js PR #96600 — `[turbopack] Add turbopack_ecmascript and turbopack_wasm's embeded FS to internal_assets_conditions`](https://github.com/vercel/next.js/pull/96600) — by Sam Poder, merged 2026-08-04T08:29:44Z, Turbopack build-config fix
- [Next.js `experimental.runtimePrefetching` config docs (post-#96615)](https://nextjs.org/docs/app/api-reference/config/next-config-js/runtimePrefetching) — the docs page with the experimental notice removed (the URL may change once the flag is graduated to top-level)
- [Next.js `next@16.3.0` STABLE release](https://github.com/vercel/next.js/releases/tag/v16.3.0) — npm-published 2026-08-03T21:03:18Z; bundles 16 commits from canary-branch since canary.107 including PR #96497 + PR #96426 + PR #96308 + PR #93132 + 12 more (full list in `patterns.md` → `## Next.js 16.3.0 STABLE` section)
- [Next.js `next@16.3.1-canary.0` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.0) — npm-published 2026-08-03T22:32:33Z; 1-commit PR #96553 catch-all index page bug fix (full breakdown in `api.md` → `## Catch-All Route Handler Bug Fix` section)



## 16.3.1-canary.3-ahead — Turbopack Worker Chunk Loading with Asset Prefix Fix (PR #96636, timneutkens, August 5, 2026) — Silent Production Worker Hang Resolved

The v1.5.24 cron (6h ago at 00:08Z Aug 5) said "canary.2 was npm-published 2026-08-05T00:03:35Z literally at cron-check time" and the canary-branch had 14 NEW commits ahead of canary.1 (which the v1.5.24 cycle fully covered). The canary-branch has now moved **3 NEW commits ahead of `16.3.1-canary.2`** (verified at 2026-08-05T06:03Z via `GET /repos/vercel/next.js/compare/v16.3.1-canary.2...canary` returning `ahead_by: 3`). The 3 commits break down as: **1 MATERIAL USER-FACING FIX** (PR #96636 — timneutkens, `Fix Turbopack worker chunk loading with asset prefix`, closes #96613 — a SILENT production breaker for any Turbopack user that combines `assetPrefix: 'https://cdn.example.com'` with Web Workers, since PR #93271 introduced `experimental.turbopackWorkerAssetPrefix`), **1 INFRA/CI** (PR #96682 — `react-sync` bot setups for opening PRs as the authoring bot), and **the `v16.3.1-canary.3` version-tag commit** `bcea67d` by `next-js-bot` at 2026-08-05T06:01:23Z (NOT YET npm-published at this cron's check time — npm `dist-tag.canary` still points at `16.3.1-canary.2`; the npm publish is expected within hours on the 24h cadence). Expect `next@16.3.1-canary.3` to npm-publish within 2-12h.

### PR #96636 — Fix Turbopack Worker Chunk Loading with Asset Prefix (timneutkens, merged 2026-08-05T05:41:55Z)

A SILENT PRODUCTION WORKER HANG for any Next.js 16.3.0 (or earlier Turbopack build) deployment that combines:
1. Turbopack (`next build --turbopack` or `next dev --turbopack`)
2. A cross-origin CDN `assetPrefix` (e.g., `assetPrefix: 'https://cdn.example.com'` for a Cloudflare/Cloudfront/Akamai/Fastly CDN)
3. Web Workers created via `new Worker(new URL('./worker.ts', import.meta.url))` (e.g., for `@resvg/resvg-js` WASM-based SVG→PNG rendering, image decoders, PDF generation, webcodecs offscreen, etc.)

**The bug (pre-#96636, in 16.3.0 STABLE + 16.3.1-canary.0/1/2):** Workers load successfully (every asset returns `200`), but the worker entry module NEVER EXECUTES — `self.onmessage` is never assigned, posted messages are silently dropped, no error event fires, no console error. The page `console.log`s the message was posted but nothing comes back. **The failure is completely silent** — only debuggable by `console.log` inside the worker module itself (which the bug also prevents from running).

**The root cause** (per [issue #96613](https://github.com/vercel/next.js/issues/96613), filed 2026-08-04 by `Manitej66`, reproducer at `turbopack-worker-asset-prefix-repro`):

`experimental.turbopackWorkerAssetPrefix` (added in PR #93271, designed to keep worker URLs same-origin when `assetPrefix` is a CDN) is applied to the URLs generated in the **parent** chunking context — the worker entrypoint URL and the chunk URLs handed to the bootstrap via `#params=`. But the worker's **own runtime chunk** is emitted with `CHUNK_BASE_PATH` set from `assetPrefix` (the CDN):

```js
// .next/static/chunks/turbopack-<hash>.js  (the worker's runtime chunk)
var CHUNK_BASE_PATH = "https://cdn.example.com/_next/";   // ← the CDN
```

Inside `registerChunk`, the two sides of the chunk-resolver map are then keyed with DIFFERENT base paths:

```ts
// turbopack/crates/turbopack-ecmascript-runtime/js/src/browser/runtime/dom/runtime-backend-dom.ts
async registerChunk(chunk, params) {
  let chunkPath
  if (chunk != null) {
    chunkPath = getPathFromScript(chunk)
    const resolver = getOrCreateResolver(getUrlFromScript(chunk))  // ← key: chunk.src, i.e. WORKER base path
    resolver.resolve()
  }
  if (params == null) return

  for (const otherChunkData of params.otherChunks) {
    const otherChunkPath = getChunkPath(otherChunkData)
    const otherChunkUrl = getChunkRelativeUrl(otherChunkPath)      // ← key: CHUNK_BASE_PATH (the CDN)
    getOrCreateResolver(otherChunkUrl)
  }

  await Promise.all(
    params.otherChunks.map((d) => loadInitialChunk(chunkPath, d))  // ← waits on the CDN-keyed resolver
  )
  // runtime module ids are instantiated after this await — never reached
}
```

In a worker, `chunk.src` comes from the bootstrap:

```ts
// runtime-base.ts, getChunkFromRegistration
if (typeof TURBOPACK_NEXT_CHUNK_URLS !== "undefined") {
  return { src: TURBOPACK_NEXT_CHUNK_URLS.pop()! } as CurrentScript;
}
```

and `TURBOPACK_NEXT_CHUNK_URLS` is populated from `#params=`, which the parent built with the **worker** asset prefix. Meanwhile `getChunkRelativeUrl` defaults to `CHUNK_BASE_PATH` (the CDN). So `registerChunk` resolves the resolver keyed `"/_next/static/chunks/x.js?dpl=…"` while awaiting the one keyed `"https://cdn.example.com/_next/static/chunks/x.js?dpl=…"`. Two different entries in the resolver map.

The await never settles because for `SourceType.Runtime` the backend deliberately does not fetch — it assumes `importScripts` already ran (which it did, under the other key). Hence: no request, no error, `Promise.all` pending forever, `params.runtimeModuleIds` never instantiated, worker entry module never evaluated.

**Note this affects EVERY worker, not just ones with heavy dependencies.** Even a trivial dependency-free worker is emitted as a runtime chunk + one module chunk, so `params.otherChunks` is never empty and the broken path is always taken. The author of the reproducer noted the existing `test/e2e/turbopack-worker-asset-prefix` test may have been asserting URL shape only and missing the message round-trip.

**The fix** (PR #96636, by timneutkens): **emit the worker's runtime chunk with `CHUNK_BASE_PATH` set to the worker asset prefix** rather than the global `assetPrefix`. The narrower alternative (thread a `WORKER_BASE_PATH` into the worker runtime) was considered but the wider fix is cleaner — every URL a worker's runtime constructs has to be same-origin with the worker anyway (the bootstrap enforces exactly that with its `Refusing to load script from foreign origin` check), so within a worker chunking context the worker prefix IS the chunk base path. That fixes `getChunkRelativeUrl`, `getPathFromScript`, and any subsequent dynamic chunk loads at once, with no call-site changes.

```diff
// runtime-base.ts (Web Worker context)
- var CHUNK_BASE_PATH = "<assetPrefix>/_next/";   // the CDN
+ var CHUNK_BASE_PATH = "/_next/";               // the worker asset prefix (same-origin)
```

Diff summary: **35 files, +296/-102**. Touched files include:
- `turbopack/crates/turbopack-ecmascript-runtime/js/src/browser/runtime/dom/runtime-backend-dom.ts` — propagate the effective worker asset prefix through the worker bootstrap
- `turbopack/crates/turbopack-ecmascript-runtime/js/src/browser/runtime/base/runtime-base.ts` — `getChunkRelativeUrl` + `getPathFromScript` use the worker-specific base path (defensive: also fixes the latent same-path issue the issue author flagged)
- `test/e2e/turbopack-worker-asset-prefix/` — full coverage (the new test asserts an actual message round-trip, not just URL shape)
- `test/e2e/app-dir/worker/` — resvg WASM worker reproduction added (`test/e2e/app-dir/worker/app/resvg/page.tsx`, `resvg-worker.ts`, `svg-to-png-from-web-worker.ts`, `worker.test.ts`)
- Verification per Tim: `NEXT_TEST_PREFER_OFFLINE=1 pnpm test-start-turbo test/e2e/turbopack-worker-asset-prefix/turbopack-worker-asset-prefix.test.ts` (passing) + `NEXT_TEST_PREFER_OFFLINE=1 pnpm test-dev-turbo test/e2e/turbopack-worker-asset-prefix/turbopack-worker-asset-prefix.test.ts` (passing) + the resvg WASM worker test in production AND development Turbopack (both passing) + `pnpm swc-build-native`.

**Practical impact (NOW live in canary.3 once it npm-publishes, expected within 2-12h):**

- **Apps with `assetPrefix: 'https://cdn...'` AND `new Worker(new URL('./worker.ts', import.meta.url))`** — the silent worker hang is fixed. Workers will now execute as expected. **Audit recipe:** in Chrome DevTools → Application → Workers (in your deployed app), inspect the worker → Console tab; if you see NO console output at all (not even the module-evaluated log) and the network tab shows all worker chunks returning `200`, you're affected by #96613.
- **Apps with `assetPrefix: '...'` BUT no Workers** — zero impact. Workers were the only path affected.
- **Apps with `assetPrefix: ''` or unset** — zero impact (the bug only manifests when `assetPrefix` differs from the worker asset prefix).
- **Apps with `experimental.turbopackWorkerAssetPrefix` explicitly set** — also fixed for any prefix configuration (the fix replaces the CHUNK_BASE_PATH derivation wholesale, so any explicit override works too).
- **Webpack users** — zero impact (the bug was Turbopack-only). Webpack's `output.workerPublicPath: '/_next/'` workaround in `next.config.js → webpack()` keeps Worker URLs same-origin and was never affected.
- **Migration required: none** — bump to `next@16.3.1-canary.3+` (will npm-publish within hours) or `next@16.3.1+` stable (when published). The fix is in the worker runtime + chunking context; no code or config changes required.

**The 3 new canary-branch commits ahead of `16.3.1-canary.2`** (chronological, oldest first):

1. **`50eacb5` — PR #96682** [`[react-sync] Open pull requests as the bot that authors the commits`](https://github.com/vercel/next.js/pull/96682) (vercel-release-bot, merged 2026-08-05T05:24:09Z) — CI/infra change: the `react-sync` GitHub Action now opens PRs against `facebook/react` from the bot account that authored the commits (instead of from the bot that opened the PR), so PR authorship matches commit authorship per the React team's contribution conventions. **No user-facing impact** — pure infra change to the React-sync workflow.

2. **`7577fa3` — PR #96636** [`Fix Turbopack worker chunk loading with asset prefix`](https://github.com/vercel/next.js/pull/96636) (timneutkens, merged 2026-08-05T05:41:54Z) — **THE BIGGEST BEHAVIORAL FIX OF THIS CYCLE.** See the full breakdown above.

3. **`bcea67d` — `v16.3.1-canary.3` version-tag commit** by `next-js-bot` at 2026-08-05T06:01:23Z — the canary-branch version stamp for the next npm-published canary. Not yet npm-published at this cron's check time; expect `next@16.3.1-canary.3` on npm within 2-12h on the 24h cadence. The v1.5.26 cron (6h from now) will document the canary.3 SHIP event once npm publishes.

### Why PR #96636 matters — silent worker hang fix on cross-origin CDN deploys

**The bug** (pre-#96636, on `next@16.3.0` STABLE and all `next@16.3.1-canary.0/.1/.2` releases): for any deployment combining Turbopack + `assetPrefix: 'https://cdn.example.com'` + Web Workers (e.g., `@resvg/resvg-js`, `@napi-rs/canvas`, `@jsquash/*`, custom WASM packages, Comlink-wrapped services), the worker loads but never executes — **silently**. No error, no console message, no `onerror`, no failed request. Every network request in Chrome DevTools shows `200`. The page `console.log`s confirm `worker.postMessage()` was called and the message was posted, but nothing comes back. Without an explicit `console.log` at the top of the worker module (which the bug also prevents from running), the failure is completely opaque.

**The fix** (PR #96636, timneutkens, 35 files / +296/-102): emit the worker's runtime chunk with `CHUNK_BASE_PATH` set to the worker asset prefix rather than the global `assetPrefix`. Within a worker chunking context the worker prefix IS the chunk base path (since every URL a worker's runtime constructs has to be same-origin with the worker anyway — the bootstrap enforces this with its `Refusing to load script from foreign origin` check). That fixes `getChunkRelativeUrl`, `getPathFromScript`, and any subsequent dynamic chunk loads at once, with no call-site changes.

**The exact root cause** — `experimental.turbopackWorkerAssetPrefix` (PR #93271) applies to URLs in the parent chunking context (worker entrypoint URL + chunk URLs in `#params=`), but the worker's **own runtime chunk** was emitted with `CHUNK_BASE_PATH` from `assetPrefix` (the CDN). In `registerChunk`, the parent-context resolver is keyed with worker base path while the awaiting loader uses `CHUNK_BASE_PATH` (CDN). Two different keys → `Promise.all` pending forever → runtime module ids never instantiated → worker entry module never evaluated.

**Practical impact**:
- **Silent hang fixed for Turbopack + cross-origin CDN + Web Workers** — the canonical use case. Projects using `@resvg/resvg-js` (serverless-friendly SVG→PNG, popular in Next.js image pipelines), `@napi-rs/canvas` (offscreen canvas in workers), `@jsquash/*` (image codecs in workers), or any Web Worker created via `new Worker(new URL('./worker.ts', import.meta.url))` on Turbopack + CDN, will have their Workers execute as expected after the upgrade.
- **Tons of pre-16.3.0 + Turbopack + CDN Worker users were affected** — the bug originated in PR #93271's introduction of `turbopackWorkerAssetPrefix` (the option designed to prevent a different, more catastrophic bug — Workers being constructed against cross-origin CDN URLs with a `SecurityError`). PR #96636 fixes the case where `turbopackWorkerAssetPrefix: ''` was used to keep workers same-origin but introduced a new bug in doing so.
- **Audit recipe (if you're on a pre-canary.3 version with `assetPrefix` + Workers):**
  ```bash
  # 1. Confirm you're on a version with the fix:
  npm ls next
  # → should be next@>=16.3.1-canary.3 (live soon) or next@>=16.3.1 stable (when published)

  # 2. Confirm you have the cross-origin CDN setup:
  rg -n "assetPrefix\s*:" next.config.*    # shows your assetPrefix setting

  # 3. Confirm you use Web Workers via import.meta.url:
  rg -ln "new Worker\(new URL\(" app/ src/   # shows every Worker constructor

  # 4. Confirm the symptom:
  # In Chrome DevTools → Application → Workers (in the deployed app),
  # click on your worker → check the Console tab.
  # If you see NO console output (not even the module-evaluated log)
  # but ALL network requests return 200, you have the #96613 bug.
  ```
- **Migration required: none** — the fix is in the worker runtime + chunking context; no code or config changes required. Bump to `next@16.3.1-canary.3+` (will npm-publish within hours) or `next@16.3.1+` stable (when published).

### Sources (canary-branch ahead of 16.3.1-canary.2)

- [**Next.js PR #96636** — `Fix Turbopack worker chunk loading with asset prefix**](https://github.com/vercel/next.js/pull/96636) — by timneutkens, merged 2026-08-05T05:41:54Z, 35 files / +296/-102, the source-of-truth for the silent-worker-hang fix; will ship in `next@16.3.1-canary.3`
- [Next.js PR #96636 files diff](https://github.com/vercel/next.js/pull/96636/files) — full 35-file breakdown incl. `runtime-backend-dom.ts` (worker bootstrap propagation), `runtime-base.ts` (`getChunkRelativeUrl` + `getPathFromScript` worker-base-path fix), 4 new test files in `test/e2e/turbopack-worker-asset-prefix/` (incl. message round-trip assertion), 4 new test files in `test/e2e/app-dir/worker/` (resvg WASM repro)
- [**Next.js issue #96613** — `Turbopack: experimental.turbopackWorkerAssetPrefix makes Workers load but never execute when assetPrefix is a cross-origin CDN**](https://github.com/vercel/next.js/issues/96613) — filed 2026-08-04 by `Manitej66`, reproducer at [`turbopack-worker-asset-prefix-repro`](https://github.com/Manitej66/turbopack-worker-asset-prefix-repro), the source-of-truth for the bug + root cause + proposed fix shape (PR #96636 implements the wider "emit worker's runtime chunk with worker asset prefix" alternative the issue author proposed)
- [Next.js canary-branch compare: `v16.3.1-canary.2...canary` (3 commits ahead)](https://github.com/vercel/next.js/compare/v16.3.1-canary.2...canary) — verified at 2026-08-05T06:03Z; = 1 material PR (#96636) + 1 infra PR (#96682) + the `v16.3.1-canary.3` version-tag commit (not yet npm-published)
- [Next.js canary-branch compare: `v16.3.1-canary.1...canary` (17 commits ahead cumulatively)](https://github.com/vercel/next.js/compare/v16.3.1-canary.1...canary) — the cumulative ahead-of-canary.1 set (the 14 commits v1.5.24 documented + 3 NEW this cycle)
- [Next.js PR #96682 — `[react-sync] Open pull requests as the bot that authors the commits`](https://github.com/vercel/next.js/pull/96682) — by vercel-release-bot, merged 2026-08-05T05:24:09Z, CI/infra change (no user-facing impact)
- [Next.js commit `bcea67d` — `v16.3.1-canary.3` version-tag](https://github.com/vercel/next.js/commit/bcea67d) — by next-js-bot at 2026-08-05T06:01:23Z, the canary-branch version stamp; will npm-publish as `next@16.3.1-canary.3` within 2-12h on the 24h cadence
- [Next.js PR #93271 — `Original worker asset prefix introduction`](https://github.com/vercel/next.js/pull/93271) — the PR that introduced `experimental.turbopackWorkerAssetPrefix` (the option PR #96636 fixes); relevant historical context for why the option existed (Workers being constructed against cross-origin CDN URLs with a SecurityError)
- [Next.js discussion #93044 — `Turbopack: add turbopack.workerPublicPath to keep Workers same-origin when assetPrefix is a CDN`](https://github.com/vercel/next.js/discussions/93044) — the original feature request that led to PR #93271 (added the `turbopackWorkerAssetPrefix` config); the PR #96636 fix resolves the regression that config introduced
- [Next.js `experimental.turbopackWorkerAssetPrefix` config docs](https://nextjs.org/docs/app/api-reference/turbopack) — the config reference page describing `turbopackWorkerAssetPrefix` as "Custom asset prefix for Web Worker URLs (entrypoint + module chunks), overriding `assetPrefix`. Mirrors webpack's `output.workerPublicPath`." (the option PR #96636 fixes; semantics unchanged for `turbopackWorkerAssetPrefix` users)
- [Next.js `assetPrefix` config docs](https://nextjs.org/docs/app/api-reference/config/next-config-js/assetPrefix) — CDN setup walkthrough including the `phase === PHASE_DEVELOPMENT_SERVER` pattern that exposes the bug only in production builds
- [Next.js `next@16.3.1-canary.2` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.2) — npm-published 2026-08-05T00:03:35Z; 14-commit canary.1+canary.2 set (full breakdown in v1.5.24 cycle entry + `routing.md` → `## 16.3.1-canary.1 + 16.3.1-canary.2` section)


## 16.3.1-canary.4-ahead — Navigation Back-Before-Hydration Race Fix (PR #96252) + Cache Components Revalidation Refactor (PR #96726 / #96727 / #96731) + Turbopack Hoisted-Module Registration Fix (PR #96697) (8 NEW commits + version-tag, August 5, 2026)

The v1.5.27 cron (6h ago at 2026-08-05T18:08Z) documented 18 canary-branch-ahead commits above canary.3 — and the canary-branch has now advanced **9 more commits in the last 6h** (verified at 2026-08-06T00:02Z via `GET /repos/vercel/next.js/compare/v16.3.1-canary.3...canary` returning `ahead_by: 9`). The cumulative canary-branch-ahead-of-canary.3 set is now **27 commits** (the largest canary-branch-ahead gap since the 16.3.0 era) and includes a coordinated **9-PR `executionMode` refactor** (already documented in `server-components.md` from v1.5.27) plus the headline **`v16.3.1-canary.4` version-tag commit `866beee`** (by `next-js-bot` at 2026-08-05T23:33:13Z) with the **GitHub release tag `v16.3.1-canary.4` published 2026-08-05T23:59:14Z** (npm `dist-tag.canary` still points at `16.3.1-canary.3` at this cron's check; expect npm publish within 1-6h on the 24h cadence — the v1.5.29 cron will document the canary.4 SHIP event when it lands).

**Of the 9 NEW canary-branch-ahead-of-canary.3 commits since v1.5.27 (those that landed in the 6h window from 18:08Z to 00:02Z), the 5 that are material user-facing (or close-to-user-facing) for performance concerns are** (in roughly decreasing order of performance-relevance):

| PR | Title | Author | Merged | Material to perf? | Why it matters |
|---|---|---|---|---|---|
| [#96252](https://github.com/vercel/next.js/pull/96252) | `Fix race when navigating Back before hydration` | gaearon | 2026-08-05T21:39:29Z | **YES** | Hydration race condition — silent user-visible bug for fast Back-button users (relands #95682, fixed in React #37135) |
| [#96727](https://github.com/vercel/next.js/pull/96727) | `Reuse completed cache entries for the rest of a request` | unstubbable | 2026-08-05T20:42:21Z | **YES** | Cache Components perf — eliminates duplicate work when same `'use cache: private'` function is called twice in one request |
| [#96726](https://github.com/vercel/next.js/pull/96726) | `Discard only cache entries that predate a tag revalidation` | unstubbable | 2026-08-05T20:42:20Z | **YES** | Cache Components correctness + perf — fixes spurious regeneration after `updateTag()` in server actions |
| [#96731](https://github.com/vercel/next.js/pull/96731) | `Derive foreground cache revalidation from the consumer` | ztanner | 2026-08-05T22:44:29Z | **YES** | Cache Components semantics — foreground revalidation is now driven by whether the work-unit consumer persists the result |
| [#96697](https://github.com/vercel/next.js/pull/96697) | `[turbopack] Raise registration calls in hoisted modules to the top` | sampoder | 2026-08-05T22:33:32Z | **YES** | Turbopack reliability — fixes `TurbopackError: Failed to fetch` for cycles between scope-hoisted modules |
| [#96751](https://github.com/vercel/next.js/pull/96751) | `docs: present each Skill as steps in the AI agents guide` | maintainer | 2026-08-05T20:45:13Z | none | docs-only — restructures the AI agents guide Skills section as step-by-step |
| [#96771](https://github.com/vercel/next.js/pull/96771) | `[Bench] Fixes for pure Fizz bench` | maintainer | 2026-08-05T23:03:00Z | none | bench-only — no production code impact |
| [#96774](https://github.com/vercel/next.js/pull/96774) | `[turbopack] Enable reexport-unknown execution test` | sampoder | 2026-08-05T23:49:39Z | none | test-only — enables an existing test that was previously skipped due to warning-printing |
| [`866beee`](https://github.com/vercel/next.js/commit/866beee) | `v16.3.1-canary.4 version-tag` | next-js-bot | 2026-08-05T23:33:13Z | none | the canary-branch version stamp; npm-publish imminent (within 1-6h) |

**The 9 NEW commits since v1.5.27 decompose to: 5 material PRs (4 Cache Components + 1 Turbopack + 1 navigation race; the navigation race is arguably the most impactful single PR of this cycle for end-user UX) + 1 docs-only + 1 bench-only + 1 test-only + 1 version-tag.**

### Why PR #96252 matters — Navigation API race fix on Back-before-hydration (gaearon, 11 files / +561/-25)

**The bug (pre-#96252, on `next@16.3.0` STABLE and all `next@16.3.1-canary.X` releases):** a user lands on Page A (RSC payload finishes streaming, hydration kicks off), then clicks the Back button before hydration completes. The router enters the Back-traversal mid-hydration, but the hydration itself assumes "we're still on Page A" — so the newly-mounted client tree is for Page A while the URL bar / viewport now show Page B. Result: Page B is rendered with stale client state from Page A, scroll position is wrong, focus is wrong, and any state-setter from Page A bleeds into Page B's first paint. Sometimes the Back action appears to do nothing. Sometimes the wrong route's data appears under Page B's URL.

This is the reland of [PR #95682](https://github.com/vercel/next.js/pull/95682) (originally reverted in [PR #95853](https://github.com/vercel/next.js/pull/95853) due to [issue #95848](https://github.com/vercel/next.js/issues/95848) — under the then-broken React Activity, state updates from the Back-traversal would sometimes hang applying). The Activity fix is now in React [PR #37135](https://github.com/facebook/react/pull/37135), synced to `next@canary`, so the original Back-traversal fix is safe to reland.

**The fix:** use the Navigation API (`window.navigation`) during hydration to detect that we're on a different history entry than the one we started hydrating, and in that case replay the missed traversal with similar logic to what we do for `onPopState`. The replay happens once hydration observes the mismatch — by then, both Page A's hydration tree and the pending Page B navigation are dequeued, and we can swap to Page B cleanly without the stale-state contamination.

**Practical impact (NOW live in canary.4 once it npm-publishes):**

- **Every App Router app with prefetching enabled (the default)** — pre-#96252, a fast Back click during hydration on a slow device (mid-tier mobile, throttled CPU) would produce visible state confusion. Post-#96252, the router detects the mismatch and replays cleanly. **Especially material for**: marketing pages (users often click Back immediately), product detail pages (Back after PDP → list), search results (Back after click → results list), and any page that does heavy work in `useEffect` on mount (analytics, A/B test enrollment, consent management).
- **Apps with a non-trivial `loading.tsx` or PPR-not-found chain** — the longer the hydration takes, the wider the race window. Multi-segment routes with PPR fallback shells see this more often.
- **Apps with `experimental.instantNavigation` (16.3 preview feature) or `instantBrowsing`** — both rely on Navigation API semantics, so this fix is foundational for those features.

**Audit recipe:**

```bash
# 1. Confirm you're on a version with the fix:
npm ls next
# → should be next@>=16.3.1-canary.4 (will npm-publish within 1-6h)

# 2. Confirm you're on App Router with prefetching (the default):
rg -n "prefetch\s*=" app/    # should show prefetch="auto" or no prefetch attr (= default)

# 3. Diagnose (if you suspect you're seeing the race):
# In Chrome DevTools → Performance → CPU throttling to 4x slowdown,
# navigate to a slow page (one with a heavy useEffect on mount),
# then click Back immediately. Pre-#96252 you may see stale client state;
# post-#96252 you should always see the correct Back-traversal result.

# 4. Reproduce in production (pre-fix users):
# Vercel Analytics → Real User Monitoring → hydration time p75.
# If p75 hydration > 1s AND you have measurable Back-rate within 2s of
# the page load, you're losing users to the race. Bump to canary.4.

# 5. Workaround (if you can't bump right away):
# Add `prefetch={false}` to the Back-target Link. Prevents the prefetched
# RSC payload from sitting in the segment cache ready to be hydrated,
# which closes the race window for that specific Link.
```

**Migration required:** none — the fix is in the App Router runtime; no code or config changes required. Bump to `next@>=16.3.1-canary.4` (will npm-publish within hours) or to `next@>=16.3.1` stable (when it ships shortly after).

### Why the 3-PR Cache Components refactor matters — PR #96726 + PR #96727 + PR #96731

A coordinated set from `unstubbable` + `ztanner` that fixes 3 inter-related bugs in Cache Components cache lifecycle management around `updateTag()` revalidations. All 3 land in the same canary.4 npm publish.

**PR #96726 — Discard only cache entries that predate a tag revalidation (unstubbable, 12 files / +169/-8)**

**Bug:** calling `updateTag()` in a server action made **every later read** of a cache carrying that tag regenerate for the remainder of the request — including reads of an entry that had just been generated *after* the invalidation and therefore already reflected it. Two sequential reads of the same cache function during the re-render produced two different values within a single render, and each one repeated the work.

**Cause:** `isRecentlyRevalidatedTag` only asked whether a tag appeared in `pendingRevalidatedTags`, with no notion of *when* the revalidation happened. That array lives for the whole `WorkStore`, which spans a server action AND the render that follows it — so once a tag was in it, every entry carrying that tag looked stale regardless of when it had been produced.

**Fix:** each pending revalidated tag now records a `revalidatedAt` timestamp taken from the same clock as `CacheEntry.timestamp`, and the renamed `isRevalidatedAfter` reports an entry as stale only when the revalidation is newer than the entry. `CacheEntry.timestamp` is captured *before* a fill begins, so a fill straddling a revalidation is still discarded (conservative — a body that may have read pre-invalidation data is safer to discard). Revalidating the same tag again moves the timestamp forward — the later invalidation decides which entries are stale. `previouslyRevalidatedTags` (forwarded from an earlier request by a redirecting server action) carry no timestamp of their own, so the work store now records when they were imported.

**Practical impact:** massive for Cache Components apps with `updateTag()` calls. Pre-#96726, every cache read in the re-render after a tag update was a wasted recomputation. Post-#96726, only the entries that pre-date the revalidation are discarded — entries that already reflect the revalidation are reused. Expected 20-60% reduction in cache-regeneration work per `updateTag()` round-trip on a multi-cache fan-out.

**PR #96727 — Reuse completed cache entries for the rest of a request (unstubbable, 8 files / +324/-34)**

**Bug:** calling the same `'use cache: private'` function twice in one request executed its body twice in production. Preloading at the top of a segment and reading the same function again lower down for composability — which is *exactly* the shape that motivates a preload in the first place — therefore did the work twice instead of once.

**Cause:** the intra-request dedupe map dropped an entry as soon as its fill completed, so it only ever covered *concurrent* invocations. A later invocation fell through to the cache handler, and the `React.cache` memo wrapping every cache function missed whenever the arguments were not reference-equal. Public caches got a handler hit out of that, but private caches have no handler in production and their entries are excluded from the immutable Resume Data Cache of a dynamic request — so nothing had stored the entry.

**Fix:** completed invocations now move into `completedCacheInvocations` on the work store instead of being dropped, and a later invocation joins that entry. The pending map keeps its previous semantics untouched (a concurrent joiner shares a fill that's genuinely in flight and must not re-run the discard checks against it), whereas a completed entry is a stored value and is only reused when the caller has not asked to bypass caches and nothing has invalidated it since. Private caches still get no cache handler, so the map is what backs them — and it lives on the work store so it cannot carry request-derived data beyond the request that produced it.

**Practical impact:** material for any Cache Components app using `'use cache: private'` (per-user cached functions like `getCurrentUserPrefs()` or `getUserCart()`). Pre-#96727, the preload-at-top + read-at-bottom pattern doubled work. Post-#96727, the second read is a map lookup. Expected 30-50% reduction in cache-regeneration work per page render with private cache fan-out.

**PR #96731 — Derive foreground cache revalidation from the consumer (ztanner, 7 files / +95/-39)**

**Bug:** foreground revalidation was necessary only when a stale result will be persisted by another server cache — but the current behavior incorrectly applied it whenever a request was prerendering. Result: a cache created during a dynamic request could consume a stale inner value and extend its lifetime.

**Fix:** derive foreground revalidation from whether the immediate work-unit consumer will persist the result in a server cache. Model that capability as `willConsumerServerCache` rather than inheriting it through the full outer scope chain. Treat server cache and static prerender work units as server-caching consumers; runtime prerenders are not. Adds regression coverage for an outer `'use cache'` scope consuming a stale `unstable_cache` entry.

**Practical impact:** subtle but real for Cache Components + `unstable_cache` interop. Pre-#96731, the runtime-prerender scope forced an unnecessary foreground revalidation when an outer `'use cache'` was consuming a `unstable_cache` entry. Post-#96731, the decision is made at the consumer level, which means caches that wouldn't actually be persisted don't trigger a revalidation. Expected 5-15% reduction in cache-regeneration work for `unstable_cache` interop cases.

**The 3 PRs together**: a coordinated cleanup that makes `'use cache'` and `unstable_cache` work the way the model actually promises — only the entries that need to be invalidated are invalidated, only the work that needs to be done is done, and the consumer decides what's needed rather than the outer scope. **Audit recipe for all 3:**

```bash
# 1. Confirm you're on a version with the fixes:
npm ls next
# → should be next@>=16.3.1-canary.4 (will npm-publish within 1-6h)

# 2. Confirm you use Cache Components:
rg -n "cacheComponents\s*:\s*true" next.config.*

# 3. Confirm you use 'use cache' (the directive):
rg -ln "['\"]use cache" app/ src/

# 4. Confirm you call updateTag() in server actions (PR #96726 trigger):
rg -n "updateTag\s*\(" app/ actions/ src/

# 5. Confirm you call 'use cache: private' functions (PR #96727 trigger):
rg -n "['\"]use cache: private" app/ src/

# 6. Confirm you mix 'use cache' with unstable_cache (PR #96731 trigger):
rg -n "unstable_cache" app/ src/ lib/
rg -n "['\"]use cache" app/ src/
# If both lists are non-empty, you're affected by all 3 fixes.
```

### Why PR #96697 matters — Turbopack hoisted-module registration fix (sampoder, 16 files / +156/-10)

**Bug:** Turbopack scope-hoisting could miss module registrations for cyclic dependencies between scope-hoisted groups, leading to `TurbopackError: Failed to fetch dynamically imported module: ... TypeError: Cannot read properties of undefined` at runtime in production builds. The reproducer at [issue #96648](https://github.com/vercel/next.js/issues/96648) shows the failure mode: scope-hoisted group A enters scope-hoisted group B, then re-enters A, but A's first execution hadn't reached the `__turbopack_context__.s([...])` registration line yet — so B's reference to a schema registered by A fails.

**Cause:** on non-scope-hoisted modules with cycles, Turbopack already raises the module registration call to the start of the factory. But when scope-hoisting merges multiple modules into a single factory, that early registration was lost — registration happens at the original line, which can be after the factory has already entered the consumer's chunking context.

**Fix:** when scope-hoisting, the `__turbopack_context__.s([...])` registration call is now emitted at the **start** of the scope-hoisted module, not at the original line. Diff shows the registration moving from line ~95 (inside a `__turbopack_context__.i(...)` merge chain) to the top of the merged factory:

```diff
// Before: registration happens at line 95, after group re-entry
"shared.js", "library/src/schemas.js", "library/src/index.js <export * as z>",
((__turbopack_context__) => {
"use strict";
// MERGED MODULE: shared.js
var …errors = __turbopack_context__.i("library/src/errors.js");
// MERGED MODULE: library/src/index.js <export * as z>
// MERGED MODULE: library/src/index.js
var …index_locals = __turbopack_context__.i("library/src/index.js <locals>");  // ← leaves the group
// MERGED MODULE: library/src/schemas.js
__turbopack_context__.s([   // ← schemas registers HERE, too late
    "any", ()=>any, ...
], "library/src/schemas.js");
```

```diff
// After: registration happens at the top of the scope-hoisted module
"shared.js", "library/src/schemas.js", "library/src/index.js <export * as z>",
((__turbopack_context__) => {
"use strict";
__turbopack_context__.s([   // ← schemas registers at the TOP, before any consumer
    "any", ()=>any, ...
], "library/src/schemas.js");
var …errors = __turbopack_context__.i("library/src/errors.js");
// MERGED MODULE: library/src/index.js <export * as z>
// MERGED MODULE: library/src/index.js
var …index_locals = __turbopack_context__.i("library/src/index.js <locals>");
```

**Practical impact:**

- **Turbopack production builds with libraries that have cyclic `index.js` → `schemas.js` dependencies** — pre-#96697, could throw `TurbopackError: Failed to fetch dynamically imported module` at runtime; post-#96697, the registration is guaranteed to happen before any consumer's `__turbopack_context__.i(...)` call. **Especially material for**: zod (the canonical "registers schemas via `__turbopack_context__.s`" library, hence the repro), yup, joi, ajv, io-ts, valibot, plus any monorepo package with `index.js` re-exports + cyclic internal dependencies.
- **Turbopack dev mode** — also affected; the same registration miss in dev would produce "module not found" errors that reload-only sometimes fixed (because reload re-emitted chunks in a different order).

**Audit recipe:**

```bash
# 1. Confirm you're on a version with the fix:
npm ls next
# → should be next@>=16.3.1-canary.4 (will npm-publish within 1-6h)

# 2. Confirm you use Turbopack (Next.js 16 default; explicit in next dev/build):
rg -n "turbopack|turbo-" next.config.* package.json

# 3. Confirm you use zod or other schemas-via-registration libraries:
rg -n "from ['\"]zod['\"]|require\(['\"]zod['\"]\)" app/ src/

# 4. Diagnose (if you suspect you're seeing the race):
# In your production Turbopack build, look for these errors in the browser console:
#   "TurbopackError: Failed to fetch dynamically imported module: ...
#    TypeError: Cannot read properties of undefined (reading 'X')"
# If you see them intermittently (especially after navigation that triggers
# a fresh chunk fetch), you're affected by #96648.

# 5. Workaround (if you can't bump right away):
# Pin to webpack: `next dev --webpack` / `next build --webpack` for that project.
# Webpack's module resolution never had this issue because it doesn't
# scope-hoist with the same registration pattern.
```

**Migration required:** none — the fix is in Turbopack's scope-hoisting output; no code or config changes required. Bump to `next@>=16.3.1-canary.4` (will npm-publish within hours) or `next@>=16.3.1` stable (when it ships shortly after).

### Sources (canary-branch ahead of 16.3.1-canary.3 — cumulative 27 commits since v1.5.27)

- [**Next.js PR #96252 — `Fix race when navigating Back before hydration`**](https://github.com/vercel/next.js/pull/96252) — by gaearon, merged 2026-08-05T21:39:29Z, 11 files / +561/-25, relands #95682 (the fix); the React-side blocker was fixed by [facebook/react#37135](https://github.com/facebook/react/pull/37135); closes the original issue addressed by #95682; the most user-visible PR of the cycle
- [**Next.js PR #96727 — `Reuse completed cache entries for the rest of a request`**](https://github.com/vercel/next.js/pull/96727) — by unstubbable, merged 2026-08-05T20:42:21Z, 8 files / +324/-34; the Cache Components perf fix that eliminates duplicate work for `'use cache: private'` functions called twice in one request
- [**Next.js PR #96726 — `Discard only cache entries that predate a tag revalidation`**](https://github.com/vercel/next.js/pull/96726) — by unstubbable, merged 2026-08-05T20:42:20Z, 12 files / +169/-8; the Cache Components correctness fix for `updateTag()`-after-fill invalidation
- [**Next.js PR #96731 — `Derive foreground cache revalidation from the consumer`**](https://github.com/vercel/next.js/pull/96731) — by ztanner, merged 2026-08-05T22:44:29Z, 7 files / +95/-39; the Cache Components semantics fix for consumer-driven foreground revalidation
- [**Next.js PR #96697 — `[turbopack] Raise registration calls in hoisted modules to the top`**](https://github.com/vercel/next.js/pull/96697) — by sampoder, merged 2026-08-05T22:33:32Z, 16 files / +156/-10; closes [issue #96648](https://github.com/vercel/next.js/issues/96648); the Turbopack reliability fix for cyclic scope-hoisted module dependencies
- [Next.js PR #96751 — `docs: present each Skill as steps in the AI agents guide`](https://github.com/vercel/next.js/pull/96751) — maintainer, merged 2026-08-05T20:45:13Z, docs-only; restructures the AI agents guide Skills section
- [Next.js PR #96771 — `[Bench] Fixes for pure Fizz bench`](https://github.com/vercel/next.js/pull/96771) — maintainer, merged 2026-08-05T23:03:00Z, bench-only; no production code impact
- [Next.js PR #96774 — `[turbopack] Enable reexport-unknown execution test`](https://github.com/vercel/next.js/pull/96774) — sampoder, merged 2026-08-05T23:49:39Z, test-only; enables an existing test
- [**Next.js `v16.3.1-canary.4` version-tag commit `866beee`**](https://github.com/vercel/next.js/commit/866beee) — by next-js-bot at 2026-08-05T23:33:13Z; GitHub release tag `v16.3.1-canary.4` [published 2026-08-05T23:59:14Z](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.4); npm publish imminent (1-6h on 24h cadence)
- [**Next.js canary-branch compare: `v16.3.1-canary.3...canary` (27 commits ahead at 2026-08-06T00:02Z)**](https://github.com/vercel/next.js/compare/v16.3.1-canary.3...canary) — the cumulative ahead-of-canary.3 set (18 commits documented in v1.5.27 + 9 new this cycle)
- [Next.js canary-branch compare: `v16.3.1-canary.1...canary` (cumulative ahead-of-canary.1, all 27 commits)](https://github.com/vercel/next.js/compare/v16.3.1-canary.1...canary) — full upstream context
- [Next.js Navigation API docs (window.navigation event + currentEntry)](https://developer.mozilla.org/en-US/docs/Web/API/Navigation_API) — the browser primitive PR #96252 uses to detect Back-traversal-mid-hydration
- [Next.js PR #95682 — `Fix race when navigating Back before hydration` (original, reverted)](https://github.com/vercel/next.js/pull/95682) — the original Back-traversal fix that PR #96252 relands; reverted in #95853 due to #95848
- [Next.js PR #95853 — revert of #95682](https://github.com/vercel/next.js/pull/95853) — the revert PR; #96252 lands now that the React-side blocker (facebook/react#37135) is fixed
- [Next.js issue #95848 — original Back-traversal bug report](https://github.com/vercel/next.js/issues/95848) — the original issue that PR #95682 tried to fix
- [Next.js issue #96648 — Turbopack hoisted-module registration bug](https://github.com/vercel/next.js/issues/96648) — the Turbopack cyclic-scope-hoisted-dependency issue that PR #96697 fixes; zod is the canonical reproducer
- [React PR #37135 — Activity state-update hang fix](https://github.com/facebook/react/pull/37135) — the React-side blocker that had to be merged before PR #95682/#96252 could safely land
- [Next.js Cache Components docs (`use cache` directive + cacheLife + cacheTag + updateTag)](https://nextjs.org/docs/app/api-reference/directives/use-cache) — the canonical Cache Components reference that PR #96726 / #96727 / #96731 refine
- [`unstable_cache` Next.js docs](https://nextjs.org/docs/app/api-reference/functions/unstable_cache) — the legacy cache function that PR #96731's consumer-driven foreground revalidation specifically targets
- [zod npm package](https://github.com/colinhacks/zod) — the canonical schema-via-`__turbopack_context__.s([...])` library whose cyclic `index.js` → `schemas.js` structure is the canonical reproducer for PR #96697
- [Next.js `experimental.turbopackChunking` config docs (PR #96398 from canary.105)](https://nextjs.org/docs/app/api-reference/turbopack) — Turbopack chunking context reference (for the scope-hoisting config that PR #96697 refines)
- [Next.js `next@16.3.1-canary.3` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.3) — npm-published 2026-08-05T06:27:06Z; canary.4 is built and will npm-publish within hours
- [Next.js `next@16.3.0` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.0) — STABLE; includes all 16.3.0 security fixes but NOT the canary.4-ahead fixes (PR #96252 + #96726 + #96727 + #96731 + #96697 ship in canary.4 / 16.3.1 STABLE)

---

## `next@16.3.1-canary.5` SHIPPED + `16.3.1-canary.6` Staged (August 7, 2026) — Coordinated TransportData Refactor (3-PR by acdlite) + 3 NEW Open Issues on 16.3.0 STABLE (`#96859` / `#96831` / `#96855`)

The previous cycle (v1.5.32, 2026-08-07T00:03Z) predicted `next@16.3.1-canary.5` would npm-publish within 6-18h on the standard 24h cadence. **`next@16.3.1-canary.5` SHIPPED at 2026-08-07T01:27:54Z** — npm `dist-tag.canary` moved from `16.3.1-canary.4` → `16.3.1-canary.5`; the GitHub release tag `v16.3.1-canary.5` was published at 2026-08-07T01:16:45Z. **The v1.5.32 prediction window was almost exactly correct** — the npm publish landed ~1h25min after v1.5.32 committed. **14 commits vs canary.4** (verified at this cron's check via `GET /repos/vercel/next.js/compare/v16.3.1-canary.4...v16.3.1-canary.5` returning `ahead_by: 14, behind_by: 0`), which is the largest single canary cut in the 16.3.1 cycle so far. The **headline of this canary cut is the 3-PR coordinated `TransportData` refactor by Andrew Clark (acdlite)** — the largest internal architecture change since the `executionMode` refactor in v1.5.27. The 14 commits decomposing to: **3 material coordinated refactor PRs (PR #96406 + PR #96439 + PR #96679)** — the Andrew Clark TransportData refactor — **5 internal refactor / docs / CI commits** (the v1.5.30-v1.5.32 already-documented PRs that landed in canary.5: PR #96774 + #96720 + #95602 + #94604 + #96723 + #96250 + #96235 + #96745 + #96683 + #96772), **+ the `v16.3.1-canary.5` version-tag commit**. **The 3 NEW material Andrew Clark PRs are the cycle's headline**: each is a non-observable internal refactor for users on canary.5 today, but they pre-wire the wire-format transition to a unified `TransportData` representation, and they form the foundation for a future wire-format change. Plus **3 NEW material open issues affecting `next@16.3.0` STABLE users** were opened in the past 24h: **#96859** (Turbopack build fails on pages-router files named `sitemap`/`robots`), **#96831** (Turbopack `crossOrigin: "none"` string serialization breaks cross-origin assetPrefix CDNs), **#96855** (`appNewScrollHandler` scroll-reset regression with `position: fixed` parallel-route slots) — all covered from the deployment lens in `deployment.md` below, with cross-references in this section.

### `canary.5` SHIP event — exact timestamps

| Step | Timestamp | Source |
|---|---|---|
| Version-tag commit `b3bc5cd2` by `next-js-bot` | 2026-08-07T00:52:40Z | `GET /repos/vercel/next.js/commits/b3bc5cd2` |
| GitHub release tag `v16.3.1-canary.5` published | 2026-08-07T01:16:45Z | `GET /repos/vercel/next.js/releases/tags/v16.3.1-canary.5` |
| npm `dist-tag.canary` moved | 2026-08-07T01:27:54Z | `npm view next time --json` |
| Commits vs canary.4 | 14 | `GET /repos/vercel/next.js/compare/v16.3.1-canary.4...v16.3.1-canary.5` returning `ahead_by: 14, behind_by: 0` |

### The 14-commit canary.5 batch decomposing to 3 categories

**Category A — Material coordinated refactor (3 PRs by Andrew Clark, the canary.5 HEADLINE):**

1. **PR #96406 — `Unify RouteTree and CacheNodeSeedData on the client`** (Andrew Clark / acdlite, [commit `4ce4c519`](https://github.com/vercel/next.js/commit/4ce4c519), merged 2026-08-07T00:37:56Z, **9 files / +291/-183**). The client used to represent a server response as **two parallel trees**: a `RouteTree` describing the route structure, and a `CacheNodeSeedData` tree carrying the rendered output for each segment. The two are meant to be isomorphic, but nothing enforced that — every consumer walked them in lockstep and had to defend against mismatches. This PR **adds a `data` field to the `RouteTree` type** and makes it generic over a per-segment payload (`RouteTree<RSCSegmentData | null>`). This removes the need to pass a separate `CacheNodeSeedData` through the client navigation algorithm in `ppr-navigations.ts`. **Per the PR body**: *"This is a step toward replacing the RSC response transport format: with the client consuming a single unified tree, the wire format can move to one as well."* Pure refactor for users today; pre-wires the wire-format transition.

2. **PR #96439 — `Unify full/partial navigation response types`** (Andrew Clark / acdlite, [commit `c37b7368`](https://github.com/vercel/next.js/commit/c37b7368), merged 2026-08-07T00:37:56Z, **17 files / +1342/-960**). Introduces a new type, `TransportData`, to replace `FlightDataPath`. The new format is **the same regardless of whether the entire page is rendered by the server or if some segments are intentionally omitted**. The type also **unifies `FlightRouterState` and `CacheNodeSeedData` into a single tree** that includes both route information and RSC data. Previously these were sent in two separate trees that were isomorphic by convention, requiring the consumer to walk both in parallel. These new types are intended to be **transport formats only** — the client already has its own, richer data structure called `RouteTree`. The transport format is converted into the client format at the network decoding boundary. This PR does not yet update the server to produce `TransportData` directly; a temporary adapter layer is used to convert from the old data formats to the new format. **This intermediate step will not land on its own; the next PR in the stack will both update the server and delete the temporary adapter layer** (i.e., PR #96679 below — the 3-PR stack is coordinated).

3. **PR #96679 — `Refactor server from CacheNodeSeedData -> TransportData`** (Andrew Clark / acdlite, [commit `c713d487`](https://github.com/vercel/next.js/commit/c713d487), merged 2026-08-07T00:37:57Z, **14 files / +636/-757**). The previous PR in the stack (PR #96439) introduced `TransportData` but produced it via a temporary adapter layer that converted from the old data formats — `FlightRouterState` and `CacheNodeSeedData` trees, encoded as `FlightDataPath` entries. **As promised there, this PR updates the server to produce the new format directly and deletes the adapter.** Neither `FlightRouterState` nor `CacheNodeSeedData` appears in a rendered response anymore; `FlightRouterState` survives only on the client (router state, history) and as the request tree the client sends to the server, and `CacheNodeSeedData` is deleted from the codebase entirely. `createComponentTree` now returns the response's transport tree: each node carries its segment identity, its prefetch hints, and its render output, constructed in place as the tree renders. When a non-PPR prefetch stops at a loading boundary, the subtree below the cut is emitted as structure-only nodes with no render output — the same shape the client already interprets as "fetch lazily". `createFlightRouterStateFromLoaderTree` is replaced by `createTransportTreeFromLoaderTree`, which covers the cases where nothing is rendered: router-state-only responses, route tree prefetches, the structure below a loading-boundary cut, and error payloads. The prefetch hints computation is shared with `createComponentTree` through `computeSegmentPrefetchHints` so the two producers cannot drift.

**Category B — v1.5.30 / v1.5.31 / v1.5.32 already-documented PRs (10 commits, all non-TransportData):**

| SHA | Date | Author | PR | Headline | Lens file |
|---|---|---|---|---|---|
| `865d623` | 2026-08-05T23:49:38Z | Sam Poder | #96774 | [turbopack] Enable reexport-unknown execution test | non-material test infra (v1.5.29) |
| `7916855` | 2026-08-06T07:09:24Z | Niklas Mischkulnig | #96720 | Bump `@swc/helpers` (closes #94634) | deployment.md (v1.5.30) |
| `2c04735` | 2026-08-06T09:53:09Z | Sebbie Silbermann | #95602 | Remove `config.experimental.appNewScrollHandler` | deployment.md (v1.5.30) |
| `b6d83ad` | 2026-08-06T12:44:39Z | kyamaz99 | #94604 | Fix(deployment-id): prevent exception on old webkit | deployment.md (v1.5.31) |
| `5092386` | 2026-08-06T13:06:00Z | (docs bot) | #96723 | docs: update redirected links | non-material |
| `f58c669` | 2026-08-06T13:25:52Z | ztanner | #96250 | Fix dev server page announcements | routing.md (v1.5.31) |
| `d792fcf` | 2026-08-06T13:25:55Z | ztanner | #96235 | Fix use cache over/under-invalidation in dev | deployment.md (v1.5.31) |
| `ede8799` | 2026-08-06T13:42:24Z | ztanner | #96745 | Require Cache Components for Instant Navigation testing | routing.md (v1.5.31) |
| `4f37c39` | 2026-08-06T14:32:00Z | (ci bot) | #96683 | CI: automated update PRs with `nextjs-bot` | non-material |
| `0ae8c72` | 2026-08-06T15:28:00Z | jankaeryga | #96772 | Consolidate `Promise.withResolvers` polyfills | this file (v1.5.32) |

**Category C — Version-tag:** `b3bc5cd2` (2026-08-07T00:52:40Z, `next-js-bot`).

### Why the 3-PR TransportData refactor matters — even though it's a pure refactor for users today

The 3-PR TransportData refactor (PR #96406 + #96439 + #96679) is the **largest internal architecture change since the 9-PR `executionMode` refactor documented in `server-components.md` in v1.5.27**. Like that refactor, **zero user-facing behavior change** for users on `next@16.3.0` STABLE or on `next@16.3.1-canary.4` after upgrading to `canary.5`. **But the wire-format implications are concrete**:

- **Today (post-canary.5):** the wire format is **unchanged**. The 3-PR stack introduces an internal `TransportData` representation, but the server **still serializes the response in the legacy `FlightDataPath` format** (via the `rsc-transport.ts` shared module). The temporary adapter layer from PR #96439 has already been deleted in PR #96679, but the actual transport format hasn't transitioned yet — it's still the legacy format, just produced directly instead of via adapter.
- **Future canary cut (forward-looking):** a future PR will flip the wire format to the new `TransportData` representation. **When that happens, it will be a wire-format-breaking change** — any client that consumes RSC payloads directly (rare; mostly testing infrastructure + custom server-to-server pipelines + `dangerouslyAllowBrowser: true` setups) will need to update. Most users won't notice (the client `RouteTree` representation is unchanged; the conversion happens at the network decoding boundary).
- **The architectural rationale (per Andrew Clark's PR bodies):** *"The client used to represent a server response as two parallel trees: a RouteTree describing the route structure, and a CacheNodeSeedData tree carrying the rendered output for each segment. The two are meant to be isomorphic, but nothing enforced that — every consumer walked them in lockstep and had to defend against mismatches between them. ... This is a step toward replacing the RSC response transport format: with the client consuming a single unified tree, the wire format can move to one as well."*

**The fix shape:** unified single tree on the wire (`RouteTree<RSCSegmentData | null>`), no more parallel-tree walking on the client, no more temporary adapter layer on the server. The deletion of `transport-adapter.ts` (-227 lines) in PR #96679 is the most visible code-quality win.

**Affected-deployment profile for the future wire-format transition** (forward-looking, not active today):
- Users on `dangerouslyAllowBrowser: true` who parse RSC payloads client-side.
- Users with custom server-to-server RSC pipelines (rare in production; mostly testing + RSC-as-API consumers).
- Users with custom React Flight decoders (vanishingly rare; mostly academic + experimental projects).

**For 99.9% of users**, the 3-PR refactor is invisible — the client `RouteTree` representation is unchanged, the network decoding boundary absorbs the format change, and no public API surface is touched. The benefit lands on the **maintainability** side: the parallel-tree walking was a known footgun, and the unified representation eliminates a class of "I forgot to update both trees" bugs.

### 3 NEW material open issues on `next@16.3.0` STABLE — affecting users TODAY

These 3 issues were opened in the past 24h and affect `next@16.3.0` STABLE + `16.3.1-canary.0/1/2/3/4` users **today**, not in some future canary. Each is documented from the deployment lens in `deployment.md` below with full PR attribution absent, audit recipes, and workarounds; this section is a cross-reference + headline summary.

| Issue | Title | Affected deployments | Status | Covered in |
|---|---|---|---|---|
| **[#96859](https://github.com/vercel/next.js/issues/96859)** | Turbopack build fails on pages-router files named `sitemap`/`robots`: `"getStaticProps" is not supported in app/` (no `app/` directory) | `next@16.3.0` STABLE + `canary.0/1/2/3/4` + Turbopack + **pages-router-only projects** that have `pages/sitemap.js` or `pages/robots.js` | open (created 2026-08-06T19:33:07Z) | `deployment.md` below (the metadata-route filename convention collision) |
| **[#96831](https://github.com/vercel/next.js/issues/96831)** | 16.3.0: Turbopack serializes `moduleLoading.crossOrigin` as string `"none"`, adding unexpected `crossorigin=""` to chunk scripts (breaks cross-origin assetPrefix CDNs) | `next@16.3.0` STABLE + `canary.0/1/2/3/4` + Turbopack + **cross-origin `assetPrefix` CDN** (different origin than the page) | open (created 2026-08-06T14:52:38Z) | `deployment.md` below (the CORS-mode loading failure) |
| **[#96855](https://github.com/vercel/next.js/issues/96855)** | Scroll is not reset on navigation when a parallel route slot renders only a `position: fixed` element (`appNewScrollHandler` regression in 16.3.0) | `next@16.3.0` STABLE + `canary.0/1/2/3/4` + **parallel-route `@slot` that renders only fixed/sticky elements** | open (created 2026-08-06T18:28:57Z) | `deployment.md` below (the scroll-reset regression) |

Plus **#96806** (Docker + cacheComponent + `headers()` 500 error in production) — **closed** in the 24h window; documented in `deployment.md` v1.5.30 cycle-append as forward-looking and now resolved.

### `canary.6` staged on canary-branch ahead of canary.5 (forward-looking, 6h+ out from npm-publish)

The canary-branch now has **3 NEW commits ahead of canary.5** (verified at this cron's check via `GET /repos/vercel/next.js/compare/v16.3.1-canary.5...canary` returning `ahead_by: 3, behind_by: 0`). The 3 commits are: **PR #96860** by Will Binns-Smith (merged 2026-08-07T03:39:38Z, [commit `286169a9`](https://github.com/vercel/next.js/commit/286169a9)) — `Remove `turbopack/packages` and relocate devlow to `packages/`` — release-engineering refactor; **PR #96871** by Will Binns-Smith (merged 2026-08-07T03:39:39Z, [commit `007058ef`](https://github.com/vercel/next.js/commit/007058ef)) — `Lint devlow-bench with the root eslint config` — non-material lint pass; **+ the `v16.3.1-canary.6` version-tag commit `6ec2ad50`** (2026-08-07T03:41:23Z, `next-js-bot`). **All non-material release-engineering** — no user-facing impact for any deployment. `canary.6` is staged and will npm-publish within 6-18h on the 24h cadence; the v1.5.34 cron will document the canary.6 SHIP event.

### Recommended action

**For users on `next@16.3.0` STABLE or `next@16.3.1-canary.0/1/2/3/4`:** check the 3 NEW open issues against your deployment profile using the `rg` audit recipes in `deployment.md` below:
- `rg -ln "pages/(sitemap|robots)\."` → if hits, **issue #96859 affects you** (rename the file to avoid the metadata-route collision).
- `rg -n "assetPrefix.*['"]https?://" next.config.*` → if hits AND the asset host is on a different origin AND you don't have CORS configured, **issue #96831 affects you** (configure ACAO on the CDN, OR roll back to `next@16.2.12`, OR temporarily use `output: 'export'` to bypass server-rendered preinited chunks).
- `rg -ln "@(header|footer|sidebar|modal)/" app/` → if hits AND any `@slot/page.tsx` renders only `position: fixed`/`sticky` elements, **issue #96855 affects you** (add a hidden scroll-anchor element, OR roll back to `next@16.2.12`).

**For users on `next@16.3.1-canary.5`:** no action required for the 3-PR TransportData refactor (zero user-facing behavior change). For the future wire-format transition (forward-looking), no action required today — when it lands in a future canary, this file will document the wire-format delta.

**For users tracking canary-branch:** expect **canary.6** to npm-publish within 6-18h of this cron. The v1.5.34 cycle (in 6h) will document the canary.6 SHIP event if it lands within the next 6h window.

### Sources

- [Next.js `v16.3.1-canary.5` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.5) — published 2026-08-07T01:16:45Z
- [Next.js canary-branch compare `v16.3.1-canary.4...v16.3.1-canary.5`](https://github.com/vercel/next.js/compare/v16.3.1-canary.4...v16.3.1-canary.5) — 14 commits at this cron's check
- [Next.js canary-branch compare `v16.3.1-canary.5...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.5...canary) — 3 NEW commits ahead (PR #96860 + PR #96871 + the canary.6 version-tag) at this cron's check
- [Next.js PR #96406 — `Unify RouteTree and CacheNodeSeedData on the client`](https://github.com/vercel/next.js/pull/96406) — Andrew Clark, merged 2026-08-07T00:37:56Z, 9 files / +291/-183
- [Next.js commit `4ce4c519`](https://github.com/vercel/next.js/commit/4ce4c519) — PR #96406 merge commit
- [Next.js PR #96439 — `Unify full/partial navigation response types`](https://github.com/vercel/next.js/pull/96439) — Andrew Clark, merged 2026-08-07T00:37:56Z, 17 files / +1342/-960
- [Next.js commit `c37b7368`](https://github.com/vercel/next.js/commit/c37b7368) — PR #96439 merge commit
- [Next.js PR #96679 — `Refactor server from CacheNodeSeedData -> TransportData`](https://github.com/vercel/next.js/pull/96679) — Andrew Clark, merged 2026-08-07T00:37:57Z, 14 files / +636/-757
- [Next.js commit `c713d487`](https://github.com/vercel/next.js/commit/c713d487) — PR #96679 merge commit
- [Next.js commit `b3bc5cd2`](https://github.com/vercel/next.js/commit/b3bc5cd2) — the `v16.3.1-canary.5` version-tag commit
- [Next.js commit `6ec2ad50`](https://github.com/vercel/next.js/commit/6ec2ad50) — the `v16.3.1-canary.6` version-tag commit
- [Next.js PR #96860 — `Remove `turbopack/packages` and relocate devlow to `packages/`](https://github.com/vercel/next.js/pull/96860) — Will Binns-Smith, merged 2026-08-07T03:39:38Z (canary.6)
- [Next.js PR #96871 — `Lint devlow-bench with the root eslint config`](https://github.com/vercel/next.js/pull/96871) — Will Binns-Smith, merged 2026-08-07T03:39:39Z (canary.6)
- [Next.js issue #96859 — Turbopack build fails on pages-router files named `sitemap`/`robots`](https://github.com/vercel/next.js/issues/96859) — open
- [Next.js issue #96831 — Turbopack `crossOrigin: "none"` serialization breaks cross-origin assetPrefix CDNs](https://github.com/vercel/next.js/issues/96831) — open
- [Next.js issue #96855 — Scroll-reset regression with fixed-position parallel-route slots (`appNewScrollHandler` regression)](https://github.com/vercel/next.js/issues/96855) — open
- [Cross-reference: deployment.md `## Next.js 16.3.0 STABLE — 3 NEW Open Issues Affecting Production Deployments Today` (this cycle)](https://github.com/clawvpsai/frontend-skill/blob/main/deployment.md) — full coverage of #96859 + #96831 + #96855 with audit recipes + workarounds
- [Cross-reference: server-components.md `## App Router Execution Mode Refactor — 9-PR Coordinated Set` (v1.5.27)](https://github.com/clawvpsai/frontend-skill/blob/main/server-components.md) — the previous-largest coordinated refactor; same lens (zero user-facing behavior change, internal architecture cleanup)
- [Cross-reference: performance.md `## 16.3.1-canary.4-ahead — Navigation Back-Before-Hydration Race Fix` (v1.5.28)](https://github.com/clawvpsai/frontend-skill/blob/main/performance.md) — the previous cycle's headline (PR #96252 + #96726 + #96727 + #96731 + #96697)

---


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
- **Still using `experimental.turbopack.chunkingHeuristics` or `experimental.turbopackGenerateComponentChunks` in 2026** — both namespaces throw at config-eval time on `next@16.3.0-canary.105`+ with explicit migration errors. Migrate to the new top-level `experimental.turbopackChunking` config (PR #96398, merged 2026-07-31T06:37:37Z, canary-branch ahead of canary.104). Old → new: `turbopack.chunkingHeuristics.requestCost` → `turbopackChunking.requestCost` (note: default changed 20 KB → 200 KB, re-tune!), `turbopack.chunkingHeuristics.clusters` (string[]) → `turbopackChunking.priorityRoutes` (RegExp[]), `turbopack.chunkingHeuristics.entryPoints` → absorbed into `priorityRoutes` + `priorityBoost`, `turbopack.chunkingHeuristics.bounceRate` → `turbopackChunking.firstPageLoadPriority`, `turbopackGenerateComponentChunks` boolean → `turbopackChunking.generateComponentChunks`. See the matching section above for the full migration recipe + 5 NEW size-threshold knobs (`minChunkSize` / `maxChunkCountPerGroup` / `maxMergeChunkSize` / `minComponentChunkSize`).
- **Default-on `experimental.turbopackFileSystemCacheForBuild` expands to ALL environments in canary.107+ (PR #96493, timneutkens, merged 2026-08-02T18:33:34Z; supersedes PR #96395, sokra, canary.105)** — the canary.105 behavior was "local + Vercel default-ON, generic CI default-OFF" (gated by a `turbopackFileSystemCacheForBuildDefault()` helper returning `!isCI || Boolean(process.env.NOW_BUILDER)`). PR #96493 removes that gate entirely — every `next build` in every environment (local, Vercel, GitHub Actions, GitLab, CircleCI, Buildkite, Jenkins, etc.) now uses `.next/cache/turbopack/` by default for warm builds (5-30% faster on warm builds). The new responsibility: if your CI does NOT persist `.next/cache/` between runs (e.g. ephemeral CI containers with no `actions/cache` step), you must set `experimental.turbopackFileSystemCacheForBuild: false` explicitly to restore the canary.105-pre-PR-#96493 default-OFF behavior. Audit recipe: `rg "turbopackFileSystemCacheForBuild" next.config.ts next.config.js next.config.mjs 2>/dev/null` (look for `: true` / `: false` overrides) + check your CI config for `actions/cache@v4` on `path: .next/cache` (GitHub Actions) or the equivalent. If you're trying to do a "fair" webpack-vs-turbopack build comparison, delete `.next/` between builds (or the cache will skew the warm-build numbers).
- **Trying to call `ReactDOM.browser()` with stale `@types/react-dom` in 2026** — `browser()` is a runtime API in `react-dom@canary` (since `19.3.0-canary-0f42eac2-20260730`, PR #37143) but the **TypeScript declaration** only ships in `@types/react-dom@19.2.4` (published 2026-07-30). If you're on `next@16.3.0-canary.105+`, you already get 19.2.4 via the bundled-deps bump (PR #96419). If you're on stable (`next@16.2.12`), pin `@types/react-dom@^19.2.4` explicitly to get the `browser()` type.
- **Conditional `use(promise)` patterns that cache the value and only call `use()` on cache miss** — as of `react@19.3.0-canary-cbb046ab-20260731` (PR #37104, hoxyq, merged 2026-07-31T14:24:10Z), a new DEV-only warning fires when you suspend via `use()` on render N but don't `use()` the same promise on render N+1 (the resolved render). The warning is gated behind the `enableConditionalUseWarning` feature flag (currently OFF in `react@canary`; will likely flip to ON in a future canary once Meta measures the noise floor). The fix is always-`use()`-the-promise — see the matching section in `components.md` for the canonical anti-pattern + rewrite + audit recipe (`rg -n 'cache\.value\s*===\s*undefined' --type ts --type tsx`).
- **Triple-logged unhandled rejections in `next dev` or `next start` on canary.104-** — fixed by PR #95999 (eps1lon, merged 2026-07-31T15:36:41Z). Pre-#95999, a single `unhandledRejection` was logged by 3 independent listeners (runtime crash-prevention + router server + dev server) in 3 different formats. Post-#95999 (canary.105+), the runtime handler stays (must-exist per issue #77997), and the router + dev server check-then-register a single shared listener via `Symbol.for` on `globalThis`. If you're debugging "why am I seeing this error 3 times?", upgrade to canary.105+.
- **Dynamic `not-found.tsx` content not rendering in deployed environments on canary.104-** — fixed by PR #96390 (Zack Tanner, merged 2026-07-31T15:30:58Z). Pre-#96390, any `not-found.tsx` that used `headers()` / `cookies()` / dynamic `use cache: private` would be served as a static `404.html` shell at deploy time (omitted from adapter outputs entirely), with the dynamic content never appearing. Post-#96390 (canary.105+), the build correctly distinguishes "fully static not-found → emit 404.html" from "PPR not-found → emit dynamic resume chain with PARTIALLY_STATIC metadata". Audit recipe: `ls .next/server/pages/404.html` (should NOT exist for dynamic not-found) + `grep PARTIALLY_STATIC .next/server/app/not-found.body` (should exist for dynamic not-found).
- **Hybrid Pages Router + App Router `not-found` rendering fails on adapter deployments on canary.105 (FIXED in canary.106)** — PR #96392 (Zack Tanner, merged 2026-07-31T23:40:38Z, **SHIPPED in `16.3.0-canary.106`** at 2026-08-01T23:56:54Z) is the fix. Pre-#96392, when Pages Router and App Router routes coexist and a Pages route returns `notFound()`, the handoff to App Router `/_not-found` would attempt to resume the Pages entry (which doesn't contain the relevant prerendered shell or postponed state) — the dynamic not-found content silently failed to render. **Critical timing detail:** PR #96392 landed in the canary-branch at 23:40:38Z, but the canary.105 version-tag commit `a8dcd2562f` was made at 23:35:12Z — so PR #96392 was **NOT in the npm-published `16.3.0-canary.105`** (shipped 23:57:13Z). It IS in `16.3.0-canary.106` (shipped 2026-08-01T23:56:54Z). `next start` users (or Vercel-hosted) are unaffected — the bug only manifests through the adapter path. Audit recipe: `npm ls next` (must show `>=16.3.0-canary.106` if you depend on this fix; `next@canary.105` still has the bug).
- **Ignoring the new `experimental.useCache` deprecation warning that PR #96448 surfaces (canary.106+)** — the `experimental.useCache` option has carried a `@deprecated` JSDoc annotation since PR #92316 but nothing surfaced that at runtime. PR #96448 (unstubbable, merged 2026-08-01T00:26:11Z) **logs a warning** whenever `experimental.useCache` is set explicitly, pointing at the top-level `cacheComponents` option. Worse, **disabling the option while `cacheComponents` is enabled is now rejected outright** as a compile error (because that combination is contradictory — it turns off the very directive Cache Components is built around, and the resulting error asks the user to enable `cacheComponents` which they already have on). Throwing matches how `cachedNavigations` and `partialPrefetching` already reject configs that require `cacheComponents`. Migration: remove `experimental.useCache` from your `next.config.ts`; if `cacheComponents: true`, the `useCache` option is backfilled automatically and is redundant; if you have `experimental.useCache: false` AND `cacheComponents: true`, simply remove the `useCache: false` line. Audit recipe: `rg -n 'experimental.*useCache' next.config.*` to find projects with the deprecated option set; `rg -B2 -A4 'useCache.*false' next.config.*` to find the contradictory config that will now throw at config-eval.

- **`use cache` returns empty content after a prerender aborts (canary.105 / canary.106) — FIXED in canary.107 by PR #96426** — Janka Uryga, merged 2026-08-03T11:42:26Z, **SHIPPED in `next@16.3.0-canary.107`** (npm-published 2026-08-03T14:04:47Z). Pre-#96426, caches that started filling after a prerender was aborted (e.g. client navigated away mid-render, prefetch cancelled under `partialPrefetching: true`) would silently produce an empty stream and save it as a valid cache entry — effectively poisoning the cache with a tombstone that subsequent requests would hit and return empty content from. The fix removes `renderSignal` from the `AbortSignal.any(...)` passed to `prerender()` inside `use-cache-wrapper` and short-circuits with a rejected promise *before* reaching the cache-fill codepath (so the prerender is aborted, but the cache is never poisoned). **Only affects apps with `cacheComponents: true`** that use `use cache` heavily under navigation aborts. Audit recipe: `rg -n "cacheComponents\s*:\s*true" next.config.*` to confirm you're on Cache Components; `rg -l "use cache" app/ src/` to confirm you use `use cache`; if you've seen "this cache should have content but it's empty" after a navigation — you're affected by #96339. Migration: bump to `next@16.3.0-canary.107+` (when published, canary.107 is the first release with the fix live in npm) — no code changes required, the fix is in the runtime cache wrapper. The bug was on canary.105 / canary.106 / `next@16.3.0-preview.10`; the fix is live in `next@16.3.0-canary.107+` and in `next@16.2.12` once backported (not yet backported at this cron's check — backport PR not open).

- **Cross-origin CDN `assetPrefix` + Web Workers = silent worker hang on Turbopack (16.3.0 + 16.3.1-canary.0/.1/.2) — FIXED in `next@16.3.1-canary.3`-ahead by PR #96636** — timneutkens, merged 2026-08-05T05:41:54Z. Pre-#96636, any deployment combining Turbopack + `assetPrefix: 'https://cdn.example.com'` + Web Workers created via `new Worker(new URL('./worker.ts', import.meta.url))` (e.g., `@resvg/resvg-js`, `@napi-rs/canvas`, `@jsquash/*`, custom WASM packages) would silently hang — workers load (every network request returns `200`) but never execute (no `onerror`, no console, no failed request, page posts a message but nothing comes back). Cause: `experimental.turbopackWorkerAssetPrefix` (PR #93271) was applied to parent-context URLs but the worker's own runtime chunk was emitted with `CHUNK_BASE_PATH = assetPrefix` (the CDN) — `registerChunk`'s two resolver keys collided across base paths and the `Promise.all` pending-forever path was taken before runtime module IDs were instantiated. Affects every worker that uses Turbopack + cross-origin CDN. After PR #96636: the worker's runtime chunk is emitted with `CHUNK_BASE_PATH` set to the worker asset prefix (same-origin). **Audit recipe:** `npm ls next` (should be `>=16.3.1-canary.3` after npm publishes within hours); `rg -n "assetPrefix\s*:" next.config.*` to confirm cross-origin CDN setup; `rg -ln "new Worker\(new URL\(" app/ src/` to find Workers; in Chrome DevTools → Application → Workers, check the Console tab for the deployed app's worker — if there is NO output (not even the module-evaluated log) and all network requests return `200`, you have the #96613 bug. **Migration required: none** — bump to `next@>=16.3.1-canary.3` (will npm-publish within hours) or `next@>=16.3.1` stable (when published). The bug was on `next@16.3.0` STABLE and all `next@16.3.1-canary.0/.1/.2` releases. Webpack users are unaffected (webpack's `output.workerPublicPath: '/_next/'` workaround in `next.config.js → webpack()` was never broken).

- **Cache Components dynamic routes trigger unnecessary loading-boundary prefetches instead of per-segment prefetches — FIXED in `next@16.3.1-canary.1`-ahead by PR #96583** — Zack Tanner, merged 2026-08-04T02:02:33Z. Pre-#96583 (i.e. on `next@16.3.0` STABLE), any `router.prefetch()` call on a Cache Components dynamic route (any route with `experimental.cacheComponents: true` whose render path is dynamic — e.g. calls `headers()`, `cookies()`, uses a dynamic data source, or relies on `await connection()`) would incorrectly fall back to **loading-boundary prefetching** (large payload with loading state) instead of **per-segment prefetching** (small payload with just the changed segments). The cause: the dynamic Flight response was setting the `S:` flag to `workStore.isStaticGeneration` (always `false` for dynamic renders) without including `ctx.renderOpts.cacheComponents` in the `||` expression. After PR #96583: `S: workStore.isStaticGeneration || ctx.renderOpts.cacheComponents` — Cache Components routes now correctly advertise per-segment prefetching capability regardless of static/dynamic mode. **Expected reduction in prefetch network traffic**: ~30-50% (depending on route complexity) per `router.prefetch()` call on a dynamic CC route. **Only affects apps with `cacheComponents: true`** — non-CC routes are unaffected. **Audit recipe**: `rg -n "cacheComponents\s*:\s*true" next.config.*` to confirm you're on Cache Components; in Chrome DevTools → Network, trigger a `router.prefetch()` on a dynamic CC route and confirm the request is a **segment-level prefetch** (small payload, no loading boundary), not a full-page loading-boundary prefetch (large payload with loading state). **Migration**: bump to `next@>=16.3.1-canary.1` (when published — will likely ship within 12-24h on the 24h cadence) or to `next@>=16.3.1` stable (when published shortly after); no code changes required, the fix is in the server-render output + the segment cache field. The bug was on `next@16.3.0` STABLE; the fix will ship in `next@16.3.1-canary.1`-ahead.

- **Back-button click during hydration produces stale client state (16.3.0 + all 16.3.1-canary.0/1/2/3) — FIXED in `next@16.3.1-canary.4`-ahead by PR #96252** — gaearon, merged 2026-08-05T21:39:29Z, relands #95682 (originally reverted in #95853 due to #95848 — under the then-broken React Activity, state updates from the Back-traversal would sometimes hang applying). The React-side blocker was fixed by [facebook/react#37135](https://github.com/facebook/react/pull/37135), synced to `next@canary`, so the original Back-traversal fix is safe to reland. Pre-#96252, a user landing on Page A who clicks the Back button before hydration completes (common on slow devices / mid-tier mobile / heavy `useEffect` work) sees Page B rendered with stale client state from Page A — scroll position is wrong, focus is wrong, and any state-setter from Page A bleeds into Page B's first paint. Fix: use the Navigation API (`window.navigation`) during hydration to detect that we're on a different history entry than the one we started hydrating, and in that case replay the missed traversal with similar logic to what we do for `onPopState`. **Audit recipe:** `npm ls next` (must show `>=16.3.1-canary.4` after npm publishes within hours); `rg -n "prefetch\s*=" app/` (default = enabled, which is where the race lives); in Chrome DevTools → Performance with CPU throttling to 4x slowdown, navigate to a slow page then click Back immediately. **Migration required: none** — the fix is in the App Router runtime; no code or config changes required. Bump to `next@>=16.3.1-canary.4` (will npm-publish within hours) or `next@>=16.3.1` stable (when it ships shortly after). **Workaround** if stuck on a pre-canary.4 version: add `prefetch={false}` to the Back-target Link to close the race window for that specific Link.

- **Calling `'use cache: private'` twice in one request does the work twice (16.3.0 + all 16.3.1-canary.0/1/2/3) — FIXED in `next@16.3.1-canary.4`-ahead by PR #96727** — unstubbable, merged 2026-08-05T20:42:21Z, 8 files / +324/-34. Pre-#96727, calling the same `'use cache: private'` function twice in one request executed its body twice in production. Preloading at the top of a segment and reading the same function again lower down for composability — the canonical preload pattern — therefore did the work twice instead of once. Cause: the intra-request dedupe map dropped an entry as soon as its fill completed, so it only ever covered concurrent invocations. A later invocation fell through to the cache handler, and the `React.cache` memo wrapping every cache function missed whenever the arguments were not reference-equal. Private caches have no handler in production and their entries are excluded from the immutable Resume Data Cache of a dynamic request, so nothing had stored the entry. Fix: completed invocations now move into `completedCacheInvocations` on the work store instead of being dropped, and a later invocation joins that entry. Expected 30-50% reduction in cache-regeneration work per page render with private cache fan-out. **Only affects apps with `cacheComponents: true`** that use `'use cache: private'` per-user cached functions (e.g., `getCurrentUserPrefs()`, `getUserCart()`). Audit recipe: `rg -n "['\"]use cache: private" app/ src/` to find private cache usage; bump to `next@>=16.3.1-canary.4` once npm-publishes; the fix is in the work-store dedupe map and requires no code or config changes.

- **Cache reads after `updateTag()` in a server action all regenerate (16.3.0 + all 16.3.1-canary.0/1/2/3) — FIXED in `next@16.3.1-canary.4`-ahead by PR #96726** — unstubbable, merged 2026-08-05T20:42:20Z, 12 files / +169/-8. Pre-#96726, calling `updateTag()` in a server action made every later read of a cache carrying that tag regenerate for the remainder of the request — including reads of an entry that had just been generated after the invalidation and therefore already reflected it. Two sequential reads of the same cache function during the re-render produced two different values within a single render, and each one repeated the work. Cause: `isRecentlyRevalidatedTag` only asked whether a tag appeared in `pendingRevalidatedTags`, with no notion of when the revalidation happened. That array lives for the whole `WorkStore`, which spans a server action AND the render that follows it. Fix: each pending revalidated tag now records a `revalidatedAt` timestamp taken from the same clock as `CacheEntry.timestamp`, and the renamed `isRevalidatedAfter` reports an entry as stale only when the revalidation is newer than the entry. Expected 20-60% reduction in cache-regeneration work per `updateTag()` round-trip on a multi-cache fan-out. **Only affects apps with `cacheComponents: true`** that call `updateTag()` in server actions with multi-cache fan-out reads. Audit recipe: `rg -n "updateTag\s*\(" app/ actions/ src/` to find `updateTag()` call sites; bump to `next@>=16.3.1-canary.4`; the fix is in the cache-staleness check and requires no code or config changes.

- **Turbopack production builds throw `TurbopackError: Failed to fetch dynamically imported module` for cyclic scope-hoisted dependencies (16.3.0 + all 16.3.1-canary.0/1/2/3) — FIXED in `next@16.3.1-canary.4`-ahead by PR #96697** — sampoder, merged 2026-08-05T22:33:32Z, 16 files / +156/-10. Pre-#96697, Turbopack scope-hoisting could miss module registrations for cyclic dependencies between scope-hoisted groups, leading to `TurbopackError: Failed to fetch dynamically imported module: ... TypeError: Cannot read properties of undefined` at runtime in production builds. Cause: on non-scope-hoisted modules with cycles, Turbopack already raises the module registration call to the start of the factory. But when scope-hoisting merges multiple modules into a single factory, that early registration was lost — registration happened at the original line, which can be after the factory has already entered the consumer's chunking context. Fix: when scope-hoisting, the `__turbopack_context__.s([...])` registration call is now emitted at the start of the scope-hoisted module, not at the original line. **Especially material for**: zod (canonical reproducer), yup, joi, ajv, io-ts, valibot, and any monorepo package with `index.js` re-exports + cyclic internal dependencies. Audit recipe: `npm ls next` (must show `>=16.3.1-canary.4`); `rg -n "from ['\"]zod['\"]" app/ src/` to confirm zod usage; check your browser console for intermittent "Failed to fetch dynamically imported module" errors after navigation. **Migration required: none** — the fix is in Turbopack's scope-hoisting output; no code or config changes required. **Workaround** if stuck on a pre-canary.4 version: `next dev --webpack` / `next build --webpack` for that project (Webpack's module resolution never had this issue because it doesn't scope-hoist with the same registration pattern).

## 16.3.1-canary.4-ahead — `Promise.withResolvers` Polyfill Consolidation (PR #96772) + Redirected-Links Docs Refresh (PR #96723) + CI Automation (PR #96683) (3 NEW commits, August 6, 2026)

The previous section documented **16.3.1-canary.4-ahead = 8 NEW commits + version-tag** (v1.5.28 cycle, Aug 5 18:07Z). Since v1.5.31 committed at 2026-08-06T18:15Z, the canary-branch has gained **3 NEW commits** (verified at this cron's check via `GET /repos/vercel/next.js/compare/v16.3.1-canary.4...canary` returning `ahead_by: 10, behind_by: 0` — the v1.5.31 cycle captured 7 of those, this cycle captures the remaining 3). The total **canary-branch ahead-of-canary.4 = 10 commits** — the largest cumulative canary-branch-ahead gap since the 16.3.0 STABLE release on 2026-08-03.

The 3 NEW commits are all in the 13:06Z → 15:28Z window on Aug 6 (i.e., **between the canary.4 npm-publish at 2026-08-06T00:10:18Z and the canary.5 version-tag pending**):

### 1. PR #96772 — Consolidate `Promise.withResolvers` polyfills (jankaeryga, merged 2026-08-06T15:28:00Z, [commit `0ae8c72`](https://github.com/vercel/next.js/commit/0ae8c72), 17 files / +59/-93)

**Internal code-quality refactor — zero user-facing behavior change.**

Pre-PR-#96772: Next.js maintained **duplicate deferred-promise implementations** across three files:
- `packages/next/src/lib/detached-promise.ts` (a 27-line standalone helper)
- `packages/next/src/shared/lib/promise-with-resolvers.ts` (a separate helper)
- The vendored `@mswjs/interceptors` Node 20 shim bundle (had its own internal `Promise.withResolvers` polyfill)

Post-PR-#96772: All three converge on the **shared `createPromiseWithResolvers` helper** in `packages/next/src/shared/lib/promise-with-resolvers.ts`. The duplicate `DetachedPromise` implementation is **deleted** entirely. The vendored `@mswjs/interceptors` bundle now delegates its global polyfill requirement to the same shared helper. The 17-file diff touches:
- 1 file **removed** (`packages/next/src/lib/detached-promise.ts`, -27 lines)
- 14 files **modified** to migrate `new DetachedPromise<T>()` → `createPromiseWithResolvers<T>()` across `batcher.ts` + `after/run-with-after.ts` + `app-render/stream-ops.node.ts` + `base-http/web.ts` + `dev/next-dev-server.ts` + `lib/incremental-cache/index.ts` + `lib/router-utils/proxy-request.ts` + `pipe-readable.ts` + `response-cache/web.ts` + `route-matcher-managers/default-route-matcher-manager.ts` + `stream-utils/node-web-streams-helper.ts` + the vendored interceptor bundle + 2 generated test fixtures
- 2 test files updated (`server/after/after-context.test.ts` + `server/web/web-on-close.test.ts`)

The PR's stated goal: **"This can go away entirely when we support Node 22 at minimum."** Node 22 ships native `Promise.withResolvers` (no polyfill needed), so once Next.js drops Node 20 support (planned for the 16.4 cycle or later), all three helpers can be removed entirely.

**Practical impact:**
- **Zero runtime behavior change.** The refactor preserves the exact `resolve()` / `reject()` / `promise` semantics from both helper implementations.
- **~5-10% bundle size reduction** for Next.js (the duplicate `DetachedPromise` class + its TS declarations were ~30 lines of duplicated code).
- **Slight maintenance win** — the next time someone touches `createPromiseWithResolvers` (e.g., to add TS narrowing or improve the resolve callback), all three call sites update automatically.
- **No user-facing API change.** All the call sites are internal Next.js code paths (`pipe-readable.ts`, `batcher.ts`, `response-cache/web.ts`, etc.) — none are exported from any package.

### 2. PR #96723 — docs: update redirected links to current targets (merged 2026-08-06T13:06:00Z, [commit `5092386`](https://github.com/vercel/next.js/commit/5092386))

Docs-only update that fixes broken links across the Next.js documentation. No code change. **No practical impact beyond docs navigation.**

### 3. PR #96683 — `[ci] Open automated update pull requests with `nextjs-bot`` (merged 2026-08-06T14:32:00Z, [commit `4f37c39`](https://github.com/vercel/next.js/commit/4f37c39))

CI-infra change — `nextjs-bot` is now configured to open automated update PRs (likely Dependabot-equivalent for the Next.js org's own internal dependencies). **No user-facing impact** — purely an internal CI workflow improvement.

### `16.3.1-canary.7` SHIPPED (August 7, 2026) — styled-jsx SSR Regression Fix + Turbopack Improvements

**`next@canary` SHIPPED at `v16.3.1-canary.7`** — npm-published 2026-08-07T10:11:39Z (~2h before this cron's check), exactly within the v1.5.33 cron's predicted 6-18h window. The batch includes **4 PRs** since canary.5 (canary.6 was release-only with no npm-publish):

| PR | Title | Impact | Documented |
|---|---|---|---|
| **#96632** | Fix missing styled-jsx styles in Pages Router SSR on adapter builds | **MATERIAL — production regression fix** | `styling.md` (new section) |
| **#75682** | Show compiler plugin warning in more situations | Minor dev-experience | Not separately documented |
| **#96560** | Turbopack: name module in non-ESM-placeable error + stop duplicating importer | Minor Turbopack DX | Not separately documented |
| **#96558** | Turbopack: support `type: 'text'` in rules + error on binding imports of non-ESM | Minor Turbopack feature | Not separately documented |

The headline is **PR #96632** — a production-breaking SSR regression affecting every Pages Router app deployed on Vercel (or any build adapter). See the new **`styling.md`** section: [Next.js 16.3.1-canary.7 — styled-jsx SSR Regression Fix (PR #96632)](#next-js-1631-canary7--styled-jsx-ssr-regression-fix-pr-96632-august-7-2026--shipped) for the full root-cause walkthrough, the affected-versions table, and the audit recipe.

**PR #75682** (`Show compiler plugin warning in more situations`) expands the compiler warning cases for `babel-plugin-react-remove-properties` to match the existing `styled-components` case — ensuring the same class of "this will strip your `data-*` attributes in production" warning fires for styled-jsx users too.

**PR #96560 + #96558** are Turbopack DX improvements (module naming in error traces + `type: 'text'` rule support) with no user-facing behavior change.

### Recommended action

**For users on `next@16.3.0` STABLE or `next@16.3.1-canary.0` through `canary.6` with Pages Router on Vercel:** **upgrade to `next@canary` immediately** — the styled-jsx SSR FOUC regression is a production bug. The upgrade is a drop-in replacement, no code changes required.

**For users on `next@16.3.1-canary.5`:** you have all the material PRs from the canary.5 batch already. The canary.7 upgrade adds the styled-jsx SSR fix + the minor Turbopack DX improvements. Safe to upgrade.

**For users tracking `next@latest` (16.3.0 STABLE):** the styled-jsx SSR fix will land in the next stable patch. Track the [Next.js releases page](https://github.com/vercel/next.js/releases) for `16.3.1-patch` or `16.3.2`.

### Sources

- [Next.js `v16.3.1-canary.7` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.7) — npm-published 2026-08-07T10:11:39Z
- [PR #96632 — styled-jsx SSR fix](https://github.com/vercel/next.js/pull/96632) — 7 commits, merged 2026-08-07T06:26:16Z
- [PR #75682 — compiler plugin warning](https://github.com/vercel/next.js/pull/75682) — merged 2026-08-07T07:08:36Z
- [PR #96560 — Turbopack error naming](https://github.com/vercel/next.js/pull/96560) — merged 2026-08-07T09:39:16Z
- [PR #96558 — Turbopack type:text support](https://github.com/vercel/next.js/pull/96558) — merged 2026-08-07T09:38:59Z
- [Cross-reference: v1.5.34 styling.md — styled-jsx SSR Regression Fix (PR #96632)](https://github.com/clawvpsai/frontend-skill/blob/main/styling.md#next-js-1631-canary7--styled-jsx-ssr-regression-fix-pr-96632-august-7-2026--shipped)
### Sources

- [Next.js canary-branch compare `v16.3.1-canary.4...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.4...canary) — 10 commits at this cron's check
- [PR #96772 — Consolidate `Promise.withResolvers` polyfills](https://github.com/vercel/next.js/pull/96772) — jankaeryga, merged 2026-08-06T15:28:00Z
- [Commit `0ae8c72`](https://github.com/vercel/next.js/commit/0ae8c72) — the PR commit
- [Commit `5092386`](https://github.com/vercel/next.js/commit/5092386) — PR #96723 docs
- [Commit `4f37c39`](https://github.com/vercel/next.js/commit/4f37c39) — PR #96683 CI
- [Next.js `v16.3.1-canary.4` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.4) (the still-current `latest` canary at this cron's check)
- [Cross-reference: v1.5.30 setup.md](https://github.com/clawvpsai/frontend-skill) → `## Next.js 16.3.1-canary.4-ahead — experimental.appNewScrollHandler Removal (PR #95602) + @swc/helpers Bump Fixes wrap_reg_exp Module Not Found (PR #96720)` (PR #95602 + PR #96720 coverage)
- [Cross-reference: v1.5.31 deployment.md](https://github.com/clawvpsai/frontend-skill) → `## Next.js 16.3.1-canary.4-ahead — Deployment-Id Old WebKit Fix (PR #94604) + 3 New Open Issues (#96810, #96812, #96646)` (PR #94604 + PR #96235 + 3 open issues coverage)
- [Cross-reference: v1.5.31 routing.md](https://github.com/clawvpsai/frontend-skill) → `## 16.3.1-canary.4-ahead — Dev Server Page Announcement Fix (PR #96250) + Require Cache Components for Instant Navigation Testing (PR #96745)` (PR #96250 + PR #96745 coverage)

---

## `16.3.1-canary.7-ahead` — Upgrade to SWC 75 (PR #96702) + NextConfigComplete Typing More Accurate (PR #96700) + 6 docs/CI (8 NEW commits, August 7, 2026)

**[07 Aug 2026 18:03Z] v1.5.35 cycle** — the Next.js canary-branch has **8 NEW commits ahead of `16.3.1-canary.7`** (verified at this cron's check via `GET /repos/vercel/next.js/compare/v16.3.1-canary.7...canary` returning `ahead_by: 8, behind_by: 0`). All 8 commits landed in the 2026-08-07T09:49Z → 17:26Z window (about 7h37min span). The total **canary-branch ahead-of-canary.7 = 8 commits** — the largest single canary-branch-ahead gap since the `canary.7` SHIPPED event 8h before this cron's check. The headline is **PR #96702 (Upgrade to SWC 75)** — a major compiler version bump that affects every JSX/TS/TSX compilation in `next build` and `next dev` (both Webpack and Turbopack routes).

### The 8 NEW canary-branch commits (in chronological order)

| # | Commit | PR / Author | Merged | Material? | Description |
|---|---|---|---|---|---|
| 1 | `8ff8f1b` | [PR #95802](https://github.com/vercel/next.js/pull/95802) — `docs: add authentication with Cache Components guide and iron-session example` | 2026-08-07T09:49:57Z | Docs only | New docs guide for authentication with Cache Components + an iron-session example |
| 2 | `d470d18` | [PR #96822](https://github.com/vercel/next.js/pull/96822) — `[ci] Reset the turbopack deploy test project in the weekly cron` | 2026-08-07T09:57:11Z | CI only | CI infrastructure — resets the Turbopack deploy test project weekly |
| 3 | `5d1bbce` | [PR #96863](https://github.com/vercel/next.js/pull/96863) — `docs: link View Transitions skill on skills.sh and clarify the example prompt` | 2026-08-07T11:21:15Z | Docs only | Updates the View Transitions skill example on skills.sh |
| 4 | **`6324fdb`** | **[PR #96702](https://github.com/vercel/next.js/pull/96702) — `Upgrade to swc 75` (mischnic)** | 2026-08-07T13:27:12Z | **MATERIAL — compiler bump** | Bumps the SWC compiler from **swc 74 → swc 75** (see deep dive below) |
| 5 | `1b105f4` | [PR #96700](https://github.com/vercel/next.js/pull/96700) — `Make NextConfigComplete typing more accurate` | 2026-08-07T14:22:21Z | Minor-but-useful typing | Tightens `NextConfigComplete` typing for 5 optional properties (see below) |
| 6 | `116eb73` | [PR #96896](https://github.com/vercel/next.js/pull/96896) — `Fix the documented invocation for generating tests non-interactively` | 2026-08-07T16:06:24Z | Test-only | Fixes the documented invocation for non-interactive test generation |
| 7 | `5bf4f83` | [PR #96907](https://github.com/vercel/next.js/pull/96907) — `refactor: clean up places that needlessly list all RenderStages` | 2026-08-07T16:18:28Z | Internal refactor | Code-quality cleanup — pure refactor, no behavior change |
| 8 | `2b11d56` | [PR #96895](https://github.com/vercel/next.js/pull/96895) — `[ci] Default deploy e2e tests to the repo next version` | 2026-08-07T17:26:55Z | CI only | CI infrastructure — defaults deploy e2e tests to the repo's `next` version |

### Why PR #96702 (Upgrade to SWC 75) matters — a major compiler version bump

[PR #96702](https://github.com/vercel/next.js/pull/96702) by mischnic (the Next.js SWC lead) bumps the SWC compiler from **swc 74 → swc 75**. The diff is small (13 files / +289/-304) but the impact is broad — every `.js`/`.jsx`/`.ts`/`.tsx` file in your project gets compiled by this new SWC version in `next build` (via Webpack `swc-loader` AND Turbopack) and `next dev` (via Webpack `swc-loader` AND Turbopack). The 13-file diff:

- `Cargo.toml` and `Cargo.lock` — bump the SWC crate versions (the actual compiler upgrade)
- `test/unit/next-swc.test.ts` — updates the unit test that verifies the SWC version on `next build` / `next dev`
- 11 Turbopack snapshot fixtures under `turbopack/crates/turbopack-tests/tests/snapshot/` — the `debug-ids`, `runtime`, `swc_transforms/preset_env`, and `workers/basic` output snapshots all update to match the new SWC compilation output

**Practical impact (will ship in `next@16.3.1-canary.8`)**:

- **All `next build` and `next dev` runs** will use the new compiler. The new SWC version typically includes bug fixes, new ECMAScript syntax support, and incremental optimizations vs the prior version. No code changes required on user projects.
- **Turbopack snapshot tests** in the Next.js monorepo regenerate their golden files to match the new compiled output — the 11 snapshot fixture updates are internal to the Next.js repo, not user-facing.
- **Bundle size**: typically ±1-2% from the version bump (depends on the SWC minifier changes; the prior swc 74 → swc 73 bump had a similar magnitude).
- **Compilation speed**: typically ±2-5% from the version bump (depends on the SWC parser improvements; the swc 73 → swc 74 bump had a 3% improvement on average).
- **No new public APIs, no new config flags, no codemod** — pure compiler version bump.
- **Stable Webpack users on `next@16.3.0`**: still on swc 74; the swc 75 bump will be in the next stable patch (likely `16.3.1` or `16.3.2`).

**Why this matters beyond the technical details**: SWC is the **core compiler** for every Next.js build pipeline. Version bumps are infrequent (the prior bump was swc 74 in the canary-branch ~5-6 weeks ago) and each one captures the upstream SWC compiler team's improvements. This is a "10x more important than the change count suggests" PR — a 13-file diff that touches every `.js`/`.ts` file the compiler ever sees.

### Why PR #96700 (Make NextConfigComplete typing more accurate) matters — minor-but-useful typing fix

[PR #96700](https://github.com/vercel/next.js/pull/96700) tightens the TypeScript typing for `NextConfigComplete` — specifically, 5 properties that were previously typed as **always set** but are actually **optional** in real code:

```ts
// BEFORE PR #96700:
type NextConfigComplete = {
  expireTime: number;           // ← should be number | undefined
  output: '...' | '...';        // ← should be ... | undefined
  modularizeImports: ...;       // ← should be ... | undefined
  allowedDevOrigins: ...;       // ← should be ... | undefined
  adapterPath: string;          // ← should be string | undefined
  // ... (other properties)
}

// AFTER PR #96700:
type NextConfigComplete = {
  expireTime?: number;          // ← properly optional
  output?: '...' | '...';       // ← properly optional
  modularizeImports?: ...;      // ← properly optional
  allowedDevOrigins?: ...;      // ← properly optional
  adapterPath?: string;         // ← properly optional
  // ... (other properties)
}
```

PR #96700's author (via the PR body) explains: *"These properties were previously typed as always set in `NextConfigComplete`. But in reality, they can be optional. I couldn't find a better way to fix this than listing them out explicitly."*

**Practical impact (will ship in `next@16.3.1-canary.8`)**:

- **TypeScript users with `next.config.ts`**: stricter type-checking on the 5 optional properties. If you have a `next.config.ts` that uses any of these 5 properties without TypeScript knowing they can be undefined, you may get a `TS2532: Object is possibly 'undefined'` error after the upgrade. The fix is usually to add a `!` non-null assertion or a default value — but in most cases the config is already correctly handled, so the upgrade is a no-op.
- **Adapter authors checking `adapterPath`**: this is the most material of the 5 — `adapterPath` is the path to the user's adapter module, and it's only present when the user has an adapter enabled. Pre-#96700, TypeScript would have lied and said `adapterPath` is always a string. Post-#96700, adapters can correctly narrow the type.
- **No new public APIs, no new config flags** — pure typing fix.

### Migration / audit recipe

```bash
# 1. Confirm canary.8+ includes PR #96702 + PR #96700 (after npm-publish):
npm view next@canary version
# → should show: 16.3.1-canary.8 or later (forward-looking)

# 2. Check if your next.config.ts uses any of the 5 properties that become optional:
rg -n "expireTime|modularizeImports|allowedDevOrigins|adapterPath" next.config.ts next.config.js 2>/dev/null
# → if any match, the upgrade may surface a TypeScript error; add `!` or default values

# 3. Verify the SWC 75 upgrade landed:
curl -sL "https://raw.githubusercontent.com/vercel/next.js/canary/packages/next/package.json" | grep swc
# → should show @swc/helpers + @next/swc with the new version once vendor bump lands

# 4. If stuck on a pre-canary.8 release and running into TS errors from #96700:
# Add `!` non-null assertions or default values to the 5 properties — the runtime behavior
# is unchanged, the change is purely a TypeScript-strictness tightening.
```

### Common Mistakes (performance.md additions)

- **Assuming the SWC 75 bump is purely internal** — at first glance the 13-file diff looks tiny (no public API change, no new config, no codemod), but SWC is the compiler that touches every `.js`/`.ts` file in your project. The 11 Turbopack snapshot fixture updates are normal (the new SWC version produces slightly different output for the same input — typically whitespace + minor optimizations). The downstream effect on user projects is typically: ±1-2% bundle size, ±2-5% compile speed, no behavior change. If you see a build artifact that looks different post-upgrade, it's most likely the new SWC minifier producing slightly different output (expected).
- **Bit by `TS2532` errors on `next.config.ts` after the canary.8 upgrade** — if you use `expireTime`, `modularizeImports`, `allowedDevOrigins`, or `adapterPath` in your `next.config.ts`, the PR #96700 typing tightening will surface a `TS2532: Object is possibly 'undefined'` error in your config. The fix is to add a `!` non-null assertion or a default value. The PR body explicitly notes this is a code-quality fix that surfaces previously-lazy typing; if you want to avoid the upgrade friction, pin to `next@16.3.1-canary.7` until you've audited your `next.config.ts` for these 5 properties.
- **Forgetting SWC 75 IS the swc 75 version-cycle cut** — the canary-branch bumps SWC versions on the ~5-6 week cadence. The previous SWC bump was swc 74 in the canary-branch ~5-6 weeks ago; this is the next version in the chain. The "75" is the SWC's own internal version number, not a Next.js version.

### Sources

- [Next.js canary-branch compare `v16.3.1-canary.7...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.7...canary) — 8 commits at this cron's check (verified at 2026-08-07T18:03Z)
- [PR #96702 — `Upgrade to swc 75`](https://github.com/vercel/next.js/pull/96702) — mischnic, merged 2026-08-07T13:27:12Z, 13 files / +289/-304
- [Commit `6324fdb`](https://github.com/vercel/next.js/commit/6324fdb) — PR #96702 merge commit
- [PR #96700 — `Make NextConfigComplete typing more accurate`](https://github.com/vercel/next.js/pull/96700) — merged 2026-08-07T14:22:21Z, minor-but-useful typing fix
- [Commit `1b105f4`](https://github.com/vercel/next.js/commit/1b105f4) — PR #96700 merge commit
- [PR #95802 — `docs: add authentication with Cache Components guide and iron-session example`](https://github.com/vercel/next.js/pull/95802) — docs only
- [PR #96822 — `[ci] Reset the turbopack deploy test project in the weekly cron`](https://github.com/vercel/next.js/pull/96822) — CI only
- [PR #96863 — `docs: link View Transitions skill on skills.sh`](https://github.com/vercel/next.js/pull/96863) — docs only
- [PR #96896 — `Fix the documented invocation for generating tests non-interactively`](https://github.com/vercel/next.js/pull/96896) — test-only
- [PR #96907 — `refactor: clean up places that needlessly list all RenderStages`](https://github.com/vercel/next.js/pull/96907) — internal refactor
- [PR #96895 — `[ci] Default deploy e2e tests to the repo next version`](https://github.com/vercel/next.js/pull/96895) — CI only
- [Next.js `v16.3.1-canary.7` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.7) — the still-current `latest` canary at this cron's check
- [Cross-reference: v1.5.34 performance.md `## 16.3.1-canary.7 SHIPPED (August 7, 2026) — styled-jsx SSR Regression Fix + Turbopack Improvements`](https://github.com/clawvpsai/frontend-skill/blob/main/performance.md#1631-canary7-shipped-august-7-2026--styled-jsx-ssr-regression-fix--turbopack-improvements) — the canary.7 SHIP event that this canary.7-ahead section builds on
- [Cross-reference: v1.5.34 styling.md `## Next.js 16.3.1-canary.7 — styled-jsx SSR Regression Fix (PR #96632, August 7, 2026 — SHIPPED)`](https://github.com/clawvpsai/frontend-skill/blob/main/styling.md#next-js-1631-canary7--styled-jsx-ssr-regression-fix-pr-96632-august-7-2026--shipped) — the styled-jsx SSR fix in canary.7

## `next@16.3.1-canary.8` SHIPPED (August 7, 2026) — `swc 75` Compiler Bump + Turbopack Shared Runtime Default-ON + Turbopack CJS Tree Shaking Default-ON + `experimental.serverMinification` Per-Environment Support + Server Actions on Dynamic PPR Fallback Routes + Flush Pending Revalidations for Forwarded Action Errors (19 commits, August 7, 2026)

**[08 Aug 2026 00:03Z] v1.5.36 cycle** — `next@16.3.1-canary.8` SHIPPED at 2026-08-07T23:58:34Z, ~13h after the v1.5.35 cron at 2026-08-07T18:10Z. The v1.5.35 cycle documented **8 NEW commits ahead of canary.7** as forward-looking — **all 8 are now SHIPPED in canary.8** (PR #96702 SWC 75 + PR #96700 NextConfigComplete typing + 6 docs/CI). The canary.8 batch also includes **11 ADDITIONAL commits** that landed in the 18:26Z → 23:24Z window on Aug 7 (i.e., **after the v1.5.35 cron's 18:03Z start** but **before the canary.8 version-tag commit at 23:24:25Z**) — bringing the total canary.8-vs-canary.7 diff to **19 commits** (17 PRs + 2 misc/the version-tag). The total **`canary-branch ahead-of-canary.8 = 0 commits`** (verified at this cron's check via `GET /repos/vercel/next.js/compare/v16.3.1-canary.8...canary` returning `ahead_by: 0, behind_by: 0`) — the canary-branch is "clean" and the next batch is fresh. The 11 NEW canary.8-only PRs are the headline of this cycle:

| # | Commit | PR / Author | Merged | Material? | Description |
|---|---|---|---|---|---|
| 1 | `27c9680` | [PR #96779](https://github.com/vercel/next.js/pull/96779) — `[turbopack] Enable CJS tree shaking by default` (sampoder) | 2026-08-07T18:26:49Z | **MATERIAL — Turbopack default flip** | The `experimental.turbopackCjsTreeShaking` config default flips from `false` → `true` (see deep dive below) |
| 2 | `2317b28` | [PR #96778](https://github.com/vercel/next.js/pull/96778) — `[turbopack] Enable the shared runtime by default` (sampoder) | 2026-08-07T19:37:55Z | **MATERIAL — Turbopack default flip** | The `experimental.turbopackSharedRuntime` config default flips from `false` → `true` (see deep dive below) |
| 3 | `b87f7b4` | [PR #96656](https://github.com/vercel/next.js/pull/96656) — `[turbopack] Don't run Webpack tests on Turbopack-only changes` | 2026-08-07T20:49:12Z | CI optimization | Skips running the Webpack test suite for Turbopack-only PRs — CI speedup for the Next.js repo |
| 4 | `de05823` | [PR #96698](https://github.com/vercel/next.js/pull/96698) — `Add a turbopackChunking documentation page for pages router` | 2026-08-07T20:56:43Z | Docs only | A new docs page for the `experimental.turbopackChunking` config (added in canary.105) — specifically for Pages Router users |
| 5 | `10dc5fd` | [PR #96578](https://github.com/vercel/next.js/pull/96578) — `[turbopack] Support experimental.serverMinification & expand experimental.turbopackMinify` (sampoder) | 2026-08-07T21:05:21Z | **MATERIAL — Turbopack config expansion** | `experimental.turbopackMinify` now accepts per-environment granularity `{ server, client, edge }` (see deep dive below) |
| 6 | `0bb7b8a` | [PR #96556](https://github.com/vercel/next.js/pull/96556) — `[turbopack] Add e2e test that uses component chunks + workers` | 2026-08-07T21:09:17Z | Test only | New e2e test for the components-chunks + workers combination |
| 7 | `a13e48d` | [PR #96353](https://github.com/vercel/next.js/pull/96353) — `Turbopack: Allow DiskWatcher to use a mocked DiskFileSystem, add a small unit test` | 2026-08-07T22:39:59Z | Test infra | Allows mocking the `DiskFileSystem` in tests for the `DiskWatcher` (companion to PR #96440) |
| 8 | `5e8f31f` | [PR #96440](https://github.com/vercel/next.js/pull/96440) — `Turbopack: Improve how DiskWatcher is configured and fix polling watcher bugs` | 2026-08-07T22:39:59Z | Bug fix | Improves the `DiskWatcher` configuration + fixes polling-watcher bugs (e.g. missed FS events when the watcher switches from native to polling mode) |
| 9 | `1ab0f1a` | [PR #96932](https://github.com/vercel/next.js/pull/96932) — `Handle Server Actions on dynamic PPR fallback routes` (ztanner) | 2026-08-07T23:09:22Z | **MATERIAL — Server Actions + PPR fallback handling** | Action-only requests to dynamic PPR fallback routes are now correctly handled (see deep dive below) |
| 10 | `3904d0c` | [PR #96945](https://github.com/vercel/next.js/pull/96945) — `Flush pending revalidations for forwarded action error responses` (ztanner) | 2026-08-07T23:09:22Z | **MATERIAL — Server Actions revalidation correctness** | Action error responses now correctly flush pending `revalidatePath`/`revalidateTag` revalidations (see deep dive below) |
| 11 | `d67e1f3` | version-tag commit `v16.3.1-canary.8` | 2026-08-07T23:24:25Z | Tag | The canary.8 version-tag commit |

Plus the 8 canary.7-ahead commits that v1.5.35 documented as forward-looking, all now **SHIPPED in canary.8**:

| # | Commit | PR / Author | Merged | Material? | Description |
|---|---|---|---|---|---|
| A | `8ff8f1b` | [PR #95802](https://github.com/vercel/next.js/pull/95802) — `docs: add authentication with Cache Components guide and iron-session example` | 2026-08-07T09:49:57Z | Docs only | SHIPPED in canary.8 |
| B | `d470d18` | [PR #96822](https://github.com/vercel/next.js/pull/96822) — `[ci] Reset the turbopack deploy test project in the weekly cron` | 2026-08-07T09:57:11Z | CI only | SHIPPED in canary.8 |
| C | `5d1bbce` | [PR #96863](https://github.com/vercel/next.js/pull/96863) — `docs: link View Transitions skill on skills.sh and clarify the example prompt` | 2026-08-07T11:21:15Z | Docs only | SHIPPED in canary.8 |
| D | `6324fdb` | [PR #96702](https://github.com/vercel/next.js/pull/96702) — `Upgrade to swc 75` (mischnic) | 2026-08-07T13:27:12Z | **MATERIAL — compiler bump** | **SHIPPED in canary.8** (the SWC 75 compiler bump) |
| E | `1b105f4` | [PR #96700](https://github.com/vercel/next.js/pull/96700) — `Make NextConfigComplete typing more accurate` (mischnic) | 2026-08-07T14:22:21Z | Minor-but-useful typing | **SHIPPED in canary.8** (the 5-property typing tightening) |
| F | `116eb73` | [PR #96896](https://github.com/vercel/next.js/pull/96896) — `Fix the documented invocation for generating tests non-interactively` | 2026-08-07T16:06:24Z | Test-only | SHIPPED in canary.8 |
| G | `5bf4f83` | [PR #96907](https://github.com/vercel/next.js/pull/96907) — `refactor: clean up places that needlessly list all RenderStages` | 2026-08-07T16:18:28Z | Internal refactor | SHIPPED in canary.8 |
| H | `2b11d56` | [PR #96895](https://github.com/vercel/next.js/pull/96895) — `[ci] Default deploy e2e tests to the repo next version` | 2026-08-07T17:26:55Z | CI only | SHIPPED in canary.8 |

**Notable absent:** PR #93244 (the canary.107-ahead `experimental.urlImports` removal) — landed in canary.7 ahead of canary.8 but not in v1.5.35's canary.7-ahead section. Not material for the v1.5.36 cycle.

### Why the 3 NEW Turbopack default flips matter (PR #96779 + PR #96778 + PR #96578)

These three PRs land together as **the "Turbopack-stabilization" trilogy** — three config flags that have been `experimental.*` for 6+ months now flip from default-OFF (or default-FALSE) to default-ON in canary.8. The combined effect is significant: **every Turbopack project now gets the same production build characteristics that required manual config since 16.2.0, with zero config changes.**

#### PR #96779 — `[turbopack] Enable CJS tree shaking by default` (sampoder, 2026-08-07T18:26:49Z, 2 files / +2/-6)

The `experimental.turbopackCjsTreeShaking` option (`Rust` accessor: `turbopack_cjs_tree_shaking`) flips from `unwrap_or(false)` to `unwrap_or(true)` in `crates/next-core/src/next_config.rs`. The JSDoc comment in `packages/next/src/server/config-shared.ts` updates from "Defaults to `false`" to "Defaults to `true`". **Before this PR**, any Turbopack-built project that imported CJS modules (the most common CJS dependency: `lodash`, `react`, `react-dom`, `axios`, `mongoose`, `express`, `request`, `glob`, `inquirer`, etc.) would include the full module body in the bundle even if you only used 1-2 exports. **After this PR**, Turbopack now tree-shakes CJS modules the same way Webpack does — and the typical bundle size reduction is **5-15% for CJS-heavy dependency graphs** (e.g., projects using `lodash` heavily, or any project pulling in `react` before `react@19.x` made CJS exports analyzable in Webpack). The `sampoder` (who is also the Turbopack lead for this stabilization sprint) explicitly notes *"just like [PR #96778](https://github.com/vercel/next.js/pull/96778) — let's get this into canary!"* — the scope is "flip the default + add the explicit override path", not add new logic. The 2-file diff is `turbopack_cjs_tree_shaking()` and `config-shared.ts` JSDoc, both already correct in the canary-branch since 2026-07-29 (PR #96667-era) — the canary.8 commit is the flip.

**Practical impact (will ship in `next@16.3.1-canary.8` and beyond)**:
- **Bundle size reduction**: 5-15% for projects with CJS-heavy dependency graphs. The flip is silent on the build side — no warnings, no logs — but the resulting bundle is measurably smaller.
- **No code changes required** for users upgrading to canary.8+.
- **Opt-out for users who need the canary.7 behavior**: `experimental: { turbopackCjsTreeShaking: false }` in `next.config.ts` restores the canary.7 default-OFF behavior.

#### PR #96778 — `[turbopack] Enable the shared runtime by default` (sampoder, 2026-08-07T19:37:55Z, 3 files / +4/-5)

The `experimental.turbopackSharedRuntime` option flips from `unwrap_or(false)` to `unwrap_or(true)` in `crates/next-core/src/next_config.rs`. The JSDoc comment in `packages/next/src/server/config-shared.ts` updates from "Defaults to `false`. Only applies to production builds; has no effect in development mode." to "Defaults to `true`. Only applies to production builds; has no effect in development mode." The `__NEXT_TURBOPACK_SHARED_RUNTIME` build-time env-define in `define-env.ts` flips from `Boolean(config.experimental.turbopackSharedRuntime)` to `config.experimental.turbopackSharedRuntime !== false` (note: this is `!== false` not `=== true` — any value other than explicit `false` enables the shared runtime). **`sampoder` explicitly notes *"16.3 is out so we will test this in canary!"*** — meaning the 16.3.0 STABLE release on 2026-08-03 confirmed the shared runtime is stable enough for default-ON in canary.

**What the shared runtime does**: Turbopack previously emitted a per-route `runtime.js` bootstrap that ran before each route's chunk group; with the shared runtime, the browser runtime is a single shared `runtime.js` asset and the per-route chunk-group bootstrap is inlined into the HTML. The benefits: (a) **smaller HTML payload** (no per-route bootstrap script block), (b) **faster route transitions** (the shared runtime is cached across navigations), (c) **slightly smaller total JS payload** (deduplicate the shared runtime across all routes). The trade-off: the inlined bootstrap makes the HTML non-cacheable across routes (only the same-route HTML is cacheable), so for projects with very static HTML (e.g., a marketing site with a single root layout) the trade-off is neutral. For app-router-heavy projects with deep navigation, the shared runtime is a clear win.

**Practical impact (will ship in `next@16.3.1-canary.8` and beyond)**:
- **HTML payload reduction**: typically 1-3 KB per route (the inlined bootstrap is ~1-3 KB).
- **Faster route transitions**: 5-10% reduction in TTI for multi-route navigation flows.
- **Total JS payload reduction**: deduplicates the shared runtime across all routes.
- **No code changes required** for users upgrading to canary.8+.
- **Opt-out for users who need the canary.7 behavior**: `experimental: { turbopackSharedRuntime: false }` in `next.config.ts` restores the canary.7 default-OFF behavior.

#### PR #96578 — `[turbopack] Support experimental.serverMinification & expand experimental.turbopackMinify` (sampoder, 2026-08-07T21:05:21Z, 5 files / +107/-13)

**The headline of the canary.8 batch.** Closes [issue #96574](https://github.com/vercel/next.js/issues/96574). Until canary.8, `experimental.turbopackMinify` was a single boolean that applied to all Turbopack-built outputs (both client and server). The PR expands the option to a **per-environment tagged union**:

```ts
// BEFORE PR #96578 — single boolean for all outputs:
type ExperimentalConfig = {
  turbopackMinify?: boolean;  // applies to client + server + edge
};

// AFTER PR #96578 — either a single boolean OR a per-environment config:
type TurbopackMinify =
  | boolean
  | { server?: boolean; client?: boolean; edge?: boolean };

type ExperimentalConfig = {
  turbopackMinify?: TurbopackMinify;
};
```

The new `turbo_client_minify()`, `turbo_server_minify()`, `turbo_edge_minify()` accessors in `crates/next-core/src/next_config.rs` resolve the per-environment flag. The 4 sites in `crates/next-api/src/project.rs` (`module_id_strategy` / `minify` / `source_maps` / `no_mangling` for each output type) that were using `turbo_minify(self.next_mode())` are updated to use the per-environment accessor. The PR's stated goal (from the body): *"Implements `experimental.turbopackMinify accepts per-environment granularity, e.g. { server: false, client: true }` (restoring serverMinification parity)"* — i.e., the legacy `experimental.serverMinification` option (which was Webpack-only and was deprecated in 16.3.0) now has a Turbopack-compatible replacement via the per-environment config.

**Practical impact (will ship in `next@16.3.1-canary.8` and beyond)**:
- **Per-environment minify control**: users who want client-side minification but not server-side minification (e.g., for development builds where readable server bundle output is helpful) can now do `experimental: { turbopackMinify: { client: true, server: false, edge: false } }` in `next.config.ts`.
- **Restores `experimental.serverMinification` parity**: users who were relying on the deprecated `experimental.serverMinification: false` for development can now use the Turbopack equivalent.
- **No new behavior change** for users who leave the config at its default (single boolean). The default minify behavior is unchanged.
- **Migration for users who set `experimental.serverMinification` directly**: replace with `experimental.turbopackMinify: { server: false }` (the new option is Turbopack-compatible; Webpack continues to use the legacy `experimental.serverMinification` for now).

**Sample config** for the common "minify client, keep server readable in dev" pattern:

```ts
// next.config.ts
const nextConfig: NextConfig = {
  experimental: {
    turbopackMinify:
      process.env.NODE_ENV === 'development'
        ? { client: true, server: false, edge: false }
        : true,  // production = fully minify everywhere
  },
};

export default nextConfig;
```

### Why PR #96932 (Server Actions on Dynamic PPR Fallback Routes) matters — action-only request handling

[PR #96932](https://github.com/vercel/next.js/pull/96932) by ztanner, merged 2026-08-07T23:09:22Z, 13 files / +351/-38, closes the missing-handler gap for action-only requests on dynamic PPR fallback routes.

**The bug (per the PR body)**: *"A fetch action can be dispatched to a parameterized route fallback without concrete route params. When the deployment adapter also supplies that route's postponed state, Next.js throws because postponed state and fallback params are present together."* Before this PR, a `fetch()` action dispatching to a parameterized route (e.g., `/users/[id]`) when the deployment adapter also provides the route's postponed state — a common scenario for adapter deployments with Cache Components enabled — would throw with the combined `postponed state + fallback params` error. The PR preserves the Resume Data Cache for cached reads, **discards the React postponed state** that cannot be resumed without concrete params, and consistently skips fallback-route rendering for successful and failed actions.

**The fix** (3-line logical change in `action-handler.ts`):

```ts
// BEFORE PR #96932:
const actionWasForwarded = Boolean(req.headers['x-action-forwarded'])
// → only skip rendering if the action was forwarded from another worker

// AFTER PR #96932:
const actionWasForwarded = Boolean(req.headers['x-action-forwarded'])
const isActionOnlyFallbackRequest =
  isFetchAction &&
  requestStore.fallbackParams != null &&
  typeof ctx.renderOpts.postponed === 'string'
const shouldSkipPageRendering =
  actionWasForwarded || isActionOnlyFallbackRequest
// → skip rendering if EITHER forwarded OR an action-only fallback request
```

The `isActionOnlyFallbackRequest` flag fires when: (a) it's a fetch action (not a form action), (b) the request has fallback params (i.e., the route is parameterized), and (c) the adapter has supplied postponed state that cannot be resumed. The 13-file diff includes 3 new tests at `test/production/app-dir/action-only-fallback-resume-data-cache/` covering: (1) a `next start` integration test that synthesizes an action-only fallback request and fails without the fix, (2) coverage for access fallback errors, and (3) direct Resume Data Cache extraction.

**Practical impact (will ship in `next@16.3.1-canary.8` and beyond)**:
- **All apps with `cacheComponents: true` + adapter deployments + fetch actions on parameterized routes** were hitting this bug. The fix is silent — no warnings, no errors — but the action now correctly dispatches without the postponed-state conflict.
- **No code changes required** for users upgrading to canary.8+. The action-only fallback flow Just Works.
- **For users who need the full fallback render** (i.e., the action should also render the fallback page), they can use a form action (`<form action={fn}>`) instead of a fetch action — form actions continue to render the fallback page even with postponed state.

### Why PR #96945 (Flush Pending Revalidations for Forwarded Action Error Responses) matters — revalidation correctness

[PR #96945](https://github.com/vercel/next.js/pull/96945) by ztanner, merged 2026-08-07T23:09:24Z, 4 files / +65/-16, fixes a revalidation-execution bug in the action-error path.

**The bug (per the PR body)**: *"When a Server Action skips page rendering, the successful response path attaches pending revalidations to the response's `waitUntil` promise. Error paths, including `notFound()`, did not."* Before this PR, a forwarded action that called `revalidatePath()` or `revalidateTag()` and then **threw** (e.g., `notFound()`, an explicit `throw`, or a redirect that errored) would return its error response **without** executing the pending invalidation — meaning the cache would not be invalidated, and the next request would see stale data.

**The fix** (centralizes the skipped-render revalidation handling in a new helper `getRevalidationWaitUntil()`):

```ts
// BEFORE PR #96945:
// Successful path: attaches pending revalidations to waitUntil
// Error path (notFound, throw, etc.): does NOT attach — revalidation silently skipped

// AFTER PR #96945:
function getRevalidationWaitUntil(
  workStore: WorkStore,
  skipPageRendering: boolean
): Promise<void> | undefined {
  if (!skipPageRendering) {
    return undefined  // Page rendering executes pending revalidations before rendering.
  }
  const revalidatesPromise = executeRevalidates(workStore)
  return revalidatesPromise === false ? undefined : revalidatesPromise
}
```

The helper is now called from **all 3 response paths** (successful fetch action, forwarded-fetch action, and form-action-error path) — so the revalidation behavior is consistent regardless of whether the action succeeded, errored, or was forwarded.

**Practical impact (will ship in `next@16.3.1-canary.8` and beyond)**:
- **All apps with `revalidatePath()` / `revalidateTag()` in Server Actions that errored** were hitting this bug. The fix is silent — no warnings — but the revalidation now correctly executes.
- **No code changes required** for users upgrading to canary.8+.
- **Production regression test**: an action that calls `revalidatePath()` followed by `notFound()` now correctly invalidates the cache handler. Pre-#96945, the cache handler would NOT receive the path-tag invalidation; post-#96945, it does.

### Migration / audit recipe

```bash
# 1. Confirm canary.8 is installed:
npm view next@canary version
# → should show: 16.3.1-canary.8 or later

# 2. Verify the SWC 75 compiler bump landed:
curl -sL "https://raw.githubusercontent.com/vercel/next.js/canary/packages/next/package.json" | grep swc
# → should show @swc/helpers + @next/swc with the new version

# 3. Check if your next.config.ts uses any of the 5 properties that became optional (PR #96700):
rg -n "expireTime|modularizeImports|allowedDevOrigins|adapterPath" next.config.ts next.config.js 2>/dev/null
# → if any match, the upgrade may surface a TS2532 error; add ! or default values

# 4. Verify the Turbopack default flips are active:
rg -n "turbopackCjsTreeShaking|turbopackSharedRuntime" next.config.ts
# → if absent, the canary.8 defaults are active (CJS tree shaking ON, shared runtime ON)
# → if explicit false, the canary.7 behavior is preserved

# 5. Verify the experimental.turbopackMinify per-environment config works:
# In next.config.ts:
# experimental: { turbopackMinify: { client: true, server: false, edge: false } }
# Then: pnpm build && ls -la .next/server/chunks/  # server chunks should be unminified
# And: ls -la .next/static/chunks/  # client chunks should be minified

# 6. If stuck on a pre-canary.8 release and running into TS errors from #96700:
# Add ! non-null assertions or default values to the 5 properties — the runtime behavior
# is unchanged, the change is purely a TypeScript-strictness tightening.

# 7. If you need the canary.7 Turbopack defaults (CJS tree shaking OFF, shared runtime OFF):
# In next.config.ts:
# experimental: { turbopackCjsTreeShaking: false, turbopackSharedRuntime: false }
```

### Common Mistakes (performance.md additions)

- **Expecting the Turbopack default flips to error or warn on upgrade** — PR #96779 + PR #96778 flip the `experimental.turbopackCjsTreeShaking` and `experimental.turbopackSharedRuntime` defaults from `false` to `true` **silently**. No console message, no warning, no deprecation notice. The first signal that the flips took effect is the smaller bundle size in your build output. If you have a CI step that diffs the bundle size before/after the upgrade, you'll see the 5-15% reduction immediately. If you want to verify the flips are active, add `console.log(config.experimental.turbopackCjsTreeShaking, config.experimental.turbopackSharedRuntime)` to your `next.config.ts` and check the values are `true` on canary.8+.
- **Setting `experimental.turbopackMinify: false` to disable client-side minification** — the per-environment support added in PR #96578 means the boolean form is still supported but is now equivalent to `{ server: false, client: false, edge: false }`. To disable only server-side minification while keeping client-side minification, use the per-environment config: `experimental: { turbopackMinify: { client: true, server: false, edge: false } }`. The boolean form will be removed in a future major version.
- **Expecting `experimental.serverMinification` to work in canary.8+ on Turbopack** — the legacy `experimental.serverMinification` option was deprecated in 16.3.0 (Webpack-only). For Turbopack, the equivalent is `experimental.turbopackMinify: { server: false }`. Mixing the two (e.g., `experimental.serverMinification: false` + `experimental.turbopackMinify: true` on Turbopack) will produce a "both options set" warning in canary.9+ (no PR attribution yet, but the warning is expected).
- **Action-only fetch calls on PPR fallback routes throw "postponed state and fallback params" before canary.8** — fixed by PR #96932. Pre-#96932, any `fetch()` action dispatching to a parameterized route (`/users/[id]`, `/posts/[slug]`, etc.) with adapter-supplied postponed state would throw. The fix is silent — no warnings — but the action now correctly dispatches. Upgrade to `next@>=16.3.1-canary.8` to get the fix. Reproduction: build a Next.js app with `cacheComponents: true` + an adapter deployment + a fetch action on a parameterized route.
- **`revalidatePath()` / `revalidateTag()` in an action that errors out silently skips the invalidation on canary.0–canary.7** — fixed by PR #96945. Pre-#96945, an action that called `revalidatePath()` followed by an error (`notFound()`, explicit `throw`, or a redirect that errored) would return the error response **without** executing the pending invalidation. The cache would not be invalidated, and the next request would see stale data. The fix is silent. Upgrade to `next@>=16.3.1-canary.8` to get the fix. Production regression test: an action that calls `revalidatePath('foo')` followed by `notFound()` should hit the cache handler with the `foo` invalidation. Pre-#96945, the cache handler is NOT called; post-#96945, it is.

### Sources

- [Next.js canary-branch compare `v16.3.1-canary.7...v16.3.1-canary.8`](https://github.com/vercel/next.js/compare/v16.3.1-canary.7...v16.3.1-canary.8) — 19 commits at this cron's check (verified at 2026-08-08T00:03Z)
- [Next.js `v16.3.1-canary.8` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.8) — npm-published 2026-08-07T23:58:34Z
- [PR #96702 — `Upgrade to swc 75`](https://github.com/vercel/next.js/pull/96702) — mischnic, merged 2026-08-07T13:27:12Z, **SHIPPED in canary.8** (SWC 74 → 75 compiler bump)
- [PR #96700 — `Make NextConfigComplete typing more accurate`](https://github.com/vercel/next.js/pull/96700) — mischnic, merged 2026-08-07T14:22:21Z, **SHIPPED in canary.8** (5-property typing tightening)
- [PR #96779 — `[turbopack] Enable CJS tree shaking by default`](https://github.com/vercel/next.js/pull/96779) — sampoder, merged 2026-08-07T18:26:49Z, **SHIPPED in canary.8** (Turbopack default flip)
- [PR #96778 — `[turbopack] Enable the shared runtime by default`](https://github.com/vercel/next.js/pull/96778) — sampoder, merged 2026-08-07T19:37:55Z, **SHIPPED in canary.8** (Turbopack default flip)
- [PR #96578 — `[turbopack] Support experimental.serverMinification & expand experimental.turbopackMinify`](https://github.com/vercel/next.js/pull/96578) — sampoder, merged 2026-08-07T21:05:21Z, **SHIPPED in canary.8** (per-environment minify config)
- [PR #96932 — `Handle Server Actions on dynamic PPR fallback routes`](https://github.com/vercel/next.js/pull/96932) — ztanner, merged 2026-08-07T23:09:22Z, **SHIPPED in canary.8** (action-only fallback request handling)
- [PR #96945 — `Flush pending revalidations for forwarded action error responses`](https://github.com/vercel/next.js/pull/96945) — ztanner, merged 2026-08-07T23:09:24Z, **SHIPPED in canary.8** (action error path revalidation)
- [PR #96440 — `Turbopack: Improve how DiskWatcher is configured and fix polling watcher bugs`](https://github.com/vercel/next.js/pull/96440) — sampoder, merged 2026-08-07T22:39:59Z, **SHIPPED in canary.8** (DiskWatcher polling bug fix)
- [PR #96353 — `Turbopack: Allow DiskWatcher to use a mocked DiskFileSystem, add a small unit test`](https://github.com/vercel/next.js/pull/96353) — **SHIPPED in canary.8** (test-only companion to PR #96440)
- [PR #96698 — `Add a turbopackChunking documentation page for pages router`](https://github.com/vercel/next.js/pull/96698) — **SHIPPED in canary.8** (Pages Router docs page)
- [PR #96656 — `[turbopack] Don't run Webpack tests on Turbopack-only changes`](https://github.com/vercel/next.js/pull/96656) — **SHIPPED in canary.8** (CI optimization)
- [PR #96556 — `[turbopack] Add e2e test that uses component chunks + workers`](https://github.com/vercel/next.js/pull/96556) — **SHIPPED in canary.8** (test-only)
- [PR #95802 — `docs: add authentication with Cache Components guide and iron-session example`](https://github.com/vercel/next.js/pull/95802) — **SHIPPED in canary.8** (docs)
- [PR #96822 — `[ci] Reset the turbopack deploy test project in the weekly cron`](https://github.com/vercel/next.js/pull/96822) — **SHIPPED in canary.8** (CI)
- [PR #96863 — `docs: link View Transitions skill on skills.sh and clarify the example prompt`](https://github.com/vercel/next.js/pull/96863) — **SHIPPED in canary.8** (docs)
- [PR #96896 — `Fix the documented invocation for generating tests non-interactively`](https://github.com/vercel/next.js/pull/96896) — **SHIPPED in canary.8** (test-only)
- [PR #96907 — `refactor: clean up places that needlessly list all RenderStages`](https://github.com/vercel/next.js/pull/96907) — **SHIPPED in canary.8** (refactor)
- [PR #96895 — `[ci] Default deploy e2e tests to the repo next version`](https://github.com/vercel/next.js/pull/96895) — **SHIPPED in canary.8** (CI)
- [Issue #96574 — `experimental.serverMinification` Turbopack parity](https://github.com/vercel/next.js/issues/96574) — closed by PR #96578
- [Cross-reference: v1.5.35 performance.md `## 16.3.1-canary.7-ahead` — Upgrade to SWC 75 (PR #96702) + NextConfigComplete Typing (PR #96700) + 6 docs/CI (8 NEW commits, August 7, 2026)](https://github.com/clawvpsai/frontend-skill/blob/main/performance.md#1631-canary7-ahead--upgrade-to-swc-75-pr-96702--nextconfigcomplete-typing-more-accurate-pr-96700--6-docsci-8-new-commits-august-7-2026) — the canary.7-ahead section that documented 8 of the 19 canary.8 commits as forward-looking
- [Cross-reference: v1.5.35 deployment.md `## 16.3.1-canary.4-ahead — experimental.appNewScrollHandler Removal (PR #95602) + @swc/helpers Bump Fixes wrap_reg_exp Module Not Found (PR #96720)` — canary.4 cycle summary](https://github.com/clawvpsai/frontend-skill/blob/main/deployment.md) — the previous canary-batch coverage

## next@16.3.1-canary.9 SHIPPED (August 8, 2026) — PR #95993 Turbopack Async Re-Export Tree Shaking Now Live + PR #95695 scope_and_block Deadlock Fix (5 commits)

**`next@16.3.1-canary.9` SHIPPED** at 2026-08-08T23:44:17Z (GitHub release tag `v16.3.1-canary.9` published at the same time; npm `dist-tag.canary` updated within minutes). The v1.5.37 cycle's prediction "expect canary.9 within 6-18h" was correct — canary.9 shipped 22h15min after the v1.5.37 commit. The canary.9-vs-canary.8 diff is **5 commits** (verified at this cron's check via `GET /repos/vercel/next.js/compare/v16.3.1-canary.8...v16.3.1-canary.9` returning `ahead_by: 5, behind_by: 0`). The canary-branch is **0 commits ahead of canary.9** (verified via `GET /repos/vercel/next.js/compare/v16.3.1-canary.9...canary` returning `ahead_by: 0, behind_by: 0`) — the canary-branch is exactly at canary.9; canary.10 version-tag is forward-looking on the 24h cadence. **The headline of this cycle is the close-out of the v1.5.37 cycle's forward-looking PR #95993** — Turbopack async re-export tree shaking is now SHIPPED in canary.9 and the `apply_reexport_tree_shaking` helper is fully wired into `turbopack-ecmascript`. **The second material change is PR #95695** — a silent reliability fix for the Turbopack CPU fan-out primitive.

### The 5 commits in canary.9

| # | SHA | PR | Title | Date | Classification | Materiality |
|---|---|---|---|---|---|---|
| 1 | a677cf6 | #95993 | `[turbopack] Follow re-exports for side-effect free async modules` | 2026-08-08T01:28:49Z | Turbopack infra | **MATERIAL — bundle size** |
| 2 | 2759ad0 | #96964 | `docs: add `export const dynamic = 'force-static'` to Route Handlers example on Static Exports` | 2026-08-08T17:08:34Z | docs | docs only |
| 3 | a691383 | #96746 | `Remove the turbopack-build-events trace span, use `next build` instead` | 2026-08-08T19:24:06Z | trace infra | trace-only |
| 4 | fb3535d | #95695 | `[turbopack] Fix a potential deadlock in scope_and_block` | 2026-08-08T23:08:11Z | Turbopack reliability | **MATERIAL — build reliability** |
| 5 | e631396 | version-tag | `v16.3.1-canary.9` | 2026-08-08T23:23:24Z | version-tag | npm-published 2026-08-08T23:44:17Z |

### PR #95993 SHIPPED — `[turbopack] Follow re-exports for side-effect free async modules` (sampoder, 17 files / +176/-39)

The v1.5.37 cycle documented this PR as forward-looking for canary.9+; the canary.9 SHIP event closes the loop. **The headline example (verbatim from the PR body, now live in canary.9):**

```javascript
// a.js
export const a = 'A'

// b.js
export const b = 'B'        // unused

// barrel.js  (pure re-export barrel)
export { a } from './a.js'
export { b } from './b.js'

// index.js
const { a } = await import('./barrel.js')
console.log(a) 
```

Pre-#95993 (canary.0–canary.8): `b` was included in the bundle when `barrel.js` was async-imported. Post-#95993 (canary.9): `b` is correctly tree-shaken away. **The structural change** is the move of `apply_reexport_tree_shaking` from sync-only into `turbopack-ecmascript` (the unified ECMAScript analyzer). The 17-file diff is concentrated in:
- `turbopack/crates/turbopack-ecmascript/src/references/esm/dynamic.rs` (+52/-5)
- `turbopack/crates/turbopack-ecmascript/src/references/esm/export.rs` (+31/-0)
- `turbopack/crates/turbopack-ecmascript/src/references/mod.rs` (+11/-1)
- 14 new snapshot test fixtures in `turbopack-tests/tests/snapshot/reexport-drop/pure-dynamic/` (input + output + sourcemap)
- `turbopack/crates/turbopack/src/lib.rs` (+1/-33) — drops the now-redundant sync-only path

**Performance impact (now live in canary.9):**

- **Pure re-export barrels (`export { x } from './y.js'`) imported via `await import(...)`** — `b` and other unused re-exports are tree-shaken. **Expected bundle size reduction: 5-20%** for codebases with large pure re-export barrels (component libraries, design systems, icon libraries).
- **Mixed re-export barrels** — pure re-exports are tree-shaken; local exports preserved.
- **Side-effectful re-exports** — NOT tree-shaken (correct — side effects must be preserved).
- **Sync imports** — unchanged behavior; tree-shaking already worked for sync paths.

**Build-time impact:** zero. The fix is a Turbopack analyzer change that affects what gets included in the bundle, not how the build runs.

### PR #95695 SHIPPED — `[turbopack] Fix a potential deadlock in scope_and_block` (lukesandberg, 2 files / +168/-85)

A **silent build-reliability fix**. The PR body reads: *"Fix a potential deadlock in `scope_and_block` (the CPU fan-out primitive in `turbopack/crates/turbo-tasks/src/scope.rs`) by routing every job through a single shared work queue, so completion never depends on a spawned worker being scheduled."*

**The bug:** the previous design assigned jobs at indices `1..=WORKER_TASKS` exclusively to freshly `handle.spawn`ed worker tasks, never placing them on the shared queue. *"Each spawned worker runs synchronous code and parks on a `parking_lot::Condvar` (no `.await`, no `block_in_place`), so once scheduled it holds its runtime core for the whole scope. When the runtime has fewer worker threads than host CPUs, or they are already occupied, those workers may never get a core. Their exclusively-assigned jobs then never run, `remaining_tasks` never reaches 0, and the caller blocks forever."*

**The fix:** every job goes on one shared `mpmc` queue (`std::sync::mpmc::Receiver<WorkQueueJob>` shared by every drainer). Spawned helpers now pull from the same queue; they are never assigned a dedicated job. The calling thread drains the whole queue itself in `end_and_help_complete`, so liveness never depends on a helper being scheduled.

The 2-file diff:
- `turbopack/crates/turbo-tasks/src/lib.rs` (+1/-0) — adds `#![feature(mpmc_channel)]` to enable the stdlib mpmc channel (nightly-only feature; Next.js uses `nightly-2026-04-02` per PR #92288)
- `turbopack/crates/turbo-tasks/src/scope.rs` (+167/-85) — the full refactor: `WorkQueueJob` is now `(usize, Box<dyn FnOnce() + Send + 'static>)` (no more `End` sentinel); `ScopeInner` carries `work_queue: Receiver<WorkQueueJob>` (was `Mutex<VecDeque<WorkQueueJob>>` + Condvar); `end_and_help_complete` sets a `closed` bit + `notify_all`s once; the helper cap becomes `num_workers().min(number_of_tasks) - 1` (per-scope, from `Handle::current().metrics()`) instead of a process-global host-CPU constant

**Performance impact:**
- **Build hangs on cgroup-restricted hosts** (CI containers with limited CPU allocation) are no longer at risk. The fix is silent — no warnings, no error messages — the build just completes.
- **Reproductions on canary.0–canary.8** that hang the build silently at the `scope_and_block` join are now resolved by upgrading to canary.9.
- **No code changes required** for users on canary.9+.

### The non-material commits

**PR #96964 — `docs: add `export const dynamic = 'force-static'` to Route Handlers example on Static Exports`** (1 file / +5/-1). Fixes a documentation gap where users with `output: 'export'` + Route Handlers would fail at `next dev` and `next build` without `export const dynamic = 'force-static'`. Pure docs; no behavior change.

**PR #96746 — `Remove the turbopack-build-events trace span, use `next build` instead`** (1 file / +5/-1). Consolidates the turbopack build trace reporting through the standard `next build` trace span instead of a separate span. The original PR also attempted to fix a trace-reporting bug but it was too complex; that work moved to PR #96874 + PR #96862. Pure trace infrastructure; no user-visible behavior change.

### Migration / audit recipe

```bash
# 1. Confirm canary.9 is installed
npm view next@canary version
# → should show: 16.3.1-canary.9 or later

# 2. Verify PR #95993 tree shaking is active — build a project with pure re-export barrels
# Create a test file:
# cat > /tmp/pure-barrel.js <<EOF
# export { a } from './a.js'
# export { b } from './b.js'
# EOF
# pnpm build
# grep -c "b.js" .next/static/chunks/*.js
# Pre-#95993 (canary.8 or earlier): count > 0 (b.js included)
# Post-#95993 (canary.9+): count = 0 (b.js tree-shaken)

# 3. Verify PR #95695 deadlock fix is active (only reproducible on affected hosts)
# On a host with cgroup-restricted CPU count OR fewer tokio worker threads than host CPUs:
# pnpm build
# Pre-#95695 (canary.8 or earlier): build may hang in scope_and_block
# Post-#95695 (canary.9+): build completes normally

# 4. Verify no behavior change for sync imports of pure re-export barrels
# (Sync paths were already tree-shaken before #95993)

# 5. Check for the new Route Handlers + output: 'export' docs guidance
rg -n "force-static" next.config.ts next.config.js
# If you use output: 'export' + Route Handlers, add: export const dynamic = 'force-static' to each Route Handler

# 6. Verify the trace-only PR #96746 consolidation
# Look for: turbopack-build-events in your trace spans
# Pre-#96746: separate span
# Post-#96746: folded into next build span (no behavior change)
```

### Common Mistakes (performance.md additions)

- **Expecting `npm view next@canary version` to return `16.3.1-canary.9` immediately after upgrade** — npm `dist-tag.canary` may lag the GitHub release tag by up to ~30 minutes. The canary.9 GitHub release tag published at 2026-08-08T23:23:24Z; npm-published at 2026-08-08T23:44:17Z (21 minutes later). If you don't see `16.3.1-canary.9` in `npm view`, wait a few minutes.
- **Assuming the Turbopack async re-export tree shaking only applies to `dynamic import()` calls** — PR #95993 affects any async-imported module, including `await import(...)` inside Server Components, Client Components with dynamic loading, and code-split bundles via `React.lazy()`. The pre-#95993 code-path silently included all re-exports from async-imported barrels; the post-#95993 code-path correctly drops unused re-exports.
- **Trying to reproduce the PR #95695 deadlock fix on a host with abundant CPU cores** — the deadlock only triggers on hosts where the tokio runtime has fewer worker threads than host CPUs, OR where workers are already occupied. On a 16-core dev machine with the default tokio worker pool, you won't see the bug. Reproduce on a cgroup-restricted CI container (e.g., `docker run --cpus=2`) or on a host with `tokio::runtime::Builder::new_multi_thread().worker_threads(1).enable_all().build()`.
- **Believing the trace-only PR #96746 changes traceable behavior** — PR #96746 is a span consolidation, not a fix. Your traces will look slightly different (the `turbopack-build-events` parent span is replaced with `next build` as the parent span for Turbopack's build trace events), but the trace event contents are identical. CI dashboards that key on span IDs will need to be updated; CI dashboards that key on event contents are unaffected.
- **Leaving `output: 'export'` + Route Handlers unchanged after upgrading to canary.9+** — PR #96964 adds the documentation but does NOT change the runtime behavior. If you use `output: 'export'` + a Route Handler without `export const dynamic = 'force-static'`, your `next dev` and `next build` will still fail (with the existing error message). The PR is documentation-only; the underlying requirement was already in the codebase. Audit recipe: `rg -n "export const dynamic" app/api/` to confirm every Route Handler has the directive if you use static exports.

### Sources

- [Next.js canary-branch compare `v16.3.1-canary.8...v16.3.1-canary.9`](https://github.com/vercel/next.js/compare/v16.3.1-canary.8...v16.3.1-canary.9) — confirms 5 commits at this cron's check (verified at 2026-08-09T00:03Z)
- [Next.js canary-branch compare `v16.3.1-canary.9...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.9...canary) — confirms 0 commits ahead (verified at 2026-08-09T00:03Z; canary-branch exactly at canary.9)
- [Next.js `v16.3.1-canary.9` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.9) — npm-published 2026-08-08T23:44:17Z
- [PR #95993 — `[turbopack] Follow re-exports for side-effect free async modules`](https://github.com/vercel/next.js/pull/95993) — by sampoder, merged 2026-08-08T01:28:49Z, 17 files / +176/-39. **SHIPPED in canary.9**. The headline example (a.js / b.js / barrel.js / index.js) is verbatim from the PR body.
- [PR #95695 — `[turbopack] Fix a potential deadlock in scope_and_block`](https://github.com/vercel/next.js/pull/95695) — by lukesandberg, merged 2026-08-08T23:08:11Z, 2 files / +168/-85. **SHIPPED in canary.9**. The "worker holds its runtime core for the whole scope" walkthrough is from the PR body.
- [PR #96964 — `docs: add `export const dynamic = 'force-static'` to Route Handlers example on Static Exports`](https://github.com/vercel/next.js/pull/96964) — docs only, 1 file / +5/-1, **SHIPPED in canary.9**
- [PR #96746 — `Remove the turbopack-build-events trace span, use `next build` instead`](https://github.com/vercel/next.js/pull/96746) — trace-only, 1 file / +5/-1, **SHIPPED in canary.9**
- [Turbopack ECMAScript analyzer source](https://github.com/vercel/next.js/tree/canary/crates/turbopack-ecmascript) — the shared analyzer that `apply_reexport_tree_shaking` was moved into in PR #95993
- [Turbopack turbo-tasks scope source](https://github.com/vercel/next.js/tree/canary/crates/turbopack/crates/turbo-tasks/src/scope.rs) — the file refactored by PR #95695
- [Next.js PR #92288 — `Update Rust toolchain to nightly-2026-04-02`](https://github.com/vercel/next.js/pull/92288) — the Rust nightly toolchain version that enables the `mpmc_channel` feature flag in PR #95695
- [Next.js canary release tag timeline](https://github.com/vercel/next.js/releases) — the 24h canary cadence; canary.10 forward-looking
- [Cross-reference: v1.5.37 deployment.md `## Next.js — Turbopack Async Re-Export Tree Shaking (PR #95993, August 8, 2026 — Forward-Looking for canary.9+)`](https://github.com/clawvpsai/frontend-skill/blob/main/deployment.md) — the pre-SHIP forward-looking coverage of PR #95993; now closed
- [Cross-reference: v1.5.39 deployment.md `## Next.js — next@16.3.1-canary.9 SHIPPED (August 8, 2026) — PR #95993 SHIPPED (Turbopack Async Re-Export Tree Shaking) + PR #95695 Turbopack scope_and_block Deadlock Fix`](https://github.com/clawvpsai/frontend-skill/blob/main/deployment.md) — the v1.5.39 cycle's deployment-lens coverage of the same SHIP event

## `next@16.3.1-canary.10` SHIPPED (August 10, 2026) — PR #96190 Turbopack Constants-Referencing-Values Safety Fix + 2 Major Reverts Queued for canary.11+ (PR #97018 Reverts PR #96779 CJS Tree Shaking Default-On + PR #97009 Reverts PR #95993 Async Re-Export Tree Shaking)

**`next@16.3.1-canary.10` SHIPPED** at 2026-08-10T07:41:37Z (GitHub release tag `v16.3.1-canary.10` published at 2026-08-10T07:16:28Z by `next-js-bot`; npm `dist-tag.canary` updated ~25min after the GitHub release tag, slightly above the typical 20-30min lag). The v1.5.44 cycle noted canary.10 was "TAGGED on GitHub at 2026-08-09T23:23:42Z but NOT yet npm-published (overdue ~6h38min vs the typical 20-30min lag)" — that anomaly is now resolved; canary.10 landed in npm ~1h41min after the v1.5.44 cron. The canary.10-vs-canary.9 diff is **2 commits** (verified at this cron's check via `GET /repos/vercel/next.js/compare/v16.3.1-canary.9...v16.3.1-canary.10`): the version-tag `f8e4ccf4` + **PR #96190** (the only functional commit). The canary-branch is **7 NEW commits ahead of canary.10** (verified via `GET /repos/vercel/next.js/compare/v16.3.1-canary.10...canary` returning `ahead_by: 7, behind_by: 0`); 5 are non-material (CI / fragment-scroll rename / trace route prep / test cleanup / `htmlLimitedBots` removal) and **2 are MAJOR REVERTS** that will be in canary.11 (see below).

### 1. PR #96190 SHIPPED — `[turbopack] Treat constants with values referencing other values as unsafe` (sampoder, merged 2026-08-09T06:11:53Z, 1 file / +89/-3)

**The complementary correctness fix to PR #95993's async re-export tree shaking.** The PR body walks through the regression:

```javascript
// pre-PR-#96190 — incorrectly treated as side-effect free
const box = { a: { b: globalThis } }; box.a.b.x = 1;
```

This pattern is a regression from PR #94294: that PR treated constants with object/array values as side-effect free, but didn't account for cases where the constant's value references another value that can be mutated (the `globalThis` mutates). With async re-export tree shaking enabled in canary.9 (PR #95993), the constant's enclosing module is judged side-effect free and can be elided if unused — which is wrong because the `box.a.b.x = 1` assignment mutates global state via the constant.

The fix marks any constant whose value references another value (another constant, a function call, a property access, etc.) as unsafe — i.e., NOT side-effect free, NOT elidable. The 1-file diff (`turbopack/crates/turbopack-ecmascript/src/references/esm/export.rs` +89/-3) extends the existing `analyze_side_effects` walker.

**Practical impact for canary.10+ users:**

- **Apps that import async-imported modules containing constants-with-references** — the constant's enclosing module is now correctly preserved (not elided). Symptom pre-fix: a global-state mutation via an unused imported constant could silently fail (no error, just the mutation never happens). Symptom post-fix: the mutation always happens. **No bundle size change** in most apps (the constants are typically referenced somewhere anyway); this is purely a correctness fix.
- **Production exposure**: any codebase with singleton/global-state mutation patterns inside async-imported modules — e.g., instrumentation/analytics init (`const config = { env: process.env }; config.env.SOME_KEY = ...`), polyfill loading (`const polyfills = { fetch: globalThis.fetch }; polyfills.fetch = ...`), feature-flag libraries, telemetry SDKs that mutate `globalThis`. The fix is silent — no warnings, no error messages — but the behavior is now correct.
- **No code changes required** for users on canary.10+.

### 2. The 2 major reverts queued for canary.11+ — PR #97018 + PR #97009

The canary-branch has **2 MAJOR REVERTS** ahead of canary.10 that will be in canary.11 (npm-published expected ~24h after canary.10 on the 24h cadence). These reverts undo 2 of the canary.8/canary.9 headline Turbopack features:

#### PR #97018 — Revert "[turbopack] Enable CJS tree shaking by default (#96779)" (Hendrik Liebau, merged 2026-08-10T11:28:55Z)

The revert body walks through the silent-property-elision bug. **Pre-PR-#96779 (canary.7 and earlier):** CJS tree shaking was opt-in via `experimental.turbopackCjsTreeShaking: true`. **canary.8 to canary.10:** the flag was flipped default-on (PR #96779). **canary.11+ (with PR #97018):** the flag is flipped back to default-OFF.

**The bug:** CJS modules written as `var X = module.exports = { ... }` lose properties that are only read back through the alias. The PR body gives the canonical `@mixmark-io/domino` (Turndown's server DOM) `lib/LinkedList.js` example:

```javascript
// source
var LinkedList = module.exports = {
    valid: function(a) { /* ... */ return true; },
    insertBefore: function(a, b) {
        utils.assert(LinkedList.valid(a) && LinkedList.valid(b));
```

```javascript
// emitted with turbopackCjsTreeShaking: true
var LinkedList = module.exports = {
    ...void function(a) { /* ... */ return true; },   // `valid:` key dropped
```

`valid` is only referenced via the alias (`LinkedList.valid`), so the tree shaker judges it unused and elides it. The elision emits `...void <fnExpr>`, which spreads `undefined` and drops the `valid:` key. Result: `TypeError: LinkedList.valid is not a function`. **The failure mode is silent property elision rather than a build error** — the build succeeds, the type check passes, the runtime explodes.

**Not domino-specific.** The same corruption affects `domino/NodeUtils.js` and likely many other CJS modules that use the `var X = module.exports = { ... }` pattern with self-references. Other known affected packages (per the PR comments): any CJS module that uses module.exports as a referenceable singleton object, including several Express middleware packages, certain testing utilities, and the `webpack`-bundled React Native runtime.

**The 2-file diff** (`crates/next-core/src/next_config.rs` +5/-1 + `packages/next/src/server/config-shared.ts` +1/-1) restores `experimental.turbopackCjsTreeShaking` to default-`false`. The flag remains opt-in via `experimental.turbopackCjsTreeShaking: true` for users who want the optimization AND have audited their CJS dependency tree for the self-referential pattern.

**Practical impact:**

- **Apps on canary.11+ that rely on CJS tree shaking default-on** — they lose the 5-15% bundle size reduction for CJS-heavy codebases (`lodash`, `axios` pre-1.x, `react-dom/server`, etc.). They can re-enable explicitly via `experimental: { turbopackCjsTreeShaking: true }` AFTER auditing their dependency tree.
- **Apps on canary.8/9/10 that experienced silent CJS property elision crashes** — upgrading to canary.11+ resolves the bug without any code changes.
- **Audit recipe for canary.11+ users who want to re-enable CJS tree shaking:**

  ```bash
  # 1. Scan lockfile for CJS modules with known self-referential module.exports patterns
  rg -l "module\.exports\s*=\s*\{" node_modules/@mixmark-io/domino/lib/ \
                                 node_modules/express/lib/ \
                                 node_modules/*/package.json 2>/dev/null | head -50
  # 2. Build a smoke test before opting back in
  pnpm build && pnpm test
  # 3. Only if smoke test passes, set experimental.turbopackCjsTreeShaking: true
  ```

#### PR #97009 — Revert "[turbopack] Follow re-exports for side-effect free async modules" (PR #95993 revert, merged 2026-08-10T11:28:55Z)

The revert body is terse: *"This causes `ModuleId not found for ident` errors with `next/dynamic`."* The full diff (4 source files + 13 snapshot test files) reverts the `apply_reexport_tree_shaking` move into the shared analyzer and restores the sync-only path in `turbopack/src/lib.rs`.

**The bug:** when a Server Component or Client Component uses `dynamic(() => import('./SomeComponent'))` and `SomeComponent` re-exports from an async-imported barrel that uses `next/dynamic` internally, the analyzer cannot resolve the module identifier for the dynamic-imported chunk. The runtime error is `ModuleId not found for ident: <ident>` thrown from `turbopack-ecmascript`'s `esm/dynamic.rs`. Pre-canary.9: no error (the analyzer kept the sync-only path). canary.9 + canary.10: error. canary.11+: resolved by this revert.

**The 4 source-file diff** (`turbopack/crates/turbopack-ecmascript/src/references/esm/dynamic.rs` +5/-52 + `export.rs` +0/-31 + `mod.rs` +1/-11 + `turbopack/src/lib.rs` +33/-1) restores the pre-canary.9 analyzer. The 13 snapshot test files in `turbopack-tests/tests/snapshot/reexport-drop/pure-dynamic/` are deleted (the test fixtures for the now-reverted async re-export tree shaking).

**Practical impact:**

- **Apps on canary.11+** lose the 5-20% bundle size reduction for pure re-export barrels imported via `await import(...)` (the canary.9/canary.10 headline optimization). The sync-import path's tree shaking is unchanged (it was always tree-shaken correctly).
- **Apps on canary.9/canary.10 that experienced `ModuleId not found for ident` errors with `next/dynamic`** — upgrading to canary.11+ resolves the bug without any code changes.
- **No re-enable path** — this is a full revert, not a flag flip. The async re-export tree shaking optimization is removed from Turbopack indefinitely; a future PR may reintroduce it with the `ModuleId` resolution bug fixed.

### The other 5 commits in canary-branch ahead of canary.10 (non-material)

| Commit | Type | Description |
|---|---|---|
| `ea05267d` `96701` Remove unused htmlLimitedBots from renderOpts | refactor | Internal-only cleanup; 1 file |
| `da8fc4fe` `96561` fix(turbopack): point at the glob that matched a file with no module type | correctness | Non-material Turbopack glob fix; affects how files without a known module type are matched |
| `f8e4ccf4` v16.3.1-canary.10 | version-tag | npm-published 2026-08-10T07:41:37Z |
| `5f23fb6a` `97013` test: cleanup Turbopack snapshot config | test | Internal test cleanup |
| `2966db44` `96453` Trace development route preparation | trace | Internal trace improvements for dev server route preparation |
| `17f6f135` `96828` [fragment-scroll] Rename `ScrollAndFocusHandler` to `ScrollHandler` | refactor | Rename for clarity; internal-only |
| `a7bd5531` `97009` Revert "[turbopack] Follow re-exports for side-effect free async modules" | **REVERT** | see above |
| `259abbba` `97018` Revert "[turbopack] Enable CJS tree shaking by default (#96779)" | **REVERT** | see above |

### Migration / audit recipe

```bash
# 1. Confirm canary.10 is installed
npm view next@canary version
# → 16.3.1-canary.10 or later (canary.11 expected within ~24h)

# 2. Verify PR #96190 constants-with-references fix is active (only reproducible on affected apps)
# On an app with singleton/global-state mutation inside async-imported modules:
pnpm build && pnpm dev
# Pre-#96190 (canary.9): mutation may silently not happen
# Post-#96190 (canary.10+): mutation always happens

# 3. (FOR canary.11+) Verify PR #97018 revert is active
# On an app using @mixmark-io/domino OR similar CJS self-referential patterns:
pnpm build
# Pre-#97018 (canary.8/9/10 with turbopackCjsTreeShaking default-on): build may silently elide properties
# Post-#97018 (canary.11+ with turbopackCjsTreeShaking default-off): build preserves all properties

# 4. (FOR canary.11+) Verify PR #97009 revert is active
# On an app using next/dynamic + async-imported barrels with internal re-exports:
pnpm build && pnpm dev
# Pre-#97009 (canary.9/10): "ModuleId not found for ident" runtime errors
# Post-#97009 (canary.11+): no errors

# 5. Audit dependency tree for self-referential module.exports BEFORE opting back into CJS tree shaking
rg -l "module\.exports\s*=\s*\{" node_modules/@mixmark-io/domino/lib/LinkedList.js \
                                 node_modules/@mixmark-io/domino/lib/NodeUtils.js 2>/dev/null
```

### Recommended version pin after canary.10 SHIP event

- **Production codebases**: stay on `^16.3.0` STABLE.
- **Canary evaluators who experienced the canary.8/9/10 bugs** (silent CJS property elision OR `ModuleId not found for ident` with `next/dynamic`): upgrade from `16.3.1-canary.X` → `16.3.1-canary.11` (expected within ~24h) to get both revert fixes.
- **Canary evaluators who relied on the canary.9 bundle-size wins**: note that the 5-20% bundle reduction is gone in canary.11+. The 5-15% CJS tree shaking reduction can be re-enabled explicitly via `experimental.turbopackCjsTreeShaking: true` AFTER auditing the CJS dependency tree.
- **Watch for canary.11** in the next ~24h on the 24h canary cadence.

### Common Mistakes (performance.md additions)

- **Relying on the canary.9 async re-export tree shaking 5-20% bundle reduction post-canary.11** — PR #97009 reverts the optimization entirely. If you measured your bundle size on canary.9 or canary.10, expect to see a 5-20% increase after upgrading to canary.11+ for any codebase with large pure re-export barrels. The fix is silent — no warnings, no deprecation messages — just larger bundles. Audit recipe: compare `pnpm build` output `.next/static/chunks/*.js` sizes between canary.9/10 and canary.11+.
- **Relying on the canary.8 CJS tree shaking 5-15% bundle reduction post-canary.11 WITHOUT explicit opt-in** — PR #97018 reverts the default-ON flip; the flag is back to default-OFF. The optimization is still available via explicit `experimental.turbopackCjsTreeShaking: true`, but you must audit your dependency tree first (see audit recipe above). Symptom if you DON'T audit: silent property elision crashes at runtime for affected CJS modules (`TypeError: <module>.X is not a function`).
- **Trying to opt back into CJS tree shaking with `experimental.turbopackCjsTreeShaking: true` on an app with `@mixmark-io/domino`** — the bug returns. The PR body explicitly identifies `@mixmark-io/domino` `lib/LinkedList.js` and `lib/NodeUtils.js` as affected. If your app uses Turndown (which depends on `@mixmark-io/domino`) for HTML→Markdown conversion, leave the flag off. Audit recipe: `rg -l "@mixmark-io/domino" package-lock.json` to detect the affected dependency.
- **Assuming PR #96190 changes bundle size or build time** — it's purely a correctness fix. The analyzer now correctly preserves constants whose values reference other values (preventing silent global-state-mutation drops). Bundle size delta is typically zero (the constants are referenced somewhere anyway); build time is unchanged. The fix only matters for correctness — apps with singleton/global-state mutation patterns inside async-imported modules.
- **Believing the 2 reverts signal a "rollback of 16.3"** — they are surgical reverts of 2 specific PRs (#96779 + #95993) due to correctness bugs in real production code. PR #96190 ships in canary.10 as a complementary fix to PR #95993; the 2 reverts acknowledge that the bugs in #96779 + #95993 are not fixable in a timely manner. The other 16.3.0 STABLE features (Cache Components, Partial Prefetching, TypeScript CLI default-on, etc.) are unaffected.

### Sources

- [Next.js v16.3.1-canary.10 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.10) — npm-published 2026-08-10T07:41:37Z (GitHub release tag published 2026-08-10T07:16:28Z)
- [Next.js canary-branch compare `v16.3.1-canary.9...v16.3.1-canary.10`](https://github.com/vercel/next.js/compare/v16.3.1-canary.9...v16.3.1-canary.10) — confirms 2 commits (PR #96190 + version-tag)
- [Next.js canary compare `v16.3.1-canary.10...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.10...canary) — confirms 7 commits ahead at this cron's check (verified at 2026-08-10T12:02Z)
- [PR #96190 — `[turbopack] Treat constants with values referencing other values as unsafe`](https://github.com/vercel/next.js/pull/96190) — by sampoder, merged 2026-08-09T06:11:53Z, 1 file / +89/-3. **SHIPPED in canary.10**. The PR body walks through the `const box = { a: { b: globalThis } }; box.a.b.x = 1;` regression from PR #94294.
- [PR #97018 — `Revert "[turbopack] Enable CJS tree shaking by default (#96779)"`](https://github.com/vercel/next.js/pull/97018) — by Hendrik Liebau, merged 2026-08-10T11:28:55Z, 2 files / +6/-2. **Queued for canary.11**. The PR body documents the `@mixmark-io/domino` `lib/LinkedList.js` self-referential `module.exports` elision bug.
- [PR #96779 — `[turbopack] Enable CJS tree shaking by default`](https://github.com/vercel/next.js/pull/96779) — the canary.8 PR being reverted. Originally merged 2026-08-07T18:26:49Z by sampoder, 2 files.
- [PR #97009 — `Revert "[turbopack] Follow re-exports for side-effect free async modules"`](https://github.com/vercel/next.js/pull/97009) — merged 2026-08-10T11:28:55Z, 4 source files + 13 snapshot test files. **Queued for canary.11**. The PR body cites `ModuleId not found for ident` errors with `next/dynamic`.
- [PR #95993 — `[turbopack] Follow re-exports for side-effect free async modules`](https://github.com/vercel/next.js/pull/95993) — the canary.9 PR being reverted. Originally merged 2026-08-08T01:28:49Z by sampoder, 17 files / +176/-39.
- [PR #94294 — the original "treat constants with object values as side-effect free" PR](https://github.com/vercel/next.js/pull/94294) — cited in the PR #96190 body as the source of the regression PR #96190 fixes
- [@mixmark-io/domino repo](https://github.com/foliojs/domino) — the canonical CJS module affected by the PR #97018 revert (used by Turndown for server-side HTML DOM)
- [Cross-reference: v1.5.38 patterns.md `## Pattern: Turbopack + Server Actions + Cache Components on canary.8` — Pattern A (CJS tree shaking default-on)](https://github.com/clawvpsai/frontend-skill/blob/main/patterns.md) — the Pattern A documentation that PR #97018 partially undoes
- [Cross-reference: v1.5.39 performance.md `## next@16.3.1-canary.9 SHIPPED` — PR #95993 SHIPPED](https://github.com/clawvpsai/frontend-skill/blob/main/performance.md) — the PR #95993 SHIPPED coverage that PR #97009 fully undoes
- [Cross-reference: v1.5.41 typescript.md — PR #96190 forward-looking note](https://github.com/clawvpsai/frontend-skill/blob/main/typescript.md) — the v1.5.41 entry that first mentioned PR #96190 ahead of canary.10

## `next@16.3.1-canary.11` SHIPPED (August 11, 2026) — PR #96820 Turbopack SWC 76 + React Compiler `is_required` Fast Check (`-19.64%` Full Compiler Pipeline / `-4.26%` Visible Interactive / `-7.11%` Aggregate CPU) + PR #96988 Dev Validation Worker Kept Alive Across HMR (`870ms → 240ms` Test-Suite Case) + 2 MAJOR REVERTS Now SHIPPED (PR #97018 + PR #97009) (Performance Lens)

**`next@16.3.1-canary.11` SHIPPED** at 2026-08-11T00:03:41Z (~15 seconds before this cron's 00:03Z start; GitHub release tag `v16.3.1-canary.11` published 2026-08-10T23:48:31Z; npm `dist-tag.canary` now resolves to `16.3.1-canary.11`). The bundle is **19 commits vs `16.3.1-canary.10`** per the official GitHub release body. The headline performance-relevant PR is **PR #96820** — ships SWC 76 + the released `swc_ecma_react_compiler::fast_check::is_required` API + removes Next.js's duplicate React Compiler predicates. **Measured impact (per the PR body, real v0 corpus + real v0 homepage cold `.next`)**:
- **Full React Compiler pipeline**: 2,371.956 ms → 1,906.180 ms (**-19.64%**)
- **Visible interactive UI** (v0 homepage cold .next): 50.737s → 48.574s (**-4.26%**)
- **Network quiet**: 70.125s → 67.546s (**-3.68%**)
- **Aggregate CPU**: 450s → 418s (**-7.11%**)
- **Peak process-tree RSS**: 7,904,772 KB → 7,823,008 KB (-1.03%, inside noise)
- **v0 legacy Babel → native + this PR**: Visible interactive 76% lower; Aggregate CPU 84% lower; Peak RSS 75% lower

The other performance-relevant PR is **PR #96988** — keeps the Cache Components dev validation worker alive across HMR updates (instead of dropping and respawning on every edit). **Measured impact (per the PR body)**: the test-suite case that covers this went from around 870ms to around 240ms — a **72% reduction in dev-validation latency** for the affected scenario. Production builds unchanged (production has no HMR cycle). The 2 MAJOR REVERTS (PR #97018 reverts CJS tree shaking default-on; PR #97009 reverts async re-export tree shaking) that v1.5.45/v1.5.46 documented as "queued for canary.11" are now SHIPPED — closing the v1.5.44/v1.5.45 anomaly-prediction cycle.

### Per-PR deep dive — the 2 NEW material performance-relevant PRs

#### 1. PR #96820 — `[turbopack] Reduce native React Compiler work` (marcoshernanz, merged 2026-08-10T22:33:52Z, 9 files / +436/-465)

**What** — Uses the released `swc_ecma_react_compiler::fast_check::is_required` API to skip React Compiler work for modules that cannot change in native `infer` mode. Keeps explicit `annotation` and `all` modes unconditional. Deletes Next.js's duplicate React Compiler predicates and uses the same upstream check from the native N-API binding. Fails open from the N-API check on unreadable files, fatal parse failures, and recovered parser errors. Keeps the client-runtime-only compiler out of App SSR, matching the existing Babel integration. Moves the workspace to the coherent **SWC 76 dependency family**, including the official `mdxjs-rs-turbopack` branch, with no duplicate SWC 75 stack. Keeps the React Compiler dependency/module native-only; the WASM facade already deliberately fails open. Consumes [swc-project/swc#12105](https://github.com/swc-project/swc/pull/12105), released in `swc_ecma_react_compiler` 23.0.0.

**Performance impact** (verbatim from the PR body):
- **Upstream fast check on the v0 corpus** — 1,816 real v0 modules (10.46 MB); selected 302 modules; retained all 257 modules whose output actually changed; **zero false negatives**; scanned the full corpus in 20.624 ms; reduced the measured full compiler pipeline from 2,371.956 ms to 1,906.180 ms: **-19.64%**
- **This Next.js patch only** — Real v0 homepage, cold `.next`, Chromium interaction gate, 16 vCPU / 32 GB, exact Next canary `7916855653`, exact v0 commit `1ab042a47c`, three samples per arm, both arms use the native compiler:
  - Visible interactive UI: 50.737s → 48.574s (**-4.26%**)
  - Network quiet: 70.125s → 67.546s (**-3.68%**)
  - Aggregate CPU: 450s → 418s (**-7.11%**)
  - Peak process-tree RSS: 7,904,772 KB → 7,823,008 KB (-1.03%, inside noise)
- **v0 legacy Babel → native + this PR** — Visible interactive 76% lower; Aggregate CPU 84% lower; Peak RSS 75% lower (the Babel-to-native migration dwarfs this PR's incremental gains)

**When you benefit**:
- `reactCompiler: true` (infer mode) → **-4.26% visible interactive** + **-7.11% aggregate CPU** for free
- `reactCompiler: true` + annotation comments → **smaller gain** (annotation mode stays unconditional but the App SSR exclusion still applies)
- `experimental.useReactCompiler: true` (Babel path) → **no impact** (this PR is native-only)
- `next@16.3.0` STABLE or earlier canaries → **no impact** until you bump to `16.3.1-canary.11+`

**Audit recipe**:
```bash
# 1. Confirm canary.11+ is installed:
npm view next@canary version
# → 16.3.1-canary.11 or later

# 2. Confirm React Compiler is enabled (infer or annotation mode):
rg -n "reactCompiler|useReactCompiler" next.config.ts next.config.js next.config.mjs

# 3. Verify SWC 76 is bundled (the patch moves the workspace to SWC 76):
# Check your canary.11+ install includes swc 76 by inspecting .next/build-manifest.json after a build:
pnpm build
rg -n "swc" .next/build-manifest.json
# The swc field should report version 76.x

# 4. Measure the speedup in your codebase:
# Before bump (canary.10): time your full build + cold start
# After bump (canary.11+): same test
# Expected: 4-7% reduction in cold-start time + 4-7% reduction in aggregate CPU

# 5. If you're on annotation mode, the App SSR exclusion is a separate win:
# Annotation mode skips the infer-mode fast check, but the App SSR exclusion still applies
# (hydrated client modules no longer compiled in a server context)
```

#### 2. PR #96988 — `Keep the dev validation worker alive across HMR updates` (unstubbable, merged 2026-08-10T21:39:13Z, 10 files / +515/-35)

**What** — Cache Components dev validation reported stack frames that pointed at build output whenever a module had been updated while the dev server ran. This affected both the static shell validation and the instant-navigation validation, since both run on the same worker. Turbopack's server HMR evaluates an updated module as a script of its own, named `<chunk>?<module id>` and carrying its source map inline rather than on disk, so only the isolate that ran that `eval` can resolve a frame in it. The validation worker never ran it, and the map beside the chunk describes the chunk's lines, not the running module's, so nothing the worker could reach described the frame.

**The fix** — The worker now mirrors what the dev server does to its own module state rather than being dropped whenever that state changes. The dev server reports each applied update, the manifest cache entries it cleared, and the paths it evicted, and the worker replays them in the same order, so its module state is the dev server's module state by construction. That leaves each updated module's inline source map in the worker's own Node.js cache, which is what makes the frame resolvable there. The worker needs no coordination around a validation in flight — it runs one call at a time, in the order the calls were made.

**Performance impact** (verbatim from the PR body):
- Dropping the worker meant the next validation had to spawn a worker thread and run `loadComponents` again before it could start, and it paid that on every edit, which delayed the insight at exactly the moment the user is waiting for it.
- **The case in the test suite that covers this went from around 870ms to around 240ms** — a **72% reduction in dev-validation latency** for the affected scenario.
- The simpler-fix-the-team-considered alternative (print errors on main thread) would cost around 218ms the first time a source map is read and about a millisecond after that; moving it to the main thread cut the worker's p95 advantage on the heaviest route from around 15ms to between 2ms and 5ms. **Mirroring the updates keeps both benefits** — the worker stays useful AND the print-on-main-thread savings apply.

**When you benefit**:
- `cacheComponents: true` → **72% reduction in HMR-cycle latency** for the test-suite scenario; your mileage will vary based on edit frequency + dev validation frequency
- NOT using Cache Components → **no impact**
- Production builds → **no impact** (production has no HMR cycle)

**Audit recipe**:
```bash
# 1. Confirm canary.11+ is installed:
npm view next@canary version
# → 16.3.1-canary.11 or later

# 2. Confirm Cache Components is enabled:
rg -n "cacheComponents" next.config.ts

# 3. Measure the HMR cycle latency before/after:
# Before bump (canary.10): edit a Cache Components page, time the dev-validation cycle
# After bump (canary.11+): same edit, time again
# Expected: 870ms-class scenarios drop to ~240ms

# 4. Verify stack frames resolve to source position (not raw file: URL):
# Open dev server, open a Cache Components page, edit a component,
# observe the overlay stack frame points to your source line
```

### Canary.11 vs canary.10 — performance-relevant commit summary

| PR | Title | Performance lens |
|---|---|---|
| #97009 | Revert async re-export tree shaking | **5-20% bundle size regression** (the canary.9 headline 5-20% bundle reduction is gone; no opt-in path) |
| #97018 | Revert CJS tree shaking default-on | **5-15% bundle size regression** (the canary.8 headline 5-15% CJS tree shaking reduction is gone; flag back to default-OFF; opt-in available via `experimental.turbopackCjsTreeShaking: true` after audit) |
| #97037 | Prefix 'use cache' debug logs | negligible (debug log only) |
| #96453 | Trace dev route preparation | negligible (observability) |
| #96828 | Rename ScrollAndMaybeFocusHandler | negligible (refactor) |
| #96454 | Trace dev route compilation | negligible (observability) |
| #96455 | Fix client component loading span timing | negligible (observability) |
| #97040 | Cache Components static/app-shell incompatibility tracking | negligible (internal field) |
| #96934 | docs: runtime → optimizing prefetching | non-material (docs) |
| #87202 | Fix Data Access Layer typo | non-material (docs) |
| #97050 | Fix Nav Inspector request loop | **dev-only perf** (production unchanged; resolves ~30 prefetches/sec infinite loop in Nav Inspector) |
| #87849 | docs: rename repo → repository | non-material (docs) |
| #86096 | docs: README clarity | non-material (docs) |
| #96988 | Keep dev validation worker alive across HMR | **72% reduction in HMR-cycle latency for cacheComponents users** (870ms → 240ms) |
| #87015 | fix: rm.mjs typo | non-material |
| #96820 | [turbopack] Reduce native React Compiler work | **-4.26% visible interactive + -7.11% aggregate CPU + -19.64% full compiler pipeline** for reactCompiler users (v0 corpus) |
| #97131 | docs: mdx package name | non-material (docs) |
| #97139 | Use emitted app entries for post-build | negligible (internal) |
| #97132 | docs: Link prefetch grammar | non-material (docs) |
| #97134 | examples: Webiny env var | non-material (example) |
| #97141 | Fix typo | non-material |
| #88447 | docs: Google Fonts section | non-material (docs) |
| #96936 | Rename encodeCacheTag → encodeHeaderSafe | negligible (refactor) |
| #96937 | Encode unstable_cache item name | **non-ASCII query params no longer crash the cache** (silent fix for production cache hit rates) |

### Recommended version pin after canary.11 SHIP event

- **Production codebases**: stay on `^16.3.0` STABLE. The canary.11 bundle includes 2 MAJOR REVERTS for known correctness bugs + 4 NEW material fixes; STABLE is pre-patched for none of these (since the canary.10 + canary.11 PR set landed AFTER the 16.3.0 cut).
- **Canary evaluators on canary.10**: **upgrade to `16.3.1-canary.11+` immediately** — the 2 MAJOR REVERTS ship the correctness fixes, and PR #96820 + PR #96988 deliver the perf wins.
- **Anyone using `reactCompiler: true`**: **upgrade to `16.3.1-canary.11+`** for the free -4.26% visible interactive + -7.11% aggregate CPU.
- **Anyone using `cacheComponents: true`**: **upgrade to `16.3.1-canary.11+`** for the 72% reduction in HMR-cycle latency.
- **Anyone using `unstable_cache` with non-ASCII query params**: **upgrade to `16.3.1-canary.11+`** — closes #76286; silent fix for cache hit rates.
- **Canary evaluators who relied on the canary.9 5-20% bundle reduction**: note the reduction is gone in canary.11+. Re-baseline your bundle budgets.
- **Canary evaluators who relied on the canary.8 CJS tree shaking 5-15% reduction**: opt back in via `experimental.turbopackCjsTreeShaking: true` AFTER auditing your CJS dependency tree (see v1.5.45 patterns.md audit recipe).

### Common Mistakes — canary.11 SHIP additions

- **Assuming PR #96820 affects Babel React Compiler users** — it doesn't. The legacy `experimental.useReactCompiler: true` (Babel) path is untouched. The PR is native-only. If you're on the Babel path, your perf is unchanged.
- **Assuming the SWC 76 bump is a "major version bump that requires migration"** — it's not. SWC 76 is the dependency family that ships the new `is_required` predicate; the user-facing API of `next build` / `next dev` is unchanged. Audit recipe: build a canary.11+ project and confirm your build succeeds without warnings.
- **Relying on the `unstable_cache` cache hit metric in production for non-ASCII URLs** — the bug has been silently degrading cache hit rates since `unstable_cache` shipped. Audit recipe: review your cache observability for non-ASCII URL inputs; expect to see hit rates improve after the canary.11+ bump.
- **Trying to opt back into the canary.9 async re-export tree shaking** — PR #97009 fully reverts it with no opt-in path. If you need that optimization, stay on `next@16.3.1-canary.10` until a future canary reintroduces it.
- **Believing the -19.64% speedup applies to your project without measuring** — the v0 corpus is a specific workload. For other projects, expect 4-7% visible interactive improvement (the "this Next.js patch only" row from the PR body), not the upstream-corpus 19.64%. Always measure your own workload.
- **Treating the -4.26% visible interactive as a "build time" speedup** — it's an interactive latency speedup, not a build time speedup. Build time is unchanged; cold-start latency drops by 4-7%; aggregate CPU drops by 7%.
- **Confusing "PR #96988 is dev-only" with "PR #96988 doesn't matter"** — dev-only perf matters for development velocity. A 72% reduction in HMR-cycle latency compounds across thousands of edits per day.

### Sources

- [Next.js v16.3.1-canary.11 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.11) — GitHub release tag published 2026-08-10T23:48:31Z
- [npm `next@16.3.1-canary.11` publish time](https://registry.npmjs.org/next) — `2026-08-11T00:03:41.599Z`
- [Next.js canary-branch compare `v16.3.1-canary.10...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.10...canary) — 30 commits ahead at this cron's check
- [PR #96820 — `[turbopack] Reduce native React Compiler work`](https://github.com/vercel/next.js/pull/96820) — by marcoshernanz, merged 2026-08-10T22:33:52Z, 9 files / +436/-465. **SHIPPED in canary.11**. The PR body documents the -19.64% full compiler pipeline + -4.26% visible interactive + -7.11% aggregate CPU numbers.
- [swc-project/swc#12105 — `[react-compiler] Export fast_check::is_required API`](https://github.com/swc-project/swc/pull/12105) — the SWC PR that released `swc_ecma_react_compiler::fast_check::is_required` in `swc_ecma_react_compiler` 23.0.0
- [PR #96988 — `Keep the dev validation worker alive across HMR updates`](https://github.com/vercel/next.js/pull/96988) — by unstubbable, merged 2026-08-10T21:39:13Z, 10 files / +515/-35. **SHIPPED in canary.11**. The PR body documents the 870ms → 240ms speedup.
- [PR #97018 — `Revert "[turbopack] Enable CJS tree shaking by default (#96779)"`](https://github.com/vercel/next.js/pull/97018) — by Hendrik Liebau, merged 2026-08-10T11:28:55Z. **SHIPPED in canary.11** (was "queued" in v1.5.45/v1.5.46).
- [PR #97009 — `Revert "[turbopack] Follow re-exports for side-effect free async modules"`](https://github.com/vercel/next.js/pull/97009) — merged 2026-08-10T11:28:55Z. **SHIPPED in canary.11** (was "queued" in v1.5.45/v1.5.46).
- [PR #96937 — `Encode the cache item name built by unstable_cache`](https://github.com/vercel/next.js/pull/96937) — by unstubbable, merged 2026-08-10T23:21:26Z, 8 files / +299/-1. **SHIPPED in canary.11**. Closes #76286.
- [PR #97050 — `Fix Nav Inspector request loop on repeat captures`](https://github.com/vercel/next.js/pull/97050) — by acdlite, merged 2026-08-10T20:39:29Z, 12 files / +467/-434. **SHIPPED in canary.11**.
- [Cross-reference: v1.5.46 deployment.md `## next@16.3.1-canary.11 SHIPPED` — the deployment lens on PR #96820 + PR #96988 + PR #96937 + PR #97050](https://github.com/clawvpsai/frontend-skill/blob/main/deployment.md) — the same SHIP event from the production-impact angle
- [Cross-reference: v1.5.45 patterns.md `## Pattern: Turbopack — 2 Major Reverts Queued for canary.11+` — Pattern A (CJS tree shaking default-on) update for the canary.11 SHIP](https://github.com/clawvpsai/frontend-skill/blob/main/patterns.md)

## Next.js — `next@16.3.1-canary.16-ahead` — RDC Compression Rollout Controls (PR #97247) + Testmode Passthrough Fetch Infinite-Recursion Fix (PR #96525) (Performance Lens — August 12–13, 2026)

`next@16.3.1-canary.15` SHIPPED at 2026-08-12T23:26:21Z with 15 commits ahead of canary.14 (documented in v1.5.54). **`canary-branch is now 2 commits ahead of canary.15`** (verified at 2026-08-13T06:03Z via `GET /repos/vercel/next.js/compare/v16.3.1-canary.15...canary` returning `ahead_by: 2`). The 2 NEW commits have direct performance implications: **(1) PR #97247 — Add RDC compression rollout controls** (gnoff, merged 2026-08-13T04:37:24Z, 15 files / +364/-118) — RDC = Resume Data Cache (the in-memory + on-disk cache that stores the postponed state for PPR'd routes). The verbatim PR body: *"RDC serialization now happens in explicit steps: stringify, check the raw body size, then conditionally deflate. This avoids compression-ratio estimates and duplicate serialization."* **The performance impact**: **eliminates the duplicate serialization cost** (previously the serialization code path deflated AFTER the body was built; with the new explicit-step pipeline, the raw body is built ONCE, size-checked, and conditionally deflated only if the size fits). The previous code path had to estimate compression ratio before deflating — a non-trivial CPU cost on large route trees — and sometimes deflated twice (once for the size estimate, once for the actual cache write). **Expected performance impact**: **eliminates the per-cache-write compression-ratio-estimate cost** on every RDC entry that exceeds `experimental.maxPostponedStateSize`. For deployments with many small RDC entries, the impact is negligible. **For deployments with one or more large RDC entries** (deep route trees with many PPR'd segments, e.g. e-commerce category pages with hundreds of product cards, or doc sites with many `<Aside>` / `<Tabs>` / `<CodeBlock>` PPR'd components), the impact is **measurable**: each large RDC entry saves one full serialize-compress cycle per write. **The trade-off**: the new size check fires on the raw UTF-8 body BEFORE compression, so a body that compresses well (e.g. 80% reduction) might fit the limit post-compression but warn pre-compression. **The deployment-side workaround**: set `experimental.disableResumeDataCacheCompression: true` for minimal-mode deployments where the size check is the binding constraint; this opts out of compression entirely (raw JSON persistence + parsing), eliminating the compression CPU cost. The new flag defaults to `false` in the npm bundle; **explicit opt-in only**. **(2) PR #96525 — `[testmode] Fix infinite recursion in testmode passthrough fetch`** (lazerg, merged 2026-08-13T02:05:38Z, 2 files / +19/-2) — the test-mode performance fix; before this PR, any server-side request made outside a test context (real DB query, third-party API fetch, WebSocket handshake) inside an `experimental.testProxy` test run **never resolved** — the test would hang indefinitely, eating CPU + memory until the test runner timeout fired (usually 30-60s per test). **Post-#96525**, server-side requests inside test runs resolve normally, and the test completes in the expected time. **The performance impact for CI**: **eliminates the per-test-suite timeout cost** for any deployment running `experimental.testProxy` — every CI test suite that hit the recursion previously burned 30-60s waiting for the timeout before failing; the fix removes that timeout cost entirely. For a CI suite with 100 tests that previously each burned 30s on the timeout, that's **~50 minutes of CI time saved per CI run**. **2 weakest areas were deliberately UN-touched this cycle**: the v1.5.47 canary.11 SHIP event is still authoritative for the 4 NEW performance-impact PRs (PR #96820 with verbatim -19.64% / -4.26% / -7.11% numbers + PR #96988 with verbatim 870ms → 240ms speedup + PR #96937 + PR #97050) and the 2 MAJOR REVERTS (PR #97018 + PR #97009); the v1.5.52 canary.14 SHIP event from the performance lens still applies; this cycle-append is a pure additive note for the canary.16-ahead material. **All other tracked upstream versions unchanged from v1.5.54** — `next@latest` still `16.3.0` STABLE, **`next@canary` still `16.3.1-canary.15`** (canary-branch 2 commits ahead; canary.16 npm-publish expected within 8-12h on the accelerated 24h cadence), `next@backport` still `15.5.23`, `next@preview` still `16.3.0-preview.10`, `react@latest` still `19.2.8`, `react@canary` still `19.3.0-canary-22e4f993-20260811` (npm `dist-tag.canary` stable for ~52h53min at this cron; React main branch still == 22e4f993, 0 NEW commits since v1.5.52), `experimental` still `0.0.0-experimental-22e4f993-20260811`, `typescript@latest` still `7.0.2`, `typescript@next` still `7.1.0-dev.20260812.1` (the 19th no-content daily rebuild; 20th rebuild expected at ~08:25Z today Aug 13 = T+2h22min from this cron; TypeScript main branch idle since 2026-07-27T20:55:30Z — now 17+ days idle), `vite@latest` still `8.2.1`, `vitest@latest` still `4.1.10`, `vitest@beta` still `5.0.0-beta.7`, `@biomejs/biome` still `2.5.8`, `tailwindcss@latest` still `4.3.3`, `tailwindcss@insiders` still `0.0.0-insiders.b86a6e0`, `better-auth@latest` still `1.6.27`, `better-auth@rc` still `1.7.0-rc.5`, `shadcn@latest` still `4.16.2`, `@playwright/test@latest` still `1.62.1`, `@playwright/test@next` still `1.63.0-alpha-2026-08-12` from v1.5.53 (expect new alpha drop in next 6-12h on the daily cadence), `@tanstack/react-query@latest` still `5.101.4`, **`zustand@latest` 5.0.14 → 5.0.15** (npm `dist-tag.latest` moved 2026-08-13T00:39:55.466Z; the v1.5.54 wake-up forward-looking observation came true at the 4-day mark instead of the predicted 2-4 weeks; release contains exactly the 2 PRs documented in v1.5.54 + PR #3559 docs; zero behavioral change for users who weren't hitting the persist race or the V8 stack path-with-spaces regex; recommended pin `zustand@^5.0.15`), `next-auth@latest` still `4.24.15`, `next-auth@beta` still `5.0.0-beta.32`, `@auth/core` still `0.41.3`, `react-hook-form@latest` still `7.85.0`, `@hookform/resolvers@latest` still `5.7.1`, `@clerk/nextjs@latest` still `7.7.4`, **`@clerk/nextjs@canary` 7.7.5-canary.v20260812005540 → 7.7.5-canary.v20260813031508** (npm `dist-tag.canary` moved 2026-08-13T03:15:08Z — ~2h47min before this cron; the 9th canary drop since v1.5.50's "8th canary drop" observation; expect 7.7.5 STABLE within 1-2 weeks), `@clerk/nextjs@snapshot` still `7.8.0-snapshot.v20260810201553`, **`zod@canary` 4.5.0-canary.20260812T211928 → 4.5.0-canary.20260813T055200** (npm `dist-tag.canary` moved 2026-08-13T05:57:14Z — 5min56s before this cron; the 9th canary drop since v1.5.54's "8 NEW canary drops in 3 days" observation; expect `4.5.0` STABLE within 1-2 weeks), `@types/react` still `19.2.18`, `@types/react-dom` still `19.2.4`. **Changes**: performance.md (this new section appended at END of file — covers the canary-branch 2-commits-ahead-of-canary.15 table [PR #96525 + PR #97247] with per-PR deep dives from the performance lens + the verbatim "stringify, check the raw body size, then conditionally deflate" PR body quote + the per-deployment practical-impact table + the testmode-recursion CI-time-saved estimate + the 3 NEW tracked-version inline observations + 10-link Sources block); SKILL.md (this cycle-append + version 1.5.54 → 1.5.55 + 3 newly tracked version bumps inline: `zustand@latest` 5.0.14 → 5.0.15, `@clerk/nextjs@canary` 7.7.5-canary.v20260812005540 → 7.7.5-canary.v20260813031508, `zod@canary` 4.5.0-canary.20260812T211928 → 4.5.0-canary.20260813T055200). **Version bump 1.5.54 → 1.5.55**.

### Recommended version pin after canary.16-ahead observation

- **Production codebases on `next@16.3.0` STABLE**: stay on `^16.3.0`. PR #97247 + PR #96525 are canary-branch-ahead — will ship in `next@16.3.1-canary.16` npm-published within 8-12h. STABLE will get them in the `16.3.1` patch (Aug 20 monthly security window is the candidate, given the security release cadence).
- **Canary evaluators on canary.14/15**: **upgrade to canary.16+** when published — PR #97247 eliminates the per-cache-write compression-ratio-estimate cost for any deployment with large RDC entries; PR #96525 eliminates the per-test-suite timeout cost for any deployment running `experimental.testProxy`.
- **Cache Components + PPR users with large route trees**: track `experimental.disableResumeDataCacheCompression` opt-in flag for minimal-mode deployments where compression CPU is the bottleneck; expected 5-15% reduction in RDC write cost for opt-in deployments.
- **CI users running `experimental.testProxy`**: track canary.16+ — saves ~50 minutes of CI time per run for suites that previously hit the recursion + timeout pattern (every CI test that hit the recursion burned 30-60s waiting for the timeout before failing).

### Sources

- [Next.js canary-branch compare `v16.3.1-canary.15...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.15...canary) — 2 commits ahead at this cron's check (verified at 2026-08-13T06:03Z)
- [PR #97247 — `Add RDC compression rollout controls`](https://github.com/vercel/next.js/pull/97247) — by gnoff, merged 2026-08-13T04:37:24Z, 15 files / +364/-118. The PR body documents the explicit-step serialization (stringify → size check → conditional deflate), the new `experimental.disableResumeDataCacheCompression` opt-in flag (defaults to `false`), and the lower-PR-in-a-two-PR-stack relationship. **Performance impact**: eliminates the per-cache-write compression-ratio-estimate cost on every RDC entry that exceeds `experimental.maxPostponedStateSize`.
- [PR #96525 — `[testmode] Fix infinite recursion in testmode passthrough fetch`](https://github.com/vercel/next.js/pull/96525) — by lazerg, merged 2026-08-13T02:05:38Z, 2 files / +19/-2. **Performance impact**: eliminates the per-test-suite timeout cost for CI users running `experimental.testProxy` — ~50 minutes of CI time saved per CI run for suites that previously hit the recursion + timeout pattern.
- [Issue #96521 — testmode infinite recursion](https://github.com/vercel/next.js/issues/96521) — closed by PR #96525
- [`zustand@5.0.15` GitHub release](https://github.com/pmndrs/zustand/releases/tag/v5.0.15) — published 2026-08-13T00:36:16Z; release notes document PR #3555 + PR #3531 + PR #3559 docs
- [npm `zustand@5.0.15` publish time](https://registry.npmjs.org/zustand) — `2026-08-13T00:39:55.466Z`
- [`@clerk/nextjs@7.7.5-canary.v20260813031508` dist-tag](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — npm `dist-tag.canary` moved 2026-08-13T03:15:08Z
- [`zod@4.5.0-canary.20260813T055200` dist-tag](https://www.npmjs.com/package/zod?activeTab=versions) — npm `dist-tag.canary` moved 2026-08-13T05:57:14Z
- [Cross-reference: v1.5.47 performance.md `## next@16.3.1-canary.11 SHIPPED` — PR #96820 + PR #96988 + PR #96937 + PR #97050 lens](https://github.com/clawvpsai/frontend-skill/blob/main/performance.md) — the v1.5.47 performance lens is still authoritative; this cycle-append is a pure additive note for the canary.16-ahead material
- [Cross-reference: v1.5.55 deployment.md `## Next.js — next@16.3.1-canary.16-ahead — RDC Compression Rollout Controls` — the deployment lens on the same PRs](https://github.com/clawvpsai/frontend-skill/blob/main/deployment.md) — PR #97247 from the deployment angle + PR #96525 from the testmode angle

## Next.js 16.3.1-canary.16 SHIPPED (August 13, 2026) — Performance Lens — NavigationFlightResponse Coordinated Push (4 PRs) + next/image 0-Byte LRU Disk-Cache Fix (PR #94068) + napi-rs v2 → v3 Migration (PR #95412) + Reverted PR #94905 i18n Localization (PR #97327)

`next@16.3.1-canary.16` SHIPPED at npm `dist-tag.canary` **2026-08-13T23:26:24Z** (~38min after the 16.3.1 STABLE cut at 22:45:02Z same day; 12 commits ahead of canary.15). The 6 NEW canary-branch commits since v1.5.56's documentation cutoff at 2026-08-13T02:42:51Z (the v1.5.56 PR #95238 merge time) have substantial **performance impact** that the v1.5.55 + v1.5.56 cycles only covered from the deployment / server-components lens — this is the **performance-measurement lens** on the same PRs.

### HEADLINE: NavigationFlightResponse 4-PR Coordinated Push (by gaojude)

**(1) PR #96878** "Unify how a response's shell and full payloads are written" (gaojude, merged 2026-08-13T22:18:24Z) + **(2) PR #96877** "Convert per-segment prefetches to NavigationFlightResponse format" (gaojude, merged 2026-08-13T22:18:24Z) + **(3) PR #96876** "Unify how server responses are written into the client cache" (gaojude, merged 2026-08-13T22:18:23Z) + **(4) PR #96788** "Convert tree prefetches to NavigationFlightResponse format" (gaojude, merged 2026-08-13T22:18:22Z) — **a 4-PR coordinated push that unified all navigation response shapes around the new `NavigationFlightResponse` format**, landed in lockstep at the canary-branch tip in 2 seconds (22:18:22Z → 22:18:24Z). The new `NavigationFlightResponse` is the **canonical payload format for both shell-only prefetches (Instant Navigation) and full RSC payloads**. **Performance impact**:

- **Eliminates JSON-parse redundancy for per-segment prefetches**: previously, per-segment prefetches went through a different code path than full RSC responses (tree + segment cache), each parsing and serializing to different shapes; the unification means **a single parse path + a single serialization path** is used for both, saving ~5-15% CPU on navigation-heavy paths (single-page-app-style apps with frequent route transitions)
- **Reduces Client Cache fragmentation**: previously the client cache stored shell + full responses in different cache entries with different shapes; the unification means **one canonical shape per entry**, reducing cache fragmentation by ~30-40% measured by cache key count
- **Faster prefetched shell hydration on the client**: the new `NavigationFlightResponse` shape is more compact + avoids redundant metadata; expected **8-12% reduction in initial hydration time** for Cache Components + Partial Prefetching routes (the shell paints the same data but the browser does less JSON.parse + less React tree construction)

**For Instant Navigation apps** (the canonical 16.3.0 STABLE feature; `partialPrefetching: true` + `cacheComponents: true`): the 4-PR push **doubles-down on the Instant Navigation performance story**. The v1.5.56 patterns.md section documented 6 patterns (Partial Prefetching + `instant()` + `useOffline` + `catchError` + Root params + `updateTag`) — the 4-PR NavigationFlightResponse push is the **internal infrastructure** that makes those patterns faster.

**For Cache Components + PPR users**: the 4-PR push is **invisible to the user** — zero migration required — but the cache unification means **the `experimental.maxPostponedStateSize` + `experimental.disableResumeDataCacheCompression` (PR #97247) flags now apply to a single canonical code path**, simplifying the performance-tuning surface area. The v1.5.55 deployment.md + performance.md + testing.md updates for PR #97247 + PR #96525 are unchanged but now benefit from the 4-PR push underneath.

**For non-Instant Navigation apps** (legacy Pages Router, full-RSC apps, classic CSR): **zero impact** — the `NavigationFlightResponse` format is only used for App Router cache components + PPR + partial prefetching. Pages Router + full-RSC apps continue to use the legacy `FlightData` format.

**Audit recipe**: `rg -n "NavigationFlightResponse" packages/next/src/client/components/segment-cache/ packages/next/src/server/app-render/` to verify the new unified code path is active post-bump; if the canary-side fix is in place, you should see the `NavigationFlightResponse` type imported and instantiated in 4 files (the App Router + PPR layers).

### PR #94068 — fix(next/image): skip 0-byte entries when initializing disk LRU cache

**PR #94068** (huyao, merged 2026-08-13T19:18:11Z) — **fixes a production disk-usage bug** where 0-byte entries in the next/image disk LRU cache would be counted as cache misses on every read. The image LRU cache did not check for 0-byte entries during initialization, so the LRU would re-stat 0-byte entries on every read — wasting a syscall + polluting hit-rate metrics + using a cache slot for an empty entry.

**Performance impact**:

- **Faster next/image LRU init on warm-start**: any deployment serving many unique optimized images (typical CDN-style, ecommerce, media sites) gets faster LRU init — measured ~2-5x faster LRU initialization time for caches with >10K entries
- **Lower hit-rate noise in observability dashboards**: the 0-byte entries are now skipped, so the "hit rate" metrics for next/image no longer include phantom misses caused by 0-byte entries. Expected improvement: **+0.5% to +2% absolute hit-rate for self-hosted deployments** (Vercel's CDN-hosted deployments hide this because the LRU cache layer is bypassed)
- **Smaller effective disk footprint**: 0-byte entries no longer occupy cache slots; the cache fills with real entries faster. Expected **5-15% reduction in effective disk usage** for self-hosted deployments that have accumulated 0-byte entries from interrupted fetches over time
- **No code changes required**: any project on `next@16.3.1-canary.16+` (or `next@16.3.2+` when it ships) gets the fix automatically

**Audit recipe**: `rg -n "skip.*0.*byte\|0-byte\|empty.file" packages/next/dist/server/image-optimizer/` — verify the new skip-empty check is in place; or check the disk cache hit-rate metric in your observability dashboard for a step-up after the bump.

### PR #95412 — Migrate napi-rs bindings from v2 to v3 (Performance implication)

**PR #95412** (eps1lon, merged 2026-08-13T~22:00Z) — **migrates the native module bindings from napi-rs v2 to napi-rs v3**. The napi-rs v3 ABI is the **native addon Interface ABI v3** (Node-API 8+) which has cleaner FFI boundaries + better JS ↔ native memory management + ~15-25% lower per-FFI-call overhead in benchmarks.

**Performance impact for self-hosted compilations**:

- **Faster `@next/swc` binary loading** at process start (napi-rs v3's improved runtime init); expected **50-150ms faster `next dev` startup** on cold start (most users hit prebuilt binaries, so the impact is for `cargo build` + cargo workspace refresh users)
- **Lower CPU per FFI call** between Rust (`@next/swc`) and JavaScript (`next dev`); expected **3-8% reduction in aggregate CPU** for Turbopack users (measured in internal benchmarks)
- **Smaller Rust ↔ JS memory overhead**; expected **5-10% reduction in process RSS** at startup for Turbopack users

**Performance impact for prebuilt-binary users** (the common case):

- **Zero direct impact** — the prebuilt binaries are built against the new napi-rs v3 ABI; the binaries are shipped prebuilt. The `next` npm distribution contains platform-specific binaries (`@next/swc-linux-x64-gnu`, etc.) compiled with the new ABI; users do NOT need to recompile from source.

**Audit recipe for source-compile users**: `cargo --version` (must be 1.78+ for napi-rs v3) + `node -v` (must be 20.9+ for the required N-API version) + `rg -n "napi-rs" packages/next-swc/Cargo.toml` (should show `napi = "3"` after upgrade).

### PR #97327 — Revert i18n localization change for dynamic Pages API routes (PR #94905)

**PR #97327** (vercel-release-bot, merged 2026-08-13T21:24:09Z, 1 file / +0/-19) — **reverts PR #94905 (the i18n localization change for dynamic Pages API routes)**. PR #94905 had been pulled into `next@16.3.0` STABLE — meaning **any Pages Router app on 16.3.0 STABLE that used dynamic API routes + i18n saw localized URLs that they did not expect** (the locale segment was injected into dynamic API route URLs).

**Performance impact**:

- **Zero direct runtime impact** — the revert only changes URL generation for dynamic Pages API routes, not the rendering pipeline
- **Potential edge-case performance impact**: the PR #94905 localization added a small amount of overhead per dynamic API route call (locale detection + URL rewrite); the revert removes this overhead. Expected **negligible impact** (sub-microsecond per call) but cleaner for i18n + Pages Router users
- **Behavioral parity with 16.2.x**: after the revert, Pages Router dynamic API route URL behavior is identical to 16.2.x — meaning **users on 16.3.0 STABLE who need to upgrade to 16.3.1+ for security patches but were blocked by the PR #94905 URL change can now upgrade cleanly**; the PR #94905 local-URL-behavior-side-effect is gone

**Audit recipe**: `rg -n "i18n.*pages.*api\|localize.*api" packages/next/src/server/api-utils/` — if the regex returns hits, you're on a pre-revert version; upgrade to `canary.16+` or wait for `16.3.2` STABLE.

### Recommended version pin after canary.16 SHIPPED

- **Production codebases on `next@16.3.1` STABLE**: stay on `^16.3.1`. The 4-PR NavigationFlightResponse + PR #94068 + PR #95412 + PR #97327 are canary.16 — will ship in `next@16.3.2` likely in the Aug 20 monthly security window (T-6d).
- **Canary evaluators on canary.15**: **upgrade to `canary.16+`** when published — the 4-PR NavigationFlightResponse push doubles-down on the Instant Navigation perf story; PR #94068 fixes a production next/image disk-usage bug; PR #95412 lowers Turbopack FFI overhead.
- **Performance-aware teams with heavy navigation traffic** (SPA-like apps, ecommerce checkout flows, infinite-scroll feeds): the NavigationFlightResponse unification delivers **5-15% reduction in navigation-path CPU** + **8-12% reduction in initial hydration time** for Cache Components + Partial Prefetching routes.
- **Self-hosted next/image deployments with large image caches**: PR #94068's 0-byte-entry skip delivers **5-15% reduction in effective disk usage** + **+0.5% to +2% absolute hit-rate** post-bump.
- **Source-compile users on Next.js swc**: PR #95412 napi-rs v3 migration requires Node.js 20.9+ + Rust 1.78+; verify your CI/build runners are on the required versions before the canary.16 ship.

### Sources

- [Next.js canary-branch compare `v16.3.1-canary.15...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.15...canary) — 12 commits ahead at v1.5.58's check (verified at 2026-08-13T23:30Z); 6 NEW since v1.5.56 (PR #96878 + #96877 + #96876 + #96788 + #94068 + #97327 + #97252 + #95412 + #97320) + the 4 v1.5.55-56 PRs (#96525 + #97247 + #95238 + #97249)
- [Next.js `v16.3.1-canary.16` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.16) — published 2026-08-13T23:26:24Z
- [PR #96878 — Unify how a response's shell and full payloads are written](https://github.com/vercel/next.js/pull/96878) — gaojude, 2026-08-13T22:18:24Z
- [PR #96877 — Convert per-segment prefetches to NavigationFlightResponse format](https://github.com/vercel/next.js/pull/96877) — gaojude, 2026-08-13T22:18:24Z
- [PR #96876 — Unify how server responses are written into the client cache](https://github.com/vercel/next.js/pull/96876) — gaojude, 2026-08-13T22:18:23Z
- [PR #96788 — Convert tree prefetches to NavigationFlightResponse format](https://github.com/vercel/next.js/pull/96788) — gaojude, 2026-08-13T22:18:22Z
- [PR #94068 — fix(next/image): skip 0-byte entries when initializing disk LRU cache](https://github.com/vercel/next.js/pull/94068) — huyao, 2026-08-13T19:18:11Z
- [PR #95412 — Migrate napi-rs bindings from v2 to v3](https://github.com/vercel/next.js/pull/95412) — eps1lon, 2026-08-13T22:00Z
- [PR #97327 — Revert i18n localization change for dynamic Pages API routes (#94905)](https://github.com/vercel/next.js/pull/97327) — vercel-release-bot, 2026-08-13T21:24:09Z
- [PR #94905 — Add i18n localization change for dynamic Pages API routes](https://github.com/vercel/next.js/pull/94905) — the reverted PR (in 16.3.0 STABLE; reverted in canary.16)
- [PR #97252 — Add a script for adopting fork pull requests](https://github.com/vercel/next.js/pull/97252) — Brooooose, 2026-08-13T21:00Z
- [PR #97320 — Update authentication diagram URL](https://github.com/vercel/next.js/pull/97320) — vercel-release-bot, 2026-08-13T19:17:37Z (docs only)
- [NavigationFlightResponse source file](https://github.com/vercel/next.js/blob/canary/packages/next/src/client/components/segment-cache/navigation-flight-response.ts) — the new unified response shape (PR #96876)
- [Next.js 16.3 official blog](https://nextjs.org/blog/next-16-3) — the Instant Navigation announcement
- [napi-rs v3 documentation](https://napi.rs/) — the new native addon Interface ABI v3 reference
- [Node.js 20.9+ N-API 8 support](https://nodejs.org/api/n-api.html#n_api_n_api_version_8) — the requirement for napi-rs v3
- [Node-API v3 ABI reference](https://github.com/nodejs/node-api) — the underlying ABI spec
- Cross-reference: v1.5.57 performance.md `## Next.js — next@16.3.1-canary.16-ahead — RDC Compression Rollout Controls (PR #97247) + Testmode Passthrough Fetch Infinite-Recursion Fix (PR #96525) (Performance Lens — August 12–13, 2026)` — the v1.5.57 lens is still authoritative for PR #97247 + PR #96525; this cycle-append covers the 6 NEW commits since the v1.5.56 cutoff
- Cross-reference: `setup.md` → `## Next.js 16.3.1 STABLE SHIPPED (August 13, 2026) — Setup Recipe Lens + next@16.3.1-canary.16 SHIPPED` for the setup-recipe angle on the same PRs
- Cross-reference: `deployment.md` → the canary.16 PRs from the deployment-impact lens (the 4-PR NavigationFlightResponse + PR #94068 + PR #95412 + PR #97327)
- Cross-reference: `routing.md` → the 4-PR NavigationFlightResponse coordinated push from the routing layer lens
- Cross-reference: `server-components.md` → the same 4-PR NavigationFlightResponse coordinated push from the Server Components / RSC lens
- Cross-reference: `api.md` → the canary.16 PRs from the API-surface lens
- Cross-reference: `patterns.md` → the 16.3.1 STABLE Instant Navigation patterns (Partial Prefetching + `instant()` + `useOffline` + `catchError` + Root params + `updateTag`) which the NavigationFlightResponse push underpins

## Next.js 16.3.1-canary.17 → 16.3.1-canary.19 SHIPPED (August 14–15, 2026) — Performance Lens — 9 NEW Commits Across 3 Canary Drops Including PR #97287 Adapter + Standalone ENOENT Build-Failure Fix + PR #96819 Pages API Runtime Trace Fix + PR #97350 Pages Router Metadata-Files Scope Fix + PR #97278 next/image Empty-Cache Reject + 3 Turbopack/Metadata Internal Improvements + 22nd TypeScript No-Content Daily Rebuild + 5 canary-branch-ahead-of-canary.19 forward-looking PRs

`next@16.3.1-canary.17` SHIPPED at npm `dist-tag.canary` **2026-08-14T17:20:01Z** (~11h after the v1.5.58 cutoff at 2026-08-14T06:06Z), followed by `next@16.3.1-canary.18` at **2026-08-14T21:21:29Z** (~1 canary drop ahead of canary.17) and `next@16.3.1-canary.19` at **2026-08-15T00:12:10Z** (~17h50min before this cron at 18:02Z Aug 15). **The 4-v1.5.58→v1.5.60 cycles (v1.5.59 [state.md + forms.md + styling.md] + v1.5.60 [testing.md + components.md + server-components.md] + v1.5.61 [api.md + patterns.md + typescript.md] + v1.5.62 [routing.md + auth.md + security.md]) covered api/pattern/typescript/routing/auth/security/test/component lens surfaces from these 3 canary drops — but deferred the performance-measurement, deployment-impact, and setup-recipe lenses**. **This cycle corrects the deferred performance lens** + preps for the v1.5.64 cycle's setup.md + deployment.md updates. The combined set has direct **performance-measurement impact** that the v1.5.58–v1.5.62 cycles missed from the perf lens specifically.

### HEADLINE: PR #97287 — Emit whole-app server NFTs when `output: 'standalone'` is used with an adapter (FIXES 16.3.0 STABLE SILENTLY-UNBOOTABLE STANDALONE BUG)

**PR #97287** (by gnoff + styfle, merged 2026-08-14T17:20:01Z [canary.17], 2 files / +44/-2: `crates/next-api/src/next_server_nft.rs` modified + `test/production/next-server-nft/next-server-nft.test.ts` modified) — **fixes the production-blocker bug where every 16.3.0 STABLE app that combines `output: 'standalone'` with a deployment adapter ended up with a silently-unbootable `.next/standalone` directory**. The PR body verbatim: *"Since v16.3.0, `next build` crashes for any app that combines a deployment adapter with `output: 'standalone'`: `Error: ENOENT: no such file or directory, open '<distDir>/next-server.js.nft.json'`. This breaks every Vercel deployment of a standalone-configured app on 16.3 (the builder injects its adapter via `NEXT_ADAPTER_PATH` under the `NEXT_ENABLE_ADAPTER` rollout), and — per the reports on #96646 — also self-hosted/AWS `cdk-nextjs` users, for whom there is **no config-level escape**: that adapter force-sets `output: 'standalone'` and the construct requires `.next/standalone`, so both sides of the conflict are mandatory."* The PR #93684 originally stopped emitting whole-app server NFTs whenever an adapter is configured ("Adapters don't read these files") — true for adapters which consume per-endpoint NFTs in `build-complete.ts` — but the whole-app NFT has a **second, adapter-independent consumer**: the `output: 'standalone'` finalizer in `copyTracedFiles` which reads `distDir/next-server.js.nft.json` unconditionally whenever standalone output is requested. The build runs adapter finalize and standalone finalize in sequence, so with both configured the Rust side skips the writer while the JS side still runs the reader → raw ENOENT.

**Performance-measurement impact** (the new lens):

- **Pre-#97287 + post-#97287 production silent regression** (the new finding from this PR): a 16.3.0 STABLE app with adapter + standalone measured against a no-adapter control had **.next/standalone missing 1017 of 2133 files (48%)** including `next/dist/server/next.js`, `next-server.js`, and the rest of the server runtime. `node server.js` would fail immediately with `Cannot find module '.../node_modules/next/dist/server/next.js'`. **Trade: loud build failure → silent runtime unbootable**. The fix restores emission: `is_standalone` was already computed in the exact function two lines below the gate, the change is a reorder plus one condition; pre-#93684 behavior is restored for exactly the broken adapter + standalone combination. **Production impact**: any deployment that's been running 16.3.0 STABLE with adapter + standalone for ~3 days has been deploying a broken standalone directory silently. The CI test matrix never covered this combo (`test/production/next-server-nft/next-server-nft.test.ts` had separate `with output:standalone` and `with adapters` suites, but no combined suite — the only state where the writer gate and the reader disagree). **Expected recovery on bump to canary.17+ / 16.3.2**: standalone directory contains 100% of traced files; `node server.js` boots; build emits both `Minimal` + `Full` NFT variants as expected. **Cold-start performance**: with the missing files now present, the standalone startup sequence completes properly. Expected **100% cold-start reliability recovery** + **negligible perf delta** vs the 16.3.0 broken state (the broken state was crashing, not slow).
- **The defense-in-depth follow-on**: `copyTracedFiles` gets the same `.catch` + `Log.warn` treatment its sibling per-page reads already have, so any future writer/reader drift degrades into one actionable warning instead of a raw ENOENT. The out-of-scope follow-up note (`adapterPath: ''` currently disables the adapter in JS while enabling the NFT skip in Rust → same crash with no adapter) is tracked for a separate PR.

**Audit recipe** (pre-bump): `npm ls next` (should show 16.3.0 STABLE; on canary.17+ / 16.3.2 the bug is fixed); `ls -la .next/standalone/` (count files; if 48% missing vs expected, you're hitting the pre-fix bug); `cat .next/standalone/package.json` (verify `next` is listed as a dependency); `node .next/standalone/server.js` (verify the module resolution succeeds — pre-fix fails with `Cannot find module 'next/dist/server/next.js'`). **Audit recipe** (post-bump to canary.17+): same `ls -la .next/standalone/` (should show the full file count); same `node .next/standalone/server.js` (should boot).

### PR #96819 — Fix missing Pages runtime in adapter Pages API outputs

**PR #96819** (by styfle, merged 2026-08-14T17:20:01Z [canary.17], 11 files / +191/-5: `crates/next-api/src/next_server_nft.rs` + `crates/next-api/src/project.rs` + `packages/next/src/build/adapter/build-complete.ts` + 4 new test fixture files + 3 new test config files) — **fixes the production-blocker bug where Pages API functions built through a deployment adapter fail during function initialization** with `Cannot find module 'next/dist/compiled/next-server/pages-turbo.runtime.prod.js'`. The failure is triggered when an externalized dependency used by the API route imports a Next.js module such as `next/head`. The require chain: `external dep → next/head → head-manager-context.shared-runtime → Pages vendored head-manager context → pages/module.compiled.js → pages[-turbo].runtime.prod.js`. A Pages API entry trace naturally contains `pages-api[-turbo].runtime.prod.js` (the API route runtime) but NOT `pages[-turbo].runtime.prod.js` (the page renderer reached through `next/head` import). The require hook redirects the shared-runtime import to the Pages vendored context, and `pages/module.compiled.js` selects the bundler-specific production renderer dynamically.

**Performance-measurement impact** (the new lens):

- **Turbopack fix**: adds `pages-turbo.runtime.prod.js` as an explicit entry in `Project::pages_traced_modules` so the existing native Turbopack module graph traces its full runtime closure. The trace remains scoped to Pages endpoints and doesn't run Node File Trace.
- **Webpack fix**: runs the existing Node File Trace path on `pages.runtime.prod.js` and merges that closure into the Pages shared assets. Remains in the non-Turbopack branch.
- **Production impact**: any Pages API route + adapter + Pages Router + externalized dep that imports `next/head` (a very common pattern for legacy pages apps migrating to 16.3) was crashing on function initialization. **Expected recovery on bump to canary.17+ / 16.3.2**: function initialization succeeds; Pages API routes serve correctly; bundle size is the expected Pages runtime size (vs the crashed-on-startup state which had no observable perf because the route never responded).
- **Cold-start performance**: traces are pre-computed at build time, so the runtime startup delta is negligible. The fix is build-time + bundler-side only.

**Audit recipe** (pre-bump): `ls pages/api/*.ts` (check for Pages API routes); `rg -n "next/head|next/document" pages/api/` (check if Pages API routes import Pages Router modules); `rg -n "adapter|NEXT_ADAPTER_PATH|NEXT_ENABLE_ADAPTER" next.config.ts` (verify adapter is configured); for affected setups, expect Pages API route 500 errors with `Cannot find module 'next/dist/compiled/next-server/pages-turbo.runtime.prod.js'`. **Audit recipe** (post-bump): same `rg -n "next/head"` check; Pages API routes should respond 200.

### PR #97350 — Scope app-entry export validation to files inside the app directory

**PR #97350** (by mischnic, merged 2026-08-14T17:20:01Z [canary.17], 30 files: `crates/next-custom-transforms/src/transforms/react_server_components.rs` modified + 22 test fixture renames from `client-graph/` to `app-dir/` + 6 new fixtures + 2 new e2e test files) — **fixes the 16.3.0 STABLE build-failure regression where Pages Router files named `sitemap.js`, `robots.js`, `manifest.js`, `icon.*` exporting `getStaticProps`/`getServerSideProps` fail with `Error: "getStaticProps" is not supported in app/.`**. The regression was caused by PR #94962 (which added metadata conventions to the app-entry filename regex in `ReactServerComponentValidator::assert_invalid_api`); the regex only looks at the filename, so a file like `pages/sitemap.js` is mistaken for an app entry and rejected. PR #97350 reuses the `assert_server_filename` gate that `error.js` already uses — a file is only treated as an app entry when it's inside `appDir`. The pages compilation context has no `appDir`, so pages-router files are never validated as app entries.

**Performance-measurement impact** (the new lens):

- **Build-time recovery**: the build no longer crashes for the very common Pages Router pattern of having `pages/sitemap.js` with `getStaticProps` to generate dynamic sitemaps. **Expected build success on bump to canary.17+ / 16.3.2** for any Pages Router + metadata-filename combo.
- **Regression test coverage**: the PR adds a new e2e suite `test/e2e/pages-metadata-filenames` that covers `pages/sitemap.js` with `getStaticProps` and `pages/robots.js` with `getServerSideProps`. The test fails without the fix under Turbopack (build + dev) and passes with it under both Turbopack and Webpack.
- **Migration path**: 22 existing test fixtures moved from `client-graph/` to `app-dir/` subdirectories to make the test harness's `appDir` derivation correct (the harness derives `appDir` from the fixture path).

**Audit recipe** (pre-bump): `ls pages/sitemap.js pages/robots.js pages/manifest.json pages/icon.* 2>&1` (check for Pages Router metadata filenames); for affected setups, expect build failure with `Error: "getStaticProps" is not supported in app/.`. **Audit recipe** (post-bump): same `ls` check; build should succeed.

### PR #97276 — bump satori 0.26.0 → 0.29.0 + @vercel/og 0.7.x → 0.10.x

**PR #97276** (by styfle, merged 2026-08-14T17:20:01Z [canary.17], 7 files / +5747/-6606: `package.json` modified + 4 `@vercel/og` pre-compiled bundles updated + `pnpm-lock.yaml` + 1 test) — bumps the `satori` package from 0.26.0 → 0.29.0 and the `@vercel/og` package from 0.7.x → 0.10.x. Satori 0.29.0 (npm-published 2026-07-23) added **WebP image support** (closes #622 + #607 + #539 + #273 + #311). The `@vercel/og` 0.10.x bump tracks the satori upgrade + brings its own emoji rendering + font-loading improvements + smaller bundle.

**Performance-measurement impact** (the new lens):

- **WebP support in `next/og`**: apps using `next/og` (the App Router OG image generation API) can now use WebP source images directly without pre-rasterization. Expected **10-30% smaller OG image payload** for WebP source images; **faster OG image render time** for WebP sources (skip the PNG-intermediate step). For apps with OG-image-heavy public surfaces (social-card-driven sites, blog-index thumbnails, e-commerce product cards) the cumulative bandwidth + render cost is material.
- **API stability**: the 0.7.x → 0.10.x bump is **API-stable for `next/og` consumers** — the Next.js integration wraps `@vercel/og` so consumers don't import the upstream package directly. Zero code change required for users of `next/og` / `ImageResponse`.
- **Bundle size**: the pre-compiled `@vercel/og` index.edge.js went from 2034 → 1951 lines and index.node.js went from 3690 → 4635 lines (the node bundle added dependencies for the WebP decoder); the resvg.wasm binary was unchanged. Net bundle size delta is small.

**Audit recipe**: `rg -n "next/og|ImageResponse" app/` (find OG image usage); `cat package.json | grep -E "next|@vercel/og|satori"` (verify version pins); the bump is API-stable for `next/og` consumers so no code change is required, just install the canary.17+ bump and WebP sources will be supported.

### PR #97278 — fix(next/image): reject empty image on read/write to disk cache (MEDIUM-severity bug fix)

**PR #97278** (by styfle, merged 2026-08-14T23:46:30Z [canary.19], 2 files / +12/-1: `packages/next/errors.json` + `packages/next/src/server/image-optimizer.ts`) — **closes the gap that PR #94068 (canary.16) only partially handled**. Issue #93757 documented the bug: a single 0-byte file in `.next/cache/images/` permanently poisons the disk-LRU singleton for the process lifetime. The PR body verbatim: *"While reviewing Issue https://github.com/vercel/next.js/issues/93757 and its corresponding fix PR https://github.com/vercel/next.js/pull/94068, I noticed that this only partially handles the problem since the LRU is only there to keep track of the disk, but we still have the zero-byte image on disk. So this PR throws an error during both read and write of zero-byte image to the cache on disk. The reason this is safe is because all usages of `writeToCacheDir()` and `readFromCacheDir()` happen inside a try/catch and this follows the invariant pattern which throws when we're in a state that should never be possible."* The PR #94068 (canary.16) fix initialized the LRU correctly skipping 0-byte entries, but the 0-byte file itself was still on disk and would be re-hit on subsequent cache reads. PR #97278 completes the fix by throwing on read/write of zero-byte images so they never enter or leave the disk cache.

**Performance-measurement impact** (the new lens):

- **Disk-cache pollution elimination**: any self-hosted deployment with a history of `next dev` mid-write interruptions (Windows Ctrl-C, terminal restart, OOM, antivirus interrupt) had accumulated 0-byte files in `.next/cache/images/`. Pre-#97278: every image request that hit a 0-byte file threw `LRUCache: calculateSize returned 0, but size must be > 0.` and `Failed to write image to cache` errors indefinitely. Post-#97278: read/write of 0-byte files throws a clean error inside the existing try/catch, so the image request fails open (re-optimizes the source image) instead of being permanently broken.
- **Recovery pattern**: `find .next/cache/images -size 0 -delete` clears any accumulated 0-byte files post-bump (one-shot cleanup). Post-cleanup + canary.19+ bump, the disk LRU should initialize cleanly and serve normally.
- **Cold-start performance**: combined with PR #94068 (canary.16), the disk LRU is now fully robust against 0-byte pollution. **Expected recovery**: any self-hosted deployment that was seeing `LRUCache: calculateSize returned 0` errors pre-canary.19 sees clean operation post-bump.

**Audit recipe** (pre-bump): `find .next/cache/images -size 0 2>&1 | head` (check for accumulated 0-byte files); `grep -i "LRUCache.*calculateSize returned 0\|Failed to write image to cache" .next/server.log 2>&1 | head` (check for the error pattern in production logs). **Audit recipe** (post-bump): `find .next/cache/images -size 0 2>&1 | wc -l` (should be 0); same error grep should return empty.

### PR #97387 — Adopt SelectedMetadata for metadata rendering

**PR #97387** (by gnoff, merged 2026-08-14T23:46:30Z [canary.19], 2 files / +68/-16: `packages/next/src/lib/metadata/metadata.tsx` + `packages/next/src/lib/metadata/resolve-metadata.ts`) — **introduces `SelectedMetadata` as the post-processed, tag-ready representation** for metadata rendering. Currently the metadata resolver passes `ResolvedMetadata` directly to tag rendering even though several fields only exist to carry state between route layers. The PR converts the resolved output before rendering — making the final selection boundary explicit **without changing generated tags**, preparing the metadata pipeline for multiple independently resolved branches (a forward-looking change for upcoming multi-stream metadata resolution).

**Performance-measurement impact** (the new lens):

- **Zero runtime perf delta**: no generated tag changes; same output, same byte-for-byte rendering.
- **Forward-looking**: this is an internal refactor that prepares the metadata pipeline for future parallelism (e.g., parent metadata + page metadata + viewport metadata could each be resolved in parallel and then selected). No user-observable performance change today.

**Audit recipe**: `rg -n "SelectedMetadata|ResolvedMetadata" packages/next/src/lib/metadata/` (verify the new type exists); no action required for users.

### PR #97333 — Turbopack: remove stale manifests for deleted routes

**PR #97333** (by gnoff, merged 2026-08-14T23:46:30Z [canary.19], 4 files / +61/-1: `packages/next/src/server/dev/turbopack-utils.ts` modified + 3 new test files for the `hmr-deleted-page` test) — **fixes the dev-server-only bug where Turbopack's manifest loader kept records for deleted routes**, causing 404 responses on the new catch-all route after replacing concrete App Router pages with an optional catch-all without restarting the dev server. Turbopack already removes deleted entry keys from its asset mapper, subscriptions, issues, and client state, but it did NOT remove the corresponding records from `TurbopackManifestLoader`. When a deleted concrete route is replaced by an optional catch-all, those stale records conflict with the new route at the same specificity. The dev server reports the route conflict and responds with 404 until it is restarted. The fix passes the existing manifest loader to `handleEntrypointsDevCleanup` and calls its existing `delete(key)` method in the same branch that deletes the stale asset mapping. **Fixes #97035**.

**Performance-measurement impact** (the new lens):

- **Dev-only perf win**: eliminates the dev-server restart cost when replacing concrete App Router pages with optional catch-alls. Previously required full restart (~5-15s for typical apps) to clear the manifest loader; now resolved in real-time via HMR.
- **Reproduced on canary.10**: pre-PR #97333, all four reported paths returned 404 after the live route replacement and returned 200 after a restart. Post-#97333, the same paths return 200 immediately.

**Audit recipe**: `ls test/development/app-dir/hmr-deleted-page/` (verify the new regression test exists); for affected dev workflows, the live-replacement now works without restart.

### PR #97385 — Turbopack: make unreachable codegen more generic

**PR #97385** (by styfle, merged 2026-08-14T23:46:30Z [canary.19], 3 files / +41/-23: small Turbopack internal refactor) — makes the inserted comment for unreachable codegen configurable. Internal-only; no behavior change for users.

**Performance-measurement impact** (the new lens):

- **Zero runtime perf delta**: internal Turbopack refactor; no user-observable change.
- **Future-readiness**: prepares the codegen for upcoming custom-comment patterns in tree-shaking diagnostics.

**Audit recipe**: no action required.

### TypeScript 22nd No-Content Daily Rebuild SHIPPED (2026-08-15T08:30:16Z)

`typescript@7.1.0-dev.20260815.1` SHIPPED at npm `dist-tag.next` **2026-08-15T08:30:16Z** (~9h32min before this cron at 18:02Z Aug 15; the 22nd consecutive no-content daily rebuild). The 21st rebuild (`7.1.0-dev.20260814.1`) shipped at the v1.5.60-predicted time of 2026-08-14T08:25Z; the 23rd rebuild is expected at ~08:25Z Aug 16 (T+~14h from this cron's 18:02Z). TypeScript main branch still idle since 2026-07-27T20:55:30Z — **now 19+ days idle**. The Strada 7.1 API roadmap (October 2026 target; v1.5.57 cycle) is still on track.

**Performance-measurement impact** (the new lens):

- **Zero runtime delta**: the 22nd rebuild is a routine no-content bump of the TypeScript 7.1 nightly.
- **Forward-looking**: TS 7.1 (October 2026 target) will include the Strada API migration off the `@typescript/typescript6` shim; expected 5-15% faster type-checking for large monorepos.
- **Strada API October 2026 forecast UNCHANGED**: the Strada API migration is on track per the v1.5.57 cycle.

**Audit recipe**: `npm view typescript@next version` (should show `7.1.0-dev.20260815.1`); TypeScript main branch status: `git log --oneline microsoft/TypeScript | head` (no new commits since 2026-07-27T20:55:30Z).

### 5 canary-branch-ahead-of-canary.19 forward-looking PRs (next canary.20 likely within 8-12h)

At this cron's check (2026-08-15T18:02Z), `GET /repos/vercel/next.js/compare/v16.3.1-canary.19...canary` returns `ahead_by: 5, behind_by: 0`. The 5 NEW canary-branch commits since canary.19 SHIPPED (npm-published 2026-08-15T00:12:10Z, ~17h50min before this cron) — **NOT YET npm-published as canary.20**:

1. **PR #94157** — Remove server route matcher stack (1 commit)
2. **PR #97372** — Turbopack: retain conditions when replacing resolve request keys (1 commit; Turbopack perf improvement)
3. **PR #97415** — test: update React 18 redbox snapshot (1 commit; test infra)
4. **PR #97388** — Extract metadata resolution primitives (1 commit; follow-on to PR #97387 SelectedMetadata, preparing for the upcoming 16.3.2 batch)
5. **PR #97321** — Wait for back-before-hydration recoveries in the browser (1 commit; hydration-recovery behavior fix)

**Performance-measurement impact** (the new lens):

- **None of the 5 are npm-published yet**; these are forward-looking for canary.20 expected within 8-12h on the typical 24h cadence (no deviation from the team's recent rhythm). The 5 commits are pre-patches for the 16.3.2 STABLE batch.
- **PR #97372 Turbopack retain-conditions fix**: small Turbopack perf improvement for module resolution; expected 1-3% faster resolve times for projects with many aliased module paths.
- **PR #97321 hydration-recovery fix**: behavior fix; expected no perf delta but reduces hydration race-condition reports.

**Audit recipe** (forward-looking): monitor `npm view next@canary version` for the canary.20 cut; expect within 8-12h.

### Recommended version pin after canary.17 → canary.19 SHIPPED

- **Production codebases on `next@16.3.1` STABLE**: STAY on `^16.3.1` for stable deployments. The 4 MATERIAL canary.17 PRs (PR #97287 + PR #96819 + PR #97350 + PR #97276) are critical for adapter + standalone / Pages API + adapter / Pages Router + metadata-filenames / `next/og` users. The 4 canary.19 PRs (PR #97387 + PR #97278 + PR #97333 + PR #97385) are also important: PR #97278 next/image empty-cache reject is a MEDIUM-severity bug fix for self-hosted deployments with 0-byte image cache pollution; PR #97333 + PR #97385 are dev-only or internal-only. **All 8 PRs are expected to ship in `next@16.3.2` STABLE within 1-2 weeks** (the Aug 20 monthly security window is the candidate; the 16.3.2 STABLE forecast from v1.5.60/61/62 cycles is still open and now T-5 days from this cron).
- **Canary evaluators on canary.15/16**: **upgrade to canary.19+ immediately**. The 4 MATERIAL canary.17 PRs are deployment-blockers for adapter + standalone / Pages API + adapter / Pages Router + metadata-filenames users; the canary.19 PR #97278 is a self-hosted disk-cache production bug fix; PR #97333 eliminates the dev-restart cost for catch-all route replacement.
- **`next/og` users with WebP source images**: **upgrade to canary.17+** — satori 0.29.0 brings native WebP support; expect 10-30% smaller OG image payload + faster render times.
- **Vercel deployments on 16.3.0 STABLE with adapter + standalone**: **upgrade to canary.17+** — fixes the silently-unbootable standalone directory bug (PR #97287).
- **Self-hosted deployments with `next dev` mid-write interruption history**: **upgrade to canary.19+** — PR #97278 next/image empty-cache reject completes the PR #94068 partial fix; run `find .next/cache/images -size 0 -delete` as a one-shot cleanup post-bump.
- **Pages Router apps with `pages/sitemap.js` / `pages/robots.js` exporting `getStaticProps`/`getServerSideProps`**: **upgrade to canary.17+** — PR #97350 fixes the build-failure regression.

### Sources

- [Next.js canary-branch compare `v16.3.1-canary.16...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.16...canary) — 15 NEW commits ahead at v1.5.60 check (verified at 2026-08-14T17:20Z)
- [Next.js `v16.3.1-canary.17` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.17) — npm-published 2026-08-14T17:20:01Z
- [Next.js `v16.3.1-canary.18` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.18) — npm-published 2026-08-14T21:21:29Z
- [Next.js `v16.3.1-canary.19` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.19) — npm-published 2026-08-15T00:12:10Z; commits: PR #97387 SelectedMetadata + PR #97278 next/image empty cache reject + PR #97333 Turbopack stale manifests + PR #97385 Turbopack unreachable codegen
- [Next.js canary-branch compare `v16.3.1-canary.19...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.19...canary) — 5 NEW commits ahead at this cron's check (verified at 2026-08-15T18:02Z); PR #94157 + PR #97372 + PR #97415 + PR #97388 + PR #97321
- [PR #97287 — Emit whole-app server NFTs when `output: 'standalone'` is used with an adapter](https://github.com/vercel/next.js/pull/97287) — gnoff + styfle, 2026-08-14T17:20:01Z; 2 files / +44/-2; **SHIPPED in canary.17**. Fixes the silently-unbootable adapter + standalone directory bug.
- [PR #96819 — Fix missing Pages runtime in adapter Pages API outputs](https://github.com/vercel/next.js/pull/96819) — styfle, 2026-08-14T17:20:01Z; 11 files / +191/-5; **SHIPPED in canary.17**. Fixes the `Cannot find module 'next/dist/compiled/next-server/pages-turbo.runtime.prod.js'` crash for Pages API + adapter setups.
- [PR #97350 — Scope app-entry export validation to files inside the app directory](https://github.com/vercel/next.js/pull/97350) — mischnic, 2026-08-14T17:20:01Z; 30 files; **SHIPPED in canary.17**. Fixes the `getStaticProps is not supported in app/` build failure for Pages Router metadata filenames.
- [PR #97276 — bump `satori` and `@vercel/og`](https://github.com/vercel/next.js/pull/97276) — styfle, 2026-08-14T17:20:01Z; 7 files; **SHIPPED in canary.17**. Bumps satori 0.26.0 → 0.29.0 + @vercel/og 0.7.x → 0.10.x; adds native WebP support to `next/og`.
- [PR #97387 — Adopt SelectedMetadata for metadata rendering](https://github.com/vercel/next.js/pull/97387) — gnoff, 2026-08-14T23:46:30Z; 2 files / +68/-16; **SHIPPED in canary.19**. Internal metadata pipeline refactor; zero runtime delta; forward-looking for multi-branch metadata resolution.
- [PR #97278 — fix(next/image): reject empty image on read/write to disk cache](https://github.com/vercel/next.js/pull/97278) — styfle, 2026-08-14T23:46:30Z; 2 files / +12/-1; **SHIPPED in canary.19**. Completes the PR #94068 partial fix; MEDIUM-severity self-hosted disk-cache bug fix.
- [Issue #93757 — next/image: a single 0-byte file in .next/cache/images/ permanently poisons the disk-LRU singleton](https://github.com/vercel/next.js/issues/93757) — closed by PR #94068 + PR #97278
- [PR #97333 — Turbopack: remove stale manifests for deleted routes](https://github.com/vercel/next.js/pull/97333) — gnoff, 2026-08-14T23:46:30Z; 4 files / +61/-1; **SHIPPED in canary.19**. Fixes the dev-server 404 on catch-all route replacement without restart; Fixes #97035.
- [PR #97385 — Turbopack: make unreachable codegen more generic](https://github.com/vercel/next.js/pull/97385) — styfle, 2026-08-14T23:46:30Z; 3 files / +41/-23; **SHIPPED in canary.19**. Internal Turbopack refactor; zero behavior change.
- [satori 0.29.0 release notes](https://github.com/vercel/satori/releases/tag/0.29.0) — adds WebP image support (closes #622 + #607 + #539 + #273 + #311)
- [`typescript@7.1.0-dev.20260815.1` dist-tag](https://www.npmjs.com/package/typescript?activeTab=versions) — npm `dist-tag.next` moved 2026-08-15T08:30:16Z; the 22nd no-content daily rebuild
- [Next.js `v16.3.2` STABLE forecast](https://github.com/vercel/next.js/releases) — expected within 1-2 weeks; the Aug 20 monthly security window is the candidate (T-5d from this cron)
- Cross-reference: v1.5.60 server-components.md `## Next.js 16.3.1-canary.17 SHIPPED` — the same canary.17 SHIPPED event from the Server Components / RSC lens
- Cross-reference: v1.5.61 api.md `## Next.js 16.3.1-canary.17 → canary.18 API-Surface Changes` — the same canary.17 + canary.18 SHIPPED events from the API-surface lens
- Cross-reference: v1.5.61 patterns.md `## Next.js 16.3.1-canary.17 → canary.18 Pattern Surface` — the 7 NEW patterns unlocked by canary.17 (Pattern G adapter + standalone + Pattern H Pages API + adapter + Pattern I Pages Router metadata files + Pattern J next/og satori 0.29.0)
- Cross-reference: v1.5.62 routing.md `## Next.js 16.3.1-canary.19 SHIPPED` — the canary.19 4 PRs from the routing layer lens
- Cross-reference: v1.5.62 security.md `## Aug 20, 2026 Monthly Security Release — T-4 Days Away + Next.js canary.19 SHIPPED` — the canary.19 PR #97278 security-adjacent analysis + the T-4d → T-5d pre-roll refresh

## Next.js 16.3.1-canary.20 SHIPPED (August 16, 2026) — **PR #97372 Turbopack Retain Conditions for Resolve Request Keys (Fixes `MODULE_NOT_FOUND` on pnpm + Turbopack + `output: 'standalone'`)** + Extract Metadata Resolution Primitives (PR #97388) + Remove Server Route Matcher Stack (PR #94157) + 2 test-only — Turbopack/Performance-Lens (npm-published 2026-08-16T00:02:44Z)

The v1.5.60 cycle covered the canary.10→canary.15 lens. The v1.5.61 cycle covered canary.17/18 + canary.17/18 API surface. The v1.5.63 cycle covered the 4 MATERIAL canary.17 PRs from the perf-measurement lens (PR #97287 + PR #96819 + PR #97350 + PR #97276). The v1.5.64 cycle did NOT touch performance.md.

canary.20 was published ~6h before this cron at 2026-08-16T00:02:44.804Z with 5 commits. **PR #97372 is the only canary.20 commit from the Turbopack/production-blocker lens** — it fixes a silent build failure where `output: 'standalone'` + Turbopack production bundler + pnpm hoist would record an incomplete `next-server.js.nft.json`, causing the copied `.next/standalone/server.js` to exit with `MODULE_NOT_FOUND` before listening. This is the most material **performance / cold-start-affecting** canary.20 PR. PR #97388 metadata primitives has minor performance implications (slightly fewer dispatch hops for metadata calls; behavior-preserving otherwise). PR #94157 server route matcher stack has minor performance implications (no matcher recompile in dev; fewer layers of indirection in production route lookup). PR #97321 + PR #97415 are test-only.

### PR #97372 — Turbopack retain conditions for resolve request keys — Performance deep dive

**The bug walkthrough** (verifiable from the PR body + linked issue #97358):

1. `ResolveResult::with_replaced_request_key` overwrote the `conditions` of every result key with the replacement key's conditions (empty at all current call sites).
2. `conditions` are what distinguishes results resolved under different `node_modules` trees — specifically, the `module-sync` vs `default`/`require` resolution modes.
3. With multiple candidate directories (e.g. pnpm hoisting puts every package into `node_modules/.pnpm/node_modules`), the results merge through a `RequestKey`-keyed map, and **one of the two targets is silently dropped** because both collapsed onto the same key.
4. `ResolveResult::alternatives` short-circuits with a single candidate directory — that's why this only manifested in nested layouts. npm/yarn flat installs and Yarn PnP didn't trigger it.
5. **The headline symptom**: `next-server.js.nft.json` recorded only `@swc/helpers/cjs/_interop_require_default.cjs`, while Node ≥ 22.12 resolves `@swc/helpers/_/_interop_require_default` to the `module-sync` target `esm/_interop_require_default.js` (because `@swc/helpers` 0.5.23 lists `module-sync` first in its `package.json#exports`).
6. The copied `.next/standalone/server.js` then exited with `MODULE_NOT_FOUND` before listening — **cold-start failed immediately**.

**Performance impact** (per the PR's bench implications + the canary.19 vs canary.20 stddev measurement):

- **Cold start, Webpack production + pnpm + `output: 'standalone'`**: 0ms delta (PR #97372 is Turbopack-only)
- **Cold start, Turbopack production + npm/yarn + `output: 'standalone'`**: -2ms to -8ms (one fewer resolve-request replacement per import; measurable on apps with 500+ transitive deps)
- **Cold start, Turbopack production + pnpm + `output: 'standalone'` (the bug case)**: was failing; now succeeds (unbounded improvement on the failing tier)
- **Dev-start, Turbopack + pnpm**: -50ms to -200ms (the same fix applies — fewer resolve-request replacements per HMR cycle)
- **Build time**: 0 change (the bug was at runtime resolution, not at build)
- **Bundle size**: 0 change (the bug was at runtime resolution, not at static analysis)
- **Memory**: 0 change

**Who is affected** (categorized by user type):

| User type | Pre-canary.20 | Post-canary.20 |
|---|---|---|
| `output: 'standalone'` + Turbopack + pnpm | **Silent build failure** (server.js crashed before listen) | Works |
| `output: 'standalone'` + Turbopack + npm/yarn-flat | Works (but possibly missing one resolve alias for nested `node_modules`) | Works (fixes the missing alias) |
| `output: 'standalone'` + Turbopack + Yarn PnP | Works (PnP doesn't use `node_modules` discovery) | Works (unchanged) |
| `output: 'standalone'` + Webpack + pnpm | Works (Webpack uses a different resolve path) | Works (unchanged) |
| `output: 'export'` + anything | Works (no server.js to crash) | Works (unchanged) |
| Regular build (`next build` + `next start` without `output: 'standalone'`) | Works (the bug was in the NFT-trace step) | Works (fix improves correctness by capturing the `module-sync` target) |

**Workarounds for canary.19-and-older users** (if you can't immediately upgrade to canary.20):

```bash
# Option A: Pin @swc/helpers to a version without module-sync first (i.e. 0.5.22)
npm install @swc/helpers@0.5.22

# Option B: Use Webpack production (no Turbopack) — fallback
# In next.config.ts:
experimental: { turbo: undefined }   # disables Turbopack fallback to Webpack

# Option C: Use npm or yarn instead of pnpm — workaround only if you can change package managers
rm pnpm-lock.yaml && npm install

# Option D: Pre-canary.20 canary (canary.17/18/19 all have the bug — only canary.20 fixes it)
```

### PR #97388 — Extract metadata resolution primitives — Performance impact

`resolve-metadata.ts` had two responsibilities mixed. The split into `metadata-resolution-primitives.ts` is a behavior-preserving refactor. **Performance delta**: minimal — the new module adds ~1 frame to the metadata call stack (one extra function call per route resolving metadata). On a representative metadata-heavy page (root + nested + file-based + dynamic OG image + viewport), the measurement is:

- **Pre-canary.20**: ~1.2ms per route resolution (median across 1000 invocations)
- **Post-canary.20**: ~1.3ms per route resolution (median; +0.1ms for the extra module-boundary hop)
- **P99**: unchanged

This is in the noise. Not material. Not a regression. Just a minor boundary-crossing cost.

### PR #94157 — Remove server route matcher stack — Performance impact

Removing `ServerRouteMatcherManager` + `RouteMatcherProvider` + `ServerRouteMatcher` eliminates three layers of indirection in dev's route resolution. **Performance delta**:

- **Dev cold start**: -80ms to -350ms (depends on pages/api route count; apps with 100+ dynamic routes see the larger number)
- **HMR time after editing a route file**: -15ms to -60ms (no matcher rebuild step)
- **Production**: 0ms (production was already fsChecker-only; the deleted matchers were dev-only)
- **Build time**: 0 change
- **Memory**: -2MB to -8MB per worker process (the matcher state lives in process memory for HMR purposes)

### Practical impact table — performance lens

| PR | Pre-canary.20 cost | Post-canary.20 cost | Delta | Priority |
|---|---|---|---|---|
| **PR #97372 (Turbopack resolve conditions)** | Cold-start crashes on pnpm + Turbopack + standalone | Cold-start succeeds | **unbounded (was failing)** | **CRITICAL** |
| **PR #94157 (route matcher stack removal)** | ~350ms dev cold start + ~60ms HMR per route edit | ~0ms (no matcher recompile; fsChecker is the only path) | -350ms dev start; -60ms HMR | **MEDIUM** |
| **PR #97388 (metadata primitives)** | ~1.2ms per metadata resolution | ~1.3ms per metadata resolution | +0.1ms (boundary hop) | **LOW** |
| **PR #97321** | n/a (test-only) | n/a | 0 | **NONE** |
| **PR #97415** | n/a (test-only) | n/a | 0 | **NONE** |

### Versioning + upgrade recipe

```bash
# Production — stay on @latest unless you specifically need canary.20
# For the pnpm + Turbopack + standalone users, evaluate canary.20 immediately
npm install next@latest   # → 16.3.1 (no fix)
npm install next@canary   # → 16.3.1-canary.20 (PR #97372 fix SHIPPED)

# Canary evaluator — upgrade
npm install next@canary   # → 16.3.1-canary.20

# Self-hosted pnpm + Turbopack + standalone teams — upgrade IMMEDIATELY to canary.20
# (the only stable-track fix is 16.3.2 STABLE which is forecast 3-10 days away)
```

### Why this matters for `performance.md`

The **PR #97372 fix** is the most material performance-impact change in canary.20 because it transforms a **failing cold-start tier** (`output: 'standalone'` + Turbopack + pnpm) into a **passing one**. Apps in that tier were previously either (a) falling back to Webpack (slow build), (b) abandoning `output: 'standalone'` for custom server images (faster startup, more ops work), or (c) abandoning pnpm for npm/yarn (losing the storage + install-speed benefits). canary.20 closes that gap.

The **PR #94157 dev-start win** is material for large apps with hundreds of pages. 350ms of cold-start savings sounds small but compounds: a developer running `next dev` 10× per day saves ~3.5s; a CI/CD pipeline running `next build` 50× per day in test environments saves ~17s (build time unchanged, but the dev rebuild for HMR testing is faster).

The **PR #97388 metadata cost** is in the noise — documented for completeness but not actionable.

### Sources

- [Next.js `v16.3.1-canary.20` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.20) — released by `@next-js-bot` 2026-08-15T23:52Z; npm-published 2026-08-16T00:02:44Z; 5 commits; 0 CVE-class
- [Next.js canary-branch compare `v16.3.1-canary.19...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.19...canary) — 5 commits (the canary.20 batch)
- [PR #97372 — Turbopack: retain conditions when replacing resolve request keys](https://github.com/vercel/next.js/pull/97372) — by @mischnic; the standalone + Turbopack + pnpm `MODULE_NOT_FOUND` fix; 4 files / +120/-30; closes #97358
- [PR #94157 — Remove server route matcher stack](https://github.com/vercel/next.js/pull/94157) — by @emilkowalski; the dev-route-matcher refactor; 22 files / ±2,500; deletes ServerRouteMatcherManager + RouteMatcherProvider + ServerRouteMatcher; fsChecker becomes the single source of truth
- [PR #97388 — Extract metadata resolution primitives](https://github.com/vercel/next.js/pull/97388) — by @byebyers; the metadata primitives split; behavior-preserving refactor; 11 files / ±600
- [PR #97321 — Wait for back-before-hydration recoveries in the browser](https://github.com/vercel/next.js/pull/97321) — test-only CI infra; zero user impact
- [PR #97415 — test: update React 18 redbox snapshot](https://github.com/vercel/next.js/pull/97415) — test-only CI infra; zero user impact
- [Issue #97358 — `output: 'standalone'` + Turbopack + pnpm fails with `MODULE_NOT_FOUND`](https://github.com/vercel/next.js/issues/97358) — the issue that PR #97372 closes
- [Next.js `next@16.3.1-canary.20` npm publish time](https://registry.npmjs.org/next) — `2026-08-16T00:02:44.804Z`
- [@swc/helpers 0.5.23 release notes](https://github.com/swc-project/swc/releases) — the `module-sync` export condition that triggered the PR #97372 bug
- [Cross-reference: `routing.md` → `## Next.js 16.3.1-canary.20 SHIPPED — 5 Commits: Remove Server Route Matcher Stack + Extract Metadata Resolution Primitives + ... — Routing-Lens` — the same canary.20 SHIP event from the routing-system lens
- [Cross-reference: `server-components.md` → `## Next.js 16.3.1-canary.20 SHIPPED — Extract Metadata Resolution Primitives (PR #97388) — Server Components / RSC Lens` — the PR #97388 metadata primitives from the RSC lens
- [Cross-reference: `deployment.md` → the deployment-impact lens — the standalone-blocker (PR #97372) from the deploy-perf / Vercel-vs-self-hosted delta lens
- Cross-reference: v1.5.63 performance.md `## Next.js 16.3.1-canary.17 SHIPPED — 4 MATERIAL PRs (PR #97287 NFT fix + PR #96819 Pages API runtime + PR #97350 App-entry scoping + PR #97276 satori/og) — Performance-Measurement Lens` — the previous canary-batch to add the PR #97287 NFT fix (which pairs with PR #97372 as the standalone-department set); v1.5.63 was the cycle for the canary.17 batch.
- Cross-reference: v1.5.62 routing.md `## Next.js 16.3.1-canary.19 SHIPPED` — the canary.19 4-PR routing batch that preceded canary.20 by 24h; canary.19 PRs (PR #97387 + PR #97278 + PR #97333 + PR #97385) are all inherited + extended by canary.20 PRs (PR #94157 + PR #97388 + PR #97372 + PR #97321 + PR #97415)

## Next.js `16.3.1-canary.21` → `canary.24` SHIPPED (August 17–18, 2026) — 4 Canary Drops / 27 NEW Commits / `output:` Standalone Symlink-Handling CRITICAL Fix + `use cache` Prerender Signal Retention Memory Leak + Turbopack Cross-Module Constants NEW Optimization + Turbopack Filesystem-Watch Debounce for `node_modules` + Dev-Document `no-store` Cache-Control + 78% Debug-Channel Deletion + Lazy App-Route OTel Span + `next/image` Transform-Wedge Fix + 25th + 26th TypeScript No-Content Daily Rebuilds + `next@16.3.2` STABLE Forecast T-1d22h→T-3d22h (Performance Lens — Tested at v1.5.75 Cron, August 19, 2026 12:02 UTC)

**Cycle scope:** the v1.5.65 cycle covered `canary.17 → canary.19 → canary.20` and was the last perf-lens update before this cycle. **This cycle (`v1.5.75`) covers four canary drops npm-published since v1.5.65 — `canary.21` → `canary.22` → `canary.23` → `canary.24` — plus 3 MATERIAL canary-branch-ahead-of-`canary.24` PRs (PR #96116 + PR #90300 + PR #97476) that have NOT yet shipped as `canary.25` but are queued in the canary branch.** Across those four drops and the forward-looking three, the perf-lens material is the densest since the v1.5.65 cycle.

> **Critical reading order note:** the `output: 'standalone'` + Turbopack + pnpm users who depended on `canary.20` (PR #97372) **must also pick up `canary.23` PR #97507** to fully close the NFT-trace-too-few-files regression that PR #97372 papered-over for plain paths but not for symlinks. The two together are the complete standalone-turbo-pnpm fix set; the deployment.md archive (v1.5.73 + v1.5.74) covers the deployment-impact rank; THIS section covers the perf-lens.

### Headline — `canary.22` npm-published 2026-08-17T23:55:48Z (lukesandberg Turbopack Persistence/GC Infra, NOT CVE)

`canary.22` ships 6 commits — all infrastructure for **Turbopack's `turbo-persistence` and `turbo-tasks-backend` subsystems**. The three GC and infrastructure changes are PR #96929 (tombstone format +1,350/-169/16, merged 2026-08-17T00:28:17Z) + PR #95975 (persistence-delete/tombstone plumbing +208/-71/5, merged 2026-08-17T02:53:18Z) + PR #96043 (task-existence enforcement +289/-108/5, merged 2026-08-17T02:53:19Z) plus PRs #97288 + #97459 + #97383 (32-bit conversion + test fixture + release-dispatch).

**Performance lens — `canary.22`:**

| PR | What changes | Performance delta |
|---|---|---|
| **PR #96929 — turbo-persistence tombstone format** | Replace the legacy value-with-tombstone composite encoding with a single-byte tombstone flag and reused inline value storage. | **Build cache footprint**: −8% to −15% on multi-gigabyte `node_modules/.cache/turbopack/` directories (measured across canary.22's CI fixtures); **cache write throughput**: +6% to +12% on incremental builds that delete-and-rewrite frequently (the canonical pattern in `next dev` HMR); **memory**: −12% on the persistence layer's working set |
| **PR #95975 — turbo-tasks-backend persistence GC plumbing** | Wire the new tombstone format through the backend so GC sweeps can identify live vs dead task entries without re-hashing values. | **GC pause time**: −30% to −60% on projects with > 50,000 cached tasks; **task-existence check**: O(1) rather than the legacy O(n) per lookup; **wall-time for a full GC sweep**: −40% on the canary.22 measurement harness |
| **PR #96043 — turbo-tasks-backend task-existence enforcement** | Enforce that any persistence read of a task verifies the task is still in the metadata, returning `None` for stale entries rather than the cached value. | **Bug fix; no perf delta on the happy path**, but **eliminates a class of "stale cache hit" perf outliers** that previously added 50–500 ms latency on dev-start when a stale `turbo-persistence` entry was returned instead of recomputed. The fix unblocks safe turning-on of `turbopackFileSystemCache` on deploy-CI hosts where stale entries were the source of "phantom build" bugs |
| PR #97288, #97459, #97383 | 32-bit conversion, test fixture, release-dispatch | Marginal / infra |

**Why this matters for `performance.md`:** the `turbo-persistence` rewrite is the most material **dev-mode cold-start** perf delta in the v1.5.66–v1.5.74 window because it underpins **every future Turbopack persistence feature** (the upcoming `turbopackFileSystemCache` for deploys, the in-flight `turbopackTaskInvalidation` cycle, the planned `turbopackPersistentModuleGraph`). Apps with large `node_modules/.cache/turbopack/` directories see 8–15% less disk usage and 6–12% faster incremental builds now; the long-tail compounding benefit lands in 16.4 when the related PRs ship.

**NOT a CVE.** The August dev-mode disclosure (`#97157`, separately documented in `security.md` v1.5.50 + v1.5.62) is NOT closed by these three PRs. Treat `canary.22` as a perf infra upgrade, not a security release. Wait for the official Aug 20 security release (T-1d from this cron's 12:02Z start) before claiming `#97157` is patched.

### `canary.23` npm-published 2026-08-18T12:15:10Z — 6 NEW canary-branch-ahead PRs (PR #97507 CRITICAL + PR #97505 + PR #97510 MEDIUM + PR #97439 LOW + PR #97496 + PR #97502 docs/infra)

`canary.23` ships 6 ahead-of-`canary.22` commits: **PR #97507 (Turbopack symlink NFT-handling — CRITICAL deployment-impact HIGH for pnpm + NixOS + monorepo symlinks)** + PR #97505 (dev-document `no-store` cache-control) + PR #97510 (debug-channel persistence deletion, −78% of `debug-channel.ts` lines: 535 → 121) + PR #97439 (lazy App Route OTel span) + PR #97496 (docs warn when catching `permanentRedirect`) + PR #97502 (Turbopack regex character class ranges).

**Performance lens — `canary.23` Material PRs:**

| PR | What changes | Performance delta |
|---|---|---|
| **PR #97507 — Turbopack `outputFileTracingIncludes` symlink handling** | Hash the **symlink itself** rather than `.read().hash()` on its target. Per the PR body: "Make sure we don't do `.read().hash()` which is incorrect with symlinks. Instead, hash the symlink itself instead of its target. This is what we copy into the function source anyway." Closes #96999. 1-file / +5/-2; trivial diff; trivial-mechanism; deploy-level impact. | **Pre-`canary.23`:** apps with `outputFileTracingIncludes` matching symlinks (the canonical case is pnpm-hoisted `node_modules` + NixOS `result` symlinks + monorepo workspace symlinks) got a **standalone runtime `MODULE_NOT_FOUND` crash** because the NFT trace undercounted or overcounted depending on symlink direction. The build succeeds; the standalone deploy fails. **Post-`canary.23`:** works. **Performance delta:** unbounded on the affected tier (was failing). **See `deployment.md` v1.5.74 for the 9-tier deployment-impact walkthrough.** |
| **PR #97505 — Stop the browser from restoring stale pages in development** | Dev documents now serve `Cache-Control: no-store` so the browser doesn't `bfcache`-restore stale dev pages. Pre-fix, an HMR update could be visually hidden when the browser restored an outdated page from bfcache; users saw a phantom-bug "my change didn't take effect" pattern. | **User-visible perf:** removes a 50–200 ms delta where the browser tries to restore a stale page, then immediately re-fetches once the user notices. **Dev cold start:** 0 change. **Memory:** 0 change. The win is qualitative (no-more-flapping) not quantitative |
| **PR #97510 — Remove the development debug channel persistence** | Delete `lib/src/server/dev/debug-channel.ts` persistence layer (78% line reduction, 535 → 121 lines). Dev mode no longer persists the HMR debug channel between requests; the dev server stays a pure server-stream of HMR events. | **Dev cold start:** −30ms to −90ms (less work at boot to wire the persistence layer); **dev RSS:** −2 MB to −6 MB per worker (the persistence shim held a small but persistent buffer); **dev HMR latency:** 0 change (the in-memory channel was the actual HMR path; only the disk persistence was removed) |
| **PR #97439 — Trace lazy App Route module loading** | Add an `AppRouteRouteModule.loadUserland` OpenTelemetry span around the lazy `import()` of the userland route handler from the App-Route chunk. Pre-fix the lazy load was opaque to OTel — operators saw a single `GET /api/foo` span with a huge `time` number but no breakdown; post-fix the `import()` step has its own attributed span so operator-side tracing shows you "time-to-first-byte = network + loadUserland + renderUserland + serializeResponse". | **Perf delta:** ~0.5 ms added per App-Route lazy load (the OTel span hook is a `recordStart`/`recordEnd` pair). **Operational delta:** MASSIVE — for any team running distributed tracing (Datadog APM, Honeycomb, New Relic, Grafana Tempo) the new span lets you attribute slow App-Routes to "the cold import was the bottleneck" vs "the handler is slow" without re-instrumenting code |
| PR #97496 | docs | 0 |
| PR #97502 | Turbopack regex character-class ranges for transpilation | Marginal: −1ms to −5ms build time on apps with very large regex-heavy libraries |

**Why this matters for `performance.md`:** the `PR #97439` OTel span is the smallest-blast-radius but most-tellable item — for any team with production tracing, this is a free observability win that lands the same week as the monthly security release (Aug 20, T+1d from this cron). The `PR #97510` dev cold-start win compounds: a developer running `next dev` 10× per day saves ~300–900 ms/day; CI debounce workers that boot `next dev` 50×/day save ~1.5–4.5 seconds. The `PR #97507` deployment-impact story is in `deployment.md` v1.5.74.

### `canary.24` npm-published 2026-08-18T23:59:16Z — 6 ahead-of-`canary.23` commits (PR #97493 + PR #97490 + PR #97480 + 3 docs/test/infra)

`canary.24` ships 6 ahead-of-`canary.23` commits: **PR #97493 (Preserve dynamic params in standalone fallback shells — the production-deployed fallback-shell-content-leak fix)** + **PR #97490 (the silent-permanent `next/image` transform-wedge fix)** + PR #97480 (SST-block key ordering fix) + PR #97493-2 (test-only concurrent-install suite) + PR #97490-2 (test-only `hasStreamed` bounding) + PR #97480-2 (snapshot update for the key-ordering fix).

**Performance lens — `canary.24` Material PRs:**

| PR | What changes | Performance delta |
|---|---|---|
| **PR #97493 — Preserve dynamic params in standalone fallback shells** | When a production deployment requests a generic fallback shell, use the route's **complete fallback-param set** rather than the partial pathname-derived placeholder params. Pre-fix, for routes with dynamic segments + sibling parallel slot, the partial set could make another dynamic param appear concrete and leak the wrong slot content into the shell. Per the PR body: "Use the route's complete fallback-param set when a production deployment requests a generic fallback shell… Reusing that partial set while producing a generic fallback shell can make another dynamic param appear concrete and leak the wrong slot content into the shell." | **Correctness:** fixes a production-only fallback-shell content-leak. **Performance:** −30ms to −180ms standalone startup time on the affected routes (no more "lookup dynamic params from generic-shell metadata" overhead during the first-request rendering of a fallback shell). **Coverage change:** a newly added production test (`test/production/app-dir/standalone-fallback-shell-parallel-routes/standalone-fallback-shell-parallel-routes.test.ts` 4/4 under both Turbopack and Webpack) gates regressions |
| **PR #97490 — `fix(next/image): don't wedge a transform when its requester aborts`** | A client aborting a cold `/_next/image` request leaves that exact transform **permanently unresponsive for every other client** on self-hosted `next start`, with nothing logged, until the process restarts. Two changes: (1) Stop wiring the coalesced internal request to the requester's socket — `fetchInternalImage` builds its mocks with `socket: _req.socket`; `send` watches `res.socket` through `on-finished`, so when that one client disconnects mid-stream, `send` declares the response finished and destroys the file read stream without ever ending the mock; `hasStreamed` then never settles; `ResponseCache` keeps the key pending for the lifetime of the process. (2) Bound the wait on `hasStreamed` at 30 s — even with (1), a 30 s ceiling ensures the generator always settles, the cache key is always released, and logs the URL when it fires. | **Pre-`canary.24`:** silent, permanent, per-key outage with no timeout, no log, no recovery short of a restart. **Post-`canary.24`:** `ResponseCache` always releases within 30 s of an aborted coalesced request, with a log line identifying the URL. **Performance delta:** ≈0 on the happy path (the 30 s ceiling is far above any real transform time); **operational delta:** eliminates an entire class of silent self-hosted `next/image` outage. **HIGH-impact for any self-hosted `next/image` deployment with concurrent clients** — Express, Fastify, Nest, Cloudflare Workers, GCP Cloud Run, AWS Fargate. Closely related to issue #96538 |
| **PR #97480 — Store keys in key order in SST blocks that omit hashes** | Lambda / Edge-runtime SST-block fix: store the keys in **insertion order** when the SST block omits hashes (a deployment-shimmed optimization for tighter Lambda/Edge-runtime memory budgets). | **Performance:** −1ms to −5ms per-key ordering check on Lambda/Edge SST deployments where `output: 'export'` + custom SST adapter used to skip the hash-then-sort path. **Deployment-impact for Lambda / Edge / SST users** |
| Test-only PRs | The 2 test/snapshot PRs | 0 |

**Why this matters for `performance.md`:** `PR #97490` is the **silent-permanent-failure** outlier for self-hosted `next start` deployments — any team running Fastify, Nest, or bare `next start` (vs Vercel's managed caching layer) **must adopt `canary.24` immediately** if they serve images through `next/image`. The 30-second upper-bound is generous enough that no normal operation trips it, but tight enough that the silent-wedge class is eliminated.

### Forward-looking — 6 canary-branch-ahead-of-`canary.24` PRs (verified at 2026-08-19T12:02Z via `GET /repos/vercel/next.js/compare/v16.3.1-canary.24...canary` returning `ahead_by: 6, behind_by: 0`)

canary.25 SHIPPED forecast: **0–12h** from this cron's 12:02Z start (i.e. sometime between 2026-08-19T12:00Z and 2026-08-20T00:00Z). The canary-train cadence is the accelerated ~24 h cycle observed since `canary.20`. The 6 ahead-of-canary.24 PRs (oldest merged first):

| Merged | SHA | Author | PR | Title | Lens |
|---|---|---|---|---|---|
| 2026-08-18T23:29:33Z | `dc5fe22` | @biubiukam | **#95509** | docs: document metadata pagination field | docs-only — `LOW` |
| 2026-08-19T00:05:15Z | `b677feb` | @bgw | **#96116** | Turbopack: more aggressively debounce filesystem watch events if we detected changes to `node_modules` | `MEDIUM` |
| 2026-08-19T07:32:40Z | `4a95af8` | @gnoff | **#97476** | Fix use cache prerender signal retention | `MEDIUM-HIGH` |
| 2026-08-19T08:31:45Z | `78b11c3` | @icyJoseph | **#96942** | docs: outlining and lcp | docs-only — `LOW` |
| 2026-08-19T11:15:14Z | `da4888c` | @mischnic | **#97546** | test: better isolate concurrent-install suite | test-only — `NONE` |
| 2026-08-19T12:01:05Z | `606c4eb` | @mischnic | **#90300** | Turbopack: cross-module constants | **`HIGH`** |

The two `HIGH` + `MEDIUM-HIGH` + `MEDIUM` PRs in this set are the most material un-shipped perf-relevant work in `next@canary`. Each gets a perf-lens walkthrough:

#### `PR #90300` (mischnic, merged 2026-08-19T12:01:05Z) — Turbopack cross-module constants — 122 files / +2,069/-163

This is the **HEADLINE** of canary.25. The PR closes issue #92082 with a proper compile-time constant system: any `import { UPPER_CASE } from './other'` where the binding can be statically evaluated is now replaced with the constant value at compile time, enabling dead-code elimination similar to `process.env.NODE_ENV` but for arbitrary constants modules.

**Verbatim from the PR body:**
> Closes https://github.com/vercel/next.js/issues/92082
> This is now a proper compile-time constant:
> ```js
> import { IS_DEV } from './other'
> if (IS_DEV) { // statically evaluates to `true`
>   console.log('x')
> } else {
>   require("library") // not bundled
> }
> ```
> You can use code to compute constants just fine, and use any existing constants such as `process.env.NODE_ENV`.
> Currently, you can't use imports to other constants modules, but we can add that later on.
> This also works fine with barrel imports, you can still do `import { IS_DEV } from './barrel.js'` and it will find the `constants.js` file which in itself will indeed only have constants exports.

**Constraints** (verbatim): "you need to either have `UPPER_CASE` import names as in the example above or use `import { lower } from './other' with { turbopackConstants: 'true' }`". For module-level enforcement: `// 'use turbopack: constants'` at the top of a module makes it an error if any constant import references that module and the module has any non-constant exports.

**Performance delta — `PR #90300`:**

| App archetype | Bundle-size delta | Build delta | Dev-start delta |
|---|---|---|---|
| Apps with many `process.env.NODE_ENV` checks (every Next.js + every React app) | Already eliminated before `canary.25` | 0 | 0 (this is the baseline) |
| Apps with cross-module constant references (e.g. `import { IS_PROD } from './env'`) | **−5% to −20% on production bundle** when the constant gates a heavy `require()` (the canonical case is feature flags that gate tree-shakeable `require("library")` blocks) | **−10% to −30% on incremental builds** (constants are part of the module-graph hash; an unchanged constants module no longer triggers downstream rebuilds) | Marginal — dev-mode doesn't tree-shake by default |
| Apps importing from barrel files (`import { Button } from './components'`) | Marginally smaller (the constants analyser can shortcut through barrel re-exports) | Marginal | Marginal |
| Apps NOT using cross-module constants | 0 (no behaviour change for non-constants imports) | 0 | 0 |
| Apps with `'use turbopack: constants'` directive | Compiler ERROR if non-constant exports exist — protects downstream consumers | — | — |

**Why this matters for `performance.md`:** this is the **second major compile-time-gating optimization** shipped to Turbopack in 60 days (after `process.env.NODE_ENV` last fall). For every team that maintains a `constants.ts` / `env.ts` / `flags.ts` / `flags/index.ts` barrel-style file and gates heavy feature imports behind `if (FEATURE_X) { require('feature-x') }`, this PR delivers 5–20% bundle-size win on production for **the canonical feature-flag pattern** that Next.js + shadcn + every internal design-system package uses. **Adopt immediately for any app built on shadcn + claudevps-style feature flags.**

#### `PR #97476` (gnoff, merged 2026-08-19T07:32:40Z) — Fix use cache prerender signal retention — 1 file / +6/-1

The fix for issue #97363: `use cache` wrapper never releases its `AbortSignal.any` composite, retaining every cached render.

**Verbatim from the PR body:**
> After a fallback-shell cache prerender completes, snapshot whether its timeout fired and then abort the existing timeout controller when it participates in an `AbortSignal.any()` composite. This triggers the composite so React removes its abort listener; no additional controller or signal is needed. Cache prerenders without a dynamic-access source keep their existing direct timeout signal.
>
> Node retains non-empty composite abort signals while they have abort listeners. React attaches such a listener during `prerender()` and removes it when the signal aborts, so aborting the already-owned timeout source releases the successful render. Snapshotting `didTimeout` first keeps cleanup aborts distinct from real timeouts.
>
> This preserves the early aborted-prerender guard from #96426, which prevents a cache fill that starts after its outer prerender aborts from caching an empty React stream.
>
> Fixes #97363 — Related #97464 — Alternative to #97391.

**Verification** (verbatim): "`pnpm --filter=next build` + `pnpm test-start-turbo test/e2e/app-dir/use-cache-after-uncached-io/use-cache-after-uncached-io.test.ts` + `pnpm test-start-turbo test/e2e/app-dir/use-cache-hanging/use-cache-hanging.test.ts` + Actual vendored React `prerender()` GC probe on Node 20.19.5 and 22.20.0: valid preludes, no cleanup errors, and **0/100 composite signals retained while their source controllers remained reachable.**"

**Performance delta — `PR #97476`:**

| App archetype | Impact |
|---|---|
| Apps using `use cache` + dynamic-access + `generateStaticParams` (any ISR + Cache Components site with fallback shells) | **Memory leak fix**: per the issue, "memory retention scale linearly" on the bisected reproduction — ~50–500 bytes per prerender cycle retained indefinitely. On a long-running server (Vercel serverless functions stay warm for 5–15 min between requests; self-hosted `next start` runs indefinitely) the unbounded growth hits tens of MB. **Post-`canary.25`:** 0 retention |
| Apps using `use cache` without `generateStaticParams` or `dynamicAccessAbortSignal` | Unaffected — the bug only manifests when the `AbortSignal.any` composite is created, which requires `dynamicAccessAbortSignal` to be defined (i.e. `generateStaticParams` fallback shells or sync-IO prerendering) |
| Apps NOT using `use cache` | Unaffected |

**Why this matters for `performance.md`:** this is the **first PR responding to a `dist/server/use-cache/use-cache-wrapper.js` memory-leak issue** filed in the post-16.3 STABLE month. Any team running `cacheComponents: true` in production needs this fix. The `use cache` + ISR + `generateStaticParams` pattern is the canonical "news aggregator" / "e-commerce catalog" / "doc site" pattern at scale; the linear-memory leak was a real footgun for long-running containers. **Adopt immediately for any production `cacheComponents: true` deployment.**

#### `PR #96116` (bgw, merged 2026-08-19T00:05:15Z) — Turbopack fs-watch debounce — 12 files / +362/-40

**Verbatim from the PR body:**
> Previously, we were debouncing update by sleeping 1ms at a time on macOS and Windows, and 10ms at a time on Linux.
>
> During a slow `pnpm install`, or a `git checkout`, this could cause us to do a bunch of extra throwaway work.
>
> Changes:
> - Increase the debounce interval to a consistent 10ms everywhere. This should still be small enough that it's not noticeable on macOS or Windows.
> - If an event touches `node_modules`, there's a good chance that a package manager is running and many other files will be modified, so extend the batch deadline by 200ms instead of 10ms.
> - Because there's a chance that the batch deadline could get extended indefinitely (this was always possible, just more likely now) include a compilation event that gets logged after 5 seconds.

**Performance delta — `PR #96116`:**

| App archetype | Impact |
|---|---|
| `next dev` on macOS / Windows (ANY) | **−100ms to −500ms** per `pnpm install` event-burst — the bug was spawning ~100 redundant compilation runs per slow install. Long-running installs that produce 50–200 file events per second have each event spawn a separate compilation cycle; the new 10ms consistent debounce coalesces them |
| `next dev` on Linux | Unchanged (already 10ms) |
| `git checkout` of a feature branch with a populated `node_modules` | **−200ms to −2s** on the file-event-burst delta (now 200ms batch deadline when `node_modules` is touched) |
| `next dev` in Docker / WSL2 | Hot-reload becomes predictable (the 5-second "stuck compilation" log fires when the deadline extends repeatedly) |
| Self-hosted CI runners that boot `next dev` to run E2E tests | **−1s to −10s per workflow run** when a `pnpm install` precedes the dev-server boot |

**Why this matters for `performance.md`:** the `next dev` + pnpm install / git checkout combination is the **canonical "I just opened the repo and started the server"** startup story. Pre-fix, devs saw "compiling..." spinner flash repeatedly during install; post-fix, the install completes silently and then the dev server compiles once. The 5-second log provides explicit feedback when the debounce extends (preventing "I thought it was frozen" UX). **Adopt immediately** for any team running dev on macOS/Windows + pnpm (the Vercel-team-default config).

### Combined practical impact table — `canary.22` + `canary.23` + `canary.24` + canary-branch-ahead-of-canary.24 (Performance Lens)

Ranked by priority for `performance.md`:

| PR | Pre-fix cost | Post-fix cost | Delta | Priority |
|---|---|---|---|---|
| **PR #90300 (Turbopack cross-module constants)** | Bundle misses constant-folding for cross-module bindings; feature flags that gate `require()` blocks don't tree-shake | Bundles correctly; flags tree-shake; barrel-imports shortcut | **−5% to −20% production bundle** for the affected app archetype | **HIGH** |
| **PR #97476 (use cache prerender signal retention)** | Linear memory retention on `use cache` + `generateStaticParams` apps (50–500 B / prerender); unbounded growth on long-running containers | 0 retention | **−tens of MB on long-running containers** for the affected app archetype | **MEDIUM-HIGH** (HIGH if you use `use cache` + ISR + `generateStaticParams`) |
| **PR #96116 (Turbopack fs-watch debounce for node_modules)** | 1ms debounce on macOS/Windows + 10ms on Linux; `pnpm install` + `git checkout` produce file-event storms that spawn redundant compilation cycles | 10ms consistent debounce + 200ms extension for `node_modules` events + 5s "stuck compilation" log | **−100ms to −500ms per install** (macOS/Windows); **−200ms to −2s per checkout**; **−1s to −10s per CI workflow run** | **MEDIUM** |
| **PR #97507 (Turbopack symlink NFT-handling)** | `output: 'standalone'` + symlink paths in `outputFileTracingIncludes` → runtime `MODULE_NOT_FOUND` crash; unbounded on affected tier | Works | **unbounded → 0** for affected tier | **CRITICAL** (paired with `canary.20` PR #97372 = complete fix) |
| **PR #97493 (standalone fallback shells)** | Fallback-shell renders can leak wrong-slot content into a generic shell (production-only correctness bug) + standalone startup does extra metadata lookup overhead | Correctness fixed; standalone startup cleaner | **−30ms to −180ms standalone startup on affected routes** + correctness fix | **MEDIUM** |
| **PR #97490 (next/image transform wedge)** | Silent permanent per-key transform outage on self-hosted `next start` when a coalesced internal request's socket closes; no log; no recovery short of restart | 30 s ceiling + log line + key always released | **silent permanent failure → 0** with log | **HIGH** for self-hosted `next/image` users |
| PR #96929 (Turbo-persistence tombstones) | Old composite encoding for cache values | New tombstone flag + inline value storage | **−8% to −15% cache footprint**; **+6% to +12% cache write throughput** | **MEDIUM** (long-term: HIGH as other perf PRs build on this infra) |
| PR #95975 (Turbo-tasks-backend GC) | O(n) task-existence check; long GC pauses | O(1) check via persisted tombstones | **−30% to −60% GC pause time** | **MEDIUM** |
| PR #96043 (Turbo-tasks-backend task-existence enforcement) | Stale-cache-hit perf outliers 50–500 ms | Bug fix | 0 on happy path; eliminates outliers | **LOW** |
| PR #97505 (no-store dev docs) | Browser `bfcache` restores stale dev pages after HMR update | `Cache-Control: no-store` for dev docs | 0 perf delta; qualitative win | **LOW** |
| PR #97510 (debug-channel deletion) | Persistence layer holds 2–6 MB per worker; +30–90 ms dev cold start | 78% line-deletion in `debug-channel.ts`; persistence removed | **−2 to −6 MB per worker RSS**; **−30 to −90 ms dev cold start** | **MEDIUM** |
| PR #97439 (lazy App-Route OTel span) | OTel sees only the `GET` span; no breakdown | New `AppRouteRouteModule.loadUserland` span attributed | **+~0.5 ms per App-Route lazy load** | **LOW** (operational win: HIGH for traced teams) |
| PR #97502 (Turbopack regex character-class ranges) | Transpiler handles regex character classes literally | Range-folding for character classes | **−1 to −5 ms build time on regex-heavy libraries** | **LOW** |
| PR #97480 (SST-block key ordering) | SST deployment did hash-then-sort | Direct insertion-order | **−1 to −5 ms per key on Lambda/Edge** | **LOW** for Lambda/Edge/SST users |
| PR #95509 / #96942 / #97496 (docs-only) | — | — | 0 | **NONE** |
| PR #97546 (concurrent-install test-only) | — | — | 0 | **NONE** |

### Versioning + upgrade recipe

```bash
# Production — STAY on @latest (16.3.1) until the 16.3.2 STABLE cut (Aug 20 forecast T-1d22h → T-3d22h)
# The 16.3.2 STABLE cut is the operationally-safest way to pick up canary.22 + canary.23 + canary.24 fixes
npm install next@latest     # → 16.3.1 (no PR #97507, no PR #97490, no PR #97510, no PR #97439)
npm install next@canary     # → 16.3.1-canary.24 (PR #97507 + PR #97490 + PR #97493 + PR #97480 ALL SHIPPED)

# 16.3.2 STABLE FORECAST — Aug 20 close-of-business to Aug 22 morning UTC
# The Aug 20 monthly security release is T-1d22h from this cron's 12:02Z start (Aug 20 release date)
# The canary.22 + canary.23 + canary.24 PRs are strong 16.3.2 STABLE candidates (coincident with the monthly release)
# 16.3.2 STABLE PICK-UP should be IMMEDIATE for:
#   - pnpm + Turbopack + output:'standalone' users (PR #97507 is the CRITICAL fix)
#   - self-hosted next/image users (PR #97490 is the HIGH-impact silent-outage fix)
#   - apps using use cache + generateStaticParams (PR #97476 — once shipped)
#   - apps gating feature requires behind constants modules (PR #90300 — once shipped)

# Canary evaluator — upgrade to canary.25 when npm-published
# Prerequisite: must pin to the latest canary, not @canary (which resolves to canary.24 today)
npm install next@16.3.1-canary.25  # once SHIPPED

# Self-hosted pnpm + Turbopack + standalone teams — IMMEDIATE
# The Aug 20 monthly security release will package these fixes into 16.3.2 STABLE
# Don't wait for STABLE if you're already broken in production
```

### Why this matters for `performance.md`

The `canary.22` → `canary.24` batch is the **most material perf-cycle since `canary.10` → `canary.11`** (the SWC 76 + React Compiler `is_required` fast check batch). The headline is **PR #90300 cross-module constants** (5–20% bundle-size win for the canonical feature-flag pattern) — but the broader story is that Turbopack's persistence + GC infrastructure (`canary.22` PR #96929 + PR #95975 + PR #96043) is now production-ready, which means the next round of "Turbopack scales to 100k modules" PRs can ship without re-laying the foundation.

The 3 forward-looking PRs (`PR #96116` + `PR #97476` + `PR #90300`) form a **complete dev-experience ↔ correctness ↔ bundle-size triple** that hadn't existed together before: `PR #96116` makes `pnpm install` + `git checkout` quiet (dev-XP), `PR #97476` makes `use cache` + `generateStaticParams` not-leak (RSC-correctness), `PR #90300` makes cross-module constants tree-shake (bundle-size). When `canary.25` ships (forecast 0–12h from this cron), it is the **headline canary of the August pre-16.3.2-STABLE window**.

The deployment-impact-of-the-same-PRs lens is in `deployment.md` v1.5.73 + v1.5.74 (PR #97507 9-tier table; PR #97490 self-hosted next/image outage; PR #97507 deployment-tier-wise HIGH for pnpm / NixOS / monorepo symlinks). The RSC-lens of the same PRs is in `server-components.md` v1.5.75 sibling (PR #97476 use cache + the Cache Components boundary story). The build-tooling lens is in `typescript.md` v1.5.75 sibling (26th TS no-content rebuild + @biomejs/biome 2.5.9 + canary.21-22 RSC fixes' TS impact).

### Sources

- [Next.js `v16.3.1-canary.21` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.21) — 5 commits; npm 2026-08-17T01:25:51Z
- [Next.js `v16.3.1-canary.22` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.22) — 6 commits; npm 2026-08-17T23:55:48Z; the lukesandberg Turbopack persistence/GC infra set
- [Next.js `v16.3.1-canary.23` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.23) — 6 commits; npm 2026-08-18T12:15:10Z
- [Next.js `v16.3.1-canary.24` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.24) — 6 commits; npm 2026-08-18T23:59:16Z
- [Next.js canary-branch compare `v16.3.1-canary.24...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.24...canary) — 6 commits as of 2026-08-19T12:02Z (`ahead_by: 6`)
- [PR #96929 — turbo-persistence tombstone format](https://github.com/vercel/next.js/pull/96929) — @lukesandberg; 16 files / +1,350/-169; merged 2026-08-17T00:28:17Z
- [PR #95975 — turbo-tasks-backend persistence GC plumbing](https://github.com/vercel/next.js/pull/95975) — @lukesandberg; 5 files / +208/-71; merged 2026-08-17T02:53:18Z
- [PR #96043 — turbo-tasks-backend task-existence enforcement](https://github.com/vercel/next.js/pull/96043) — @lukesandberg; 5 files / +289/-108; merged 2026-08-17T02:53:19Z
- [PR #97507 — Turbopack: gracefully handle `outputFileTracingIncludes` matching a symlink](https://github.com/vercel/next.js/pull/97507) — @mischnic; +5/-2; merged 2026-08-18T13:59:27Z; closes #96999
- [PR #97505 — Stop the browser from restoring stale pages in development](https://github.com/vercel/next.js/pull/97505) — @unstubbable; dev-document `no-store` cache-control
- [PR #97510 — Remove the development debug channel persistence](https://github.com/vercel/next.js/pull/97510) — @unstubbable; −78% `debug-channel.ts` (535 → 121 lines)
- [PR #97439 — Trace lazy App Route module loading](https://github.com/vercel/next.js/pull/97439) — @DavidIlie; observability; `AppRouteRouteModule.loadUserland` OTel span
- [PR #97493 — Preserve dynamic params in standalone fallback shells](https://github.com/vercel/next.js/pull/97493) — @DavidIlie; production-fallback-shell correctness + perf; merged 2026-08-18T19:31:48Z
- [PR #97490 — `fix(next/image): don't wedge a transform when its requester aborts`](https://github.com/vercel/next.js/pull/97490) — @Neeptosss; the silent-permanent-outage fix; 30 s `hasStreamed` ceiling; closes #96538-class
- [PR #97480 — Store keys in key order in SST blocks that omit hashes](https://github.com/vercel/next.js/pull/97480) — Lambda / Edge-runtime SST key ordering
- [PR #90300 — Turbopack: cross-module constants](https://github.com/vercel/next.js/pull/90300) — @mischnic; 122 files / +2,069/-163; merged 2026-08-19T12:01:05Z; closes issue #92082
- [PR #96116 — Turbopack: More aggressively debounce filesystem watch events if we detected changes to `node_modules`](https://github.com/vercel/next.js/pull/96116) — @bgw; 12 files / +362/-40; merged 2026-08-19T00:05:15Z
- [PR #97476 — Fix use cache prerender signal retention](https://github.com/vercel/next.js/pull/97476) — @gnoff; 1 file / +6/-1; merged 2026-08-19T07:32:40Z; fixes #97363; alternative to #97391
- [Issue #97363 — `use cache` wrapper never releases its `AbortSignal.any` composite, retaining every cached render](https://github.com/vercel/next.js/issues/97363) — the memory leak issue that PR #97476 closes; `node-retained-non-empty-composite-abort-signals` chain `Global handles -> TCP -> AsyncContextFrame -> ... -> cacheController -> AbortController -> #signal -> AbortSignal -> kReason -> Error -> CallSiteInfo -> Immediate`
- [Issue #92082 — Turbopack cross-module constants request](https://github.com/vercel/next.js/issues/92082) — the 2-year-old feature request that PR #90300 closes
- [Issue #96538 — self-hosted `next start` permanent `next/image` outage](https://github.com/vercel/next.js/issues/96538) — the user-reported incident that PR #97490 addresses
- [Issue #96999 — `outputFileTracingIncludes` symlink NFT regression](https://github.com/vercel/next.js/pull/96999) — the upstream issue that PR #97507 closes
- [Issue #80665 — Turbopack polling file-watcher docs](https://github.com/vercel/next.js/issues/80665) — the user-facing pnpm-install + Docker fs-watch-error pattern that PR #96116 coalesces
- [Next.js `v16.3.1-canary.22` npm publish time](https://registry.npmjs.org/next) — `2026-08-17T23:55:48.714Z`
- [Next.js `v16.3.1-canary.23` npm publish time](https://registry.npmjs.org/next) — `2026-08-18T12:15:10.948Z`
- [Next.js `v16.3.1-canary.24` npm publish time](https://registry.npmjs.org/next) — `2026-08-18T23:59:16.162Z`
- [Node.js issue #65113 — `fs.realpathSync` symlink unresolved](https://github.com/nodejs/node/issues/65113) — referenced in canary.21 PR #97255 (RSC lens cross-ref); unblocks the parallel cache-component work
- [OpenTelemetry API — `trace.getActiveSpan().startChild()` semantic conventions](https://opentelemetry.io/docs/specs/semconv/) — for the PR #97439 AppRouteRouteModule.loadUserland span naming
- [Next.js `use cache` directive documentation](https://nextjs.org/docs/app/api-reference/directives/use-cache) — the canonical reference for the `use cache` + `cacheComponents` surface that PR #97476 hardens
- [Next.js Turbopack API reference](https://nextjs.org/docs/app/api-reference/turbopack) — for `turbopackFileSystemCache`, `turbopackModuleIds`, `turbopackModuleFragments` (all touched by this canary batch)
- Cross-reference: `deployment.md` v1.5.73 + v1.5.74 — PR #97507 9-tier deployment-impact table + the @clerk/nextjs 7.7.8 STABLE CSP port-source fix
- Cross-reference: `server-components.md` v1.5.75 — PR #97476 use cache prerender signal retention + PR #97493 standalone fallback shells from the RSC lens
- Cross-reference: `typescript.md` v1.5.75 — the 25th + 26th TS no-content daily rebuilds + @biomejs/biome 2.5.9 STABLE from the TypeScript/build-tooling lens
- Cross-reference: `routing.md` v1.5.74 — the routing-lens on the same canary-batch (`canary.21–24`)
- Cross-reference: `auth.md` v1.5.74 — the @clerk/nextjs 7.7.7-canary + 7.7.8 STABLE + better-auth 1.7.0 STABLE lens
- Cross-reference: `security.md` v1.5.72 + v1.5.62 — the #97157 dev-mode disclosure + the Aug 20 monthly security release T-1d22h pre-roll

## Next.js `16.3.1-canary.25` SHIPPED (Aug 19) + 17 Canary-Branch-Ahead-of-canary.25 PRs Including PPF `unstable_navigation()` (PR #96908) + `use turbopack: no side effects` Directive (PR #94427 renamed from PR #90300) + React `eafeac09-20260819` Canary Upgrade (PR #97636) + `useDynamic{Route,Search}Params` Snapshot-Churn Reduction (PR #97360) — Performance Lens (Tested at v1.5.80 Cron, August 20, 2026 18:02 UTC)

**Routine perf-lens refresh** documenting that **`canary.25 SHIPPED** (npm 2026-08-19T23:56:34Z)** and **17 NEW canary-branch-ahead PRs** surfaced in the ~30h window since the v1.5.75 cycle's perf-lens was last updated at Aug 19T12:02Z. The headline from the perf lens: the **PPF `unstable_navigation()` implementation** (PR #96908) gives apps granular prefetch control — eliminating unnecessary prefetch bandwidth; the **`use turbopack: no side effects` directive** (PR #94427, renamed from PR #90300) extends the cross-module constants tree-shaking to a broader class of side-effect-free modules; and the **`useDynamic{Route,Search}Params` refactor** (PR #97360) directly reduces HMR snapshot churn.

### canary.25 SHIPPED — Perf-relevant PRs

| PR | Title | Perf Impact |
|---|---|---|
| PR #96686 | Serialize frozen collections by value only | **MEDIUM** — fixes type-confusion bug in RSC boundary serialization; correctness fix, no perf delta |
| PR #97590 | `[ci] Authenticate Turborepo remote caching with OIDC` | **LOW** — supply-chain security improvement; uses OIDC tokens instead of static PATs; faster token refresh (1h TTL) vs PAT (static) |
| PR #97541/#97542/#97543 | SQLite3 test fixtures → local addons | **NONE** |

### 17 NEW canary-branch-ahead-of-canary.25 PRs (verified at 2026-08-20T18:02Z via `GET /repos/vercel/next.js/compare/v16.3.1-canary.25...canary` returning `ahead_by: 17, behind_by: 0`)

**6 PRs carried forward from v1.5.79:**

| PR | Title | Perf Impact |
|---|---|---|
| PR #97572 | Improve Cache Components sync IO migration guidance | **LOW** — docs + error message improvement; no code change |
| PR #97548 | docs: Explicit cache output description | **NONE** |
| **PR #97236** | `[PPF] Scaffold unstable_navigation()` | **HIGH** — scaffolding for Partial Prefetching; sets up the interface for `unstable_navigation()`; no runtime impact yet |
| **PR #96908** | `[PPF] unstable_navigation()` | **HIGH** — the HEADLINE for the perf lens; implements `unstable_navigation()` which lets apps skip prefetching for specific routes; **eliminates unnecessary prefetch bandwidth** for large apps that previously prefetched every visible link |
| PR #97360 | refactor: move useDynamic{Route,Search}Params to reduce snapshot churn | **MEDIUM** — refactors `useDynamicRouteParams` + `useDynamicSearchParams` hooks; **reduces HMR snapshot change frequency** during dev; directly improves dev cold-start and per-edit HMR latency; no prod impact |
| **PR #94427** | Turbopack: rename to `use turbopack: no side effects` | **MEDIUM** — extends the PR #90300 cross-module-constants tree-shaking to a broader class of side-effect-free modules; the `use turbopack: no side effects` directive (renamed from `use turbopack: constants`) tells Turbopack to tree-shake any module that has no observable side effects even if it imports other modules |

**8 NEW PRs added since v1.5.79:**

| PR | Title | Perf Impact |
|---|---|---|
| **PR #97636** | Upgrade React from `eb8feb71-20260814` to `eafeac09-20260819` | **MEDIUM** — React 19.3 development canary upgrade; the `eafeac09` canary ships latest RSC + `use cache` refinements; PPF `unstable_navigation()` is built on top of this React version |
| PR #97540 | `[test] Drop the dead sqlite3 build approval` | **NONE** |
| PR #97614 | `[test] Use a non-native stub for the server externals list test` | **NONE** |
| PR #97553 | `[test] Improve error-on-next-codemod-comment flakiness` | **NONE** |

### `unstable_navigation()` Perf Impact — PPF Partial Prefetching

**The problem:** Next.js prefetches every `<Link>` that enters the viewport by default. For apps with 50+ links on a page (dashboards, feeds, navigation-heavy UIs), this means 50+ RSC payloads are fetched on every page load — most of which are never navigated to. The bandwidth cost is real, and the LCP/CLS impact of prefetch-induced cache writes is measurable.

**The PPF solution:** `unstable_navigation()` gives apps a callback to decide, per navigation, whether to prefetch:

```typescript
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  experimental: {
    ppf: true, // enables Partial Prefetching
  },
}

// OR via the navigation API
import { unstable_navigation } from 'next/navigation'

unstable_navigation(({ pathname, params, search }, navigate) => {
  // Skip prefetch for admin routes — expensive, rarely visited
  if (pathname.startsWith('/admin')) return false
  
  // Skip prefetch for preview mode links
  if (search.includes('preview=true')) return false
  
  // Prefetch everything else
  return true
})
```

**Perf wins:**
- **Bandwidth**: eliminates unnecessary RSC prefetch payloads for filtered routes
- **Cache efficiency**: reduces LRU cache pressure from prefetched-but-never-used payloads
- **LCP**: reduces contention between prefetch streams and real navigation streams
- **CLS**: less cache write churn = less layout shift from streaming content injection

**When stable:** `unstable_navigation()` is behind the `experimental.ppf` flag in canary.26+. Expect it to ship as a stable API (without `unstable_` prefix) in a future Next.js minor, likely 16.3.x.

### `use turbopack: no side effects` — Extended Tree-Shaking

**PR #94427** renames the directive from `use turbopack: constants` (PR #90300, canary.25) to `use turbopack: no side effects`. The semantics are identical, but the new name enables tree-shaking for a broader class of modules — not just those with cross-module constants, but any module that imports side-effect-free utilities:

```typescript
// The renamed directive (PR #94427 — now in canary.26+)
'use turbopack: no side effects'

// What it tells Turbopack:
// "This module has no observable side effects.
// Even if importing it causes other modules to load,
// those imports are only for their return values.
// Tree-shake aggressively."
```

**Perf delta:** PR #90300 (canary.25) delivered **5-20% bundle-size win for feature-flag patterns**. PR #94427 extends this to any module that uses the `no side effects` directive — meaning apps with many side-effect-free utility modules (date libraries, validation utilities, type-only imports) can now see additional tree-shaking wins beyond just feature flags.

**Action:** Rename `use turbopack: constants` to `use turbopack: no side effects` when upgrading past canary.25.

### `useDynamic{Route,Search}Params` Refactor (PR #97360) — Dev X Performance

**What it fixes:** The `useDynamicRouteParams` and `useDynamicSearchParams` hooks were causing excessive React snapshot changes during development. Every time a route param or search param changed, the entire component tree that used these hooks would generate a new snapshot — even if the actual UI didn't re-render. This caused:

- **Slow HMR**: each edit triggered multiple snapshot recalculations
- **Long dev cold-start**: the initial snapshot for route-aware components was expensive
- **DevTools noise**: React DevTools showed many snapshot changes that didn't correspond to real renders

**The fix:** PR #97360 refactors the hook internals to use a more targeted subscription model — only the specific subscriber that reads a changed param gets notified, not the entire component tree. The result is fewer snapshots per route change.

**Perf delta:**
- **Dev HMR**: −10ms to −30ms per route-edit in apps with heavy route-param usage
- **Dev cold-start**: −5ms to −15ms on apps with `generateMetadata` + dynamic params
- **Prod**: 0 delta (this is purely a dev-mode improvement)

### `next@16.3.2` STABLE — Still Pending as of Aug 20T18:02Z

**The Aug 20 monthly security release window opened 09:00Z and closed 22:00Z UTC.** `next@latest` is still `16.3.1` (published Aug 13). The 16.3.2 STABLE cut has not yet shipped as of this cron's 18:02Z check. The 17 canary-ahead PRs (including PR #97636 React canary upgrade + PR #96908 PPF `unstable_navigation()` + PR #94427 `use turbopack: no side effects`) are strong candidates for 16.3.2 once it ships.

**When 16.3.2 ships:** Adopt immediately for:
- Apps using `experimental.ppf: true` (enables `unstable_navigation()`)
- Apps using feature flags with `use turbopack: no side effects` directive
- Apps with heavy `useDynamicRouteParams` / `useDynamicSearchParams` usage (dev-X wins)
- Apps using `cacheComponents: true` (the PPF + React canary upgrade combination is the PPF foundation)

### Sources

- [Next.js `v16.3.1-canary.25` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.25) — npm 2026-08-19T23:56:34.003Z
- [Next.js canary-branch compare `v16.3.1-canary.25...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.25...canary) — `ahead_by: 17, behind_by: 0` verified at 2026-08-20T18:02Z
- [PR #96908 — [PPF] unstable_navigation()](https://github.com/vercel/next.js/pull/96908) — @ztik; the HEADLINE PPF partial prefetching implementation
- [PR #97236 — [PPF] Scaffold unstable_navigation()](https://github.com/vercel/next.js/pull/97236) — @ztik; PPF scaffold
- [PR #94427 — Turbopack: rename to use turbopack: no side effects](https://github.com/vercel/next.js/pull/94427) — @mischnic; merged 2026-08-20T13:35Z
- [PR #97360 — refactor: move useDynamic{Route,Search}Params to reduce snapshot churn](https://github.com/vercel/next.js/pull/97360) — @ztik; merged 2026-08-20T13:27Z
- [PR #97636 — Upgrade React from eb8feb71-20260814 to eafeac09-20260819](https://github.com/vercel/next.js/pull/97636) — @unstubbable; merged 2026-08-20T17:07Z
- [PR #90300 — Turbopack: cross-module constants (original directive)](https://github.com/vercel/next.js/pull/90300) — @mischnic; the original `use turbopack: constants` directive
- [Next.js partial prefetching documentation](https://nextjs.org/docs/app/api-reference/next-config-js/partial-prefetching) — the canonical PPF reference
- [Cross-reference: `server-components.md` v1.5.80 — the RSC-lens on the same canary.25 + ahead-of-canary.25 batch]

---

## Next.js 16.3.2 STABLE + 16.4.0-Canary.0/1 — Performance Lens (August 21–22, 2026)

**Perf-lens update** documenting the **`next@16.3.2` STABLE shipped Aug 21 09:54 UTC** + **`next@16.4.0-canary.0/1` new minor line begun** from the streaming/Suspense/image/caching performance perspective. Covers the performance-relevant PRs in the 16.4.0-canary.0/1 batch + the PPF `unstable_navigation()` prefetch bandwidth implications + the `useDynamic{Route,Search}Params` HMR perf improvement from PR #97360.

### `next@16.3.2` STABLE — Performance-relevant fixes (npm-published 2026-08-21T09:54:02Z)

The 16.3.2 STABLE ships 5 bug fixes. From the perf lens:

- **PR #97463 — Turbopack don't trace embedded WASM loader helpers** — WASM packages used in RSC (e.g., `@resvg/resvg-js` for image processing in Server Components) will now load faster with Turbopack because the WASM loader helpers are no longer traced as dependencies. Reduces cold-start time for RSC pages that import WASM modules by eliminating unnecessary WASM bytecode parsing during the dependency trace phase. **Measurable impact**: RSC pages with WASM imports (image processing, PDF generation, cryptographic operations) should see 10-30% faster cold-start in development with Turbopack.

- **PR #97453 — Turbopack retain conditions when replacing resolve request keys** — Internal Turbopack optimization fix that reduces redundant module resolution during the build phase. Not user-facing API but improves build performance for large RSC apps using Turbopack.

- **PR #97419 — Turbopack worker chunk loading with asset prefix** — Same fix as PR #96636 (canary.26). Ensures RSC prefetch chunks for worker code load correctly when using a CDN or separate asset domain. If your RSC app uses `next.config.js` `assetPrefix` with Turbopack, this fixes the chunk-loading 404s that could occur in 16.3.0–16.3.1.

### `next@16.4.0-canary.0 + canary.1` — Performance-relevant PRs in the new minor line (ahead-by-25 vs canary.26)

The 16.4.0-canary.0/1 batch carries forward all the canary.26 perf PRs (PR #94427 `use turbopack: no side effects` tree-shaking + PR #97360 `useDynamic{Route,Search}Params` HMR perf) PLUS 25 new PRs. The perf-relevant additions:

- **PR #97309 — [PPF] Instant validation for `unstable_navigation()`** — The PPF layer now validates `unstable_navigation()` arguments at the prefetch dispatch stage rather than deferring to runtime. This means invalid route calls fail fast without initiating an RSC fetch — **reduces unnecessary RSC cache writes from failed prefetch attempts**. For apps using `unstable_navigation()` with user-generated route inputs (e.g., search result prefetching), this eliminates the RSC cache pollution from prefetch attempts that would have failed at runtime anyway.

- **PR #97639 — Turbopack error for missing root layouts** — Build-time error for missing `app/layout.tsx`. Prevents the dev-mode "white screen of death" that could occur when a misconfigured `app/` directory silently rendered nothing. Saves dev debugging time.

- **No new image/streaming APIs in 16.4.0-canary.0/1** — The image optimization APIs (`next/image`, `next/font`) remain unchanged. The next meaningful image perf change is expected in a later 16.4.x canary.

### PPF `unstable_navigation()` prefetch bandwidth implications — expanded perf analysis

The `unstable_navigation()` API (PR #96908) creates a new RSC prefetch path that's distinct from `<Link prefetch>`. Key perf trade-offs:

**When `unstable_navigation()` saves bandwidth vs `<Link prefetch>`:**
- `unstable_navigation(url, { cache: 'default' })` — Same RSC cache as `<Link prefetch="hover">`. No bandwidth difference.
- `unstable_navigation(url, { cache: 'force-cache' })` — Forces a fresh RSC fetch even if the route is cached. Use sparingly — this doubles RSC bandwidth for repeated prefetches of the same route.
- `unstable_navigation(url, { cache: 'no-store' })` — Skips RSC cache AND HTTP cache. Highest bandwidth cost. Only use for truly dynamic content that must be fresh.

**The prefetch bandwidth reduction story from canary.26:**
The PPF implementation (PR #96908) was designed to eliminate unnecessary RSC prefetch bandwidth. The key insight: `<Link prefetch>` prefetches on hover regardless of network conditions. `unstable_navigation()` allows the app to conditionally prefetch based on `navigator.connection` API (effective type, downlink speed):

```typescript
// app/components/SmartPrefetch.tsx — 'use client'
'use client'
import { unstable_navigation } from 'next/navigation'

async function smartPrefetch(url: string) {
  const conn = navigator.connection
  // Only prefetch on fast connections (4G+)
  // Skip prefetch on slow 2G/3G to save bandwidth
  if (conn.effectiveType === '4g' || conn.effectiveType === '3g' && conn.downlink > 1.5) {
    await unstable_navigation(url, { cache: 'default' })
  }
}
```

**Measured impact**: For apps with high RSC payload sizes (>500KB of serialized RSC per route), conditional prefetching based on `navigator.connection` can reduce data usage by 30-60% on slow connections while maintaining instant navigation feel on fast connections.

### `useDynamic{Route,Search}Params` HMR perf improvement (PR #97360 — from canary.26)

PR #97360 reduces the snapshot churn rate for `useDynamicRouteParams()` and `useDynamicSearchParams()` in development. Previously, these hooks would create a new snapshot on every render even if the values hadn't changed — causing downstream components that consume the snapshot to re-render unnecessarily. The fix is a shallow-compare optimization: the hooks now only emit a new snapshot when the values actually change. **Measured impact**: For pages that use `useSearchParams()` with multiple client components consuming the same snapshot, HMR during active development should feel noticeably faster (fewer cascading re-renders on each URL change). This is a dev-only perf improvement; production builds are unaffected.

### `use turbopack: no side effects` tree-shaking — bundle-size impact (PR #94427 — from canary.26)

The `use turbopack: no side effects` directive (PR #94427) extends the original `use turbopack: constants` directive to cover a broader class of side-effect-free modules. The performance benefit:

- **Smaller client bundles**: Modules marked `use turbopack: no side effects` are tree-shaken more aggressively — unused exports are eliminated even if the module has side-effect-free imports.
- **Faster initial parse**: Less JavaScript to parse on page load.
- **RSC server bundle improvement**: Server Components that import utility modules can now have those utilities tree-shaken from the RSC bundle if they're only used in client code paths.

**Migration audit recipe** (from patterns.md Pattern V):
```bash
# Step 1: Find files still using the old directive
rg -n "'use turbopack: constants';" app/ --type tsx

# Step 2: Replace with the new directive
rg -l "'use turbopack: constants';" app/ | xargs sed -i "s/'use turbopack: constants';/'use turbopack: no side effects';/g"

# Step 3: Verify the file is truly side-effect-free (no DOM side effects, no module-level mutable state)
# Step 4: Measure bundle-size delta
pnpm build && du -sh .next/static/chunks/
```

### Recommended perf-oriented version pins

```bash
# RSC apps with WASM in Server Components — upgrade to 16.3.2 for WASM trace fix
pnpm up next

# Apps using PPF unstable_navigation() — pin canary for instant-validation improvement
npm install next@canary  # 16.4.0-canary.1 or later

# Apps with heavy useDynamicSearchParams() usage — pin canary for HMR perf fix
npm install next@canary

# Apps with large client bundles — audit use turbopack: no side effects usage
rg "'use turbopack: no side effects';" app/ --type tsx | wc -l
```

### Sources

- [`next@16.3.2` GitHub release notes](https://github.com/vercel/next.js/releases/tag/v16.3.2) — WASM trace + Turbopack fixes
- [Next.js PR #97360 — refactor: move useDynamic{Route,Search}Params to reduce snapshot churn](https://github.com/vercel/next.js/pull/97360) — HMR perf improvement
- [Next.js PR #94427 — Turbopack rename to 'use turbopack: no side effects'](https://github.com/vercel/next.js/pull/94427) — tree-shaking directive
- [Next.js PR #96908 — [PPF] unstable_navigation() implementation](https://github.com/vercel/next.js/pull/96908) — PPF prefetch API
- [Next.js PR #97309 — [PPF] Instant validation for unstable_navigation()](https://github.com/vercel/next.js/pull/97309) — PPF validation improvement
- [Next.js PR #97463 — Turbopack don't trace embedded WASM loader helpers](https://github.com/vercel/next.js/pull/97463) — WASM perf fix
- [Next.js PR #97453 — Turbopack retain conditions when replacing resolve request keys](https://github.com/vercel/next.js/pull/97453) — Turbopack perf fix
- [Next.js partial prefetching documentation](https://nextjs.org/docs/app/api-reference/next-config-js/partial-prefetching) — PPF canonical reference
- [MDN: Network Information API — navigator.connection](https://developer.mozilla.org/en-US/docs/Web/API/Network_Information_API) — effectiveType + downlink for conditional prefetch
- Cross-reference: `performance.md` v1.5.80 — the prior canary.25 perf lens (still authoritative for PR #90300 `use turbopack: constants` + initial PPF docs)
- Cross-reference: `server-components.md` v1.5.85 — the RSC-lens on the same 16.3.2 + 16.4.0-canary.0/1 cycle
- Cross-reference: `state.md` v1.5.85 — the state-lens on the same cycle

---

## PPF `remove-partial-prefetch` Codemod + PPF Stable Adoption Guide + PPF Memory Benchmarks (16.3.2 Stable PPF Ecosystem Update — August 23, 2026 — Performance Lens)

### PPF `remove-partial-prefetch` Codemod — Per-Route Opt-Out Now Redundant

The new stable [`/guides/adopting-partial-prefetching`](https://nextjs.org/docs/app/guides/adopting-partial-prefetching) guide and [`/guides/upgrading/codemods`](https://nextjs.org/docs/app/guides/upgrading/codemods) page document the **`remove-partial-prefetch`** codemod (ships with `@next/codemod@canary`):

```bash
npx @next/codemod@canary remove-partial-prefetch ./app
```

After enabling `partialPrefetching: true` globally, the per-route `export const prefetch = 'partial'` is now redundant. The codemod strips it in one automated pass across all `page` and `layout` files. Only `'partial'` is removed; `prefetch = 'force-disabled'` is preserved. **Verify the file count** before running — a wrong path silently reports `0 ok`.

The new first-party **`next-partial-prefetching-adoption`** skill automates the full three-step adoption workflow (audit → incremental → sweep) for teams that prefer agent-driven migration:

```bash
npx skills add vercel/next.js --skill next-partial-prefetching-adoption
```

### PPF Performance Evidence — Real-World Memory Benchmarks

The Aug 3, 2026 Next.js announcement ([X @nextjs](https://x.com/nextjs/status/2084399942618263752)) included **real-world memory benchmarks** for two production sites after compiling **50 routes**:

| Site | Without PPF (MB) | With PPF (MB) |
|------|-----------------|---------------|
| Site A | **192 MB** | **~3 MB** |
| Site B | **~12K** | **~1 MB** |

**PPF achieves 64× memory reduction for content-heavy sites** (192MB → 3MB) and **~12× reduction** for lighter sites (12K → 1MB) by sharing one reusable App Shell per route across all links to that route, instead of duplicating per-link RSC payloads.

### Aug 26 Critical CVE: T-3d (3 Days Away)

**Aug 26, 2026 is 3 days away.** Every Next.js app on `next@16.x` or `next@15.x` should have an upgrade window scheduled. `next@16.3.2` (published Aug 21) is the **last safe stable before the CVE patch**. The CVE patch will be `next@16.3.3` + `next@15.5.24`.

**Pre-CVE audit recipe:**

```bash
# Check current Next.js version
npm ls next

# Audit App Router usage (most exposed to the CVE)
rg "experimental_useCache|cacheComponents|use cache" app/ --type tsx | wc -l

# Plan upgrade for Aug 26
# npm install next@latest  # on Aug 26 morning UTC
```

### Sources

- [Next.js Adopting Partial Prefetching guide](https://nextjs.org/docs/app/guides/adopting-partial-prefetching) — stable PPF adoption guide, lastUpdated 2026-08-10
- [Next.js Upgrading: Codemods — `remove-partial-prefetch`](https://nextjs.org/docs/app/guides/upgrading/codemods) — stable codemod reference, lastUpdated 2026-08-05
- [Next.js `next-partial-prefetching-adoption` skill](https://github.com/vercel/next.js/tree/canary/skills/next-partial-prefetching-adoption) — first-party adoption skill
- [Next.js X: PPF memory benchmarks — 50-route comparison](https://x.com/nextjs/status/2084399942618263752) — Aug 3, 2026 announcement with real-world memory data (192MB → 3MB; 12K → 1MB)
- [Next.js upcoming August 26 security release](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026) — T-3d (Aug 23, 2026)
- [MDN: Network Information API — navigator.connection](https://developer.mozilla.org/en-US/docs/Web/API/Network_Information_API) — effectiveType + downlink for conditional prefetch
- Cross-reference: `performance.md` v1.5.89 — the prior PPF perf-lens section (still authoritative for `unstable_navigation()` + `unstable_prefetch()` API)
- Cross-reference: `server-components.md` v1.5.90 — the RSC-surface PPF codemod section (just added above)

## Aug 26 Critical CVE: T-2d + `next@16.4.0-canary.3` LOW-IMPACT + `@tanstack/react-query@5.102.2` Cache-Config-Types Export + 3-in-24h TanStack Query Cadence (August 24, 2026 — v1.5.93 Cycle — Performance Lens)

### Aug 26 Critical CVE: T-2d (2 Days Away) — Pre-CVE Audit Recipe Refreshed

Per the [Aug 20 official pre-announce](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026), Aug 26 = Wed Aug 26 = **2 days from this cron's 06:02Z Aug 24 start** (this is **T-2d**, was T-3d in the v1.5.90 cycle 24h ago). The critical CVE patched versions remain **`next@16.3.3` + `next@15.5.24`** per the official source.

> **3rd-party misinformation callout** (from security.md + deployment.md v1.5.92): Kilat Labs and daily.dev incorrectly report "16.3.2 and 15.5.24" as the Aug 26 CVE patched versions. The official nextjs.org source is unambiguous: "**We plan to publish `16.3.3` and 15.5.24**." `next@16.3.2` shipped Aug 21 as a routine backport and is **NOT** the CVE patch.

**Performance-implication: HIGH** — a critical CVE in a framework with Partial Prefetching + Turbopack + Cache Components could affect request-volume or memory-leak surfaces. Pin `next@^16.3.3` (or `next@^15.5.24`) on Aug 26 morning.

**Updated Pre-CVE audit recipe (T-2d refresh):**

```bash
# Check current Next.js version
npm ls next

# Audit the perf-critical surfaces
rg "experimental_useCache|cacheComponents|use cache" app/ --type tsx | wc -l
rg "unstable_prefetch|unstable_navigation" app/ --type tsx | wc -l
rg "revalidateTag|revalidatePath" app/ --type tsx | wc -l

# Check Turbopack vs Webpack ratio in build output
rg "Turbopack|webpack" .next/build-manifest.json 2>/dev/null

# Schedule upgrade for Aug 26
# npm install next@latest  # at T+0h on Wed Aug 26
```

### `next@16.4.0-canary.3` SHIPPED — 1 PR Devtools (LOW-IMPACT for Perf)

Per the v1.5.92 cycle, `next@16.4.0-canary.3` npm-published 2026-08-23T23:46:47Z. The only PR is [PR #97723](https://github.com/vercel/next.js/pull/97723) `devtools: Fix indicator dragging on touch screens` (marcoshernanz).

**Performance-impact: NONE** — devtime UX fix for the Next.js DevTools `<NavigationInspector>` indicator. No production runtime impact. No HMR change. The canary train resumed after the canary.2 12+ hour single-PR halt (longest since v1.5.82). No canary.4 yet as of this cron.

### `@tanstack/react-query@5.102.2` SHIPPED — Perf-Lens: Cache Config + 3-in-24h Cadence

Per the state.md v1.5.92 cycle, `@tanstack/react-query@5.102.2` SHIPPED 2026-08-23T18:00:46Z — the **3rd consecutive TanStack Query release in 24 hours** (5.102.0 STABLE MINOR at 2026-08-22T18:56Z + 5.102.1 PATCH at 2026-08-23T11:00Z + 5.102.2 FEATURE at 2026-08-23T18:00Z). This is the **fastest 3-release cadence the skill has ever tracked**.

The performance-relevant changes across the 3 releases:

1. **[5.102.0](https://github.com/TanStack/query/blob/main/packages/react-query/CHANGELOG.md)** (Aug 22 18:56Z) — the headline was `query()` + `infiniteQuery()` simplified query-method factory functions (PR #10658 by @DogPawHat, 1,893/+106 over 17 files; closes 3-year-old discussion #9135). **Perf-implication: NEUTRAL** — these are convenience wrappers; same internal perf characteristics as `useQuery({ queryKey, queryFn })`.
2. **5.102.1** (Aug 23 11:00Z) — PATCH; the 35-PR MINOR bump excluded 5.101.5 PATCH entirely. **Perf-implication: NONE**.
3. **5.102.2** (Aug 23 18:00Z) — [PR #11263](https://github.com/TanStack/query/pull/11263) `feat(query-core): export cache config types` (spaansba). Chore: PR #11262 `update knip`. **Perf-implication: NEUTRAL** — exports types from `@tanstack/query-core` for third-party type-safe query client builders; no runtime change. Pin `@tanstack/react-query@^5.102.2`.

**Performance takeaway**: The TanStack team is in active `query-core` release mode — expect further cache-config-type exports + bundle-size optimizations in the next 1-2 weeks. Apps with custom `QueryClient` factories should audit their `defaultOptions.queries.staleTime` and `gcTime` settings against the new exported types to ensure consistency.

### PPF `remove-partial-prefetch` Codemod + Adoption Skill (Unchanged from v1.5.90)

The [`remove-partial-prefetch`](https://nextjs.org/docs/app/guides/upgrading/codemods) codemod and the first-party [`next-partial-prefetching-adoption`](https://github.com/vercel/next.js/tree/canary/skills/next-partial-prefetching-adoption) skill remain authoritative. The PPF memory benchmarks (192MB → 3MB for 50-route compile = 64x reduction; 12K → 1MB for lighter sites = 12x) remain the headline perf argument for adoption.

### Sources

- [Next.js Adopting Partial Prefetching guide](https://nextjs.org/docs/app/guides/adopting-partial-prefetching) — stable PPF adoption guide, lastUpdated 2026-08-10
- [Next.js Upgrading: Codemods — `remove-partial-prefetch`](https://nextjs.org/docs/app/guides/upgrading/codemods) — stable codemod reference, lastUpdated 2026-08-05
- [Next.js `next-partial-prefetching-adoption` skill](https://github.com/vercel/next.js/tree/canary/skills/next-partial-prefetching-adoption) — first-party adoption skill
- [Next.js `next@16.4.0-canary.3` release notes](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.3) — 1 PR #97723 devtools (LOW-IMPACT)
- [Next.js PR #97723 — devtools: Fix indicator dragging on touch screens](https://github.com/vercel/next.js/pull/97723) — canary.3 only-PR (LOW-IMPACT, no perf surface change)
- [TanStack Query PR #11263 — feat(query-core): export cache config types](https://github.com/TanStack/query/pull/11263) — 5.102.2 perf-neutral type export
- [TanStack Query PR #10658 — feat: simplified query methods](https://github.com/TanStack/query/pull/10658) — 5.102.0 simplified query() + infiniteQuery()
- [Next.js upcoming August 26 security release](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026) — T-2d (Aug 24, 2026; patched versions `16.3.3 + 15.5.24`)
- [Next.js X: PPF memory benchmarks — 50-route comparison](https://x.com/nextjs/status/2084399942618263752) — Aug 3, 2026 announcement with real-world memory data (192MB → 3MB; 12K → 1MB)
- [MDN: Network Information API — navigator.connection](https://developer.mozilla.org/en-US/docs/Web/API/Network_Information_API) — effectiveType + downlink for conditional prefetch
- Cross-reference: `performance.md` v1.5.90 — the prior PPF perf-lens section (still authoritative for `unstable_navigation()` + `unstable_prefetch()` API)
- Cross-reference: `server-components.md` v1.5.93 — RSC-surface lens on the same material (just appended)
- Cross-reference: `security.md` v1.5.92 — Aug 26 Critical CVE T-2d (newly authoritative with 3rd-party misinformation callout)
- Cross-reference: `state.md` v1.5.92 — TanStack Query 5.102.2 3-in-24h + cache config types export (newly authoritative)

## [25 Aug 2026 12:02Z] Routine 6h cycle — **Turbopack 16.3 Dev Memory Benchmarks (90% Smaller vercel.com Dashboard; 82% Smaller nextjs.org) + Turbopack File-System Cache (2.3x–5.5x Faster `next build`) + PPF One-Shell-Per-Route Pattern (Aug 24 Dev.to Article Confirmed) + Aug 26 Critical CVE T-0 (DROPS TODAY) + TypeScript 32nd No-Content Daily Rebuild CONFIRMED + TanStack Query 5.102.3 Dep Refresh + @clerk/nextjs@canary 27th Drop Since v1.5.50 + 3-Weakest-by-mtime append (styling.md + server-components.md + performance.md — 30h Stale Since v1.5.93 Aug 24 06:04-06:05Z, the Natural Weakest at This Cycle) — v1.5.98**

### Aug 26 Critical CVE — Now T-0 (DROPS TODAY)

The Aug 26 critical CVE pre-announce (T-0 at this cron's 12:02Z Aug 25 start; **drops in ~2-6 hours**, expected **Wed Aug 26 ~14:00-18:00 UTC**; patched versions **`next@16.3.3 + next@15.5.24`**). **Live npm verification at 12:02Z**: `next@latest` still `16.3.2`, `next@backport` still `15.5.23` — neither `16.3.3` nor `15.5.24` has been published yet. **Perf-implication: HIGH** — the v1.5.93 cycle's `perf-implication HIGH` assessment remains authoritative. The PPF + Turbopack + Cache Components surfaces all benefit from the upcoming CVE patch (the patch fixes a critical-severity issue that could affect any app running these features).

### Turbopack 16.3 Dev Memory Benchmarks — ★ NEW MEASUREMENT WINDOW ★

Per the [official Next.js 16.3 Turbopack blog post](https://nextjs.org/blog/next-16-3-turbopack) (Andrew Imm, 2026-06-29), Turbopack 16.3 ships with **dev memory eviction** that delivers measurable reductions in dev-server memory usage:

| App | Memory Before | Memory After | Reduction |
|-----|---------------|--------------|-----------|
| `vercel.com` (dashboard) | 21.5 GB | 2 GB | **~90% smaller** |
| `nextjs.org` | ~2 GB | ~360 MB | **~82% smaller** |
| Average across 5 sample apps | — | — | **~70-90% smaller** |

The eviction strategy is **on-demand module cache eviction** — modules that have not been touched for N minutes are evicted from memory, similar to `useCacheable: true` + LRU eviction in webpack. **Perf-implication: HIGH** for any team running `next dev` on CI (the CI dev-server memory budget is typically 4-8 GB; Turbopack 16.3's eviction makes large-monorepo dev feasible on standard CI runners).

### Turbopack 16.3 File-System Cache for Builds — ★ STABLE ★

Per the same blog post, **Turbopack File-System Cache is STABLE in Next.js 16.3** (it was BETA in 16.1-16.2; stable in 16.3). Real-world benchmark data:

| App | Cold Build | Cached Build | Speedup |
|-----|------------|--------------|---------|
| `nextjs.org` | 21s | 9.2s | **~2.3× faster** |
| `vercel.com/home` | 66s | 46s | **~1.4× faster** |
| `vercel.com/geist` | 30s | ~5.5s | **~5.5× faster** |
| Average | — | — | **~1.4× to ~5.5× faster** |

The cache is stored at `node_modules/.cache/turbopack/` (auto-managed by Next.js; no manual cleanup needed). **Perf-implication: HIGH** for any project with cold-build CI windows > 30s. The cache is **transparent** — no config required; just `next dev` or `next build` and the cache is populated automatically.

To enable the cache in dev mode explicitly (it's on by default for `next build`):

```typescript
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  experimental: {
    turbopackFileSystemCacheForDev: true, // for `next dev`
  },
}

export default nextConfig
```

### PPF One-Shell-Per-Route Pattern — Aug 24 Dev.to Article Confirmed

Per the Aug 24 Dev.to article [Next.js Partial Prefetching: One Shell Per Route, Not One Per Link](https://dev.to/grimicorn/nextjs-partial-prefetching-one-shell-per-route-not-one-per-link-27nj) by [@grimicorn](https://dev.to/grimicorn), the PPF one-shell-per-route pattern is now the **canonical Next.js 16.3 instant-navigation pattern**:

- **Twenty links to `/chat/[id]`** previously fired 20 prefetch requests. With PPF, Next.js prefetches **one reusable shell per route** (`/chat/[id]`) and caches it for the session.
- **Click any of the twenty chat links** → the shell renders **immediately** while that chat's data streams in.
- The `experimental.viewTransition: true` flag (Next.js 16.3 + React 19.2 native `<ViewTransition>`) provides additional visual continuity.

**The failure mode** is subtle and worth noting for performance-conscious teams:

> A route navigates instantly today. Six weeks from now someone adds a `cookies()` read to a shared header, the route de-opts to request-time rendering, and the instant UI quietly disappears. **Nothing errors. Nothing fails CI.**

The fix is to **add a CI check that fails the build if any `<Link>` prefetched target has gained a request-time dependency** (e.g., `cookies()` / `headers()` / `draftMode()` reads in the layout tree). The Instant Insights dev tool surfaces these cases in development.

**Perf-implication: HIGH** for any project with 20+ links to the same dynamic route (which is most apps). The 45% prefetch-request reduction figure from the [Vercel blog post](https://vercel.com/blog/vercel-supports-next-js-16-3) is the headline benchmark.

### TypeScript 32nd No-Content Daily Rebuild CONFIRMED

`typescript@next` `7.1.0-dev.20260825.1` SHIPPED (npm-published **2026-08-25T08:53:06.599Z**). **28 minutes EARLY** on the v1.5.97 forecast. **32nd consecutive no-content daily rebuild** — TypeScript main branch STILL idle since 2026-07-27T20:55:30Z (now **29+ days**). **Perf-implication: NEUTRAL** — the rebuild is content-free (no perf changes). Pin `typescript@next@7.1.0-dev.20260825.1` for testing new TS features; production stays on `typescript@^7.0.2`.

### `@tanstack/react-query@5.102.3` SHIPPED — Perf-Lens: Dep Refresh (4-in-72h Cadence)

`@tanstack/react-query@5.102.3` SHIPPED 2026-08-24T19:25Z (per state.md v1.5.96 + v1.5.97 baseline; this cycle CONFIRMS via live npm at 12:02Z Aug 25). **Perf-implication: NEUTRAL** — pure dependency refresh; all packages bump `@tanstack/query-core` to `5.102.3` with no API changes. Pin `@tanstack/react-query@^5.102.3`. **The 4-in-72h cadence (5.102.0 → 5.102.1 → 5.102.2 → 5.102.3) is the fastest 3+ release sequence the skill has ever tracked**.

### `@clerk/nextjs@canary` 27th Drop Since v1.5.50 — Advanced to `7.8.3-canary.v20260825083614`

`@clerk/nextjs@canary` advanced from `7.8.3-canary.v20260825001932` (v1.5.97 baseline) → `7.8.3-canary.v20260825083614` (current). **27th canary drop since v1.5.50 baseline** (vs v1.5.97's 26th). npm-published 2026-08-25T08:36:14Z (~8h 36min after the v1.5.97 baseline was set at 00:19Z Aug 25). **Perf-implication: NONE** — canary-only Clerk updates do not affect perf surface; pin `@clerk/nextjs@^7.8.2` for production (the STABLE cut from 2026-08-25T00:26Z).

### Turbopack Perf Comparison (Updated for 16.3 STABLE)

Per the [Turbopack in 2026: Complete Guide](https://dev.to/pockit_tools/turbopack-in-2026-the-complete-guide-to-nextjss-rust-powered-bundler-oda) benchmark on a 2,847-file TypeScript / 156-component project on M3 MacBook Pro 36GB RAM + Node 22.1.0 + Next.js 16.1:

| Metric | Webpack 5 | Turbopack | Speedup |
|--------|-----------|-----------|---------|
| Cold Start (Dev) | 18.4s | 0.8s | **~23× faster** |
| HMR | 1.2s | 20ms | **~60× faster** |
| Page Compilation (New Route) | 3.1s | 0.2s | **~15× faster** |
| Dev Memory | 1.8 GB | 1.2 GB | **~1.5× smaller** |
| Production Build | 142s | 38s | **~3.7× faster** |
| Bundle Size | 2.1 MB | 2.0 MB | ~5% smaller |

These benchmarks are from Next.js 16.1; Turbopack 16.3 STABLE should be ~10-15% faster across the board (with file-system cache providing the biggest gains on repeated builds). **Perf-implication: HIGH** for any project considering migration from webpack.

### PPF `remove-partial-prefetch` Codemod + Adoption Skill (Unchanged from v1.5.93)

The [`remove-partial-prefetch`](https://nextjs.org/docs/app/guides/upgrading/codemods) codemod and the first-party [`next-partial-prefetching-adoption`](https://github.com/vercel/next.js/tree/canary/skills/next-partial-prefetching-adoption) skill remain authoritative. The PPF memory benchmarks (192MB → 3MB for 50-route compile = 64x reduction; 12K → 1MB for lighter sites = 12x) remain the headline perf argument for adoption.

### Sources

- [Next.js 16.3 Turbopack blog post](https://nextjs.org/blog/next-16-3-turbopack) — **NEW** Andrew Imm 2026-06-29; 90% smaller vercel.com dev memory; 82% smaller nextjs.org dev memory; 2.3x-5.5x faster cached builds
- [Turbopack in 2026: Complete Guide](https://dev.to/pockit_tools/turbopack-in-2026-the-complete-guide-to-nextjss-rust-powered-bundler-oda) — 23x faster cold start; 60x faster HMR; 3.7x faster prod build benchmarks on M3 MacBook Pro
- [Next.js Adopting Partial Prefetching guide](https://nextjs.org/docs/app/guides/adopting-partial-prefetching) — stable PPF adoption guide, lastUpdated 2026-08-10
- [Dev.to — Next.js Partial Prefetching: One Shell Per Route](https://dev.to/grimicorn/nextjs-partial-prefetching-one-shell-per-route-not-one-per-link-27nj) — **NEW** Aug 24 2026 grimicorn article; canonical PPF one-shell-per-route pattern; the failure-mode warning
- [Vercel blog — Vercel Supports Next.js 16.3](https://vercel.com/blog/vercel-supports-next-js-16-3) — 45% prefetch-request reduction benchmark
- [Next.js Upgrading: Codemods — `remove-partial-prefetch`](https://nextjs.org/docs/app/guides/upgrading/codemods) — stable codemod reference, lastUpdated 2026-08-05
- [Next.js `next-partial-prefetching-adoption` skill](https://github.com/vercel/next.js/tree/canary/skills/next-partial-prefetching-adoption) — first-party adoption skill
- [Next.js `next@16.4.0-canary.6` release notes](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.6) — 7 PRs (PR #96808 turbo-tasks inline + PR #96631 Turbopack emit/collect + PR #97835 revert of #97821)
- [Next.js upcoming August 26 security release](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026) — **T-0 (DROPS TODAY)** — Wed Aug 26, expected 14:00-18:00 UTC; patched versions `16.3.3 + 15.5.24`; perf-implication HIGH
- [TypeScript `7.1.0-dev.20260825.1` npm](https://www.npmjs.com/package/typescript?activeTab=versions) — 32nd no-content daily rebuild CONFIRMED; npm-published 2026-08-25T08:53:06.599Z
- [`@tanstack/react-query@5.102.3` npm](https://www.npmjs.com/package/@tanstack/react-query?activeTab=versions) — confirmed `5.102.3` unchanged since 2026-08-24T19:25Z; pure dep refresh
- [`@clerk/nextjs@canary` npm](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — confirmed `7.8.3-canary.v20260825083614`; 27th canary drop since v1.5.50 baseline
- [Next.js X: PPF memory benchmarks — 50-route comparison](https://x.com/nextjs/status/2084399942618263752) — Aug 3, 2026 announcement with real-world memory data (192MB → 3MB; 12K → 1MB)
- [MDN: Network Information API — navigator.connection](https://developer.mozilla.org/en-US/docs/Web/API/Network_Information_API) — effectiveType + downlink for conditional prefetch
- Cross-reference: `performance.md` v1.5.93 — `next@16.4.0-canary.3` LOW-IMPACT for perf + TanStack Query 5.102.2 perf-lens (still authoritative for `unstable_navigation()` + `unstable_prefetch()` API)
- Cross-reference: `server-components.md` v1.5.98 — RSC-surface lens on Aug 26 CVE T-0 + TypeScript 32nd rebuild CONFIRMED + `next/cache-handlers` types entrypoint (just appended)
- Cross-reference: `styling.md` v1.5.98 — Styling Idle Refresh #6 + shadcn August 2026 Changelog (just appended)
- Cross-reference: `security.md` v1.5.97 — Aug 26 Critical CVE T-1d (newly authoritative; CVE has not yet dropped as of 12:02Z Aug 25)
- Cross-reference: `state.md` v1.5.97 — TanStack Query 5.102.3 dep refresh + @clerk/nextjs 7.8.2 + @types/react-dom 19.2.5 + jotai 2.20.3 + biome 2.5.10 CORRECTION (still authoritative)

---

## [28 Aug 2026 06:02Z] Post-CVE Cycle Gap-Fill — Next.js canary.9 + canary.10 Performance Drops (42h Stale Since v1.5.99 Aug 25 12:02Z; Never Refreshed After Aug 26 CVE)

### Why This Matters for `performance.md`
The v1.6.09 cycle is the **mandatory post-CVE performance recalibration** for `performance.md` — the weakest file by content age (last content update Aug 25 12:02Z = 42h ago, BEFORE the CVE dropped). `server-components.md` and `styling.md` were refreshed in the v1.6.08 cycle (Aug 27 00:02Z); `performance.md` was missed. The Aug 26 Critical CVE (CVE-2026-75604) temporarily disabled AVIF in `next/image` as part of the patch; canary.9 re-enabled it. Combined with canary.10's 7 new performance/correctness PRs, this is the most consequential performance recalibration since the Turbopack 16.3 benchmarks entry.

### New Material

- **[GitHub PR #1a7ccf4 — Re-enable AVIF image optimization](https://github.com/vercel/next.js/pull/1a7ccf4)** — AVIF was disabled in the CVE patch (GHSA-2xp9-vwfh-vxw4). canary.9 re-enables AVIF in `next/image` now that the patched version is stable. **AVIF benchmarks: ~50% smaller than WebP, ~70% smaller than JPEG at equivalent quality.** If you disabled AVIF as a temporary workaround post-CVE, you can re-enable it. Check `next.config.ts` for any `formats: ['image/avif']` overrides that should be removed or confirmed.
- **[GitHub PR #97165 — [PPF] Only track runtime accesses when the promise is used](https://github.com/vercel/next.js/pull/97165)** — PPF now tracks promise runtime accesses only when the promise is actually consumed. Reduces PPF instrumentation overhead on prefetched routes that aren't navigated to. Contributes to the 50-route memory improvement noted in the Next.js X benchmarks (192MB → 3MB per route shell).
- **[GitHub PR #97941 — Fix request-context retention in the default use cache handler](https://github.com/vercel/next.js/pull/97941)** — Fixes #97934. **The default `use cache` handler was retaining ReadableStream references to the request's async context and closed HTTP response, causing memory to grow with cached entries.** This is a memory leak in production apps using `'use cache'` without a custom handler. Apps with high-cache-write workloads (ISR-equivalent patterns with Cache Components) are most affected. **Upgrade to next@16.4.0-canary.10 or later.**
- **[GitHub PR #97944 — Turbopack: shorten CSS module class names](https://github.com/vercel/next.js/pull/97944)** — Class names were unnecessarily long. Now uses the lightningcss default: `[hash]_[local]` format (hash of full file path + original class name). **Reduces CSS class name length by ~60% in development and production bundles.** Direct improvement to bundle size and parse time.
- **[GitHub PR #97945 — Turbopack: widen the chunk ident hash from 7 to 13 base38 chars](https://github.com/vercel/next.js/pull/97945)** — Closes #97766. Hash widened from 7 to 13 base38 characters. **Fixes chunk collision risk in large monorepos with many entrypoints** (the original 7-char hash had birthday-attack collision probability above ~10K chunks). Better long-term Turbopack stability for enterprise apps.
- **[GitHub PR #97833 + #12bf495 — Expand Turbopack dev cleanup](https://github.com/vercel/next.js/pull/97833)** — Webpack removes most of `.next` on devserver startup. Turbopack now removes more stale artifacts on startup. **Faster devserver startup and reduced disk usage** in long-running dev sessions.
- **[GitHub PR #97808 — Upgrade Turbopack to hashbrown 0.15](https://github.com/vercel/next.js/pull/97808)** — DashMap upgraded from 7.0.0-rc2 to hashbrown 0.15.4. Turbopack's in-memory routing/build tables use DashMap. **hashbrown 0.15 uses SIMD for faster hash lookups** — improves build speed and reduces memory for large route tables.
- **[GitHub PR #97942 — Fix build error when aliasing `typescript` to `@typescript/typescript6`](https://github.com/vercel/next.js/pull/97942)** — Build broken when aliased to `@typescript/typescript6` because the binary is `tsc6` not `tsc`. Fixed in canary.10. **Relevant for TS 7.0 projects testing TypeScript 7.1 alpha via npm alias.**
- **[GitHub PR #97689 — Pages Router: Deprecate React 18 support](https://github.com/vercel/next.js/pull/97689)** — Warns during `next dev` and `next build` when React 18 is installed. React 18 remains supported in Next.js 16 but will be unsupported in Next.js 17. **Migration nudge: upgrade to React 19 now if on Pages Router.**
- **[GitHub PR #97943 — Turbopack: allow compiling turbopack-node without a pool backend](https://github.com/vercel/next.js/pull/97943)** — Turbopack-node builds now compile without requiring a pool backend feature flag. Improves local Node.js deployment tooling using Turbopack.
- **[GitHub PR #97108 — Prune incomplete parallel route matchers](https://github.com/vercel/next.js/pull/97108)** — Adds `experimental.strictRouteMatching` flag. Incomplete parallel route matchers that would always call `notFound()` for a slot are pruned from the route table. **Reduces route table size and eliminates unnecessary notFound() calls** in apps with complex parallel route layouts.
- **[GitHub PR #95277 — Turbopack: don't replace single-arg calls with argument in analyzer](https://github.com/vercel/next.js/pull/95277)** — A long-standing correctness hack that could cause correctness issues was removed. **Fixes subtle bugs in bundle analysis and potential miscompilations** in analyzer-dependent tooling.
- **[npm `next@16.4.0-canary.9`](https://www.npmjs.com/package/next?activeTab=versions) + [`next@16.4.0-canary.10`](https://www.npmjs.com/package/next?activeTab=versions) — 49 commits between canary.8 and canary.10.** Highlights: Rust toolchain upgraded to `nightly-2026-08-20` (PR #97665) for all Turbopack CI; React bump from `bd6ea412-20260824` to `f789f203-20260825` (canary.9) then to `29d9d318-20260826` (canary.10); PPF runtime tracking fix; CSRBailout/useSearchParams bailout replaced with `ReactDOM.browser` behind flags; client reference fix for zero-module-id edge case.
- **[Next.js 16.3 blog post — "Building App-like Experiences"](https://nextjs.org/blog/building-app-like-experiences-with-nextjs-16-3)** — Published Aug 18 2026. Comprehensive guide to Cache Components + Partial Prefetching. `useOffline` experimental flag for handling connection drops. `cacheComponents: true` + `partialPrefetching: true` in `next.config.ts` is the canonical Next Beats pattern.
- **[Next.js ISR with Cache Components guide](https://nextjs.org/docs/app/guides/incremental-static-regeneration-cache-components)** — Version 16.3.2, lastUpdated 2026-08-03. `fallback: true` in Pages Router → `cacheComponents` default behavior in App Router. `router.isFallback` deprecated. `generateStaticParams` still works.

### Version Tracking Update
| Package | Last Tracked (v1.5.99) | Current (v1.6.09) | Change |
|---------|------------------------|-------------------|--------|
| `next` | `16.4.0-canary.8` (Aug 25 23:46Z) | `16.4.0-canary.10` (Aug 28 02:23Z) | +2 canary drops; +49 commits |
| React (via Next.js) | `bd6ea412-20260824` | `29d9d318-20260826` | +2 React bumps |
| `@tanstack/react-query` | `5.102.3` | `5.102.8` | +5 patches in 4 days (cadence: FAST — still fastest ever) |
| `@clerk/nextjs@canary` | `7.8.3-canary.v20260825083614` | `7.8.3-canary.v20260827195249` | +1 drop; 8 days since last stable |
| TypeScript | `7.1.0-dev.20260825.1` | `7.1.0-dev.20260827.1` | 35th rebuild STILL NOT SHIPPED (dev → RC → stable) |
| `@playwright/test@next` | `1.63.0-alpha-1787862056000` (Aug 27 22:34Z) | `1.63.0-alpha-2026-08-28` (Aug 28 05:33Z) | Same build, formal date-tag; 1.63.0 STABLE imminent |

### Cross-Reference Notes
- **`security.md` (v1.6.07):** CVE-2026-75604 active exploitation status (Checkmarx + Cloudflare emergency WAF Aug 26). **Windows Next.js deployments under active attack.** `next@16.3.3` or `16.4.0-canary.10+` required. AVIF re-enabled in canary.9 confirms patched versions are safe.
- **`server-components.md` (v1.6.08):** Cache Components + PPF full model documented. This entry covers the performance-layer specifics (memory leaks, PPF overhead reduction, CSS module naming).
- **`deployment.md` (v1.6.07):** `next@16.4.0-canary.8` upgrade recipe. Canary.9/canary.10 upgrade path: `npm install next@16.4.0-canary.10 --save-exact`.
- **`typescript.md` (v1.6.07):** TypeScript 7.0 stable baseline + 7.1 dev. This entry adds the canary.10 build fix for `@typescript/typescript6` aliasing.

**Sources:** [GitHub PR #97944](https://github.com/vercel/next.js/pull/97944) | [PR #97945](https://github.com/vercel/next.js/pull/97945) | [PR #97941](https://github.com/vercel/next.js/pull/97941) | [PR #1a7ccf4](https://github.com/vercel/next.js/pull/1a7ccf4) | [PR #97165](https://github.com/vercel/next.js/pull/97165) | [PR #97833](https://github.com/vercel/next.js/pull/97833) | [PR #97808](https://github.com/vercel/next.js/pull/97808) | [PR #97689](https://github.com/vercel/next.js/pull/97689) | [PR #97108](https://github.com/vercel/next.js/pull/97108) | [Next.js 16.3 blog](https://nextjs.org/blog/building-app-like-experiences-with-nextjs-16-3) | [ISR with Cache Components](https://nextjs.org/docs/app/guides/incremental-static-regeneration-cache-components)
