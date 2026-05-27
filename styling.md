# Styling — Tailwind CSS v4 + Design Tokens

## Tailwind CSS v4 Key Changes

v4 is a rewrite with significant changes:
- **No `tailwind.config.js` by default** — uses CSS `@theme` directive instead
- **CSS-first configuration** — all tokens defined in CSS, not JS
- **New `@tailwindcss/vite` plugin** for Vite projects
- **Built-in Next.js support** — no plugin needed in Next

## Tailwind v4 with Next.js

### `globals.css`

```css
@import "tailwindcss";

@theme {
  /* Color tokens — maps to CSS variables */
  --color-primary: hsl(222.2 47.4% 11.2%);
  --color-primary-foreground: hsl(210 40% 98%);
  --color-secondary: hsl(210 40% 96.1%);
  --color-destructive: hsl(0 84.2% 60.2%);
  --color-destructive-foreground: hsl(210 40% 98%);
  
  /* Border radius */
  --radius: 0.5rem;
  
  /* Custom spacing */
  --spacing-18: 4.5rem;
}

/* Use tokens with Tailwind */
.card {
  background-color: var(--color-primary);
  border-radius: var(--radius);
}
```

## Tailwind v4.3 — New Features

v4.3 introduced first-party scrollbar styling, logical properties, zoom/tab-size utilities, and improved `@variant` support.

### Scrollbar Styling (First-Party)

```tsx
// Custom scrollbar — no need for arbitrary values or vendor prefixes
<div className="scrollbar scrollbar-thumb-rounded scrollbar-track-slate-100 dark:scrollbar-thumb-slate-700 dark:scrollbar-track-slate-800">
  {/* Scrollable content */}
</div>

// Available utilities:
scrollbar          // display: scrollbar
scrollbar-thin     // scrollbar-width: thin
scrollbar-none     // scrollbar-width: none

// Color — use any color token
scrollbar-thumb-{color}-{shade}
scrollbar-track-{color}-{shade}

// Rounded corners
scrollbar-thumb-rounded-full
scrollbar-thumb-rounded

// Width/thickness
scrollbar-p-1   // 4px
scrollbar-p-2   // 8px
scrollbar-p-3   // 12px
scrollbar-p-4   // 16px
```

### Logical Properties (Inline/Block)

CSS logical properties adapt to text direction (LTR/RTL, writing mode). These replace physical `margin-left`/`margin-right` patterns for internationalized apps:

```tsx
// Margin — logical (inline = left/right depending on direction)
ms-4   // margin-inline-start
me-4   // margin-inline-end
mx-4   // margin-inline
mx-auto  // center block

// Padding
ps-4   // padding-inline-start
pe-4   // padding-inline-end
px-4   // padding-inline

// Position
inset-inline-start   // left in LTR, right in RTL
inset-inline-end     // right in LTR, left in RTL
start-4              // left in LTR, right in RTL
end-4                // right in LTR, left in RTL

// Border
border-inline-end    // right border in LTR, left in RTL
border-inline-start  // left border in LTR, right in RTL
rounded-s           // start (inline) corner rounded
rounded-e           // end (inline) corner rounded
```

### Zoom Utilities

```tsx
// Scale elements — useful for accessible focus indicators, tooltips
zoom-0        // transform: scale(0)
zoom-50       // transform: scale(0.5)
zoom-75       // transform: scale(0.75)
zoom-90       // transform: scale(0.9)
zoom-100      // transform: scale(1)
zoom-125      // transform: scale(1.25)
zoom-150      // transform: scale(1.5)
zoom-200      // transform: scale(2)

// Combined with origin for pivot point
<div className="zoom-125 origin-center">...</div>
<div className="zoom-125 origin-top-left">...</div>
```

### Tab-Size Utilities

```tsx
// Control tab character rendering width
tab-1    // tab-size: 1
tab-2    // tab-size: 2
tab-3    // tab-size: 3
tab-4    // tab-size: 4
tab-5    // tab-size: 5
tab-6    // tab-size: 6
tab-7    // tab-size: 7
tab-8    // tab-size: 8
tab-9    // tab-size: 9
tab-10   // tab-size: 10
tab-11   // tab-size: 11
tab-12   // tab-size: 12
tab-13   // tab-size: 13
tab-14   // tab-size: 14
tab-15   // tab-size: 15
tab-16   // tab-size: 16

// Use for code blocks where tabs render inconsistently across browsers
<pre className="tab-4"><code>{code}</code></pre>
```

### `@variant` Improvements

v4.3 expanded `@variant` for custom pseudo-class and media query variants:

```css
@import "tailwindcss";

@variant hover (&:hover);
@variant focus (&:focus);
@variant dark (&:is(.dark *));

/* Use in classes */
<button className="bg-primary dark:bg-primary-dark hover:brightness-110">
  Hover me
</button>
```

## shadcn/ui Theming

### CSS Variables Pattern

shadcn/ui uses a `hsl()` pattern for theming:

