# Patterns — Composite Recipes for Common Flows

## Turbopack (Next.js 16)

Turbopack is Next.js's Rust-based bundler. In Next.js 16, Turbopack is the default for new projects — stable for both development and production builds, and used in production on vercel.com and nextjs.org serving 1.2B+ requests. It replaces Webpack for development, offering significantly faster hot module replacement (HMR) and cold start times.

```bash
# Use Turbopack in development (Next.js 16 default)
npm run dev

# Force Turbopack explicitly
next dev --turbopack

# Force Webpack if needed
next dev --webpack
```

**Current status (Next.js 16.2.9):**
- ✅ Stable for development — hot reload, fast refresh, error overlay
- ✅ Stable for production builds (`next build --turbopack`) — 2x–5x faster builds than Webpack; used in production on vercel.com and nextjs.org serving 1.2B+ requests
- ✅ App Router and Pages Router both supported
- ⚠️ Some edge-case features may still differ from Webpack; check the [migration guide](https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopack)

**Enable Turbopack production builds:**
```bash
# In next.config.ts — enable Turbopack for production builds
# next build --turbo  (or set environment variable NEXT_BUILD_USE_TURBOPACK=1)
```

**When using Turbopack:**
```ts
// next.config.ts — Turbopack config options
const nextConfig: NextConfig = {
  turbopack: {
    // Enable experimental features if needed
    // experimental: { ... }
  },
}
```

## Next.js 16.2 Debugging Improvements

Next.js 16.2 introduced several major debugging and development experience improvements:

### Hydration Diff Indicator

Next.js 16.2 adds a **Hydration Diff Indicator** in the error overlay — when hydration fails, the overlay now shows exactly which attributes differ between the server-rendered HTML and the client output. This makes it dramatically faster to find server/client mismatches.

**Common causes of hydration mismatches:**
- `Date` or `Math.random()` used during render — these produce different values on server vs client
- Browser-only APIs accessed during initial render without proper client guards
- Conditional rendering based on `typeof window`

### Server Function Logging

Next.js 16.2 logs **Server Function execution** to the dev terminal — when a Server Action or Route Handler runs, you see which function was called, how long it took, and any errors.

```
[Server Function] POST /api/submit-form
  ✓ Completed in 45ms
  → Returned: { success: true, id: "abc123" }
```

**In the browser DevTools:**
- Server Function calls appear as dedicated entries in the Network tab
- You can replay requests, inspect inputs/outputs, and measure performance

**Debugging tip:** Filter the Network tab by `Server Action` to see only Server Function network requests.

### `--inspect` for `next start`

Next.js 16.2 adds **`--inspect`** flag for production (`next start`):

```bash
# Attach Node.js debugger to production server
next start --inspect
```

This lets you connect Chrome DevTools or VS Code to your production server for live debugging.

