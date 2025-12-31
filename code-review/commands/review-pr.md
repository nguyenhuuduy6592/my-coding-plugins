---
description: Multi-dimensional PR review with parallel specialized reviewers
argument-hint: [pr-number] [base-branch]
allowed-tools: Bash, Read, Task, AskUserQuestion
---

# review-pr - Pull Request Code Review

This command performs a comprehensive PR review using 5 parallel specialized reviewers.

## Workflow

### Phase 1: Gather PR Information

1. Determine PR_ID and BASE_BRANCH from arguments or user input:
   - If `$1` is provided, use it as PR_ID
   - If `$1` is not provided, ask the user for the PR ID (just the number, e.g., `271`)
   - Wait for user to provide the PR ID before proceeding

2. Determine BASE_BRANCH:
   - If `$2` is provided, use it as BASE_BRANCH
   - If `$2` is not provided, use `develop` as the default
   - Display: "Using base branch: ${BASE_BRANCH}"

### Phase 2: Fetch and Show PR Overview

3. Validate input and check temp branch status:
   ```bash
   # Validate PR_ID is numeric
   if ! [[ "${PR_ID}" =~ ^[0-9]+$ ]]; then
       echo "Error: PR_ID must be a number"
       exit 1
   fi

   # Validate BASE_BRANCH exists
   if ! git rev-parse --verify "${BASE_BRANCH}" >/dev/null 2>&1; then
       echo "Error: Branch '${BASE_BRANCH}' does not exist"
       echo "Available branches:"
       git branch -r
       exit 1
   fi

   # Fetch to FETCH_HEAD for comparison (doesn't require branch creation)
   if ! git fetch origin "pull/${PR_ID}/head" 2>/dev/null; then
       echo "Error: Failed to fetch PR ${PR_ID}"
       echo "Possible causes: PR doesn't exist, network issues, or remote not named 'origin'"
       exit 1
   fi

   # Check if temp branch exists
   if git show-ref --verify --quiet "refs/heads/pr-${PR_ID}-temp"; then
       # Get commit hashes
       LOCAL_COMMIT=$(git rev-parse "pr-${PR_ID}-temp")
       REMOTE_COMMIT=$(git rev-parse FETCH_HEAD)

       if [ "$(git branch --show-current)" = "pr-${PR_ID}-temp" ]; then
           # On the temp branch
           if [ "$LOCAL_COMMIT" = "$REMOTE_COMMIT" ]; then
               echo "TEMP_BRANCH_CURRENT_UP_TO_DATE"
           else
               echo "TEMP_BRANCH_CURRENT_STALE"
           fi
       else
           # Not on temp branch - safe to delete
           git branch -D "pr-${PR_ID}-temp"
           if ! git fetch origin "pull/${PR_ID}/head:pr-${PR_ID}-temp"; then
               echo "Error: Failed to create temp branch"
               exit 1
           fi
       fi
   else
       # Temp branch doesn't exist - create it
       if ! git fetch origin "pull/${PR_ID}/head:pr-${PR_ID}-temp"; then
           echo "Error: Failed to create temp branch"
           exit 1
       fi
   fi
   ```

4. If the bash output indicates the temp branch is current and up-to-date:
   - Display: "Local branch pr-${PR_ID}-temp is up-to-date with the remote PR. Using existing branch."
   - Skip to step 8 (list commits)

5. If the bash output indicates the temp branch is stale (current branch is behind remote), prompt the user using `AskUserQuestion`:
   ```json
   {
     "questions": [
       {
         "question": "You're on branch pr-${PR_ID}-temp which is behind the remote PR. How would you like to proceed?",
         "header": "Stale branch",
         "multiSelect": false,
         "options": [
           {
             "label": "Switch and update",
             "description": "Switch to ${BASE_BRANCH}, delete temp branch, fetch fresh content"
           },
           {
             "label": "Abort",
             "description": "Stop execution"
           }
         ]
       }
     ]
   }
   ```

6. If user chooses "Switch and update":
   ```bash
   if ! git checkout "${BASE_BRANCH}"; then
       echo "Error: Failed to switch to branch ${BASE_BRANCH}"
       exit 1
   fi
   git branch -D "pr-${PR_ID}-temp"
   if ! git fetch origin "pull/${PR_ID}/head:pr-${PR_ID}-temp"; then
       echo "Error: Failed to create temp branch"
       exit 1
   fi
   ```

7. Fetch the PR head (if not already fetched in step 3 or step 6):
   ```bash
   if ! git fetch origin "pull/${PR_ID}/head:pr-${PR_ID}-temp"; then
       echo "Error: Failed to fetch PR ${PR_ID}"
       echo "Possible causes: PR doesn't exist, network issues, or remote not named 'origin'"
       exit 1
   fi
   ```

