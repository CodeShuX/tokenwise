---
name: tokenwise
description: Measurement-driven model router for Claude Code. Routes work to Haiku/Sonnet, reserves Opus for synthesis, logs every routed task with real token/cost numbers, and proves savings via on-demand A/B tests. Use when the user wants to reduce Claude Code token spend without sacrificing output quality, or asks "how much am I saving by using Haiku?", "is Sonnet good enough for this?", or "what's the cheapest way to run this task?".
user_invocable: true
---

# TokenWise — Measurement-Driven Model Router

> Cut Claude Code token spend without sacrificing quality — and prove it.

---

## When to invoke

- User wants to install or configure TokenWise (`/tokenwise install`)
- User wants to see savings (`/tokenwise report`, `/tokenwise summary`)
- User wants to A/B test a task at different tiers (`/tokenwise ab "<task>"`)
- User wants to undo TokenWise config changes (`/tokenwise undo`)
- User asks a question the router answers: "is Haiku enough for X?", "how much did I save today?", "why did this task escalate to Opus?"

## When NOT to invoke

- Generic "make my code cheaper" — that's optimization, not routing. Refuse.
- Multi-vendor routing (LiteLLM/OpenRouter) — out of scope, refer them elsewhere.
- Bulk async batch processing — that's the Anthropic Batch API, not TokenWise.

---

## Subcommands

### `/tokenwise install [--guided | --manual] [--dry-run]`

Configures the user's Claude Code to route subtasks through TokenWise. Two modes:

- **Guided (default for clean configs):** shows diff, backs up originals, writes with confirmation
- **Manual (default for power users):** prints copy-paste blocks, exits without writing

`--dry-run` shows all proposed changes without writing — works in both modes.

### `/tokenwise report`

Prints the current session's routing report: tokens per model, $ saved vs all-Opus baseline, escalations, quality flags.

### `/tokenwise summary [--week | --month | --all]`

Aggregates `.tokenwise/log.ndjson` over a window. Default: `--week`.

### `/tokenwise ab "<task description>" [--tiers haiku,sonnet,opus]`

Runs the same task on multiple tiers, captures outputs, diffs them, scores quality. Writes `tokenwise-ab-<timestamp>.md` in the project root.

### `/tokenwise undo`

Lists `.tokenwise-backup-*` files in `~/.claude/` and project root. User picks one; TokenWise restores it.

---

## The five phases

### Phase 1 — Detect

When `/tokenwise install` runs:

1. **Read user's Claude Code config:**
   - `~/.claude/CLAUDE.md` (global)
   - `./CLAUDE.md` (project, if any)
   - `~/.claude/settings.json`
2. **Run `claude --version`** — record for compatibility warnings
3. **Test env vars** — set `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` in a probe shell, verify it's honored (Anthropic Issue #36381 has broken this in past versions)
4. **Test subagent routing** — spawn a probe Haiku subagent, verify the model param wasn't silently overridden (Issue #47488 regression check)
5. **Inventory skills/hooks** — list existing skills to detect conflicts
6. **Print detection report** to user — no writes yet

**If routing probe fails, refuse to install.** Tell the user their Claude Code build has a known routing bug and link the issue.

### Phase 2 — Configure

After detection, propose changes:

**For CLAUDE.md (global or project, user picks):**

```markdown
<!-- BEGIN TokenWise — routing rules. Managed by /tokenwise. Do not edit by hand. -->
## Model routing (TokenWise)

When the user asks for work, delegate to the cheapest model that can handle it:

- **Haiku** (cheapest, 5× cheaper than Opus) — mechanical bulk: file reads, grep,
  format, rename, simple edits, doc lookups. No judgment calls.
- **Sonnet** (~1.67× cheaper than Opus) — scoped reasoning: single-file refactor,
  test writing, scoped research, code exploration, bug-fix in known file.
- **Opus** — synthesis only: architecture decisions, multi-file refactor synthesis,
  security review, ambiguous requirements, cross-cutting bug RCA.

Safety caps:
- Haiku never spawns further subagents. If a Haiku task wants to delegate, it
  was wrong-sized — return to parent for re-classification.
- Max spawn depth = 2 (parent → subagent → one more tier).
- If a subagent realizes it needs a smarter model, it returns to parent —
  it does not escalate on its own.
- If task description <100 chars AND no file context, run inline. Subagent
  overhead exceeds savings on trivial tasks.
- If subagent input would exceed 30k tokens, bump up a tier
  (Haiku → Sonnet, Sonnet → Opus).

After every routed task, append one NDJSON line to `.tokenwise/log.ndjson`:
{"ts": "...", "task_class": "...", "model_used": "...",
 "input_tokens": N, "output_tokens": N,
 "cost_actual_usd": N, "cost_baseline_usd": N, "savings_usd": N,
 "escalated": bool, "duration_ms": N}

Pricing (May 2026, per 1M tokens, input/output):
- Opus 4.7:    $5 / $25
- Sonnet 4.6:  $3 / $15
- Haiku 4.5:   $1 / $5

When the user invokes `/tokenwise report`, `summary`, or `ab`, hand off to the
TokenWise skill.
<!-- END TokenWise -->
```

