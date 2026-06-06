# Testing — Vitest + Playwright

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
npm install -D vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/jest-dom
```

### `vitest.config.ts`

```ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
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

**Sources:**
- [Testing Library docs](https://testing-library.com/docs/react-testing-library/intro/)
- [Vitest docs](https://vitest.dev/)
- [Playwright docs](https://playwright.dev/)
- [Testing React 19 components](https://react.dev/learn/testing-react-components)
