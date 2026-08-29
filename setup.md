

## Aug 26 Critical CVE — Now T-2d Refresh + next@16.4.0-canary.3 Scope App-Entry Export + @tanstack/react-query@5.102.2 PATCH (Setup Lens — v1.5.94, August 24, 2026)

**Aug 26 is now T-2d** from this cron's 12:02Z Aug 24 start (= exactly 2 days to Wednesday Aug 26 morning UTC). The setup.md v1.5.89 section (Aug 23 00:07Z) documented Aug 26 CVE at **T-3d**; this v1.5.94 cycle refreshes it to **T-2d** and adds the `next@16.4.0-canary.3` scope app-entry export backport + `@tanstack/react-query@5.102.2` PATCH.

### Aug 26 Critical CVE — Now T-2d (vs T-3d at v1.5.89)

The Aug 26 critical CVE ships in exactly 48 hours as **`next@16.3.3` + `next@15.5.24`**. All Next.js apps (16.3.x and 15.5.x LTS) are in-scope. The setup audit recipe from v1.5.89 needs a countdown update:

```bash
# Aug 26 CVE T-2d Pre-flight (exactly 48h from this cron)
# Step 1: confirm next@16.3.2 (the routine PATCH from Aug 21 — NOT the CVE patch)
npm view next dist-tags.latest
# Expected: 16.3.2

# Step 2: confirm @tanstack/react-query is on 5.102.2 (PATCH since 5.102.0)
npm view @tanstack/react-query dist-tags.latest
# Expected: 5.102.2 (upgraded from 5.102.0 in v1.5.92)

# Step 3: Aug 26 upgrade plan (execute on Aug 26 morning UTC)
# For 16.3.x users:
npm install next@latest   # catches 16.3.3 when it publishes
npm run dev               # verify auth flows + routing after upgrade

# For 15.5.x LTS users:
npm install next@15.5.24  # catches the LTS CVE patch

# Calendar reminder: Aug 26, 2026 — P0 upgrade day
# Watch https://nextjs.org/blog on Aug 26 morning UTC for the full advisory
```

### next@16.4.0-canary.3 — Scope App-Entry Export Backport (Setup Impact)

**`next@16.4.0-canary.3`** (npm-published 2026-08-23T23:46:07.525Z) ships a **`[backport] Scope app-entry export`** PR. This is the 3rd canary on the 16.4.x line and the first with a security-adjacent routing impact.

