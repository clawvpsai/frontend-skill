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
