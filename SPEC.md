# TokenWise — Spec v0.1.0

> A Claude Code skill that routes work to the cheapest model that can handle it, measures actual savings on your real workload, and proves the routing didn't hurt quality.

**Status:** locked v0.1.0 spec. Source of truth — features outside this doc are out of scope until v0.2.

---

## 1. Problem

Claude Code users on Max plans burn through Opus tokens on tasks Haiku could handle in a tenth the cost. The pain is twofold:

1. **Cost.** Opus is 5× Haiku and 1.67× Sonnet (per Anthropic pricing, May 2026: Opus 4.7 $5/$25, Sonnet 4.6 $3/$15, Haiku 4.5 $1/$5 per 1M tokens input/output). A "research this file" task running on Opus is paying 5× for no quality lift.
2. **Context overrun.** The primary thread balloons fast when grunt work runs inline. Subagents have their own context budgets — work routed out and synthesized back keeps the primary thread alive for long sessions without forced auto-compact.

**Anthropic Issue #27665** documents that 93.8% of Max-subscriber tokens flow to Opus despite community demand for auto-routing (Issue #44976). Anthropic has not shipped this. Existing routers (claude-router, wshobson, VoltAgent) either pin models statically or use vibes-based heuristics with no measurement.

**TokenWise fills that gap with measurement-first routing.**

---

## 2. Target user

Claude Code users on Max plans (or pay-as-you-go) who:
- Are hitting token caps mid-session
- Suspect they're overspending but can't quantify it
- Want savings without trusting a marketing claim
- Are comfortable with `/plugin install` but not necessarily editing `settings.json` by hand

Secondary: power users who want a programmable routing layer they can extend.

---

## 3. Scope (v0.1.0)

### In scope

1. **Router** — Opus orchestrator delegates to Haiku/Sonnet subagents based on task taxonomy
2. **Measurement** — every subagent spawn logged with tokens-per-task-class to `.tokenwise/log.ndjson`
3. **A/B-test mode** — `/tokenwise ab <task>` runs the same task at multiple tiers, diffs outputs, reports cost+quality
4. **Config installer** — guided (auto-write with diff preview) or manual (print-and-exit) — installs routing rules + env vars
5. **Reports** — `/tokenwise report` (session), `/tokenwise summary --week` (trend), `/tokenwise undo` (restore)
6. **Real-time cost ticker** — optional running $ counter, toggleable
7. **Budget cap** — alert when session crosses configured threshold
8. **Prompt-cache hint detector** — flag re-reads of same file as cache-eligible
9. **Context-window watcher** — alert at 70%/85%/95% of context
10. **Privacy-first** — zero telemetry, all logs local

### Out of scope (deferred to v0.2+)

