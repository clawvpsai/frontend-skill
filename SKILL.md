---
name: Frontend
slug: frontend-skill
version: 1.4.22
description: Production-grade React/Next.js frontend development — ship modern web apps without common pitfalls.
metadata: {"emoji":"⚛️","requires":{"bins":["node","npm"]},"os":["linux","darwin","win32"]}
---

## Navigation

| Topic | File | When to Use |
|---|---|---|
| Project setup, TypeScript, Vite, Biome, env vars | `setup.md` | Starting a new project |
| React components, shadcn/ui, composition, ref forwarding | `components.md` | Building UI |
| Server vs Client components, data fetching, `use cache`, `use()` hook | `server-components.md` | Next.js App Router |
| Routing, layouts, loading, error boundaries, `proxy.ts`, parallel routes `default.tsx` (Next 16), `experimental.prefetchInlining` / `cachedNavigations` (16.2) | `routing.md` | Navigation & page structure |
| Forms with React Hook Form + Zod | `forms.md` | Any form or input |
| Zustand, React Query, TanStack Query v5 (incl. `useSuspenseQuery` + `React.use(query.promise)`), v5 migration | `state.md` | State & server state |
| Auth patterns, NextAuth.js v4/v5, JWT, session management, 2026 alternatives (Clerk, Better Auth) | `auth.md` | User auth & sessions |
| Route handlers, Server Actions, API routes, SSE, WebSockets | `api.md` | Backend API endpoints |
| Tailwind CSS v4, design tokens, themes | `styling.md` | Styling & theming |
| Streaming, Suspense, image optimization, PPR, App Shells (canary), prefetch controls | `performance.md` | Speed & UX |
| Vercel (incl. Vercel Connect, eve, Chat SDK), Docker, Node adapter, self-hosted, Next.js MCP Server | `deployment.md` | Going live |
| XSS, CSRF, CSP, input sanitization | `security.md` | Hardening |
| Vitest 4 (Browser Mode stable, Visual Regression, Playwright Trace) + Playwright + component tests | `testing.md` | Test-driven dev |
| Strict TypeScript, generics, utilities, `import defer`, Temporal API | `typescript.md` | Type safety |
| React Compiler, `<Activity>`, useOptimistic, `after()`, View Transitions | `patterns.md` | Composite recipes |

## Critical Rules (Never Forget)

- **`'use client'` boundary** — Only add when you need browser APIs, hooks, or interactivity. Keep everything else server-side.
- **Server Components can't use hooks** — No `useState`, `useEffect`, `useContext` in server components.
- **Colocate data fetching** — Fetch data in the component that needs it, not prop-drill from the top.
- **Zod for all schema validation** — Never trust raw form/API data without validation.
- **Tailwind = utility-first** — Don't fight it. Compose utilities, don't fight the cascade.
- **Server Actions = mutations** — Use them for form submits and data mutations; fetch for reads.
- **`<Image>` over `<img>`** — Always use Next's Image for automatic optimization and CLS prevention.
- **Abort in-flight requests** — Use `AbortController` to prevent race conditions on navigation.
- **Types in, types out** — Mirror backend response types exactly. No `any` from API calls.

## Version Defaults

- **Next.js 16.2.9** (latest stable — App Router, Server Components, Server Actions, Turbopack stable for production, Node.js proxy, `use cache` directive, PPR stable; 16.3 canary available with App Shells + fine-grained prefetch dedup)
- **React 19.2.7** (React Compiler 1.0 stable, `use()` hook, `useOptimistic`, `useFormStatus`, `useActionState`, `useEffectEvent`, `cacheSignal`, `cache`, `<Activity>`)
- **TypeScript 6.0.3** (strict by default, ES2026 target, import defer; TS 7 beta available with Go-based compiler)
- **Zod 4.4.3** (14x faster string parsing, strict/loose object modes, `z.file()`, `z.templateLiteral()`)
- **Tailwind CSS v4.3.1** + **shadcn/ui** (CSS-first config via `@theme` directive — no tailwind.config.js by default; v4.3.1 patch added `--silent` CLI flag, `@apply` with CSS mixins, cleaner spacing output)
- **Vite 8** (for non-Next projects; Next uses Turbopack)
- **@biomejs/biome 2.5.0** (recommended linter/formatter — 10–100x faster than ESLint, v2 has breaking changes from v1; run `npx biome migrate --write` after every upgrade)
- **TanStack Query v5.101.0** (React Query v5 — gcTime replaces cacheTime, improved SSR hydration, `skipToken` for dependent queries)
- **Vitest 4.1.9** (Browser Mode stable, Visual Regression testing via `toMatchScreenshot`, Playwright Trace support; requires Vite ≥ 6 + Node.js ≥ 20; **≥ 4.1.8 required for the CDP RCE fix (GHSA-g8mr-85jm-7xhm, CVSS 9.8)**; use `api.allowWrite: false, api.allowExec: false` in CI; Vitest 5 in beta)
- **React Hook Form v7.80.0** + **@hookform/resolvers v5.4.0** (7.80.0 ships per-field `disabled` on `useFieldArray` items; broad perf pass; fixes `[]` vs `{}` `deepEqual` regression from 7.79.0) (compatible with Zod v4; v8.0.0-beta available with `createForm` API)
- **Node.js 24 LTS** (Node.js 22 LTS also supported)