**For settings.json (optional, only if Phase 1 detected the env vars work):**

```json
{
  "env": {
    "CLAUDE_CODE_DISABLE_1M_CONTEXT": "1",
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "80"
  }
}
```

**Process:**
1. Show full diff per file
2. Ask `[Y/n]` per file
3. On Y: write `<file>.tokenwise-backup-<YYYYMMDD-HHMMSS>`, then write the modification
4. On n: skip that file, continue
5. After all writes: re-read both files, confirm changes are present, print "Install complete."

**Manual mode:** skip steps 2-5. Print the same blocks as fenced code blocks with copy-paste instructions. Exit.

### Phase 3 — Measure (runtime, automatic)

Every routed subagent spawn writes one NDJSON line to `.tokenwise/log.ndjson`. Schema:

```json
{
  "ts": "2026-05-11T14:32:18Z",
  "session_id": "abc123",
  "task_class": "mechanical|scoped|synthesis",
  "task_summary": "<first 80 chars of task description, no PII>",
  "model_used": "haiku-4-5|sonnet-4-6|opus-4-7",
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

**Cost calculation (May 2026 pricing):**

| Model | Input $/1M | Output $/1M |
|---|---|---|
| Opus 4.7 | 5.00 | 25.00 |
| Sonnet 4.6 | 3.00 | 15.00 |
| Haiku 4.5 | 1.00 | 5.00 |

`cost = (input_tokens / 1_000_000) * input_price + (output_tokens / 1_000_000) * output_price`

`baseline_cost` is what the same tokens would have cost at Opus rates.

`savings = baseline_cost - actual_cost`

### Phase 4 — A/B test

`/tokenwise ab "<task>" [--tiers haiku,sonnet,opus]`:

1. Parse task description
2. For each tier in `--tiers` (default: haiku,sonnet):
   - Spawn a subagent at that tier with identical prompt
   - Capture stdout, token usage, duration, errors
3. Compute pairwise diffs (structural: line count, token overlap; semantic: ask Opus to score each output 1-10 on completeness + correctness)
4. Write report to `tokenwise-ab-<timestamp>.md`:

```markdown
# A/B Test — <task summary>
Run: 2026-05-11 14:32:18

## Tier comparison

| Tier | Tokens (in/out) | Cost | Duration | Quality (1-10) |
|---|---|---|---|---|
| Haiku | 12.4k / 890 | $0.017 | 4.2s | 8/10 |
| Sonnet | 11.8k / 1.2k | $0.053 | 6.8s | 9/10 |

## Recommendation

For this task class, **Haiku is sufficient** (quality 8/10, 68% cheaper).

## Output diffs

<full outputs follow>
```

### Phase 5 — Report

`/tokenwise report` — reads `.tokenwise/log.ndjson` filtered to current session_id:

```
TokenWise Session Report
========================

Tasks routed:        47
Duration:            2h 14m

Per model:
  Haiku    32 tasks   1.2M input  /  84K output   →  $1.62
  Sonnet   12 tasks   480K input  /  41K output   →  $2.06
  Opus      3 tasks   145K input  /  28K output   →  $1.43

Total spent:         $5.11
Baseline (all-Opus): $24.93
Savings:             $19.82  (79.5%)

Quality flags:
  Escalations:        2 (Haiku → Sonnet, mid-task)
  User overrides:     0
```

`/tokenwise summary --week`:
```
TokenWise Summary — last 7 days
================================

Sessions: 18
Tasks routed: 612
Total spent: $34.21
Baseline: $189.40
Savings: $155.19 (81.9%)

Top task classes:
  code-search          287 tasks   $4.12
  file-read            142 tasks   $1.80
  scoped-research       89 tasks   $11.40
  synthesis             34 tasks   $15.20
  ...
```

`/tokenwise undo` — interactive:
```
TokenWise backups found:

  1. ~/.claude/CLAUDE.md.tokenwise-backup-20260511-143218
  2. ~/.claude/settings.json.tokenwise-backup-20260511-143219

