# Migrating aipcc-claudio from GitLab to GitHub

Absolutely. Since this repo has a real CI pipeline with container builds, it falls into "Wave 2" based on the patterns we established with central-linter. Here is what needs to happen, broken into phases.

## Before We Start: Investigation

I need to look at a few things in the aipcc-claudio repo before we commit to a plan:

1. **Examine the GitLab CI config** (`.gitlab-ci.yml`) to understand what the pipeline does -- build targets, container push destinations, any multi-arch builds, any `include: project:` references to or from other repos.
2. **Check for existing license and policy files** -- do we already have Apache-2.0 or something else?
3. **Identify stale branches** to clean up before migration.
4. **Check if the GitHub repo already exists** in opendatahub-io or red-hat-data-services.
5. **Check if the Quay repo exists** at `quay.io/aipcc-cicd/aipcc-claudio`.

Can you point me at the GitLab URL? I am assuming `gitlab.com/redhat/rhel-ai/ci-cd/aipcc-claudio` based on the team's standard path, but confirm if it is different.

## Migration Plan (6 Phases)

### Phase 1: Repo Cleanup (on GitLab)

All done on a single feature branch, one commit per change, each with `Signed-off-by:` tag.

- **Add `POLICY.md`** -- download the AIPCC commit/MR guidelines Google Doc as markdown, update terminology (MR becomes PR, etc.)
- **Add `LICENSE`** -- Apache-2.0 (standard for opendatahub-io)
- **Delete stale branches** -- we will identify which ones after investigation
- **Update README** -- remove/replace any GitLab-specific URLs or workflow references

Then open a merge request on GitLab and get it reviewed (likely Xiang or whoever owns this repo). This must merge before we proceed.

### Phase 2: GitHub Repo Setup

Once cleanup is merged:

```bash
# Add GitHub remote
git remote add github git@github.com:opendatahub-io/aipcc-claudio.git

# Sync from GitLab
git fetch origin
git merge --ff-only origin/main

# Push to GitHub
git push github main
```

The repo must already exist in the target GitHub org. If it does not, that requires the repo request form (same one we used for central-linter -- org, license, maintainer list). Ken has the process down.

### Phase 3: Quay OIDC Federation

This is the passwordless auth setup so GitHub Actions can push container images to `quay.io/aipcc-cicd/aipcc-claudio` without storing secrets.

Steps:
1. **Create or verify Quay repo** under `quay.io/aipcc-cicd/`
2. **Create an org-level robot account** -- e.g., `aipcc-cicd+aipcc_claudio_github` (must be org-level, not personal -- federation does not work on personal robots)
3. **Grant the robot Write permission** on the Quay repo
4. **Get the numeric subject claim** from the GitHub repo settings (format: `repo:opendatahub-io@<ORG_ID>/aipcc-claudio@<REPO_ID>:ref:refs/heads/main`) -- this must use the immutable numeric ID format, not human-readable names
5. **Configure federation** on the robot account in Quay (Issuer: `https://token.actions.githubusercontent.com`, Subject: the claim from step 4)
6. **Add `.github/scripts/quay-oidc-login.sh`** to the repo

Key gotcha from central-linter: OIDC errors from Quay are opaque. If authentication fails, decode the JWT to inspect the actual subject claim GitHub is sending and compare it to what Quay has configured.

### Phase 4: GitHub Actions CI

This is the biggest phase since aipcc-claudio has a real build pipeline. We need to translate the GitLab CI into GitHub Actions.

**What we will create:**

- `.github/workflows/ci.yml` -- main workflow with build, lint, test, and push jobs
- `.github/scripts/` -- modularized shell scripts (build.sh, load-image.sh, test.sh, push.sh, quay-oidc-login.sh)
- `.github/dependabot.yml` -- automated action SHA updates (opendatahub-io uses Dependabot, not Renovate)

**CI translation rules learned from central-linter:**

| GitLab | GitHub Actions | Notes |
|--------|---------------|-------|
| `CI_MERGE_REQUEST_*` | `github.event.pull_request.*` | Different env var names entirely |
| `CI_COMMIT_SHORT_SHA` (8 chars) | `GITHUB_SHA` (40 chars) | Downstream consumers expecting 8 chars will break silently |
| `bash -c` in containers | same, but... | Does NOT inherit `set -euo pipefail` -- set it explicitly inside the block |
| Vault-based Quay creds | OIDC federation | No secrets to manage |
| `include: project:` | N/A on GitHub side | Handled by GitLab mirroring (Phase 5) |
| `rules:` | `on:` triggers + `if:` conditions | Different syntax, same concepts |

**Best practices we are applying:**

- Pin all actions to SHA commits, not version tags (supply chain safety)
- Scope `id-token: write` to the push job only, not the whole workflow
- Set concurrency groups to prevent duplicate builds on rapid pushes
- Add `timeout-minutes` to every job
- Modularize CI logic into `.github/scripts/` instead of inline `run:` blocks

### Phase 5: GitLab Pull Mirroring

After GitHub Actions CI is working:

```bash
# Get the GitLab project ID
glab api projects/redhat%2Frhel-ai%2Fci-cd%2Faipcc-claudio --method GET | jq '.id'

# Enable pull mirroring
glab api projects/<PROJECT_ID> --method PUT \
  -f mirror=true \
  -f import_url="https://github.com/opendatahub-io/aipcc-claudio.git" \
  -f mirror_trigger_builds=true
```

This ensures any downstream GitLab repos using `include: project:` to reference aipcc-claudio CI templates (if any) keep working. GitLab pulls from GitHub on a ~5 minute interval.

### Phase 6: Announce

- Update README with GitHub URL as the canonical source
- Notify the team: GitLab is now mirror-only, all new work goes to GitHub
- Update any external references (Jira tickets, Confluence docs, etc.)

## Timing Estimate

Based on central-linter:

| Phase | Estimate |
|-------|----------|
| Repo cleanup + MR | 1-2 sessions |
| GitHub repo + push | 5 minutes (assuming repo already created) |
| Quay OIDC federation | 30 minutes (longer if debugging subject claims) |
| GitHub Actions CI | 1-2 sessions (most of the effort, since this repo has real CI) |
| GitLab mirroring | 5 minutes |
| Announcement | 15 minutes |

## What I Need From You to Start

1. **Confirm the GitLab path** -- is it `gitlab.com/redhat/rhel-ai/ci-cd/aipcc-claudio`?
2. **Target GitHub org** -- `opendatahub-io` or `red-hat-data-services`?
3. **Has the GitHub repo already been created?** If not, we need to submit the repo request form first.
4. **Has the Jira ticket been created?** We should have a ticket under AIPCC-16303 for this migration.
5. **Who should review the cleanup MR?** (Xiang? Someone else who owns this repo?)

Once you confirm those details, I will clone the repo as a submodule, inspect the CI configuration, and start Phase 1 (cleanup). I can automate most of the cleanup commits and draft the GitHub Actions workflow based on what I find in the `.gitlab-ci.yml`.