## Pro Tips

- Use `create-next-app --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"` for a fully typed Next.js project
- Run `npx tsc --noEmit` to type-check without building
- Use `shadcn@latest add <component>` to add components — they live in your codebase, not node_modules
- Use `next.config.ts` for all Next.js configuration (not `.js`)
- Use `react-hook-form` + `zod` for all forms — never manage form state with useState
- Enable React Compiler in `next.config.ts` with `reactCompiler: true` — eliminates most `useMemo`/`useCallback` manually
- Use Biome (`npx biome check`) for linting — 10–100x faster than ESLint
- Run `npx biome migrate --write` after every Biome upgrade to migrate your config for breaking changes
- Use `use cache` for all server-side data fetching in Next.js 16 — it's the explicit opt-in caching model

Start with `setup.md` to initialize a project, then `components.md` for shadcn/ui patterns.

## Next.js 16 Migration Notes

Key breaking changes from Next.js 15 → 16:

| Change | Impact |
|---|---|
| **`use cache` directive** replaces `unstable_cache` | New explicit caching API — mark data functions with `use cache` + `cacheTag` |
| **Implicit caching removed** — everything is dynamic by default | Old `fetch` caching patterns deprecated; use `use cache` explicitly |
| **PPR (Partial Prerendering) stable** — `cacheComponents: true` | Static shell + streaming dynamic content; no longer experimental |
| **Router scroll optimization** enabled by default | Previously scroll was reset on navigation; now preserved by default |
| Flat config default in `@next/eslint-plugin-next` | ESLint config format change |
| Deprecated `.turbo` config object removed | Use `turbopack` key in `next.config.ts` instead |
| `publicRuntimeConfig` / `serverRuntimeConfig` removed | Use environment variables directly |
| **`next lint` removed** | Use Biome (`npx biome check`) or ESLint (`npx eslint .`) directly |
| **`middleware.ts` deprecated → `proxy.ts`** | `middleware` export renamed to `proxy`; must be `async`; `matcher` moved to `next.config.ts` or exported from `proxy.ts` |

**Upgrade:** `npx @next/codemod@canary upgrade latest` (Next.js 16.2+) — automated upgrade CLI that handles codemods, config migrations, and breaking changes in one command. Alternatively `npx next upgrade` (interactive, Next.js 16.1+) or `npm install next@latest`.

