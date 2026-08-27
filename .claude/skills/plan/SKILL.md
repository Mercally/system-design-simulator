---
name: plan
description: Build or refresh the project roadmap from open issues/PRs, priorities and dependencies. Use on request, or proactively when no recent plan exists and the user is deciding what to work on next.
---

# Plan

Roadmap/backlog pass. Run manually, or offer to run it if the project state looks stale (no plan discussed recently, backlog is large/unsorted) before helping the user pick next work.

## Steps

1. Gather state: `gh issue list --state open`, `gh pr list --state open`, recently closed issues (`gh issue list --state closed --limit 20`) for what just shipped.
2. Summarize: what's done, what's in flight (open PRs), what's open and unprioritized.
3. Propose a prioritized list of remaining work with rough dependencies between items.
4. For genuinely new work not yet tracked as an issue, list suggested `gh issue create --title ... --label ...` commands — don't run them, just show them for the user to approve.

## Notes

- Don't create issues or discussions automatically — this is a planning aid, output goes to chat (or a file the user names), not to GitHub.
