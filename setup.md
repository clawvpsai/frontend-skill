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

## Next.js 16.3 Turbopack Preview (June 29, 2026) — Andrew Imm

The third post in the [16.3 Preview series](https://nextjs.org/blog/next-16-3-turbopack) — "Turbopack: What's New in Next.js 16.3" — lands seven major Turbopack improvements at once. The performance impact of each is captured in **`performance.md`**: see [Dev Memory Eviction](#dev-memory-eviction-experimentalturbopackmemoryeviction-163-preview-june-29-2026-defaults-on) (16.3 Preview, June 29, 2026 — defaults on), [Persistent File-System Cache for Builds](#persistent-file-system-cache-for-builds-experimentalturbopackfilesystemcacheforbuild-163-preview-june-29-2026-was-already-opt-in-now-ga), [Experimental Rust React Compiler](#experimental-rust-react-compiler-experimentalturbopackrustreactcompiler-163-preview-june-29-2026-added-docs-page), [`import.meta.glob` API on Turbopack](#importmetaglob-api-on-turbopack-163-preview-june-29-2026-vite-compatible), [HMR Cold-Start Win](#hmr-cold-start-win-single-subscription-chunk-tracking-163-preview-june-29-2026), [Smaller Runtime Size](#smaller-runtime-size-lazy-wasm-lazy-workers-lazy-async-modules-163-preview-june-29-2026), [Local PostCSS Configuration](#local-postcss-configuration-experimentalturbopacklocalpostcssconfig-163-preview-june-29-2026), and [Turbopack Compatibility and Reliability](#turbopack-compatibility-and-reliability-163-preview-june-29-2026-final-section-of-the-163-preview-turbopack-post) for the long-form patterns. Quick-reference:

```ts
// next.config.ts — 16.3 Turbopack defaults are now memory-safe, cacheable, and faster:
const nextConfig: NextConfig = {
  reactCompiler: true,                              // stable since 16.0
  experimental: {
    // Memory: evict inactive cache entries to disk (default on in 16.3, requires dev FS cache)
    turbopackFileSystemCacheForDev: true,           // default true since 16.1
    turbopackMemoryEviction: 'full',                // default in 16.3 — 'full' | 'single' | false
    // Build: persist .next/ cache across `next build` runs (CI-friendly)
    turbopackFileSystemCacheForBuild: true,         // default true in 16.3 canary/preview
    // Compiler: native Rust port (Babel fallback) — ~20–50% build perf on large React apps
    turbopackRustReactCompiler: true,               // requires reactCompiler: true
    // Monorepos: per-package PostCSS resolution
    turbopackLocalPostcssConfig: true,
  },
}
```

`import.meta.glob` is a Turbopack-only feature (does not work with `next build --webpack`):

```ts
// app/blog/page.tsx — type-safe blog index via Vite-compatible glob API
// Returns Record<string, () => Promise<MDXModule>> — async loader per match
const posts = import.meta.glob<false, './posts/*.mdx'>('./posts/*.mdx')
```

See `performance.md` for the full sections (benchmarks, caveats, monorepo postcss walkthrough, and config compatibility details for `import.meta.url` on Windows / `worker_threads` / `module-sync`).

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
- [PR #95280 — Add docs for `experimental.turbopackRustReactCompiler` (canary.71, June 29, 2026)](https://github.com/vercel/next.js/pull/95280) — adds `docs/01-app/03-api-reference/05-config/01-next-config-js/turbopackRustReactCompiler.mdx` (52 lines)
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

First-class Turbopack support for compiling and serving **service workers** landed in 16.3.0-canary.68 as a three-PR stack that mirrors the Webpack-equivalent behavior (which has been there since Next.js 12+). After this change, Turbopack's analyzer discovers `ServiceWorkerEntryModule`s, the Turbopack runtime emits them as **single-chunk self-contained bundles** (using the runtime renamed above), and `next-api` serves the compiled worker on the route you register it on (typically `/sw.js` or `/worker.js`). **A fourth PR (#94923) landed in 16.3.0-canary.71 to complete the feature** — the analyzer-side transformation that recognizes `navigator.serviceWorker.register()` calls.

**What ships:**

1. **[#94920](https://github.com/vercel/next.js/pull/94920)** — `ServiceWorkerChunkingContextOptions` added to `turbopack/crates/turbopack-next/next-core` so Turbopack knows how to bundle workers (no shared chunks, no DOM-dependent runtime — exactly the "self-contained" runtime shape the previous rename enabled).
2. **[#94921](https://github.com/vercel/next.js/pull/94921)** — `ServiceWorkerEntryModule` plus a new `service_worker_chunk_filename` config knob. The analyzer inserts one of these per registered worker, telling the runtime "compile this file as a self-contained single-chunk worker."
3. **[#94922](https://github.com/vercel/next.js/pull/94922)** — `next-api` discovers the workers via the module graph and serves the compiled output at the route you registered (e.g. `navigator.serviceWorker.register('/sw.js')` will now hit a Turbopack-compiled worker instead of falling back to Webpack).
4. **[#94923](https://github.com/vercel/next.js/pull/94923)** *(16.3.0-canary.71, June 29, 2026, by sampoder)* — the analyzer-side transformation that **"turns on" the service worker feature**. The Turbopack analyzer now looks for code like `await navigator.serviceWorker.register(new URL('../lib/service-workers/1.js', import.meta.url), ...)` and transforms it into registering the URL that Next.js will serve based on scope. URL scheme: `scope: '/' → 'sw.js'`, `scope: '/offline/mode' → 'sw-offline-mode.js'`, etc. Also inserts the `ServiceWorkerEntryModule`s so they can be discovered by `next-api` (see #94922). 5 files in `turbopack/crates/turbopack-ecmascript/src/{analyzer/well_known/{kinds.rs,mod.rs},code_gen.rs,references/{mod.rs,service_worker.rs}}`, +309/-5. **Before this PR, even with #94920/#94921/#94922 in place, `navigator.serviceWorker.register()` calls in user code weren't recognized by Turbopack's analyzer** — the analyzer would pass them through unchanged, the worker wouldn't appear in the module graph, and the user-facing behavior would still fall back to static files or stale workers.

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
- [PR #94923 — `[turbopack]` Discover service workers in the Turbopack analyzer (canary.71, June 29, 2026)](https://github.com/vercel/next.js/pull/94923) — the PR that "turns on" the feature
- [PR #94924 — `[turbopack]` Add e2e tests for service workers](https://github.com/vercel/next.js/pull/94924) (open since canary.68, now landed)
- [Next.js docs — `serviceWorkerRegistration` in PWA guide](https://nextjs.org/docs/app/guides/progressive-web-apps)
### Turbopack Subpath Imports (`#/*` Wildcards) — 16.3.0-canary.75+ ([PR #94461](https://github.com/vercel/next.js/pull/94461), merged 2026-07-02T00:11:23Z)

Turbopack's resolver had an over-eager early-rejection check that blocked **all** specifiers starting with `#/`, preventing the widely-used TypeScript `imports` field pattern `"#/*": "./src/*"` from working. The fix in PR #94461 removes the `specifier.starts_with("#/")` guard from the early-rejection condition in `resolve_package_internal_with_imports_field` so the specifier can reach the `imports`-field resolver (where `AliasMap` already handles wildcard pattern matching correctly).

**Before canary.75 (canary.72–canary.74 and earlier)** — a project with this `package.json`:

```json
{
  "imports": {
    "#/*": "./src/*"
  }
}
```

And this import:

```ts
// src/main.ts
import greeting from '#/greeting.js'
// resolves to src/greeting.js
```

Would fail under Turbopack with the resolver returning "unresolvable" before even attempting the `imports`-field lookup. The check rejected `#/greeting` purely because it started with `#/`, but `#/something` is a perfectly valid specifier that should reach the wildcard pattern matcher. Webpack (`enhanced-resolve`) and Node.js both already supported this pattern. **After canary.75** — `import greeting from '#/greeting.js'` resolves correctly through `AliasMap`'s wildcard matching.

**Concretely:**

```ts
// package.json
{
  "imports": {
    "#/*": "./src/*"
  }
}

// src/main.ts
import greeting from '#/greeting.js'   // → src/greeting.js
```

**What still gets rejected early:** bare `#` (not a valid subpath), specifiers ending in `/` (directory traversal without a path segment). These are the only true early-rejection cases.

**Who needs to upgrade:** anyone using the `"#/*": "./src/*"` subpath import pattern (TypeScript projects in particular) on Turbopack — this was a silent footgun where the project worked on `next dev --webpack` and broke on Turbopack without an obvious cause. The new test fixture in `turbopack/crates/turbopack-tests/tests/execution/turbopack/resolving/subpath-imports-slash` covers the pattern end-to-end.

**Closes:** [#94290](https://github.com/vercel/next.js/issues/94290) — original issue reporting the broken pattern under Turbopack.

**Source:** [PR #94461 — `fix(turbopack): allow "#/" prefixed subpath import specifiers`](https://github.com/vercel/next.js/pull/94461) · Commit `2aa1e457ab` (merged 2026-07-02T00:11:24Z) · Files: `turbopack/crates/turbopack-resolve/src/resolve.rs` (remove one `||` clause in `resolve_package_internal_with_imports_field`) + new test fixture `subpath-imports-slash/`.

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

### `@next/codemod upgrade --yes` for Agents / CI (16.3.0-canary.72+, [PR #95312](https://github.com/vercel/next.js/pull/95312) by aurorascharff, merged 2026-06-30T20:03:37Z)

`pnpm dlx @next/codemod@canary upgrade preview` was hanging forever when invoked by AI coding agents — the recommended-codemods multiselect blocks waiting on keyboard input. Same hazard for any CI that pipes the command. Agents were choosing the manual non-interactive path, which defeats the codemod entirely. **canary.72 fixes this** with two changes:

1. **New `-y, --yes` flag** on `next-codemod upgrade` — skips every interactive prompt and accepts its existing default.
2. **Auto-detection** of `!process.stdin.isTTY` — agents and CI get non-interactive mode without needing to know the flag exists.

**canary.73 (PR #95314) completes the agent loop** by also bumping `eslint` to a version that satisfies the new `eslint-config-next`'s `peerDependencies.eslint` range — see `@next/codemod upgrade` also bumps `eslint` (canary.73, PR #95314) below.

A single `nonInteractive` flag is plumbed into the four `suggest*` helpers, each short-circuits before its `prompts(...)` call and returns the default. Type-check clean.

**What each prompt accepts in non-interactive mode:**

| Prompt | Default accepted | Notes |
|---|---|---|
| React 18-or-19 upgrade | **upgrade React past 18** (`shouldStayOnReact18 = false`) | Existing `initial: false` is the no-op default |
| Turbopack enable | **yes** | If target version is 15.0.1-canary.3+ (uses `--turbopack`) |
| Custom dev-script fallback | **leave the script untouched** | Without a TTY we can't ask for a replacement |
| Recommended codemods multiselect | **apply all** | Matches `selected: true` on every choice; logged to stdout so the agent sees what ran |
| React 19 codemod | **yes** | Runs `codemod@latest react/19/migration-recipe --no-interactive --allow-dirty` |
| React 19 Types codemod | **yes** | Runs `types-react-codemod@latest --yes preset-19 .` |

The CLI also prints `Running in non-interactive mode. Every prompt will accept its default.` at the top so it's obvious why no prompts appear.

**Usage patterns:**

```bash
# From an AI coding agent or CI — auto-detected, no flag needed
npx @next/codemod upgrade canary

# From a terminal, explicit opt-in (same effect)
npx @next/codemod upgrade canary --yes

# CI / sandboxed run — explicit for log readability
npx @next/codemod upgrade minor --yes --verbose

# Upgrade to the latest patch only
npx @next/codemod upgrade patch --yes

**`useTscCli` CLI spinner fix shipped in canary.86 ([PR #95753](https://github.com/vercel/next.js/pull/95753))** — when `experimental.useTscCli: true` is enabled in `next.config.ts`, the Next.js type-check delegates to the new Go-based `tsc` subprocess for ~10× faster type-checks. Before canary.86 the spinner rendered `Running Typescript ...` and never cleared (the normal pause/resume pattern doesn't work with a subprocess). canary.86+ now stops the spinner as soon as the TSC subprocess produces output. **End-user impact:** `useTscCli` is now a clean experience on canary.86+ instead of a stuck spinner. See "canary.86" section above for full details.
```

**For agents running `next-dev-loop` / `next-cache-components-adoption`:** this PR is the reason `next-codemod upgrade` no longer breaks the loop. Before canary.72, agents had to invoke the codemods manually because the upgrade command hung.

**Files touched** (3 files, +93/-38):
- `docs/01-app/02-guides/upgrading/codemods.mdx` (+5/-0) — docs note that `-y, --yes` is auto-enabled on non-TTY
- `packages/next-codemod/bin/next-codemod.ts` (+5/-0) — `-y, --yes` option declared in `program.option(...)`
- `packages/next-codemod/bin/upgrade.ts` (+83/-38) — `nonInteractive` plumbing into all four `suggest*` helpers + CLI log line

**Sources:**
- [PR #95312 — `codemod: non-interactive upgrade for agents and CI`](https://github.com/vercel/next.js/pull/95312)
- [PR #95312 commit `6ab848610c`](https://github.com/vercel/next.js/commit/6ab848610ce3c478c5d4fb22ce8c187a709237f3)
- [Next.js docs — `codemods.mdx` for `-y, --yes`](https://github.com/vercel/next.js/blob/v16.3.0-canary.72/docs/01-app/02-guides/upgrading/codemods.mdx) (new section)
- [`packages/next-codemod/bin/upgrade.ts` at v16.3.0-canary.72](https://raw.githubusercontent.com/vercel/next.js/v16.3.0-canary.72/packages/next-codemod/bin/upgrade.ts) — `nonInteractive` flag (line ~114) + per-helper branches

### `@next/codemod upgrade` also bumps `eslint` (16.3.0-canary.73+, [PR #95314](https://github.com/vercel/next.js/pull/95314) by aurorascharff, merged 2026-07-01T14:25:47Z)

PR #95312 (the `--yes` non-interactive flag) made the upgrade codemod actually runnable from agents and CI. **PR #95314 closes the last remaining failure mode**: when the codemod bumps the installed `eslint-config-next` to the target version, the new `eslint-config-next` carries a `peerDependencies.eslint` range that the old `eslint` install on the project doesn't satisfy. The result is `npm install` failing with `ERESOLCVE` (or the equivalent `pnpm`/`yarn` error), which crashed the upgrade mid-flight.

**The fix** is in the `upgraders/eslint/index.ts` flow inside the codemod:

1. After detecting the target Next.js version, read the new `eslint-config-next`'s `peerDependencies.eslint` range (e.g. `^8.0.0`, `>=9.10.0`).
2. Resolve the highest matching `eslint` from the npm registry (`npm view eslint versions --json` filtered by the range).
3. Run `npm install --save-dev eslint@<resolved>` as part of the upgrade transaction (or `pnpm add -D`, depending on the package manager detected via the lockfile).
4. If the user's `eslint` was already a non-default pinned version (`^7.x.0` on a project that doesn't want to move), the resolver falls back to a **warning log** + skip rather than auto-bumping — so teams that deliberately pin `eslint` for compatibility don't get silently rewritten. Detected via `eslintConfig.extends` not referencing `eslint-config-next`.

**What changes for agents:**

```bash
# Before canary.73 / PR #95314: had to manually `npm install -D eslint@^9` first, then run upgrade
# After: the upgrade does it for you
npx @next/codemod upgrade canary --yes
# → installs next@canary + bumps eslint-config-next to match
# → bumps eslint to highest matching version of the new peer range
# → single command, single lockfile update
```

**Test fixture:** `cna-15.0.0-app` (default create-next-app output for Next.js 15.x, which has `eslint@^8`). Before PR #95314, the upgrade from 15 → 16.3 canary failed at the `eslint-config-next@^16.3.0` install with `ERESOLVE`; after, the upgrade installs `eslint@^9` (highest matching `eslint-config-next@16.3`'s `peerDependencies.eslint` of `^8.57.0 || ^9.0.0`) and proceeds. The fixture is checked into `packages/next-codemod/src/upgrade/__tests__/eslint-bump.test.ts`.

**Failure modes that are now handled:**

| Old behaviour (canary.72 + PR #95312) | New behaviour (canary.73 + PR #95314) |
|---|---|
| Upgrade succeeds → second `npm install` fails with `ERESOLVE` from `eslint-config-next`'s peer dep | Upgrade completes, lockfile is consistent, `npm install` is a no-op |
| Agent retries the upgrade command → same error | Agent runs upgrade once and it's done |
| Teams that pinned `eslint` to a non-default range got auto-bumped → broken plugins | Teams with non-default `eslint` pin get a warning + skip (preserves user choice) |

**Files touched** (3 files, +90/-12):
- `packages/next-codemod/src/upgrade/upgraders/eslint/index.ts` (+47/-5) — `resolveTargetEslintVersion(packageManager, peerRange)` helper + transaction integration
- `packages/next-codemod/src/upgrade/index.ts` (+28/-4) — wired the new helper into the `upgraders` pipeline; pin-detection logic
- `packages/next-codemod/src/upgrade/__tests__/eslint-bump.test.ts` (+15/-3) — `cna-15.0.0-app` regression test

**Sources:**
- [PR #95314 — `codemod: bump eslint alongside eslint-config-next`](https://github.com/vercel/next.js/pull/95314)
- [PR #95314 commit `8cbc9269`](https://github.com/vercel/next.js/commit/8cbc9269)
- [PR #95312 — `codemod: non-interactive upgrade for agents and CI` (canary.72)](https://github.com/vercel/next.js/pull/95312) — the upstream fix PR #95314 extends

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

### Vite 8.1.1 (June 30, 2026) — Patch Release with `resolve.tsconfigPaths` CSS Fix

Vite 8.1.1 ([npm `latest` pointer moved 2026-06-30T10:31:53Z](https://www.npmjs.com/package/vite)) is a **patch release** — 43 commits, no breaking changes, no API removals, no deprecations. It's the recommended upgrade for any project that adopted Vite 8.1.0 and runs into the `resolve.tsconfigPaths` regression or any of the other fixes below.

**The headline fix — `resolve.tsconfigPaths` for CSS and Sass `@import` (and Sass `@use`):**

PR [#22775](https://github.com/vitejs/vite/pull/22775) (Jerry Zhao, merged 2026-06-30T06:57:34Z, closes [#22766](https://github.com/vitejs/vite/issues/22766) by TimonLukas filed June 24) restores a regression introduced by the oxc-resolver upgrade in Vite 8.1.0. Since Vite 8.1.0, `resolve.tsconfigPaths: true` stopped resolving `paths` aliases inside CSS `@import` and Sass `@use`, e.g. `@import "@/styles/main.css"`. The Vite CSS plugin was passing a synthetic `<dir>/*` importer to the resolver, which oxc-resolver interpreted as "this file isn't in any tsconfig" and skipped the `paths` mapping. The fix passes the real importing file as the importer — CSS (postcss-import) uses the `@import` AtRule's source file, Sass already used `context.containingUrl`. Because tsconfig matches by extension, **CSS and Sass files must be declared explicitly in `tsconfig.json` `include`** to receive `paths`, e.g. `include: ["src", "src/**/*.css", "src/**/*.scss"]` — Vercel documents this under `resolve.tsconfigPaths`. Less is **not supported** by this fix (its plugin API only exposes the importer's directory, not the file); use a relative path or `resolve.alias` for `@import` in Less. The `playground/resolve-tsconfig-paths` Vite fixture gains CSS `@import` and Sass `@use` cases that resolve via `paths`, plus a Less case asserting the alias stays unresolved.

**Other user-facing fixes worth noting:**

| Fix | Why it matters | PR |
|---|---|---|
| **Escape ids with multiple null bytes** | `wrapId` / `unwrapId` / the Lightning CSS filename encoder used `String.prototype.replace`, which only replaces the first `\0`. When a virtual module was wrapped by another plugin (e.g. a CommonJS proxy around a virtual module produces `\0commonjs-proxy:\0virtual`), only the first null byte was handled, leaving raw null bytes in import URLs (invalid) and only restoring the first placeholder on decode (asymmetric). Lightning CSS treats null bytes as string terminators, so a leftover null byte truncated the filename. Switched to `replaceAll`. | [#22687](https://github.com/vitejs/vite/pull/22687) (Sujal Shah, 08:33Z, fixes #22690) |
| **`import.meta.hot.invalidate()` stack overflow in Bundled Dev Mode** | Calling `import.meta.hot.invalidate()` inside Bundled Dev Mode triggered an infinite recursion in the invalidation handler, crashing the dev server with a stack overflow. | [#22797](https://github.com/vitejs/vite/pull/22797) (Jerry Zhao, 07:49Z) |
| **Bundled Dev Mode: serve assets emitted during HMR/lazy compile** | Assets emitted by a plugin during HMR or lazy compilation inside Bundled Dev Mode were not being served, leaving imports broken until a full reload. | [#22745](https://github.com/vitejs/vite/pull/22745) (andrew, 09:56Z) |
| **Invert `esbuild.jsxSideEffects` → `oxc.jsx.pure`** | When Vite converts from esbuild's JSX side-effect flag to oxc's `jsx.pure`, it was copying the boolean directly, missing the inversion that the two transformers expect. Side-effectful JSX components could be incorrectly tree-shaken in Rolldown-backed builds. | [#22809](https://github.com/vitejs/vite/pull/22809) (Jerry Zhao, 05:54Z) |
| **Dynamic-import warning now links to Vite docs** | The warning you get from importing a non-existent module now points at the Vite docs page explaining the resolution rules, instead of just printing the error text. | [#22823](https://github.com/vitejs/vite/pull/22823) (翠, 09:43Z) |
| **Malformed URI in `indexHtmlMiddleware`** | A malformed URI in the request could crash the Vite dev server's HTML middleware. Now caught and handled. | [#22781](https://github.com/vitejs/vite/pull/22781) (greymoth, 09:57Z) |
| **pnpm `.modules.yaml` resolved from workspace root** | Vite resolved `.pnpm/.modules.yaml` from `cwd` instead of the workspace root in pnpm workspaces, leading to "could not find pnpm modules manifest" failures in pnpm monorepos when running Vite from a sub-package. Now resolves from the workspace root. | [#22757](https://github.com/vitejs/vite/pull/22757) (btea, June 25) |
| **Sourcemap field returned from plugins that were missing it** | Several internal plugins were missing the `sourcemap: true` field in their `transform` hook result, so the sourcemap pipeline silently dropped output sourcemaps. | [#22782](https://github.com/vitejs/vite/pull/22782) (翠, 09:34Z) |
| **Optimize-deps scanner ignores `ERR_CLOSED_SERVER`** | The dep-scanner used to crash on `ERR_CLOSED_SERVER` from the internal HTTP server when the parent process closed mid-scan (common in test harnesses and CI). Now ignored. | [#22784](https://github.com/vitejs/vite/pull/22784) (翠, 02:02Z) |

**Who should upgrade:** anyone on Vite 8.1.0 with `resolve.tsconfigPaths: true` and CSS `@import` statements (the regression above is silent — Vite throws a confusing `postcss` "can't find" error instead of a clean resolution-failure error), anyone using Bundled Dev Mode with `import.meta.hot.invalidate()` or custom HMR-emitting plugins, anyone in a pnpm workspace hitting the `.modules.yaml` error, and anyone who noticed sourcemaps vanishing from a sub-set of plugins in 8.1.0.

**Sources:**
- [Vite 8.1.1 release on GitHub](https://github.com/vitejs/vite/releases/tag/v8.1.1)
- [Vite 8.1.1 `vite/CHANGELOG.md` (raw)](https://github.com/vitejs/vite/blob/v8.1.1/packages/vite/CHANGELOG.md)
- [PR #22775 — `fix(css): resolve tsconfig paths in CSS and Sass @import` (Vite 8.1.1)](https://github.com/vitejs/vite/pull/22775)
- [Issue #22766 — `Vite 8.1 doesn't resolve tsconfigPaths in CSS @import directives`](https://github.com/vitejs/vite/issues/22766)
- [PR #22687 — `fix: escape ids with multiple null bytes`](https://github.com/vitejs/vite/pull/22687)
- [PR #22797 — `fix(bundled-dev): avoid stack overflow on import.meta.hot.invalidate()`](https://github.com/vitejs/vite/pull/22797)
- [PR #22745 — `fix(bundled-dev): serve assets emitted during HMR/lazy compile`](https://github.com/vitejs/vite/pull/22745)
- [PR #22809 — `fix: invert esbuild.jsxSideEffects when converting to oxc.jsx.pure`](https://github.com/vitejs/vite/pull/22809)
- [PR #22823 — `feat: update dynamic import warning to link to Vite docs`](https://github.com/vitejs/vite/pull/22823)
- [PR #22781 — `fix(server): handle malformed URI in indexHtmlMiddleware`](https://github.com/vitejs/vite/pull/22781)
- [PR #22757 — `fix: resolve pnpm .modules.yaml from workspace root instead of cwd`](https://github.com/vitejs/vite/pull/22757)

### Vite 8.1.2 (June 30, 2026) — Patch Release with Two Reverts

Vite 8.1.2 ([npm `latest` pointer moved 2026-06-30T16:46:39Z](https://www.npmjs.com/package/vite), same day as 8.1.1, ~6h later) is a **patch release** that reverts **two** changes from 8.1.1 that broke production users. **If you are on Vite 8.1.1, you should upgrade to 8.1.2** — the reverts below address real failures, not style preferences.

**Revert #1 — `es-module-lexer` 2.2.0 → 2.1.0 (PR [#22827](https://github.com/vitejs/vite/pull/22827), merged 2026-06-30T16:32:49Z):**

PR [#22827](https://github.com/vitejs/vite/pull/22827) (翠 / green@sapphi.red, reopens [#22750](https://github.com/vitejs/vite/issues/22750), fixes [#22826](https://github.com/vitejs/vite/issues/22826), refs [#22804](https://github.com/vitejs/vite/issues/22804)) reverts the `es-module-lexer` upgrade from `2.1.0` to `^2.2.0` that landed in 8.1.1. The `2.2.0` release (cut 2026-06-29T05:12:43Z) ships a **regression**: `parse()` throws `WebAssembly.Memory.grow(): Maximum memory size exceeded` on source inputs larger than ~4 MB. See [guybedford/es-module-lexer#214](https://github.com/guybedford/es-module-lexer/issues/214) (open as of 2026-06-30). The minimum reproduction:

```js
import { parse, init } from 'es-module-lexer';
await init;
const big = 'const x = "' + 'a'.repeat(5 * 1024 * 1024) + '";\n';
parse(big);  // throws on 2.2.0, works on 2.1.0
```

**Why this matters for Vite users:** Vite's `build-import-analysis` pass runs `parse()` against pre-bundled dependency chunks in dev. Any **production build with an output chunk over ~4 MB** now fails on Vite 8.1.1. Concrete failure reported in [#22826](https://github.com/vitejs/vite/issues/22826): dev server crashes with `Failed to parse source for import analysis because the content contains invalid JS syntax` when `@vitejs/plugin-react` is used alongside `@rolldown/plugin-babel` with `reactCompilerPreset`. The crash is also reachable in any bundle that combines several large vendor libraries into a single chunk (most production React apps with `lodash`, `monaco-editor`, `@mui/material`, or `react-pdf`).

**Revert #2 — `escape ids with multiple null bytes` (PR [#22687](https://github.com/vitejs/vite/pull/22687) reverted by [#cccef55](https://github.com/vitejs/vite/commit/cccef55dfaa6253929d2cb58d3af53f217efc877)):**

PR [#22687](https://github.com/vitejs/vite/pull/22687) (Sujal Shah, merged into 8.1.1) was reverted in 8.1.2 by sapphi-red with the explicit reason "broke some ecosystem-ci suites". The PR changed `replace('\0', NULL_BYTE_PLACEHOLDER)` to `replaceAll('\0', NULL_BYTE_PLACEHOLDER)` in `compileLightningCSS`, intending to fix Lightning CSS filename truncation when a virtual module was wrapped by another plugin (CommonJS-proxy → virtual modules like `\0commonjs-proxy:\0virtual`). The ecosystem-ci failures were:

- [**waku** (vite-ecosystem-ci run 28438066374, job 84268775303)](https://github.com/vitejs/vite-ecosystem-ci/actions/runs/28438066374/job/84268775303) — react-server-components framework on top of Vite
- [**vite-plugin-rsc** (vite-ecosystem-ci run 28438066374, job 84268775251)](https://github.com/vitejs/vite-ecosystem-ci/actions/runs/28438066374/job/84268775251) — Vite plugin for React Server Components

Both rely on virtual-module wrapping patterns that interact poorly with the broader replace-all. The revert restores the 8.1.0 behavior (single-null-byte replace), and the multi-null-byte edge case remains unfixed.

**Bonus — `#22757` re-PR'd with the actual fix (PR [#22825](https://github.com/vitejs/vite/pull/22825), merged 2026-06-30T15:47:27Z):**

PR [#22825](https://github.com/vitejs/vite/pull/22825) (翠) restores the `pnpm .modules.yaml` resolution-from-workspace-root behavior from #22757 with a corrected implementation. The first attempt (in 8.1.1) introduced a regression in `pnpm` monorepos where the same root `node_modules/.modules.yaml` was read twice with slightly different path contexts; the corrected version reads it once via `searchForWorkspaceRoot(root)` and uses the returned `workspaceRoot` consistently for both the manifest path and the `virtualStoreDir` resolution. **8.1.2 has the corrected version; 8.1.1 should be skipped entirely.**

**Who should upgrade to 8.1.2:**

- **Skip 8.1.1 entirely.** Going 8.1.0 → 8.1.2 directly is safe; 8.1.2 contains all of 8.1.1's intended fixes (resolve.tsconfigPaths CSS regression, Bundled Dev Mode fixes, sourcemap-field fix, optimize-deps ERR_CLOSED_SERVER handling, etc.) **plus** the corrected pnpm fix and **minus** the two breaking changes above.
- Anyone using **waku** or **vite-plugin-rsc** — these projects would not build on 8.1.1.
- Anyone with **production build output chunks over ~4 MB** — 8.1.1 would crash the import-analysis pass.
- Anyone in a **pnpm monorepo** — get the corrected `.modules.yaml` resolution.
- **Anyone on 8.1.0 with the `resolve.tsconfigPaths` CSS regression** — 8.1.2 has the fix.

**Who can stay on 8.1.0:** if you don't use `resolve.tsconfigPaths`, aren't in a pnpm monorepo, don't use waku or vite-plugin-rsc, and your prod output chunks are all under ~4 MB, Vite 8.1.0 still works. But there's no reason to stay — 8.1.2 is a pure patch on top.

**Sources:**
- [Vite 8.1.2 release on GitHub](https://github.com/vitejs/vite/releases/tag/v8.1.2)
- [Vite 8.1.2 `vite/CHANGELOG.md` (raw)](https://github.com/vitejs/vite/blob/v8.1.2/packages/vite/CHANGELOG.md)
- [PR #22827 — `fix(deps): revert es-module-lexer to 2.1.0`](https://github.com/vitejs/vite/pull/22827)
- [Issue #22826 — `vite:import-analysis fails on large files`](https://github.com/vitejs/vite/issues/22826)
- [guybedford/es-module-lexer#214 — `parse() throws "WebAssembly.Memory.grow(): Maximum memory size exceeded" on inputs larger than ~4 MB`](https://github.com/guybedford/es-module-lexer/issues/214)
- [Commit cccef55 — `fix: revert, "fix: escape ids with multiple null bytes (#22687)"`](https://github.com/vitejs/vite/commit/cccef55dfaa6253929d2cb58d3af53f217efc877)
- [PR #22687 (the reverted change) — `fix: escape ids with multiple null bytes`](https://github.com/vitejs/vite/pull/22687)
- [PR #22825 — `fix: restore, "fix: resolve pnpm .modules.yaml from workspace root instead of cwd (#22757)"`](https://github.com/vitejs/vite/pull/22825)
- [PR #22757 (the original) — `fix: resolve pnpm .modules.yaml from workspace root instead of cwd`](https://github.com/vitejs/vite/pull/22757)

**Recommended Vite version for new projects (June 30, 2026):** **Vite 8.1.2** — supersedes 8.1.1 (which should be **skipped entirely**) and contains all the 8.1.1-intended fixes (the `resolve.tsconfigPaths` CSS regression fix #22775, Bundled Dev Mode fixes #22797/#22745, sourcemap field #22782, ERR_CLOSED_SERVER #22784, etc.) plus the corrected pnpm workspace-root resolution (#22825) and **minus** the two breaking changes from 8.1.1 (es-module-lexer 2.2.0 WebAssembly regression and #22687 multi-null-byte escape). The skill previously recommended 8.1.1 in the immediately-prior update; that recommendation is **withdrawn** as of this update — the previous-update's recommendation of 8.1.1 should not be acted on without also taking this update's correction.

**Superseded by 8.1.3 (July 2, 2026):** The 8.1.2 "Recommended Vite version" recommendation above is now superseded by [Vite 8.1.3](#vite-813-july-2-2026--patch-release-with-es-module-lexer-230), which retains all of 8.1.2's contents **plus** the four 8.1.3 fixes (most importantly the `es-module-lexer` 2.1.0 → 2.3.0 bump in PR #22838, which removes the need for the 2.1.0 pin that 8.1.2 was forced into and brings the actual fix for the >~4 MB chunk issue that motivated the 8.1.2 revert). **Go 8.1.0 → 8.1.3 directly, or 8.1.2 → 8.1.3 for the four extra fixes.** Nothing in 8.1.2 changes; 8.1.3 is a pure additive patch on top.


### Vite 8.1.3 (July 2, 2026) — Patch Release with es-module-lexer 2.3.0

Vite 8.1.3 ([npm `latest` pointer moved 2026-07-02T04:45:03Z](https://www.npmjs.com/package/vite), 6 commits since 8.1.2, all bug fixes) is the **new recommended Vite version** as of this update. The headline is that 8.1.2's es-module-lexer 2.1.0 pin is no longer needed — **es-module-lexer 2.3.0** now ships the actual fix for the `WebAssembly.Memory.grow()` regression that 2.2.0 introduced and that 8.1.2 reverted around.

**Headline — `es-module-lexer` 2.1.0 → 2.3.0 (PR [#22838](https://github.com/vitejs/vite/pull/22838), merged 2026-07-02T02:57:23Z, fixes [#22750](https://github.com/vitejs/vite/issues/22750)):**

Vite 8.1.3 bumps `es-module-lexer` from `^2.1.0` (the 8.1.2 pin) to `2.3.0`. The 2.3.0 release ([es-module-lexer tag `2.3.0`, commit `dbac1c30c2`](https://github.com/guybedford/es-module-lexer/releases/tag/v2.3.0), cut 2026-07-02T01:13:23Z) ships **the actual fix for the `WebAssembly.Memory.grow(): Maximum memory size exceeded` regression** that caused Vite 8.1.1 to fail on prod output chunks >~4 MB and forced 8.1.2 to revert to 2.1.0. The fix is [#217](https://github.com/guybedford/es-module-lexer/pull/217) (es-module-lexer, merged 2026-07-02T00:52:30Z) — allows the WASM memory to grow beyond the initial 4 MB limit when sources exceed that size. 2.3.0 also adds [#211](https://github.com/guybedford/es-module-lexer/pull/211) — a minimal build exposed as `es-module-lexer/minimal` for es-module-shims integration (low-impact for Vite users; Vite doesn't import the minimal build).

**Why this matters:** 8.1.2 was the recommended version only because it pinned `es-module-lexer` to 2.1.0 to dodge the 2.2.0 regression. Projects that hit the >~4 MB chunk issue on 8.1.1 (most production React apps with `lodash`, `monaco-editor`, `@mui/material`, or `react-pdf` combined into a single chunk, plus `@vitejs/plugin-react` + `@rolldown/plugin-babel` with `reactCompilerPreset`) can now upgrade all the way to 8.1.3 with **no known es-module-lexer regressions**. Going 8.1.2 → 8.1.3 should be a no-op for projects that didn't have the >~4 MB issue; for projects that did, it's the actual upgrade path.

**Other fixes in 8.1.3:**

| Fix | Why it matters | PR |
|---|---|---|
| **Inlined CSS injected after the shebang line (`es`-format chunks)** | For `es`-format chunks that start with a shebang (`#!/usr/bin/env node\n<body>`), `injectInlinedCSS` was inserting the CSS-injection code *into* the shebang line instead of after it. Result: `#!/usr/bin/env nodeinjectCSS();\n<body>` — the shebang no longer pointed at a valid interpreter, and JS treats the leading `#!...` line as a hashbang comment, so the appended injection code was dropped and the inlined CSS was never injected. The no-newline fallback (`= 0`) was also wrong: it prepended the code *before* the shebang. Fix injects at `newlinePos + 1` and appends a newline if the shebang has no trailing one. Matches the existing `importAnalysisBuild.ts` behavior for the same case. | [#22717](https://github.com/vitejs/vite/pull/22717) (翠, 02:26:49Z, commit `1534d36`) |
| **Preload CSS for nested dynamic imports (Rolldown bug)** | In a nested dynamic import like `import('a').then(() => import('b'))` where `a` has a CSS dependency, the production build set the outer import's preload deps to `void 0`. The stylesheet for `a` was emitted to disk and listed in `__vite__mapDeps`, but nothing referenced its index, so no `<link>` was created and the CSS never loaded in production. Dev was unaffected (CSS injected by JS there). Root cause: Rolldown wraps the whole `import().then()` (from rolldown#8328 to tree-shake `import().then((m) => m.prop)`), which nests the inner `__VITE_PRELOAD__` marker before the outer one; the next-marker pairing picked the inner marker for both imports and the second write overwrote the first. Fix matches markers with a stack in a single pass (they nest like balanced brackets). Only affects Rolldown-backed builds (i.e., Vite 8+); Rollup-era Vite was unaffected because the legacy wrappe left `.then()` outside the `__vitePreload(...)` wrapper, keeping markers interleaved correctly. | [#22759](https://github.com/vitejs/vite/pull/22759) (翠, 03:55:06Z, fixes #22700, closes #22721) |
| **SSR correct stacktrace column position for first line** | Vite's SSR stacktrace mapping applied the wrong column offset when the stack frame pointed at the first line of the source, producing off-by-one or zero column positions in error overlay. | [#22828](https://github.com/vitejs/vite/pulls/22828) (翠, 02:19:11Z, fixes [#19627](https://github.com/vitejs/vite/issues/19627)) |

**Who should upgrade to 8.1.3:**

- **Anyone on Vite 8.1.0 or 8.1.1** who was holding back on 8.1.2 because of the >~4 MB chunk crash — go 8.1.0 → 8.1.3 directly (you get all of 8.1.1's intended fixes plus all of 8.1.3's, with no known es-module-lexer regressions).
- **Anyone on Vite 8.1.2** — the upgrade is safe and you get the four bug fixes above. Particularly relevant if you ship a service worker that starts with a shebang, a nested dynamic import with CSS deps behind Rolldown, or rely on SSR error overlay column positions.
- **Anyone on Vite 8.1.1** — **skip 8.1.2** and go straight to 8.1.3. There's no scenario where 8.1.2 is preferable to 8.1.3.

**Sources:**
- [Vite 8.1.3 release on GitHub](https://github.com/vitejs/vite/releases/tag/v8.1.3)
- [Vite 8.1.3 `vite/CHANGELOG.md` (raw)](https://github.com/vitejs/vite/blob/v8.1.3/packages/vite/CHANGELOG.md)
- [PR #22838 — `fix(deps): bump es-module-lexer to 2.3.0`](https://github.com/vitejs/vite/pull/22838)
- [es-module-lexer 2.3.0 release](https://github.com/guybedford/es-module-lexer/releases/tag/v2.3.0)
- [es-module-lexer #217 — `fix: allow wasm memory growth for sources over ~4MB`](https://github.com/guybedford/es-module-lexer/pull/217)
- [Issue #22750 — `Dev server corrupts a method named "import"` (the root es-module-lexer regression that motivated 2.2.0 → 2.1.0 revert in 8.1.2)](https://github.com/vitejs/vite/issues/22750)
- [PR #22717 — `fix(css): inject inlined CSS after the shebang line`](https://github.com/vitejs/vite/pull/22717)
- [PR #22759 — `fix: preload css for nested dynamic imports`](https://github.com/vitejs/vite/pull/22759)
- [PR #22828 — `fix(ssr): correct stacktrace column position for first line`](https://github.com/vitejs/vite/pull/22828)

**Recommended Vite version for new projects (July 2, 2026):** **Vite 8.1.3** — supersedes both 8.1.1 and 8.1.2 as the recommended Vite version. 8.1.3 contains all of 8.1.2's contents (the `resolve.tsconfigPaths` CSS regression fix #22775, Bundled Dev Mode fixes #22797/#22745, sourcemap field #22782, ERR_CLOSED_SERVER #22784, the corrected pnpm workspace-root resolution #22825, etc.) **plus** the four 8.1.3 fixes (`es-module-lexer` 2.1.0 → 2.3.0 in PR #22838, which removes the need for the 2.1.0 pin and brings the >~4 MB chunk fix in; CSS-after-shebang injection in #22717; nested-dynamic-import CSS preload in #22759; SSR stacktrace column position in #22828). **Skip 8.1.1 entirely** — go 8.1.0 → 8.1.3 (or 8.1.2 → 8.1.3). The 8.1.2 recommendation is superseded by 8.1.3; nothing about 8.1.2 changes, 8.1.3 just adds four fixes on top.

**Superseded by 8.1.4 (July 9, 2026):** The 8.1.3 recommendation above is now superseded by [Vite 8.1.4](#vite-814-july-9-2026--patch-release-7-bug-fixes--dep-updates), which retains all of 8.1.3's contents **plus** seven more bug fixes (plus two dependency bumps). Go 8.1.0 → 8.1.4 directly, or 8.1.3 → 8.1.4 for the seven extra fixes. Nothing in 8.1.3 changes; 8.1.4 is a pure additive patch on top.

### Vite 8.1.4 (July 9, 2026) — Patch Release (7 bug fixes + dep updates)

Vite 8.1.4 ([released 2026-07-09T04:40Z by `sapphi-red`](https://github.com/vitejs/vite/releases/tag/v8.1.4), 10 commits since 8.1.3, 7 bug fixes plus 2 dependency bumps) is the **new recommended Vite version** as of this update. Compares: [`v8.1.3...v8.1.4`](https://github.com/vitejs/vite/compare/v8.1.3...v8.1.4). Snyk reports 0 known vulnerabilities in 8.1.4 (same as 8.1.3). Wikipedia and the [Vite npm `latest` pointer](https://www.npmjs.com/package/vite) now point at 8.1.4. **No behavior changes** — every commit is a bug fix or a non-major dependency bump.

**Bug fixes in 8.1.4 (raw [CHANGELOG](https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md)):**

| Area | Fix | PR / Commit |
|---|---|---|
| **build** | Add workaround for building on [StackBlitz](https://stackblitz.com/). The StackBlitz WebContainer environment had a toolchain mismatch that prevented `vite build` from completing; the workaround is now active in 8.1.4. Unblocks production builds on StackBlitz (e.g., for teachers and StackBlitz-embedded demos). | [#22840](https://github.com/vitejs/vite/issues/22840) (commit `575c32c2`) |
| **build** | Keep `import.meta.url` in preload function as-is. Vite's import-analysis pass was rewriting `import.meta.url` references inside the `__vitePreload(...)` function body. That changed the semantics for libraries that read `import.meta.url` to resolve the module's own URL inside the preload helper (e.g., `new URL('./data.json', import.meta.url)` used to look up JSON from the helper's URL rather than the calling module's URL). Now preserved verbatim. | [#22839](https://github.com/vitejs/vite/issues/22839) (commit `f1f90ed4`) |
| **html** | Avoid regex backtracking in import-only check. The HTML transform's "this tag is an import-only inline module" heuristic had a regex pattern that backtracked catastrophically on long attribute values (a few KB of attribute text could lock up the dev server's transform worker for seconds). Now linear-time. | [#22848](https://github.com/vitejs/vite/issues/22848) (commit `b5868c01`) |
| **optimizer** | Avoid optimizer run for transform request before init. The dep-optimizer was responding to `transform` requests before its initialization completed, occasionally producing wrong output for the first transform request after `vite dev` startup (the dev-overlay would show a transient "unresolved export" error that cleared on the next HMR). Now waits for init. | [#22852](https://github.com/vitejs/vite/issues/22852) (commit `72a5e219`) |
| **ssr** | Align named-export function call stacktrace column with Node. For SSR errors that throw from inside a *named* function call (e.g., `add();` inside a server-side render function), Vite's stacktrace mapper was off-by-one column from what Node prints natively, so source maps pointed to a different column than the browser's stack frame would suggest. Brings the column in line with Node's native output for named calls — matches the same fix that #22828 shipped for the anon-function case in 8.1.3. | [#22829](https://github.com/vitejs/vite/issues/22829) (commit `173a1b64`) |
| **css chunks** | Strip pure-CSS-chunk imports when `experimental.chunkImportMap` is enabled. Pure CSS chunks (chunks whose only exports are side-effect CSS imports) were being kept as `__vitePreload(...)` dependencies for their consumers, producing redundant `<link>` tags for CSS that no JS imports. Now stripped at build time. | [#22841](https://github.com/vitejs/vite/issues/22841) (commit `648bd049`) |
| **deps** | `deps: update all non-major dependencies` — routine monthly dep bump. Pin for the specific diff is in the [8.1.4 release notes](https://github.com/vitejs/vite/releases/tag/v8.1.4). No API changes. | [#22865](https://github.com/vitejs/vite/issues/22865) (commit `d4295a9f`) |
| **deps** | `deps: update rolldown-related dependencies` — keeps Vite 8's rolldown toolchain in lockstep with upstream rolldown patches. | [#22866](https://github.com/vitejs/vite/issues/22866) (commit `7cf07e4c`) |

**Why this matters:**

- For the **vast majority** of Vite 8.1.x users, 8.1.4 is "8.1.3 plus seven tiny fixes" and the upgrade is a no-op. The safest upgrade path is `8.1.3 → 8.1.4` (a single patch bump, no API changes).
- The **`import.meta.url` fix (#22839)** is worth paying attention to if you use `new URL(..., import.meta.url)` to resolve sibling assets inside a preload function (rare, but people who hit it had no workaround in 8.1.3).
- The **`html` regex backtrack fix (#22848)** is worth paying attention to if you embed large data-URI images / JSON blobs / base64 inline in `<script>` tags or `<link>` attributes — the old regex could hang dev transforms on those (production was unaffected, dev would stall).
- The **stacktrace-column fix (#22829)** is the named-function companion to the SSR fix shipped in 8.1.3 (#22828 for anon functions). Together they make Vite's SSR error-overlay stack frames line up exactly with Node's native output, so source maps are now click-through-accurate for any frame.

**Who should upgrade to 8.1.4:**

- **Anyone on Vite 8.1.3** — pure additive patch; recommended upgrade. No API changes, no peer-dep changes, no behavior changes beyond the seven listed fixes.
- **Anyone on Vite 8.1.0 / 8.1.1 / 8.1.2** — go straight to 8.1.4 (you get every fix from 8.1.1 through 8.1.4 in one bump; skip 8.1.2 and 8.1.3 unless you have a specific reason to step).

**Sources:**

- [Vite 8.1.4 release on GitHub](https://github.com/vitejs/vite/releases/tag/v8.1.4)
- [Vite 8.1.4 `vite/CHANGELOG.md` (raw)](https://github.com/vitejs/vite/blob/v8.1.4/packages/vite/CHANGELOG.md)
- [Compare `v8.1.3...v8.1.4`](https://github.com/vitejs/vite/compare/v8.1.3...v8.1.4)
- PRs: [#22840](https://github.com/vitejs/vite/issues/22840), [#22839](https://github.com/vitejs/vite/issues/22839), [#22848](https://github.com/vitejs/vite/issues/22848), [#22852](https://github.com/vitejs/vite/issues/22852), [#22829](https://github.com/vitejs/vite/issues/22829), [#22841](https://github.com/vitejs/vite/issues/22841)

**Recommended Vite version for new projects (July 9, 2026):** **Vite 8.1.4** — supersedes 8.1.3 as the recommended Vite version (and transitively 8.1.1 / 8.1.2). 8.1.4 contains all of 8.1.3's contents (including the `es-module-lexer` 2.1.0 → 2.3.0 bump from #22838, the CSS-after-shebang injection in #22717, the nested-dynamic-import CSS preload in #22759, and the SSR stacktrace column position in #22828) **plus** seven more bug fixes (`import.meta.url` preservation in #22839, html-regex backtrack fix in #22848, optimizer init race in #22852, SSR named-call stacktrace column in #22829, pure-CSS-chunk strip in #22841, StackBlitz build workaround in #22840, and two monthly dep bumps in #22865 / #22866). **Skip 8.1.1 entirely** — go 8.1.0 → 8.1.4 (or 8.1.3 → 8.1.4). All of the 8.1.1 → 8.1.3 recommendation is preserved in 8.1.4.

### Vite 8.1.5 (July 16, 2026) — Patch Release (6 bug fixes + 2 doc fixes + 2 test/build refactors)

Vite 8.1.5 ([released 2026-07-16T06:51:13Z](https://github.com/vitejs/vite/releases/tag/v8.1.5), [compare `v8.1.4...v8.1.5`](https://github.com/vitejs/vite/compare/v8.1.4...v8.1.5), 10 commits since 8.1.4) is the **new recommended Vite version** as of this update. The npm `latest` dist-tag pointer moved to 8.1.5 at 2026-07-16T07:00Z. **No behavior changes** — every commit is a bug fix, documentation fix, or internal refactor.

**Bug fixes in 8.1.5 (raw [CHANGELOG](https://github.com/vitejs/vite/blob/v8.1.5/packages/vite/CHANGELOG.md)):**

| Area | Fix | PR / Commit |
|---|---|---|
| **bundled-dev** | Avoid duplicated `buildEnd` ([#22931](https://github.com/vitejs/vite/issues/22931)). In `BundledDev` mode the `buildEnd` plugin hook was being invoked twice when an HMR rebuild kicked off after a lazy-compile cycle, which could break plugins that rely on `buildEnd` being called exactly once per build (e.g. resource-allocators that release temp files in `buildEnd`). Now single-fired. | commit `810032097079be1a7da0e2b09ec9d92dd07ec13f` |
| **client** | Overlay error message format aligned with rolldown ([#22869](https://github.com/vitejs/vite/issues/22869)). The dev overlay's error message rendering was producing slightly different text than rolldown's native error reporter (different escaping for nested template literals, different line numbers in stack traces). Now matches what rolldown prints, so the dev overlay and a `vite build --logLevel info` log line show the same error text. | commit `5a72b8780705b575026617e86b0b92dea63a56a5` |
| **module-runner** | Don't crash stack-trace source mapping when `globalThis.Buffer` is absent ([#22945](https://github.com/vitejs/vite/issues/22945)). In browser-style module-runner hosts (Cloudflare Workers, edge runtimes) `globalThis.Buffer` is `undefined`; the stack-trace mapper was reading it as if it were always defined and crashed mid-mapping. Now guarded with `typeof globalThis.Buffer === 'function' ? ... : undefined`. | commit `f8b38e316bcefbf29f762f90ee49c88cd52c43b5` |
| **optimizer** | Respect importer module format for dynamic import interop with CJS deps ([#22951](https://github.com/vitejs/vite/issues/22951)). When the optimizer encounters a dynamic `import()` to a CJS dep, the interop wrapper was being generated assuming ESM-by-default, which produced wrong `default` exports on CJS modules that already had an ESM-shaped wrapper. Now respects the importer's module format. Fixes a class of "function missing on CJS dep" errors that only surfaced in builds with mixed CJS/ESM dep graphs. | commit `6c08c39ac4fb5868d080a51ff976a44693fc56ab` |
| **ssr** | Scope `switch-case` declarations to the switch, not the function ([#22893](https://github.com/vitejs/vite/issues/22893)). Vite's SSR transform was hoisting `let`/`const` declarations inside `case` clauses up to function scope, which broke the lexical scoping users expect from `switch` statements (e.g. `case 1: let x = 1; case 2: let x = 2` would error in user code but Vite's transform made it silently work and produced wrong behavior). Now wraps case bodies in their own block. | commit `b59a73f76f5557492d83d097bb33b3dd02f27d51` |
| **deps** | `deps: update all non-major dependencies` ([#22921](https://github.com/vitejs/vite/issues/22921)) — routine monthly dep bump. | commit `fef682d3f067d534a559faf6fd9baedda2e9f8f1` |
| **deps** | `deps: update rolldown-related dependencies` ([#22922](https://github.com/vitejs/vite/issues/22922)) — keeps Vite 8's rolldown toolchain in lockstep with upstream rolldown patches. | commit `3c345e475a5546a1cc6374682af89caebfe9c593` |
| **docs** | Fix incorrect `@default` for `build.cssMinify` ([#22948](https://github.com/vitejs/vite/issues/22948)) and for `build.lib.formats` ([#22911](https://github.com/vitejs/vite/issues/22911)). JSDoc-only — the runtime defaults are correct, the type-doc annotations were stale. | commits `c88c2361868d8e2384653e382dd0960ca1ff5b24c` + `369ed609a4aace3aee4e4194a54990694aa4e7ac` |
| **tests** | Avoid scanner scanning all files under `__tests__` ([#22912](https://github.com/vitejs/vite/issues/22912)). Test-infrastructure only. | commit `c961cae2868cc1521457ec60583867f0440e6949` |

**Why this matters:**

- For the **vast majority** of Vite 8.1.x users, 8.1.5 is "8.1.4 plus six tiny fixes" and the upgrade is a no-op. The safest upgrade path is `8.1.4 → 8.1.5` (a single patch bump, no API changes).
- The **`module-runner` `globalThis.Buffer` fix (#22945)** is worth paying attention to if you use Vite's module-runner in a browser-style host (Cloudflare Workers, Deno Deploy, Vercel Edge Functions). The crash was a silent-during-dev, hard-fail-in-prod class.
- The **`switch-case` scoping fix (#22893)** is a subtle correctness fix: if your code had multiple `case` clauses with `let`/`const` declarations of the same name (which should be a lexical error), Vite 8.1.4 was silently hoisting them and producing wrong runtime behavior. Upgrade to get the expected error.

**Who should upgrade to 8.1.5:**

- **Anyone on Vite 8.1.4** — pure additive patch; recommended upgrade. No API changes, no peer-dep changes, no behavior changes beyond the six listed fixes.
- **Anyone on Vite 8.1.0 / 8.1.1 / 8.1.2 / 8.1.3** — go straight to 8.1.5 (you get every fix from 8.1.1 through 8.1.5 in one bump).

**Sources:**

- [Vite 8.1.5 release on GitHub](https://github.com/vitejs/vite/releases/tag/v8.1.5)
- [Vite 8.1.5 `vite/CHANGELOG.md` (raw)](https://github.com/vitejs/vite/blob/v8.1.5/packages/vite/CHANGELOG.md)
- [Compare `v8.1.4...v8.1.5`](https://github.com/vitejs/vite/compare/v8.1.4...v8.1.5)
- PRs: [#22931](https://github.com/vitejs/vite/issues/22931), [#22869](https://github.com/vitejs/vite/issues/22869), [#22945](https://github.com/vitejs/vite/issues/22945), [#22951](https://github.com/vitejs/vite/issues/22951), [#22893](https://github.com/vitejs/vite/issues/22893), [#22921](https://github.com/vitejs/vite/issues/22921), [#22922](https://github.com/vitejs/vite/issues/22922)

**Recommended Vite version for new projects (July 16, 2026):** **Vite 8.1.5** — supersedes 8.1.4 as the recommended Vite version (and transitively 8.1.1 / 8.1.2 / 8.1.3). 8.1.5 contains all of 8.1.4's contents (the seven 8.1.4 bug fixes + two monthly dep bumps) **plus** six more bug fixes (bundled-dev duplicate `buildEnd`, dev-overlay error format alignment with rolldown, module-runner `globalThis.Buffer` guard, optimizer importer-format-aware CJS dynamic-import interop, SSR `switch-case` lexical scoping, plus two doc fixes + one test cleanup). **Skip 8.1.1 entirely** — go 8.1.0 → 8.1.5 (or 8.1.4 → 8.1.5). All of the 8.1.1 → 8.1.4 recommendation is preserved in 8.1.5.
### Vite `build.input` Option (July 17, 2026, ahead of Vite 8.1.5 — Vite PR [#22642](https://github.com/vitejs/vite/pull/22642) by sapphi-red, merged 2026-07-17T07:50:31Z)

**The most material new Vite feature since 8.1.5** — adds a new top-level **`build.input`** option typed as `string | string[] | Record<string, string>` that replaces manual `rollupOptions.input` declarations. Currently on Vite `main` ahead of the Vite 8.1.5 tag — will ship in either a Vite 8.1.6 patch or Vite 8.2.0 minor (no stable tag cut yet; verify with `npm view vite dist-tags`).

**Why it exists (4 reasons per the PR description):**

1. **Multi-page apps** — declaring multiple entrypoints declaratively without manually building `rollupOptions.input` arrays.
2. **Environment API per-environment entrypoints** — the long-standing [issue #21336](https://github.com/vitejs/vite/issues/21336) gap where each Environment API environment needed its own entrypoint configuration.
3. **Full-bundle mode** — Rolldown needs to know the entrypoint up front to generate the bundle efficiently; `build.input` makes that explicit.
4. **HMR bug class** — fixes [#17806](https://github.com/vitejs/vite/issues/17806) and [#16664](https://github.com/vitejs/vite/issues/16664) where the entrypoint resolution wasn't stable across dev restarts.

### Single-page setup (default behavior, unchanged)

```ts
// vite.config.ts
import { defineConfig } from 'vite'

export default defineConfig({
  build: {
    // Omit `input` — Vite auto-detects index.html and uses its <script type="module" src="...">
  },
})
```

### Multi-page setup (new `build.input` option)

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import { resolve } from 'node:path'

export default defineConfig({
  build: {
    input: {
      // key = output chunk name, value = entry HTML
      main: resolve(__dirname, 'index.html'),
      admin: resolve(__dirname, 'admin/index.html'),
      blog: resolve(__dirname, 'blog/index.html'),
    },
  },
})
```

This produces three HTML entrypoints in the build output, each with its own preload graph and chunk set. Replaces the old `rollupOptions.input` workaround:

```ts
// OLD pattern (still works, but `build.input` is preferred)
export default defineConfig({
  build: {
    rollupOptions: {
      input: {
        main: resolve(__dirname, 'index.html'),
        admin: resolve(__dirname, 'admin/index.html'),
      },
    },
  },
})
```

### Environment API per-environment setup (resolves issue #21336)

```ts
// vite.config.ts
import { defineConfig } from 'vite'

export default defineConfig({
  environments: {
    client: {
      build: {
        input: './src/entry-client.ts',
      },
    },
    ssr: {
      build: {
        input: './src/entry-server.ts',
      },
    },
    edge: {
      build: {
        input: './src/entry-edge.ts',
      },
    },
  },
})
```

**Why this matters:** before `build.input`, configuring per-environment entrypoints required manual `rollupOptions.input` per environment AND duplicated HTML plugin configuration. The new option is the canonical Environment API entrypoint declaration.

### Migration impact + audit recipe

```bash
# Find projects using the manual-rollup workaround
rg -l "rollupOptions\s*:\s*{[^}]*input\s*:" vite.config.* 
```

**Migration is trivial:** replace `rollupOptions: { input: ... }` with `build: { input: ... }`. No behavior change beyond cleanup — the old pattern continues to work (Vite's `build.input` is internally passed through to `rollupOptions.input`).

### Common mistakes

- **Confusing `build.input` with `build.lib.entry`** — `lib.entry` is for library mode (single bundle output, no HTML); `build.input` is for app mode (multi-entry HTML, separate chunk graphs). Pick one.
- **Using `build.input` to point at `.ts` files directly when you have an HTML entrypoint** — Vite's HTML plugin auto-discovers JS modules from `<script type="module" src="...">` tags in the HTML; pointing `input` at the `.ts` file directly skips that HTML→JS chain and produces a different output structure.
- **Setting `build.input` to a relative path without `resolve(__dirname, ...)`** — Vite resolves `build.input` relative to the project root, but if the entry HTML lives in a subdirectory you need an absolute path or `resolve()` to anchor it.

**Sources:**
- [Vite PR #22642 — `feat: add input option`](https://github.com/vitejs/vite/pull/22642)
- [Vite issue #21336 — Environment API per-environment entrypoints](https://github.com/vitejs/vite/issues/21336)
- [Vite issue #17806 — HMR entrypoint stability bug](https://github.com/vitejs/vite/issues/17806)
- [Vite issue #16664 — HMR entrypoint stability bug](https://github.com/vitejs/vite/issues/16664)
- [Vite `build` config docs (will be updated when 8.1.6 / 8.2.0 ships)](https://vite.dev/config/build-options.html)

### Vite `native` Config Loader Warnings (July 17, 2026, ahead of Vite 8.1.5 — Vite PR [#22850](https://github.com/vitejs/vite/pull/22850) by sapphi-red, merged 2026-07-17T06:10:20Z)

**Vite 9 will switch the default config loader from `bundle` (current) to `native` (Node's native ESM loader).** A class of features is unsupported in `native`:

- `__dirname` / `__filename` usage in ESM files (no `__dirname` in ESM by default)
- Extension-less imports in ESM files (`import './foo'` for `foo.js`)
- Directory index imports in ESM files (`import './foo'` for `foo/index.js`)
- JSON imports without attributes (`import './foo.json'` instead of `import foo from './foo.json' with { type: 'json' }`)
- ESM syntax in files Node loads as CommonJS

Vite 8.x now emits warnings for these under the current `bundle` loader so you can fix them ahead of the Vite 9 default change. **Warning format:**

```
(!) Your Vite config uses features that are unsupported by `configLoader: 'native'`,
which is planned to become the default in a future major version of Vite:
  - import "./svgVirtualModulePlugin" without a file extension (vite.config-base.js:3). Add the file extension
  - import "./matrixTestResultPlugin" without a file extension (vite.config-base.js:4). Add the file extension
  - import "./windows83Filename" without a file extension (vite.config-base.js:5). Add the file extension
Set `VITE_CONFIG_NATIVE_IGNORE_WARNING=true` to suppress this warning.
```

**Migration recipe (today, while still on Vite 8.x):**

```ts
// vite.config.ts — fixes for native-loader compatibility

// BEFORE (works in bundle, warns in native)
import svgPlugin from './svgVirtualModulePlugin'        // ❌ extension-less
import data from './config.json'                          // ❌ JSON without attribute
const __dirname = import.meta.dirname                   // ❌ workaround for ESM __dirname

// AFTER (works in both bundle and native)
import svgPlugin from './svgVirtualModulePlugin.js'      // ✅ explicit extension
import data from './config.json' with { type: 'json' }   // ✅ import attribute
const __dirname = new URL('.', import.meta.url).pathname // ✅ URL-based __dirname
```

**Suppress warnings temporarily:**

```bash
VITE_CONFIG_NATIVE_IGNORE_WARNING=true vite build
```

Useful for meta-frameworks that load the Vite config internally and can't easily fix the warnings.

**Sources:**
- [Vite PR #22850 — `feat(config): warn features incompatible with native loader in bundle loader`](https://github.com/vitejs/vite/pull/22850)
- [Vite issue #21546 — Original feature request](https://github.com/vitejs/vite/issues/21546)
- [Vite 9.0 config-loader migration tracking discussion](https://github.com/vitejs/vite/discussions)

### Vite `build.chunkImportMap` — CSS Chunks Now Mapped (July 17, 2026, ahead of Vite 8.1.5 — Vite PR [#22947](https://github.com/vitejs/vite/pull/22947) by DebadityaHait, merged 2026-07-17T13:08:28Z, fixes [#22946](https://github.com/vitejs/vite/issues/22946))

**Update to the Vite 8.1.4 fix** ([PR #22841](https://github.com/vitejs/vite/issues/22841), documented in setup.md above) that stripped pure-CSS-chunk imports when `experimental.build.chunkImportMap` is enabled. The 8.1.4 fix correctly stripped the JS-side preload references but **left the CSS preload array pointing at content-hashed filenames**, so a CSS-only change could leave a parent chunk filename unchanged while changing its cached preload array (stale preload, missing CSS until hard refresh). The new fix tracks each emitted CSS asset's originating chunk and adds a corresponding **stable CSS entry** to the generated import map, so preload rewriting uses the stable specifier instead of the content-hashed filename.

**Effect:**

- **Before:** CSS changes broke HMR-style incremental preload rewrites; users had to hard-refresh.
- **After:** CSS changes are stable across incremental rebuilds; preload array updates correctly without hard refresh.

**When this matters:**

- Multi-page apps where CSS is shared across pages
- Apps with a large CSS bundle that ships frequently
- Anyone using `experimental.build.chunkImportMap: true` (the experimental flag from Vite 8.1.0)

**No config change required** — the fix is in the existing chunkImportMap logic; the import map will automatically include CSS entries when this version (or later) is in your build path. Builds without `chunkImportMap` are unchanged.

**Sources:**
- [Vite PR #22947 — `fix(build): map CSS chunks in chunk import maps`](https://github.com/vitejs/vite/pull/22947)
- [Vite issue #22946 — Reported bug](https://github.com/vitejs/vite/issues/22946)
- [Vite PR #22841 — Original CSS chunk stripping fix in 8.1.4](https://github.com/vitejs/vite/issues/22841)

### Vite `networkInterfaceName` in Dev Output (July 17, 2026, ahead of Vite 8.1.5 — Vite PR [#22830](https://github.com/vitejs/vite/pull/22830) by Xavrir, merged 2026-07-17T07:01:24Z)

When running `vite --host` and you have multiple network interfaces (VPN, Docker bridge, WSL, Wi-Fi), the Network URL list now labels each URL with the interface name from `os.networkInterfaces()`. Interface names are capped at 20 characters with an ellipsis to keep the column tidy.

**Example output:**

```
  ➜  Local:   http://localhost:5173/
  ➜  Network: http://172.18.0.1:5173/     xray_tun
  ➜  Network: http://10.233.157.11:5173/  Wi-Fi
  ➜  Network: http://192.168.1.5:5173/    vEthernet (WSL (Hyp…
```

**Why it matters:** before this fix, the Network list was just a stack of IPs with no hint which one to open on your phone / external device. Now you can tell at a glance.

**Sources:**
- [Vite PR #22830 — `feat(dev): label network URLs with their interface name`](https://github.com/vitejs/vite/pull/22830)

### Vite `root` Real-Path Resolution (July 17, 2026, ahead of Vite 8.1.5 — Vite PR [#22832](https://github.com/vitejs/vite/pull/22832) by sapphi-red, merged 2026-07-17T11:12:16Z)

`root` is now resolved via `fs.realpathSync` so symlinked project roots behave consistently across dev and build. Previously, when the project lived behind a symlink (e.g., `~/projects/my-app` → `/srv/code/my-app`), dev and build could resolve `root` differently, causing intermittent "file not found" errors during HMR or build.

**No API change, no migration needed.** If you've been hitting symlink-related issues, upgrade to get the fix.

**Sources:**
- [Vite PR #22832 — `fix: resolve root to real path`](https://github.com/vitejs/vite/pull/22832)

### Turbopack `turbopackTreeShaking` → `turbopackModuleFragments` Rename (16.3.0-canary.92 in-flight, [PR #95978](https://github.com/vercel/next.js/pull/95978) by bgw, merged 2026-07-21T01:14:11Z)

The `experimental.turbopackTreeShaking` config option is **renamed** to `experimental.turbopackModuleFragments`. The old name was misleading — people thought it controlled Rolldown-style tree-shaking, but it's actually a different module-fragmentation optimization controlled by `turbopackRemoveUnusedExports` + `turbopackRemoveUnusedImports` (both default to `true`). Per [issue #95698](https://github.com/vercel/next.js/issues/95698) and the linked issues, the contributor confusion was significant.

**Action for users who set `turbopackTreeShaking` in their `next.config.ts`:**

```ts
// Before (still works, emits deprecation warning)
const nextConfig: NextConfig = {
  experimental: {
    turbopack: {
      turbopackTreeShaking: true,
    },
  },
}

// After (canonical)
const nextConfig: NextConfig = {
  experimental: {
    turbopack: {
      turbopackModuleFragments: true,
    },
  },
}
```

The old key will continue to work for one minor with a deprecation warning, then be removed.

**Audit recipe:**

```bash
rg "turbopackTreeShaking" next.config.* 
```

**Sources:**
- [Next.js PR #95978 — `[turbopack] Rename turbopackTreeShaking to turbopackModuleFragments`](https://github.com/vercel/next.js/pull/95978)
- [Next.js issue #95698 — Original rename request](https://github.com/vercel/next.js/issues/95698)

### Vite `PostcssUserConfig` Type Export (July 16, 2026, ahead of Vite 8.1.5 — Vite PR [#22792](https://github.com/vitejs/vite/pull/22792) by linyiru, merged 2026-07-16T11:42:15Z)

`PostcssUserConfig` is now re-exported from `vite`, so PostCSS configs can be typed against the same definition Vite loads them with. Closes [issue #19109](https://github.com/vitejs/vite/issues/19109).

```ts
// postcss.config.ts — now type-safe against Vite's internal definition
import type { PostcssUserConfig } from 'vite'

const config: PostcssUserConfig = {
  plugins: [/* ... */],
}

export default config
```

**Why this matters:** before, you had to either hand-write the type or use `any` for PostCSS configs; now you get autocomplete + type checking against the same definition Vite uses internally.

**Sources:**
- [Vite PR #22792 — `feat(css): export PostCSS config type for type-safe configs`](https://github.com/vitejs/vite/pull/22792)
- [Vite issue #19109 — Original feature request](https://github.com/vitejs/vite/issues/19109)

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

## `.browser.ts(x)` Variants — Browser-Only Module Splitting (16.3.0-canary.76+ post-canary.76, PR [#95201](https://github.com/vercel/next.js/pull/95201) + [#95200](https://github.com/vercel/next.js/pull/95200) by Sebastian "Sebbie" Silbermann / eps1lon, merged 2026-07-03T06:45:24Z)

First batch of fixes from Sebbie that addresses most of the recent bundle size regressions. Convention: when a module uses `typeof window !== 'undefined'` to conditionally require a browser-only dependency, Next.js can split the browser branch into a separate `.browser.ts(x)` sibling and resolve it via compiler alias instead of bundling both branches into the server bundle.

**The pattern it replaces** (works but ships unused server code):

```ts
// some-module.ts (or .ts, .tsx — any source file)
if (typeof window !== 'undefined') {
  // browser-only code path
  const { SomeBrowserOnlyDep } = require('some-browser-only-dep')
  // ...
} else {
  // server-only code path
}
```

The browser-only branch is dead code on the server, but the require call is still parsed and the dependency is still tracked in the server bundle's module graph. For modules that pull in sizable browser-only deps (analytics SDKs, charting libs, DOM utilities, IndexedDB wrappers, etc.), this can add hundreds of KB to the server bundle.

**The new pattern** (same module + a `.browser` sibling):

```ts
// some-module.ts (server / shared)
export function sharedHelper() { /* ... */ }

export async function browserOnlyOp() {
  if (typeof window === 'undefined') {
    // server stub
    return null
  }
  // dynamic require kept for backward compat; can be replaced with
  // an import once the .browser variant is set up
  const { SomeBrowserOnlyDep } = await import('./some-module.browser')
  return new SomeBrowserOnlyDep()
}
```

```ts
// some-module.browser.ts (browser-only sibling, picked up automatically)
import { SomeBrowserOnlyDep } from 'some-browser-only-dep'
export { SomeBrowserOnlyDep }
```

The compiler alias maps any module that has a `.browser` sibling to that sibling when bundling for the browser target. On the server, the original module is used as-is. **No hardcoded list of browser-variant modules** — PR #95200 detects any `.browser` sibling automatically and creates the compiler alias at build time.

**Files involved:**

- `crates/next-core/src/browser_variant_modules.rs` — Rust side: auto-detects `.browser` siblings and registers them as compiler aliases.
- `packages/next/src/build/browser-variant-modules.ts` — JS side: the list source for both webpack and Turbopack (kept for compatibility, but auto-augmented by detection).
- `packages/next/src/build/create-compiler-aliases.ts` — alias wiring.
- `.config/ast-grep/rules/no-typeof-window-require.yml` + `no-typeof-window-require-tsx.yml` (+ tests + snapshots) — a new lint rule that flags `typeof window` conditional `require()` calls and warns when the corresponding `.browser` sibling is missing. Future regressions handled automatically.

**Concrete examples of files converted in the PR:**

- `packages/next/src/client/components/client-boundary-params.ts` → `client-boundary-params.browser.ts`
- `packages/next/src/client/components/navigation.ts` → `navigation.browser.ts` (the navigation-dynamic-rendering + navigation-untracked modules also got `.browser` siblings)
- `packages/next/src/client/components/redirect.ts` → `redirect.browser.ts`
- `packages/next/src/client/components/handle-isr-error.tsx` → `handle-isr-error.browser.tsx`
- `packages/next/src/client/components/server-async-storage.ts` → `server-async-storage.browser.ts`

Test fixtures that were hardcoded in `test/production/app-dir/browser-chunks/browser-chunks.test.ts` (51 lines of expected chunk mapping) were removed because the detection is now automatic.

**Why this matters:** the PR description notes "First batch of fixes that addresses most of the recent bundle size regressions." Sebbie is systematically going through Next.js's internal modules to apply this pattern. Once the convention is established, app code can adopt it too: any time you find yourself writing `if (typeof window !== 'undefined') { require('something-big') }`, you can split out a `.browser.ts` sibling and Next.js will pick it up automatically.

**Tradeoff / known limitation** (from the PR body):

> "This isn't the smoothest DX yet. Ideally we'd generate the list at Next.js compile time (i.e. before publishing) to avoid having to rerun a script (I'll follow-up). Generating the list during build time (next dev or next build) is just another tiny cost users would have to pay."

For now, the detection runs at `next dev` / `next build` start. The follow-up will move it to Next.js's compile time (before publishing) so apps that adopt the convention don't pay even that tiny cost.

**Who needs to know:**

- **Anyone debugging a recent bundle size regression** on the server bundle — most of the candidates are now eligible for `.browser` splitting via the new convention.
- **Anyone maintaining a Next.js internal module** with a `typeof window` branch — adopt the `.browser` sibling pattern; the new ast-grep lint rule will flag missing siblings.
- **App authors who want smaller server bundles** — you can adopt the convention in your own code; the detection is automatic.

**Sources:**

- [PR #95201 — `Split typeof-window server requires into .browser variants`](https://github.com/vercel/next.js/pull/95201) · Commit `d6594bd111` (2026-07-03T06:45:24Z) · Sebastian "Sebbie" Silbermann.
- [PR #95200 — `Collect modules with browser variants statically`](https://github.com/vercel/next.js/pull/95200) · Commit `d3169de7e5` (2026-07-03T06:45:23Z) · Sebastian "Sebbie" Silbermann.
- `.config/ast-grep/rules/no-typeof-window-require.yml` + `.tsx` variant — new lint rule with regression tests.

## `cacheHandlers` Config Validation Now Anchored — PR [#95358](https://github.com/vercel/next.js/pull/95358) by Partha Shankar (16.3.0-canary.75+, merged 2026-07-02T16:12:24Z)

The `cacheHandlers` config validation previously used `/[a-z-]/`, which only checked whether a key contained at least one lowercase letter or hyphen. As a result, invalid keys such as `abc123`, `abc_123`, `abc.def`, and `handler!` were accepted even though the validation error message stated handler names must only contain `a-z` and `-`. **The regex wasn't anchored** — it was checking "any character" rather than "all characters".

**Fix:** Replace `/[a-z-]/` with `/^[a-z-]+$/` so the entire handler name is validated. Now the message and the enforcement match: only handler names made of lowercase letters and hyphens (e.g. `default`, `my-handler`, `cdn-cache`) are accepted; anything containing digits, underscores, dots, or other punctuation is rejected with the existing error message.

**Regression tests** added in `packages/next/src/server/config.test.ts` cover both valid and invalid handler names.

**Who needs to know:**

- **Anyone with an existing `cacheHandlers` config that uses digits, underscores, or other characters** — your handler names were silently accepted before; upgrading to canary.75+ will now reject them. Audit with `grep -E 'cacheHandlers' next.config.ts` and update any keys to `^[a-z-]+$`.
- **Anyone who noticed handler names like `abc123` "working" but felt uneasy** — they weren't supposed to.

**Source:** [PR #95358 — `fix(config): correctly validate cacheHandlers names`](https://github.com/vercel/next.js/pull/95358) · Commit `8688f98b1e` (2026-07-02T16:12:24Z) · Files: `packages/next/src/server/config-schema.ts` (validation regex) + `packages/next/src/server/config.test.ts` (regression tests).

## `experimental.serverComponentsHmrCancellation` — Stale-Refresh Cancellation Flag (16.3.0-canary.78+, [PR #95462](https://github.com/vercel/next.js/pull/95462) by unstubbable, merged 2026-07-04T12:23:36Z)

First PR in a series that lets `next dev` cancel an in-flight Server Components HMR refresh once a newer refresh supersedes it. **The flag itself is intentionally inert on its own** — flipping it to `true` in canary.78 does nothing observable yet. The plumbing (runtime config, env var, render options, build template) lands now so a follow-up PR can wire the actual cancellation logic without churning public config.

**What ships in canary.78:**

- `experimental.serverComponentsHmrCancellation?: boolean` — defaults `false`. Add to your `next.config.ts`:
  ```ts
  // next.config.ts
  export default {
    experimental: {
      serverComponentsHmrCancellation: true,
    },
  } satisfies import('next').NextConfig
  ```
- `process.env.__NEXT_SERVER_COMPONENTS_HMR_CANCELLATION` — runtime override, used by the `test-cache-components-dev` CI shard to force the flag on and by `test-cache-components-prod` as a guard. Leave this alone unless you're running the Next.js test suite.
- `RenderOptions.serverComponentsHmrCancellation` — plumbed from config through `base-server.ts` into the app-page build template.
- Production SSR template hardcodes `false` because edge rendering does not expose the Node response-close signal that the future cancellation logic relies on. The flag is dev-only meaningful; production SSR ignores it.

**Why it ships as inert plumbing:**

The rollout mirrors the `cachedNavigations` pattern from earlier in 16.3 — land the type + config + env var + render option + build-template wiring first so apps can opt in early and Next.js can collect telemetry on the config shape, then enable the actual behavior in a follow-up once the cancellation mechanism is finalized. This avoids a "toggle in the dark" rollout where users flip a flag and get inconsistent behavior depending on which canary build they're running.

**Who needs to know:**

- **App authors:** nothing to do today. The flag is opt-in and a no-op until the follow-up lands. Tracking is for awareness only — when the behavior PR lands, your `next dev` will start cancelling stale Server Component refreshes if you have the flag on (likely recommended default going forward).
- **Next.js contributors / test runners:** set `__NEXT_EXPERIMENTAL_SERVER_COMPONENTS_HMR_CANCELLATION=1` in your dev CI shard to exercise the flag-on path; the production shard uses the same env var as a guard against accidental production enablement (since edge SSR hardcodes `false`).
- **Anyone debugging dev-only HMR stale-update bugs:** once the follow-up lands, set the flag to `true` to opt into the new cancellation behavior. Default remains `false` for one or two more canaries until stability is confirmed.

**Source:** [PR #95462 — `Add serverComponentsHmrCancellation experimental flag`](https://github.com/vercel/next.js/pull/95462) · Commit `4eb5e7b1f1` (2026-07-04T12:23:36Z) · Files: `packages/next/src/server/config-shared.ts` (config schema) + `packages/next/src/server/base-server.ts` (render options) + `packages/next/src/build/templates/app-page.ts` (build template — hardcodes `false` for prod SSR) + `packages/next/src/server/app-render/rsc/entrypoints/rsc.ts` + `packages/next/src/server/app-render/work-unit-async-storage.external.ts` (dev-only plumbing).

### Activation — canary.79 + canary.80 wire the cancellation through

The canary.78 flag is no longer inert — two follow-up PRs landed in **canary.79 (July 7, 2026)** and **canary.80 (July 8, 2026)** that wire the actual cancellation behaviour:

- **[PR #95463](https://github.com/vercel/next.js/pull/95463) `Abort superseded Server Components HMR requests on the client` (canary.79, andrewimm)** — when `experimental.serverComponentsHmrCancellation: true`, the client now aborts an in-flight Server Components HMR fetch when a newer edit lands. Uses an `AbortController` per refresh + a version-stamped HMR route param so the dev server can detect stale work.
- **[PR #95486](https://github.com/vercel/next.js/pull/95486) `Cancel a superseded Server Components HMR refresh's server-side work` (canary.80, unstubbable)** — server-side counterpart. The dev render pipeline now respects the client abort signal and stops rendering work for a superseded refresh; previously the server kept rendering until completion and discarded the result.
- Both gated on `process.env.__NEXT_EXPERIMENTAL_SERVER_COMPONENTS_HMR_CANCELLATION` for the CI `test-cache-components-dev` shard, with a guard in `test-cache-components-prod`.

**Practical:** opt in only if you're testing the cancellation path on a non-shipping project — the flag is still experimental, and edge-rendered server components don't expose the Node response-close signal that the cancellation relies on (production SSR hardcodes `false`). For most projects, default `false` is correct until the experimental flag is documented as production-safe in a future stable.

## `experimental.requestInsights` — Dev-Only Request Diagnostics Stack (16.3.0-canary.84+, PR [#93974](https://github.com/vercel/next.js/pull/93974) + [#93975](https://github.com/vercel/next.js/pull/93975) + [#93976](https://github.com/vercel/next.js/pull/93976) by [@feedthejim](https://github.com/feedthejim), merged 2026-07-12T11:47:38–11:47:41Z)

Request Insights is a 5-PR dev-only feature stack that records framework OTEL spans, derives a bounded request history (last 100 requests, deduped fetch records), and exposes a snapshot over a private dev endpoint + HMR so AI agents + DevTools can diagnose what a route is doing without leaving the dev server. **Status of the 5 PRs:** PRs 1/5 ([#93974](https://github.com/vercel/next.js/pull/93974)) + 2/5 ([#93975](https://github.com/vercel/next.js/pull/93975)) + 3/5 ([#93976](https://github.com/vercel/next.js/pull/93976)) shipped in canary.84 (July 12, 2026); 4/5 ([#93977](https://github.com/vercel/next.js/pull/93977) — `next experimental-request-insights` CLI + `get_request_insights` MCP tool) shipped in canary.85 (July 13, 2026, 12:01:53Z); 5/5 ([#93978](https://github.com/vercel/next.js/pull/93978) — DevTools request panel) merged on the canary branch 2026-07-14T11:26:19Z and shipped in canary.86 (2026-07-14).

### What the 3 shipped PRs add

1. **[PR #93974](https://github.com/vercel/next.js/pull/93974) `request insights: record local framework spans (1/5)`** — adds a dev-only `LocalSpanRecorder` + `LocalRecordingSpan` (OTEL `Span` API-compatible; satisfies the `next/dist/compiled/@opentelemetry/api` `Span` interface) that mirrors framework spans into an in-memory store when no user OTEL provider is installed. The recorder is stored on `globalThis[Symbol.for('@next/local-span-recorder')]`; `tracer.ts` consults it on every span end. Preserves existing OTEL span API behavior when a user provider is installed.
2. **[PR #93975](https://github.com/vercel/next.js/pull/93975) `request insights: derive request history and fetch data (2/5)`** — wraps the recorder in a bounded `InMemoryRequestInsightsStore` (`MAX_REQUEST_INSIGHTS = 100`) keyed by `requestId`. Dedupes fetch records across completed `AppRender.fetch` spans vs direct fetch metrics. Sanitizes sensitive attribute keys via a `SENSITIVE_PARAM_NAME_RE` (matches `*token*` / `*secret*` / `*key*` / `*password*` / `*auth*` / `*signature*` / `*jwt*` / etc — case-insensitive) by replacing them with `'redacted'`; only the keys in `SAFE_SPAN_ATTRIBUTE_KEYS` (`http.method`, `http.route`, `http.status_code`, `http.url`, `next.fetch.cache_reason`, `next.fetch.cache_status`, `next.route`, `next.rsc`, `next.segment`, `next.span_name`, `next.span_type`, etc) pass through. URL query strings are sanitized via `sanitizeUrl()` before snapshotting. No raw header values, no auth headers, no bodies.
3. **[PR #93976](https://github.com/vercel/next.js/pull/93976) `request insights: expose dev snapshots to tools and HMR (3/5)`** — wires the new `experimental.requestInsights?: boolean` config flag (default `false`), the private dev endpoint, and the HMR transport.

### Configuration

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  experimental: {
    // Dev-only request diagnostics. Default false.
    // When true, next dev:
    //   - starts the LocalSpanRecorder
    //   - exposes GET /_next/development/request-insights (returns the last 100 requests)
    //   - pushes REQUEST_INSIGHTS_UPDATE messages over the HMR transport
    requestInsights: true,
  },
}

export default nextConfig
```

**The flag is dev-only.** `next build` and `next start` ignore it (no production exposure, no telemetry). When the flag is `false`, the endpoint returns a 404 with a helpful JSON body:

```json
{ "error": "Request Insights is not enabled. Set experimental.requestInsights = true and restart next dev." }
```

### Surface — the private dev endpoint

While `next dev` is running with the flag on:

```bash
# Fetch the last 100 requests as a JSON snapshot
curl -sS http://localhost:3000/_next/development/request-insights | jq '.requests[0]'
```

The endpoint is dev-only (`opts.dev` gate) and additionally gated on `blockCrossSiteDEV(req, res, development.config.allowedDevOrigins, opts.hostname)` — same CSRF guard the existing dev-only Next.js internal endpoints use. Same-origin requests from `http://localhost:3000` work; cross-origin requests get a CSRF error.

**Snapshot shape** (from `packages/next/src/next-devtools/shared/request-insights.ts`):

```ts
type RequestInsightsSnapshot = {
  requests: RequestInsight[]
}

type RequestInsight = {
  requestId: string
  htmlRequestId: string
  route?: string
  url?: string
  startTime: number
  durationMs?: number
  status: 'ok' | 'error' | 'pending'
  spans: RequestInsightSpan[]  // framework OTEL spans, sanitized
  fetches: RequestInsightFetch[]  // deduped fetch records
}

type RequestInsightFetch = {
  url?: string
  method?: string
  statusCode?: number
  startTime?: number
  durationMs?: number
  cacheStatus?: string
  cacheReason?: string
  index?: number
}
```

### Surface — the HMR transport

`HMR_MESSAGE_SENT_TO_BROWSER.REQUEST_INSIGHTS_UPDATE` (`'requestInsightsUpdate'`) carries per-request updates to subscribed tools. The browser-side reducer in `dev-overlay/shared.ts` accepts two new action types:

- `ACTION_REQUEST_INSIGHTS_SNAPSHOT` (`'request-insights-snapshot'`) — full snapshot replacement
- `ACTION_REQUEST_INSIGHTS_UPDATE` (`'request-insights-update'`) — incremental per-request update

The `DevBundlerService` subscribes to the in-memory store on construction (`subscribeRequestInsights` in `dev-bundler-service.ts`) and forwards each `RequestInsight` to the HMR socket. `unsub` is wired into `close()` so dev-server shutdown doesn't leak the subscription.

### How an agent (or the DevTools panel) consumes it — SHIPPED in canary.85 (4/5) and canary.86 (5/5)

The agent-consumption path landed across the last two canary cuts:

- **[PR #93977](https://github.com/vercel/next.js/pull/93977) `request insights: add agent diagnosis access (4/5)`** — SHIPPED in **canary.85 (July 13, 2026, 12:01:53Z)**. Adds **two** agent-accessible surfaces, both backed by the same in-memory store:

  - **`next experimental-request-insights` CLI** (new) — `packages/next/src/cli/next-request-insights.ts` (+201/-0) + `bin/next.ts` (+30/-0) wire the new `program.command('experimental-request-insights')` subcommand. Auto-discovers the running dev server from the project's `.next/lock` file, calls the `/_next/development/request-insights` endpoint, and prints a human summary (newest-first, default 20 requests, with per-request route + duration + status + first 5 fetches). Flags:
    - `--url <url>` — override the auto-discovered dev server URL (must be `http://` or `https://`)
    - `--json` — print the raw `RequestInsightsSnapshot` JSON (pipe to `jq`)
    - `--limit <n>` — max number of recent request summaries to print (positive integer)
  - **`get_request_insights` MCP tool** — `packages/next/src/server/mcp/tools/get-request-insights.ts` (+57/-0) is now a standard tool registered in `get-or-create-mcp-server.ts`. The tool's input schema is `{ requestId?: string, htmlRequestId?: string }` (both optional filters) and it returns the filtered `RequestInsight[]` as the MCP `content[].text` payload. The tool records its own telemetry call (`mcp/get_request_insights`) via the existing `mcpTelemetryTracker`. If `experimental.requestInsights` is off, the tool returns a friendly JSON error rather than throwing. **Note:** the original PR description suggested a `subscribe_to_request_insights` tool would land alongside — that surface was **deferred** (the live stream is already available over the HMR transport `HMR_MESSAGE_SENT_TO_BROWSER.REQUEST_INSIGHTS_UPDATE` for browser subscribers; for MCP clients, the snapshot tool is the only shipped surface, and the call should be re-invoked on a short interval to approximate streaming).

- **[PR #93978](https://github.com/vercel/next.js/pull/93978) `request insights: add DevTools request panel (5/5)`** — SHIPPED on canary branch 2026-07-14T11:26:19Z (will be in **canary.86**, shipped 2026-07-14T23:31:39Z in canary.86). Adds a new "Request Insights" panel in the dev-overlay DevTools (`packages/next/src/next-devtools/dev-overlay/components/request-insights/{request-insights-panel.tsx, request-list.ts, trace-viewer.ts, request-insights-panel.css, format-duration.ts}`). The panel shows:

  - A retained request list with **stable selection** as new requests arrive (selected request id is preserved until the request is no longer in the 100-request buffer)
  - A **"Page load"** marker on the exact initial document request (identified by `self.__next_r === request.requestId`)
  - An **end-to-end trace timeline** that follows the recorded span parent/child hierarchy (timeline range is the request's full duration; outlier spans are clipped to that range)
  - A **focused default view** showing the most useful high-level spans, and a **verbose toggle** that surfaces every recorded span for deep investigation
  - **Fetch + cache + request + status + duration + span-count** details for the selected request
  - **Sub-millisecond durations rendered readably** and **right-aligned request metrics** for at-a-glance comparison
  - **Merged fetch metrics with their corresponding `AppRender.fetch` spans** so a single timeline row shows the full latency without double-counting

  Empty state copy: *"Request insights will appear after the next App Router request."*

**As of 16.3.0-canary.85+, the full agent loop looks like this (4/5 + 5/5 of the stack are live, canary.85 ships the MCP + CLI, canary.86 ships the DevTools panel):**

1. Agent edits a route in `app/dashboard/page.tsx`.
2. Agent calls `mcp__next-devtools-mcp__get_request_insights({ requestId: '<latest>' })` (or, for shell-only access, runs `next experimental-request-insights --json | jq '.requests[0]'`).
3. Agent reads the `spans[]` — finds e.g. `next.fetch /api/user` took 1800ms, with `http.status_code: 500` and a `cache_reason: 'cache-miss'`.
4. Agent correlates with the edit — the user just added a new `headers().get('x-tenant')` call in a Server Component, but the `next/cache` directive is on the same scope. Agent surfaces: "this Server Component is no longer eligible for `use cache` because it reads request-time headers; either pass the header into the cached function or split the read into a separate Server Component."
5. Agent edits the file. Repeats.
6. *The human can also pop the DevTools overlay (canary.86+)* → "Request Insights" tab to see the same data as a sortable table with span-tree drill-down — no MCP required.

**The CLI is the right starting point** for shell-only agents and CI scripts (no MCP overhead, no jq one-liner required to be useful on its own). **The MCP tool is the right surface** for in-editor agents (Claude Code, Cursor, etc.) that already have `next-devtools-mcp` configured. **The DevTools panel** is the right surface for humans visually verifying what an agent just claimed.

### `next experimental-request-insights` CLI (16.3.0-canary.85+, PR [#93977](https://github.com/vercel/next.js/pull/93977))

A first-class CLI for reading the request insights snapshot from a running dev server, without needing curl + jq. The CLI lives in `packages/next/src/cli/next-request-insights.ts` and is wired into `bin/next.ts`.

```bash
# Auto-discover the dev server from .next/lock in the current directory
# and print a human summary (default: newest 20 requests, 5 fetches per request)
next experimental-request-insights

# Output:
#   Showing 20 of 23 retained requests (newest first).
#   /dashboard 1.24s 200
#     request abc12345 page page12345
#     fetch 18ms 200 miss GET https://api.example.com/me
#     fetch 240ms 200 miss GET https://api.example.com/feed
#   /settings 312ms 200
#     request def67890 page page67890
#     ...

# Pipe to jq for filtering
next experimental-request-insights --json | jq '.requests[] | select(.durationMs > 1000) | {route, durationMs}'

# Override the dev server URL (skips lockfile auto-discovery)
next experimental-request-insights --url http://localhost:3000

# Limit the number of requests printed
next experimental-request-insights --limit 5
```

**Auto-discovery** reads the project's `<distDir>/lock` file (default `.next/lock`) and parses the `appUrl` field — same lockfile the rest of the dev server uses for port assignment. Discovery is bounded by `DEV_SERVER_DISCOVERY_TIMEOUT_MS = 1000` with a `DEV_SERVER_DISCOVERY_RETRY_MS = 50` poll interval. If no dev server is found, the CLI exits with: `Unable to discover a running Next.js dev server from <lockfile>. Start next dev or pass --url.`

**Validation:**

- `--url` must be a valid `http://` or `https://` URL (rejects `ftp://` etc.)
- `--limit` must be a positive integer
- Response is validated: must be a JSON object with `requests: RequestInsight[]` where each request has `fetches: []` — otherwise the CLI exits with `Invalid response from <endpoint>: expected requests and fetches to be arrays.` and dumps the raw response

**Duration formatting:** sub-second values render as `120ms`, longer values as `1.24s`. Request IDs and page IDs are truncated to 8 chars for readability.

**Source:** [PR #93977 — `next experimental-request-insights` CLI](https://github.com/vercel/next.js/pull/93977) · [`next-request-insights.ts` at v16.3.0-canary.85+](https://github.com/vercel/next.js/blob/v16.3.0-canary.85/packages/next/src/cli/next-request-insights.ts) · [`bin/next.ts` at v16.3.0-canary.85+](https://github.com/vercel/next.js/blob/v16.3.0-canary.85/packages/next/src/bin/next.ts)
### What this does NOT do

- **Does not record or replace the user's own OTEL provider.** The `LocalSpanRecorder` is bypassed when a user OTEL provider is installed. No double-emission, no leaks.
- **Does not persist anything to disk.** Pure in-memory `Map<requestId, RequestInsight>` + an `order[]` array for the most-recently-inserted-first ordering. Server restart loses the buffer. The 100-request bound is hard — older requests are evicted in LRU order.
- **Does not send any data off-box.** The endpoint is `localhost`-only by the CSRF gate; the HMR transport is the same `/_next/webpack-hmr` / Turbopack HMR socket that's already authenticated. No new outbound network surface.
- **Does not apply to production builds.** `next build` strips the flag (config-schema + define-env default). No perf impact on prod bundles.
- **Does not record bodies, headers, cookies, or auth tokens.** Only safe attribute keys pass through, sensitive params are redacted, URLs are sanitized. Designed to be safe to leave on in a multi-tenant dev environment.

### Audit + verification commands

```bash
# Verify the flag is set
rg 'requestInsights' next.config.ts

# Verify the config-schema knows about it
rg 'requestInsights' node_modules/next/dist/server/config-schema.d.ts 2>/dev/null   || curl -s https://raw.githubusercontent.com/vercel/next.js/v16.3.0-canary.84/packages/next/src/server/config-schema.ts | rg 'requestInsights'

# Pull a snapshot from a running dev server
curl -sS http://localhost:3000/_next/development/request-insights | jq '.requests[].status' | sort | uniq -c

# Show only failing requests with their top span
curl -sS http://localhost:3000/_next/development/request-insights |   jq '.requests[] | select(.status == "error") | {route, durationMs, topSpan: .spans[0].name}'

# Tail spans for one request by id
curl -sS http://localhost:3000/_next/development/request-insights |   jq '.requests[] | select(.requestId == "abc123") | .spans[] | {name, durationMs, status}'
```

### Files touched (43 files, ~1.7k lines added, mostly test scaffolding)

- `packages/next/src/server/config-shared.ts` (+9/-0) — `ExperimentalConfig.requestInsights?: boolean` + JSDoc; `requestInsights: false` default in the legacy config-defaults block
- `packages/next/src/server/config-schema.ts` (+1/-0) — Zod entry, picked up via the existing `experimental` schema
- `packages/next/src/build/define-env.ts` (+2/-0) — `__NEXT_REQUEST_INSIGHTS` build-time define (default `false`)
- `packages/next/src/server/lib/trace/local-span-recorder.ts` (+473/-1) — `LocalSpanRecorder` type, `createLocalSpan` / `getActiveLocalSpan` / `isLocalRecordingSpan` / `withLocalSpan`; hex-pad synthetic trace/span IDs for visual continuity with real OTEL IDs
- `packages/next/src/server/lib/trace/span-store.ts` (+83/-0) — `SpanStoreRecord` + `recordSpan` + `isLocalSpanRecordingEnabled`; test-only recorder hook
- `packages/next/src/server/lib/trace/tracer.ts` (+125/-11) — recorder is consulted on span start/end; when no user OTEL provider is installed, the recorder's local span is the active one
- `packages/next/src/server/lib/trace/request-insights.ts` (+337/-0) — `InMemoryRequestInsightsStore`; `recordSpan` / `recordFetch` / `getSnapshot`; `subscribeRequestInsights` listener registry; `sanitizeSpanAttributes` / `sanitizeUrl` / `getFetchInsight` helpers
- `packages/next/src/server/lib/router-server.ts` (+46/-0) — `GET /_next/development/request-insights` endpoint; CSRF-gated, dev-only, returns 404-with-helpful-error when off
- `packages/next/src/server/lib/patch-fetch.ts` (+43/-8) — `isRequestInsightsEnabled()` check before recording a fetch metric
- `packages/next/src/server/dev/hot-middleware.ts` (+8/-0) — emits `HMR_MESSAGE_SENT_TO_BROWSER.REQUEST_INSIGHTS_UPDATE` to connected clients
- `packages/next/src/server/dev/hot-reloader-turbopack.ts` (+9/-0) — Turbopack equivalent of the above
- `packages/next/src/server/dev/hot-reloader-types.ts` (+12/-0) — `RequestInsightsUpdateMessage` type + new `HMR_MESSAGE_SENT_TO_BROWSER.REQUEST_INSIGHTS_UPDATE` enum value
- `packages/next/src/server/dev/next-dev-server.ts` (+3/-0) — sets `process.env.__NEXT_REQUEST_INSIGHTS = 'true'` when the flag is on
- `packages/next/src/server/next-server.ts` (+8/-0) — same env-var plumbing on the next-server path
- `packages/next/src/server/base-server.ts` (+6/-0) — request identity propagation through the existing app-render async storage (so spans across boundaries share a `requestId`)
- `packages/next/src/server/app-render/app-render.tsx` (+4/-0) — record `app-render` span start/end
- `packages/next/src/server/app-render/work-async-storage.external.ts` (+8/-0) — `htmlRequestId` plumbing
- `packages/next/src/server/async-storage/work-store.ts` (+2/-0) — `workStore.htmlRequestId` field
- `packages/next/src/server/lib/dev-bundler-service.ts` (+17/-2) — `subscribeRequestInsights` on construction, `unsub` on `close()`
- `packages/next/src/next-devtools/dev-overlay/shared.ts` (+30/-0) — `OverlayState.requestInsights` slice + 2 new reducer action types + new `'requestInsightsUpdate'` HMR-message kind
- `packages/next/src/next-devtools/dev-overlay.browser.tsx` (+18/-0) — browser-side reducer wired
- `packages/next/src/next-devtools/shared/request-insights.ts` (+59/-0) — `RequestInsight` / `RequestInsightSpan` / `RequestInsightFetch` / `RequestInsightsSnapshot` types (shared with the server via a normal `import type`)
- `packages/next/src/client/dev/hot-reloader/app/hot-reloader-app.tsx` (+6/-0) — HMR message handler dispatches the new action
- `packages/next/src/client/dev/hot-reloader/pages/hot-reloader-pages.ts` (+1/-0) — Pages-router equivalent (no-op today, but the action type is reserved)
- `packages/next/src/client/page-bootstrap.ts` (+1/-0) — hydrate initial snapshot from the dev server
- `packages/next/.storybook/decorators/use-overlay-reducer.ts` (+4/-0) — Storybook wiring
- Plus **8 new test files** under `packages/next/src/server/lib/trace/{local-span-recorder,span-store,request-insights,patch-fetch,tracer}.test.ts` (~990 lines)

#### Span collection hardening — `next.span_category` + request-identity propagation (16.3.0-canary.88+, PR [#95818](https://github.com/vercel/next.js/pull/95818) by [feedthejim](https://github.com/feedthejim), merged 2026-07-16T12:24:44Z)

Followup to the canary.84–86 Request Insights stack that fixes a class of dev-only span-collection bugs that affected the `get_request_insights` MCP tool, the `next experimental-request-insights` CLI, and the DevTools "Request Insights" panel. Summary of the PR:

- **All non-hidden Next.js spans are now recorded locally in dev** (including non-vanilla trace and wrapped spans that the canary.84 stack missed because the original `LocalSpanRecorder` only observed the bare OTEL tracer). The existing OpenTelemetry export allowlist is preserved unless verbose tracing is enabled.
- **Request identity is propagated before work-async storage exists** so timelines are correct even for very early spans (previously a span whose parent had no `workUnitStore` yet would have a wrong `requestId` and get filtered out as orphan). Explicit span timestamps are honored.
- **Request duration is anchored to the `BaseServer` request span** (was previously taken from the first recorded child span, which under-counted by the span-startup window on cold start).
- **Span categories are standardized on a new `next.span_category` attribute** — e.g. `next.span_category = 'fetch' | 'cache' | 'render' | 'compile' | 'request'`. Presentation-only UI filtering (the focused/verbose toggle in the DevTools panel) reads this attribute instead of internal span-name heuristics, so custom framework spans don't accidentally fall into a hidden category.
- **Internal request-identity headers are filtered at ingress** to prevent the same header values from leaking into the dev-snapshot store (was previously filtered only at egress).

**Why this matters in practice:** if you were getting an empty span list in `get_request_insights` despite enabling `experimental.requestInsights: true`, or if your `AppRender.fetch` spans were missing from the DevTools panel's "focused" view, this PR is the fix. It does not change the public API (the flag, endpoint, MCP tool, CLI, and DevTools panel are all unchanged) — it just makes the dev snapshot match the actual on-page behavior more accurately. **Verification:** `pnpm test-dev-turbo test/development/app-dir/request-insights/request-insights.test.ts`.

**Source:** [PR #95818 — `Fix Request Insights span collection`](https://github.com/vercel/next.js/pull/95818) · Commit `491f7809` · feedthejim · merged 2026-07-16T12:24:44Z · **will be in canary.88** (canary branch HEAD at 2026-07-16T18:00Z is `491f7809`, 11 commits ahead of the canary.87 tag `50925bb80`).

**Sources:**
- [PR #93974 — `request insights: record local framework spans (1/5)`](https://github.com/vercel/next.js/pull/93974)
- [PR #93975 — `request insights: derive request history and fetch data (2/5)`](https://github.com/vercel/next.js/pull/93975)
- [PR #93976 — `request insights: expose dev snapshots to tools and HMR (3/5)`](https://github.com/vercel/next.js/pull/93976)
- [PR #93977 — `request insights: add agent diagnosis access (4/5)` (SHIPPED in canary.85, 2026-07-13T12:01:53Z)](https://github.com/vercel/next.js/pull/93977)
- [PR #93978 — `request insights: add DevTools request panel (5/5)` (SHIPPED in canary.86 on 2026-07-14T23:31:39Z; PR merged on canary branch 2026-07-14T11:26:19Z)](https://github.com/vercel/next.js/pull/93978)
- [canary.84 release notes](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.84)
- [canary.85 release notes](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.85)
- [`config-shared.ts` at v16.3.0-canary.84](https://raw.githubusercontent.com/vercel/next.js/v16.3.0-canary.84/packages/next/src/server/config-shared.ts) — `requestInsights` declaration
- [`request-insights.ts` (server) at v16.3.0-canary.84](https://raw.githubusercontent.com/vercel/next.js/v16.3.0-canary.84/packages/next/src/server/lib/trace/request-insights.ts) — store + sanitization
- [`request-insights.ts` (shared types) at v16.3.0-canary.84](https://raw.githubusercontent.com/vercel/next.js/v16.3.0-canary.84/packages/next/src/next-devtools/shared/request-insights.ts) — `RequestInsight` shape
- [`next-request-insights.ts` CLI at v16.3.0-canary.85](https://github.com/vercel/next.js/blob/v16.3.0-canary.85/packages/next/src/cli/next-request-insights.ts) — new `next experimental-request-insights` command
- [`get-request-insights.ts` MCP tool at v16.3.0-canary.85](https://github.com/vercel/next.js/blob/v16.3.0-canary.85/packages/next/src/server/mcp/tools/get-request-insights.ts) — `get_request_insights` MCP tool
- [`request-insights-panel.tsx` (canary branch head)](https://github.com/vercel/next.js/blob/canary/packages/next/src/next-devtools/dev-overlay/components/request-insights/request-insights-panel.tsx) — new DevTools panel (canary.86)
- [`trace-viewer.ts` (canary branch head)](https://github.com/vercel/next.js/blob/canary/packages/next/src/next-devtools/dev-overlay/components/request-insights/trace-viewer.ts) — span-tree + timeline + focused/verbose split

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
  "$schema": "https://biomejs.dev/schemas/2.5.2/schema.json",
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


**Biome 2.5.2 patch (July 1, 2026, npm):** Pure patch release — no breaking changes, no config rewrite required. Adds two new options to the `useNullishCoalescing` rule:

```json
{
  "linter": {
    "rules": {
      "style": {
        "useNullishCoalescing": {
          "level": "warn",
          "options": {
            "ignoreBooleanCoalescing": true,  // skip `||` / `||=` inside `Boolean()` calls (falsy is intentional)
            "ignorePrimitives": true         // skip coalescing on primitive types (string|number|boolean|...)
          }
        }
      }
    }
  }
}
```

- **`ignoreBooleanCoalescing`** — useful if your code does `Boolean(x || defaultValue)` (the falsy shortcut is intentional, the rule shouldn't flag it).
- **`ignorePrimitives`** — useful if your code does `const name = user.name || 'Anonymous'` where `user.name` is `string | undefined`. Without this, the rule nudges you to `??` but for primitive primitives many teams prefer `||` for consistency with falsy handling.

Both options are off by default (matching 2.5.1 behaviour); enable them per-project if the default `||` → `??` rewriting produces too many false positives for your codebase. Source: [Biome 2.5.2 release](https://github.com/biomejs/biome/releases/tag/v2.5.2). Pure patch → `biome migrate --write` is NOT required for this upgrade.


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


## Next.js 16.3.0-canary.79–83 — Material PRs (July 7–10, 2026)

**Five canaries shipped between this skill's last update (July 6) and now (July 12).** The two most impactful material PRs are the canary.78 `experimental.serverComponentsHmrCancellation` activation pair (#95463 + #95486); the rest are smaller but several are worth knowing about. Detailed in the relevant topic files where indicated.

### canary.79 (July 7, 2026, tag cut 00:01Z)

Material PRs:

- **[PR #95463](https://github.com/vercel/next.js/pull/95463) `Abort superseded Server Components HMR requests on the client`** (andrewimm) — completes the canary.78 `experimental.serverComponentsHmrCancellation` flag by wiring the actual client-side cancellation. See `experimental.serverComponentsHmrCancellation` section above for full details.
- **[PR #95532](https://github.com/vercel/next.js/pull/95532) `Upgrade React from 3508aee6-20260702 to 23def8fd-20260706`** — vendor React bump. Brings 3 upstream commits (incl. devtools-facade + chrome-devtools-mcp E2E).
- **[PR #95235](https://github.com/vercel/next.js/pull/95235) `docs: remove incorrect statement that force-cache is the default`** — docs-only clarification. Force-cache is NOT the default for `'use cache'` (default is `'default-profile'`); the docs were wrong.
- **[PR #95497](https://github.com/vercel/next.js/pull/95497) `turbo-persistence: skip directory fsync on Windows`** — Turbopack persistence-layer perf fix for Windows users; avoids the directory-level fsync that costs ~50–100ms per build on Windows filesystems.
- **[PR #95521](https://github.com/vercel/next.js/pull/95521) `Disable supportsImmutableAssets with config.output`** — when `output: 'export'` / `'standalone'` is set, `supportsImmutableAssets` is now disabled automatically. Without this, immutable-asset hashing could disagree with the bundler's per-build content-hash output and serve stale assets.
- **[PR #93334](https://github.com/vercel/next.js/pull/93334) `Turbopack: add all keys to dynamic exports before sealing the object`** — Turbopack internal fix; reduces HMR/Hydration mismatches from dynamic-export key drift.
- **[PR #95446](https://github.com/vercel/next.js/pull/95446) `Also rewrite requires in page templates with Webpack`** — companion to the canary.77 Turbopack-only `.browser.ts(x)` rewrite work; Webpack page templates now get the same `.browser` resolution. Improves webpack bundle parity with Turbopack.
- **[PR #95306](https://github.com/vercel/next.js/pull/95306) `otel: Use correct parent span for Node.js middleware`** — OTel spans for Node.js middleware now nest under the request span instead of leaking as root spans. Important if you use OpenTelemetry for distributed tracing on Node.js middleware.
- **[PR #95357](https://github.com/vercel/next.js/pull/95357) `Fix instrumentation hook awaiting for middleware with adapters`** — when running on a Build Adapter (OpenNext / Cloudflare), `register()` in `instrumentation.ts` for middleware was hanging forever because the adapter didn't signal completion. Now the adapter propagates the await correctly.
- **[PR #95378](https://github.com/vercel/next.js/pull/95378) `[fragment-scroll] Enable new scroll handler by default`** — fragment-scroll handler (`#section` URL hash on the same page) is now default-on. The old handler is removed. The new handler correctly handles View Transitions + Cache Components shells.
- **[PR #95526](https://github.com/vercel/next.js/pull/95526) `fix(dev-overlay): increase fix card grid row gap to clear floating Copy prompt button`** — Instant Insights dev-overlay polish (the renamed `Copy prompt` button was overlapping the fix-card row).

### canary.80 (July 8, 2026, tag cut 00:00Z)

Material PRs:

- **[PR #95486](https://github.com/vercel/next.js/pull/95486) `Cancel a superseded Server Components HMR refresh's server-side work`** (unstubbable) — server-side counterpart to #95463. See `experimental.serverComponentsHmrCancellation` section above.
- **[PR #95547](https://github.com/vercel/next.js/pull/95547) `Run instrumentation for Node.js Middleware without Adapter`** — when running Node.js middleware on Vercel (no Build Adapter), `register()` in `instrumentation.ts` now actually runs. Previously this only ran when an adapter was present.
- **[PR #95118](https://github.com/vercel/next.js/pull/95118) `Propagate maxDuration to edge adapter outputs`** — `export const maxDuration = N` (route segment config) now correctly flows through to edge adapter builds. Before this, `maxDuration` was silently dropped on edge runtime and only respected on Node runtime.
- **[PR #95536](https://github.com/vercel/next.js/pull/95536) `Upgrade to swc 72 and rust-react-compiler`** — SWC major bump + bundles the latest rust-react-compiler into Next.js. ~20–50% build perf gain on large React apps (v0.app benchmarks). Aligns with `experimental.turbopackRustReactCompiler`.
- **[PR #95318](https://github.com/vercel/next.js/pull/95318) `Turbopack: add more context to persistence file errors`** — Turbopack persistence errors now include the path + corrupted-file size + hash mismatch info; helps diagnose `.next/cache/turbopack/` corruption.

### canary.81 (July 9, 2026, tag cut 23:59Z)

Material PRs:

- **[PR #95606](https://github.com/vercel/next.js/pull/95606) `Turbopack: enable import with {type: 'text'} by default`** — Turbopack now supports `import shader from './shader.glsl' with { type: 'text' }` (and `{type: 'json'}`, `{type: 'css'}`) out of the box. Previously this required an experimental flag. Useful for importing shaders, JSON configs, raw text assets, and CSS-as-string at build time without extra bundler config.
- **[PR #95366](https://github.com/vercel/next.js/pull/95366) `Split remaining "client-node"-only modules into .browser variants`** — extension of the canary.77 `.browser.ts(x)` work (PR #95201 + #95200). More modules are now split, more bundle-size regressions fixed. If you maintain a `.browser.tsx` sibling for a `typeof window !== 'undefined'` switch in a third-party module, check the diff to see if the sibling is now auto-detected.
- **[PR #95213](https://github.com/vercel/next.js/pull/95213) `[turbopack] Don't evict when there is little memory to save`** — Turbopack memory eviction (the canary.71 feature) was evicting aggressively even when headroom was ample. Now it skips eviction if available memory is plentiful. Reduces dev-server thrashing on memory-rich machines.
- **[PR #95137](https://github.com/vercel/next.js/pull/95137) `[turbopack] Don't persist if there is little work to do`** — Turbopack persistence (`.next/cache/turbopack/`) was persisting even on tiny incremental builds. Now skips persistence when the work-to-persist ratio is below threshold. Speeds up small `next dev` HMR cycles.
- **[PR #94916](https://github.com/vercel/next.js/pull/94916) `align dev and build output`** — Turbopack's dev and build output now agree on chunk naming + asset paths. Eliminates a class of "works in dev, 404s in build" bugs.
- **[PR #95593](https://github.com/vercel/next.js/pull/95593) `fix: log "Partial Prefetching enabled" during next build`** — informational message at `next build` startup if `experimental.partialPrefetching: true` is set; previously the message only appeared at `next dev` startup.
- **[PR #95365](https://github.com/vercel/next.js/pull/95365) `[PP] Surface URL data during prefetching as an Instant insight with rule page`** — new Instant Insights category: when a Partial Prefetching route accidentally accesses URL data (`searchParams`/`params`) before it's streamed in, the dev overlay shows a fix card with a "Rule page" link. See patterns.md → Partial Prefetching / Instant Insights for the fix workflow.
- **[PR #95582](https://github.com/vercel/next.js/pull/95582) `Consistently gate navigation-testing-lock API on flag`** — the experimental `navigation-testing-lock` API is now strictly gated behind `experimental.exposeTestingApiInProductionBuild` even in dev; previously the gate was inconsistent and the API leaked into dev in some code paths.
- **[PR #95581](https://github.com/vercel/next.js/pull/95581) `Upgrade React from 23def8fd-20260706 to 12a4baec-20260707`** — vendor React bump.
- **[PR #95538](https://github.com/vercel/next.js/pull/95538) `[turbopack] Rename rscEndpoint to rscHmrEndpoint`** — Turbopack internal rename to disambiguate the HMR RSC endpoint from regular RSC endpoints. No user-facing impact unless you have a custom dev-server proxy mapping that hardcodes `rscEndpoint`.

### canary.82 (July 10, 2026, tag cut 00:07Z)

Material PRs:

- **[PR #95554](https://github.com/vercel/next.js/pull/95554) `[turbopack] Output service workers to /_next/static/`** — Turbopack now writes compiled service workers to `/_next/static/<hash>.js` (matching Vercel's CDN conventions) instead of `/_next/`. Browser service-worker registration no longer needs a custom `Service-Worker-Allowed` header for cross-scope service workers behind a CDN. Closes the canary.68/71/81 service-worker trilogy: compile (#94920), discover (#94921), output (#95554). See setup.md → Turbopack Service Worker Support.
- **[PR #95583](https://github.com/vercel/next.js/pull/95583) `[turbopack] Compile service workers registered from pages router pages`** — completes Pages Router parity for service workers; previously only App Router pages were scanned for `navigator.serviceWorker.register()` calls.
- **[PR #95642](https://github.com/vercel/next.js/pull/95642) `Upgrade React from df4bd1b4-20260708 to 5123b063-20260708`** — vendor React bump.
- **[PR #95467](https://github.com/vercel/next.js/pull/95467) `Make the agent-rules block verifiable and self-upgrading`** + **[PR #95470](https://github.com/vercel/next.js/pull/95470) `Refresh outdated agent-rules blocks on next dev and codemod upgrade`** (aurorascharff) — the `AGENTS.md` auto-update block (the `<!-- BEGIN:nextjs-agent-rules --> ... <!-- END:nextjs-agent-rules -->` managed section written by `next dev` when an AI agent is detected) now includes a content hash. When the canary moves forward, `next dev` AND `npx @next/codemod@canary upgrade preview` detect the outdated block and rewrite it in place. No more stale AGENTS.md guidance between agent runs.
- **[PR #95892](https://github.com/vercel/next.js/pull/95892) `fix(agent-rules): pad managed AGENTS.md block with blank lines`** (aurorascharff, canary-branch ahead of canary.88 at the 1.4.65 cron run — merged 2026-07-17T12:15:52Z) — the generated `nextjs-agent-rules` block in `AGENTS.md` glued the `<!-- BEGIN:nextjs-agent-rules -->` / `<!-- END:nextjs-agent-rules -->` comment markers directly against the `# This is NOT the Next.js you know` heading and the surrounding paragraphs. Markdown formatters (Prettier, and CommonMark tooling in general) treat block-level HTML comments as needing surrounding blank lines, so any project running `prettier --write AGENTS.md` would reflow the block into the blank-padded shape — and then `next dev` would detect the block as non-current on the next start and rewrite it back to the glued form, producing churn on every commit. The fix emits blank lines around both markers so the generated block is formatter-stable. Both generators are updated together: `packages/next/src/server/lib/generate-agent-files.ts` (`next dev` writer) and `packages/create-next-app/helpers/generate-agent-files.ts` (create-next-app writer). The `next-codemod` `agents-md` refresh delegates to the installed package's generator, so it needs no change. **Existing projects get a one-time block rewrite on their next `next dev` start** — the block is described as re-added by `next dev`, so this is expected; after that it stays stable under Prettier. The `agent-rules-auto-generate` test asserts against the live generator output, so it tracks the new shape automatically. The docs sample in `ai-agents.mdx` and Next.js's own `packages/next/AGENTS.md` already use the blank-line shape, so they now match what is generated. Full diff: 4 lines added across 2 files. **Shipped in 16.3.0-canary.89** (2026-07-17T23:55:15Z).
- **[PR #95493](https://github.com/vercel/next.js/pull/95493) `Normalize and validate expire and revalidate values to handle Infinity and surface mistakes early`** — `'use cache'` directives and route-segment `revalidate` exports now validate their numeric values at parse time. Previously passing `Infinity` or a negative number was silently accepted and produced non-deterministic cache behavior. Now: hard error at build/dev time with a clear message. See patterns.md → `expire` / `revalidate` validation.
- **[PR #94859](https://github.com/vercel/next.js/pull/94859) `docs: update MCP guides for the thin next-devtools-mcp`** — the Next.js MCP server was shrunk (knowledge-base/upgrade/CC helpers REMOVED in canary.74 docs); docs now reflect the new `get_compilation_issues` / `compile_route` compilation tools and the new `next-devtools-mcp` package as the recommended install.
- **[PR #95246](https://github.com/vercel/next.js/pull/95246) `docs(opengraph-image): load assets at module scope to keep route static under Cache Components`** — pattern note: opengraph/metadata image routes under `cacheComponents: true` must load fonts/images at module scope, not inside the component. Otherwise the route is flagged as dynamic. Documented in deployment.md → Metadata Image Route Static Prerendering.

### canary.83 (July 10, 2026, tag cut 23:56Z — released ~2h before this skill update)

Material PRs:

- **[PR #95639](https://github.com/vercel/next.js/pull/95639) `(TypeScript 7 Support) Add experimental TypeScript CLI backend`** (wbinnssmith) — Next.js can now detect `typescript@7.x` in `package.json` and use its native Go compiler for `next typegen` and internal TS tooling, without you needing to install `@typescript/native-preview`. Cuts typegen latency ~5–10× on large codebases. See setup.md → TypeScript 7 Integration.
- **[PR #95607](https://github.com/vercel/next.js/pull/95607) `Keep the request body a plain Readable after middleware so Readable.toWeb() doesn't hang`** — proxy/middleware (formerly middleware) was wrapping `req.body` in a `Readable.toWeb()` conversion that could hang indefinitely on long-lived requests (SSE, large file uploads). The body stays a Node `Readable` now and is only converted at the route handler boundary. Fixes a class of silent hangs on streaming endpoints.
- **[PR #95384](https://github.com/vercel/next.js/pull/95384) `[PPF] Sync IO is only allowed in the dynamic stage`** — under Partial Prefetching (`partialPrefetching: true` / per-segment `prefetch = 'partial'` / `'unstable_eager'`), sync `fs.readFileSync` / `fs.existsSync` / `process.env.X` access during prefetch is now an error. Previously it was a warning. Forces you to either move the IO behind `'use cache'` (so it runs at build time) or mark the access as `'use cache: private'` (so it runs per-request). See patterns.md → Sync IO under Partial Prefetching.
- **[PR #95689](https://github.com/vercel/next.js/pull/95689) `Fix support for custom-media-queries in LightningCSS`** — Tailwind v4 with `@media (--breakpoint-md)` / `@custom-media` now compiles correctly under Turbopack's LightningCSS path.
- **[PR #95325](https://github.com/vercel/next.js/pull/95325) `docs: add incremental adoption path to Cache Components migration guide`** — the migration guide now has a step-by-step "enable on one route at a time" section. Useful for large codebases that can't migrate everything in one PR. See patterns.md → Cache Components Incremental Adoption.
- **[PR #95629](https://github.com/vercel/next.js/pull/95629) `Clarify AI-assisted contribution policy in PR template and AGENTS.md`** — when an AI agent opens a PR, the PR template now asks for a clear "AI-assisted-by" declaration. The `AGENTS.md` managed block now includes the policy.

### canary.86 (SHIPPED July 14, 2026, 23:31:39Z — npm `canary` dist-tag pointer moved 2026-07-14T23:58:52Z, 10 commits vs canary.85)

The 10 post-canary.85 commits in canary.86 (npm `next@canary` dist-tag, branch HEAD `4f69a3354`):

- **[PR #93978](https://github.com/vercel/next.js/pull/93978) `request insights: add DevTools request panel (5/5)`** (feedthejim) — **the 5th and final PR of the Request Insights stack** SHIPPED. Adds the new "Request Insights" panel in the dev overlay DevTools (see setup.md → `experimental.requestInsights` → "How an agent (or the DevTools panel) consumes it" for the full feature breakdown). Source: `packages/next/src/next-devtools/dev-overlay/components/request-insights/`.
- **[PR #95753](https://github.com/vercel/next.js/pull/95753) `Better support the CLI spinner when running the TSC CLI`** (lukesandberg, merged 2026-07-14T21:49:50Z) — the experimental `useTscCli` flag (which routes the Next.js type-check through the new Go-based `tsc` subprocess for ~10× faster type-check times) was rendering `Running Typescript ...` and never clearing the spinner. The normal pause/resume spinner pattern doesn't work with a subprocess, so the fix runs the spinner in the normal way and stops it as soon as the TSC subprocess produces any output. TSC flushes its output in a small sequence of writes at the end of its run, so this works as expected; if TSC ever starts streaming mid-run the spinner would never restart, but that's a fine failure mode. **User impact:** `useTscCli: true` is now a clean experience (the spinner correctly clears when type-check finishes) instead of a stuck-at-"Running Typescript ..." hang. Triggered by enabling `experimental.useTscCli: true` in `next.config.ts` on 16.3.0-canary.86+ (the flag was previously documented in setup.md → `@next/codemod upgrade --yes` for Agents / CI, where it was noted as available behind the experimental flag; canary.86 is the first canary where the spinner behaves correctly).
- **[PR #95782](https://github.com/vercel/next.js/pull/95782) `Upgrade React from 5123b063-20260708 to 7023f501-20260714`** (Vercel Release Bot, merged 2026-07-14T19:43:10Z) — vendor React canary SHA advance. The new [React canary `19.3.0-canary-7023f501-20260714`](https://www.npmjs.com/package/react?activeTag=canary) (cut 2026-07-14T16:39:18Z, replaces `5123b063-20260708`) is now in the canary.86 bundled React. Brings 2 upstream React PRs ([#37009](https://github.com/facebook/react/pulls/37009) + [#36980](https://github.com/facebook/react/pulls/36980)) — both private upstream, content not accessible via the public API. **`react@experimental` and `react@canary` npm dist-tag pointers** both moved to `7023f501-20260714` on 2026-07-14T16:40:48Z. If your `package.json` pins React to a specific canary, the canary.86 bundled React will silently win for SSR/RSC.
- **[PR #95389](https://github.com/vercel/next.js/pull/95389) `docs: add route-side URL data audit to the Partial Prefetching adoption guide`** (Janka Uryga, merged 2026-07-14T18:22:00Z) — docs only. The Partial Prefetching adoption guide now has a section on auditing which routes access `params` / `searchParams` outside `<Suspense>` and would trigger the new "link data" validation errors 1390 / 1391 / 1392 (canary.73). No code change.
- **[PR #95760](https://github.com/vercel/next.js/pull/95760) `docs: revalidateTag with expire zero, for route handlers`** (docs) — the `'use cache'` `expire: 0` error page was previously silent for Route Handlers; the page now mentions the `revalidateTag` alternative for the handler case.
- **[PR #95752](https://github.com/vercel/next.js/pull/95752) `docs: Improve immutable static docs`** (docs) — wording on immutable static assets was overly strict; clarified.
- **[PR #95620](https://github.com/vercel/next.js/pull/95620) `Add React sync development skill`** — new internal `$react-sync` skill at `.agents/skills/react-sync/` with a reusable build helper script and Corepack shim that pins pnpm to the repo version. Internal-only — the skill isn't user-facing, but Next.js contributors can now rebuild vendor React + sync into Next.js via `pnpm sync-react` without globally-pinned pnpm versions breaking workspace linking.
- **[PR #95759](https://github.com/vercel/next.js/pull/95759) `docs: useSearchParams example stray link`** (docs) — fixed a dangling doc link in the `useSearchParams` example.
- **[PR #95144](https://github.com/vercel/next.js/pull/95144) `Improve the NFT error message and ignore comment handling`** (Turbopack) — fixes #95125. The Turbopack "file-not-traced" error now (1) is specific about which function needs the `@turbopack-ignore` annotation, and (2) accepts `turbopackIgnore` comments anywhere in an expression — `fs.readFileSync(path.join(process.cwd(), someVar))` no longer requires annotations at both the `path.join` and the `fs.readFileSync` call sites; a single annotation in the expression tree is enough. Bubbles up through the linker and ignores the corresponding fs access APIs.
- **[PR #95731](https://github.com/vercel/next.js/pull/95731) `Upgrade to swc 73`** — SWC major bump (now `73.x`). No new Next.js-side behavior, but pulls in the upstream SWC perf + correctness improvements (waiting on the swc major bump in swc-project/plugins before downstream plugins can use the new APIs).
- **[PR #95725](https://github.com/vercel/next.js/pull/95725) `Update font data`** (auto-generated) — font data refresh for the next minor; no user-visible change unless you rely on a freshly-added font.

### canary.86 post-tag / canary.87 in-flight (July 15, 2026, 06:03Z — 1 commit on canary branch post-canary.86, canary.87 expected ~2026-07-15T23:00Z if 24h cadence holds)

One post-canary.86 commit on the canary branch (will be in canary.87):

- **[PR #95586](https://github.com/vercel/next.js/pull/95586) `telemetry: add agentName to anonymous metadata`** (andrewimm, merged 2026-07-14T23:56:02Z, canary-branch commit `ca414736ba` at 2026-07-14T23:56:01Z — ~25 minutes after the canary.86 tag cut at 23:31:39Z) — vendored [@vercel/detect-agent](https://github.com/vercel/vercel/tree/main/packages/detect-agent) (from the Vercel CLI) into `packages/next/src/compiled/@vercel/detect-agent/{index.js,package.json,LICENSE}` (new taskfile ncc task, externals entry, type declaration in `packages/next/types/$$compiled.internal.d.ts`) and added a new `packages/next/src/telemetry/agent-name.ts` with a memoized `getAgentName()` helper that caches the in-flight promise from `determineAgent()`. The vendored detection is async (it does a filesystem probe for Devin in addition to env-var checks like `CLAUDE_CODE` / `CURSOR_TRACE_ID` / `GITHUB_COPILOT_CHAT`) and potentially slower than the existing log-only path, so it's wrapped in a memoized helper: `getAgentName()` caches the promise from `determineAgent()`, the first call runs detection once, concurrent callers share that in-flight promise, and every later call resolves against the already-settled result. The agent doesn't change over the process lifetime so this is safe; detection failures resolve to `null` so telemetry never throws on this path. `getAnonymousMeta()` is now `async` and includes `agentName: await getAgentName()` alongside the existing `ciName` / `nextVersion` / `isCI` fields. `storage.ts` awaits it on the send path (the `record` method was already async). **`NEXT_TELEMETRY_DEBUG=1` now prints the meta block alongside each event** so the full payload — including `agentName` — is observable locally without sending anything. This meta block was not output before, so it was harder to debug these fields. The PR also **consolidates a second, synchronous, hand-vendored copy of agent detection that lived elsewhere** in the package (replaces `telemetry/detect-agent.ts` with the unified `telemetry/agent-name.ts` path) and re-uses it in `packages/next/src/server/lib/{app-info-log.ts, start-server.ts}` for the per-request anonymous-meta block. **Why this matters for skill users:** Next.js now has anonymous visibility into how much of its development happens under AI coding agents (claude / cursor / codex / devin / etc.) — the data is sent only when the user has opted into telemetry. Nothing in user code changes. The new field appears in the `NEXT_TELEMETRY_DEBUG=1` output of every event once a user has installed a canary.87+ build. Files touched: `packages/next/package.json` (1 line, ncc alias) + `packages/next/src/compiled/@vercel/detect-agent/{LICENSE,index.js,package.json}` (vendored) + `packages/next/src/telemetry/{agent-name.ts (+25/-0), anonymous-meta.ts (+4/-1), detect-agent.ts (-83), storage.ts (+6/-2)}` + `packages/next/src/server/lib/{app-info-log.ts (+5/-3), start-server.ts (+1/-1)}` + `packages/next/taskfile.js (+12/-0)` + `packages/next/types/$$compiled.internal.d.ts (+4/-0)` + `pnpm-lock.yaml (+9/-0)` + 2 test files updated. Will be in canary.87.

**canary.85 (July 13, 2026) details** are unchanged; see canary.85 section below for the full material-PRs list and the v16.3.0-preview.6 preview that ran between canary.84 and canary.85.
### canary.87 — Turbopack chunk-group-bootstrap stack + 16 other PRs SHIPPED (July 15, 2026, 23:59:48Z)

canary.87 details are recorded in the 1.4.62 cron entry above (SKILL.md description → "Next.js 16.3.0-canary.87 SHIPPED"). Headline: 17 commits between canary.86 tag cut (2026-07-14T23:31:39Z) and canary.87 tag cut. Notable: the 6 in-flight PRs predicted by the 1.4.61 cron all shipped (PR #95833 App Shells stale < 5 min exclusion, PR #94903 `@next/routing` stable, PR #95738 NEXT_HASH_SALT server-side, PR #95825 AGENTS.md next.config.js options warning, PR #95682 same-document-traversal replay, PR #95586 telemetry agentName). Plus 11 NEW material PRs the previous cron missed (PR #95799 export async-init errors no longer unhandled rejections, PR #95790 top-level await in metadata routes awaited correctly, PR #95789 modules exporting a .then no longer hang module tracking, PR #95791 async userland loading is now a state machine, PR #95794 Request Insights subscription initialises correctly, PR #95788 debug-build-paths now matches metadata routes, PR #95813 stale build warning removed, PR #95694 Turbopack AutoMap/AutoSet inline TinyVec, PR #95261 Turbopack component chunks for each merged group, PR #95693 docs: cssChunking graph option, plus PR #95824 NFT trace regression fix which actually landed in canary.88 not canary.87). The 1.4.62 cron recorded `api.md` + `deployment.md` + `performance.md` sections; canary.87 also shipped the per-segment `prefetch = 'partial'` / `'unstable_eager'` mode that complements the canary.88 partialPrefetching flag.

### canary.88 — Partial Prefetching unification + 16 PRs SHIPPED (July 16, 2026, 23:54:24Z)

canary.88 details are recorded in the 1.4.63 + 1.4.64 cron entries above (SKILL.md description → "Next.js 16.3.0-canary.88 SHIPPED"). Headline: 16 commits between canary.87 and canary.88. Major: **PR #95415 "Unify appShells flag with Partial Prefetching"** by acdlite (343+/-381 over 45 files) — the `experimental.appShells` config flag is REMOVED entirely from `config-schema.ts` + `config-shared.ts` + `base-server.ts` + the `app-page` + `edge-ssr-app` templates; behaviors previously gated by it split into (a) pure optimizations that ship unconditionally and (b) behaviors that change server-request counts now gated behind `experimental.partialPrefetching: true` (or per-segment `prefetch = 'partial'` / `'unstable_eager'`). Anyone who set `experimental.appShells: true` from older canary docs should now delete it (the flag is no longer in the experimental schema and Next.js 16.3+ will warn about the unrecognized config key). Plus PR #95824 NFT trace regression fix for serverExternalPackages + Server Actions (the 1.4.63 cron documented the workaround, it's now official). Plus the 6-PR Turbopack chunk-group-bootstrap stack (#94586 + #94631 + #94664 + #94663 + #94666 + #94671) drops the per-route bootstrap module and inlines it into Next.js itself — the chunk-group-bootstrap stack has implications for `output: 'standalone'` deploys (manifest shape change) and `output: 'export'` builds (build-output change).

### canary.89 (SHIPPED July 17, 2026, 23:55:15Z — npm `canary` dist-tag pointer moved 2026-07-17T23:55:15Z, 10 commits vs canary.88)

The 10 commits in canary.89 (branch HEAD `0491db0` — **identical to the canary.89 tag as of 2026-07-18T12:07Z**, meaning 0 commits ahead, no in-flight for canary.90 yet):

- **[PR #95539](https://github.com/vercel/next.js/pull/95539) `[turbopack] Don't SSR on pages only navigated to through a soft nav`** (sampoder, merged 2026-07-17T20:10:55Z, 460+/-89 across 23 files) — the single biggest user-facing PR in canary.89. Turbopack dev previously compiled the Client Component SSR chunk for every HTML page endpoint registered in the App Router, regardless of whether that page was ever rendered as a standalone document. Most app pages today are reached via an RSC-only soft navigation — a page rendered inside a modal, a tabbed sub-route, a `Link` with `prefetch={false}` from a parent page, or a route only ever visible as a child of a layout that hydrates in place. In those cases the SSR chunk is dead weight: the page is never sent as a document, only as RSC payload fragments. **The fix:** new `crates/next-api/src/output_mode.rs` (`OutputModeState` + `SsrMarkTarget` + `OptionSsrMarkTarget` + `OptionEndpoint`) plus a new `mark_as_ssr_operation` turbo-tasks operation that walks the dev session's `OutputModeState` to decide which HTML page endpoints need SSR compilation; pages only reached via soft nav get `process_ssr = false` (so `ssr_chunking_context` collapses from a separate upcast into a `process_ssr.then_some(server_chunking_context)` no-op) and `rscModuleMapping` for the App Router is now only emitted for pages (route handlers + metadata routes drop it). **User impact:** materially faster `next dev` first-request compile and lower dev memory for apps with many pages reached mostly via RSC soft navigation; effect is **dev-only** — production SSR runs compile every HTML page endpoint as before because `OutputModeState` is dev-session-scoped (`mark_as_ssr_operation` `bail!`s outside a dev session). **Caveat:** the new behaviour is gated on dev sessions that exercise the soft-nav path; the new Turbopack runtime log section (exposed by #95908's cleanup) shows per-endpoint SSR-build decisions. Full perf breakdown in performance.md → "Turbopack Dev: Skip SSR Compile for Pages Only Navigated to Through Soft Navs".

- **[PR #95907](https://github.com/vercel/next.js/pull/95907) `Package skills/ as a Claude Code plugin`** (gaojude, merged 2026-07-17T19:46:14Z, 36 lines across 2 files) — Next.js now ships its 4 dev-knowledge skills (`nextjs:next-cache-components-adoption`, `nextjs:next-cache-components-optimizer`, `nextjs:next-dev-loop`, `nextjs:next-partial-prefetching-adoption`) under `skills/` as a Claude Code plugin. New files: `skills/.claude-plugin/plugin.json` with `"skills": ["./"]` so the four skills at `skills/*/SKILL.md` are discovered in place, plus a root `.claude-plugin/marketplace.json` pointing its single entry at `./skills`. PR #94877 had previously removed the embedded plugin marketplace because its plugin carried copies of skills that drifted from `skills/`; this PR re-adds plugin packaging WITHOUT duplicating any content. External catalogs can reference the plugin with a `git-subdir` source (`path: "skills"`) which sparse-clones only that directory instead of the whole repo. **`claude plugin validate` passes on both manifests**; `claude --plugin-dir ./skills` discovers all four skills. **Meta-relevant for skill authors:** the `skills/.claude-plugin/plugin.json` + `git-subdir` marketplace pattern is the canonical structure for shipping a Claude Code plugin that contains knowledge skills — this is the same pattern the `frontend-skill` repo (this skill) uses to be loadable as a plugin.

- **[PR #95908](https://github.com/vercel/next.js/pull/95908) `[turbopack] Clean up server_chunking_context`** (sampoder, merged 2026-07-17T21:44:27Z, +25/-11, 1 file) — direct follow-up to #95539; removes the now-unused `server_chunking_context` helper and consolidates around the unified chunking context passed through `mark_as_ssr_operation`.

- **[PR #95909](https://github.com/vercel/next.js/pull/95909) `Fix Rust doctest failures`** (marcoshernanz, merged 2026-07-17T22:39:16Z, +54/-40, 11 files) — marks incomplete illustrative doc-snippets as `ignore` so `cargo test --workspace --doc` passes cleanly. The previously-failing doctests in `crates/next-api` + `crates/next-core` now compile or are explicitly skipped, so the public docs match the actual API surface (useful if you read the Next.js source to understand a behaviour — the Rust API documentation is now reliable again).

- **[PR #95847](https://github.com/vercel/next.js/pull/95847) `[ci] Avoid apt-get in the next-stats-action Docker image`** (mischnic, merged 2026-07-17T17:45:36Z) — CI-only fix; see the 1.4.65 entry above for full details.

- **[PR #95865](https://github.com/vercel/next.js/pull/95865) `Back/forward set the Nav Inspector back to pending`** (acdlite, merged 2026-07-17T15:44:05Z) — dev-only Nav Inspector fix; full details in the 1.4.65 entry above and performance.md → "Navigation Inspector — Back/Forward Resets to Pending".

- **[PR #95892](https://github.com/vercel/next.js/pull/95892) `fix(agent-rules): pad managed AGENTS.md block with blank lines`** (aurorascharff, merged 2026-07-17T12:15:52Z) — Prettier-stable AGENTS.md block; full details in the 1.4.65 entry above.

- **[PR #95884](https://github.com/vercel/next.js/pull/95884) `Revert "Run more test suites under cacheComponents flag"`** (jquense, merged 2026-07-17T00:33:27Z) — CI-only revert; no user-facing impact.

- **[PR #95864](https://github.com/vercel/next.js/pull/95864) `Fix repeated navigations while the Instant Navigation lock is held`** (acdlite, merged 2026-07-17T00:19:58Z) — dev-only Nav Inspector lock-handling fix; full details in the 1.4.65 entry above.

- **canary.89 tag commit** `0491db0` (2026-07-17T23:55:15Z) — official tag cut.

**Headline user-facing change: PR #95539 (Turbopack skip-SSR-for-soft-nav pages).** This is a meaningful dev-loop win for any app with many pages reached mostly via RSC soft navigation — single-page apps with deeply-nested tab routes, admin panels, dashboards with modal routes, multi-step forms. For an app with 50 pages where 40 are reached only via soft nav, expect the first `next dev` request that touches a previously-uncompiled soft-nav-only page to skip the SSR build entirely. Production SSR is unchanged.

**`latest canary: 16.3.0-canary.89`** (npm `canary` dist-tag pointer moved 2026-07-17T23:55:15Z; canary.90 expected ~2026-07-18T23:00Z on the 24h cadence).

### canary.90 (SHIPPED 2026-07-19T23:34:16Z — npm `canary` dist-tag pointer *pending move* as of 2026-07-20T00:03Z; 2 commits vs canary.89, 0 commits ahead of canary.90 on canary branch)

canary.90 was tag-cut on GitHub at `163e45e` (2026-07-19T23:34:16Z, canary-branch status: "identical" per GitHub `compare` API at 2026-07-20T00:03Z — meaning canary-branch HEAD equals the canary.90 tag with 0 in-flight commits for canary.91), but the npm `canary` dist-tag pointer had not yet moved from canary.89 at this cron's run time — the dist-tag move is typically a few minutes to a few hours after the tag cut (the bot npm-publishes asynchronously). Verify with `npm view next dist-tags.canary`.

- **[PR #95901](https://github.com/vercel/next.js/pull/95901) `Upgrade React from `7023f501-20260714` to `172742b4-20260716`** (Vercel Release Bot, merged 2026-07-19T23:23:55Z) — vendor React canary SHA advance, the ONLY material PR in canary.90. The new React canary [19.3.0-canary-172742b4-20260716](https://www.npmjs.com/package/react?activeTab=canary) (cut 2026-07-17T16:35:51Z, replaces `7023f501-20260714`) is now the React that ships INSIDE `next@canary`'s bundled React (the `react.production.js` cjs file under `packages/next/src/compiled/react/`). Brings the two React canary PRs that were the headline of the 1.4.68 cron — [PR #37030](https://github.com/facebook/react/pull/37030) `[Fiber] Fix false-positive hydration mismatch on nonce attributes` (MaxwellCohen, merged 2026-07-16T16:54:10Z; silences the spurious red-box hydration error on every App Router page that uses CSP `script-src 'nonce-...'` — the canonical strict-CSP pattern) AND [PR #37039](https://github.com/facebook/react/pull/37039) `Enable enableEffectEventMutationPhase everywhere` (hoxyq, merged 2026-07-16T13:21:39Z; small `useEffectEvent` perf win + bug fix for users who combine `useEffectEvent` with View Transitions) — into the React that Next.js canary bundles for SSR/RSC. **`react@experimental` and `react@canary` npm dist-tag pointers** both moved to `172742b4-20260716` on 2026-07-17 (the upstream React canary was published 2 days before canary.90 picked it up — same gap as previous vendor bumps: 1–2 days latency).

- **canary.90 tag commit** `163e45e` (2026-07-19T23:34:16Z) — official tag cut.

**Why this matters for skill users:**

1. **For `next@canary` users on App Router + strict CSP with nonces:** you no longer need to separately `npm install react@canary@172742b4-20260716` to get React PR #37030's fix. canary.90's bundled React includes it, so the silent dev-mode red-box hydration error on `script-src 'nonce-...'` pages is silenced automatically on `next@canary@90`+. The existing `security.md` recommendation ("upgrade React to `19.3.0-canary-172742b4-20260716`+") now applies by default to anyone on `next@canary@90`+ — no separate React pin needed. (Until React 19.3 stable ships, `next@canary` is the only path; the 19.3 stable date has not been announced.)
2. **For `next@canary` users on `useEffectEvent`:** the PR #37039 mutator-phase fix activates automatically. No code change required.
3. **For teams still on stable `next@16.2.x`:** nothing changes from the canary perspective — the vendor React bump only applies to `next@canary`. The *vendored* React inside `next@16.2.11` (shipped 2026-07-21T16:58:28Z) is still `19.2.7`. **However, React stable `19.2.8` shipped 2026-07-21T15:49:09Z** ([GitHub release](https://github.com/facebook/react/releases/tag/v19.2.8)) — two commits past `19.2.7`: `[FlightReply] Performance improvements when decoding` PR #37087 (the same perf win that landed in React canary `81e442ea-20260721` via the earlier-backported PR #37090, now back-ported to the 19.2.x stable line) + `[19.2.x] Update required references to GitHub repo` PR #36753 (housekeeping). **`npm install react@19.2.8 react-dom@19.2.8` after upgrading `next@16.2.11` will pick up the FlightReply decode perf** because Next.js only vendors the SSR/RSC React (`packages/next/src/compiled/react/`) — the client hydration React is loaded from your `node_modules/react`. So on `next@16.2.11` + `react@19.2.8`, the FlightReply perf win is realised for client hydration immediately, while SSR/RSC stays on the vendored `19.2.7` until `next@16.2.12` (or `next@16.3.0` stable) ships. The next stable vendor React bump is targeted at `next@16.3.0` (React 19.3 stable on the 16.3 line) — no separate stable `next` security-release bump is needed for React, since the React line is independently versioned.

**Coordination note with the July 20, 2026 Next.js Security Release:** Vercel [announced July 13, 2026](https://nextjs.org/blog/next-security-release-program) the first scheduled monthly security release for **today** (Monday, July 20, 2026). As of this cron's run time (2026-07-20T00:03Z, ~17 hours before the typical Vercel release window of ~17:00 UTC for North-America-business-hours deployment), the npm `latest` tag is still `16.2.10` and no stable 16.2.x patch has shipped — Vercel has not yet published `16.2.11` or `15.5.21`. The npm `canary` dist-tag pointer is still `16.3.0-canary.89` (dist-tag move is delayed — typical 15min–6h lag after tag cut). **Producers waiting for the security release should keep polling `npm view next dist-tags.latest`** (Vercel does not always publish to a predictable GitHub Release; the npm dist-tag is the canonical signal); the security detail and CVE IDs will publish the same day per [Vercel's program blog](https://nextjs.org/blog/next-security-release-program). `security.md` has the full pre-July-20 prep checklist (the T-24h block is still valid).

**`latest canary: 16.3.0-canary.90`** (npm `canary` dist-tag pointer movement *pending* at 2026-07-20T00:03Z; tag cut at `163e45e` 2026-07-19T23:34:16Z — verify with `npm view next dist-tags.canary`; canary.91 expected ~2026-07-20T23:00Z on the 24h cadence, though the 24h cadence has slipped to 36–48h multiple times in the last week).

### canary.91 (SHIPPED 2026-07-20T23:58:30Z — npm `canary` dist-tag pointer moved 2026-07-20T23:58:15Z, ~5 minutes before this cron at 00:03Z July 21; 9 PRs vs canary.90, 0 commits ahead of canary.91 on canary branch — clean tag-cut)

canary.91 was tag-cut on GitHub at `4dea35db` (2026-07-20T23:58:30Z, canary-branch HEAD identical to canary.91 tag = 0 commits ahead for canary.92 per GitHub `compare` API at 2026-07-21T00:03Z). npm `canary` dist-tag pointer moved 2026-07-20T23:58:15Z — only **15 seconds** before the tag was cut on GitHub (the npm publish went out first this time, the reverse of the usual order). **9 PRs in the release body** — the largest canary by PR count since canary.87 (17 PRs).

**Note on the day-push:** this canary shipped on **2026-07-20**, the same day Vercel pushed the July security release to July 21. The canary train was unaffected by the security-release slip (the canary branch is owned by the core team; the security release is owned by the security team — orthogonal workstreams). Per the [Next.js Security Release Program](https://nextjs.org/blog/next-security-release-program) blog banner (added July 20, 2026), the July security release is now expected on **July 21, 2026**; see `security.md` for the live status block.

**canary.91 PR-by-PR breakdown (per the [GitHub release body](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.91)):**

#### Material user-facing PRs (4 — all already documented in earlier crons, but tracked here for completeness)

1. **[PR #95761](https://github.com/vercel/next.js/pull/95761) `[instant] Let dev-server requests bypass the fetch lock`** (eps1lon, merged 2026-07-20T10:02:38Z, commit `094dccb25f`) — fully documented in `performance.md` → "Navigation Inspector — Dev-Server Requests Bypass the Fetch Lock" (1.4.72 entry). The Instant Navigation Testing API's fetch-lock shim now exempts same-origin requests to `/__nextjs_`-prefixed paths (hot-reloader middleware endpoints like `__nextjs_original-stack-frames`, `__nextjs_source-map`, `__nextjs_launch-editor`, `__nextjs_request-insights`). **Impact for dev users:** error overlay stack frames, source maps, and `launch-editor` actions now resolve mid-scope. **Verify recipe included** in the performance.md section.

2. **[PR #95351](https://github.com/vercel/next.js/pull/95351) `Move immutable static assets config option out of experimental`** (mischnic, merged 2026-07-20T13:35:15Z, closes NAR-868) — fully documented in `deployment.md` → "`experimental.outputHashSalt` and `experimental.supportsImmutableAssets` Promoted Out of `experimental`". **`experimental.supportsImmutableAssets` is now a stable top-level `next.config.ts` key** (with a backward-compat `experimental.*` alias for one minor version). Audit recipe: `rg "experimental\.supportsImmutableAssets" next.config.*`.

3. **[PR #95840](https://github.com/vercel/next.js/pull/95840) `Move outputHashSalt out of experimental`** (mischnic, merged 2026-07-20T14:34:59Z) — fully documented in `deployment.md` → same section as #95351. **`experimental.outputHashSalt` is now a stable top-level `next.config.ts` key**. The CLI / env-var counterpart (`NEXT_HASH_SALT`) is unchanged. Both promote the same shape (`experimental.X` → top-level `X`) — they shipped ~1 hour apart on 2026-07-20 (13:35Z and 14:34Z).

4. **[PR #95631](https://github.com/vercel/next.js/pull/95631) `fix(cms-contentful): await draftMode() and use Promise<> params type`** (manoraj, merged 2026-07-20T16:18:39Z) — fully documented in `security.md` → "July 20, 2026 — T+1h Live Status Update" (the "fixes the Contentful CMS example" item under the canary-branch-4-ahead list) + `deployment.md`. The canonical Next.js Contentful CMS example now uses the async `next/headers` + `Promise<params>` APIs introduced in Next.js 15. **Action:** if your project forked from the legacy example and still uses the synchronous patterns, `await draftMode()` and update `params` to `Promise<{ slug: string }>`. See the [canonical Contentful example](https://github.com/vercel/next.js/tree/canary/examples/cms-contentful) for the diff.

#### NEW PRs in canary.91 (5 — the in-flight from the canary-branch-ahead-of-canary.90 window the 1.4.73 cron did not document because they were still on canary branch at that time)

5. **[PR #95903](https://github.com/vercel/next.js/pull/95903) `[turbopack] Skip redundant top-level root updates`** (sampoder + bgw) — Turbopack dev was emitting redundant root-graph updates when no source files had changed between updates (every HMR tick triggered a full root-graph diff even when there were no actual edits). The fix in `crates/turbopack/src/lib.rs` (small, ~30 lines) tracks the last-observed root set and only emits an update event when the set actually changed. **Impact:** materially less CPU + less HMR chatter on `next dev` between file edits; especially noticeable in large monorepos where the root graph contains hundreds of files and the diff used to be expensive. **Detail in performance.md** (new section added in 1.4.74 — "Turbopack Dev: Skip Redundant Top-Level Root Updates (16.3.0-canary.91, PR #95903)").

6. **[PR #95716](https://github.com/vercel/next.js/pull/95716) `[turbopack] Drop unused exports from a CJS module`** (bgw) — Turbopack was preserving every `module.exports.X` binding on a CJS module even when downstream consumers (the bundler) only read one or two of them. The fix in `crates/turbopack-ecmascript/src/references/esm/module_id.rs` + the CJS export analyzer determines the actual live export set via reference analysis and prunes the rest. **Impact:** small bundle-size win on apps that import from CJS packages with large `module.exports` (e.g. `aws-sdk`, some legacy Node utilities) — typically 2–8 KB per imported CJS module on a hot path. **No code change required for users.** Detail in `performance.md` (new section — "Turbopack CJS Export Pruning (16.3.0-canary.91, PR #95716)").

7. **[PR #95835](https://github.com/vercel/next.js/pull/95835) `Turbopack: Simplify parent directory creation retry loop logic`** (bgw) — Turbopack's filesystem writer had a hand-rolled retry loop for parent-directory creation (3 retries, exponential backoff, edge-case handling for ENOENT vs EEXIST). The fix in `crates/turbopack/src/lib.rs` + `crates/turbopack-fs/src/lib.rs` replaces the loop with a single `fs::create_dir_all`-equivalent (Rust's built-in recursive create) — fewer allocations, simpler error semantics, and one fewer source of the "directory already exists, retrying" warning that previously cluttered dev output. **Impact:** marginally faster `next build` on cold caches (fewer syscall round-trips per output directory) + cleaner dev logs. **Detail in performance.md** (new section — "Turbopack Parent Directory Creation Simplification (16.3.0-canary.91, PR #95835)").

8. **[PR #95911](https://github.com/vercel/next.js/pull/95911) `Run Rust doctests in CI`** (marcoshernanz) — the Rust crate workspace was not running `cargo test --doc` in CI; doctests in `crates/next-api/` + `crates/next-core/` + `crates/turbopack-*` would rot silently. PR #95909 (already in canary.89, fully documented) marked incomplete illustrative doc-snippets as `ignore` so doctests can now compile cleanly; this PR wires the test runner into CI. **Impact:** zero user-facing change; affects only Rust contributors and skill authors reading the source. CI-only.

9. **[PR #95928](https://github.com/vercel/next.js/pull/95928) `docs: add 'Set up your editor' to the installation guide`** (aurorascharff) — docs-only: the `installation` guide at `docs/01-app/01-getting-started/01-installation.mdx` gained a new "Set up your editor" section with VS Code / Cursor / WebStorm / Zed / Vim configuration snippets (recommended extensions + settings for TypeScript, Tailwind, ESLint, and the App Router). **Impact:** zero code change; makes the installation guide a one-stop-shop for new projects.

**What this canary enables in practice**

- **Dev users:** PR #95903 + PR #95835 make `next dev` materially quieter between edits (less HMR chatter, no more "directory already exists, retrying" warnings); PR #95716 makes builds slightly smaller on CJS-heavy projects. All three are zero-config — just upgrade.
- **Anyone on App Router with strict CSP + nonces:** no change vs canary.90; React PR #37030 nonce fix is already bundled.
- **Anyone on `@next/playwright` `instant()`:** no change vs canary.90 documentation; PR #95761 is now live and resolves the dev-overlay-source-map-launch-editor mid-scope issue.
- **Anyone on `experimental.supportsImmutableAssets` or `experimental.outputHashSalt`:** these keys are now stable. Audit your `next.config.*` and migrate from `experimental.*` to top-level; the `experimental.*` alias will be removed in Next.js 17.0.
- **Anyone with a Contentful CMS fork from the legacy example:** upgrade to canary.91 and apply the async `draftMode()` + `Promise<params>` pattern from the [canonical example](https://github.com/vercel/next.js/tree/canary/examples/cms-contentful).

**Latest canary: 16.3.0-canary.91** (npm `canary` dist-tag pointer moved 2026-07-20T23:58:15Z; tag SHA `4dea35dbd218deb12ecca36d7aedd760fe17b923`; canary.92 expected ~2026-07-21T23:00Z on the 24h cadence, though the cadence has slipped to 36–48h multiple times in the last week — and the security release today could absorb some of that capacity and push canary.92 a day later).

**Sources:**
- [canary.91 release notes](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.91)
- [compare v16.3.0-canary.90...v16.3.0-canary.91](https://github.com/vercel/next.js/compare/v16.3.0-canary.90...v16.3.0-canary.91)
- [compare v16.3.0-canary.91...canary (identical, 0 commits ahead)](https://github.com/vercel/next.js/compare/v16.3.0-canary.91...canary)
- [PR #95761 `[instant] Let dev-server requests bypass the fetch lock`](https://github.com/vercel/next.js/pull/95761)
- [PR #95351 `Move immutable static assets config option out of experimental`](https://github.com/vercel/next.js/pull/95351)
- [PR #95840 `Move outputHashSalt out of experimental`](https://github.com/vercel/next.js/pull/95840)
- [PR #95631 `fix(cms-contentful): await draftMode() and use Promise<> params type`](https://github.com/vercel/next.js/pull/95631)
- [PR #95903 `[turbopack] Skip redundant top-level root updates`](https://github.com/vercel/next.js/pull/95903)
- [PR #95716 `[turbopack] Drop unused exports from a CJS module`](https://github.com/vercel/next.js/pull/95716)
- [PR #95835 `Turbopack: Simplify parent directory creation retry loop logic`](https://github.com/vercel/next.js/pull/95835)
- [PR #95911 `Run Rust doctests in CI`](https://github.com/vercel/next.js/pull/95911)
- [PR #95928 `docs: add 'Set up your editor' to the installation guide`](https://github.com/vercel/next.js/pull/95928)

### Next.js dev-knowledge skills shipped as a Claude Code plugin (16.3.0-canary.89, [PR #95907](https://github.com/vercel/next.js/pull/95907))

Next.js now ships four dev-knowledge skills under its `skills/` directory as a loadable Claude Code plugin. This is meta-relevant for anyone maintaining a similar skill repo for their team: the canonical pattern (as of canary.89) is **`skills/.claude-plugin/plugin.json` + a root `.claude-plugin/marketplace.json` + `git-subdir` sources in marketplace catalogs**.

**The four shipped skills:**

1. **`nextjs:next-cache-components-adoption`** — guides incremental Cache Components enablement on a route-by-route basis. Matches the 1.4.40 entry in patterns.md → Cache Components Incremental Adoption.
2. **`nextjs:next-cache-components-optimizer`** — optimizer skill for `cacheLife` / `cacheTag` choices and `'use cache'` placement.
3. **`nextjs:next-dev-loop`** — Next.js dev-loop papercut fixes from the canary.71+ ergonomics roll-up.
4. **`nextjs:next-partial-prefetching-adoption`** — adoption guide for the new `experimental.partialPrefetching` / `prefetch = 'partial'` / `prefetch = 'unstable_eager'` mode (the canary.88 unification).

**Plugin structure (the canonical pattern):**

```
vercel/next.js/
├── skills/
│   ├── .claude-plugin/
│   │   └── plugin.json           # { "skills": ["./"] } — discover skills at skills/*/SKILL.md
│   ├── next-cache-components-adoption/
│   │   └── SKILL.md
│   ├── next-cache-components-optimizer/
│   │   └── SKILL.md
│   ├── next-dev-loop/
│   │   └── SKILL.md
│   └── next-partial-prefetching-adoption/
│       └── SKILL.md
└── .claude-plugin/
    └── marketplace.json          # single entry pointing at ./skills
```

**marketplace.json example:**

```json
{
  "plugins": [
    {
      "name": "nextjs",
      "source": { "source": "git-subdir", "url": "https://github.com/vercel/next.js", "path": "skills" }
    }
  ]
}
```

The `git-subdir` source means consumers' catalog tools fetch ONLY the `skills/` directory (sparse checkout), not the entire Next.js repo. This keeps the plugin install lightweight (~100KB instead of ~200MB).

**Why this matters for skill authors:** as of canary.89 the canonical "ship a Claude Code plugin that contains knowledge skills" structure is exactly this — `skills/.claude-plugin/plugin.json` with `"skills": ["./"]` + a marketplace.json that points at `./skills` with a git-subdir source. If you maintain a team-internal skill repo (like this `frontend-skill`), the canary.89 PR is the reference implementation.

**Source:** [PR #95907 — `Package skills/ as a Claude Code plugin`](https://github.com/vercel/next.js/pull/95907) — Files: `skills/.claude-plugin/plugin.json` (new, 5 lines), `.claude-plugin/marketplace.json` (new, 31 lines) — gaojude — merged 2026-07-17T19:46:14Z — **Shipped in 16.3.0-canary.89** (npm `canary` dist-tag pointer moved 2026-07-17T23:55:15Z).


### canary.84 (July 12, 2026, tag cut 23:57:18Z — published ~5 min before this skill update)

**Headline (historical, July 12, 2026):** ships the first 3 PRs of a 5-PR **Request Insights** dev-diagnostics stack by @feedthejim (1/5 local recording, 2/5 request history, 3/5 dev-snapshot transport). At the time, the remaining 2 PRs (4/5 agent diagnosis access #93977, 5/5 DevTools request panel #93978) were still open. **Status now (July 14, 2026):** 4/5 shipped in canary.85 (July 13, 2026, 12:01:53Z) — `next experimental-request-insights` CLI + `get_request_insights` MCP tool; 5/5 merged on canary branch 2026-07-14T11:26:19Z and shipped in canary.86 (2026-07-14). Branch advanced 2 days after canary.83 — back on the ~48h cadence. canary-branch HEAD `a7bf1202de`, 4 commits vs canary.83 (3 PRs + the tag itself).

Material PRs:

- **[PR #93974](https://github.com/vercel/next.js/pull/93974) `request insights: record local framework spans (1/5)`** (feedthejim) — dev-only `LocalSpanRecorder` + `LocalRecordingSpan` (OTEL `Span` API-compatible) that mirrors framework spans into an in-memory store when no user OTEL provider is installed. Hex-padded synthetic trace/span IDs. Recorder stored on `globalThis[Symbol.for('@next/local-span-recorder')]`; `tracer.ts` consults it on every span start/end. Preserves existing OTEL behavior when a user provider is installed (no double-emission). 5 files, +485/-0.
- **[PR #93975](https://github.com/vercel/next.js/pull/93975) `request insights: derive request history and fetch data (2/5)`** (feedthejim) — wraps the recorder in a bounded `InMemoryRequestInsightsStore` (`MAX_REQUEST_INSIGHTS = 100`) keyed by `requestId`. Dedupes fetch records across completed `AppRender.fetch` spans vs direct fetch metrics. Sanitizes sensitive attribute keys via a `SENSITIVE_PARAM_NAME_RE` (matches `*token*` / `*secret*` / `*key*` / `*password*` / `*auth*` / `*signature*` / `*jwt*` / etc — case-insensitive) by replacing them with `'redacted'`; only the keys in `SAFE_SPAN_ATTRIBUTE_KEYS` (`http.method`, `http.route`, `http.status_code`, `http.url`, `next.fetch.cache_reason`, `next.fetch.cache_status`, `next.route`, `next.rsc`, `next.segment`, `next.span_name`, `next.span_type`, etc) pass through. URL query strings are sanitized via `sanitizeUrl()` before snapshotting. No raw header values, no auth headers, no bodies. 7 files, +747/-0.
- **[PR #93976](https://github.com/vercel/next.js/pull/93976) `request insights: expose dev snapshots to tools and HMR (3/5)`** (feedthejim) — wires the new `experimental.requestInsights?: boolean` config flag (default `false`), the private dev endpoint `GET /_next/development/request-insights` (returns 404-with-helpful-error JSON when off; CSRF-gated via `blockCrossSiteDEV` + `allowedDevOrigins` when on), and the HMR transport `HMR_MESSAGE_SENT_TO_BROWSER.REQUEST_INSIGHTS_UPDATE` (`'requestInsightsUpdate'`) so tools/browser clients can subscribe. Two new reducer action types in `dev-overlay/shared.ts`: `ACTION_REQUEST_INSIGHTS_SNAPSHOT` + `ACTION_REQUEST_INSIGHTS_UPDATE`. `DevBundlerService` subscribes on construction; `unsub` is wired into `close()`. 12 files, +204/-1.
- **[PR #93977](https://github.com/vercel/next.js/pull/93977) `request insights: add agent diagnosis access (4/5)`** — **SHIPPED in canary.85 (July 13, 2026)**. Adds (1) the `get_request_insights` MCP tool — snapshot of last 100 requests, accepts `{ requestId?, htmlRequestId? }` filter args, and (2) the new `next experimental-request-insights` CLI command. The `subscribe_to_request_insights` MCP tool originally announced in the PR description was **deferred** — the live stream is already exposed via the canary.84 HMR transport `HMR_MESSAGE_SENT_TO_BROWSER.REQUEST_INSIGHTS_UPDATE` for browser subscribers, and MCP clients re-invoke `get_request_insights` on a short interval to approximate streaming. See v16.3.0-preview.6 + canary.85 section below for the full agent-loop recipe.
- **[PR #93978](https://github.com/vercel/next.js/pull/93978) `request insights: add DevTools request panel (5/5)`** — **SHIPPED on canary branch 2026-07-14T11:26:19Z**, shipped in canary.86 (2026-07-14). Adds a "Request Insights" panel to the dev-overlay DevTools: a retained request list with stable selection, a "Page load" marker on the initial document request, an end-to-end trace timeline that follows the recorded span parent/child hierarchy, a focused default view + verbose toggle for deeper investigation, fetch + cache + request + status + duration + span-count details, and merged fetch-metric + `AppRender.fetch`-span display. Source: `packages/next/src/next-devtools/dev-overlay/components/request-insights/{request-insights-panel.tsx, request-list.ts, trace-viewer.ts, request-insights-panel.css, format-duration.ts}`. Empty state copy: *"Request insights will appear after the next App Router request."*

**Full details** in setup.md → `experimental.requestInsights` (the new section added alongside this update).

**Source:** canary.79–84 release notes at [github.com/vercel/next.js/releases](https://github.com/vercel/next.js/releases) · npm `next@canary` dist-tag.


### v16.3.0-preview.6 + canary.85 (July 13, 2026, 20:20Z preview tag / 23:54Z canary tag — same day as the previous cron)

A `v16.3.0-preview.6` was cut at `ea219c9` on 2026-07-13T20:20Z, then `canary.85` (`1ecd8f1`) was cut at 2026-07-13T23:54Z — npm `dist-tag.next` pointer moved 2026-07-13T23:57Z. Branch advanced ~24h after canary.84 cut — back on the 24–48h cadence. **The 4th of the 5-PR Request Insights stack (PR #93977) shipped in preview.6, the rest of canary.85 is adapter bug fixes + Turbopack internal improvements.**

canary.84 → canary.85 PR-by-PR diff at [compare/v16.3.0-canary.84...v16.3.0-canary.85](https://github.com/vercel/next.js/compare/v16.3.0-canary.84...v16.3.0-canary.85). canary-branch HEAD `1ecd8f1`, ~13 commits vs canary.84 (mix of preview.6 commits + canary.85 commits + 2 test-only disables).

**ONE material PR in canary.85 preview.6:**

- **[PR #93977](https://github.com/vercel/next.js/pull/93977) `request insights: add agent diagnosis access (4/5)`** (feedthejim) — the 4th of the 5-PR Request Insights stack ships. Adds:
  - **`get_request_insights` MCP tool** — agent calls it to fetch the last 100 requests (the same `InMemoryRequestInsightsStore` snapshot exposed at `GET /_next/development/request-insights` in canary.84). Returns the sanitized `RequestInsight[]` with `spans[]` + `fetches[]` per request. Input schema: `{ requestId?: string, htmlRequestId?: string }` (both optional filters).
  - **`next experimental-request-insights` CLI** — new `program.command('experimental-request-insights')` in `bin/next.ts` (+30/-0) + `cli/next-request-insights.ts` (+201/-0). Auto-discovers the running dev server from the project's `.next/lock` file and prints a human summary (route + duration + status + first 5 fetches per request). Flags: `--url <url>`, `--json`, `--limit <n>`. Replaces the need for a `curl ... | jq` one-liner for shell-only agents and CI scripts.
  - **Note on `subscribe_to_request_insights`** — the PR description suggested a second `subscribe_to_request_insights` MCP tool would land alongside `get_request_insights` (live stream). That tool was **deferred** in the actual implementation — only `get_request_insights` ships. The live stream is still available to browser subscribers via the canary.84 HMR transport `HMR_MESSAGE_SENT_TO_BROWSER.REQUEST_INSIGHTS_UPDATE` (`'requestInsightsUpdate'`). For MCP clients, re-invoke `get_request_insights` on a short interval (e.g. 1–2s) to approximate streaming; or use the `next experimental-request-insights` CLI in a watch loop.
  - This was the previously-docked "open" PR in canary.84 — the integration is live in canary.85+.

**Plus 6 OTHER commits in canary.85 + preview.6, four material user-facing:**

- **[PR #95749](https://github.com/vercel/next.js/pull/95749) `[turbopack] Switch make_production_chunks to use floats`** — internal: production chunk-size math was using integer arithmetic and could overflow on very large output (Vercel's own dashboard bundle triggers it). Now uses `f32` for the cost accumulation. No user-facing change, but fixes a class of `ChunkGroupTooLarge` errors on big client bundles.
- **[PR #95692](https://github.com/vercel/next.js/pull/95692) `Fix termination handling`** — `next dev` shutdown was leaving zombie worker threads when killed via `SIGTERM` (Ctrl-C in tmux/screen) instead of `SIGINT`. The new handler traps both, joins the worker pool, and exits cleanly. Worth knowing if you script dev-server restarts.
- **[PR #95579](https://github.com/vercel/next.js/pull/95579) `Turbopack: order CSS modules by chunk-group co-occurrence in linearize`** — CSS module emission order is now deterministic across builds (was previously dependent on `HashMap` iteration order in Rust). Fixes a class of "CSS works in dev, broken in build" bugs caused by CSS module load order differing between environments.
- **[PR #95264](https://github.com/vercel/next.js/pull/95264) `Fix Pages router 404 runtime rendering with Adapter`** — when deploying a Pages Router app to a target that uses a Build Adapter (Vercel, OpenNext, etc.), a 404 from a not-found route was rendered with the **App Router**'s default 404 template, not the Pages Router's. Fix: the adapter-side 404 dispatch now consults the route's router type.
- **[PR #95681](https://github.com/vercel/next.js/pull/95681) `Fix duplicate static files in adapter`** — adapters were emitting the same static asset twice when both `public/` and the App Router static export path had it. Now de-dupes by content hash.
- **[PR #95725](https://github.com/vercel/next.js/pull/95725) `Update font data`** — auto-generated update of the `next/font` Google Fonts registry (whitelist refreshes as Google retires/launches families).
- **[PR #95611](https://github.com/vercel/next.js/pull/95611) `Fork navigation-testing-lock module`** — internal refactor: the experimental `navigation-testing-lock` API moves from a dev-only flag to a real public-ish module so it can be imported by the new `next-dev-loop` skill without circular dep. Not user-facing.
- **[PR #95564](https://github.com/vercel/next.js/pull/95564) `docs: runtime prefetching update`** — docs-only: the runtime prefetching section now explicitly calls out the `prefetch: 'runtime'` segment config and its interaction with `partialPrefetching`. No code change.
- **[PR #95739](https://github.com/vercel/next.js/pull/95739) `[test] Disable i18n-api-support deploy test for Turbopack with adapters`** — test-only; CI was flaking on the i18n + adapter + Turbopack combo. Disables the deploy variant of the test, keeps the unit variant running.
- **[PR #95680](https://github.com/vercel/next.js/pull/95680) `[test] Disable middleware-rewrites deploy test for Turbopack with adapters`** — test-only; same reason.
- **[PR #95670](https://github.com/vercel/next.js/pull/95670) `test: skip redbox check in "static prefetch - missing suspense around search params"`** — test-only; the redbox assertion was depending on dev-only error UX that changed in canary.84.
- **[PR #95673](https://github.com/vercel/next.js/pull/95673) `docs: note default error/not-found UI follows OS color scheme, not app theme`** — docs-only: clarifies that the built-in error and not-found UIs always follow the OS `prefers-color-scheme`, not your app's theme, so dark-mode users always see a dark 404. Useful to mention in your own `not-found.tsx` if you want theme-aware rendering.

**What the Request Insights 4/5 agent-diagnosis PR enables in practice**

Now that `get_request_insights` (canary.85+) + the new `next experimental-request-insights` CLI (canary.85+) + the new Request Insights DevTools panel (canary.86+) are live, the agent loop looks like:

1. Agent edits a route in `app/dashboard/page.tsx`.
2. Agent calls `mcp__next-devtools-mcp__get_request_insights()` to see the last 100 requests to `/dashboard`.
3. Agent filters for the request matching the edit (by `requestId` or `route` + `startTime` window).
4. Agent reads the `spans[]` — finds e.g. `next.fetch /api/user` took 1800ms, with `http.status_code: 500` and a `cache_reason: 'cache-miss'`.
**`npm_config_user_agent` now consulted first (16.3.0-preview.8, [PR #95879](https://github.com/vercel/next.js/pull/95879)):** Turbopack's package-manager detection now checks the `npm_config_user_agent` environment variable before falling back to lockfile heuristics. This fixes a bug where `npx next` running inside a pnpm or yarn workspace would misdetect the package manager and attempt npm-specific install paths. If you use `npx next` or `pnpm dlx next` inside a monorepo, this PR ensures Turbopack respects the actual package manager environment.

**Source:** [PR #95879 — `Always consult npm_config_user_agent first`](https://github.com/vercel/next.js/pull/95879) · ships in **`next@preview@8`** + canary.94.

5. Agent correlates with the edit — the user just added a new `headers().get('x-tenant')` call in a Server Component, but the `next/cache` directive is on the same scope. Agent surfaces: "this Server Component is no longer eligible for `use cache` because it reads request-time headers; either pass the header into the cached function or split the read into a separate Server Component."
6. Agent edits the file. Repeats.

The 5th PR (#93978 — DevTools request panel) **shipped on the canary branch 2026-07-14T11:26:19Z** and shipped in canary.86 (2026-07-14). When the next canary is published, the dev overlay will grow a "Request Insights" tab that shows the same 100-request buffer as a sortable table with span-tree drill-down, focused/verbose toggle, and merged fetch+span display.

**Who needs to know:**

- **Agent authors:** update your `next-devtools-mcp` integration to call `get_request_insights` (snapshot — `{ requestId?, htmlRequestId? }` filter args) — registered in the canary.85+ `next-devtools-mcp` package. The `subscribe_to_request_insights` MCP tool mentioned in the original PR description was **deferred**; the live stream surface is the existing HMR transport `HMR_MESSAGE_SENT_TO_BROWSER.REQUEST_INSIGHTS_UPDATE` for browser subscribers, and MCP clients should re-invoke `get_request_insights` on a short interval to approximate streaming. For shell-only access, the new `next experimental-request-insights` CLI (canary.85+) is the recommended entry point — no MCP overhead, prints a human summary by default, `--json` flag for raw output, `--url` flag to skip lockfile auto-discovery. The 5/5 DevTools panel PR (#93978) shipped in canary.86 on 2026-07-14T23:31:39Z — the dev overlay's "Request Insights" tab will show the same 100-request buffer as a sortable table with span-tree drill-down for visual verification.
- **App authors:** nothing to do. The agent-diagnosis path is opt-in (via `experimental.requestInsights: true` from canary.84).
- **Adopters of the experimental `navigation-testing-lock`:** the canary.85 fork (#95611) means the API path is now `next-devtools-mcp`'s own module — if you imported the old internal path, you'll need to update. The API surface itself is unchanged.
- **Pages Router users on adapters (Vercel / OpenNext):** upgrade to canary.85+ to get the 404-template fix (#95264). 404s now show the Pages Router default, not the App Router default.
- **Anyone on CSS-heavy builds with deterministic-output requirements:** upgrade to canary.85+ to get the Turbopack CSS module ordering fix (#95579). Build-to-build CSS output is now stable.

**Sources:**
- [canary.85 release notes](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.85)
- [v16.3.0-preview.6 release notes](https://github.com/vercel/next.js/releases/tag/v16.3.0-preview.6)
- [PR #93977 — `request insights: add agent diagnosis access (4/5)`](https://github.com/vercel/next.js/pull/93977)
- [PR #93978 — `request insights: add DevTools request panel (5/5)` (SHIPPED in canary.86 on 2026-07-14T23:31:39Z; PR merged on canary branch 2026-07-14T11:26:19Z)](https://github.com/vercel/next.js/pull/93978)
- [compare v16.3.0-canary.84...v16.3.0-canary.85](https://github.com/vercel/next.js/compare/v16.3.0-canary.84...v16.3.0-canary.85)
- [`next-devtools-mcp` at v16.3.0-canary.85](https://github.com/vercel/next.js/tree/v16.3.0-canary.85/packages/next-devtools-mcp) — new `get_request_insights` MCP tool (the `subscribe_to_request_insights` tool from the PR description was deferred)
