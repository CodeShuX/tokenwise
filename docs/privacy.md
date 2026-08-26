# Privacy

TokenWise has **zero telemetry**. This document spells out exactly what that means and how you can verify it.

## What TokenWise stores

One file: `.tokenwise/log.ndjson` in your project root. Append-only NDJSON. Every routed task adds one line:

```json
{
  "ts": "2026-05-11T14:32:18Z",
  "session_id": "abc123",
  "task_class": "mechanical",
  "task_summary": "rename getCwd to getCurrentWorkingDirectory (truncated to 80 char...",
  "model_used": "haiku-4-5",
  "model_baseline": "opus-4-7",
  "input_tokens": 12450,
  "output_tokens": 890,
  "cost_actual_usd": 0.0169,
  "cost_baseline_usd": 0.0844,
  "savings_usd": 0.0675,
  "escalated": false,
  "escalation_reason": null,
  "duration_ms": 4210
}
```

That's it. No file contents. No identifiers beyond a per-session UUID. No IP, no user agent, no machine ID.

## What TokenWise does NOT do

- ❌ Send any data over the network
- ❌ Phone home for "anonymous usage statistics"
- ❌ Report crashes or errors to a service
- ❌ Check for updates remotely
- ❌ Submit any data to a leaderboard or public board
- ❌ Read or store API keys, credentials, or session tokens
- ❌ Include file contents, code excerpts, or full task descriptions in the log

The Anthropic API call itself is between you and Anthropic — TokenWise doesn't interpose. Your API key never touches a TokenWise endpoint because there isn't one.

## How to verify

```bash
# Grep the skill source for any network call:
grep -nE 'fetch\(|http(s)?://|axios|curl|wget|got\(' skills/*/SKILL.md

# Result: zero matches (apart from documentation links in markdown).
```

The skill is five markdown files (one per command, under `skills/`). Each contains routing rules and instructions for Claude Code's built-in subagent + log-writing primitives. There's nothing for it to phone home to.

## Task description redaction

When TokenWise logs a task, it truncates the description to **80 characters** and strips:
- Anything matching common credential patterns (`eyJ...`, `sk-...`, `AKIA...`)
- File contents (only file paths are logged, never bodies)
- Stack traces and error messages with absolute paths

If you want to inspect the redaction, check the routing block in your `CLAUDE.md` after install — it's all instructions, no hidden steps.

## Disclosure

If you find a privacy issue, open an issue at https://github.com/CodeShuX/tokenwise/issues — we treat privacy bugs as P0.

## What about Anthropic's own logs?

Anthropic logs API calls server-side for billing and abuse detection. That's outside TokenWise's scope and applies to any Anthropic API use. TokenWise doesn't add to or interact with Anthropic's logging.

## Deleting your data

```bash
rm -rf .tokenwise/
rm -rf <file>.tokenwise-backup-*
```

That's all of it. There's no remote copy.
