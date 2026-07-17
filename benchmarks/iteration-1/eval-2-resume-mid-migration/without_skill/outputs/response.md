## Dashboard Migration Status

**Repo:** `dashboard` (AIPCC-18270, wave 3)
**Source:** `gitlab.com/redhat/rhel-ai/ci-cd/dashboard`
**Target:** `opendatahub-io/dashboard`

**Completed phases:**
- Cleanup: done (POLICY.md, LICENSE, stale branches, README, MR merged)
- GitHub setup: done (repo already had content, remote added)

**Next phase: Quay OIDC Federation (Phase 3)**

Since `detected.has_container_push` is `true`, the repo pushes container images to `quay.io/aipcc-cicd/dashboard`, so you need OIDC federation set up before CI can push from GitHub Actions. Here is the walkthrough of Phase 3, item by item:

---

### Phase 3 Items

**1. Create Quay repo** (human)
The manifest shows `detected.quay_repo_exists: true`, so this step can be **skipped** -- the repo at `quay.io/aipcc-cicd/dashboard` already exists.

**2. Create org-level robot account** (automatable)
Create the robot `aipcc-cicd+dashboard_github` via the Quay API:
```bash
curl -sf -X PUT \
  -H "Authorization: Bearer $QUAY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"description": "GitHub Actions OIDC push for dashboard"}' \
  "https://quay.io/api/v1/organization/aipcc-cicd/robots/dashboard_github"
```
Note: robot names use underscores, not hyphens (Quay restriction).

**3. Grant robot write permission** (automatable)
```bash
curl -sf -X PUT \
  -H "Authorization: Bearer $QUAY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role": "write"}' \
  "https://quay.io/api/v1/repository/aipcc-cicd/dashboard/permissions/user/aipcc-cicd+dashboard_github"
```

**4. Get numeric subject claim** (human -- you need to do this)
Go to the GitHub repo settings for `opendatahub-io/dashboard` and find the immutable subject claim string. The format is:
```
repo:opendatahub-io@<ORG_NUMERIC_ID>/dashboard@<REPO_NUMERIC_ID>:ref:refs/heads/main
```
The numeric org and repo IDs are visible on the GitHub repo settings page. The human-readable format (`repo:opendatahub-io/dashboard:ref:...`) will silently fail -- you must use the numeric ID version.

**5. Configure robot federation** (human -- you need to do this in Quay UI)
1. Go to `quay.io/organization/aipcc-cicd` -> Robot Accounts
2. Click the `dashboard_github` robot -> Federation tab
3. If no Federation tab is visible, enable the v2 UI in Quay organization settings
4. Add a federation entry:
   - Issuer: `https://token.actions.githubusercontent.com`
   - Subject: the claim string from step 4

**6. Add quay-oidc-login.sh** (automatable)
Write the OIDC login script to `.github/scripts/quay-oidc-login.sh` in the dashboard repo. This script exchanges a GitHub Actions OIDC token for a Quay bearer token and logs in with buildah. Reference: `github.com/ktdreyer/quay-oidc-demo/blob/main/quay-oidc.md`

**7. Verify push succeeds** (automatable, but depends on CI being set up)
This item is really a Phase 4 dependency -- you can only verify the push after the GitHub Actions workflow exists. It is fine to defer verification until after Phase 4 (CI) is complete.

---

### After Phase 3: Phase 4 (GitHub Actions CI)

Since `detected.has_ci` is `true`, you will need to translate the GitLab CI pipeline to GitHub Actions. The manifest has 8 items for this phase:

1. Create `.github/workflows/ci.yml` -- translate from `.gitlab-ci.yml`
2. Create `.github/scripts/` -- modularize CI logic into shell scripts
3. Pin all actions to SHA digests (not version tags)
4. Add `.github/dependabot.yml` (opendatahub-io uses Dependabot, not Renovate)
5. Scope `id-token: write` to the push job only (not workflow-level)
6. Set concurrency groups to prevent duplicate builds
7. Add `timeout-minutes` to every job
8. Verify CI passes end-to-end

Key detail: `detected.has_tekton: true` -- check whether the Tekton config needs to be migrated or if it is GitLab-specific and can be dropped.

### After Phase 4: Phase 5 (Mirroring)

Set up GitLab pull mirroring so the GitLab repo mirrors from GitHub. This is important because the manifest shows `included_by: [redhat/rhel-ai/ci-cd/rhaiis/pipeline]` -- there is a downstream GitLab project that includes CI from this repo. The mirroring ensures those downstream `include: project:` references keep working.

### After Phase 5: Phase 6 (Announce)

Update the README with a notice that GitHub is canonical, notify the team, and update Jira/Confluence references.

---

### Recommended Order of Operations

1. **Now:** Start Phase 3 (Quay OIDC). Steps 1-3 are API calls you can run immediately if you have a `$QUAY_API_TOKEN`. Steps 4-5 require manual work in the GitHub and Quay UIs.
2. **Next session:** Phase 4 (CI). This is typically 1-2 sessions of work to translate the GitLab CI pipeline.
3. **Quick follow-up:** Phase 5 (mirroring, ~5 min) and Phase 6 (announce, ~15 min).

### Things to Watch Out For

- **Robot naming**: `dashboard_github` (underscore), not `dashboard-github`. Quay robot names cannot contain hyphens.
- **Immutable subject claims**: Must use the numeric ID format. The human-readable format will silently fail.
- **v2 UI**: The Federation tab may not appear in the classic Quay UI. Toggle v2 UI in organization settings.
- **Downstream dependency**: `rhaiis/pipeline` includes from this repo. Verify it still works after mirroring is set up (Phase 5, item 3).
- **Don't archive GitLab**: The GitLab repo should remain as a read-only mirror, not archived. Archiving breaks the mirror and downstream includes.
- **Confirm with Ken** before pushing to the production Quay org (`aipcc-cicd`).

Would you like to start with the Quay robot account creation (steps 2-3), or do you need to get the `$QUAY_API_TOKEN` first?
