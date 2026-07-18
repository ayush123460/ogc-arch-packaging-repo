# OPERATIONS.md — ogc-arch-repo runbook

> Operator reference for the collection pipeline: the full `collect.yml` workflow spec, operational characteristics, manual operation, health checks, and failure-mode recovery.

## The collection workflow

`.github/workflows/collect.yml` is the single workflow that owns `ogc.db.tar.gz`. It runs on an hourly cron and on manual dispatch. It never builds packages; it only ingests (from GitHub release assets and/or OCI/ORAS images), signs, merges, and uploads.

### Triggers
- `schedule: cron: "0 * * * *"` — hourly, on the hour.
- `workflow_dispatch` — manual fallback / ad-hoc runs, with an optional `repo` string input to collect from a specific source only (empty = collect from all sources listed in `packages.toml`). The `repo` value is matched against both `[[packages]].repo` and `[[images]].source_repo`, so a manual run can target either source type.

### Concurrency
```yaml
concurrency:
  group: collect-publish
  cancel-in-progress: false
```
Prevents overlapping runs. A run in progress is allowed to finish; a newly-triggered run waits. This preserves the single-writer invariant on `ogc.db.tar.gz`.

### Runner
`archlinux/archlinux:base-devel` container image — provides `repo-add`, `gpg`, `pacman`, `bsdtar` without needing to install Arch tools on an Ubuntu runner. `oras` and `cosign` are installed at runtime via their official setup actions (see step-by-step below).

### Step-by-step behavior

1. **Checkout** this repo (to read `packages.toml`).
2. **Install tools** — `pacman -Syu --noconfirm aws-cli` (the base-devel image already has `repo-add`, `gpg`, `bsdtar`), then install `oras` via `oras-project/setup-oras` and `cosign` via `sigstore/cosign-installer`. Both actions work inside the Arch container.
3. **Import PGP key** — `echo "$PGP_KEY" | gpg --import --batch` using the `PGP_SIGNING_KEY` secret.
4. **Fetch existing repo DB** — download `ogc.db.tar.gz` (and `.sig` if present) from the bucket's public URL into the working directory. Tolerate failure: on the first run ever, or if the DB was deleted, no file exists yet and `repo-add` will create a fresh one. On a non-first run, a fetch failure is fatal (see [Failure modes & recovery](#failure-modes--recovery)).
5. **Collect & merge new packages — release-asset pass** — iterate `[[packages]]` entries (pseudocode below).
6. **Collect & merge new packages — OCI/ORAS pass** — iterate `[[images]]` entries: resolve tag, verify cosign signature, `oras pull`, then dedup/sign/merge each extracted file (pseudocode below).
7. **Upload to S3** — upload only changed files: newly-added `*.pkg.tar.zst` + `*.pkg.tar.zst.sig`, and always the updated `ogc.db.tar.gz` + `ogc.db.tar.gz.sig`. If nothing was added across **both** passes, skip the upload entirely.

### Core collect & merge logic (pseudocode)

```
config   = parse_toml("packages.toml")           # {packages: [...], images: [...]}
filter   = workflow_dispatch.repo                # empty for scheduled runs
packages = filter ? [p for p in config.packages if p.repo == filter]       : config.packages
images   = filter ? [i for i in config.images   if i.source_repo == filter]: config.images
added_files = []

# --- Pass 1: GitHub release assets ([[packages]]) ---
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

# --- Pass 2: OCI/ORAS images ([[images]]) ---
for img in images:
    if img.tag is set:
        resolved_tag = img.tag
    else:
        release = gh_release_view(img.source_repo)        # -> {tagName}
        version  = release.tagName.lstrip("v")            # "7.1.3-ogc3.2"
        tags     = oras_repo_tags(img.image)              # all tags in the OCI repo
        builds   = [t for t in tags if t matches ^<version>\.[0-9]+$]
        if builds is empty:
            warn("no OCI build for {img.source_repo} latest version {version}; skipping")
            continue
        resolved_tag = max(builds)                        # highest build_num, e.g. "7.1.3-ogc3.2.5"

    # Supply-chain verification: fail fast if the image is not signed by the source's workflow.
    identity = img.cosign_identity or f"^https://github.com/{img.source_repo}/.github/workflows/.*@refs/.*"
    cosign_verify(
        "--certificate-identity-regexp", identity,
        "--certificate-oidc-issuer",   "https://token.actions.githubusercontent.com",
        f"{img.image}:{resolved_tag}")
    # If verify fails -> exit non-zero; do not pull or ingest anything from this image.

    workdir = f"oci/{slug(img.image)}/{resolved_tag}"
    oras_pull(f"{img.image}:{resolved_tag}", "-o", workdir)   # extracts *.pkg.tar.zst into workdir

    for file in glob(f"{workdir}/{img.asset_glob}"):
        if db_contains_entry(ogc.db.tar.gz, file.name):
            continue
        move(file, ".")                              # place alongside ogc.db.tar.gz for repo-add
        gpg_detach_sign(file.name)                   # -> file.name + ".sig"
        repo_add("--sign", "ogc.db.tar.gz", file.name)
        added_files.append(file.name)

if added_files:
    upload_to_s3(added_files + [f+".sig" for f in added_files])
    upload_to_s3(["ogc.db.tar.gz", "ogc.db.tar.gz.sig"])
```

