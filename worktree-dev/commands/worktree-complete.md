---
name: worktree-complete
description: Merge a worktree back to main and clean up, with AI-assisted conflict resolution
argument-hint: "[folder-name]"
allowed-tools: ["Bash", "Read", "Edit", "AskUserQuestion", "Glob"]
---

# worktree-complete

Merge a git worktree back to the main branch and clean up, with intelligent conflict resolution support.

## Overview

This command handles the complete workflow of:
1. Detecting the target worktree
2. Killing any running processes in the worktree
3. Merging the worktree branch to main
4. AI-assisted merge conflict resolution (if needed)
5. Cleaning up the worktree and branch

## When This Command Is Used

Use this command when you:
- Have completed a feature in a worktree
- Want to merge your changes back to main
- Need help resolving merge conflicts

## How This Command Works

### Step 1: Detect Worktree

Determine which worktree to complete:

**Auto-detection from current directory**:
1. Check if current working directory path contains `.tree/`
2. Extract worktree name from path (e.g., from `D:\Code\project\.tree\feature-branch`, extract `feature-branch`)
3. Verify worktree exists using `git worktree list`

**If not in a worktree**:
1. Run `git worktree list` to get all worktrees
2. Parse output to find worktree paths
3. If multiple worktrees exist in `.tree/`, present a numbered list to user:
   ```
   Multiple worktrees found. Select one to complete:

   1. feature-add-search (.tree/feature-add-search)
   2. fix-login-bug (.tree/fix-login-bug)
   3. refactor-api (.tree/refactor-api)

   Enter the number of the worktree to complete:
   ```
4. Use AskUserQuestion to get user selection
5. If only one worktree exists, use it automatically

**If folder-name argument provided**: Use it directly (skip detection)

### Step 2: Kill Processes

Terminate any Node.js processes running in the worktree:

```bash
powershell -NoProfile -Command 'Get-CimInstance Win32_Process | Where-Object { $_.Name -eq \"node.exe\" -and $_.CommandLine -like \"*D:\\Code\\Personal\\my-coding-plugins\\.tree\\test-ps-2*\" } | ForEach-Object { Stop-Process -Id $_.ProcessId -Force }'
```

Where `<worktree-full-path>` is the absolute path to `.tree/<folder-name>`.

Continue even if no processes are found (not an error).

### Step 3: Switch to Main

Switch to the main branch:

```bash
git checkout main
```

If this fails, report the error and stop.

### Step 4: Attempt Merge

Attempt to merge the worktree branch to main:

```bash
git merge <folder-name>
```

**If merge succeeds**: Proceed to Step 7 (Cleanup)

**If merge has conflicts**: Proceed to Step 5 (Conflict Resolution)

### Step 5: Conflict Resolution - Abort and Reverse

When conflicts are detected:

1. Abort the merge to main:
   ```bash
   git merge --abort
   ```

2. Switch back to the worktree branch:
   ```bash
   git checkout <folder-name>
   ```

3. Merge main into the worktree branch:
   ```bash
   git merge main
   ```

### Step 6: AI-Assisted Conflict Resolution

Attempt to resolve conflicts using AI:

**Identify conflicted files**:
```bash
git diff --name-only --diff-filter=U
```

**For each conflicted file**:
1. Read the file with conflict markers
2. Analyze both sides of the conflict
3. Use Edit tool to resolve conflicts, preserving changes from both branches when possible

**Resolution Strategy**:
- **Code blocks**: If both branches added different code sections, include both
- **Import statements**: Combine imports from both branches
- **Function definitions**: If both modified the same function, intelligently merge the changes
- **Configuration**: Merge configuration options, preferring more specific settings
- **Comments/documentation**: Preserve documentation from both sides

**After resolving all conflicts in a file**:
```bash
git add <resolved-file>
```

**Check if conflicts remain**:
```bash
git diff --name-only --diff-filter=U
```

- If no conflicts remain: Complete the merge with `git commit`
- If conflicts remain: Ask user to resolve manually

**If AI cannot resolve conflicts**:
1. Inform user: "I encountered conflicts I couldn't automatically resolve. Please resolve them manually."
2. Show list of remaining conflicted files
3. Instructions: "After resolving conflicts, run `git add <files>` and `git commit` to complete the merge, then run `/worktree-complete <folder-name>` again."

**After successful resolution**:
Use AskUserQuestion to ask user:
- Question: "Conflicts resolved. Restart complete flow to merge to main?"
- Options:
  - Yes, restart merge flow
  - No, I'll handle it manually

If user chooses Yes: Return to Step 3 (Switch to Main)

If user chooses No: Inform user worktree remains and they can complete manually later.

### Step 7: Cleanup

Once merge to main is successful:

1. Remove the worktree:
   ```bash
   git worktree remove .tree/<folder-name>
   ```

2. Delete the branch:
   ```bash
   git branch -d <folder-name> || git branch -D <folder-name>
   ```

3. Report success to user.

## Arguments

- `folder-name` (optional): Name of the worktree folder in `.tree/`. If not provided, auto-detects from current directory or shows selection list.

## Usage Examples

```bash
/worktree-complete
```
Auto-detects worktree from current directory, or shows selection list if multiple exist.

```bash
/worktree-complete feature-add-search
```
Directly completes the `feature-add-search` worktree.

## Important Notes

- **Location**: Can be run from any directory in the repository
- **Safety**: Killing processes ensures ports and files are released before merging
- **Conflict handling**: Conflicts are resolved in the worktree, keeping main clean
- **Data preservation**: AI attempts to preserve changes from both branches during conflict resolution

## Error Handling

- If `git checkout main` fails: Report error and stop
- If `git worktree remove` fails: Force remove with `git worktree remove --force`
- If user cancels conflict resolution: Worktree remains, user can retry later

## Merge Conflict Behavior

1. Abort merge to main (keeps main clean)
2. Merge main into worktree branch
3. AI attempts to resolve conflicts automatically
4. If AI fails, user resolves manually
5. After resolution, user chooses whether to restart complete flow

## See Also

- `/worktree-dev` - Create a new worktree and start feature-dev
- [Git Worktrees Documentation](https://git-scm.com/docs/git-worktree)
- feature-dev plugin documentation for workflow details
