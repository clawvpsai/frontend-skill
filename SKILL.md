---
name: Frontend
slug: frontend-developer
version: 1.3.0
description: Production-grade React/Next.js frontend development — ship modern web apps without common pitfalls.
metadata: {"emoji":"⚛️","requires":{"bins":["node","npm"]},"os":["linux","darwin","win32"]}
---

## Navigation

| Topic | File | When to Use |
|---|---|---|
| Project setup, TypeScript, Vite, Biome, env vars | `setup.md` | Starting a new project |
| React components, shadcn/ui, composition | `components.md` | Building UI |
| Server vs Client components, data fetching, `use cache` | `server-components.md` | Next.js App Router |
| Routing, layouts, loading, error boundaries | `routing.md` | Navigation & page structure |
| Forms with React Hook Form + Zod | `forms.md` | Any form or input |
| Zustand, React Query, data fetching | `state.md` | State & server state |
| Auth patterns, NextAuth.js, JWT | `auth.md` | User auth & sessions |
| Route handlers, Server Actions, API routes | `api.md` | Backend API endpoints |
| Tailwind CSS v4, design tokens, themes | `styling.md` | Styling & theming |
| Streaming, Suspense, image optimization, PPR | `performance.md` | Speed & UX |
| Vercel, Docker, Node adapter, self-hosted | `deployment.md` | Going live |
| XSS, CSRF, CSP, input sanitization | `security.md` | Hardening |
| Vitest, Playwright, component tests | `testing.md` | Test-driven dev |
| Strict TypeScript, generics, utilities | `typescript.md` | Type safety |
| Turbopack, React Compiler, advanced patterns | `patterns.md` | Composite recipes |
| Zustand, React Query, data fetching | `zustand.md` | Client state management |

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

- **Next.js 16.2.6** (latest — App Router, Server Components, Server Actions, Turbopack stable for production, Node.js middleware, `use cache` directive, PPR stable)
- **React 19.2.6** (React Compiler stable, `use()` hook, `useOptimistic`, `useFormStatus`, `useActionState`)
- **TypeScript 6.0.3** (strict by default, ES2025 target, import defer)
- **Zod 4.4.3** (14x faster string parsing, strict/loose object modes, `z.file()`, `z.templateLiteral()`)
- **Tailwind CSS v4.3** + **shadcn/ui**
- **Vite 8** (for non-Next projects; Next uses Turbopack)
- **Biome 2.4.16** (recommended linter/formatter — 10–100x faster than ESLint, v2 has breaking changes from v1)
- **TanStack Query v5** (React Query v5 — gcTime replaces cacheTime, improved SSR hydration)
- **React Hook Form v7** + **@hookform/resolvers v5** (compatible with Zod v4)
- **Node.js 22 LTS**

## Pro Tips

- Use `create-next-app --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"` for a fully typed Next.js project
- Run `npx tsc --noEmit` to type-check without building
- Use `shadcn@latest add <component>` to add components — they live in your codebase, not node_modules
- Use `next.config.ts` for all Next.js configuration (not `.js`)
- Use `react-hook-form` + `zod` for all forms — never manage form state with useState
- Enable React Compiler in `next.config.ts` with `reactCompiler: true` — eliminates most `useMemo`/`useCallback` manually
- Use Biome (`npx biome check`) for linting — 10–100x faster than ESLint
- Use `use cache` for all server-side data fetching in Next.js 16 — it's the explicit opt-in caching model

Start with `setup.md` to initialize a project, then `components.md` for shadcn/ui patterns.

## Next.js 16 Migration Notes

Key breaking changes from Next.js 15 → 16:

| Change | Impact |
|---|---|
| **`use cache` directive** replaces `unstable_cache` | New explicit caching API — mark data functions with `use cache` + `cacheTag` |
| **Implicit caching removed** — everything is dynamic by default | Old `fetch` caching patterns deprecated; use `use cache` explicitly |
| **PPR (Partial Prerendering) stable** — `cacheComponents: true` | Static shell + streaming dynamic content; no longer experimental |
| `images.minimumCacheTTL` default: 1min → **4 hours** | Image caching behavior change; set explicitly if you need 1min |
| **Router scroll optimization** enabled by default | Previously scroll was reset on navigation; now preserved by default |
| Flat config default in `@next/eslint-plugin-next` | ESLint config format change |
| Deprecated `.turbo` config object removed | Use `turbopack` key in `next.config.ts` instead |
| `publicRuntimeConfig` / `serverRuntimeConfig` removed | Use environment variables directly |
| **`next lint` removed** | Use Biome (`npx biome check`) or ESLint (`npx eslint .`) directly |

**Sources:**
- [Next.js 16 release notes](https://nextjs.org/blog/next-16)
- [Next.js `use cache` directive](https://nextjs.org/docs/app/api-reference/directives/use-cache)
- [Next.js `cacheTag`](https://nextjs.org/docs/app/api-reference/functions/cacheTag)
- [Next.js 16.2.6 security release](https://github.com/vercel/next.js/releases/tag/v16.2.6)