Restore which? (1-2, or 'all', or 'cancel'):
```

---

## Real-time features (always-on after install)

### Cost ticker

Print a one-line cost summary at the end of every routed task in dev mode (toggle: `TOKENWISE_TICKER=1` in env):

```
[TokenWise] task=code-search model=haiku cost=$0.017 saved=$0.07 (79.5%)
```

### Budget cap

Set in CLAUDE.md routing block:
```yaml
tokenwise:
  budget_session_usd: 10.00
  budget_alert_at_pct: 80
```

When 80% reached: print warning. When 100%: refuse to route, ask user to confirm continued spending.

### Prompt-cache hint detector

When the same file is read >1 time in a session and total content >1024 tokens, flag it: `[TokenWise] cache-eligible: <file> read 3× (4.2k tokens). See docs/cache-hints.md`.

### Context-window watcher

At 70%, 85%, 95% of primary thread context: print warning. At 95%, suggest spawning a `report`-only subagent to summarize before auto-compact triggers.

---

## Inputs (from `/tokenwise install`)

| Input | When asked | Default |
|---|---|---|
| Mode | Always | Detected (guided/manual) |
| Which CLAUDE.md to modify | If both ~/.claude/ and ./ exist | Project (./) |
| Modify settings.json? | If env-var probe passes | Yes |
| Budget cap (session $) | Always | None (off) |
| Cost ticker on/off | Always | Off |

All inputs accept `cancel` to abort with no writes.

---

## Outputs

- **From `install`:** modified CLAUDE.md, modified settings.json, backup files, detection report
- **From routing (automatic):** appended lines in `.tokenwise/log.ndjson`
- **From `report`:** stdout text (no file)
- **From `summary`:** stdout text + optional `--out <path>` to save markdown
- **From `ab`:** `tokenwise-ab-<timestamp>.md` in project root
- **From `undo`:** restored files, removed backup of restored target

---

## Tools used

- **Read / Edit / Write** — config files, log files
- **Bash** — `claude --version`, env-var probe, settings.json validation
- **Subagent spawn** (via Claude Code's Task tool) — routed tasks + A/B test tiers

No external dependencies. No network calls.

---

## Key principles

1. **Measurement > marketing.** Never print "you saved 50%!" without backing numbers from `.tokenwise/log.ndjson`.
2. **Real pricing.** Hardcoded May-2026 rates. Update via PR when Anthropic publishes new rates. Show "rates as of YYYY-MM" in every report.
3. **Privacy.** Zero telemetry. Logs local. PII-redact task descriptions (truncate to 80 chars, no file contents).
4. **No silent writes.** Every config mutation is confirmed and backed up. Undo always available.
5. **Probe before configure.** If routing primitives are broken on the user's Claude Code build, fail loud and refuse to install.
6. **Honest limits.** When asked "should I trust Haiku for X?", run `/tokenwise ab "<X>"` — don't guess.

---

## Edge cases & limitations

- **Token counts approximate:** Claude Code's exposed token counts may not match Anthropic's billing exactly. We aim for ±2%. For exact billing, check the Anthropic Console.
- **Subagent overhead:** spawning a Haiku subagent costs ~500 tokens of orchestration overhead. The trivial-task floor (`<100 chars + no file context → inline`) prevents regressions on tiny tasks.
- **Prompt caching:** the 90% cached-input discount is honored by Anthropic transparently. TokenWise reports raw cost; cached input is billed at 10% but our cost calculation uses full rate (conservative). Real savings may be larger than reported.
- **Plan-tier detection:** Claude Code doesn't expose the user's Max plan tier programmatically. We can't auto-adjust for plan caps.
- **A/B test cost:** running `/tokenwise ab` costs extra tokens (one task × N tiers). This is intentional — A/B is a one-time validation, not continuous.

---

## What TokenWise does NOT do

- ❌ Multi-vendor routing (use LiteLLM or OpenRouter)
- ❌ Non-Anthropic models (Claude-only by design)
- ❌ Bulk async processing (use Anthropic Batch API)
- ❌ Cost prediction before a task runs (v0.2 maybe)
- ❌ Self-healing prompt rewrites for cheaper-model performance
- ❌ Sharing your savings to a public board
- ❌ Modifying any file without explicit confirmation
- ❌ Phoning home

---

## Versioning

This is `tokenwise` v0.1.0. Updates go via PR on `CodeShuX/tokenwise`. See `CHANGELOG.md`.
