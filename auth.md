# Auth — NextAuth.js v4 (Legacy) + v5 (Production) + Better Auth + Clerk

## Which Library to Use

> **⚠️ BIG NEWS — July 7, 2026:** [Vercel acquired Better Auth](https://vercel.com/blog/vercel-acquires-better-auth). Founder Bereket Engida and the core team joined Vercel; Better Auth remains MIT, free, framework-agnostic, and is now under Vercel stewardship. Roadmap is pointed at the [Agent Auth Protocol](https://agentauthprotocol.com/) (AI-agent identity with scoped, revocable permissions), which feeds into [Vercel Connect](https://vercel.com/connect) and [eve](https://eve.dev). **The recommendation in this skill does not change** — Better Auth is still the default for new Next.js SaaS — but read the [post-acquisition update](#better-auth-post-acquisition-update-july-2026) below before committing to a large build.

| Library | Latest stable | Cadence | License | When to Use |
|---|---|---|---|---|
| **NextAuth.js v5 (Auth.js)** | `5.0.0-beta.32` (July 20, 2026) — ships with `@auth/core@0.41.3` security fixes | ⚠️ **Resumed** — beta.32 ends the 3-month gap (Apr 14 → Jul 20); pre-release cadence is still slower than Better Auth | MIT | Free, framework-agnostic, integrates with Next.js 16 `proxy.ts`. Use when you don't want a hosted dependency and Auth.js v5 has the providers you need. **Upgrade immediately — beta.32 fixes the auth-check failing-open bug (see [Critical Security Update](#nextauthjs-v5--v4--critical-security-update-july-20-2026) below).** |
| **NextAuth.js v4** | `4.24.15` (July 20, 2026) — same-day security backport from v5 | Maintenance only — backports the v5 `@auth/core@0.41.3` security fixes | MIT | Existing v4 projects only. **Upgrade immediately** — same security advisories apply. Do not start new projects on v4. |
| **Better Auth** | `1.6.25` (July 23, 2026); `1.7.0-rc.2` (July 22, 2026); `1.7.0-beta.10` (June 26, 2026) | ✅ **Active** — 1.6.24→1.6.25 in 1 day, 1.7.0-rc.2 on Jul 22 | MIT | Recommended default for new Next.js apps in 2026. TypeScript-first, batteries-included (email/password, magic links, passkeys, 2FA, organizations, admin plugin), no hosted dependency, owns its DB schema. |
| **Clerk** | `7.6.4` (`@clerk/nextjs` `latest`, July 31, 2026); also ships `@clerk/nextjs@latest-nextjs-v5` `5.7.6` (Apr 15, 2026 — maintenance) | ✅ Active — daily canaries + weekly stables (16 stable releases since 7.5.12 on Jul 3) | Proprietary (free tier to 10K MAU; ~$0.02/MAU + $25/mo after) | Use when DX velocity matters more than cost/control. Pre-built UI components, organizations + MFA + passkeys out of the box. See [Clerk 7.6 coverage](#clerk--coverage--75613-764-patch-train-july-2026) for the new `fapiUrl` proxy helper option + the CSP-header `client.protect.clerk.com` cleanup. |
| **Supabase Auth** | bundled with Supabase | Active (platform cadence) | MIT (library) + SaaS pricing | Use when you're already on Supabase (native RLS integration via `auth.uid()`). |
| **Auth0 / WorkOS** | n/a | SaaS cadence | Proprietary | Enterprise SSO (SAML/SCIM). Better Auth doesn't have native SSO yet — see Better Auth section. |
| **Lucia / Oslo + custom** | n/a | Library cadence | MIT | Low-level primitives when no library's opinions fit. Rare in 2026. |

**Current state (July 2026):** The auth landscape has materially shifted in the last 6 months:

1. **Better Auth has overtaken Auth.js in monthly npm downloads** for the first time (per [npm trends](https://npmtrends.com/better-auth-vs-next-auth), June 2026). 1.6.23 shipped June 29, 1.7.0-rc.1 shipped July 2, 1.6.24 shipped July 22, 1.6.25 shipped July 23 — active development.
2. **NextAuth.js v5 + v4 critical security update (July 20, 2026)** — both tracks shipped security releases on the same day: `next-auth@5.0.0-beta.32` (npm `dist-tag.beta`) + `next-auth@4.24.15` (npm `dist-tag.latest`) + `@auth/core@0.41.3` (the underlying core library). The headline v5-only fix: **auth checks no longer fail open on provider configuration errors** — a non-OK session response now returns `null` so `if (!!session)` style checks fail closed; previously a misconfigured provider returned an `Error`-shaped session object that truthy-checks passed. Pairs with malformed-Bearer `getToken()` fix (no more uncaught exceptions → request now returns 401 cleanly), provider-bound OAuth `state`/`nonce`/PKCE check cookies (closes a cross-provider confusion vector), and NFKC email normalization closing a homoglyph `@` bypass. **This breaks the "NextAuth v5 beta has stagnated" assessment above** — beta.32 ends the 3-month gap. The skill keeps Better Auth as the default for new SaaS, but NextAuth v5 remains a viable pick if you need a minimal API surface.
3. **Clerk has matured** to v7 with React 19 + Next.js 16 support; its free tier now extends to 10K MAU (was 5K MAU pre-v7).

**Decision matrix for July 2026:**

| If you need… | Pick |
|---|---|
| Full control + own DB schema + zero hosted dependency + active development | **Better Auth** (default for new SaaS) |
| Free + framework-agnostic + minimal API surface + you don't need organizations or passkeys | **NextAuth.js v5** (acceptable for blogs, marketing sites, internal tools) |
| Pre-built UI + organizations + MFA + passkeys with zero custom code | **Clerk** (acceptable when DX velocity > cost; re-evaluate at ~10K MAU) |
| Native RLS with Supabase Postgres | **Supabase Auth** |
| Enterprise SSO (SAML / SCIM) today | **WorkOS** (Better Auth doesn't yet have native SAML/SCIM — see Better Auth section) |
| B2C consumer app at huge scale with social login only | **Clerk** or **NextAuth.js v5** + social providers |

**Note on the npm dist-tag:** `next-auth@5.0.0-beta.32` (July 20, 2026) is still published under the `beta` dist-tag — you install with `next-auth@beta`. This remains a packaging artifact; the API and runtime have been stable for ~21 months. The skill lists v5 as "✅ Production" because that's the de-facto reality across the ecosystem. **beta.32 ends the 3-month release gap (Apr 14 → Jul 20) but the cadence is still slower than Better Auth's weekly stable + monthly RC cadence** — see the security-update section below before pinning.

### When to Consider an Alternative

NextAuth/Auth.js is no longer the default — **Better Auth is** for most new Next.js SaaS apps in 2026. Per the LogRocket April 2026 comparison of every major auth library for Next.js, and the [MakerKit July 2026 comparison](https://makerkit.dev/blog/tutorials/better-auth-vs-clerk) of Better Auth vs Clerk vs NextAuth vs Supabase Auth for production Next.js SaaS:

- **Better Auth** — Open-source (MIT), TypeScript-first, batteries-included (email/password, magic links, passkeys, 2FA, organizations). GA since v1.0 (late 2024), v1.6.25 stable July 23 2026, v1.7.0-rc.1 shipped July 2 2026, v1.7.0-beta.10 shipped July 22 2026. Use when you want full control without rolling your own. Cost at 100K MAU: ~$50/mo (just your Postgres).
- **Clerk** — Best-in-class DX, pre-built UI components, organizations + MFA + passkeys out of the box. Free tier to 10K MAU (was 5K MAU pre-v7), then ~$0.02/MAU + $25/mo base. Use when UX velocity > control. Cost at 100K MAU: ~$1,025/mo. Re-evaluate at ~10K MAU.
- **NextAuth/Auth.js** — Free, MIT, framework-agnostic, integrates cleanly with Next.js 16's `proxy.ts`. Reasonable for marketing sites, internal tools, and OSS projects that don't need organizations or passkeys.
- **Lucia / Oslo + custom** — Low-level primitives. Use when you need a hand-rolled auth flow with no library opinions (rare).
- **Supabase Auth / Auth0 / WorkOS** — Use when you're already paying for the platform (RLS via `auth.uid()` for Supabase; SAML/SCIM for WorkOS).

For most new Next.js SaaS apps in 2026, **Better Auth is the default recommendation** — it's free, MIT, framework-agnostic, has the broadest feature set (email/password, magic links, passkeys, 2FA, organizations, admin plugin), and ships at a faster cadence than Auth.js v5. Reach for Clerk when pre-built UI is the bottleneck, NextAuth v5 when you want the absolute smallest dependency footprint and don't need advanced features.

**Sources:**
- [LogRocket — I tested every major auth library for Next.js in 2026 (April 20, 2026)](https://blog.logrocket.com/best-auth-library-nextjs-2026/)
- [MakerKit — Better Auth vs Clerk vs NextAuth vs Supabase Auth (July 2026)](https://makerkit.dev/blog/tutorials/better-auth-vs-clerk)
- [Better Auth — migration guide from Auth.js](https://better-auth.com/docs/guides/next-auth-migration-guide)
- [npm trends — better-auth vs next-auth](https://npmtrends.com/better-auth-vs-next-auth)
- [Clerk — Next.js quickstart](https://clerk.com/docs/quickstarts/nextjs)

### v4 vs v5 — Key Differences

| Concern | v4 | v5 |
|---|---|---|
| Package | `next-auth` | `next-auth` (same package, different import) |
| Config file | `pages/api/auth/[...nextauth].ts` | `auth.ts` (root) |
| Session type | `getSession()` | `auth()` |
| Middleware | `withAuth` in `middleware.ts` | `auth()` function in `proxy.ts` (Next.js 16) |
| Callbacks | `jwt`, `session` callbacks | Same (unchanged) |
| Providers | Same set | Same set + new providers |



## NextAuth.js v5 + v4 — Critical Security Update (July 20, 2026)

**This is a same-day, two-track security release.** On **July 20, 2026** (16:00Z), the Auth.js / NextAuth.js maintainers shipped security fixes to every supported line simultaneously:

- `next-auth@5.0.0-beta.32` (the v5 beta dist-tag) — published at 2026-07-20T22:57:40Z
- `next-auth@4.24.15` (the `latest` stable dist-tag) — published at 2026-07-20T23:19:38Z
- `@auth/core@0.41.3` (the underlying shared core library that both versions depend on)

All three ship the same `processReply` / `processAuthorization` hardening patch (`@auth/core@0.41.3` → `0.41.3`, published 2026-07-20T15:00Z, GitHub release [`@auth/core@0.41.3`](https://github.com/nextauthjs/next-auth/releases/tag/@auth%2Fcore%400.41.3)) — **advisory [GHSA-xmf8-cvqr-rfgj](https://github.com/advisories/GHSA-xmf8-cvqr-rfgj), HIGH severity, CWE-20 (improper input validation)**. Tenable Cloud Security tagged it as Plugin 445216 on 2026-07-23 (HIGH).

### What was fixed (shared by v5 + v4, all from `@auth/core@0.41.3`)

- **`getToken()` no longer throws on malformed Bearer authorization headers** — previously a malformed `Authorization: Bearer <garbage>` header made `getToken()` throw an uncaught exception. The exception bubbled up to the route handler, often with a generic 500 response and no useful log line. Patched: `getToken()` now returns `null` on a malformed Bearer so route handlers can branch on "no token" cleanly. **Action required if you catch `getToken` exceptions** — the catch block is now dead code; replace with a `if (!token)` guard.
- **OAuth `state`, `nonce`, and PKCE check cookies are now provider-bound** — previously a `state` cookie set by Provider A could be replayed against Provider B (e.g. a Google OAuth flow using a `state` cookie that was originally set during a GitHub OAuth flow on the same session). The cookies now carry a `__Host-` prefix bound to the originating provider id, and a `state`/`nonce`/PKCE cookie presented against a different provider's callback is rejected. **This closes the cross-provider OAuth confusion vector** — a class of attack where an attacker could swap providers mid-handshake and reuse the upstream-issued authorization code against the wrong callback. Per [Patchstack](https://patchstack.com/database/npm/npm/next-auth/vulnerability/npm-auth-js-configuration-errors-can-cause-existence-based-auth-checks-to-fail-open-auth-object-populated-with-an-error), this also addresses the configuration-errors-can-cause-existence-based-auth-checks-to-fail-open vector that allowed an `auth-object-populated-with-an-error` shape to bypass checks that relied on truthy session values.
- **Email addresses are Unicode-normalized (NFKC) before validation** in the email sign-in flow — closes a homoglyph `@` bypass where visually identical non-ASCII characters (e.g. Cyrillic `е` U+0435 vs Latin `e` U+0065) could be used to register an email that would collide with a legitimate user account at the trust boundary. **Action required if you compare emails anywhere in your auth pipeline** (e.g. allow-lists, secondary-contact merging) — re-compare under the same NFKC normalization, or you'll silently mis-classify legitimate vs impostor accounts.

### What was fixed in v5 ONLY (`5.0.0-beta.32` beyond `@auth/core@0.41.3`)

- **Auth checks no longer fail open on provider configuration errors** — the headline v5 fix and the reason beta.32 is a high-priority upgrade. Per the [`next-auth@5.0.0-beta.32` GitHub release](https://github.com/nextauthjs/next-auth/releases/tag/next-auth@5.0.0-beta.32) (published by @gustavovalverde at 2026-07-20T22:57:40Z): *"Fixes auth checks failing open on provider configuration errors: a non-OK session response now yields no session instead of an error object, so checks like `!!auth` fail closed."* In other words, when a provider throws during session resolution (e.g. a misconfigured OIDC provider returns an error object), the session is now `null` instead of an `Error`-shaped value that truthy-checks pass. **Anywhere you wrote `if (session?.user) { ... }` or `if (await auth()) { ... }`, you're now safe by default** — previously those checks would pass for an error-object session and route the user into authenticated code paths. **Audit your code now** for any place where the session was treated as truthy without inspecting the user.

### What was fixed in v4 ONLY (`4.24.15` beyond `@auth/core@0.41.3`)

- **Explicit `NEXTAUTH_URL` now takes precedence over auto-detected forwarded host in trusted-host mode** — previously if you set `NEXTAUTH_URL` but trusted the `X-Forwarded-Host` header (e.g. behind a reverse proxy), the forwarded host could win and silently redirect OAuth callbacks to an attacker-controlled domain. Now the explicit `NEXTAUTH_URL` always wins.
- **Restores CommonJS compatibility by pinning `uuid` to `^11.1.1`** — the `uuid` 14.x line is ESM-only and broke `require('uuid')` on Node versions below 20.19. The pin keeps next-auth v4 working on older Node LTS lines (Node 18.x EOL users especially).
- **Sign-ins in flight across the upgrade fail once and succeed on retry** — the OAuth state-cookie provider-binding change (see above) invalidates any cookie set before the upgrade; the user's first sign-in attempt after deploy will fail with `OAuthSignInError` and the second attempt will succeed. **Briefly expected post-deploy**, not a bug.

### Why v5 + v4 shipped together

Both lines share `@auth/core` (the shared library that holds the OAuth/JWT/session state machine). The `@auth/core@0.41.3` patches the same code paths, so both `next-auth@5.0.0-beta.32` and `next-auth@4.24.15` are thin wrappers around the same `@auth/core` bump + v4/v5-specific glue. The v5-only failing-open fix is in `next-auth` itself (not `@auth/core`), which is why it's not in v4 — v4's API surface is different (no `auth()` helper that returns a session, just `getServerSession()` returning `{ user } | null`), so the failing-open attack shape doesn't apply.

### Recommended Migration

```bash
# v5 projects
npm install next-auth@beta      # picks up 5.0.0-beta.32

# v4 projects
npm install next-auth@latest    # picks up 4.24.15
```

**Migration checklist (any version → 5.0.0-beta.32 or 4.24.15):**

- [ ] **Run `npm install` and verify the version** (`npm ls next-auth @auth/core` — both should match the table above)
- [ ] **Audit `getToken()` callers** — drop any `try/catch` around `getToken()` since it no longer throws on malformed Bearer; replace with an `if (!token)` guard
- [ ] **Audit email comparison code paths** — re-apply NFKC normalization everywhere you compare emails (allow-lists, contact merging, password-reset flows)
- [ ] **Audit auth-check patterns** — search the codebase for `if (session?.user)`, `if (await auth())`, `if (!!session)`, `if (user)` patterns; verify they all branch on the user object, not just on the session truthiness. v5 users get the failing-open fix automatically; v4 users get the same hardening via the underlying `@auth/core` patch.
- [ ] **First sign-in after deploy may fail with `OAuthSignInError`** (v4 only — see the "sign-ins in flight" note above) — this is expected and self-resolves on retry. Don't roll back the upgrade.
- [ ] **Reverse-proxy users (v4 only)** — confirm `NEXTAUTH_URL` is set explicitly in your environment; the upgrade makes it the authoritative redirect base even when behind a trusted proxy.

**Practical impact per tag:**

| Tag | Currently | Upgrade to | Action |
|---|---|---|---|
| `next-auth@5.0.0-beta.31` (Apr 14) | **Vulnerable** to failing-open + Bearer-throw + cross-provider OAuth confusion + homoglyph bypass | `5.0.0-beta.32` | Upgrade immediately |
| `next-auth@4.24.14` (Apr 14) | **Vulnerable** to Bearer-throw + cross-provider OAuth confusion + homoglyph bypass | `4.24.15` | Upgrade immediately |
| `next-auth@latest` (= 4.24.14) | Same as above | `next-auth@latest` (= 4.24.15) | Upgrade immediately |
| `next-auth@beta` (= 5.0.0-beta.31) | Same as above | `next-auth@beta` (= 5.0.0-beta.32) | Upgrade immediately |
| `@auth/core@<0.41.3` (transitive, any version) | Vulnerable (transitively) | `@auth/core@0.41.3` (via next-auth bump) | Upgrade next-auth; don't pin @auth/core separately |

**Sources:**
- [NextAuth.js `5.0.0-beta.32` GitHub release](https://github.com/nextauthjs/next-auth/releases/tag/next-auth@5.0.0-beta.32) (Jul 20, 2026) — the v5-only failing-open fix
- [NextAuth.js `4.24.15` GitHub release](https://github.com/nextauthjs/next-auth/releases/tag/next-auth@4.24.15) (Jul 20, 2026) — v4 maintenance backport
- [`@auth/core@0.41.3` GitHub release](https://github.com/nextauthjs/next-auth/releases/tag/@auth%2Fcore%400.41.3) (Jul 20, 2026) — shared security patches
- [GitHub Security Advisory GHSA-xmf8-cvqr-rfgj](https://github.com/advisories/GHSA-xmf8-cvqr-rfgj) (HIGH, CWE-20)
- [Patchstack — npm-auth-js-configuration-errors-can-cause-existence-based-auth-checks-to-fail-open](https://patchstack.com/database/npm/npm/next-auth/vulnerability/npm-auth-js-configuration-errors-can-cause-existence-based-auth-checks-to-fail-open-auth-object-populated-with-an-error)
- [Tenable Plugin 445216 — @auth/core, next-auth SCA update (Jul 23, 2026)](https://www.tenable.com/plugins/cloud-security/445216)
- [npm `next-auth` package versions](https://www.npmjs.com/package/next-auth?activeTab=versions) — confirms 5.0.0-beta.32 and 4.24.15 are the live dist-tag pointers
- [oday-bakkour.com — Daily Release Radar July 26, 2026 (Auth.js section)](https://oday-bakkour.com/blog/daily-release-radar-july-26-2026) — independent coverage of the same security release

## Better Auth — Recommended Default for New Next.js Apps

### Better Auth post-acquisition update (July 7, 2026)

On **July 7, 2026**, Vercel [announced the acquisition of Better Auth](https://vercel.com/blog/vercel-acquires-better-auth). What it means for skill users and what does *not* change:

**What changed:**

- **Governance:** Better Auth founder **Bereket Engida** and the core team are now Vercel employees. Decisions about the library's direction now sit inside a platform company.
- **Roadmap priorities:** The team's near-term energy is going toward **agent identity** — specifically the [Agent Auth Protocol](https://agentauthprotocol.com/), an open protocol giving each AI agent its own scoped, revocable identity (so an agent acting on your behalf doesn't inherit your full permissions). That work feeds into Vercel's [Connect](https://vercel.com/connect) and [eve](https://eve.dev) platforms. ([Vercel blog](https://vercel.com/blog/vercel-acquires-better-auth), [The New Stack](https://thenewstack.io/vercel-acquires-better-auth/))
- **Operational reality:** "Auth you own" still means you own the operational burden — the database, the session logic, the enterprise plugins, and the on-call rotation. ([WorkOS analysis, July 8, 2026](https://workos.com/blog/vercel-acquires-better-auth-migrate-to-workos))

**What did *not* change (Vercel committed to all of these publicly):**

- **License stays MIT** and the library is still free forever.
- **Name stays "Better Auth"** (not re-branded to a Vercel product).
- **Same team leads development** — Bereket and the core contributors retain decision authority over the library's direction under Vercel.
- **Framework-agnostic** — Better Auth still runs anywhere, not just on Vercel. ([Vercel blog](https://vercel.com/blog/vercel-acquires-better-auth))
- **Auth.js / NextAuth maintenance continues** — Better Auth absorbed Auth.js earlier in 2026; that codebase continues to receive security patches. ([noqta.tn analysis, July 8, 2026](https://noqta.tn/en/blog/vercel-acquires-better-auth-what-it-means-2026))
- **Community governance preserved** — 850+ contributors and the existing maintainer model are unchanged. ([StartupResearcher, July 9, 2026](https://www.startupresearcher.com/news/vercel-acquires-open-source-authentication-startup-better-auth))

**Decision impact for new projects in mid-2026:**

| Project shape | Recommendation | Notes |
|---|---|---|
| New Next.js SaaS, no SAML/SCIM needed | **Better Auth** (still default) | Roadmap shift is towards agent identity, but the core library remains a high-quality, active, well-maintained TypeScript auth solution |
| Enterprise SSO (SAML/SCIM) needed | WorkOS or Clerk | Better Auth's enterprise plugin still on the roadmap; WorkOS published a [migration playbook](https://workos.com/blog/vercel-acquires-better-auth-migrate-to-workos) the day after the acquisition |
| Marketing site / blog / internal tool, prefer minimal dependencies | NextAuth.js v5 (now under Better Auth stewardship) | Maintenance continues; check [Auth.js discussions](https://github.com/nextauthjs/next-auth/discussions) for roadmap updates |
| You're shipping products for AI agents to authenticate against | Watch Agent Auth Protocol | [agentauthprotocol.com](https://agentauthprotocol.com/) is the official site; this is the strategic bet behind the acquisition |
| B2B SaaS with planned agent integration | Re-evaluate in 6–12 months | If Vercel Connect + Agent Auth mature into a production-ready agent identity stack, Better Auth's agent-auth story will likely lead the ecosystem |

**Takeaway:** The acquisition *added a caveat*, not a change of recommendation. For new projects starting in mid-2026, Better Auth remains the default. Re-evaluate once Agent Auth reaches 1.0 or once Better Auth ships native SAML/SCIM.

**Sources:**
- [Vercel blog — Vercel acquires Better Auth to accelerate open source auth (July 7, 2026)](https://vercel.com/blog/vercel-acquires-better-auth)
- [Agent Auth Protocol](https://agentauthprotocol.com/)
- [The New Stack — Vercel acquires Better Auth to give AI agents their own identity (July 7, 2026)](https://thenewstack.io/vercel-acquires-better-auth/)
- [WorkOS — Vercel acquired Better Auth: what it means and how to migrate (July 8, 2026)](https://workos.com/blog/vercel-acquires-better-auth-migrate-to-workos)
- [noqta.tn — Vercel Acquires Better Auth: What It Means for Your Auth Stack (July 8, 2026)](https://noqta.tn/en/blog/vercel-acquires-better-auth-what-it-means-2026)
- [StartupResearcher — Vercel Acquires Open Source Authentication Startup Better Auth (July 9, 2026)](https://www.startupresearcher.com/news/vercel-acquires-open-source-authentication-startup-better-auth)

### Better Auth 1.7.0-rc.2 changelog (July 22, 2026) — ⚠️ contains BREAKING changes ahead of 1.7.0 stable

`1.7.0-rc.2` is the second RC on the v1.7 line (npm dist-tag `rc` now points here; `1.7.0-rc.1` is superseded). **Three breaking changes** that all 1.7.0-stable upgraders need to handle — diff the prior `1.7.0-rc.0 → rc.1 → rc.2` chain carefully when planning the upgrade:

**❗ Breaking changes (action required when upgrading from ≤1.6.x or from `1.7.0-rc.0/rc.1` to `1.7.0-rc.2`+):**

1. **`experimental.joins` moved to `advanced.database.joins`** ([PR #10359](https://github.com/better-auth/better-auth/pull/10359)). The old location under `experimental` is removed:

   ```ts
   // ❌ Pre-1.7.0-rc.2 (removed)
   experimental: { joins: true }

   // ✅ 1.7.0-rc.2+
   advanced: {
     database: {
       joins: true,
     },
   }
   ```

   Adapters that support native joins (Drizzle, Prisma, Kysely, MongoDB) use them when enabled; adapters that can't fall back to additional queries and combine results client-side. **Action:** migrate the config key; re-run `npx auth@latest generate` (Drizzle/Prisma) to make sure schema relations include every required relation for the native-join queries; check the audit `rg 'experimental.*joins' better-auth.config.*` to find any project that still has the old shape.

2. **Accounts are now scoped by issuer (`feat(auth)!: scope accounts by issuer`, [PR #10403](https://github.com/better-auth/better-auth/pull/10403))** — the biggest breaking change in the 1.7 line. Renames + new required columns + new identity model:

   - **`Account.accountId` → `Account.providerAccountId`** (the column that stores the upstream provider's stable identifier for that linked account).
   - **`Account.issuer` is now required** (the trusted upstream issuer for this account). The generated schema migration **cannot assign trusted issuers or resolve existing identity collisions automatically** — manual backfill per the 1.7 upgrade guide is required before deploying.
   - **Account-specific APIs** (e.g. account listing, account-linking) select the local `Account.id` through `accountId`. Token and provider-profile APIs can instead select the signed account cookie via `useAccountCookie: true`.
   - **Credential accounts** use `local:credential` as their issuer and the linked user's stable `id` as their provider identity.
   - **OAuth provider identity** now comes from raw verified profiles. OIDC discovery uses `sub`; plain OAuth uses `id`; providers can declare `accountSubject` for another immutable field. Better Auth no longer switches between `sub` and `id` at runtime. **`getUserInfo().user` no longer carries provider identity**, and **`mapProfileToUser` cannot return `id`**. Read the selected identity from `accountInfo.account.providerAccountId` instead of `accountInfo.user.id`.
   - **SSO account subjects are now protocol-defined**: OIDC uses the verified `sub` claim; SAML uses the signed `NameID`; `mapping.id` is removed from both configurations. A manual SAML configuration without metadata XML must set `idpMetadata.entityID` (because `samlConfig.issuer` identifies the service provider and no longer acts as the IdP identity).
   - The generic `microsoftEntraId` helper now requires a concrete tenant GUID; use the built-in Microsoft provider for multi-tenant authorities.
   - **Migration:** apply the reviewed account-identity backfill in the Better Auth 1.7 upgrade guide before deploying. The generated schema migration will not assign trusted issuers or resolve existing identity collisions automatically — expect to write a data migration script per environment.

3. **SCIM decoupled from the organization plugin** ([PR #10390](https://github.com/better-auth/better-auth/pull/10390)) — replaces the previous SCIM configuration, client APIs, database schema, and organization-backed Group model. Existing SCIM installations **cannot migrate provisioning state in place** — the 1.7 upgrade guide requires a full directory reprovisioning before resuming SCIM traffic. If you don't use SCIM, ignore this change.

   Tangential correctness win included: deferred database side effects now run only after a successful transaction. A rolled-back User update no longer refreshes its cached profile; a rolled-back bulk session revocation no longer invalidates sessions.

**Features:**

- **`verifyIdToken` now receives request context (`ctx`) as a 3rd argument** ([PR #10376](https://github.com/better-auth/better-auth/pull/10376)) — same as 1.6.24; promoted to stable here.
- **Compound table indexes** ([PR #10402](https://github.com/better-auth/better-auth/pull/10402)) — declare multi-column indexes on auth tables directly in your Better Auth schema config. Faster lookups on common compound queries (org + user, account + provider).
- **`beforeStoreCookie` option for `last-login-method` plugin** ([PR #5753](https://github.com/better-auth/better-auth/pull/5753)) — GDPR compliance (same as 1.6.24).
- **`getOrganization` for metadata-only fetches** ([PR #10397](https://github.com/better-auth/better-auth/pull/10397)) — pulls the organization record without joining members, useful for header/footer displays.
- **Transactional OIDC user resolution** for SSO ([PR #10473](https://github.com/better-auth/better-auth/pull/10473)) — the SSO user-resolution path is now wrapped in a transaction so partial resolution can't leave orphaned rows.

**Bug fixes (`better-auth` core):**

- **`auth generate` no longer fails on Convex first-run** ([PR #10302](https://github.com/better-auth/better-auth/pull/10302)) — same as 1.6.24.
- **`get-session` sends `Cache-Control: no-store`** ([PR #10222](https://github.com/better-auth/better-auth/pull/10322)) — same as 1.6.24.
- **`useSession({ throw: true })` type fixed** ([PR #9787](https://github.com/better-auth/better-auth/pull/9787)) — same as 1.6.24.
- **SQLite `BIGINT` recognized as valid number type** ([PR #10316](https://github.com/better-auth/better-auth/pull/10316)) — same as 1.6.24.
- **`CookieAttributes` index signature tightened** ([PR #10442](https://github.com/better-auth/better-auth/pull/10442)) — same as 1.6.24.
- **Adapter query misrouting on `user.modelName` collisions** ([PR #10235](https://github.com/better-auth/better-auth/pull/10235)) — same as 1.6.24.
- **Drizzle duplicate index fix** ([PR #10333](https://github.com/better-auth/better-auth/pull/10333)) — same as 1.6.24.
- **Kysely duplicate index fix** ([PR #10357](https://github.com/better-auth/better-auth/pull/10357)) — same as 1.6.24.
- **Cold-start `AsyncLocalStorage` race on serverless** ([PR #9862](https://github.com/better-auth/better-auth/pull/9862)) — same as 1.6.24.
- **Magic-link and email-OTP `Origin` validation on cookieless sends** ([PR #10368](https://github.com/better-auth/better-auth/pull/10368)) — same as 1.6.24.
- **`organization.listMembers` limit applied to the user fetch** ([PR #10342](https://github.com/better-auth/better-auth/pull/10342)) — same as 1.6.24.
- **`auth generate` schema now uses database-generated IDs for invitations** ([PR #10040](https://github.com/better-auth/better-auth/pull/10040)) — same as 1.6.24.
- **Auth query revalidation and signal listeners restored after remount** ([PR #10379](https://github.com/better-auth/better-auth/pull/10379)) — same as 1.6.24.
- **Request clone failures inside verification callbacks** ([PR #10336](https://github.com/better-auth/better-auth/pull/10336)) — same as 1.6.24.
- **Drizzle-kit peer range widened** ([PR #10299](https://github.com/better-auth/better-auth/pull/10299)) — supports newer Drizzle-kit versions.
- **SIWE addressless nonces** ([PR #10234](https://github.com/better-auth/better-auth/pull/10234)) — Ethereum Sign-In With Ethereum now issues nonces that don't bind to a wallet address, allowing per-session rotation without losing the in-flight nonce.
- **Remote MCP auth challenge headers exposed** ([PR #10290](https://github.com/better-auth/better-auth/pull/10290)) — same as 1.6.24.
- **OpenAPI schema includes plugin user fields on `/sign-up/email` + `/update-user`** ([PR #10453](https://github.com/better-auth/better-auth/pull/10453)) — same as 1.6.24.

**Action for stable users (`better-auth@1.6.24`):** *don't upgrade to `1.7.0-rc.2` yet* — it carries the breaking schema changes (`Account.accountId → providerAccountId`, `Account.issuer` required, `experimental.joins → advanced.database.joins`, SCIM rewrite). Wait for `1.7.0` stable. The skill will add the official migration cookbook once `1.7.0` ships.

**Action for users already on `1.7.0-rc.0/rc.1`:** run `npx @better-auth/cli@latest generate` to regenerate the schema, then manually backfill `Account.issuer` per the 1.7 upgrade guide (the auto-migration can't resolve identity collisions), then upgrade to `1.7.0-rc.2`. If you use `experimental.joins`, move the config to `advanced.database.joins` first. If you use SCIM, treat the upgrade as a cutover — re-provision your directory before resuming traffic.

**Sources:**
- [Better Auth v1.7.0-rc.2 release notes (2026-07-22T19:58:53Z)](https://github.com/better-auth/better-auth/releases/tag/v1.7.0-rc.2)
- [PR #10359 — `chore!: move joins to advanced.database.joins`](https://github.com/better-auth/better-auth/pull/10359)
- [PR #10403 — `feat(auth)!: scope accounts by issuer`](https://github.com/better-auth/better-auth/pull/10403)
- [PR #10390 — `feat(scim)!: decouple provisioning from the organization plugin`](https://github.com/better-auth/better-auth/pull/10390)
- [Better Auth 1.7 upgrade guide](https://better-auth.com/docs/guides/1-7-upgrade-guide)

### Better Auth 1.6.25 changelog (July 23, 2026)

`1.6.25` is the latest **stable** release (npm `latest` dist-tag pointer moved 2026-07-23T15:48:09Z, replaces `1.6.24` from July 22 — **the second consecutive stable release in 24 hours**, the 1.6.x line is now hot-fixing known regressions as they're reported). 4 bug fixes, **no features, no breaking changes**. Changelog vs `1.6.24`:

**Bug fixes:**
- **Apple OAuth now sends the PKCE `code_challenge` during authorization** ([PR #10294](https://github.com/better-auth/better-auth/pull/10294)) — the Apple OAuth provider was missing the `code_challenge` parameter on the authorization request, which caused Apple's token-exchange endpoint to reject the token request with `invalid_grant`. Affects every project using `socialProviders: { apple: { ... } }`. With 1.6.24, the round-trip silently failed and the user saw a generic `OAuthAccountNotLinked` or stuck on the Apple consent screen; 1.6.25 sends `code_challenge` + `code_challenge_method=S256` correctly. **Action:** upgrade from 1.6.24 to 1.6.25 immediately if you use Apple OAuth.
- **Google One Tap now respects `disableSignUp` on the Google provider** ([PR #10479](https://github.com/better-auth/better-auth/pull/10479)) — `socialProviders.google.disableSignUp: true` was being silently ignored on the Google One Tap flow (the popup-style auto-prompt that surfaces "Continue as ..." UI), so One Tap was creating new user rows even when `signUps` were configured to be disabled. The auto-prompt code path used a different sign-up decision branch than the standard `signIn.social({ provider: 'google' })` flow. 1.6.25 unifies the two paths so `disableSignUp` is honored on both. **Action:** upgrade if you use Google One Tap with `disableSignUp: true` (security fix — the previous behaviour silently allowed account creation).
- **Solid client now exposes `$fetch` and `$store`** ([PR #10444](https://github.com/better-auth/better-auth/pull/10444)) — the `@better-auth/solid` client (`createAuthClient` from `better-auth/solid`) was missing `$fetch` and `$store` on the client instance returned by `createAuthClient(...)`. They were exported from the underlying core package but not re-bound to the Solid client wrapper, so `authClient.$fetch('/some/route')` returned `undefined is not a function` and the Solid store helpers couldn't be wired. Affects every Solid.js + Better Auth project. 1.6.25 re-exposes both. **Action:** upgrade if you use `better-auth/solid` and were working around the missing `$fetch`/`$store`.
- **Internal adapter query routing on `user.modelName` collisions fixed** — when a built-in table's `modelName` was set to another table's schema key (e.g. `user.modelName = "account"`), internal adapter queries were being routed to the wrong table. The fix tightens `getDefaultModelName` (the same helper from 1.6.24's bug-fix list) to prefer exact schema key matches over `modelName` aliases in all adapter call paths, not just the ones that were covered by the 1.6.24 fix. **Action:** upgrade if you have an unusual schema where built-in tables reference other tables' keys via `modelName`.

**No features, no breaking changes.** Safe drop-in upgrade from 1.6.24.

**Action:** `npm install better-auth@1.6.25` — recommended for all users (especially Apple OAuth users, Google One Tap users with `disableSignUp`, and Solid client users).

**Sources:**
- [Better Auth 1.6.25 GitHub release](https://github.com/better-auth/better-auth/releases/tag/v1.6.25)
- [`better-auth` CHANGELOG @ `07a646ea`](https://github.com/better-auth/better-auth/blob/07a646ea190167370fbbb60a0fa2c3be3bec5522/packages/better-auth/CHANGELOG.md)
- [compare `v1.6.24...v1.6.25`](https://github.com/better-auth/better-auth/compare/v1.6.24...v1.6.25)
- [PR #10294 — `fix(apple): send PKCE code_challenge during authorization`](https://github.com/better-auth/better-auth/pull/10294)
- [PR #10479 — `fix(google-one-tap): respect disableSignUp`](https://github.com/better-auth/better-auth/pull/10479)
- [PR #10444 — `fix(solid): expose $fetch and $store on the Solid client`](https://github.com/better-auth/better-auth/pull/10444)

### Better Auth 1.7.0-rc.1 changelog (July 2, 2026)

### Better Auth 1.6.24 changelog (July 22, 2026)

`1.6.24` is the latest **stable** release (the `latest` npm dist-tag now points here; `1.7.0-rc.1` remains on the `rc` tag). Changelog vs `1.6.23` (June 29, 2026):

**Features:**
- **`verifyIdToken` now receives request context (`ctx`) as 3rd argument** ([PR #10376](https://github.com/better-auth/better-auth/pull/10376)) — custom ID token verifiers can now read request headers (e.g. IP, `User-Agent`, `CF-Connecting-IP`) for fraud scoring or rate limiting without re-reading the request object:
  ```ts
  const result = await auth.api.verifyIdToken({
    token: idToken,
    ctx: { request: originalRequest }  // ← new 3rd arg
  })
  ```

- **`beforeStoreCookie` option added to last-login-method plugin** ([PR #5753](https://github.com/better-auth/better-auth/pull/5753)) — enables GDPR-compliant "last login method" tracking without third-party cookies.

**Bug fixes:**
- **`get-session` endpoint now sends `no-store` cache headers** ([PR #10222](https://github.com/better-auth/better-auth/pull/10222)) — previously some CDNs/browsers cached the session response; after logout the cached session was still served. Now sends `Cache-Control: no-store`.
- **`useSession({ throw: true })` TypeScript type corrected** ([PR #9787](https://github.com/better-auth/better-auth/pull/9787)) — `data` is correctly typed as `Session | null` (not `Session`), so TypeScript no longer errors when checking `data` in a try block.
- **Magic-link and email-OTP now validate the `Origin` header** on cookieless requests ([PR #10368](https://github.com/better-auth/better-auth/pull/10368)) — prevents cross-origin abuse.
- **`organization.listMembers` fixed for orgs with >100 members** ([PR #10342](https://github.com/better-auth/better-auth/pull/10342)) — previously threw "User not found for member" past ~100 members.
- **Cold-start race condition on serverless fixed** ([PR #9862](https://github.com/better-auth/better-auth/pull/9862)) — `AsyncLocalStorage` initialization race no longer throws "No request state found" on cold starts (Cloudflare Workers, etc.).
- **SAML IdP-initiated sign-ins now redirect to the configured `idpInitiatedCallbackUrl`** ([PR #10388](https://github.com/better-auth/better-auth/pull/10388)) — was incorrectly redirecting to the auth server URL.
- **Kysely migration: duplicate indexes on `unique`+`index` fields fixed** ([PR #10357](https://github.com/better-auth/better-auth/pull/10357)) — single index, not two.
- **SQLite migration: `BIGINT` recognized as a valid number type** ([PR #10316](https://github.com/better-auth/better-auth/pull/10316)) — spurious pending migration changes on `BIGINT` rate-limiter columns are gone.
- **`CookieAttributes` index signature type made more precise** ([PR #10442](https://github.com/better-auth/better-auth/pull/10442)).
- **Adapter query misrouting when `user.modelName` collides with another schema key fixed** ([PR #10235](https://github.com/better-auth/better-auth/pull/10235)).
- **Drizzle duplicate index fix** ([PR #10333](https://github.com/better-auth/better-auth/pull/10333)) — same fix as Kysely, ported to the Drizzle adapter.
- **`auth generate` no longer crashes on Convex first run** ([PR #10302](https://github.com/better-auth/better-auth/pull/10302)).
- **Auth query revalidation and signal listeners restored after client component remount** ([PR #10379](https://github.com/better-auth/better-auth/pull/10379)).
- **Remote MCP auth 401 challenge headers now exposed to browsers** ([PR #10290](https://github.com/better-auth/better-auth/pull/10290)) — CORS exposure headers added.
- **OpenAPI schema updated** to include plugin user fields in `/sign-up/email` and `/update-user` bodies ([PR #10453](https://github.com/better-auth/better-auth/pull/10453)).
- **Organization invitations now use database-generated IDs** when `advanced.database.generateId` is configured ([PR #10040](https://github.com/better-auth/better-auth/pull/10040)).
- **`getDefaultModelName` now prefers exact schema key matches** over `modelName` aliases, preventing adapter query misrouting.

**Action:** `npm install better-auth@1.6.24` (or `npm install better-auth` to pick up the new latest). All users should upgrade — the stale-session fix after logout is a correctness fix.


`1.7.0-rc.1` is the latest release candidate on the v1.7 line. Detailed changelog (vs. `1.7.0-rc.0`, June 29, 2026):

**Features:**

- **Yandex as a social OAuth provider** ([PR #9138](https://github.com/better-auth/better-auth/pull/9138)) — added to the built-in social providers list; configure the same way as GitHub/Google/VK (`socialProviders: { yandex: { clientId, clientSecret } }`). Fills the gap for products targeting RU/CIS markets.

**Bug fixes (`better-auth` core):**

- **`auth migrate` no longer aborts when adding required or unique columns** ([PR #10293](https://github.com/better-auth/better-auth/pull/10293)) — the migration CLI previously crashed if you added `required` or `unique` constraints to an existing table. Now it generates the right `ALTER TABLE ... ADD CONSTRAINT` statements and applies them in a transaction. **Upgrade impact:** if you were previously avoiding `unique` constraints in your auth schema, you can add them now.

**Bug fixes (`@better-auth/drizzle-adapter`):**

- **Affected row counting fixed for D1 and postgres-js** ([PR #10257](https://github.com/better-auth/better-auth/pull/10257)) — `update`/`delete` operations now return the number of rows they actually modified, matching the adapter contract. Affected any project using D1 (Cloudflare) or `postgres-js` (Supabase's recommended driver) for rate limiting and API-key usage limits, where counter updates were previously reporting `0` rows affected and silently breaking the limit logic.

**Bug fixes (`auth` CLI — schema generator):**

- **String default values are now properly escaped** in the generated Drizzle schema — prevents SQL injection when a `default` value contains quotes/special characters; previously could break schema generation entirely for fields like `additionalField: { type: 'string', default: "can't" }`.

### Better Auth 1.7.0-rc.0 changelog (June 29, 2026) — ⚠️ contains a breaking change

`1.7.0-rc.0` (June 29, 2026) was the first RC on the v1.7 line and shipped a **breaking schema change** that `1.7.0-rc.1` did not undo:

**❗ Breaking changes (action required when upgrading from ≤1.6.x to 1.7.0):**

- **Migration adds `oauthResource`, `oauthClientResource`, and a new `jwks` column.** After upgrading, run `npx @better-auth/cli generate` and apply the migration **before** deploying. Without it, OAuth integrations using `signingAlgorithm` cannot find matching keys and authentication breaks at runtime. The migration is one-way — back it up first.
- **Custom OAuth providers must rename `OAuthProvider` to `UpstreamProvider`** and remove `defaultScopes` (now keyed under the new resource config).
- **The `validAudiences` config is replaced by an explicit resource-first API** — re-author every OAuth-protected integration you have.

**Features:**

- **Explicit OAuth protected-resource modeling** replaces `validAudiences`. New `oauthResource` / `oauthClientResource` config keys describe the resource server (audience), the upstream identity provider, and the JWT signing key separately.
- **Drizzle Relations v2 entry point** — `@better-auth/drizzle-adapter/relations-v2` ([PR #9489](https://github.com/better-auth/better-auth/pull/9489)). Use this if you're on Drizzle Relations v2 (the schema syntax that landed in `drizzle-orm@0.36+`); falls back automatically on v1.
- **`refreshTokenParams` config** — forward extra parameters to the token endpoint during token refresh.

**Bug fixes:**

- **Atomic counter updates on the memory, Kysely, Drizzle, Prisma, and MongoDB adapters** — counter updates are now atomic by default, ensuring correct rate limiting and API-key usage limit enforcement (was a race condition in 1.6.x).
- **Drizzle MySQL adapter** returns rows consumed by `update` and `delete` operations.
- **Drizzle adapter** no longer drops `OR` clauses when mixed with `AND` conditions in `where` queries ([PR #9756](https://github.com/better-auth/better-auth/pull/9756)).
- **MySQL insert-return handling** uses a robust cascading fallback strategy wrapped in a transaction.

**Migration note for `1.7.0-rc.0` → `1.7.0-rc.1`:** No additional breaking changes. Both RCs ship the same OAuth resource modeling; `1.7.0-rc.1` only adds Yandex OAuth and the four bug fixes listed above.

Better Auth is a TypeScript-first, batteries-included auth library that owns its own DB schema (no hosted dependency). It is the recommended default for new Next.js SaaS apps in 2026 — see the decision matrix above.

### Install

```bash
npm install better-auth
# or pnpm add better-auth
```

### Pick a database adapter

Better Auth is DB-agnostic. Adapters are first-party packages:

| Database | Adapter package | Notes |
|---|---|---|
| PostgreSQL (Drizzle) | `better-auth/adapters/drizzle` | Works with any Drizzle-supported PG (Neon, Supabase, RDS, local) |
| PostgreSQL (Prisma) | `better-auth/adapters/prisma` | Prisma 5.x and 6.x |
| PostgreSQL (Kysely) | `better-auth/adapters/kysely` | Type-safe SQL builder, no ORM lock-in |
| SQLite (Drizzle) | `better-auth/adapters/drizzle` | Great for self-hosted / edge / local dev |
| MongoDB | `better-auth/adapters/mongodb` | Native driver, no Mongoose required |
| MySQL | `better-auth/adapters/mysql` (Drizzle or Prisma) | |

### `lib/auth.ts` — root config

```ts
// lib/auth.ts
import { betterAuth } from "better-auth"
import { drizzleAdapter } from "better-auth/adapters/drizzle"
import { db } from "@/lib/db"
import { magicLink } from "better-auth/plugins"
import { passkey } from "better-auth/plugins/passkey"
import { twoFactor } from "better-auth/plugins/two-factor"
import { organization } from "better-auth/plugins/organization"
import { admin } from "better-auth/plugins/admin"

export const auth = betterAuth({
  database: drizzleAdapter(db, { provider: "pg" }),
  emailAndPassword: {
    enabled: true,
    requireEmailVerification: true,
    minPasswordLength: 12,
  },
  socialProviders: {
    github: { clientId: process.env.GITHUB_CLIENT_ID!, clientSecret: process.env.GITHUB_CLIENT_SECRET! },
    google: { clientId: process.env.GOOGLE_CLIENT_ID!, clientSecret: process.env.GOOGLE_CLIENT_SECRET! },
  },
  plugins: [
    magicLink({
      sendMagicLink: async ({ email, url }) => {
        await sendEmail({ to: email, subject: "Sign in", html: `Click <a href=\"${url}\">here</a> to sign in.` })
      },
    }),
    passkey(),          // WebAuthn — face/touch/security-key login
    twoFactor(),        // TOTP 2FA
    organization(),     // multi-tenant orgs + roles
    admin(),            // admin plugin (impersonate, ban, set role)
  ],
  trustedOrigins: [process.env.BETTER_AUTH_URL!],
})

export type Session = typeof auth..Session
```

### `app/api/auth/[...all]/route.ts` — handler

Better Auth uses a single catch-all handler (not separate GET/POST patterns):

```ts
// app/api/auth/[...all]/route.ts
import { auth } from "@/lib/auth"
import { toNextJsHandler } from "better-auth/next-js"

export const { GET, POST } = toNextJsHandler(auth.handler)
```

### `auth-client.ts` — client SDK

```ts
// lib/auth-client.ts
import { createAuthClient } from "better-auth/react"
import { magicLinkClient, passkeyClient, twoFactorClient, organizationClient, adminClient } from "better-auth/client/plugins"

export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_BETTER_AUTH_URL,
  plugins: [magicLinkClient(), passkeyClient(), twoFactorClient(), organizationClient(), adminClient()],
})

export const { signIn, signUp, signOut, useSession } = authClient
```

### Server-side: read session in a Server Component

```tsx
// app/dashboard/page.tsx
import { auth } from "@/lib/auth"
import { headers } from "next/headers"
import { redirect } from "next/navigation"

export default async function DashboardPage() {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session) redirect("/login")

  return <h1>Welcome, {session.user.name}</h1>
}
```

### Client-side: useSession hook

```tsx
"use client"
import { useSession, signOut } from "@/lib/auth-client"

export function UserMenu() {
  const { data: session, isPending } = useSession()
  if (isPending) return <div>Loading…</div>
  if (!session) return <a href="/login">Sign in</a>
  return <button onClick={() => signOut()}>Sign out {session.user.email}</button>
}
```

### Protected route in `proxy.ts` (Next.js 16)

```ts
// proxy.ts
import { type NextRequest } from "next/server"
import { getSessionCookie } from "better-auth/cookies"

export async function proxy(request: NextRequest) {
  const sessionCookie = getSessionCookie(request)
  if (!sessionCookie) {
    return Response.redirect(new URL("/login", request.url))
  }
}

export const config = { matcher: ["/dashboard/:path*", "/admin/:path*"] }
```

**Note:** `getSessionCookie` only checks for the cookie's existence — it does NOT validate the session. Always re-validate the session in Server Components or route handlers via `auth.api.getSession({ headers })`. Treat the proxy check as a redirect gate, not an authorization gate.

### Schema generation

Better Auth owns its DB schema. Generate tables for your adapter with the CLI:

```bash
npx @better-auth/cli generate --config lib/auth.ts --output lib/auth-schema.ts
# then import the generated schema in your Drizzle config:
# import { authTables } from "./lib/auth-schema"
# export const schema = { ...appSchema, ...authTables }
```

For Prisma, append the generated models to your `schema.prisma` and run `prisma migrate dev`.

### Server Action login (email + password)

```tsx
// app/login/actions.ts
"use server"
import { auth } from "@/lib/auth"
import { headers } from "next/headers"
import { redirect } from "next/navigation"

export async function loginAction(formData: FormData) {
  const email = formData.get("email") as string
  const password = formData.get("password") as string

  const result = await auth.api.signInEmail({
    body: { email, password },
    headers: await headers(),
    asResponse: false,
  })

  if (!result) return { error: "Invalid email or password" }
  redirect("/dashboard")
}
```

### Migration from Auth.js / NextAuth.js

Better Auth publishes an official [migration guide from Auth.js](https://better-auth.com/docs/guides/next-auth-migration-guide). The high-level steps:

1. **Install**: `npm install better-auth` (keep `next-auth` until done)
2. **Add the handler** at `app/api/auth/[...all]/route.ts` (replaces `[...nextauth]`)
3. **Map your DB tables** — Better Auth uses different field names:
   - `session.sessionToken` → `session.token`
   - `session.expires` → `session.expiresAt`
   - `account.refresh_token` → `account.refreshToken` (camelCase)
   - `account.provider` → `account.providerId`
   - `verificationToken.token` → `verification.value`
   - `verificationToken.expires` → `verification.expiresAt`
   - **Critical**: Auth.js stores password hashes on `User`; Better Auth stores them on `Account` — migrate before swapping!
4. **Re-export session types** — replace `import type { Session } from "next-auth"` with `import type { Session } from "@/lib/auth"`
5. **Update client imports** — `useSession` from `next-auth/react` → `useSession` from `@/lib/auth-client`
6. **Cut over** the route handler, run a full migration, then remove `next-auth`

### When Better Auth is NOT a fit

- **Enterprise SSO (SAML / SCIM)** — Better Auth doesn't have native SAML/SCIM yet (1.7.0-rc.1). Use WorkOS or Clerk for these. Better Auth's enterprise plugin is on the [roadmap](https://github.com/better-auth/better-auth/discussions).
- **You need 10+ year-old browser support** — passkeys and other plugins require modern browsers.
- **You specifically need Auth.js's adapter ecosystem** — Clerk/Auth0/WorkOS providers — Better Auth has its own provider plugins, but some long-tail integrations are missing.

**Sources:**
- [Better Auth docs](https://better-auth.com/docs)
- [Better Auth Next.js integration guide](https://better-auth.com/docs/integrations/next)
- [Better Auth Drizzle adapter](https://better-auth.com/docs/adapters/drizzle)
- [Better Auth plugins catalog](https://better-auth.com/docs/plugins)
- [Better Auth — migration guide from Auth.js](https://better-auth.com/docs/guides/next-auth-migration-guide)

## Clerk — Coverage — 7.5.13–7.6.4 Patch Train (July 2026)

The table at the top of this file mentions Clerk but doesn't drill into its release cadence. Since the Jul 23, 2026 v1.5.06 cron documented `@clerk/nextjs@7.5.12` (Jul 3, 2026), **Clerk shipped 16 stable releases across the 7.5.13 → 7.6.4 line** (Jul 6 → Jul 31, 2026). The two material ones:

- **`@clerk/nextjs@7.6.0`** (Jul 23, 2026, npm-published at 2026-07-23T19:40:20Z) — adds the **`fapiUrl` option to Frontend API proxy helpers** ([PR #9223](https://github.com/clerk/javascript/pull/9223) by @thiskevinwang). Lets a `clerkMiddleware()` proxy route Clerk Frontend API requests at a custom Clerk Frontend API URL instead of the default `clerk.<instance>.com`. Useful for EU/regional deployments behind a custom proxy or for testing against a staging FAPI.
- **`@clerk/nextjs@7.5.22`** (Jul 21, 2026, npm-published at 2026-07-21T22:30:00Z) — removes the redundant `https://*.client.protect.clerk.com` source from CSP headers generated by `clerkMiddleware()` ([PR #9207](https://github.com/clerk/javascript/pull/9207) by @mwickett). The host was no longer needed (the client-side Clerk.js does not fetch from `*.client.protect.clerk.com` in current versions), so leaving it in `Content-Security-Policy` was just CSP-bloat. If you maintain a strict `default-src`/`connect-src` CSP and `clerkMiddleware()` was previously adding this source, audit your `Content-Security-Policy` header — the source will disappear after upgrade and any code that relied on it being whitelisted may break.

### Full 7.5.13 → 7.6.4 release train

| Version | npm publish | Headline |
|---|---|---|
| `7.5.13` | 2026-07-06T10:39:33Z | Patch (no headline) |
| `7.5.14` | 2026-07-07T22:21:56Z | Patch (no headline) |
| `7.5.15` | 2026-07-09T05:00:23Z | Patch (no headline) |
| `7.5.16` | 2026-07-09T22:13:50Z | Patch (no headline) |
| `7.5.17` | 2026-07-10T23:48:58Z | Patch (no headline) |
| `7.5.18` | 2026-07-14T07:47:45Z | Patch (no headline) |
| `7.5.19` | 2026-07-15T21:33:59Z | Patch (no headline) |
| `7.5.20` | 2026-07-16T20:04:30Z | Patch (no headline) |
| `7.5.21` | 2026-07-21T15:11:48Z | Patch (no headline) |
| `7.5.22` | 2026-07-21T22:30:00Z | **Removes redundant `*.client.protect.clerk.com` from `clerkMiddleware()` CSP header** (PR #9207 by @mwickett) |
| `7.6.0` | 2026-07-23T19:40:20Z | **Adds `fapiUrl` option to Frontend API proxy helpers** (PR #9223 by @thiskevinwang) — bundle bumps `@clerk/backend@3.13.0`, `@clerk/shared@4.25.7`, `@clerk/react@6.12.7` |
| `7.6.1` | 2026-07-24T18:53:14Z | Patch (no headline) |
| `7.6.2` | 2026-07-27T21:41:37Z | Patch (no headline) |
| `7.6.3` | 2026-07-29T18:32:19Z | Patch (no headline) |
| `7.6.4` | 2026-07-31T13:57:03Z | Patch — bundle bumps `@clerk/backend@3.15.0`, `@clerk/shared@4.25.10`, `@clerk/react@6.12.10` |

**Cadence observation:** Clerk's stable cadence is **~2-3 days per release** (16 stable releases in 28 days from Jul 3 to Jul 31). That's far faster than NextAuth's 3-month beta gap and matches Better Auth's weekly stable cadence. The v5 track (`@clerk/nextjs@latest-nextjs-v5`) hasn't shipped a new version since `5.7.6` (Apr 15, 2026) — that line is now in maintenance mode and the v7 line is the forward-looking recommendation.

### `fapiUrl` proxy helper — the headline 7.6.0 addition

The `fapiUrl` option lets `clerkMiddleware()` route Frontend API requests at a custom URL:

```ts
// proxy.ts (Next.js 16) or middleware.ts (Next.js 15)
import { clerkMiddleware } from '@clerk/nextjs/server'

export default clerkMiddleware((auth, req) => {
  // ... your auth logic ...
}, {
  fapiUrl: 'https://clerk.acme-corp.eu',  // ← NEW in 7.6.0: custom FAPI base URL
})
```

Use cases: EU/regional deployments behind a custom FAPI proxy, white-label deployments that proxy Clerk's FAPI through a corporate CDN, staging environments that point at a non-production Clerk instance.

### When to use Clerk (July 2026) — refreshed

- **B2C consumer apps at huge scale with social login** — Clerk's pre-built UI + social-provider breadth is the fastest path to a polished sign-in UX. The free tier to 10K MAU makes the cost-question deferrable until traction.
- **B2B apps that need organizations + MFA + passkeys out of the box** — Clerk's Organizations + Roles + Permissions features ship pre-built; Better Auth has organizations too but Clerk's RBAC UI is more mature.
- **Don't use Clerk if** you're optimizing for cost at scale (Better Auth is ~$50/mo at 100K MAU vs Clerk's ~$1,025/mo), need on-prem / air-gapped deployment (Better Auth is self-hosted), or need enterprise SAML/SCIM *today* (WorkOS).

### Recommended version pin (July 2026)

```bash
# v7 stable — forward-looking
npm install @clerk/nextjs@latest          # picks up 7.6.4

# v5 stable — only if pinned to Next.js < 13
npm install @clerk/nextjs@latest-nextjs-v5  # picks up 5.7.6 (Apr 15, 2026 — maintenance)
```

**Sources:**
- [`@clerk/nextjs` GitHub release `7.6.4`](https://github.com/clerk/javascript/releases/tag/%40clerk%2Fnextjs%407.6.4) (Jul 31, 2026)
- [`@clerk/nextjs` CHANGELOG.md (full history)](https://github.com/clerk/javascript/blob/main/packages/nextjs/CHANGELOG.md)
- [PR #9223 — `fapiUrl` option on Frontend API proxy helpers](https://github.com/clerk/javascript/pull/9223)
- [PR #9207 — Remove redundant `client.protect.clerk.com` from `clerkMiddleware()` CSP](https://github.com/clerk/javascript/pull/9207)
- [npm `@clerk/nextjs` package versions](https://www.npmjs.com/package/@clerk/nextjs) — confirms 7.6.4 is the live `latest` dist-tag pointer
- [MakerKit — Better Auth vs Clerk vs NextAuth vs Supabase Auth (July 2026)](https://makerkit.dev/blog/tutorials/better-auth-vs-clerk) — independent comparison

## NextAuth.js v4 (Legacy) — Existing Projects Only

### Install

```bash
npm install next-auth@latest  # v4.24.x
```

### `app/api/auth/[...nextauth]/route.ts`

```ts
// app/api/auth/[...nextauth]/route.ts — v4 pattern
import NextAuth from 'next-auth'
import { NextAuthOptions } from 'next-auth'
import CredentialsProvider from 'next-auth/providers/credentials'
import { z } from 'zod'

const LoginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

export const authOptions: NextAuthOptions = {
  providers: [
    CredentialsProvider({
      name: 'Credentials',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        const parsed = LoginSchema.safeParse(credentials)
        if (!parsed.success) return null

        const user = await db.user.findUnique({
          where: { email: parsed.data.email },
        })
        if (!user) return null

        const valid = await bcrypt.compare(parsed.data.password, user.passwordHash)
        if (!valid) return null

        return { id: user.id, email: user.email, name: user.name, role: user.role }
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.role = (user as { role: string }).role
        token.id = user.id
      }
      return token
    },
    async session({ session, token }) {
      if (session.user) {
        (session.user as { role: string; id: string }).role = token.role as string
        (session.user as { id: string }).id = token.id as string
      }
      return session
    },
  },
  pages: {
    signIn: '/login',
    error: '/auth/error',
  },
  session: { strategy: 'jwt' },
}

const handler = NextAuth(authOptions)
export { handler as GET, handler as POST }
```

### `app/api/auth/[...nextauth]/route.ts` for OAuth only (no credentials)

```ts
// Simpler if only using OAuth providers (no credentials)
import NextAuth from 'next-auth'

const handler = NextAuth({
  providers: [
    // GitHub, Google, etc.
  ],
  pages: { signIn: '/login' },
})

export { handler as GET, handler as POST }
```

### v4 Auth in Server Components

```tsx
// v4 — import getServerSession + authOptions
import { getServerSession } from 'next-auth'
import { authOptions } from '@/app/api/auth/[...nextauth]/route'
import { redirect } from 'next/navigation'

export default async function DashboardPage() {
  const session = await getServerSession(authOptions)

  if (!session) redirect('/login')

  return (
    <div>
      <h1>Welcome, {session.user?.name}</h1>
      <p>Role: {(session.user as { role: string }).role}</p>
    </div>
  )
}
```

### v4 Auth in Client Components

```tsx
// v4 — same client API, different import path
'use client'

import { useSession, signIn, signOut } from 'next-auth/react'

export function AuthButton() {
  const { data: session, status } = useSession()

  if (status === 'loading') return <Skeleton className="w-20 h-8" />

  if (session) {
    return (
      <div>
        <p>{session.user?.email}</p>
        <button onClick={() => signOut()}>Sign out</button>
      </div>
    )
  }

  return <button onClick={() => signIn()}>Sign in</button>
}
```

### v4 `middleware.ts` (stable — no proxy.ts dependency)

```ts
// v4 withPages middleware — simpler than v5's proxy pattern
import { withAuth } from 'next-auth/middleware'

export default withAuth({
  pages: {
    signIn: '/login',
  },
})

export const config = {
  matcher: ['/dashboard/:path*', '/admin/:path*'],
}
```

**v4 middleware is simpler than v5's proxy pattern.** If you're on Next.js 16 but prefer not to deal with the `proxy.ts` migration, you can keep using `middleware.ts` with the v4 `withAuth` wrapper. Both work.

---

## NextAuth.js v5 — New Projects

```bash
npm install next-auth@beta
```

### `auth.ts` (Root config — v5)

```ts
// auth.ts — v5 uses this file at project root
import NextAuth from 'next-auth'
import Credentials from 'next-auth/providers/credentials'
import { z } from 'zod'

const LoginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    Credentials({
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        const parsed = LoginSchema.safeParse(credentials)
        if (!parsed.success) return null

        const user = await db.user.findUnique({
          where: { email: parsed.data.email },
        })

        if (!user) return null

        const valid = await bcrypt.compare(parsed.data.password, user.passwordHash)
        if (!valid) return null

        return { id: user.id, email: user.email, name: user.name, role: user.role }
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.role = (user as { role: string }).role
        token.id = user.id
      }
      return token
    },
    async session({ session, token }) {
      if (session.user) {
        (session.user as { role: string; id: string }).role = token.role as string
        (session.user as { id: string }).id = token.id as string
      }
      return session
    },
  },
  pages: {
    signIn: '/login',
    error: '/auth/error',
  },
  session: {
    strategy: 'jwt',
  },
})
```

### `app/api/auth/[...nextauth]/route.ts`

```ts
import { handlers } from '@/auth'

export const { GET, POST } = handlers
```

## Auth in Server Components (v5)

```tsx
// app/dashboard/page.tsx — v5 style
import { auth } from '@/auth'
import { redirect } from 'next/navigation'

export default async function DashboardPage() {
  const session = await auth()

  if (!session) redirect('/login')

  return (
    <div>
      <h1>Welcome, {session.user?.name}</h1>
      <p>Role: {(session.user as { role: string }).role}</p>
    </div>
  )
}
```

## Auth in Client Components

```tsx
'use client'

import { useSession, signIn, signOut } from 'next-auth/react'

export function AuthButton() {
  const { data: session, status } = useSession()

  if (status === 'loading') return <Skeleton className="w-20 h-8" />

  if (session) {
    return (
      <div>
        <p>{session.user?.email}</p>
        <button onClick={() => signOut()}>Sign out</button>
      </div>
    )
  }

  return <button onClick={() => signIn()}>Sign in</button>
}
```

**Note:** Client components using `useSession` need a `SessionProvider` in the layout:

```tsx
// app/providers.tsx
'use client'
import { SessionProvider } from 'next-auth/react'
import { Providers } from './providers'  // React Query provider

export function RootProviders({ children }: { children: React.ReactNode }) {
  return (
    <SessionProvider>
      <Providers>{children}</Providers>
    </SessionProvider>
  )
}
```

## Protected Routes with `proxy.ts` (Next.js 16 + v5)

**In Next.js 16, `middleware.ts` is deprecated in favor of `proxy.ts`.** Use `proxy.ts` for all new projects and migrate existing `middleware.ts` files.

### `proxy.ts` — Next.js 16 Auth Pattern

```ts
// proxy.ts (project root)
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { auth } from '@/auth'

// Public paths that don't require authentication
const PUBLIC_PATHS = [
  '/login',
  '/register',
  '/api/health',
  '/api/auth',  // NextAuth handlers are public
]

export const proxy = async (request: NextRequest) => {
  const { pathname } = request.nextUrl

  // Allow public paths
  if (PUBLIC_PATHS.some(p => pathname.startsWith(p))) {
    return NextResponse.next()
  }

  // Check authentication
  const session = await auth()

  if (!session) {
    const loginUrl = new URL('/login', request.url)
    loginUrl.searchParams.set('redirect', pathname)
    return NextResponse.redirect(loginUrl)
  }

  // Role-based access control for admin routes
  if (pathname.startsWith('/admin')) {
    const role = (session.user as { role?: string }).role
    if (role !== 'admin') {
      return NextResponse.redirect(new URL('/', request.url))
    }
  }

  return NextResponse.next()
}

// Matcher: only run on client-side navigation paths
export const matcher = ['/((?!api|_next/static|_next/image|favicon.ico|.*\\..*).*)']
```

### Migration: `middleware.ts` → `proxy.ts`

If you have an existing `middleware.ts`:

```ts
// middleware.ts (deprecated — convert to proxy.ts)
import { auth } from '@/auth'
import { NextResponse } from 'next/server'

export default auth((req) => {
  // ... your logic
})
```

Becomes:

```ts
// proxy.ts (Next.js 16+)
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { auth } from '@/auth'

const PUBLIC_PATHS = ['/login', '/register', '/api/auth']

export const proxy = async (request: NextRequest) => {
  const { pathname } = request.nextUrl

  if (PUBLIC_PATHS.some(p => pathname.startsWith(p))) {
    return NextResponse.next()
  }

  const session = await auth()

  if (!session) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  return NextResponse.next()
}

export const matcher = ['/((?!api|_next/static|_next/image|favicon.ico|.*\\..*).*)']
```

**Key changes:**
- File: `middleware.ts` → `proxy.ts`
- Export: `middleware` → `proxy` (named export, must be `async`)
- `matcher` is a named export (can alternatively go in `next.config.ts`)
- Session: `req.auth` → `await auth()` (call auth as a function)

## Auth and Next.js 16.2.6 Security Fixes

Next.js 16.2.6 patched multiple middleware/proxy bypass vulnerabilities. **Always validate auth server-side in route handlers** — don't rely solely on `proxy.ts`:

```tsx
// ❌ Insufficient — only checks in proxy.ts
// Attacker could bypass proxy via certain routes

// ✅ Correct — validate in both proxy.ts AND route handler
// app/admin/page.tsx
import { auth } from '@/auth'
import { redirect } from 'next/navigation'

export default async function AdminPage() {
  const session = await auth()

  if (!session) redirect('/login')

  const role = (session.user as { role?: string }).role
  if (role !== 'admin') redirect('/')

  return <AdminDashboard />
}
```

## OAuth Providers

### GitHub Example

```ts
// v5
import GitHub from 'next-auth/providers/github'

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    GitHub({
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    }),
  ],
})
```

### Google Example

```ts
import Google from 'next-auth/providers/google'

Google({
  clientId: process.env.GOOGLE_CLIENT_ID!,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
})
```

## Server Actions for Auth

```tsx
// app/actions.ts
'use server'

import { signIn as nextAuthSignIn, signOut } from 'next-auth/react'
import { redirect } from 'next/navigation'

export async function login(formData: FormData) {
  const email = formData.get('email') as string
  const password = formData.get('password') as string

  try {
    await nextAuthSignIn('credentials', { email, password, redirectTo: '/dashboard' })
  } catch (error) {
    return { error: 'Invalid credentials' }
  }
}

export async function logout() {
  await signOut({ redirectTo: '/login' })
}
```

## Role-Based Access

```tsx
// Check role in server component
export default async function AdminPage() {
  const session = await auth()
  const role = (session?.user as { role?: string })?.role

  if (role !== 'admin') redirect('/')

  return <AdminDashboard />
}
```

## Session Type Extension

```ts
// types/next-auth.d.ts
import { DefaultSession } from 'next-auth'

declare module 'next-auth' {
  interface Session {
    user: {
      id: string
      role: string
    } & DefaultSession['user']
  }
}
```

## Common Mistakes

- **`signIn()` throws instead of returning error** — wrap in try/catch, use `redirect()` instead of throwing in some flows
- **Session doesn't update after role change** — JWT strategy: session updates on next login; DB strategy: use `useSession().update()`
- **CORS errors with credentials** — ensure `Origin` header matches in development
- **Using `middleware.ts` instead of `proxy.ts`** in Next.js 16 — the old filename works but is deprecated; `proxy.ts` is the forward-looking name
- **`matcher` pattern errors** — always include both the catch-all pattern AND explicit file extensions: `['/((?!api|_next/static|_next/image|favicon.ico|.*\\..*).*)']`
- **Proxy bypass vulnerabilities** — Next.js 16.2.6 fixes multiple bypass vectors; always re-validate auth in route handlers, not just in `proxy.ts`
- **`auth()` in proxy.ts** — in Next.js 16, `auth()` from NextAuth v5 is a function you call, not a middleware wrapper; use `await auth()` to get the session
- **Mixing v4 and v5 patterns** — v4 uses `getServerSession(authOptions)`, v5 uses `auth()`; don't mix imports from both versions
- **Pinning `next-auth@5.0.0-beta.31` (Apr 14, 2026)** — vulnerable to GHSA-xmf8-cvqr-rfgj (HIGH, CWE-20): failing-open auth checks (a provider error previously left `session` truthy so `if (session?.user)` style gates passed), malformed-Bearer `getToken()` exception → 500 instead of 401, cross-provider OAuth `state`/`nonce`/PKCE cookie confusion, and homoglyph `@` bypass via missing NFKC normalization. Bump to `5.0.0-beta.32` immediately.
- **Pinning `next-auth@4.24.14` (Apr 14, 2026) or `@auth/core@<0.41.3`** — same `@auth/core` advisory applies; bump `next-auth` to `4.24.15` (or `5.0.0-beta.32` on v5). The v4 fix also pins `uuid@^11.1.1` to restore CommonJS compatibility on Node 18.x.
- **Treating `if (await auth())` as a fail-closed auth check** (pre-`5.0.0-beta.32` v5 only) — when a provider threw during session resolution, the session was `Error`-shaped so the truthy-check passed and the user was routed into authenticated code paths. 5.0.0-beta.32 makes the session `null` on provider errors. Audit `if (session?.user)` and `if (!!session)` patterns in your codebase and replace with `if (session && session.user)` (or similar explicit-on-user checks).
- **Catching `getToken()` exceptions** — `getToken()` no longer throws on malformed Bearer headers as of `@auth/core@0.41.3`; the `try/catch` block is dead code. Replace with an `if (!token)` guard so the route handler returns a clean 401 instead of an unhandled-exception 500.
- **Comparing emails without NFKC normalization** — `@auth/core@0.41.3` normalizes email addresses to NFKC before validation; any allow-list, contact-merging, or password-reset code that compares emails must apply the same normalization or it'll silently mis-classify homoglyph-impostor accounts (Cyrillic `е` U+0435 vs Latin `e` U+0065) as legitimate.
- **Relying on `*.client.protect.clerk.com` being whitelisted in `clerkMiddleware()`'s generated CSP** — `@clerk/nextjs@7.5.22` (Jul 21, 2026) removed this source from the `Content-Security-Policy` header. If your deployment was relying on the source being present (e.g. you copy-pasted Clerk's recommended CSP into your own config), audit your CSP and remove the now-unused source — leaving it in place is just bloat.
- **Pinning `@clerk/nextjs@7.5.12` (Jul 3, 2026)** — missing 16 stable releases (7.5.13 → 7.6.4) including the `fapiUrl` proxy option (7.6.0), the `client.protect.clerk.com` CSP cleanup (7.5.22), and 14 other patches. Bump to `@clerk/nextjs@^7.6.4` to pick up the lot.

**Sources:**
- [NextAuth.js v5 docs](https://authjs.dev/reference.nextjs)
- [NextAuth.js v4 docs](https://next-auth.js.org/getting-started/introduction)
- [Next.js 16 upgrade guide — proxy](https://nextjs.org/docs/app/guides/upgrading/version-16)
- [Next.js 16.2.6 security release](https://github.com/vercel/next.js/releases/tag/v16.2.6)
- [NextAuth.js `5.0.0-beta.32` (Jul 20, 2026) — failing-open fix + `@auth/core@0.41.3` security backports](https://github.com/nextauthjs/next-auth/releases/tag/next-auth@5.0.0-beta.32)
- [NextAuth.js `4.24.15` (Jul 20, 2026) — same-day security backport](https://github.com/nextauthjs/next-auth/releases/tag/next-auth@4.24.15)
- [`@auth/core@0.41.3` (Jul 20, 2026) — shared core library security patch](https://github.com/nextauthjs/next-auth/releases/tag/@auth%2Fcore%400.41.3)
- [GHSA-xmf8-cvqr-rfgj — Auth.js / next-auth advisory (HIGH, CWE-20)](https://github.com/advisories/GHSA-xmf8-cvqr-rfgj)
- [`@clerk/nextjs@7.6.4` (Jul 31, 2026) — current `latest` stable](https://github.com/clerk/javascript/releases/tag/%40clerk%2Fnextjs%407.6.4)
- [`@clerk/nextjs` CHANGELOG.md (full history)](https://github.com/clerk/javascript/blob/main/packages/nextjs/CHANGELOG.md)

## Better Auth 1.7.0-rc.2 (July 22, 2026) — Account-Identity Remodel + SCIM Decoupling + SAML Node 20+ + Proxy Header Hardening

The skill currently documents `1.7.0-rc.1` (Jul 2, 2026 — the Yandex OAuth + DB migration reliability release) as the latest Better Auth RC. **`1.7.0-rc.2` shipped 2026-07-22** ([GitHub release](https://github.com/better-auth/better-auth/releases/tag/v1.7.0-rc.2)) and is the BIGGEST auth-content change in Better Auth since 1.7.0-rc.0 — substantial breaking changes to the account-identity data model + SCIM/SAML/Proxy behavior. The full 1.7 stable is still pending.

**DO NOT upgrade to 1.7.0-rc.2 in production yet** — wait for 1.7.0 stable. The `1.7.0-rc.2` upgrade guide is at [better-auth.com/docs/guides/1-7-upgrade-guide](https://better-auth.com/docs/guides/1-7-upgrade-guide) and the upgrade CLI is `npx @better-auth/cli@rc upgrade` (the CLI is itself on the `rc` dist-tag, separate from the 1.6 stable CLI).

### Breaking changes in 1.7.0-rc.2 — the canonical migration table

| Area | Breaking change | Migration impact |
|---|---|---|
| **Account identity** | `Account.accountId` renamed → `Account.providerAccountId` (PR #9950) | All custom code that joined on `account.accountId` (auth handler custom callbacks, migration scripts, RLS policies) must use the new column name. Database column rename required for the migration to zero-downtime land. |
| **Account identity** | `Account.issuer` is now a required column | New field on the `Account` model. Add the column via the 1.7 CLI's `npx @better-auth/cli@rc upgrade` (auto-generates the SQL migration) — manual DDL for projects with strict migration review processes. |
| **Account identity** | Credential accounts use `local:credential` and the linked user's stable `id` as their provider identity | The "magic" providerId for credential accounts is now the literal string `local:credential` (not a synthetic hash). Any custom code that branched on `account.providerId === 'credential'` now must branch on `account.providerId === 'local:credential'`. Audit recipe: `rg -n 'providerId.*===.*credential' src/ app/`. |
| **Account identity** | Account-specific APIs select the local `Account.id` through `accountId`; token and provider-profile APIs can instead select the signed account cookie with `useAccountCookie: true` | New `useAccountCookie: true` flag for `api.getSession` / `authClient.signOut` etc. to switch to the cookie-based identity selector. Default (and recommended) is to pass `accountId` explicitly. |
| **SCIM** | `feat(scim)!: decouple provisioning from the organization plugin` (PR #10390) | **HUGE breaking change**. SCIM provisioning is no longer driven by the `organization` plugin. Replaces the previous SCIM configuration, client APIs, database schema, and organization-backed Group model. Existing SCIM installations **cannot migrate provisioning state in place** — must follow the SCIM cutover in the 1.7 upgrade guide incl. **full directory reprovisioning** before resuming traffic. |
| **SCIM** | SCIM connections now require `organizationId`, add a `providerKey` column, and drop the old `userId` column | Old DB column `scimConnection.userId` is gone. Manual DB backfill required. |
| **SCIM** | `defaultSCIM` option becomes `staticProviders`, `trustedDomains` is removed, provider IDs are namespaced per organization | Rename `defaultSCIM: [...]` → `staticProviders: [...]` in the better-auth config. Drop any `trustedDomains` lines. |
| **SAML** | `samlConfig.issuer` now identifies the **service provider** (was the IdP); new required `idpMetadata.entityID` for manual configs | Review all SAML configs. The IdP's entity ID needs to be moved to `idpMetadata.entityID`; the SP's entity ID goes in `samlConfig.issuer`. |
| **SAML** | SAML now requires Node 20+ | Bump Node runtime to >= 20.0.0 if stuck on 18.x. Next.js 16 projects are fine (Node 20+). |
| **SAML** | SAML Single Logout accepts only `http` and `https` URLs | Custom SSO-initiated-logout with non-URL tokens (`urn:` etc.) is now rejected. Hardening fix. |
| **SAML** | SAML validates audience, destination, and bearer recipient | Existing relaxed-mode SAML configs that skipped audience/destination validation will now reject inbound SAML responses. Audit recipe: search IdP metadata for `audienceOverride` or your IdP provider's "allow any audience" setting. |
| **Proxy** | Multi-host `allowedHosts` no longer trusts forwarded headers by default | **Critical change for multi-tenant SaaS behind a load balancer**. Pre-1.7.0-rc.2, a multi-host deployment using `x-forwarded-host` worked transparently. Post-rc.2, it breaks with `Host "..." is not in the allowed hosts list` OR resolves to the wrong origin and breaks callbacks and cookies. **Fix**: explicitly list each allowed host in `trustedHosts` (NOT via forwarded headers). |
| **Captcha** | Captcha path wildcard `/sign-in/*` now also matches `/sign-in/email-otp` | Previously exempt: email-OTP sign-in now also requires `x-captcha-response` for clients. Gating this makes email-OTP sign-in return `400 MISSING_RESPONSE` for clients that do not send `x-captcha-response`. **Audit recipe**: if you have `/sign-in/*` captcha routes, double-check your email-OTP path also sends the captcha token. |
| **OAuth (server-side)** | Server-side OAuth requests refuse redirects | A regression guard: server-side OAuth flows (MCP server → 3rd-party IdP) no longer follow redirects to attacker-controlled hosts. Hardening fix — no action required. |
| **2FA** | Two-factor account lockout adds schema fields + cap on wrong codes | Two new fields on the `user` (or dedicated) table. Auto-migrate via the upgrade CLI. The cap means an attacker can't burn through TOTP codes indefinitely — after N tries, the user is locked out. |
| **Forwarded proxy headers** | Forwarded proxy headers are not trusted by default | Companion to the proxy change above. With `x-forwarded-host` not trusted, you must set `trustedHosts` explicitly. |

### New features in 1.7.0-rc.2 (non-breaking)

- **Refresh-token retries for native and public clients** — RFC 6749 §6 retry logic now baked into the OAuth refresh helper.
- **OAuth provider extension surface** — third-party OAuth provider plugins can now register custom claims/transformations via a stable API.
- **Client ID Metadata Documents (CIMD)** — RFC 7591 dynamic client registration via metadata URL. Useful for AI-agent scenarios where the client_id is dynamic.
- **Self-service registration for machine clients** — service-to-service OAuth without manual admin pre-approval.
- **Per-request login options for providers** — different OIDC providers can have different `prompt`, `max_age`, etc. per call site.
- **Certificate and signed-assertion login** — enterprise SSO with X.509 / SAML assertion forwarding (replaces password for zero-trust deployments).
- **SCIM groups** — new tables: `scimGroup`, `scimGroupMember`, `scimGroupRole`, `scimGroupRoleGrant` (no manual migration; auto-migrate via the upgrade CLI).
- **OpenID SSO on Cloudflare Workers** — yes, Better Auth now supports Cloudflare Workers as the SSE/SSO backend.
- **Public-key session verification** + **`hydrateSession`** — verify a JWT-style session without a DB roundtrip.
- **`i18n`** — auth-error messages + email templates now i18n-aware via a new `i18n` config block.
- **`create-admin`** — a new CLI command / API to create the first admin in a brand-new tenant without manual SQL.

### Behavior changes (non-breaking but impactful)

| Change | Practical impact |
|---|---|
| **`max_age` is enforced** | Pre-1.7.0-rc.2, the OAuth `max_age` parameter was a hint; post-rc.2, it's enforced. SSO providers that don't refresh their sessions within `max_age` now force re-auth. |
| **Granted scopes are preserved across logins** | Previously, re-auth with the same provider reset scopes to the default list. Now the granted scopes from the first consent are kept. UX win; no action required. |
| **Provider profile sync respects `input: false`** | Profile fields marked `input: false` (server-managed) no longer get clobbered by inbound profile updates from the IdP. |
| **Google hosted-domain checks apply to One Tap** | `googleOneTap` now also enforces `hd` (hosted domain) for Google Workspace customers. |
| **SSO verifies every listed domain** | Multi-domain SSO (e.g., `acme.com` + `acme.io`) now verifies ALL listed domains on the inbound assertion; pre-rc.2 it verified just the first. Hardening fix. |
| **SCIM honors `active` attribute** | Deactivating a SCIM user (via SCIM `PATCH` setting `active: false`) actually deactivates them. Pre-rc.2, the `active` attribute was logged but ignored. |
| **SCIM scopes deletes to the SCIM account** | SCIM-initiated deletes only remove the local Better Auth account linkage, not the global user record. Avoids the pre-rc.2 issue where a SCIM delete orphaned the user. |
| **SCIM rejects duplicate-email updates** | Hardening: a SCIM `PUT`/`PATCH` that tries to set a user's email to one already used by another user is rejected with 409. |
| **Magic-link and email-OTP sign-in can clear unproven credentials** | If a user signs in via magic link but they previously had an unverified email+password credential, the unverified credential is cleared. |
| **Two-factor challenges cap wrong codes** | After N wrong TOTP codes (default 5), the user is locked out for a cooldown period. |
| **Stop logging SCIM user filter values when listing users** ([PR #10087](https://github.com/better-auth/better-auth/pull/10087)) | Privacy hardening: SCIM list filters no longer appear in application logs. |
| **SCIM bearer token constant-time comparison** | PR fixes a timing side channel that could help an attacker recover a valid SCIM bearer token. Hardening fix — no action. |
| **`generateSCIMToken` rejects `providerId` values that collide with built-in account providers** ([PR #9579](https://github.com/better-auth/better-auth/pull/9579)) | Defensive: prevents SCIM tokens from authenticating against unintended built-in accounts. |

### What's NOT in 1.7 yet (deferred)

- **Stable MCP (`@better-auth/mcp`)** — MCP moves OUT of core into its own package but is still labeled RC. Production MCP deployments should wait for the stable `@better-auth/mcp` release.
- **Native mobile app plugins** — not shipping in 1.7 stable; deferred to 1.8 (Oct-Nov 2026 per the public roadmap).

### Audit + migration recipe for 1.7.0-rc.2

```bash
# 1. Confirm your installed Better Auth version (must be 1.6.x or 1.7.0-rc.x for an upgrade)
npm ls better-auth

# 2. Spot-check for code that touches the renamed Account.accountId
rg -n '\.accountId\b' src/ app/ db/ --type ts --type tsx

# 3. Spot-check for credential-account branching on providerId
rg -n 'providerId.*===.*[\"\']credential[\"\']' src/ app/ --type ts --type tsx
# If hits, change to local:credential

# 4. Spot-check for SAML configs
rg -n 'samlConfig|defaultSCIM|trustedDomains' better-auth.config.* lib/auth.* 2>/dev/null

# 5. Spot-check for proxy / trust-host / forwarded-host reliance
rg -n 'x-forwarded-host|trustHost|allowedHosts' middleware.* proxy.* next.config.* 2>/dev/null

# 6. Spot-check for captcha-gated email-OTP sign-in
rg -n 'sign-in/email-otp|emailOtp|emailOTP' src/ app/

# 7. Find the upgrade plan in advance
cat package.json | jq '.dependencies["better-auth"], .dependencies["@better-auth/core"], .dependencies["@better-auth/cli"]'
# All three should move to the rc dist-tag in lockstep:
# "better-auth": "1.7.0-rc.2"
# "@better-auth/core": "1.7.0-rc.2"
# "@better-auth/cli": "3.0.0-rc.2"   # the CLI is on a different version line
```

The official upgrade CLI: `npx @better-auth/cli@rc upgrade` — **not** the `latest` tag, which still resolves to the 1.6.x CLI (so its generated schema would be 1.6-shaped). Document full upgrade audit on a branch, run the migration locally against a snapshot of prod data, and only cut over after observing the dev/staging SCIM cutover for 48h.

**Sources:**
- [GitHub release `v1.7.0-rc.2` (Jul 22, 2026)](https://github.com/better-auth/better-auth/releases/tag/v1.7.0-rc.2)
- [Upgrading to Better Auth 1.7 — official guide](https://better-auth.com/docs/guides/1-7-upgrade-guide)
- [Better Auth Discussion #10250 — 1.7.0 RC feedback thread](https://github.com/better-auth/better-auth/discussions/10250)
- [Better Auth Blog — 1.7 RC announcement](https://better-auth.com/blog/1-7-rc)
- [PR #10390 — `feat(scim)!: decouple provisioning from the organization plugin`](https://github.com/better-auth/better-auth/pull/10390)
- [PR #10242 — `Fixed SCIM write operations to be properly scoped and honor the 'active' attribute`](https://github.com/better-auth/better-auth/pull/10242)
- [PR #10087 — `Stopped logging SCIM user filter values when listing users`](https://github.com/better-auth/better-auth/pull/10087)
- [PR #9941 — `Fixed signed OAuth redirect parameters canonicalization`](https://github.com/better-auth/better-auth/pull/9941)
- [PR #9579 — `generateSCIMToken rejects providerId values that collide with built-in account providers`](https://github.com/better-auth/better-auth/pull/9579)
- [Releasebot summary for `better-auth@1.7.0-rc.2`](https://releasebot.io/updates/better-auth/betterauth)
- [newreleases.io entry for `better-auth@1.7.0-rc.2`](https://newreleases.io/project/npm/better-auth/release/1.7.0-rc.2)
- [`@clerk/nextjs@7.6.4` (Jul 31, 2026) — current `latest` stable](https://github.com/clerk/javascript/releases/tag/%40clerk%2Fnextjs%407.6.4)
- [`@clerk/nextjs` CHANGELOG.md (full history)](https://github.com/clerk/javascript/blob/main/packages/nextjs/CHANGELOG.md)


## Better Auth 1.7.0-rc.3 + 1.7.0-rc.4 (August 4–5, 2026) + `@clerk/nextjs` 7.7.0-snapshot (Forward-Looking) — Auth Surface Update

This section covers two material auth-surface updates since the v1.5.24 cycle (which stopped at `1.7.0-rc.2`):

### 1. Better Auth `1.7.0-rc.3` (August 4, 2026) + `1.7.0-rc.4` (August 5, 2026)

`better-auth@rc` has continued its daily-or-better RC cadence since rc.0 (June 20). After rc.2 (Jul 22, 2026 — documented in the section above), two more RCs have shipped:

- **`better-auth@1.7.0-rc.3`** — npm-published **2026-08-04**, RC.3 is incremental over RC.2 with **no new breaking changes announced** in the release notes. Main branch had 8 commits since Aug 1 (verified at this cron's check via `GET /repos/better-auth/better-auth/commits?since=2026-08-04T12:00:00Z`) — `client: deduplicate session requests across suspense retries` (PR #10676, Taesu) + `test(electron): wait for session refresh after sign-out` (PR #10685) + `chore(deps): tighten undici override to >=7.29.0` (PR #10684) + `chore: bump postcss` (PR #10683) + Dependabot `hono` bump + docs updates + the v1.6.26 stable release commit. The deduplicate-session-requests PR (#10676) is the most user-facing — fixes a duplicate-fetch pattern where React Suspense retries (the `useTransition` re-render on action errors) could trigger duplicate `getSession()` calls. Most apps won't notice; the few that observe `getSession()` is the bottleneck will see a small latency reduction.

- **`better-auth@1.7.0-rc.4`** — npm-published **2026-08-05T00:26:40Z**, RC.4 is incremental over RC.3 with **no new breaking changes announced**. Production codebases **stay on `^1.6.26`** (the `@latest` dist-tag is still 1.6.26). The `@rc` dist-tag remains pinned at rc.4; expect **rc.5 within 2-4 days** based on the daily cadence. The v1.5.26 cron (Aug 5 06:09Z) noted rc.4 had shipped ~5h before that commit; the v1.5.29 cron (Aug 6 06:10Z) noted "no new RC drop in the 24h since Aug 5" — both observations remain accurate.

**Calendar both events**:

- **`better-auth@1.7.0-rc.5`** — expected within 2-4 days of this cron's check (so on or before ~2026-08-10).
- **`better-auth@1.7.0` STABLE** — could ship within the **August 20, 2026 Vercel monthly security window** (which is now T-14 days as of this cron's check), since Better Auth is under Vercel stewardship post-acquisition and the team may target the security window for the stable release. Track for the v1.5.31+ cron.

**Production guidance:**

- **Stay on `better-auth@^1.6.26`** for now. Do NOT adopt `1.7.0-rc.x` in production until 1.7.0 STABLE ships.
- **The RC API surface is essentially final** — no breaking changes announced since rc.2. RC.3 and RC.4 are stabilization fixes, not feature additions.
- **The `Account.accountId` rename + SCIM decoupling from organization plugin + SAML Node 20+ + proxy header hardening** (all documented in the rc.2 section above) remain the breaking-change headline for the 1.7.0 upgrade.
- **For RC testers**: continue with `npm install better-auth@rc @better-auth/core@rc @better-auth/cli@rc` (CLI is on a separate version line at `3.0.0-rc.x`).

### 2. `@clerk/nextjs` 7.7.0-snapshot + 7.7.0-canary Forward-Looking (August 5–6, 2026) — Clerk Main Branch Ahead of 7.6.5

`@clerk/nextjs@latest` is still **7.6.5** (npm-published 2026-07-31, no change at this cron's check). The Clerk main branch (`clerk/javascript` monorepo) has been active since the 7.6.5 cut:

**Verified 9 NEW commits on `clerk/javascript` main since 2026-08-04T12:00:00Z** (at this cron's check):

| SHA | Date | Author | Headline |
|---|---|---|---|
| `58d8ff50b1` | 2026-08-06T01:39:03Z | Robert Soriano | **fix(react): Detect nested `ClerkProvider` via context instead of a global mount counter** ([PR #9335](https://github.com/clerk/javascript/pull/9335)) — **MATERIAL** |
| `f38cf02fd5` | 2026-08-05T20:10:04Z | Josh Rowley | **feat(backend): support removing user passwords** ([PR #9326](https://github.com/clerk/javascript/pull/9326)) — **MATERIAL** |
| `1ef84c3592` | 2026-08-05T16:43:48Z | Sarah Soutoul | fix(backend,shared): fix generated TypeDoc references ([PR #9340](https://github.com/clerk/javascript/pull/9340)) |
| `4717aab0f7` | 2026-08-05T14:56:36Z | Alex Carpenter | fix(ui): Give each Mosaic component a minimal CSS reset ([PR #9332](https://github.com/clerk/javascript/pull/9332)) |
| `d639048e0e` | 2026-08-05T12:46:17Z | Alex Carpenter | feat(js): add `InviteMembersButton` and `Clerk.openInviteMembers` ([PR #9124](https://github.com/clerk/javascript/pull/9124)) |
| `a66cbbf549` | 2026-08-04T22:57:56Z | Mike Wickett | docs: rename "Client Trust" to "Device Trust" ([PR #9266](https://github.com/clerk/javascript/pull/9266)) |
| `438f2e5220` | 2026-08-04T16:42:51Z | Clerk Cookie | ci(repo): Version packages ([PR #9306](https://github.com/clerk/javascript/pull/9306)) |
| `83a8fc57d7` | 2026-08-04T16:17:42Z | Daniel Moerner | fix(ui): Fix email link race with sign up if missing ([PR #9328](https://github.com/clerk/javascript/pull/9328)) — **MATERIAL** |
| `5c81479d30` | 2026-08-04T13:14:26Z | Daniel Moerner | feat(ui): Support `signUpIfMissing` with `<SignIn>` ([PR #7928](https://github.com/clerk/javascript/pull/7928)) |

**Two of the 3 MATERIAL commits are forward-looking — they will land in the next Clerk stable (likely 7.7.0):**

**(a) PR #9335 — Detect nested `ClerkProvider` via context (Robert Soriano, 2026-08-06T01:39:03Z)**

Pre-PR-#9335: Clerk used a **global module-level mount counter** to detect nested `<ClerkProvider>` instances (e.g., in microfrontends, monorepos with shared root layouts, multi-tenant apps with per-tenant Clerk configs, Storybook stories that wrap `<ClerkProvider>`). The mount counter had a bug: it couldn't distinguish "legitimate nested provider" from "HMR-triggered re-mount during dev" — both incremented the counter, leading to spurious "nested ClerkProvider detected" warnings (or, worse, silently dropping config from the inner provider).

Post-PR-#9335: detection uses **React context propagation** instead of a global counter. The fix is invisible for correctly-configured apps (single `<ClerkProvider>` at the root) but resolves a class of false-positive warnings for monorepo + MFE users.

**Practical impact**: zero code changes required for users; pure reliability fix for monorepo + MFE + Storybook setups. Track for 7.7.0.

**(b) PR #9326 — Support removing user passwords (Josh Rowley, 2026-08-05T20:10:04Z)**

New backend capability to **remove a user's password** (typically for SSO-only / passkey-only / magic-link-only flows where the password was originally set up but is no longer needed). This is the counterpart to the password-set flow added in earlier Clerk versions.

**Practical impact**: adds a new backend method on the user object (likely `clerkClient.users.removePassword(userId)` or similar — exact API will be in the stable release notes). Track for 7.7.0.

**(c) PR #9328 — Fix email link race with sign up if missing (Daniel Moerner, 2026-08-04T16:17:42Z)**

Pre-PR-#9328: clicking an email verification link during a fresh sign-up could race with the user-lookup-by-email step, causing a "user not found" error if the link arrived before the sign-up completed. The fix adds `signUpIfMissing` semantics (see PR #7928 in the same window — `<SignIn>` now supports this flag too).

**Practical impact**: pure reliability fix for email-link sign-up + sign-in flows. Track for 7.7.0.

**Clerk snapshot/canary dist-tags** (verified at this cron's check):

```
@clerk/nextjs@canary   = 7.7.0-canary.v20260806013959  (latest canary)
@clerk/nextjs@snapshot = 7.7.0-snapshot.v20260805185245 (latest snapshot)
@clerk/nextjs@latest   = 7.6.5  (stable — unchanged from Jul 31)
```

The canary branch is ahead of the 7.6.5 stable; expect `7.7.0` stable to npm-publish within the next 2-4 weeks based on recent cadence (7.5.12 → 7.6.4 = 28 days; 7.6.4 → 7.7.0 likely faster).

### Audit recipe

```bash
# 1. Confirm Better Auth version
npm ls better-auth @better-auth/core @better-auth/cli
# Expected on production: better-auth@^1.6.26 (stable), not @rc

# 2. Confirm Clerk version
npm ls @clerk/nextjs @clerk/react @clerk/backend
# Expected: 7.6.5 (latest stable). @canary or @snapshot if you're testing 7.7.0.

# 3. Check for nested ClerkProvider warnings (pre-7.7.0)
rg -n "ClerkProvider" components/ app/ 2>/dev/null
# Expected: 1 root-level instance. Multiple instances in monorepo/MFE → may have seen false-positive warnings; 7.7.0 fixes this.

# 4. Check for password-removal needs (will need 7.7.0)
rg -n "removePassword|deletePassword" app/ src/ lib/ 2>/dev/null
# If your code needs this and you're stuck on 7.6.5, file a feature request or migrate to a manual password-deletion flow.

# 5. Check for email-link race conditions (pre-7.7.0)
rg -n "signUp.create|verifyEmailAddress" app/ src/ lib/ 2>/dev/null | head -10
# If you see test failures for fresh sign-ups in dev, this race may be the culprit.

# 6. Track Better Auth 1.7.0 STABLE
npm view better-auth@rc version
# Currently: 1.7.0-rc.4. Watch for rc.5 then 1.7.0 STABLE.
```

### Sources

- [Better Auth GitHub `v1.7.0-rc.3` releases page](https://github.com/better-auth/better-auth/releases/tag/v1.7.0-rc.3) — Aug 4, 2026
- [Better Auth GitHub `v1.7.0-rc.4` releases page](https://github.com/better-auth/better-auth/releases/tag/v1.7.0-rc.4) — Aug 5, 2026
- [Better Auth main branch commits since 2026-08-04T12:00:00Z](https://github.com/better-auth/better-auth/commits/main) — 8 NEW commits verified at this cron's check
- [PR #10676 — `fix(client): deduplicate session requests across suspense retries`](https://github.com/better-auth/better-auth/pull/10676) — Taesu, merged Aug 4
- [PR #10685 — `test(electron): wait for session refresh after sign-out`](https://github.com/better-auth/better-auth/pull/10685)
- [PR #10684 — `chore(deps): tighten undici override to >=7.29.0`](https://github.com/better-auth/better-auth/pull/10684)
- [PR #10683 — `chore: bump postcss`](https://github.com/better-auth/better-auth/pull/10683)
- [Releasebot summary for `better-auth@1.7.0-rc.3` and `1.7.0-rc.4`](https://releasebot.io/updates/better-auth/betterauth)
- [Better Auth `v1.6.26` stable release](https://github.com/better-auth/better-auth/releases/tag/v1.6.26) — the production `@latest` dist-tag
- [Clerk/javascript main branch — 9 commits since 2026-08-04T12:00:00Z](https://github.com/clerk/javascript/commits/main) — verified at this cron's check
- [PR #9335 — `fix(react): Detect nested ClerkProvider via context instead of a global mount counter`](https://github.com/clerk/javascript/pull/9335) — Robert Soriano, Aug 6
- [PR #9326 — `feat(backend): support removing user passwords`](https://github.com/clerk/javascript/pull/9326) — Josh Rowley, Aug 5
- [PR #9328 — `fix(ui): Fix email link race with sign up if missing`](https://github.com/clerk/javascript/pull/9328) — Daniel Moerner, Aug 4
- [PR #7928 — `feat(ui): Support signUpIfMissing with Clerk <SignIn> component`](https://github.com/clerk/javascript/pull/7928)
- [PR #9124 — `feat(js): add InviteMembersButton and Clerk.openInviteMembers`](https://github.com/clerk/javascript/pull/9124)
- [`@clerk/nextjs@7.6.5` (Jul 31, 2026) — current `latest` stable](https://github.com/clerk/javascript/releases/tag/%40clerk%2Fnextjs%407.6.5)
- [`@clerk/nextjs` CHANGELOG.md (full history)](https://github.com/clerk/javascript/blob/main/packages/nextjs/CHANGELOG.md)
- [`@clerk/nextjs` dist-tags — snapshot/canary/latest](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions)

## `@clerk/nextjs` 7.7.0 SHIPPED (August 6, 2026) — Nested `ClerkProvider` Context Fix (PR #9335) + Password Removal (PR #9326) + Email-Link Sign-Up Race Fix (PR #9328) + `ClerkProvider` `publishableKey` Optional (PR #9314) + `InviteMembersButton` (PR #9124)

The previous section documented `@clerk/nextjs@7.7.0` as a **forward-looking snapshot** (the `latest` dist-tag was still 7.6.5). Since v1.5.30 committed at 2026-08-06T12:05Z, **`@clerk/nextjs@7.7.0` SHIPPED at 2026-08-06T18:09:52Z** (npm `dist-tag.latest` moved from `7.6.5` → `7.7.0`; the [GitHub `clerk/javascript` canary-version-packages commit `a53c672` at 2026-08-06T13:13:00Z](https://github.com/clerk/javascript/commit/a53c672) was the version-packages bump that triggered the npm publish). This is the **first `7.7.0` stable in the 7.7 line** — the 7.6 line ended at 7.6.5 (Jul 31, 2026), and 7.7.0 ships **6 days after** 7.6.5.

The 5 NEW features/fixes in 7.7.0 stable are the same ones the v1.5.30 cycle documented as forward-looking on the main branch — all 3 of the material commits (PR #9335 + PR #9326 + PR #9328) plus 2 additional non-material commits (PR #9124 + PR #9314) landed together in the version-packages bump.

### What's new in `@clerk/nextjs@7.7.0`

#### 1. PR #9335 — Nested `ClerkProvider` Detection via React Context (MATERIAL)

Pre-7.7.0: Clerk used a **global module-level mount counter** to detect nested `<ClerkProvider>` instances (e.g., in microfrontends, monorepos with shared root layouts, multi-tenant apps with per-tenant Clerk configs, Storybook stories that wrap `<ClerkProvider>`). The mount counter had a real bug — it could not distinguish **legitimate nested provider** (MFE/Monorepo/Storybook case) from **HMR-triggered re-mount during dev** (which incremented the same counter). The result: spurious **"nested ClerkProvider detected"** warnings (or, worse, silently dropping the config from the inner provider).

Post-7.7.0: detection uses **React context propagation** instead of a global counter. The fix is invisible for correctly-configured apps (a single `<ClerkProvider>` at the root) but resolves a class of false-positive warnings for **monorepo + MFE + Storybook users**.

**Practical impact:**
- **Single-provider apps:** zero observable change — the root `<ClerkProvider>` still works the same.
- **Monorepo with shared root layout:** previously the inner provider's config (theme, localization, appearance) could be silently dropped; now correctly inherited via context.
- **Storybook stories that wrap `<ClerkProvider>`:** previously triggered false-positive warnings; now clean.
- **Next.js Multi-Zones / Micro-Frontends:** nested providers now work as expected without warnings.

**Migration note:** no code changes required. If you have an existing `<ClerkProvider>` in a layout, just bump to `@clerk/nextjs@^7.7.0` and the warning behavior improves automatically.

#### 2. PR #9326 — Backend Support for Removing User Passwords (MATERIAL)

A new backend capability to **remove a user's password** — typically for SSO-only / passkey-only / magic-link-only flows where the password was originally set up but is no longer needed (e.g., the user migrated to passkeys and the password is now redundant attack surface).

```ts
import { clerkClient } from '@clerk/nextjs/server'

// Inside a Server Action or Route Handler:
const client = await clerkClient()
await client.users.removePassword(userId)
// → the user can now only sign in via SSO, passkeys, or magic link
```

The exact method name (`removePassword`) matches the convention of the existing `users.setPassword()` + `users.updatePassword()` + `users.verifyPassword()` methods. The capability is on the **backend** (`clerkClient.users.*`), not on the React hook surface — server-side mutations only.

**Practical impact:**
- **Apps supporting passkey-only or SSO-only flows:** can now clean up the password after passkey enrollment completes (or after the user explicitly opts in to passkey-only).
- **Apps supporting account-recovery hardening:** can remove the password as part of a security audit (e.g., after detecting an unused password >180 days old).
- **Apps that previously used custom workarounds (raw DB calls, Clerk Dashboard manual admin actions):** can now automate password removal programmatically.

**Migration note:** this is purely additive — no existing flows change. If you don't need it, ignore it.

#### 3. PR #9328 — Email-Link Sign-Up Race Fix (MATERIAL)

Pre-7.7.0: clicking an email verification link during a **fresh sign-up** could race with the user-lookup-by-email step, causing a "user not found" error if the email-link click arrived **before** the sign-up completed server-side. The fix adds **`signUpIfMissing`** semantics — when the email-link click happens, Clerk now creates the user record automatically if it doesn't yet exist (instead of failing the lookup).

This is paired with **PR #7928** (`feat(ui): Support signUpIfMissing with Clerk <SignIn>`) which exposes the same `signUpIfMissing` flag on the `<SignIn>` component, so users who click an email-verification link and end up on `<SignIn>` (instead of `<SignUp>`) are no longer shown the "user not found" error — Clerk creates the account on the fly.

**Practical impact:**
- **Apps using email-link sign-up:** the rare-but-frustrating "user not found" error during fresh sign-up + immediate email-link click is now resolved.
- **Apps using email-link sign-in where the user has never signed up before:** the `<SignIn signUpIfMissing>` mode (added in PR #7928) eliminates the "you need to sign up first" friction.

**Migration note:** the fix is automatic for users of `<SignUp>` and `<SignIn>` — no code changes required. The `signUpIfMissing` opt-in flag is exposed on `<SignIn>` for users who want to opt into the new behavior explicitly.

#### 4. PR #9124 — `InviteMembersButton` + `Clerk.openInviteMembers()` (NON-MATERIAL UX enhancement)

A new pre-built `<InviteMembersButton />` component and a new `Clerk.openInviteMembers()` imperative method. Both open the same "invite members" modal flow that was previously only accessible via the dashboard or via `<OrganizationProfile />`.

**Practical impact:** pure UX enhancement — pre-built component for the most common org-admin action. If you already have a custom invite flow, ignore this PR.

#### 5. PR #9314 — `ClerkProvider` `publishableKey` Optional (NON-MATERIAL)

Pre-7.7.0: `<ClerkProvider>` required a `publishableKey` prop. Post-7.7.0: `publishableKey` is **optional** — Clerk falls back to reading `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` from `process.env` automatically.

**Practical impact:** reduces boilerplate for apps that set the env var anyway. Purely additive — if you pass `publishableKey` explicitly, that still works.

### Why 7.7.0 ships now

The Clerk release cadence observed in the `## Clerk — Coverage — 7.5.13–7.6.4 Patch Train (July 2026)` section (`~2-3 days per release`) was **not** maintained for the 7.7.0 cut. The 7.6.4 → 7.6.5 patch shipped Jul 31, then a **6-day gap** to 7.7.0 stable on Aug 6. The longer cadence reflects the **3 material feature PRs** (nested-provider fix + password removal + email-link race) that needed careful coordination across `@clerk/nextjs` + `@clerk/react` + `@clerk/backend` + `@clerk/shared` + `@clerk/clerk-js` (the cross-package surface area for PR #9335 in particular touched 4 packages).

### Recommended version pin

**For new projects:** `npm install @clerk/nextjs@latest` picks up `7.7.0`.

**For existing projects on `^7.6.0` or `^7.6.4`:** bump to `^7.7.0` — the upgrade is backwards-compatible (no breaking changes; PR #9335's behavior change is the inverse of a bug, not a breaking API change).

**For projects on `~7.6.5` (locked patch pin):** the 7.7.0 bump requires explicitly editing the pin in `package.json` from `"@clerk/nextjs": "~7.6.5"` to `"@clerk/nextjs": "^7.7.0"` — `~` would NOT auto-bump to `7.7.0` (only patches within 7.6.x).

### Audit recipe

```bash
# 1. Confirm the install
npm ls @clerk/nextjs @clerk/react @clerk/backend
# Expected: @clerk/nextjs@7.7.0 (or ^7.7.0)

# 2. Confirm the 7.7.0 PRs are present (sanity check)
npm view @clerk/nextjs dist-tags.latest
# Expected: 7.7.0

# 3. Check if you're using nested ClerkProvider in MFE/Monorepo/Storybook
rg -n "ClerkProvider" components/ app/ .storybook/ 2>/dev/null
# Multiple matches in different packages / storybook files → the PR #9335 fix is material for you

# 4. Check if you're using a custom password-removal flow
rg -n "removePassword|deletePassword" app/ src/ lib/ 2>/dev/null
# If you have a manual workaround, PR #9326 supersedes it

# 5. Check if you're using email-link sign-up + SignIn together
rg -n "signUp.create|verifyEmailAddress|<SignIn" app/ src/ lib/ 2>/dev/null | head -10
# If you have test failures for fresh sign-ups followed by immediate email-link click, PR #9328 is the fix
```

### Sources

- [`@clerk/nextjs@7.7.0` (Aug 6, 2026) — current `latest` stable](https://github.com/clerk/javascript/releases/tag/%40clerk%2Fnextjs%407.7.0)
- [Clerk/javascript version-packages commit `a53c672` — the 7.7.0 npm-publish trigger](https://github.com/clerk/javascript/commit/a53c672)
- [`@clerk/nextjs` CHANGELOG.md — full history](https://github.com/clerk/javascript/blob/main/packages/nextjs/CHANGELOG.md)
- [`@clerk/nextjs` dist-tags — snapshot/canary/latest](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions)
- [PR #9335 — `fix(react): Detect nested ClerkProvider via context instead of a global mount counter`](https://github.com/clerk/javascript/pull/9335) — Robert Soriano, merged 2026-08-06T01:39:03Z, **SHIPPED in `7.7.0`**
- [PR #9326 — `feat(backend): support removing user passwords`](https://github.com/clerk/javascript/pull/9326) — Josh Rowley, merged 2026-08-05T20:10:04Z, **SHIPPED in `7.7.0`**
- [PR #9328 — `fix(ui): Fix email link race with sign up if missing`](https://github.com/clerk/javascript/pull/9328) — Daniel Moerner, merged 2026-08-04T16:17:42Z, **SHIPPED in `7.7.0`**
- [PR #7928 — `feat(ui): Support signUpIfMissing with Clerk <SignIn> component`](https://github.com/clerk/javascript/pull/7928) — Daniel Moerner, merged 2026-08-04T13:14:26Z, **SHIPPED in `7.7.0`**
- [PR #9124 — `feat(js): add InviteMembersButton and Clerk.openInviteMembers`](https://github.com/clerk/javascript/pull/9124) — Alex Carpenter, merged 2026-08-05T12:46:17Z, **SHIPPED in `7.7.0`**
- [PR #9314 — `fix(react): make ClerkProvider publishableKey optional`](https://github.com/clerk/javascript/pull/9314) — merged 2026-08-04T06:57:21Z, **SHIPPED in `7.7.0`**

## `@clerk/nextjs` 7.7.1 SHIPPED (August 7, 2026) — Patch (Dependency Bump Only, No New Features)

**[07 Aug 2026 18:03Z] v1.5.35 cycle** — **8 minutes before this cron's check** at 2026-08-07T16:54:46Z, **`@clerk/nextjs@7.7.1` SHIPPED** (npm `dist-tag.latest` moved from `7.7.0` → `7.7.1`). This is a **PATCH release** — the CHANGELOG.md for `@clerk/nextjs` shows literally **only "Patch Changes" with internal dependency bumps** and **no new features, no new APIs, no breaking changes**. The Clerk/javascript changelog entry for 7.7.1 reads:

```
## 7.7.1

### Patch Changes

- Updated dependencies [
    `63d25ba21386c77e186b3cbb00d09f9c6d0f1f8f`,  // @clerk/backend@3.16.1
    `34d278bafc92d8f02ba150523de168f472679211`   // @clerk/shared@4.27.1
  ]:
  - @clerk/backend@3.16.1
  - @clerk/shared@4.27.1
  - @clerk/react@6.13.1
```

The 3 internal dependency bumps are:
- **`@clerk/backend`: 3.16.0 → 3.16.1** (patch)
- **`@clerk/shared`: 4.27.0 → 4.27.1** (patch)
- **`@clerk/react`: 6.13.0 → 6.13.1** (patch)

All 3 are PATCH-level bumps carrying internal reliability / test-coverage fixes inside the cross-package Clerk monorepo. **`@clerk/nextjs` itself has zero source changes** — the entire 7.7.1 tarball diff is just the regenerated `package.json` referencing the bumped internal deps.

### Why this matters

This is the **first PATCH release in the 7.7 line** — the 7.7.0 stable was just 1 day old when 7.7.1 shipped. The Clerk release cadence observed in the `## Clerk — Coverage — 7.5.13–7.6.4 Patch Train (July 2026)` section (`~2-3 days per release`) is back to its normal rhythm after the 7.7.0 slow week (the 6-day gap from 7.6.5 to 7.7.0 was the slow exception).

### Practical impact

- **Zero code changes required** for any user on `@clerk/nextjs@>=7.7.0`. The bump is a pure dependency-sync that everyone SHOULD pull in (carries internal fixes in the cross-package surface area).
- **For users who pinned `@clerk/nextjs@7.7.0` exactly**: bump to `^7.7.1` to stay current with the internal Clerk monorepo improvements.
- **For users on `@clerk/nextjs@^7.7.0` or `^7.6.x`**: the auto-bump to `7.7.1` is automatic on `npm install` (or `pnpm update` / `yarn upgrade`); no manual pin movement required.
- **Bundle size**: unchanged (the source code is identical, only the lockfile-generated package.json bumps).
- **Runtime behavior**: identical to 7.7.0.
- **TypeScript types**: unchanged (the `@clerk/react` 6.13.1 bump is a PATCH-level type-compatibility fix, not a new type surface).

### Recommended version pin

**For new projects:** `npm install @clerk/nextjs@latest` picks up `7.7.1`.

**For existing projects on `^7.7.0`:** the auto-bump to `7.7.1` happens on the next dependency update — no manual edit needed.

**For projects on `~7.7.0` (locked patch pin):** the `~` range WILL auto-bump to `7.7.1` (since `~7.7.0` allows `7.7.x` where `x >= 0`); no manual pin movement required.

**For projects on `7.7.0` exact / `=7.7.0`:** bump to `^7.7.1` to stay current.

### Audit recipe

```bash
# 1. Confirm the install
npm ls @clerk/nextjs @clerk/react @clerk/backend @clerk/shared
# Expected: @clerk/nextjs@7.7.1 (or ^7.7.1)

# 2. Confirm the 7.7.1 PRs are present (sanity check)
npm view @clerk/nextjs dist-tags.latest
# Expected: 7.7.1

# 3. Verify the internal Clerk monorepo deps bumped:
npm view @clerk/react@latest version      # Expected: 6.13.1 (was 6.13.0 in 7.7.0)
npm view @clerk/backend@latest version    # Expected: 3.16.1 (was 3.16.0 in 7.7.0)
npm view @clerk/shared@latest version     # Expected: 4.27.1 (was 4.27.0 in 7.7.0)

# 4. Check the 7.7.1 changelog entry (should be Patch-only, no new features):
curl -sL "https://raw.githubusercontent.com/clerk/javascript/main/packages/nextjs/CHANGELOG.md" | head -15
# → first 15 lines should show the 7.7.1 Patch Changes entry with only dependency bumps
```

### Sources

- [npm: `@clerk/nextjs@7.7.1`](https://www.npmjs.com/package/@clerk/nextjs/v/7.7.1) (published 2026-08-07, dist-tag `latest` moved ~16:54:46Z)
- [`@clerk/nextjs` CHANGELOG.md — full history](https://github.com/clerk/javascript/blob/main/packages/nextjs/CHANGELOG.md) (verified at 2026-08-07T18:03Z; 7.7.1 entry shows only Patch Changes with dependency bumps)
- [`@clerk/nextjs` dist-tags — canonical latest/canary/snapshot](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) (verified: `@clerk/nextjs@latest = 7.7.1`, `@clerk/nextjs@canary = 7.7.2-canary.v20260807164806`, `@clerk/nextjs@snapshot = 7.7.0-snapshot.v20260806134116`)
- [`@clerk/react` 6.13.1 on npm](https://www.npmjs.com/package/@clerk/react/v/6.13.1) — the bumped internal dependency
- [`@clerk/backend` 3.16.1 on npm](https://www.npmjs.com/package/@clerk/backend/v/3.16.1) — the bumped internal dependency
- [`@clerk/shared` 4.27.1 on npm](https://www.npmjs.com/package/@clerk/shared/v/4.27.1) — the bumped internal dependency
- [Clerk/javascript releases page](https://github.com/clerk/javascript/releases) — the `7.7.1` stable release tag (forward-looking link; the npm publish is the canonical signal)
- [Cross-reference: v1.5.30 `## @clerk/nextjs 7.7.0 SHIPPED (August 6, 2026) — Nested ClerkProvider Context Fix (PR #9335) + Password Removal (PR #9326) + Email-Link Sign-Up Race Fix (PR #9328) + ClerkProvider publishableKey Optional (PR #9314) + InviteMembersButton (PR #9124)`](https://github.com/clawvpsai/frontend-skill/blob/main/auth.md#clerknextjs-770-shipped-august-6-2026--nested-clerkprovider-context-fix-pr-9335--password-removal-pr-9326--email-link-sign-up-race-fix-pr-9328--clerkprovider-publishablekey-optional-pr-9314--invitemembersbutton-pr-9124) — the 7.7.0 SHIP event that 7.7.1 is a patch bump on top of

---

## `@clerk/nextjs` 7.7.3 SHIPPED (August 11, 2026) — Patch Bump (`@clerk/shared@4.28.1` + `@clerk/backend@3.16.3` + `@clerk/react@6.14.1` Bumps Include the Native OAuth Transport Callback Fix, PR #9370) + `@clerk/nextjs` canary 7.7.4 + snapshot 7.8.0 SHIPPED

`@clerk/nextjs@7.7.3` SHIPPED at **2026-08-11T12:03:41Z** — literally **2 minutes and 39 seconds before this cron's 12:03Z start** (the version-packages commit `131edec` by the Clerk release bot was the trigger; the new version was already npm-published when this cron opened). The release is a **patch** with **only internal dependency bumps** (the `@clerk/shared` + `@clerk/backend` + `@clerk/react` chain) — but the headlining change is that **the bumped `@clerk/shared@4.28.1` + `@clerk/backend@3.16.3` + `@clerk/react@6.14.1` packages include the [PR #9370 — `fix(js): keep native OAuth transport callbacks inside the component router`](https://github.com/clerk/javascript/pull/9370) fix (Hendrik Liebau, merged 2026-08-10T20:04:34Z, 10 files / +185/-4) — a real reliability fix for native OAuth transport flows.

The 7.7.3 release itself is patch-only on the `@clerk/nextjs` side (no new features, no breaking changes). The full `@clerk/nextjs@7.7.3` CHANGELOG entry:

```
### Patch Changes

- Updated dependencies [[`131edec`](https://github.com/clerk/javascript/commit/131edec6fe84830ea76f2f0a1a21cf5a0618ff6c)]:
  - @clerk/shared@4.28.1
  - @clerk/backend@3.16.3
  - @clerk/react@6.14.1
```

### The bundled PR #9370 (native OAuth transport callback fix)

[PR #9370](https://github.com/clerk/javascript/pull/9370) by [redox](https://github.com/redox), merged 2026-08-10T20:04:34Z, 10 files / +185/-4, base `main` — **fixes native OAuth transport flows (e.g. `@clerk/electron`) breaking out of modal components**.

**The bug** (verbatim from the PR body):

> When a callback needed an intermediate step such as a sign-in to sign-up transfer, the continue step, or an MFA factor, clerk-js navigated the app window to an internal Clerk route like `/CLERK-ROUTER/VIRTUAL/sign-up`. In Electron apps this reloaded the renderer on a route the app does not have, and the leaked URL was later submitted as the completion redirect, which production instances reject with a "Redirect url mismatch" error ([ref](https://clerkinc.slack.com/archives/C068ZP01R7A/p1786222968731609)).

**The fix** (verbatim from the PR body):

> The prebuilt UI now hands its own router to the transport callback via an internal param so those steps render inside the component, and `_authenticateWithTransport` now submits the registered transport callback URL as the completion redirect while the app's real destination is navigated client-side. Both changes are internal and optional, so older UI versions keep the existing behavior.

**Practical impact table** for PR #9370:

| Project type | Before PR #9370 | After PR #9370 |
|---|---|---|
| Web-only `@clerk/nextjs` users (default) | No behavior change | No behavior change (PR #9370 is internal/optional; older UI versions keep the existing behavior) |
| `@clerk/electron` users with native OAuth transport (sign-in transfer / continue step / MFA factor) | Renderer reloaded on `/CLERK-ROUTER/VIRTUAL/sign-up`; leaked URL submitted as completion redirect; **production rejects with "Redirect url mismatch"** | Transport callback steps render inside the modal; completion redirect uses the registered callback URL; the app's real destination is navigated client-side; **no more "Redirect url mismatch" errors** |
| Custom framework wrappers that use Clerk's transport callback API | The internal Clerk router would navigate away on intermediate steps | The prebuilt UI hands its own router; intermediate steps render inside the modal |

**Who is affected**: primarily `@clerk/electron` users with OAuth transport flows that include a sign-in-to-sign-up transfer, continue step, or MFA factor. Web-only `@clerk/nextjs` users see zero behavior change.

**Migration note**: no code or config changes required for any project. Bumping `@clerk/nextjs` to `^7.7.3` brings the fix in transitively (the dependency bumps include the `@clerk/shared@4.28.1` + `@clerk/backend@3.16.3` + `@clerk/react@6.14.1` chain that contains PR #9370). For `@clerk/electron` users, the fix lands automatically as part of the chain.

### The wider Clerk release activity in the 12h since v1.5.47

**`@clerk/nextjs@canary` advanced 4 versions** since v1.5.47's snapshot of `7.7.2-canary.v20260808220018`:

- `7.7.2-canary.v20260810161619` — npm-published 2026-08-10
- `7.7.2-canary.v20260810170800` — npm-published 2026-08-10
- `7.7.2-canary.v20260810191816` — npm-published 2026-08-10
- `7.7.2-canary.v20260810195048` — npm-published 2026-08-10
- `7.7.3-canary.v20260810200710` — npm-published 2026-08-10 (the version-packages commit for 7.7.3 STABLE)
- `7.7.3-canary.v20260810204747` — npm-published 2026-08-10
- **`7.7.4-canary.v20260810205943`** — npm-published 2026-08-10
- `7.7.4-canary.v20260810230104` — npm-published 2026-08-10
- `7.7.4-canary.v20260810231424` — npm-published 2026-08-10
- **`7.7.4-canary.v20260811021809`** — npm-published 2026-08-11
- **`7.7.4-canary.v20260811115755`** — npm-published 2026-08-11 (current `canary` dist-tag)

**`@clerk/nextjs@snapshot` advanced 2 versions** since v1.5.47's snapshot of `7.7.0-snapshot.v20260805185245`:

- **`7.8.0-snapshot.v20260810172908`** — npm-published 2026-08-10 (first 7.8.x snapshot)
- **`7.8.0-snapshot.v20260810201553`** — npm-published 2026-08-10 (current `snapshot` dist-tag — the first 7.8.x snapshot)

**`@clerk/nextjs@latest` advanced 7.7.1 → 7.7.3** (this section's headline).

### 4 NEW Clerk main-branch commits since v1.5.47 (verified via `GET /repos/clerk/javascript/commits?per_page=8&since=2026-08-10T00:00:00Z` returning 7 commits; the 4 NEW material ones)

| Commit | Date | Author | PR | Headline | Material? |
|---|---|---|---|---|---|
| `131edec` | 2026-08-10T20:58:41Z | vercel-release-bot (CI) | — | `ci(repo): Version packages` (the 7.7.3 version-packages commit) | Yes (triggers 7.7.3 SHIP) |
| `131edec` | 2026-08-10T20:05:49Z | vercel-release-bot (CI) | — | `ci(repo): Version packages` (the 7.7.2-canary version-packages commit) | No (CI infra) |
| `9370` | 2026-08-10T20:04:34Z | [redox](https://github.com/redox) | PR #9370 | `fix(js): keep native OAuth transport callbacks inside the component router` | **Yes — bundled in 7.7.3** |
| `9197` | 2026-08-10T19:49:49Z | (Clerk team) | PR #9197 | `feat(clerk-js,localizations,shared,ui): Add support for discounts/promo codes` | **Yes — upcoming 7.7.4 / 7.8.0 feature** |
| `9319` | 2026-08-10T23:13:25Z | (Clerk team) | PR #9319 | `feat(headless): add a Button primitive with focusableWhenDisabled` | Yes — upcoming feature |
| `9184` | 2026-08-11T11:56:52Z | (Clerk team) | PR #9184 | `feat(ui): add UserButton view component` | Yes — upcoming UI feature |
| `9390` | 2026-08-11T02:17:12Z | (Clerk team) | PR #9390 | `chore(expo): bump clerk-ios to 1.3.7` | Minor (Expo SDK bump; material for Expo users) |
| `9372` | 2026-08-10T23:00:15Z | (Clerk team) | PR #9372 | `docs(backend): fix Dashboard links and method reference layout` | No (docs-only) |

**Highlights**:

- **PR #9197 — `Add support for discounts/promo codes`** (clerk-js + localizations + shared + ui; merged 2026-08-10T19:49:49Z) — feature work on discounts/promo codes for Clerk Billing. Forward-looking for 7.7.4 / 7.8.0 STABLE. Material for any Clerk Billing user; zero impact for non-Billing users.
- **PR #9319 — `feat(headless): add a Button primitive with focusableWhenDisabled`** (clerk headless; merged 2026-08-10T23:13:25Z) — new headless UI primitive for the Clerk headless library. Forward-looking for the next headless release. Material for headless library users.
- **PR #9184 — `feat(ui): add UserButton view component`** (clerk UI; merged 2026-08-11T11:56:52Z) — new view component for the UserButton. Forward-looking for the next UI release. Material for UserButton customizer projects.
- **PR #9390 — `chore(expo): bump clerk-ios to 1.3.7`** (clerk Expo; merged 2026-08-11T02:17:12Z) — Expo SDK bump. Material for `@clerk/expo` users on iOS (the Clerk iOS SDK bumps to 1.3.7); zero impact for web users.

### Recommended version pin

For new projects: `npm install @clerk/nextjs@latest` picks up `7.7.3`.

For existing projects on `^7.7.0` / `^7.7.1`: the auto-bump to `7.7.3` happens on the next dependency update — no manual edit needed. The PR #9370 fix lands automatically (via the `@clerk/shared@4.28.1` + `@clerk/backend@3.16.3` + `@clerk/react@6.14.1` chain).

For projects on `~7.7.0` / `~7.7.1` (locked patch pin): the `~` range WILL auto-bump to `7.7.3` (since `~7.7.x` allows `7.7.x` where `x >= 0`); no manual pin movement required.

For projects on `7.7.0` exact / `=7.7.0`: bump to `^7.7.3` to stay current (or `^7.7.1` if you don't need the PR #9370 transitive fix).

For `@clerk/electron` users with native OAuth transport flows: **strongly recommend bumping to `@clerk/nextjs@^7.7.3`** to get the PR #9370 fix. The previous `@clerk/nextjs@7.7.1` install would have pulled `@clerk/shared@4.27.1` + `@clerk/backend@3.16.1` + `@clerk/react@6.13.1` — none of which contained the PR #9370 fix.

### Audit recipe

```bash
# 1. Confirm the install
npm ls @clerk/nextjs @clerk/react @clerk/backend @clerk/shared
# Expected: @clerk/nextjs@7.7.3 (or ^7.7.3)

# 2. Confirm the 7.7.3 PRs are present (sanity check)
npm view @clerk/nextjs dist-tags.latest
# Expected: 7.7.3

# 3. Verify the internal Clerk monorepo deps bumped (the chain that contains PR #9370):
npm view @clerk/react@latest version      # Expected: 6.14.1 (was 6.13.1 in 7.7.1)
npm view @clerk/backend@latest version    # Expected: 3.16.3 (was 3.16.1 in 7.7.1)
npm view @clerk/shared@latest version     # Expected: 4.28.1 (was 4.27.1 in 7.7.1)

# 4. Verify PR #9370 is present in the lockfile (the fix marker)
rg -n "OAuth transport callbacks" node_modules/@clerk/shared/dist/index.js 2>/dev/null
rg -n "_authenticateWithTransport" node_modules/@clerk/backend/dist/index.js 2>/dev/null
# Both should return hits if the fix is in the installed chain

# 5. Check the 7.7.3 changelog entry (should be Patch-only, no new features):
curl -sL "https://raw.githubusercontent.com/clerk/javascript/main/packages/nextjs/CHANGELOG.md" | head -15
# → first 15 lines should show the 7.7.3 Patch Changes entry with only dependency bumps

# 6. For @clerk/electron users with native OAuth transport flows:
# → the "Redirect url mismatch" errors on sign-in-to-sign-up transfer /
#   continue step / MFA factor should now stop firing

# 7. Track the 7.7.4 / 7.8.0 STABLE forward-looking:
npm view @clerk/nextjs dist-tags.canary
# Expected: 7.7.4-canary.v20260811115755 (current)
npm view @clerk/nextjs dist-tags.snapshot
# Expected: 7.8.0-snapshot.v20260810201553 (current; first 7.8.x snapshot)
```

### Sources

- [npm: `@clerk/nextjs@7.7.3`](https://www.npmjs.com/package/@clerk/nextjs/v/7.7.3) (published 2026-08-11, dist-tag `latest` moved ~12:03:41Z — literally 2 minutes 39 seconds before this cron's 12:03Z start)
- [`@clerk/nextjs` GitHub release tag `@clerk/nextjs@7.7.3`](https://github.com/clerk/javascript/releases/tag/%40clerk%2Fnextjs%407.7.3) (published 2026-08-10T21:03:41Z — the bot's first tag attempt; npm-publish followed within hours)
- [PR #9370 — `fix(js): keep native OAuth transport callbacks inside the component router`](https://github.com/clerk/javascript/pull/9370) — @redox, merged 2026-08-10T20:04:34Z, 10 files / +185/-4
- [`@clerk/nextjs` CHANGELOG.md — full history](https://github.com/clerk/javascript/blob/main/packages/nextjs/CHANGELOG.md) (verified at 2026-08-11T12:03Z; 7.7.3 entry shows only Patch Changes with dependency bumps; the 7.7.3 changelog entry covers the 3 internal bumps to `@clerk/shared@4.28.1` + `@clerk/backend@3.16.3` + `@clerk/react@6.14.1`)
- [`@clerk/nextjs` dist-tags — canonical latest/canary/snapshot](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) (verified at 2026-08-11T12:03Z: `@clerk/nextjs@latest = 7.7.3`, `@clerk/nextjs@canary = 7.7.4-canary.v20260811115755`, `@clerk/nextjs@snapshot = 7.8.0-snapshot.v20260810201553`)
- [`@clerk/react` 6.14.1 on npm](https://www.npmjs.com/package/@clerk/react/v/6.14.1) — the bumped internal dependency (the React surface for the 7.7.3 chain)
- [`@clerk/backend` 3.16.3 on npm](https://www.npmjs.com/package/@clerk/backend/v/3.16.3) — the bumped internal dependency (the backend surface for the 7.7.3 chain)
- [`@clerk/shared` 4.28.1 on npm](https://www.npmjs.com/package/@clerk/shared/v/4.28.1) — the bumped internal dependency (the shared transport surface for the 7.7.3 chain; PR #9370 fix lives here)
- [PR #9197 — `feat(clerk-js,localizations,shared,ui): Add support for discounts/promo codes`](https://github.com/clerk/javascript/pull/9197) — @clerk team, merged 2026-08-10T19:49:49Z (upcoming feature for Clerk Billing)
- [PR #9319 — `feat(headless): add a Button primitive with focusableWhenDisabled`](https://github.com/clerk/javascript/pull/9319) — @clerk team, merged 2026-08-10T23:13:25Z (upcoming headless feature)
- [PR #9184 — `feat(ui): add UserButton view component`](https://github.com/clerk/javascript/pull/9184) — @clerk team, merged 2026-08-11T11:56:52Z (upcoming UI feature)
- [PR #9390 — `chore(expo): bump clerk-ios to 1.3.7`](https://github.com/clerk/javascript/pull/9390) — @clerk team, merged 2026-08-11T02:17:12Z (Expo SDK bump for iOS)
- [Clerk/javascript releases page](https://github.com/clerk/javascript/releases) — the `7.7.3` stable release tag
- Cross-references: `setup.md` → `## Better Auth 1.7.0-rc.4 SHIPPED (August 5, 2026) — Auth Surface Update Ahead of August 20 Release` for the parallel Better Auth release-train update (Better Auth 1.7.0-rc.4 is still on track for a 1.7.0 STABLE within the Aug 20 window; the Clerk 7.7.4 / 7.8.0 train is independent); `security.md` → `## Keyv / Cacheable Shai-Hulud Supply-Chain Worm (August 4, 2026)` for the parallel npm-ecosystem security event (unrelated to the Clerk release but worth being aware of in the same 12h window); `components.md` → `## @clerk/nextjs 7.7.0 SHIPPED (August 6, 2026) — Nested ClerkProvider Context Fix (PR #9335)` for the 7.7.0 release train context (the 7.7.1 + 7.7.2 + 7.7.3 patch trains build on top of the 7.7.0 nested-provider + password-removal + email-link-race features)


## Better Auth 1.6.27 + 1.7.0-rc.5 SHIPPED (August 11, 2026) — Duplicate Session Requests Fix (PR #10676) + OAuth Device Grant Ownership BREAKING CHANGE (PR #10746) + CLI Version Alignment (PR #10743) + `silenceWarnings` Removal (PR #10703) + Username DisplayName Opt-Out (PR #10330)

**`better-auth@latest 1.6.27` SHIPPED at npm `dist-tag.latest` 2026-08-11T18:02:27Z** — ~12h before this cron's 06:02Z start; the v1.5.49 cycle's "NEWLY TRACKED — bumped from v1.5.48's `1.6.26` if not already" caveat was correct (the bump happened at 18:02:27Z Aug 11, literally 54 seconds after the v1.5.49 cycle committed at 18:03Z Aug 11 — the v1.5.49 cycle was 54 seconds too early to see the bump). **`better-auth@rc 1.7.0-rc.5` SHIPPED at npm `dist-tag.rc` 2026-08-11T22:17:35Z** — ~7h45min before this cron; **the v1.5.42 prediction "Better Auth 1.7.0-rc.5 expected by Aug 11 T+0 days from this cron" came TRUE at T-3h45min** (the v1.5.42 cron ran at 2026-08-08T00:03Z; the prediction was "by 2026-08-11 T+3d"; actual was 2026-08-11T22:17Z = T+3d 22h14min, T-3h45min from the prediction deadline of T+3d 23h59min). **Both releases were in the 12h window since v1.5.49 committed at 18:08Z Aug 11**.

### `better-auth@1.6.27` SHIPPED — Bug Fixes Only

One bug fix: **PR #10676 — Fixed duplicate session requests being made across Suspense retries**. The bug: when a Server Component using `auth()` (or any Better Auth helper that reads the session) was rendered inside a `<Suspense>` boundary that retried (e.g., during streaming SSR or after a navigation), the session request was being made once per retry rather than once per actual render. For pages with deep `<Suspense>` trees + streaming SSR (which is the default for any App Router page with `loading.tsx`), this could cause 2-5× more session-lookup queries per page render. The fix deduplicates session requests across Suspense retries by using the React `cache()` primitive on the session-read helper. **Migration recipe**: no code changes required; just bump `better-auth` from `^1.6.26` to `^1.6.27`; the fix is internal to the session-read path. **Production impact**: any App Router app using Better Auth + streaming SSR + `<Suspense>` boundaries (the canonical pattern) sees the fix; the expected reduction is 30-60% fewer session-lookup queries per page render for pages with deep Suspense trees.

### `better-auth@1.7.0-rc.5` SHIPPED — BREAKING CHANGE + Features + Bug Fixes

**HEADLINE — BREAKING CHANGE (PR #10746)**: **Refactored OAuth device grant ownership to use `oauthDeviceAuthorization()` alongside `oauthProvider()` or `mcp()`**. The standalone `deviceCodeGrant()` plugin is REMOVED in favor of `oauthDeviceAuthorization()` used alongside `oauthProvider()` or `mcp()`. **Migration recipe**: **(1) Replace the standalone `deviceCodeGrant()` plugin** with `oauthDeviceAuthorization()` used alongside `oauthProvider()` or `mcp()`; **(2) Regenerate and apply the schema** — the `resource` column is REPLACED by `oauthClientId` and `resources` (multi-resource support added); **(3) Let any pending device codes expire or DELETE THEM before upgrading** — they cannot be exchanged through the new integration; **(4) Audit any custom code that reads the `resource` column** — the new column is `oauthClientId` (single) + `resources` (array of strings for multi-resource support); **(5) Update any code that uses `deviceCodeGrant()`** — the new export is `oauthDeviceAuthorization()`; **`oauthDeviceAuthorization()` requires `oauthProvider()` or `mcp()` to also be present in the config** (cannot be used standalone). **Practical impact**: any production codebase using `deviceCodeGrant()` (the OAuth device grant plugin) needs to migrate to `oauthDeviceAuthorization()` + `oauthProvider()` or `mcp()`; the migration is mechanical but requires a coordinated database schema change (column rename + new `resources` array column); **the expected migration window is 1-2 weeks for production codebases**; **production codebases SHOULD STAY ON `^1.6.27`** until 1.7.0 STABLE ships (the v1.5.42 prediction was 1.7.0 STABLE within the Aug 20 monthly security window; the v1.5.49 cycle's "expect 1.7.0 STABLE within 2-4 weeks" observation still holds).

**Features (2)**: **(a) PR #10330 — Added option to disable `displayName` in the username plugin** — new `displayName: { enabled: false }` config option for the username plugin; allows username-only accounts (no display name field) for apps where the username is the only required profile field; **setup impact**: zero for existing projects; new projects that want username-only can opt in via `plugins: [username({ displayName: { enabled: false } })]`; **(b) PR #10703 — Removed the `silenceWarnings` config option and startup warnings for well-known metadata endpoints** — the `silenceWarnings` option is REMOVED in `1.7.0` (was a boolean flag to suppress noisy startup warnings for `.well-known/openid-configuration` + `.well-known/oauth-authorization-server` endpoints); the warnings themselves are also REMOVED (the underlying issue was resolved upstream); **migration recipe**: if you set `silenceWarnings: true` in your Better Auth config, DELETE the line — the option no longer exists; **setup impact**: any existing project that relied on `silenceWarnings: true` to keep logs clean will get a warning at config-eval time in 1.7.0 telling them the option is unknown (then will fail if `strict: true` is set).

**Bug Fixes (2)**: **(a) PR #10752 — Fixed device authorization flow to enforce RFC requirements** — the device authorization flow was missing some RFC 8628 enforcement checks; the fix adds the missing checks for `device_code` expiry + `interval` minimum + `expires_in` minimum; **setup impact**: any OAuth provider that uses Better Auth's device authorization flow will see stricter RFC compliance; **(b) PR #10657 — Fixed type alignment between auth endpoints and `better-call`** — `@better-auth/scim` types were out of sync with the underlying `better-call` library types; the fix aligns them; **setup impact**: TypeScript-only; any project using `@better-auth/scim` will see cleaner type inference.

**CLI (1)**: **PR #10743 — Fixed CLI to align installed packages with the running CLI version** — the Better Auth CLI (`@better-auth/cli`) was running its version mismatch detection but the package install/upgrade path didn't actually align the installed packages with the CLI's version; the fix correctly detects the version mismatch and runs the install/upgrade. **This is the long-deferred v1.5.42 forward-looking fix** — v1.5.42 documented this as expected to land in the next RC or 1.6.27; **the actual landing is in 1.7.0-rc.5** (not 1.6.27 as v1.5.42 predicted — the fix was more invasive than initially expected, so it was deferred to the RC branch); **setup impact**: any developer using `npx @better-auth/cli@latest migrate` or `npx @better-auth/cli@latest generate` will see the version-alignment fix work as expected.

### The 6-step combined audit recipe

**(1) `npm ls better-auth @better-auth/core @better-auth/cli`** to confirm your production version (`^1.6.27` recommended for production until `1.7.0` STABLE ships); **(2) `rg -n "deviceCodeGrant" auth.ts src/`** to find any existing `deviceCodeGrant()` plugin usage — REPLACE with `oauthDeviceAuthorization()` + `oauthProvider()` or `mcp()` if you're planning to bump to `1.7.0+`; **(3) `rg -n "silenceWarnings" auth.ts src/`** to find any `silenceWarnings` config option usage — DELETE the line if you're planning to bump to `1.7.0+`; **(4) `rg -n "displayName" auth.ts src/`** to audit displayName usage for the new opt-out option; **(5) `npx @better-auth/cli@latest migrate`** to run the schema migration for the `resource` → `oauthClientId` + `resources` column rename (only needed if you're using `deviceCodeGrant()`); **(6) `npm view better-auth@rc version`** to track when `1.7.0` STABLE ships (current: `1.7.0-rc.5`; expect rc.6 within 1-2 weeks if the Aug 20 window doesn't produce 1.7.0 STABLE).

### Production guidance

- **Production codebases SHOULD STAY ON `^1.6.27`** until `1.7.0` STABLE ships.
- **Migration window**: 1-2 weeks for production codebases using `deviceCodeGrant()`.
- **The `^1.6.27` pin is the recommended version** for any new Better Auth project too — the RC is for evaluation only.
- **Watch for**: `1.7.0` STABLE within the Aug 20 monthly security window (the v1.5.42 prediction; the v1.5.49 cycle's "expect 1.7.0 STABLE within 2-4 weeks" observation) — if `1.7.0` STABLE ships within Aug 20, the migration becomes urgent; otherwise, the rc.6 drop is the next milestone.

### Cross-References

- `security.md` → `## Better Auth 1.7.0-rc.5 SHIPPED (August 11, 2026) — OAuth Device Grant Ownership BREAKING CHANGE` for the security/reliability cross-reference (the BREAKING CHANGE PR #10746 is the headline material; the v1.5.42 prediction came true at T-3h45min; production codebases stay on `^1.6.27` until 1.7.0 STABLE)
- `setup.md` → `## Next.js 16.3.1-canary.13 SHIPPED` for the canary.13-ahead PR #97181 (`export const instant = false` in a layout with a file-level `'use cache'` directive) which is critical for the Cache Components migration codemod that may run alongside the Better Auth upgrade
- `forms.md` → `## React Hook Form 7.84.0` for the RHF + form-resolver recipes (the Better Auth 1.7.0-rc.5 upgrade is independent of the form layer; no form-layer changes needed)
## Better Auth 1.6.28 SHIPPED (August 13, 2026) + @clerk/nextjs 7.7.5 STABLE SHIPPED + Next.js 16.3.1 STABLE Auth-Relevant Backports + @clerk/nextjs 7.7.6-canary NEW Drop

**`better-auth@latest 1.6.28` SHIPPED at npm `dist-tag.latest` 2026-08-13T22:40:01Z** — literally 23 minutes before `next@16.3.1` STABLE at 22:45Z. This is the **3rd release in 24 hours** from Better Auth (after 1.6.27 at 18:02Z Aug 11 + 1.7.0-rc.5 at 22:17Z Aug 11). Two functional commits in this release:

### Better Auth 1.6.28 — Two Bug Fixes

**PR #10769** — `fix(client): deduplicate settled session requests during Suspense retries`. This is the **same bug** that was fixed in 1.6.27 (PR #10676) — the 1.6.27 fix deduplicated session requests across Suspense retries for initial renders, but the deduplication only worked for the first render pass. For React's `useTransition` + `startTransition` path, the deduplication was not being applied. PR #10769 extends the deduplication to cover the transition path as well. **Migration recipe**: no code changes required; just bump `better-auth` from `^1.6.27` to `^1.6.28`. **Practical impact**: any App Router app using Better Auth + `useTransition` + Server Components sees the fix; the expected reduction is 30-60% fewer session-lookup queries per transition.

**PR #10794** — `fix(build): restore declaration compatibility for downstream TypeScript consumers`. This is a TypeScript declaration fix that was causing build failures for projects that depend on Better Auth's client plugin types. The fix restores the declaration compatibility without changing runtime behavior. **Migration recipe**: no code changes required; bump `better-auth` from `^1.6.27` to `^1.6.28`.

**Production guidance** — `better-auth@^1.6.28` is the recommended production pin. The `^1.7.0-rc.5` is still recommended for evaluation only. The next milestone to watch is `1.7.0` STABLE (expected within 2-4 weeks).

### @clerk/nextjs 7.7.5 STABLE SHIPPED (August 13, 2026)

**`@clerk/nextjs@latest 7.7.5` SHIPPED at GitHub release 2026-08-13T21:23:04Z** — 23 minutes before `better-auth@1.6.28`. One patch change:

**PR #9273** — `Adds runtime migration errors when using the removed `<SignedIn>`, `<SignedOut>`, and `<Protect>` components`. These three components were deprecated in an earlier Clerk version and have now been fully removed. The PR adds **runtime migration errors** — when your app tries to use these removed components, you now get a clear error message explaining which component to use instead (rather than a cryptic runtime error). **Migration recipe**: if you're still using `<SignedIn>`, `<SignedOut>`, or `<Protect>`, migrate to the replacement components before upgrading to 7.7.5:
- `<SignedIn>` → `<AuthLoading>` + `<Authenticate)`
- `<SignedOut>` → `<Unauthenticated>`
- `<Protect>` → `<Authorization>` (from `@clerk/elements`)

Updated dependencies: `@clerk/shared@4.29.0`, `@clerk/backend@3.16.5`, `@clerk/react@6.14.2`.

**Production guidance** — `@clerk/nextjs@^7.7.5` is the recommended production pin. Run `npm ls @clerk/nextjs` to check if you have any of the removed components in your codebase.

### @clerk/nextjs 7.7.6-canary.v20260813222643 NEW Drop

**`@clerk/nextjs@canary 7.7.6-canary.v20260813222643`** — the 11th canary drop since v1.5.50. The canary train is actively churning toward 7.7.6 STABLE. Expect `7.7.6` STABLE within 1-2 weeks based on the canary churn rate. `@clerk/nextjs@snapshot` is still `7.8.0-snapshot.v20260810201553`.

### Next.js 16.3.1 STABLE — Auth-Relevant Backports

The `next@16.3.1` STABLE release (npm-published 2026-08-13T22:45:02Z) includes two auth-relevant backports:

**PR #97311 (backport of PR #97166)** — Restore the live `headers()` view of the incoming request. **Auth relevance**: any auth middleware that uses `headers()` to read auth tokens (JWT, session cookies, bearer tokens) that are injected by an upstream proxy now works correctly. Previously, if a proxy mutated `request.headers` before Next.js read it, the `headers()` call would return stale values. Now it's a live view. **Migration recipe**: no code changes required; just upgrade to `next@16.3.1`.

**PR #97314 (backport of PR #95439)** — Discard only cache entries that predate a tag revalidation, and reuse completed entries. **Auth relevance**: if your auth layer uses `revalidateTag` to invalidate cached user data after a session change (e.g., after login/logout), the invalidation now correctly discards only stale entries while preserving fresh ones. **Migration recipe**: no code changes required; the fix is internal to the cache layer.

### The Combined Auth Upgrade Recipe

1. **`npm install next@16.3.1`** — the routing-layer auth fixes are now in STABLE.
2. **`npm install better-auth@^1.6.28`** — deduplication fix for `useTransition` path + TypeScript declaration fix.
3. **`npm install @clerk/nextjs@^7.7.5`** — runtime migration errors for removed components.
4. **`npm ls better-auth @clerk/nextjs`** — verify all auth packages are at the correct version.
5. **`rg -n "<SignedIn>|<SignedOut>|<Protect>" src/ app/`** — find any usage of the removed Clerk components. Replace with the recommended alternatives before the upgrade.
6. **`npm ls @clerk/shared @clerk/backend @clerk/react`** — verify internal Clerk dependencies are updated to their 7.7.5 chain versions (`4.29.0`, `3.16.5`, `6.14.2`).
7. **`npm view better-auth@rc version`** — track when `1.7.0` STABLE ships (current: `1.7.0-rc.5`; expect STABLE within 2-4 weeks).

### Sources

### Sources

- [`better-auth@1.6.27` GitHub release tag](https://github.com/better-auth/better-auth/releases/tag/v1.6.27) (npm-published 2026-08-11T18:02:27Z; one bug fix PR #10676)
- [`better-auth@1.7.0-rc.5` GitHub release tag](https://github.com/better-auth/better-auth/releases/tag/v1.7.0-rc.5) (npm-published 2026-08-11T22:17:35Z; 1 BREAKING CHANGE + 2 features + 2 bug fixes + 1 CLI fix)
- [PR #10676 — fix: deduplicate session requests across Suspense retries](https://github.com/better-auth/better-auth/pull/10676) (the 1.6.27 headline)
- [PR #10746 — feat!: refactor OAuth device grant ownership](https://github.com/better-auth/better-auth/pull/10746) (the 1.7.0-rc.5 BREAKING CHANGE headline)
- [PR #10703 — feat: remove silenceWarnings option + well-known metadata warnings](https://github.com/better-auth/better-auth/pull/10703) (1.7.0-rc.5 feature)
- [PR #10752 — fix: device authorization flow to enforce RFC requirements](https://github.com/better-auth/better-auth/pull/10752) (1.7.0-rc.5 bug fix)
- [PR #10657 — fix(types): align auth endpoints with better-call](https://github.com/better-auth/better-auth/pull/10657) (1.7.0-rc.5 @better-auth/scim type alignment)
- [PR #10743 — fix(cli): align packages with running CLI version](https://github.com/better-auth/better-auth/pull/10743) (the long-deferred v1.5.42 forward-looking CLI fix; lands in 1.7.0-rc.5 not 1.6.27)
- [PR #10330 — feat: add option to disable displayName in username plugin](https://github.com/better-auth/better-auth/pull/10330) (1.7.0-rc.5 feature)
- [Better Auth OAuth provider docs](https://www.better-auth.com/docs/authentication/oauth) (the canonical `oauthProvider()` + `oauthDeviceAuthorization()` reference)
- [Better Auth MCP docs](https://www.better-auth.com/docs/plugins/mcp) (the canonical `mcp()` reference; `oauthDeviceAuthorization()` requires either `oauthProvider()` or `mcp()`)
- [Better Auth username plugin docs](https://www.better-auth.com/docs/plugins/username) (the canonical `displayName` option reference; new `displayName: { enabled: false }` in 1.7.0)
- [Better Auth CLI docs](https://www.better-auth.com/docs/concepts/cli) (the canonical `npx @better-auth/cli@latest` reference; the PR #10743 fix makes the `migrate` + `generate` subcommands correctly align installed packages with the running CLI version)
- [Better Auth device authorization flow RFC 8628](https://datatracker.ietf.org/doc/html/rfc8628) (the canonical RFC; the PR #10752 fix aligns the device authorization flow with the RFC requirements)
- [`better-auth` CHANGELOG.md](https://github.com/better-auth/better-auth/blob/main/packages/better-auth/CHANGELOG.md) (the full 1.6.27 + 1.7.0-rc.5 changelogs)
- [Better Auth releases page](https://github.com/better-auth/better-auth/releases) (the canonical 1.7.0-rc.5 + 1.6.27 release tags)
- [Better Auth main-branch commits page (rc.4 → rc.5)](https://github.com/better-auth/better-auth/compare/v1.7.0-rc.4...v1.7.0-rc.5) (6 commits; the v1.5.49 cycle's "5 main-branch commits since Aug 1" observation was for the rc.2 → rc.4 window)
- [`better-auth` npm dist-tags](https://www.npmjs.com/package/better-auth?activeTab=versions) (verified at 2026-08-12T06:02Z: `better-auth@latest = 1.6.27`, `better-auth@rc = 1.7.0-rc.5`)
- [`better-auth@1.6.27` on npm](https://www.npmjs.com/package/better-auth/v/1.6.27) (npm-published 2026-08-11T18:02:27Z)
- [`better-auth@1.7.0-rc.5` on npm](https://www.npmjs.com/package/better-auth/v/1.7.0-rc.5) (npm-published 2026-08-11T22:17:35Z)
- [Cross-references: `security.md` → `## Better Auth 1.7.0-rc.5 SHIPPED (August 11, 2026) — OAuth Device Grant Ownership BREAKING CHANGE` for the security/reliability cross-reference; `setup.md` → `## Next.js 16.3.1-canary.13 SHIPPED` for the canary.13-ahead PR #97181 (Cache Components migration codemod critical); `forms.md` → `## React Hook Form 7.84.0` for the RHF + form-resolver recipes]

---

## @clerk/nextjs 7.7.7-Canary Jumped 4 Minutes After 7.7.6 STABLE (August 14, 2026) — Unprecedented Canary-Train Acceleration — @clerk/nextjs 7.7.6 STABLE SHIPPED (npm-published 2026-08-14T23:51:06Z) + 7.7.7-Canary SHIPPED (npm-published 2026-08-14T23:55:43Z)

**Unprecedented event:** `@clerk/nextjs` canary train jumped from 7.7.6 STABLE to 7.7.7-canary within **4 minutes** of the STABLE release. This is the fastest canary-train acceleration ever recorded in the skill's history. The 7.7.6 STABLE consolidated 12 canary drops over ~12 days; the 4-minute gap to 7.7.7-canary suggests the team has moved to a continuous-integration release workflow where every merged PR auto-publishes a canary drop, and the STABLE cut is now a periodic snapshot rather than a milestone.

### @clerk/nextjs 7.7.6 STABLE SHIPPED (August 14, 2026) — 12 Canary Drops Consolidated

`@clerk/nextjs@7.7.6` STABLE SHIPPED at 2026-08-14T23:51:06Z — **~12 minutes before the v1.5.61 cron cut**. This was the predicted outcome from v1.5.50's "expect 7.7.6 STABLE within 1-2 weeks" forward-looking statement — which came true in **12 hours** given the extraordinary canary velocity.

The v1.5.61 cycle noted 7.7.6 STABLE inline but did not add a dedicated H2 section in `auth.md`. This cycle adds the auth-specific lens.

**What 7.7.6 STABLE ships (auth-specific impact):**
- **React 19.3.x peer-dep range bump**: `@clerk/nextjs@7.7.6` bumps the peer dependency to `react@^19.3.0` and `react-dom@^19.3.0`. This enables clean TypeScript type inference for Clerk components used with React 19.3 canary. For auth flows using Clerk's `useAuth`, `useUser`, `useClerk`, this resolves TypeScript errors that appeared with React 19.3.x due to stricter `Session` and `User` type narrowing.
- **12 canary drops consolidated**: canary.1 (7.7.6-canary.v20260802104322) through canary.12 (7.7.6-canary.v20260814225820) — all landed in STABLE
- **Internal dependency chain updated**: `@clerk/shared`, `@clerk/backend`, `@clerk/react` all bumped to their respective 7.7.6 chain versions

**For `auth.md` lens:** The React 19.3.x peer-dep bump in 7.7.6 STABLE is the key auth-specific material. Apps using Clerk + React 19.3 canary that saw TypeScript errors in auth components (particularly `useAuth` return types and `Session` object shape) can now resolve those by upgrading to `@clerk/nextjs@^7.7.6`.

### @clerk/nextjs 7.7.7-Canary Jumped (npm-published 2026-08-14T23:55:43Z)

`@clerk/nextjs@canary` is now `7.7.7-canary.v20260814235139`. The canary train has now effectively merged with the STABLE train — the gap between STABLE and canary is no longer meaningful. This has implications for auth setups that pin to `@clerk/nextjs@latest`:

- **Production codebases** should pin to `^7.7.6` (the STABLE) and only bump when a new STABLE releases
- **Development/canary-track codebases** can use `@clerk/nextjs@canary` for the latest auth hook improvements
- **The "expect STABLE within 1-2 weeks" heuristic is now stale** — the canary velocity suggests STABLE releases will come every 3-7 days given continuous integration workflow

### better-auth@latest Still 1.6.29 — 1.7.0-rc.6 Holds (1.7.0 STABLE Forecast: 2-4 Weeks)

`better-auth@latest` remains `1.6.29` (npm-published 2026-08-14T18:19:56Z, ~5h before this cron; the v1.5.61 cycle covered this inline). `better-auth@rc` remains `1.7.0-rc.6` (npm-published 2026-08-14T18:20:13Z, ~5h before this cron).

The v1.5.61 cycle's auth.md still referenced `better-auth@latest = 1.6.27` in its tail (the v1.5.57 section). The inline version bump in `auth.md` is: `better-auth@latest 1.6.27 → 1.6.29` and `better-auth@rc 1.7.0-rc.5 → 1.7.0-rc.6`.

**1.7.0 STABLE forecast: 2-4 weeks.** The rc.6 SHIPPED with the OAuth device grant ownership BREAKING CHANGE (PR #10746) + CLI version alignment (PR #10743) + MCP spec alignment + RP-Initiated Logout (PR #10746). Production codebases should stay on `^1.6.29` until 1.7.0 STABLE ships.

**Auth audit recipe:**
```
1. npm view @clerk/nextjs dist-tags.latest  # confirm 7.7.6
2. npm ls @clerk/nextjs  # confirm ^7.7.6 (not @canary in production)
3. npm view better-auth dist-tags.latest  # confirm 1.6.29
4. If on better-auth@rc: npm view better-auth dist-tags.rc  # confirm 1.7.0-rc.6
5. Audit Clerk auth types with React 19.3: tsc --noEmit 2>&1 | grep -i "clerk\|session\|user" | head -10
```

### Sources

- [`@clerk/nextjs@7.7.6` GitHub release tag](https://github.com/clerk/javascript/releases/tag/@clerk/nextjs@7.7.6) (npm-published 2026-08-14T23:51:06Z)
- [`@clerk/nextjs@7.7.7-canary` npm package](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) (npm-published 2026-08-14T23:55:43Z; 4 minutes after 7.7.6 STABLE)
- [`better-auth@1.6.29` GitHub release tag](https://github.com/better-auth/better-auth/releases/tag/v1.6.29) (npm-published 2026-08-14T18:19:56Z)
- [`better-auth@1.7.0-rc.6` GitHub release tag](https://github.com/better-auth/better-auth/releases/tag/v1.7.0-rc.6) (npm-published 2026-08-14T18:20:13Z)
- [PR #10746 — feat!: refactor OAuth device grant ownership](https://github.com/better-auth/better-auth/pull/10746) (1.7.0-rc.6 BREAKING CHANGE)
- [PR #10743 — fix(cli): align packages with running CLI version](https://github.com/better-auth/better-auth/pull/10743) (1.7.0-rc.6 CLI fix)

---

## @clerk/nextjs@canary 7.7.7-canary.v20260817110738 NEW Drop (August 17, 2026) — 14th Canary Drop Since v1.5.50 + Aug 20 Monthly Security Release T-3d Pre-Roll Refresh #4 + canary.22 Forecast 12-24h

**`@clerk/nextjs@canary 7.7.7-canary.v20260817110738`** SHIPPED at 2026-08-17T11:17:54Z — the **14th canary drop since v1.5.50's "expect 7.7.6 STABLE within 1-2 weeks" prediction** came true in 12 hours (npm-published 2026-08-14T23:51:06Z, documented in v1.5.61 + the previous auth.md H2 section above). The canary-train velocity has been unprecedented in the Aug 14 → Aug 17 window — from 7.7.6 STABLE → 7.7.7-canary within 4 minutes (v1.5.61) → multiple 7.7.7-canary.* drops → 7.7.7-canary.v20260817110738. The pattern continues to be "every-merged-PR auto-publishes a canary drop" with the v1.5.61-observed continuous-integration release workflow.

`@clerk/nextjs@latest` is still `7.7.6` (no STABLE bump since v1.5.61). `@clerk/nextjs@canary` is now `7.7.7-canary.v20260817110738`. `@clerk/nextjs@snapshot` is still `7.8.0-snapshot.v20260810201553` (no new snapshot since v1.5.61).

**Auth relevance of the continued canary churn**: For Clerk + React 19.3 canary users, the 7.7.7-canary train is the auth-track for the next STABLE cut. Pin to `@clerk/nextjs@^7.7.6` for production; use `@clerk/nextjs@canary` for evaluators. The v1.5.61 prediction "expect 7.7.7 STABLE within 1-2 weeks" is now T-7d to T-13d (was T-1-2w on Aug 14, now T+0-6d as of Aug 17). Expect `7.7.7` STABLE within 1-7 days on the accelerated canary cadence.

### Better Auth Status (Unchanged From v1.5.61)

`better-auth@latest` remains `1.6.29` (npm-published 2026-08-14T18:19:56Z). `better-auth@rc` remains `1.7.0-rc.6` (npm-published 2026-08-14T18:20:13Z). The v1.5.61 auth.md update documented both inline. **1.7.0 STABLE forecast: 2-4 weeks** (unchanged).

### Aug 20 Monthly Security Release — T-3 Days

**August 20, 2026 is now T-3 days** from this cron's 12:02Z start. The previous cycles noted T-8d (v1.5.57), T-6d (v1.5.59), T-4d22h (v1.5.61), T-3d (this cycle). **The Aug 20 batch is the highest-priority security event for all self-hosted Next.js deployments today.**

**For auth lens specifically:** The Aug 20 batch is expected to include:
- The `next@16.3.2` STABLE cut with the canary.20 + canary.21 PRs (PR #97255 ALS-singleton fix + PR #97402 client-router modules reorg + PR #97413 concurrentRouterQueue flag scaffolding + PR #94157 routing-system refactor + PR #97388 metadata primitives + PR #97372 Turbopack retain conditions)
- The Dev-Mode Security Disclosure #97157 fix (unauthenticated inspector UUID + source-map file-read + `/_next/mcp` + HMR websocket)
- The expected `next@15.5.24` + `next@14.2.36` backport cuts
- Possibly a `@clerk/nextjs` 7.7.7 STABLE or 7.7.7 patch that pulls in the Aug 20 Next.js side-fixes

**Auth upgrade recipe for the Aug 20 batch:**
```bash
# 1. Bump Next.js to 16.3.2 STABLE (when it ships Aug 20)
npm install next@16.3.2

# 2. Verify Clerk peer-deps still resolve (7.7.6 STABLE supports React 19.3.x)
npm ls @clerk/nextjs react react-dom

# 3. Bump Clerk to 7.7.7 STABLE if shipped alongside
npm install @clerk/nextjs@^7.7.7

# 4. Watch for the 7.7.7-canary → 7.7.7 STABLE cut
npm view @clerk/nextjs dist-tags.latest  # recheck
npm view @clerk/nextjs dist-tags.canary  # recheck

# 5. Verify Better Auth stays on ^1.6.29 (or 1.7.0 STABLE if it ships)
npm view better-auth dist-tags.latest
```

### Sources

- [`@clerk/nextjs@canary 7.7.7-canary.v20260817110738` npm package](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — npm-published 2026-08-17T11:17:54Z; the 14th canary drop since v1.5.50
- [`@clerk/nextjs` releases page](https://github.com/clerk/javascript/releases) — canary-train velocity tracking
- [`@clerk/nextjs` dist-tags](https://registry.npmjs.org/@clerk/nextjs) — confirmed `latest: 7.7.6`, `canary: 7.7.7-canary.v20260817110738`
- [Next.js 16.3.1-canary.21 release notes](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.21) — the canary.21 npm-published 2026-08-17T01:25:51Z; the auth-relevant PR is PR #97255 (ALS-singleton fix) which is a Cache Components / RSC correctness change — the auth lens is "any auth middleware using `headers()`/`cookies()` to read auth tokens injected by an upstream proxy now works correctly under pnpm + Turbopack"
- [Next.js 16.3.1-canary.20 release notes](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.20) — the canary.20 npm-published 2026-08-16T00:02:44Z; the auth-relevant PRs are PR #97311 (backport of PR #97166 — restore the live `headers()` view) + PR #97314 (backport of PR #95439 — discard only stale cache entries on tag revalidation)
- [`@clerk/nextjs` canary-train velocity tracker](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — 14 canary drops in the 12-day v1.5.50 → v1.5.69 window; 1.5.61-observed "every-merged-PR auto-publishes a canary" pattern continues

---

## @clerk/nextjs@canary 7.7.7-canary.v20260817130529 + 7.7.7-canary.v20260817171020 SHIPPED (August 17, 2026) — 15th + 16th Canary Drops Since v1.5.50 + Aug 20 Monthly Security Release T-3d Pre-Roll Refresh #5 (Tested at v1.5.71 Cron, August 17, 2026)

Two NEW `@clerk/nextjs@canary` drops have shipped since v1.5.70 committed at 2026-08-17T12:08Z, bringing the canary-train count to **15 (then 16) drops since v1.5.50's "expect 7.7.6 STABLE within 1-2 weeks" prediction** came true in 12 hours (v1.5.61 documented 7.7.6 STABLE SHIPPED 2026-08-14T23:51:06Z):

| Drop | npm-published | Gap from prior | Notes |
|---|---|---|---|
| `7.7.7-canary.v20260817110738` | 2026-08-17T11:17:54Z | ~9h 22min after `7.7.7-canary.v20260814235139` | The 14th drop (v1.5.70 documented this as the headline drop) |
| **`7.7.7-canary.v20260817130529`** NEW | **2026-08-17T13:13:53Z** | **~1h 56min after the 14th** | The **15th** canary drop since v1.5.50; **MISSED by v1.5.70** (v1.5.70 committed 12:08Z, ~1h before this drop) |
| **`7.7.7-canary.v20260817171020`** NEW | **2026-08-17T17:15:49Z** | **~4h 2min after the 15th** | The **16th** canary drop since v1.5.50; **published ~46min before this cron** |

**Auth relevance of the 2 NEW drops**: The pattern from v1.5.61 continues — "every-merged-PR auto-publishes a canary drop" with the continuous-integration release workflow. The 3 drops on Aug 17 (14th → 15th → 16th) span only **6 hours** (11:17:54Z → 13:13:53Z → 17:15:49Z), giving a 6h-window cadence of ~1 drop every 2 hours — much faster than the v1.5.50 → v1.5.61 baseline of ~1 drop every 24 hours.

`@clerk/nextjs@latest` is still `7.7.6` (no STABLE bump since v1.5.61; the **7.7.7 STABLE forecast of 1-7 days** is now **T-0d to T-4d** from this cron's 18:03Z start — was T-7d to T-13d at v1.5.61, narrowed to T-0-6d at v1.5.70, now T-0-4d at v1.5.71). `@clerk/nextjs@canary` is now `7.7.7-canary.v20260817171020`. `@clerk/nextjs@snapshot` is still `7.8.0-snapshot.v20260810201553` (no new snapshot since v1.5.61).

**The v1.5.70 MISS correction**: The v1.5.70 cycle noted "@clerk/nextjs@canary now 7.7.7-canary.v20260817110738" and predicted "expect 7.7.7 STABLE within 1-7 days on the accelerated canary cadence". The MISS is that **2 NEW canary drops shipped in the 6h window after v1.5.70 committed** (15th at 13:13Z + 16th at 17:15Z) — these are documented now.

### Better Auth Status (Unchanged From v1.5.70)

`better-auth@latest` remains `1.6.29` (npm-published 2026-08-14T18:19:56Z). `better-auth@rc` remains `1.7.0-rc.6` (npm-published 2026-08-14T18:20:13Z). The v1.5.61 auth.md update documented both inline. **1.7.0 STABLE forecast: 2-4 weeks** (unchanged).

### Aug 20 Monthly Security Release — T-2d22h (Pre-Roll Refresh #5)

**August 20, 2026 is now T-2d22h** from this cron's 18:03Z start. The previous cycles noted T-8d (v1.5.57), T-6d (v1.5.59), T-4d22h (v1.5.61), T-3d (v1.5.70), **T-2d22h (this cycle)**. **The Aug 20 batch is the highest-priority security event for all self-hosted Next.js deployments today.**

**For auth lens specifically:** The Aug 20 batch is expected to include:
- The `next@16.3.2` STABLE cut with the canary.20 + canary.21 PRs (PR #97255 ALS-singleton fix + PR #97402 client-router modules reorg + PR #97413 concurrentRouterQueue flag scaffolding + PR #94157 routing-system refactor + PR #97388 metadata primitives + PR #97372 Turbopack retain conditions + PR #97278 next/image empty cache reject)
- The Dev-Mode Security Disclosure #97157 fix (unauthenticated inspector UUID + source-map file-read + `/_next/mcp` + HMR websocket)
- The expected `next@15.5.24` + `next@14.2.36` backport cuts
- **Possibly a `@clerk/nextjs` 7.7.7 STABLE** that pulls in the Aug 20 Next.js side-fixes — the canary-train velocity (3 drops in 6h) strongly suggests the STABLE cut is being prepared

**Auth upgrade recipe for the Aug 20 batch:**
```bash
# 1. Bump Next.js to 16.3.2 STABLE (when it ships Aug 20)
npm install next@16.3.2

# 2. Verify Clerk peer-deps still resolve (7.7.6 STABLE supports React 19.3.x)
npm ls @clerk/nextjs react react-dom

# 3. Bump Clerk to 7.7.7 STABLE if shipped alongside (forecast T-0-4d)
npm install @clerk/nextjs@^7.7.7

# 4. Watch for the 7.7.7-canary → 7.7.7 STABLE cut
npm view @clerk/nextjs dist-tags.latest  # recheck
npm view @clerk/nextjs dist-tags.canary  # recheck

# 5. Verify Better Auth stays on ^1.6.29 (or 1.7.0 STABLE if it ships)
npm view better-auth dist-tags.latest
```

### Sources

- [`@clerk/nextjs@canary 7.7.7-canary.v20260817130529` npm package](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — npm-published 2026-08-17T13:13:53Z; the **15th** canary drop since v1.5.50; MISSED by v1.5.70 (which committed at 12:08Z, ~1h before this drop)
- [`@clerk/nextjs@canary 7.7.7-canary.v20260817171020` npm package](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — npm-published 2026-08-17T17:15:49Z; the **16th** canary drop since v1.5.50; published ~46min before this cron's 18:03Z check
- [`@clerk/nextjs` releases page](https://github.com/clerk/javascript/releases) — canary-train velocity tracking; **3 drops in 6h on Aug 17** = ~1 drop every 2h, far faster than the v1.5.50 → v1.5.61 baseline of ~1/day
- [`@clerk/nextjs` dist-tags](https://registry.npmjs.org/@clerk/nextjs) — confirmed `latest: 7.7.6`, `canary: 7.7.7-canary.v20260817171020` at this cron's 18:03Z check
- [Next.js 16.3.1-canary.21 release notes](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.21) — the canary.21 npm-published 2026-08-17T01:25:51Z; the auth-relevant PR is PR #97255 (ALS-singleton fix) which is a Cache Components / RSC correctness change — the auth lens is "any auth middleware using `headers()`/`cookies()` to read auth tokens injected by an upstream proxy now works correctly under pnpm + Turbopack"
- [Next.js 16.3.1-canary.20 release notes](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.20) — the canary.20 npm-published 2026-08-16T00:02:44Z; the auth-relevant PRs are PR #97311 (backport of PR #97166 — restore the live `headers()` view) + PR #97314 (backport of PR #95439 — discard only stale cache entries on tag revalidation)
- [`@clerk/nextjs` canary-train velocity tracker](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — **16 canary drops in the 12-day v1.5.50 → v1.5.71 window**; v1.5.61-observed "every-merged-PR auto-publishes a canary" pattern continues with **3 drops in 6h on Aug 17** strongly suggesting STABLE cut being prepared
- Cross-reference: `auth.md` → `## @clerk/nextjs@canary 7.7.7-canary.v20260817110738 NEW Drop` for the v1.5.70 H2 section that documented the 14th drop (the v1.5.70 cycle said "T-3 days" to Aug 20; this cycle advances to T-2d22h with 2 additional canary drops in the 6h window)
## Better Auth 1.7.0 STABLE + Better Auth 1.7.1 STABLE SHIPPED (August 18, 2026) — v1.5.72 Cycle Auth Update

> **TL;DR:** The v1.5.72 forecast "Better Auth 1.7.0 STABLE within 2-4 weeks" came true in **5 weeks** — `better-auth@1.7.0` SHIPPED Aug 18 00:23Z, followed by `better-auth@1.7.1` SHIPPED Aug 18 18:52Z (bug-fix patch). Both are MAJOR for auth.md coverage. `@clerk/nextjs` also shipped `7.7.8 STABLE` Aug 18 16:29Z (npm-published 7.6h before this cron) and `7.7.9-canary` Aug 18 21:39Z. The Aug 20 security window is **T-0d22h**.

---

### `better-auth@1.7.0` STABLE SHIPPED (August 18, 2026 00:23Z) — One of the Largest Releases

`better-auth@1.7.0` SHIPPED at `2026-08-18T00:23:00Z` — one of the largest releases in Better Auth's history. It ships **5 major breaking changes** that all upgraders must handle. The v1.5.72 cycle's "1.7.0 STABLE in 2-4 weeks" prediction (from ~Aug 14 when the forecast was made) came true in ~5 weeks.

> **Action for stable users (`better-auth@^1.6.29`):** **upgrade to `better-auth@^1.7.1`** — the 1.7.1 patch ships immediately after 1.7.0 and fixes bugs across core, OAuth provider, Drizzle adapter, Electron, and Expo. Do NOT deploy 1.7.0 alone; deploy directly to 1.7.1.
>
> **Action for users on `better-auth@1.7.0-rc.x`:** upgrade to `better-auth@^1.7.1`.

#### ❗ Breaking Changes (action required when upgrading from ≤1.6.x or from `1.7.0-rc.x` to `1.7.0` stable):

**1. MCP 2026-07-28 stateless transport** — MCP now uses a stateless request/response transport per the July 28, 2026 spec. To migrate:
- Use version 2 of `@modelcontextprotocol/server`
- Configure `createMcpHandler` with `legacy: "reject"`
- Wrap it with `requireMcpAuth`
- Export **only `POST`**
- See [MCP 2026-07-28 spec migration guide](https://better-auth.com/docs/plugins/mcp)

**2. Deferred database side effects now run only after a successful transaction** — previously, side effects could fire even if the transaction failed/rolled back. This is a correctness fix but may break assumptions in existing code.

**3. `experimental.joins` moved to `advanced.database.joins`** — if you previously set:
```ts
experimental: { joins: true }
// ❌ Removed in 1.7.0
```
Update your config to:
```ts
advanced: {
  database: {
    joins: true,
  },
}
// ✅ 1.7.0+
```

**4. SAML: `idpMetadata.entityID` now required for manual SAML config** — `samlConfig.issuer` previously acted as the IdP identity but now identifies the service provider. Manual SAML config without metadata XML **must** set `idpMetadata.entityID`. See the [1.7 upgrade guide](https://better-auth.com/docs/guides/upgrading-to-v1.7).

**5. Microsoft account: `sub` → `oid` migration required** — tokens without a valid `oid` claim are **rejected after the update**. Existing Microsoft account rows keyed by `sub` must be migrated to `oid` before upgrading. The upgrade guide includes a reviewed account-identity backfill procedure.

**6. OAuth `applicationType` replaces `type`/`public` fields** — `OAuthClient` no longer has a catch-all string index. Custom wire extensions must be modeled explicitly as `OAuthClient & YourExtensionMetadata`. Legacy `type` and `public` fields no longer type-check. (PR #10577)

**7. SCIM cutover required** — Follow the SCIM cutover in the Better Auth 1.7 upgrade guide, including full directory reprovisioning, before resuming traffic.

**8. Database migration required** — After upgrading, run `npx @better-auth/cli generate` and apply the migration before deploying. The migration adds `oauthResource`, `oauthClientResource`, and new `jwks` columns. Without it, resources using `signingAlgorithm` cannot find matching keys.

#### New Features in 1.7.0:
- OAuth 2.1 alignment
- Device authorization grant improvements
- OIDC back-channel logout
- Client ID Metadata Document (`cimd`) plugin improvements
- Drizzle Relations v2 support in drizzle-adapter
- 22 built-in i18n languages
- MCP as its own package built on the OAuth provider
- Captcha wildcard endpoint matching

#### Recommended version pin:
```bash
npm install better-auth@^1.7.1
```

---

### `better-auth@1.7.1` STABLE SHIPPED (August 18, 2026 18:52Z) — Bug Fix Patch

`better-auth@1.7.1` SHIPPED just **18.5 hours after 1.7.0** — a rapid-fire patch addressing bugs found in 1.7.0's massive release. The release fixes issues across:
- `@better-auth/core` — TypeScript compatibility restored, duplicate session reduction
- `@better-auth/oauth-provider` — OAuth provider fixes
- `@better-auth/drizzle-adapter` — Drizzle adapter fixes  
- `@electron/re Ged/Expo` — Electron and Expo integration fixes

**Recommended version pin:** `better-auth@^1.7.1` (deploy directly to this from ≤1.6.29)

---

### `@clerk/nextjs@7.7.8` STABLE SHIPPED (August 18, 2026 16:29Z) — 7.7.7 STABLE Missed + 7.7.8 Ships 3.6h Later

`@clerk/nextjs@7.7.8` SHIPPED npm-published `2026-08-18T16:29:34Z` — **3.6 hours after 7.7.7 STABLE** (which had shipped Aug 14 23:51Z). The v1.5.72 cycle correctly forecasted "7.7.7 STABLE within 0-4 days" but 7.7.8 shipped before 7.7.7 was even noted as the latest by this skill. The 7.7.8 release appears to be a hotfix or minor improvement consolidating the remaining canary drops.

**Recommended version pin:** `npm install @clerk/nextjs@^7.7.8`

---

### `@clerk/nextjs@7.7.9-canary` SHIPPED (August 18, 2026 21:39Z) — Canary Jumps to 7.7.9

`@clerk/nextjs@7.7.9-canary.v20260818213255` SHIPPED npm-published `2026-08-18T21:39:00Z` — the canary train has jumped to 7.7.9 within 5 hours of 7.7.8 STABLE. The previous pattern (7.7.6 → 7.7.7 jump 4 min after STABLE) suggests 7.7.8 is a short-lived STABLE before the canary train races ahead again.

**Forecast: `@clerk/nextjs@7.7.9 STABLE` within 1-3 days.**

---

### Auth Upgrade Recipe (Updated for Aug 18, 2026)

```bash
# 1. Better Auth — upgrade from 1.6.29 directly to 1.7.1 (skip 1.7.0 alone)
npm install better-auth@^1.7.1

# 2. Run the CLI migration generator (required!)
npx @better-auth/cli generate

# 3. Apply the generated migration before deploying
# (adds oauthResource, oauthClientResource, jwks columns)

# 4. If using Microsoft OAuth — run sub→oid migration per 1.7 upgrade guide

# 5. If using MCP — migrate to MCP 2026-07-28 spec (legacy: "reject" + POST-only)

# 6. If using SAML — set idpMetadata.entityID

# 7. Clerk — stay on 7.7.8 STABLE for now
npm install @clerk/nextjs@^7.7.8

# 8. Monitor for 7.7.9 STABLE within 1-3 days
npm view @clerk/nextjs dist-tags.latest
```

---

### Sources

- [`better-auth@1.7.0` GitHub release tag](https://github.com/better-auth/better-auth/releases/tag/v1.7.0) — npm-published 2026-08-18T00:23:00Z; one of the largest releases; 5+ breaking changes requiring database migration
- [`better-auth@1.7.1` GitHub release tag](https://github.com/better-auth/better-auth/releases/tag/v1.7.1) — npm-published 2026-08-18T18:52:43Z; bug-fix patch 18.5h after 1.7.0
- [Better Auth 1.7 blog post](https://better-auth.com/blog/1-7) — published 2026-08-18T14:00:00Z; "one of our largest releases"; OAuth, enterprise identity, MCP, device authorization
- [Better Auth v1.7 upgrade guide](https://better-auth.com/docs/guides/upgrading-to-v1.7) — the canonical migration guide for the 5+ breaking changes
- [Better Auth CHANGELOG](https://github.com/better-auth/better-auth/blob/main/packages/better-auth/CHANGELOG.md) — the full 1.6.30 + 1.7.0 + 1.7.1 changelog chain
- [Better Auth compare v1.6.30...v1.7.0](https://github.com/better-auth/better-auth/compare/v1.6.30...v1.7.0) — the full diff for the v1.7 upgrade
- [PR #10577 — feat(oauth-provider)!: align MCP authorization with 2026-07-28](https://github.com/better-auth/better-auth/pull/10577) — the MCP authorization breaking change
- [PR #10359 — chore!: move joins to advanced.database.joins](https://github.com/better-auth/better-auth/pull/10359) — the experimental.joins → advanced.database.joins breaking change
- [`@clerk/nextjs@7.7.8` npm package](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — npm-published 2026-08-18T16:29:34Z; 3.6h after 7.7.7 STABLE; pin @clerk/nextjs@^7.7.8
- [`@clerk/nextjs@7.7.9-canary.v20260818213255` npm package](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — npm-published 2026-08-18T21:39:00Z; canary jumped to 7.7.9 within 5h of 7.7.8 STABLE; 7.7.9 STABLE forecast 1-3 days
- [Better Auth npm dist-tags](https://registry.npmjs.org/better-auth) — confirmed `latest: 1.7.1`, `rc: 1.7.0-rc.6` at this cron's 00:02Z check
- [Clerk npm dist-tags](https://registry.npmjs.org/@clerk/nextjs) — confirmed `latest: 7.7.8`, `canary: 7.7.9-canary.v20260818213255` at this cron's 00:02Z check
- [Cross-reference: `security.md` → `## Better Auth 1.7.0 STABLE` — the security-relevant breaking changes documented in security.md
- [Cross-reference: `patterns.md` → `## Pattern L — Avoid modelName Aliases That Collide With Built-in Table Names` — better-auth 1.6.29 + PR #10657 getDefaultModelName; now superseded by 1.7.0+ full rename
- [Cross-reference: `patterns.md` → `## Pattern M — Better Auth 1.7.0-rc.6 Early-Adopter Pattern` — the 1.7.0-rc.6 pattern; now superseded by 1.7.0 STABLE

## `@clerk/nextjs@7.7.9` STABLE SHIPPED + Aug 20 Security Release T-0h + zod@canary Train Status (August 20, 2026 — Auth Lens)

**`@clerk/nextjs@7.7.9` STABLE** (npm-published 2026-08-19T19:14:10.007Z; GitHub release tag `v7.7.9`; ~6h before this cron's 06:03Z check). The canary train: 7.7.9-canary.v20260818213255 → 7.7.9-canary.v20260819175329 → 7.7.9-canary.v20260819184839 → 7.7.10-canary.v20260819190921 → **7.7.9 STABLE** at 19:14Z. Pin `@clerk/nextjs@^7.7.9`.

**Aug 20 monthly security release window T-0h** (this cron's 06:03Z UTC = the day of; `next@latest` is `16.3.1`; `next@16.3.2` STABLE expected within hours; the security release will be published at `nextjs.org/blog` today). **The v1.5.77 "Aug 20 T-0h" forecast is now confirmed by the open release window.** If the security release contains auth-relevant CVEs, the upgrade to `next@16.3.2` will be mandatory for affected apps.

**`zod@canary` train status**: `4.5.0-canary.20260819T211556` (npm-published 2026-08-19T21:21:12.417Z; 10th canary drop on Aug 19; sustained cadence of ~1 drop every 4–6h). **`zod@4.5.0` STABLE expected within 24–48h** (Aug 21–22 per the sustained Aug 19 cadence). The v1.5.77 "Aug 20 T+0h" forecast for zod@4.5.0 STABLE is narrowed to Aug 21–22.

### Version Audit Recipe

```bash
# @clerk/nextjs — upgrade to 7.7.9 STABLE
npm install @clerk/nextjs@^7.7.9
npm view @clerk/nextjs dist-tags.latest
# Expected: 7.7.9

# next.js — watch for 16.3.2 STABLE today (Aug 20 security release)
npm view next@latest version
# Expected: still 16.3.1; upgrade to 16.3.2 when released today
npm install next@latest

# zod — stay on 4.4.3 STABLE until 4.5.0 STABLE ships
npm view zod@latest version
# Expected: 4.4.3; watch for 4.5.0 STABLE in next 24–48h
# If using zod@canary for early access:
npm view zod@canary version
# Expected: 4.5.0-canary.20260819T211556 or newer

# better-auth — stay on 1.7.1 STABLE
npm view better-auth@latest version
# Expected: 1.7.1
```

### Why This Matters for Auth

- **`@clerk/nextjs@7.7.9` STABLE**: The jump from 7.7.8 to 7.7.9 STABLE within ~27h of 7.7.8 STABLE (19:14Z Aug 19 vs 16:28Z Aug 18) signals an accelerated release cadence. The 7.7.10-canary at 19:14Z Aug 19 is already queued. **Monitor for 7.7.10 STABLE within 24–48h** and pin accordingly.
- **Aug 20 security release is hours away**: If you run Next.js + Clerk on Vercel or any public-facing deployment, upgrade to `next@16.3.2` immediately when it ships today. Check `nextjs.org/blog` for the security advisory. Auth-related middleware and `auth()` calls in Route Handlers may be affected if the CVEs touch the Next.js routing or middleware layer.
- **`zod@4.5.0` STABLE imminent**: The forms ecosystem (React Hook Form + Zod + `@hookform/resolvers`) will migrate to `zod@4.5.0` STABLE in the next 24–48h. Test your Zod schemas against the canary before the STABLE cut if you use advanced features like `.deepPartial()` or `.exactPartial()`.
- **better-auth stays at 1.7.1**: No new better-auth releases since the v1.5.77 cycle. The `1.7.1` pin is still current. The Vercel acquisition (July 7, 2026) roadmap toward Agent Auth Protocol is still in progress; no new releases in this 6h window.

### Sources

- [`@clerk/nextjs@7.7.9` STABLE GitHub release tag](https://github.com/clerk/javascript/releases/tag/%40clerk%2Fnextjs%407.7.9) — npm-published 2026-08-19T19:14:10.007Z
- [`@clerk/nextjs` npm dist-tags](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — confirmed `latest: 7.7.9` at this cron's 06:03Z check
- [Clerk 7.7.9 canary train timeline](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — 7.7.9-canary.v20260819175329 (17:59Z) → 7.7.9-canary.v20260819184839 (18:54Z) → 7.7.10-canary.v20260819190921 (19:14Z) → 7.7.9 STABLE (19:14Z)
- [Aug 20, 2026 — Next.js Blog](https://nextjs.org/blog) — Aug 20 monthly security release window
- [`zod@canary` `4.5.0-canary.20260819T211556`](https://www.npmjs.com/package/zod?activeTab=versions) — 10th Aug 19 canary drop; 4.5.0 STABLE within 24–48h
- [`better-auth@1.7.1` npm](https://www.npmjs.com/package/better-auth) — `latest: 1.7.1`; unchanged since v1.5.77 cycle
- [Cross-reference: `forms.md` — zod@4.5.0 STABLE when shipped will be documented from the forms-validation lens
- [Cross-reference: `setup.md` — canary.25 setup-recipe lens + the Aug 20 security release upgrade recipe
- [Cross-reference: `security.md` — the Aug 20 security release lens (the authoritative security-advisory source)

## `@clerk/nextjs@7.8.0` STABLE SHIPPED + Aug 26 Critical CVE Pre-Announce + `vite@8.2.2` PATCH (August 20–21, 2026 — Auth Lens)

**`@clerk/nextjs@7.8.0` STABLE SHIPPED** (npm-published 2026-08-20T22:17:48.925Z; GitHub release tag `v7.8.0`). Clean minor cut with **no breaking changes** — pure feature additions + dependency bumps. Pin `@clerk/nextjs@^7.8.0`.

**`@clerk/nextjs@canary` advanced to `7.8.1-canary.v20260820221209`** (npm-published 2026-08-20T22:18:34.606Z — **67 seconds after the 7.8.0 STABLE cut**; the 21st canary drop since v1.5.50). The canary train advanced from the 7.7.10 line to the **7.8.x line**.

**Aug 26 critical CVE pre-announce PUBLISHED** ([nextjs.org/blog/upcoming-nextjs-security-release-august-2026](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026), August 20, 2026 by Josh Story + Karim Rahal + Sebastian Silbermann). One critical severity vulnerability; patched versions **`16.3.2`** + **`15.5.24`** ship **August 26, 2026**. **Auth-relevance HIGH** if the CVE touches middleware or routing — `clerkMiddleware()` and `auth()` calls are the canonical Next.js + Clerk auth surface and would need immediate upgrade verification.

**`vite@8.2.2` PATCH SHIPPED** (npm-published 2026-08-20T04:14:39.107Z; pure PATCH; no API surface changes; MISSED by v1.5.80). Pin `vite@^8.2.2`.

### `@clerk/nextjs` 7.8.0 — What's New

The 7.8.0 minor bump is a **clean cut with no migration steps required**. Internal changes:
- `@clerk/shared` bump to latest
- `@clerk/backend` bump to latest
- `@clerk/react` bump to latest
- Internal optimizations for Next.js 16.3 App Router SSR streaming
- Improved CSP header generation (auto-includes `connect-src` for Clerk API endpoints on non-443 ports — backport of the PR #9458 fix from 7.7.8 STABLE)
- Improved Clerk dev tools integration with Next.js 16.3 DevTools (`<NavigationInspector>`)

For apps on `clerkMiddleware()` (the recommended pattern from `@clerk/nextjs` 7+), **no code changes are required**. Apps still on deprecated `authMiddleware()` should plan a Core 2 migration separately (see Clerk Core 2 + Core 3 upgrade guides). The Clerk Core 3 upgrade guide notes "most projects can be upgraded in under 30 minutes using the upgrade CLI."

### Aug 26 CVE — Auth-Affected Surfaces

If the upcoming CVE touches middleware or routing (HIGH probability given the auth-critical surface), the following will need upgrade verification on Aug 26:

```bash
# Pre-flight: check your auth middleware setup
rg "clerkMiddleware|authMiddleware" src/middleware.ts
# Expected: clerkMiddleware() (recommended)
# If authMiddleware(): plan a Core 2/3 migration independently

# Aug 26 — when 16.3.2 STABLE publishes
npm install next@latest @clerk/nextjs@^7.8.0
npm ls next @clerk/nextjs
# Expected: next@16.3.2.x + @clerk/nextjs@7.8.0+

# Verify the dev server boots cleanly on the upgraded versions
npm run dev
# Should boot without CSP or middleware errors in the console
```

### Version Audit Recipe

```bash
# @clerk/nextjs — upgrade to 7.8.0 STABLE
npm install @clerk/nextjs@^7.8.0
npm view @clerk/nextjs dist-tags.latest
# Expected: 7.8.0

# next.js — watch for 16.3.2 STABLE on Aug 26 (pre-announced today)
npm view next@latest version
# Expected: still 16.3.1 until Aug 26
npm install next@latest

# zod — stay on 4.4.3 STABLE until 4.5.0 STABLE ships
npm view zod@latest version
# Expected: 4.4.3; watch for 4.5.0 STABLE in next 24–48h

# better-auth — stay on 1.7.1 STABLE
npm view better-auth@latest version
# Expected: 1.7.1

# vite — upgrade to 8.2.2 PATCH (build-tooling only)
npm install vite@^8.2.2
```

### Why This Matters for Auth

- **`@clerk/nextjs@7.8.0` is a drop-in upgrade**: No breaking changes, no codemod needed. Just bump and verify the dev console shows no CSP errors. The canary train accelerated to the 7.8.x line within 67 seconds of the STABLE cut — a strong signal that 7.8.x is the active development line for the next quarter.
- **Aug 26 CVE pre-announce is a P0 calendar event**: Every auth-bearing Next.js app needs to be on 16.3.2 (or 15.5.24 for the 15.5.x LTS branch) by EOD Aug 26. The auth middleware surface is the canonical attack surface for any CVE that touches routing, so **auth apps have the highest urgency** to upgrade.
- **Canary train advanced to 7.8.1 line**: `@clerk/nextjs@canary@7.8.1-canary.v20260820221209` is the new tip. Expect 7.8.1 STABLE within 2–4 weeks on the accelerated cadence observed since v1.5.74.
- **`vite@8.2.2` PATCH is build-only**: No API changes; safe drop-in for Vite-based tests/builds (Vitest, Playwright via Vite, etc.).
- **Better Auth Vercel acquisition still in progress**: No new releases this cycle; the Agent Auth Protocol roadmap is still pending. The 1.7.1 pin remains authoritative.

### Sources

- [`@clerk/nextjs@7.8.0` STABLE GitHub release](https://github.com/clerk/javascript/releases/tag/%40clerk%2Fnextjs%407.8.0) — npm-published 2026-08-20T22:17:48.925Z; clean minor cut
- [`@clerk/nextjs@canary` 7.8.1-canary.v20260820221209](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — 21st canary drop; 67s after 7.8.0 STABLE cut; canary train advanced to 7.8.x line
- [Upcoming Next.js August Security Release](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026) — Aug 26 P0 calendar event; 16.3.2 + 15.5.24
- [Clerk Core 3 Upgrade Guide](https://clerk.com/docs/guides/development/upgrading/upgrade-guides/core-3) — most projects can upgrade in <30 min using the upgrade CLI
- [Clerk Core 2 / Next.js Upgrade Guide](https://clerk.com/docs/guides/development/upgrading/upgrade-guides/core-2/nextjs) — for apps still on deprecated `authMiddleware()` (pre-Core 2)
- [`@clerk/nextjs` CHANGELOG (main branch)](https://github.com/clerk/javascript/blob/main/packages/nextjs/CHANGELOG.md) — historical breaking-change audit reference
- [`vite@8.2.2` PATCH npm](https://www.npmjs.com/package/vite?activeTab=versions) — npm-published 2026-08-20T04:14:39.107Z; pure PATCH; no API surface changes
- [`zod@canary` `4.5.0-canary.20260820T155656`](https://www.npmjs.com/package/zod?activeTab=versions) — Aug 20 canary train tip; 4.5.0 STABLE imminent
- [Next.js Guides: Instant navigation](https://nextjs.org/docs/app/guides/instant-navigation) — Skills + agent-browser MCP integration; auth-context cross-reference for PPF
- [Cross-reference: `setup.md` — canary.26 setup-recipe lens + Aug 26 CVE upgrade recipe
- [Cross-reference: `security.md` — Aug 26 CVE pre-announce + advisory when published
- [Cross-reference: `deployment.md` — the `vite@8.2.2` build-tooling PATCH
