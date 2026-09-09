---
name: gitlab-to-github
description: >
  Guided migration of a GitLab repository to GitHub with stateful progress tracking.
  Use this skill whenever the user mentions migrating a repo from GitLab to GitHub,
  moving CI from GitLab to GitHub Actions, setting up Quay OIDC federation for GitHub,
  or configuring GitLab pull mirroring. Also use when the user references a
  migration-manifest.yaml file or asks about migration status. This skill handles
  the full lifecycle: repo cleanup, GitHub push, Quay OIDC, CI translation,
  and GitLab mirroring.
user-invocable: true
compatibility: "Requires glab CLI, gh CLI, yq, curl, and ldapsearch (with a Kerberos ticket for CODEOWNERS lookups — see the ldap skill). Optional: QUAY_API_TOKEN for Quay robot setup."
allowed-tools:
  - Bash(git *)
  - Bash(glab *)
  - Bash(gh *)
  - Bash(curl *)
  - Bash(yq *)
  - Bash(grep *)
  - Bash(find *)
  - Bash(jq *)
  - Bash(mkdir *)
  - Bash(chmod *)
  - Bash(sed *)
  - Bash(cat *)
  - Bash(ls *)
  - Bash(echo *)
  - Bash(ldapsearch *)
  - Read(*)
  - Write(*)
  - Edit(*)
---

# GitLab to GitHub Migration

A stateful, phased migration skill for moving repositories from GitLab to GitHub.
Progress is tracked in a `migration-manifest.yaml` file that survives across sessions,
enabling cold-resume and teammate handoff.

**This skill is designed to be run multiple times.** Migrations involve waiting
(MR reviews, repo provisioning, approval processes), so the skill saves progress
and picks up where it left off on the next invocation. Tell the user this upfront.

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
  ✓ Add LICENSE (Apache-2.0)
  · Update README                         ← next step
  · Update GitLab self-references
  · Update CODEOWNERS for GitHub usernames
  · Commit with Signed-off-by
  · MR submitted and merged
Phase 2: GitHub Setup ......... <status>
Phase 3: Konflux/KRD .......... <status or "skip (reason)">
Phase 4: Quay OIDC ............ <status or "skip (reason)">
Phase 5: CI ................... <status>
Phase 6: Mirroring ............ <status>

Next step: <first incomplete item in active phase>
```

  Status values: `complete`, `pending`, `waiting`, `skip (reason)`, `blocked`, `optional`
  Show item-level detail (✓/·) only for the active phase — collapsed for others.
  Mark the active phase with `← you are here` and the next item with `← next step`.

- Ask: "Resume from where you left off, or start fresh?"
- If resuming, jump to the current phase's first incomplete item

**If not found:**
- Proceed to Init

### 2. Init (New Migration)

Ask two questions:
1. "What's the GitLab repo URL or path?" (e.g., `gitlab.com/redhat/rhel-ai/ci-cd/central-linter` or just `redhat/rhel-ai/ci-cd/central-linter`)
2. "Which GitHub org?" (`opendatahub-io` for upstream/community, `red-hat-data-services` for downstream/internal)

Then:
- Parse the GitLab input: strip `https://gitlab.com/` prefix if present, extract the repo name (last path segment) and full GitLab path
- Read the manifest template from `templates/migration-manifest.yaml` (relative to this skill)
- Fill in the template with:
  - `repo`: the repo name (last segment of the GitLab path)
  - `source_gitlab`: the full GitLab project path
  - `target_github_org`: user's choice
  - `target_github_repo`: `<org>/<repo>`
  - Leave `quay_org` and `quay_repo` empty — Phase 0 populates them if container push is detected
- Create `.claude/migrations/<repo>/` directory
- Write the populated manifest to `.claude/migrations/<repo>/manifest.yaml`
- Immediately run Phase 0

### 3. Phase 0: Pre-flight Assessment

Automated scan — no user input needed. Populate the `detected.*` fields in the manifest, then
present the assessment summary and proceed to Phase 1.

Read `references/phase-0-preflight.md` for the exact checks to run, what to populate, and the
summary table format.

### 4. Phase Loop

For each phase (cleanup → github_setup → konflux_krd → quay_oidc → ci → mirroring):

1. **Check skip conditions** before loading the phase reference file:
   - `konflux_managed != true` → skip `konflux_krd` phase entirely (requires the explicit user confirmation from Phase 0, not just `detected.has_tekton`)
   - `detected.has_ci == false` → skip `ci` phase
   - `detected.has_container_push == false` → skip `quay_oidc` phase
   - `detected.github_state == "has_content"` → skip push step in `github_setup`
   - `push_tags != true` → skip "Push tags to GitHub" in `github_setup`
   - `detected.has_license == true && detected.license_type == "Apache-2.0"` → skip license item in `cleanup`
   - Otherwise, no existing license is detected — this is **not** an auto-skip. Ask the user
     whether to add the Apache-2.0 license or skip it (e.g. internal-only repo, license handled
     elsewhere, GPL repo pending legal review — see Gotchas in `phase-1-cleanup.md`). Record the
     answer in `license_decision` (`add` or `skip`) so re-runs don't ask again. If `skip`, mark
     the license item `status: skipped` with the user's reason.
   - `detected.gitlab_self_references` is empty → skip "Update GitLab self-references" item in `cleanup`
   - `detected.has_codeowners == false` → skip "Update CODEOWNERS for GitHub usernames" item in `cleanup`
   - `detected.has_container_push == false` → skip "Scope id-token to push job" in `ci` phase
   - Mark skipped phases/items in the manifest with `status: skipped` and a `reason`

