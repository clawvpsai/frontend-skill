# Deployment — Vercel, Docker, Node Adapter, Self-Hosted

## Vercel (Recommended for Next.js)

Vercel is the creator of Next.js — zero-config deployment, edge network, automatic ISR.

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy from project root
vercel

# Deploy to production
vercel --prod
```

### `vercel.json` (optional)

```json
{
  "buildCommand": "npm run build",
  "outputDirectory": ".next",
  "framework": "nextjs",
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "X-Content-Type-Options", value: 'nosniff' }
      ]
    }
  ]
}
```

### Environment Variables

Set in Vercel dashboard or CLI:
```bash
vercel env add NEXT_PUBLIC_API_URL
vercel env add NEXTAUTH_SECRET
```

## Vercel Connect (June 17, 2026) — Agent Tokens

If your Next.js app or agent needs to call third-party APIs (Slack, GitHub, Snowflake, Salesforce, Notion, Linear) **on behalf of users**, prefer Vercel Connect over storing provider tokens in `.env`. Connect mints short-lived, user-scoped, audit-logged tokens at request time. See `security.md` for the full pattern and rationale.

- Public beta as of Ship 2026
- Providers: Slack, GitHub, Snowflake, Salesforce, Notion, Linear (+ any OAuth/API)
- Docs: https://vercel.com/docs/connect
- Launched at: [Vercel Ship 2026](https://vercel.com/blog/vercel-ship-2026-recap)

## eve — Vercel Agent Framework (June 17, 2026)

**eve** is Vercel's open-source agent framework, launched at Ship 2026 and built on top of [Vercel Workflows](https://vercel.com/docs/workflows), [Vercel Sandbox](https://vercel.com/docs/sandbox), [AI Gateway](https://vercel.com/ai-gateway), and [Vercel Connect](https://vercel.com/connect). It is "filesystem-first": an agent is just a directory of files, and eve compiles it into a Vercel Function that runs with durable execution, sandboxed compute, human-in-the-loop approvals, subagents, and evals built in. It is the framework Vercel uses internally to run its own agents (d0, Vercel Agent, etc.).

> **Status:** Public beta. APIs, docs, and behavior may change before GA.

### The Smallest Agent

```bash
# Scaffold a new agent
npx eve@latest init my-agent
cd my-agent
npm run dev
```

The minimum viable agent is **two files**:

```
my-agent/
└── agent/
    ├── agent.ts            # Optional: model + runtime config
    └── instructions.md     # Required: the always-on system prompt
```

```ts
// agent/agent.ts
import { defineAgent } from "eve";
export default defineAgent({
  model: "anthropic/claude-opus-4.8",
});
```

```markdown
<!-- agent/instructions.md -->
You are a research assistant. Always cite sources. Be concise.
```

That is it — `npm run dev` starts a local agent. The HTTP API is at `POST /eve/v1/session` to create a session, then `POST /eve/v1/session/:id/turn` to send a message.

### Adding Tools

Each file in `agent/tools/` is one tool. The filename becomes the tool name the model sees:

```ts
// agent/tools/get_weather.ts
import { defineTool } from "eve/tools";
import { z } from "zod";

export default defineTool({
  description: "Get the current weather for a city.",
  inputSchema: z.object({
    city: z.string(),
  }),
  async execute(input) {
    return { city: input.city, condition: "Sunny", temperatureF: 72 };
  },
});
```

No registration step. Drop a file in `tools/`, the model sees `get_weather`, the framework wires it up.

### Adding Channels

```bash
# Add a channel to surface your agent where your users are
npx eve@latest add channel-slack
npx eve@latest add channel-web-nextjs
npx eve@latest add channel-discord
```

Each channel is just another file in `agent/channels/`. Slack renders approvals as buttons, Web Chat renders inline, Discord gets typing indicators — same agent, channel-appropriate affordances.

### Adding Skills (Markdown Playbooks)

```markdown
<!-- agent/skills/debugging.md -->
# Debugging a failed deploy

