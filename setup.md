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

## Next.js Configuration

### `next.config.ts`

```ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
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
  "$schema": "https://biomejs.dev/schemas/2.4.16/schema.json",
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
