# Phase 5: GitLab Pull Mirroring

Set up GitLab to pull-mirror the GitHub repo so downstream `include: project:` references keep working.

## Items

- **[automatable] Enable GitLab pull mirroring** — API call to configure the GitLab project to mirror from GitHub.
- **[automatable] Verify mirror syncs** — Confirm the mirror pulls and triggers builds.
- **[human] Confirm downstream includes work** — User verifies that other GitLab projects using `include: project:` from this repo still build successfully.

## Automation Details

### Getting the GitLab project ID

```bash
PROJECT_ID=$(glab api "projects/$(echo "$SOURCE_GITLAB" | sed 's|/|%2F|g')" --method GET | jq '.id')
echo "Project ID: $PROJECT_ID"
```

### Enabling pull mirroring

```bash
glab api "projects/$PROJECT_ID" --method PUT \
  -f mirror=true \
  -f import_url="https://github.com/<target_org>/<repo>.git" \
  -f mirror_trigger_builds=true
```

For private GitHub repos, include a personal access token:
```bash
-f import_url="https://<GITHUB_PAT>@github.com/<target_org>/<repo>.git"
```

### Verifying the mirror config

```bash
glab api "projects/$PROJECT_ID" --method GET | jq '{mirror, import_url, mirror_trigger_builds}'
```

Expected output:
```json
{
  "mirror": true,
  "import_url": "https://github.com/<target_org>/<repo>.git",
  "mirror_trigger_builds": true
}
```

### Triggering an immediate sync

The GitLab UI has a sync button:
**Settings → Repository → Mirroring repositories → Retry** (sync icon)

Or wait ~5 minutes for the automatic interval.

### Checking downstream projects

If `detected.included_by` is non-empty, list the downstream projects that include CI from this repo. The user should trigger a pipeline in each one to confirm they still work.

```bash
# For each downstream project, trigger a pipeline:
glab api "projects/$DOWNSTREAM_ID/pipeline" --method POST -f ref=main
```

## Gotchas

- **Public repos need no auth**: If the GitHub repo is public, the plain HTTPS URL works. No PAT needed.
- **GitLab Premium required**: Pull mirroring requires GitLab Premium or higher. Red Hat has Ultimate, so this is always available.
- **Mirror preserves all branches and tags**: Everything on GitHub will be pulled to GitLab.
- **First sync may take a moment**: Large repos with deep history can take a minute or two for the initial mirror sync.
- **GitLab becomes read-only**: After mirroring is enabled, pushes should go to GitHub only. Direct pushes to GitLab will be overwritten on the next mirror sync.

## Troubleshooting

### Mirror sync fails with 401
The GitHub PAT is invalid or expired (for private repos). Regenerate it and update the mirror URL.

### Mirror sync shows stale data
Force a sync from the GitLab UI. If it still doesn't update, check that the `import_url` points to the correct GitHub repo.

### Downstream `include: project:` fails after mirroring
The branch or file path may have changed. Check that the included file exists at the expected path on the mirrored branch.

## Reverting

To disable mirroring and return to direct GitLab development:

```bash
glab api "projects/$PROJECT_ID" --method PUT -f mirror=false
```