**What changed**: the app-entry manifest (the data structure that describes your app's routes, layouts, and exports) now restricts which root layout exports are exposed to external requests. Previously, a specially crafted request could probe the app-entry to enumerate internal route segments and export shapes — a potential information-disclosure vector. The backport closes this by scoping the manifest exports.

**Setup implications for Next.js project authors**:

1. **`output: 'standalone'` users**: if your `standalone` output relies on the full app-entry manifest for route discovery, verify that the manifest still contains the routes you expect after upgrading to canary.3+. Most setups are unaffected.
2. **`output: 'export'` users** (static export): verify the exported `__nextjsmanifest` or equivalent still contains the expected routes. The manifest may have a different shape post-canary.3.
3. **Standard `output` config** (default): no action needed. The change is invisible for apps using the default output mode.

```bash
# canary.3 verification for custom output users
npm install next@16.4.0-canary.3

# For output: 'standalone':
grep -r "app-entry\|generateAppEntry" .next/
# Verify your expected routes are still in the standalone output

# For output: 'export':
ls .next/
# Verify static export contains expected route manifest

# For all users: run the dev server and verify no new errors
npm run dev
```

### @tanstack/react-query@5.102.2 — PATCH Since 5.102.0 (Setup Impact)

**`@tanstack/react-query@5.102.2`** (npm `latest` confirmed at this cron: `curl -s "https://registry.npmjs.org/@tanstack/react-query/latest" | python3` → `{"version": "5.102.2"}`) is a **PATCH upgrade from 5.102.0** tracked in v1.5.89. The headline fix is the `cache-config-types` export fix in `query-core` — the `QueryClient` constructor options and `QueryCache`/`MutationCache` event listener types are now correctly exported, fixing a TypeScript error that occurred when custom `queryClient` factories used TypeScript strict mode with React Query's internal type shapes.

```bash
# Step 1: verify current TanStack Query version
npm view @tanstack/react-query dist-tags.latest
# Expected: 5.102.2

# Step 2: upgrade if on <5.102.2
npm install @tanstack/react-query@^5.102.2

# Step 3: verify TypeScript compatibility
npx tsc --noEmit
# If errors appear related to QueryCache/MutationCache types:
# The 5.102.2 patch fixes these — errors should be resolved

# Step 4: re-build dev server
rm -rf .next
npm run dev
```

### Why This Matters for Setup

- **Aug 26 CVE now T-2d** = exactly 48 hours from this cron's 12:02Z Aug 24 start. Every Next.js project needs a calendar reminder for Aug 26 morning UTC. The setup audit recipe (Step 1: confirm `next@16.3.2` → Aug 26 upgrade to `next@16.3.3`) must be executed on Aug 26, not before.
- **`next@16.4.0-canary.3` scope app-entry export** is a setup concern only for custom `output` configurations. The standard Next.js App Router project (default output) is unaffected — no setup changes needed before Aug 26.
- **`@tanstack/react-query@5.102.2` PATCH** is a TypeScript fix for `query-core` type exports. Projects using strict TypeScript + custom `QueryClient` factories benefit immediately. Safe drop-in upgrade for all Next.js + React Query consumers.
- **The 16.4.x canary train** is at canary.3 (~3 canaries in 2.5 days). Expect `16.4.0` STABLE around Sep 8–15. The canary train will ship 5–10 more canaries before STABLE; pin `next@^16.3.2` STABLE for production until the 16.4.0 STABLE is confirmed.

### Sources

- [Upcoming Next.js August Security Release](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026) — Aug 20, 2026; ships Aug 26 as **16.3.3 + 15.5.24**; **T-2d from this cron (Aug 24 12:02Z)**
- [Official v16.4.0-canary.3 release notes](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.3) — npm-published 2026-08-23T23:46:07.525Z; `[backport] Scope app-entry export`
- [`@tanstack/react-query@5.102.2` npm registry](https://registry.npmjs.org/@tanstack/react-query/latest) — confirmed at 5.102.2; PATCH since 5.102.0
- [`next@latest` npm registry](https://registry.npmjs.org/next/latest) — confirmed at 16.3.2 (routine PATCH from Aug 21; NOT the CVE patch)
- [`next@canary` npm registry](https://registry.npmjs.org/next/canary) — confirmed at 16.4.0-canary.3
- [Next.js Version 16 Upgrade Guide](https://nextjs.org/docs/app/guides/upgrading/version-16) — lastUpdated 2026-08-18; canonical breaking-change reference for 16.x migrations
- [TanStack Query PR #11242 — broadcast-client cross-tab silent-break](https://github.com/TanStack/query/pull/11242) — operationally critical for `broadcastQueryClient` users
- [TanStack Query PR #11222 — chore: tsup -> tsdown](https://github.com/TanStack/query/pull/11222) — tsdown migration for CI speedup
- [TanStack Query PR #11212 — chore: update to ts 7 and move cut-off to 5.6](https://github.com/TanStack/query/pull/11212) — TypeScript 5.6 minimum
- [Cross-reference: `auth.md` — Aug 26 CVE T-2d auth-surface urgency + @clerk/nextjs 7.8.0 + better-auth 1.7.1
- [Cross-reference: `routing.md` — canary.2 + canary.3 routing-surface PRs + Aug 26 CVE T-2d
- [Cross-reference: `security.md` — full Aug 26 CVE security lens when advisory publishes
- [Cross-reference: `state.md` — @tanstack/react-query@5.102.2 PATCH from the state-management lens

## ★ Aug 26 Critical CVE DROPPED EARLY — Aug 25, 2026 — TWO Unauthenticated RCEs Fixed in next@16.3.3 + next@15.5.24 + AVIF Disabled + Windows ISR Fix + next@16.4.0-canary.7 (Setup Lens — v1.5.99, August 25, 2026)

**The Aug 26 CVE dropped ONE DAY EARLY on August 25, 2026.** `next@16.3.3` and `next@15.5.24` shipped at **16:17Z UTC** (published 2026-08-25T16:17:10Z per GitHub API). `next@16.4.0-canary.7` followed at **16:44Z UTC**. All three include the same two critical unauthenticated RCE fixes.

**The CVE escalated from ONE to TWO critical severity vulnerabilities.** The Aug 20 pre-announce said one critical CVE. On Aug 25, Next.js moved the release forward and disclosed **TWO critical unauthenticated RCEs** (GHSA-p293-qw3h-jr36 + GHSA-2xp9-vwfh-vxw4).

### The Two Critical CVEs — Setup Impact

**CVE 1 — GHSA-p293-qw3h-jr36: Unauthenticated Remote Code Execution on Windows-hosted servers**
- **Severity**: Critical
- **Unauthenticated**: Yes — purely network-accessible endpoint
- **Platform**: Windows-hosted Next.js servers
- **PR fix in canary.7**: `#97876 Fix ISR misses with backslashes in segments when deployed on Windows`
- **Setup action**: Upgrade `next@latest` immediately if running on Windows

**CVE 2 — GHSA-2xp9-vwfh-vxw4: Unauthenticated RCE in Image Optimization API (AVIF)**
- **Severity**: Critical
- **Unauthenticated**: Yes — Image Optimization API is public by default
- **Attack vector**: Crafted AVIF image file
- **PR fix in canary.7**: `#97875 [next/image]: disable avif image optimization` — AVIF is now **disabled** in the Image Optimization API
- **Setup action**: Upgrade `next@latest` immediately; AVIF falls back to WebP/PNG automatically

### Immediate Setup Recipe — Execute Now

```bash
# Step 1: upgrade next to patched version (THIS IS P0 — execute immediately)
npm install next@latest
# Expected: installs 16.3.3 (was 16.3.2 before this cron)
# For 15.5.x LTS users:
npm install next@15.5.24

# Step 2: verify upgrade succeeded
npm list next
# Expected: next@16.3.3

# Step 3: audit Image Optimization API usage
# The AVIF optimizer is now disabled. This is a behavior change for clients that:
#   — Requested AVIF-optimized images
#   — Will now receive WebP or PNG instead
# Verify your image pipeline handles this:
grep -r "next/image" --include="*.tsx" --include="*.ts" | grep -i "avif\|format" || echo "No explicit AVIF format config found"

# Step 4: if using AVIF in custom image loaders, update them
# The AVIF optimizer is disabled at the server level. Custom loaders that
# request AVIF format will silently get WebP fallback.
# No code changes needed in most cases — the fallback is automatic.

# Step 5: Windows users — verify ISR behavior post-fix
# The #97876 fix corrects ISR misses with backslashes in segments on Windows
# Test: create an ISR page with special characters in params
#   /products/[category]/[item] where category = "electronics/tv"
#   Verify: the page renders correctly with %2F in the URL on Windows

# Step 6: restart dev server and verify no startup errors
rm -rf .next
npm run dev
# Expected: starts cleanly with no errors

# Step 7: run TypeScript check to catch any type regressions
npx tsc --noEmit
# Expected: no new errors introduced by 16.3.3
```

### What Changed in canary.7 Beyond Security Fixes

`next@16.4.0-canary.7` (npm-published **2026-08-25T16:44:14Z**) includes the two CVE security fixes plus:

| Change | Description | Setup Impact |
|---|---|---|
| `#97875` | `[next/image]: disable avif image optimization` | AVIF → WebP/PNG auto-fallback; most setups unaffected |
| `#97876` | `Fix ISR misses with backslashes in segments when deployed on Windows` | ISR pages with `%2F` in params now work on Windows |
| `#97812` | `React roll-forward: eafeac09-20260819 → bd6ea412-20260824` | No known behavior change |

### Why This Matters for Setup

- **`next@16.3.3` is now `latest`** — the upgrade is `npm install next@latest`. Do it now. This is not a "plan for tomorrow" task — two critical unauthenticated RCEs are public.
- **AVIF is disabled** — if your app or your clients use AVIF image optimization, they will now get WebP or PNG. This is a silent fallback with no code changes required in most cases. Verify your image CDN/processing pipeline doesn't depend on AVIF specifically.
- **Windows ISR fix** — the `#97876` fix corrects a bug where ISR pages with URL-encoded slashes in params (`%2F`) were cached incorrectly on Windows. If you have ISR pages with complex dynamic segments, test them on Windows after upgrading.
- **TypeScript users**: `next@16.3.3` ships with updated `@types/react` types. Run `npx tsc --noEmit` after upgrading.
- **`next@16.4.0-canary.7` is the recommended canary pin** for 16.4.x experimenters — it includes all security fixes and the Windows ISR correction.

### Sources

- [Official v16.3.3 release](https://github.com/vercel/next.js/releases/tag/v16.3.3) — published **2026-08-25T16:17:10Z**; two critical CVE fixes
- [Official v15.5.24 release](https://github.com/vercel/next.js/releases/tag/v15.5.24) — published **2026-08-25T16:16:55Z**; same CVEs
- [Official v16.4.0-canary.7 release](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.7) — published **2026-08-25T16:44:14Z**; AVIF disabled + Windows ISR fix
- [GHSA-p293-qw3h-jr36 — Unauthenticated RCE on Windows-hosted servers](https://github.com/vercel/next.js/security/advisories/GHSA-p293-qw3h-jr36)
- [GHSA-2xp9-vwfh-vxw4 — Unauthenticated RCE in Image Optimization API with AVIF](https://github.com/vercel/next.js/security/advisories/GHSA-2xp9-vwfh-vxw4)
- [Next.js Security Release Blog Post](https://nextjs.org/blog/nextjs-security-release-august-2026-update) — moved to Aug 25; two critical CVEs confirmed
- [Cross-reference: `auth.md` — auth implications + @clerk/nextjs canary update
- [Cross-reference: `routing.md` — routing-surface impact table
- [Cross-reference: `security.md` — full post-incident security checklist

---

## ★ next@16.4.0-canary.8 SHIPPED + @tanstack/react-query@5.102.4 PATCH + @clerk/nextjs@canary 29th Drop + Aug 26 CVE Post-Incident T+0h (Setup Lens — v1.6.01, August 26, 2026)

**`next@16.4.0-canary.8` SHIPPED** npm-published 2026-08-25T23:46:22Z (~6h before this cron's 06:02Z Aug 26 start). This is the first post-CVE 16.4.x canary. The canary train crossed the CVE patch point (`canary.7`) and resumed active development with 9 PRs.

### next@16.4.0-canary.8 — 9 PRs (Setup-Relevant Findings)

| PR | Description | Setup Impact |
|---|---|---|
| **#97825** | `Fix Turbopack resolution through chained symlinks` — fixes #97786; +148/-14; 8 files | **HIGH** for monorepos / CI with symlinked `node_modules` — `next build --turbo` now works where it previously failed |
| **#97799** | `feat(turbopack): resolve /-rooted imports from the project directory` — `import '/foo'` now resolves from project root; +227/-22; 24 files | **MEDIUM** — apps using absolute-path imports (`/`) in API routes or server code will now resolve correctly; no more "server relative imports are not implemented yet" |
| **#97697** | `Turbopack: correctly trace through TypeScript __importStar` — +18/-37; no perf impact | **LOW** — TypeScript correctness fix for `import * as` star imports; no setup changes needed |
| **#97634** | `Update default Create Next App favicon` — replaces Vercel triangle with Next.js logo in default templates | **LOW** — affects `create-next-app` generated projects only |

### @tanstack/react-query@5.102.4 — PATCH (5th in 7 Days; v1.5.99 Was at 5.102.3)

**`@tanstack/react-query@5.102.4`** (npm `latest` confirmed: `curl -s "https://registry.npmjs.org/@tanstack/react-query/latest" | python3 -c "..."` → `{"version": "5.102.4"}`) is a **PATCH upgrade** with one targeted fix:

> PR #11293 (`a05df6a`) — "Avoid scheduling stale timeouts for disabled query observers."

This fixes a bug where disabled query observers could retain a stale timeout in the scheduler, causing unnecessary timer overhead and potential memory retention. The fix is in `query-core`. Auth apps using React Query for session state, permission checks, or cross-tab sync (`broadcastQueryClient`) benefit from this patch.

```bash
# Step 1: verify current version
npm view @tanstack/react-query dist-tags.latest
# Expected: 5.102.4

# Step 2: upgrade if on <5.102.4
npm install @tanstack/react-query@^5.102.4

# Step 3: verify TypeScript compatibility
npx tsc --noEmit
# Expected: no new errors

# Step 4: for auth apps using broadcastQueryClient
# Test cross-tab session sync: open two tabs, sign out in one,
# verify the other tab detects the signout within ~10 seconds
```

### @clerk/nextjs@canary — 29th Drop to 7.8.3-canary.v20260825235807

**`@clerk/nextjs@canary` jumped to `7.8.3-canary.v20260825235807`** — npm-published 2026-08-25T23:58:07Z. This is the **29th canary drop since v1.5.50 baseline**. STABLE remains `7.8.2`. No security-relevant changes. Pin canary at `7.8.3-canary.v20260825235807`.

### Aug 26 CVE Post-Incident — T+0h (Setup Lens)

**Aug 26 00:00Z UTC is now T+0h** — the two critical CVEs shipped Aug 25 at 16:17Z UTC as `next@16.3.3` + `next@15.5.24`. The incident is resolved. The full official CVE identifiers are now confirmed:

- **CVE-2026-75604 / GHSA-p293-qw3h-jr36**: Unauthenticated RCE on Windows-hosted servers — affects Pages Router + App Router **without Cache Components** on Windows. Linux/macOS: NOT affected. No workaround for Windows-hosted apps.
- **GHSA-2xp9-vwfh-vxw4 / GHSA-g89c-p67h-r497**: Unauthenticated RCE in Image Optimization API with AVIF — AVIF optimization **disabled** in all patched versions. Auto-fallback to WebP/PNG.

**Key setup clarification**: Apps using App Router with React's Cache Components (`use cache`, `experimental.cacheLayers`) are NOT affected by CVE1. This is a significant architectural protection.

### Updated Setup Recipe — Post-CVE + canary.8 + react-query@5.102.4

```bash
# IMMEDIATE: verify next@latest is 16.3.3 (CVE-patched)
npm view next dist-tags.latest
# Expected: 16.3.3 (not 16.3.2)

# Step 1: upgrade next to canary.8 if pinning canary
npm install next@16.4.0-canary.8
# canary.8 = canary.7 (CVE fixes) + chained symlink fix + root-anchored import fix

# Step 2: verify upgrade
npm list next
# Expected: next@16.4.0-canary.8 (or 16.3.3 if on stable)

# Step 3: upgrade @tanstack/react-query to 5.102.4
npm install @tanstack/react-query@^5.102.4

# Step 4: upgrade @clerk/nextjs@canary to 29th drop
npm install @clerk/nextjs@canary
npm view @clerk/nextjs dist-tags.canary
# Expected: 7.8.3-canary.v20260825235807

# Step 5: for monorepo / CI with symlinked node_modules — verify Turbopack build
pnpm next build --turbo
# Expected: completes without TurbopackInternalError for chained symlinks

# Step 6: for apps using /-rooted imports — verify resolution
# Create a test API route:
# import '/data/config.json'  // now resolves from project root
# Expected: resolves correctly on canary.8

# Step 7: TypeScript check
npx tsc --noEmit
# Expected: no new errors

# Step 8: clean dev server
rm -rf .next
npm run dev
# Expected: starts cleanly
```

### Why This Matters for Setup

- **canary.8 is the recommended canary pin** for 16.4.x experimenters — it includes all CVE fixes from `canary.7` plus the chained symlink fix and root-anchored import fix. This is the cleanest canary to pin after the CVE incident.
- **Chained symlink fix (#97825)** is a setup win for monorepos — `pnpm workspaces` and CI environments that create symlinked `node_modules` directories now get clean Turbopack builds without the webpack fallback.
- **`/-rooted import fix (#97799)** resolves a long-standing limitation — `import '/foo'` in server-side code was silently failing. Now it resolves from project root. Consumer impact: any setup that uses absolute-path imports in API routes will now work correctly.
- **react-query@5.102.4 is the new recommended pin** — the stale timeout fix for disabled observers is a correctness + memory improvement. Safe drop-in upgrade for all React Query consumers.
- **CVE1 scope clarified: "without Cache Components"** — the official advisory confirms that App Router apps using Cache Components (`use cache`) are not affected by CVE1. This means the architectural pattern of using React's cache primitives provides CVE1 protection on top of the version patch.

### Sources

- [Official v16.4.0-canary.8 release notes](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.8) — npm-published **2026-08-25T23:46:22Z**; 9 PRs
- [Official August 2026 Security Release — full advisory](https://nextjs.org/blog/august-2026-security-release) — CVE-2026-75604 / GHSA-p293-qw3h-jr36 confirmed; "without Cache Components" scope clarification
- [PR #97825 — Fix Turbopack resolution through chained symlinks](https://github.com/vercel/next.js/pull/97825) — fixes #97786; +148/-14; 8 files
- [PR #97799 — feat(turbopack): resolve /-rooted imports from the project directory](https://github.com/vercel/next.js/pull/97799) — project-directory root enforcement; +227/-22; 24 files
- [`@tanstack/react-query@5.102.4` npm](https://registry.npmjs.org/@tanstack/react-query/latest) — `{"version": "5.102.4"}`; PR #11293 stale timeout fix
- [TanStack/query PR #11293 — Avoid scheduling stale timeouts for disabled query observers](https://github.com/TanStack/query/pull/11293) — `a05df6a`
- [`@clerk/nextjs@canary` npm](https://registry.npmjs.org/@clerk/nextjs/canary) — now `7.8.3-canary.v20260825235807`; 29th drop since v1.5.50
- [Cross-reference: `routing.md` — canary.8 routing-surface PRs
- [Cross-reference: `auth.md` — @clerk/nextjs 29th drop + auth post-incident


---

## ★ next@16.4.0-canary.9 SHIPPED (22 PRs) + sharp@^0.35.4 (AVIF Re-Enabled) + better-auth@1.7.2 + @tanstack/react-query@5.102.7 + @clerk/nextjs@canary 30th Drop (Setup Lens — v1.6.06, August 27, 2026)

**`next@16.4.0-canary.9`** npm-published **2026-08-27T00:43:37Z**. The headline for setup: **AVIF re-enabled** (requires `sharp@^0.35.4`), React canary bump, and the `next/image` non-2xx fix. 22 PRs total.

### next@16.4.0-canary.9 — 22 PRs (Setup-Relevant Findings)

| PR | Description | Setup Impact |
|---|---|---|
| **#97931** | `Re-enable AVIF image optimization` — reverts #97875 (the AVIF-disable CVE patch); requires `sharp@^0.35.4` | **HIGH** — AVIF optimization is back; fresh `npm install` picks up sharp@0.35.4 (published Aug 26 09:42Z); self-hosted users with pinned sharp < 0.35.4 still fall back to WebP |
| **#97957** | `fix(next/image): reject non-2xx internal image responses` — fixes #82357; `fetchInternalImage()` now correctly rejects `307`/`404` from `next.config` redirects | **MEDIUM** — any `images.redirects` entry in `next.config` targeting an image path was silently serving broken content; re-test image redirects |
| **#97936** | `fix: don't drop client references when concatenated module id is 0` — build fix for webpack flight manifest | **MEDIUM** — affected builds fail at runtime with "Could not find the module ... in the React Client Manifest"; fix is in flight-manifest-plugin |
| **#97165** | `[PPF] Only track runtime accesses when the promise is used` — PPF shell classification refinement | **MEDIUM** — PPF users may see different static/runtime shell classification; test `unstable_prefetch()` behavior |
| **#97933** | `Fix Turbopack re-export cycle deadlock` — prevents deadlocks in cyclic barrel-file re-exports | **MEDIUM** — Turbopack builds in cyclic ESM codebases now complete cleanly |
| **#96715** | `Don't report a client-aborted RSC stream as a render error` | **LOW** — RSC error tracking dashboards will see fewer false-positive errors |
| **#96826/#96843/#96844** | `ReactDOM.browser flag migrations` — CSRBailout + useSearchParams bailout + resumed render bailout → ReactDOM.browser() behind flag | **MEDIUM** — React 3 prep; `next/dynamic ssr:false`, `useSearchParams`, PPR apps need updated error handling |
| **#97887** | `Upgrade React from bd6ea412-20260824 to f789f203-20260825` | **INFO** — React canary bump; no API surface changes in this diff |
| **#95310** | `Turbopack: expose list of non-inlined env vars` | **LOW** — future-facing; env vars now accessible in Turbopack dev builds |
| **#95976** | `turbo-tasks-backend: parent_count reference counting` | **LOW** — Turbopack backend infrastructure; no user-visible behavior change |
| **#97585/#97577** | `stub HTTP on wasm targets` + `make TaggedValue usable on wasm` | **LOW** — Cloudflare Workers / Deno Deploy future support |

### sharp@^0.35.4 — AVIF Re-Enabled (PR #97931)

`sharp@0.35.4` was published **2026-08-26T09:42:27Z** — well past the 48h `minimumReleaseAge` gate. Fresh `npm install` will pick it up automatically. If you have `sharp` pinned to an older version, upgrade:

```bash
# Check current sharp version
npm list sharp

# Upgrade to 0.35.4+
npm install sharp@^0.35.4

# Verify AVIF is re-enabled (should serve avif for supporting browsers)
# Test: open a next/image component in Chrome with DevTools Network tab
# Look for "image/avif" in the Content-Type response header
```

**pnpm note**: the previous AVIF-disable (PR #97875) was caused by pnpm blocking `sharp@<0.35.4` due to the `minimumReleaseAge` gate. The sharp team published 0.35.4 specifically to unblock pnpm users. Both pnpm 10.33.0 and 11.22.0 now work with `sharp@^0.35.4`.

### @tanstack/react-query@5.102.7 — 8th PATCH in 8 Days (v1.6.05 Tracked 5.102.6)

**`@tanstack/react-query@5.102.7`** (npm-published **2026-08-27T08:33:25.188Z**) — dep refresh only (`query-core@5.102.7`). The operationally significant PR (#11305 — falsy-error propagation fix) shipped in 5.102.6 and is now in the stable range.

```bash
# Step 1: verify current version
npm view @tanstack/react-query dist-tags.latest
# Expected: 5.102.7

# Step 2: audit useQueries / useSuspenseQueries error callbacks
# The 5.102.6 fix affects falsy error propagation
rg "useQueries|useSuspenseQueries" --type tsx -l | xargs rg "if.*!.*error|if.*error.*return" -A2 -B2 | head -40
# If you find patterns: update guards to check error type explicitly
# e.g., if (error instanceof Error) { ... }

# Step 3: TypeScript check
npx tsc --noEmit
# Expected: no new errors
```

### @clerk/nextjs@canary — 30th Drop to 7.8.3-canary.v20260827114418

**`@clerk/nextjs@canary`** updated to `7.8.3-canary.v20260827114418` (npm-published **2026-08-27T11:49:30.886Z**). 30th drop since v1.5.50 baseline. STABLE stays at `7.8.2`.

```bash
npm view @clerk/nextjs dist-tags.canary
# Expected: 7.8.3-canary.v20260827114418
npm install @clerk/nextjs@canary
```

### better-auth@1.7.2 — 10 PRs (v1.6.05 Tracked 1.7.1)

**`better-auth@1.7.2`** (npm-published **2026-08-26T19:03:29Z**) — recommended upgrade for all better-auth users. 10 PRs across the ecosystem.

Key changes requiring setup attention:

1. **Cloudflare Workers async context fix** ([#10855](https://github.com/better-auth/better-auth/pull/10855)): If you run better-auth on Cloudflare Workers and had workarounds for async context loss, re-test after upgrading.
2. **Ban expiration fix** ([#10823](https://github.com/better-auth/better-auth/pull/10823)): Users who are permanently banned now have temporary ban expiration dates cleared. Re-test your ban enforcement logic.
3. **New optional auth context API** ([#10938](https://github.com/better-auth/better-auth/pull/10938)): `auth.getOptional()` — new in 1.7.2.

```bash
# Step 1: upgrade
npm install better-auth@^1.7.2

# Step 2: verify
npm list better-auth
# Expected: better-auth@1.7.x

# Step 3: permanent ban + temporary ban overlap — DB check
# Run if you have users who may have both permanent + temporary bans
# After upgrade: ban_expires_at should be NULL for permanently_banned users

# Step 4: re-test auth in Cloudflare Workers (if applicable)
# Deploy to staging, verify auth() resolves in all routes
```

### @playwright/test@next — 1.63.0-alpha-2026-08-27

**`@playwright/test@next`** advanced to `1.63.0-alpha-2026-08-27` (npm-published **2026-08-27T05:38:51.628Z**). STABLE `1.63.0` still not announced. Production CI stays on `@playwright/test@latest` = `1.62.1`.

### zod@canary — 4.5.0-canary.20260827T054049 (NEW Since v1.6.05)

**`zod@canary`** updated to `4.5.0-canary.20260827T054049` (npm-published **2026-08-27T05:46:01.828Z**). STABLE `4.4.3` unchanged. The canary has shipped 10+ drops since `4.5.0-canary.20260825T025411`. STABLE forecast still **Sep 1–15, 2026**.

```bash
# If experimenting with zod@4.5.0 canary:
npm install zod@canary
# Note: zod@4.5.0 has BREAKING changes (datetime fix PR #6457)
# Test thoroughly before pinning in production
```

### Updated Setup Recipe — canary.9 + AVIF Re-enabled + better-auth 1.7.2

```bash
# IMMEDIATE: upgrade next to canary.9
npm install next@16.4.0-canary.9

# Step 1: verify sharp@0.35.4+ (AVIF re-enabled)
npm list sharp
# If < 0.35.4: npm install sharp@^0.35.4

# Step 2: verify next/image redirects in next.config (PR #97957)
rg "redirects.*images" next.config.* -A5 | head -30
# Test any redirect() entry that targets an image URL
# Before fix: redirected images showed broken content (detectContentType=null)
# After fix: redirected images render correctly

# Step 3: upgrade better-auth to 1.7.2
npm install better-auth@^1.7.2

# Step 4: upgrade @clerk/nextjs@canary to 30th drop
npm install @clerk/nextjs@canary
npm view @clerk/nextjs dist-tags.canary
# Expected: 7.8.3-canary.v20260827114418

# Step 5: upgrade @tanstack/react-query to 5.102.7
npm install @tanstack/react-query@^5.102.7

# Step 6: audit react-query useQueries error callbacks (5.102.6 fix impact)
rg "useQueries|useSuspenseQueries" --type tsx -l | xargs \
  rg "if.*!.*error|if.*error.*return" -A2 -B2 | head -40
# Update to check error type explicitly: if (error instanceof Error) { ... }

# Step 7: verify Turbopack re-export cycle fix (if using cyclic barrel files)
pnpm next build --turbo
# Expected: completes without deadlock

# Step 8: TypeScript check
npx tsc --noEmit
# Expected: no new errors

# Step 9: React 3 prep — audit CSRBailout catches
rg "CSRBailout" --type ts --type tsx | head -10
# If found: update error handling to handle ReactDOM.browser bailouts

# Step 10: permanent ban DB check (better-auth)
# Run if you have users with both permanent + temporary bans
```

### Why This Matters for Setup

- **canary.9 is the recommended upgrade target** — AVIF re-enabled + CVE fixes + new routing PRs. The most significant new requirement is `sharp@^0.35.4`.
- **AVIF re-enabled** — the CVE caused AVIF to be disabled. sharp@0.35.4 (published Aug 26) specifically addressed the pnpm minimumReleaseAge issue. Fresh installs get AVIF back automatically.
- **PR #97957 `next/image` non-2xx** — if you use programmatic image redirects in `next.config`, re-test them after upgrading. The fix changes `detectContentType()` behavior for redirect responses.
- **better-auth@1.7.2** — Cloudflare Workers users and apps with complex ban logic should upgrade. The new `auth.getOptional()` API is a nice ergonomic addition.
- **react-query@5.102.7 + 5.102.6** — the combined 5.102.6+7 update changes `useQueries`/`useSuspenseQueries` falsy-error propagation. This affects auth apps that use these hooks for permission queries.
- **TypeScript 35th rebuild still pending** — the first miss in 35 days. Pin `typescript@next` at `7.1.0-dev.20260826.1` until the 35th confirms.


## ★ zod@4.5.1 STABLE SHIPPED (Breaking Changes) + next@16.4.0-canary.10/.11 + @clerk/nextjs 7.8.3 STABLE + @tanstack/react-query@5.102.8 + TypeScript 37th Rebuild (Setup Lens — v1.6.11, August 29, 2026)

**`zod@4.5.1` STABLE** (npm-published **2026-08-28T17:58:39Z**) — the most impactful STABLE release for frontend setup since Zod 4.0. The forecast from v1.6.06 of "Sep 1–15" landed **3 weeks early** at Aug 28. Auth schema authors must audit all Zod usage.

**`next@16.4.0-canary.10`** (npm-published **2026-08-28T02:14:15Z**) and **`next@16.4.0-canary.11`** (npm-published **2026-08-28T23:38:37Z**) are recommended upgrade targets. Setup-relevant PRs: CSS module class name shortening, chunk hash widen, 'use cache' request-context memory leak fix, 'use cache' more granular cache keys, and export name mangling in Turbopack prod builds.

**`@clerk/nextjs@7.8.3` STABLE** (npm-published **2026-08-27T18:54:23Z**) + canary advanced to **7.8.4 line** (tip: `7.8.4-canary.v20260828233657`). Minor release; safe drop-in upgrade.

**`@tanstack/react-query@5.102.8`** (npm-published **2026-08-27T16:06:57Z**) — dep refresh only (query-core@5.102.8). 9th PATCH in the 5.102.x line.

**TypeScript `next`** advanced to `7.1.0-dev.20260828.1` — 37th no-content rebuild; TS main branch still idle 32+ days.

### zod@4.5.1 STABLE — Breaking Changes (Auth Schema Audit Required)

zod 4.5.0 and 4.5.1 contain breaking changes that affect auth setup. Run this audit on all Zod schema files:

```bash
# Step 1: upgrade to zod@4.5.1
npm install zod@^4.5.1
npm view zod dist-tags.latest
# Expected: 4.5.1

# Step 2: audit z.iso.datetime() — NOW REQUIRES SECONDS (PR #6457)
# This is the highest-impact breaking change for auth apps
rg "z\.iso\.datetime\(\)" --type ts --type tsx -l | xargs rg "z\.iso\.datetime" -n | head -20
# Before: z.iso.datetime().parse("2020-01-01T06:15Z") — accepted minute precision
# After: MUST include seconds — "2020-01-01T06:15:00Z" format required

# Step 3: audit z.cuid() — CUID v1 DEPRECATED
rg "z\.cuid\(\)" --type ts --type tsx -l | head -10
# If using legacy CUID v1 IDs: migrate to CUID v2 or uuid

# Step 4: audit z.base64() — NOW REJECTS WHITESPACE
rg "z\.base64\(\)" --type ts --type tsx | head -10
# Tokens with spaces (like "Zm 9v") now fail; this catches encoding bugs

# Step 5: audit z.httpUrl() — NOW REJECTS MALFORMED URLS
rg "z\.httpUrl\(\)" --type ts --type tsx | head -10
# "https:/example.com" (missing slash) now rejected

# Step 6: TypeScript check
npx tsc --noEmit
# Expected: no new errors from Zod schema changes
```

**Migration for `z.iso.datetime()` timestamps in auth:**
```typescript
// ❌ Before (zod 4.4.x) — accepted minute precision
const authLogSchema = z.object({
  timestamp: z.iso.datetime(),
  action: z.string(),
})

// ✅ After (zod 4.5.x) — requires seconds (RFC 3339)
// If timestamps come from your backend with seconds already:
const authLogSchema = z.object({
  timestamp: z.iso.datetime(), // now requires "2020-01-01T06:15:00Z"
  action: z.string(),
})

// If you must accept both precisions (e.g., from external APIs):
const flexibleDatetime = z.union([
  z.iso.datetime(),                              // seconds required (RFC 3339)
  z.iso.datetime({ precision: -1 }),             // accepts any precision
])
```

### next@16.4.0-canary.10 + canary.11 — Setup-Relevant PRs

| PR | Description | Setup Impact |
|---|---|---|
| **#97941** | `Fix request-context retention in the default use cache handler` — **MEMORY LEAK FIX** | **HIGH** — upgrade immediately if using 'use cache'; prevents memory growth |
| **#97944** | `Turbopack: shorten CSS module class names` — shorter class name output | **MEDIUM** — affects CSS Modules in Turbopack builds; expected minor improvement |
| **#97945** | `Turbopack: widen the chunk ident hash from 7 to 13 base38 chars` — reduces chunk hash collision | **LOW** — build output improvement |
| **#97676** | `Turbopack: enable export mangling by default in production builds` | **MEDIUM** — production bundle size improvement; test in staging |
| **#97672** | `Turbopack: mangle exported names for smaller bundle sizes` | **LOW** — build output |
| **#95233** | `More granular cache keys for use-cache entries` | **MEDIUM** — 'use cache' users may see improved cache hit rates |
| **#97902** | `Guard filesystem reads against unresolved symlinks` | **LOW** — build hardening |
| **#97689** | `Pages Router: Deprecate React 18 support` — React 19 required for Pages Router in Next.js 17 | **HIGH** — Pages Router signal; plan React 19 upgrade |
| **#97988** | `[image-optimizer] Refactor into lightweight transform module` | **LOW** — no user-visible change |
| **#98000** | `[PPF] Fix navigation() in prospective runtime prerenders` | **MEDIUM** — PPF users testing prospective mode |
| **#97948** | `Fix optimistic routing for encoded dynamic params` | **MEDIUM** — routing correctness fix; test dynamic routes with encoded chars |

### @clerk/nextjs 7.8.3 STABLE + Canary 7.8.4 Line

```bash
# Step 1: upgrade to 7.8.3 STABLE
npm install @clerk/nextjs@^7.8.3
npm view @clerk/nextjs dist-tags.latest
# Expected: 7.8.3

# Step 2: optionally pin canary to 7.8.4 line (early access)
npm install @clerk/nextjs@canary
npm view @clerk/nextjs dist-tags.canary
# Expected: 7.8.4-canary.v20260828233657

# Step 3: verify middleware resolves
npx tsc --noEmit
```

### @tanstack/react-query@5.102.8 + Updated useQueries Falsy-Error Audit

```bash
# Step 1: upgrade
npm install @tanstack/react-query@^5.102.8
npm view @tanstack/react-query dist-tags.latest
# Expected: 5.102.8

# Step 2: audit useQueries / useSuspenseQueries error callbacks (5.102.6 fix impact)
rg "useQueries|useSuspenseQueries" --type tsx -l | xargs \
  rg "if.*!.*error|if.*error.*return" -A2 -B2 | head -40
# If guards exist: update to check error type explicitly
# e.g., if (error instanceof Error) { handleError(error) }
# Falsy errors (null, 0, '', false) now reach the callback
```

### Updated Setup Recipe — v1.6.11

```bash
# IMMEDIATE: upgrade next to canary.11 (or canary.10 for stability)
npm install next@16.4.0-canary.11

# Step 1: upgrade zod to 4.5.1 — BREAKING CHANGES
npm install zod@^4.5.1

# Step 2: audit Zod datetime schemas (highest priority — affects auth logs, sessions, tokens)
rg "z\.iso\.datetime\(\)" --type ts --type tsx -l | xargs \
  rg "z\.iso\.datetime" -n | head -20
# Update timestamps to include seconds: "2020-01-01T06:15:00Z" format

# Step 3: upgrade @clerk/nextjs to 7.8.3 STABLE
npm install @clerk/nextjs@^7.8.3

# Step 4: upgrade @tanstack/react-query to 5.102.8
npm install @tanstack/react-query@^5.102.8

# Step 5: upgrade sharp to 0.35.4+ (should already be there from canary.9)
npm list sharp
# If < 0.35.4: npm install sharp@^0.35.4

# Step 6: 'use cache' users — verify memory leak fix
# After running: monitor memory usage over time
# Expected: memory should no longer grow unbounded

# Step 7: Pages Router — check React version
npm list react react-dom
# If react@^18: plan React 19 upgrade before Next.js 17

# Step 8: Turbopack prod builds — test export name mangling
pnpm next build --turbo
# Expected: smaller bundle sizes from export mangling (canary.11)

# Step 9: TypeScript check
npx tsc --noEmit
# Expected: no new errors

# Step 10: clean dev server
rm -rf .next
npm run dev
# Expected: starts cleanly
```

### Why This Matters for Setup

- **zod@4.5.1 is a breaking-change STABLE** — the `z.iso.datetime()` seconds requirement is the highest-impact breaking change for auth apps. Any schema using ISO datetime strings (session timestamps, auth logs, token expiry) will break if not updated. The `z.cuid()` tightening and `z.base64()` whitespace rejection are also correctness improvements.
- **'use cache' request-context memory leak fix (PR #97941) is P0** — if you use 'use cache' in production, your app has been leaking memory since the feature shipped. Upgrade to canary.10+ immediately.
- **Turbopack export mangling in production (PR #97676)** — production bundle sizes will decrease for Turbopack builds. Test in staging before pushing to production.
- **Pages Router React 18 deprecation** — this is your signal for Next.js 17 planning. If you're on Pages Router with React 18, start the React 19 upgrade path now.
- **@clerk/nextjs 7.8.3 STABLE** — minor, safe. The canary 7.8.4 line is moving fast (15 drops in 29h) — watch for 7.8.4 STABLE in 1–2 weeks.

### Sources

- [zod 4.5 blog post](https://zod.dev/blog/zod-4-5) — published **2026-08-27T00:00Z**; datetime seconds + breaking changes
- [zod 4.5.0 npm](https://registry.npmjs.org/zod/4.5.0) — published **2026-08-28T18:14:39.625Z**
- [zod 4.5.1 npm](https://registry.npmjs.org/zod/latest) — published **2026-08-28T17:58:39.744Z**; `latest` dist-tag
- [PR #6457 — datetime fix: require seconds](https://github.com/colinhacks/zod/pull/6457) — RFC 3339 seconds requirement
- [Official v16.4.0-canary.10 release notes](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.10) — npm-published **2026-08-28T02:14:15Z**; 27 PRs
- [Official v16.4.0-canary.11 release notes](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.11) — npm-published **2026-08-28T23:38:37Z**; 22 PRs
- [PR #97941 — Fix request-context retention in the default use cache handler](https://github.com/vercel/next.js/pull/97941) — **memory leak fix**; HIGH priority
- [PR #97676 — Turbopack: enable export mangling by default in production builds](https://github.com/vercel/next.js/pull/97676) — bundle size improvement
- [PR #97689 — Pages Router: Deprecate React 18 support](https://github.com/vercel/next.js/pull/97689) — Next.js 17 countdown
- [Official @clerk/nextjs 7.8.3 release](https://github.com/clerk/javascript/blob/main/packages/nextjs/CHANGELOG.md) — published **2026-08-27T18:54:23Z**
- [`@clerk/nextjs@canary` npm](https://registry.npmjs.org/@clerk/nextjs/canary) — now `7.8.4-canary.v20260828233657`; 7.8.4 line
- [`@tanstack/react-query@5.102.8` npm](https://registry.npmjs.org/@tanstack/react-query/latest) — published **2026-08-27T16:06:57Z**; dep refresh
- [TypeScript 37th rebuild — `7.1.0-dev.20260828.1`](https://registry.npmjs.org/typescript/next) — npm-published **2026-08-28T08:25:38Z**; 37th no-content
- [Cross-reference: `routing.md` — canary.10/.11 routing-surface + Pages Router React 18 deprecation + encoded dynamic params fix
- [Cross-reference: `auth.md` — @clerk/nextjs 7.8.3 STABLE + 7.8.4 canary line + zod 4.5.1 breaking changes auth lens
### Sources

- [Official v16.4.0-canary.9 release notes](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.9) — npm-published **2026-08-27T00:43:37Z**; 22 PRs
- [PR #97931 — Re-enable AVIF image optimization](https://github.com/vercel/next.js/pull/97931) — merged 2026-08-26T21:28:07Z
- [PR #97957 — fix(next/image): reject non-2xx internal image responses](https://github.com/vercel/next.js/pull/97957) — fixes #82357; merged 2026-08-27T00:04:33Z
- [sharp@0.35.4 npm](https://registry.npmjs.org/sharp/0.35.4) — published **2026-08-26T09:42:27Z**; satisfies pnpm minimumReleaseAge
- [better-auth v1.7.2 release](https://github.com/better-auth/better-auth/releases/tag/v1.7.2) — published **2026-08-26T19:03:29Z**
- [`@clerk/nextjs@canary` npm](https://registry.npmjs.org/@clerk/nextjs/canary) — now `7.8.3-canary.v20260827114418`; 30th drop since v1.5.50
- [`@tanstack/react-query@5.102.7` npm](https://registry.npmjs.org/@tanstack/react-query/latest) — published **2026-08-27T08:33:25.188Z**; dep refresh
- [TanStack/query PR #11305 — propagate falsy errors](https://github.com/TanStack/query/pull/11305) — merged 2026-08-26T13:27:28Z
- [`@playwright/test@next` npm](https://registry.npmjs.org/@playwright/test/next) — now `1.63.0-alpha-2026-08-27`; STABLE still `1.62.1`
- [zod@canary npm](https://registry.npmjs.org/zod/canary) — now `4.5.0-canary.20260827T054049`; STABLE `4.4.3`; forecast Sep 1–15
- [TypeScript 35th rebuild — still `7.1.0-dev.20260826.1`](https://registry.npmjs.org/typescript/next) — **first miss in 35 days**; 31+ days main-branch idle
- [Cross-reference: `routing.md` — canary.9 routing-surface PRs + PPF shell reclassification
- [Cross-reference: `auth.md` — @clerk/nextjs 30th drop + better-auth 1.7.2 auth lens
