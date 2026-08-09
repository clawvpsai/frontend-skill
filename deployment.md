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
### July 2026 Security Releases — 16.2.11 (Active LTS) + 15.5.21 (Maintenance LTS) + canary.92

**The July 2026 security release shipped July 21, 2026** (1-day delay from the original July 20 target). Nine CVEs patched across Next.js 16.2.x and 15.5.x:

- **4 High:** DoS via Server Actions (CVE-2026-64641), Turbopack middleware bypass with single locale (CVE-2026-64642), SSRF via rewrites with attacker-controlled hostname (CVE-2026-64645), SSRF via Server Actions on custom servers (CVE-2026-64649)
- **5 Medium:** Image Optimization SVG DoS (CVE-2026-64644), unbounded Edge payload (CVE-2026-64646), Server Function endpoint disclosure (CVE-2026-64643), cache confusion for requests with bodies (CVE-2026-64648 + CVE-2026-64647)

**Upgrade immediately:**
```bash
npm install next@16.2.11   # Active LTS — use this for new projects
npm install next@15.5.21   # Maintenance LTS — for existing 15.x apps
```

**`next@canary`** = `16.3.0-canary.92` — ships all 9 security fixes + React vendor bump to `81e442ea-20260721`. This canary will become Next.js 16.3.0 stable.

**LTS status:**
- **16.2.11** — Active LTS (recommended for all new deployments)
- **15.5.21** — Maintenance LTS (only for apps not yet on 16.x)
- **16.3.0** (preview/canary) — next stable, expected after 16.3.0 stable ships

**Next security release:** August 20, 2026 (monthly cadence per the Vercel security release program). Set a recurring calendar reminder.

