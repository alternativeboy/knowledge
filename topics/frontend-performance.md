---
title: Frontend Performance Notes
date: 2026-07-20
tags: [frontend, performance, react, preact, bundling, code-splitting]
source: own knowledge — not verified against primary sources; check official docs before citing specific numbers
---

# Frontend Performance Notes

Working notes on frontend performance topics. Content here is synthesized from general knowledge, not a primary-source research pass — treat specific benchmark numbers as approximate and verify before relying on them for a decision.

## React vs Preact

Same modern component model (JSX, hooks), very different weight.

| | React (react + react-dom) | Preact |
|---|---|---|
| Size (min+gzip) | ~40-45 KB | ~3-4 KB core, ~4-5 KB with `preact/compat` |
| API | Hooks, JSX, context | Same — `preact/hooks` mirrors the React hooks API |
| Concurrent features | React 18+ concurrent rendering, `useTransition`, streaming SSR, Server Components | Not implemented — Preact has its own simpler SSR (`preact-render-to-string`) |
| Fine-grained reactivity | No built-in signals (React Compiler does build-time auto-memoization instead) | First-class `@preact/signals` — skips diff/re-render entirely for signal-driven updates |
| Ecosystem | Much larger: Next.js, React Query, DevTools maturity, hiring pool | `preact/compat` aliases `react`/`react-dom` so most React libraries work, but not universally (libraries touching React internals can break) |

**Runtime performance**: Preact's VDOM diffing is simpler/lighter than React's Fiber, and has generally benchmarked modestly faster in classic create/update/swap-row benchmarks — though the gap has narrowed as React's scheduler matured. For most real apps, bundle size and hydration cost matter more than diff-speed microbenchmarks.

**Rule of thumb**: size-sensitive or embedded contexts (widgets, PWAs, low-end devices) → Preact. Need Server Components / the newest React concurrent features / biggest ecosystem and hiring pool → React.

## Code splitting

Breaking one large JS bundle into chunks that load on demand, instead of shipping everything upfront. Reduces initial download, parse, and — critically for JS — main-thread compile/execute time.

### Mechanism

Built on native dynamic `import()`, which every modern bundler (Webpack, Rollup, Vite, esbuild, Bun) recognizes as a chunk boundary:

```js
// static — bundled together, eager
import Editor from './Editor';

// dynamic — separate chunk, on demand
const Editor = await import('./Editor');
```

In React/Preact, `lazy()` + `Suspense` wraps this with rendering integration:

```jsx
const Editor = React.lazy(() => import('./Editor'));

<Suspense fallback={<Spinner />}>
  <Editor />
</Suspense>
```

Preact's `preact/compat` exports the same `lazy`/`Suspense` API — code-splitting patterns port over unchanged between the two.

### Where to split (in priority order)

1. **Route-based** — highest ROI, usually near-automatic in modern meta-frameworks. Each page becomes its own chunk.
2. **Component-based** — heavy, non-immediately-needed pieces: modals, rich text editors, charting libs, code editors (Monaco is the classic example).
3. **Vendor/library splitting** — separate large third-party deps into their own chunk(s) for independent caching (app code changes every deploy, vendor code doesn't) — a caching optimization more than a load-time one.

### Tradeoffs

- **More requests** — cheap under HTTP/2+ multiplexing, not free.
- **Waterfalls** — if chunk B is only discovered after chunk A loads, you lose parallelism. Mitigate with prefetch hints (Webpack's `/* webpackPrefetch: true */`, Vite's automatic modulepreload) or framework-level prefetching (e.g. Next.js prefetching `<Link>` targets on viewport/hover).
- **Loading-state proliferation** — every split boundary needs a fallback; over-splitting means a UI full of flickering spinners.
- **SSR complexity** — server needs to know synchronously which chunks a render needs, for preload tags/streaming. Historically solved with `@loadable/component`; modern frameworks (Next.js App Router, RSC) handle it internally.

### Practical approach

Don't split speculatively — profile first with a bundle analyzer (`webpack-bundle-analyzer`, `rollup-plugin-visualizer`) to find what's actually large, then: route-split first → component-split heavy non-critical-path pieces → add prefetch hints so bundle-size wins don't trade in a visible loading stutter.
