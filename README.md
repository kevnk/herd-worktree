# Herd Worktree

A Claude Code skill that automates setting up git worktrees for any project served by Laravel Herd on macOS.

## What It Does

When you need to work on a feature branch in isolation, this skill:

1. **Creates a git worktree** in `.worktrees/<branch-name>`
2. **Links with Laravel Herd** as a subdomain: `http://<branch>.<parent>.test`
3. **Configures environment** - copies `.env` and updates URLs if requested
4. **Installs dependencies** - detects and runs appropriate package managers

It also handles cleanup and PR creation when you're done.

## URL Pattern

Worktrees are served as **subdomains** of the parent project:

| Parent Project | Branch | Worktree URL |
|----------------|--------|--------------|
| `my-app.test` | `feature/login` | `http://feature-login.my-app.test` |
| `api.test` | `bugfix/auth` | `http://bugfix-auth.api.test` |

This keeps worktrees visually grouped with their parent project.

## Prerequisites

- **Laravel Herd** installed and running on macOS
- **Git** for version control

## Supported Package Managers

The skill auto-detects and can install dependencies for:

- **Composer** (PHP)
- **npm / yarn / pnpm** (Node.js)
- **Cargo** (Rust)
- **pip** (Python)
- **go mod** (Go)

## Installation

### Via GitHub

```bash
/plugin install kevnk/herd-worktree
```

### Manual Installation

1. Clone this repository:
   ```bash
   git clone https://github.com/kevnk/herd-worktree.git ~/.claude/plugins/herd-worktree
   ```

2. Add to your Claude Code configuration:
   ```bash
   claude plugins add ~/.claude/plugins/herd-worktree
   ```

## Usage

The skill is automatically invoked when you mention worktrees with Herd-served projects. Example prompts:

- "Set up a worktree for feature-login"
- "Create an isolated workspace for this branch"
- "I need to work on a feature branch in isolation"

### Manual Invocation

```
/herd-worktree
```

## What Gets Created

```
your-project/
├── .worktrees/
│   └── feature-branch/      # Your isolated worktree
│       ├── .env             # Copied and updated (if applicable)
│       ├── vendor/          # Fresh dependency install
│       └── node_modules/    # Fresh dependency install
```

Plus a Herd site at `http://feature-branch.your-project.test`

## Common Issues

### Site not loading
- Verify Herd link exists: `herd links | grep <branch>`
- Ensure Herd is running
- Check URL uses subdomain pattern: `http://branch.parent.test`

### Wrong domain in config
- Update `.env` URLs to use the worktree subdomain
- Run any framework-specific cache clear commands

### Mixed Content Error
- Don't secure the Herd site (use HTTP, not HTTPS)
- Keep URLs as `http://` not `https://`

### Missing dependencies
- Worktrees don't share `vendor/` or `node_modules/`
- Run the appropriate package manager in the worktree

## License

MIT - see [LICENSE](LICENSE)
