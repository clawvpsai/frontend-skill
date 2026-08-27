

## Aug 26 Critical CVE — Now T-2d Refresh + next@16.4.0-canary.3 + @tanstack/react-query@5.102.2 (Auth Lens — v1.5.94, August 24, 2026)

**Aug 26 is now T-2d** from this cron's 12:02Z Aug 24 start (= exactly 2 days to Wednesday Aug 26 morning UTC). The auth.md v1.5.92 section (Aug 23 00:07Z) documented Aug 26 CVE at **T-3d**; this v1.5.94 cycle refreshes it to **T-2d** and adds the next@16.4.0-canary.3 routing-surface finding + @tanstack/react-query@5.102.2 PATCH.

### Aug 26 Critical CVE — Now T-2d (vs T-3d at v1.5.92)

Since v1.5.92 (Aug 23 00:07Z = 36h ago), one material update affects the auth surface:

- **`next@16.4.0-canary.3` SHIPPED Aug 23 23:46Z** (npm-published 2026-08-23T23:46:07.525Z) — the 3rd canary on the 16.4.x line ships a **`[backport] Scope app-entry export`** PR that restricts which exports from the root layout are exposed in the app-entry manifest. This is a security-adjacent backport that closes an information-disclosure vector. **Auth implication**: apps that use custom `output: 'standalone'` and expose auth-related route segments through the app-entry manifest may see behavior changes. Most App Router auth apps are unaffected (standard `output` config). Verify on Aug 26 when the CVE patch ships.
- **`next@canary` is now `16.4.0-canary.3`** — the canary train is at canary.3 (~3 canaries in ~2.5 days). The Aug 26 CVE ships as **`next@16.3.3` + `next@15.5.24`**; canary users should plan to test auth flows after the CVE upgrade.
- **`@tanstack/react-query@5.102.2` SHIPPED** (npm `latest` confirmed at this cron 12:02Z Aug 24 = **PATCH since 5.102.0** tracked in v1.5.92). The `cache-config-types` export fix from 5.102.2 is in `query-core`; no auth-specific API change but React Query consumers (including auth state management via `broadcastQueryClient`) should be on 5.102.2.

### @clerk/nextjs — 7.8.0 STABLE npm-published Confirmed + Canary Still at v20260821144536

**`@clerk/nextjs@7.8.0` npm-published confirmed** via direct registry check at this cron: `curl -s "https://registry.npmjs.org/@clerk/nextjs/latest" | python3` → `{"version": "7.8.0"}`. The auth.md v1.5.92 `npm dist-tags.latest: 7.8.0` observation is confirmed accurate — Snyk's page showed `7.7.9` as stale/incorrect.

**`@clerk/nextjs@canary` still at `7.8.1-canary.v20260821144536`** — verified via `curl -s "https://registry.npmjs.org/@clerk/nextjs/canary" | python3` → `{"version": "7.8.1-canary.v20260821144536"}`. No new drops since Aug 21 14:51Z (the 24th canary drop documented in v1.5.92). The 24h pause on Aug 22 (noted in v1.5.92) + the continued silence on Aug 23–24 suggests the canary train may be stabilizing toward a 7.8.1 STABLE release in the next 1–3 weeks.

### Auth Audit Recipe — Verify Post-Aug 23 Setup + Aug 26 Pre-Flight

```bash
# Step 1: confirm @clerk/nextjs 7.8.0 STABLE npm-published
npm view @clerk/nextjs dist-tags.latest
# Expected: 7.8.0 (confirmed via npm registry at this cron)

# Step 2: confirm @clerk/nextjs@canary tip
npm view @clerk/nextjs dist-tags.canary
# Expected: 7.8.1-canary.v20260821144536 (no new drops since Aug 21)

# Step 3: upgrade @tanstack/react-query to 5.102.2 (PATCH since 5.102.0)
npm view @tanstack/react-query dist-tags.latest
# Expected: 5.102.2
npm install @tanstack/react-query@^5.102.2

# Step 4: verify better-auth is still 1.7.1
npm view better-auth dist-tags.latest
# Expected: 1.7.1 (unchanged since v1.5.79)

# Step 5: Aug 26 CVE pre-flight (T-2d = exactly 48h from this cron)
# Calendar reminder: Aug 26 morning UTC
echo "=== Aug 26 CVE T-2d Pre-flight ==="
echo "Expected: next@16.3.3 + 15.5.24 publish on Aug 26 morning UTC"
echo "Action: npm install next@latest immediately; run npm run dev to verify auth flows"

# Step 6: verify auth middleware still resolves
rg "clerkMiddleware|authMiddleware" src/middleware.ts
# If clerkMiddleware(): should be on @clerk/nextjs 7.8.0 = safe
# If authMiddleware(): plan Core 2/3 migration before Aug 26
```

