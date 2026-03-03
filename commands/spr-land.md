---
name: spr-land
description: Land a PR via SPR and clean up
---

# /spr-land

Land (squash-merge) a PR on GitHub and sync the local repository.

## Instructions

**First:** Check if you're in a jj workspace (`.jj/repo` is a file, not a
directory). If so, `cd` to the main colocated repo before proceeding.

Use the `jj-spr-landing` skill to guide the landing process.

1. Identify which PR to land (ask user or infer from context)
2. Run pre-checks (approval status, CI)
3. `jj spr land -r <change-id>`
4. `jj git fetch`
5. `jj abandon <landed-change-id>` (now empty, content is on master)
6. `jj rebase -r @ -d main@origin` (single PR) or
   `jj rebase -s <next-change> -d main@origin` (stack -- note `-s` not `-r`)
7. If part of a stack: `jj spr diff -r <next-change>::<top>` to update
   synthetic base content

**Always land bottom-up** in a stack. Warn the user if they're trying to
land from the middle or top.

**After step 7, remaining PRs will still target their synthetic base branches
on GitHub. This is normal and expected.** The diffs are correct (synthetic
base content matches master), and `jj spr land` retargets to master
automatically at merge time. NEVER suggest `gh pr edit --base master` as a
workaround -- it bypasses SPR's synthetic commit management.

**CRITICAL: Never merge on GitHub directly (click "Merge" or `gh pr merge`).
Always use `jj spr land`.** For stacked PRs, a direct GitHub merge sends
the code to the synthetic base branch, not master. See the skill for
recovery steps if this already happened.