## Step 1
Run `vercel logs --follow` to capture the last 100 lines of production logs.
```

Skills are loaded **only when relevant** (skill matching is framework-driven), so the model gets focused guidance without carrying the full playbook in every prompt. Same pattern as [Vercel's Chat SDK skills](https://vercel.com/blog/chat-sdk-brings-agents-to-your-users) (`npx skills add vercel/chat`).

### What eve Gives You (vs. rolling your own)

| Concern | DIY (LangChain / Vercel AI SDK + glue) | eve |
|---|---|---|
| Durable execution | Roll your own queue / DB checkpoints | [Vercel Workflows](https://vercel.com/docs/workflows), built in |
| Sandboxed code execution | Fly machines / Modal / E2B | [Vercel Sandbox](https://vercel.com/docs/sandbox), built in |
| Per-user tool credentials | Roll your own token vault | [Vercel Connect](https://vercel.com/connect), built in |
| Model routing + fallbacks | Wire it up yourself | [AI Gateway](https://vercel.com/ai-gateway), built in |
| Human-in-the-loop approvals | Build the UI + state machine | Built in (renders as channel-native buttons) |
| Subagents | Roll your own dispatcher | Built in (other agent directories as children) |
| Evals | DIY | Built in |
| Observability | DIY | [Vercel Observability](https://vercel.com/docs/observability) — sessions, turns, tools, token usage |
| Deploy | Configure a server | `vercel deploy` (it is a Vercel project) |

### When to Reach for eve

- ✅ You are building a production agent that needs to survive deploys, call external APIs on behalf of users, and ship to channels (Slack, Discord, Web Chat)
- ✅ You want a Vercel-native stack (Workflows + Sandbox + Connect + AI Gateway) without gluing it together
- ✅ You have many agents and want a shared framework
- ❌ You are prototyping a one-off LLM call (use the [Vercel AI SDK](https://sdk.vercel.ai) directly)
- ❌ You need a model fine-tuning / RL framework (eve is for agents, not training)
- ❌ You need on-prem / air-gapped deployment (eve deploys to Vercel; local dev runs on Docker / microsandbox / just-bash)

### Connect to Existing Next.js App

```bash
# Add eve to an existing Next.js project
npm install eve@latest
mkdir -p agent
echo "You are a helpful assistant." > agent/instructions.md
```

The agent lives alongside your Next.js routes. The same project can serve both the marketing site (Next.js) and the agent (eve), and they share `vercel deploy`.

**Sources:**
- [Introducing eve (Vercel blog, June 17, 2026)](https://vercel.com/blog/introducing-eve)
- [Vercel Ship 2026 recap](https://vercel.com/blog/vercel-ship-2026-recap)
- [eve docs (vercel.com/docs/eve)](https://vercel.com/docs/eve)
- [eve on GitHub (vercel/eve)](https://github.com/vercel/eve)
- [The Agent Stack (Vercel blog)](https://vercel.com/blog/agent-stack)
- [Chat SDK — agents across Slack, Discord, GitHub, etc.](https://vercel.com/blog/chat-sdk-brings-agents-to-your-users)

## Chat SDK — Multi-Platform Chat Bots (Ship 2026)

**Chat SDK** (`npm i chat`) is the third pillar of the [Agent Stack](https://vercel.com/blog/agent-stack), alongside [eve](#eve--vercel-agent-framework-june-17-2026) and [Vercel Connect](#vercel-connect-june-17-2026--agent-tokens). It's a single TypeScript library for shipping the same chat bot to **Slack, Microsoft Teams, Google Chat, Discord, WhatsApp, Telegram, GitHub, and Linear** without rewriting platform-specific code. Open-sourced in February 2026, GA-positioned at Ship 2026 (June 17). Current version: **`chat@4.31.0`** (published June 16, 2026), with 70+ npm dependents.

### The Smallest Bot

```bash
npx create-chat-sdk@latest my-bot
cd my-bot
npm run dev
```

The CLI scaffolds a working Next.js bot (webhook route, `Chat` class config, `.env.example`) and lets you `npx chat-sdk add slack` (or `teams`, `discord`, `github`, etc.) to install platform adapters. The minimum viable bot is a single file:

```ts
// app/api/chat/route.ts
import { Chat } from "chat";
import { createSlackAdapter } from "@chat-adapter/slack";
import { createRedisState } from "@chat-adapter/state-redis";

const bot = new Chat({
  userName: "mybot",
  adapters: { slack: createSlackAdapter() },
  state: createRedisState(),
});

bot.onNewMention(async (thread, message) => {
  await thread.post(`Echo: ${message.text}`);
});
```

This single route handles Slack mentions, thread replies, edits, and the markdown-to-native formatting conversion for free. Add another platform by installing the adapter — no code changes:

```ts
// app/api/chat/route.ts
import { createDiscordAdapter } from "@chat-adapter/discord";
import { createTeamsAdapter } from "@chat-adapter/teams";

