# Phase 6: Announce

Declare the migration complete and redirect the team to the new GitHub repo.

## Items

- **[automatable] Update README with GitHub URL** — Add a prominent notice that GitHub is now the canonical source.
- **[human] Notify team** — Send a message to the team (Slack, email, or both) announcing the migration.
- **[human] Update external references** — Update Jira tickets, Confluence docs, and any other external systems that link to the old GitLab repo.

## Automation Details

### Updating the README

Add a notice block at the top of the README (on GitHub, push to main):

```markdown
> **This repository has moved to GitHub.**
> The canonical source is now [`github.com/<target_org>/<repo>`](https://github.com/<target_org>/<repo>).
> The GitLab repository at `gitlab.com/<source_gitlab>` is a read-only mirror.
```

Also update any clone URLs, badge URLs, or workflow references in the README body.

```bash
cd src/<repo>
# Edit README.md with the notice
git add README.md
git commit -s -m "docs: mark GitHub as canonical source, GitLab as mirror"
git push github main
```

The GitLab mirror will pick up this change automatically on the next sync (~5 min).

### Announcement template

Provide this to the user for their team notification:

```
Subject: [Migration] <repo> moved to GitHub

Hi team,

<repo> has been migrated from GitLab to GitHub:

- New canonical repo: https://github.com/<target_org>/<repo>
- GitLab mirror (read-only): https://gitlab.com/<source_gitlab>

What changed:
- All new PRs and issues should go to GitHub
- GitLab is now a read-only pull mirror (syncs every ~5 min)
- CI runs on GitHub Actions
- Container images still push to quay.io/<quay_org>/<repo>

What didn't change:
- Downstream GitLab CI jobs using `include: project:` still work (via mirror)
- Container image location in Quay is unchanged

Please update your local remotes:
  git remote set-url origin git@github.com:<target_org>/<repo>.git

Questions? Ping me or Ken.
```

### External reference checklist

Prompt the user to check:
- **Jira tickets**: Update the repo URL in AIPCC-18266 or the repo-specific ticket
- **Confluence docs**: Search for the old GitLab URL and update
- **Slack bookmarks**: If the team has pinned repo links
- **CI/CD references**: Any other systems pointing to the GitLab URL

## Gotchas

- **Don't archive GitLab yet**: The GitLab repo should remain as a mirror, not be archived. Archiving breaks the mirror and downstream `include: project:` references.
- **Timing**: Send the announcement after verifying mirroring works (Phase 5) and CI passes (Phase 4). Don't announce a migration that's half-done.
- **Clone URL update**: Teammates with existing clones need to update their remote. Include the `git remote set-url` command in the announcement.

## Troubleshooting

### Team didn't get the notification
If using Slack, check the channel. If email, check that the distribution list is correct. For AIPCC, the team channel and individual pings to active contributors are both appropriate.

### Someone pushed to GitLab after migration
The mirror will overwrite their changes on the next sync. Help them recover their work from the GitLab reflog and push it to GitHub instead.
