# Migration Status: dashboard

**Repo:** `dashboard`
**Source:** `gitlab.com/redhat/rhel-ai/ci-cd/dashboard`
**Target:** `opendatahub-io/dashboard`
**Jira:** AIPCC-18270 | **Wave:** 3

---

## Phase Summary

| Phase | Status | Items |
|-------|--------|-------|
| 1. Cleanup | COMPLETE | 6/6 done |
| 2. GitHub Setup | COMPLETE | 1 done, 2 skipped (repo already has content) |
| 3. Quay OIDC | **PENDING** | 0/7 done |
| 4. CI | PENDING | 0/8 done |
| 5. Mirroring | PENDING | 0/3 done |
| 6. Announce | PENDING | 0/3 done |

**Overall progress: 2 of 6 phases complete.**

---

## Skip Conditions Applied

- `github_setup` > "Repo exists in target org": **skipped** (github_state is has_content)
- `github_setup` > "Push main to GitHub": **skipped** (github_state is has_content)

### Additional skip to apply now

- `quay_oidc` > "Create Quay repo": **should be skipped** because `detected.quay_repo_exists == true`. The Quay repo `aipcc-cicd/dashboard` already exists.

---

## Detected Configuration

| Property | Value |
|----------|-------|
| CI pipeline | yes |
| Container push | yes |
| Multi-arch | no |
| Tekton/Konflux | yes |
| spec inputs | no |
| LICENSE | missing (added during cleanup) |
| POLICY.md | missing (added during cleanup) |
| GitHub repo | has content |
| Quay repo | exists |
| Branches | 5 total (2 keep, 0 delete, 1 to review) |
| Included by | `redhat/rhel-ai/ci-cd/rhaiis/pipeline` |

**Note:** `branches_to_ask` still lists `feature/new-dashboard-ui`. If this was not resolved during cleanup, confirm with the repo owner whether to keep or delete it.

---

## Next Phase: Quay OIDC Federation (Phase 3)

This phase sets up passwordless container image push from GitHub Actions to `quay.io/aipcc-cicd/dashboard` using OIDC federation. No long-lived secrets to manage or rotate.

### Items to complete (7 total, 1 skip)

#### 1. Create Quay repo -- SKIP
The Quay repo `aipcc-cicd/dashboard` already exists (`detected.quay_repo_exists: true`). Mark this item `status: skipped` with `reason: "quay_repo_exists is true"`.

#### 2. Create org-level robot account [automatable]
Create a robot account named `aipcc-cicd+dashboard_github` at the org level:

```bash
curl -sf -X PUT \
  -H "Authorization: Bearer $QUAY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"description": "GitHub Actions OIDC push for dashboard"}' \
  "https://quay.io/api/v1/organization/aipcc-cicd/robots/dashboard_github"
```

**Prerequisite:** You need a valid `$QUAY_API_TOKEN`. Generate one from Quay > Account Settings > CLI Password > Generate Encrypted Password.

**Important:** Use underscores in the robot name (`dashboard_github`), not hyphens. Quay robot names do not support hyphens.

If the robot already exists (from a prior attempt or another migration), verify it has the correct permissions rather than creating a duplicate.

#### 3. Grant robot write permission [automatable]
After the robot is created, grant it Write access to the `aipcc-cicd/dashboard` repo:

```bash
curl -sf -X PUT \
  -H "Authorization: Bearer $QUAY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role": "write"}' \
  "https://quay.io/api/v1/repository/aipcc-cicd/dashboard/permissions/user/aipcc-cicd+dashboard_github"
```

#### 4. Get numeric subject claim [human]
Navigate to the GitHub repo settings page for `opendatahub-io/dashboard` and get the immutable subject claim string. The format uses numeric IDs, not human-readable names:

```
repo:opendatahub-io@<ORG_NUMERIC_ID>/dashboard@<REPO_NUMERIC_ID>:ref:refs/heads/main
```

The numeric org and repo IDs are visible on the GitHub repo settings page. The human-readable format (`repo:opendatahub-io/dashboard:ref:refs/heads/main`) will silently fail -- you must use the numeric version.

#### 5. Configure robot federation [human -- UI only]
No API is available for this step. Go to the Quay UI:

1. Navigate to `quay.io/organization/aipcc-cicd` > **Robot Accounts**
2. Click the `dashboard_github` robot > **Federation** tab
3. If the Federation tab is not visible, enable the **v2 UI** in Quay organization settings
4. Add a federation entry:
   - **Issuer:** `https://token.actions.githubusercontent.com`
   - **Subject:** the claim string from step 4

For multiple branches or triggers, add separate federation entries for each ref pattern.

**Coordinate with Ken** (org owner for aipcc-cicd) before pushing to production Quay.

#### 6. Add quay-oidc-login.sh [automatable]
Write the OIDC login script to `.github/scripts/quay-oidc-login.sh` in the dashboard repo. This script:
- Fetches the GitHub Actions OIDC token
- Exchanges it for a Quay federation token
- Logs into Quay via `buildah login`

The script requires `id-token: write` permission in the GitHub Actions workflow and the `QUAY_ROBOT_USER` environment variable set to `aipcc-cicd+dashboard_github`.

After writing:
```bash
chmod +x .github/scripts/quay-oidc-login.sh
git add .github/scripts/quay-oidc-login.sh
git commit -s -m "ci: add Quay OIDC login script for GitHub Actions"
```

#### 7. Verify push succeeds [automatable -- depends on Phase 4]
This step requires CI to be configured first (Phase 4). After CI is set up, trigger a workflow run and confirm the image lands in Quay:

```bash
gh workflow run ci.yml --repo opendatahub-io/dashboard --ref main
gh run list --repo opendatahub-io/dashboard --limit 1
# Then watch the run:
gh run watch <RUN_ID> --repo opendatahub-io/dashboard
```

Check Quay for the new image:
```bash
curl -sf "https://quay.io/api/v1/repository/aipcc-cicd/dashboard/tag/" | jq '.tags[0]'
```

---

## Gotchas for This Phase

1. **Org-level robots only** -- Federation is not available on personal robot accounts. Must be an org robot.
2. **Immutable subject claims** -- The `repo:org@ID/repo@ID:ref:...` format is the only one that works.
3. **v2 UI required** -- The Federation tab may not appear in the Quay classic UI.
4. **Opaque OIDC errors** -- If token exchange fails, decode the JWT to compare `.sub`:
   ```bash
   echo "${OIDC_TOKEN}" | cut -d. -f2 | base64 -d 2>/dev/null | jq .sub
   ```
5. **Coordinate with Ken** before pushing images to production Quay repos.

---

## Remaining Work After Quay OIDC

| Phase | Est. Duration | Key Items |
|-------|--------------|-----------|
| 4. CI | 1-2 sessions | Translate GitLab CI to GitHub Actions, pin actions to SHA, add dependabot |
| 5. Mirroring | 5 minutes | Enable GitLab pull mirroring, verify downstream `include:` refs work (important: `rhaiis/pipeline` includes from this repo) |
| 6. Announce | 15 minutes | Update README, notify team, update external references |

---

## Ready to proceed?

The first actionable item is **creating the robot account** (item 2). Do you have a valid `$QUAY_API_TOKEN`? If so, I can run the create-robot API call. If not, generate one first from Quay > Account Settings.