**Sources:**
- [Next.js 16 release notes](https://nextjs.org/blog/next-16)
- [Next.js `use cache` directive](https://nextjs.org/docs/app/api-reference/directives/use-cache)
- [Next.js `cacheTag`](https://nextjs.org/docs/app/api-reference/functions/cacheTag)
- [Next.js `revalidateTag` — required `profile` argument](https://nextjs.org/docs/app/api-reference/functions/revalidateTag) (May 2026 update)
- [Next.js `cacheLife` profiles reference](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheLife)
- [Next.js 16.2.9 release](https://github.com/vercel/next.js/releases/tag/v16.2.9)
- [Next.js 16.2 — automated upgrade CLI](https://nextjs.org/blog/next-16-2)
- [Next.js 16.2 — AI Improvements (AGENTS.md, browser log forwarding, dev server lock file, next-browser)](https://nextjs.org/blog/next-16-2-ai)
- [Next.js 16.1 — Turbopack File System Caching stable + `next dev --inspect` + Bundle Analyzer](https://nextjs.org/blog/next-16-1)
- [Next.js 16.3 canary — prefetch controls + dedup](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.26)
- [Tailwind CSS v4.3.1 release notes](https://github.com/tailwindlabs/tailwindcss/releases/tag/v4.3.1)
- [shadcn/ui — GitHub Registries (June 2026)](https://ui.shadcn.com/docs/changelog/2026-06-github-registries)
- [shadcn eject (May 2026) — take ownership of registry CSS](https://ui.shadcn.com/docs/changelog)
- [React Hook Form 7.79.0 — `useFieldArray` `disabled` option (June 13, 2026)](https://github.com/react-hook-form/react-hook-form/releases/tag/v7.79.0)
- [React Hook Form 7.80.0 — per-field `disabled` + perf + `deepEqual` fix (June 20, 2026)](https://github.com/react-hook-form/react-hook-form/releases/tag/v7.80.0)
- [React Compiler 1.0 stable release](https://react.dev/blog/2025/10/07/react-compiler-1)
- [Next.js React Compiler integration](https://nextjs.org/docs/app/api-reference/config/next-config-js/reactCompiler)
- [Vercel research: AGENTS.md outperforms skills (100% vs 79%)](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals)
- [Vitest 4.0 announcement (Oct 21, 2025) — Browser Mode stable + Visual Regression + Playwright Trace](https://voidzero.dev/posts/announcing-vitest-4)
- [Vitest Visual Regression Testing docs](https://vitest.dev/guide/browser/visual-regression-testing)
- [Vitest 3 → 4 migration guide](https://vitest.dev/guide/migration.html)
- [GHSA-g8mr-85jm-7xhm — Vitest Browser Mode CDP RCE (CVSS 9.8, June 1, 2026)](https://github.com/vitest-dev/vitest/security/advisories/GHSA-g8mr-85jm-7xhm)
- [GHSA-2h32-95rg-cppp — Vitest otelCarrier XSS → RCE (CVSS 9.6, May 19, 2026)](https://github.com/vitest-dev/vitest/security/advisories/GHSA-2h32-95rg-cppp)
- [GHSA-5xrq-8626-4rwp — Vitest UI arbitrary file read + RCE on Windows (CVSS 9.8, May 19, 2026)](https://github.com/vitest-dev/vitest/security/advisories/GHSA-5xrq-8626-4rwp)
- [Vitest browser.api config — allowWrite / allowExec (4.1.0+)](https://main.vitest.dev/config/browser/api)
- [VoidZero is joining Cloudflare (June 4, 2026) — Vite, Vitest, Rolldown, Oxc, Vite+ remain MIT and community-driven](https://blog.cloudflare.com/voidzero-joins-cloudflare/)
- [VoidZero's own announcement (Evan You)](https://voidzero.dev/posts/voidzero-cloudflare)
- [StepSecurity — Mastra npm supply chain attack (June 17, 2026) — 142+ packages, easy-day-js typosquat, easy-day-js@1.11.22 RCE](https://www.stepsecurity.io/blog/mastra-npm-packages-compromised-using-easy-day-js)
- [Snyk — Mastra npm Scope Takeover (June 16, 2026) — forgotten contributor account, postinstall dropper, RAT + crypto stealer](https://snyk.io/blog/a-forgotten-contributor-account-compromised-the-entire-mastra-npm-package-scope/)
- [Node.js — Thursday, June 18, 2026 Security Releases (12 CVEs, 2 HIGH: CVE-2026-48933 WebCrypto DoS + CVE-2026-48618 TLS wildcard bypass via Unicode normalization)](https://nodejs.org/en/blog/vulnerability/june-2026-security-releases)
- [Next.js docs — Data fetching: Server Actions security (public POST endpoints)](https://nextjs.org/docs/app/guides/data-security)
- [Next.js docs — Authentication: Server Actions (return null in layout is not recommended)](https://nextjs.org/docs/app/guides/authentication)
- [BuildMVPFast — Server Actions Security: Real Vulnerabilities (June 18, 2026)](https://www.buildmvpfast.com/blog/nextjs-server-actions-security-vulnerabilities-2026)
- [Makerkit — Server Actions Security: 5 Vulnerabilities (Next 16.2.6)](https://makerkit.dev/blog/tutorials/secure-nextjs-server-actions)
- [Vercel Ship 2026 recap (June 17, 2026) — Agent Stack, Vercel Connect, Vercel Agent (private beta)](https://vercel.com/blog/vercel-ship-2026-recap)
- [Vercel Connect — scoped, short-lived tokens for agents (June 17, 2026)](https://vercel.com/blog/introducing-vercel-connect)
- [ProjectDiscovery — The Vulnerability Curve Bent With the AI Curve (June 18, 2026)](https://projectdiscovery.io/blog/the-vulnerability-curve-bent-with-the-ai-curve)
- [Authgear — Next.js Security Best Practices 2026](https://www.authgear.com/post/nextjs-security-best-practices/)
- [pnpm — minimumReleaseAge (supply-chain defense, blocks short-lived malicious publishes)](https://pnpm.io/supply-chain-security)
- [OpenAI — Our response to the TanStack npm supply chain attack (May 13, 2026)](https://openai.com/index/our-response-to-the-tanstack-npm-supply-chain-attack/)

- [Digital Applied — Node.js June 2026 Security Releases: 12 CVEs, 2 HIGH (full list + patch guide)](https://www.digitalapplied.com/blog/nodejs-june-2026-security-releases-cve-patch-guide)
- [GitHub Advisory: undici WebSocket client DoS via fragment count bypass (CVE-2026-11525, June 19, 2026)](https://github.com/advisories?query=undici+type%3Areviewed)
- [GitHub Advisory: undici HTTP header injection via Set-Cookie percent-decoding (CVE-2026-12151, June 19, 2026)](https://github.com/advisories?query=undici+type%3Areviewed)
- [GitHub Advisory: undici Set-Cookie SameSite attribute downgrade (CVE-2026-9679, June 19, 2026)](https://github.com/advisories?query=undici+type%3Areviewed)
- [undici on npm (current: 8.5.0, 7.28.0 LTS)](https://www.npmjs.com/package/undici)
- [Chat SDK: bring agents to your users (Vercel blog, March 19, 2026)](https://vercel.com/blog/chat-sdk-brings-agents-to-your-users)
- [Chat SDK on npm (chat@4.31.0, June 16, 2026)](https://www.npmjs.com/package/chat)
- [Chat SDK docs (chat-sdk.dev)](https://chat-sdk.dev/docs)
- [Chat SDK adapter directory (10+ official + vendor-official adapters)](https://chat-sdk.dev/adapters)
- [Introducing eve — Vercel's open-source agent framework (June 17, 2026) — durable execution + sandbox + approvals + subagents + evals](https://vercel.com/blog/introducing-eve)
- [Vercel Ship 2026 recap — Agent Stack, eve, Vercel Connect, Vercel Agent (private beta), Claude Managed Agents](https://vercel.com/blog/vercel-ship-2026-recap)
- [TanStack Query v5 — Suspense guide](https://tanstack.com/query/v5/docs/framework/react/guides/suspense)
- [TanStack Query v5 — `useSuspenseQuery` reference](https://tanstack.com/query/v5/docs/framework/react/reference/useSuspenseQuery)
- [TanStack Query v5 — `useSuspenseInfiniteQuery` reference](https://tanstack.com/query/v5/docs/framework/react/reference/useSuspenseInfiniteQuery)
- [TanStack Query v5 — `useSuspenseQueries` reference](https://tanstack.com/query/v5/docs/framework/react/reference/useSuspenseQueries)
- [Next.js docs — Prefetching guide (16.2.9)](https://nextjs.org/docs/app/guides/prefetching)
- [Next.js docs — Parallel Routes file convention (16.2.9 — `default.tsx` required)](https://nextjs.org/docs/app/api-reference/file-conventions/parallel-routes)
- [Next.js 16 release notes — Layout deduplication + Incremental prefetching](https://nextjs.org/blog/next-16)
- [LogRocket — I tested every major auth library for Next.js in 2026 (April 20, 2026)](https://blog.logrocket.com/best-auth-library-nextjs-2026/)
- [Aikido Security — Multiple JetBrains IDE plugins caught stealing AI keys (June 16, 2026 — full IOC list, 15 plugins, 7 banned publisher accounts)](https://www.aikido.dev/blog/multiple-jetbrains-ide-plugins-caught-stealing-ai-keys)
- [JetBrains Marketplace Ecosystem Security Update — Addressing Malicious Third-Party AI Plugins (official disclosure, June 17, 2026)](https://blog.jetbrains.com/platform/2026/06/marketplace-ecosystem-security-update-malicious-ai-plugins)
- [Threat-Modeling.com — JetBrains Marketplace Malicious Plugins Stealing AI API Keys (June 17, 2026)](https://threat-modeling.com/jetbrains-marketplace-malicious-plugins-ai-key-theft-june-2026/)
- [OffSeq Threat Radar — 15 JetBrains Marketplace plugins quietly stealing developers\' AI API keys (~70,000 installs, June 17, 2026)](https://radar.offseq.com/threat/15-jetbrains-marketplace-plugins-were-quietly-stea-8bacd71f)
- [SANS Stormcast Thursday June 18, 2026 — JetBrains Plugins segment](https://isc.sans.edu/podcastdetail/9978)
- [JetBrains Marketplace Approval Guidelines (the manual review process that failed)](https://plugins.jetbrains.com/docs/marketplace/jetbrains-marketplace-approval-guidelines.html#approval-process)