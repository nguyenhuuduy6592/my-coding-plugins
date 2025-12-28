---
name: worktree-dev
description: Create a new git worktree and start the feature-dev workflow
argument-hint: "[branch-name]"
allowed-tools: ["Bash", "Skill", "AskUserQuestion", "Read"]
---

# worktree-dev

Create a new git worktree for parallel feature development and automatically start the feature-dev workflow.

## Overview

This command streamlines the workflow of:
1. Creating a new git worktree (isolated development environment)
2. Launching the feature-dev workflow for guided feature implementation

## When This Command Is Used

Use this command when you want to:
- Start working on a new feature in parallel with other work
- Keep the main branch clean while developing
- Get AI assistance with architecture and implementation planning

## How This Command Works

### Step 1: Generate Branch Name

If no `branch-name` argument is provided:

1. **AskUserQuestion**: Prompt "What feature are you working on?"

2. **Validate response**:
   - If empty, whitespace-only, or < 3 characters: Ask again with "Please provide more details (at least 3 characters)."
   - Maximum 3 attempts for valid input

3. **Generate branch name** from the user's validated response using these rules:
   - Convert to kebab-case (lowercase letters, numbers, hyphens only)
   - Remove special characters, spaces, underscores
   - Replace multiple consecutive hyphens with single hyphen
   - Cannot start or end with hyphen
   - Maximum length: 50 characters (truncate at hyphen to preserve word boundary)
   - Avoid git reserved words: "HEAD", "main", "master", "ORIG_HEAD", "FETCH_HEAD", "MERGE_HEAD"
   - Must not conflict with existing branch names

**Examples**:
- "Add search feature" → `add-search-feature`
- "Fix login bug" → `fix-login-bug`
- "Update API integration with OAuth2" → `update-api-integration` (truncated at word boundary)

4. **Handle generation failure**:
   - If generated name is empty or invalid: Use AskUserQuestion with "Could not generate a valid branch name. Please provide one (lowercase, numbers, hyphens only, max 50 chars):"
   - If still invalid after 1 retry: Report error and stop

### Step 2: Create Worktree

Create the git worktree with the following commands in order:

1. Create `.tree` directory if it doesn't exist:
   ```bash
   mkdir -p .tree
   ```

2. Create the git worktree with new branch:
   ```bash
   git worktree add -b <branch-name> .tree/<branch-name>
   ```

3. Navigate to the worktree directory:
   ```bash
   cd .tree/<branch-name>
   ```

If any step fails, report the error to the user and stop.

### Step 3: Start Feature-Dev Workflow

**CRITICAL: Work in the worktree directory**

After creating the worktree, you MUST perform all subsequent work in the `.tree/<branch-name>/` directory to ensure proper code isolation:
- All file reads must use absolute paths to the worktree
- All file edits must target files in the worktree
- All bash commands must be run from the worktree directory

**Verify you are in the correct directory** before proceeding.

Once verified, use the Skill tool to start the feature-dev workflow:

```
Use the Skill tool to invoke feature-dev:feature-dev
```

This will launch the guided feature development workflow with codebase understanding and architecture focus.

## Arguments

- `branch-name` (optional): Name for the new branch. If not provided, generates from context.

## Usage Examples

```bash
/worktree-dev
```
Generates branch name from context, creates worktree, starts feature-dev.

```bash
/worktree-dev add-user-auth
```
Uses "add-user-auth" as branch name, creates worktree, starts feature-dev.

## Important Notes

- **Location**: This command should be run from the main working directory (not from within an existing worktree)
- **Directory**: Worktrees are created in `.tree/<branch-name>/`
- **Code isolation**: All file edits MUST be made in the worktree directory, not the main branch
- **Cleanup**: Use `/worktree-complete` to merge the worktree back to main and clean up
- **Feature-dev**: The feature-dev skill provides comprehensive guidance for implementing features

## Error Handling

- If user provides invalid input 3 times: Report error and stop
- If `.tree/` directory creation fails: Report error and stop
- If git worktree creation fails: Report git error message and stop
- If branch name generation fails: Ask user for explicit branch name (1 retry)
- If feature-dev skill fails: Report error but worktree is still created (user can manually run feature-dev)

## See Also

- `/worktree-complete` - Merge worktree back to main and clean up
- [Git Worktrees Documentation](https://git-scm.com/docs/git-worktree)
- feature-dev plugin documentation for workflow details
