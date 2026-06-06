# Zustand — Client State Management

> **Note:** Zustand v5 (5.0.14) is current. The patterns below document v5. If you're on v4, see the [v4→v5 migration guide](https://zustand.docs.pmnd.rs/reference/migrations/migrating-to-v5). Key breaking changes: no auto-`act()` in tests, `subscribeWithSelector` moved to separate middleware import, `combine` requires explicit type.

## v5 Migration: Key Breaking Changes

### `subscribeWithSelector` — Separate Middleware

In v5, `subscribeWithSelector` is no longer built into `create` — import it separately:

```ts
// ❌ v4 pattern — broken in v5
const useStore = create((set) => ({
  count: 0,
  inc: () => set(s => ({ count: s.count + 1 })),
}))
// subscribeWithSelector was enabled by default
useStore.subscribe((state) => { /* selector */ })

// ✅ v5 pattern — import the middleware
import { create } from 'zustand'
import { subscribeWithSelector } from 'zustand/middleware'

const useStore = create(
  subscribeWithSelector((set) => ({
    count: 0,
    inc: () => set(s => ({ count: s.count + 1 })),
  }))
)
useStore.subscribe((state) => { /* selector — now works */ })
```

### Testing: No Auto-`act()` Wrapping

v4 auto-wrapped state updates in React Testing Library's `act()` during tests. v5 removes this — you must wrap updates manually:

```ts
// ❌ v4 pattern — act() auto-wrapped (broken in v5)
await user.click(button)
expect(screen.getByText('1')).toBeInTheDocument()

// ✅ v5 pattern — wrap updates in act() manually
import { act } from 'react-dom/test-utils'

await act(async () => { user.click(button) })
expect(screen.getByText('1')).toBeInTheDocument()
```

### `combine` — Explicit Type Required

```ts
// ❌ v4 — implicit type (broken in v5)
const useStore = create(
  combine({ count: 0 }, (set) => ({ inc: () => set(s => ({ count: s.count + 1 })) }))
)

// ✅ v5 — explicit type parameter
const useStore = create(
  combine({ count: 0 } as { count: number }, (set) => ({
    inc: () => set(s => ({ count: s.count + 1 })),
  }))
)
```

## When to Use Zustand

> **Note:** For foundational Zustand patterns (basic store, selectors, persist middleware, Immer, slice pattern), see **`state.md`**. This file covers only content not found in `state.md`.

Use Zustand for **client-side UI state** only. Not for server data — that's React Query's job.

**Good Zustand use cases:**
- Theme (light/dark/system)
- Sidebar open/closed state
- Modal visibility
- Shopping cart
- Current user's preferences (not auth — that's NextAuth)
- Global UI flags (e.g., "show onboarding")

**Not Zustand use cases:**
- API data / fetched content → React Query
- Auth sessions → NextAuth session
- Form state → React Hook Form (with server action for submission)

## Async Actions with Loading/Error State

For async operations managed in Zustand (e.g., polling, WebSocket updates):

```tsx
interface AsyncStore {
  user: User | null
  isLoading: boolean
  error: string | null
  fetchUser: (id: string) => Promise<void>
}

export const useUserStore = create<AsyncStore>((set) => ({
  user: null,
  isLoading: false,
  error: null,

  fetchUser: async (id) => {
    set({ isLoading: true, error: null })
    try {
      const user = await api.getUser(id)
      set({ user, isLoading: false })
    } catch (err) {
      set({ error: (err as Error).message, isLoading: false })
    }
  },
}))
```

**When to use Zustand for async vs React Query:**
- **Zustand async** — when the data is tied to a UI action (button triggers fetch, show result in store)
- **React Query** — for cached, shared server data that needs automatic refetching, background sync, pagination

## Resetting State

Reset to initial state — useful for auth logout, form clear, wizard restart:

```tsx
interface Store {
  count: number
  step: number
  data: Partial<FormData>
  reset: () => void
}

const initialState = {
  count: 0,
  step: 1,
  data: {},
}

export const useStore = create<Store>((set) => ({
  ...initialState,

  reset: () => set(initialState),
}))
```

**Use in logout flow:**

```tsx
// On sign-out, reset all client stores
async function signOut() {
  await nextAuthSignOut()
  useUIStore.getState().reset()
  useCartStore.getState().reset()
  router.push('/login')
}
```

## Common Mistakes

- **Selecting entire store** — causes re-renders when any field changes
  - ❌ `const store = useStore()`
  - ✅ `const count = useStore(s => s.count)`
- **Using Zustand for server data** — use React Query instead
- **Not using persist middleware** — if you need localStorage, add the middleware
- **Over-storing** — if only one component uses it, use `useState` instead
- **Mutating state directly** — always use `set(state => ({ ... }))`, or use Immer middleware
- **v4 `subscribeWithSelector` pattern in v5** — must import `subscribeWithSelector` from `zustand/middleware` explicitly
- **v4 test auto-act in v5** — wrap updates in `act()` manually; v5 removed auto-wrapping
