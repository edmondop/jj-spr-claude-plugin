---
name: spr-amend
description: Amend a jj change and update its PR
---

# /spr-amend

Modify an existing jj change and update its associated GitHub PR.

## Instructions

**First:** Check if you're in a jj workspace (`.jj/repo` is a file, not a
directory). If so, `cd` to the main colocated repo before proceeding.

Use the `jj-spr-amend-update` skill to guide the amend-update cycle.

1. Ask the user which change to modify (or infer from context)
2. `jj edit <change-id>` to make it the working copy
3. Make the requested code changes
4. **MANDATORY:** `jj new <change-id>` to reposition `@`
5. `jj spr diff -r <change-id>` to update the PR
6. If the change is in the middle of a stack, update the full range

Never skip step 4. The user must be able to review with `jj diff -r @-`.
