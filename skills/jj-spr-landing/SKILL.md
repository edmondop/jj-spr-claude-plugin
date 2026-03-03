---
name: jj-spr-landing
description: Use when landing (merging) a PR via SPR and cleaning up afterward. Triggers on "land PR", "merge PR", "spr land", "ship it", or when a PR is approved and ready to merge.
---

# jj SPR Landing

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

Land a PR by squash-merging it on GitHub, then sync the local repo. Covers
single PRs and stacked PR landing with proper cleanup.

## Pre-checks

Before landing:

1. **PR is approved** (if `spr.requireApproval` is configured)
   ```bash
   gh pr view <number> --json reviewDecision
   ```

2. **CI is passing**
   ```bash
   gh pr checks <number>
   ```

3. **No merge conflicts** -- SPR will refuse to land if the PR can't be
   cleanly squash-merged

## Landing a Single PR

```bash
jj spr land -r <change-id>
```

This:
- If the PR targets a synthetic base, retargets it to master first
- Squash-merges the PR on GitHub
- Deletes the remote head branch (and base branch if it was a synthetic one)

Then sync locally:

```bash
jj git fetch
jj rebase -r @ -d main@origin
```

## How SPR Handles Synthetic Bases at Land Time

**Critical to understand:** `jj spr land` retargets the PR base from its
synthetic base branch to master BEFORE squash-merging. This is automatic.
You never need to manually retarget a PR base to master. See `land.rs`
lines 102-147.

This means a PR can safely "sit on" a synthetic base branch between
`jj spr diff` updates. The synthetic base is cosmetically ugly on GitHub
but functionally correct: the diff shows only that PR's own changes, and
landing works because SPR retargets at merge time.

## Landing the Bottom of a Stack

**Always land bottom-up.** Landing from the middle or top of a stack is
dangerous and will leave orphaned PRs.

### Step 1: Land the bottom PR

```bash
jj spr land -r <bottom-change-id>
```

SPR retargets to master, squash-merges, and deletes the landed PR's head
and base branches.

### Step 2: Sync locally

```bash
jj git fetch
```

### Step 3: Abandon the landed change and rebase remaining stack

```bash
# The landed change is now empty (its content is on master)
jj abandon <landed-change-id>

# Rebase the remaining stack onto updated master
jj rebase -s <next-change> -d main@origin
```

Where `<next-change>` is the change that was directly above the landed one.
Use `-s` (not `-r`) to rebase the entire subtree.

### Step 4: Update remaining PRs

```bash
jj spr diff -r <next-change>::<top-of-stack>
```

**What this does and does NOT do:**

- DOES update the synthetic base branch content (so GitHub diffs remain
  correct, showing only each PR's own changes)
- Does NOT retarget existing PRs from their synthetic base to master

The next PR in the stack will still show its synthetic base branch as the
target on GitHub. **This is expected and harmless.** When you land that PR
(Step 1 of the next cycle), SPR retargets to master automatically.

**NEVER suggest `gh pr edit <number> --base master` as a workaround.**
This bypasses SPR's synthetic commit management and can produce wrong diffs.
If a cosmetically clean master target is required, close the PR, strip the
`Pull Request:` URL from the commit message, and recreate with
`jj spr diff -r <change>`.

### Step 5: Verify

```bash
jj log          # clean graph, no orphaned changes
jj spr list     # remaining PRs look correct
```

Check GitHub: diffs should show only each PR's own changes. Ignore the
synthetic base branch name in the PR target -- it's cosmetic.

## Landing Multiple Stacked PRs

If all PRs in a stack are approved, land them one at a time, bottom-up:

```bash
# Land first
jj spr land -r <change-1>
jj git fetch
jj abandon <change-1>
jj rebase -s <change-2> -d main@origin
jj spr diff -r <change-2>::<top>

# Land second
jj spr land -r <change-2>
jj git fetch
jj abandon <change-2>
jj rebase -s <change-3> -d main@origin
jj spr diff -r <change-3>::<top>

# Continue until done
```

**Do not try to land all at once.** Each land changes the trunk, and the
next change needs to be rebased before it can be landed.

**You can skip the `jj spr diff` step between landings** if you're landing
the entire stack in one session. SPR retargets at land time, so the
intermediate diff updates are only needed if you want correct GitHub diffs
while waiting for reviews.

## Post-Landing Cleanup

After all landing is complete:

```bash
# Sync and rebase working copy
jj git fetch
jj rebase -r @ -d main@origin

# Verify clean state
jj log
```

The landed changes will appear as immutable commits in the log.

Orphan SPR branches from landed PRs should be cleaned up automatically
(SPR deletes head + base branches on land). If any remain, use
`jj spr cleanup` to find and remove them.

## NEVER Merge on GitHub Directly

**Always use `jj spr land`. Never click "Merge" on GitHub or use
`gh pr merge`.**

`jj spr land` retargets stacked PRs from their synthetic base to master
before merging. If you merge on GitHub directly:

- **PR targeting master** (non-stacked): merge goes to master. Works, but
  you skip SPR's branch cleanup and local sync. You'll need to manually
  `jj git fetch`, abandon the change, rebase, and delete leftover remote
  branches.

- **PR targeting a synthetic base** (stacked): merge goes to the synthetic
  base branch, NOT master. **Master is not updated.** Your code ends up on
  a dead-end branch. Recovery requires cherry-picking to master or closing
  and redoing the PR.

If someone already merged a stacked PR on GitHub directly:

1. Check if master was actually updated: `git log origin/master --oneline -5`
2. If not, the merge went to the synthetic base. You need to manually
   get the changes onto master (cherry-pick, or close and recreate).
3. Clean up orphan branches: `jj spr cleanup --confirm`

## Common Mistakes

- **Landing from the middle of a stack** -- leaves upper PRs orphaned with
  wrong base branches. Always land bottom-up.
- **Forgetting `jj git fetch`** -- local repo won't know about the merge
  on GitHub. Must fetch before rebase.
- **Forgetting to rebase remaining stack** -- remaining PRs will have stale
  base branches and show wrong diffs on GitHub.
- **Using `-r` instead of `-s` for rebase** -- `-r` rebases only one
  change, `-s` rebases the entire subtree. Use `-s` for stacks.
- **Manually retargeting PRs with `gh pr edit --base master`** -- bypasses
  SPR's synthetic commit system. Let SPR handle retargeting at land time.
- **Panicking because a PR targets a synthetic base after diff update** --
  this is normal. The diff is correct and landing will work. SPR retargets
  to master before squash-merging.
- **Landing with failing CI** -- SPR may allow it, but reviewers and
  automation expect green CI first.
