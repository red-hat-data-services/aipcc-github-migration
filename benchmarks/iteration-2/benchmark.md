# Skill Benchmark: gitlab-to-github (Iteration 2)

**Model**: claude-opus-4-6
**Date**: 2026-07-17
**Evals**: 3 (1 run each per configuration)
**Changes since iteration 1**: Removed Phase 7, bundled POLICY.md, added early GitHub repo request prompt, marked cross-repo fields as future use

## Summary

| Metric | With Skill | Without Skill | Delta |
|--------|------------|---------------|-------|
| Pass Rate | 100% ± 0% | 61% ± 35% | **+39%** |

## Per-Eval Breakdown

| Eval | With Skill | Without Skill | Delta |
|------|-----------|---------------|-------|
| 1. Full CI migration | 6/6 (100%) | 2/6 (33%) | +67% |
| 2. Resume mid-migration | 6/6 (100%) | 6/6 (100%)* | ±0% |
| 3. No-CI skip phases | 6/6 (100%) | 3/6 (50%) | +50% |

*Eval 2 baseline was given the manifest as context, inflating its score.

## Iteration 2 Changes Validated

All iteration-2 changes correctly surfaced in the with-skill responses:

1. **6 phases, no Phase 7**: Resume dashboard (eval 2) and fresh assessments (evals 1, 3) both show exactly 6 phases. No reference to `gitlab_branch_cleanup` anywhere.

2. **Bundled POLICY.md**: Eval 1 response uses `cp templates/POLICY.md .` instead of the old Google Doc export approach. Removes an external dependency.

3. **Early GitHub repo request prompt**: Evals 1 and 3 (where `github_state == "absent"`) correctly show the prompt in Phase 0 output. Eval 2 (where `github_state == "has_content"`) correctly omits it. Critical path optimization working as intended.

4. **Self-reference skip logic**: All evals produce `gitlab_self_references: []` → cleanup item skipped with reason. New feature validates cleanly.

5. **Cross-repo fields as future use**: Manifest shows `includes_from: []` and `included_by: []` without acting on them. Comment updated to "(future use)".

## Analyst Observations

### Consistency with Iteration 1

The core value proposition is unchanged: persistent state (manifest), structured output (assessment tables), deterministic flow (phases), and actionable commands. The delta vs. baseline remains +39% — baseline results are identical since the without-skill configuration is unchanged.

### What Improved

- **POLICY.md handling** is now deterministic. Iteration 1 relied on downloading from Google Docs (fragile, requires auth). Iteration 2 bundles the template — zero external dependencies for this step.

- **Phase count** dropped from 7 to 6 without losing any passing assertions. Phase 7 was optional cleanup that added complexity without adding value in the eval scenarios.

- **GitHub repo request timing** moved from Phase 2 (blocking) to Phase 0 (parallel). This doesn't affect eval pass rates but represents a real-world critical path optimization — the form submission can take hours/days, and now that wait overlaps with Phase 1 cleanup.

### Key Insight: Fewer Phases, Same Coverage

Dropping Phase 7 simplified the skill without losing correctness. The 6-phase flow covers the full migration lifecycle, and branch cleanup during Phase 1 (where it naturally fits) is cleaner than a separate optional phase at the end.
