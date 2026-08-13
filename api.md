# API — Route Handlers + Server Actions

## Route Handlers (app/api/)

Route handlers are the Next.js equivalent of API endpoints. They live in `app/api/` and map directly to HTTP methods.

### Basic REST Handler

```ts
// app/api/users/route.ts
import { NextResponse } from 'next/server'
import { z } from 'zod'
import { db } from '@/lib/db'

const CreateUserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  role: z.enum(['admin', 'user']).default('user'),
})

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url)
  const page = Number(searchParams.get('page') ?? '1')
  const limit = Number(searchParams.get('limit') ?? '10')
  
  const [users, total] = await Promise.all([
    db.user.findMany({ 
      skip: (page - 1) * limit, 
      take: limit,
      orderBy: { createdAt: 'desc' },
    }),
    db.user.count(),
  ])
  
  return NextResponse.json({
    data: users,
    meta: { page, limit, total, pages: Math.ceil(total / limit) },
  })
}

export async function POST(request: Request) {
  const body = await request.json()
  const parsed = CreateUserSchema.safeParse(body)
  
  if (!parsed.success) {
    return NextResponse.json(
      { error: 'Validation failed', details: parsed.error.flatten() },
      { status: 400 }
    )
  }
  
  const user = await db.user.create({ data: parsed.data })
  return NextResponse.json(user, { status: 201 })
}
```

### Dynamic Route Handler

```ts
// app/api/users/[id]/route.ts
import { NextResponse } from 'next/server'
import { db } from '@/lib/db'

interface Params { params: Promise<{ id: string }> }

export async function GET(_request: Request, { params }: Params) {
  const { id } = await params
  const user = await db.user.findUnique({ where: { id } })
  
  if (!user) {
    return NextResponse.json({ error: 'User not found' }, { status: 404 })
  }
  
  return NextResponse.json(user)
}

export async function PATCH(request: Request, { params }: Params) {
  const { id } = await params
  const body = await request.json()
  
  const user = await db.user.update({ where: { id }, data: body })
  return NextResponse.json(user)
}

export async function DELETE(_request: Request, { params }: Params) {
  const { id } = await params
  await db.user.delete({ where: { id } })
  return new NextResponse(null, { status: 204 })
}
```

## Server Actions (app/actions.ts)

Server Actions are functions that run on the server and can be called from client components or forms — no API route needed.

### Defining Server Actions

```ts
// app/actions.ts
'use server'

import { revalidatePath, revalidateTag } from 'next/cache'
import { redirect } from 'next/navigation'
import { z } from 'zod'
import { auth } from '@/auth'
import { db } from '@/lib/db'

const CreatePostSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1),
  published: z.boolean().default(false),
})

export async function createPost(formData: FormData) {
  // Server Actions are public POST endpoints — authenticate and authorize
  // INSIDE the action body. Page-level checks do not protect this endpoint.
  // See security.md → "Server Actions Are Public POST Endpoints (2026)"
  const session = await auth()
  if (!session?.user) {
    redirect('/login')
  }

  const parsed = CreatePostSchema.parse({
    title: formData.get('title'),
    content: formData.get('content'),
    published: formData.get('published') === 'on',
  })

  // The authorId MUST come from the authenticated session, never from formData
  // or a hardcoded placeholder — the previous pattern (`authorId: 'current-user-id'`)
  // was a critical IDOR footgun.
  await db.post.create({
    data: { ...parsed, authorId: session.user.id },
  })

  revalidatePath('/posts')
  redirect('/posts')
}
```

### Server Action with Error Handling

```ts
export async function updatePost(postId: string, formData: FormData) {
  const parsed = UpdatePostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
  })
  
  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors }
  }
  
  try {
    await db.post.update({ where: { id: postId }, data: parsed.data })
    revalidatePath(`/posts/${postId}`)
    return { success: true }
  } catch (err) {
    return { error: { root: ['Failed to update post'] } }
  }
}
```

### Calling Server Actions from Client Components

```tsx
'use client'

import { createPost } from '@/app/actions'
import { useFormStatus } from 'react' // React 19
import { useActionState } from 'react'  // React 19
import { useEffect } from 'react'

export function CreatePostForm() {
  // React 19 useActionState
  const [state, formAction, isPending] = useActionState(createPost, null)
  
  // Handle success/error from state
  useEffect(() => {
    if (state?.success) {
      // Form succeeded, possibly redirect happened
    }
  }, [state])
  
  return (
    <form action={formAction}>
      <input name="title" placeholder="Post title" />
      <textarea name="content" placeholder="Content" />
      <label><input type="checkbox" name="published" /> Publish</label>
      {state?.error?.root && <p className="text-destructive">{state.error.root}</p>}
      <SubmitButton />
    </form>
  )
}

function SubmitButton() {
  const { pending } = useFormStatus()
  return <button type="submit" disabled={pending}>{pending ? 'Saving...' : 'Save'}</button>
}
```
### Server Action Security Hardening (16.3.0-canary.92, 2026-07-21)

canary.92 shipped **five security hardening PRs** for Server Actions (plus one for redirects + one for fetch cache). All are included in 16.2.11 and 15.5.21:

