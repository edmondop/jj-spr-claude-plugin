---
name: spr-diff
description: Create or update PRs for jj changes using SPR
---

# /spr-diff

Create or update GitHub pull requests for jj changes.

## Instructions

**First:** Check if you're in a jj workspace (`.jj/repo` is a file, not a
directory). If so, `cd` to the main colocated repo before proceeding.

Use the `jj-spr-stacked-prs` skill to manage PRs.

1. Run `jj log` to understand the current change stack
2. Follow the pre-flight checklist from the skill
3. Determine the correct range (single change vs. stack)
4. Run `jj spr diff --dry-run -r <range>` to preview what would happen
5. If the dry-run output looks correct, run `jj spr diff -r <range>`
   - **When updating existing PRs**, always pass `-m "reason"` to avoid
     an interactive prompt: `jj spr diff -m "updated" -r <range>`
   - When creating new PRs only, `-m` is not needed (SPR uses the commit
     description as the PR body)
6. Report the created/updated PR URLs

If the user specifies a revision, use it. Otherwise, analyze the stack and
suggest the appropriate range before proceeding.

**Always follow the `@` positioning convention** — if any edits were made,
run `jj new <change-id>` before running `jj spr diff`.
