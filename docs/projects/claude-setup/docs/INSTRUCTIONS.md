# INSTRUCTIONS.md — How to execute this project

**Audience:** Claude Code, working with Prashuk on the Appscrip PLP take-home.
**Read this file at the start of every session**, after `CLAUDE.md` (which loads automatically).

`CLAUDE.md` tells you *how to behave and how to write code*. This file tells you *how to run the project*: which files to use when, the order of work, what you do versus what Prashuk does, and how to keep continuity across sessions.

---

## 0. What success looks like

The evaluators will:
1. Open the live Vercel site and check it matches the Figma, works fully, and is server-rendered.
2. Read the repository: structure, naming, readability, comments, commit history.
3. Read the README.
4. Interview Prashuk about any line of code.

So every piece of work must be:
- **correct** against the docs,
- **readable** by a stranger,
- **explainable** by Prashuk.

Speed matters less than all three. When in doubt, choose the simpler implementation and say so.

---

## 1. File map

| File | What it is | When you read it |
|------|------------|------------------|
| `CLAUDE.md` (root) | Behaviour, hard rules, readability and comment standards, definition of done | Auto-loaded every session |
| `frontend/CLAUDE.md` | Frontend rules, allowed deps, rendering/SEO/a11y rules | Auto-loaded when working in `frontend/` |
| `backend/CLAUDE.md` | Backend rules, layering, allowed deps, DB rules | Auto-loaded when working in `backend/` |
| `docs/INSTRUCTIONS.md` | This file: how to run the project | Start of every session |
| `docs/progress.md` | Status of every task, plus session notes | Start and end of every session |
| `docs/assignment.pdf` | The original brief: **highest authority** | When a requirement is unclear, and before Q-08/Q-09 |
| `docs/product.md` | What we're building, URL contract, design decisions (§8) | Any task touching features or UX |
| `docs/development-plan.md` | Task list with IDs and done-when checks (§4), conventions (§2), quality gates (§6), cut list (§8) | Every task |
| `docs/frontend-plan.md` | Frontend architecture | Every frontend task, only the relevant sections |
| `docs/backend-plan.md` | Backend architecture, API contract | Every backend task, only the relevant sections |
| `docs/dbmodal.md` | Schema, indexes, migrations, facets, seed flow | Every DB or seed task |
| `docs/ai-log.md` | Prashuk's log of corrected AI output | Only to draft entries (shown to Prashuk first) and for Q-08 |
| `.claude/commands/task.md` | The `/task <ID>` workflow | Invoked by Prashuk |
| `.claude/commands/review.md` | The `/review` pre-commit audit | Invoked by Prashuk |

**Context discipline:** don't load every doc every time. For a task, read its row in `development-plan.md` §4 and only the doc sections that task needs. Long contexts make you less accurate.

---

## 2. Session protocol

### Start of session
1. Read `docs/progress.md`.
2. Tell Prashuk in 3 lines or fewer:
   - the last completed task,
   - anything marked `blocked` or with open notes,
   - the next task by ID.
3. Wait for him to confirm the next task, or to give a different one.

### During a task
Follow `/task` (`.claude/commands/task.md`): read → plan → **wait for "go"** → implement → verify → report → stop.

### End of a task
1. After the report, update the task's row in `docs/progress.md`: status, date, one-line note. This edit will ask Prashuk for permission; that is expected.
2. Remind Prashuk to run `/review` and commit before the next task.
3. If the conversation is long, suggest `/clear` before the next task. `progress.md` carries the continuity, not chat history.

### End of session
Add a dated entry to the "Session notes" section of `progress.md`:
- what was done,
- what's in progress and exactly where it stopped,
- open questions,
- the next task.

Keep it to 5 lines or fewer.

---

## 3. Who does what

You cannot access Prashuk's accounts, dashboards, secrets or git history. For these, give him **exact, numbered steps** and wait for confirmation.

| You (Claude Code) | Prashuk |
|-------------------|---------|
| Write and edit code, configs, migrations, scripts, tests | Review every diff, stage, commit, push |
| Run lint, typecheck, build, tests, local dev servers, `curl` checks | Create accounts: Neon, Render, Vercel, cron-job.org |
| Run local migrations and seed against Docker Postgres | Put secrets in `.env` files and hosting dashboards |
| Draft README sections and `ai-log.md` entries for approval | Run the production migration and seed against Neon (you give the exact command) |
| Read the Figma through the Figma MCP (if connected) | Run Lighthouse and Rich Results Test in the browser |
| Propose solutions when something is blocked | Make every decision not already in the docs |

---

## 4. Execution order and phase notes

Work strictly in this order; tasks are defined in `docs/development-plan.md` §4. Below are **execution notes**: practical details that the plan doesn't spell out. Where these notes and the plan seem to disagree, ask.

