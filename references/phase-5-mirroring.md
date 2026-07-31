# Phase 5: GitLab Pull Mirroring

Set up GitLab to pull-mirror the GitHub repo so downstream `include: project:` references keep working.

> **This phase changed.** Mirrors are no longer configured by hand with your own
> `glab` credentials. They are configured **and** kept authenticated centrally by the
> `gitlab-mirror-sync` chart in the `aipcc-claudio` repo. You register a repo by adding
> one entry to that chart's config; the chart does the rest. Do **not** run a manual
> `glab api ... mirror=true` with a personal GitHub PAT — that is exactly what broke
> mirroring in AIPCC-28357 (personal credentials can't push to protected branches).

## Why central (the identity fix)

A GitLab pull mirror needs a credential that can *read* the GitHub source. A personal
PAT works until the person's access changes, and it embeds an identity that isn't
allowed to write to protected branches on the GitLab side — so the mirror fails with
`not allowed to push code to protected branches`. This is an **identity** problem, not a
branch-protection-grant problem.

The fix (AIPCC-28668) uses two **service** identities instead of a person's:

| Identity | Role |
|----------|------|
| GitHub App `aipcc-gitlab-repo-mirror` | Read-only (Contents + Metadata) on the GitHub source. Its short-lived installation token is embedded in the GitLab mirror's `import_url`. |
| GitLab bot `aipcc-cicd-bot` (`BOT_PAT`) | Authenticates to the GitLab API to configure the mirror. Owns the mirror, so no protected-branch error. |

GitHub App installation tokens expire after ~1 hour, so they can't be a static
credential. The `gitlab-mirror-sync` CronJob (schedule `*/45 * * * *`) refreshes them:
it mints a JWT from the App key, exchanges it for a fresh installation token, and PUTs
`{mirror: true, import_url: https://x-access-token:<token>@github.com/<repo>.git}` to the
GitLab project via the bot PAT. Every repo listed in the chart's config is refreshed on
each run.

## Items

- **[human] Confirm the GitHub App is installed on the source repo** — `aipcc-gitlab-repo-mirror` must be installed on the GitHub repo, or the installation token can't read it. New repos need to be added to the App installation (admin action, tracked under AIPCC-28668).
- **[automatable] Register the repo in `gitlab-mirror-sync`** — Add one entry to the chart's `mirrors` list and open an MR to `aipcc-claudio`.
- **[human] Verify the mirror syncs** — After the chart deploys, confirm the GitLab project shows `mirror: true` with a fresh `import_url` and pulls content.
- **[human] Confirm downstream includes work** — Trigger a pipeline in each project that uses `include: project:` from this repo and confirm it still builds.

## Automation Details

### Registering a repo (the only edit you make)

The chart reads its mirror list from `gitlab-mirror-sync/values.yaml` under `mirrors:`.
Helm templates that list into a ConfigMap that the sync script reads as
`/config/mirrors.json`. Each entry has exactly three fields:

```yaml
mirrors:
  - github: red-hat-data-services/aipcc-konflux-data   # GitHub source (source of truth)
    gitlab_project: redhat/rhel-ai/konflux-data        # GitLab project path being mirrored
    gitlab_host: gitlab.com
  # add the new repo here:
  - github: <github_org>/<repo>
    gitlab_project: <gitlab/group/path/repo>
    gitlab_host: gitlab.com
```

- `github` — `org/repo` of the GitHub source. The App must be installed on this repo.
- `gitlab_project` — full path of the GitLab project that will pull-mirror it.
- `gitlab_host` — `gitlab.com` for public GitLab (use the internal host only for internal projects).

Open an MR to `aipcc-claudio` with this change. The chart is deployed from that repo's
CI (`deploy-mirror-sync` job / `make deploy-mirror-sync`); once your entry is on `main`
and deployed, the next CronJob run (within ~45 min) establishes and authenticates the
mirror. No manual `glab` call is needed.

### How the sync configures the mirror

You don't run this — it's what the CronJob does per entry, shown so you can reason about
failures:

1. `POST /app/installations/<id>/access_tokens` → fresh GitHub installation token.
2. `GET /projects/<url-encoded gitlab_project>` (bot PAT) → GitLab project id.
3. `PUT /projects/<id>` (bot PAT) with
   `{"mirror": true, "import_url": "https://x-access-token:<token>@github.com/<github>.git"}`.

Note it sets `mirror` and `import_url` only — it does **not** set
`mirror_trigger_builds`. If the mirrored project itself must run pipelines on each sync,
that flag has to be enabled separately (see Gotchas).

### Verifying the mirror (read-only, safe to run)

```bash
PROJECT_ID=$(glab api "projects/$(echo "$GITLAB_PROJECT" | sed 's|/|%2F|g')" | jq '.id')
glab api "projects/$PROJECT_ID" | jq '{mirror, import_url}'
```

Expect `mirror: true` and an `import_url` of the form
`https://x-access-token:*****@github.com/<org>/<repo>.git`. (GitLab masks the token in
API responses.) If you have cluster access, the CronJob logs are the ground truth:

```bash
oc -n rhel-ai-cicd--claudio logs job/<gitlab-mirror-sync-job>   # "  OK" per mirror, or "  FAILED: <status>"
```

### Checking downstream projects

If the repo's `detected.included_by` is non-empty, trigger a pipeline in each downstream
project to confirm the `include: project:` refs still resolve against the mirror:

```bash
glab api "projects/$DOWNSTREAM_ID/pipeline" --method POST -f ref=main
```

## Gotchas

- **App must be installed per source repo**: The installation token can only read repos the App is installed on. A new repo needs adding to the App installation first, or the sync returns 403/404 for it.
- **422 until the GitHub source has content**: GitLab rejects a mirror `import_url` for an empty GitHub repo. Push at least one commit to GitHub before registering.
- **All repos use the App token**: Even public GitHub repos are mirrored via `x-access-token:<installation-token>`, not an anonymous URL. This keeps one consistent, rotating identity.
- **`mirror_trigger_builds` is not set by the sync**: The central sync only keeps content current. If a mirrored project needs its own pipelines to run on each mirror pull, enable that flag on the project separately.
- **GitLab Premium required**: Pull mirroring needs Premium or higher. Red Hat has Ultimate, so this is always available.
- **GitLab becomes read-only**: Pushes go to GitHub only. Direct pushes to GitLab are overwritten on the next sync.
- **Refresh cadence is ~45 min**: A newly registered repo (or one whose token just expired) may show a stale/failing mirror until the next CronJob run.

## Troubleshooting

### Sync logs show 403 or 404 for a repo
The GitHub App isn't installed on that source repo (or lacks Contents/Metadata read).
Add the repo to the `aipcc-gitlab-repo-mirror` installation (AIPCC-28668 admin action).

### Sync logs show 422 on the `PUT`
The GitHub source repo is empty, or the `gitlab_project` path is wrong. Confirm the
GitHub repo has content and the GitLab path resolves with
`glab api "projects/$(echo "$GITLAB_PROJECT" | sed 's|/|%2F|g')"`.

### Sync logs show 401 on the GitLab call
The bot PAT (`aipcc-cicd-bot`) is expired or lacks API scope. This is an infra fix on
the `gitlab-mirror-sync` secret, not something to patch per repo.

### Mirror shows stale data
Check the most recent CronJob run's logs. If the run succeeded but GitLab is stale, force
a sync from the GitLab UI (**Settings → Repository → Mirroring repositories → Retry**).

### Downstream `include: project:` fails after mirroring
The branch or file path may have moved. Confirm the included file exists at the expected
path on the mirrored branch.

## Reverting

To stop mirroring a repo, remove its entry from `mirrors:` in
`gitlab-mirror-sync/values.yaml` and open an MR. The CronJob will stop refreshing it. To
also turn the mirror off on the GitLab side (so the last token isn't left in place), have
someone with bot access set it once:

```bash
glab api "projects/$PROJECT_ID" --method PUT -f mirror=false
```
