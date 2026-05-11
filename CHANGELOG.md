# Changelog

All notable changes to TokenWise will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

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
