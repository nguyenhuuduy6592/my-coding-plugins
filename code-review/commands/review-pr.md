---
description: Multi-dimensional PR review with parallel specialized reviewers. Auto-discovers PRs across all Talgent Gitea repos.
argument-hint: [pr-number-or-branch-name]
allowed-tools: Bash, Read, Task, AskUserQuestion
---

# review-pr - Pull Request Code Review

This command performs a comprehensive PR review using 5 parallel specialized reviewers. It auto-discovers which Gitea repo the PR belongs to.

## Workflow

### Phase 1: Discover PR Across Repos

1. Determine SEARCH_TERM from arguments or user input:
   - If `$1` is provided, use it as SEARCH_TERM (can be a PR number like `3` or a branch name like `TD-17`)
   - If `$1` is not provided, ask the user: "Enter a PR number or branch name to review"
   - Wait for user to provide the value before proceeding

2. Query all Talgent org repos to find ones with open PRs:
   ```bash
   MSYS_NO_PATHCONV=1 powershell -Command '
   $json = powershell -File D:/Code/Talgent/.claude/skills/gitea/Invoke-Gitea.ps1 -Endpoint "/orgs/Talgent/repos?limit=50"
   $repos = $json | ConvertFrom-Json
   $repos | Where-Object { $_.open_pr_counter -gt 0 } | ForEach-Object { $_.name }
   '
   ```

3. For each repo with open PRs, list all open PRs:
   ```bash
   MSYS_NO_PATHCONV=1 powershell -File D:/Code/Talgent/.claude/skills/gitea/Invoke-Gitea.ps1 -Endpoint "/repos/Talgent/{REPO_NAME}/pulls?state=open"
   ```
   You can query multiple repos in parallel (separate Bash calls).

4. Match PRs against SEARCH_TERM. IMPORTANT: collect ALL matches across ALL repos before deciding:
   - If SEARCH_TERM is numeric: match by PR `number` field. Note: the same PR number can exist in different repos, so check every repo with open PRs.
   - If SEARCH_TERM is non-numeric: match by `head.ref` (branch name) using substring match

5. Handle match results:
   - **0 matches**: Display "No open PR found matching '{SEARCH_TERM}' across all Talgent repos." and mention that only open PRs are searched (use `state=closed` or `state=all` if reviewing a merged PR). Stop.
   - **1 match**: Use it. Display: "Found PR #{number} in {repo}: {title} ({head.ref} -> {base.ref})"
   - **Multiple matches**: Display numbered list and ask user to pick:
     ```
     Multiple PRs found matching '{SEARCH_TERM}':
     1. [WebApp] PR #368: WKF-560 Title here (WKF-560 -> develop)
     2. [Backend] PR #3: TD-17 Other title (TD-17 -> master)
     Enter number to review:
     ```

   Store the matched PR's repo name as REPO_NAME, PR number as PR_ID, PR title as PR_TITLE, and `diff_url` as DIFF_URL.

### Phase 2: Fetch PR Diff via Gitea API

6. Fetch the PR diff using the helper script's `-RawUrl` parameter (uses Basic auth for web URLs):
   ```bash
   MSYS_NO_PATHCONV=1 powershell -File D:/Code/Talgent/.claude/skills/gitea/Invoke-Gitea.ps1 -RawUrl "{DIFF_URL}"
   ```
   Store the output as PR_DIFF.

   Validate the response: if PR_DIFF starts with `<!DOCTYPE` or `<html`, the fetch returned a login page instead of diff content. Display "Error: Failed to fetch diff (got HTML instead of diff content). Check Gitea credentials." and stop.

   IMPORTANT: No git fetch or local branch creation needed. The diff comes entirely from the Gitea API.

7. Display diff stats summary:
   ```
   PR #{PR_ID} in {REPO_NAME}: {PR_TITLE}
   Branch: {head.ref} -> {base.ref}
   Diff size: {line count} lines
   ```

### Phase 3: Launch 5 Parallel Reviewers

