# Phase 1: Repo Cleanup (on GitLab)

All changes happen on a feature branch in the GitLab repo. One commit per change.

## Items

- **[optional] Add LICENSE (Apache-2.0)** — Never block a migration on a license. Skip automatically if Phase 0 detected an existing Apache-2.0 license. Otherwise, ask whether the user wants one — see "Deciding on the license" below. If skipped, record the reason and continue.
- **[automatable] Update README** — Fix any GitLab-specific URLs or workflow references that won't apply post-migration.
- **[automatable] Update GitLab self-references** — Replace `gitlab>REPO_PATH` preset references, `local>` paths, and similar with their GitHub equivalents. Skip if `detected.gitlab_self_references` is empty. Show diff and ask before changing cross-platform references.
- **[automatable] Update CODEOWNERS for GitHub usernames** — Resolve each GitLab username/handle in CODEOWNERS to its GitHub username via LDAP and rewrite the file. Skip if `detected.has_codeowners` is false.

> **Note:** Branch cleanup on GitLab is deferred to post-migration. Since GitHub becomes canonical via mirroring, only the branches pushed to GitHub matter. Stale GitLab-only branches are cosmetic noise on a read-only mirror.
- **[automatable] Commit with Signed-off-by** — All commits in this phase must use `git commit -s` to add the `Signed-off-by:` trailer.
- **[human] MR submitted and merged** — User opens the MR on GitLab, gets review, and merges. Wait for this before proceeding.

## Automation Details

### Creating a feature branch

```bash
cd src/<repo>
git checkout main
git pull origin main
git checkout -b cleanup/migration-prep
```

### Deciding on the license