### Phase 0 — Setup (S-01 → S-05)
- **S-01:** create the root files:
  - `.gitignore`: `node_modules`, `.env`, `.env.local`, `.next`, `dist`, `coverage`, `.DS_Store`.
  - `.editorconfig`: 2 spaces, LF, final newline.
  - `.nvmrc`: `22`.
  - `.prettierrc`: `{ "singleQuote": true, "semi": true, "trailingComma": "all", "printWidth": 100 }`.
  - `README.md`: a stub with the 9 section headings from `development-plan.md` §7.
- **S-04:** `docker-compose.yml` with one `postgres:16` service, a named volume, port 5432, and credentials matching `backend/.env.example`.
- **S-05** is entirely Prashuk's (accounts). List the steps for him and wait.

### Phase 1 — Static HTML/CSS (ST-01 → ST-06)
- **Figma:** use the frame IDs in Appendix B. Use as few Figma calls as possible: the account is on a free plan with a low limit. Pull tokens once (ST-01), save them to `static/tokens.css`, and work from that file afterwards.
- If the Figma MCP isn't connected or the limit is hit, stop and tell Prashuk. Don't guess colours or fonts.
- **Images for the static page:** download 6–8 product images once from `https://cdn.dummyjson.com` into `static/images/`, renamed with descriptive slugs. Ask before downloading.
- The static page has **no JavaScript**. Accordions are `<details>`; the sort menu is a `<details>` with links. Show the "filters shown" (3-column) layout; demonstrate the 4-column layout with a CSS class switch, documented in a comment.
- Use **real semantic markup from the start** (headings, landmarks, `<article>` cards). This markup becomes the Next.js components later, so get it right here.

### Phase 2 — Backend skeleton (B-01 → B-04)
- Scripts in `backend/package.json`:
  - `dev`: `tsx watch --env-file=.env src/server.ts`
  - `build`: `tsc`
  - `start`: `node dist/server.js`
  - `lint`, `typecheck` (`tsc --noEmit`)
  - `test`: `tsx --test test/**/*.test.ts`
  - Prisma seed config pointing at `prisma/seed/seed.ts` via `tsx`.
- **B-04** is mostly Prashuk's (Render dashboard). Give him the exact values:
  - Root Directory `backend`, region Virginia.
  - Build command, pre-deploy command, start command and health check path, as written in `backend-plan.md` §10.
  - Env var names from `.env.example`.

### Phase 3 — Database and seed (D-01 → D-02, SD-01 → SD-04)
- The schema must match `docs/dbmodal.md` **exactly**: names, types, constraints, indexes.
- Migration `0002` is created with `--create-only` and then filled with the SQL in `dbmodal.md` §8.
- **SD-01/SD-02** fetch from DummyJSON **once**; the outputs (`products.json`, `public/images/`) are committed by Prashuk. `seed.ts` must never touch the network.
- **SD-03:** before writing the facet rule table, show it to Prashuk as a table (category → allowed values per facet) and wait for approval. The data is synthetic, so he must be comfortable defending it.
- After seeding, run `SELECT count(*)` on each table and report the counts.

### Phase 4 — Core API (A-01 → A-09)
- Implement exactly the contract in `backend-plan.md` §4: fields, names, envelope, error shape.
- Verify each endpoint with `curl` and paste the important lines of the output in your report: one success case and one error case per endpoint.
- **A-06 (disjunctive facet counts):** if it's taking a long time, it's acceptable to ship plain counts first. Ask before doing that, since it's on the cut list (`development-plan.md` §8).
- **A-09** production steps are Prashuk's. Give him the exact commands, with `DATABASE_URL` coming from his own shell, never typed into chat.

### Phase 5 — Frontend (F-01 → F-14)
- **F-01:** `create-next-app` with TypeScript, App Router, ESLint, `src/` directory, `@/*` alias, **no Tailwind**. Delete the boilerplate content and assets.
- Port `static/tokens.css` and the static markup. Don't redesign.
- **F-05** (Vercel) is Prashuk's dashboard work: Root Directory `frontend`, function region `iad1`, env vars `API_URL` and `SITE_URL`. After he deploys, verify SSR yourself:
  ```bash
  curl -s https://<site> | grep -c "<article"
  ```
  The count must be greater than 0.
- Build the frontend tasks strictly in plan order. Don't build components ahead of their task.
- After every frontend task, run `npm run build`, not just dev. Server/client boundary errors often only show up in the build.

### Phase 6 — Quality and ship (Q-01 → Q-10)
- **Q-06:** Prashuk runs Lighthouse in Chrome. You fix the issues he pastes back.
- **Q-08 (README):** draft section by section in the order of `development-plan.md` §7, using facts from the docs, not new claims.
  - The **AI usage** section must use real entries from `docs/ai-log.md`. If the log is empty, say so; never invent an example.
  - The **known limitations** section must list every deliberate deviation from `product.md` §8 and every remaining `TODO(prashuk)`.
