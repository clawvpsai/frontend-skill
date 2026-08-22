# Security — XSS, CSRF, CSP, Input Sanitization + Next.js Security

## Why This Matters in 2026

The defensive posture in this file is calibrated to a specific shift: **attackers are now faster than the disclosure cycle**. Per ProjectDiscovery's "The Vulnerability Curve Bent With the AI Curve" (June 18, 2026), median time-to-exploit collapsed from **63 days (2018) → 5 days (2023) → negative (2024+)**, meaning vulnerabilities are now exploited *before* a CVE is published. At the same time, total published CVEs jumped from ~18,000/year in 2018 to a projected ~50,000/year in 2025–2026.

The implication: dependency hygiene, supply-chain vetting, and the "patch fast" reflex in this file are not paranoia — they are the only window defenders have. A package added to your `package.json` with a caret range can be pwned the same day. A typo in a Server Action ownership check is exploitable before your WAF rule is live.

Concrete rules of thumb that fall out of this:

- **Pin exact versions for security-critical deps** — `^1.11.21` lets an attacker bump the minor. Use exact versions or `npm ci` with a lockfile review.
- **Audit dormant maintainer accounts in your org scopes quarterly** — TanStack (May 11), node-ipc (May 15), Mini Shai-Hulud (June 1), Mastra (June 17), JetBrains Marketplace (June 16) all exploited stale contributor / publisher access.
- **Behavioral analysis > `npm audit`** — the Mastra `easy-day-js` typosquat had a clean `npm audit` profile. Use Socket.dev or Snyk for behavior, not just CVE matching.
- **Run `auth()` + ownership check inside Server Actions** — Server Actions are public POST endpoints reachable directly. Page-level checks do not protect them.
- **Treat WebCrypto, RSC, and middleware as the highest-value patch surface** — these are the most-cited vulnerable primitives in 2026 Next.js/React CVEs.

