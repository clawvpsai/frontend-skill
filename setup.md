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

### 1. AGENTS.md in `create-next-app` (Default: Yes) + Auto-Update (16.3+)

`npx create-next-app@latest` now scaffolds an `AGENTS.md` file at the project root. This file instructs AI coding agents (Claude Code, Cursor, etc.) to read the bundled Next.js documentation from `node_modules/next/dist/docs/` *before* writing any code — giving agents version-matched, local documentation instead of fetching external data.

Vercel's research found that bundled docs achieved a **100% pass rate** on Next.js evals vs. **79%** for skill-based approaches (where agents must decide when to search for docs).

```bash
# Created automatically by create-next-app — do not delete
cat AGENTS.md
# → "Read the docs bundled at node_modules/next/dist/docs/ before writing any code"
```

**16.3 update:** `next dev` now writes and updates the managed `AGENTS.md` block automatically when it detects an AI coding agent in the environment (`CLAUDE_CODE`, `CURSOR_TRACE_ID`, `GITHUB_COPILOT_CHAT`, etc.). The exact text it inserts is delimited by `<!-- BEGIN:nextjs-agent-rules -->` / `<!-- END:nextjs-agent-rules -->` markers; anything outside the markers is preserved verbatim. **Opt out** with `agentRules: false` in `next.config.ts`. For 16.1 or earlier (or to opt in manually without running `next dev`), run `npx @next/codemod@canary agents-md`. Full details in `patterns.md` → Next.js 16.3 AI Improvements.

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

### 4. `agent-browser` — Browser CLI for Agents (formerly `next-browser`, merged in 16.3)