**PR [#96012](https://github.com/vercel/next.js/pull/96012) — Enforce `serverActions.bodySizeLimit` for Edge runtime.** Previously, `serverActions.bodySizeLimit` (the per-action body size cap) was not enforced in the Edge runtime — a crafted request could bypass it. canary.92 enforces it consistently across Node.js and Edge runtimes. **Action:** audit your `serverActions.bodySizeLimit` value in `next.config.ts`. If you use the default and have Edge-deployed Server Actions, consider setting an explicit limit.

**PR [#96011](https://github.com/vercel/next.js/pull/96011) — Set correct origin for internal redirects in custom server.** When a custom server (e.g. `server.ts`, Express, Fastify) forwards a request to the Next.js router and the route handler calls `redirect()`, the `Location` header was using the custom server's origin instead of Next.js's own origin. This caused incorrect redirect URLs in deployments behind a proxy or CDN. **Action:** if you run a custom server behind a reverse proxy, re-test your `redirect()` calls after upgrading.

**PR [#96010](https://github.com/vercel/next.js/pull/96010) — Ensure exotic rewrite param values are properly encoded.** Param values in rewrites that contain unusual characters (Unicode, special symbols) were not consistently URL-encoded when forwarded to the destination. This could lead to open-redirect in rewrites that concatenate params into the destination URL. **Action:** audit every `beforeFiles`/`afterFiles` rewrite that uses route params in the destination path.

**PR [#96007](https://github.com/vercel/next.js/pull/96007) — Validate server reference IDs during manifest lookup.** Server Action / `use cache` IDs were previously accepted without verifying they exist in the manifest. An attacker who discovered an ID could invoke arbitrary server functions. canary.92 validates IDs against the manifest before allowing invocation. **This closes the endpoint disclosure (CVE-2026-64643) attack path.** Already-patched in 16.2.11 / 15.5.21.

**PR [#95608](https://github.com/vercel/next.js/pull/95608) — Preserve `basePath` in redirect destinations.** When `basePath` is configured in `next.config.ts` and a route handler calls `redirect(url)`, the redirect's `Location` header was dropping the `basePath`. canary.92 preserves it. **Action:** if you use `basePath`, re-test redirects after upgrading.

**PR [#96009](https://github.com/vercel/next.js/pull/96009) — fix(fetch-cache): key `fetch(Request, init)` by the effective request.** The two-argument form `fetch(new Request(url, init), differentInit)` had its cache key computed from the first argument only, ignoring the second. This is the root cause of CVE-2026-64648 (cache confusion). **Fix:** cache key is now derived from the effective merged request after combining both arguments. **Note:** the single-argument form `fetch(url, { body })` was already correct.

**PR [#96008](https://github.com/vercel/next.js/pull/96008) — fix(incremental-cache): byte-exact fetch cache key for binary bodies.** Binary request bodies were not included byte-exactly in the fetch cache key. This allowed cache collisions between requests with different binary bodies (the UTF-16 byte sequence issue in CVE-2026-64647). **Fix:** binary bodies are now hashed byte-exactly, not normalized or truncated.

**Security audit reminder:** The Server Action endpoint disclosure (CVE-2026-64643) was closed by PR #96007 — but the defense-in-depth pattern from `security.md` (auth + authorization inside every action body) remains the recommended posture. Layered security is never optional.


## Response Patterns

### Standard API Response

```ts
// Success
return NextResponse.json({ data: result, meta: { ... } })

// Created
return NextResponse.json(created, { status: 201 })

// No content
return new NextResponse(null, { status: 204 })

// Error
return NextResponse.json({ error: 'Message' }, { status: 400 })
```

### Streaming Responses

#### Streaming JSON Lines (JSONL)

For AI/LLM integrations, data pipelines, or long computations — stream newline-delimited JSON:

```ts
// app/api/stream-data/route.ts
export async function GET() {
  const encoder = new TextEncoder()
  
  const stream = new ReadableStream({
    async *asyncIterator() {
      const items = await fetchLargeDataset()
      for (const item of items) {
        yield encoder.encode(JSON.stringify(item) + '\n')
      }
    },
  })
  
  return new Response(stream, {
    headers: {
      'Content-Type': 'application/jsonl',
      'Transfer-Encoding': 'chunked',
      'Cache-Control': 'no-cache',
    },
  })
}
```

#### AI/LLM Streaming (OpenAI-compatible)

For streaming AI responses (OpenAI, Anthropic, etc.):

```ts
// app/api/chat/route.ts
import { OpenAI } from 'openai'
import { auth } from '@/auth'

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY })

export async function POST(request: Request) {
  const session = await auth()
  if (!session) return new Response('Unauthorized', { status: 401 })

  const { messages } = await request.json()
  
  const stream = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages,
    stream: true,
  })

  // Stream the OpenAI response directly to the client
  return new Response(stream.toReadableStream(), {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  })
}
```

#### Client-Side Streaming Consume

```tsx
// components/chat-stream.tsx
'use client'

import { useState } from 'react'

export function ChatStream() {
  const [messages, setMessages] = useState<Array<{ role: string; content: string }>>([])
  const [input, setInput] = useState('')
  const [streamText, setStreamText] = useState('')

  async function handleSubmit(e: React.SubmitEvent) {
    e.preventDefault()
    const userMessage = { role: 'user', content: input }
    setMessages(prev => [...prev, userMessage])
    setInput('')
    setStreamText('')

    const res = await fetch('/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ messages: [...messages, userMessage] }),
    })

    if (!res.ok) return

    const reader = res.body?.getReader()
    const decoder = new TextDecoder()

    if (reader) {
      let done = false
      while (!done) {
        const { value, done: doneReading } = await reader.read()
        done = doneReading
        if (value) {
          const chunk = decoder.decode(value)
          setStreamText(prev => prev + chunk)
        }
      }
    }
  }

  return (
    <div>
      <div className="whitespace-pre-wrap">{streamText}</div>
      <form onSubmit={handleSubmit}>
        <input value={input} onChange={e => setInput(e.target.value)} />
        <button type="submit">Send</button>
      </form>
    </div>
  )
}
```

### Server-Sent Events (SSE)

SSE is one-way server-to-client streaming — simpler than WebSockets, works over HTTP/2, auto-reconnects. Ideal for notifications, live updates, progress bars.

**Server side:**

```ts
// app/api/notifications/stream/route.ts
export async function GET(request: Request) {
  const encoder = new TextEncoder()
  
  const stream = new ReadableStream({
    start(controller) {
      // Send initial connection message
      controller.enqueue(encoder.encode('event: connected\ndata: {}\n\n'))

      // Subscribe to notification events
      const unsubscribe = subscribeToNotifications((notification) => {
        const data = `event: notification\ndata: ${JSON.stringify(notification)}\n\n`
        controller.enqueue(encoder.encode(data))
      })

      // Clean up when client disconnects
      request.signal.addEventListener('abort', () => {
        unsubscribe()
        controller.close()
      })
    },
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Connection': 'keep-alive',
      'Cache-Control': 'no-cache',
      'X-Accel-Buffering': 'no',  // Disable Nginx buffering
    },
  })
}
```

**Client side:**

```tsx
// components/notification-listener.tsx
'use client'

import { useEffect, useRef } from 'react'

export function NotificationListener({ onNotification }: { onNotification: (n: Notification) => void }) {
  const esRef = useRef<EventSource | null>(null)

  useEffect(() => {
    const es = new EventSource('/api/notifications/stream')
    esRef.current = es

    es.addEventListener('notification', (e) => {
      const notification = JSON.parse(e.data)
      onNotification(notification)
    })

    es.addEventListener('connected', () => {
      console.log('SSE connected')
    })

    es.onerror = () => {
      // EventSource auto-reconnects; you can add custom backoff here
      console.error('SSE error — reconnecting...')
    }

    return () => {
      es.close()
    }
  }, [onNotification])

  return null  // Invisible component — manages the EventSource
}
```

**SSE vs WebSockets vs Streaming:**

| Pattern | Direction | Best For | Auto-Reconnect |
|---|---|---|---|
| SSE | Server → Client | Notifications, live updates, progress | ✅ Yes |
| WebSockets | Bidirectional | Real-time chat, games, collaborative | ❌ Manual |
| Fetch + ReadableStream | Client → Server → Stream | AI/LLM responses, file downloads | ❌ N/A |
| Server Actions | Client → Server | Form submits, data mutations | ❌ N/A |

**SSE Gotchas:**
- `X-Accel-Buffering: 'no'` — required if behind Nginx, otherwise Nginx buffers and delays SSE
- Browser `EventSource` auto-reconnects on disconnect — useful for resilient connections
- SSE works over HTTP/2 — prefer HTTP/2 to avoid connection limits
- If you need bidirectional communication, use WebSockets instead

## WebSockets (via Custom Server)

Next.js Route Handlers don't natively support WebSockets. For real-time bidirectional communication, use a custom server or a dedicated WebSocket provider (Socket.io, Pusher, Ably, Liveblocks).

### Option 1: Custom Server with WebSocket

For Next.js + WebSocket on the same port, use a custom server:

```ts
// server.ts — custom server with WebSocket support
import { createServer } from 'http'
import { parse } from 'url'
import next from 'next'
import { WebSocketServer, WebSocket } from 'ws'

const dev = process.env.NODE_ENV !== 'production'
const app = next({ dev })
const handle = app.getRequestHandler()

app.prepare().then(() => {
  const server = createServer((req, res) => {
    const parsedUrl = parse(req.url!, true)
    handle(req, res, parsedUrl)
  })

  // Attach WebSocket server
  const wss = new WebSocketServer({ server, path: '/ws' })

  wss.on('connection', (ws: WebSocket, req) => {
    console.log('WebSocket connected')

    ws.on('message', (data) => {
      // Handle incoming message
      const message = JSON.parse(data.toString())
      console.log('Received:', message)

      // Broadcast to all clients
      wss.clients.forEach((client) => {
        if (client !== ws && client.readyState === WebSocket.OPEN) {
          client.send(JSON.stringify(message))
        }
      })
    })

    ws.on('close', () => {
      console.log('WebSocket disconnected')
    })
  })

  server.listen(3000, () => {
    console.log('> Ready on http://localhost:3000')
  })
})
```

**Dockerfile for custom server:**

```dockerfile
# Stage 1: Build
FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Runtime
FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

### Option 2: Third-Party Real-Time Providers

For production, prefer managed real-time services over self-hosted WebSockets:

| Provider | Best For | SDK |
|---|---|---|
| **Pusher** | General real-time (notifications, chat) | `pusher-js` |
| **Ably** | High-scale messaging, presence | `ably` |
| **Liveblocks** | Collaborative editing (Yjs integration) | `@liveblocks/client` |
| **PartyKit** | Cloudflare Workers-based, serverless WebSockets | `partysocket` |
| **Socket.io** | Self-hosted, familiar API | `socket.io` |

**PartyKit example (serverless WebSockets on Cloudflare):**

```ts
// party/index.ts — deploys to Cloudflare Workers
import { Party } from '@party/partykit'

export default class MyParty extends Party {
  onConnect(conn) {
    console.log('Connected:', conn.id)
    conn.send(JSON.stringify({ type: 'welcome', message: 'Hello!' }))
  }

  onMessage(message, sender) {
    // Broadcast to all except sender
    this.broadcast(message, [sender])
  }
}
```

## CORS

Route handlers don't automatically set CORS headers. For cross-origin requests:

```ts
import { NextResponse } from 'next/server'

export async function GET(request: Request) {
  const response = NextResponse.json({ data: 'test' })
  
  // For specific origins
  const origin = request.headers.get('origin')
  if (origin === 'https://allowed-site.com') {
    response.headers.set('Access-Control-Allow-Origin', origin)
  }
  
  response.headers.set('Access-Control-Allow-Methods', 'GET, POST, OPTIONS')
  response.headers.set('Access-Control-Allow-Headers', 'Content-Type, Authorization')
  
  return response
}

export async function OPTIONS() {
  return new NextResponse(null, {
    status: 204,
    headers: {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    },
  })
}
```

## Rate Limiting

Use a simple in-memory or Redis rate limiter:

```ts
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '10 s'),  // 10 requests per 10 seconds
})

export async function POST(request: Request) {
  const ip = request.headers.get('x-forwarded-for') ?? 'anonymous'
  const { success, remaining } = await ratelimit.limit(ip)
  
  if (!success) {
    return NextResponse.json(
      { error: 'Too many requests' },
      { status: 429, headers: { 'X-RateLimit-Remaining': remaining.toString() } }
    )
  }
  
  // Handle request...
}
```

## Request Validation

Always validate request bodies with Zod before processing:

```ts
// ❌ Never trust raw request body
const { title, content } = await request.json()
await db.post.create({ data: { title, content } })

// ✅ Always validate
const body = await request.json()
const parsed = CreateUserSchema.safeParse(body)
if (!parsed.success) return NextResponse.json(parsed.error.flatten(), { status: 400 })
```

## 16.3 canary.80–86 Route Handler & Middleware Updates (July 8–14, 2026)

The route-handler / middleware surface changed in three ways since this file's last full pass on July 5, 2026. None are breaking; all are quality-of-life fixes that show up in edge-adapter deploys and Node-middleware instrumented routes.

### `maxDuration` Propagates to Edge Adapter (16.3.0-canary.80, PR [#95118](https://github.com/vercel/next.js/pull/95118))

The `export const maxDuration = N` route-segment config (which sets the function-execution timeout for a route handler or page) now propagates correctly to the edge adapter's runtime config. Previously the value was dropped on the floor for edge-rendered handlers — a route declared `maxDuration = 60` would silently run with the default 10s (Node) or 25s (edge) instead. **If you set `maxDuration` on a route that runs on the edge adapter, upgrade to canary.80+ to get the actual configured timeout.**

```ts
// app/api/long-running-report/route.ts
export const maxDuration = 300 // 5 min — now actually respected on edge
export const runtime = 'edge'  // edge adapter

export async function GET() {
  // Long-running aggregation that previously timed out at 25s now gets 300s
  const report = await aggregateLargeDataset()
  return Response.json(report)
}
```

### Middleware `request.body` is Now a `Readable` After `await next()` (16.3.0-canary.83, PR [#95607](https://github.com/vercel/next.js/pull/95607))

This was a subtle silent-corruption class: in middleware (`proxy.ts` in Next.js 16) when you consumed `request.body` (or called `request.json()` / `request.formData()`) and then invoked `NextResponse.next()` (or rewrote to a downstream route), the downstream route handler sometimes saw an already-consumed body — `request.json()` returned `{ }` or `request.body` was a closed stream. **Fix (canary.83):** the dev-server pipeline now constructs the downstream request body as a fresh `Readable` from the buffered chunks before forwarding, so route handlers after a middleware read still get a usable body.

```ts
// proxy.ts — BEFORE canary.83, downstream saw an empty body
export async function proxy(request: Request) {
  const body = await request.clone().json() // log the payload for debugging
  console.log('[proxy]', body)
  return NextResponse.next()
}

// proxy.ts — canary.83+ behavior: downstream route handler gets the same body
// (no code change required, but the silent-corruption footgun is gone)
export async function proxy(request: Request) {
  // request.json() no longer breaks downstream consumption
  if (request.headers.get('content-type')?.includes('application/json')) {
    const body = await request.clone().json()
    console.log('[proxy]', body)
  }
  return NextResponse.next()
}
```

**Practical:** if you previously worked around this with `request.clone()` + a "consume the body twice" dance, the workaround is no longer needed on canary.83+.

### Middleware Instrumentation-Await Fix (16.3.0-canary.79, PR [#95357](https://github.com/vercel/next.js/pull/95357))

OTEL middleware instrumentation was missing `await` on the proxy result in some code paths, which caused the `next` continuation to race ahead of the instrumentation span end — leading to spans that ended *before* the request actually finished (correlated traces were missing the tail of the request). canary.79 adds the missing `await` so middleware instrumentation spans correctly wrap the full request lifecycle. **No code change required** — this is a dev-only-observable fix (the trace shape improves in OTEL backends like Honeycomb, Datadog, Vercel Observability).


### `output: 'export'` Async-Init Errors Are No Longer Unhandled Rejections (16.3.0-canary.87, PR [#95799](https://github.com/vercel/next.js/pull/95799))

`output: 'export'` mode had no error handling around async module initialization. If a `route.ts`, `page.tsx`, or any other App Router entry point performed an async top-level operation (e.g. `await someInitStep()` at module scope, or `await import(...)` of a large data table), an error during that init ran as an unhandled rejection rather than a build error. canary.87 wraps async module initialization in proper error handling so the failure surfaces as a real build error with a stack trace pointing at the offending file.

```ts
// app/sitemap.xml/route.ts — pre-canary.87, an init error silently ran as unhandled rejection
const data = await fetchSitemapData() // if this throws, the build "completes" with an unhandled rejection
export async function GET() { return new Response(xml(data)) }
```

```ts
// canary.87+ — same code, init error now fails the build with a clear message
const data = await fetchSitemapData() // if this throws, `next build` exits non-zero with the error
export async function GET() { return new Response(xml(data)) }
```

**Action:** if you had a `process.on('unhandledRejection', ...)` shim in `instrumentation.ts` to surface these errors, it now swallows a real error and should be removed. Audit: `rg "unhandledRejection" instrumentation.ts` (or wherever the shim lives) — if found and only there to catch this, delete it.

### Top-Level Await in Metadata Routes is Now Correctly Awaited (16.3.0-canary.87, PR [#95790](https://github.com/vercel/next.js/pull/95790))

This is a quiet but important fix for any app that does heavy work at module scope in `opengraph-image.tsx`, `icon.tsx`, `apple-icon.tsx`, `twitter-image.tsx`, or `sitemap.ts`. Two classes were fixed:

1. **Cache-warming race (build-time error):** if a metadata module was slow to evaluate (e.g. `const font = await readFile(...)` at module scope), cache warming didn't wait for the top-level await to resolve. The actual build then ran the module again, didn't find the warm-up result, and failed with `Unexpected cache miss after cache warming phase`.
2. **`generateStaticParams` demotion (prerender loss):** if the evaluation was fast-but-async (e.g. `await readFile` on a small font), the build succeeded but the segment collector read the module's exports too early — before `generateStaticParams` was visible. The image/sitemap became dynamic instead of prerendered, breaking `dynamicIO` / `cacheComponents` guarantees.

Fix: `instrumentModuleGetter` is now placed around metadata modules and `await ensureUserLand` is called before export inspection. Remaining callsites are wrapped for consistency even where not strictly required.

**No code change required** — this is a fix. If you were working around either of these with custom `generateStaticParams` re-derivation or cache-warm scripts, those workarounds can be removed.

### Modules Exporting a `.then` (Thenable) No Longer Hang Module Tracking (16.3.0-canary.87, PR [#95789](https://github.com/vercel/next.js/pull/95789))

A subtle interaction with Cache Components + Turbopack: `trackPendingImport` was previously happy to adopt any thenable (anything with a `.then` method). But thenables are not always Promises, and the downstream code assumed Promise semantics — `Promise.resolve` was used to paper over the gap. The result: a client module that exported a `.then` function (intentionally, e.g. a tiny custom thenable wrapper) would hang the build indefinitely under CC + Turbopack. canary.87 narrows the check to `instanceof Promise` so only real top-level-awaited modules are tracked. `module.exports = new Promise(() => {})` (a never-resolving promise) is no longer tracked as a pending import, and client modules with a `then` function export are now safe.

**Practical:** if you have a non-Promise thenable exported from a client module and your CC + Turbopack build was hanging, try canary.87 before adding `// @ts-ignore` workarounds. If you depend on `new Promise(() => {})` being treated as a pending module to keep the route dynamic (an unintended side-effect of the old behaviour), use `await fetch('/keepalive', { cache: 'no-store' })` or a similar explicit dynamic API instead.

### Async Userland Loading is Now a State Machine (16.3.0-canary.87, PR [#95791](https://github.com/vercel/next.js/pull/95791))

Internal refactor: the per-module "is this loaded yet" state was previously scattered across multiple flags and a non-thunk overload of the `userland` factory. canary.87 extracts a `LazyModule` state machine with three verbs: `loadIfNeeded` (kick off loading), `waitUntilLoaded` (await completion), and `assertLoaded` (peek-or-throw). Reading `.userland` while pending now throws with an invariant so future regressions surface loudly instead of silently returning a half-initialised value. **No user-facing change** — purely a maintainability improvement. If you were relying on the previous `userland` factory's non-thunk overload, that overload is removed (no public API).

### `--debug-build-paths` Now Matches Metadata Routes (16.3.0-canary.87, PR [#95788](https://github.com/vercel/next.js/pull/95788))

The test-only `next build --debug-build-paths=app/foo/route.ts` flag was using a homecooked regex that didn't match App Router's actual route resolution. It silently skipped metadata routes (`opengraph-image.tsx`, `icon.tsx`, `apple-icon.tsx`, `twitter-image.tsx`, `sitemap.ts`, `robots.ts`). canary.87 makes it use the same App Router matching as production routing, so the flag now resolves the target correctly. If your CI debug scripts previously got "no match" for a metadata route, switch to the App-Router-relative path (e.g. `app/opengraph-image.tsx` not `app/sitemap.xml/route.ts`) and the flag will work.

### Request Insights Subscription Now Initialises Correctly (16.3.0-canary.87, PR [#95794](https://github.com/vercel/next.js/pull/95794))

The dev-only Request Insights stack (`experimental.requestInsights`, shipped across canary.84–86) had a subtle subscription bug: the dev bundler service is constructed outside request-local tracing state, so the previous `isRequestInsightsEnabled()` check could miss the configured flag and skip the subscription. The result was that `GET /_next/development/request-insights` returned an empty store, the `get_request_insights` MCP tool got an empty list, and `subscribe_to_request_insights` delivered no updates — even with `experimental.requestInsights: true` in `next.config.js`. canary.87 passes the configured state into the service constructor at boot so the subscription is established at startup. **Action:** if you've been seeing empty Request Insights despite enabling the flag, this is the fix.

## 16.3 RouteContext Helper + `generateStaticParams` + Cache Components Route Handler Patterns (July 21, 2026 — gap-fill)

Three material Next.js 16.x route-handler patterns that aren't in `api.md` yet but ship in `canary.91+` and stable `next@16.2.x` — and that any agent building an API today will hit on day one. The first cron to cover these is the 1.4.76 cron (2026-07-21).

### `RouteContext<'/route'>` Helper — Strongly-Typed Params Without Importing Anything

Added to Next.js in 16.x (docs last updated 2026-03-03) and globally available everywhere in your project after typegen. The `RouteContext` helper is a generic that types the `context` parameter of a route handler using the route's file path as a literal type — so `params` are inferred from the dynamic segments in the file name and you get autocomplete + compile-time safety without writing the type yourself.

```ts
// app/users/[id]/route.ts
import type { NextRequest } from 'next/server'

export async function GET(_req: NextRequest, ctx: RouteContext<'/users/[id]'>) {
  // ctx.params is typed as Promise<{ id: string }> — no manual annotation
  const { id } = await ctx.params
  return Response.json({ id })
}

// app/shop/[tag]/[item]/route.ts — multi-segment types are inferred
export async function GET(_req: NextRequest, ctx: RouteContext<'/shop/[tag]/[item]'>) {
  const { tag, item } = await ctx.params // both string
  return Response.json({ tag, item })
}

// app/blog/[...slug]/route.ts — catch-all
export async function GET(_req: NextRequest, ctx: RouteContext<'/blog/[...slug]'>) {
  const { slug } = await ctx.params // slug: string[]
  return Response.json({ slug })
}
```

**How it works:** `RouteContext` is generated by `next dev`, `next build`, or explicitly via `npx next typegen` (the standalone typegen command added in Next.js 16.3 for CI/editor use). After type generation, the helper is in the global type namespace — **no import needed**. If your editor doesn't see it, run `npx next typegen` to refresh the generated `.next/types/` directory.

**Why this beats hand-rolled types:**
```ts
// ❌ BEFORE — manual annotation, drift-prone
export async function GET(
  _req: Request,
  { params }: { params: Promise<{ id: string }> }
) { ... }

// ✅ AFTER — inferred from route literal, refactor-safe
export async function GET(_req: NextRequest, ctx: RouteContext<'/users/[id]'>) { ... }
```

If you rename the file from `[id]` to `[userId]`, the `RouteContext` literal auto-updates and the type errors surface at the call sites — no `string` vs `string` mismatch going silently unnoticed.

**Audit recipe:** `rg "params: Promise<" app/` — any route handler still using manual `{ params: Promise<{ ... }> }` annotations should migrate to `RouteContext` for typegen-driven safety.

Source: [nextjs.org/docs/app/api-reference/file-conventions/route#route-context-helper](https://nextjs.org/docs/app/api-reference/file-conventions/route#route-context-helper) (last updated March 3, 2026).

### `generateStaticParams` + Route Handlers + `cacheComponents: true` — Build-Time API Endpoints

Next.js 16.3 supports `generateStaticParams` on Route Handlers to prerender JSON responses at build time. Combined with Cache Components, you get the same model as Pages: routes prerender when they don't access uncached or runtime data, and you wrap data access in `use cache` to cache both the prerendered and the runtime params.

```ts
// app/api/posts/[id]/route.ts
export async function generateStaticParams() {
  // Must return at least one param under cacheComponents — see below
  return [{ id: '1' }, { id: '2' }, { id: '3' }]
}

// Pattern A: simple static — GET runs at build for each generated id
export async function GET(
  _req: NextRequest,
  ctx: RouteContext<'/api/posts/[id]'>
) {
  const { id } = await ctx.params
  const post = await getPost(id)
  return Response.json(post)
}

// Pattern B: cacheComponents + 'use cache' — cache the data lookup across prerendered AND runtime params
async function getPost(id: string) {
  'use cache'
  cacheLife('hours')
  return db.post.findUnique({ where: { id } })
}

export async function GET(
  _req: NextRequest,
  ctx: RouteContext<'/api/posts/[id]'>
) {
  const { id } = await ctx.params
  const post = await getPost(id)
  return Response.json(post)
}
```

**Hard constraint under `cacheComponents: true`:** `generateStaticParams` MUST return at least one param. An empty array raises `empty-generate-static-params` at build time:

```text
Error: generateStaticParams must return at least one param when cacheComponents is enabled.
See https://nextjs.org/docs/messages/empty-generate-static-params
```

**Why:** Cache Components requires every route to be prerenderable. Empty `generateStaticParams` means "no paths to prerender" which contradicts the model. Workarounds: (1) drop `cacheComponents` (revert to the legacy dynamic-by-default behaviour), (2) return at least one placeholder param, or (3) set `export const dynamic = 'force-dynamic'` and accept the route runs on every request.

**Replacing legacy `dynamic = 'force-static'`:** if you're migrating from a non-CC project, replace `export const dynamic = 'force-static'` with `'use cache'` on the data function. The route's caching is now driven by the cacheable function, not by the route config — this is the same model Cache Components uses for pages.

**Combining with dynamic params at runtime:** under `cacheComponents`, params not in `generateStaticParams` resolve at request time using the same `use cache` lookup — so `getPost('4')` reuses the cached `db.post.findUnique({ where: { id } })` call rather than fetching fresh each request. See `server-components.md` → `use cache` for the full model.

Sources:
- [nextjs.org/docs/app/api-reference/functions/generate-static-params](https://nextjs.org/docs/app/api-reference/functions/generate-static-params) — "With Route Handlers" + "With Cache Components" sections (last updated 2026-03-13)
- [nextjs.org/docs/app/guides/migrating-to-cache-components#route-handlers-get](https://nextjs.org/docs/app/guides/migrating-to-cache-components) — "Replace `dynamic = 'force-static'` with `use cache`" (June 23, 2026)
- [nextjs.org/docs/messages/empty-generate-static-params](https://nextjs.org/docs/messages/empty-generate-static-params) — the build error reference

### Static-by-Default GET Handlers Under `cacheComponents: true`

A subtle behavioural change that catches teams off guard: under `cacheComponents: true`, `GET` route handlers remain **static by default**. If your GET handler reads any uncached runtime data (cookies, headers, `await params`, `await searchParams`, `new Date()`, `Math.random()`, `crypto.randomUUID()`, etc.) without wrapping the data access in `use cache`, the handler **runs at build time** rather than per-request — which is usually a data-leakage footgun if the data is supposed to be per-user.

**The mistake** (GitHub discussion [#85326](https://github.com/vercel/next.js/discussions/85326), arjunkomath, October 2025):
```ts
// app/api/me/route.ts — looks dynamic, runs at build under cacheComponents
export async function GET() {
  const user = await getCurrentUser() // reads cookies() — runtime data
  return Response.json({ user })
}
// → `next build` calls GET once with no cookies, returns the unauthenticated shape,
//   and every visitor gets the same cached response
```

**The fix** — explicitly opt out:
```ts
// app/api/me/route.ts — now runs per-request
export const dynamic = 'force-dynamic'

export async function GET() {
  const user = await getCurrentUser()
  return Response.json({ user })
}
```

**Alternative:** if only part of the handler reads runtime data, push that part into a separate function and wrap the cacheable parts in `use cache`:
```ts
export async function GET(_req: NextRequest, ctx: RouteContext<'/api/me'>) {
  // The user ID is the runtime read — small + cheap, fine to fetch per-request
  const userId = await getCurrentUserId()
  // The profile payload is cacheable per-user
  const profile = await getProfile(userId) // wraps 'use cache' + cacheLife
  return Response.json({ profile })
}
```

**Audit recipe:** `rg "export async function GET" app/` — for each match, trace whether the body reads any uncached runtime data. If yes AND `cacheComponents: true`, verify either `dynamic = 'force-dynamic'` is set OR the runtime data is isolated in a non-cached scope.

**`generateStaticParams` interaction:** if you ALSO export `generateStaticParams`, the GET handler runs at build for those generated params AND the `dynamic = 'force-dynamic'` opt-out still works — force-dynamic just means "don't try to prerender, run per-request". Combining both is the cleanest pattern for "build a few representative params + run per-request for everything else".

Sources:
- [vercel/next.js Discussion #85326 — "Route handlers are running during build (cacheComponents)"](https://github.com/vercel/next.js/discussions/85326)
- [nextjs.org/docs/app/guides/migrating-to-cache-components](https://nextjs.org/docs/app/guides/migrating-to-cache-components)



## 16.3.0-canary.95 `generateStaticParams` Validation Hardening (PR #95968 + #95969 — gap-fill)

Two follow-up PRs from the same author (devjiwonchoi / SukkaW, both merged 2026-07-23T21:25:57Z and 2026-07-23T21:25:56Z, stacked on each other, supersedes #95388) harden `generateStaticParams` validation across **all** build modes — not just `cacheComponents: true`. The 1.4.76 cron covered the `empty-generate-static-params` error from canary.71 (which is the *only* error `generateStaticParams` raised previously). canary.95 adds **six new error codes** (1450–1455) to `packages/next/errors.json` that fire earlier in the build pipeline and give operators a stack trace pointing at the offending `return` line.

### PR #95968 — `Throw when generateStaticParams returns invalid values`

The previous validation only checked "is the array empty?" under `cacheComponents: true` — it would *silently* produce a wrong output when the function returned the wrong shape (a non-array, or items that weren't plain objects). The new errors fire on every build mode:

- **Error 1450** — `Invalid value returned from generateStaticParams for "<route>". Expected an array, but received type <type>.`
- **Error 1451** — `Invalid value at index <i> returned from generateStaticParams for "<route>". Expected an object, but received type <type>.`

Where `<route>` is the dynamic route's path (e.g. `app/blog/[slug]/page.tsx`) and `<type>` is the JS `typeof` result (`"object"`, `"string"`, `"undefined"`, `"null"`, etc.). The errors link to the new troubleshooting page `nextjs.org/docs/messages/generate-static-params`.

The validation **allows `{}`** as a single-item object (so the existing "all paths at runtime" pattern still works). The follow-up PR #95969 adds `output: 'export'`-specific checks that reject `{}` (since static export requires every params object to carry every dynamic segment).

**Mistakes that now throw** (previously silent):
```ts
// ❌ Returns an object instead of an array
export function generateStaticParams() {
  return { slug: 'first-post' } // throws error 1450
}

// ❌ Returns an array of strings
export function generateStaticParams() {
  return ['first-post', 'second-post'] // throws error 1451 (each index is a string, not an object)
}

// ❌ Returns an array with a null
export function generateStaticParams() {
  return [null, { slug: 'first-post' }] // throws error 1451 at index 0
}

// ❌ Returns a Promise<Array> that didn't resolve properly
export async function generateStaticParams() {
  return undefined // throws error 1450 (the awaited value is not an array)
}

// ✅ Correct shape (only this passes)
export async function generateStaticParams() {
  return [{ slug: 'first-post' }, { slug: 'second-post' }]
}
```

The PR also adds 247 lines of new test coverage in `packages/next/src/build/static-paths/app.test.ts` covering arrays-of-primitives, arrays-of-nulls, undefined returns, plain objects, arrays-of-arrays, and Promise<T> unwrapping cases.

### PR #95969 — `Throw for empty or incomplete generateStaticParams results with output: export`

Stacks on #95968. Tightens `output: 'export'` specifically — since static export builds every dynamic route at build time and writes it to disk, an empty or incomplete `generateStaticParams` was producing a silent missing-page state that only surfaced on the deploy target. canary.95 throws four new errors **only when `output: 'export'` is set**:

- **Error 1452** — `Page "<route>" is missing "generateStaticParams()" so it cannot be used with "output: export" config.` (dynamic route with no `generateStaticParams` export)
- **Error 1453** — `Page "<route>" is missing exported function "generateStaticParams()", which is required with "output: export" config.` (similar; raised by the dev server when the function isn't exported properly)
- **Error 1454** — `Page "<route>" returned an empty array from "generateStaticParams()". With "output: export", at least one route must be generated.` (the empty-array case for static export specifically — note: under non-export modes, `[]` is the documented "all paths at runtime" pattern and is still valid)
- **Error 1455** — `Page "<route>" returned incomplete params from "generateStaticParams()". With "output: export", every params object must include all dynamic route parameters. Missing: <names>.` (a params object is missing one or more dynamic segment values — e.g. for `/blog/[year]/[slug]` returning `{ year: '2026' }` without `slug`)

The "composed parent + child params" check is the new one: when multiple route segments each export `generateStaticParams`, Next.js combines them before generating routes; if the combined params omit a dynamic route parameter, error 1455 fires with the missing names. The doc page (modified by PR #95969) explains: *"When multiple route segments export `generateStaticParams`, Next.js combines the params returned by each parent and child function before generating routes. Export `generateStaticParams` from every dynamic route and make sure the combined params include all dynamic route parameters for every generated route. If the paths cannot be known at build time, remove `output: 'export'` from your Next.js configuration."*

**The full error table for `generateStaticParams` after canary.95:**

| Build mode | What throws | Error code |
|---|---|---|
| Any | Function returns a non-array | **1450** |
| Any | Array item is not a plain object | **1451** |
| `cacheComponents: true` | Function returns `[]` | `empty-generate-static-params` (already documented, 1.4.76) |
| `output: 'export'` | Dynamic route doesn't export `generateStaticParams` | **1452** / **1453** |
| `output: 'export'` | Function returns `[]` | **1454** |
| `output: 'export'` | Composed params omit a dynamic segment | **1455** |

**Audit recipe** — the new errors don't surface until `next build`, so a CI-friendly preflight catches them before you hit the deploy:
```bash
# 1. Find every dynamic route in app/
rg -l '\[.*\]' app/ -g '*.{ts,tsx}' | rg 'page\.(ts|tsx)|route\.(ts|tsx)$'

# 2. For each match, check that the file exports generateStaticParams
for f in $(rg -l '\[.*\]' app/ -g 'page\.(ts|tsx)|route\.(ts|tsx)$'); do
  if ! rg -q 'export.*generateStaticParams' "$f"; then
    echo "⚠️  $f has dynamic segments but no generateStaticParams"
  fi
done

# 3. If output: 'export' is set, additionally check that the function isn't empty
# and that it returns an object with every dynamic segment
rg 'output.*["']export["']' next.config.*
```

The error pages are also updated: `errors/generate-static-params.mdx` (the new doc added by #95968, extended by #95969) is now the canonical troubleshooting entry for all six error codes.

Sources:
- [PR #95968 — `Throw when generateStaticParams returns invalid values`](https://github.com/vercel/next.js/pull/95968) · devjiwonchoi (SukkaW) · merged 2026-07-23T21:25:56Z · **Shipped in `16.3.0-canary.95`** (`cf10c50` on canary-branch HEAD)
- [PR #95969 — `Throw for empty or incomplete generateStaticParams results with output: export`](https://github.com/vercel/next.js/pull/95969) · devjiwonchoi (SukkaW) · merged 2026-07-23T21:25:57Z · **Shipped in `16.3.0-canary.95`** (stacked on #95968, supersedes #95388)
- [`packages/next/errors.json` at canary.95 — errors 1450–1455](https://github.com/vercel/next.js/blob/canary/packages/next/errors.json#L1450)
- [`errors/generate-static-params.mdx` at canary.95 — troubleshooting page](https://github.com/vercel/next.js/blob/canary/errors/generate-static-params.mdx)
- [nextjs.org/docs/messages/generate-static-params — invalid-values page](https://nextjs.org/docs/messages/generate-static-params)
- [nextjs.org/docs/app/api-reference/functions/generate-static-params — returns section](https://nextjs.org/docs/app/api-reference/functions/generate-static-params#returns)

## Plain-Text 404 for Non-Document Requests (16.3.0-canary.96, [PR #95930](https://github.com/vercel/next.js/pull/95930) by Tobias Koppers / bgw, merged 2026-07-24T02:59:42Z — **SHIPPED in `16.3.0-canary.96` at 2026-07-25T00:00:34Z**)

When the browser requests something Next.js can't resolve as a document — favicons, manifest icons, broken `<img src>`, wrong `<link>`s, `new Worker()` calls — the server previously **still invoked the root layout and rendered the full HTML 404 page** (including any expensive `await cookies()` / `await headers()` reads in `app/layout.tsx`). The new behavior inspects the request's `Sec-Fetch-Dest` header and returns a **plain-text `Not Found` 404** directly from `base-server.ts` / `router-server.ts` without touching any app code.

**The new file [`packages/next/src/server/lib/is-non-html-sec-fetch-dest.ts`](https://github.com/vercel/next.js/blob/canary/packages/next/src/server/lib/is-non-html-sec-fetch-dest.ts)** matches the spec values: `audio`, `audioworklet`, `font`, `image`, `manifest`, `model`, `paint-worklet`, `script`, `serviceworker`, `sharedworker`, `style`, `track`, `video`, `worker`, `xslt`. Any request with a `Sec-Fetch-Dest` matching one of these gets the fast 404 path.

**Why this matters:**

- **Dev server perf** — `next dev` no longer pays the layout cost for a missing favicon or a typo'd `<img>`. The author measured a real case: a manifest icon hit that previously took 2 seconds in app code (per the request log) now returns in milliseconds.
- **Self-hosted prod** — `next start` with a slow or dynamic `app/layout.tsx` (e.g. one that reads `await headers()` or fetches the current user) no longer wastes a request budget on every broken image/manifest request.
- **Static self-hosted** — same as before for the static case (HTML 404 from `out/`) — the change only affects dynamic layouts.

**Behavior nuances:**

- **No DevTools log entry.** The request is handled at the router layer (`router-server.ts`) before reaching app code, so you won't see it in `next dev`'s terminal log. This matches `/_next/static/*` 404 behavior and the author notes "maybe a bit weird it doesn't show" — but it would be hard to log without losing the consistency.
- **`Sec-Fetch-Dest: 'document'`** requests still get the full HTML 404 (these are real page navigations that want the styled 404).
- **`Sec-Fetch-Dest: 'empty'`** (programmatic fetch with no specific destination) also gets the fast 404 path.
- **Browsers without `Sec-Fetch-Dest`** (old browsers, custom HTTP clients) get the legacy behavior because the header is absent.

**Test fixtures:**
- `test/e2e/app-dir/not-found-non-document/` — e2e with full app code
- `test/e2e/app-dir/not-found-non-document-dynamic/` — e2e with a slow dynamic layout
- `test/production/app-dir/not-found-non-document-minimal/` — adapter-level test

**Audit recipe:**
```bash
# Find places that might be hot-path on non-document requests
# (broken images, manifest icons, service workers)
rg -l "navigator.serviceWorker.register" app/
rg -l 'rel="manifest"' app/
rg -l 'rel="icon"' app/
# If you have many of these and a slow layout, canary.96+ will be measurable.
```

No code change required — the change is in the request routing layer.

Source: [PR #95930 — `Return plain text 404 for non-document requests to unknown paths`](https://github.com/vercel/next.js/pull/95930) · bgw · merged 2026-07-24T02:59:42Z · **Shipped in `16.3.0-canary.96`** (2026-07-25T00:00:34Z, ahead of the predicted 22:30Z window by ~22h).


## RSC HMR Flurry of Refetches Fixed — #96102 (16.3.0-canary.96, July 24, 2026 — was canary-branch ahead of canary.95)

A silent HMR regression that fired when a Server Component was imported from many routes simultaneously. To reproduce: a server component `Foo` imported from `/a`, `/b`, `/c`, ... — then edit `Foo`. **Expected:** one HMR request per open tab. **Actual:** a flurry of HMR requests, one per already-compiled route per tab. The PR author observed "a flurry of requests" in their terminal on every save.

**Two root causes** (both fixed in #96102):

1. The "server component updated" message was emitted **once per already-compiled route**, so a single tab received N messages in quick succession and processed them all.
2. The `debounce` for these messages was broken — it always fired immediately due to a wrong condition, instead of delaying to coalesce.

**Fix:** switch to expressing Server Component changes as a **final message** that gets sent at the end of the update via a dirty flag. With the dirty flag, the number of messages is decoupled from the number of compiled routes — one final message per update, one HMR fetch per tab. The broken `debounce` was deliberately left unfixed (the fix is more risky because existing callers may rely on the bug; `git blame` would show who).

**Who needs to audit:** any dev workflow that edits Server Components imported from many routes — common in monorepos or shared `components/` directories. **Verify the fix:** in a project where a single SC is imported from 10+ routes, open 3 tabs, edit the SC, and count the HMR requests in the dev terminal. Pre-fix: tens of requests; canary.96+: 3 (one per tab). **Source:** [PR #96102 — `[RSC HMR] Fix a flurry of refetches when a editing component imported from many routes`](https://github.com/vercel/next.js/pull/96102) · gaearon · merged 2026-07-24T02:59:27Z · **Shipped in `16.3.0-canary.96`**.

## Turbopack Re-Export Tree-Shake — #95989 (16.3.0-canary.96, July 24, 2026 — was canary-branch ahead of canary.95)

Tree-shake fix for async imports of re-export modules. **Root cause:** code like `export { foo } from 'bar'` contributed to the module's `exports` set but did **not** contribute to Turbopack's tracking of imports from `'bar'`. Turbopack therefore fell back to the default `ImportUsage::TopLevel`, which keeps every export of `'bar'` alive in the bundle. For **sync** imports/exports, Turbopack follows re-exports and the bug is invisible; for **async** imports, the path isn't followed and the bug fires.

**Real-world impact:** [viem](https://viem.sh/) chain definitions are async-imported; importing a single chain from `viem/chains` previously pulled the full chains namespace into the bundle because Turbopack couldn't prove the re-export was only referencing one name. Fixes [issue #95698](https://github.com/vercel/next.js/issues/95698). Same shape bug bites any other "barrel file + async import" pattern (Material UI v5+ via `@mui/material`, Lodash via `lodash-es`, most monorepo internal barrel exports).

**Fix:** `import_usage` now includes re-exports when computing which imports a module actually reads. The downstream module's unused exports are then dropped as normal. **Who needs to audit:** any project that does `await import('some-barrel-file')` and only references a single name from the barrel. **Verify:** `pnpm next build` + `npx next-bundle-analyzer` before/after; expect the barrel's unused exports to drop from the bundle. **Source:** [PR #95989 — `[turbopack] Track re-exports in import_usage inside of compute_import_usage`](https://github.com/vercel/next.js/pull/95989) · bgw · merged 2026-07-24T00:34:54Z · **Shipped in `16.3.0-canary.96`** · fixes [issue #95698](https://github.com/vercel/next.js/issues/95698).

## Server Action Redirects Now Return `200` for Client-Handled Redirects (16.3.0-canary.100, [PR #96310](https://github.com/vercel/next.js/pull/96310) by Zack Tanner, merged 2026-07-28T08:40:12Z, SHIPPED in `16.3.0-canary.100` at 2026-07-28T21:04:54Z)

Server Action redirects used to return a **`303 See Other`** without a `Location` header (and may have included a streamed RSC response body in the same response). The combination was hostile to HTTP intermediaries — reverse proxies, WAFs, CDN edge nodes, and corporate proxies commonly **reject** a `303` without `Location` as malformed, or **strip the response body** because it looks like a redirect response that shouldn't carry content. The fix changes the client-handled redirect path to **return `200 OK`** while continuing to use the `x-action-redirect` header and the RSC response body.

### The three Server Action redirect paths

| Redirect type | Pre-#96310 status | Post-#96310 status | Headers | Notes |
|---|---|---|---|---|
| **Server-rendered redirect** (call comes from a non-JS context — progressive enhancement, `curl`, no `Next-Action` header) | `303 See Other` | `303 See Other` (unchanged) | `Location: <target>` | Standards-compliant — proxies understand it. **Unchanged.** |
| **Client-handled redirect** (call comes from the JS runtime — `useFormState`, `<form action={action}>`, `startTransition`) | `303 See Other` **without `Location`** + RSC body | **`200 OK`** + `x-action-redirect: <target>` + RSC body | `x-action-redirect: <target>` | The fix. `200` is the correct status for "client, please read the body and apply the redirect that the `x-action-redirect` header tells you". Proxies now pass it through cleanly. |
| **`redirect()` from a Route Handler** (no Server Action context) | `307 Temporary Redirect` (unchanged) | `307 Temporary Redirect` (unchanged) | `Location: <target>` | Per the [`redirect()` docs](https://nextjs.org/docs/app/api-reference/functions/redirect). **Unchanged.** |

### Why the `x-action-redirect` header pattern is correct

The pre-#96310 `303` without `Location` was a misuse of the status code — `303` semantically means "I've moved the resource, go to this URL" but Next.js's client-handled redirect carries the destination **only in the response body / a custom header**, not in `Location`. This is fundamentally incompatible with HTTP/1.1 [RFC 7231 §6.4.4](https://datatracker.ietf.org/doc/html/rfc7231#section-6.4.4). The post-#96310 `200` is the honest status — "request succeeded, body and `x-action-redirect` header contain instructions for what to do next." HTTP intermediaries are expected to pass through `200 OK` responses and let the client (Next.js's action-result decoder) interpret the `x-action-redirect` header.

### Who needs to audit

- **Anyone with HTTP intermediary monitoring on Server Action endpoints** — your monitoring/alerting may have flagged `303` responses from Server Action POSTs as "potential redirect loop" or "incomplete response". Those alerts should be reviewed and **filtered to exclude Next.js Server Action endpoints** (or updated to whitelist `200` for these routes going forward).
- **Anyone with custom WAF rules** that block `303` without `Location` as a defensive measure — these will now be triggered by **pre-#96310 Server Action responses**, not by post-#96310 ones (which return `200`). Re-test your WAF rules on canary.100+.
- **Anyone proxying Next.js through Cloudflare, Fastly, Vercel Edge, AWS CloudFront, or nginx** — the body-stripping behavior that some intermediaries apply to `3xx` responses will now stop stripping the RSC body. This is a **fix for the bug**, not a regression.
- **Anyone with end-to-end (E2E) tests asserting on the redirect status code from a Server Action POST** — those tests may have been asserting `303` to detect a redirect. Update to assert `200` + check the `x-action-redirect` header for the destination, or check the post-action navigation (e.g. `await page.waitForURL('/target')`).

### Migration recipe for E2E tests

```ts
// Before (Playwright/Vitest/Detox):
const response = await page.request.post('/action-endpoint', { form: { id: '1' } })
expect(response.status()).toBe(303)  // fragile — depends on canary version
expect(response.headers().location).toBe('/target/1')

// After (works on canary.100+ AND pre-canary.100):
const response = await page.request.post('/action-endpoint', { form: { id: '1' } })
// Server-rendered: 303 + Location
// Client-handled:  200 + x-action-redirect (no Location)
// Either way, the browser ends up at /target/1 — just navigate and assert the URL:
await page.waitForURL('/target/1')
expect(page.url()).toBe('http://localhost:3000/target/1')
```

### Audit recipe

```bash
# Check what status code your Server Actions are returning today
rg -B1 -A5 "x-action-redirect" .next/server 2>/dev/null | head -30

# Or test directly:
curl -i -X POST http://localhost:3000/your-action-endpoint   -H "Content-Type: application/x-www-form-urlencoded"   -H "Next-Action: <action-id>"   --data "id=1"
# Pre-canary.100:  HTTP/1.1 303 See Other  (no Location)
# Post-canary.100: HTTP/1.1 200 OK          x-action-redirect: /target/1
```

### Source

- [PR #96310 — `Return 200 for client-handled Server Action redirects`](https://github.com/vercel/next.js/pull/96310) · Zack Tanner · merged 2026-07-28T08:40:12Z · **Shipped in `16.3.0-canary.100`** (2026-07-28T21:04:54Z) · fixes [#92882](https://github.com/vercel/next.js/issues/92882) · fixes [#74026](https://github.com/vercel/next.js/issues/74026)

## Common Mistakes — App Router uses `app/api/` with route handlers; don't mix with `pages/api/`
- **Missing `await` on params** — In Next.js 15, route handler params are Promises
- **Not returning proper status codes** — 201 for create, 204 for delete, 404 for not found
- **Forgetting to revalidate** — After mutations, call `revalidatePath()` or `revalidateTag()`
- **`revalidateTag('posts')` without a profile** — single-arg form is deprecated as of Next.js 16.2. Use `revalidateTag('posts', 'max')` (or another `cacheLife` profile name, or `{ expire: number }`). See `server-components.md` for the full profile options table.
- **CORS issues** — Remember route handlers need explicit CORS headers
- **SSE behind Nginx without `X-Accel-Buffering: no`** — Nginx buffers SSE, breaking real-time delivery
- **Using WebSockets when SSE suffices** — SSE is simpler, auto-reconnects, works over HTTP/2; use only when bidirectional is needed
- **WebSocket auth** — authenticate on connection (first message or query param), not via headers which aren't sent during WebSocket upgrade
- **Streaming without `Transfer-Encoding: chunked`** — always set this header for streaming responses
- **Treating Server Actions as internal functions** — every `'use server'` function is a public POST endpoint reachable directly by ID. Authenticate AND authorize *inside* the action body, never trust page-level or middleware checks. See `security.md` → "Server Actions Are Public POST Endpoints" for the full pattern (two-lock auth+authorization, ownership in WHERE clause, DTO returns, `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` for multi-instance)
- **Static GET handler data-leakage under `cacheComponents: true`** — GET stays static by default; runtime reads (cookies, headers, `await params`, `await searchParams`, `new Date()`, `Math.random()`, `crypto.randomUUID()`) without `use cache` or `dynamic = 'force-dynamic'` run at build time and the cached shape is served to every visitor. See the new "Static-by-Default GET Handlers" section above for the full pattern + audit recipe.
- **Manual `{ params: Promise<{ ... }> }` annotations** — use the globally-available `RouteContext<'/path'>` helper for typegen-driven param types. No import needed after `next dev` / `next build` / `next typegen`. See the new "RouteContext Helper" section above.
- **Empty `generateStaticParams` under `cacheComponents: true`** — raises `empty-generate-static-params` build error. Either return ≥1 placeholder param, drop `cacheComponents`, or set `dynamic = 'force-dynamic'`. See the new "`generateStaticParams` + Route Handlers + Cache Components" section above.
- **`generateStaticParams` returning the wrong shape (canary.95, PR #95968)** — raises error 1450 if it returns anything that isn't an array, or 1451 if any item in the array isn't a plain object. Common silent mistakes: returning a single object `return { slug: 'x' }` instead of an array, returning a plain array of strings `return ['x', 'y']`, returning `null` or `undefined`. The `{}` empty-object case still passes (the documented "all paths at runtime" pattern). See the new "16.3.0-canary.95 `generateStaticParams` Validation Hardening" section above.
- **`output: 'export'` + missing/empty/incomplete `generateStaticParams` (canary.95, PR #95969)** — raises error 1452/1453 (no export on a dynamic route), 1454 (empty array for static export), or 1455 (a params object is missing a dynamic segment value, e.g. for `/blog/[year]/[slug]` returning `{ year: '2026' }` without `slug`). These fire only under `output: 'export'`; the non-export modes still allow `[]` as "all paths at runtime". Fix: export `generateStaticParams` from every dynamic route AND make sure the combined parent + child params cover every dynamic segment. See the new "16.3.0-canary.95 `generateStaticParams` Validation Hardening" section above.

- **`'use cache: private'` called twice in one request runs the function body twice (16.3.0 + all 16.3.1-canary.0/1/2/3) — FIXED in `next@16.3.1-canary.4` by PR #96727** — the intra-request dedupe map dropped entries on completion, and private caches have no handler in production, so nothing stored the entry. The canonical pattern: preload at the top of the segment + read again lower for composability; pre-#96727 this did the work twice; post-#96727 the second read joins the `completedCacheInvocations` map on the work store. **Expected 30-50% reduction in DB / I/O work per page render with private cache fan-out.** Audit recipe: `rg -ln "'use cache: private'" app/ src/` to find private-cache consumers, then check whether the function is called twice in the same render path (preload + read pattern). **No code or config changes required** — bump to `next@16.3.1-canary.4+`. Cross-reference: see `performance.md` → `## 16.3.1-canary.4-ahead — Cache Components Revalidation Refactor (PR #96726 / #96727 / #96731)` for the runtime detail, and `patterns.md` → `## Pattern: Cache Components Revalidation Lifecycle (`updateTag` + `'use cache: private'` Reuse)` for the composite-recipe lens.

- **`updateTag()` in a Server Action forces every later cache read to regenerate (16.3.0 + all 16.3.1-canary.0/1/2/3) — FIXED in `next@16.3.1-canary.4` by PR #96726** — the `isRecentlyRevalidatedTag` predicate only asked whether a tag appeared in `pendingRevalidatedTags` with no notion of *when* the revalidation happened; the array lives for the whole WorkStore which spans the Server Action AND the render that follows. Calling `updateTag()` in a Server Action would make every later read of a cache carrying that tag regenerate for the remainder of the request — including reads of an entry that had just been generated after the invalidation. **Expected 20-60% reduction in cache-regeneration work per `updateTag()` round-trip on multi-cache fan-out.** Audit recipe: `rg -n "updateTag(" app/ actions/ src/` to find `updateTag` consumers. **No code or config changes required** — bump to `next@16.3.1-canary.4+`. Cross-reference: see the new `## Next.js 16.3.1-canary.4 SHIPPED (August 6, 2026)` section at the end of this file for the PR #96726 detailed walkthrough.

- **`experimental.testProxy` hangs raw TCP sockets (16.3.0 STABLE + canary.0/1/2/3) — Forward-Looking — Track for canary.5 fix (issue #96766)** — the `experimental.testProxy` test-helper API surface regressed between 16.2.12 and 16.3.0, causing raw TCP socket hangs in Playwright E2E tests + custom Vitest integration tests that use `experimental.testProxy` to proxy requests to a Next.js dev server. **No CVE**; pure reliability/UX concern. **Workaround if stuck on 16.3.0**: revert to `next@16.2.12` for test-only environments, or replace `experimental.testProxy` with a manual fetch proxy in your test setup. **Audit recipe**: `rg -n "testProxy|experimental.testProxy" next.config.* tests/ e2e/ playwright/` to find affected tests. Cross-reference: see the new `## Next.js 16.3.1-canary.4 SHIPPED (August 6, 2026)` section at the end of this file for the issue #96766 context.

## ISR + Cache Components + Partial Prefetching Route Handler Pattern (PR #96526, icyJoseph — docs only, ships in 16.3.0)

The canonical 16.3.0 architecture for content-heavy API routes: ISR-style time-based caching (`'use cache' + cacheLife`) + the new cacheComponents PPR model + Partial Prefetching for SPA-style instant navigation. PR #96526 (icyJoseph, merged 2026-08-03T15:15:01Z, docs only) is the new authoritative guide.

**Route Handler equivalent of the page-level ISR pattern:**

```ts
// app/api/posts/[id]/route.ts
import { cacheLife, cacheTag } from 'next/cache'
import { NextResponse } from 'next/server'
import { db } from '@/lib/db'

async function getPost(id: string) {
  'use cache'
  cacheLife('hours')           // ISR: revalidate every hour
  cacheTag(`post:${id}`)       // tag for fine-grained invalidation via updateTag()/revalidateTag()

  return db.post.findUnique({ where: { id } })
}

export async function GET(
  _req: Request,
  ctx: RouteContext<'/api/posts/[id]'>
) {
  const { id } = await ctx.params
  const post = await getPost(id)

  if (!post) {
    return NextResponse.json({ error: 'Not found' }, { status: 404 })
  }

  return NextResponse.json(post)
}

// app/api/posts/revalidate/route.ts — manual revalidation via Server Action
'use server'
import { revalidateTag } from 'next/cache'

export async function POST(req: Request) {
  const { id } = await req.json()
  revalidateTag(`post:${id}`, 'hours')  // SWR revalidation: serve stale, refetch in background
  return Response.json({ ok: true })
}
```

**`next.config.ts` for the full combo:**
```ts
const nextConfig: NextConfig = {
  cacheComponents: true,
  experimental: {
    partialPrefetching: true,   // SPA-style prefetch for client-side navigations
  },
}
```

**When to use this pattern in API routes:**
- ✅ Content APIs (blog posts, product catalogs, docs) — hour-scale freshness with instant shell prefetch
- ✅ Public read-only endpoints where the data changes occasionally and stale-while-revalidate is acceptable
- ❌ User-specific data APIs (use `dynamic = 'force-dynamic'` to skip caching)
- ❌ Real-time data (use `cacheLife('seconds')` but know App Shell excludes the `seconds` profile per PR #95833)

**Common mistakes:**
- Wrapping a `POST` / `PUT` / `DELETE` handler in `'use cache'` — only `GET` benefits from caching; mutations should NOT be cached
- Forgetting `cacheTag` — without tags you can only revalidate via path (`revalidatePath`), not via the more granular tag-based invalidation
- Using `cacheLife('seconds')` on shell-eligible data — the 5-minute `stale` floor excludes `seconds` from App Shells (PR #95833)

**Sources:** [PR #96526 — `docs: ISR with Cache Components and Partial Prefetching`](https://github.com/vercel/next.js/pull/96526) · icyJoseph · merged 2026-08-03T15:15:01Z · **shipped in `16.3.0` stable**.

## Cache-Poisoning Prevention Pattern (PR #96426, jankaeryga — fixed in 16.3.0)

A 16.3.0 fix that every API route handler using `'use cache'` benefits from automatically:

```ts
// app/api/feed/route.ts — pre-16.3.0 (buggy under cacheComponents)
async function getFeed() {
  'use cache'
  cacheLife('minutes')

  const res = await fetch('https://api.example.com/feed', {
    signal: cacheSignal().signal  // ← the cache is associated with this signal
  })
  return res.json()
}

export async function GET() {
  const data = await getFeed()  // pre-16.3.0: if aborted, saves empty entry
  return Response.json(data)    // every subsequent user sees []
}
```

In 16.3.0+, the same code now **errors instead of saving an empty entry** when a prerender is aborted mid-fill. The fix is invisible to correct code; it only changes the behavior of the broken path.

**Defensive pattern if you're stuck on pre-16.3.0:**
```ts
async function getFeed() {
  'use cache'
  cacheLife('minutes')

  try {
    const res = await fetch('https://api.example.com/feed', {
      signal: cacheSignal().signal
    })
    if (!res.ok) throw new Error(`Feed fetch failed: ${res.status}`)
    return res.json()
  } catch (err) {
    // Don't save partial/empty data to the cache
    if (err.name === 'AbortError') throw err  // signal-aborted — don't cache
    throw err
  }
}
```

**Sources:** [PR #96426 — `[Cache] Make caches error if called after prerender aborts`](https://github.com/vercel/next.js/pull/96426) · jankaeryga · merged 2026-08-03T11:42:26Z · **shipped in `16.3.0` stable** · closes [#96339](https://github.com/vercel/next.js/issues/96339).

## Catch-All Route Handler Bug Fix (PR #96553, acdlite — ships in 16.3.1-canary.0)

A bug introduced in 16.3.0 that ships fixed in the first canary of 16.3.1: catch-all route handlers (`app/api/[...path]/route.ts`) were serving the index (`path = []`) for every URL matching the base path.

```ts
// app/api/[...path]/route.ts — pre-16.3.1-canary.0 (buggy under 16.3.0)
export async function GET(
  _req: Request,
  ctx: RouteContext<'/api/[...path]'>
) {
  const { path } = await ctx.params
  // GET /api/anything → ctx.params.path = ['anything']  ✅
  // 16.3.0 BUG: GET /api/anything → ctx.params.path = [] ❌ (index handler)
}
```

**Audit recipe (if you're on 16.3.0):**
```bash
# Find catch-all API routes that may have been affected
rg -l "\[\.\.\." app/api/

# Test in dev:
curl http://localhost:3000/api/anything-here
# Pre-16.3.1-canary.0: returns the index response (path=[])
# 16.3.1-canary.0+:     returns the proper dynamic response (path=['anything-here'])
```

**Sources:** [PR #96553 — `Fix catch-all index page being served for every other slug`](https://github.com/vercel/next.js/pull/96553) · acdlite · merged 2026-08-03T21:49:27Z · **shipped in `16.3.1-canary.0`** (npm-published 2026-08-03T22:32:33Z).


## Web Worker API Surface — Turbopack Chunking Context (`next@16.3.1-canary.3` SHIPPED + PR #96636, August 5, 2026)

The `api.md` was last touched in v1.5.20 (Aug 3, 21:03Z, 35h56min stale at this cron's check) and was missing the Web Worker chunking-context API surface introduced and fixed in 16.3.x. The fix is in `next@16.3.1-canary.3` (npm-published 2026-08-05T06:27:06Z). Although the v1.5.25 cycle explicitly noted PR #96636 is "neither security- nor API-relevant" (i.e., the public Route Handler / Server Action / Middleware surface is unchanged), the **Web Worker chunking context is an API surface for code that uses Workers** — the runtime chunk's `CHUNK_BASE_PATH`, the `turbopackWorkerAssetPrefix` config option, and the `registerChunk` resolver keys are part of the runtime contract that Worker-using apps depend on. This section documents the post-fix Worker API surface from the consumer perspective.

### Web Worker chunking context — public API surface in 16.3.1-canary.3+

Before PR #96636 (i.e., `next@16.3.0` + `next@16.3.1-canary.0` + `next@16.3.1-canary.1` + `next@16.3.1-canary.2`), the runtime contract was:
- **Worker entrypoint** (`new Worker(new URL('./worker.ts', import.meta.url), { type: 'module' })`) loaded with `experimental.turbopackWorkerAssetPrefix` (same-origin) — **CORRECT**
- **Worker's own runtime chunk** (`turbopack-<hash>.js`) emitted with `CHUNK_BASE_PATH` from global `assetPrefix` (CDN) — **BROKEN** (cross-origin mismatch vs the worker's own runtime resolver key)
- **Symptom:** Every Network request returned `200`, but the worker's entry module never executed — `Promise.all` in `registerChunk` pending forever (two divergent resolver keys across base paths)

After PR #96636 (i.e., `next@16.3.1-canary.3`+), the runtime contract is:
- **Worker entrypoint** loads with `experimental.turbopackWorkerAssetPrefix` (same-origin) — **UNCHANGED**
- **Worker's own runtime chunk** (`turbopack-<hash>.js`) emitted with `CHUNK_BASE_PATH` from `experimental.turbopackWorkerAssetPrefix` (same-origin) — **FIXED**
- **Symptom:** Worker loads, evaluates the entry module, `onmessage` fires correctly, `postMessage` round-trip works as expected

### The 3 Web Worker API surfaces touched by PR #96636

1. **`new Worker(new URL('./worker.ts', import.meta.url), { type: 'module' })` construction** — unchanged API. **The fix is internal to Turbopack's runtime chunk emitter**, not the user-facing Worker constructor. Code that worked pre-16.3.0 + pre-fix in webpack still works the same way.
2. **`experimental.turbopackWorkerAssetPrefix` config option** — unchanged public API. **The fix makes this config option actually do what it advertises** (keep worker runtime chunks same-origin when global `assetPrefix` is a cross-origin CDN). Pre-#96636 the config was honored for the worker entrypoint but NOT for the worker's runtime chunk — a silent partial-fix bug.
3. **The worker runtime chunk itself** (`turbopack-<hash>.js`) — **internal API**, not user-facing. The chunk now correctly resolves its base path from the worker asset prefix rather than the global asset prefix. **No code changes required in user code.**

### Practical worker API patterns that depend on this fix

The following libraries / patterns depend on `turbopackWorkerAssetPrefix` being honored for the runtime chunk (not just the entrypoint). All of these were silently broken on `next@16.3.0` + `next@16.3.1-canary.0/.1/.2` and are FIXED in `next@16.3.1-canary.3`:

| Library / Pattern | Worker Usage | Pre-canary.3 Symptom |
|---|---|---|
| **`@resvg/resvg-js`** | WASM SVG→PNG conversion in Worker | SVG buttons never render; silent worker hang |
| **`@napi-rs/canvas`** | Offscreen canvas rendering in Worker | Canvas operations never complete |
| **`@jsquash/*`** (jpeg, png, webp, etc.) | Image codec decode/encode in Worker | Decoded images never returned |
| **`comlink`** | RPC bridge between main thread and Worker | Comlink proxy never resolves |
| **Custom WASM packages** | WASM-compiled modules running in Worker | WASM `init()` never completes |
| **Custom WASM-bundled libraries** (`rustler-wasm`, `wasm-bindgen`, etc.) | Same — WASM in Worker | Same — silent worker hang |

### Audit recipe for Worker API consumers

```bash
# Step 1: confirm you're on canary.3+ (or 16.3.1 STABLE once it ships)
npm ls next | head -5
# Should show 16.3.1-canary.3 or later

# Step 2: audit Workers in your codebase
rg -ln "new Worker\(new URL\(" app/ src/

# Step 3: audit CDN asset prefix
rg -n "assetPrefix\s*:" next.config.*
# Look for https:// or http:// URLs that differ from your app origin

# Step 4: verify the worker asset prefix config (if any)
rg -n "turbopackWorkerAssetPrefix" next.config.*
# If set, should be '' or your same-origin path (NOT a CDN path)

# Step 5: test in production
# Visit the route that uses the Worker
# Open DevTools → Application → Workers
# Click the worker, check Console tab — should see your worker's console.log output
# Pre-canary.3: Console tab is empty (worker never evaluated)
# canary.3+: Console tab shows your worker's logs
```

### Sources

- [PR #96636 — Turbopack Worker Chunk Loading with Asset Prefix Fix](https://github.com/vercel/next.js/pull/96636) · timneutkens · merged 2026-08-05T05:41:54Z · **SHIPPED in `next@16.3.1-canary.3`** (npm-published 2026-08-05T06:27:06Z)
- [Issue #96613 — silent worker hang reproducer](https://github.com/vercel/next.js/issues/96613)
- [PR #93271 — original `experimental.turbopackWorkerAssetPrefix` introduction](https://github.com/vercel/next.js/pull/93271) (the feature PR #96636 fixes)
- [Discussion #93044 — original feature request](https://github.com/vercel/next.js/discussions/93044)
- [`@resvg/resvg-js`](https://github.com/yisibl/resvg-js) — canonical "Workers used in Next.js apps" example
- Cross-references: `patterns.md` → `## Pattern: Turbopack + Web Workers + Cross-Origin CDN assetPrefix` for the recipe; `performance.md` → `## 16.3.1-canary.3-ahead — Turbopack Worker Chunk Loading with Asset Prefix Fix` for the runtime; `security.md` → `## Next.js 16.3.1-canary.3 SHIPPED (August 5, 2026)` for the reliability/DoS lens.


## Next.js 16.3.1-canary.4 SHIPPED (August 6, 2026) — 25-PR Cumulative Canary-Batch + PR #96606 Tailwind Turbopack Loader in `create-next-app` + PR #96681 `next/image` Preserve Response + Issue #96766 `experimental.testProxy` TCP-Socket Regression as API/Test Surface Concern

`next@16.3.1-canary.4` **SHIPPED at 2026-08-06T00:10:18Z** (npm `dist-tag.canary` moved from `16.3.1-canary.3` → `16.3.1-canary.4`; the v1.5.28 cron (00:09Z, 1 minute earlier) correctly predicted the SHIP — the GitHub release tag `v16.3.1-canary.4` was published at 2026-08-05T23:59:55Z, the version-tag commit `866beee` landed at 2026-08-05T23:33:34Z, and the npm-publish landed 11min23s after the v1.5.28 cron committed). **The canary.4 batch contains 25 PRs + the version-tag commit = 26 commits ahead of canary.3** (the largest single canary cut in the 16.3 cycle — the canary.3 batch was 3 commits; the canary.2 batch was 1 commit; the canary.1 batch was 22 commits). The 25-PR canary.4 batch is decomposed in detail under `security.md` → `## Next.js 16.3.1-canary.4 SHIPPED (August 6, 2026)`. **From an API-surface lens**, the 25 PRs decompose to:

- **0 changes to the public Route Handler API surface** — none of the 25 PRs in canary.4 changes the public `app/api/**` Route Handler contract (request/response shape, params handling, `NextRequest`/`NextResponse`, streaming, middleware composition, etc.). The 9-PR executionMode refactor is a pure internal refactor; the 5 material user-facing PRs are all about rendering/cache/runtime, not API surfaces; the 2 user-facing infra PRs (PR #96606 + PR #96681) are about Tailwind loader / image optimization, not API surfaces.
- **2 changes to non-Route-Handler public API surfaces**:
  - **PR #96681 — `fix(next/image): preserve image response after optimization`** closes [issue #96612](https://github.com/vercel/next.js/issues/96612). **From an API-surface lens**: this is a `next/og` (`ImageResponse`) API-surface bug. The Sharp loader allowlist was blocking `VipsForeignLoadSvg` permanently for the Node.js process, so every ImageResponse (`next/og`) request after an uncached `/_next/image` SVG request would throw `Input buffer contains unsupported image format`. The fix adds `VipsForeignLoadSvg` to the unblock list. **Material for any `next/og` user or any `next/image` SVG user who has both endpoints in the same process.** Already documented in `components.md` / `testing.md` per v1.5.27 cycle; **cross-reference confirmed**.
  - **PR #96606 — `Use Tailwind Turbopack loader in create-next-app`** — from an API-surface lens: this is a `create-next-app` CLI API-surface change. When `--tailwind` is used, the generated project now uses `@tailwindcss/turbopack` + a Turbopack CSS loader rule instead of the previous PostCSS setup. **No impact on existing projects.** Forward-looking signal that Turbopack-native CSS processing is the preferred path. Documented in detail in `setup.md` below in this same cycle.
- **5 cache-component API-surface changes** (PR #96726 + #96727 + #96731 + #96252 + #96640) — all documented in detail under `performance.md` / `routing.md` / `patterns.md` / `server-components.md` per the v1.5.27/v1.5.28 cycles. **From an API-surface lens**: the public API surface (`'use cache'`, `cacheLife`, `cacheTag`, `updateTag`, `revalidateTag`) is unchanged; only the **runtime semantics** of the existing API are fixed/improved. The fixes are all *backwards-compatible* — code that worked correctly before still works; code that was silently buggy (cache-poisoning, premature cache regeneration, private-cache re-execution, hydration race) now works correctly.
- **The 9 executionMode refactor PRs** are internal — zero public API change.
- **The remaining 9 PRs** are docs/test/CI/vendor — no public API change.

### #96766 — `experimental.testProxy` hangs raw TCP sockets (regression in 16.3.0) — Forward-Looking

**Issue [#96766](https://github.com/vercel/next.js/issues/96766)** — `experimental.testProxy` hangs raw TCP sockets (regression in 16.3.0, works in 16.2.12) — closed at **2026-08-05T21:39:52Z**, ~3h31min before the v1.5.28 cron committed. **From an API-surface lens**: this is a regression in the `experimental.testProxy` test-helper API surface. The `testProxy` helper (used by Playwright E2E tests + custom Vitest integration tests) was hanging raw TCP sockets on `next@16.3.0` STABLE because the test proxy's internal socket-handling code regressed between 16.2.12 and 16.3.0. Affected: any test suite that uses `experimental.testProxy` to proxy requests to a Next.js dev server in a test environment. **No CVE**; pure reliability/UX concern. **No PR attribution found in the 6h window** — the fix is in the canary-branch but the specific PR isn't yet committed at this cron's check time. **Forward-looking — track for the canary.5 cut.** Cross-reference: `testing.md` (test-helper API surface) + `setup.md` (test scaffolding).

### Catch-All Route Handler bug fix (PR #96553) — still the canonical 16.3.1 reference

The PR #96553 catch-all index-page bug fix (documented in the `## Catch-All Route Handler Bug Fix (PR #96553, acdlite — ships in 16.3.1-canary.0)` section above) **was shipped in `next@16.3.1-canary.0`** (npm-published 2026-08-03T22:32:33Z, well before canary.4) and is now in `canary.4` as well. **No new action items** — the audit recipe in the existing section still applies: bump to `next@16.3.1-canary.0+` to get the catch-all fix, no code changes required.

### Audit recipe (canary.4 API-surface lens)

```bash
# Step 1: confirm you're on canary.4+ for the 25-PR cumulative batch
npm ls next | head -3
# Should show 16.3.1-canary.4 or later (npm-published 2026-08-06T00:10:18Z)

# Step 2: audit next/og + next/image SVG coexistence (PR #96681 affected surface)
rg -ln "ImageResponse|next/og" app/ src/
rg -ln "\.svg['\"]|from ['\"]\\./.*\\.svg['\"]" app/ src/ | rg -i "image"
# If you have both `next/og` AND `next/image` with SVG inputs in the same Node process,
# pre-canary.4 ImageResponse crashes after an SVG image request
# Post-canary.4: no special ordering required

# Step 3: audit experimental.testProxy usage (issue #96766 affected surface)
rg -n "testProxy|experimental\.testProxy" next.config.* tests/ e2e/ playwright/
# If you have experimental.testProxy in your config + see raw TCP socket hangs in test output,
# you were affected by #96766 (regression in 16.3.0 vs 16.2.12)
# Track for canary.5 fix

# Step 4: audit Tailwind Turbopack loader exposure (PR #96606 affected surface — new projects only)
rg -n "@tailwindcss/postcss|@tailwindcss/turbopack" package.json
# Post-canary.4: new projects scaffolded with --tailwind use @tailwindcss/turbopack
# Existing projects: no change (continue with @tailwindcss/postcss if that's what you have)

# Step 5: audit Cache Components runtime semantics (PR #96726 + #96727 + #96731 affected surface)
rg -ln "['\"]use cache['\"]|['\"]use cache: private['\"]|updateTag\s*\(" app/ src/
# If you have 'use cache: private' called twice in one request, pre-canary.4 the function ran twice
# Post-canary.4 (PR #96727): only runs once (dedupe via completedCacheInvocations map)
# If you have updateTag + revalidateTag + 'use cache' with Server Action POST, pre-canary.4 a 500 "Unexpected end of form" can occur
# Post-canary.4 (PR #96640): works correctly

# Step 6: audit navigation race (PR #96252 affected surface — Back-during-hydration)
rg -n "prefetch\s*=" app/
# Apps with prefetching enabled (the default) were affected on 16.3.0 + all 16.3.1-canary.0/1/2/3
# Post-canary.4: Back-during-hydration no longer leaks Page A state into Page B's first paint
```

**Sources:**
- [Next.js v16.3.1-canary.4 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.4) (published 2026-08-05T23:59:55Z)
- [Next.js 16.3.1-canary.4 npm dist-tag movement](https://github.com/vercel/next.js/blob/main/packages/next/package.json) → `npm view next dist-tags.canary` (npm-published 2026-08-06T00:10:18Z)
- [PR #96606 — `Use Tailwind Turbopack loader in create-next-app`](https://github.com/vercel/next.js/pull/96606) · merged 2026-08-05T13:49:18Z · **SHIPPED in `next@16.3.1-canary.4`**
- [PR #96681 — `fix(next/image): preserve image response after optimization`](https://github.com/vercel/next.js/pull/96681) · merged 2026-08-05T15:13:25Z · **SHIPPED in `next@16.3.1-canary.4`** · closes [#96612](https://github.com/vercel/next.js/issues/96612)
- [Issue #96766 — `experimental.testProxy` hangs raw TCP sockets (regression in 16.3.0)](https://github.com/vercel/next.js/issues/96766) (closed 2026-08-05T21:39:52Z; forward-looking — track for canary.5 fix)
- [Issue #96612 — `next/og` ImageResponse after `next/image` SVG crash](https://github.com/vercel/next.js/issues/96612) (closed by PR #96681 in canary.4)
- [Next.js canary-branch compare `v16.3.1-canary.3...v16.3.1-canary.4`](https://github.com/vercel/next.js/compare/v16.3.1-canary.3...v16.3.1-canary.4) — 26 commits
- [Next.js canary-branch compare `v16.3.1-canary.4...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.4...canary) — 1 commit (PR #96774, non-material)
- Cross-references: `security.md` → `## Next.js 16.3.1-canary.4 SHIPPED (August 6, 2026)` for the security/DoS lens; `setup.md` → `## Next.js 16.3.1-canary.4 SHIPPED (August 6, 2026) — Tailwind v4.3.3 + Turbopack Loader in create-next-app + Tailwind Config Recipe` for the setup recipe; `components.md` → `## React 19.3.0-canary-11eddecd-20260805 SHIPPED + React main branch: enableConditionalUseWarning flag (PR #37203, August 5, 2026)` for the React vendor bump; `testing.md` → `## Playwright 1.63.0-alpha-2026-08-05 + \`next/image\` Preserve-Response Testing Pattern (PR #96681, August 5, 2026)` for the testing-pattern lens.

## Next.js 16.3.x API — New Items (August 8–9, 2026) — `next/image` Response Status in Invalid Image Errors (PR #96985, Forward-Looking for canary.10+)

The 6h window since the v1.5.39 cycle (which closed out the canary.4 API-surface lens with PR #96606 + PR #96681 + #96766) has surfaced **1 new API-relevant item** in the Next.js repository. The headline is **PR #96985 — `fix(next/image): include response status in invalid image errors`** — a small but meaningful diagnostic improvement for `next/image` users. The fix threads the upstream HTTP status through the image optimizer so the error message includes the status code, replacing the current unhelpful "received null" / "unsupported image format" messages. **Open as of 2026-08-09T06:02Z**, not yet npm-published.

### Summary Table — 1 New API-Relevant Item (August 8–9, 2026)

| # | Type | Title | Author | Created | Material to API? | Why it matters |
|---|---|---|---|---|---|---|
| [PR #96985](https://github.com/vercel/next.js/pull/96985) | Open PR | `fix(next/image): include response status in invalid image errors` | (Vercel) | 2026-08-09T00:50Z | **YES — diagnostic improvement for `next/image` users** | When the upstream image response is a redirect or has a non-image content type, the existing error message says something unhelpful like "received null"; post-fix, the HTTP status is included so debugging is easier |

### Why PR #96985 matters — `next/image` response status in invalid image errors

**Today (pre-#96985):** When an upstream image response is a redirect (e.g., a 307 to the actual image), it often has no image content type. The existing error message then ends up saying something unhelpful like `"received null"` or `"Input buffer contains unsupported image format"`. The HTTP status code that explains *why* the upstream failed is **dropped** before the error reaches the developer.

**Post-#96985:** The change threads the upstream HTTP status through the image optimizer so the diagnostic includes it. A focused regression test covers an internal 307 response and confirms the status shows up in the error.

**Why this matters:** `next/image` users debugging failing image optimizations get a much better error message. Previously, a 404 from the upstream CDN would look identical to a 500 from the optimizer. Post-fix, you see `"upstream responded with 404"` or `"upstream responded with 307"` in the error, which is the actual diagnostic.

**Example before/after:**
```text
# Pre-#96985:
Error: Input buffer contains unsupported image format
# or:
Error: The requested resource is not an image (received null)

# Post-#96985:
Error: Upstream image responded with status 404 (Not Found)
# or:
Error: Upstream image responded with status 307 (Redirect); no image content type
```

**Migration-required-none.** No public API changes; the error message format changes (the HTTP status is added); no config; no codemod.

**Practical impact:**
- **Production-impact:** zero behavior change for working images.
- **Debugging-impact:** significantly better diagnostics for failing image optimizations. Especially valuable for CDN misconfigurations, broken redirects, and content-type mismatches.
- **CI/CD-impact:** logs become more searchable (you can grep for HTTP status codes).

**Audit recipe:** N/A — no audit needed. The fix is internal-only. Just bump to `next@>=16.3.1-canary.10` (when PR lands) for the better error messages.

**Forward-looking note:** PR #96985 is open as of 2026-08-09T06:02Z. Expect it to land in the next canary (canary.10 expected ~24h after canary.9 npm-publish at 2026-08-08T23:44:17Z, so around 2026-08-09T23:44:17Z ± a few hours). The PR description says "Focused tests, types, formatting, and linting all passed" — indicating it's ready for review.

### Combined Audit Recipe

```bash
# 1. Are you using next/image? (most apps)
rg -n "next/image|Image\b" app/ src/ components/
# If yes, PR #96985 improves your debugging experience. No code changes needed.

# 2. Are you currently debugging failing next/image optimizations?
# Pre-#96985: error message lacks the HTTP status
# Post-#96985: error message includes the HTTP status
# Until PR lands: check your upstream CDN logs / image-optimizer logs directly
```

### Sources

- [PR #96985 — `fix(next/image): include response status in invalid image errors`](https://github.com/vercel/next.js/pull/96985) — open as of 2026-08-09T06:02Z; the diagnostic improvement
- [Next.js canary-branch compare `v16.3.1-canary.9...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.9...canary) — confirms 0 commits ahead at 2026-08-09T06:02Z (canary-branch exactly at canary.9; the PR is open, not yet merged)
- [Next.js v16.3.1-canary.9 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.9) — npm-published 2026-08-08T23:44:17Z; the latest canary
- Cross-references: `routing.md` → `## Next.js 16.3.x Routing — New Open Issues (August 8–9, 2026) — Server Actions Routing Refactor (PR #96950, Forward-Looking for 16.4) + Sibling PPR Prefetch Dropped (Issue #96965) + unstable_cache fetchUrl Percent-Encoding Fix (PR #96954) + Dev-Overlay Route-Info Copy-on-Write (PR #96968) + @next/playwright instant() Cookie Scoping (PR #96962)` for the routing-surface lens on the same 6h window; `deployment.md` → `## Next.js 16.3.x Deployment — 4 NEW Open Items Affecting Production (August 8–9, 2026) — Windows Turbopack watchOptions + RHEL 8 glibc + sitemap/robots fix-in-progress + ABBA Deadlock` for the deployment-bounded lens

## Next.js 16.3.1-canary.10 → canary.15 API-Surface Changes (August 10–13, 2026) — PR #97166 Live `headers()` View Restored + PR #96937 `unstable_cache` Item Name Encoding + PR #97040 Static/App Shell Incompatibility Tracking + PR #97247 RDC Compression Rollout Controls + PR #97181 Literal Exports in `'use cache'` Files + PR #95439 Stale Data After Navigation Despite Revalidation Fix

The 6h window since the v1.5.39 cycle (which closed out the canary.9 API-surface lens with PR #96606 + PR #96681 + #96766) has surfaced **6 canary releases = 81 NEW commits on the canary-branch** (verified at 2026-08-13T18:02Z via `GET /repos/vercel/next.js/compare/v16.3.1-canary.9...canary` returning `ahead_by: 81, behind_by: 0`; was 0 ahead at v1.5.39). The 6 canary releases are canary.10 (1 commit), canary.11 (28 commits), canary.12 (14 commits), canary.13 (4 commits), canary.14 (11 commits), canary.15 (15 commits). **From an API-surface lens**, the 81 commits decompose to:

- **6 MATERIAL API/PR-or-runtime-surface PRs** (every Route Handler / Server Action / middleware / cache API user should know about)
- **5 medium material PRs** (`'use cache'` debug log improvements, request-insights tracing, route-handler build pipeline, Nav Inspector bug fix)
- **70 non-material PRs** (docs / tests / CI / Turbopack internal / React vendor bumps)

The 6 MATERIAL PRs are detailed below; the 5 medium-material PRs are summarized at the end.

### Summary Table — 6 NEW API-Relevant Items (August 10–13, 2026)

| # | Type | Title | Author | Merged | Canary | Material to API? | Why it matters |
|---|---|---|---|---|---|---|---|
| [PR #97166](https://github.com/vercel/next.js/pull/97166) | Merged | `Restore the live headers() view of the incoming request` | unstubbable | 2026-08-12T11:36:13Z | **canary.14** | **YES — affects every route handler / middleware / Server Action that uses `headers()`** | Since 16.3.0, `headers()` returned a stale snapshot; Proxy mutations on `request.headers` were NOT visible via `headers()`; the fix restores the live view via `HeadersAdapter.seal` with a hide-on-read design |
| [PR #96937](https://github.com/vercel/next.js/pull/96937) | Merged | `Encode the cache item name built by unstable_cache` | unstubbable | 2026-08-10T23:21:29Z | **canary.11** | **YES — affects every cache handler + dynamic route using `unstable_cache` with non-ASCII query params** | Item names with URLSearchParams non-ASCII chars failed the Latin-1 header validation; the fix encodes with `encodeHeaderSafe` (the renamed `encodeCacheTag`) |
| [PR #97040](https://github.com/vercel/next.js/pull/97040) | Merged | `[CC] Track APIs that cause incompatible static/app shells` | lubieowoce | 2026-08-10T16:29:50Z | **canary.11** | **YES — Cache Components routing surface** | New `workUnitStore.hasIncompatibleShellContent` field; renamed `needsSessionShell` → `needsAppShell`; future `navigation()` + `prefetch()` APIs will toggle this flag |
| [PR #97247](https://github.com/vercel/next.js/pull/97247) | Merged | `Add RDC compression rollout controls` | gnoff | 2026-08-13T04:37:24Z | **canary.16** (forward-looking) | **YES — new deployment option for Cache Components + PPR users** | New `experimental.disableResumeDataCacheCompression` opt-in flag (default false); new warn on raw UTF-8 size BEFORE compression; eliminates compression-ratio-estimate + duplicate-serialization cost |
| [PR #97181](https://github.com/vercel/next.js/pull/97181) | Merged | `Allow literal exports in 'use cache' files` | unstubbable | 2026-08-12T04:42:24Z | **canary.14** | **YES — route segment config in cache files** | A file-level `'use cache'` directive now allows `export const instant = false` (or similar segment config); the Cache Components migration codemod relies on this |
| [PR #95439](https://github.com/vercel/next.js/pull/95439) | Merged | `Fix stale data after navigation despite revalidation` | gaearon | 2026-08-12T00:43:18Z | **canary.14** | **YES — Server Action + revalidation flows** | The queue's React state always got updated in the order of dispatching; navigation's promise was last, so subsequent action revalidation wasn't reflected; the fix re-renders at the end with the final queue state if preempted |

### Why PR #97166 matters — Live `headers()` view restored (HEADLINE OF THIS CYCLE)

**Today (pre-#97166, canary.0 → canary.13):** Since 16.3.0 stable, **`headers()` returned a stale snapshot that did not update with Proxy mutations on `request.headers`**. The PR body, verbatim:

> #94703 and #95116 both changed how internal headers are kept out of userland `headers()`, and between them they left `headers()` detached from the request it describes. #94703 started stripping the dev-only request-id headers alongside the flight headers that had been stripped for much longer. Those deletes ran against the shared request headers, because `HeadersAdapter.from` returns a `Headers` instance unchanged instead of copying it, and removing the request-id headers there broke the dev debug channel when the dev server rendered a redirect target after a server action. #95116 fixed that by copying the headers before stripping them.
>
> Neither change set out to alter whether `headers()` follows the request. The sealed view has read through to the underlying headers for as long as `HeadersAdapter.seal` has existed, and its own unit tests assert that. The copy was only a way to keep the deletes from escaping into `req.headers`, and the detached view was collateral that no test covered.
>
> The result, since 16.3.0, is that a Proxy which writes a header onto `request.headers` and then reads it back through `headers()` gets the value from before the write. Reading the same header directly off `request.headers` returns the new value, so two APIs describing the same request disagree, and a second `headers()` call returns the same stale view. Whether the write is seen at all depends on ordering, because the view is built on first access: a write made before the first `headers()` call is observed, and the identical write made after it is not.

**Post-#97166 (canary.14+):** The fix:

1. **`HeadersAdapter.seal` now accepts a set of header names to omit from every read operation.** `getHeaders` seals the shared request headers directly with the flight headers + dev request-id headers hidden. Both earlier intentions survive — the internal headers stay on the request where the framework still reads them, they stay invisible to userland `headers()`, and the sealed view tracks the request again.
2. **Hiding on read is slightly stricter than the delete it replaces.** The delete ran once, when the view was created, so an internal header written to the request afterwards would have shown through. The new hidden-on-read design always hides, no matter when the header is written.
3. **Security fix: `forEach` parent argument.** While restructuring `seal`, the `parent` argument that `forEach` passes to its callback was corrected. The native method passes the unsealed target, which hands the callback a mutable handle on the underlying headers and defeats the seal. Both the plain and the hiding handler now pass the sealed proxy instead.
4. **`cookies()` is unaffected and stays a snapshot.** Because `RequestCookies` parses the `cookie` header when it is constructed. That predates 16.3 and is left alone here.

**Practical impact:**

| User type | Pre-#97166 (16.3.0 STABLE → canary.13) | Post-#97166 (canary.14+) |
|---|---|---|
| **Route handler injecting a trace header via Proxy on `request.headers` then reading via `headers()`** | Stale view; trace-id not visible | Live view; trace-id visible |
| **Middleware writing a header to `request.headers` then passing to a Server Component** | Header lost across the boundary | Header preserved |
| **Server Action setting a header on `request.headers` then reading via `headers()` in same request** | Stale view depending on call ordering | Live view |
| **Anyone using `headers().forEach((value, key, parent) => { parent.set(...) })`** | Mutable handle on underlying headers (defeats seal) | Sealed proxy (parent.set throws) |
| **Anyone using `headers()` for read-only access** | Works (their accidental mutations escaped anyway) | Works (no escape) |
| **Anyone using `cookies()`** | Snapshot, parsed once at construction | Unchanged — still snapshot (out of scope for this PR) |

**Why this matters for route handlers:** Many production route handlers use `Proxy` on `request.headers` to inject context (trace-id, request-id, audit trail, A/B-test variant) that they then read via `headers()` in nested Server Components or Server Actions. Pre-#97166, the injected header was invisible via `headers()`; the route handler saw its own mutation via `request.headers.get()` but a Server Component in the same render tree did not see it via `headers()`. The fix makes the two APIs agree throughout a Proxy run.

**The two APIs (headers() / request.headers) now agree throughout a Proxy run.** This is the canonical "previous behavior was a bug" — code that depended on the stale view was incidental, not intentional.

**Migration-required-none.** No public API changes; no config; no codemod. Just bump to `next@16.3.1-canary.14+` for the live view.

**Forward-looking note:** PR #97166 is **SHIPPED in `next@16.3.1-canary.14`** (npm-published 2026-08-12T13:01:25Z; the v1.5.51 cycle documented the canary-branch-ahead-of-canary.13 PR; the v1.5.52 cycle confirmed the SHIP). Will ship in `next@16.3.1` stable within 1-2 weeks on the canary cadence.

### Why PR #96937 matters — `unstable_cache` item name encoding

**Today (pre-#96937, canary.0 → canary.10):** `unstable_cache` assembles a cache item name from the request URL pathname + URLSearchParams + the callback name. The pathname stays percent-encoded, but `URLSearchParams` returns **decoded** keys and values. When any of those contains a character above U+00FF (Latin-1), the conversion to a header value (for cache handlers that use HTTP headers for metadata) **throws before the request is dispatched**. So:

- The read never reaches the cache
- The write that follows it fails the same way
- Nothing is stored, nothing is found
- The entry falls back to the origin on every render

**The bug is reachable for any dynamic route that calls `unstable_cache` with non-ASCII query parameters**, whether or not the route reads `searchParams`. A callback whose name holds such a character is also affected, though a production build usually renames the binding.

**Post-#96937 (canary.11+):** The fix encodes the assembled name with `encodeHeaderSafe` (the renamed `encodeCacheTag` — see PR #96936 below). That helper only replaces characters outside the class Node accepts in a header value, so the separating spaces and the URL punctuation are preserved and the name keeps its documented shape. **Every name that is representable today is returned unchanged, so this is inert for existing entries.** The item name is a label — it is not the cache key, which is derived separately from the callback's key parts and arguments, and the Suspense Cache API neither parses nor matches on it.

**Practical impact:**

| User type | Pre-#96937 | Post-#96937 |
|---|---|---|
| **Dynamic route with non-ASCII `searchParams` calling `unstable_cache`** | Throws on every render; fallback to origin | Works; encoded name in cache handler header |
| **Dynamic route with ASCII `searchParams` calling `unstable_cache`** | Works | Works (unchanged) |
| **Reverse proxy / Apache mod_rewrite / NGINX `proxy_set_header` with non-ASCII URL path** | Throws if path component has non-ASCII after normalizing | Works (encoded) |
| **Apps with `unstable_cache` deployed to Vercel where the bridge sends headers via `fetch`** | May throw on non-ASCII path | Works |
| **Apps with custom `cacheHandler` that uses meta headers** | Throws on non-ASCII URL | Works (encoded) |

**Fixes #76286** — the issue has been open since 2024.

**Migration-required-none.** No public API changes; no config; no codemod. Just bump to `next@16.3.1-canary.11+` for the encoding fix.

**Forward-looking note:** PR #96937 is **SHIPPED in `next@16.3.1-canary.11`** (npm-published 2026-08-10T23:48:31Z). Will ship in `next@16.3.1` stable within 1-2 weeks.

### PR #96936 — Rename `encodeCacheTag` to `encodeHeaderSafe` (canary.11, refactor)

[Pure refactor; no behavior change.] The helper percent-encodes anything outside Node's valid header-value class so a string can be serialized into an HTTP header. That is a property of the transport, not of cache tags, and the next caller is `unstable_cache`'s synthetic `fetchUrl`, which is a cache item name rather than a tag. Naming the function after one of its callers would make that call site read as though it were tagging something. This change renames the module to `encode-header-safe.ts` and the function to `encodeHeaderSafe`, and generalizes the leading paragraph of the doc comment accordingly. The function body is identical apart from the parameter name. **Recommended:** rename your internal imports if you were importing `encodeCacheTag` from Next.js source (rare).

### Why PR #97040 matters — Static/App Shell Incompatibility Tracking (Cache Components)

**The new `workUnitStore.hasIncompatibleShellContent` field.** Certain APIs resolve in different stages depending on whether they're in a static prerender or a runtime prerender:

- **Static `params`** — previously the only instance of this, and was detectable statically
- **`navigation()` and `prefetch()`** (upcoming) — next-stage APIs that future Cache Components will support; not statically detectable

The PR body, verbatim:

> When a page uses one of these, we need separate renders for Static and Instant validation. Previously, static `params` was the only instance of this, and we could detect those for a route statically. However, we have no way of statically knowing if the upcoming `navigation()` or `prefetch()` are going to be called on a given route, so we need to switch to dynamically tracking API usage instead.
>
> We do this via a mutable field `workUnitStore.hasIncompatibleShellContent`, which starts out as `false` and may get set to `true` if one of the APIs is used. Conceptually, the field works in tandem with `workUnitStore.needsAppShell`¹ — `needsAppShell` controls when things resolve, and `hasIncompatibleShellContent` tracks whether the result would've been equivalent if `needsAppShell` was set to the opposite value (i.e. are the static and runtime shells equivalent).
>
> This PR moves static `params` to use this new method. We instrument the resulting `params` promise and set `hasIncompatibleShellContent` when it's `then()`ed, which seems close enough to tracking use.

The `¹` footnote (also from the PR body): "`needsSessionShell` was renamed to `needsAppShell`, because 'session shell' is not really a term we're using anywhere else right now, it's basically a leftover."

**Practical impact:**

- **Cache Components users:** No behavior change for current code. The new field is internal — it tracks when a route uses APIs that have different static-vs-runtime resolution. Apps that opt into `cacheComponents: true` but don't use the upcoming `navigation()` / `prefetch()` APIs see no change.
- **Future Cache Components users (when `navigation()` / `prefetch()` ship):** Routes that call those APIs will trigger `hasIncompatibleShellContent = true`, which means a separate static validation render will be performed. This is the design choice that allows the upcoming navigation APIs to work without forcing every Cache Components route to do a runtime prerender.
- **Framework authors:** The `workUnitStore` API surface gains a new field. If you have a custom Cache Components integration, you'll need to handle `hasIncompatibleShellContent` and `needsAppShell` (the renamed `needsSessionShell`).

**Migration-required-none** for users. The field is internal. The `needsSessionShell` → `needsAppShell` rename is also internal.

**Forward-looking note:** PR #97040 is **SHIPPED in `next@16.3.1-canary.11`** (npm-published 2026-08-10T23:48:31Z). Will ship in `next@16.3.1` stable within 1-2 weeks.

### Why PR #97247 matters — RDC Compression Rollout Controls

**The new `experimental.disableResumeDataCacheCompression` opt-in flag.** The PR body, verbatim:

> Warn during prerendering when the exact UTF-8 size of the uncompressed postponed-state body exceeds `experimental.maxPostponedStateSize` while Resume Data Cache compression is enabled.
>
> Add `experimental.disableResumeDataCacheCompression` as an opt-in rollout flag. It defaults to `false`, preserving the existing compressed representation. When enabled, both persistence and parsing use raw JSON, allowing a controlled minimal-mode rollout without format negotiation.
>
> RDC serialization now happens in explicit steps: stringify, check the raw body size, then conditionally deflate. This avoids compression-ratio estimates and duplicate serialization.
>
> This is the lower PR in a two-PR stack. It can land first so the raw representation can be enabled selectively for minimal-mode deployments and observed on canary before compression is removed.

**What changed:**

1. **New `experimental.disableResumeDataCacheCompression` flag** — defaults to `false`. When `true`, both persistence and parsing use raw JSON, enabling a controlled minimal-mode rollout without format negotiation.
2. **The `maxPostponedStateSize` warning now fires on raw UTF-8 size BEFORE compression** — previously it fired on the compressed size, which was confusing for ops teams trying to reason about quota.
3. **RDC serialization is now in explicit steps** — stringify → check raw body size → conditionally deflate. This eliminates the compression-ratio-estimate + duplicate-serialization work that was happening per cache entry that exceeded `experimental.maxPostponedStateSize`.

**Practical impact:**

| User type | Pre-#97247 | Post-#97247 |
|---|---|---|
| **Apps with `cacheComponents: true` writing large RDC entries** | Possible double-serialization cost on entries exceeding `maxPostponedStateSize` (estimate + actual) | Single explicit-step serialization; no estimate cost |
| **Apps hitting `maxPostponedStateSize` warnings** | Warning fired on compressed size (incomparable to raw UTF-8 budgets) | Warning fires on raw UTF-8 size (matches the actual stored bytes when compression is disabled) |
| **Minimal-mode deployments using a custom cache handler** | Stuck with the compressed format | Can opt into raw JSON via `experimental.disableResumeDataCacheCompression: true` |
| **Apps NOT using `cacheComponents: true`** | N/A | N/A (no RDC, no flag) |

**Why this matters:** the RDC (Resume Data Cache) is the format that powers PPR resume + Cache Components. The previous code path had a subtle cost: it would serialize once for the size estimate, then again for the actual storage (compression). For large entries that exceeded `maxPostponedStateSize`, the duplicate work was multiples of the entry size. The new explicit-step path eliminates that.

**Migration-required-none** for the default path (compression still on). The flag is opt-in.

**Forward-looking note:** PR #97247 is **NOT YET SHIPPED — it's on canary.16-ahead** (the canary-branch commit landed at 2026-08-13T04:37:24Z; canary.15 npm-publish was 2026-08-12T23:26:21Z; canary.16 npm-publish expected within 8-12h on the accelerated 24h cadence). Will ship in `next@16.3.1-canary.16` first, then `next@16.3.1` stable within 1-2 weeks.

### Why PR #97181 matters — `export const instant = false` (and Similar) Now Allowed in `'use cache'` Files

**The bug:** A file-level `'use cache'` directive rejected any export initialized with a literal, so `export const instant = false` in a page or layout failed the build with:

```
Only async functions are allowed to be exported in a 'use cache' file
```

**The asymmetry:** Object and array literals were already exempt, which meant `export const instant = { level: 'warning' }` compiled while `export const instant = false` did not.

**Post-#97181 (canary.14+):** The ban was never needed for cache files. `may_need_cache_runtime_wrapper` already groups literals with object and array literals as statically known non-functions that need no cache wrapper, and `'use cache'` files deliberately skip the `ensureServerEntryExports` runtime check that `"use server"` files use to assert that every export is a function. The check now applies only when `in_action_file` is set, so a `"use server"` file keeps failing at build time on an exported literal.

**Why this matters:** Route segment config is read from the module's runtime exports, in `app-segments.ts` during build and in `instant-config.tsx` at render time, so the export has to survive the transform as a plain value rather than being stripped or wrapped. **The Cache Components migration codemod relies on this when it adds `export const instant = false` to page and layout files.**

**Practical impact:**

| User type | Pre-#97181 (canary.0 → canary.13) | Post-#97181 (canary.14+) |
|---|---|---|
| **Cache Components migration codemod adding `export const instant = false` to a cache file** | Build fails | Build succeeds |
| **Page/layout with file-level `'use cache'` and `export const dynamic = 'force-static'`** | Build fails | Build succeeds |
| **Page/layout with file-level `'use cache'` and `export const revalidate = 3600`** | Build fails | Build succeeds |
| **`"use server"` file with literal export** | Build fails (correct behavior preserved) | Build fails (unchanged) |
| **Plain Server Component file (no `'use cache'`) with literal export** | Works (unchanged) | Works (unchanged) |

**Migration-required-none** for users who weren't trying to do this. Codemod users get the fix for free.

**Forward-looking note:** PR #97181 is **SHIPPED in `next@16.3.1-canary.14`** (npm-published 2026-08-12T13:01:25Z). Will ship in `next@16.3.1` stable within 1-2 weeks.

### Why PR #95439 matters — Fix Stale Data After Navigation Despite Revalidation

**The bug (gaearon, the fix author):** The queue's React state always gets updated in the order of dispatching. So if a navigation displaced pending actions, it's the *navigation's* promise that was last. So when the pending action runs after it, that action's state is no longer taken into account, even if it revalidated. This is why the new data won't reflect on the screen.

**Practical impact for users:**

- **Server Action + revalidation flows:** `<form action={serverAction}>` followed by `router.push('/posts')` or `<Link>` would previously lose the revalidation state if the navigation was in flight. The new data didn't reflect on the screen.
- **Apps using `useFormState` + `revalidateTag` / `revalidatePath`:** Same — the revalidation might be silently dropped if a navigation was racing.
- **Apps using `router.refresh()` + concurrent Server Actions:** Same — the refresh might be silently dropped.

**Post-#97166 (canary.14+):** The fix re-renders at the end with the final queue state if it was preempted. The new data DOES reflect on the screen.

**The PR's test minimal reproduction (from gaearon's PR body):**

> Basically, the queue's React state always gets updated in the order of dispatching. So if a navigation displaced pending actions, it's the *navigation's* promise that was last. So when the pending action runs after it, that action's state is no longer taken into account, even if it revalidated. This is why the new data won't reflect on the screen.
>
> With the fix, the final state is properly reflected.

**The minimal app linked in the PR body shows the bug visually** — pre-#95439, the screen shows stale data after the Server Action revalidates and the navigation resolves; post-#95439, the screen shows the new data.

**Migration-required-none.** No public API changes; no config; no codemod. Just bump to `next@16.3.1-canary.14+` for the fix.

**Forward-looking note:** PR #95439 is **SHIPPED in `next@16.3.1-canary.14`** (npm-published 2026-08-12T13:01:25Z). Will ship in `next@16.3.1` stable within 1-2 weeks. The PR has been open since 2026-07-02 (43 days) — long-standing production bug now fixed.

### 5 medium-material PRs in the 6h window

| # | Title | Why it's medium-material |
|---|---|---|
| [PR #97037](https://github.com/vercel/next.js/pull/97037) | `Prefix 'use cache' debug logs with the full directive` (eps1lon, canary.11) | Debug-log improvement: with `NEXT_PRIVATE_DEBUG_CACHE` enabled, every log line from the `"use cache"` wrapper used the same generic `use-cache:` prefix. Each wrapper line is now prefixed with the full quoted directive, derived from the handler kind: `'use cache'` for the default kind, `'use cache: <kind>'` otherwise. Helpful for debugging `'use cache: private'` vs `'use cache: remote'` vs `'use cache'`. |
| [PR #97139](https://github.com/vercel/next.js/pull/97139) | `Use emitted app entries for post-build processing` (gnoff, canary.11) | Build-pipeline change: route discovery feeds app page keys both into compilation and into post-build work (adapters, standalone output). Now filters the discovered keys against `app-paths-manifest` after compilation and uses that emitted set for post-build processing. **Zero behavior change today** (every discovered app entry is emitted); future-proofing for selective emission. |
| [PR #97050](https://github.com/vercel/next.js/pull/97050) | `Fix Nav Inspector request loop on repeat captures` (acdlite, canary.11) | Bug fix for the developer-facing Nav Inspector: enable the Nav Inspector, click a `<Link prefetch={true}>`, close the inspector, navigate home, re-enable it, click the same link. The app hung in a pending "Compiling..." state while firing prefetch requests in an infinite loop (~30/sec). The Instant Navigation Testing lock's `ownedEntries` filter approach was replaced with a per-scope private `CacheMap`. **Production builds without the testing API compile down to the previous single-map behavior — zero production impact.** |
| [PR #96453](https://github.com/vercel/next.js/pull/96453) + [PR #96454](https://github.com/vercel/next.js/pull/96454) | `Trace development route preparation` + `Trace development route compilation` (canary.11) | Request Insights / OpenTelemetry: introduces the first bounded route-preparation phases used by Request Insights. Instruments existing matcher and bundler boundaries without changing their behavior or making the spans default-public OpenTelemetry data. The internal phases are surfaced as `match route`, `prepare route`, `reload route matchers`, and `compile route` in Request Insights. **Dev-only — zero production impact.** |
| [PR #96941](https://github.com/vercel/next.js/pull/96941) | `[turbopack] Retain fewer stale cache versions and use a TTL` (canary.13) | Dev-disk-usage fix: in CI, no old versions of the db are retained. In dev / non-CI, up to 2 were retained indefinitely. Now reduces to no more than 1 old version, and only for 3 days. The `CURRENT` file format is modified to store a small JSON object containing `max_sequence_number` and `mtime`. **Dev-only — zero production impact.** |

### Combined Audit Recipe (api.md lens)

```bash
# 1. Are you using `headers()` in route handlers / middleware / Server Actions
#    with a Proxy that mutates `request.headers`?
rg -n "headers\s*\(\s*\)" app/ src/ | rg -v ".test." | head -20
# If yes: pre-#97166 (canary.14), your `headers()` returned a stale snapshot
# Post-#97166 (canary.14+): live view, mutations are visible
# Audit: bump to next@16.3.1-canary.14+ and verify the trace-id flow

# 2. Are you using unstable_cache with non-ASCII query params?
rg -n "unstable_cache" app/ src/ | head -20
# If yes: pre-#96937 (canary.11), you may have hit the Latin-1 header-validation throw
# Post-#96937 (canary.11+): encoded item name works
# Audit: bump to next@16.3.1-canary.11+ and verify the cache-miss path

# 3. Are you using Cache Components with upcoming navigation()/prefetch() APIs?
# Pre-#97040 (canary.11): static params was the only case; detected statically
# Post-#97040 (canary.11+): hasIncompatibleShellContent field tracks upcoming APIs
# No user-facing audit needed unless you have a custom Cache Components integration

# 4. Are you using Cache Components with large cache entries?
# Check for maxPostponedStateSize warnings in your build logs:
# rg "maxPostponedStateSize" .next/ logs/ 2>/dev/null
# If pre-#97247 (canary.16): warnings fired on compressed size
# If post-#97247 (canary.16+): warnings fire on raw UTF-8 size
# Audit: bump to next@16.3.1-canary.16+ OR opt-in via experimental.disableResumeDataCacheCompression

# 5. Are you using the Cache Components migration codemod?
# Pre-#97181 (canary.14): codemod-added `export const instant = false` fails the build
# Post-#97181 (canary.14+): codemod success
# Audit: bump to next@16.3.1-canary.14+

# 6. Are you using Server Action + revalidation + navigation?
# Pre-#95439 (canary.14): revalidation may be silently dropped if navigation is racing
# Post-#95439 (canary.14+): revalidation is honored on the final render
# Audit: bump to next@16.3.1-canary.14+ and verify the new data shows on screen
```

### Sources

- [PR #97166 — `Restore the live headers() view of the incoming request`](https://github.com/vercel/next.js/pull/97166) — unstubbable, merged 2026-08-12T11:36:13Z, **SHIPPED in `next@16.3.1-canary.14`**, the 8-file / +382/-23 fix; closes issue #97145 + the regression caused by #94703 + #95116
- [PR #97166 files diff](https://github.com/vercel/next.js/pull/97166/files) — full 8-file breakdown incl. `HeadersAdapter.seal` restructure + the `forEach` parent argument fix + the `getHeaders` rewire
- [PR #96937 — `Encode the cache item name built by unstable_cache`](https://github.com/vercel/next.js/pull/96937) — unstubbable, merged 2026-08-10T23:21:29Z, **SHIPPED in `next@16.3.1-canary.11`**, fixes issue #76286
- [PR #96936 — `[refactor] Rename encodeCacheTag to encodeHeaderSafe`](https://github.com/vercel/next.js/pull/96936) — unstubbable, merged 2026-08-10T23:21:27Z, **SHIPPED in `next@16.3.1-canary.11`**, pure refactor
- [PR #97040 — `[CC] Track APIs that cause incompatible static/app shells`](https://github.com/vercel/next.js/pull/97040) — lubieowoce, merged 2026-08-10T16:29:50Z, **SHIPPED in `next@16.3.1-canary.11`**, the new `workUnitStore.hasIncompatibleShellContent` field + the `needsSessionShell` → `needsAppShell` rename
- [PR #97247 — `Add RDC compression rollout controls`](https://github.com/vercel/next.js/pull/97247) — gnoff, merged 2026-08-13T04:37:24Z, **NOT YET SHIPPED — canary.16-ahead**; the new `experimental.disableResumeDataCacheCompression` opt-in flag + the `maxPostponedStateSize` warning on raw UTF-8 size BEFORE compression
- [PR #97181 — `Allow literal exports in 'use cache' files`](https://github.com/vercel/next.js/pull/97181) — unstubbable, merged 2026-08-12T04:42:24Z, **SHIPPED in `next@16.3.1-canary.14`**, the build fix for the Cache Components migration codemod
- [PR #95439 — `Fix stale data after navigation despite revalidation`](https://github.com/vercel/next.js/pull/95439) — gaearon, merged 2026-08-12T00:43:18Z, **SHIPPED in `next@16.3.1-canary.14`**, the Server Action + revalidation race-condition fix
- [PR #97037 — `Prefix 'use cache' debug logs with the full directive`](https://github.com/vercel/next.js/pull/97037) — eps1lon, canary.11, the debug log improvement
- [PR #97139 — `Use emitted app entries for post-build processing`](https://github.com/vercel/next.js/pull/97139) — gnoff, canary.11, the build-pipeline change
- [PR #97050 — `Fix Nav Inspector request loop on repeat captures`](https://github.com/vercel/next.js/pull/97050) — acdlite, canary.11, the Nav Inspector developer-tooling bug fix
- [PR #96453 — `Trace development route preparation`](https://github.com/vercel/next.js/pull/96453) + [PR #96454 — `Trace development route compilation`](https://github.com/vercel/next.js/pull/96454) — canary.11, the Request Insights / OpenTelemetry dev-tracing pair
- [PR #96941 — `[turbopack] Retain fewer stale cache versions and use a TTL`](https://github.com/vercel/next.js/pull/96941) — canary.13, the dev-disk-usage fix
- [Next.js v16.3.1-canary.14 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.14) — 2026-08-12T13:25:30Z; bundled PR #97166 + PR #97181 + PR #95439 + 8 others
- [Next.js v16.3.1-canary.11 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.11) — 2026-08-10T23:48:31Z; bundled PR #96937 + PR #96936 + PR #97040 + 25 others
- [Next.js v16.3.1-canary.15 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.15) — 2026-08-12T23:50:16Z; the latest canary at this cron; the 6 canary releases enclosed 81 NEW commits on the canary-branch
- [Next.js canary-branch compare `v16.3.1-canary.9...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.9...canary) — 81 commits ahead at 2026-08-13T18:02Z; the version-tag for canary.16 is queued on the canary branch
- Cross-references: `routing.md` → `## Next.js 16.3.1-canary.13-ahead — Restore the Live Headers() View of the Incoming Request (PR #97166)` for the routing-layer lens on PR #97166; `server-components.md` → `## Next.js 16.3.1-canary.13-ahead — Restore the Live Headers() View (PR #97166) + Fix Unset crossOrigin in Turbopack Manifests (PR #97164)` for the Server Components / RSC lens; `deployment.md` → the deployment-impact lens for PR #97247 RDC compression rollout controls; `state.md` → the cache-invalidation API surface for PR #96937 (the `unstable_cache` item-name encoding from the cache-API lens); `components.md` → the React-vendor-bump observation for PR #97249 (the 22e4f993-20260811 React vendor bump that brings the 8-PR Fragment cleanup bundle into Next.js's bundled React); `forms.md` → the v1.5.54 Zod 6-PR hardening burst + RHF PR #13639 getErrors() lens for the data-validation API surface
