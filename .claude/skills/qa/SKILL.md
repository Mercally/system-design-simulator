---
name: qa
description: Verify a finished piece of work before commit/PR — build, type-check, lint, structure review, and browser check for UI changes. Use after finishing a dev task, or when the user says "run qa" or "/qa".
---

# QA

Run this **after** finishing a dev task, before commit/PR.

## Steps

1. `npm run build` — must pass (this also runs `tsc`, per this repo's CLAUDE.md).
2. `npx tsc --noEmit` if you want a faster type-only check.
3. `npm run lint`.
4. Review the diff (`git diff`) for structure/readability issues: dead code, misplaced logic, violations of `CLAUDE.md` invariants (simulation engine, scoring, store persistence rules — check `CLAUDE.md` "Key invariants" section against what changed).
5. **If the change touches UI**: start the dev server and actually exercise the feature in a browser (use the chrome-devtools MCP tools) — golden path plus edge cases. This repo has no unit tests; this is the real verification step for UI work, per `CLAUDE.md`.
6. Report pass/fail per check. If something fails and the cause is a known/mechanical fix, say so and suggest running `/pr-fix` — don't invoke it automatically.

## Notes

- Don't invent test suites or add testing frameworks — this repo intentionally has none (see `CLAUDE.md`).
- Never claim "tests pass" or "verified" without having actually run the commands above.
