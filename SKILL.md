---
name: Frontend
slug: frontend-skill
version: 1.4.32
description: Production-grade React/Next.js frontend development — ship modern web apps without common pitfalls.
metadata: {"emoji":"⚛️","requires":{"bins":["node","npm"]},"os":["linux","darwin","win32"]}
---

## Navigation

| Topic | File | When to Use |
|---|---|---|
| Project setup, TypeScript, Vite 8.1, Biome, env vars, Turbopack `self-contained` runtime rename (16.3 canary.61), **Turbopack Service Worker support** (16.3 canary.68 — discover/compile/serve via `ServiceWorkerEntryModule` + self-contained single-chunk runtime, `service_worker_chunk_filename` config), `cache-components-instant-false` codemod | `setup.md` | Starting a new project |
| React components, shadcn/ui, composition, ref forwarding, ref-callback cleanup (React 19), `useFormStatus` / `useId` / `useEffectEvent` (React 19/19.2) | `components.md` | Building UI |
| Server vs Client components, data fetching, `use cache`, `use()` hook | `server-components.md` | Next.js App Router |
| Routing, layouts, loading, error boundaries, `proxy.ts`, parallel routes `default.tsx` (Next 16), `experimental.prefetchInlining` (now documented) / `cachedNavigations` (16.2; widened to `boolean \| 'allow-runtime'` in 16.3 canary.61), `experimental.useExperimentalReact` (16.3 canary), `generateStaticParams` required for root params with CC (canary.67), suppress `prefetch={true}` warning on `instant = false` opt-out (canary.67) | `routing.md` | Navigation & page structure |
| Forms with React Hook Form + Zod | `forms.md` | Any form or input |
| Zustand, React Query, TanStack Query v5 (incl. `useSuspenseQuery` + `React.use(query.promise)`), v5 migration | `state.md` | State & server state |
| Auth patterns, NextAuth.js v4/v5, JWT, session management, 2026 alternatives (Clerk, Better Auth) | `auth.md` | User auth & sessions |
| Route handlers, Server Actions, API routes, SSE, WebSockets | `api.md` | Backend API endpoints |
| Tailwind CSS v4, design tokens, themes | `styling.md` | Styling & theming |
| Tailwind CSS v4, design tokens, modern cross-engine CSS (`field-sizing`, `@starting-style`, anchor positioning, OKLCH, `light-dark()`, scroll-driven animations) | `styling.md` | Styling & theming |
| Streaming, Suspense, image optimization, PPR (legacy codepaths REMOVED in 16.3 canary.61 then REVERTED in canary.66 — use `cacheComponents: true`), metadata image route static prerendering under Cache Components (16.3 canary.61, canary.67 local-fonts Buffer-corruption follow-up), App Shells (canary), Dev Insights, Navigation Inspector blocking-route error (canary.67), prefetch controls, **dev cold-cache badge gated behind `experimental.coldCacheBadge`** (canary.68, default off; transient pill unchanged), **`partialPrefetching` Shell Prefetch reveal-after-ShellRuntime** (canary.68) | `performance.md` | Speed & UX |
| Vercel (incl. Vercel Connect, eve, Chat SDK), Docker, Node adapter, self-hosted, Next.js MCP Server | `deployment.md` | Going live |
| XSS, CSRF, CSP, input sanitization | `security.md` | Hardening |
| Vitest 4 (Browser Mode stable, Visual Regression, Playwright Trace) + **Playwright 1.61** (WebAuthn passkeys `browserContext.credentials`, WebStorage `page.localStorage/sessionStorage`, video modes parity, expect.soft.poll) + component tests | `testing.md` | Test-driven dev |
| Vitest 4 (Browser Mode stable, Visual Regression, Playwright Trace) + **Playwright 1.61** (WebAuthn passkeys `browserContext.credentials`, WebStorage `page.localStorage/sessionStorage`, video modes parity, expect.soft.poll) + component tests + **Vitest 5.0-beta breaking changes (beta.3/4/5 May 19–June 15, 2026)** — strict `toHaveTextContent` + new `toMatchTextContent`, `locators.exact: true` default in Browser Mode, Node 22 / Vite 6.4 hard requirement, removed entry points (`vitest/coverage`, `vitest/environments`, `vitest/snapshot`, `vitest/runners`, `vitest/suite`, `vitest/reporters`, `vitest/mocker`), `expect.poll` fails on timeout, hoistable-methods-out-of-scope throw, `bench` moved into `test()` fixture, `test.sequential` → `{ concurrent: false }`, no ancestor-directory config lookup, `@vitest/runner` inlined into `vitest`, `TestModule.id` 1-based, `coverage.thresholds.perFile` accepts an object, browser orchestrator requires `sessionId`, happy-dom/jsdom `window` mutation allowed, `--repeats` CLI flag | `testing.md` | Test-driven dev |
| Strict TypeScript, generics, utilities, `import defer`, Temporal API, `satisfies`, `const` type parameters, branded/opaque types, template literal types, **TS 7.0 RC** (June 18, 2026, Go compiler via `typescript@rc`, `--checkers`/`--builders`/`--singleThreaded`, `stableTypeOrdering` mandatory, `@typescript/typescript6` compat package) | `typescript.md` | Type safety |
| React Compiler, `<Activity>`, useOptimistic, `after()`, View Transitions, `prefetch` segment config (`allow-runtime` rename in 16.3), `cacheComponents` adoption (`instant = false` opt-out, `cache-components-instant-false` codemod, `next-cache-components-adoption` agent skill in 16.3 canary.61; canary.67 — skill leads with `next-dev-loop` + adds build-only verification) | `patterns.md` | Composite recipes |

