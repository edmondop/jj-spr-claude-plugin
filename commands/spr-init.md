---
name: spr-init
description: Set up SPR in a Jujutsu repository
---

# /spr-init

Set up this repository to use jj SPR for GitHub pull request management.

## Instructions

**First:** Check if you're in a jj workspace (`.jj/repo` is a file, not a
directory). If so, `cd` to the main colocated repo before proceeding.

Use the `jj-spr-init` skill to guide the setup process.

1. Verify prerequisites (jj, git, colocated repo)
2. Run `jj spr init`
3. Verify configuration
4. Test connectivity with `jj spr list`

Follow the skill's checklist exactly. Report each step's result to the user.
