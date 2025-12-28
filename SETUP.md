# GitHub Setup Guide

After pushing this repository to GitHub, follow these steps to add it as a Claude Code marketplace.

## Step 1: Create GitHub Repository

```bash
# From the my-coding-plugins directory
git remote add origin https://github.com/YOUR_USERNAME/my-coding-plugins.git
git branch -M main
git push -u origin main
```

## Step 2: Add Marketplace to Claude Code

Edit `C:\Users\YOUR_USER\.claude\plugins\known_marketplaces.json`:

```json
"my-coding-plugins": {
  "source": {
    "source": "github",
    "repo": "YOUR_USERNAME/my-coding-plugins"
  },
  "installLocation": "C:\\Users\\YOUR_USER\\.claude\\plugins\\marketplaces\\my-coding-plugins",
  "lastUpdated": "2025-12-28T00:00:00.000Z"
}
```

**Important**: Replace:
- `YOUR_USERNAME` with your actual GitHub username (both places)
- Update `lastUpdated` to current timestamp

## Step 3: Restart Claude Code

Claude Code will automatically discover and load the marketplace.

## Step 4: Install Plugins

```bash
/plugin install worktree-dev
```

## File Structure After Installation

```
C:\Users\YOUR_USER\.claude\plugins\
├── marketplaces/
│   └── my-coding-plugins/           # Cloned from your GitHub repo
│       ├── .claude-plugin/
│       │   └── marketplace.json
│       └── worktree-dev/
└── known_marketplaces.json        # Contains your marketplace entry
```

## Updating Plugins

To add new plugins or update existing ones:

1. Edit files in `D:\Code\Personal\my-coding-plugins\`
2. Commit and push to GitHub
3. Claude Code will pull updates automatically

## Testing Locally

To test marketplace changes before pushing:

1. Copy marketplace files to local marketplace:
   ```bash
   cp -r D:\Code\Personal\my-coding-plugins C:\Users\YOUR_USER\.claude\plugins\marketplaces\
   ```

2. Update `known_marketplaces.json` to use `directory` source:
   ```json
   "my-coding-plugins": {
     "source": {
       "source": "directory",
       "path": "D:\\Code\\Personal\\my-coding-plugins"
     },
     "installLocation": "C:\\Users\\YOUR_USER\\.claude\\plugins\\marketplaces\\my-coding-plugins",
     "lastUpdated": "2025-12-28T00:00:00.000Z"
   }
   ```

3. Restart Claude Code