8. Check diff is non-empty:
   ```
   If PR_DIFF is empty, display "This PR has no changes to review." and stop.
   ```

9. Determine DIFF_SIZE (line count of PR_DIFF). If DIFF_SIZE > 1000:
   ```
   "Large diff detected ({DIFF_SIZE} lines). Agents will focus on critical/high-severity issues only."
   ```

10. Construct agent prompt (substitute {PR_ID}, {REPO_NAME}, {PR_TITLE}, and {PR_DIFF} with actual values):
    ```
    CRITICAL: You are reviewing PR #{PR_ID} in repo {REPO_NAME}: "{PR_TITLE}".

    Below is the COMPLETE diff output for this PR. Analyze ONLY this diff content:

    --- START OF DIFF ---
    {PR_DIFF}
    --- END OF DIFF ---

    IMPORTANT INSTRUCTIONS:
    - Analyze ONLY the code shown in the diff above (lines added/modified with + prefix)
    - Do NOT explore any files with Glob/Grep/Read - the diff contains everything you need
    - Do NOT analyze files from the current working directory
    - Focus your review on the changes shown in this diff

    Provide your specialized review following your system prompt output format.
    ```

    If DIFF_SIZE > 1000, append: "IMPORTANT: This is a LARGE diff ({DIFF_SIZE} lines). Focus only on CRITICAL and HIGH severity issues. Limit your output to top 5-7 most important findings."

11. Launch all 5 reviewer agents in parallel using a SINGLE message with FIVE Task tool calls:
    - `bug-correctness-reviewer`
    - `security-reviewer`
    - `performance-reviewer`
    - `code-quality-reviewer`
    - `architecture-reviewer`

    CRITICAL - Use this EXACT Task tool call format for each agent:
    ```json
    {
      "subagent_type": "code-review:<agent-name>",
      "description": "Review PR {PR_ID} ({REPO_NAME}) for <review-focus>",
      "prompt": "<prompt from step 10>",
      "run_in_background": true,
      "timeout": 600000
    }
    ```

    Where `<review-focus>` is specific to each agent:
    - bug-correctness-reviewer: "bugs and correctness"
    - security-reviewer: "security vulnerabilities"
    - performance-reviewer: "performance issues"
    - code-quality-reviewer: "code quality"
    - architecture-reviewer: "architectural design"

12. Display the captured task IDs:
    ```
    Reviewer tasks launched for PR #{PR_ID} ({REPO_NAME}):
    - bug-correctness-reviewer: <task_id_1>
    - security-reviewer: <task_id_2>
    - performance-reviewer: <task_id_3>
    - code-quality-reviewer: <task_id_4>
    - architecture-reviewer: <task_id_5>
    ```

### Phase 4: Collect and Synthesize

13. Display: "Collecting and synthesizing reviews from all agents..."

14. Collect agent outputs using TaskOutput with the captured task IDs from step 12:
    ```json
    {
      "task_id": "<task_id>",
      "block": true
    }
    ```
    Send all 5 TaskOutput calls in ONE message. Handle errors gracefully.

15. Synthesize all agent outputs into a final report:
    - **Summary**: PR ID, repo name, title, branch info, diff size
    - **Critical Blockers**: Issues that MUST be fixed, with severity (Critical/High/Medium) and file:line references
    - **Final Verdict**: Approve / Approve with suggestions / Request changes
    - **Average Rating**: Calculate average of all 5 reviewers' ratings (1-10)
    - **Prioritized Action List**: Top 5-10 action items, prioritized by impact

## Agent Output Format

Each agent outputs structured markdown with findings, code snippets, and ratings (see individual agent files for details).

### Input Validation

- SEARCH_TERM can be numeric (PR number) or alphanumeric (branch name)
- PR numbers are per-repo, so the same number can exist in multiple repos - always search ALL repos
- Branch names are matched as substrings (e.g., "TD-17" matches "TD-17-fix-oauth")
- Only open PRs are searched by default. Mention `state=closed` or `state=all` if no matches found
