# worktree-dev Plugin

Streamlined git worktree workflow with integrated feature-dev guidance for parallel feature development.

## Overview

`worktree-dev` combines git worktree management with the feature-dev workflow, enabling you to work on multiple features in parallel while keeping your main branch clean.

## Features

- **Quick worktree creation** with auto-generated branch names
- **Integrated feature-dev workflow** launches automatically
- **Smart worktree detection** for easy completion
- **AI-assisted merge conflict resolution** preserves changes from both branches
- **Automatic cleanup** of worktrees and branches

## Installation

This plugin is installed globally and available in all Claude Code sessions.

### Updating the Plugin

When changes are made to the plugin (e.g., command improvements, bug fixes):

```bash
/plugin update worktree-dev
```

Or reinstall to force refresh:

```bash
/plugin install worktree-dev --force
```

Claude Code will automatically reload the updated command definitions.

## Prerequisites

- Git installed and configured
- Feature-dev plugin installed (for `/worktree-dev` integration)

## Usage

### Start a New Feature

```bash
/worktree-dev
```

This command:
1. Generates a branch name from your context
2. Creates a new git worktree at `.tree/<branch-name>/`
3. Navigates to the worktree
4. Launches the feature-dev workflow

Or specify a branch name:

```bash
/worktree-dev feature-add-search
```

### Complete a Feature

```bash
/worktree-complete
```

This command:
1. Auto-detects your worktree (or shows selection list)
2. Kills any running Node processes in the worktree
3. Merges your branch to main
4. Handles conflicts with AI assistance if needed
5. Cleans up the worktree and branch

Or specify a worktree:

```bash
/worktree-complete feature-add-search
```

## Workflow

```
main branch
    │
    ├─ /worktree-dev "add search"
    │   │
    │   ▼
    │   .tree/add-search/  ← Isolated development
    │   └─ feature-dev guides implementation
    │
    │   [development complete]
    │
    ├─ /worktree-complete
    │   │
    │   ▼
    │   main branch ← Changes merged
    │   worktree cleaned up
```

## Merge Conflict Handling

When merge conflicts are detected:

1. **Abort merge to main** (keeps main clean)
2. **Merge main into worktree** (conflicts now in worktree)
3. **AI attempts resolution**:
   - Preserves changes from both branches
   - Combines imports, code blocks, and config
   - Intelligently merges function modifications
4. **If AI fails**: You resolve manually
5. **After resolution**: Choose to restart merge flow

## Branch Name Generation

Branch names are auto-generated from your context:

| Request | Branch Name |
|---------|-------------|
| "Add search feature" | `add-search-feature` |
| "Fix login bug" | `fix-login-bug` |
| "Update API integration" | `update-api-integration` |

**Rules**:
- Kebab-case (lowercase, hyphens only)
- Maximum 50 characters
- Git-safe characters only
- Avoids reserved words

## Commands

### `/worktree-dev [branch-name]`

Create a new git worktree and start feature-dev workflow.

**Arguments**:
- `branch-name` (optional): Branch name. Auto-generated from context if not provided.

**Requirements**:
- Run from main directory (not within existing worktree)

### `/worktree-complete [folder-name]`

Merge worktree back to main and clean up.

**Arguments**:
- `folder-name` (optional): Worktree folder name. Auto-detected if not provided.

**Requirements**:
- Commits should be made in worktree before completing
- Node processes in worktree will be terminated

## Directory Structure

```
project/
├── .tree/
│   ├── feature-add-search/    ← Worktree 1
│   ├── fix-login-bug/         ← Worktree 2
│   └── refactor-api/          ← Worktree 3
└── [main project files]
```

## See Also

- [Git Worktrees Documentation](https://git-scm.com/docs/git-worktree)
- [feature-dev Plugin](https://github.com/anthropics/claude-code/tree/main/plugins/feature-dev)

## Version

0.2.0
