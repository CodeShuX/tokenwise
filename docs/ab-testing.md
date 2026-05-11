# A/B Testing

The whole point of TokenWise is that you don't have to trust the router — you can verify it.

`/tokenwise ab "<task>"` runs the same task on multiple tiers, captures outputs, scores them, and writes a comparison report.

## When to A/B

- **Onboarding TokenWise to a new codebase.** Run 3-5 A/B tests on representative tasks. The results tell you which tier is actually sufficient *for your code*.
- **Calibrating an override.** Before adding a `tokenwise.overrides` rule to your CLAUDE.md, A/B the task class.
- **Anthropic releases a new model.** Re-A/B your common task classes to see if the routing tiers should shift.
- **Suspecting over-escalation.** If `/tokenwise report` shows lots of Haiku → Sonnet escalations on the same task class, A/B that class to see if Sonnet was actually needed.

## What `ab` does

```
/tokenwise ab "<task description>" [--tiers haiku,sonnet,opus]
```

Steps:

1. Parse the task description
2. For each tier in `--tiers` (default: `haiku,sonnet`):
   - Spawn a subagent at that tier with the exact prompt
   - Capture stdout, token usage, duration, errors
3. Compute pairwise diffs:
   - **Structural:** line count, token overlap, file-count touched
   - **Semantic:** ask Opus to score each output 1-10 on completeness + correctness
4. Write `tokenwise-ab-<timestamp>.md` to project root

## A/B test cost

A/B is intentionally not cheap — you're running the task N times. The cost is logged the same way regular routing is logged, with `task_class: "ab-test"` so it doesn't pollute your normal stats.

**Rule of thumb:** an A/B costs roughly `N × the task's normal cost at the highest tier tested`. So an A/B of `haiku,sonnet` for a task that would cost $0.05 at Sonnet costs ~$0.07.

A/B tests pay for themselves quickly: one verified "Haiku is fine for this" saves the cost of the A/B in 1-3 future runs.

## Interpreting results

A typical report:

```markdown
| Tier   | Tokens (in/out) | Cost   | Duration | Quality |
|--------|-----------------|--------|----------|---------|
| Haiku  | 12.4k / 890     | $0.017 | 4.2s     | 8/10    |
| Sonnet | 11.8k / 1.2k    | $0.053 | 6.8s     | 9/10    |
```

**Quality delta of 1 point** is usually noise. Use the lower tier unless the difference is functional.

**Quality delta of 2+ points** means something. Inspect the diff to see what the higher tier caught.

**Equal quality, big cost gap:** strong signal to add an override sending this task class to the cheaper tier.

## Limits

- `ab` is **not** an automated regression test. It runs the task in isolation — you should still spot-check outputs.
- Some tasks produce non-deterministic outputs (e.g. "suggest improvements"). The semantic quality score helps, but for these tasks the A/B report is advisory, not definitive.
- Multi-step tasks that depend on user input mid-flow can't be A/B'd cleanly. Decompose into a single-shot task first.

## Example

You suspect Haiku is good enough for "find unused exports across the codebase":

```
/tokenwise ab "find all unused exports in src/ and list them in a markdown table"
```

Report says: `Haiku 9/10, Sonnet 9/10`. Same output, 5× cheaper.

Add to CLAUDE.md:
```yaml
tokenwise:
  overrides:
    - pattern: "find all unused exports"
      tier: haiku
```

Now every future run of that task class costs 20¢ on the dollar — verified by data, not vibes.
