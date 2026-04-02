---
description: Multi-dimensional PR review with parallel specialized reviewers. Auto-discovers PRs across all Talgent Gitea repos.
argument-hint: [pr-number-or-branch-name]
allowed-tools: Bash, Read, Agent, AskUserQuestion
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
   $json = & "D:/Code/Talgent/.claude/skills/gitea/Invoke-Gitea.ps1" -Endpoint "/orgs/Talgent/repos?limit=50"
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

   Error handling:
   - If the command exits with non-zero status, display the stderr message and stop.
   - If PR_DIFF starts with `<!DOCTYPE` or `<html`, the fetch returned a login page. Display "Error: Failed to fetch diff (got HTML instead). Check Gitea credentials." and stop.

   IMPORTANT: No git fetch or local branch creation needed. The diff comes entirely from the Gitea API.

7. Display diff stats summary:
   ```
   PR #{PR_ID} in {REPO_NAME}: {PR_TITLE}
   Branch: {head.ref} -> {base.ref}
   Diff: {additions} additions, {deletions} deletions, {total} total lines
   ```

### Phase 3: Launch 5 Parallel Reviewers

8. Check diff is non-empty:
   ```
   If PR_DIFF is empty, display "This PR has no changes to review." and stop.
   ```

9. Determine DIFF_SIZE (line count of PR_DIFF):
   - If DIFF_SIZE > 5000: Display "Diff is very large ({DIFF_SIZE} lines). This will be sent to 5 agents. Continue? (y/n)" and wait for confirmation. If user declines, stop.
   - If DIFF_SIZE > 3000: Display "⚠️ Review is based on partial diff (first 3000 of {DIFF_SIZE} total lines). Agents will focus on critical/high-severity issues only."

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

    If DIFF_SIZE > 3000, append: "IMPORTANT: This is a LARGE diff ({DIFF_SIZE} lines). Focus only on CRITICAL and HIGH severity issues. Limit your output to top 5-7 most important findings."

    CRITICAL CONSTRAINTS:
    - The SAME diff content must be passed to all 5 reviewers - do not truncate/summarize differently per reviewer
    - If truncating diff (any size), you MUST inform user: "⚠️ Review is based on partial diff (first X lines)"

11. Launch all 5 reviewer agents in parallel using the **Agent tool** (NOT Task tool). Send a SINGLE message with FIVE Agent tool calls:

    CRITICAL - Use this EXACT Agent tool call format for each agent:
    ```json
    {
      "subagent_type": "code-review:<agent-name>",
      "description": "Review PR {PR_ID} ({REPO_NAME}) for <review-focus>",
      "prompt": "<prompt from step 10>",
      "run_in_background": true
    }
    ```

    Launch these 5 agents:
    | Agent | subagent_type | description |
    |-------|--------------|-------------|
    | Bug & Correctness | `code-review:bug-correctness-reviewer` | "Review PR {PR_ID} ({REPO_NAME}) for bugs and correctness" |
    | Security | `code-review:security-reviewer` | "Review PR {PR_ID} ({REPO_NAME}) for security vulnerabilities" |
    | Performance | `code-review:performance-reviewer` | "Review PR {PR_ID} ({REPO_NAME}) for performance issues" |
    | Code Quality | `code-review:code-quality-reviewer` | "Review PR {PR_ID} ({REPO_NAME}) for code quality" |
    | Architecture | `code-review:architecture-reviewer` | "Review PR {PR_ID} ({REPO_NAME}) for architectural design" |

12. Display:
    ```
    5 reviewer agents launched for PR #{PR_ID} ({REPO_NAME}).
    Waiting for background agents to complete...
    ```

### Phase 4: Collect and Synthesize

13. **STOP. Do NOTHING. Just end your response and wait.**

    After launching agents in step 11, you MUST end your current response immediately. Do NOT call any tool. Do NOT try to collect results. Do NOT call TaskOutput (it is a completely different system and WILL fail with "No task found"). Do NOT read output files. Do NOT synthesize manually.

    Background agents deliver results automatically. When each agent finishes, you will receive a `<task-notification>` message containing the agent's review in its `<result>` field. This happens WITHOUT you doing anything.

    **After receiving all 5 notifications** (or after the user tells you to proceed with partial results), move to step 14.

    If the user asks "where are the results?", explain that background agents are still running and results will arrive automatically. Ask: "X of 5 reviewers have responded so far. Wait for remaining agents or proceed with available results?"

14. Once all available agent results are collected from `<task-notification>` messages, synthesize into a final report:
    - **Summary**: PR ID, repo name, title, branch info, diff size, number of reviewers that responded
    - **Critical Blockers**: Issues that MUST be fixed, with severity (Critical/High/Medium) and file:line references
    - **Final Verdict**: Approve / Approve with suggestions / Request changes
    - **Average Rating**: Calculate average of responding reviewers' ratings (1-10)
    - **Prioritized Action List**: Top 5-10 action items, prioritized by impact

## Agent Output Format

Each agent outputs structured markdown with findings, code snippets, and ratings (see individual agent files for details).

### Input Validation

- SEARCH_TERM can be numeric (PR number) or alphanumeric (branch name)
- PR numbers are per-repo, so the same number can exist in multiple repos - always search ALL repos
- Branch names are matched as substrings (e.g., "TD-17" matches "TD-17-fix-oauth")
- Only open PRs are searched by default. Mention `state=closed` or `state=all` if no matches found
