# Routing — Next.js App Router

## File-Based Routing

Next.js App Router uses filenames as routes:

| File | Route |
|---|---|
| `app/page.tsx` | `/` |
| `app/about/page.tsx` | `/about` |
| `app/blog/[slug]/page.tsx` | `/blog/:slug` |
| `app/(marketing)/layout.tsx` | Route groups (no URL impact) |
| `app/api/users/route.ts` | `/api/users` |
| `app/@modal/(.)photo/[id]/page.tsx` | Parallel routes |

## Route Groups

Use `(group)` to organize routes without affecting the URL:

```
app/
├── (marketing)/
│   ├── layout.tsx        # Shared layout for marketing pages
│   ├── about/page.tsx   # /about
│   └── pricing/page.tsx # /pricing
├── (app)/
│   ├── layout.tsx        # Shared layout for app pages (with auth)
│   ├── dashboard/page.tsx # /dashboard
│   └── settings/page.tsx # /settings
```

## Dynamic Routes

### Single Segment: `[slug]`

```tsx
// app/blog/[slug]/page.tsx
interface BlogPostPageProps {
  params: Promise<{ slug: string }>
}

export default async function BlogPostPage({ params }: BlogPostPageProps) {
  const { slug } = await params
  const post = await getPostBySlug(slug)
  
  if (!post) notFound()
  return <article>{post.content}</article>
}
```

**Note:** In Next.js 15, `params` is a `Promise` — always `await` it.

### Multiple Segments: `[...slug]`

```tsx
// app/docs/[...slug]/page.tsx
// Matches /docs/intro, /docs/api/auth, /docs/api/auth/login, etc.

export default async function DocPage({ params }: { params: Promise<{ slug: string[] }> }) {
  const { slug } = await params
  const path = slug.join('/')
  return <DocContent path={path} />
}
```

### Catch-All: `[[...slug]]`

```tsx
// Optional catch-all — matches /, /a, /a/b, etc.
// [[...slug]] vs [...slug]: the latter requires at least one segment
```

## Layouts

### Root Layout

```tsx
// app/layout.tsx — required, wraps every page
import type { Metadata } from 'next'
import './globals.css'

export const metadata: Metadata = {
  title: 'My App',
  description: 'Description of my app',
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
```

### Nested Layouts

```tsx
// app/dashboard/layout.tsx — wraps all /dashboard/* pages
// Persists across navigation within dashboard

export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex h-screen">
      <Sidebar />
      <main className="flex-1">{children}</main>
    </div>
  )
}
```

## Dynamic Metadata (`generateMetadata`)

For metadata that depends on dynamic data (e.g., blog post titles, OG images), use `generateMetadata`:

```tsx
// app/blog/[slug]/page.tsx
import type { Metadata } from 'next'

interface Props {
  params: Promise<{ slug: string }>
}

// Generate metadata at build time or request time
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params
  const post = await getPostBySlug(slug)
  
  if (!post) return { title: 'Post not found' }
  
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [{ url: post.ogImage, width: 1200, height: 630 }],
    },
    twitter: {
      card: 'summary_large_image',
      title: post.title,
      description: post.excerpt,
      images: [post.ogImage],
    },
  }
}

export default async function BlogPostPage({ params }: Props) {
  const { slug } = await params
  const post = await getPostBySlug(slug)
  if (!post) notFound()
  return <article>{post.content}</article>
}
```

**Key pattern:** `params` in `generateMetadata` is also a `Promise` in Next.js 15 — always `await` it.

### `generateViewport` (Mobile Responsiveness)

```tsx
import type { Metadata, Viewport } from 'next'

export const viewport: Viewport = {
  themeColor: [
    { media: '(prefers-color-scheme: light)', color: '#ffffff' },
    { media: '(prefers-color-scheme: dark)', color: '#0a0a0a' },
  ],
  width: 'device-width',
  initialScale: 1,
}

// Or use generateViewport for dynamic viewport config
export function generateViewport({ params }: Props): Viewport {
  return {
    themeColor: '#4f46e5',
  }
}
```

