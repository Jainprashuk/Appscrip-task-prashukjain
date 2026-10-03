# Backend Plan — Node.js + Express + Prisma + PostgreSQL

## 1. Stack

| Choice | Why |
|--------|-----|
| Node.js 22 LTS + TypeScript (strict) | Typed end to end; types inferred from validation schemas |
| Express 5 | Accepted by the brief "with a clean structure"; every line is plain and explainable. v5 forwards rejected promises from async handlers to the error middleware, so no try/catch wrappers or `express-async-errors` |
| Prisma | Typed client, first-class migrations, simple seeding |
| PostgreSQL (Neon) | Preferred; serverless free tier with fast wake-up |
| zod | Request and env validation; `z.infer` gives the TS types, so validation and types have one source of truth |
| @asteasolutions/zod-to-openapi + swagger-ui-express | OpenAPI generated from the same zod schemas, so docs can't drift from validation; served at `/docs` |
| helmet | Security headers in one line |
| cors | CORS allowlist; handles preflight edge cases correctly, which a hand-rolled header wouldn't |
| sharp (dev only) | WebP conversion in the image script; never shipped at runtime |
| tsx (dev only) | Runs TS directly for dev, scripts and tests, with no build step |
| supertest (dev only) | HTTP-level e2e tests against the app without opening a port |

Deliberately not added:
- **dotenv:** Node's built-in `--env-file=.env` loads env files.
- **A logging library:** a ~10-line request-timing middleware is enough. Pino is listed as a future improvement.

**Why Express over NestJS** (for the README): every line is plain and explainable, and the structure is enforced by the conventions below rather than by the framework. The trade-off is that validation, docs and error-handling glue (~100 lines) is written by hand instead of coming from decorators.

Django mental map, for interview prep: router ≈ `urls.py`, controller ≈ view, service ≈ business logic layer, zod schema ≈ serializer validation, error middleware ≈ DRF custom exception handler, Prisma migrations ≈ Django migrations.

## 2. Folder structure

```
backend/
├── src/
│   ├── server.ts                    # listen + graceful shutdown (SIGTERM → close server, prisma.$disconnect)
│   ├── app.ts                       # createApp(): middleware, routes, docs, errors; no listen(), so tests import it
│   ├── config/env.ts                # zod-validated process.env, typed export; fails fast on boot
│   ├── lib/
│   │   ├── prisma.ts                # single PrismaClient instance
│   │   └── http-error.ts            # class HttpError(status, message, details?)
│   ├── middleware/
│   │   ├── validate.ts              # validate({ query?, params? }) → res.locals.validated
│   │   ├── cache-control.ts
│   │   ├── request-logger.ts
│   │   ├── not-found.ts             # unmatched route → 404 in the shared error shape
│   │   └── error-handler.ts         # ZodError / HttpError / Prisma / unknown → shared error shape
│   ├── modules/
│   │   ├── products/
│   │   │   ├── products.routes.ts       # Router: path + validate() + controller
│   │   │   ├── products.controller.ts   # req/res only: read validated input, call service, send JSON
│   │   │   ├── products.service.ts      # business logic + Prisma queries; knows nothing about Express
│   │   │   ├── products.schema.ts       # zod: list query, id params, response shapes
│   │   │   └── products.mapper.ts       # Prisma row → API shape (cents → amount, image URLs)
│   │   ├── categories/  (routes, controller, service, schema)
│   │   ├── facets/      (routes, controller, service, schema)
│   │   └── health/health.routes.ts
│   └── docs/openapi.ts              # registers zod schemas + paths → OpenAPI document
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed/
│       ├── seed.ts                  # reads snapshot → upserts DB
│       ├── fetch-snapshot.ts        # one-off: DummyJSON → data/products.json + images
│       └── data/products.json       # committed snapshot (reproducible, offline seed)
├── public/images/                   # committed WebP files: <slug>-<n>.webp
├── test/products.e2e.test.ts
└── .env.example                     # DATABASE_URL, DIRECT_URL, PORT, CORS_ORIGINS, PUBLIC_BASE_URL
```

**Layering rules:**
- Routes know only paths and middleware.
- Controllers know only `req`/`res`.
- Services never import Express.

There is no separate repository layer. Prisma already is the data-access layer, and wrapping it in a repository with a single implementation adds indirection without benefit. That reasoning goes in the README, since an evaluator may ask about it.

## 3. Database schema