**Source:** [ProjectDiscovery — The Vulnerability Curve Bent With the AI Curve (June 18, 2026)](https://projectdiscovery.io/blog/the-vulnerability-curve-bent-with-the-ai-curve)

## Vercel Connect (June 17, 2026) — Scoped, Short-Lived Tokens for Agents

Launched at Vercel Ship 2026 (London, June 17, 2026), **Vercel Connect** is a new primitive in the Agent Stack that gives agents scoped, short-lived access to third-party APIs (Slack, GitHub, Snowflake, Salesforce, Notion, Linear, plus any OAuth/API service) **without** storing long-lived provider secrets in environment variables or your database.

### Why It Matters for Frontend Skills

If you (or an agent) are calling external APIs from a Next.js route, Server Action, or background worker, the standard pattern is to read a `SLACK_BOT_TOKEN` or `GITHUB_TOKEN` from `process.env`. That token is:

1. **Long-lived** — leaked once, valid forever (until manually rotated)
2. **Broad-scoped** — covers every action the agent might ever take
3. **Untraceable** — no record of which user authorized which action

Vercel Connect replaces that with **per-task, user-authorized, scoped tokens**. Your code calls a connector, the connector mints a short-lived token scoped only to the permissions the user explicitly granted, and the call is auditable end-to-end (user → agent → service).

### Pattern

```ts
// app/api/post-to-slack/route.ts
import { connect } from '@vercel/connect';

export async function POST(req: Request) {
  const user = await auth(); // your existing auth check
  if (!user) return new Response('Unauthorized', { status: 401 });

  // Mint a Slack token scoped to this single request, on behalf of this user
  const token = await connect.slack.tokenFor({ userId: user.id });

  const res = await fetch('https://slack.com/api/chat.postMessage', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ channel: req.body.channel, text: req.body.text }),
  });

  return Response.json(await res.json());
}
```

No `SLACK_BOT_TOKEN` in `.env`. No standing secret. Every token request is logged with the user that triggered it.

### When to Use Vercel Connect

- ✅ Agent calls third-party APIs (Slack, GitHub, Linear, Notion, etc.) on behalf of users
- ✅ Background jobs that need per-tenant credentials
- ✅ MCP servers that broker access to a user's data
- ❌ Your app's own database (use your normal auth, not Connect)
- ❌ Single-tenant backend services where a service account is appropriate

**Status:** Public beta. Supported providers at launch: Slack, GitHub, Snowflake, Salesforce, Notion, Linear. Any other service via OAuth or API.

**Sources:**
- [Vercel Ship 2026 recap](https://vercel.com/blog/vercel-ship-2026-recap)
- [The Agent Stack (June 17, 2026)](https://vercel.com/blog/agent-stack)
- [Introducing Vercel Connect](https://vercel.com/blog/introducing-vercel-connect)
- [Vercel Connect KB guide](https://vercel.com/kb/guide/vercel-connect)
- [Vercel Connect docs](https://vercel.com/docs/connect)

## Vercel Next.js Security Release Program (July 13, 2026)

On July 13, 2026, the Next.js core team (Andrew Imm + Josh Story) published a [formal security release program](https://nextjs.org/blog/next-security-release-program) — Next.js is moving from "ship fixes when we have them" to a **scheduled, pre-announced monthly security release cycle**. This is the same model Node.js, React, and most major OSS runtimes adopted years ago, and it changes how skill users should plan their patch cadence.

### What Changes Going Forward

- **Roughly once a month**, Vercel will publish advance notice of the upcoming security release on the Next.js blog. Each notice includes the expected release timeline and the highest anticipated severity.
- The release ships as **patch versions** on supported lines (currently Next.js 16.2 and 15.5). Skill users on Next.js 16.3 preview / canary should follow the canary branch for early fixes.
- CVEs and detailed advisories are published on the day of the release, not pre-announced. The advance notice tells you *when* and *how severe* — not *what*.
- The bug-bounty scope is expanded and Vercel runs the same class of internal tooling (deepsec) against Next.js that they ship to customers, so more issues reach defenders before they reach attackers.

### Upcoming July 20, 2026 Release — ACT NOW

The **first scheduled security release** is targeted for **Monday, July 20, 2026**. It will include patch releases for:

- **Next.js 16.2.x** → next 16.2 patch
- **Next.js 15.5.x** → next 15.5 patch

The advance notice (July 13) states it covers **4 high-severity + 5 medium-severity vulnerabilities**. Specific CVEs and GHSA IDs will be published on the day.

**Action items for the next 5 days:**

1. **Pin a calendar reminder for July 20, 2026** to upgrade `next` and `eslint-config-next` the same day. Subscribe to the [Next.js blog RSS](https://nextjs.org/blog) or watch the [vercel/next.js releases feed](https://github.com/vercel/next.js/releases) so you see the patch notes the moment they ship.
2. **Audit your current Next.js version** — anything older than the latest 16.2.x or 15.5.x patch will be missing prior fixes; the July 20 release will be a drop-in replacement.
3. **If you are on 16.3.0-canary.\* or 16.3.0-preview.\***: the canary branch receives most security fixes ahead of stable. Confirm your `next` dep resolves to a canary newer than the canary-branch HEAD on the day of the July 20 release (compare against the [`canary` npm dist-tag](https://www.npmjs.com/package/next?activeTab=versions)). Canary builds do not get formal CVE attribution, so do not run production on canary for security-sensitive apps.
4. **If you cannot upgrade immediately** (legacy app, blocked by a regression, third-party plugin incompatibility): the same WAF rules Vercel deployed for CVE-2026-23869 will likely activate for the high-severity items here. **WAF is not a substitute for patching** — the rules are best-effort and only apply to Vercel-hosted deployments.
5. **If you maintain an OSS Next.js plugin or example** that pins a `next` peer range: widen the range to `^16.2.0 || ^15.5.0` (or wider) so users can pick up the July 20 patch without forking your plugin.

### Pre-July-20 Last-24-Hour Prep Checklist (Updated 2026-07-19 — T-24h)

It is **Sunday, July 19, 2026 at 12:03 UTC** at the time of this writing — the scheduled security release is **Monday, July 20, 2026 (~24 hours from now)**. Saturday's T-48h inventory + Sunday's T-24h staging rehearsal are now consolidated into a single **last-call checklist** because there is no time left to do incremental work — everything must be ready so Monday is just a `npm install next@latest` + redeploy:

**Right now (Sunday) — final lock-down before Monday morning**

1. **Confirm `npm list next` was already run Saturday and recorded per app.** If you skipped Saturday's audit, do it NOW: note current version per app (`next@16.2.10`, `next@15.5.18`, `next@14.2.35`, `next@13.5.11`). The July 20 release patches 16.2.x and 15.5.x only — anything older is missing prior fixes; 13.x and 14.x do NOT get the patch (upgrade to 16.2 / 15.5 before Monday or remain exposed to BOTH the May 2026 batch AND the July 2026 batch forever).
2. **Confirm your Dockerfile / CI cache will not block the patch.** List deployable artifacts that pin `next` explicitly: Dockerfile `FROM` lines, base image digests, CI matrix versions, Helm chart values, Terraform modules, serverless function deploy configs. The patch is useless if your container image still pins `node:20-slim` + a frozen `package-lock.json` from June — bust the cache with `--no-cache` on `docker build` (or `docker buildx build --pull`) when you redeploy Monday.
3. **Staging rehearsal — do it TODAY, not Monday.** In a non-prod environment, run `npm install next@^16.2 --save-exact` (or `^15.5`), refresh the lockfile, then `next build` + your full test suite. Confirm zero regressions on the **patch-line bump** alone (the actual security patch will land as a small version bump on top of your current line; if the line-bump itself breaks you, you want to know Sunday, not during Monday's deploy window). If you have any `next.config.js` codemods pending (`npx @next/codemod@canary upgrade latest`), apply them today.
4. **For apps on canary/preview** (`16.3.0-canary.*` or `16.3.0-preview.*`): the canary branch receives most security fixes ahead of stable. Confirm your `next` dep resolves to a canary newer than the canary-branch HEAD on Monday (compare against the [`canary` npm dist-tag](https://www.npmjs.com/package/next?activeTab=versions)). **Caveat:** canary builds do not get formal CVE attribution, so do not run production on canary for security-sensitive apps.
5. **Open / confirm a tracking issue / Slack thread per app** with: current version, target version (the patched 16.2.x / 15.5.x), deploy owner, ETA. The patch itself is a one-line `next` bump; the work is the redeploy + verification.
6. **If you maintain OSS Next.js plugins or examples**, widen the peer range to `^16.2.0 || ^15.5.0` (or wider) so your users can pick up the July 20 patch without forking. Examples: `peerDependencies: { "next": "^16.2.0 || ^15.5.0" }`.
7. **Verify your CI dependency-update bot is healthy NOW.** Renovate / Dependabot should auto-open a PR within hours of the patch's Monday publication. If your bot is broken or rate-limited, set a manual calendar reminder for Monday morning UTC (08:00 UTC is a safe alert time — Vercel has historically shipped between 14:00 and 20:00 UTC, so the PR may land late afternoon).
8. **Pre-pull the Vercel WAF rule guidance** (if you self-host). Vercel said for the May 2026 release that "Vercel has not deployed new WAF rules for this release; these advisories cannot be reliably blocked at the WAF layer." Expect the same for July — but if you're behind Cloudflare / Fastly / AWS WAF, pre-stage a rule that drops requests matching the (about-to-be-disclosed) CVSS-9+ indicators once they're published. Do not rely on this — patching is the only complete mitigation.

**Monday — execute the patch + redeploy**

9. **Block calendar time Monday afternoon UTC** for the actual upgrade. The patch historically ships between 14:00–20:00 UTC (see [byteiota: Next.js Patches 9 Vulnerabilities on July 20](https://byteiota.com/next-js-patches-9-vulnerabilities-on-july-20-act-now) for community-tracking). `npm install next@latest`, verify `package-lock.json` was updated, bust your Docker cache (`docker build --no-cache` or `docker buildx build --pull`), redeploy, verify the new version is actually serving (`curl -I https://your-app.com` and inspect the `X-Powered-By: Next.js` version, or `npm list next` inside the running container).
10. **Subscribe to the [Next.js blog RSS](https://nextjs.org/blog)** or watch the [vercel/next.js releases feed](https://github.com/vercel/next.js/releases) live on Monday. Read the actual CVE descriptions as they publish — they often include workarounds that matter even on the patched version (e.g. "the patch fixes the auth bypass but you should also add `matcher` config to your middleware to harden the surface"). The 4 high + 5 medium CVEs typically land with rich writeups — read them, don't just upgrade.
11. **If you self-host behind Vercel WAF**, wait for Vercel's blog post to confirm which (if any) WAF rules they deployed — they generally do not block the high-severity items, so patching is the only path. If you self-host behind Cloudflare / Fastly / AWS WAF, add your pre-staged indicator-matching rule the moment the CVE descriptions publish.
12. **After patching**, add a recurring monthly calendar entry for the **20th of every month** going forward. The new Vercel program ships on the 20th every month — same-day upgrade is the expectation. Pair it with the Renovate / Dependabot auto-PR check above so the patch lands with zero human coordination on most months.

**What we know about the July 20 batch (pre-announcement, July 13):**

- **4 high-severity + 5 medium-severity CVEs**
- Patches on Next.js **16.2.x and 15.5.x** (no patches for 13.x / 14.x)
- CVE IDs and GHSA links published the day-of
- Aimed at: middleware / proxy bypass, Server Components / RSC, App Router / Pages Router, Image Optimization, Server Actions, and likely the canonical vuln classes that have dominated 2026 (cache poisoning, RSC payload corruption, route parameter injection)

**Sources:**
- [Next.js Security Release and Our Next Patch Release (July 13, 2026)](https://nextjs.org/blog/next-security-release-program)
- [socket.dev: Next.js moves to scheduled security releases (July 16, 2026)](https://socket.dev/blog/nextjs-moves-to-scheduled-security-releases)
- [byteiota: Next.js Patches 9 Vulnerabilities on July 20 — Act Now (July 17, 2026)](https://byteiota.com/next-js-patches-9-vulnerabilities-on-july-20-act-now)
- [gbhackers: Next.js Announces July Security Release (July 16, 2026)](https://gbhackers.com/next-js-announces-july-security-release/)

### Patch Cadence Best Practice (2026)

With the new monthly cadence, the recommended workflow is:

- **Production:** stay on the latest stable patch line (`next@^16.2` → `^15.5` etc.). Renovate / Dependabot should auto-open PRs within hours of each scheduled security release.
- **Staging:** upgrade the same day. Most months the patch is a pure CVE fix with no API change.
- **Canary/preview:** use for advance warning only — pin a specific canary version in `package.json` and rebuild on every canary bump. The canary branch gets security fixes days to weeks earlier than stable.
- **Vendored React (Next.js ships its own React):** always upgrade `next` and `react` together. Do not pin `react` to a different version than what your `next` version bundles — Vercel backports security patches only into the bundle `next` ships, not into a separately-installed `react`.

**Common mistake:** assuming "canary is ahead of stable so I am safer on canary." Canary is ahead on *features* but stable is usually behind on *patches* by a few weeks. For a security-sensitive production app, stable + prompt upgrade is the lower-risk posture; canary is for *early-warning* on a staging deploy.

**Sources:**
- [Next.js Security Release and Our Next Patch Release (July 13, 2026)](https://nextjs.org/blog/next-security-release-program)
- [Vercel Open Source Bug Bounty (HackerOne)](https://hackerone.com/vercel-open-source)
- [vercel-labs/deepsec — internal Next.js security tooling](https://github.com/vercel-labs/deepsec)


### July 21, 2026 — SECURITY RELEASE SHIPPED (Updated 2026-07-21 — 18:03Z)

**IT SHIPPED.** After a 1-day delay (original target: July 20, pushed to July 21 per the official Vercel blog banner), the July 2026 security release landed on **Monday July 21, 2026**. This was the 8th consecutive 6-hourly cron with a live status checkpoint — every prior checkpoint correctly tracked the release state without a false alarm.

**What shipped:**

- **`next@latest`** = `16.2.11` — published **2026-07-21T16:58:28Z** (GitHub) / **~16:00 UTC** (npm)
- **`next@backport`** = `15.5.21` — published **2026-07-21T16:58:17Z** (GitHub) / **~16:00 UTC** (npm)
- **`next@canary`** = `16.3.0-canary.92` — published **2026-07-21T17:51:18Z** (GitHub) / **~17:51 UTC** (npm)

**Upgrade now (do not delay):**
```bash
npm install next@latest   # → 16.2.11 (Active LTS)
npm install next@15.5.21   # → 15.5.21 (Maintenance LTS)
```

**Vercel blog post:** [July 2026 Security Release](https://nextjs.org/blog/july-2026-security-release) — published with full CVE details the same day.

**Sources:**
- [npm `next` package metadata (16.2.11 live)](https://registry.npmjs.org/next)
- [GitHub: `v16.2.11` release tag](https://github.com/vercel/next.js/releases/tag/v16.2.11)
- [GitHub: `v15.5.21` release tag](https://github.com/vercel/next.js/releases/tag/v15.5.21)
- [GitHub: `v16.3.0-canary.92` release tag](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.92)
- [Next.js blog: July 2026 Security Release](https://nextjs.org/blog/july-2026-security-release)
- [Netlify: Next.js security vulnerabilities (July 21, 2026)](https://www.netlify.com/changelog/2026-07-21-nextjs-security-vulnerabilities/)

### July 2026 CVEs — All 9 Fixed in 16.2.11 / 15.5.21

The July 2026 batch addressed **4 high-severity + 5 medium-severity vulnerabilities** across Next.js 16.2.x and 15.5.x. canary.92 (the 16.3.0 canary for the upcoming stable release) includes all 9 fixes plus the React vendor bump to `81e442ea-20260721`.

| Severity | CVE ID | Description | Attacker's Goal |
|---|---|---|---|
| **High** | [CVE-2026-64641](https://www.cve.org/CVERecord?id=CVE-2026-64641) | DoS via Server Actions — crafted requests cause CPU exhaustion, blocking all other in-process requests | Deny service; take down the app |
| **High** | [CVE-2026-64642](https://www.cve.org/CVERecord?id=CVE-2026-64642) | Middleware/Proxy bypass via Turbopack + single `i18n.locales` entry | Bypass auth, security checks, WAF rules |
| **High** | [CVE-2026-64645](https://www.cve.org/CVERecord?id=CVE-2026-64645) | SSRF via rewrites using attacker-controlled hostname — `rewrites()`/`redirects()` rule with user-supplied hostname can hit arbitrary hosts | Internal service scanning, credential theft |
| **High** | [CVE-2026-64649](https://www.cve.org/CVERecord?id=CVE-2026-64649) | SSRF via Server Actions on custom servers — when an action forwards/redirects, attacker-controlled `Host` headers send outbound requests to attacker-chosen hosts | Same as above |
| **Moderate** | [CVE-2026-64644](https://www.cve.org/CVERecord?id=CVE-2026-64644) | Image Optimization DoS — malicious SVG served via `/_next/image` when remote image loading is configured causes CPU exhaustion | Deny service via SVG bomb |
| **Moderate** | [CVE-2026-64646](https://www.cve.org/CVERecord?id=CVE-2026-64646) | Unbounded Server Action payload in Edge runtime — crafted request causes memory consumption | Memory exhaustion DoS |
| **Moderate** | [CVE-2026-64643](https://www.cve.org/CVERecord?id=CVE-2026-64643) | Internal Server Function endpoint disclosure — Server Action and `use cache` endpoint IDs globally disclosed with no auth | Reconnaissance; map your internal API surface |
| **Moderate** | [CVE-2026-64648](https://www.cve.org/CVERecord?id=CVE-2026-64648) | Cache confusion (same-URL, different body) — `fetch(new Request(url), differentInit)` may return wrong cached body | Response data leakage between users |
| **Moderate** | [CVE-2026-64647](https://www.cve.org/CVERecord?id=CVE-2026-64647) | Cache confusion (invalid UTF-8 bodies) — UTF-16 byte sequences in request body share same cache key as other invalid-UTF-8 bodies | Same as above |

**CVE → GHSA cross-reference:**

| CVE ID | GHSA ID |
|---|---|
| CVE-2026-64641 | GHSA-m99w-x7hq-7vfj |
| CVE-2026-64642 | GHSA-6gpp-xcg3-4w24 |
| CVE-2026-64643 | GHSA-955p-x3mx-jcvp |
| CVE-2026-64644 | GHSA-q8wf-6r8g-63ch |
| CVE-2026-64645 | GHSA-p9j2-gv94-2wf4 |
| CVE-2026-64646 | GHSA-4c39-4ccg-62r3 |
| CVE-2026-64647 | GHSA-4633-3j49-mh5q |
| CVE-2026-64648 | GHSA-68g3-v927-f742 |
| CVE-2026-64649 | GHSA-89xv-2m56-2m9x |

**What to do right now:**

1. **Upgrade `next` + `react` immediately** — `npm install next@latest react@latest react-dom@latest` (or `next@15.5.21` if on the 15.x line). Bust Docker cache and redeploy.
2. **If you use Turbopack + single `i18n.locales` entry** (CVE-2026-64642): auth/security checks in middleware are bypassed — audit your middleware for reliance on locale-based routing for security decisions.
3. **If you use `rewrites()` or `redirects()` with dynamic hostnames** (CVE-2026-64645): audit every `beforeFiles`/`afterFiles`/`fallback` rewrite where the destination hostname is built from `request.headers.get('host')`, `request.nextUrl`, or any user-supplied input. Add explicit allowlist validation.
4. **If you use custom servers with Server Actions** (CVE-2026-64649): audit every Server Action that calls `redirect()` or `NextResponse.redirect()`. Ensure the `Host` header in forwarded requests is validated against an allowlist, not passed through from the client.
5. **If you expose `/_next/image` with remote image loading enabled** (CVE-2026-64644): restrict the `remotePatterns` config to known-safe domains only. Do not use `hostname: '*'`.
6. **If you use Edge runtime Server Actions** (CVE-2026-64646): set `serverActions.bodySizeLimit` explicitly — do not rely on defaults.
7. **If you use `fetch()` with request bodies** (CVE-2026-64648 + CVE-2026-64647): audit every server-side `fetch(new Request(...), { body: ... })` pattern. Prefer `fetch(url, { body: ... })` form (not the two-arg form) or ensure cache keys include the body.
8. **Subscribe to the Next.js security release calendar** — the next release is targeted for **August 20, 2026** (same day each month). Set a calendar reminder now.

**Monthly cadence reminder:** Going forward, Vercel ships security patches on the **20th of each month** (or the next business day if the 20th falls on a weekend). Upgrade within 24h of each release — the window from announcement (7 days prior) to patch availability (the 20th) is your preparation runway.

**Sources:**
- [Next.js blog: July 2026 Security Release](https://nextjs.org/blog/july-2026-security-release) (full CVE descriptions)
- [GitHub: v16.2.11 release](https://github.com/vercel/next.js/releases/tag/v16.2.11)
- [GitHub: v15.5.21 release](https://github.com/vercel/next.js/releases/tag/v15.5.21)
- [GitHub: v16.3.0-canary.92 release](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.92)
- [Netlify Changelog: Next.js security vulnerabilities (July 21, 2026)](https://www.netlify.com/changelog/2026-07-21-nextjs-security-vulnerabilities/)

## CVE-2026-23869 — React RSC DoS (April 2026)

A high-severity denial-of-service vulnerability (CVSS 7.5) in React Server Components was disclosed April 8, 2026. The bug lives in the React Flight protocol's deserialization — a specially crafted HTTP request to any App Router Server Function endpoint can trigger excessive CPU consumption, crashing the server.

**Affected versions:**
- React 19.0.0 through 19.0.4
- React 19.1.0 through 19.1.5
- React 19.2.0 through 19.2.4
- All Next.js versions using App Router (13.x, 14.x, 15.x, 16.x)

**Fixed in:** React 19.0.5, 19.1.6, 19.2.5 (April 2026)
**Also patched in:** Next.js 16.2.6 (May 2026 security bundle)

**Upgrade:** `npm install react@latest react-dom@latest` then `npm install next@latest`

**Vercel WAF:** Vercel deployed automatic WAF rules to protect all Vercel-hosted projects, but you should still upgrade — WAF protection is not a substitute for patching.

### How It Works

The attacker sends a malformed RSC payload to a Server Function endpoint. When Next.js/React deserializes it via the Flight protocol, it triggers unbounded CPU usage. A single small request can take down the server.

### Mitigation If You Can't Upgrade Immediately

1. **Rate-limit Server Function endpoints** — limit requests per IP to routes handling RSC
2. **Block RSC endpoints at the edge** — use your hosting provider's WAF to filter suspicious payloads
3. **Disable Server Actions for unauthenticated users** — if possible, require auth before hitting any `'use server'` function

**Sources:**
- [Vercel: Summary of CVE-2026-23869](https://vercel.com/changelog/summary-of-cve-2026-23869)
- [NVD: CVE-2026-23869](https://nvd.nist.gov/vuln/detail/CVE-2026-23869)
- [Netlify: Next.js & React DoS vulnerability](https://www.netlify.com/changelog/2026-04-08-react-nextjs-dos-vulnerability/)
- [Imperva: React2DoS analysis](https://www.imperva.com/blog/react2dos-cve-2026-23869-when-the-flight-protocol-crashes-at-takeoff/)

## Next.js 16.2.6 Security Fixes (May 2026)

Next.js 16.2.6 is a **security-focused release** patching multiple high and moderate severity vulnerabilities. If you're on an earlier version, upgrade immediately.

### What Was Fixed

| Severity | Advisory | Issue |
|---|---|---|
| **High** | GHSA-8h8q-6873-q5fj | Denial of Service with Server Components |
| **High** | GHSA-267c-6grr-h53f | Middleware/Proxy bypass via segment-prefetch routes |
| **High** | GHSA-26hh-7cqf-hhc6 | Incomplete fix follow-up for middleware bypass |
| **High** | GHSA-mg66-mrh9-m8jx | DoS via connection exhaustion in Cache Components |
| **High** | GHSA-492v-c6pp-mqqv | Middleware bypass via dynamic route parameter injection |
| **High** | GHSA-c4j6-fc7j-m34r | SSRF via WebSocket upgrades |
| **Moderate** | GHSA-ffhc-5mcf-pf4q | XSS in App Router via CSP nonces |
| **Moderate** | GHSA-gx5p-jg67-6x7h | XSS in beforeInteractive scripts with untrusted input |
| **Moderate** | GHSA-h64f-5h5j-jqjh | DoS in Image Optimization API |
| **Moderate** | GHSA-wfc6-r584-vfw7 | Cache poisoning in RSC responses |
| **Low** | GHSA-vfv6-92ff-j949 | Cache poisoning via collisions in RSC cache-busting |
| **Low** | GHSA-3g8h-86w9-wvmq | Middleware redirect cache poisoning |

**Upgrade:** `npm install next@latest` to get 16.2.6 or later.


## React 19.2.4 Security Fixes (January 2026)

React 19.2.4 patches **three critical vulnerabilities** in React Server Components (RSC), affecting React 19.0 through 19.2.2. These were discovered after the initial React2Shell patches and affect any framework using RSC (Next.js, Remix, etc.).

### What Was Fixed

| CVE | Severity | Issue |
|---|---|---|
| CVE-2025-55184 | Critical | Source code exposure via crafted request to React Server Components |
| CVE-2025-67779 | Critical | Denial of Service via unbounded resource consumption in RSC |
| CVE-2025-55183 | High | Additional RSC parsing vulnerability (follow-up to December 2025 fixes) |

**Affected versions:** React 19.0.0 through 19.2.2
**Fixed in:** React 19.2.4 (January 26, 2026)
**Upgrade:** `npm install react@latest react-dom@latest`

### What Attackers Could Do

- **CVE-2025-55184**: Send a specially crafted request to trigger RSC parsing that leaks server-side source code (environment variables, secrets, internal logic)
- **CVE-2025-67779**: Exploit RSC's request handling to cause unbounded memory/CPU consumption (DoS)
- **CVE-2025-55183**: Additional RSC parsing flaw enabling further attacks on patched systems

### Mitigation

If you cannot upgrade immediately:

1. **Upgrade to React 19.2.4** — `npm install react@latest react-dom@latest`
2. **Audit RSC usage** — review `app/**/*.tsx` for Server Components that handle user-supplied data
3. **Rate limit RSC endpoints** — add rate limiting to any API that processes RSC payloads from untrusted sources
4. **Environment variable hygiene** — avoid using `process.env` directly in Server Components; use Next.js's env config patterns instead

**Sources:**
- [React blog: Denial of Service and Source Code Exposure in RSC](https://react.dev/blog/2025/12/11/denial-of-service-and-source-code-exposure-in-react-server-components)
- [Ox Security: React CVEs analysis](https://www.ox.security/blog/react-cve-2025-55184-67779-55183-react-19-vulnerabilities/)
---

## React DevTools Standalone HTML Injection (June 23, 2026) — canary fix

PR [#36839](https://github.com/facebook/react/pull/36839) (released as **React 19.3.0-canary-99e86060-20260623**, June 23, 2026) fixes an HTML-injection issue in the **React DevTools standalone shell** (`npx react-devtools`). Standalone DevTools errors received from the DevTools server were rendered into the page using HTML strings (innerHTML). A crafted error message could therefore inject arbitrary HTML — including `<script>` — into the DevTools surface when connecting to a remote/untrusted DevTools backend.

**The fix:** DevTools standalone now renders server errors as **DOM nodes built via `document.createElement` and `textContent`** instead of as innerHTML. The existing error box classes and copy are preserved; only the insertion mechanism changed. Because `textContent` treats input as literal text, any HTML-looking payload from the DevTools server is rendered as text, not parsed as markup.

### Why This Matters for Frontend Skills

- **Local-only attack surface in normal use.** React DevTools standalone opens a local WebSocket (default port 8097) to the page being inspected, and the page injects errors into the local DevTools UI. If you only ever connect to your own dev server, there is no remote attacker — the fix is hardening against future misuse.
- **Risk when connecting to a remote/shared DevTools server.** If you (or your team) use the standalone shell to inspect a page running on someone else's DevTools backend — common in shared QA environments, on flaky CI runners, or when a coworker is running `react-devtools` over a forwarded port — a malicious server can now no longer pivot into the inspector UI via crafted error text. Pre-patch, this was an actual injection vector; post-patch, it isn't.
- **Not a runtime user-facing CVE.** The injection point is inside the DevTools standalone app (a developer-only Electron app, not the page itself). End users are never exposed. No production user data is reachable through this path.
- **No version bump for stable yet.** The fix lives only in the canary builds (`react@19.3.0-canary-99e86060-20260623` and the matched experimental channel). It will roll forward into the next 19.x patch release on the stable channel.

### What To Do

- **No action required** if you only use the standalone DevTools locally against your own app — the fix is already in the channel your editor/browser DevTools uses.
- **Pin `react-devtools` to the latest stable** when installing the standalone CLI (`npm i -g react-devtools` / `npx react-devtools@latest`). Avoid running the standalone shell pointed at remote/untrusted DevTools servers on canary or pre-canary builds that predate this PR.
- **If you build your own dev-tooling on top of the standalone shell** — for example, a custom inspector that forwards errors from a remote page — adopt the same `createElement` + `textContent` pattern for any error text you render. Never use innerHTML, `dangerouslySetInnerHTML`, or template-literal interpolation for untrusted error copy.

**Sources:**
- [React PR #36839 — Avoid HTML injection in standalone errors (commit 99e86060, June 23, 2026)](https://github.com/facebook/react/pull/36839)
- [React canary commit 99e86060ac35ea81153ac39ddab9b4cd744d9391 — "Avoid HTML injection in standalone errors"](https://github.com/facebook/react/commit/99e86060ac35ea81153ac39ddab9b4cd744d9391)
- [react-devtools standalone docs — `npx react-devtools`](https://www.npmjs.com/package/react-devtools)

## TanStack npm Supply Chain Attack (May 11, 2026)

On May 11, 2026, an attacker published 84 malicious versions across 42 @tanstack/* npm packages via a compromised npm publisher account. This affected @tanstack/react-query, @tanstack/query-core, and all other TanStack packages.

**What happened:**
- Attacker combined a compromised npm publisher with legitimate package ownership takeover tactics
- 84 malicious versions published across 42 packages between 19:20 and 19:26 UTC
- Packages appeared legitimate with correct metadata, signatures reassigned

**Who was affected:**
- Anyone who installed or updated @tanstack/* packages during the 6-minute window (May 11, 2026, 19:20–19:26 UTC)
- GitHub Advisories: [GHSA-8xpr-6pg5-7r99](https://github.com/advisories/GHSA-8xpr-6pg5-7r99) and [GHSA-c2qf-rx4j-6f4g](https://github.com/advisories/GHSA-c2qf-rx4j-6f4g)

**What to do:**
1. **Audit lock files** — check if you pulled any @tanstack/* versions during the 6-minute window
2. **Use npm ci** — `npm ci` respects package-lock.json, preventing new malicious versions from being installed
3. **Pin versions** — use exact versions (`@tanstack/react-query@5.101.0`) in package.json, not ranges
4. **Check package-lock.json** — look for unexpected @tanstack/* versions published between 19:20–19:26 UTC on May 11, 2026
5. **Reinstall clean** — delete node_modules and package-lock.json, then `npm install` with known-good versions

**TanStack confirmed the incident and published guidance.** See their official postmortem:
- [TanStack npm supply-chain compromise postmortem](https://tanstack.com/blog/npm-supply-chain-compromise-postmortem)

**This is a reminder:** Always use `npm ci` in CI/CD pipelines, pin exact versions, and consider using a software composition analysis (SCA) tool like Socket.dev or Snyk to detect supply chain attacks.

## Vitest Browser Mode CVEs (May–June 2026)

Three **critical** vulnerabilities were published against Vitest's Browser Mode between May 19 and June 1, 2026. Vitest 4.0 made Browser Mode stable, and the skill explicitly recommends it for layout / visual regression / `IntersectionObserver` / etc. — so this is the most likely dev-time attack surface introduced by following the skill.

| CVE | GHSA | Severity | CVSS | Fixed in | Issue |
|---|---|---|---|---|---|
| CVE-2026-53633 | [GHSA-g8mr-85jm-7xhm](https://github.com/vitest-dev/vitest/security/advisories/GHSA-g8mr-85jm-7xhm) | **Critical** | 9.8 | vitest 4.1.8, 3.2.6, 5.0.0-beta.4 | `cdp()` API proxy → CDP `Page.setDownloadBehavior` overwrites `vite.config.ts` → RCE when browser API is network-exposed |
| (advisory-only) | [GHSA-2h32-95rg-cppp](https://github.com/vitest-dev/vitest/security/advisories/GHSA-2h32-95rg-cppp) | **Critical** | 9.6 | vitest 4.1.6, 5.0.0-beta.3 | Unsanitized `otelCarrier` query param injected into inline module script → recovers `VITEST_API_TOKEN` → chained RCE |
| (advisory-only) | [GHSA-5xrq-8626-4rwp](https://github.com/vitest-dev/vitest/security/advisories/GHSA-5xrq-8626-4rwp) | **Critical** | 9.8 | vitest 4.1.0 (Windows) | `__vitest_attachment__` path traversal (`\\?\..\`) → arbitrary file read + RCE when Vitest UI server is exposed via `--api.host` |

**The skill recommends `vitest@4.1.9` in Version Defaults, which is safe** — the 4.1.8 patch (June 1, 2026) is the most recent fix in the line. Run `npm ls vitest @vitest/browser` to confirm. If a user is locked to an older patch line, upgrade `vitest` **and** `@vitest/browser` together (they share the same version number and the fix is in the matching version).

### How the CDP RCE Works (GHSA-g8mr-85jm-7xhm)

1. Vitest Browser Mode exposes a `cdp()` RPC that forwards **raw** Chrome DevTools Protocol commands to the browser over the Vitest WebSocket.
2. The fix-gate (`browser.api.allowWrite` / `api.allowWrite` / `browser.api.allowExec` / `api.allowExec`) was supposed to block privileged CDP calls, but it didn't actually cover `cdp()` itself.
3. With the browser API exposed to the network (e.g. `--browser.api.host=0.0.0.0` — sometimes done in CI or remote dev containers), the generated browser runner page leaks the API token, active session id, project name, and project root path.
4. Attacker calls CDP `Page.setDownloadBehavior` to set the download dir to the project root, then CDP `Runtime.evaluate` to download a `vite.config.ts` they control. Vitest reloads the config and runs it in Node → RCE.

**Even with `allowWrite: false` and `allowExec: false` set, this CVE was exploitable in 4.1.7 and below.** Only the 4.1.8+ patch closes the gap.

### Safe Browser Mode Configuration (Vitest 4.1.0+)

Vitest 4.1.0 added two security gates that *are* respected post-4.1.8. Use them in `vitest.browser.config.ts`:

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    browser: {
      enabled: true,
      provider: playwright(),
      api: {
        // Default: true if api.host === 'localhost', false otherwise.
        // Setting explicitly to false in CI / shared dev environments
        // blocks the cdp() RPC, saveTestFile, and rerun APIs.
        allowWrite: false,
        allowExec: false,
      },
      // Don't bind to all interfaces unless you really need to.
      // host: '127.0.0.1' (default) keeps the browser API local-only.
    },
  },
})
```

**Operational rules:**
- **Never run Browser Mode on a network-exposed host** (Docker port-forward, public VM, dev container with `0.0.0.0` binding). The browser API was not designed for hostile network exposure.
- **CI: keep `allowWrite: false`, `allowExec: false`** — visual-regression tests don't need either.
- **Local dev: the defaults are safe** (localhost-only host ⇒ `allowWrite: true`, `allowExec: true`). If you need to share a session with a colleague, tunnel via SSH instead of binding to `0.0.0.0`.
- **If upgrading Vitest, also upgrade `@vitest/browser` and `@vitest/browser-playwright` (or `@vitest/browser-webdriverio`)** — they share the same version number and the fix is in the matching version.

### Vitest UI Server (Windows)

If you're on Windows, the Vitest UI server is vulnerable to arbitrary file read + RCE via the `__vitest_attachment__` endpoint when `--api.host` binds to anything other than localhost. 4.1.0+ fixes this. **On Windows, never run `vitest --ui --api.host=0.0.0.0`** — use `localhost` only, or run the UI inside a Linux container.

**Sources:**
- [GHSA-g8mr-85jm-7xhm — CDP RCE (CVSS 9.8)](https://github.com/vitest-dev/vitest/security/advisories/GHSA-g8mr-85jm-7xhm)
- [GHSA-2h32-95rg-cppp — otelCarrier XSS → RCE (CVSS 9.6)](https://github.com/vitest-dev/vitest/security/advisories/GHSA-2h32-95rg-cppp)
- [GHSA-5xrq-8626-4rwp — Vitest UI arbitrary file read + RCE on Windows (CVSS 9.8)](https://github.com/vitest-dev/vitest/security/advisories/GHSA-5xrq-8626-4rwp)
- [Vitest browser.api config — allowWrite / allowExec (4.1.0+)](https://main.vitest.dev/config/browser/api)
- [Vitest 4.1.8 release notes — cdp() client disabled when allowWrite/allowExec: false](https://github.com/vitest-dev/vitest/releases/tag/v4.1.8)
- [Vitest 4.1.6 release notes — otel carrier simplified](https://github.com/vitest-dev/vitest/releases/tag/v4.1.6)

## Mastra npm Scope Compromise (June 17, 2026)

On June 17, 2026, an attacker republished the entire `@mastra` npm scope — **142+ packages** (Mastra AI agent framework) — by abusing a **former contributor account whose scope access was never revoked**. Each compromised package added `easy-day-js` (a typosquat of the popular `dayjs` date library) as a runtime dependency. `easy-day-js@1.11.22` contained an obfuscated `postinstall` hook (`setup.cjs`) that disabled TLS verification, downloaded a second-stage payload from a raw IP address, and ran a cross-platform cryptocurrency stealer + remote access trojan — then deleted itself to erase evidence.

**This is the same attack pattern as the TanStack compromise (May 11, 2026) and the "Mini Shai-Hulud" campaigns** — compromised or stale maintainer credentials are used to mass-republish trusted packages with an injected malicious dependency.

### Attack Timeline

| Time (UTC) | Event |
|---|---|
| Jun 16 07:05 | Attacker npm account `sergey2016` publishes `easy-day-js@1.11.21` — a clean, fully functional `dayjs` clone with matching author metadata, version, homepage, repo, license, and keywords. Bait package. |
| Jun 17 | Attacker publishes `easy-day-js@1.11.22` — same metadata, now with obfuscated `setup.cjs` postinstall dropper. |
| Jun 17 | Attacker republishes 142+ `@mastra/*` packages (including `@mastra/core@1.42.1`, `mastra@1.13.1`, `create-mastra@1.13.1`), each adding `easy-day-js` as a `^1.11.21` dependency — pulling in the malicious `1.11.22`. |
| Jun 17 (discovery) | StepSecurity / Snyk / Hacker News publish coordinated disclosure. Mastra removes the unauthorized npm owner and ships clean forward-rolled versions. |

### Indicators of Compromise (IOCs)

- **Malicious package:** `easy-day-js` versions `1.11.21` and `1.11.22` (Snyk advisory [SNYK-JS-EASYDAYJS-17353313](https://security.snyk.io/package/npm/easy-day-js))
- **Stage-2 download:** `https://23.254.164.92:8000/update/49890878`
- **Stage-2 beacon:** `23.254.164.123:443`
- **Attacker npm account:** `sergey2016` (registered with `sergey2016@tutamail.com`)
- **Dropper file:** `setup.cjs` (~4,572 bytes, obfuscated) in the `easy-day-js` tarball
- **Affected ecosystem:** Entire `@mastra` npm scope — `@mastra/core` alone had ~4M downloads/month

### How the Typosquat Bypassed Visual Inspection

The `easy-day-js` package copied `dayjs`'s metadata wholesale to look legitimate in casual review:

| Attribute | Legitimate `dayjs` | Malicious `easy-day-js` |
|---|---|---|
| Version | `1.11.x` | `1.11.21` → `1.11.22` (mirrored) |
| Author metadata | `iamkun` | `iamkun` (copied) |
| Homepage | `https://day.js.org` | `https://day.js.org` (copied) |
| Repository | `github.com/iamkun/dayjs` | `github.com/iamkun/dayjs` (copied) |
| License | `MIT` | `MIT` (copied) |
| Keywords | `dayjs, date, time, moment` | `dayjs, date, time, moment` (copied) |
| Postinstall hook | None | `node setup.cjs --no-warnings` ⚠️ |
| Maintainer | `iamkun` | `sergey2016` ⚠️ (the only obvious signal) |

The only reliable distinguishing features are the **npm maintainer** and the **presence of `setup.cjs` in the tarball**. A `package.json` review or `npm audit` would not have flagged it.

### Why It Worked (Same Pattern as TanStack)

- **Dormant contributor account** still had publish rights on the `@mastra` scope
- **Trusted scope identity** — packages were republished under the legitimate `@mastra` namespace, so consumers saw no change in the package name
- **Clean bait version** (`1.11.21`) established credibility before the malicious `1.11.22` was published
- **Wide version range** (`^1.11.21`) ensured the malicious patch was pulled in automatically

### What to Do If You Installed `@mastra` Packages on June 17, 2026

1. **Treat the environment as compromised** — the postinstall hook ran at `npm install` time
2. **Rotate all credentials** that were present in the install environment: npm tokens, GitHub PATs, AWS/GCP keys, Vercel/Cloudflare tokens, SSH keys, `.npmrc` auth, signing keys, `~/.aws/credentials`, CI secrets
3. **Audit for the IOCs** above (stage-2 IPs, `easy-day-js` artifacts, `sergey2016` account references)
4. **Reinstall from clean versions** — delete `node_modules` and `package-lock.json`, then `npm install` after rotating
5. **Check for persistence** — the payload included a remote access trojan; run an AV/malware scan
6. **Forward-rolled packages** are now safe to install; pin to specific safe versions

### Defensive Measures (For All npm Projects)

1. **Use `npm ci` in CI/CD** — respects `package-lock.json`, prevents newly-published malicious versions from being installed mid-build
2. **Pin exact versions** — use `@tanstack/react-query@5.101.0` (no `^`/`~`) for high-value packages, especially in `dependencies` (not just `devDependencies`)
3. **Audit npm publish rights** — remove former maintainers from your org's npm scope immediately when they leave; treat npm scope access as production credential, not commit access
4. **Lock the lockfile** — enable `package-lock.json` in git and review all `package-lock.json` diffs in PRs (the TanStack attack showed up immediately in the lockfile)
5. **Run an SCA tool** — Socket.dev, Snyk, or GitHub's Dependabot security updates catch newly-published malicious versions within minutes
6. **Postinstall hooks are RCE** — if you have an npm `postinstall` script you didn't write, treat it as a code execution backdoor. Use `npm config set ignore-scripts true` for untrusted installs
7. **Watch for typosquats** — `easy-day-js` ≠ `dayjs`. Tools like `socket npm install <pkg>` flag lookalike packages at install time
8. **Use `minimumReleaseAge` (pnpm 11+ / npm 11.16+)** — define a minimum age (in minutes) that must pass between a package version's publish time and the moment your lockfile / install will pull it. A 7-day delay blocks nearly every short-lived supply-chain attack from the last 8 years, because malicious versions are usually detected and unpublished within hours. Configure in `.npmrc` (`minimum-release-age=10080` = 7 days) or `pnpm-workspace.yaml` (`minimumReleaseAge: 10080`). [pnpm docs](https://pnpm.io/supply-chain-security) · [OpenAI confirmed](https://openai.com/index/our-response-to-the-tanstack-npm-supply-chain-attack/) in their May 13, 2026 post-mortem on the TanStack compromise that they were deploying this control fleet-wide after two employee devices were impacted.

**The pattern is accelerating:** TanStack (May 11) → node-ipc malicious versions (May 15) → Mini Shai-Hulud variants (June 1) → Mastra (June 17). Every npm scope is a target. Lock down contributor access and pin everything.

**Sources:**
- [StepSecurity — Mastra npm supply chain attack analysis](https://www.stepsecurity.io/blog/mastra-npm-packages-compromised-using-easy-day-js)
- [Snyk — Mastra npm Scope Takeover (full technical writeup)](https://snyk.io/blog/a-forgotten-contributor-account-compromised-the-entire-mastra-npm-package-scope/)
- [Snyk advisory — easy-day-js embedded malicious code (SNYK-JS-EASYDAYJS-17353313)](https://security.snyk.io/package/npm/easy-day-js)
- [Kodem Security — IOCs and first-hour response playbook](https://www.kodemsecurity.com/resources/mastra-npm-packages-compromised-easy-day-js-supply-chain-attack-iocs-and-response-runbook)
- [Mastra issue #18045 — incident tracking](https://github.com/mastra-ai/mastra/issues/18045)


## Node.js June 2026 Security Release (June 17, 2026)

The Node.js project shipped a coordinated security release on **Wednesday, June 17, 2026** (announced June 18) covering all supported release lines: **Node 22 (LTS), Node 24 (LTS), and Node 26 (current)**. The release patches **12 CVEs** in total, with **2 rated HIGH** — one in `crypto.webcrypto` (WebCrypto AES DoS) and one in `tls` (wildcard-cert verification bypass). Frontend apps on Next.js / standalone Node / Docker / self-hosted workers all need to be on the patched line. The exact patch versions are **Node 22.23.0**, **Node 24.17.0**, and **Node 26.3.1** — the previous latest for each line is vulnerable.

### CVEs Fixed

| CVE | Severity | Component | Impact | Affected lines |
|---|---|---|---|---|
| [CVE-2026-48933](https://nodejs.org/en/blog/vulnerability/june-2026-security-releases) | **HIGH** | `crypto.webcrypto` (WebCrypto AES) | Integer overflow in `subtle.encrypt()` when the input length is a multiple of 2 GiB → **remote process abort (DoS)** | 22, 24, 26 |
| [CVE-2026-48618](https://nodejs.org/en/blog/vulnerability/june-2026-security-releases) | **HIGH** | `tls` hostname verification | Unicode dot-separator normalization mismatch between resolver and verifier → **TLS wildcard-depth authentication bypass** (a hostname can pass `tls.checkServerIdentity()` against a `*.example.com` cert while resolving to a different host) | 22, 24, 26 |
| [CVE-2026-48937](https://nodejs.org/en/blog/vulnerability/june-2026-security-releases) | **MEDIUM** | `node:http2` server | After a `GOAWAY` frame is sent in response to an invalid protocol error, the server keeps accepting new data → connection / memory exhaustion | 22, 24 |
| [CVE-2026-48936](https://nodejs.org/en/blog/vulnerability/june-2026-security-releases) | **LOW** | `--permission` flag | Unix-domain-socket server can be started without `--allow-net`, completing a previous partial fix for CVE-2026-21636 | 26 only |
| [CVE-2026-48931](https://nodejs.org/en/blog/vulnerability/june-2026-security-releases) | **LOW** | `http.Agent` | TOCTOU race lets a client accept a response that was sent **before** its request — response queue poisoning | 22, 24, 26 |
| _(7 more MEDIUM / LOW CVEs in this batch)_ | MEDIUM/LOW | TLS, HTTP/2, proxy, DNS, nghttp2, Permission Model | See the [Node.js June 2026 advisory](https://nodejs.org/en/blog/vulnerability/june-2026-security-releases) for the full list (6 MEDIUM + 1 additional LOW, depending on release line) | varies |

### Why the Second HIGH (CVE-2026-48618) Matters for Frontend Apps

Any Next.js app that does **outbound HTTPS to APIs whose certificates use wildcards** (`*.example.com`, `*.amazonaws.com`, `*.s3.amazonaws.com`, etc.) and relies on Node's built-in `tls.checkServerIdentity()` is exposed. The bug is a Unicode dot-separator normalization mismatch: an attacker who can register a hostname that contains a Unicode "dot-like" character (e.g. U+3002, U+FF0E, U+FF61) can make Node's resolver and verifier disagree about which host they're talking to. The result: `tls.checkServerIdentity()` passes the check against a wildcard cert, but the underlying connection resolves to a host the attacker controls.

**At-risk population:**
- Next.js Route Handlers / Server Actions that call third-party APIs over HTTPS using the built-in `fetch` + `tls` stack (almost all apps — Vercel, Stripe, GitHub, Slack, etc. all use wildcard certs)
- Custom HTTPS clients that override `checkServerIdentity` (sometimes done to disable cert checks in dev — **don't**)
- Apps using AWS SDK, OpenAI SDK, Anthropic SDK, etc. — all rely on the same underlying `tls` module

**Less exposed:**
- Apps that terminate TLS at a reverse proxy (nginx, Cloudflare, Vercel's edge) and talk plaintext to Node internally — Node's TLS verifier is never invoked
- Apps using a third-party HTTP client that does its own cert verification (rare)

**Mitigation:** bump to the patched Node version **AND** add `unicode-sanitization` on any user-controlled hostname before passing it to `fetch` / `tls.connect` — don't trust that the URL is ASCII-clean. Code that does `new URL(req.body.host).hostname` and then `fetch(\`https://${host}/...\`)` is the canonical foot-gun here.

```ts
// ✅ Defense in depth — strip non-ASCII dots from hostnames before TLS
const ASCII_DOT = /[\u3002\uFF0E\uFF61\u2024\u2025\u2026]/g
function safeHost(input: string): string {
  const url = new URL(input)
  if (ASCII_DOT.test(url.hostname)) {
    throw new Error(`Non-ASCII dot in hostname: ${url.hostname}`)
  }
  return url.hostname
}
```

### Bundled Dependency Upgrades (all release lines)

The release also rolls forward the bundled HTTP/2 / TLS / fetch stack:

- `llhttp` → **9.4.2**
- `nghttp2` → **1.69.0**
- `openssl` → **3.5.7**
- `undici` → **8.5.0** (Node 26.3.1), **7.28.0** (Node 24.17.0), **6.27.0** (Node 22.23.0)

If you pin Node via `engines` or `.nvmrc`, bump the patch: `22.23.0` / `24.17.0` / `26.3.1` (or any later). On Vercel, the `engines.node` field is honored by `@vercel/node` and triggers an automatic redeploy to a fresh Lambda runtime.

### Which Apps Are Exposed

- **Self-hosted Next.js** (`next start`, `output: 'standalone'`, Docker) — direct exposure. Bump the base image (`node:22.23.0-bookworm-slim`, `node:24.17.0-bookworm-slim`, or `node:26.3.1-bookworm-slim`).
- **Server Actions / Route Handlers that call `crypto.subtle.encrypt()` with attacker-controllable input** — exposed. The HIGH CVE is a remote DoS via the standard `subtle.encrypt()` API. If you accept untrusted input into WebCrypto (rare for frontend-only apps, common in workers / middleware that proxy crypto), you are exposed.
- **Any Node process that does outbound HTTPS to hosts with wildcard certs** — exposed to CVE-2026-48618 (HIGH). This is the common case for almost every app (Vercel APIs, GitHub, Stripe, AWS, OpenAI, etc.). The fix is the same: bump the Node patch version.
- **`node:http2` servers** (rare in Next.js apps, common in custom server gateways) — exposed.
- **Vercel / Netlify / Cloudflare managed runtimes** — the platform rolls forward; you don't control the version, but you can pin via `engines.node` to get a deterministic runtime.
- **`http.Agent` clients with high concurrency** — race condition is theoretical for typical SPA fetches but real for HTTP/2 multiplexed clients.

### Why the HIGH CVE Matters for Frontend Apps

CVE-2026-48933 is a **remote DoS in the standard `WebCrypto` API** — `globalThis.crypto.subtle.encrypt()` crashes the process if the input buffer length is exactly a multiple of 2 GiB (`2**31` bytes). Any code path that pipes user-controlled data into `subtle.encrypt()` (E2E-encrypted form fields, in-browser / server-side WebAuthn flows, custom JWT signing) can be crashed with a single request. Pin the Node patch version in CI, and add a length cap in your WebCrypto wrapper (defense in depth).

CVE-2026-48618 is a **TLS hostname-verification bypass** — a Unicode dot-separator normalization mismatch lets an attacker present a hostname that passes `tls.checkServerIdentity()` against a `*.example.com` cert but resolves to a different host. Any Next.js Route Handler / Server Action that does outbound HTTPS via `fetch` is exposed. Pin the Node patch version in CI, and add a Unicode-dot sanitizer in any code path that builds a `URL` from user input (defense in depth).

```ts
// ✅ Defense in depth — cap input before calling subtle.encrypt
const MAX_PLAINTEXT = 16 * 1024 * 1024 // 16 MiB; never let untrusted input approach 2 GiB
export async function safeEncrypt(data: ArrayBuffer, key: CryptoKey) {
  if (data.byteLength === 0 || data.byteLength > MAX_PLAINTEXT) {
    throw new RangeError('Invalid payload size')
  }
  return crypto.subtle.encrypt({ name: 'AES-GCM', iv }, key, data)
}
```

**Sources:**
- [Node.js — Thursday, June 18, 2026 Security Releases](https://nodejs.org/en/blog/vulnerability/june-2026-security-releases)
- [Node.js pre-alert on X (June 11, 2026)](https://x.com/nodejs/status/2064916362460295539)
- [Node.js release schedule / LTS lines](https://nodejs.org/en/about/previous-releases)
- [Digital Applied — Node.js June 2026 Security Releases: 12 CVEs, 2 HIGH (full list + patch guide)](https://www.digitalapplied.com/blog/nodejs-june-2026-security-releases-cve-patch-guide)
- [CVE-2026-48618 — TLS wildcard bypass via Unicode normalization (Node.js advisory)](https://nodejs.org/en/blog/vulnerability/june-2026-security-releases)

## Node.js July 29, 2026 Security Release — 9 CVEs / 3 HIGH (Originally July 27, Delayed Twice)

The Node.js project's [Wednesday, July 29, 2026 Security Releases](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases) advisory is **now published** after being postponed twice — first from Monday July 27 to Tuesday July 28 ("additional testing and validation"), then again to Wednesday July 29 ("infrastructure issues"). Patches are out for all three active release lines — **Node 22 (Maintenance LTS), Node 24 (Active LTS), Node 26 (Current)** — at versions **Node 22.23.2**, **Node 24.18.1**, **Node 26.5.1**. The release addresses **9 CVEs** total (3 HIGH, 5 MEDIUM, 3 LOW) plus **two bundled-dependency updates** (`undici` 8.9.0 / 7.29.0 / 6.28.0 and `llhttp` 9.4.3 across 26.x / 24.x / 22.x). The pre-announce-game ended with the HIGH ceiling confirmed in each line, and the released CVE list shows the ceiling holds: **HTTP/2 memory-exhaustion DoS**, **HTTP/2 re-entrant heap-use-after-free**, and **Permission Model path-matching over-grant** are all HIGH.

This is the **fourth Node.js security release of 2026** (after January 13, March 24, and June 17/18) and lands **8 days after the Next.js July 21, 2026 security release** (9 CVEs, 4 HIGH). Two patches in one week is a "patch week" — the same calendar pattern that has been repeating across the JavaScript stack (Next.js, Node.js, Vercel, OpenSSL) starting in late 2025. The two-day postpone-to-Wednesday pattern is unusual — Node.js hasn't postponed a security release twice in a row since the 2022 schedule change — so future release-day pre-announcement windows should leave a 3-day buffer to absorb similar delays.

### CVE List — 9 Total (3 HIGH, 5 MEDIUM, 3 LOW)

| CVE | Severity | Component | Affected Lines | Reporter / Fixer | Impact |
|---|---|---|---|---|---|
| [CVE-2026-56846](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases) | **HIGH** | `node:http/2` retained-header accounting | 24.x, 22.x | @leduckhuong / @mcollina | Retained header blocks can evade `maxSessionMemory` limits → remote memory exhaustion DoS |
| [CVE-2026-56848](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases) | **HIGH** | `node:http/2` `nghttp2_session_mem_send` re-entry | 26.x, 24.x, 22.x | @hahahkim / @mcollina | Re-entrant `mem_send` while `mem_recv` is executing → heap-use-after-free (potential RCE surface) |
| [CVE-2026-58043](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases) | **HIGH** | Permission Model radix-tree prefix matching | 22.x, 24.x, 26.x | @sy2n0 / @RafaelGSS | Granted path can over-grant read/write across radix-tree prefix boundaries under `--permission` |
| [CVE-2026-56850](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases) | MEDIUM | `https.Agent` PFX object-array key collision | 26.x, 24.x, 22.x | @yottt / @RafaelGSS | mTLS client identity reused across PFX certificates → cross-request identity leak |
| [CVE-2026-58040](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases) | MEDIUM | `https.Agent` TLS session reuse | 26.x, 24.x, 22.x | @vnyuh / @mcollina | TLS session reuse can skip hostname verification across identity policies (incomplete fix for CVE-2026-48934) |
| [CVE-2026-58041](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases) | MEDIUM | `node:sqlite` `SQLTagStore` iterator | 26.x, 24.x | @cantina-security / @mcollina | Stale `StatementSyncIterator` from `DatabaseSync#createTagStore()` continues executing a cached prepared statement after reset+rebind → write replay |
| [CVE-2026-58042](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases) | MEDIUM | `node:dns` `dns.resolveAny()` | 26.x, 24.x, 22.x | @cantina-security / @RafaelGSS | DNS response with >256 A records aborts the process → DoS |
| [CVE-2026-58045](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases) | MEDIUM | `node:zlib` sync APIs | 26.x, 24.x, 22.x | @byvini / @RafaelGSS | Spoofed `TypedArray.byteLength` triggers a reachable assertion in sync zlib → process crash (DoS) |
| [CVE-2026-56847](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases) | LOW | Permission Model `trace_events` | 26.x, 24.x, 22.x | @0xoroot / @RafaelGSS | `trace_events.createTracing().enable()` writes trace logs outside `--allow-fs-write` paths |
| [CVE-2026-58039](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases) | LOW | Permission Model `process.report` | 26.x, 24.x, 22.x | @sinan-polat / @RafaelGSS | `process.report` writes / overwrites files outside `--allow-fs-write` paths |
| [CVE-2026-58044](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases) | LOW | `node:http` parser header truncation | 26.x, 24.x, 22.x | @yushengchen / @mcollina | Headers beyond `maxHeadersCount` / `maxHeaderPairs` are omitted from `req.headers` / `req.rawHeaders` / `req.headersDistinct` but still used for HTTP framing → request smuggling in Node-based forwarding proxies |

### Bundled-Dependency Updates (Same Release)

| Dependency | Affected Lines | Bumped To | Vulnerability Class |
|---|---|---|---|
| `undici` | 26.x, 24.x, 22.x | **8.9.0** / **7.29.0** / **6.28.0** | Public fetch-stack fixes rolled forward (also bumped independently on npm — see [undici CVEs June 2026](#undici-cves-june-19-2026) section for the prior CVE set) |
| `llhttp` | 26.x, 24.x, 22.x | **9.4.3** | HTTP parser hardening |

Apps that pin `undici` directly via `npm overrides` / `pnpm.overrides` — and SDKs that bundle their own `undici` (AWS SDK v3, OpenAI SDK, Anthropic SDK, Supabase, Prisma, NextAuth, WebSocket clients) — should run `npm ls undici` after the patch and refine the override range to match the new bundled version (`^6.28.0` / `^7.29.0` / `^8.9.0`).

### Release Versions & Affected Lines

| Field | Value |
|---|---|
| Release date | **Wednesday, July 29, 2026** (postponed twice from Monday July 27 — first to Tuesday July 28 "additional testing and validation", then to Wednesday July 29 "infrastructure issues") |
| Patch versions | **Node 22.23.2** · **Node 24.18.1** · **Node 26.5.1** |
| Affected lines | **Node 22.x** (Maintenance LTS), **Node 24.x** (Active LTS), **Node 26.x** (Current) — all three |
| CVE count | **9** (3 HIGH, 5 MEDIUM, 3 LOW) |
| Dependency updates | `undici` 8.9.0 / 7.29.0 / 6.28.0 + `llhttp` 9.4.3 |
| Node.js 18 / 20 | **EOL — no upstream patches.** Node 20 EOL April 30, 2026; Node 18 EOL April 30, 2025. |
| EOL-line maintained patches | [HeroDevs Node.js NES](https://www.herodevs.com/) — drop-in patched builds for EOL 18 / 20 |
| Full disclosure URL | <https://nodejs.org/en/blog/vulnerability/july-2026-security-releases> |
| Status as of 2026-07-29 18:03Z | Release published; patches available on nodejs.org / Docker Hub / GHCR |

### Pre-Patch Audit Recipe (Run Before You Bump)

The patch is useless if you can't find the deployment. Inventory every Node version you have running:

```bash
# Local — current node (this skill's host machine is on Node 24.18.0 → bump to 24.18.1)
node -v

# Docker — every base image
grep -rE "FROM node:.*[0-9]+\.[0-9]+\.[0-9]+" --include=Dockerfile . 2>/dev/null
grep -rE "FROM node:.*-(bookworm|slim|alpine)" --include=docker-compose.y*ml . 2>/dev/null

# CI — every matrix entry
grep -rE "node-version:|setup-node" .github/workflows/ 2>/dev/null

# Vercel / engines.node per project
grep -rE "engines.node" --include=package.json . 2>/dev/null

# Vercel / Netlify / Cloudflare — declarative node in the platform config
grep -rE "engines.node|NODE_VERSION" .vercel/ netlify.toml .nvmrc 2>/dev/null
```

Record the output per app. Target upgrade list is the existing version + bump to the **22.23.2 / 24.18.1 / 26.5.1** line.

### Required Actions (On Patch Day, Now)

The pattern from the June 17, 2026 release is the playbook. Frontend apps on Next.js / standalone Node / Docker / self-hosted workers / Vercel Lambda functions / Cloudflare Pages (build pipelines) all need to be on the patched line. Concrete steps:

1. **The [July 29, 2026 advisory](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases) is now published.** Patch versions are **Node 22.23.2**, **Node 24.18.1**, **Node 26.5.1**. Full CVE list is in the table above (9 CVEs — 3 HIGH in HTTP/2 + Permission Model, 5 MEDIUM, 3 LOW). The two-day postpone-to-Wednesday pattern is unusual; future pre-announcement windows should leave a 3-day buffer.
2. **Update Docker base images immediately** — bump `FROM node:22.x-bookworm-slim` → `node:22.23.2-bookworm-slim`, etc. **Bust the cache** with `--no-cache` on `docker build` (or `docker buildx build --pull`) so the patched layers are actually pulled in. Pin by digest (`@sha256:...`) for production deployments so the next layer-cache scrub doesn't regress you.
3. **Update `.nvmrc` / `engines.node`** — pin the new patched version in `package.json`'s `engines.node` (e.g. `"node": ">=22.23.2"` or use a `.nvmrc` with the exact patch version). Vercel and Netlify honour the `engines.node` field and trigger automatic redeploys to a fresh Lambda runtime.
4. **CI matrix** — update GitHub Actions `actions/setup-node` matrix versions, Renovate / Dependabot config, and base-image digests in any Dockerfile that pins a specific Node version. The patch is useless if your CI still runs on a frozen lockfile from June.
5. **Vercel deployment** — set `engines.node` in `package.json` to the patched version; Vercel uses the highest `engines.node` declared across the project's serverless functions to pick the runtime. After the bump, force a redeploy (or wait for the next git push) to migrate to the new Lambda runtime.
6. **Watch for the `undici` / `llhttp` / `nghttp2` / `openssl` bundled-version bumps** — this release ships `undici` **8.9.0 / 7.29.0 / 6.28.0** and `llhttp` **9.4.3** (the 28-Jun-2026 bundled versions prior to this release were `undici` 8.5.0 / 7.28.0 / 6.27.0 and `llhttp` 9.4.2; the June 17 release bumped `llhttp` 9.4.2, `nghttp2` 1.69.0, `openssl` 3.5.7, `undici` 6.27.0 / 7.28.0 / 8.5.0). Apps that pin `undici` directly via `npm overrides` / `pnpm.overrides` need to refine the override range to the new bundled version. Run `npm ls undici` after the patch.
7. **Subscribe to the low-volume `nodejs-sec` mailing list** — the [Node.js security policy](https://nodejs.org/en/security/) page links to `https://groups.google.com/forum/#!forum/nodejs-sec` for release-day announcements.
8. **Node 18 / Node 20 users** — Node 18 EOL April 30, 2025; Node 20 EOL April 30, 2026. **Neither line receives upstream security fixes** — including for this July 29 release. HeroDevs ships [Node.js NES](https://www.herodevs.com/) drop-in EOL builds if you can't migrate to Node 22 LTS yet. For new projects, always target Node 22 LTS (EOL April 30, 2027) or Node 24 LTS (EOL April 30, 2028).
9. **If you maintain OSS Node-based packages** — widen the `engines.node` peer range to `^22.0.0 || ^24.0.0 || ^26.0.0` (or at least the patched versions) so users can pick up the July 29 patch without forking your plugin.

### What "HIGH" Actually Means in This Release

Cross-referencing the March / June 2026 Node.js security releases (the two most recent before this July 29 release): HIGH-severity Node.js CVEs in 2026 have tended to land in **crypto / TLS / HTTP** subsystems — `crypto.webcrypto` integer overflow (CVE-2026-48933, June), TLS wildcard-cert Unicode bypass (CVE-2026-48618, June), `node:http` DoS through `__proto__` header (CVE-2026-21710, March), `node:tls` SNICallback remote crash (CVE-2026-21637, March). The HIGH CVEs in this July 29 release **match the historical pattern precisely**: two are in `node:http/2` (CVE-2026-56846 retained-headers memory-exhaustion, CVE-2026-56848 re-entrant heap-use-after-free) and one is in the **Permission Model** (CVE-2026-58043 path-over-grant across radix-tree prefix boundaries). The Permission Model HIGH is the first of 2026 — every prior 2026 HIGH was a crypto/TLS/HTTP DoS class. **Practical implication:** if you run Node with `--permission` (rare but growing on serverless edge runtimes + Deno-like sandboxes), CVE-2026-58043 is the one to prioritize — radix-tree-prefix-boundary over-grant is a security-boundary bypass, not a DoS, and the same code path is used by `fs.readFile` / `fs.writeFile` / `fs.watch` / `fs.createReadStream`. The HTTP/2 violations are the dominant risk for the typical frontend-stack Node service (the Next.js prod server, standalone Node, Vercel Lambda, custom Docker, K8s cluster) — both are **remotely triggerable** from any HTTP/2 client, and CVE-2026-56848 in particular is a heap-use-after-free class (the kind of memory-safety bug that frequently gets turned into RCE by an experienced attacker).

### Why This Matters for Frontend Skills

- **Next.js apps on Vercel**: Vercel hosts the Lambda runtime. The `engines.node` field in `package.json` is the only lever you have to force a runtime upgrade; bump it the same day as the patch ships.
- **Self-hosted Next.js** (`next start`, `output: 'standalone'`, Docker, K8s, ECS, Nomad): direct exposure. Bump the base image tag and `--no-cache` the build.
- **Edge runtimes running on Cloudflare Workers / Vercel Edge**: NOT affected (use workerd or Vercel's Edge runtime, not Node). But CI pipelines that build these still run on Node.
- **Microservices calling outbound HTTPS** through Node's `tls` / `https` modules: the March / June pattern (CVE-2026-21637, CVE-2026-48618) extends here — CVE-2026-58040 is the **incomplete fix for CVE-2026-48934** surfacing again, and CVE-2026-56850 (mTLS identity reuse across PFX) is the new addition. Apply the same defense-in-depth hostname sanitization as for the June CVE (see ASCII-dot sanitizer in the [Node.js June 2026](#nodejs-june-2026-security-release-june-17-2026) section above).
- **HTTP/2 servers** (any custom `node:http2` server, Next.js's own prod server, fetch-with-prior-knowledge clients): the two HIGH CVEs land here. If you have a Node-based HTTP/2 reverse proxy (e.g. `http2-wrapper` + `http-proxy` patterns, or any custom `createSecureServer` in front of a Vite/Turbopack dev server), prioritise the upgrade — CVE-2026-56848 is a remotely-triggerable heap-use-after-free.
- **CI runners / build agents**: often the longest-lived Node version in a stack. Bump the GitHub Actions `actions/setup-node` / `node:XX-bookworm` image the same day.
- **Vercel Connect / OpenAI SDK / Anthropic SDK / AWS SDK / Supabase / Prisma / NextAuth / WebSocket clients**: SDKs that bundle `undici` directly through `npm overrides` or `pnpm.overrides` may need the override range adjusted to match the new bundled version (`^6.28.0` / `^7.29.0` / `^8.9.0`). Run `npm ls undici` after the patch.

### Post-Patch Verification Recipe

Once the bump is in, sanity-check that the new version is actually live:

```bash
# 1. Verify the Node binary version
node -v
# expect: v22.23.2 / v24.18.1 / v26.5.1

# 2. Verify the bundled undici version
node -e "console.log(process.versions.undici)"
# expect: 6.28.0 / 7.29.0 / 8.9.0

# 3. Verify the bundled llhttp version
node -e "console.log(process.versions.llhttp)"
# expect: 9.4.3

# 4. Verify the bundled nghttp2 version (unchanged)
node -e "console.log(process.versions.nghttp2)"
# expect: 1.69.0

# 5. Verify the bundled OpenSSL version (unchanged)
node -e "console.log(process.versions.openssl)"
# expect: 3.5.7

# 6. Permission Model regression test (for CVE-2026-58043)
node --permission --allow-fs-read=/tmp/grant \
  --allow-fs-write=/tmp/grant \
  -e "require('fs').writeFileSync('/tmp/grant/escape.txt', 'pwned')"
# expect: ERR_PERMISSION_DENIED (in 22.23.0 / 24.18.0 / 26.5.0 the write succeeds due to the radix-tree prefix bug)
```

### Cross-Reference: 2026 Node.js Security-Release Cadence

| Release | Date | Lines | CVEs | HIGH | Notes |
|---|---|---|---|---|---|
| Node.js January 2026 | Mon Jan 13 | 22, 24, 26 (current) | (per advisory) | (per advisory) | First 2026 release |
| Node.js March 2026 | Tue Mar 24 | 20, 22, 24, 25 | 8 | 2 (CVE-2026-21637 TLS SNICallback, CVE-2026-21710 `__proto__` header) | 8 CVEs incl. two HIGH process crashes |
| Node.js June 2026 | Wed Jun 17/18 | 22, 24, 26 | 12 | 2 (CVE-2026-48933 WebCrypto DoS, CVE-2026-48618 TLS wildcard bypass) | See [Node.js June 2026](#nodejs-june-2026-security-release-june-17-2026) section above |
| **Node.js July 2026** | **Wed Jul 29** (postponed twice from Mon Jul 27) | **22, 24, 26** | **9** | **3 (CVE-2026-56846 HTTP/2 memory DoS, CVE-2026-56848 HTTP/2 heap UAF, CVE-2026-58043 Permission Model over-grant)** | **This section** |
| Next.js July 2026 | Tue Jul 21 | 16.2, 15.5 | 9 | 4 | First drop of Vercel's monthly security release program (see [Vercel Next.js Security Release Program](#vercel-next-js-security-release-program-july-13-2026) section above) |

### Sources

- [Node.js — Wednesday, July 29, 2026 Security Releases (RELEASED — 9 CVEs, 3 HIGH)](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases)
- [Node.js vulnerability blog (live CVE feed)](https://nodejs.org/en/blog/vulnerability)
- [Node.js security policy / `nodejs-sec` mailing list](https://nodejs.org/en/security/)
- [Node.js release schedule / LTS lines](https://github.com/nodejs/release)
- [Node.js — Thursday, June 18, 2026 Security Releases (12 CVEs, 2 HIGH — same playbook)](https://nodejs.org/en/blog/vulnerability/june-2026-security-releases)
- [Node.js — Tuesday, March 24, 2026 Security Releases (8 CVEs, 2 HIGH)](https://nodejs.org/en/blog/vulnerability/march-2026-security-releases)
- [TechTimes — Node.js Security Release: HIGH Severity Vulnerabilities Hit All Three Lines (July 23, 2026)](https://www.techtimes.com/articles/321432/20260723/nodejs-security-release-high-severity-vulnerabilities-hit-all-three-lines.htm)
- [Digital Applied — Next.js and Node.js Patch Week: What to Update Before Monday (July 25, 2026)](https://www.digitalapplied.com/blog/nextjs-nodejs-july-2026-security-patch-week)
- [Teramont — Node.js Security Release: July 27, 2026 (pre-announcement analysis)](https://teramont.net/blog/nodejs-security-release-july-27-2026)
- [byteiota — Node.js July 2026 Security Patch: Patch All Three Lines Now (July 27, 2026)](https://byteiota.com/nodejs-july-2026-security-patch/)
- [HeroDevs — Node.js NES for EOL 18 / 20 lines (drop-in patches)](https://www.herodevs.com/)
- [undici 8.9.0 / 7.29.0 / 6.28.0 release notes](https://github.com/nodejs/undici/releases)
- [NGHTTP2 release history (1.69.0 ship with Node 26.5.0 / 24.18.0 / 22.23.0)](https://github.com/nghttp2/nghttp2/releases)



## undici CVEs (June 19, 2026)

On **June 19, 2026** the GitHub Advisory Database published **three new CVEs** against the standalone `undici` npm package (the HTTP/1.1 client that powers Node.js's built-in `fetch` and most third-party SDKs). These are the same fixes that were just bundled into the Node.js June 18 release, but they are published as **separate npm advisories** — meaning the GitHub Advisory feed, Dependabot, and `npm audit` only flag them if you have `undici` in your dependency tree (directly or transitively), not if you're on a patched Node.

| CVE | Severity | Component | Impact | Fixed in |
|---|---|---|---|---|
| [CVE-2026-11525](https://github.com/advisories?query=undici+type%3Areviewed) | **HIGH** | `undici` WebSocket client | DoS via fragment count bypass — attacker can crash a Node process with a crafted WebSocket frame | `undici` 7.28.0 / 8.5.0 |
| [CVE-2026-12151](https://github.com/advisories?query=undici+type%3Areviewed) | MODERATE | `undici` `fetch` | HTTP header injection via `Set-Cookie` percent-decoding — a malicious server can inject arbitrary headers into the response | `undici` 7.28.0 / 8.5.0 |
| [CVE-2026-9679](https://github.com/advisories?query=undici+type%3Areviewed) | LOW | `undici` `fetch` | `Set-Cookie` SameSite attribute downgrade via permissive substring matching | `undici` 7.28.0 / 8.5.0 |

### Why This Matters Even Though the Node.js Patch Is Out

Every Node 18+ app uses `undici` for `fetch` — but `process.versions.undici` (the version Node bundles) is what you get by default. If you have `undici` listed anywhere in `package.json` (direct, dev, or peer), or you import a package that depends on `undici` at install time, you may get the **npm-published version**, not the bundled one. Affected ecosystem packages include (in order of frequency seen in real apps):

- `@aws-sdk/*` (AWS SDK v3 uses `undici` as its default HTTP handler)
- `@anthropic-ai/sdk` and `openai` (both use `undici` for fetch when not in the browser)
- `@supabase/supabase-js`, `@planetscale/database`, `@neondatabase/serverless`
- `@prisma/client` (in some configurations)
- `discord.js`, `tiktok-live-connector`, and any WebSocket-heavy client
- `next-auth` / `@auth/core` (uses `undici` for OAuth provider calls in some adapter paths)

If your `package.json` has no `undici` line but the SDKs above are in your tree, `npm ls undici` will show you what version `npm` actually resolved.

### Three Things To Do Today

1. **On a patched Node line, you're safe by default.** Node 22.23.0 / 24.17.0 / 26.3.1 ships `undici` 6.27.0 / 7.28.0 / 8.5.0 respectively — all three CVE-fix lines. Bump the base image tag or `engines.node`.
2. **If you depend on a package that bundles its own `undici`**, `npm ls undici` to see the resolved version. If it's older than 6.27.0 / 7.28.0 / 8.5.0, add a top-level `overrides` (npm) or `pnpm.overrides` block:

   ```jsonc
   // package.json
   {
     "overrides": {
       "undici": "^7.28.0"  // or "^8.5.0" if you want to track the 8.x line
     }
   }
   ```

   ```yaml
   # pnpm-workspace.yaml
   pnpm:
     overrides:
       undici: ^7.28.0
   ```

3. **For Server Actions and Route Handlers that call `fetch` with user-supplied URLs**, the same `safeHost()` Unicode-dot sanitizer from the Node.js section above applies — the `undici` 7.28.0+ `fetch` is still subject to `tls.checkServerIdentity()` and is still vulnerable to the wildcard-bypass class of bugs in unpatched Node. Defense in depth.

**Sources:**
- [GitHub Advisory: undici WebSocket client DoS via fragment count bypass (CVE-2026-11525)](https://github.com/advisories?query=undici+type%3Areviewed)
- [GitHub Advisory: undici HTTP header injection via Set-Cookie percent-decoding (CVE-2026-12151)](https://github.com/advisories?query=undici+type%3Areviewed)
- [GitHub Advisory: undici Set-Cookie SameSite attribute downgrade (CVE-2026-9679)](https://github.com/advisories?query=undici+type%3Areviewed)
- [undici on npm (7.28.0 / 8.5.0 release notes)](https://www.npmjs.com/package/undici)
- [Node.js June 18, 2026 Security Release (bundles the same undici fixes)](https://nodejs.org/en/blog/vulnerability/june-2026-security-releases)

## JetBrains Marketplace Malicious Plugins (June 16, 2026)

On **June 16, 2026**, Aikido Security disclosed a coordinated malware campaign on the JetBrains Marketplace: **at least 15 third-party IDE plugins** published under **7 vendor accounts**, all variants of the same codebase, that **silently exfiltrate AI provider API keys** (OpenAI, Anthropic, DeepSeek, SiliconFlow, Google AI) the moment you paste them into the plugin's settings panel. The campaign was **active for ~8 months** (first version shipped October 31, 2025; latest malicious version shipped June 10, 2026) and accumulated **~70,000 installs** across IntelliJ IDEA, PyCharm, WebStorm, GoLand, CLion, Android Studio, and other JetBrains-based IDEs. JetBrains has since purged all 15 plugins, banned the 7 publisher accounts, and triggered remote kill-switches — but anyone who installed one of them before June 17, 2026 should treat their AI provider keys as compromised.

### Why This Matters in a Frontend Skill

This is **a direct supply-chain attack on AI coding tools**, the same threat class as Mastra (June 17), TanStack (May 11), and node-ipc (May 15). It is in scope here because the skill documents AI-assisted workflows (`AGENTS.md` in `create-next-app`, `next-browser`, Vercel `eve`, Chat SDK, Vercel Agent), and **every one of those workflows ends with an AI provider API key in a developer environment**. The JetBrains plugin exploit demonstrates that the IDE itself is now an attack surface, not just `node_modules`.

### How The Theft Works

All 15 plugins share the same disguised codebase, renamed and repackaged for each listing. Every plugin offers a feature that **legitimately requires** the user to paste an API key into a settings panel (DeepSeek chat, commit-message generation, code review, bug finding, unit tests). The plugins actually deliver the advertised feature — which is what kept them functional and undetected for 8 months. The key exfiltration is a single side-channel call: as soon as the user saves the settings panel, the API key is POSTed (encrypted) to an attacker-controlled C2 server, then the settings panel returns success and the user never knows the key left their machine.

### Indicators of Compromise

If any of the following plugins were installed in any JetBrains IDE before June 17, 2026, treat **all API keys configured in that IDE** (OpenAI, Anthropic, DeepSeek, SiliconFlow, Google AI, plus any other provider keys present in env vars or `~/.config/`) as compromised:

| Plugin | Plugin ID | Downloads | First Seen |
|---|---|---|---|
| DeepSeek Junit Test | `org.sm.yms.toolkit` | 1,121 | 2025-10-31 |
| DeepSeek Git Commit | `com.json.simple.kit` | 1,894 | 2025-11-01 |
| DeepSeek FindBugs | `org.bug.find.tools` | 1,485 | 2025-11-09 |
| DeepSeek AI Chat | `org.translate.ai.simple` | 1,317 | 2025-11-23 |
| DeepSeek Dev AI | `com.yy.test.ai.kit` | 740 | 2025-11-30 |
| DeepSeek AI Coding | `com.dev.ai.toolkit` | 450 | 2025-12-06 |
| AI FindBugs | `com.json.view.simple` | 623 | 2025-12-14 |
| AI Git Commitor | `com.my.git.ai.kit` | 301 | 2026-01-10 |
| AI Coder Review | `org.check.ai.ds` | 735 | 2026-01-11 |
| DeepSeek Coder AI | `com.review.tool.code` | 3,498 | 2026-01-15 |
| AI Coder Assistant | `org.code.assist.dev.tool` | 319 | 2026-02-01 |
| DeepSeek Code Review | `com.coder.ai.dpt` | 278 | 2026-04-18 |
| **CodeGPT AI Assistant** | `com.my.code.tools` | **25,571** | 2026-06-09 |
| **DeepSeek AI Assist** | `ord.cp.code.ai.kit` | **27,727** | 2026-06-10 |
| Coding Simple Tool | `com.dp.git.ai.tool` | 3,931 | (no online versions) |

Publisher accounts (all banned): **CodePilot (mycode), StackSmith (misshewei), CodeCrafter (keteme), CodeWeaver (simpledev), JetCode (skyblue), DailyCode (dialycode), ZenCoder** (UUID).

The two highest-download plugins — `CodeGPT AI Assistant` (25,571) and `DeepSeek AI Assist` (27,727) — were the last two published before disclosure. The campaign was still active at disclosure time.

### Why Manual Review Failed

JetBrains Marketplace does run a [manual code review](https://plugins.jetbrains.com/docs/marketplace/jetbrains-marketplace-approval-guidelines.html#approval-process) on every plugin before publication. This attack evaded review because:

1. The plugin **delivers the advertised feature** — chat, commit messages, code review, etc. all work.
2. The exfiltration code is **a single small function call** inside an otherwise legitimate codebase — easy to miss in a 5,000-line plugin review.
3. The C2 server looks like a normal analytics endpoint (HTTPS POST, small JSON payload, user-agent strings matching the IDE).
4. **8 months of dormancy between updates** for some plugins lowered reviewer suspicion ("looks like an established, low-activity plugin").

The JetBrains team has acknowledged this and is updating their review process to look for outbound network calls inside plugin configuration handlers specifically.

### Five Things To Do Today

1. **Audit installed JetBrains plugins** — `Settings → Plugins → Installed` in any JetBrains IDE (IntelliJ, PyCharm, WebStorm, GoLand, CLion, Rider, Android Studio, RubyMine, PhpStorm, DataGrip). Look for any of the 15 plugin IDs above or any AI-themed plugin from an unfamiliar publisher.
2. **Rotate every AI provider API key** that was configured in any JetBrains IDE before June 17, 2026 — OpenAI, Anthropic, Google AI, DeepSeek, SiliconFlow, and any custom provider. The exfiltration was silent; absence of evidence is not evidence of absence.
3. **Check API provider billing dashboards** for unexpected usage spikes. Stolen keys are reportedly **resold as pay-per-use LLM proxy credits**, so the attacker monetizes by burning your quota. High `gpt-4o` or `claude-3-7-sonnet` spend with unfamiliar usage patterns is a strong signal.
4. **Revoke and reissue keys with usage limits.** OpenAI, Anthropic, and Google AI all support per-key hard spend caps (`$50/month` default for OpenAI, configurable in Anthropic Console, hard quotas in Google AI Studio). Set the cap **below** your expected normal usage so a stolen key burns out before it costs you a meaningful amount.
5. **Prefer vendor-verified AI plugins only** going forward. JetBrains' own AI Assistant, GitHub Copilot (now a [native JetBrains partner integration](https://blog.jetbrains.com/ai/2026/06/)), Continue.dev (open source, auditable on GitHub), Cursor (separate IDE, separate marketplace), and Tabnine are all vendor-verified or self-hostable. For any third-party AI plugin, require (a) the publisher to be a company you can name, (b) the source to be public on GitHub, and (c) the network calls to be documented in a privacy policy.

### The Pattern Is Now IDE → Registry, Not Just Registry → npm

TanStack (May 11), node-ipc (May 15), Mini Shai-Hulud (June 1), Mastra (June 17), and JetBrains Marketplace (June 16) all fit the same broad pattern: **a trusted marketplace or scope is compromised, malicious versions are published, and the malicious code is buried inside working features.** The JetBrains attack is the first one in this cluster to target the **IDE runtime itself** rather than the project's `node_modules`. Treat IDE plugins with the same vetting rigor you would apply to npm packages — verify the publisher, check the source, audit network calls. See also `setup.md` → `AGENTS.md` for how this skill recommends documenting trusted AI tooling in the project itself.

**Sources:**
- [Aikido Security — Multiple JetBrains IDE plugins caught stealing AI keys (June 16, 2026 — full IOC list, publisher accounts, exfiltration flow)](https://www.aikido.dev/blog/multiple-jetbrains-ide-plugins-caught-stealing-ai-keys)
- [JetBrains Marketplace Ecosystem Security Update — Addressing Malicious Third-Party AI Plugins (official disclosure, June 17, 2026)](https://blog.jetbrains.com/platform/2026/06/marketplace-ecosystem-security-update-malicious-ai-plugins)
- [Threat-Modeling.com — JetBrains Marketplace Malicious Plugins Stealing AI API Keys from Developer Environments (June 17, 2026)](https://threat-modeling.com/jetbrains-marketplace-malicious-plugins-ai-key-theft-june-2026/)
- [OffSeq Threat Radar — 15 JetBrains Marketplace plugins quietly stealing developers' AI API keys (~70,000 installs)](https://radar.offseq.com/threat/15-jetbrains-marketplace-plugins-were-quietly-stea-8bacd71f)
- [SANS Stormcast Thursday June 18, 2026 — JetBrains Plugins segment (ISC overview)](https://isc.sans.edu/podcastdetail/9978)
- [JetBrains Marketplace Approval Guidelines (the manual review process that failed)](https://plugins.jetbrains.com/docs/marketplace/jetbrains-marketplace-approval-guidelines.html#approval-process)

## Server Actions Are Public POST Endpoints (2026 Architectural Principle) (2026 Architectural Principle)

**This is the most important Server Actions security pattern in 2026, and it applies to every `'use server'` function in the codebase.** Multiple authoritative 2026 sources (Next.js official docs, Vercel, Makerkit, Authgear, BuildMVPFast) converge on the same rule: **authenticate and authorize *inside* every Server Action body. Never rely on page-level checks, never rely on middleware as your authorization layer.**

From the [Next.js data security docs](https://nextjs.org/docs/app/guides/data-security):
> By default, when a Server Action is created and exported, it is reachable via a direct POST request, not just through your application's UI. This means, even if a Server Action or utility function is not imported elsewhere in your code, it can still be called externally. […] However, you should still treat Server Actions as reachable via direct POST requests and verify authentication and authorization inside each one.

And from the [Next.js authentication guide](https://nextjs.org/docs/app/guides/authentication):
> A common pattern in SPAs is to `return null` in a layout or a top-level component if a user is not authorized. This pattern is **not recommended** since Next.js applications have multiple entry points, which will not prevent nested route segments and Server Actions from being accessed. Ensure that any Server Actions called from these components also perform their own authorization checks, as client-side UI restrictions alone are not sufficient for security.

Vercel's own [postmortem on CVE-2025-29927](https://vercel.com/blog/postmortem-on-next-js-middleware-bypass) reaches the same conclusion: **middleware is fine for coarse redirects. It is not your authorization layer.**

### The Two-Lock Pattern (auth + authorization)

Every Server Action needs **two** independent checks, not one:

1. **Authentication** — `const session = await auth()` (or `getSession()`). Confirms *who* is calling.
2. **Authorization** — confirms *this* user is allowed to act on *this specific* resource. Skip this and you have an **IDOR** (Insecure Direct Object Reference): an attacker passes `postId=42` they don't own, and the action happily acts on it.

```ts
// app/actions/deletePost.ts
'use server'

import { z } from 'zod'
import { auth } from '@/lib/auth'
import { db } from '@/lib/db'

const DeletePostSchema = z.object({
  postId: z.string().cuid2(),
})

// ✅ Authenticate AND authorize inside the action body
export async function deletePost(input: unknown) {
  const session = await auth()
  if (!session?.user) throw new Error('Unauthorized')

  const { postId } = DeletePostSchema.parse(input)

  // ✅ Authorization via ownership — pass the userId into the WHERE clause
  // (not a separate findUnique + check; that's racy and leaks existence)
  const { count } = await db.post.deleteMany({
    where: { id: postId, authorId: session.user.id },
  })
  if (count === 0) {
    // Don't leak existence: 404 vs 403 are different signals
    throw new Error('Not found')
  }

  revalidatePath('/posts')
  return { success: true }
}
```

```ts
// ❌ WRONG — only authenticates; IDOR. Any logged-in user can delete any post.
export async function deletePost(input: unknown) {
  const session = await auth()
  if (!session?.user) throw new Error('Unauthorized')
  const { postId } = DeletePostSchema.parse(input)
  await db.post.delete({ where: { id: postId } })  // ← no ownership check
}
```

```ts
// ❌ WRONG — page-level auth check, but the action is a separate entry point
// app/posts/[id]/page.tsx
export default async function PostPage({ params }) {
  const session = await auth()
  if (!session) redirect('/login')
  const post = await db.post.findUnique({ where: { id: (await params).id } })
  if (post.authorId !== session.user.id) notFound()
  return <Post post={post} />
}
// Attacker skips the page entirely and POSTs to the Server Action ID — page check never runs.
```

### Pre-Ship Checklist for Every Server Action

1. **`auth()` + ownership check inside the action body** — every time, no exceptions
2. **Validate every input with a Zod schema** — parse into an allowlist of fields
3. **Return DTOs, not raw rows** — `toJSON()` or a mapper function; never `return db.user.findUnique(...)`
4. **Don't treat middleware as your authorization layer** — coarse gating only (redirects, A/B buckets)
5. **Rate-limit sensitive actions** — auth, OTP, password reset, anything expensive. Use Vercel's `unstable_after` or an Upstash Redis token bucket
6. **Set `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`** — required if you run more than one instance (otherwise the action IDs are encrypted with a per-instance ephemeral key, breaking the fleet)
7. **Run production mode in production** — `next dev` is debug-friendly but exposes source maps and stack traces
8. **Keep Next.js patched** — Server Actions share the React Server Components protocol that was exploited by CVE-2025-66478 (React2Shell, CVSS 10) and CVE-2025-55184/67779/55183. The 16.2.6 patch (May 2026) bundles 13 framework-level fixes

**Sources:**
- [Next.js docs — Data fetching: Server Actions security](https://nextjs.org/docs/app/guides/data-security)
- [Next.js docs — Authentication: Server Actions](https://nextjs.org/docs/app/guides/authentication)
- [Vercel — Postmortem on the Next.js middleware bypass (CVE-2025-29927)](https://vercel.com/blog/postmortem-on-next-js-middleware-bypass)
- [Authgear — Next.js Security Best Practices 2026](https://www.authgear.com/post/nextjs-security-best-practices/)
- [Makerkit — Next.js Server Actions Security: 5 Vulnerabilities (Next 16.2.6)](https://makerkit.dev/blog/tutorials/secure-nextjs-server-actions)
- [BuildMVPFast — Server Actions Security: Real Vulnerabilities (June 18, 2026)](https://www.buildmvpfast.com/blog/nextjs-server-actions-security-vulnerabilities-2026)
- [OWASP — Insecure Direct Object Reference Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html)


## XSS (Cross-Site Scripting)

### The Threat

Attacker injects malicious scripts via unsanitized user input. In React, this is rare because React escapes values by default — but there are exceptions.

### React's Built-in Protection

```tsx
// React escapes this automatically — safe by default
const userInput = '<script>alert("xss")</script>'
return <div>{userInput}</div>  // Renders as text, not script
```

### When XSS Happens in React

**1. Using `dangerouslySetInnerHTML`:**

```tsx
// ❌ Dangerous — bypasses React's escaping
<div dangerouslySetInnerHTML={{ __html: userContent }} />

// ✅ Safe alternatives
<div>{userContent}</div>  // React escapes it

// Or sanitize server-side before rendering
import DOMPurify from 'isomorphic-dompurify'

const sanitized = DOMPurify.sanitize(userContent)
return <div dangerouslySetInnerHTML={{ __html: sanitized }} />
```

**2. URLs in `href` or `src`:**

```tsx
// ❌ If userInput = "javascript:alert('xss')"
<a href={userInput}>Click</a>
<img src={userInput} />

// ✅ Always validate URLs
const isSafeUrl = (url: string) =>
  url.startsWith('http://') || url.startsWith('https://')

if (!isSafeUrl(linkUrl)) return '#'
```

**3. Dynamic attribute injection:**

```tsx
// ❌ Never interpolate into attributes
<div data-value={userInput}>  // Can be exploited with JS payloads

// ✅ Use controlled attributes
<div data-id={sanitizedId}>
```

### Content Security Policy (CSP)

Add CSP headers to `next.config.ts`:

```ts
const nextConfig: NextConfig = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'Content-Security-Policy',
            value: [
              "default-src 'self'",
              "script-src 'self' 'unsafe-inline' 'unsafe-eval'", // disable eval in prod
              "style-src 'self' 'unsafe-inline'",  // shadcn needs inline styles
              "img-src 'self' data: https:",
              "font-src 'self'",
              "connect-src 'self' https://api.example.com",
              "frame-ancestors 'none'",
              "base-uri 'self'",
              "form-action 'self'",
            ].join('; '),
          },
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          {
            key: 'Permissions-Policy',
            value: 'camera=(), microphone=(), geolocation=()',
          },
        ],
      },
    ]
  },
}
```

### Next.js CSP Nonce Considerations

When using CSP with Next.js App Router, **do not use untrusted user input in CSP nonce generation**. The 16.2.6 patch fixes XSS via CSP nonces — ensure your nonce implementation reads from a server-generated seed, not from client-supplied values:

```tsx
// ❌ Never derive nonce from client-supplied input
const nonce = request.headers.get('x-nonce') // ← attacker-controlled

// ✅ Generate nonce server-side and pass via crypto.getRandomValues
import { randomBytes } from 'crypto'

function generateCspNonce(): string {
  return randomBytes(16).toString('base64')
}
```

### beforeInteractive Script XSS

Avoid passing unsanitized user input to `beforeInteractive` scripts:

```tsx
// ❌ Dangerous — user input in beforeInteractive script
<Script
  src={`/analytics.js?campaign=${userProvidedParam}`}
  strategy="beforeInteractive"
/>

// ✅ Validate or hardcode all beforeInteractive script sources
<Script
  src="/analytics.js"
  strategy="beforeInteractive"
/>
```

### `ReactDOM.preloadModule()` Silently Dropped `nonce` (June 29, 2026 — fixed in React canary `e2731312-20260630`, still in current canary `3508aee6-20260702`)

React PR [#36851](https://github.com/facebook/react/pull/36851) (Udit Dewan, merged June 29, 2026) fixed a silent CSP violation in `ReactDOM.preloadModule()`. The public API accepted a `nonce` option in its type (`PreloadModuleOptions`), but the implementation only forwarded `as`, `crossOrigin`, and `integrity` to the host dispatcher — `nonce` was silently dropped. The emitted `<link rel="modulepreload">` carried no nonce attribute, so under a strict `script-src 'nonce-...'` CSP, the browser blocked the preload and the hint did nothing.

This is the React counterpart to the [16.2.6 XSS via CSP nonces fix](#nextjs-csp-nonce-considerations): same theme, different layer. `ReactDOM.preload()` and `ReactDOM.preinitModule()` already forwarded `nonce` correctly — `preloadModule()` was the inconsistent one. The type was already correct (`PreloadModuleOptions.nonce` + `PreloadModuleImplOptions.nonce` both declared); only the runtime forwarding was missing. The fix is one line in `ReactDOMFloat.js` plus a client/Fizz/Flight spread that was already correct.

**Practical impact:**

- **If you call `ReactDOM.preloadModule(href, { nonce, as, ... })` from client code with strict CSP (`script-src 'nonce-...'`)**, your `modulepreload` hints were silently blocked before this fix. After upgrading to React canary `19.3.0-canary-e2731312-20260630`+ (current canary is `19.3.0-canary-3508aee6-20260702`, but the fix shipped in `e2731312-20260630`), the nonce flows through and the hint loads.
- **If you use Next.js's `<link rel="modulepreload">` infrastructure indirectly** (Next.js does this for bootstrap chunks), the fix is irrelevant to your code — Next.js sets nonces via the same mechanism, and React's internal bootstrap already emitted them correctly. The bug only affected the public `preloadModule()` API.
- **If you don't use strict CSP with nonces**, you don't notice this bug — `modulepreload` links work fine without a nonce when CSP isn't enforcing one.
- **If you only use `ReactDOM.preload()` and `ReactDOM.preinitModule()`**, you don't notice this bug — those APIs already forwarded `nonce` correctly before the fix.

**Audit:**

```bash
# Find any calls to ReactDOM.preloadModule with a nonce option
grep -rn "ReactDOM\.preloadModule\|reactDom\.preloadModule" --include="*.tsx" --include="*.ts" --include="*.jsx" --include="*.js" . 2>/dev/null | grep -i "nonce"

# Check your React version
npm ls react react-dom
```

If the second query shows `react` < `19.3.0-canary-e2731312-20260630` and the first query has any hits, upgrade to the latest React canary (current: `19.3.0-canary-3508aee6-20260702`, or stable once 19.3 ships) to get the fix. No code changes needed — just upgrade.

### False-Positive Hydration Mismatch on `nonce` Attributes in Dev (July 16, 2026 — fixed in React canary `172742b4-20260716`, [PR #37030](https://github.com/facebook/react/pull/37030) by MaxwellCohen, merged 2026-07-16T16:54:10Z)

Every Next.js App Router page that uses CSP with nonce-based script tags — i.e. the standard `script-src 'nonce-...'` pattern — was getting a **spurious red-box hydration mismatch error in dev on every page load**, even when the server-rendered HTML and the client render actually matched. The cause was a React Fiber bug in how it compared `nonce` attributes during hydration.

**Why it was happening:**

The [HTML spec](https://html.spec.whatwg.org/multipage/urls-and-fetching.html#cryptographicnonce) deliberately hides the cryptographic nonce from any read API other than the `HTMLOrSVGElement.nonce` IDL property, in order to defeat CSS attribute selectors as a side channel:

> "Elements that have a nonce content attribute ensure that the cryptographic nonce is only exposed to script (and not to side-channels like CSS attribute selectors) by taking the value from the content attribute, moving it into an internal slot named [[CryptographicNonce]], exposing it to script via the HTMLOrSVGElement interface mixin, and setting the content attribute to the empty string. Unless otherwise specified, the slot's value is the empty string."

React's hydration mismatch check used `element.getAttribute('nonce')` to compare the server-rendered nonce to the client-rendered one. In a real browser, that getter returns `""` (the empty string) because the spec hid it — so React always saw the client nonce as `""` and complained that `""` didn't match the server nonce. JSDOM (used in tests + in some dev paths) did **not** implement this hiding, so tests were green while real browsers saw the error.

**Practical impact:**

- **If you use Next.js App Router with strict CSP (`Content-Security-Policy: script-src 'nonce-...'`)** — the canonical pattern for nonce-based CSP — every dev-mode page load triggered a red-box hydration mismatch. Prod hydration completed fine because the IDs/structure matched, but dev noise was constant and obscured real hydration errors.
- **If you use `<Script strategy="beforeInteractive">` from `next/script`** with a nonce, the same false positive fired on every navigation.
- **If you use Pages Router with nonces** — same root cause, same error.
- **If you don't use CSP nonces** — you didn't see this (your nonce attribute was absent on both sides, so the empty-string comparison was a non-issue).
- **If you use loose CSP (`script-src 'self' 'unsafe-inline' ...`)** — also didn't see this.

**The fix** (PR #37030) makes the test environment hide the nonce the way real browsers do, by adding a setter guard that the JSDOM element respects but the production browser path already respected. The change is dev/test-only: production browsers already implemented the spec, so the fix just brings JSDOM into alignment. (The PR author did NOT enable the nonce-hide for all tests because doing so breaks `ReactDOMFizzServer-test.js` server-side tests; the hide is enabled only where the comparison would otherwise fail in the same way a real browser would.)

**Audit:**

```bash
# Find code that uses nonces with strict CSP
grep -rn "nonce=\|nonce: \|nonce={\|getNonce\|headers.*nonce" \
  --include="*.tsx" --include="*.ts" --include="*.jsx" --include="*.js" \
  --include="*.ts" middleware.ts next.config.* app/ 2>/dev/null

# Check your React version
npm ls react react-dom
```

**Bundled into Next.js 16.3.0-canary.90** (2026-07-19T23:34:16Z, [PR #95901](https://github.com/vercel/next.js/pull/95901) — vendor React bump from `7023f501-20260714` to `172742b4-20260716`) — if you're running `next@canary@90`+, the React `172742b4-20260716` that includes PR #37030 is what `next@canary` ships INSIDE its `react.production.js` bundle at `packages/next/src/compiled/react/`. So canary.90+ users don't need a separate `react@canary@172742b4-20260716` install — the fix is automatic on canary. `npm view next dist-tags.canary` should show `16.3.0-canary.90` or later. For teams on stable `next@16.2.x` (which still bundles React 19.2.7), the fix requires either (a) a separate `npm install react@canary@172742b4-20260716` pin (Next.js will use the canary React for SSR/RSC even with the older `next` installed, since the version resolution is the user's package.json's win), (b) the July 20, 2026 Next.js Security Release (may include the React bump if Vercel decides to ship security-adjacent React fixes per their May 2026 precedent), or (c) React 19.3 stable (date TBA — React's release cadence has been ~6 months since 19.2 in Oct 2025).

If the first query has any hits and the second query shows `react` < `19.3.0-canary-172742b4-20260716`, upgrade React to `19.3.0-canary-172742b4-20260716` or later to silence the false-positive red-box. **No code change is needed** — the hydration logic itself is correct, it's just that the dev-mode check needed to learn the spec's nonce-hiding behavior.

**Source:** [React PR #37030](https://github.com/facebook/react/pull/37030) (MaxwellCohen, based on #33806, requested by @eps1lon). Related: the older `ReactDOM.preloadModule()` nonce-drop bug (above) and the Next.js 16.2.6 XSS via CSP nonces fix (below).

### OTel Proxy Tracer Silent Corruption in Cache Components Prerenders (July 1, 2026 — fixed in Next.js 16.3.0-canary.73, [PR #95317](https://github.com/vercel/next.js/pull/95317) by Jiwon Choi, merged 2026-07-01T16:14:49Z)

A silent dev-only corruption fired in any Cache Components (`cacheComponents: true`) route that also configured OpenTelemetry tracing: the prerender pipeline threw `Error: encountered unstable value in proxy tracer during prerender` mid-prerender, with no recoverable state, no useful stack frame (the tracer's proxy is opaque), and no log entry that pointed at the OTel setup. The build would fail; the cause was the OTel SDK that was installed in `instrumentation.ts` (or via the `OTEL_*` env vars) before the Next.js `workUnitStore` was fully initialized. Closes issue [#94753](https://github.com/vercel/next.js/issues/94753).

**Why it was silent:**

1. The error came from inside the `proxy` proxy object's `get` trap on the OTel tracer — the stack frame at the throw site was inside the SDK, not in your code.
2. The Next.js dev overlay showed the throw but the message `encountered unstable value in proxy tracer during prerender` gave no hint that OTel was the cause; the error looked like a generic Cache Components violation.
3. There was no OTel-side breadcrumb — the SDK was just operating normally and the proxy's get() trap happened to be invoked during the prerender's tracking phase.
4. CI suites that didn't have OTel configured didn't reproduce; dev with OTel did.

**The fix** (PR #95317) is to **defer proxy-tracer creation until the OTel span is actually started**, guarded by `workUnitStore !== null`. The proxy-tracer creation is now lazy: it only happens on the first `tracer.startSpan()` call within a real `workUnitStore` (either a build worker or a request-time work unit), not during the prerender's setup phase. The 5-line change is in `packages/next/src/server/dev/next-prerender-proxy.ts`; the `proxy` is wrapped in a function that's invoked on first access rather than eagerly constructed.

**Practical impact:**

- **If you use Next.js + OpenTelemetry + `cacheComponents: true`:** upgrade to 16.3.0-canary.73 (npm `next@canary` is now pinned to it) to get the fix. No code changes needed — the OTel SDK's tracer is still installed the same way.
- **If you hit this error and were working around it by removing OTel from `instrumentation.ts`:** you can re-enable OTel. The most common workaround before the fix was to gate the OTel SDK import behind `if (process.env.NEXT_RUNTIME === 'nodejs' && process.env.NODE_ENV === 'production')`, which left dev without traces — no longer necessary.
- **If you use OTel but NOT Cache Components:** the bug never fired. The proxy-tracer's get() trap was only invoked during the prerender's tracking phase, which is a Cache Components concept.

**Audit:**

```bash
# 1. Confirm you have OTel configured (any of the below count)
grep -l "instrumentation\.ts\|@opentelemetry" instrumentation.ts node -r --include="*.ts" 2>/dev/null

# 2. Check if cacheComponents is on
grep -n "cacheComponents" next.config.ts next.config.js next.config.mjs 2>/dev/null

# 3. Check your Next.js version
npm ls next
# If `next` < 16.3.0-canary.73 AND you hit OTel-related prerender errors, upgrade.

# 4. Pin to the canary that has the fix
npm install next@canary
npx next --version  # should show 16.3.0-canary.73
```

**Why the OTel setup vs `workUnitStore` race condition was hard to spot:** OTel's tracer uses a JavaScript `Proxy` to track span access; the proxy was eagerly created at SDK import time (in `instrumentation.ts`), which runs before the per-request work unit is set up. When the prerender's tracking phase read the tracer's properties to instrument the render, the proxy's `get` trap tried to read the tracer's underlying `activeSpan` and found none (no span started yet), which the cache-components validation interpreted as an "unstable value" — but the value being unstable was a Proxy, not a real value. The error message intentionally doesn't say "OTel" or "tracer" because the validation layer doesn't know about either.

**Source:** [PR #95317 — `Fix early OTel proxy tracers in Cache Components prerenders`](https://github.com/vercel/next.js/pull/95317) · [Commit `fa71595a`](https://github.com/vercel/next.js/commit/fa71595a) · Closes issue [#94753](https://github.com/vercel/next.js/issues/94753) · Files: `packages/next/src/server/dev/next-prerender-proxy.ts` (+5/-3, 1 file)

## Middleware Security

### The Middleware Bypass Fixes (16.2.6)

Next.js 16.2.6 patches multiple middleware/proxy bypass vulnerabilities. **Do not rely on middleware alone for security-critical decisions** — always validate server-side too.

Key mitigations to ensure are in place:

```ts
// middleware.ts — validate EVERYTHING server-side, not just in middleware
export async function middleware(request: NextRequest) {
  // ❌ Incomplete — path check only
  // if (request.nextUrl.pathname.startsWith('/admin')) { ... }

  // ✅ Verify auth properly in middleware AND in route handlers
  const session = await getSession(request)

  if (request.nextUrl.pathname.startsWith('/admin')) {
    if (!session?.user?.role === 'admin') {
      return NextResponse.redirect(new URL('/login', request.url))
    }
  }

  return NextResponse.next()
}
```

### WebSocket Upgrade SSRF

If your app uses WebSocket upgrades, **validate all URLs server-side** before upgrading:

```tsx
// ❌ Dangerous — attacker can supply internal URLs
const wsUrl = request.nextUrl.searchParams.get('ws')

// ✅ Always validate WebSocket URLs
const isAllowedWsUrl = (url: string) => {
  try {
    const parsed = new URL(url)
    return parsed.protocol === 'wss:' &&
           !parsed.hostname.includes('localhost') &&
           !parsed.hostname.includes('127.0.0.1') &&
           !parsed.hostname.startsWith('192.168.') &&
           !parsed.hostname.startsWith('10.')
  } catch {
    return false
  }
}
```

## CSRF (Cross-Site Request Forgery)

### The Threat

Attacker tricks a logged-in user into submitting a form/request to your site.

### NextAuth CSRF Protection

NextAuth.js handles CSRF automatically via signed cookies and the `state` parameter in OAuth flows. No extra work needed for Server Actions.

### Custom Header Validation

```ts
// For API routes, validate Origin header
export async function POST(request: Request) {
  const origin = request.headers.get('origin')
  const host = request.headers.get('host')

  // In production, verify origin matches expected domain
  if (process.env.NODE_ENV === 'production') {
    const allowedOrigins = ['https://myapp.com', 'https://www.myapp.com']
    if (!allowedOrigins.includes(origin ?? '')) {
      return new Response('Forbidden', { status: 403 })
    }
  }

  // Process request...
}
```

### Double Submit Cookie Pattern (for forms without NextAuth)

```ts
import { cookies } from 'next/headers'
import { randomBytes } from 'crypto'

function generateCsrfToken(): string {
  return randomBytes(32).toString('hex')
}

export async function setCsrfToken() {
  const csrfToken = generateCsrfToken()
  const cookieStore = await cookies()
  cookieStore.set('csrf-token', csrfToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    path: '/',
  })
  return csrfToken
}

export async function validateCsrfToken(token: string): Promise<boolean> {
  const cookieStore = await cookies()
  const storedToken = cookieStore.get('csrf-token')?.value
  return storedToken === token
}
```

## SQL Injection

SQL injection is a backend concern, but frontend code that constructs queries matters:

```tsx
// ❌ Never interpolate user input into SQL
const userId = formData.get('userId')
await db.$queryRaw`SELECT * FROM users WHERE id = ${userId}`

// ✅ Prisma automatically escapes values — always use Prisma
const user = await db.user.findUnique({ where: { id: userId } })
```

## Input Validation (Zod)

**Validate ALL user input, client and server:**

```ts
// Server Action — always validate
export async function updateProfile(formData: FormData) {
  const parsed = z.object({
    email: z.string().email().max(255),
    bio: z.string().max(500).optional(),
    website: z.string().url().optional().or(z.literal('')),
  }).safeParse(Object.fromEntries(formData))

  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors }
  }

  // Now it's safe to use parsed.data
}
```

## Secure Cookies

```ts
// When setting cookies in API routes
const cookieStore = await cookies()

cookieStore.set('session', token, {
  httpOnly: true,     // Not accessible from JavaScript
  secure: process.env.NODE_ENV === 'production',
  sameSite: 'strict', // CSRF protection
  maxAge: 60 * 60 * 24 * 7, // 1 week
  path: '/',
})
```

## Password Handling

Never handle passwords directly in frontend code — use backend APIs or NextAuth's credential provider which handles hashing with bcrypt/bcryptjs.

```tsx
// Client — never hash client-side, always send to server
<form action={async (formData) => {
  'use server'
  // Hash on the server with bcrypt
  const hash = await bcrypt.hash(formData.get('password'), 12)
}}>
```

## Secrets Management

```bash
# .env.local — local only, never commit
DATABASE_URL="postgresql://..."
NEXTAUTH_SECRET="..."

# Environment variables in deployment
# Vercel: set in dashboard
# Docker: pass via -e or docker-compose environment:
#   environment:
#     - NEXTAUTH_SECRET=${NEXTAUTH_SECRET}
```

**Rule:** `NEXT_PUBLIC_` prefix = public client-exposed variable. Never put secrets with this prefix.

## Security Headers Summary

| Header | Value | Purpose |
|---|---|---|
| `Content-Security-Policy` | `script-src 'self'; ...` | Prevents XSS/injection |
| `X-Frame-Options` | `DENY` | Prevents clickjacking |
| `X-Content-Type-Options` | `nosniff` | Prevents MIME sniffing |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Controls referrer info |
| `Permissions-Policy` | `camera=(); microphone=(); geolocation=()` | Disables unused APIs |

## Common Mistakes

- **`dangerouslySetInnerHTML`** without sanitization — use DOMPurify
- **`NEXT_PUBLIC_` for secrets** — anything with that prefix is public
- **No `httpOnly` on session cookies** — allows XSS to steal sessions
- **Missing CSRF validation** — always validate Origin header in API routes
- **Not validating user input on the server** — client validation is UX only, not security
- **Storing passwords in plain text** — use bcrypt, always hash server-side
- **Relying on middleware alone for auth** — always re-validate in route handlers
- **Unvalidated user input in WebSocket upgrade URLs** — validate URLs before upgrading
- **Unvalidated user input in beforeInteractive scripts** — hardcode script sources
- **Deriving CSP nonces from client-supplied values** — generate server-side only
- **Caret-ranged deps for high-value packages** — `^1.11.21` (as used in the Mastra attack) auto-pulls in any new minor/patch. Pin exact versions for security-critical deps, or use `npm ci` everywhere
- **Dormant npm maintainer accounts in your org** — TanStack (May 11), node-ipc (May 15), Mini Shai-Hulud (June 1), Mastra (June 17) all exploited stale contributor access. Audit npm scope ownership quarterly and remove former maintainers immediately
- **Trusting `npm audit` to catch active supply chain attacks** — the Mastra `easy-day-js` typosquat copied `dayjs`'s metadata perfectly and `npm audit` showed nothing. Use Socket.dev or Snyk for behavioral analysis, not just CVE matching
- **Vitest Browser Mode on a network-exposed host (--browser.api.host=0.0.0.0)** — exposes API token, project root, and CDP RPC; CVSS 9.8 RCE on pre-4.1.8 versions. Keep Browser Mode on localhost and upgrade to vitest ≥ 4.1.8
- **Relying on page-level auth checks for Server Actions** — Server Actions are separate entry points reachable via direct POST. A page redirect does not protect them. Always run `auth()` + ownership check inside the action body, never trust a parent's auth state
- **Authentication without authorization in Server Actions (IDOR)** — `auth()` only checks *who* the caller is. The action also needs to verify *this user is allowed to act on this specific resource* (ownership, team membership, role, or explicit permission). Without it, any logged-in user can pass any `id` they want
- **Using `findUnique` + JS check for ownership in Server Actions** — races (TOCTOU between the read and the write) and leaks the row's existence to unauthorized actors. Use `deleteMany` / `updateMany` with the ownership filter in the `where` clause, then check the returned `count`. A 0 count means "not found" or "not yours" — return the same error in both cases
- **Server Action that returns raw DB rows** — leaks `passwordHash`, `internalNote`, `stripeCustomerId`, soft-delete flags, `updatedAt`, etc. Return a DTO (Data Transfer Object) or use `toJSON()` on the model. This is a top-3 OWASP API risk (Broken Object Property Level Authorization)
- **Missing `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` on multi-instance deployments** — Server Action IDs are encrypted with a per-instance ephemeral key by default. Two instances can't share action IDs. Generate a stable secret (`openssl rand -base64 32`) and set it in the deploy environment
- **Running pre-June-17-2026 Node.js in production** — Node 22.x (pre-22.23.0), Node 24.x (pre-24.17.0), and Node 26.x (pre-26.3.1) are vulnerable to **12 CVEs** including **2 HIGH**: CVE-2026-48933 (WebCrypto AES DoS) and CVE-2026-48618 (TLS wildcard-cert bypass via Unicode normalization). Bump the base image tag or `engines.node` now
- **WebCrypto without input size cap** — `crypto.subtle.encrypt()` crashes the Node process when the input length is a multiple of 2 GiB (CVE-2026-48933). Even on a patched Node, defense in depth: cap plaintext size before encryption so untrusted input can never approach 2 GiB
- **Outbound HTTPS calls to wildcard-cert hosts without hostname validation** — CVE-2026-48618 lets Unicode dot separators (U+3002, U+FF0E, U+FF61) pass `tls.checkServerIdentity()` against `*.example.com` while resolving to a different host. If you build a URL from user input, validate the hostname is ASCII-clean and reject Unicode dots at the boundary
- **Calling `fetch` with a hostname derived from user input** — even on a patched Node, the canonical TLS-bypass / SSRF pattern is `const host = new URL(req.body.url).hostname; await fetch(\`https://${host}/api\`)`; an attacker can supply `evil。example.com` (with U+3002) and slip past any `*.example.com` check. Use a URL allowlist, not string interpolation
- **Trusting third-party JetBrains Marketplace AI plugins with production API keys (June 16, 2026 disclosure)** — at least 15 plugins published under 7 publisher accounts (CodePilot, StackSmith, CodeCrafter, CodeWeaver, JetCode, DailyCode, ZenCoder) exfiltrated OpenAI / Anthropic / DeepSeek / SiliconFlow / Google AI keys to attacker C2 for 8 months (~70,000 installs) before JetBrains banned them. Treat any AI provider key that was ever pasted into a JetBrains IDE settings panel before June 17, 2026 as compromised. Rotate the key, check the billing dashboard for anomalous spend, and prefer vendor-verified AI plugins only (JetBrains AI Assistant, GitHub Copilot native integration, Continue.dev, Tabnine)
- **Missing the Vercel Next.js monthly security release** (program launched July 13, 2026 — see [Vercel Next.js Security Release Program](#vercel-next-js-security-release-program-july-13-2026) above) — Next.js now ships a scheduled, pre-announced security release on the **20th of each month** (first one: July 20, 2026, 4 high + 5 medium CVEs). Calendar a recurring reminder, watch the [Next.js blog RSS](https://nextjs.org/blog) / [vercel/next.js releases feed](https://github.com/vercel/next.js/releases), and configure Renovate / Dependabot to auto-open PRs for `next` and `eslint-config-next`. Same-day upgrade is the expectation; the monthly cadence means each patch is small (CVE-only, no API change) so a one-day turnaround is realistic
- **Trusting the Node.js June 18 patch to cover all `undici` CVEs** — it only covers the version bundled with Node (`process.versions.undici`). If any package in your dependency tree pulls in its own `undici` (AWS SDK, OpenAI SDK, Anthropic SDK, Supabase, Prisma, NextAuth, WebSocket clients, etc.), `npm ls undici` may show an older version. Use `overrides` / `pnpm.overrides` to force a patched `undici` (7.28.0 / 8.5.0) across the whole tree
- **Mismatched `typescript`/`next` versions on `next@15.5.21` or earlier** — since `next@15.5.22` (PR [#96110](https://github.com/vercel/next.js/pull/96110), shipped 2026-07-25T20:45:27Z), the 15.5 line fails fast with an actionable `CompileError` if `typescript@>=7.0.0` is installed (the legacy `typescript.js` Compiler API is no longer shipped by TS 7's Go-native compiler). Pre-15.5.22, the same setup produced a silent `SIGSEGV`/`SIGABRT` inside `verify-typescript-setup` that looked like an unrelated crash. The actionable error is a defensive improvement: **always pin `typescript` explicitly in `package.json`** rather than relying on `npm install -D typescript@latest` to do the right thing — `typescript@latest` is now `7.0.2`, which only works on `next@16.2.12+` with `experimental.useTypeScriptCli: true`. For 15.5.x: `typescript: "^6.0.0"`. See `typescript.md` → "Next.js 16.2.12 / 15.5.22 — TypeScript 7 Compatibility Matrix" and `setup.md` → "TypeScript 7 Integration".

- **Missing the Node.js July 29, 2026 security release (patch published, now 9 CVEs / 3 HIGH)** — the release was postponed twice from Mon Jul 27 → Tue Jul 28 ("additional testing and validation") → **Wed Jul 29** ("infrastructure issues"). Patches are **Node 22.23.2**, **Node 24.18.1**, **Node 26.5.1** (the local machine running this skill is on Node 24.18.0 → bump to 24.18.1). The 9 CVEs span 3 HIGH (HTTP/2 retained-headers memory-exhaustion CVE-2026-56846, HTTP/2 re-entrant heap-use-after-free CVE-2026-56848, Permission Model path-over-grant CVE-2026-58043), 5 MEDIUM (mTLS identity reuse CVE-2026-56850, TLS hostname-verification skip CVE-2026-58040, SQLite iterator write-replay CVE-2026-58041, DNS aborted-on-many-A-records CVE-2026-58042, zlib spoofed-TypedArray crash CVE-2026-58045), and 3 LOW (Permission Model trace_events CVE-2026-56847, Permission Model process.report CVE-2026-58039, HTTP parser header-truncation request smuggling CVE-2026-58044) + `undici` 8.9.0 / 7.29.0 / 6.28.0 + `llhttp` 9.4.3. The HIGH ceiling that was pre-announced holds: CVE-2026-56848 in particular is a **remotely-triggerable heap-use-after-free** in `node:http/2` (the kind of memory-safety bug that frequently gets turned into RCE by an experienced attacker); CVE-2026-58043 is the **first Permission Model HIGH of 2026** — a security-boundary bypass, not a DoS, that affects every project running with `--permission`. Calendar a Wed morning reminder for future pre-announcements (account for the 2-day postpone buffer), and bump the base image tag + `engines.node` + `npm overrides`-range on `undici` the same day. See the full [Node.js July 29, 2026 Security Release](#nodejs-july-29-2026-security-release--9-cves--3-high-originally-july-27-delayed-twice) section for the 9-CVE table, post-patch verification recipe, and 9-step patch-day checklist.

**Sources:**
- [Next.js 16.2.6 security release](https://github.com/vercel/next.js/releases/tag/v16.2.6)
- [GHSA-8h8q-6873-q5fj: DoS with Server Components](https://github.com/vercel/next.js/security/advisories/GHSA-8h8q-6873-q5fj)
- [GHSA-267c-6grr-h53f: Middleware/Proxy bypass](https://github.com/vercel/next.js/security/advisories/GHSA-267c-6grr-h53f)
- [GHSA-c4j6-fc7j-m34r: SSRF via WebSocket upgrades](https://github.com/vercel/next.js/security/advisories/GHSA-c4j6-fc7j-m34r)
- [GHSA-ffhc-5mcf-pf4q: XSS via CSP nonces](https://github.com/vercel/next.js/security/advisories/GHSA-ffhc-5mcf-pf4q)
- [GHSA-wfc6-r584-vfw7: Cache poisoning in RSC](https://github.com/vercel/next.js/security/advisories/GHSA-wfc6-r584-vfw7)
- [Node.js — Wednesday, July 29, 2026 Security Releases (9 CVEs, 3 HIGH — postponed twice from Mon Jul 27)](https://nodejs.org/en/blog/vulnerability/july-2026-security-releases)

## Next.js 16.3.0 STABLE — Cache Poisoning, Supply-Chain via TypeScript CLI Default, Catch-All Page Hijack (August 3, 2026)

The 6-day-old `security.md` was missing three material security-relevant changes that ship in **`next@latest` = `16.3.0`** (npm-published 2026-08-03T21:03:18Z, stable release tagged by Tim Neutkens):

### Cache Poisoning After Prerender Abort — Silent Cache Corruption (PR #96426, jankaeryga — fixed in 16.3.0)

A correctness bug with security implications: under `cacheComponents: true`, `'use cache'` entries that started filling *after* a prerender was aborted (fast-click navigation, prefetch cancellation under `partialPrefetching: true`, slow connections) silently produced **empty cache entries**. The aborted `renderSignal` propagated into `AbortSignal.any(...)` inside the cache-fill codepath, so the cache wrote an empty stream and saved it — every subsequent user reading from that cache got an empty result for the duration of `cacheLife()`.

**Security implications:**
- **User-targeted attack via fast clicks** — an attacker can clear a cached value via repeated fast navigation, then trick a victim into seeing the empty response (e.g., empty shopping cart, empty notifications list, empty feed). This is a *low-effort* denial-of-functionality attack.
- **Cache poisoning amplification** — once an empty entry is written, the original data is *not* re-cached (the cache key now has an empty entry). Even after the attack ends, users see the empty result until `expire` fires.
- **Only affected apps with `cacheComponents: true`** + heavy `'use cache'` use + interactive navigation. Pre-CC apps (16.2.x and earlier default) were NOT affected.

**The fix:** PR #96426 removes `renderSignal` from `AbortSignal.any(...)` and short-circuits with a rejected promise before the cache-fill codepath. The cache now errors instead of saving an empty entry. **No user-visible behavior change for correct code** — the fix only changes the behavior of the broken path.

**Action items:**
1. Bump to `next@16.3.0` (stable) or `next@16.3.1-canary.0` (live now) — no code changes required
2. Audit your codebase for `cacheComponents: true` usage: `rg -n "cacheComponents\s*:\s*true" next.config.*`
3. If you're stuck on a pre-16.3.0 version, defensively wrap `use cache` data fetches in try/catch and reject on `AbortError` (signal-aborted fetches shouldn't write to cache)

**Sources:** [PR #96426 — `[Cache] Make caches error if called after prerender aborts`](https://github.com/vercel/next.js/pull/96426) · jankaeryga · merged 2026-08-03T11:42:26Z · **shipped in `16.3.0` stable** · closes [#96339](https://github.com/vercel/next.js/issues/96339).

### TypeScript CLI Default-ON — Supply-Chain Implications (PR #96497, timneutkens — ships in 16.3.0)

**The single biggest behavioral change in 16.3.0** is invisible but security-relevant: `experimental.useTypeScriptCli` flips from default-`false` to default-`true`. Every `next build` now runs the project-local `tsc` CLI to type-check.

**Supply-chain implications:**
- **Your installed `typescript` package is now on the build path.** Pre-16.3.0, Next.js bundled its own `typescript` JS API adapter; the project's `typescript` was only used by editors and CLI tools. In 16.3.0, Next.js spawns *your* installed `tsc` binary. A compromised `typescript` package can now affect every build, not just your IDE.
- **TypeScript 7 (Go-native) is the supported fast path.** If you've pinned `typescript: "^6.x"` to avoid the Go compiler, your build now runs the TS 6 JS API through the CLI path — slower (~50-200ms per build) but functionally equivalent.
- **Pin `typescript` explicitly** in `package.json` — `npm install -D typescript@latest` could install a typo-squat (the supply-chain-attack surface for `@types/*` and `typescript` is well-known; see the [ProjectDiscovery 2026 vulnerability curve](#why-this-matters-in-2026) above).
- **CI cache invalidation** — type checking is now a build step; bump `engines.node` and add `typescript` to your Renovate / Dependabot `typescript` group so upgrades trigger a cache rebuild.

**Action items:**
1. Confirm `typescript` is pinned in `package.json` (not caret-ranged): `rg "typescript" package.json`
2. If using TS 7, verify `tsc --version` resolves correctly in your build environment: `npx tsc --version` should print `7.x.x`
3. If using a custom transformer or `typescript` as a library, opt out: `experimental: { useTypeScriptCli: false }` in `next.config.ts`
4. Verify CI `engines.node` matches the TS 7 requirement (Node 20.15+ recommended)

**Sources:** [PR #96497 — `Enable TypeScript CLI by default`](https://github.com/vercel/next.js/pull/96497) · timneutkens · merged 2026-08-03T16:10:51Z · **shipped in `16.3.0` stable**.

### Catch-All Index Page Hijack (PR #96553, acdlite — fixed in 16.3.1-canary.0)

A 16.3.0-introduced bug with security implications: requesting `/blog/anything` served the catch-all index page (`/blog/[...slug]/page.tsx` with `slug = []`) instead of the proper dynamic page. The bug was caused by Next.js's catch-all routing logic mistaking the URL path for a slug array.

**Security implications:**
- **Information disclosure via fallback rendering** — any URL matching a catch-all base served the index page, which could leak the existence of internal routes or expose a less-restricted version of the page (e.g., admin index with public content vs. admin/[id] with PII). This is an OWASP API4:2023 "Unrestricted Resource Consumption" + partial API3:2023 "Broken Object Property Level Authorization" issue.
- **Authorization bypass potential** — if a catch-all route served the index while a dynamic `[slug]` page had stricter auth, attackers could request `/{admin-path}/any-garbage` and see the unauthenticated index instead of being redirected to login.

**The fix:** PR #96553 (acdlite, merged 2026-08-03T21:49:27Z) ships in `next@16.3.1-canary.0` (npm-published 2026-08-03T22:32:33Z).

**Action items:**
1. If you're on `next@16.3.0`, audit catch-all routes immediately: `rg -l "\[\.\.\." app/`
2. Test in dev: visit `/{catch-all-base}/anything-here` — should serve the dynamic page, not the index
3. Upgrade to `next@16.3.1-canary.0` (live now) or wait for `16.3.1` stable
4. Defensive workaround if stuck on 16.3.0: replace `[...slug]` with `[slug]` + add a static index page

**Sources:** [PR #96553 — `Fix catch-all index page being served for every other slug`](https://github.com/vercel/next.js/pull/96553) · acdlite · merged 2026-08-03T21:49:27Z · **shipped in `16.3.1-canary.0`** (npm-published 2026-08-03T22:32:33Z).

### Vercel August 2026 Security Release — Forward-Looking (Program Established July 13, 2026)

The Vercel Next.js monthly security release program (launched 2026-07-13, see [Vercel Next.js Security Release Program](#vercel-next-js-security-release-program-july-13-2026) above) means the **next scheduled security release is August 20, 2026** (one month after July 20, 2026's 9-CVE release). Calendar a reminder for August 19-20 to catch any pre-announced advisories.

**As of this writing (August 4, 2026)**, no `v16.3.x` security releases have been published (only the 16.2.11 / 15.5.21 LTS lines received the July 20 patch). The next patch-day for the LTS line is most likely August 20, 2026.


- **`revalidateTag(tag, 'max')` + `'use cache'` + Server Action POST returns 500 "Unexpected end of form" (canary.3 + 16.3.0 STABLE) — FIXED in `next@16.3.1-canary.4` by PR #96640 (closes [#96519](https://github.com/vercel/next.js/issues/96519))** — under `cacheComponents: true` + heavy `'use cache'` usage + `revalidateTag(tag, 'max')` called from a Server Action, the next Server Action POST would receive a 500 error with `Unexpected end of form`. The bug was that WorkStore inferred (rather than received explicitly) the request shape during Server Action POST revalidation, and the inferred shape misjudged the request, breaking the form parser. **FIXED in canary.4 via the executionMode refactor (PR #96640) which moves execution intent to entrypoints where the work is known**. **No code or config changes required** — just bump to `next@16.3.1-canary.4+`. Audit recipe: `rg -ln "revalidateTag.{0,5}.{0,50}max" app/ src/  # finds revalidateTag('tag', 'max') calls; also rg -n "updateTag" app/ src/` + check for any Server Action POST returning 500 with `Unexpected end of form` in your production logs. Cross-reference: see the new `## Next.js 16.3.1-canary.4 SHIPPED (August 6, 2026)` section below for the full PR #96640 architectural rationale + the executionMode refactor context.

- **Back-button click during hydration leaks Page A state into Page B (16.3.0 + all 16.3.1-canary.0/1/2/3) — FIXED in `next@16.3.1-canary.4` by PR #96252** — a Back click during hydration on slow devices / mid-tier mobile / heavy `useEffect` work could leak the previous page's client state (scroll position, focused input, state-setters from Page A) into Page B's first paint. For high-stakes apps (banking, health, admin dashboards) this is a **state-leak surface**: a user could briefly see Page A's authenticated state under Page B's URL after a Back click. **NOT a CVE-class vulnerability** (no memory disclosure, no auth bypass) but worth tracking. **FIXED in canary.4 via Navigation API hydration race fix (PR #96252, gaearon)**. **No code or config changes required** — bump to `next@16.3.1-canary.4+`. Workaround for stuck-on-pre-canary.4 users: set `prefetch={false}` on `<Link>` to opt out of the prefetching path that triggers the race. Cross-reference: see the new `## Next.js 16.3.1-canary.4 SHIPPED (August 6, 2026)` section below for the full PR #96252 Navigation API hydration contract walkthrough.

- **`experimental.serverSourceMaps: true` + Turbopack produces 3.37 GB of source maps per build (16.3.0 STABLE + canary.0/1/2/3) — FIXED in `next@16.3.1-canary.4`-ahead by a Turbopack config-flag-respect fix** — when `experimental.serverSourceMaps: true` is set, the Turbopack build path was silently **ignoring the flag** and emitting 3.37 GB of server source maps per build (the build artifact, not just dev). **From a security lens**: the leak surface is the **build artifact itself** — if the build artifact is shipped to production (or to a CDN), the 3.37 GB of source maps would expose the entire server codebase. **This is an information-disclosure vector** (server-side source code leaking via the build artifact) but it requires the deployment to (1) set `experimental.serverSourceMaps: true` AND (2) ship build artifacts to production. Most deployments ship only the `.next/server` output (not the source maps), so the practical impact is bounded to deployments that explicitly enable source maps in production. **FIXED in canary.4** (issue [#96748](https://github.com/vercel/next.js/issues/96748)). **No code changes required** — bump to `next@16.3.1-canary.4+`. Audit recipe: `rg -n "serverSourceMaps.*true" next.config.*` + ensure build artifacts are not shipped to production. Cross-reference: see the new `## Next.js 16.3.1-canary.4 SHIPPED (August 6, 2026)` section below.



## Next.js 16.3.1-canary.3 SHIPPED (August 5, 2026) — Silent Worker Hang Reliability Fix (PR #96636) + August 20 Monthly Security Release Pre-Roll + Better Auth 1.7.0-rc.4 (August 5, 2026)

The `security.md` was last touched in v1.5.20 (Aug 3, 21:03Z, 35h56min stale at this cron's check) and was missing several material security-and-reliability updates that have landed since. This cycle updates all three: the canary.3 SHIP of the silent-worker-hang fix (PR #96636, relevant as a reliability / denial-of-functionality concern), the Better Auth 1.7.0-rc.4 RC drop (auth-surface impact), and a forward-looking pre-roll of the August 20, 2026 Vercel monthly security release.

### PR #96636 Silent Worker Hang — Reliability / DoS Surface, NOT a Data-Exfiltration Vector (timneutkens, merged 2026-08-05T05:41:54Z, SHIPPED in `next@16.3.1-canary.3` npm-published 2026-08-05T06:27:06Z)

[PR #96636](https://github.com/vercel/next.js/pull/96636) by Tim Neutkens fixes the [silent production worker hang](https://github.com/vercel/next.js/issues/96613) that affected deployments combining Turbopack + cross-origin CDN `assetPrefix` + Web Workers via `new Worker(new URL('./worker.ts', import.meta.url))`. **From a security perspective this is a denial-of-functionality (DoS) surface, not a data-exfiltration vector** — the worker module simply never executes (every DevTools request returns 200, no error of any kind), so the worst case is silent feature breakage (an SVG-to-PNG button that never renders, an image-codec worker that never returns a decoded buffer, a Comlink-wrapped service that never responds). This is **NOT a CVE-class vulnerability** (no memory disclosure, no auth bypass, no remote code execution), but it IS a reliability concern that should be tracked alongside the monthly security cadence.

**Practical impact (security lens):**
- **Affected deployments:** Next.js 16.3.0 (or any earlier Turbopack build) + `assetPrefix: 'https://cdn.example.com'` (cross-origin) + any Web Worker construction via `new Worker(new URL('./worker.ts', import.meta.url))`. Common libraries that trigger this code path: `@resvg/resvg-js` (WASM SVG→PNG), `@napi-rs/canvas` (offscreen canvas), `@jsquash/*` (image codecs), `comlink` (RPC), custom WASM packages.
- **NOT affected:** Webpack users (webpack's `output.workerPublicPath: '/_next/'` workaround in `next.config.js → webpack()` was never broken); deployments where `assetPrefix` is same-origin (`assetPrefix: '/cdn-static'` style — no protocol/host change); deployments without Web Workers; Turbopack-only builds where workers are never instantiated.
- **Worker runtime chunk (`turbopack-<hash>.js`) was emitted with `CHUNK_BASE_PATH` from `assetPrefix` (the CDN)** while the worker entrypoint correctly used `experimental.turbopackWorkerAssetPrefix` (same-origin). Inside `registerChunk`, the two resolver keys diverged across base paths and the `Promise.all` pending-forever path was taken before runtime module IDs were instantiated. **No console error, no DevTools `onerror`, no Network tab failure** — the worker is just stuck.

**Action items:**
1. **If you're on `next@16.3.0` or `16.3.1-canary.0/.1/.2`** with Turbopack + cross-origin CDN + Workers: **upgrade to `next@16.3.1-canary.3` immediately** (no code changes required; npm-published 2026-08-05T06:27:06Z, the canary.3 version-tag landed 2026-08-05T06:01:23Z). The fix is in canary.3, NOT in stable 16.3.0 — stable 16.3.0 still has the bug.
2. Audit for Workers: `rg -ln "new Worker\(new URL\(" app/ src/`
3. Audit for cross-origin CDN: `rg -n "assetPrefix\s*:" next.config.*` — look for `https://` or `http://` URLs that differ from your app origin
4. Confirm `experimental.turbopackWorkerAssetPrefix` (if set) is the empty string `''` or your same-origin path (NOT a CDN path); the config line stays in place post-upgrade.
5. If stuck on 16.3.0 (i.e., cannot upgrade), the **workaround** is to bundle the Worker inline (no `new Worker(new URL(...))` pattern) — this avoids the cross-origin CDN path entirely.

**Sources:** [PR #96636](https://github.com/vercel/next.js/pull/96636) · timneutkens · merged 2026-08-05T05:41:54Z · **SHIPPED in `next@16.3.1-canary.3`** (npm-published 2026-08-05T06:27:06Z) · closes [#96613](https://github.com/vercel/next.js/issues/96613). Predecessor: [PR #93271](https://github.com/vercel/next.js/pull/93271) (the original `turbopackWorkerAssetPrefix` introduction, fixed by PR #96636's regression). See `performance.md` + `patterns.md` for the runtime + recipe lens on this same PR.

### Better Auth 1.7.0-rc.4 SHIPPED (August 5, 2026) — Auth Surface Update Ahead of August 20 Release

[Better Auth v1.7.0-rc.4](https://github.com/better-auth/better-auth/releases/tag/v1.7.0-rc.4) published 2026-08-05T00:26:40Z (literally the same day as the August 5 monthly cadence — Better Auth has been on a daily-or-better RC cadence since rc.0 on Jun 20, 2026). **DO NOT upgrade to 1.7.0-rc.4 in production yet** — wait for 1.7.0 stable. The RC.4 release is incremental over RC.3 (no new breaking changes announced); production codebases stay on `^1.6.26` until `1.7.0` STABLE.

**Why this is on `security.md` and not `auth.md`:** the RC train is now daily-or-better, and any release-candidate drop has security implications (auth libraries touch session cookies, account identity, SCIM, SAML — all of which are in-scope for `security.md`). The RC.4 changes per the [Better Auth 1.7.0-rc.4 GitHub release](https://github.com/better-auth/better-auth/releases/tag/v1.7.0-rc.4) are minor (documentation tightening + a small SCIM accounting fix), but they DO advance the 1.7 timeline by one release. **1.7.0 STABLE could ship within the August 20 Vercel monthly security window** — calendar both events.

**Audit recipe:**
```bash
# Confirm pinned dist-tag
rg "\"better-auth\"" package.json
# Should show ^1.6.26 for production (1.7.0-rc.X is the RC dist-tag, not @latest)

# If you have 1.7.0-rc.X installed (RC dist-tag) in any environment:
npm ls better-auth | grep rc
# Confirm it's only in staging/preview, NOT production
```

### August 20, 2026 Vercel Monthly Security Release — Pre-Roll (16.3.0 STABLE Pre-Patch Status)

The Vercel Next.js monthly security release program (launched 2026-07-13, first release July 20 with 4 HIGH + 5 MEDIUM = 9 CVEs across 16.2.11 / 15.5.21) means **the next scheduled release is August 20, 2026** — **15 days from this cron check** (Aug 5). Vercel's program announcement said: "We will publish security patches more regularly, and will give advance notice of these patches." Expect pre-announcement of CVEs around Aug 18-19.

**As of August 5, 2026**, the current Next.js stable line is `next@16.3.0` (npm-published 2026-08-03T21:03:18Z). **Critical fact for the August 20 release:**
- **`next@16.3.0` already includes ALL the canary.92+ security fixes** that were backported from the July 20, 2026 release (verified via the [`v16.3.0` release notes](https://github.com/vercel/next.js/releases/tag/v16.3.0) — every fix from the July 20 LTS batch is present in 16.3.0 since the canary.92+ PR set was merged BEFORE 16.3.0 cut). So **users on `next@16.3.0` STABLE are pre-patched for the August 20 batch for the same vulnerability set**.
- **Users on `next@16.2.x`** (Active LTS) will receive a `next@16.2.12` (or higher) LTS patch on or around August 20.
- **Users on `next@15.5.x`** (Maintenance LTS) will receive a `next@15.5.22` (or higher) LTS patch on or around August 20.
- **Users on `next@16.3.1-canary.X`** (canary train) will likely receive `next@16.3.1` STABLE before August 20 (canary.3 shipped Aug 5; canary cadence is 1 version per ~24h; STABLE typically follows canary by 5-14 days).

**Action items (calendar these for August 18-20):**
1. **Set a calendar reminder for August 19, 2026** (the day before expected release) to catch Vercel's pre-announcement blog post on [nextjs.org/blog](https://nextjs.org/blog).
2. **Audit your current Next.js pin** for the LTS line:
   ```bash
   npm ls next  # check the installed version
   rg "\"next\"" package.json  # check the declared range
   ```
3. **If on `next@16.2.x`:** plan to bump to the August 20 LTS patch (likely `16.2.12`).
4. **If on `next@15.5.x`:** plan to bump to the August 20 LTS patch (likely `15.5.22`). 15.x is in Maintenance LTS — security fixes only, no new features.
5. **If on `next@16.3.0` STABLE:** you're ahead of the August 20 LTS batch (pre-patched), but still check the [Next.js GitHub security advisories](https://github.com/vercel/next.js/security/advisories) for any NEW vulns reported since 16.3.0 cut.
6. **If on `next@16.3.1-canary.X`:** the canary train gets fixes earlier than the LTS line — you'll get the canary-branch-ahead fixes (including the PR #96636 silent-worker-hang fix) on the 24h cadence, so the monthly release is less relevant. Watch the canary train.

**Sources:**
- [Vercel Next.js Security Release Program announcement (July 13, 2026)](https://nextjs.org/blog/next-security-release-program)
- [July 2026 Security Release (July 20, 2026)](https://nextjs.org/blog/july-2026-security-release)
- [Socket.dev: Next.js moves to scheduled security releases (July 16, 2026)](https://socket.dev/blog/nextjs-moves-to-scheduled-security-releases)
- [Next.js 16.3.0 release notes (Aug 3, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0) — confirms canary.92+ fixes are in 16.3.0
- [Better Auth v1.7.0-rc.4 release](https://github.com/better-auth/better-auth/releases/tag/v1.7.0-rc.4) (Aug 5, 2026)
- [Hacktron AI Next.js Security Changelog](https://www.hacktron.ai/security-changelog/nextjs) — for tracking GHSA-published CVEs as they land


## Next.js 16.3.1-canary.4 SHIPPED (August 6, 2026) — 25-PR Cumulative Canary-Batch + August 20 Monthly Security Release Pre-Roll Refresh (T-14 days) + New Closed Issues #96748 / #96752 / #96797

`next@16.3.1-canary.4` **SHIPPED at 2026-08-06T00:10:18Z** (npm `dist-tag.canary` moved from `16.3.1-canary.3` → `16.3.1-canary.4`; the v1.5.28 cron (00:09Z, 1 minute earlier) correctly predicted the SHIP — the GitHub release tag `v16.3.1-canary.4` was published at 2026-08-05T23:59:55Z, the version-tag commit `866beee` landed at 2026-08-05T23:33:34Z, and the npm-publish landed 11min23s after the v1.5.28 cron committed). **The canary.4 batch contains 25 PRs + the version-tag commit = 26 commits ahead of canary.3**, which makes it the **largest single canary cut in the 16.3 cycle** (the canary.3 batch was 3 commits; the canary.2 batch was 1 commit; the canary.1 batch was 22 commits). The 25-PR canary.4 batch decomposes to:

- **9 internal refactor PRs (the executionMode refactor)** — PR #96570 + #96572 + #96576 + #96640 + #96659 + #96660 + #96662 + #96670 + #96674. These are the **same 9 PRs the v1.5.27 cycle documented in detail** under `server-components.md` → `## App Router Execution Mode Refactor — 9-PR Coordinated Set Ahead of 16.3.1-canary.3`. All pure internal refactors; **zero observable security or reliability change for App Router users on `next@16.3.0` STABLE**. PR #96640 (Move App Router execution intent to entrypoints) closes [issue #96519](https://github.com/vercel/next.js/issues/96519) (`revalidateTag(tag, 'max')` with `'use cache'` makes the next Server Action POST fail with 500 "Unexpected end of form") — **this is the security-relevant bug from the v1.5.28 closure list**, and it IS now fixed in canary.4.
- **5 material user-facing PRs** — PR #96252 (Back-before-hydration race fix), PR #96726 (cache revalidation correctness/perf), PR #96727 (`'use cache: private'` reuse), PR #96731 (consumer-driven foreground revalidation), PR #96697 (Turbopack hoisted-module registration). All 5 are documented in detail under `performance.md` / `routing.md` / `patterns.md` per the v1.5.28 cycle. From a **security lens**: only PR #96252 has a direct (low-severity) security implication — the Back-during-hydration race could previously leak Page A's stale client state into Page B's first paint, which for certain apps (banking, health) could mean a user's stale authenticated state from a prior account briefly persisting into the new page's UI. The fix uses the Navigation API to detect mid-hydration history-entry mismatch and replays the missed traversal. **No CVE-class vulnerability, but worth flagging for high-stakes apps.**
- **2 user-facing infra PRs** — PR #96606 (`Use Tailwind Turbopack loader in create-next-app`; affects new project scaffolding when `--tailwind` is used; no impact on existing projects) + PR #96681 (`fix(next/image): preserve image response after optimization`; closes issue #96612 — `next/og` ImageResponse after an SVG `next/image` request crash). From a security lens: **neither is CVE-relevant**; the next/image issue is a process-state module-cache bug, not exploitable.
- **6 docs-only PRs** — PR #96663 + #96696 + #96703 + #96751 + the docs touched by the executionMode refactor + the docs rewritten by #96606.
- **2 test/CI PRs** — PR #96725 (test fix) + #96735 (React vendor bump `7dfc7ccd-20260803` → `11eddecd-20260805`; documented in v1.5.27 cycle's `components.md` / `server-components.md`).
- **1 next/font internal refactor** — PR #95808 (next/font BeforeResolvePlugins as ImportMappingReplacement; Plugin API internal change, non-user-facing).
- **1 manifest loading edge case** — PR #96530 (`loadManifest` returns undefined with `handleMissing`; opt-in behavior for manifests with optional fields; non-material).
- **The version-tag commit** — `866beee` by `next-js-bot` at 2026-08-05T23:33:34Z.

**Canary-branch state after canary.4 SHIP**: `next@canary` → `16.3.1-canary.4`; canary-branch has **1 NEW commit ahead of canary.4** (PR #96774 [turbopack] Enable reexport-unknown execution test, merged 2026-08-05T23:49:39Z, already documented in v1.5.28 cycle as non-material test-enable). canary.5 expected within 2-12h on the 24h cadence; the v1.5.30 cron will document the canary.5 SHIP event if it lands in the next 6h.

### #96519 closure detail — `revalidateTag(tag, 'max')` + `'use cache'` Server Action 500 — NOW FIXED in canary.4

**Issue [#96519](https://github.com/vercel/next.js/issues/96519)** (`revalidateTag(tag, 'max')` with `'use cache'` makes the next Server Action POST fail with 500 "Unexpected end of form") was closed at **2026-08-05T12:51:12Z** by **PR #96640** (Move App Router execution intent to entrypoints; merged 2026-08-05T12:51:11Z). The bug: under `cacheComponents: true` + heavy `'use cache'` usage + `revalidateTag(tag, 'max')` called from a Server Action, the next Server Action POST (the one that triggers the revalidation) would receive a 500 error with `Unexpected end of form`. This is a **denial-of-functionality surface** for any Server-Action-driven revalidation pattern (the canonical `updateTag` + `revalidateTag` + `use cache` revalidation cycle) — apps on `next@16.3.0` STABLE were silently broken when this pattern was used. The fix moves the execution intent to entrypoints where the work is known, removing the inferable-but-not-explicit signal that caused the WorkStore to misjudge the request shape. **The user-observable change**: `revalidateTag(tag, 'max')` + Server Action POST now works correctly. **No code changes required** — the fix is internal to the App Router execution pipeline. **Affected deployments**: `next@16.3.0` STABLE since 2026-08-03 + all `next@16.3.1-canary.0/1/2/3`. **FIXED in `next@16.3.1-canary.4`**.

### New closed issues in the 6h window (security-adjacent, not CVE-class)

The following issues were closed in the 6h window since the v1.5.28 cron (00:09Z Aug 6 → 06:03Z Aug 6). **None are CVE-class vulnerabilities** (no memory disclosure, no auth bypass, no RCE); all are reliability/UX/telemetry concerns worth tracking alongside the monthly security cadence:

- **[Issue #96748 — Turbopack ignores `experimental.serverSourceMaps` (closed 2026-08-05T19:43Z)](https://github.com/vercel/next.js/issues/96748)** — when `experimental.serverSourceMaps: true` is set, the Turbopack build path was silently emitting **3.37 GB of server source maps per build** because the `serverSourceMaps` config flag was being ignored. From a security lens: the leak surface is the **build artifact itself** — if the build artifact is shipped to production (or to a CDN), the 3.37 GB of source maps would expose the entire server codebase. **This is an information-disclosure vector** (server-side source code leaking via the build artifact) but it requires the deployment to (1) set `experimental.serverSourceMaps: true` AND (2) ship build artifacts to production. Most deployments ship **only** the `.next/server` output (not the source maps), so the practical impact is bounded to deployments that explicitly enable source maps in production. No PR attribution found yet in the 6h window; **forward-looking — track for the canary.5 cut**.
- **[Issue #96752 — OTel `http.route` propagated to parent span after that span has ended (closed 2026-08-05T21:53Z)](https://github.com/vercel/next.js/issues/96752)** — `http.route` was set on the parent span after the parent span had already ended, so platform observability dashboards would show `http.route` stuck at a stale value for the lifetime of the parent span. From a security lens: **observability data integrity bug** (stale route labels in traces); no CVE but affects SOC dashboards and alerting rules that depend on `http.route` being current. No PR attribution found yet in the 6h window; **forward-looking — track for the canary.5 cut**.
- **[Issue #96797 — App Router announcer can announce the previous page's `<h1>` (closed 2026-08-06T04:27:59Z, ~1h35min before this cron)](https://github.com/vercel/next.js/issues/96797)** — when navigating between App Router pages rapidly, the `<next-route-announcer>` could announce the **previous page's `<h1>`** because `document.title` is transiently empty between the old and new `<title>` nodes. From a security lens: **a11y surface** (screen reader users get wrong page announcements); from a UX lens: misleading announcements. No CVE. The fix is in the canary-branch but not yet attributed to a specific PR in the 6h window; **forward-looking — track for the canary.5 cut**. Cross-references `components.md` (announcer-related content) and `routing.md` (App Router navigation patterns).

### August 20, 2026 Vercel Monthly Security Release — Pre-Roll Refresh (T-14 days)

The Vercel Next.js monthly security release program (launched 2026-07-13, first release July 20 with 4 HIGH + 5 MEDIUM = 9 CVEs across 16.2.11 / 15.5.21) means the next scheduled release is **August 20, 2026 — 14 days from this cron check** (Aug 6). The T-15-days note from v1.5.26 is updated to T-14-days here; the rest of the pre-roll content (calendar reminders, audit recipes, per-line bump plans) is unchanged from the `### August 20, 2026 Vercel Monthly Security Release — Pre-Roll (16.3.0 STABLE Pre-Patch Status)` subsection above.

**New since the T-15-days note**: the August 20 batch will likely include any NEW CVEs reported to Vercel between Aug 5 and Aug 18-19 (the typical pre-announcement window). The 3 new closed issues in the 6h window (#96748, #96752, #96797) are **not CVE-class** so they are NOT expected in the August 20 batch; they are routine bug fixes that ship on the canary cadence, not the monthly security cadence. **No new pre-announced CVEs as of Aug 6.** Calendar remains: **Aug 19 reminder** to catch Vercel's pre-announcement blog post.

### Better Auth 1.7.0-rc.4 status — 1 day after the rc.4 SHIP (no new RC drop)

The Better Auth RC train has been on a **daily-or-better cadence** since rc.0 on Jun 20, 2026 (rc.1, rc.2, rc.3, rc.4 each ~7-10 days apart, but with 2-4 day gaps between some). **1.7.0-rc.4 SHIPPED Aug 5 at 00:26:40Z** (per v1.5.26 cycle). **No new RC drop in the 24h since** (Aug 5 → Aug 6). The current RC cadence suggests the next RC drop is expected within 2-4 days (likely Aug 8-9 as rc.5) unless the team pivots to STABLE. **1.7.0 STABLE could ship within the August 20 Vercel monthly security window** (14 days out) — calendar both events. **Production codebases stay on `^1.6.26`** until 1.7.0 STABLE.

### Audit recipe (canary.4 SHIP + Aug 20 pre-roll refresh)

```bash
# Step 1: confirm you're on canary.4+ for the #96519 fix
npm ls next | head -3
# Should show 16.3.1-canary.4 or later (npm-published 2026-08-06T00:10:18Z)

# Step 2: audit Server Action revalidation patterns (the #96519 affected surface)
rg -ln "revalidateTag\s*\(.*['\"]use cache['\"]|updateTag\s*\(" app/ src/
# If you have this pattern + cacheComponents: true + you saw 500 "Unexpected end of form" on a Server Action POST,
# you were affected by #96519 — bump to canary.4+

# Step 3: audit serverSourceMaps usage (the #96748 affected surface)
rg -n "serverSourceMaps\s*:\s*true" next.config.*
# If set, ensure build artifacts (including source maps) are NOT shipped to production/CDN

# Step 4: audit OTel / instrumentation usage (the #96752 affected surface)
rg -ln "@opentelemetry/api|@vercel/otel" instrumentation.ts src/
# If using OTel, monitor for stale `http.route` values on parent spans

# Step 5: audit announcer usage (the #96797 affected surface)
rg -ln "next-route-announcer|<h1>" app/
# If using custom announcer patterns, check screen-reader announcements for stale content

# Step 6: pre-August-20 audit (unchanged from v1.5.26 cycle)
npm ls next  # check the installed version
rg "\"next\"" package.json  # check the declared range
```

**Sources:**
- [Next.js v16.3.1-canary.4 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.4) (published 2026-08-05T23:59:55Z)
- [Next.js 16.3.1-canary.4 npm dist-tag movement](https://github.com/vercel/next.js/blob/main/packages/next/package.json) → `npm view next dist-tags.canary` (npm-published 2026-08-06T00:10:18Z)
- [Next.js canary-branch compare `v16.3.1-canary.3...v16.3.1-canary.4`](https://github.com/vercel/next.js/compare/v16.3.1-canary.3...v16.3.1-canary.4) — 26 commits
- [Next.js canary-branch compare `v16.3.1-canary.4...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.4...canary) — 1 commit (PR #96774, non-material)
- [Issue #96519 — `revalidateTag(tag, 'max')` + `'use cache'` Server Action 500](https://github.com/vercel/next.js/issues/96519) (closed 2026-08-05T12:51:12Z by PR #96640)
- [Issue #96748 — Turbopack ignores `experimental.serverSourceMaps`](https://github.com/vercel/next.js/issues/96748) (closed 2026-08-05T19:43Z; source-map build artifact information-disclosure vector)
- [Issue #96752 — OTel `http.route` propagated to parent span after span ended](https://github.com/vercel/next.js/issues/96752) (closed 2026-08-05T21:53Z; observability data integrity bug)
- [Issue #96797 — App Router announcer can announce the previous page's `<h1>`](https://github.com/vercel/next.js/issues/96797) (closed 2026-08-06T04:27:59Z; a11y surface)
- [Vercel Next.js Security Release Program announcement (July 13, 2026)](https://nextjs.org/blog/next-security-release-program)
- [Next.js 16.3.0 release notes (Aug 3, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0) — confirms canary.92+ fixes are in 16.3.0
- [Better Auth v1.7.0-rc.4 release](https://github.com/better-auth/better-auth/releases/tag/v1.7.0-rc.4) (Aug 5, 2026) — most recent RC drop; 1.7.0 STABLE could ship within Aug 20 window
- Cross-references: `performance.md` → `## 16.3.1-canary.4-ahead — Navigation Back-Before-Hydration Race Fix (PR #96252) + Cache Components Revalidation Refactor (PR #96726 / #96727 / #96731) + Turbopack Hoisted-Module Registration Fix (PR #96697)` for the runtime + audit-recipe lens on the 5 material user-facing PRs; `routing.md` → `## 16.3.1-canary.4-ahead — Navigation Back-Before-Hydration Race Fix (PR #96252, gaearon, August 5, 2026)` for the navigation race lens; `patterns.md` → `## Pattern: Cache Components Revalidation Lifecycle (`updateTag` + `'use cache: private'` Reuse) — PR #96726 + PR #96727 + PR #96731 (unstubbable + ztanner, August 5, 2026)` for the composite recipe lens; `server-components.md` → `## App Router Execution Mode Refactor — 9-PR Coordinated Set Ahead of 16.3.1-canary.3 (August 5, 2026)` for the executionMode refactor (including the PR #96640 #96519 fix); `components.md` → `## React 19.3.0-canary-11eddecd-20260805 SHIPPED + React main branch: enableConditionalUseWarning flag (PR #37203, August 5, 2026)` for the React vendor bump.

---

## Keyv / Cacheable Shai-Hulud Supply-Chain Worm (August 4, 2026) — +440 Legitimate npm Packages Compromised, >2B Monthly Installs — `keyv-shai-hulud` — `setup.md` Update Required

**The single largest frontend-relevant supply-chain event since the TanStack compromise (May 11, 2026).** Starting at **2026-08-04T10:53Z**, an attacker compromised the GitHub maintainer account behind the `keyv` namespace ("Jaredwray") and used it to inject a credential-stealing worm into the entire `keyv` + `cacheable` + `cacheable-request` + `file-entry-cache` + `flat-cache` + `cache-manager` + `@cacheable/*` ecosystem. By **2026-08-05T13:15Z** the worm had spread to **~444 legitimate npm packages** across **~1,381–2,236 malicious versions** (count varies by source — Socket reports ~2,236 malicious versions, Aikido reports ~1,381 versions across ~444 packages) with a **combined monthly install count of >2 billion**.

**Why this matters for frontend skills:** even though most affected packages are **server-side** (not direct frontend deps), they pull into **any Node.js / Next.js / Express / Vite / Vitest project** as **transitive dependencies** via very common tooling:

- **`flat-cache` 6.1.24** (~580M monthly downloads) — transitive dep of `eslint` + every ESLint config + many bundlers' plugin pipelines
- **`file-entry-cache` 11.1.6** (~571M monthly downloads) — transitive dep of `eslint` + many linting tools
- **`keyv` 6.0.0** (~604M monthly downloads, the source of the worm) — transitive dep of `cacheable-request` + many caching layers + storage adapters
- **`cacheable-request` 13.0.20** (~137M monthly downloads) — transitive dep of `got` + `@npmcli/arborist` + `update-notifier`
- **`cacheable` 2.5.1** + **`@cacheable/memory` 2.2.1** + **`@cacheable/utils` 2.5.1** + **`@cacheable/node-cache` 3.1.2** + **`@cacheable/net` 2.1.1** + **`cache-manager` 7.2.10** — all transitive deps of various API/storage layers
- **`ecto` 5.0.1** — niche transitive dep

The payload is **delivered through install-time lifecycle (`preinstall`) scripts** — `npm install` itself executes the malicious code. Once on a developer or CI machine, the worm:

1. **Harvests credentials**: cloud keys (AWS / GCP / Azure), HashiCorp Vault tokens, Kubernetes service-account tokens, GitHub Actions OIDC tokens, npm publish tokens, `.npmrc` auth, plus any plaintext secrets in env / `.bash_history` / project files
2. **Uses any recovered npm publish token** to push malicious versions to other packages the compromised account controls (the worm is **self-propagating** via `npm OIDC trusted publishing`)
3. **Persists in the IDE** (the very first action on Aug 4 was introducing IDE persistence payloads to the `keyv` repo at ~9:00Z — before the npm publish wave)
4. **Uses an Ethereum smart contract to dynamically retrieve C2 domains** — the contract was initially configured with three domains before being updated to return only `npm-cache[.]com`

This is **Mini Shai-Hulud** lineage (sharing characteristics with TeamPCP and the earlier `antv` campaigns from June 2026). The same pattern as TanStack (May 11), node-ipc (May 15), the original Shai-Hulud (Sep 2025), Mini Shai-Hulud (June 1), and Mastra (June 17) — **a trusted marketplace or scope is compromised, malicious versions are published, and the malicious code is buried inside working features**.

### Affected versions (the worm's npm-published malicious versions — DO NOT INSTALL)

The `keyv-shai-hulud` tag tracks ~2,236 malicious versions across ~444 packages. The headline confirmed-package+version pairs as of Aug 5, 2026:

| Package | Malicious versions | Monthly downloads | Direct vs transitive |
|---|---|---|---|
| `keyv` | 6.0.0 | ~604M | Transitive (via `cacheable-request`, `got`, `@npmcli/arborist`, `update-notifier`) |
| `flat-cache` | 6.1.24 | ~580M | **Transitive of `eslint`** — affects every ESLint-using project |
| `file-entry-cache` | 11.1.6 | ~571M | **Transitive of `eslint`** |
| `cacheable-request` | 13.0.20 | ~137M | Transitive of `got`, `@npmcli/arborist`, `update-notifier` |
| `cacheable` | 2.5.1 | ~30M | Transitive |
| `@cacheable/memory` | 2.2.1 | ~28M | Transitive |
| `@cacheable/utils` | 2.5.1 | ~34M | Transitive |
| `@cacheable/node-cache` | 3.1.2 | ~6M | Transitive |
| `@cacheable/net` | 2.1.1 | ~3.7K | Transitive |
| `cache-manager` | 7.2.10 | ~16M | Transitive |
| `ecto` | 5.0.1 | ~4.5K | Transitive |
| `@adminide-stack/clock-tik-browser` | 12.0.24 | — | Indirect (org-scoped) |
| `@adminide-stack/yantra-mobile` | 12.0.33 | — | Indirect (org-scoped) |
| `@arv-bedrock/auth` | 1.1.7, 1.1.8 | — | Indirect |
| `@arv-bedrock/auth-admin` | 1.0.2, 1.0.3 | — | Indirect |
| `@arv-bedrock/logger` | 1.7.1, 1.7.2 | — | Indirect |

The **full list** (~444 packages) is tracked at [github.com/wiz-sec-public/wiz-research-iocs/blob/main/reports/keyv-packages.csv](https://github.com/wiz-sec-public/wiz-research-iocs/blob/main/reports/keyv-packages.csv) (Wiz Research IOCs repo, refreshed as new packages are identified). The full worm activity tag is [`#keyv-shai-hulud`](https://opensourcemalware.com/?search=%23keyv-shai-hulud) on [opensourcemalware.com](http://opensourcemalware.com).

### Detection: Indicators of Compromise (IOCs)

Wiz Research published the IOC list. The most critical high-signal indicators:

- **Malicious file names dropped during install**: `math_init.js` and `Math_Symbol.js` (the worm's loader names — searched in `~/.npm`, `node_modules/.cache`, and project root)
- **HTTP user-agent on npm publish calls**: `Bun/1.3.13` (the worm uses Bun to publish malicious versions)
- **C2 domain** (current): `npm-cache[.]com` (the Ethereum smart contract was updated to return only this domain; the contract was funded by an address previously flagged for scam activity)
- **Eth smart contract**: dynamically resolves C2; contract owner address funded by known-scam address
- **IDE persistence artifacts** (the very first action on Aug 4 introduced these to the `keyv` repo at ~9:00Z): VS Code `settings.json` hooks + workspace trust bypasses
- **Filesystem paths searched by the worm**: `~/.aws/credentials`, `~/.aws/config`, `~/.docker/config.json`, `~/.kube/config`, `~/.npmrc`, `~/.bash_history`, env vars matching `*TOKEN*`, `*SECRET*`, `*KEY*`, `*PASS*`

### The lockfile is the source of truth — `npm audit` is NOT enough

`npm audit` does **not** detect compromised-package / typosquat-with-correct-metadata attacks. To detect, you need:

1. **Lockfile-vs-version pin diff** — `npm ls` + check the `resolved` field in `package-lock.json` against the known-good version list
2. **Behavioral SCA** — Socket.dev, Snyk, or Wiz (any of them) detect install-time `preinstall` scripts as suspicious
3. **Filesystem scan** for the IOCs above (`math_init.js`, `Math_Symbol.js`, IDE persistence artifacts)
4. **Outbound network monitor** on developer machines and CI runners — flag connections to `npm-cache[.]com`

### Recommended action (Priority order — Aug 5+ teams)

**Step 1 — IMMEDIATE (within 24h): audit your lockfile**

```bash
# 1. Check if any of the worm-affected packages are in your lockfile
npm ls keyv flat-cache file-entry-cache cacheable cacheable-request cache-manager 2>/dev/null
npm ls @cacheable/memory @cacheable/utils @cacheable/node-cache @cacheable/net 2>/dev/null

# 2. If any are present, check the installed version against the malicious list
# (do not run `npm install` again until you've verified — the malicious version
# runs on install)

# 3. For each installed package, check if it's in the malicious list above
# If yes: STOP — do NOT rebuild. Treat the machine as compromised.

# 4. Search your filesystem for the IOCs
find ~ /tmp /var/tmp -name 'math_init.js' -o -name 'Math_Symbol.js' 2>/dev/null
rg -l 'Bun/1.3.13' ~/.npm/ /var/log/ 2>/dev/null
```

**Step 2 — Pin to known-good versions** (after verifying the lockfile is clean)

For the most common transitive exposure (`flat-cache` + `file-entry-cache` via `eslint`):

```json
// package.json overrides (npm) or pnpm.overrides (pnpm)
{
  "overrides": {
    "flat-cache": "6.1.23",        // last-known-good before 6.1.24
    "file-entry-cache": "11.1.5",  // last-known-good before 11.1.6
    "keyv": "^5.0.0 || ~5.3.0",    // avoid 6.x line entirely until clean
    "cacheable-request": "^12.0.0", // avoid 13.x line until clean
    "cacheable": "^2.4.0"          // avoid 2.5.1
  }
}
```

**Step 3 — Rotate any secrets that touched the affected machines**

Treat all machines where `npm install` ran between **2026-08-04T10:53Z** and **2026-08-05T13:15Z** as potentially compromised. Rotate:

- npm publish tokens (`npm login` to invalidate + the next publish forces re-auth)
- GitHub Actions OIDC trust (revoke + re-create the trust policy for affected repos)
- AWS / GCP / Azure access keys (any env vars with these exposed)
- Kubernetes service-account tokens
- HashiCorp Vault tokens
- Any `.npmrc` auth tokens (`//registry.npmjs.org/:_authToken=...`)

**Step 4 — Disable install scripts as a defense-in-depth**

```bash
# Add to ~/.npmrc (and CI runners' .npmrc) to refuse ALL install scripts:
ignore-scripts=true

# OR per-project (less restrictive — only blocks the high-risk ones):
# npm config set ignore-scripts true
```

**Step 5 — Forward-looking: the pattern is accelerating**

This is the **6th major npm supply-chain event in 2026**: TanStack (May 11) → node-ipc (May 15) → Mini Shai-Hulud (June 1) → Mastra (June 17) → keyv/Cacheable Shai-Hulud (August 4). The pattern of **dormant maintainer accounts in trusted scopes** is now the dominant attack vector. Audit npm scope ownership quarterly and remove former maintainers immediately.

### Audit recipe (single bash one-liner)

```bash
# 1. Detect presence of worm-affected packages in your lockfile
npm ls --all 2>/dev/null | grep -E '(keyv@|flat-cache@|file-entry-cache@|cacheable@|cacheable-request@|cache-manager@|@cacheable/(memory|utils|node-cache|net)@|ecto@)'

# 2. Scan filesystem for worm IOCs
find ~ /tmp /var/tmp -name 'math_init.js' -o -name 'Math_Symbol.js' 2>/dev/null

# 3. Scan npm logs for Bun user-agent
grep -r 'Bun/1.3.13' ~/.npm/ 2>/dev/null

# 4. Check for IDE persistence artifacts
rg -n 'npm-cache' ~/.vscode/ ~/.vscode-server/ 2>/dev/null
rg -n 'malicious' ~/.config/Code/User/settings.json 2>/dev/null

# 5. Rotate: npm token
npm token list  # show current tokens
npm token revoke <token-id>  # revoke any token used on a compromised machine

# 6. Check the lockfile for the exact malicious versions
rg -n '"flat-cache":\s*"6\.1\.24"|"file-entry-cache":\s*"11\.1\.6"|"keyv":\s*"6\.0\.0"|"cacheable":\s*"2\.5\.1"|"cacheable-request":\s*"13\.0\.20"|"cache-manager":\s*"7\.2\.10"' package-lock.json pnpm-lock.yaml yarn.lock 2>/dev/null
```

### Why this matters in the broader Next.js / frontend context

- **`next@16.3.0` STABLE is NOT affected** (no Next.js core deps in the malicious list), but **transitively pulled packages** (`flat-cache` + `file-entry-cache` via `eslint`, `keyv` via various plugins) ARE affected. **Run the audit recipe on every project before the next `npm install`.**
- **`create-next-app@latest`** does not directly use any of the affected packages, but the generated project's `eslint.config.mjs` + ESLint plugins DO pull in `flat-cache` + `file-entry-cache`.
- **CI runners** that ran `npm install` between Aug 4 10:53Z and Aug 5 13:15Z need to be rotated (treat them as compromised — see Step 3 above).
- **Vercel deployments** are not directly affected (Vercel pins its own internal lockfile), but **your project's Vercel build may have pulled a malicious transitive dep** if you ran `npm install` locally before pushing. Audit `package-lock.json` against the malicious list.

## Next.js 16.3.1 STABLE SHIPPED (August 13, 2026) + 16.3.1-canary.16 SHIPPED (August 14, 2026) + August 20 Monthly Security Release T-6 Days + React 19.3.0-canary-beef6d60 SHIPPED + @clerk/nextjs 7.7.5 STABLE

**`next@16.3.1` STABLE SHIPPED at npm `dist-tag.latest` 2026-08-13T22:45:02Z** — 18 minutes before this cron's 00:03Z start. This is the **first 16.3.x STABLE patch release** and includes 19 backports from the canary train. The most security-relevant backports:

**PR #97315 (backport of PR #96988)** — `Keep the dev validation worker alive across HMR updates`. The dev validation worker was being recreated on every HMR update, which meant any state in the worker (including security validation state) was reset. The fix keeps the worker alive across HMR updates, preserving validation context. **Security relevance**: for apps using custom security headers validation in dev mode, the validation context is now preserved across hot reloads.

**PR #97328 (backport of PR #97190)** — `Compile the middleware redirect routes up front in dev`. Middleware redirect routes are now compiled at startup rather than lazily on first request. **Security relevance**: earlier compilation means security-relevant middleware logic (rate limiting, IP allowlisting, bot detection, auth checks on redirect routes) is validated at startup rather than at request time. Runtime errors in redirect middleware now surface at `next dev` startup instead of at the first request.

**PR #97311 (backport of PR #97166)** — `Restore the live headers() view of the incoming request`. **Security relevance**: for apps using `headers()` to read security-relevant headers (CORS, CSP, auth tokens) that are injected or modified by an upstream proxy/load balancer, the live view now correctly reflects the final state of the request. Previously, a proxy that mutated `request.headers` could leave `headers()` returning stale pre-mutation values, causing security checks to see incorrect data.

**PR #96733** — `fix(next/image): preserve image response after optimization`. The image optimization response was being lost after the optimization step, causing the original response to be returned instead. **Security relevance**: this is a medium-severity issue for apps that rely on image optimization to enforce security policies (e.g., hotlink protection, referrer checking at the image optimization layer).

**PR #97317 (backport of PR #97213)** — `Turbopack: Fix HMR for dynamic imports evaluated from layouts`. HMR now correctly invalidates when a dynamic import changes. **Security relevance**: for apps with security-critical dynamic imports (e.g., dynamic auth module loading), HMR now correctly triggers re-evaluation when the imported module changes.

**The 16.3.0 → 16.3.1 security upgrade checklist** (5 steps):
1. `npm install next@16.3.1` — the canonical upgrade. No config changes required.
2. `npm ls next` — verify `16.3.1` is installed.
3. **`rg -n "headers\(\)" app/middleware.ts app/proxy.ts`** — if your middleware/proxy reads security-relevant headers via `headers()`, verify the values are correct after the PR #97166 live-view fix. The fix resolves the stale snapshot regression.
4. **`next dev &` then check startup logs** — after upgrading to 16.3.1, middleware redirect routes are compiled at startup. Any compilation errors in redirect middleware now surface at startup instead of at first request.
5. If using custom security validation in dev mode: verify that validation context is preserved across hot reloads (the PR #96988 fix).

### Next.js 16.3.1-canary.16 SHIPPED (August 14, 2026) — Self-Hosted Deployment Security Fix

**`next@16.3.1-canary.16` SHIPPED at npm `dist-tag.canary` 2026-08-14T00:05:06Z** — literally 2 minutes after this cron's 00:03Z start. The most important canary.16 commit for security is:

**PR #95238** — `fix(cache-components): decompress postponed resume body before parsing`. **This is a self-hosted deployment security/reliability fix**. When the resume body arrives gzip-compressed (which happens behind Vercel's infrastructure and self-hosted nginx/cloudflare setups with gzip-on-the-wire), the gzip binary bytes (`0x1f 0x8b`) were being misinterpreted as UTF-8. This caused `parsePostponedState` to fail, throwing `Invariant: invalid postponed state` (E314). The error was caught, degraded to `type:1`, and resulted in a logged server error with an HTTP 200 fallback — meaning the route couldn't resume its prerendered HTML shell. **The security angle**: for self-hosted deployments, this silent fallback could bypass security checks that rely on the Cache Components resume path. The fix adds proper `Content-Encoding` detection and decompression. **Self-hosted users with gzip-on-the-wire (nginx with `gzip on`, Cloudflare with auto-gzip, Vercel) should upgrade to canary.16** for the full resume path.

**Workaround for self-hosted users until canary.16 ships as STABLE**: disable gzip for the POST `/_next/data/.../...resume` endpoints:
```nginx
# nginx — disable gzip for Cache Components resume endpoints
location ~* ^/\_next/data/.*\.json$ {
    gzip off;
    proxy_set_header Content-Encoding "";
}
```

### August 20 Monthly Security Release — T-6 Days

**The Aug 20 batch is now T-6 days away as of 2026-08-14**. The v1.5.50 cycle documented the Critical Dev-Mode Security Disclosure #97157 (unauthenticated inspector UUID + CDP RCE + webpack source-map file-read + unauthenticated `/_next/mcp` + HMR websocket) as the headline candidate. **The v1.5.50 prediction was: `next@16.3.1` is pre-patched for the Aug 20 batch** — that prediction is now confirmed, since `next@16.3.1` STABLE was published without addressing #97157 (the fix was not in the canary train).

**The pre-batch triage state as of this cron**:
- **Issue #97157 (dev-mode inspector UUID + source-map file-read + `/_next/mcp` + HMR websocket)** — closed by `github-actions[bot]` at 2026-08-11T07:18:25Z (not fixed). **The canonical fix is expected in the Aug 20 batch.** The issue remains the headline candidate.
- **No new pre-announced CVEs** as of this cron's check (verified via `GET https://github.com/advisories?query=next.js`).
- **next@16.3.1 STABLE is pre-patched** for whatever the Aug 20 batch contains — the canary train from `canary.92+` contains the fix candidate.
- **Expected Aug 20 patch versions**: `next@16.3.1-patch` (if any fixes land in 16.3.1) + `next@16.3.2` + `next@15.5.24` + `next@14.2.36`.
- **The Aug 20 monthly security release is independent of next@16.3.1 STABLE** — upgrading to 16.3.1 STABLE does NOT address #97157. Continue applying the 5 mitigations from v1.5.50.

**The 5 mitigations for #97157 (until the Aug 20 fix ships)**:
1. **DO NOT bind `next dev` to all interfaces** — explicitly bind to `127.0.0.1` or use `--hostname 127.0.0.1`
2. **DO NOT expose `next dev` to a LAN** (common Docker dev setups forward port 3000 to the LAN)
3. **DO NOT visit malicious sites while running `next dev`** — the DNS rebinding drive-by is the worst case
4. **Disable MCP explicitly** in `next.config.ts` via `experimental: { mcpServer: false }` if not using it
5. **Watch for the fix** in canary.14+ and in the Aug 20 Vercel monthly security release

### React 19.3.0-canary-beef6d60 SHIPPED (August 13, 2026)

**`react@canary 19.3.0-canary-beef6d60-20260813` SHIPPED at npm `dist-tag.canary` 2026-08-13T16:30:24Z**. The new canary bundle includes PR #37168 + PR #37169 (the 9th and 10th PRs in the Jack Pope Fragment cleanup series). No security-material changes in this bundle. The Fragment event listener fixes reduce the attack surface for XSS via orphaned event listeners on Fragment children. **No security-relevant React changes in this cycle.**

### @clerk/nextjs 7.7.5 STABLE SHIPPED (August 13, 2026)

**`@clerk/nextjs@latest 7.7.5` SHIPPED at GitHub release 2026-08-13T21:23:04Z**. One patch change: **PR #9273 adds runtime migration errors when using the removed `<SignedIn>`, `<SignedOut>`, and `<Protect>` components**. This is a reliability fix (clearer error messages) rather than a security fix. **Production guidance**: `@clerk/nextjs@^7.7.5` is the recommended production pin. Run `rg -n "<SignedIn>|<SignedOut>|<Protect>" src/ app/` to find any usage of the removed components before upgrading.

### Sources

- [Socket.dev blog — Popular npm Packages in the keyv and Cacheable Namespaces Compromised in Active Supply Chain Attack](https://socket.dev/blog/popular-npm-packages-in-the-keyv-and-cacheable-namespaces-compromised-in-active-supply-chain) — Aug 4, 2026 (the first-mover coverage; identified the `Bun/1.3.13` user-agent; tracked the worm's npm publish wave in real-time)
- [Aikido Security — Keyv and friends compromised in active Shai-Hulud supply chain attack](https://www.aikido.dev/blog/keyv-and-friends-compromised-in-npm-supply-chain-attack) — Aug 4, 2026 (the prevalence data: 444 packages / ~2,236 malicious versions / >2B monthly installs)
- [OX Security — A New Infostealer Worm Hits npm, affecting Keyv and Cacheable](https://www.ox.security/blog/a-new-infostealer-worm-hits-npm-affecting-keyv-and-cacheable) — Aug 4, 2026 (the Shai-Hulud lineage attribution + cross-campaign comparison)
- [SC Media — Keyv, cacheable npm supply chain attack hits 400-plus packages](https://www.scworld.com/news/keyv-cacheable-npm-supply-chain-attack-hits-400-plus-packages) — Aug 4, 2026 (the news-cycle coverage)
- [Cloudsmith — Keyv and Cacheable npm Packages Compromised in Active Supply-Chain Attack](https://cloudsmith.com/blog/keyv-and-cacheable-npm-packages-compromised-in-active-supply-chain-attack) — Aug 5, 2026 (the detailed worm-mechanics walkthrough + the Ethereum smart contract C2 explanation)
- [opensourcemalware.com — `#keyv-shai-hulud` activity tracker](https://opensourcemalware.com/?search=%23keyv-shai-hulud) — the live worm activity feed
- [Cloudsmith — Evolution of Shai-Hulud Worms](https://cloudsmith.com/blog/evolution-of-shai-hulud-worms) — the worm-family lineage context (the August 4 attack is the **Mini Shai-Hulud** descendant — sharing characteristics with TeamPCP and the `antv` campaigns from June 2026)
- [`npm` security advisories feed](https://github.com/advisories?query=keyv) — the npm-side advisory tracker for affected packages (each affected package gets a GHSA advisory shortly after the worm publishes)
- Cross-references: `setup.md` → `## npm `overrides` + `pnpm.overrides` for Lockfile Pinning in Monorepos` for the canonical lockfile-pin recipe (the Step 2 overrides recipe above is the canonical application of that pattern for this worm) + `setup.md` → `## Snyk / Socket.dev / npm-audit-rescan for Supply-Chain Detection` for the SCA tooling lens; `auth.md` → `## Secrets Rotation After a Supply-Chain Incident` for the secret-rotation procedure (Step 3 above is a partial application; the canonical procedure covers npm + GitHub OIDC + cloud + vault + .npmrc tokens)


## Next.js 16.3.1-canary.12 SHIPPED (August 11, 2026) + 16.3.1-canary.13 SHIPPED (August 12, 2026) + Critical Dev-Mode Security Disclosure #97157 Unauthenticated Inspector UUID → CDP RCE + 3-PR Legacy PPR Refactor Now Live + August 20 Monthly Security Release Pre-Roll Refresh (T-8 days)

**`next@16.3.1-canary.12` SHIPPED at npm `dist-tag.canary` 2026-08-11T18:03:19Z** — **13h20min EARLIER than predicted**. v1.5.49 predicted the npm-publish at ~2026-08-12T07:23Z (the typical 24h cadence window after the canary-branch tag landed at 17:23:13Z); actual npm-publish landed at 18:03:19Z, 13h20min before the predicted window. The canary-branch tag was created at 17:23:13Z, GitHub release tag `v16.3.1-canary.12` published at 17:48:10Z (~25min later), npm-publish followed at 18:03:19Z (~40min after the GitHub release tag). The v1.5.49 prediction was based on the typical 24h cadence observed since canary.106; the v1.5.49 cycle correctly captured that the canary-branch-ahead state was 15 commits with 6 MATERIAL PRs + 8 non-material PRs + 1 version-tag, but did not predict the GitHub release tag + npm-publish would land 13h20min early. The bundle contains **15 commits ahead of canary.11 = 6 MATERIAL + 8 non-material + 1 version-tag** — same as the v1.5.49 cycle's documented canary-branch-ahead state, now SHIPPED. **`next@16.3.1-canary.13` SHIPPED at npm `dist-tag.canary` 2026-08-12T00:02:03Z** — ~6h before this cron's 06:02Z start, 4 commits ahead of canary.12. **canary-branch is now 4 commits ahead of canary.13** at this cron's check (verified via `GET /repos/vercel/next.js/compare/v16.3.1-canary.13...canary` returning `ahead_by: 4, behind_by: 0`) — 4 PRs (PR #97208 [turbopack] shared runtime default-on-canary-only + PR #95439 fix stale data after navigation despite revalidation + PR #97215 fix shared Turbopack runtime initialization race + PR #97181 allow literal exports in `'use cache'` files). **Critical new material this cycle**:

**(1) `## Next.js 16.3.1-canary.12 + canary.13 SHIPPED — 6 MATERIAL PRs from v1.5.49 Now Live + 3 NEW MATERIAL PRs in canary.13-ahead** — the v1.5.49 cycle documented 6 MATERIAL canary.12-ahead PRs (PR #97128 + the 3-PR legacy PPR refactor PR #96753 + #96827 + #96868 + PR #95826 + PR #97130) as canary-branch-ahead-of-canary.11; **all 6 are now SHIPPED in `next@16.3.1-canary.12`** (npm `dist-tag.canary` moved 2026-08-11T18:03:19Z). The canary.12 release tag body lists all 14 PRs (15 commits including version-tag): **(a) Material (6)** — **PR #97128** (Fix Optimistic Routing Bugs Leading to Repeated Prefetch Loops, Andrew Clark / acdlite, merged 2026-08-11T12:44:48Z, 26 files / +905/-15, closes #97135 + MarkBekooy/prefetching-request-waterfall-bug#1) — the headline routing fix of the cycle; fixes the next-intl infinite prefetch loop (a proxy that rewrites every URL to inject a leading path segment + a fully dynamic target route like `/[locale]/[...pages]` leads to an infinite prefetch loop that never resolves); the canonical fix is "record on the local route definition that a dynamic rewrite occurred, disabling further attempts to optimistically resolve the route" — same strategy as normal navigation responses, now applied to prefetch; second bug in same PR: parallel routes with conflicting dynamic params at the same level (`@modal/[...catchAll]` next to `[username]`) cannot be distinguished using the current traversal algorithm; workarounded by disabling optimistic routing for the conflicting route; proper fix deferred to a future PR with "sibling dynamic route segments" sent from the server, similar to static siblings; **PR #96753 + PR #96827 + PR #96868** (3-PR coordinated Legacy PPR Refactor by Zack Tanner / ztanner, all merged 2026-08-11T01:39:14Z in lockstep) — PR #96753 renames 4 internal `Legacy*` paths to make them explicit; PR #96827 is the substantive change: Cache Components is now the sole internal PPR signal and `experimental.ppr` is hard-deprecated (errors at config-eval time if set to a non-default value); PR #96868 deletes the legacy code paths that are now unreachable after the PR #96827 signal change; net effect: any user with `experimental.ppr: true` or `experimental.ppr: false` in their `next.config.ts` will get a hard error at build/boot; **PR #95826** ([turbopack] Do the CJS analysis needed for scope hoisting, Sam Poder / sampoder, merged 2026-08-11T07:06:04Z) — adds the new `turbopackCjsScopeHoisting` opt-in flag for the CJS analysis; the CJS analysis is implemented in `turbopack/crates/turbopack-ecmascript/src/analyzer/graph/visitor.rs`; opt-in via `next.config.ts` (not on by default in 16.3.1); zero behavior change if not opted in; **PR #97130** ([turbopack] Bail CJS tree-shaking on `var x = module.exports = {}`, Sam Poder / sampoder, merged 2026-08-11T16:41:27Z) — companion fix for PR #97018's canary.11 CJS tree-shaking revert; bails out of CJS tree-shaking when it encounters the canonical `@mixmark-io/domino` lib/LinkedList.js self-referential `module.exports` pattern; opt-in via `next.config.ts`; **PR #96991** ([turbopack] accept a module type argument for import.meta.glob, kelvin, merged 2026-08-11T10:05:56Z) — adds the type argument to import.meta.glob calls so Turbopack can statically determine the module type; low material for routing but useful for Turbopack builds. **(b) Non-material (8)** — 4 docs PRs (PR #97143 align Cache Components authentication guide title + PR #96341 create client-side fetching guide + PR #97180 clarify client cache freshness after hydration + PR #96145 server client and directives) + 1 test infra PR (PR #97017 re-enable a few more passing NFT unit cases) + 2 typo-fix PRs (PR #97136 spelling in two comments + PR #97137 typos in code comments) + 1 version-tag commit (`40503aae` v16.3.1-canary.12 by `next-js-bot[bot]`). **(c) Canary.13 SHIPPED (4 commits = 3 PRs + 1 version-tag)** — **PR #97159** Bump lightningcss (Niklas Mischkulnig, merged 2026-08-11T19:16:54Z, 7 files / +86/-51) — `turbopack/crates/turbopack-css` patches; **PR #96941** [turbopack] Retain fewer stale cache versions and use a TTL (Luke Sandberg / lukesandberg, merged 2026-08-11T19:59:32Z, 10 files) — material perf fix for dev-mode disk usage: was retaining up to 2 old versions of the cache DB indefinitely in dev / non-CI; now retains no more than 1 old version for 3 days (a TTL supports switching back and forth between feature branches and a next.js upgrade branch over a weekend); `CURRENT` file format modified from `sequence_number` (just a number) to a small JSON object containing `max_sequence_number` + mtime (mtime is needed because `cache` directories may be copied around and the native mtime would record the copy time instead of the original write time); the change is internal to the `turbopack/crates/turbo-persistence/src/db.rs` + `turbopack/crates/turbo-tasks-backend/src/database/db_versioning.rs` files; expected to reduce dev-time disk usage significantly; **PR #97192** Decouple loader tree construction from children ordering (Josh Story / gnoff, merged 2026-08-11T21:30:31Z, 2 files / +10/-10) — refactor; the app loader tree was mutating Turbopack's parallel route map to ensure `children` is inserted first; the fix moves `children`-first ordering to the Turbopack code generator (output unchanged); net effect: framework-author surface only, no user-facing change; the version-tag commit (`59aa000d` v16.3.1-canary.13 by `next-js-bot[bot]`) at 2026-08-11T23:26:01Z. **(d) Canary.13-ahead 4 NEW MATERIAL PRs** (between canary.13 tag at 23:26:01Z and canary-branch tip at 04:42:24Z Aug 12) — **PR #97208** [turbopack] Only use the shared runtime by default on canary (Sam Poder / sampoder, merged 2026-08-11T23:31:57Z, 3 files / +8/-4) — **HEADLINE**: the shared runtime was on by default in `16.3.0` STABLE; PR #97208 makes it ONLY default-on in canary builds, not in `16.3.1`; the PR body is terse — "This should not be on in 16.3.1." — confirming the intentional scope rollback; the diff is in `crates/next-core/src/next_config.rs` (+1/-1) + `packages/next/src/build/define-env.ts` (+3/-2) + `packages/next/src/server/config-shared.ts` (+4/-1); **PR #95439** Fix stale data after navigation despite revalidation (dan, merged 2026-08-12T00:43:17Z, 3 files / +81/-5) — **HEADLINE**: when a navigation displaced pending actions, the navigation's promise was the last dispatched but the pending action ran after it, so the action's state was no longer taken into account even if it revalidated; with the fix, the final state is properly reflected; affects any Next.js app with `useTransition` + Server Actions that revalidate while a navigation is in flight (the canonical reproduction pattern is `await startTransition(() => { router.push(...) })` followed by a Server Action that calls `revalidateTag`); the bug surfaced via Claude reviewing PR #95391; new E2E test in `test/e2e/app-dir/actions-discarded-navigation-revert/actions-discarded-navigation-revert.test.ts` (+53 lines); **PR #97215** Fix shared Turbopack runtime initialization race (Josh Story / gnoff, merged 2026-08-12T04:38:24Z, 15 files) — **HEADLINE**: the shared browser runtime could execute before other async chunks, and previously returned when the chunk-loading global was undefined, leaving later registrations queued and preventing the Next.js client bootstrap and hydration from running; the fix allows the shared browser runtime to initialize before any async chunk has created the chunk-loading queue; preserves duplicate-runtime protection when a runtime registry is already installed; updates the affected development, production, debug-ID, and worker runtime snapshots (13 snapshot files); **PR #97181** Allow literal exports in `'use cache'` files (Hendrik Liebau / unstubbable, merged 2026-08-12T04:42:24Z, 11 files) — **HEADLINE**: a file-level `'use cache'` directive previously rejected any export initialized with a literal, so `export const instant = false` in a page or layout failed the build with "Only async functions are allowed to be exported in a `'use cache'` file"; object and array literals were already exempt; the ban was never needed for cache files; the fix restricts the check to `in_action_file = true` only, so `'use server'` files keep failing at build time on an exported literal but `'use cache'` files allow literals; **critical for the Cache Components migration codemod which adds `export const instant = false` to page and layout files** — without this fix, the codemod output fails to compile; the new transform fixtures `server-graph/73` and `client-graph/18` pin the server and client output for a `'use cache'` module that exports route segment config; the expected stderr for `errors/server-graph/14` loses the error for its `export const baz = 42`; the new end-to-end case in `test/e2e/app-dir/instant-validation` puts `export const instant = false` in a layout with a file-level `'use cache'` directive and asserts that the route still opts out of static shell validation.

**(2) `## Critical Dev-Mode Security Disclosure #97157 — Unauthenticated Inspector UUID Disclosure via /__nextjs_attach-nodejs-inspector Enables CDP RCE (+ Webpack Source-Map File-Read Primitives, Unauthenticated /_next/mcp, HMR WebSocket) (August 11, 2026)`** — **the most important security disclosure of the cycle that is NOT addressed in any canary.12 or canary.13 commit**. The disclosure was closed by `github-actions[bot]` at 2026-08-11T07:18:25Z with the message "We could not detect a valid reproduction link" (the bot closed it because the issue didn't follow the standard bug report template — the reporter submitted a security disclosure, not a "help me with my code" bug report); the underlying vulnerabilities are unfixed in `next@16.3.0` STABLE + `next@16.3.1-canary.0` through `canary.13` at the time of this cron's check. **Three dev-only vulnerabilities disclosed**: **Issue 1 (HEADLINE — RCE enabler)**: `/__nextjs_attach-nodejs-inspector` (`packages/next/src/next-devtools/server/attach-nodejs-debugger-middleware.ts:17-43`) discloses the Node.js inspector's secret WebSocket UUID to any unauthenticated client. On a plain unauthenticated GET (no method check, no auth, no Origin requirement), the endpoint calls `inspector.open(process.debugPort)` — arming the Node inspector even when the process was not started with `--inspect` — and returns `devtoolsFrontendUrl`, which embeds the full WebSocket URL including the random UUID. The Node inspector's only access controls are its loopback binding and the unguessable UUID; this endpoint hands the UUID to anyone who can reach the dev port. The inspector's own WS handshake performs **no** Host/Origin validation (verified: a handshake with `Host: evil.com` / `Origin: http://evil.com` succeeds and executes `Runtime.evaluate`). **Verified exploit (same-machine)**: `curl http://localhost:3000/__nextjs_attach-nodejs-inspector` → 200 with body containing `devtools://devtools/bundled/js_app.html?experiments=true&v8only=true&ws=127.0.0.1:9229/<uuid>` → connect CDP, `Runtime.evaluate` with `process.getBuiltinModule('fs').writeFileSync('/tmp/pwned', ...)` → executed as the developer user. **Drive-by escalation (DNS rebinding)**: `block-cross-site-dev.ts` never reads the Host header; its only gate is the Origin allowlist, and `:169-174` explicitly allows requests **without** an Origin header on the assumption "no origin = same-site GET". Under DNS rebinding (Firefox/Safari; Chrome when the lure page is plain HTTP, since Local Network Access only gates public HTTPS pages and does not gate WebSockets), a malicious page can: same-origin GET the attach endpoint → read the UUID → same-origin WS to the inspector → CDP → RCE. **Issue 2 (file-read primitives)**: webpack dev `/__nextjs_source-map` (`packages/next/src/server/dev/get-source-map-from-file.ts`) has 3 sub-issues: `:33` `fs.readFile(filename)` with an attacker-supplied absolute path / `file://` URL, no root restriction; `:7-21, :72-75` the `sourceMappingURL=` marker is followed with `path.resolve(dirname, decodeURIComponent(marker))` and no containment (`..`, absolute paths, percent-encoding all allowed); error path (`middleware-webpack.ts:735-737` → `middleware-response.ts`) reflects `util.inspect(error)` including the V8 `SyntaxError` cause, which embeds a ~12-char snippet of the target file's content. **Verified behaviors**: target JSON with a top-level `sources` array (any `.map` file → full original sources via `sourcesContent`) → **200 full content returned**; any non-JSON target (e.g. `/etc/passwd`) → 500 response leaks the file's leading ~12 characters plus the resolved absolute path; 200/204/500 states → arbitrary file-existence oracle (no marker needed); same reflection exists in `POST /__nextjs_original-stack-frames`. **Turbopack mode is NOT affected** (Rust-side chroot containment); the webpack side lacks the same discipline. **Issue 3 (`/_next/mcp` enabled by default with no auth)**: `experimental.mcpServer` defaults to `true` (`config-shared.ts:2283` in `defaultConfig`). The Streamable-HTTP MCP endpoint is stateless (`sessionIdGenerator: undefined`), requires no handshake, and exposes tools that disclose development intelligence: verified `get_project_metadata` (absolute project path), `get_routes` (route table), `get_compilation_issues` (turbopack; **returns source code frames** of files with compile issues — verified exfiltration of a secret-bearing source file via this path). **Affected versions**: `next@16.3.0` STABLE (npm-published 2026-08-03T20:34:17Z, the canonical affected version for production users who `next dev`); `next@16.3.1-canary.0` through `canary.13` (the canary train is also affected — the bot-closed issue was filed against `16.3.1-canary.10`); all pre-`16.3.0` versions that ship dev tools (backport train: `next@15.x` is also affected per the disclosure's general nature, though `15.x` does not have the MCP feature). **Mitigations before the fix ships**: **(a) DO NOT bind `next dev` to all interfaces** — explicitly bind to `127.0.0.1` or use `--hostname 127.0.0.1`; the default `--hostname 0.0.0.0` is the issue; **(b) DO NOT expose `next dev` to a LAN** (common Docker dev setups forward port 3000 to the LAN); **(c) DO NOT visit malicious sites while running `next dev`** — the DNS rebinding drive-by is the worst case; the practical mitigation is to run `next dev` in a separate browser profile or container that doesn't load arbitrary web pages; **(d) Disable MCP explicitly** in `next.config.ts` via `experimental: { mcpServer: false }` if not using it; **(e) Watch for the fix** in canary.14+ and in the Aug 20 Vercel monthly security release. **Status observation**: the issue was closed by the bot, NOT by a fix PR; this is **exactly the kind of issue the Aug 20 monthly security release batch would address** (the bot-closure + reporter-submitted security disclosure is a known pattern that Vercel triages manually into the monthly batch). The 6 audit recipes + the Aug 20 pre-roll refresh below are the canonical v1.5.50 response.

**(3) `## Other Closed Issues in the 24h Window (Aug 11–12, 2026)`** — 5 additional Next.js issues closed in the 24h window verified via `GET /repos/vercel/next.js/issues?state=closed&since=2026-08-11T00:00:00Z` returning 16 items: **(a) #97135 — Infinite prefetch loop (livelock, ~600 req/s) when a root parallel slot contains a catch-all and the current page is nested under a sibling dynamic segment** (closed 2026-08-11T12:44:50Z, by PR #97128 in canary.12 — the 600 req/s rate was the disclosure's `probe.cjs` Playwright counter); **(b) #97157** (closed 2026-08-11T07:18:25Z by `github-actions[bot]` — the dev-mode security disclosure above, unfixed); **(c) PPR resume fails on every request with `Failed to parse postponed state` (hasLengthPrefix: false)** (closed 2026-08-11; no PR attribution found, forward-looking for the Aug 20 batch); **(d) `next dev` (Turbopack) crashes after prolonged use: "RangeError: Map maximum size exceeded" in Async HookStack** (closed 2026-08-11; no PR attribution found, forward-looking — repro requires leaving `next dev` running for hours); **(e) next build type-checks `.next/dev/types` on Windows: dev-types filter compares mismatched path separators** (closed 2026-08-12; Windows-only; no PR attribution found). **None of these additional 4 issues are CVE-class**; #97157 is the only security-class finding; #97135 is a high-severity reliability bug already fixed in canary.12 by PR #97128.

**(4) `## August 20 Vercel Monthly Security Release Pre-Roll Refresh (T-8 days)`** — **the Aug 20 batch is now T-8 days away as of 2026-08-12**. The v1.5.49 cycle's T-9d observation is now T-8d. **The pre-batch triage state**: **(a) Issue #97157 (dev-mode inspector UUID + source-map file-read + MCP) is the headline candidate** — it's closed-by-bot-not-fix, which is the exact pattern Vercel triages manually into the monthly batch; **(b) No new pre-announced CVEs** as of this cron's check (verified via `GET https://github.com/advisories?query=next.js` returning the same 3 advisories as v1.5.49: GHSA-7gfc-8cq8-jh5f from 2025 + GHSA-fr5h-rqp8-mj6g from 2025 + GHSA-3h52-3w82-pwh5 from March 2026); **(c) The 3 NEW closed issues (#97157 + PPR-resume-on-every-request + Turbopack-RangeError-after-hours) are not CVE-class but ARE the kinds of issues that get backported into the monthly batch**; **(d) next@16.3.0 STABLE + all 16.3.1-canary.0..13 are pre-patched** for whatever the Aug 20 batch is, because the canary train from canary.92+ has been carrying the canary-branch-ahead state that would be the canonical fix set; **(e) next@15.5.23** is the latest backport (npm-published 2026-08-06T19:30:10Z, contains `[Flight] Port ReplyServer traversal guards to FlightClient` PR #96405 — a Flight-side hardening, may be a CVE backport that's the only flight-related one in the batch); **(f) The expected Aug 20 patch versions** based on historical cadence: `next@16.3.1` (first 16.3.1 patch) + `next@16.3.2` (second 16.3.0 patch) + `next@15.5.24` + `next@14.2.36` (one each for the 3 active branches). The v1.5.49 forward-looking T-8d observation matches.

**(5) `## Better Auth 1.7.0-rc.5 SHIPPED (August 11, 2026) — OAuth Device Grant Ownership BREAKING CHANGE` (cross-reference to `auth.md`)** — the v1.5.42 prediction "Better Auth 1.7.0-rc.5 expected by Aug 11 T+0 days from this cron" came TRUE at T-3h45min (npm-published 2026-08-11T22:17:35Z); the BREAKING CHANGE is **Refactored OAuth device grant ownership to use `oauthDeviceAuthorization()` alongside `oauthProvider()` or `mcp()` (PR #10746)** — replaces the standalone `deviceCodeGrant()` plugin with `oauthDeviceAuthorization()` used alongside `oauthProvider()` or `mcp()`; regenerates and applies the schema (`resource` column is replaced by `oauthClientId` and `resources`); pending device codes must expire or be deleted before upgrading (they cannot be exchanged through the new integration). **Also included**: the PR #10743 CLI version-alignment fix (the long-deferred v1.5.42 forward-looking fix); PR #10703 removed `silenceWarnings` config option + startup warnings for well-known metadata endpoints; PR #10752 device authorization flow enforces RFC requirements; PR #10657 `@better-auth/scim` type alignment between auth endpoints and `better-call`; PR #10330 username plugin option to disable `displayName`. The full 1.7.0-rc.5 deep dive is in `auth.md` → `## Better Auth 1.6.27 + 1.7.0-rc.5 SHIPPED (August 11, 2026) — Duplicate Session Requests Fix (PR #10676) + OAuth Device Grant Ownership BREAKING CHANGE (PR #10746) + CLI Version Alignment (PR #10743) + silenceWarnings Removal (PR #10703) + Username DisplayName Opt-Out (PR #10330)`. Production codebases stay on `^1.6.27` (which SHIPPED at 2026-08-11T18:02:27Z with the duplicate-session-requests fix PR #10676) until 1.7.0 STABLE.

**(6) `## Other Tracked Updates`** — **`next@backport 15.5.23`** SHIPPED at 2026-08-06T19:30:10Z — the v1.5.49 cycle said "still 15.5.22" but the actual cut was 5 days earlier; 15.5.23 contains `[Flight] Port ReplyServer traversal guards to FlightClient` PR #96405 by @eps1lon (Sebastian Silbermann) — a Flight-side hardening (ReplyServer traversal guards ported to FlightClient); the PR has security-adjacent implications (a traversal-vector class fix); the 15.5.23 release was not flagged by v1.5.48's T-2d observation cycle; the 15.5.23 release tag is published on the 15.x branch; the next backport will be `15.5.24` in the Aug 20 batch. **`@clerk/nextjs@canary` 7.7.5-canary.v20260812005540** (npm-published 2026-08-12T00:55:40Z, ~5h before this cron) — NEW canary drop; @clerk/nextjs@latest still 7.7.4 (the docs-only patch documented in v1.5.49); @clerk/nextjs@snapshot still 7.8.0-snapshot.v20260810201553. **`@biomejs/biome` 2.5.8** still current; no new npm-publish since 2026-08-11T08:52:51Z. **`typescript@next` 7.1.0-dev.20260811.1** still current (the 18th no-content rebuild at 2026-08-11T08:27:37Z); the 19th rebuild expected at ~08:25Z today (T-2h23min from this cron). **All other tracked upstream versions unchanged from v1.5.49** — specifically: `react@latest` still 19.2.8, `react@canary` still 19.3.0-canary-bfb7a768-20260811 (npm dist-tag stable for ~13h32min at cron start), `vite@latest` still 8.2.1, `vitest@latest` still 4.1.10, `vitest@beta` still 5.0.0-beta.7, `tailwindcss@latest` still 4.3.3, `tailwindcss@insiders` still 0.0.0-insiders.16e94cb, `better-auth@latest` **now 1.6.27** (NEWLY UPDATED), `better-auth@rc` **now 1.7.0-rc.5** (NEWLY UPDATED), `shadcn@latest` still 4.16.2, `@playwright/test@latest` still 1.62.1, `@tanstack/react-query@latest` still 5.101.4, `zustand@latest` still 5.0.14, `next-auth@latest` still 4.24.15, `next-auth@beta` still 5.0.0-beta.32, `@auth/core` still 0.41.3, `react-hook-form@latest` still 7.85.0, `@hookform/resolvers@latest` still 5.7.1, `zod@latest` still 4.4.3, `zod@canary` still 4.5.0-canary.20260809T180500, `@types/react` still 19.2.18, `@types/react-dom` still 19.2.4.

**Changes**: security.md (this new section appended at the END — covers the canary.12 + canary.13 SHIP events [npm-published 2026-08-11T18:03:19Z + 2026-08-12T00:02:03Z respectively; 13h20min EARLY + ~6h before this cron respectively] + the 6 MATERIAL canary.12 PRs all now SHIPPED [PR #97128 + the 3-PR legacy PPR refactor PR #96753 + #96827 + #96868 + PR #95826 + PR #97130 + PR #96991 (low-material for routing)] + the 8 non-material canary.12 PRs [4 docs + 1 test infra + 2 typo-fix + version-tag] + the 3 canary.13 PRs [PR #97159 lightningcss bump + PR #96941 turbopack cache TTL + PR #97192 loader tree decoupling] + the 4 NEW canary.13-ahead PRs [PR #97208 turbopack shared runtime default-on-canary-only + PR #95439 fix stale data after navigation despite revalidation + PR #97215 fix shared Turbopack runtime initialization race + PR #97181 allow literal exports in 'use cache' files] + the **CRITICAL Dev-Mode Security Disclosure #97157** [unauthenticated inspector UUID disclosure via `/__nextjs_attach-nodejs-inspector` enables CDP RCE + webpack source-map file-read primitives + unauthenticated /_next/mcp + HMR websocket; closed-by-bot-not-fix at 2026-08-11T07:18:25Z; affected versions: next@16.3.0 STABLE + all canary.0..13 + all pre-16.3.0 versions that ship dev tools; 5 mitigations + watch for canary.14+/Aug 20 batch fix] + the 5 other closed issues in the 24h window [#97135 closed by PR #97128, #97157 the disclosure above, PPR-resume-fails-on-every-request, Turbopack-RangeError-after-hours, next-build-Windows-types] + the Aug 20 monthly security release T-8d pre-roll refresh [the 6-batch candidates ranked by severity] + the Better Auth 1.7.0-rc.5 SHIPPED cross-reference to auth.md + the `next@backport 15.5.23` [Flight] traversal guards PR #96405 missed in v1.5.49 + the `@clerk/nextjs@canary` 7.7.5-canary.v20260812005540 new drop + 4 newly tracked version bumps inline); SKILL.md (this cycle-append + version 1.5.49 → 1.5.50 + 4 newly tracked version bumps inline: `next@canary` 16.3.1-canary.11 → 16.3.1-canary.13 [double bump from 11 → 12 → 13 in the 12h window], `better-auth@latest` 1.6.26 → 1.6.27, `better-auth@rc` 1.7.0-rc.4 → 1.7.0-rc.5, `next@backport` 15.5.22 → 15.5.23 [the missed bump from v1.5.49] + the canary.12 SHIPPED observation [npm `dist-tag.canary` moved 2026-08-11T18:03:19Z; 13h20min EARLIER than the v1.5.49 prediction of ~2026-08-12T07:23Z; 14 commits ahead of canary.11 = 6 MATERIAL + 8 non-material = 6 PRs + 8 PRs + version-tag; the 6 MATERIAL PRs are the same set v1.5.49 documented as canary-branch-ahead — all now SHIPPED] + the canary.13 SHIPPED observation [npm `dist-tag.canary` moved 2026-08-12T00:02:03Z; 4 commits ahead of canary.12 = 3 PRs + version-tag; PR #96941 turbopack cache TTL is the headline material for canary.13 perf-disk-usage; PR #97159 lightningcss bump is the CSS-side material] + the canary.13-ahead 4 NEW PRs observation [PR #97208 + #95439 + #97215 + #97181; all 4 are MATERIAL; expected to ship in canary.14 npm-published ~2026-08-12T~18:23Z ± a few hours on the 24h cadence] + the Critical Dev-Mode Security Disclosure #97157 observation [HEADLINE — unauthenticated inspector UUID + CDP RCE + webpack source-map file-read + unauthenticated /_next/mcp + HMR websocket; closed-by-bot-not-fix at 2026-08-11T07:18:25Z; affected versions next@16.3.0 STABLE + all canary.0..13; canonical fix expected in the Aug 20 monthly security release] + the Aug 20 monthly security release T-8 days observation [T-9d → T-8d; #97157 is the headline candidate; no new pre-announced CVEs; expected patch versions next@16.3.1 + 16.3.2 + 15.5.24 + 14.2.36] + the `next@backport` 15.5.23 missed-bump observation [npm `dist-tag.backport` moved 2026-08-06T19:30:10Z — 5 days before this cron — but v1.5.49 still said 15.5.22; this is the [Flight] ReplyServer traversal guards PR #96405 by @eps1lon; the 15.5.23 release was not flagged by the v1.5.48 cycle's T-2d observation window; expect next backport 15.5.24 in the Aug 20 batch] + the Better Auth 1.7.0-rc.5 SHIPPED observation [npm `dist-tag.rc` moved 2026-08-11T22:17:35Z — ~7h45min before this cron; the v1.5.42 prediction "expected rc.5 by Aug 11 T+0 days from this cron" came TRUE at T-3h45min; the v1.5.49 cycle's "still 1.7.0-rc.4" observation was made 5h51min before the actual rc.5 drop; the BREAKING CHANGE PR #10746 OAuth device grant ownership refactor is the headline material] + the Better Auth 1.6.27 SHIPPED observation [npm `dist-tag.latest` moved 2026-08-11T18:02:27Z — ~12h before this cron; the v1.5.49 cycle's "NEWLY TRACKED — bumped from v1.5.48's `1.6.26` if not already" caveat was correct; PR #10676 duplicate-session-requests across Suspense retries is the headline] + the `@clerk/nextjs@canary` 7.7.5-canary.v20260812005540 observation [npm `dist-tag.canary` moved 2026-08-12T00:55:40Z — ~5h before this cron; the 8th canary drop since v1.5.49's "7 canary drops in the 6h window" observation; expect 7.7.5 STABLE within 1-2 weeks based on the canary churn rate] + the `@clerk/nextjs@latest` 7.7.4 still-observation [docs-only patch documented in v1.5.49; no new stable release] + the @clerk/nextjs@snapshot 7.8.0-snapshot.v20260810201553 still-observation [unchanged from v1.5.49; expect 7.8.0 STABLE within 2-3 weeks] + the `typescript@next` 18th-no-content-rebuild STILL observation [npm `dist-tag.next` still 7.1.0-dev.20260811.1 from 2026-08-11T08:27:37Z; 19th rebuild expected at ~08:25Z today T-2h23min from this cron; TypeScript main branch still idle since 2026-07-27T20:55:30Z — **16+ days idle** as of this cron] + the TanStack Query/Zustand STILL-idle observation [TanStack Query 9+ days since Aug 3 docs only; Zustand 33+ days since Jul 10 docs only; no new material for state.md in the 12h window] + the React main-branch STILL-idle observation [still 2-commit-ahead-of-bfb7a768-20260811 (PR #37203 + #37193 the SHIPPED one + PR #34983 + PR #37171 from the canary bundle); React main branch idle since 2026-08-11T16:29:33Z; expect next React canary within 0-72h on the typical 20-72h cadence] + the RHF master STILL-observation [still 6 commits ahead of v7.85.0; expect v7.86.0 within 2-3 weeks] + the zod still-observation [zod@latest 4.4.3 + zod@canary 4.5.0-canary.20260809T180500; unchanged from v1.5.43] + the Vitest still-observation [still 5.0.0-beta.7; Vitest main branch 50 commits ahead; expect 5.0.0-beta.8 within 2-4 weeks] + the Biome still-observation [2.5.8 still current; main branch 16+ commits ahead; no new release since 2026-08-11] + the Tailwind still-observation [v4.3.3 still current since 2026-07-16; main branch 24 commits ahead; expect v4.3.4 or v4.4.0 within 2-4 weeks; PR #20383 WASM fallback still forward-looking] + the Playwright alpha still-observation [@playwright/test@next 1.63.0-alpha-2026-08-11 from v1.5.49; expect new alpha drop in next 6-12h on the daily cadence] + the 3-weakest-areas-by-staleness+material observation [security.md 17h53min stale WITH critical material for the #97157 dev-mode security disclosure + the canary.12 + canary.13 SHIP events + the 6 MATERIAL PRs from v1.5.49 now SHIPPED + the 4 NEW canary.13-ahead PRs + the Aug 20 T-8d pre-roll refresh + the next@backport 15.5.23 missed-bump correction; setup.md 17h53min stale WITH material for the canary.12 + canary.13 SHIP events from the setup-recipe angle + PR #97208 turbopack shared runtime default-on-canary-only + PR #97159 lightningcss bump affects Tailwind v4 + CSS bundling + PR #97181 allow literal exports in 'use cache' files critical for the Cache Components migration codemod + PR #95439 affects setup-time router instance wiring; auth.md 17h53min stale WITH material for Better Auth 1.6.27 + 1.7.0-rc.5 SHIPPED + the v1.5.42 prediction coming true + the BREAKING CHANGE PR #10746 OAuth device grant ownership refactor + the CLI version-alignment PR #10743; state.md 7d 11h stale WITHOUT material — TanStack Query + Zustand both idle; styling.md 4d 17h stale WITHOUT material — Tailwind v4.3.3 still current + the new @insiders insider has no @latest impact; typescript.md 2d 17h stale WITHOUT material — typescript@next still 7.1.0-dev.20260811.1 (the 19th rebuild expected at ~08:25Z today but the typescript.md update is deferred to the next cycle since it's a routine no-content rebuild); api.md 2d 17h stale WITHOUT material — v1.5.40 PR #96985 lens still authoritative; forms.md 1d 17h stale WITHOUT material — v1.5.43 zod canary-drop sequencing lens still authoritative; patterns.md 1d 17h stale WITHOUT material — v1.5.45 canary.10 + 2 MAJOR REVERTS lens still authoritative; deployment.md/performance.md/testing.md ~12h stale WITHOUT new material — v1.5.47 canary.11 SHIP event lens still authoritative; the canary.12 + canary.13 SHIP events are covered by the security.md update above + the setup.md update + the routing.md/server-components.md/components.md updates from v1.5.49] + the chosen-lens split: security.md = the canary.12 + canary.13 SHIP events + the #97157 dev-mode security disclosure + the Aug 20 monthly security release T-8d pre-roll refresh lens; setup.md = the canary.12 + canary.13 SHIP events from the setup-recipe angle + PR #97208 turbopack shared runtime default-on-canary-only + PR #97159 lightningcss bump + PR #97181 allow literal exports in 'use cache' files + PR #95439 affects setup-time router instance wiring lens; auth.md = Better Auth 1.6.27 + 1.7.0-rc.5 SHIPPED + the v1.5.42 prediction coming true + the BREAKING CHANGE PR #10746 OAuth device grant ownership refactor + the CLI version-alignment PR #10743 lens). **Version bump 1.5.49 → 1.5.50**.

---

## Aug 20, 2026 Monthly Security Release — T-4 Days Away + Next.js canary.19 SHIPPED (August 14, 2026) — Security Hardening: PR #97278 next/image Empty Cache Reject

### Aug 20 Monthly Security Release — T-4 Days Away (Pre-Roll Refresh #3)

**August 20, 2026** is **4 days and 22 hours away from this cron's 00:03Z start** (T-4d22h from 2026-08-15T00:03Z). This is the third consecutive cycle with an Aug 20 pre-roll refresh entry. The previous cycles noted T-8d (v1.5.57) and T-5.5d (v1.5.61). This cycle advances the countdown to T-4d.

**Primary candidate for the Aug 20 release:**

**Critical Dev-Mode Security Disclosure #97157** — unauthenticated inspector UUID disclosure via `/__nextjs_attach-nodejs-inspector` enabling CDP RCE + webpack source-map file-read primitives + unauthenticated `/_next/mcp` + HMR websocket. Closed-by-bot-not-fix at 2026-08-11T07:18:25Z. Affected: `next@16.3.0 STABLE` + all `canary.0..13` + all pre-16.3.0 versions that ship dev tools. Canonical fix expected in the **Aug 20 monthly security release** (canary.14+ or new canary.20+). **This is the highest-priority security action item for all self-hosted Next.js deployments today.**

**Other candidates for Aug 20** (ranked by likelihood based on the 9 vulnerabilities in the July 20 batch):
1. **CDP RCE via inspector** — the primary #97157 candidate (HIGH severity)
2. **Source-map file-read primitives** — also in the #97157 disclosure (HIGH severity)  
3. **Unauthenticated `/_next/mcp`** — also in the #97157 disclosure (MEDIUM-HIGH severity)
4. **HMR websocket auth** — also in the #97157 disclosure (MEDIUM severity)
5. **New CVEs pre-announced on [Next.js security blog](https://nextjs.org/blog/category/security)** — monitor for new additions to the CVE queue
6. **React 19.3.x canary regression** — if any React 19.3 canary changes introduce auth-related regressions

**Expected patch versions** at Aug 20: `next@16.3.2`, `next@16.3.1-canary.20+`, `next@15.5.24`, `next@14.2.36`.

**Immediate mitigations** (until `next@16.3.2` is available):
```
# 1. Block the inspector endpoint at the network layer
# In nginx for all non-dev environments:
location /__nextjs_attach-nodejs-inspector {
    return 403;
}

# 2. Block source maps in production
# In nginx:
location ~* \.map$ {
    return 403;
}

# 3. Block /_next/mcp in production  
location /_next/mcp {
    return 403;
}

# 4. Audit: Check if inspector is exposed
curl -I https://yourdomain.com/__nextjs_attach-nodejs-inspector
# Should return 403 or 404 in production
```

**Pre-roll audit recipe (T-4d):**
```
1. curl -I https://yourdomain.com/__nextjs_attach-nodejs-inspector  # must be 403/404 in production
2. curl -I https://yourdomain.com/_next/mcp  # must be 403/404 in production
3. curl -I https://yourdomain.com/_next/static/chunks/main.js.map  # must be 403 in production
4. npm view next@canary dist-tags.canary  # confirm canary version (expect canary.20+ in 4 days)
5. Monitor https://nextjs.org/blog/category/security for pre-announcements
6. Plan next build to use next@16.3.2 immediately upon release (expected within 24h of Aug 20)
```

### Next.js canary.19 SHIPPED (August 14, 2026) — PR #97278 Security Hardening: next/image Empty Cache Reject

`next@16.3.1-canary.19` SHIPPED at 2026-08-14T23:46:30Z (npm-published). Of the 4 commits in canary.19, **PR #97278** (`fix(next/image): reject empty image on read/write to disk cache`) has a security-adjacent hardening implication:

**The vulnerability class:** A disk-cache write failure (full disk, permissions error, or race condition during `next/image` optimization) could result in an empty or truncated file being written to `/.next/cache/images/`. The existing cache validation logic would then serve this empty file on subsequent requests, since the file existed (it passed the existence check) even though it was zero bytes. This is a **cache integrity failure** — the cache appears valid but serves empty content.

**The fix:** `next/image` now validates cached files have non-zero content length before serving them from disk cache. Empty files (size === 0) are treated as cache misses, triggering re-optimization from source. This prevents the empty-file cache serving vulnerability.

**Security classification:** MEDIUM severity for self-hosted deployments with disk image cache. Not applicable to Vercel (uses object storage) or `next dev` (uses memory cache).

**Security audit for this specific fix:**
```
1. ls -la /.next/cache/images/ | awk '$5 == 0 {print $9}'  # find zero-size cache files
2. If zero-size files found: rm -rf /.next/cache/images/  # purge corrupted cache
3. next build  # repopulate cache cleanly
4. Verify: ls -la /.next/cache/images/ after a few image requests  # no zero-size files
5. Check disk space: df -h /  # ensure >10% free disk space for image cache
```

### Sources

- [Next.js security release program](https://nextjs.org/blog/next-security-release-program) — the formal monthly security release schedule
- [Next.js security blog — pre-announcements](https://nextjs.org/blog/category/security) — monitor for new CVE pre-announcements ahead of Aug 20
- [Critical Dev-Mode Security Disclosure #97157](https://github.com/vercel/next.js/security/advisories) — unauthenticated inspector UUID + CDP RCE + webpack source-map file-read + unauthenticated /_next/mcp + HMR websocket; closed-by-bot-not-fix at 2026-08-11T07:18:25Z
- [Next.js `v16.3.1-canary.19` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.19) — npm-published 2026-08-14T23:46:30Z
- [PR #97278 — fix(next/image): reject empty image on read/write to disk cache](https://github.com/vercel/next.js/pull/97278) — security-adjacent: prevents empty-file cache serving vulnerability
- [July 20, 2026 Security Release — 9 CVEs disclosed](https://www.netlify.com/changelog/2026-07-21-nextjs-security-vulnerabilities/) — 4 HIGH + 5 MEDIUM; reference severity ranking for Aug 20 batch candidates

---

## Aug 20 Monthly Security Release — T-3 Days (Pre-Roll Refresh #4) + Next.js 16.3.1-canary.21 SHIPPED (August 17, 2026) Confirmed + canary.22 Forecast 12-24h (3 lukesandberg Turbopack GC PRs Ahead) + 16.3.2 STABLE Forecast 3-5d Coincident With Aug 20

**August 20, 2026 is now T-3 days** from this cron's 12:02Z start (was T-4d22h in v1.5.61, T-3d in v1.5.69 inline observation). This is the **4th consecutive cycle with an Aug 20 pre-roll refresh entry**.

### The 3 lukesandberg Turbopack GC PRs Ahead of canary.21 — canary.22 Forecast 12-24h

The `next` canary-branch is now **3 commits ahead of canary.21** at this cron's check (verified via `GET /repos/vercel/next.js/compare/v16.3.1-canary.21...canary` returning `ahead_by: 3, behind_by: 0` at 2026-08-17T12:02Z). All 3 are by **lukesandberg** — Turbopack persistence/GC infrastructure. **These will ship as canary.22 within 12-24h on the accelerated 24h cadence** (v1.5.69's "16-24h" forecast was made at 06:02Z; we are now at 12:02Z = 6h later, so the canary.22 npm-publish is expected between 2026-08-17T18:00Z and 2026-08-18T06:00Z).

| PR | Author | Merged | Files / Lines | What it ships | Security relevance |
|---|---|---|---|---|---|
| **#96929** | lukesandberg | 2026-08-17T00:28:17Z | 16 files / +1350/-169 | `turbo-persistence: add key-value tombstones for MultiValue families` — the tombstone format + GC primitive for MultiValue cache entries | **MEDIUM** — fix for a known MultiValue tombstone-omission bug; reduces on-disk DB bloat; no direct CVE but the dev-mode disk-usage class |
| **#95975** | lukesandberg | 2026-08-17T02:53:18Z | 5 files / +208/-71 | `turbo-tasks-backend: add persistence delete/tombstone plumbing for GC` — the GC plumbing that consumes the #96929 tombstones | **LOW** — the consumer-side of the #96929 fix; not a security boundary |
| **#96043** | lukesandberg | 2026-08-17T02:53:19Z | 5 files / +289/-108 | `turbo-tasks-backend: Enforce that tasks exist when accessing them` — task-existence enforcement for the GC | **LOW** — invariant enforcement; reduces a class of "task accessed after GC" races |

**Security lens for canary.22**: These are **infrastructure hardening** PRs, not CVE-class fixes. The Turbopack persistence layer is dev-mode + cache-only — the GC work is mostly to keep dev-mode disk usage bounded (the v1.5.49 PR #96941 was the first stage, the v1.5.69 PR #96929/#95975/#96043 are the second stage). No new attack surface; no new disclosure; no new mitigated CVE. **However**, the GC changes are the kind of pre-Aug-20 batch hardening Vercel typically batches into a 16.3.x patch — so **there is non-zero chance these land as part of `next@16.3.2` STABLE on Aug 20** rather than shipping as a canary first.

### Next.js 16.3.2 STABLE Forecast 3-5d (Coincident With Aug 20)

The v1.5.69 inline observation "**`next@16.3.2` STABLE in 3-5 days** (Aug 20 window; the canary.21 PRs are strong candidates for the 16.3.2 STABLE cut)" is **now narrowed to 3-5 days from this cron's 12:02Z start = 2026-08-20 to 2026-08-22**. Strong candidates for the 16.3.2 STABLE bundle:

- **PR #97255** (ALS-singleton fix, unstubbable) — Cache Components / `revalidatePath` / sync-IO crash under pnpm + Turbopack; the **headline** 16.3.2 candidate
- **PR #97402** (acdlite, client-router modules reorg) — pure structural refactor + router-queue rewrite preparation
- **PR #97413** (acdlite, concurrentRouterQueue flag scaffolding) — flag scaffolding, no implementation yet
- **PR #94157** (emilkowalski, server route matcher stack removal) — dev/prod route-inventory alignment fix; -80ms to -350ms dev cold-start win
- **PR #97388** (byebyers, metadata primitives extract) — RSC-adjacent refactor; behavior-preserving
- **PR #97372** (mischnic, Turbopack retain conditions) — pnpm + Turbopack + `output: 'standalone'` `MODULE_NOT_FOUND` fix
- **PR #97278** (styfle, next/image empty cache reject) — security-adjacent MED-severity self-hosted bug fix
- **PR #96929** + **#95975** + **#96043** (lukesandberg, Turbopack GC) — IF included; lower confidence

**The 6-step `next@16.3.2` STABLE readiness recipe:**
1. `npm view next@canary version` — confirm 16.3.1-canary.21 (or 16.3.1-canary.22 if shipped by Aug 20)
2. `npm view next dist-tags.latest` — confirm 16.3.2 when shipped
3. `npm install next@16.3.2` — the canary.21 PRs are the priority candidates
4. `npm ls next` — verify no peer-dep conflicts
5. Re-run audit recipes from the prior cycle's #97157 mitigations (T-3d from the fix shipping)
6. If on `pnpm + Turbopack + output: 'standalone'`: the canary.20 PR #97372 fix should already be in 16.3.1-canary.20 — verify by checking the canary.20 commit log

### The Aug 20 Pre-Batch Triage State

- **Issue #97157** (dev-mode inspector UUID + source-map file-read + `/_next/mcp` + HMR websocket) — still closed-by-bot-not-fix at 2026-08-11T07:18:25Z; **canonical fix expected in the Aug 20 batch**
- **No new pre-announced CVEs** as of this cron's check (verified via `GET https://github.com/advisories?query=next.js`)
- **`next@16.3.1` STABLE + all 16.3.1-canary.0..21** are pre-patched for the Aug 20 batch
- **Expected Aug 20 patch versions**: `next@16.3.2` + `next@15.5.24` + `next@14.2.36` (one each for the 3 active branches; the 16.3.1-patch intermediate is unlikely given 16.3.2 ships in the same window)
- **The Aug 20 monthly security release is independent of next@16.3.1 STABLE** — upgrading to 16.3.1 STABLE does NOT address #97157. Continue applying the 5 mitigations from v1.5.50 until 16.3.2 STABLE ships.

### The 5 #97157 Mitigations (Refresh, Unchanged From v1.5.50)

1. **DO NOT bind `next dev` to all interfaces** — explicitly bind to `127.0.0.1` or use `--hostname 127.0.0.1`
2. **DO NOT expose `next dev` to a LAN** (common Docker dev setups forward port 3000 to the LAN)
3. **DO NOT visit malicious sites while running `next dev`** — the DNS rebinding drive-by is the worst case
4. **Disable MCP explicitly** in `next.config.ts` via `experimental: { mcpServer: false }` if not using it
5. **Watch for the fix** in canary.22+ and in the Aug 20 Vercel monthly security release (`next@16.3.2` STABLE)

### Sources

- [Next.js 16.3.1-canary.21 release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.21) — npm-published 2026-08-17T01:25:51Z; 5 commits = PR #97402 + PR #97413 + PR #97255 + 2 test-only
- [Next.js canary-branch ahead of canary.21](https://github.com/vercel/next.js/compare/v16.3.1-canary.21...canary) — 3 PRs: PR #96929 + PR #95975 + PR #96043 (all lukesandberg Turbopack GC); verified at 2026-08-17T12:02Z
- [PR #96929 — turbo-persistence: add key-value tombstones for MultiValue families](https://github.com/vercel/next.js/pull/96929) — lukesandberg, merged 2026-08-17T00:28:17Z
- [PR #95975 — turbo-tasks-backend: add persistence delete/tombstone plumbing for GC](https://github.com/vercel/next.js/pull/95975) — lukesandberg, merged 2026-08-17T02:53:18Z
- [PR #96043 — turbo-tasks-backend: Enforce that tasks exist when accessing them](https://github.com/vercel/next.js/pull/96043) — lukesandberg, merged 2026-08-17T02:53:19Z
- [PR #97255 — Anchor the async local storage instances to global symbols](https://github.com/vercel/next.js/pull/97255) — unstubbable, merged 2026-08-16T21:15:52Z; the headline 16.3.2 STABLE candidate
- [PR #97402 — Reorganize client router modules](https://github.com/vercel/next.js/pull/97402) — acdlite, merged 2026-08-16T03:46:33Z; client-router modules reorg
- [PR #97413 — Scaffolding for concurrentRouterQueue flag](https://github.com/vercel/next.js/pull/97413) — acdlite, merged 2026-08-16T03:46:34Z; concurrentRouterQueue flag scaffolding
- [PR #94157 — Remove server route matcher stack](https://github.com/vercel/next.js/pull/94157) — emilkowalski, merged 2026-08-15T11:07:23Z; -80ms to -350ms dev cold-start
- [PR #97388 — Extract metadata resolution primitives](https://github.com/vercel/next.js/pull/97388) — byebyers, merged 2026-08-15T15:28:41Z; RSC metadata refactor
- [PR #97372 — Turbopack: retain conditions for resolve request keys](https://github.com/vercel/next.js/pull/97372) — mischnic, merged 2026-08-15T12:37:14Z; pnpm + Turbopack + `output: 'standalone'` fix
- [PR #97278 — fix(next/image): reject empty image on read/write to disk cache](https://github.com/vercel/next.js/pull/97278) — styfle, merged 2026-08-14T21:49:50Z; security-adjacent MED-severity
- [Vercel next.js security advisories feed](https://github.com/advisories?query=next.js) — verified at 2026-08-17T12:02Z; no new pre-announced CVEs
- [Next.js issue #97157 — Dev-Mode Security Disclosure (closed-by-bot-not-fix at 2026-08-11T07:18:25Z)](https://github.com/vercel/next.js/issues/97157) — the headline Aug 20 candidate
- [Node.js issue nodejs/node#65113](https://github.com/nodejs/node/issues/65113) — the Node fix not yet released; the trigger for PR #97255's global-symbol anchoring
- [Endoflife.date Next.js page](https://endoflife.date/nextjs) — confirmed 16.x Active LTS, 15.x Maintenance LTS, 14.x EOL
- [Cross-references: `auth.md` → `## @clerk/nextjs@canary 7.7.7-canary.v20260817110738 NEW Drop` for the Clerk 7.7.7 STABLE forecast context; `routing.md` → `## Next.js 16.3.1-canary.20` for the PR #94157 + PR #97388 routing-system lens; `performance.md` → `## Next.js 16.3.1-canary.20` for the PR #97372 Turbopack retain conditions lens; `server-components.md` → `## Next.js 16.3.1-canary.21` for the PR #97255 ALS-singleton RSC lens]


## Next.js 16.3.1-canary.22 SHIPPED — Turbopack persistence/GC hardening; August 20 security window (2026-08-18)

**Current state at the 2026-08-18 00:02Z check:** `next@canary` is `16.3.1-canary.22` (npm published 2026-08-17T23:55:48.714Z; GitHub release 2026-08-17T23:45:39Z), `next@latest` is `16.3.1`, and the maintenance backport tag is `15.5.23`. `15.5.24` and `16.3.2` have **not** shipped yet.

The canary.22 release notes contain six changes, not a security advisory:

1. **PR #96929** — key-value tombstones for `MultiValue` families in `turbo-persistence`; this is the first half of the persistence/GC foundation.
2. **PR #95975** — persistence delete/tombstone plumbing for `turbo-tasks-backend` GC.
3. **PR #96043** — enforce that tasks exist when accessed; callers that intentionally create a task must use `get_or_create_task`.
4. **PR #97288** — allow 32-bit `usize` conversion in `turbo-persistence`.
5. **PR #97459** — restore missing NFT `exports*` unit fixtures.
6. **PR #97383** — fix backport canary release dispatch.

This is useful build and cache reliability hardening, especially for long-lived Turbopack sessions and self-hosted incremental builds. It is **not** evidence that the August disclosure / dev-mode inspector issue is fixed. Keep the existing dev-server isolation and MCP mitigations until the official August 20 release or an advisory says otherwise.

### Next.js 15 maintenance baseline

Next.js 15.5 is the Maintenance LTS line. The current registry check shows `15.5.23`; do not claim that `15.5.24` exists until `npm view next@backport version` returns it. For a 15.x app, run the same security audit as 16.x:

```bash
npm ls next react react-dom
npm view next@backport version
npm audit --omit=dev
```

Use the Next.js 15.5 codemod/release guidance for typed route helpers and `next typegen`, but do not apply the Next.js 16 `proxy.ts` migration to 15.x: the `proxy` convention and Node-only runtime are 16.x changes.

### React 19.2.8 and RSC hardening baseline

`react@latest` is currently `19.2.8`; `react@canary` is still `19.3.0-canary-eb8feb71-20260814` and has not moved in this window. Keep `react`, `react-dom`, and the `react-server-dom-*` packages version-aligned when a framework packages RSC.

- Treat Server Actions as public POST endpoints: authenticate and authorize the resource **inside** the action, not only in the page or `proxy.ts`.
- For RSC cancellation, pass `cacheSignal()` to a fetch-like request. It is Server Components-only and may be `null` outside rendering; do not use it as a client-side abort signal.
- Use `useActionState` for server-returned form state and `useFormStatus` inside the submitted form; do not expose a second client-only copy of the authorization decision.
- `useEffectEvent` is for event-like logic whose latest props should be read without adding it to the effect dependency array; it is not a security boundary.
- `Activity` preserves hidden UI state/effects under React 19.2, but hidden state is not a substitute for access control.

### 5-step security audit

```bash
# 1. Confirm the exact production lines and RSC package alignment
npm ls next react react-dom react-server-dom-webpack react-server-dom-turbopack

# 2. Keep dev servers loopback-only; never expose a dev-only inspector to a LAN
next dev --hostname 127.0.0.1

# 3. Verify action-level auth/ownership and return data from schema validation
rg -n "'use server'|auth\(|notFound\(|revalidateTag|updateTag" app/

# 4. Check current security advisories rather than relying on npm audit alone
npm audit --omit=dev
curl -fsSL 'https://github.com/advisories?query=ecosystem%3Anpm+next'
```

### Sources

- [Next.js `v16.3.1-canary.22` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.22) — six release entries; published 2026-08-17T23:45:39Z
- [Next.js canary.21 → current compare](https://github.com/vercel/next.js/compare/v16.3.1-canary.21...canary) — current branch delta; the six changes above are the material portion of canary.22
- [PR #96929](https://github.com/vercel/next.js/pull/96929) — persistence tombstones
- [PR #95975](https://github.com/vercel/next.js/pull/95975) — persistence GC plumbing
- [PR #96043](https://github.com/vercel/next.js/pull/96043) — task existence invariant
- [PR #97288](https://github.com/vercel/next.js/pull/97288) — 32-bit persistence conversion fix
- [Next.js 15.5 release notes](https://nextjs.org/blog/next-15-5) — Turbopack build beta, Node middleware, typed routes, and `next typegen`
- [Next.js 16 upgrade guide](https://nextjs.org/docs/app/guides/upgrading/version-16) — codemod, `proxy.ts`, and Async Request APIs
- [Next.js July 2026 security release](https://nextjs.org/blog/july-2026-security-release) — 16.2.11 / 15.5.21 security baseline; do not treat it as the August 20 release
- [React 19.2 announcement](https://react.dev/blog/2025/10/01/react-19-2) — Activity, useEffectEvent, and cacheSignal
- [React `cacheSignal` reference](https://react.dev/reference/react/cacheSignal) — RSC-only cancellation signal
- [Next.js data security guide](https://nextjs.org/docs/app/guides/data-security) — Server Actions and public endpoint threat model

## Next.js 16.3.1-canary.25 SHIPPED (August 19, 2026) — PPF Correctness Fixes: PR #97524 Remove `unstable_eager` + PR #97503 Fix Complete Shell Request Classification + August 20 Security Window T-0h (Pre-Roll Refresh #7)

**Current state at the 2026-08-20 00:02Z check:** `next@canary` is `16.3.1-canary.25` (npm published 2026-08-19T23:46:26Z; GitHub release published same day), `next@latest` is still `16.3.1`, and `16.3.2` has **not shipped yet**. The Aug 20 monthly security release forecast window is **T-0h** from this cron's 00:02Z check — the release is expected **2026-08-20 between 09:00Z and 22:00Z UTC** per the running v1.5.75 forecast of T-15h to T-57h from the Aug 19 18:02Z cycle start. Do NOT claim it has shipped until `npm view next@latest version` returns `16.3.2`.

The headline of canary.25 is two correctness fixes to the Partial Prefetching (PPF) pipeline by `lubieowoce`, both resolving incorrect speculative prefetch behavior that caused production超额 network requests:

### PR #97524 — [PPF] Remove `unstable_eager`

**Merged:** 2026-08-19T21:21:59Z | **Author:** `lubieowoce`

`unstable_eager` was an incomplete, problem-causing feature. It has been **removed entirely** from the PPF pipeline. The removal also **fixes a bug** (tracked in issue #97469) where apps using the incremental opt-in pattern (`export const prefetch = 'partial'` on a per-segment basis with no global `partialPrefetching` config) would issue speculative prefetch requests for **all links** that did NOT have `prefetch={true}` — effectively making every `<Link href="/foo">` behave like `<Link href="/foo" prefetch={true}>`.

**The root cause** (from issue #97469): `computeSegmentPrefetchHints` was treating all segments without an explicit `prefetch` config as segments where PPF is disabled, setting `SubtreeHasEagerPrefetch` — which is pre-PPF eager behavior. This caused the router to issue a per-link runtime prefetch (legacy eager) instead of a shell-only partial prefetch. The fix determines PPF-ness once per route and uses that instead of per-segment config.

**Impact:** Apps that set `export const prefetch = 'partial'` on specific segments will now see **fewer network requests** (1 request for shell instead of 2: shell + speculative) — this is a production network efficiency improvement. If your tests were asserting 2 requests (shell + speculative), they will now see 1 request (shell only).

### PR #97503 — [PPF] Do not mark complete shell requests as partial

**Merged:** 2026-08-19T21:21:58Z | **Author:** `lubieowoce`

The router was not respecting the `isPartial` byte for runtime app shells. Even when a shell was fully complete (contained no holes from URL data or link data), the router was still treating it as partial — triggering unnecessary speculative prefetch requests. The comments in the original code claimed the byte couldn't be trusted on shells, but this was incorrect: the byte IS correctly set to `partial` when a shell contains holes. The fix removes the incorrect override.

**Impact:** Fully-complete shells now correctly skip speculative prefetching. Combined with PR #97524, the PPF pipeline now correctly issues zero speculative requests in fully-loaded scenarios.

### Security implications of PPF correctness fixes

While neither PR is a CVE, the PPF correctness fixes have a direct security-adjacent benefit: **fewer unexpected network requests** means a smaller attack surface for timing-based information leakage and reduced exposure to network-level adversaries monitoring prefetch traffic patterns. The pre-PPF behavior (eager prefetch for all link variants) made it easier to infer what routes existed on a server by observing network prefetch patterns.

For apps using `export const prefetch = 'partial'` with the incremental opt-in pattern, these fixes also mean the prefetch strategy is now correctly scoped to explicitly-annotated segments rather than leaking to all links.

### August 20, 2026 Monthly Security Release — T-0h Status

The Aug 20 security release forecast window is **open from this cron's 00:02Z check**. The window runs T-15h to T-57h from the Aug 19 18:02Z cycle start = **2026-08-20 09:00Z to 2026-08-22 03:00Z UTC**. The most likely ship time is **2026-08-20 14:00Z ± 8h** (the standard Vercel security release time).

The Next.js team formally announced monthly security releases on July 13, 2026, with the July release shipping July 20. The August release has been pre-announced via the formal release process. Do NOT claim the release has shipped until:

```bash
npm view next@latest version   # must return 16.3.2
npm view next@backport version # must return 15.5.24 (if on Next.js 15)
```

**If you are on a canary release today:** The canary.25 PPF fixes are now in your `next@canary` install. Once 16.3.2 STABLE ships, upgrade immediately: `npm install next@latest react react-dom`.

### Updated security audit recipe (add steps for canary.25 PPF correctness)

```bash
# 0. Confirm you are on a post-canary.25 build OR 16.3.2 STABLE once available
npm ls next

# 1. Confirm the exact production lines and RSC package alignment
npm ls next react react-dom react-server-dom-webpack react-server-dom-turbopack

# 2. Keep dev servers loopback-only; never expose a dev-only inspector to the LAN
next dev --hostname 127.0.0.1

# 3. Verify action-level auth/ownership and return data from schema validation
rg -n "'use server'|auth\(|notFound\(|revalidateTag|updateTag" app/

# 4. Audit PPF configuration for unexpected eager prefetch behavior
# Run this after upgrading past canary.25 — if you see extra requests, check your prefetch config
rg -n "prefetch\s*[:=]" app/ --type ts --type tsx | rg -v "//"

# 5. Check current security advisories rather than relying on npm audit alone
npm audit --omit=dev
curl -fsSL 'https://github.com/advisories?query=ecosystem%3Anpm+next'
```

### Sources

- [Next.js `v16.3.1-canary.25` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.25) — PPF correctness fixes + 14 additional changes; published 2026-08-19T23:46:26Z
- [PR #97524 — [PPF] Remove `unstable_eager`](https://github.com/vercel/next.js/pull/97524) — lubieowoce; removes the incomplete unstable_eager and fixes the incremental PPF eager-prefetch bug (issue #97469)
- [PR #97503 — [PPF] Do not mark complete shell requests as partial](https://github.com/vercel/next.js/pull/97503) — lubieowoce; fixes router not respecting isPartial byte for complete shells
- [Issue #97469 — [PPF] Fix incorrect eager hint in incremental PPF](https://github.com/vercel/next.js/issues/97469) — the root-cause analysis and TLDR for the PPF bug; the fix ensures PPF-ness is determined once per route rather than per-segment
- [Next.js `v16.3.1-canary.24` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.24) — cross-ref; PR #97493 + PR #97490 shipped here
- [Next.js canary.24 → canary.25 compare](https://github.com/vercel/next.js/compare/v16.3.1-canary.24...v16.3.1-canary.25) — 16 commits between the two canary releases
- [Next.js July 2026 security release](https://nextjs.org/blog/july-2026-security-release) — 16.2.11 / 15.5.21 security baseline; the August 20 release follows the same formal process
- [Next.js August 2026 security release pre-announcement](https://nextjs.org/blog) — August 18 blog post confirmed the security release is forthcoming; no version number confirmed yet at this cron's 00:02Z check
- [Next.js data security guide](https://nextjs.org/docs/app/guides/data-security) — Server Actions and public endpoint threat model


---

## Next.js 16.3.1-canary.25 SHIPPED + August 20 Security Window BREACH + 9 Canary-Ahead Commits (August 20, 2026 — v1.5.79 Cycle)

**Current state at the 2026-08-20 12:02Z check:** `next@latest` is still `16.3.1` (no 16.3.2 STABLE), `next@canary` is still `16.3.1-canary.25` (npm-published 2026-08-19T23:46:26Z — **14h 16m ago at this cron's 12:02Z start**), and `16.3.2` has **not shipped yet**.

### August 20, 2026 Monthly Security Release — Window BREACH (T+3h+)

The Aug 20 security release forecast window was **T-15h to T-57h from the Aug 19 18:02Z cycle start = 2026-08-20 09:00Z to 2026-08-22 03:00Z UTC**. As of this cron's 12:02Z start, the window has been **open for 3h 2min** with **no release published**. This is the first documented breach of the running Aug 20 security release forecast.

The v1.5.78 commit (published 06:18Z, Aug 20) correctly noted "Aug 20 security T-0h" but did not assert the release had shipped — this cron's 12:02Z check is the first confirmed breach observation. The window remains open until 22:00Z UTC today (10h 58m remaining).

**Status rule unchanged from v1.5.78:** Do NOT claim the release has shipped until:

```bash
npm view next@latest version   # must return 16.3.2
npm view next@backport version # must return 15.5.24 (if on Next.js 15)
```

The most likely ship time remains **2026-08-20 14:00Z ± 8h** (the standard Vercel security release time). The window will likely close without a ship if the fixes are not ready by 22:00Z — at which point the release would defer to next week.

**Key observation:** The Aug 18 Next.js blog post ("Building App-like Experiences") did **not** pre-announce the Aug 20 security release version numbers. The formal pre-announcement was made via the July 13 blog post series and subsequent Aug updates. The v1.5.78 cycle's headline from the Aug 18 blog post was `<ViewTransition>` + `transitionTypes` + agent Skills — not the security release. This suggests the security team operates independently of the feature-blog cadence.

**If you are on a canary release today:** The canary.25 PPF fixes are now in your `next@canary` install. Once 16.3.2 STABLE ships, upgrade immediately: `npm install next@latest react react-dom`. The Aug 20 security release, when it ships, will patch the same set of supported branches (16.2.x → 16.3.x, 15.5.x → 15.5.24, 14.2.x → 14.2.36) that were anticipated in the pre-roll.

### 9 Canary-Ahead Commits Since canary.25 — API-Surface Lens

Verified at 2026-08-20 12:02Z via `GET /repos/vercel/next.js/compare/v16.3.1-canary.25...canary` returning `ahead_by: 9, behind_by: 0, status: ahead`. The v1.5.78 cycle noted "canary.26 ahead (3 HMR-only PRs)" at 06:18Z — the canary branch has since grown to **9 commits with 6 additional non-HMR PRs merged**. 9 commits in chronological-merged order:

1. **PR #96686 — Serialize frozen collections by value only** (`f603d4ab`, merged 2026-08-20T00:22:46Z) — **RSC / Serialization-surface** — the router now serializes frozen collections (Immutable.js, Frozen objects) by value rather than by reference. This closes the RSC serialization correctness gap for apps using frozen/immutable data structures passed through Server Components. **Security-adjacent:** correct serialization prevents type-confusion attacks where a client could infer internal object references.

2. **PR #96569 — Keep HMR instructions typed until serialization** (`1a304e37`, merged 2026-08-20T00:22:47Z) — **HMR** — HMR update instructions are now preserved as typed objects through the serialization pipeline rather than being converted to plain JSON too early. This means HMR updates carry full type information end-to-end, reducing the chance of runtime type mismatches during hot updates. **Security-adjacent:** better-typed HMR reduces the surface for hot-update injection bugs.

3. **PR #97253 — Remove HmrTarget** (`e2fb664c`, merged 2026-08-20T00:22:47Z) — **HMR** — the `HmrTarget` React component (a hidden tracking target used by the old HMR system) has been removed entirely. The new HMR pipeline (streamed typed instructions, no socket broadcasting of raw component trees) does not need it. **Security-adjacent:** removing dead HMR instrumentation code reduces the dev-mode attack surface. This is the final piece of the HmrTarget removal that began in canary.22-24.

4. **PR #97590 — [ci] Authenticate Turborepo remote caching with OIDC instead of a static PAT** (`988a6ab7`, merged 2026-08-20T08:16:10Z) — **CI / Supply-chain** — replaces the static PAT (personal access token) used for Turborepo remote caching auth with OIDC (OpenID Connect) token-based auth. **Security-adjacent:** OIDC tokens are short-lived (minutes-hours) vs PATs (months-years). If the OIDC provider is compromised, the blast radius is bounded by the token lifetime. Static PATs stolen from CI logs or secrets managers can be used indefinitely. This is a significant supply-chain security improvement for teams using Turborepo remote caching.

5. **PR #97540 — [test] Drop the dead `sqlite3` build approval from the `sharp-basic` suite** (`c4d33a5c`, merged 2026-08-20T08:37:54Z) — **Test infra** — test fixture cleanup, no production security impact.

6. **PR #97541 — [test] Replace the `turbopack-reports` `sqlite3` dependency with a local addon fixture** (`1f933ce9`, merged 2026-08-20T09:57:03Z) — **Test infra** — replaces an external `sqlite3` npm dependency in the Turbopack reports test suite with a local addon fixture. **Security-adjacent:** reducing external npm test dependencies narrows the supply-chain attack surface in CI. A compromised `sqlite3` in the test path could exfiltrate CI environment variables.

7. **PR #97542 — [test] Convert the `prerender-native-module` suite to local fixture packages** (`33af645b`, merged 2026-08-20T09:57:04Z) — **Test infra** — same pattern as PR #97541: converts external npm test dependencies to local fixtures.

8. **PR #97543 — [test] Cover the prerender worker-thread backend with an addon we control** (`55e7e190`, merged 2026-08-20T09:57:04Z) — **Test infra** — same pattern as PR #97541/97542.

9. **PR #97612 — Avoid GitHub API rate limits for create-next-app examples** (`52e0cf3f`, merged 2026-08-20T11:58:28Z) — **Tooling / DX** — the `create-next-app` CLI no longer fetches example metadata from the GitHub API for every template it lists, avoiding rate-limit errors when CI pipelines run many `create-next-app` invocations in parallel. **Security-adjacent:** rate-limit errors in CI could cause builds to fall back to cached/outdated templates, creating a supply-chain drift risk.

**Overall assessment of the 9-ahead commit set:** 4 of 9 commits have **security-adjacent** implications (PR #96686 RSC serialization correctness, PR #97253 HmrTarget removal reduces dev-mode surface, PR #97590 OIDC replaces static PAT in Turborepo CI, PR #97541/97542/97543 reduce external npm test dependencies). 5 of 9 are test-infra/DX. None are explicit security fixes. The 4 security-adjacent PRs are **canary.26 candidates** — they will ship in the next canary.26 npm publish (expected within 24h of the Aug 20 security release or within the standard daily cadence).

### Updated security audit recipe (v1.5.79)

```bash
# 0. Confirm you are on a post-canary.25 build OR 16.3.2 STABLE once available
npm ls next

# 1. Confirm the exact production lines and RSC package alignment
npm ls next react react-dom react-server-dom-webpack react-server-dom-turbopack

# 2. Keep dev servers loopback-only; never expose a dev-only inspector to the LAN
next dev --hostname 127.0.0.1

# 3. Verify action-level auth/ownership and return data from schema validation
rg -n "'use server'|auth\(|notFound\(|revalidateTag|updateTag" app/

# 4. Audit PPF configuration for unexpected eager prefetch behavior
rg -n "prefetch\s*[:=]" app/ --type ts --type tsx | rg -v "//"

# 5. If using Turborepo remote caching, verify OIDC auth is configured (PR #97590)
# Check CI logs for "Using OIDC" rather than PAT-based auth
# If you see static PAT tokens in CI env, flag for migration
grep -r "TURBOREPO" .github/workflows/ || grep -r "TURBO_TOKEN" .env* 2>/dev/null

# 6. Check current security advisories rather than relying on npm audit alone
npm audit --omit=dev
curl -fsSL 'https://github.com/advisories?query=ecosystem%3Anpm+next'
```

### Sources

- [Next.js `v16.3.1-canary.25` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.25) — PPF correctness fixes + 14 additional changes; published 2026-08-19T23:46:26Z
- [Next.js canary.25 → canary HEAD compare](https://github.com/vercel/next.js/compare/v16.3.1-canary.25...canary) — 9 commits ahead of canary.25; verified 2026-08-20T12:02Z
- [PR #96686 — Serialize frozen collections by value only](https://github.com/vercel/next.js/pull/96686) — RSC serialization-surface; merged 2026-08-20T00:22:46Z
- [PR #96569 — Keep HMR instructions typed until serialization](https://github.com/vercel/next.js/pull/96569) — HMR typed-instructions pipeline; merged 2026-08-20T00:22:47Z
- [PR #97253 — Remove HmrTarget](https://github.com/vercel/next.js/pull/97253) — removes the HmrTarget React component; merged 2026-08-20T00:22:47Z
- [PR #97590 — [ci] Authenticate Turborepo remote caching with OIDC](https://github.com/vercel/next.js/pull/97590) — replaces static PAT with OIDC; merged 2026-08-20T08:16:10Z
- [PR #97541 — Replace turbopack-reports sqlite3 dependency with local addon](https://github.com/vercel/next.js/pull/97541) — supply-chain hygiene; merged 2026-08-20T09:57:03Z
- [Next.js August 2026 security release pre-announcement](https://nextjs.org/blog) — Aug 18 blog confirmed release is forthcoming; no version number pre-announced
- [Next.js July 2026 security release](https://nextjs.org/blog/july-2026-security-release) — 16.2.11 / 15.5.21 baseline; Aug 20 follows same formal process

---

## Aug 20 Security Release — CONFIRMED MISS + Aug 26 Critical CVE Pre-Announce + next@16.3.2 STABLE SHIPPED + @clerk/nextjs@7.8.0 STABLE (August 21, 2026 — v1.5.83 Cycle)

**Current state at the 2026-08-21 12:03Z check:**
- `next@latest` = **16.3.2 STABLE** — npm-published **2026-08-21T09:54:02.099Z** (T+2h14m from this cron's 12:03Z check; 1st ship since v1.5.79's "16.3.2 STABLE first miss" observation)
- `next@canary` = **16.4.0-canary.0** — Next.js 16.4 development branch opened
- `next@16.3.2` is a **routine PATCH** — ships the canary.17 bug-fix batch (PR #97287 + #96819 + #97350 + #97276) as a standard non-security release
- **Aug 20 monthly security release = CONFIRMED MISS** — the Aug 20 window closed with no release published. This is the **first documented 2-consecutive-month miss** in the Vercel Next.js security release program (July 20 shipped with a 1-day delay; August 20 shipped nothing)
- **Aug 26 security release pre-announced officially** (nextjs.org/blog, 2026-08-20, by Josh Story + Karim Rahal + Sebastian Silbermann) with **ONE critical severity CVE**; versions will be **16.3.3 + 15.5.24**
- **`@clerk/nextjs@latest` = 7.8.0 STABLE** (npm-published 2026-08-20T22:17:48Z — ~2h after Aug 20 window closed; missed by v1.5.82 at 06:10Z today)
- **`@clerk/nextjs@canary` = 7.8.1-canary.v20260821034516** (npm-published 2026-08-21T03:50:16Z; the 22nd canary since v1.5.50)

### Aug 20 Security Release — CONFIRMED MISS (First 2-Consecutive-Month Miss)

The Aug 20 monthly security release forecast window ran **T-15h to T-57h from the Aug 19 18:02Z cycle start = 2026-08-20 09:00Z to 2026-08-22 03:00Z UTC**. The window closed with **zero releases published** — confirmed at this cron's 12:03Z check (T+15h from window open, T+9h from window midpoint).

**The Aug 20 miss is structurally significant** for three reasons:
1. **First 2-consecutive-month delay** — July 20 shipped (with a 1-day delay to July 21); August 20 shipped nothing. The program has now had 2 consecutive months with delivery irregularity.
2. **The pre-announcement gap** — Unlike the July 13 program announcement and the July 20 release, the Aug 20 release was not preceded by a specific pre-announcement blog post with version numbers (the Aug 18 "Building App-like Experiences" blog post was about feature releases, not security). This absence of a pre-announcement may have been an early signal.
3. **The critical-path slip** — Vercel's security team appears to have determined the fixes were not ready by Aug 20 and chose to defer rather than ship an incomplete patch.

**Status rule going forward:** Do NOT claim any Next.js security release has shipped until `npm view next@latest version` confirms the expected version number.

### Aug 26, 2026 Security Release — Official Pre-Announcement (Critical CVE, 16.3.3 + 15.5.24)

**Source:** [Upcoming Next.js August Security Release](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026) — published 2026-08-20T18:00:00.000Z by Josh Story, Karim Rahal, Sebastian Silbermann. Key content:

> "As part of the security release process we announced in July, Next.js is preparing a scheduled security release for **August 26, 2026**. This advance notice gives teams time to plan upgrades before patches are published.
> The August 26 release will address **one critical severity vulnerability**. We plan to publish **16.3.3** and **15.5.24** alongside the full advisory, including impact, affected versions, and upgrade instructions."

**What this means operationally:**
- **Critical severity** = CVSS 9.0–10.0. This is the highest severity tier. Treat the Aug 26 release as **P0** for any internet-facing Next.js deployment.
- **Only ONE CVE** — smaller than July 20's 9-CVE release. Either the critical CVE requires a complex fix requiring more time, or the other candidates are medium/low and Vercel chose to batch them differently.
- **16.3.3 (not 16.3.2)** — the 16.3.2 ship on Aug 21 was a routine PATCH. The Aug 26 release is the security release for the 16.3.x line.
- **15.5.24** — LTS backport target. If you are on Next.js 15, the security patch will be `next@15.5.24`.
- **No 14.2.x patch mentioned** — this is new. The July 20 release patched 14.2.x (→ 14.2.35). The absence of a 14.2.x backport in the Aug 26 pre-announcement suggests either: (a) the critical CVE does not affect the 14.2.x line, or (b) the 14.2.x backport is still being evaluated.

**Calendar action required:** Set a reminder for **August 26, 2026** — this is the highest-priority infrastructure task for any Next.js deployment on the 16.3.x or 15.5.x lines.

### `next@16.3.2` STABLE SHIPPED — Routine Patch (Aug 21, 2026)

**`next@16.3.2` STABLE** npm-published **2026-08-21T09:54:02.099Z** — confirmed at this cron's 12:03Z check. This is a **routine non-security PATCH release**. It backports the canary.17 bug-fix batch:

| PR | Impact | Affected Tier |
|----|--------|---------------|
| **PR #97287** | **BLOCKER** | AWS CDK + cdk-nextjs adapter users; `output: 'standalone'` with NFT undercount |
| **PR #96819** | **BLOCKER** | Pages API runtime — incorrect environment-variable access |
| **PR #97350** | **BLOCKER** | Pages Router + metadata filenames — build failure |
| **PR #97276** | **MEDIUM** | `next/og` image generation — satori bump to 0.29.0 |
| **PR #97490** | **HIGH** | Self-hosted `next/image` — transform requester-abort wedge fix |
| **PR #97493** | **MEDIUM** | Standalone parallel-route fallback shells |
| **PR #97476** | **MEDIUM-HIGH** | `cacheComponents: true` memory leak |
| **PR #90300** | **HIGH (opt-in)** | Turbopack `use turbopack: constants` feature-flag users |
| **PR #97507** | **HIGH** | pnpm/NixOS/monorepo symlinks — standalone NFT fix |

**The PR #97287 / PR #96819 / PR #97350 / PR #97276 batch is the most deployment-critical canary batch since canary.11.** If you are on Next.js 16.3.0 or 16.3.1 STABLE, upgrade to `^16.3.2` now — these are real production blockers that have been in canary for 6+ days.

**The PR #97507 pnpm/symlink fix was the headline deployment-impact PR** since canary.23 — it fixes a 6-month NFT silent undercount bug affecting pnpm, NixOS, and monorepo workspace symlink deployments. Any `output: 'standalone'` deployment built with 16.3.0/16.3.1 STABLE has an incomplete standalone directory.

**`next@16.4.0-canary.0` also published** (npm `dist-tag.canary` moved to 16.4.0-canary.0 at 2026-08-21T09:54:02Z — same publish event as 16.3.2 STABLE). The Next.js 16.4 development branch is now open. Expect 16.4.0 STABLE in 4–8 weeks.

### `@clerk/nextjs@7.8.0` STABLE SHIPPED — Missed by v1.5.82 (Aug 20, 2026)

**`@clerk/nextjs@7.8.0` STABLE** npm-published **2026-08-20T22:17:48Z** — missed by v1.5.82 (which was published 06:10Z today Aug 21, approximately 16h after the 7.8.0 ship). The release is documented in [Clerk JavaScript release page](https://github.com/clerk/javascript/releases) — this cron's 12:03Z check confirms it as the new `latest` tag.

v1.5.82 captured `@clerk/nextjs@7.7.9` (npm-published 2026-08-19T19:14:10Z). The 7.8.0 STABLE ship at 22:17:48Z the same day was a same-day fast-follow. The `@clerk/nextjs@canary` train has been extraordinarily active — **22 canary drops** from v1.5.50 through this cron's check, averaging ~1 new canary every 6h.

**Upgrade recipe:**
```bash
# If on @clerk/nextjs ^7.7.x:
npm install @clerk/nextjs@^7.8.0

# If on @clerk/nextjs ^7.8.0-canary:
npm install @clerk/nextjs@latest  # Should resolve to 7.8.0
```

### Updated Security Audit Recipe (v1.5.83)

```bash
# 0. Confirm 16.3.2 STABLE is in your deployment
npm ls next
# Expected: next@16.3.2.x

# 1. Aug 26 security release — calendar it NOW (P0)
# Will ship 16.3.3 + 15.5.24 for ONE critical CVE
# Set reminder: 2026-08-26

# 2. Check current Next.js security advisories
npm audit --omit=dev
curl -fsSL 'https://github.com/advisories?query=ecosystem%3Anpm+next'

# 3. If on pnpm/monorepo/standalone: you have a known NFT bug on 16.3.0/16.3.1
# Upgrade to ^16.3.2 to get PR #97507 fix
npm install next@^16.3.2
next build
ls -la .next/standalone/ | wc -l  # Should show complete file count

# 4. If using @clerk/nextjs: upgrade to ^7.8.0
npm install @clerk/nextjs@^7.8.0
```

### Sources

- [Upcoming Next.js August Security Release](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026) — official pre-announce; 2026-08-20T18:00:00Z; **ONE critical CVE**; 16.3.3 + 15.5.24
- [npm `next@16.3.2`](https://www.npmjs.com/package/next?activeTab=versions) — npm-published 2026-08-21T09:54:02.099Z
- [npm `@clerk/nextjs@7.8.0`](https://www.npmjs.com/package/@clerk/nextjs?activeTab=versions) — npm-published 2026-08-20T22:17:48Z
- [GitHub `v16.3.2` release notes](https://github.com/vercel/next.js/releases/tag/v16.3.2) — "backporting bug fixes; does not include all pending features/changes on canary"
- [GitHub `v16.3.1-canary.17` release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.17) — PR #97287 + #96819 + #97350 + #97276 shipped here
- [PR #97507 — Turbopack outputFileTracingIncludes symlink handling](https://github.com/vercel/next.js/pull/97507) — HIGH for pnpm/NixOS/monorepo
- [PR #97490 — Fix next/image transform requester-abort wedge](https://github.com/vercel/next.js/pull/97490) — HIGH for self-hosted next/image
- [PR #97287 — Fix standalone output for server-only files with adapter](https://github.com/vercel/next.js/pull/97287) — BLOCKER for adapter users

---

## `next@16.3.2` STABLE SHIPPED as Routine Bug-Fix Patch (NO CVE Included) + Aug 26 Critical CVE Pre-Announce T-4d (Aug 21–22, 2026 — v1.5.88 Cycle — Security Lens)

### `next@16.3.2` STABLE npm-Confirmed as Routine Patch (npm-published 2026-08-21T09:54:02Z)

**Confirmed by the official release notes** ([v16.3.2 GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.2)):

> [!NOTE]
> This release is backporting bug fixes. It does **not** include all pending features/changes on canary.

The 5 backports are: PR #97357 (scope app-entry export validation to files inside the app directory), PR #97416 (fix catch-all index page being served for every other slug), PR #97353/#97463 (Turbopack: don't trace embedded WASM loader helpers), PR #97453 (Turbopack: retain conditions when replacing resolve request keys), PR #97419 (Turbopack worker chunk loading with asset prefix), PR #97603 (Turborepo remote-cache OIDC replaces static PAT). **This is NOT the Aug 26 critical CVE patch release** — Aug 26 will ship `16.3.3` + `15.5.24` per the [official pre-announce](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026).

### Why `16.3.2` is NOT a Security Patch — Implications

The v1.5.83 cycle noted `next@16.3.2` was forecast as the Aug 20 monthly security release target. The release timeline was: Aug 20 monthly security release window CLOSED without shipping (the first miss since v1.5.0 on Jun 19), then `16.3.2` shipped Aug 21 as a routine bug-fix backport — independent of the security release cycle. The Aug 26 critical CVE patch will ship as **`next@16.3.3` + `next@15.5.24`** in 4 days (T-4d at 2026-08-22T18:02Z). Every Next.js app on 16.x or 15.x must plan an upgrade window for **August 26, 2026**.

### Security-Relevant Change That DID Land in 16.3.2: PR #97603 Turborepo OIDC

[PR #97603](https://github.com/vercel/next.js/pull/97603) — `Authenticate Turborepo remote caching with OIDC instead of a static PAT` (eps1lon). This is a **supply-chain security hardening** that landed in `16.3.2`:

- **Before**: Turborepo remote-cache used a static Personal Access Token (PAT) for CI authentication. A leaked PAT grants long-lived, broad-scope access to the remote cache.
- **After**: OIDC token-based authentication — short-lived, scoped to a single CI job, auto-rotated by the CI provider (GitHub Actions OIDC, GitLab CI/CD, CircleCI OIDC).
- **Blast-radius impact**: a leaked OIDC token expires within the CI job's lifetime (typically 5–60 minutes), dramatically reducing the window of exploitation vs a long-lived PAT.
- **User-action required**: NONE for most apps. Only monorepo users with Turborepo remote-cache need to verify their CI's OIDC token issuance and `turbo.json` `authentication: "oidc"` config. The change is backwards-compatible (OIDC preferred, PAT fallback).

### The 16.4.0 Canary Line — Security-Relevant Changes

Two new canary drops opened the 16.4.0 line on Aug 21:

**[v16.4.0-canary.0](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.0)** (npm 2026-08-21T10:15:26Z) — 6 misc changes including: `feat(turbopack): isolate HMR listeners across microfrontends` (PR #95997 — microfrontend isolation = security boundary between independent HMR scopes) + `Turbopack: deduplicate Pages Router app chunks` (PR #97664 — supply-chain-relevant dedup) + `fix: preserve trailing slash during export MPA fallback` (PR #97613 — export correctness fix) + `Turbopack: Split the read and write codepath data structures for symlinks` (PR #97395 — symlink-handling separation, no security regression vs the PR #97507 fix in 16.3.2) + `Turbopack: Show last modified file when waiting for the filesystem to settle` (PR #97648 — dev-only diagnostic) + `fix: [react-sync] Check assignability before assigning the actor` (PR #97638 — type-system-correctness fix for the [react-sync](https://github.com/facebook/react/tree/main/packages/react-sync) actor-assignment, prevents a narrow type-confusion class).

**[v16.4.0-canary.1](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.1)** (npm 2026-08-21T23:53:40Z) — 17 misc changes including: `[PPF] unstable_prefetch()` (PR #97622 + scaffold PR #97618 — **the new programmatic partial-prefetching API**) + `[PPF] Instant validation for unstable_navigation()` (PR #97309) + `turbo-tasks: add scope_unbounded, a scoped execution primitive that allows more work to be discovered` (PR #95974 — internal scheduler primitive) + `Remove generated error codes` (PR #97687 — **breaking change**: any code that string-matches Next.js error codes will break; rely on error class names instead) + `Add a Turbopack error for missing root layouts` (PR #97639 — better DX, no security impact) + `Improve Partial Prefetching adoption checks` (PR #97637) + `docs: correct the upper stale bound for App Shell exclusion in cacheLife` (PR #97653) + `fix(turbopack): give eager import.meta.glob values the ESM namespace` (PR #96559 — ESM correctness fix) + `Turbopack: Avoid cloning paths in fs watcher if the path is already in a map` (PR #97655 — fs-watch perf, no security impact) + `Turbopack: trace graceful-fs calls` (PR #97694 — observability) + React canary upgrade `eb8feb71-20260814` → `eafeac09-20260819` (PR #97636 — already in 16.3.1-canary.26, refreshed here).

**Security implications for the 16.4.0 canary line**: PR #95997 (microfrontend HMR isolation = improved security boundary) and PR #97687 (Remove generated error codes = **breaking change for any code that string-matches error codes**) are the two material security-relevant items.

### Aug 26 Critical CVE Pre-Announce — Updated T-4d Status

Per the [official pre-announce](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026) (published 2026-08-20T18:00:00Z): the Aug 26 release addresses **ONE critical severity vulnerability** affecting both 16.x and 15.x. The patched versions will be **`16.3.3` + `15.5.24`**. Full advisory publishes alongside the release. T-4d from this cron's 18:02Z Aug 22 start = **Wednesday, August 26, 2026** (during North American business hours per the Next.js team's historical security-release timing).

### Updated Security Audit Recipe (v1.5.88)

1. **Verify your `next` pin**: `npm ls next`. Pin `next@^16.3.2` (most recent STABLE, includes PR #97603 OIDC + backport fixes) or stay on `next@15.5.x` for LTS users.
2. **Aug 26 calendar reminder**: pre-flight `npm view next@latest version` at 14:00Z Aug 26 + immediate `npm install next@latest && npm install next@15.5.24` depending on your pin.
3. **For Turborepo monorepos with remote-cache**: verify `turbo.json` has `"remoteCache": { "signature": true }` + the CI provider has OIDC token issuance enabled (GitHub Actions: `permissions: { id-token: write }`). The PR #97603 OIDC hardening ships in 16.3.2.
4. **Audit string-matched error codes**: PR #97687 in 16.4.0-canary.1 removes generated error codes — any code that string-matches `E###` codes will break in 16.4.0+. Migrate to `instanceof ErrorClass` checks now.
5. **For 16.4.0-canary.1 adopters**: PR #95997 microfrontend HMR isolation is the new security boundary — verify microfrontend boundaries in your module-federation setup.
6. **Verify React `eafeac09` canary**: 16.4.0-canary.1 bundles React 19.3.0-canary-eafeac09-20260819 internally via PR #97636. Do NOT install `react@canary` separately — the App Router bundles it.

### Why this matters for `security.md`

The v1.5.83 cycle flagged `next@16.3.2` as "routine patch shipped Aug 21" but **did not confirm that 16.3.2 is a routine PATCH, not the Aug 26 critical CVE patch**. The v1.5.88 confirmation comes from the official release body which states: "This release is backporting bug fixes. It does **not** include all pending features/changes on canary." This rules out 16.3.2 being a CVE patch. The Aug 26 CVE patch release will be `next@16.3.3 + next@15.5.24`, NOT `16.3.2`. This is the single most consequential security clarification since the v1.5.83 cycle.

### Sources

- [Official v16.3.2 release notes](https://github.com/vercel/next.js/releases/tag/v16.3.2) — "backporting bug fixes; does not include all pending features/changes on canary" (verbatim confirmation that 16.3.2 is NOT the Aug 26 CVE patch)
- [Official v16.4.0-canary.0 release notes](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.0) — 6 misc changes including PR #95997 microfrontend HMR isolation + PR #97395 symlink-handling split + PR #97638 react-sync assignability
- [Official v16.4.0-canary.1 release notes](https://github.com/vercel/next.js/releases/tag/v16.4.0-canary.1) — 17 misc changes including PR #97622 `[PPF] unstable_prefetch()` SHIPPED + PR #97687 Remove generated error codes (BREAKING) + PR #97309 `[PPF] Instant validation for unstable_navigation()`
- [Upcoming Next.js August Security Release (Aug 26 CVE pre-announce)](https://nextjs.org/blog/upcomingnextjs-security-release-august-2026) — ONE critical CVE; `16.3.3 + 15.5.24`; full advisory publishes alongside the release
- [npm `next@16.3.2`](https://www.npmjs.com/package/next?activeTab=versions) — npm-published 2026-08-21T09:54:02.099Z
- [npm `next@16.4.0-canary.0`](https://www.npmjs.com/package/next?activeTab=versions) — npm-published 2026-08-21T10:15:26.029Z (15min 24s after 16.3.2 STABLE cut)
- [npm `next@16.4.0-canary.1`](https://www.npmjs.com/package/next?activeTab=versions) — npm-published 2026-08-21T23:53:40.907Z
- [PR #97603 — Turborepo remote-cache OIDC](https://github.com/vercel/next.js/pull/97603) — supply-chain hardening (PAT → OIDC); shipped in 16.3.2
- [PR #97622 — [PPF] `unstable_prefetch()`](https://github.com/vercel/next.js/pull/97622) — the new programmatic partial-prefetching API
- [PR #97687 — Remove generated error codes](https://github.com/vercel/next.js/pull/97687) — BREAKING change for string-matched error code consumers
- [PR #95997 — feat(turbopack): isolate HMR listeners across microfrontends](https://github.com/vercel/next.js/pull/95997) — security boundary between microfrontend HMR scopes
- [PR #97638 — [react-sync] Check assignability before assigning the actor](https://github.com/vercel/next.js/pull/97638) — type-system-correctness fix
- [Cross-reference: `routing.md` — PPF `unstable_prefetch()` + `unstable_navigation()` instant validation routing-surface impact
- [Cross-reference: `deployment.md` — full deployment-impact lens for 16.3.2 + 16.4.0-canary.0/1 + Aug 26 deployment-readiness checklist
