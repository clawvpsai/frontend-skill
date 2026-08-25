

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
