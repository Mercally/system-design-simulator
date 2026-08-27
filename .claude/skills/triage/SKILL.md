---
name: triage
description: Validate whether a GitHub issue is ready to work on — checks for duplicates, spam, missing context, suggests labels/type. Use before starting work on an issue, or when the user says "triage issue #N" or "/triage".
---

# Triage

Run this **before** picking up an issue, to decide if it's actually workable.

## Steps

1. Fetch the issue: `gh issue view <N> --comments`.
2. List repo labels: `gh label list`.
3. Search for duplicates/related issues: `gh issue list --search "<keywords>"`.
4. Assess:
   - **Spam/invalid** — gibberish, bot-generated, test issue. Recommend closing (`gh issue close <N> --reason "not planned"`), don't go further.
   - **Incomplete** — missing repro steps, expected/actual behavior, environment. Draft a comment asking for what's missing, don't apply labels yet.
   - **Ready** — has enough to act on. Suggest issue type (bug/feature/task), suggest labels from the existing label set only (don't invent new ones), note duplicates/related issues found.
5. Report a short verdict: **Ready to work / Needs info / Not suitable (reason)**. If ready, note anything relevant for the dev step (repro steps, likely files, open questions).

## Notes

- Don't apply labels/close the issue without confirming with the user first — this is advisory, not autonomous.
- Use `WebSearch` only if the issue references an external error/API that needs lookup.
