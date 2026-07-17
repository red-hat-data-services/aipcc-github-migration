# Phase 1: Repo Cleanup (on GitLab)

All changes happen on a feature branch in the GitLab repo. One commit per change.

## Items

- **[automatable] Add POLICY.md** — Download the Google Doc guidelines as markdown, update terminology (MR → PR). Commit separately.
- **[automatable] Add LICENSE (Apache-2.0)** — Standard Apache-2.0 text. Commit separately. Skip if Phase 0 detected an existing Apache-2.0 license.
- **[automatable] Update README** — Fix any GitLab-specific URLs or workflow references that won't apply post-migration.
- **[automatable] Update GitLab self-references** — Replace `gitlab>REPO_PATH` preset references, `local>` paths, and similar with their GitHub equivalents. Skip if `detected.gitlab_self_references` is empty. Show diff and ask before changing cross-platform references.

> **Note:** Branch cleanup on GitLab is deferred to post-announce. Since GitHub becomes canonical via mirroring, only the branches pushed to GitHub matter. Stale GitLab-only branches are cosmetic noise on a read-only mirror.
- **[automatable] Commit with Signed-off-by** — All commits in this phase must use `git commit -s` to add the `Signed-off-by:` trailer.
- **[human] MR submitted and merged** — User opens the MR on GitLab, gets review (typically from Xiang/XiyangDong for central-linter), and merges. Wait for this before proceeding.

## Automation Details

### Creating a feature branch

```bash
cd src/<repo>
git checkout main
git pull origin main
git checkout -b cleanup/migration-prep
```

### Adding POLICY.md

Copy the bundled `templates/POLICY.md` from this skill into the repo root. The file is the AIPCC commit/merge guidelines, already in markdown with PR terminology.

```bash
# After creating POLICY.md:
git add POLICY.md
git commit -s -m "docs: add POLICY.md with commit and merge request guidelines"
```

### Adding LICENSE

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

```bash
git add <changed files>
git commit -s -m "chore: update self-references from GitLab to GitHub"
```

### Opening the MR

```bash
glab mr create --title "Migration prep: add license, policy, cleanup branches" \
  --description "Part of GitLab → GitHub migration (AIPCC-18266)" \
  --assignee XiyangDong
```

## Gotchas

- **Separate commits**: Each change gets its own commit. Don't squash cleanup into one commit — reviewers expect granular changes.
- **Branch naming**: Use `cleanup/migration-prep` or similar — makes the MR purpose obvious.
- **POLICY.md terminology**: The Google Doc uses "merge request" — update to "pull request" since the doc will live on GitHub.
- **GPL-licensed repos**: Two repos (product-management-tool, product-management-configs) are GPL-3.0. These may need legal review before changing to Apache-2.0. If `detected.license_type` is `GPL-3.0`, flag this to the user instead of auto-replacing.

## Troubleshooting

### MR creation fails with 403
You need Maintainer+ access on the GitLab project. Ask the repo owner (check CODEOWNERS or the Jira ticket) to grant access.
