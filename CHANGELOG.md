# Changelog

All notable changes to TokenWise will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- **Fable 5 as the Planning tier** — routing is now a symmetric 4-way classification by task type: Mechanical→Haiku, Execution→Sonnet, Review→Opus, Planning→Fable ($10/$50 per 1M tokens, 2× Opus). All four lanes route automatically with no per-task confirmation.
- One-time install-time pricing notice for Fable, printed at the end of `/tokenwise:install` — replaces the old per-task `[Y/n]` gate.
- `/tokenwise:ab` now accepts `fable` in `--tiers` (still opt-in, alongside `opus`).

### Changed
- `task_class` values renamed to `mechanical|execution|review|planning`; `model_used` accepts `fable-5`.
- Escalation replaced by **reclassification**: a misrouted subagent never re-routes itself — it returns to the parent, which reclassifies directly to the correct lane (any lane, one hop, at most once). `escalation_reason` values are `needs-mechanical|needs-execution|needs-review|needs-planning|ambiguous-spec|insufficient-capability`. Subagents still never self-escalate; max spawn depth 2 and the >30k-token input bump (which stops at Opus) are retained.
- Planning (Fable) log lines carry negative `savings_usd` by design (baseline stays pinned to Opus) — `/tokenwise:report` and `/tokenwise:summary` render it as-is, not clamped to zero.
- SPEC.md's "Budget cap" scope item (in-scope since v0.1.0, never actually specified) is now explicitly deferred to v0.2; the one-time install notice + per-tier report breakdown is the v0.1 answer.

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
