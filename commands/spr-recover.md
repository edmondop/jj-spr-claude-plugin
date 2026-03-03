---
name: spr-recover
description: Diagnose and recover a broken jj+SPR stack
---

# /spr-recover

Diagnose and recover a jj+SPR stack that's in a bad state (stale PRs,
ghost changes, orphaned PRs, wrong diffs on GitHub).

## Instructions

**First:** Check if you're in a jj workspace (`.jj/repo` is a file, not a
directory). If so, `cd` to the main colocated repo before proceeding.

Use the `jj-spr-recovery` skill to guide the full recovery process.

1. Diagnose: map changes to PRs, identify ghosts and stale bases
2. **Present diagnosis and get user confirmation before any destructive action**
3. Close stale/orphaned PRs with `--delete-branch`
4. Abandon empty ghost changes
5. Strip `Pull Request:` URLs from remaining changes
6. `jj git fetch` and rebase onto `main@origin`
7. Dry-run `jj spr diff` to verify range
8. Recreate PRs with `jj spr diff`
9. Verify with `jj log` and `jj spr list`

**This is a destructive recovery operation.** Always confirm with the user
before closing PRs or abandoning changes.