```prisma
generator client { provider = "prisma-client-js" }
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")   // Neon pooled
  directUrl = env("DIRECT_URL")     // Neon direct, used by migrations
}

model Category {
  id        Int       @id @default(autoincrement())
  slug      String    @unique @db.VarChar(80)
  name      String    @db.VarChar(80)
  products  Product[]
  createdAt DateTime  @default(now()) @map("created_at")
  @@map("categories")
}

model Product {
  id          Int            @id @default(autoincrement())
  slug        String         @unique @db.VarChar(160)
  title       String         @db.VarChar(200)
  description String
  brand       String?        @db.VarChar(100)
  priceCents  Int            @map("price_cents")
  currency    String         @default("USD") @db.Char(3)
  rating      Decimal?       @db.Decimal(2, 1)
  ratingCount Int            @default(0) @map("rating_count")
  stock       Int            @default(0)
  isCustomizable Boolean     @default(false) @map("is_customizable")
  categoryId  Int            @map("category_id")
  category    Category       @relation(fields: [categoryId], references: [id], onDelete: Restrict)
  images      ProductImage[]
  attributeValues ProductAttributeValue[]
  createdAt   DateTime       @default(now()) @map("created_at")
  updatedAt   DateTime       @updatedAt @map("updated_at")

  @@index([categoryId, priceCents])   // category filter + price range / price sort
  @@index([priceCents])               // price range / sort without category
  @@index([createdAt(sort: Desc)])    // "newest"
  @@index([ratingCount(sort: Desc)])  // "popular"
  // "recommended" index (rating DESC NULLS LAST, rating_count DESC, id) is raw SQL, see dbmodal.md §8
  @@map("products")
}

model ProductImage {
  id        Int     @id @default(autoincrement())
  productId Int     @map("product_id")
  product   Product @relation(fields: [productId], references: [id], onDelete: Cascade)
  fileName  String  @map("file_name") @db.VarChar(200)  // e.g. mens-cotton-slim-fit-tshirt-1.webp
  alt       String  @db.VarChar(250)
  width     Int
  height    Int
  position  Int     @default(0)
  @@unique([productId, position])
  @@map("product_images")
}

model Attribute {                       // a facet, e.g. "Ideal for"
  id       Int              @id @default(autoincrement())
  slug     String           @unique @db.VarChar(40)   // ideal-for (also the URL param name)
  name     String           @db.VarChar(60)
  position Int              @default(0)               // order in the filter panel
  values   AttributeValue[]
  @@map("attributes")
}

model AttributeValue {                  // a facet option, e.g. "Men"
  id          Int                     @id @default(autoincrement())
  attributeId Int                     @map("attribute_id")
  attribute   Attribute               @relation(fields: [attributeId], references: [id], onDelete: Cascade)
  slug        String                  @db.VarChar(60)
  name        String                  @db.VarChar(60)
  position    Int                     @default(0)
  products    ProductAttributeValue[]
  @@unique([attributeId, slug])
  @@map("attribute_values")
}

model ProductAttributeValue {           // join: product ↔ facet option
  productId        Int            @map("product_id")
  attributeValueId Int            @map("attribute_value_id")
  product          Product        @relation(fields: [productId], references: [id], onDelete: Cascade)
  attributeValue   AttributeValue @relation(fields: [attributeValueId], references: [id], onDelete: Cascade)
  @@id([productId, attributeValueId])
  @@index([attributeValueId, productId])
  @@map("product_attribute_values")
}
```

