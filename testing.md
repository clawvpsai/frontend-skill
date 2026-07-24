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


## DOM Environment Updates — happy-dom 20.11.x + jsdom 29.1.1 (July 2026)

The two most-used DOM environments for Vitest got **meaningful updates in July 2026** that affect every test suite still on older versions. Both pin to their respective `@latest` dist-tags as of 2026-07-24.

### happy-dom 20.11.0 (July 18, 2026) — CookieStore API + Element.checkVisibility()

Released **one day after the original 20.10.6** — the most material feature batch in the happy-dom 20.x line:

1. **CookieStore API** (`window.cookieStore`) — first-class support for the async, [Storage Access API](https://developer.mozilla.org/en-US/docs/Web/API/Cookie_Store_API)-compatible cookie API. `await cookieStore.get(name)`, `await cookieStore.set(name, value)`, `change` event subscription, and the `subscribe(changes)` async iterator all work.

   ```ts
   // In a test (or any component code under Vitest + happy-dom):
   const cookie = await cookieStore.get('session')
   expect(cookie?.value).toBe('abc123')

   // Subscribe to cookie changes:
   cookieStore.addEventListener('change', (e) => {
     console.log('cookie changed:', e.changed)
   })
   ```

   **Why this matters:** code that uses the modern `window.cookieStore` API (often behind a feature flag with `document.cookie` fallback) now actually works in tests. Previously happy-dom would throw `cookieStore is not defined` and the test would only pass in real browsers.

2. **`Element.checkVisibility()`** — returns whether the element is rendered (not `display: none`, not `visibility: hidden`, not affected by `content-visibility: hidden`, etc.). The full [`Element.checkVisibility()` spec](https://developer.mozilla.org/en-US/docs/Web/API/Element/checkVisibility) is implemented, including the `{ contentVisibilityAuto, opacityProperty, visibilityProperty, checkOpacity, checkVisibilityCSS }` options.

   ```ts
   const hidden = document.querySelector('.modal').checkVisibility()
   // false if display: none, visibility: hidden, opacity: 0, or content-visibility: hidden
   ```

   **Why this matters:** libraries that check visibility before mounting (e.g., intersection observer polyfills, animation libraries) now work correctly. Previously the method was missing and tests fell back to `getComputedStyle` checks.

3. **Patch fixes bundled in 20.11.x:**
   - **20.11.1 (July 22, 2026)** — performance pass on query selectors by avoiding unnecessary `DOMException` construction (cyfung1031, task #2228). Most noticeable in tests that run thousands of `querySelector`/`querySelectorAll` calls in a single run.

**Recommended version:** `happy-dom@^20.11.1` (supersedes 20.10.6).

**Audit:**

```bash
# Who uses window.cookieStore in their component code or tests?
rg "cookieStore" --type ts --type tsx src/ tests/

# Who uses element.checkVisibility()?
rg "checkVisibility" --type ts --type tsx src/ tests/
```

### jsdom 29.1.1 (April 30, 2026) — stability + 29.1.x patch train

The **jsdom 29.x line** shipped 5 releases between February and April 2026 (`29.0.0` → `29.0.1` → `29.0.2` → `29.1.0` → `29.1.1`). The headline changes:

- **29.0.0 (March 15, 2026)** — Node 22+ minimum (drops Node 18 / 20 support), tightens WHATWG DOM conformance, updated CSS parser; lots of internal API changes for libraries that touched jsdom internals
- **29.1.0 (April 27, 2026)** — feature additions including `Element.checkVisibility()` parity with happy-dom's 20.11.0 (the two projects shipped within ~9 days of each other), better `getBoundingClientRect` accuracy for transformed elements
- **29.1.1 (April 30, 2026)** — bug fix for `AbortController` timing edge cases

**Recommended version:** `jsdom@^29.1.1` (supersedes 28.x and 29.0.x).

**Pin choice for new projects:**

| Scenario | Recommended environment |
|---|---|
| Most React / Next.js apps (server actions, server components, simple DOM) | `happy-dom` (faster, ~10× quicker than jsdom on cold-start) |
| Apps that need `getBoundingClientRect` precision + a fuller Web Platform implementation | `jsdom` (slower but more spec-accurate) |
| Tests that need real browser APIs (clipboard, layout, IntersectionObserver, Web Animations API) | **Vitest Browser Mode** (not a DOM environment at all — see Vitest 4 Browser Mode section above) |

**Audit:**

```bash
# Who still pins an old jsdom?
rg "\"jsdom\":" package.json package-lock.json pnpm-lock.yaml yarn.lock

# Who still pins an old happy-dom?
rg "\"happy-dom\":" package.json package-lock.json pnpm-lock.yaml yarn.lock
```

### Combined Audit Recipe (One-Shot)

```bash
# Run this in your repo to verify both deps are on the recommended versions
node -e "
  const p = require('./package.json')
  const dev = { ...p.devDependencies, ...p.dependencies }
  const happy = dev['happy-dom'] || 'n/a'
  const jsdom = dev['jsdom'] || 'n/a'
  console.log('happy-dom:', happy, happy.includes('20.11') ? '✅' : '⚠️ < 20.11')
  console.log('jsdom:    ', jsdom, jsdom.includes('29.1') ? '✅' : '⚠️ < 29.1.1')
"
```

**Sources:**
- [happy-dom 20.11.0 release notes](https://github.com/capricorn86/happy-dom/releases/tag/v20.11.0) — CookieStore API + Element.checkVisibility()
- [happy-dom 20.11.1 release notes](https://github.com/capricorn86/happy-dom/releases/tag/v20.11.1) — query selector perf
- [Element.checkVisibility() spec](https://developer.mozilla.org/en-US/docs/Web/API/Element/checkVisibility)
- [Cookie Store API spec](https://developer.mozilla.org/en-US/docs/Web/API/Cookie_Store_API)
- [jsdom 29.1.0 release notes](https://github.com/jsdom/jsdom/releases/tag/29.1.0)
- [jsdom 29.1.1 release notes](https://github.com/jsdom/jsdom/releases/tag/29.1.1)


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

## Vitest 4 — `fsModuleCache` Promoted From `experimental.*` to Top-Level Option (July 20, 2026, PR [#10734](https://github.com/vitest-dev/vitest/pull/10734) by sheremet-va, merged 2026-07-20T08:04:04Z, fixes [#10701](https://github.com/vitest-dev/vitest/issues/10701))

**`experimental.fsModuleCache`** and **`experimental.fsModuleCachePath`** are MOVED to top-level **`test.fsModuleCache`** and **`test.fsModuleCachePath`**. The cache directory defaults to `<workspaceRoot>/node_modules/.vitest-cache` (a single workspace-root directory shared by every project in a workspace), `fsModuleCache` defaults to `false` (off by default — opt-in for now), and the old `experimental.*` options are migrated with a deprecation warning. This unblocks **Vitest 5** stabilization — issue #10701 confirmed "We haven't received any issues regarding this feature, and we have also been running tests with this flag enabled for a while now" so the core team is comfortable promoting it. **No Vitest 4 patch has shipped yet** — the promotion is on Vitest 4 `main` and will land in the next Vitest 4 patch (probably 4.1.11 in the coming days). Vitest 5.0 stable is still `beta.6` (last released 2026-07-06) and no `beta.7` has shipped yet.

### What is `fsModuleCache`?

A persistent on-disk cache of *transformed* (Vite-transformed) source modules. When `fsModuleCache: true`, Vitest serializes every transformed module's output to `<cacheDir>/<hash>.json` (or wherever `fsModuleCachePath` points) on first load. On subsequent runs, the cached transformed output is reused without re-running the Vite plugin chain (TypeScript, JSX, PostCSS, etc.) — large monorepos see 5–10× faster cold-start times when the cache is warm.

### Migration

**Before (Vitest 4.x with `experimental.*`):**

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    experimental: {
      fsModuleCache: true,
      fsModuleCachePath: './node_modules/.vitest-cache',  // optional override
    },
  },
})
```

**After (Vitest 4.1.11+ / Vitest 5 stable, top-level):**

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    fsModuleCache: true,                                    // opt-in, default false
    fsModuleCachePath: './node_modules/.vitest-cache',     // optional override
    // OR omit fsModuleCachePath entirely to use the default
    // <workspaceRoot>/node_modules/.vitest-cache
  },
})
```

**Audit recipe:**

```bash
# Find projects that need migration
rg "experimental\.fsModuleCache" vitest.config.* test/vitest.config.* -l
```

**Migration impact:**

- **Behavior:** identical — same cache file format, same default cache path, same on-disk layout. The migration is purely schema.
- **Default change:** `fsModuleCache` was always `false` under `experimental.*` (opt-in), and remains `false` at the top level (still opt-in). **You have to set `fsModuleCache: true` explicitly to get the speedup.**
- **Cache location change:** the default moves from `node_modules/.cache/vitest` (the experimental default) to `<workspaceRoot>/node_modules/.vitest-cache` (the new top-level default). First run after upgrade will rebuild the cache.
- **Monorepo behavior:** the new default `<workspaceRoot>/node_modules/.vitest-cache` is a *single workspace-root directory* shared by every project — if you want per-project isolation, set `fsModuleCachePath` to a project-relative path inside each project's `vitest.config.ts`.
- **CI behavior:** the cache survives across CI runs as long as the cache directory is restored (configure your CI's cache step to preserve `**/node_modules/.vitest-cache` or your custom `fsModuleCachePath`).
- **Vitest 5 migration:** when Vitest 5 ships stable, `fsModuleCache` may flip to `true` by default (TBD — track [issue #10701](https://github.com/vitest-dev/vitest/issues/10701)). Setting it explicitly now means no behavior change on upgrade.

### When to use `fsModuleCache: true`

| Project shape | Recommendation |
|---|---|
| **Small app / single project** (< 1,000 test files) | Leave at default `false` — overhead of managing the cache outweighs the speedup. |
| **Large monorepo** (5+ projects, 5,000+ test files) | Set `fsModuleCache: true` + keep the default path. Expect 5–10× cold-start improvement with warm cache. |
| **CI on ephemeral runners** | Set `fsModuleCache: true` only if your CI restores `**/node_modules/.vitest-cache` between runs; otherwise the cache is rebuilt every run (no harm, but no speedup). |
| **Watch-mode development** | Leave at default `false` — the cache adds complexity to HMR-style incremental runs; the default behavior is well-tested. |

### Common mistakes

- **Setting `fsModuleCache: true` without excluding it from git** — add `**/node_modules/.vitest-cache` (or your custom path) to `.gitignore` if it lives outside `node_modules` (the default path is already inside `node_modules` so it's ignored by default; only matters if you override `fsModuleCachePath` to a non-`node_modules` location).
- **Sharing one `fsModuleCachePath` across incompatible Vitest versions** — the cache format is version-pinned; if you have a monorepo with mixed Vitest versions, set per-project `fsModuleCachePath` to avoid cross-version cache corruption.
- **Forgetting to bust the cache after a Vite plugin upgrade** — if you bump `@vitejs/plugin-react` or `vite` itself, delete the cache directory or set `fsModuleCache: false` for the next run.
- **Setting `fsModuleCache: true` for the first time on a project that has flaky tests** — the cache will mask some test isolation issues because stale transformed modules can be served. If you see tests pass inconsistently after enabling, the cache is hiding a real bug — clear it, fix the bug, re-enable.

**Sources:**
- [PR #10734 — `feat: promote fsModuleCache to a top-level option`](https://github.com/vitest-dev/vitest/pull/10734)
- [Issue #10701 — `Stabilize fsModuleCache`](https://github.com/vitest-dev/vitest/issues/10701)
- [Vitest docs — `test.fsModuleCache` (main, pre-5.0)](https://main.vitest.dev/config/fsmodulecache)
- [Vitest docs — `test.fsModuleCachePath` (main, pre-5.0)](https://main.vitest.dev/config/fsmodulecachepath)
- [Vitest docs — `experimental.fsModuleCache` (current, deprecated after 4.1.11)](https://vitest.dev/config/experimental.html#experimental-fsmodulecache)

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

### Upcoming (Vitest 5.0-beta.6, July 6, 2026)

Vitest 5.0 is on beta.6 (published 2026-07-06T06:52:52Z). Four beta releases have shipped between May 19 and July 6, 2026 (beta.3 on May 19, beta.4 on June 1, beta.5 on June 15, **beta.6 on July 6**) and the breaking-change list has grown materially. **The list below supersedes anything earlier.** If you pin to a specific beta, check the inline `[#PR]` links to verify the change is still in your range. The migration guide is the canonical reference: https://main.vitest.dev/guide/migration

#### New breaking changes in beta.6 (July 6, 2026) — **MUST READ before bumping**

**1. `screenshotDirectory` config for `browser.expect.toMatchScreenshot` — [#10592](https://github.com/vitest-dev/vitest/issues/10592)** by @macarie

A new `screenshotDirectory` config option for `browser.expect.toMatchScreenshot`. If you set the screenshot output directory in your Visual Regression tests, the field is renamed / made required. The default is no longer implicit.

```ts
// vitest.browser.config.ts
export default defineConfig({
  test: {
    browser: {
      expect: {
        toMatchScreenshot: {
          // NEW (beta.6) — explicit path:
          screenshotDirectory: './test/screenshots',
        },
      },
    },
  },
})
```

**2. `vi.clearMocks()` runs by default before each test — [#10613](https://github.com/vitest-dev/vitest/issues/10613)** by @sheremet-va

Mocks are now cleared by default before every test (analogous to `clearMocks: true`). This changes test behavior for any suite that relied on mock state persisting between tests.

```ts
// Before (beta.5 and earlier) — mock call counts persist between tests:
test('a', () => { vi.mocked(fn).mockReturnValue(1); expect(fn).toHaveBeenCalled() })
test('b', () => { /* fn.mock.calls is still 1 from test 'a' */ })

// After (beta.6+) — call counts auto-cleared:
//   If you relied on this behavior, add `clearMocks: false` in config
//   or wrap state setup in beforeEach().
```

**3. JSON / JUnit / HTML reporter output defaults to `.vitest/` — [#10621](https://github.com/vitest-dev/vitest/issues/10621) + [#10620](https://github.com/vitest-dev/vitest/issues/10620)**

Reporter output files are now written to `.vitest/` (not the project root):

| Reporter | Before (beta.5) | After (beta.6+) |
|---|---|---|
| `json` | `./vitest-results.json` | `./.vitest/vitest-results.json` |
| `junit` | `./junit.xml` | `./.vitest/junit.xml` |
| `html` (UI) | `./html/` | `./.vitest/html/` |

**4. `webdriverio` provider removed — [#10675](https://github.com/vitest-dev/vitest/issues/10675)** by @sheremet-va

The `webdriverio` provider package is no longer published as part of `@vitest/browser`. If you were using it as a Browser Mode provider, switch to `playwright` (recommended), `preview`, `puppeteer`, or the (still experimental) `native playwright` provider.

```ts
// vitest.browser.config.ts
export default defineConfig({
  test: {
    browser: {
      // ❌ No longer available:
      // provider: 'webdriverio',
      // ✅ Switch to:
      provider: 'playwright',  // or 'preview' / 'puppeteer'
    },
  },
})
```

**5. `@sinonjs/fake-timers` updated — supports mocking `Temporal` — [#10654](https://github.com/vitest-dev/vitest/issues/10654)**

The bundled `@sinonjs/fake-timers` is updated. New: full `Temporal` API mocking support (`Temporal.Now`, `Temporal.PlainDate`, etc.). If you have a custom `vi.useFakeTimers` config that referenced internal sinon fields, audit it.

**6. Node 26: no more `localStorage` warnings — [#10293](https://github.com/vitest-dev/vitest/issues/10293)**

`localStorage` warnings are no longer emitted on Node 26 (they were noisy false-positives on the latest Node). If you were silencing them with `onConsoleLog`, you can remove the filter. Worker startup also fails gracefully instead of crashing on transient errors.

**7. UI: API access hardened — [#10583](https://github.com/vitest-dev/vitest/issues/10583)**

The Vitest UI API endpoint tightens access controls. If you run the UI exposed on a non-localhost host, the hardening restricts endpoints that were previously accessible. Combine with `api.allowWrite: false, api.allowExec: false` (already required by the CVSS 9.8 RCE advisory, GHSA-g8mr-85jm-7xhm) when exposing the UI to a network.

**8. New: `vi.when()` helper — [#10174](https://github.com/vitest-dev/vitest/issues/10174)** by @macarie

A new `vi.when()` helper for conditional mocking based on test environment / arguments. Use it for environment-aware mocks (e.g., a stub that returns different values in jsdom vs. node). Non-breaking — opt-in.

**Sources for beta.6:**
- [Vitest 5.0.0-beta.6 release notes — July 6, 2026](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.6)
- [Vitest 5 migration guide (beta)](https://main.vitest.dev/guide/migration)

#### Cross-version bug fixes in 4.1.10 + 3.2.7 (July 6, 2026)

Two security / correctness fixes backported from beta.6 to the stable v4 + v3 lines:

- **Browser Mode: fs access check in builtin commands — [#10680](https://github.com/vitest-dev/vitest/issues/10680)** — vitest 4.1.10
- **vm: external module resolve error with deps optimizer query for encoded URI — [#10661](https://github.com/vitest-dev/vitest/issues/10661)** — vitest 4.1.10
- **Browser Mode: fs access check in builtin commands — [#10679](https://github.com/vitest-dev/vitest/issues/10679)** — vitest 3.2.7

**Action:** bump `vitest` and `@vitest/browser` to 4.1.10 (or 3.2.7 for legacy projects) when running in CI on a shared runner. The fs-access check closes a class of unintended file-system reads through Browser Mode's `cdp()` and builtin commands.

#### Version pinning guidance (updated July 6, 2026)

```jsonc
// package.json
{
  "devDependencies": {
    // For most projects (stable):
    "vitest": "4.1.10",
    "@vitest/browser": "4.1.10",
    "@vitest/coverage-v8": "4.1.10",

    // For early-adopter projects (beta; track breaking changes):
    // "vitest": "5.0.0-beta.6",
    // "@vitest/browser": "5.0.0-beta.6",

    // For legacy Next.js 14 / Vite 5 projects:
    // "vitest": "3.2.7"
  }
}
```

### Hard requirements (beta.3 — [#10178](https://github.com/vitest-dev/vitest/issues/10178))

- **Node.js ≥ 22** (Node 20 is no longer supported)
- **Vite ≥ 6.4** (matches Vitest 4's hard floor of Vite 6)

If you previously upgraded to Vitest 4 because Node 20 was still allowed, plan to bump Node to 22 LTS (or 24 LTS) before Vitest 5 ships stable.

### Removed deprecated entry points (beta.3 + migration guide)

| Removed | Use instead |
|---|---|
| `vitest/coverage` | `vitest/node` |
| `vitest/reporters` | `vitest/node` |
| `vitest/environments` | `vitest/runtime` |
| `vitest/snapshot` | `vitest/runtime` |
| `vitest/runners` | `TestRunner` from `vitest` |
| `vitest/suite` | static methods on `TestRunner` from `vitest` (e.g. `TestRunner.getCurrentTest()`) |
| `vitest/mocker` | `@vitest/mocker` package directly (the standalone package was always published, `vitest/mocker` was removed) |
| `vitest/internal/module-runner` | (no replacement — was internal) |

```ts
// Before (Vitest 4)
import { coverageConfigDefaults } from 'vitest/coverage'

// After (Vitest 5)
import { coverageConfigDefaults } from 'vitest/node'
```

### `expect.poll` now fails on timeout (beta.3 — [#10233](https://github.com/vitest-dev/vitest/issues/10233))

Previously a `expect.poll(fn)` that never resolved would hang until Vitest killed it. It now **fails the test with a timeout error** if `fn` does not resolve in time. Practical impact: if you were relying on `expect.poll` to keep polling past the timeout to surface async state to a debugger, you'll need a different strategy. Most usages just need to confirm the timeout is configured correctly (`{ timeout: 5_000 }` or whatever your test needs).

### Strict `toHaveTextContent` + new `toMatchTextContent` (beta.4 — [#10473](https://github.com/vitest-dev/vitest/issues/10473))

Browser Mode's `toHaveTextContent` matcher used to do a **partial, case-sensitive substring match** and accepted `RegExp`. In 5.0 it does **strict equality** and rejects regex. The old behaviour (including `RegExp` support) is moved to `toMatchTextContent`.

```ts
// Before (Vitest 4 — partial match, accepts regex)
await expect(element).toHaveTextContent(/hello/i)
await expect(element).toHaveTextContent('world')  // matched "Hello World" too

// After (Vitest 5 — strict, no regex)
await expect(element).toMatchTextContent(/hello/i)  // use the new matcher for regex
await expect(element).toHaveTextContent('Hello World')  // exact string only
```

If your suite relied on the old "matches if the element contains the string" behaviour, plan to either tighten assertions to exact strings or switch the affected calls to `toMatchTextContent` during the 5.0 migration.

### Browser Mode: `locators.exact: true` by default (beta.4 — [#10430](https://github.com/vitest-dev/vitest/issues/10430))

`page.getByText('Hello, World')` used to be a fuzzy match — it would find elements containing that substring or matching it case-insensitively in many cases. In 5.0 the `exact` flag is `true` by default, matching only the exact string. If you have tests like `getByText('Submit')` that previously matched a button labelled `Submit Order`, they will now fail.

```ts
// Before (Vitest 4 — fuzzy)
const submit = page.getByText('Submit')  // matches "Submit", "Submit Order", "Submitted"

// After (Vitest 5 — strict by default)
const submit = page.getByText('Submit', { exact: true })  // exact match only
const submit = page.getByText('Submit', { exact: false }) // opt back into fuzzy
```

Audit your `getByText` / `getByRole` / `getByLabel` calls before bumping.

### Browser Mode: nested mark trace in UI (beta.5 — [#10437](https://github.com/vitest-dev/vitest/issues/10437))

New UI feature — not breaking, but worth knowing: `page.mark` and `context.mark` calls now display as a **nested trace** in the Vitest UI, with custom commands expandable in the test panel. If you were using `page.mark` for custom debug breadcrumbs, they'll now show up hierarchically rather than flat.

### Browser Mode: `sessionId` required for orchestrator HTML request (beta.5 — [#10522](https://github.com/vitest-dev/vitest/issues/10522))

The Vitest browser orchestrator's HTML request (the one the test runner polls to discover tests) **now requires a `sessionId`** in 5.0. If you have any custom integrations that hit the orchestrator endpoint (custom reporters, dashboard bridges, CI-side test selection), they'll start getting 400s after upgrading. Pass a session ID via the standard Vitest browser API or the `VITEST_BROWSER_SESSION_ID` env var.

### `thresholds.perFile` accepts an object (beta.5 — [#10190](https://github.com/vitest-dev/vitest/issues/10190))

Previously `coverage.thresholds.perFile` was a boolean. In 5.0 it can be `true | false | { lines, statements, branches, functions }` — letting you opt into per-file checking for specific metrics only. Glob coverage thresholds **no longer inherit `perFile`** from the top-level config — set `perFile` explicitly on each glob that needs it.

```ts
// vitest.config.ts (Vitest 5)
export default defineConfig({
  test: {
    coverage: {
      thresholds: {
        perFile: { lines: true, statements: true },  // only these metrics checked per-file
        'src/critical/**': { lines: 95, perFile: true },  // glob needs its own perFile
        'src/legacy/**': { lines: 60 },                   // no perFile, top-level is no longer inherited
      },
    },
  },
})
```

### No more config lookup from ancestor directories (beta.5 — [#10428](https://github.com/vitest-dev/vitest/issues/10428))

Vitest historically walked up the directory tree looking for a `vitest.config.*` file. In 5.0 it **only looks in the project root** (the directory you ran `vitest` from, or the `root` you configured). Practical impact: monorepo packages with a config two levels up, or workspaces where `vitest` was previously invoked from a sub-package and picked up the root config, will silently start using the in-package config (or failing with "no config found"). Set `root` explicitly per-project, or run `vitest --config path/to/config.ts` from the package you want to test.

### `@vitest/runner` inlined — package no longer published (beta.5 — [#10511](https://github.com/vitest-dev/vitest/issues/10511))

`@vitest/runner` is **no longer a separate package on npm**. Its types and runtime are merged into `vitest` itself. If your `package.json` lists `@vitest/runner` as a direct dependency (uncommon, but it happened), remove it and rely on `vitest`'s re-exports. The migration guide documents the replacement imports — `TestRunner` is now exported from `vitest` directly.

### `TestModule` diagnostics: `concurrencyId` / `workerId` exposed, `id` is 1-based (beta.5 — [#10516](https://github.com/vitest-dev/vitest/issues/10516))

`TestModule` now exposes `concurrencyId` and `workerId` directly on the diagnostics object, and the `id` field is **1-based instead of 0-based**. If you have any custom reporters or test-isolation logic that filters by `module.id === 0` to detect "the first test", that filter will now miss everything.

### Happy-dom / jsdom window mutation allowed (beta.5 — [#10373](https://github.com/vitest-dev/vitest/issues/10373))

Vitest 4 threw if you mutated the `window` object provided by `happy-dom` or `jsdom`. In 5.0 that restriction is relaxed — useful for libraries that patch `window.fetch`, `window.matchMedia`, or similar globals during test setup. Not a breaking change unless your code relied on the throw for debugging; if so, replace it with an explicit assertion.

### CLI: `--repeats` option (beta.5 — [#10504](https://github.com/vitest-dev/vitest/issues/10504))

New flag to rerun the entire suite N times in a single invocation — handy for flake hunting. Not a breaking change.

### Removed: `test.sequential`, `describe.sequential`, `sequential` option (migration guide)

The deprecated `sequential` options are removed. To opt a test or suite out of inherited / global concurrency, use `concurrent: false` instead.

```ts
// Before (Vitest 4)
test.sequential('runs alone', async () => { /* ... */ })

// After (Vitest 5)
test('runs alone', { concurrent: false }, async () => { /* ... */ })
```

### Hoistable methods outside top-level scope throw (beta.4 — [#10460](https://github.com/vitest-dev/vitest/issues/10460))

Methods that Vitest hoists (`vi.mock`, `vi.hoisted`, `vi.doMock`, etc.) used to be tolerated if called inside `describe` or `beforeEach`. In 5.0 they **throw at module evaluation time**. The codemod handles most cases; the failure mode is loud so you'll know immediately if your setup needs re-ordering.

### Benchmarking API rewrite (beta.4 — [#10113](https://github.com/vitest-dev/vitest/issues/10113))

`bench` is **no longer a top-level import from `vitest`** — it's now a test-context fixture accessed from inside a regular `test()`. The old `bench.skip`, `bench.only`, `bench.todo`, `benchmark.reporters`, `benchmark.outputFile`, `benchmark.compare`, `benchmark.outputJson`, and the `--compare` / `--outputJson` CLI flags are all removed. `Vitest` instance `mode` is always `'test'` now (the old `'benchmark'` value is gone — benchmarks run inside a dedicated project of the same instance).

```ts
// Before (Vitest 4)
import { bench, describe } from 'vitest'

describe('sort', () => {
  bench('Array.sort', () => { /* ... */ })
})

// After (Vitest 5)
import { test } from 'vitest'

test('sort - Array.sort', ({ bench }) => {
  bench('Array.sort', () => { /* ... */ })
})
```

If your project uses `vitest bench` as a CI gate, the new API requires updating both the test definitions and the reporter config (`test.reporters` / `--reporter=json --outputFile=<path>` for benchmark capture).

### Upgrading on a real codebase

1. **Audit `getByText` / `getByRole` calls** for substring matches before bumping — strict-by-default will surface these as immediate failures.
2. **Audit `toHaveTextContent`** for partial matches and regex arguments.
3. **Move `vi.mock` / `vi.hoisted` calls** to the top level if any are inside `describe` or hooks — beta.4's hoistable-methods-out-of-scope error will surface them at module load.
4. **Replace `vitest/coverage`, `vitest/environments`, `vitest/snapshot`, `vitest/runners`, `vitest/suite`, `vitest/reporters`, `vitest/mocker` imports** with the `vitest/node` / `vitest/runtime` equivalents.
5. **Replace `test.sequential`** with `{ concurrent: false }`.
6. **Bump Node to 22 LTS or 24 LTS** before installing.
7. **Rewrite benchmark suites** if you use `vitest bench` — the API moved inside `test()`.
8. **Set `root` explicitly** in monorepo workspaces — ancestor-directory config lookup is gone.
9. **Remove `@vitest/runner`** from `dependencies` / `devDependencies` if you had it pinned directly.

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
- **Mixing vitest and @vitest/browser versions** — cdp() fix in 4.1.8+ only protects if @vitest/browser is also at the matching version. Pin them together: `"vitest": "4.1.10", "@vitest/browser": "4.1.10"` (4.1.10 closed a second class of fs-access via builtin commands — backport of beta.6's #10680).

## `@next/playwright` — `instant()` Test Helper for Instant Navigations (Next.js 16.3)

Next.js 16.3 ships **`@next/playwright`**, a first-party Playwright helper that lets you assert what must be **instantly visible** after a navigation — without waiting for the network. This is the E2E complement to Instant Insights in dev: write a test once, and catch regressions to your instant routes in CI.

```ts
import { expect, test } from '@playwright/test'
import { instant } from '@next/playwright'

test('product title is available immediately after navigation', async ({ page }) => {
  await page.goto('/products/shoes')

  // Assert what's visible WITHOUT waiting for the network
  await instant(page, async () => {
    await page.click('a[href="/products/hats"]')
    // The shell renders instantly — these assertions fire immediately
    await expect(page.locator('h1')).toContainText('Baseball Cap')
    await expect(page.getByText('Checking inventory...')).toBeVisible()
  })

  // After instant() scope exits, deferred content resolves
  await expect(page.getByText('12 in stock')).toBeVisible()
})
```

**How it works:** `instant()` sets up a `cacheComponents`-aware navigation scope. Inside the callback, clicking a link triggers an **instant navigation** (shell-only RTT + prefetch), and assertions run against the shell immediately — no network wait. After the callback exits, Playwright resumes normal (full-payload) navigation, so deferred content assertions work normally.

**Use `instant()` when you want to:**
- Verify the shell renders the right static content immediately after a click
- Catch regressions where a Suspense boundary is missing and the shell is blank
- Assert that loading skeletons / pending states are visible before deferred data arrives

**Do not use `instant()` for:**
- Testing full page loads from cold (use `page.goto()` instead)
- Testing routes that don't use `export const instant` — the helper is a no-op outside the instant nav path

**Source:** [Next.js 16.3 — Instant Navigations blog post](https://nextjs.org/blog/next-16-3-instant-navigations) · [`@next/playwright` — Instant Navigation Testing API docs](https://nextjs.org/docs/app/guides/testing/instant-navigation)

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
- [Vitest 5.0.0-beta.5 release notes — June 15, 2026](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.5)
- [Vitest 5.0.0-beta.4 release notes — June 1, 2026](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.4)
- [Vitest 5.0.0-beta.3 release notes — May 19, 2026](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.3)
- [Vitest 5.0.0-beta.6 release notes — July 6, 2026](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.6)
- [Vitest 4.1.10 release notes — July 6, 2026](https://github.com/vitest-dev/vitest/releases/tag/v4.1.10)
- [Vitest 3.2.7 release notes — July 6, 2026](https://github.com/vitest-dev/vitest/releases/tag/v3.2.7)
