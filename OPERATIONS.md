# OPERATIONS.md — ogc-arch-repo runbook

> Operator reference for the collection pipeline: the full `collect.yml` workflow spec, operational characteristics, manual operation, health checks, and failure-mode recovery.

## The collection workflow

`.github/workflows/collect.yml` is the single workflow that owns `ogc.db.tar.gz`. It runs on an hourly cron and on manual dispatch. It never builds packages; it only ingests, signs, merges, and uploads.

### Triggers
- `schedule: cron: "0 * * * *"` — hourly, on the hour.
- `workflow_dispatch` — manual fallback / ad-hoc runs, with an optional `repo` string input to collect from a specific source repo only (empty = collect from all packages listed in `packages.toml`).

### Concurrency
```yaml
concurrency:
  group: collect-publish
  cancel-in-progress: false
```
Prevents overlapping runs. A run in progress is allowed to finish; a newly-triggered run waits. This preserves the single-writer invariant on `ogc.db.tar.gz`.

### Runner
`archlinux/archlinux:base-devel` container image — provides `repo-add`, `gpg`, `pacman`, `bsdtar` without needing to install Arch tools on an Ubuntu runner.

### Step-by-step behavior

1. **Checkout** this repo (to read `packages.toml`).
2. **Install tools** — `pacman -Syu --noconfirm aws-cli` (the base-devel image already has `repo-add`, `gpg`, `bsdtar`).
3. **Import PGP key** — `echo "$PGP_KEY" | gpg --import --batch` using the `PGP_SIGNING_KEY` secret.
4. **Fetch existing repo DB** — download `ogc.db.tar.gz` (and `.sig` if present) from the bucket's public URL into the working directory. Tolerate failure: on the first run ever, or if the DB was deleted, no file exists yet and `repo-add` will create a fresh one.
5. **Collect & merge new packages** — the core logic (pseudocode below).
6. **Upload to S3** — upload only changed files: newly-added `*.pkg.tar.zst` + `*.pkg.tar.zst.sig`, and always the updated `ogc.db.tar.gz` + `ogc.db.tar.gz.sig`. If nothing was added, skip the upload entirely.

### Core collect & merge logic (pseudocode)

```
packages = parse_toml("packages.toml")          # or [{repo: <input>}] if workflow_dispatch.repo provided
added_files = []

for pkg in packages:
    release = gh_release_view(pkg.repo)         # -> {tagName, assets[]}
    for asset in release.assets:
        if not matches(asset.name, pkg.asset_glob):
            continue
        # Idempotency: skip if this package filename is already an entry in the DB.
        # The DB entry's %FILENAME% matches the asset's name.
        if db_contains_entry(ogc.db.tar.gz, asset.name):
            continue
        # New package — download, sign, merge.
        gh_release_download(pkg.repo, pattern=asset.name)
        gpg_detach_sign(asset.name)             # -> asset.name + ".sig"
        repo_add("--sign", "ogc.db.tar.gz", asset.name)
        added_files.append(asset.name)

if added_files:
    upload_to_s3(added_files + [f+".sig" for f in added_files])
    upload_to_s3(["ogc.db.tar.gz", "ogc.db.tar.gz.sig"])
```

### Dedup / idempotency algorithm

The database is the single source of truth for "what has already been ingested." An asset is considered already-ingested if its filename appears as a `%FILENAME%` entry inside `ogc.db.tar.gz`. Concretely:

```bash
# Check whether <filename> is already an entry in the DB.
bsdtar -xOzf ogc.db.tar.gz 2>/dev/null \
    | strings \
    | grep -Fx -- "<filename>"
```

If the grep matches, skip the asset. If not, download + sign + `repo-add` it.

This makes the workflow **idempotent and stateless**: re-running it changes nothing if there is nothing new. No state file, no commit-back-to-repo, no separate "last ingested" record. If the DB ever drifts (e.g. a previous run partially failed), the next run simply re-ingests whatever is missing.

### What gets signed
- Each new `*.pkg.tar.zst` → `gpg --detach-sign --batch` → produces `<filename>.sig`.
- `ogc.db.tar.gz` → `repo-add --sign` produces `ogc.db.tar.gz.sig`.

