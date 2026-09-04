# aipcc-github-migration

A Claude Code plugin marketplace for AIPCC. It currently hosts one plugin, `gitlab-to-github`: a
guided migration skill for moving GitLab repositories to GitHub, with stateful progress tracking
across sessions.

## What it does

Walks you through the full migration lifecycle in 6 phases:

1. **Repo Cleanup** — license, README, branch pruning, self-reference updates
2. **GitHub Setup** — create repo, push branches
3. **Konflux/KRD Reconfiguration** — Konflux Component/ImageRepository resources in `konflux-release-data` for Tekton-managed container build repos (skipped unless confirmed Konflux-managed)
4. **Quay OIDC** — federated identity for container pushes (skipped if no container builds)
5. **CI** — translate `.gitlab-ci.yml` to GitHub Actions (skipped if no CI pipeline)
6. **Mirroring** — GitLab pull mirror from GitHub so downstream `include:` refs keep working

The skill creates a `migration-manifest.yaml` that tracks progress, detected repo state, and phase/item completion. This manifest survives across sessions, so you can resume a migration exactly where you left off.

## Installation

Add this repo as a plugin marketplace, then install the plugin:

```
/plugin marketplace add red-hat-data-services/aipcc-github-migration
/plugin install gitlab-to-github@aipcc-github-migration
```

Then invoke the skill by telling Claude to migrate a repo, or type `/gitlab-to-github`.

## AIPCC defaults

The skill ships with defaults for the AIPCC productization team:

- GitLab prefix: `redhat/rhel-ai/ci-cd/`
- Quay org: `aipcc-cicd`
- Target GitHub org: `opendatahub-io` (upstream/community) or `red-hat-data-services` (downstream/internal), asked at init
- Maintainers list pre-populated

These are set in the manifest template and SKILL.md — adjust for your org.

## Repo structure

```
├── .claude-plugin/
│   └── marketplace.json                    # Marketplace manifest — lists plugins and their source paths
├── src/gitlab-to-github/
│   ├── .claude-plugin/plugin.json          # Plugin manifest
│   ├── README.md                           # Plugin-level overview
│   └── skills/
│       ├── gitlab-to-github/
│       │   ├── SKILL.md                    # Main skill document (loaded by Claude Code)
│       │   ├── references/                 # Per-phase detailed guidance, loaded on demand
│       │   │   ├── phase-0-preflight.md
│       │   │   ├── phase-1-cleanup.md
│       │   │   ├── phase-2-github-setup.md
│       │   │   ├── phase-3-konflux-krd.md
│       │   │   ├── phase-4-quay-oidc.md
│       │   │   ├── phase-5-ci.md
│       │   │   └── phase-6-mirroring.md
│       │   └── templates/
│       │       └── migration-manifest.yaml # Manifest template
│       └── ldap/
│           └── SKILL.md                    # Red Hat LDAP lookups, used for CODEOWNERS translation
└── tests/gitlab-to-github/skills/gitlab-to-github/
    └── evals.json                          # Eval scenario definitions
```

## License

Apache-2.0
