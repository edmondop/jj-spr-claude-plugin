# jj-hud: Jujutsu Status Line for Claude Code

## Summary

Add jj awareness to the Claude Code status line via two components:
a forked claude-hud with jj support, and a new `spr status --json`
subcommand in jj-spr for PR data.

## Architecture

Three components, clear separation of concerns:

```
jj (native)              jj-spr (Rust)              jj-hud (Node, claude-hud fork)
┌──────────┐            ┌──────────────┐            ┌─────────────────────────┐
│ jj log   │── JSON ──▶ │              │            │ Basic mode:             │
│ template │            │ spr status   │── JSON ──▶ │   calls jj directly     │
└──────────┘            │   --json     │            │ Full mode:              │
                        └──────────────┘            │   calls spr status      │
                                                    └─────────────────────────┘
```

- **Basic mode (jj only):** HUD calls `jj log` with a JSON template.
  Works in any jj repo. Shows change stack, no PR info.
- **Full mode (jj + jj-spr):** HUD calls `spr status --json --at @ --span N`.
  Returns stack + PR data. Only works if jj-spr is installed.

## Two Display Modes (toggleable)

### Changes only (default)
```
jj: ↓ fix-auth → ↓ add-tests → @ current-work → ↑ refactor-api
```

### Changes + PRs (full mode, requires jj-spr)
```
jj: ↓ fix-auth [#42✓] → ↓ add-tests [#43○] → @ current-work → ↑ refactor-api
```

## Toggle Mechanism

- Config sets the default mode (`changes` or `full`)
- Slash command `/jj-hud-toggle` cycles modes within a session
- State stored in `~/.claude/plugins/jj-hud/state.json`
- Resets to config default on next session

## Layout

- **Horizontal** (default): one-line chain with arrows
- **Vertical**: multi-line, reads like jj log
- Configurable in `~/.claude/plugins/jj-hud/config.json`

## Stack Depth

- Configurable: how many changes above/below @ to show
- Default: 2
- Passed as `--span` arg to jj-spr, or used to slice jj log output

## JSON Contract: `spr status --json`

Input args: `--at <revision>` (default @), `--span <N>` (default 2)

Output (flat change list, bottom-to-top):
```json
{
  "stack": [
    {
      "change_id": "abc1",
      "description": "fix auth flow",
      "pr": { "number": 42, "url": "https://github.com/org/repo/pull/42" },
      "is_current": false
    },
    {
      "change_id": "def2",
      "description": "add test coverage",
      "pr": null,
      "is_current": true
    },
    {
      "change_id": "ghi3",
      "description": "refactor API",
      "pr": { "number": 43, "url": "https://github.com/org/repo/pull/43" },
      "is_current": false
    }
  ]
}
```

## jj-only JSON (via jj template)

For basic mode, HUD runs jj directly:
```
jj log --no-graph -r 'ancestors(@, N) | descendants(@, N) & ~ancestors(trunk(), 1)' \
  -T '{"change_id": " ++ change_id.short().escape_json() ++ ", ...}'
```

## Detection

On each status line tick:
1. Check for `.jj/` in cwd → if absent, skip jj line entirely
2. If present and mode is `full`, try `spr status --json`
3. If spr not available, fall back to basic jj mode

## Repos

| Repo | What | Action |
|------|------|--------|
| `jj-spr` (Rust fork) | Add `spr status --json` subcommand | Extend |
| `jj-hud` (new repo) | Fork of claude-hud with jj line | Create |
| `jj-spr-claude-plugin` → `jj-claude-plugin` | Rename, organize skills | Rename |

## Plugin Organization (after rename to jj-claude-plugin)

Skills and commands split into two categories:

**jj (works without jj-spr):**
- jj-hud status line (basic mode)

**jj-spr (requires jj-spr):**
- spr-diff, spr-amend, spr-land, spr-reorg, spr-clean, spr-recover
- jj-hud full mode (PR info)
- spr-status, spr-init, spr-health

## Execution Steps

1. Fork claude-hud into new `jj-hud` repo
2. Add jj detection + jj line rendering to the fork
3. Add `spr status --json` subcommand to jj-spr Rust fork
4. Wire full mode: jj-hud calls spr for PR data
5. Add toggle slash command + state file
6. Add config options (depth, layout, default mode)
7. Rename jj-spr-claude-plugin → jj-claude-plugin on GitHub
8. Reorganize skills/commands with clear jj vs jj-spr labeling
9. Update settings.json: disable old claude-hud, enable jj-hud
