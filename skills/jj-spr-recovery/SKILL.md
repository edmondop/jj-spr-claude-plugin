---
name: jj-spr-recovery
description: Use when a jj+SPR stack is in a bad state after out-of-order merges, GitHub web merges, or stale synthetic bases. Triggers on "fix stack", "recover PRs", "stale PRs", "ghost changes", "PRs show wrong diff", "stack is broken", or when diagnosis reveals stale/orphaned PRs.
---

# jj SPR Stack Recovery

## Workspace Detection (MUST check first)

```bash
if [ -f .jj/repo ]; then
  MAIN_REPO=$(dirname "$(dirname "$(cat .jj/repo)")")
  cd "$MAIN_REPO"
fi
```

## Overview

Recover a jj+SPR stack that is in a bad state. Common causes:

- PRs merged on GitHub web UI instead of `jj spr land`
- PRs merged out of order (not bottom-up)
- Parent PRs landed but remaining stack not refreshed
- Orphaned PRs for changes that are now empty

This skill diagnoses the problem and performs a full recovery:
close stale PRs, abandon ghost changes, strip URLs, rebase, recreate.

## Step 1: Diagnose

### Map changes to PRs

```bash
for c in $(jj log -r 'ancestors(@, 20) & ~ancestors(trunk(), 1)' \
  -T 'change_id.short() ++ "\n"' --no-graph); do
  desc=$(jj log -r "$c" --no-graph -T 'description.first_line()')
  url=$(jj log -r "$c" -T 'description' --no-graph \
    | grep 'Pull Request:' | head -1 | sed 's/Pull Request: //')
  empty=""
  if jj log -r "$c" --no-graph -T 'description' 2>/dev/null | head -1 \
    | grep -q '(empty)'; then
    empty=" [EMPTY]"
  fi
  echo "$c: $desc$empty"
  echo "  PR: ${url:-none}"
done
```

### Check PR states on GitHub

```bash
gh pr list --author @me --state all --limit 20 \
  --json number,title,state,baseRefName,headRefName
```

### Identify problems

For each open PR, classify it:

| PR State | Local Change | Problem |
|----------|-------------|---------|
| OPEN | empty | Orphaned — close it |
| OPEN | has content, targets synthetic base | Possibly stale — check if base content is current |
| OPEN | has content, targets master | May be fine, or may include parent content in diff |
| MERGED | empty | Ghost — abandon local change |
| MERGED | has content | Content landed via different path — rebase should make it empty |
| CLOSED | any | Already handled — just strip URL if present |

### Present diagnosis to user

Before taking any action, present a summary:

```
Stack diagnosis:
  - N empty ghost changes to abandon
  - M orphaned PRs to close
  - K PRs with stale synthetic bases to recreate
  - Total changes with content: X

Proposed recovery:
  1. Close PRs: #A, #B, #C
  2. Abandon empty changes: id1, id2
  3. Strip URLs from: id3, id4, id5
  4. Rebase stack onto main@origin
  5. Recreate PRs with jj spr diff
```

**Get user confirmation before proceeding.**

## Step 2: Close Stale/Orphaned PRs

**Prefer `jj spr close`** — it closes the PR, strips the `Pull Request:`
URL from the commit message, and deletes both head and synthetic base
branches:

```bash
# Close a single PR (also strips URL and deletes branches)
jj spr close -r <change>

# Close a range
jj spr close -r <bottom>::<top>
```

If `jj spr close` isn't available (sandboxed, SPR broken), fall back to:

```bash
gh pr close <number> --delete-branch
```

Then you must manually strip URLs in Step 4.

Close PRs in this order:
1. Orphaned PRs (empty local change, PR is open)
2. PRs targeting stale synthetic bases that will be recreated

## Step 3: Abandon Ghost Changes

Ghost changes are `(empty)` changes whose content was already merged to
master. They clutter the graph and serve no purpose.

```bash
jj abandon <empty-change-id>
```

**Verify the change is truly empty before abandoning.** Check with
`jj diff -r <id>` — an empty diff confirms it's a ghost.

## Step 4: Strip Pull Request URLs (skip if Step 2 used `jj spr close`)

`jj spr close` already strips URLs. This step is only needed if PRs were
closed via `gh pr close` or the GitHub UI.

Remove stale `Pull Request:` lines from all changes that will get new PRs:

```bash
for c in <change-ids>; do
  desc=$(jj log --no-graph -r "$c" -T 'description')
  clean=$(echo "$desc" | grep -v 'Pull Request:' | sed '/^$/N;/^\n$/d')
  jj desc -r "$c" -m "$clean"
done
```

**Verify:** `jj log` should show no `Pull Request:` lines in the stack.

## Step 5: Fetch and Rebase

```bash
jj git fetch
jj rebase -s <bottom-of-stack> -d main@origin
```

Use `-s` (not `-r`) to rebase the entire subtree.

**After rebase:** Some changes may become empty if their content was already
on master. Abandon those too.

## Step 6: Recreate PRs

```bash
jj spr diff --dry-run -r <bottom>::<top>
```

**Always dry-run first.** Verify:
- Correct number of PRs would be created
- No unexpected immutable commits in the range
- Range starts from the first mutable change above trunk

If dry-run looks good:

```bash
jj spr diff -r <bottom>::<top>
```

## Step 7: Verify

```bash
# Local graph is clean
jj log

# PRs are created with correct bases
jj spr list

# Check GitHub
gh pr list --author @me --state open \
  --json number,title,baseRefName
```

## Step 8: Reposition @

```bash
jj new <top-of-stack>
```

## When to Use This vs. Other Skills

| Situation | Use This? |
|-----------|-----------|
| Stack is fine, just want to land a PR | No — use `jj-spr-landing` |
| Want to reorganize a healthy stack | No — use `jj-spr-reorganize` |
| PRs show wrong diffs after parent merge | **Yes** |
| Empty ghost changes in the graph | **Yes** |
| Someone merged on GitHub web UI | **Yes** |
| PRs target stale synthetic bases | **Yes** |
| `jj spr diff` errors about closed PRs | **Yes** — strip stale URLs first |

## Common Mistakes

- **Not getting user confirmation before closing PRs** — always present
  diagnosis and get approval
- **Abandoning changes that aren't empty** — verify with `jj diff` first
- **Forgetting `jj git fetch` before rebase** — rebase onto stale master
  leaves the stack in the same bad state
- **Skipping the dry-run** — `jj spr diff --dry-run` catches range errors
  before creating duplicate PRs
- **Leaving stale `Pull Request:` URLs** — even one causes SPR to update
  a closed PR instead of creating a new one