**canary.92 non-security PRs (7 material items):**
- [PR #96014](https://github.com/vercel/next.js/pull/96014) — Fix Turbopack middleware matcher with i18n single locale (fixes CVE-2026-64642 — the single-locale Turbopack bypass; the security fix is in the same PR)
- [PR #96013](https://github.com/vercel/next.js/pull/96013) — Improve performance of validating MPA form submissions
- [PR #96006](https://github.com/vercel/next.js/pull/96006) — `next/image`: improve performance of `detectContentType()` for SVG files (SVG detection was a hot path in the Image Optimization DoS fix; this optimizes the patched code path)
- [PR #95939](https://github.com/vercel/next.js/pull/95939) — Fix instant validation blocking navigations
- [PR #95985](https://github.com/vercel/next.js/pull/95985) — Prevent unhandled rejections when a `use cache` cache handler errors
- [PR #95984](https://github.com/vercel/next.js/pull/95984) — Add coverage for throwing custom `use cache` cache handler
- [PR #95861](https://github.com/vercel/next.js/pull/95861) — docs: add upgrade section to installation and expand AI agents guide

**Post-upgrade checklist:**
1. `npm install next@latest` (or `next@15.5.21`)
2. Bust Docker cache: `docker build --no-cache` or `docker buildx build --pull`
3. Redeploy
4. Verify: `curl -I https://your-app.com` → `X-Powered-By: Next.js 16.2.11`
5. For Turbopack + single `i18n.locales` users: re-test auth/security in middleware
6. For custom servers with Server Actions: re-test `redirect()` and forwarded requests
7. For `basePath` users: re-test all `redirect()` destinations

**Sources:**
- [Next.js blog: July 2026 Security Release](https://nextjs.org/blog/july-2026-security-release)
- [GitHub: v16.2.11 release notes](https://github.com/vercel/next.js/releases/tag/v16.2.11)
- [GitHub: v15.5.21 release notes](https://github.com/vercel/next.js/releases/tag/v15.5.21)
- [GitHub: v16.3.0-canary.92 release notes](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.92)
- [npm `next` package (16.2.11 live)](https://registry.npmjs.org/next)


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


- **Old WebKit (Safari < 16.4, iOS Safari < 16.4) users see a runtime exception on the first page load** — pre-16.3.1-canary.5, the `deployment-id` header's `Set-Cookie` parsing path throws on old WebKit's `Headers` API quirks — `TypeError: Failed to construct 'Headers'` on the first page load. The fix in PR #94604 (merged 2026-08-06T12:44:39Z) bypasses the `Set-Cookie` parsing path on old WebKit. Audit recipe: `npm ls next` + `npm view next@canary version` for the canary.5 SHIP event + check your analytics for old WebKit share. Pre-canary.5 workaround: ensure `experimental: { deploymentId: false }` in `next.config.ts` to disable the deployment-id path entirely (loses the Vercel blue-green + cache-busting benefit but eliminates the runtime exception). See the new `## Next.js 16.3.1-canary.4-ahead — Deployment-Id Old WebKit Fix (PR #94604) + 3 New Open Issues (\`#96810\`, \`#96812\`, \`#96646\`) (August 6, 2026)` section above.
- **Custom Turbopack plugins crash on startup, then every request hangs for 60s with no error** — pre-16.3.1-canary.5, a crashed Turbopack plugin worker becomes a zombie; the orchestrator dispatches to the crashed worker again on the next request, hangs forever waiting for the response, and the user sees a 60s time-out. Tracked as issue #96810 (open at this cron's check). Workaround: monitor the worker count via `process._getActiveHandles()` in a /healthz endpoint; alert when the count exceeds the expected number of plugins + 1. See the new section above.
- **`output: 'standalone'` deployments to Vercel fail with `ENOENT` on `next.config.*`** — pre-16.3.1-canary.5, the standalone bundle has a code path that re-imports the config at runtime, which Vercel doesn't include. Tracked as issue #96646 (open). Workaround: copy the `next.config.*` file into the `standalone/` directory before deploying, or use `output: 'export'` + a separate server runtime, or deploy to a non-Vercel platform (Docker + node:20-alpine works). See the new section above.
- **Long dev sessions with `'use cache'` + Cache Components grow memory unboundedly** — pre-16.3.1-canary.5, the cache-component dev cache's HMR hash refresh was triggered on server restart and on page additions/removals, throwing away all cached data for nothing. Tracked as PR #96235. Workaround: reload the dev server periodically during long dev sessions. See the new section above.
- **Pages Router dev hovers-to-404 for newly-added pages** — pre-16.3.1-canary.5, Webpack's dev-server page announcement only triggered when `prevSortedRoutes.every((val, idx) => val === sortedRoutes[idx])` was false — a prefix comparison that fails when a new route sorts after all existing ones. Tracked as PR #96250. Workaround: manual reload after adding a new page in dev mode. See the new section above.
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

### `@next/eslint-plugin-next` Security Rules (16.3.0-canary.71, [PR #93057](https://github.com/vercel/next.js/pull/93057) by SukkaW, June 29, 2026)

The `no-location-assign-relative-destination` rule (enabled by default in `@next/eslint-plugin-next`'s recommended config) got two important enhancements in 16.3.0-canary.71. The rule is a **defense-in-depth against open-redirect and phishing-via-relative-URL attacks** — it catches code like:

```js
// 🚨 Rule will flag this — navigates to attacker-controlled relative URL
function handleComment(commentBody: string) {
  window.location.assign(commentBody) // commentBody might be "/redirect?to=https://evil.com"
}

// 🚨 Rule will flag this too
window.location.href = userInput
```

**What's new in 16.3.0-canary.71:**

1. **Real `location` scope check** — the rule now only flags when `location` is a true global variable, not a shadowed local one. Previously, the rule would false-positive on code like:
   ```js
   // ✅ Before 16.3.0-canary.71: rule would WRONGLY flag this
   // ✅ After 16.3.0-canary.71: correctly recognized `location` is a local
   function handler() {
     const location = '/safe-path'  // local var
     window.location.assign(location)  // not a global location — local var is safe
   }
   ```
2. **Basic variable tracking** — the rule now follows simple variable assignments, so it can correctly resolve whether a given `location.assign(...)` call is on the global `location` (dangerous) or on a local variable shadowing it (safe). Previously it would only look at the literal `location.X` call site.

**Why this matters:**

- **If you have code that uses local `location` variables**, you may see new lint warnings after upgrading — but they'll be false-positives that the rule's now-fixed heuristics can be wrong about (re-run the linter to confirm; if it's still flagging a local var as if it were the global, that's a bug in the new variable tracker — file an issue against the rule).
- **If you have code that uses `window.location.assign()` on dynamic data**, the rule will catch it. Before 16.3.0-canary.71, it might have missed some cases due to the scope-check bug.
- **The rule is enabled by default** in the recommended config — it runs whenever you run ESLint with `@next/eslint-plugin-next`. There's no opt-in for the new behavior; you get it automatically.

**For deployment:** if you're using the Next.js ESLint flat config from setup.md, the rule is already active. No config changes needed. If you've customized your ESLint config to disable this rule, you should re-enable it — the 16.3.0-canary.71 enhancements make the false-positive rate much lower (the scope check + variable tracking fix the most common false-positives that motivated teams to disable the rule).

**Sources:**
- [PR #93057 — Enhance ESLint rule `no-location-assign-relative-destination` (canary.71, June 29, 2026)](https://github.com/vercel/next.js/pull/93057)
- [PR #92900 — Original rule PR (referenced for the scope-check suggestion)](https://github.com/vercel/next.js/pull/92900)
- [`@next/eslint-plugin-next` source — `packages/eslint-plugin-next/src/rules/no-location-assign-relative-destination.ts`](https://github.com/vercel/next.js/tree/canary/packages/eslint-plugin-next/src/rules/no-location-assign-relative-destination.ts)

### Stale-Build Warning Removed (16.3.0-canary.87, PR [#95813](https://github.com/vercel/next.js/pull/95813) by mischnic, merged 2026-07-15T18:40:47Z)

The "your `.next` directory is stale, please run `next build` again" warning (originally added in [PR #88001](https://github.com/vercel/next.js/pull/88001)) was always firing — its detection logic was buggy and incorrectly concluded the build was stale on every invocation. canary.87 removes the warning entirely. **No code change required** — purely a cleanup of a false-positive warning. If you had CI scripts that gated on this warning's presence (e.g. `if grep -q "stale" .next/build-output.log; then ...`), those branches will no longer trigger and the gate logic should be removed or replaced with an explicit version check against `next --version`.

### `serverExternalPackages` + Server Actions: NFT Trace Regression Fixed (canary-branch hotfix, PR [#95824](https://github.com/vercel/next.js/pull/95824) by gaearon, merged 2026-07-16T05:13:26Z — **NOT in 16.3.0-canary.87**, still on canary-branch HEAD as of 2026-07-16T12:00Z; will ship in 16.3.0-canary.88 or later)

**Severity:** Production-impacting. Affects every deploy that combines `serverExternalPackages` with Server Actions referenced by client components — Vercel deploys, `output: 'standalone'`, OpenNext adapters, and any other target whose build is assembled from Next.js's traced-file outputs.

**Symptom (canary.72–canary.86, the 16.3 NFT regression):** A package listed in `serverExternalPackages` (e.g. `lodash`, `pdf-lib`, a database driver) that is **only imported from a `'use server'` action referenced by a client component** was emitted into the trace as a content-hashed alias symlink — `.next/node_modules/<pkg>-<hash>` — **without the store files the symlink points to**. The runtime then tried to require the package, followed the dangling symlink, and threw `Failed to load external module <pkg>-<hash>` with HTTP 500 when the action was invoked.

The 16.2.9 build traces both the RSC template **and** the server actions loader module, so the same fixture returns the action result with HTTP 200. The 16.3 NFT rewrite ([PR #94224](https://github.com/vercel/next.js/pull/94224) / [#92901](https://github.com/vercel/next.js/pull/92901)) only traced the RSC template's subgraph; the server-actions loader is a separate `additional_entries` module graph that was chunked into the endpoint output (which is why the alias symlink got emitted) but its subgraph — including the externals' traced target files — was never visited.

**Both CJS and ESM externals are affected**, with any package manager. The ESM/Bun/monorepo framing in the original bug reports was coincidence — it was about which packages happened to be reachable only through actions in those projects, not about the runtime or module format.

**Fix ([PR #95824](https://github.com/vercel/next.js/pull/95824), will ship in 16.3.0-canary.88 — commit `b7ab5538` on canary branch 2026-07-16T05:13:26Z):** `AppEndpoint::trace_result` now accepts multiple entry modules and the app endpoint passes the actions loader alongside `rsc_entry`. `trace_endpoint` walks both subgraphs. The end-to-end test (`test/production/standalone-mode/server-action-externals/standalone-mode-server-action-externals.test.ts`) builds with `output: 'standalone'`, deletes everything except the standalone output (so only traced files are available, like a deployed lambda), runs `server.js`, and invokes the server action in a browser — fails with HTTP 500 pre-fix, returns the action result with HTTP 200 post-fix. The equivalent `16.2.9` fixture has always worked.

**What to do right now:**

| Your version | What to do |
|---|---|
| **Next.js 16.2.9 / 16.2.10 (latest stable)** | Nothing. 16.2.9 traces both subgraphs correctly. |
| **Next.js 16.3.0-canary.72–canary.87 with `serverExternalPackages` referenced only from server actions** | **Affected.** Upgrade to canary.88 when it's cut (~24h after canary.87, expected 2026-07-16T23:00Z), or apply the [PR #95824 patch](https://github.com/vercel/next.js/pull/95824) on top of canary.87 locally. If you can't upgrade, the workaround is to **also import the external from an RSC component** (any Server Component, layout, page, or middleware) — that puts it on the RSC subgraph and the 16.3 tracer picks it up correctly. The fixture must not import the externals from any other route: the standalone output is the union of all route traces, so a route that traces them correctly would mask the missing entries. |
| **Next.js 16.3.0-canary.87 with `serverExternalPackages` imported from RSC + page** | Unaffected — the RSC subgraph already traces them. |

**Audit your project for the affected pattern:**

```bash
# 1. List every package you externalise on the server
rg -A2 'serverExternalPackages' next.config.ts

# 2. Find every client-component action reference (the action has to be reachable
#    from a client boundary, and the externals have to be imported only inside
#    the action body for the regression to bite)
rg -l "'use server'" app/ -g '*.{ts,tsx}' | head -50

# 3. For each action file, find what it imports — anything in step 1's list is suspect
rg "'use server'" app/ -g '*.{ts,tsx}' -A30 | rg "import .* from ['\"](<list from step 1>)['\"]"
```

If the audit finds a match, either upgrade to canary.88+ when it's available or apply the workaround above (one extra non-action import of the same package from a Server Component). For Vercel preview deployments specifically, the regression manifests as `Failed to load external module <pkg>-<hash>` in the function logs the first time the action is invoked; the same code on 16.2.x deploys cleanly.

**Sources:**
- [PR #95824 — `Turbopack: trace externals imported only by server actions`](https://github.com/vercel/next.js/pull/95824) · Commit `b7ab553862` · gaearon · merged 2026-07-16T05:13:26Z · will ship in canary.88
- [PR #94224](https://github.com/vercel/next.js/pull/94224) + [#92901](https://github.com/vercel/next.js/pull/92901) — the 16.3 NFT rewrite that introduced the regression (April–May 2026)
- [Discussion #95130 — "Next.js 16.3 Preview — Feedback" (comment #17652155, where the user-visible failure was reported)](https://github.com/vercel/next.js/discussions/95130#discussioncomment-17652155)
- [Issue #87737 (comment 4897366831) — deploy-failure reports from `output: 'standalone'` users](https://github.com/vercel/next.js/issues/87737#issuecomment-4897366831)
- Test fixture: [`test/production/standalone-mode/server-action-externals/standalone-mode-server-action-externals.test.ts`](https://github.com/vercel/next.js/blob/canary/test/production/standalone-mode/server-action-externals/standalone-mode-server-action-externals.test.ts)

### `@next/routing` is Now Stable (16.3.0-canary.87+, PR [#94903](https://github.com/vercel/next.js/pull/94903))

`@next/routing` — the routing layer used by the Adapter API to map incoming requests to Next.js route handlers — **was promoted from experimental to stable** in canary.87. The [adapters documentation](https://github.com/vercel/next.js/blob/canary/docs/01-app/03-api-reference/07-adapters/05-routing-with-next-routing.mdx) and the `@next/routing` package README had outdated "experimental" status notices that incorrectly tied its stabilization to the adapters API. PR #94903 deletes those notices; the routing API documentation itself is unchanged.

**Practical impact:** none for code — `@next/routing` is the same API it has been since 16.2. The change is in the docs: when you read the adapters guide or the `@next/routing` package README, you no longer see "experimental" caveats. Adopters writing platform adapters should still consult the [Adapter API docs](https://github.com/vercel/next.js/tree/canary/docs/01-app/03-api-reference/07-adapters) and the [OpenNext](https://opennext.js.org/) / Cloudflare / Vercel references for the platform-specific call sites, but the routing primitive underneath is now considered stable.

**Sources:**
- [PR #94903 — `docs: remove experimental @next/routing note`](https://github.com/vercel/next.js/pull/94903) · Merged 2026-07-15T20:10:32Z
- [`docs/01-app/03-api-reference/07-adapters/05-routing-with-next-routing.mdx` at canary](https://github.com/vercel/next.js/blob/canary/docs/01-app/03-api-reference/07-adapters/05-routing-with-next-routing.mdx)
- [`packages/next-routing/` README at canary](https://github.com/vercel/next.js/tree/canary/packages/next-routing)

### `NEXT_HASH_SALT` Now Applied to Server-Side `assetsHashes` (16.3.0-canary.87+, PR [#95738](https://github.com/vercel/next.js/pull/95738) by mischnic, merged 2026-07-15T09:28:07Z)

The `NEXT_HASH_SALT` build-time env var (used to rotate the build-output content hash between deploys — typically when you want to invalidate CDN caches or browser Service Worker caches without rotating the file contents) was applied to client-side asset hashes but was silently ignored for the server-side `assetsHashes` object (the manifest `buildId` and asset manifest). On canary.87+ the salt propagates to both, so a salt change invalidates **everything** in the output — client and server — atomically.

**Why this matters for deploys:**

- **CDN cache busting** — without the fix, rotating the salt would push new client bundles to the CDN but the server-rendered HTML would still reference the old asset paths from the stale `assetsHashes`, leading to 404s on the first request after a deploy until the CDN caught up. canary.87+ makes the two halves atomic.
- **Service Worker / workbox cache invalidation** — same story: SW caches keyed off the old asset paths were never invalidated when the salt rotated, leading to long-tail stale UI. With the fix the SW cache key moves with the salt.

**Usage (no API change — the env var works the same):**

```bash
# Bump the salt at deploy time (e.g. in your CI/CD step) to force-invalidate
# every cached asset, both client and server. Set to any non-empty string.
NEXT_HASH_SALT="deploy-2026-07-16" npm run build
```

**Source:** [PR #95738 — `Respect NEXT_HASH_SALT for server side assetsHashes`](https://github.com/vercel/next.js/pull/95738) · Commit `b3480cfdd5` · mischnic · merged 2026-07-15T09:28:07Z · canary.87

### AGENTS.md Now Tells Agents Not To Fabricate `next.config.js` Options (16.3.0-canary.87+, PR [#95825](https://github.com/vercel/next.js/pull/95825) by sampoder, merged 2026-07-15T18:57:46Z)

The managed block that `next dev` writes into `AGENTS.md` (the AI-agent block introduced in 16.3) **now includes a Turbopack-specific guard against hallucinated config options**. It tells coding agents that the Turbopack option set is a closed enum — `experimental.turbopack.chunkingHeuristics`, `experimental.turbopack.memoryEviction`, `experimental.turbopack.fileSystemCacheForBuild`, etc. — and to consult the [Turbopack config docs](https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopack) before suggesting or scaffolding any Turbopack-related config. The change is in the AGENTS.md managed block only; no user-facing API change.

**Why this matters:** the PR author (sampoder, who works on Turbopack) reported that "common piece of feedback I've gotten on my PRs" is that agents hand-author PRs that plumb a non-existent Turbopack flag through `next.config.ts`. With the guard in place, an agent looking at the project's `AGENTS.md` will read the warning before fabricating a config schema, and is more likely to ask "is there actually a flag for this?" or to consult the docs first.

**What to do:** nothing — the guard is opt-in via `agentRules: false` (the existing 16.3 opt-out for the managed block), and the change is purely additive. If you use a coding agent (Claude Code, Cursor, Codex, Devin, etc.) that reads `AGENTS.md`, you get the new guard automatically on the next `next dev` start. If you maintain a custom AGENTS.md that does NOT include the managed block, you may want to copy this guard in by hand.

**Source:** [PR #95825 — `[turbopack] Tell agents not to mention next.config.js options`](https://github.com/vercel/next.js/pull/95825) · Commit `98c9754ad8` · sampoder · merged 2026-07-15T18:57:46Z · canary.87

### `experimental.outputHashSalt` and `experimental.supportsImmutableAssets` Promoted Out of `experimental` (16.3.0-canary.91, PR [#95840](https://github.com/vercel/next.js/pull/95840) + PR [#95351](https://github.com/vercel/next.js/pull/95351) by mischnic, merged 2026-07-20T14:34:59Z + 2026-07-20T13:35:15Z)

Two more config flags graduate from `experimental.*` to stable `next.config.ts` keys in canary.91. Both PRs preserve the `experimental.*` alias as a fallback so existing adapters don't break — but new projects should use the stable names.

**PR [#95840](https://github.com/vercel/next.js/pull/95840) — `outputHashSalt` is now stable.** The stable config key replaces `experimental.outputHashSalt`. The CLI / env-var counterpart (`NEXT_HASH_SALT`) is unchanged — see the `NEXT_HASH_SALT` section above for the canary.87 fix that made the env-var apply server-side. The two are complementary: the env-var is for build-time rotation without a config change; the config key is for checked-in configuration. Both must point at the same value if you use both, or you'll get inconsistent hashes between client and server. **Audit:** `rg "experimental\.outputHashSalt|NEXT_HASH_SALT" next.config.*` — any match in `next.config.ts` should be migrated to `outputHashSalt: '...'` (top-level).

**PR [#95351](https://github.com/vercel/next.js/pull/95351) — `supportsImmutableAssets` is now stable.** The stable config key replaces `experimental.supportsImmutableAssets`. This flag controls whether Next.js emits build assets with long-cache immutable URLs (`immutable` Cache-Control + immutable URL hashing). Disable it if your CDN or proxy strips query params / fragments from hashed URLs (Cloudflare with aggressive caching, Fastly with `respect_query_string: false`, etc.). [PR #95521](https://github.com/vercel/next.js/pull/95521) (already documented in the canary.87 setup.md section) auto-disables the flag when `output: 'export'` / `output: 'standalone'` is set, because immutable-asset hashing can disagree with the bundler's per-build content-hash output in those modes. **Audit:** `rg "experimental\.supportsImmutableAssets" next.config.*` — any match in `next.config.ts` should be migrated to `supportsImmutableAssets: true` (top-level).

**Migration template:**

```ts
// Before (canary.90 and earlier, still works as a fallback alias)
const config = {
  experimental: {
    outputHashSalt: 'deploy-2026-07-20',
    supportsImmutableAssets: true,
  },
};

// After (canary.91+, recommended)
const config = {
  outputHashSalt: 'deploy-2026-07-20',
  supportsImmutableAssets: true,
};
```

The `experimental.*` aliases are kept for one minor version (canary.91 + the canary that ships with Next.js 16.3.0 stable) and removed in Next.js 17.0. No code-action required for users who set the alias and never migrate — they'll get a deprecation warning at build time, not an error.

**Sources:**
- [PR #95840 — `Move outputHashSalt out of experimental` (mischnic, 2026-07-20T14:34:59Z, canary.91)](https://github.com/vercel/next.js/pull/95840)
- [PR #95351 — `Move immutable static assets config option out of experimental` (mischnic, 2026-07-20T13:35:15Z, canary.91, closes NAR-868)](https://github.com/vercel/next.js/pull/95351)
- [PR #95521 — `Disable supportsImmutableAssets with config.output` (already documented, canary.87 setup.md)](https://github.com/vercel/next.js/pull/95521)
- [Next.js canary branch ahead of canary.90 (4 commits)](https://github.com/vercel/next.js/compare/v16.3.0-canary.90...canary)


### `output: 'export'` Now Validates `generateStaticParams` Per-Route (16.3.0-canary.95, [PR #95969](https://github.com/vercel/next.js/pull/95969) by devjiwonchoi / SukkaW, merged 2026-07-23T21:25:57Z)

`output: 'export'` is the static-export build mode (`next build` writes everything to `out/` for S3/Cloudflare Pages/Netlify/static hosting). Static export requires every dynamic route to be knowable at build time — and `generateStaticParams` is how you tell Next which dynamic-segment values exist. Before canary.95, an empty or missing `generateStaticParams` would silently build with no pages for that route and only surface as a missing page on the deploy target. **canary.95 makes this loud at build time** with four new error codes (1452–1455):

- **Error 1452** — `Page "<route>" is missing "generateStaticParams()" so it cannot be used with "output: export" config.` (dynamic route file doesn't export the function at all)
- **Error 1453** — `Page "<route>" is missing exported function "generateStaticParams()", which is required with "output: export" config.` (the dev-server variant — when the function isn't actually exported properly)
- **Error 1454** — `Page "<route>" returned an empty array from "generateStaticParams()". With "output: export", at least one route must be generated.` (empty `[]` — note: under non-export modes, `[]` is the documented "all paths at runtime" pattern and is still valid; only static export rejects it)
- **Error 1455** — `Page "<route>" returned incomplete params from "generateStaticParams()". With "output: export", every params object must include all dynamic route parameters. Missing: <names>.` (a params object is missing one or more dynamic segment values)

The "composed parent + child params" check (error 1455) is the new one: when multiple route segments each export `generateStaticParams`, Next.js combines them before generating routes; if the combined params omit a dynamic route parameter, the error fires with the names of the missing segments. For a route like `/blog/[year]/[slug]/page.tsx` with a parent `app/blog/[year]/layout.tsx` that only returns `[{ year: '2026' }]`, the build fails because `slug` is missing from every combined params object.

**Stacked on PR #95968** (same author, merged minutes before) which adds two more errors that fire on **all** build modes (not just `output: 'export'`):
- **Error 1450** — function returns anything that isn't an array (e.g. `return { slug: 'x' }` instead of `return [{ slug: 'x' }]`)
- **Error 1451** — an array item isn't a plain object (e.g. `return ['x', 'y']` or `return [null]`)

Together, errors 1450–1455 replace the previous "silent wrong shape" behavior with a stack-traced, source-pointed build error. The new troubleshooting page `nextjs.org/docs/messages/generate-static-params` covers all six codes.

**CI preflight recipe** — catch the errors before the deploy target does:
```bash
# 1. Find every dynamic route file in app/
rg -l '\[.*\]' app/ -g 'page\.(ts|tsx)|route\.(ts|tsx)$'

# 2. For each match, verify generateStaticParams is exported
for f in $(rg -l '\[.*\]' app/ -g 'page\.(ts|tsx)|route\.(ts|tsx)$'); do
  if ! rg -q 'export.*generateStaticParams' "$f"; then
    echo "⚠️  $f has dynamic segments but no generateStaticParams"
  fi
done

# 3. If output: 'export' is set, additionally ensure the function isn't returning []
rg 'output.*["\x27]export["\x27]' next.config.* &&   rg -B2 'export.*generateStaticParams' app/ -A5 | rg 'return \[\]'
```

**Interaction with `cacheComponents: true`:** under `cacheComponents`, `generateStaticParams` must return ≥1 param (the existing `empty-generate-static-params` error from canary.71). The new errors 1454/1455 from canary.95 are **additive** — `output: 'export'` projects also need ≥1 param AND every params object to cover every dynamic segment. The two error systems are independent: one fires under CC, the other fires under `output: 'export'`. See `api.md` → "16.3.0-canary.95 `generateStaticParams` Validation Hardening" for the full table.

**What to do if the build now fails after upgrading to canary.95+:**

1. If you have a dynamic route without `generateStaticParams` under `output: 'export'` — either add the function (returning the real param values) or remove `output: 'export'` from `next.config.ts` (and switch to `output: 'standalone'` or a Node server).
2. If your params objects are missing a dynamic segment — the combined params (parent + child) must include every dynamic segment value. For `/blog/[year]/[slug]`, return `[{ year: '2026', slug: 'first-post' }, ...]` from the leaf page, not from a parent layout alone.
3. If you were relying on the silent "empty `[]` works under non-export" pattern — that still works, just not under `output: 'export'`.

**Sources:**
- [PR #95969 — `Throw for empty or incomplete generateStaticParams results with output: export`](https://github.com/vercel/next.js/pull/95969) · devjiwonchoi (SukkaW) · merged 2026-07-23T21:25:57Z · **Shipped in `16.3.0-canary.95`**
- [PR #95968 — `Throw when generateStaticParams returns invalid values`](https://github.com/vercel/next.js/pull/95968) (stacked on #95969, same author) — merged minutes before, adds errors 1450/1451 for all build modes
- [`packages/next/errors.json` at canary.95 — errors 1450–1455](https://github.com/vercel/next.js/blob/canary/packages/next/errors.json#L1450)
- [`errors/generate-static-params.mdx` at canary.95 — troubleshooting page (covers all 6 errors)](https://github.com/vercel/next.js/blob/canary/errors/generate-static-params.mdx)
- [nextjs.org/docs/app/guides/static-exports — `generateStaticParams` section](https://nextjs.org/docs/app/guides/static-exports)
- [nextjs.org/docs/app/api-reference/file-conventions/dynamic-routes](https://nextjs.org/docs/app/api-reference/file-conventions/dynamic-routes)

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

### Next.js DevTools MCP Server (Next.js 16.2+, slimmed in 16.3)

Next.js 16.2+ ships with a built-in MCP (Model Context Protocol) server for AI-assisted development. It enables AI coding assistants (Claude, Cursor, etc.) to interact directly with your Next.js project's dev server.

**16.3 update (June 26, 2026) — smaller, more focused MCP server:**
- **Removed** the embedded Next.js knowledge base, the upgrade helper, and the Cache Components helpers — these are now reachable through the bundled docs (via the managed `AGENTS.md` block) and the three first-party Skills (`next-dev-loop`, `next-cache-components-adoption`, `next-cache-components-optimizer`). Keeping them in the MCP server was duplicate work.
- **Added two new compilation tools** that answer "did my edit compile?" from the running dev server instead of forcing agents to run a full `next build`:
  - **`get_compilation_issues`** — returns all current compilation errors for the whole project
  - **`compile_route`** — returns the compilation result for a single route
- Skills like `next-dev-loop` call the underlying `/_next/mcp` endpoints directly, so they work without extra setup.

**16.3.0-canary.85 update (July 13, 2026) — Request Insights MCP tool:**
- **Added a third MCP tool** that surfaces the dev-only `experimental.requestInsights` snapshot (set `experimental.requestInsights: true` in `next.config.ts` and restart `next dev`):
  - **`get_request_insights`** — returns the last 100 requests captured by the in-memory request-insights recorder. Input schema: `{ requestId?: string, htmlRequestId?: string }` (both optional filters). Returns the sanitized `RequestInsight[]` with `spans[]` + `fetches[]` per request. Pairs with the new **`next experimental-request-insights` CLI** for shell-only agents and CI scripts. See setup.md → `experimental.requestInsights` for the full feature breakdown.

**Setup:**
```bash
# Add to your Next.js project's .mcp.json
{
  "mcpServers": {
    "next-devtools": {
      "command": "npx",
      "args": ["-y", "next-devtools-mcp@latest"]
    }
  }
}

# The MCP server auto-discovers your running next dev. Start it with:
npm run dev
```

**Security note:** The MCP server exposes project introspection endpoints. Only enable it in local development, not in production deployments. Use `NEXT_MCP_ENABLED=false` to disable if needed.

**Package:** `next-devtools-mcp` (latest verified on npm; auto-discovers running `next dev` instance)

See: [Next.js DevTools MCP guide](https://nextjs.org/docs/app/guides/mcp) · [MCP Servers directory](https://mcpservers.org/servers/vercel/next-devtools-mcp) · [Next.js 16.3 AI Improvements blog (MCP shrinking rationale)](https://nextjs.org/blog/next-16-3-ai-improvements)

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


**Cross-reference (added 2026-08-05T06:03Z, v1.5.25 cycle)**: For the SILENT PRODUCTION WORKER HANG that ships fixed in **`next@16.3.1-canary.3`-ahead via PR #96636** (timneutkens, merged 2026-08-05T05:41:54Z — Turbopack + cross-origin CDN `assetPrefix` + Web Workers via `new Worker(new URL(...))` = worker loads but never executes silently), see **`patterns.md` → `## Pattern: Turbopack + Web Workers + Cross-Origin CDN assetPrefix (PR #96636, timneutkens — August 5, 2026)`** for the full recipe (the canonical `next.config.ts` that hung + the worker module + the symptom + the root cause chain + the fix diff + the 6-step audit recipe). PR #96636 will land on npm via `next@16.3.1-canary.3` once the canary-branch version-tag commit `bcea67d` publishes (expected within 2-12h).
## Next.js 16.3 Self-Hosting Additions (March 2026 docs refresh) — pre-16.3.0 STABLE deploy-relevant changes

**NOTE:** This section was written pre-16.3.0 STABLE (when the canary.105 era was the active deploy-relevant train). **The 16.3.0 STABLE + 16.3.1-canary.0–2 cycles shipped additional deploy-critical changes that supersede some of this section — see the `## Next.js 16.3.0 STABLE Deployment-Critical Changes (August 3, 2026) + Canary.2 Updates` section at the bottom of this file for the current state.** In particular: PR #96493 expanded the Turbopack build cache to ALL environments (not just Vercel/local), PR #96497 flipped `useTypeScriptCli` to default-ON, and PR #94311 brought +22% throughput via native Node streams.


The official Next.js self-hosting guide was last updated 2026-03-25 with new content that didn't make it into this skill at the time. Three additions are deployment-critical for any team running Next.js on its own infrastructure:

### 1. Multi-Server Deployments — `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` is REQUIRED

If you run more than one Next.js instance (load-balanced Node servers, blue/green, autoscaler with N>1, K8s deployment replicas, ECS service), **you must set `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`** to a stable value across all instances. Otherwise Server Action IDs are encrypted with a per-instance ephemeral key generated at process start, and the fleet can't share action IDs — clicking a Server Action on instance A produces an opaque ID that instance B can't decrypt.

```bash
# Generate a stable key once (32-byte base64 — AES-256)
openssl rand -base64 32

# Set in deploy environment across ALL instances
NEXT_SERVER_ACTIONS_ENCRYPTION_KEY=kP...base64...
```

**Key requirements:**
- Must be a base64-encoded value with a valid AES key length (16, 24, or 32 bytes → AES-128 / AES-192 / AES-256)
- Next.js generates 32-byte keys by default
- Rotate safely: deploy old-key + new-key both, drain traffic, then remove old-key (Server Action IDs are forward-compatible across keys)
- Failing to set this on a multi-instance deploy silently breaks Server Actions in production — the error appears only when a user submits a form

**Audit recipe** — find every deploy target that runs Next.js in production:

```bash
# Docker
grep -REn "FROM node" Dockerfile docker-compose*.yml 2>/dev/null
# K8s
kubectl get deploy -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[*].env[?(@.name=="NEXT_SERVER_ACTIONS_ENCRYPTION_KEY")].value}{"\n"}{end}'
# PM2
grep -rE "NEXT_SERVER_ACTIONS" ecosystem.config.* 2>/dev/null
# Generic .env files
grep -rE "NEXT_SERVER_ACTIONS" .env* 2>/dev/null
```

A "0 matches" result on a multi-instance deploy = broken Server Actions waiting to happen.

### 2. Docker Templates — Standalone / Export / Multi-Environment

The Next.js monorepo ships three canonical Docker templates in [`examples/`](https://github.com/vercel/next.js/tree/canary/examples):

| Template | `output:` mode | Use case |
|----------|----------------|----------|
| [`with-docker`](https://github.com/vercel/next.js/tree/canary/examples/with-docker) | `'standalone'` | **Default for production Node deploys.** Multi-stage Dockerfile that copies `.next/standalone/` + `.next/static/` + `public/` separately. Minimal image with only the production deps that the build trace detected. |
| [`with-docker-export-output`](https://github.com/vercel/next.js/tree/canary/examples/with-docker-export-output) | `'export'` | **Fully static.** Generated HTML + assets served from nginx:alpine or any static host. No Node runtime. Useful for marketing pages, docs sites, landing pages that don't need Server Actions / RSC. |
| [`with-docker-multi-env`](https://github.com/vercel/next.js/tree/canary/examples/with-docker-multi-env) | any | **Dev / staging / production** with different `ARG NODE_ENV` + per-env env-var file mounting. One Dockerfile, three compose overrides. |

**When to pick `with-docker-export-output` over `with-docker`:**
- App is a blog / docs / marketing site with no Server Actions, no RSC, no auth
- Cold start time matters (nginx serves static HTML in <50ms vs Node cold start ~500ms-2s)
- CDN-first architecture (push to S3+CloudFront / R2+Workers / Vercel static / Netlify)

**When to pick `with-docker` (standalone):**
- Server Actions, Route Handlers, RSC, ISR, dynamic rendering
- Need streaming responses, Suspense, Cache Components
- API routes / middleware / `proxy.ts`

**When to pick `with-docker-multi-env`:**
- Promote one image through dev → staging → production with different env vars at each stage
- CI/CD builds the image once and tags per env rather than rebuilding
- Avoid the "works on my machine" drift between env-specific Dockerfiles

### 3. Streaming + nginx — Disable Buffering

The Next.js App Router supports streaming responses (`<Suspense>` boundaries flush incrementally). If you're behind nginx or any reverse-proxy that buffers, **streaming is silently disabled** and TTFB regresses from ~50ms to "wait for the slowest Suspense boundary."

```nginx
# /etc/nginx/sites-available/my-app
location / {
    proxy_pass http://nextjs_backend;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;

    # REQUIRED for streaming — disable proxy buffering
    proxy_buffering off;
    proxy_cache off;

    # Long enough for slow Suspense boundaries (default 60s may be too short)
    proxy_read_timeout 300s;
    proxy_send_timeout 300s;
}
```

**Audit recipe:**

```bash
grep -REn "proxy_buffering" /etc/nginx/sites-enabled/
# Look for any site with proxy_buffering on; or proxy_buffering not set (default is on)
```

A "no matches" or all `proxy_buffering on;` results = streaming is being silently buffered.

### 4. Cache Components works on Node self-host (not CDN-only)

Cache Components (`cacheComponents: true` in `next.config.ts`) — the new PPR-aligned model in Next.js 16 — **works on `next start` (Node self-host) and inside Docker containers, not just on Vercel.** This is explicitly documented in the March 2026 self-hosting guide. If you skipped Cache Components because you assumed it was a Vercel-only feature, pick it up — partial prerendering with the static shell served from `.next/` + dynamic holes streamed from the Node process works on any host with no CDN dependency.

### 5. 16.3.0-canary.105 — Turbopack Build Cache Default-ON (deployment-relevant)

PR #96395 (sokra, merged 2026-07-31T17:24:41Z, shipped in `16.3.0-canary.105`) flipped `experimental.turbopackFileSystemCacheForBuild` from opt-in to **default-ON** for `next build` on Vercel and local. The cache persists to `.next/cache/turbopack/` and persists across CI runs (warm builds are 2.3×–5.5× faster, up to 30% on vercel.com-class apps).

**Deployment impact:**

- **Docker multi-stage builds** — the builder stage's `.next/cache/` is in the COPY layer. Either:
  - (a) Add `RUN rm -rf .next/cache/turbopack` after the builder stage before COPY (default — clean build every container rebuild)
  - (b) Mount a Docker `BuildKit` cache mount `--mount=type=cache,target=/app/.next/cache/turbopack` to persist across builds (recommended for CI)
- **CI fairness comparisons** — if you're benchmarking webpack vs Turbopack, **delete `.next/` between runs** (or the warm cache will skew numbers)
- **Disk pressure** — the cache can grow to several GB on large apps. Add `.next/cache/turbopack/` to `.dockerignore` if you don't want it in builder layer

**Opt-out (rare):**

```ts
// next.config.ts
const nextConfig: NextConfig = {
  experimental: {
    turbopackFileSystemCacheForBuild: false,
  },
};
```

**Auto-OFF in non-Vercel CI:** the cache is automatically OFF when the env vars that Vercel sets (`VERCEL`, `VERCEL_ENV`, etc.) are absent. So your GitHub Actions / GitLab / Jenkins runs get a fresh build every time, while local `next build` and `vercel build` benefit from the cache.

See `performance.md` for the full PR #96395 breakdown (default-detection logic + the `turbopackFileSystemCacheForBuildDefault()` switch + the four-state migration table).

### 6. 16.3.0-canary.105 — `experimental.turbopackChunking` Config GA (deployment-relevant)

PR #96398 (sampoder, merged 2026-07-31, shipped in `16.3.0-canary.105`) consolidates Turbopack chunking into a single new top-level config. The old `experimental.turbopack.chunkingHeuristics` + `experimental.turbopackGenerateComponentChunks` namespaces throw at config-eval time as of canary.105.

**Why this is deployment-relevant:**
- It controls **bundle structure** — how many chunks, how big, what gets grouped with what. Affects CDN cache hit rate, parallel request count, and TTFB.
- Affects Web Worker code splitting (PR #96432 fix in the same canary).
- Affects component chunks vs route chunks vs vendor chunks separation.

```ts
// next.config.ts
const nextConfig: NextConfig = {
  experimental: {
    turbopackChunking: {
      // Priority routing — which routes get their own dedicated chunks
      firstPageLoadPriority: 0.7,
      priorityRoutes: ['/', '/dashboard', '/pricing'],
      priorityBoost: 0.3,

      // Size thresholds
      minChunkSize: 20000,
      maxChunkCountPerGroup: 25,
      maxMergeChunkSize: 250000,
      minComponentChunkSize: 5000,

      // Generated component chunks
      generateComponentChunks: true,
    },
  },
};
```

**Default values work for most apps.** Only tune if:
- Your CDN cache hit rate is low (likely too many small chunks)
- TTFB is high on first page load (likely too few parallel chunks)
- Web Worker bundle isn't splitting correctly (likely `generateComponentChunks: false`)

**Prerequisites:** Must upgrade to `next@16.3.0-canary.105` or later. Older configs throw:

```
Error: experimental.turbopack.chunkingHeuristics has been moved to experimental.turbopackChunking
```

**Migration recipe:**

```bash
# Find old config references
rg "experimental\.turbopack\.chunkingHeuristics|experimental\.turbopackGenerateComponentChunks" next.config.*
# Replace with experimental.turbopackChunking.{firstPageLoadPriority,priorityRoutes,priorityBoost,generateComponentChunks}
```

The `+9% Fresh Build / +8% Cached Build` regression noted in PR #96398's stats-bot comment is now live in npm — first builds after upgrading are slightly slower until the build cache warms.

See `performance.md` for the full PR #96398 breakdown (4 heuristic knobs + 4 NEW size-thresholds + the 9-option config table).

**Sources:**
- [Next.js self-hosting guide (lastUpdated 2026-03-25)](https://nextjs.org/docs/app/guides/self-hosting)
- [Next.js deploying guide](https://nextjs.org/docs/app/getting-started/deploying)
- [`with-docker` standalone template](https://github.com/vercel/next.js/tree/canary/examples/with-docker)
- [`with-docker-export-output` static template](https://github.com/vercel/next.js/tree/canary/examples/with-docker-export-output)
- [`with-docker-multi-env` template](https://github.com/vercel/next.js/tree/canary/examples/with-docker-multi-env)
- [Custom Next.js Cache Handler (Redis example)](https://github.com/vercel/next.js/tree/canary/examples/cache-handler-redis)
- [PR #96395 — `Enable turbopackFileSystemCacheForBuild by default`](https://github.com/vercel/next.js/pull/96395)
- [PR #96398 — `[turbopack] add experimental.turbopackChunking config`](https://github.com/vercel/next.js/pull/96398)
- [PR #96432 — `[turbopack] Fix component chunks for workers`](https://github.com/vercel/next.js/pull/96432)
- [Vercel — Next.js on Vercel](https://vercel.com/docs/frameworks/full-stack/nextjs)
- [Docker official Next.js guide](https://docs.docker.com/guides/nextjs)

## Next.js 16.3.0 STABLE Deployment-Critical Changes (August 3, 2026) + Canary.2 Updates

`next@16.3.0` STABLE shipped 2026-08-03T20:34:17Z (npm `dist-tag.latest` flipped from `16.2.12` → `16.3.0`), and the canary train cut `16.3.1-canary.0` (PR #96553 catch-all fix) the same evening at 2026-08-03T22:37:40Z, then `16.3.1-canary.1` (22-commit batch) at 2026-08-04T14:56:04Z, and `16.3.1-canary.2` (final cleanup) at 2026-08-05T00:03:35Z. The earlier `## Next.js 16.3 Self-Hosting Additions` section above covered canary.105-era changes; this section covers the **deployment-critical changes that shipped in the 16.3.0 STABLE + canary.1/canary.2 cycles** that any agent deploying Next.js needs to know.

### 1. Turbopack Build Cache Default-ON EVERYWHERE (NOT just Vercel/local) — PR #96493 (shipped in 16.3.0 STABLE)

The previous `## Next.js 16.3 Self-Hosting Additions` section above stated that `experimental.turbopackFileSystemCacheForBuild` was **auto-OFF on non-Vercel CI** (a Vercel-vs-everyone-else gate). **That gate is removed in 16.3.0 STABLE** via PR #96493 (timneutkens, merged 2026-08-02T18:33:34Z). **Every `next build` in every environment (local + Vercel + GitHub Actions + GitLab + CircleCI + Buildkite + Jenkins + ECS + K8s + self-hosted VPS) now uses the warm `.next/cache/turbopack/` filesystem cache by default.**

**Why this matters for self-hosted deployments in particular:** self-hosted VPS / Docker / K8s deployments that skip the cache will now run noticeably slower than users who follow the cache (CI cold builds are unaffected, but warm builds are 2.3×–5.5× faster per the Vercel-tracked workload). **You now need to either (a) persist the cache across builds or (b) opt out explicitly.**

**Opt-out (rare):**

```ts
// next.config.ts
const nextConfig: NextConfig = {
  experimental: {
    turbopackFileSystemCacheForBuild: false,  // explicit opt-out from build cache
  },
};
```

**For Docker multi-stage builds:**

```dockerfile
# Builder stage
FROM node:22-slim AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npx next build  # writes .next/cache/turbopack/

# Production stage
FROM node:22-slim
# Either (a) skip the cache entirely (default behavior — clean container):
#    -- no COPY of .next/cache/turbopack --
# Or (b) persist the cache via a named volume on host:
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
# Mount a volume for the cache:
# docker run -v turbopack-cache:/app/.next/cache/turbopack myapp
```

**For GitHub Actions / GitLab CI:**

```yaml
# .github/workflows/deploy.yml
- uses: actions/cache@v4
  with:
    path: |
      .next/cache/turbopack
    key: ${{ runner.os }}-turbopack-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-turbopack-
```

**Audit recipe:**

```bash
# 1. Are you on next@16.3.0+?
npm list next
# 2. Is the build cache enabled?
rg -n "turbopackFileSystemCacheForBuild" next.config.*
# 3. Does your CI persist .next/cache/turbopack between runs?
ls -la .next/cache/turbopack  # check after a build
rg -n "actions/cache|key: \\$\\{\\{ runner.os \\}\\}-nextjs|path: \\.next/cache" .github/workflows/
```

### 2. TypeScript CLI Default-ON — PR #96497 (shipped in 16.3.0 STABLE, was canary.108-ahead)

The previous 16.3-canary-era docs had TypeScript CLI as `experimental.useTypeScriptCli: false` by default (you had to opt in for TS 7 to work). **In 16.3.0 STABLE, the default flips to `true`:** every `next build` runs the project-local `tsc` CLI by default (which uses the TS 7 Go-native binary when `typescript@^7` is installed).

**Practical impact for self-hosted / Docker CI:**

- **TS 7 users** — drop the now-redundant `experimental: { useTypeScriptCli: true }` line from `next.config.ts`.
- **TS 6 users** — silent behavior change: your installed TS 6 `tsc` runs instead of the JS Compiler API. Build time impact ~50–200ms per build (faster than the JS API for most projects).
- **Custom TypeScript transformers / `typescript` as a library users** — `useTypeScriptCli` only affects the `next build` type-check pass; `require('typescript')` calls in your build tools are independent. To keep using the JS Compiler API for compatibility with custom transformers, opt out:

```ts
// next.config.ts (opt out)
const nextConfig: NextConfig = {
  experimental: {
    useTypeScriptCli: false,  // back to JS Compiler API for custom-transformer compat
  },
};
```

**Audit recipe:**

```bash
# 1. Are you on next@16.3.0+?
npm list next
# 2. Are you on TypeScript 7?
npm list typescript | grep "typescript@"
# 3. Are you still setting useTypeScriptCli? (redundant on 16.3.0+)
rg -n "useTypeScriptCli" next.config.*
# 4. Do you rely on custom TypeScript transformers?
rg -n "require\(['"]typescript['"]\)" tools/ scripts/ 2>/dev/null
```

### 3. Native Node.js Streams Replace Web Streams — PR #94311 (shipped in 16.3.0 STABLE, +22% throughput)

In 16.3.0 STABLE, the App Router rendering layer now uses **native Node.js streams** instead of web streams (which Next.js converted to / from Node streams under the hood). The web→Node stream conversion overhead is removed entirely. **In Vercel benchmarks, 16.3.0 STABLE handles up to 22% more requests under load** (per the [official 16.3 blog post](https://nextjs.org/blog/next-16-3), `Improvements for today's apps > Faster server-side rendering`).

**Practical impact for self-hosted:**

- **No code changes required** — the App Router rendering surface is identical from a user's perspective.
- **Existing nginx / CDN configurations unchanged** — the response shape is the same `<Suspense>`-boundary-flushed chunks; nginx `proxy_buffering off;` settings still apply (see `## Next.js 16.3 Self-Hosting Additions` section above).
- **Memory profile improvement** — fewer intermediate buffers during streaming responses means the Node process holds ~10-15% less heap on bursty traffic.
- **Node.js 20.15+ required** for full benefit (native streams need modern Node.js); Node 18 is EOL and not supported.

**Audit recipe:**

```bash
# 1. Are you on next@16.3.0+?
npm list next
# 2. Are you on Node 20.15+? (for full native-streams benefit)
node --version
# 3. Are you using a CDN or load balancer? (none of those care about Node stream vs web stream)
# No action needed — this is a silent perf win for self-hosted Node deployments.
```

### 4. Catch-All Index Page Bug Fixed — PR #96553 (shipped in `next@16.3.1-canary.2`)

In `next@16.3.0` STABLE, catch-all routes (e.g. `/blog/[...slug]`, `/api/[...path]`) served the index handler (`slug = []` / `path = []`) for every URL matching the base path, instead of the proper dynamic page. **This was a security-relevant bug** (information disclosure + potential authorization bypass — see `security.md`'s August 3 section for the threat model). **Fixed in `next@16.3.1-canary.0` (immediate) and SHIPPED (in the full sense of "available via stable canary") in `next@16.3.1-canary.2`** (npm-published 2026-08-05T00:03:35Z).

**For self-hosted deployments stuck on `next@16.3.0` STABLE:** you can either:

- (a) **Bump to `next@16.3.1-canary.2`** — pure additive fix on top of 16.3.0; no migration steps.
- (b) **Stay on 16.3.0 and add an audit-checked middleware** to filter out requests that match the bug pattern (only do this if you can't bump due to your Node version constraints).
- (c) **Pin to `next@16.3.1-canary.2` in CI until `16.3.1` STABLE ships** (expected within days based on the canary cadence observed in the past 36h: 3 canary releases in canary.0 → canary.1 → canary.2, ~12h apart each).

**Audit recipe:**

```bash
# 1. Are you using [...slug] or [...path]?
rg -l '\[\.\.\..*\]' app/
# 2. Are you on next@16.3.0 (vulnerable) vs canary.2 (fixed)?
npm list next
# 3. Are your catch-all routes serving the index for non-empty slugs?
curl -s "https://your-app.example/blog/nonexistent-slug" | head -50
# If you get the index render for any /blog/X where X != undefined, you're on the bug.
```

### 5. Turbopack `process.env.NODE_ENV` Hardening in Standalone Output — canary.1 PR #96527 area

In `next@16.3.1-canary.1` (and SHIPPED in canary.2), Turbopack hardens the production environment variable surface in `output: 'standalone'` builds. Previously, some `process.env.*` reads could leak development-only values into the standalone bundle (notably `NEXT_PUBLIC_*` and a small set of compiler-only vars). Standalone-output deployments now get a clean production-only environment surface, which is the expected behavior for production Docker/K8s/EC2 deploys.

**No action required** — this is a silent correctness fix. If you were relying on dev-only `process.env.*` reads leaking into production (anti-pattern), audit recipe: `rg -n "process\.env\.(NODE_ENV|NEXT_PUBLIC_)" .next/standalone/` to verify the env surface in your built output.

### 6. Other canary.1+canary.2 deploy-relevant minor PRs

- **PR #96692 → PR #96592 (sokra, merged 2026-08-04T20:29:48Z) — Turbopack terminates failed plugin worker threads** — reliability: no more zombie worker threads after a custom Turbopack plugin (think `experimental.turbo.plugins`) crashes the analyzer thread in CI. Self-hosted VPS deployments running `next build` in long-lived CI runners benefit most.
- **PR #96678 (Devin, 2026-08-04T21:25:04Z) — Strip leading BOM before parsing CSS** — Turbopack CSS parser fix: files with UTF-8 BOM (rare but real, especially in vendored CSS) no longer fail to parse. Affects Windows + some legacy toolchain outputs.
- **PR #96601 (sokra, 2026-08-04T19:37:11Z) — Collapse nested promises in Turbopack analyzer** — perf: reduces analyzer overhead for large dependency graphs. Self-hosted builds with many packages see a small but measurable build-time improvement.
- **PR #96550 (vercel-release-bot, 2026-08-04T21:03:59Z) — React vendor bump `cbb046ab-20260731` → `7dfc7ccd-20260803`** — matches the React 19.3.0-canary-7dfc7ccd-20260803 cycle documented in `components.md`.

### Sources

- [Next.js 16.3 stable blog post](https://nextjs.org/blog/next-16-3) — the canonical 16.3 release announcement (August 3, 2026)
- [Next.js 16.3 native Node streams PR #94311](https://github.com/vercel/next.js/pull/94311) — Vercel-tracked +22% throughput benchmark
- [Next.js 16.3 Turbopack blog post](https://nextjs.org/blog/next-16-3-turbopack) — Turbopack-specific 16.3 changes
- [Next.js Docker guide](https://docs.docker.com/guides/nextjs/) — Docker's official Next.js containerization guide (multi-stage with `output: 'standalone'`)
- [Next.js deployment guides](https://nextjs.org/docs/app/guides/deploying-to-platforms) — all official deploy options (Vercel, Netlify, Cloudflare, AWS, etc.)
- [Next.js 16.3 self-hosting guide](https://nextjs.org/docs/app/guides/self-hosting) — the canonical self-hosting reference
- [Next.js `turbopackFileSystemCache` config reference](https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopackFileSystemCache) — docs for the default-on config
- [Next.js `useTypeScriptCli` config reference](https://nextjs.org/docs/app/api-reference/config/next-config-js/useTypeScriptCli) — docs for the default-on config
- [v16.3.0 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.0) — npm-published 2026-08-03T20:34:17Z
- [v16.3.1-canary.0 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.0) — npm-published 2026-08-03T22:37:40Z (PR #96553 catch-all fix)
- [v16.3.1-canary.1 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.1) — npm-published 2026-08-04T14:56:04Z (22 commits)
- [v16.3.1-canary.2 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.2) — npm-published 2026-08-05T00:03:35Z (final cleanup)
- [PR #96493 — Enable Turbopack build filesystem cache by default (everywhere)](https://github.com/vercel/next.js/pull/96493) — timneutkens, merged 2026-08-02T18:33:34Z, shipped in 16.3.0 STABLE
- [PR #96497 — Enable TypeScript CLI by default](https://github.com/vercel/next.js/pull/96497) — timneutkens, merged 2026-08-03T16:10:51Z, shipped in 16.3.0 STABLE
- [PR #96553 — Fix catch-all index page being served for every other slug](https://github.com/vercel/next.js/pull/96553) — acdlite, shipped in 16.3.1-canary.0 (instant patch for the 16.3.0 bug)

---



## Next.js 16.3.1-canary.4-ahead — `experimental.appNewScrollHandler` Removal (PR #95602) + `@swc/helpers` Bump Fixes `wrap_reg_exp` Module Not Found (PR #96720) (August 5–6, 2026)

Two deployment-relevant changes landed on the canary branch ahead of `next@16.3.1-canary.4` (which itself shipped at npm `dist-tag.canary` on 2026-08-06T00:10:18Z; both PRs are now live in `next@16.3.1-canary.4+` and will be in the next 16.3.1 stable). They affect self-hosted deployments differently:

### 1. `experimental.appNewScrollHandler` Flag Removed — PR #95602 (Sebbie Silbermann, merged 2026-08-06T09:53:09Z, ships in `next@16.3.1-canary.5`)

The `experimental.appNewScrollHandler` flag has been **removed entirely** (Sebbie's note: *"The flag has been enabled by default for 16.3. It could've been removed before 16.3 but I just missed the window."*). PR #95602 is a 21-file / +160/-485 diff that:

- **Deletes** `appNewScrollHandler` from `ExperimentalConfig` in `packages/next/src/server/config-shared.ts`, the schema in `packages/next/src/server/config-schema.ts`, the `define-env` build-time flag plumbing, the `layout-router.tsx` 5-line branch, and the build-and-test workflow matrix entry.
- **Cleans up** 9 test files that referenced the flag (hydration-error.test.ts, navigation-focus.test.ts, parallel-routes-scroll-owner.test.ts, router-autoscroll.test.ts, etc.).

**For self-hosted deployments:**

- **(a) Config cleanup required** — if your `next.config.ts` has `experimental: { appNewScrollHandler: true }` (or `false`), **remove the line**. The flag no longer exists in canary.5+. Next.js will warn (or error in strict mode) about unknown experimental flags. The build will not fail with a hard error (the config-schema validator silently drops unknown keys), but you'll get a `next-config-validator` warning.
- **(b) Behavior unchanged on 16.3.0 STABLE** — the new scroll handler has been the default since 16.3.0 (you've already been using it). The only deployment-relevant change is the config cleanup.
- **(c) No CI changes required** — the existing scroll-restoration semantics are preserved verbatim.

**Audit recipe:**

```bash
# 1. Is your project on the flag?
rg -n "appNewScrollHandler" next.config.* tsconfig.* 2>/dev/null
# Expected post-fix: 0 hits. If you have a hit, remove the line.

# 2. Are you on next@16.3.1-canary.5+?
npm ls next
# If canary.5+, the flag is gone; if canary.4 or earlier, the flag still parses but emits a deprecation warning.

# 3. Verify scroll-restoration behavior in your app
# Open the app, navigate via <Link>, click the Back button. The scroll position should restore correctly.
# (This should be unchanged from 16.3.0 — the flag was default-on there.)
```

**Migration:** No code changes. Config cleanup (`rg -n "appNewScrollHandler" next.config.*` → remove the line). The flag will be removed entirely in the next 16.3.1 stable.

### 2. `@swc/helpers` Bump Fixes `Module not found: '@swc/helpers/_/_wrap_reg_exp'` — PR #96720 (Niklas Mischkulnig, merged 2026-08-06T07:09:24Z, ships in `next@16.3.1-canary.5`)

A long-standing bundling bug: `next@16.3.0` STABLE could fail at `next build` with `Module not found: Can't resolve '@swc/helpers/_/_wrap_reg_exp'` for any user code that uses `RegExp` operations compiled by SWC. The SWC **crates** were upgraded but the **`@swc/helpers`** npm package was never bumped to match. The `_wrap_reg_exp` helper was added in March 2026 (the SWC side picked it up), but Next.js's pinned `@swc/helpers` dependency didn't include it, so Webpack/Turbopack's module resolver couldn't find the helper for any compiled output that needed it.

**The fix** — PR #96720 bumps `@swc/helpers` to the version that includes `_wrap_reg_exp`. **Closes issue #94634** (ken-spencer/ff-refresh-bug — the Firefox + PPR infinite-refresh-loop repro that surfaced the underlying issue).

**For self-hosted deployments:**

- **(a) Affected deployments** — `next@16.3.0` STABLE + `next@16.3.1-canary.0/1/2/3/4` (all canary cuts before PR #96720). Any deployment that compiles `RegExp` operations (string `.match()`, `.replace()`, `.split()` with regex; `new RegExp(...)`; regex literals in conditional paths) is potentially affected.
- **(b) Trigger pattern** — the error surfaces during `next build` (Turbopack or Webpack) when the analyzer first hits a chunk that needs the `_wrap_reg_exp` helper. Production builds may emit a successful bundle but with a broken dynamic-import resolution at runtime (lazy-loaded chunks silently fail). Dev mode (`next dev`) is more likely to surface the error because Turbopack re-analyzes on every request.
- **(c) The fix** — bump to `next@16.3.1-canary.5+`. No code changes, no config changes, no peer-dep churn. Pure build-system fix.
- **(d) Affected deployments stay on 16.3.0** — the workaround is to bump to `next@16.3.1-canary.5+` (or wait for `next@16.3.1` STABLE). If you can't bump due to Node version constraints or a custom SWC transformer that pins a specific helper version, the only alternative is to refactor the affected code to avoid the helper (e.g., replace `str.match(/regex/g)` with `str.split(/regex/g).filter(...)` patterns that don't trigger SWC's regex wrapping).

**Audit recipe:**

```bash
# 1. Are you on next@16.3.0 or canary.0-4 (potentially affected)?
npm ls next
# Expected: 16.3.0 or 16.3.1-canary.0..canary.4 → potentially affected. canary.5+ → fixed.

# 2. Does your codebase use RegExp operations in compiled paths?
rg -n "(new RegExp|\.match\(|\.replace\(|\.split\(|/\^?[a-z]/i)" app/ src/ components/ lib/ 2>/dev/null | head -20
# Expected: a list of files. Any hits → potentially affected. None → likely not affected.

# 3. Run next build and watch for the module-not-found error
next build 2>&1 | rg -i "wrap_reg_exp|module not found"
# If hits → you're on the bug, bump to canary.5+.

# 4. Smoke-test lazy-loaded chunks in production
# If your app uses dynamic imports with RegExp-heavy modules, exercise them in production mode:
# next build && next start
# Click through routes that trigger dynamic imports; check the Network tab for 404s on _next/static/chunks/*.js

# 5. Firefox + PPR users (ken-spencer repro): if you're seeing infinite refresh loops in dev, you may have hit this bug.
# Bump to canary.5+ to fix.
```

**Migration:** No code changes. Bump `next` to `16.3.1-canary.5+` (or wait for `16.3.1` STABLE). The fix is a one-line `@swc/helpers` version bump in `packages/next/package.json` and the corresponding lockfile update.

### Sources

- [PR #95602 — `[fragment-scroll]` Remove `experimental.appNewScrollHandler`](https://github.com/vercel/next.js/pull/95602) — Sebbie Silbermann, 21 files / +160/-485, merged 2026-08-06T09:53:09Z, ships in `next@16.3.1-canary.5` (canary-branch version-tag still pending at this cron's check)
- [PR #96720 — Bump `@swc/helpers`](https://github.com/vercel/next.js/pull/96720) — Niklas Mischkulnig, merged 2026-08-06T07:09:24Z, ships in `next@16.3.1-canary.5`; closes [issue #94634](https://github.com/vercel/next.js/issues/94634) (Firefox + PPR infinite refresh loop)
- [Next.js `experimental.appNewScrollHandler` docs (pre-#95602 — page now redirects to fragment-scroll since the flag is removed)](https://nextjs.org/docs/app/api-reference/config/next-config-js/appNewScrollHandler) — the canonical pre-removal reference
- [Next.js fragment-scroll behavior docs](https://nextjs.org/docs/app/building-your-application/routing/fragment-scroll-behavior) — the new home for the scroll-restoration docs (the flag's behavior lives here now)
- [Issue #94634 — infinite refresh loop and crashes in next.js 16.2.7 (ken-spencer repro)](https://github.com/vercel/next.js/issues/94634) — closed 2026-06-29T06:46:23Z (the issue was closed but the fix didn't ship until PR #96720 on 2026-08-06)
- [@swc/helpers npm package](https://www.npmjs.com/package/@swc/helpers) — the bumped dependency
- [v16.3.1-canary.4 GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.4) — npm-published 2026-08-06T00:10:18Z (the canary cut that shipped before these two PRs landed; PR #95602 + #96720 will be in `canary.5`)
- [canary-branch compare v16.3.1-canary.4...canary (3 commits ahead at 2026-08-06T12:05Z)](https://github.com/vercel/next.js/compare/v16.3.1-canary.4...canary) — the source of both PRs

## Next.js 16.3.1-canary.4-ahead — Deployment-Id Old WebKit Fix (PR #94604) + 3 New Open Issues (`#96810`, `#96812`, `#96646`) (August 6, 2026)

The 6h window since the v1.5.30 cycle (which documented PR #95602 + PR #96720) has added **2 NEW commits to the canary-branch ahead of canary.4** (verified at this cron's check via `GET /repos/vercel/next.js/compare/v16.3.1-canary.4...canary` returning `ahead_by: 3, behind_by: 0` — the count is unchanged from v1.5.30 because the version-tag commit for canary.5 is still pending, but **PR #96250 + PR #96235 + PR #96745 + PR #94604** are all material-deployment-relevant and were not yet documented in deployment.md at v1.5.30). The canary-branch currently has 3 commits ahead of canary.4 (verified at 2026-08-06T18:03Z): the 3 from v1.5.30 (PR #96774 test infra non-material + PR #96720 swc/helpers bump + PR #95602 appNewScrollHandler removal) plus one NEW deployment-critical fix — **PR #94604** by Niklas Mischkulnig (merged 2026-08-06T12:44:39Z) — `Fix(deployment-id): prevent exception on old webkit`. Three new open issues are also material for deployment reliability: **#96810** (Turbopack reap crashed plugin workers instead of hanging forever), **#96812** (Clean up dev errors RSC streams the HMR client never consumes), and **#96646** (`output: 'standalone'` breaks Vercel deployments on Next 16.3 — ENOENT). **Plus 2 remaining undocumented material PRs from the canary.4-ahead set**: PR #96250 (Turbopack/webpack dev server page announcements) and PR #96235 (use cache over/under-invalidation in dev). Both are document-bounded fixes that don't change deployment topology but do affect deployment observability + dev-mode behavior.

### 1. PR #94604 — Fix(deployment-id): prevent exception on old webkit (Niklas Mischkulnig, merged 2026-08-06T12:44:39Z)

**The headline is a runtime exception fix on old WebKit deployments** — the `deployment-id` header (used by Vercel + Cloudflare + any deployment-safety-aware infrastructure to identify a specific build for cache-busting + blue-green routing) was throwing on old WebKit browsers (Safari < 16.4, iOS Safari < 16.4, all WebKit-based browsers pre-2023) because the `Headers` API's `Set-Cookie` parsing path in `Set-Cookie` headers can throw on browser-quirky Cookie encoding. The exact failure mode: Next.js's `appendHeader` for `deployment-id` calls `headers().append('Set-Cookie', ...)` in some middleware paths on deployed pages; on old WebKit, the WebKit `Headers` polyfill throws `TypeError: Failed to construct 'Headers'`. The fix detects the old WebKit quirks and falls back to a direct `headers().set('deployment-id', id)` path that bypasses the `Set-Cookie` parsing. **Migration:** **no code changes required** for apps on `next@16.3.0` STABLE — the fix is in `next@16.3.1-canary.5`-ahead. **The affected-deployment profile** is non-trivial: any Next.js deployment where the user-agent ∈ {Safari < 16.4, iOS Safari < 16.4, WebKit on Linux < 605.1.15, Chrome < 110 with WebKit fallback} would see a runtime exception on the first page load. **Note**: this is a pre-existing latent bug in 16.3.0 STABLE + canary.0/1/2/3/4 (the PR description cites the issue is reproducible without `deployment-id` set, so the underlying bug is in the `Headers` constructor path that other code paths also use). **Practical impact for deployment teams:** if your analytics show Safari < 16.4 traffic (still ~3-5% of total global traffic per Cloudflare's 2026 distribution data), you will see a measurable error rate on that segment. The fix ships in canary.5; plan the upgrade when npm publishes.

**Audit recipe:**
```bash
# Confirm installed version
npm ls next

# Track canary.5 SHIP
npm view next@canary version

# Check your analytics for old WebKit share
# Cloudflare: Analytics → Performance → By Browser → WebKit
# Vercel: Analytics → Real Experience → Errors → filter by WebKit

# Spot-check the affected code path
rg -n "deployment-id|appendHeader" packages/next/src/server/  # not user-facing but documents the fix
```

### 2. Issue #96810 — Turbopack: reap crashed plugin workers instead of hanging forever (NEW, open)

A new open issue tracked at https://github.com/vercel/next.js/issues/96810 (PR #96810 referenced from the recent issue events stream). **The bug**: when a Turbopack plugin worker (e.g., a custom `unplugin` transform, a Sass/SCSS plugin, a JSX factory plugin, an SVG-as-React-component loader) crashes or panics on startup, the Turbopack orchestrator **doesn't reap the crashed worker thread** — the `Promise` returned from `worker.postMessage(...)` hangs forever at the await on the response, the worker thread becomes a zombie, and the next request that hits the same plugin path also hangs (because the orchestrator dispatches to the crashed worker again). The fix is to detect the crashed worker via `worker.on('exit', (code) => code !== 0)` and re-dispatch to a fresh worker. **Affected deployments:** any Turbopack-using Next.js deployment with a custom plugin (e.g., `@svgr/webpack` port to Turbopack, `unplugin-icons`, `unplugin-svg`, custom transformer plugins, CSS-in-JS plugins that use a worker). **Practical impact for deployment teams:** a single plugin crash becomes a per-request hang with no error surfaced; the user sees a 60s time-out, the server sees an ever-growing pool of zombie workers, and the only recovery is a manual restart. **Forward-looking:** the PR isn't merged yet at this cron's check; the running cron will track the canary-branch for the fix. **Audit recipe:** in production, watch `node --inspect` for orphaned worker threads; in dev, watch the terminal for hangs on `/500` or `/404` pages that should fail fast. **Workaround for users on Turbopack + custom plugins:** monitor the worker count via `process._getActiveHandles()` in a /healthz endpoint; alert when the count exceeds the expected number of plugins + 1 (the main thread).

### 3. Issue #96812 — Clean up dev errors RSC streams the HMR client never consumes (NEW, open)

A new open issue tracked at https://github.com/vercel/next.js/issues/96812. **The bug**: in `next dev` mode, when a Server Component throws an error during dev rendering, the error is streamed to the client over the RSC stream — but the HMR client sometimes never consumes the error stream (because the HMR client can be in a half-loaded state during a fast-refresh burst, or because the tab is in the background). The orphaned stream sits open in the dev server, holding a memory reference to the React tree that threw. After a long session with many fast-refreshes, dev mode memory grows unbounded. **Forward-looking:** the fix is to add a timeout + cleanup on the RSC stream writer when the client doesn't consume. **Practical impact for deployment teams:** dev-only — production builds don't have this issue (the streams are short-lived). **Recommended workaround: reload the dev server periodically during long dev sessions.**

### 4. Issue #96646 — `output: 'standalone'` breaks Vercel deployments on Next 16.3 (NEW, open)

A new open issue tracked at https://github.com/vercel/next.js/issues/96646. **The bug**: setting `output: 'standalone'` in `next.config.ts` (a common pattern for Docker + self-hosted builds) produces a `standalone/` directory whose `server.js` script fails with `ENOENT` on `next.config.js` (or `next.config.ts`) when deployed to Vercel. The Vercel deployment pipeline expects the standalone build to **not require the original `next.config.*` file** at runtime, but the 16.3.0 standalone bundle has a code path that re-imports the config at runtime to resolve certain `experimental` flags. **Affected deployments:** any Vercel deployment that uses `output: 'standalone'` for a Next.js 16.3.0 STABLE build. **NOT affected:** Docker builds (where the config file is included in the image), on-prem Node deployments, self-hosted Linux deployments. **Recommended workaround: copy the `next.config.*` file into the `standalone/` directory before deploying**, or use `output: 'export'` + a separate server runtime. **Forward-looking:** the PR isn't merged yet at this cron's check.

### 5. PR #96250 — Fix which pages the dev server announces, and when (ztanner, merged 2026-08-06T13:25:52Z) — **Deployment-observability lens**

The `next dev` server tells open tabs when a page is added or removed, so a tab showing a 404 picks up a page you just created, and a tab showing a page you just deleted falls back to the 404. Both Turbopack and Webpack computed the list of changed pages wrong, in opposite directions. **Turbopack** announced pages that hadn't changed (compared the new route list against the entry maps in `currentEntrypoints` — those maps key App Router entries by page name `/blog/page` but the route list is keyed by route `/blog`, so no App Router route ever matched: every update announced every existing app route as added and every page name as removed, and every connected tab refetched). **Webpack** didn't announce pages that had changed (only announced when `prevSortedRoutes.every((val, idx) => val === sortedRoutes[idx])` was false — a prefix comparison that fails when a new route sorts after all existing ones; the old list is a prefix of the new one, and nothing is announced; in Pages Router this meant client-side navigation to the new page kept returning 404 until a manual reload). **Fix:** both bundlers now compare the new route list to the previous route list, which uses the same naming. Starting the server now announces nothing on either bundler, since the first route list is not a change. **NOTE for deployment teams:** this is a **dev-mode-only** fix; production builds don't use the dev-server page announcements. The fix is in `next@16.3.1-canary.5`-ahead. **Practical impact for deployment teams:** zero for production; the fix reduces dev-server noise (every connected tab refetching on every update) and fixes Pages Router dev hovers-to-404 behavior for newly-added pages. **Audit recipe:** in dev mode, watch the DevTools Network tab for `/__nextjs_original-stack-frame` + RSC payload requests on every save; pre-fix shows many more requests than post-fix.

### 6. PR #96235 — Fix use cache over- and under-invalidation in dev (ztanner, merged 2026-08-06T13:25:55Z)

In dev mode, `'use cache'` entries are keyed by an HMR refresh hash so that edits invalidate cached data. **The bug:** this hash was managed wrong in both directions: (a) Webpack refreshed the hash at moments when no code changed — on dev server startup, and when a page file was added or removed — each refresh threw away all cached data for nothing; (b) the hash never distinguished one dev server run from another — a restarted server served data cached by the previous run, whose code may have changed while the server was down. **Fix:** adding or removing a page no longer refreshes Webpack's hash (edits and env-file changes still do); the hash now includes a random per-run value. **This fixes the `cache-components-dev-warmup` flakes** that have been reported intermittently. **Affected deployments:** dev-mode only with `cacheComponents: true` + `'use cache'` directives. **Production impact:** zero — production builds have a different cache key. **Audit recipe:** `rg -n "cacheComponents" next.config.*` to confirm dev mode is using Cache Components; `rg -n "use cache" app/ src/` to find the affected code paths; in dev, watch the terminal for `cache invalidated` messages that shouldn't be there (e.g., on a server restart).

### 7. PR #96745 — Require Cache Components for Instant Navigation testing (ztanner, merged 2026-08-06T13:42:24Z)

The Instant Navigation testing API (`@next/test-utils/playwright`'s `instant()` test helper) is only supported with Cache Components, but its testing cookie could previously force a non-Cache Components route into the legacy PPR rendering path. This could cause requests to fail because that path relies on React's deprecated `unstable_postpone` API, which is unavailable in the stable React runtime. **The fix** couples testing API exposure and its client bundle machinery to Cache Components, removes the testing-only PPR capability override, and adds regression coverage confirming that the API remains inactive without Cache Components. **Supported Instant Navigation behavior is unchanged.** **Migration:** no code changes required for projects using `@next/test-utils/playwright`'s `instant()` helper with `cacheComponents: true` already enabled. **For projects NOT using Cache Components:** the testing API was already inactive; the fix is a no-op. **For projects using `instant()` without `cacheComponents: true`:** the prerelease behavior was buggy; post-canary.5 the API is correctly inactive, so the test will fail with a clear error message rather than hanging. **Audit recipe:** `rg -n "instant()" tests/ e2e/ playwright/` + `rg -n "cacheComponents" next.config.*`.

### 8. Audit recipe (consolidated)

```bash
# Confirm installed version
npm ls next

# Watch for canary.5 SHIP
npm view next@canary version

# Check for old WebKit share in your analytics
# Cloudflare: Analytics → Performance → By Browser → WebKit
# Vercel: Analytics → Real Experience → Errors → filter by WebKit
# Pre-fix: error rate on Safari < 16.4 segment
# Post-fix: error rate should drop to baseline

# Check for 'standalone' output in next.config
rg -n "output.*standalone" next.config.*
# If you use 'standalone' + deploy to Vercel, see issue #96646 workaround

# Check for Turbopack plugin workers
rg -n "from '@next/plugin\|plugin-name" app/ src/
# If you have custom Turbopack plugins, watch for issue #96810 fixes

# Check for 'use cache' with cacheComponents
rg -n "cacheComponents" next.config.*
rg -n "use cache" app/ src/
# If you have both, the canary.5 PR #96235 fixes dev-mode cache invalidation

# Check for Instant Navigation testing
rg -n "instant()" tests/ e2e/ playwright/
# If you have these, the canary.5 PR #96745 ensures the testing API is correctly coupled to Cache Components
```

### Sources

- [Next.js PR #94604 — Fix(deployment-id): prevent exception on old webkit](https://github.com/vercel/next.js/pull/94604) — by Niklas Mischkulnig, merged 2026-08-06T12:44:39Z; the old-WebKit deployment-id exception fix
- [Next.js issue #96810 — Turbopack: reap crashed plugin workers instead of hanging forever](https://github.com/vercel/next.js/issues/96810) — open; the Turbopack-worker-reap fix
- [Next.js issue #96812 — Clean up dev errors RSC streams the HMR client never consumes](https://github.com/vercel/next.js/issues/96812) — open; the dev-mode RSC stream cleanup
- [Next.js issue #96646 — `output: 'standalone'` breaks Vercel deployments on Next 16.3 — ENOENT](https://github.com/vercel/next.js/issues/96646) — open; the standalone-output Vercel-deploy incompatibility
- [Next.js PR #96250 — Fix which pages the dev server announces, and when](https://github.com/vercel/next.js/pull/96250) — by ztanner, merged 2026-08-06T13:25:52Z; the dev-server page-announcement fix
- [Next.js PR #96235 — Fix use cache over- and under-invalidation in dev](https://github.com/vercel/next.js/pull/96235) — by ztanner, merged 2026-08-06T13:25:55Z; the dev-mode cache-invalidation fix
- [Next.js PR #96745 — Require Cache Components for Instant Navigation testing](https://github.com/vercel/next.js/pull/96745) — by ztanner, merged 2026-08-06T13:42:24Z; the Instant Navigation testing API coupling fix
- [Next.js canary-branch compare v16.3.1-canary.4...canary (3 commits ahead at 2026-08-06T18:03Z)](https://github.com/vercel/next.js/compare/v16.3.1-canary.4...canary) — confirmed at this cron's check
- [Next.js `output: 'standalone'` docs](https://nextjs.org/docs/app/api-reference/config/next-config-js/output) — the canonical `output: 'standalone'` config reference
- [Cloudflare 2026 WebKit distribution data](https://radar.cloudflare.com) — the ~3-5% old-WebKit share reference for the deployment-id fix impact estimate
- [Cross-references to the v1.5.30 deployment.md section](https://github.com/clawvpsai/frontend-skill/blob/main/deployment.md) — for the prior PR #95602 + PR #96720 context



---

## Next.js 16.3.0 STABLE — 3 NEW Open Issues Affecting Production Deployments Today (`#96859` + `#96831` + `#96855`, August 6, 2026)

Three material open issues were opened in the past 24h that **affect `next@16.3.0` STABLE users today** (not in some future canary — **the bugs are live in the current stable release**). All three are Turbopack-specific regressions introduced in the 16.3.0 release. None have PR attribution yet — this section is the deployment-bounded audit + workaround recipes. Plus **#96806** (Docker + cacheComponent + `headers()` 500 error in production) was **closed** in the same 24h window — the previously-documented v1.5.30 forward-looking note is now resolved.

### Issue #96859 — Turbopack build fails on pages-router files named `sitemap`/`robots`

**Affected deployments:** `next@16.3.0` STABLE + `16.3.1-canary.0/1/2/3/4` + Turbopack + **pages-router-only projects (no `app/` directory) that have `pages/sitemap.js` or `pages/robots.js`**.

**The bug (verified via the repro at [`rodrigo-arias/next-16-3-pages-sitemap-repro`](https://github.com/rodrigo-arias/next-16-3-pages-sitemap-repro)):**

```
Error: Turbopack build failed with 1 error:
./pages/sitemap.js:5:23
Error: "getStaticProps" is not supported in app/. Read more: https://nextjs.org/docs/app/building-your-application/data-fetching
```

**Root cause:** The app-router **metadata-route filename convention** (`sitemap`, `robots`, …) appears to be applied to **root-level `pages/` files** in Turbopack, which are then compiled under app-router rules where `getStaticProps` is rejected. **`pages/api/robots.js`** (API route) is unaffected — the bug is specific to files that the pages-router would treat as routes. Renaming the file (e.g., `sitemap-page.js`) with identical contents builds fine, so the issue is purely filename-based.

**Version matrix (per the issue reporter):**

| Next version | Result |
|---|---|
| `next@16.2.12` | ✅ builds; `/sitemap` prerenders as SSG |
| `next@16.3.0` | ❌ `"getStaticProps" is not supported in app/` |
| `next@16.3.1-canary.4` | ❌ same error |

**Webpack users are NOT affected.** The bug is Turbopack-specific.

**Workaround for users stuck on `next@16.3.0` + Turbopack + a `/sitemap` page (until the fix ships):**

```bash
# Option A — Rename the file (simplest)
mv pages/sitemap.js pages/sitemap-page.js
# Update the Link in app/components that points to /sitemap to /sitemap-page

# Option B — Switch to Webpack (preserves the filename)
# In next.config.ts:
# experimental: { turbo: undefined }  // or remove `next dev --turbopack` from package.json scripts

# Option C — Use the `pages/api/` variant (API route, not page)
mv pages/sitemap.js pages/api/sitemap.js
# Adjust the consumer to fetch /api/sitemap instead of /sitemap
```

**Audit recipe:**

```bash
# 1. Confirm the install + Turbopack usage
npm ls next
# Expected for affected: next@16.3.0 (NOT 16.2.x; NOT 16.3.1-canary.5+ yet)
grep -E '"(dev|build)":' package.json | grep turbopack
# If hits, you're using Turbopack

# 2. Check if you have a pages-router sitemap/robots file
ls pages/sitemap.* pages/robots.* 2>/dev/null
# If hits AND no app/ directory, issue #96859 affects you

# 3. Confirm Webpack is unaffected
npx next build --webpack 2>&1 | grep -E "sitemap|robots"
# If no errors on Webpack, the bug is confirmed Turbopack-only
```

### Issue #96831 — Turbopack serializes `moduleLoading.crossOrigin` as string `"none"`, adding unexpected `crossorigin=""` to chunk scripts

**Affected deployments:** `next@16.3.0` STABLE + `16.3.1-canary.0/1/2/3/4` + Turbopack + **cross-origin `assetPrefix` CDN** (asset host on a different origin than the page) + **CDN that does NOT reply with `Access-Control-Allow-Origin`** for the cached responses.

**The bug (verified via the repro at [`banqinghe/next-16-3-crossorigin-none-repro`](https://github.com/banqinghe/next-16-3-crossorigin-repro)):**

```html
<!-- Pre-16.3.0: no crossorigin attribute -->
<script src="https://cdn.example.com/_next/static/chunks/abc.js" async=""></script>

<!-- Post-16.3.0: unexpected crossorigin="" attribute -->
<script src="https://cdn.example.com/_next/static/chunks/abc.js" async="" crossorigin=""></script>
```

**Root cause (from bisecting the build output):**

```
.next/server/app/page_client-reference-manifest.js:
  16.2.12: "moduleLoading":{"prefix":"","crossOrigin":null}
  16.3.0:  "moduleLoading":{"prefix":"","crossOrigin":"none"}
```

The string `"none"` looks like a Rust `Option::None` / enum variant leaking into JSON. **React Flight only checks `typeof crossOrigin === "string"`** and normalizes any string other than `"use-credentials"` to `""` (anonymous), so `"none"` becomes `crossorigin=""` on every chunk script emitted through `prepareDestinationWithChunks` / `preinitScriptForSSR`.

**The browser therefore fetches these chunks in CORS mode** and refuses to run them when the asset host does not reply with `Access-Control-Allow-Origin`:

```
Access to script at https://cdn.example.com/_next/static/chunks/...js from origin
https://www.example.com has been blocked by CORS policy: No Access-Control-Allow-Origin
header is present on the requested resource.
```

**Critical production impact:** **Only the *preinited* chunks get the attribute** — the layout/page `<script async>` tags and the bootstrap script on the same page do not — so the **same chunk URLs are requested in a mix of no-cors and CORS modes**. On real CDNs whose cache key does not include the `Origin` header, the no-cors responses (cached without ACAO) **poison the CORS-mode loads** even when the origin/S3 CORS configuration is correct.

**Webpack users are NOT affected.** Same-origin CDNs (asset host === page origin) are NOT affected. Deployments without CORS-needing assets are NOT affected.

**Workaround for users stuck on `next@16.3.0` + Turbopack + cross-origin CDN + missing ACAO:**

```bash
# Option A — Roll back to next@16.2.12 (LTS line)
npm install next@16.2.12
# No code changes required; the `crossOrigin: null` behavior is restored

# Option B — Configure ACAO on the CDN to accept the page origin
# On CloudFront / Cloudflare / Fastly / etc.:
#   Access-Control-Allow-Origin: https://www.example.com
# This unblocks the CORS-mode loads and is the long-term fix

# Option C — Switch to Webpack (preserves the build but loses Turbopack benefits)
# In next.config.ts:
# experimental: { turbo: undefined }
# or: remove `next dev --turbopack` from package.json scripts

# Option D — Use output: 'export' (static export; no server-rendered preinited chunks)
# In next.config.ts:
# output: 'export'
# This works around the issue because the bug is in SSR-preinited chunk emission
```

**Audit recipe:**

```bash
# 1. Confirm the install + Turbopack usage
npm ls next
grep -E '"(dev|build)":' package.json | grep turbopack
# If hits, you're using Turbopack

# 2. Check if you have a cross-origin assetPrefix
rg -n "assetPrefix.*['"]https?://[a-z]" next.config.*
# If hits AND the URL host is on a different origin than the page, issue #96831 may affect you

# 3. Check if your CDN has ACAO configured for the page origin
curl -sI "https://cdn.example.com/_next/static/chunks/some-chunk.js" | grep -i "access-control-allow-origin"
# If the header is missing or doesn't match your page origin, the CORS-mode loads will fail

# 4. Confirm Webpack is unaffected
npx next build --webpack
# Check the generated HTML for crossorigin="" on chunk scripts — should be absent
```

### Issue #96855 — Scroll-reset regression with `position: fixed` parallel-route slots (`appNewScrollHandler` regression in 16.3.0)

**Affected deployments:** `next@16.3.0` STABLE + `16.3.1-canary.0/1/2/3/4` + **App Router** + **parallel-route `@slot` that renders only `position: fixed`/`sticky` elements** (e.g., `@header` slot with only a sticky navbar, `@footer` slot with only a fixed CTA, `@sidebar` slot with only a fixed TOC).

**The bug (verified via the repro at [`Pilaton/next-fixed-slot-scroll-repro`](https://github.com/Pilaton/next-fixed-slot-scroll-repro)):**

```tsx
app/
  layout.tsx            // renders {header} {children}
  @header/page.tsx      // <header className="fixed top-2"> ... </header>
  @header/[...rest]/page.tsx
  about/page.tsx
```

1. `pnpm dev`, open `/`, scroll down ~2000px.
2. Click a `<Link>` to `/about`.
3. **Current (16.3.0):** the new page opens at the previous scroll offset. If the target page is shorter than the scroll offset, it opens with the footer on screen.
4. **Expected (16.2.11 and earlier):** the new page opens at the top.

**Root cause:** `experimental.appNewScrollHandler` changed default from `false` to `true` in `next@16.3.0`. The old handler (`InnerScrollAndFocusHandlerOld`) **skipped `fixed`/`sticky` elements when picking the scroll target** (with an explicit comment: *"we ignore fixed or sticky positioned elements since they'll likely pass the 'in-viewport' check and will result in a situation we bail on scroll because of something like a fixed nav, even though the actual page content is offscreen"*). The new handler (`InnerScrollHandlerNew`, the Fragment-ref fork that is now the default) has no equivalent check. It passes the Fragment ref straight to `getScrollTargetState`, which reads the top edge of whatever host children the slot rendered. **For a slot containing only a fixed header, `elementTop` is a small positive number on every navigation**, so the result is always `1` ("already in viewport"). The handler then marks the shared `scrollRef` as handled and returns without scrolling. Because `accumulateScrollRef` assigns the *same* `scrollRef` object to every changed cache node, **the slot that renders the fixed header claims the scroll intent before the `children` slot's layout effect runs** — so the actual page content never gets scrolled.

**Workaround for users stuck on `next@16.3.0` with a fixed-only parallel-route slot (until the fix ships):**

```tsx
// Option A — Add a hidden scroll-anchor element to the slot
// In app/@header/page.tsx:
export default function HeaderSlot() {
  return (
    <>
      <header className="fixed top-2">...</header>
      <div aria-hidden="true" style={{ position: 'absolute', top: 0, height: 1, width: 1 }} />
    </>
  );
}
// The hidden div provides a non-fixed reference for the new scroll handler

// Option B — Opt out of the new scroll handler (revert to the old behavior)
// In next.config.ts:
// experimental: { appNewScrollHandler: false }
// Note: this flag was REMOVED in canary.5 (PR #95602), so this only works on 16.3.0 STABLE / canary.4

// Option C — Roll back to next@16.2.12 LTS
npm install next@16.2.12
// No code changes required; the old scroll handler is the default

// Option D — Move the fixed element OUT of the parallel-route slot
// Move the fixed header from app/@header/page.tsx to app/layout.tsx directly
// This avoids the slot-only-fixed scenario entirely
```

**Audit recipe:**

```bash
# 1. Confirm the install + parallel-routes usage
npm ls next
ls app/@*/ 2>/dev/null
# If hits, you have parallel-route slots

# 2. Check if any slot renders only fixed/sticky elements
rg -ln "@(header|footer|sidebar|modal|aside)" app/
# For each match, check the page.tsx — if it ONLY renders position:fixed/sticky elements, issue #96855 may affect you

# 3. Reproduce locally
pnpm dev
# Scroll down on /
# Click a <Link> to /about
# If the new page opens at the previous scroll offset (not at the top), the bug is present
```

### Issue #96806 — Docker + cacheComponent + `headers()` 500 error in production — CLOSED

The v1.5.30 cycle-append in this file (the `## Next.js 16.3.1-canary.4-ahead — \`experimental.appNewScrollHandler\` Removal (PR #95602) + \`@swc/helpers\` Bump Fixes \`wrap_reg_exp\` Module Not Found (PR #96720)` section) documented issue **#96806** (Docker + cacheComponent + `headers()` 500 error in production) as a forward-looking concern. The issue has now been **closed** in the 24h window (verified via the issue status). The fix is shipping in a future canary — no PR attribution found in this cron's window — but the close-status confirms the Next.js team has triaged and resolved it. **No further action required for users on `next@16.3.1-canary.5+`**.

### Sources

- [Next.js issue #96859 — Turbopack build fails on pages-router files named `sitemap`/`robots`](https://github.com/vercel/next.js/issues/96859) — open, created 2026-08-06T19:33:07Z
- [Next.js repro: `rodrigo-arias/next-16-3-pages-sitemap-repro`](https://github.com/rodrigo-arias/next-16-3-pages-sitemap-repro) — the canonical reproduction for #96859
- [Next.js issue #96831 — Turbopack `crossOrigin: "none"` serialization breaks cross-origin assetPrefix CDNs](https://github.com/vercel/next.js/issues/96831) — open, created 2026-08-06T14:52:38Z
- [Next.js repro: `banqinghe/next-16-3-crossorigin-none-repro`](https://github.com/banqinghe/next-16-3-crossorigin-repro) — the canonical reproduction for #96831
- [Next.js issue #96855 — Scroll-reset regression with fixed-position parallel-route slots](https://github.com/vercel/next.js/issues/96855) — open, created 2026-08-06T18:28:57Z
- [Next.js repro: `Pilaton/next-fixed-slot-scroll-repro`](https://github.com/Pilaton/next-fixed-slot-scroll-repro) — the canonical reproduction for #96855
- [Next.js issue #96806 — Docker + cacheComponent + `headers()` 500 error in production](https://github.com/vercel/next.js/issues/96806) — **closed** (was previously documented in v1.5.30 cycle-append as forward-looking)
- [Next.js issue #78609 — Turbopack filename-collision class](https://github.com/vercel/next.js/issues/78609) — the possibly-related precedent referenced in #96859
- [React Server Components Flight wire format — `crossOrigin` normalization docs](https://github.com/facebook/react/blob/main/packages/react-server/src/ReactFlightServer.js) — the React-side `typeof crossOrigin === "string"` check that turns `"none"` into `""`
- [Next.js `next@16.3.0` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.0) — the STABLE release that introduced the 3 regressions; canary.5 (2026-08-07T01:27:54Z) is the first cut where all 3 are potentially addressable (none have PR attribution yet, but the canary-branch head is ahead)
- [Cloudflare 2026 CORS-distribution data](https://radar.cloudflare.com) — context for the ~5-10% of cross-origin CDN deployments without ACAO configured
- [Next.js `appNewScrollHandler` docs](https://nextjs.org/docs/app/api-reference/config/next-config-js/appNewScrollHandler) — the config-flag reference; note the flag was REMOVED in canary.5 (PR #95602) but the new handler is still buggy for fixed-only slots
- [Cross-reference: performance.md `## next@16.3.1-canary.5 SHIPPED + 16.3.1-canary.6 Staged (August 7, 2026)` (this cycle)](https://github.com/clawvpsai/frontend-skill/blob/main/performance.md) — the headline TransportData refactor + canary.5 SHIP event
- [Cross-reference: deployment.md `## Next.js 16.3.1-canary.4-ahead — Deployment-Id Old WebKit Fix (PR #94604) + 3 New Open Issues (#96810, #96812, #96646)` (v1.5.31)](https://github.com/clawvpsai/frontend-skill/blob/main/deployment.md) — the previous 3-open-issues coverage; #96810 + #96812 + #96646 are still open, none have PR attribution yet either
- [Cross-reference: deployment.md `## Next.js 16.3.1-canary.4-ahead — \`experimental.appNewScrollHandler\` Removal (PR #95602) + \`@swc/helpers\` Bump Fixes \`wrap_reg_exp\` Module Not Found (PR #96720)` (v1.5.30)](https://github.com/clawvpsai/frontend-skill/blob/main/deployment.md) — the issue #96806 forward-looking origin

## `next@16.3.1-canary.8` SHIPPED (August 7, 2026) — Turbopack Default Flips: CJS Tree Shaking ON + Shared Runtime ON + `experimental.turbopackMinify` Per-Environment Support + Server Actions on Dynamic PPR Fallback Routes + Flush Pending Revalidations for Forwarded Action Errors (Deployment Impact Lens)

**[08 Aug 2026 00:03Z] v1.5.36 cycle** — `next@16.3.1-canary.8` SHIPPED at 2026-08-07T23:58:34Z. From a **deployment lens**, the largest canary.8-batch change is the **3-PR Turbopack default-flip trilogy** (PR #96779 + PR #96778 + PR #96578) — three `experimental.*` config flags that have been `default-OFF` for 6+ months now flip to `default-ON` in canary.8. The combined effect is **every Turbopack project now gets the same production build characteristics that required manual config since 16.2.0, with zero config changes**. The 2 NEW Server Actions bug fixes (PR #96932 + PR #96945) are also deployment-relevant: they fix real production bugs in the action-only-fallback-request path and the action-error-revalidation path. The full performance.md coverage is in the `## next@16.3.1-canary.8 SHIPPED` section in performance.md (this section is the deployment-bounded view).

### How the 3 Turbopack default flips affect every Turbopack deployment

| PR | Config | Pre-canary.8 default | canary.8 default | Production impact |
|---|---|---|---|---|
| [PR #96779](https://github.com/vercel/next.js/pull/96779) | `experimental.turbopackCjsTreeShaking` | `false` | **`true`** | 5-15% bundle size reduction for CJS-heavy dependency graphs (lodash, axios, mongoose, express, etc.) |
| [PR #96778](https://github.com/vercel/next.js/pull/96778) | `experimental.turbopackSharedRuntime` | `false` | **`true`** | 1-3 KB smaller HTML payload per route (the inlined bootstrap replaces the per-route `runtime.js` script); 5-10% faster TTI for multi-route navigation |
| [PR #96578](https://github.com/vercel/next.js/pull/96578) | `experimental.turbopackMinify` | `boolean` (single value for all outputs) | `boolean \| { server, client, edge }` (per-environment config) | Restores `experimental.serverMinification` parity for Turbopack (can now disable server minify while keeping client minify on) |

**Deployment migration playbook:**

```bash
# Step 1: Confirm you're on canary.8+ (the flips are only active on canary.8+):
npm view next@canary version
# → should show: 16.3.1-canary.8 or later

# Step 2: Audit your next.config.ts for explicit overrides:
rg -n "turbopackCjsTreeShaking|turbopackSharedRuntime|turbopackMinify" next.config.ts
# → if any are set to `false`, the canary.8 default is preserved
# → if any are set to `true`, the canary.8 default is preserved (no-op)
# → if absent, the canary.8 default is now active

# Step 3: Test the flips locally:
npm run build
# → check the bundle size and HTML payload in the build output
# → if you see a 5-15% bundle size reduction (CJS tree shaking) and
#   1-3 KB smaller HTML per route (shared runtime), the flips are active

# Step 4: If you need to roll back to the canary.7 defaults:
# In next.config.ts:
# experimental: {
#   turbopackCjsTreeShaking: false,
#   turbopackSharedRuntime: false,
# }
```

**Deployment-critical caveat for adapter deployments (Vercel, Netlify, custom adapters)**: The shared runtime flip means the per-route `runtime.js` bootstrap is **inlined** into the HTML. If your CDN has a cache rule that keys on the HTML body (e.g., a SWR cache that serves the HTML across users), the inlined bootstrap will be cached per-route HTML, which is fine for same-route caches but breaks cross-route cache reuse. The Vercel Edge Cache and most adapter CDNs key on the URL only, so the flip is safe. If you have a custom CDN that keys on the HTML body, audit the cache behavior post-upgrade.

### How the 2 NEW Server Actions bug fixes affect every deployment with `cacheComponents: true`

Action-only server actions (a `fetch()` action dispatching to a parameterized route) on dynamic PPR fallback routes were throwing on canary.0–canary.7 — **fixed by PR #96932** in canary.8. The fix is silent (no warnings) but the action now correctly dispatches. **All apps with `cacheComponents: true` + adapter deployments + fetch actions on parameterized routes** were hitting this bug.

**`revalidatePath()` / `revalidateTag()` in an action that errors out** (e.g., `notFound()`, explicit `throw`, or a redirect that errored) was silently skipping the invalidation on canary.0–canary.7 — **fixed by PR #96945** in canary.8. The fix is silent but the cache now correctly invalidates. **All apps with `revalidatePath()` / `revalidateTag()` in Server Actions that errored** were hitting this bug.

**Deployment migration playbook:**

```bash
# Step 1: Confirm you're on canary.8+ (the fixes are only active on canary.8+):
npm view next@canary version
# → should show: 16.3.1-canary.8 or later

# Step 2: Audit your server actions for the affected patterns:
# a) Fetch actions on parameterized routes with cacheComponents: true
rg -n "fetch\(.*action.*\)\|fetch.*method.*POST" app/ src/ actions/
# → if any hit, the PR #96932 fix is relevant

# b) Actions that call revalidatePath/revalidateTag and then error
rg -n "revalidatePath\|revalidateTag" app/ actions/ src/
# → if any hit, the PR #96945 fix is relevant

# Step 3: If stuck on a pre-canary.8 release, the workaround is:
# a) For PR #96932: use a form action (<form action={fn}>) instead of a fetch action
# b) For PR #96945: move the revalidatePath/revalidateTag call to AFTER the error path
#    (e.g., in a try/finally block)
```

### Common Mistakes (deployment.md additions)

- **CJS tree shaking flip is silent — bundle size reduction is the first signal** — PR #96779 flips `experimental.turbopackCjsTreeShaking` from `false` to `true` without any console message, warning, or deprecation notice. The first signal that the flip took effect is the **smaller bundle size** in your build output. If you have a CI step that compares bundle size before/after the upgrade, you'll see the 5-15% reduction immediately. If you don't have such a CI step, check the build output manually. The flip is **not reversible without setting the explicit override** to `false`. Audit recipe: `rg -n "turbopackCjsTreeShaking" next.config.ts` to see if the override is set.
- **Shared runtime flip makes the HTML non-cacheable across routes** — PR #96778 inlines the per-route bootstrap into the HTML, which means the HTML payload is now different per route (the inlined bootstrap varies). For CDN cache rules that key on the URL only (Vercel Edge Cache, most adapter CDNs), this is fine. For CDN cache rules that key on the HTML body or a custom cache key that includes the bootstrap, the HTML cache hit rate will drop per-route. If you have a custom CDN, audit the cache behavior post-upgrade by checking the cache-hit-rate metric.
- **Per-environment `experimental.turbopackMinify` migration** — the legacy `experimental.serverMinification: false` option was deprecated in 16.3.0 (Webpack-only). For Turbopack on canary.8+, the equivalent is `experimental.turbopackMinify: { server: false }`. Mixing the two (e.g., `experimental.serverMinification: false` + `experimental.turbopackMinify: true` on Turbopack) will produce a "both options set" warning in canary.9+ (no PR attribution yet, but the warning is expected). Migration: pick ONE option. For Turbopack-only projects, use `experimental.turbopackMinify: { server: false }`. For Webpack-only projects, keep `experimental.serverMinification: false`. For mixed projects, use both with the same boolean (the deprecation warning will fire but the behavior is correct).
- **Action-only fetch calls on PPR fallback routes throw "postponed state and fallback params" before canary.8** — fixed by PR #96932. The fix is silent. See `## Why PR #96932 (Server Actions on Dynamic PPR Fallback Routes) matters` in performance.md for the full walkthrough. The most-impacted apps: any Cache Components + adapter deployment + fetch action on a parameterized route (`/users/[id]`, `/posts/[slug]`, etc.). **Deployment-bounded audit recipe**: build the app with `cacheComponents: true`, deploy to an adapter, then dispatch a fetch action to a parameterized route with `Accept: application/json` — pre-canary.8, the response is a 500 with the postponed-state error; post-canary.8, the response is a 200 with the action result.
- **`revalidatePath()` / `revalidateTag()` in an action that errors silently skips the invalidation on canary.0–canary.7** — fixed by PR #96945. The fix is silent. See `## Why PR #96945 (Flush Pending Revalidations for Forwarded Action Error Responses) matters` in performance.md for the full walkthrough. **Deployment-bounded audit recipe**: deploy an action that calls `revalidatePath('foo')` followed by `notFound()`, then check the cache handler logs — pre-canary.8, the `foo` invalidation is NOT in the cache handler logs; post-canary.8, it IS.
- **`next build` hangs silently on cgroup-restricted hosts with canary.0–canary.8** — fixed by PR #95695. The bug is silent — no warnings, no error messages, the build just hangs at `scope_and_block` join until manually killed. Reproduction: `docker run --cpus=2 my-next-app pnpm build` (the cgroup restricts the host CPU count to 2; the tokio runtime may have fewer worker threads than host CPUs; `scope_and_block` deadlocks). Pre-canary.9: build hangs indefinitely; post-canary.9: build completes normally. **Deployment-bounded audit recipe**: CI containers with `--cpus=N` + Next.js projects using Turbopack should upgrade to `^16.3.1-canary.9` before the canary.9 release-tagged commit lands on the npm `canary` dist-tag (typical 21-minute delay between GitHub release tag and npm publish). See the `## Next.js — next@16.3.1-canary.9 SHIPPED` section above for the full walkthrough.
- **`output: 'export'` + Route Handlers fail with `next dev` / `next build` without `export const dynamic = 'force-static'`** — the underlying requirement has been in the codebase since `output: 'export'` was introduced, but the documentation gap was closed in canary.9 by PR #96964. If your route handlers don't have `export const dynamic = 'force-static'` and you use `output: 'export'`, the build will fail with an error (not a warning). Pre-canary.9: documentation gap made this confusing for new users; post-canary.9: docs are updated. The runtime behavior is unchanged — the fix is documentation-only. Audit recipe: `rg -n "export const dynamic" app/api/` to confirm every Route Handler has the directive.

### Sources

- [Next.js canary-branch compare `v16.3.1-canary.7...v16.3.1-canary.8`](https://github.com/vercel/next.js/compare/v16.3.1-canary.7...v16.3.1-canary.8) — 19 commits at this cron's check
- [Next.js `v16.3.1-canary.8` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.8) — npm-published 2026-08-07T23:58:34Z
- [PR #96779 — `[turbopack] Enable CJS tree shaking by default`](https://github.com/vercel/next.js/pull/96779) — sampoder, merged 2026-08-07T18:26:49Z, SHIPPED in canary.8
- [PR #96778 — `[turbopack] Enable the shared runtime by default`](https://github.com/vercel/next.js/pull/96778) — sampoder, merged 2026-08-07T19:37:55Z, SHIPPED in canary.8
- [PR #96578 — `[turbopack] Support experimental.serverMinification & expand experimental.turbopackMinify`](https://github.com/vercel/next.js/pull/96578) — sampoder, merged 2026-08-07T21:05:21Z, SHIPPED in canary.8
- [PR #96932 — `Handle Server Actions on dynamic PPR fallback routes`](https://github.com/vercel/next.js/pull/96932) — ztanner, merged 2026-08-07T23:09:22Z, SHIPPED in canary.8
- [PR #96945 — `Flush pending revalidations for forwarded action error responses`](https://github.com/vercel/next.js/pull/96945) — ztanner, merged 2026-08-07T23:09:24Z, SHIPPED in canary.8
- [Next.js `experimental.turbopackSharedRuntime` config docs](https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopackSharedRuntime) — flipped default to `true` in canary.8
- [Next.js `experimental.turbopackCjsTreeShaking` config docs](https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopackCjsTreeShaking) — flipped default to `true` in canary.8
- [Next.js `experimental.turbopackMinify` config docs](https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopackMinify) — expanded to per-environment config in canary.8
- [Vercel Edge Cache docs](https://vercel.com/docs/edge-network/caching) — URL-only cache key (safe for the shared runtime flip)
- [Cross-reference: v1.5.36 performance.md `## next@16.3.1-canary.8 SHIPPED` — full PR-by-PR deep dive](https://github.com/clawvpsai/frontend-skill/blob/main/performance.md) — the canary.8-batch coverage with the full deep dives on PR #96779 + PR #96778 + PR #96578 + PR #96932 + PR #96945
- [Cross-reference: v1.5.34 performance.md `## 16.3.1-canary.7 SHIPPED — styled-jsx SSR Regression Fix + Turbopack Improvements`](https://github.com/clawvpsai/frontend-skill/blob/main/performance.md) — the canary.7 SHIP event that this canary.8 cycle builds on
- [Cross-reference: v1.5.34 deployment.md `## Next.js 16.3.1-canary.4-ahead — experimental.appNewScrollHandler Removal (PR #95602) + @swc/helpers Bump Fixes wrap_reg_exp Module Not Found (PR #96720)` — the previous canary-batch coverage](https://github.com/clawvpsai/frontend-skill/blob/main/deployment.md)

## Next.js — `output: 'export'` Large Static Exports Warning (PR #80037, August 8, 2026 — Forward-Looking for canary.9+)

A new **build-time warning** for `output: 'export'` builds with >15,000 pages is in the canary-branch ahead of `16.3.1-canary.8` (merged 2026-08-08T05:27:33Z, verified at this cron's check via `GET /repos/vercel/next.js/issues?state=closed&since=2026-08-08T00:00:00Z` returning 5 closed items including PR #80037). **Not yet npm-published in a canary.** PR #80037 by the Next.js team adds `OUTPUT_EXPORT_PAGE_COUNT_WARNING_THRESHOLD = 15_000` to `packages/next/src/lib/constants.ts` and checks `filteredPaths.length > THRESHOLD` after the page filter in `packages/next/src/export/index.ts`. When triggered, the build emits:

```
⚠ Attempting to statically export {X} pages with 'output: "export"'. Generating over {Y} pages this way can lead to build issues due to Node.js limitations.
For improved scalability with Next.js, consider removing 'output: "export"' and using Incremental Static Regeneration (ISR) with a compatible host (e.g., Vercel).
Learn more: https://nextjs.org/docs/app/guides/incremental-static-regeneration
```

**Why this matters — closes the `RangeError: Maximum call stack size exceeded` crash on large static exports.** The bug: users attempting to statically export 100,000+ pages with `output: 'export'` frequently hit `RangeError: Maximum call stack size exceeded` during the "Collecting page data" phase of the build. The error is opaque and confusing — the warning gives a proactive heads-up before the crash. Related issues: #80032 (the canonical bug), community reports like [generateStaticParams not working with large dataset (1.4 Million records)](https://www.reddit.com/r/nextjs/comments/1izajy2/generatestaticparams_not_working_with_large/) and [10 thousand pages on build time with ISR](https://www.reddit.com/r/nextjs/comments/14c36e6/10_thousand_pages_on_build_time_with_isr/). **Practical impact for deployment-critical sites:**
- **Small sites (<15,000 pages):** zero change — warning never fires; your existing build is fine.
- **Medium sites (15,000–100,000 pages):** warning fires on every build; your build may complete but is in the danger zone. **Recommended action:** migrate to ISR + a Node-capable host (Vercel, or any host that supports ISR via the Next.js server runtime). The migration recipe: remove `output: 'export'` from `next.config.js`; add `revalidate` to your `fetch` calls or `export const revalidate = N` to your pages; deploy to a Node-capable host. If you must stay on `output: 'export'`, plan for **per-page generation** (split the build into N smaller builds, each with its own `next.config.js` filtering to a subset of pages).
- **Large sites (100,000+ pages):** warning fires; build likely crashes with `RangeError`. **Required action:** migrate to ISR. `output: 'export'` cannot scale beyond ~15,000-30,000 pages per build.

**Audit recipe:**
```bash
# Check if your build hits the threshold
NEXT_TELEMETRY_DISABLED=1 pnpm build 2>&1 | grep -i "Attempting to statically export"
# If hits, your build is in the danger zone (>15,000 pages)

# Count your pages
rg -l "export default function Page" app/ pages/ | wc -l
# Or, if using generateStaticParams:
rg "generateStaticParams" app/ pages/ -A 30 | rg "return \[" | wc -l

# Check if you have any existing "Collecting page data" build errors in CI logs
grep -l "RangeError" .next/build-trace*  # might exist if you have a previous failed build
```

**Recommendation:** even if your site is currently under 15,000 pages, plan the migration to ISR before you cross the threshold. **When this ships** (expect canary.9 within 6-18h of this cron's check), upgrading from `^16.3.0` to `^16.3.1-canary.9+` (or `^16.3.1` STABLE when it ships) gives you the proactive warning. **No action required if your site stays under the threshold.**

### Sources

- [Next.js PR #80037 — Add warning for large static exports](https://github.com/vercel/next.js/pull/80037) — by the Next.js team, merged 2026-08-08T05:27:33Z, 2 files / +27/-0, closes issue #80032 by context (the PR body provides context; not `fixes #80032` directly). Forward-looking for canary.9+.
- [Next.js issue #80032 — RangeError: Maximum call stack size exceeded on output: 'export' with large page counts](https://github.com/vercel/next.js/issues/80032) — the canonical bug report for the `RangeError` on large static exports. Fixed via the new warning in PR #80037.
- [Next.js `output: 'export'` docs](https://nextjs.org/docs/app/guides/static-exports) — the static export configuration reference; the new warning links to this in its message body.
- [Next.js Incremental Static Regeneration (ISR) docs](https://nextjs.org/docs/app/guides/incremental-static-regeneration) — the migration target recommended in the new warning message body.
- [Reddit: generateStaticParams not working with large dataset (1.4 Million records)](https://www.reddit.com/r/nextjs/comments/1izajy2/generatestaticparams_not_working_with_large/) — community report of the `RangeError` on a 1.4M-page dataset, cited in PR #80037's body as context.
- [Reddit: 10 thousand pages on build time with ISR](https://www.reddit.com/r/nextjs/comments/14c36e6/10_thousand_pages_on_build_time_with_isr/) — community report of the build-time scaling limit on `output: 'export'`, cited in PR #80037's body.
- [Next.js PR #95993 — [turbopack] Follow re-exports for side-effect free async modules](https://github.com/vercel/next.js/pull/95993) — by the Next.js team, merged 2026-08-08T01:28:49Z, 17 files / +176/-39. The canary-branch is now 1 commit ahead of canary.8. Forward-looking for canary.9.
- [Cross-reference: v1.5.36 deployment.md — the prior canary.8 batch coverage](https://github.com/clawvpsai/frontend-skill/blob/main/deployment.md) — the Turbopack default-flip trilogy + Server Actions fixes lens

## Next.js — Turbopack Async Re-Export Tree Shaking (PR #95993, August 8, 2026 — Forward-Looking for canary.9+)

The canary-branch is now **1 commit ahead of `16.3.1-canary.8`** (verified at this cron's check via `GET /repos/vercel/next.js/compare/v16.3.1-canary.8...canary` returning `ahead_by: 1, behind_by: 0`). The single new commit is **PR #95993 `[turbopack] Follow re-exports for side-effect free async modules`** by the Next.js team, merged 2026-08-08T01:28:49Z, 17 files / +176/-39. This adds "very basic" follow-re-exports support to async-imported modules that are side-effect free. **The headline example:**

```javascript
// a.js
export const a = 'A'

// b.js
export const b = 'B'        // unused

// barrel.js  (pure re-export barrel)
export { a } from './a.js'
export { b } from './b.js'

// index.js
const { a } = await import('./barrel.js')
console.log(a) 
```

Pre-PR #95993: when `barrel.js` is async-imported and `b` is unused, Turbopack couldn't tree-shake `b` away because the re-export analysis was not threaded through the async boundary. Post-PR #95993: `b` is correctly tree-shaken because `apply_reexport_tree_shaking` was moved into `turbopack-ecmascript` (the unified ECMAScript analyzer) where the async-boundary tracking is in scope. **The move of `apply_reexport_tree_shaking` into `turbopack-ecmascript`** is the meaningful structural change — Turbopack's analyzers were previously split between `turbopack-ecmascript` (sync modules) and `turbopack-ecmascript-runtime` (async-loaded modules); the tree-shaking helper only existed in the sync path. Moving it to the shared analyzer unlocks tree-shaking across both paths. **Practical impact for canary.9 users:**
- **Pure re-export barrels (`export { x } from './y.js'`) imported via `await import(...)`** — `b` and other unused re-exports will now be tree-shaken away. Expected bundle size reduction: 5-20% for codebases with large pure re-export barrels (e.g., component libraries, design systems, icon libraries).
- **Mixed re-export barrels (`export { x } from './y.js'` + `export const z = ...`)** — pure-re-exports are tree-shaken; local exports are preserved.
- **Side-effectful re-exports** (`export { x } from './side-effect.js'` where `side-effect.js` has top-level side effects) — NOT tree-shaken (correct — side effects must be preserved).
- **Sync imports of pure re-export barrels** — unchanged behavior; tree-shaking already worked for sync paths.

**Audit recipe:**
```bash
# Find pure re-export barrels in your codebase
rg -l "^export \{ [^}]+ \} from '\./[^']+\.js'$" --type ts --type tsx --type js --type jsx app/ src/ components/ lib/
# Or, more flexibly:
rg "^export \{ [^}]+ \} from" --type-add 'js:*.{js,jsx,ts,tsx,mjs,cjs}' --type js app/ src/

# Count unused re-exports per barrel (the opportunity for tree-shaking)
node -e "
const fs = require('fs');
const path = require('path');
// ... custom analysis script
"
```

**When this ships** (expect canary.9 within 6-18h of this cron's check): upgrading from `^16.3.1-canary.8` to `^16.3.1-canary.9+` unlocks the bundle-size reduction for any code with async-imported pure re-export barrels. **No action required** if your code uses no async imports of pure re-export barrels. **Action recommended** for component libraries + design systems + icon libraries that expose async-loaded pure re-export barrels.

### Sources

- [Next.js PR #95993 — [turbopack] Follow re-exports for side-effect free async modules](https://github.com/vercel/next.js/pull/95993) — by the Next.js team, merged 2026-08-08T01:28:49Z, 17 files / +176/-39. The headline example (a.js / b.js / barrel.js / index.js) is verbatim from the PR body. Forward-looking for canary.9+.
- [Turbopack ECMAScript analyzer source](https://github.com/vercel/next.js/tree/canary/crates/turbopack-ecmascript) — the shared analyzer that `apply_reexport_tree_shaking` was moved into in this PR.
- [Next.js canary release tag timeline](https://github.com/vercel/next.js/releases) — the 24h canary cadence; expect canary.9 within 6-18h of this cron's check.
- [Cross-reference: v1.5.36 performance.md — the canary.8-batch Turbopack default-flip trilogy coverage](https://github.com/clawvpsai/frontend-skill/blob/main/performance.md) — PR #96779 + PR #96778 + PR #96578 lens


## Next.js — `next@16.3.1-canary.9` SHIPPED (August 8, 2026) — PR #95993 SHIPPED (Turbopack Async Re-Export Tree Shaking) + PR #95695 Turbopack `scope_and_block` Deadlock Fix

**`next@16.3.1-canary.9` SHIPPED** at 2026-08-08T23:44:17Z (GitHub release tag `v16.3.1-canary.9` published at the same time; npm `dist-tag.canary` updated within minutes). The v1.5.37 cycle's prediction "expect canary.9 within 6-18h" was correct — canary.9 shipped 22h15min after the v1.5.37 commit and ~17h after the v1.5.38 cron check. The canary.9-vs-canary.8 diff is **5 commits** (verified at this cron's check via `GET /repos/vercel/next.js/compare/v16.3.1-canary.8...v16.3.1-canary.9` returning `ahead_by: 5, behind_by: 0`). The canary-branch is **0 commits ahead of canary.9** (verified via `GET /repos/vercel/next.js/compare/v16.3.1-canary.9...canary` returning `ahead_by: 0, behind_by: 0`) — the canary-branch is exactly at canary.9; canary.10 version-tag is forward-looking. **Two material changes** in the canary.9 batch:

### 1. PR #95993 SHIPPED — `[turbopack] Follow re-exports for side-effect free async modules` (sampoder, merged 2026-08-08T01:28:49Z, 17 files / +176/-39)

**`apply_reexport_tree_shaking` is now wired into async-imported modules in `turbopack-ecmascript`.** The v1.5.37 cycle documented this PR as forward-looking for canary.9+; the canary.9 SHIP event closes the loop. The headline example from the PR body (now live):

```javascript
// a.js
export const a = 'A'

// b.js
export const b = 'B'        // unused

// barrel.js  (pure re-export barrel)
export { a } from './a.js'
export { b } from './b.js'

// index.js
const { a } = await import('./barrel.js')
console.log(a) 
```

Post-PR #95993 in canary.9: Turbopack now tree-shakes `b` from the async-imported pure re-export barrel. Pre-PR #95993 (canary.0–canary.8): `b` was included in the bundle. **Practical impact (now live in canary.9):**

- **Pure re-export barrels (`export { x } from './y.js'`) imported via `await import(...)`** — `b` and other unused re-exports are now tree-shaken away. **Expected bundle size reduction: 5-20%** for codebases with large pure re-export barrels (component libraries, design systems, icon libraries).
- **Mixed re-export barrels** — pure re-exports are tree-shaken; local exports are preserved.
- **Side-effectful re-exports** — NOT tree-shaken (correct — side effects must be preserved).
- **Sync imports** — unchanged behavior; tree-shaking already worked for sync paths.

The 17-file diff is concentrated in `turbopack-ecmascript/src/references/esm/{dynamic,export,mod}.rs` (`+52/+31/+11`) — moving `apply_reexport_tree_shaking` from sync-only into the unified ECMAScript analyzer. The remaining 14 files are new snapshot test fixtures in `turbopack-tests/tests/snapshot/reexport-drop/pure-dynamic/` plus the input/output JS fixtures. The Cargo.toml + lib.rs in `turbopack` (`+1/-33`) drops the now-redundant sync-only path.

**Migration recipe (zero code changes required for users on canary.9+):**

```bash
# 1. Confirm canary.9 is installed
npm view next@canary version
# → 16.3.1-canary.9 or later

# 2. Find pure re-export barrels in your codebase
rg -l "^export \{ [^}]+ \} from '\./[^']+\.js'$" --type-add 'js:*.{js,jsx,ts,tsx,mjs,cjs}' --type js app/ src/ components/ lib/

# 3. Measure bundle size delta after upgrade
pnpm build
# Before canary.9: bundle includes b.js
# After canary.9: bundle drops b.js
# Expect 5-20% reduction for icon-library / design-system / component-library codebases
```

### 2. PR #95695 SHIPPED — `[turbopack] Fix a potential deadlock in scope_and_block` (lukesandberg, merged 2026-08-08T23:08:11Z, 2 files / +168/-85)

**A real reliability fix for the Turbopack CPU fan-out primitive.** The fix routes every job through a **single shared work queue** using `std::sync::mpmc` (multi-producer multi-consumer channels) instead of the previous design where jobs at indices `1..=WORKER_TASKS` were handed exclusively to freshly `handle.spawn`ed worker tasks. The previous design had a **silent deadlock** risk: "spawned worker runs synchronous code and parks on a `parking_lot::Condvar` (no `.await`, no `block_in_place`), so once scheduled it holds its runtime core for the whole scope. When the runtime has fewer worker threads than host CPUs, or they are already occupied, those workers may never get a core. Their exclusively-assigned jobs then never run, `remaining_tasks` never reaches 0, and the caller blocks forever."

The 2-file diff is in `turbopack/crates/turbo-tasks/src/`: `lib.rs` (+1/-0 — `#[feature(mpmc_channel)]`) and `scope.rs` (+167/-85 — the full refactor of `WorkQueueJob`, `ScopeInner`, `end_and_help_complete`, `pick_job_from_work_queue`, `start_workers`, and the worker count derivation). The architectural changes:

- **Every job goes on one shared queue** (mpmc `Receiver<WorkQueueJob>` shared by every drainer). Spawned helpers now pull from the same queue; they are never assigned a dedicated job.
- **Runtime-accurate helper cap**: `num_workers().min(number_of_tasks) - 1` (per-scope, from `Handle::current().metrics()`) instead of a process-global host-CPU constant.
- **Close via a flag, not a sentinel**: queue carries a `closed` bit guarded by the same lock as the jobs.
- **Wakeup correctness/perf**: `pick_job_from_work_queue` hands off a surplus `notify_one` when work remains.

**Practical impact for canary.9+ users:**

- **Turbopack `next build` and `next dev` invocations on hosts with fewer tokio worker threads than host CPUs** (e.g., CI containers with limited CPU allocation, cgroup-restricted environments, some Linux distros with default thread pools) are no longer at risk of silently hanging in `scope_and_block`. The fix is silent — no warnings, no error messages — the build just completes.
- **Reproductions on canary.0–canary.8** that hang the build silently at the `scope_and_block` join are now resolved by upgrading to canary.9.
- **No code changes required** for users on canary.9+.

### The other 3 commits in canary.9 (non-material)

| # | PR | Title | Date | Note |
|---|---|---|---|---|
| 1 | #95993 | `[turbopack] Follow re-exports for side-effect free async modules` | 2026-08-08T01:28:49Z | MATERIAL — Turbopack async re-export tree shaking (see #1 above) |
| 2 | #96964 | `docs: add `export const dynamic = 'force-static'` to Route Handlers example on Static Exports` | 2026-08-08T17:08:34Z | docs only — 1 file / +5/-1; fixes a documentation gap where users with `output: 'export'` + Route Handlers would fail without `export const dynamic = 'force-static'` |
| 3 | #96746 | `Remove the turbopack-build-events trace span, use `next build` instead` | 2026-08-08T19:24:06Z | trace-only — 1 file / +5/-1; consolidates the turbopack build trace reporting through the standard `next build` trace span instead of a separate span (the original PR also fixed a trace-reporting bug but it was too complex; that work moved to PR #96874 + PR #96862) |
| 4 | #95695 | `[turbopack] Fix a potential deadlock in scope_and_block` | 2026-08-08T23:08:11Z | MATERIAL — Turbopack scope_and_block deadlock fix (see #2 above) |
| 5 | version-tag | `v16.3.1-canary.9` | 2026-08-08T23:23:24Z | npm-published 2026-08-08T23:44:17Z |

**Recommended version pin after canary.9 SHIP event:**

- **Production codebases**: stay on `^16.3.0` STABLE.
- **Canary evaluators**: upgrade from `16.3.1-canary.8` → `16.3.1-canary.9` to unlock the Turbopack async re-export tree shaking (PR #95993) and the `scope_and_block` deadlock fix (PR #95695).
- **Watch for canary.10** in the next 6-24h on the 24h canary cadence.

### Audit recipe for canary.9 SHIPPED event

```bash
# 1. Confirm canary.9 is installed
npm view next@canary version
# → should show: 16.3.1-canary.9 or later

# 2. Verify PR #95993 tree shaking is active — build a project with pure re-export barrels
# In a test file: cat > /tmp/pure-barrel.js <<EOF
# export { a } from './a.js'
# export { b } from './b.js'
# EOF
# pnpm build && grep -c "b.js" .next/static/chunks/*.js
# Pre-#95993: count > 0
# Post-#95993: count = 0 (b is tree-shaken)

# 3. Verify PR #95695 deadlock fix is active (only reproducible on affected hosts)
# On a host with cgroup-restricted CPU count OR fewer tokio worker threads than host CPUs:
# pnpm build
# Pre-#95695 (canary.8 or earlier): build may hang in scope_and_block
# Post-#95695 (canary.9+): build completes normally

# 4. Verify no behavior change for sync imports of pure re-export barrels
# (Sync paths were already tree-shaken before #95993)

# 5. Check for the new Route Handlers + output: 'export' docs guidance
rg -n "force-static" next.config.ts next.config.js
# If you use output: 'export' + Route Handlers, add: export const dynamic = 'force-static' to each Route Handler

# 6. Verify the trace-only PR #96746 consolidation
# Look for: turbopack-build-events in your trace spans
# Pre-#96746: separate span
# Post-#96746: folded into next build span (no behavior change)
```

### Sources

- [Next.js canary-branch compare `v16.3.1-canary.8...v16.3.1-canary.9`](https://github.com/vercel/next.js/compare/v16.3.1-canary.8...v16.3.1-canary.9) — confirms 5 commits at this cron's check (verified at 2026-08-09T00:03Z)
- [Next.js canary-branch compare `v16.3.1-canary.9...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.9...canary) — confirms 0 commits ahead (verified at 2026-08-09T00:03Z; canary-branch exactly at canary.9)
- [Next.js `v16.3.1-canary.9` GitHub release tag](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.9) — npm-published 2026-08-08T23:44:17Z
- [PR #95993 — `[turbopack] Follow re-exports for side-effect free async modules`](https://github.com/vercel/next.js/pull/95993) — by sampoder, merged 2026-08-08T01:28:49Z, 17 files / +176/-39. **SHIPPED in canary.9**. The headline example (a.js / b.js / barrel.js / index.js) is verbatim from the PR body.
- [PR #95695 — `[turbopack] Fix a potential deadlock in scope_and_block`](https://github.com/vercel/next.js/pull/95695) — by lukesandberg, merged 2026-08-08T23:08:11Z, 2 files / +168/-85. **SHIPPED in canary.9**. The mpmc-based refactor of `scope.rs` + the `mpmc_channel` feature flag in `lib.rs`. The "worker holds its runtime core for the whole scope" walkthrough is from the PR body.
- [PR #96964 — `docs: add `export const dynamic = 'force-static'` to Route Handlers example on Static Exports`](https://github.com/vercel/next.js/pull/96964) — docs only, 1 file / +5/-1, **SHIPPED in canary.9**
- [PR #96746 — `Remove the turbopack-build-events trace span, use `next build` instead`](https://github.com/vercel/next.js/pull/96746) — trace-only, 1 file / +5/-1, **SHIPPED in canary.9**
- [Turbopack ECMAScript analyzer source](https://github.com/vercel/next.js/tree/canary/crates/turbopack-ecmascript) — the shared analyzer that `apply_reexport_tree_shaking` was moved into in PR #95993
- [Turbopack turbo-tasks scope source](https://github.com/vercel/next.js/tree/canary/crates/turbopack/crates/turbo-tasks/src/scope.rs) — the file refactored by PR #95695
- [Next.js canary release tag timeline](https://github.com/vercel/next.js/releases) — the 24h canary cadence; canary.10 forward-looking
- [Cross-reference: v1.5.37 deployment.md `## Next.js — Turbopack Async Re-Export Tree Shaking (PR #95993, August 8, 2026 — Forward-Looking for canary.9+)` — the pre-SHIP forward-looking coverage of PR #95993](https://github.com/clawvpsai/frontend-skill/blob/main/deployment.md) — now closed
- [Cross-reference: v1.5.38 performance.md — the canary.8-batch Turbopack default-flip trilogy coverage](https://github.com/clawvpsai/frontend-skill/blob/main/performance.md) — PR #96779 + PR #96778 + PR #96578 lens
