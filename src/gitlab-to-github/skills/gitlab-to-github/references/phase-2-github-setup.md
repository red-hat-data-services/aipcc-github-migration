# Phase 2: GitHub Repo Setup

Push the cleaned-up code from GitLab to the target GitHub org.

## Items

- **[human] Repo exists in target org** — If `detected.github_state == "absent"`, the user needs to request the repo. The process differs by org:
  Submit the repo request Google Form (ask the team chat if you don't have the URL yet). Phase 0 should have prompted for this already.
  
  Skip if repo already exists. If Phase 0 already prompted the form submission, just confirm the repo is now available: `gh repo view <target_org>/<repo> --json isEmpty 2>/dev/null`
- **[automatable] Add github remote** — Add the GitHub repo as a second remote. Skip if `detected.github_state == "has_content"`.
- **[automatable] Push main to GitHub** — Push main branch (and any release branches) to GitHub. Skip if `detected.github_state == "has_content"`.
- **[automatable] Push tags to GitHub** — Push tags only when `push_tags: true`. Skip when `push_tags` is false or absent.

## Automation Details

### Adding the GitHub remote

```bash
cd src/<repo>
git remote add github git@github.com:<target_org>/<repo>.git
```

Verify remotes:
```bash
git remote -v
# Should show both 'origin' (GitLab) and 'github' (GitHub)
```

### Pushing to GitHub

```bash
# Ensure local main is up to date (cleanup MR should be merged by now)
git fetch origin
git checkout main
git merge --ff-only origin/main

# Push to GitHub
git push github main
```

For repos with release branches that should also be pushed:
```bash
# Push all branches classified as "keep" by Phase 0
for branch in $(yq '.detected.branches_to_keep[]' .claude/migrations/$REPO/manifest.yaml); do
  git push github "$branch"
done
```

Before pushing kept branches, ensure each local branch is current with its
GitLab `origin/<branch>` counterpart. This phase refreshes `main` explicitly;
it assumes other kept branches have already been updated locally.

### Pushing tags

This is opt-in through `push_tags: true` in the migration manifest. Before
enabling it, ensure the local clone contains exactly the tags intended for
GitHub. `git fetch --tags` adds tags but does not remove local tags that were
deleted from GitLab, so review and remove stale local tags first.

```bash
git fetch origin --tags
git tag -l
git push github --tags
```

### Verifying the push

```bash
gh repo view <target_org>/<repo> --json defaultBranchRef,isEmpty
git ls-remote --tags github
```

## Gotchas

- **Merge cleanup MR first**: The push must include the Phase 1 cleanup commits (README, CODEOWNERS, license if added). Always `git fetch origin && git merge --ff-only origin/main` before pushing.
- **SSH key access**: `git push github` uses SSH. If the user hasn't set up SSH for GitHub, they'll need to do that first or use HTTPS instead:
  ```bash
  git remote set-url github https://github.com/<org>/<repo>.git
  ```
- **Force push risk**: Never use `--force`. If the push fails because the GitHub repo has existing content, investigate — don't overwrite.
- **Empty repos**: GitHub repos created via the request form are sometimes initialized with a README. If so, the push will fail unless you pull first. Ask the user whether to force-push (overwriting the auto-generated README) or rebase.

## Troubleshooting

### Push rejected: "non-fast-forward"
The GitHub repo has commits that don't exist locally (usually an auto-generated initial commit). Options:
1. If the GitHub repo only has an auto-generated README: `git push github main --force` (with user approval)
2. If it has real content: investigate what's there before deciding

### Permission denied (publickey)
SSH key isn't configured for GitHub. User needs to add their SSH key to their GitHub account, or switch the remote to HTTPS.

### Repository not found
Either the repo doesn't exist yet, or the user doesn't have push access. Check org membership via app-interface (`rhoai/dev` role).
