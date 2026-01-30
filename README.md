# Herd Worktree

<p align="center">
  <img src="art/logo.png" alt="Herd Worktree" width="400">
</p>

A Claude Code skill that automates setting up git worktrees for any project served by Laravel Herd on macOS.

## What It Does

When you need to work on a feature branch in isolation, this skill:

1. **Creates a git worktree** in `.worktrees/<branch-name>`
2. **Links with Laravel Herd** to serve the worktree at `http://<branch-name>.test`
3. **Detects project type** - automatically identifies Laravel, WordPress, or static projects
4. **Configures environment** - copies and updates configuration files as needed
5. **Installs dependencies** - runs appropriate install commands for your project type

It also handles cleanup and PR creation when you're done.

## Supported Project Types

- **Laravel** - Full support including `.env` configuration, Composer, npm, Vite, and Sanctum
- **WordPress** - Database cloning, `wp-config.php` updates, and search-replace for URLs
- **Static/Other** - Basic worktree setup with Herd linking

## Prerequisites

- **Laravel Herd** installed and running on macOS
- **Git** for version control

### For Laravel Projects
- **Vite** as the frontend build tool
- **npm** as package manager (or yarn/pnpm)

### For WordPress Projects
- **WP-CLI** for database operations (`brew install wp-cli`)

## Installation

### Via Claude Marketplace (Recommended)

```bash
/plugin install herd-worktree@harrisrafto-laravel-tools
```

### Manual Installation

1. Clone this repository:
   ```bash
   git clone https://github.com/harris21/herd-worktree.git ~/.claude/plugins/herd-worktree
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

## Configuration

### Default Branch

The skill defaults to `develop` as the base branch for PRs. If your project uses a different default branch (e.g., `main` or `master`), you can:

1. Let Claude detect it automatically (it will check `git symbolic-ref refs/remotes/origin/HEAD`)
2. Specify it when creating the PR

### Package Manager (Laravel)

The skill assumes npm. If you use yarn or pnpm, the skill will ask before running install commands.

### Composer Flags (Laravel)

If your project requires specific composer flags (like `--ignore-platform-reqs`), the skill will ask during setup.

## What Gets Created

### Laravel Projects

```
your-project/
├── .worktrees/
│   └── feature-branch/      # Your isolated worktree
│       ├── .env             # Configured for http://feature-branch.test
│       ├── vendor/          # Fresh composer install
│       └── node_modules/    # Fresh npm install
```

### WordPress Projects

```
your-project/
├── .worktrees/
│   └── feature-branch/      # Your isolated worktree
│       └── wp-config.php    # Updated with worktree database
```

Plus a cloned database with URLs updated via `wp search-replace`.

### Static/Other Projects

```
your-project/
├── .worktrees/
│   └── feature-branch/      # Your isolated worktree
```

All worktrees get a Herd site at `http://feature-branch.test`.

## Common Issues

### Laravel: 401 Unauthorized on API routes
- Add worktree domain to `SANCTUM_STATEFUL_DOMAINS` in `.env`
- Run `php artisan config:clear`

### Laravel: Cookie rejected for invalid domain
- Update `SESSION_DOMAIN` in `.env` to match worktree domain
- Add `SESSION_SECURE_COOKIE=false` for HTTP sites

### Laravel: CORS Errors / White page
- Ensure `vite.config.js` has `host: 'localhost'` and `cors: true`
- Restart Vite from the worktree directory

### Laravel: Mixed Content Error
- Don't secure the Herd site (use HTTP, not HTTPS)
- Keep `APP_URL` as `http://` not `https://`

### Laravel: Assets not loading
- Kill all Vite processes: `pkill -f "node.*vite"`
- Remove hot file: `rm -f public/hot`
- Restart Vite from worktree

### WordPress: Database connection errors
- Ensure MySQL/MariaDB is running via Herd
- Check that WP-CLI can connect: `wp db check`

### WordPress: URLs not updated
- Run `wp search-replace` manually if needed
- Check `wp-config.php` has correct database name

See the skill's "Common Issues" section for complete troubleshooting.

## Publishing (For Maintainers)

To add this skill to the Claude marketplace:

```bash
/plugin marketplace add harris21/herd-worktree
```

To update an already installed plugin:

```bash
/plugin update herd-worktree@harrisrafto-laravel-tools
```

## License

MIT - see [LICENSE](LICENSE)
