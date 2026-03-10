# code-review

A comprehensive multi-dimensional pull request code review plugin for Claude Code.

## Overview

`code-review` performs thorough PR reviews using 5 specialized parallel reviewers, each focusing on a different aspect of code quality:

- **Bug & Correctness** - Logical errors, edge cases, nulls, wrong assumptions
- **Security** - Vulnerabilities, injections, auth bypass, secrets, OWASP
- **Performance** - Slow queries, N+1, redundant work, scalability risks
- **Code Quality** - Readability, naming, duplication, structure, comments, standards
- **Architecture & Best Practices** - Design, separation of concerns, testability, error handling, logging

## Installation

Place this plugin in your Claude Code plugins directory:

```bash
# Or manually
cp -r code-review ~/.claude/plugins/
```

## Usage

### Review a Pull Request

```bash
# By PR number (auto-discovers which repo)
/code-review:review-pr 3

# By branch name
/code-review:review-pr TD-17

# Interactive mode (prompts for PR number or branch)
/code-review:review-pr
```

The command accepts:
1. **PR number or branch name** - Searches all Talgent Gitea repos for matching open PRs

The plugin will:
1. Query all Talgent org repos on Gitea for open PRs
2. Match by PR number or branch name (substring match)
3. If multiple matches, present a numbered list to choose from
4. Fetch the PR diff via Gitea API (no local git fetch needed)
5. Launch 5 parallel specialized reviewers
6. Provide a final verdict with:
   - Critical blockers (if any)
   - Verdict: Approve / Approve with suggestions / Request changes
   - Prioritized action list

## Requirements

- Gitea credentials in `~/.git-credentials` (for gitea.talgent.me)
- Gitea helper script at `D:/Code/Talgent/.claude/skills/gitea/Invoke-Gitea.ps1`
- PowerShell (for Gitea API calls)

## Reviewers

The plugin uses 5 specialized parallel reviewers, each with distinct focus areas:

| Reviewer | Color | Focus |
|----------|-------|-------|
| **bug-correctness-reviewer** | Red | Logical errors, edge cases, null handling, race conditions |
| **security-reviewer** | Yellow | OWASP Top 10, injection attacks, secrets, authentication |
| **performance-reviewer** | Magenta | N+1 queries, algorithmic complexity, scalability |
| **code-quality-reviewer** | Cyan | Readability, naming conventions, duplication, structure |
| **architecture-reviewer** | Blue | Design patterns, SOLID principles, testability, logging |

Each reviewer provides:
- Structured findings with severity levels
- Code snippets showing issues and fixes
- 1-10 rating with justification

## Output Format

The final review includes:

```markdown
## Summary
- PR ID, repo name, title, branch info

## Critical Blockers
Issues that MUST be fixed before merge

## Final Verdict
- Approve / Approve with suggestions / Request changes
- Average Rating: X.X/10 (across all 5 reviewers)

## Prioritized Action List
Top 5-10 action items for the developer
```

## Version History

### 0.6.1 (2026-03-11)
- **Critical Fix:** Replace Task/TaskOutput with Agent tool for launching reviewers
  - Task/TaskOutput is a task management system, not agent launching
  - Background agents deliver results via `<task-notification>` messages
  - This fixes "No task found with ID" errors that prevented collecting review results
- **Fix:** Add large diff cap (5000+ lines) with user confirmation
- **Fix:** Add error handling for non-zero exit, agent timeout, partial failure
- **Fix:** Replace nested PowerShell invocation with `&` operator

### 0.6.0 (2026-03-11)
- **Major Feature:** Auto-discover PRs across all Talgent Gitea repos
  - No longer need to be in the correct repo directory
  - Search by PR number or branch name (substring match)
  - If multiple matches found, presents numbered list to choose from
- **Major Improvement:** Fetch diff via Gitea API instead of git fetch
  - No local temp branches created or cleaned up
  - Works from any directory regardless of git context
  - Uses Gitea diff_url with Basic auth
- **Removed:** git fetch, temp branch creation, cleanup phase
- **Removed:** BASE_BRANCH parameter (no longer needed - diff comes from Gitea)

### 0.5.0 (2026-01-15)
- **Major Improvement:** Simplified branch handling - no more branch switching required
  - Uses `git fetch --force` to cleanly create/update temp branches
  - Enables truly autonomous PR reviews across multiple concurrent PRs

### 0.4.2 (2025-12-31)
- **Fix:** Added missing `description` parameter to Task tool calls

### 0.4.1 (2025-12-31)
- **Critical Fix:** Agents no longer analyze wrong files - git diff content passed directly

### 0.4.0 (2025-12-31)
- **Fix:** Critical issue where agents analyzed wrong project instead of PR content
- **Fix:** Output truncation for large diffs
- **Improvement:** ~50% token reduction in review-pr command

### 0.3.0 (2025-12-31)
- **Fix:** Critical issue where agents returned 0 tool uses

### 0.2.1 (2025-12-31)
- **Fix:** Agents now properly execute git diff commands

### 0.2.0 (2025-12-31)
- **Feature:** Add command argument support

### 0.1.0 (2025-12-31)
- Initial release

## Version

0.6.1
