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

If no `branch-name` argument is provided, generate a branch name from the user's context:

**Generation Rules**:
- Convert the user's request to kebab-case
- Use only lowercase letters, numbers, and hyphens
- Remove special characters, spaces, underscores
- Replace multiple consecutive hyphens with single hyphen
- Maximum length: 50 characters
- Avoid git reserved words (e.g., "HEAD", "main", "master")

**Examples**:
- "Add search feature" → `add-search-feature`
- "Fix login bug" → `fix-login-bug`
- "Update API integration" → `update-api-integration`

**If generation fails**: Ask user to provide a branch name explicitly.

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

Once the worktree is created and you're in the worktree directory, use the Skill tool to start the feature-dev workflow:

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
- **Cleanup**: Use `/worktree-complete` to merge the worktree back to main and clean up
- **Feature-dev**: The feature-dev skill provides comprehensive guidance for implementing features

## Error Handling

- If `.tree/` directory creation fails: Report error and stop
- If git worktree creation fails: Report git error message and stop
- If feature-dev skill fails: Report error but worktree is still created (user can manually run feature-dev)

## See Also

- `/worktree-complete` - Merge worktree back to main and clean up
- [Git Worktrees Documentation](https://git-scm.com/docs/git-worktree)
- feature-dev plugin documentation for workflow details
