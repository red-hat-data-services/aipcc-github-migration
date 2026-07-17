# Phase 3: Quay OIDC Federation

Passwordless container image push from GitHub Actions to Quay.io. No secrets to rotate.

Skip this entire phase if `detected.has_container_push == false`.

## Items

- **[human] Create Quay repo** — User creates the repo at `quay.io/<quay_org>/<repo>` if it doesn't exist. Skip if `detected.quay_repo_exists == true`.
- **[automatable] Create org-level robot account** — API call to create `<quay_org>+<repo>_github` robot.
- **[automatable] Grant robot write permission** — API call to grant the robot Write on the target repo.
- **[human] Get numeric subject claim** — User navigates to GitHub repo Settings to get the immutable subject claim string with numeric IDs.
- **[human] Configure robot federation** — User configures the federation entry in Quay UI (no API available for this step).
- **[automatable] Add quay-oidc-login.sh** — Write the OIDC login script to `.github/scripts/`.
- **[automatable] Verify push succeeds** — Trigger a workflow run and confirm the image lands in Quay.

## Automation Details

### Creating the robot account

```bash
curl -sf -X PUT \
  -H "Authorization: Bearer $QUAY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"description": "GitHub Actions OIDC push for <repo>"}' \
  "https://quay.io/api/v1/organization/<quay_org>/robots/<repo>_github"
```

The robot name convention is `<repo>_github` (underscores, not hyphens — Quay robot names can't contain hyphens).

### Granting write permission

```bash
curl -sf -X PUT \
  -H "Authorization: Bearer $QUAY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role": "write"}' \
  "https://quay.io/api/v1/repository/<quay_org>/<repo>/permissions/user/<quay_org>+<repo>_github"
```

### Subject claim format

The user needs to get this from the GitHub repo settings page. The format is **immutable** — GitHub will not accept the human-readable variant:

```
repo:<org>@<org_numeric_id>/<repo>@<repo_numeric_id>:ref:refs/heads/main
```

Example:
```
repo:opendatahub-io@57720972/central-linter@1292747540:ref:refs/heads/main
```

The numeric org and repo IDs are visible on the GitHub repo settings page.

### Federation configuration (UI only)

Tell the user to:
1. Go to `quay.io/organization/<quay_org>` → **Robot Accounts**
2. Click the robot account → **Federation** tab
3. If no Federation tab visible, enable the **v2 UI** in Quay organization settings
4. Add a federation entry:
   - **Issuer:** `https://token.actions.githubusercontent.com`
   - **Subject:** the claim string from the previous step

For multiple branches/triggers, add separate federation entries for each ref pattern.

### Adding quay-oidc-login.sh

Write this to `.github/scripts/quay-oidc-login.sh`:

```bash
#!/usr/bin/env bash
# Quay OIDC federation login for GitHub Actions
# Reference: https://github.com/ktdreyer/quay-oidc-demo/blob/main/quay-oidc.md
set -euo pipefail

: "${QUAY_ROBOT_USER:?QUAY_ROBOT_USER is required}"
: "${ACTIONS_ID_TOKEN_REQUEST_TOKEN:?Must run inside GitHub Actions with id-token: write}"
: "${ACTIONS_ID_TOKEN_REQUEST_URL:?Must run inside GitHub Actions with id-token: write}"

OIDC_TOKEN=$(curl -sSf \
  -H "Authorization: bearer ${ACTIONS_ID_TOKEN_REQUEST_TOKEN}" \
  "${ACTIONS_ID_TOKEN_REQUEST_URL}" | jq -r .value)
echo "::add-mask::${OIDC_TOKEN}"

QUAY_TOKEN=$(curl -sSf \
  "https://quay.io/oauth2/federation/robot/token" \
  -u "${QUAY_ROBOT_USER}:${OIDC_TOKEN}" | jq -r .token)
echo "::add-mask::${QUAY_TOKEN}"

buildah login -u "${QUAY_ROBOT_USER}" --password-stdin quay.io <<< "${QUAY_TOKEN}"
```

Then:
```bash
chmod +x .github/scripts/quay-oidc-login.sh
git add .github/scripts/quay-oidc-login.sh
git commit -s -m "ci: add Quay OIDC login script for GitHub Actions"
```

### Verifying the push

After CI is configured (Phase 4), trigger a run:
```bash
gh workflow run ci.yml --repo <target_org>/<repo> --ref main
gh run list --repo <target_org>/<repo> --limit 1
gh run watch <RUN_ID> --repo <target_org>/<repo>
```

Check Quay for the new image:
```bash
curl -sf "https://quay.io/api/v1/repository/<quay_org>/<repo>/tag/" | jq '.tags[0]'
```

## Gotchas

- **Org-level robots only**: Federation is not available on personal robot accounts. Must be an org robot.
- **Immutable subject claims**: The `repo:org@ID/repo@ID:ref:...` format is the only one that works. Human-readable `repo:org/repo:ref:...` will silently fail.
- **v2 UI required**: The Federation tab may not appear in the Quay classic UI. Toggle v2 UI in organization settings.
- **OIDC errors are opaque**: If the token exchange fails, add JWT decode diagnostics temporarily:
  ```bash
  echo "${OIDC_TOKEN}" | cut -d. -f2 | base64 -d 2>/dev/null | jq .sub
  ```
  Compare the decoded `.sub` with what's configured in Quay.
- **Robot naming**: Use underscores (`central_linter_github`), not hyphens. Quay robot names don't support hyphens.
- **Don't push to production Quay until federation is confirmed** with the org owner (Ken for aipcc-cicd).

## Troubleshooting

### "Token does not match robot"
The subject claim in Quay doesn't match what GitHub sends. Decode the JWT (see above) and compare `.sub` field character-by-character with the Quay federation config.

### HTTP 400 from `quay.io/oauth2/federation/robot/token`
Common causes:
- Robot account is personal, not org-level
- Federation entry not saved (v2 UI may be required)
- Issuer URL wrong (must be exactly `https://token.actions.githubusercontent.com`)

### HTTP 401 from Quay API when creating robot
The `$QUAY_API_TOKEN` is invalid or expired. Generate a new one from Quay → Account Settings → CLI Password → Generate Encrypted Password → select "Kubernetes Secret" or "Docker Configuration" for the token.

### Robot account already exists
If another migration already created the robot, just verify it has Write permission on the correct repo. Don't create a duplicate.