**Search index.** Prisma's `contains` with `mode: 'insensitive'` compiles to `ILIKE '%q%'`, which a B-tree can't serve. Add a hand-written migration step:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX products_title_trgm_idx ON products USING GIN (title gin_trgm_ops);
```

The same hand-written migration also adds the "recommended" rating index and the CHECK constraints. Full SQL is in `dbmodal.md` §8.

**Design notes for the README:**
- Price is stored as integer cents to avoid float errors.
- snake_case in the DB, camelCase in code.
- Every index maps to a specific query.
- `onDelete: Restrict` on category stops a category with products from being deleted.
- The design's filters use a normalised attribute / value / join model; `dbmodal.md` §5a explains why over columns or `jsonb`.
- At under 100 rows, Postgres may choose sequential scans anyway. The indexes are designed for growth, and `EXPLAIN` output can be shown in the follow-up.

## 4. API contract

Base URL is the API root, with Swagger at `/docs`. All responses are JSON. Lists use the envelope `{ data, meta }`.

### `GET /products`

| Param | Validation | Notes |
|-------|-----------|-------|
| `page` | int, ≥ 1, default 1 | |
| `limit` | int, 1–48, default 12 | |
| `category` | comma-separated slugs, each `^[a-z0-9-]+$`, max 10 | Unknown slugs give an empty result (200), not 404, because filters are queries, not resources |
| `minPrice`, `maxPrice` | number ≥ 0, max 2 decimals; `maxPrice ≥ minPrice` | Major units; converted to cents in the service |
| `customizable` | `true` only | Products marked customizable |
| `ideal-for`, `occasion`, `work`, `fabric`, `segment`, `suitable-for`, `raw-materials`, `pattern` | comma-separated value slugs, same rules as `category` | OR within a facet, AND across facets; unknown value slugs give an empty result |
| `sort` | enum `recommended \| newest \| popular \| price-desc \| price-asc` | default `recommended`; `popular` = most ratings |
| `q` | string, trimmed, 1–100 chars | Case-insensitive title match |

Unknown params return 400 (the zod query schema is `.strict()`).

```json
{
  "data": [{
    "id": 12, "slug": "mens-cotton-slim-fit-tshirt", "title": "Men's Cotton Slim Fit T-Shirt",
    "price": { "amount": 19.99, "currency": "USD" },
    "rating": { "average": 4.3, "count": 120 },
    "isNew": false, "inStock": true, "isCustomizable": true,
    "category": { "slug": "mens-shirts", "name": "Men's Shirts" },
    "image": { "url": "https://api.example.com/images/mens-cotton-slim-fit-tshirt-1.webp",
               "alt": "Men's Cotton Slim Fit T-Shirt in navy, front view", "width": 800, "height": 800 }
  }],
  "meta": { "page": 1, "limit": 12, "total": 48, "totalPages": 4 }
}
```

### `GET /products/:id`
- `id` is validated with `z.coerce.number().int().positive()`: a non-integer returns 400; a missing product returns 404.
- Returns the summary fields plus `description`, `brand`, `stock`, and `images[]` ordered by position.

### `GET /categories`
- Returns `{ data: [{ id, slug, name, productCount }] }`, sorted by name. `productCount` comes from `_count`.

### `GET /facets`
- Accepts the same filter params as `GET /products`, but not `page`, `limit` or `sort`.
- Returns every facet with all its values and their counts under the current filters, plus the customizable count:
  `{ data: [{ slug: "ideal-for", name: "Ideal for", values: [{ slug: "men", name: "Men", count: 65 }] }], meta: { customizableCount: 12 } }`
- Counts are **disjunctive**: each facet's counts apply every filter *except that facet's own selection*. Picking "Men" therefore doesn't zero out "Women". This is standard e-commerce behaviour and matches the design's per-option counts.
- Zero-count values are still returned, so the panel layout stays stable; the frontend disables them.
- Implementation: one Prisma `groupBy` on `ProductAttributeValue` per facet, filtered by `attributeValue.attributeId` and by the product filters minus that facet. That's 8 small aggregates, run in parallel.

### `GET /health`
- Returns `{ status: "ok" }` after a `SELECT 1`. Used by the keep-warm cron and the Render health check.

### Error shape (all errors)

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "Invalid query parameters",
  "details": [{ "field": "maxPrice", "message": "maxPrice must be greater than or equal to minPrice" }],
  "path": "/products?minPrice=50&maxPrice=10",
  "timestamp": "2026-10-01T10:00:00.000Z"
}
```

`error-handler.ts` (registered last in `app.ts`) handles four cases:
- `ZodError` becomes 400, with each issue flattened into `details` (`field` is the issue path joined with `.`).
- `HttpError`s keep their status.
- Prisma `P2025` (record not found) becomes 404.
- Anything else becomes 500 with a generic message. The stack trace is logged, never returned.

Unmatched routes hit `not-found.ts` first and get a 404 in the same shape.

## 5. Service logic (`products.service.ts`)

