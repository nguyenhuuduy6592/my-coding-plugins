# Claude Plugins Marketplace

Personal collection of Claude Code plugins for enhanced productivity and workflow automation.

## Installation

### Add Marketplace

Run this command in Claude Code to add the marketplace:

```bash
/marketplace
```

Use the `Tab` key to switch to the `Marketplaces` tab
Select `+ Add Marketplace`
Enter: `YOUR_USERNAME/my-coding-plugins`. Replace `YOUR_USERNAME` with your GitHub username.

### Install Plugins

After adding the marketplace, install plugins using:

```bash
# Install code-review plugin
/plugin install code-review

# Install worktree-dev plugin
/plugin install worktree-dev
```

### Updating Plugins

When the marketplace is updated with new versions:

```bash
/plugin update code-review
/plugin update worktree-dev
```

Or reinstall to force refresh:

```bash
/plugin install code-review --force
/plugin install worktree-dev --force
```

Claude Code will automatically reload the updated command definitions after the update.

### Refreshing the Marketplace

When the marketplace itself is updated (new plugins added, versions changed):

```bash
/marketplace
```

Use the `Tab` key to switch to the `Marketplaces` tab, select your marketplace, and click **Refresh** to fetch the latest changes.

## Available Plugins

### code-review

**Version**: 0.4.0
**Category**: Code Quality

Comprehensive multi-dimensional pull request code review with 5 parallel specialized reviewers.

**Features**:
- Bug & Correctness analysis
- Security vulnerability scanning (OWASP Top 10)
- Performance issue detection (N+1, scalability)
- Code quality assessment (readability, duplication)
- Architecture & best practices review

**Usage**:
```bash
/code-review:review-pr 271
/code-review:review-pr 271 main
```

### worktree-dev

**Version**: 0.4.1
**Category**: Productivity

Streamlined git worktree workflow with integrated feature-dev guidance for parallel feature development.

**Features**:
- Auto-generated branch names from context
- Integrated feature-dev workflow
- Smart worktree detection with multi-worktree selection
- AI-assisted merge conflict resolution
- Automatic cleanup of worktrees and branches

**Usage**:
```bash
# Start a new feature
/worktree-dev
/worktree-dev feature-name

# Complete and merge
/worktree-complete
/worktree-complete feature-name
```

## Development

### Adding New Plugins

1. Create plugin directory: `mkdir plugin-name`
2. Add plugin manifest: `plugin-name/.claude-plugin/plugin.json`
3. Add commands, agents, or skills
4. Update `marketplace.json` with plugin entry
5. Commit and push to GitHub

### Plugin Structure

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json          # Required: Plugin manifest
├── commands/                 # Slash commands (.md files)
├── agents/                   # Subagent definitions (.md files)
├── skills/                   # Agent skills (subdirectories)
│   └── skill-name/
│       └── SKILL.md
├── hooks/
│   └── hooks.json
└── README.md
```

## License

MIT

## Contributing

This is a personal plugin collection. Feel free to fork and create your own marketplace!

---

**Official Documentation**:
- [Claude Code Plugins](https://github.com/anthropics/claude-plugins-official)
- [Plugin Development Guide](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/plugin-dev)
