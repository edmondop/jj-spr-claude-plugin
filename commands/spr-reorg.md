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

Use the `jj-spr-reorganize` skill. Follow its decision tree to determine
which PRs need to be closed:

- **Removing a change** → close only that change's PR with `jj spr close`
- **Splitting a change** → the original PR stays on one half, new half gets
  a new PR automatically
- **Reordering changes** → close PRs for changes whose position changed
- **Full restructuring** → close all PRs with `jj spr close -r <range>`

**Always use `jj spr close`, NOT `gh pr close`.** `jj spr close` strips
the `Pull Request:` URL from the commit message and deletes both the head
branch and synthetic base branch. `gh pr close` does none of this.

**Always pass `-m` when updating existing PRs:**
```bash
jj spr diff -m "reorganized stack" -r <bottom>::<top>
```
Without `-m`, `jj spr diff` prompts interactively when updating (not
creating) PRs, which blocks automation.

**Always dry-run before pushing:**
```bash
jj spr diff --dry-run -r <bottom>::<top>
```
Verify changes with URLs show "UPDATE" and changes without show "CREATE".
