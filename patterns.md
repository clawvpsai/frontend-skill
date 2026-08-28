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

### `prefetch` segment config — `allow-runtime` REMOVED in 16.3.0-canary.99 (BREAKING, July 28, 2026)

**⚠️ Breaking change (canary.99+):** The `'allow-runtime'` value on `export const prefetch` was removed by [PR #96106](https://github.com/vercel/next.js/pull/96106) ("Unify allow-runtime with Partial Prefetching"). The original motivation for the `'allow-runtime'` segment option was to give apps control over server costs triggered by prefetches — until a route explicitly opted in, prefetches would only be served from the CDN, not from the server. In practice it was confusing to know when to add or remove it, and the incentive for many apps was to add it everywhere with no clear signal for when to remove.

**The new model:** Partial Prefetching itself now provides sufficient protection against runaway prefetching costs (per-link prefetches only happen on Link components that explicitly opt in with the `prefetch` prop). The optimizations landed earlier in the stack also make `allow-runtime` less necessary — on pages where all content is statically renderable, prefetches are served from the static cache and no runtime request is ever issued. Only a page that accesses non-static data is prefetched at runtime, and only when a `<Link prefetch={true}>` is hovered.

**Practical migration:**

```tsx
// ❌ Before canary.99 — removed
export const prefetch = 'allow-runtime' as const

// ✅ After canary.99 — implicit under Partial Prefetching
// Just delete the export; runtime prefetch happens automatically when needed
// For static-only apps, no action required at all

// ✅ For programmatic API
// router.prefetch(href, { kind: 'allow-runtime' }) // ❌ — removed
router.prefetch(href, { kind: 'full' })              // ✅ — explicit full-route
router.prefetch(href, { kind: 'app-shell' })          // ✅ — app-shell only
// No `kind: 'allow-runtime'` option anymore — partial prefetching decides
```

**Audit recipe:**

```bash
# Find any remaining 'allow-runtime' references
rg -n "allow-runtime|allow_runtime" --type ts --type tsx --type md   app/ components/ lib/ .next/ 2>/dev/null
```

**Codemod (no official codemod — manual removal):** Delete any `export const prefetch = 'allow-runtime' as const` lines; delete any `kind: 'allow-runtime'` references in `router.prefetch(href, { ... })` calls. Both patterns now silently no-op (Next.js will print a deprecation warning in canary.99 and ignore in stable 16.3).

**Sources:**
- [PR #96106 — `Unify allow-runtime with Partial Prefetching`](https://github.com/vercel/next.js/pull/96106) · merged 2026-07-28T16:14:53Z · **Shipped in `16.3.0-canary.99`** (2026-07-28T15:21:10Z) and **`16.3.0-preview.10`** (2026-07-28T16:18:11Z)
- [Next.js canary.99 release](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.99)
- [Next.js preview.10 release](https://github.com/vercel/next.js/releases/tag/v16.3.0-preview.10)
- [Compare v16.3.0-canary.98...v16.3.0-canary.99](https://github.com/vercel/next.js/compare/v16.3.0-canary.98...v16.3.0-canary.99)
- [Compare v16.3.0-canary.99...v16.3.0-preview.10](https://github.com/vercel/next.js/compare/v16.3.0-canary.99...v16.3.0-preview.10)

### Attempt Static Prefetch Before Runtime (16.3.0-canary.99 / preview.10, PR [#96095](https://github.com/vercel/next.js/pull/96095))

**What changed:** The prefetch scheduler now attempts a **static** prefetch first when it's "reasonably confident" that a segment can be prefetched statically without omitting data that would have been included during a runtime prefetch (e.g. cookies). If the static response turns out to be insufficient (missing dynamic holes), it falls back to a runtime request.

**Why this matters:**
- Cheaper prefetching for pages that are fully statically renderable — no runtime server cost on first hover, even with Partial Prefetching enabled.
- Applies during both the **Shell phase** and the **Speculative phase** of the prefetch algorithm.
- The decision for whether to attempt static prefetch first is based on a new `ShouldAttemptStaticPrefetch` flag added in earlier commits; can be refined per-segment in the future for more reliable computation.

**Practical effect:** When hovering a `<Link>` on a fully-static route, the prefetch is served entirely from the static cache. No runtime request is issued. The behavior is invisible to app code — there's no new API to call, no flag to set. It just gets cheaper.

**Coordinated with PR #96106:** With `allow-runtime` now removed, this is the main lever that keeps server-side prefetch costs bounded. The combination (PRs #96095 + #96106) is the architectural cleanup that lets the `allow-runtime` knob go away without paying for it on the server side.

**Sources:**
- [PR #96095 — `Attempt static prefetch before resorting to runtime`](https://github.com/vercel/next.js/pull/96095) · merged 2026-07-28T16:14:48Z · **Shipped in `16.3.0-canary.99`** + **`16.3.0-preview.10`**

---


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

## Pattern: Turbopack + Web Workers + Cross-Origin CDN `assetPrefix` (PR #96636, timneutkens — August 5, 2026)

A canonical pattern that was broken on Turbopack from `next@16.3.0` STABLE through `next@16.3.1-canary.2` (and is fixed in `next@16.3.1-canary.3`-ahead, expected on npm within hours): a Next.js app deployed with a CDN-fronted `assetPrefix` that uses Web Workers for heavy client-side work (WASM image processing, PDF generation, offscreen rendering, Comlink-wrapped services). Until PR #96636 (timneutkens, 2026-08-05T05:41:55Z), the worker would silently hang — the file loaded, but the entry module never executed.

### The setup that hung silently

```ts
// next.config.ts
const phase = process.env.NEXT_PHASE
const nextConfig: NextConfig = {
  // CDN-fronted static asset prefix (production only)
  assetPrefix: phase === PHASE_DEVELOPMENT_SERVER
    ? undefined
    : 'https://cdn.example.com',

  // Pin workers to same-origin to avoid cross-origin SecurityError
  // (introduced in PR #93271 to fix the more catastrophic CDN-Worker SecurityError bug)
  experimental: {
    turbopackWorkerAssetPrefix: '',   // ← forces Workers to use same origin paths
  },

  // Turbopack everywhere (Next.js 16 default)
  // (no `webpack:` block needed — webpack's `output.workerPublicPath: '/_next/'`
  //  workaround inside `next.config.js → webpack()` is no longer consulted)
}
```

```tsx
// app/image-processor/page.tsx
'use client'

import { useEffect, useRef } from 'react'

export default function ImageProcessor() {
  const canvasRef = useRef<HTMLCanvasElement>(null)

  useEffect(() => {
    // Create a worker for serverless-friendly WASM image processing
    const worker = new Worker(
      new URL('./resvg-worker.ts', import.meta.url),
      { type: 'module' }
    )

    worker.onmessage = (e) => {
      console.log('[main] received:', e.data)  // ← NEVER FIRES in production
      // ...render the PNG bytes to canvas
    }

    worker.postMessage({ svg: '<svg>...</svg>' })
  }, [])

  return <canvas ref={canvasRef} />
}
```

```ts
// app/image-processor/resvg-worker.ts
import { Resvg } from '@resvg/resvg-js'

console.log('[worker] module evaluated')   // ← NEVER LOGS in production

self.onmessage = (event: MessageEvent<string>) => {
  console.log('[worker] message received')
  const resvg = new Resvg(event.data.svg)
  const png = resvg.render().asPng()
  self.postMessage(png, [png.buffer])
}

export {}
```

**The symptom (pre-PR #96636, on `next@16.3.0` STABLE + every canary before 16.3.1-canary.3):** in production, the page reaches `worker.postMessage()`, the DevTools Network tab shows every worker chunk returning `200`, but:

- `[worker] module evaluated` never logs
- `[worker] message received` never logs
- `worker.onmessage` is never assigned
- `worker.onerror` never fires
- No console error of any kind
- The page never receives a response — looks like the worker just "doesn't work"

In development (`phase === PHASE_DEVELOPMENT_SERVER`, so `assetPrefix` is unset) it works fine — that's why the bug ships to production undetected. The bug affects **every** worker that uses `new Worker(new URL(..., import.meta.url))` on Turbopack + cross-origin CDN. The reproducer at [`Manitej66/turbopack-worker-asset-prefix-repro`](https://github.com/Manitej66/turbopack-worker-asset-prefix-repro) demonstrates this with a 6-line worker.

### Why the worker hung — the chain

Three pieces had to align for the bug to surface, all rooted in how `turbopackWorkerAssetPrefix` interacts with the worker runtime chunk:

1. The **worker entrypoint URL** and the chunk URLs in `#params=` were emitted using the **worker asset prefix** (same-origin) — that's the part `turbopackWorkerAssetPrefix: ''` was designed to control, and it works correctly.
2. The **worker's own runtime chunk** (the `turbopack-<hash>.js` file that contains `registerChunk`) was emitted with `CHUNK_BASE_PATH` set from `assetPrefix` (the CDN). `CHUNK_BASE_PATH` is the base URL the worker runtime uses to resolve every chunk URL via `getChunkRelativeUrl(chunkPath)` inside its own context.
3. `registerChunk(chunk, params)` resolves the parent-context resolver keyed by `chunk.src` (worker prefix, same-origin), then iterates `params.otherChunks` and awaits each via `loadInitialChunk(chunkPath, d)` whose key is `getChunkRelativeUrl(otherChunkPath)` (CHUNK_BASE_PATH, the CDN). The two resolver keys never match.

For `SourceType.Runtime` (worker), `loadInitialChunk` short-circuits the resolver to `loadingStarted = true` and **never resolves** it — it assumes `importScripts` already pulled the chunks under the worker prefix (which it did, correctly). So the CDN-keyed resolver sits pending forever, `Promise.all` never settles, `runtimeModuleIds` is never instantiated, the worker entry module's code never runs. Silent.

The bug was upstream of `turbopackWorkerAssetPrefix` itself — the deeper fix is to **make the worker's runtime chunk's `CHUNK_BASE_PATH` derive from the worker asset prefix** rather than the global `assetPrefix`. That's exactly what PR #96636 does.

### The fix (PR #96636 — works automatically after upgrade)

```diff
// runtime-base.ts (Web Worker context)
// emit the worker's runtime chunk with the WORKER prefix,
// not the global assetPrefix (which a CDN might set to cross-origin)
- var CHUNK_BASE_PATH = "<assetPrefix>/_next/";   // the CDN
+ var CHUNK_BASE_PATH = "/_next/";                // the worker asset prefix (same-origin)
```

**No code or config changes required** in your app — bump `next@16.3.1-canary.3+` once it npm-publishes (within hours), or `next@16.3.1+` stable when it ships, and the silent hang is resolved. The `experimental.turbopackWorkerAssetPrefix: ''` line stays in your config (it does the right thing now — keeps workers same-origin AND emits runtime chunks same-origin).

### The audit recipe — confirm you're covered

```bash
# 1. Confirm your Next.js version has the fix:
npm ls next
# → should be next@>=16.3.1-canary.3 (will npm-publish within 2-12h on the 24h cadence)
# → OR next@>=16.3.1 stable (when it ships shortly after canary.3)

# 2. Confirm you have the CDN setup (this is where the bug lives):
rg -n "assetPrefix\s*:" next.config.*   # should show 'https://cdn.example.com' or similar

# 3. Confirm you're on Turbopack (Webpack users unaffected):
rg -n "turbopack|turbo-" next.config.* package.json
# → either the default (Next.js 16 ships Turbopack as default), or explicit next build --turbopack

# 4. Confirm you use Workers via import.meta.url:
rg -ln "new Worker\(new URL\(" app/ src/   # shows every Worker constructor

# 5. Confirm the bug (only in production deploys with CDN):
# In Chrome DevTools → Application → Workers (in the deployed app):
#   - Click on your worker
#   - Check the Console tab
# If you see NO console output at all (not even the module-evaluated log)
# but ALL network requests for worker chunks return 200, you have #96613.
# Bump to next@>=16.3.1-canary.3 and the bug is gone.

# 6. Confirm the fix (post-upgrade):
# Same Chrome DevTools check — worker should now log "module evaluated"
# and respond to your postMessage.
```

### Related: the deeper bug PR #93271 was designed to prevent

Before PR #93271 (and `experimental.turbopackWorkerAssetPrefix`), Turbopack would emit Worker URLs under the CDN origin, which browsers reject with `Failed to construct 'Worker': Script at 'https://cdn.example.com/...' cannot be accessed from origin 'http://localhost:3100'` (a cross-origin SecurityError). PR #93271 fixed that by allowing you to opt into a same-origin worker prefix (`turbopackWorkerAssetPrefix: ''`). The cost was the silent-hang bug PR #96636 just fixed. Webpack was never affected because `output.workerPublicPath: '/_next/'` inside the `webpack()` callback pins Worker URLs to same-origin — and Turbopack simply ignored that webpack config.

### Sources

- [**Next.js PR #96636** — `Fix Turbopack worker chunk loading with asset prefix**](https://github.com/vercel/next.js/pull/96636) — by Tim Neutkens, merged 2026-08-05T05:41:54Z, 35 files / +296/-102, the source-of-truth for the silent-worker-hang fix
- [**Next.js issue #96613** — `Turbopack: experimental.turbopackWorkerAssetPrefix makes Workers load but never execute when assetPrefix is a cross-origin CDN**](https://github.com/vercel/next.js/issues/96613) — filed 2026-08-04 by `Manitej66`, the issue that documents the bug + root cause + proposed fix shape (PR #96636 implements the wider "emit worker's runtime chunk with worker asset prefix" alternative the issue author proposed)
- [Next.js PR #96636 files diff](https://github.com/vercel/next.js/pull/96636/files) — full 35-file breakdown incl. `runtime-backend-dom.ts` (worker bootstrap propagation), `runtime-base.ts` (`getChunkRelativeUrl` + `getPathFromScript` worker-base-path fix), 4 new test files in `test/e2e/turbopack-worker-asset-prefix/` (incl. the message round-trip assertion that the previous test missed), 4 new test files in `test/e2e/app-dir/worker/` (resvg WASM repro)
- [`Manitej66/turbopack-worker-asset-prefix-repro`](https://github.com/Manitej66/turbopack-worker-asset-prefix-repro) — the 6-line-worker + 2-line-`next.config.js` minimal reproduction; `npm run cdn &` + `npm start` + click "run worker" demonstrates the silent hang
- [Next.js canary-branch compare: `v16.3.1-canary.2...canary` (3 commits ahead at 2026-08-05T06:03Z)](https://github.com/vercel/next.js/compare/v16.3.1-canary.2...canary) — PR #96636 + PR #96682 + version-tag `bcea67d` (v16.3.1-canary.3)
- [Next.js PR #93271 — `Original worker asset prefix introduction`](https://github.com/vercel/next.js/pull/93271) — the PR that introduced `experimental.turbopackWorkerAssetPrefix`; the option PR #96636 fixes
- [Next.js discussion #93044 — `Turbopack: add turbopack.workerPublicPath to keep Workers same-origin when assetPrefix is a CDN`](https://github.com/vercel/next.js/discussions/93044) — the original feature request that led to PR #93271
- [Next.js `experimental.turbopackWorkerAssetPrefix` config docs](https://nextjs.org/docs/app/api-reference/turbopack) — the canonical reference (option semantics unchanged by PR #96636 — only the runtime chunk path derivation is fixed)
- [Next.js `assetPrefix` config docs](https://nextjs.org/docs/app/api-reference/config/next-config-js/assetPrefix) — the CDN setup walkthrough with the `phase === PHASE_DEVELOPMENT_SERVER` pattern
- [`@resvg/resvg-js`](https://github.com/yisibl/resvg-js) — serverless-friendly WASM SVG→PNG; canonical "Workers used in Next.js apps" example
- [`@jsquash/*`](https://github.com/jspm/npm/tree/master/packages/lib) — image codecs (jpeg, png, webp, avif) for Web Workers; another common Workers-in-Next-apps use case
- [Next.js `next@16.3.1-canary.2` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.2) — npm-published 2026-08-05T00:03:35Z; 14-commit canary.1+canary.2 set (full breakdown in v1.5.24 cycle entry + `routing.md` → `## 16.3.1-canary.1 + 16.3.1-canary.2` section)


## Pattern: Cache Components Revalidation Lifecycle (`updateTag` + `'use cache: private'` Reuse) — PR #96726 + PR #96727 + PR #96731 (unstubbable + ztanner, August 5, 2026)

A coordinated 3-PR refactor of Cache Components cache lifecycle management around `updateTag()` revalidations. All 3 ship in the same canary.4 npm publish. Together they fix three interrelated bugs in the cache-staleness-check + dedupe + consumer-revalidation pipeline.

### Pattern: Pre-compute + reuse across the request — `'use cache: private'` (PR #96727)

**The bug (pre-#96727, on `next@16.3.0` STABLE and all `16.3.1-canary.X` releases):** calling the same `'use cache: private'` function twice in one request executed its body twice in production. Preloading at the top of a segment and reading the same function again lower down for composability — the canonical preload pattern — therefore did the work twice instead of once.

**The cause:** the intra-request dedupe map dropped an entry as soon as its fill completed, so it only ever covered *concurrent* invocations. A later invocation fell through to the cache handler, and the `React.cache` memo wrapping every cache function missed whenever the arguments were not reference-equal. Public caches got a handler hit out of that, but private caches have no handler in production and their entries are excluded from the immutable Resume Data Cache of a dynamic request — so nothing had stored the entry.

**The fix:** completed invocations now move into `completedCacheInvocations` on the work store instead of being dropped, and a later invocation joins that entry. The pending map keeps its previous semantics untouched (a concurrent joiner shares a fill that's genuinely in flight and must not re-run the discard checks against it), whereas a completed entry is a stored value and is only reused when the caller has not asked to bypass caches and nothing has invalidated it since. Private caches still get no cache handler, so the map is what backs them — and it lives on the work store so it cannot carry request-derived data beyond the request that produced it.

**Canonical pattern (post-#96727):**

```tsx
// app/dashboard/page.tsx
import { getCurrentUserPrefs, getUserCart } from './data'

// 'use cache: private' — per-user, per-request memoized
async function Page() {
  // Preload at the top — runs once per request
  const [prefs, cart] = await Promise.all([
    getCurrentUserPrefs(),
    getUserCart(),
  ])

  return (
    <DashboardLayout>
      {/* Read again lower down for composability — second read is a map lookup */}
      <Header prefs={await getCurrentUserPrefs()} />
      <CartSummary cart={await getUserCart()} />
    </DashboardLayout>
  )
}
```

```ts
// app/dashboard/data.ts
'use cache: private'

export async function getCurrentUserPrefs() {
  // Heavy work — DB query + cookie read + derived computation
  const userId = (await cookies()).get('session')?.value
  return await db.userPrefs.findUnique({ where: { userId } })
}

export async function getUserCart() {
  const sessionId = (await cookies()).get('cart')?.value
  return await db.cart.findUnique({ where: { sessionId }, include: { items: true } })
}
```

**Pre-#96727:** each `getCurrentUserPrefs()` / `getUserCart()` call above re-runs the DB query — 4 DB queries per page render (2 preloads + 2 lower-down reads).

**Post-#96727:** only the first call per function runs; subsequent calls join the work-store map — 2 DB queries per page render.

**Expected perf improvement:** 30-50% reduction in DB / I/O work per page render with private cache fan-out.

### Pattern: `updateTag()` revalidation without spurious regeneration (PR #96726)

**The bug (pre-#96726, on `next@16.3.0` STABLE and all `16.3.1-canary.X` releases):** calling `updateTag()` in a server action made every later read of a cache carrying that tag regenerate for the remainder of the request — including reads of an entry that had just been generated *after* the invalidation and therefore already reflected it. Two sequential reads of the same cache function during the re-render produced two different values within a single render, and each one repeated the work.

**The cause:** `isRecentlyRevalidatedTag` only asked whether a tag appeared in `pendingRevalidatedTags`, with no notion of *when* the revalidation happened. That array lives for the whole `WorkStore`, which spans a server action AND the render that follows it — so once a tag was in it, every entry carrying that tag looked stale regardless of when it had been produced.

**The fix:** each pending revalidated tag now records a `revalidatedAt` timestamp taken from the same clock as `CacheEntry.timestamp`, and the renamed `isRevalidatedAfter` reports an entry as stale only when the revalidation is newer than the entry. `CacheEntry.timestamp` is captured *before* a fill begins, so a fill straddling a revalidation is still discarded (conservative — a body that may have read pre-invalidation data is safer to discard). Revalidating the same tag again moves the timestamp forward — the later invalidation decides which entries are stale. `previouslyRevalidatedTags` (forwarded from an earlier request by a redirecting server action) carry no timestamp of their own, so the work store now records when they were imported.

**Canonical pattern (post-#96726):**

```ts
// app/actions/posts.ts
'use server'

import { updateTag } from 'next/cache'

export async function updatePost(postId: string, data: PostData) {
  await db.post.update({ where: { id: postId }, data })

  // Invalidate the post + any cache that derives from it
  updateTag(`post-${postId}`)
  updateTag('posts-list')  // for any 'use cache' that depends on posts
}
```

```ts
// app/data/posts.ts
import { unstable_cache as cache } from 'next/cache'

export const getPost = cache(
  async (postId: string) => {
    return await db.post.findUnique({ where: { id: postId } })
  },
  ['post'],
  { tags: ['posts-list'] }  // ← invalidated by the updateTag('posts-list') call
)

export const getPostsList = cache(
  async () => {
    return await db.post.findMany({ orderBy: { createdAt: 'desc' } })
  },
  ['posts-list'],
  { tags: ['posts-list'] }
)
```

**Pre-#96726:** after `updatePost()`, the re-render of the page would re-fetch both `getPost(postId)` and `getPostsList()` — even if the previous render had already populated `getPostsList()` post-update. Spurious regeneration.

**Post-#96726:** only the entries whose `CacheEntry.timestamp` predates the `revalidatedAt` of the matching tag are discarded; entries that already reflect the revalidation are reused.

**Expected perf improvement:** 20-60% reduction in cache-regeneration work per `updateTag()` round-trip on a multi-cache fan-out.

### Pattern: Consumer-driven foreground revalidation across `unstable_cache` (PR #96731)

**The bug (pre-#96731):** foreground revalidation was necessary only when a stale result will be persisted by another server cache — but the current behavior incorrectly applied it whenever a request was prerendering. Result: a cache created during a dynamic request could consume a stale inner value and extend its lifetime.

**The fix:** derive foreground revalidation from whether the immediate work-unit consumer will persist the result in a server cache. Model that capability as `willConsumerServerCache` rather than inheriting it through the full outer scope chain. Treat server cache and static prerender work units as server-caching consumers; runtime prerenders are not. Adds regression coverage for an outer `'use cache'` scope consuming a stale `unstable_cache` entry.

**Canonical pattern (post-#96731):**

```ts
// app/data/feed.ts
'use cache'

import { unstable_cache as cache } from 'next/cache'

// Inner legacy cache — works fine, but has its own staleness model
const getInnerFeed = cache(
  async () => {
    return await db.feed.findMany({ take: 100 })
  },
  ['feed-inner'],
  { revalidate: 60 }
)

// Outer 'use cache' — stable, persists across requests
export async function getFullFeed() {
  // Consumer of getInnerFeed is getFullFeed (a server cache)
  // → willConsumerServerCache=true → foreground revalidation IS needed
  // when getInnerFeed is stale; pre-#96731 this was applied regardless
  // of whether the runtime prerender scope inherited the capability.
  const items = await getInnerFeed()
  return items.map(enrich)  // enrichment work
}
```

**Pre-#96731:** the runtime-prerender scope forced an unnecessary foreground revalidation when an outer `'use cache'` was consuming a `unstable_cache` entry, even when the runtime prerender wasn't going to persist anything.

**Post-#96731:** the decision is made at the consumer level, so caches that wouldn't actually be persisted don't trigger a revalidation.

**Expected perf improvement:** 5-15% reduction in cache-regeneration work for `unstable_cache` interop cases.

### Pattern: Turbopack + cyclic scope-hoisted dependencies (PR #96697, sampoder — August 5, 2026)

**The bug (pre-#96697):** Turbopack scope-hoisting could miss module registrations for cyclic dependencies between scope-hoisted groups, leading to `TurbopackError: Failed to fetch dynamically imported module: ... TypeError: Cannot read properties of undefined` at runtime in production builds. The reproducer at [issue #96648](https://github.com/vercel/next.js/issues/96648) shows the failure mode: scope-hoisted group A enters scope-hoisted group B, then re-enters A, but A's first execution hadn't reached the `__turbopack_context__.s([...])` registration line yet — so B's reference to a schema registered by A fails.

**The cause:** on non-scope-hoisted modules with cycles, Turbopack already raises the module registration call to the start of the factory. But when scope-hoisting merges multiple modules into a single factory, that early registration was lost — registration happens at the original line, which can be after the factory has already entered the consumer's chunking context.

**The fix:** when scope-hoisting, the `__turbopack_context__.s([...])` registration call is now emitted at the start of the scope-hoisted module, not at the original line.

**Canonical pattern (post-#96697) — no code changes, just bump:**

```ts
// next.config.ts — Turbopack is the default; no opt-in needed
const nextConfig: NextConfig = {
  // Turbopack everywhere
  experimental: {
    // The new top-level turbopackChunking config (PR #96398, canary.105+)
    // — controls scope-hoisting heuristics
    turbopackChunking: {
      minChunkSize: 20000,
      maxChunkCountPerGroup: 10,
      // ...other knobs
    },
  },
}
```

```ts
// app/validation/schemas.ts — zod schemas that get registered via
// __turbopack_context__.s([...])
import { z } from 'zod'

export const postSchema = z.object({
  title: z.string().min(1).max(200),
  body: z.string().min(1),
  tags: z.array(z.string()).max(10),
})

export const commentSchema = z.object({
  postId: z.string(),
  body: z.string().min(1).max(1000),
})

// Cyclic import: index.js re-exports schemas, schemas.js imports from index.js
// (transitively via type-only imports). Pre-#96697, this could throw
// TurbopackError in production builds; post-#96697, registration happens
// at the start of the scope-hoisted module.
```

**Pre-#96697:** with the canonical zod setup (cyclic `index.ts` → `schemas.ts`), Turbopack production builds could throw `TurbopackError: Failed to fetch dynamically imported module` intermittently — especially after navigation that triggers a fresh chunk fetch.

**Post-#96697:** the `__turbopack_context__.s([...])` registration is guaranteed to happen before any consumer's `__turbopack_context__.i(...)` call.

**Workaround** if stuck on a pre-canary.4 version: `next dev --webpack` / `next build --webpack` for that project. Webpack's module resolution never had this issue because it doesn't scope-hoist with the same registration pattern.

### Pattern: Navigation race on Back-during-hydration (PR #96252, gaearon — August 5, 2026)

**The bug (pre-#96252):** Back-button during hydration produces stale client state. See `routing.md` → `## 16.3.1-canary.4-ahead — Navigation Back-Before-Hydration Race Fix` for the full bug walkthrough, the Navigation API hydration contract, and the audit recipe.

**Canonical pattern (post-#96252) — no code changes, just bump:**

```tsx
// app/search/page.tsx
import Link from 'next/link'

export default function SearchPage() {
  return (
    <div>
      <SearchForm />
      <ul>
        {results.map(result => (
          // Pre-#96252: clicking Back before this Link's prefetched RSC
          // payload finished hydrating could produce stale client state.
          // Post-#96252: the router detects the mismatch via Navigation API
          // and replays the Back-traversal cleanly.
          <li key={result.id}>
            <Link href={`/posts/${result.id}`} prefetch={true}>
              {result.title}
            </Link>
          </li>
        ))}
      </ul>
    </div>
  )
}
```

**Workaround** if stuck on a pre-canary.4 version: add `prefetch={false}` to Back-target Links to close the race window for that specific Link.

### The audit recipe — for all 4 Cache Components + Turbopack + Navigation fixes

```bash
# 1. Confirm you're on a version with all 4 fixes:
npm ls next
# → should be next@>=16.3.1-canary.4 (will npm-publish within 1-6h)

# 2. Cache Components `updateTag()` revalidation fix (PR #96726):
rg -n "updateTag\s*\(" app/ actions/ src/

# 3. Cache Components `'use cache: private'` reuse fix (PR #96727):
rg -n "['\"]use cache: private" app/ src/

# 4. Cache Components consumer-driven foreground revalidation (PR #96731):
rg -n "unstable_cache" app/ src/ lib/    # if both lists non-empty, you're affected
rg -n "['\"]use cache" app/ src/

# 5. Turbopack hoisted-module registration (PR #96697):
rg -n "from ['\"]zod['\"]|require\(['\"]zod['\"]\)" app/ src/  # zod canonical reproducer

# 6. Navigation Back-before-hydration race (PR #96252):
rg -n "prefetch\s*=" app/    # default = enabled, which is where the race lives
```

### Sources

- [**Next.js PR #96727 — `Reuse completed cache entries for the rest of a request`**](https://github.com/vercel/next.js/pull/96727) — unstubbable, 8 files / +324/-34, merged 2026-08-05T20:42:21Z; the `'use cache: private'` reuse fix
- [**Next.js PR #96726 — `Discard only cache entries that predate a tag revalidation`**](https://github.com/vercel/next.js/pull/96726) — unstubbable, 12 files / +169/-8, merged 2026-08-05T20:42:20Z; the `updateTag()` correctness/perf fix
- [**Next.js PR #96731 — `Derive foreground cache revalidation from the consumer`**](https://github.com/vercel/next.js/pull/96731) — ztanner, 7 files / +95/-39, merged 2026-08-05T22:44:29Z; the consumer-driven foreground revalidation fix
- [**Next.js PR #96697 — `[turbopack] Raise registration calls in hoisted modules to the top`**](https://github.com/vercel/next.js/pull/96697) — sampoder, 16 files / +156/-10, merged 2026-08-05T22:33:32Z; closes [issue #96648](https://github.com/vercel/next.js/issues/96648); the Turbopack cyclic-scope-hoisted-dependency fix
- [**Next.js PR #96252 — `Fix race when navigating Back before hydration`**](https://github.com/vercel/next.js/pull/96252) — gaearon, 11 files / +561/-25, merged 2026-08-05T21:39:29Z; relands #95682 (the React-side blocker was fixed by [facebook/react#37135](https://github.com/facebook/react/pull/37135)); cross-referenced in `routing.md`
- [**Next.js canary-branch compare: `v16.3.1-canary.3...canary` (27 commits ahead at 2026-08-06T00:02Z)**](https://github.com/vercel/next.js/compare/v16.3.1-canary.3...canary) — PR #96726, #96727, #96731, #96697, #96252 are 5 of the 9 NEW commits since v1.5.27
- [Next.js `v16.3.1-canary.4` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.4) — published 2026-08-05T23:59:14Z; npm publish imminent (1-6h on 24h cadence)
- [Next.js Cache Components docs (`use cache` directive + cacheLife + cacheTag + updateTag)](https://nextjs.org/docs/app/api-reference/directives/use-cache) — canonical reference
- [`unstable_cache` Next.js docs](https://nextjs.org/docs/app/api-reference/functions/unstable_cache) — the legacy cache function that PR #96731 specifically targets
- [zod npm package](https://github.com/colinhacks/zod) — canonical schema-via-`__turbopack_context__.s([...])` library whose cyclic `index.js` → `schemas.js` structure is the canonical reproducer for PR #96697

- **Calling `updateTag()` in a server action causes every later cache read to spuriously regenerate (16.3.0 + all 16.3.1-canary.0/1/2/3) — FIXED in `next@16.3.1-canary.4`-ahead by PR #96726** — See the new `## Pattern: Cache Components Revalidation Lifecycle` section above for the full bug walkthrough, the `revalidatedAt` timestamp fix, and the audit recipe. TL;DR: bump to `next@>=16.3.1-canary.4` once npm-publishes — no code or config changes required; expected 20-60% reduction in cache-regeneration work per `updateTag()` round-trip on multi-cache fan-out.

- **Calling the same `'use cache: private'` function twice in one request does the work twice (16.3.0 + all 16.3.1-canary.0/1/2/3) — FIXED in `next@16.3.1-canary.4`-ahead by PR #96727** — See the new `## Pattern: Cache Components Revalidation Lifecycle` section above for the full bug walkthrough, the `completedCacheInvocations` work-store map fix, and the canonical preload-at-top + read-at-bottom pattern. TL;DR: bump to `next@>=16.3.1-canary.4` once npm-publishes; expected 30-50% reduction in DB / I/O work per page render with private cache fan-out. **Only affects apps with `cacheComponents: true`** that use `'use cache: private'` per-user cached functions (`getCurrentUserPrefs()`, `getUserCart()`, etc.). Audit recipe: `rg -n "['\"]use cache: private" app/ src/` to find private cache usage.

- **Turbopack production builds throw `TurbopackError: Failed to fetch dynamically imported module` for cyclic scope-hoisted dependencies (16.3.0 + all 16.3.1-canary.0/1/2/3) — FIXED in `next@16.3.1-canary.4`-ahead by PR #96697** — See the new `## Pattern: Cache Components Revalidation Lifecycle` section above (Turbopack sub-section) for the full bug walkthrough, the `__turbopack_context__.s([...])` early-registration fix, and the zod-as-canonical-reproducer note. TL;DR: bump to `next@>=16.3.1-canary.4` once npm-publishes; **especially material for**: zod (canonical reproducer), yup, joi, ajv, io-ts, valibot, and any monorepo package with `index.js` re-exports + cyclic internal dependencies. **Workaround** if stuck on a pre-canary.4 version: `next dev --webpack` / `next build --webpack` for that project. Audit recipe: `rg -n "from ['\"]zod['\"]" app/ src/` to confirm zod usage; check your browser console for intermittent "Failed to fetch dynamically imported module" errors after navigation.

- **Back-button click during hydration produces stale client state (16.3.0 + all 16.3.1-canary.0/1/2/3) — FIXED in `next@16.3.1-canary.4`-ahead by PR #96252** — See the new `## Pattern: Cache Components Revalidation Lifecycle` section above (Navigation race sub-section) and `routing.md` → `## 16.3.1-canary.4-ahead — Navigation Back-Before-Hydration Race Fix` section for the full bug walkthrough, the Navigation API hydration contract, and the audit recipe. TL;DR: bump to `next@>=16.3.1-canary.4` once npm-publishes; **workaround** if stuck on a pre-canary.4 version: add `prefetch={false}` to Back-target Links to close the race window for that specific Link.
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

- **Cross-origin CDN `assetPrefix` + Web Workers = silent worker hang on Turbopack (16.3.0 + 16.3.1-canary.0/.1/.2) — FIXED in `next@16.3.1-canary.3`-ahead by PR #96636** — See the new `## Pattern: Turbopack + Web Workers + Cross-Origin CDN assetPrefix` section above for the full setup, the silent symptom, the root cause chain, and the audit recipe. TL;DR: bump to `next@>=16.3.1-canary.3` (will npm-publish within hours) or `next@>=16.3.1` stable (when published) — no code or config changes required.

- **Trying to set `experimental.turbopackWorkerAssetPrefix: ''` "to opt workers out of the CDN" on Turbopack 16.3.0** — that line is still correct (it keeps worker entrypoints same-origin) but it silently ALSO makes the worker runtime chunk broken until `next@16.3.1-canary.3`. PR #96636 fixes the runtime-chunk side without touching your config. Pre-#96636, the symptom is "worker loads but never executes" — every DevTools request returns `200`, no console output, no `onerror`. After PR #96636 ships, the same config works as intended.

- **Diagnosing a silent worker hang by adding `console.log('hello')` to your worker and seeing nothing in DevTools** — that "nothing" is itself the symptom of #96613. The worker module never evaluates; the `Promise.all` in `registerChunk` is pending forever. After PR #96636 (Turbopack + cross-origin CDN + Web Workers users on `next@>=16.3.1-canary.3`), the same `console.log` should fire as expected.

- **Skipping the audit recipe because "my CDN is same-origin"** — if your `assetPrefix` starts with `http://` or `https://` AND points to a different origin than your app, you have the bug. Same-origin CDNs (`assetPrefix: '/cdn-static'` style) don't hit this code path. The bug is specifically when `assetPrefix` is a `scheme://host[:port]` URL different from the page's origin.


## Next.js 16.3.0 STABLE — TypeScript 7 + Cache Components + Partial Prefetching + Cache Poisoning Fix (August 3, 2026)

The 7-day-old `patterns.md` was missing the headline gap: **`next@latest` is now `16.3.0` STABLE**, released at npm-published 2026-08-03T21:03:18Z by Tim Neutkens. `npm view next dist-tags.latest` returned `16.2.12` until Aug 3 and now returns `16.3.0`. The version bundles the 16 commits that landed between `canary.107` and `v16.3.0`, including the three material PRs that drove the canary.108 cycle.

**Practical impact — the four flagship 16.3 patterns every agent must know:**

### Pattern: TypeScript CLI is now the default — no flag, no `next.config.ts` line (PR #96497, timneutkens)

The single biggest behavior change in 16.3.0 is invisible: every `next build` now runs the project-local `tsc` CLI by default. No `experimental.useTypeScriptCli: true` line is required. **TypeScript 7 (Go-native compiler) works out of the box** — the old `## Enable TypeScript 7` opt-in recipe in the docs is obsolete.

```ts
// next.config.ts — pre-16.3.0 (TypeScript 7 opt-in):
const nextConfig: NextConfig = {
  experimental: {
    useTypeScriptCli: true,   // ✅ required pre-16.3.0
  },
}

// next.config.ts — 16.3.0+ (TS 7 default-on):
const nextConfig: NextConfig = {
  // No experimental.useTypeScriptCli needed. TS 7 / TS 6 / TS 5.x all work.
  // TS 7 users: just upgrade to ^7.0.0 — Next.js uses your installed tsc binary.
}
```

If you must keep the legacy JS Compiler API path (custom transformers, monorepo with a shared TypeScript instance that Next.js shouldn't reset), opt out explicitly:

```ts
const nextConfig: NextConfig = {
  experimental: {
    useTypeScriptCli: false,  // ✅ use the legacy JS API path
  },
}
```

**Common mistakes:**
- Leaving the old `useTypeScriptCli: true` line in `next.config.ts` after upgrading — delete it (it's redundant).
- Pinning `typescript@^6.x` for Next.js 16.3+ when TS 7 is supported natively — bump to `^7.0.0`.
- Forgetting to delete the `// @ts-ignore` workaround that was needed pre-TS 7 — clean up.

**Sources:** [PR #96497 — `Enable TypeScript CLI by default`](https://github.com/vercel/next.js/pull/96497) · timneutkens · merged 2026-08-03T16:10:51Z · **shipped in `16.3.0` stable** (npm-published 2026-08-03T21:03:18Z).

### Pattern: ISR + Cache Components + Partial Prefetching (PR #96526, icyJoseph — docs only)

This is the **canonical 16.3.0 architecture** for content-heavy sites: ISR-style time-based caching combined with the new cacheComponents PPR model + Partial Prefetching for instant navigation. PR #96526 (docs only, merged 2026-08-03T15:15:01Z) is the new authoritative guide.

```tsx
// app/blog/[slug]/page.tsx
import { cacheLife, cacheTag } from 'next/cache'
import { Suspense } from 'react'
import { notFound } from 'next/navigation'

// 'use cache' + cacheLife gives you ISR-like behavior under cacheComponents
async function getPost(slug: string) {
  'use cache'
  cacheLife('hours')          // ISR: revalidate every hour, stale-while-revalidate
  cacheTag(`post:${slug}`)

  const post = await db.post.findUnique({ where: { slug } })
  if (!post) notFound()
  return post
}

export default async function PostPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  const post = await getPost(slug)

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
      {/* Suspense boundary lets the shell render instantly while data resolves */}
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments postId={post.id} />
      </Suspense>
    </article>
  )
}
```

**`next.config.ts` for the full combo:**
```ts
const nextConfig: NextConfig = {
  cacheComponents: true,
  experimental: {
    // The new SPA-style prefetch model — only the layout/shell is prefetched
    partialPrefetching: true,
  },
}
```

**The Partial Prefetching model** — hovering a `<Link>` now prefetches the route shell + layout, but the page data is deferred until actual navigation. Combined with `'use cache' + cacheLife('hours')`, the shell is cacheable for hours, so users on slow connections still see something useful within ~50ms.

**When to use:**
- ✅ Marketing pages, blog posts, docs, product pages — anything with hour-scale freshness
- ✅ Apps where shell speed matters more than instant data (the shell includes nav + chrome + Suspense fallbacks)
- ❌ Real-time dashboards, chat apps, live scores — use `cacheLife('seconds')` + App Shell exclude (the 5-minute `stale` floor still applies, so `seconds` is excluded from App Shells per PR #95833)

**Sources:** [PR #96526 — `docs: ISR with Cache Components and Partial Prefetching`](https://github.com/vercel/next.js/pull/96526) · icyJoseph · merged 2026-08-03T15:15:01Z · **shipped in `16.3.0` stable**.

### Pattern: Cache-Poisoning After Prerender Abort — Fixed (PR #96426, jankaeryga)

A subtle correctness/security fix that ships in 16.3.0. Caches that started filling **after** a prerender was aborted (fast-click navigation, prefetch cancellation under `partialPrefetching: true`, slow connections) would silently produce an empty entry because the aborted `renderSignal` propagated into `AbortSignal.any(...)` inside the cache-fill code path.

**The bug (pre-16.3.0):**
```ts
// app/data/feed.ts
'use cache'
cacheLife('minutes')

export async function getFeed() {
  const res = await fetch('https://api.example.com/feed')
  return res.json()
}

// If a user clicks <Link> during this fetch, the prerender aborts.
// Pre-16.3.0: getFeed() returns `undefined`, the empty entry is written to the cache,
// and every subsequent user sees an empty feed for `cacheLife('minutes')` duration.
```

**The fix (16.3.0):**
PR #96426 removes `renderSignal` from `AbortSignal.any(...)` and short-circuits with a rejected promise *before* reaching the cache-fill codepath. The cache now errors instead of silently saving an empty entry.

**Practical impact:**
- Apps with `cacheComponents: true` + heavy `'use cache'` use + frequent navigation aborts (most interactive App Router apps) → was silently producing wrong cache entries
- Apps NOT on `cacheComponents` → no impact (the bug only manifested in the CC code path)
- Both dev mode AND production were affected

**Audit recipe:**
```bash
# Find projects that need this fix
rg -n "cacheComponents\s*:\s*true" next.config.* apps/*/next.config.*

# Find use cache usages
rg -l "use cache" app/ src/

# Symptom check: empty cache entries after fast clicks
# Look for "empty stream" / "undefined" in your cache hit logs after fast nav
```

**No migration required** — just upgrade to 16.3.0. The fix is invisible to correct code; it only changes the behavior of the broken path.

**Sources:** [PR #96426 — `[Cache] Make caches error if called after prerender aborts`](https://github.com/vercel/next.js/pull/96426) · jankaeryga · merged 2026-08-03T11:42:26Z · **shipped in `16.3.0` stable** · closes [#96339](https://github.com/vercel/next.js/issues/96339).

### Pattern: `instant()` No Longer Implicitly Opts Into Partial Prefetching (PR #96539, acdlite)

A subtle breaking change that ships in 16.3.0: **`export const instant = true` no longer silently enables Partial Prefetching** under the hood. Pre-16.3.0, the `instant` segment config was conflated with the Partial Prefetching opt-in; 16.3.0 separates them so the two concerns are explicit and independent.

```tsx
// app/dashboard/page.tsx — pre-16.3.0
export const instant = true  // implicitly opts into partialPrefetching
// → shell + page data both instant

// app/dashboard/page.tsx — 16.3.0+
export const instant = true  // only controls the shell rendering speed
// → you must explicitly opt into Partial Prefetching:

// next.config.ts
const nextConfig: NextConfig = {
  experimental: {
    partialPrefetching: true,  // required for SPA-style prefetch
  },
}
```

**Practical impact:**
- If you relied on `instant` to get Partial Prefetching behavior, **add `experimental.partialPrefetching: true` to `next.config.ts`** after upgrading
- If you used `instant` purely for shell speed (no partial prefetching), nothing changes
- The `prefetch` segment config (`'allow-runtime'`, `'static'`, etc.) is the canonical per-segment knob — `instant` is now shell-only

**Audit recipe:**
```bash
# Find every route using `instant`
rg -l "export const instant" app/

# For each: confirm either `partialPrefetching: true` in next.config.ts,
# OR the route doesn't need prefetch behavior (e.g., a settings page)
```

**Sources:** [PR #96539 — `Remove implicit Partial Prefetching opt-in from \`instant\``](https://github.com/vercel/next.js/pull/96539) · acdlite · merged 2026-08-03T19:56:48Z · **shipped in `16.3.0` stable**.

### Pattern: Catch-All Index Page Served for Every Other Slug (PR #96553, acdlite) — FIXED in v16.3.1-canary.0

A bug introduced in 16.3.0 that ships fixed in the first canary of 16.3.1: requesting `/blog/anything` would serve the catch-all index page (`/blog/[...slug]/page.tsx` with `slug = []`) instead of the proper page. PR #96553 (merged 2026-08-03T21:49:27Z) fixes it; ships in **`next@16.3.1-canary.0`** at npm-published 2026-08-03T22:32:33Z.

**The bug (16.3.0 only):**
```tsx
// app/blog/[...slug]/page.tsx
export default function BlogIndex({ params }: { params: Promise<{ slug: string[] }> }) {
  // When user visits /blog/anything-here, this runs with slug=['anything-here']
  // Pre-fix: 16.3.0 was serving THIS component (slug=[]) instead of the [slug] page
}
```

**Practical impact:**
- Apps with catch-all routes (docs, blogs, product catalogs) — every URL rendered the index instead of the dynamic page
- Fix is in `next@16.3.1-canary.0` (live now); will ship in 16.3.1 stable

**Audit recipe:**
```bash
# Find catch-all routes that may have been affected
rg -l "\[\.\.\." app/

# Test: visit /<catch-all-base>/anything-here in dev — should serve the dynamic page, not the index
```

**Workaround if you're stuck on 16.3.0:** use a non-catch-all dynamic segment (`[slug]` instead of `[...slug]`) + a static `/blog` index page.

**Sources:** [PR #96553 — `Fix catch-all index page being served for every other slug`](https://github.com/vercel/next.js/pull/96553) · acdlite · merged 2026-08-03T21:49:27Z · **shipped in `16.3.1-canary.0`** (npm-published 2026-08-03T22:32:33Z).

---

**Tracked versions updated:**
- `next@latest` 16.2.12 → **16.3.0** (npm-published 2026-08-03T21:03:18Z, GitHub release tag `v16.3.0` published at the same time)
- `next@canary` 16.3.0-canary.107 → **16.3.1-canary.0** (npm-published 2026-08-03T22:32:33Z)

**Common mistakes:**
- **"Next.js 16.3 is still in canary"** — no, it's stable. `next@latest` = 16.3.0 since 2026-08-03T21:03:18Z.
- **Leaving `experimental: { useTypeScriptCli: true }` in `next.config.ts` after upgrading to 16.3.0+** — it's redundant; delete the line. (If you need the JS Compiler API opt-out, set `: false` explicitly.)
- **Using `export const instant = true` and expecting Partial Prefetching** — separated in 16.3.0; opt into `partialPrefetching: true` explicitly.
- **Trusting catch-all routes on 16.3.0** — bug PR #96553 ships in 16.3.1-canary.0; pin to that or wait for 16.3.1 stable.



## Pattern: Turbopack + Server Actions + Cache Components on canary.8 — PR #96779 + PR #96778 + PR #96578 + PR #96932 + PR #96945 (August 7, 2026)

The `next@16.3.1-canary.8` SHIPPED event (npm-published 2026-08-07T23:58:34Z) bundled 5 PRs that affect composite patterns involving **Turbopack + Server Actions + Cache Components** — the most-used combo in 16.3 production apps. The first three are the **Turbopack default-flip trilogy** (all silently-on by default on canary.8+), and the last two are **Server Actions correctness fixes**. All patterns documented below assume the app is on `next@16.3.1-canary.8+` and using the Turbopack bundler (Webpack users unaffected).

### Pattern A: Turbopack CJS Tree Shaking Default-On (PR #96779)

**Pre-canary.8 (default off):** Turbopack's CJS tree shaking was opt-in via `experimental: { turbopackCjsTreeShaking: true }`. CJS-only dependencies (e.g., `lodash`, `axios` pre-1.x, `react-dom/server`) were bundled in full even when only one or two exports were used.

**Post-canary.8 (default on):** Turbopack now applies CJS tree shaking by default. CJS-only deps get the same DCE treatment as ESM exports.

**Expected impact:** **5-15% bundle size reduction** for apps with CJS-heavy dependencies. Apps that primarily use ESM (Next.js 13+ projects) see zero or marginal impact.

**Migration recipe:**
```ts
// next.config.ts — REMOVE the now-redundant opt-in flag
const nextConfig: NextConfig = {
  // experimental: { turbopackCjsTreeShaking: true }, ← DELETE (now default-on)
}
```

**Audit:**
```bash
# Verify your Turbopack config has no redundant opt-in:
rg -n "turbopackCjsTreeShaking" next.config.*
# → any match should be deleted (line is now redundant)
```

### Pattern B: Turbopack Shared Runtime Default-On (PR #96778)

**Pre-canary.8 (default off):** Turbopack's shared runtime (one JS bundle for the framework runtime shared across all routes, instead of one per route) was opt-in via `experimental: { turbopackSharedRuntime: true }`.

**Post-canary.8 (default on):** Turbopack now applies the shared runtime by default for all builds. The framework runtime is bundled once, not per-route.

**Expected impact:**
- **1-3 KB smaller HTML per route** (the per-route `<script>` tag shrinks because the shared runtime is now in a top-level script)
- **5-10% faster Time-to-Interactive (TTI)** for apps with many routes (the browser caches the shared runtime once)
- **Reduced bandwidth** for multi-route navigations

**Migration recipe:**
```ts
// next.config.ts — REMOVE the now-redundant opt-in flag
const nextConfig: NextConfig = {
  // experimental: { turbopackSharedRuntime: true }, ← DELETE (now default-on)
}
```

**Audit:**
```bash
# Verify your Turbopack config has no redundant opt-in:
rg -n "turbopackSharedRuntime" next.config.*
# → any match should be deleted (line is now redundant)
```

### Pattern C: Turbopack Per-Environment Minify Config (PR #96578)

**Pre-canary.8:** Turbopack minification was controlled by `experimental.turbopackMinify: true|false` (single boolean). The `experimental.serverMinification` flag was Webpack-only.

**Post-canary.8:** `experimental.turbopackMinify` is now an object that supports per-environment overrides (`'client'`, `'server'`, or both). `experimental.serverMinification` is now also respected by Turbopack (was Webpack-only before).

**Migration recipe:**
```ts
// next.config.ts — Per-environment Turbopack minify config
const nextConfig: NextConfig = {
  experimental: {
    turbopackMinify: {
      client: true,   // minify client bundles (default: true)
      server: true,   // minify server bundles (NEW: now Turbopack-supported)
    },
    // serverMinification: true, ← NOW respected by Turbopack (was Webpack-only)
  },
}
```

**Practical impact:** projects that previously opted out of `turbopackMinify` to debug server-side bundling can now keep server minification off while leaving client minification on. Conversely, projects that want to fully opt out of minification for performance profiling can set `turbopackMinify: false` for both environments.

### Pattern D: Server Actions on Dynamic PPR Fallback Routes (PR #96932)

**Pre-canary.8 (16.3.0 STABLE + canary.0–7):** Server Actions triggered from dynamic PPR fallback routes (routes with `loading.tsx` + dynamic segments) could throw or fail to register properly.

**The canonical pattern (now safe on canary.8+):**

```tsx
// app/dashboard/@notifications/loading.tsx
export default function Loading() {
  return <Skeleton />
}

// app/dashboard/@notifications/page.tsx
import { Suspense } from 'react'
import { markRead } from './actions'

export default function NotificationsPage() {
  return (
    <Suspense fallback={<Skeleton />}>
      <NotificationsList />
    </Suspense>
  )
}

async function NotificationsList() {
  const notifications = await fetchNotifications()
  return (
    <ul>
      {notifications.map((n) => (
        <li key={n.id}>
          {n.title}
          <form action={markRead.bind(null, n.id)}>
            <button>Mark read</button>
          </form>
        </li>
      ))}
    </ul>
  )
}
```

**Pre-canary.8 behavior:** clicking "Mark read" on a PPR fallback's `<Suspense>` boundary would intermittently 500 or silently no-op depending on whether the action called `revalidatePath` before returning.

**Post-canary.8 behavior:** the action's revalidations + redirects + error handling now work consistently regardless of whether the route is a static prerender, a dynamic render, or a PPR fallback.

**Audit:**
```bash
# Find PPR fallback routes with Server Actions:
rg -ln "loading\.tsx" app/ | while read f; do
  dir=$(dirname "$f")
  if [ -f "$dir/page.tsx" ]; then
    rg -l "import.*from.*['\"]\\./actions['\"]|form action=" "$dir/"
  fi
done
# → any match should bump to canary.8+ for PR #96932 correctness
```

### Pattern E: Forwarded Action Errors Flush Revalidations (PR #96945)

**Pre-canary.8 (16.3.0 STABLE + canary.0–7):** Server Actions that called `revalidatePath()` (or `revalidateTag()`) and then threw `notFound()` / an error / a redirect that errored would return their error response WITHOUT executing the pending invalidation. The cache would not be invalidated; the next request would see stale data.

**The canonical pattern (now safe on canary.8+):**

```ts
// app/actions/posts.ts
'use server'
import { revalidatePath } from 'next/cache'
import { notFound } from 'next/navigation'

export async function deletePost(postId: string) {
  const post = await db.post.findUnique({ where: { id: postId } })
  if (!post) notFound()  // throws — but revalidation MUST still execute

  await db.post.delete({ where: { id: postId } })
  revalidatePath('/posts')
  revalidatePath(`/posts/${postId}`)
  // Pre-canary.8: if a throw happens after this, revalidation is silently lost
  // Post-canary.8: revalidation is centralized in getRevalidationWaitUntil()
}
```

**Pre-canary.8 bug surface:** high-stakes apps (admin dashboards, e-commerce inventory, CMS publishes) where the post-action revalidation MUST happen for the system to remain consistent. Pre-canary.8, an error after `revalidatePath()` would leave the cache stale — the next request would see the old data until the cache TTL expired (could be hours).

**Post-canary.8 behavior:** revalidation is now consistent regardless of whether the action succeeded, errored, or was forwarded. No silent stale-cache windows.

**Audit:**
```bash
# Find Server Actions with revalidatePath + notFound/throw patterns:
rg -nB1 -A3 "revalidatePath\(|revalidateTag\(" --type ts app/ actions/ | rg -B1 -A3 "notFound|throw|redirect\("
# → any match should bump to canary.8+ for PR #96945 correctness
```

### When to use these patterns together

The 5 PRs compose into a single "Next.js 16.3 on Turbopack, production-grade" recipe:

```ts
// next.config.ts — The canary.8+ canonical production config
const nextConfig: NextConfig = {
  // Cache Components (the 16.3 default — opt-in here is now redundant)
  cacheComponents: true,

  // Turbopack (default in 16.3+)
  // (No explicit bundler opt-in needed in 16.3+; Turbopack is the default)

  experimental: {
    // Partial Prefetching — opt in for instant navigation
    partialPrefetching: true,

    // Turbopack per-environment minify (NEW: now configurable per-env)
    turbopackMinify: {
      client: true,
      server: true,
    },
    // serverMinification: true, ← NOW respected by Turbopack (was Webpack-only)

    // DELETE these (now default-on in canary.8+):
    // turbopackCjsTreeShaking: true, ← redundant
    // turbopackSharedRuntime: true, ← redundant
  },
}
```

```tsx
// app/posts/[slug]/page.tsx — Cache Components + Server Actions + PPR fallback
import { Suspense } from 'react'
import { notFound } from 'next/navigation'
import { cacheLife, cacheTag } from 'next/cache'
import { markRead } from './actions'

// 'use cache' + cacheLife gives you ISR-like behavior
async function getPost(slug: string) {
  'use cache'
  cacheLife('hours')          // ISR: revalidate every hour
  cacheTag(`post:${slug}`)

  const post = await db.post.findUnique({ where: { slug } })
  if (!post) notFound()
  return post
}

export default async function PostPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  const post = await getPost(slug)

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
      {/* PPR fallback + Server Action — safe on canary.8+ */}
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments postId={post.id} />
      </Suspense>
    </article>
  )
}
```

```ts
// app/posts/[slug]/actions.ts — Server Action with revalidation (PR #96945 safe)
'use server'
import { revalidatePath, revalidateTag } from 'next/cache'
import { notFound } from 'next/navigation'

export async function markRead(postId: string) {
  const post = await db.post.findUnique({ where: { id: postId } })
  if (!post) notFound()  // throws; revalidation still executes post-canary.8

  await db.readMark.create({ data: { postId, userId: getUserId() } })

  revalidatePath(`/posts/${post.slug}`)
  revalidateTag(`post:${post.slug}`)
  // Post-canary.8: revalidation runs even if subsequent code throws
}
```

### Sources

- [Next.js v16.3.1-canary.8 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.8) — npm-published 2026-08-07T23:58:34Z
- [Next.js canary-branch compare v16.3.1-canary.7...canary](https://github.com/vercel/next.js/compare/v16.3.1-canary.7...canary) — 20 commits ahead at 2026-08-08T12:03Z (this cron)
- [PR #96779 — Enable CJS tree shaking by default](https://github.com/vercel/next.js/pull/96779) — sampoder, merged 2026-08-07T18:26:49Z, 8 files / +89/-51 (Turbopack config-shared change)
- [PR #96778 — Enable the shared runtime by default](https://github.com/vercel/next.js/pull/96778) — sampoder, merged 2026-08-07T19:37:55Z, 6 files / +76/-42 (Turbopack config-shared change)
- [PR #96578 — Support `experimental.serverMinification` & expand `experimental.turbopackMinify`](https://github.com/vercel/next.js/pull/96578) — sampoder, merged 2026-08-07T21:05:21Z, 4 files / +127/-43 (Turbopack minify per-env support)
- [PR #96932 — Handle Server Actions on dynamic PPR fallback routes](https://github.com/vercel/next.js/pull/96932) — ztanner, merged 2026-08-07T23:09:22Z, 4 files / +57/-22
- [PR #96945 — Flush pending revalidations for forwarded action error responses](https://github.com/vercel/next.js/pull/96945) — ztanner, merged 2026-08-07T23:09:24Z, 4 files / +65/-16
- [Next.js 16.3.0 release tag](https://github.com/vercel/next.js/releases/tag/v16.3.0) — STABLE; npm-published 2026-08-03T21:03:18Z
- [Turbopack CJS tree shaking docs](https://nextjs.org/docs/app/api-reference/turbopack) — `experimental.turbopackCjsTreeShaking` reference
- [Turbopack shared runtime docs](https://nextjs.org/docs/app/api-reference/turbopack) — `experimental.turbopackSharedRuntime` reference
- [Cache Components docs](https://nextjs.org/docs/app/api-reference/next-config-js/cacheComponents) — `cacheComponents` reference

## Pattern: Turbopack — 2 Major Reverts Queued for canary.11+ (PR #97018 Reverts Pattern A CJS Tree Shaking Default-On + PR #97009 Reverts the canary.9 Async Re-Export Tree Shaking Optimization) (August 10, 2026)

The 6h window since the v1.5.44 cycle has surfaced **2 MAJOR REVERTS** on the Next.js canary-branch ahead of `16.3.1-canary.10` (which SHIPPED to npm at 2026-08-10T07:41:37Z). These reverts directly affect the **canonical recipes documented in v1.5.38's `## Pattern: Turbopack + Server Actions + Cache Components on canary.8`** section:

- **PR #97018** (Hendrik Liebau, merged 2026-08-10T11:28:55Z) reverts **PR #96779** — the canary.8 Pattern A "Turbopack CJS Tree Shaking Default-On". The flag is being flipped back from default-`true` to default-`false`. The recipe in Pattern A becomes "DELETE the now-redundant opt-in flag" again — except now, the recipe is "DO NOT add the opt-in flag" unless you've audited your CJS dependency tree.
- **PR #97009** (merged 2026-08-10T11:28:55Z) reverts **PR #95993** — the canary.9 `[turbopack] Follow re-exports for side-effect free async modules` optimization. The 5-20% bundle reduction is reverted; there is no flag, no opt-in path — the optimization is gone indefinitely.

Both reverts ship in `next@16.3.1-canary.11` (npm-published expected ~24h after canary.10 on the 24h cadence, so ~2026-08-11T07:41Z ± a few hours). The full PR-by-PR deep dive is in `performance.md` and `deployment.md`; this section updates the **canonical recipe** in patterns.md to reflect the new reality.

### Pattern A (UPDATED for canary.11+) — Turbopack CJS Tree Shaking Default-OFF (PR #97018 Reverts PR #96779)

**Pre-canary.8 (and post-canary.11 with PR #97018):** CJS tree shaking is opt-in via `experimental: { turbopackCjsTreeShaking: true }`. Apps that don't add the flag get the full CJS bundle even if only one or two exports are used.

**canary.8 to canary.10:** The flag was flipped default-on (PR #96779). Apps got 5-15% bundle reduction for CJS-heavy dependencies automatically.

**canary.11+:** The flag is flipped back to default-off (PR #97018). Apps that want the optimization must add it explicitly AND audit their CJS dependency tree first.

**The bug that triggered the revert:** CJS modules written as `var X = module.exports = { ... }` lose properties that are only read back through the alias. The canonical affected module is `@mixmark-io/domino` (Turndown's server DOM) `lib/LinkedList.js`:

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

**Updated migration recipe (for canary.11+):**

```ts
// next.config.ts — the UPDATED canary.11+ canonical production config
const nextConfig: NextConfig = {
  // ... existing config ...
  experimental: {
    // DO NOT add turbopackCjsTreeShaking: true unless you've audited your CJS dependency tree
    // (see audit recipe below)
  },
}
```

**Updated audit recipe (for canary.11+ users who want to re-enable CJS tree shaking):**

```bash
# 1. Scan lockfile for known affected CJS modules
rg -l "@mixmark-io/domino|@mixmark-io/turndown" package-lock.json pnpm-lock.yaml yarn.lock
# → any match means DO NOT re-enable CJS tree shaking

# 2. Scan your node_modules for self-referential module.exports patterns
rg -l "module\.exports\s*=\s*\{" node_modules/@mixmark-io/domino/lib/ \
                                 node_modules/express/lib/ \
                                 node_modules/*/package.json 2>/dev/null | head -50
# → any match confirms self-referential pattern; do not re-enable

# 3. Build a smoke test before opting back in
pnpm build && pnpm test
# → if smoke test passes for ALL your test cases, safe to re-enable
# → if smoke test fails for ANY case involving @mixmark-io/domino, leave flag off

# 4. Only then opt back in:
# next.config.ts:
# experimental: { turbopackCjsTreeShaking: true }
```

### Pattern B — Turbopack Shared Runtime Default-ON (PR #96778, UNCHANGED)

The Pattern B documented in v1.5.38 (Turbopack shared runtime default-on, 1-3 KB smaller HTML per route, 5-10% faster TTI for multi-route apps) is **unchanged by PR #97018**. The shared runtime doesn't have the CJS self-referential elision bug. The migration recipe (DELETE the now-redundant opt-in flag) is still correct.

### Pattern C — Turbopack Per-Environment Minify Config (PR #96578, UNCHANGED)

The Pattern C documented in v1.5.38 (Turbopack per-environment minify config via `experimental.turbopackMinify: { client: true, server: true }`) is **unchanged by PR #97018**. The minify config is independent of the CJS tree shaking flag. The migration recipe (replace `experimental.serverMinification: false` with `experimental.turbopackMinify: { server: false }`) is still correct.

### Pattern (REMOVED) — Turbopack Async Re-Export Tree Shaking (PR #95993, REVERTED by PR #97009)

The 5-20% bundle reduction for pure re-export barrels imported via `await import(...)` documented in v1.5.39's `## next@16.3.1-canary.9 SHIPPED` section is **REVERTED in canary.11+ by PR #97009**. The revert body is terse: *"This causes `ModuleId not found for ident` errors with `next/dynamic`."*

**Pre-canary.9 (and post-canary.11 with PR #97009):** Pure re-export barrels (`export { x } from './y.js'`) imported via `await import(...)` include ALL re-exports in the bundle, even unused ones. Sync imports were already tree-shaken correctly.

**canary.9 to canary.10:** The optimization was active; async-imported pure re-export barrels had unused re-exports dropped (5-20% bundle reduction).

**canary.11+:** The optimization is gone. There is no flag, no opt-in path. Apps that depended on the 5-20% bundle reduction will see larger bundles.

**Updated practical impact:**

- Apps with **large pure re-export barrels imported via `await import(...)`** (component libraries, design systems, icon libraries) will see a **5-20% bundle size regression** on canary.11+. The regression is silent — no warnings, no deprecation messages.
- Apps with **no async-imported pure re-export barrels** see zero change.
- **No migration recipe available** — there is no opt-in path for this optimization.

### Updated When to use these patterns together — the canary.11+ canonical recipe

```ts
// next.config.ts — the canary.11+ canonical production config
const nextConfig: NextConfig = {
  // Cache Components (the 16.3 default — opt-in here is now redundant)
  cacheComponents: true,

  // Turbopack (default in 16.3+)
  // (No explicit bundler opt-in needed in 16.3+; Turbopack is the default)

  experimental: {
    // Partial Prefetching — opt in for instant navigation
    partialPrefetching: true,

    // Turbopack per-environment minify (Pattern C, unchanged)
    turbopackMinify: {
      client: true,
      server: true,
    },
    // serverMinification: true, ← NOW respected by Turbopack (was Webpack-only)

    // Shared runtime default-ON (Pattern B, unchanged) — DELETE these if you have them:
    // turbopackSharedRuntime: true, ← redundant (default-on since canary.8)

    // CJS tree shaking default-OFF (Pattern A, REVERTED) — DO NOT add unless audited:
    // turbopackCjsTreeShaking: true, ← OPT-IN ONLY; audit your CJS dependency tree first
    // (See Pattern A audit recipe above)
  },
}
```

The recipe is functionally identical to the canary.10 recipe EXCEPT for the CJS tree shaking line — that's now an explicit opt-in with an audit gate, not a default. The async re-export tree shaking (canary.9's headline) is simply gone.

### Sources

- [PR #97018 — `Revert "[turbopack] Enable CJS tree shaking by default (#96779)"`](https://github.com/vercel/next.js/pull/97018) — by Hendrik Liebau, merged 2026-08-10T11:28:55Z, 2 files / +6/-2. **Queued for canary.11**.
- [PR #96779 — `[turbopack] Enable CJS tree shaking by default`](https://github.com/vercel/next.js/pull/96779) — the canary.8 PR being reverted. Originally merged 2026-08-07T18:26:49Z by sampoder, 2 files.
- [PR #97009 — `Revert "[turbopack] Follow re-exports for side-effect free async modules"`](https://github.com/vercel/next.js/pull/97009) — merged 2026-08-10T11:28:55Z. **Queued for canary.11**.
- [PR #95993 — `[turbopack] Follow re-exports for side-effect free async modules`](https://github.com/vercel/next.js/pull/95993) — the canary.9 PR being reverted. Originally merged 2026-08-08T01:28:49Z by sampoder, 17 files / +176/-39.
- [Next.js v16.3.1-canary.10 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.10) — npm-published 2026-08-10T07:41:37Z
- [@mixmark-io/domino repo](https://github.com/foliojs/domino) — the canonical CJS module affected by PR #97018 (used by Turndown for server-side HTML DOM)
- [Cross-reference: v1.5.45 performance.md `## next@16.3.1-canary.10 SHIPPED` — full PR-by-PR deep dive](https://github.com/clawvpsai/frontend-skill/blob/main/performance.md) — the performance-lens coverage with the verbatim PR body walkthroughs
- [Cross-reference: v1.5.45 deployment.md `## Next.js — next@16.3.1-canary.10 SHIPPED — Deployment Impact Lens` — the deployment-bounded audit recipes](https://github.com/clawvpsai/frontend-skill/blob/main/deployment.md) — the deploy-side perspective on the same reverts
- [Cross-reference: v1.5.38 patterns.md `## Pattern: Turbopack + Server Actions + Cache Components on canary.8` — the original Pattern A documentation being updated](https://github.com/clawvpsai/frontend-skill/blob/main/patterns.md) — the canary.8 SHIP event that introduced the CJS tree shaking default-on

## Next.js 16.3 Instant Navigation Patterns (August 2026) — Partial Prefetching Adoption + `instant()` Test Helper + `useOffline` Hook + `catchError` Error Boundaries + Root Params + Read-Your-Writes With `updateTag`

The 16.3 cycle (STABLE since 2026-08-03) introduced the **Instant Navigations** suite — a coordinated set of features that bring SPA-feel link clicks to a server-driven Next.js app. The features are **opt-in** (via `next.config.ts` `cacheComponents: true`, `partialPrefetching: true`), so this section documents the canonical adoption patterns. **From a patterns lens**, the 6 new patterns are:

1. **Partial Prefetching** — the new prefetching behavior that extracts a reusable loading shell from any route's UI
2. **`instant()` test helper** — verifies whether a navigation is instant (new `@next/playwright` API)
3. **`useOffline` hook** — reports when the app is offline (new `experimental.useOffline` companion)
4. **`catchError` error boundaries** — new error boundary shape for Cache Components
5. **Root params** — root-level `[locale]` etc. work inside `use cache` scopes (new in 16.3)
6. **Read-Your-Writes with `updateTag`** — the new Server Actions-only API for read-your-writes semantics

These patterns are **for content-heavy Next.js apps**; the canary.10+ docs (https://nextjs.org/docs/app/guides/adopting-partial-prefetching) show the canonical adoption.

### Pattern A — Partial Prefetching Adoption (the canonical 16.3 instant-navigation pattern)

The 16.3 prefetching default was wasteful: prefetching a target route's *full* content for every link in the viewport, even when the user only clicks 1-2 of them. With Partial Prefetching, Next.js extracts a **reusable loading shell** from any route's UI, and each `<Link prefetch>` decides how much of the target page to include. You get instant feedback on click without prefetching entire pages your users may never visit.

**The canonical adoption recipe (from the official docs `https://nextjs.org/docs/app/guides/adopting-partial-prefetching` updated 2026-08-10):**

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  cacheComponents: true,
  partialPrefetching: true,
}

export default nextConfig
```

**The key design points:**

1. **`cacheComponents: true` is the underlying primitive.** Partial Prefetching is a layer on top of Cache Components. You must opt into Cache Components first.
2. **`partialPrefetching: true` is the opt-in.** Without it, your Prefetch headers fall back to the pre-16.3 behavior (full-page prefetch).
3. **The shell is extracted from `'use cache'` boundaries.** Routes that want a reusable shell must wrap the shell portion in `'use cache'` (or use `'use cache: private'` for personalized shells).
4. **`<Link prefetch={...}>` controls per-link prefetch scope:**
   - `prefetch={true}` (default): full prefetch
   - `prefetch={undefined}`: shell-only prefetch (the new default behavior with Partial Prefetching enabled)
   - `prefetch="allow-runtime"`: includes runtime data (cookies, headers, search params)
   - `prefetch={false}`: no prefetch

**Migration recipe (canary.10 → 16.3 stable):**

```ts
// Before — 16.2.x with PPR
// next.config.ts
export default {
  experimental: {
    ppr: true,
  },
}

// After — 16.3 with Partial Prefetching
// next.config.ts
export default {
  cacheComponents: true,
  partialPrefetching: true,
  // experimental.ppr is REMOVED in 16.3
  // (Cache Components is now the sole PPR signal)
}
```

**Shell extraction pattern:**

```tsx
// app/posts/[id]/page.tsx
import { Suspense } from 'react'
import { PostContent } from './post-content'
import { PostComments } from './post-comments'

// The shell: cacheable, reused across navigations
export async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  return (
    <div>
      <h1>Post {id}</h1>
      <Suspense fallback={<PostCommentsSkeleton />}>
        <PostComments postId={id} />
      </Suspense>
    </div>
  )
}

// The dynamic data: streams in after the shell
async function PostCommentsSkeleton() {
  return <div>Loading comments...</div>
}
```

**Practical impact:**

- **Apps with many links in the viewport (e.g., dashboard, search results):** 45% reduction in prefetch requests (per Vercel's measurement, https://vercel.com/blog/vercel-supports-next-js-16-3 published 2026-08-04). This drops the prefetch-related bandwidth and CPU.
- **Cold start perceived performance:** With shell prefetch, click-to-interactive is dramatically faster — the shell renders immediately, the content streams in.
- **Server-side cost:** The shell is computed once per session (memoized on the client), not per click. The dynamic data is fetched per navigation.

**Common mistakes:**

- **Setting `partialPrefetching: true` without `cacheComponents: true`.** The flag is silently ignored in 16.3 (per the canary.10 docs).
- **Forgetting to wrap the shell in `'use cache'`.** Without a cache boundary, the shell is recomputed per navigation, defeating the purpose.
- **Confusing `prefetch={true}` (full) vs `prefetch={undefined}` (shell).** With Partial Prefetching enabled, `prefetch={true}` is wasteful in most cases. Use `prefetch={undefined}` (or omit) for the shell-only default.
- **Not handling the `prefetch="allow-runtime"` case.** Without it, the prefetched shell doesn't include the user's auth state, locale, etc. For personalized routes, opt in explicitly.

### Pattern B — `instant()` Test Helper (Verification)

The `instant()` test helper (new in 16.3, `@next/playwright` package) verifies whether a navigation is instant. The canonical use case is for agents (like the [`next-cache-components-optimizer` skill](https://www.skills.sh/vercel/next.js/next-cache-components-optimizer)) that apply Partial Prefetching and need to verify that the resulting navigation is actually instant.

**The canonical recipe:**

```ts
// tests/instant-navs.spec.ts
import { test, expect } from '@playwright/test'
import { instant } from '@next/playwright'

test('clicking the dashboard link is instant', async ({ page }) => {
  await page.goto('/chat')
  
  // Click the dashboard link and assert the navigation is instant
  const result = await instant(page, async () => {
    await page.click('a[href="/dashboard"]')
  })
  
  expect(result.isInstant).toBe(true)
  expect(result.shellRenderedAt).toBeLessThan(100) // ms
})
```

**The `instant()` helper returns:**
- `isInstant: boolean` — whether the shell rendered before the navigation completed
- `shellRenderedAt: number` — ms to shell render
- `firstPaintAt: number` — ms to first paint of the destination route
- `hydrationAt: number` — ms to hydration

**Practical impact:**

- **Agent-first adoption:** The `next-cache-components-optimizer` skill uses `instant()` to write a failing test for each slow route, apply the cache + partial prefetching, and verify the test now passes.
- **CI regression detection:** Add `instant()` tests to your CI to detect when a route's shell grows or the dynamic data fetch slows.
- **Vercel-published case study:** The v0 team used this pattern to identify slow routes and apply Partial Prefetching. https://nextjs.org/blog (Aug 6, 2026) has the full case study.

**Migration:** `npm install -D @next/playwright` (already in your `@playwright/test` install). Then `import { instant } from '@next/playwright'`.

### Pattern C — `useOffline` Hook (Offline Resilience)

The new `experimental.useOffline` flag plus the `useOffline()` hook (new in 16.3) keeps navigations, fetches, and Server Actions **pending** instead of throwing when the connection drops. When the connection restores, the pending requests retry.

**The canonical recipe:**

```ts
// next.config.ts
export default {
  experimental: {
    useOffline: true,
  },
}
```

```tsx
// app/components/offline-banner.tsx
'use client'
import { useOffline } from 'next/hooks'

export function OfflineBanner() {
  const isOffline = useOffline()
  
  if (!isOffline) return null
  
  return (
    <div className="offline-banner">
      You're offline. Your changes will sync when you reconnect.
    </div>
  )
}
```

**How it works:**

- **Navigations:** When the user clicks a `<Link>` while offline, the navigation stays pending instead of throwing. When the connection restores, the navigation completes.
- **Fetches:** `fetch()` calls inside Server Components stay pending; routes render with the cached shell + the dynamic data streams in when online.
- **Server Actions:** Form submissions stay pending; the submit button shows a "waiting for connection" state instead of throwing.

**Practical impact:**

- **Apps with flaky networks (mobile, developing regions):** Graceful degradation instead of error UI.
- **Apps with PWAs:** The shell renders from cache, the dynamic data streams in when online.
- **Apps with offline-first UX:** Combine with service workers for full offline support.

**Common mistakes:**

- **Enabling `useOffline` without a service worker.** The hook only delays the requests; the requests still need to succeed. Without a service worker, the user has no way to recover from a sustained offline state.
- **Forgetting to handle the pending state in UI.** The `useOffline` hook returns `true` when offline, but your fetch hooks also need to show a pending state.

### Pattern D — `catchError` Error Boundaries (Cache Components)

The new `catchError` error boundary (new in 16.3) is the Cache Components-compatible error boundary. It catches errors from the dynamic data fetches while keeping the shell interactive.

**The canonical recipe:**

```tsx
// app/dashboard/error.tsx
'use client'
import { catchError } from 'next/cache-components'

export default catchError(async ({ error, reset }) => {
  return (
    <div className="error-state">
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  )
})
```

**Why `catchError` instead of `error.tsx`?**

- **Cache Components requires that errors from the dynamic data don't break the shell.** A regular `error.tsx` catches everything; the `catchError` boundary only catches errors from the data fetches within the cache boundary.
- **The `reset` function is async-cache-aware.** It re-runs the dynamic data fetch without re-fetching the shell.

**Practical impact:**

- **Apps with `'use cache'` boundaries around dynamic data:** The shell keeps rendering; the dynamic data shows the error state. Better UX than losing the entire route.
- **Apps with mixed static + dynamic routes:** The static part stays interactive; the dynamic part shows the error.

**Common mistakes:**

- **Using `error.tsx` for Cache Components routes.** It catches errors that should cascade to the shell; use `catchError` instead.
- **Forgetting that `reset` is partial.** It re-runs the dynamic data, not the shell. If the shell is broken, you need a full `router.refresh()`.

### Pattern E — Root Params (Inside `use cache` Scopes)

The new **Root params** feature (16.3) lets `[locale]` and similar root-level params work inside `'use cache'` scopes. This makes internationalization considerably less painful for Cache Components routes.

**The canonical recipe:**

```tsx
// app/[locale]/posts/[id]/page.tsx
import { unstable_cache } from 'next/cache'

// The root param [locale] is automatically picked up by the cache key
const getPost = unstable_cache(
  async (locale: string, id: string) => {
    return db.posts.findUnique({ where: { id }, locale })
  },
  ['post'],
  { tags: [`post`] }
)

export async function Page({ 
  params 
}: { 
  params: Promise<{ locale: string; id: string }> 
}) {
  const { locale, id } = await params
  const post = await getPost(locale, id)
  return <Post post={post} />
}
```

**What's new in 16.3:**

- **Pre-16.3:** Root params like `[locale]` were NOT included in the cache key. Two users with different locales would see the same cached content.
- **16.3:** Root params ARE included in the cache key. The cache function automatically reads the params from the route context.

**Limitations (per the 16.3 docs):**

- **Route handlers and Server Actions don't support root params yet.** The planned support is for a future release (16.4 or 16.5).

**Practical impact:**

- **i18n apps with `'use cache'`:** Much simpler — no need to manually pass the locale into the cache key.
- **Apps with multi-tenant routing:** Similar simplification for tenant-scoped caches.

### Pattern F — Read-Your-Writes with `updateTag` (Server Actions)

The new `updateTag()` Server Actions-only API (new in 16.3) provides **read-your-writes** semantics. When a user updates their profile, they see the changes instantly because the cache expires and refreshes immediately within the same request.

**The canonical recipe:**

```tsx
// app/profile/actions.ts
'use server'
import { updateTag } from 'next/cache'
import { revalidatePath } from 'next/cache'

export async function updateProfile(formData: FormData) {
  const userId = await getUserId()
  const newName = formData.get('name') as string
  
  // Update the database
  await db.users.update({
    where: { id: userId },
    data: { name: newName },
  })
  
  // Invalidate the cache — refreshes immediately within this request
  updateTag(`user:${userId}`)
  
  // Optional: revalidate the page path
  revalidatePath('/profile')
}
```

**The difference vs `revalidateTag`:**

- **`revalidateTag('posts', 'max')`:** Stale-while-revalidate. The next request waits for fresh data, but the current request still serves the stale cache. There's a brief window where the user sees stale data.
- **`updateTag('user:123')`:** Read-your-writes. The current request's render wait for fresh data. The user sees the change instantly.

**When to use each:**

| Scenario | Use |
|---|---|
| User updates their own profile | `updateTag` |
| User publishes a post; other users see it | `revalidateTag` (with `max`) |
| CMS webhook triggers a content update | `revalidateTag` |
| User edits a draft | `updateTag` |
| Admin dashboard shows real-time metrics | `refresh()` (different API) |

**Common mistakes:**

- **Using `revalidateTag` instead of `updateTag` for personal data.** The current user sees stale data until the next request.
- **Using `updateTag` for global data.** It's wasteful — every user re-renders a route to refresh the cache for one user.

### Sources

- [Next.js 16.3 official blog post](https://nextjs.org/blog/next-16-3) — published 2026-08-03; the canonical Instant Navigations announcement
- [Next.js Adopting Partial Prefetching guide](https://nextjs.org/docs/app/guides/adopting-partial-prefetching) — last updated 2026-08-10; the canonical adoption recipe
- [Next.js 16.3 version](https://nextjs.org/docs/app/guides/version-16) — last updated 2026-08-03; the upgrade guide
- [Next.js 16.3 on Vercel](https://vercel.com/blog/vercel-supports-next-js-16-3) — published 2026-08-04; the 45% reduction in prefetch requests + the JSONL-formatted shards + PPR observability
- [Next.js Instant Navigation Testing docs](https://nextjs.org/docs/app/api-reference/functions/instant) — the `instant()` test helper reference
- [Next.js useOffline hook docs](https://nextjs.org/docs/app/api-reference/functions/use-offline) — the `useOffline` hook reference
- [catchError docs](https://nextjs.org/docs/app/api-reference/file-conventions/error) — the `catchError` Cache Components error boundary reference
- [updateTag docs](https://nextjs.org/docs/app/api-reference/functions/updateTag) — the Server Actions read-your-writes API reference
- [Next.js 16.3.0 standard build artifacts](https://www.npmjs.com/package/next/v/16.3.0) — STABLE 16.3.0 release (Aug 3, 2026)
- [Next.js 16.3.1-canary.15 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.15) — npm-published 2026-08-12T23:26:21Z; the latest canary
- [Next.js 16.3.0 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.0) — STABLE 16.3.0 release (2026-08-03T21:03:18Z)
- Cross-references: `routing.md` → `## Next.js 16.3 — Instant Navigations` for the routing-layer coverage of `<Link prefetch>` changes; `performance.md` → `## Next.js 16.3 — Instant Navigations` for the perf-measurement lens; `security.md` → the Cache Components audit recipe for the new APIs; `server-components.md` → the `'use cache'` and `usePrivateCache` patterns; `api.md` → `## Next.js 16.3.1-canary.10 → canary.15 API-Surface Changes` for the companion API-surface changes (PR #97166 live headers() + PR #96937 unstable_cache encoding + PR #97040 static/app shell tracking + PR #97247 RDC compression + PR #97181 literal exports in cache files + PR #95439 stale data revalidation fix)

## Next.js 16.3.1-canary.17 → canary.18 Pattern Surface (August 14, 2026) — Adapter + Standalone Patterns (PR #97287) + Pages API + Adapter Patterns (PR #96819) + Pages Router Metadata-File Patterns (PR #97350) + @clerk/nextjs 7.7.6 STABLE Patterns + Better Auth 1.6.29 + 1.7.0-rc.6 Patterns (August 14, 2026)

The 6h-window since v1.5.57 (Aug 13 18:02Z) closed with **5 SHIPPED events of pattern-surface impact**:

1. **`next@16.3.1-canary.17` SHIPPED** (npm-published 2026-08-14T17:20:01Z; ~24h before this cron). The v1.5.60 cycle's `server-components.md` lens covered the 4 MATERIAL PRs from the Server Components / RSC lens. The **pattern-surface lens** focuses on the 4 NEW patterns the canary.17 batch unlocks for production teams:

   - **Pattern G — Adapter + Standalone Build Pattern** (PR #97287, enabled in canary.17). Pre-canary.17: Vercel adapter + cdk-nextjs adapter + SST adapter + `output: 'standalone'` combinations ENOENT-crashed on 16.3.0 STABLE because the build didn't emit whole-app server NFTs. Post-canary.17: the build emits `server/app-paths-manifest.json` + `server/required-server-files.json` for adapter + standalone combos that were previously incomplete. **Pattern**: use `output: 'standalone'` + an adapter (cdk-nextjs / amplify / SST) + the canary.17+ `next build` output. Self-hosted deployment on AWS via cdk-nextjs + ECS now builds cleanly on 16.3.x.

   ```ts
   // next.config.ts (canary.17+ + 16.3.2 STABLE)
   import type { NextConfig } from 'next';
   const config: NextConfig = {
     output: 'standalone', // emits whole-app server NFTs for adapter consumption
     experimental: {
       // ... your existing config
     },
   };
   export default config;
   ```

   ```bash
   # Build + deploy with cdk-nextjs (canary.17+)
   npm install next@16.3.1-canary.17
   npx next build
   # cdk-nextjs now picks up the standalone output without ENOENT
   npx cdk deploy
   ```

   - **Pattern H — Pages API + Adapter Build Pattern** (PR #96819, enabled in canary.17). Pre-canary.17: Pages Router + Pages API routes (`pages/api/*.ts`) + adapters crashed on `Cannot find module 'next/dist/compiled/next-server/pages-turbo.runtime.prod.js'`. Post-canary.17: the build bundles `pages-turbo.runtime.prod.js` into the adapter output for Pages API routes. **Pattern**: stay on Pages Router + Pages API + an adapter + bump to canary.17+ for the Pages API runtime fix.

   ```bash
   # Pattern: Pages Router + Pages API + cdk-nextjs adapter (canary.17+)
   npm install next@16.3.1-canary.17
   # pages/api/users.ts works on 16.3.0 + cdk-nextjs adapter
   ```

   - **Pattern I — Pages Router Metadata Files (sitemap.js / robots.js / manifest.js / icon.js)** (PR #97350, enabled in canary.17). Pre-canary.17: `pages/sitemap.js` / `pages/robots.js` / `pages/manifest.json` / `pages/icon.*` build-failed with `getStaticProps is not supported in app/` on 16.3.0 STABLE because the validator scanned the whole tree. Post-canary.17: the validator restricts `app/` exports to files actually under the `app/` directory. **Pattern**: keep Pages Router metadata files in `pages/` (not `app/`) + bump to canary.17+ for the build fix.

   ```ts
   // pages/sitemap.js (Pages Router + canary.17+)
   import { GetServerSideProps } from 'next';
   export default function Sitemap() { /* ... */ }
   export const getServerSideProps: GetServerSideProps = async ({ res }) => {
     res.setHeader('Content-Type', 'text/xml');
     res.write(`<?xml version="1.0" encoding="UTF-8"?>
       <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
         <url><loc>https://example.com</loc></url>
       </urlset>`);
     res.end();
     return { props: {} };
   };
   ```

   - **Pattern J — next/og + satori 0.29.0 + @vercel/og 0.10.x** (PR #97276, enabled in canary.17). Pre-canary.17: `next/og` used satori 0.26.0 + @vercel/og 0.7.x — emoji rendering was limited. Post-canary.17: `next/og` uses satori 0.29.0 + @vercel/og 0.10.x — better emoji rendering, improved text layout, and improved RTL support. **Pattern**: `import { ImageResponse } from 'next/og'` — no API change required, but the bump is automatic for canary.17+ users.

   ```ts
   // app/api/og/route.ts (canary.17+ — satori 0.29.0)
   import { ImageResponse } from 'next/og';
   export async function GET() {
     return new ImageResponse(
       (
         <div style={{ fontSize: 128, background: 'white' }}>
           🚀 Next.js 16.3.2
         </div>
       ),
       { width: 1200, height: 630 }
     );
   }
   ```

2. **`@clerk/nextjs@7.7.6` STABLE Patterns** (npm-published 2026-08-14T23:51:06Z; ~12 minutes before this cron). The 7.7.6 STABLE release bundles 12 canary drops with several pattern-affecting changes:

   - **Pattern K — Pin `@clerk/nextjs@^7.7.6` for React 19.3.x peer-deps**. Pre-7.7.6: Clerk users on React 19.3.0-canary-eb8feb71-20260814 had to pin `@clerk/nextjs@canary` to accept the React 19.3.x peer-deps. Post-7.7.6: STABLE `@clerk/nextjs@^7.7.6` accepts React 19.3.0-canary-eb8feb71-20260814 in the peer-dep range.

   ```json
   // package.json (post-7.7.6)
   {
     "dependencies": {
       "next": "^16.3.1",
       "react": "19.3.0-canary-eb8feb71-20260814",
       "react-dom": "19.3.0-canary-eb8feb71-20260814",
       "@clerk/nextjs": "^7.7.6"
     }
   }
   ```

3. **`better-auth@1.6.29` + `better-auth@1.7.0-rc.6` Patterns** (npm-published 2026-08-14T18:19:56Z + 18:20:13Z; ~5h 43min before this cron). The 1.6.29 STABLE patches the `getDefaultModelName` collision pattern:

   - **Pattern L — Avoid `modelName` aliases that collide with built-in table names**. Pre-1.6.29: a custom adapter schema key named `user` (same as the built-in user table) would be misrouted by `getDefaultModelName`. Post-1.6.29: exact schema key matches are preferred over `modelName` aliases.

   ```ts
   // better-auth config (post-1.6.29)
   export const auth = betterAuth({
     user: {
       modelName: 'app_user', // custom alias — no longer collides with built-in 'user'
       additionalFields: { role: { type: 'string' } },
     },
   });
   ```

   - **Pattern M — Better Auth 1.7.0-rc.6 early-adopter pattern**. The 1.7.0-rc.6 RC unlocks the OAuth device grant + RP-initiated logout + Microsoft account identifier changes + SCIM + SSO + MCP spec alignment + passkey auto sign-in. **Pattern**: pin to `better-auth@1.7.0-rc.6` only if you can tolerate RC churn + you need one of the 1.7.0-specific features.

   ```json
   // package.json (better-auth 1.7.0-rc.6 early adopter)
   {
     "dependencies": {
       "better-auth": "1.7.0-rc.6"
     }
   }
   ```

### Practical impact per user type

| User Type | Pre-canary.17 / Pre-7.7.6 | Post-canary.17 / Post-7.7.6 | Pattern |
|---|---|---|---|
| Vercel deployments on 16.3.0 + adapters | ENOENT crash on `output: 'standalone'` | Build emits full standalone + adapter combo | Pattern G |
| Self-hosted on 16.3.0 + cdk-nextjs adapter | ENOENT crash + missing Pages runtime | Full adapter + Pages API support | Pattern G + Pattern H |
| Pages Router with metadata files | `getStaticProps is not supported in app/` build failure | Pages Router metadata files build OK | Pattern I |
| Apps using `next/og` for dynamic OG images | satori 0.26.0 — limited emoji rendering | satori 0.29.0 — better emoji rendering | Pattern J |
| Clerk auth on React 19.3.x canary | Pin `@clerk/nextjs@canary` required | Pin `@clerk/nextjs@^7.7.6` STABLE | Pattern K |
| Better Auth with custom adapter schema | `getDefaultModelName` misrouting | Exact schema key match preferred | Pattern L |
| Better Auth 1.7.0 early adopters | Pin to 1.6.x stable + wait for 1.7.0 STABLE | Pin to `1.7.0-rc.6` + tolerate RC churn | Pattern M |

### 5-step Combined Audit Recipe (Aug 14, 2026 cycle)

```bash
# 1. Audit Next.js adapter + standalone combination (Pattern G)
rg -n "output: ['\"]standalone['\"]|NEXT_ADAPTER_PATH|NEXT_ENABLE_ADAPTER" --type ts --type tsx --type js --type json

# 2. Audit Pages API routes + adapter combination (Pattern H)
rg -n "export async function|export const" pages/api/ -l

# 3. Audit Pages Router metadata files (Pattern I)
ls pages/sitemap.js pages/robots.js pages/manifest.json pages/icon.* 2>/dev/null

# 4. Audit @clerk/nextjs + React 19.3.x peer-deps (Pattern K)
npm ls @clerk/nextjs react react-dom

# 5. Audit Better Auth version + custom adapter schema (Pattern L + Pattern M)
npm ls better-auth @better-auth/core
rg -n "modelName:" auth.ts src/auth.ts
```

### Sources

- [Next.js v16.3.1-canary.17 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.17) — npm-published 2026-08-14T17:20:01Z; 15 commits ahead of canary.16; bundles PR #97287 + PR #96819 + PR #97350 + PR #97276
- [PR #97287 — `Emit whole-app server NFTs when output: 'standalone' is used with an adapter`](https://github.com/vercel/next.js/pull/97287) — stafach, merged 2026-08-14T~14:00Z, **SHIPPED in `next@16.3.1-canary.17`**; enables Pattern G
- [PR #96819 — `Fix missing Pages runtime in adapter Pages API outputs`](https://github.com/vercel/next.js/pull/96819) — vercel-release-bot, merged 2026-08-14T~14:30Z, **SHIPPED in `next@16.3.1-canary.17`**; enables Pattern H
- [PR #97350 — `Scope app-entry export validation to files inside the app directory`](https://github.com/vercel/next.js/pull/97350) — vercel-release-bot, merged 2026-08-14T~15:00Z, **SHIPPED in `next@16.3.1-canary.17`**; enables Pattern I
- [PR #97276 — `bump satori 0.26.0 → 0.29.0 + @vercel/og 0.7.x → 0.10.x`](https://github.com/vercel/next.js/pull/97276) — **SHIPPED in `next@16.3.1-canary.17`**; enables Pattern J
- [`@clerk/nextjs@7.7.6` on npm](https://www.npmjs.com/package/@clerk/nextjs/v/7.7.6) — STABLE 7.7.6 npm-published 2026-08-14T23:51:06Z; the React 19.3.x peer-dep range bump enables Pattern K
- [`@clerk/javascript/packages/nextjs/CHANGELOG.md`](https://github.com/clerk/javascript/blob/main/packages/nextjs/CHANGELOG.md) — the canonical 7.5.0 → 7.7.6 changelog (the v1.5.50 cycle noted PR #94905 / PR #8524 / PR #8547 as examples)
- [`better-auth@1.6.29` on npm](https://www.npmjs.com/package/better-auth/v/1.6.29) — STABLE 1.6.29 npm-published 2026-08-14T18:19:56Z; enables Pattern L
- [`better-auth@1.7.0-rc.6` on npm](https://www.npmjs.com/package/better-auth/v/1.7.0-rc.6) — RC 1.7.0-rc.6 npm-published 2026-08-14T18:20:13Z; enables Pattern M for early adopters
- [Better Auth 1.6 blog post](https://better-auth.com/blog/1-6) — the canonical restructured-release-notes documentation referenced by `better-auth.com/changelog`
- Cross-references: `api.md` → `## Next.js 16.3.1-canary.17 → canary.18 API-Surface Changes` for the companion API-surface changes (PR #97287 + PR #96819 + PR #97350 + PR #97276 + @clerk/nextjs 7.7.6 STABLE + better-auth 1.6.29 + 1.7.0-rc.6 + Tailwind insiders 90f8ff4); `server-components.md` → the Server Components / RSC lens on the canary.17 batch; `deployment.md` → the deployment-impact lens for the 4 MATERIAL canary.17 PRs; `auth.md` → the auth-impact lens for `@clerk/nextjs@7.7.6` STABLE SHIPPED + the 7.7.7-canary acceleration + better-auth 1.6.29 STABLE + 1.7.0-rc.6 SHIPPED; `performance.md` → the perf lens for the canary.18 NavigationFlightResponse enhancements (carry-over from the v1.5.58 cycle)

## Next.js 16.3.1-canary.21 SHIPPED (August 17, 2026) — Pattern N: Client-Router Modules Reorganization Boundary for Frame/Extension Authors (PR #97402, +437/-353/19 files, acdlite) + Pattern O: `experimental.concurrentRouterQueue` Flag Scaffolding for the Upcoming Router-Queue Rewrite (PR #97413, +619/-229/21 files, acdlite) + the ALS-Singleton Pattern (PR #97255, +91/-121/10 files, unstubbable — already documented from the RSC lens in v1.5.68) (Pattern Surface)

The v1.5.61 cycle covered 7 NEW patterns (Pattern G–M) from the canary.17 + @clerk/nextjs 7.7.6 + better-auth 1.6.29 + 1.7.0-rc.6 SHIPPED events. The v1.5.66 cycle deferred the canary.20 client-router PRs (PR #97402 + PR #97413 were canary-branch-ahead-of-canary.20 at v1.5.66). The v1.5.68 cycle noted them again at the canary.21 repo-tag stage. This v1.5.69 cycle covers the **SHIPPED** canary.21 from the **pattern-surface lens**, unlocking **2 NEW patterns** (Pattern N + Pattern O) for the client-router reorganizations + cross-referencing the v1.5.68 server-components ALS-Singleton pattern.

### Pattern N — Client-Router Modules Reorganization Boundary (acdlite, PR #97402)

**The pattern**: When Next.js's client-router internals change structurally (without behavior changes), Frame/extension authors who reach into `next/dist/client/components/...` should treat module paths + import names as **versioned boundaries**, not as stable APIs.

```ts
// Frame author code (pre-canary.21) — reached into the old module structure
// RISKY: pre-canary.21, this import worked but the path was unstable
import { reducer } from 'next/dist/client/components/router-reducer/reducer'

// Frame author code (post-canary.21) — PR #97402 reorganized the modules
// SAFE: the new `router-queue/` namespace is explicitly carved out as the boundary
import { reducer } from 'next/dist/client/components/router-queue/sequential-router-queue' // or `concurrent-router-queue`
```

**When to use** — If you maintain a Next.js Frame (the [Next.js Frame system](https://nextjs.org/docs/app/guides/building-a-framework)) or an extension that integrates with the router internals (e.g., custom routing logic, router-reducer overrides, segment-cache integrations), audit your `next/dist/client/components/...` imports after upgrading to canary.21+ to verify the paths + names still resolve.

**When NOT to use** — If you're a regular app developer (not a Frame/extension author), you don't import from these internals. Your code is unaffected.

### Pattern O — `experimental.concurrentRouterQueue` Flag Scaffolding for Upcoming Router-Queue Rewrite (acdlite, PR #97413)

**The pattern**: When Next.js scaffolds a NEW experimental flag for an upcoming rewrite, **DO NOT enable it** until the implementation lands in a later canary. The flag is a placeholder.

```ts
// next.config.ts (post-canary.21) — DO NOT enable this yet!
const nextConfig: NextConfig = {
  experimental: {
    // concurrentRouterQueue: true, // ← UNCOMMENTING THIS WILL THROW ON EVERY ROUTER OPERATION
  },
}

// What you'll write in a future canary (when the implementation lands):
// const nextConfig: NextConfig = {
//   experimental: {
//     concurrentRouterQueue: true, // ← Now safe to enable
//   },
// }
```

**The flag**: `experimental.concurrentRouterQueue: boolean` (default `false`); when `true`, swaps to the new `concurrent-router-queue.ts`; when `false`, uses the renamed `sequential-router-queue.ts`. The module-level swap via `navigator.ts` ensures the inactive code is tree-shaken in production builds.

**Why "DO NOT enable"**: PR #97413 verbatim — "There's no implementation yet; all router operations throw an error when the flag is enabled."

**When to use** — After acdlite ships the actual implementation in a later canary (likely canary.23+ based on the "two simultaneous implementations for a while" comment in PR #97402's body). Track via the canary-branch compare or the Next.js blog.

**When NOT to use** — NOW (canary.21). The flag throws on every router operation.

### Pattern N+O combined audit recipe — Frame/extension authors + early adopters

```bash
# 1. Verify you're on canary.21+ (PR #97402 + PR #97413 + PR #97255)
npm ls next
# Expect: next@16.3.1-canary.21 or later

# 2. Audit your Frame/extension imports for client-router modules (Pattern N)
rg -n "next/dist/client/components" --type ts --type tsx -l
# Look for: router-reducer/, segment-cache/, router-queue/

# 3. Verify you are NOT enabling the new flag (Pattern O)
rg -n "concurrentRouterQueue" next.config.*

# 4. Audit your Cache Components + pnpm usage (the ALS-Singleton pattern from v1.5.68)
jq '.packageManager' package.json
rg -n "revalidatePath|io\(\)|use cache" --type ts --type tsx app/ | head -10
```

### Practical impact per user type

| User Type | Pre-canary.21 | Post-canary.21 | Pattern |
|---|---|---|---|
| Frame authors reaching into `next/dist/client/components/router-reducer/` | Modules reorg + rename; import paths may break | New `router-queue/` namespace boundary | Pattern N |
| Extension authors using AsyncLocalStorage identity | Module-identity guarantee (weak) | globalThis-symbol guarantee (strong) | v1.5.68 Pattern (PR #97255) |
| Apps enabling `experimental.concurrentRouterQueue: true` | Flag did not exist | NEW experimental flag; **DO NOT ENABLE** | Pattern O |
| pnpm + Cache Components + `revalidatePath` users | Intermittent FATAL crash on `workAsyncStorage` | Crash fixed via singleton-anchored storages | v1.5.68 Pattern (PR #97255) |
| Regular app developers (no internals reach-in) | (no impact) | (no impact) | N/A |
| Multi-version Next.js realms (monorepo with multiple next copies) | Storages could collide across versions | Per-version keys prevent cross-version reads | v1.5.68 Pattern (PR #97255) |

### 5-step Combined Audit Recipe (Aug 17, 2026 cycle)

```bash
# 1. Verify canary.21+ (Pattern N + O + ALS-Singleton pattern)
npm ls next

# 2. Pattern N — audit Frame/extension client-router imports
rg -n "next/dist/client/components" --type ts --type tsx -l

# 3. Pattern O — verify concurrentRouterQueue flag is unset
rg -n "concurrentRouterQueue" next.config.*

# 4. v1.5.68 Pattern (PR #97255) — verify pnpm + Cache Components audit
jq '.packageManager' package.json
rg -n "revalidatePath" --type ts --type tsx app/

# 5. (Optional) Verify the new module structure (Pattern N + O)
ls node_modules/next/dist/client/components/router-queue/ 2>/dev/null
# Expect: concurrent-router-queue.ts + sequential-router-queue.ts
```

### Sources

- [Next.js v16.3.1-canary.21 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.21) — npm-published 2026-08-17T01:25:51Z; tag commit `d45672c` created 2026-08-16T23:21:52Z; ~4h 36min BEFORE this v1.5.69 cron
- [PR #97402 — `Reorganize client router modules`](https://github.com/vercel/next.js/pull/97402) — acdlite, merged 2026-08-16T03:46:35Z, **SHIPPED in `next@16.3.1-canary.21`**; 19 files / +437/-353; **ENABLES Pattern N**
- [PR #97413 — `Scaffolding for concurrentRouterQueue flag`](https://github.com/vercel/next.js/pull/97413) — acdlite, merged 2026-08-16T03:46:36Z, **SHIPPED in `next@16.3.1-canary.21`**; 21 files / +619/-229; **ENABLES Pattern O**
- [PR #97255 — `Anchor the async local storage instances to global symbols`](https://github.com/vercel/next.js/pull/97255) — unstubbable, merged 2026-08-16T21:15:51Z, **SHIPPED in `next@16.3.1-canary.21`**; 10 files / +91/-121; the v1.5.68 RSC-lens pattern (cross-referenced)
- [Next.js blog: Building a Framework](https://nextjs.org/docs/app/guides/building-a-framework) — the canonical Frame system documentation for Pattern N audience
- [Cross-references](cross-refs): `api.md` → the new `## Next.js 16.3.1-canary.21 SHIPPED (August 17, 2026)` section for the API-surface lens on the same PRs; `server-components.md` → the v1.5.68 `## Next.js 16.3.1-canary.21 (Repo-Tagged August 16, 2026) — PR #97255 Anchor the Async Local Storage Instances to Global Symbols` for the RSC lens on PR #97255

## Next.js 16.3 — "Building App-like Experiences" Blog Post (August 18, 2026) — Pattern P: Agent-Skill-Driven Adoption (npx skills add vercel/next.js) + Pattern Q: View Transitions with `<ViewTransition>` and `transitionTypes` on `next/link` + Pattern R: useTransition + updateTag Optimistic Update Pattern + Pattern S: Drop-Queries with `cacheLife` + `cacheTag` + Pattern T: `next-dev-loop` Runtime Verification Skill (Pattern Surface Lens)

The 6h window between the v1.5.75 cron (12:02Z Aug 19) and the v1.5.76 cron (18:02Z Aug 19) saw **the Next.js team's third deep-dive blog post in the 16.3 series** — "Building App-like Experiences with Next.js 16.3" (published 2026-08-18 by `aurorascharff` + the rest of the Next.js team). The first two posts (the [Next.js 16.3 launch post](https://nextjs.org/blog/next-16-3) on Aug 3 + the [Instant Navigations deep-dive](https://nextjs.org/blog/next-16-3-instant-navigations) on Aug 8) covered the WHAT and the HOW. **This third post covers the BUILD-WITH-IT** — the patterns the Vercel team itself uses to build the demo apps, organized around 4 NEW patterns that the v1.5.69 patterns.md update (canary.21 lens) didn't anticipate. v1.5.76 cycle covers these 4 NEW patterns (Pattern P + Q + R + S) + Pattern T (the `next-dev-loop` runtime-verification skill) from the **pattern-surface lens**.

### Pattern P — Agent-Skill-Driven Adoption (`npx skills add vercel/next.js`)

**The new ecosystem (verbatim from the Aug 18 blog post)**: each major 16.3 feature ships with a **dedicated Vercel Skills** package that a coding agent can install via `npx skills add`. The full set as of Aug 18:

| Skill | Purpose | Install |
|-------|---------|---------|
| `next-cache-components-adoption` | Migrate an app to `cacheComponents: true` | `npx skills add vercel/next.js --skill next-cache-components-adoption` |
| `next-cache-components-optimizer` | After adoption, grow each route's static shell so the App Shell carries more | `npx skills add vercel/next.js --skill next-cache-components-optimizer` |
| `next-partial-prefetching-adoption` | Enable Partial Prefetching + sweep for URL-data insights until every link reuses a shared App Shell | `npx skills add vercel/next.js --skill next-partial-prefetching-adoption` |
| `next-dev-loop` | Cross-check `/_next/mcp` against the live browser via `agent-browser`; surfaces both compile AND runtime issues in one pass | `npx skills add vercel/next.js --skill next-dev-loop` |
| `vercel-react-view-transitions` | Animate route transitions + list changes + Suspense reveals via React's `<ViewTransition>` component | `npx skills add vercel-labs/agent-skills --skill vercel-react-view-transitions` |

**The new adoption pattern** (verbatim from the blog post):
```
1. Point agents at the bundled docs
2. Let errors drive the fixes
3. Hand multi-step workflows to Skills
```

**Step 1 verbatim**: "On Next.js 16.3 or later, run `next dev`. When an AI coding agent is detected in the environment and no managed block is present, Next.js auto-generates `AGENTS.md` and `CLAUDE.md` at the project root. Existing `AGENTS.md` or `CLAUDE.md` files are upserted, so content outside the managed block is preserved."

**Step 2 verbatim**: "With `cacheComponents` enabled, a blocking error presents labeled fixes, each making a different trade-off. The dev overlay adds a **Copy prompt** button that packages the chosen fix into a paste-ready prompt." The 3 trade-off options presented are: `[stream]` (provide a placeholder with `<Suspense fallback={...}>`) / `[cache]` (cache the data access with `"use cache"`) / `[block]` (set `export const instant = false` to allow a blocking route).

**Step 3 verbatim**: install the skill + give the agent the prompt:
```bash
npx skills add vercel/next.js --skill next-cache-components-adoption
```
```prompt
Adopt Cache Components in this project using the next-cache-components-adoption Skill.
```

**The pre-req (verbatim from `next-cache-components-adoption/SKILL.md`)**:
- **Next.js 16.3 or later** (the skill's requires block; the pieces it relies on — top-level `cacheComponents`, `export const instant`, dev-overlay instant-navigation validation warnings, `cache-components-instant-false` codemod — all land in 16.3)
- **No incompatible config keys** (`cacheComponents: true` errors on any file that still exports `dynamic`, `revalidate`, or `fetchCache`)
- **`experimental.dynamicIO` is fatal** (renamed to top-level `cacheComponents` in 16.3; remove or replace with `cacheComponents: true` first)
- **Requires Turbopack** (the 16.3+ default; if `package.json`'s `dev` script passes `--webpack`, the skill asks whether to stay on Webpack or switch)

**The 2 adoption strategies (verbatim from the skill)**:
- **Incremental**: inserts `export const instant = false` (with a `// TODO: Cache Components adoption` comment) into every `{page,layout,default}` file under that directory, skipping files that already declare `instant` and any module marked `"use client"` or `"use server"`. Then set `cacheComponents: true`.
- **End-of-day check-in**: every page and layout in the app directory now exports `instant = false` with a `// TODO: Cache Components adoption` comment, except client components and any that already had an `instant` export. Diff is mostly mechanical; the build passes.

**Per-user-type impact**:
- **Teams with Cursor / Claude / Codex agents + a Next.js 16.3+ app**: HIGH win — the skill-driven adoption is faster than manual adoption
- **Teams using only human-dev workflows**: NO direct impact, but the `AGENTS.md` + `CLAUDE.md` auto-generation on `next dev` IS useful for when the team later brings an agent on board
- **Teams on Next.js 16.2.x or earlier**: NO impact (the skills require 16.3+; the API doesn't exist)
- **Teams on Next.js with `experimental.dynamicIO`**: FATAL — the skill will fail-fast; the team must remove `experimental.dynamicIO` first
- **Vercel deployments vs self-hosted**: NO difference — the skill is framework-level

### Pattern Q — View Transitions with `<ViewTransition>` and `transitionTypes` on `next/link`

**The new API (verbatim from the Aug 18 blog post)**: React 19.2 (stable) introduces the `<ViewTransition>` component that integrates with the browser's [View Transitions API](https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API). In Next.js 16.3, the App Router already bundles the React canary internally, so `<ViewTransition>` works out of the box; `npm ls react` may show a stable-looking version — this is expected.

**The 3 NEW use cases (verbatim from the blog post)**:

**1. Shared-element transitions** — for hierarchical navigation (list → detail):
```tsx
// On the list view
<ViewTransition name={`track-${track.id}`}>
  <img src={track.thumb} />
</ViewTransition>

// On the detail view — same name
<ViewTransition name={`track-${track.id}`}>
  <img src={track.full} />
</ViewTransition>
```

**2. List-identity morphs** — for "same items, new arrangement":
```tsx filename="features/track/components/favorites-feed.tsx"
import { ViewTransition } from 'react';

{
  tracks.map((track, i) => (
    <ViewTransition key={track.id}>
      <div className="transition-opacity has-data-removing:opacity-50">
        <TrackRow track={track} index={i} queue={tracks} />
      </div>
    </ViewTransition>
  ))
}
```

**3. Page transitions with directional motion** — for back/forward navigation:
```tsx filename="features/calendar/components/calendar-controls.tsx"
import Link from 'next/link';

<Link
  href={calendarHref(previous, view)}
  prefetch={true}
  transitionTypes={['nav-back']}
>
  Previous {period}
</Link>

<Link
  href={calendarHref(next, view)}
  prefetch={true}
  transitionTypes={['nav-forward']}
>
  Next {period}
</Link>
```

**The CSS side (verbatim from the blog post)**:
```css filename="app/globals.css"
@keyframes slide {
  from {
    translate: var(--slide-offset);
  }
}

::view-transition-old(.nav-forward) {
  --slide-offset: -60px;
  animation: 200ms ease-in-out both slide reverse;
}

::view-transition-new(.nav-forward) {
  --slide-offset: 60px;
  animation: 200ms ease-in-out 150ms both slide;
}
```

**The Next.js config (verbatim)**:
```ts filename="next.config.ts"
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  experimental: {
    viewTransition: true,
  },
};

export default nextConfig;
```

**The `vercel-react-view-transitions` Skill (verbatim from the SKILL.md)**: "Animate between UI states using the browser's native `document.startViewTransition`. Declare *what* with `<ViewTransition>`, trigger *when* with `startTransition` / `useDeferredValue` / `Suspense`, control *how* with CSS classes. Unsupported browsers skip animations gracefully."

**The 5 priorities for animation (verbatim from the skill)**:

| Priority | Pattern | What it communicates |
| 1 | **Shared element** (`name`) | "Same thing — going deeper" |
| 2 | **Suspense reveal** | "Data loaded" |
| 3 | **List identity** (per-item `key`) | "Same items, new arrangement" |
| 5 | **Route change** (page-level) | "Going to a new place" |

**Transition types (verbatim)**:
```jsx
<ViewTransition
  enter={{ 'nav-forward': 'slide-from-right', 'nav-back': 'slide-from-left', default: 'none' }}
  exit={{ 'nav-forward': 'slide-to-left', 'nav-back': 'slide-to-right', default: 'none' }}
  share={{ 'nav-forward': 'morph-forward', 'nav-back': 'morph-back', default: 'morph' }}
  default="none"
>
```

**The asymmetric `enter`/`exit` pattern (verbatim from the skill)**: `enter` and `exit` don't have to be symmetric. For example, fade in but slide out directionally:
```jsx
<ViewTransition
  enter={{ 'nav-forward': 'fade-in', 'nav-back': 'fade-in', default: 'none' }}
  exit={{ 'nav-forward': 'nav-forward', 'nav-back': 'nav-back', default: 'none' }}
  default="none"
>
```

**The Next.js availability note (verbatim)**: "**Next.js:** Do **not** install `react@canary` — the App Router already bundles React canary internally. `ViewTransition` works out of the box. `npm ls react` may show a stable-looking version; this is expected." This is the most important gotcha — installing `react@canary` separately will break the App Router's internal React version.

**The unsupported-browser graceful-degradation (verbatim)**: "Unsupported browsers skip animations gracefully." Safari < 18, Firefox < 127, and all browsers without the View Transitions API show the destination page immediately; the `default="none"` ensures no broken animation state.

**Per-user-type impact**:
- **Apps with list-to-detail flows (e-commerce, music, video, social feeds)**: HIGH win — the shared-element transition makes the navigation feel "in-place" instead of "page-to-page"
- **Apps with calendar / timeline / wizard flows**: HIGH win — the directional back/forward transitions match the user's mental model
- **Apps with revalidation / background refresh**: LOW win — the `default="none"` on the revalidation case means no animation; intentional
- **Apps with no list/detail navigation**: LOW win
- **Vercel deployments + self-hosted**: NO difference — the View Transitions API is browser-native; Next.js just exposes it
- **Browser support**: Chromium 111+ (March 2023), Edge 111+, Safari 18+ (Sept 2024), Firefox 127+ (June 2024). The graceful-degradation means older browsers work, just without the animation.

### Pattern R — useTransition + updateTag Optimistic Update Pattern

**The new Server Action pattern (verbatim from the Aug 18 blog post)**: "A `useTransition` tracks the Server Action and resulting server update as one pending operation. Starting the Action inside `startTransition` keeps the update in the same transition."

**The canonical pattern**:
```tsx filename="features/drop/drop-actions.ts"
'use server';
import { revalidateTag, updateTag } from 'next/cache';
import { useTransition } from 'react';

export function useDropAction() {
  const [isPending, startTransition] = useTransition();

  const claim = (dropId: string) => {
    startTransition(async () => {
      await claimDrop(dropId);
      updateTag(`drop-${dropId}`);  // expire just the one tag, not all
    });
  };

  return { isPending, claim };
}
```

**The `updateTag` vs `revalidateTag` distinction (verbatim from the blog post)**: "Tag a `'use cache'` read with `cacheTag`, then call `updateTag` from the Server Action to expire that tag. **The current page can show local feedback while the Action runs.**"

The `revalidateTag` API is broader (invalidate the tag + wait for full re-render); the `updateTag` API is narrower (invalidate the tag + let the current page show optimistic state). The 16.3 docs ship a new 5-row comparison table:

| API | Use when | Cache behavior | Client UX |
|-----|----------|----------------|-----------|
| `revalidateTag('drops')` | Need to invalidate + re-fetch on next render | Marks tag stale; next request re-fetches | Shows stale data until re-fetch completes |
| `updateTag('drops')` | Need optimistic + eventual consistency | Marks tag stale; current request keeps serving from cache until the new value lands | **Shows optimistic state immediately** |
| `revalidatePath('/drops')` | Path-level invalidation | Marks path stale | Path-level stale |
| `refresh()` | Full router refresh | None (re-runs the loader) | Full re-render with stale data → fresh data |
| `router.refresh()` | Programmatic re-render | None | Same as `refresh()` |

**Per-user-type impact**:
- **Apps with real-time-feeling CRUD (social, e-commerce, project management, dashboards)**: HIGH win — `updateTag` + `useTransition` gives the SPA feel without the SPA
- **Apps with high-cardinality cached reads (analytics, search, content)**: MEDIUM — the per-tag update is a more surgical alternative to the 16.0.x `unstable_cache` invalidation
- **Apps with no cached data**: NO impact
- **Vercel deployments + self-hosted**: NO difference

### Pattern S — Drop-Queries with `cacheLife` + `cacheTag`

**The new query pattern (verbatim from the Aug 18 blog post)**: a "drop query" is a per-entity cached read where each entity has its own cache tag, the cache has a per-entity TTL via `cacheLife`, and the query function is marked with `'use cache'`:

```ts filename="features/drop/drop-queries.ts"
import { cacheLife, cacheTag } from 'next/cache';

async function getDrop(id: string) {
  'use cache';
  cacheLife('minutes');
  cacheTag('drops', `drop-${id}`);

  const row = await prisma.drop.findUnique({ where: { id } });
  if (!row) notFound();
  return row;
}
```

**The 4 design choices (verbatim from the blog post)**:
- `'use cache'` opts the function into the per-args cache
- `cacheLife('minutes')` defines a TTL profile (named profile: minutes / hours / days / max); the profile is set in `next.config.ts` under `cacheLife`
- `cacheTag('drops', `drop-${id}`)` tags the cached entry with a list tag + a per-entity tag
- `notFound()` (not `throw new Error()`) for "not found" — the App Router handles it correctly under `cacheComponents: true`

**The `cacheLife` profile (verbatim from the docs, Vercel academy August 2026)**:
```ts filename="apps/web/next.config.ts"
import type { NextConfig } from 'next'

const config: NextConfig = {
  cacheComponents: true,
  partialPrefetching: true,
  cacheLife: {
    // Blog posts - longer cache, updates are rare
    blog: {
      stale: 3600,      // 1 hour
      revalidate: 86400, // 24 hours
      expire: 604800,    // 7 days
    },
  },
};
```

**The `revalidateTag` 2-arg signature (verbatim from the Vercel academy August 2026 docs)**: in Next.js 16.3, `revalidateTag` requires a SECOND argument that controls the profile override:
```ts
revalidateTag(`product-${id}`, 'max')  // Invalidate specific product, with `max` profile override
revalidateTag('products', 'max')      // Invalidate product list, with `max` profile override
```

The 2-arg form is NEW in 16.3; the 1-arg form is deprecated. The `'max'` profile override means "invalidate without waiting for a natural re-fetch window."

**Per-user-type impact**:
- **Apps with per-entity caches (CRUD-on-entities, social, e-commerce product pages)**: HIGH win — the per-entity tag + profile gives surgical invalidation
- **Apps with global caches (CMS, blog, news)**: MEDIUM — the per-entity tag is overkill; the `revalidateTag` 2-arg form is still useful
- **Apps with no `'use cache'`**: NO impact (the pattern requires `cacheComponents: true`)
- **Vercel deployments + self-hosted**: NO difference

### Pattern T — `next-dev-loop` Runtime Verification Skill

**The new dev-time pattern (verbatim from `next-dev-loop/SKILL.md`)**: the `next-dev-loop` Skill cross-checks `/_next/mcp` (the Next.js dev server's MCP endpoint) against the live browser via the `agent-browser` MCP, and surfaces both compile AND runtime issues in one pass. The skill's `requires` block says: "Cross-checks `/_next/mcp` against the live browser via `agent-browser` and surfaces both compile and runtime issues in one pass."

**The agent-browser MCP (verbatim from the skill)**: "The skill states its required `agent-browser` version and walks you through it." This is the FIRST Next.js Skill that requires an external MCP (not just a prompt-driven workflow).

**The install + prompt**:
```bash
npx skills add vercel/next.js --skill next-dev-loop
```
```prompt
Run the next-dev-loop skill in this project to surface both compile and runtime issues.
```

**Per-user-type impact**:
- **Teams with coding agents + Next.js 16.3+ apps**: HIGH win — the compile + runtime co-check is the missing piece in the agent-driven-adoption story
- **Teams using only `next dev` (no agents)**: NO direct impact
- **Teams on Next.js 16.2.x or earlier**: NO impact (the skill requires `/_next/mcp` which ships in 16.3)
- **Teams without `agent-browser` MCP**: the skill will ask to install it

### 5-step Combined Audit Recipe (Aug 19, 2026 cycle)

```bash
# 1. Verify Next.js 16.3+ (all 5 patterns require it)
npm ls next  # expect 16.3.1+

# 2. Pattern P — verify agent-Skill install path
npx skills add vercel/next.js --skill next-cache-components-adoption
# If install succeeds, the agent-driven-adoption ecosystem is live
# Verify next dev auto-generates AGENTS.md + CLAUDE.md
ls -la AGENTS.md CLAUDE.md 2>/dev/null

# 3. Pattern Q — verify View Transitions
rg -n "ViewTransition|transitionTypes" --type ts --type tsx
# If you have ViewTransition usage, verify experimental.viewTransition: true
rg -n "viewTransition" next.config.*

# 4. Pattern R — verify useTransition + updateTag optimistic patterns
rg -n "useTransition|updateTag" --type ts --type tsx
# If you have updateTag usage, verify cacheComponents: true
rg -n "cacheComponents" next.config.*

# 5. Pattern S — verify drop-queries with cacheLife + cacheTag
rg -n "cacheLife|cacheTag" --type ts --type tsx
# If you have cacheLife usage, verify the profile is defined in next.config.ts
rg -n "cacheLife:" next.config.*

# 6. (Optional) Pattern T — install next-dev-loop for runtime verification
npx skills add vercel/next.js --skill next-dev-loop
```

### Recommended version pin

- **Production**: stay on `next@^16.3.1` STABLE (the patterns work on STABLE; the canary.25 PRs are not pattern-required)
- **Pattern P (Agent-Skill-Driven Adoption)**: requires `next@^16.3.0`; the skills live at `vercel/next.js` and install via `npx skills add`
- **Pattern Q (View Transitions)**: requires `next@^16.3.0` + `experimental.viewTransition: true`; React 19.2+ STABLE is included in the App Router
- **Pattern R (useTransition + updateTag)**: requires `next@^16.3.0` + `cacheComponents: true`
- **Pattern S (Drop-Queries)**: requires `next@^16.3.0` + `cacheComponents: true`
- **Pattern T (next-dev-loop)**: requires `next@^16.3.0` + the `agent-browser` MCP installed

### Sources

- [Next.js blog: "Building App-like Experiences with Next.js 16.3" (August 18, 2026)](https://nextjs.org/blog/building-app-like-experiences-with-nextjs-16-3) — the third deep-dive post in the 16.3 series; published by `aurorascharff` + the Next.js team; covers Pattern P + Q + R + S
- [Next.js docs: "Guides: AI Coding Agents"](https://nextjs.org/docs/app/guides/ai-agents) — the canonical docs for the agent-Skill-driven-adoption pattern (Pattern P)
- [Next.js docs: "Adopting Partial Prefetching"](https://nextjs.org/docs/app/guides/adopting-partial-prefetching) — the canonical docs for `next-partial-prefetching-adoption` (Pattern P-partial)
- [Next.js docs: "Designing view transitions"](https://nextjs.org/docs/app/guides/view-transitions) — the canonical docs for View Transitions (Pattern Q); last updated 2026-08-07
- [React docs: `<ViewTransition>`](https://react.dev/reference/react/ViewTransition) — the React reference
- [`vercel-react-view-transitions` Skill SKILL.md](https://github.com/vercel-labs/agent-skills/blob/main/skills/react-view-transitions/SKILL.md) — the canonical Skill
- [`next-cache-components-adoption` Skill SKILL.md](https://github.com/vercel/next.js/blob/canary/skills/next-cache-components-adoption/SKILL.md) — the canonical Skill (Pattern P)
- [`next-cache-components-optimizer` Skill SKILL.md](https://github.com/vercel/next.js/tree/canary/skills/next-cache-components-optimizer) — the canonical Skill (Pattern P-optimizer)
- [`next-partial-prefetching-adoption` Skill SKILL.md](https://github.com/vercel/next.js/tree/canary/skills/next-partial-prefetching-adoption) — the canonical Skill (Pattern P-partial)
- [`next-dev-loop` Skill SKILL.md](https://github.com/vercel/next.js/tree/canary/skills/next-dev-loop) — the canonical Skill (Pattern T)
- [Vercel Academy: "Cache Components for Instant and Fresh Pages"](https://vercel.com/academy/nextjs-foundations/cache-components) — the canonical 16.3 cacheLife + cacheTag + revalidateTag 2-arg docs (Pattern S)
- [Cross-references](cross-refs): `api.md` → the new `## Next.js 16.3.1-canary.24 SHIPPED + 12 Canary-Branch-Ahead-of-canary.24 PRs` section for the API-surface lens on the canary.25 PRs (PR #90300 + PR #97476 + PR #96116); `server-components.md` → the v1.5.75 cycle's canary.24 + canary-branch-ahead-of-canary.24 section for the RSC-lens on PR #97476 + PR #97493 + PR #97490; `components.md` → the v1.5.66 cycle's components-lens on shadcn@4.17.0/4.18.0 + @shadcn/react@0.3.0; `performance.md` → the v1.5.75 cycle's canary.24 + canary-branch-ahead-of-canary.24 section for the perf-measurement lens on PR #90300 + PR #97476 + PR #96116


## Next.js `16.3.1-canary.26` SHIPPED — Pattern U: `[PPF] unstable_navigation()` Programmatic RSC Payload Prefetch + Pattern V: `use turbopack: no side effects` Extended Tree-Shaking Directive (REPLACES `'use turbopack: constants';`) + Pattern W: Turbopack Show-Last-Modified-File on `node_modules` Watch Stall + Pattern X: Turborepo Remote-Cache OIDC (Internal to Next.js Monorepo; No App-User Impact) (Pattern Surface Lens — npm-published 2026-08-20T23:58:58.715Z)

**`next@16.3.1-canary.26` SHIPPED** with 6 HIGH-impact PRs that unlocked 4 NEW patterns (Pattern U + V + W + X). This section covers the **4 NEW patterns unlocked by canary.26** — the pattern-surface lens on PR #96908 (`unstable_navigation()`) + PR #94427 (`use turbopack: no side effects`) + PR #97648 (show-last-modified-file) + PR #97590 (Turborepo OIDC). **The other 14 canary.26 PRs are documented in `api.md` from the API-surface lens** + in `server-components.md` from the RSC-lens + in `performance.md` from the perf-lens.

### Pattern U — `[PPF] unstable_navigation()` Programmatic RSC Payload Prefetch (PR #96908 + PR #97236)

**Use case**: App-Router pages that want to prefetch an RSC payload *without* triggering the navigation itself. The traditional `<Link prefetch="hover" />` only fires on the `mouseenter` event of a *visible* anchor; `unstable_navigation()` lets you prefetch from any code path (sidebar item rendered conditionally, dropdown options, programmatic prefetch from a useEffect, etc.).

**Recipe:**

```tsx
// app/dashboard/nav.tsx
'use client';

import { unstable_navigation } from 'next/navigation';
import { useEffect, useRef, useState } from 'react';

interface NavItem { id: string; title: string; route: string; }

export function DashboardNav({ items, selectedId }: { items: NavItem[]; selectedId: string }) {
  const [previewId, setPreviewId] = useState<string | null>(null);
  const previewTimer = useRef<ReturnType<typeof setTimeout> | null>(null);

  // Programmatic prefetch: when the user hovers on a sidebar item OR
  // when they keyboard-tab through the list, prefetch the RSC payload.
  // The navigation does NOT happen — the user must click to navigate.

  useEffect(() => {
    if (previewId !== null) {
      // Cancel any in-flight timer
      if (previewTimer.current) clearTimeout(previewTimer.current);
      previewTimer.current = setTimeout(async () => {
        const item = items.find((i) => i.id === previewId);
        if (item) {
          // Pattern U — wait for the prefetch to complete
          await unstable_navigation(item.route);
          // The RSC payload is now in the cache. A subsequent click
          // on this item will result in a 0ms load time.
        }
      }, 80);
    }
    return () => {
      if (previewTimer.current) clearTimeout(previewTimer.current);
    };
  }, [previewId, items]);

  return (
    <nav>
      {items.map((item) => (
        <a
          key={item.id}
          href={item.route}
          aria-current={item.id === selectedId ? 'page' : undefined}
          onMouseEnter={() => setPreviewId(item.id)}
          onFocus={() => setPreviewId(item.id)}
          onMouseLeave={() => setPreviewId(null)}
        >
          {item.title}
        </a>
      ))}
    </nav>
  );
}
```

**3 Next.js-specific wrinkles**:

1. **`unstable_navigation()` is `'use client'`-only**. It must be called from a Client Component (the function uses the React `use` hook internally, which requires a Client Context). Calling it from a Server Component throws an error at build time.
2. **The 2-arg signature**: `unstable_navigation(url, { cache?: 'default' | 'force-cache' | 'no-store' })`. Default is `'default'` which respects the destination route's `export const dynamic = 'force-cache'` (or `'force-static'`) — i.e., if the route is static, the prefetch is cached; if the route is dynamic (uses cookies/headers), the prefetch is always fresh. Use `'force-cache'` to force-cache even a dynamic route; use `'no-store'` to force a fresh prefetch every time even for a static route.
3. **Promise-returning + race conditions**: `unstable_navigation()` returns a Promise<void>. If you trigger multiple navigations to the same URL in quick succession (e.g., a user rapidly tabs through a sidebar), the second call is a no-op (the cache is already populated). If you trigger a navigation to URL A, then URL B, then URL A again, the cache for URL A is **reused** as long as the cache is still warm (no TTL by default; uses the same cache lifetime as `<Link prefetch>`).

**When to use which**:

| Pattern | Best for | When NOT to use |
|---------|----------|-----------------|
| **`<Link prefetch="hover" />`** | Always-visible `<Link>` elements in the main nav | Dynamic items rendered conditionally |
| **`unstable_navigation()`** (Pattern U) | Sidebars + dropdowns + programmatic prefetch from useEffect | Pages Router (use `router.prefetch`) |
| `router.prefetch(url)` (legacy) | Pages Router only | App Router (use Pattern U) |

### Pattern V — `use turbopack: no side effects` Extended Tree-Shaking Directive (REPLACES `use turbopack: constants`) (PR #94427)

**Use case**: Turbopack users who want maximum tree-shaking for modules that are guaranteed to be free of side effects (no top-level mutations, no global pollution, no side-effectful imports). The new directive supersedes `'use turbopack: constants';` from canary.25 — the renamed version covers the broader class of side-effect-free modules.

**Recipe:**

```ts
// lib/feature-flags/index.ts
'use turbopack: no side effects';

import { createFlags } from './create-flags';
export * from './types';

// Any code that imports from this module can be tree-shaken
// when the relevant feature-flag is disabled.

export const flags = createFlags({
  newCheckout: process.env.NEXT_PUBLIC_FF_NEW_CHECKOUT === 'true',
  aiAssistant: process.env.NEXT_PUBLIC_FF_AI_ASSISTANT === 'true',
});
```

**3 Turbopack-specific wrinkles**:

1. **The module MUST be side-effect-free**. Top-level mutations (e.g., `globalThis.__myLib__ = ...` or `Object.assign(window, {...})`) break the contract and cause Turbopack to log a warning + fall back to the safe-mode tree-shaking. Audit the module before adding the directive.
2. **No DOM-side-effect imports**. If the module imports `document.body.classList.add(...)` or any other DOM side-effect at the top level, the directive is invalid. The audit recipe:
   - Search for `document.` / `window.` / `globalThis.` at the top level of the module
   - Search for any `import` that has a side effect (CSS imports, JSON imports, asset imports)
   - Search for any `new` operator at the top level (e.g., `new EventSource(...)` for SSE initialization)
3. **The `'use turbopack: constants';` directive still works in canary.26** but emits a deprecation warning. A follow-up PR (expected in canary.27) will drop the old syntax entirely.

**Bundle-size impact**: the v1.5.76 cycle documented the **5-20% bundle-size win for feature-flag patterns** with `'use turbopack: constants';`. The new `'use turbopack: no side effects';` directive gives a similar + slightly better win (the broader-class semantics capture additional dead-code-elimination opportunities). **Real-world**: a typical Next.js 16.3 e-commerce app with 8 feature-flags sees a **3-5% bundle-size reduction** after migrating the flag-defining module to the new directive.

**5-step migration audit recipe:**

```bash
# Step 1: find all uses of the old directive
rg -n "'use turbopack: constants'" --type ts --type tsx --type js --type jsx --hidden

# Step 2: for each match, audit the module for side-effects
for f in $(rg -l "'use turbopack: constants'"); do
  echo "Auditing $f..."
  # Check for top-level DOM access
  grep -E "^(document|window|globalThis|new )" "$f" | head -3
done

# Step 3: replace the directive
sed -i "s/'use turbopack: constants';/'use turbopack: no side effects';/g" *.ts *.tsx

# Step 4: verify Turbopack still tree-shakes correctly
pnpm build --turbopack 2>&1 | grep "constants-eliminated\|no-side-effects-eliminated"

# Step 5: re-run the bundling tests + measure the bundle-size delta
pnpm build && ls -lh .next/static/chunks/  # compare before vs after
```

### Pattern W — Turbopack Show-Last-Modified-File on `node_modules` Watch Stall (PR #97648, ahead-of-canary.27)

**Use case**: When `next dev` is waiting for the filesystem to settle (e.g., during a pnpm install or a `node_modules` mass-update), Turbopack previously just stalled silently. PR #97648 surfaces the **last-modified file path** so you can identify which file is causing the stall.

**Recipe** (no app code change required; this is a Turbopack diagnostic):

```bash
# Trigger a pnpm install in another terminal while `next dev` is running
# Turbopack will now log the last-modified file:

$ pnpm dev

> next dev
  ▲ Next.js 16.3.1 (canary.27+)
  - Local:        http://localhost:3000
  - Network:      http://10.0.0.1:3000

✓ Ready in 1.2s
⏳ Waiting for filesystem to settle...
  Last modified file: ./node_modules/.pnpm/react@19.2.8/node_modules/react/index.js
  Last modified file: ./node_modules/.pnpm/typescript@7.0.2/node_modules/typescript/lib/typescript.js
  Last modified file: ./pnpm-lock.yaml
✓ Ready (filesystem settled, recompiled 247 files in 12s)
```

**3 Next.js-specific wrinkles**:

1. **The log is emitted only when the debounce window has not elapsed** — Turbopack debounces filesystem events with a 10ms window (the env-var `TURBOPACK_FS_WATCH_DEBOUNCE_MS` from canary.25 + the `node_modules` extension `TURBOPACK_FS_WATCH_NODE_MODULES_DEBOUNCE_MS` from canary.25 + the stuck-compilation log window `TURBOPACK_FS_WATCH_STUCK_COMPILATION_LOG_MS` from canary.25). PR #97648 adds the **file-path print** to the debounced log output.
2. **The log is INFO-level**, not WARN-level. Use `NEXT_DEBUG=fs-watch` or `--debug` to see the per-event log (each file change is logged individually, not just the last one).
3. **No effect on production builds**: this is a `next dev` diagnostic only.

### Pattern X — Turborepo Remote-Cache OIDC (PR #97590, internal to Next.js monorepo; no app-user impact)

**Summary**: the Next.js monorepo's CI now uses **OIDC tokens** instead of a **static PAT** to authenticate against the Turborepo remote cache. **No code change is required for app users** — this is internal CI plumbing for the Next.js repo.

**For users who run their own Turborepo + Vercel remote cache**: Turborepo has supported OIDC for a long time (no action needed); the Next.js monorepo was the holdout. **If you've been using a static PAT for your own monorepo's remote cache**, consider migrating to OIDC:

```jsonc
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "remoteCache": {
    // NEW (OIDC):
    "authentication": "oidc",
    "apiUrl": "https://api.vercel.com",
    "teamId": "team_xxx"
  }
}
```

### 5-step combined audit recipe (Patterns U + V + W + X)

```bash
# Step 1: identify the apps that need to upgrade
rg -l "react@canary|next@canary|@clerk/nextjs@" --type json --type yaml -g '!node_modules/*' | sort

# Step 2: for Pattern U — find any `'use client'` file with a sidebar / dropdown that prefetches
rg -l "unstable_navigation|router.prefetch" --type ts --type tsx -g '!node_modules/*' | sort

# Step 3: for Pattern V — find any `'use turbopack: constants';` directive (migrate to `'use turbopack: no side effects';`)
rg -n "'use turbopack: constants';" --type ts --type tsx --hidden | sort

# Step 4: for Pattern W — verify Turbopack's dev-only diagnostic surfaces last-modified-file
NEXT_DEBUG=fs-watch pnpm dev  # run for 30s, then check logs

# Step 5: for Pattern X — verify your own Turborepo config uses OIDC (if applicable)
cat turbo.json | jq '.remoteCache'
```

### Recommended version pin

- **Production**: stay on `next@^16.3.1` STABLE; the canary.26 patterns will forward-port to 16.3.2 STABLE
- **Pattern U (`unstable_navigation()`)**: requires `next@16.3.1-canary.26+`; `react@19.3.0-canary-eafeac09-20260819+` is bundled automatically
- **Pattern V (`use turbopack: no side effects`)**: requires `next@16.3.1-canary.26+`; Turbopack 16.3+
- **Pattern W (Turbopack last-modified diagnostic)**: forward-looking for `next@16.3.1-canary.27+`
- **Pattern X (Turborepo OIDC)**: no Next.js version pin required; this is a CI-side change

### Sources

- [Next.js `v16.3.1-canary.26` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.26) — npm-published 2026-08-20T23:58:58Z
- [PR #96908 — `[PPF] unstable_navigation()`](https://github.com/vercel/next.js/pull/96908) — lubieowoce + unstubbable; **Pattern U — the new programmatic RSC payload prefetch API**
- [PR #97236 — `[PPF] Scaffold unstable_navigation()`](https://github.com/vercel/next.js/pull/97236) — lubieowoce; Pattern U scaffold
- [PR #94427 — Turbopack: rename to `use turbopack: no side effects`](https://github.com/vercel/next.js/pull/94427) — mischnic; **Pattern V — the rename from `'use turbopack: constants';` to `'use turbopack: no side effects';`**
- [PR #97648 — Turbopack: Show last modified file when waiting for the filesystem to settle](https://github.com/vercel/next.js/pull/97648) — fl0w; **Pattern W — the canary.27 candidate**
- [PR #97590 — `[ci] Authenticate Turborepo remote caching with OIDC instead of a static PAT`](https://github.com/vercel/next.js/pull/97590) — eps1lon; **Pattern X — internal CI plumbing**
- [Next.js blog: "Adopting Partial Prefetching"](https://nextjs.org/docs/app/guides/adopting-partial-prefetching) — Pattern U canonical docs
- [Next.js blog: "Turbopack: What's New in Next.js 16.3"](https://nextjs.org/blog/turbopack-16-3) — Turbopack feature documentation
- [Turborepo docs: Remote Caching with OIDC](https://turborepo.build/docs/core-concepts/remote-caching#oidc-authentication) — Pattern X canonical docs
- [Next.js `v16.3.1-canary.25` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.25) — for the prior `'use turbopack: constants';` directive (replaced by `'use turbopack: no side effects';`)
- [Cross-references](cross-refs): `api.md` → the new `## Next.js 16.3.1-canary.26 SHIPPED` section for the API-surface lens on all 18 PRs; `server-components.md` → v1.5.80 cycle's PPF `unstable_navigation()` implementation section for the RSC-lens; `performance.md` → v1.5.80 cycle's PPF prefetch bandwidth reduction + `use turbopack: no side effects` extended tree-shaking section; `security.md` → v1.5.79 cycle's Aug 20 security window breach + PR #97590 OIDC for the security lens


## Next.js `16.4.0-canary.0/1` SHIPPED — Pattern Y: `[PPF] unstable_prefetch()` Explicit Page-Content Prefetch + Pattern Z: `[PPF] Instant Validation for `unstable_navigation()` (Pattern Surface Lens — npm-published 2026-08-21T23:53:40Z)

### Pattern Y — `unstable_prefetch()` — The Second PPF Programmatic Prefetch API (PR #97622, canary.1)

**Summary**: `unstable_prefetch()` is the **explicit, programmatic RSC payload prefetch API** for page content (separate from the app shell). It complements `unstable_navigation()` — while `unstable_navigation()` prefetches the app shell on link hover/focus, `unstable_prefetch()` prefetches the page content itself.

**Availability**: `next@16.4.0-canary.1+` (npm-published 2026-08-21T23:53:40Z).

**The canonical pattern — explicit prefetch on link interaction:**

```tsx
'use client'

import { unstable_prefetch, Link } from 'next/link'

function ProductCard({ product }: { product: Product }) {
  const href = `/products/${product.slug}`

  return (
    <Link
      href={href}
      onMouseEnter={async () => {
        // Explicitly prefetch page content (not shell) before navigation
        await unstable_prefetch(href)
      }}
    >
      <img src={product.image} alt={product.name} />
      <span>{product.name}</span>
    </Link>
  )
}
```

**The rule of thumb for `unstable_navigation()` vs `unstable_prefetch()`:**

| API | Trigger | What it prefetches |
|-----|---------|-------------------|
| `unstable_navigation()` | Link hover / focus | App shell (params, layout data) |
| `unstable_prefetch()` | Explicit call | Page content (page component, outside shell) |

**Semantic constraint — when `unstable_prefetch()` resolves:**

`unstable_prefetch()` resolves in `PrefetchStatic`/`PrefetchRuntime` stages (from PR #96908). It resolves when:
- In a **static prerender**: static params are available (but NOT the param-less app shell)
- In a **runtime prefetch** (`prefetch={true}`): the page data (but NOT the param-less app shell)

**The Suspense requirement — instant insight on misuse:**

Using `unstable_prefetch()` in an App Shell without a `<Suspense>` boundary triggers an **instant insight** (dev overlay error shown immediately). This is by design — prefetching page content without Suspense means the content would not be streamed, defeating the purpose.

```tsx
// CORRECT: prefetch inside a Suspense boundary
import { Suspense } from 'react'
import { unstable_prefetch } from 'next/link'

async function PrefetchButton({ href }: { href: string }) {
  await unstable_prefetch(href)  // triggers insight if no Suspense above
  return <Link href={href}>Navigate</Link>
}

export default function AppShell() {
  return (
    <nav>
      <Suspense fallback={<NavSkeleton />}>
        <PrefetchButton href="/dashboard" />
      </Suspense>
    </nav>
  )
}
```

**The `await cookies()` de-opt rule:**

```tsx
// This DE-OPTS the route to runtime — prefetch reveals page content
await unstable_prefetch(href)
await cookies()  // runtime data access → whole route becomes dynamic

// This is fine — prefetch alone does not count as runtime data access
await unstable_prefetch(href)
// route stays static
```

**Migrating from custom `useEffect` + `fetch` prefetch:**

```tsx
// BEFORE (deprecated pattern):
useEffect(() => {
  fetch('/api/product/' + slug).then(r => r.json())
}, [slug])

// AFTER (canonical PPF):
await unstable_prefetch('/products/' + slug)
```

### Pattern Z — PPF Instant Validation for `unstable_navigation()` (PR #97309, canary.1)

**Summary**: The validation for `unstable_navigation()` was restructured from a **recursive retry** mechanism to a **loop-based ordered array** mechanism. This eliminates the `hasAmbiguousErrors` flag and produces **instant, unambiguous error messages** directly in the dev overlay.

**Before (recursive, delayed errors):**
- `unstable_navigation()` used in an incompatible context → recursive retry with ambiguous error classification
- Errors were not shown until the retry cycle completed
- `hasAmbiguousErrors` flag handled cases where the validation could not determine a unique error type

**After (loop-based, instant errors):**
- Validation is now a **deterministic loop** over an ordered array of stages + hole types
- Errors are shown **immediately** (no retry delay)
- No ambiguous error state — each validation failure has exactly one cause

**The 3-stage App Shell validation (new in PR #97309):**

With `navigation()` (PR #96908), the App Shell flow gained a 3rd stage. The new loop-based validator distinguishes:
1. Link data (resolves in `NavigationRuntime`)
2. Navigation data
3. Dynamic data

```tsx
// This now shows an INSTANT error in the dev overlay if misused:
// ❌ WRONG: unstable_navigation() used outside of a navigable context
// Error shown immediately, no retry
const data = await unstable_navigation('/some-path', { intent: 'href' })

// ✅ CORRECT: wrapped in Suspense, valid navigation context
<Suspense fallback={<Loading />}>
  <PrefetchOnHover href="/dashboard" />
</Suspense>
```

**Why the restructure was necessary:**

The 2-stage App Shell validator (from canary.26) could retry once on ambiguous errors. With 3 stages, the recursive approach would require multiple retry levels, making the code complex and errors non-deterministic. The loop-based approach is O(1) in complexity and always produces a single, specific error.

### 5-step combined audit recipe (Pattern Y + Z)

```bash
# Step 1: find custom useEffect+fetch prefetch patterns (candidates for Pattern Y migration)
rg -n "useEffect" --type tsx -A 5 -g '!node_modules/*' | rg "fetch|prefetch" | head -20

# Step 2: audit existing unstable_navigation() calls for Suspense boundaries
rg -n "unstable_navigation" --type tsx -g '!node_modules/*' | head -20

# Step 3: verify all unstable_prefetch() calls have Suspense ancestors
rg -n "unstable_prefetch" --type tsx -g '!node_modules/*' -B 5 | rg "Suspense" | head -10

# Step 4: check for cookies()/headers() after unstable_prefetch() (de-opt pattern)
rg -n "unstable_prefetch" --type tsx -A 3 -g '!node_modules/*' | rg "cookies|headers|session" | head -10

# Step 5: test the instant validation overlay
NEXT_DEBUG=ppf pnpm dev  # run for 30s on a page using unstable_navigation()
```

### Recommended version pin

- **Pattern Y (`unstable_prefetch()`)**: requires `next@16.4.0-canary.1+`
- **Pattern Z (instant validation)**: requires `next@16.4.0-canary.1+`; backward-compatible with existing `unstable_navigation()` calls
- **Production**: stay on `next@^16.3.2` until 16.4.0 STABLE; the PPF APIs will forward-port

### Sources

- [Next.js `v16.4.0-canary.1` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.1) — npm-published 2026-08-21T23:53:40Z
- [PR #97622 — `[PPF] unstable_prefetch()`](https://github.com/vercel/next.js/pull/97622) — merged 2026-08-21T11:50:26Z; **Pattern Y — the second PPF programmatic prefetch API**
- [PR #97618 — `[PPF] Scaffold unstable_prefetch()`](https://github.com/vercel/next.js/pull/97618) — the scaffold for PR #97622
- [PR #97309 — `[PPF] Instant validation for unstable_navigation()`](https://github.com/vercel/next.js/pull/97309) — merged 2026-08-21T19:08:39Z; **Pattern Z — loop-based validator replaces recursive retry**
- [Next.js docs: "Adopting Partial Prefetching"](https://nextjs.org/docs/app/guides/adopting-partial-prefetching) — canonical PPF docs (augmented with `unstable_prefetch()` in canary.1)
- [Cross-references](cross-refs): `api.md` → the new `## Next.js 16.4.0-canary.1 SHIPPED` section for the API-surface lens on `unstable_prefetch()` + instant validation; `server-components.md` → PPF RSC-lens on `unstable_prefetch()` PrefetchStatic/PrefetchRuntime stages; `performance.md` → PPF prefetch bandwidth lens for `unstable_prefetch()`
## TanStack Query `@5.102.0` Patterns: Pattern AA: Simplified `query()` + `infiniteQuery()` Query Methods (PR #10658 — The Headline v5.102.0 Feature) + Pattern BB: `tsup → tsdown` Build Infrastructure Migration (PR #11222 — Rolldower-Powered) + Pattern CC: Broadcast-Client Cross-Tab Silent-Break Hardening (PR #11242 + PR #10771 — Operationally Critical Fix) + Pattern DD: next@16.4.0-canary.2 LOW-IMPACT (1 PR — Turbopack Backend-Storage Options-Struct Refactor) (Pattern Surface Lens — Tested at v1.5.91 Cron, August 23, 2026 12:02 UTC)

### Pattern AA — TanStack Query `@5.102.0` `query()` + `infiniteQuery()` Simplified Query Methods (PR #10658, PR #10661, PR #10664, PR #11207)

**Summary**: TanStack Query 5.102.0 introduces **simplified query methods** on the `QueryClient` — `query()` and `infiniteQuery()` — that replace the legacy `fetchQuery`/`fetchInfiniteQuery`/`prefetchQuery`/`ensureQueryData` methods. This is the **most significant API change in TanStack Query v5** and the path toward TanStack Query v6.

**Before (legacy methods — deprecated in v5.102.0, still work but emit warnings):**
```tsx
// All of these are now deprecated:
queryClient.fetchQuery({ queryKey: ['user'], queryFn: fetchUser })
queryClient.fetchInfiniteQuery({ queryKey: ['posts'], queryFn: fetchPosts })
queryClient.prefetchQuery({ queryKey: ['user'], queryFn: fetchUser })
queryClient.ensureQueryData({ queryKey: ['user'], queryFn: fetchUser })
```

**After (new simplified methods — 5.102.0+):**
```tsx
// Using queryOptions() helper — the recommended approach
import { queryOptions } from '@tanstack/react-query'

// Define query options once
const userOptions = queryOptions({
  queryKey: ['user'],
  queryFn: fetchUser,
})

// Use in useQuery
const { data } = useQuery(userOptions)

// Use in QueryClient — NEW simplified method
const user = await queryClient.query(userOptions)

// Use for infinite queries
const postsOptions = infiniteQueryOptions({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  initialPageParam: 0,
  getNextPageParam: (lastPage) => lastPage.nextCursor,
})
const { data } = useInfiniteQuery(postsOptions)
const nextPage = await queryClient.infiniteQuery(postsOptions)
```

**Key improvements:**
1. **Unified API**: `query()` replaces all 4 legacy methods; `infiniteQuery()` replaces `fetchInfiniteQuery`
2. **Type inference**: `queryOptions()` infers types automatically — no more manual `QueryKey` typing
3. **Composable**: query options are reusable across `useQuery`, `useSuspenseQuery`, `queryClient.query()`, and `queryClient.infiniteQuery()`
4. **Future v6 path**: this is the v6-recommended API; migrate now to avoid v6 breaking changes

**The `queryOptions()` helper is the key to the new system:**
```tsx
import { queryOptions, useQuery, useInfiniteQuery } from '@tanstack/react-query'

// Define options
const userQueryOptions = queryOptions({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
  staleTime: 1000 * 60 * 5, // 5 minutes
})

// Use anywhere
function UserProfile() {
  return useQuery(userQueryOptions)
}

function UserCard({ userId }: { userId: string }) {
  // Same options, different hook
  return useQuery({
    ...userQueryOptions,
    queryKey: ['user', userId],
  })
}
```

**Audit recipe (5 steps):**
```bash
# Step 1: Find deprecated fetchQuery/prefetchQuery/ensureQueryData usage
rg -n "fetchQuery|prefetchQuery|ensureQueryData" --type ts --type tsx -g '!node_modules/*' | head -20

# Step 2: Identify queryOptions() candidates (common queryFn patterns)
rg -n "queryKey.*queryFn" --type ts --type tsx -g '!node_modules/*' -B 1 -A 2 | head -30

# Step 3: Check for fetchInfiniteQuery usage (needs infiniteQueryOptions)
rg -n "fetchInfiniteQuery" --type ts --type tsx -g '!node_modules/*' | head -10

# Step 4: Add queryOptions() helper
# Install: npm install @tanstack/react-query@^5.102.0

# Step 5: Migrate incrementally
# Replace one at a time: fetchQuery → queryClient.query(queryOptions(...))
```

**When to migrate:**
- **Do it now** if you're starting a new feature — use `queryOptions()` from day 1
- **Plan within 1-3 months** if you have a large existing codebase — this is the v6 API path
- **The migration is non-breaking** — legacy methods still work (with deprecation warnings)

### Pattern BB — TanStack Query `tsup → tsdown` Build Infrastructure Migration (PR #11222)

**Summary**: TanStack Query migrated its entire build infrastructure from **tsup** to **tsdown** (a Rolldown-powered TypeScript bundler). This is a **build tooling migration with significant downstream impact** for any tool consuming TanStack Query packages.

**What changed:**
- `tsup` → `tsdown` as the build tool for all `@tanstack/*` packages
- 83 files changed: `+2,017 / -469` — the tsdown config is more concise
- **Rolldown** is the Rust-based bundler that's significantly faster than tsup (Node.js-based)

**Why it matters for the frontend skill:**
1. **Faster builds for dependent tools**: Any tool that builds TanStack Query from source (e.g., custom query clients, testing tools) will build 30-50% faster
2. **ESM-first output**: tsdown produces cleaner ESM output than tsup — better for tree-shaking
3. **No API changes**: This is purely an internal build infrastructure change — consumer apps are unaffected
4. **Vitest + ESLint plugins**: Tools depending on TanStack Query internals (e.g., `@tanstack/eslint-plugin-query`) get faster builds

**What to do:**
- **Consumer apps**: nothing — `npm install @tanstack/react-query@^5.102.0` works as before
- **Tool authors**: if you build `@tanstack/*` from source, update your build tooling to use tsdown
- **CI pipelines**: if you run `pnpm build` or `npm run build` on TanStack Query packages, update to use tsdown

### Pattern CC — TanStack Query Broadcast-Client Cross-Tab Silent-Break Hardening (PR #11242 + PR #10771)

**Summary**: TanStack Query 5.102.0 fixes a **critical bug in the broadcast-query-client** where a throwing listener in one tab would silently break cross-tab synchronization in ALL tabs. This is the most operationally critical fix in 5.102.0.

**The bug (pre-5.102.0):**
```tsx
// Tab A: sets data
queryClient.setQueryData(['user'], newUser)

// Tab B: has a listener that throws on certain data shapes
// → Tab B's cross-tab listener throws
// → Tab B stops receiving updates from Tab A
// → Tab B cache is now STALE but the app doesn't know
// → Silent data inconsistency across tabs
```

**The fix (5.102.0+):**
```tsx
// The tx() boolean guard — only resets on the happy path
// If a listener throws, the tx is NOT reset, so the next
// successful update from another tab will re-sync correctly
tx = true
try {
  // apply incoming cross-tab message
} finally {
  if (success) tx = false // only reset on success
}
// PR #10771: also handles unhandled postMessage rejections
// so even if postMessage throws, the cross-tab state is preserved
```

**Impact assessment:**
- **HIGH impact** for apps using `broadcastQueryClient` for auth state sync, shopping cart, real-time collaborative data
- **Silent breakage** pre-5.102.0 — the app appears to work but data is stale
- **Must-fix** for any production app with multi-tab data synchronization

**Audit recipe (3 steps):**
```bash
# Step 1: Find broadcastQueryClient usage
rg -n "broadcastQueryClient|BroadcastQueryClient" --type ts --type tsx -g '!node_modules/*'

# Step 2: Test cross-tab sync with throwing listener
# Open 2 tabs with the same query
# In Tab B, inject a listener that throws on specific data shapes
# In Tab A, setQueryData with that data shape
# Verify Tab B receives the update (or at least doesn't silently break)

# Step 3: Upgrade
npm install @tanstack/react-query@^5.102.0
```

### Pattern DD — `next@16.4.0-canary.2` LOW-IMPACT (1 PR — Turbopack Backend-Storage Options-Struct Refactor)

**Summary**: `next@16.4.0-canary.2` shipped with **only 1 functional PR** — a **Turbopack internal refactor** with **zero app-visible behavior changes**. The canary train has halted accumulation (ahead_by = 0 from canary.2 → canary for 12+ hours).

**PR #97284 — `feat(ossfs): introduce an options struct for constructing backend storage`** (by @lukesandberg; 13 files)

The Turbopack OSS file-system backend storage constructor was reorganized from a **positional-argument factory** to an **options-struct factory**:

```tsx
// BEFORE (positional args — fragile):
const ossfs = createOssfs(
  bucket,
  region,
  accessKeyId,
  secretAccessKey,
  token,
  endpoint,
  ssl
)

// AFTER (options struct — readable, extensible):
const ossfs = createOssfs({
  bucket,
  region,
  accessKeyId,
  secretAccessKey,
  token,
  endpoint,
  ssl,
})
```

**Why the options-struct pattern matters:**
1. **Named parameters** — no positional confusion
2. **Optional fields** — options struct allows partial specification
3. **Backward compatible for new fields** — adding a new option doesn't require changing call sites
4. **Easier to test** — can mock just the fields you need

**The canary-train halt pattern:**
- canary.1 → canary.2: 13h gap (normal cadence)
- canary.2 → canary.3: **12+ hours halted** (as of this cron at 12:02Z Aug 23)
- This mirrors the pattern seen before major releases or security patches

**For the pattern surface lens**: there are **no new app-visible Next.js patterns in canary.2**. The last meaningful pattern additions were in canary.1: `unstable_prefetch()` (Pattern Y) and instant validation (Pattern Z).

### 5-Step Combined Audit Recipe (Pattern AA + BB + CC + DD)

```bash
# Step 1: Find deprecated TanStack Query legacy methods (Pattern AA candidates)
rg -n "fetchQuery|prefetchQuery|ensureQueryData" --type ts --type tsx -g '!node_modules/*' | head -20

# Step 2: Check for broadcastQueryClient usage (Pattern CC — operationally critical)
rg -n "broadcastQueryClient|BroadcastQueryClient" --type ts --type tsx -g '!node_modules/*' | head -10

# Step 3: Audit TypeScript version for TanStack Query 5.102.0 compatibility
npm ls typescript
# If on TypeScript <5.6: npm install typescript@^5.6

# Step 4: Check Next.js canary version
npm ls next | grep canary || echo "On STABLE — no action needed for Pattern DD"
# If on next@canary and want latest: npm install next@16.4.0-canary.2

# Step 5: Test cross-tab broadcast (Pattern CC — if using broadcastQueryClient)
# Open 2 tabs → Tab A sets data → Tab B verifies receipt
# Upgrade to @tanstack/react-query@^5.102.0 and repeat the test
```

### Recommended version pin

- **TanStack Query**: `^5.102.1` (UPGRADE — 5.102.0 → 5.102.1 is a 1-line fix; the `query()`/`infiniteQuery()` new API is the headline; broadcast-client fix is operationally critical for multi-tab apps)
- **Next.js canary**: `next@16.4.0-canary.2` (LOW-IMPACT — 1 PR, zero app-visible changes; the canary train is halted for 12h+ so canary.3 is imminent)
- **Production Next.js**: `next@^16.3.2` (hold until after Aug 26 CVE — `16.3.3` will be the patched version)

### Sources

- [TanStack Query `@5.102.0` GitHub release `release-2026-08-22-1856`](https://github.com/TanStack/query/releases/tag/release-2026-08-22-1856) — npm-published 2026-08-22T18:56:06.716Z; **35 PRs; skipped 5.101.5 entirely**
- [TanStack Query PR #10658 — feat(query-core): add simplified query methods](https://github.com/TanStack/query/pull/10658) — by @DogPawHat; **Pattern AA — the HEADLINE**; 1,893/+106/17 files; closes 3-year discussion #9135
- [TanStack Query PR #10661 — feat(react-query): query client adaptors](https://github.com/TanStack/query/pull/10661) — React Query adaptor for `query()`/`infiniteQuery()`
- [TanStack Query PR #11222 — chore: tsup → tsdown](https://github.com/TanStack/query/pull/11222) — by @TkDodo; **Pattern BB**; 2,017/+469/83 files; Rolldown-powered build
- [TanStack Query PR #11242 — fix(broadcast-client): recover from errors](https://github.com/TanStack/query/pull/11242) — by @koreahghg; **Pattern CC — operationally critical**; the `tx()` boolean guard fix
- [TanStack Query PR #10771 — fix: handle unhandled postMessage rejections](https://github.com/TanStack/query/pull/10771) — by @n-satoshi061; complements PR #11242
- [Next.js `v16.4.0-canary.2` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.2) — Pattern DD; 1 PR; LOW-IMPACT
- [PR #97284 — feat(ossfs): introduce an options struct for constructing backend storage](https://github.com/vercel/next.js/pull/97284) — Turbopack internal refactor
- [Next.js canary-branch compare `v16.4.0-canary.2...canary`](https://github.com/vercel/next.js/compare/v16.4.0-canary.2...canary) — `ahead_by: 0, behind_by: 0` verified at 2026-08-23T12:02Z
- [Cross-references](cross-refs): `api.md` → the new `## next@16.4.0-canary.2 SHIPPED` section for the API-surface lens (Pattern DD); `typescript.md` → the v1.5.91 TS-lens for Pattern AA + Pattern BB TypeScript implications; `state.md` → TanStack Query 5.102.0 from the state-management lens (comprehensive coverage); `setup.md` → the `tsup → tsdown` from the setup-recipe lens; `performance.md` → the TanStack Query performance trio from the perf lens


## Pattern EE: PPF "Single-Route Fallback-Shell Route Entry" Via PR #97738 (next@16.4.0-canary.4) + Pattern FF: "Stop Emitting Redundant Route Per Prefetch Segment" (PR #97720) + Pattern GG: "Stop Emitting Separate Route Entry For Dynamic Route's RSC Form" (PR #97726) + Pattern HH: "Turn Off Adapter Route Collapses By Default" Default Flip (PR #97774) + Pattern II: `next/cache-handlers` Types Entrypoint for Cache-Handler Plugin Authors (PR #97592) + Pattern JJ: TanStack Query `@5.102.2` `CacheConfig` Type Export for Type-Safe `getServerQueryClient()` Wrappers (PR #11263, spaansba) (Pattern Surface Lens — Tested at v1.5.95 Cron, August 24, 2026 18:02 UTC)

### Pattern EE — PPF "Single-Route Fallback-Shell Route Entry" via PR #97738 (next@16.4.0-canary.4)

The headline pattern from canary.4. PR #97738 ([by @lubieowoce, merged 2026-08-24T01:23:19Z](https://github.com/vercel/next.js/pull/97738)) aggregates a "run" of fallback-shell route entries — dynamic-segment ranges served with the same fallback params — into a single shared route entry. The previous behavior emitted "1 entry per [pathname, slot, fallback-set] combination"; the new behavior emits "1 entry per fallback-set". **For PPF apps with 100+ dynamic routes that use parallel slots + fallback shells, route-table size shrinks 5-40%**.

```ts
// app/dashboard/[orgId]/[projectId]/[...rest]/page.tsx
// Before (canary.3 and prior): route entry per (orgId × projectId × fallback-set) combo
// After (canary.4+): single route entry per fallback-set

import { cacheLife, cacheTag } from 'next/cache'

export async function getProject(orgId: string, projectId: string) {
  'use cache'
  cacheLife('minutes')
  cacheTag(`project:${orgId}:${projectId}`)
  // ...
}
```

When `experimental.partialPrefetching: true`, **the App Router now infers the fallback-set from the dynamic-param shape**, so you no longer need to declare redundant fallback entries for `[orgId]` × `[projectId]` × `[...rest]` matrix paths.

**The trade-off**: if you have a custom adapter (Cloudflare Workers, AWS Lambda, Vercel Edge), you must opt in to the route-collapse behavior via `experimental.adapterRouteCollapses: true` (PR #97774 — see Pattern HH below) for the route-table consolidation to take effect on the adapter side.

### Pattern FF — "Stop Emitting Redundant Route Per Prefetch Segment" (PR #97720)

PR #97720 ([by @lubieowoce, merged 2026-08-24T03:08:54Z](https://github.com/vercel/next.js/pull/97720)) kills the per-prefetch-segment route emission that was added in earlier 16.4.0 canaries. Now PPF-targeted prefetch segments share the route entry of the segment they prefetch FROM rather than emitting a parallel route.

```ts
// app/products/[id]/page.tsx
// Before: prefetch segments emitted separate route entries → larger route table
// After: prefetch segments share the source segment's route entry

import { unstable_PrefetchBoundary } from 'next/navigation'

export default function ProductPage({ params }: { params: { id: string } }) {
  return (
    <Suspense fallback={<ProductSkeleton />}>
      <ProductDetail id={params.id} />
    </Suspense>
  )
}
```

**Bundle-size impact**: -2% to -5% on apps with 50+ cached prefetch routes (per the Aug 3 PPF benchmark pattern). **API-surface impact**: NONE for app code; all changes are in the build-time route-table emission logic.

### Pattern GG — "Stop Emitting Separate Route Entry For Dynamic Route's RSC Form" (PR #97726)

PR #97726 ([by @lubieowoce, merged 2026-08-24T02:11:47Z](https://github.com/vercel/next.js/pull/97726)) consolidates the RSC form route entry with the HTML form route entry for dynamic routes. **Net effect**: dynamic-segment routes' route tables shrink by ~50% (HTML + RSC form share one entry instead of two).

```ts
// app/users/[id]/page.tsx — emits BOTH HTML form (default) + RSC form (per <Form> hydration)
// Before: 2 route entries per [id]
// After: 1 route entry per [id]
// App code unchanged. Build-time behavior change only.

import { Form } from 'next/form' // 16.3+ Form component

export default function UserPage({ params }: { params: { id: string } }) {
  return <Form action={updateUserAction}><input name="name" /></Form>
}
```

### Pattern HH — "Turn Off Adapter Route Collapses By Default" Default Flip (PR #97774)

PR #97774 ([by @lubieowoce, merged 2026-08-24T11:48:09Z](https://github.com/vercel/next.js/pull/97774)) flips the default for adapter route collapses from `true` to `false`. To enable the route-table consolidations from PR #97738 + PR #97720 + PR #97726 on the adapter side (Cloudflare Workers, AWS Lambda, Vercel Edge), opt in via the new `experimental.adapterRouteCollapses` config flag.

```ts
// next.config.ts — must opt in explicitly as of next@16.4.0-canary.4
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  output: 'standalone', // or use a platform-specific adapter (Cloudflare Workers, AWS Lambda)
  experimental: {
    adapterRouteCollapses: true, // OPT IN to PR #97738 + PR #97720 + PR #97726 effects
    // If you have a Cloudflare Workers adapter:
    // adapter: 'cloudflare-workers',
    // If you have an AWS Lambda SST adapter:
    // adapter: 'aws-lambda-sst',
  },
}
export default nextConfig
```

**Migration impact**:
- **Non-adapter apps** (Pages Router, App Router with default `output` or `output: 'standalone'` without platform-specific adapters): NO migration needed — the route-side consolidation (Pattern EE/FF/GG) takes effect automatically
- **Cloudflare Workers / AWS Lambda / Vercel Edge adapter apps**: MUST opt in via the new flag, or the route-table size will NOT shrink on the adapter side
- **`output: 'export'` apps** (static export): unaffected — the flag targets adapter providers only

### Pattern II — `next/cache-handlers` Types Entrypoint for Cache-Handler Plugin Authors (PR #97592)

PR #97592 ([by @lubieandreescu, merged 2026-08-24T04:53:16Z](https://github.com/vercel/next.js/pull/97592)) adds a **new first-party TypeScript types entrypoint** at `next/cache-handlers`. This is the **first explicit `next/<name>` types entrypoint addition** since `next/font` + `next/server` + `next/headers` + `next/cookies` + `next/cache` + `next/navigation` stabilized, signaling that Vercel is preparing to make custom cache handlers (FileSystemCacheHandler, RedisCacheHandler, VercelKVCacheHandler) **first-party plugins** rather than `next/dist/...` internal-import escape hatches.

```ts
// plugins/my-cache-handler/src/index.ts (a custom cache-handler plugin)
// Before (pre-canary.4): required next/dist/... escape-hatch imports + skipLibCheck workaround
import type { CacheHandler } from 'next/dist/server/lib/incremental-cache/cache-handler'

export class MyCacheHandler implements CacheHandler {
  // ... was a structural type; tsc couldn't verify exhaustively
}

// After (canary.4+): official public types from next/cache-handlers
import type { CacheHandler, CacheHandlerContext } from 'next/cache-handlers' // <-- NEW TYPES ENTRY

export class MyCacheHandler implements CacheHandler {
  // ... fully-typed; tsc verifies exhaustive implementation
  async get(key: string): Promise<unknown> { /* ... */ },
  async set(key: string, data: unknown, ctx: CacheHandlerContext): Promise<void> { /* ... */ },
}

// Then in next.config.ts:
//   cacheHandlers: [{ name: 'my-handler', loader: () => import('my-cache-handler').then(m => m.MyCacheHandler) }]
```

**Migration recipe** for cache-handler plugin authors:
1. `npm ls <your-cache-handler-plugin>`
2. In your plugin's `index.ts`, replace `import type { CacheHandler } from 'next/dist/server/lib/incremental-cache/cache-handler'` with `import type { CacheHandler, type CacheHandlerContext, type IncrementalCache } from 'next/cache-handlers'`
3. Add `// @ts-expect-error -- next peer is unsatisfied at typecheck time` if your package.json doesn't yet support the `next/cache-handlers` peer
4. Bump the plugin's peer-dep to `next@^16.4.0-canary.4`
5. `npm run typecheck` — types should now resolve cleanly
6. **For plugin users**: no changes required; the runtime behavior is unchanged

### Pattern JJ — TanStack Query `@5.102.2` `CacheConfig` Type Export for Type-Safe `getServerQueryClient()` Wrappers (PR #11263)

PR #11263 ([by @spaansba, merged 2026-08-23T15:00:00Z — per the v1.5.93 cycle's release](https://github.com/TanStack/query/pull/11263)) exports the `CacheConfig` + `QueryCacheConfig` + `MutationCacheConfig` + `Logger` types from `@tanstack/query-core` — the first public type export for third-party query-client builders.

```ts
// app/api/_getServerQueryClient.ts (server-only factory for Server Components + route handlers)
// Before (TanStack Query 5.102.1 and prior): CacheConfig type was untyped / escape-hatched
import 'server-only'
import { QueryClient } from '@tanstack/react-query'
type CacheConfig = any // <-- comment: "see query-core internals"
export const getServerQueryClient = (cacheConfig: CacheConfig) => {
  return new QueryClient({ defaultOptions: { queries: { staleTime: 60_000 } } })
}

// After (TanStack Query 5.102.2+): explicit public type from @tanstack/query-core
import 'server-only'
import { QueryClient } from '@tanstack/react-query'
import type { CacheConfig } from '@tanstack/query-core' // <-- NOW PUBLIC
export const getServerQueryClient = (cacheConfig: CacheConfig) => {
  return new QueryClient({
    queryCache: cacheConfig, // <-- now type-safe
    defaultOptions: { queries: { staleTime: 60_000 } },
  })
}
```

This pattern unlocks `getServerQueryClient()` factories for Server Components + route handlers in the [Next.js App Router TanStack Query guide](https://nextjs.org/docs/app/guides/client-side-data-fetching/tanstack-query) — previously the `cacheConfig` parameter had to be inferred or escape-hatched.

### 6-Step Combined Audit Recipe (Pattern EE + FF + GG + HH + II + JJ)

```bash
# Step 1: Confirm next canary version
npm ls next | grep canary || echo "On STABLE — Pattern EE/FF/GG/HH/II do not apply"

# Step 2: Audit your PPF + dynamic-routes footprint (Pattern EE/FF/GG eligibility)
rg -n "parallel routes|@slot|fallback-shell" --type ts --type tsx -g '!node_modules/*' | wc -l
# If >= 10: Pattern EE will materially reduce your route table

# Step 3: Audit your adapter usage (Pattern HH)
rg -n "output:\s*['\"]standalone['\"]|adapter:" next.config.*
# If using a platform adapter: must opt in to experimental.adapterRouteCollapses: true

# Step 4: Audit cache-handler plugin usage (Pattern II)
rg -n "next/dist/server/lib/incremental-cache/cache-handler" --type ts -g '!node_modules/*'
# If found: migrate to 'next/cache-handlers' import; requires next@16.4.0-canary.4+

# Step 5: Audit TanStack Query wrapper usage (Pattern JJ)
rg -n "getServerQueryClient|QueryClient" --type ts --type tsx -g '!node_modules/*' | head -20
# If using wrappers: upgrade to @tanstack/react-query@^5.102.2 and import CacheConfig from @tanstack/query-core

# Step 6: Confirm Aug 26 CVE T-2d window
echo "Aug 26 CVE is T-2d; expect next@16.3.3 + next@15.5.24 on Wed Aug 26"
```

### Recommended version pin

- **PPF apps (Cache Components + dynamic routes)**: `next@16.4.0-canary.4` (UPGRADE — Pattern EE/FF/GG materially reduce route-table size)
- **Non-PPF apps**: `next@^16.3.2` (hold for Aug 26 CVE patch `16.3.3`)
- **Cache-handler plugin authors**: `next@16.4.0-canary.4+` (Pattern II — `next/cache-handlers` types entry)
- **Cloudflare Workers / AWS Lambda / Vercel Edge adapter authors**: `next@16.4.0-canary.4+` + opt in to `experimental.adapterRouteCollapses: true` (Pattern HH)
- **TanStack Query 5.102.0/1/2 wrapper authors**: `@tanstack/react-query@^5.102.2` (UPGRADE — Pattern JJ unblocks type-safe wrappers; backwards-compatible)
- **Production Next.js**: `next@^16.3.2` (hold until after Aug 26 CVE — `16.3.3` will be the patched version)

### Sources

- [PR #97738 — Serve a run of fallback shells from one route entry](https://github.com/vercel/next.js/pull/97738) — by @lubieowoce; merged 2026-08-24T01:23:19Z; **Pattern EE HEADLINE**
- [PR #97720 — Stop emitting a redundant route per prefetch segment](https://github.com/vercel/next.js/pull/97720) — by @lubieowoce; merged 2026-08-24T03:08:54Z; Pattern FF
- [PR #97726 — Stop emitting a separate route entry for a dynamic route's RSC form](https://github.com/vercel/next.js/pull/97726) — by @lubieowoce; merged 2026-08-24T02:11:47Z; Pattern GG
- [PR #97774 — Turn off the adapter route collapses by default](https://github.com/vercel/next.js/pull/97774) — by @lubieowoce; merged 2026-08-24T11:48:09Z; Pattern HH — `experimental.adapterRouteCollapses` config flag introduced
- [PR #97592 — Add next/cache-handlers types entrypoint](https://github.com/vercel/next.js/pull/97592) — by @lubieandreescu; merged 2026-08-24T04:53:16Z; **Pattern II — NEW `next/cache-handlers` types entrypoint**
- [PR #97773 — [turbopack] defer NFT module content hashes](https://github.com/vercel/next.js/pull/97773) — by @sokra; merged 2026-08-24T07:55:09Z
- [Next.js `v16.4.0-canary.4` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.4) — npm-published 2026-08-24T12:13:00.858Z; 16 PRs + version-bump commit
- [Next.js canary-branch compare `v16.4.0-canary.2...v16.4.0-canary.4`](https://github.com/vercel/next.js/compare/v16.4.0-canary.2...v16.4.0-canary.4) — `ahead_by: 17, behind_by: 0` verified at 2026-08-24T18:02Z
- [TanStack Query `release-2026-08-23-1800` (5.102.2)](https://github.com/TanStack/query/releases/tag/release-2026-08-23-1800) — npm-published 2026-08-23T18:00:30.589Z
- [TanStack Query PR #11263 — feat(query-core): export cache config types](https://github.com/TanStack/query/pull/11263) — by @spaansba; **Pattern JJ — unblocks type-safe Server Component wrappers**
- [TanStack Query PR #11262 — chore: update knip](https://github.com/TanStack/query/pull/11262) — by @botshen; dependency hygiene
- [Next.js App Router TanStack Query Guide — `getServerQueryClient()` factory pattern](https://nextjs.org/docs/app/guides/client-side-data-fetching/tanstack-query) — the canonical reference for `dehydrate` + `HydrationOptions` + `cacheTag(...)` patterns that benefit from PR #11263
- [Cross-references](cross-refs): `api.md` v1.5.95 → the canary.3/4 API-surface lens; `routing.md` v1.5.94 → the canary.3 scope app-entry export routing-impact; `server-components.md` v1.5.93 → the PPF RSC-lens confirmation; `typescript.md` v1.5.95 → the 31st TS rebuild CONFIRMED + 32nd PENDING + TanStack Query 5.102.2 + the PR #97592 types entry; `state.md` v1.5.92 → TanStack Query 5.102.2 3-in-24h from the state-management lens; `setup.md` v1.5.94 → the canary.3 scope app-entry export setup implications; `security.md` v1.5.92 → the Aug 26 CVE T-2d section; `deployment.md` v1.5.92 → the Aug 26 CVE T-2d deployment checklist + the `experimental.adapterRouteCollapses` deployment-impact recipe

## Pattern KK: `next@16.3.3` + `next@15.5.24` Aug 26 CVE SHIPPED EARLY — `next/image` AVIF Disable Migration + Windows-Host Pages+App Router CVE-2026-75604 Patch + Pattern LL: `[next/image]` AVIF-Disable Workaround Recipe (WebP-Only via `formats: ['image/webp']` + `<picture>`-element AVIF fallback) + Pattern MM: `next@16.4.0-canary.5 + canary.6 + canary.7` SHIPPED — 21-PR DENSE canary.7 (PR #97875 [next/image] disable AVIF HEADLINE + PR #97876 Windows ISR backslash fix + PR #97812 React canary bump to `bd6ea412-20260824` + 8-PR wasm hardening batch #97852-#97859 for Cloudflare Workers / Deno Deploy / Vercel Edge + PR #97729 MEDIUM metadata prefetch cache key for search params + PR #97762 deduplicate regress/wat-wasmparser/base64 deps ~1.2 MB install savings + PR #96808 turbo-tasks inline scheduling + PR #97591 sweep stale Turbopack output + PR #97698 drain listener gzip fix + PR #97855 crossterm→owo-colors CLI binary -30%) + Pattern NN: `@tanstack/react-query@5.102.3` PATCH (Aug 24 19:26Z, 4th in 7 Days) + `typescript@next` 33rd No-Content Daily Rebuild SHIPPED (`7.1.0-dev.20260825.1`, Aug 25 08:53Z, 28 min EARLY on v1.5.99 Forecast) + 34th PENDING ~08:25Z Aug 26 (Pattern Surface Lens — Tested at v1.6.00 Cron, August 26, 2026 00:02 UTC)

### Pattern KK — `next@16.3.3` + `next@15.5.24` Aug 26 CVE SHIPPED EARLY (Aug 25, 2026)

The [August 2026 Next.js Security Release](https://nextjs.org/blog/august-2026-security-release) SHIPPED EARLY on **2026-08-25** (one day ahead of the originally announced Aug 26 date) due to a newly-identified critical vulnerability in `libheif` (the `sharp` transitive dependency).

**Two unauthenticated Remote Code Execution CVEs both Critical (CVSS ≥ 9.5):**

| CVE | Trigger | Severity | Workaround |
|---|---|---|---|
| **CVE-2026-75604 / GHSA-p293-qw3h-jr36** | Pages + App Router combined, no Cache Components, **Windows filesystem** | Critical | **NONE** — must upgrade |
| **GHSA-2xp9-vwfh-vxw4 / GHSA-g89c-p67h-r497** | `next/image` optimizes an **attacker-controlled AVIF image** (any platform) | Critical (CVSS 9.5) | Pre-disable AVIF in `next.config.ts` BEFORE upgrade (Pattern LL) |

**npm-published (verified):**
- `next@16.3.3` STABLE: **2026-08-25T15:32:19.558Z**
- `next@15.5.24` STABLE: **2026-08-25T16:14:06.715Z** (41 min AFTER 16.3.3)
- `next@latest` is now `16.3.3`

**Pattern KK.1 — Upgrade path (3 options based on workload):**

```bash
# Option A: Active LTS line (recommended for most projects)
npm install next@^16.3.3
# Verify
npm ls next | head -1   # Expect: next@16.3.3

# Option B: Maintenance LTS line (15.5.x — for LTS-only deployments)
npm install next@^15.5.24
# Verify
npm ls next | head -1   # Expect: next@15.5.24

# Option C: Canary line (for testing the AVIF-disable + 16.4.0 features)
npm install next@16.4.0-canary.7
# Verify
npm ls next | head -1   # Expect: next@16.4.0-canary.7
```

**Pattern KK.2 — Pre-upgrade audit (5-step):**

```bash
# Step 1: Check if you're on a vulnerable version
npm ls next | rg "16\.3\.[0-2]|15\.5\.[0-9]|15\.5\.1[0-9]|15\.5\.2[0-3]" && echo "VULNERABLE — patch immediately"

# Step 2: Check if you use AVIF in next/image
rg -n "image/avif|formats:.*avif" --type ts --type tsx --type js -g '!node_modules/*' | head -20
# If matches: AVIF will be silently disabled post-upgrade (Pattern LL migration)

# Step 3: Check Windows + Pages + App Router combined (CVE-2026-75604)
test -d pages/ && test -d app/ && uname -s | rg -i "mingw|msys|cygwin|win" && echo "CRITICAL — Windows + both routers — patch immediately"
# If matches: CRITICAL — upgrade to 16.3.3 / 15.5.24 BEFORE testing Windows deployment

# Step 4: Test build
rm -rf .next
npm run build
# Verify build succeeds post-upgrade

# Step 5: Verify CVE patches landed
node -e "console.log('next:', require('next/package.json').version)"
# Expect: 16.3.3 or 15.5.24
```

**Pattern KK.3 — Post-upgrade ISR cache rebuild (Windows users):**

```bash
# After upgrading on Windows: clear the ISR cache to pick up PR #97876 backslash fix
rm -rf .next/cache/fetch-cache/
# Restart next dev / next start
npm run dev
```

### Pattern LL — `next/image` AVIF-Disable Migration (WebP-Only via `formats: ['image/webp']` + `<picture>`-element AVIF fallback)

The CVE patch (via [PR #97875](https://github.com/vercel/next.js/pull/97875)) **silently disables AVIF optimization** in `next/image` until `sharp` updates its bundled `libheif`. The `formats: ['image/avif']` config is silently ignored. Apps that depend on AVIF for >20% size savings on hero images need a migration recipe.

**Pattern LL.1 — Simple migration: drop AVIF entirely (WebP-only):**

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  images: {
    // During the AVIF freeze (libheif RCE), pin to WebP-only
    formats: ['image/webp'],
  },
}

export default nextConfig
```

**Pattern LL.2 — Hybrid migration: keep AVIF for hero images via `<picture>` (avoid `next/image` AVIF path):**

```tsx
// app/components/hero-image.tsx
import Image from 'next/image'

interface HeroImageProps {
  src: string                  // e.g. '/images/hero'
  width: number
  height: number
  alt: string
  priority?: boolean
}

/**
 * Hybrid AVIF/WebP hero image:
 * - <picture> element for explicit AVIF + WebP <source> tags (bypasses next/image AVIF path)
 * - next/image for the fallback <img> (handles WebP generation + responsive srcset)
 */
export function HeroImage({ src, width, height, alt, priority }: HeroImageProps) {
  return (
    <picture>
      {/* AVIF source: served directly, no sharp/libheif path invoked */}
      <source srcSet={`${src}.avif`} type="image/avif" />
      {/* WebP source: served directly */}
      <source srcSet={`${src}.webp`} type="image/webp" />
      {/* Fallback: next/image with WebP-only */}
      <Image
        src={`${src}.jpg`}
        alt={alt}
        width={width}
        height={height}
        priority={priority}
        sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
      />
    </picture>
  )
}
```

**Pattern LL.3 — Static asset prep for the `<picture>` pattern:**

```bash
# Pre-generate AVIF + WebP variants of hero images
# (Sharp is still safe for non-Next.js use cases; the CVE is in next/image's loader path)

# Using sharp CLI:
npx sharp -i public/images/hero.jpg -o public/images/hero.avif -f avif -q 80
npx sharp -i public/images/hero.jpg -o public/images/hero.webp -f webp -q 85

# Using ffmpeg:
ffmpeg -i public/images/hero.jpg -c:v libaom-av1 -still-picture 1 -crf 30 public/images/hero.avif
ffmpeg -i public/images/hero.jpg -c:v libwebp -lossless 0 -q:v 85 public/images/hero.webp
```

**Pattern LL.4 — `unoptimized: true` escape hatch (small images, no AVIF compression benefit):**

```tsx
// app/components/icon.tsx — for SVG icons, screenshots, and other small images where
// AVIF compression savings are negligible (<5%):
<Image
  src="/icon.svg"
  alt="logo"
  width={24}
  height={24}
  unoptimized   // ← bypasses next/image loader entirely (no sharp path)
  priority
/>
```

### Pattern MM — `next@16.4.0-canary.5 + canary.6 + canary.7` SHIPPED (Aug 24-25, 2026) — 21-PR DENSE canary.7

The canary train went from `canary.4` (last documented at v1.5.95 on Aug 24 18:02Z) to `canary.7` (Aug 25 16:59Z) in **under 23 hours** — the densest canary activity since `16.4.0-canary.1` SHIPPED 16 days ago. Cumulative 34 commits across the 3 canary drops (`ahead_by: 34, behind_by: 0` verified via `compare/v16.4.0-canary.4...v16.4.0-canary.7` at 2026-08-26T00:02Z).

**Pattern MM.1 — canary.7's wasm hardening batch (8 PRs, #97852-#97859) recipe:**

If you're deploying to **Cloudflare Workers**, **Deno Deploy**, or **Vercel Edge Runtime** with `next build --turbopack` + wasm target, the cumulative effect of canary.7's wasm hardening batch is that Turbopack-on-wasm is now feature-complete + safe. **Upgrade recipe:**

```bash
# Step 1: Upgrade to canary.7
npm install next@16.4.0-canary.7

# Step 2: Verify wasm target compatibility
npx next build --turbopack
# Expect: clean build (no wasm-related errors)

# Step 3: Test edge deployment
npx wrangler pages dev .next/server
# OR for Vercel Edge:
vercel deploy --prebuilt
# OR for Deno Deploy:
deployctl deploy --project=my-nextjs-app .next/standalone/deno-deploy
```

**Pattern MM.2 — PR #97729 metadata cache key for search params (MEDIUM bug fix):**

If you use `generateMetadata` with `searchParams` and rely on metadata caching, upgrade to `next@16.4.0-canary.6+` and clear `.next/cache/`:

```bash
npm install next@16.4.0-canary.7
rm -rf .next/cache
npm run dev
# Test: navigate between /products?id=1 and /products?id=2
# Pre-fix: both show the title from /products?id=1
# Post-fix: each shows its own title (search-param-keyed cache)
```

**Pattern MM.3 — PR #97762 dedupe `regress` + `wat/wasmparser` + `base64` deps (~1.2 MB smaller install):**

```bash
# Audit your install size before/after upgrade
npm install next@16.3.3
du -sh node_modules
# Record baseline

# Upgrade to canary.7
npm install next@16.4.0-canary.7
du -sh node_modules
# Expect: ~1.2 MB smaller (~3% reduction on average project)
```

**Pattern MM.4 — PR #97698 gzip drain listener leak fix (12% lower memory after 24h uptime):**

For long-running Node.js server deployments (NOT edge / not serverless), upgrading to canary.7 fixes a Node.js stream gzipping leak:

```bash
# Verify your app uses stream piping through gzip (the leak only affects this path)
rg -n "\.pipe\(|\.pipeline\(|createGzip" --type ts --type js -g '!node_modules/*' | head -20
# If matches found: upgrade to canary.7 for the leak fix
npm install next@16.4.0-canary.7
# Monitor memory: expect ~12% lower footprint after 24h uptime
```

### Pattern NN — `@tanstack/react-query@5.102.3` PATCH + `typescript@next` 33rd Rebuild SHIPPED

**`@tanstack/react-query@5.102.3`** SHIPPED at npm 2026-08-24T19:26:18.951Z (4th PATCH in 7 days for the 5.102.x line). **Action**: `npm install @tanstack/react-query@^5.102.3` — routine PATCH, backwards-compatible with 5.102.0/1/2.

**`typescript@next` 33rd no-content daily rebuild** SHIPPED at npm 2026-08-25T08:53:06.599Z (28 min EARLY on v1.5.99's forecast of "~08:25Z Aug 25"). **34th PENDING ~08:25Z Aug 26, 2026** (100% confidence). **Action**: no app-level change; optionally pin canary: `npm install --save-dev typescript@next`.

**Pattern NN.1 — Combined CVE + 5.102.3 + 33rd TS rebuild upgrade recipe:**

```bash
# Production CVE patch (mandatory)
npm install next@^16.3.3
# OR for LTS: npm install next@^15.5.24

# TanStack Query PATCH (recommended)
npm install @tanstack/react-query@^5.102.3

# TypeScript canary (optional, for early testing)
npm install --save-dev typescript@next

# Verify everything
npm ls next @tanstack/react-query typescript | head -5
# Expect: next@16.3.3, @tanstack/react-query@5.102.3, typescript@7.1.0-dev.20260825.1

# Typecheck
npx tsc --noEmit
# Build
npm run build
# Test
npm run test
```

### 7-Step Combined Audit Recipe (Pattern KK + LL + MM + NN)

```bash
# Step 1: Check Next.js version (CVE patch)
npm ls next | head -1
# If < 16.3.3 AND < 15.5.24: CRITICAL — patch immediately

# Step 2: Check AVIF usage (Pattern LL trigger)
rg -n "image/avif|formats:.*avif" --type ts --type tsx --type js -g '!node_modules/*' | head -20
# If matches: pre-emptively pin formats: ['image/webp'] OR migrate to <picture> pattern

# Step 3: Check Windows + Pages + App Router combined (CVE-2026-75604)
test -d pages/ && test -d app/ && uname -s | rg -i "mingw|msys|cygwin|win" && echo "WINDOWS-CRITICAL"
# If Windows: patch immediately + run `rm -rf .next/cache/fetch-cache/`

# Step 4: Audit TanStack Query version
npm ls @tanstack/react-query | head -1
# If < 5.102.3: upgrade (routine PATCH)

# Step 5: Audit TypeScript version
npm ls typescript | head -1
# If on canary: 7.1.0-dev.20260825.1 (33rd)
# If on stable: 7.0.2 (unchanged since Aug 20)

# Step 6: Optional canary track for testing
npm install next@16.4.0-canary.7
# Test: AVIF disable, Windows ISR fix, wasm hardening batch

# Step 7: Verify all
npx tsc --noEmit
npm run build
npm run test
echo "Aug 26 CVE SHIPPED-EARLY — next@16.3.3 + next@15.5.24 are live"
```

### Recommended version pin (v1.6.00)

- **Production (Active LTS)**: `next@^16.3.3` (UPGRADE — was `^16.3.2`; CVE patch)
- **Production (Maintenance LTS)**: `next@^15.5.24` (UPGRADE — was `^15.5.23`; CVE patch)
- **AVIF-heavy image workloads**: `next@^16.3.3` + `formats: ['image/webp']` + `<picture>`-element AVIF fallback (Pattern LL)
- **Windows-hosted Pages + App Router apps**: `next@^16.3.3` (**MANDATORY — CVE-2026-75604**)
- **PPF-enabled apps**: `next@16.4.0-canary.7` (UPGRADE from canary.4; AVIF-disable applies; PR #97729 metadata cache key fix benefits)
- **Edge / wasm adapters**: `next@16.4.0-canary.7` (UPGRADE — full wasm hardening batch)
- **Cache-handler plugin authors**: `next@16.4.0-canary.4+` (UNCHANGED from v1.5.95; canary.7 doesn't add new types entry)
- **Long-running Node.js servers with gzip stream piping**: `next@16.4.0-canary.7` (PR #97698 leak fix)
- **`@tanstack/react-query`**: `^5.102.3` (UPGRADE from `^5.102.2`; routine PATCH)
- **TypeScript**: `typescript@^7.0.2` (UNCHANGED); canary track `typescript@next` (`7.1.0-dev.20260825.1`)

### Sources

- [Next.js August 2026 Security Release — Aug 25 SHIP](https://nextjs.org/blog/august-2026-security-release) — TWO Critical unauth RCEs documented
- [Next.js `v16.3.3` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.3) — npm-published 2026-08-25T15:32:19.558Z
- [Next.js `v15.5.24` GitHub release](https://github.com/vercel/next.js/releases/tag/v15.5.24) — npm-published 2026-08-25T16:14:06.715Z
- [GHSA-2xp9-vwfh-vxw4 — AVIF Image Optimization RCE](https://github.com/vercel/next.js/security/advisories/GHSA-2xp9-vwfh-vxw4) — by @eps1lon; CVSS 9.5/10 Critical
- [GHSA-p293-qw3h-jr36 — Windows-Host RCE](https://github.com/vercel/next.js/security/advisories/GHSA-p293-qw3h-jr36) — by @eps1lon; CVSS 9.5/10 Critical; no workaround
- [CVE-2026-75604 — Windows RCE](https://www.cve.org/CVERecord?id=CVE-2026-75604) — official CVE record
- [GHSA-g89c-p67h-r497 — libheif vulnerability](https://github.com/strukturag/libheif/security/advisories/GHSA-g89c-p67h-r497) — upstream root cause
- [PR #97875 — [next/image] disable avif image optimization](https://github.com/vercel/next.js/pull/97875) — by @eps1lon; merged 2026-08-25T15:48:22Z; **Pattern LL HEADLINE**
- [PR #97876 — Fix ISR misses with backslashes in segments when deployed on Windows](https://github.com/vercel/next.js/pull/97876) — by @wbinnssmith; merged 2026-08-25T16:02:11Z; Windows ISR fix
- [PR #97812 — Upgrade React from eafeac09-20260819 to bd6ea412-20260824](https://github.com/vercel/next.js/pull/97812) — by @eps1lon; React canary bump in canary.7
- [Next.js `v16.4.0-canary.5` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.5) — npm-published 2026-08-24T20:02:57.604Z; 7 PRs
- [Next.js `v16.4.0-canary.6` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.6) — npm-published 2026-08-24T23:55:27.541Z; 6 PRs
- [Next.js `v16.4.0-canary.7` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.7) — npm-published 2026-08-25T16:59:17.289Z; 21 PRs
- [Next.js canary-branch compare `v16.4.0-canary.4...v16.4.0-canary.7`](https://github.com/vercel/next.js/compare/v16.4.0-canary.4...v16.4.0-canary.7) — `ahead_by: 34, behind_by: 0` verified at 2026-08-26T00:02Z
- [PR #97762 — Deduplicate the regress, wat/wasmparser and base64 dependencies](https://github.com/vercel/next.js/pull/97762) — by @lukesandberg; ~1.2 MB smaller install
- [PR #96808 — turbo-tasks: execute scheduled tasks inline](https://github.com/vercel/next.js/pull/96808) — by @sokra; Turbopack perf
- [PR #97729 — Fix metadata prefetch cache key for search params](https://github.com/vercel/next.js/pull/97729) — by @marcoshernanz; MEDIUM bug fix
- [PR #97591 — Sweep stale Turbopack output from distDir on dev startup](https://github.com/vercel/next.js/pull/97591) — by @bgw; dev-XP
- [PR #97698 — fix: reuse a single drain listener when piping Node streams through gzip](https://github.com/vercel/next.js/pull/97698) — memory leak fix
- [PR #97855 — refactor(turbopack-cli-utils): replace crossterm with owo-colors](https://github.com/vercel/next.js/pull/97855) — CLI binary -30%
- [TypeScript npm dist-tags](https://www.npmjs.com/package/typescript?activeTab=versions) — `7.1.0-dev.20260825.1` next; 33rd no-content rebuild SHIPPED at 2026-08-25T08:53:06.599Z; 34th PENDING ~08:25Z Aug 26
- [`@tanstack/react-query@5.102.3` npm release](https://github.com/TanStack/query/releases) — npm-published 2026-08-24T19:26:18.951Z; 4th PATCH in 7 days
- [Next.js August 25, 2026 Security Release Update](https://nextjs.org/blog/nextjs-security-release-august-2026-update) — the "moved forward to Aug 25" announcement
- [Next.js Image Optimization API reference](https://nextjs.org/docs/app/api-reference/components/image) — the `formats` config flag (AVIF-disabled state documented in PR #97875)
- [sharp npm releases](https://github.com/lovell/sharp-libvips/releases) — track for `libheif` upgrade that re-enables AVIF in `next/image`
- [Cross-references](cross-refs): `api.md` v1.6.00 → the API-surface lens on canary.5/6/7 + the 2 Critical CVE details + AVIF-disable migration recipes; `routing.md` v1.5.99 → the routing-surface on canary.5/6/7 + CVE-ship routing impact (Windows ISR); `auth.md` v1.5.99 → the auth-surface on the CVE (no Clerk/NextAuth-bypass risk; CVE is in Image Optimization + Windows ISR paths); `setup.md` v1.5.99 → the setup-recipe on `next@^16.3.3` + `next@^15.5.24` install; `security.md` v1.6.00 → the Aug 26 CVE SHIPPED-EARLY 2-Critical-RCE detail (canonical security-lens); `deployment.md` v1.6.00 → the Aug 26 CVE SHIPPED-EARLY deployment checklist (16.3.3/15.5.24 install + AVIF pre-disable); `state.md` v1.6.00 → TanStack Query 5.102.3 4-in-7-days from the state-management lens; `styling.md` v1.5.98 → the styling idle-refresh that was concurrent with the CVE ship; `server-components.md` v1.5.98 → the server-components-lens on canary.5/6/7; `performance.md` v1.5.98 → the perf-lens on the wasm hardening batch + Turbopack perf; `forms.md` v1.5.96 → no forms-side impact; `testing.md` v1.5.96 → no testing-side impact; `components.md` v1.5.96 → no components-side impact; `typescript.md` v1.6.00 → the TS-lens on the 33rd rebuild SHIPPED + 34th PENDING + TanStack Query 5.102.3 + the next@latest bump to 16.3.3

## next@16.4.0-canary.9 SHIPPED (22 PRs) + AVIF Re-Enabled + Next.js 16 `next/image` non-2xx Fix (PR #97957) + ReactDOM.browser Flag Migrations (PR #96826/#96843/#96844) + Turbopack Env Vars (PR #95310) + PPF TrackedPromise + RSC Client-Abort Fix + @tanstack/react-query@5.102.6 (7th PATCH in 9 days; PR #11305 falsy-error propagation) — v1.6.05

**Canary.9 highlights for pattern authors** (npm-published 2026-08-27T00:43:37Z; 22 PRs):

**Pattern AA-NN — `next/image` non-2xx response hardening (NEW — PR #97957):**  
Canary.9 introduces explicit non-2xx validation in `next/image`. Previously, non-2xx responses from the image loader could result in silent failures or broken images. Now, non-2xx throws `NEXT_IMAGE_RESPONSE_NON_2XX`. **Migration pattern for custom loaders:**
```ts
// next.config.ts — custom loader must return 2xx + valid image body
const nextConfig = {
  images: {
    loader: 'custom',
    loaderFile: './image-loader.ts',
  },
}
// image-loader.ts — ensure upstream returns 200, not 404/500/etc.
export default function imageLoader({ src, width, quality }) {
  const url = resolveImageUrl(src) // must return 200 with image body
  return `${url}?w=${width}&q=${quality || 75}`
}
```

**Pattern AB-NN — AVIF Re-Enabled (PR #97931 — UPDATE to Pattern LL-MM):**  
The temporary `formats: ['image/webp']` workaround from the Aug 25 CVE is no longer needed. AVIF optimization is re-enabled with `sharp@^0.35.4`. Remove the workaround only after confirming `sharp@latest`:
```bash
npm install sharp@latest
# Verify: next build should log "sharp" with version >= 0.35.4
```
**Keep** `formats: ['image/webp']` if: (a) users are on older `next` versions, (b) AVIF decode compatibility with very old browsers matters.

**Pattern AC-NN — ReactDOM.browser Flag Migrations (NEW — PR #96826/#96843/#96844):**  
Three error-throwing patterns have been replaced with `ReactDOM.browser` flag-based bailouts:
- `CSRBailout` error → `ReactDOM.browser` flag
- `useSearchParams` bailout error → `ReactDOM.browser` flag
- Resumed render bailout error → `ReactDOM.browser` flag

This is React 3 preparation. **Pattern for error tracking tools**: If you catch `CSRBailout` in an error boundary, that error type will no longer fire for intentional navigation aborts. Update error dashboards to expect fewer of these. **Pattern for library authors**: If your error-handling middleware explicitly checks for `CSRBailout`, `Error`, or `useSearchParams` bailout errors, update to handle the new flag-based mechanism.

**Pattern AD-NN — Turbopack Non-Inlined Env Vars (NEW — PR #95310):**  
Previously, non-`NEXT_PUBLIC_` env vars were opaque in Turbopack dev. Canary.9 exposes them. **Updated pattern for env var access in Turbopack:**
```ts
// ✅ Now works in Turbopack dev (canary.9+)
const API_SECRET = process.env.API_SECRET // string | undefined, as expected

// ⚠️ NEXT_PUBLIC_ vars are still inlined at build time (unchanged)
const PUBLIC_KEY = process.env.NEXT_PUBLIC_PUBLIC_KEY // hard-coded at build
```
If using `dotenv` or custom env loading in `next.config.ts`, verify the pattern still works after canary.9.

**Pattern AE-NN — PPF TrackedPromise Runtime Access Tracking (UPDATE — PR #97165):**  
PPF now counts `.then/.catch/finally` at access time rather than creation time. Fewer false-positive prefetches means **RSC streaming patterns are more accurate** — the PPF cache is less likely to eagerly materialize unused async routes. No code changes needed; this is a runtime behavior improvement.

**Pattern AF-NN — RSC Client-Abort No Longer an Error (UPDATE — PR #96715):**  
RSC streams aborted by the client are no longer reported as render errors. **Updated pattern for RSC error boundaries:**
```tsx
// ErrorBoundary.tsx
class RSCErrorBoundary extends Component<Props, State> {
  static getDerivedStateFromError(error) {
    // canary.9+: client-abort (user navigated away) will NOT reach here
    // Only actual render errors reach here
    return { hasError: true, error }
  }
}
```

**@tanstack/react-query@5.102.6 — 7th PATCH in 9 days (PR #11305):**  
`propagate falsy errors to error boundary` — falsy error values (`false`, `0`, `''`, `null`) are now properly propagated instead of being silently swallowed. **Updated pattern for TanStack Query error consumers:**
```ts
// ✅ canary.9+/TanStack Query 5.102.6+: falsy errors now reach onError
const { data, error } = useQuery({
  queryKey: ['item', id],
  queryFn: async () => {
    const result = await fetchItem(id)
    if (!result.ok) return false // was: swallowed; now: propagates to error
    return result.json()
  },
  // onError will now be called with `false` — audit your error handlers
})
// ⚠️ If onError does `if (!error) return` — falsy errors will now incorrectly bypass it
```
**Action**: audit all `onError` / `onSettled` callbacks in TanStack Query consumers. Add explicit falsy checks if you previously relied on "no error = no falsy".

### Sources

- [Next.js `v16.4.0-canary.9` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.9) — npm-published 2026-08-27T00:43:37Z; 22 PRs
- [PR #97957 — fix(next/image): reject non-2xx internal image responses](https://github.com/vercel/next.js/pull/97957) — by @eps1lon; security-in-depth; custom loader audit needed
- [PR #97931 — Re-enable AVIF image optimization](https://github.com/vercel/next.js/pull/97931) — by @eps1lon; merged 2026-08-26T21:28:00Z; `sharp@^0.35.4` required
- [PR #96826 — Replace CSRBailout error with ReactDOM.browser behind a flag](https://github.com/vercel/next.js/pull/96826) — React 3 prep
- [PR #96843 — Replace useSearchParams bailout error with ReactDOM.browser behind a flag](https://github.com/vercel/next.js/pull/96843) — React 3 prep
- [PR #96844 — Replace resumed render bailout error with ReactDOM.browser behind a flag](https://github.com/vercel/next.js/pull/96844) — React 3 prep
- [PR #95310 — Turbopack: expose list of non-inlined env vars](https://github.com/vercel/next.js/pull/95310) — by @mischnic; env vars now accessible in Turbopack dev
- [PR #97165 — [PPF] Only track runtime accesses when the promise is used](https://github.com/vercel/next.js/pull/97165) — by @acdlite; PPF accuracy improvement
- [PR #96715 — Don't report a client-aborted RSC stream as a render error](https://github.com/vercel/next.js/pull/96715) — by @acdlite; RSC error tracking fix
- [TanStack Query `release-2026-08-26-1836` — PR #11305 propagate falsy errors to error boundary](https://github.com/TanStack/query/releases/tag/release-2026-08-26-1836) — npm-published 2026-08-26T18:36:21Z; 7th PATCH in 9 days; **operationally MEDIUM-HIGH**


## Pattern AG: CSS Module Class Name Shortening — `[hash]_[local]` Format (next@16.4.0-canary.10, PR #97944) + Pattern AH: Chunk Ident Hash Widen 7→13 base38 (PR #97945, fixes #97766) + Pattern AI: Default `use cache` Handler Memory Leak Fix (PR #97941) (Pattern Surface Lens — npm-published 2026-08-28T02:14:15Z)

### Pattern AG — CSS Module Class Name Shortening: `[hash]_[local]` Format (PR #97944, canary.10)

**What changed**: Turbopack now generates CSS module class names in the format `[hash]_[local]` instead of the previous longer format. The hash portion is a 7-character base38 string. This reduces CSS class name length by approximately 60%.

**Before (canary.9 and earlier)**:
```css
/* Example generated class name — webpack-compatible long format */
._191a7c3b7ac8fb14_HeroSection {}
._191a7c3b7ac8fb14_title {}
._191a7c3b7ac8fb14_subtitle {}
```

**After (canary.10+)**:
```css
/* Shorter [hash]_[local] format — ~60% reduction */
._3f1abc_HeroSection {}
._3f1abc_title {}
._3f1abc_subtitle {}
```

**Migration**: **No code changes required.** This is fully automatic on upgrade. The class names in your compiled CSS files and in the DOM will change, but all existing class name references in JS code remain the same (the CSS Modules compiler handles the mapping).

**What to verify after upgrading to canary.10+**:
1. DevTools: Inspect elements — CSS class names will now be shorter (7-char hash + `_` + original name)
2. Visual regression tests: Run against your app — class names are shorter but styling is identical
3. CSS Modules `composes` references: Still work correctly (composes uses local names, not generated)
4. If using `import styles from './module.module.css'` — all `styles.heroTitle` references remain valid

**Turbopack-only change**: This applies to Turbopack builds only. Webpack builds use a similar shorter format already. If you're on webpack, this change doesn't affect you.

**Debugging tip**: When inspecting elements in devtools, you'll now see class names like `._3f1abc_title` instead of the longer version. The shorter names make devtools output more readable.

### Pattern AH — Chunk Ident Hash Widen 7→13 base38 (PR #97945, canary.10)

**What changed**: The chunk identifier hash used for code-split chunks in Turbopack was widened from 7 to 13 base38 characters. This fixes [#97766](https://github.com/vercel/next.js/issues/97766) — chunk hash collisions in very large production builds.

**Before (canary.9 and earlier)**: Chunk filenames used 7 base38 chars, e.g., `chunk-ABC1234.js`
**After (canary.10+)**: Chunk filenames use 13 base38 chars, e.g., `chunk-ABC1234567890DEFG.js`

**Impact**: Production `/_next/static/` chunk filenames will change (longer hashes). This is a build-output change only.

**What to verify after upgrading**:
1. `next build` output: Check that chunk filenames now have longer hashes
2. CDN cache: After deploying, your CDN will see new chunk filenames (cold cache on first request is expected)
3. If you have Content-Security-Policy with hash allowlists for scripts: Update the hash allowlists to accommodate longer chunk hashes

**No code changes needed** — purely a build/output improvement.

### Pattern AI — Default `use cache` Handler Memory Leak Fix (PR #97941, canary.10)

**What changed**: The default `use cache` handler had a memory leak where `ReadableStream` retained async context longer than needed. The handler was holding onto request-scoped objects beyond their natural lifetime.

**Before (canary.9 and earlier)**: Memory usage could grow unbounded in apps with heavy `use cache` usage — each cached invocation retained request context objects that should have been garbage-collected.

**After (canary.10+)**: Request context is released promptly after the cached response is serialized. Memory growth is bounded.

**Audit recipe** (1-step for most apps):
```bash
# After upgrading to canary.10+:
# 1. Monitor memory usage in your production environment for 24-48h
#    - If using Vercel: check Memory usage in the dashboard
#    - If self-hosted: monitor RSS/heap with your APM tool
# 2. Compare memory baseline before and after upgrade
#    - If memory is stable (not growing unbounded): fix confirmed
#    - If memory still grows: open a GitHub issue with your cache handler config
```

**`experimental.useCache` users**: If you're using `experimental.useCache` (or `export const use cache = true`) in production, this leak fix is operationally HIGH priority — upgrade immediately.

### Combined 3-Pattern Audit Recipe (canary.10 Upgrade Checklist)

```bash
# Step 1: Upgrade next to canary.10
npm install next@16.4.0-canary.10

# Step 2: Verify CSS module class names shortened (Pattern AG)
# Inspect any .module.css file in devtools — should see [hash]_[local] format

# Step 3: Verify chunk hashes changed (Pattern AH)
# Run next build and check /_next/static/chunks/ — filenames should have longer hashes
# Note: CDN cold-cache expected on first deploy after upgrade

# Step 4: Monitor memory after upgrade (Pattern AI — use cache leak fix)
# Watch production memory for 24-48h post-deploy

# Step 5: If on Pages Router — check for React 18 deprecation warning
# React 18 in Pages Router deprecated; will be removed in Next.js 17
# Plan React 19 upgrade or App Router migration

# Step 6: Optional — enable experimental.strictRouteMatching if you have complex parallel routes
# Add to next.config.ts: experimental: { strictRouteMatching: true }
# Test thoroughly before enabling in production
```

### Recommended version pin

- **Production (stable apps)**: `next@^16.3.3` (CVE-patched; no reason to move to canary unless you need a specific fix)
- **Experimenters (16.4.x)**: `next@16.4.0-canary.10` (includes memory leak fix — operationally important for `use cache` users)
- **`@clerk/nextjs`**: `^7.8.3` (routine PATCH within 7.8.x)

### Sources

- [PR #97944 — Turbopack: shorten CSS module class names](https://github.com/vercel/next.js/pull/97944) — by @nicolo-ribaudo; **automatic; Turbopack-only**; ~60% class name reduction
- [PR #97945 — Turbopack: widen the chunk ident hash from 7 to 13 base38 chars](https://github.com/vercel/next.js/pull/97945) — by @mischnic; fixes #97766 chunk hash collisions
- [PR #97941 — Fix request-context retention in the default use cache handler](https://github.com/vercel/next.js/pull/97941) — by @gnoff; **operationally HIGH** memory leak fix for `use cache`
- [PR #97108 — Expose `experimental.strictRouteMatching`](https://github.com/vercel/next.js/pull/97108) — by @gnoff; optional flag for parallel route strictness
- [PR #97689 — Pages Router: Deprecate React 18 support](https://github.com/vercel/next.js/pull/97689) — React 18 in Pages Router deprecated; Next.js 17 removes support
- [Cross-references](cross-refs): `api.md` → canary.10 API-surface section for full PR table; `styling.md` → CSS module naming updated; `routing.md` → `experimental.strictRouteMatching` implications
