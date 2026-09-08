# Phase 1: Repo Cleanup (on GitLab)

All changes happen on a feature branch in the GitLab repo. One commit per change.

## Items

- **[automatable] Add LICENSE (Apache-2.0)** — Not always required. Skip automatically if Phase 0 detected an existing Apache-2.0 license. Otherwise, ask the user whether to add one — see "Deciding on the license" below. Commit separately.
- **[automatable] Update README** — Fix any GitLab-specific URLs or workflow references that won't apply post-migration.
- **[automatable] Update GitLab self-references** — Replace `gitlab>REPO_PATH` preset references, `local>` paths, and similar with their GitHub equivalents. Skip if `detected.gitlab_self_references` is empty. Show diff and ask before changing cross-platform references.

> **Note:** Branch cleanup on GitLab is deferred to post-migration. Since GitHub becomes canonical via mirroring, only the branches pushed to GitHub matter. Stale GitLab-only branches are cosmetic noise on a read-only mirror.
- **[automatable] Commit with Signed-off-by** — All commits in this phase must use `git commit -s` to add the `Signed-off-by:` trailer.
- **[human] MR submitted and merged** — User opens the MR on GitLab, gets review, and merges. Wait for this before proceeding.

## Automation Details

### Creating a feature branch

```bash
cd src/<repo>
git checkout main
git pull origin main
git checkout -b cleanup/migration-prep
```

### Deciding on the license

If `detected.has_license` is false (or the existing license isn't Apache-2.0), don't add it
automatically — ask the user first:

```
No Apache-2.0 LICENSE detected. Add one?
  - Add Apache-2.0 (standard for AIPCC upstream/community repos)
  - Skip (internal-only repo, license handled elsewhere, or pending legal review)
```

Record the answer in the manifest as `license_decision: add` or `license_decision: skip` so
re-runs of the skill don't ask again. If `skip`, mark the item `status: skipped` with the user's
stated reason.

### Adding LICENSE

Only run this if `license_decision == "add"`.

```bash
# Write the Apache-2.0 license text to LICENSE
git add LICENSE
git commit -s -m "docs: add Apache-2.0 license"
```

### Updating README

Look for:
- GitLab CI badge URLs → remove or replace with GitHub Actions badges
- `gitlab.com` clone URLs → update to `github.com`
- References to MRs → change to PRs
- GitLab-specific instructions

```bash
git add README.md
git commit -s -m "docs: update README for GitHub migration"
```

### Updating GitLab self-references

Skip if `detected.gitlab_self_references` is empty. Otherwise, for each file in the list, replace
GitLab-style references with their GitHub equivalents. Common patterns:

| GitLab pattern | GitHub equivalent |
|---|---|
| `gitlab>GITLAB_PATH//file.json` | `github>GITHUB_ORG/REPO//file.json` |
| `local>GITLAB_PATH` | `local>GITHUB_ORG/REPO` |
| `gitlab.com/GITLAB_PATH` | `github.com/GITHUB_ORG/REPO` |
| `project: 'GITLAB_PATH'` (in CI includes) | n/a — handled in Phase 4 CI translation |

Show the user a diff of each file before committing. Some references may be intentionally
cross-platform (e.g., a GitLab-hosted preset consumed by GitLab-side repos) — ask before changing
those.

#### Flags to raise before rewriting

**`local>` vs `gitlab>` are different protocols.** `gitlab>` tells the tool to fetch from GitLab
specifically. `local>` means "same platform I'm running on." Before rewriting `local>` paths,
ask the user which platform the consuming tool (e.g., Renovate) targets post-migration. If the
tool still reads from GitLab (via mirror), `local>` paths should keep the GitLab org/path.
If the tool moves to GitHub, update to the GitHub org/repo.

**Downstream blast radius.** If this repo is consumed as a shared preset or library by other
repos, changing `gitlab>` to `github>` here will break every consumer still pointing to the old
`gitlab>` path. Flag this to the user:

```
⚠ This repo appears to be a shared preset/config consumed by other repositories.
  Changing gitlab> references to github> will require downstream repos to update
  their references too. Coordinate with consuming teams before rewriting.
```

Detect shared-preset repos by checking for Renovate `extends` patterns, npm/PyPI package
references, or CI `include: project:` pointing to this repo from other projects.

```bash
git add <changed files>
git commit -s -m "chore: update self-references from GitLab to GitHub"
```

### Opening the MR

```bash
glab mr create --title "Migration prep: add license, policy, and README updates" \
  --description "Part of GitLab → GitHub migration"
```

## Gotchas

- **Separate commits**: Each change gets its own commit. Don't squash cleanup into one commit — reviewers expect granular changes.
- **Branch naming**: Use `cleanup/migration-prep` or similar — makes the MR purpose obvious.
- **GPL-licensed repos**: Two repos (product-management-tool, product-management-configs) are GPL-3.0. These may need legal review before changing to Apache-2.0. If `detected.license_type` is `GPL-3.0`, flag this to the user instead of auto-replacing.
- **License is optional now**: Don't assume Apache-2.0 gets added by default — always ask when no license is detected, and respect `license_decision: skip`.

## Troubleshooting

### MR creation fails with 403
You need Maintainer+ access on the GitLab project. Ask the repo owner (check CODEOWNERS or the Jira ticket) to grant access.
