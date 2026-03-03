# jj-claude-plugin

A [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code) for [Jujutsu](https://github.com/jj-vcs/jj) — adds a HUD status line showing your change stack, plus optional skills for [jj SPR](https://github.com/LucioFranco/jj-spr) stacked PR workflows.

## Install

```bash
claude --plugin-dir /path/to/jj-claude-plugin
```

## jj HUD Status Line

Shows your current jj change stack in the [claude-hud](https://github.com/jarrodwatts/claude-hud) status line.

```
jj: ↓ tkun add readme → ↓ ptzs add fizz → @ uxmm add config → ↑ svpu (empty)
```

- `↓` changes below `@` (toward trunk)
- `@` current working copy
- `↑` changes above `@` (children)
- Shows change ID (4 chars) and first line of description
- Displays `@ xxxx (on trunk)` when no changes off trunk

### Setup

Requires [claude-hud](https://github.com/jarrodwatts/claude-hud) with `--extra-cmd` support:

```bash
node path/to/claude-hud/dist/index.js --extra-cmd="path/to/jj-claude-plugin/bin/jj-stack-hud"
```

### Configuration

| Env var | Default | Description |
|---------|---------|-------------|
| `JJ_HUD_DEPTH` | `2` | Number of changes above/below `@` to show |
| `JJ_HUD_LAYOUT` | `horizontal` | `horizontal` (one line) or `vertical` (one change per line) |

### Prerequisites (HUD only)

- [jj](https://github.com/jj-vcs/jj) installed and in PATH
- A jj repository (`.jj/` directory)

---

## jj SPR Commands (optional)

Skills and slash commands for the full [jj SPR](https://github.com/LucioFranco/jj-spr) stacked PR lifecycle. These require jj-spr to be installed separately.

### Credits

[jj SPR](https://github.com/LucioFranco/jj-spr) is created by [Lucio Franco](https://github.com/LucioFranco). This plugin provides Claude Code skills for working with jj SPR — it does not bundle or modify the tool itself.

### Commands

- **`/spr-init`** — Set up SPR in a repository
- **`/spr-diff`** — Create or update PRs
- **`/spr-amend`** — Amend a change and update its PR
- **`/spr-land`** — Land a PR and clean up
- **`/spr-reorg`** — Reorganize a stack with existing PRs
- **`/spr-health`** — Diagnose stack health (read-only)
- **`/spr-recover`** — Recover a broken stack (stale PRs, ghost changes)
- **`/spr-status`** — Show change-to-PR mapping
- **`/spr-clean`** — Strip stale PR URLs from commit messages

### Skills

| Skill | Triggers On |
|-------|-------------|
| `jj-spr-init` | "set up spr", "configure jj spr", "init spr" |
| `jj-spr-stacked-prs` | "create PR", "push changes", "spr diff", "stacked PRs" |
| `jj-spr-amend-update` | "update PR", "address review feedback", "amend and push" |
| `jj-spr-landing` | "land PR", "merge PR", "spr land", "ship it" |
| `jj-spr-reorganize` | "reorganize stack", "rebase changes", "squash changes" |
| `jj-spr-recovery` | "fix stack", "recover PRs", "stale PRs", "ghost changes" |

### Prerequisites (SPR)

- [jj](https://github.com/jj-vcs/jj) installed and in PATH
- [jj-spr](https://github.com/LucioFranco/jj-spr) installed and in PATH
- A colocated repository (both `.jj/` and `.git/` directories)
- GitHub personal access token with `repo` scope

### Key Convention

After Claude edits code in a jj change, it always repositions `@`:

```bash
jj new <change-id>   # @ is empty, @- is the modified change
```

This lets you review changes with `jj diff -r @-` before pushing.

## License

MIT
