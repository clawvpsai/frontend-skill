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

## Tailwind v4.2/v4.3 — New Features

v4.2 and v4.3 shipped quietly with significant new features: four new color palettes, a dedicated webpack plugin with 2x+ build speed improvements, block-size container queries, and first-party scrollbar/zoom/tab-size utilities.

### New Color Palettes (v4.2)

Four new neutral-adjacent palettes fill a gap between grays and other colors:

```tsx
// mauve — warm purple-gray
bg-mauve-100  text-mauve-600  border-mauve-300

// olive — desaturated green-gray
bg-olive-100  text-olive-600  border-olive-300

// mist — cool blue-gray
bg-mist-100  text-mist-600  border-mist-300

// taupe — warm brown-gray
bg-taupe-100  text-taupe-600  border-taupe-300
```

Each palette has shades from 50 through 900 (same scale as slate/zinc). Use these instead of gray when you want warmth or coolness without going full color.

### Webpack Plugin (v4.2)

A dedicated Tailwind webpack plugin offers 2x+ faster build speeds for large projects:

```ts
// vite.config.ts — Vite (Next.js uses built-in, no plugin needed)
import tailwindcss from '@tailwindcss/vite'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [tailwindcss()],
})
```

**Note:** Next.js already uses its own built-in Tailwind integration — no additional plugin needed. The webpack plugin is primarily for non-Next.js webpack projects.

### `@container-size` — Block-Size Container Queries (v4.3)

v4.3 adds block-size (`cqb`/`cqh`) container queries via `@container-size`. Previously `@container` only supported inline-size queries. Now you can style based on the container's block dimension (height in horizontal writing mode):

```tsx
// Define a container that can be queried
<div className="container-type-inline-size">
  <div className="cqb-4">    {/* Shows when container block-size >= 4 (1rem) */}
    <Sidebar />
  </div>
</div>

// Available utilities:
container-type-inline-size   // enables @container queries on inline (width) axis
container-type-block-size     // enables @container queries on block (height) axis
container-type-size          // enables both axes

// Container query variants — inline-size (existing)
@container              // shorthand for min-width: inline-size
cqb-{size}              // min-block-size >= size
cqb-{size}-{variant}    // responsive variants (sm, md, lg, etc.)

// Container query variants — block-size (NEW in v4.3)
cqh-{size}              // min-block-size >= size (queries container HEIGHT)
cqh-{size}-{variant}    // responsive variants

// Real-world example: content cards that adapt to container height
<div className="container-type-block-size">
  <Card className="cqh-48:h-48 cqh-64:h-64 cqh-96:h-96" />
</div>
```

**Why `@container-size` over media queries?** Container queries respond to the parent container's size, not the viewport — better for reusable component libraries where the same component appears in different layout contexts.

### Font-Features-* Utility (v4.2)

Control OpenType font features directly in Tailwind:

```tsx
// OpenType features — use for advanced typography
font-features-tnum    // tabular-nums — numbers align in columns
font-features-liga    // ligatures — common ligatures like fi, fl
font-features-kern    // kerning — letter-spacing optimization
font-features-smcp    // small-caps
font-features-onum    // old-style-nums — variable-width numbers like 1, 2

// For code: use tabular-nums so digits align in columns
<code className="font-features-tnum font-mono">0123456789</code>

// For rich text: old-style nums look more natural in body copy
<p className="font-features-onum">The price is $199.99</p>
```

### Scrollbar Styling (v4.3 — First-Party)

Custom scrollbars without vendor prefixes or arbitrary values:

```tsx
// Custom scrollbar
<div className="scrollbar scrollbar-thumb-rounded scrollbar-track-slate-100 dark:scrollbar-thumb-slate-700 dark:scrollbar-track-slate-800">
  {/* Scrollable content */}
</div>

// Available utilities:
scrollbar              // display: scrollbar
scrollbar-thin         // scrollbar-width: thin
scrollbar-none         // scrollbar-width: none

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

### Zoom Utilities (v4.3)

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

### Tab-Size Utilities (v4.3)

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

### `inert` HTML Attribute (v4)

The HTML `inert` attribute makes an element and its descendants inert — they cannot be focused, selected, or interacted with. Unlike `disabled` (which only works on form elements), `inert` works on any element. This is the modern replacement for manually disabling groups of elements.

**Browser support:** Chrome 102+, Safari 16.4+, Firefox 121+. Supported in all modern browsers.

```tsx
// Native HTML — inert makes the whole subtree non-interactive
<div inert>
  <button>Can't click this</button>
  <input placeholder="Can't type here" />
  <a href="/">Can't navigate here</a>
</div>

// Toggle inert programmatically
function Modal({ isOpen, children }: { isOpen: boolean; children: React.ReactNode }) {
  return (
    <div className={isOpen ? undefined : 'inert'}>
      {children}
    </div>
  )
}

// Tailwind v4 — use inert: variant for conditional styling
// inert:applies styles when the element has inert attribute
<div className="inert:opacity-50 inert:pointer-events-none">
  <button>Visually dimmed and non-interactive</button>
</div>
```

**Real-world pattern — accessible dialog backdrop:**

```tsx
'use client'