2. **Load the phase reference file** — Read the corresponding file from `references/`:
   - `cleanup` → `references/phase-1-cleanup.md`
   - `github_setup` → `references/phase-2-github-setup.md`
   - `konflux_krd` → `references/phase-3-konflux-krd.md`
   - `quay_oidc` → `references/phase-4-quay-oidc.md`
   - `ci` → `references/phase-5-ci.md`
   - `mirroring` → `references/phase-6-mirroring.md`

3. **Walk through items** in the phase one by one:
   - **Automatable items** (`type: automatable`): Present the command/action, ask "Run this?", execute on approval
   - **Human-required items** (`type: human`): Explain what needs to happen, ask user to confirm when done.
     If the user can't complete it now (waiting for MR review, repo provisioning, etc.),
     mark it `status: waiting`, save the manifest, and tell them:
     "Run `/gitlab-to-github` again when that's done — I'll pick up right here."
   - **Blocked items**: If user says something is blocked, mark it `status: blocked` with a note, move on

   A `waiting` item is a persisted checkpoint, not a completed item. Save the
   external reference, the current gate, and the next required confirmation in
   the manifest before stopping. On the next invocation, re-read that state and
   resume from the checkpoint; do not recreate a branch, commit, or MR that is
   already recorded.

   Before starting an independent change or MR in any repository:
   - Confirm the worktree is clean and identify the repository's default branch
     (normally `main`). Do not discard uncommitted work to make it clean.
   - Fetch the remote and fast-forward the local default branch from its remote
     counterpart (`git fetch origin --prune`, then `git merge --ff-only origin/main`;
     use the configured default branch if it is not `main`).
   - Create the new feature branch from that refreshed branch. Record the base
     commit in the manifest when the change belongs to a multi-MR sequence.

   The Konflux/KRD track and the Quay OIDC track are independent. Do not delay
   the KRD Component-removal checkpoint until Quay OIDC is complete. KRD
   Component deletion is gated by the ImageRepository preservation annotation
   and ArgoCD confirmation, not by GitHub Actions' Quay authentication.

4. **After each item**, update the manifest: set item `status` to `complete`, `waiting`, `skipped`, or `blocked`. A waiting item pauses the phase; do not mark the phase complete or continue to a dependent item.

5. **After all items in a phase**, set the phase `status` to `complete` and present the next phase

6. **After all phases**, congratulate the user and show a final summary

## Safety Rules

These are non-negotiable:

- **Never push to any upstream repo without explicit user approval** — always show the exact command and wait
- **Never delete branches without user confirmation** — show the branch list and classification first
- **Verify mirroring works before declaring migration complete** — the downstream CI must pass
- **All git commits must include `Signed-off-by:`** — use `git commit -s`
- **Pin GitHub Actions to SHA digests**, not version tags
- **Refresh the default branch before each independent MR** — never base the next
  change in a sequential migration on a stale pre-merge branch

## AIPCC Defaults

These are baked into the manifest template. Override at init if needed.

| Setting | Default |
|---------|---------|
| GitLab path | asked at init |
| Target GitHub org | `opendatahub-io` or `red-hat-data-services` (asked at init) |
| Quay org | `aipcc-cicd` (only if container push detected) |
| License (optional, if added) | Apache-2.0 |
| App-interface role | `rhoai/dev` |

Maintainers list is in the manifest template.

## Reference Files

Load these on demand — do not read them all at init.

| File | When to read |
|------|-------------|
| `references/phase-0-preflight.md` | Running Phase 0 assessment |
| `references/phase-1-cleanup.md` | Starting cleanup phase |
| `references/phase-2-github-setup.md` | Starting github_setup phase |
| `references/phase-3-konflux-krd.md` | Starting konflux_krd phase (only if `konflux_managed` confirmed) |
| `references/phase-4-quay-oidc.md` | Starting quay_oidc phase |
| `references/phase-5-ci.md` | Starting ci phase |
| `references/phase-6-mirroring.md` | Starting mirroring phase |

## Manifest Location

The manifest lives at `.claude/migrations/<repo>/manifest.yaml`. This path is project-agnostic —
it works whether the user has a Layer 1/2 workspace, a flat checkout, or any other structure.
The `.claude/` directory is Claude Code's config space and is typically gitignored.

Always re-read the manifest at the start of each invocation — another session or teammate may
have updated it.

## Timing Expectations

Typical durations (based on past migrations):

| Phase | Typical Duration |
|-------|-----------------|
| Cleanup + MR review | 1-2 sessions |
| GitHub repo + push | 5 minutes |
| Konflux/KRD reconfiguration | small MR per item + ArgoCD sync wait (minutes to tens of minutes); budget more for Component delete/recreate checkpoints, not just ImageRepositories |
| Quay OIDC federation | 30 minutes |
| GitHub Actions CI | 1-2 sessions |
| GitLab mirroring | small MR + up to ~45 min for first sync |
