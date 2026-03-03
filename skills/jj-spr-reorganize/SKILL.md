---
name: jj-spr-reorganize
description: Use when reorganizing a jj change stack that has existing PRs. Triggers on "reorganize stack", "rebase changes", "squash changes", "split change", "reorder PRs", "remove PR from stack", or after restructuring a stack that had open PRs.
---

# jj SPR Stack Reorganization

## Workspace Detection (MUST check first)

**Before running ANY SPR command, check if you're in a jj workspace.**

```bash
if [ -f .jj/repo ]; then
  MAIN_REPO=$(dirname "$(dirname "$(cat .jj/repo)")")
  cd "$MAIN_REPO"
fi
```

SPR requires `.git/` which only exists in the main colocated repo, not in
workspaces. All `jj spr` commands must run from the main repo.

## How SPR Handles Reorganization

SPR tracks PRs via `Pull Request:` URLs in commit messages. When you run
`jj spr diff -r <range>`:

- Changes **with** a `Pull Request:` URL → SPR **updates** the existing PR
- Changes **without** a URL → SPR **creates** a new PR

This means you do NOT need to close all PRs when reorganizing. You only
close the PRs for changes that are being removed or fundamentally altered.
Changes that keep their `Pull Request:` URL will have their PRs updated
in place.

`jj spr close -r <change>` handles cleanup for a single PR:
1. Closes the PR on GitHub
2. Strips `Pull Request:` and `Reviewed By:` from the commit description
3. Deletes both the head branch and synthetic base branch

## Decision Tree

### Removing a change from the stack (squash into another)

Only the removed change's PR needs to close. All other PRs stay.

```bash
# 1. Close just the PR being removed
jj spr close -r <change-to-remove>

# 2. Squash it into its target
jj squash --from <change-to-remove> --into <target>

# 3. Update remaining PRs (pass -m to avoid interactive prompt)
jj spr diff -m "squashed change" -r <bottom>::<top>

# 4. Reposition @
jj new <top-of-stack>
```

### Splitting a change into two

The original PR stays on one half. The new half gets a new PR.

```bash
# 1. Split the change (interactive file selection)
jj split -r <change>

# 2. Update the stack — SPR updates the PR on the half that kept
#    the URL, creates a new PR for the half without one
jj spr diff -m "split change" -r <bottom>::<top>
```

### Reordering changes

Close PRs for changes whose position changed, then update the stack.

```bash
# 1. Close PRs for changes being moved
jj spr close -r <change-being-moved>

# 2. Reorder
jj rebase -r <change> --after <target>

# 3. Update stack — moved change gets a new PR, others are updated
jj spr diff -m "reordered stack" -r <bottom>::<top>
```

### Full reorganization (everything changes)

When the stack structure changes fundamentally (e.g., rewriting history),
close all PRs and recreate.

```bash
# 1. Close all PRs in the range
jj spr close -r <bottom>::<top>

# 2. Reorganize freely
jj rebase / squash / split / etc.

# 3. Verify the new stack
jj log

# 4. Recreate PRs
jj spr diff -r <new-bottom>::<new-top>

# 5. Reposition @
jj new <top-of-stack>
```

## Important: Always Pass `-m` When Updating

When `jj spr diff` updates an existing PR (not creating), it prompts
interactively for an update message if `-m` is not provided. **Always
pass `-m`** to avoid blocking on a prompt:

```bash
jj spr diff -m "reorganized stack" -r <range>
```

## Verify Before Updating

Always dry-run before pushing:

```bash
jj spr diff --dry-run -r <bottom>::<top>
```

Check that:
- The correct number of PRs would be created/updated
- Changes with existing URLs show "UPDATE" (not "CREATE")
- Changes without URLs show "CREATE"

## Common Mistakes

- **Closing all PRs when only one is being removed** — wasteful and loses
  review history. Use `jj spr close -r <single-change>` instead.
- **Forgetting `-m` on `jj spr diff`** — causes an interactive prompt that
  blocks automation. Always pass `-m "reason"`.
- **Not verifying the stack shape before updating** — run `jj log` and
  `jj spr diff --dry-run` first.
- **Leaving `@` on a PR change** — always `jj new` after reorganization.
- **Using `gh pr close` instead of `jj spr close`** — `gh pr close` does
  NOT strip the `Pull Request:` URL from the commit description or delete
  the synthetic base branch. Always use `jj spr close`.
