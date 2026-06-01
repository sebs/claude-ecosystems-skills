---
name: upgrade-impact
description: >
  Assess the impact of upgrading a dependency to a new version using the
  ecosyste.ms CLI — what the bump pulls into the tree, and whether it introduces
  new advisories or license changes. Trigger when the user asks "what happens if
  I bump X to Y?", "is this upgrade safe?", "what does this version pull in?",
  "diff the dependency tree for this version bump", or weighs an upgrade.
---

# Upgrade Impact Analysis

Before bumping a dependency, show what the new version drags in and what new risk (if
any) it introduces.

## Tooling

`ecosystems` CLI with `--format json` and `--mailto "$ECOSYSTEMS_MAILTO"` for
polite-pool access (email configurable via the `ECOSYSTEMS_MAILTO` env var; a harmless
no-op if unset). The `resolve` and `diff` jobs are async — pass `--polling-interval 1`
to block until done.

## Step 1 — Resolve both versions' dependency trees

Resolve the dependency tree for the current and target version (registry is the second
positional arg, e.g. `npm`, `pypi`, `rubygems`):

```bash
ecosystems resolve create_job <package_name> <registry> --version "<CURRENT>" --polling-interval 1 --format json
ecosystems resolve create_job <package_name> <registry> --version "<TARGET>"  --polling-interval 1 --format json
```

`--version` accepts a range (e.g. `"^4.18.0"`). `--before <date>` resolves the tree as
it would have looked at a past date — useful for reproducing an old build.

## Step 2 — Diff the two trees

Compare the resolved sets to find **added / removed / changed** transitive deps. You can
diff the two resolve outputs directly, or, for source archives, use:

```bash
ecosystems diff create_job "<ARCHIVE_URL_CURRENT>" "<ARCHIVE_URL_TARGET>" --polling-interval 1 --format json
```

## Step 3 — Re-check risk on what changed

For every newly added or version-changed dependency in the diff:

```bash
# New advisories pulled in?
ecosystems advisories lookup_advisories_by_purl --purl "pkg:<eco>/<name>@<version>" --format json

# License change on a bumped dep?
ecosystems packages get_registry_package --purl "pkg:<eco>/<name>" --format json
```

## Report

| Change | Package | From → To | New advisory? | License change? |
|---|---|---|---|---|
| added / removed / bumped | | | | |

Lead with the verdict: **safe / review / risky**, justified by the worst item found
(new critical advisory, new copyleft license, large transitive expansion). Quantify the
tree delta (e.g. "+12 transitive deps, 1 new high-severity advisory"). Hand off to
[[dep-audit]] for a full vulnerability pass if the change is large.
