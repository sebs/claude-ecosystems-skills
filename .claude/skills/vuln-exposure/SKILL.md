---
name: vuln-exposure
description: >
  Triage the blast radius of a security advisory using the ecosyste.ms CLI —
  given a CVE/GHSA or a vulnerable package, find what depends on it and how
  exposed a project or the wider ecosystem is. Trigger when the user says "a CVE
  just dropped for X", "how exposed are we to <advisory>", "what depends on this
  vulnerable package?", "blast radius of GHSA-…", or incident-response triage.
version: 1.1.0
userInvocable: true
---

# Advisory Blast-Radius Triage

Start from a single advisory or vulnerable package and work outward to "who is affected
and what's the fix path."

## Tooling

`ecosystems` CLI, `--format json`, and `--mailto "$ECOSYSTEMS_MAILTO"` for polite-pool
access (email configurable via the `ECOSYSTEMS_MAILTO` env var; a harmless no-op if
unset). The dependent-packages endpoint is slow for popular packages — use
`--timeout 90` (or higher).

## Step 1 — Pin down the advisory

- If given an advisory UUID:
  ```bash
  ecosystems advisories get_advisory <UUID> --format json
  ```
- If given a package (optionally a version), find its advisories:
  ```bash
  ecosystems advisories lookup_advisories_by_purl --purl "pkg:npm/lodash@4.17.20" --format json
  ```
- Cross-check Dependabot: `ecosystems dependabot get_advisories --purl "pkg:npm/lodash" --format json`

Record the affected ecosystem, package, vulnerable version range, and fixed version.

> The version pinned in the lookup PURL does **not** filter results —
> `lookup_advisories_by_purl` returns *all* advisories for the package. Pick out the
> specific advisory you're triaging, then read its `packages[].versions[]` for the
> `vulnerable_version_range` and `first_patched_version`.

## Step 2 — Map downstream dependents

How many / which packages pull in the vulnerable package (transitive exposure):

```bash
ecosystems packages get_registry_package_dependent_packages --purl "pkg:npm/left-pad" --timeout 90 --format json
```

And which repositories use it across the ecosystem:

```bash
ecosystems repos usage_package --purl "pkg:npm/lodash" --format json
```

## Step 3 — Check the user's own exposure

If a project/manifest is in context, determine whether *they* are affected:
read their lockfile, find the installed version of the package (and of any dependent
from step 2), and compare against the vulnerable range from step 1. Direct vs.
transitive matters — report which it is.

> Use the **installed** version from a lockfile / `pip freeze`, not a manifest range
> spec (`^`, `~=`, `>=`) — the spec isn't a pinned version. To decide membership, check
> the advisory's `packages[].affected_versions` / `unaffected_versions` lists when
> present (exact, no range parsing); otherwise compare against `vulnerable_version_range`
> in the ecosystem's grammar (SemVer / PEP 440). An installed version *equal to*
> `first_patched_version` is **safe**. These version lists only appear on records with
> package statistics, so fall back to range comparison when they're absent.

## Step 4 — Report

- **Verdict:** affected / not affected (and why), with the installed vs. fixed version.
- **Ecosystem blast radius:** count of dependent packages/repos (from step 2).
- **Remediation:** exact upgrade target, plus whether it's a direct bump or requires a
  transitive override.

Be precise about version comparisons — an off-by-one on the fixed version is the whole
ballgame here. If the range is ambiguous, say so rather than guessing.
