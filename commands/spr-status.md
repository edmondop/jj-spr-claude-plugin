---
name: spr-status
description: Show which jj changes have associated PRs
---

# /spr-status

Show the mapping between local jj changes and their GitHub PRs.

## Instructions

**First:** Check if you're in a jj workspace (`.jj/repo` is a file, not a
directory). If so, `cd` to the main colocated repo before proceeding.

1. Get the list of changes in the current stack:
   ```bash
   jj log -r 'ancestors(@, 20) & ~ancestors(trunk(), 1)' \
     -T 'change_id.short() ++ "\n"' --no-graph
   ```
   Adjust the revset if the user specifies a different range.

2. For each change, extract the `Pull Request:` URL from its description:
   ```bash
   for c in <change-ids>; do
     url=$(jj log -r "$c" -T 'description' --no-graph | grep 'Pull Request:' | head -1)
     if [ -n "$url" ]; then
       echo "$c: $url"
     else
       echo "$c: (no PR)"
     fi
   done
   ```

3. Present a summary table:
   - Change ID → PR URL (or "no PR")
   - Count: N changes with PRs, M without

This is useful before:
- `/spr-diff` — see what already has PRs
- `/spr-reorg` — know which PRs to close
- General awareness of stack state
