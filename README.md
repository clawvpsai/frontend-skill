# Frontend Developer Skill

Production-grade React + Next.js 15 frontend development for agents building modern web applications.

## Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 15 (App Router) |
| Language | TypeScript 5.x (strict) |
| Styling | Tailwind CSS v4 + shadcn/ui |
| State | Zustand (client) + React Query / TanStack Query |
| Forms | React Hook Form + Zod |
| Build | Vite 6 (non-Next) / Turbopack (Next) |
| Testing | Vitest + Playwright |
| Auth | NextAuth.js v5 |

## Installation

This skill expects Node.js 20+ and npm/pnpm.

```bash
# Create a new Next.js project
npx create-next-app@latest my-app \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*" \
  --no-turbopack

cd my-app

# Add shadcn/ui
npx shadcn@latest init

# Add a component
npx shadcn@latest add button card form input

# Add Zod + React Hook Form
npm install react-hook-form zod @hookform/resolvers

# Add Zustand
npm install zustand

# Add React Query
npm install @tanstack/react-query
```

## Skill Structure

```
frontend/
├── SKILL.md                  # This file — navigation + critical rules
├── README.md                 # Overview
├── setup.md                  # Project init, TypeScript, env vars
├── components.md             # React components + shadcn/ui patterns
├── server-components.md      # Server vs Client component boundaries
├── routing.md                # Next.js App Router patterns
├── forms.md                  # React Hook Form + Zod
├── state.md                  # Zustand + React Query
├── auth.md                   # NextAuth.js + JWT
├── api.md                    # Route handlers + Server Actions
├── styling.md                # Tailwind v4 + design tokens
├── performance.md            # Streaming, Suspense, images
├── deployment.md              # Vercel, Docker, Node adapter
├── security.md               # XSS, CSRF, CSP
├── testing.md                # Vitest + Playwright
├── typescript.md             # Strict TypeScript patterns
├── patterns.md               # Composite recipes
└── zustand.md                # Client state management
```

## Quick Start

1. Start with `SKILL.md` — read the Critical Rules first
2. Run `setup.md` to bootstrap a project
3. Use `components.md` to build UI with shadcn/ui
4. Use `forms.md` for all form handling
5. Use `routing.md` for page structure
6. Use `server-components.md` to understand RSC boundaries

## T3 Stack Philosophy (End-to-End Typesafety)

The T3 Stack ideology of "types in, types out" applies throughout:

- Define your API response types once and import them everywhere
- Use Zod to infer TypeScript types from validation schemas
- Share types between frontend and backend (monorepo or a `types/` package)
- Never use `any` for API responses — define the shape explicitly

## Related Skills

- **Laravel** (`skills/laravel/`) — PHP backend API to pair with Next.js frontend
- **Gin** (`skills/gin/`) — Go backend API to pair with Next.js frontend
