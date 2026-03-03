---
name: jj-spr-amend-update
description: Use when amending a jj change and updating its associated PR. Triggers on "update PR", "address review feedback", "amend and push", "fix review comments", or when modifying code that already has an open PR.
---

# jj SPR Amend & Update

## Workspace Detection (MUST check first)

**Before running ANY SPR command, check if you're in a jj workspace.**

```bash
if [ -f .jj/repo ]; then
  MAIN_REPO=$(dirname "$(dirname "$(cat .jj/repo)")")
  cd "$MAIN_REPO"
fi
```

SPR requires `.git/` which only exists in the main colocated repo, not in
workspaces. All `jj spr` commands must run from the main repo.

## Overview

The core day-to-day cycle: modify a jj change locally, then update its PR on
GitHub. This covers addressing review feedback, fixing bugs in open PRs, and
any post-creation modification.

## CRITICAL: Preserving the `Pull Request:` URL

**`jj describe -m "..."` replaces the ENTIRE commit message.** If the change
already has a PR (created by `jj spr diff`), a blind `jj describe` wipes
the `Pull Request:` tracking URL and causes SPR to create a duplicate PR.

**Before ANY `jj describe` on a change that has a PR:**

```bash
# 1. Read current description to check for Pull Request: line
jj log -r <change-id> --no-graph -T 'description'

# 2. Include the Pull Request: line at the bottom of the new message
jj describe -r <change-id> -m 'Updated summary

Pull Request: https://github.com/org/repo/pull/12345'
```

**This applies every time you touch a commit message** — whether adding a
description, fixing a typo, or appending a section. Read first, preserve
the URL, always.

## The Amend-Update Loop

### Step 1: Identify the Target Change

```bash
jj log   # find the change-id for the PR to modify
```

Look for the `Pull Request:` URL in the commit message to confirm which PR
this change corresponds to.

### Step 2: Edit the Change

```bash
jj edit <change-id>   # @ now points at the PR change
```

This makes the PR change the current working copy. All file modifications
will be recorded in this change.

### Step 3: Make Modifications

Edit files as needed. jj automatically tracks all changes — no `git add`
required.

### Step 4: Reposition `@`

**MANDATORY after every edit session:**

```bash
jj new <change-id>   # @ is now empty, @- is the modified change
```

This lets the user review what changed with `jj diff -r @-` before pushing.

**Never skip this step.** Never leave `@` pointing at a PR change after edits.

### Step 5: Update the PR

```bash
jj spr diff -m "addressed feedback" -r <change-id>
```

**Always pass `-m`** when updating an existing PR. Without it, SPR prompts
interactively for an update message, blocking automation. The `-m` value
is used as the update comment, not the PR description.

SPR pushes a new synthetic commit to the PR branch. GitHub shows it as an
incremental update — reviewers see only the new changes in the diff.

### Step 6: Verify

```bash
# Quick check that the PR updated correctly
jj spr list
```

Or verify on GitHub that the diff looks right.

## Mid-Stack Modifications

When modifying a change in the **middle** of a stack:

1. Follow Steps 1-4 above as normal
2. After editing, descendants in the stack are automatically rebased by jj
3. Update the **full range** to ensure all PRs reflect the rebase:
   ```bash
   jj spr diff -m "updated mid-stack change" -r <modified-change>::<top-of-stack>
   ```
4. Verify all PRs in the stack have correct diffs

**Why the full range?** Because the rebase changes the tree of every
descendant, and SPR needs to push updated synthetic commits for each one.

## Review Feedback Workflow

When addressing review comments on a PR:

1. **Read review comments**
   ```bash
   gh pr view <number> --comments
   ```

2. **Identify which change to modify** — match the PR number to a change
   via `Pull Request:` URL in `jj log`

3. **Follow the amend-update loop** (Steps 1-6 above)

4. **Reply to review comments** after pushing updates:
   ```bash
   gh pr comment <number> --body "Addressed — updated in latest push"
   ```

## Multiple PRs with Feedback

If multiple PRs in a stack have review feedback:

1. Start from the **bottom** of the stack (earliest ancestor)
2. Amend that change, reposition `@`
3. Move to the next change up, amend it
4. After all edits, update the full range:
   ```bash
   jj spr diff -m "addressed review feedback" -r <first-modified>::<top-of-stack>
   ```

Working bottom-up prevents redundant rebases.

## Common Mistakes

- **Forgetting to reposition `@`** — leaves user unable to review changes
  before pushing
- **Updating only the modified change** in a stack — descendants won't
  reflect the rebase on GitHub
- **Using `jj squash` instead of `jj edit`** — `jj edit` modifies in place,
  `jj squash` merges changes from a child into a parent. For the
  amend-update loop, `jj edit` is what you want.
- **Using `jj describe -m "..."` without preserving the `Pull Request:`
  URL** — wipes SPR's tracking URL and causes duplicate PRs on next
  `jj spr diff`. Always read the current description first.
