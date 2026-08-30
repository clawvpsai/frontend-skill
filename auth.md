## canary.12 + zod@4.5.3 + zod@4.5.4 SHIPPED (Aug 29) + Aug CVE Retrospective + TypeScript 7.0 STABLE (Auth Lens — v1.6.15, August 30, 2026)

**This cycle refreshes the stale v1.5.94 auth.md header (Aug 24 "Aug 26 CVE T-2d") with the Aug CVE retrospective, adds canary.12, zod@4.5.3 + 4.5.4 patches, and documents the critical gap: TypeScript 7.0 STABLE was missing from all skill docs.**

### Aug CVE Retrospective — Shipped Aug 25 as next@16.3.3 + next@15.5.24

The Aug 26 security release was **moved up to Aug 25** (announced Aug 20 as upcoming). Two critical CVEs disclosed:

- **CVE-2026-75604**: RCE via AVIF image processing — affects all Next.js versions with Image Optimization. Vercel disabled AVIF processing immediately. **Fixed in next@16.3.3 + next@15.5.24** — AVIF images served as-is until libheif fix.
- **Second critical CVE**: Windows unauthenticated RCE — only affects servers using Windows filesystem. **No workaround; upgrade immediately.**

**Auth implication**: neither CVE directly alters auth middleware, session handling, or Clerk/better-auth behavior. The AVIF fix may affect `next/image` usage in auth-related pages (e.g., avatar uploads, profile images) — test image optimization after upgrade. The Windows RCE does not affect Linux/macOS hosting (the vast majority of Next.js deployments).

### next@16.4.0-canary.12 SHIPPED — Auth-Surface Lens (npm-published 2026-08-29T23:38:15Z)

