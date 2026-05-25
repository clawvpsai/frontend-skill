# Zustand — Client State Management

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

## Basic Store

```tsx
// stores/ui-store.ts
import { create } from 'zustand'

interface UIStore {
  theme: 'light' | 'dark' | 'system'
  sidebarOpen: boolean
  setTheme: (theme: 'light' | 'dark' | 'system') => void
  toggleSidebar: () => void
}

export const useUIStore = create<UIStore>((set) => ({
  theme: 'system',
  sidebarOpen: false,
  
  setTheme: (theme) => set({ theme }),
  
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
}))
```

### Usage

```tsx
// In a component
import { useUIStore } from '@/stores/ui-store'

function ThemeToggle() {
  // Select only what you need — prevents re-renders
  const theme = useUIStore(s => s.theme)
  const setTheme = useUIStore(s => s.setTheme)
  
  return (
    <select value={theme} onChange={(e) => setTheme(e.target.value as any)}>
      <option value="light">Light</option>
      <option value="dark">Dark</option>
      <option value="system">System</option>
    </select>
  )
}
```

## Persisting State (localStorage)

```tsx
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface CartStore {
  items: CartItem[]
  addItem: (item: CartItem) => void
  removeItem: (id: string) => void
  clearCart: () => void
}

export const useCartStore = create<CartStore>()(
  persist(
    (set) => ({
      items: [],
      
      addItem: (item) => set((state) => {
        const existing = state.items.find(i => i.id === item.id)
        if (existing) {
          return {
            items: state.items.map(i =>
              i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i
            ),
          }
        }
        return { items: [...state.items, { ...item, quantity: 1 }] }
      }),
      
      removeItem: (id) => set((state) => ({
        items: state.items.filter(i => i.id !== id),
      })),
      
      clearCart: () => set({ items: [] }),
    }),
    {
      name: 'cart-storage',
      // Optional: only persist certain fields
      partialize: (state) => ({ items: state.items }),
    }
  )
)
```

## Immer Middleware (Immutable Updates Made Easy)

```tsx
import { create } from 'zustand'
import { immer } from 'zustand/middleware/immer'

interface EditorStore {
  content: string
  history: string[]
  redoStack: string[]
  setContent: (content: string) => void
  undo: () => void
  redo: () => void
}

export const useEditorStore = create<EditorStore>()(
  immer((set, get) => ({
    content: '',
    history: [],
    redoStack: [],
    
    setContent: (content) => set((draft) => {
      draft.history.push(get().content)
      draft.redoStack = []
      draft.content = content
    }),
    
    undo: () => set((draft) => {
      const prev = draft.history.pop()
      if (prev !== undefined) {
        draft.redoStack.push(draft.content)
        draft.content = prev
      }
    }),
    
    redo: () => set((draft) => {
      const next = draft.redoStack.pop()
      if (next !== undefined) {
        draft.history.push(draft.content)
        draft.content = next
      }
    }),
  }))
)
```

## Combining Slices

```tsx
import { create } from 'zustand'

interface CartSlice {
  items: CartItem[]
  addItem: (item: CartItem) => void
  removeItem: (id: string) => void
}

interface UISlice {
  sidebarOpen: boolean
  toggleSidebar: () => void
}

type Store = CartSlice & UISlice

export const useStore = create<Store>((set) => ({
  // Cart
  items: [],
  addItem: (item) => set((state) => ({
    items: [...state.items, item],
  })),
  removeItem: (id) => set((state) => ({
    items: state.items.filter(i => i.id !== id),
  })),
  
  // UI
  sidebarOpen: false,
  toggleSidebar: () => set((state) => ({ 
    sidebarOpen: !state.sidebarOpen 
  })),
}))
```

## Derived State (Selectors)

```tsx
// Compute derived values outside the store or with useMemo
const useCartStore = create<CartStore>()((set) => ({ ... }))

// Usage in component — select multiple slices
import { useMemo } from 'react'

function CartSummary() {
  const items = useCartStore(s => s.items)
  
  const total = useMemo(() => 
    items.reduce((sum, item) => sum + item.price * item.quantity, 0),
    [items]
  )
  
  const itemCount = useMemo(() =>
    items.reduce((sum, item) => sum + item.quantity, 0),
    [items]
  )
  
  return (
    <div>
      <p>Items: {itemCount}</p>
      <p>Total: ${total.toFixed(2)}</p>
    </div>
  )
}
```

## Async Actions

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

## Resetting State

```tsx
interface Store {
  count: number
  reset: () => void
}

export const useStore = create<Store>((set) => ({
  count: 0,
  reset: () => set({ count: 0 }),
}))
```

## Common Mistakes

- **Selecting entire store** — causes re-renders when any field changes
  - ❌ `const store = useStore()` 
  - ✅ `const count = useStore(s => s.count)`
- **Using Zustand for server data** — use React Query instead
- **Not using persist middleware** — if you need localStorage, add the middleware
- **Over-storing** — if only one component uses it, use `useState` instead
- **Mutating state directly** — always use `set(state => ({ ... }))`, or use Immer middleware
