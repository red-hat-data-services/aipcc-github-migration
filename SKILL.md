---
name: gitlab-to-github
description: >
  Guided migration of a GitLab repository to GitHub with stateful progress tracking.
  Use this skill whenever the user mentions migrating a repo from GitLab to GitHub,
  moving CI from GitLab to GitHub Actions, setting up Quay OIDC federation for GitHub,
  or configuring GitLab pull mirroring. Also use when the user references a
  migration-manifest.yaml file or asks about migration status. This skill handles
  the full lifecycle: repo cleanup, GitHub push, Quay OIDC, CI translation,
  GitLab mirroring, and team announcement.
user-invocable: true
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Agent
  - WebFetch
---

# GitLab to GitHub Migration

A stateful, phased migration skill for moving repositories from GitLab to GitHub.
Progress is tracked in a `migration-manifest.yaml` file that survives across sessions,
enabling cold-resume and teammate handoff.

## Invocation Flow

On every invocation, follow this sequence exactly:

### 1. Check for Existing Manifest

Look for the manifest at `.claude/migrations/<repo>/manifest.yaml` (check all subdirectories
under `.claude/migrations/`). If the user specifies a repo name, check that specific path.

**If found:**
- Read the manifest
- Show a status dashboard using this exact format:

```
Migration Status: <repo>
──────────────────────────────────
Source:  gitlab.com/<source_gitlab>
Target:  <target_github_repo>
Ticket:  <jira_ticket or "not set">

Phase 1: Cleanup .............. <status>  ← you are here (on active phase)
  ✓ Add POLICY.md
  ✓ Add LICENSE (Apache-2.0)
  · Update README                         ← next step
  · Update GitLab self-references
  · Commit with Signed-off-by
  · MR submitted and merged
Phase 2: GitHub Setup ......... <status>
Phase 3: Quay OIDC ............ <status or "skip (reason)">
Phase 4: CI ................... <status>
Phase 5: Mirroring ............ <status>
Phase 6: Announce ............. <status>

Next step: <first incomplete item in active phase>
```

  Status values: `complete`, `pending`, `skip (reason)`, `blocked`, `optional`
  Show item-level detail (✓/·) only for the active phase — collapsed for others.
  Mark the active phase with `← you are here` and the next item with `← next step`.

- Ask: "Resume from where you left off, or start fresh?"
- If resuming, jump to the current phase's first incomplete item

**If not found:**
- Proceed to Init

### 2. Init (New Migration)

Ask two questions:
1. "Which repo are you migrating?" (e.g., `central-linter`, `aipcc-claudio`)
2. "Which GitHub org?" (default: `red-hat-data-services`, alternative: `opendatahub-io`)

Then:
- Read the manifest template from `templates/migration-manifest.yaml` (relative to this skill)
- Fill in the template with:
  - `repo`: the repo name
  - `source_gitlab`: `redhat/rhel-ai/ci-cd/<repo>` (AIPCC default prefix)
  - `target_github_org`: user's choice
  - `target_github_repo`: `<org>/<repo>`
  - `quay_org`: `aipcc-cicd`
  - `quay_repo`: `aipcc-cicd/<repo>`
- Create `.claude/migrations/<repo>/` directory
- Write the populated manifest to `.claude/migrations/<repo>/manifest.yaml`
- Immediately run Phase 0

### 3. Phase 0: Pre-flight Assessment

Automated scan — no user input needed. Populate the `detected.*` fields in the manifest.

**Checks to run:**

```bash
# GitLab repo accessible?
glab api projects/$(echo "$SOURCE_GITLAB" | sed 's|/|%2F|g') --method GET

# .gitlab-ci.yml exists?
glab api "projects/$PROJECT_ID/repository/files/.gitlab-ci.yml?ref=main" --method GET

# LICENSE file?
glab api "projects/$PROJECT_ID/repository/files/LICENSE?ref=main" --method GET

# POLICY.md?
glab api "projects/$PROJECT_ID/repository/files/POLICY.md?ref=main" --method GET

# .tekton/ directory?
glab api "projects/$PROJECT_ID/repository/tree?path=.tekton&ref=main" --method GET

# Branch list
glab api "projects/$PROJECT_ID/repository/branches?per_page=100" --method GET

# GitHub repo state
gh repo view "$TARGET_GITHUB_REPO" --json isEmpty 2>/dev/null

# Quay repo exists?
curl -sf "https://quay.io/api/v1/repository/$QUAY_REPO" >/dev/null 2>&1
```

**If `.gitlab-ci.yml` exists**, parse it for:
- `buildah push` / `podman push` / `skopeo copy` → `detected.has_container_push: true`
- `--platform` / `--manifest` / architecture matrix → `detected.has_multi_arch: true`
- `include: project:` → populate `detected.includes_from[]`

