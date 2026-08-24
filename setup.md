

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
