# Product Brief — Appscrip PLP

## 1. What we are building

A server-rendered **Product Listing Page (PLP)** for an e-commerce storefront. It is backed by our own **Node.js + Express REST API** and a **PostgreSQL** database, and deployed publicly.

The Figma (a "mettā muse" storefront) has been reviewed. Where it conflicts with the brief, §8 records the decision.

The user lands on the page and sees real products immediately (server-rendered). They can then narrow and reorder the list with filters, sort, search and pagination. Every view is a shareable, crawlable URL.

The core deliverable is one page: the PLP. A minimal Product Detail Page (PDP) is a stretch goal. It would exercise `GET /products/:id` and give the JSON-LD `ItemList` real item URLs.

## 2. Goals, in priority order

Priorities follow the evaluation weights, so effort goes where the marks are.

| # | Goal | Weight | What "done" means |
|---|------|--------|-------------------|
| 1 | Clean structure and naming (both apps) | 20% | Predictable folders, one responsibility per module, consistent names |
| 2 | Solid backend and DB | 20% | Validated inputs, consistent errors, indexed schema, migrations, idempotent seed |
| 3 | Frontend matches Figma, works fully | 15% | Filters, sort and pagination all functional; loading, empty and error states present |
| 4 | Real SSR, lean output | 15% | "View source" shows products; minimal DOM; near-zero extra dependencies |
| 5 | SEO | 10% | One H1, logical H2/H3, JSON-LD, canonical, OG tags, slug image names, alt text |
| 6 | Responsive | 10% | Mobile, tablet and desktop; no horizontal scroll anywhere |
| 7 | Deployment and README | 5% | Working links (no cold-start failures); all 9 README sections |
| 8 | AI usage | 5% | Honest section with a real rejected-output example; `CLAUDE.md` in repo |

## 3. Scope

**In scope**
- Static HTML/CSS version of the design, committed before the Next.js conversion.
- PLP with header, footer, product grid, filters, sort, search, pagination.
- REST API: `GET /products`, `GET /products/:id`, `GET /categories`, `GET /facets`, `GET /health`.
- Postgres schema, migrations, indexes, seed script, locally hosted SEO-named images.
- Swagger docs, README, `CLAUDE.md`.
- Deployment: frontend on Vercel, backend on Render, DB on Neon, all in US-East regions to keep SSR fetches fast.

**Out of scope** (listed under "Known limitations" in the README)
- Cart, checkout, auth, server-side wishlist, newsletter backend, language switching. The wishlist heart toggles in browser storage only; the bag, profile and ENG controls are visual only.
- Admin/CMS for products.
- Infinite scroll. Pagination was chosen because it is crawlable and works without JS.

## 4. Functional specification

### F1 — Header and footer
- Header (per Figma): black announcement strip; logo icon on the left and "LOGO" centred; search, wishlist, bag and profile icons; an "ENG" language dropdown (visual only); nav row Shop · Skills · Stories · About · Contact Us.
- Footer (per Figma), on black:
  - Newsletter: "Be the first to know", email input and Subscribe.
  - Contact Us (phone, email) and Currency (USD).
  - Link columns "mettā muse" and "Quick Links".
  - Follow Us (Instagram, LinkedIn), payment-method icons, copyright.
- Nav and footer links render as real `<a>` elements inside `<nav>` and `<footer>`.
- Mobile: hamburger menu toggle (keyboard operable); breadcrumb Home / Shop under the header; footer link columns collapse into accordions.
- Newsletter is UI only: client-side email validation and a confirmation message, no backend.

### F2 — Page heading and toolbar
- Page heading is the **only H1** on the page.
- The heading reads "Discover our products", with the intro paragraph below it.
- Toolbar shows the result count ("3425 ITEMS" style), a "‹ Hide filter" / "Show filter" toggle, and the sort control.
- Hiding filters switches the desktop grid from 3 to 4 columns.
- On mobile the toolbar is the designed split bar: FILTER | sort.
- The result count is announced to screen readers when it changes (`aria-live="polite"`).

