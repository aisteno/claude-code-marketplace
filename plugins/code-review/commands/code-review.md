---
description: Incremental, high-signal PR review with sticky state
argument-hint: "[--comment] [owner/repo/pull/123]"
allowed-tools: ["Bash(gh pr view:*)", "Bash(gh pr diff:*)", "Bash(gh pr comment:*)", "Bash(gh api:*)", "Bash(jq:*)", "Bash(git rev-parse:*)", "Bash(git diff:*)", "Read", "Task"]
---

Perform an incremental code review for a pull request with low-noise output.

Primary goals:
- Review only meaningful risk introduced by the latest revision when possible.
- Avoid repeating already-addressed feedback.
- Prefer "good enough" over nitpicks.
- Keep feedback short and actionable.

Arguments:
- Optional PR target: `owner/repo/pull/123`
- Optional `--comment` flag to publish/update PR comment.

Rules:
1. Skip review if PR is closed or draft.
2. Skip trivial PRs (docs/changelog/version-only changes).
3. Never report style nits, minor refactors, or linter-only issues.
4. Report only high-confidence issues in these buckets:
   - correctness bugs
   - security problems
   - clear regressions
   - explicit CLAUDE.md violations
5. If uncertain, do not report.
6. Cap total reported issues to 5.

Incremental scope:
1. Resolve PR target:
   - If argument includes `owner/repo/pull/123`, use it.
   - Else infer from current checkout/context via `gh pr view`.
2. Determine comparison range:
   - If this is a `synchronize` event and `before/after` SHAs exist in `GITHUB_EVENT_PATH`, review `before..after`.
   - Else use `base.sha..head.sha` from PR metadata.
3. Build review scope from that range and focus analysis on changed hunks in this range.

Review process:
1. Gather relevant CLAUDE.md files (root + directories of changed files).
2. Launch two parallel agents:
   - Agent A: high-signal bug/regression/security audit on changed hunks.
   - Agent B: CLAUDE.md compliance audit only for scoped files.
3. For each candidate finding, run one validation subagent to confirm it is real and in-scope.
4. Keep only validated high-confidence findings.

Deduplication and continuity:
1. Find prior Claude summary comment with marker `<!-- steno-claude-review -->`.
2. Extract prior issue IDs from markdown checkboxes in that comment.
3. For each current finding, generate stable ID:
   - `ID = <path>#L<line>:<short-slug>`
4. Classify:
   - New Findings: IDs now present, not previously present.
   - Still Open: IDs present now and previously present.
   - Resolved Since Last Review: IDs previously present but not present now.

Output format (always):

## Incremental Review

### New Findings
- [ ] `<ID>`: <short finding>

### Still Open
- [ ] `<ID>`: <short finding>

### Resolved Since Last Review
- [x] `<ID>`

If no actionable issues remain, write exactly:
`No blocking issues in this revision.`

Publishing behavior:
- If `--comment` is NOT provided: print results and stop.
- If `--comment` is provided:
  1. Include markers at end of comment:
     - `<!-- steno-claude-review -->`
     - `<!-- steno-claude-review-head:<HEAD_SHA> -->`
  2. If prior marker comment exists, edit it in place via `gh api`.
  3. Else create one PR comment.

Do not post inline comments unless explicitly requested.
Do not create multiple summary comments when marker comment exists.
