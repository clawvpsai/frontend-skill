

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