If `detected.has_license` is false (or the existing license isn't Apache-2.0), the migration can
continue without adding a license. Ask whether the user wants to add Apache-2.0:

```
No Apache-2.0 LICENSE detected. Add one?
  - Add Apache-2.0 (standard for AIPCC upstream/community repos)
  - Skip (internal-only repo, license handled elsewhere, or pending legal review)
```

Record the answer in the manifest as `license_decision: add` or `license_decision: skip` so
re-runs of the skill don't ask again. If `skip`, mark the item `status: skipped` with the user's
stated reason.

### Adding LICENSE

Only run this if `license_decision == "add"`.

```bash
# Write the Apache-2.0 license text to LICENSE
git add LICENSE
git commit -s -m "docs: add Apache-2.0 license"
```

### Updating README

Look for:
- GitLab CI badge URLs → remove or replace with GitHub Actions badges
- `gitlab.com` clone URLs → update to `github.com`
- References to MRs → change to PRs
- GitLab-specific instructions

```bash
git add README.md
git commit -s -m "docs: update README for GitHub migration"
```

### Updating GitLab self-references

Skip if `detected.gitlab_self_references` is empty. Otherwise, for each file in the list, replace
GitLab-style references with their GitHub equivalents. Common patterns:

| GitLab pattern | GitHub equivalent |
|---|---|
| `gitlab>GITLAB_PATH//file.json` | `github>GITHUB_ORG/REPO//file.json` |
| `local>GITLAB_PATH` | `local>GITHUB_ORG/REPO` |
| `gitlab.com/GITLAB_PATH` | `github.com/GITHUB_ORG/REPO` |
| `project: 'GITLAB_PATH'` (in CI includes) | n/a — handled in Phase 5 CI translation |

Show the user a diff of each file before committing. Some references may be intentionally
cross-platform (e.g., a GitLab-hosted preset consumed by GitLab-side repos) — ask before changing
those.

#### Flags to raise before rewriting

**`local>` vs `gitlab>` are different protocols.** `gitlab>` tells the tool to fetch from GitLab
specifically. `local>` means "same platform I'm running on." Before rewriting `local>` paths,
ask the user which platform the consuming tool (e.g., Renovate) targets post-migration. If the
tool still reads from GitLab (via mirror), `local>` paths should keep the GitLab org/path.
If the tool moves to GitHub, update to the GitHub org/repo.

**Downstream blast radius.** If this repo is consumed as a shared preset or library by other
repos, changing `gitlab>` to `github>` here will break every consumer still pointing to the old
`gitlab>` path. Flag this to the user:

```
⚠ This repo appears to be a shared preset/config consumed by other repositories.
  Changing gitlab> references to github> will require downstream repos to update
  their references too. Coordinate with consuming teams before rewriting.
```

Detect shared-preset repos by checking for Renovate `extends` patterns, npm/PyPI package
references, or CI `include: project:` pointing to this repo from other projects.

```bash
git add <changed files>
git commit -s -m "chore: update self-references from GitLab to GitHub"
```

### Updating CODEOWNERS for GitHub usernames

Skip if `detected.has_codeowners` is false. GitLab CODEOWNERS entries use GitLab handles
(`@gitlab-handle`); GitHub requires GitHub handles for the same syntax to resolve to review
requests. Read the file at `detected.codeowners_path` and resolve every `@handle` via LDAP.

Use the **ldap skill** for these lookups (Red Hat IPA LDAP over `ldapsearch`, requires a valid
Kerberos ticket — `kinit` first if a query returns nothing):

```bash
ldapsearch -ZZ -H "ldap:///dc=ipa%2Cdc=redhat%2Cdc=com" -QLLL \
  -b 'cn=users,cn=accounts,dc=ipa,dc=redhat,dc=com' \
  '(rhatSocialURL=Gitlab->https://gitlab.com/GITLAB_HANDLE)' rhatSocialURL uid
```

`rhatSocialURL` returns both the `Gitlab->` and `Github->` entries for a user when both are set
on their Red Hat account — pull the GitHub handle out of the `Github->https://github.com/USER`
value.

For each owner token in CODEOWNERS:
- Resolve person owners through LDAP and replace each GitLab handle with its confirmed GitHub username.
- Resolve group owners separately. GitHub team owners use `@<target_github_org>/<team-slug>`; add
  one only after confirming the team exists in the target organization and has repository write access.
- If any owner cannot be resolved, keep the migration item blocked and ask the user for a valid
  GitHub user or team. Do not leave an invalid active owner, guess a team slug, or silently drop a reviewer.

Validate the resulting file before showing the diff:
- Put it at `.github/CODEOWNERS`, `CODEOWNERS`, or `docs/CODEOWNERS`. If the source is
  `.gitlab/CODEOWNERS`, move it to a supported location. If multiple CODEOWNERS files exist,
  consolidate the intended rules so GitHub's precedence does not hide entries.
- Each active line must contain a GitHub-supported path pattern followed by one or more owners in
  `@username` or `@org/team` form. Preserve comments and intended patterns; check path case against
  the repository because GitHub treats paths as case-sensitive. Keep multiple owners for one
  pattern on the same line; later matching patterns take precedence.
- GitHub CODEOWNERS does not support `!` negation or `[ ]` character ranges. Rewrite unsupported
  patterns with equivalent supported patterns before committing.
- Confirm every person/team owner has write access to the target repository. Once the file is on
  the GitHub base branch, check `gh api "repos/$TARGET_GITHUB_ORG/$REPO_NAME/codeowners/errors"`;
  resolve any reported syntax or owner errors before marking the item complete.

Show the full diff before committing.

Set `CODEOWNERS_SOURCE` to the detected source path and `CODEOWNERS_DESTINATION` to the chosen
GitHub-supported location. Use the same path for both when no move is needed.

```bash
git add -A -- "$CODEOWNERS_SOURCE" "$CODEOWNERS_DESTINATION"
git commit -s -m "chore: update CODEOWNERS with GitHub usernames"
```

Show the full diff, including any file move, before committing.

### Opening the MR

```bash
glab mr create --title "Migration prep: update CODEOWNERS, policy, and README" \
  --description "Part of GitLab → GitHub migration"
```

## Gotchas

- **Separate commits**: Each change gets its own commit. Don't squash cleanup into one commit — reviewers expect granular changes.
- **Branch naming**: Use `cleanup/migration-prep` or similar — makes the MR purpose obvious.
- **GPL-licensed repos**: Two repos (product-management-tool, product-management-configs) are GPL-3.0. Never replace an existing license. If a user asks to add Apache-2.0 to a GPL repo, flag the possible legal review and wait for direction.
- **License is optional**: A missing Apache-2.0 license is not a migration blocker. Respect `license_decision: skip` and do not re-ask on resume.
- **CODEOWNERS group entries**: GitLab groups are not necessarily GitHub teams. Resolve them separately and use a target-org team slug only after confirming it exists and has write access.

## Troubleshooting

### MR creation fails with 403
You need Maintainer+ access on the GitLab project. Ask the repo owner (check CODEOWNERS or the Jira ticket) to grant access.

### LDAP lookup returns nothing for a CODEOWNERS handle
The user running the migration needs a valid Kerberos ticket — run `kinit` and retry. If it still
returns nothing, the person may not have a GitHub URL set in their Red Hat social profile; flag it
for manual resolution instead of guessing.
