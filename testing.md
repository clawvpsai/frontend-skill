# Testing — Vitest + Playwright

> **Vitest 4.0 (October 21, 2025)** brought Browser Mode stable, Visual Regression testing (`toMatchScreenshot`), and Playwright Trace support. This file uses Vitest 4 patterns. For Vitest 3 → 4 migration, see [Migration Guide](https://vitest.dev/guide/migration.html).

## Testing Pyramid

```
        ┌──────────────┐
        │     E2E      │  ← Few, slow, high confidence
        │  (Playwright)│
        ├──────────────┤
        │  Integration │  ← More, moderate speed
        │   (Vitest)   │
        ├──────────────┤
        │    Unit      │  ← Many, fast, isolated
        │   (Vitest)   │
        └──────────────┘
```

## Vitest Setup

```bash
npm install -D vitest@4 @vitejs/plugin-react @vitest/browser @testing-library/react @testing-library/jest-dom
```

### `vitest.config.ts`

```ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',  // or 'happy-dom' — or use Browser Mode (see below)
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    include: ['src/**/*.{test,spec}.{js,ts,jsx,tsx}'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
    },
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
})
```

### `src/test/setup.ts`

```ts
import '@testing-library/jest-dom'
```

## Unit Tests

### Component Tests

```tsx
// components/counter.test.tsx
import { describe, it, expect, vi } from 'vitest'
import { render, screen, userEvent } from '@testing-library/react'
import { Counter } from '../counter'

describe('Counter', () => {
  it('increments count on click', async () => {
    const user = userEvent.setup()
    render(<Counter initialCount={0} />)
    
    const button = screen.getByRole('button', { name: /increment/i })
    await user.click(button)
    
    expect(screen.getByText('1')).toBeInTheDocument()
  })
  
  it('decrements count on click', async () => {
    const user = userEvent.setup()
    render(<Counter initialCount={5} />)
    
    const button = screen.getByRole('button', { name: /decrement/i })
    await user.click(button)
    
    expect(screen.getByText('4')).toBeInTheDocument()
  })
})
```

### Hook Tests

```tsx
// hooks/use-counter.test.ts
import { describe, it, expect } from 'vitest'
import { renderHook, act } from '@testing-library/react'
import { useCounter } from './use-counter'

describe('useCounter', () => {
  it('initializes with default value', () => {
    const { result } = renderHook(() => useCounter())
    expect(result.current.count).toBe(0)
  })
  
  it('accepts initial value', () => {
    const { result } = renderHook(() => useCounter(10))
    expect(result.current.count).toBe(10)
  })
  
  it('increments', () => {
    const { result } = renderHook(() => useCounter())
    act(() => { result.current.increment() })
    expect(result.current.count).toBe(1)
  })
})
```

### Utility Function Tests

```ts
// lib/utils.test.ts
import { describe, it, expect } from 'vitest'
import { cn } from './utils'
import { clsx } from 'clsx'

describe('cn()', () => {
  it('merges class names', () => {
    expect(cn('foo', 'bar')).toBe('foo bar')
  })
  
  it('handles conditional classes', () => {
    expect(cn('base', false && 'hidden')).toBe('base')
    expect(cn('base', true && 'active')).toBe('base active')
  })
  
  it('handles undefined', () => {
    expect(cn('base', undefined)).toBe('base')
  })
})
```

## Integration Tests

### Testing Server Actions

```ts
// app/actions.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { createPost, updatePost } from '../actions'
import { db } from '@/lib/db'

// Mock the database
vi.mock('@/lib/db', () => ({
  db: {
    post: {
      create: vi.fn(),
      update: vi.fn(),
    },
  },
}))

beforeEach(() => {
  vi.clearAllMocks()
})

describe('createPost', () => {
  it('creates a post with valid data', async () => {
    vi.mocked(db.post.create).mockResolvedValue({
      id: '1',
      title: 'Test',
      content: 'Content',
      published: false,
      authorId: 'author-1',
      createdAt: new Date(),
      updatedAt: new Date(),
    })
    
    const formData = new FormData()
    formData.set('title', 'Test')
    formData.set('content', 'Content')
    
    await createPost(formData)
    
    expect(db.post.create).toHaveBeenCalledWith({
      data: expect.objectContaining({
        title: 'Test',
        content: 'Content',
      }),
    })
  })
})
```

### Testing API Routes

```ts
// app/api/posts/route.test.ts
import { describe, it, expect } from 'vitest'
import { GET, POST } from './route'
import { NextRequest } from 'next/server'

describe('GET /api/posts', () => {
  it('returns paginated posts', async () => {
    const request = new NextRequest('http://localhost:3000/api/posts?page=1&limit=10')
    const response = await GET(request)
    const data = await response.json()
    
    expect(response.status).toBe(200)
    expect(data).toHaveProperty('data')
    expect(data).toHaveProperty('meta')
  })
})
```

### Testing Server Components

Server Components render on the server and return JSX — they can be tested by checking their output:

```tsx
// app/posts/page.test.tsx — testing a Server Component
import { describe, it, expect, vi } from 'vitest'
import { render } from '@testing-library/react'
import PostsPage from '../page'
import { db } from '@/lib/db'

vi.mock('@/lib/db', () => ({
  db: {
    post: {
      findMany: vi.fn(),
    },
  },
}))

describe('PostsPage (Server Component)', () => {
  it('renders posts', async () => {
    vi.mocked(db.post.findMany).mockResolvedValue([
      { id: '1', title: 'Post 1', content: 'Content 1', published: true, authorId: 'a1', createdAt: new Date(), updatedAt: new Date() },
      { id: '2', title: 'Post 2', content: 'Content 2', published: true, authorId: 'a1', createdAt: new Date(), updatedAt: new Date() },
    ])

    // Server Components can be rendered with render()
    const { getByText } = render(await PostsPage())
    
    expect(getByText('Post 1')).toBeInTheDocument()
    expect(getByText('Post 2')).toBeInTheDocument()
  })
})
```

**Key insight:** Server Components are `async` functions in Next.js 15+. Use `render(await Component())` in tests — the `await` is necessary because the component fetches data.

### Testing React 19 `useActionState`

```tsx
// components/contact-form.test.tsx
import { describe, it, expect } from 'vitest'
import { render, screen, userEvent, waitFor } from '@testing-library/react'
import { ContactForm } from './contact-form'

// Mock the server action
vi.mock('@/app/actions', () => ({
  submitContact: vi.fn(),
}))

describe('ContactForm with useActionState', () => {
  it('shows pending state while submitting', async () => {
    const { submitContact } = await import('@/app/actions')
    vi.mocked(submitContact).mockImplementation(() => {
      return new Promise(() => {}) // Never resolves — keeps pending state
    })
    
    render(<ContactForm />)
    
    const submitButton = screen.getByRole('button', { name: /send message/i })
    await userEvent.click(submitButton)
    
    // Button should show pending state
    await waitFor(() => {
      expect(screen.getByRole('button', { name: /sending/i })).toBeDisabled()
    })
  })
})
```

### Testing React 19 `useOptimistic`

```tsx
// components/like-button.test.tsx
import { describe, it, expect } from 'vitest'
import { render, screen, userEvent, waitFor } from '@testing-library/react'
import { LikeButton } from './like-button'

describe('LikeButton with useOptimistic', () => {
  it('shows optimistic update immediately on click', async () => {
    const user = userEvent.setup()
    
    render(<LikeButton post={{ id: '1', content: 'Test', likes: 10 }} />)
    
    const button = screen.getByRole('button')
    
    await user.click(button)
    
    // Should show 11 immediately (optimistic), before server responds
    expect(screen.getByText('11 likes')).toBeInTheDocument()
  })
})
```

### Testing React 19 `<Activity>` (React 19.2)

```tsx
// components/submit-form.test.tsx
import { describe, it, expect } from 'vitest'
import { render, screen, userEvent, waitFor } from '@testing-library/react'
import { SubmitForm } from './submit-form'

describe('SubmitForm with Activity', () => {
  it('shows activity indicator while submitting', async () => {
    render(<SubmitForm />)
    
    const submitButton = screen.getByRole('button', { name: /submit/i })
    await userEvent.click(submitButton)
    
    // Activity should detect pending state
    await waitFor(() => {
      expect(screen.getByRole('status')).toBeInTheDocument() // or specific loading indicator
    })
  })
})
```

## Playwright (E2E)

```bash
npm install -D @playwright/test
npx playwright install chromium
```

### `playwright.config.ts`

```ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
})
```

### E2E Test Example

```ts
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Authentication', () => {
  test('user can login with valid credentials', async ({ page }) => {
    await page.goto('/login')
    
    await page.fill('[name="email"]', 'test@example.com')
    await page.fill('[name="password"]', 'password123')
    await page.click('[type="submit"]')
    
    await expect(page).toHaveURL('/dashboard')
    await expect(page.getByText('Welcome')).toBeVisible()
  })
  
  test('shows error with invalid credentials', async ({ page }) => {
    await page.goto('/login')
    
    await page.fill('[name="email"]', 'wrong@example.com')
    await page.fill('[name="password"]', 'wrongpassword')
    await page.click('[type="submit"]')
    
    await expect(page.getByText('Invalid credentials')).toBeVisible()
    await expect(page).toHaveURL('/login')
  })
})
```

### Running Tests

```bash
# Unit + Integration
npm run test

# Unit + Integration with coverage
npm run test:coverage

# E2E
npx playwright test

# E2E with UI
npx playwright test --ui

# Single file
npx playwright test e2e/auth.spec.ts
```

## Playwright 1.61 (June 15, 2026) — What's New

Playwright 1.61 is a **major feature release**. Two pieces (WebAuthn passkeys + WebStorage) are first-class APIs that change how you write E2E tests for auth and storage; the rest are smaller quality-of-life wins. **Browsers bumped to Chromium 149, Firefox 151, WebKit 26.5.** Tested against Google Chrome 149 and Microsoft Edge 149 stable channels. Playwright 1.61.1 (June 23, 2026) shipped the next day with five regression fixes — pin to 1.61.1+.

### WebAuthn passkeys — `browserContext.credentials`

The single biggest addition. A **virtual authenticator** that lets you register passkeys and answer `navigator.credentials.create()` / `navigator.credentials.get()` ceremonies in the page — no real hardware key, no WebAuthn shim library, works in all three engines. Replaces every team's home-rolled "mock `navigator.credentials`" helper.

```ts
// e2e/passkey.spec.ts
import { test, expect } from '@playwright/test'

test('user can log in with a passkey', async ({ browser }) => {
  const context = await browser.newContext()

  // 1) Seed a passkey your backend provisioned for a test user.
  await context.credentials.create('example.com', {
    id: 'credential-id-from-backend',
    userHandle: 'user-handle-from-backend',
    privateKey: 'base64url-encoded-private-key',
    publicKey: 'base64url-encoded-public-key',
  })
  await context.credentials.install()

  const page = await context.newPage()
  await page.goto('https://example.com/login')

  // 2) The page's navigator.credentials.get() is answered with the seeded passkey.
  await page.getByRole('button', { name: /sign in with passkey/i }).click()
  await expect(page).toHaveURL('/dashboard')
})
```

You can also let the app register a passkey once in a setup test, read it back with `credentials.get()`, and seed it into later tests — see the [Credentials docs](https://playwright.dev/docs/api/class-credentials).

**Cross-reference:** if you're using Better Auth (which supports passkeys natively — see `auth.md`), you can now E2E the full passkey flow against the real Better Auth UI instead of mocking the navigator API.

### WebStorage — `page.localStorage` / `page.sessionStorage`

A proper origin-aware WebStorage API that replaces the brittle `page.evaluate(() => localStorage.setItem(...))` boilerplate. Reads and writes the page's storage **for the current origin**, no string-serialization in test code:

```ts
// Set tokens / seeded state without page.evaluate gymnastics
await page.localStorage.setItem('token', 'abc123')
await page.sessionStorage.setItem('wizard-step', '2')

// Read back
const token = await page.localStorage.getItem('token')
const items = await page.sessionStorage.items() // → Array<{ name, value }>
```

### New APIs at a glance

| Area | API | What it does |
|---|---|---|
| Network | `apiResponse.securityDetails()` / `apiResponse.serverAddr()` | Mirror the browser-side `response.securityDetails()` / `serverAddr()` on API contexts — parity between `request.fetch()` and `page.request.get()` |
| Browser / CDP | `browserType.connectOverCDP({ artifactsDir })` | New option controls where traces + downloads are stored when attaching to an existing browser (was always `cwd`) |
| Screencast | `screencast.showActions({ cursor: 'always' \| 'never' \| 'auto' })` | New `cursor` option controls pointer-action cursor decoration in screencast recordings |
| Screencast | `screencast.start({ onFrame })` | The `onFrame` callback now receives a `timestamp` of when the frame was presented by the browser — lets you measure end-to-end paint latency in screencast tests |
| Test runner | `testOptions.video: 'on-all-retries' \| 'retain-on-first-failure' \| 'retain-on-failure-and-retries'` | Video modes now match the `trace` mode vocabulary — use `'on-all-retries'` for flaky-test forensics, `'retain-on-first-failure'` to debug just the first failure without filling disk |
| Test runner | `expect.soft.poll(...)` | Soft-assertion polling — fails the test at the end if any soft assertion never resolved, doesn't stop the run mid-test |
| Test runner | `fullConfig.argv` | Snapshot of `process.argv` from the runner process — read custom CLI args passed after `--` without parsing env vars |
| Test runner | `fullConfig.failOnFlakyTests` | Mirrors the `failOnFlakyTests` config option so reporters can explain why a flaky run failed |
| Test runner | `testInfo.errors` | Each sub-error of an `AggregateError` is now a separate entry — your reporter can render parallel-call failures properly |
| CLI | `-G` shorthand | New shorthand for `--grep-invert` (analog of `-g` for `--grep`) |
| Platform | Ubuntu 26.04 | Playwright now supports Ubuntu 26.04 LTS as a host |
| Recording | HAR + trace WebSocket capture | HAR and trace recordings now include WebSocket frames — debug flaky WS tests the same way you debug flaky HTTP |

### Browser versions (1.61)

- Chromium 149.0.7827.55
- Mozilla Firefox 151.0
- WebKit 26.5

### Playwright 1.61.1 (June 23, 2026) — regression fixes

Eight days after 1.61.0, the team shipped 1.61.1 with **five fixes for regressions** introduced in 1.61. Pin to ≥ 1.61.1 if you upgraded on day one:

1. **`expect.extend` overriding default matchers** — A custom matcher registered with the same name as a built-in matcher (e.g. `expect.extend({ toBeVisible: ... })`) corrupted the default `toBeVisible` implementation. Custom matchers no longer shadow built-ins.
2. **UI mode API request byte count mismatch** — `apiRequestContext._wrapApiCall` reported wrong byte counts in UI mode (same test passed in headed mode). Byte counts now consistent across UI and headed.
3. **Trace viewer WebSocket time scaling** — WebSocket message timestamps in the trace viewer were divided by 1000 (a 1-second delay looked like 1 ms). Fixed.
4. **Sync loader crash on Node 22.15** — ESM loader threw `context.conditions?.includes is not a function` on Node 22.15. Fixed.
5. **pnpm workspace symlink resolution** — Extensionless `.ts` subpath imports across pnpm workspace symlinks failed in the sync ESM loader. Fixed.

## React Testing Library Patterns

```tsx
// Prefer queries by role, label, placeholder, text — NOT data-testid

// ✅ Good — semantic queries
screen.getByRole('button', { name: /submit/i })
screen.getByLabelText(/email/i)
screen.getByText(/hello world/i)

// ❌ Bad — fragile test IDs
screen.getByTestId('submit-btn')
```

### Async Queries

```tsx
// Wait for async data
const { findByText } = render(<UserProfile userId="1" />)

// Wait for loading to finish
await expect(screen.findByText('Loading...')).toBeVisible()
const content = await screen.findByText('User Name')
```

## Mocking

### Mocking Modules

```ts
vi.mock('@/lib/api', () => ({
  fetchUser: vi.fn().mockResolvedValue({ id: '1', name: 'Alice' }),
}))
```

### Mocking Time

```ts
import { vi } from 'vitest'

it('shows correct relative time', async () => {
  const now = new Date('2025-01-01T12:00:00Z')
  vi.setSystemTime(now)
  
  render(<RelativeTime date={new Date('2025-01-01T11:59:00Z')} />)
  expect(screen.getByText('just now')).toBeInTheDocument()
  
  vi.useRealTimers()
})
```

### Mocking Next.js `use cache`

Server functions using `use cache` can be mocked like any other async function:

```ts
vi.mock('@/lib/data', () => ({
  getTopPosts: vi.fn().mockResolvedValue([
    { id: '1', title: 'Post 1', views: 1000 },
  ]),
}))

// In your test, the mocked version is used instead of the 'use cache' version
```

## Testing Zustand Stores

Zustand stores can be tested directly without rendering components:

```ts
// stores/cart-store.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

// Recreate the store inline for testing (or import and reset between tests)
const createTestCartStore = () =>
  create<{ items: { id: string; name: string; price: number }[]; addItem: (item: any) => void; removeItem: (id: string) => void }>(
    (set) => ({
      items: [],
      addItem: (item) => set((s) => ({ items: [...s.items, item] })),
      removeItem: (id) => set((s) => ({ items: s.items.filter((i) => i.id !== id) })),
    })
  )

describe('Cart Store', () => {
  it('starts with empty cart', () => {
    const store = createTestCartStore()
    expect(store.getState().items).toEqual([])
  })

  it('adds an item', () => {
    const store = createTestCartStore()
    store.getState().addItem({ id: '1', name: 'Widget', price: 9.99 })
    expect(store.getState().items).toHaveLength(1)
    expect(store.getState().items[0].name).toBe('Widget')
  })

  it('removes an item', () => {
    const store = createTestCartStore()
    store.getState().addItem({ id: '1', name: 'Widget', price: 9.99 })
    store.getState().removeItem('1')
    expect(store.getState().items).toHaveLength(0)
  })

  it('notifies subscribers on state change', () => {
    const store = createTestCartStore()
    const listener = vi.fn()
    store.subscribe(listener)

    store.getState().addItem({ id: '1', name: 'Widget', price: 9.99 })

    expect(listener).toHaveBeenCalledTimes(1)
    expect(listener).toHaveBeenCalledWith(
      expect.objectContaining({ items: expect.arrayContaining([expect.objectContaining({ id: '1' })]) })
    )
  })
})
```

### Testing Zustand with Immer Middleware

```ts
// stores/editor-store.test.ts
import { describe, it, expect } from 'vitest'
import { create } from 'zustand'
import { immer } from 'zustand/middleware/immer'

const createEditorStore = () =>
  create<{ content: string; history: string[]; setContent: (c: string) => void }>()(
    immer((set) => ({
      content: '',
      history: [],
      setContent: (c) =>
        set((s) => {
          s.history.push(s.content)
          s.content = c
        }),
    }))
  )

describe('Editor Store (Immer)', () => {
  it('tracks history on content change', () => {
    const store = createEditorStore()
    store.getState().setContent('Hello')
    store.getState().setContent('World')
    expect(store.getState().history).toEqual(['', 'Hello'])
    expect(store.getState().content).toBe('World')
  })
})
```

### Testing Zustand with `persist` Middleware

When testing stores with `persist`, temporarily disable storage:

```ts
import { useCartStore } from '@/stores/cart-store'

describe('CartStore with persistence', () => {
  it('persists to localStorage', () => {
    // Set a storage mock
    const store = create<{ items: any[]; addItem: (i: any) => void }>()(
      persist(
        (set) => ({ items: [], addItem: (i) => set((s) => ({ items: [...s.items, i] })) }),
        { name: 'test-cart' }
      )
    )

    store.getState().addItem({ id: '1', name: 'Test' })
    
    // Read from the storage (default is localStorage)
    const stored = JSON.parse(localStorage.getItem('test-cart') ?? '{}')
    expect(stored.state?.items).toHaveLength(1)
  })
})
```

## Testing React Query Mutations

React Query mutations have different testing needs than queries — focus on success, error, and loading state:

```ts
// components/create-post.test.tsx
import { describe, it, expect, vi } from 'vitest'
import { render, screen, userEvent, waitFor } from '@testing-library/react'
import { CreatePostForm } from './create-post-form'
import * as actions from '@/app/actions'

// Mock the server action
vi.mock('@/app/actions', () => ({
  createPost: vi.fn(),
}))

describe('CreatePostForm — mutation', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('shows success and resets form on success', async () => {
    vi.mocked(actions.createPost).mockResolvedValue({ success: true })
    const user = userEvent.setup()

    render(<CreatePostForm />)

    await user.fill(screen.getByPlaceholderText(/title/i), 'My Post')
    await user.fill(screen.getByPlaceholderText(/content/i), 'Post content')
    await user.click(screen.getByRole('button', { name: /create post/i }))

    await waitFor(() => {
      expect(screen.getByPlaceholderText(/title/i)).toHaveValue('')
    })
  })

  it('shows error message on failure', async () => {
    vi.mocked(actions.createPost).mockResolvedValue({
      error: { root: ['Failed to create post'] },
    })
    const user = userEvent.setup()

    render(<CreatePostForm />)

    await user.fill(screen.getByPlaceholderText(/title/i), 'My Post')
    await user.fill(screen.getByPlaceholderText(/content/i), 'Post content')
    await user.click(screen.getByRole('button', { name: /create post/i }))

    await waitFor(() => {
      expect(screen.getByText(/failed to create post/i)).toBeInTheDocument()
    })
  })
})
```

### Testing React Query Error States

```ts
it('handles server action errors gracefully', async () => {
  vi.mocked(actions.createPost).mockRejectedValue(new Error('Network error'))
  const user = userEvent.setup()

  render(<CreatePostForm />)

  await user.fill(screen.getByPlaceholderText(/title/i), 'My Post')
  await user.click(screen.getByRole('button', { name: /create post/i }))

  await waitFor(() => {
    expect(screen.getByText(/something went wrong/i)).toBeInTheDocument()
  })
})
```

## Vitest 4 — Browser Mode (Stable)

Vitest 4.0 (Oct 21, 2025) marked Browser Mode as **stable** — real browser tests replace jsdom for components that need actual browser APIs (clipboard, layout, computed styles, IntersectionObserver, etc.). Setup is via a separate `vitest.browser.config.ts` so unit (jsdom) and browser tests can coexist.

### Init

```bash
# Adds vitest.browser.config.ts + @vitest/browser + framework adapter
npx vitest init browser
```

Choose: TypeScript → playwright → chromium → React → install Playwright browsers.

### `vitest.browser.config.ts`

```ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import { playwright } from '@vitest/browser-playwright'

export default defineConfig({
  plugins: [react()],
  test: {
    browser: {
      enabled: true,
      headless: true,
      provider: playwright(),
      instances: [{ browser: 'chromium' }],
      viewport: { width: 1280, height: 720 },
    },
  },
})
```

### Writing a Browser Mode test

```ts
// counter.browser.test.tsx
import { render } from 'vitest-browser-react'
import { expect, test } from 'vitest'
import { Counter } from './counter'

test('renders the initial count from a real browser', async () => {
  const screen = await render(<Counter initialCount={5} />)
  await expect.element(screen.getByRole('button', { name: /5/i })).toBeInTheDocument()
})
```

### Running

```bash
# Unit / integration tests (jsdom)
npm run test

# Browser mode tests (real Chromium via Playwright)
npm run test:browser          # alias to: vitest run --config=vitest.browser.config.ts
```

**When to use Browser Mode vs jsdom:**

| Need | Use |
|---|---|
| 99% of component logic, hooks, state | jsdom (default) — faster, no browser launch |
| Real layout / CSS / computed styles | Browser Mode |
| `IntersectionObserver`, `ResizeObserver`, `Clipboard` | Browser Mode |
| Visual regression testing (see next section) | Browser Mode (required) |
| `getBoundingClientRect`, `matchMedia` | Browser Mode |
| Service workers, Web Workers | Browser Mode |


### Browser Mode — Security Hardening (Vitest 4.1.0+)

The Browser Mode API is a privileged dev tool — it can write to project files, run arbitrary test files, and (in 4.1.7 and below) forward raw Chrome DevTools Protocol commands. **Three critical CVEs were published against it in May–June 2026** (CDP RCE 9.8, otelCarrier XSS 9.6, Windows file read 9.8). Full advisory breakdown is in `security.md` under "Vitest Browser Mode CVEs (May–June 2026)". Short version:

1. **Always run on vitest ≥ 4.1.8** (skill default is 4.1.9 — safe). The 4.1.8 patch closes the `cdp()` API bypass.
2. **Keep Browser Mode on localhost** — don't bind `--browser.api.host=0.0.0.0` in CI or dev containers. The browser API leaks the API token and project root, which is the attack surface for all three CVEs.
3. **Use the new `browser.api.allowWrite` / `allowExec` gates (4.1.0+)** in `vitest.browser.config.ts` for CI:

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    browser: {
      enabled: true,
      provider: playwright(),
      api: {
        // Default: true on localhost, false if host is bound to all interfaces.
        // In CI / shared environments, set explicitly to block cdp(),
        // saveTestFile, and rerun APIs.
        allowWrite: false,
        allowExec: false,
      },
    },
  },
})
```

If you need to share a Browser Mode session with a teammate, tunnel via SSH instead of binding to `0.0.0.0`.

## Vitest 4 — Visual Regression Testing

`toMatchScreenshot` ships in Vitest 4.0 — pixel-by-pixel screenshot diffs integrated into the test runner. Requires Browser Mode (above).

### Basic visual test

```ts
// components/hero.browser.test.tsx
import { render } from 'vitest-browser-react'
import { expect, test } from 'vitest'
import { Hero } from './hero'