```ts
const SORT_MAP: Record<SortKey, Prisma.ProductOrderByWithRelationInput[]> = {
  recommended: [{ rating: { sort: 'desc', nulls: 'last' } }, { ratingCount: 'desc' }, { id: 'asc' }],
  newest:      [{ createdAt: 'desc' }, { id: 'desc' }],
  'price-asc': [{ priceCents: 'asc' },  { id: 'asc' }],
  'price-desc':[{ priceCents: 'desc' }, { id: 'asc' }],
  popular:     [{ ratingCount: 'desc' }, { id: 'asc' }],
};

export async function listProducts(query: ListProductsQuery) { // type inferred from zod
  const where = buildWhere(query); // category slugs IN, price gte/lte, title contains insensitive,
                                    // isCustomizable, and one `attributeValues: { some: … }` per selected facet (ANDed)
  const [total, rows] = await prisma.$transaction([
    prisma.product.count({ where }),
    prisma.product.findMany({
      where, orderBy: SORT_MAP[query.sort],
      skip: (query.page - 1) * query.limit, take: query.limit,
      select: SUMMARY_SELECT, // only needed columns + first image (position 0)
    }),
  ]);
  return { data: rows.map(toSummary), meta: buildMeta(query, total) };
}
```

- The `id` tie-breaker on every sort keeps pagination stable, so there are no duplicate or missing items across pages.
- `select` instead of `include` avoids over-fetching. The first image is fetched via `images: { where: { position: 0 }, take: 1 }`.
- Offset pagination is fine at this scale. The README notes cursor pagination as the scaling path.

## 6. App wiring (`app.ts`)

Middleware order matters:
1. `app.disable('x-powered-by')` and `app.set('trust proxy', 1)`, since the app runs behind Render's proxy.
2. `helmet({ crossOriginResourcePolicy: { policy: 'cross-origin' } })`. Images must be loadable by the frontend's image optimizer.
3. `cors({ origin: env.CORS_ORIGINS, methods: ['GET'] })`. The API is read-only, so only GET is allowed.
4. Request logger.
5. `/images` → `express.static('public/images', { maxAge: '365d', immutable: true })`. The long cache makes cold starts irrelevant for images after the first fetch.
6. `/docs` → swagger-ui-express serving the generated document, plus the raw spec at `/openapi.json`.
7. Routes: `/health`, `/categories`, `/products`. The product and category routers get the `cache-control` middleware: `public, max-age=60, stale-while-revalidate=300`.
8. `not-found`, then `error-handler`. The error handler must be last and must have the 4-argument signature, or Express won't treat it as one.

`server.ts` calls `createApp().listen(env.PORT)`. On SIGTERM it closes the server and then calls `prisma.$disconnect()`.

**Validation middleware (`middleware/validate.ts`):**

```ts
export const validate = (schemas: { query?: ZodTypeAny; params?: ZodTypeAny }) =>
  (req: Request, res: Response, next: NextFunction) => {
    res.locals.validated = {
      query: schemas.query?.parse(req.query),
      params: schemas.params?.parse(req.params),
    };
    next(); // a ZodError thrown above goes straight to error-handler
  };
```

Two Express 5 gotchas to handle (and good follow-up talking points):
- `req.query` is a read-only getter in v5, so parsed values can't be written back to it. They live on `res.locals.validated`, read through a small typed helper.
- The default query parser is "simple": no nested objects, and repeated keys become arrays. The schema accepts both `category=a,b` and `category=a&category=b`.

**List query schema (`products.schema.ts`):**

```ts
const slug = z.string().regex(/^[a-z0-9-]+$/);

export const listProductsQuery = z.object({
  page:     z.coerce.number().int().min(1).default(1),
  limit:    z.coerce.number().int().min(1).max(48).default(12),
  category: z.union([z.string(), z.array(z.string())]).optional()
              .transform(v => v === undefined ? undefined : [v].flat().flatMap(s => s.split(',')))
              .pipe(z.array(slug).max(10).optional()),
  minPrice: z.coerce.number().min(0).optional(),
  maxPrice: z.coerce.number().min(0).optional(),
  customizable: z.literal('true').transform(() => true).optional(),
  ...facetParams, // one optional slug-list field per facet in FACETS = ['ideal-for', 'occasion', …], built like `category`
  sort:     z.enum(['recommended', 'newest', 'popular', 'price-desc', 'price-asc']).default('recommended'),
  q:        z.string().trim().min(1).max(100).optional(),
}).strict()
  .refine(d => d.minPrice === undefined || d.maxPrice === undefined || d.maxPrice >= d.minPrice, {
    path: ['maxPrice'], message: 'maxPrice must be greater than or equal to minPrice',
  });

export type ListProductsQuery = z.infer<typeof listProductsQuery>;
```

Prices are converted to cents in the service with `Math.round(x * 100)`. Checking two decimal places with float math in the schema is unreliable.

When wiring zod-to-openapi, pin versions that support your zod major version. Transformed fields like `category` may need an explicit `.openapi({ type: 'string', example: 'tops,mens-shirts' })` so the docs show the input format.

