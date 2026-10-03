# frontend/CLAUDE.md — Next.js PLP

Applies to everything under `frontend/`. The root `CLAUDE.md` rules still apply. Architecture is defined in `docs/frontend-plan.md`; read the relevant section before each task.

## Stack and allowed dependencies

Next.js 15 (App Router) · React · TypeScript strict · CSS Modules · deployed on **Vercel** (region `iad1`).

| Allowed | Packages |
|---------|----------|
| Runtime | `next`, `react`, `react-dom`, `server-only` |
| Dev | `typescript`, `@types/node`, `@types/react`, `@types/react-dom`, `eslint`, `eslint-config-next`, `prettier` |

Anything else (Tailwind, UI kits, state libraries, `clsx`, `axios`, icon packages, date libraries) is **not allowed**. Ask first.

## Commands

```bash
npm run dev         # local dev server (needs API_URL in .env.local)
npm run lint
npm run typecheck   # tsc --noEmit
npm run build       # must pass before reporting a task done
```

## Rendering rules (graded: SSR and performance, 15%)

1. **Server Components by default.** The product grid, cards, pagination, breadcrumb, header and footer are server components.
2. **`'use client'` only in these components:** `FilterForm`, `SortMenu` (enhancement only), `FilterToggle`, `WishlistButton`, `MobileNav`, `NewsletterForm`, `error.tsx`. Adding `'use client'` anywhere else needs approval.
3. **No data fetching in the browser.** No `useEffect` + `fetch`, and no client calls to the API. Client components change the URL with `router.push(url, { scroll: false })` inside `startTransition`, and the server re-renders.
4. **All API calls go through `src/lib/api.ts`** (`import 'server-only'`, 8s timeout, typed errors). Never call `fetch` on the API anywhere else.
5. **The URL is the state.** Read and build listing URLs only through `src/lib/search-params.ts` (`parseSearchParams`, `serialise`, `withChange`). Never concatenate query strings by hand.
6. `searchParams` is a Promise in Next 15: `const params = await searchParams;`.
7. Wrap the results in `<Suspense key={serialise(state)}>` so the skeleton shows on every change. Keep that line's why-comment.
8. Progressive enhancement: filters are a real `<form method="get">`, and sort options and pagination are real links. Everything must work with JS disabled.

## Styling rules

- CSS Modules only: `Component.tsx` next to `Component.module.css`. Global CSS only in `app/globals.css` (reset + tokens).
- Use tokens (`var(--color-text)`, `var(--space-4)`) from `globals.css`; no raw hex colours or pixel values in modules unless commented with a Figma reference.
- Mobile-first media queries using the token breakpoints (`40rem` tablet, `64rem` desktop).
- Keep the DOM minimal: no wrapper `<div>`s just for styling when the parent element can take the class.
- Never cause horizontal scroll: `min-width: 0` on grid children, `max-width: 100%` on media.

## Markup, accessibility and SEO rules (graded: SEO 10%, responsive 10%)

- Exactly **one `<h1>`** per page. "Products" and "Filters" are `<h2>`; product names are `<h3>`. Footer column titles are never `<h1>`.
- Semantic elements: `<header>`, `<nav aria-label>`, `<main id="main">`, `<aside>`, `<article>` per card, `<footer>`, `<form role="search">`.
- Every input has a `<label>`. Checkbox groups use `<fieldset>` + `<legend>`. Icon-only buttons have `aria-label`.
- Accordions use `<details>`/`<summary>`. The mobile filter drawer uses `<dialog>`.
- `next/image` for every product image: explicit `width`/`height` from the API, `sizes` matching the grid, `priority` on the first 4 cards only, and `alt` from the API (never empty for products).
- Metadata only via `generateMetadata` and the helpers in `src/lib/seo.ts`. JSON-LD only via the builders in `src/lib/json-ld.ts`.
- Follow the canonical policy in `docs/frontend-plan.md` §9 exactly.

## Design rules

- The Figma is the visual reference. Deliberate deviations are listed in `docs/product.md` §8 (price shown, Category + Price filters added, pagination added). Do not invent other deviations; ask.
- Use Figma's wording for labels, e.g. "Recommended", "Newest first", "Popular", "Price : high to low". Fix only the design's typos ("Customizble" → "Customizable", "low ot high" → "low to high").

## File conventions

- Components: `PascalCase.tsx`; lib files: `kebab-case.ts`; one component per file.
- Props type named `<Component>Props`, declared right above the component.
- Component order inside a file: file header comment → imports → types → constants → component → small private helpers.
