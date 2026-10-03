# progress.md — Task status

Updated by Claude Code at the end of every task (with Prashuk's approval), and read at the start of every session. Task details live in `development-plan.md` §4.

**Status values:** `todo` · `in progress` · `blocked` · `done` · `cut`


## Phase 0 — Setup

| ID | Task | Status | Date | Notes |
|----|------|--------|------|-------|
| S-01 | Create public repo Appscrip-task-PrashukJain | done | 2026-10-03 | Root .gitignore, .editorconfig, .nvmrc, .prettierrc, README skeleton |
| S-02 | Add docs/ with all five plan docs | done | 2026-10-03 | Plan docs, ai-log, INSTRUCTIONS and progress copied into docs/; docs/projects/ to be removed before submission |
| S-03 | Add the Claude Code setup | done | 2026-10-03 | CLAUDE.md files and .claude/ at root; brief renamed to docs/assignment.pdf |
| S-04 | docker-compose.yml with Postgres 16 | blocked | 2026-10-03 | File written; done-when not run: Docker not installed on this machine |
| S-05 | Accounts: Neon (us-east-1), Render (Virginia), Vercel, cron-job.org | todo | | |

## Phase 1 — Static HTML/CSS

| ID | Task | Status | Date | Notes |
|----|------|--------|------|-------|
| ST-01 | Extract tokens from Figma | todo | | |
| ST-02 | Header (announcement bar, logo row, icons, nav; mobile variant) | todo | | |
| ST-03 | Heading block, toolbar, filter sidebar (`<details>` accordions), sort menu | todo | | |
| ST-04 | Product grid + card (badge, out-of-stock overlay, heart, price slot) | todo | | |
| ST-05 | Footer (desktop columns, mobile accordions) | todo | | |
| ST-06 | Responsive pass at 320 / 375 / 768 / 1024 / 1440 | todo | | |

## Phase 2 — Backend skeleton

| ID | Task | Status | Date | Notes |
|----|------|--------|------|-------|
| B-01 | Backend TS project: Express 5, tsx, tsc, ESLint, Prettier; app/server split | todo | | |
| B-02 | Env validation, HttpError, not-found + error-handler, helmet, cors, logger | todo | | |
| B-03 | Prisma init + lib/prisma.ts; GET /health runs SELECT 1 | todo | | |
| B-04 | Deploy to Render with Neon | todo | | |

## Phase 3 — Database and seed

| ID | Task | Status | Date | Notes |
|----|------|--------|------|-------|
| D-01 | schema.prisma: all six tables incl. facets | todo | | |
| D-02 | Hand-written migration 0002: trigram, rating index, CHECKs | todo | | |
| SD-01 | fetch-snapshot.ts: DummyJSON → storefront categories → products.json | todo | | |
| SD-02 | Image pipeline: download, sharp → 800px WebP, `<slug>-<n>.webp` | todo | | |
| SD-03 | Facet rule table + slug hash → facet values and isCustomizable in the snapshot | todo | | |
| SD-04 | seed.ts: transactional upserts on natural keys | todo | | |

## Phase 4 — API

| ID | Task | Status | Date | Notes |
|----|------|--------|------|-------|
| A-01 | validate() middleware → res.locals.validated; typed helper | todo | | |
| A-02 | GET /categories with productCount | todo | | |
| A-03 | GET /products: filters, facets, sort, pagination, mapper | todo | | |
| A-04 | GET /products/:id with all images | todo | | |
| A-05 | express.static for /images with an immutable cache | todo | | |
| A-06 | GET /facets with disjunctive counts (one groupBy per facet, in parallel) | todo | | |
| A-07 | OpenAPI via zod-to-openapi + swagger-ui at /docs, /openapi.json | todo | | |
| A-08 | e2e tests (node:test + supertest) | todo | | |
| A-09 | Production: migrate deploy, seed Neon, env vars | todo | | |

## Phase 5 — Frontend

| ID | Task | Status | Date | Notes |
|----|------|--------|------|-------|
| F-01 | create-next-app (TS, App Router, no Tailwind) | todo | | |
| F-02 | Layout: AnnouncementBar, Header, MobileNav, Footer, NewsletterForm; skip link | todo | | |
| F-03 | types/api.ts, lib/api.ts (server-only, 8s timeout, ApiError) | todo | | |
| F-04 | page.tsx renders the unfiltered grid from the live API | todo | | |
| F-05 | Deploy to Vercel (Root Directory frontend, function region iad1) | todo | | |
| F-06 | lib/search-params.ts: parse, normalise, serialise, withChange | todo | | |
| F-07 | Pagination (`<Link>`s, aria-current, out-of-range redirect) | todo | | |
| F-08 | SortMenu (`<details>` + links, checkmark, Esc/outside-click) | todo | | |
| F-09 | FilterForm: Customizable, Category, Price, 8 facets with counts | todo | | |
| F-10 | Header search → q | todo | | |
| F-11 | FilterToggle with a cookie; 3- vs 4-column grid rendered by the server | todo | | |
| F-12 | States: loading, Suspense skeleton, error + retry, empty | todo | | |
| F-13 | WishlistButton (browser storage, aria-pressed) | todo | | |
| F-14 | Mobile: split bar, `<dialog>` drawer, footer accordions; tablet | todo | | |

## Phase 6 — Quality and ship

| ID | Task | Status | Date | Notes |
|----|------|--------|------|-------|
| Q-01 | Metadata, canonical policy, noindex for q, OG/Twitter tags | todo | | |
| Q-02 | JSON-LD: ItemList → Product + Offer, and BreadcrumbList | todo | | |
| Q-03 | robots.ts, sitemap.ts (incl. category URLs) | todo | | |
| Q-04 | Heading audit (one H1, H2 sections, H3 cards); semantic landmarks | todo | | |
| Q-05 | A11y pass: keyboard, focus, labels, contrast | todo | | |
| Q-06 | Lighthouse (mobile + desktop) on the deployed URL; fix the top issues | todo | | |
| Q-07 | CI: GitHub Actions running lint + typecheck + backend tests on push | todo | | |
| Q-08 | README: all 9 sections (see §7) | todo | | |
| Q-09 | Final verification (§6), tag v1.0.0, send the 3 links | todo | | |
| Q-10 | *Stretch:* PDP `/products/[idSlug]` | todo | | |

---

## Session notes

Newest first. Five lines or fewer per session: done · in progress (exact stopping point) · open questions · next task.

<!-- Example:
### 2026-10-03
- Done: ST-01, ST-02
- In progress: ST-03, sidebar accordions built; sort menu not started
- Open: confirm the "Ideal for" counts are placeholder text in the static page
- Next: finish ST-03
-->
