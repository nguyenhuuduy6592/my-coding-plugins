---
name: review-pr
description: Comprehensive multi-dimensional pull request review with 5 parallel specialized reviewers
argument-hint: [PR_ID]
allowed-tools: ["Bash", "AskUserQuestion", "Task"]
---

# review-pr - Pull Request Code Review

This command performs a comprehensive PR review using 5 parallel specialized reviewers.

## Workflow

**Step 1: Gather PR Information**

1. Ask the user for the PR ID (just the number, e.g., `271`)
   - Wait for user to provide the PR ID before proceeding

2. Ask the user for the base branch to compare against
   - Default: `develop`
   - Allow user to change it (e.g., `main`, `master`)
   - Present as: "Base branch to compare against (default: develop)?"

**Step 2: Fetch and Show PR Overview**

3. Fetch the PR head safely:
   ```bash
   git fetch origin pull/${PR_ID}/head:pr-${PR_ID}-temp
   ```
   - If this fails, show an error message and abort
   - Error message should include possible causes (PR doesn't exist, network issues, remote not named `origin`)

4. List all commits unique to the PR:
   ```bash
   git log --oneline --graph ${BASE_BRANCH}...pr-${PR_ID}-temp
   ```

5. Show diff stats summary:
   ```bash
   git diff --stat ${BASE_BRANCH}...pr-${PR_ID}-temp
   ```

6. If no commits are found (PR already merged), show message and exit:
   - "This PR has no commits unique to ${BASE_BRANCH}. It may already be merged."
   - Exit gracefully

**Step 3: Launch 5 Parallel Reviewers**

7. Launch all 5 reviewer agents in parallel using a single message with multiple Task tool calls:
   - `bug-correctness-reviewer` - Analyze for bugs and correctness issues
   - `security-reviewer` - Analyze for security vulnerabilities
   - `performance-reviewer` - Analyze for performance issues
   - `code-quality-reviewer` - Analyze for code quality issues
   - `architecture-reviewer` - Analyze for architecture and best practices

   Each agent should receive:
   - PR ID
   - Base branch
   - Temp branch name (`pr-${PR_ID}-temp`)
   - The git diff command to analyze: `git diff ${BASE_BRANCH}...pr-${PR_ID}-temp`

   Use this prompt for each agent:
   ```
   Review the pull request with ID ${PR_ID}. The changes are between ${BASE_BRANCH} and pr-${PR_ID}-temp.

   Run this command to see the full diff:
   git diff ${BASE_BRANCH}...pr-${PR_ID}-temp

   Analyze ONLY the code introduced in those new commits (not the entire file).

   Provide your specialized review following your system prompt output format.
   ```

**Step 4: Collect and Synthesize Results**

8. Wait for all 5 agents to complete their reviews

9. Collect all agent outputs and synthesize into a final report

10. As Lead Reviewer, provide:

   **Summary Section:**
   - PR ID and base branch used
   - Number of commits reviewed
   - Number of files changed

   **Critical Blockers:**
   - List any critical issues that MUST be fixed before merge
   - Categorize by severity (Critical, High, Medium)
   - Reference specific files and line numbers

   **Final Verdict:**
   Choose one:
   - **Approve** - No significant issues, safe to merge
   - **Approve with suggestions** - Minor improvements recommended but not blocking
   - **Request changes** - Significant issues that should be addressed before merge

   **Average Rating:**
   - Calculate average of all 5 reviewers' ratings (1-10 scale)
   - Display as: "Average Rating: X.X/10"

   **Prioritized Action List:**
   - List top 5-10 action items for the developer
   - Prioritize by impact (Critical > High > Medium > Low)
   - Format: "- [Priority] Description (file:line)"

**Step 5: Cleanup Reminder**

11. Show this message at the end:
   ```
   You can later clean up with: git branch -D pr-${PR_ID}-temp
   ```

## Agent Output Format

Each agent will output structured markdown:

```markdown
## Review Findings

### Critical Issues
- [Issue description] (file:line)

### Medium/Low Issues
- [Issue description] (file:line)

### Positive Observations
- [What was done well]

## Code Snippets

### Issue: [Title]
**Location:** src/file.ts:42

**Before:**
```typescript
// original code
```

**After (suggested):**
```typescript
// fixed code
```

## Rating

**Score:** X/10

**Justification:** [One sentence explaining the score]
```

## Notes

- The command orchestrates the workflow but doesn't do code analysis itself
- All 5 agents run in parallel for efficiency
- The lead reviewer (you) makes independent judgment after considering all inputs
- The temp branch is left for the user to clean up manually

## Troubleshooting

**Error: "Couldn't find remote ref"**
- Check that the PR ID is correct
- Verify the PR exists in the repository
- Ensure you have network access

**Error: "pathspec 'develop' did not match"**
- The base branch may not exist
- Check available branches with `git branch -r`
- Try a different base branch (e.g., `main`, `master`)
