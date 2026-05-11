# Changelog

All notable changes to TokenWise will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.1.1] — 2026-05-11

### Fixed
- **Plugin structure** — added required `.claude-plugin/plugin.json` manifest. Without it, Claude Code did not register TokenWise as a plugin contributing skills, and slash commands were silently unavailable.
- **Skills directory layout** — moved from single `skill/SKILL.md` to canonical `skills/<name>/SKILL.md` per-skill structure required by Claude Code's plugin loader.
- **Slash command form** — commands are now correctly namespaced: `/tokenwise:install`, `/tokenwise:report`, `/tokenwise:summary`, `/tokenwise:ab`, `/tokenwise:undo`. Previous docs incorrectly described `/tokenwise install` (space-separated, which Claude Code parses as a skill called `tokenwise` with arguments `install`).
- **Frontmatter** — dropped non-canonical `name:` and `user_invocable:` fields; kept canonical `description:`.

### Changed
- Each subcommand is now its own skill file (`skills/install/`, `skills/report/`, `skills/summary/`, `skills/ab/`, `skills/undo/`) so each is independently discoverable via `/help`.

## [0.1.0] — 2026-05-11

### Added
- Initial public release.
- `/tokenwise install` — guided + manual config installer with diff preview, backup, and undo.
- `/tokenwise report` — current session routing report with $ saved vs all-Opus baseline.
- `/tokenwise summary [--week|--month|--all]` — historical routing aggregates.
- `/tokenwise ab "<task>"` — A/B test the same task at multiple tiers, score quality + cost.
- `/tokenwise undo` — restore CLAUDE.md / settings.json from TokenWise backups.
- Routing taxonomy: Haiku (mechanical) / Sonnet (scoped reasoning) / Opus (synthesis).
- Safety caps: Haiku never spawns subagents; max depth 2; trivial-task floor.
- NDJSON measurement log at `.tokenwise/log.ndjson` (zero telemetry, local only).
- Phase-1 probes for `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` and subagent `model:` param (Anthropic Issues #36381, #47488).
- Prompt-cache hint detector + context-window watcher + budget cap.

### Known limitations
- Token counts approximate to ±2% vs Anthropic billing.
- A/B test mode costs extra tokens by design (one task × N tiers).
- No multi-vendor routing — Anthropic-only.
- No bulk async (use Anthropic Batch API).