test('hero section looks correct', async () => {
  const screen = await render(<Hero title="Hello" />)
  await expect(screen.getByTestId('hero')).toMatchScreenshot('hero-default')
})
```

### Global config — `vitest.browser.config.ts`

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    browser: {
      enabled: true,
      expect: {
        toMatchScreenshot: {
          comparatorName: 'pixelmatch',
          comparatorOptions: {
            threshold: 0.2,                   // 0–1, how different can colors be
            allowedMismatchedPixelRatio: 0.01, // 1% of pixels can differ
          },
        },
      },
    },
  },
})
```

### Per-test overrides

```ts
// More lax comparison for text-heavy elements (font hinting varies by OS)
await expect(btn).toMatchScreenshot('button-hover', {
  allowedMismatchedPixelRatio: 0.1,
})
```

### Workflow

1. **First run** — Vitest creates a baseline at `__screenshots__/<test>.<browser>-<platform>.png` and **fails** the test with a message: `"No existing reference screenshot found; a new one was created. Review it before running tests again."`
2. **Inspect** the baseline — make sure it looks right.
3. **Commit** the `__screenshots__/` folder to git.
4. **Subsequent runs** — Vitest captures the actual screenshot and diffs against the baseline. On mismatch, the report includes:
   - **Reference** (the expected baseline)
   - **Actual** (what was captured)
   - **Diff** (highlighted pixel differences — for visual triage)

