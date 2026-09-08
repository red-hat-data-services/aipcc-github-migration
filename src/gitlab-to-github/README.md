# gitlab-to-github

A Claude Code plugin that guides an AIPCC repository through migrating from GitLab to GitHub,
with stateful progress tracking across sessions.

## Skills

- **`skills/gitlab-to-github`** — the migration skill itself. Walks the user through pre-flight
  assessment, repo cleanup, GitHub setup, Quay OIDC federation, CI translation, and GitLab pull
  mirroring. See `skills/gitlab-to-github/SKILL.md` for the full invocation flow.
- **`skills/ldap`** — Red Hat LDAP lookups via `ldapsearch`. Used by the migration skill to
  translate GitLab usernames in CODEOWNERS to GitHub usernames, but usable standalone for any
  Red Hat people/group lookup.

## Development

See the repo-root `CLAUDE.md` for architecture notes (stateful manifest design, phase-skip logic,
progressive disclosure via `references/`) and `README.md` for installation instructions.