### F3 — Product grid and card
- Each card shows a 3:4 image, the uppercase product name, the price, and a wishlist heart (filled when active; state kept in the browser only).
- **Price replaces the design's "Sign in or Create an account to see pricing" line**, in the same slot and typography. The brief requires price on the card and price sorting, and there is no auth in scope (see §8).
- Badges: "New product" (created within the last 30 days) at top-left; an "Out of stock" overlay when stock is 0.
- Responsive columns: 2 on mobile, 3 on tablet, 3 on desktop with filters shown, 4 with filters hidden.
- Images use `next/image` with explicit sizes. The first row is prioritised; the rest lazy-load.
- Title is an H3 inside the "Products" H2 section.
- If the PDP is built, the card links to it.

### F4 — Filters
- Panel order, all in the design's accordion style:
  1. **Customizable**: a single checkbox at the top. The design misspells it "Customizble".
  2. **Category** *(added, not in Figma)*: options from `GET /categories`. Required by the brief's API.
  3. **Price** *(added, not in Figma)*: min/max inputs. Required by the brief's API.
  4. **Ideal for, Occasion, Work, Fabric, Segment, Suitable for, Raw materials, Pattern**: the design's attribute facets.
- Each accordion shows "All" (or the selected values) when closed. When open, it shows "Unselect all" plus checkboxes with live counts, e.g. "Men (65)", from `GET /facets`. Options with a zero count stay visible but disabled, unless selected.
- Semantics: OR within one facet, AND across facets.
- Facet values are synthetic, since the source data has none. They are assigned deterministically at seed time, and the README says so.
- "Clear all" resets every filter.
- Changing any filter resets `page` to 1.
- Filters work without JS (native GET form) and are enhanced with JS (no full reload).

### F5 — Sort
- Options, in Figma's order and wording: Recommended (default), Newest first, Popular, Price: high to low, Price: low to high.
- A custom dropdown with a checkmark on the active option, as designed. Each option is a link, so sorting works without JS.

### F6 — Search
- Text query `q` matches product title, case-insensitive and partial.
- Opened from the header search icon (the design has no visible search field) as a search form that sets `q`.

### F7 — Pagination
- Numbered pages with previous/next, rendered as real `<a href>` links. They are crawlable and work without JS.
- Current page is marked with `aria-current="page"`.
- Not in the Figma (the grid simply ends), so it is styled in the design language below the grid. The brief requires pagination or infinite scroll.

### F8 — States
- **Loading**: a skeleton grid shows on the first load *and* on every filter, sort or page change.
- **Empty**: a friendly message plus a "Clear filters" action.
- **Error**: the API being unreachable or timing out shows an error state with retry. It never shows a blank page or a crash.
- **404**: unknown product id on the PDP shows the Next.js not-found page.

### F9 — Responsive
- Breakpoints: mobile < 640px, tablet 640–1023px, desktop ≥ 1024px.
- Tablet layout isn't in Figma, so it is our own design, kept consistent with the design language.
- On mobile, FILTER opens the filter panel in a drawer (native `<dialog>`). The drawer isn't designed, so it reuses the desktop accordion styles.
- No horizontal scroll at any width from 320px up.

### F10 — Product Detail Page (stretch)
- Route `/products/[id]-[slug]`, server-rendered from `GET /products/:id`.
- Gallery, title, price, description, `Product` JSON-LD, and a breadcrumb with `BreadcrumbList`.

## 5. URL state contract

The URL is the single source of truth for listing state.

| Param | Type | Default | Example |
|-------|------|---------|---------|
| `category` | comma-separated slugs | all | `category=mens-shirts,tops` |
| `minPrice` | number | none | `minPrice=10` |
| `maxPrice` | number | none | `maxPrice=100` |
| `customizable` | `true` | none | `customizable=true` |
| `ideal-for`, `occasion`, `work`, `fabric`, `segment`, `suitable-for`, `raw-materials`, `pattern` | comma-separated value slugs | all | `ideal-for=men,women&fabric=cotton` |
| `sort` | `recommended` \| `newest` \| `popular` \| `price-desc` \| `price-asc` | `recommended` | `sort=price-asc` |
| `q` | string (max 100 chars) | none | `q=shirt` |
| `page` | integer ≥ 1 | 1 | `page=2` |

