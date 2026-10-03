# Frontend Plan — Next.js PLP

## 1. Stack

| Choice | Why |
|--------|-----|
| Next.js 15, App Router | Server Components give SSR by default; the brief prefers it |
| TypeScript (strict) | Typed API contracts shared with the backend's response shapes |
| CSS Modules + CSS custom properties | Zero runtime, scoped styles, no extra dependency; design tokens live in one file |
| `next/font` | Self-hosted fonts, no layout shift, no external request |
| `next/image` | Responsive `srcset`, lazy loading, WebP/AVIF |

Runtime dependencies are `next`, `react` and `react-dom` only. Dev dependencies are `typescript`, `eslint` and `prettier`. No Tailwind, no UI kit, no state library: the URL is the state.

## 2. Phase 1 — Static HTML/CSS (`/static`)

1. Inspect the Figma and extract tokens (colors, type scale, spacing, radii, breakpoints) into `static/tokens.css`.
2. Build `static/index.html` with semantic markup only: `header`, `nav`, `main`, `section`, `article`, `footer`. Use 8–12 hard-coded product cards.
3. Build `static/styles.css`, mobile-first, with Grid for the product grid and Flexbox for bars.
4. Check widths 320, 375, 768, 1024 and 1440 for no horizontal scroll.
5. **Commit:** `feat(static): implement PLP design in plain HTML/CSS`.

The tokens and markup carry over directly into the Next.js components. The static step is the design spike, not throwaway work.

## 3. Folder structure

```
frontend/
├── src/
│   ├── app/
│   │   ├── layout.tsx            # html/body, fonts, Header, Footer, skip link
│   │   ├── page.tsx              # PLP (server component)
│   │   ├── loading.tsx           # skeleton for first load
│   │   ├── error.tsx             # error boundary with retry (client)
│   │   ├── not-found.tsx
│   │   ├── globals.css           # reset + tokens
│   │   └── products/[idSlug]/page.tsx   # PDP (stretch)
│   ├── components/
│   │   ├── layout/   AnnouncementBar, Header, MobileNav, Footer, NewsletterForm
│   │   ├── plp/      Breadcrumb, ProductGrid, ProductCard, ProductGridSkeleton,
│   │   │             WishlistButton, FilterToggle, FilterPanel, FilterForm,
│   │   │             FacetGroup, PriceFilter, SortMenu, Pagination,
│   │   │             ResultCount, EmptyState
│   │   └── ui/       Icon (inline SVG sprite), VisuallyHidden
│   ├── lib/
│   │   ├── api.ts            # server-only fetch wrapper (timeout, errors, types)
│   │   ├── search-params.ts  # parse / normalise / serialise URL state
│   │   ├── seo.ts            # metadata + canonical builders
│   │   └── json-ld.ts        # ItemList, Product, BreadcrumbList builders
│   └── types/api.ts          # Product, Category, Paginated<T>, ApiError
├── public/                   # favicon, og-image
├── .env.example              # API_URL, SITE_URL
└── next.config.ts            # images.remotePatterns → API host
```

Each `.tsx` component sits next to its `.module.css`. Components are PascalCase, files in `lib/` are kebab-case.

## 4. Rendering architecture

```
page.tsx (Server)
 ├─ parseSearchParams(await searchParams)        → normalised ListingState
 ├─ Promise.all: getCategories() (revalidate 3600), getFacets(state)
 ├─ Breadcrumb + <h1> + intro
 ├─ Toolbar (ResultCount, FilterToggle*, SortMenu*)
 ├─ FilterPanel → FilterForm* (customizable, category, price, facet groups with counts)
 └─ <Suspense key={serialise(state)} fallback={<ProductGridSkeleton/>}>
      ProductResults (Server) → getProducts(state)
        ├─ ProductGrid → ProductCard × n     (pure server, zero JS)
        ├─ EmptyState                        (when total = 0)
        ├─ Pagination                        (server, <Link> only)
        └─ <script type="application/ld+json"> ItemList + BreadcrumbList
```
`*` marks client components. They are the only JS shipped beyond the Next.js runtime.

