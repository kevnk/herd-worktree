---
name: herd-worktree
description: Use when setting up a git worktree for local development with Laravel Herd, or when user asks to work on a feature branch in isolation
license: MIT
---

# Herd Worktree Setup

## Overview

Sets up a git worktree for any project served by Laravel Herd, ensuring the worktree has its own site URL.

**Announce at start:** "I'm using the herd-worktree skill to set up an isolated workspace with Herd."

## IMPORTANT: User Interaction Guidelines

**ALWAYS use the `AskUserQuestion` tool for ALL user interactions.** Never stop and wait for user reply. The AskUserQuestion tool allows the workflow to continue seamlessly.

All question blocks in this skill (shown in YAML-like format) should be implemented using the AskUserQuestion tool with the specified options.

## When to Use

- User wants to work on a feature branch in isolation
- User mentions "worktree" and the project uses Laravel Herd
- Starting work on a task that needs isolation from main branch

## Initial Flow

1. **First, check if any worktrees already exist** using `git worktree list`
2. **If worktrees exist**, immediately use AskUserQuestion to ask if they want to:
   - Set up a new worktree
   - Finish work on an existing worktree (then ask which one and proceed to "Finishing Work" section)
3. **If no worktrees exist**, use AskUserQuestion to ask for the branch name, then proceed with setup

## Prerequisites

- **Laravel Herd** installed and running on macOS
- **Git** for version control

## Setup Steps

### 0. Get Branch Name

**Use AskUserQuestion to get the branch name:**

```
AskUserQuestion:
  question: "What branch name would you like to use for this worktree?"
  header: "Branch Name"
  options:
    - label: "feature/new-feature"
      description: "Generic feature branch - you can provide a custom name"
    - label: "bugfix/fix-issue"
      description: "Bugfix branch pattern"
    - label: "experiment/test"
      description: "Experimental/testing branch"
```

**Note:** The branch name will be sanitized for the Herd link (slashes replaced with dashes), and the worktree URL will be a subdomain of the parent project. For example, if the parent project is `my-app.test` and the branch is `feature/new-feature`, the worktree URL becomes `http://feature-new-feature.my-app.test`.

### 1. Detect Parent Project's Herd Link

Before creating the worktree, detect the parent project's Herd link name:

```bash
# Get the parent project directory name (used as fallback)
PARENT_DIR=$(basename "$(pwd)")

# Check if the parent is already linked in Herd
PARENT_LINK=$(herd links | grep "$(pwd)" | awk '{print $2}' | head -1)

# Use the Herd link name if found, otherwise use directory name
PARENT_DOMAIN=${PARENT_LINK:-$PARENT_DIR}
```

### 2. Create Worktree (if needed)

**First, check that `.worktrees/` is git-ignored.** If it isn't, the main project's `git status` lists `.worktrees/` as untracked:

```bash
git check-ignore -q .worktrees/ && echo "ignored"
```

**If it prints nothing, use AskUserQuestion. Don't change `.gitignore` without asking:**

```
AskUserQuestion:
  question: "`.worktrees/` isn't git-ignored in this project. How should I handle it?"
  header: "Gitignore"
  options:
    - label: "Add to .gitignore (Recommended)"
      description: "Append `.worktrees/` to the project's .gitignore. You commit that change yourself"
    - label: "Ignore locally only"
      description: "Add it to .git/info/exclude, which affects only this clone"
    - label: "Leave it"
      description: "Change nothing. `git status` in the main project will list .worktrees/"
```

**If "Add to .gitignore":**

```bash
# Start a new line if .gitignore doesn't end with one
if [ -n "$(tail -c1 .gitignore 2>/dev/null)" ]; then echo >> .gitignore; fi
echo ".worktrees/" >> .gitignore
```

**If "Ignore locally only":**

