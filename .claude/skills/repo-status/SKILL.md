---
name: repo-status
description: Snapshot of recent repo activity (commits, PRs, issues) to refresh context on project progress. Use on request, or proactively before planning if recent state is unclear.
---

# Repo Status

Quick situational snapshot. Run manually, or offer to run it before `/plan` if recent activity isn't already known from context.

## Steps

1. `git log --oneline -20` — recent commits.
2. `gh pr list --state all --limit 10` — recent PR activity.
3. `gh issue list --state all --limit 10` — recent issue activity.
4. Summarize concisely: what shipped recently, what's in progress, anything stalled (old open PR/issue with no activity).

## Notes

- This is a read-only status check, not a report — keep the summary short, no need for GitHub issue/discussion output.