### Why This Matters for Auth

- **Aug 26 CVE now T-2d** = exactly 48 hours. Every auth-bearing Next.js app needs a calendar reminder for Aug 26 morning UTC. The auth middleware surface is the highest-risk target for any CVE that touches the routing layer.
- **`next@16.4.0-canary.3` `Scope app-entry export` backport** — the first security-adjacent PR in the 16.4.x canary line. Apps with custom `output: 'standalone'` that expose auth-protected route segments via the app-entry manifest should verify their routing behavior on canary.3 before Aug 26.
- **`@clerk/nextjs@7.8.0` npm-published confirmed** — the v1.5.92 observation is accurate. No action needed for Clerk users; 7.8.0 is the recommended pin.
- **`@clerk/nextjs@canary` paused at 7.8.1-canary.v20260821144536** — 3+ days of silence on the canary train suggests stabilization toward a 7.8.1 STABLE. Watch for it in the next 1–3 weeks.
- **`@tanstack/react-query@5.102.2` is the new recommended pin** — the `cache-config-types` export fix from 5.102.2 is in `query-core`. Auth apps that use `broadcastQueryClient` for cross-tab session sync should upgrade to 5.102.2.
- **Better Auth 1.7.1 unchanged** — the Vercel acquisition integration continues. The Agent Auth Protocol roadmap (Q4 2026–Q1 2027) is still pending. No new Better Auth releases since v1.5.79.

### Sources

