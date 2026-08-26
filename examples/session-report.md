# Example Session Report

> Anonymized real session from a Next.js + Postgres SaaS codebase audit. 2h 14m of mixed work: code search, single-file edits, two architecture discussions.

```
TokenWise Session Report
========================

Session ID:          7c3f9a1b
Started:             2026-05-09 09:14:02 UTC
Duration:            2h 14m 38s

Tasks routed:        47

Per model:
  Haiku    32 tasks   1,218,420 input  /   84,310 output   →   $1.6431
  Sonnet   12 tasks     481,990 input  /   41,220 output   →   $2.0639
  Opus      3 tasks     145,100 input  /   28,440 output   →   $1.4365

Total spent:                                                   $5.1435
Baseline (if all-Opus):                                       $24.9320
Savings:                                                      $19.7885
Savings %:                                                       79.4%

Top task classes (by count):
  code-search             18 tasks    avg cost $0.041   model: haiku
  file-read               14 tasks    avg cost $0.012   model: haiku
  scoped-refactor          7 tasks    avg cost $0.187   model: sonnet
  scoped-research          5 tasks    avg cost $0.241   model: sonnet
  architecture             2 tasks    avg cost $0.489   model: opus
  security-review          1 task     avg cost $0.458   model: opus

Quality flags:
  Escalations:        2 (Haiku → Sonnet, mid-task)
    - "trace caller of formatInvoice across 4 files" (escalated: 2 file deps + ambiguous return type)
    - "refactor src/billing/checkout.ts to use new pricing util" (escalated: cross-file dependency)
  User overrides:     0
  Regressions:        0
  Cache-eligible:     3 files (read >3× — see hint detector log)

Pricing snapshot (used for this session, May 2026):
  Opus 4.7:    $5.00 input  / $25.00 output  per 1M tokens
  Sonnet 4.6:  $3.00 input  / $15.00 output  per 1M tokens
  Haiku 4.5:   $1.00 input  / $5.00 output   per 1M tokens
```

**Note:** this session predates the Fable/Planning tier (added August 2026) — it ran under the original 3-tier taxonomy, so "architecture" and "security-review" landed on Opus rather than being split into today's Review (Opus) vs. Planning (Fable) lanes. Real historical numbers, left as originally logged rather than retrofitted to the current taxonomy.

## What this session shows

- **79.4% real savings.** Not a marketing number — the NDJSON log backs every line.
- **Haiku handled 68% of tasks** — almost all the code-search and file-read work.
- **2 escalations.** Both were tasks with cross-file dependencies. The router noticed and bumped up a tier. No user intervention needed.
- **Architecture and security work stayed on Opus.** That's the point — Opus where it earns its cost.

## Reproducing this report

The `.tokenwise/log.ndjson` file for this session has 47 lines. Run:

```bash
cat .tokenwise/log.ndjson | jq -r '[.model_used, .cost_actual_usd, .cost_baseline_usd] | @tsv' | \
  awk '{a[$1]+=$2; b[$1]+=$3} END {for (m in a) printf "%s\t$%.4f\t$%.4f\n", m, a[m], b[m]}'
```

You get the same per-model totals TokenWise reports. The math is auditable.
