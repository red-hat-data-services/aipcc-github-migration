# GitLab to GitHub Migration: product-management-configs

## Init

No existing `migration-manifest.yaml` found. Starting a new migration.

**Repo:** product-management-configs
**GitHub org:** red-hat-data-services (default)

I've created the manifest at `.claude/migrations/product-management-configs/manifest.yaml` with the AIPCC defaults:
- **Source:** `gitlab.com/redhat/rhel-ai/ci-cd/product-management-configs`
- **Target:** `github.com/red-hat-data-services/product-management-configs`
- **Quay:** `quay.io/aipcc-cicd/product-management-configs`

Running Phase 0 now...

---

## Phase 0: Pre-flight Assessment

I ran the automated checks against the GitLab repo. Here are the results:

```
Pre-flight Assessment for product-management-configs
─────────────────────────────────────────────────────
CI pipeline:     no
Container push:  no
Multi-arch:      no
Tekton/Konflux:  no
LICENSE:         GPL-3.0
POLICY.md:       no
GitHub repo:     absent
Quay repo:       missing
Self-references: 0 files
Branches:        3 total (1 keep, 0 delete, 2 to review)
```

### Wave Classification: Wave 1 (simple/no CI)

This repo has no CI pipeline and no container builds -- it's a config-only repo. That puts it in **Wave 1**, the simplest migration path. Phases 3 (Quay OIDC) and 4 (CI) are skipped entirely.

### License: GPL-3.0 -- Legal Review Required

> **WARNING:** This repo has a **GPL-3.0** license. The AIPCC standard is Apache-2.0, but
> GPL-3.0 cannot be automatically replaced -- this requires **legal review** to determine
> whether relicensing is permitted. The license cleanup item is flagged as a human action
> instead of automatable.
>
> **Action needed:** Check with Ken Dreyer or legal whether `product-management-configs`
> can be relicensed to Apache-2.0. If not, the existing GPL-3.0 license stays. Either way,
> this must be resolved before the cleanup MR is submitted.

### Branch Classification

| Branch           | Action | Reason                            |
|------------------|--------|-----------------------------------|
| `main`           | Keep   | Default branch                    |
| `feature-branch-1` | Ask | Not merged, no matching keep pattern |
| `feature-branch-2` | Ask | Not merged, no matching keep pattern |

**Action needed:** What should we do with the two feature branches? Are they safe to clean up post-migration, or does someone still need them?

### Skip Conditions Evaluated

| Condition                              | Result    | Effect                                          |
|----------------------------------------|-----------|--------------------------------------------------|
| `has_ci == false`                      | YES       | **Phase 4 (CI) SKIPPED** -- no pipeline to translate |
| `has_container_push == false`          | YES       | **Phase 3 (Quay OIDC) SKIPPED** -- no container push |
| `has_license == true` + Apache-2.0     | NO (GPL-3.0) | License step NOT skipped -- flagged for legal review |
| `has_policy_md == true`                | NO        | POLICY.md step NOT skipped                        |
| `github_state == "has_content"`        | NO        | GitHub push step NOT skipped                      |
| `gitlab_self_references` empty         | YES       | "Update GitLab self-references" step SKIPPED      |

### GitHub Repo Request

> GitHub repo does not exist yet.
>
> For red-hat-data-services: ask Ken Dreyer to create it.
>
> Submit the request now -- approval can take hours/days, and Phase 1 cleanup
> runs in parallel.

---

## Migration Status: product-management-configs
```
──────────────────────────────────────
Source:  gitlab.com/redhat/rhel-ai/ci-cd/product-management-configs
Target:  red-hat-data-services/product-management-configs
Ticket:  not set

Phase 1: Cleanup .............. pending  ← you are here
  · Add POLICY.md                        ← next step
  · License (GPL-3.0 — legal review)
  · Update README
  · Commit with Signed-off-by
  · MR submitted and merged
Phase 2: GitHub Setup ......... pending
Phase 3: Quay OIDC ............ skip (no container push)
Phase 4: CI ................... skip (no CI pipeline)
Phase 5: Mirroring ............ pending
Phase 6: Announce ............. pending
```

---

## Phase 1: Repo Cleanup (Guidance)

All changes happen on a feature branch in the GitLab repo. Each change gets its own commit with `Signed-off-by:` trailer.

### Step-by-step plan

**1. Create a feature branch**

```bash
cd src/product-management-configs
git checkout main
git pull origin main
git checkout -b cleanup/migration-prep
```

**2. Add POLICY.md** [automatable]

Copy the bundled AIPCC commit and merge request guidelines into `POLICY.md`, update terminology ("merge request" becomes "pull request" since the doc will live on GitHub).

```bash
# After creating POLICY.md:
git add POLICY.md
git commit -s -m "docs: add POLICY.md with commit and merge request guidelines"
```

**3. License (GPL-3.0 -- legal review needed)** [human action required]

This repo currently has a GPL-3.0 license. Do NOT auto-replace with Apache-2.0. You need to:

1. Confirm with Ken Dreyer or legal whether relicensing to Apache-2.0 is approved
2. If approved: replace the LICENSE file with Apache-2.0 text and commit
3. If not approved: keep GPL-3.0, update the manifest, and move on

```bash
# Only after legal approval:
git add LICENSE
git commit -s -m "docs: replace GPL-3.0 license with Apache-2.0"
```

**4. Update README** [automatable]

I'll scan the README for:
- GitLab CI badge URLs (remove or replace with GitHub Actions badges)
- `gitlab.com` clone URLs (update to `github.com`)
- References to "merge requests" (change to "pull requests")
- GitLab-specific instructions

```bash
git add README.md
git commit -s -m "docs: update README for GitHub migration"
```

**5. (Skipped) Update GitLab self-references**

No self-referencing GitLab paths detected -- nothing to update.

**6. All commits use Signed-off-by** [automatable]

Every `git commit` in this phase uses the `-s` flag. Already reflected in the commands above.

**7. Submit MR for review** [human action required]

```bash
glab mr create --title "Migration prep: add license, policy, update README" \
  --description "Part of GitLab → GitHub migration (AIPCC-16303)" \
  --assignee XiyangDong
```

You'll need to get the MR reviewed and merged before we proceed to Phase 2.

---

## What Comes After Phase 1

Once the cleanup MR is merged, the remaining phases are:

| Phase | What happens | Estimated time |
|-------|-------------|----------------|
| **Phase 2: GitHub Setup** | Create repo in `red-hat-data-services`, push main | 5 minutes |
| ~~Phase 3: Quay OIDC~~ | *Skipped -- no container push* | -- |
| ~~Phase 4: CI~~ | *Skipped -- no CI pipeline* | -- |
| **Phase 5: Mirroring** | Enable GitLab pull mirroring from GitHub, verify downstream CI | 5 minutes |
| **Phase 6: Announce** | Update references, notify team that GitHub is now canonical | 15 minutes |

Since this is a config-only repo with no CI or container builds, the migration is straightforward. After cleanup, the active path is: **cleanup -> github_setup -> mirroring -> announce**. Total estimated time after Phase 1: under 30 minutes.

---

## Next Steps

To proceed, I need you to:

1. **Resolve the GPL-3.0 license question** -- check with Ken/legal on whether relicensing to Apache-2.0 is approved
2. **Confirm the branch classification** -- what to do with the two feature branches
3. **Request the GitHub repo** from Ken Dreyer (for red-hat-data-services org)
4. **Say "go"** and I'll start executing the Phase 1 items on the feature branch

The manifest is saved and will persist across sessions. If you stop now and come back later, I'll pick up exactly where we left off.
