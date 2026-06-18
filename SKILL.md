---
name: Frontend
slug: frontend-skill
version: 1.4.13
description: Production-grade React/Next.js frontend development — ship modern web apps without common pitfalls.
metadata: {"emoji":"⚛️","requires":{"bins":["node","npm"]},"os":["linux","darwin","win32"]}
---

## Navigation

| Topic | File | When to Use |
|---|---|---|
| Project setup, TypeScript, Vite, Biome, env vars | `setup.md` | Starting a new project |
| React components, shadcn/ui, composition, ref forwarding | `components.md` | Building UI |
| Server vs Client components, data fetching, `use cache`, `use()` hook | `server-components.md` | Next.js App Router |
| Routing, layouts, loading, error boundaries, `proxy.ts` | `routing.md` | Navigation & page structure |
| Forms with React Hook Form + Zod | `forms.md` | Any form or input |
| Zustand, React Query, TanStack Query v5 patterns | `state.md` | State & server state |
| Auth patterns, NextAuth.js, JWT, session management | `auth.md` | User auth & sessions |
| Route handlers, Server Actions, API routes, SSE, WebSockets | `api.md` | Backend API endpoints |
| Tailwind CSS v4, design tokens, themes | `styling.md` | Styling & theming |
| Streaming, Suspense, image optimization, PPR, App Shells (canary), prefetch controls | `performance.md` | Speed & UX |
| Vercel, Docker, Node adapter, self-hosted, Next.js MCP Server | `deployment.md` | Going live |
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
- **Vitest 4.1.9** (Browser Mode stable, Visual Regression testing via `toMatchScreenshot`, Playwright Trace support; requires Vite ≥ 6 + Node.js ≥ 20; Vitest 5 in beta)
- **React Hook Form v7.79.0** + **@hookform/resolvers v5.4.0** (compatible with Zod v4; v8.0.0-beta available with `createForm` API)
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
- [React Compiler 1.0 stable release](https://react.dev/blog/2025/10/07/react-compiler-1)
- [Next.js React Compiler integration](https://nextjs.org/docs/app/api-reference/config/next-config-js/reactCompiler)
- [Vercel research: AGENTS.md outperforms skills (100% vs 79%)](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals)
- [Vitest 4.0 announcement (Oct 21, 2025) — Browser Mode stable + Visual Regression + Playwright Trace](https://voidzero.dev/posts/announcing-vitest-4)
- [Vitest Visual Regression Testing docs](https://vitest.dev/guide/browser/visual-regression-testing)
- [Vitest 3 → 4 migration guide](https://vitest.dev/guide/migration.html)