**Sources:**
- [Next.js 16.2 release notes](https://nextjs.org/blog/next-16-2)
- [Next.js 16.2 Turbopack improvements](https://nextjs.org/blog/next-16-2-turbopack)

### App Shells Now Exclude Caches With `stale < 5 minutes` (16.3.0-canary.87+, PR [#95833](https://github.com/vercel/next.js/pull/95833) by gnoff, merged 2026-07-15T21:43:49Z)

App Shells are meant to be useful for handling navigations that are **not** prefetched — they fill in the prefetch cache with a usable fallback so the user sees the right shell + layout + error boundary when they click a link that wasn't hovered first. If the underlying `'use cache'` entries have a very low `stale` time, they have to be evicted often from the client router and the App Shell's utility is undermined (a stale shell that immediately invalidates is no better than no shell at all).

canary.87+ **excludes caches with `stale < 5 minutes (300s)` from App Shells**. Concretely, for the built-in default `cacheLife` profiles this means only the `seconds` profile is excluded — every other profile (`minutes`, `hours`, `days`, `max`) has a `stale` of exactly 5 minutes at the time of writing, so they pass through unchanged. The threshold is hard-coded for now; the PR notes "in the future we may want to allow for this threshold to be configured" but didn't ship a knob.

**What this means in practice:**

- **If you use `cacheLife('seconds')` on any data fetched by a route's App Shell** — that data is **no longer eligible to be served from the App Shell**. The shell will fall back to the runtime prefetch path (or the blocking boundary, if the route has `instant = false`). If you depended on the shell serving a 1-second-stale value, that path no longer works; bump to `cacheLife('minutes')` (or higher) for shell-eligibility.
- **If you use a custom profile with `stale < 300s`** — same outcome. Bump `stale` to at least 300s to keep the entry shell-eligible.
- **For everyone else** — no change. The threshold of 5 minutes was chosen specifically because every built-in non-`seconds` profile already meets it.

**Why 5 minutes:** `stale` is the longest an entry may still be served before being treated as expired. Below 5 minutes, App Shell eviction + refetch happens often enough that the user-visible behavior of "shell then content" flips to "shell then loading" — at which point the App Shell's optimization is purely overhead. 5 minutes is the lowest value at which the optimization still pays for itself in the common case.

**Audit:**

```bash
# Find every cacheLife profile in your project
rg -l 'cacheLife\s*\(' app/ -g '*.{ts,tsx}'

# For each, check the stale value — anything below 300s is now excluded from App Shells
rg "cacheLife\(['\"]seconds['\"]\)|cacheLife\(\s*\{" app/ -g '*.{ts,tsx}' -A2
```

**Sources:**
- [PR #95833 — `Exclude stale under 5 minutes from app shells`](https://github.com/vercel/next.js/pull/95833) · Commit `83e99f0a2e` · gnoff · merged 2026-07-15T21:43:49Z · canary.87
- Related: `MIN_PREFETCHABLE_STALE` (renamed from `DYNAMIC_STALE` in [PR #95361](https://github.com/vercel/next.js/pull/95361), canary.74) is the **30s** threshold for a different question — whether an entry can be included in a partial-prefetch App Shell. The canary.87 threshold is **300s** (5 min) for whether the entry can be served from the App Shell at all. Both thresholds work in tandem: `MIN_PREFETCHABLE_STALE` is the floor for partial-prefetch eligibility, the canary.87 threshold is the floor for App Shell retention.


### `partialFallback` Re-Gated Behind `partialPrefetching` (SHIPPED in `16.3.0-canary.94` 2026-07-23T00:02:38Z, [PR #96074](https://github.com/vercel/next.js/pull/96074) by gnoff, merged 2026-07-22T21:06:42Z)

`partialFallback` is an internal flag in the **build output** that controls ISR behavior for Cache Components apps: when enabled, pages that aren't generated at build time get upgraded to a full ISR entry on the first request. That's a useful optimization for a known traffic pattern, but it had a cost-side footgun: **even a prefetch request was sufficient to trigger the upgrade**, which could cause an explosion in ISR costs in apps where prefetch fanout is wider than real navigation fanout (the common case).

**The fix (PR #96074, gnoff, merged 2026-07-22T21:06:42Z):** the `partialFallback` flag is **turned back off by default** until a new build-output configuration can differentiate between a shell-prefetch request (no `prefetch` prop on the link) and an actual prefetch (`prefetch={true}`).

**The carve-out for `experimental.partialPrefetching: true` apps:** when a project opts into Partial Prefetching, **`partialFallback` stays `true`**. Why: in a Partial Prefetching app, per-link prefetch requests are always opt-in (`prefetch={true}` required to trigger a real fetch). So even without the new ISR-differentiation mechanism, the upgrade cost is bounded — there's at most **one request per distinct route** (not per distinct URL), which is the right granularity. That single-request-per-route upgrade is left in place until the new mechanism lands.

**Practical impact for `cacheComponents: true` apps:**

- **Default (`cacheComponents: true` without `experimental.partialPrefetching: true`):** prefetch traffic no longer triggers ISR upgrades. Cache-miss + ISR-upgrade path only fires on real navigations. If you saw unexpected ISR-cache growth on a non-PP project, upgrade to canary.94 and verify the size stays bounded.
- **With `experimental.partialPrefetching: true`:** unchanged — `partialFallback` remains `true`, the bounded one-request-per-route upgrade continues. Wait for the follow-up build-output config to differentiate shell vs full prefetches (not yet designed; PR #96074 is a safety brake, not the final shape).
- **With `experimental.partialPrefetching: false` (or unset) + `cacheComponents: true`:** prefetch-no-upgrade is the new default. If you depended on the prefetch-triggered-upgrade behavior, opt in to `partialPrefetching: true`.

**No public API change, no `next.config.ts` change.** The flag is in the build output, not the user-visible config.

**Source:** [PR #96074 — `Gate partialFallback behavior behind partialPrefetching flag`](https://github.com/vercel/next.js/pull/96074) · gnoff · merged 2026-07-22T21:06:42Z · **Shipped in `16.3.0-canary.94`** (2026-07-23T00:02:38Z).


### `next-cache-components-adoption` Skill Refinements + Dev-Only Validation Sweep (SHIPPED in `16.3.0-canary.94` 2026-07-23T00:02:38Z, [PR #95817](https://github.com/vercel/next.js/pull/95817) by aurorascharff + [PR #96057](https://github.com/vercel/next.js/pull/96057) by aurorascharff, merged 2026-07-22T14:08:08Z + 15:01:18Z)

Two coordinated refinements to the **adoption skills** shipped in canary.94 — both aimed at improving first-attempt success rates when an agent drives Cache Components or Partial Prefetching adoption against a real app.

**PR #95817 — `Refine Cache Components and Partial Prefetching adoption skills`** (aurorascharff, 2026-07-22T14:08:08Z, 11 files +52/-52 across `docs/01-app/02-guides/{adopting-partial-prefetching, instant-navigation, runtime-prefetching}.mdx` + `docs/01-app/03-api-reference/01-directives/use-cache.mdx` + `docs/01-app/03-api-reference/03-file-conventions/02-route-segment-config/instant.mdx` + `skills/next-cache-components-adoption/{SKILL.md, references/per-page-decisions.md}` + `skills/next-cache-components-optimizer/{SKILL.md, reference/patterns.md, test-template.md}` + `skills/next-partial-prefetching-adoption/SKILL.md`):

- **PPF skill — URL-data sweep reworked to feature-by-feature with a resumable hand-off.** The old "scan everything, fix everything in one pass" was too aggressive for large apps; the new flow walks one feature at a time and records state so an agent can resume after an interruption. The `## requires` section now says unfinished Cache Components adoption **does not gate** the PPF skill — Prerender insights are non-blocking dev signals expected on a fresh branch; only a build-blocking failure stops the run.
- **PPF skill — install `next-dev-loop` without asking, and verify a real blocker before falling back.** Removes the permission prompt that previously interrupted agents in sandboxed environments; the skill now installs `next-dev-loop` automatically when it's missing.
- **CC skill — dropped the hard-coded `agent-browser` version pin.** Replaced with the documented minimum (0.27) so the skill stays correct as `agent-browser` ships new versions without forcing skill updates.
- **CC skill — updated the PPF cross-reference** to the current enable-first flow (the prior cross-reference pointed at an outdated sequence).
- **Docs — `runtime-prefetching`:** the per-card server cost of `allow-runtime` grids is now correctly framed as specific to `prefetch={true}`, with hover-triggered prefetch as the mitigation. (Previously the cost framing applied to all prefetches, which overstated the impact.)
- **Docs — `use-cache`:** the request-API restriction now follows the call stack (`next-request-in-use-cache`, can pass build and fail under `next start`) — previously the doc implied it was a hard block at build time, which isn't accurate. The nested-`cacheLife` build error and its fix are now documented inline.
- **Docs — `instant` route segment config:** removed a stale `version: draft` so the page publishes (was preventing the page from showing up in search/docs index).

**PR #96057 — `skill(cc-adoption): add dev-only validation sweep reference`** (aurorascharff, 2026-07-22T15:01:18Z, 2 files +30/-2 in `skills/next-cache-components-adoption/{SKILL.md, references/dev-only-validations.md}`):

- Adds an optional post-adoption reference for the `next-cache-components-adoption` skill — `references/dev-only-validations.md` — that walks the loop of "after build passes, sweep the dev-only insights". The mechanics mirror the tested `next-partial-prefetching-adoption` skill: reuse `next-dev-loop` preflight, load each route in `next dev`, read the Insights tab + dev log, fix.
- The `## Further reading` section now recommends the sweep and drops the standalone Prefetching bullet (the PPF adoption skill already carries that).
- **Why this exists:** after CC adoption the build passes, but dev validation (default `validationLevel: 'warning'`) still raises instant-navigation insights on every page load that never fail the build. The skill had no recipe for sweeping them — this closes that gap as the recommended next step for users who don't want to adopt Partial Prefetching.

**Practical impact for agents driving CC / PPF adoption:**

- **First-attempt success rate improves** — the feature-by-feature PPF sweep, automatic `next-dev-loop` install, and the dev-only validation sweep all address the most common "the skill loops because it didn't quite finish" failure mode.
- **Docs are accurate again** — the `runtime-prefetching` cost framing, the `use-cache` request-API block, and the `instant` route segment config publish-state are all corrected.
- **Migration:** no code change needed. Update your skill pin (`npx skills update`) after upgrading `next` to canary.94.

**Sources:**

- [PR #95817 — `Refine Cache Components and Partial Prefetching adoption skills`](https://github.com/vercel/next.js/pull/95817) · aurorascharff · merged 2026-07-22T14:08:08Z · **Shipped in `16.3.0-canary.94`** (2026-07-23T00:02:38Z).
- [PR #96057 — `skill(cc-adoption): add dev-only validation sweep reference`](https://github.com/vercel/next.js/pull/96057) · aurorascharff · merged 2026-07-22T15:01:18Z · **Shipped in `16.3.0-canary.94`** (2026-07-23T00:02:38Z).


### `experimental.appShells` Flag Removed — Behavior Unified With Partial Prefetching (16.3.0-canary.88+, PR [#95415](https://github.com/vercel/next.js/pull/95415) by acdlite, merged 2026-07-16T20:30:58Z, 343 +/-381 over 45 files)

**⚠️ Config change:** The `experimental.appShells` config flag was an internal development flag, not meant as a public-facing option. It is now **removed entirely** from `packages/next/src/server/config-schema.ts` + `packages/next/src/server/config-shared.ts` + `packages/next/src/server/base-server.ts` + the `app-page` + `edge-ssr-app` build templates + every segment-cache prefetch test fixture (`prefetch-app-shell`, `prefetch-app-shell-global-eager`, `prefetch-fallback-retry`, `search-params`, `staleness`, `vary-params`, `sub-shell-generation-middleware`, `parallel-route-navigations`, `instant-navigation-testing-api/partial-prefetch`, `instant-navigation-testing-api/root-params`, `use-cache-non-deterministic-args`). The behavior App Shells gated is now split two ways:

- **Pure optimizations** (e.g. prefetched-shell reuse on cross-page navigation, layout-state preservation across shell hops) → **land unconditionally**. No opt-in required; no measurable prefetch-cost regression for apps that don't otherwise change.
- **Behaviors that change the number of requests the server sees** (e.g. one request for the shell + a separate request for the page data, allowing the shell to render before the route data resolves) → **gated behind `experimental.partialPrefetching: true`** (or per-segment `export const prefetch = 'partial'` / `'unstable_eager'`). This is the existing Partial Prefetching flag — it's already the canonical opt-in for "I want the new SPA-style prefetch model".

**Net effect for users:**

| Before canary.88 | After canary.88 |
|---|---|
| Set `experimental.appShells: true` to get the full behavior. | Set `experimental.partialPrefetching: true` to get the same full behavior. |
| `experimental.appShells: false` or unset → App Shell internals inert. | No flag needed → App Shell internals run as pure optimizations (no extra requests); the request-splitting bit is gated behind Partial Prefetching only. |
| `experimental.appShells: true` + `experimental.partialPrefetching: false` → App Shells fully on, no PP. | **Not possible** — to get the request-splitting behavior you must opt into PP (which is a superset). |
| No flag set → legacy full-route prefetch. | No flag set → legacy full-route prefetch. |

**Why this matters for skill users:** the canary.87 commit message and the partial-prefetching docs both alluded to "App Shells" as a separate concept; from canary.88 onward there is **only one knob** (`experimental.partialPrefetching`), and App Shells is just the runtime stage of Partial Prefetching, not a separate feature. Anyone who copied `experimental.appShells: true` from older 16.3 canary docs (June 2026 era) should:

```diff
 // next.config.ts
 const config: NextConfig = {
-  experimental: {
-    appShells: true,            // ❌ no longer recognized in canary.88+
-  },
+  experimental: {
+    partialPrefetching: true,   // ✅ canonical opt-in for the new prefetch model
+    // (or use `export const prefetch = 'partial'` per-segment)
+  },
 }
```

If you leave `appShells: true` in, canary.88+ will warn about an unrecognized `experimental` key (the AGENTS.md managed block added in PR #95825 now actively steers agents away from fabricating Turbopack-related config; the same instinct applies here — the old flag is gone).

**Migration audit:**

```bash
# Find every project that still has the dead flag
rg -l "appShells\s*:\s*true" next.config.* apps/*/next.config.*

# For each match: replace with partialPrefetching: true (or remove the block entirely
# if you didn't actually need the request-splitting behavior — most projects don't).
```

**Interaction with the canary.87 5-minute-stale floor (PR #95833):** unchanged. The `cacheLife('seconds')` exclusion from App Shells still applies when you're on Partial Prefetching; the `appShells` flag's removal doesn't relax that rule. The exclusion is now documented as "App Shells on Partial Prefetching routes" rather than "App Shells under the `experimental.appShells` flag".

**Sources:**
- [PR #95415 — `Unify appShells flag with Partial Prefetching`](https://github.com/vercel/next.js/pull/95415) · acdlite / Andrew Clark · merged 2026-07-16T20:30:58Z · canary.88 · commit `bc48ef7` (45 files, +343/-381)
- [PR #95825 — `AGENTS.md` now warns agents not to fabricate `next.config.js` options](https://github.com/vercel/next.js/pull/95825) · sampoder · merged 2026-07-15T18:57:46Z (relevant: confirms `experimental.appShells` is no longer a recognized key in the experimental schema)
- [PR #95833 — `Exclude stale under 5 minutes from app shells`](https://github.com/vercel/next.js/pull/95833) · canary.87 (the upstream rule that PR #95415 inherits; details above)
- [Partial Prefetching docs](https://nextjs.org/docs/app/guides/partial-prerendering) — the canonical opt-in for the new prefetch model that PR #95415 makes the home of the "full" App Shells behavior

## App Shell Prefetch — Next.js 16.3+ (Canary Feature)

**⚠️ Status:** App Shell Prefetch is a **Next.js 16.3 canary feature** — not yet stable. The current stable release (**Next.js 16.2.9**) still defaults to `prefetch="full"` (full route prefetch on hover). This section describes what is coming when 16.3 stabilizes.

**What changes in 16.3:** Next.js 16.3 will change the **default prefetch behavior** for `<Link>` components. Currently, hovering a link triggers a full route prefetch (all segments). In 16.3, the default becomes **App Shell prefetching** — only the layout, loading UI, and critical shell assets are prefetched, while route segment data is deferred until actual navigation.

**Why App Shell prefetch?**
- Reduces unnecessary data transfer on hover (only prefetches the shell, not full page data)
- Full page is fetched on click/navigation — same end result, less wasted prefetch bandwidth
- Particularly beneficial for data-heavy pages where full prefetch was expensive

**Prefetch configuration options (Next.js 16.3+ canary — available now in canary, default when 16.3 stabilizes):**

| Prefetch Mode | Behavior | Use When |
|---|---|---|
| `prefetch="app-shell"` (16.3 default when stable) | Only App Shell prefetched on hover | Most cases — cheaper, sufficient for fast navigation |
| `prefetch="full"` | Entire route prefetched on hover | Critical user journeys (checkout, sign-up) |
| `prefetch={false}` | No prefetch | Rarely-used links, authenticated pages |

**Current stable behavior (Next.js 16.2.9):** The default is `prefetch="full"`. You can manually opt into App Shell prefetch:

```tsx
// App Shell prefetch — opt in on Next.js 16.2.x
<Link href="/blog/my-post" prefetch="app-shell">Read more</Link>

// Full prefetch for high-value conversion paths (default in 16.2.x)
<Link href="/checkout" prefetch="full">Checkout now</Link>

// No prefetch — for rarely-accessed or authenticated routes
<Link href="/admin/settings" prefetch={false}>Settings</Link>
```

**Programmatic prefetch with custom granularity (Next.js 16.3+):**

```tsx
import { useRouter } from 'next/navigation'

function PrefetchButton({ href }: { href: string }) {
  const router = useRouter()

  function handleHover() {
    // Full prefetch on hover for this specific link (16.3+ kind option)
    router.prefetch(href, { kind: 'full' })
  }

  return <Link href={href} onMouseEnter={handleHover}>{href}</Link>
}
```

**`use cache` directive — improved deduping (Next.js 16.3 canary):**

Next.js 16.3 canary improves deduping of concurrent `use cache` invocations. When the same cached function is called multiple times during a single request (e.g., from multiple components), Next.js 16.3 properly deduplicates the calls — only one execution, shared result. **This improvement is available in 16.3 canary only; on 16.2.x, concurrent calls may execute multiple times.**

```tsx
// Before Next.js 16.3 — could execute twice if called from two components simultaneously
// Next.js 16.3+ — deduplicates concurrent calls, executes once
import { cache } from 'react'

const getUser = cache(async (id: string) => {
  'use cache'
  return await db.user.findUnique({ where: { id } })
})

// Called from two components simultaneously — only one DB query in 16.3+
const user1 = await getUser('user-123')
const user2 = await getUser('user-123') // Returns cached result from first call
```

**When this matters:**
- Data-heavy pages with multiple components reading the same cached data
- Reducing database load on navigation
- Particularly impactful in dashboard/feed patterns where many components share reference data (16.3 canary only)

**`prefetch` prop on Route Segments (Next.js 16.3+):**

Route segment config also supports prefetch control at the route level (available in 16.3 canary):

```tsx
// app/dashboard/page.tsx
export const prefetch = 'app-shell' as const // or 'full' | false
```


This sets the default prefetch behavior for all `<Link>` components pointing to this route.

**Sources:**
- [Next.js 16.3.0-canary release notes](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.50)
- [Next.js prefetching guide](https://nextjs.org/docs/app/guides/prefetching)
- [Next.js 16.3.0-canary.26 — App Shell server response](https://newreleases.io/project/npm/next/release/16.3.0-canary.26)
- [Next.js 16.3.0-canary.1 — Partial prefetch default](https://newreleases.io/project/npm/next/release/16.3.0-canary.1)

### `unstable_instant` — REMOVED in 16.3.0-preview.0 (June 9, 2026)

**⚠️ Breaking change (16.3+):** `unstable_instant` was **removed** in `16.3.0-preview.0`. Instant insights now default to `validationLevel: 'warning'` (validates every navigation in dev, no opt-in needed). The previous AI agent hints instructing agents to export `unstable_instant` to surface feedback are obsolete and have been deleted from the docs ([linking-and-navigating, caching, fetching-data, streaming, instant-navigation, loading](https://github.com/vercel/next.js/pull/94577)).

**What replaced it:**
- The dev server now runs instant-navigation validation by default — you'll see the Cold cache indicator / insights badge in the dev overlay whenever a navigation wasn't served in a production-representative way, with no route-level export required.
- The behavior is the same: served from prefetch cache instantly, RSC response patches dynamic holes in the background.
- The Cold cache indicator is **scoped to shell cache misses only** (16.3.0-canary.57, [#94911](https://github.com/vercel/next.js/pull/94911)) — initial loads count cache misses up to the static shell, runtime-prefetch navigations count up to the runtime shell. Reads that resolve after the shell no longer count, so warm cache hits no longer incorrectly show the badge.

**Migration:** Just delete any `export const unstable_instant = true` lines from your routes. The default dev validation now does the same thing with no opt-in.

### `prefetch` segment config — `allow-runtime` + stabilization (16.3.0-preview.0)

**Renamed ([#94568](https://github.com/vercel/next.js/pull/94568)):** The runtime-prefetch option on `export const prefetch` was renamed from `force-runtime` to `allow-runtime`. The semantic shift: `allow-runtime` conveys "this segment is designed for and makes sense to runtime prefetch" (a property of the segment), not "force runtime prefetching right now" (a runtime choice). Future runtime-prefetch optimizations can skip a segment without breaking, and `force-runtime` can be re-added later as a codemod-recoverable option if needed.

**Stabilized ([#94571](https://github.com/vercel/next.js/pull/94571)):** `export const prefetch` is now a stable API in Next.js 16.3. Some individual option values may still be marked `unstable_` (e.g. `unstable_allow_runtime`), but the segment-config export itself ships stable.

```tsx
// app/dashboard/page.tsx — 16.3 stable segment config
export const prefetch = 'app-shell' as const    // default: only app shell on hover
export const prefetch = 'full' as const         // whole route on hover (high-value paths)
export const prefetch = false                   // no prefetch (auth, admin, rare)
export const prefetch = 'allow-runtime' as const // opt in to runtime prefetch (renamed from force-runtime)
```

**Programmatic prefetch with new options (16.3+):**

```tsx
'use client'
import { useRouter } from 'next/navigation'

function PrefetchButton({ href }: { href: string }) {
  const router = useRouter()

  function handleHover() {
    router.prefetch(href, { kind: 'full' })            // full route
    // router.prefetch(href, { kind: 'allow-runtime' }) // runtime prefetch (new option)
    // router.prefetch(href, { kind: 'app-shell' })     // app shell only
  }

  return <Link href={href} onMouseEnter={handleHover}>{href}</Link>
}
```

**Sources:**
- [Next.js 16.3.0-preview.0 release notes](https://github.com/vercel/next.js/releases/tag/v16.3.0-preview.0)
- [PR #94568 — Rename `force-runtime` to `allow-runtime`](https://github.com/vercel/next.js/pull/94568)
- [PR #94571 — Stabilize `export const prefetch`](https://github.com/vercel/next.js/pull/94571)
- [PR #94577 — Remove `unstable_instant` agent hints; insights validate by default](https://github.com/vercel/next.js/pull/94577)
- [PR #94911 — Scope the Cold cache indicator to shell cache misses](https://github.com/vercel/next.js/pull/94911)
- [Next.js prefetching guide](https://nextjs.org/docs/app/guides/prefetching)
- [Next.js instant navigation guide](https://nextjs.org/docs/app/guides/instant-navigation)

## Search with Debounce + URL Sync + React Query

```tsx
// components/post-search.tsx
'use client'

import { useCallback, useEffect } from 'react'
import { useRouter, useSearchParams } from 'next/navigation'
import { useQuery } from '@tanstack/react-query'
import { useDebouncedValue } from '@/hooks/use-debounced-value'

export function PostSearch() {
  const router = useRouter()
  const searchParams = useSearchParams()
  const query = searchParams.get('q') ?? ''
  
  // Debounce input to avoid excessive API calls
  const [debouncedQuery, isDebouncing] = useDebouncedValue(query, 300)
  
  // React Query for caching + loading
  const { data: results, isLoading } = useQuery({
    queryKey: ['posts', 'search', debouncedQuery],
    queryFn: () => searchPosts(debouncedQuery),
    enabled: debouncedQuery.length > 2,  // Don't search until 3+ chars
  })
  
  // Sync input to URL
  function handleSearch(e: React.ChangeEvent<HTMLInputElement>) {
    const value = e.target.value
    const params = new URLSearchParams(searchParams)
    if (value) {
      params.set('q', value)
    } else {
      params.delete('q')
    }
    router.push(`?${params.toString()}`, { scroll: false })
  }
  
  return (
    <div>
      <input 
        value={query} 
        onChange={handleSearch}
        placeholder="Search posts..."
      />
      {isLoading && <Spinner />}
      {results && <PostList posts={results} />}
    </div>
  )
}
```

### Debounce Hook

```ts
// hooks/use-debounced-value.ts
import { useState, useEffect } from 'react'

export function useDebouncedValue<T>(value: T, delay: number): [T, boolean] {
  const [debouncedValue, setDebouncedValue] = useState(value)
  const [isDebouncing, setIsDebouncing] = useState(false)
  
  useEffect(() => {
    setIsDebouncing(true)
    const timer = setTimeout(() => {
      setDebouncedValue(value)
      setIsDebouncing(false)
    }, delay)
    
    return () => clearTimeout(timer)
  }, [value, delay])
  
  return [debouncedValue, isDebouncing]
}
```

## Multi-Step Form with React Hook Form + Zod

```tsx
// components/multi-step-form.tsx
'use client'

import { useState } from 'react'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { Button } from '@/components/ui/button'
import { Form } from '@/components/ui/form'

const step1Schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Min 8 characters'),
})

const step2Schema = z.object({
  firstName: z.string().min(1),
  lastName: z.string().min(1),
  age: z.coerce.number().int().min(18),
})

const fullSchema = step1Schema.merge(step2Schema)
type FormData = z.infer<typeof fullSchema>

export function MultiStepForm() {
  const [step, setStep] = useState(1)
  const form = useForm<FormData>({
    resolver: zodResolver(step === 1 ? step1Schema : fullSchema),
    mode: 'onChange',
  })
  
  async function handleNext() {
    const fields = step === 1 ? ['email', 'password'] : ['firstName', 'lastName', 'age']
    const valid = await form.trigger(fields as any)
    if (valid) setStep(s => s + 1)
  }
  
  async function handleSubmit(data: FormData) {
    await fetch('/api/register', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    })
  }
  
  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(handleSubmit)}>
        {step === 1 && (
          <fieldset className="space-y-4">
            <Form.Field control={form.control} name="email">
              <Form.Item>
                <Form.Label>Email</Form.Label>
                <Form.Control><input {...form.register('email')} /></Form.Control>
                <Form.Message />
              </Form.Item>
            </Form.Field>
            <Form.Field control={form.control} name="password">
              <Form.Item>
                <Form.Label>Password</Form.Label>
                <Form.Control><input type="password" {...form.register('password')} /></Form.Control>
                <Form.Message />
              </Form.Item>
            </Form.Field>
            <Button type="button" onClick={handleNext}>Next</Button>
          </fieldset>
        )}
        
        {step === 2 && (
          <fieldset className="space-y-4">
            <Form.Field control={form.control} name="firstName">
              <Form.Item><Form.Label>First Name</Form.Label>
                <Form.Control><input {...form.register('firstName')} /></Form.Control>
                <Form.Message />
              </Form.Item>
            </Form.Field>
            <Form.Field control={form.control} name="lastName">
              <Form.Item><Form.Label>Last Name</Form.Label>
                <Form.Control><input {...form.register('lastName')} /></Form.Control>
                <Form.Message />
              </Form.Item>
            </Form.Field>
            <Form.Field control={form.control} name="age">
              <Form.Item><Form.Label>Age</Form.Label>
                <Form.Control><input type="number" {...form.register('age')} /></Form.Control>
                <Form.Message />
              </Form.Item>
            </Form.Field>
            <div className="flex gap-2">
              <Button type="button" variant="outline" onClick={() => setStep(1)}>Back</Button>
              <Button type="submit">Register</Button>
            </div>
          </fieldset>
        )}
      </form>
    </Form>
  )
}
```

## Server Component → Client Component Data Passing

### Pattern: Pass Serializable Data

```tsx
// app/users/[id]/page.tsx — server component
import { db } from '@/lib/db'
import { UserProfileClient } from '@/components/user-profile-client'

export default async function UserPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const user = await db.user.findUnique({ where: { id } })
  
  if (!user) notFound()
  
  // Pass plain serializable data to client component
  return (
    <UserProfileClient 
      user={{
        id: user.id,
        name: user.name,
        email: user.email,
        role: user.role,
        joinedAt: user.createdAt.toISOString(),  // Convert Date to string
      }}
    />
  )
}
```

```tsx
// components/user-profile-client.tsx
'use client'

import { useState } from 'react'

interface UserProfileProps {
  user: {
    id: string
    name: string
    email: string
    role: string
    joinedAt: string
  }
}

export function UserProfileClient({ user }: UserProfileProps) {
  const [following, setFollowing] = useState(false)
  // ...
}
```

### Pattern: Pass a Promise (React 19 `use()`)

```tsx
// app/users/[id]/page.tsx — server component
import { getUser } from '@/lib/api'

export default async function UserPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  
  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserProfile userPromise={getUser(id)} />
    </Suspense>
  )
}
```

```tsx
// components/user-profile.tsx — client component
'use client'

import { use } from 'react'

export function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise)  // Suspends until resolved
  return <div>{user.name}</div>
}
```

## Infinite Scroll with React Query

```tsx
// components/post-feed.tsx
'use client'

import { useInfiniteQuery } from '@tanstack/react-query'
import { useCallback } from 'react'
import { useInView } from 'react-intersection-observer'

export function PostFeed() {
  const { ref, inView } = useInView()
  
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam = 0 }) => fetchPosts({ cursor: pageParam }),
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    initialPageParam: 0,
  })
  
  useEffect(() => {
    if (inView && hasNextPage) {
      fetchNextPage()
    }
  }, [inView, hasNextPage, fetchNextPage])
  
  return (
    <div>
      {data?.pages.flatMap(page => page.posts).map(post => (
        <PostCard key={post.id} post={post} />
      ))}
      
      <div ref={ref}>
        {isFetchingNextPage && <Spinner />}
        {!hasNextPage && <p>No more posts</p>}
      </div>
    </div>
  )
}
```

## Optimistic UI with `useOptimistic` (React 19)

React 19's `useOptimistic` provides a declarative way to show immediate UI feedback while a mutation is pending:

```tsx
'use client'

import { useOptimistic } from 'react'
import { updatePost } from '@/app/actions'

interface Post {
  id: string
  content: string
  likes: number
}

export function LikeButton({ post }: { post: Post }) {
  const [optimisticPost, addOptimistic] = useOptimistic(
    post,
    (state, newLikes: number) => ({ ...state, likes: newLikes })
  )

  async function handleLike() {
    addOptimistic(post.likes + 1)
    try {
      await updatePost(post.id, { likes: optimisticPost.likes + 1 })
    } catch {
      // React reverts on error automatically
    }
  }

  return (
    <button onClick={handleLike}>
      {optimisticPost.likes} likes
    </button>
  )
}
```

**When to use `useOptimistic` vs React Query's optimistic update:**
- `useOptimistic` — simpler, for single-item optimistic updates (likes, follows, toggles)
- React Query `onMutate` — for complex list mutations (add/remove from lists), full rollback control

## File Upload with Preview

```tsx
// components/image-upload.tsx
'use client'

import { useState } from 'react'
import { useForm } from 'react-hook-form'
import Image from 'next/image'

export function ImageUpload() {
  const [preview, setPreview] = useState<string | null>(null)
  const [uploading, setUploading] = useState(false)
  
  function handleFileChange(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0]
    if (!file) return
    
    // Create preview
    const url = URL.createObjectURL(file)
    setPreview(url)
  }
  
  async function handleUpload(e: React.ChangeEvent<HTMLFormElement>) {
    e.preventDefault()
    const formData = new FormData(e.currentTarget)
    const file = formData.get('file') as File
    
    setUploading(true)
    try {
      const res = await fetch('/api/upload', {
        method: 'POST',
        body: formData,
      })
      const { url } = await res.json()
      console.log('Uploaded to:', url)
    } finally {
      setUploading(false)
    }
  }
  
  return (
    <form onSubmit={handleUpload}>
      <input type="file" name="file" accept="image/*" onChange={handleFileChange} />
      {preview && (
        <div className="mt-4 relative w-64 h-64">
          <Image src={preview} alt="Preview" fill className="object-cover" />
        </div>
      )}
      <button type="submit" disabled={uploading}>
        {uploading ? 'Uploading...' : 'Upload'}
      </button>
    </form>
  )
}
```


## Deferred Operations with `after()` (Next.js 16)

Next.js 15+ introduces `after()` from `next/server` — a way to run code **after** the response is sent to the user. This is ideal for non-critical operations that shouldn't block the response.

```tsx
// app/dashboard/page.tsx
import { after } from 'next/server'
import { logAnalytics, sendSlackNotification } from '@/lib/analytics'

export default async function DashboardPage() {
  const data = await getDashboardData()
  
  // Run AFTER the response is sent — doesn't delay the page
  after(async () => {
    await logAnalytics({ 
      page: '/dashboard', 
      userId: data.user.id,
      timestamp: new Date().toISOString()
    })
  })
  
  after(async () => {
    if (data.user.isNew) {
      await sendSlackNotification(`New user signed up: ${data.user.email}`)
    }
  })
  
  return <Dashboard data={data} />
}
```

**When to use `after()`:**

| Use Case | Example |
|---|---|
| Analytics / logging | `logAnalytics()`, `trackEvent()` |
| Notifications | Send Slack/email after user action |
| Cache warming | Pre-fetch related data after page load |
| Background jobs | Trigger async jobs without blocking response |

**Rules:**
- `after()` runs **after** the response is streamed — the user sees the page immediately
- It still runs on the server — it has access to server-only resources (DB, filesystem)
- If `after()` throws, the page still renders normally — errors don't affect the user
- Multiple `after()` calls run in parallel

```tsx
// Real-world: log page view without slowing down response
import { after } from 'next/server'

export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  const post = await getPost(slug)
  
  after(async () => {
    // Non-blocking — doesn't affect TTFB
    await fetch('/api/analytics', {
      method: 'POST',
      body: JSON.stringify({ path: `/blog/${slug}`, referrer: headers().get('referer') }),
    })
  })
  
  return <Article post={post} />
}
```

**`after()` vs alternatives:**

| Pattern | Use When |
|---|---|
| `after()` (Next.js 15) | Server-side ops after response, inside Next.js |
| `queue` (Redis/Bull) | Background jobs that need durability across restarts |
| `useEffect` (client) | Client-side ops like analytics after hydration |

### `MIN_PRERENDERABLE_EXPIRE` / `MIN_PREFETCHABLE_STALE` Constant Rename — 16.3.0-canary.74+ ([PR #95361](https://github.com/vercel/next.js/pull/95361))

Next.js 16.3.0-canary.74 renamed two internal constants in `packages/next/src/server/use-cache/constants.ts` for clarity:

| Old name | New name | Value (seconds) | Purpose |
|---|---|---|---|
| `DYNAMIC_EXPIRE` | `MIN_PRERENDERABLE_EXPIRE` | 300 | Maximum `expire` for an entry to be included in a static prerender (PPR HTML shell). Smaller expire → more dynamic. |
| `DYNAMIC_STALE` | `MIN_PREFETCHABLE_STALE` | 30 | Maximum `stale` for an entry to be included in a runtime prefetch (partial-prefetching App Shell). Smaller stale → more dynamic. |

**Pure rename, no behaviour change.** Every comparison site changes from `expire < DYNAMIC_EXPIRE` → `expire < MIN_PRERENDERABLE_EXPIRE` and `stale < DYNAMIC_STALE` → `stale < MIN_PREFETCHABLE_STALE` — the values (300s / 30s) are identical and the semantics at every site are identical.

**Why the rename matters (semantic, not functional):**

The old names were confusing because the constants act as the *exclusive upper bound of the dynamic range* — i.e., the **smallest value at which an entry is treated as dynamic**. Reading `expire < DYNAMIC_EXPIRE` and parsing "if the entry is below the dynamic threshold" requires the reader to do an inversion in their head. The new names read correctly at every comparison site: `expire < MIN_PRERENDERABLE_EXPIRE` parses as "if the entry is below the minimum prerenderable expire, it's not prerenderable" — direction-of-thinking matches the comparison operator.

**Concretely, this affects `'use cache'` semantics when paired with `cacheComponents: true`:**

- A `'use cache'` function with `cacheLife('hours')` (default `expire = 3600` seconds) is *above* `MIN_PRERENDERABLE_EXPIRE` (300) → included in the static prerender shell.
- A `'use cache'` function with `cacheLife('minutes')` (default `expire = 60`) is *below* `MIN_PRERENDERABLE_EXPIRE` → treated as dynamic (forced into a `<Suspense>` boundary).
- The same threshold logic applies for `stale` vs `MIN_PREFETCHABLE_STALE` (30s) when deciding whether an entry can be served from a partial-prefetch App Shell.

The values are unchanged in canary.74; this is purely a code-clarity refactor. **If you were searching for "DYNAMIC_EXPIRE" / "DYNAMIC_STALE" in the skill, the new names are what you'll find in canary.74+ source.**

**Source:** [PR #95361 — `Rename DYNAMIC_EXPIRE to MIN_PRERENDERABLE_EXPIRE and DYNAMIC_STALE to MIN_PREFETCHABLE_STALE`](https://github.com/vercel/next.js/pull/95361) · Files: `packages/next/src/server/use-cache/constants.ts` (+2/-2)


### Short-`expire` `'use cache'` Dev-Reload Behavior — `MIN_PRERENDERABLE_EXPIRE` Floor in `next dev` (16.3.0-canary.75+, [PR #95362](https://github.com/vercel/next.js/pull/95362) by unstubbable/Hendrik Liebau, merged 2026-07-02T09:26:09Z)

A `'use cache'` value with an explicit short `expire` — typically `cacheLife({ expire: 0 })`, `cacheLife({ expire: 1 })`, or the built-in `'seconds'` profile (`expire = 1`) — was treated as a miss on every dev reload by both the built-in default handler AND custom handlers. Reloads re-ran the cache function and streamed slowly, because `expire` is the entry's expiration bound (the longest it may still be served before being treated as expired), and the dev handler had no window in which a reload could be served from the cache when `expire` was zero or one.

**Why this happened:** `expire` is the entry's expiration bound in both dev and production; what differs is which threshold the built-in in-memory handler enforces. In `next dev` it serves stale entries up to `expire` to keep reloads fast; in production it drops them earlier, once past `revalidate`. An `expire` of zero therefore leaves the dev handler no window for a reload to be served from the cache, and the wrapper's serve-vs-regenerate check (which also keys on `expire`) regenerates instead.

**Fix ([PR #95362](https://github.com/vercel/next.js/pull/95362)):** the built-in default handler now retains an entry for at least `MIN_PRERENDERABLE_EXPIRE` (300s) in dev, a minimum the custom front handler inherits by being a built-in default handler, and the wrapper applies the same minimum when deciding whether to serve or regenerate. That affects the **retain** and **serve** decisions only — the entry keeps its real `expire`, so the staged dev render still resolves it at the appropriate stage (not the shell stage). A short-`expire` entry is also **re-warmed in the background** on every dynamic-request render, so a reload serves the previously cached value immediately and the freshly recomputed one appears on the next reload. This is the same stale-while-revalidate trade-off already accepted for `'use cache: private'` and `cacheMaxMemorySize: 0` dev optimizations, which likewise favor a fast reload over serving a value these configurations would not otherwise cache at all.

**What still differs from non-short `expire`:**

- **Production behavior is unchanged.** Entries with `expire: 0` are still regenerated on every read (see PR #95363 below).
- **Stale-while-revalidate applies.** The re-warmed entry appears one reload later, not on the current one. The reload-serves-old behavior is the same as `'use cache: private'` and `cacheMaxMemorySize: 0`.

**Why `MIN_PRERENDERABLE_EXPIRE` and not `revalidate: 0`?** Forcing `revalidate: 0` would leak into the cache life propagated to an enclosing `'use cache'` and trip the nested-dynamic error with the wrong message. The `expire` floor is local to the entry, doesn't change its resolved cache life, and keeps the dev optimization invisible to nested-cache detection.

**Why not just keep the resolved life and rely on existing logic?** Because a short `expire` is exactly what makes the dev handler drop the entry — the minimum retention is needed at the handler level. The `MIN_PRERENDERABLE_EXPIRE` floor is the smallest change that makes the dev reload fast without altering production semantics.

**Companion PR #95363 ([Skip saving `expire: 0` values in the default cache handler in prod](https://github.com/vercel/next.js/pull/95363), canary.76+, merged 2026-07-02T21:53:18Z):** the built-in default cache handler now skips the `set()` for `expire: 0` entries in production (gated on `!process.env.__NEXT_DEV_SERVER`). The entry is regenerated on every read regardless, so persisting it is wasted work; for a remote handler it's a needless round-trip and stored payload. Dev keeps storing because the dev handler's minimum retention (#95362) serves the previously cached value across reloads while the entry re-warms in the background. New `use-cache-default-handler-expire-zero` e2e test enables `NEXT_PRIVATE_DEBUG_CACHE` and asserts on the handler's `set()` decisions; skips deployment.

**Companion PR #95373 ([Fix false-positive nested-cache error for a short default profile](https://github.com/vercel/next.js/pull/95373), canary.76+, merged 2026-07-02T21:53:18Z):** overriding the `default` `cacheLife` profile with a short cache life (the built-in `default` profile's `expire` is `INFINITE_CACHE`) makes every `'use cache'` without an inline `cacheLife()` inherit that short life. In this configuration the nested-`'use cache'` error (short-`expire` and zero-`revalidate` variants) fired in two cases where it shouldn't: (1) a single non-nested `'use cache'` already resolved to a short life with nothing nested, killing the build in prod and throwing on the request in dev; (2) for a genuinely nested cache, the warning was pointless because the default profile already makes every cache a dynamic hole. Fix requires that a dynamic nested cache actually propagated its short life upward (`dynamicNestedCacheError` is set) AND that the `default` profile is itself prerenderable. Otherwise the short-lived entry is omitted as a dynamic hole instead, exactly as an inline short `cacheLife()` already was. The dev-only private-cache exclusion (`kind !== 'private'` check) is subsumed — a private cache's `revalidate: 0` is forced by us rather than by a nested cache and never carries a nested error.

**Why this matters for agents:** if a `'use cache'` function looks "fine" in production but feels slow on `next dev` reloads, the most likely cause was a short `expire` (or any cache with `expire: 0` / `'seconds'` profile) without a minimum-retention floor. Pre-canary.75 this was a known sharp edge — every reload re-ran the function. Post-canary.75 it's a no-op. **Audit:**

```bash
rg -l 'cacheLife\s*\(\s*\{[^}]*expire:\s*0' app/ -g '*.{ts,tsx}'
rg -l 'cacheLife\s*\(\s*\{[^}]*expire:\s*1' app/ -g '*.{ts,tsx}'
rg -l "cacheLife\(\s*'seconds'\s*\)" app/ -g '*.{ts,tsx}'
```

**Use `cacheLife({ expire: 0 })` deliberately when you want server-side opt-out:**

- **Conditional opt-out** — a function with an otherwise normal cache life sets `expire: 0` for a result that is not worth persisting, e.g. when an upstream fetch errored.
- **Client-only caching** — a value is `expire: 0` by design so that it lives only in the client cache (via Cached Navigations or Runtime Prefetches when the route opts in) and never on the server.

**Sources:**

- [PR #95362 — `Cache short-\`expire\` \`'use cache'\` values across dev reloads`](https://github.com/vercel/next.js/pull/95362) · Commit `2850659b74` (2026-07-02T09:26:08Z) · Hendrik Liebau
- [PR #95363 — `Skip saving \`expire: 0\` values in the default cache handler in prod`](https://github.com/vercel/next.js/pull/95363) · Commit `34dbafb46d` (2026-07-02T21:53:16Z) · Hendrik Liebau
- [PR #95373 — `Fix false-positive nested-cache error for a short default profile`](https://github.com/vercel/next.js/pull/95373) · Commit `b420536fe8` (2026-07-02T21:53:17Z) · Hendrik Liebau

### Instant Navigation Testing API — Deployed Recovery + Full-Page Loads (16.3.0-canary.75+, PR [#95222](https://github.com/vercel/next.js/pull/95222) + [#95227](https://github.com/vercel/next.js/pull/95227) + [#95398](https://github.com/vercel/next.js/pull/95398) by unstubbable, June 29–July 2, 2026)

The Instant Navigation Testing API (controlled by the `__next_instant_test` cookie set by the DevTools) locks navigation to a route's static shell so you can inspect the prefetched shell before any dynamic content streams in. In `next dev` the Instant Navigation DevTools set/clear it; in any environment you can drive it by hand by setting the cookie in browser DevTools. Three companion PRs harden the deployed case:

**(1) PR #95227 — `Recover from blocking routes under Instant Navigation lock when deployed` (canary.75+, merged 2026-07-02T12:51:23Z).** A blocking route (one with a Suspense boundary above `<body>`, or `export const instant = false`) has an empty static shell. When you did a full-page load of such a route while the cookie was set, that empty shell was served as a blank document with no way to release the lock, and every reload rendered the same blank shell and left you stuck.

Previously the handler threw for an empty shell. In dev that surfaced as the error overlay and the catch cleared the cookie so a reload recovered. It did not recover when deployed: the cookie could only be cleared via `Set-Cookie`, which cannot take effect there because the empty shell is served from the ISR cache with its response headers already committed, so the user stayed on the blank shell across reloads. In `next start` it recovered but rendered only a generic "Internal Server Error" page.

Fix: for the empty-shell case we serve a minimal document whose inline script clears the cookie client-side, so the next full-page load renders the route normally; we clear it from the document rather than with `Set-Cookie` because the headers are already committed when the shell is served from the cache. Development still throws so the error overlay shows. As a secondary improvement, `next start` now serves that same document with a readable explanation instead of the generic "Internal Server Error" page. The blocking-route test now asserts the user-facing outcome (after a reload the route renders normally) rather than transport details such as the status code and `Set-Cookie`, and it now runs on deploy as well.

**(2) PR #95222 — `Make Instant Navigation Testing full-page loads work when deployed` (canary.75+, merged 2026-07-02T12:51:22Z).** With the testing API active, a full-page load (reload, MPA navigation, or direct URL entry) on a deployment failed with a minified React hydration error, and releasing the lock then hard-reloaded the page instead of resolving client-side. Client-side navigations under the lock already worked; the gap was full-page loads.

The API works by injecting an inline script that sets `self.__next_instant_test`, which the client reads at `app-index` init as its hydration source. That read only happens on a full-page load; client-side navigations go through the segment cache and the lock and never read it. The script was previously injected at request time through a transform stream, which runs only when the function renders the document. On a deployed full-page load the browser instead parses the prebuilt static prelude served from the edge cache, which was prerendered without the cookie and so carried no script, leaving `self.__next_instant_test` undefined.

When the testing API is enabled (in dev, or in prod via `experimental.exposeTestingApiInProductionBuild`), we embed the cookie-guarded bootstrap into the prerendered prelude through React's `bootstrapScriptContent`, so it lands in the cached static shell and runs before the client bootstrap module. That production-build flag is meant for protected preview environments, so a normal prod build never embeds the script, and even where it is embedded it stays inert on requests without the instant-navigation cookie (the prelude is shared across all requests). The same content is folded into the dynamic render path in `renderToStream`, and the now-redundant request-time transform is removed.

**(3) PR #95398 — `Clear a resurrected instant cookie on unlock so a hard reload recovers` (canary.75+, merged 2026-07-02T12:51:21Z).** Under the Instant Navigation Testing lock, an MPA page load's bootstrap writes the captured cookie value asynchronously via `cookieStore`. When a scope is released (the external actor deletes the cookie), that pending write can land in the narrow gap between the delete and the deleted-event handler that tears the lock down, resurrecting the cookie after the scope has ended. If the unlock then falls back to a hard reload (which happens when the freshly loaded shell has not hydrated the router yet), the reload carries the resurrected cookie, the server serves the shell again, and the page re-enters instant mode with no scope left to release it, so the deferred content never streams in and the navigation hangs.

Once the lock is released, `writeCookieValue`'s guard rejects any further captured write, so the deleted-event handler now clears the cookie right after `releaseLock` to neutralize a write that resurrected it in the gap, and an added guard ignores the re-entrant change event that the clear itself produces.

This removes a family of flaky timeouts in `instant-navigation-testing-api.test.ts`, where a full-page (MPA) navigation inside an `instant()` scope would intermittently never stream in its deferred content after the scope exits.

**Who needs to know:**

- **Anyone testing in a deployed preview environment with the Instant Navigation DevTools cookie set** — the three PRs together make the deployed testing experience parity with `next dev`. Before canary.75 you could lock the navigation, but reload would either hang or recover incorrectly; now it recovers cleanly via the in-document cookie-clear script.
- **Anyone enabling `experimental.exposeTestingApiInProductionBuild` for a protected preview env** — that's how the bootstrap script gets embedded into the prerendered prelude. A normal prod build never embeds it.
- **Anyone whose `instant-navigation-testing-api.test.ts` was flaky** — most of those flakes are now fixed; if you see remaining ones, file at https://github.com/vercel/next.js/issues with the test name and CI link.

**Sources:**

- [PR #95227 — `Recover from blocking routes under Instant Navigation lock when deployed`](https://github.com/vercel/next.js/pull/95227) · Commit `f14048b691` (2026-07-02T12:51:20Z)
- [PR #95222 — `Make Instant Navigation Testing full-page loads work when deployed`](https://github.com/vercel/next.js/pull/95222) · Commit `3177443336` (2026-07-02T12:51:19Z)
- [PR #95398 — `Clear a resurrected instant cookie on unlock so a hard reload recovers`](https://github.com/vercel/next.js/pull/95398) · Commit `f15aa4e6db` (2026-07-02T12:51:18Z)




## React Compiler 1.0 (React 19.2 + Next.js 16)

The React Compiler 1.0 (October 2025) is a build-time tool that automatically optimizes component rendering — eliminating the need for manual `useMemo` and `useCallback` in most cases. It analyzes your code and generates memoized versions of components and hooks.

**How it works:** The compiler looks at your component code and determines which values are "expensive" and which are stable. It wraps those values in `useMemo`/`useCallback` automatically, so you don't have to.

### Enable in Next.js

Next.js uses an SWC-based implementation that's more efficient than the standalone Babel plugin — it only runs on files that actually use JSX or React hooks.

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  reactCompiler: true,  // ✅ Auto-compile all eligible components
}
```

**With granular control** (opt-in mode — only compile explicitly annotated components):

```ts
const nextConfig: NextConfig = {
  reactCompiler: {
    mode: 'annotation',  // Only compile components/hooks with "use memo" directive
  },
}
```

### Usage in Components

With the compiler enabled, you can write components naturally without manual memoization:

```tsx
// ❌ Before React Compiler — manual memoization required
function ExpensiveList({ items, filter }: Props) {
  const filtered = useMemo(
    () => items.filter(i => i.category === filter),
    [items, filter]
  )
  const sorted = useMemo(
    () => [...filtered].sort((a, b) => a.name.localeCompare(b.name)),
    [filtered]
  )
  return <div>{sorted.map(item => <Item key={item.id} {...item} />)}</div>
}

function Parent() {
  const [count, setCount] = useState(0)
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveList items={data} filter="active" />
    </div>
  )
}
```

```tsx
// ✅ With React Compiler — write naturally, compiler handles memoization
function ExpensiveList({ items, filter }: Props) {
  const filtered = items.filter(i => i.category === filter)
  const sorted = [...filtered].sort((a, b) => a.name.localeCompare(b.name))
  return <div>{sorted.map(item => <Item key={item.id} {...item} />)}</div>
}
```

### ESLint Plugin (Recommended Alongside Compiler)

The `eslint-plugin-react-compiler` catches code patterns that violate compiler rules:

```bash
npm install -D eslint-plugin-react-compiler
```

```js
// eslint.config.mjs
import reactCompiler from 'eslint-plugin-react-compiler'

export default [
  {
    plugins: {
      'react-compiler': reactCompiler,
      // Or via flat config rules:
    },
    rules: {
      'react-compiler/react-compiler': 'warn',  // or 'error' for strict
    },
  },
]
```

**Note:** Even with the React Compiler, keep the ESLint plugin — it catches cases where the compiler can't optimize and tells you why.

#### ESLint Compiler Error Messages Restore Code Frames (React 19.3 canary-e2731312-20260630+, PR [#36901](https://github.com/facebook/react/pull/36901))

When the Rust port of the React Compiler (PR [#36173](https://github.com/facebook/react/pull/36173), which now ships as `experimental.turbopackRustReactCompiler` / `experimental.rustReactCompiler` in Next.js 16.3) became the default compiler backend, ESLint error printing regressed for a few weeks:

- The Rust `CompileError` `LoggerEvent`s carry **plain serialized detail objects** instead of `CompilerError`/`CompilerDiagnostic` class instances.
- The replacement `printErrorMessage()` helper emitted **only** `reason` and `description` — **dropping the source code frame and `file:line:column` location** that used to appear for each error detail.

PR #36901 (Pieter De Baets, merged 2026-06-30T09:20:27Z into React `main`) restores the previous behavior: `printCodeFrame` is exported from `CompilerError` and reused from both ESLint integrations; `printErrorMessage` rebuilds the full message (reason + description + per-detail code frames + hints); the detail-loop is normalized to handle **both** the `details` array (Rust compiler shape) and the legacy flat `loc` shape (deprecated `CompilerErrorDetail`) — without the normalization, iterating over `error.details` on the flat-`loc` path would have thrown `TypeError: not iterable`.

**Practical impact:** if you've been running `eslint-plugin-react-compiler` against the Rust compiler and the output felt "missing" — no source line, no `file:line:column`, just a one-liner reason — upgrade React to a canary with #36901 included (released in `19.3.0-canary-…-20260630+`). The fix lands automatically when Next.js bumps its React canary pin to that canary; if you're on the current canary (`19.3.0-canary-e2731312-20260630` as of this update), the #36901 fix is already included — no further action needed. If your project pins React directly (not via Next.js), pin React to `19.3.0-canary-…-20260630+` to get the fix.

### What the Compiler Handles

| Pattern | Compiler Action |
|---|---|
| `useMemo(() => expr)` | Included automatically |
| `useCallback(fn)` | Included automatically |
| `React.memo(Component)` | Often unnecessary with compiler |
| `[] deps that never change` | Detected and preserved |
| Mutations of props/state | Flagged as errors |

### What the Compiler Requires

The compiler works when your code follows React's rules:
- **No mutation of props or state** — treat all values as immutable
- **Stable hook signatures** — don't call hooks conditionally
- **Functions called during render must be pure** — same inputs → same outputs

```tsx
// ❌ Compiler skips this component — it mutates a prop
function BadComponent({ items }: Props) {
  items.push({ id: 'new' })  // Mutation — compiler skips entire component
  return <List items={items} />
}

// ✅ Compiler optimizes this component — pure render
function GoodComponent({ items }: Props) {
  const newItems = [...items, { id: 'new' }]  // Creates new array
  return <List items={newItems} />
}
```

### When to Use the Compiler

| Scenario | Recommendation |
|---|---|
| New project | ✅ Enable by default |
| Existing codebase with `useMemo`/`useCallback` | ✅ Enable, then gradually remove manual memoization |
| Codebase with frequent prop/state mutations | ⚠️ Fix mutation patterns first |
| Strict performance requirements | ✅ Enable + keep ESLint plugin |

**Sources:**
- [React Compiler 1.0 release blog](https://react.dev/blog/2025/10/07/react-compiler-1)
- [React Compiler docs](https://react.dev/reference/react-compiler)
- [Next.js React Compiler config](https://nextjs.org/docs/app/api-reference/config/next-config-js/reactCompiler)
- [React Compiler configuration](https://react.dev/reference/react-compiler/configuration)



## Activity + `useSyncExternalStore` — Stale State on Reveal Fixed (React main-branch ahead of canary `83840902-20260719`, [PR #36947](https://github.com/facebook/react/pull/36947) by sophiebits, merged 2026-07-21T03:39:10Z, fixes [facebook/react#27670](https://github.com/facebook/react/issues/27670))

**A real bug class** for any app combining React 19.2's `<Activity>` primitive with an external store. The Activity section above documents that `mode='hidden'` tears down Effects (subscriptions, intervals, WebSockets). What wasn't fully documented: **on reveal, the component could be left stale if the store changed during the hidden period.**

### The bug

The reveal path: when `<Activity>` flips from `mode='hidden'` back to `'visible'`, React replays the fiber's effect list *without* an initial render to re-attach passive effects. If `updateStoreInstance` (the internal effect that re-syncs a `useSyncExternalStore` subscription with the current store snapshot) is in the passive effect list, it works. But in practice the effect is often *not* in the passive effect list — `useSyncExternalStore` schedules `updateStoreInstance` only when the snapshot has actually changed. Two manifestations:

**Manifestation 1 — layout effect mutates the store during reveal:**

```tsx
'use client'

function NotificationsPanel() {
  // External store subscription
  const unreadCount = useStore(useNotificationsStore, s => s.unreadCount)

  useLayoutEffect(() => {
    // This layout effect fires DURING the reveal commit,
    // AFTER the subtree rendered but BEFORE the store subscription is re-attached
    if (unreadCount > 0) {
      notificationsStore.markAllRead()
    }
  }, [unreadCount])

  return <div>{unreadCount} unread</div>
}

function App() {
  const [tab, setTab] = useState('inbox')
  return (
    <>
      <button onClick={() => setTab(tab === 'inbox' ? 'archive' : 'inbox')}>Switch</button>
      <Activity mode={tab === 'inbox' ? 'visible' : 'hidden'}>
        <NotificationsPanel />
      </Activity>
    </>
  )
}
```

Sequence of events when switching `archive` → `inbox`:

1. `<Activity mode='visible'>` triggers the reveal.
2. React replays the effect list to re-attach subscriptions.
3. **Before** the store subscription is re-attached, the layout effect fires, calls `markAllRead()` which mutates the store.
4. The store subscription re-attaches but reads the *post-mutation* snapshot (unread=0).
5. The component shows `0 unread` even though there might be unread notifications.

**Manifestation 2 — store changed while hidden:**

```tsx
'use client'

function LiveTicker() {
  // External WebSocket-backed store
  const price = useStore(usePriceStore, s => s.prices.BTC)

  return <div>BTC: {price}</div>
}

function App() {
  const [view, setView] = useState('home')
  return (
    <>
      <button onClick={() => setView(view === 'home' ? 'detail' : 'home')}>Switch</button>
      <Activity mode={view === 'home' ? 'visible' : 'hidden'}>
        <LiveTicker />
      </Activity>
    </>
  )
}
```

Sequence of events when the price feed updates BTC price while `view === 'detail'`:

1. WebSocket message arrives, store updates `BTC` price.
2. User switches `detail` → `home`, `<Activity mode='visible'>` triggers reveal.
3. React replays the effect list. `useSyncExternalStore` reads the current snapshot (new BTC price) but doesn't kick a re-render because the subscription was never notified of the change during hidden mode.
4. The component shows the *old* BTC price until the user navigates or another mutation occurs.

### The fix (in PR #36947)

`updateStoreInstance` is now **pushed unconditionally** into the effect list, but tagged with `HookHasEffect` **only under the same conditions as before**. Two outcomes:

- **Regular commits** (no Activity reveal): the effect runs only when the snapshot actually changed, just like before — no perf cost.
- **Reconnection after reveal**: the effect is forced to run, which reads the current store snapshot and triggers a re-render if the snapshot changed during the hidden period. The component re-renders with fresh data.

**Practical effect:** any app using `<Activity>` + an external store (Zustand, Jotai, custom `useSyncExternalStore`, TanStack Query's `useSyncExternalStore`-based hooks like `useSyncQuery`) now correctly sees store mutations that happened during the hidden period. Previously it would silently show stale state until the user navigated or interacted.

### Migration

**No code change needed** — just upgrade to a React canary that includes PR #36947 (expected in the next React canary after `83840902-20260719`). Once React 19.3 stable ships, the fix is automatic.

**No changes to your `<Activity>` usage, your `useSyncExternalStore` usage, or your store implementation.** The fix is purely internal to React's effect-scheduling for Activity reveals.

### When this matters in practice

- **Tab interfaces using `<Activity>` + a shared store** (e.g., a chat app with `feed` / `messages` / `notifications` tabs backed by a Zustand store that receives WebSocket updates)
- **Sidebar / drawer components using `<Activity>` + an auth-store or feature-flag store** that can change while the sidebar is hidden
- **Modal stacks using `<Activity>` + a notification store** (notifications can arrive while the modal is hidden, and the badge should reflect the new count on reveal)
- **Dashboard widgets using `<Activity>` + a TanStack Query subscription** (queries can invalidate or fetch new data while the widget is hidden; the revealed widget should show fresh data)

### Common mistakes (after the fix)

- **Relying on `<Activity>` to "reset" external store state** — `<Activity>` preserves state, it doesn't reset it. If you want a store reset on hide/show, use `key={open}` to force remount, or call your store's reset action in a `useEffect` watching the `mode` prop.
- **Combining `<Activity>` with `useTransition` for "loading" semantics** — `<Activity>` is for visibility, not for pending. Use `useTransition` or `useFormStatus` / `useActionState`'s `isPending` for loading state.
- **Nesting `<Activity>` boundaries incorrectly** — a child `<Activity>` inside a parent `<Activity>` is fine and intentional (use case: pre-rendering content at lower priority inside a hidden panel), but if the child flips `mode='visible'` while the parent is still `'hidden'`, the child's Effects don't fire (they're transitively hidden). Fix: structure boundaries so the visible/hidden state is consistent across the chain.

**Sources:**
- [PR #36947 — `[Fiber] Detect useSyncExternalStore mutations missed while Activity tree was hidden`](https://github.com/facebook/react/pull/36947)
- [Issue #27670 — Original bug report](https://github.com/facebook/react/issues/27670)
- [React docs — `useSyncExternalStore`](https://react.dev/reference/react/useSyncExternalStore)
- [React docs — `<Activity>`](https://react.dev/reference/react/Activity)

## React 19.2 New Primitives (October 2025)

React 19.2 stabilizes several previously-experimental APIs and introduces new primitives for fine-grained reactivity and loading states.

### `<Activity />` — Hide UI Without Losing State

`<Activity>` is React 19.2's new component primitive. It lets you **hide a part of the UI while preserving its state and DOM** — and optionally cleaning up its Effects. Think of it as `display: none` for a component subtree, with the side benefit that React restores everything when you show it again.

**Props:**

| Prop | Type | Description |
|---|---|---|
| `children` | `ReactNode` | The UI to show/hide. Required. |
| `mode` | `'visible' \| 'hidden'` | `'visible'` (default) renders children normally. `'hidden'` hides them with `display: none`, unmounts their Effects, but keeps their state and DOM around. |

That's it — there is no `detection` prop, no `isActivity` render-prop callback, no separate `visible={true}` boolean. The entire API surface is `children` + `mode`.

**Basic example — preserve sidebar state when collapsed:**

```tsx
'use client'

import { Activity, useState } from 'react'

export function App() {
  const [sidebarOpen, setSidebarOpen] = useState(true)

  return (
    <div className="flex">
      {/* Sidebar stays mounted and keeps its state (scroll position, expanded submenus,
          unsaved form values) when hidden. Only the Effects are torn down. */}
      <Activity mode={sidebarOpen ? 'visible' : 'hidden'}>
        <Sidebar />
      </Activity>
      <main>
        <button onClick={() => setSidebarOpen(o => !o)}>Toggle sidebar</button>
      </main>
    </div>
  )
}
```

**Pre-render content that's likely to become visible:**

```tsx
// Pre-render the heavy Posts tab at low priority before the user clicks it.
// When they switch to it, it's already there.
<Activity mode="hidden">
  <PostsTab />
</Activity>
```

While hidden during initial render, children still render at a lower priority than the visible content, but **without mounting their Effects**. When `mode` flips to `'visible'`, Effects mount normally. This makes tab-switching feel instant without the hidden content doing work in the background.

**Tab interface — preserve per-tab state:**

```tsx
'use client'

import { Activity, useState } from 'react'

const tabs = [
  { id: 'feed', component: Feed },
  { id: 'messages', component: Messages },
  { id: 'notifications', component: Notifications },
] as const

export function SocialSections() {
  const [active, setActive] = useState<typeof tabs[number]['id']>('feed')

  return (
    <>
      <nav>
        {tabs.map(t => (
          <button
            key={t.id}
            onClick={() => setActive(t.id)}
            className={active === t.id ? 'font-bold' : ''}
          >
            {t.id}
          </button>
        ))}
      </nav>

      {tabs.map(t => {
        const Component = t.component
        return (
          <Activity key={t.id} mode={active === t.id ? 'visible' : 'hidden'}>
            <Component />
          </Activity>
        )
      })}
    </>
  )
}
```

**Important behaviors of `mode="hidden"`:**
- Children are hidden via `display: none` (not unmounted)
- Their **Effects are destroyed** — subscriptions, intervals, WebSockets all clean up
- Their **state is preserved** — useState values, refs, scroll position
- Children **still re-render** in response to new props, but at a lower priority than visible content
- When flipped back to `'visible'`, Effects re-mount and state is intact

**Troubleshooting:**

- **"My hidden component has Effects that aren't running"** — That's by design. In `'hidden'` mode, Effects are torn down. Use a visible/invisible check inside the effect if you need it to keep running.
- **"My hidden component has unwanted side effects"** — Same answer. Effects should be tied to the user actually seeing the UI. For analytics/tracking that should always fire, put them outside the Activity boundary.

**`<Activity>` is NOT a loading-state mechanism.** For loading state, use `useFormStatus`, `useActionState`'s `isPending`, or React Query's `isLoading`. `<Activity>` is purely about *visibility vs. hidden*, not about *pending vs. done*.

**Common mistakes:**
- Treating Activity as a spinner (`isPending` replacement) — it's not; use it for tab/modal/sidebar state preservation only
- Wrapping the entire app in a single `<Activity>` — defeats the purpose; the boundary should match a meaningful UI unit (tab, panel, modal, sidebar)
- Expecting children to be completely frozen in `'hidden'` mode — they still re-render on prop changes (at lower priority)

**Sources:**
- [React Activity reference](https://react.dev/reference/react/Activity)
- [React 19.2 release notes](https://react.dev/blog/2025/10/01/react-19-2)

**Practical example — Server Action inside an Activity boundary:**

```tsx
// app/actions.ts
'use server'

import { revalidateTag } from 'next/cache'

export async function publishPost(postId: string) {
  await db.post.update({ where: { id: postId }, data: { published: true } })
  revalidateTag('posts', 'max')
}
```

```tsx
// components/post-actions.tsx
'use client'

import { useActionState } from 'react'
import { publishPost } from '@/app/actions'

const initialState = { error: null as string | null }

// Note: pending state for the button comes from useActionState's isPending,
// not from <Activity>. <Activity> is for keeping the form UI around if the
// user navigates away and back (with cacheComponents).
function PublishButton({ postId }: { postId: string }) {
  const [state, formAction, isPending] = useActionState(
    async (_prev: typeof initialState, _formData: FormData) => {
      try {
        await publishPost(postId)
        return { error: null }
      } catch {
        return { error: 'Failed to publish' }
      }
    },
    initialState
  )

  return (
    <form action={formAction}>
      <button
        type="submit"
        disabled={isPending}
        className={isPending ? 'opacity-50' : ''}
      >
        {isPending ? 'Publishing...' : 'Publish'}
      </button>
      {state.error && <p className="text-sm text-destructive">{state.error}</p>}
    </form>
  )
}
```

**When to use `<Activity>` vs alternatives:**

| Pattern | Best Use |
|---|---|
| `<Activity mode="hidden">` | Preserve state for a tab/sidebar/modal that's currently hidden |
| `useFormStatus` | Form-level pending state — for one submit button |
| `useActionState` (`isPending`) | Server Action pending state — for one mutation |
| `isPending` (React Query) | Query/mutation-specific loading |
| Manual `isLoading` state | When you need precise control of multiple distinct loading states |

**Common `<Activity>` mistakes:**
- Using it as a loading spinner — it's not; use `useFormStatus` or `isPending`
- Wrapping too large a tree — be specific to the UI unit (one tab, one panel, one modal)
- Expecting it to run Effects in hidden mode — Effects are deliberately torn down

### `cacheComponents` Migration — State Preservation & Route Behavior Changes

When you enable `cacheComponents: true` in Next.js 16, several fundamental behaviors change that affect existing Next.js 15 code:

#### State Now Persists Across Navigations (Not Cleared)

With `cacheComponents`, Next.js wraps routes in React's `<Activity mode="hidden">` internally. This means **component state is preserved** when navigating away and restored when navigating back:

```tsx
// ❌ Old Next.js 15 behavior — state was cleared on navigation
// User fills a form, navigates away, comes back — form is empty

// ✅ New cacheComponents behavior — state is PRESERVED
// User fills a form, navigates away, comes back — form still has values
function ContactForm() {
  const [message, setMessage] = useState('')
  // With cacheComponents: this state survives navigation
  return <textarea value={message} onChange={e => setMessage(e.target.value)} />
}
```

**What this means for migrations:**
- If you relied on navigation clearing stale state (e.g., resetting a search input, clearing a draft form), you now need **explicit reset logic**
- Add cleanup in `useEffect` with an empty dependency array, or reset state when params change

```tsx
// Explicitly reset state when route params change (migration pattern)
function SearchPage({ params }: { params: Promise<{ q: string }> }) {
  const [query, setQuery] = useState('')
  const { q } = use(params)

  // Reset when the search query param changes
  useEffect(() => {
    setQuery(q ?? '')
  }, [q])

  return <input value={query} onChange={e => setQuery(e.target.value)} />
}
```

#### Effects Clean Up and Re-Run on Route Restore

When a route is hidden (not currently active), Next.js cleans up its Effects. When the user navigates back, Effects re-run from scratch:

```tsx
// This useEffect runs when route is VISITED, cleans up when route is HIDDEN
useEffect(() => {
  const ws = connectWebSocket()
  return () => ws.close()  // Called when route is hidden
}, [])
```

#### `cacheComponents` + Edge Runtime — Not Supported

If you currently use `runtime = 'edge'` in your route segment config, **it is not compatible with `cacheComponents`**. You must remove it:

```tsx
// ❌ Next.js 15 — Edge runtime
export const runtime = 'edge'

// ✅ Next.js 16 with cacheComponents — Node.js runtime only (default)
export default async function Page() { ... }
```

If you need edge-like behavior for specific routes, use Next.js 16's **Proxy** (`proxy.ts`) instead, which replaces `middleware.ts`.

**Constraint confirmed in 16.3 docs ([#94897](https://github.com/vercel/next.js/pull/94897), canary.61, June 22, 2026):** Cache Components requires the Node.js runtime — even at the segment level. If you have `export const runtime = 'edge'` on any page or layout that participates in `cacheComponents`, the build will fail. Edge runtime segments and Cache Components are mutually exclusive project-wide; this is now called out explicitly in the migration guide.

#### Cache Components Adoption — The `instant = false` Opt-Out + Adoption Skill (16.3+, [#94941](https://github.com/vercel/next.js/pull/94941))

Enabling `cacheComponents: true` on an existing app **fails the build immediately** for any route that reads request-time data outside a `<Suspense>` boundary. On a large app that's a wall of failures with no clear order to fix them in. Two new first-party tools give agents and humans a sequenced adoption path:

**1. `cache-components-instant-false` codemod** (`@next/codemod`, registered at `version: '16.3.0'`) — blanket-inserts `export const instant = false` (with a `// TODO: Cache Components adoption` comment) into every `app/**/{page,layout,default}.{js,jsx,ts,tsx}` that doesn't already declare or export `instant` in any form. Idempotent: skips files with existing `instant` exports (including aliased `export { x as instant }`), skips Client Components (`"use client"`), and skips route handlers. The resulting build is green; the TODO comments are the work queue for milestone B.

```bash
# Run from the project root
npx @next/codemod@canary cache-components-instant-false ./app

# Then enable the flag
# next.config.ts
#   cacheComponents: true,
```

**What the codemod produces:**

```tsx
// app/dashboard/page.tsx — after codemod
// TODO: Cache Components adoption. Refactor this route so this opt-out can be removed.
// See: https://nextjs.org/docs/app/guides/migrating-to-cache-components
export const instant = false

export default function Page() {
  return <h1>Dashboard</h1>
}
```

**Important quirks of `instant = false`:**

- **Highest-wins resolution.** Resolution is top-down, first-explicit-config-wins — the **highest** `instant = false` in a route's tree decides the whole subtree. So removing a leaf's opt-out does nothing while an ancestor still holds one. The codemod opts **every** segment out on purpose: remove them top-down (root layout first, then descend) so the blast radius is one segment at a time.
- **Doesn't clear sync-IO errors.** `new Date()`, `Date.now()`, `Math.random()`, `crypto.randomUUID()` called at render time still fail the prerender (`blocking-prerender-current-time` / `-random` / `-crypto`) even with `instant = false`, because they produce different results on every render. So a shared layout that calls `new Date()` directly will block the build regardless of the opt-out — fix that explicitly.
- **Client Components don't get an opt-out.** `instant` is a Server Component route segment config; exporting it from a `"use client"` module is a build error (`E1344`). They don't need one: a client page is covered by its nearest server layout's opt-out, and a client page can't read server request data (`cookies()`, `headers()`, `await params`) itself.
- **Framework routes (`/_not-found`, etc.) have no user file.** Don't try to add `instant = false` to `app/not-found.tsx` or similar — the directive wouldn't apply. When `/_not-found` blocks, the cause is the **root layout** it renders through; add the opt-out to `app/layout.tsx` instead.

**2. `next-cache-components-adoption` agent skill** ([Next.js repo: `skills/next-cache-components-adoption/SKILL.md`](https://github.com/vercel/next.js/tree/canary/skills/next-cache-components-adoption), June 22, 2026) — an installable agent skill that drives the full adoption in two milestones across five steps:

```bash
# Install with the Vercel skills CLI
npx skills add https://github.com/vercel/next.js/tree/canary/skills/next-cache-components-adoption
```

- **Milestone A — Green build** (steps 1–2): choose Blanket vs Direct strategy, run the codemod (Blanket) or enable the flag directly (Direct), confirm with `next build`.
- **Milestone B — Remove `instant = false`** (steps 2–3): walk the route tree top-down, one subtree at a time, removing each opt-out and either making the route prerenderable or documenting it as a deliberate Block. This is the loop where adoption actually happens — most of the time is spent here. Verifies each change at runtime via [`next-dev-loop`](https://github.com/vercel/next.js/tree/canary/skills/next-dev-loop) (with a manual dev-overlay fallback).
- **Requires Next.js 16.3+ and App Router only** — Pages Router projects should stop and tell the user. Hybrid apps (`pages/` + `app/`) are fine: the flag affects only the `app/` routes.
- **Canary.63 prereq restructure (PR [#95082](https://github.com/vercel/next.js/pull/95082), June 24, 2026):** the `## requires` section is now a labeled checklist — **App Router project**, **Next.js 16.3 or later**, **No incompatible config keys** — with each item naming what's checked, what the failure mode is, and where to resolve it. The "No green baseline before the flag" note was hoisted out of the prereq pile into a separate `### notes` block (alongside **Offline docs**) so a reader scanning prereqs doesn't conflate it with a hard requirement. The codemod block now explains `@canary` necessity in one line so an agent hitting `Invalid transform choice` from `@latest` knows the workaround (driven by three friction logs: react.dev migration where `@latest` 16.2.9 returned `Invalid transform choice`; a marketing-site case where `revalidate` exports were left in and `cacheComponents: true` rejected the build; an e-commerce case where `experimental.dynamicIO` / `useCache` left in `next.config` caused a fatal config error).
- **Canary.63 doc clarification (PR [#95081](https://github.com/vercel/next.js/pull/95081)):** the caching guide and "ISR without Cache Components" pages now reference `generateStaticParams` explicitly (previously they didn't mention it). The guidance also recommends `revalidate = 30 days` (`max`) for CMS-driven content that changes rarely — the default 15-minute revalidate is "too often" for marketing/blog/docs content, and 30 days is enough because managed hosting likely evicts the generated asset after 30 days anyway. Self-host can use longer revalidate too. Both updates surface as `## Usage` notes inside the relevant caching pages.
- **Canary.65 doc clarification (PR [#94997](https://github.com/vercel/next.js/pull/94997), June 24, 2026):** four friction-log fixes that change the *meaning* of common cache-migration terms — re-read this if you were trained on the canary.45 docs:
  - **`adopting-partial-prefetching.mdx`** — reframes the `allow-runtime` row in the audit table. It's an *enhancement* to opt the segment into the runtime-prefetch stage, **not** the fix for the `<Link prefetch={true}>` runtime warning. The per-route opt-in to silence that warning is `prefetch = 'partial'`, not `prefetch = 'allow-runtime'`.
  - **`migrating-to-cache-components.mdx`** — splits `## Enable Cache Components` into two halves with a clean handoff: **before flag** (remove `dynamic` / `revalidate` / `fetchCache` — those are config-level deprecations the flag can't coexist with) and **after flag** (translate `revalidate` → `cacheLife`, `fetchCache` → per-fetch cache directives, `unstable_cache` → `'use cache'`, `fetch` cache options → fetch-level directives). The `revalidateTag` second-argument (`profile`) requirement is now flagged inline with a Before/After example. Two new "Good to know" callouts: off-grid `revalidate` values map to the closest `cacheLife` profile (e.g. `revalidate = 600` → `hours` profile, not a literal-600 entry), and `instant = false` means *allowed-to-block-until-runtime*, not "forced dynamic route".
  - **`blocking-prerender-current-time` / `-random` / `-crypto`** (and the `-client` siblings) — the "Don't want this validation?" opt-out section is removed. `instant = false` is an **instant-navigation knob**; it does *not* suppress sync-IO prerender errors. If you hit one, fix the sync-IO source, don't add `instant = false` hoping it'll bypass the check.
  - **All other `blocking-prerender-*` pages** — standardize the CLS fallback callout so they all link the canonical section (previously each page worded it differently, which made the fall-back rule look optional).
  - **Migration takeaway:** if a skill / agent prompt was written against `cacheComponents` docs from canary.45 or earlier, three of its core recommendations are now subtly wrong: the `allow-runtime` advice for the prefetch warning, the "throw `instant = false` at any blocking error" instinct, and the single-step migration sequence. The two-phase (before-flag / after-flag) sequence is now mandatory.
- **Canary.67 adoption-skill update (PR [#95122](https://github.com/vercel/next.js/pull/95122), June 24, 2026):** the `next-cache-components-adoption` skill is updated to (a) lead with the `next-dev-loop` skill as the recommended runtime-verification path — wording-only change in `## Verifying a fix at runtime`, with a draft user prompt the agent can ask permission to install it — and (b) add a build-only verification path for environments that can't run a dev server (CI, sandboxed agents). Concretely:
  - `## surfacing errors` now opens with **"Prefer `next dev`, but build-only works too."** and reorders so `next dev` comes first.
  - `## Verifying a fix at runtime` now leads with **"`next-dev-loop` is the recommended way to do this"**, gives the agent a draft user prompt to ask permission to install it, and labels the alternatives as "Fallback" / "Build-only verification" / "No browser and no build either" so the preference order is unambiguous.
  - The build-only verification path tells the agent **what it can and cannot honestly call "verified"** without a browser, so agents in CI / sandboxed environments can finish the work without faking runtime verification.
  - The companion docs page `migrating-to-cache-components.mdx` is reorganized: the `next-cache-components-adoption` skill pointer is hoisted into a `## Use the adoption skill (recommended)` H2 with a `npx skills add vercel/next.js --skill next-cache-components-adoption` install command and a `## Or migrate by hand` H2 below. The old `> Good to know:` blockquote was promoted to a top-level section.
  - **Why this matters for agents:** the skill used to read "the fastest path" among three equally valid options. Agents without `next-dev-loop` installed were silently picking option 2 or 3, so the user never saw the recommendation. The new wording forces the agent to explicitly ask whether to install `next-dev-loop` before milestone B. If you're driving CC adoption with a coding agent, the agent's first action under the new skill is a permission prompt — that's expected, not a loop.

**Useful build flags** when iterating:

- `next build --debug-build-paths="app/admin/**/page.tsx"` — builds only the named routes. **Pass file paths relative to project root**, not URL paths — `--debug-build-paths=/admin` matches nothing and silently exits 0.
- `next dev --debug-prerender` — dev-only, prints a fuller stack trace so the error names the originating file and line.

**Optional further optimization** (after adoption is complete): making navigations instant via dev-overlay Insights, adopting Partial Prefetching, locking the result in with e2e tests, growing static shells — all covered by linked docs in the skill's "further reading" section, not re-taught in the skill itself.

#### Dynamic Routes with `cacheComponents` — Wrap in Suspense

Dynamic routes like `blog/[slug]` require special handling with `cacheComponents`. The dynamic data fetching part must be wrapped in `<Suspense>`:

```tsx
// app/blog/[slug]/page.tsx

export default async function BlogPostPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params

  return (
    <div>
      <BlogHeader />  {/* Static — prerendered */}
      <Suspense fallback={<PostSkeleton />}>
        <BlogContent slug={slug} />  {/* Dynamic — streams in */}
      </Suspense>
    </div>
  )
}

async function BlogContent({ slug }: { slug: string }) {
  // This function uses cookies()/headers() or fetches user-specific data
  // It must be wrapped in Suspense to avoid prerender errors
  const post = await getPostBySlug(slug)
  return <article>{post.content}</article>
}
```

**Why:** With `cacheComponents`, Next.js prerenders everything at build time. If a component uses `cookies()`, `headers()`, or other dynamic APIs directly in the page, it will fail prerendering. Wrapping in `<Suspense>` marks it as dynamic.

**Sources:**
- [Next.js migrating to Cache Components guide](https://nextjs.org/docs/app/guides/migrating-to-cache-components)
- [Next.js preserving UI state](https://nextjs.org/docs/app/guides/preserving-ui-state)
- [React Activity hidden mode](https://react.dev/reference/react/Activity#activity)
- [React Activity hidden mode](https://react.dev/reference/react/Activity#activity)

### Next.js 16.3 AI Improvements — `AGENTS.md` Auto-Update + Three First-Party Skills + Actionable Errors + Smaller MCP + Docs as Markdown + `agent-browser` React Commands (June 26, 2026)

On June 26, 2026 — the same day shadcn 4.12.0 Chat Components shipped — Aurora Scharff and Jude Gao published the second installment in the 16.3-preview series: ["Next.js 16.3: AI Improvements"](https://nextjs.org/blog/next-16-3-ai-improvements). Six coordinated changes land in 16.3 that reshape how AI coding agents interact with Next.js. If you're building with Claude Code, Cursor, or any skills.sh-aware agent, every one of these is a behavioral change you'll feel on the next project. Five of the six weren't documented in this skill at all before this update; the sixth (the `next-dev-loop` skill) was referenced only as a sub-bullet.

**1. `AGENTS.md` auto-update** — `next dev` writes and updates the managed pointer block on its own. Projects created with `create-next-app@16.2` already have `AGENTS.md`, but projects upgraded from 16.1 or earlier need the pointer added. In 16.3, the dev server detects when an AI coding agent is in the environment (`CLAUDE_CODE`, `CURSOR_TRACE_ID`, `GITHUB_COPILOT_CHAT`, etc.), checks for the `<!-- BEGIN:nextjs-agent-rules -->` … `<!-- END:nextjs-agent-rules -->` markers in `AGENTS.md`, and inserts (or refreshes) the managed block if it's missing. The exact text it inserts:

```md
<!-- BEGIN:nextjs-agent-rules -->

# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` before writing any code. Heed deprecation notices.

<!-- END:nextjs-agent-rules -->
```

The block is **only written** when (a) an AI coding agent is detected in the environment AND (b) the markers aren't already there. Anything outside the markers is preserved verbatim — your project's own AGENTS.md instructions survive the update. The block shows up as an uncommitted change; commit it as-is. **Opt out** with `agentRules: false` in `next.config.ts`. For 16.1 or earlier (or if you want to opt in without waiting for `next dev`), run the manual codemod: `npx @next/codemod@canary agents-md`. The new managed block is intentionally shorter and more pointed than the 16.2 default — it tells the agent explicitly that *this is not the Next.js it knows* and that the docs in `node_modules` are the source of truth.

**Why this matters for agents:** the previous AGENTS.md behavior (a long-lived file you create once) had two failure modes: (a) the file got out of sync with the Next.js version, so an agent reading it on a 16.3 project might think it was reading 16.1 instructions; (b) agents on a project without `AGENTS.md` had no signal at all that they should read the bundled docs. The auto-update solves both — the marker block is regenerated every time the agent runs `next dev` against a project on a newer Next.js than the file was last written against. The skill previously told agents "if you don't see AGENTS.md, create it" — that's still valid, but the 16.3 default is "let `next dev` create it for you, opt out if you don't want it."

**2. Three first-party Skills** — the `next-cache-components-adoption` skill (already documented, since canary.61) is joined by two new ones. All three live at [github.com/vercel/next.js/tree/canary/skills](https://github.com/vercel/next.js/tree/canary/skills). Install with `npx skills add vercel/next.js --skill <name>`:

- **`next-dev-loop`** (June 2026, general-purpose) — gives the agent the full dev feedback loop: drive the browser, read the console, follow network requests, inspect the React tree, watch compilation issues. Combines two surfaces: **`/_next/mcp`** (the framework's view of routes, server logs, compilation issues) and **`agent-browser`** (the browser's view of DOM, console, network, React tree). Requires **`agent-browser >= 0.27`** (see point 6 below). Prompt your agent: *"After every edit, verify the page still works at runtime using the next-dev-loop skill."*
- **`next-cache-components-adoption`** (June 2026, canary.67+ update) — turns `cacheComponents: true` on in your project and works through the app one feature at a time. Two modes: **Incremental** (lands a mechanical PR up front that opts every route out via `instant = false`, then adopts in follow-up PRs) and **Direct** (adopts every route in one branch). The per-feature loop is the same in both: read the Instant Insight, fetch the per-error docs page it links to, apply the fix, drive the browser through `next-dev-loop` to confirm the static shell renders the right content. Prompt your agent: *"Adopt Cache Components in this project using the next-cache-components-adoption skill."*
- **`next-cache-components-optimizer`** (June 2026, **new in 16.3** — not documented in the skill before this update) — optimizes a Cache Components route for instant navigation by running an observe-fix-iterate loop against the static shell. Use it whenever you want a route to feel faster, by growing the static shell so more of the page is ready at request time. Two modes: **Page-render mode** (push I/O deeper into the component tree or cache it with `'use cache'` so more of a single page can render statically) and **Nav mode** (ensure navigations between pages are instant; server-side work needed for the next page streams in without blocking the navigation). The skill screenshots before and after each change through `next-dev-loop`; **identical screenshots roll the change back.** Prompt your agent: *"Increase the static shell on /dashboard using the next-cache-components-optimizer skill."*

**Retirement of older knowledge Skills** — the earlier Vercel knowledge Skills (App Router conventions, caching APIs, etc., previously distributed via `skills.sh`) are **retired** in 16.3 because the bundled docs (now reachable through the managed `AGENTS.md` block) make them redundant. **Run `npx skills update` to remove the old knowledge Skills from your installed set.** The three first-party Skills above are the new recommended set; the knowledge Skills are explicitly deprecated. If you see an agent prompt that still references "Vercel App Router skill" or "Vercel Caching skill", that prompt is from the pre-16.3 era and should be rewritten.

**3. Actionable errors — Instant Insights with labeled fixes + `Copy prompt`** — when `cacheComponents: true` and a server-side `await` outside `<Suspense>` is detected, Instant Insights presents **three labeled fixes** as a product decision with trade-offs. (The button label was renamed `Copy as prompt` → **`Copy prompt`** in 16.3.0-canary.73 by [PR #95309](https://github.com/vercel/next.js/pull/95309); the underlying data model was also renamed `FixOption` → `FixCard` and the grid container `FixOptionsList` → `FixCardGrid`. If you see older tutorials referencing "Copy as prompt" or "FixOption", they predate canary.73.) **Stream** (wrap in `<Suspense>`), **Cache** (`'use cache'`), or **Block** (`export const instant = false`). Each label is a button that links to the matching docs section. The **Copy prompt** button packages the chosen fix into a paste-ready prompt for your coding agent — a 7-step checklist that walks the agent through (1) confirming browser tooling is set up, (2) identifying the failing code, (3) reading the rule docs at `https://nextjs.org/docs/messages/<error-key>` and the per-fix Patterns + Gotchas, (4) applying the chosen pattern, (5) verifying at runtime via `next-dev-loop`, (6) checking the shell isn't empty (a Suspense boundary placed too high leaves a build-passing shell with nothing in it), (7) re-checking sibling routes if shared code was touched. The same fix menu shows up in the terminal during `next dev`, and `next build` emits it when an error stops a prerender — so an agent reading errors from CI logs gets the same labeled fixes and links as an agent reading the dev overlay. **Why this matters:** the previous Instant Insights were developer-facing ("here's a fix card, click the doc link"); 16.3 makes them agent-facing ("here's a fix card, click `Copy prompt`, paste into your agent"). The 7-step checklist tells the agent *what it can and cannot honestly call verified* — explicit "the Insight clearing in the dev overlay confirms the build is happy, but not what actually renders" wording.

**4. MCP server — smaller and more focused** — the Next.js DevTools MCP server **removes** its embedded Next.js knowledge base, the upgrade helper, and the Cache Components helpers (these are now reachable through the bundled docs and the three first-party Skills above — keeping them in the MCP server was duplicate work). It adds **two new compilation tools**: **`get_compilation_issues`** (whole-project — returns all current compilation errors) and **`compile_route`** (single-route — returns the compilation result for one route, without doing a full `next build`). **Why this matters:** agents were running `next build` just to check whether code compiles, which is overkill while still editing. The two new tools answer the same question from the running dev server, much faster. Skills like `next-dev-loop` call the underlying `/_next/mcp` endpoints directly, so they work without extra setup. To use these tools from your own agent client, add `next-devtools-mcp` to `.mcp.json` (the MCP server discovers the running `next dev` automatically). The skill's `deployment.md` MCP section currently describes the 16.2-era MCP server with the knowledge-base tools — that section is updated alongside this one.

**4a. MCP `get_request_insights` tool (added canary.85, July 13, 2026, PR [#93977](https://github.com/vercel/next.js/pull/93977))** — the MCP server also grew a third tool, **`get_request_insights`**, that returns the last 100 requests captured by the dev-only `experimental.requestInsights` feature (gated on `experimental.requestInsights: true` in `next.config.ts`). Input schema: `{ requestId?: string, htmlRequestId?: string }` (both optional filters). Returns the sanitized `RequestInsight[]` with `spans[]` + `fetches[]` per request. **Why this matters:** instead of an agent parsing a `curl http://localhost:3000/_next/development/request-insights` output and traversing it, it can call `mcp__next-devtools-mcp__get_request_insights` directly. The full Request Insights stack is documented in setup.md → `experimental.requestInsights`. For shell-only agents / CI scripts that don't have an MCP client, the companion **`next experimental-request-insights` CLI** (also added in canary.85) prints a human summary directly to stdout — `next experimental-request-insights --json` for raw output. The `subscribe_to_request_insights` MCP tool from the original PR description was **deferred** (snapshot + re-invoke-on-interval is the shipped streaming approximation).

**5. Docs as Markdown** — append `.md` to any Next.js docs URL to get the page as plain markdown: `https://nextjs.org/docs/app/api-reference/directives/use-cache.md`. Works for any page on `nextjs.org/docs`, including the per-error pages (`https://nextjs.org/docs/messages/blocking-prerender-dynamic.md`). Clients that send an `Accept: text/markdown` header get the markdown version automatically. **Full index** at `/docs/llms.txt`; **`/docs/llms-full.txt`** bundles all doc pages into a single file. Both follow the [llms.txt convention](https://llmstxt.org), so any agent that already reads `llms.txt` for other tools can read Next.js docs the same way. **Why this matters:** agents that don't have a filesystem (curl-only agents, agents in restricted sandboxes) can now `curl` a docs page and get parseable markdown instead of HTML soup.

**6. `agent-browser` 0.27+ with React DevTools introspection** — the experimental `next-browser` CLI from Next.js 16.2 has **merged into the general-purpose [`agent-browser`](https://www.npmjs.com/package/agent-browser) CLI**. Everything `next-browser` did is now in `agent-browser`, and it works beyond Next.js too. **`agent-browser` 0.27+** adds React DevTools introspection on top of the existing DOM, console, network, and Web Vitals access. New commands:

| Command | Purpose |
|---|---|
| `agent-browser react tree` | List the React component tree with fiber IDs |
| `agent-browser react inspect <fiberId>` | Inspect a single component (props, hooks, state, source location) |
| `agent-browser react renders start` / `stop` | Profile re-renders over a time window |
| `agent-browser react suspense --only-dynamic --json` | See what's holding a render — machine-readable JSON for agents |

Install or upgrade with `npm install -g agent-browser@^0.27`. The React commands require `--enable react-devtools` at launch (the `next-dev-loop` skill launches `agent-browser` with React DevTools enabled by default). **`agent-browser` 0.31.1** is the current version (verified on npm); 0.27 is the minimum for React introspection, but `next-dev-loop` now requires 0.31.1+ (canary.69, PR [#95209](https://github.com/vercel/next.js/pull/95209), June 26, 2026). Tool profiles for the MCP server (`agent-browser mcp --tools all`, `--tools core,network,react`, etc.) are documented in the README; default profile is `core`, which keeps MCP context small for everyday browser automation.

**Practical implications for agents driving Next.js projects in 16.3:**

- **On a 16.1 or earlier project:** `next dev` won't auto-write AGENTS.md (the block is opt-out, but the dev server only writes it for 16.3+). Run `npx @next/codemod@canary agents-md` once, or write the managed block yourself. If you've installed the old Vercel knowledge Skills, run `npx skills update` to remove them — they're deprecated.
- **On a 16.2 project:** `next dev` may write the managed block on next run (if the markers aren't there). The existing AGENTS.md content (from `create-next-app@16.2`) is preserved outside the markers, so no information is lost. Update your dev tooling to use `agent-browser` instead of the now-deprecated `next-browser` (see point 6).
- **On a 16.3 project (canary/preview):** `next dev` writes the managed block automatically. Three new MCP tools are available: `get_compilation_issues` + `compile_route` (canary.73+) replace `next build` for "did my edit compile?" checks; `get_request_insights` (canary.85+) returns the last 100 requests from the dev-only `experimental.requestInsights` snapshot for diagnosing slow renders / fetch failures / cache behavior. The `next-cache-components-adoption` and `next-cache-components-optimizer` Skills replace the old "migrate to CC by hand" workflow. `Copy prompt` in the dev overlay replaces "open the doc link and read it yourself." The new `next experimental-request-insights` CLI (canary.85+) is the shell-only equivalent of the `get_request_insights` MCP tool — use it in CI scripts and shell-only agents.

**Telemetry note (canary.87+, [PR #95586](https://github.com/vercel/next.js/pull/95586) `telemetry: add agentName to anonymous metadata` by andrewimm, merged 2026-07-14T23:56:02Z — canary-branch commit post-canary.86):** Next.js's anonymous telemetry payload now includes a new `agentName` field (string | null) alongside the existing `ciName` / `nextVersion` / `isCI` fields. The field is populated by a vendored copy of `@vercel/detect-agent` (from the Vercel CLI) wrapped in a memoized `getAgentName()` helper (`packages/next/src/telemetry/agent-name.ts`). Detection reads env-var signals (`CLAUDE_CODE`, `CURSOR_TRACE_ID`, `GITHUB_COPILOT_CHAT`, etc.) and probes the filesystem for Devin. Detection failures resolve to `null` so telemetry never throws; concurrent callers share the in-flight detection promise. **`NEXT_TELEMETRY_DEBUG=1` now prints the meta block alongside each event** (the meta block was not output before, so this is a debugging win for anyone running a local agent loop). The data is sent only when telemetry is opted in. Nothing in user code changes. **Why this matters for skill users:** it gives the Next.js team anonymous visibility into how much of Next.js's development happens under AI coding agents (claude / cursor / codex / devin / etc.) — the kind of signal that shapes future agent-tooling investment.

**Sources:**
- [Next.js 16.3: AI Improvements (official blog)](https://nextjs.org/blog/next-16-3-ai-improvements)
- [Next.js 16.3: Instant Navigations (companion post, June 17, 2026)](https://nextjs.org/blog/next-16-3-instant-navigations)
- [Next.js — first-party Skills directory (`vercel/next.js/tree/canary/skills`)](https://github.com/vercel/next.js/tree/canary/skills)
- [`next-dev-loop` skill source](https://github.com/vercel/next.js/tree/canary/skills/next-dev-loop)
- [`next-cache-components-optimizer` skill source (NEW in 16.3)](https://github.com/vercel/next.js/tree/canary/skills/next-cache-components-optimizer)
- [Next.js docs — Actionable errors / `Copy prompt` flow](https://nextjs.org/docs/app/guides/instant-insights)
- [Next.js docs — Per-error pages (`/docs/messages/`)](https://nextjs.org/docs/messages/blocking-prerender-dynamic)
- [`agent-browser` README — React DevTools profile](https://github.com/vercel-labs/agent-browser)
- [`agent-browser` on npm (current: 0.31.1)](https://www.npmjs.com/package/agent-browser)
- [Next.js docs — Docs as Markdown (`/docs/llms.txt`)](https://nextjs.org/docs/llms.txt)
- [Next.js docs — Full Markdown bundle (`/docs/llms-full.txt`)](https://nextjs.org/docs/llms-full.txt)
- [llms.txt convention](https://llmstxt.org)
- [PR #95209 — `next-dev-loop` requires `agent-browser >= 0.31.1` (canary.69)](https://github.com/vercel/next.js/pull/95209)

### `cacheLife` Profile Recommendations — Explicit Profile Name Recommended (16.3.0-canary.73, [PR #95311](https://github.com/vercel/next.js/pull/95311) by icyJoseph, merged 2026-07-01T13:09:25Z — docs only)

The 16.3 caching guide now **explicitly recommends passing the profile name** (`cacheLife('hours')`, `cacheLife('days')`) rather than relying on the implicit `default-profile` (which is `15 minutes` revalidate / `1 hour` expire). The implicit default works for short-lived queries (data that's stale after 15 minutes is acceptable), but it's a footgun for marketing-site or blog content where 15 minutes is too aggressive — your origin gets hammered on every revalidation even though the content barely changes. The recommended pattern is:

```tsx
// ✅ Recommended — explicit profile name
'use cache'
async function getBlogPost(slug: string) {
  const post = await db.posts.findUnique({ where: { slug } })
  return post
}
// getBlogPost revalidates every 'hours' (= 1 hour revalidate, 1 day expire, 1 week stale)

// ❌ Avoid — implicit default-profile
'use cache'
async function getBlogPost(slug: string) {
  // no cacheLife() call → uses 'default-profile' (15 min revalidate)
  // fine for live dashboards, wrong for rarely-changing CMS content
}
```

**Overriding built-in profiles.** The five built-in profiles (`default-profile`, `hours`, `days`, `weeks`, `max`) are looked up by name; you can override any of them by exporting `cacheLife` from a file with the same name (e.g. `app/blog/cacheLife.ts` exporting `cacheLife('days', { revalidate: 60 * 60 * 24 * 7, expire: 60 * 60 * 24 * 30, stale: 60 * 60 * 24 * 30 })`). The override is local to the route tree under that directory; the `days` profile is unchanged everywhere else. The rule was previously only documented in an issue comment — PR #95311 promotes it to the public docs.

**Source:** [PR #95311 — `docs: recommend explicit cacheLife and clarify overriding built-in cache profiles`](https://github.com/vercel/next.js/pull/95311) · [Commit `b3118fca`](https://github.com/vercel/next.js/commit/b3118fca)

### `cacheLife` Profile Typing — `ResolvedCacheLifeProfiles` Threads `Required<CacheLife>.default` Through the Pipeline (16.3.0-canary.77+, [PR #95428](https://github.com/vercel/next.js/pull/95428) by unstubbable)

Until canary.77, the resolved `cacheLife` profile set was typed as `Partial<CacheLife>` with a required-but-optional `default` — meaning the type system would not catch a missing `default` profile and the runtime had to fall back to `assertDefaultCacheLife()` plus optional-chaining + per-`cacheLife()`-presence `InvariantError` checks. PR #95428 replaces that whole dance with a new `ResolvedCacheLifeProfiles` type whose `default` is **non-optional** (`Required<CacheLife>['default']`).

**What changed in the type pipeline:**

- **Config-complete (`packages/next/src/server/config/complete.ts`):** resolves user-provided `cacheLife` profiles together with the five built-ins (`default-profile`, `hours`, `days`, `weeks`, `max`) into a single non-optional map keyed by profile name.
- **Render options (`packages/next/src/server/route-modules/render.ts`):** the resolved map is passed straight through as `cacheLifeProfiles`.
- **Work store (`packages/next/src/server/app-render/work-store.ts`):** carries the same map so a `'use cache'` function can look up its profile by name without re-resolving.
- **Build / export / dev workers (`packages/next/src/build/templates/app-page.ts`):** all three templates instantiate the typed map instead of `Partial<CacheLife>`.
- **Proxy / middleware work store:** a separate sentinel whose `default` getter throws if ever read — proxy/middleware never uses `cacheLife`, so the type system enforces that the unused field cannot be accessed.

**What disappeared:**

- The runtime `assertDefaultCacheLife()` function — no longer needed; the type guarantees `default` is present.
- `InvariantError` throws in the `cacheLife()` lookup helper for "profile name not in map" — replaced by a type-level guarantee plus a simple runtime check (the profile was never present at runtime either, but now the type prevents the mistake from compiling).
- Every optional-chain (`profiles?.default`) at the call sites — the type is non-optional.

**Why this matters for users:**

- If you only call `cacheLife('hours')` or other named profiles, **nothing changes for you**. The runtime behavior is identical.
- If you were somehow relying on `cacheLifeProfiles?.default` being possibly-undefined (e.g. in a custom cache handler that read it directly from the work store), you'll need to drop the `?.` — it's now non-optional.
- The proxy/middleware work-store sentinel means **proxy/middleware code paths cannot accidentally read or write `cacheLife` profiles** — the getter throws if accessed. This is a stronger guarantee than what previously existed (where the field was just absent).
- Less Next.js internal code overall — the runtime check is faster too (no `assertDefaultCacheLife` import + invariant creation).

**Source:** [PR #95428 — `Type resolved cacheLife profiles, dropping runtime asserts`](https://github.com/vercel/next.js/pull/95428)

### `useEffectEvent` — Stable Event Handlers in Effects (React 19.2)

`useEffectEvent` was previously experimental (`useEffectEvent` from `react experimental`) and is now **stable** in React 19.2. It solves the "stale closure" problem in `useEffect` — you can define event handlers inside `useEffect` that always see the latest values without needing them in the dependency array:

```tsx
'use client'

import { useEffect, useEffectEvent } from 'react'

function ChatRoom({ roomId }: { roomId: string }) {
  const [status, setStatus] = useState<'connected' | 'disconnected'>('disconnected')

  // ❌ Problem: messageHandler captures stale roomId if not in deps
  // useEffect(() => {
  //   const ws = new WebSocket(`wss://chat/${roomId}`)
  //   ws.onmessage = (event) => {
  //     // roomId is stale here if deps aren't managed carefully
  //     handleMessage(roomId, event.data)
  //   }
  // }, [roomId])

  // ✅ useEffectEvent — event handler that always sees current roomId
  const handleMessage = useEffectEvent((message: string) => {
    // This function can access current props/state without them being deps
    console.log(`[Room ${roomId}]:`, message)
  })

  useEffect(() => {
    const ws = new WebSocket(`wss://chat/${roomId}`)
    setStatus('connected')

    ws.onmessage = (event) => {
      handleMessage(event.data) // Always sees current roomId
    }

    return () => {
      ws.close()
      setStatus('disconnected')
    }
  }, [roomId])

  return <div className={status === 'connected' ? 'text-green-500' : 'text-gray-400'}>{status}</div>
}
```

**Why `useEffectEvent` instead of just putting the function inside the effect?**
- Putting logic inside `useEffect` makes it harder to test and reason about
- `useEffectEvent` lets you keep the effect focused on subscription lifecycle while extracting event handlers that need current values

**Rules:**
- `useEffectEvent` can only be called inside `useEffect`
- The event handler it returns can reference any current value without causing the effect to re-run
- Unlike `useCallback`, there's no dependency array — it captures values at call time, not render time

### `cacheSignal` — AbortSignal for Cached Renders (RSC)

`cacheSignal` returns an `AbortSignal` that aborts when React finishes a render — successfully, aborted, or failed. It's the proper way to tie async work to React's render lifetime in Server Components, so cancelled renders don't waste bandwidth or hold the process open.

It is **not** a reactive memoization primitive. There is no `.read()` or `.get()` method.

**Signature:**

```ts
const signal: AbortSignal | null = cacheSignal()
```

- Returns an `AbortSignal` when called **during rendering** in a Server Component
- Returns `null` outside of rendering, or in Client Components (for now)

**Use case — cancel in-flight fetches when render is superseded:**

```tsx
// app/user/[id]/page.tsx — Server Component
import { cache, cacheSignal } from 'react'

// Wrap fetch in cache() to dedupe across components in the same render
const fetchUser = cache(async (id: string, signal: AbortSignal) => {
  const res = await fetch(`https://api.example.com/users/${id}`, { signal })
  return res.json()
})

export default async function UserPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const signal = cacheSignal()
  if (!signal) throw new Error('cacheSignal must be called during render')

  const user = await fetchUser(id, signal)
  return <div>{user.name}</div>
}
```

If the user navigates away mid-fetch, React aborts the render → the `AbortSignal` fires → the `fetch` is cancelled and the connection is released.

**Pitfall — the request must be started inside the render that owns the signal:**

```tsx
// 🚩 Pitfall: starting the fetch outside the render means cacheSignal() can't cancel it
const response = fetch(url, { signal: cacheSignal() })

export default async function Page() {
  await response  // Will not be aborted when render ends
  return <div>...</div>
}
```

**Sources:**
- [React cacheSignal reference](https://react.dev/reference/react/cacheSignal)

### `cache()` — Function Memoization (React 19.2)

`cache()` memoizes a function's return value based on its arguments — similar to `useMemo` but for standalone functions:

```tsx
import { cache } from 'react'

// Memoized function — same args return cached result
const getUserById = cache(async (id: string) => {
  console.log('API call for:', id) // Only runs once per unique id
  const res = await fetch(`/api/users/${id}`)
  return res.json()
})

// In server components:
async function UserCard({ id }: { id: string }) {
  const user = await getUserById(id) // First call: fetches
  // Next UserCard with same id: returns cached result
  return <div>{user.name}</div>
}

// In client components:
function UserCard({ id }: { id: string }) {
  const userPromise = getUserById(id)
  const user = use(userPromise) // Suspends until resolved
  return <div>{user.name}</div>
}
```

**When to use `cache()`:**
- Shared data fetching functions used across multiple components
- Server-side data access in RSC patterns where you want request-level memoization
- Replacing ad-hoc memoization patterns with a cleaner API

**Note:** `cache()` in React 19.2 is distinct from `use cache` in Next.js 16. React's `cache()` is a general-purpose memoization primitive; Next.js `use cache` is a framework-level caching directive with server-side persistence.

### Combined Example: Activity + Cache + EffectEvent

```tsx
'use client'

import { Activity, useEffectEvent, cache, useEffect, useState } from 'react'

// cache() is a React primitive that memoizes a function per request.
// Same args in the same render = same return value (server or client).
const fetchNotifications = cache(async (userId: string) => {
  const res = await fetch(`/api/notifications?userId=${userId}`)
  return res.json()
})

export function NotificationBell({ userId }: { userId: string }) {
  const [unread, setUnread] = useState(0)
  const [panelOpen, setPanelOpen] = useState(false)

  // useEffectEvent — event handler that always sees current userId without
  // forcing the polling effect to reconnect
  const markAsRead = useEffectEvent(async () => {
    await fetch(`/api/notifications/read`, { method: 'POST' })
    setUnread(0)
  })

  useEffect(() => {
    const interval = setInterval(async () => {
      const data = await fetchNotifications(userId)
      setUnread(data.unread)
    }, 30000)
    return () => clearInterval(interval)
  }, [userId])

  return (
    <>
      {/* Activity preserves the panel's state (scroll, filters) when it's closed */}
      <Activity mode={panelOpen ? 'visible' : 'hidden'}>
        <NotificationPanel onMarkAsRead={markAsRead} />
      </Activity>
      <button onClick={() => setPanelOpen(o => !o)}>
        🔔 {unread > 0 ? `(${unread})` : ''}
      </button>
    </>
  )
}
```

**Sources:**
- [React 19.2 release notes](https://react.dev/blog/2025/10/01/react-19-2)
- [React Activity component](https://react.dev/reference/react/Activity)
- [React useEffectEvent](https://react.dev/reference/react/useEffectEvent)
- [React cache API](https://react.dev/reference/react/cache)
## Practical Activity Patterns (React 19.2)

Beyond the basic patterns above, here are production-ready Activity combinations. Remember: `<Activity>` is a visibility primitive, **not** a loading-state detector. The `isPending` / `isLoading` flags come from `useActionState`, `useFormStatus`, or React Query.

### Activity + useOptimistic — Follow/Following Button

`useOptimistic` gives instant UI feedback; `useActionState` provides the real pending flag. `<Activity>` is not needed here — it would just preserve button state on a hidden tab. The pending UI is driven entirely by `isPending`:

```tsx
'use client'

import { useActionState, useOptimistic } from 'react'
import { followUser, unfollowUser } from '@/app/actions'

interface FollowButtonProps {
  userId: string
  isFollowing: boolean
  followerCount: number
}

function FollowButton({ userId, isFollowing, followerCount }: FollowButtonProps) {
  // useActionState — pending flag comes from here
  const [, formAction, isPending] = useActionState(
    async (_prev: { following: boolean }, formData: FormData) => {
      const action = formData.get('action') as 'follow' | 'unfollow'
      if (action === 'follow') await followUser(userId)
      else await unfollowUser(userId)
      return { following: action === 'follow' }
    },
    { following: isFollowing }
  )

  // useOptimistic — instant visual feedback before server confirms
  const [optimistic, addOptimistic] = useOptimistic(
    { following: isFollowing, count: followerCount },
    (current, newFollowing: boolean) => ({
      following: newFollowing,
      count: current.count + (newFollowing ? 1 : -1),
    })
  )

  function handleClick() {
    const newFollowing = !optimistic.following
    addOptimistic(newFollowing)
    const fd = new FormData()
    fd.set('action', newFollowing ? 'follow' : 'unfollow')
    formAction(fd)
  }

  return (
    <button
      onClick={handleClick}
      disabled={isPending}
      className={optimistic.following ? 'bg-primary text-white' : 'border border-primary'}
    >
      {isPending
        ? '...'
        : optimistic.following
          ? `Following (${optimistic.count})`
          : `Follow (${followerCount})`}
    </button>
  )
}
```

### Activity for Hidden Tabs — Preserve State + Save CPU

Where `<Activity>` **does** shine: keeping a heavy tab's state and DOM around without paying the cost of its Effects running. Use it to defer work that the user might trigger later:

```tsx
'use client'

import { Activity, useState, useEffect } from 'react'

// A "heavy" tab that subscribes to a websocket, polls a feed, and holds scroll state
function ActivityFeed({ userId }: { userId: string }) {
  const [posts, setPosts] = useState<Post[]>([])
  const [scrollPos, setScrollPos] = useState(0)

  useEffect(() => {
    const ws = new WebSocket(`/feed?userId=${userId}`)
    ws.onmessage = (e) => setPosts(prev => [JSON.parse(e.data), ...prev])
    return () => ws.close()
  }, [userId])

  return <FeedList posts={posts} initialScroll={scrollPos} onScroll={setScrollPos} />
}

function ProfileTab({ userId }: { userId: string }) {
  return <ProfileDetails userId={userId} />
}

export function Dashboard({ userId }: { userId: string }) {
  const [tab, setTab] = useState<'feed' | 'profile'>('feed')

  return (
    <div>
      <nav>
        <button onClick={() => setTab('feed')}>Feed</button>
        <button onClick={() => setTab('profile')}>Profile</button>
      </nav>

      {/* When hidden: scroll position, form drafts, expanded threads all survive.
          The websocket is closed (Effect cleaned up), so no background traffic. */}
      <Activity mode={tab === 'feed' ? 'visible' : 'hidden'}>
        <ActivityFeed userId={userId} />
      </Activity>
      <Activity mode={tab === 'profile' ? 'visible' : 'hidden'}>
        <ProfileTab userId={userId} />
      </Activity>
    </div>
  )
}
```

**Why this matters for INP (Interaction to Next Paint):**
- A naive implementation that always renders both tabs keeps both websockets open, both intervals running, both event listeners attached
- `<Activity mode="hidden">` tears down Effects for hidden tabs → no wasted CPU, no duplicate subscriptions
- When the user switches back, state is restored instantly without a network round-trip

**Rule of thumb:** Use `<Activity mode="hidden">` for any tab/panel/modal that's expensive to keep alive and likely to be revisited.

**Sources:**
- [React Activity reference](https://react.dev/reference/react/Activity)
- [React 19.2 release notes](https://react.dev/blog/2025/10/01/react-19-2)

### Activity + Server Action + Error Boundary — Complete Pattern

A complete, production-ready pattern for a publish button. The pending state comes from `useActionState`; `<Activity>` is used (optionally) to keep the form mounted in a sidebar/drawer that's not always visible:

```tsx
// app/actions.ts
'use server'

import { revalidateTag } from 'next/cache'

export async function publishPost(postId: string): Promise<{ error: string | null }> {
  try {
    await db.post.update({ where: { id: postId }, data: { published: true } })
    revalidateTag('posts', 'max')
    return { error: null }
  } catch {
    return { error: 'Failed to publish post. Please try again.' }
  }
}
```

```tsx
// components/post-actions.tsx
'use client'

import { Activity, useActionState } from 'react'
import { publishPost } from '@/app/actions'
import { AlertCircle, CheckCircle } from 'lucide-react'

const initialState = { error: null as string | null }

function PublishButton({ postId, inDrawer, onClose }: { postId: string; inDrawer: boolean; onClose: () => void }) {
  const [state, formAction, isPending] = useActionState(
    async (_prev: typeof initialState, _formData: FormData) => {
      return await publishPost(postId)
    },
    initialState
  )

  return (
    // Activity preserves the form state (and any validation errors) when the
    // drawer is closed. Effects for the form are torn down while hidden.
    <Activity mode={inDrawer ? 'visible' : 'hidden'}>
      <div className="flex flex-col gap-2">
        <button
          type="submit"
          formAction={formAction}
          disabled={isPending}
          className="inline-flex items-center gap-2 px-4 py-2 bg-primary text-white rounded-md disabled:opacity-50"
        >
          {isPending ? (
            <>Publishing...</>
          ) : (
            <>
              <CheckCircle className="h-4 w-4" />
              Publish
            </>
          )}
        </button>
        {state.error && (
          <div className="flex items-center gap-2 text-sm text-destructive">
            <AlertCircle className="h-4 w-4" />
            {state.error}
          </div>
        )}
      </div>
    </Activity>
  )
}
```

**Key points:**
- Server Action returns `{ error: string | null }` — allows the component to display errors inline without throwing
- `isPending` comes from `useActionState`, not from `<Activity>`
- `<Activity mode="hidden">` is used to preserve the form's state when the parent drawer/tab is closed — it's a UX nicety, not a loading mechanism
- Error display is inline (not an Error Boundary replacement), so the rest of the UI remains interactive
- For critical errors requiring full-UI replacement, wrap in an Error Boundary


## `captureOwnerStack` — Debug Component Ownership (React 19.1)

React 19.1 introduced `captureOwnerStack` — a development-only API that captures the "owner chain" for a component. An owner stack shows **which components are responsible for rendering a particular component**, making it easier to trace why something rendered.

This is distinct from a "component stack" which shows the tree hierarchy. Owner stacks show the call chain through `createElement` — useful when debugging why unexpected renders occur.

```tsx
import { captureOwnerStack } from 'react'

function MyComponent() {
  const ownerStack = captureOwnerStack()
  console.log('Owner stack:', ownerStack)
  // Output: "  at Card\n  at Dashboard\n  at App"

  return <div>Content</div>
}
```

**When to use `captureOwnerStack`:**
- Debugging unexpected renders — trace back through owners
- Understanding component responsibility in complex trees
- Logging in error boundaries to show what caused an error

**Note:** `captureOwnerStack` is **development-only**. It returns `null` in production builds. Don't use it in production code — it's purely for debugging during development.

**Common patterns:**

```tsx
// In an error boundary — log the owner stack alongside the error
class ErrorBoundary extends Component {
  componentDidCatch(error, info) {
    const ownerStack = captureOwnerStack()
    console.error('Error caused by:', ownerStack)
    // Shows which component chain caused the render that threw
  }
}
```

```tsx
// In development — log owner stacks for expensive renders
function ExpensiveList({ items }: { items: Item[] }) {
  if (process.env.NODE_ENV === 'development') {
    const owner = captureOwnerStack()
    console.log('[ExpensiveList] rendered by:', owner)
  }
  // ... render logic
}
```

**Owner Stack vs Component Stack:**

| Concern | Owner Stack | Component Stack |
|---|---|---|
| Shows | Which components **caused** this render (via `createElement` calls) | Which components **contain** this component (tree hierarchy) |
| Use when | Debugging **why** something rendered | Debugging **where** in the tree an error occurred |
| Format | Call chain of responsible components | Breadcrumb of parent components |

**Sources:**
- [React 19.1 release notes](https://react.dev/blog/2025/03/28/react-19)
- [React captureOwnerStack reference](https://react.dev/reference/react/captureOwnerStack)

## React `<ViewTransition>` Component (React 19.2)

React 19.2 provides a **`<ViewTransition>` component** — the idiomatic React way to use the View Transitions API. This is preferred over `document.startViewTransition()` because it hooks into React's render cycle automatically, handles SSR safely, and provides a declarative API with `enter`/`exit`/`default` animation class props.

### Basic Usage

Wrap any elements you want to animate with `<ViewTransition>` and give them the same `name` prop. When the wrapped content changes, React automatically starts a view transition:

```tsx
'use client'
import { ViewTransition } from 'react'

function ProductGallery({ images }: { images: string[] }) {
  const [selected, setSelected] = useState(0)

  return (
    <div>
      {/* Same name on both — React auto-transitions when selected changes */}
      <ViewTransition name={`gallery-${selected}`}>
        <img
          key={selected}
          src={images[selected]}
          alt="Product gallery"
          className="w-full h-64 object-cover rounded-lg"
        />
      </ViewTransition>

      <div className="thumbnails">
        {images.map((src, i) => (
          <button key={i} onClick={() => setSelected(i)}>
            <img src={src} alt={`Thumbnail ${i}`} />
          </button>
        ))}
      </div>
    </div>
  )
}
```

### Named View Transitions — Shared Element Animation

For the classic "card to detail page" morph, use the same dynamic `name` on matching elements in different routes. The browser matches them automatically:

```tsx
// app/products/page.tsx — grid
'use client'
import { ViewTransition } from 'react'

export function ProductCard({ product }: { product: Product }) {
  return (
    <Link href={`/products/${product.id}`}>
      <ViewTransition name={`product-img-${product.id}`}>
        <img
          src={product.image}
          alt={product.name}
          className="w-full aspect-square object-cover rounded-lg"
        />
      </ViewTransition>
      <p className="mt-2 font-medium">{product.name}</p>
    </Link>
  )
}
```

```tsx
// app/products/[id]/page.tsx — detail page
'use client'
import { ViewTransition } from 'react'

export function ProductDetail({ product }: { product: Product }) {
  return (
    <ViewTransition name={`product-img-${product.id}`}>
      <img
        src={product.image}
        alt={product.name}
        className="w-full aspect-square object-cover rounded-lg"
      />
    </ViewTransition>
  )
}
```

**Browser matches** `product-img-{id}` between the two pages and morphs the image position/size automatically. Add CSS to customize the animation.

### Animation Props — `enter`, `exit`, `default`

Use animation class props to control what CSS classes are applied during each transition phase:

```tsx
<ViewTransition
  name="modal-backdrop"
  default="fade-in"     {/* Applied when transition activates (fallback/default) */}
  enter="slide-up"       {/* Applied to entering element */}
  exit="fade-out"        {/* Applied to exiting element */}
>
  <Modal />
</ViewTransition>
```

```css
/* Define animations via view-transition classes */
.fade-in { animation: fadeIn 300ms ease-out; }
.slide-up { animation: slideUp 300ms ease-out; }
.fade-out { animation: fadeOut 200ms ease-out forwards; }

@keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
@keyframes slideUp { from { transform: translateY(20px); opacity: 0; } to { transform: translateY(0); opacity: 1; } }
@keyframes fadeOut { to { opacity: 0; } }
```

**Default vs named transitions:**
| Prop | When Applied |
|---|---|
| `default` | Applied when the transition activates with no specific type |
| `enter` | Applied when element is entering (new) |
| `exit` | Applied when element is exiting (old) |

### `addTransitionType` — Router Integration

React's `addTransitionType()` lets routers annotate transitions with semantic types (e.g., `navigation-forward`, `navigation-back`) so animations can differ by navigation direction:

```tsx
// In your router — annotate the transition type before navigating
import { startTransition } from 'react'

function navigateTo(href: string, direction: 'forward' | 'back') {
  startTransition(() => {
    addTransitionType(`navigation-${direction}`)
    router.push(href)
  })
}
```

```tsx
// In component — ViewTransition reads the type and applies different CSS classes
<ViewTransition
  name="page-content"
  default={{ 'navigation-back': 'slide-right', 'navigation-forward': 'slide-left' }}
>
  {children}
</ViewTransition>
```

**CSS:**
```css
.slide-left { animation: slideFromRight 300ms ease-out; }
.slide-right { animation: slideFromLeft 300ms ease-out; }

@keyframes slideFromRight { from { transform: translateX(100%); } to { transform: translateX(0); } }
@keyframes slideFromLeft { from { transform: translateX(-100%); } to { transform: translateX(0); } }
```

### SSR Safety

**`<ViewTransition>` is SSR-safe** — it only activates `startViewTransition()` when running in a browser. Unlike `document.startViewTransition()` (which throws in SSR), the component gracefully skips the transition during server render.

**When to use `<ViewTransition>` (component) vs `document.startViewTransition()` (browser API):**
| Approach | Use When |
|---|---|
| **`<ViewTransition>` (React)** ✅ | React apps — automatically hooks into render cycle, SSR-safe, `enter`/`exit` props |
| `document.startViewTransition()` | Need fine-grained control over when transition fires; non-React environments |

**Sources:**
- [React ViewTransition component reference](https://react.dev/reference/react/ViewTransition)
- [React ViewTransition blog](https://react.dev/blog/2025/10/01/react-19-2)
- [Kent C. Dodds — ViewTransition tutorial](https://www.epicreact.dev/use-react-view-transition-to-smoothly-transition-images-and-titles-lu6ks)
- [Frontend at Scale — Experimenting with View Transitions](https://frontendatscale.com/issues/43/)

### `<ViewTransition>` `parentEnter` / `parentExit` — Container-Level Transitions (React 19.3.0-canary `3508aee6-20260702`+ (was `ec0fca31-20260701`), [PR #36690](https://github.com/facebook/react/pull/36690) by Jack Pope, merged 2026-07-01T16:16:48Z, behind `enableViewTransitionParentEnterExit` flag)

React 19.2's `<ViewTransition>` only animated the specific child element. The new `parentEnter` / `parentExit` props (also gated behind the experimental `enableViewTransitionParentEnterExit` feature flag in `react/src/ReactFeatureFlags.js`) let the *parent container* also transition when any of its children transition — so the page chrome (header, side nav, toolbar) animates alongside the inner content rather than sitting still underneath.

```tsx
// Without parentEnter/parentExit (React 19.2): the inner <img> morphs, but the <div> wrapping the gallery doesn't
// With parentEnter/parentExit (React 19.3 canary): both the inner image AND its parent <div> animate

'use client'
import { ViewTransition } from 'react'

function ProductGallery({ images }: { images: string[] }) {
  const [selected, setSelected] = useState(0)

  return (
    <ViewTransition
      name={`gallery-wrap-${selected}`}     // child morph
      parentEnter={{ name: 'gallery-fade-in', className: 'gallery-fade-in' }}   // parent enter
      parentExit={{ name: 'gallery-fade-out', className: 'gallery-fade-out' }}  // parent exit
    >
      <ViewTransition name={`gallery-img-${selected}`}>
        <img src={images[selected]} alt="" className="w-full aspect-square object-cover rounded-lg" />
      </ViewTransition>
    </ViewTransition>
  )
}
```

**The API surface** (from the PR description):

```ts
type ViewTransitionProps = {
  name?: string | Array<string>
  // existing 19.2 props
  enter?: string | Array<string>
  exit?: string | Array<string>
  default?: string | Array<string>
  // NEW 19.3 canary
  parentEnter?: { name?: string | Array<string>; className?: string | Array<string> }
  parentExit?:  { name?: string | Array<string>; className?: string | Array<string> }
  parentDefault?: { name?: string | Array<string>; className?: string | Array<string> }
}
```

**Feature flag.** The feature is off by default. To opt in (one project, dev only):

```js
// next.config.js (16.3+)
module.exports = {
  experimental: {
    reactCompiler: false,  // unrelated
    useExperimentalReact: true,
    enableViewTransitionParentEnterExit: true,  // proposed flag name (not yet in next config)
  },
}

// Alternative: pin to a React canary that has the flag enabled
// 19.3.0-canary-3508aee6-20260702 (current; was ec0fca31-20260701) — the flag is exported from `react/src/ReactFeatureFlags.js`
// To force-enable without going through Next.js config, set in a setup file:
//   globalThis.__REACT_FEATURE_FLAGS__ = { enableViewTransitionParentEnterExit: true }
```

The flag isn't yet exposed in `next.config.ts` (the PR was merged 2 days ago, Next.js usually picks up React feature flags via `experimental.useExperimentalReact` + a re-export in 1–2 canaries). For now, the most reliable way to try the feature is to install `react@canary` + `react-dom@canary` in your project and let Next pick up the canary.

**Common use cases:**

| Use case | How to model with `parentEnter`/`parentExit` |
|---|---|
| Page chrome (header, sidebar) fades in/out when content cross-fades | Wrap the content area in a `<ViewTransition parentEnter/parentExit>` |
| Card-to-detail morph where the card's parent list item also animates | Wrap each list item in a parent VT; child VT does the image morph |
| Modal open/close where the modal backdrop and its child dialog both animate | Two VTs nested; the outer one handles the backdrop, the inner one handles the dialog content |
| Page transitions where the entire page chrome transitions as one unit | Wrap the page layout in a single parent VT; child VTs are per-element |

**SSR safety** (unchanged from 19.2): the `startViewTransition()` call only fires in the browser; the parent-level animation classes are also SSR-skipped.

**Source:** [React PR #36690 — `Add parentEnter/parentExit props to ViewTransition`](https://github.com/facebook/react/pull/36690) · [Commit `ec0fca31f`](https://github.com/facebook/react/commit/ec0fca31f419e821018fc67bc88f2ce62ffb2050) · React canary `19.3.0-canary-3508aee6-20260702` (current, npm dist-tag pointer moved 2026-07-02T16:54:01Z, replaces `19.3.0-canary-ec0fca31-20260701` which itself replaced `19.3.0-canary-e2731312-20260630`)

## View Transitions API (React 19.2)

React 19.2 adds support for the **View Transitions API** — a browser-native way to animate between page states or UI updates with smooth, choreographed transitions.

### Browser-Native API (Works Everywhere)

The core API is `document.startViewTransition()`:

```tsx
'use client'

function Modal({ isOpen, onClose, children }: { isOpen: boolean; onClose: () => void; children: React.ReactNode }) {
  function handleToggle() {
    if (isOpen) {
      document.startViewTransition(() => {
        onClose()
      })
    } else {
      onOpen()
    }
  }

  return (
    <>
      <button onClick={handleToggle}>{isOpen ? 'Close' : 'Open'} Modal</button>
      {isOpen && <div className="modal">{children}</div>}
    </>
  )
}
```

### With React State

```tsx
'use client'
import { useState } from 'react'

function ProductGallery({ images }: { images: string[] }) {
  const [selected, setSelected] = useState(0)

  function handleSelect(index: number) {
    document.startViewTransition(() => {
      setSelected(index)
    })
  }

  return (
    <div>
      <img
        src={images[selected]}
        style={{ viewTransitionName: 'product-image' }}
        alt="Product"
      />
      <div className="thumbnails">
        {images.map((src, i) => (
          <button key={i} onClick={() => handleSelect(i)}>
            <img src={src} alt={`Thumbnail ${i}`} />
          </button>
        ))}
      </div>
    </div>
  )
}
```

### CSS View Transition Names

For named transitions (to animate specific elements across state changes), use `view-transition-name`:

```css
/* In your CSS file */
.product-image {
  view-transition-name: product-image;
}

/* Disable transitions for specific elements */
.no-transition {
  view-transition-name: none;
}
```

### Dynamic `viewTransitionName` — List to Detail Animations

The most powerful View Transitions pattern is animating from a card in a grid to a detail page. Use a **dynamic** `viewTransitionName` so each card's image participates in the transition:

```tsx
// app/products/page.tsx — product grid
'use client'

import Link from 'next/link'
import { ProductCard } from '@/components/product-card'

export function ProductGrid({ products }: { products: Product[] }) {
  return (
    <div className="grid grid-cols-3 gap-4">
      {products.map(product => (
        <Link key={product.id} href={`/products/${product.id}`}>
          <ProductCard product={product} />
        </Link>
      ))}
    </div>
  )
}
```

```tsx
// components/product-card.tsx
'use client'

interface ProductCardProps {
  product: { id: string; name: string; image: string }
}

export function ProductCard({ product }: ProductCardProps) {
  return (
    <div className="group cursor-pointer">
      <img
        src={product.image}
        alt={product.name}
        // Dynamic viewTransitionName — each card animates independently
        style={{ viewTransitionName: `product-image-${product.id}` }}
        className="w-full aspect-square object-cover rounded-lg"
      />
      <p className="mt-2 font-medium">{product.name}</p>
    </div>
  )
}
```

```tsx
// app/products/[id]/page.tsx — detail page
import { ProductDetail } from '@/components/product-detail'

export default async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const product = await getProduct(id)

  return (
    <ProductDetail
      product={product}
      // Same dynamic name — browser matches and animates between them
      viewTransitionName={`product-image-${id}`}
    />
  )
}
```

```tsx
// components/product-detail.tsx
'use client'

interface ProductDetailProps {
  product: { id: string; name: string; image: string; description: string }
  viewTransitionName: string
}

export function ProductDetail({ product, viewTransitionName }: ProductDetailProps) {
  return (
    <div className="max-w-2xl mx-auto">
      <img
        src={product.image}
        alt={product.name}
        style={{ viewTransitionName }}
        className="w-full aspect-square object-cover rounded-lg"
      />
      <h1 className="text-3xl font-bold mt-4">{product.name}</h1>
      <p className="mt-2 text-muted-foreground">{product.description}</p>
    </div>
  )
}
```

**How it works:** The browser matches `view-transition-name: product-image-{id}` between the grid card and detail page. When the user navigates from the grid to the detail, the browser morphs the image from its grid position to its detail-page position — automatically.

### Async `startViewTransition` — Await DOM Updates

By default, `startViewTransition` runs the callback synchronously before the transition. For transitions that need to wait for data or DOM updates, use the async variant:

```tsx
async function handleAddToCart() {
  // Option 1: wrap async work in startViewTransition
  await document.startViewTransition(async () => {
    await addToCart(productId)
    await fetch('/api/cart/refresh', { method: 'POST' })
    // DOM has been updated when the transition starts
  }).finished

  // Option 2: update DOM first, then transition
  setCartItems([...cartItems, newItem])
  document.startViewTransition(() => {
    // DOM is already updated — this runs synchronously
  })
}
```

**When to use async:** When the state update itself triggers async work (e.g., Server Action, fetcher) that you want to complete before the transition visually begins.

### View Transitions CSS — Timing and Keyframes

Control the transition animation with CSS:

```css
/* In globals.css or component CSS */

/* Default: both old and new animate (crossfade) */
::view-transition-old(product-image),
::view-transition-new(product-image) {
  animation-duration: 300ms;
  animation-timing-function: ease-out;
}

/* Slide-in for new content (common pattern) */
::view-transition-new(product-image) {
  animation: none;
  clip-path: inset(0);  /* Start from left */
}
::view-transition-new(product-image) {
  animation: slide-in 300ms ease-out;
}

@keyframes slide-in {
  from { transform: translateX(100%); }
  to { transform: translateX(0); }
}

/* Fade only (simpler) */
::view-transition-old(*),
::view-transition-new(*) {
  animation-duration: 200ms;
}
::view-transition-old(*) {
  animation: fade-out 200ms ease-out forwards;
}
::view-transition-new(*) {
  animation: fade-in 200ms ease-out forwards;
}

@keyframes fade-out {
  to { opacity: 0; }
}
@keyframes fade-in {
  from { opacity: 0; }
}
```

### Next.js 16 Integration

For Next.js 16 page transitions, use the `ViewTransition` component from `next/navigation`:

```tsx
// app/layout.tsx
import { ViewTransition } from 'next/navigation'

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <ViewTransition>
      {children}
    </ViewTransition>
  )
}
```

This wraps all `<Link>` navigations in View Transitions automatically.

### Browser Support

The View Transitions API is supported in Chrome 111+, Edge 111+, and Safari 18.2+. For unsupported browsers, the transitions gracefully degrade — the navigation/state change still happens, just without animation.

**Sources:**
- [CSS View Transitions Module Level 1](https://www.w3.org/TR/css-view-transitions-1/)
- [View Transitions API — MDN](https://developer.mozilla.org/en-US/docs/Web/API/View_Transitions_API)
- [React 19.2 release notes](https://react.dev/blog/2025/10/01/react-19-2)
- [Next.js View Transitions (next/navigation)](https://nextjs.org/docs/app/api-reference/components/link)

## Common Mistakes in Composite Patterns

- **Not aborting previous requests** — always use `AbortController` or React Query's built-in cancellation
- **Not handling loading states** — every async operation needs a loading state
- **Not handling errors** — always show error UI, not just console.log
- **Mutations in render** — never call mutation functions in render; always in event handlers
- **Not resetting forms after submit** — `form.reset()` after successful submission
- **Stale closures in useEffect** — use refs or proper dependency arrays
- **Passing server-side data to client without serialization** — Dates must be `.toISOString()`, non-serializable objects can't cross the RSC boundary
- **`use()` with non-Promise** — `use()` only accepts Promises; for regular values just use them directly
- **Mutating props/state with React Compiler** — the compiler skips components that mutate; fix mutations first
- **`cache()` vs `use cache` confusion** — React's `cache()` is client-side function memoization; Next.js `use cache` is server-side persistence; don't confuse the two
- **Using `<Activity>` as a loading spinner** — `<Activity>` only toggles visibility (`display: none`); it is NOT a pending/loading detector. For loading state, use `useActionState`'s `isPending`, `useFormStatus`, or React Query's `isLoading`. Inventing a `detection`/`isActivity` prop pattern from older notes will not work.
- **Expecting Effects to run inside `mode="hidden"`** — Effects are deliberately torn down when hidden. Move analytics, telemetry, and "always on" subscriptions outside the Activity boundary.
- **Wrapping the entire app in a single `<Activity>`** — defeats the purpose; the boundary should match a meaningful UI unit (one tab, one panel, one sidebar, one modal).
- **View Transitions without `::view-transition-*` CSS** — `view-transition-name` only declares the element's identity; without CSS `::view-transition-old`/`::view-transition-new` rules, the browser uses a default crossfade that may look abrupt or wrong for your use case; always add explicit transition CSS
- **View Transitions with duplicate `viewTransitionName`** — two elements on the same page with the same `viewTransitionName` causes the browser to skip the transition silently; use unique names per element (`product-image-${id}` not just `product-image`)
- **`<ViewTransition>` missing `name` prop** — without a `name` prop, React doesn't know which elements should transition together; always use `name` for cross-page or state-change animations
- **View Transitions in SSR without hydration guard** — `document.startViewTransition()` throws in SSR/Server Components; only call it inside event handlers or in Client Components, never during server render
- **`useEffectEvent` as a dependency shortcut** — `useEffectEvent` is for extracting non-reactive logic from Effects; do NOT use it to silence the dependency linter when you should be adding proper dependencies; this hides bugs; instead, only extract logic that genuinely doesn't need to trigger re-runs