- [Upcoming Next.js August Security Release](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026) — Aug 20, 2026; one critical CVE; ships Aug 26 as **16.3.3 + 15.5.24**; **T-2d from this cron (Aug 24 12:02Z)**
- [Official v16.4.0-canary.3 release notes](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.3) — npm-published 2026-08-23T23:46:07.525Z; `[backport] Scope app-entry export`
- [`@clerk/nextjs@7.8.0` npm registry confirmed](https://registry.npmjs.org/@clerk/nextjs/latest) — `{"version": "7.8.0"}`; direct curl check at 2026-08-24T12:02Z
- [`@clerk/nextjs@canary` npm registry confirmed](https://registry.npmjs.org/@clerk/nextjs/canary) — `{"version": "7.8.1-canary.v20260821144536"}`; no new drops since Aug 21 14:51Z
- [`@tanstack/react-query@5.102.2` npm registry](https://registry.npmjs.org/@tanstack/react-query/latest) — confirmed at 5.102.2; PATCH since 5.102.0 tracked in v1.5.92
- [Better Auth 1.7.1 npm](https://www.npmjs.com/package/better-auth?activeTab=versions) — unchanged from v1.5.79
- [Clerk Core 3 Upgrade Guide](https://clerk.com/docs/guides/development/upgrading/upgrade-guides/core-3) — most projects upgrade in <30 min using the CLI
- [Clerk Core 2 / Next.js Upgrade Guide](https://clerk.com/docs/guides/development/upgrading/upgrade-guides/core-2/nextjs) — for deprecated `authMiddleware()` (pre-Core 2)
- [Cross-reference: `setup.md` — Aug 26 CVE T-2d setup recipe + canary.3 scope app-entry export
- [Cross-reference: `routing.md` — canary.2 + canary.3 routing-surface PRs + Aug 26 CVE T-2d
- [Cross-reference: `security.md` — full Aug 26 CVE security lens when advisory publishes
- [Cross-reference: `state.md` — @tanstack/react-query@5.102.2 PATCH from the state-management lens

## ★ Aug 26 Critical CVE DROPPED EARLY — Aug 25, 2026 — TWO Unauthenticated RCEs + next@16.3.3 + next@15.5.24 + next@16.4.0-canary.7 + @clerk/nextjs@canary Updated to 7.8.3-canary.v20260825175547 (Auth Lens — v1.5.99, August 25, 2026)

**The Aug 26 CVE dropped ONE DAY EARLY on August 25, 2026.** `next@16.3.3` and `next@15.5.24` shipped at **16:17Z UTC** (published 2026-08-25T16:17:10Z per GitHub API). `next@16.4.0-canary.7` followed at **16:44Z UTC**. The two critical unauthenticated RCEs are now public.

**The CVE escalated from ONE to TWO critical severity vulnerabilities.** The Aug 20 pre-announce said one critical CVE. On Aug 25, Next.js moved the release forward and disclosed **TWO critical unauthenticated RCEs** (GHSA-p293-qw3h-jr36 + GHSA-2xp9-vwfh-vxw4). Both require **no authentication** to exploit.

### The Two Critical CVEs — Auth Implications

**CVE 1 — GHSA-p293-qw3h-jr36: Unauthenticated Remote Code Execution on Windows-hosted servers**
- **Auth implication — HIGH**: RCE means attacker gets full server access. Once an attacker has RCE on the server, they can:
  - Steal `AUTH_SECRET`, `SESSION_SECRET`, database credentials, environment variables
  - Modify `middleware.ts` or API routes to intercept credentials
  - Escalate from unauthenticated network access to full server compromise
  - **No auth token or session needed to exploit** — purely network-accessible endpoint
- **Applies to**: Self-hosted Next.js on Windows servers (Azure App Service Windows, Windows VMs, IIS)
- **Does NOT apply to**: Vercel, Linux servers, Edge Runtime

**CVE 2 — GHSA-2xp9-vwfh-vxw4: Unauthenticated Remote Code Execution in Image Optimization API (AVIF)**
- **Auth implication — HIGH**: RCE via crafted AVIF image upload. Attacker can:
  - Upload a malicious AVIF file that triggers RCE when Next.js optimizes it server-side
  - Achieve full server access — same credential theft as CVE1
  - **No authentication required** — the Image Optimization API is publicly accessible
- **Applies to**: Any Next.js deployment using Next.js Image Optimization with AVIF format
- **Mitigation in canary.7**: AVIF optimization is **disabled** (#97875 `[next/image]: disable avif image optimization`)

### Auth Audit Recipe — IMMEDIATE (Post-CVE-Drop)

```bash
# IMMEDIATE ACTION: upgrade to patched next version now
npm install next@latest
# Expected: installs 16.3.3 (was 16.3.2 before this cron)

# Step 1: identify Windows-hosted deployments (CVE1 applies)
# If your Next.js server runs on Windows:
#   → RCE via path handling — upgrade to 16.3.3 immediately
#   → Then audit: any env secrets on that server are potentially compromised

# Step 2: audit Image Optimization API usage (CVE2 applies to all deployments)
rg "next/image|import.*Image.*from 'next'" --type ts --type tsx | grep -i avif
# If AVIF used: upgrade to 16.3.3 — AVIF disabled in patched version
# If no AVIF: still upgrade for CVE1

# Step 3: if Clerk is in use, verify session integrity post-upgrade
# After upgrading to 16.3.3, test:
#   1. User login — verify session cookie set correctly
#   2. Protected route access — verify middleware auth still works
#   3. Sign-out — verify session invalidated correctly

# Step 4: rotate secrets if Windows-hosted deployment was unpatched
# If you were on next@<16.3.3 on Windows before today:
#   → Rotate AUTH_SECRET, database passwords, API keys
#   → Review server access logs for IOCs

# Step 5: @clerk/nextjs canary check
npm view @clerk/nextjs dist-tags.canary
# Expected: 7.8.3-canary.v20260825175547 (updated since v1.5.98)
```

### @clerk/nextjs — Canary Updated to 7.8.3-canary.v20260825175547 (v1.5.98: 7.8.3-canary.v20260825083614)

**`@clerk/nextjs@canary` jumped from `7.8.3-canary.v20260825083614` to `7.8.3-canary.v20260825175547`** — npm-published 2026-08-25T17:55:47Z (the v1.5.98 cycle tracked `v20260825083614` from 08:36Z; this drop is ~9h later). This is the **28th canary drop since v1.5.50 baseline** (v1.5.98 tracked the 27th). No security-relevant changes in this drop. Pin `@clerk/nextjs@canary` at `7.8.3-canary.v20260825175547`. **STABLE remains `7.8.2`** (unchanged).

### Why This Matters for Auth

- **Two critical unauthenticated RCEs = P0 incident for any Windows-hosted Next.js auth app.** The attacker gets full server access without any credentials. If your Next.js auth app runs on Windows, treat this as a confirmed breach until you upgrade.
- **CVE2 (AVIF RCE) applies to ALL Next.js deployments** using Image Optimization with AVIF. The Image Optimization API is public by default in App Router. Upgrade even if you're on Linux.
- **`next@16.3.3` is now `latest`** — the upgrade is a one-liner. Do it now.
- **@clerk/nextjs canary 28th drop** — no auth changes, but stay current for future stability.
- **Windows ISR fix in canary.7 (#97876)** — if ISR is used on Windows alongside auth, the backslash path handling fix prevents cache poisoning that could leak auth-related pages.

### Sources

- [Official v16.3.3 release](https://github.com/vercel/next.js/releases/tag/v16.3.3) — published **2026-08-25T16:17:10Z**; two critical CVE fixes
- [Official v15.5.24 release](https://github.com/vercel/next.js/releases/tag/v15.5.24) — published **2026-08-25T16:16:55Z**; same CVEs
- [Official v16.4.0-canary.7 release](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.7) — published **2026-08-25T16:44:14Z**; AVIF disabled + Windows ISR fix
- [GHSA-p293-qw3h-jr36 — Unauthenticated RCE on Windows-hosted servers](https://github.com/vercel/next.js/security/advisories/GHSA-p293-qw3h-jr36)
- [GHSA-2xp9-vwfh-vxw4 — Unauthenticated RCE in Image Optimization API with AVIF](https://github.com/vercel/next.js/security/advisories/GHSA-2xp9-vwfh-vxw4)
- [Next.js Security Release Blog Post](https://nextjs.org/blog/nextjs-security-release-august-2026-update) — moved to Aug 25; two critical CVEs confirmed
- [`@clerk/nextjs@canary` npm](https://registry.npmjs.org/@clerk/nextjs/canary) — now `7.8.3-canary.v20260825175547`; 28th drop since v1.5.50
- [Cross-reference: `routing.md` — full CVE routing-surface lens
- [Cross-reference: `security.md` — full CVE security checklist + post-incident response
- [Cross-reference: `setup.md` — immediate upgrade recipe + AVIF disable implications

---

## ★ next@16.4.0-canary.8 SHIPPED + @clerk/nextjs@canary 29th Drop + @tanstack/react-query@5.102.4 PATCH + Aug 26 CVE Post-Incident T+0h (Auth Lens — v1.6.01, August 26, 2026)

**`next@16.4.0-canary.8` SHIPPED** npm-published 2026-08-25T23:46:22Z (~6h before this cron's 06:02Z Aug 26 start). This is the first post-CVE 16.4.x canary. Auth consumers on `next@canary` should pin to `canary.8` for the chained symlink fix + root-anchored import fix.

### @clerk/nextjs — Canary Updated to 7.8.3-canary.v20260825235807 (v1.5.99: 7.8.3-canary.v20260825175547)

**`@clerk/nextjs@canary` jumped from `7.8.3-canary.v20260825175547` to `7.8.3-canary.v20260825235807`** — npm-published 2026-08-25T23:58:07Z (the v1.5.99 cycle tracked `v20260825175547` from 17:55Z; this drop is ~6h later). This is the **29th canary drop since v1.5.50 baseline** (v1.5.99 tracked the 28th). **STABLE remains `7.8.2`** (unchanged). No security-relevant changes in this drop. Pin `@clerk/nextjs@canary` at `7.8.3-canary.v20260825235807`.

### @tanstack/react-query@5.102.4 — PATCH (5th in 7 Days; v1.5.99 Was at 5.102.3)

**`@tanstack/react-query@5.102.4`** (npm `latest` confirmed at this cron) is a **PATCH upgrade** with one targeted fix:

> PR #11293 (`a05df6a`) — "Avoid scheduling stale timeouts for disabled query observers."

This fixes a bug where a query observer that had been disabled could still have a stale timeout scheduled against it, causing unnecessary timer overhead and potential memory retention. The fix ensures disabled observers don't schedule stale timeouts. Auth apps that use React Query for session state, permission queries, or `broadcastQueryClient` for cross-tab sync benefit from this patch.

```bash
# Step 1: upgrade to 5.102.4 (PATCH since 5.102.3)
npm view @tanstack/react-query dist-tags.latest
# Expected: 5.102.4
npm install @tanstack/react-query@^5.102.4

# Step 2: verify TypeScript compatibility
npx tsc --noEmit
# Expected: no new errors

# Step 3: for Clerk + React Query apps — verify session sync
# If using broadcastQueryClient for cross-tab auth:
# Test: open app in two tabs; sign out in one; verify other tab detects signout
```

### Aug 26 CVE Post-Incident — T+0h (Auth Lens)

**Aug 26 00:00Z UTC is now T+0h** — the two CVEs shipped Aug 25 at 16:17Z UTC. The auth-surface incident is resolved.

**Post-incident auth audit checklist**:

1. **If on `next@16.3.3` (recommended)**: auth middleware is CVE-patched. Session integrity is preserved. No auth-specific regression expected.
2. **If on `next@canary` (pinning canary)**: upgrade to `canary.8` now — it includes CVE fixes from `canary.7`.
3. **Windows-hosted Clerk apps**: the Windows ISR fix (#97876) in `canary.7` corrects backslash path handling for ISR pages with special characters. If your ISR pages include auth-protected content with `%2F` in params, test after upgrading.
4. **AVIF Image Optimization in Clerk**: the AVIF optimizer is disabled in all CVE-patched versions. Clerk's default image handling (which uses `next/image`) will fall back to WebP/PNG. No session/auth impact — just image format change.

### Official CVE Identifiers — Now Confirmed

The Aug 25 security blog post confirms the exact CVE identifiers:

| CVE ID | GHSA | Description | Affected |
|---|---|---|---|
| **CVE-2026-75604** | GHSA-p293-qw3h-jr36 | Unauthenticated RCE on Windows-hosted servers | Pages Router + App Router **without Cache Components** on Windows filesystem |
| (no separate CVE) | GHSA-2xp9-vwfh-vxw4 / GHSA-g89c-p67h-r497 | Unauthenticated RCE in Image Optimization API with AVIF | All Next.js deployments using AVIF image optimization |

**Key clarification for auth consumers**: CVE1 explicitly affects "applications using both the Pages Router and App Router **without Cache Components**". Apps using `experimental.cacheLayers` or `use cache` are NOT affected by CVE1. Auth apps on the App Router that have adopted React's cache component patterns are protected by architecture.

### Auth Audit Recipe — Post-CVE + canary.8 + react-query@5.102.4

```bash
# Step 1: confirm next@latest is 16.3.3 (CVE-patched)
npm view next dist-tags.latest
# Expected: 16.3.3

# Step 2: upgrade @clerk/nextjs@canary to 29th drop
npm view @clerk/nextjs dist-tags.canary
# Expected: 7.8.3-canary.v20260825235807
npm install @clerk/nextjs@canary

# Step 3: upgrade @tanstack/react-query to 5.102.4
npm view @tanstack/react-query dist-tags.latest
# Expected: 5.102.4
npm install @tanstack/react-query@^5.102.4

# Step 4: if on next@canary, upgrade to canary.8
npm install next@16.4.0-canary.8

# Step 5: verify auth middleware resolves post-upgrade
rg "clerkMiddleware|authMiddleware" src/middleware.ts
# Expected: clerkMiddleware() on @clerk/nextjs 7.8.x = safe

# Step 6: test session integrity
# 1. Sign in — verify session cookie set
# 2. Navigate protected route — verify middleware allows
# 3. Sign out — verify session invalidated in all tabs (if using broadcastQueryClient)

# Step 7: Windows + ISR + Clerk — verify ISR auth pages with special chars
# If ISR pages use dynamic segments with %2F or special characters:
# Test on canary.8: create ISR page with params containing /
# Expected: page renders correctly with backslash fix from canary.7
```

### Why This Matters for Auth

- **canary.8 is the recommended canary pin for auth apps** — it includes all CVE fixes + the chained symlink fix + root-anchored import fix. No auth-specific regressions.
- **@clerk/nextjs@canary 29th drop** — no auth API changes, but staying current with the canary train reduces the gap to the next STABLE release.
- **react-query@5.102.4** — the stale timeout fix for disabled observers is operationally meaningful for auth apps that conditionally enable/disable session queries based on auth state. No auth API change, but a correctness improvement.
- **CVE1 scope clarified: "without Cache Components"** — auth apps on App Router using `use cache` or `experimental.cacheLayers` are architecturally protected from CVE1 (Windows RCE). This is a significant clarification not available at v1.5.99.

### Sources

- [Official v16.4.0-canary.8 release notes](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.8) — npm-published **2026-08-25T23:46:22Z**
- [Official August 2026 Security Release — full advisory](https://nextjs.org/blog/august-2026-security-release) — CVE-2026-75604 / GHSA-p293-qw3h-jr36 confirmed; "without Cache Components" scope clarification
- [GHSA-p293-qw3h-jr36 — Unauthenticated RCE on Windows](https://github.com/vercel/next.js/security/advisories/GHSA-p293-qw3h-jr36) — CVE-2026-75604
- [GHSA-2xp9-vwfh-vxw4 — AVIF RCE](https://github.com/vercel/next.js/security/advisories/GHSA-2xp9-vwfh-vxw4)
- [`@clerk/nextjs@canary` npm](https://registry.npmjs.org/@clerk/nextjs/canary) — now `7.8.3-canary.v20260825235807`; 29th drop since v1.5.50
- [`@tanstack/react-query@5.102.4` npm](https://registry.npmjs.org/@tanstack/react-query/latest) — PR #11293 stale timeout fix
- [TanStack/query PR #11293 — Avoid scheduling stale timeouts for disabled query observers](https://github.com/TanStack/query/pull/11293) — `a05df6a`
- [Cross-reference: `routing.md` — canary.8 routing-surface PRs
- [Cross-reference: `setup.md` — canary.8 + react-query@5.102.4 setup recipe


---

## ★ next@16.4.0-canary.9 + @clerk/nextjs@canary 30th Drop + better-auth@1.7.2 + @tanstack/react-query@5.102.7 (Auth Lens — v1.6.06, August 27, 2026)

**`next@16.4.0-canary.9`** (npm-published **2026-08-27T00:43:37Z**) ships 22 PRs. Auth-surface routing impact: ReactDOM.browser flag migrations (PR #96826/#96843/#96844) affect `next/dynamic ssr:false`, `useSearchParams`, and PPR resumed renders — all relevant for Clerk-integrated auth pages that use Suspense boundaries.

### @clerk/nextjs@canary — 30th Drop to 7.8.3-canary.v20260827114418 (v1.6.05: 29th Drop at v20260826205903)

**`@clerk/nextjs@canary` jumped from `7.8.3-canary.v20260826205903` to `7.8.3-canary.v20260827114418`** — npm-published **2026-08-27T11:49:30.886Z** (13 minutes before this cron's 12:02Z start). This is the **30th canary drop since v1.5.50 baseline**. STABLE remains `7.8.2`. No auth-API changes. Pin `@clerk/nextjs@canary` at `7.8.3-canary.v20260827114418`.

```bash
npm install @clerk/nextjs@canary
npm view @clerk/nextjs dist-tags.canary
# Expected: 7.8.3-canary.v20260827114418
```

### @tanstack/react-query@5.102.7 — 8th PATCH in 8 Days (Dep Refresh Only; v1.6.05 Tracked 5.102.6)

**`@tanstack/react-query@5.102.7`** (npm-published **2026-08-27T08:33:25.188Z**) is a **dependency refresh** — `query-core@5.102.7`. No API changes.

The **5.102.6** PR (#11305 by @alex-js-ltd) remains the most operationally significant change in the 8-patch sprint:

> **PR #11305**: `useQueries` + `useSuspenseQueries` falsy-error propagation fix. The optional-chain guard `?.error` on `firstSingleResultWhichShouldThrow` silently swallowed falsy errors (`Promise.reject()`, `null`, `''`, `0`). Fix: replace with `if (firstSingleResultWhichShouldThrow)` check. **`useQuery` was never affected** — only `useQueries` and `useSuspenseQueries`.

Auth apps that use `useQueries` for permission checks, role queries, or multi-session state: audit any `onError`/`onSettled` callback that guards with `if (!error) return`. After 5.102.6+, falsy error values now correctly reach the error boundary.

```bash
# Audit for affected patterns
rg "useQueries|useSuspenseQueries" --type tsx | xargs rg "if.*!.*error|if.*error.*return" -A2 -B2
# Look for: if (!error) return  // This now runs for falsy errors too
```

### better-auth@1.7.2 SHIPPED (Auth Lens — v1.6.05 Tracked 1.7.1)

**`better-auth@1.7.2`** npm-published **2026-08-26T19:03:29Z**. A meaningful patch with 10 merged PRs across the better-auth ecosystem. Auth-surface highlights:

**`@better-auth/core` (new in 1.7.2):**
- Fixed async context loss in Cloudflare Workers bundles with multiple runtime conditions ([#10855](https://github.com/better-auth/better-auth/pull/10855))
- Added synchronous and optional access to the current auth endpoint context ([#10938](https://github.com/better-auth/better-auth/pull/10938)) — **NEW API**: `useAuth()` and `auth()` now have an optional/sync variant for cases where the context is not yet initialized
- Auth request logs now respect configured logger, log level, and disabled setting ([#10939](https://github.com/better-auth/better-auth/pull/10939))

**`better-auth` (core):**
- Fixed permanent user bans to clear expiration dates from previous temporary bans ([#10823](https://github.com/better-auth/better-auth/pull/10823)) — **BREAKING CORRECTNESS**: previously, a user who was temporarily banned and then permanently banned would retain the temporary ban's expiration date; now the expiration is cleared
- Added warnings for invalid signed session data in the cookie cache ([#10934](https://github.com/better-auth/better-auth/pull/10934)) — helps debug session cookie corruption
- Allowed `~` in relative callback URLs validated by trusted-origin checks ([#10041](https://github.com/better-auth/better-auth/pull/10041))
- Improved validation of relative callback and redirect URLs with paths, queries, and fragments ([#10979](https://github.com/better-auth/better-auth/pull/10979))
- Allowed same-origin form submissions with `Referrer-Policy: no-referrer` ([#10959](https://github.com/better-auth/better-auth/pull/10959))

**`@better-auth/oauth-provider`:**
- Fixed Client ID Metadata Document registration when clients share at least one supported grant with the server ([#11010](https://github.com/better-auth/better-auth/pull/11010))
- Fixed relative redirect URLs containing fragments ([#10983](https://github.com/better-auth/better-auth/pull/10983))

**`@better-auth/drizzle-adapter`:**
- Fixed one-to-one Drizzle relations when `usePlural` is enabled ([#10941](https://github.com/better-auth/better-auth/pull/10941))

**Migration actions for 1.7.2:**
```bash
# Step 1: upgrade
npm install better-auth@^1.7.2

# Step 2: permanent ban + temporary ban overlap
# If you have users who were temporarily banned and then permanently banned:
# Their session expiration behavior changes — re-test your ban enforcement
SELECT * FROM users WHERE ban_expires_at IS NOT NULL AND permanently_banned = true;
# Expected after upgrade: ban_expires_at is cleared for permanently-banned users

# Step 3: new sync/optional auth context access
# Before: const session = await auth() // throws if not initialized
# After (1.7.2): const session = auth.getOptional() // returns null if not initialized
import { auth } from 'better-auth/auth'
const session = auth.getOptional?.() // new in 1.7.2
```

### Auth Audit Recipe — v1.6.06

```bash
# Step 1: @clerk/nextjs@canary — upgrade to 30th drop
npm view @clerk/nextjs dist-tags.canary
# Expected: 7.8.3-canary.v20260827114418
npm install @clerk/nextjs@canary

# Step 2: better-auth — upgrade to 1.7.2
npm view better-auth dist-tags.latest
# Expected: 1.7.2
npm install better-auth@^1.7.2

# Step 3: react-query — audit useQueries falsy-error patterns
# The 5.102.6 fix (now in 5.102.7) affects useQueries + useSuspenseQueries
rg "useQueries|useSuspenseQueries" --type tsx -l | xargs rg "if.*!.*error|if.*error.*return" -A2 -B2
# Look for guards that will now fire for falsy errors (null, 0, '', false)

# Step 4: Cloudflare Workers + better-auth — test async context
# If your better-auth app runs on Cloudflare Workers:
# The #10855 fix for async context loss with multiple runtime conditions
# May change behavior if you had workarounds for this issue
# Test: deploy to staging, verify auth() resolves correctly in all routes

# Step 5: verify ReactDOM.browser flag migrations (PR #96826/96843/96844)
# If you catch CSRBailout in error boundaries:
rg "CSRBailout" --type ts --type tsx
# Update error handling to account for ReactDOM.browser bailouts

# Step 6: permanent ban + temporary ban overlap (better-auth)
# If your DB has users who are both permanently and temporarily banned:
# Re-test ban enforcement after upgrading better-auth
```

### Why This Matters for Auth

- **@clerk/nextjs@canary 30th drop** — the canary train is healthy. No auth API changes, but the 30th drop confirms the train is moving toward the next STABLE promotion. Clerk apps should pin `@clerk/nextjs@canary` at `7.8.3-canary.v20260827114418`.
- **better-auth@1.7.2 is the most consequential better-auth update since 1.7.0** — the permanent-ban + temporary-ban overlap fix is a correctness improvement; the Cloudflare Workers async context fix affects production CF workers; the new sync/optional auth context API (`auth.getOptional()`) is a new ergonomic pattern for auth-in-middleware scenarios.
- **react-query@5.102.7 is a dep refresh on top of the 5.102.6 correctness fix** — the 5.102.6 PR #11305 change is now in stable and will affect auth apps using `useQueries` for permission or role checks. Audit your `onError`/`onSettled` callbacks.
- **ReactDOM.browser flag migrations** — any Clerk-protected page that uses `next/dynamic ssr:false` + error boundaries is affected by the CSRBailout → ReactDOM.browser migration. Error tracking dashboards need updating.

### Sources

- [Official v16.4.0-canary.9 release notes](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.9) — npm-published **2026-08-27T00:43:37Z**
- [`@clerk/nextjs@canary` npm](https://registry.npmjs.org/@clerk/nextjs/canary) — now `7.8.3-canary.v20260827114418`; 30th drop since v1.5.50
- [`@tanstack/react-query@5.102.7` npm](https://registry.npmjs.org/@tanstack/react-query/latest) — published **2026-08-27T08:33:25.188Z**; dep refresh only
- [TanStack/query PR #11305 — propagate falsy errors to the error boundary](https://github.com/TanStack/query/pull/11305) — merged 2026-08-26T13:27:28Z; @alex-js-ltd
- [better-auth v1.7.2 release](https://github.com/better-auth/better-auth/releases/tag/v1.7.2) — published **2026-08-26T19:03:29Z**; 10 PRs
- [better-auth PR #10855 — async context loss Cloudflare Workers](https://github.com/better-auth/better-auth/pull/10855)
- [better-auth PR #10938 — sync optional auth context](https://github.com/better-auth/better-auth/pull/10938) — **NEW API**
- [better-auth PR #10823 — permanent ban clears temp ban expiration](https://github.com/better-auth/better-auth/pull/10823)
- [PR #96826/96843/96844 — ReactDOM.browser flag migrations](https://github.com/vercel/next.js/pull/96826) — React 3 preparation
- [Cross-reference: `routing.md` — canary.9 routing-surface PRs + PPF shell reclassification
- [Cross-reference: `setup.md` — canary.9 upgrade + better-auth 1.7.2 setup + zod@canary 4.5.0-canary.20260827T054049
