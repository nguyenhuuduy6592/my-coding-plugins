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

4. If the bash output indicates the temp branch is current and up-to-date, skip to step 7 (list commits).

5. If the bash output indicates the temp branch is stale, prompt the user using `AskUserQuestion`:
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

7. List all commits unique to the PR:
   ```bash
   git log --oneline --graph "${BASE_BRANCH}...pr-${PR_ID}-temp"
   ```

8. Show diff stats summary:
   ```bash
   git diff --stat "${BASE_BRANCH}...pr-${PR_ID}-temp"
   ```

### Phase 3: Launch 5 Parallel Reviewers

9. Verify the PR has commits to review:
   ```bash
   COMMITS=$(git log --oneline "${BASE_BRANCH}...pr-${PR_ID}-temp" | wc -l)
   if [ "$COMMITS" -eq 0 ]; then
       echo "This PR has no commits unique to ${BASE_BRANCH}. It may already be merged."
       exit 0
   fi
   ```

10. Check diff size:
   ```bash
   DIFF_SIZE=$(git diff "${BASE_BRANCH}...pr-${PR_ID}-temp" | wc -l)
   if [ "$DIFF_SIZE" -gt 1000 ]; then
       echo "Large diff detected (${DIFF_SIZE} lines). Agents will focus on critical/high-severity issues only."
   fi
   ```

11. Construct agent prompt (substitute {PR_ID} and {BASE_BRANCH} with actual values):
   ```
   CRITICAL: First step is to IMMEDIATELY run this git command:
   git diff "{BASE_BRANCH}...pr-{PR_ID}-temp"

   Do NOT explore any files before running this command. This git diff shows ALL the changes for PR #{PR_ID}.

   After getting the diff, analyze ONLY the code introduced in those new commits (not the entire file).

   Provide your specialized review following your system prompt output format.
   ```

   If DIFF_SIZE > 1000, append: "IMPORTANT: This is a LARGE diff. Focus only on CRITICAL and HIGH severity issues. Limit your output to top 5-7 most important findings."

12. Launch all 5 reviewer agents in parallel using a SINGLE message with FIVE Task tool calls:
   - `bug-correctness-reviewer`
   - `security-reviewer`
   - `performance-reviewer`
   - `code-quality-reviewer`
   - `architecture-reviewer`

   CRITICAL - Use this EXACT Task tool call format for each agent:
   ```json
   {
     "subagent_type": "code-review:<agent-name>",
     "prompt": "<prompt from step 11>",
     "run_in_background": true,
     "timeout": 600000
   }
   ```

13. Display the captured task IDs:
   ```
   Reviewer tasks launched:
   - bug-correctness-reviewer: <task_id_1>
   - security-reviewer: <task_id_2>
   - performance-reviewer: <task_id_3>
   - code-quality-reviewer: <task_id_4>
   - architecture-reviewer: <task_id_5>
   ```

14. Collect agent outputs using TaskOutput with the captured task IDs from step 13:
   ```json
   {
     "task_id": "<task_id>",
     "block": true
   }
   ```
   Send all 5 TaskOutput calls in ONE message. Handle errors gracefully.

15. Display: "Collecting and synthesizing reviews from all agents..."

16. Synthesize all agent outputs into a final report:
   - **Summary**: PR ID, base branch, commits reviewed, files changed
   - **Critical Blockers**: Issues that MUST be fixed, with severity (Critical/High/Medium) and file:line references
   - **Final Verdict**: Approve / Approve with suggestions / Request changes
   - **Average Rating**: Calculate average of all 5 reviewers' ratings (1-10)
   - **Prioritized Action List**: Top 5-10 action items, prioritized by impact

### Phase 4: Cleanup

17. Show this message at the end:
   ```
   You can later clean up with: git branch -D pr-${PR_ID}-temp
   ```

## Agent Output Format

Each agent outputs structured markdown with findings, code snippets, and ratings (see individual agent files for details).

### Input Validation

**"PR_ID must be a number"** - Provide only numeric ID (e.g., "271" not "PR-271").
