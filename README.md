# Claude Plugins

Personal collection of Claude Code plugins.

## Plugins

### worktree-dev

Streamlined git worktree workflow with integrated feature-dev guidance for parallel feature development.

**Version**: 0.1.0

**Features**:
- Auto-generated branch names from context
- Integrated feature-dev workflow
- Smart worktree detection
- AI-assisted merge conflict resolution
- Automatic cleanup

**Usage**:
```bash
# Start a new feature
/worktree-dev
/worktree-dev feature-name

# Complete and merge
/worktree-complete
/worktree-complete feature-name
```

**Location**: `worktree-dev/`

## Installation

Each plugin is published to the official Claude marketplace. To install:

```bash
/plugin install worktree-dev
```

## Development

Plugins are developed using the [plugin-dev](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/plugin-dev) toolkit.

## License

MIT
