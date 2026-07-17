# Skill Benchmark: gitlab-to-github

**Model**: claude-opus-4-6
**Date**: 2026-07-16
**Evals**: 3 (1 run each per configuration)

## Summary

| Metric | With Skill | Without Skill | Delta |
|--------|------------|---------------|-------|
| Pass Rate | 100% ± 0% | 61% ± 35% | **+39%** |
| Time | 106.9s ± 17.3s | 74.6s ± 30.0s | +32.3s |
| Tokens | 82,670 ± 434 | 82,419 ± 7,041 | +251 |

## Per-Eval Breakdown

| Eval | With Skill | Without Skill | Delta |
|------|-----------|---------------|-------|
| 1. Full CI migration | 6/6 (100%) | 2/6 (33%) | +67% |
| 2. Resume mid-migration | 6/6 (100%) | 6/6 (100%)* | ±0% |
| 3. No-CI skip phases | 6/6 (100%) | 3/6 (50%) | +50% |

*Eval 2 baseline was given the manifest as context, inflating its score.

## Analyst Observations

### Discriminating Assertions

The strongest differentiators between with_skill and baseline:

1. **Manifest creation** (evals 1, 3): The skill always produces a `migration-manifest.yaml` with structured state. The baseline never does. This is the skill's core value proposition — persistent cross-session state.

2. **Branch classification** (eval 1): The skill proactively classifies branches into keep/delete/ask using pattern matching (release branches, renovate/*, merged branches). The baseline defers this to the user as a question.

3. **Structured Phase 0 output** (eval 3): The skill produces a formal assessment table. The baseline distributes the same information across prose but lacks a structured artifact.

### Non-Discriminating Assertions

- **Skip logic** (evals 2, 3): Both the skill and baseline correctly identify that phases should be skipped when CI/container push are absent. The baseline's intelligence matches the skill here — it's the structure and persistence that differentiate.

- **Eval 2 entirely** (6/6 both): When the baseline is given the manifest, it performs identically. This confirms the manifest format is intuitive enough for the model to work with — the skill's value is creating and maintaining it, not interpreting it.

### Time/Token Tradeoff

The skill costs ~43% more wall-clock time (+32s) with negligible token overhead (+0.3%). The extra time goes to manifest generation and structured Phase 0 output. For a multi-session migration workflow, this upfront cost pays back through cold-resume capability.

### High-Variance Observations

- Baseline time variance is high (30s stddev, range 44-104s). The skill's time is more consistent (17s stddev). This suggests the skill provides a more predictable experience.
- Baseline pass rate variance is very high (35% stddev). The skill always achieves 100%, making outcomes predictable.

### Key Insight: Intelligence vs. Structure

The baseline is remarkably intelligent — it correctly identifies the GPL-3.0 blocker, understands skip logic, produces good plans, and even found project runbooks autonomously. The skill's advantage is not making the model smarter, but giving it:
1. **Persistent state** (manifest survives across sessions)
2. **Structured output** (assessment tables, phase summaries)
3. **Deterministic flow** (same phases, same order, same format every time)
4. **Actionable commands** (exact curl/glab/gh commands vs. high-level guidance)
