# Auth — NextAuth.js v4 (Legacy) + v5 (Production) + Better Auth + Clerk

## Which Library to Use

| Library | Latest stable | Cadence | License | When to Use |
|---|---|---|---|---|
| **NextAuth.js v5 (Auth.js)** | `5.0.0-beta.31` (April 14, 2026) | ⚠️ **Stagnant** — no new beta since April 14, 2026 (~3 months stale; previous beta.30 was October 27, 2025 — 6-month gap) | MIT | Free, framework-agnostic, integrates with Next.js 16 `proxy.ts`. Use when you don't want a hosted dependency and Auth.js v5 has the providers you need. |
| **NextAuth.js v4** | `4.24.14` (April 14, 2026) | Maintenance only — same date as v5 beta | MIT | Existing v4 projects only. Do not start new projects on v4. |
| **Better Auth** | `1.6.23` (June 29, 2026); `1.7.0-rc.1` (July 2, 2026) | ✅ **Active** — 1.6.16→1.6.23 in 4 weeks, 1.7.0-rc.1 3 days ago | MIT | Recommended default for new Next.js apps in 2026. TypeScript-first, batteries-included (email/password, magic links, passkeys, 2FA, organizations, admin plugin), no hosted dependency, owns its DB schema. |
| **Clerk** | `7.5.12` (`@clerk/nextjs` `latest`, July 3, 2026); also ships `@clerk/nextjs@latest-nextjs-v5` `5.7.6` | ✅ Active — daily canaries + weekly stables | Proprietary (free tier to 10K MAU; ~$0.02/MAU + $25/mo after) | Use when DX velocity matters more than cost/control. Pre-built UI components, organizations + MFA + passkeys out of the box. |
| **Supabase Auth** | bundled with Supabase | Active (platform cadence) | MIT (library) + SaaS pricing | Use when you're already on Supabase (native RLS integration via `auth.uid()`). |
| **Auth0 / WorkOS** | n/a | SaaS cadence | Proprietary | Enterprise SSO (SAML/SCIM). Better Auth doesn't have native SSO yet — see Better Auth section. |
| **Lucia / Oslo + custom** | n/a | Library cadence | MIT | Low-level primitives when no library's opinions fit. Rare in 2026. |

**Current state (July 2026):** The auth landscape has materially shifted in the last 6 months:

1. **Better Auth has overtaken Auth.js in monthly npm downloads** for the first time (per [npm trends](https://npmtrends.com/better-auth-vs-next-auth), June 2026). 1.6.23 shipped June 29, 1.7.0-rc.1 shipped July 2 — active development.
2. **NextAuth v5 beta has stagnated**: beta.31 (April 14, 2026) is the latest, ~3 months old. The previous release (beta.30) was October 27, 2025 — a 6-month gap. The API is stable but the release cadence is a yellow flag if you need new providers/features. Plan for a possible Auth.js v6 announcement — check the [Auth.js discussions](https://github.com/nextauthjs/next-auth/discussions) before committing to a major build on top of it.
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

**Note on the npm dist-tag:** `next-auth@5.0.0-beta.31` (April 14, 2026) is still published under the `beta` dist-tag — you install with `next-auth@beta`. This remains a packaging artifact; the API and runtime have been stable for ~21 months. The skill lists v5 as "✅ Production" because that's the de-facto reality across the ecosystem, but the cadence concern above is real.

### When to Consider an Alternative

NextAuth/Auth.js is no longer the default — **Better Auth is** for most new Next.js SaaS apps in 2026. Per the LogRocket April 2026 comparison of every major auth library for Next.js, and the [MakerKit July 2026 comparison](https://makerkit.dev/blog/tutorials/better-auth-vs-clerk) of Better Auth vs Clerk vs NextAuth vs Supabase Auth for production Next.js SaaS:

- **Better Auth** — Open-source (MIT), TypeScript-first, batteries-included (email/password, magic links, passkeys, 2FA, organizations). GA since v1.0 (late 2024), v1.6.23 stable June 29 2026, v1.7.0-rc.1 shipped July 2 2026. Use when you want full control without rolling your own. Cost at 100K MAU: ~$50/mo (just your Postgres).
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



## Better Auth — Recommended Default for New Next.js Apps

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

**Sources:**
- [NextAuth.js v5 docs](https://authjs.dev/reference.nextjs)
- [NextAuth.js v4 docs](https://next-auth.js.org/getting-started/introduction)
- [Next.js 16 upgrade guide — proxy](https://nextjs.org/docs/app/guides/upgrading/version-16)
- [Next.js 16.2.6 security release](https://github.com/vercel/next.js/releases/tag/v16.2.6)
