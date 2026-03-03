---
name: jj-spr-reorganize
description: Use when reorganizing a jj change stack that has existing PRs. Triggers on "reorganize stack", "rebase changes", "squash changes", "split change", "reorder PRs", or after restructuring a stack that had open PRs.
---

# jj SPR Stack Reorganization

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

Reorganizing a stack (rebase, squash, split, reorder) when PRs already exist
is the most dangerous SPR operation. SPR tracks PRs solely via
`Pull Request:` URLs in commit messages. After reorganization, old URLs are
stale and will cause errors or wrong updates.

**The only safe path is: close → clean → reorganize → recreate.**

## Why Close-and-Recreate?

- `Pull Request:` URLs are SPR's ONLY tracking mechanism
- Stale URLs point to closed or wrong PRs
- Partial cleanup (updating some URLs) is fragile and error-prone
- A clean slate is the only reliable approach

## Step 1: Close All Affected PRs

Close every PR in the range you're about to reorganize:

```bash
# List current PRs to know what to close
jj spr list

# Close each one (deletes the remote branch too)
gh pr close <number-1> --delete-branch
gh pr close <number-2> --delete-branch
# ... repeat for all affected PRs
```

**Do not skip `--delete-branch`.** Leftover SPR branches cause confusion.

## Step 2: Strip Stale `Pull Request:` URLs

**A single leftover URL will cause SPR to try to update a closed PR.**

Use this loop to strip all `Pull Request:` lines from a set of changes:

```bash
for c in <change-id-1> <change-id-2> ...; do
  desc=$(jj log --no-graph -r "$c" -T 'description')
  clean=$(echo "$desc" | grep -v 'Pull Request:' | sed '/^$/N;/^\n$/d')
  echo "--- $c ---"
  echo "Before: $(echo "$desc" | grep 'Pull Request:' || echo '(none)')"
  jj desc -r "$c" -m "$clean"
  echo
done
```

**What this does:**
1. Reads the current description for each change
2. Removes lines containing `Pull Request:`
3. Collapses doubled blank lines left behind
4. Rewrites the description
5. Prints before/after for verification

To get the list of change IDs in the stack:

```bash
jj log -r 'ancestors(@, 20) & ~ancestors(trunk(), 1)' \
  -T 'change_id.short() ++ "\n"' --no-graph
```

**Note:** `jj desc` works from workspaces — no need to `cd` to the main
repo for this step. Only `jj spr` commands require `.git/`.

## Step 3: Perform the Reorganization

Now that PRs are closed and URLs are stripped, use standard jj commands:

### Rebase (move a change to a new parent)
```bash
jj rebase -r <change> -d <new-parent>
```

### Squash (fold a change into its parent)
```bash
jj squash -r <change>
```

### Split (break one change into two)
```bash
jj split -r <change>
```

### Reorder (move changes around in the stack)
```bash
jj rebase -r <change> --after <target>
# or
jj rebase -r <change> --before <target>
```

## Step 4: Verify the New Stack

```bash
jj log
```

Confirm:
- Changes are in the desired order
- No unexpected conflicts
- Commit messages are clean (no stale `Pull Request:` URLs)

## Step 5: Recreate PRs

```bash
jj spr diff -r <first-change>::<last-change>
```

SPR creates fresh PRs with correct base branch chaining.

## Step 6: Reposition `@`

```bash
jj new <top-of-stack>
```

`@` is now an empty change above the stack. Ready for new work.

## Quick Reference

```
1. jj spr list                              # note PR numbers
2. gh pr close <N> --delete-branch           # for each PR
3. jj desc <id> -m "<clean message>"         # for each change
4. jj rebase / squash / split               # reorganize
5. jj log                                    # verify shape
6. jj spr diff -r <range>                   # recreate PRs
7. jj new <top>                              # reposition @
```

## Common Mistakes

- **Reorganizing without closing PRs first** — stale URLs cause SPR to
  update wrong PRs or error out
- **Forgetting to strip `Pull Request:` URLs** — even one leftover URL
  means SPR thinks a PR already exists for that change
- **Trying to be clever with partial cleanup** — updating some URLs while
  keeping others. This is fragile. Always do a full close-and-recreate.
- **Not verifying the stack shape before recreating** — if the stack has
  conflicts or wrong order, PRs will be created in the wrong state
- **Leaving `@` on a PR change** — always `jj new` after reorganization
