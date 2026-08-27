# Agent flow

Local-only agent team, run via Claude Code skills. No cron, no GitHub Actions, no cloud triggers — every agent is invoked manually (`/name`) or offered proactively by the main session when the moment fits. Replaces the earlier gh-aw cloud setup, which is no longer part of this repo.

There is no separate orchestrator agent: the main Claude Code session (you + Claude) decides when to invoke each skill, the same way it decides to invoke any other skill.

| Agent (`/command`) | Responsibility | Triggers | Tools | Can suggest |
|---|---|---|---|---|
| `/triage <issue#>` | Validate whether an issue is workable: duplicate/spam check, missing-context check, suggest type/labels | Manual, **before** starting work on an issue | `gh` CLI (Bash), WebSearch (only if an external error/API needs lookup) | — |
| `/qa` | Verify a finished change: `npm run build`, `tsc --noEmit`, lint, structure/CLAUDE.md-invariant review, browser check for UI changes | Manual, **after** finishing a dev task, before commit/PR | Bash (npm scripts), Read/Grep, chrome-devtools MCP (UI verification) | `/pr-fix`, if it finds a fixable failure — never auto-invoked |
| `/plan` | Build/refresh roadmap from open issues/PRs and priorities | Manual, or offered when no recent plan exists | Bash (`gh` CLI), Read | — |
| `/repo-status` | Snapshot of recent commits/PRs/issues to refresh project-state context | Manual, or offered before `/plan` if recent state is unclear | Bash (`gh` CLI, `git log`) | — |
| `/pr-fix <pr#>` | Fix a PR with failing CI: read logs, implement fix, verify, push, comment | Manual, when CI is red on a PR | Bash (`git`, npm scripts), `gh` CLI | — |

## Design rules

- No skill applies GitHub-visible changes (labels, closes, comments, pushes) without the user's go-ahead in chat first — these are advisory/assistive, not autonomous.
- `/qa` never claims a check passed without having actually run it.
- None of these invent test infrastructure — this repo has no unit tests by design (see root `CLAUDE.md`); verification is build + type-check + lint + manual browser testing.
