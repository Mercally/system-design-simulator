---
name: pr-fix
description: Fix a pull request with failing CI — reads failure logs, identifies root cause, implements a fix, runs tests/lint, and pushes to the PR branch. Use when CI is red on a PR, or when the user says "/pr-fix".
---

# PR Fix

Run when a PR's CI is failing and needs a fix pushed to its branch.

## Steps

1. Check out the PR branch: `gh pr checkout <N>`.
2. Get the failing run's logs: `gh run list --branch <branch> --limit 5`, then `gh run view <run-id> --log-failed`.
3. Identify the root cause from the error output.
4. Implement the fix.
5. Verify: `npm run build`, `npx tsc --noEmit`, `npm run lint` — must all pass before pushing.
6. Push to the PR branch: `git push`.
7. Comment on the PR summarizing what was fixed and why (`gh pr comment <N> --body "..."`).

## Notes

- Confirm with the user before pushing if the fix touches more than the failing area — scope creep here affects someone else's PR.
