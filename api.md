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
