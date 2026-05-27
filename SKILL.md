---
name: Frontend
slug: frontend-developer
version: 1.0.0
description: Production-grade React/Next.js frontend development — ship modern web apps without common pitfalls.
metadata: {"emoji":"⚛️","requires":{"bins":["node","npm"]},"os":["linux","darwin","win32"]}
---

## Navigation

| Topic | File | When to Use |
|---|---|---|
| Project setup, TypeScript, Vite, env vars | `setup.md` | Starting a new project |
| React components, shadcn/ui, composition | `components.md` | Building UI |
| Server vs Client components, data fetching | `server-components.md` | Next.js App Router |
| Routing, layouts, loading, error boundaries | `routing.md` | Navigation & page structure |
| Forms with React Hook Form + Zod | `forms.md` | Any form or input |
| Zustand, React Query, data fetching | `state.md` | State & server state |
| Auth patterns, NextAuth.js, JWT | `auth.md` | User auth & sessions |
| Route handlers, Server Actions, API routes | `api.md` | Backend API endpoints |
| Tailwind CSS v4, design tokens, themes | `styling.md` | Styling & theming |
| Streaming, Suspense, image optimization | `performance.md` | Speed & UX |
| Vercel, Docker, Node adapter, self-hosted | `deployment.md` | Going live |
| XSS, CSRF, CSP, input sanitization | `security.md` | Hardening |
| Vitest, Playwright, component tests | `testing.md` | Test-driven dev |
| Strict TypeScript, generics, utilities | `typescript.md` | Type safety |
| RSC vs client boundaries, patterns | `patterns.md` | Composite recipes |
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

- **Next.js 15** (latest — App Router, Server Components, Server Actions)
- **React 19** (concurrent features, use() hook, useFormStatus)
- **TypeScript 6.0** (strict by default, ES2025 target, import defer)
- **Zod 4** (14x faster string parsing, strict/loose object modes)
- **Tailwind CSS v4** + **shadcn/ui**
- **Vite 6** (for non-Next projects; Next uses Turbopack)
- **Node.js 22 LTS**

## Pro Tips

- Use `create-next-app --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"` for a fully typed Next.js project
- Run `npx tsc --noEmit` to type-check without building
- Use `shadcn@latest add <component>` to add components — they live in your codebase, not node_modules
- Use `next.config.ts` for all Next.js configuration (not `.js`)
- Use `react-hook-form` + `zod` for all forms — never manage form state with useState

Start with `setup.md` to initialize a project, then `components.md` for shadcn/ui patterns.