Key decisions:
- **The grid is never a client component.** Products render on the server, so "view source" has real content.
- **`<Suspense key>`** keyed on the serialised state makes every filter, sort or page change show the skeleton instead of stale content.
- **No browser → API calls.** Client components call `router.push(newUrl, { scroll: false })` inside `useTransition`. Next re-runs the server component, which fetches from the API. That means no client-side fetching code, no duplicated state, and almost no CORS surface.
- **Progressive enhancement:** `FilterForm` is a real `<form method="get">`, so it works with JS disabled. With JS, it intercepts changes and pushes the URL; checkboxes apply instantly, while price applies on submit or blur.
- Sort is a custom menu, as designed: a `<details>`/`<summary>` disclosure holding one `<Link>` per option, with a checkmark and `aria-current` on the active one. It works without JS; JS adds Esc-to-close and outside-click. Pagination is plain `<Link>`s.
- The show/hide filter toggle is the only state not in the URL. It's a layout preference, not a listing state, so it is stored in a cookie. The server reads the cookie and renders 3 or 4 columns directly, which avoids a layout flash. The default is shown.

## 5. Data fetching (`lib/api.ts`)

```ts
import 'server-only'; // build-time guard, no runtime cost
const API_URL = process.env.API_URL!; // server-only, never NEXT_PUBLIC_

async function apiGet<T>(path: string, revalidate = 60): Promise<T> {
  const res = await fetch(`${API_URL}${path}`, {
    next: { revalidate },
    signal: AbortSignal.timeout(8000),
  });
  if (!res.ok) throw new ApiError(res.status, await res.json().catch(() => null));
  return res.json();
}
```
- `getProducts(state)` serialises state to API query params.
- `getCategories()` uses `revalidate: 3600`.
- `getFacets(state)` uses `revalidate: 60` and sends the same filter params as `getProducts`.
- `getProduct(id)` maps a 404 to `notFound()`.
- A timeout or a 5xx is thrown and caught by `error.tsx`, which shows a retry button that calls `reset()`.

`server-only` is a tiny guard package. Its justification for the README: it prevents the API URL and fetch logic from ever leaking into the client bundle.

## 6. URL state (`lib/search-params.ts`)

- `parseSearchParams(raw) → ListingState` handles normalisation:
  - Unknown sort becomes `recommended`.
  - Non-numeric price is dropped; if `minPrice > maxPrice`, they are swapped.
  - `page < 1` or non-integer becomes 1.
  - `q` is trimmed and capped at 100 characters.
  - Category and facet value slugs are deduped and sorted, so URLs are deterministic. Unknown facet keys are dropped.
  - `customizable` is kept only when `true`.
- `serialise(state) → string` omits defaults, which gives one canonical URL per state.
- `withChange(state, patch)` resets `page` to 1 whenever anything other than `page` changes.
- Unit-testable pure functions. If tests are added, use Node's built-in `node:test` to avoid a dependency.

## 7. States

| State | Where | Behaviour |
|-------|-------|-----------|
| First load | `loading.tsx` | Full-page skeleton |
| Filter/sort/page change | `<Suspense key>` fallback | Grid skeleton; toolbar and filters stay interactive |
| Pending transition | `useTransition` `isPending` | Subtle opacity on the grid, `aria-busy="true"` |
| Empty | `EmptyState` | "No products match these filters" + Clear filters link to `/` |
| API error / timeout | `error.tsx` | Message + Retry (`reset()`); layout still renders |
| Out-of-range page | page.tsx | `redirect` to the last valid page |
| PDP not found | `notFound()` | `not-found.tsx` |

Skeleton cards reuse the card's grid and aspect ratio, so there is no layout shift.

## 8. Responsive plan

- Mobile-first CSS. Breakpoints `40rem` (tablet) and `64rem` (desktop) are stored as tokens.
- Grid (from Figma): 2 columns on mobile, 3 on tablet, 3 on desktop with filters shown, 4 with filters hidden. The desktop container is 1248px max with 96px side padding; cards are 300px wide with 3:4 images (`aspect-ratio: 3 / 4`).
- Filters:
  - Desktop has a 300px sidebar beside the grid with the "‹ Hide filter" / "Show filter" toggle, as in the design.
  - Tablet and mobile open a `<dialog>` drawer from the designed FILTER | RECOMMENDED split bar. Native focus handling and Esc to close come for free.
