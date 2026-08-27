# Phase 4: GitHub Actions CI

Translate the GitLab CI pipeline to GitHub Actions. Skip this entire phase if `detected.has_ci == false`.

## Items

- **[automatable] Create .github/workflows/ci.yml** — Translate the GitLab CI config to GitHub Actions workflow.
- **[automatable] Create .github/scripts/** — Modularize CI logic into shell scripts instead of inline `run:` blocks.
- **[automatable] Pin actions to SHA** — All `uses:` references must use SHA digests, not version tags.
- **[automatable] Add dependabot.yml** — opendatahub-io uses Dependabot (not Renovate) for automated action SHA updates.
- **[automatable] Scope id-token to push job** — `id-token: write` permission only on the job that pushes to Quay, not workflow-level.
- **[automatable] Set concurrency groups** — Prevent duplicate builds on the same branch/PR.
- **[automatable] Add timeout-minutes** — Every job gets a timeout to prevent hung builds.
- **[automatable] Verify CI passes** — Push to GitHub, watch the workflow run, confirm green.

## Automation Details

### Reading the GitLab CI config

```bash
# Fetch the raw .gitlab-ci.yml
glab api "projects/$PROJECT_ID/repository/files/.gitlab-ci.yml?ref=main" --method GET | jq -r '.content' | base64 -d
```

Parse it to understand:
- What stages exist (build, test, lint, push)
- What container images are used
- What scripts are run
- What artifacts are produced
- Whether there are `include: project:` references (cross-repo dependencies)

### Workflow structure

A typical translated workflow:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  IMAGE_NAME: quay.io/<quay_org>/<repo>
  QUAY_ROBOT_USER: <quay_org>+<repo>_github

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@<SHA>
      - name: Build
        run: .github/scripts/build.sh

  test:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    needs: build
    steps:
      - uses: actions/checkout@<SHA>
      - name: Test
        run: .github/scripts/test.sh

  push:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    timeout-minutes: 15
    needs: [build, test]
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@<SHA>
      - name: Login to Quay via OIDC
        run: .github/scripts/quay-oidc-login.sh
      - name: Push image
        run: .github/scripts/push.sh
```

### Script modularization

Create scripts in `.github/scripts/` for each CI step. This keeps the workflow YAML readable and the bash testable.

```bash
mkdir -p .github/scripts
# build.sh, test.sh, push.sh, etc.
chmod +x .github/scripts/*.sh
```

Every script starts with:
```bash
#!/usr/bin/env bash
set -euo pipefail
```

### Pinning actions to SHA

Find the current SHA for each action:
```bash
gh api repos/actions/checkout/git/ref/tags/v4 --jq '.object.sha'
```

Use the full SHA in `uses:`:
```yaml
- uses: actions/checkout@<full-40-char-sha>  # v4
```

### Adding dependabot.yml

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

### Verifying CI

```bash
git add .github/
git commit -s -m "ci: add GitHub Actions workflow"
git push github main

# Watch the run
gh run list --repo <target_org>/<repo> --limit 1
gh run watch <RUN_ID> --repo <target_org>/<repo>
```

## CI Translation Reference

### Environment variable mapping

| GitLab | GitHub Actions | Notes |
|--------|---------------|-------|
| `CI_COMMIT_SHORT_SHA` | `${GITHUB_SHA::8}` | GitLab = 8 chars, GitHub SHA = 40 chars |
| `CI_COMMIT_SHA` | `$GITHUB_SHA` | Full SHA |
| `CI_MERGE_REQUEST_IID` | `${{ github.event.pull_request.number }}` | Only in PR context |
| `CI_MERGE_REQUEST_SOURCE_BRANCH_NAME` | `${{ github.head_ref }}` | Only in PR context |
| `CI_PIPELINE_ID` | `${{ github.run_id }}` | |
| `CI_JOB_NAME` | `${{ github.job }}` | |
| `CI_PROJECT_DIR` | `${{ github.workspace }}` | |

### GitHub Actions secrets and variables

The OIDC approach avoids stored secrets for Quay authentication — `id-token: write` handles it at runtime. Most migrated workflows need no repository secrets at all.

If the GitLab CI uses project-level variables (e.g., API tokens, registry credentials), the user must add them manually via **GitHub repo → Settings → Secrets and variables → Actions**. List which variables are needed and tell the user to set them up before the verification step.

### Things to exclude

- **GitLab-specific CI targets** like `linter-mr-commit` that only make sense in GitLab
- **Renovate configuration** — opendatahub-io uses Dependabot
- **`rules:` with `CI_MERGE_REQUEST_*`** — translate to `on: pull_request` triggers

### Container execution

GitLab's `image:` keyword runs the entire job in a container. GitHub Actions equivalent:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    container:
      image: registry.example.com/my-image:tag
    steps:
      - run: echo "running inside the container"
```

## Gotchas

- **`bash -c` blocks** in containers don't inherit `set -euo pipefail` — set it explicitly inside the block
- **SHA format differences**: `GITHUB_SHA` is 40 chars. If downstream consumers expect 8-char short SHAs, use `${GITHUB_SHA::8}`
- **`buildah pull` from OCI archives**: capture the image ID from stdout, don't assume a tag format
- **id-token scope**: `id-token: write` on the push job only, never workflow-level. Other jobs shouldn't request OIDC tokens.
- **Concurrency groups**: Without them, multiple pushes to main can trigger parallel builds that race on Quay image tags

## Troubleshooting

### Workflow doesn't trigger
Check the `on:` trigger configuration. Common issue: the workflow file must be on the default branch (main) for `push` triggers to work.

### Container image not found
GitLab CI images may be in registries that require authentication. For GitHub Actions, either use public images or configure `docker login` in a setup step.

### `set -euo pipefail` errors in container
When using `container:` in GitHub Actions, the shell may default to `sh`, not `bash`. Specify the shell explicitly:
```yaml
steps:
  - run: |
      set -euo pipefail
      # your commands
    shell: bash
```
