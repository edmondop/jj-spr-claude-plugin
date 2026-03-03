---
name: jj-spr-init
description: Use when setting up SPR in a Jujutsu repository for the first time. Triggers on "set up spr", "configure jj spr", "init spr", or when a repo needs SPR configuration.
---

# jj SPR Init

## Overview

Set up a Jujutsu repository to use SPR for GitHub pull request management. This skill guides the full initialization: prerequisites, token setup, config verification, and connectivity test.

## Workspace Detection (MUST check first)

**Before running ANY SPR command, check if you're in a jj workspace.**

A jj workspace has `.jj/` but NO `.git/`. SPR requires `.git/` and will
fail silently or with confusing errors in a workspace.

```bash
# Check: is .jj/repo a FILE (workspace) or a DIRECTORY (main repo)?
if [ -f .jj/repo ]; then
  echo "IN WORKSPACE — read .jj/repo for main repo path"
  cat .jj/repo
else
  echo "IN MAIN REPO"
fi
```

**If `.jj/repo` is a file:** You're in a workspace. The file contains the
path to the main repo's `.jj/repo`. Derive the main repo root:

```bash
# .jj/repo contains e.g. /Users/you/code/myproject/.jj/repo
# The main repo root is two directories up from that path
MAIN_REPO=$(dirname "$(dirname "$(cat .jj/repo)")")
cd "$MAIN_REPO"
```

**All SPR commands must run from the main colocated repo, not a workspace.**

## Prerequisites Check

Before running `jj spr init`, verify all prerequisites:

```bash
# 1. jj is installed
jj version

# 2. git is installed
git version

# 3. Repo is colocated (BOTH must exist)
ls -d .jj .git
```

**If `.git/` is missing:** The repo is not colocated. SPR requires a colocated repo because it pushes Git branches to GitHub.

```bash
# Convert to colocated (from inside the jj repo)
jj git init --colocate
```

**If `.jj/` is missing:** This is a plain Git repo. Initialize jj first:

```bash
jj git init --colocate
```

## Running Init

```bash
jj spr init
```

This interactive command will prompt for:
- **GitHub token** — needs `repo` scope for private repos, `public_repo` for public
- **Repository** — `owner/repo` format (auto-detected from git remote if possible)

The config is stored in `.git/config` under the `spr.*` namespace.

## Verify Configuration

After init, verify everything is set:

```bash
# Check required config values
git config spr.githubRepository   # should print owner/repo
git config spr.githubToken        # should print the token

# Optional config
git config spr.branchPrefix       # default: spr/
git config spr.requireApproval    # default: not set
```

## Test Connectivity

```bash
jj spr list
```

If this succeeds (even with "No pull requests found"), the setup is complete.

**Common failures:**
- `401 Unauthorized` — token is invalid or expired
- `404 Not Found` — wrong `spr.githubRepository` value, or token lacks `repo` scope for private repos
- `Could not find .git directory` — not a colocated repo

## Checklist

1. `jj version` works
2. `git version` works
3. `.jj/` and `.git/` both exist
4. `jj spr init` completed
5. `git config spr.githubRepository` returns `owner/repo`
6. `git config spr.githubToken` is set
7. `jj spr list` succeeds