## Route Segment Config

Control how a route is rendered and cached with export config:

```tsx
// app/api/health/route.ts
export const dynamic = 'force-dynamic'     // Always render at request time (no cache)
export const revalidate = 0                 // Revalidate every request (same as above)
```

```tsx
// app/blog/[slug]/page.tsx
export const revalidate = 3600  // Revalidate this page every hour (ISR)
```

**Options for `dynamic`:**
| Value | Behavior |
|---|---|
| `'auto'` | Default — Next.js decides based on data fetching |
| `'force-dynamic'` | Always render at request time (no cache) |
| `'force-static'` | Force static rendering (throws if you try to use dynamic data) |
| `'error'` | Error if dynamic data is accessed without explicitly opting out |

**Note:** `revalidate = 0` is equivalent to `dynamic = 'force-dynamic'`.

## Loading UI

```tsx
// app/dashboard/loading.tsx — shown while dashboard pages load
export default function Loading() {
  return <Skeleton className="h-32 w-full" />
}
```

This works with `<Suspense>` boundaries — the loading.tsx is the Suspense fallback for the content inside the layout.

## Error Handling

### Not Found

```tsx
// 404 — app/blog/[slug]/page.tsx
import { notFound } from 'next/navigation'

if (!post) notFound()
```

```tsx
// Custom not-found page — app/not-found.tsx
export default function NotFound() {
  return (
    <div className="flex flex-col items-center justify-center h-screen">
      <h1 className="text-4xl font-bold">404</h1>
      <p>This page doesn't exist.</p>
    </div>
  )
}
```

### Error Boundary

```tsx
// app/dashboard/error.tsx — client component for error handling
'use client'

import { useEffect } from 'react'

export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  useEffect(() => {
    // Log to error tracking service
    console.error(error)
  }, [error])

  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  )
}
```

## Parallel Routes

Render multiple routes simultaneously in the same layout:

```tsx
// app/@feed/(.)timeline/page.tsx  → /timeline
// app/@feed/(.)notifications/page.tsx → /notifications

// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <div>
      <Header />
      <FeedTabs />       {/* Tab navigation */}
      {children}         {/* @feed slot */}
      <Sidebar />
    </div>
  )
}
```

## Intercepting Routes

Intercept a route to show a modal instead of navigating:

```
app/
├── (feed)/
│   ├── photo/[id]/page.tsx      # Full page: /photo/123
│   └── @modal/(.)photo/[id]/page.tsx  # Intercepted modal overlay
```

When visiting `/photo/123` from within the app, the modal version is shown. Direct navigation loads the full page.

## Navigation API

### `useRouter` (Client Component)

```tsx
'use client'
import { useRouter } from 'next/navigation'

export function BackButton() {
  const router = useRouter()
  return <button onClick={() => router.back()}>Go back</button>
}
```

### `redirect` (Server Component)

```tsx
import { redirect } from 'next/navigation'
import { auth } from '@/lib/auth'

export default async function AdminPage() {
  const session = await auth()
  if (!session?.user) redirect('/login')
  // ...
}
```

### `Link` Component

```tsx
import Link from 'next/link'

// Prefer <Link> over <a> for client-side navigation
<Link href="/blog/nextjs-15" className="text-primary hover:underline">
  Read more
</Link>
```

**Never use `<a href="...">` for internal links** — it causes a full page reload.

## Programmatic Navigation

```tsx
// Client-side navigation
router.push('/dashboard')
router.replace('/login') // replaces history entry

// With search params
router.push('/search?q=nextjs&page=2')

// Prefetch a route (preload on hover)
router.prefetch('/dashboard')
```

## Route Handlers vs Pages

| Route Handler | Page |
|---|---|
| `app/api/*/route.ts` | `app/*/page.tsx` |
| Handles API requests (REST/GraphQL) | Renders UI |
| Can use all HTTP methods | One export (default) |
| No UI rendering | Server or Client Component |