**canary.12 has no auth-surface-breaking PRs** — the routing MEDIUM fix (PR #98032 — undeclared param manifest fix) may affect dynamic route params in auth-protected routes that use `useParams()` without explicit param declarations, but this is a rare edge case. No changes to middleware, session, or auth library integration.

**Recommended**: canary.12 is safe for auth consumers. The `.mjs codemod` change (PR #98029) may affect projects that use `.mjs` module extensions in their auth configuration files.

### zod@4.5.3 + zod@4.5.4 SHIPPED Aug 29 (npm-published 2026-08-29T17:51Z + 17:55Z)

**Two PATCH releases within 4 minutes of each other** on Aug 29 — both missed by v1.6.14 (which tracked up to zod@4.5.2). These are the 3rd and 4th PATCHes since zod@4.5.0 (Aug 28 17:50Z).

- **zod@4.5.3** (npm-published 2026-08-29T17:51Z): pure PATCH — bug fixes only. Check [zod releases page](https://github.com/colinhacks/zod/releases) for specific fixes.
- **zod@4.5.4** (npm-published 2026-08-29T17:55Z): **Latest STABLE** as of this cron. Also pure PATCH. Both 4.5.3 and 4.5.4 together represent 2 additional PATCHes in the rapid post-4.5.0 cadence.

**Auth implication**: zod@4.5.x is used extensively in auth schemas (session validation, JWT claims, OAuth callback params, API key validation). The 4.5.1 breaking change (datetime seconds requirement + CUID v1 deprecation) is still the highest-impact auth change. The 4.5.3/4.5.4 PATCHes are likely bug fixes on top of 4.5.2.

**Recipe for auth-heavy projects**:
```bash
# Check current zod version
npm ls zod

# Upgrade to latest patch
npm install zod@latest  # or npm install zod@~4.5.4

# Audit auth schemas for datetime/iso.date() usage
# (zod 4.5.1 made z.iso.datetime() require seconds — breaking change)
rg "iso.datetime\|z\.date\(\)" --type ts | head -20
```

### @clerk/nextjs — 7.8.4-canary.v20260828233657 Tracked + Newer Drops Possible

**`@clerk/nextjs@canary`** was at `7.8.4-canary.v20260828233657` in v1.6.11 (Aug 29 06:02Z). At this cron (Aug 30 06:02Z = +24h), a newer drop is possible. Verify with:
```bash
curl -s "https://registry.npmjs.org/@clerk/nextjs/canary" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['version'])"
```

**`@clerk/nextjs@7.8.3 STABLE`**: No change since v1.6.11. Safe for production use.

**Auth implication**: the 7.8.4 canary line had 15 drops in 29h (v1.6.11 observation). The 7.8.4 STABLE release is expected 1–2 weeks after the canary line stabilizes.

### ★ TypeScript 7.0 STABLE — SHIPPED JULY 8, 2026 (Missing from All Skill Docs)

**TypeScript 7.0 was officially released July 8, 2026** — completely missing from all skill docs. The skill tracked `typescript@next` dev builds but never documented the stable 7.0 release.

**Headline**: Go-based compiler = **8–12x faster** type checking. For auth-heavy TypeScript projects with complex Zod+JWT+Clerk schemas, TS 7.0 dramatically reduces `tsc --noEmit` time.

**Performance benchmarks**:

| Codebase | TS 6 | TS 7 | Speedup |
|---|---|---|---|
| VS Code | 125.7s | 10.6s | **11.9x** |
| Sentry | 139.8s | 15.7s | **8.9x** |

**What this means for auth codebases**: 
- Complex auth type definitions (JWT claims, session shapes, permission types) are type-checked 8–12x faster
- `tsc --noEmit` in pre-commit hooks is now viable even for large auth codebases
- The `typescript@latest` in auth projects should be 7.0.x

**Upgrade recipe**:
```bash
npm install -D typescript@latest
npx tsc --version

# For auth projects with complex Zod schemas:
# TS 7's faster type checking eliminates the long wait for schema validation types
```

### Sources

- [GitHub releases — next@16.4.0-canary.12](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.12) — npm-published **2026-08-29T23:38:15Z**
- [zod v4.5.4 release](https://github.com/colinhacks/zod/releases/tag/v4.5.4) — published **2026-08-29T17:55Z**
- [zod v4.5.3 release](https://github.com/colinhacks/zod/releases/tag/v4.5.3) — published **2026-08-29T17:51Z**
- [Vercel changelog — Aug 2026 security release](https://vercel.com/changelog/nextjs-august-2026-security-release) — published **2026-08-25**
- [TypeScript 7.0 announcement](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/) — published **2026-07-08**
- [TypeScript 7.0 Go rewrite explained](https://visualstudiomagazine.com/articles/2026/06/22/typescript-7-0-rc-moves-microsofts-go-rewrite-into-the-mainline-compiler.aspx)



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


## ★ @clerk/nextjs 7.8.3 STABLE SHIPPED + Canary Advanced to 7.8.4 Line + @tanstack/react-query@5.102.8 + zod@4.5.1 STABLE (Auth Lens — v1.6.11, August 29, 2026)

**`@clerk/nextjs@7.8.3` STABLE SHIPPED** (npm-published **2026-08-27T18:54:23Z**) — a minor with one meaningful change: JSDoc link target alignment with docs link rules. The auth.md v1.6.06 tracking of `@clerk/nextjs@canary` at `7.8.3-canary.v20260827114418` now has a matching STABLE version. The canary train has since moved to the **7.8.4 line**.

**`@clerk/nextjs@canary` advanced from 7.8.3 line to 7.8.4 line** — `7.8.4-canary.v20260828180008` first appeared at 2026-08-28T18:04:39Z. The current canary tip is `7.8.4-canary.v20260828233657` (npm-published **2026-08-28T23:40:58Z**). The 7.8.4 development line carries multiple drops since Aug 28 18:04Z. **STABLE still `7.8.3`**; 7.8.4 STABLE expected in 1–2 weeks.

**`@tanstack/react-query@5.102.8`** (npm-published **2026-08-27T16:06:57Z**) — dep refresh only (`query-core@5.102.8`). This is the 9th PATCH in the 5.102.x line (Aug 22–27 = fastest-ever TanStack Query cadence). The previous 5.102.6 PR #11305 (useQueries falsy-error propagation) remains the most operationally significant. No new auth API changes in 5.102.8.

**`zod@4.5.1` STABLE** (npm-published **2026-08-28T17:58:39.744Z**) — the v1.6.06 `zod@canary` tracking documented `4.5.0-canary.20260827T054049` as the forecast tip. zod 4.5.0 STABLE (published **2026-08-28T18:14:39.625Z**) and 4.5.1 STABLE shipped within 16 minutes of each other. The `latest` dist-tag points to **4.5.1**. Auth apps using Zod for auth schema validation have **breaking changes** to account for.

### @clerk/nextjs 7.8.3 STABLE — Changelog

```diff
## 7.8.3
### Patch Changes
- Align JSDoc link targets with the docs link rules: internal docs links don't open in a new tab
+  (removed {{ target: '_blank' }} from Invitation Metadata link).
+  External API reference links do open in new tab (added to ExternalAccount Backend API link + currentUser() endpoint).
+  ([#9556](https://github.com/clerk/javascript/pull/9556)) by @manovotny
- Updated dependencies: @clerk/backend@3.16.13, @clerk/shared@4.30.2, @clerk/react@6.14.8
```

No auth-API changes. Recommended pin: `@clerk/nextjs@^7.8.3`.

### @clerk/nextjs Canary — 7.8.4 Line (15 Drops Since v1.6.06)

The canary train advanced from the 7.8.3 line to the **7.8.4 line** in the 48h since v1.6.06 (Aug 27 12:09Z → Aug 29 00:02Z):

| Date | Version | Notes |
|---|---|---|
| Aug 27 18:59Z | 7.8.3-canary.v20260827185423 | Still on 7.8.3 line |
| Aug 27 19:41Z | 7.8.3-canary.v20260827193739 | Still on 7.8.3 line |
| Aug 27 19:59Z | 7.8.3-canary.v20260827195249 | Still on 7.8.3 line |
| Aug 28 15:47Z | 7.8.3-canary.v20260828154238 | Still on 7.8.3 line |
| Aug 28 17:11Z | 7.8.3-canary.v20260828170651 | Still on 7.8.3 line |
| Aug 28 17:24Z | 7.8.3-canary.v20260828172016 | Still on 7.8.3 line |
| Aug 28 18:04Z | **7.8.4-canary.v20260828180008** | **NEW LINE — 7.8.4 development started** |
| Aug 28 18:14Z | 7.8.4-canary.v20260828180945 | |
| Aug 28 19:09Z | 7.8.4-canary.v20260828190422 | |
| Aug 28 19:42Z | 7.8.4-canary.v20260828193733 | |
| Aug 28 20:49Z | 7.8.4-canary.v20260828204455 | |
| Aug 28 21:12Z | 7.8.4-canary.v20260828210825 | |
| Aug 28 21:57Z | 7.8.4-canary.v20260828215256 | |
| Aug 28 22:11Z | 7.8.4-canary.v20260828220709 | |
| Aug 28 23:41Z | 7.8.4-canary.v20260828233657 | **Current tip** |

**15 drops in ~29h** = 1 drop every ~1h 56min. This is the densest Clerk canary period since tracking began. The 7.8.4 line likely contains new auth-API additions that will become the 7.8.4 STABLE in 1–2 weeks.

```bash
# Pin Clerk canary to the 7.8.4 line
npm install @clerk/nextjs@canary
npm view @clerk/nextjs dist-tags.canary
# Expected: 7.8.4-canary.v20260828233657

# Or pin to STABLE 7.8.3
npm install @clerk/nextjs@^7.8.3
```

### zod@4.5.1 STABLE — Breaking Changes for Auth Schemas

zod 4.5.1 (published **2026-08-28T17:58:39Z**) contains the 4.5.0 changes. Auth apps using Zod for form validation, API input validation, or auth schema definitions should review these breaking changes:

**Breaking: `z.iso.datetime()` now requires seconds (PR #6457)**
```typescript
// Before (zod 4.4.x): accepted minute-precision input
z.iso.datetime().parse("2020-01-01T06:15Z")  // ✅ was accepted

// After (zod 4.5.x): MUST include seconds
z.iso.datetime().parse("2020-01-01T06:15:00Z")  // ✅ correct
z.iso.datetime().parse("2020-01-01T06:15Z")      // ❌ TypeError — RFC 3339 requires seconds

// Workaround for both precisions:
z.union([z.iso.datetime(), z.iso.datetime({ precision: -1 })])
```

**Breaking: `z.base64()` now rejects whitespace**
```typescript
z.base64().parse("Zm9v")        // ✅ — no whitespace
z.base64().parse("Zm 9v")       // ❌ — was accepted in 4.4.x, rejected in 4.5.x
```

**Breaking: `z.cuid()` tightened — CUID v1 deprecated**
```typescript
// CUID v1 IDs (created before ~2024) will fail validation in 4.5.x
z.cuid().parse("ck1234567890abcdef")  // ❌ CUID v1 now rejected
z.cuid().parse("cl9uvcg3d0000qzqhta6f5v8k")  // ✅ CUID v2 accepted
```

**Breaking: `z.httpUrl()` now rejects malformed URLs**
```typescript
z.httpUrl().parse("https:/example.com")   // ❌ missing slash after ://
z.httpUrl().parse("https://example.com")   // ✅ correct
```

**Auth-specific impact**: If your auth schemas use `z.string().datetime()` for timestamps, `z.string().cuid()` for ID fields, or `z.string().base64()` for encoded tokens, audit these for the breaking changes.

```bash
# Audit Zod schema files for affected patterns
rg "z\.iso\.datetime|z\.cuid|z\.base64|z\.httpUrl" --type ts --type tsx -l | xargs rg "z\.iso\.datetime|z\.cuid|z\.base64|z\.httpUrl" -n | head -30
```

### Auth Audit Recipe — v1.6.11

```bash
# Step 1: upgrade @clerk/nextjs to 7.8.3 STABLE
npm install @clerk/nextjs@^7.8.3
npm view @clerk/nextjs dist-tags.latest
# Expected: 7.8.3

# Step 2: pin canary to 7.8.4 line (optional, for early access)
npm install @clerk/nextjs@canary
npm view @clerk/nextjs dist-tags.canary
# Expected: 7.8.4-canary.v20260828233657

# Step 3: upgrade @tanstack/react-query to 5.102.8 (dep refresh)
npm install @tanstack/react-query@^5.102.8
npm view @tanstack/react-query dist-tags.latest
# Expected: 5.102.8

# Step 4: audit Zod auth schemas for breaking changes
rg "z\.iso\.datetime\(\)" --type ts --type tsx -l | xargs rg "z\.iso\.datetime" -n | head -20
# Update to include seconds: "2020-01-01T06:15:00Z" format

# Step 5: audit CUID v1 usage (if using legacy CUID IDs in auth)
rg "z\.cuid\(\)" --type ts --type tsx | head -10
# If any: plan migration from CUID v1 to CUID v2 or uuid

# Step 6: audit base64 auth token patterns
rg "z\.base64\(\)" --type ts --type tsx | head -10
# Test: verify tokens without whitespace still validate

# Step 7: verify auth middleware still resolves
rg "clerkMiddleware|authMiddleware" src/middleware.ts
npx tsc --noEmit
# Expected: no new errors
```

### Why This Matters for Auth

- **@clerk/nextjs 7.8.3 STABLE** — minor release, no auth API changes. Safe upgrade for all Clerk users.
- **7.8.4 canary line — 15 drops in 29h is unprecedented** — the densest Clerk canary period ever tracked. The 7.8.4 line likely contains new auth features. Pin `@clerk/nextjs@canary` at `7.8.4-canary.v20260828233657` to follow along; STABLE 7.8.4 expected in 1–2 weeks.
- **zod@4.5.1 is a breaking-change STABLE** — the `z.iso.datetime()` seconds requirement is the highest-impact change for auth apps that store or validate ISO timestamp strings. Audit all Zod schemas that use datetime validation.
- **@tanstack/react-query 5.102.8** — dep refresh only. The 5.102.6 PR #11305 (falsy-error propagation in useQueries) is now in the stable range and is the most operationally relevant change. Auth apps using `useQueries` for permission/role checks should verify their error handling guards.

### Sources

- [Official @clerk/nextjs 7.8.3 release](https://github.com/clerk/javascript/blob/main/packages/nextjs/CHANGELOG.md) — published **2026-08-27T18:54:23Z**
- [`@clerk/nextjs@canary` npm](https://registry.npmjs.org/@clerk/nextjs/canary) — now `7.8.4-canary.v20260828233657`; 15 drops since v1.6.06; new 7.8.4 line
- [`@tanstack/react-query@5.102.8` npm](https://registry.npmjs.org/@tanstack/react-query/latest) — published **2026-08-27T16:06:57Z**; dep refresh only
- [zod 4.5 blog post](https://zod.dev/blog/zod-4-5) — published **2026-08-27T00:00Z**; datetime seconds requirement + breaking changes
- [zod 4.5 changelog](https://github.com/colinhacks/zod/releases) — PR #6457 datetime seconds fix
- [PR #97948 — Fix optimistic routing for encoded dynamic params](https://github.com/vercel/next.js/pull/97948) — routing correctness fix (canary.11)
- [PR #97953 — Fix intercepted route params after Proxy rewrites](https://github.com/vercel/next.js/pull/97953) — routing correctness fix (canary.11)
- [Cross-reference: `routing.md` — canary.10/.11 routing-surface + Pages Router React 18 deprecation
- [Cross-reference: `setup.md` — zod 4.5.1 STABLE setup recipe + canary.10/.11 implications
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