### Tag resolution details (OCI pass)

- The source repo's latest release tag is read with `gh release view --json tagName -R <source_repo>`.
- The leading `v` is stripped to produce `<version>` (e.g. `v7.1.3-ogc3.2` → `7.1.3-ogc3.2`).
- All tags in the OCI repo are listed with `oras repo tags <image>`.
- Tags matching `^<version>\.[0-9]+$` are candidate build tags; the numerically highest `<N>` wins. This mirrors the source build workflow, which tags each build as `<version>.<build_num>` and increments `<build_num>` on rebuilds of the same content.
- If no candidate exists, the source is skipped with a warning (the build may not have finished publishing yet). The rest of the run continues.
- If `[[images]].tag` is set explicitly, all of the above is bypassed and that tag is used verbatim.

### Cosign verification details (OCI pass)

- Verification uses keyless cosign (sigstore) with GitHub Actions OIDC as the certificate issuer, matching how the source build workflow signs.
- The default `--certificate-identity-regexp` is `^https://github.com/<source_repo>/.github/workflows/.*@refs/.*`, which accepts any workflow in the source repo signing from any ref. Override per-entry with `cosign_identity` to tighten this (e.g. pin to `@refs/tags/v.*` for release-only signing).
- Verification runs **before** `oras pull`. On failure, the workflow exits non-zero and does not extract or ingest anything from that image. Already-ingested packages from other sources in the same run are not uploaded either — the whole run aborts to preserve the "no partial publishes" invariant.

### Dedup / idempotency algorithm

The database is the single source of truth for "what has already been ingested." A package (whether from a GitHub release asset or an extracted OCI layer) is considered already-ingested if its filename appears as a `%FILENAME%` entry inside `ogc.db.tar.gz`. The same algorithm covers both ingestion paths — no source-type distinction is needed. Concretely:

```bash
# Check whether <filename> is already an entry in the DB.
bsdtar -xOzf ogc.db.tar.gz 2>/dev/null \
    | strings \
    | grep -Fx -- "<filename>"
```

If the grep matches, skip the file. If not, download/extract + sign + `repo-add` it.

This makes the workflow **idempotent and stateless**: re-running it changes nothing if there is nothing new. No state file, no commit-back-to-repo, no separate "last ingested" record. If the DB ever drifts (e.g. a previous run partially failed), the next run simply re-ingests whatever is missing from either source type.

### What gets signed
- Each new `*.pkg.tar.zst` → `gpg --detach-sign --batch` → produces `<filename>.sig`.
- `ogc.db.tar.gz` → `repo-add --sign` produces `ogc.db.tar.gz.sig`.

### When the upload is skipped
If `added_files` is empty (every `[[packages]]` source's latest assets **and** every `[[images]]` source's latest build are already in the DB), the workflow does **not** upload anything. This avoids unnecessary S3 writes and keeps the public DB byte-identical.

## Operational characteristics

- **Poll cadence:** hourly (`cron: "0 * * * *"`).
- **Expected lag:** ≤ 1 hour between a source repo publishing a release and the package appearing in `ogc.db.tar.gz`. GitHub scheduled workflows can be delayed during busy periods, so in practice lag is usually under an hour but is not guaranteed to be exactly one hour.
- **Statelessness:** `ogc.db.tar.gz` is the single source of truth. No state file, no commit-back-to-repo. Re-running the workflow is always safe.
- **Single writer / no race:** only this repo writes the DB; `concurrency` prevents overlapping runs. No S3-level locking required.
- **Cost:** this repo is public, so standard GitHub-hosted runners are free (see [GitHub Actions billing](https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions)). Hourly cron is effectively $0 and does not consume the org's private-repo minute allowance. Scheduled runs that find nothing new still consume a runner minute or two for setup, but at zero dollar cost.

