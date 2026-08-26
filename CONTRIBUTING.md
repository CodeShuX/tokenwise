# Contributing to TokenWise

Thanks for considering a contribution. TokenWise is MIT-licensed and welcomes pull requests.

## Ground rules

1. **Measurement > marketing.** Every claim about savings must be backed by data in `.tokenwise/log.ndjson` or an A/B test report. No vibes.
2. **One PR per concern.** Don't bundle "fix typo + add feature + update docs."
3. **No silent file writes.** Any feature that mutates a user's config must show a diff and require confirmation. No exceptions.
4. **No telemetry.** Period. TokenWise stays local-only.

## What needs help

- **Pricing updates.** When Anthropic publishes new rates, update `skills/install/SKILL.md`'s pricing table and `CHANGELOG.md`.
- **Bug reports** with real `.tokenwise/log.ndjson` excerpts (PII-redacted).
- **Routing taxonomy refinements** — backed by A/B test reports showing quality regressions on the current taxonomy.
- **Cross-platform install testing** — Linux / Mac / Windows / WSL.
- **Docs** — especially `docs/troubleshooting.md` for common Claude Code routing-bug workarounds.

## PR checklist

- [ ] Tested on a clean Claude Code install
- [ ] If touching the router: included an A/B test report showing no quality regression
- [ ] If touching pricing: cited Anthropic pricing docs with date
- [ ] If touching the installer: ran `/tokenwise undo` to verify restoration works
- [ ] No new dependencies (TokenWise is self-contained by design)

## License

By contributing, you agree your code is MIT-licensed.