import { useState } from 'react'

function DialogDemo() {
  const [isOpen, setIsOpen] = useState(false)

  return (
    <>
      <button onClick={() => setIsOpen(true)}>Open Dialog</button>

      {isOpen && (
        <div className="fixed inset-0 z-50 flex items-center justify-center">
          {/* Backdrop — clicking it closes the dialog */}
          <div
            className="absolute inset-0 bg-black/50"
            onClick={() => setIsOpen(false)}
            aria-hidden="true"
          />

          {/* Dialog content — backdrop inert so focus stays inside */}
          <div className="relative bg-white rounded-lg p-6">
            <h2>Dialog Title</h2>
            <p>You can only interact with this dialog.</p>
            <button onClick={() => setIsOpen(false)}>Close</button>
          </div>
        </div>
      )}
    </>
  )
}
```

**`inert` vs alternatives:**

| Pattern | Scope | Use When |
|---|---|---|
| `disabled` | Form elements only (`<button>`, `<input>`, etc.) | Disabling individual form fields |
| `inert` | Any element + all descendants | Dismissing/modaling a whole subtree, "putting away" a section |
| `aria-hidden` | Visual only — still focusable | Hiding decorative elements |
| `pointer-events-none` | Disables mouse/touch, still focusable | Temporarily disabling interactions without removing from DOM |

**Sources:**
- [MDN: HTML inert attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/inert)
- [Can I use: inert](https://caniuse.com/mdn-html_global_attribute_inert)

### `@variant` Improvements (v4.3)

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


## Tailwind v4.3.1 — Patch (June 12, 2026)

A small but useful patch release. v4.3.1 ships one new CLI flag, an `@apply` upgrade, and ~20 quality-of-life fixes — no breaking changes.

### New: `--silent` Flag for `@tailwindcss/cli`

Suppress all output from the CLI (useful for CI / Docker builds where Tailwind output is noise):

```bash
# Default: shows "Rebuilding..." messages
npx @tailwindcss/cli -i input.css -o output.css --watch

# With --silent: no output
npx @tailwindcss/cli -i input.css -o output.css --watch --silent
```

### `@apply` Now Works With CSS Mixins

You can now `@apply` CSS custom properties / mixins — previously the parser would reject them:

```css
/* Define a mixin */
@theme {
  --shadow-elevated: 0 4px 16px rgb(0 0 0 / 0.1);
  --shadow-elevated-lg: 0 8px 32px rgb(0 0 0 / 0.15);
}

/* Apply it from a utility (NEW in v4.3.1) */
@utility shadow-elevated-* {
  box-shadow: --value(--shadow-elevated);
}

/* Or via @apply in a custom class */
.card-elevated {
  @apply bg-white p-4;
  @apply shadow-[var(--shadow-elevated)];  /* now works */
}
```

### Cleaner Spacing Output (m-0 / m-1)

v4.3.1 generates cleaner CSS for the spacing scale. `m-0` used to generate `margin: calc(var(--spacing) * 0)` — now it generates `margin: 0`. Same for `m-1` → `margin: var(--spacing)` (no `* 1`). Smaller CSS bundle, easier to read in DevTools:

```css
/* Before v4.3.1 */
.m-0  { margin: calc(var(--spacing) * 0); }
.m-1  { margin: calc(var(--spacing) * 1); }
.left-0 { left: calc(var(--spacing) * 0); }

/* v4.3.1 */
.m-0  { margin: 0; }
.m-1  { margin: var(--spacing); }
.left-0 { left: 0; }
```

### Other Notable v4.3.1 Fixes

- **Sourcemap warnings** gone for `@tailwindcss/vite` (#20103)
- **`@tailwindcss/webpack`** now installable in Rspack without forcing `webpack` peer dependency (#20027)
- **`@source` globs preserve symlinks** and re-include files excluded by `@source not` (#20203)
- **`@variant` usable inside `addBase`** (#19480)
- **Calc canonicalization** no longer folds divisions that need high precision (e.g. `w-[calc(100%/3.5)]` stays readable) (#20221)
- **`drop-shadow-*` colors** now work with custom shadow values containing `calc()` (#20080)
- **`insetshadow-none` transitions** to other inset shadows work correctly (#20208)
- **Twig `addClass`/`removeClass`** calls now extract class candidates (#20198)
- **ESM type declarations** served to ESM importers of `@tailwindcss/postcss` (#20228)

**Upgrade:** `npm install tailwindcss@latest` (or just `tailwindcss@^4.3.1`). Patch release — no code changes required.

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

**Sources:**
- [Tailwind CSS v4.3.1 release notes (GitHub)](https://github.com/tailwindlabs/tailwindcss/releases/tag/v4.3.1)
- [Tailwind Weekly: v4.3.1 highlights](https://tailwindweekly.com/issue-218/)
- [Tailwind CSS v4.3 release notes](https://tailwindcss.com/blog/tailwindcss-v4-3)
- [Tailwind CSS v4.2/v4.3 new features](https://app.daily.dev/posts/what-s-new-in-tailwind-css-v4-2-and-v4-3-oybkeyde7)