```bash
EXCLUDE_FILE=$(git rev-parse --git-path info/exclude)
mkdir -p "$(dirname "$EXCLUDE_FILE")"
if [ -n "$(tail -c1 "$EXCLUDE_FILE" 2>/dev/null)" ]; then echo >> "$EXCLUDE_FILE"; fi
echo ".worktrees/" >> "$EXCLUDE_FILE"
```

**Then create the worktree:**

```bash
# Check if worktree exists
git worktree list | grep "$BRANCH_NAME"

# If not, create it
git worktree add .worktrees/$SANITIZED_BRANCH_NAME -b $BRANCH_NAME
```

### 3. Link with Laravel Herd (Subdomain Pattern)

The worktree is linked as a **subdomain** of the parent project, making the relationship clear:

```bash
cd /path/to/project/.worktrees/$SANITIZED_BRANCH_NAME

# Link as subdomain: {branch}.{parent}.test
# Example: feature-new-feature.my-app.test
herd link $SANITIZED_BRANCH_NAME.$PARENT_DOMAIN
```

This creates a site at `http://$SANITIZED_BRANCH_NAME.$PARENT_DOMAIN.test`

**Example URLs:**
- Parent project: `http://my-app.test`
- Worktree for `feature/login`: `http://feature-login.my-app.test`
- Worktree for `bugfix/auth`: `http://bugfix-auth.my-app.test`

**Note:** Do NOT run `herd secure` - the site should use HTTP to avoid mixed content issues with dev servers.

### 4. Configure Environment

**First, check if .env exists in the main project:**

```bash
test -f /path/to/main/project/.env && echo "exists"
```

**If .env exists, use AskUserQuestion:**

```
AskUserQuestion:
  question: "I found a .env file. Would you like me to configure it for the worktree?"
  header: "Environment"
  options:
    - label: "Copy and update URLs (Recommended)"
      description: "Copy .env and update any URLs containing the original domain to use the worktree domain"
    - label: "Copy without changes"
      description: "Just copy the .env file as-is"
    - label: "Skip .env"
      description: "Don't copy or configure .env"
```

**If "Copy and update URLs":**

```bash
# Copy .env from main project
cp /path/to/main/project/.env /path/to/worktree/.env

# Update any .test URLs to use the worktree subdomain
# This replaces any URL like http://parent.test with http://$BRANCH_NAME.parent.test
sed -i '' "s|//$PARENT_DOMAIN\.test|//$SANITIZED_BRANCH_NAME.$PARENT_DOMAIN.test|g" /path/to/worktree/.env
```

**If "Copy without changes":**

```bash
cp /path/to/main/project/.env /path/to/worktree/.env
```

### 5. Install Dependencies

**CRITICAL: Worktrees do NOT share vendor/, node_modules/, or other dependency directories with the main project.**

**Detect package managers by checking for lock/config files:**

```bash
# Check what package managers are available
ls -la composer.json package.json pnpm-lock.yaml yarn.lock package-lock.json Cargo.toml requirements.txt pyproject.toml go.mod 2>/dev/null
```

**Use AskUserQuestion to confirm which to run:**

```
AskUserQuestion:
  question: "I detected these package managers. Which should I run?"
  header: "Dependencies"
  multiSelect: true
  options:
    - label: "composer install"
      description: "PHP dependencies (composer.json found)"
    - label: "npm install"
      description: "Node dependencies (package.json + package-lock.json found)"
    - label: "pnpm install"
      description: "Node dependencies (pnpm-lock.yaml found)"
    - label: "yarn install"
      description: "Node dependencies (yarn.lock found)"
    - label: "cargo build"
      description: "Rust dependencies (Cargo.toml found)"
    - label: "pip install"
      description: "Python dependencies (requirements.txt or pyproject.toml found)"
    - label: "go mod download"
      description: "Go dependencies (go.mod found)"
    - label: "Skip dependency installation"
      description: "I'll install dependencies manually later"
```

**Run the selected package managers:**

