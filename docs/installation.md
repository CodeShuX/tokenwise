# Installation

## Prerequisites

- [Claude Code](https://claude.com/claude-code) installed (verified on Claude Code v2.x)
- A working session — open Claude Code in any project

## One-line install (recommended)

```
/plugin marketplace add CodeShuX/tokenwise
/plugin install tokenwise@tokenwise
```

That registers TokenWise as a Claude Code skill. To activate it, run:

```
/tokenwise:install
```

TokenWise will scan your config, run health probes, show diffs, and ask for confirmation before modifying any file.

## Manual install (no marketplace)

```bash
git clone https://github.com/CodeShuX/tokenwise.git ~/tokenwise
mkdir -p ~/.claude/skills
ln -s ~/tokenwise/skill/SKILL.md ~/.claude/skills/tokenwise.md
```

Restart Claude Code if it was running.

## What `/tokenwise:install` does

1. Reads `~/.claude/CLAUDE.md`, project `./CLAUDE.md`, `~/.claude/settings.json`
2. Runs `claude --version` to record build identifier
3. Probes `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` env var (Anthropic Issue #36381 has broken this in past versions)
4. Spawns a Haiku probe subagent to verify the `model:` param is honored (Anthropic Issue #47488)
5. Inventories existing skills/hooks for conflict detection
6. Prints a detection report
7. Proposes config changes, one file at a time
8. For each: shows diff, asks `[Y/n]`, backs up original, writes change
9. Verifies writes by re-reading

If any probe fails, install refuses and prints the affected Anthropic issue.

## Modes

| Flag | Behavior |
|---|---|
| (default) | Auto-detects power-user signals and picks guided or manual |
| `--guided` | Force guided mode (shows diffs, asks Y/n per file) |
| `--manual` | Print copy-paste blocks, exit without writing |
| `--dry-run` | Show all proposed changes without writing or printing copy-paste |

## Verifying install

```
/tokenwise:report
```

Should print "No routed tasks yet" if you just installed. After your next routed task, it'll show real numbers.

```
cat ~/.claude/CLAUDE.md | grep -A 2 "BEGIN TokenWise"
```

Should show the routing block.

## Uninstall

```
/tokenwise:undo
```

Lists backups, lets you restore. Then:

```
/plugin uninstall tokenwise@tokenwise
/plugin marketplace remove tokenwise
```

Cleans up the plugin registration.

## Common issues

**"Routing probe failed."** Your Claude Code build has a known routing bug. Check `claude --version` against Anthropic Issues #47488 and #36381. Either update Claude Code or run `/tokenwise:install --manual` to skip the env-var addition.

**"Could not find CLAUDE.md."** TokenWise creates a project `./CLAUDE.md` if neither global nor project exists. You'll be prompted to confirm.

**"Permission denied writing settings.json."** Your `~/.claude/settings.json` is read-only or owned by another user. Fix permissions or use `--manual`.
