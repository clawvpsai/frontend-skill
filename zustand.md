# Zustand — Client State Management

> **Note:** For foundational Zustand patterns (basic store, selectors, persist middleware, Immer, slice pattern), see **`state.md`**. This file covers only content not found in `state.md`.

## When to Use Zustand

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