| Package Manager | Detection File | Command |
|-----------------|----------------|---------|
| Composer | `composer.json` | `composer install --no-interaction` |
| npm | `package-lock.json` or `package.json` (no other lock) | `npm install` |
| pnpm | `pnpm-lock.yaml` | `pnpm install` |
| yarn | `yarn.lock` | `yarn install` |
| Cargo | `Cargo.toml` | `cargo build` |
| pip | `requirements.txt` | `pip install -r requirements.txt` |
| pip | `pyproject.toml` | `pip install -e .` |
| Go | `go.mod` | `go mod download` |

**Ask user if installation fails:** "The dependency installation failed. Would you like me to:
1. Retry with different flags
2. Skip and continue (not recommended)
3. Show the error for manual debugging"

## Finishing Work (Integrating Back to Main Tree)

When development is complete and you want to integrate the worktree changes, **immediately use AskUserQuestion** to ask how they want to proceed:

```
AskUserQuestion:
  question: "How would you like to finish your worktree changes?"
  header: "Finish Work"
  options:
    - label: "Create PR from worktree (Recommended)"
      description: "Commit, push, and create a PR directly from the worktree branch"
    - label: "Transfer to main directory"
      description: "Merge changes into the main project directory, then clean up the worktree"
    - label: "Abandon changes"
      description: "Discard all changes and remove the worktree"
```

**After setup completes, tell the user:** "When you're ready to finish your work, run `/herd-worktree` again and I'll help you integrate or clean up your changes."

---

### Option A: Create PR from Worktree

Use this when you want to keep changes isolated and create a PR directly from the worktree.

#### A.0. Gather Information

**Use AskUserQuestion to gather information (can ask multiple questions at once):**

```
AskUserQuestion:
  questions:
    - question: "Do you have a task/issue number for this work?"
      header: "Task ID"
      options:
        - label: "No task number"
          description: "Skip adding a task identifier to the branch and commit"
        - label: "Enter task number"
          description: "I'll provide a task/issue number to include in the branch name and commit"
    - question: "How would you like to handle the PR description?"
      header: "PR Body"
      options:
        - label: "I'll write it"
          description: "Create PR with empty body - I'll fill in the description on GitHub"
        - label: "Generate for me"
          description: "Analyze the changes and generate a PR description automatically"
        - label: "Leave empty"
          description: "Create PR with no description"
```

#### A.1. Commit All Changes

```bash
cd /path/to/project/.worktrees/$BRANCH_NAME
git add -A
git commit -m "Your commit message (#TASK_NUMBER)"  # Omit (#TASK_NUMBER) if none provided
```

#### A.2. Rename Branch with Task Number (if provided)

If a task number was provided and the branch doesn't include it:

```bash
git branch -m old-branch-name TASK_NUMBER-descriptive-name
```

#### A.3. Push and Create PR

Set `DEFAULT_BRANCH` before `gh pr create`, in the same shell. It uses the remote's default branch, then the default GitHub reports, then `main`. If the project merges feature work into another branch (for example `develop`), set `DEFAULT_BRANCH` to that branch instead.

```bash
DEFAULT_BRANCH=$(git symbolic-ref --quiet --short refs/remotes/origin/HEAD 2>/dev/null)
DEFAULT_BRANCH=${DEFAULT_BRANCH#origin/}
[ -n "$DEFAULT_BRANCH" ] || DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq .defaultBranchRef.name 2>/dev/null)
DEFAULT_BRANCH=${DEFAULT_BRANCH:-main}
echo "PR base: $DEFAULT_BRANCH"

git push -u origin BRANCH_NAME

# Based on user's PR description preference:
# - "I'll write it" or "Leave empty": gh pr create --base $DEFAULT_BRANCH --title "Title" --body ""
# - "Generate for me": Generate description from git diff, then use --body "$DESCRIPTION"
gh pr create --base $DEFAULT_BRANCH --title "Description (#TASK_NUMBER)" --body "$BODY"
```

#### A.4. Cleanup After Merge

After the PR is merged, clean up the worktree:

```bash
# From main project directory
cd /path/to/main/project
herd unlink $SANITIZED_BRANCH_NAME.$PARENT_DOMAIN
git worktree remove .worktrees/$SANITIZED_BRANCH_NAME
git branch -d TASK_NUMBER-branch-name  # Delete local branch
```

---

### Option B: Transfer Changes to Main Directory

Use this when you want to bring your worktree changes back into the main project directory, review them there, and then commit/push from the main directory.

#### B.1. Confirm Transfer

**Use AskUserQuestion to confirm:**
```
AskUserQuestion:
  question: "This will merge your worktree branch into the main directory. Continue?"
  header: "Confirm"
  options:
    - label: "Yes, transfer changes"
      description: "Merge the worktree branch into the main project and clean up"
    - label: "Cancel"
      description: "Go back without transferring"
```

#### B.2. Check the Worktree Is Clean

The merge brings over committed work only. Uncommitted edits in the worktree would be left behind, and `git worktree remove` would refuse to remove the worktree later. Check first:

```bash
# Navigate to main project directory
cd /path/to/main/project

# Prints nothing when the worktree is clean
git -C .worktrees/$SANITIZED_BRANCH_NAME status --porcelain
```

**If it prints anything, stop.** Show the user the list, then use AskUserQuestion:

```
AskUserQuestion:
  question: "The worktree has uncommitted changes, and the transfer would leave them behind. What should I do?"
  header: "Uncommitted"
  options:
    - label: "Commit them first (Recommended)"
      description: "Commit the changes on the worktree branch, then continue the transfer"
    - label: "Cancel"
      description: "Stop here and leave the worktree as it is"
```

If they choose to commit, run `git add -A && git commit` inside the worktree, then run the check again. Continue to B.3 only when the check prints nothing.

#### B.3. Merge Worktree Branch

```bash
# Fetch and merge the worktree branch into current branch
git merge $BRANCH_NAME --no-commit --no-ff
```

The `--no-commit` flag stages the changes without committing, allowing the user to review and commit manually. The `--no-ff` ensures a merge commit is created for clarity.

**If merge conflicts occur:**
- Inform the user about the conflicts
- List the conflicting files
- Ask if they want to resolve them now or abort
- If they abort, run `git merge --abort` and stop. The branch and worktree are untouched

#### B.4. Remove the Worktree (Keep the Branch)

After the merge is staged:

```bash
# Remove the worktree, then unlink from Herd only if that worked
git worktree remove .worktrees/$SANITIZED_BRANCH_NAME && herd unlink $SANITIZED_BRANCH_NAME.$PARENT_DOMAIN
```

**If `git worktree remove` fails, stop and tell the user why.** Don't retry with `--force`, which deletes uncommitted files. Don't tell the user the cleanup is done.

**Don't delete `$BRANCH_NAME` yet.** The merge is staged, not committed. If the user runs `git merge --abort`, the branch is the only thing still pointing at the worktree's commits.

#### B.5. Final Steps

**Inform the user:**
"Your changes have been merged into the main directory and are staged for commit. The worktree has been removed. The `$BRANCH_NAME` branch stays until you commit, so `git merge --abort` can't lose your work.