## Manual operation

The same `collect.yml` accepts `workflow_dispatch` with an optional `repo` string input. The value is matched against both `[[packages]].repo` and `[[images]].source_repo`, so the same escape hatch works for either source type.

**Collect from all listed sources now** (don't wait for cron):
```bash
gh workflow run collect.yml -R <this-repo>
```

**Collect from a specific release-asset source now:**
```bash
gh workflow run collect.yml -R <this-repo> -f repo=OpenGamingCollective/asusctl
```

**Collect from a specific OCI/ORAS source now:**
```bash
gh workflow run collect.yml -R <this-repo> -f repo=OpenGamingCollective/kernel-packages
```
(`repo=` takes the `source_repo` value from the `[[images]]` entry, not the OCI image ref.)

This is the escape hatch for:
- Forcing ingestion after publishing a release (or a new OCI build) without waiting up to an hour.
- Re-running after a failed scheduled run.
- Ingesting a newly-added `packages.toml` entry immediately.

## Health check

Quick ways to verify the pipeline is healthy:

1. **Last run status** — check the Actions tab of this repo for the most recent `collect.yml` run. A green run with "nothing to add" is healthy. A red run needs attention (see [Failure modes & recovery](#failure-modes--recovery)).
2. **DB freshness** — verify the published database is recent:
   ```bash
   curl -I "<BUCKET_PUBLIC_URL>/ogc.db.tar.gz"
   ```
   Check the `Last-Modified` header. If it is older than a few hours despite known new releases being published, a scheduled run was likely missed or failed — trigger a manual run.
3. **DB integrity** — spot-check that the DB lists the packages you expect:
   ```bash
   curl -sf "<BUCKET_PUBLIC_URL>/ogc.db.tar.gz" -o /tmp/ogc.db.tar.gz
   bsdtar -xOzf /tmp/ogc.db.tar.gz | strings | grep -E '^(asusctl|rog-control-center|supergfxctl|linux-ogc)-'
   ```
   Missing packages that you know have been released indicate a fetch or merge problem — trigger a manual run.
4. **OCI source freshness** (for each `[[images]]` entry) — confirm the expected build tag exists in the OCI registry and matches the source repo's latest release:
   ```bash
   # Source repo's latest release tag:
   gh release view -R OpenGamingCollective/kernel-packages --json tagName -q .tagName
   # Tags present in the OCI repo (look for <version-stripped-of-v>.<N>):
   oras repo tags ghcr.io/opengamingcollective/kernel-packages-arch
   ```
   A latest release with no matching build tag in the OCI repo means the source build hasn't published yet (or failed); the workflow will skip it with a warning.
5. **OCI signature validity** (for each `[[images]]` entry) — confirm the image still verifies with cosign using the same identity the workflow uses:
   ```bash
   cosign verify \
     --certificate-identity-regexp='^https://github.com/OpenGamingCollective/kernel-packages/.github/workflows/.*@refs/.*' \
     --certificate-oidc-issuer='https://token.actions.githubusercontent.com' \
     ghcr.io/opengamingcollective/kernel-packages-arch:<resolved_tag>
   ```
   A failure here explains why a scheduled run would have aborted before ingesting that image.

## Failure modes & recovery

| Failure | Cause | Behavior | Recovery |
|---|---|---|---|
| **S3 unreachable on DB fetch** | Network blip, endpoint down, wrong endpoint config. | The `curl` for `ogc.db.tar.gz` fails. The workflow tolerates this only if the DB genuinely doesn't exist yet (first run). On a non-first run, a missing DB fetch means the workflow would rebuild the DB from scratch — which would lose all previously-ingested packages not present in current releases. **The workflow treats a fetch failure as fatal on non-first runs** and exits non-zero rather than silently rebuild. | Fix S3 connectivity/credentials and re-run. |
| **S3 unreachable on upload** | Network blip, endpoint down. | Upload step fails; workflow exits non-zero. The DB and signed packages are present locally from the merge step. | Fix S3 and re-run. Idempotency ensures no double-add. |
| **DB fetch fails on the very first run** | Bucket is empty / DB was deleted. | Tolerated. `repo-add` creates a fresh `ogc.db.tar.gz` from the first ingested package. | None needed — this is the expected first-run path. |
| **A source repo's latest release has no matching assets** | Source repo tagged a release but didn't attach `*.pkg.tar.zst`, or the asset glob is wrong. | The repo is skipped; no error. Other repos in the list are still processed. | Fix the source repo's release (attach assets) or fix the glob in `packages.toml`, then re-run. |
| **An OCI source's latest release has no matching build tag** | Source repo tagged a release but the OCI build hasn't published yet, or the build workflow failed, or `[[images]].image` points at the wrong registry path. | The image is skipped with a warning (`no OCI build for <source_repo> latest version <version>; skipping`). Other sources in the run are still processed. | Wait for the source build to publish, or fix the source build, or fix the `[[images]]` entry in `packages.toml`, then re-run. |
| **Cosign signature missing or fails to verify** | The OCI image was pushed without signing, was signed by a different identity than `cosign_identity` expects, or was tampered with after pushing. | The workflow exits non-zero at the verify step **before** pulling or ingesting anything from that image. The offending `image:tag` is surfaced in the failure message. No upload happens; the public DB is unaffected. (Other already-merged packages from the same run are also not uploaded — the run aborts to preserve the no-partial-publish invariant.) | Re-push a correctly-signed image from the source workflow, or correct the `cosign_identity` / `tag` in `packages.toml`, then re-run. |
| **`oras repo tags` / `oras pull` fails** | Network blip, registry down, image or tag deleted, rate-limited by GHCR (anonymous pull limits). | The workflow exits non-zero at the OCI pass. No upload happens. | Re-run after the registry is reachable. If rate-limited, consider switching the entry to authenticated pulls (future work). |
| **Extracted OCI image has no files matching `asset_glob`** | The source build pushed an OCI image whose layers don't contain `*.pkg.tar.zst` (e.g. wrong files pushed), or `asset_glob` is wrong. | The image is skipped with a warning; other sources in the run are still processed. | Fix the source build's `oras push` invocation or fix `asset_glob` in `packages.toml`, then re-run. |
| **A downloaded package is corrupt or malformed** | Truncated upload on the source side, network corruption. | `repo-add` will fail when trying to add the bad file. The workflow exits non-zero. No upload happens, so the public DB is unaffected. | Re-attach a valid asset (release-asset source) or re-push a valid OCI image (OCI source), then re-run. The corrupt file is not ingested. |
| **`repo-add` fails for one package** | Malformed package, disk full, transient error. | The workflow exits non-zero at the merge step. No upload happens. The DB on S3 remains at its previous good state. | Re-run after resolving the cause. Already-ingested packages are skipped due to idempotency. |
| **Scheduled cron run is missed / delayed** | GitHub Actions scheduled-workload jitter, or a run was cancelled. | No ingestion happens for that interval. The next scheduled run picks up everything missed, including any releases or OCI builds published in the meantime (idempotency makes this safe). | Optionally trigger a manual run to catch up immediately. |
| **Manual re-run triggered while a scheduled run is in progress** | Human error or overlap. | `concurrency` queues the second run; it executes after the first completes. The second run finds nothing new (the first already ingested it) and skips upload. | None needed — safe by design. |
| **PGP key import fails** | Malformed secret, expired key. | Signing step fails; workflow exits non-zero before any upload. | Re-set the `PGP_SIGNING_KEY` secret with a valid key and re-run. |
| **A source repo is deleted or made private** | Repo archived/removed, or visibility changed. | `gh release view` fails for that repo. The workflow skips that source with a warning and continues with the rest, rather than aborting the whole run. Applies to both `[[packages]].repo` and `[[images]].source_repo`. | Remove the entry from `packages.toml` to stop the warning noise. |

## References

- GitHub Actions `schedule` event: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule
- GitHub Actions `workflow_dispatch` event: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch
- GitHub Actions `concurrency` syntax: https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#concurrency
- GitHub Actions billing (public repos = free standard runners): https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions
- `repo-add` man page: https://man.archlinux.org/man/repo-add.1
- `gh release view` / `gh release download` CLI reference: https://cli.github.com/manual/gh_release
- ORAS CLI: https://oras.land/docs/
- `oras pull`: https://oras.land/docs/commands/oras_pull
- `oras repo tags`: https://oras.land/docs/commands/oras_repo_tags
- `oras-project/setup-oras` action: https://github.com/oras-project/setup-oras
- cosign (sigstore): https://github.com/sigstore/cosign
- `sigstore/cosign-installer` action: https://github.com/sigstore/cosign-installer
- Keyless cosign with GitHub Actions OIDC: https://docs.sigstore.dev/cosign/signing/overview/
- GitHub Container Registry (GHCR): https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry
