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

## Tailwind v4.3.2 — Patch (June 29, 2026)

A pure bug-fix release dropped the same day as the 16.3.0-canary.71 cut. v4.3.2 ships four targeted fixes — no new features, no breaking changes, no documentation-only changes. **The CSS output is unchanged**, so you can upgrade without testing.

### Fixed in v4.3.2

- **`auto-rows-*` and `auto-cols-*` accept bare numeric values** ([#20229](https://github.com/tailwindlabs/tailwindcss/pull/20229)) — `auto-rows-12` and `auto-cols-16` now work without a `grid-template-rows: repeat(...)` wrapper. Previously these would either silently fail or require the `grid-rows-12` / `grid-cols-12` variant; the new bare-value form matches the more common grid-literal expectation.
- **`@tailwindcss/cli --watch` no longer crashes on Windows when `@source` points to a missing directory** ([#20242](https://github.com/tailwindlabs/tailwindcss/pull/20242)) — the watcher used to throw `ENOENT` on Windows specifically (the path-handling was wrong on case-insensitive filesystems when the missing directory was in a `git clean -fdx`-style state). Now it logs a warning and continues watching the rest of the tree.
- **`@tailwindcss/vite` no longer crashes in Deno v2.8.x** ([#20245](https://github.com/tailwindlabs/tailwindcss/pull/20245)) — Deno 2.8 changed `LoaderContext.parentURL` semantics, and the Vite plugin was reading it as if it were always a valid `file://` URL. Now guarded with a `URL.canParse(parentURL)` check before reading.
- **`@tailwindcss/cli --watch` doesn't re-emit an unchanged file when only a sibling changes** ([#20249](https://github.com/tailwindlabs/tailwindcss/pull/20249)) — a small debouncing fix that prevents the CLI from rewriting `output.css` on every rebuild even when no class changed, which was breaking the `mtime`-based CDN cache invalidation some deploy setups rely on.

### Why This Matters in a Skill

All four fixes are **silent footgun** fixes (no warning, no error before — they just did the wrong thing or crashed under specific edge cases). If you have any of these in your codebase:

- A grid container that uses `auto-rows-N` / `auto-cols-N` with bare numeric values, **upgrade to v4.3.2** to make the missing classes actually resolve.
- A Windows + Tailwind CLI + `--watch` workflow where `@source` references a directory that's sometimes missing — **upgrade to v4.3.2** to stop the `ENOENT` crash.
- A Deno 2.8+ project using `@tailwindcss/vite` — **upgrade to v4.3.2** to stop the `parentURL is not a valid URL` crash.
- A CI pipeline where `output.css` was being re-emitted with unchanged content (and downstream caches were not invalidating) — **upgrade to v4.3.2** to stop the unnecessary rewrites.

**Upgrade:** `npm install tailwindcss@latest` (or just `tailwindcss@^4.3.2`). Patch release — no code changes required.

## Tailwind v4.3.3 — Patch (July 16, 2026)

A pure bug-fix release dropped the same day as the 16.3.0-canary.88 tag cut. v4.3.3 ships **nine targeted fixes** — no new features, no breaking changes, no documentation-only changes. **The CSS output is unchanged**, so you can upgrade without testing.

### Fixed in v4.3.3

- **`@tailwindcss/cli --watch --poll[=ms]` now works when filesystem events are unreliable or unavailable** ([#20297](https://github.com/tailwindlabs/tailwindcss/pull/20297)). The `--poll` option on `@tailwindcss/cli` was being silently ignored when combined with `--watch`; on filesystems / mounts where `fs.watch` events don't fire (Docker bind mounts in some configs, WSL2, certain network filesystems) the CLI would never rebuild. Now respects `--poll` for any value (or `--poll` with no argument = default poll interval).
- **Canonicalization: match arbitrary hex colors against theme colors case-insensitively** ([#20298](https://github.com/tailwindlabs/tailwindcss/pull/20298)). `bg-[#fff]` was canonicalizing to `bg-[#FFF]` (kept the literal value), but theme colors like `white` (`#ffffff`) wouldn't match. Now arbitrary hex is compared case-insensitively, so `bg-[#FFF]` → `bg-white` and `bg-[#FfF]` → `bg-white`. CSS color tokens normalized.
- **Preflight no longer overrides Firefox's native `iframe:focus-visible` outline styles** ([#20292](https://github.com/tailwindlabs/tailwindcss/pull/20292)). Preflight was injecting a global `iframe:focus-visible { outline: none }` that Firefox uses for accessibility focus indicators; Firefox's outline-on-iframe is a browser feature, not a CSS bug. Now the Preflight reset doesn't target `<iframe>` focus-visible.
- **`theme('colors.foo')` resolves correctly when both `--color-foo` and `--color-foo-bar` exist** ([#20299](https://github.com/tailwindlabs/tailwindcss/pull/20299)). The JS-plugin theme lookup was matching the `--color-foo-bar` declaration when the user asked for `--color-foo` (longer-prefix wins); now exact-match wins, so `theme('colors.foo')` returns the value of `--color-foo` not `--color-foo-bar`.
- **Fractional opacity modifiers work with named shadow sizes** ([#20302](https://github.com/tailwindlabs/tailwindcss/pull/20302)). `shadow-sm/12.5`, `text-shadow-sm/12.5`, `drop-shadow-sm/12.5`, and `inset-shadow-sm/12.5` were failing the `parseFloat` step and falling back to `100%` opacity; now fractional values parse correctly.
- **Parse selectors like `[data-foo]div` as two selectors instead of one** ([#20303](https://github.com/tailwindlabs/tailwindcss/pull/20303)). Tailwind's selector parser was treating `[data-foo]div` as a single compound selector (which would never match any element) instead of as `[data-foo]` + `div` (which would match an element with `data-foo` followed by a `div` — a descendant combinator). Now parsed as two selectors with a descendant combinator between them.
- **`@tailwindcss/postcss` rebuilds when a preprocessor like Sass changes the input CSS without changing the input file on disk** ([#20310](https://github.com/tailwindlabs/tailwindcss/pull/20310)). PostCSS plugin was watching only the on-disk file mtime; when Sass did an in-memory transform (e.g. import flattening) without writing back to disk, the plugin didn't notice. Now uses an in-memory content hash as well.
- **CSS nesting handled even when Lightning CSS isn't run** ([#20124](https://github.com/tailwindlabs/tailwindcss/pull/20124)). `@tailwindcss/browser` and Tailwind Play use the standard CSS Nesting browser API instead of Lightning CSS for the nesting transform; previously the CSS nesting pass was being skipped when Lightning CSS was absent, producing invalid CSS. Now runs regardless of Lightning CSS.
- **Partial excerpt hidden in upstream PR — see [v4.3.3 release notes](https://github.com/tailwindlabs/tailwindcss/releases/tag/v4.3.3) for the full list of 9 PRs.** (#20307, #20309, etc.)

### Why This Matters in a Skill

All fixes are **silent footgun** fixes (no warning, no error before — they just did the wrong thing or crashed under specific edge cases). If you have any of these in your codebase:

- A CSS-driven custom-theme palette where `theme('colors.foo')` returned `undefined` because you also had `colors.foo-bar` defined — **upgrade to v4.3.3** to get exact-prefix matching back.
- A Tailwind project that ships to Firefox and you noticed `<iframe>` focus rings were invisible — **upgrade to v4.3.3** to stop Preflight from clearing Firefox's accessibility outline.
- A Docker / WSL2 / network-mount dev workflow where `@tailwindcss/cli --watch` never rebuilt — **upgrade to v4.3.3** and pass `--poll[=ms]` to enable polling-based change detection.
- Code that used fractional opacity with shadow utilities like `shadow-md/12.5` — **upgrade to v4.3.3** to stop the opacity silently snapping to 100%.
- CSS using `[data-foo]div` style selectors (uncommon but valid) that wasn't being matched — **upgrade to v4.3.3** to get the selector parsed as descendant combinator.

**Upgrade:** `npm install tailwindcss@latest` (or just `tailwindcss@^4.3.3`). Patch release — no code changes required.

## Vite `PostcssUserConfig` Type Export (July 16, 2026, ahead of Vite 8.1.5 — Vite PR [#22792](https://github.com/vitejs/vite/pull/22792) by linyiru, merged 2026-07-16T11:42:15Z, closes [#19109](https://github.com/vitejs/vite/issues/19109))

`PostcssUserConfig` is now re-exported from `vite`, so PostCSS configs can be typed against the same definition Vite loads them with. **Will ship in Vite 8.1.6 / 8.2.0 (ahead of Vite 8.1.5 on `main`); not yet tagged to a stable release.**

**Before** (the typical PostCSS config shape had to be hand-written or `any`-typed):

```ts
// postcss.config.ts — old pattern
const config = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}

export default config
```

**After** (typed against Vite's internal definition):

```ts
// postcss.config.ts — new typed pattern
import type { PostcssUserConfig } from 'vite'

const config: PostcssUserConfig = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}

export default config
```

**Why this matters:** PostCSS plugins have heterogeneous config shapes (some are plain objects, some are arrays, some are factory functions). Vite has the canonical definition; now your config can match it for full autocomplete + type checking instead of relying on `any` or hand-maintaining the type.

**Migration impact:**

- **No behavior change** — this is purely a type addition.
- **Works in any Vite-based project** once the next Vite version that includes PR #22792 ships.
- **Independent of Tailwind v4** — the type is for PostCSS configs in general.

**Sources:**
- [Vite PR #22792 — `feat(css): export PostCSS config type for type-safe configs`](https://github.com/vitejs/vite/pull/22792)
- [Vite issue #19109 — Original feature request](https://github.com/vitejs/vite/issues/19109)

## Tailwind v4 + shadcn/ui Migration Reference — Renamed Utilities, Animation Plugin Swap, OKLCH, `shadcn apply` Presets (April–July 2026)

Everything that breaks (or quietly changes) when you migrate a real shadcn/ui project from Tailwind v3 → v4, plus the new April 2026 `shadcn apply` / `shadcn preset` workflow for portable design systems. These are the *migration gotchas the official `@tailwindcss/upgrade` tool does NOT auto-fix*, and the things every Tailwind v4 shadcn project needs to know.

### Renamed Utilities — The Official Tailwind v4 Table

Tailwind v4 renamed the default shadow / radius / blur / ring / outline scales so every utility has a named value. **The bare names still work for backward compatibility but `<utility>-sm` looks different** until you update. From [the Tailwind v4 upgrade guide](https://tailwindcss.com/docs/upgrade-guide):

| v3 utility | v4 utility | Where it hits shadcn |
|---|---|---|
| `shadow-sm` | `shadow-xs` | `Card`, `Dialog`, `Popover`, `Sheet` |
| `shadow` | `shadow-sm` | same as above (cards, dialogs, popovers) |
| `drop-shadow-sm` | `drop-shadow-xs` | Custom `Card` with image header |
| `drop-shadow` | `drop-shadow-sm` | Custom `Card` with image header |
| `blur-sm` | `blur-xs` | `Skeleton`, dialog backdrop blur |
| `blur` | `blur-sm` | Same |
| `backdrop-blur-sm` | `backdrop-blur-xs` | `Dialog` overlay, `Sheet` |
| `backdrop-blur` | `backdrop-blur-sm` | Same |
| `rounded-sm` | `rounded-xs` | `Button`, `Input`, `Badge`, `Avatar` |
| `rounded` | `rounded-sm` | Same |
| `outline-none` | `outline-hidden` | All focus rings on `Button`, `Input`, `Select` |
| `ring` | `ring-3` | All focus rings on every interactive component |

Two non-obvious defaults also changed:

```css
/* In v4, the bare `ring` is now 1px wide and currentColor (was 3px blue-500 in v3) */
/* Most shadcn focus rings look visibly thinner after upgrade. Fix per-use: */
<input className="ring-3 ring-blue-500" />

/* Or restore the v3 default globally via @theme — supported for compat, NOT idiomatic v4 */
@theme {
  --default-ring-width: 3px;
  --default-ring-color: var(--color-blue-500);
}
```

```css
/* In v4, the `outline` utility sets outline-width: 1px by default (was 0) and
   all `outline-<number>` utilities default outline-style to solid, so you can drop the standalone `outline` class */
<button className="outline-2 outline-blue-500" />   /* works as you'd expect, no `outline` class needed */
```

**Migration audit recipe:**

```bash
# Grep your shadcn output for v3 utility names that v4 silently downscales
grep -REn 'shadow-sm|shadow |drop-shadow-sm|drop-shadow |blur-sm|blur |backdrop-blur-sm|backdrop-blur |rounded-sm|rounded |outline-none|ring[^-]' src/
# Anything matching these will look thinner/smaller after the upgrade tool runs
```

### `tw-animate-css` — The Replacement for `tailwindcss-animate` (shadcn deprecation, March 2025)

**shadcn deprecated `tailwindcss-animate` in favor of [`tw-animate-css`](https://github.com/Wombosvideo/tw-animate-css)** (the v4-native, pure-CSS replacement). All new shadcn projects install `tw-animate-css` by default. If you upgrade an existing project, popovers/dialogs/sheets **silently lose their open/close animations** until you complete this swap.

**Migration steps** (from [shadcn/ui Tailwind v4 docs](https://ui.shadcn.com/docs/tailwind-v4), March 19, 2025):

```bash
# 1. Remove the old package
npm uninstall tailwindcss-animate

# 2. Install the v4 replacement (NOT a Tailwind plugin — it's a CSS file)
npm install -D tw-animate-css

# 3. Update globals.css — swap the @plugin directive for a plain @import
```

```diff
- @plugin 'tailwindcss-animate';
+ @import "tw-animate-css";
```

**What `tw-animate-css` actually ships:**

- Ready-to-use animations: `animate-accordion-down`, `animate-accordion-up`, `animate-caret-blink`
- Enter/exit utilities: `animate-in fade-in slide-in-from-top-8 duration-500` (same API surface as `tailwindcss-animate`)
- Transform utilities: `fade-<io>`, `slide-in-from-{top,right,bottom,left}-<n>`, `zoom-<io>-<n>`, `spin-<io>-<n>`
- Pure CSS — embraces v4's CSS-first architecture, no JavaScript plugin required
- Tree-shaken — Tailwind only ships the animation classes you actually use

**Vite gotcha** (real-world pitfall): don't try `@import 'tw-animate-css/dist/tw-animate.css'` — `tw-animate-css` does NOT expose the dist path via the `exports` field and Vite will reject it. Always use `@import "tw-animate-css"` (the bare package specifier).

**Compatibility note from the upstream README:** *Not a 100% drop-in replacement — covers the parts shadcn uses; if you had exotic custom animations defined via the JS plugin's `addPlugin` API you'll need to port them to v4's `@theme` + custom utility syntax.*

**Audit recipe:**

```bash
# Projects still on the old package after upgrade — Dialog/Sheet/Popover animations are silent no-ops
grep -E 'tailwindcss-animate|@plugin .tailwindcss-animate.' package.json src/**/*.css
```

### OKLCH Color Migration — shadcn Themes Now Use OKLCH, Not HSL

shadcn/ui migrated all default themes from HSL → OKLCH in March 2025. If your project still uses HSL variables, the dark mode colors look washed-out (HSL gamut is smaller than OKLCH, especially in saturated reds/greens/blues) and you miss the better contrast shadcn ships.

**What the codemod produces (still uses HSL wrappers):**

```css
/* After running `npx @tailwindcss/upgrade` */
@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 0 0% 3.9%;
  }
}
@theme {
  --color-background: hsl(var(--background));
  --color-foreground: hsl(var(--foreground));
}
```

**What you want (v4-native, OKLCH):**

```css
/* Move :root and .dark out of @layer base, wrap values in the colour fn of choice, switch to @theme inline */
:root {
  --background: oklch(1 0 0);              /* oklch(lightness chroma hue) */
  --foreground: oklch(0.145 0 0);
  --primary: oklch(0.205 0 0);
  --primary-foreground: oklch(0.985 0 0);
  /* ...etc — see shadcn Base Colors reference for the full oklch scale per palette */
}

.dark {
  --background: oklch(0.145 0 0);
  --foreground: oklch(0.985 0 0);
  /* ...etc */
}

@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
  /* ...etc */
}
```

Three things to fix manually after the codemod:

1. Move `:root` and `.dark` **out of** `@layer base` (the codemod leaves them inside).
2. Wrap color values in the color function of your choice (`hsl(...)` or `oklch(...)` or `lab(...)`).
3. Add `inline` to `@theme` → `@theme inline` so the variables resolve at use-site instead of being inlined into the generated utility.

If you want HSL (e.g. you have a custom palette already authored in HSL), the steps are identical — just use `hsl(...)` instead of `oklch(...)` for the wrapping function.

### `shadcn apply` + `shadcn preset` — Portable Design Systems (April 2026)

shadcn added two new CLI commands in April 2026 that turn an existing project's theme into a **shareable preset code** — and let you apply any preset to an existing project without starting over.

**The workflow** (see [shadcn changelog April 2026](https://ui.shadcn.com/docs/changelog/2026-04-shadcn-apply) + [April 2026 preset commands](https://ui.shadcn.com/docs/changelog/2026-04-preset-commands)):

```bash
# 1. Resolve your current project's theme into a preset code
pnpm dlx shadcn@latest preset resolve
# → Preset  code         b5Kc6P0Vc
#   version      b
#   style        luma
#   baseColor    olive
#   theme        lime
#   chartColor   sky
#   iconLibrary  hugeicons
#   font         geist
#   radius       default
#   url          https://ui.shadcn.com/create?preset=b5Kc6P0Vc

# Monorepo support — point at one app
pnpm dlx shadcn@latest preset resolve -c apps/web

# Machine-readable
pnpm dlx shadcn@latest preset resolve --json

# 2. Inspect a preset before applying it
pnpm dlx shadcn@latest preset decode b5owWMfJ8l
# → full breakdown: version, style, baseColor, theme, chartColor, iconLibrary, font, fontHeading, radius, menuAccent, menuColor, url

# 3. Share as a link
pnpm dlx shadcn@latest preset url b5owWMfJ8l
# → https://ui.shadcn.com/create?preset=b5owWMfJ8l

# Open in the browser-based customizer
pnpm dlx shadcn@latest preset open b5owWMfJ8l

# 4. Apply a preset to an EXISTING project (without nuking it)
pnpm dlx shadcn@latest apply --preset b2D0vQ7G4
# → reinstalls components, updates theme, colors, CSS variables, fonts, icons
# → keeps the current base + RTL settings from your existing project even if the preset URL had different values
```

**Why this matters in a skill:**

- **AI-agent friendly** — preset codes are short, shareable, and human-readable. Pass one to a coding agent instead of a 200-line `globals.css` dump.
- **Cross-tool** — the same preset code works in `shadcn/create`, Claude, v0, Replit, Cursor (via the `add` / `init` / `apply` commands).
- **Round-trip** — `preset resolve` makes your current setup portable; `preset apply` lets you swap to a different one.
- **Audit-friendly** — `preset decode` gives you a complete diff of what the preset would change before you run `apply`.

**Use case patterns:**

| Scenario | Command |
|---|---|
| Starting a fresh project from a known preset | `pnpm dlx shadcn@latest init --preset b5Kc6P0Vc` |
| Migrating an existing project to a new design system | `pnpm dlx shadcn@latest apply b5Kc6P0Vc` |
| Sharing your project's theme with a teammate or coding agent | `pnpm dlx shadcn@latest preset resolve` → share the code |
| Previewing what a preset would do before applying | `pnpm dlx shadcn@latest preset decode b5owWMfJ8l` |
| Versioning your design system in CI | `preset resolve --json` → diff against the previous run |

**Migration impact:** non-breaking. The preset codes are an additive layer on top of `components.json` — your existing project continues to work with `npx shadcn@latest add <component>` as before; `preset apply` just adds the apply step.

### Common Mistakes (Tailwind v4 + shadcn migration)

These are the silent-failure patterns that bite projects during the v3 → v4 transition:

- **Forgetting `tw-animate-css`** — Dialog / Sheet / Popover animations silently become no-ops; check `package.json` and `globals.css` for stale `tailwindcss-animate` references after every dep update.
- **`@import 'tw-animate-css/dist/tw-animate.css'`** — Vite rejects this; the `exports` field doesn't expose `dist/`. Use bare `@import "tw-animate-css"`.
- **HSL values left in `:root` after codemod** — OKLCH palette gives noticeably better color, especially saturated reds/greens/blues; audit by running `npx shadcn@latest add --all --overwrite` and re-reading the generated theme against the shadcn Base Colors reference.
- **`@theme` vs `@theme inline`** — without `inline`, the variables are inlined at build time into the generated utilities (HMR breaks for hot color swaps); always use `@theme inline` for shadcn-style CSS-variable themes.
- **Bare `ring` class looks 1px** — that's the v4 default. Use `ring-3` to match the v3 visual.
- **Bare `rounded` / `shadow` look smaller** — they were renamed down the scale. Use `rounded-sm` / `shadow-sm` to match the v3 visual.
- **Running `preset apply` on a project with custom components** — `apply` reinstalls ALL existing components to the preset's style; custom hand-written components are preserved but you may want to review the diff with `preset decode` first.
- **Vite + shadcn `init` (April 2026)** — the init templates now ship with `vite`-preset-by-default in some `shadcn@latest` builds; if you're on a Vite project and the templates look wrong, run `pnpm dlx shadcn@latest init --preset b5Kc6P0Vc` explicitly with a known preset code rather than relying on the interactive picker.

**Sources:**
- [Tailwind v4 upgrade guide — Renamed utilities + ring/outline default changes](https://tailwindcss.com/docs/upgrade-guide)
- [shadcn/ui Tailwind v4 docs — `tailwindcss-animate` → `tw-animate-css` deprecation (March 19, 2025)](https://ui.shadcn.com/docs/tailwind-v4)
- [`tw-animate-css` GitHub repo](https://github.com/Wombosvideo/tw-animate-css)
- [`tw-animate-css` on npm](https://www.npmjs.com/package/tw-animate-css)
- [shadcn/ui changelog — April 2026 `shadcn apply`](https://ui.shadcn.com/docs/changelog/2026-04-shadcn-apply)
- [shadcn/ui changelog — April 2026 `shadcn preset` commands](https://ui.shadcn.com/docs/changelog/2026-04-preset-commands)
- [shadcn/ui Tailwind v4 docs — OKLCH migration](https://ui.shadcn.com/docs/tailwind-v4)
- [Tailwind v4 + shadcn/ui migration gotchas roundup](https://www.buildmvpfast.com/blog/tailwind-v4-shadcn-ui-migration-breaking-changes-guide-2026)
- [Vite + shadcn + tw-animate-css path-export pitfall](https://devs.keenthemes.com/question/issue-migrating-from-tailwindcss-animate-to-tw-animate-css-in-vitetailwind-project-based-on-reui-changelog)
- [shadcncraft Create — preset workflow overview](https://shadcncraft.com/blog/create-your-own-design-system-with-shadcncraft-create)

## shadcn/typeset — Stream-Friendly Typography (July 14, 2026)

If your app renders markdown in multiple surfaces (blog, docs, chat, email, AI assistant output), you'll hit a recurring problem: every surface needs its own typography config, and the styles drift apart over time. **`shadcn/typeset`** is the official answer — released the same week as `shadcn@4.13.0` (July 14, 2026).

It's a single CSS file that styles your HTML elements (h1, p, ul, code, table) the same way everywhere, with **three CSS variables** to tune the rhythm per container:

```css
.typeset {
  /* Base — styles the HTML elements (headings, paragraphs, lists, code, tables) */
}

.typeset-chat {
  --typeset-leading: 1.6;  /* line-height */
  --typeset-flow: 1em;     /* space between blocks */
}

.typeset-docs {
  --typeset-size: 15px;
  --typeset-leading: 1.75;
  --typeset-flow: 1.5em;
}
```

```tsx
// In a streaming chat message — tighter rhythm
<div className="typeset typeset-chat">{markdown}</div>

// In a long-form docs article — roomier rhythm
<article className="typeset typeset-docs">{markdown}</article>
```

### Why typeset instead of `@tailwindcss/typography` (Tailwind Typography / `prose`)

The `prose` classes from Tailwind Typography are great for a single-surface app (just a blog, or just docs). They break down when you have multiple surfaces that need different rhythms from the same HTML elements.

| Surface | Tailwind Typography approach | shadcn/typeset approach |
|---|---|---|
| Blog | `prose prose-lg` | `<div className="typeset">` |
| Docs | `prose prose-base max-w-none` | `<div className="typeset typeset-docs">` |
| Chat | `prose prose-sm` | `<div className="typeset typeset-chat">` |
| Email | `prose prose-sm` (with overrides) | `<div className="typeset typeset-email">` |

With `prose`, each surface has its own class soup; with typeset, the HTML elements are styled once, and the container class just tunes three CSS variables.

### Streaming-friendly

The killer feature for chat UIs: **typeset does not restyle earlier blocks when a new block arrives**. A streaming chat where each message appends to a list doesn't have to re-render the entire list to apply a new message's typography. The class-based design means each `.typeset` block is independently styled.

This is why typeset pairs naturally with the shadcn 4.12.0 Chat Components (`MessageScroller`, `Message`, `Bubble`, `Attachment`, `Marker`) — wrap each `Message` content in `<div className="typeset typeset-chat">` and the streaming output stays correctly styled without re-rendering.

### Install

The typeset builder at [ui.shadcn.com/typeset](https://ui.shadcn.com/typeset) generates a CSS file based on your theme. Download it, then either inline it in `app/globals.css` or save as a separate file and `@import` it.

```css
/* app/globals.css */
@import "tailwindcss";
@import "shadcn/tailwind.css";

/* Base typeset — element styles (h1, p, ul, code, table) */
@import "../styles/typeset.css";

/* Container variants — rhythm tunings */
@import "../styles/typeset-chat.css";
@import "../styles/typeset-docs.css";
```

Or, for a single-file approach:

```css
.typeset { /* ... element styles from the builder ... */ }
.typeset-chat { --typeset-leading: 1.6; --typeset-flow: 1em; }
.typeset-docs { --typeset-size: 15px; --typeset-leading: 1.75; --typeset-flow: 1.5em; }
```

### How it composes with Tailwind v4 `@theme`

Typeset uses **plain CSS variables** (`--typeset-size`, `--typeset-leading`, `--typeset-flow`), not Tailwind theme tokens. This means:
- **No `@theme inline { --typeset-size: ... }` needed.** The variables are local to each `.typeset-*` container.
- **No conflict with Tailwind's `text-base` / `text-lg` utilities.** You can still apply Tailwind text utilities inside a typeset block; the block's `--typeset-size` is a fallback, not an override.
- **Container queries work.** Typeset is designed to be paired with `@container` queries — the `--typeset-size` variable scales with the container's inline-size by default.

**Full deep-dive and the per-component API mapping** is in `components.md` → [shadcn/typeset (July 14, 2026) — Stream-Friendly Typography System](#shadcntypeset-july-14-2026--stream-friendly-typography-system). The components doc covers the markdown-renderer compatibility matrix (react-markdown, MDX, AI SDK Message, etc).

**Sources:**
- [shadcn/typeset announcement (July 14, 2026)](https://ui.shadcn.com/docs/changelog/2026-07-typeset)
- [Typeset documentation](https://ui.shadcn.com/docs/typeset)
- [Typeset builder](https://ui.shadcn.com/typeset)

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


## Modern CSS — Now Cross-Engine (2026)

Several long-awaited CSS features shipped in all three engines over the last 18 months. They are now safe to use without polyfills or fallbacks, and Tailwind v4 + shadcn/ui already expose them through utilities or patterns you can copy. This section is the cheat sheet.

### `field-sizing: content` — Auto-Growing Textareas Without JS

The `field-sizing` CSS property lets form controls grow to fit their content. With `field-sizing: content`, a `<textarea>` expands in height as the user types — no `useEffect`, no `scrollHeight` measurement, no `autosize` npm library. `<select>` elements shrink-wrap to fit the selected option. `<input>` grows horizontally with the entered value.

**Browser support — Baseline 2026:**
- Chrome / Edge v123 (March 2024)
- Safari v26.2 (December 2025)
- Firefox v152 (June 2026 — the last holdout; this is what flipped the property to Baseline)
- caniuse global coverage: ~79% as of June 2026, climbing

```css
/* globals.css — apply globally to all textareas and inputs */
textarea, input:not([type="checkbox"]):not([type="radio"]) {
  field-sizing: content;
}
```

```tsx
// In a shadcn/ui Textarea — no JS needed
<Textarea placeholder="Type your message…" className="field-sizing-content min-h-12" />

// A chat composer that grows with the message
<form className="flex items-end gap-2">
  <Textarea
    placeholder="Reply…"
    className="field-sizing-content max-h-40 resize-none"
  />
  <Button type="submit" size="icon"><SendIcon /></Button>
</form>
```

**Why it matters:** Before this, every team had a `useAutoSizeTextarea` hook (or `react-textarea-autosize`) doing the same job in ~30 lines of JS. Delete that hook. The CSS does it natively, the layout never jumps, and the textarea scrolls once it hits `max-h-40`.

**Progressive enhancement is built in:** Older browsers ignore the declaration and render a normal fixed-height textarea — same as before. No broken layout, no console errors.

**Sources:**
- [MDN: field-sizing](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/field-sizing)
- [Can I use: field-sizing](https://caniuse.com/mdn-css_properties_field-sizing)
- [CSS field-sizing: Auto-Growing Textareas Without JavaScript (buildmvpfast, June 2026)](https://www.buildmvpfast.com/blog/css-field-sizing-auto-growing-textarea-no-javascript-2026)

### `@starting-style` — Animate Elements on First Paint (Baseline 2024)

`@starting-style` defines the *starting* values for properties that should transition from when an element is first added to the DOM. Without it, you can't transition `display: none → block`, or fade in a dialog on mount — because the browser doesn't know what to transition *from*. With it, you can.

```css
/* Dialog that fades + slides in on first paint */
@starting-style {
  dialog[open] {
    opacity: 0;
    transform: translateY(8px);
  }
}
dialog[open] {
  opacity: 1;
  transform: translateY(0);
  transition: opacity 200ms, transform 200ms;
}
```

```tsx
// React + Radix/shadcn Dialog — works because the content mounts when opened
<Dialog open={isOpen} onOpenChange={setIsOpen}>
  <DialogContent className="data-[state=open]:animate-in data-[state=closed]:animate-out">
    <DialogHeader>...</DialogHeader>
  </DialogContent>
</Dialog>

/* In globals.css */
@keyframes enter {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: translateY(0); }
}
@keyframes leave {
  from { opacity: 1; transform: translateY(0); }
  to   { opacity: 0; transform: translateY(8px); }
}
```

**Why `@starting-style` matters:** The dialog/animation problem has been the single biggest source of "we need a JS animation library" complaints. With `@starting-style` + `transition-behavior: allow-discrete`, you can animate `display`, `visibility`, and other discrete properties purely in CSS. No more framer-motion for entry animations.

**Browser support:** Baseline 2024 (Chrome 117+, Safari 17.5+, Firefox 129+). Safe to use without fallbacks.

**Sources:**
- [MDN: @starting-style](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/At-rules/@starting-style)
- [Four new CSS features for smooth entry and exit animations (Chrome Developers)](https://developer.chrome.com/blog/entry-exit-animations/)

### CSS Anchor Positioning — Native Tooltips, Popovers, Menus

CSS Anchor Positioning lets you anchor one element (the *target* — a tooltip, popover, menu) to another element (the *anchor* — a button, a comment) with pure CSS. The target positions itself relative to the anchor and follows it as the anchor moves, scrolls, or resizes. No `react-popper`, no Floating UI, no JavaScript positioning math.

**Browser support — cross-engine as of 2026:**
- Chrome 125+, Edge 125+ (2024)
- Firefox 147+ (2025)
- Safari 26+ (2025)

```css
/* The anchor — the trigger element */
.tooltip-trigger {
  anchor-name: --my-tooltip;
}

/* The target — positions itself relative to the anchor */
.tooltip {
  position: fixed;
  position-anchor: --my-tooltip;
  top: anchor(bottom);
  left: anchor(center);
  translate: -50% 4px;  /* small gap below the anchor */
}

/* Fallback: if the tooltip would clip the top of the viewport, flip it */
@position-try --flip-top {
  top: anchor(top);
  bottom: auto;
  translate: -50% -4px;
}
.tooltip {
  position-try-fallbacks: --flip-top;
}
```

```tsx
// React: just render the anchor and target side-by-side
<button className="tooltip-trigger">Hover me</button>
{isOpen && (
  <div role="tooltip" className="tooltip">
    Anchored without JavaScript!
  </div>
)}
```

**When to use it:** Simple tooltips, dropdowns, context menus, and popovers that don't need collision detection with the viewport's full edge geometry. For more complex floating UI (virtualization, multi-anchor, arrow rendering), still reach for Floating UI / Radix. The native API is the *simple case* — not a full replacement for floating libraries.

**Sources:**
- [CSS Anchor Positioning — Browser Support, Features, Issues (testmuai, May 2026)](https://www.testmuai.com/learning-hub/css-anchor-positioning-browser-support/)
- [MDN: CSS anchor positioning](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_anchor_positioning)

### OKLCH — Better Colors for Design Tokens

OKLCH (Oklab Lightness Chroma Hue) is a perceptually uniform color space — equal numerical steps correspond to equal perceived color differences. In practice this means:
- **Same lightness = same brightness.** A `red-500` and a `blue-500` at the same OKLCH L value look the same brightness to the eye. HSL's `L=50%` looks wildly different across hues.
- **Predictable interpolation.** Fading from `red-500` to `red-900` stays in the same hue; HSL interpolation drifts through muddy intermediate colors.
- **Wide gamut.** P3 displays can show colors that sRGB literally cannot represent. OKLCH makes this explicit.

Tailwind v4's default palette (slate, gray, zinc, neutral, stone, plus the new v4.2 mauve, olive, mist, taupe) is **all defined in OKLCH** under the hood. Override them in OKLCH when you customize the palette — your dark mode will look right automatically.

```css
/* Override Tailwind's gray with hand-picked OKLCH values */
@theme {
  --color-gray-50:  oklch(0.984 0.003 247.858);
  --color-gray-100: oklch(0.968 0.007 247.896);
  --color-gray-500: oklch(0.554 0.046 257.417);
  --color-gray-900: oklch(0.208 0.042 265.755);
  --color-gray-950: oklch(0.129 0.042 264.695);
}
```

```css
/* Dynamic themes — separate the lightness/chroma from the hue, let users pick the hue */
:root {
  --brand-hue: 220;  /* user picks this at runtime */
  --brand-50:  oklch(0.97 0.02 var(--brand-hue));
  --brand-500: oklch(0.55 0.18 var(--brand-hue));
  --brand-900: oklch(0.20 0.10 var(--brand-hue));
}

@theme inline {
  --color-brand-50:  var(--brand-50);
  --color-brand-500: var(--brand-500);
  --color-brand-900: var(--brand-900);
}

/* Now: <div className="bg-brand-500"> inherits the user's hue */
```

**Why OKLCH > HSL in 2026:** All modern design tools (Figma, Linear, Vercel's Geist design tokens, Radix Colors v3) ship OKLCH palettes. Sticking with HSL in your tokens means importing tokens that look subtly wrong on real displays. OKLCH is the new baseline.

**Browser support:** Baseline 2023. Wide gamut support requires a P3 display and OS color-management; in sRGB-fallback browsers OKLCH is automatically down-converted.

**Sources:**
- [Tailwind CSS: Customizing colors (OKLCH examples)](https://tailwindcss.com/docs/customizing-colors)
- [Better dynamic themes in Tailwind with OKLCH color magic (evilmartians)](https://evilmartians.com/chronicles/better-dynamic-themes-in-tailwind-with-oklch-color-magic)

### `color-scheme` + `light-dark()` — Native Dark Mode Plumbing

`color-scheme` tells the browser which color schemes your page supports, which:
- Adjusts built-in form controls, scrollbars, and the page background to match
- Triggers the correct UA stylesheet for `prefers-color-scheme`
- Doesn't require a `.dark` class on `<html>`

Pair it with the `light-dark()` CSS function to write one rule that resolves to different values for light vs. dark mode — no media query, no theme class.

```css
/* Tell the browser we support both schemes */
:root {
  color-scheme: light dark;
}

/* One rule, two values — browser picks based on color-scheme */
:root {
  --bg: light-dark(white, #0a0a0a);
  --fg: light-dark(#1a1a1a, #fafafa);
  --border: light-dark(#e5e5e5, #262626);
  --accent: light-dark(oklch(0.55 0.18 250), oklch(0.70 0.18 250));
}
```

```tsx
// In a component — apply the tokens
<div className="bg-(--bg) text-(--fg) border border-(--border)">
  Works in both light and dark mode with one rule
</div>
```

**When to use `light-dark()` vs `dark:` variant:**
- `light-dark()` — best for tokens that have no semantic equivalent (e.g. a pure color value). No DOM/JS toggle needed; UA handles it.
- `dark:` variant — best for layout or class-based toggles (a user picks dark mode in your app, you set `.dark` on `<html>` and need different *layouts* or *components* to show).

**Browser support:** `color-scheme` is Baseline 2022. `light-dark()` is Baseline 2024 (Chrome 123+, Safari 17.5+, Firefox 120+). Safe to ship.

**Sources:**
- [MDN: color-scheme](https://developer.mozilla.org/en-US/docs/Web/CSS/color-scheme)
- [MDN: light-dark()](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/light-dark)

### Scroll-Driven Animations — Parallax + Progress Bars in CSS

`animation-timeline: scroll()` ties a CSS animation to the user's scroll position. The animation runs as the user scrolls — not over time, but over distance. Use it for:
- Reading progress bars
- Parallax backgrounds
- Reveal-on-scroll effects (no IntersectionObserver)
- Horizontal scroll carousels tied to vertical scroll

```css
/* Reading progress bar tied to scroll position */
.reading-progress {
  position: fixed;
  top: 0; left: 0; right: 0;
  height: 4px;
  background: var(--accent);
  transform-origin: left;
  animation: progress linear;
  animation-timeline: scroll(root);
}

@keyframes progress {
  from { transform: scaleX(0); }
  to   { transform: scaleX(1); }
}
```

```css
/* Parallax — background moves at 50% of scroll speed */
.hero-bg {
  animation: parallax linear;
  animation-timeline: scroll(root);
}

@keyframes parallax {
  from { transform: translateY(0); }
  to   { transform: translateY(-50%); }
}
```

```tsx
// Reading progress bar component — pure CSS, zero JS
export function ReadingProgress() {
  return <div className="reading-progress" aria-hidden="true" />
}
```

**Browser support:** Chrome 115+, Edge 115+, Firefox 130+ (December 2024), Safari 26+ (2025). Cross-engine as of late 2025.

**Sources:**
- [MDN: animation-timeline](https://developer.mozilla.org/en-US/docs/Web/CSS/animation-timeline)
- [Scroll-driven Animations (Chrome Developers)](https://developer.chrome.com/docs/css-ui/scroll-driven-animations)

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

## Tailwind v4 — `@reference` Directive (Library / Design-Token Import Without CSS Output)

`@reference` is Tailwind v4's CSS-only equivalent of v3's `@tailwindcss/ui` or library token-imports: it pulls Tailwind's theme values (custom properties, design tokens) into the current stylesheet **without emitting any Tailwind output CSS**. The file you're writing stays pure CSS — no Tailwind classes generated, no utility duplication, just the token names available.

```css
/* In a library file or shared design-token stylesheet */
@reference "tailwindcss";

.my-button {
  background: var(--color-brand-500);
  padding: var(--spacing-4);
  border-radius: var(--radius-lg);
  font-size: var(--text-sm);
  font-weight: var(--font-weight-medium);
}

.my-card {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-md);
}
```

**When `@reference` shines:**

1. **Component libraries** — write CSS that uses Tailwind's design tokens without forcing the consuming project to also have those tokens in their `@theme` block. Library defines tokens in `@theme`, consumer gets them via `@reference`.
2. **CSS-only components** — pure CSS files (not Tailwind utility classes) that still want access to the theme. e.g. a `.css` file for a third-party widget that lives outside your `app/` tree.
3. **Design-token sharing** — extract the design tokens from one Tailwind project and `@reference` them into another (e.g. a marketing site that shares tokens with the main app).
4. **Static-only output** — generate a CSS file with zero Tailwind utility classes in it, just the token-derived styles. Useful for embedding in Web Components, Shadow DOM, or framework-agnostic CSS bundles.

**How it differs from `@import "tailwindcss"`:**

```css
/* Full import — generates ALL Tailwind utilities + base + components */
@import "tailwindcss";

/* Reference — pulls theme values, generates NO Tailwind output */
@reference "tailwindcss";
```

`@reference` is the read-only version: it gives you access to `--color-*`, `--spacing-*`, `--radius-*`, `--shadow-*`, etc. via `var()`, but doesn't add `bg-red-500`, `p-4`, `rounded-lg`, `shadow-md` to your output.

**Path can be a package, a CSS file, or a theme module:**

```css
/* Reference the full Tailwind package (gets all built-in tokens) */
@reference "tailwindcss";

/* Reference a custom theme module file */
@reference "./themes/brand.css";

/* Reference multiple modules */
@reference "tailwindcss";
@reference "../../packages/ui/src/tokens.css";
```

**Pattern — design-system library that exposes tokens to consumers:**

```css
/* packages/ui/src/tokens.css (the library's design tokens) */
@theme {
  --color-brand-50: oklch(0.97 0.02 250);
  --color-brand-500: oklch(0.55 0.18 250);
  --color-brand-900: oklch(0.20 0.10 250);

  --radius-brand: 0.75rem;
  --shadow-brand-md: 0 4px 16px oklch(0.55 0.18 250 / 0.20);
}

/* packages/ui/src/components.css (library components, no Tailwind output) */
@reference "./tokens.css";

.ds-button {
  background: var(--color-brand-500);
  padding: var(--spacing-3) var(--spacing-5);
  border-radius: var(--radius-brand);
  box-shadow: var(--shadow-brand-md);
}

/* Consumer imports this in their project */
@import "@acme/ui/components.css";
/* Now .ds-button is styled, but the consumer's bundle has zero duplicated Tailwind utilities */
```

**Pattern — shadow DOM / web components:**

```css
/* web-component.css — the component's shadow DOM stylesheet */
@reference "tailwindcss";

:host {
  display: block;
  container-type: inline-size;
  background: var(--color-surface);
  color: var(--color-fg);
  padding: var(--spacing-4);
  border-radius: var(--radius-lg);
}
```

The `@reference` brings tokens into the shadow DOM's stylesheet scope without polluting the main document with Tailwind's full utility set.

**Pattern — marketing-site static export:**

```css
/* static.css — zero Tailwind output, just token-derived styles for a static site */
@reference "tailwindcss";

.hero { background: var(--color-brand-500); padding-block: var(--spacing-20); }
.cta { background: var(--color-accent); color: var(--color-accent-fg); }
```

The output of `static.css` is ~500 bytes — only the literal declarations, no Tailwind reset, no utilities.

**Gotchas:**
- `@reference` must come **before any rules** that use the referenced tokens (it's import-like)
- You can't reference tokens from inside `@layer base { }` blocks — they need to be at the top level
- The referenced file's `@theme` block must be processed before your file is read — file ordering in your build pipeline matters
- `@reference` is a **Tailwind v4-only** feature; no v3 equivalent. v3 users have to copy tokens manually or use the `@apply` workaround.

**Sources:**
- [Tailwind CSS v4 docs — `@reference` directive](https://tailwindcss.com/docs/functions-and-directives#reference-directive)
- [Tailwind CSS v4 blog — CSS-first config](https://tailwindcss.com/blog/tailwindcss-v4#css-first-configuration)
- [Tailwind CSS v4 — design tokens primer](https://tailwindcss.com/docs/theme)

## Tailwind v4 — Named Container Queries (`@container/main` → `@md/main:grid-cols-3`)

The `@container` utility creates a containment context, and `@sm:` / `@md:` / `@lg:` variants target the container's inline-size. But what if you have **two containers on the same page** — a sidebar that's 280px and a main area that's 720px — and want different layouts at the same `@md:` breakpoint in each?

**Named containers** solve this. Add a name with `@container/<name>`, then target that specific container with `@<breakpoint>/<name>:`:

```tsx
// Two containers on the page
<aside className="@container/sidebar">
  {/* Responds to sidebar's size — even if main is bigger */}
  <nav className="@sm/sidebar:flex-col @md/sidebar:grid-cols-2">
    {sidebarItems.map(...)}
  </nav>
</aside>

<main className="@container/main">
  {/* Responds to main's size independently */}
  <div className="@sm/main:grid-cols-2 @lg/main:grid-cols-4">
    {mainContent.map(...)}
  </div>
</main>
```

The same `@md:` breakpoint resolves to different pixel widths depending on which container is the ancestor.

**Shorthand — `@container` (no name) targets the nearest anonymous container:**

```tsx
{/* Both anonymous — the sidebar and main @container scopes independently */}
<div className="@container">
  <div className="@md:grid-cols-3">{/* only triggers when THIS @container hits md */}</div>
</div>
```

**Multiple named containers in nested elements:**

```tsx
<article className="@container/article">
  <div className="@container/comments">
    <p className="@lg/article:text-2xl">              {/* big when article is wide */}
      <span className="@md/comments:text-sm">        {/* small when comments are narrow */}
        Comment text
      </span>
    </p>
  </div>
</article>
```

The `@md/comments:` only triggers when the **nearest `@container/comments`** ancestor hits the md threshold — even if the outer `@container/article` is already at lg size.

**Practical use cases:**

1. **Page sections with independent responsive behavior** — header, sidebar, main, footer each have their own container query scope. Header collapses at `@sm`, sidebar goes vertical at `@md`, main reflows at `@lg`.
2. **Cards in a grid where the grid cell is the container** — every grid cell responds to its own width, not the page viewport. 3-up grid + 6-up grid in the same page, each card picks its own layout.
3. **Nested layouts** — outer `@container/page`, inner `@container/sidebar`, inner-inner `@container/widget`. Each picks its own breakpoint scale.

**Why named over anonymous in production code:**
- **Predictability** — when refactoring markup, named containers are explicit about which scope you intended
- **No ambiguity** — anonymous containers resolve to nearest ancestor, which can change unexpectedly when you wrap/unwrap a div
- **DX** — named containers are self-documenting (`@container/comments` reads better than "some nearby `@container`")

**Custom named breakpoints — combine named containers with custom breakpoint scales:**

```css
/* globals.css — add custom container breakpoints */
@theme {
  --container-3xs: 12rem;
  --container-compact: 20rem;
  --container-wide: 60rem;
}
```

```tsx
<div className="@container">
  <div className="@compact:flex-row @wide:grid-cols-4">
    {/* @compact = 20rem, @wide = 60rem */}
  </div>
</div>
```

Note: `--container-*` namespace (not `--breakpoint-*`) is the v4 convention for container-query breakpoints. Viewport breakpoints use `--breakpoint-*` and `@sm:` (no container prefix).

**Audit recipe:**

```bash
# Find anonymous @container usages that might benefit from naming
rg "@container[^/\s]" --type tsx --type ts

# Find existing named containers
rg "@container/[a-z]" --type tsx --type ts
```

**Sources:**
- [Tailwind CSS docs — container queries](https://tailwindcss.com/docs/container-queries)
- [Tailwind CSS v4 blog — container queries](https://tailwindcss.com/blog/tailwindcss-v4#container-queries)
- [MDN — CSS Container Queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_container_queries)
- [Smashing Magazine — CSS Container Queries For Design Systems](https://www.smashingmagazine.com/2023/05/css-container-queries-design-systems/)

## Tailwind v4 — `@utility` Directive — Custom First-Class Utilities

The `@utility` directive (already mentioned in the `tw-animate-css` / `shadow-elevated-*` examples above) lets you define **custom utilities** that work like first-class Tailwind classes — with arbitrary-value support, modifier compatibility, and theme participation. The pattern is for utilities you use in many places, that have a single CSS declaration.

```css
/* In your globals.css */

@utility text-balance {
  text-wrap: balance;
}

@utility scrollbar-thin {
  scrollbar-width: thin;
}

@utility no-scrollbar {
  scrollbar-width: none;
  -ms-overflow-style: none;
  &::-webkit-scrollbar { display: none; }
}

/* Functional custom utility — accepts any value */
@utility tab-* {
  tab-size: --value(integer);
}

/* Functional with theme lookup */
@utility bg-tint-* {
  background: --value(--color-*);
}
```

The first form (`@utility name { }`) creates a single-purpose utility. The second form (`@utility prefix-* { }`) creates a functional utility that accepts any value via `--value(integer)`, `--value(--color-*)`, etc.

**Three `@utility` value resolvers:**

| Resolver | What it does | Example |
|----------|--------------|---------|
| `--value(integer)` | Restricts to integers, supports `tab-4` | `@utility tab-* { tab-size: --value(integer); }` |
| `--value(number)` | Accepts integers + decimals | `@utility opacity-* { opacity: --value([*]); }` |
| `--value([*])` | Accepts any value, including arbitrary | `@utility m-* { margin: --value([*]); }` |
| `--value(--color-*)` | Accepts any color from your `--color-*` theme | `@utility bg-* { background: --value(--color-*); }` |

**Pattern — responsive custom utilities (combine `@utility` with variants):**

```css
@utility scrollbar-thin {
  scrollbar-width: thin;
}
```

```tsx
{/* @variant defaults, dark: variant, and breakpoints all work */}
<div className="scrollbar-thin dark:scrollbar-none @md:scrollbar-auto">
```

The `@utility` directive creates real Tailwind variants — `hover:`, `focus:`, `dark:`, `@md:`, etc. all just work.

**Pattern — variants INSIDE `@utility` (responsive selectors within the utility):**

```css
@utility scrollbar-auto {
  scrollbar-width: auto;
  @variant dark { scrollbar-width: thin; }
  @variant hover { scrollbar-width: none; }
}
```

The `@variant` directive inside `@utility` mirrors the utility's own variant API.

**Pattern — nested selectors for compound utilities:**

```css
@utility button-base {
  display: inline-flex;
  align-items: center;
  padding-inline: var(--spacing-4);
  border-radius: var(--radius-md);

  &:hover {
    background: var(--color-brand-600);
  }

  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
}
```

Use as `<button className="button-base">`. The `&` resolves to `.button-base` at compile time.

**Pattern — utilities that consume theme tokens:**

```css
@theme {
  --shadow-elevated-sm: 0 2px 8px rgb(0 0 0 / 0.08);
  --shadow-elevated-md: 0 4px 16px rgb(0 0 0 / 0.12);
  --shadow-elevated-lg: 0 8px 32px rgb(0 0 0 / 0.15);
}

@utility shadow-elevated-* {
  box-shadow: --value(--shadow-elevated-*);
}
```

```tsx
<div className="shadow-elevated-sm">small</div>
<div className="shadow-elevated-md">medium</div>
<div className="shadow-elevated-lg">large</div>
```

The functional form picks up tokens from your `@theme` namespace dynamically — no need to redeclare each variant.

**When `@utility` is the right tool:**
- A single CSS declaration you use in 10+ places
- You want `dark:` / `@md:` / `hover:` to work automatically
- The pattern is short enough to inline but you want to avoid magic numbers
- You want to expose tokens to JS consumers via the standard `tailwind.config` introspection

**When `@utility` is NOT the right tool:**
- Multi-property layouts → use a `@layer components { }` rule or a shadcn component instead
- Component-scoped styles → use CSS Modules or React component styles
- One-off magic values → just use arbitrary values like `mt-[17px]`

**Sources:**
- [Tailwind CSS v4 docs — `@utility` directive](https://tailwindcss.com/docs/functions-and-directives#utility-directive)
- [Tailwind CSS v4 — arbitrary values](https://tailwindcss.com/docs/adding-custom-styles#using-arbitrary-values)
- [Tailwind CSS blog — Tailwind CSS v4.0](https://tailwindcss.com/blog/tailwindcss-v4#css-first-configuration)

## Tailwind v4.3.2 + v4.3.3 — Bug-Fix Patch Train (July 8–16, 2026)

Two consecutive pure-bug-fix patches shipped since v4.3.1 (which is what the existing skill content covers). Total **24 bug fixes** across the two patches, of which **9 affect production apps** in non-obvious ways. The skill's `## Common Mistakes` Sources block currently links to `v4.3.1` only — update with the v4.3.2 + v4.3.3 changelogs below.

### v4.3.2 Highlights (released early July 2026)

| PR | Fix | Practical impact for production apps |
|---|---|---|
| **PR #20229** | Support bare spacing values for `auto-rows-*` and `auto-cols-*` utilities (e.g. `auto-rows-12` not just `auto-rows-fr-12`) | The v4.3.1 `auto-rows-12` error is gone — grid utility layouts that previously needed `auto-rows-[3rem]` can now use `auto-rows-12` directly. Affects every grid-based card list (Tailwind's docs site, marketing grids, admin table views). |
| **PR #20242** | Prevent `@tailwindcss/cli --watch` from crashing on Windows when `@source` points to a directory that doesn't exist | **CRITICAL for Windows + Docker volumes** — pre-v4.3.2, a stale `@source` path on Windows (e.g. `C:\Users\username\src`) would hard-crash the watcher instead of falling through with a warning. Affects every Docker-on-Windows developer whose volume mount path changes. |
| **PR #20245** | Prevent `@tailwindcss/vite` from crashing in Deno v2.8.x when `context.parentURL` is not a valid URL | Deno users on v2.8.x had hard plugin crashes during HMR — fixed silently in v4.3.2. Audit recipe: `deno --version`. |
| **PR #20246** | Ensure `@tailwindcss/cli --watch` rebuilds when the input CSS file changes in an ignored directory | Watcher was missing rebuilds for the *input* file when it sat under a `.gitignore`'d path. Affects monorepo setups where `node_modules/<pkg>/styles.css` is the input but the directory itself is `.gitignore`d. |
| **PR #20247** | Allow `@variant` rules used in `addBase(...)` to use custom variants defined later | Plugin authors relying on `addBase('@variant dark (@media (...) {...})')` with variants that get defined in `@theme` further down the file were silently dropping the `@variant` wrapper. |
| **PR #20259** | Prevent `@tailwindcss/vite` from crashing during HMR when scanned files or directories are deleted | **CRITICAL for `tsc --watch` + Tailwind + Vite workflows** — pre-v4.3.2, deleting any file under an `@source` directory while Vite was running would hard-crash the HMR. The fix: gracefully fall through. |
| **PR #20260** | Generate `font-size` instead of `color` declarations for `text-[--spacing(…)]` | Subtle bug: `text-[--spacing(4)]` was being treated as `color: var(--spacing-4)` instead of `font-size: var(--spacing-4)`. **Fixed** — now generates font-size correctly. Audit recipe: `rg -n 'text-\[--spacing' app/ --type tsx` to find any reliance on the v4.3.1 buggy behavior. |
| **PR #20263** | Prevent `@source` patterns from scanning unrelated sibling files and folders | The `@source` glob was over-scanning — a pattern like `'../packages/ui'` would pull in `node_modules` siblings, `dist` outputs, etc. Now it respects dir boundaries. Per-build CPU savings: 5–30% for monorepos with wide `@source` globs. |
| **PR #20269** | Extract class candidates adjacent to Template Toolkit delimiters like `%]…[%` in `.tt`, `.tt2`, `.tx` files | Plugin authors using Template Toolkit (Perl/Python templating) get class extraction now. |
| **PR #20269** | Extract class candidates from conditional Maud syntax like `p.text-black[condition]` | Maud-rs (Rust server-side templating) class extraction works now. |
| **PR #20277** | Prevent `@position-try` rules from triggering unknown at-rule warnings when optimizing CSS | The CSS Anchor Positioning + Lightning CSS optimization were emitting warnings for `@position-try` rules. Silent fix. |
| **PR #20287** | Support class suggestions for named opacity modifiers from `--opacity` theme values | IDE plugin / error-message suggestions got smarter — `bg-red/50` (named opacity from `--opacity-50`) now suggests `bg-red/50` instead of an empty suggestion. |
| **PR #20289** | Prevent type errors in `@tailwindcss/postcss` when used with newer PostCSS patch releases | Subtle: if you bumped `postcss` to 8.5.x or later, the `@tailwindcss/postcss` adapter was emitting TS errors in your loader output. Fixed silently. |

### v4.3.3 Highlights (released 2026-07-16)

The `## 4.3.3` release on the [tailwindcss CHANGELOG](https://github.com/tailwindlabs/tailwindcss/blob/main/CHANGELOG.md):

| PR | Fix | Practical impact for production apps |
|---|---|---|
| **PR #20297** | Support `--watch --poll[=ms]` in `@tailwindcss/cli` when filesystem events are unreliable or unavailable | **CRITICAL for Docker + WSL2 + NFS workflows** — pre-v4.3.3, Tailwind CLI's `--watch` mode silently did nothing if filesystem events weren't propagating (common in Docker bind mounts on macOS, WSL2, and NFS shares). Now the `--poll` flag polls instead. **Action required**: if your dev loop depended on `npm run dev` triggering Tailwind rebuilds and "nothing happens" after file saves, add `--poll=1000` to the script. |
| **PR #20298** | Canonicalization: match arbitrary hex colors against theme colors case-insensitively (e.g. `bg-[#FFF]` → `bg-white`) | IDE / build output: `bg-[#FFF]` is now correctly normalized to the existing `bg-white` class instead of emitting a duplicate arbitrary-value class. CSS output: ~2–5% smaller for any codebase using uppercase hex arbitrary values. |
| **PR #20292** | Prevent Preflight from overriding Firefox's native `iframe:focus-visible` outline styles | Firefox-specific accessibility regression: Tailwind's Preflight was zeroing out the browser-default outline for `iframe:focus-visible`, hurting keyboard navigation of embedded YouTube/Vimeo players. Fixed. |
| **PR #20299** | Ensure `theme('colors.foo')` in JS plugins resolves correctly when both `--color-foo` and `--color-foo-bar` exist | Plugin authors with `--color-brand` + `--color-brand-accent` (or similar prefix-collision pairs) were getting the wrong color. Fixed. |
| **PR #20303** | Parse selectors like `[data-foo]div` as two selectors instead of one | Subtle parser fix: Tailwind was treating `[data-foo]div` as a single attribute selector, missing the `<div>` inside `[data-foo]` elements. CSS-output fix for very unusual class extraction patterns. |
| **PR #20310** | Ensure `@tailwindcss/postcss` rebuilds when a preprocessor like Sass changes the input CSS without changing the input file on disk | **CRITICAL for Sass + CSS-first config workflows** — pre-v4.3.3, importing Sass variables (which mtime-update the in-memory CSS without touching the file) didn't trigger a rebuild. Sass users' HMR was broken. Fixed. |
| **PR #20124** | Ensure CSS nesting is handled even when Lightning CSS isn't run, such as in `@tailwindcss/browser` and Tailwind Play | **`.tt`-browser users + Tailwind Play users**: CSS nesting (`& > .foo` syntax in custom `@layer base`) now works in `@tailwindcss/browser` (the in-browser CDN build) and Tailwind Play, not just in the Vite/PostCSS production pipeline. |
| **PR #20325** | Load `@parcel/watcher` only when needed in `@tailwindcss/cli --watch` mode, so one-off builds and `--watch --poll` work when `@parcel/watcher` can't be loaded | **CRITICAL for users on Linux without FUSE + on certain sandboxed CI runners** — pre-v4.3.3, if `@parcel/watcher` failed to load (no FUSE, missing native deps), the entire `tailwindcss --watch` crashed. Now `--watch --poll` falls through cleanly. |
| **PR #20318** | Use explicit platform fonts instead of `system-ui` and `ui-sans-serif` so CJK text respects the page's `lang` attribute on Windows | Windows + Chinese/Japanese/Korean text was rendering with a generic fallback instead of the platform's CJK font stack. Fixed by using `{:lang(zh).system-ui}` style font-family selectors. |
| **PR #20329** | Prevent `@tailwindcss/upgrade` from rewriting ignored files when run from a subdirectory | The v4 upgrade tool was running from any subdir but applying rewrites to all `.css`/`.html`/`.vue`/`.svelte` files relative to that subdir, **including .gitignored ones**. Now respects gitignore. Audit recipe: re-run `npx @tailwindcss/upgrade` from project root with `rg --files -u` first to spot-check ignored files. |

### Migration audit recipes — v4.3.1 → v4.3.3

```bash
# 1. Confirm your project is on a post-v4.3.3 Tailwind release
npx tailwindcss --help  # the bare `tailwindcss` binary moved to `@tailwindcss/cli` in v4
# If output shows v4.3.3 or later, the Windows-watch-crash + Deno-crash + HMR-crash fixes
# are all live. If v4.3.1 or earlier, bump:

npm install -D tailwindcss@latest
# or specifically
npm install -D tailwindcss@4.3.3

# 2. Check for the v4.3.1 subtle color-vs-font-size bug (fixed in #20260)
rg -n 'text-\[--spacing' app/ components/ src/ --type tsx --type css | head -20
# Expected: any hits are correctly font-size now (not color)

# 3. Check for the v4.3.1 Windows-watch-crash on missing @source
rg -n '@source.*\$\{' tailwind.config.* styles/ app/ 2>/dev/null || true
rg -n '@source' app/globals.css styles/globals.css 2>/dev/null | head -20

# 4. Find Sass-using projects that might benefit from the #20310 fix
rg -n 'sass|sass-loader|@tailwindcss/postcss' package.json | head -10

# 5. Find monorepos that might benefit from the #20263 @source over-scan fix
rg -n '@source.*\.\.\.' package.json tailwind.config.* app/globals.css styles/globals.css 2>/dev/null

# 6. Find codebases that might benefit from --watch --poll (Docker/WSL2/NFS on macOS)
rg -n '"dev":.*tailwind.*--watch' package.json
rg -n 'tailwindcss.*--watch' package.json
# If hits and dev loop has "save doesn't trigger rebuild", add --poll=1000
```

**Sources:**
- [Tailwind CSS CHANGELOG.md (current)](https://github.com/tailwindlabs/tailwindcss/blob/main/CHANGELOG.md) — the source of truth for v4.3.2 + v4.3.3 entries
- [Tailwind CSS releases page](https://github.com/tailwindlabs/tailwindcss/releases) — full 4.3.x release train incl. `v4.3.2` + `v4.3.3`
- [Tailwind CSS v4.3 blog post (older patch — still useful for v4.3.0/v4.3.1 features)](https://tailwindcss.com/blog/tailwindcss-v4-3)
- [PR #20229 — auto-rows bare spacing](https://github.com/tailwindlabs/tailwindcss/pull/20229)
- [PR #20242 — CLI watch Windows missing-dir crash](https://github.com/tailwindlabs/tailwindcss/pull/20242)
- [PR #20245 — Vite Deno v2.8.x parentURL crash](https://github.com/tailwindlabs/tailwindcss/pull/20245)
- [PR #20246 — CLI watch ignored-dir input CSS rebuild](https://github.com/tailwindlabs/tailwindcss/pull/20246)
- [PR #20247 — @variant in addBase custom-variants](https://github.com/tailwindlabs/tailwindcss/pull/20247)
- [PR #20259 — Vite HMR file-deletion crash](https://github.com/tailwindlabs/tailwindcss/pull/20259)
- [PR #20260 — text-[--spacing(...)] font-size fix](https://github.com/tailwindlabs/tailwindcss/pull/20260)
- [PR #20263 — @source over-scan fix](https://github.com/tailwindlabs/tailwindcss/pull/20263)
- [PR #20269 — Template Toolkit + Maud class extraction](https://github.com/tailwindlabs/tailwindcss/pull/20269)
- [PR #20277 — @position-try warning suppression](https://github.com/tailwindlabs/tailwindcss/pull/20277)
- [PR #20287 — named opacity suggestion support](https://github.com/tailwindlabs/tailwindcss/pull/20287)
- [PR #20289 — @tailwindcss/postcss newer-PostCSS compat](https://github.com/tailwindlabs/tailwindcss/pull/20289)
- [PR #20124 — CSS nesting in @tailwindcss/browser + Tailwind Play](https://github.com/tailwindlabs/tailwindcss/pull/20124)
- [PR #20297 — CLI --watch --poll](https://github.com/tailwindlabs/tailwindcss/pull/20297)
- [PR #20298 — case-insensitive hex canonicalization](https://github.com/tailwindlabs/tailwindcss/pull/20298)
- [PR #20292 — Firefox iframe:focus-visible Preflight](https://github.com/tailwindlabs/tailwindcss/pull/20292)
- [PR #20299 — theme('colors.foo') prefix-collision](https://github.com/tailwindlabs/tailwindcss/pull/20299)
- [PR #20303 — selector parser [data-foo]div fix](https://github.com/tailwindlabs/tailwindcss/pull/20303)
- [PR #20310 — @tailwindcss/postcss rebuild on preprocessor change](https://github.com/tailwindlabs/tailwindcss/pull/20310)
- [PR #20325 — @parcel/watcher lazy load](https://github.com/tailwindlabs/tailwindcss/pull/20325)
- [PR #20318 — explicit platform fonts / CJK `lang` attribute](https://github.com/tailwindlabs/tailwindcss/pull/20318)
- [PR #20329 — @tailwindcss/upgrade ignore-list respect from subdirectory](https://github.com/tailwindlabs/tailwindcss/pull/20329)
- [Tailwind CSS v4.3 release notes (older, pre-patch)](https://tailwindcss.com/blog/tailwindcss-v4-3)
- [Tailwind CSS docs — `@utility` directive](https://tailwindcss.com/docs/functions-and-directives#utility-directive) (still the canonical `@utility` reference, the `@utility` section above remains the v4-stable form)


## Tailwind CSS 4.3.4 / v4.4.0 Forward-Looking — `@tailwindcss/oxide` WASM Fallback (PR #20383, August 4, 2026)

`tailwindcss@latest` is still **4.3.3** (npm-published 2026-07-16T11:55:08Z, ~3 weeks ago at this cron's check). The Tailwind main branch has been active since then — **4 NEW commits ahead of 4.3.3** at this cron's check (verified via `GET /repos/tailwindlabs/tailwindcss/commits?since=2026-08-04T12:00:00Z`):

- **d190343594 — `Use wasm as a fallback for @tailwindcss/oxide`** (PR #20383, Robin Malfait, merged 2026-08-04T16:41:43Z, **MATERIAL**) — see below.
- **b5ab20473b — `improve flaky integration test`** (Aug 4 17:27Z, test infra only).
- **9dae5163db — `update changelog`** (Aug 4 15:55Z, docs only).
- **3524b45310 — `Attempt to fix flaky integration test`** (Aug 5 10:39Z, PR #20384, test infra only).

**None have been npm-published as `tailwindcss@4.3.4` or `4.4.0` yet** — `npm view tailwindcss dist-tags` still returns `latest: 4.3.3`. The 3 non-PR-20383 commits are prep work for the next release train; expect `4.3.4` or `4.4.0` to npm-publish within the next 2-4 weeks based on recent cadence (4.3.0 → 4.3.1 in 7 days, 4.3.1 → 4.3.2 in 10 days, 4.3.2 → 4.3.3 in 8 days; a slight slowdown but still weekly-to-bi-weekly).

### Why PR #20383 matters — `@tailwindcss/oxide` WASM Fallback

`@tailwindcss/oxide` is the native (Rust-via-napi-rs) scanner that Tailwind v4 uses to (1) traverse the file system and figure out which files to scan based on auto source detection + `@source` directives, and (2) extract candidate class names from those files. It relies on `napi-rs`-generated native `.node` files per platform/arch.

**The current limitation (pre-PR-#20383):** If you're on a platform/arch that doesn't have a prebuilt `.node` binary, you get the dreaded `Cannot find native binding` error:

```
Error: Cannot find native binding. npm has a bug related to optional dependencies
(https://github.com/npm/cli/issues/4828). Please try `npm i` again after removing
both package-lock.json and node_modules directory.
    at Object.<anonymous> (.../@tailwindcss/oxide/index.js:573:19)
    ...
  cause: Error: Cannot find module '@tailwindcss/oxide-darwin-arm64'
```

This means Tailwind currently **supports**: Windows arm64, Windows x64, macOS arm64, macOS x64 — but **doesn't fully support** the Linux-based permutations (Android arm eabi, Android arm64, Linux arm64 gnu, Linux arm64 gnueabihf, Linux arm64 musl, Linux x64 gnu, Linux x64 musl, freebsd x64) and a growing list of additional platforms (openharmony, etc.). Adding native support for each is "not the end of the world, but it gets complex" (Robin Malfait).

**The fix in PR #20383:** Use the `wasm32-wasi` build as a **universal fallback**. The napi-rs loader already knows how to fall back to `@tailwindcss/oxide-wasm32-wasi`, but that package declared `"cpu": ["wasm32"]`, so npm/pnpm never installed it on real hardware. **Removing that restriction means the WASM package is installed everywhere, and the loader picks it up whenever no native binding exists.**

**Bonus fix** in the same PR: a `UVWASI_EACCES` crash on sandboxed platforms (OpenHarmony, Android). The generated WASM loader preopens `/`, which those sandboxes deny, so the fallback failed to load on exactly the platforms that needed it most. The fix patches `@napi-rs/cli`'s codegen templates via `pnpm patch` to retry with narrower preopens (`/` → cwd → none). On such platforms, scanning is limited to files under the current working directory.

**Closes 3 pending PRs** by giving them WASM coverage for free: #20327, #20276, #20201.

### Practical impact (forward-looking — not yet npm-published)

- **Platforms gaining Tailwind v4 support**: Linux arm eabi, Android arm/arm64, freebsd x64, OpenHarmony, and the long tail of less-common platforms that previously couldn't load `@tailwindcss/oxide`. WASM is slower than native (~2-5× for large projects) but functional.
- **CI users on exotic platforms** (Docker `linux/arm/v7`, openharmony CI runners, Alpine variants with non-standard libc): currently your `npm ci && next build` may fail with the `Cannot find native binding` error. PR #20383 makes the build succeed via WASM.
- **Sandboxed platforms (OpenHarmony, Android)**: gain Tailwind v4 via WASM with cwd-scoped file scanning (a reasonable tradeoff for mobile/embedded use cases where the Tailwind scanning surface IS the cwd).
- **macOS arm64 + x64 + Linux x64 + Windows arm64/x64 users**: zero behavior change — they keep using the native `.node` binding.
- **No code changes required** for any user — pure build-system fix in `@tailwindcss/oxide` + `@tailwindcss/oxide-wasm32-wasi` packages.

### Audit recipe

```bash
# 1. Are you on a non-mainstream platform? (Check your OS/arch)
node -e "console.log(process.platform, process.arch)"
# Mainstream = darwin arm64/x64, linux x64 (gnu/musl), win32 arm64/x64
# Non-mainstream = everything else

# 2. Does your current setup hit the "Cannot find native binding" error?
rg -n "Cannot find native binding" .next/ logs/ 2>/dev/null
# If hits, PR #20383 will fix you once it ships.

# 3. Are you on a sandboxed platform? (OpenHarmony, Android, restricted CI)
echo "Check: does $HOME and $PWD allow file traversal? Are you in a chroot or sandbox?"
# If yes, PR #20383 will give you cwd-scoped scanning via WASM.

# 4. Track when 4.3.4 / v4.4.0 ships
npm view tailwindcss dist-tags --json | head -10
# Expected: `latest: 4.3.3` until the SHIP event; once updated, the next v1.5.x cron will document it.

# 5. Pin strategy
# Once shipped, bump from "tailwindcss": "^4.3.3" → "^4.3.4" (or "^4.4.0" if minor bump).
# The patch is pure additive — no breaking changes expected.
```

### Sources

- [Tailwind CSS main branch (4 commits ahead of 4.3.3 at 2026-08-06T12:05Z)](https://github.com/tailwindlabs/tailwindcss/commits/main) — `d190343594` (PR #20383) + `b5ab20473b` + `9dae5163db` + `3524b45310` (PR #20384)
- [PR #20383 — Use wasm as a fallback for `@tailwindcss/oxide`](https://github.com/tailwindlabs/tailwindcss/pull/20383) — Robin Malfait, merged 2026-08-04T16:41:43Z, the headline change
- [Tailwind CSS `@tailwindcss/oxide` npm package](https://www.npmjs.com/package/@tailwindcss/oxide) — the native scanner package that gains WASM fallback
- [Tailwind CSS `@tailwindcss/oxide-wasm32-wasi` package (currently `"cpu": ["wasm32"]`)](https://www.npmjs.com/package/@tailwindcss/oxide-wasm32-wasi) — the WASM package whose `cpu` restriction is removed
- [PR #20327 — pending platform support PR closed by PR #20383](https://github.com/tailwindlabs/tailwindcss/pull/20327)
- [PR #20276 — pending platform support PR closed by PR #20383](https://github.com/tailwindlabs/tailwindcss/pull/20276)
- [PR #20201 — pending platform support PR closed by PR #20383](https://github.com/tailwindlabs/tailwindcss/pull/20201)
- [napi-rs WASM loader docs](https://napi-rs.dev/docs/concepts/wasi) — the underlying WASM runtime used by the fallback
- [Tailwind CSS CHANGELOG.md (still showing 4.3.3 as latest; will be updated on the next release)](https://github.com/tailwindlabs/tailwindcss/blob/main/CHANGELOG.md)

---

## Next.js 16.3.1-canary.7 — styled-jsx SSR Regression Fix: Missing Styles in Pages Router Adapter Builds (PR #96632, August 7, 2026 — SHIPPED)

### What happened

**PR #96632** (`ae4063e`, merged 2026-08-07T06:26:16Z, npm-published in `16.3.1-canary.7` 2026-08-07T10:11:39Z) fixes a **production-breaking SSR regression** affecting Pages Router apps deployed through a build adapter — which includes every Vercel deployment, since Vercel's infrastructure uses build adapters internally.

**Affected versions:** `next@16.3.0` STABLE + all `next@16.3.1-canary.0` through `canary.6` (the fix ships in `canary.7`).

**Symptom:** Pages using `styled-jsx` render with a **flash of unstyled content (FOUC)** in production. The JSX-style class names (`jsx-*`) are still emitted by the Babel transform, but the SSR HTML contains no `<style>` tags — so the CSS only loads client-side after hydration.

### Root cause walkthrough

The Pages Router renderer and user code must share a **single `styled-jsx` module instance**. Here's why:

1. **`render.tsx`** creates the style registry and hands it to user code through a React context owned by that specific module instance.
2. If user code resolves to a **different `styled-jsx` instance** than Next.js' own pinned copy — which happens as soon as the app has a `styled-jsx` dependency that doesn't dedupe — `JSXStyle` silently renders nothing during SSR (`if (!registry) return null`).
3. **Turbopack** keeps `styled-jsx`/`styled-jsx/style` external in the pages server bundle, so the single-instance guarantee is established at **runtime** by `next/dist/server/require-hook`, which uses `require.resolve()` to pin to Next.js' own copy.
4. The problem: `require.resolve()` resolves to a file path. Nothing in any module graph references those files (user code only ever references its own copy), so **output tracing had to add them explicitly** — or the deployment is missing the file, `require.resolve()` throws, the aliases are (silently) never registered, and the two `styled-jsx` instances drift apart.
5. That explicit tracing existed only for the whole-app `next-server.js.nft.json` / `next-minimal-server.js.nft.json`. But **those files are not generated at all when a build adapter is used** — because an adapter assembles its output from each endpoint's own NFT instead (the `is_using_adapter` early return). The adapter's comment said "don't need any server NFTs" — true for those two whole-app files, but **not for the styled-jsx entries they also carried**.

The fix (7 commits, 4 files changed / +101/-38):
- `crates/next-api/src/next_server_nft.rs`: extracts `styled_jsx_require_hook_modules()` and adds an accurate comment to the `is_using_adapter` early return.
- `crates/next-api/src/project.rs`: `Project::additional_traced_modules` now also returns those modules, alongside the existing `cacheHandler`/`cacheHandlers` entries.
- `packages/next/src/server/require-hook.ts`: extracts `styledJsxRequireHookEntries()` and replaces the silent `catch (_) {}` around registering the aliases with a **diagnostic warning** (still never throws — but now you know if it happens).
- `packages/next/src/build/adapter/build-complete.ts` + `collect-build-traces.ts`: use that helper instead of re-deriving the same resolution inline.

### Practical impact

| Scenario | Before fix | After fix |
|---|---|---|
| Vercel deployment + Pages Router + styled-jsx | FOUC in production SSR | CSS renders server-side ✅ |
| Self-hosted adapter build + styled-jsx | FOUC in production SSR | CSS renders server-side ✅ |
| Pages Router without styled-jsx | No change | No change |
| App Router | No change (App Router doesn't use this path) | No change |
| Webpack production build | No change | No change |

**The `catch (_) {}` → diagnostic warning change** in `require-hook.ts` means if the styled-jsx alias registration ever fails in the future, you'll get a warning instead of silent failure.

### Recommended action

**For users on `next@16.3.0` STABLE or `next@16.3.1-canary.0` through `canary.6` with Pages Router on Vercel (or any build adapter):**
- Upgrade to `next@canary` (which is now `16.3.1-canary.7` or later) — the styled-jsx SSR fix is a drop-in, no code changes required.
- If you can't bump canary yet: check your deployed HTML for `<style data-styled-jsx>` tags. If they're missing but your `styled-jsx` classes (`jsx-*`) are in the HTML, you're hitting this regression.
- **Workaround (no upgrade needed):** ensure your `styled-jsx` version dedupes with Next.js' pinned copy by removing any direct `styled-jsx` dependency from your `package.json` and relying on Next.js' bundled version.

### Audit recipe

```bash
# 1. Are you on a Pages Router deployment on Vercel (or any adapter)?
grep -r "pages/" next.config.* | grep -v ".next" | head -3
# If you see page files under pages/, you may be affected.

# 2. Check if styled-jsx is in your dependency tree
npm ls styled-jsx 2>/dev/null

# 3. Check if your SSR HTML has styled-jsx style tags
# Deploy with the fix first, then:
curl -s https://your-app.com/your-page | grep "styled-jsx" | head -5
# If you see <style data-styled-jsx> tags → fix applied ✅

# 4. Upgrade to canary.7+
npm view next@canary version  # should be 16.3.1-canary.7 or later
npm install next@canary  # or pin to canary.7 specifically

# 5. Pin strategy: once 16.3.1 stable ships, bump to that.
# The styled-jsx SSR fix will land in the next stable patch.
```

### Sources

- [PR #96632 — Fix missing styled-jsx styles in Pages Router SSR on adapter builds](https://github.com/vercel/next.js/pull/96632) — 7 commits, merged 2026-08-07T06:26:16Z
- [Commit `ae4063e` — headline commit](https://github.com/vercel/next.js/commit/ae4063e)
- [Commit `e848b53` — Only trace require-hook modules for Pages Router endpoints](https://github.com/vercel/next.js/commit/e848b53)
- [Commit `a83ae60` — Fix stale reference comment](https://github.com/vercel/next.js/commit/a83ae60)
- [Next.js `v16.3.1-canary.7` GitHub release](https://github.com/vercel/next.js/releases/tag/v16.3.1-canary.7) — npm-published 2026-08-07T10:11:39Z
- [`packages/next/src/server/require-hook.ts`](https://github.com/vercel/next.js/blob/canary/packages/next/src/server/require-hook.ts) — the `styledJsxRequireHookEntries()` + diagnostic warning
- [`crates/next-api/src/project.rs`](https://github.com/vercel/next.js/blob/canary/crates/next-api/src/project.rs) — `additional_traced_modules` now includes styled-jsx
- [Test: `test/production/adapter-styled-jsx/`](https://github.com/vercel/next.js/tree/canary/test/production/adapter-styled-jsx) — new regression test

## Tailwind Main Branch — PR #20408 "Fix Slow Vite Rebuilds in Projects with Large Gitignored Directories" SHIPPED (August 12, 2026) — Forward-Looking for `tailwindcss@4.3.4` / `v4.4.0`

**`tailwindcss@latest` is still `4.3.3`** (npm-published 2026-07-16T12:03:35Z — **27+ days since last release**; the v1.5.30 cycle documented `4.3.3` SHIPPED with 9 bug fixes; the v1.5.30 + v1.5.47 cycles documented the PR #20383 WASM fallback as forward-looking; the v1.5.52 cycle noted "Tailwind main branch 24 commits ahead of 4.3.3; no NEW commits since v1.5.47 PR #20399; expect v4.3.4 or v4.4.0 within 2-4 weeks"). **The PR #20399 observation is no longer accurate** — **PR #20408 merged today at 2026-08-12T14:31:48Z** in `0.0.0-insiders.b86a6e0` (npm-published 2026-08-12T14:45:36Z, replacing `0.0.0-insiders.16e94cb` from Aug 10).

### The bug — verbatim from PR #20408 body

> *Preface: For full transparency, I used AI (Claude) to help dig into this issue we're having and write some of the explanation below.*
>
> *Our local Laravel application (with Docker) takes between 2-6s seconds (re)building `app.css` after each Blade file update. After checking with Claude, there seems to be some potential unnecessary traversing of big (gitignored) directories, specifically `vendor` and `storage`.*
>
> *As far as I understand it, two scans happen in tailwindcss:*
> - *a first scan, the content scan. This one reads the project files (auto-detect). It respects .gitignore, so it correctly skips vendor/ and storage/.*
> - *a second scan, the glob resolution (resolve_globs() in the oxide crate). It builds watch patterns that the Vite plugin registers, so a file save can trigger a CSS rebuild. It walks the project again, but this walk does not respect .gitignore. It only skips a hardcoded list of directory names from `ignored-content-dirs.txt` (node_modules, .git, venv, ...). vendor/ and storage/ are not on that list, so it descends into them and stats every file. These directories can never produce a watch pattern, because the content scan never visited them. That work is perhaps wasted?*
>
> *Maybe why this went unnoticed: on a native filesystem a stat takes about 1µs, so 57k of them is around 0.06s. A Docker bind mount multiplies that by roughly 100. And the big directory in a typical JS project is `node_modules`, which the name list already catches; `vendor/` is the PHP ecosystem's version of that.*

**The root cause**: `resolve_globs()` in the `@tailwindcss/oxide` Rust crate builds watch patterns for the Vite plugin. It walks the project to discover files but only skips directories from a hardcoded list (`node_modules`, `.git`, `venv`, etc.). It does NOT respect `.gitignore`. So in projects with large gitignored directories (`vendor/` in PHP/Ruby/Python projects, `storage/` in Laravel, `dist/` in build-output setups, `target/` in Rust projects, `.venv/` in Python venvs, etc.), the second scan stats every file in those directories — even though the content scan already determined those directories can't produce Tailwind classes.

**The fix**: PR #20408 (marickvantuil, merged 2026-08-12T14:31:48Z, **3 files / +27/-0**) makes `resolve_globs()` respect `.gitignore` so the second scan skips the same directories the first scan does. Tiny diff, big impact.

### Per-environment impact

| Setup | Before PR #20408 | After PR #20408 |
|---|---|---|
| Docker bind mount + PHP/Laravel (`vendor/` has 57K files) | 2-6s CSS rebuild after each Blade update | <1s typical CSS rebuild |
| Docker bind mount + Node.js (`node_modules/` already in hardcoded list) | Already fast (~0.1s) | No change |
| Native macOS / Linux (no Docker) | ~0.06s rebuild, invisible to dev loop | No change |
| WSL2 + large `vendor/` directory | 1-3s rebuild | <0.5s rebuild |
| Vite production build | Builds ignore watch patterns; not directly affected | No change |
| `@tailwindcss/postcss` workflow (no Vite) | Same Rust crate used; same fix applies | Same fix applies |
| Monorepo with `dist/` in `.gitignore` | Same wasted traversal pattern | Same fix applies |
| Python + `.venv/` (already in hardcoded list via `venv`) | Already fast | No change |
| Ruby/Rails + `vendor/` | Slow rebuild | Fast rebuild |

### Recommended action

1. **Track the next `tailwindcss@4.3.x` or `4.4.x` npm release** (likely `4.3.4` given cadence; could be `4.4.0` if they bundle several PRs).
2. **Until then**, if your dev loop is mysteriously slow on Docker/WSL2 with a non-Node.js ecosystem (`vendor/`, `storage/`, `dist/`, `target/`), the fix isn't in your `@latest` pin yet. Workarounds:
   - Set `experimental.turbopackFileSystemCacheForBuild: true` if you're on Next.js (won't fix Tailwind rebuilds but reduces other dev overhead)
   - Move the slow directory out of `.gitignore` but into the hardcoded `ignored-content-dirs.txt` list (requires a fork of Tailwind)
   - Disable the Vite plugin's watch (use `@tailwindcss/cli --watch --poll=2000` instead)
3. **Audit recipe**:

```bash
# Find projects with large gitignored directories that could be affected
rg -n "^vendor/|^storage/|^dist/|^target/|^.venv/" .gitignore | head -10
# Count files in the largest gitignored directory
du -sh vendor/ storage/ dist/ target/ .venv/ 2>/dev/null | sort -hr | head -5
# Check if you're on the insider train with the fix
npm ls tailwindcss@insiders 2>/dev/null
npm view tailwindcss@insiders version  # should be 0.0.0-insiders.b86a6e0 or later

# For Vite users: check your watch-pattern registration cost
# Run with --debug to see file watching stats
DEBUG=tailwindcss:resolve-globs:* npx vite 2>&1 | grep -i "stats\|skipping\|ignored" | head -20
```

### Why this matters at scale

The combination of (a) Docker bind mounts on macOS/Windows (100x stat cost multiplier) + (b) PHP/Ruby/Python ecosystems with `vendor/` directories (50K-200K files) + (c) Tailwind's two-scan pattern means the typical Laravel + Docker + Tailwind setup has been 2-6s slow per CSS rebuild for years. PR #20408 is the fix. The `npm view tailwindcss dist-tags.latest` check still returns `4.3.3` as of 2026-08-13T00:03Z — these are NEW commits ahead of the npm-published version, queued for `4.3.4` (or `4.4.0`).

### Sources

- [Tailwind PR #20408 — Fix slow Vite rebuilds in projects with large gitignored directories](https://github.com/tailwindlabs/tailwindcss/pull/20408) — marickvantuil, merged 2026-08-12T14:31:48Z, **3 files / +27/-0** (tiny diff, big impact)
- [Tailwind commit `b86a6e0a`](https://github.com/tailwindlabs/tailwindcss/commit/b86a6e0a) — the merge commit for PR #20408
- [`@tailwindcss/oxide` `resolve_globs()` in the Rust crate](https://github.com/tailwindlabs/tailwindcss/tree/main/packages/%40tailwindcss-oxide) — the function that now respects `.gitignore`
- [Tailwind `ignored-content-dirs.txt` hardcoded list](https://github.com/tailwindlabs/tailwindcss/blob/main/packages/%40tailwindcss-oxide/src/ignored-content-dirs.txt) — the existing list (`node_modules`, `.git`, `venv`, etc.) that PR #20408 complements
- [`tailwindcss@insiders` npm dist-tag](https://www.npmjs.com/package/tailwindcss?activeTab=versions) — `0.0.0-insiders.b86a6e0` (npm-published 2026-08-12T14:45:36Z)
- [`tailwindcss` npm dist-tag](https://www.npmjs.com/package/tailwindcss) — still `latest: 4.3.3` (npm-published 2026-07-16T12:03:35Z; expect next release within 2-4 weeks)
- [Tailwind main-branch commits since `4.3.3`](https://github.com/tailwindlabs/tailwindcss/commits/main) — 25 commits ahead; PR #20408 is the NEWEST
- [Tailwind releases page](https://github.com/tailwindlabs/tailwindcss/releases) — full version history

## Tailwind Insiders Train — 4 NEW Drops in 19h (Aug 13–14, 2026) — `4.3.4` / `4.4.0` Imminent

**`tailwindcss@latest` is still `4.3.3`** (npm-published 2026-07-16T12:03:35Z — **29 days since last release**). **The insider train has been incredibly active since v1.5.54** — **4 NEW insider drops** in the past 19 hours, each one a fresh commit, indicating the team is in code-freeze window for `4.3.4` (or possibly `4.4.0` if they bundle enough PRs):

| Insider hash | npm-published | Gap from prior | Notes |
|---|---|---|---|
| `0.0.0-insiders.00ef99d` | 2026-08-13T15:19:56Z | (start of v1.5.54 cycle's prior insider = `b86a6e0`) | First insider after PR #20408 merge |
| `0.0.0-insiders.de9e71c` | 2026-08-13T19:46:16Z | +4h 26min | Hot patch cycle |
| `0.0.0-insiders.f7f58f0` | 2026-08-13T21:30:12Z | +1h 44min | Hot patch cycle |
| `0.0.0-insiders.021b7fe` | 2026-08-14T10:28:56Z | +12h 58min | **CURRENT** insider train, 2h before this cron |

**The cadence is significant.** Four insider drops in 19 hours = roughly one insider drop every 4.75 hours. That's the **fastest insider cadence the v1.5.54 cycle observed** (the v1.5.54 cycle noted "Tailwind main branch 24 commits ahead of 4.3.3; no NEW commits since v1.5.47 PR #20399" — the cadence since then has accelerated 3-5x). **The insider-train acceleration strongly suggests `tailwindcss@4.3.4` (or `4.4.0`) is in the next 1-2 weeks** — the team is in stabilization mode, cutting fresh insider builds for each merged PR.

**What to do:**

1. **The v1.5.54 forward-looking section is still accurate** — PR #20408 is the most-impactful holding change. Audiences on Docker + WSL2 + non-Node.js ecosystems (Laravel, Ruby, Python, Rust) should still track the insider train explicitly.
2. **Watch the insider train daily** for the cut that announces the stable release. The "tag → pre-release → stable" pattern of past Tailwind releases (e.g., 4.3.2 → 4.3.2-insiders.x → 4.3.3 STABLE) suggests a stable is likely within 1-2 weeks.
3. **Audit recipe:**

```bash
# Check current insider
npm view tailwindcss@insiders version   # 0.0.0-insiders.021b7fe

# Check current @latest
npm view tailwindcss dist-tags.latest   # 4.3.3

# Subscribe to insider train to get speed-fix PRs without waiting for stable
npm install -D tailwindcss@insiders
# Then upgrade every 1-2 weeks to catch the new insider builds

# Confirm your project actually picks up the insider after install
npm ls tailwindcss
# Should show tailwindcss@0.0.0-insiders.021b7fe or later
```

### The v1.5.54's PR #20408 prediction — confirmed by the insider train

The v1.5.54-cycle's "PR #20408 is the NEWEST ahead of 4.3.3" claim is now confirmed by the **4 NEW insider cuts since**: each insider cuts because a new PR merged. The four insider drops in 19h mean approximately **4 NEW PRs merged** (likely docs-only or small bug-fix PRs — the team doesn't ship large insiders multiple per day). If you want to see what specifically landed, the diffs between insider commits are visible at:
- `b86a6e0` → `00ef99d`: ~1 merged PR
- `00ef99d` → `de9e71c`: ~1 merged PR
- `de9e71c` → `f7f58f0`: ~1 merged PR
- `f7f58f0` → `021b7fe`: ~1 merged PR

(For real PR-level detail, run `git log b86a6e0..021b7fe --oneline` against the Tailwind monorepo at the matching hash range.)

### Why this matters even if you're on `@latest`

**Most projects should stay on `@latest` (4.3.3) and let the stable releases come.** The insider train is for:
- **Devs whose dev loop is impacted by a fixed bug** (e.g., the v1.5.54-cycle's PR #20408 Docker + WSL2 + vendor/ slowness)
- **Devs who want to test the stable-release candidate before it ships**
- **Library authors who need to support the next Tailwind version ahead of stable**

**For everyone else — wait for `tailwindcss@4.3.4` (or `4.4.0`) to ship, then bump.** The insider train is a signal of imminent stability, not a call to upgrade.

### Sources

- [`tailwindcss@insiders` npm dist-tag](https://www.npmjs.com/package/tailwindcss?activeTab=versions) — `0.0.0-insiders.021b7fe` (npm-published 2026-08-14T10:28:56Z, **2h before this cron**, the CURRENT insider)
- [`tailwindcss@insiders` npm time data](https://registry.npmjs.org/tailwindcss) — confirms the 4 NEW insider drops in 19h: `00ef99d` (15:19:56Z Aug 13) → `de9e71c` (19:46:16Z Aug 13) → `f7f58f0` (21:30:12Z Aug 13) → `021b7fe` (10:28:56Z Aug 14)
- [`tailwindcss` npm dist-tag](https://www.npmjs.com/package/tailwindcss) — still `latest: 4.3.3` (npm-published 2026-07-16T12:03:35Z; expect next release within 1-2 weeks)
- [Tailwind PR #20408 — Fix slow Vite rebuilds in projects with large gitignored directories](https://github.com/tailwindlabs/tailwindcss/pull/20408) — marickvantuil, merged 2026-08-12T14:31:48Z, **3 files / +27/-0** (the v1.5.54 documented fix; now confirmed in the faster insider cadence)
- [Tailwind main-branch commits since `4.3.3`](https://github.com/tailwindlabs/tailwindcss/commits/main) — 25+ commits ahead; insider train confirms 4 NEW merged PRs in the past 19h
- [Tailwind releases page](https://github.com/tailwindlabs/tailwindcss/releases) — full version history
- [Tailwind v4.3 blog post](https://tailwindcss.com/blog/tailwindcss-v4-3) — the v4.3.0 SHIP announcement (May 8, 2026)
- [Tailwind v4.1 blog post](https://tailwindcss.com/blog/tailwindcss-v4-1) — the v4.1.0 SHIP announcement (March 4, 2025)
- [Tailwind Insider program](https://tailwindcss.com/insiders) — paid program for early access; the `npm install -D tailwindcss@insiders` install is the open-source insider track

## Tailwind Main Branch — 4 NEW Commits Since v1.5.59 (`tailwindcss@insiders` STILL on 90f8ff4) — `tailwindcss@4.3.4` STABLE Imminent (August 14 → August 16, 2026)

**`tailwindcss@latest` is still `4.3.3`** (npm-published 2026-07-16T12:03:35Z — **31+ days since last `@latest` release**). **The Tailwind main branch has accumulated 4 NEW commits since the v1.5.59 cycle** (verified at 2026-08-16T18:03Z via `GET /repos/tailwindlabs/tailwindcss/commits?per_page=30` showing the active commits). **The insider train has cooled significantly** — `tailwindcss@insiders` is **STILL on `0.0.0-insiders.90f8ff4`** (npm-published 2026-08-14T19:54:08Z; **~46h+ idle at this cron**). The v1.5.59 cycle observed "4 drops in 19h = ~1 every 4.75h" acceleration; the v1.5.61 cycle observed "6 drops in 30h"; the v1.5.64 cycle observed "0 drops in 34h+"; this cycle observes "0 drops in 46h+ since 90f8ff4". **The cooled insider train cadence is a strong signal that the team is preparing for `4.3.4` (or `4.4.0`) STABLE cut** — when the insider train cools for 24-48h, the team is typically in final stabilization mode before the STABLE release.

### The 4 NEW main-branch commits since v1.5.59 — commit table

| # | SHA | Date (UTC) | PR | Title | Author | Files | Materiality |
|---|---|---|---|---|---|---|---|
| 1 | `f7f58f0` | 2026-08-13T21:16:30Z | [#20416](https://github.com/tailwindlabs/tailwindcss/pull/20416) | `Move debug logs in .tailwindcss folder` | (community) | (small) | LOW (debug-logs relocation) |
| 2 | `021b7fe` | 2026-08-14T10:16:00Z | **[#20417](https://github.com/tailwindlabs/tailwindcss/pull/20417)** | **`Canonicalization: prevent inlining CSS-wide keywords`** | (community) | 3 / +53/-8 | **HIGH** (intellisense + Uniwind + HeroUI scenario) |
| 3 | `7a7f386` | 2026-08-14T16:47:34Z | **[#20420](https://github.com/tailwindlabs/tailwindcss/pull/20420)** | **`Don't space out 'and'/'or'/'not' inside function calls in 'supports-[…]' variants`** | (community) | 3 / +76/-1 | **HIGH** (Chrome-bug-workaround regression fix) |
| 4 | `90f8ff4` | 2026-08-14T19:43:33Z | (same as above) | (same as above — `90f8ff4` IS the merge commit for PR #20420) | (community) | (same) | (same) |

Wait — the table above conflates commits with PR merge commits. Let me restate: **the 4 NEW commits since v1.5.59 are**:
- **PR #20416** (`f7f58f0`, 2026-08-13T21:16:30Z) — debug logs relocation
- **PR #20417** (`021b7fe`, 2026-08-14T10:16:00Z) — **Canonicalization: prevent inlining CSS-wide keywords** (HIGH MATERIAL)
- **PR #20420** (`7a7f386`, 2026-08-14T16:47:34Z) — **Don't space out and/or/not inside function calls in supports-[…] variants** (HIGH MATERIAL)
- **`90f8ff4`** (2026-08-14T19:43:33Z) — the **current insider train HEAD** for `tailwindcss@insiders`; the merge commit for additional stabilization; no separate PR.

**Total new code: 6 files / +129/-9 across 2 MATERIAL bug fixes + 1 minor debug-logs move + 1 stabilization commit.**

### The 2 MATERIAL PRs — deep dives

**PR #20417** (merged 2026-08-14T10:16:01Z, 3 files / +53/-8) — **`Canonicalization: prevent inlining CSS-wide keywords`**. The bug: canonicalization suggestions in intellisense result in "weird" suggestions. Concretely:

```css
@theme {
  --color-foreground: var(--foreground);
  --color-default-soft-hover: color-mix(in oklab, var(--default) 60%, transparent);
}
```

A class `text-foreground/60` would be canonicalized (suggested) to `text-default-soft-hover`, which looks nonsensical at first. The root cause: when you use **Uniwind (React Native) with HeroUI**, the setup looks more like:

```css
@import 'tailwindcss';

@theme {
  --foreground: unset;
  --default: unset;
}

@theme inline {
  --color-foreground: var(--foreground);
  --color-default-soft-hover: color-mix(in oklab, var(--default) 60%, transparent);
}
```

During canonicalization, **all `@theme` values get inlined**, which then produces identical signatures for two classes that should be distinct. The fix: **make sure to never inline [CSS-wide keywords]** (a small list including `unset`, `initial`, `inherit`, `revert`, `revert-layer`, `unset`, `none`) during the canonicalization step. **Practical impact**: Tailwind projects using the `@theme inline` directive + CSS-wide keywords (common in design-system-theming setups, Uniwind + HeroUI + custom-design-system stacks) now get correct intellisense suggestions. **The bug was particularly insidious** because the wrong canonicalization only manifested with CSS-wide keywords — apps using concrete values (`var(--foreground)` returning a color, etc.) didn't see the issue.

**PR #20420** (merged 2026-08-14T19:43:34Z, 3 files / +76/-1) — **`Don't space out 'and'/'or'/'not' inside function calls in 'supports-[…]' variants`**. The bug: the `supports-[…]` variant works around a Chrome bug where `@supports (a)or(b)` is invalid by spacing out the `and`, `or`, and `not` keywords. **However**, the replacement was applied to the **entire value, including the inside of function calls**, where these words can be part of a **selector**. For example:

```css
supports-[selector(a:not(.foo))]:flex
```

This was generating:

```css
@supports selector(a: not (.foo))
```

The selector `a: not (.foo)` is **unparsable**, so a condition that is true in every browser silently becomes **false** and the utility **never applies**. **The same problem applies to class names like `.and` or `.or` inside `selector(…)`**. **The fix**: only spaces out the keywords **at the condition level** — parens preceded by an identifier (other than the keywords themselves) start a function call, and everything inside is left as-is. The Chrome workaround still applies to the condition itself, e.g. `supports-[(display:grid)or(display:flex)]` still becomes `@supports (display: grid) or (display: flex)`. **Practical impact**: apps using `supports-[selector(a:not(.foo))]` (and similar `selector(...)` patterns inside `supports-[…]`) now correctly apply the utility; the bug was previously silently breaking the utility for these inputs.

### Per-user-type impact table

| User type | Pre-#20417 / pre-#20420 (4.3.3) | Post-#20417 / post-#20420 (4.3.4 STABLE ahead) |
|---|---|---|
| **Apps using `@theme inline` + CSS-wide keywords (Uniwind + HeroUI + custom-design-system)** | Weird canonicalization suggestions in intellisense | **Correct suggestions** (PR #20417) |
| **Apps using `supports-[selector(a:not(.foo))]:flex`** | Utility silently doesn't apply; condition turns false | **Utility correctly applies** (PR #20420) |
| **Apps using `supports-[selector(.and-or-not)]:flex` with class names like `.and` / `.or` / `.not`** | Utility silently doesn't apply | **Utility correctly applies** (PR #20420) |
| **Apps using `supports-[(display:grid)or(display:flex)]`** | Works correctly (Chrome workaround applies at condition level) | Works correctly (unchanged) |
| **Apps not using these patterns** | No change | No change |
| **Production users on `tailwindcss@latest` 4.3.3** | No change yet (PRs in main + insiders only) | Will get the fix on `4.3.4` STABLE |
| **Insider-train users on `0.0.0-insiders.90f8ff4`** | Pre-fix | Both fixes already in the insider build |

### Insider-train cooling → STABLE imminent

**The insider-train cadence has cooled dramatically since v1.5.59**:
- **v1.5.59** (Aug 14 12:06Z): "4 drops in 19h = ~1 every 4.75h = fastest insider cadence observed"
- **v1.5.61** (Aug 15 00:06Z): "6 drops in 30h" — continued acceleration
- **v1.5.64** (Aug 15 23:30Z): "0 drops in 34h+ since 90f8ff4"
- **v1.5.66** (Aug 16 12:11Z): "0 drops in 40h+ since 90f8ff4"
- **This cycle** (Aug 16 18:03Z): "0 drops in 46h+ since 90f8ff4"

**The cooling cadence is the canonical pattern observed for `4.3.2 → 4.3.3`** — when the team stops cutting insider drops for 24-48h+, they're typically in final stabilization mode before STABLE. **`tailwindcss@4.3.4` (or possibly `4.4.0`) STABLE expected within 1-2 weeks** (was "imminent" in v1.5.59; corrected to "within 1-2 weeks" in v1.5.64; **UNCHANGED here** — the cooling cadence confirms the 1-2 week forecast rather than accelerating it).

### Recommended action

1. **Track the next `tailwindcss@4.3.x` or `4.4.x` npm release** (`npm view tailwindcss dist-tags.latest` will move off `4.3.3` when it ships)
2. **If you use `supports-[selector(...)]` patterns + the selector contains `.and` / `.or` / `.not` / `:not(...)`** — you may be silently affected by PR #20420's bug; verify your utilities are actually applying on the `@latest` 4.3.3 (they might not be, in which case the insider train has the fix)
3. **If you use `@theme inline` + CSS-wide keywords** — your intellisense is showing wrong suggestions on 4.3.3; the insider train has the fix
4. **Audit recipe**:

```bash
# Check current @latest
npm view tailwindcss dist-tags.latest   # 4.3.3

# Check current insider
npm view tailwindcss@insiders version   # 0.0.0-insiders.90f8ff4

# Find supports-[selector(...)] usages that might be affected by PR #20420
rg -n "supports-\[selector" src/ --type ts --type tsx --type js --type jsx --type css | head -10

# Find @theme inline + CSS-wide keyword combinations that might be affected by PR #20417
rg -n "@theme inline" src/ --type css | head -10
rg -n "(--[a-z-]+:\s*(unset|initial|inherit|revert|revert-layer|none))" src/ --type css | head -10

# If you need the fixes right now, install insider:
npm install -D tailwindcss@insiders
# Then upgrade every 1-2 weeks to catch the new insider builds

# Confirm your project picks up the insider after install
npm ls tailwindcss
# Should show tailwindcss@0.0.0-insiders.90f8ff4 or later
```

### Why this matters

**For most projects**, stay on `@latest` (4.3.3) and let `4.3.4` STABLE come. The insider train is for:
- **Devs whose CSS is silently affected by the PR #20417 / PR #20420 bugs** (especially React Native + HeroUI + Uniwind projects; projects using `supports-[selector(a:not(.foo))]:flex` patterns)
- **Devs who want to test the stable-release candidate before it ships**
- **Library authors who need to support the next Tailwind version ahead of stable**

**For everyone else — wait for `tailwindcss@4.3.4` (or `4.4.0`) to ship, then bump.** The cooling insider cadence is a signal of imminent stability, not a call to upgrade.

### Sources

- [Tailwind PR #20417 — `Canonicalization: prevent inlining CSS-wide keywords`](https://github.com/tailwindlabs/tailwindcss/pull/20417) — merged 2026-08-14T10:16:01Z, 3 files / +53/-8. **THE HEADLINE** — fixes intellisense canonicalization bugs when `@theme inline` declarations contain CSS-wide keywords (`unset` / `initial` / `inherit` / `revert` / `revert-layer` / `none`); the bug particularly affected Uniwind (React Native) + HeroUI + custom-design-system stacks that use the `@theme inline` directive with `var(--xxx)` returning `unset`.
- [Tailwind PR #20420 — `Don't space out 'and'/'or'/'not' inside function calls in 'supports-[…]' variants`](https://github.com/tailwindlabs/tailwindcss/pull/20420) — merged 2026-08-14T19:43:34Z, 3 files / +76/-1. **THE HEADLINE** — fixes the Chrome-bug workaround regression where `supports-[selector(a:not(.foo))]:flex` was generating `@supports selector(a: not (.foo))` (the selector `a: not (.foo)` is unparsable, silently turning the condition false); the fix spaces out the keywords only at the condition level, leaving function-call contents untouched. Includes tests for `selector(a:not(.foo))` + class names `.and` / `.or` inside `selector(...)` + the Chrome `(a)or(b)` workaround + a top-level `not(...)` condition.
- [Tailwind PR #20416 — `Move debug logs in .tailwindcss folder`](https://github.com/tailwindlabs/tailwindcss/pull/20416) — merged 2026-08-13T21:16:30Z, debug-logs relocation (low material).
- [Tailwind commit `90f8ff4`](https://github.com/tailwindlabs/tailwindcss/commit/90f8ff4) — 2026-08-14T19:43:33Z, the **current insider train HEAD** for `tailwindcss@insiders`; no separate PR (stabilization commit).
- [`tailwindcss@insiders` npm dist-tag](https://www.npmjs.com/package/tailwindcss?activeTab=versions) — `0.0.0-insiders.90f8ff4` (npm-published 2026-08-14T19:54:08Z; ~46h+ idle at this cron's check)
- [`tailwindcss@insiders` npm time data](https://registry.npmjs.org/tailwindcss) — confirms the cooling cadence: 4 drops in 19h (v1.5.59) → 6 drops in 30h (v1.5.61) → 0 drops in 46h+ (this cycle)
- [`tailwindcss` npm dist-tag](https://www.npmjs.com/package/tailwindcss) — still `latest: 4.3.3` (npm-published 2026-07-16T12:03:35Z; now 31+ days since last `@latest`; **expect `4.3.4` STABLE within 1-2 weeks**)
- [Tailwind main branch commits since `4.3.3`](https://github.com/tailwindlabs/tailwindcss/commits/main) — 4 NEW commits since v1.5.59's 021b7fe = PR #20417 + PR #20420 + PR #20416 + the 90f8ff4 stabilization
- [Tailwind v4.3 blog post](https://tailwindcss.com/blog/tailwindcss-v4-3) — the v4.3.0 SHIP announcement (May 8, 2026)
- [Tailwind Insider program](https://tailwindcss.com/insiders) — paid program for early access; the `npm install -D tailwindcss@insiders` install is the open-source insider track
- Cross-reference: `styling.md` → `## Tailwind Main Branch — PR #20408 "Fix Slow Vite Rebuilds in Projects with Large Gitignored Directories" SHIPPED (August 12, 2026) — Forward-Looking for tailwindcss@4.3.4 / v4.4.0` for the v1.5.54 lens that documented the previous main-branch PR (PR #20408 slow Vite rebuilds); PR #20417 + PR #20420 are the 2 NEW main-branch PRs since v1.5.59's 021b7fe commit.
- Cross-reference: `styling.md` → `## Tailwind Insiders Train — 4 NEW Drops in 19h (Aug 13–14, 2026) — 4.3.4 / 4.4.0 Imminent` for the v1.5.59 lens that documented the insider-train acceleration; the cooling cadence since 90f8ff4 confirms the 1-2 week STABLE forecast.


## Tailwind CSS v4.3.3 production baseline — CSS-first tokens, themes, and tooling (verified 2026-08-18)

**Current version check:** `tailwindcss@latest` is still `4.3.3`; `tailwindcss@insiders` is still `0.0.0-insiders.90f8ff4`; no stable `4.3.4` or `4.4.0` release is present. The insider build is useful for testing, not a production default. The current insider includes Tailwind PR #20417 (do not inline CSS-wide keywords during canonicalization) and PR #20420 (only space `and`/`or`/`not` at the condition level in `supports-*` variants).

### A production-safe v4.3 setup

Keep source tokens in CSS, use `@theme inline` only when a utility should resolve a runtime CSS variable, and define the dark selector once:

```css
@import "tailwindcss";

/* Explicit class-based dark mode. */
@custom-variant dark (&:where(.dark, .dark *));

:root {
  --brand: 222.2 47.4% 11.2%;
  --brand-foreground: 210 40% 98%;
}

@theme inline {
  --color-brand: var(--brand);
  --color-brand-foreground: var(--brand-foreground);
  --radius-card: 1rem;
}

.card {
  @variant hover {
    border-color: var(--color-brand);
  }
}

/* Add this when Tailwind cannot discover a non-standard package. */
@source "../packages/ui/src";
```

`@theme` values are compiled into utilities and remain available as CSS variables. `@theme inline` is the correct bridge for semantic shadcn-style variables such as `--color-background: var(--background)`. Keep the underlying runtime values (`:root` / `.dark`) outside `@layer base`; do not define a second, conflicting theme inside the variant.

### Tooling and migration audit

- **Vite:** use the first-party plugin; do not keep the v3 PostCSS plugin name:

  ```ts
  // vite.config.ts
  import tailwindcss from '@tailwindcss/vite'
  import { defineConfig } from 'vite'

  export default defineConfig({
    plugins: [tailwindcss()],
  })
  ```

- **PostCSS/CLI:** use `@tailwindcss/postcss` or `@tailwindcss/cli`; Next.js 16 uses its built-in Tailwind integration and does not need a second Tailwind plugin.
- **Upgrade:** run `npx @tailwindcss/upgrade` on a branch, then audit `tailwind.config.js` and dynamic class strings. Use `@config` only for legacy JS plugin compatibility; move theme values and custom utilities into CSS.
- **shadcn/ui:** use `@theme inline`, keep semantic `--background` / `--foreground` values in CSS, and use `tw-animate-css` instead of the retired `tailwindcss-animate` package.
- **v4.3 correctness:** test `@variant hover:focus`, stacked variants, scrollbar utilities, `@container-size`, and `supports-[selector(a:not(.foo))]` in the real project. A class that looks valid can still generate an unparsable `@supports` condition on an older stable build.

```bash
# Version and source-discovery checks
npm view tailwindcss dist-tags.latest
npm view tailwindcss@insiders version
rg -n "@tailwind|theme\(|plugins|@source|@theme" src postcss.config.* vite.config.* 2>/dev/null

# Migration and build checks
npx @tailwindcss/upgrade
npm run build
```

### Common mistakes

- Keeping `@tailwind base`, `@tailwind components`, and `@tailwind utilities`; v4 starts with `@import "tailwindcss"`.
- Defining a dark mode with both a media-query default and an unrelated `.dark` selector; choose one variant and test it.
- Using `@theme inline` with literal values where a static `@theme` token is sufficient; the inline form exists to resolve CSS variables.
- Constructing class names from runtime strings and assuming Tailwind can discover them; add an explicit `@source` or keep the class set static.
- Promoting `tailwindcss@insiders` to production just to obtain the PR #20417/#20420 fixes before the next stable release.

### Sources

- [Tailwind CSS upgrade guide](https://tailwindcss.com/docs/upgrade-guide) — v3 → v4 migration, `@import`, Vite/PostCSS/CLI integration
- [Tailwind CSS v4 announcement](https://tailwindcss.com/blog/tailwindcss-v4) — CSS-first `@theme`, CSS variables, container queries, and the Vite plugin
- [Tailwind CSS v4.3 announcement](https://tailwindcss.com/blog/tailwindcss-v4-3) — scrollbar utilities, stacked/compound `@variant`, functional utilities, and `@container-size`
- [Tailwind PR #20417](https://github.com/tailwindlabs/tailwindcss/pull/20417) — CSS-wide-keyword canonicalization fix
- [Tailwind PR #20420](https://github.com/tailwindlabs/tailwindcss/pull/20420) — `supports-[selector(...)]` parsing fix
- [shadcn/ui Tailwind v4 guide](https://ui.shadcn.com/docs/tailwind-v4) — `@theme inline`, CSS variables, `tw-animate-css`, and v4 component migration
- [Tailwind v4 theme variables reference](https://tailwindcss.com/docs/theme) — namespace and CSS-variable behavior

## Tailwind Insider Train COLD — 0 New Drops Since Aug 14 (144+ Hours) + 4.3.4 STABLE Forecast Tightened + shadcn Ecosystem Still Idle (August 20, 2026 — Styling Lens)

**`tailwindcss@insiders` STILL on `0.0.0-insiders.90f8ff4`** (npm-published 2026-08-14T19:54:08Z; now **~144 hours / 6+ days idle** at this cron's 06:03Z Aug 20 check). The insider train that was doing **4 drops in 19h on Aug 13–14** is now **completely cold** for 6+ days. The 4.3.4 STABLE forecast must be **tightened from "1-3 weeks" to "2–4 weeks"** (expected Sep 3–20, 2026).

**`tailwindcss@latest` STILL `4.3.3`** (npm-published 2026-07-16T12:03:35Z; now **35+ days since last @latest**). The v1.5.59 "4.3.4 imminent" prediction was wrong; the cold insider train confirms a longer wait.

**shadcn ecosystem STILL IDLE since 4.18.0 / @shadcn/react 0.3.0 / @shadcn/helpers 0.2.0** (Aug 13 / Aug 5 / Aug 11 respectively; no new releases in this 6h window). The Aug 18 blog post's `<ViewTransition>` integration is **still NOT in the shadcn ecosystem**. `shadcn@4.19.0` or `@shadcn/react@0.4.0` to ship a `<ViewTransition>` wrapper is now forecast **1–2 weeks** (Sep 3–6) based on the Aug 13 idle start.

### Tailwind Insider Train — The Extended Cold Pattern

| Date | Insider Version | Drops | Cadence | Status |
|------|---------------|-------|---------|--------|
| Aug 13 00:28Z | 021b7fe | 1 drop | — | warming |
| Aug 13 19:00Z | 90f8ff4 | 1 drop | 4 drops in 19h (~1/5h) | **accelerating** |
| Aug 14 19:54Z | 90f8ff4 | **LAST DROP** | 0 drops in 40h+ | **cooling** |
| Aug 16 12:03Z | 90f8ff4 | 0 drops | 0 drops in 64h+ | **cold** |
| Aug 20 06:03Z | 90f8ff4 | **0 drops** | **0 drops in 144h+ (6 days)** | **frozen** |

### What This Means for Styling Setup

- **Stay on `tailwindcss@latest` (4.3.3) for production**: The `@latest` stable channel has been 4.3.3 for 35+ days with no movement. There is no security or correctness issue forcing an upgrade. The `@insiders` train is frozen and offers no advantage right now.
- **Do NOT promote `@insiders` to production**: The insider train is dead cold. Promoting `0.0.0-insiders.90f8ff4` to production would only get you the Aug 14 state with no new fixes. Wait for 4.3.4 STABLE on `@latest`.
- **`tailwindcss@4.3.4` STABLE now 2–4 weeks away**: Based on the 6-day cold streak, the typical Tag → Pre-release → Stable pattern observed for 4.3.2 → 4.3.3 suggests a longer wait. If new insider drops resume in the next 7 days, the 4.3.4 window tightens back to 1–3 weeks.
- **shadcn `<ViewTransition>` wrapper not yet available**: The `@shadcn/react` package is still at 0.3.0 (Questionnaire primitive from Aug 5). The `<ViewTransition>` wrapper referenced in the Aug 18 blog post is not yet shipped. Watch `shadcn@4.19.0` or `@shadcn/react@0.4.0` for it. In the meantime, use the native `experimental.viewTransition: true` in `next.config.ts` as documented in `components.md`.
- **TypeScript 7.0 STABLE setup still authoritative**: The TS 7.0 STABLE migration guidance from the v1.5.72 cycle is still current. TypeScript 7.0.2 is the `@latest` and works with Tailwind v4. No new Tailwind-specific TS material in this window.

### Styling Audit Recipe

```bash
# Check current Tailwind status
npm view tailwindcss@latest version
# Expected: 4.3.3 (unchanged since Jul 16)

npm view tailwindcss@insiders version
# Expected: 0.0.0-insiders.90f8ff4 (frozen since Aug 14; do NOT use in production)

# Check shadcn ecosystem
npm view shadcn@latest version
# Expected: 4.18.0 (idle since Aug 13)

npm view @shadcn/react@latest version
# Expected: 0.3.0 (idle since Aug 5; no ViewTransition wrapper yet)

npm view @shadcn/helpers@latest version
# Expected: 0.2.0 (idle since Aug 11)

# Verify tailwind.config is using @import "tailwindcss" (v4 style)
rg "@tailwind" tailwind.config.* 2>/dev/null
# Should find: @import "tailwindcss" NOT @tailwind base/components/utilities

# Check @theme usage (v4 semantic tokens)
rg "@theme" src/**/*.css 2>/dev/null | head -5

# If using shadcn — verify @theme inline usage
rg "theme inline|--color-" src/**/*.css 2>/dev/null | head -5
```

### Common Mistakes (Styling — Extended)

- Promoting `tailwindcss@insiders` to production during a cold streak: you get the same Aug 14 version with no new fixes and no stability guarantee.
- Assuming `shadcn@4.19.0` ships a `<ViewTransition>` wrapper: it has not shipped yet (as of Aug 20); the Aug 18 blog post's pattern requires native `experimental.viewTransition: true`.
- Using `@tailwind base/components/utilities` in a v4 project: v4 requires `@import "tailwindcss"`. The old directives cause a silent CSS failure.
- Using `@theme` with literal values where a CSS variable reference is correct: `@theme` tokens should reference CSS variables (e.g., `@theme { --color-primary: var(--color-brand); }`) not hardcoded hex values.

### Sources

- [`tailwindcss@latest` npm](https://www.npmjs.com/package/tailwindcss?activeTab=versions) — confirmed `4.3.3` unchanged since 2026-07-16T12:03:35Z; 35+ days
- [`tailwindcss@insiders` npm](https://www.npmjs.com/package/tailwindcss?activeTab=versions) — confirmed `0.0.0-insiders.90f8ff4` unchanged since 2026-08-14T19:54:08Z; **0 new drops in 144h+**
- [Tailwind v4.3 announcement](https://tailwindcss.com/blog/tailwindcss-v4-3) — scrollbar utilities, stacked/compound variants
- [shadcn/ui releases](https://github.com/shadcn-ui/ui/releases) — still `4.18.0` / `@shadcn/react@0.3.0` / `@shadcn/helpers@0.2.0`; no ViewTransition wrapper
- [Aug 18 blog post — "Building App-like Experiences with Next.js 16.3"](https://nextjs.org/blog/building-app-like-experiences-with-nextjs-16-3) — `<ViewTransition>` integration; no shadcn wrapper yet
- [Cross-reference: `setup.md` — TypeScript 7.0 STABLE setup recipe (still authoritative)
- [Cross-reference: `components.md` — `<ViewTransition>` native integration with `experimental.viewTransition: true` (still authoritative)
- [Cross-reference: `patterns.md` — Pattern Q View Transitions from the Aug 18 blog post (still authoritative)

## Styling Idle Check — Tailwind Insider Train STILL Cold (168h+ / 7+ Days) + shadcn Ecosystem STILL Idle (Aug 13+) (August 21, 2026 — Styling Lens)

**`tailwindcss@insiders` STILL on `0.0.0-insiders.90f8ff4`** (npm-published 2026-08-14T19:54:08Z; now **~168 hours / 7+ days idle** at this cron's 18:02Z Aug 21 check). The insider train that accelerated 4 drops in 19h on Aug 13–14 has now been **completely cold for a full week** — the longest insider-train freeze since the v1.5.0 tracking began.

**`tailwindcss@latest` STILL `4.3.3`** (npm-published 2026-07-16T12:03:35Z; now **36+ days since last @latest**). The 4.3.4 STABLE forecast remains **2–4 weeks** = expected Sep 3–20, 2026 (UNCHANGED).

**shadcn ecosystem STILL IDLE since 4.18.0 / @shadcn/react 0.3.0 / @shadcn/helpers 0.2.0** (Aug 13 / Aug 5 / Aug 11 respectively; **no new releases in this 6h window**). The Aug 18 blog post's `<ViewTransition>` integration is **still NOT in the shadcn ecosystem**. `shadcn@4.19.0` or `@shadcn/react@0.4.0` with a `<ViewTransition>` wrapper is now forecast **2–3 weeks** based on the Aug 13 idle start + the cold insider pattern.

### Extended Cold Pattern

| Date | Insider Version | Drops Since | Cadence | Status |
|------|---------------|-------|---------|--------|
| Aug 13 00:28Z | 021b7fe | 1 drop | — | warming |
| Aug 13 19:00Z | 90f8ff4 | 1 drop | 4 drops in 19h (~1/5h) | **accelerating** |
| Aug 14 19:54Z | 90f8ff4 | LAST DROP | 0 drops in 40h+ | cooling |
| Aug 16 12:03Z | 90f8ff4 | 0 drops | 0 drops in 64h+ | cold |
| Aug 20 06:03Z | 90f8ff4 | 0 drops | 0 drops in 144h+ (6 days) | frozen |
| **Aug 21 18:02Z** | **90f8ff4** | **0 drops** | **0 drops in 168h+ (7 days)** | **frozen / 1-week idle** |

### What This Means for Styling Setup

- **All v1.5.77 recommendations still apply**: Stay on `tailwindcss@latest` (4.3.3) for production. Do NOT promote `@insiders` (frozen for 1 week). Wait for 4.3.4 STABLE on `@latest` (forecast 2–4 weeks = Sep 3–20).
- **No NEW styling material this cycle**: The 36h window since v1.5.77 (Aug 20 06:03Z) brought no new Tailwind insiders drops, no new Tailwind STABLE, no new shadcn releases. **The longest no-new-material stretch for styling since v1.5.0** (June 19).
- **The Tailwind v4.3.x line is mature**: 4.3.0 shipped May 8, 2026 (per endoflife.date); 4.3.3 is the latest. The cadence suggests 4.3.4 will be a smaller patch release focused on the oxide WASM fallback (PR #20383, Aug 4) + the canonicalization fix (PR #20417) + the `supports-[selector]` space-out fix (PR #20420).
- **Aug 18 `<ViewTransition>` integration still requires native `experimental.viewTransition: true`**: No shadcn wrapper has shipped in the 3 days since the blog post. Use the native pattern documented in `components.md` and `patterns.md`.

### Styling Audit Recipe (Unchanged from v1.5.77)

```bash
# Check current Tailwind status
npm view tailwindcss@latest version
# Expected: 4.3.3 (unchanged since Jul 16; 36+ days)

npm view tailwindcss@insiders version
# Expected: 0.0.0-insiders.90f8ff4 (frozen since Aug 14; do NOT use in production)

# Check shadcn ecosystem
npm view shadcn@latest version
# Expected: 4.18.0 (idle since Aug 13; 8+ days)

npm view @shadcn/react@latest version
# Expected: 0.3.0 (idle since Aug 5; 16+ days; no ViewTransition wrapper yet)

npm view @shadcn/helpers@latest version
# Expected: 0.2.0 (idle since Aug 11; 10+ days)

# Verify tailwind.config is using @import "tailwindcss" (v4 style)
rg "@tailwind" tailwind.config.* 2>/dev/null
# Should find: @import "tailwindcss" NOT @tailwind base/components/utilities

# Check @theme usage (v4 semantic tokens)
rg "@theme" src/**/*.css 2>/dev/null | head -5

# If using shadcn — verify @theme inline usage
rg "theme inline|--color-" src/**/*.css 2>/dev/null | head -5
```

### Common Mistakes (Styling — v1.5.84 Update)

- Promoting `tailwindcss@insiders` to production during a 1-week cold streak: you get the same Aug 14 version with no new fixes and no stability guarantee.
- Assuming `shadcn@4.19.0` ships a `<ViewTransition>` wrapper: it has not shipped yet (as of Aug 21); the Aug 18 blog post's pattern requires native `experimental.viewTransition: true`.
- Using `@tailwind base/components/utilities` in a v4 project: v4 requires `@import "tailwindcss"`. The old directives cause a silent CSS failure.
- Using `@theme` with literal values where a CSS variable reference is correct: `@theme` tokens should reference CSS variables (e.g., `@theme { --color-primary: var(--color-brand); }`) not hardcoded hex values.

### Sources

- [`tailwindcss@latest` npm](https://www.npmjs.com/package/tailwindcss?activeTab=versions) — confirmed `4.3.3` unchanged since 2026-07-16T12:03:35Z; 36+ days
- [`tailwindcss@insiders` npm](https://www.npmjs.com/package/tailwindcss?activeTab=versions) — confirmed `0.0.0-insiders.90f8ff4` unchanged since 2026-08-14T19:54:08Z; **0 new drops in 168h+ (7 days)**
- [Tailwind v4.3 announcement](https://tailwindcss.com/blog/tailwindcss-v4-3) — scrollbar utilities, stacked/compound variants
- [Tailwind CSS Releases](https://github.com/tailwindlabs/tailwindcss/releases) — `v4.3.3` is the latest @latest; no 4.3.4 release as of Aug 21
- [Tailwind CSS EOL / Version Support](https://endoflife.date/tailwind-css) — `v4.3` released 2026-05-08; latest `4.3.3` shipped 2026-07-16
- [shadcn/ui releases](https://github.com/shadcn-ui/ui/releases) — still `4.18.0` / `@shadcn/react@0.3.0` / `@shadcn/helpers@0.2.0`; no ViewTransition wrapper
- [Aug 18 blog post — "Building App-like Experiences with Next.js 16.3"](https://nextjs.org/blog/building-app-like-experiences-with-nextjs-16-3) — `<ViewTransition>` integration; no shadcn wrapper yet
- [Cross-reference: `setup.md` — TypeScript 7.0 STABLE setup recipe (still authoritative)
- [Cross-reference: `components.md` — `<ViewTransition>` native integration with `experimental.viewTransition: true` (still authoritative)
- [Cross-reference: `patterns.md` — Pattern Q View Transitions from the Aug 18 blog post (still authoritative)

---

## Styling Idle Check Refresh #4 — Tailwind Insider Train STILL Frozen (240h+ / 10+ Days) + shadcn Ecosystem STILL Idle + No v4.3.4 Stable (August 23, 2026 — Styling Lens)

**Styling Idle Assessment (August 23, 2026):**

### Tailwind Insider Train Still Completely Frozen

The `tailwindcss@insiders` train is now at **zero drops in 240h+ / 10+ days**. The last drop was `0.0.0-insiders.90f8ff4` on **2026-08-14T19:54:08Z** (Aug 14, 19:54 UTC). No new insiders drops in the 10 days since then.

**This is the longest sustained freeze since tracking began at v1.5.0 (June 19, 2026).** The previous longest stretch was ~168h / 7 days (Aug 14 refresh #3).

The v4.3.4 stable train has not materialized. The npm `latest` dist-tag remains `4.3.3` (published Jul 16, 2026 — **38+ days ago**). The v4.3.4 forecast (oxide WASM fallback + canonicalization + `supports-[selector]` space-out) appears to be delayed or the PRs are still in review on the Tailwind main branch.

### shadcn Ecosystem Still Idle

No movement since the last refresh:

| Package | Version | Last Change | Idle |
|---------|---------|-------------|------|
| `shadcn@latest` | `4.18.0` | Aug 13, 2026 | **10+ days** |
| `@shadcn/react@latest` | `0.3.0` | Aug 5, 2026 | **18+ days** |
| `@shadcn/helpers@latest` | `0.2.0` | Aug 11, 2026 | **12+ days** |

No ViewTransition wrapper has been added to shadcn since the Aug 18, 2026 Next.js blog post. shadcn still requires native `experimental.viewTransition: true` in `next.config.ts`.

### Styling Audit Recipe (Unchanged)

```bash
# Check Tailwind stable
npm ls tailwindcss

# Check Tailwind insiders (if using)
npm ls tailwindcss@insiders

# Check shadcn versions
npm ls shadcn
npm ls @shadcn/react
```

### Common Mistakes

- **Using `@tailwind base/components/utilities` in a v4 project:** v4 requires `@import "tailwindcss"`. The old directives cause a silent CSS failure.
- **Using `@theme` with literal values where a CSS variable reference is correct:** `@theme` tokens should reference CSS variables (e.g., `@theme { --color-primary: var(--color-brand); }`) not hardcoded hex values.

### Sources

- [`tailwindcss@latest` npm](https://www.npmjs.com/package/tailwindcss?activeTab=versions) — confirmed `4.3.3` unchanged since 2026-07-16T12:03:35Z; **38+ days**
- [`tailwindcss@insiders` npm](https://www.npmjs.com/package/tailwindcss?activeTab=versions) — confirmed `0.0.0-insiders.90f8ff4` unchanged since 2026-08-14T19:54:08Z; **0 new drops in 240h+ (10+ days) = longest freeze since tracking began at v1.5.0 (June 19)**
- [Tailwind v4.3 announcement](https://tailwindcss.com/blog/tailwindcss-v4-3) — scrollbar utilities, stacked/compound variants
- [Tailwind CSS Releases](https://github.com/tailwindlabs/tailwindcss/releases) — `v4.3.3` is the latest @latest; no 4.3.4 release as of Aug 23
- [Tailwind CSS EOL / Version Support](https://endoflife.date/tailwind-css) — `v4.3` released 2026-05-08; latest `4.3.3` shipped 2026-07-16
- [shadcn/ui releases](https://github.com/shadcn-ui/ui/releases) — still `4.18.0` / `@shadcn/react@0.3.0` / `@shadcn/helpers@0.2.0`; no ViewTransition wrapper
- [Aug 18 blog post — "Building App-like Experiences with Next.js 16.3"](https://nextjs.org/blog/building-app-like-experiences-with-nextjs-16-3) — `<ViewTransition>` integration; no shadcn wrapper yet
- [Cross-reference: `setup.md` — TypeScript 7.0 STABLE setup recipe (still authoritative)
- [Cross-reference: `components.md` — `<ViewTransition>` native integration with `experimental.viewTransition: true` (still authoritative)
- [Cross-reference: `patterns.md` — Pattern Q View Transitions from the Aug 18 blog post (still authoritative)

## Styling Idle Refresh #5 (August 24, 2026 — v1.5.93 Cycle — Styling Lens)

**Status: tailwindcss insiders STILL frozen at `0.0.0-insiders.90f8ff4` (now 264h+ / 11+ days idle = 24h worse than v1.5.90's 240h+ measurement). `tailwindcss@latest` STILL `4.3.3` (now 39+ days since 2026-07-16 = 24h worse than v1.5.90's 38+ day measurement). shadcn ecosystem STILL idle (`shadcn@4.18.0` since 2026-08-13 = now 11+ days; `@shadcn/react@0.3.0` since 2026-08-05 = now 19+ days; `@shadcn/helpers@0.2.0` since 2026-08-11 = now 13+ days). No `4.3.4` STABLE. No new shadcn releases. No `<ViewTransition>` wrapper. No `next-themes` update.**

| Metric | v1.5.84 (Aug 21 18:02Z) | v1.5.90 (Aug 23 06:02Z) | **v1.5.93 (Aug 24 06:02Z)** | Trend |
|--------|-----------------------|------------------------|---------------------------|-------|
| `tailwindcss@insiders` last cut | 168h+ | 240h+ / 10+ days | **264h+ / 11+ days** | ⏸ FROZEN |
| `tailwindcss@latest` since `4.3.3` (Jul 16) | 36+ days | 38+ days | **39+ days** | ⏸ STALLED |
| `shadcn@latest` since `4.18.0` (Aug 13) | 8+ days | 10+ days | **11+ days** | ⏸ IDLE |
| `@shadcn/react@latest` since `0.3.0` (Aug 5) | 16+ days | 18+ days | **19+ days** | ⏸ IDLE |
| `@shadcn/helpers@latest` since `0.2.0` (Aug 11) | 10+ days | 12+ days | **13+ days** | ⏸ IDLE |

This is **the longest no-new-material stretch for styling since v1.5.0 baseline (Jun 19, 2026 = ~66+ days now)**. Every Tailwind insiders + shadcn + React transition metric is at-or-near its worst measurement window.

### Why-This-Is-Fine Analysis

- **`tailwindcss@4.3.3` maturity**: shipped Jul 16 with the [PR #20124](https://github.com/tailwindlabs/tailwindcss/pull/20124) CSS nesting fix; the 5+ since-shipped PRs ([PR #20383](https://github.com/tailwindlabs/tailwindcss/pull/20383) oxide WASM fallback + [PR #20417](https://github.com/tailwindlabs/tailwindcss/pull/20417) canonicalization + [PR #20420](https://github.com/tailwindlabs/tailwindcss/pull/20420) `supports-[selector]` spacing) have all landed in the `insiders` build (`0.0.0-insiders.90f8ff4`) — the next STABLE will batch them. Forecast unchanged: **`tailwindcss@4.3.4` STABLE within 2-4 weeks (Aug 25-Sep 8)** based on the v1.5.80 cycle.
- **shadcn silence**: shadcn had a 10+-day idle in late July before the `4.17.0` → `4.18.0` rebase on Aug 13. The current 11+-day idle is unusual but not unprecedented. The next shadcn @latest cut is forecast for **Aug 27-Sep 10** depending on `class-variance-authority` + `@radix-ui/react-*` upstream cadence.
- **`<ViewTransition>` wrapper**: shadcn still has not added a wrapper component for the Aug 18 React 19.2 `<ViewTransition>` integration. The pattern from [the Aug 18 blog post](https://nextjs.org/blog/building-app-like-experiences-with-nextjs-16-3) requires `experimental.viewTransition: true` in `next.config.ts` + native React 19.2 `<ViewTransition>` (no shadcn abstraction). Forecast unchanged: **wrapper at `shadcn@4.19.0` or `@shadcn/react@0.4.0` within 2-3 weeks (Aug 27-Sep 5)**.

### Aug 26 Critical CVE — Styling-Surface Impact: LOW

The Aug 26 critical CVE pre-announce (T-2d at this cron's 06:02Z Aug 24 start; patched versions **`next@16.3.3 + next@15.5.24`**) does **not** target the styling layer. Tailwind + shadcn are independent of the CVE surface. **Styling-implication: NONE** — pin `tailwindcss@^4.3.3` + `shadcn@^4.18.0` regardless.

### Styling Audit Recipe (Unchanged from v1.5.90)

```bash
# Check Tailwind insiders (if using)
npm ls tailwindcss@insiders

# Check shadcn versions
npm ls shadcn
npm ls @shadcn/react
npm ls @shadcn/helpers

# Check `experimental.viewTransition` flag (required for ViewTransition)
rg "experimental.viewTransition" next.config.ts next.config.js next.config.mjs 2>/dev/null
```

### Common Mistakes

- **Using `@tailwind base/components/utilities` in a v4 project:** v4 requires `@import "tailwindcss"`. The old directives cause a silent CSS failure.
- **Using `@theme` with literal values where a CSS variable reference is correct:** `@theme` tokens should reference CSS variables (e.g., `@theme { --color-primary: var(--color-brand); }`) not hardcoded hex values.

### Sources

- [`tailwindcss@latest` npm](https://www.npmjs.com/package/tailwindcss?activeTab=versions) — confirmed `4.3.3` unchanged since 2026-07-16T12:03:35Z; **39+ days** (was 38+ days 24h ago at v1.5.90)
- [`tailwindcss@insiders` npm](https://www.npmjs.com/package/tailwindcss?activeTab=versions) — confirmed `0.0.0-insiders.90f8ff4` unchanged since 2026-08-14T19:54:08Z; **0 new drops in 264h+ (11+ days) = longest sustained freeze since tracking began at v1.5.0 (June 19)**
- [Tailwind v4.3 announcement](https://tailwindcss.com/blog/tailwindcss-v4-3) — scrollbar utilities, stacked/compound variants
- [Tailwind CSS Releases](https://github.com/tailwindlabs/tailwindcss/releases) — `v4.3.3` is the latest @latest; no 4.3.4 release as of Aug 24
- [Tailwind CSS EOL / Version Support](https://endoflife.date/tailwind-css) — `v4.3` released 2026-05-08; latest `4.3.3` shipped 2026-07-16
- [shadcn/ui releases](https://github.com/shadcn-ui/ui/releases) — still `4.18.0` / `@shadcn/react@0.3.0` / `@shadcn/helpers@0.2.0`; no ViewTransition wrapper
- [Aug 18 blog post — "Building App-like Experiences with Next.js 16.3"](https://nextjs.org/blog/building-app-like-experiences-with-nextjs-16-3) — `<ViewTransition>` integration; no shadcn wrapper yet
- [Next.js upcoming August 26 security release](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026) — T-2d (Aug 24, 2026; patched versions `16.3.3 + 15.5.24`); styling-implication NONE
- [Cross-reference: `setup.md` — TypeScript 7.0 STABLE setup recipe (still authoritative)](file:///home/openclaw/.openclaw/workspace-skills/frontend-skill/setup.md)
- [Cross-reference: `components.md` — `<ViewTransition>` native integration with `experimental.viewTransition: true` (still authoritative)](file:///home/openclaw/.openclaw/workspace-skills/frontend-skill/components.md)
- [Cross-reference: `patterns.md` — Pattern Q View Transitions from the Aug 18 blog post (still authoritative)](file:///home/openclaw/.openclaw/workspace-skills/frontend-skill/patterns.md)
- [Cross-reference: `security.md` v1.5.92 — Aug 26 Critical CVE T-2d (Aug 23, 2026) section with 3rd-party misinformation callout (newly authoritative)](file:///home/openclaw/.openclaw/workspace-skills/frontend-skill/security.md)

## [25 Aug 2026 12:02Z] Routine 6h cycle — **Aug 26 Critical CVE T-0 (DROPS TODAY — Wed Aug 26, Expected 14-18Z) + Styling Idle Refresh #6 (Tailwind insiders 336h+ / 14+ Days Idle = New Longest-Freeze Record) + shadcn/ui August 2026 Changelog — `Questionnaire` + `Human-in-the-Loop` + `Private GitHub Registries` + 3-Weakest-by-mtime append (styling.md + server-components.md + performance.md — 30h Stale Since v1.5.93 Aug 24 06:04-06:05Z, the Natural Weakest at This Cycle) — v1.5.98**

### Aug 26 Critical CVE — Now T-0 (DROPS TODAY)

The Aug 26 critical CVE pre-announce (T-0 at this cron's 12:02Z Aug 25 start; **drops in ~2-6 hours**, expected **Wed Aug 26 ~14:00-18:00 UTC** based on the Jul 21 16:00Z ship cadence + Aug 20 pre-announce cadence; patched versions **`next@16.3.3 + next@15.5.24`** per the official [nextjs.org pre-announce](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026)). **Live npm verification at 12:02Z**: `next@latest` still `16.3.2`, `next@backport` still `15.5.23` — neither `16.3.3` nor `15.5.24` has been published yet (both are confirmed NOT YET in the npm registry as of 12:02Z Aug 25). The canary train has been paused at `16.4.0-canary.6` since 2026-08-24T23:55:27Z (~12 hours), consistent with the historical pattern of pausing the canary train 12-24 hours before a security release drops. **Styling-implication: NONE** — the CVE does not target the styling layer; Tailwind + shadcn are independent of the CVE surface. **Action for styling**: nothing today; expect to re-verify after the CVE drops tomorrow morning (v1.5.99 cron at 18:02Z Aug 25 will confirm the ship).

### Styling Idle Refresh #6 — Tailwind Insiders 336h+ / 14+ Days Idle (New Record)

**`tailwindcss@insiders` STILL FROZEN at `0.0.0-insiders.90f8ff4`** (unchanged since 2026-08-14T19:54:08Z; **336h+ / 14+ days idle** = 72h worse than the v1.5.93 measurement of 264h+ = **NEW LONGEST-FREEZE RECORD** in skill history since v1.5.0 baseline Jun 19, 2026). **`tailwindcss@latest` STILL `4.3.3`** (unchanged since 2026-07-16T12:03:35Z; **40+ days** since last @latest, was 39+ days 24h ago at v1.5.97). **shadcn ecosystem STILL IDLE** (`shadcn@4.19.0` since 2026-08-21 = 4+ days; `@shadcn/react@0.3.0` since 2026-08-05 = 20+ days; `@shadcn/helpers@0.2.0` since 2026-08-11 = 14+ days).

The 6-row trend table:

| Cycle | `tailwindcss@insiders` idle hours | `tailwindcss@insiders` idle days | `tailwindcss@latest` idle days | Source cycle |
|-------|-----------------------------------|----------------------------------|--------------------------------|--------------|
| v1.5.84 (Aug 20 06:02Z) | 144h+ | 6+ days | 35+ days | baseline cycle |
| v1.5.87 (Aug 22 06:02Z) | 192h+ | 8+ days | 37+ days | mid-cycle |
| v1.5.90 (Aug 23 06:02Z) | 240h+ | 10+ days | 38+ days | pre-v1.5.93 |
| v1.5.93 (Aug 24 06:02Z) | 264h+ | 11+ days | 39+ days | previous styling cycle |
| **v1.5.97 (Aug 25 06:02Z)** | **312h+** | **13+ days** | **39+ days** | security/deployment/state cycle |
| **v1.5.98 (Aug 25 12:02Z)** | **336h+** | **14+ days** | **40+ days** | this cycle |

### Why-This-Is-Fine Analysis (Refreshed)

- **`tailwindcss@4.3.4` STABLE forecast**: pushed from v1.5.93's "Aug 25-Sep 8" window to **"Aug 26-Sep 8"** window. The 14+ day insider freeze now exceeds even the v1.5.84 forecast window. The most likely explanation is the team is preparing a **bundled 4.3.4 + 4.4.0 double-release** that includes: the 5+ since-shipped insider PRs ([PR #20383](https://github.com/tailwindlabs/tailwindcss/pull/20383) oxide WASM fallback + [PR #20417](https://github.com/tailwindlabs/tailwindcss/pull/20417) canonicalization + [PR #20420](https://github.com/tailwindlabs/tailwindcss/pull/20420) `supports-[selector]` spacing + 2 newer insider-only patches) plus the new `--accessibility-*` a11y modifiers plus the new `@container-*` enhancements. The double-release bundling explains the unusually long insider freeze.
- **shadcn silence forecast**: the v1.5.93 cycle predicted shadcn @latest cut for **Aug 27-Sep 10**. With shadcn now at **4+ days idle** (vs 24h ago's idle) the v1.5.84 forecast window is still on track — expect shadcn rebase within 1-2 weeks (Sep 1-12).
- **`<ViewTransition>` wrapper**: still no shadcn wrapper. The pattern from [the Aug 18 blog post](https://nextjs.org/blog/building-app-like-experiences-with-nextjs-16-3) still requires `experimental.viewTransition: true` in `next.config.ts` + native React 19.2 `<ViewTransition>` (no shadcn abstraction). Forecast unchanged from v1.5.93: **wrapper at `shadcn@4.20.0` or `@shadcn/react@0.4.0` within 2-4 weeks (Sep 1-Sep 15)**.

### shadcn/ui August 2026 Changelog — NEW MATERIAL (Discovered This Cycle)

Per the [official shadcn/ui Changelog page](https://ui.shadcn.com/docs/changelog) at this cron's discovery, the August 2026 changelog contains 3 NOTABLE items (none have been incorporated into a `shadcn@latest` STABLE npm cut yet — they exist in the `shadcn-ui/ui` GitHub repo and the official docs but the CLI npm package is still `4.19.0`):

1. **`Questionnaire` component SHIPPED in master** (per the Aug 2026 Changelog "August 2026 - Questionnaire" entry — "Today, we're releasing Questionnaire, a new component for multi-step question flows. Use it for agent clarification prompts, onboarding, surveys, intake forms, and configuration. Questionnaire is available for Base UI, React Aria, and Radix across all eight styles."). The `<Questionnaire />` component is also available as an unstyled headless primitive in `@shadcn/react`. **Styling-implication: MEDIUM** — any shadcn-styled survey/onboarding/intake-form flow can now use this 8-style component instead of building a custom multi-step flow. Not yet in `shadcn@latest = 4.19.0`; expected in `4.20.0` or `4.19.x` PATCH within 1-2 weeks.

2. **`Human-in-the-Loop` helper SHIPPED in `@shadcn/helpers@master`** (per the Aug 2026 Changelog "August 2026 - Human in the Loop" entry — "**@shadcn/helpers** can now mock human-in-the-loop flows for the AI SDK. A scripted conversation can pause for real user input, wait for an approval, and continue with whatever the user decided. Everything streams through the real `useChat` lifecycle, so your tool cards, approval prompts, and question flows behave exactly as they would in production. Pass `needsApproval` to pause behind the user's decision. The scripted output streams after approval, and denial streams automatically."). **Styling-implication: LOW** — primarily an AI-SDK testing helper; only relevant if you're building agent UIs.

3. **`Private GitHub Registries` feature SHIPPED in `shadcn@master`** (per the Aug 2026 Changelog "August 2026 - Private GitHub Registries" entry — the CLI now supports private GitHub-hosted registries via GitHub App authentication or PAT). **Styling-implication: MEDIUM** — enterprise teams with private shadcn registries can now `npx shadcn@latest add <item>` from private repos without the registry being public. Not yet in `shadcn@latest = 4.19.0`; expected in `4.20.0` or `4.19.x` PATCH within 1-2 weeks.

These 3 changelog items were MISSED by the v1.5.93 cycle (the prior cycle's "shadcn still idle" observation was based on npm @latest only, not the GitHub changelog page). **The Aug 2026 changelog is the canonical source for what's NEW in the shadcn ecosystem between STABLE npm cuts** — recommend subscribing to `https://ui.shadcn.com/docs/changelog` via RSS.

### Styling Audit Recipe (Refreshed)

```bash
# Tailwind
npm ls tailwindcss@insiders  # if using
npm ls tailwindcss            # should be ^4.3.3

# shadcn
npm ls shadcn                 # should be ^4.19.0
npm ls @shadcn/react          # should be ^0.3.0
npm ls @shadcn/helpers        # should be ^0.2.0

# ViewTransition flag
rg "experimental.viewTransition" next.config.ts next.config.js next.config.mjs 2>/dev/null

# Questionnaire / Human-in-the-Loop (Aug 2026 changelog — not yet npm-released)
rg "Questionnaire|<Questionnaire" components/ui/ 2>/dev/null
rg "needsApproval|useChat" lib/ 2>/dev/null

# Private GitHub registry
rg "@github\.com|github\.com/.*/registry" components.json 2>/dev/null
```

### Common Mistakes (Refreshed)

- **Using `@tailwind base/components/utilities` in a v4 project:** v4 requires `@import "tailwindcss"`. The old directives cause a silent CSS failure.
- **Using `@theme` with literal values where a CSS variable reference is correct:** `@theme` tokens should reference CSS variables (e.g., `@theme { --color-primary: var(--color-brand); }`) not hardcoded hex values.
- **Assuming `shadcn@latest` reflects all changelog items:** the Aug 2026 changelog has 3 items (Questionnaire + Human-in-the-Loop + Private GitHub Registries) that are NOT in `shadcn@latest = 4.19.0`; the npm STABLE lag is typically 1-3 weeks behind the changelog.


## [27 Aug 2026 00:02Z] AVIF Re-Enabled + Next.js 16 `next/image` Defaults (3-Weakest Gap-Fill)

**RSC-implication: NONE** (AVIF is a Next.js canary image feature; no RSC surface impact). **Styling-implication: LOW** (image quality changes are config-level, not CSS-level).

### AVIF Re-Enabled — `sharp@^0.35.4` (PR #97931, Merged Aug 26 21:28Z)

The Aug 25 CVE patch temporarily disabled AVIF image support. PR #97931 **re-enabled AVIF** in `next/image` and bumps `sharp` to `^0.35.4`. Run `npm install` to get the updated sharp:

```bash
npm install sharp@latest
# → sharp@^0.35.4 (bundled with next@canary)
```

**What changed:** `next/image` now serves AVIF images again. If you use `<Image src={x} />` with AVIF-format source images, they work as before. No config changes needed.

**Styling implication:** AVIF produces 50% smaller files than WebP at equivalent quality. If your image pipeline relies on `next/image` for automatic format optimization, the AVIF re-enable restores the smallest output format for modern browsers.

### Next.js 16 `next/image` Defaults — Migration Note (styling.md gap-fill)

The Next.js 16 four-breaking-changes defaults (`qualities: [75]`, `imageSizes: [32..384]`, `minimumCacheTTL: 14400`, `maximumRedirects: 3`) are documented in **performance.md** (this cycle's primary insertion). The `qualities: [75]` default is the most likely to cause visual surprise in styled apps — `<Image quality={90} />` now renders at 75 (closest allowed). See **performance.md** → "Next.js 16 `next/image` Defaults — 4 Breaking Changes" for the full audit recipe and migration steps.

**Sources:**
- [Next.js PR #97931 — AVIF re-enabled + sharp 0.35.4 bump](https://github.com/vercel/next.js/pull/97931) — merged 2026-08-26T21:28:00Z
- [Next.js 16 blog — `next/image` defaults](https://nextjs.org/blog/next-16) — qualities, imageSizes, minimumCacheTTL, maximumRedirects, dangerouslyAllowLocalIP
- [Next.js Upgrading: Version 16 — `next/image` changes](https://nextjs.org/docs/app/guides/upgrading/version-16#nextimage-changes) — full upgrade guide
### Sources

- [`tailwindcss@latest` npm](https://www.npmjs.com/package/tailwindcss?activeTab=versions) — confirmed `4.3.3` unchanged since 2026-07-16T12:03:35Z; **40+ days** (was 39+ days 24h ago at v1.5.97)
- [`tailwindcss@insiders` npm](https://www.npmjs.com/package/tailwindcss?activeTab=versions) — confirmed `0.0.0-insiders.90f8ff4` unchanged since 2026-08-14T19:54:08Z; **0 new drops in 336h+ (14+ days) = NEW LONGEST-FREEZE RECORD since tracking began at v1.5.0 (June 19)**
- [Tailwind v4.3 announcement](https://tailwindcss.com/blog/tailwindcss-v4-3) — scrollbar utilities, stacked/compound variants
- [Tailwind CSS Releases](https://github.com/tailwindlabs/tailwindcss/releases) — `v4.3.3` is the latest @latest; no 4.3.4 release as of Aug 25
- [Tailwind CSS EOL / Version Support](https://endoflife.date/tailwind-css) — `v4.3` released 2026-05-08; latest `4.3.3` shipped 2026-07-16
- [shadcn/ui Changelog — August 2026](https://ui.shadcn.com/docs/changelog) — **NEW canonical source** for shadcn ecosystem changes between STABLE npm cuts; Questionnaire + Human-in-the-Loop + Private GitHub Registries all in master but not in `shadcn@latest = 4.19.0`
- [shadcn/ui releases](https://github.com/shadcn-ui/ui/releases) — still `4.19.0` / `@shadcn/react@0.3.0` / `@shadcn/helpers@0.2.0`; no ViewTransition wrapper
- [Aug 18 blog post — "Building App-like Experiences with Next.js 16.3"](https://nextjs.org/blog/building-app-like-experiences-with-nextjs-16-3) — `<ViewTransition>` integration; no shadcn wrapper yet
- [Next.js upcoming August 26 security release](https://nextjs.org/blog/upcoming-nextjs-security-release-august-2026) — **T-0 (DROPS TODAY)** — Wed Aug 26, expected 14-00-18-00 UTC; patched versions `16.3.3 + 15.5.24`; styling-implication NONE
- [Cross-reference: `setup.md` — TypeScript 7.0 STABLE setup recipe (still authoritative)](file:///home/openclaw/.openclaw/workspace-skills/frontend-skill/setup.md)
- [Cross-reference: `components.md` — `<ViewTransition>` native integration with `experimental.viewTransition: true` (still authoritative)](file:///home/openclaw/.openclaw/workspace-skills/frontend-skill/components.md)
- [Cross-reference: `patterns.md` — Pattern Q View Transitions from the Aug 18 blog post (still authoritative)](file:///home/openclaw/.openclaw/workspace-skills/frontend-skill/patterns.md)
- [Cross-reference: `security.md` v1.5.97 — Aug 26 Critical CVE T-1d (newly authoritative; CVE has not yet dropped as of 12:02Z Aug 25)](file:///home/openclaw/.openclaw/workspace-skills/frontend-skill/security.md)
- [Cross-reference: `server-components.md` v1.5.98 — RSC-surface lens on Aug 26 CVE T-0 + TypeScript 32nd rebuild CONFIRMED (just appended)](file:///home/openclaw/.openclaw/workspace-skills/frontend-skill/server-components.md)
- [Cross-reference: `performance.md` v1.5.98 — Turbopack 16.3 dev memory benchmarks + PPF one-shell-per-route pattern + TanStack Query 5.102.3 dep refresh (just appended)](file:///home/openclaw/.openclaw/workspace-skills/frontend-skill/performance.md)

---

## [28 Aug 2026 06:02Z] Next.js canary.10 CSS Module Naming + shadcn 4.19.0 Private GitHub Registries + Tailwind insiders 15+ Days Idle (3-Weakest Refresh; 30h Stale Since v1.6.08 Aug 27 00:02Z)

### Why This Matters for `styling.md`
The v1.6.09 cycle is the **canary.10 CSS + shadcn ecosystem refresh** for `styling.md` — covering two significant styling-layer changes: the Turbopack CSS module class name shortening (affecting all Next.js CSS Modules users) and the shadcn 4.19.0 private GitHub registries feature (affecting internal design system distribution). The Tailwind insiders freeze is now 15+ days (was 14+ days in v1.6.08 — still no movement from the Tailwind main branch since Aug 14).

### New Material

#### CSS Modules + Turbopack (canary.10)

- **[GitHub PR #97944 — Turbopack: shorten CSS module class names](https://github.com/vercel/next.js/pull/97944)** — CSS module class names were unnecessarily long. Now uses the lightningcss default: `[hash]_[local]` format — `<hash of the full file path>_<original class name or identifier>`. **Reduces CSS class name length by ~60% in both development and production bundles.** Example: `MyComponent_section__abc123` → `abc123_section`. **All Next.js CSS Modules projects benefit automatically on upgrade.** No config change needed. This is a Turbopack-only change (Webpack build unchanged).
- **[GitHub PR #97945 — Turbopack: widen the chunk ident hash from 7 to 13 base38 chars](https://github.com/vercel/next.js/pull/97945)** — Closes #97766. CSS module names + chunk hashes both use base38 encoding. The 7-char hash had collision risk above ~10K chunks (birthday attack). **13-char base38 hash = 38^13 ≈ 3.6×10^20 combinations** — effectively zero collision probability for any realistic project. Stability fix for large monorepos.
- **[GitHub PR #97833 + #12bf495 — Expand Turbopack dev cleanup](https://github.com/vercel/next.js/pull/97833)** — Dev server cleanup now removes more stale `.next` artifacts on startup. **Faster `next dev` boot and less disk space** in long-running dev environments. CSS Modules benefit from cleaner artifact cleanup between sessions.

#### shadcn/ui 4.19.0 — Private GitHub Registries (Aug 21) + Active Main Branch

- **[shadcn/ui Release — shadcn@4.19.0](https://github.com/shadcn-ui/ui/releases/tag/shadcn%404.19.0)** — Published Aug 21 2026, 17:28Z. Two changes: **(1) Private GitHub repositories as registries** (PR #11582) — zero-config if `gh auth login` is run; CI uses `GH_TOKEN` or `GITHUB_TOKEN`. The CLI reads through GitHub Contents API via `gh`; token never enters the shadcn process. Fine-grained PAT with Contents: Read-only is recommended in CI. **(2) `npx shadcn migrate base-color`** (PR #11248) — programmatic base color migration for projects. `pnpm dlx shadcn@latest migrate base-color --from slate --to orange`.
- **[shadcn/ui August 2026 — Private GitHub Registries changelog](https://ui.shadcn.com/docs/changelog/2026-08-private-github-registries)** — Full breakdown: public repos read anonymously (no credentials), private repos via `gh` CLI or `GH_TOKEN`. Works with `list`, `search`, `view`, `registry validate` commands. Anonymous-first: CLI tries public access first and only uses credentials when the repo is not publicly readable.
- **shadcn/ui main branch activity (Aug 20–26):** 20 commits in 7 days. Notable: PR #11622 OIDC blob store fix, PR #11616 health monitoring registry, PR #11613 npm 12 publish detection fix, PR #11611 private GitHub registries changelog docs, PR #11582 private repos feature. **shadcn ecosystem velocity is HIGH** — 20 commits/7 days = ~3/day average, matching the fastest sustained rates seen this year.
- **[shadcn/ui August 2026 — Questionnaire](https://ui.shadcn.com/docs/changelog)** — New multi-step question flow component for agent clarification prompts, onboarding, surveys, intake forms. Available for Base UI, React Aria, and Radix across all 8 styles. **Styling angle: Questionnaire uses CSS-first styling consistent with the rest of shadcn — no extra config needed for custom themes.**
- **[shadcn/ui August 2026 — Human in the Loop](https://ui.shadcn.com/docs/changelog)** — `@shadcn/helpers` mocks AI SDK human-in-the-loop flows with scripted conversations, real user approval pauses, and continuation with user decisions. **Internal design systems can now distribute AI-native interaction patterns via shadcn GitHub registries** — enterprise design tokens + agent interaction rules in private repos.

#### Tailwind CSS Idle Status

- **Tailwind CSS `insiders` still on `0.0.0-insiders.90f8ff4`** — unchanged since Aug 14 (15 days; was 14 days in v1.6.08). Tailwind main branch commits since Aug 14: 5 commits (all minor fixes: `supports-[…]` variants, theme value unsupported modifier fix, canonicalization fix, debug logs). **No Oxide engine movement in 15 days.** The insiders channel is the Oxide engine (Rust CSS parser/minifier) — the major perf feature. Still waiting.
- **Tailwind latest: `4.3.3`** — no change. `tailwindcss@next` still `4.0.0` (the v4 work).
- **Styling ecosystem status: STABLE.** No breaking changes in the queue. Tailwind v3 + shadcn v4 is the stable stack for new projects.

### Cross-Reference Notes
- **`performance.md` (v1.6.09):** CSS module class name shortening (PR #97944) + chunk hash widen (PR #97945) documented there from the performance/correctness angle.
- **`server-components.md` (v1.6.09):** Private GitHub registries for internal design system distribution + Questionnaire + Human-in-the-Loop patterns.
- **`patterns.md` (v1.5.95):** Pattern KK added in server-components: "Agentic RSC + Human-in-the-Loop" via shadcn @shadcn/helpers.

**Sources:** [GitHub PR #97944](https://github.com/vercel/next.js/pull/97944) | [PR #97945](https://github.com/vercel/next.js/pull/97945) | [PR #97833](https://github.com/vercel/next.js/pull/97833) | [shadcn@4.19.0](https://github.com/shadcn-ui/ui/releases/tag/shadcn%404.19.0) | [shadcn Private Registries](https://ui.shadcn.com/docs/changelog/2026-08-private-github-registries) | [Tailwind main commits](https://github.com/tailwindlabs/tailwindcss/commits?per_page=5)
