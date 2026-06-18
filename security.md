# Security — XSS, CSRF, CSP, Input Sanitization + Next.js Security

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
- **Vitest Browser Mode on a network-exposed host (--browser.api.host=0.0.0.0)** — exposes API token, project root, and CDP RPC; CVSS 9.8 RCE on pre-4.1.8 versions. Keep Browser Mode on localhost and upgrade to vitest ≥ 4.1.8

**Sources:**
- [Next.js 16.2.6 security release](https://github.com/vercel/next.js/releases/tag/v16.2.6)
- [GHSA-8h8q-6873-q5fj: DoS with Server Components](https://github.com/vercel/next.js/security/advisories/GHSA-8h8q-6873-q5fj)
- [GHSA-267c-6grr-h53f: Middleware/Proxy bypass](https://github.com/vercel/next.js/security/advisories/GHSA-267c-6grr-h53f)
- [GHSA-c4j6-fc7j-m34r: SSRF via WebSocket upgrades](https://github.com/vercel/next.js/security/advisories/GHSA-c4j6-fc7j-m34r)
- [GHSA-ffhc-5mcf-pf4q: XSS via CSP nonces](https://github.com/vercel/next.js/security/advisories/GHSA-ffhc-5mcf-pf4q)
- [GHSA-wfc6-r584-vfw7: Cache poisoning in RSC](https://github.com/vercel/next.js/security/advisories/GHSA-wfc6-r584-vfw7)
