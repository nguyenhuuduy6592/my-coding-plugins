---
name: worktree-dev
description: Create a new git worktree and start the feature-dev workflow
argument-hint: "[branch-name]"
allowed-tools: ["Bash", "Skill", "AskUserQuestion", "Read"]
---

# worktree-dev

Create a new git worktree for parallel feature development and launch the feature-dev workflow.

## Execution

1. **If branch-name argument is provided**: Skip to step 3

2. **If no branch-name argument**:
   - IMMEDIATELY use `AskUserQuestion` to ask: "What feature are you working on?"
   - Validate: at least 3 characters, max 3 attempts. If invalid: "Please provide more details (at least 3 characters)."
   - Generate kebab-case branch name:
     - Convert to lowercase, replace spaces/special chars with hyphens
     - Remove consecutive hyphens, trim leading/trailing hyphens
     - Max 50 chars, truncate at word boundary if needed
     - Avoid git reserved words: HEAD, main, master, ORIG_HEAD, FETCH_HEAD, MERGE_HEAD
     - Must not conflict with existing branches (check via `git branch`)
   - Examples: "Add search feature" → `add-search-feature`, "Fix login bug" → `fix-login-bug`
   - If generation fails or name conflicts: Ask user for explicit branch name with prompt "Could not generate a valid branch name. Please provide one (lowercase, numbers, hyphens only, max 50 chars):"

3. **Create worktree**:
   ```bash
   mkdir -p .tree && git worktree add -b <branch-name> .tree/<branch-name>
   cd .tree/<branch-name>
   ```

4. **Start feature-dev**: Use `Skill` tool to invoke `feature-dev:feature-dev`

## When to Use

Use this command when you want to:
- Start working on a new feature in parallel with other work
- Keep the main branch clean while developing
- Get AI assistance with architecture and implementation planning

## Arguments

- `branch-name` (optional): Name for the new branch

## Error Handling

- Invalid input 3 times: Report error and stop
- `.tree/` creation fails: Report error and stop
- `git worktree add` fails: Report git error and stop
- Branch name generation fails: Ask user for explicit branch name (1 retry)
- feature-dev fails: Report error but worktree exists (user can manually run feature-dev)

## Notes

- **Location**: Run from main working directory (not within an existing worktree)
- **Work directory**: All file operations MUST happen in `.tree/<branch-name>/` after creation
- **Code isolation**: All file reads/edits must use absolute paths to the worktree
- **Cleanup**: Use `/worktree-complete` to merge back to main and clean up
