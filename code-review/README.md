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
/code-review:review-pr
```

You will be prompted for:
1. **PR ID** – The pull request number (e.g., `271`)
2. **Base branch** – The branch to compare against (default: `develop`)

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
- `origin` remote configured
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

0.1.2