const bot = new Chat({
  userName: "mybot",
  adapters: {
    slack: createSlackAdapter(),
    discord: createDiscordAdapter(),
    teams: createTeamsAdapter(),
  },
  state: createRedisState(),
});
```

### Architecture

| Layer | Package | Purpose |
|---|---|---|
| Core | `chat` | Event routing, message lifecycle, JSX cards, emoji helpers, type-safe message formatting |
| Platform adapters | `@chat-adapter/{slack,teams,discord,whatsapp,telegram,github,linear,gchat}` | Webhook verification, platform API calls, native streaming |
| State adapters | `@chat-adapter/state-{redis,postgres,filesystem}` | Subscriptions, distributed locking, message dedup (default TTL 5 min) |

State adapters are **required** for production. `filesystem` is fine for dev; `redis` / `postgres` for multi-instance / serverless.

### When Chat SDK Is the Right Tool

- ✅ You want a single codebase that runs on Slack, Discord, Teams, GitHub, Linear, etc.
- ✅ You're building an AI agent that needs to live where users already work
- ✅ You need native streaming (Slack Block Kit streaming, Teams DM native) without per-platform work
- ❌ You only need one platform and don't care about future portability — direct Slack/Discord SDKs are fine
- ❌ You need real-time bidirectional WebSocket UX (e.g., voice agents) — Chat SDK is webhook-first, not socket-first

### Chat SDK + eve

Chat SDK composes with eve: eve handles the agent's reasoning loop, sandbox, and approvals; Chat SDK handles "where the user lives" (Slack, Teams, Discord). The combined pattern is "eve is the brain, Chat SDK is the channel":

```ts
// app/api/chat/route.ts
import { Chat } from "chat";
import { createSlackAdapter } from "@chat-adapter/slack";
import { runAgent } from "eve"; // or your own agent loop

const bot = new Chat({ userName: "agent", adapters: { slack: createSlackAdapter() } });

bot.onNewMention(async (thread, message) => {
  for await (const chunk of runAgent({ prompt: message.text })) {
    await thread.post(chunk);  // streamed, markdown-aware
  }
});
```

### Claude Managed Agents on Vercel (Ship 2026 Day Session)

At Ship 2026, Anthropic + Vercel demoed **Claude Managed Agents**: Anthropic hosts the agent loop, but every command the agent runs executes inside a Vercel Sandbox you own — keeping filesystem, processes, and network egress in your environment. Combined with Chat SDK, the pattern is: Chat SDK receives a Slack mention → forwards to Claude Managed Agent → Claude calls Vercel Sandbox to run code → streams the result back through Chat SDK. The user never leaves Slack; the agent code never leaves your Sandbox; the LLM never sees your secrets.

**Sources:**
- [Chat SDK: bring agents to your users (Vercel blog, March 19, 2026)](https://vercel.com/blog/chat-sdk-brings-agents-to-your-users)
- [Chat SDK on npm (`chat@4.31.0`)](https://www.npmjs.com/package/chat)
- [Chat SDK docs (chat-sdk.dev/docs)](https://chat-sdk.dev/docs)
- [Chat SDK adapter directory](https://chat-sdk.dev/adapters)
- [Vercel Ship 2026 recap (Agent Stack primitives)](https://vercel.com/blog/vercel-ship-2026-recap)
- [The Agent Stack (Vercel blog, June 17, 2026)](https://vercel.com/blog/agent-stack)
- [Chat SDK: tables + streaming markdown (Vercel changelog, March 6, 2026)](https://vercel.com/changelog/chat-sdk-adds-table-rendering-and-streaming-markdown)
- [Claude Managed Agents on Vercel (Ship 2026 session)](https://vercel.com/blog/vercel-ship-2026-recap)

## Docker (Self-Hosted / VPS)

For VPS deployment (e.g., ServerAvatar users who want full control):

### `.dockerignore`

Add a `.dockerignore` to prevent unwanted files from being copied into the Docker build context:

```
# Dependencies
node_modules
.npm

# Next.js build artifacts
.next
out

# Git
.git
.gitignore

# Development files
*.md
*.log
.env*
.dockerignore
Dockerfile
docker-compose*

# Testing
coverage

# IDE
.vscode
.idea

