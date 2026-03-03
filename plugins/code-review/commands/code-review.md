---
description: High-signal incremental PR review with continuity and severity tiers
argument-hint: "[--comment] [owner/repo/pull/123]"
allowed-tools: ["Bash(gh pr view:*)", "Bash(gh pr diff:*)", "Bash(gh pr list:*)", "Bash(gh pr comment:*)", "Bash(gh api:*)", "Bash(jq:*)", "Bash(git rev-parse:*)", "Bash(git diff:*)", "Read", "Task", "mcp__github_comment__update_claude_comment"]
---

Provide an incremental code review for the given pull request.

This command keeps Anthropic-style rigor (multi-agent + validation + todo discipline), while adding continuity across pushes and reducing repeated noise.

Arguments:
- Optional PR target: `owner/repo/pull/123`
- Optional `--comment` flag to publish/update a sticky PR review comment

Agent assumptions (applies to all agents and subagents):
- Tools are functional; do not make exploratory calls.
- Only call tools required to complete the task.
- False positives are worse than missing low-signal suggestions.

Rules:
1. Skip review if PR is closed or draft.
2. Skip trivial PRs (docs/changelog/version-only changes) unless explicitly asked to review anyway.
3. Never report style nits, minor refactors, or linter-only concerns.
4. Report only high-confidence findings in changed code:
   - correctness bugs
   - security problems
   - regressions
   - explicit, in-scope CLAUDE.md violations
5. If uncertain, do not report.
6. Do not use a hard issue cap. Report all validated High/Medium findings.

Step-by-step process:

1. Create and maintain a todo checklist before starting. Mark tasks done as you go.

2. Resolve PR target:
   - If argument includes `owner/repo/pull/123`, use it.
   - Else infer from current context (`gh pr view` or provided PR metadata).

3. Determine incremental review range:
   - If synchronize event has `before/after` SHAs in `GITHUB_EVENT_PATH`, use `before..after`.
   - Else use `base.sha..head.sha`.
   - Focus primary analysis on changed hunks in this range.

4. Gather context:
   - PR title/body and intent.
   - Relevant CLAUDE.md files:
     - root CLAUDE.md (if present)
     - CLAUDE.md in directories containing changed files (and applicable parents)

5. Run adaptive parallel review agents:
   - For small/medium diffs: 2 parallel agents
     - Agent A: bug/security/regression review
     - Agent B: CLAUDE.md compliance review
   - For larger or riskier diffs: 4 parallel agents
     - Two CLAUDE.md compliance agents
     - Two bug/security/regression agents
   - Pass PR title/body context to all agents.

6. Validation pass:
   - For every candidate finding, run a validation subagent.
   - Keep only findings validated as real, in-scope, and high-confidence.

7. Assign severity tiers:
   - High: must fix before merge (correctness break, clear security flaw, severe regression).
   - Medium: should fix before release or within this PR if low effort.
   - Low: optional improvement; include only if concrete, in changed code, and non-speculative.

8. Continuity and deduplication:
   - Find prior Claude sticky comment marker: `<!-- steno-claude-review -->`.
   - Extract prior issue IDs from checklist items.
   - Use stable issue IDs: `<path>#L<line>:<short-slug>`.
   - Classify findings:
     - New Findings
     - Still Open
     - Resolved Since Last Review
   - Do not re-flag already-resolved findings.
   - Prefer updating prior status over creating new noise.

9. Output format:

## Incremental Code Review

### High (Must Fix)
- [ ] `<ID>`: <short finding>

### Medium
- [ ] `<ID>`: <short finding>

### Low (Optional)
- [ ] `<ID>`: <short finding>

### Still Open from Previous Review
- [ ] `<ID>`: <short finding>

### Resolved Since Last Review
- [x] `<ID>`

When no High/Medium findings remain, include exactly:
`No blocking issues in this revision.`

10. Publishing behavior:
 - If `--comment` is NOT provided: print results and stop.
 - If `--comment` is provided:
   - Use one sticky summary comment only.
   - Include markers:
     - `<!-- steno-claude-review -->`
     - `<!-- steno-claude-review-head:<HEAD_SHA> -->`
   - If marker comment exists, update it in place (prefer `mcp__github_comment__update_claude_comment`; fallback to `gh api` edit).
   - Else create one comment.

11. Comment hygiene:
 - Do not post inline comments unless explicitly requested.
 - Do not create multiple summary comments when a marker comment already exists.
 - Keep findings concise, actionable, and tied to exact file/line.

False-positive guardrails (never flag):
- Pre-existing issues not introduced by current changes.
- Speculative concerns requiring assumptions about unknown runtime state.
- Style-only preferences.
- Issues that are explicitly allowed/silenced by scoped CLAUDE.md rules.
