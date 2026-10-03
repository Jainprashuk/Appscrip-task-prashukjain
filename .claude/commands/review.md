---
description: Review the uncommitted changes before Prashuk commits
---

Review the current uncommitted changes (`git status` and `git diff`) before Prashuk commits them. **Do not modify any files.** Report findings only.

Check each changed file against:

1. **Docs:** does it match `docs/` (API shapes, schema, URL contract, component list, design decisions)? Quote the doc section for any mismatch.
2. **Root `CLAUDE.md` §4 readability:** naming, function size, nesting, no clever code, no `any`/unexplained casts, explicit exported types.
3. **Root `CLAUDE.md` §5 comments:** file header present; TSDoc on exports; why-comments on non-obvious lines; no commented-out code or bare TODOs; no stale comments.
4. **App `CLAUDE.md` rules:**
   - frontend: server/client boundary, no browser fetching, `search-params.ts` usage, CSS Modules + tokens, headings, a11y.
   - backend: layering table, validation, `HttpError`, `id` tie-breaker, cents, `select` over `include`.
5. **Scope:** any change unrelated to the current task.
6. **Dependencies:** any `package.json` change not on the allowed list.
7. **Secrets:** anything that looks like a credential or a real connection string.

Output:
- a table: file · issue · rule/doc reference · suggested fix (one row per issue, most serious first);
- "Ready to commit" or "Fix before committing";
- a suggested Conventional Commit message for the changes.
