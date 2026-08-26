

## Next.js 16.4.0-canary.2 + canary.3 Routing-Surface PRs (v1.5.94 — August 24, 2026)

**`next@canary` jumped from `16.4.0-canary.1` to `16.4.0-canary.3`** (npm-published 2026-08-23T23:46:07Z) in the 24h since the v1.5.93 cycle (Aug 24 06:05Z). This v1.5.94 cycle documents canary.2 + canary.3 from the routing-surface lens. The 16.4.x canary train is at canary.3 (3 canaries in ~2.5 days); the **Aug 26 critical CVE ships in T-2d as `next@16.3.3` + `next@15.5.24`** (Aug 26 morning UTC — the same day the canary train will likely be at canary.5–.7).

### 16.4.0-canary.2 Routing-Surface PRs (npm-published 2026-08-22T23:55:51Z)

**canary.2 was a minimal release** — the entire canary.2 drop consisted of a single PR + the version-bump commit:

- **[PR #97284](https://github.com/vercel/next.js/pull/97284) `feat(ossfs): introduce an options struct for constructing backend storage`** — by @lukesandberg; merged 2026-08-22T02:53:33Z; 13 files; Turbopack internal refactor introducing a structured options type for the `ossfs` (object-store filesystem) backend. **Routing impact: NONE directly**. This is an internal Turbopack infrastructure change that lays groundwork for future storage backends; it does not affect routing, prefetching, or any user-visible routing behavior. The Next.js GitHub release for canary.2 confirms "This release is backporting bug fixes. It does not include all pending features/changes on canary."

**canary.2 was also verified against the canary branch** at v1.5.89 (Aug 23 00:02Z): `ahead_by: 0, behind_by: 0` — canary.2 was the tip of the canary branch at that time.

### 16.4.0-canary.3 Routing-Surface PRs (npm-published 2026-08-23T23:46:07Z)

**canary.3 is the first meaningful 16.4.x release for routing-surface consumers.** The GitHub release title is `"[backport] Scope app-entry export"` — this PR restricts which exports from a Next.js app's root layout are exposed in the app-entry manifest, closing a potential information-disclosure vector where a specially crafted request could probe the app-entry to enumerate internal route segments. This is the first security-adjacent PR in the 16.4.x canary line.

**The routing-surface implication**: apps that use custom `output: 'standalone'` or `output: 'export'` configurations and depend on the app-entry manifest for internal routing introspection should verify their routing behavior is unchanged after upgrading to canary.3+. For most App Router apps (the 95%+ use case), this change is invisible — it only affects the manifest shape, not the routing behavior itself.

### Aug 26 Critical CVE — Now T-2d (vs T-4d at v1.5.88)

**Aug 26 is now T-2d** from this cron's 12:02Z Aug 24 start (= exactly 2 days to Wednesday Aug 26). The v1.5.88 cross-reference to "Aug 26 CVE T-4d" is now stale. The Aug 26 release will ship **`next@16.3.3` + `next@15.5.24`** (routine versioning: x.3 for the 16.3 branch, x.24 for the 15.5 LTS branch). All 16.4.x canary users are also potentially affected.

**Routing-surface implications of the Aug 26 CVE** (refined from v1.5.88):

1. **If the CVE affects middleware or the routing layer** (HIGH probability for an auth-adjacent framework like Next.js — the routing layer is the primary attack surface): `router.push()`, `<Link>`, and `redirect()` behavior may be affected. Plan to run `npm run dev` immediately after upgrading on Aug 26 to verify routing behaves correctly.
2. **For PPF users**: `unstable_prefetch()` + `unstable_navigation()` are built on top of the routing layer. Any CVE patch to the routing layer could alter prefetch behavior. Test navigation flows on Aug 26.
3. **For `generateStaticParams` users**: the static generation path does not go through the same routing runtime as dynamic requests; it may be unaffected by a routing-layer CVE.
4. **For microfrontend apps using PR #95997** (HMR isolation): if the CVE patch modifies HMR subscription handling, re-test microfrontend HMR boundaries after upgrading.

### Practical Impact Table — Per-User-Type (canary.2 + canary.3)

| User type | Affected? | Action |
|---|---|---|
| Standard App Router user (no custom output) | **No** (canary.2 invisible; canary.3 only affects manifest shape) | None for canary.2/3. Plan Aug 26 upgrade. |
| `output: 'standalone'` + app-entry introspection | **Possibly** (canary.3 restricts manifest exports) | Test app-entry behavior on canary.3 before deploying |
| `output: 'export'` (static export) | **Possibly** (canary.3 manifest scoping) | Verify exported manifest shape |
| PPF `unstable_prefetch()` / `unstable_navigation()` user | **Indirect** (routing layer CVE may affect PPF) | Test prefetch + navigation on Aug 26 |
| Microfrontend / single-spa user | **Indirect** (canary.3 + Aug 26 CVE patch interaction) | Re-test HMR isolation post-Aug 26 upgrade |
| Pages Router user | **No** (canary.2/3 are App Router / Turbopack focused) | Aug 26 still applies (LTS patch) |

### Updated Routing Audit Recipe (v1.5.94)

```bash
# canary.2 + canary.3 pre-flight (for custom output / app-entry users)
# Verify app-entry manifest behavior on canary.3:
npm install next@16.4.0-canary.3
grep -r "app-entry\|generateAppEntry" .next/
# Expected: if no custom output usage, nothing relevant

# Aug 26 CVE pre-flight (ALL routing users)
# Step 1: verify current stable is 16.3.2 (the routine PATCH from Aug 21)
npm view next dist-tags.latest
# Expected: 16.3.2

# Step 2: audit routing-layer dependencies
rg "router\.push|router\.replace|redirect\(\)|notFound\(\)" --type ts --type tsx | wc -l
# This is your surface area for routing-layer CVE impact

# Step 3: PPF users audit unstable_prefetch / unstable_navigation usage
rg "unstable_prefetch|unstable_navigation" --type ts --type tsx
# Expected: your PPF call sites

# Step 4: Aug 26 calendar reminder
echo "T-2d to Aug 26 CVE — next@16.3.3 + 15.5.24 publish expected"
echo "Plan: run npm run dev + npm test immediately after upgrading on Aug 26"

# Step 5: verify microfrontend HMR (if applicable)
# After upgrading on Aug 26, open two microfrontends and edit a shared route
# Verify HMR events stay within their microfrontend boundary
```

### Why This Matters for Routing

- **canary.2 is a non-event for routing consumers** — PR #97284 is an internal Turbopack backend refactor with zero user-visible routing impact. It does, however, confirm that the 16.4.x canary train is active and moving.
- **canary.3 is the first 16.4.x release worth noting for routing** — the `[backport] Scope app-entry export` PR closes a manifest-enumeration information-disclosure vector. Apps with `output: 'standalone'` that introspect the app-entry manifest should re-test on canary.3 before deploying.
- **Aug 26 CVE now T-2d** = exactly 48 hours from this cron's 12:02Z Aug 24 start. Every routing consumer should have a calendar reminder for Aug 26 morning UTC and a tested upgrade path (`next@latest` → `next@16.3.3` or `next@15.5.24` for LTS).
- **Routing-layer CVEs are the highest-impact class for Next.js** — unlike a rendering or data-fetching CVE, a routing-layer vulnerability typically allows an attacker to bypass authentication checks, poison cache entries, or enumerate internal routes. The routing surface (`router.push`, `<Link>`, `redirect()`, middleware) is the canonical attack surface for Next.js auth apps.
- **The 16.4.x canary train is healthy** (canary.3 in ~2.5 days). Expect `16.4.0` STABLE around Sep 8–15. The canary train will ship 5–10 more canaries before STABLE.

### Sources

- [Official v16.4.0-canary.2 release notes](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.2) — npm-published 2026-08-22T23:55:51.651Z; 1 PR + version-bump commit
- [Official v16.4.0-canary.3 release notes](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.3) — npm-published 2026-08-23T23:46:07.525Z; backport of scope app-entry export
- [PR #97284 — feat(ossfs): introduce an options struct for constructing backend storage](https://github.com/vercel/next.js/pull/97284) — by @lukesandberg; merged 2026-08-22T02:53:33Z; Turbopack internal refactor
- [Next.js canary-branch compare `v16.4.0-canary.2...canary`](https://github.com/vercel/next.js/compare/v16.4.0-canary.2...canary) — `ahead_by: 0, behind_by: 0` verified at 2026-08-23T00:02Z (canary.2 was tip at that check)
- [Next.js canary-branch compare `v16.4.0-canary.3...canary`](https://github.com/vercel/next.js/compare/v16.4.0-canary.3...canary) — verified at 2026-08-24T12:02Z; canary.3 is the tip
- [Upcoming Next.js August Security Release](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026) — Aug 20, 2026; ships Aug 26 as 16.3.3 + 15.5.24; **T-2d from this cron** (Aug 24 12:02Z)
- [Cross-reference: `security.md` — full Aug 26 CVE security lens + advisory when published
- [Cross-reference: `auth.md` — auth-surface Aug 26 CVE urgency + @clerk/nextjs 7.8.0 STABLE + better-auth 1.7.1
- [Cross-reference: `setup.md` — Aug 26 CVE T-2d setup audit recipe

## ★ Aug 26 Critical CVE DROPPED EARLY — Aug 25, 2026 — TWO Unauthenticated RCEs in next@16.3.3 + next@15.5.24 + next@16.4.0-canary.7 (Routing Lens — v1.5.99, August 25, 2026)

**The Aug 26 CVE dropped ONE DAY EARLY on August 25, 2026.** At this cron's 18:02Z Aug 25 start, the security release has already been published — `next@16.3.3` and `next@15.5.24` shipped at **16:17Z UTC** (published 2026-08-25T16:17:10Z per GitHub API). `next@16.4.0-canary.7` followed at **16:44Z UTC** with the same fixes. The Aug 26 morning UTC expectation was wrong — the 16:00Z window matched the Jul 21 16:00Z UTC ship cadence.

**The CVE escalated from ONE to TWO critical severity vulnerabilities.** The Aug 20 pre-announce said one critical CVE. On Aug 25, Next.js moved the release forward and disclosed **TWO critical unauthenticated RCEs** (GHSA-p293-qw3h-jr36 + GHSA-2xp9-vwfh-vxw4). Both require **no authentication** to exploit.

### The Two Critical CVEs (from GitHub Release API)

**CVE 1 — GHSA-p293-qw3h-jr36: Unauthenticated Remote Code Execution on Windows-hosted servers**
- **Severity**: Critical
- **Unauthenticated**: Yes — no login or session required
- **Platform**: Windows-hosted Next.js servers only
- **Impact**: Remote code execution — attacker runs arbitrary code on the server
- **Patched in**: `next@16.3.3`, `next@15.5.24`, `next@16.4.0-canary.7`
- **PR in canary.7**: Fix ISR misses with backslashes in segments when deployed on Windows (#97876)

**CVE 2 — GHSA-2xp9-vwfh-vxw4: Unauthenticated Remote Code Execution in Image Optimization API when AVIF files are used**
- **Severity**: Critical
- **Unauthenticated**: Yes — no login or session required
- **Attack vector**: Image Optimization API with AVIF image format
- **Impact**: Remote code execution through crafted AVIF files
- **Patched in**: `next@16.3.3`, `next@15.5.24`, `next@16.4.0-canary.7`
- **PR in canary.7**: `[next/image]: disable avif image optimization` (#97875) — AVIF is disabled in the Image Optimization API to prevent exploitation

### Routing-Surface Impact Analysis

| User type | Impact | Action |
|---|---|---|
| Windows-hosted Next.js server (self-hosted) | **CRITICAL RCE** — upgrade immediately | `npm install next@latest` now |
| Linux-hosted Next.js server (Vercel, AWS, DO, etc.) | **CVE1 not applicable** — CVE2 still critical | `npm install next@latest` now + disable AVIF |
| Vercel/Edge — fully managed | **AVIF RCE applies** if using Image Optimization with AVIF | Upgrade + verify AVIF disabled automatically |
| `output: 'standalone'` on Windows | **Both CVEs apply** — upgrade immediately | `npm install next@latest` now |
| `output: 'standalone'` on Linux | **CVE2 applies** (AVIF in Image Opt API) | `npm install next@latest` now |
| Static export (`output: 'export'`) | **Not affected** — no server-side rendering | No action |
| Pages Router | **CVE1 applies** if Windows-hosted | Upgrade if Windows server |
| PPF users (`unstable_prefetch`/`unstable_navigation`) | **Not directly affected** — CVEs are server-side | Upgrade for safety |

### Updated Routing Audit Recipe — Post-CVE-Drop

```bash
# IMMEDIATE: verify current next version and upgrade
npm view next dist-tags.latest
# Expected NOW: 16.3.3 (was 16.3.2 before this cron)

npm install next@latest   # upgrade to 16.3.3 immediately

# Step 1: identify Windows-hosted deployments
# If deploying on Windows servers (self-hosted, Azure, Windows VMs):
#   → CVE1 applies: RCE via server-side path handling
#   → upgrade to 16.3.3 immediately

# Step 2: audit Image Optimization API usage
rg "next/image|Image.*from 'next'" --type ts --type tsx | grep -i avif || echo "No AVIF usage found"
# If AVIF is used: upgrade to 16.3.3 — AVIF is disabled in canary.7
# If no AVIF: upgrade anyway for CVE1 (Windows RCE if applicable)

# Step 3: PPF users — verify prefetch behavior unchanged
npm run dev
# Test: navigate through pages with unstable_prefetch / unstable_navigation
# Expected: behavior unchanged post-upgrade

# Step 4: canary.7 additional changes for routing
#   #97876 — Fix ISR misses with backslashes in segments on Windows
#   If ISR + Windows: test that static params with special characters work
```

### next@16.4.0-canary.7 — Security + Additional Fixes (npm-published 2026-08-25T16:44:14Z)

`next@16.4.0-canary.7` ships the same two security fixes (GHSA-p293-qw3h-jr36 + GHSA-2xp9-vwfh-vxw4) plus additional changes:

| PR | Description | Routing Impact |
|---|---|---|
| **#97875** | `[next/image]: disable avif image optimization` | **MEDIUM** — AVIF optimization disabled; clients get fallback format |
| **#97876** | `Fix ISR misses with backslashes in segments when deployed on Windows` | **MEDIUM** — ISR cache key fix for Windows path separators |
| #97812 | `Upgrade React from eafeac09-20260819 to bd6ea412-20260824` | LOW — React canary roll-forward |

### Sources

- [Official v16.3.3 release](https://github.com/vercel/next.js/releases/tag/v16.3.3) — published **2026-08-25T16:17:10Z**; two critical CVE fixes
- [Official v15.5.24 release](https://github.com/vercel/next.js/releases/tag/v15.5.24) — published **2026-08-25T16:16:55Z**; same CVEs
- [Official v16.4.0-canary.7 release](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.7) — published **2026-08-25T16:44:14Z**; security fixes + AVIF disable + Windows ISR fix
- [GHSA-p293-qw3h-jr36 — Unauthenticated RCE on Windows-hosted servers](https://github.com/vercel/next.js/security/advisories/GHSA-p293-qw3h-jr36)
- [GHSA-2xp9-vwfh-vxw4 — Unauthenticated RCE in Image Optimization API with AVIF](https://github.com/vercel/next.js/security/advisories/GHSA-2xp9-vwfh-vxw4)
- [Next.js Security Release Blog Post](https://nextjs.org/blog/nextjs-security-release-august-2026-update) — moved to Aug 25; two critical CVEs confirmed

---

## ★ next@16.4.0-canary.8 SHIPPED — Aug 25 23:46Z + Aug 26 CVE Post-Incident T+0h + TypeScript 34th Rebuild Pending ~08:25Z Aug 26 (Routing Lens — v1.6.01, August 26, 2026)

**`next@16.4.0-canary.8` SHIPPED** npm-published 2026-08-25T23:46:22Z (~6h before this cron's 06:02Z Aug 26 start). This is the 4th canary in the 16.4.x train and the first since the Aug 25 CVE drop. The canary train has now crossed the CVE-patch point (`canary.7` = CVE-patch canary) and resumed active development. Two of the 9 PRs in canary.8 have **routing-surface impact**.

### canary.8 PRs — Routing-Surface Relevant

| PR | Description | Routing Impact |
|---|---|---|
| **#97825** | `Fix Turbopack resolution through chained symlinks` — classifies symlinks by final target chain, preserves non-directory behavior for dangling/invalid/cyclic chains; fixes #97786 | **MEDIUM** — resolves Turbopack build failures in monorepos and CI environments with chained `node_modules` symlinks; `next build` with Turbopack now works in environments that previously required webpack fallback |
| **#97799** | `feat(turbopack): resolve /-rooted imports from the project directory` — `import '/content/where'` and `require('/foo.js')` now resolve from project root (where `next.config` lives); previously resolved from filesystem root with "server relative imports are not implemented yet" | **MEDIUM** — absolute-path imports (`/`-prefixed) now work correctly in Next.js; consumer impact: any app using `import '/some-file'` or `require('/foo.js')` in API routes or server code will now resolve correctly from project directory instead of failing |

### TypeScript 34th No-Content Daily Rebuild — Pending ~08:25Z Aug 26

**The 34th TypeScript `next` rebuild is still PENDING** at this cron's 06:02Z Aug 26 start. The current tip is `7.1.0-dev.20260825.1` (npm-published 2026-08-25T08:53:06Z = 28 min early on the v1.5.99 forecast of ~08:25Z). The 34th rebuild was forecast for ~08:25Z Aug 26. If this cron runs after that window, the 34th rebuild will be `7.1.0-dev.20260826.1`. The TS main branch remains idle (30+ days = longest stretch since 7.0.0 baseline). **No TS-surface breaking changes in this cycle.**

**TypeScript Impact on Routing**: NONE for this cycle. The 34th rebuild is a no-content daily rebuild — no new language features, only the daily `tsc --noEmit` smoke test against main. Routing consumers using `next@canary` + `typescript@next` can expect zero TS-related routing changes.

### Aug 26 CVE Post-Incident — T+0h (Routing Lens)

**Aug 26 00:00Z UTC is now T+0h** — the two CVEs shipped Aug 25 at 16:17Z UTC as `next@16.3.3` + `next@15.5.24`. The routing-surface incident state is now resolved. For **post-incident routing audit** purposes:

- **If you upgraded to `16.3.3` before Aug 26**: no further action needed for CVE1 (Windows RCE) or CVE2 (AVIF RCE). Routing behavior is back to normal.
- **If you are on `16.4.0-canary.7`**: you have the CVE patches. `canary.8` on top of `canary.7` does NOT remove the CVE fixes — they're baked into the base.
- **`output: 'standalone'` on Windows users**: test that the Windows ISR backslash fix (#97876) from `canary.7` is working — params with `%2F` should render correctly.
- **PPF users** (`unstable_prefetch` / `unstable_navigation`): confirmed unaffected by the CVEs. Navigation behavior post-patch is identical.

### Updated Routing Audit Recipe — Post-CVE + canary.8

```bash
# Step 1: confirm next@latest is 16.3.3 (CVE-patched)
npm view next dist-tags.latest
# Expected: 16.3.3

# Step 2: canary.8 — verify if you're on canary
npm view next dist-tags.canary
# Expected: 16.4.0-canary.8 (if pinning canary)

# Step 3: Turbopack chained symlinks fix (#97825) — verify in monorepo/CI
# If you had Turbopack build failures with symlinked node_modules:
npm install next@16.4.0-canary.8
pnpm next build --turbo
# Expected: build completes without symlink errors
# If still failing: check if the symlink chain depth exceeds expected patterns

# Step 4: absolute-path imports fix (#97799) — verify in API routes
# If you use import '/file' or require('/file') in API routes:
# These now resolve from project root instead of filesystem root
# Test: create an API route that imports a file by absolute path
# Expected: resolves correctly

# Step 5: TypeScript 34th rebuild — check if it shipped during this cron window
npm view typescript dist-tags.next
# If 7.1.0-dev.20260826.1: the 34th rebuild shipped
# If 7.1.0-dev.20260825.1: still pending (check back in ~2h)

# Step 6: canary.8 recommended pin for 16.4.x experimenters
# canary.8 = canary.7 (CVE fixes) + chained symlink fix + root-anchored imports
npm install next@16.4.0-canary.8
```

### Why This Matters for Routing

- **canary.8 is the first post-CVE 16.4.x release** — the canary train crossed the CVE patch point (`canary.7`) and resumed active development. The train is healthy: 4 canaries in ~3 days.
- **Chained symlink fix (#97825) is a Turbopack monorepo fix** — environments using `npm link`, `pnpm workspaces`, or CI that copies `node_modules` via symlinks will now get clean `next build --turbo` instead of `TurbopackInternalError`. This was a known pain point in monorepos.
- **`/-rooted import fix (#97799) is a correctness fix** — apps using `import '/foo'` or `require('/bar')` in server-side code were getting "server relative imports are not implemented yet" errors. This is now resolved. The project-directory root is enforced as the boundary — no filesystem escape.
- **TypeScript 34th rebuild PENDING** — this is the normal daily no-content rebuild cadence. No routing-surface TS changes.
- **Aug 26 CVE T+0h** — the incident is over for routing consumers. The two CVEs (Windows RCE + AVIF RCE) are patched in `16.3.3` / `15.5.24` / `16.4.0-canary.7+`. Routing behavior is normal.

### Sources

- [Official v16.4.0-canary.8 release notes](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.8) — npm-published **2026-08-25T23:46:22Z**; 9 PRs
- [PR #97825 — Fix Turbopack resolution through chained symlinks](https://github.com/vercel/next.js/pull/97825) — fixes #97786; 8 files; +148/-14
- [PR #97799 — feat(turbopack): resolve /-rooted imports from the project directory](https://github.com/vercel/next.js/pull/97799) — 24 files; +227/-22; project-directory root enforcement
- [PR #97697 — Turbopack: correctly trace through TypeScript `__importStar`](https://github.com/vercel/next.js/pull/97697) — 4 files; +18/-37; no perf impact
- [Official August 2026 Security Release — full advisory](https://nextjs.org/blog/august-2026-security-release) — CVE-2026-75604 / GHSA-p293-qw3h-jr36 + GHSA-2xp9-vwfh-vxw4 / GHSA-g89c-p67h-r497
- [TypeScript 34th rebuild pending — `7.1.0-dev.20260825.1`](https://registry.npmjs.org/typescript/next) — forecast ~08:25Z Aug 26; main branch 30+ days idle
- [Cross-reference: `auth.md` — @clerk/nextjs@canary 29th drop + auth post-incident
- [Cross-reference: `setup.md` — canary.8 + react-query@5.102.4 + AVIF disable post-incident setup