## 7. Seed and images

**Step 1 — `npm run seed:snapshot`** (one-off, output committed):
1. Fetch `https://dummyjson.com/products?limit=0` and its categories, keeping only the categories that fit the design's storefront (bags, accessories, fashion).
2. For each product, slugify the title (deduplicating with `-2`, `-3`), and download up to 3 images.
3. Use `sharp` to resize to 800px wide and convert to WebP at quality 78, saving `public/images/<slug>-<n>.webp`.
4. Generate alt text: `"<title>, <category name>"` plus a view qualifier for extra images (e.g. "side view"). No "image of".
5. Assign facet values and `isCustomizable` deterministically. A rule table per category limits which values make sense (e.g. bags → Raw materials: Leather or Cotton), and a hash of the product slug picks among them. Values come only from the Figma's option lists.
6. Write `prisma/seed/data/products.json` with clean fields, image metadata (fileName, width, height, alt) and facet values.

**Step 2 — `npx prisma db seed`** (repeatable):
- Reads the snapshot and upserts categories by `slug`, products by `slug`, images by `(productId, position)`, and attributes and values by slug. Product-to-value links are replaced per product.
- Runs inside a transaction, so it is idempotent and safe to re-run.
- Converts price to cents and staggers `createdAt` so "newest" sorting is meaningful.

Why the split: the deployed DB and any local setup never depend on a third-party API being up. Reviewers can run the seed offline. Image filenames are deterministic and SEO-friendly. The README states that images are roughly 5 MB in the repo, as a deliberate trade-off versus adding an object-storage service.

## 8. Configuration and security

- `config/env.ts` parses `process.env` with zod: `DATABASE_URL`, `DIRECT_URL`, `PORT` (default 3001), `CORS_ORIGINS` (comma-separated, transformed to an array), `PUBLIC_BASE_URL` (used to build absolute image URLs) and `NODE_ENV`. On invalid config, the process exits on boot with a readable message.
- Locally, env is loaded with `tsx --env-file=.env` or `node --env-file=.env`; no dotenv.
- `.env` is gitignored and `.env.example` is committed. Real values go only in the Render and Neon dashboards.
- In production, 500 responses never include stack traces.
- Read-only API: no write endpoints, so no auth is needed. The README notes rate limiting (`express-rate-limit`) as a known limitation and future addition.

## 9. Testing (lightweight, high signal)

`test/products.e2e.test.ts` uses Node's built-in test runner (`tsx --test`) and supertest against `createApp()` and a test DB. It covers:
- Default list returns 12 items with a correct `meta`.
- Category + price filters return only matching items.
- Each sort key orders correctly, and pages don't overlap.
- Facet filters are OR within a facet and AND across facets; `/facets` counts are disjunctive.
- `maxPrice < minPrice`, `limit=1000`, `sort=foo` and an unknown param each return 400 with `details`.
- `/products/abc` returns 400, and `/products/999999` returns 404, both in the shared error shape.

This is enough to show validation and error discipline without over-investing.

## 10. Deployment

- **Neon:** create the project and copy the pooled (`DATABASE_URL`) and direct (`DIRECT_URL`) connection strings.
- **Render web service:**
  - Root `backend/`.
  - Build: `npm ci && npx prisma generate && npm run build`.
  - Pre-deploy: `npx prisma migrate deploy`.
  - Start: `node dist/server.js`.
  - Health check path: `/health`.
- Run the seed once against Neon from your machine.
- **Keep-warm:** cron-job.org pings `/health` every 10 minutes. The frontend's 8s timeout and error state cover any remaining cold start.
- **Verify:**
  - `curl $API/products?category=x&sort=price-asc` returns data.
  - `/docs` loads.
  - An image URL returns `image/webp` with the immutable cache header.

## 11. Build order

1. TS project init, `app.ts`/`server.ts` skeleton, env validation, error-handler + not-found + health. Commit.
2. Schema (incl. facet tables) + first migration + hand-written SQL migration (trigram, rating index, CHECKs). Commit.
3. Snapshot script, images, seed. Commit.
4. Categories module. Then the products list with its zod schema, filters, sort and pagination. Then detail. Then facets. Commit per endpoint.
5. OpenAPI generation + `/docs`, cache-control middleware. Commit.
6. e2e tests. Commit.
7. Deploy to Render + Neon, seed production, set up the keep-warm cron.
8. Write the README backend sections: endpoints, schema and index rationale, dependencies and why, why Express.
