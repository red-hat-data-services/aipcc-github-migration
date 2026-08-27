# gitlab-to-github-migration

A Claude Code skill for guided migration of GitLab repositories to GitHub, with stateful progress tracking across sessions.

## What it does

Walks you through the full migration lifecycle in 5 phases:

1. **Repo Cleanup** — license, README, branch pruning, self-reference updates
2. **GitHub Setup** — create repo, push branches
3. **Quay OIDC** — federated identity for container pushes (skipped if no container builds)
4. **CI** — translate `.gitlab-ci.yml` to GitHub Actions (skipped if no CI pipeline)
5. **Mirroring** — GitLab pull mirror from GitHub so downstream `include:` refs keep working

The skill creates a `migration-manifest.yaml` that tracks progress, detected repo state, and phase/item completion. This manifest survives across sessions, so you can resume a migration exactly where you left off.

## Installation

Clone (or symlink) this repo into your project's `.claude/skills/` directory:

```bash
# Option A: git clone
git clone https://github.com/red-hat-data-services/aipcc-github-migration.git \
      your-project/.claude/skills/gitlab-to-github

# Option B: git submodule
git submodule add https://github.com/red-hat-data-services/aipcc-github-migration.git \
      your-project/.claude/skills/gitlab-to-github

# Option C: symlink (if you already have it cloned elsewhere)
ln -s /path/to/gitlab-to-github-migration \
      your-project/.claude/skills/gitlab-to-github
```

Then invoke the skill by telling Claude to migrate a repo, or type `/gitlab-to-github`.

## AIPCC defaults

The skill ships with defaults for the AIPCC productization team:

- GitLab prefix: `redhat/rhel-ai/ci-cd/`
- Quay org: `aipcc-cicd`
- Target GitHub org: `opendatahub-io`
- Maintainers list pre-populated

These are set in the manifest template and SKILL.md — adjust for your org.

## Repo structure

```
├── SKILL.md              # Main skill document (loaded by Claude Code)
├── references/           # Per-phase detailed guidance
│   ├── phase-1-cleanup.md
│   ├── phase-2-github-setup.md
│   ├── phase-3-quay-oidc.md
│   ├── phase-4-ci.md
│   └── phase-5-mirroring.md
└── templates/
    └── migration-manifest.yaml   # Manifest template
```

## License

Apache-2.0
