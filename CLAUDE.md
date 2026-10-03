# CLAUDE.md — Appscrip PLP take-home

You are helping Prashuk build a take-home assignment that will be **read line by line by evaluators**, and Prashuk must be able to **explain every line in a follow-up interview**. Optimise for code that is correct, plain and easy to read, not clever or short.

Also read `frontend/CLAUDE.md` or `backend/CLAUDE.md` when working in those folders.

---

## 1. Source of truth

Read the relevant doc **before** writing code. Do not rely on memory of a previous session.

| Doc | Use it for |
|-----|-----------|
| `docs/INSTRUCTIONS.md` | How to run the project: session protocol, phase order, who does what. Read at the start of every session |
| `docs/progress.md` | Status of every task + session notes. Read at session start; update at the end of each task |
| `docs/assignment.pdf` | The original brief. Requirements here always win |
| `docs/product.md` | What we're building: features, URL contract, design decisions |
| `docs/development-plan.md` | Task list (IDs like `A-03`), done-when checks, conventions, comment standard (§2) |
| `docs/frontend-plan.md` | Next.js architecture, components, SSR, SEO, responsive rules |
| `docs/backend-plan.md` | Express structure, API contract, validation, seed |
| `docs/dbmodal.md` | Database schema, indexes, migrations, facet model |

**Precedence:** assignment brief > `product.md` > the other docs > your own judgement.

If two docs disagree, a doc is silent on something that matters, or the plan looks wrong, **stop and ask**. Never improvise a design decision and continue.

---

## 2. Hard rules — never break these

1. **No git writes.** Never run `git add`, `git commit`, `git push`, `git reset`, `git rebase`, `git restore`, `git checkout -- <file>`, `git stash` or `git clean`. Prashuk reviews and commits everything himself. Read-only git (`git status`, `git diff`, `git log`) is fine.
2. **No new dependencies without asking.** Only packages listed in the app's `CLAUDE.md` are allowed. If you think another is needed, explain why and wait.
3. **No secrets.** Never read, print or edit `.env` files. Use `.env.example` for variable names.
4. **Stay in scope.** Do only the task you were given. No drive-by refactors, renames or "improvements" elsewhere. If you notice a problem outside the task, mention it at the end; don't fix it.
5. **Don't edit `docs/`** unless explicitly asked.
6. **Follow the planned structure.** Create files only where the plan says. If a new file or folder is needed, propose it first.
7. **Never claim something works without running it.** If you couldn't run a check, say so plainly.
8. **No placeholder code** (`// TODO: implement`, fake data, stubbed handlers) unless the task says so.

---

## 3. Workflow for every task

Prashuk will usually say `/task A-03` or describe a task. Then:

1. **Read** the task row in `docs/development-plan.md` §4 and every doc section it references.
2. **Plan, then wait.** Reply with:
   - the files you'll create or change,
   - the approach in 3–6 bullets,
   - anything ambiguous.

   Wait for "go" before writing code. Skip this only if Prashuk says "go ahead directly".
3. **Implement** the smallest change that satisfies the task's "done when". Match existing patterns in nearby files.
4. **Verify:** run lint, typecheck, tests (where they exist) and the task's "done when" check. Fix failures before reporting.
5. **Report**, in this order:
   - **What changed:** files, one line each.
   - **How I verified:** commands run and their results.
   - **Deviations:** anything that differs from the docs, and why. Write "None" if none.
   - **Explain-it notes:** 2–4 bullets Prashuk can use to explain the non-obvious parts in an interview.
   - **Suggested commit message:** Conventional Commits, e.g. `feat(api): add product listing with filters`.

   Then stop. Do not start the next task.

When Prashuk rejects or corrects your output, draft a short entry for `docs/ai-log.md` (what you produced, what was wrong, what changed). Show it to him; only write it to the file if he says so.

---

## 4. Readability rules (apply to all code)

**The test:** could Prashuk explain every line without looking anything up? If not, simplify it, or add a "why" comment.

**Naming**
- Use full, descriptive words: `productCount`, not `pc` or `cnt`. Allowed abbreviations: `id`, `url`, `api`, `db`, `dto`.
- Booleans read as yes/no questions: `isNew`, `hasStock`, `shouldShowFilters`.
- Functions start with a verb: `listProducts`, `buildWhere`, `parseSearchParams`.
- Constants that never change: `UPPER_SNAKE_CASE` (`MAX_PAGE_SIZE`).
- Name things the way the docs name them, so code and docs can be cross-referenced.

**Structure**
- One responsibility per function. Aim for under ~40 lines; split anything longer into named helpers.
- Return early instead of nesting. Keep nesting to 2–3 levels at most.
- One React component per file. One feature per folder (as in the plans).
- No magic numbers: name them, and say where they come from (`// Figma: card width 300px`).

**Avoid clever code**
- No nested ternaries.
- No `reduce` where `map`, `filter` or a plain loop is clearer.
- No dense one-liners, no chained optional-call tricks, no metaprogramming.
- Prefer explicit over implicit: name intermediate values instead of inlining long expressions.

**TypeScript**
- Strict mode. No `any`. No `as` casts or `!` non-null assertions, except where a comment explains why it's safe.
- Explicit parameter and return types on every exported function.
- Use `type` for data shapes. Derive types from zod schemas (`z.infer`) instead of duplicating them.

**Errors**
- Never swallow an error (`catch {}`). Handle it at the layer the plans assign, or let it propagate.
- Error messages say what went wrong and with which input.

**Formatting**
- Prettier: 2 spaces, single quotes, semicolons, trailing commas, print width 100.
- Import order: Node built-ins → external packages → internal aliases (`@/…`) → relative imports → styles. Blank line between groups.

---

## 5. Comments (from `docs/development-plan.md` §2)

Comments explain **why**; code explains **what**.

- **File header:** 1–2 lines at the top of every source file stating its responsibility and boundaries.
- **TSDoc** (`/** … */`) on every exported function, component, type and zod schema: purpose, key params, return value, errors thrown.
- **"Why" comments** at the exact line of every non-obvious decision (the plans list them, e.g. the `id` sort tie-breaker, `<Suspense key>`, prices in cents).
- **Prisma:** `///` doc comments on models and non-obvious fields. **SQL:** a comment above each statement saying what it serves.
- **CSS:** section headers (`/* ── Product card ── */`); Figma reference for any specific value.

Never:
- Narrate obvious code.
- Leave commented-out code.
- Write a bare `TODO`. Use `TODO(prashuk): <what and why>` only for deliberate gaps.

Update comments in the same change as the code they describe.

---

## 6. Definition of done (every task)

- [ ] The task's "done when" check in `docs/development-plan.md` passes, and you ran it.
- [ ] Lint and typecheck pass in the app you touched; tests pass if they exist.
- [ ] Code follows §4 and §5 above.
- [ ] Nothing outside the task changed.
- [ ] No new dependencies, unless approved.
- [ ] Report delivered (§3 step 5). No git writes.
