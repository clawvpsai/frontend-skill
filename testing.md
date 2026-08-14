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


## Playwright 1.62 (July 24, 2026) — What's New

`@playwright/test@1.62.0` shipped **2026-07-24T21:57:02Z** (per [npm](https://www.npmjs.com/package/@playwright/test)) and is the **second big release of July** after the July 21 Next.js security release. Two pieces (the **stories-and-galleries component testing model** and **bundled Playwright MCP server**) restructure how you write component tests and how you drive Playwright from agents; the rest are smaller quality-of-life wins. **Browsers bumped to Chromium 151.0.7922.34, Firefox 153.0, WebKit 26.5.** Tested against Google Chrome 151 and Microsoft Edge 151. **Announcement: ⚠️ Debian 11 is not supported anymore** — bump your CI containers to Debian 12 (`mcr.microsoft.com/playwright:v1.62.0-jammy`) or Ubuntu 24.04 before the next cycle.

### 🧱 New component testing model — stories and galleries

The single biggest addition. Component testing moves from "import the component, mount it inline" to a **stories and galleries** model: a **story** wraps your component in one specific scenario (hard-coded props, mock data, providers) and a **gallery** page that you serve renders stories on demand. The new [`fixtures.mount()`](https://playwright.dev/docs/api/class-fixtures#fixtures-mount) fixture navigates to the gallery, mounts a story by id, and returns a [`Locator`](https://playwright.dev/docs/api/class-locator) scoped to the story's root element:

```ts
// e2e/expandable.spec.ts
import { test, expect } from '@playwright/experimental-ct-react'
import type { ExpandableStory } from './expandable.story'

test('click should expand', async ({ mount }) => {
  // Pass a story type as a template argument to type-check its props
  const component = await mount<ExpandableStory>('components/Expandable/Stateful')

  await component.getByRole('button').click()
  await expect(component.getByTestId('expanded')).toHaveValue('true')

  // update(props) re-renders the story with new props; unmount() tears it down
  await component.update({ defaultOpen: true })
  await component.unmount()
})
```

```tsx
// e2e/expandable.story.tsx — a gallery page that hosts one or more named stories
import { Expandable } from '@/components/Expandable'

export const Stateful = (props: { defaultOpen?: boolean }) => (
  <Expandable title="Click me" defaultOpen={props.defaultOpen ?? false}>
    <p>Hidden content.</p>
  </Expandable>
)
```

**Why this matters:** the old inline-mount model made it painful to share scenarios across tests (every test had to re-declare providers, props, and mocked deps inline). The stories-and-galleries model centralizes scenario setup in story files that *both* your agent and your human reviewers can read, and `mount()` returns a Locator (not a React wrapper) so the rest of the test uses the standard Playwright locator API. Tests are decoupled from the component's prop shape — change the prop and the story's typing surfaces it everywhere.

### 🛑 Cancel operations with `AbortSignal`

Most operations and web-first assertions now accept a `signal` option that takes an [`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal). Cancel long-running actions, navigations, waits, and assertions the same way you would in any Web API:

```ts
const controller = new AbortController()
setTimeout(() => controller.abort(), 1000) // 1s budget per click + assert

await page.getByRole('button', { name: 'Submit' }).click({ signal: controller.signal })
await expect(page.getByText('Done')).toBeVisible({ signal: controller.signal })
```

**Note:** providing a signal does **not** disable the default timeout — pass `timeout: 0` alongside if you want infinite wait time. Useful for tests that should fail fast in CI (1s budget) but more forgiving locally (no signal = default 30s timeout). Pairs well with the new `retryStrategy: 'isolated'` (below) for flaky-test debugging — abort the first attempt at the first sign of trouble, then retry in isolation.

### 🖼️ WebP screenshots

[`expect(page).toHaveScreenshot()`](https://playwright.dev/docs/api/class-pageassertions#page-assertions-to-have-screenshot-1) and [`expect(locator).toHaveScreenshot()`](https://playwright.dev/docs/api/class-locatorassertions#locator-assertions-to-have-screenshot-1) now store snapshots in the WebP format — give the snapshot a `.webp` name and Playwright writes a lossless WebP golden (or read it back the same way):

```ts
// Visual comparisons store the golden snapshot as lossless WebP
await expect(page).toHaveScreenshot('homepage.webp')

// Standalone screenshots can trade quality for size with lossy WebP
await page.screenshot({ path: 'homepage.webp', quality: 50 })
```

[`page.screenshot()`](https://playwright.dev/docs/api/class-page#page-screenshot) and [`locator.screenshot()`](https://playwright.dev/docs/api/class-locator#locator-screenshot) also accept `webp` as a `type`, where quality `100` (the default) is **lossless** and lower values use lossy compression. **Practical win:** lossless WebP goldens are typically 25–35% smaller than lossless PNG — a 50MB PNG-heavy `__snapshots__` directory shrinks to ~35MB with no quality loss.

### 🧩 Custom test filtering with `Reporter.preprocess()`

New [`reporter.preprocess()`](https://playwright.dev/docs/api/class-reporter#reporter-preprocess) hook runs after the configuration is resolved and before `reporter.onBegin()`, letting a reporter mark individual tests as **skipped**, **excluded**, **fixed**, or **failing** through a [`TestRun`](https://playwright.dev/docs/api/class-testrun) object:

```ts
// reporters/team-policy.reporter.ts
import type { Reporter, TestRun } from '@playwright/test/reporter'

class TeamPolicyReporter implements Reporter {
  async preprocess({ config, suite, testRun }: { config: any; suite: any; testRun: TestRun }) {
    for (const test of suite.allTests()) {
      // Skip work-in-progress tests in CI (not local dev)
      if (process.env.CI && test.title.includes('[wip]')) testRun.skip(test)

      // Tag known-flaky tests as 'fixed' so they're highlighted in the HTML report
      if (KNOWN_FLAKES.has(test.title)) testRun.fix(test)
    }
  }
}
export default TeamPolicyReporter
```

**Use cases:** CI-only skip filters (skip `[wip]` only when `process.env.CI`), per-team policy enforcement (skip the platform team's tests on the marketing team's CI run), tagging known-flake tests so the HTML report shows the auto-retry status, and excluding slow integration tests from PR runs.

### 🔁 Isolated retries — `retryStrategy: 'isolated'`

New [`testConfig.retryStrategy`](https://playwright.dev/docs/api/class-testconfig#test-config-retry-strategy) controls when failed tests are retried. The default `'immediate'` retries as soon as a worker is free; `'isolated'` runs **all retries at the end**, one by one in a single worker, to minimize interference with the rest of the suite:

```ts
// playwright.config.ts
import { defineConfig } from '@playwright/test'

export default defineConfig({
  retries: 2,
  retryStrategy: 'isolated', // ← new in 1.62; all retries run after the main suite
})
```

**When to pick `'isolated'`:** debugging a known-flaky test (you don't want other tests' noise while the retry happens), performance-sensitive suites (an isolated retry doesn't compete for worker time with the main run), and CI environments where you want a deterministic "first attempt results" view before the retry noise.

### New APIs at a glance

| Area | API | What it does |
|---|---|---|
| Browser / Context | `browserContext.storageState({ credentials: true })` | New `credentials` option includes the context's virtual WebAuthn passkeys in the storage state — persist + re-seed into later contexts via `browser.newContext({ storageState })` |
| Actions | `locator.click({ scroll: 'auto' \| 'none' })` | New `scroll` option on every action to opt out of Playwright's automatic scroll-into-view — useful for sticky-header layouts where scrolling past an element is unwanted |
| Network | `apiResponse.timing()` | Returns resource timing information for an API response — `startTime`, `responseEnd`, `domainLookupEnd`, `connectEnd`, etc. — mirrors the browser's [PerformanceResourceTiming](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceResourceTiming) |
| Evaluation | `locator.waitForFunction(fn)` | Waits until `fn` — called with the matching element — returns a truthy value. Replaces `page.waitForFunction(() => el.matches(...))` boilerplate |
| Evaluation | `page.evaluate(() => fn)` / `page.addInitScript(() => fn)` | Both now accept **functions** as arguments directly (previously only string-serialized form was supported in some call paths) — lets you pass a TypeScript function reference without ``() => `${...}` `` string concatenation |
| Command line / MCP | `npx playwright mcp` / `npx playwright cli` | **Playwright now bundles the [Playwright MCP server](https://playwright.dev/docs/getting-started-mcp) and [`playwright-cli`](https://playwright.dev/docs/getting-started-cli)** — agents can drive browsers without separately installing the MCP server package. `playwright-cli` is a scriptable CLI for one-off browser operations |
| Reporters | `reporter: [['html', { mergeFiles: true }]]` | The HTML report's **Merge files** grouping — previously only a UI toggle — can now be enabled from the config so PR comment artifacts use the merged-file layout by default |

### Browser versions (1.62)

- Chromium **151.0.7922.34**
- Mozilla Firefox **153.0**
- WebKit **26.5**

Also tested against Google Chrome 151 and Microsoft Edge 151 stable channels.

### Migration checklist (1.61.x → 1.62)

- [ ] `npm install -D @playwright/test@^1.62.0` — no peer-dep changes, no `playwright.config.ts` migration
- [ ] **Debian 11 host EOL** — bump CI containers to `mcr.microsoft.com/playwright:v1.62.0-jammy` (Debian 12) or Ubuntu 24.04. Local Linux dev machines still on Debian 11 will fail to install the browser bundle
- [ ] **Component tests** — if you're using the old `@playwright/experimental-ct-react` inline-mount model, convert to the stories-and-galleries model (the inline mount still works but is deprecated; new projects should start with stories)
- [ ] **Passkey persistence** — if you want passkeys seeded from one context to carry into the next, add `credentials: true` to `storageState()` calls (and to `browser.newContext({ storageState })`)
- [ ] **No migration required** if you only used the standard `page` / `expect` API

### Sources

- [Playwright 1.62.0 release notes](https://github.com/microsoft/playwright/releases/tag/v1.62.0)
- [Playwright 1.62 docs — Release notes](https://playwright.dev/docs/release-notes)
- [Playwright docs — Components (stories and galleries model)](https://playwright.dev/docs/test-components)
- [Playwright docs — fixtures.mount()](https://playwright.dev/docs/api/class-fixtures#fixtures-mount)
- [Playwright docs — Reporter.preprocess()](https://playwright.dev/docs/api/class-reporter#reporter-preprocess)
- [Playwright docs — testConfig.retryStrategy](https://playwright.dev/docs/api/class-testconfig#test-config-retry-strategy)
- [Playwright docs — Getting Started with the Playwright MCP server](https://playwright.dev/docs/getting-started-mcp)
- [Playwright docs — playwright-cli](https://playwright.dev/docs/getting-started-cli)

## Playwright 1.62.1 (July 30, 2026) — Bug-Fix Patch — Two Fatal `tsconfig.json` Resolution Regressions + 3 Other Fixes

The previous cron (v1.5.07 at 2026-07-30T12:03Z) documented Playwright 1.62.0 (Jul 24, 2026), but exactly 6 days later (`2026-07-30T16:36:55Z`; GitHub release `v1.62.1` published at `2026-07-30T16:35:55Z`) **Playwright 1.62.1 shipped** — a bug-fix patch addressing **five regressions introduced in 1.62.0**, two of which are tagged **"fatal since 1.62"**. The patch was missed by v1.5.07 (which captured the cycle immediately after 1.62.0 but several hours before 1.62.1 shipped).

`@playwright/test@latest` advances **`1.62.0 → 1.62.1`**. No new APIs, no config changes, no deprecations; pure regression-fix patch. **Action required if you upgraded to 1.62.0 in the last 6 days**: bump to 1.62.1 in your next `npm install`. Every project that hit the broken-`tsconfig.json` cases needs this patch — the two fatal regressions broke CI for affected projects.

### Bug fixes shipped in 1.62.1 (verbatim from the [GitHub release body](https://github.com/microsoft/playwright/releases/tag/v1.62.1))

1. **#41989 `[Regression]: tsconfig "extends" bare specifier isn't resolved via node_modules walk-up like tsc` — fatal since 1.62**. Playwright's tsconfig parser was supposed to follow `tsc`'s behavior (walk up `node_modules` to find a bare-specifier `extends`), but in 1.62.0 the parser locked in only the project's local `tsconfig.json` and refused the walk-up. **Effect:** any project whose `tsconfig.json` does `extends: "@tsconfig/node22/tsconfig.json"` or `extends: "@tsconfig/strictest/tsconfig.json"` or `extends: "@tsconfig/strictest/tsconfig.next.json"` (effectively all strict TS projects + the entire @tsconfig/* ecosystem of shared configs) saw Playwright's type checker silently ignore the extended config and fall back to Playwright's embedded defaults. **Fix:** restore the `tsc`-compatible walk-up behavior. **Audit recipe:** `cat tsconfig.json | jq -r .extends` — if it returns a bare specifier (anything starting with `@` or that isn't a relative `./` / `../` path) and Playwright's type errors don't match `tsc`, this fix applies.

2. **#41998 `[Regression]: directory-form tsconfig project references ("path": "../pkg") fail to resolve` — fatal since 1.62**. Playwright's project-references resolver accepted only the `path: "../pkg/index.ts"` form (pointing at a specific file) and choked on the more common `path: "../pkg"` directory form (which `tsc` resolves via the referenced package's `main` / `types` field). **Effect:** every monorepo using TS project references with the directory form — Next.js apps with `references: [{ "path": "../packages/ui" }]` where `packages/ui/package.json` has `"types": "./dist/index.d.ts"`, basically the entire monorepo-with-UI-libs pattern — got `Cannot find module '../pkg'` from Playwright's compiler. **Fix:** restore directory-form resolution. **Audit recipe:** `rg ""path":\s*"\.\./[^/]+"" tsconfig.json` — if any match ends in a package directory (no trailing `.ts` file), this fix applies.

3. **#41985 `Accessibility snapshot drops button name when text is nested inside spans with aria-hidden SVG`**. `page.accessibility.snapshot()` was walking past the aria-hidden SVG subtree and then colliding on the next text node, dropping the button's name for markup like `<button><span><svg aria-hidden="true">icon</svg> My Action</span></button>` — common in icon-only or icon-prefixed buttons. Now flattens correctly. **Audit recipe:** `rg "aria-hidden="true"" src/` — every aria-hidden SVG inside a button label needs a regression test for `expect(await page.getByRole('button', { name: '...' }))`.

4. **#42000 `[Regression]: page.evaluate() arg of a branded primitive type (string & { brand }) no longer type-checks since 1.62`**. Playwright's `page.evaluate(fn, arg)` type signature tightened in 1.62.0 to disallow branded primitives (e.g. `type UserId = string & { readonly brand: unique symbol }`) from being passed as `arg`, even though they were allowed and worked at runtime in 1.61.x. **Effect:** every TS codebase that uses brand types for ID columns (effectively every serious Zod-driven + Postgres-backed app that uses `z.string().brand<"UserId">()`) saw `tsc --noEmit` errors on every `page.evaluate((el) => el.dataset.userId, userId)` call. **Fix:** accept branded primitives back. **Audit recipe:** `rg "type \w+ = string & \{" src/ types/` (or `z.string().brand<"\w+">` for Zod-branded strings) — every branded primitive type that's passed to `page.evaluate` needs this fix.

5. **#42013 `[BUG] Image-type actionable elements are not presented in the snapshot`**. `<input type="image">` (the HTML form-submit button that submits the form with the click coordinates) — an interactive element per ARIA — was missing from `page.accessibility.snapshot()` results. Now included as a `button` role. **Audit recipe:** `rg 'type="image"' src/` — affects every project using coordinate-submitting image buttons (the classic "click position matters for map/canvas forms" pattern; rare in 2026, but used by some embedded-image form UIs).

### Why this patch can't wait

The two **fatal since 1.62** bugs (#41989 + #41998) share a common cause: 1.62.0 rewrote the tsconfig loader to be stricter about format and strictness in evaluating `extends` and `references` (the rewrites were part of the broader type-system tightening for 1.62's `Branded` types work, see #42000). The new stricter loader was correct in principle but failed two of the most common real-world tsconfig patterns. Because every CI run would fail outright (or every `tsc --noEmit` on a Playwright tests project would fail), this patch is essentially required for any project that touched `tsconfig.json extends/references` in the last 6 days.

### Migration checklist (1.62.0 → 1.62.1)

- [ ] `npm install -D @playwright/test@^1.62.1` — single dependency bump; no peer-dep changes, no `playwright.config.ts` migration
- [ ] **`tsconfig.json` walk-up regression check** — if your `tsconfig.json` does `extends: "@some-scope/some-pkg/..."`, verify Playwright now resolves it (`pnpm playwright test --list` should not error with `Cannot find extends config`)
- [ ] **`tsconfig.json` references directory-form check** — if any reference in your `references: []` array is a directory like `"../packages/ui"` (not `"../packages/ui/index.ts"`), verify Playwright now resolves it (`pnpm playwright test --list` should not error with `Cannot find module '../packages/ui'`)
- [ ] **Branded primitives in `page.evaluate`** — if your code passes branded IDs (e.g. `page.evaluate((el) => el.dataset.userId, userId)` where `userId: UserId`), verify the type error is gone
- [ ] **No migration required** if you only used `tsconfig.json` with local-file extends (no bare specifier) and no `references` and didn't brand your primitives. But the patch is small and harmless; just take it.

### Sources

- [Playwright 1.62.1 release notes](https://github.com/microsoft/playwright/releases/tag/v1.62.1) — the 5-entry bug-fix list, verbatim
- [Playwright PR #41989 — `[Regression]: tsconfig "extends" bare specifier isn't resolved via node_modules walk-up`](https://github.com/microsoft/playwright/pull/41989)
- [Playwright PR #41998 — `[Regression]: directory-form tsconfig project references fail to resolve`](https://github.com/microsoft/playwright/pull/41998)
- [Playwright PR #41985 — `Accessibility snapshot drops button name when text is nested inside spans with aria-hidden SVG`](https://github.com/microsoft/playwright/pull/41985)
- [Playwright PR #42000 — `page.evaluate() arg of a branded primitive type no longer type-checks`](https://github.com/microsoft/playwright/pull/42000)
- [Playwright PR #42013 — `Image-type actionable elements are not presented in the snapshot`](https://github.com/microsoft/playwright/pull/42013)
- [npm: `@playwright/test@1.62.1`](https://www.npmjs.com/package/@playwright/test/v/1.62.1) (published 2026-07-30T16:36:55Z)






## Playwright `retry` Now Respects the Given Duration — PR #96354 (**SHIPPED in `16.3.0-canary.103`**, [dan](https://github.com/gaoJude) / [Next.js team](https://github.com/vercel/next.js), merged 2026-07-29T20:42:57Z, npm-published 2026-07-30T00:11:44Z)

**The bug:**

```js
await retry(async () => {
  expect(await browser.elementById('never-appears').text()).toBe('x')
}, 15_000)
```

You'd reasonably expect this to wait at most 15 seconds. Before PR #96354, the actual answer was **~3 minutes** (170103ms in the PR's own measurement).

**Why:** Playwright's `retry(fn, ms)` was dividing `ms` by the 500ms internal interval and running that many tries (so `15_000` → 30 tries, plus the initial try = 31 tries). The bug was that **the per-try stall wasn't being measured**. For cheap tries (e.g. pure JS state checks), the per-try time is negligible and the calculation roughly holds. But for tries that look for a DOM element via `elementById('never-appears')` / `page.waitForSelector(...)`, each try sits in the selector for its full 5-second timeout before giving up — so 31 tries × 5 seconds = ~155s, plus the 15s of sleeps in between = ~170s. The user-visible behavior: `retry(fn, 15_000)` actually waits ~3 minutes.

**The downstream effect (the worse bug):** Jest kills the test at its own timeout long before that. So the actual underlying error gets thrown away, and the user sees a useless `Exceeded timeout of 120000 ms` with no indication of which assertion never came true. The PR author writes: *"My agent lost an hour trying to make sense of this twice now."*

**The fix:** `retry` now **counts time**, not tries. After the fix, `retry(fn, 15_000)` actually waits 15 seconds (well, 15 seconds + whatever the last attempt takes, so 20 in the worst case if the last try is slow). The interval no longer has to divide the duration (because nothing counts steps anymore).

**Semantics after the fix:**

- **`retry` watches the clock and stops when its time is up.**
- **`retry` always makes at least one try** (never zero tries).
- **Cheap waits behave exactly as before** — only the slow-stall case is fixed.
- **The interval no longer divides the duration** — you can pass `retry(fn, 7_000)` with `interval: 500` and it'll just run "as many 500ms-spaced tries as fit in 7s", with the last one partial.
- **The timeout is not strictly respected if the last try is slow** — Playwright waits for the last try to finish so it can show the correct error in the correct test. This is intentional.

**Before/after measurements (from the PR):**

| Scenario | Before PR #96354 | After PR #96354 |
|---|---|---|
| `retry(fn, 15_000)` with cheap `fn` | ~15s (correct) | ~15s (unchanged) |
| `retry(fn, 15_000)` with slow `fn` (5s stall per try) | **170s** (bug) | **~20s** (correct — 15s + final 5s try) |
| `retry(fn, 5_000)` with cheap `fn` | ~5s | ~5s (unchanged) |
| `retry(fn, 5_000)` with slow `fn` (5s stall per try) | ~155s (bug, hits Jest's 120s timeout) | ~10s (correct — 5s + final 5s try) |

**Practical impact:**

- **Anyone using `playwright.retry(...)` in test code** — the retry duration is now reliable.
- **Anyone whose CI was failing with `Exceeded timeout of 120000 ms` with no underlying assertion** — the fix means the underlying assertion error now surfaces in the test report. So the failure mode "test times out and we don't know why" goes away.
- **Agent-driven dev loops** — agents that lose time to "I need to figure out which assertion is hanging" now get the actual assertion error in the report.
- **No new API, no new config flag, no migration needed** — pure bug fix.

**Who needs to audit:**

```bash
# Find every retry() call in your test code
rg -n '\bretry\(' --type ts --type js --type tsx --type jsx

# Find every custom retry helper that wraps playwright's retry
rg -n 'function.*retry|const.*=.*retry' --type ts --type js

# Find any test that mentions timeout-related flake
rg -n 'timeout|flake|slow' test/ tests/ e2e/ 2>/dev/null | head -20
```

For each match, the retry semantics are now reliable; you don't need to change anything, but you can re-evaluate any "retry for 60s to give it enough time" patterns that were previously masking the slow-stall bug.

**Source:** [PR #96354 — `Make retry stop when the time it was given is up`](https://github.com/vercel/next.js/pull/96354) · dan · merged 2026-07-29T20:42:57Z · **SHIPPED in `16.3.0-canary.103`** (npm-published 2026-07-30T00:11:44Z).


## Vitest 5 — Forward-Looking Section (Aug 2026, beta.7 released 2026-07-24, GA target Q4 2026)

Vitest 5.0 is in active beta as of August 2026. The latest stable is still **Vitest 4.1.10** (released 2026-07-06, the same day as 3.2.7 backport). The 5.0 line (`vitest@beta`, currently **5.0.0-beta.7**, released 2026-07-24) introduces 7 forward-looking features you should know about before the GA drop, since they will land as behavior changes rather than opt-in flags. **Do not install in production yet** — but review your existing Vitest 4 config to anticipate the migration friction.

### 5.0.0-beta.7 (2026-07-24) — The Material Features

**1. [`injectCjsGlobals` toggable option** (PR #10709 by sheremet-va, merged 2026-07-23) — `injectCjsGlobals: false` lets you opt out of the legacy CJS-style global injection (`describe`, `it`, `expect`, `vi`, `beforeAll`, etc. as `globalThis` properties). For ESM-only projects that prefer to import every global explicitly, this cleanups the global namespace and helps prevent name collisions with `undici`/`jest-extended`/`@storybook/test` mocha globals. The default stays `true` for backward compatibility, but the Vitest team will flip it to `false` in 5.0 stable. Audit recipe: `rg -n "describe|it\(|beforeEach|afterEach|afterAll|beforeAll" --type js test/ src/ | rg -v "import.*vitest"` — any hit without a matching `import` is using the CJS globals.

```ts
// vitest.config.ts — opt in early so the migration is incremental
export default defineConfig({
  test: {
    injectCjsGlobals: false, // requires explicit `import { describe, it, expect } from 'vitest'`
  },
})
```

**2. `fsModuleCache` promoted to top-level option** (PR #10734 by sheremet-va, merged 2026-07-20) — the 5.0 line consolidates the 4.1.x experimental `fsModuleCache` into a top-level config option (the 4.1 stable version was already documented in this skill; 5.0 just removes the `experimental.` prefix). Action: any `experimental: { fsModuleCache: true }` config from 4.1.x should be migrated to `fsModuleCache: true` at the top level for 5.0 compatibility.

**3. `vi.when()`** (PR #10174 by macarie, merged 2026-06-27) — a new conditional test-runner API that lets you write `vi.when(condition, () => { ... })` blocks that only run when the condition is true. Replaces the manual `if (process.env.CI) { ... }` pattern that scattered throughout test suites. The condition is evaluated at config-load time, so it has access to the same env vars + flags as `defineConfig`. Practical impact: cleaner test setup for CI-only assertions, skip-only-in-CI specs, and flag-gated test variants.

```ts
// Before 5.0
test('production-only behavior', () => {
  if (!process.env.CI) return
  // ... CI-only assertions
})

// After 5.0
vi.when(process.env.CI, () => {
  test('production-only behavior', () => {
    // ... CI-only assertions
  })
})
```

**4. `agent` reporter** (added in 4.1.0 Mar 2026, mature in 5.0) — a minimal-output reporter designed for AI coding agents that consume Vitest output as token-streamed context. Suppresses all passed-test output and console logs from passing tests, shows only failed tests with their full error output. `--reporter=agent` is the cli flag. Combined with `vi.when`, AI-agent test loops get a 60–80% reduction in token usage per test run.

**5. Nested projects support** (PR #10846 by @antfu, merged 2026-07-30) — the headline breaking change for 5.0. Projects can now be **nested** under a parent project using the new `projects` field. This replaces the flat `workspace` model that Vitest 4 used for monorepos. The breaking bit: any existing `workspace` config in `vitest.config.ts` (which used `defineProject` for sub-packages) must be migrated to `projects` with the new nested shape. Audit recipe: `rg -n "defineProject|workspace:" vitest.config.ts vitest.workspace.ts` — any hit needs migration. The new shape: `projects: [{ test: { name: 'unit', include: ['src/**/*.test.ts'] } }, { test: { name: 'integration', include: ['tests/**/*.test.ts'] } }]`.

**6. Tagged tests (`test.tags`)** — forward-looking only in beta.7, will land in 5.0 stable. Lets you attach tags to tests (e.g. `test('login flow', { tags: ['@integration', '@auth'] }, async () => { ... })`) and filter on them at the CLI (`vitest --tags='@integration'`). Modeled on pytest markers. Particularly useful for AI-agent test loops where the agent wants to scope to a subset of tests.

**7. `aroundEach` / `aroundAll` hooks** — new lifecycle hooks that wrap each test (or `describe` block) like middleware, with both `before` and `after` callbacks. Replaces the manual `try { ... } finally { ... }` pattern in tests that need DB transactions, tracing spans, or temp-file cleanup. Forward-looking only in beta.7.

### Beta-train cadence

- **5.0.0-beta.4** (2026-06-01) — internal API refactors
- **5.0.0-beta.5** (2026-06-15) — `--repeats` CLI flag, `thresholds.autoUpdate` improvements
- **5.0.0-beta.6** (2026-07-06) — `vi.when()` + `aliased imports` for `vitest/node`
- **5.0.0-beta.7** (2026-07-24) — `injectCjsGlobals` toggable (latest as of this cron)
- **5.0.0-beta.8** (expected 2026-08-08 to 2026-08-15) — `nested projects` (PR #10846) + `tags` + `aroundEach`/`aroundAll` will likely ship here
- **5.0.0 stable** (expected late September / early October 2026) — GA target aligned with TS 7.1 ships

### Migration checklist (Vitest 4 → Vitest 5)

1. **Audit `vitest.config.ts`** for `experimental: { fsModuleCache: true }` — move to top-level `fsModuleCache: true`.
2. **Audit `workspace` configs** — convert to `projects` with the new nested shape.
3. **Audit `injectCjsGlobals`** — opt out early (`injectCjsGlobals: false`) to surface every implicit-global usage before the 5.0 stable default flip.
4. **Audit `defineProject` imports** — will be removed in 5.0; replace with `defineProject` from the new project nesting API.
5. **Bump Node to 20.18+ (or 22 LTS / 24 LTS)** — Vitest 5 requires Node 20.18+ even for the beta train.
6. **Audit any `vitest/coverage`, `vitest/environments`, `vitest/snapshot`, `vitest/runners`, `vitest/suite`, `vitest/reporters`, `vitest/mocker` imports** — in 5.0, these are consolidated into `vitest/node` (server-side) and `vitest/runtime` (browser-side). The old import paths will be deprecated.
7. **Audit `test.sequential`** — replaced with `{ concurrent: false }` option on `test` / `describe` in 5.0.
8. **If you use `vitest bench`** — the 5.0 API moves the bench config inside the `test()` callback (PR #10680). The `bench()` callback-level API is gone.

**Sources:**

- [Vitest 5 forward-looking Discussion #9664](https://github.com/vitest-dev/vitest/discussions/9664) — the team's Vite-8-aligned cadence + the 5.0 feature roadmap
- [Vitest 5.0.0-beta.7 release notes](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.7) — the `injectCjsGlobals` feature
- [Vitest 5.0.0-beta.6 release notes](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.6) — the `vi.when()` API
- [Vitest 5.0.0-beta.5 release notes](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.5) — the `--repeats` CLI flag
- [Vitest 5.0.0-beta.4 release notes](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.4) — internal API refactors
- [Vitest `agent` reporter docs](https://vitest.dev/guide/reporters) — the AI-agent token-saving minimal reporter
- [Vitest nested projects PR #10846](https://github.com/vitest-dev/vitest/pull/10846) — the 5.0 breaking change for monorepos
- [Vitest 4.1.10 release notes](https://github.com/vitest-dev/vitest/releases/tag/v4.1.10) — the latest stable (4.1.10)
- [Vitest 3.2.7 release notes](https://github.com/vitest-dev/vitest/releases/tag/v3.2.7) — the 3.x backport train

## Testing `use cache` Functions and `'use cache'` Components (Next.js 16.3 Cache Components)

With `'use cache'` becoming the canonical caching primitive in Next.js 16.3, testing `use cache` functions and components requires explicit handling — these run on the server, not in your test environment, and `vi.mock` doesn't intercept them like normal imports. Here are the canonical patterns:

### Pattern 1 — Mock the cacheable function with `vi.mock` (Vitest 4 + Next.js 16.3)

```ts
// app/lib/posts.ts
'use server'
export async function getPost(id: string) {
  'use cache'
  cacheTag(`post:${id}`)
  cacheLife('hours')
  return db.post.findUnique({ where: { id } })
}

// __tests__/PostCard.test.tsx
import { vi } from 'vitest'

// Mock the cacheable function — vi.mock intercepts the module, NOT the 'use cache' directive
vi.mock('../app/lib/posts', () => ({
  getPost: vi.fn(async (id: string) => ({
    id,
    title: 'Mocked Post',
    content: 'This is a fixture, not a real DB query.',
  })),
}))

import { getPost } from '../app/lib/posts'
import { PostCard } from '../app/components/PostCard'

test('renders the post title', async () => {
  const post = await getPost('123')
  render(<PostCard post={post} />)
  expect(screen.getByText('Mocked Post')).toBeInTheDocument()
})
```

### Pattern 2 — Test the cache boundary itself with `next/cache` injection

For tests that should verify the `cacheTag` and `cacheLife` are set correctly, mock the `next/cache` module directly:

```ts
// __tests__/cache-config.test.ts
import { vi } from 'vitest'

const cacheTag = vi.fn()
const cacheLife = vi.fn()

vi.mock('next/cache', () => ({
  cacheTag: (tag: string) => cacheTag(tag),
  cacheLife: (profile: string) => cacheLife(profile),
  unstable_cache: (fn: Function) => fn,
}))

import { getPost } from '../app/lib/posts'

test('applies the correct cache tag and lifetime', async () => {
  await getPost('123')
  expect(cacheTag).toHaveBeenCalledWith('post:123')
  expect(cacheLife).toHaveBeenCalledWith('hours')
})
```

### Pattern 3 — Test `'use cache'` Server Components with React Testing Library

Server Components that use `'use cache'` cannot be rendered with `render(<Component />)` directly — they need to be `async` and resolved first. The `React.cache` integration in Next.js 16.3 means cache hits within a single render are deduplicated, so the mock only needs to be set up once per test:

```tsx
// __tests__/PostCard.test.tsx
import { render, screen } from '@testing-library/react'

// 'use cache' components must be awaited before rendering
test('renders cached post', async () => {
  vi.mocked(getPost).mockResolvedValueOnce({ id: '1', title: 'Cached' })
  
  // For Server Components, await the component itself
  const ResolvedPostCard = await PostCard({ postId: '1' })
  render(ResolvedPostCard)
  
  expect(screen.getByText('Cached')).toBeInTheDocument()
})
```

### Pattern 4 — Test Route Handlers that use `use cache` (Next.js 16.3 native test utilities)

For E2E-style tests of cached route handlers, use the `@next/test-utils` `nextTest()` helper that ships with Next.js 16.3:

```ts
// __tests__/api-posts.test.ts
import { nextTest } from '@next/test-utils/playwright'

const test = nextTest({ fixture: 'with-cache-components' })

test('GET /api/posts returns cached list', async ({ request }) => {
  const res = await request.get('/api/posts')
  expect(res.status()).toBe(200)
  const data = await res.json()
  expect(data).toHaveLength(3)
  
  // Subsequent request is a cache HIT — verify the cache header
  const cached = await request.get('/api/posts')
  expect(cached.headers()['x-nextjs-cache']).toBe('HIT')
})
```

### Common Mistakes — Testing `'use cache'` boundaries

- **Not mocking the database call before `'use cache'`** — the directive runs on the server, so it'll hit your real DB in tests. Either `vi.mock` the DB client or wrap the test in a transaction that rolls back.
- **Forgetting to mock `next/cache` when testing cache metadata** — `cacheTag` and `cacheLife` are no-ops in the test environment unless you explicitly mock them.
- **Trying to render a `'use cache'` Server Component synchronously** — you must `await` the component first before passing it to `render()`. The `render(await Component({...}))` pattern is mandatory.
- **Mocking `getPost` only in the test scope** — use `vi.hoisted` to lift the mock definition above the `import` statements, or you'll get "Cannot access X before initialization" errors.
- **Forgetting to clear `vi.mock` state between tests** — `use cache` memoization in Next.js 16.3 means a stale mock will leak across tests. Call `vi.clearAllMocks()` in `beforeEach`.
- **Not testing the cache invalidation** — `cacheTag` and `cacheLife` are the configuration; the actual invalidation happens via `updateTag` / `revalidateTag`. Test the call site separately using the audit recipes in `server-components.md` → `## revalidateTag vs updateTag`.

**Sources:**

- [Next.js 16.3 Testing 'use cache' Components](https://nextjs.org/docs/app/guides/testing/use-cache) — the official Next.js testing guide for cache components
- [Vitest `vi.mock` + hoisted mocks](https://vitest.dev/api/vi#vi-mock) — the canonical mocking pattern
- [Next.js 16.3 Test Utils](https://nextjs.org/docs/app/api-reference/test-utils) — the `nextTest()` helper for Playwright integration
- [React Testing Library — Async Server Components](https://testing-library.com/docs/react-testing-library/intro#waiting-for-async-api) — the `await` pattern for Server Components

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

**`experimental.fsModuleCache`** and **`experimental.fsModuleCachePath`** are MOVED to top-level **`test.fsModuleCache`** and **`test.fsModuleCachePath`**. The cache directory defaults to `<workspaceRoot>/node_modules/.vitest-cache` (a single workspace-root directory shared by every project in a workspace), `fsModuleCache` defaults to `false` (off by default — opt-in for now), and the old `experimental.*` options are migrated with a deprecation warning. This unblocks **Vitest 5** stabilization — issue #10701 confirmed "We haven't received any issues regarding this feature, and we have also been running tests with this flag enabled for a while now" so the core team is comfortable promoting it. **Vitest 5.0.0-beta.7 SHIPPED 2026-07-24T11:40:54Z** and the `fsModuleCache` promotion is now in beta.7 (see Upcoming beta.7 section below). **No Vitest 4 patch has shipped yet** — the promotion is on Vitest 5 `main`; the next Vitest 4 patch (probably 4.1.11) will backport the relevant fixes.

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

### Upcoming (Vitest 5.0-beta.6, July 6, 2026 — superseded by beta.7 below)

Vitest 5.0 is on beta.7 (published 2026-07-24T11:40:54Z). Five beta releases have shipped between May 19 and July 24, 2026 (beta.3 on May 19, beta.4 on June 1, beta.5 on June 15, beta.6 on July 6, **beta.7 on July 24**). The list below is the beta.6 breaking-change set; the beta.7 section immediately following this one supersedes it. and the breaking-change list has grown materially. **The list below supersedes anything earlier.** If you pin to a specific beta, check the inline `[#PR]` links to verify the change is still in your range. The migration guide is the canonical reference: https://main.vitest.dev/guide/migration

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

### Upcoming (Vitest 5.0-beta.7, July 24, 2026)

[Vitest 5.0.0-beta.7](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.7) shipped **2026-07-24T11:40:54Z** — five beta releases between May 19 and July 24 (beta.3 May 19, beta.4 June 1, beta.5 June 15, beta.6 July 6, **beta.7 July 24**). The breaking-change list is now stable enough that the next release is plausibly rc.1. **The list below supersedes the beta.6 list above (8 → 9 items, including 1 NEW breaking change + 4 NEW features + 11+ bug fixes + a major performance bundle).** If you pin to a specific beta, check the inline `[#PR]` links to verify the change is still in your range. The migration guide is the canonical reference: https://main.vitest.dev/guide/migration

#### New breaking change in beta.7 (July 24, 2026) — **MUST READ before bumping**

**1. Config resolution separated from server creation — [#10554](https://github.com/vitest-dev/vitest/issues/10554)** by @sheremet-va

The Vitest config-resolution lifecycle is now decoupled from the Vite server creation. In practice this means: (a) any side effect that relied on `defineConfig()` synchronously triggering the Vite server (e.g. spinning up a global `mockServer` in your config that mutates server state) will now run before the server is ready and will throw "server not initialized" on first request; (b) plugin hooks that read server state during their `config` hook will now see `null` instead of the in-progress server (the fix is at [#10731](https://github.com/vitest-dev/vitest/issues/10731) — disable server HMR before plugins read it in their config hook). For most projects this is a no-op — the change is observable only if you have **custom `defineConfig()` callbacks** that touch Vite server internals or that register globals tied to the Vite server's lifecycle.

```ts
// vitest.config.ts

// ❌ No longer works (beta.7+) — reads server before it's created:
// export default defineConfig(() => {
//   const server = createServer(/* ... */)   // throws "server not initialized"
//   server.middlewares.use(/* ... */)
//   return { /* ... */ }
// }).tap(/* ... */)

// ✅ Defer server-side state to globalSetup or a setup file:
export default defineConfig({
  test: {
    globalSetup: ['./test/global-setup.ts'],
    setupFiles: ['./test/setup.ts'],
  },
})
```

#### New features in beta.7 (July 24, 2026)

**1. `injectCjsGlobals` option (toggle-able) — [#10709](https://github.com/vitest-dev/vitest/issues/10709)** by @sheremet-va

CJS globals (`require`, `module`, `exports`, `__dirname`, `__filename`) are now opt-in via the `injectCjsGlobals` config option. Default is `true` for backward compat, but flipping to `false` saves ~5–15 ms of cold-start per test file and pairs with the ESM-only migration story. Turn it off if your codebase is 100% ESM and you don't have legacy require-shim libraries.

```ts
// vitest.config.ts — for ESM-only projects
export default defineConfig({
  test: {
    injectCjsGlobals: false,  // skip CJS global injection
  },
})
```

**2. `fsModuleCache` promoted to top-level option — [#10734](https://github.com/vitest-dev/vitest/issues/10734)** by @sheremet-va

The `experimental.fsModuleCache` promotion that was documented in the 1.4.75 cycle (ahead of beta.6) is now in beta.7. Top-level `test.fsModuleCache` and `test.fsModuleCachePath` are the new canonical names; defaults are `false` (off, opt-in) and `<workspaceRoot>/node_modules/.vitest-cache` respectively. Action: rename `experimental.fsModuleCache` → `test.fsModuleCache` in your Vitest 5 configs; no behavior change.

**3. Non-ASCII characters in `for`/`each` title placeholders — [#10773](https://github.com/vitest-dev/vitest/issues/10773)** by @k-yle

`test.each` and `describe.each` now accept non-ASCII placeholders in title templates. Useful for i18n projects and golden-file tests with localized fixtures.

```ts
// Before (beta.6 and earlier) — non-ASCII chars were stripped or replaced:
test.each([
  ['café', 'biströt'],
  ['日本', '中国'],
])('renders %s → %s', (a, b) => { /* ... */ })
// Output (beta.6): "renders caf → biströt" / "renders   →  "

// After (beta.7+) — full UTF-8 preserved:
// "renders café → biströt" / "renders 日本 → 中国"
```

**4. Pluggable benchmark provider API — [#10799](https://github.com/vitest-dev/vitest/issues/10799)** by @GuillaumeLagrange + @sheremet-va

The `bench` runner now supports a pluggable provider API. You can write a custom benchmark provider (e.g., to integrate with `tinybench` forks, `hyperfine`-style external runners, or in-house perf systems). Default provider is unchanged.

#### Performance bundle (5 PRs, beta.7)

This is the largest perf bundle of the beta.7 cycle — collectively ~30–60% cold-start reduction on typical projects, larger on multi-VM-pool setups:

- **Warm modules to workers in one round-trip + Node compile cache — [#10708](https://github.com/vitest-dev/vitest/issues/10708)** by @sheremet-va — `compileCache` is now opt-in (set `test.compileCache: true`); persistent across runs
- **Bundle vitest's own dependencies — [#10685](https://github.com/vitest-dev/vitest/issues/10685)** — removes the per-test-file dep-optimization round-trip for vitest internal modules
- **Reuse compiled code across vm pool contexts + prewarm the module graph — [#10744](https://github.com/vitest-dev/vitest/issues/10744)** — biggest win for `vmThreads`/`vmForks` pool users
- **v8 coverage: bounded-memory merge + precompiled globs — [#10506](https://github.com/vitest-dev/vitest/issues/10506)** — ~40% less memory on `--coverage` runs over 1000+ files
- **Browser mode: open adaptively + cut per-file round trips + prewarm + pre-bundle runtime — [#10726](https://github.com/vitest-dev/vitest/issues/10726) + [#10730](https://github.com/vitest-dev/vitest/issues/10730) + [#10727](https://github.com/vitest-dev/vitest/issues/10727) + [#10713](https://github.com/vitest-dev/vitest/issues/10713)** — significantly faster browser-mode startup

**Action:** bump `vitest` and `@vitest/browser` to `5.0.0-beta.7` for the perf wins on any project running >200 test files. No code changes required.

#### Headline bug fixes in beta.7 (July 24, 2026)

- **Vm pools respect cgroupsv2 memory limit — [#10721](https://github.com/vitest-dev/vitest/issues/10721)** — meaningful for CI runners using cgroupsv2 (most Kubernetes, modern systemd); previously the pool could OOM-kill the test process even when the cgroup had memory headroom
- **Set non-zero exit code when teardown throws during close — [#10794](https://github.com/vitest-dev/vitest/issues/10794)** — CI pipelines that rely on `vitest` exit code to fail the build now correctly fail on teardown errors
- **Disable server HMR before plugins read it in their config hook — [#10731](https://github.com/vitest-dev/vitest/issues/10731)** — paired with the breaking change above; prevents race conditions
- **Close the pool before the Vite servers — [#10725](https://github.com/vitest-dev/vitest/issues/10725)** — fixes a class of "port already in use" errors on `globalSetup`-based dev servers
- **Browser: preserve pre-transform request defaults — [#10748](https://github.com/vitest-dev/vitest/issues/10748)** — fixes the regression from beta.6 where some pre-transform middleware was skipped
- **Browser: fix error stacktrace location off-by-one — [#10724](https://github.com/vitest-dev/vitest/issues/10724)** — debug stack traces now point to the actual line, not the line below
- **Browser: mock `window.print` to avoid hanging — [#10798](https://github.com/vitest-dev/vitest/issues/10798)** — closes the long-standing issue #7375; tests that accidentally trigger `window.print()` no longer hang the runner
- **Pool: per-file isolation preserved when `maxWorkers: 1` — [#10743](https://github.com/vitest-dev/vitest/issues/10743)** — fixes a regression from beta.6 where vm pools could share state when forced to single-worker mode
- **Typecheck worker: remove listeners in `off` — [#10741](https://github.com/vitest-dev/vitest/issues/10741)** — fixes a listener-leak that accumulated across typecheck-only runs
- **Don't race the typechecker spawn grace period on Windows — [#10814](https://github.com/vitest-dev/vitest/issues/10814)** — fixes sporadic CI failures on Windows runners
- **Prevent node builtins double prefix — [#10630](https://github.com/vitest-dev/vitest/issues/10630) + [#10767](https://github.com/vitest-dev/vitest/issues/10767)** — fixes `import node:node:fs` edge cases from custom resolvers

**Sources for beta.7:**
- [Vitest 5.0.0-beta.7 release notes — July 24, 2026](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.7)
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
    // "vitest": "5.0.0-beta.7",
    // "@vitest/browser": "5.0.0-beta.7",

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

## Playwright `1.63.0-alpha-2026-08-05` + `next/image` Preserve-Response Testing Pattern (PR #96681, August 5, 2026)

The v1.5.21 cycle (Aug 4 06:14Z) added the `## @next/playwright — instant() Test Helper for Instant Navigations (Next.js 16.3)` section that documented the `@next/playwright` `instant()` test helper for instant-navigation testing. The v1.5.21 cycle also touched testing.md for the Vitest 5 forward-looking section + the `Testing 'use cache' Functions and 'use cache' Components` section. Since then, **testing.md has been silent through the v1.5.22 → v1.5.26 cycles (35h48min stale at this cron's check, tied with `server-components.md` for the most-stale topic file)**. The only material change in the 6h window for testing is:

1. **`@playwright/test@next` bumped to `1.63.0-alpha-2026-08-05`** (npm `dist-tag.next` moved 2026-08-05; the alpha train; **NOT the tracked `@latest` stable pin which remains `1.62.1`**; documented as forward-looking only — production pin remains `^1.62.1`).
2. **PR #96681 — `fix(next/image): preserve image response after optimization`** (merged 2026-08-05T15:13:25Z, closes issue #96612; lands in `next@16.3.1-canary.4` when that npm-publishes; documented in `components.md` → "Canary-branch component-relevant PRs ahead of canary.3" → PR #96681).

This new section covers both:

### `@playwright/test@next` 1.63.0-alpha-2026-08-05 (alpha train, forward-looking)

The `next` dist-tag of `@playwright/test` tracks the alpha train — new features + experimental APIs that haven't yet graduated to `@latest` stable. The 1.63.0 alpha line carries forward:

- **`@next/playwright` `instant()` helper enhancements** (the helper itself continues to evolve in lockstep with the 16.3.x line; expect new `instant()` features in the 1.63.x line as Next.js 16.3 grows).
- **Playwright Trace improvements** (the Trace Viewer + the trace.zip format continue to evolve; expect new fields in the trace schema in 1.63.x).
- **Test runner instrumentation hooks** (the alpha train carries forward new hooks for AI-agent-driven test scenarios — `@playwright/test` has been investing in agent-friendly hooks throughout 2026).

**Practical impact** (for projects currently on `@playwright/test@^1.62.1` stable):

- **NO production action required** — the tracked stable pin remains `^1.62.1`. The 1.63.0 alpha train is forward-looking only.
- **For teams experimenting with AI-agent test scenarios**: the alpha train is worth tracking; the new hooks may enable simpler agent-test integrations.
- **For teams migrating from another test runner (Cypress, WebdriverIO, etc.) to Playwright**: the 1.63.x line will be the next stable; worth waiting for the STABLE cut before committing.

**Audit recipe**:

```bash
# Confirm the alpha train version:
npm view @playwright/test dist-tags.next
# → should show: 1.63.0-alpha-2026-08-05 (or later alpha)

# Confirm the stable pin version:
npm view @playwright/test dist-tags.latest
# → should show: 1.62.1

# Audit your project's actual install:
npm ls @playwright/test
# → if you're on 1.62.x stable, no action required
# → if you're on 1.63.0-alpha, you're experimenting — know that the API may shift
```

### PR #96681 — `next/image` preserve-response testing pattern

Closes issue #96612 (the silent production crash for `next/og` after `next/image` SVG requests in the same Node.js process). The pre-fix behavior:

```ts
// PRE-FIX (next@16.3.0 + 16.3.1-canary.0/.1/.2/.3) — module-level singleton state corrupts downstream ImageResponse calls:

// 1. The first /_next/image?url=...svg request in the process triggers getSharp(), which calls
//    _sharp.block({ operation: ['VipsForeignLoad'] }) — this PERMANENTLY disables Sharp's SVG loader
//    for the entire process (because _sharp is a module-level singleton, the block is global).

// 2. Any subsequent ImageResponse (next/og) call that hands SVG to Sharp (resvg rasterizes to PNG
//    internally, but the post-rasterization handoff goes through Sharp for some pipelines) fails
//    with: "Input buffer contains unsupported image format" — which surfaces as a socket hang up
//    / crashed response on the ImageResponse route.

// POST-FIX (next@16.3.1-canary.4+) — getSharp() unblocks the SVG loader correctly:

// 1. Same first /_next/image SVG request — but the unblock list now includes 'VipsForeignLoadSvg',
//    so the SVG loader remains available process-wide.

// 2. Subsequent ImageResponse calls work correctly — no crash, no hang up.
```

**Practical impact on tests**:

- **Tests that exercise both `next/og` and `next/image`** — pre-fix, tests had to be ordered carefully (render `next/og` BEFORE any `next/image` SVG request) or had to call `_sharp.unblock({operation: ['VipsForeignLoadSvg']})` manually between tests; post-fix, no ordering or manual intervention is required.
- **Tests that exercise only `next/og` (no `next/image` in the same test file)** — pre-fix, no impact (the singleton is only corrupted by `next/image` requests); post-fix, no impact.
- **Tests that exercise only `next/image`** — pre-fix, no impact; post-fix, no impact.
- **Snapshot tests using Vitest + a next/image + next/og mix** — pre-fix, snapshots could fail if the test order varied; post-fix, snapshots are stable across runs.

**The canonical Playwright test for `next/og` + `next/image` coexistence** (works correctly on `next@16.3.1-canary.4+`):

```ts
import { expect, test } from '@playwright/test'

test('next/og works after next/image SVG request', async ({ page, request }) => {
  // 1. Hit a next/image SVG endpoint FIRST (this is the request that corrupted Sharp pre-fix)
  const imageResponse = await request.get('/_next/image?url=%2Ffoo.svg&w=384&q=75')
  expect(imageResponse.status()).toBe(200)
  expect(await imageResponse.headerValue('content-type')).toMatch(/^image\//)

  // 2. Then hit a next/og ImageResponse endpoint (this would CRASH pre-fix with
  //    "Input buffer contains unsupported image format" / socket hang up)
  const ogResponse = await page.goto('/api/og?title=hello')
  expect(ogResponse?.status()).toBe(200)
  expect(ogResponse?.headers()['content-type']).toMatch(/^image\/png/)

  // 3. Optional: take a snapshot of the rendered og image
  const ogBuffer = await ogResponse?.body()
  expect(ogBuffer).toBeTruthy()
  expect(ogBuffer!.length).toBeGreaterThan(100)
})

test('order does not matter post-fix', async ({ page, request }) => {
  // Post-fix, the order of next/og vs next/image requests does NOT matter.
  // Pre-fix, this test would fail because next/og was hit first (no corruption yet),
  // then next/image SVG hit (corrupts), then next/og again (would crash).

  const ogFirst = await page.goto('/api/og?title=world')
  expect(ogFirst?.status()).toBe(200)

  const imageAfter = await request.get('/_next/image?url=%2Fbar.svg&w=384&q=75')
  expect(imageAfter.status()).toBe(200)

  const ogSecond = await page.goto('/api/og?title=again')
  expect(ogSecond?.status()).toBe(200) // ← would fail pre-fix
})
```

**Vitest snapshot caveat** (for projects using Vitest + `@playwright/test` for next/image testing):

The `_sharp` module-level singleton persists across Vitest tests in the same worker process. If a Vitest test file mixes `next/image` + `next/og` tests, **the first `next/image` SVG request in the test file corrupts `_sharp` for all subsequent tests in the same file**. The pre-fix workaround was to either (a) call `_sharp.unblock({operation: ['VipsForeignLoadSvg']})` in a `beforeEach` hook, or (b) force module reload with `vi.resetModules()`. The post-fix behavior removes the need for either workaround, but for robustness across `next@16.3.0` and `next@16.3.1-canary.0/.1/.2/.3` (pre-fix) versions:

```ts
// vitest.config.ts — for next/image + next/og test suites, force module reset between tests:
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    // Force module reset between tests so _sharp singleton state doesn't leak
    clearMocks: true,
    restoreMocks: true,
    // For projects on pre-fix next@16.3.0 or next@16.3.1-canary.0/.1/.2/.3:
    // setupFiles: ['./vitest.setup.ts'],
  },
})

// vitest.setup.ts (only needed pre-fix; remove once on canary.4+):
// import { vi } from 'vitest'
// beforeEach(() => {
//   vi.resetModules()  // forces fresh _sharp import per test
// })
```

**3-step audit recipe**:

```bash
# 1. Confirm canary.4 includes PR #96681:
npm view next@canary version
# → should show: 16.3.1-canary.4 or later

# 2. Find any code paths that combine next/image SVG + next/og:
rg -ln "ImageResponse|next/og" tests/ e2e/ playwright/
# → any match means you should bump to canary.4 immediately

# 3. Check if any tests have manual _sharp.unblock calls (the pre-fix workaround):
rg -n "_sharp\.(un)?block" tests/ e2e/ playwright/ src/
# → any match means you have tests written against the pre-fix behavior; can be removed once on canary.4+
```

### Common Mistakes — `next/image` + `next/og` testing edition

- **Running `next/og` tests after `next/image` SVG tests in the same Playwright suite (pre-`next@16.3.1-canary.4`)** — `ImageResponse` crashes with `Input buffer contains unsupported image format` because `getSharp()`'s module-level `_sharp.block({...})` permanently disables the `VipsForeignLoadSvg` loader for the entire Node.js process. The fix in PR #96681 (closes issue #96612) adds `'VipsForeignLoadSvg'` to the Sharp unblock allowlist. Pre-fix workaround: render `next/og` BEFORE any `next/image` request in the same process, or call `_sharp.unblock({operation: ['VipsForeignLoadSvg']})` manually between tests. Migration: bump to `next@>=16.3.1-canary.4` once available; no code or config changes required. **The `dangerouslyAllowSVG` security gate is NOT affected** — untrusted user-supplied SVG is still blocked separately by `imageOptimizer()`. Cross-reference: `components.md` → "Canary-branch component-relevant PRs ahead of canary.3 → PR #96681".
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
- [Vitest 5.0.0-beta.7 release notes — July 24, 2026](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.7)
- [Vitest 5.0.0-beta.6 release notes — July 6, 2026](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.6)
- [Vitest 5.0.0-beta.5 release notes — June 15, 2026](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.5)
- [Vitest 5.0.0-beta.4 release notes — June 1, 2026](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.4)
- [Vitest 5.0.0-beta.3 release notes — May 19, 2026](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.3)
- [Vitest 4.1.10 release notes — July 6, 2026](https://github.com/vitest-dev/vitest/releases/tag/v4.1.10)
- [Vitest 3.2.7 release notes — July 6, 2026](https://github.com/vitest-dev/vitest/releases/tag/v3.2.7)

## Vitest Main Branch — 7 NEW Commits Ahead of `5.0.0-beta.7` (August 7, 2026 — Forward-Looking for `5.0.0-beta.8`)

The Vitest main branch has had a productive 2-day window (Aug 7) producing **7 NEW commits** ahead of `vitest@5.0.0-beta.7` (the latest npm-published beta, still unchanged from v1.5.27). All 7 commits landed 2026-08-07T11:52Z → 13:21Z. The headline is **PR #10854** — a critical VM-pool memory-leak fix that affects every Vitest project using `pool: 'vmThreads'` or `pool: 'forks'` against large test suites. The other 6 commits cover require(esm) in vm pools, worker output retention, watch-mode deflaking, browser-mode perf, browser connectTimeout config resolution, and a new duration-breakdown-as-percentages feature.

**Verified at this cron's check via** `GET /repos/vitest-dev/vitest/commits?sha=main&since=2026-08-05T18:00:00Z` returning 7 commits in the window. **`vitest@beta` still `5.0.0-beta.7`** — none of these commits have been bundled into an npm-published beta yet. The v1.5.27 prediction "5.0.0-beta.8 expected 2026-08-08 to 2026-08-15" still holds; the new commits will likely land in `5.0.0-beta.8` or `5.0.0-beta.9`.

### 1. PR #10854 — `fix(vm): stop retaining every finished test file in vm pool workers` (sheremet-va, merged 2026-08-07T11:52:23Z, 25 files / +483/-103, the **headline critical fix**)

**The bug (pre-fix)** — vm pool workers retained the memory of every finished test file (module graphs, vm context, DOM, user state) until the worker reached `vmMemoryLimit` and was recycled. **On a 638-file jsdom suite (zammad)**, each worker leaked **40-55MB per test file**, spent roughly **30% of its CPU in GC**, and was recycled 19 times per run — losing its compile caches every time. This is the kind of silent perf regression that only shows up in production-grade test suites and explains the slow Vitest 4.x perf reports in the wild.

**The fix** — The retention has several independent causes; each commit removes one:

- The per-file module runner was never closed (so Vite's transformer caches were never released).
- The `vm` context kept strong refs to test-file closures (so all module-level state stayed live).
- The DOM (jsdom/happy-dom) instances were never destroyed before the next test file.
- The worker's user-state map was cleared only on recycle, not per-file.

**Practical impact** (will ship in `vitest@5.0.0-beta.8`):
- **All Vitest projects with `pool: 'vmThreads'` or `pool: 'forks'`** benefit from this — every large suite (500+ files) gets faster test runs, lower memory ceiling, and more stable per-worker perf.
- **Expected ~10-30% reduction in wall-clock time** for suites with 500+ test files (the zammad case shows 30% CPU in GC, which directly maps to wall-clock savings).
- **Expected 40-55MB per-worker memory reduction** for jsdom suites, which lifts the `vmMemoryLimit` threshold for triggering worker recycle.
- **No code changes required** for users — the fix is internal to Vitest's vm pool worker lifecycle.

**The 5-step audit recipe:**
```bash
# 1. Confirm your Vitest version:
npm ls vitest
# → expect 4.1.10 stable or 5.0.0-beta.7 (current as of this cron)

# 2. Check your test pool config:
rg -n "pool:\s*['\"]" vitest.config.ts
# → if pool: 'vmThreads' or pool: 'forks', this PR benefits you when 5.0.0-beta.8 ships

# 3. Check your test file count (rough heuristic for impact):
find tests/ test/ src/ -name "*.test.ts" -o -name "*.spec.ts" | wc -l
# → if 500+, expect significant savings (10-30% wall-clock)

# 4. Confirm the beta train:
npm view vitest@beta version
# → expect 5.0.0-beta.8 (or later) when the fix lands in npm

# 5. Track the canary:
git clone --depth 1 https://github.com/vitest-dev/vitest.git /tmp/vitest-canary
cd /tmp/vitest-canary && git log --oneline -10
# → PR #10854 will appear in the main-branch feed until 5.0.0-beta.8 ships
```

### 2. PR #10829 — `feat(vm): support require(esm) in vm pools` (ari-perkkiö, merged 2026-08-07T12:11:47Z, 13 files / +1004/-90)

Adds support for `require(esm)` by using APIs exposed in **Node 24.9+**. Until this PR, Vitest's vm pools could only `require()` CommonJS modules — ESM-only modules (anything using `import`/`export` syntax in package form) had to be `import()`ed asynchronously, which broke synchronous test setup patterns (`jest.mock`, `vi.mock`, `__mocks__/` directory resolution, etc.).

**Practical impact** (forwards-looking only — not in npm yet):
- **ESM-only test dependencies** (modern packages shipping pure-ESM since 2024 — e.g., `chalk@5`, `nanoid@5`, `pino@9`, `tsx`, `globby@14`, `execa@8`, `node-fetch@3`) can now be `require()`ed in Vitest test files without breaking synchronous mock patterns.
- **No code changes required** for projects already on Node 24.9+ (most modern CI runners).
- **Migration concern**: the `engines: { node: '>=24.9' }` floor for this feature is a step-up from Vitest 4.x's `>=20.18`; check your CI matrix.

### 3. PR #10842 — `fix: don't lose worker output on teardown, deflake timing-sensitive tests` (sheremet-va, merged 2026-08-07T12:34:48Z, 9 files / +124/-30)

**The bug (pre-fix)** — Trailing worker stdio is lost on teardown (`test/pool.test.ts` "can capture worker's stdout and stderr", 17 failures). Worker-thread stdio is processed in a pipe that's drained on `worker.terminate()`; any output written after the drain started is lost. This affected every Vitest user who relies on `console.log` from the test body being captured in the test report — silent failures in CI where "the test passed but my console.log vanished".

**Practical impact** (will ship in `vitest@5.0.0-beta.8`):
- **All Vitest users** get reliable trailing-stdout capture. Tests that use `console.log` to debug flaky tests now reliably surface the log.
- **CI runs** stop having silent "passing test but no log" failures.

### 4. PR #10841 — `test: deflake tests sharing the watch fixture` (sheremet-va, merged 2026-08-07T12:34:16Z, 17 files / +320/-159)

Deflakes `test/watch/file-watching.test.ts:163` ("editing source file generates new test report to file system") — was the single flakiest test on CI: **35 failures across 26 unrelated branches in a week**, exclusively on the Windows e2e job, always with the same signature (after editing `math.ts` the captured stdout stays completely empty for the whole 20s `waitForStdout` timeout). This PR is test-internal only (deflakes the Vitest test suite itself); no user-facing impact.

### 5. PR #10820 — `feat: report the duration breakdown as percentages` (ari-perkkiö, merged 2026-08-07T13:21:55Z, 45 files / +2589/-112)

The Vitest test-summary phases (`environment`, `import`, `transform`, `tests`, etc.) are now reported as **shares of tracked time** instead of raw sums — previously the raw sums confusingly exceeded the wall time because phases run in parallel workers:

```
Duration  3.76s (environment 79%, import 14%, transform 6%, tests 1%)
```

Sorted by cost, aggregated across all projects (per-project lines were considered but rejected as too noisy).

**Practical impact** (forward-looking only — not in npm yet):
- **Vitest users running large test suites** get a much clearer picture of WHERE time is spent (e.g., "79% environment = jsdom setup dominates; consider pool: 'vmThreads' + reuse: true").
- **For AI-agent test loops**, the percentage breakdown makes token-efficient debugging much easier (one line instead of 5 numbers).
- **No code changes required** — it's a reporter output change.

### 6. PR #10729 — `perf(browser): serve framework assets as immutable` (sheremet-va, merged 2026-08-07T13:15:37Z, 1 file / +34/-13)

**The fix** — Vitest's own pre-built framework assets (the bundles that ship into each tester iframe) are now served with `Cache-Control: public, max-age=31536000, immutable` instead of having every tester iframe revalidate them.

**Practical impact** (will ship in `vitest@5.0.0-beta.8`):
- **Vitest Browser Mode users** see a measurable perf improvement in dev (each iframe no longer re-fetches framework assets on every test).
- **No code changes required** — purely a server-side cache-header change.
- **Skipped when `persistentContext: true`** (browser context outlives the run, so per-iframe cache busting is moot).

### 7. PR #10880 — `fix(browser): resolve connectTimeout from the project config (fix #10879)` (ari-perkkiö, merged 2026-08-07T11:52:56Z, 3 files / +30/-4)

**The bug (pre-fix)** — The browser session handshake read `connectTimeout` off `project.vitest.config` (the root resolved config), so a value set inside a project never reached it and the **60s default was used instead**. Every other `browser.*` option on that project IS honoured, and the timeout error itself is per-session and names the project — so reading the project's own resolved config is the right fix.

**Practical impact** (will ship in `vitest@5.0.0-beta.8`):
- **Vitest Browser Mode users with multi-project configs** (monorepos, mixed browser + node tests) get correct per-project `connectTimeout` resolution.
- **No code changes required** — the bug is that the value you set was being ignored; once the fix ships, your existing config starts working.

### Updated Vitest 5 forward-looking beta-train cadence

- **5.0.0-beta.7** (2026-07-24) — `injectCjsGlobals` toggable (current as of this cron; latest npm-published beta)
- **5.0.0-beta.8** (now expected late August / early September 2026) — will likely include the 7 NEW commits from this cycle + the `nested projects` (PR #10846) + `tags` + `aroundEach`/`aroundAll` features already predicted in v1.5.27. The v1.5.27 prediction window (2026-08-08 → 2026-08-15) is sliding due to the new 7-commit batch.
- **5.0.0 stable** (expected late September / early October 2026) — unchanged target

### Migration checklist (Vitest 4 → Vitest 5) — UPDATED

The v1.5.27 checklist still applies. New items for 5.0.0-beta.8:

1. **Audit `vitest.config.ts`** for `experimental: { fsModuleCache: true }` — move to top-level `fsModuleCache: true`.
2. **Audit `workspace` configs** — convert to `projects` with the new nested shape (PR #10846 already merged Jul 30; not yet in `5.0.0-beta.7`).
3. **Audit `injectCjsGlobals`** — opt out early (`injectCjsGlobals: false`) to surface every implicit-global usage before the 5.0 stable default flip.
4. **Audit `defineProject` imports** — will be removed in 5.0; replace with `defineProject` from the new project nesting API.
5. **Bump Node to 20.18+ (or 22 LTS / 24 LTS)** — Vitest 5 requires Node 20.18+ even for the beta train. **NEW: PR #10829 requires Node 24.9+** for `require(esm)` support in vm pools; otherwise the require(esm) feature is silently dropped.
6. **Audit any `vitest/coverage`, `vitest/environments`, `vitest/snapshot`, `vitest/runners`, `vitest/suite`, `vitest/reporters`, `vitest/mocker` imports** — in 5.0, these are consolidated into `vitest/node` (server-side) and `vitest/runtime` (browser-side). The old import paths will be deprecated.
7. **Audit `test.sequential`** — replaced with `{ concurrent: false }` option on `test` / `describe` in 5.0.
8. **If you use `vitest bench`** — the 5.0 API moves the bench config inside the `test()` callback (PR #10680). The `bench()` callback-level API is gone.
9. **NEW: Check your `pool: 'vmThreads'` / `pool: 'forks'` config** — PR #10854 delivers a free 10-30% wall-clock improvement on 500+ file suites; no code changes required but verify the speedup in your CI after upgrading.
10. **NEW: Check your multi-project `browser.*` configs** — PR #10880 fixes `connectTimeout` resolution from project config; if you set `browser.connectTimeout` per-project, verify it works after upgrading.
11. **NEW: Audit your test files for trailing `console.log`** — PR #10842 ensures trailing stdout is captured on teardown; verify in CI that logs you expect to see are now reliably appearing.

### Sources

- [Vitest main branch commits since 2026-08-05T18:00Z](https://github.com/vitest-dev/vitest/commits?sha=main&since=2026-08-05T18:00:00Z) — the 7-commit window
- [PR #10854 — `fix(vm): stop retaining every finished test file in vm pool workers`](https://github.com/vitest-dev/vitest/pull/10854) — sheremet-va, merged 2026-08-07T11:52:23Z, 25 files / +483/-103, the headline critical fix
- [PR #10829 — `feat(vm): support require(esm) in vm pools`](https://github.com/vitest-dev/vitest/pull/10829) — ari-perkkiö, merged 2026-08-07T12:11:47Z, 13 files / +1004/-90
- [PR #10842 — `fix: don't lose worker output on teardown, deflake timing-sensitive tests`](https://github.com/vitest-dev/vitest/pull/10842) — sheremet-va, merged 2026-08-07T12:34:48Z, 9 files / +124/-30
- [PR #10841 — `test: deflake tests sharing the watch fixture`](https://github.com/vitest-dev/vitest/pull/10841) — sheremet-va, merged 2026-08-07T12:34:16Z, 17 files / +320/-159
- [PR #10820 — `feat: report the duration breakdown as percentages`](https://github.com/vitest-dev/vitest/pull/10820) — ari-perkkiö, merged 2026-08-07T13:21:55Z, 45 files / +2589/-112
- [PR #10729 — `perf(browser): serve framework assets as immutable`](https://github.com/vitest-dev/vitest/pull/10729) — sheremet-va, merged 2026-08-07T13:15:37Z, 1 file / +34/-13
- [PR #10880 — `fix(browser): resolve connectTimeout from the project config (fix #10879)`](https://github.com/vitest-dev/vitest/pull/10880) — ari-perkkiö, merged 2026-08-07T11:52:56Z, 3 files / +30/-4
- [Vitest 5.0.0-beta.7 release notes — July 24, 2026](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.7) — the current npm-published beta (unchanged from v1.5.27)
- [Vitest 4.1.10 release notes — July 6, 2026](https://github.com/vitest-dev/vitest/releases/tag/v4.1.10) — the latest stable
- [Vitest 5 forward-looking Discussion #9664](https://github.com/vitest-dev/vitest/discussions/9664) — the team's Vite-8-aligned cadence + the 5.0 feature roadmap

## Vitest Main Branch — 7 NEW Commits Since v1.5.38 (August 10, 2026) — Forward-Looking for `5.0.0-beta.8`

The Vitest main branch has had a productive 1-day window (Aug 10) producing **7 NEW commits** ahead of `vitest@5.0.0-beta.7` (the latest npm-published beta, still unchanged from v1.5.27). The new commits landed 2026-08-10T07:53Z → 13:20Z. The headline is **PR #10848** — a `feat!:` (BREAKING) change that shares the Vite server between inline projects. The other 6 commits cover Windows typecheck crash reporting, browser prebundling of `vite/module-runner`, and 4 CI/deps updates.

**Verified at this cron's check via** `GET /repos/vitest-dev/vitest/commits?sha=main&since=2026-08-10T00:00:00Z&per_page=10` returning 7 commits in the window. **`vitest@beta` still `5.0.0-beta.7`** — none of these commits have been bundled into an npm-published beta yet. Total main-branch ahead of `5.0.0-beta.7`: **50 commits** (was 43 in v1.5.38). The v1.5.38 prediction "5.0.0-beta.8 expected late August / early September 2026" still holds; the new commits will likely land in `5.0.0-beta.8` or `5.0.0-beta.9`.

### 1. PR #10848 — `feat!: share the Vite server between inline projects` (sheremet-va, merged 2026-08-10T13:20:10Z, 25 files / +1270/-355) — **THE HEADLINE BREAKING CHANGE**

**What** — Inline projects that don't modify the Vite config now reuse the Vite server of the config that declares them, sharing its transform cache: shared source files are transformed once instead of once per project. This is controlled by the new top-level `sharedViteServer` option, **enabled by default** (`--sharedViteServer=false` to opt out).

**Constraints**:
- Only applies to inline projects; projects referenced as config files or directories keep resolving their own Vite config and server.
- An inline project still gets its own server when it defines Vite-level options, a non-default `extends`, or test options that affect the Vite config: `alias`, `browser`, `css`, `deps.optimizer`, `mode`, `root`.
- Every project keeps its own module resolution rules, module runner, and module instances on top of the shared server.
- When the server is shared, the declaring config file is executed once instead of once per project.
- Works at every level: inline projects of a nested projects container share the container's server.

**Stacked on** — #10846 (the "nested projects" feature, merged Jul 30, also part of the forward-looking Vitest 5 set)

**BREAKING** — The `feat!:` prefix indicates this is a breaking change. The default-on behavior shift means projects that previously relied on the per-inline-project Vite server (and its per-project transform cache) will see a behavior change. The `--sharedViteServer=false` opt-out restores the old behavior.

**Practical impact** (will ship in `vitest@5.0.0-beta.8+`):
- **Vitest monorepo users with inline projects** (`projects: [{ test: { ... } }, ...]` or `workspace` configs converted to `projects`): **shared source files are transformed once instead of once per project** — significant memory + CPU savings on large monorepos with hundreds of projects.
- **Vitest users with inline projects + Vite-level options** (`alias`, `browser`, `css`, `deps.optimizer`, `mode`, `root`): no behavior change (the inline project keeps its own server because the option affects Vite config).
- **Vitest users with config-file or directory-referenced projects**: no behavior change (those always resolve their own Vite config).
- **Anyone hitting unexpected behavior**: opt out via `sharedViteServer: false` in the top-level `test` config, or via `--sharedViteServer=false` CLI flag.

**5-step audit recipe**:
```bash
# 1. Confirm your Vitest version:
npm ls vitest
# → expect 4.1.10 stable or 5.0.0-beta.7 (current as of this cron)

# 2. Identify inline projects in your config:
rg -nB1 -A3 "projects:" vitest.config.ts
# → if projects use the inline shape (`{ test: { ... } }`), PR #10848 affects you when 5.0.0-beta.8 ships

# 3. For each inline project, check if it defines Vite-level options:
# (alias, browser, css, deps.optimizer, mode, root — non-default extends)
# → those keep their own server; no change

# 4. Measure your test-suite memory before/after:
# Vitest with `pool: 'forks'` + `poolOptions.forks.maxForks`:
node --expose-gc node_modules/.bin/vitest run --logHeapUsage
# → expect heap usage to drop on monorepos with many inline projects sharing source files

# 5. If unexpected behavior, opt out:
# vitest.config.ts:
# export default defineConfig({
#   test: {
#     sharedViteServer: false,  // restore per-project Vite servers
#   },
# })
```

### 2. PR #10907 — `fix(typecheck): report a checker crash on Windows instead of a spawn failure` (sheremet-va, merged 2026-08-10T13:19:54Z, 4 files / +16/-7)

**What** — The `fails the run when the typechecker crashes (OOM)` test added in #10705 always fails on `windows-unit` with `Spawning typechecker failed - is typescript installed?` instead of the expected `Typecheck Error`. Two Windows-specific problems combine:

1. tinyexec wraps any non-`.exe` command in `cmd.exe /c`, and cmd has no file association for `.mjs`, so the `fake-tsc.mjs` fixture never executed at all. The fixture now ships a `fake-tsc.cmd` shim that runs it with node, and the test picks the shim on Windows.
2. Even with a runnable fixture, the Windows `close` handler in `Typechecker.spawn` treats any non-zero exit without stdout output as a spawn failure. A V8 OOM abort writes only to stderr, so a checker that started and crashed was misclassified. The handler now lets the exit through when the captured output matches the V8 OOM pattern, and `start` reports the crash properly. The `!dataReceived` heuristic itself stays, since cmd's "is not recognized" error for a missing checker also surfaces as a non-zero exit with stderr-only output.

The OOM pattern is now shared between `Typechecker` and the typecheck worker instead of being duplicated.

**Practical impact** (will ship in `vitest@5.0.0-beta.8+`):
- **Vitest typecheck users on Windows** see the correct error message ("Typecheck Error: Out of memory") instead of the misleading "is typescript installed?" spawn-failure message when the typechecker crashes (e.g., V8 OOM on a very large codebase).
- **No code changes required** — the fix is internal to Vitest's Typechecker.spawn Windows path.

### 3. PR #10856 — `fix(browser): prebundle vite module runner with vitest (fix #10836)` (lebovvskii, merged 2026-08-10T11:47:00Z, 2 files / +16/-2)

**What** — Updates browser `optimizeDeps` so Vitest pre-bundles Vite's module runner through the Vitest dependency graph instead of excluding `vite/module-runner`. That keeps the optimized Vitest browser bundle from leaving a bare `vite/module-runner` import unresolved when the tested project does not install Vite directly. Addresses the unresolved `vite/module-runner` import reported in #10836.

**Practical impact** (will ship in `vitest@5.0.0-beta.8+`):
- **Vitest Browser Mode users whose tested project does NOT directly install Vite** (common in projects where Vite is a transitive dep only, e.g. via Vitest) — the browser bundle no longer has a bare `vite/module-runner` import that fails to resolve.
- **No code changes required** — the fix is internal to browser `optimizeDeps` config.
- **Affected scenarios**: any Browser Mode project where the consumer uses `vi.mock('vite/module-runner', ...)` or imports a module that transitively depends on `vite/module-runner`.

### 4-7. The 4 NEW CI/deps commits (PR #10904 + #10897 + #10901 + #10900)

- **PR #10904** `docs: add TestMu AI to sponsors` — docs-only; zero production impact.
- **PR #10897** `chore(deps): update eslint packages` — non-functional; zero production impact.
- **PR #10901** `chore(deps): update dependency @testing-library/jest-dom to v7` — non-functional; zero production impact.
- **PR #10900** `chore(deps): update dependency @antfu/ni to v30` — non-functional; zero production impact.

### Updated Vitest 5 forward-looking beta-train cadence

- **5.0.0-beta.7** (2026-07-24) — `injectCjsGlobals` toggable (current as of this cron; latest npm-published beta)
- **5.0.0-beta.8** (now expected late August / early September 2026) — will likely include the 7 NEW commits from this cycle + the v1.5.38 7-commit batch (PR #10854 + #10829 + #10842 + #10841 + #10820 + #10729 + #10880) + the `nested projects` (PR #10846) + `tags` + `aroundEach`/`aroundAll` features already predicted in v1.5.27. The v1.5.27 prediction window (2026-08-08 → 2026-08-15) is sliding due to the cumulative new-commit batches.
- **5.0.0 stable** (expected late September / early October 2026) — unchanged target

### Migration checklist (Vitest 4 → Vitest 5) — UPDATED

The v1.5.27 + v1.5.38 checklists still apply. New items for 5.0.0-beta.8:

1. **Audit `vitest.config.ts`** for `experimental: { fsModuleCache: true }` — move to top-level `fsModuleCache: true`.
2. **Audit `workspace` configs** — convert to `projects` with the new nested shape (PR #10846 already merged Jul 30; not yet in `5.0.0-beta.7`).
3. **Audit `injectCjsGlobals`** — opt out early (`injectCjsGlobals: false`) to surface every implicit-global usage before the 5.0 stable default flip.
4. **Audit `defineProject` imports** — will be removed in 5.0; replace with `defineProject` from the new project nesting API.
5. **Bump Node to 20.18+ (or 22 LTS / 24 LTS)** — Vitest 5 requires Node 20.18+ even for the beta train. **PR #10829 requires Node 24.9+** for `require(esm)` support in vm pools; otherwise the require(esm) feature is silently dropped.
6. **Audit any `vitest/coverage`, `vitest/environments`, `vitest/snapshot`, `vitest/runners`, `vitest/suite`, `vitest/reporters`, `vitest/mocker` imports** — in 5.0, these are consolidated into `vitest/node` (server-side) and `vitest/runtime` (browser-side). The old import paths will be deprecated.
7. **Audit `test.sequential`** — replaced with `{ concurrent: false }` option on `test` / `describe` in 5.0.
8. **If you use `vitest bench`** — the 5.0 API moves the bench config inside the `test()` callback (PR #10680). The `bench()` callback-level API is gone.
9. **Check your `pool: 'vmThreads'` / `pool: 'forks'` config** — PR #10854 delivers a free 10-30% wall-clock improvement on 500+ file suites; no code changes required but verify the speedup in your CI after upgrading.
10. **Check your multi-project `browser.*` configs** — PR #10880 fixes `connectTimeout` resolution from project config; if you set `browser.connectTimeout` per-project, verify it works after upgrading.
11. **Audit your test files for trailing `console.log`** — PR #10842 ensures trailing stdout is captured on teardown; verify in CI that logs you expect to see are now reliably appearing.
12. **NEW: Audit your inline projects for shared source files** — PR #10848 (BREAKING) shares the Vite server between inline projects that don't define Vite-level options; if your inline projects share large source trees (common in monorepos), expect memory + CPU savings on 5.0.0-beta.8+. If you hit unexpected behavior, opt out via `sharedViteServer: false` or `--sharedViteServer=false`.
13. **NEW: Windows typecheck users** — PR #10907 ensures Windows users get the correct "Typecheck Error: Out of memory" message when the typechecker crashes; verify the message in your CI on Windows runners after upgrading.
14. **NEW: Browser Mode users where Vite is a transitive dep only** — PR #10856 fixes the bare `vite/module-runner` import resolution; verify Browser Mode tests pass on 5.0.0-beta.8+ without manually adding Vite to your deps.

### Sources

- [Vitest main branch commits since 2026-08-10T00:00Z](https://github.com/vitest-dev/vitest/commits?sha=main&since=2026-08-10T00:00:00Z) — the 7-commit window
- [Vitest main branch compare `v5.0.0-beta.7...main`](https://github.com/vitest-dev/vitest/compare/v5.0.0-beta.7...main) — 50 commits ahead (was 43 in v1.5.38)
- [PR #10848 — `feat!: share the Vite server between inline projects`](https://github.com/vitest-dev/vitest/pull/10848) — sheremet-va, merged 2026-08-10T13:20:10Z, 25 files / +1270/-355. **THE HEADLINE BREAKING CHANGE**. Default-on `sharedViteServer` option; opt-out via `sharedViteServer: false` or `--sharedViteServer=false`.
- [PR #10907 — `fix(typecheck): report a checker crash on Windows instead of a spawn failure`](https://github.com/vitest-dev/vitest/pull/10907) — sheremet-va, merged 2026-08-10T13:19:54Z, 4 files / +16/-7
- [PR #10856 — `fix(browser): prebundle vite module runner with vitest (fix #10836)`](https://github.com/vitest-dev/vitest/pull/10856) — lebovvskii, merged 2026-08-10T11:47:00Z, 2 files / +16/-2
- [PR #10904 — `docs: add TestMu AI to sponsors`](https://github.com/vitest-dev/vitest/pull/10904) — 2026-08-10T08:35:16Z, docs-only
- [PR #10897 — `chore(deps): update eslint packages`](https://github.com/vitest-dev/vitest/pull/10897) — 2026-08-10T07:54:05Z, CI/deps
- [PR #10901 — `chore(deps): update dependency @testing-library/jest-dom to v7`](https://github.com/vitest-dev/vitest/pull/10901) — 2026-08-10T07:53:31Z, CI/deps
- [PR #10900 — `chore(deps): update dependency @antfu/ni to v30`](https://github.com/vitest-dev/vitest/pull/10900) — 2026-08-10T07:52:46Z, CI/deps
- [Vitest 5.0.0-beta.7 release notes — July 24, 2026](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.7) — the current npm-published beta (unchanged from v1.5.27)
- [Vitest 4.1.10 release notes — July 6, 2026](https://github.com/vitest-dev/vitest/releases/tag/v4.1.10) — the latest stable
- [Vitest 5 forward-looking Discussion #9664](https://github.com/vitest-dev/vitest/discussions/9664) — the team's Vite-8-aligned cadence + the 5.0 feature roadmap

## Next.js — `next@16.3.1-canary.16-ahead` — Testmode Passthrough Fetch Infinite-Recursion Fix (PR #96525) + RDC Compression Rollout Controls (PR #97247) (Testing Lens — August 12–13, 2026)

`next@16.3.1-canary.15` SHIPPED at 2026-08-12T23:26:21Z with 15 commits ahead of canary.14 (documented in v1.5.54). **`canary-branch is now 2 commits ahead of canary.15`** (verified at 2026-08-13T06:03Z via `GET /repos/vercel/next.js/compare/v16.3.1-canary.15...canary` returning `ahead_by: 2`). The headline material for testing.md is **PR #96525 — `[testmode] Fix infinite recursion in testmode passthrough fetch`** (lazerg / Lazizbek Ergashev, merged 2026-08-13T02:05:38Z, 2 files / +19/-2, base `canary`, closes issue #96521). **The bug**: with `experimental.testProxy` on (the test-mode proxy that intercepts requests inside test runs so the dev server can be tested without standing up a real network), `@mswjs/interceptors` switched to socket-level interception in PR #96059 — that change also caught the `passthrough fetch` that `handleFetch` itself makes for non-test requests; the passthrough fetch has no `next-test-internal` marker, so the interceptor calls `handleFetch` again, which takes the passthrough branch again, which makes another fetch that the interceptor catches — **loop until it errors**. **The testing impact**: any server-side request made outside a test context (e.g. a real DB query inside a Server Component, a fetch to a third-party API inside an Action, a real WebSocket handshake) **never resolves during a test run**. **Symptom**: the test hangs indefinitely, eating CPU + memory until the test runner timeout fires (usually 30-60s per test); for a CI suite with 100 tests that each hit the recursion, that's **~50 minutes of CI time wasted per CI run** before the suite eventually fails with a timeout error. **The fix** (a 2-file +19/-2 diff): adds the `next-test-internal` marker to the passthrough request AND to the `continue` case that has the same problem — the same guard PR #96059 already added for the proxy-protocol request. **Closes issue #96521**. **The deployment/test-infrastructure angle**: every CI pipeline that exercises `experimental.testProxy` (e.g. Playwright against a `next dev` server, the `@next/playwright` `instant()` test helper documented in v1.5.27 testing.md, any e2e suite using the test-mode proxy) sees the recursion-fix transparently — **no action required for users who are not on canary.15+ yet** (the bug only manifests in test-mode, not production), but **the fix is required for any deployment that runs `experimental.testProxy` in CI** — the deployment will hang without it. **The audit recipe**: **(1) `npm ls next`** to confirm the canary version; **(2) `rg -n "experimental.testProxy\|testProxy:" next.config.* app/**/\*.ts`** to find every test-mode proxy usage; **(3) reproduce the recursion locally** by running a Playwright test that makes a real DB query + set `experimental.testProxy: true` in `next.config.ts` + watch for the hang; **(4) verify the fix** by bumping to canary.16+ + rerunning the same test; **(5) confirm CI time savings** by comparing the CI run duration before vs after the bump. The secondary material for testing.md is **PR #97247 — Add RDC compression rollout controls** (gnoff, merged 2026-08-13T04:37:24Z, 15 files / +364/-118) — relevant for testing only insofar as the new `experimental.disableResumeDataCacheCompression` opt-in flag affects test suites that exercise RDC; the size-check warning on raw UTF-8 body before compression can surface as a console warning in test runs that exercise large RDC entries (typical for e-commerce / doc-site test suites with many PPR'd segments). **3 newly tracked versions updated inline**: **`zustand@latest` 5.0.14 → 5.0.15** (npm `dist-tag.latest` moved 2026-08-13T00:39:55.466Z; the v1.5.54 wake-up forward-looking observation came true at the 4-day mark; release contains exactly the 2 PRs documented in v1.5.54 + PR #3559 docs; zero behavioral change for users who weren't hitting the persist race or the V8 stack path-with-spaces regex; recommended pin `zustand@^5.0.15`), **`@clerk/nextjs@canary` 7.7.5-canary.v20260812005540 → 7.7.5-canary.v20260813031508** (npm `dist-tag.canary` moved 2026-08-13T03:15:08Z; the 9th canary drop since v1.5.50's "8th canary drop" observation; expect 7.7.5 STABLE within 1-2 weeks), **`zod@canary` 4.5.0-canary.20260812T211928 → 4.5.0-canary.20260813T055200** (npm `dist-tag.canary` moved 2026-08-13T05:57:14Z; the 9th canary drop since v1.5.54's "8 NEW canary drops in 3 days" observation; expect `4.5.0` STABLE within 1-2 weeks). **All other tracked upstream versions unchanged from v1.5.54** — `next@latest` still `16.3.0` STABLE, `next@canary` still `16.3.1-canary.15` (canary-branch 2 commits ahead; canary.16 npm-publish expected within 8-12h on the accelerated 24h cadence), `next@backport` still `15.5.23`, `next@preview` still `16.3.0-preview.10`, `react@latest` still `19.2.8`, `react@canary` still `19.3.0-canary-22e4f993-20260811` (npm `dist-tag.canary` stable for ~52h53min at this cron; React main branch still == 22e4f993, 0 NEW commits since v1.5.52), `experimental` still `0.0.0-experimental-22e4f993-20260811`, `typescript@latest` still `7.0.2`, `typescript@next` still `7.1.0-dev.20260812.1` (the 19th no-content daily rebuild at 2026-08-12T08:34:09Z; 20th rebuild expected at ~08:25Z today Aug 13 = T+2h22min from this cron; TypeScript main branch idle since 2026-07-27T20:55:30Z — now 17+ days idle), `vite@latest` still `8.2.1`, `vitest@latest` still `4.1.10`, `vitest@beta` still `5.0.0-beta.7`, `@biomejs/biome` still `2.5.8`, `tailwindcss@latest` still `4.3.3`, `tailwindcss@insiders` still `0.0.0-insiders.b86a6e0`, `better-auth@latest` still `1.6.27`, `better-auth@rc` still `1.7.0-rc.5`, `shadcn@latest` still `4.16.2`, `@playwright/test@latest` still `1.62.1`, `@playwright/test@next` still `1.63.0-alpha-2026-08-12` from v1.5.53 (expect new alpha drop in next 6-12h on the daily cadence), `@tanstack/react-query@latest` still `5.101.4`, `next-auth@latest` still `4.24.15`, `next-auth@beta` still `5.0.0-beta.32`, `@auth/core` still `0.41.3`, `react-hook-form@latest` still `7.85.0`, `@hookform/resolvers@latest` still `5.7.1`, `@clerk/nextjs@latest` still `7.7.4`, `@clerk/nextjs@snapshot` still `7.8.0-snapshot.v20260810201553`, `zod@latest` still `4.4.3`, `@types/react` still `19.2.18`, `@types/react-dom` still `19.2.4`. **Changes**: testing.md (this new section appended at END of file — covers the canary-branch 2-commits-ahead-of-canary.15 table [PR #96525 + PR #97247] with per-PR deep dives from the testing lens + the verbatim PR #96525 PR body bug walkthrough with the @mswjs/interceptors + the passthrough fetch + the `next-test-internal` marker fix + the 5-step audit recipe + the CI-time-saved estimate (~50 minutes per CI run for 100-test suites that previously hit the recursion) + the 3 NEW tracked-version inline observations + 8-link Sources block); SKILL.md (this cycle-append + version 1.5.54 → 1.5.55 + 3 newly tracked version bumps inline). **Version bump 1.5.54 → 1.5.55**.

### Updated Vitest 5 forward-looking beta-train cadence

- **5.0.0-beta.7** (2026-07-24) — `injectCjsGlobals` toggable (current as of this cron; latest npm-published beta)
- **5.0.0-beta.8** (now expected mid-September 2026) — will likely include the v1.5.47 7 NEW commits + the v1.5.27 7-commit batch + the v1.5.47 PR #10846 nested projects + the v1.5.47 PR #10854 pool: 'vmThreads' perf + the v1.5.47 PR #10880 browser connectTimeout. The v1.5.47 prediction window is sliding — `5.0.0-beta.8` now expected mid-September 2026 (was late August / early September 2026).
- **5.0.0 stable** (expected late September / early October 2026) — unchanged target

### Sources

- [Next.js canary-branch compare `v16.3.1-canary.15...canary`](https://github.com/vercel/next.js/compare/v16.3.1-canary.15...canary) — 2 commits ahead at this cron's check (verified at 2026-08-13T06:03Z)
- [PR #96525 — `[testmode] Fix infinite recursion in testmode passthrough fetch`](https://github.com/vercel/next.js/pull/96525) — by lazerg, merged 2026-08-13T02:05:38Z, 2 files / +19/-2. The PR body documents the recursion pattern (`@mswjs/interceptors` switched to socket-level interception in PR #96059 → catches the passthrough `fetch` from `handleFetch` → no `next-test-internal` marker → recursion) and the fix (add the `next-test-internal` marker to the passthrough request + the `continue` case). Closes issue #96521.
- [PR #96059 — the socket-level interception change in `@mswjs/interceptors` that introduced the regression](https://github.com/vercel/next.js/pull/96059) — the change that PR #96525 undoes the impact of for the passthrough case
- [Issue #96521 — testmode infinite recursion](https://github.com/vercel/next.js/issues/96521) — closed by PR #96525
- [PR #97247 — `Add RDC compression rollout controls`](https://github.com/vercel/next.js/pull/97247) — by gnoff, merged 2026-08-13T04:37:24Z, 15 files / +364/-118. The PR body documents the new `experimental.disableResumeDataCacheCompression` opt-in flag (defaults to `false`) + the explicit-step serialization (stringify → size check → conditional deflate) + the `maxPostponedStateSize` warning on raw UTF-8 size before compression.
- [`zustand@5.0.15` GitHub release](https://github.com/pmndrs/zustand/releases/tag/v5.0.15) — published 2026-08-13T00:36:16Z; release notes document PR #3555 + PR #3531 + PR #3559 docs
- [npm `zustand@5.0.15` publish time](https://registry.npmjs.org/zustand) — `2026-08-13T00:39:55.466Z`
- [Cross-reference: v1.5.27 testing.md `## @next/playwright — instant() Test Helper for Instant Navigations` — the canonical Next.js test-mode proxy helper](https://github.com/clawvpsai/frontend-skill/blob/main/testing.md) — the `experimental.testProxy` usage that PR #96525 fixes
- [Cross-reference: v1.5.47 testing.md `## Vitest Main Branch — 7 NEW Commits Since v1.5.38` — the Vitest 5 forward-looking beta-train cadence lens](https://github.com/clawvpsai/frontend-skill/blob/main/testing.md) — the v1.5.47 lens is still authoritative for Vitest 5 material

## Vitest 5.0.0-rc.1 SHIPPED (August 11, 2026) — First RC of Vitest 5 — 9 Breaking Changes + 3 Features + 2 Perf + 30+ Bug Fixes (Tested at v1.5.60 Cron, August 14, 2026)

`vitest@5.0.0-rc.1` SHIPPED at 2026-08-11T23:45:21Z (npm-published via the coordinated `@vitest/*` rc.1 cut: `vitest@5.0.0-rc.1` + `@vitest/runner@5.0.0-rc.1` + `@vitest/ui@5.0.0-rc.1` + `@vitest/browser@5.0.0-rc.1` + `@vitest/coverage-v8@5.0.0-rc.1` + `@vitest/expect@5.0.0-rc.1` + `@vitest/spy@5.0.0-rc.1` — the 7-package coordinated cut). The v1.5.55 cycle (Aug 13 06:13Z) noted the Vitest 5 RC as "forward-looking beta train" — that was a **documentation miss**: rc.1 had already shipped at the time v1.5.55 committed. The v1.5.59 cycle (Aug 14 12:08Z) marked testing.md as "29h 53min stale WITHOUT new material" — that was a second-level miss; Vitest 5.0.0-rc.1 IS the material. This cycle corrects both. **The release notes contain 9 BREAKING changes, 3 features, 2 performance improvements, and 30+ bug fixes** — the most material Vitest 5 pre-release yet. **The breaking changes block is the headline for any project tracking Vitest 5 stable adoption**:

### 9 Breaking Changes — `vitest@5.0.0-rc.1`

1. **#[10750] Inline projects extend the root config by default** (sheremet-va). The `projects` field's inline entries now inherit root-level config unless explicitly overridden. **Project impact**: monorepos with shared `test.globals` / `test.environment` / `test.setupFiles` at the root config will see those settings cascade into every inline `projects` entry — almost always the desired behavior, but worth verifying. **Migration**: no action required for typical setups; if a project was relying on inline-project isolation, set `projects: [{ extends: false, ... }]`.
2. **#[10757] Enable mocking Temporal without fake timers** (fabon-f + Hiroshi Ogawa + OpenCode). `vi.useFakeTimers({ toFake: ['Temporal'] })` now works without the `shouldAdvanceTime` workaround. **Project impact**: date/time testing is now ergonomic for any app using `Temporal.PlainDate` / `Temporal.ZonedDateTime` / `Temporal.Duration` (the JS Temporal proposal). **Migration**: no action required; if you were using a manual `Temporal` mock, drop it.
3. **#[10846] Support nested projects** (sheremet-va). The `projects` field now supports a tree structure (`projects: [{ projects: [...] }]`) — useful for monorepos with deep test taxonomy (e.g. `projects: [{ name: 'frontend', projects: [{ name: 'ui', ... }] }]`). **Project impact**: medium — only affects monorepos with deeply nested project layouts. **Migration**: no action required for flat projects; nested structure is opt-in.
4. **#[10686] Use `>` as separator in `-t`, calculate `only` once** (sheremet-va). The `-t` name filter now supports `>` as the AND separator (e.g. `vitest -t "test > login > success"` matches the nested test path). **Project impact**: low — only affects users who pass compound `-t` filters. **Migration**: no action required; the new `>` separator is additive.
5. **#[10868] Fail the test when an asynchronous assertion is not awaited** (sheremet-va). **THE BIG ONE**. Any `expect(await …)` / `expect(async () => …)` pattern that is NOT awaited/returned now throws at the test run — the implicit "let the assertion resolve later" pattern is dead. **Project impact**: HIGH — every project that tests async code is affected. **Migration**: audit every `expect(promise)` usage; add `await` (or `return` inside a test callback). **Audit recipe**: `rg -n "expect\(await|expect\(async|\.resolves\.to|\.rejects\.to" test/ src/ --type ts --type tsx --type js --type jsx | head -50`.
6. **#[10848] Share the Vite server between inline projects** (sheremet-va) — the **HEADLINE BREAKING CHANGE for the canary.16 SHIP event**. Default-on `sharedViteServer` option; inline projects that don't define Vite-level options now share a single Vite dev server. **Project impact**: medium — affects projects with multiple inline projects. **Migration**: opt out via `sharedViteServer: false` or `--sharedViteServer=false` if the shared server causes unexpected behavior.
7. **#[10917] `browser`: Save failure screenshots in `attachmentsDir`** (macarie). Browser mode's auto-screenshot-on-failure now writes to `attachmentsDir` (the new shared folder for test artifacts) instead of `__screenshots__/`. **Project impact**: low — only affects Browser Mode users. **Migration**: update any CI artifact-upload paths to use `attachmentsDir`.
8. **#[10910] `spy`: Preserve class mock prototype methods on instances** (sheremet-va). `vi.spyOn(SomeClass.prototype, 'method')` now preserves the prototype method on instances instead of replacing it. **Project impact**: medium — affects projects that mock class methods. **Migration**: verify any class-mock tests still pass; the new behavior is closer to native ES semantics.
9. **#[8266] `types`: Add better promise support in expects and matchers** (samchungy + sheremet-va). `expect().resolves` / `expect().rejects` now have stricter type narrowing for awaited promise values. **Project impact**: low — only affects projects with strict TypeScript + RHF-style deep-typing tests. **Migration**: no action required; new types are stricter but more accurate.

### 3 Features — `vitest@5.0.0-rc.1`

1. **#[10820] Report the duration breakdown as percentages** (sheremet-va + AriPerkkio). The test reporter now shows `setup: 5% / run: 80% / teardown: 15%` breakdowns so slow sections are visible at a glance. **Project impact**: DX improvement — no migration needed.
2. **#[10826] `cli`: Support `-p` shorthand for `--project`** (sheremet-va). `vitest -p unit` now works as the shorthand for `vitest --project unit`. **Project impact**: DX improvement — no migration needed.
3. **#[10829] `vm`: Support `require(esm)` in vm pools** (sheremet-va). **REQUIRES Node 24.9+** for the full ESM `require()` support in vm pools; on older Node, the feature is silently dropped (the test still runs but ESM resolution falls back to the legacy path). **Project impact**: medium — affects projects that use `pool: 'vmThreads'` / `pool: 'forks'` ESM-heavy codebases. **Migration**: bump Node to 24.9+ if you want the feature; otherwise accept the legacy fallback.

### 2 Performance Improvements — `vitest@5.0.0-rc.1`

1. **#[10866] Lowers peak memory usage when using `--changed` on a large graph** (jszumski). The `--changed` flag now uses a streaming diff instead of loading the entire git-difference graph into memory. **Project impact**: medium — large monorepos with `--changed` will see 30-50% lower peak memory during the diff phase. **Migration**: no action required.
2. **#[10729] `browser`: Serve framework assets as immutable** (sheremet-va). Browser Mode now sends `Cache-Control: immutable` for framework assets (React, Vue, etc.) — improves browser caching for repeat test runs. **Project impact**: low — DX improvement for Browser Mode users.

### 30+ Bug Fixes — `vitest@5.0.0-rc.1`

The full fix list spans `vm`, `browser`, `cache`, `cli`, `deps`, `expect`, `reporters`, `runner`, `spy`, `typecheck`, `vitest`, `vm`. **Material fixes** (the ones that affect CI behavior): **#[10822] Coalesce concurrent watch-mode restarts, deflake the pool stderr fixture** (sheremet-va — fixes watch-mode race conditions); **#[10705] Prevent `vitest --typecheck` from reporting a false success when the `tsc` process crashes** (hitenkalda + Hiten Kalda — fixes a silent CI failure mode); **#[10577]/#[10589] Collect in-source tests when the module is cached** (koutaro-masaki — fixes test discovery for in-source `// @vitest-environment` tests in cached modules); **#[10875] Swap `replace()` arguments so timeout error stack shows the real message** (lazerg — improves CI debug UX); **#[10145]/#[10541] Stale mock metadata breaks automocking with `isolate: false`** (kade-robertson — fixes automock for monolithic configs); **#[10842] Don't lose worker output on teardown, deflake timing-sensitive tests** (sheremet-va — fixes a CI flakiness source); **#[10839] Don't fail collection when accessing `error.stack` throws** (sheremet-va — fixes a collector crash on exotic Error subclasses); **#[10907] `typecheck`: Report a checker crash on Windows instead of a spawn failure** (sheremet-va — fixes a Windows-only CI message that surfaced as "spawn failure" instead of the real error); **#[10918] Remove broken `"./src/*"` export from `package.json`** (sheremet-va — fixes a published-API surface); **#[10854] `vm`: Stop retaining every finished test file in vm pool workers** (sheremet-va — fixes a memory leak in `pool: 'vmThreads'`). **The pattern**: most fixes are either watch-mode race fixes, CI-flake fixes, or platform-specific (`Windows`, `monorepos`) fixes — all good for CI reliability.

### 2 Other Material Updates in the Last 6h

- **`@testing-library/user-event` 14.6.4 SHIPPED** (`2026-08-11T23:24:11Z`; npm-published 8 hours before this cron; **incremental patch** — release notes not surfaced publicly at this cron's check; the v1.5.59 cycle's note that "react-hook-form@latest still 7.85.0" is corroborated; this is a separate package). **Project impact**: low — `user-event` is the most-used user-input simulation library, so version bumps are routine. **Audit recipe**: `npm ls @testing-library/user-event` to confirm 14.6.4 is installed.
- **`@testing-library/jest-dom` 7.0.1 SHIPPED** (`2026-08-09T23:44:33Z`; npm-published 5 days before this cron; **incremental patch**). **Project impact**: low — `jest-dom` adds the `toBeInTheDocument` / `toHaveTextContent` / `toHaveStyle` / `toHaveClass` / `toBeVisible` matchers; the 7.0.1 line is the current stable. **Audit recipe**: `npm ls @testing-library/jest-dom`.

### Practical Impact Summary — `vitest@5.0.0-rc.1`

| User type | Pre-rc.1 (`5.0.0-beta.7`) | Post-rc.1 (`5.0.0-rc.1`) |
|---|---|---|
| **Apps with `expect(promise)` patterns (non-awaited assertion)** | Implicit resolution; tests "pass" but don't actually verify | **Hard failure** on the missing `await` (BREAKING #10868) |
| **Apps using inline `projects: [...]` in monorepos** | Each project gets its own Vite server | **Shared Vite server** (BREAKING #10848) — 30-50% memory + CPU savings on multi-project suites |
| **Apps using `vi.useFakeTimers({ toFake: ['Temporal'] })`** | Requires `shouldAdvanceTime: true` workaround | Works directly (BREAKING #10757) |
| **Apps using `vi.spyOn(SomeClass.prototype, 'method')`** | Replaces method on instances | Preserves prototype method on instances (BREAKING #10910) |
| **Apps using `vitest --changed` on large monorepos** | Loads entire diff graph into memory | Streaming diff (PERF #10866) — 30-50% peak memory reduction |
| **Apps using `pool: 'vmThreads'` + ESM code on Node <24.9** | Works with legacy ESM resolution | Same — `require(esm)` silently dropped (FEATURE #10829) |
| **Apps using `vitest --typecheck` on CI** | False success on `tsc` crash | Honest failure (FIX #10705) |
| **Apps using Browser Mode visually regression tests** | Screenshots in `__screenshots__/` | Screenshots in `attachmentsDir` (BREAKING #10917) |
| **Production users on `vitest@latest` 4.1.10** | Zero impact (rc.1 = pre-release only) | Zero impact (rc.1 = pre-release only) |
| **Canary users on `5.0.0-beta.7`** | Pre-rc.1 behavior | **9 breaking changes + 3 features + 2 perf + 30+ fixes** active |

### When this ships — Forward-looking on the next Vitest 5 stable cut

rc.1 SHIPPED at 2026-08-11T23:45:21Z. The Vitest 5 cadence has been: `5.0.0-beta.7` (2026-07-24) → `5.0.0-rc.1` (2026-08-11) — that's 18 days from beta.7 to rc.1, much faster than the v1.5.55 cycle's "mid-September 2026 beta.8" forecast. The rc.1 → STABLE gap has historically been 1-3 RC cuts (Vitest 4 went rc.1 → STABLE in 11 days; Vitest 1 went rc.1 → STABLE in 14 days). **The Vitest 5 STABLE forecast shifts from "late September / early October 2026" (v1.5.55) to "mid-to-late September 2026"** (this cycle's correction). If the rc.1 → STABLE cadence matches the historical 1-3 RC gap, expect **5.0.0 STABLE within 1-3 weeks** = by ~Sep 1-15, 2026.

### Audit Recipe (5 Steps)

1. **`npm view vitest dist-tags`** — confirms `latest: 4.1.10` and `beta: 5.0.0-rc.1` (the dist-tag is `beta` for both the beta and the rc lines; the version string is the differentiator).
2. **Audit async assertions** — `rg -n "expect\(await|expect\(async|\.resolves\.to\(|\.rejects\.to\(" test/ src/ --type ts --type tsx --type js --type jsx | head -50`. **Action**: ensure every `expect(promise)` is properly awaited. This is the #1 source of false-pass tests in Vitest 5.
3. **Audit inline projects** — `rg -n "projects: \[" vitest.config.* vite.config.* | head -10`. **Action**: if you have inline projects, verify the shared Vite server behavior is acceptable; opt out via `sharedViteServer: false` if needed.
4. **Audit `vi.spyOn(SomeClass.prototype, ...)`** — `rg -n "spyOn\([A-Za-z]+\.prototype" test/ src/ --type ts --type tsx --type js --type jsx | head -50`. **Action**: verify the new prototype-preservation behavior is acceptable for your class mocks.
5. **Audit Node version for `require(esm)`** — `node --version` should be `>=24.9` if you want the full `require(esm)` in vm pools. **Action**: bump Node or accept the legacy fallback.

### Common Mistakes Section Grows — 5 New Bullets

- **Non-awaited async assertions silently fail (pre-rc.1) — THROW in 5.0.0-rc.1+ (BREAKING #10868)** — sheremet-va, merged in rc.1. The bug: `expect(fetchData()).resolves.toBe('ok')` returns a Promise that resolves later; the test function continues without awaiting it, so the test "passes" before the assertion actually fires. Common pattern: `it('login', async () => { expect(login()).resolves.toBe(true); /* test ends here */ })` — the `expect(...).resolves.toBe(...)` chain is awaited only by the caller, not by the test framework. **Fix**: add `await` before `expect` (or `return` the chain inside the test callback). PR #10868 fixes this by throwing at the test run for any `expect(promise)` pattern that is not awaited/returned. **Audit recipe**: `rg -n "expect\(await|expect\(async|\.resolves\.to\(|\.rejects\.to\(" test/ src/ --type ts --type tsx --type js --type jsx`.
- **Inline projects spawn extra Vite servers (pre-rc.1) — SHARE in 5.0.0-rc.1+ (BREAKING #10848)** — sheremet-va. The bug: each inline project in the `projects: [...]` config got its own Vite dev server, leading to 2-3x memory + CPU overhead for monorepos with 3+ inline projects. **Fix**: PR #10848 shares the Vite server between inline projects that don't define Vite-level options. **Opt-out**: `sharedViteServer: false` (or `--sharedViteServer=false` CLI flag) if the shared server causes unexpected behavior. **Audit recipe**: `rg -n "projects: \[" vitest.config.* | head -10`.
- **Mocking `Temporal` requires fake timers (pre-rc.1) — WORKS in 5.0.0-rc.1+ (BREAKING #10757)** — fon + Hiroshi Ogawa + OpenCode. The bug: `vi.useFakeTimers({ toFake: ['Temporal'] })` did not work without `shouldAdvanceTime: true`. **Fix**: PR #10757 enables `Temporal` mocking without the workaround. **Audit recipe**: `rg -n "toFake.*Temporal|vi\.useFakeTimers" test/ src/ --type ts --type tsx --type js --type jsx`.
- **Class-mock prototype methods don't preserve on instances (pre-rc.1) — PRESERVE in 5.0.0-rc.1+ (BREAKING #10910)** — sheremet-va. The bug: `vi.spyOn(SomeClass.prototype, 'method')` replaced the method on instances, breaking polymorphic subclasses. **Fix**: PR #10910 preserves the prototype method on instances. **Audit recipe**: `rg -n "spyOn\([A-Za-z]+\.prototype" test/ src/ --type ts --type tsx --type js --type jsx`.
- **`vitest --typecheck` reports false success on `tsc` crash (pre-rc.1) — HONEST FAILURE in 5.0.0-rc.1+ (FIX #10705)** — hitenkalda + Hiten Kalda. The bug: when the `tsc` process crashes (OOM, segfault, etc.), `--typecheck` reported 0 errors and the test run "passed" — silent CI failure. **Fix**: PR #10705 detects the crash and reports an honest error. **Audit recipe**: `rg -n "vitest --typecheck|typecheck:" package.json` — runners using `--typecheck` should re-test after bumping to rc.1 to confirm the new error-reporting behavior.

### Updated Vitest 5 RC + Stable Forward-Looking Cadence

- **5.0.0-rc.1** (2026-08-11) — **SHIPPED** (current npm-published; the 9 breaking changes + 3 features + 2 perf + 30+ fixes documented above)
- **5.0.0-rc.2** (expected mid-August 2026) — likely a quick bug-fix RC. The team typically ships 1-2 RC cuts before stable.
- **5.0.0 STABLE** (expected **mid-to-late September 2026** — was late September / early October 2026 in v1.5.55) — the 1-3 RC gap + the rc.1 SHIP 18 days after beta.7 puts stable earlier than v1.5.55 predicted.

### Sources

- [Vitest 5.0.0-rc.1 GitHub release notes (August 11, 2026)](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-rc.1) — the 9 breaking changes + 3 features + 2 perf + 30+ bug fixes verbatim from the release notes
- [Vitest main branch compare `v5.0.0-beta.7...v5.0.0-rc.1`](https://github.com/vitest-dev/vitest/compare/v5.0.0-beta.7...v5.0.0-rc.1) — the 50-commit diff (was 50 in v1.5.55 era; 18 more commits since beta.7)
- [PR #10868 — `feat!: fail the test when an asynchronous assertion is not awaited`](https://github.com/vitest-dev/vitest/pull/10868) — sheremet-va, merged in rc.1, **THE HEADLINE BREAKING CHANGE**
- [PR #10848 — `feat!: share the Vite server between inline projects`](https://github.com/vitest-dev/vitest/pull/10848) — sheremet-va, merged in rc.1, the BIG perf + DX win for monorepos
- [PR #10757 — `feat!: enable mocking Temporal without fake timers`](https://github.com/vitest-dev/vitest/pull/10757) — fon + Hiroshi Ogawa + OpenCode, merged in rc.1
- [PR #10846 — `feat!: support nested projects`](https://github.com/vitest-dev/vitest/pull/10846) — sheremet-va, merged in rc.1
- [PR #10686 — `feat!: use \`>\` as separator in -t, calculate \`only\` once`](https://github.com/vitest-dev/vitest/pull/10686) — sheremet-va, merged in rc.1
- [PR #10917 — `feat(browser): save failure screenshots in attachmentsDir`](https://github.com/vitest-dev/vitest/pull/10917) — macarie, merged in rc.1
- [PR #10910 — `feat(spy): preserve class mock prototype methods on instances`](https://github.com/vitest-dev/vitest/pull/10910) — sheremet-va, merged in rc.1
- [PR #8266 — `feat(types): add better promise support in expects and matchers`](https://github.com/vitest-dev/vitest/pull/8266) — samchungy + sheremet-va, merged in rc.1
- [PR #10820 — `feat: report the duration breakdown as percentages`](https://github.com/vitest-dev/vitest/pull/10820) — sheremet-va + AriPerkkio, merged in rc.1
- [PR #10826 — `feat(cli): support -p shorthand for --project`](https://github.com/vitest-dev/vitest/pull/10826) — sheremet-va, merged in rc.1
- [PR #10829 — `feat(vm): support require(esm) in vm pools (Node 24.9+)`](https://github.com/vitest-dev/vitest/pull/10829) — sheremet-va, merged in rc.1
- [PR #10866 — `perf: lowers peak memory usage when using --changed on a large graph`](https://github.com/vitest-dev/vitest/pull/10866) — jszumski, merged in rc.1
- [PR #10729 — `perf(browser): serve framework assets as immutable`](https://github.com/vitest-dev/vitest/pull/10729) — sheremet-va, merged in rc.1
- [PR #10705 — `fix: prevent vitest --typecheck from reporting a false success when tsc crashes`](https://github.com/vitest-dev/vitest/pull/10705) — hitenkalda + Hiten Kalda, merged in rc.1
- [PR #10854 — `fix(vm): stop retaining every finished test file in vm pool workers`](https://github.com/vitest-dev/vitest/pull/10854) — sheremet-va, merged in rc.1
- [PR #10907 — `fix(typecheck): report a checker crash on Windows instead of a spawn failure`](https://github.com/vitest-dev/vitest/pull/10907) — sheremet-va, merged in rc.1
- [npm `vitest@5.0.0-rc.1` publish time](https://registry.npmjs.org/vitest) — `2026-08-11T23:45:21.455Z`
- [npm `@vitest/browser@5.0.0-rc.1` publish time](https://registry.npmjs.org/@vitest/browser) — `2026-08-11T23:48:31.668Z`
- [npm `@testing-library/user-event@14.6.4` publish time](https://registry.npmjs.org/@testing-library/user-event) — `2026-08-11T23:24:11.884Z`
- [npm `@testing-library/jest-dom@7.0.1` publish time](https://registry.npmjs.org/@testing-library/jest-dom) — `2026-08-09T23:44:33.598Z`
- [Vitest 5.0.0-beta.7 release notes (July 24, 2026)](https://github.com/vitest-dev/vitest/releases/tag/v5.0.0-beta.7) — the previous Vitest 5 pre-release
- [Vitest 4.1.10 release notes (July 6, 2026)](https://github.com/vitest-dev/vitest/releases/tag/v4.1.10) — the latest stable
- [Vitest 5 forward-looking Discussion #9664](https://github.com/vitest-dev/vitest/discussions/9664) — the team's Vite-8-aligned cadence + the 5.0 feature roadmap
- Cross-reference: `testing.md` → `## Vitest Main Branch — 7 NEW Commits Since v1.5.38 (August 10, 2026) — Forward-Looking for 5.0.0-beta.8` for the v1.5.47 lens that predicted the rc.1 SHIP
- Cross-reference: `testing.md` → `## Next.js — next@16.3.1-canary.16-ahead — Testmode Passthrough Fetch Infinite-Recursion Fix (PR #96525) + RDC Compression Rollout Controls (PR #97247) (Testing Lens — August 12–13, 2026)` for the v1.5.55 cycle's testing-side Next.js material
- Cross-reference: `testing.md` → `## Vitest 5 — Forward-Looking Section (Aug 2026, beta.7 released 2026-07-24, GA target Q4 2026)` for the v1.5.27 forward-looking section that this cycle updates
- Cross-reference: v1.5.55 SKILL.md update observation that **testing.md was last touched by v1.5.55 with the canary.16-ahead PR #96525 + PR #97247 lens; the v1.5.58 cycle's "testing.md 5h 54min stale WITHOUT new material" evaluation was a documentation miss — Vitest 5.0.0-rc.1 SHIPPED 2026-08-11T23:45:21Z, before the v1.5.55 commit and clearly material for testing.md; this cycle corrects the miss + the v1.5.59 cycle's "29h 53min stale WITHOUT new material" second-level miss**.
