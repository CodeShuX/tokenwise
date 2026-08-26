# Routing Taxonomy

How TokenWise classifies tasks and picks a model. Four parallel lanes, keyed
on task TYPE. Every lane is a direct, automatic destination — no tier
requires confirmation, and none is a "ceiling" bolted onto the others.

## The four tiers

| Tier | Model | Cost vs Opus | Task type |
|---|---|---|---|
| **Mechanical** | Haiku 4.5 | 0.20× input / 0.20× output | One-right-answer work, no judgment |
| **Execution** | Sonnet 4.6 | 0.60× input / 0.60× output | Bounded implementation in a known scope |
| **Review** | Opus 4.7 | 1.0× (baseline) | Judgment over existing work or given options |
| **Planning** | Fable 5 | 2.0× input / 2.0× output | Large, cross-cutting, or ambiguous design work |

## Tier 1: Mechanical (Haiku)

Tasks where there's exactly one correct answer and the question is "find /
format / transform it":

- File reads (`Read this file and tell me what's in it`)
- Symbol grep (`Find all uses of X`)
- Rename (`Rename foo to bar across the codebase`)
- Format (`Reformat this JSON / Pretty-print this YAML`)
- Lint-style fixes (`Add missing semicolons`)
- Documentation lookups (`What does function X do?`)
- Dependency listing (`What npm packages does this use?`)
- Simple find-and-replace edits

**Why Haiku:** the work is verifiable. Wrong output is immediately obvious. No
reasoning needed beyond pattern matching.

**Safety cap:** Haiku never spawns subagents. If a Mechanical task realizes it
needs more reasoning, it returns to the parent for reclassification.

## Tier 2: Execution (Sonnet)

Tasks that need real thinking but are bounded to a known, small surface:

- Single-file refactor (`Refactor this function to use early returns`)
- Test writing for one module (`Write unit tests for src/billing/checkout.ts`)
- Bug fix in a known file (`Fix the off-by-one in the loop at line 42`)
- Scoped research (`How does this codebase handle auth?`)
- Code exploration (`What does this 200-line file do?`)
- Scoped code review of a single change (`Review the changes in this diff`) —
  see the Review tier below for judgment calls over a whole feature or system
- **Small-scoped planning** (`Plan the refactor of this file`, `Sketch the
  test strategy for this module`) — a plan bounded to one file or module is
  execution prep. It belongs here, not in the Planning tier.

**Why Sonnet:** judgment is required, but the scope is tight enough that
Sonnet's reasoning is plenty. A more capable model's extra headroom would be
wasted.

**Safety cap:** if the input context would exceed 30k tokens, run the task on
Opus instead (more context = more reasoning load). The task is still logged
as `execution` — this is a capability bump within the lane, not a
reclassification.

## Tier 3: Review (Opus)

Tasks whose essence is judgment over something that already exists, or a
decision among options someone has already laid out:

- Security review (`Review the auth flow for vulnerabilities`)
- Full-PR or cross-cutting code review (`Review this feature end to end`)
- Ambiguous requirements needing diagnosis (`Make this faster` — first
  identify the bottleneck, then decide)
- Cross-cutting bug RCA (`Why are users reporting random logouts?`)
- Auditing outputs (including judging `/tokenwise:ab` results)
- Choosing between already-stated options (`Postgres or SQLite for this,
  given these constraints — which?`)

**Why Opus:** these are the tasks where capability differences show up in
output quality, but the work is evaluating or diagnosing something that
already exists rather than inventing a design from a blank page. That's what
keeps it a rung below Planning.

**Note:** the orchestrator — the parent thread that classifies and dispatches
— is the session's own model and is not itself routed.

## Tier 4: Planning (Fable)

Generative design work that is genuinely large: cross-cutting, system-wide,
or starting from ambiguous requirements.

- System-wide architecture (`Should we extract this into a separate
  service?`, `Design the migration off Passport to Auth.js across the
  project`)
- Multi-file or cross-system design (`Plan how to split this monolith module
  into services`)
- Decomposing ambiguous requirements into a work plan before anyone executes
  anything
- Novel algorithm or protocol design

