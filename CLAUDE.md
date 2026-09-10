# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repo is a **Claude Code plugin marketplace** (`.claude-plugin/marketplace.json`) that
publishes a single plugin: `gitlab-to-github`, a guided-migration skill for moving AIPCC
repositories from GitLab to GitHub. There is no application code, build step, or test runner —
the "product" is the skill's Markdown/YAML content, and correctness means the skill's
instructions are internally consistent and produce the right agent behavior when invoked.

## Repo Structure

```
.claude-plugin/marketplace.json          # Marketplace manifest — lists plugins and their source paths
src/gitlab-to-github/
  .claude-plugin/plugin.json             # Plugin manifest (name, version, repo, keywords)
  skills/gitlab-to-github/
    SKILL.md                             # The skill itself — frontmatter + full invocation flow
    references/phase-{0..6}-*.md         # Per-phase detailed guidance, loaded on demand by SKILL.md
    templates/migration-manifest.yaml    # Template written to `.claude/migrations/<repo>/manifest.yaml`
  skills/ldap/SKILL.md                   # Red Hat LDAP lookups, used to translate CODEOWNERS handles
tests/gitlab-to-github/skills/gitlab-to-github/evals.json   # Eval scenarios (prompt + expected behavior)
```

Adding a second plugin means adding a new `src/<plugin-name>/` tree and a matching entry in
`.claude-plugin/marketplace.json`'s `plugins` array. A plugin directory with no corresponding
`plugins[]` entry is invisible to `/plugin install` — always register it.

### Marketplace registration

`marketplace.json`'s top-level `name` field (currently `aipcc-github-migration`) is the identifier
end users pass as `<marketplace-name>` in `/plugin install <plugin-name>@<marketplace-name>`. If
you ever rename it, update the install snippet in README.md to match — it's not derived
automatically.

## Working on the Skill

`SKILL.md` is the entry point Claude Code loads when the skill is invoked; `references/*.md` are
loaded lazily per-phase (see the "Reference Files" table at the bottom of SKILL.md) — don't merge
them into SKILL.md itself, that defeats the point of the progressive-disclosure structure.

Key invariants baked into the skill design — preserve these when editing:
- **Stateful, resumable**: all progress lives in `.claude/migrations/<repo>/manifest.yaml`
  (per-project, gitignored). The skill must re-read this manifest at the start of every
  invocation rather than assuming in-memory state.
- **Phase skip logic** lives in SKILL.md's "Phase Loop" section (e.g. no CI → skip `ci` phase,
  no container push → skip `quay_oidc` phase). If you add a new detected condition, wire the
  skip logic here, not just in the phase reference file.
- **Safety rules are non-negotiable** (see SKILL.md "Safety Rules"): never push upstream or
  delete branches without explicit user approval, commits must be signed off, GitHub Actions
  must be pinned to SHA digests.
- AIPCC-specific defaults (GitLab prefix, Quay org `aipcc-cicd`, GitHub orgs
  `opendatahub-io`/`red-hat-data-services`) are intentionally hardcoded in the template and
  SKILL.md — this skill is AIPCC-flavored, not a generic migration tool.

### Frontmatter linting

SKILL.md frontmatter is checked by an external linter ("skillsaw") run outside this repo.
YAML string values with special characters (e.g. `compatibility:`) must be quoted or the lint
fails — there's no local lint command to run this yourself; verify by inspection.

### Evals

`tests/gitlab-to-github/skills/gitlab-to-github/evals.json` defines eval scenarios (prompt,
expected output, and typed assertions: `file_check`, `content_check`, `output_check`,
`logic_check`, `flow_check`). These are run by an external eval harness, not from this repo —
when changing SKILL.md's flow or phase-skip logic, update the corresponding eval assertions in
the same change.

## Git Conventions

Commits in this repo use `Signed-off-by:` trailers (`git commit -s`) and reference AIPCC Jira
ticket IDs where applicable (see `git log` for examples).
