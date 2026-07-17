# Migration Simulation: product-management-configs

## Invocation Flow (Simulated)

### Step 1: Check for Existing Manifest

No existing `migration-manifest.yaml` found. Proceeding to Init.

### Step 2: Init (New Migration)

**User inputs (simulated):**
1. Repo: `product-management-configs`
2. GitHub org: `red-hat-data-services` (default)

**Template populated with:**
- `repo`: product-management-configs
- `source_gitlab`: redhat/rhel-ai/ci-cd/product-management-configs
- `target_github_org`: red-hat-data-services
- `target_github_repo`: red-hat-data-services/product-management-configs
- `quay_org`: aipcc-cicd
- `quay_repo`: aipcc-cicd/product-management-configs
- `wave`: 1 (simple/no CI)

Manifest written. Proceeding to Phase 0.

---

### Step 3: Phase 0 — Pre-flight Assessment

**Commands that WOULD be run:**

```bash
# GitLab repo accessible?
glab api projects/redhat%2Frhel-ai%2Fci-cd%2Fproduct-management-configs --method GET

# .gitlab-ci.yml exists?
glab api "projects/$PROJECT_ID/repository/files/.gitlab-ci.yml?ref=main" --method GET
# SIMULATED RESULT: 404 — no CI file

# LICENSE file?
glab api "projects/$PROJECT_ID/repository/files/LICENSE?ref=main" --method GET
# SIMULATED RESULT: 200 — GPL-3.0 license detected

# POLICY.md?
glab api "projects/$PROJECT_ID/repository/files/POLICY.md?ref=main" --method GET
# SIMULATED RESULT: 404 — no POLICY.md

# .tekton/ directory?
glab api "projects/$PROJECT_ID/repository/tree?path=.tekton&ref=main" --method GET
# SIMULATED RESULT: 404 — no Tekton config

# Branch list
glab api "projects/$PROJECT_ID/repository/branches?per_page=100" --method GET
# SIMULATED RESULT: 3 branches — main, feature/config-update (merged)

# GitHub repo state
gh repo view red-hat-data-services/product-management-configs --json isEmpty 2>/dev/null
# SIMULATED RESULT: error — repo does not exist (github_state: absent)

# Quay repo exists?
curl -sf "https://quay.io/api/v1/repository/aipcc-cicd/product-management-configs" >/dev/null 2>&1
# SIMULATED RESULT: 404 — no Quay repo
```

**No `.gitlab-ci.yml` found** — skipping CI content parsing (no buildah/podman/skopeo, no multi-arch, no includes).

**Branch classification:**
- **Keep:** `main` (default branch)
- **Delete:** `feature/config-update` (fully merged into main)
- **Ask:** (none)

---

### Phase 0 Results

```
Pre-flight Assessment for product-management-configs
─────────────────────────────────────────────────────
CI pipeline:     no
Container push:  no
Multi-arch:      no
Tekton/Konflux:  no
LICENSE:         GPL-3.0  *** REVIEW REQUIRED ***
POLICY.md:       no
GitHub repo:     absent
Quay repo:       missing
Branches:        3 total (1 keep, 1 delete, 0 to review)
```

---

### Phase Skip Decisions

Based on the Phase 0 assessment, the skill applies skip conditions from the Phase Loop logic:

| Phase | Status | Reason |
|-------|--------|--------|
| cleanup | **pending** | POLICY.md missing (needs creation). LICENSE is GPL-3.0 — flagged for human review, NOT auto-replaced. |
| github_setup | **pending** | GitHub repo is absent — needs creation and push. |
| quay_oidc | **SKIPPED** | `detected.has_container_push == false` — no container builds, so Quay OIDC federation is unnecessary. |
| ci | **SKIPPED** | `detected.has_ci == false` — no CI pipeline exists, so GitHub Actions translation is unnecessary. |
| mirroring | **pending** | Still needed — downstream GitLab references may depend on this repo path. |
| announce | **pending** | Still needed — team must know the canonical location changed. |

### License Handling: GPL-3.0 Flag

The skill template defaults to "Add LICENSE (Apache-2.0)" as an automatable item. However, the pre-flight scan detected an **existing GPL-3.0 license**. The skill's skip logic says:

> `detected.has_license == true && detected.license_type == "Apache-2.0"` -> skip license item in `cleanup`

Since the license is GPL-3.0 (not Apache-2.0), the item is NOT skipped. But it is also NOT automatically replaced. Instead, the manifest item is modified to:

- **Name** changed to: "Replace LICENSE with Apache-2.0"
- **Type** changed from `automatable` to `human`
- **Note** added: GPL-3.0 is a copyleft license requiring human review before relicensing

This is the correct behavior: automatic replacement of a GPL-3.0 license with Apache-2.0 could violate contributor agreements or third-party code licensing. A human must evaluate whether relicensing is permissible.

---

### Remaining Phases (What Would Happen Next)

If this were a real migration, the skill would proceed through these phases:

**Phase 1: Cleanup** (4 active items + 1 flagged for human review)
1. Add POLICY.md (download Google Doc as markdown)
2. Replace LICENSE with Apache-2.0 — **HUMAN REVIEW REQUIRED** (GPL-3.0 detected)
3. Delete stale branch: `feature/config-update`
4. Update README
5. Commit with Signed-off-by
6. Submit MR for Xiang's review

**Phase 2: GitHub Setup** (3 items)
1. Request repo creation in red-hat-data-services org
2. Add github remote
3. Push main to GitHub

**Phase 3: Quay OIDC** — SKIPPED (no container push)

**Phase 4: CI** — SKIPPED (no CI pipeline)

**Phase 5: Mirroring** (3 items)
1. Enable GitLab pull mirroring from GitHub
2. Verify mirror syncs
3. Confirm downstream includes work

**Phase 6: Announce** (3 items)
1. Update README with GitHub URL
2. Notify team
3. Update external references

---

### Summary

This is a **Wave 1 (simple) migration** — config files only, no CI, no containers. The skill correctly:

1. **Detected** the absence of CI and container push capabilities
2. **Skipped** the quay_oidc and ci phases entirely (with reasons documented in manifest)
3. **Flagged** the GPL-3.0 license for human review instead of auto-replacing
4. **Classified** branches correctly (keep main, delete merged feature branch)
5. **Preserved** mirroring and announce phases (still needed regardless of CI status)

Estimated total time: 1-2 sessions (cleanup + MR review dominate; no CI/Quay work needed).