### When the upload is skipped
If `added_files` is empty (every listed source repo's latest assets are already in the DB), the workflow does **not** upload anything. This avoids unnecessary S3 writes and keeps the public DB byte-identical.

## Operational characteristics

- **Poll cadence:** hourly (`cron: "0 * * * *"`).
- **Expected lag:** ≤ 1 hour between a source repo publishing a release and the package appearing in `ogc.db.tar.gz`. GitHub scheduled workflows can be delayed during busy periods, so in practice lag is usually under an hour but is not guaranteed to be exactly one hour.
- **Statelessness:** `ogc.db.tar.gz` is the single source of truth. No state file, no commit-back-to-repo. Re-running the workflow is always safe.
- **Single writer / no race:** only this repo writes the DB; `concurrency` prevents overlapping runs. No S3-level locking required.
- **Cost:** this repo is public, so standard GitHub-hosted runners are free (see [GitHub Actions billing](https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions)). Hourly cron is effectively $0 and does not consume the org's private-repo minute allowance. Scheduled runs that find nothing new still consume a runner minute or two for setup, but at zero dollar cost.

## Manual operation

The same `collect.yml` accepts `workflow_dispatch` with an optional `repo` string input.

**Collect from all listed packages now** (don't wait for cron):
```bash
gh workflow run collect.yml -R <this-repo>
```

**Collect from a specific source repo now:**
```bash
gh workflow run collect.yml -R <this-repo> -f repo=OpenGamingCollective/asusctl
```

This is the escape hatch for:
- Forcing ingestion after publishing a release without waiting up to an hour.
- Re-running after a failed scheduled run.
- Ingesting a newly-added `packages.toml` entry immediately.

## Health check

Two quick ways to verify the pipeline is healthy:

1. **Last run status** — check the Actions tab of this repo for the most recent `collect.yml` run. A green run with "nothing to add" is healthy. A red run needs attention (see [Failure modes & recovery](#failure-modes--recovery)).
2. **DB freshness** — verify the published database is recent:
   ```bash
   curl -I "<BUCKET_PUBLIC_URL>/ogc.db.tar.gz"
   ```
   Check the `Last-Modified` header. If it is older than a few hours despite known new releases being published, a scheduled run was likely missed or failed — trigger a manual run.
3. **DB integrity** — spot-check that the DB lists the packages you expect:
   ```bash
   curl -sf "<BUCKET_PUBLIC_URL>/ogc.db.tar.gz" -o /tmp/ogc.db.tar.gz
   bsdtar -xOzf /tmp/ogc.db.tar.gz | strings | grep -E '^(asusctl|rog-control-center|supergfxctl)-'
   ```
   Missing packages that you know have been released indicate a fetch or merge problem — trigger a manual run.

## Failure modes & recovery

| Failure | Cause | Behavior | Recovery |
|---|---|---|---|
| **S3 unreachable on DB fetch** | Network blip, endpoint down, wrong endpoint config. | The `curl` for `ogc.db.tar.gz` fails. The workflow tolerates this only if the DB genuinely doesn't exist yet (first run). On a non-first run, a missing DB fetch means the workflow would rebuild the DB from scratch — which would lose all previously-ingested packages not present in current releases. **The workflow should treat a fetch failure as fatal on non-first runs** and exit non-zero rather than silently rebuild. | Fix S3 connectivity/credentials and re-run. |
| **S3 unreachable on upload** | Network blip, endpoint down. | Upload step fails; workflow exits non-zero. The DB and signed packages are present locally from the merge step. | Fix S3 and re-run. Idempotency ensures no double-add. |
| **DB fetch fails on the very first run** | Bucket is empty / DB was deleted. | Tolerated. `repo-add` creates a fresh `ogc.db.tar.gz` from the first ingested package. | None needed — this is the expected first-run path. |
| **A source repo's latest release has no matching assets** | Source repo tagged a release but didn't attach `*.pkg.tar.zst`, or the asset glob is wrong. | The repo is skipped; no error. Other repos in the list are still processed. | Fix the source repo's release (attach assets) or fix the glob in `packages.toml`, then re-run. |
| **A downloaded package is corrupt or malformed** | Truncated upload on the source side, network corruption. | `repo-add` will fail when trying to add the bad file. The workflow exits non-zero. No upload happens, so the public DB is unaffected. | Re-attach a valid asset to the source release, then re-run. The corrupt file is not ingested. |
| **`repo-add` fails for one package** | Malformed package, disk full, transient error. | The workflow exits non-zero at the merge step. No upload happens. The DB on S3 remains at its previous good state. | Re-run after resolving the cause. Already-ingested packages are skipped due to idempotency. |
| **Scheduled cron run is missed / delayed** | GitHub Actions scheduled-workload jitter, or a run was cancelled. | No ingestion happens for that interval. The next scheduled run picks up everything missed, including any releases published in the meantime (idempotency makes this safe). | Optionally trigger a manual run to catch up immediately. |
| **Manual re-run triggered while a scheduled run is in progress** | Human error or overlap. | `concurrency` queues the second run; it executes after the first completes. The second run finds nothing new (the first already ingested it) and skips upload. | None needed — safe by design. |
| **PGP key import fails** | Malformed secret, expired key. | Signing step fails; workflow exits non-zero before any upload. | Re-set the `PGP_SIGNING_KEY` secret with a valid key and re-run. |
| **A source repo is deleted or made private** | Repo archived/removed, or visibility changed. | `gh release view` fails for that repo. The workflow should skip that repo with a warning and continue with the rest, rather than aborting the whole run. | Remove the entry from `packages.toml` to stop the warning noise. |

## References

- GitHub Actions `schedule` event: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule
- GitHub Actions `workflow_dispatch` event: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch
- GitHub Actions `concurrency` syntax: https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#concurrency
- GitHub Actions billing (public repos = free standard runners): https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions
- `repo-add` man page: https://man.archlinux.org/man/repo-add.1
- `gh release view` / `gh release download` CLI reference: https://cli.github.com/manual/gh_release
