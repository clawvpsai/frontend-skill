# Security — XSS, CSRF, CSP, Input Sanitization + Next.js Security

## Why This Matters in 2026

The defensive posture in this file is calibrated to a specific shift: **attackers are now faster than the disclosure cycle**. Per ProjectDiscovery's "The Vulnerability Curve Bent With the AI Curve" (June 18, 2026), median time-to-exploit collapsed from **63 days (2018) → 5 days (2023) → negative (2024+)**, meaning vulnerabilities are now exploited *before* a CVE is published. At the same time, total published CVEs jumped from ~18,000/year in 2018 to a projected ~50,000/year in 2025–2026.

The implication: dependency hygiene, supply-chain vetting, and the "patch fast" reflex in this file are not paranoia — they are the only window defenders have. A package added to your `package.json` with a caret range can be pwned the same day. A typo in a Server Action ownership check is exploitable before your WAF rule is live.

Concrete rules of thumb that fall out of this:

- **Pin exact versions for security-critical deps** — `^1.11.21` lets an attacker bump the minor. Use exact versions or `npm ci` with a lockfile review.
- **Audit dormant maintainer accounts in your org scopes quarterly** — TanStack (May 11), node-ipc (May 15), Mini Shai-Hulud (June 1), Mastra (June 17) all exploited stale contributor access.
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

## Server Actions Are Public POST Endpoints (2026 Architectural Principle)

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
- **Trusting the Node.js June 18 patch to cover all `undici` CVEs** — it only covers the version bundled with Node (`process.versions.undici`). If any package in your dependency tree pulls in its own `undici` (AWS SDK, OpenAI SDK, Anthropic SDK, Supabase, Prisma, NextAuth, WebSocket clients, etc.), `npm ls undici` may show an older version. Use `overrides` / `pnpm.overrides` to force a patched `undici` (7.28.0 / 8.5.0) across the whole tree

**Sources:**
- [Next.js 16.2.6 security release](https://github.com/vercel/next.js/releases/tag/v16.2.6)
- [GHSA-8h8q-6873-q5fj: DoS with Server Components](https://github.com/vercel/next.js/security/advisories/GHSA-8h8q-6873-q5fj)
- [GHSA-267c-6grr-h53f: Middleware/Proxy bypass](https://github.com/vercel/next.js/security/advisories/GHSA-267c-6grr-h53f)
- [GHSA-c4j6-fc7j-m34r: SSRF via WebSocket upgrades](https://github.com/vercel/next.js/security/advisories/GHSA-c4j6-fc7j-m34r)
- [GHSA-ffhc-5mcf-pf4q: XSS via CSP nonces](https://github.com/vercel/next.js/security/advisories/GHSA-ffhc-5mcf-pf4q)
- [GHSA-wfc6-r584-vfw7: Cache poisoning in RSC](https://github.com/vercel/next.js/security/advisories/GHSA-wfc6-r584-vfw7)