# OS
.DS_Store
Thumbs.db
```


### Dockerfile

```dockerfile
# Stage 1: Build
FROM node:24-alpine AS builder
WORKDIR /app

# Copy dependency manifests first — Docker caches this layer separately
COPY package.json package-lock.json* ./

# Install dependencies BEFORE copying source code
# This layer only rebuilds when package*.json changes, not on every code change
RUN npm ci

# Now copy the rest of the source
COPY . .

# Build — this runs after deps are installed
RUN npm run build

# Stage 2: Runtime
FROM node:24-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production

COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

EXPOSE 3000

ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

### Enable Standalone Output

```ts
// next.config.ts
const nextConfig = {
  output: 'standalone',
}
```

### Docker Compose (with PostgreSQL)

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: "postgresql://postgres:password@db:5432/myapp"
      NEXTAUTH_SECRET: "your-secret"
      NEXT_PUBLIC_API_URL: "http://localhost:3000"
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  postgres_data:
```

```bash
docker-compose up -d
docker-compose logs -f app
```

### Docker Healthcheck

Add a healthcheck so Docker, Kubernetes, and load balancers know when the app is ready:

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: "postgresql://postgres:password@db:5432/myapp"
      NEXTAUTH_SECRET: "your-secret"
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

**Why `depends_on` with `condition: service_healthy`?**
- Without it, `depends_on` only waits for the container to **start**, not to be **ready**
- `service_healthy` waits for the healthcheck to pass before starting dependent containers
- The app's healthcheck hits `/api/health` which verifies the DB connection

## Node Adapter (Traditional Server)

Use `@vercel/node` adapter for deploying to any Node.js server:

```bash
npm install @vercel/node
```

### Custom Server (rarely needed)

```ts
// server.ts — only for custom routing, adds overhead
import { createServer } from 'http'
import { parse } from 'url'
import next from 'next'

const dev = process.env.NODE_ENV !== 'production'
const app = next({ dev })
const handle = app.getRequestHandler()

app.prepare().then(() => {
  createServer((req, res) => {
    const parsedUrl = parse(req.url!, true)
    handle(req, res, parsedUrl)
  }).listen(3000, () => {
    console.log('> Ready on http://localhost:3000')
  })
})
```

## proxy.ts (Next.js 16 — Request Proxy)

**In Next.js 16, `middleware.ts` is deprecated in favor of `proxy.ts`.** The rename reflects that this file intercepts and proxies requests. Both files work during the deprecation period, but `proxy.ts` is the forward-looking name. See `routing.md` for the full migration guide.

### Basic proxy.ts Pattern

```ts
// proxy.ts (project root)
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export const proxy = async (request: NextRequest) => {
  // Add security headers
  const response = NextResponse.next()

  response.headers.set('X-Frame-Options', 'DENY')
  response.headers.set('X-Content-Type-Options', 'nosniff')
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin')

  return response
}

export const matcher = ['/((?!api|_next/static|_next/image|favicon.ico).*)']
```

### Protected Routes with Auth

```ts
// proxy.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { jwtVerify } from 'jose'

const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET!)

export const proxy = async (request: NextRequest) => {
  const token = request.cookies.get('session')?.value

  const protectedPaths = ['/dashboard', '/settings', '/admin']
  const isProtected = protectedPaths.some(p =>
    request.nextUrl.pathname.startsWith(p)
  )

  if (isProtected) {
    if (!token) {
      return NextResponse.redirect(new URL('/login', request.url))
    }

    try {
      await jwtVerify(token, JWT_SECRET)
    } catch {
      return NextResponse.redirect(new URL('/login', request.url))
    }
  }

  return NextResponse.next()
}