You can now:
- Review the changes with `git status` and `git diff --cached`
- Commit when ready with `git commit`
- Or unstage specific files if needed with `git reset HEAD <file>`
- After you commit, delete the branch with `git branch -d $BRANCH_NAME` (it refuses if the merge isn't committed)"

---

### Option C: Abandon Changes

When abandoning a worktree without merging:

```bash
# Unlink from Herd (use the subdomain link name)
herd unlink $SANITIZED_BRANCH_NAME.$PARENT_DOMAIN

# Remove worktree
git worktree remove .worktrees/$SANITIZED_BRANCH_NAME

# Delete branch if not needed
git branch -D $BRANCH_NAME
```

## Quick Reference

| Step | Command |
|------|---------|
| Detect parent | `PARENT_DOMAIN=$(basename "$(pwd)")` |
| Create worktree | `git worktree add .worktrees/BRANCH -b BRANCH` |
| Link to Herd | `herd link BRANCH.PARENT_DOMAIN` |
| Copy .env | `cp ../.env .env` |
| Update URLs | `sed -i '' "s\|//$PARENT_DOMAIN\.test\|//$BRANCH.$PARENT_DOMAIN.test\|g" .env` |
| Unlink | `herd unlink BRANCH.PARENT_DOMAIN` |
| Remove worktree | `git worktree remove .worktrees/BRANCH` |

## Troubleshooting

### Site not loading
- **Cause:** Herd link doesn't exist or is misconfigured
- **Fix:**
  1. Verify link exists: `herd links | grep $BRANCH_NAME`
  2. If missing, create it: `herd link $SANITIZED_BRANCH_NAME.$PARENT_DOMAIN`
  3. Ensure Herd is running
  4. Verify the URL uses subdomain pattern: `http://branch.parent.test`

### Missing dependencies
- **Cause:** Worktrees don't share dependency directories
- **Fix:** Run the appropriate package manager in the worktree:
  - PHP: `composer install`
  - Node: `npm install` / `yarn install` / `pnpm install`
  - Rust: `cargo build`
  - Python: `pip install -r requirements.txt`
  - Go: `go mod download`

### Wrong domain in config
- **Cause:** .env URLs still point to the parent domain instead of worktree subdomain
- **Fix:** Update .env URLs to use the worktree subdomain:
  ```bash
  sed -i '' "s|//$PARENT_DOMAIN\.test|//$SANITIZED_BRANCH_NAME.$PARENT_DOMAIN.test|g" .env
  ```

### Port conflicts
- **Cause:** Another dev server is using the same port
- **Fix:** Stop conflicting processes or use a different port in your dev server configuration

### Permission issues with Herd
- **Cause:** Herd needs to access worktree directory
- **Fix:** Ensure worktree is in a Herd-accessible path

### Mixed content errors
- **Cause:** HTTPS site trying to load HTTP assets
- **Fix:**
  1. Unsecure the site: `herd unsecure $SANITIZED_BRANCH_NAME.$PARENT_DOMAIN`
  2. Update URLs to use `http://` instead of `https://`

## Checklist

Before considering setup complete, verify:

- [ ] Parent domain identified (`$PARENT_DOMAIN`)
- [ ] `.worktrees/` is git-ignored, or the user chose to leave it
- [ ] Worktree created at `.worktrees/$SANITIZED_BRANCH_NAME`
- [ ] Herd link created as subdomain (`herd link $SANITIZED_BRANCH_NAME.$PARENT_DOMAIN`)
- [ ] Site is NOT secured (use HTTP, not HTTPS)
- [ ] .env copied (if applicable) with URLs updated to subdomain
- [ ] Dependencies installed (composer/npm/etc.)
- [ ] Site accessible at `http://$SANITIZED_BRANCH_NAME.$PARENT_DOMAIN.test`

## CRITICAL: Working Directory After Setup

**After worktree setup is complete, ALL subsequent work MUST use the worktree path.**

The worktree is a separate copy of the codebase. Using the main repo path will edit the wrong files.

- **Worktree path:** `/path/to/project/.worktrees/$SANITIZED_BRANCH_NAME/`
- **Worktree URL:** `http://$SANITIZED_BRANCH_NAME.$PARENT_DOMAIN.test`
- **All file reads, edits, and writes** must use the worktree absolute path
- **All Bash commands** must run from the worktree directory
- **All git operations** must run from the worktree directory

**After setup, remind the user and any subsequent agents:** "All work for this task should use the worktree at `/path/to/project/.worktrees/$SANITIZED_BRANCH_NAME/` accessible at `http://$SANITIZED_BRANCH_NAME.$PARENT_DOMAIN.test`. Do not modify files in the main project directory."

## Integration

**Pairs with:**
- Git worktree workflows
- Any development workflow using Laravel Herd for local serving
