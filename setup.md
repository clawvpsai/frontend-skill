# Setup — Project Init, TypeScript, Build Tools

## Project Initialization

### Next.js (App Router)

```bash
npx create-next-app@latest my-app \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*"
```

Flags explained:
- `--typescript` — required for production
- `--tailwind` — use Tailwind CSS v4
- `--eslint` — adds ESLint config; in Next.js 16, use Biome or direct ESLint since `next lint` is removed
- `--app` — App Router (not Pages Router)
- `--src-dir` — put code in `src/` directory
- `--import-alias "@/*"` — `@/*` maps to `src/*`

**Note:** Turbopack is now stable for development in Next.js 16 — no `--no-turbopack` flag needed. It's the default dev bundler. Production builds use Turbopack by default.


**Next.js 16.2 prompts (March 2026):** `create-next-app@latest` now also asks about:
- **Linter choice:** `ESLint` / `Biome` / `None` (default: `ESLint`; choose `Biome` for 10–100x faster linting)
- **React Compiler:** `No` / `Yes` (recommended: `Yes` — eliminates most manual `useMemo`/`useCallback`)
- **AGENTS.md:** `No` / `Yes` (default: `Yes` — see [Next.js 16.2 AI Improvements](#nextjs-162-ai-improvements-march-2026) below)

## Next.js 16.2 AI Improvements (March 2026)

Next.js 16.2 ships four features specifically designed for AI-assisted development. These are significant if you're an AI agent (like this skill's users) or you work alongside one.

### 1. AGENTS.md in `create-next-app` (Default: Yes)

`npx create-next-app@latest` now scaffolds an `AGENTS.md` file at the project root. This file instructs AI coding agents (Claude Code, Cursor, etc.) to read the bundled Next.js documentation from `node_modules/next/dist/docs/` *before* writing any code — giving agents version-matched, local documentation instead of fetching external data.

Vercel's research found that bundled docs achieved a **100% pass rate** on Next.js evals vs. **79%** for skill-based approaches (where agents must decide when to search for docs).

```bash
# Created automatically by create-next-app — do not delete
cat AGENTS.md
# → "Read the docs bundled at node_modules/next/dist/docs/ before writing any code"
```

**For existing projects**, you can opt in manually by creating `AGENTS.md` with the same directive. Vercel's [research on AGENTS.md](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals) shows always-available context beats on-demand retrieval because agents often fail to recognize when they should search.

### 2. Browser Log Forwarding

Browser `console.error` / `console.warn` / unhandled errors are now forwarded to the **terminal** in real time during `next dev`. This lets AI agents debug client-side issues without running a headless browser.

```bash
npm run dev
# → "console.error: TypeError: Cannot read properties of undefined (reading 'map') at ProductList (src/components/product-list.tsx:42:18)"
```

Forwarded log levels (configurable in `next.config.ts`):
- `error` (always forwarded)
- `warn`
- `info` (opt-in)

### 3. Dev Server Lock File

Next.js writes the running dev server's PID, port, and URL to `.next/dev/lock`. A second `next dev` attempt reads it and prints an actionable error — preventing agents (and humans) from accidentally spinning up duplicate servers or corrupting build artifacts.

```
Error: Another next dev server is already running.

- Local:        http://localhost:3000
- PID:          12345
- Dir:          /path/to/project
- Log:          .next/dev/logs/next-development.log

Run kill 12345 to stop it.
```

The lock also blocks two concurrent `next build` processes. If you need to override (stale lock after crash), delete `.next/dev/lock`.

### 4. `@vercel/next-browser` — Experimental CLI for Agents

[`@vercel/next-browser`](https://github.com/vercel-labs/next-browser) is an experimental CLI that exposes browser-level data (screenshots, network requests, console logs) plus framework-specific insights from React DevTools and the Next.js dev overlay — all as structured text via shell commands. An LLM can't read a DevTools panel, but it can run `next-browser tree` and parse the output.

Install as a [skills.sh](https://skills.sh) skill:

```bash
npx skills add vercel-labs/next-browser
```

Then use `/next-browser` in Claude Code, Cursor, or any AI agent that supports skills. The CLI manages a Chromium instance with React DevTools pre-loaded.

**Available commands (as of June 2026):**
| Command | Purpose |
|---|---|
| `next-browser tree` | Inspect React component tree (props, hooks, state, source locations) |
| `next-browser network` | Track requests since navigation, including Server Actions |
| `next-browser logs` | Retrieve build + runtime issues from dev server |
| `next-browser ppr` | Analyze PPR shells — identify static vs dynamic regions and blocked Suspense boundaries |
| `next-browser screenshot` | Capture current viewport (or loading filmstrip) |
| `next-browser errors` | Surface dev overlay errors with `Error.cause` chains |

Each command is a one-shot request against a persistent browser session, so agents can query the app repeatedly without managing browser state. **Note:** `next-browser` is still experimental — APIs may change.

**Sources:**
- [Next.js 16.2: AI Improvements (official blog)](https://nextjs.org/blog/next-16-2-ai)
- [Next.js AI Coding Agents guide](https://nextjs.org/docs/app/guides/ai-agents)
- [`@vercel/next-browser` repo](https://github.com/vercel-labs/next-browser)
- [Vercel research: AGENTS.md outperforms skills (100% vs 79%)](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals)

## Next.js 16.1 Highlights (December 2025)

If you're starting a new project today, you're getting these by default. Worth knowing what they do:

### Turbopack File System Caching (Stable, Default On)

Turbopack now persists compiler artifacts to disk under `.next/`. Restarting `next dev` is dramatically faster — the first route compile time goes from seconds to ~hundreds of milliseconds.

| Project | Cold | Cached | Speedup |
|---|---|---|---|
| react.dev | 3.7s | 380ms | ~10× |
| nextjs.org | 3.5s | 700ms | ~5× |
| Large internal Vercel app | 15s | 1.1s | ~14× |

No config needed — enabled by default since 16.1. The `experimental.turbopackFileSystemCacheForDev` / `...ForBuild` flags are no longer required (the dev cache is on; build cache is still experimental).

**Source:** [Turbopack FileSystem Caching config docs](https://nextjs.org/docs/app/api-reference/config/next-config-js/turbopackFileSystemCache)

### `next dev --inspect` (and `next start --inspect` in 16.2)

Attach the Node.js debugger to the dev server for profiling:

```bash
next dev --inspect
# → "Debugger listening on ws://127.0.0.1:9229/..."
# Open chrome://inspect in Chrome to attach
```

In Next.js 16.2 this also works with `next start` for production debugging.

### Next.js Bundle Analyzer (Experimental)

Inspect client and server bundles to find bloat:

```ts
// next.config.ts
const nextConfig: NextConfig = {
  experimental: {
    bundleAnalyzer: true,
  },
}
```

```bash
ANALYZE=true npm run build
# → Opens interactive HTML report
```

### Next.js DevTools MCP `get_routes` Tool

The Next.js DevTools MCP server (shipped in 16.0, expanded in 16.1) now includes a `get_routes` tool that returns the full list of routes in your application. Useful for AI agents exploring a codebase.

**Source:** [Next.js 16.1 release notes](https://nextjs.org/blog/next-16-1)

## Next.js 16.2 Turbopack Improvements (March 2026)

Turbopack (now the default Next.js 16 bundler) shipped a major update in 16.2. Most of these are zero-config and you get them for free when you upgrade. Devs upgrading from 16.1 reported **~80% faster dev startup** and **25–60% faster rendering**.

### Server Fast Refresh (default on, no flag)

Turbopack now only re-executes the module you changed — not the entire import chain. For most edits, the HMR cycle is effectively instant. Server Actions, Route Handlers, and proxy.ts do not yet participate (those run in the existing Node.js system); support is on the way.

### Tree Shaking of Dynamic Imports

Destructured dynamic imports are now tree-shaken the same way static imports are. Unused exports from a dynamic chunk are removed from the bundle:

```ts
// Before 16.2 — entire ./lib module was shipped even if only `cat` was used
const { cat } = await import('./lib')

// 16.2+ — unused exports are removed (same behavior as static import)
const { cat } = await import('./lib')
```

This is a free win for code-split apps with mixed exports.

### `postcss.config.ts` Support

Turbopack now supports `postcss.config.ts` alongside the existing `.js` and `.cjs` variants. No action needed if you’re already on `.ts`; if you hit a type error on PostCSS plugin config, the fix is renaming the file.

### Subresource Integrity (SRI)

Generate cryptographic hashes of your JS files at build time, so browsers can verify scripts haven’t been tampered with. Useful for hard CSP without forcing all pages to be dynamic:

```ts
// next.config.ts
const nextConfig: NextConfig = {
  experimental: {
    sri: {
      algorithm: 'sha256',  // sha256 or sha384
    },
  },
}
```

Browser sees a `<script integrity="sha384-...">` attribute and refuses to run anything whose hash doesn’t match. Works alongside Turbopack builds.

### Web Worker Origin (WASM in Workers)

Turbopack now supports running WASM-based libraries inside Web Workers without a separate bundler step. Useful for image processing, encryption, or any CPU-heavy client logic that benefits from a worker thread.

### Inline Loader Configuration (Import Attributes)

Per-import loader configuration via the new import attributes syntax — override the default loader for a single import without changing global config:

```ts
// Override the loader for one import — e.g. force CSS modules
import styles from './button.module.css' with { type: 'css' }
```

### Lightning CSS Configuration (Experimental)

Experimental support for configuring LightningCSS directly in `next.config.ts`. LightningCSS is faster than PostCSS for most transformations (autoprefixing, minification, nesting). If you’re hitting PostCSS perf issues, this is the path forward:

```ts
// next.config.ts
const nextConfig: NextConfig = {
  experimental: {
    lightningcss: {
      // target modern browsers, no legacy prefixes
      targets: { chrome: 100, firefox: 100, safari: 16 },
    },
  },
}
```

### Log Filtering — `ignoreIssue`

Suppress noisy Turbopack warnings on a per-rule basis. Useful when a known library triggers a deprecation warning that doesn’t apply to your codebase:

```ts
// next.config.ts
const nextConfig: NextConfig = {
  logging: {
    ignoreIssue: [
      '*next/font*local font not found*',  // pattern match
      'SWC-WARN-001',                       // or specific issue code
    ],
  },
}
```

`ignoreIssue` accepts glob patterns or specific issue codes. Use it sparingly — most warnings are real.

### ImageResponse Performance (React)

ImageResponse (used for OG image generation via `next/og`) is **2–20× faster** in 16.2 because React replaced the old `JSON.parse` reviver with a plain `JSON.parse()` plus a recursive walk. Free win for any app that generates dynamic OG images.

**Sources:**
- [Turbopack: What’s New in Next.js 16.2 (official)](https://nextjs.org/blog/next-16-2-turbopack)
- [Next.js 16.2 Turbopack API reference](https://nextjs.org/docs/app/api-reference/turbopack)
- [Roboto Studio: Next.js 16.2 for dummies (benchmarks)](https://roboto.studio/blog/nextjs-16-2-for-dummies)

### Vite (React SPA, non-Next)

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app && npm install
npm install -D tailwindcss @tailwindcss/vite
```

## TypeScript Configuration

### `tsconfig.json` — Strict Mode

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "target": "ES2025",
    "lib": ["ES2025", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*", "next-env.d.ts"],
  "exclude": ["node_modules"]
}
```

**Key strict flags:**
- `noImplicitAny` — every variable needs a type, no `any` by accident
- `strictNullChecks` — null/undefined handling must be explicit
- `noUnusedLocals` — catch dead code at compile time
- `noUncheckedIndexedAccess` — arrays/objects return `T | undefined` on index access
- `exactOptionalPropertyTypes` — `?:` vs `!:` semantics are strict

## Vite Configuration

### `vite.config.ts` (with Tailwind v4)

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'
import path from 'path'

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 3000,
  },
})
```

### React Compiler with Vite 8 + @vitejs/plugin-react v6

**`@vitejs/plugin-react` v6 (2026) replaced Babel with oxc.** The old `react({ babel: { plugins: ['babel-plugin-react-compiler'] } })` pattern no longer works in v6. Use the `compiler` option instead:

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'
import path from 'path'

export default defineConfig({
  plugins: [
    // React Compiler via oxc — the correct approach in plugin-react v6
    react({
      compiler: true,  // enables oxc-based React Compiler (babel approach no longer works)
    }),
    tailwindcss(),
  ],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
})
```

**`react({ compiler: true })` — The new way (plugin-react v6+):** Plugin-react v6 exposes a `compiler` option that uses `oxc-plugin-react-compiler` under the hood. This is the recommended approach for React 19 projects.

**For React 18 + plugin-react v5**, you can still use the Babel pass approach:
```ts
react({ babel: { plugins: ['babel-plugin-react-compiler'] } })
```

**ESLint for development-time checking** (recommended alongside the build plugin):
```bash
npm install -D eslint-plugin-react-compiler
```
```js
// eslint.config.mjs
import reactCompiler from 'eslint-plugin-react-compiler'

export default [
  {
    plugins: { 'react-compiler': reactCompiler },
    rules: {
      'react-compiler/react-compiler': 'warn',
    },
  },
]
```

**Sources:**
- [React Compiler + Vite 8 + plugin-react v6 guide](https://dev.to/recca0120/react-compiler-10-vite-8-the-right-way-to-install-after-vitejsplugin-react-v6-drops-babel-p0i)
- [vite-plugin-react discussion #1240 — oxc-plugin-react-compiler](https://github.com/vitejs/vite-plugin-react/discussions/1240)

## Next.js Configuration

### `next.config.ts`

```ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  // Enable Partial Prerendering (PPR) — stable in Next.js 16
  // cacheComponents: true,
  //   → PPR serves a static shell instantly; dynamic Suspense boundaries stream in
  //   → Required for App Shells (16.3+)
  //   → Recommended for all new Next.js 16 projects
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'images.unsplash.com' },
      { protocol: 'https', hostname: '*.amazonaws.com' },
    ],
  },
}
```

**Note:** In Next.js 15+, Server Actions are stable — no `experimental.serverActions` block needed.

## Environment Variables

### `.env.local` (local development)

```
# Database (if using Prisma or direct DB)
DATABASE_URL="postgresql://user:password@localhost:5432/mydb"

# API Base URL (frontend → backend)
NEXT_PUBLIC_API_URL="http://localhost:8080"

# Auth (NextAuth)
NEXTAUTH_URL="http://localhost:3000"
NEXTAUTH_SECRET="generate-with-openssl rand -base64 32"

# Third-party APIs
STRIPE_PUBLIC_KEY="pk_test_..."
STRIPE_SECRET_KEY="sk_test_..."
```

### `.env.production`

```
# No NEXT_PUBLIC_ prefix for server-only secrets
# All NEXT_PUBLIC_ vars are PUBLIC — anyone can read them
DATABASE_URL="postgresql://..."
NEXTAUTH_SECRET="..."
STRIPE_SECRET_KEY="sk_live_..."
```

**Rule:** Prefix only client-exposed vars with `NEXT_PUBLIC_`. Server-only secrets never get the prefix.

## shadcn/ui Initialization

```bash
# Initialize shadcn/ui
npx shadcn@latest init

# Default settings:
# - Style: Default
# - Base color: Slate
# - CSS variables: Yes
# - Custom prefix: No
# - tailwind.config.ts: Yes
# - Components: @/components/ui
# - Utils: @/lib/utils
# - CSS: src/app/globals.css

# Add components
npx shadcn@latest add button card form input label toast
```

### `globals.css` (Tailwind v4 + shadcn/ui)

```css
@import "tailwindcss";

@theme inline {
  --color-background: hsl(var(--background));
  --color-foreground: hsl(var(--foreground));
  --color-card: hsl(var(--card));
  --color-card-foreground: hsl(var(--card-foreground));
  --color-primary: hsl(var(--primary));
  --color-primary-foreground: hsl(var(--primary-foreground));
  --color-secondary: hsl(var(--secondary));
  --color-muted: hsl(var(--muted));
  --color-accent: hsl(var(--accent));
  --color-destructive: hsl(var(--destructive));
  --color-border: hsl(var(--border));
  --color-ring: hsl(var(--ring));
  --radius: calc(var(--radius) * 1px);
}

body {
  background-color: var(--color-background);
  color: var(--color-foreground);
}
```

## Package Manager

Prefer `pnpm` for monorepos and workspaces:

```bash
npm install -g pnpm
pnpm install
```

pnpm's `node_modules` layout prevents phantom dependency bugs and saves disk space.

## TypeScript Type Generation

### Generate types from API responses (Zod)

```ts
import { z } from 'zod'

const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  email: z.string().email(),
  role: z.enum(['admin', 'user', 'moderator']),
  createdAt: z.coerce.date(),
})

type User = z.infer<typeof UserSchema>
// → { id: string; name: string; email: string; role: 'admin' | 'user' | 'moderator'; createdAt: Date }
```

**Never define types manually when Zod can infer them.** One source of truth.

### Generate types from backend OpenAPI spec

```bash
npm install -D @rtk-query/codegen-openapi
```

## Linting and Formatting

**Important:** In Next.js 16, `next lint` is **removed**. Use Biome or ESLint directly.

### Option 1: Biome 2.x (Recommended — Fastest)

[Biome](https://biomejs.dev) is a fast linter and formatter for JavaScript/TypeScript, written in Rust. It's 10–100x faster than ESLint for large codebases. **Biome 2.x has breaking changes from 1.x** — config format and some rule names changed.

```bash
npm install -D @biomejs/biome
npx biome init  # Creates biome.json
```

```json
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/2.5.0/schema.json",
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true
  },
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "suspicious": {
        "noExplicitAny": "warn"
      },
      "correctness": {
        "useExhaustiveDependencies": "warn"
      }
    }
  },
  "formatter": {
    "enabled": true,
    "formatWithErrors": false,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  },
  "javascript": {
    "jsxAttributeApi": "react"
  }
}
```

**Run Biome:**
```bash
npx biome check .        # Lint + format check
npx biome check --write  # Lint + auto-fix
npx biome format .       # Format only
```

**Concise reporter (Biome 2.5+):** For compact single-line diagnostics:
```bash
npx biome check --reporter=concise
# Output:
# ! index.ts:2:10 lint/correctness/noUnusedImports: Several of these imports are unused.
# × index.ts:8:5 lint/suspicious/noImplicitAnyLet: This variable implicitly has the 'any' type.
```

Add to `package.json` scripts:
```json
{
  "scripts": {
    "lint": "biome check --write",
    "format": "biome format --write"
  }
}
```

**Biome 2.x migration from 1.x:**
- Config still uses `biome.json` but schema URL must update to `2.x`
- `vcs` top-level key is new in v2 — enables git integration for ignored files
- Most rules unchanged; check [Biome v2 migration guide](https://biomejs.dev/migration-guide/) for details

**`biome migrate` — Run After Every Upgrade:**

Biome requires running a migration script after every version upgrade to update your config and suppressions for breaking changes:

```bash
# After upgrading Biome to any new version
npx biome migrate --write

# What it does:
# 1. Updates biome.json schema URL to the new version
# 2. Rewrites rule suppressions that changed between versions
# 3. Flags config keys that need manual review
```

**Always run `biome migrate --write` after `npm install @biomejs/biome@latest`** — skipping it means Biome may error on config keys that changed between versions. Add it to your upgrade workflow:

```bash
npm install -D @biomejs/biome@latest
npx biome migrate --write
npx biome check --write  # Verify everything still passes
```


### Option 2: ESLint (Flat Config — Next.js 16)

ESLint's flat config (`eslint.config.mjs`) is the modern approach and works with Next.js 16:

```bash
npm install -D eslint @eslint/js typescript-eslint eslint-plugin-react-hooks
```

```js
// eslint.config.mjs
import js from '@eslint/js'
import tseslint from 'typescript-eslint'
import reactHooks from 'eslint-plugin-react-hooks'

export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.recommended,
  {
    files: ['**/*.ts', '**/*.tsx'],
    plugins: {
      'react-hooks': reactHooks,
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/no-explicit-any': 'warn',
    },
  },
  {
    ignores: ['.next/**', 'node_modules/**', 'dist/**'],
  }
)
```

**Run ESLint directly:**
```bash
npx eslint .
```

Add to `package.json` scripts:
```json
{
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix"
  }
}
```

**Why `next lint` was removed:** It wrapped ESLint with a simplified CLI that hid options, made upgrades harder to track, and added maintenance burden. Using ESLint or Biome directly gives full control and faster iteration.

### Formatting with Prettier (if using ESLint for linting)

```bash
npm install -D prettier
```

```json
// .prettierrc
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "es5",
  "printWidth": 100
}
```

```json
// package.json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

## Key Files Created

```
my-app/
├── src/
│   ├── app/
│   │   ├── layout.tsx        # Root layout (server component)
│   │   ├── page.tsx         # Home page
│   │   ├── globals.css       # Tailwind imports
│   │   └── loading.tsx      # Suspense fallback
│   ├── components/
│   │   └── ui/              # shadcn/ui components
│   └── lib/
│       ├── utils.ts         # cn() helper from shadcn
│       └── api.ts           # React Query setup
├── next.config.ts
├── tsconfig.json
└── tailwind.config.ts       # or tailwind.config.js
```

## Common Issues

- **`moduleResolution: Bundler`** — required for Next.js 16, not `"Node"` or `"Node16"`
- **Tailwind v4** — use `@tailwindcss/vite` plugin for Vite, or the built-in Next.js Tailwind support
- **ESM vs CJS** — prefer `"module": "ESNext"` + `"type": "module"` in package.json
- **`experimental.serverActions` in next.config.ts** — remove it in Next.js 15+, Server Actions are stable
- **`next lint` in Next.js 16** — removed; use Biome (`npx biome check`) or ESLint (`npx eslint .`) directly instead
- **`target: "ES2022"`** — update to `"ES2025"` for Next.js 16 / React 19 compatibility
- **React Compiler + Vite 8 + plugin-react v6** — `react({ babel: { plugins: [...] } })` no longer works; use `react({ compiler: true })` instead (oxc-based)
