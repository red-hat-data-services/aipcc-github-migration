# Migrating product-management-configs to GitHub

Good news: since this repo has no CI pipeline and no container builds, the migration is significantly simpler than central-linter. Here is a breakdown of what applies and what you can skip.

## Steps That Apply

### 1. License Review (BLOCKER - Needs Resolution First)

The repo currently has a GPL-3.0 license, and the team standard is Apache-2.0. **Do not simply swap the license file.** GPL-3.0 and Apache-2.0 have different copyleft implications, and changing the license on an existing codebase requires:

- **Legal review:** Check with Red Hat legal whether the existing contributions can be relicensed. GPL-3.0 is copyleft; Apache-2.0 is permissive. This is not a drop-in replacement.
- **Contributor consent:** If there are contributors beyond the immediate team, their consent may be needed to relicense.
- **Dependency audit:** Confirm no config files pull in or reference GPL-only dependencies that would conflict with Apache-2.0.

**Recommendation:** File a question with Red Hat legal or the team's open-source program office before proceeding. If relicensing is not approved, the repo can still migrate to GitHub under GPL-3.0 — it just will not match the team's standard license.

### 2. Repo Cleanup (Same as Central-Linter Pattern)

- Delete any stale branches (check with the team which branches are active)
- Review and update the README to point at the new GitHub location
- Add or verify CODEOWNERS if applicable

### 3. Create the GitHub Repository

- Create the repo under `red-hat-data-services` (or `opendatahub-io`, confirm the correct org)
- Push the full history
- Set up branch protection rules on `main`

### 4. Set Up GitHub-to-GitLab Mirroring

Even without CI, downstream consumers may reference this repo via GitLab paths (e.g., `include: project:` in CI configs, or direct file references). Set up mirroring from GitHub to GitLab so existing references continue to resolve. This is the same mirroring pattern used for central-linter.

### 5. Update References and Announce

- Update any internal documentation, wiki pages, or Confluence pages that link to the GitLab URL
- Search for references in other repos (CI configs, scripts, documentation) that point to the GitLab path
- Announce to the team that the canonical location is now GitHub

## Steps You Can Skip

### No CI Pipeline Setup
There is no GitLab CI to port and no GitHub Actions workflow to create. Skip the entire CI migration phase.

### No Container Build / Quay Integration
No container images are built from this repo, so there is no need for:
- Quay OIDC configuration
- Quay robot account or static keypair setup
- Container push workflows
- Image verification or signing

### No Vault Credential Migration
Since there are no CI jobs, there are no Vault-based credentials to migrate or replace.

### No Runner Infrastructure
No need for self-hosted runners, ARC setup, or ppc64le/s390x runner considerations.

## Summary: Migration Checklist

1. [ ] **Resolve GPL-3.0 license question** (blocker -- consult legal)
2. [ ] Identify and delete stale branches
3. [ ] Update README with GitHub URL and any needed content changes
4. [ ] Apply license change if approved (or keep GPL-3.0 if not)
5. [ ] Create GitHub repo in the correct org
6. [ ] Push full git history to GitHub
7. [ ] Configure branch protection on `main`
8. [ ] Set up GitHub-to-GitLab mirroring
9. [ ] Search for and update cross-repo references to the GitLab URL
10. [ ] Announce migration to the team

## Estimated Effort

Without CI/container complexity, this migration should take a fraction of the time central-linter requires. The only potential delay is the license review, which depends on legal response time. The technical migration work itself (steps 2-10) could be completed in a single session once the license question is resolved.

## Jira Ticket Recommendation

Create a single ticket under AIPCC-16303 for this migration. Unlike central-linter, which needed multiple sub-tickets for CI, Quay, and runner work, this repo is simple enough to track in one ticket with the checklist above.
