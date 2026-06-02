# Auth — NextAuth.js v5 + JWT

## NextAuth.js v5 Overview

NextAuth v5 (beta, available via `next-auth@beta`) is the current recommended version for new projects. It uses JWT sessions by default and supports multiple providers.

```bash
npm install next-auth@beta
```

## Configuration

### `auth.ts` (Root config)

```ts
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
        
        // Replace with your actual DB call
        const user = await db.user.findUnique({ 
          where: { email: parsed.data.email } 
        })
        
        if (!user) return null
        
        // Verify password with bcrypt
        const valid = await bcrypt.compare(parsed.data.password, user.passwordHash)
        if (!valid) return null
        
        return { id: user.id, email: user.email, name: user.name, role: user.role }
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      // Add custom fields to token
      if (user) {
        token.role = (user as { role: string }).role
        token.id = user.id
      }
      return token
    },
    async session({ session, token }) {
      // Expose token fields to client
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

## Auth in Server Components

```tsx
// app/dashboard/page.tsx
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

## Protected Routes with `proxy.ts` (Next.js 16)

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

import { auth, signIn as nextAuthSignIn } from '@/auth'
import { redirect } from 'next/navigation'

export async function login(formData: FormData) {
  const email = formData.get('email') as string
  const password = formData.get('password') as string
  
  try {
    await nextAuthSignIn('credentials', { email, password, redirectTo: '/dashboard' })
  } catch (error) {
    // NextAuth throws on error — redirect doesn't throw
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

## Common Issues

- **`signIn()` throws instead of returning error** — wrap in try/catch, use `redirect()` instead of throwing in some flows
- **Session doesn't update after role change** — JWT strategy: session updates on next login; DB strategy: use `useSession().update()`
- **CORS errors with credentials** — ensure `Origin` header matches in development
- **Using `middleware.ts` instead of `proxy.ts`** in Next.js 16 — the old filename works but is deprecated; `proxy.ts` is the forward-looking name
- **`matcher` pattern errors** — always include both the catch-all pattern AND explicit file extensions: `['/((?!api|_next/static|_next/image|favicon.ico|.*\\..*).*)']`
- **Proxy bypass vulnerabilities** — Next.js 16.2.6 fixes multiple bypass vectors; always re-validate auth in route handlers, not just in `proxy.ts`
- **`auth()` in proxy.ts** — in Next.js 16, `auth()` from NextAuth v5 is a function you call, not a middleware wrapper; use `await auth()` to get the session

**Sources:**
- [NextAuth.js v5 docs](https://authjs.dev/reference.nextjs)
- [Next.js 16 upgrade guide — proxy](https://nextjs.org/docs/app/guides/upgrading/version-16)
- [Next.js 16.2.6 security release](https://github.com/vercel/next.js/releases/tag/v16.2.6)