- **Q-09:** walk through the pre-submission table in `development-plan.md` §6 row by row. Report each row as pass, fail, or "needs Prashuk".
- **Q-10 (PDP)** is a stretch goal. Start it only if Prashuk explicitly says to.

---

## 5. How to ask questions

When something is unclear or the docs conflict:
- Ask **before** writing code, in the plan step.
- Number the questions. For each, give options and your recommendation:
  > **Q1.** `dbmodal.md` says X, `backend-plan.md` says Y. Options: (a) follow X, (b) follow Y. **Recommend (a)** because… Which?
- At most 3 questions at a time. If there are more, the task is probably too big; propose splitting it.
- Never treat silence as approval. Wait for an answer.

---

## 6. Self-check before every report

Run through this before reporting a task done:

1. Does every file I touched have a header comment, and does every export have TSDoc?
2. Would a junior developer understand each function on first read? If one needs a second read, simplify it or add a why-comment.
3. Did I use names that match the docs (`listProducts`, `ProductCard`, `price_cents`…)?
4. Did I change anything outside the task's scope? If so, revert it and mention it instead.
5. Did I add a dependency? It must be on the allowed list or approved.
6. Did I actually run the done-when check, and am I reporting its real output?
7. Is there anything I'm unsure about? Put it under **Deviations** or as a question, never hidden.

---

## 7. When things go wrong

| Situation | What to do |
|-----------|------------|
| A check fails twice after fixes | Stop. Report the error, what you tried, and your best hypothesis. Don't keep guessing |
| The docs seem wrong or impossible | Stop. Quote the doc section, explain the problem, propose a fix to the doc. Don't silently diverge |
| A library behaves differently than the docs assume (versions, APIs) | Stop. Report the actual behaviour (with the version) and options |
| A task is much bigger than expected | Propose splitting it into smaller sub-steps, each with its own done-when |
| You made a mistake in an earlier task | Say so plainly, explain the impact, propose the fix as its own small change |
| Prashuk corrects you | Fix it, then draft the `ai-log.md` entry for him to approve |
| The conversation is getting long or you're losing track | Update `progress.md` and suggest `/clear` |

---

## Appendix A — Prompts for Prashuk

*(For Prashuk to copy-paste. Claude: you can ignore this appendix.)*

**First session ever:**
> Read `docs/INSTRUCTIONS.md` fully, then `docs/progress.md`, then skim the headings of every file in `docs/`. Don't write any code. Reply with: (1) your understanding of the project in 5 bullets, (2) the order of phases, (3) anything in the docs that looks inconsistent or unclear. Then wait.

**Start of every later session:**
> Start-of-session protocol from `docs/INSTRUCTIONS.md` §2.

**Do a task:**
> `/task ST-01`

**Approve a plan:**
> go

**Before every commit:**
> `/review`

**End of session:**
> End-of-session protocol: update `docs/progress.md` session notes.

**When the code is too clever or hard to read:**
> This is harder to read than it needs to be. Re-read `CLAUDE.md` §4 and rewrite it for a junior developer: smaller functions, plainer names, no clever constructs. Keep the behaviour identical.

**When it drifted from the docs:**
> This doesn't match `docs/<file>.md` §<n>. Re-read that section, list every difference, then fix them. Draft an `ai-log.md` entry.

**When it went out of scope:**
> Revert everything not required by task <ID>, then show me the remaining diff summary.

**When you need to understand the code for the interview:**
> Explain `<file>` line by line as if I'm being interviewed on it. Include why each non-obvious decision was made and what the alternatives were.

---

## Appendix B — Figma reference

- **File** (Prashuk's editable copy): `https://www.figma.com/design/D5rZDKQK4OsuCgcOphaxdG/Design-Task---PLP--Copy-`
- **File key:** `D5rZDKQK4OsuCgcOphaxdG`

| Frame | Node ID | Use for |
|-------|---------|---------|
| Web/PLP/With Filter | `653:492` | Default desktop layout (3-column grid + sidebar) |
| Web/PLP/With Filter Expanded | `653:1814` | Open accordion (checkboxes, counts, "Unselect all") and open sort menu |
| Web/PLP/Hidden Filter | `653:3588` | 4-column grid, "Show filter" state |
| Phone/PLP | `653:3166` | Mobile layout, FILTER/RECOMMENDED bar, footer accordions |

**Connect the Figma MCP to Claude Code** (Prashuk, once):

```bash
claude mcp add --transport http figma https://mcp.figma.com/mcp
```

Then run `/mcp` inside Claude Code to authenticate. If the command differs in Figma's current docs, follow their docs.
