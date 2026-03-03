---
name: jj-spr-stacked-prs
description: Use when creating, updating, or managing stacked PRs in a jj repository. Triggers on "create PR", "push changes", "spr diff", "open PR", "stacked PRs", or after making changes that need to go to GitHub.
---

# jj SPR Stacked PRs

## Workspace Detection (MUST check first)

**Before running ANY SPR command, check if you're in a jj workspace.**

A jj workspace has `.jj/` but NO `.git/`. SPR requires `.git/` and will
fail silently or with confusing errors in a workspace.

```bash
# Check: is .jj/repo a FILE (workspace) or a DIRECTORY (main repo)?
if [ -f .jj/repo ]; then
  echo "IN WORKSPACE — must switch to main repo"
  MAIN_REPO=$(dirname "$(dirname "$(cat .jj/repo)")")
  cd "$MAIN_REPO"
fi
```

**All SPR commands must run from the main colocated repo, not a workspace.**
Do NOT try to pass `--repository` or any flag to SPR — it doesn't support
remote repo paths. You must `cd` to the main repo.

## How SPR Works (must understand before using)

SPR pushes **synthetic commits** to GitHub, never the real jj commits. Each
PR branch gets a synthetic commit with the same tree content but different
parents (forming an append-only history for clean review diffs).

For stacked PRs, SPR creates a **base branch** (`spr/.../master.<title>`)
with a synthetic commit whose tree matches the parent commit's tree. This
makes GitHub show only each PR's own changes in the diff.

**The `Pull Request:` URL in the commit message is SPR's ONLY way to track
which local commit maps to which PR.** There is no cache, config, or other
mechanism. If this URL is missing or stale, SPR loses track.

## CRITICAL: Never Wipe the `Pull Request:` URL

**`jj describe -m "..."` replaces the ENTIRE commit message.** If the
commit already has a `Pull Request: https://...` line from a previous
`jj spr diff`, using `-m` blindly will delete it. SPR will then create a
DUPLICATE PR on the next `jj spr diff` run.

**Before ANY `jj describe` on a change that may have a PR:**

```bash
# 1. Read the current description
jj log -r <change-id> --no-graph -T 'description'

# 2. If it has a "Pull Request:" line, PRESERVE it at the bottom
jj describe -r <change-id> -m 'New summary here

Pull Request: https://github.com/org/repo/pull/12345'
```

**Safe alternatives:**
- `jj spr amend` — updates the commit message from the GitHub PR body
  (never wipes the URL)
- Edit only the summary while preserving the rest:
  ```bash
  CURRENT=$(jj log -r <change-id> --no-graph -T 'description')
  # Modify $CURRENT keeping the Pull Request: line intact
  jj describe -r <change-id> -m "$MODIFIED"
  ```

**This is the #1 cause of duplicate PRs.** Treat the `Pull Request:` line
as sacred — read before writing, always.

## How SPR Handles Immutable Commits

SPR's workflow is two phases:
1. **Push branch + create PR on GitHub** (irreversible)
2. **`jj describe` to write PR URL back into commit message** (can fail)

If a commit is immutable, Phase 1 succeeds but Phase 2 fails. The PR exists
on GitHub but SPR can't write the URL back. This means:
- The PR is created and correct — no action needed on GitHub
- SPR has no local record of it — running `jj spr diff` again would create
  a DUPLICATE PR
- **Never include an immutable commit in SPR ranges after its PR exists**

## The `@` Positioning Convention

**After any code edits to a jj change, always reposition `@`:**

```bash
jj new <change-id>   # @ is now an empty change, @- is the modified change
```

Never leave `@` pointing at a PR change after edits. The user needs `@-` to
point at the modified change so they can review with `jj diff -r @-`.

## Pre-flight Checklist

Before running any SPR command:

1. **Strip stale PR URLs** — if you reorganized the stack (rebase/squash),
   every commit may have `Pull Request: https://...` lines from old PRs.
   Run `jj desc <change-id> -m "<clean message>"` for each affected commit.

2. **Identify immutable commits** — look for `◆` in `jj log`. If an
   immutable commit already has its PR, exclude it from the range.

3. **Determine the range** — never use `jj spr diff --all` blindly. Always
   use an explicit range:
   ```bash
   jj spr diff -r <first-change>::<last-change>
   ```

4. **Ensure `@` is on the stack** — SPR operates relative to `@`. If `@`
   is an empty side change, move it first:
   ```bash
   jj new <top-of-stack>
   ```

5. **Run SPR from colocated workspace** — SPR needs a `.git` directory.

## CRITICAL: First PR in Range Always Targets Main

**SPR determines the base branch mechanically:** the first commit in the
range ALWAYS targets `main`/`master`. SPR does NOT look at existing PRs
below the range to chain off them.

This means: if there is a parent change below your range that already has
a PR, and you exclude it from the range, the first new PR will target
`main` instead of chaining off the parent. **The diff on GitHub will
include ALL changes from the parent — not just the new commit's changes.**

**Fix:** Always include the parent change (with its existing `Pull Request:`
URL) in the range. SPR will see the URL, update that PR instead of creating
a new one, and chain subsequent PRs off it.

```bash
# WRONG — parent PR exists but is excluded, first new PR targets main
jj spr diff -r <new-change-1>::<new-change-N>

# RIGHT — include parent so SPR chains off its existing PR
jj spr diff -r <parent-with-pr>::<new-change-N>
```

**When in doubt, use `jj spr diff --dry-run -r <range>` to preview what
SPR would do without pushing or creating anything.**

## Decision Tree

### Single Change (one PR)
```bash
jj spr diff -r <change-id>
```

### Stack of Changes (multiple PRs)
```bash
jj spr diff -r <first-mutable-change>::<last-change>
```

SPR creates one PR per commit with automatic base branch chaining.

### Adding to an Existing Stack
```bash
# Include the last change that already has a PR
jj spr diff -r <existing-pr-change>::<new-last-change>
```

### Never
```bash
# DON'T — includes everything back to trunk, possibly immutable commits
jj spr diff --all
```

## Creating Stacked PRs

```bash
# From colocated workspace, with @ on top of stack:
jj spr diff -r <first-mutable-change>::<last-change>
```

SPR creates one PR per commit with automatic base branch chaining via
synthetic base branches. Do NOT manually change PR base branches — SPR's
synthetic commits handle the diffs correctly.

## Common Mistakes

- **Never use `jj git push` + `gh pr create`** for PR workflows. Always
  use SPR. Manual PRs don't share ancestry with SPR's synthetic commits,
  so GitHub diffs will be wrong.
- **Never merge on GitHub directly** (click "Merge" or `gh pr merge`).
  Always use `jj spr land`. For stacked PRs, a direct GitHub merge sends
  the code to the synthetic base branch, not master.
- **Never manually change a PR's base branch** away from SPR's managed
  base branches. Don't use `gh pr edit --base master` -- SPR retargets
  to master automatically at land time.
- **Never use `jj spr diff --all`** without verifying no immutable commits
  with existing PRs are in the ancestry path.
- **Never forget to clean commit messages** after closing old PRs and
  before running SPR again.
- **Never include an immutable commit in an SPR range twice** — it will
  create duplicate PRs since SPR can't write the URL back.
- **Never use `jj describe -m "..."` without preserving the `Pull Request:`
  URL** — this is the #1 cause of duplicate PRs. Always read the current
  description first and include the `Pull Request:` line in the new message.