- Header: a mobile menu toggle with `aria-expanded`; the desktop nav row hides below the tablet breakpoint.
- Footer: link columns become `<details>` accordions on mobile, as designed.
- Guard rails: `img { max-width: 100% }`, `min-width: 0` on grid children, long titles clamped with `line-clamp`.
- Test at 320, 375, 414, 768, 1024, 1280 and 1440 using real-device emulation.

## 9. SEO

| Requirement | Implementation |
|-------------|----------------|
| Title + description | `generateMetadata` driven by state, e.g. "Men's Shirts — Shop \| Brand", falling back to a generic title |
| Headings | One H1 (page heading). H2 for "Products" (visually hidden if the design has no visible label) and "Filters". H3 per card. Footer column titles styled as `<p>` or H2, never extra H1s |
| JSON-LD | `ItemList` of `ListItem → Product { name, image, url, offers: Offer { price, priceCurrency, availability }, aggregateRating }`, plus `BreadcrumbList` (Home › Shop) on the PLP and on the PDP |
| Canonical | Category filter and page: self-canonical. Sort, price or `q` present: canonical without those params. `q` present: also `robots: noindex, follow` |
| Open Graph / Twitter | `og:title`, `og:description`, `og:url`, `og:image` (static `og-image.png`), `twitter:card` |
| Images | API supplies slug file names and meaningful alt text. `next/image` with `sizes` matching the grid. `priority` on the first 4 cards only; the rest lazy |
| Semantics | `<main>`, `<nav aria-label>`, `<article>` per card, `<form role="search">`, `lang="en"` |
| Crawlability | Pagination and nav are real links. `robots.txt` and `sitemap.ts` (with categories) |

The canonical policy goes in the README "SEO decisions" section, since it's a judgment call evaluators will look for.

## 10. Accessibility checklist

- Skip-to-content link as the first focusable element.
- `:focus-visible` outlines on everything interactive; never `outline: none` without a replacement.
- All inputs labelled. Price inputs use `inputmode="decimal"`. Checkbox groups use `<fieldset>` and `<legend>`.
- Icon-only buttons (wishlist, cart, menu) have `aria-label`.
- Result count uses `aria-live="polite"`. The grid gets `aria-busy` while pending.
- Contrast checked against WCAG AA. If Figma's grey text fails, darken slightly and note it in the README.

## 11. Configuration and deployment

- Env: `API_URL` (server-only) and `SITE_URL` (canonical/OG base). Only `.env.example` is committed.
- `next.config.ts`: `images.remotePatterns` for the API host; `poweredByHeader: false`.
- Vercel: import the repo with Root Directory `frontend`; Next.js is auto-detected. Set `API_URL` and `SITE_URL` in Project → Settings → Environment Variables.
- Set the Vercel function region to US East (`iad1`) to sit next to the Render API (Virginia) and Neon (AWS us-east-1). Otherwise every SSR request crosses continents.
- Vercel's Hobby plan has a limited image-optimisation quota. ~200 product images is well within it, but don't add `unoptimized` as a workaround.
- **Verify on the deployed URL:**
  - `curl -s $SITE | grep "<article"` returns product cards.
  - JSON-LD validates in Google's Rich Results Test.
  - Lighthouse (mobile) scores are recorded in the README.

## 12. Build order

1. Scaffold Next.js, tokens and `globals.css` from `/static`, then layout (Header, Footer, fonts).
2. `types/api.ts` + `lib/api.ts` against the live or local API. Render an unfiltered grid with SSR.
3. `search-params.ts`, then Pagination, then the Sort menu, then the Category, Price, Customizable and facet filters (with counts), then Search.
4. Suspense key + skeleton, `error.tsx`, EmptyState.
5. Responsive pass and filter drawer.
6. SEO: metadata, canonical, JSON-LD, sitemap/robots.
7. A11y pass, Lighthouse pass, deploy, verify SSR in production.
8. PDP (stretch).

Commit after each step, e.g. `feat(plp): add URL-driven category filter`.
