# backend/CLAUDE.md — Express + Prisma API

Applies to everything under `backend/`. The root `CLAUDE.md` rules still apply. Architecture is defined in `docs/backend-plan.md`; the schema and indexes in `docs/dbmodal.md`. Read the relevant section before each task.

## Stack and allowed dependencies

Node.js 22 · Express 5 · TypeScript strict · Prisma · PostgreSQL (Neon in prod, Docker locally) · deployed on Render (Virginia).

| Allowed | Packages |
|---------|----------|
| Runtime | `express`, `@prisma/client`, `zod`, `@asteasolutions/zod-to-openapi`, `swagger-ui-express`, `helmet`, `cors` |
| Dev | `prisma`, `typescript`, `tsx`, `@types/node`, `@types/express`, `@types/cors`, `@types/swagger-ui-express`, `supertest`, `@types/supertest`, `sharp`, `eslint`, `typescript-eslint`, `prettier` |

Not allowed without asking: `dotenv` (use `--env-file`), `express-async-errors` (Express 5 handles async errors), `joi`/`class-validator`, `morgan`/`pino`, `lodash`, `axios`.

## Commands

```bash
docker compose up -d                 # local Postgres (repo root)
npm run dev                          # tsx watch with --env-file=.env
npm run lint
npm run typecheck                    # tsc --noEmit
npm test                             # node:test + supertest
npx prisma migrate dev --name <name> # create + apply a migration locally
npx prisma db seed                   # idempotent seed from the committed snapshot
```

Never run `prisma migrate reset`, `prisma db push`, or anything against the production `DATABASE_URL` without asking.

## Layering rules (graded: code structure 20%, backend 20%)

Request flow: `*.routes.ts` → `validate()` → `*.controller.ts` → `*.service.ts` → Prisma → `*.mapper.ts` → response.

| Layer | May | May not |
|-------|-----|---------|
| `*.routes.ts` | Define paths, attach `validate()` and the controller | Contain logic |
| `*.controller.ts` | Read `res.locals.validated`, call one service function, `res.json()` the result | Touch Prisma, build queries, contain business rules |
| `*.service.ts` | Business logic and Prisma queries; throw `HttpError` | Import Express, `req` or `res` |
| `*.schema.ts` | zod schemas for input (and OpenAPI registration) and `z.infer` types | Contain logic |
| `*.mapper.ts` | Convert Prisma rows to API shapes (cents → amount, file name → URL, `isNew`) | Query the DB |

- Services are plain exported functions, not classes.
- There is no repository layer (see `docs/backend-plan.md` §2). Do not add one.

## API rules

- **Validation:** every route with query or params uses `validate({ query, params })`. Parsed values live on `res.locals.validated`, because Express 5's `req.query` is read-only. Keep that why-comment.
- **Schemas are strict** (`.strict()`): unknown params → 400.
- **Errors:** throw `HttpError(status, message, details?)` or let a `ZodError` propagate. Never call `res.status()` for errors in controllers. `error-handler.ts` is the only place that formats error responses, using the shared shape in `docs/backend-plan.md` §4.
- **Response shapes** follow `docs/backend-plan.md` §4 exactly. Lists use `{ data, meta }`. Don't add, rename or drop fields without asking.
- **Every `orderBy` ends with `id`** as a tie-breaker.
- **Money** is integer cents in the DB and in services; it becomes a decimal amount only in the mapper.
- Use `select`, not `include`, in list queries; fetch only the fields the response needs.
- Use Prisma's query API. `$queryRaw` only if a doc says so; always the tagged-template form, never string concatenation.

## Database rules

- The schema must match `docs/dbmodal.md`: table names, columns, constraints and indexes.
- **Never edit a migration that has already been applied.** Create a new one.
- Things Prisma can't express (trigram index, `NULLS LAST` rating index, CHECK constraints) go in hand-written SQL migrations, created with `prisma migrate dev --create-only` and then edited. Each statement gets a comment saying what it serves.
- `snake_case` in the DB via `@map`/`@@map`; `camelCase` in TypeScript.
- The seed reads only the committed snapshot (`prisma/seed/data/products.json`), never the live DummyJSON API. It is transactional and idempotent: running it twice gives identical results.

## Testing

- e2e tests in `test/*.e2e.test.ts` using `node:test` + supertest against `createApp()`. Never call `listen()` in tests.
- Each test name states the behaviour: `'GET /products rejects maxPrice below minPrice with 400'`.
- Cover the cases listed in `docs/backend-plan.md` §9. Don't pad with trivial tests.

## File conventions

- `kebab-case.ts`, grouped by feature under `src/modules/<feature>/` as `<feature>.routes.ts`, `.controller.ts`, `.service.ts`, `.schema.ts`, `.mapper.ts`.
- Shared code lives only in `src/lib/`, `src/middleware/` and `src/config/`.
- Read env only through `src/config/env.ts`; never use `process.env` elsewhere.
