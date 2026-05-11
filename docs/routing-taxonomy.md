# Routing Taxonomy

How TokenWise classifies tasks and picks a model.

## The three tiers

| Tier | Model | Cost vs Opus | Use it for |
|---|---|---|---|
| **Mechanical** | Haiku 4.5 | 0.20× input / 0.20× output | Tasks with no judgment calls |
| **Scoped reasoning** | Sonnet 4.6 | 0.60× input / 0.60× output | Tasks bounded to a small scope |
| **Synthesis / planning** | Opus 4.7 | 1.0× (baseline) | Tasks requiring tradeoffs or cross-cutting reasoning |

## Tier 1: Mechanical (Haiku)

Tasks where there's exactly one correct answer and the question is "find / format / transform it":

- File reads (`Read this file and tell me what's in it`)
- Symbol grep (`Find all uses of X`)
- Rename (`Rename foo to bar across the codebase`)
- Format (`Reformat this JSON / Pretty-print this YAML`)
- Lint-style fixes (`Add missing semicolons`)
- Documentation lookups (`What does function X do?`)
- Dependency listing (`What npm packages does this use?`)
- Simple find-and-replace edits

**Why Haiku:** the work is verifiable. Wrong output is immediately obvious. No reasoning needed beyond pattern matching.

**Safety cap:** Haiku never spawns subagents. If a Haiku task realizes it needs more reasoning, it returns to the parent for re-classification.

## Tier 2: Scoped reasoning (Sonnet)

Tasks that need real thinking but are bounded to a small surface:

- Single-file refactor (`Refactor this function to use early returns`)
- Test writing for one module (`Write unit tests for src/billing/checkout.ts`)
- Scoped research (`How does this codebase handle auth?`)
- Code exploration (`What does this 200-line file do?`)
- Bug fix in a known file (`Fix the off-by-one in the loop at line 42`)
- Scoped code review (`Review the changes in PR diff <X>`)

**Why Sonnet:** judgment is required, but the scope is tight enough that Sonnet's reasoning is plenty. Opus's extra capability would be wasted.

**Safety cap:** if Sonnet's input would exceed 30k tokens, escalate to Opus (more context = more reasoning load).

## Tier 3: Synthesis / planning (Opus)

Tasks that span the codebase, weigh tradeoffs, or have ambiguous requirements:

- Architecture decisions (`Should we extract this into a separate service?`)
- Multi-file refactor synthesis (`Migrate auth from Passport to Auth.js across the project`)
- Security review (`Review the auth flow for vulnerabilities`)
- Ambiguous requirements (`Make this faster` — needs to identify the bottleneck first)
- Novel algorithm design
- Cross-cutting bug RCA (`Why are users reporting random logouts?`)
- Orchestration (the parent thread that dispatches subagents is always Opus)

**Why Opus:** these are the tasks where capability differences show up in output quality. This is what Opus is for.

## How a task gets classified

The orchestrator (Opus, always) reads the task description and picks a tier using these heuristics:

1. **Trivial floor.** Task description <100 chars AND no file context → run inline. Don't spawn anything.
2. **Single-symbol mechanical.** Description contains rename/find/format/grep + a clear scope → Haiku.
3. **Single-file scope.** Description names exactly one file AND verb is refactor/test/explain/fix → Sonnet.
4. **Multi-file or ambiguous.** Description names multiple files OR uses words like "decide / weigh / migrate / synthesize / audit" → Opus.
5. **Override:** user-provided hint (`# tokenwise: haiku` in CLAUDE.md context) takes precedence.

## Escalation

A subagent can request escalation by returning to its parent with reason. The parent (Opus) re-classifies the task one tier higher and re-spawns.

Common escalation triggers (logged in `escalation_reason`):

- `more-than-2-file-deps` — Haiku discovered the task crosses 3+ files
- `ambiguous-spec` — Haiku/Sonnet can't confidently disambiguate
- `external-deps` — task needs to reason about third-party library behavior
- `tradeoff-required` — multiple valid solutions, need to pick

**Important:** subagents NEVER escalate on their own. They always return control. This prevents runaway cascading escalations that would blow past budgets.

## Overriding the default taxonomy

Add to the TokenWise block in your `CLAUDE.md`:

```yaml
tokenwise:
  overrides:
    - pattern: "write release notes for"
      tier: sonnet     # default would be opus; we know our release notes are formulaic
    - pattern: "review auth"
      tier: opus       # default would be sonnet; we always want Opus for auth
```

In v0.1.0, pattern is a substring match. In v0.2, glob + regex support.

## When the taxonomy is wrong

Run `/tokenwise ab "<task>"` on a representative example. The A/B report tells you which tier was actually sufficient. Update the override list based on real data, not intuition.
