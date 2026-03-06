---
allowed-tools: Bash(gh pr comment:*),Bash(gh pr diff:*),Bash(gh pr view:*)
description: Review a pull request
argument-hint: "[--comment] [owner/repo/pull/123]"
---

Perform a comprehensive code review using subagents for key areas:

- code-quality-reviewer
- performance-reviewer
- test-coverage-reviewer
- documentation-accuracy-reviewer
- security-code-reviewer

Instruct each to only provide noteworthy feedback. Once they finish, review the feedback and post only the feedback that you also deem noteworthy.

Review only the current pull request diff and current pull request state.
If this is a later review of the same pull request in the same resumed session, do not repeat issues that are already fixed in the current revision.
If you do not find any noteworthy remaining issues, say so plainly instead of stretching to find more.

If `--comment owner/repo/pull/123` is provided, post a fresh top-level review comment to that pull request.
Do not update prior review comments.

Provide feedback using inline comments for specific issues.
Use top-level comments for general observations or praise.
Keep feedback concise.

---
