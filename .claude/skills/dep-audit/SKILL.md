---
name: dep-audit
description: >
  Audit a project's dependencies for known security vulnerabilities using the
  ecosyste.ms CLI. Trigger when the user asks to "audit dependencies", "check
  for vulnerabilities/CVEs", "scan my lockfile/package.json/requirements.txt",
  "are any of my deps vulnerable?", or wants a security report for a manifest or
  a GitHub repo. Produces a severity-ranked report with fixed versions.
---

# Dependency Vulnerability Audit

Audit the dependencies of a project for known advisories and report which to upgrade.

## Tooling

All work goes through the `ecosystems` CLI. Always add `--format json` so output is
parseable, and `--mailto "$ECOSYSTEMS_MAILTO"` for polite-pool access on bulk queries.
The email is configurable via the `ECOSYSTEMS_MAILTO` env var; if it's unset the flag is
a harmless no-op, so never block on it. Some endpoints are slow — raise `--timeout 90`
if a call times out.

**Scanning many deps?** `export ECOSYSTEMS_MAILTO=you@example.com` once so every call
gets polite-pool access — without it, bulk loops rate-limit and you'll see intermittent
empty responses that look like "no advisories" but aren't. Serialize the loop (or add a
short delay between calls), retry a transient empty once, and pipe each response through
`jq` to just the fields you need: the full package record is ~100 KB and pulling it in a
tight loop is what trips the limiter.

## Step 1 — Get the dependency list

Pick the path that matches the input:

- **Local manifest / lockfile** (`package.json`, `requirements.txt`, `Gemfile.lock`,
  `go.mod`, `Cargo.toml`, …): read the file yourself with the Read tool and extract
  each `(ecosystem, name, version)`. Build a PURL per dependency,
  e.g. `pkg:npm/lodash@4.17.20`, `pkg:pypi/django@4.2.0`.

  > **Audit *installed* versions, not declared ranges.** A manifest with range
  > specifiers (`^1.2.0`, `~=2.33`, `>=4`) does not pin a version, so auditing the spec
  > gives wrong results. Prefer a lockfile (`package-lock.json`, `poetry.lock`,
  > `Gemfile.lock`, `Cargo.lock`) or the resolved environment (`pip freeze`,
  > `npm ls --all`). With only a range manifest, resolve it first via
  > `ecosystems resolve create_job "<MANIFEST_URL>" --polling-interval 2 --format json`
  > rather than guessing the version.
  >
  > **Include transitive dependencies.** Most real vulnerabilities live in deps you
  > never declared — audit the full resolved tree, not just top-level manifest entries.

  > `parser`/`sbom` jobs take a **URL**, not a local path — so for local files,
  > read them directly rather than uploading.

- **Hosted manifest or repo archive** (a raw file URL or a `.zip`/`.tar` URL):
  let the API parse it.
  ```bash
  ecosystems parser create_job "<RAW_MANIFEST_OR_ARCHIVE_URL>" --polling-interval 2 --format json
  ```
  `--polling-interval` blocks until the job finishes; otherwise grab the job id and
  later run `ecosystems parser get_job <ID> --format json`.

- **GitHub repo already published**: use its SBOM/manifests instead of parsing files
  (see `repos get_host_repository_sbom` / `get_host_repository_manifests`).

## Step 2 — Look up advisories per dependency

For each dependency PURL:

```bash
ecosystems advisories lookup_advisories_by_purl --purl "pkg:npm/lodash@4.17.20" --format json
```

> **The version in the PURL does *not* filter the results.** This endpoint returns
> *every* advisory ever filed for the package, regardless of the version you pin —
> deciding which actually apply to the installed version is on you, and it's the crux
> of the whole audit (a package can show a CRITICAL advisory yet be perfectly safe at
> the installed version).

**Deciding if the installed version is actually affected.** Each advisory's
`packages[].versions[]` carries `vulnerable_version_range` and `first_patched_version`.
Two ways to decide:
- **Preferred — list membership.** Some records also carry
  `packages[].affected_versions` / `unaffected_versions` (explicit version lists). When
  present, just check whether the installed version is in `affected_versions` — this
  sidesteps fragile range-string parsing. Note these arrays only appear on records that
  include package statistics, so treat them as a supplement, not a guarantee.
- **Fallback — range comparison.** Otherwise compare the installed version against
  `vulnerable_version_range`, respecting the ecosystem's version grammar (SemVer for
  npm/cargo, PEP 440 for PyPI). Boundary cases are where audits go wrong: a version
  *equal to* `first_patched_version` is **safe** (range `< 2.20.0`, installed `2.20.0`
  → not affected). If a range is genuinely ambiguous, say so rather than guess.

Cross-check Dependabot's database too (it sometimes carries advisories the primary
source lacks):

```bash
ecosystems dependabot get_advisories --purl "pkg:npm/lodash" --format json
```

To skip clean packages quickly when scanning many deps, you can also pre-filter by
severity: `ecosystems advisories get_advisories --purl "pkg:npm/x" --severity high`.

## Step 3 — Find the fix version

For every vulnerable dependency, list available versions and pick the lowest release
that is outside the advisory's affected range:

```bash
ecosystems packages get_registry_package_version_numbers --purl "pkg:npm/lodash" --format json
```

## Step 4 — Report

Produce a table sorted by severity (critical → low):

| Package | Installed | Severity | Advisory (CVE/GHSA) | Affected range | Fixed in |
|---|---|---|---|---|---|

Then summarise: counts by severity, and a copy-pasteable list of upgrade actions.
If nothing is found, say so plainly — don't invent findings. Note any packages whose
ecosystem the API didn't recognise so coverage gaps are explicit.
