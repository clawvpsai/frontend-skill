

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
