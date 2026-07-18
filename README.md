# ogc-arch-repo — Central Arch Package Repository

> Publishes the `ogc` Arch Linux pacman repository — one PGP-signed home for packages from many source repos.

> **Status:** Planned, not yet implemented. This document is the scope & overview for the central packaging repository. It is self-contained: a reader landing here with no other context should understand what this repo does, how to use its output, and how to contribute a new package. Operator/runbook details live in [`OPERATIONS.md`](./OPERATIONS.md).

## Using the `ogc` repo

End users consume the published bucket as a pacman repository. To use it:

1. Add the repo to `/etc/pacman.conf`:
   ```ini
   [ogc]
   Server = <BUCKET_PUBLIC_URL>/$arch
   ```
   Replace `<BUCKET_PUBLIC_URL>` with the public URL the bucket is served at (the same value the workflow fetches `ogc.db.tar.gz` from). The `$arch` placeholder lets pacman pick the right architecture directory; if the bucket is flat (no per-arch dirs), use `Server = <BUCKET_PUBLIC_URL>` instead.

2. Import the repo's PGP key so pacman can verify package and database signatures:
   ```bash
   sudo pacman-key --recv-keys <KEY_ID>
   sudo pacman-key --lsign-key <KEY_ID>
   ```
   See [PGP key](#pgp-key) below for the fingerprint and where the public key is published.

3. Install packages as usual:
   ```bash
   sudo pacman -Syu asusctl
   ```

## PGP key

One PGP key signs every package and the repo database in `ogc`. Users trust this single key regardless of how many source repositories contribute packages.

- **Fingerprint:** `<KEY_ID>` _(placeholder — fill in once the key is generated)_
- **Public key:** published in this repo as `ogc.asc` and optionally on a public keyserver.

Until the key is generated and the fingerprint is filled in above, the `ogc` repo is not yet usable by end users.

## Adding a new package

This is the one thing a contributor does. No edits to any source repo beyond that source repo's own normal release process (tag a release, attach `*.pkg.tar.zst` assets).

1. Ensure the source repo's release process attaches `*.pkg.tar.zst` files as GitHub release assets. (How the source repo builds them is its own concern; this repo does not care.)
2. Append a `[[packages]]` block to [`packages.toml`](./packages.toml) in this repo:
   ```toml
   [[packages]]
   repo = "OpenGamingCollective/<new-package>"
   asset_glob = "*.pkg.tar.zst"
   ```
3. Commit and push to the default branch.
4. The next hourly cron run ingests the source repo's latest release assets and publishes them to `ogc`. To ingest immediately instead of waiting, run the workflow manually with the `repo` input set to the new source repo (see [`OPERATIONS.md`](./OPERATIONS.md) for manual-run syntax).

## Purpose

This repository is the single owner and publisher of the `ogc` Arch Linux pacman repository. It does **not** build packages. It **ingests** prebuilt Arch packages that other repositories attach to their GitHub releases as assets, **signs** them with one PGP key, **merges** them into a single shared repo database (`ogc.db.tar.gz`), and **publishes** the database plus all signed packages to an S3-compatible bucket. Users consume the bucket as a pacman repo.

One PGP key signs everything in the `ogc` repo, so users trust a single key regardless of how many source repositories contribute packages.

## Scope

### In scope
- Polling registered source repos' latest GitHub releases for `*.pkg.tar.zst` assets.
- Downloading new (not-yet-ingested) assets.
- GPG-signing each downloaded package.
- Merging new packages into the shared `ogc.db.tar.gz` via `repo-add`.
- Signing `ogc.db.tar.gz`.
- Uploading new package files, signatures, and the updated database to S3.
- Providing a manual trigger for ad-hoc collection runs.

### Out of scope
- Building packages from source. Source repos build and attach assets themselves.
- Editing source repositories. Adding a package requires only a commit to this repo's `packages.toml`.
- Pruning old package versions from the database or S3 (see [History & retention](#history--retention)).
- Hosting non-Arch artifacts (RPMs, debs, etc.).
- Real-time ingestion. Collection runs on an hourly schedule (see [Future upgrade path](#future-upgrade-path) for the planned automation upgrade).
- Reacting to events in source repos. GitHub Actions cannot natively react to cross-repo events; this repo polls instead.

## Architecture

```
Source repos (asusctl, supergfxctl, ...)          This repo (public)
┌────────────────────────────────┐               ┌─────────────────────────────────┐
│ release.yml:                   │               │ collect.yml (hourly cron +       │
│  1. makepkg (build)            │   GitHub      │      workflow_dispatch fallback):│
│  2. gh release upload          │──Release──────│  1. Read packages.toml           │
│     *.pkg.tar.zst as assets    │   Assets      │  2. Fetch ogc.db.tar.gz from S3  │
│                                │               │  3. For each source repo:        │
│ Builds & attaches.             │               │     gh release view --json       │
│ No S3, no PGP, no DB.          │               │     skip assets already in DB    │
│                                │               │     download new *.pkg.tar.zst   │
│                                │               │  4. gpg --detach-sign each       │
│                                │               │  5. repo-add ogc.db.tar.gz       │
│                                │               │  6. aws s3 cp packages+sig+db   │
└────────────────────────────────┘               └─────────────────────────────────┘
                                                          │
                                                   ┌──────▼──────┐
                                                   │  S3 bucket  │
                                                   │ ogc.db.tar.gz│
                                                   │ *.pkg.tar.zst│
                                                   └─────────────┘
```

### Single-writer model

Only this repository writes to `ogc.db.tar.gz`. Source repos never touch the database or the bucket. A `concurrency` group on the workflow prevents overlapping runs of this repo's own job. Because there is a single writer, no S3-level locking (e.g. conditional writes via `If-Match`) is needed.

## Repository layout

```
.
├── packages.toml                  # list of source repos to poll
├── ogc.asc                        # PGP public key (published for users)
├── .github/
│   └── workflows/
│       └── collect.yml            # hourly cron + manual dispatch workflow
├── README.md                      # this document
└── OPERATIONS.md                  # runbook: workflow spec, manual operation, failure modes
```

## `packages.toml` spec

This file is the registry of source repositories whose release assets should be ingested. Adding a package = appending a block and committing. The next hourly cron ingests it.

### Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `[[packages]]` | table | yes | Marks the start of a package entry. One per source repo. |
| `repo` | string | yes | GitHub repository in `owner/name` form, e.g. `OpenGamingCollective/asusctl`. |
| `asset_glob` | string | yes | Glob pattern matching the release assets to fetch. Almost always `*.pkg.tar.zst`. |

### Example

```toml
# List of source repos whose GitHub release assets should be ingested into ogc.db.tar.gz.
# To add a package: append a [[packages]] block and commit. The hourly cron will pick it up.

[[packages]]
repo = "OpenGamingCollective/asusctl"
asset_glob = "*.pkg.tar.zst"

[[packages]]
repo = "OpenGamingCollective/supergfxctl"
asset_glob = "*.pkg.tar.zst"
```

## The collection workflow

A single workflow, `.github/workflows/collect.yml`, runs hourly on a cron schedule (with a manual `workflow_dispatch` fallback). It reads `packages.toml`, fetches the existing `ogc.db.tar.gz` from S3, polls each listed source repo's latest GitHub release, downloads any `*.pkg.tar.zst` assets not already present in the database, GPG-signs them, merges them into the database via `repo-add`, and uploads the new files + updated database back to S3. The workflow is idempotent and stateless — the database itself is the source of truth for what has already been ingested.

For the full behavioral spec — triggers, concurrency, step-by-step behavior, the dedup/merge pseudocode, what gets signed, when the upload is skipped — see [`OPERATIONS.md`](./OPERATIONS.md).

## Secrets & variables

All configuration is stored as repository secrets in the central repo.

| Name | Sensitive | Purpose |
|---|---|---|
| `PGP_SIGNING_KEY` | yes | PGP private key used to sign every package and the repo database. Lives only in this repo. |
| `BUCKET_ACCESS_KEY` | yes | S3 access key for writing to the bucket. |
| `BUCKET_SECRET_KEY` | yes | S3 secret key for writing to the bucket. |
| `BUCKET_ENDPOINT` | yes | S3-compatible endpoint URL (e.g. `https://s3.example.com`). |
| `S3_BUCKET` | no (but not secret-free) | Bucket name to upload to. |
| `BUCKET_PUBLIC_URL` | no | Public URL where the bucket is served. Used to fetch the existing DB at the start of each run. Not sensitive — it's the URL users put in their pacman.conf. |
| `GITHUB_TOKEN` | auto | Auto-provided by GitHub Actions. Used by `gh release view` / `gh release download` to read public source repos' release assets. No extra secret needed for public source repos; if a source repo is private, use a PAT with `contents: read` on that repo instead. |

## Signing model

- One PGP key pair is generated for this repo. The private key is stored as the `PGP_SIGNING_KEY` secret; it never leaves this repo.
- Every ingested `*.pkg.tar.zst` is signed with `gpg --detach-sign`, producing a `<filename>.sig` alongside it.
- `ogc.db.tar.gz` is signed via `repo-add --sign`, producing `ogc.db.tar.gz.sig`.
- Users add the public key to their pacman keyring once and trust it for all `ogc` packages, regardless of which source repo contributed them.
- The public key is published as `ogc.asc` in this repo (and optionally to a keyserver) so users can retrieve it. See [PGP key](#pgp-key) above.

## History & retention

- **No pruning.** When a new version of a package is ingested, `repo-add` (without `--remove`) updates the database entry for that package name to point at the new file, but the **old** `.pkg.tar.zst` and `.sig` files are **not deleted** from S3.
- The database always points at the latest version of each package.
- Old versions remain accessible via their direct S3 URL, enabling manual rollback or pinning.
- The bucket grows over time. This is accepted. If pruning becomes necessary later, it can be added as a separate step without changing the ingestion model.

## Non-goals

Things this repository deliberately does **not** do, by design:

- **Build packages.** Source repos build and attach assets. This repo never runs `makepkg`.
- **Edit source repositories.** No commits, dispatches, or webhooks sent to source repos.
- **Prune old versions.** History is retained (see [History & retention](#history--retention)).
- **Host non-Arch artifacts.** Only `*.pkg.tar.zst` and `ogc.db.tar.gz` are managed.
- **Real-time ingestion.** Hourly polling only. See [Future upgrade path](#future-upgrade-path).
- **Cross-repo event reaction.** GitHub Actions cannot natively react to events in other repos; this repo polls instead.
- **Multi-architecture support.** The current scope is `x86_64` only (whatever architectures the source repos attach). If `aarch64` or others are added later, the dedup logic must account for the `-<arch>` suffix in filenames to avoid cross-arch collisions.

## Future upgrade path

If hourly lag becomes unacceptable, replace the cron trigger with a **webhook bridge**: a small HTTP receiver (e.g. a Cloudflare Worker) that receives "Release published" webhooks from source repos and forwards them to this repo as `repository_dispatch` events. The workflow would gain `on: repository_dispatch` alongside the existing `schedule` and `workflow_dispatch` triggers, with no other changes — the same collect & merge logic runs either way. The one PAT the bridge needs lives only in the bridge, not in any source repo. This is a pure automation upgrade; the ingestion, signing, and publishing model is unchanged.

## License

_(To be set when the repository is created. Recommended: a permissive license consistent with the rest of the org's repos.)_

## References

- GitHub Actions `schedule` event: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule
- GitHub Actions `workflow_dispatch` event: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch
- GitHub Actions `repository_dispatch` event (for the future upgrade path): https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#repository_dispatch
- GitHub Actions billing (public repos = free standard runners): https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions
- `repo-add` man page: https://man.archlinux.org/man/repo-add.1
- `gh release view` / `gh release download` CLI reference: https://cli.github.com/manual/gh_release
