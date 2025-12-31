# code-review

A comprehensive multi-dimensional pull request code review plugin for Claude Code.

## Overview

`code-review` performs thorough PR reviews using 5 specialized parallel reviewers, each focusing on a different aspect of code quality:

- **Bug & Correctness** – Logical errors, edge cases, nulls, wrong assumptions
- **Security** – Vulnerabilities, injections, auth bypass, secrets, OWASP
- **Performance** – Slow queries, N+1, redundant work, scalability risks
- **Code Quality** – Readability, naming, duplication, structure, comments, standards
- **Architecture & Best Practices** – Design, separation of concerns, testability, error handling, logging

## Installation

Place this plugin in your Claude Code plugins directory:

```bash
# Via marketplace (coming soon)
claude plugin install code-review

# Or manually
cp -r code-review ~/.claude/plugins/
```

## Usage

### Review a Pull Request

```bash
# Quick review with default base branch (develop)
/code-review:review-pr 271

# Specify both PR number and base branch
/code-review:review-pr 271 main

# Interactive mode (prompts for PR number)
/code-review:review-pr
```

The command accepts:
1. **PR ID** – The pull request number (e.g., `271`)
2. **Base branch** (optional) – The branch to compare against (default: `develop`)

The plugin will:
1. Fetch the PR head safely: `git fetch origin pull/${PR_ID}/head:pr-${PR_ID}-temp`
2. List all commits on the PR: `git log --oneline --graph develop...pr-${PR_ID}-temp`
3. Show diff stats: `git diff --stat develop...pr-${PR_ID}-temp`
4. Launch 5 parallel specialized reviewers
5. Provide a final verdict with:
   - Critical blockers (if any)
   - Verdict: Approve / Approve with suggestions / Request changes
   - Prioritized action list

### Cleanup

After review, clean up the temporary branch:

```bash
git branch -D pr-${PR_ID}-temp
```

## Requirements

- Git repository
- Remote configured (`origin` or any git remote)
- Network access to fetch PR refs

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
- PR ID and base branch
- Number of commits and files changed

## Critical Blockers
Issues that MUST be fixed before merge

## Final Verdict
- Approve / Approve with suggestions / Request changes
- Average Rating: X.X/10 (across all 5 reviewers)

## Prioritized Action List
Top 5-10 action items for the developer
```

## Roadmap

- [ ] `review-branch` – Review feature branches against base branches

## Version History

### 0.4.1 (2025-12-31)
- **Critical Fix:** Agents no longer analyze wrong files - git diff content is now passed directly to agents
  - The parent command now captures git diff output before launching agents
  - Agents receive the complete diff content in their prompt instead of running git diff themselves
  - This fixes the issue where background agents would analyze files from a different codebase
  - All 5 reviewers updated to use provided diff content instead of running git commands

### 0.4.0 (2025-12-31)
- **Fix:** Critical issue where agents analyzed wrong project instead of PR content
  - Added explicit instruction to run git diff FIRST before any file exploration
  - Agents no longer wander into current working directory files
- **Fix:** Output truncation for security and code-quality reviewers on large diffs
  - Added output constraints (max 5-7 findings, 2-3 sentences each)
  - Large diff detection (>1000 lines) triggers focused analysis mode
- **Improvement:** ~50% token reduction in review-pr command
  - Simplified Task/TaskOutput examples (5 repetitive blocks → 1 template)
  - Compressed troubleshooting section
  - Streamlined agent output format reference
- **Improvement:** Works with any git remote (GitHub, Gitea, GitLab, etc.)
  - Removed GitHub-specific references from documentation
- **Docs:** Added troubleshooting for common agent issues

### 0.3.0 (2025-12-31)
- **Fix:** Critical issue where agents returned 0 tool uses and TaskOutput retrieval failed with "No task found with ID"
- **Fix:** Added explicit `run_in_background: true` parameter to all Task tool calls for proper task ID generation
- **Fix:** Added explicit TaskOutput syntax with `block: true` for proper result collection
- **Improvement:** Added task ID capture and display format for better debugging
- **Improvement:** Added agent launch verification and error handling steps
- **Improvement:** Added prompt template with clear variable substitution instructions
- **Documentation:** Added new troubleshooting sections for "No task found with ID" and "Agent returns 0 tool uses" errors

### 0.2.1 (2025-12-31)
- **Fix:** Remove timestamp-related bash echo commands that were causing unnecessary user permission prompts
- **Fix:** Agents now properly execute git diff commands instead of hallucinating reviews (fixed variable placeholder substitution issue)
- **Fix:** All reviewer agents updated to receive actual diff command values instead of `${PR_ID}`/`${BASE_BRANCH}` placeholders

### 0.2.0 (2025-12-31)
- **Feature:** Add command argument support for faster usage (`/code-review:review-pr 271 main`)
- **Feature:** Auto-use `develop` as default base branch without prompting
- **Improvement:** Add proper frontmatter fields (`allowed-tools`, `argument-hint`)
- **Improvement:** Add validation to prevent launching agents on empty/merged PRs
- **Refactor:** Rewrite documentation-style sections as direct Claude instructions
- **UX:** Display "Using base branch: X" message for clarity

### 0.1.2 (2025-12-31)
- Fix: Use three-dot diff (`...`) instead of two-dot diff (`..`) to show only PR-specific changes
- This prevents showing unrelated "removed" code from other merged branches

### 0.1.1 (2025-12-31)
- Fix: Plugin author nane

### 0.1.0 (2025-12-31)
- Initial release
- 5 parallel specialized reviewers (Bug, Security, Performance, Code Quality, Architecture)
- Interactive PR ID and base branch prompts
- Git fetch, commit log, and diff stats display
- Final verdict with average rating and prioritized action list
- Cleanup command reminder

## Version

0.4.1