As of Next.js 16.3 (June 26, 2026), the experimental [`@vercel/next-browser`](https://github.com/vercel-labs/next-browser) CLI from 16.2 has **merged into the general-purpose [`agent-browser`](https://github.com/vercel-labs/agent-browser) CLI**. Everything `next-browser` did is now in `agent-browser`, and it works beyond Next.js too (any web app). **`agent-browser` 0.27+** adds React DevTools introspection on top of the existing DOM, console, network, and Web Vitals access — the equivalent commands are renamed from `next-browser *` to `agent-browser *`.

Install or upgrade with `npm install -g agent-browser@^0.27` (0.31.1 is current; the `next-dev-loop` skill now requires 0.31.1+, see `patterns.md` → Next.js 16.3 AI Improvements). React commands require `--enable react-devtools` at launch.

```bash
# Install globally
npm install -g agent-browser@^0.27

# Or as a skills.sh skill
npx skills add vercel-labs/agent-browser
```

**Available commands (as of June 2026, agent-browser 0.31.1):**

| Command | Purpose |
|---|---|
| `agent-browser tree` | Inspect DOM/component tree with selectors + source locations |
| `agent-browser react tree` | List the React component tree with fiber IDs |
| `agent-browser react inspect <fiberId>` | Inspect a single React component (props, hooks, state, source location) |
| `agent-browser react renders start` / `stop` | Profile re-renders over a time window |
| `agent-browser react suspense --only-dynamic --json` | See what's holding a render — machine-readable JSON for agents |
| `agent-browser network` | Track requests since navigation, including Server Actions |
| `agent-browser logs` | Retrieve build + runtime issues from dev server |
| `agent-browser screenshot` | Capture current viewport (or loading filmstrip) |
| `agent-browser errors` | Surface dev overlay errors with `Error.cause` chains |
| `agent-browser mcp` | Start an MCP stdio server (for Claude Desktop, Cursor, etc.) |

Each command is a one-shot request against a persistent browser session, so agents can query the app repeatedly without managing browser state. **Note:** `agent-browser` is still under active development — APIs may change.

The previous `next-browser` repo is now archived/merged; **do not install `next-browser` for new projects in 16.3+** — it duplicates `agent-browser` and won't get React DevTools introspection. The `next-dev-loop` Skill (`npx skills add vercel/next.js --skill next-dev-loop`) wraps `agent-browser` for agents and is the recommended way to drive a Next.js dev loop from a coding agent.

### 5. 16.3 Update — `AGENTS.md` Auto-Update + Knowledge-Skills Retirement

Two behavioral changes land in 16.3 that supersede the 16.2 AGENTS.md behavior:

**(a) `next dev` writes and updates the managed `AGENTS.md` block automatically.** The dev server detects AI coding agents in the environment (`CLAUDE_CODE`, `CURSOR_TRACE_ID`, `GITHUB_COPILOT_CHAT`, etc.) and inserts (or refreshes) the marker block on its own — no manual codemod needed. The exact text:

```md
<!-- BEGIN:nextjs-agent-rules -->

# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` before writing any code. Heed deprecation notices.

<!-- END:nextjs-agent-rules -->
```

The block is only written when (a) an AI coding agent is detected AND (b) the markers aren't already there. Anything outside the markers is preserved verbatim. **Opt out** with `agentRules: false` in `next.config.ts`. For 16.1 or earlier (or to opt in manually), run `npx @next/codemod@canary agents-md`.

**(b) Older Vercel knowledge Skills are retired.** The earlier knowledge Skills (App Router conventions, caching APIs) distributed via `skills.sh` are **retired in 16.3** because the bundled docs (now reachable through the managed AGENTS.md block) make them redundant. **Run `npx skills update` to remove them from your installed set.** The new recommended Skills are the three first-party Skills from `vercel/next.js/tree/canary/skills`: `next-dev-loop`, `next-cache-components-adoption`, and `next-cache-components-optimizer`. Full breakdown in `patterns.md` → Next.js 16.3 AI Improvements.

**Sources:**
- [Next.js 16.2: AI Improvements (official blog, March 2026)](https://nextjs.org/blog/next-16-2-ai)
- [Next.js 16.3: AI Improvements (official blog, June 26, 2026 — merged `next-browser` into `agent-browser`, AGENTS.md auto-update, knowledge-Skills retirement)](https://nextjs.org/blog/next-16-3-ai-improvements)
- [Next.js AI Coding Agents guide](https://nextjs.org/docs/app/guides/ai-agents)
- [`agent-browser` repo (successor to `next-browser`)](https://github.com/vercel-labs/agent-browser)
- [`@vercel/next-browser` repo (archived/merged into `agent-browser`)](https://github.com/vercel-labs/next-browser)
- [`agent-browser` on npm (current: 0.31.1)](https://www.npmjs.com/package/agent-browser)
- [Vercel research: AGENTS.md outperforms skills (100% vs 79%)](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals)

## Next.js 16.1 Highlights (December 2025)## Next.js 16.1 Highlights (December 2025)

If you're starting a new project today, you're getting these by default. Worth knowing what they do:

### Turbopack File System Caching (Stable, Default On)

Turbopack now persists compiler artifacts to disk under `.next/`. Restarting `next dev` is dramatically faster — the first route compile time goes from seconds to ~hundreds of milliseconds.

| Project | Cold | Cached | Speedup |
|---|---|---|---|
| react.dev | 3.7s | 380ms | ~10× |
| nextjs.org | 3.5s | 700ms | ~5× |
| Large internal Vercel app | 15s | 1.1s | ~14× |

No config needed — enabled by default since 16.1. The `experimental.turbopackFileSystemCacheForDev` flag is no longer required (the dev cache is on). The `experimental.turbopackFileSystemCacheForBuild` flag is **on by default in canary/preview releases** when certain env vars are detected (16.3.0-preview.3, [#94616](https://github.com/vercel/next.js/pull/94616)) — stabilization will move this into build-adaptors as a more generic solution.

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

### VoidZero → Cloudflare (June 4, 2026)

The company behind Vite, Vitest, Rolldown, Oxc, and Vite+ (VoidZero, founded by Evan You) **joined Cloudflare on June 4, 2026**. The relevant points for Vite/Vitest users:

- **Vite, Vitest, Rolldown, Oxc, and Vite+ remain MIT-licensed and open source** — license, governance, and roadmap are unchanged
- **Evan You and the VoidZero team continue to lead all projects** — same maintainers, same contribution model
- **Cloudflare commits engineering and resources to the projects**, not redirects them away
- **Vite stays vendor-agnostic** — apps built with Vite run anywhere and continue to do so
- **Short term:** nothing changes for Vite/Vitest users
- **Long term:** Cloudflare's Vite plugin and the Cloudflare CLI will deepen Vite integration; Vite will get new "provider-agnostic primitives for full-stack apps and agents" that work on any platform; the Void platform will be open-sourced

This is the same playbook Cloudflare used for the Astro acquisition earlier in 2026 — open source remains, team keeps shipping the same roadmap, and Cloudflare is one more major contributor (not a hijacker). The relevant implication for this skill: Vite 8 / Vitest 4 / Vitest 5 betas continue to be safe picks. Cloudflare-published improvements (deeper Workers integration, better Environment API primitives) will benefit non-Cloudflare deployments too.

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

### React Compiler on Turbopack — Experimental Rust Port (16.3.0-canary.52, June 16, 2026)

Next.js 16.3 canary adds **experimental Turbopack React Compiler** support ([#94573](https://github.com/vercel/next.js/pull/94573)). It runs the compiler directly on Turbopack's swc AST with no `gen` + reparse, applied to client code and SSR (not RSC). Opt-in:

```ts
// next.config.ts
const nextConfig: NextConfig = {
  reactCompiler: true,           // required — error if missing
  experimental: {
    rustReactCompiler: true,     // opt in to the Rust port on Turbopack
  },
}
```

The Rust compiler also **detects the installed React version** (16.3.0-canary.53, [#94836](https://github.com/vercel/next.js/pull/94836)) — it now builds for React 18 apps, not just React 19 (which it previously assumed). React 17 is supported by the compiler but not by Next.js 16.

**Sources:**
- [PR #94573 — Turbopack: add experimental React compiler support (canary.52)](https://github.com/vercel/next.js/pull/94573)
- [PR #94836 — rust react compiler: detect and build for react 18 (canary.53)](https://github.com/vercel/next.js/pull/94836)
- [vite-plugin-react discussion #1240 — oxc-plugin-react-compiler](https://github.com/vitejs/vite-plugin-react/discussions/1240)

### Turbopack "Edge Runtime" → "Self-Contained Runtime" Rename (16.3.0-canary.61, [#94726](https://github.com/vercel/next.js/pull/94726), June 22, 2026)

The Turbopack ecmascript-runtime package's `edge` chunk-loading variant has been renamed to `self-contained`. The runtime code's name reflects what it actually does: it performs no runtime chunk loading, registers chunks only via `globalThis`/`self` (no DOM), and is used for **both** the Edge execution environment and single-chunk (service-worker) bundles, where everything is inlined into one file.

**What changed:**

- `turbopack/crates/turbopack-ecmascript-runtime/js/src/browser/runtime/edge/` → `…/runtime/self-contained/`
- File renames: `dev-backend-edge.ts` → `dev-backend-self-contained.ts`, `runtime-backend-edge.ts` → `runtime-backend-self-contained.ts`
- `package.json` scripts: `check:browser-runtime-edge` → `check:browser-runtime-self-contained`
- `browser_runtime.rs` matches now `(ChunkLoading::Edge, RuntimeType::{Development,Production})` and pushes the renamed paths

**Impact for Next.js users:** This is **not a functionality change** for application code. The rename is internal to Turbopack's runtime code generation. If you have any custom code that reads from `turbopack/crates/turbopack-ecmascript-runtime/js/src/browser/runtime/edge/` (extremely unlikely for a Next.js app), update the paths. If you see documentation or error messages still referencing the old `edge` runtime path in canary.61+, that's a stale reference — the canonical name is now `self-contained`.

### Turbopack Service Worker Support (16.3.0-canary.68, [#94920](https://github.com/vercel/next.js/pull/94920) + [#94921](https://github.com/vercel/next.js/pull/94921) + [#94922](https://github.com/vercel/next.js/pull/94922), June 25, 2026)

First-class Turbopack support for compiling and serving **service workers** landed in 16.3.0-canary.68 as a three-PR stack that mirrors the Webpack-equivalent behavior (which has been there since Next.js 12+). After this change, Turbopack's analyzer discovers `ServiceWorkerEntryModule`s, the Turbopack runtime emits them as **single-chunk self-contained bundles** (using the runtime renamed above), and `next-api` serves the compiled worker on the route you register it on (typically `/sw.js` or `/worker.js`).

**What ships:**

1. **[#94920](https://github.com/vercel/next.js/pull/94920)** — `ServiceWorkerChunkingContextOptions` added to `turbopack/crates/turbopack-next/next-core` so Turbopack knows how to bundle workers (no shared chunks, no DOM-dependent runtime — exactly the "self-contained" runtime shape the previous rename enabled).
2. **[#94921](https://github.com/vercel/next.js/pull/94921)** — `ServiceWorkerEntryModule` plus a new `service_worker_chunk_filename` config knob. The analyzer inserts one of these per registered worker, telling the runtime "compile this file as a self-contained single-chunk worker."
3. **[#94922](https://github.com/vercel/next.js/pull/94922)** — `next-api` discovers the workers via the module graph and serves the compiled output at the route you registered (e.g. `navigator.serviceWorker.register('/sw.js')` will now hit a Turbopack-compiled worker instead of falling back to Webpack).

**User-facing impact:**

- If you ship a PWA, **migrate from `next dev --webpack` to Turbopack now**. Previously, service worker registration on a Turbopack dev/build would silently fall back to a static file or fail to serve a compiled worker — both are easy-to-miss footguns that broke offline behavior in dev and produced stale prod workers.
- **`navigator.serviceWorker.register('/sw.js')`** continues to work — you do **not** need to change your registration code. The compiled output path is the same. Internally, Turbopack now produces a single-file bundle that loads with `globalThis`/`self` only (matching the self-contained runtime shape) so the worker is fully self-contained and registers cleanly without depending on a chunk loader that the worker context can't access.
- **No config needed.** The bundling and serving are automatic for any file you register as a service worker, via the same module-graph discovery path Webpack has used for years.
- **e2e tests still being added** ([#94924](https://github.com/vercel/next.js/pull/94924), open at canary.68) — expect rough edges if you're an early adopter. If your existing PWA test suite was passing on Turbopack canary ≤ canary.67, it should keep passing (the old code path was a no-op or fall-through, so any "passing" result was likely a stale-cache hit).

**Files that change in Turbopack source if you ever inspect them:**

- `turbopack/crates/turbopack-ecmascript-runtime/js/src/browser/runtime/self-contained/` — the runtime that now backs workers
- `turbopack/crates/turbopack-next/next-core/src/next_shared/resolve.rs` — the analyzer rule that inserts `ServiceWorkerEntryModule`
- `packages/next/src/server/lib/router-utils/filesystem.ts` — the route that serves the compiled worker

**Sources:**
- [PR #94920 — `[turbopack]` Create `ServiceWorkerChunkingContextOptions` in `next-core`](https://github.com/vercel/next.js/pull/94920)
- [PR #94921 — `[turbopack]` Create `ServiceWorkerEntryModule` and `service_worker_chunk_filename`](https://github.com/vercel/next.js/pull/94921)
- [PR #94922 — `[turbopack]` Discover `ServiceWorkerEntryModule`s in `next-api` and compile + serve those service workers](https://github.com/vercel/next.js/pull/94922)
- [PR #94924 — `[turbopack]` Add e2e tests for service workers](https://github.com/vercel/next.js/pull/94924) (still open at canary.68)
- [Next.js docs — `serviceWorkerRegistration` in PWA guide](https://nextjs.org/docs/app/guides/progressive-web-apps)

### Cache Components Adoption — `cache-components-instant-false` Codemod (16.3, [#94941](https://github.com/vercel/next.js/pull/94941))

Enabling `cacheComponents: true` fails the build immediately for any route that reads request-time data outside `<Suspense>`. The new `cache-components-instant-false` codemod in `@next/codemod` (registered at `version: '16.3.0'`) blanket-inserts `export const instant = false` (with a `// TODO: Cache Components adoption` comment) into every `app/**/{page,layout,default}.{js,jsx,ts,tsx}` that doesn't already export `instant`. Skips Client Components (`"use client"`), route handlers, and files with existing `instant` exports (including aliased `export { x as instant }`). Idempotent — safe to re-run.

```bash
# Run from project root, point at your app directory
npx @next/codemod@canary cache-components-instant-false ./app
```

For agents working on a larger app, Vercel ships a matching [`next-cache-components-adoption`](https://github.com/vercel/next.js/tree/canary/skills/next-cache-components-adoption) skill that drives the full two-milestone adoption (green build → remove `instant = false` top-down) and walks the dev-overlay validation insights:

```bash
npx skills add https://github.com/vercel/next.js/tree/canary/skills/next-cache-components-adoption
```

See `patterns.md` → "Cache Components Adoption — The `instant = false` Opt-Out + Adoption Skill" for the full breakdown: highest-wins resolution, why `instant = false` doesn't clear sync-IO errors (`new Date()`, `Math.random()`, `crypto.randomUUID()`), and why you remove opt-outs top-down (root layout first, then descend).

### Vite 8.1 (June 23, 2026) — What's New

Vite 8.1.0 is a minor release with several notable additions. **No breaking changes for plugin authors or app code** — everything below is opt-in.

| Feature | Why it matters | Config / API |
|---|---|---|
| **WASM ESM Integration** (`.wasm` direct imports) | Import a `.wasm` file as an ES module — no `?init`, no `WebAssembly.instantiate` boilerplate. Also works in SSR now. | `import wasm from './module.wasm'` |
| **`server.hmr` → `server.ws` rename** | HMR is implemented over a WebSocket; the `ws` namespace reflects the actual transport. The old `server.hmr` keys still work (deprecation warning). | `server: { ws: { host, port, protocol } }` |
| **`import.meta.glob` `caseSensitive` option** | Glob matching on case-sensitive filesystems (Linux prod, Docker) can now be forced even on case-insensitive dev hosts (macOS, Windows). | `import.meta.glob('./**/*.svg', { caseSensitive: true })` |
| **`html.additionalAssetSources` option** | Declare extra roots (e.g. a `public-cdn/` folder, or a remote manifest) whose files become addressable as static assets. | `html: { additionalAssetSources: ['public-cdn'] }` |
| **Chunk importmap** | Vite now emits a build-time import map so non-bundled script tags can resolve hashed chunk URLs — great for module federation and micro-frontend setups. | (automatic, in build output) |
| **Lightning CSS as a CSS plugin** | First-class Lightning CSS support — `transform`, `minify`, and `define` are wired in via a plugin dependency. Use it instead of PostCSS for new projects. | `import lightningcss from 'vite-plugin-lightningcss'` |
| **`envFile` deprecation warning** | The old `envFile: false` is deprecated; use `envDir` + explicit `.env` exclusion via `loadEnv('mode', root, '')`. | warning at startup |
| **Vite Task (zero-config build cache)** | Optional Rust-based build cache. Speeds up cold builds in CI and large monorepos. | opt-in via `vite task enable` |
| **Extended `server.fs.deny` list** | Common leaked files (`.env`, `.env.*`, `*.pem`, `*.key`) are now denied by default in dev. Override per-entry. | `server: { fs: { deny: ['custom/secret'] } }` |
| **Rolldown 1.1.2** | Faster bundling, smaller binaries. Drop-in for Vite 8. | (automatic) |

**Migration impact:** None. Vite 8.1.0 is fully backward-compatible with Vite 8.0.x. The `server.hmr` keys still work (deprecation warning) and the new `server.ws` keys are recommended for new code.

**The WASM ESM integration in practice:**

```ts
// Before Vite 8.1 (still works)
import init, { add } from './add.wasm?init'
await init()
console.log(add(1, 2))

// Vite 8.1+ — direct ESM import (works in dev, build, and SSR)
import * as addWasm from './add.wasm'
const { add } = await addWasm
console.log(add(1, 2))
```

**The `server.ws` rename in practice:**

```ts
// Vite 8.0
export default defineConfig({
  server: {
    hmr: { host: 'myhost.local', port: 24678, protocol: 'wss' },
  },
})

// Vite 8.1+ (recommended)
export default defineConfig({
  server: {
    ws: { host: 'myhost.local', port: 24678, protocol: 'wss' },
  },
})
// Vite logs: "server.hmr.* is deprecated, use server.ws.* instead"
```

**Sources:**
- [Vite 8.1.0 CHANGELOG (GitHub)](https://github.com/vitejs/vite/blob/v8.1.0/packages/vite/CHANGELOG.md)
- [Vite 8.0 announcement — foundational features that 8.1 builds on](https://vite.dev/blog/announcing-vite8)
- [Vite 8.1.0 release on GitHub](https://github.com/vitejs/vite/releases/tag/v8.1.0)

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
