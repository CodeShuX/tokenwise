# TokenWise

> **Cut Claude Code token spend without sacrificing quality — and prove it.**

A Claude Code skill that routes work to the cheapest model that can handle it, measures actual savings on your real workload, and shows you the proof.

> ⚠️ **v0.1.0 — in active development.** See [SPEC.md](./SPEC.md) for the locked v0.1.0 plan. Skill is not yet installable; this README will be filled in alongside the implementation.

## What it does

- **Routes** — Opus orchestrates, Haiku/Sonnet handle the grunt work
- **Measures** — every routed task logged with real token + cost numbers, locally
- **A/B-tests** — compare Haiku vs Sonnet vs Opus on YOUR codebase, see the quality delta
- **Reports** — `/tokenwise report` shows verified savings vs all-Opus baseline
- **Privacy-first** — zero telemetry, all logs stay in your project

## Status

| Milestone | Status |
|---|---|
| Spec lock | ✅ [SPEC.md](./SPEC.md) |
| Skill build | 🚧 In progress |
| Smoke test | ⏳ |
| Public launch | ⏳ |

## License

MIT — see [LICENSE](./LICENSE).