**Scan for self-referencing GitLab paths** in all non-binary files:
```bash
# Look for gitlab> preset references, GitLab API URLs, or project paths pointing to this repo
grep -rl "gitlab>.*$REPO_NAME\|gitlab\.com/.*$SOURCE_GITLAB" . \
  --include="*.json" --include="*.yaml" --include="*.yml" --include="*.md" \
  --exclude-dir=.git
```
Populate `detected.gitlab_self_references[]` with the matched file paths.

**Branch classification** (informational — branch cleanup happens post-announce):
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
POLICY.md:       yes/no
GitHub repo:     absent / empty / has content
Quay repo:       exists / missing
Self-references:  N files with gitlab> paths (list them)
Branches:        N total (K keep, D delete, A to review)
```

**If `github_state == "absent"`**, prompt the user to request the repo now so approval
overlaps with Phase 1 cleanup:

```
⚠ GitHub repo does not exist yet.

For opendatahub-io: submit the repo request Google Form
  (ask the team chat if you don't have the URL yet).
For red-hat-data-services: ask Ken Dreyer to create it.

Submit now — approval can take hours/days, and Phase 1 cleanup
runs in parallel.
```

Then proceed to Phase 1.

### 4. Phase Loop

For each phase (cleanup → github_setup → quay_oidc → ci → mirroring → announce):

1. **Check skip conditions** before loading the phase reference file:
   - `detected.has_ci == false` → skip `ci` phase
   - `detected.has_container_push == false` → skip `quay_oidc` phase
   - `detected.github_state == "has_content"` → skip push step in `github_setup`
   - `detected.has_license == true && detected.license_type == "Apache-2.0"` → skip license item in `cleanup`
   - `detected.has_policy_md == true` → skip POLICY.md item in `cleanup`
   - `detected.gitlab_self_references` is empty → skip "Update GitLab self-references" item in `cleanup`
   - Mark skipped phases/items in the manifest with `status: skipped` and a `reason`

2. **Load the phase reference file** — Read the corresponding file from `references/`:
   - `cleanup` → `references/phase-1-cleanup.md`
   - `github_setup` → `references/phase-2-github-setup.md`
   - `quay_oidc` → `references/phase-3-quay-oidc.md`
   - `ci` → `references/phase-4-ci.md`
   - `mirroring` → `references/phase-5-mirroring.md`
   - `announce` → `references/phase-6-announce.md`

3. **Walk through items** in the phase one by one:
   - **Automatable items** (`type: automatable`): Present the command/action, ask "Run this?", execute on approval
   - **Human-required items** (`type: human`): Explain what needs to happen, ask user to confirm when done
   - **Blocked items**: If user says something is blocked, mark it `status: blocked` with a note, move on

4. **After each item**, update the manifest: set item `status` to `complete`, `skipped`, or `blocked`

5. **After all items in a phase**, set the phase `status` to `complete` and announce the next phase

6. **After all phases**, congratulate the user and show a final summary

## Safety Rules

These are non-negotiable:

- **Never push to any upstream repo without explicit user approval** — always show the exact command and wait
- **Never delete branches without user confirmation** — show the branch list and classification first
- **Verify mirroring works before declaring migration complete** — the downstream CI must pass
- **All git commits must include `Signed-off-by:`** — use `git commit -s`
- **Pin GitHub Actions to SHA digests**, not version tags

## AIPCC Defaults

These are baked into the manifest template. Override at init if needed.

| Setting | Default |
|---------|---------|
| GitLab path prefix | `redhat/rhel-ai/ci-cd/` |
| Target GitHub org | `red-hat-data-services` |
| Quay org | `aipcc-cicd` |
| License | Apache-2.0 |
| App-interface role | `rhoai/dev` |

Maintainers list is in the manifest template.

## Reference Files

Load these on demand — do not read them all at init.

| File | When to read |
|------|-------------|
| `references/phase-1-cleanup.md` | Starting cleanup phase |
| `references/phase-2-github-setup.md` | Starting github_setup phase |
| `references/phase-3-quay-oidc.md` | Starting quay_oidc phase |
| `references/phase-4-ci.md` | Starting ci phase |
| `references/phase-5-mirroring.md` | Starting mirroring phase |
| `references/phase-6-announce.md` | Starting announce phase |

## Manifest Location

The manifest lives at `.claude/migrations/<repo>/manifest.yaml`. This path is project-agnostic —
it works whether the user has a Layer 1/2 workspace, a flat checkout, or any other structure.
The `.claude/` directory is Claude Code's config space and is typically gitignored.

Always re-read the manifest at the start of each invocation — another session or teammate may
have updated it.

## Timing Expectations

From the central-linter migration:

| Phase | Typical Duration |
|-------|-----------------|
| Cleanup + MR review | 1-2 sessions |
| GitHub repo + push | 5 minutes |
| Quay OIDC federation | 30 minutes |
| GitHub Actions CI | 1-2 sessions |
| GitLab mirroring | 5 minutes |
| Announcement | 15 minutes |
