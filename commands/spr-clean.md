---
name: spr-clean
description: Strip stale Pull Request URLs from jj commit messages
---

# /spr-clean

Remove `Pull Request:` URLs from commit messages in a range of jj changes.
Useful before re-running `jj spr diff`, after manually closing PRs, or as
part of a stack reorganization.

## Instructions

1. Determine the affected changes. If the user specifies a range, use it.
   Otherwise, find all changes with `Pull Request:` URLs in the stack:
   ```bash
   jj log -r 'ancestors(@, 20) & ~ancestors(trunk(), 1)' \
     -T 'change_id.short() ++ "\n"' --no-graph
   ```

2. Strip URLs from each change:
   ```bash
   for c in <change-ids>; do
     desc=$(jj log --no-graph -r "$c" -T 'description')
     clean=$(echo "$desc" | grep -v 'Pull Request:' | sed '/^$/N;/^\n$/d')
     echo "--- $c ---"
     echo "Before: $(echo "$desc" | grep 'Pull Request:' || echo '(none)')"
     jj desc -r "$c" -m "$clean"
     echo
   done
   ```

3. Report results: how many changes were cleaned, how many had no URL.

**Note:** This command works from jj workspaces — `jj desc` does not
require `.git/`. No need to `cd` to the main repo for this step.

**Tip:** Run `/spr-status` first to see which changes have URLs.
