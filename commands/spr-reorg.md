---
name: spr-reorg
description: Reorganize a jj change stack with existing PRs
---

# /spr-reorg

Reorganize a jj change stack (rebase, squash, split, reorder) when PRs
already exist.

## Instructions

**First:** Check if you're in a jj workspace (`.jj/repo` is a file, not a
directory). If so, `cd` to the main colocated repo before proceeding.

Use the `jj-spr-reorganize` skill. This is the most dangerous operation —
follow the skill exactly.

1. List current PRs with `jj spr list`
2. Close all affected PRs: `gh pr close <N> --delete-branch`
3. Strip `Pull Request:` URLs from all affected commit messages
4. Perform the reorganization (rebase, squash, split, etc.)
5. Verify the new stack shape with `jj log`
6. Recreate PRs with `jj spr diff -r <range>`
7. Reposition `@` with `jj new <top-of-stack>`

**Never skip the close-and-clean steps.** There is no safe shortcut.
