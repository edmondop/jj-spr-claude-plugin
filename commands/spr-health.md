---
name: spr-health
description: Check the health of a jj+SPR stack
---

# /spr-health

Diagnose the health of the current jj+SPR stack without making any changes.

## Instructions

**First:** Check if you're in a jj workspace (`.jj/repo` is a file, not a
directory). If so, `cd` to the main colocated repo before proceeding.

This is a **read-only** diagnostic. It never modifies anything.

### Step 1: Identify the stack

```bash
jj log -r 'ancestors(@, 20) & ~ancestors(trunk(), 1)'
```

### Step 2: Map changes to PRs

For each change above trunk, extract the `Pull Request:` URL and check
if the change is empty:

```bash
for c in $(jj log -r 'ancestors(@, 20) & ~ancestors(trunk(), 1)' \
  -T 'change_id.short() ++ "\n"' --no-graph); do
  desc=$(jj log -r "$c" --no-graph -T 'description.first_line()')
  url=$(jj log -r "$c" -T 'description' --no-graph \
    | grep 'Pull Request:' | head -1 | sed 's/Pull Request: //')
  echo "$c: $desc"
  if [ -n "$url" ]; then
    echo "  PR: $url"
  else
    echo "  PR: none"
  fi
done
```

### Step 3: Check GitHub PR states

```bash
gh pr list --author @me --state open \
  --json number,title,state,baseRefName,headRefName
```

### Step 4: Present health report

Summarize findings in a table:

| Change | Description | PR | PR State | Base | Issue |
|--------|-------------|-----|----------|------|-------|

Flag these issues:
- **Ghost**: empty change with merged PR
- **Orphan**: empty change with open PR
- **Stale base**: PR targets a synthetic base whose parent was already merged
- **Missing PR**: change has content but no PR URL
- **Stale URL**: change has PR URL but PR is closed
- **Clean**: no issues

End with a summary:
```
Health: N issues found
  - X ghost changes (recommend: jj abandon)
  - Y orphaned PRs (recommend: gh pr close)
  - Z stale bases (recommend: /spr-recover)
```

If issues are found, suggest running `/spr-recover`.