**What does NOT come here:**
- Planning scoped to ≤2 files or one module → Execution (Tier 2)
- Evaluating a design someone already produced, or picking between stated
  options → Review (Tier 3)

**Why Fable:** on large planning, plan quality compounds — every downstream
Execution task inherits the plan's mistakes. Paying 2× Opus once, at the top
of the funnel, is the cheapest place in the whole workflow to buy extra
capability. Fable routes automatically here, exactly like the other three
tiers — there is no confirmation step.

**Cost note:** `cost_baseline_usd` stays pinned to Opus (the all-Opus
baseline), so a Planning line's `savings_usd` is negative. That's by design —
it reports honestly that this task cost more than the baseline. Reports show
it as-is, not clamped to zero.

Anthropic also runs its own safety layer on Fable: in high-risk categories
(cybersecurity, biology, chemistry) it can decline and fall back to Opus. If a
Planning task comes back as an Opus response, that's Anthropic's guardrail,
not a TokenWise bug — see [troubleshooting](./troubleshooting.md).

## How a task gets classified

The orchestrator reads the task description and applies these rules in order:

1. **Trivial floor.** Description <100 chars AND no file context → run
   inline. Don't spawn anything.
2. **Mechanical.** Verb is find/grep/rename/format/read/list with a clear
   scope → Haiku.
3. **Execution.** Verb is implement/refactor/fix/test/explore, scoped to ≤2
   named files or one module → Sonnet. Scoped planning questions land here too.
4. **Review.** Verb is review/audit/evaluate/diagnose over existing code or
   given options → Opus.
5. **Planning.** Generative design spanning 3+ files, crossing systems, or
   starting from ambiguous requirements → Fable.
6. **Override:** user-provided hint (`# tokenwise: haiku` in CLAUDE.md
   context) takes precedence over all of the above.

When two types fit, prefer the cheaper lane. Misroutes are corrected by one
reclassification (next section), so under-routing is cheap; over-routing
isn't.

## Reclassification

A subagent that discovers it was misclassified stops and returns control to
its parent with a reason — it never re-routes or escalates on its own. That
invariant predates the 4-lane model and stays unchanged.

The parent reclassifies **by type, directly to any lane** — a Sonnet
Execution task that turns out to be a large planning problem goes straight to
Fable in one hop; there's no ladder to climb through Opus first. **At most
one reclassification per task** — if the re-spawned task bounces again, the
parent finishes it inline rather than ping-ponging.

Common escalation triggers (logged in `escalation_reason`):

- `needs-mechanical` / `needs-execution` / `needs-review` / `needs-planning`
  — wrong type; the value names the lane the subagent believes is correct
- `ambiguous-spec` — the subagent can't confidently disambiguate what's asked
- `insufficient-capability` — right type, but the task outgrew the model
  (e.g. an Execution task whose real scope turned out to span 5 files)

Two size-based caps interact with classification but don't replace it:

- **Input bump:** context >30k tokens runs on the next more capable model
  within a lane (Haiku → Sonnet, Sonnet → Opus). It stops at Opus — the bump
  compensates for context volume, which Opus fully handles. Fable is reached
  by task type (Planning), never by input size.
- **Max spawn depth = 2** (parent → subagent → one more), and Haiku never
  spawns subagents at all.

## Overriding the default taxonomy

Add to the TokenWise block in your `CLAUDE.md`:

```yaml
tokenwise:
  overrides:
    - pattern: "write release notes for"
      tier: sonnet     # default would be opus; we know our release notes are formulaic
    - pattern: "review auth"
      tier: opus       # default would be sonnet; we always want Opus for auth
    - pattern: "design the new consensus algorithm"
      tier: fable      # pin it — this specific task class always warrants the Planning tier
    - pattern: "plan the sprint"
      tier: sonnet     # our sprint plans are scoped and templated; Fable would be overkill
```

In v0.1.0, pattern is a substring match; `tier` is the model name
(`haiku`/`sonnet`/`opus`/`fable`). In v0.2, glob + regex support.

## When the taxonomy is wrong

Run `/tokenwise:ab "<task>"` on a representative example. The A/B report
tells you which tier was actually sufficient. Update the override list based
on real data, not intuition — this is also how you calibrate the Planning
lane if you suspect Fable is being reached too often for your codebase.
