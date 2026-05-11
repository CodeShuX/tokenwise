# Example A/B Test

> Real A/B test on a single task. Same prompt, two tiers, side-by-side comparison.

```markdown
# A/B Test — rename getCwd → getCurrentWorkingDirectory
Run: 2026-05-09 11:47:22 UTC
Task: "Rename all uses of getCwd to getCurrentWorkingDirectory across the codebase"
Tiers tested: haiku, sonnet

## Tier comparison

| Tier   | Tokens (in/out)  | Cost     | Duration | Files touched | Quality |
|--------|------------------|----------|----------|---------------|---------|
| Haiku  | 12,408 / 890     | $0.0169  | 4.2s     | 8 files       | 8/10    |
| Sonnet | 11,820 / 1,213   | $0.0536  | 6.8s     | 8 files       | 9/10    |

## Output equivalence

- Files modified: **identical** (same 8 files, same lines)
- New symbol name: **identical** (getCurrentWorkingDirectory)
- Existing tests still pass: **both** (verified locally)

## Quality breakdown

**Haiku output (8/10):**
- ✅ All 8 files updated correctly
- ✅ Import statements updated
- ⚠️ Did not update the JSDoc comment referencing the old name in `src/utils/path.ts:24`
- ⚠️ Did not update the README example

**Sonnet output (9/10):**
- ✅ All 8 files updated correctly
- ✅ Import statements updated
- ✅ Updated JSDoc comment in `src/utils/path.ts:24`
- ✅ Updated README example
- ⚠️ Missed one comment in `CHANGELOG.md` referencing the old name (semantically correct to leave it; arguable)

## Recommendation

For pure code-symbol renames, **Haiku is sufficient** — both produced functionally identical code.
For renames that should sweep through docs + comments + tests, **Sonnet is worth the 3.2× cost**.

Suggested router rule:
- Task includes "across the codebase" + no docs mentioned → Haiku
- Task includes "and update docs/comments" → Sonnet

## Cost summary

This A/B test cost $0.0705 to run. If you do this rename on Haiku going forward instead of all-Opus (which would have cost $0.234 for similar token use), you save $0.217 per rename. The A/B paid for itself in **<1 future rename**.
```

## When to run an A/B

- Before trusting Haiku for a recurring task class
- When you suspect the router is over-escalating
- When Anthropic releases a new model and you want to revalidate the taxonomy
- When you adopt TokenWise on a new codebase

You don't need to run A/B tests constantly. Run a handful when you start, save the reports, and the router config that emerges is calibrated to *your* code — not a generic average.