### Updating baselines

```bash
# Re-record all baselines after intentional UI changes
vitest run --update --config=vitest.browser.config.ts
```

**Common pitfalls:**
- **Forget to commit `__screenshots__/`** — without baselines, every test fails on first CI run
- **Animations + non-deterministic rendering** — disable animations or use `prefers-reduced-motion` in tests
- **Font rendering differs by OS** — use `allowedMismatchedPixelRatio: 0.01` (1%) or higher for cross-OS CI
- **Don't run in parallel with viewport changes** — lock viewport in the config

## Vitest 4 — Playwright Trace Support

Vitest 4.0 can emit **Playwright Traces** for failed browser tests, so you can replay them in the Playwright Trace Viewer (`npx playwright show-trace trace.zip`).

```ts
// vitest.browser.config.ts
export default defineConfig({
  test: {
    browser: {
      enabled: true,
      provider: playwright(),
      trace: {
        mode: 'on-first-retry',   // 'on' | 'off' | 'retain-on-failure' | 'on-first-retry'
        attachments: true,        // include screenshots + snapshots
      },
    },
  },
})
```

When a test fails, the trace lives in `test-results/` — open it locally:
```bash
npx playwright show-trace test-results/my-test-chromium/trace.zip
```

## Vitest 3 → 4 Migration Notes

