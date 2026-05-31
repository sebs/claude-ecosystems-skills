---
name: pkg-peek
description: >
  Inspect a package's contents, README, and changelog without installing it,
  using the ecosyste.ms CLI archives API. Trigger when the user asks "what's in
  this package?", "show me the README/changelog for X", "what changed between
  versions?", "is this package what it claims to be?", or wants to eyeball a
  dependency's source before pulling it in.
---

# Package Peek (inspect without installing)

Look inside a published package's archive — files, README, changelog — to sanity-check
it before adoption or to understand what changed.

## Tooling

`ecosystems` CLI with `--format json` and `--mailto "$ECOSYSTEMS_MAILTO"` for
polite-pool access (email configurable via the `ECOSYSTEMS_MAILTO` env var; a harmless
no-op if unset). The `archives` commands operate on a **`--url`** pointing at the
package tarball/archive.

## Step 1 — Find the archive (download) URL

Get package metadata and pull the registry download URL for the version you care about:

```bash
ecosystems packages get_registry_package --purl "pkg:npm/lodash" --format json
ecosystems packages get_registry_package_version --purl "pkg:npm/lodash@4.17.21" --format json
```

Use the `download_url` (or equivalent tarball link) from that output as `<ARCHIVE_URL>`.

## Step 2 — Inspect the archive

```bash
# List files (spot suspicious/unexpected content, install scripts, binaries)
ecosystems archives list --url "<ARCHIVE_URL>" --format json

# README
ecosystems archives readme --url "<ARCHIVE_URL>" --format json

# Changelog (what changed)
ecosystems archives changelog --url "<ARCHIVE_URL>" --format json

# A specific file's contents (requires --path)
ecosystems archives contents --url "<ARCHIVE_URL>" --path "package.json" --format json

# Repopack: a single bundled view of the repo, handy for review/LLM context
ecosystems archives repopack --url "<ARCHIVE_URL>" --format json
```

## Step 3 — Diff two versions (optional)

To see what changed between releases, fetch both versions' archive URLs (step 1) and:

```bash
ecosystems diff create_job "<ARCHIVE_URL_OLD>" "<ARCHIVE_URL_NEW>" --polling-interval 2 --format json
```

## Report

Summarise what's actually in the package: entry points, notable files, anything
surprising (postinstall scripts, prebuilt binaries, telemetry), and a short
README/changelog digest. If reviewing for safety, flag anything that doesn't match the
package's stated purpose. Related: [[dep-vet]] for the full adoption scorecard.