- Cost-per-task pre-estimator
- GitHub Action that comments savings on PRs
- Multi-vendor routing (LiteLLM/OpenRouter-style)
- Batch API integration (24h SLA doesn't fit interactive use)
- Shared/public leaderboard
- Cost projection across full project history
- Self-healing prompt rewrites for cheaper-model performance

### Explicitly NOT doing

- We do NOT promise a specific % savings ("50% guaranteed!"). Savings depend on workload. We measure and report the truth.
- We do NOT route to non-Anthropic models. Anthropic-only by design.
- We do NOT silently modify any file. Every write needs explicit confirmation with diff preview.
- We do NOT phone home. All logs stay in `.tokenwise/` in the user's project.

---

## 4. The five phases

### Phase 1 — Detect

When the user invokes `/tokenwise install`:
- Scan for `~/.claude/CLAUDE.md`, project `CLAUDE.md`, `~/.claude/settings.json`
- Detect Claude Code version (run `claude --version`)
- Test whether `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` actually works on this build (set, read back, verify)
- Detect plan tier from session metadata if available
- Inventory existing subagents/skills to avoid conflicts

**Output:** detection report shown to user before any change.

### Phase 2 — Configure (guided or manual)

Two modes:

**Guided (default):**
1. Show diff of proposed CLAUDE.md additions
2. Show diff of proposed settings.json additions
3. User confirms each [Y/n] per file
4. Backup original to `<file>.tokenwise-backup-<timestamp>`
5. Write changes
6. Verify by reading back

**Manual (for power users):**
1. Print copy-paste block with the same content
2. Exit with no file mutations
3. User edits by hand

Auto-detect mode preference from user signals:
- Existing custom hooks/skills → likely power user → default to manual with override
- Clean config → default to guided

The user can always force a mode: `/tokenwise install --guided` or `--manual`.

### Phase 3 — Measure

Once installed, every routed subagent spawn writes one NDJSON line to `.tokenwise/log.ndjson`:

```json
{
  "ts": "2026-05-11T14:32:18Z",
  "session_id": "abc123",
  "task_class": "code-search",
  "model_used": "haiku-4-5",
  "model_baseline": "opus-4-7",
  "input_tokens": 12450,
  "output_tokens": 890,
  "cost_actual_usd": 0.0169,
  "cost_baseline_usd": 0.0844,
  "savings_usd": 0.0675,
  "escalated": false,
  "duration_ms": 4210
}
```

Logging is append-only, no PII, no remote send.

### Phase 4 — A/B test

`/tokenwise ab <task description>` does:
1. Runs the same task on Haiku, Sonnet, and (optionally) Opus
2. Captures outputs verbatim
3. Diffs outputs structurally (token-level + semantic comparison)
4. Generates a markdown report: `tokenwise-ab-<timestamp>.md`
5. Reports: cost per tier, output similarity score, recommended tier

Useful for users to validate "is Haiku really good enough for my codebase?" before trusting the router.

### Phase 5 — Report

Three subcommands:

- `/tokenwise report` — current session: tokens per model, $ saved vs all-Opus, task-class breakdown, escalations, quality flags
- `/tokenwise summary --week` (or `--month`, `--all`) — historical trend across `.tokenwise/log.ndjson`
- `/tokenwise undo` — list backups, restore one in place

---

## 5. The routing taxonomy

Tasks classified into 3 tiers. The orchestrator (Opus) uses this taxonomy to pick a subagent model.

| Task class | Model | Examples |
|---|---|---|
| **Mechanical** | Haiku | file reads, grep, format, rename symbol, list files, simple text edits, doc lookup, dependency listing |
| **Scoped reasoning** | Sonnet | single-file refactor, test writing, scoped research, code exploration, bug-fix in known file, scoped review |
| **Synthesis / planning** | Opus | architecture decisions, multi-file refactor synthesis, security review, ambiguous requirements, novel algorithm design, cross-cutting bug RCA |

### Safety caps

- Haiku never spawns subagents (if the task needs delegation, it was wrong-sized)
- Max spawn depth = 2 (parent → subagent → one more tier)
- If a subagent realizes it needs a smarter model, it returns control to parent — never escalates on its own
- Subagent input size cap: if `>30k tokens` of context needed, bump tier (Haiku → Sonnet, Sonnet → Opus)
- Trivial-task floor: if task description `<100 chars` AND no file context, do inline on parent (no subagent — subagent overhead exceeds savings)

---

## 6. Honest limits

**TokenWise is NOT:**

| Tool | Use case |
|---|---|
| LiteLLM / OpenRouter | Multi-vendor model routing |
| Anthropic Batch API | 24h-SLA bulk processing |
| ccusage / claude-monitor | Token tracking only (no routing) |
| claude-router | Heuristic routing without measurement |
| wshobson/agents | Static subagent catalog with pinned models |

TokenWise specifically does **measurement-driven routing for Anthropic on Claude Code**. Adjacent tools cover other angles.

**Known unknowns:**
- Claude Code's subagent `model:` param has had silent-fail bugs (Issue #47488). TokenWise tests routing on install and refuses to configure if routing isn't working on the user's build.
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=80` semantics have changed across Claude Code versions. TokenWise tests-and-skips if the env var is non-functional.
- Subagent overhead can hit 7× in worst case if mis-tuned. The trivial-task floor exists specifically to prevent regressions.

---

## 7. Killer demo

**The README hero is a before/after measurement screenshot:**

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
  Regressions:        0
```

This is the screenshot we put at the top of the README. Real numbers from a real session — no synthetic demo.

---

## 8. File layout

```
tokenwise/
├── README.md                 # marketing + install + roadmap
├── LICENSE                   # MIT
├── CONTRIBUTING.md
├── CHANGELOG.md
├── SPEC.md                   # this file
├── .claude-plugin/
│   └── marketplace.json      # /plugin marketplace add install path
├── skill/
│   └── SKILL.md              # the Claude Code skill
├── docs/
│   ├── installation.md
│   ├── routing-taxonomy.md   # how the router decides
│   ├── ab-testing.md
│   ├── privacy.md            # explicit "no telemetry" doc
│   └── troubleshooting.md
├── examples/
│   ├── session-report.md     # real anonymized session
│   └── ab-comparison.md      # real A/B test output
└── assets/
    ├── demo.gif              # ~60sec, real session
    ├── social-preview.png    # 1280x640
    └── screenshots/
```

---

## 9. Versioning

- v0.1.0 — this spec
- v0.2 — pre-task cost estimator, GitHub Action, multi-month digest
- v1.0 — workload profiles (a user can save "my-Rails-app" taxonomy as a profile and share it)

Per-skill decisions to revisit:
- Whether the auto-config writer should be opt-in instead of default-guided
- Whether to expose the routing taxonomy as user-editable YAML (likely yes in v0.2)

---

## 10. Success criteria

v0.1.0 ships when:
- ✅ Router works against current Claude Code (verified end-to-end against Issue #47488 regression test)
- ✅ Measurement log accurate to ±2% vs Anthropic billing
- ✅ A/B test produces interpretable output on a real codebase (not just toy)
- ✅ Guided install completes on a clean Mac and a clean Linux box without errors
- ✅ `undo` restores config exactly to pre-install state
- ✅ README has a real session-report screenshot (not synthetic)
- ✅ Smoke test passes: `validate` → `marketplace add` → `install` → `list` → invoke → `uninstall` → `remove`

---

*Locked: 2026-05-11. Updates after this point go via SPEC.md PR, not silent edits.*
