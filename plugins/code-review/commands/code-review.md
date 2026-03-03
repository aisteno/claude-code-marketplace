---
allowed-tools: Bash(gh issue view:*), Bash(gh search:*), Bash(gh issue list:*), Bash(gh pr comment:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(gh api:*), Bash(jq:*), mcp__github_inline_comment__create_inline_comment, mcp__github_comment__update_claude_comment
description: Code review a pull request
argument-hint: "[--comment] [owner/repo/pull/123]"
---

Provide a code review for the given pull request.

**Agent assumptions (applies to all agents and subagents):**
- All tools are functional and will work without error. Do not test tools or make exploratory calls. Make sure this is clear to every subagent that is launched.
- Only call a tool if it is required to complete the task. Every tool call should have a clear purpose.

To do this, follow these steps precisely:

1. Create a todo list before starting. Keep it updated as tasks complete.

2. Launch a haiku agent to check if any of the following are true:
   - The pull request is closed
   - The pull request is a draft
   - The pull request does not need code review (e.g. automated PR, trivial change that is obviously correct)

   If any condition is true, stop and do not proceed.

   Note: still review Claude-generated PRs.

3. Launch a haiku agent to return a list of file paths (not their contents) for all relevant CLAUDE.md files including:
   - The root CLAUDE.md file, if it exists
   - Any CLAUDE.md files in directories containing files modified by the pull request

4. Launch a sonnet agent to view the pull request and return a summary of the changes.

5. Launch 4 agents in parallel to independently review the changes. Each agent should return issues with a reason (e.g. "CLAUDE.md adherence", "bug").

   Agents 1 + 2: CLAUDE.md compliance sonnet agents
   - Audit changes for CLAUDE.md compliance in parallel.
   - When evaluating CLAUDE.md compliance for a file, only consider CLAUDE.md files scoped to that file path and its parents.

   Agent 3: Opus bug agent
   - Scan for obvious bugs in the diff.
   - Flag only significant, high-confidence issues.

   Agent 4: Opus bug/security/regression agent
   - Look for correctness, security, and regression issues in changed code.

   **CRITICAL: We only want HIGH SIGNAL issues.** Flag issues where:
   - The code will fail to compile or parse (syntax errors, type errors, missing imports, unresolved references)
   - The code will definitely produce wrong results regardless of inputs (clear logic errors)
   - There is a concrete security issue in changed code
   - There is a clear, unambiguous CLAUDE.md violation where you can quote the exact rule being broken

   Do NOT flag:
   - Code style or quality concerns
   - Potential issues that depend on specific unknown inputs or runtime state
   - Subjective suggestions or refactor preferences
   - Linter-only concerns

   If you are not certain an issue is real, do not flag it.

   In addition, each subagent should be given the PR title and description.

6. For each issue found in step 5, launch parallel validation subagents.
   - Validate that each issue is truly real, in-scope, and high-confidence.
   - Use Opus validators for bugs/security/logic issues.
   - Use Sonnet validators for CLAUDE.md scope/compliance checks.

7. Filter out any unvalidated issues. Remaining issues become the final findings.

8. Assign severity to final findings:
   - High: must fix before merge (correctness break, concrete security flaw, severe regression)
   - Medium: should fix in this PR or before release
   - Low: optional improvement; include only if concrete and directly tied to changed code

9. Incremental continuity (required):
   - If `--comment` is provided, fetch existing PR comments and find prior Claude sticky summary using marker `<!-- steno-claude-review -->`.
   - Generate stable issue IDs in format `<path>#L<line>:<short-slug>`.
   - Compare against prior IDs and classify:
     - New Findings
     - Still Open
     - Resolved Since Last Review
   - Do not re-flag resolved findings as new.

10. Output a summary to terminal:
   - If issues were found, list them by severity and continuity classification.
   - If no High/Medium issues were found, state exactly:
     `No blocking issues in this revision.`

   If `--comment` argument was NOT provided, stop here. Do not post GitHub comments.

11. If `--comment` argument IS provided, publish one sticky summary comment (no duplicates):
   - If prior marker comment exists, update it in place.
   - Else create one comment.
   - Include markers at the end:
     - `<!-- steno-claude-review -->`
     - `<!-- steno-claude-review-head:<HEAD_SHA> -->`

12. Inline comments:
   - Do not post inline comments unless explicitly requested.
   - If explicitly requested, post one inline comment per unique issue only.

Use this list when evaluating issues in Steps 5 and 6 (these are false positives, do NOT flag):

- Pre-existing issues
- Something that appears to be a bug but is actually correct
- Pedantic nitpicks that a senior engineer would not flag
- Issues that a linter will catch (do not run the linter to verify)
- General code quality concerns (e.g., lack of test coverage, broad non-concrete security claims) unless explicitly required in scoped CLAUDE.md
- Issues mentioned in CLAUDE.md but explicitly silenced in code

Notes:

- Use gh CLI to interact with GitHub. Do not use web fetch.
- Keep findings concise, actionable, and tied to exact changed code.
- Use severity tiers in the summary (`High`, `Medium`, `Low`).
- Prefer sticky-summary continuity over repeated fresh comments.
