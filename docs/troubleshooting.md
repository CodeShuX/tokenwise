# Troubleshooting

## "Routing probe failed at install"

TokenWise refused to install because it couldn't verify subagent model routing works on your Claude Code build. Two likely causes:

**1. Anthropic Issue #47488** — subagent `model:` param silently overridden to Haiku regardless of what you set.

```bash
claude --version
```

If you're on an affected build, options:
- Update Claude Code: `claude update` (or your package manager's equivalent)
- Pin to a known-good version (community has reported v2.0.85+ working)
- Run `/tokenwise install --manual` — TokenWise will print the config block without writing, and the routing will work at the orchestrator-prompt level even if the SDK primitive is broken

**2. Probe timeout.** Slow network or busy Anthropic API. Retry: `/tokenwise install --retry-probe`.

## "Env-var probe failed"

TokenWise tested `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` and found Claude Code didn't honor it (Anthropic Issue #36381).

This isn't fatal — TokenWise will still install routing rules. It just won't add the env vars. The CLAUDE.md additions provide ~80% of the savings; env vars are the last 20%.

Override:

```
/tokenwise install --skip-env-probe
```

## "Cannot find ~/.claude/CLAUDE.md"

TokenWise creates a project-level `./CLAUDE.md` if neither global nor project file exists. You'll be prompted to confirm. If you want a global file instead:

```bash
touch ~/.claude/CLAUDE.md
/tokenwise install
```

## "TokenWise wrote to CLAUDE.md but routing doesn't seem to work"

Three quick checks:

1. **Restart Claude Code.** `CLAUDE.md` is loaded on session start. Existing sessions don't see the routing rules.
2. **Check the block was written:**
   ```bash
   grep -c "BEGIN TokenWise" ~/.claude/CLAUDE.md
   # should print 1
   ```
3. **Test routing manually:**
   ```
   /tokenwise probe
   ```
   This spawns a probe subagent at each tier and reports back. If all three tiers return their expected model identifier, routing works.

## "Log file is huge"

`.tokenwise/log.ndjson` is append-only. After a few months it can hit tens of MB. Truncate safely:

```bash
# Keep last 30 days
jq -c 'select(.ts > (now - 30*86400 | strftime("%Y-%m-%dT%H:%M:%SZ")))' \
   .tokenwise/log.ndjson > .tokenwise/log.tmp && \
   mv .tokenwise/log.tmp .tokenwise/log.ndjson
```

Or rotate it yourself — TokenWise doesn't care about the file's history beyond what `summary` is reporting on.

## "Report shows wrong cost"

Two common causes:

1. **Stale pricing.** Anthropic updates rates. Check `skill/SKILL.md` Phase 3 pricing table against current Anthropic pricing. Open a PR if outdated.
2. **Prompt caching.** TokenWise reports raw cost (conservative). If your work hits prompt cache often, actual cost is lower than reported. The report's `savings_usd` is therefore an underestimate.

## "I want to disable TokenWise temporarily"

Don't uninstall — just comment out the routing block:

```bash
sed -i.bak 's|^## Model routing (TokenWise)|<!-- DISABLED: ## Model routing (TokenWise)|' ~/.claude/CLAUDE.md
```

Re-enable: revert from `.bak`. Or use `/tokenwise undo` to restore the original.

## "I want to switch back to all-Opus"

```
/tokenwise undo
```

Pick the backup from before install. Done. The router instructions are no longer in CLAUDE.md, so Claude Code reverts to default Opus behavior.

## "I want to file a bug"

Include:
- `claude --version` output
- The exact `/tokenwise` command you ran
- The error message verbatim
- A redacted `.tokenwise/log.ndjson` excerpt if relevant

Open at https://github.com/CodeShuX/tokenwise/issues.