## Critical Rules (Never Forget)

- **`'use client'` boundary** — Only add when you need browser APIs, hooks, or interactivity. Keep everything else server-side.
- **Server Components can't use hooks** — No `useState`, `useEffect`, `useContext` in server components.
- **Colocate data fetching** — Fetch data in the component that needs it, not prop-drill from the top.
- **Zod for all schema validation** — Never trust raw form/API data without validation.
- **Tailwind = utility-first** — Don't fight it. Compose utilities, don't fight the cascade.
- **Server Actions = mutations** — Use them for form submits and data mutations; fetch for reads.
- **`<Image>` over `<img>`** — Always use Next's Image for automatic optimization and CLS prevention.
- **Abort in-flight requests** — Use `AbortController` to prevent race conditions on navigation.
- **Types in, types out** — Mirror backend response types exactly. No `any` from API calls.

## Version Defaults

- **Next.js 16.2.9** (latest stable — App Router, Server Components, Server Actions, Turbopack stable for production, Node.js proxy, `use cache` directive, PPR stable; **16.3.0-canary.68** released June 25, 2026 ([`64a39af`](https://github.com/vercel/next.js/commit/64a39af), 18 commits since canary.67) — **Turbopack Service Worker support** ([#94920](https://github.com/vercel/next.js/pull/94920) + [#94921](https://github.com/vercel/next.js/pull/94921) + [#94922](https://github.com/vercel/next.js/pull/94922) — first-class Turbopack compilation + serving of `ServiceWorkerEntryModule`s via the new self-contained single-chunk runtime, `service_worker_chunk_filename` config; closes a long-standing footgun where PWA registrations on Turbopack dev/build silently fell back to a static file or stale worker), **`next-dev-loop` cookies fix** ([#95177](https://github.com/vercel/next.js/pull/95177) — bumps to the fixed `agent-browser` from `vercel-labs/agent-browser#1486`, no more inter-iteration cookie-logout during agentic dev loops), **dev Cold-cache badge now gated behind `experimental.coldCacheBadge`** ([#95169](https://github.com/vercel/next.js/pull/95169) — the persistent "Cold cache" badge added in canary.57 is now off by default; transient "Rendering (cold cache)" pill during navigation is unchanged; `process.env.__NEXT_EXPERIMENTAL_COLD_CACHE_BADGE` env var plumbed via define plugin), **`partialPrefetching` Shell Prefetch simulation — reveal after `ShellRuntime`** ([#95149](https://github.com/vercel/next.js/pull/95149) — dev-only cold-cache signal now counts misses up to the App Shell boundary, not the runtime tree behind it; `revealAfterStage`/`holdStreamUntilRevealed` collapsed into a single `DevNavigationKind` object)); **16.3.0-canary.65** released June 24, 2026 (Cache Components docs clarify `allow-runtime` / sync-IO / `instant = false` / CLS fallback ([#94997](https://github.com/vercel/next.js/pull/94997)), `cacheMaxMemorySize: 0` no longer forces a dynamic cache life in dev so a fully cached route can prerender again ([#95100](https://github.com/vercel/next.js/pull/95100)), `getHeaders` no longer mutates `req.headers` when stripping internal headers — fixes a dev instant-navigation regression where the React dev overlay stayed in "Rendering..." after a server action redirect because the wrong HTML request id was fabricated from a missing request id ([#95116](https://github.com/vercel/next.js/pull/95116))); **16.3.0-canary.67** released June 24–25, 2026 (Restore canary version 16.3.0-canary.66 after v16.3.0-preview.4 preview release, **fix local fonts in statically prerendered `ImageResponse` metadata route** ([#95121](https://github.com/vercel/next.js/pull/95121) — Node `Buffer.toJSON` round-trip corruption, follow-up to #94957; cache key is now SHA-256 of serialized element + content hash of options, font bytes hashed by their bytes, options paired in-memory to avoid Flight round-trip), **docs(`root-params`): `generateStaticParams` section and CC requirement** ([#95073](https://github.com/vercel/next.js/pull/95073) — with `cacheComponents: true`, every root param must have a value in `generateStaticParams` or the build fails), **surface an error for blocking routes under the Navigation Inspector** ([#95139](https://github.com/vercel/next.js/pull/95139) — empty static shell detection, clears the instant-navigation cookie, no more "blank document, no DevTools, can't reload out" footgun), **suppress `prefetch={true}` warning when route opts out via `instant = false`** ([#95099](https://github.com/vercel/next.js/pull/95099) — new `SubtreeHasInstantFalse` `PrefetchHint` propagated to root, dev warning skips routes with this bit, warning-only and separate from `PrefetchDisabled`), **`next-cache-components-adoption` skill update** ([#95122](https://github.com/vercel/next.js/pull/95122) — leads with `next-dev-loop` recommendation, adds build-only verification path for CI/sandboxed agents, hoists skill pointer to top-level `## Use the adoption skill (recommended)` H2 in `migrating-to-cache-components.mdx`)); **16.3.0-canary.66** released June 24, 2026 (Revert "Remove legacy PPR codepaths" ([#95113](https://github.com/vercel/next.js/pull/95113) — "Builder isn't ready. Need to figure out what config it is actually reading from."; the `experimental.ppr` config + `isAppPPREnabled` + `__NEXT_PPR` are BACK, error code #1375 is GONE, `remove-experimental-ppr` codemod is reverted; the skill's "use `cacheComponents: true` for all PPR behavior" guidance still holds), `<Link prefetch={true}>` Partial Prefetching warning wired into Instant Insights via shared `instant-messages.ts` factory ([#94798](https://github.com/vercel/next.js/pull/94798) — code frame + call stack + three fix cards: Upgrade / Disable / Ignore), Preview tarballs available while new canaries publish ([#95112](https://github.com/vercel/next.js/pull/95112)), ad-hoc preview release cut from canary ([#95086](https://github.com/vercel/next.js/pull/95086) — preview releases now cut from canary with bump-to-preview + revert-to-canary commits, base = max(canary, @preview) version), Fix Navigation Inspector styles in dark mode ([#95126](https://github.com/vercel/next.js/pull/95126))); **16.3.0-canary.64** released June 24, 2026 (Turbopack single-entry chunks ([#94727](https://github.com/vercel/next.js/pull/94727)), consider merging chunks when `overlap == 1` ([#95102](https://github.com/vercel/next.js/pull/95102)), `TURBOPACK_DEBUG_CSS_CHUNKING` env var dumps `turbopack-css-chunking-debug-<ts>.json` snapshots of the `experimental.cssChunking: "graph"` pipeline ([#95080](https://github.com/vercel/next.js/pull/95080)), CD pipeline now publishes on release tags instead of branch push ([#95085](https://github.com/vercel/next.js/pull/95085))); **16.3.0-canary.63** released June 24, 2026 (ISR fallback shells served to prefetch requests with bounded client retries that only fire on segments that can actually upgrade ([#94534](https://github.com/vercel/next.js/pull/94534)), drop `generateStaticParams` fix card and filter the `use cache` card for `connection()` in Dev Insights ([#94926](https://github.com/vercel/next.js/pull/94926)), Navigation Inspector UI rewritten around the new "Pause on navigations" toggle ([#94959](https://github.com/vercel/next.js/pull/94959)), `next-cache-components-adoption` prereq section restructured into a labeled checklist ([#95082](https://github.com/vercel/next.js/pull/95082)), partial-prefetching routes — `prefetch = 'partial'` / `'unstable_eager'` or `experimental.partialPrefetching: true` — now opt into the runtime stage of Cached Navs ([#95097](https://github.com/vercel/next.js/pull/95097)), caching docs explicitly call out `generateStaticParams` and recommend `revalidate = 30 days` for CMS-driven content ([#95081](https://github.com/vercel/next.js/pull/95081)), dev fallback `params` computed from the most-specific matching prerendered route via `getRouteRegex` ([#95066](https://github.com/vercel/next.js/pull/95066)), instant-navs test lock released without `clearCookies` so the cookie jar is briefly empty ([#94947](https://github.com/vercel/next.js/pull/94947)), linktime `scattered-collect` fix for the Windows miscompile ([#95098](https://github.com/vercel/next.js/pull/95098))); **16.3.0-canary.62** released June 23, 2026 (marks `insight-error-page` and `next-rspack` skills as internal [#95070], Turbopack `clone`-avoidance on module sort by path [#95079], false-positive `export const dynamic` in Cache Components detection fix [#95083], new **Interactive Apps** guide [#94020]); **16.3.0-preview.4** released June 24, 2026 ([`pnpm install` no longer auto-installs native bindings](https://github.com/vercel/next.js/pull/95114) — CI almost exclusively wants bindings built from the current commit; opt-in to automatic native binding install in CI is preserved for `test_examples` only); **16.3.0-preview.5** released June 25, 2026 ([`70543c5`](https://github.com/vercel/next.js/releases/tag/v16.3.0-preview.5) — `instant()` only renders the static shell unless `{ prefetch: true }` is passed ([#95150](https://github.com/vercel/next.js/pull/95150) — `<Link prefetch>` is unaffected, custom integration code that expected the full payload must now pass `prefetch: true`), dev now replicates production shell-only prefetch behavior ([#95067](https://github.com/vercel/next.js/pull/95067) — closes a long-standing dev/prod discrepancy that masked shell-only correctness bugs until ship-time), `next-dev-loop` papercut fixes ([#95153](https://github.com/vercel/next.js/pull/95153)), docs: expand `io()` reference ([#95147](https://github.com/vercel/next.js/pull/95147)))
- **React 19.2.7** (React Compiler 1.0 stable, `use()` hook, `useOptimistic`, `useFormStatus`, `useActionState`, `useEffectEvent`, `cacheSignal`, `cache`, `<Activity>`)
- **TypeScript 6.0.3** (strict by default, ES2026 target, import defer) + **TypeScript 7.0.1-rc** (June 18, 2026, RC release of Go-based compiler ships as the main `typescript` package — `npx tsc` is now Go, ~10× faster, Strada internal API; stable targeted within 2 months of RC)
- **Zod 4.4.3** (14x faster string parsing, strict/loose object modes, `z.file()`, `z.templateLiteral()`)
- **Tailwind CSS v4.3.1** + **shadcn/ui** (CSS-first config via `@theme` directive — no tailwind.config.js by default; v4.3.1 patch added `--silent` CLI flag, `@apply` with CSS mixins, cleaner spacing output)
- **Modern CSS 2026** — `field-sizing: content` (Baseline 2026, Firefox 152 just shipped), `@starting-style` (Baseline 2024), CSS Anchor Positioning (cross-engine 2025–26), OKLCH color space, `color-scheme` + `light-dark()` function, scroll-driven animations
- **Vite 8.1.0** (for non-Next projects; Next uses Turbopack — 8.1 ships WASM ESM Integration, `server.hmr` → `server.ws` rename, `import.meta.glob` `caseSensitive`, chunk importmap, lightningcss support, extended `server.fs.deny` defaults; **≥ 8.1.0 required for the new `caseSensitive` glob option**)
- **@biomejs/biome 2.5.1** (recommended linter/formatter — 10–100x faster than ESLint, v2 has breaking changes from v1; run `npx biome migrate --write` after every upgrade)
- **TanStack Query v5.101.1** (React Query v5 — gcTime replaces cacheTime, improved SSR hydration, `skipToken` for dependent queries)
- **Vitest 4.1.9** (Browser Mode stable, Visual Regression testing via `toMatchScreenshot`, Playwright Trace support; requires Vite ≥ 6 + Node.js ≥ 20; **≥ 4.1.8 required for the CDP RCE fix (GHSA-g8mr-85jm-7xhm, CVSS 9.8)**; use `api.allowWrite: false, api.allowExec: false` in CI; **Vitest 5.0 in beta** — three beta releases between May 19 and June 15, 2026 (beta.3, beta.4, beta.5) introduced hard requirement Node 22 + Vite 6.4, strict `toHaveTextContent` + new `toMatchTextContent`, Browser Mode `locators.exact: true` default, `expect.poll` fails on timeout, hoistable-methods-out-of-scope throw, removed entry points (`vitest/coverage` → `vitest/node`, `vitest/environments` / `vitest/snapshot` → `vitest/runtime`, `vitest/runners` / `vitest/suite` → `TestRunner` from `vitest`, `vitest/mocker` → `@vitest/mocker`), `bench` moved into `test()` fixture, `test.sequential` → `{ concurrent: false }`, no ancestor-directory config lookup, `@vitest/runner` inlined into `vitest`, `TestModule.id` 1-based, `coverage.thresholds.perFile` accepts an object, browser orchestrator requires `sessionId`)
- **@playwright/test 1.61.1** (E2E — **1.61.0 (June 15, 2026)** shipped WebAuthn passkeys virtual authenticator via `browserContext.credentials.create/install`, `page.localStorage` / `page.sessionStorage` (origin-aware, replaces `page.evaluate('localStorage…')` boilerplate), `apiResponse.securityDetails()` / `serverAddr()` network parity with browser responses, `testOptions.video` modes parity with trace (`'on-all-retries'`, `'retain-on-first-failure'`, `'retain-on-failure-and-retries'`), `expect.soft.poll(...)`, `fullConfig.argv`, `fullConfig.failOnFlakyTests`, `testInfo.errors` AggregateError sub-entries, `-G` shorthand for `--grep-invert`, HAR + trace now include WebSocket frames, Ubuntu 26.04 host support, browsers Chromium 149.0.7827.55 / Firefox 151.0 / WebKit 26.5; **1.61.1 (June 23, 2026)** fixed five 1.61 regressions — `expect.extend` matcher-name shadowing built-ins, UI-mode API request byte count mismatch, trace-viewer WebSocket timestamps scaled by 1/1000, sync ESM loader crash on Node 22.15, pnpm workspace symlink `.ts` subpath resolution)
- **React Hook Form v7.80.0** + **@hookform/resolvers v5.4.0** (7.80.0 ships per-field `disabled` on `useFieldArray` items; broad perf pass; fixes `[]` vs `{}` `deepEqual` regression from 7.79.0) (compatible with Zod v4; v8.0.0-beta available with `createForm` API)
- **Node.js 24 LTS** (Node.js 22 LTS also supported)

## Pro Tips

- Use `create-next-app --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"` for a fully typed Next.js project
- Run `npx tsc --noEmit` to type-check without building
- Use `shadcn@latest add <component>` to add components — they live in your codebase, not node_modules
- Use `next.config.ts` for all Next.js configuration (not `.js`)
- Use `react-hook-form` + `zod` for all forms — never manage form state with useState
- Enable React Compiler in `next.config.ts` with `reactCompiler: true` — eliminates most `useMemo`/`useCallback` manually
- Use Biome (`npx biome check`) for linting — 10–100x faster than ESLint
- Run `npx biome migrate --write` after every Biome upgrade to migrate your config for breaking changes
- Use `use cache` for all server-side data fetching in Next.js 16 — it's the explicit opt-in caching model

Start with `setup.md` to initialize a project, then `components.md` for shadcn/ui patterns.

## Next.js 16 Migration Notes

Key breaking changes from Next.js 15 → 16:

| Change | Impact |
|---|---|
| **`use cache` directive** replaces `unstable_cache` | New explicit caching API — mark data functions with `use cache` + `cacheTag` |
| **Implicit caching removed** — everything is dynamic by default | Old `fetch` caching patterns deprecated; use `use cache` explicitly |
| **PPR (Partial Prerendering) stable** — `cacheComponents: true` | Static shell + streaming dynamic content; no longer experimental |
| **Router scroll optimization** enabled by default | Previously scroll was reset on navigation; now preserved by default |
| Flat config default in `@next/eslint-plugin-next` | ESLint config format change |
| Deprecated `.turbo` config object removed | Use `turbopack` key in `next.config.ts` instead |
| `publicRuntimeConfig` / `serverRuntimeConfig` removed | Use environment variables directly |
| **`next lint` removed** | Use Biome (`npx biome check`) or ESLint (`npx eslint .`) directly |
| **`middleware.ts` deprecated → `proxy.ts`** | `middleware` export renamed to `proxy`; must be `async`; `matcher` moved to `next.config.ts` or exported from `proxy.ts` |

**Upgrade:** `npx @next/codemod@canary upgrade latest` (Next.js 16.2+) — automated upgrade CLI that handles codemods, config migrations, and breaking changes in one command. Alternatively `npx next upgrade` (interactive, Next.js 16.1+) or `npm install next@latest`.

**Sources:**
- [Next.js 16 release notes](https://nextjs.org/blog/next-16)
- [Next.js `use cache` directive](https://nextjs.org/docs/app/api-reference/directives/use-cache)
- [Next.js `cacheTag`](https://nextjs.org/docs/app/api-reference/functions/cacheTag)
- [Next.js `revalidateTag` — required `profile` argument](https://nextjs.org/docs/app/api-reference/functions/revalidateTag) (May 2026 update)
- [Next.js `cacheLife` profiles reference](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheLife)
- [Next.js 16.2.9 release](https://github.com/vercel/next.js/releases/tag/v16.2.9)
- [Next.js 16.2 — automated upgrade CLI](https://nextjs.org/blog/next-16-2)
- [Next.js 16.2 — AI Improvements (AGENTS.md, browser log forwarding, dev server lock file, next-browser)](https://nextjs.org/blog/next-16-2-ai)
- [Next.js 16.1 — Turbopack File System Caching stable + `next dev --inspect` + Bundle Analyzer](https://nextjs.org/blog/next-16-1)
- [Next.js 16.3 canary — prefetch controls + dedup](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.26)
- [Tailwind CSS v4.3.1 release notes](https://github.com/tailwindlabs/tailwindcss/releases/tag/v4.3.1)
- [shadcn/ui — GitHub Registries (June 2026)](https://ui.shadcn.com/docs/changelog/2026-06-github-registries)
- [shadcn eject (May 2026) — take ownership of registry CSS](https://ui.shadcn.com/docs/changelog)
- [React Hook Form 7.79.0 — `useFieldArray` `disabled` option (June 13, 2026)](https://github.com/react-hook-form/react-hook-form/releases/tag/v7.79.0)
- [React Hook Form 7.80.0 — per-field `disabled` + perf + `deepEqual` fix (June 20, 2026)](https://github.com/react-hook-form/react-hook-form/releases/tag/v7.80.0)
- [React Compiler 1.0 stable release](https://react.dev/blog/2025/10/07/react-compiler-1)
- [Next.js React Compiler integration](https://nextjs.org/docs/app/api-reference/config/next-config-js/reactCompiler)
- [Vercel research: AGENTS.md outperforms skills (100% vs 79%)](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals)
- [Vitest 4.0 announcement (Oct 21, 2025) — Browser Mode stable + Visual Regression + Playwright Trace](https://voidzero.dev/posts/announcing-vitest-4)
- [Vitest Visual Regression Testing docs](https://vitest.dev/guide/browser/visual-regression-testing)
- [Vitest 3 → 4 migration guide](https://vitest.dev/guide/migration.html)
- [GHSA-g8mr-85jm-7xhm — Vitest Browser Mode CDP RCE (CVSS 9.8, June 1, 2026)](https://github.com/vitest-dev/vitest/security/advisories/GHSA-g8mr-85jm-7xhm)
- [GHSA-2h32-95rg-cppp — Vitest otelCarrier XSS → RCE (CVSS 9.6, May 19, 2026)](https://github.com/vitest-dev/vitest/security/advisories/GHSA-2h32-95rg-cppp)
- [GHSA-5xrq-8626-4rwp — Vitest UI arbitrary file read + RCE on Windows (CVSS 9.8, May 19, 2026)](https://github.com/vitest-dev/vitest/security/advisories/GHSA-5xrq-8626-4rwp)
- [Vitest browser.api config — allowWrite / allowExec (4.1.0+)](https://main.vitest.dev/config/browser/api)
- [VoidZero is joining Cloudflare (June 4, 2026) — Vite, Vitest, Rolldown, Oxc, Vite+ remain MIT and community-driven](https://blog.cloudflare.com/voidzero-joins-cloudflare/)
- [VoidZero's own announcement (Evan You)](https://voidzero.dev/posts/voidzero-cloudflare)
- [StepSecurity — Mastra npm supply chain attack (June 17, 2026) — 142+ packages, easy-day-js typosquat, easy-day-js@1.11.22 RCE](https://www.stepsecurity.io/blog/mastra-npm-packages-compromised-using-easy-day-js)
- [Snyk — Mastra npm Scope Takeover (June 16, 2026) — forgotten contributor account, postinstall dropper, RAT + crypto stealer](https://snyk.io/blog/a-forgotten-contributor-account-compromised-the-entire-mastra-npm-package-scope/)
- [Node.js — Thursday, June 18, 2026 Security Releases (12 CVEs, 2 HIGH: CVE-2026-48933 WebCrypto DoS + CVE-2026-48618 TLS wildcard bypass via Unicode normalization)](https://nodejs.org/en/blog/vulnerability/june-2026-security-releases)
- [Next.js docs — Data fetching: Server Actions security (public POST endpoints)](https://nextjs.org/docs/app/guides/data-security)
- [Next.js docs — Authentication: Server Actions (return null in layout is not recommended)](https://nextjs.org/docs/app/guides/authentication)
- [BuildMVPFast — Server Actions Security: Real Vulnerabilities (June 18, 2026)](https://www.buildmvpfast.com/blog/nextjs-server-actions-security-vulnerabilities-2026)
- [Makerkit — Server Actions Security: 5 Vulnerabilities (Next 16.2.6)](https://makerkit.dev/blog/tutorials/secure-nextjs-server-actions)
- [Vercel Ship 2026 recap (June 17, 2026) — Agent Stack, Vercel Connect, Vercel Agent (private beta)](https://vercel.com/blog/vercel-ship-2026-recap)
- [Vercel Connect — scoped, short-lived tokens for agents (June 17, 2026)](https://vercel.com/blog/introducing-vercel-connect)
- [ProjectDiscovery — The Vulnerability Curve Bent With the AI Curve (June 18, 2026)](https://projectdiscovery.io/blog/the-vulnerability-curve-bent-with-the-ai-curve)
- [Authgear — Next.js Security Best Practices 2026](https://www.authgear.com/post/nextjs-security-best-practices/)
- [pnpm — minimumReleaseAge (supply-chain defense, blocks short-lived malicious publishes)](https://pnpm.io/supply-chain-security)
- [OpenAI — Our response to the TanStack npm supply chain attack (May 13, 2026)](https://openai.com/index/our-response-to-the-tanstack-npm-supply-chain-attack/)

- [Playwright 1.61.0 release notes (June 15, 2026)](https://github.com/microsoft/playwright/releases/tag/v1.61.0)
- [Playwright 1.61.1 release notes — 5 regression fixes (June 23, 2026)](https://github.com/microsoft/playwright/releases/tag/v1.61.1)
- [Playwright docs — Credentials (WebAuthn virtual authenticator)](https://playwright.dev/docs/api/class-credentials)
- [Playwright docs — WebStorage (`page.localStorage`, `page.sessionStorage`)](https://playwright.dev/docs/api/class-webstorage)
- [Playwright docs — `fullConfig.argv` / `fullConfig.failOnFlakyTests`](https://playwright.dev/docs/api/class-fullconfig)
- [Playwright docs — video modes (1.61 parity with trace modes)](https://playwright.dev/docs/test-use-options#video-modes)
- [Playwright release notes page (1.61 + 1.61.1)](https://playwright.dev/docs/release-notes)

- [Digital Applied — Node.js June 2026 Security Releases: 12 CVEs, 2 HIGH (full list + patch guide)](https://www.digitalapplied.com/blog/nodejs-june-2026-security-releases-cve-patch-guide)
- [GitHub Advisory: undici WebSocket client DoS via fragment count bypass (CVE-2026-11525, June 19, 2026)](https://github.com/advisories?query=undici+type%3Areviewed)
- [GitHub Advisory: undici HTTP header injection via Set-Cookie percent-decoding (CVE-2026-12151, June 19, 2026)](https://github.com/advisories?query=undici+type%3Areviewed)
- [GitHub Advisory: undici Set-Cookie SameSite attribute downgrade (CVE-2026-9679, June 19, 2026)](https://github.com/advisories?query=undici+type%3Areviewed)
- [undici on npm (current: 8.5.0, 7.28.0 LTS)](https://www.npmjs.com/package/undici)
- [Chat SDK: bring agents to your users (Vercel blog, March 19, 2026)](https://vercel.com/blog/chat-sdk-brings-agents-to-your-users)
- [Chat SDK on npm (chat@4.31.0, June 16, 2026)](https://www.npmjs.com/package/chat)
- [Chat SDK docs (chat-sdk.dev)](https://chat-sdk.dev/docs)
- [Chat SDK adapter directory (10+ official + vendor-official adapters)](https://chat-sdk.dev/adapters)
- [Introducing eve — Vercel's open-source agent framework (June 17, 2026) — durable execution + sandbox + approvals + subagents + evals](https://vercel.com/blog/introducing-eve)
- [Vercel Ship 2026 recap — Agent Stack, eve, Vercel Connect, Vercel Agent (private beta), Claude Managed Agents](https://vercel.com/blog/vercel-ship-2026-recap)
- [TanStack Query v5 — Suspense guide](https://tanstack.com/query/v5/docs/framework/react/guides/suspense)
- [TanStack Query v5 — `useSuspenseQuery` reference](https://tanstack.com/query/v5/docs/framework/react/reference/useSuspenseQuery)
- [TanStack Query v5 — `useSuspenseInfiniteQuery` reference](https://tanstack.com/query/v5/docs/framework/react/reference/useSuspenseInfiniteQuery)
- [TanStack Query v5 — `useSuspenseQueries` reference](https://tanstack.com/query/v5/docs/framework/react/reference/useSuspenseQueries)
- [Next.js docs — Prefetching guide (16.2.9)](https://nextjs.org/docs/app/guides/prefetching)
- [Next.js docs — Parallel Routes file convention (16.2.9 — `default.tsx` required)](https://nextjs.org/docs/app/api-reference/file-conventions/parallel-routes)
- [Next.js 16 release notes — Layout deduplication + Incremental prefetching](https://nextjs.org/blog/next-16)
- [LogRocket — I tested every major auth library for Next.js in 2026 (April 20, 2026)](https://blog.logrocket.com/best-auth-library-nextjs-2026/)
- [Aikido Security — Multiple JetBrains IDE plugins caught stealing AI keys (June 16, 2026 — full IOC list, 15 plugins, 7 banned publisher accounts)](https://www.aikido.dev/blog/multiple-jetbrains-ide-plugins-caught-stealing-ai-keys)
- [JetBrains Marketplace Ecosystem Security Update — Addressing Malicious Third-Party AI Plugins (official disclosure, June 17, 2026)](https://blog.jetbrains.com/platform/2026/06/marketplace-ecosystem-security-update-malicious-ai-plugins)
- [Threat-Modeling.com — JetBrains Marketplace Malicious Plugins Stealing AI API Keys (June 17, 2026)](https://threat-modeling.com/jetbrains-marketplace-malicious-plugins-ai-key-theft-june-2026/)
- [OffSeq Threat Radar — 15 JetBrains Marketplace plugins quietly stealing developers\' AI API keys (~70,000 installs, June 17, 2026)](https://radar.offseq.com/threat/15-jetbrains-marketplace-plugins-were-quietly-stea-8bacd71f)
- [SANS Stormcast Thursday June 18, 2026 — JetBrains Plugins segment](https://isc.sans.edu/podcastdetail/9978)
- [JetBrains Marketplace Approval Guidelines (the manual review process that failed)](https://plugins.jetbrains.com/docs/marketplace/jetbrains-marketplace-approval-guidelines.html#approval-process)
- [Next.js 16.3.0-canary.65 release notes (June 24, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.65)
- [Next.js 16.3.0-canary.64 release notes (June 24, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.64)
- [Next.js 16.3.0-canary.61 release notes (June 22–23, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.61)
- [Next.js 16.3.0-canary.62 release notes (June 23, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.62)
- [Vite 8.1.0 release notes + CHANGELOG (June 23, 2026)](https://github.com/vitejs/vite/releases/tag/v8.1.0)
- [Vite 8.1.0 CHANGELOG.md (raw)](https://github.com/vitejs/vite/blob/v8.1.0/packages/vite/CHANGELOG.md)
- [React docs — `useFormStatus`](https://react.dev/reference/react-dom/hooks/useFormStatus)
- [React docs — `useId`](https://react.dev/reference/react/useId)
- [React docs — `useEffectEvent`](https://react.dev/reference/react/useEffectEvent)
- [React docs — Separating events from effects](https://react.dev/learn/separating-events-from-effects)
- [tkdodo — Ref Callbacks, React 19 and the Compiler](https://tkdodo.eu/blog/ref-callbacks-react-19-and-the-compiler)
- [Matt Pocock — Const type parameters bring 'as const' to functions](https://www.totaltypescript.com/const-type-parameters)
- [learningtypescript.com — Branded Types](https://www.learningtypescript.com/articles/branded-types)
- [ferreira.io — Opaque / Branded Types in TypeScript](https://ferreira.io/posts/opaque-branded-types-in-typescript)
- [TypeScript 5.0 release notes — `const` Type Parameters](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html)
- [TypeScript 4.9 release notes — `satisfies`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html)
- [TypeScript docs — Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html)
- [PR #95064 — Widen `cachedNavigations` to `boolean | 'allow-runtime'` (canary.61)](https://github.com/vercel/next.js/pull/95064)
- [PR #94941 — `cache-components-instant-false` codemod + `next-cache-components-adoption` skill (canary.61)](https://github.com/vercel/next.js/pull/94941)
- [Next.js `next-cache-components-adoption` skill (canary source)](https://github.com/vercel/next.js/tree/canary/skills/next-cache-components-adoption)
- [PR #94955 — Remove legacy PPR codepaths (canary.61, REVERTED in canary.66)](https://github.com/vercel/next.js/pull/94955)
- [PR #95113 — Revert "Remove legacy PPR codepaths" (canary.66)](https://github.com/vercel/next.js/pull/95113)
- [PR #94957 — Statically prerender metadata image routes under Cache Components (canary.61)](https://github.com/vercel/next.js/pull/94957)
- [PR #95121 — Fix local fonts in statically-prerendered `ImageResponse` metadata route (canary.67, follow-up to #94957)](https://github.com/vercel/next.js/pull/95121)
- [PR #95073 — docs(root-params): `generateStaticParams` section and CC requirement (canary.67)](https://github.com/vercel/next.js/pull/95073)
- [PR #95099 — Suppress `prefetch={true}` warning when route opts out via `instant = false` (canary.67)](https://github.com/vercel/next.js/pull/95099)
- [PR #95139 — Surface an error for blocking routes under the Navigation Inspector (canary.67)](https://github.com/vercel/next.js/pull/95139)
- [PR #95122 — `next-cache-components-adoption` skill leads with `next-dev-loop` + build-only path (canary.67)](https://github.com/vercel/next.js/pull/95122)
- [PR #95114 — `[ci]` Make automatic native binding install opt-in (canary.66 / preview.4)](https://github.com/vercel/next.js/pull/95114)
- [Next.js 16.3.0-canary.66 release notes (June 24, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.66)
- [Next.js 16.3.0-canary.67 release notes (June 24–25, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.67)
- [Next.js 16.3.0-canary.68 release notes (June 25, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0-canary.68)
- [PR #94920 — `[turbopack]` Create `ServiceWorkerChunkingContextOptions` in `next-core` (canary.68)](https://github.com/vercel/next.js/pull/94920)
- [PR #94921 — `[turbopack]` Create `ServiceWorkerEntryModule` and `service_worker_chunk_filename` (canary.68)](https://github.com/vercel/next.js/pull/94921)
- [PR #94922 — `[turbopack]` Discover `ServiceWorkerEntryModule`s in `next-api` and compile + serve those service workers (canary.68)](https://github.com/vercel/next.js/pull/94922)
- [PR #94924 — `[turbopack]` Add e2e tests for service workers (canary.68, still open)](https://github.com/vercel/next.js/pull/94924)
- [PR #95169 — Gate the dev Cold cache badge behind an experimental flag (canary.68)](https://github.com/vercel/next.js/pull/95169)
- [PR #95149 — `[PP]` Reveal after ShellRuntime when simulating a Shell Prefetch in dev (canary.68)](https://github.com/vercel/next.js/pull/95149)
- [PR #95177 — `[next-dev-loop]` Fix cookies (canary.68)](https://github.com/vercel/next.js/pull/95177)
- [vercel-labs/agent-browser PR #1486 — fixed session flakiness (drives #95177)](https://github.com/vercel-labs/agent-browser/pull/1486)
- [Next.js 16.3.0-preview.4 release notes (June 24, 2026)](https://github.com/vercel/next.js/releases/tag/v16.3.0-preview.4)
- [PR #94726 — Turbopack `edge` runtime → `self-contained` rename (canary.61)](https://github.com/vercel/next.js/pull/94726)
- [PR #94897 — Cache components requires Node.js runtime (docs, canary.61)](https://github.com/vercel/next.js/pull/94897)
- [Next.js docs — Migrating to Cache Components](https://nextjs.org/docs/app/guides/migrating-to-cache-components) (updated to mention `next-cache-components-adoption` skill)
- [Next.js docs — Codemods reference (16.3)](https://nextjs.org/docs/app/guides/upgrading/codemods) (now lists `cache-components-instant-false`)