8. List all commits unique to the PR:
   ```bash
   git log --oneline --graph "${BASE_BRANCH}...pr-${PR_ID}-temp"
   ```

9. Show diff stats summary:
   ```bash
   git diff --stat "${BASE_BRANCH}...pr-${PR_ID}-temp"
   ```

10. If no commits are found (PR already merged), show message and exit:
   - "This PR has no commits unique to ${BASE_BRANCH}. It may already be merged."
   - Exit gracefully

### Phase 3: Launch 5 Parallel Reviewers

11. Before launching agents, verify the PR has commits to review:
   ```bash
   COMMITS=$(git log --oneline "${BASE_BRANCH}...pr-${PR_ID}-temp" | wc -l)
   if [ "$COMMITS" -eq 0 ]; then
       echo "NO_COMMITS"
   fi
   ```

   If the output is `NO_COMMITS`:
   - Display: "This PR has no commits unique to ${BASE_BRANCH}. It may already be merged."
   - Exit gracefully

12. Launch all 5 reviewer agents in parallel using a single message with multiple Task tool calls:
   - `bug-correctness-reviewer` - Analyze for bugs and correctness issues
   - `security-reviewer` - Analyze for security vulnerabilities
   - `performance-reviewer` - Analyze for performance issues
   - `code-quality-reviewer` - Analyze for code quality issues
   - `architecture-reviewer` - Analyze for architecture and best practices

   Set timeout to 600000ms (10 minutes) for each agent.

   Each agent should receive a prompt with the actual values substituted (for example, if PR_ID=254 and BASE_BRANCH=develop):

   ```
   Review the pull request with ID 254. The changes are between develop and pr-254-temp.

   Run this command to see the full diff:
   git diff "develop...pr-254-temp"

   Analyze ONLY the code introduced in those new commits (not the entire file).

   Provide your specialized review following your system prompt output format.
   ```

   CRITICAL: You MUST substitute the actual PR_ID and BASE_BRANCH values in the prompt. Do NOT use placeholder variables like ${PR_ID} or ${BASE_BRANCH}.

### Phase 4: Collect and Synthesize Results

13. Wait for all 5 agents to complete their reviews

14. If any agent timed out, display: "Warning: [agent-name] timed out after 10 minutes. Results may be incomplete."

15. Collect all agent outputs and synthesize into a final report

16. As Lead Reviewer, provide:

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

### Phase 5: Cleanup Reminder

17. Show this message at the end:
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

### Git-Related Errors

**Error: "Couldn't find remote ref"**
- **Cause**: PR doesn't exist or is inaccessible
- **Diagnosis**: Run `gh pr view ${PR_ID}` to verify
- **Solutions**:
  - Verify PR ID is correct
  - Check you have repository access
  - Ensure PR is in the correct repository
  - Check network connection

**Error: "pathspec 'develop' did not match"**
- **Cause**: Base branch doesn't exist locally or remotely
- **Diagnosis**: Run `git branch -r | grep develop`
- **Solutions**:
  - List available branches: `git branch -r`
  - Fetch all branches: `git fetch --all`
  - Try alternative base branch (main, master)
  - Create the branch if it should exist

**Error: "fatal: cannot delete branch 'pr-XXX-temp' checked out at"**
- **Cause**: You're currently on the temp branch
- **Solutions**:
  - Switch to main branch first: `git checkout main`
  - Then delete: `git branch -D pr-XXX-temp`

### Agent-Related Errors

**Error: Agent fails to start or times out**
- **Cause**: Agent tool not available or configuration issue
- **Diagnosis**: Check agent is installed with `Skill` tool
- **Solutions**:
  - Verify agents are available
  - Check internet connection
  - Retry individual agent if one fails
  - Check Claude Code status

### Permission Errors

**Error: "Permission denied (publickey)"**
- **Cause**: SSH key not configured or invalid
- **Solutions**:
  - Check SSH config: `ssh -T git@github.com`
  - Verify key is loaded: `ssh-add -l`
  - Consider using HTTPS instead of SSH

### Network Issues

**Error: "Could not resolve host" or "Connection timed out"**
- **Solutions**:
  - Check internet connection
  - Verify VPN/proxy settings
  - Try again after a few moments
  - Check GitHub status: https://www.githubstatus.com

### Input Validation Errors

**Error: "PR_ID must be a number"**
- **Cause**: Non-numeric value provided for PR ID
- **Solutions**:
  - Provide only the numeric PR ID (e.g., "271" not "PR-271")
  - Find PR ID from URL: github.com/user/repo/pull/XXX

**Error: "Branch 'X' does not exist"**
- **Cause**: Specified base branch doesn't exist
- **Solutions**:
  - Run `git branch -r` to see available branches
  - Use a different base branch (main, master, develop)