Most projects upgrade with **zero code changes** — the public API is the same. The big internal change is `vite-node` → Vite's **Module Runner** ([Vite docs](https://vite.dev/guide/api-environment-runtimes.html)).

### Breaking changes (action required)

1. **`VITE_NODE_DEPS_MODULE_DIRECTORIES` → `VITEST_MODULE_DIRECTORIES`** — env var rename
2. **`deps.optimizer.web` → `deps.optimizer.client`** — config key rename (and now any name can be used per-environment)
3. **`vitest/execute` entry point removed** — it was always internal
4. **Custom environments** — drop `transformMode`, add `viteEnvironment` instead
5. **`vitest/mocker` removed** — use `@vitest/mocker` directly
6. **No more `__vitest_executor` injection** — `moduleRunner` is injected instead (only matters for custom environments)

### New requirements

- **Vite ≥ 6.0.0**
- **Node.js ≥ 20.0.0** (Node 22 LTS / Node 24 LTS fine)

### Auto-migration

```bash
# Vitest can auto-rewrite your config
npx vitest migrate
```

### Upcoming (Vitest 5.0-beta)

5.0 removes the remaining deprecated entry points (`vitest/coverage`, `vitest/reporters`, `vitest/environments`, `vitest/snapshot`, `vitest/runners`, `vitest/suite`). Use `vitest/node` and `vitest/runtime` instead. The migration is mechanical — see [migration guide](https://main.vitest.dev/guide/migration).

## Common Mistakes


- **Testing implementation details** — test behavior, not how it's built
- **No `await` for async operations** — always `await` user events and async renders
- **Overusing `act()`** — usually a sign that something else is wrong
- **Not cleaning up mocks** — use `beforeEach` to clear mock state
- **E2E tests as the primary strategy** — they're slow; unit tests catch most bugs faster
- **Forgetting `await` for Server Components** — Server Components are `async`; use `render(await Component())`
- **Not mocking `'use cache'` functions** — these run on the server; mock them in tests or use a test database
- **Testing `use()` hook without mocking the Promise** — always ensure the Promise is properly mocked in the test environment
- **`useOptimistic` not reverting on test failure** — ensure your mock actions don't inadvertently succeed; reset mocks between tests
- **v4 auto-act in Zustand v5 tests** — v5 removed auto-`act()` wrapping; always wrap state updates in `act()` from `react-dom/test-utils`
- **Testing Zustand without resetting state** — always call `vi.clearAllMocks()` and recreate store instances between tests to prevent state leakage
- **Testing React Query mutations with only success cases** — always test error paths and loading states too
- **Visual regression tests without committed baselines** — Vitest creates a baseline on first run and the test fails; commit `__screenshots__/` to git or every CI run will fail
- **Running Browser Mode tests without `npx playwright install`** — provider needs the browser binaries; CI must run this first
- **Mixing `environment: 'jsdom'` with `browser.enabled: true`** — pick one per config file; browser mode does not use `environment`
- **Running Browser Mode on a network-exposed host (`--browser.api.host=0.0.0.0`)** — CVSS 9.8 RCE on vitest < 4.1.8 via the cdp() RPC + CDP `Page.setDownloadBehavior` chain. Keep Browser Mode on localhost, set `api.allowWrite: false, api.allowExec: false` in CI, upgrade to vitest ≥ 4.1.8. See `security.md` § Vitest Browser Mode CVEs.
- **Mixing vitest and @vitest/browser versions** — cdp() fix in 4.1.8 only protects if @vitest/browser is also ≥ 4.1.8. Pin them to matching versions: `"vitest": "4.1.9", "@vitest/browser": "4.1.9"`

**Sources:**
- [Testing Library docs](https://testing-library.com/docs/react-testing-library/intro/)
- [Vitest docs](https://vitest.dev/)
- [Vitest 4.0 announcement — VoidZero](https://voidzero.dev/posts/announcing-vitest-4) (Browser Mode stable, Visual Regression, Playwright Trace)
- [Vitest Browser Mode guide](https://vitest.dev/guide/browser/)
- [Vitest Visual Regression Testing](https://vitest.dev/guide/browser/visual-regression-testing)
- [Vitest 3 → 4 migration guide](https://vitest.dev/guide/migration.html)
- [Vitest 5 migration guide (beta)](https://main.vitest.dev/guide/migration)
- [Playwright docs](https://playwright.dev/)
- [Testing React 19 components](https://react.dev/learn/testing-react-components)
- [Vitest browser.api config — allowWrite / allowExec (4.1.0+)](https://main.vitest.dev/config/browser/api)
- [GHSA-g8mr-85jm-7xhm — Vitest Browser Mode CDP RCE (CVSS 9.8)](https://github.com/vitest-dev/vitest/security/advisories/GHSA-g8mr-85jm-7xhm)