export const matcher = ['/((?!api|_next/static|_next/image|favicon.ico).*)']
```

**Key changes from `middleware.ts`:**
- File renamed: `middleware.ts` → `proxy.ts`
- Export renamed: `middleware` → `proxy`
- Function must be `async` (required in Next.js 16)
- `matcher` is a named export (can alternatively be placed in `next.config.ts`)

**Matcher alternative — in `next.config.ts`:**

```ts
// next.config.ts
const nextConfig: NextConfig = {
  matcher: ['/dashboard/:path*', '/admin/:path*'],
}
```

### Node.js Runtime by Default

`proxy.ts` runs on the **Node.js runtime** by default (as of Next.js 15.5). This means full access to Node.js APIs — `fs`, `crypto`, `Buffer`, `child_process`, etc.

| Edge Runtime | Node.js Runtime (Next.js 15.5+) |
|---|---|
| Limited APIs (fetch, crypto sub) | Full Node.js API access |
| No `Buffer`, no `fs` | `Buffer`, `fs`, `crypto`, `child_process` |
| Web Crypto API only | Node.js `crypto` module |
| No native addons | Native addons work |

**Edge runtime** is still available if you specifically need edge characteristics (cold starts, global distribution) — add `export const runtime = 'edge'` to `proxy.ts`.

**Sources:**
- [Next.js 16 upgrade guide — proxy](https://nextjs.org/docs/app/guides/upgrading/version-16)
- [Next.js proxy.ts migration](https://krishna-adhikari.com.np/blogs/next16-middleware-to-proxy)

## Nginx Reverse Proxy

For running Next.js behind Nginx on a VPS:

```nginx
# /etc/nginx/sites-available/my-app
upstream nextjs {
    server 127.0.0.1:3000;
    keepalive 64;
}

