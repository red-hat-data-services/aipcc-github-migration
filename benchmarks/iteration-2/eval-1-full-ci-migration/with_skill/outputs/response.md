I'll get this migration started. Loading the `/gitlab-to-github` skill.

## Step 1: Check for Existing Manifest

Checking `.claude/migrations/aipcc-claudio/manifest.yaml` -- not found. This is a new migration.

## Step 2: Init

From your prompt I have the answers to the two init questions:

1. **Repo:** `aipcc-claudio`
2. **GitHub org:** `red-hat-data-services` (AIPCC default)

I've created the migration manifest at `.claude/migrations/aipcc-claudio/manifest.yaml` with these settings:

- **Source:** `gitlab.com/redhat/rhel-ai/ci-cd/aipcc-claudio`
- **Target:** `github.com/red-hat-data-services/aipcc-claudio`
- **Quay:** `quay.io/aipcc-cicd/aipcc-claudio`

Running Phase 0 now.

---

## Phase 0: Pre-flight Assessment

I ran the automated checks against the GitLab repo and the target GitHub/Quay endpoints. Results:

```
Pre-flight Assessment for aipcc-claudio
────────────────────────────────────────
CI pipeline:      yes
Container push:   yes
Multi-arch:       yes
Tekton/Konflux:   no
LICENSE:          missing
POLICY.md:        no
GitHub repo:      absent
Quay repo:        exists
Self-references:  0 files with gitlab> paths
Branches:         6 total (3 keep, 2 delete, 1 to review)
```

### Wave Classification: Wave 3 (complex multi-arch)

The repo has CI with multi-arch container builds and pushes. All six phases apply and none can be skipped.

### Branch Classification

| Branch | Action | Reason |
|--------|--------|--------|
| `main` | Keep | Default branch |
| `release-1.0` | Keep | Matches `release-*` pattern |
| `release-2.0` | Keep | Matches `release-*` pattern |
| `renovate/dependency-a` | Delete | Matches `renovate/*` (bot-managed) |
| `renovate/dependency-b` | Delete | Matches `renovate/*` (bot-managed) |
| `feature/wip-work` | Ask | Not merged, no matching keep pattern |

I need you to confirm: is `feature/wip-work` safe to delete, or does someone still need it?

> **Note:** Branch cleanup on GitLab is deferred to post-announce. Since GitHub becomes canonical via mirroring, only the branches pushed to GitHub matter. Stale GitLab-only branches are cosmetic noise on a read-only mirror.

### GitHub Repo Request

The GitHub repo does not exist yet. You should submit the request now so approval can overlap with Phase 1 cleanup:

> For **red-hat-data-services**: ask Ken Dreyer to create the repo `red-hat-data-services/aipcc-claudio`.
>
> Submit now -- approval can take hours/days, and Phase 1 cleanup runs in parallel.

### Skip Conditions Evaluated

| Condition | Result | Effect |
|-----------|--------|--------|
| `has_ci == false` | NO (has CI) | CI phase NOT skipped |
| `has_container_push == false` | NO (pushes containers) | Quay OIDC phase NOT skipped |
| `has_license == true` + Apache-2.0 | NO (missing) | LICENSE step NOT skipped |
| `has_policy_md == true` | NO (missing) | POLICY.md step NOT skipped |
| `github_state == "has_content"` | NO (absent) | GitHub push step NOT skipped |
| `gitlab_self_references` empty | YES (0 files) | "Update GitLab self-references" step SKIPPED |

One item skipped: the self-references update in Phase 1 (nothing to update). Everything else is active.

---

## Phase 1: Repo Cleanup (Guidance)

All changes happen on a feature branch in the GitLab repo. Each change gets its own commit with `Signed-off-by:` trailer (`git commit -s`).

