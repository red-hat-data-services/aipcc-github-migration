# Phase 0: Pre-flight Assessment

Automated scan — no user input needed. Populate the `detected.*` fields in the manifest.

The repo must be cloned locally (e.g., as a git submodule in `src/`). Pull latest before scanning.

```bash
# Locate the repo — check src/<repo>, or ask the user
REPO_DIR="src/$REPO_NAME"

# Pull latest
git -C "$REPO_DIR" checkout main
git -C "$REPO_DIR" pull origin main

# File checks
test -f "$REPO_DIR/.gitlab-ci.yml"    # has_ci
test -f "$REPO_DIR/LICENSE"           # has_license (parse for type)
test -d "$REPO_DIR/.tekton"           # has_tekton

# CODEOWNERS can live in any of these locations (GitLab and GitHub both support root;
# GitLab also checks .gitlab/ and docs/)
for p in CODEOWNERS .gitlab/CODEOWNERS docs/CODEOWNERS; do
  test -f "$REPO_DIR/$p" && echo "$p"  # has_codeowners / codeowners_path
done

# Branch list
git -C "$REPO_DIR" branch -r --list 'origin/*' | sed 's|origin/||'
```

**External checks** (always run):

```bash
# GitHub repo state
gh repo view "$TARGET_GITHUB_REPO" --json isEmpty 2>/dev/null

# Quay repo exists? (only if has_container_push is true)
curl -sf "https://quay.io/api/v1/repository/$QUAY_ORG/$REPO_NAME" >/dev/null 2>&1
```

**If `.gitlab-ci.yml` exists**, parse it for:
- `buildah push` / `podman push` / `skopeo copy` → `detected.has_container_push: true`
- `--platform` / `--manifest` / architecture matrix → `detected.has_multi_arch: true`
- `include: project:` → populate `detected.includes_from[]`

**If `has_container_push` is true**, populate the Quay fields in the manifest:
- `quay_org`: `aipcc-cicd` (AIPCC default — ask user to confirm)
- `quay_repo`: `<quay_org>/<repo>`
- Run the Quay check: `curl -sf "https://quay.io/api/v1/repository/$QUAY_ORG/$REPO_NAME"`
- Set `detected.quay_repo_exists` accordingly

**Scan for self-referencing GitLab paths** in all non-binary files:
```bash
# Look for gitlab> preset references, GitLab API URLs, or project paths pointing to this repo
grep -rl "gitlab>.*$REPO_NAME\|gitlab\.com/.*$SOURCE_GITLAB" . \
  --include="*.json" --include="*.yaml" --include="*.yml" --include="*.md" \
  --exclude-dir=.git
```
Populate `detected.gitlab_self_references[]` with the matched file paths.

**Check for shared-preset/config pattern** — if the repo contains Renovate presets (`extends`
patterns in JSON files), npm packages, PyPI packages, or CI templates consumed by other repos,
set `detected.is_shared_preset: true`. This flags downstream coordination needs in Phase 1
(self-reference rewriting).

**Branch classification** (informational — branch cleanup happens post-migration):
- **Keep**: `main`, `release-*`, `rhel-*`, `rhoai-*`
- **Stale**: `renovate/*`, branches fully merged into main
- **Ask**: everything else

Update the manifest with all detected values and write it back.

**Present results** to user as a summary table:

```
Pre-flight Assessment for <repo>
────────────────────────────────
CI pipeline:     yes/no
Container push:  yes/no
Multi-arch:      yes/no
Tekton/Konflux:  yes/no
LICENSE:         Apache-2.0 / MIT / missing
CODEOWNERS:      present (<path>) / missing
GitHub repo:     absent / empty / has content
Quay repo:       exists / missing
Self-references:  N files with gitlab> paths (list them)
Shared preset:   yes/no (if yes, downstream repos need coordinated updates)
Branches:        N total (K keep, D delete, A to review)
```

**If `github_state == "absent"`**, prompt the user to request the repo now so approval
overlaps with Phase 1 cleanup:

```
⚠ GitHub repo does not exist yet.

Submit the repo request Google Form
  (ask the team chat if you don't have the URL yet).

Submit now — approval can take hours/days, and Phase 1 cleanup
runs in parallel.
```

Then proceed to Phase 1.