server {
    listen 80;
    server_name myapp.com;

    location / {
        proxy_pass http://nextjs;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Static assets — let Next.js handle these or serve directly
    location /_next/static {
        proxy_pass http://nextjs;
        proxy_cache_valid 200 60m;
        expires 60m;
        add_header Cache-Control "public, immutable";
    }
}
```

## Systemd Service

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=Next.js App
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/var/www/myapp
ExecStart=/usr/bin/node .next/standalone/server.js
Restart=on-failure
Environment=NODE_ENV=production
Environment=PORT=3000

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable myapp
sudo systemctl start myapp
sudo systemctl status myapp
```

## PM2 Process Manager

```bash
npm install -g pm2

# Start with cluster mode (uses all CPU cores)
pm2 start .next/standalone/server.js --name myapp

# Or with a config file
pm2 ecosystem

pm2 save
pm2 startup  # auto-restart on reboot
```

### `ecosystem.config.js`

```js
module.exports = {
  apps: [{
    name: 'myapp',
    script: '.next/standalone/server.js',
    instances: 'max',
    exec_mode: 'cluster',
    env: {
      NODE_ENV: 'production',
      PORT: 3000,
    },
  }],
}
```

## Environment Variables for Production

```bash
# .env.production — never commit this
DATABASE_URL="postgresql://user:pass@host:5432/db"
NEXTAUTH_SECRET="long-random-string"
NEXTAUTH_URL="https://myapp.com"

# Public vars (safe to commit pattern)
NEXT_PUBLIC_APP_URL="https://myapp.com"
NEXT_PUBLIC_API_URL="https://api.myapp.com"
```

**Load order:** `.env` → `.env.local` → `.env.[mode]` → `.env.[mode].local`
Production uses `.env.production` + `.env.production.local`

## Health Check Endpoint

```ts
// app/api/health/route.ts
import { NextResponse } from 'next/server'

export async function GET() {
  try {
    // Optional: check DB connection
    await db.$queryRaw`SELECT 1`
    return NextResponse.json({ status: 'ok', timestamp: new Date().toISOString() })
  } catch {
    return NextResponse.json({ status: 'error' }, { status: 503 })
  }
}
```

## Build Optimization

```bash
# Check bundle size
npm run build 2>&1 | grep -E "(Route|Size|L First Load JS)"

# Analyze bundle
npm install @next/bundle-analyzer
npx ANALYZE=true npm run build
```

## CI/CD with GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      # Deploy to server via SSH or Vercel API
```

## Common Mistakes

- **`output: 'standalone'`** not set — Docker build won't work without it
- **Missing `NEXT_PUBLIC_` prefix** for client vars — double-check what's exposed
- **Not setting `NEXTAUTH_URL`** in production — causes redirect loops
- **Port mismatch** — ensure `PORT` env var matches Nginx/docker config
- **No health check** — Kubernetes/load balancers need `/api/health` to know if the app is ready
- **Using `middleware.ts` instead of `proxy.ts`** in Next.js 16 — the old filename works but is deprecated

## Next.js 16 Deployment Notes

Next.js 16 is current. Key deployment considerations:

### `next lint` Removed
Use Biome or ESLint directly:
```bash
# Biome (recommended — fastest)
npx biome check --write

# ESLint
npx eslint . --fix
```

### Turbopack Production Builds
Next.js 16 uses Turbopack for production builds by default:
```bash
# next build uses Turbopack automatically in Next.js 16
npm run build

# Force Webpack if needed (rare)
NEXT_BUILD_USE_WEBPACK=1 npm run build
```

### PPR (Partial Prerendering) Deployment
PPR is stable in Next.js 16. Enable it for improved TTFB:
```ts
// next.config.ts
const nextConfig: NextConfig = {
  cacheComponents: true,  // Enables PPR
}
```

PPR requires Suspense boundaries around dynamic content — if you don't have them, Next.js will warn during build.

### Caching Changes
Next.js 16 removed implicit caching. For self-hosted deployments, ensure your caching strategy uses `use cache` + `cacheTag` for data functions. For invalidation, use `revalidateTag` (background refresh) or `updateTag` (immediate expiration). See `server-components.md` for details.

### Next.js DevTools MCP Server (Next.js 16.2+)
Next.js 16.2+ ships with a built-in MCP (Model Context Protocol) server for AI-assisted development. It enables AI coding assistants (Claude, Cursor, etc.) to interact directly with your Next.js project's dev server for tasks like:
- Reading route configuration and file structure
- Inspecting component trees and data fetching patterns
- Running Next.js CLI commands and reading output
- Analyzing bundle stats and performance metrics

**Setup:**
```bash
# In your Next.js project, the MCP server starts automatically with:
npm run dev

# Or manually via the Next.js CLI:
npx next dev --mcp

# Configure AI clients (Claude Desktop, Cursor, etc.) to connect to:
# localhost:3000/.next/mcp (or your dev server URL)
```

**Security note:** The MCP server exposes project introspection endpoints. Only enable it in local development, not in production deployments. Use `NEXT_MCP_ENABLED=false` to disable if needed.

**Package:** `next-devtools-mcp` v0.3.10 (verified on npm)

See: [Next.js DevTools MCP guide](https://nextjs.org/docs/app/guides/mcp) · [MCP Servers directory](https://mcpservers.org/servers/vercel/next-devtools-mcp)

### Next.js 16.2 Adapter API (Stable)

Next.js 16.2 stabilizes the **Adapter API** — a first-class, documented interface for deploying Next.js to any platform. Previously, platforms like Cloudflare, Netlify, and AWS had to reverse-engineer Next.js's build output. Now they implement a standard adapter contract.

**Who supports it:** Vercel, Netlify, Cloudflare Workers, AWS Amplify, Google Cloud — all signed the same public contract.

**Why it matters:** Write once, deploy anywhere. No more platform-specific workarounds or fear of breaking changes when Next.js updates internal build output format.

#### How Adapters Work

Adapters transform the Next.js build output for a specific deployment target. The most mature option for Cloudflare Workers:

```bash
# Install the Cloudflare adapter (verified on npm as @opennextjs/cloudflare v1.19.11)
npm install @opennextjs/cloudflare wrangler

# Build with the adapter
OPENNEXT_ADAPTER=@opennextjs/cloudflare npm run build

# Deploy to Cloudflare Workers
wrangler deploy
```

**For other platforms** — check their official adapters (Vercel, Netlify, etc. maintain their own):
- Cloudflare: `@opennextjs/cloudflare` (verified on npm)
- Other platforms: refer to their official deployment guides

#### Adapter API Reference

The adapter interface handles these concerns:

| Concern | What the Adapter Does |
|---|---|
| **Routing** | Maps incoming requests to Next.js route handlers |
| **Runtime** | Wraps the Next.js runtime for the target environment |
| **Output** | Formats build artifacts for the target platform |
| **Caching** | Integrates with the target's caching layer |

**Note:** Platform-specific adapters (Cloudflare, Vercel, Netlify) are maintained by each platform's team. Only install adapters from sources you trust. For self-hosted VPS deployments, the standalone Docker/PM2 approach above is recommended.

**Sources:**
- [Next.js deployment guides](https://nextjs.org/docs/app/guides/deploying-to-platforms)
- [Next.js self-hosting guide](https://nextjs.org/docs/app/guides/self-hosting)
- [Next.js MCP guide](https://nextjs.org/docs/app/guides/mcp)
- [Next.js 16 release notes](https://nextjs.org/blog/next-16)
- [Next.js 16.2 release notes](https://nextjs.org/blog/next-16-2)
- [OpenNext adapter for Cloudflare](https://opennext.js.org/)