```css
:root {
  --background: 0 0% 100%;
  --foreground: 222.2 84% 4.9%;
  --card: 0 0% 100%;
  --card-foreground: 222.2 84% 4.9%;
  --primary: 222.2 47.4% 11.2%;
  --primary-foreground: 210 40% 98%;
  --secondary: 210 40% 96.1%;
  --muted-foreground: 215.4 16.3% 46.9%;
  --accent: 210 40% 96.1%;
  --destructive: 0 84.2% 60.2%;
  --border: 214.3 31.8% 91.4%;
  --radius: 0.5rem;
}

.dark {
  --background: 222.2 84% 4.9%;
  --foreground: 210 40% 98%;
  --card: 222.2 84% 4.9%;
  --primary: 210 40% 98%;
  --primary-foreground: 222.2 47.4% 11.2%;
}
```

## Design Token Architecture

### Token Tiers

```
┌─────────────────────────────────────────────┐
│  Step 1: Primitive tokens (raw values)     │
│  hsl(222 47% 11%)  →  #1a365d               │
├─────────────────────────────────────────────┤
│  Step 2: Semantic tokens (contextual names) │
│  --color-primary                           │
├─────────────────────────────────────────────┤
│  Step 3: Component tokens (specific use)    │
│  --button-bg: var(--color-primary)          │
└─────────────────────────────────────────────┘
```

### Semantic Token Example

```css
/* primitives */
--color-blue-500: hsl(217 91% 60%);
--color-blue-600: hsl(221 83% 53%);

/* semantic */
--color-interactive-default: var(--color-blue-500);
--color-interactive-hover: var(--color-blue-600);

/* component */
--button-bg: var(--color-interactive-default);
--button-bg-hover: var(--color-interactive-hover);
```

## Tailwind Utility Patterns

### Responsive Design

```tsx
// Mobile-first — base styles apply to all, larger with prefixes
<div className="w-full md:w-1/2 lg:w-1/3">
  {/* Full width on mobile, half on tablet, third on desktop */}
</div>

// Common breakpoints
// sm: 640px  md: 768px  lg: 1024px  xl: 1280px  2xl: 1536px
```

### Dark Mode

```tsx
// Using Tailwind's dark: prefix
<div className="bg-white dark:bg-slate-900 text-slate-900 dark:text-white">
  Adapts to the .dark class on the html element
</div>

// Class-based dark mode (default for shadcn/ui)
<div className="dark:[&_*]:text-white">
  {/* All children dark mode */}
</div>
```

### Conditional Classes with `cn()`

```tsx
import { cn } from '@/lib/utils'

<Button className={cn(
  "base-class",
  isActive && "ring-2 ring-primary",
  isDisabled && "opacity-50 cursor-not-allowed",
  size === "sm" && "h-8 px-3 text-xs"
)}>
```

### Arbitrary Values

```tsx
// Use sparingly — usually a sign of bad design token
<div className="top-[calc(100%+8px)]">Positioned below</div>
<div className="h-[1lh]">Line height unit</div>
```

## Typography

### Custom Fonts

```tsx
// app/layout.tsx
import { Inter } from 'next/font/google'

const inter = Inter({ subsets: ['latin'], variable: '--font-inter' })

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html className={inter.variable}>
      <body>{children}</body>
    </html>
  )
}
```

```css
/* In globals.css */
body {
  font-family: var(--font-inter);
}
```

### Typography Scale

```tsx
// Never use arbitrary font sizes — use Tailwind's scale
<h1 className="text-4xl font-bold tracking-tight">       {/* 2.25rem */}
<h2 className="text-3xl font-semibold tracking-tight">  {/* 1.875rem */}
<h3 className="text-2xl font-semibold">                  {/* 1.5rem */}
<h4 className="text-xl font-medium">                     {/* 1.25rem */}
```

## Animation

### Tailwind Animation Classes

```tsx
// Transitions
<button className="transition-all duration-200 ease-in-out hover:scale-105">
  Hover me
</button>

// Keyframe animations
<div className="animate-spin">Loading...</div>
<div className="animate-pulse">Fetching...</div>
<div className="animate-bounce">Jump</div>

// Custom animations in tailwind config
@layer utilities {
  @keyframes slide-in {
    from { transform: translateX(-100%); }
    to { transform: translateX(0); }
  }
  .animate-slide-in {
    animation: slide-in 0.3s ease-out;
  }
}
```

### Framer Motion for Complex Animations

```tsx
import { motion } from 'framer-motion'

<motion.div
  initial={{ opacity: 0, y: 20 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: 0.5, ease: 'easeOut' }}
>
  {children}
</motion.div>
```

## Responsive Navigation Example

```tsx
<nav className="flex items-center justify-between px-4 py-3 border-b">
  <div className="flex items-center gap-6">
    <Logo />
    {/* Desktop nav */}
    <div className="hidden md:flex gap-4">
      <NavLink href="/about">About</NavLink>
      <NavLink href="/pricing">Pricing</NavLink>
      <NavLink href="/docs">Docs</NavLink>
    </div>
  </div>
  {/* Mobile menu */}
  <div className="md:hidden">
    <MobileMenu items={navItems} />
  </div>
</nav>
```

## Common Mistakes

- **Arbitrary pixel values** — `className="mt-[17px]"` → use a design token or Tailwind's scale
- **Deep nesting** — if you need `> div > div > div`, refactor the component
- **Repeated styles** — extract to a component or use `@apply` sparingly
- **Missing `dark:` prefix** — always test dark mode, not just light
- **Inline styles mixed with Tailwind** — pick one and stick to it
- **Overly specific selectors** — Tailwind's cascade respects specificity; don't fight it
