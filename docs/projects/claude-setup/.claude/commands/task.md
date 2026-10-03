---
description: Plan and implement one task from docs/development-plan.md
argument-hint: <task-id, e.g. A-03>
---

Task: **$ARGUMENTS**

1. Read the root `CLAUDE.md` and the `CLAUDE.md` of the app this task touches (`frontend/` or `backend/`).
2. Find task `$ARGUMENTS` in `docs/development-plan.md` §4. Read its row, its "done when" check, and every doc section it depends on (`product.md`, `frontend-plan.md`, `backend-plan.md`, `dbmodal.md`). If the task ID doesn't exist, say so and stop.
3. Reply with a plan only:
   - files to create or change, with one line each on why,
   - the approach in 3–6 bullets,
   - anything ambiguous or conflicting between docs (as questions).
4. **Stop and wait for "go".** Do not write code yet.
5. After "go":
   - implement the smallest change that meets the "done when",
   - follow the readability and comment rules in `CLAUDE.md`,
   - run lint, typecheck, tests (if any) and the task's "done when" check, and fix failures.
6. Report in the format from root `CLAUDE.md` §3 step 5: what changed · how I verified · deviations · explain-it notes · suggested commit message.
7. Update this task's row in `docs/progress.md` (status, date, one-line note). Prashuk will be asked to approve the edit.
8. Remind Prashuk to run `/review` and commit. Then stop. No git writes. Do not start the next task.