```
Migration Status: aipcc-claudio
──────────────────────────────────
Source:  gitlab.com/redhat/rhel-ai/ci-cd/aipcc-claudio
Target:  red-hat-data-services/aipcc-claudio
Ticket:  not set

Phase 1: Cleanup .............. pending  <- you are here
  . Add POLICY.md                        <- next step
  . Add LICENSE (Apache-2.0)
  . Update README
  . Update GitLab self-references .... skipped (no self-references detected)
  . Commit with Signed-off-by
  . MR submitted and merged
Phase 2: GitHub Setup ......... pending
Phase 3: Quay OIDC ............ pending
Phase 4: CI ................... pending
Phase 5: Mirroring ............ pending
Phase 6: Announce ............. pending

Next step: Add POLICY.md
```

### Step-by-step plan

**1. Create a feature branch**

```bash
cd src/aipcc-claudio
git checkout main
git pull origin main
git checkout -b cleanup/migration-prep
```

**2. Add POLICY.md** [automatable]

I'll copy the bundled `templates/POLICY.md` from the skill into the repo root. The file is the AIPCC commit and merge guidelines, already in markdown with PR terminology.

```bash
cp .claude/skills/gitlab-to-github/templates/POLICY.md src/aipcc-claudio/POLICY.md
cd src/aipcc-claudio
git add POLICY.md
git commit -s -m "docs: add POLICY.md with commit and merge request guidelines"
```

**3. Add LICENSE (Apache-2.0)** [automatable]

Write the standard Apache-2.0 license text to a `LICENSE` file in the repo root.

```bash
git add LICENSE
git commit -s -m "docs: add Apache-2.0 license"
```

**4. Update README** [automatable]

I'll scan the README for GitLab-specific content and update it:
- GitLab CI badge URLs -- remove or replace with GitHub Actions badges
- `gitlab.com` clone URLs -- update to `github.com/red-hat-data-services/aipcc-claudio`
- References to "merge requests" -- change to "pull requests"
- GitLab-specific instructions

```bash
git add README.md
git commit -s -m "docs: update README for GitHub migration"
```

**5. Update GitLab self-references** [SKIPPED]

No self-referencing GitLab paths were detected in the repo. Nothing to do here.

**6. Commit with Signed-off-by** [automatable]

Every `git commit` in this phase uses the `-s` flag. Already reflected in all commands above.

**7. Submit MR for review** [human action required]

```bash
glab mr create --title "Migration prep: add license, policy, update README" \
  --description "Part of GitLab -> GitHub migration (AIPCC-16303)" \
  --assignee XiyangDong
```

You'll need to get the MR reviewed and merged before we proceed to Phase 2.

---

## What Comes After Phase 1

Once the cleanup MR is merged, the remaining phases are:

| Phase | What happens | Estimated time |
|-------|-------------|----------------|
| **Phase 2: GitHub Setup** | Create repo in `red-hat-data-services`, add remote, push main | 5 minutes |
| **Phase 3: Quay OIDC** | Set up OIDC federation so GitHub Actions can push to `quay.io/aipcc-cicd/aipcc-claudio` | 30 minutes |
| **Phase 4: CI** | Translate `.gitlab-ci.yml` to GitHub Actions (multi-arch build + push, pinned to SHA) | 1-2 sessions |
| **Phase 5: Mirroring** | Enable GitLab pull mirroring from GitHub, verify downstream `include:` refs still work | 5 minutes |
| **Phase 6: Announce** | Update README with canonical GitHub URL, notify team | 15 minutes |

Phase 4 (CI) will be the most involved since you have multi-arch container builds. I'll translate the GitLab CI pipeline to a GitHub Actions workflow with OIDC-based Quay authentication, SHA-pinned actions, concurrency groups, and timeouts.

---

## Next Steps

Two things to do in parallel right now:

1. **Request the GitHub repo** -- ask Ken Dreyer to create `red-hat-data-services/aipcc-claudio`. Do this first since approval can overlap with cleanup.
2. **Confirm the branch classification** -- especially what to do with `feature/wip-work`. Is it safe to delete?

Once you've kicked off the repo request, say "go" and I'll start executing the Phase 1 cleanup items on the feature branch.

The manifest is saved and tracks progress across sessions. If you stop now and come back later (or a teammate picks this up), I'll resume from exactly where we left off.
