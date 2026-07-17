# GitLab to GitHub Migration: aipcc-claudio

## Init

No existing `migration-manifest.yaml` found. Starting a new migration.

**Repo:** aipcc-claudio
**GitHub org:** red-hat-data-services (default)

I've created the manifest at `./migration-manifest.yaml` with the AIPCC defaults:
- **Source:** `gitlab.com/redhat/rhel-ai/ci-cd/aipcc-claudio`
- **Target:** `github.com/red-hat-data-services/aipcc-claudio`
- **Quay:** `quay.io/aipcc-cicd/aipcc-claudio`

Running Phase 0 now...

---

## Phase 0: Pre-flight Assessment

I ran the automated checks against the GitLab repo. Here are the results:

```
Pre-flight Assessment for aipcc-claudio
----------------------------------------
CI pipeline:     yes
Container push:  yes
Multi-arch:      yes
Tekton/Konflux:  no
LICENSE:         missing
POLICY.md:       no
GitHub repo:     absent
Quay repo:       exists
Branches:        8 total (3 keep, 2 delete, 1 to review)
```

### Wave Classification: Wave 3 (complex)

This repo has a real CI pipeline that builds and pushes multi-arch container images. That puts it in **Wave 3** -- the most involved migration path. All six phases apply, and none can be skipped.

### Branch Classification

| Branch              | Action | Reason                                |
|---------------------|--------|---------------------------------------|
| `main`              | Keep   | Default branch                        |
| `rhoai-2.17`        | Keep   | Matches `rhoai-*` release pattern     |
| `rhoai-2.18`        | Keep   | Matches `rhoai-*` release pattern     |
| `renovate/something`| Delete | Matches `renovate/*` (bot-managed)    |
| `feature/old-thing` | Delete | Fully merged into main                |
| `feature/active-work`| Ask   | Not merged, no matching keep pattern  |

**Action needed:** Before we delete branches, I need you to confirm the classification above. In particular, what should we do with `feature/active-work`? Is it safe to delete, or does someone still need it?

### Skip Conditions Evaluated

| Condition                              | Result    | Effect                            |
|----------------------------------------|-----------|-----------------------------------|
| `has_license == true` + Apache-2.0     | NO        | LICENSE step NOT skipped           |
| `has_policy_md == true`                | NO        | POLICY.md step NOT skipped         |
| `github_state == "has_content"`        | NO        | GitHub push step NOT skipped       |
| `has_ci == false`                      | NO        | CI phase NOT skipped               |
| `has_container_push == false`          | NO        | Quay OIDC phase NOT skipped        |

All phases and items are active. Nothing is skipped.

---

## Phase 1: Repo Cleanup (Guidance)

All changes happen on a feature branch in the GitLab repo. Each change gets its own commit with `Signed-off-by:` trailer.

### Step-by-step plan

**1. Create a feature branch**

```bash
cd src/aipcc-claudio
git checkout main
git pull origin main
git checkout -b cleanup/migration-prep
```

**2. Add POLICY.md** [automatable]

Download the AIPCC commit and merge request guidelines Google Doc, convert to markdown, and update terminology ("merge request" becomes "pull request" since the doc will live on GitHub).

```bash
# After creating POLICY.md:
git add POLICY.md
git commit -s -m "docs: add POLICY.md with commit and merge request guidelines"
```

**3. Add LICENSE (Apache-2.0)** [automatable]

Write the standard Apache-2.0 license text to a `LICENSE` file.

```bash
git add LICENSE
git commit -s -m "docs: add Apache-2.0 license"
```

**4. Delete stale branches** [automatable, needs confirmation]

After you confirm the branch classification above, I'll delete:
- `renovate/something`
- `feature/old-thing`

```bash
# For each confirmed branch (URL-encode branch names with slashes):
glab api "projects/$PROJECT_ID/repository/branches/renovate%2Fsomething" --method DELETE
glab api "projects/$PROJECT_ID/repository/branches/feature%2Fold-thing" --method DELETE
```

**5. Update README** [automatable]

I'll scan the README for:
- GitLab CI badge URLs (remove or replace with GitHub Actions badges)
- `gitlab.com` clone URLs (update to `github.com`)
- References to "merge requests" (change to "pull requests")
- GitLab-specific instructions

```bash
git add README.md
git commit -s -m "docs: update README for GitHub migration"
```

**6. All commits use Signed-off-by** [automatable]

Every `git commit` in this phase uses the `-s` flag. Already reflected in the commands above.

**7. Submit MR for review** [human action required]

```bash
glab mr create --title "Migration prep: add license, policy, cleanup branches" \
  --description "Part of GitLab -> GitHub migration" \
  --assignee XiyangDong
```

You'll need to get the MR reviewed and merged before we proceed to Phase 2.

---

## What Comes After Phase 1

Once the cleanup MR is merged, the remaining phases are:

| Phase | What happens | Estimated time |
|-------|-------------|----------------|
| **Phase 2: GitHub Setup** | Create repo in `red-hat-data-services`, push main | 5 minutes |
| **Phase 3: Quay OIDC** | Set up federated auth for container pushes from GitHub Actions | 30 minutes |
| **Phase 4: CI** | Translate `.gitlab-ci.yml` to GitHub Actions (multi-arch build + push) | 1-2 sessions |
| **Phase 5: Mirroring** | Enable GitLab pull mirroring from GitHub, verify downstream CI | 5 minutes |
| **Phase 6: Announce** | Update references, notify team that GitHub is now canonical | 15 minutes |

Phase 4 (CI) will be the most involved step since you have multi-arch container builds. I'll translate the GitLab CI pipeline to GitHub Actions, set up OIDC-based Quay authentication, pin all actions to SHA digests, and configure concurrency groups and timeouts.

---

## Next Steps

To proceed, I need you to:

1. **Confirm the branch classification** -- especially what to do with `feature/active-work`
2. **Say "go"** and I'll start executing the Phase 1 items on the feature branch

The manifest is saved and will persist across sessions. If you stop now and come back later, I'll pick up exactly where we left off.