Rules:
- Default values are omitted from the URL, so there is one canonical form per state.
- The frontend **normalises** bad params: invalid values are dropped and an out-of-range page is clamped. The API **rejects** bad params with 400.
- Page size is fixed at 12 on the frontend and is not exposed in the URL.

## 6. Data

**Source:** DummyJSON, filtered to the categories that fit the design's storefront (bags, accessories, fashion): roughly 60–80 products with multiple images, ratings, stock and brand.

FakeStore's 20 products make pagination and filtering look trivial. The brief says we *may* use FakeStore, so this is allowed; the reason goes in the README.

**Entities:** Category, Product, ProductImage, plus Attribute, AttributeValue and ProductAttributeValue for the design's facets. See `dbmodal.md` for the full model.

**Images:** downloaded once, converted to WebP, renamed to `<product-slug>-<n>.webp`, and served by our own API. The frontend never calls DummyJSON or FakeStore.

## 7. Non-functional requirements

- **SSR:** the product grid HTML is in the initial response. Verified on the *deployed* URL.
- **Performance:** minimal DOM with no wrapper divs for styling; CSS Modules; `next/font` self-hosted; no UI kits; images sized correctly with modern formats.
- **Dependencies:** frontend runtime deps are `next`, `react` and `react-dom` only. Every added package is justified in the README.
- **Accessibility:** semantic landmarks, a skip link, visible `:focus-visible` styles, labelled inputs, WCAG AA contrast, and full keyboard navigation including the filter drawer.
- **Reliability:** server fetches have an 8s timeout. A cron ping keeps the free-tier API warm.
- **Security:** no secrets committed (`.env.example` only), CORS allowlist, Helmet headers, validated inputs.

## 8. Assumptions and open decisions

| Item | Current decision | Revisit if |
|------|------------------|------------|
| Deadline | **Unknown, ask the recruiter** | — |
| Figma filter list | The design's 8 facets + Customizable, plus Category and Price added in the same style | — (resolved from the Figma) |
| Data source | DummyJSON | You'd rather stay strictly with FakeStore |
| Currency | USD (confirmed by the Figma footer) | — |
| Card pricing | Show the price in place of the design's "Sign in to see pricing" line | Appscrip asks you to follow the design literally |
| Pagination | Numbered pagination, styled to match | — (not in the Figma) |
| Facet data | Synthetic, assigned deterministically at seed time | — |
| Frontend host | Vercel (decided; allowed by the brief) | — |
| PDP | Stretch goal | Time is short, then drop it and use `/` URLs in JSON-LD |

## 9. Milestones (≈5 working days)

| Day | Milestone | Done when |
|-----|-----------|-----------|
| 0 | Repo setup | Monorepo scaffolded, `CLAUDE.md`, `.env.example`, first commit |
| 1 | Static HTML/CSS | `/static` matches Figma at desktop and mobile; committed |
| 2 | Backend + DB | Schema (incl. facet tables), migrations, seed, endpoints incl. `/facets`, validation, Swagger, deployed to Render + Neon |
| 3 | Frontend SSR | PLP server-rendered from the live API; filters, sort, search, pagination working via URL |
| 4 | Polish | All states, responsive pass, SEO (metadata, JSON-LD, canonical), a11y, Lighthouse |
| 5 | Ship | Vercel deploy verified, keep-warm cron, README complete, PDP if time allows |

## 10. Submission checklist

- [ ] Public repo `Appscrip-task-PrashukJain` with `/static`, `/frontend`, `/backend`
- [ ] Commit history is incremental with meaningful messages
- [ ] Live frontend URL: SSR verified via "view source"
- [ ] Live API URL plus `/docs` (Swagger)
- [ ] README: live URLs, stack and why, local setup, architecture and folders, endpoints, SSR and SEO decisions, dependencies and why, AI usage, limitations
- [ ] `CLAUDE.md` / `AGENTS.md` committed
- [ ] Lighthouse run on the deployed site; scores noted in the README
