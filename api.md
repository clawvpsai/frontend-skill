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
import { db } from '@/lib/db'

const CreatePostSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1),
  published: z.boolean().default(false),
})

export async function createPost(formData: FormData) {
  const parsed = CreatePostSchema.parse({
    title: formData.get('title'),
    content: formData.get('content'),
    published: formData.get('published') === 'on',
  })
  
  await db.post.create({ data: { ...parsed, authorId: 'current-user-id' } })
  
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

  async function handleSubmit(e: React.FormEvent) {
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

## Common Mistakes

- **Route handlers vs API routes in Pages Router** — App Router uses `app/api/` with route handlers; don't mix with `pages/api/`
- **Missing `await` on params** — In Next.js 15, route handler params are Promises
- **Not returning proper status codes** — 201 for create, 204 for delete, 404 for not found
- **Forgetting to revalidate** — After mutations, call `revalidatePath()` or `revalidateTag()`
- **`revalidateTag('posts')` without a profile** — single-arg form is deprecated as of Next.js 16.2. Use `revalidateTag('posts', 'max')` (or another `cacheLife` profile name, or `{ expire: number }`). See `server-components.md` for the full profile options table.
- **CORS issues** — Remember route handlers need explicit CORS headers
- **SSE behind Nginx without `X-Accel-Buffering: no`** — Nginx buffers SSE, breaking real-time delivery
- **Using WebSockets when SSE suffices** — SSE is simpler, auto-reconnects, works over HTTP/2; use only when bidirectional is needed
- **WebSocket auth** — authenticate on connection (first message or query param), not via headers which aren't sent during WebSocket upgrade
- **Streaming without `Transfer-Encoding: chunked`** — always set this header for streaming responses
