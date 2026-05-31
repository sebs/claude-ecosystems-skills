---
name: license-compliance
description: >
  Generate an SBOM and check dependency licenses against a policy using the
  ecosyste.ms CLI. Trigger when the user asks to "generate an SBOM", "check
  licenses", "any GPL/copyleft in here?", "license compliance report",
  "is this safe to ship in a proprietary product?", or needs a license
  inventory for legal/compliance. Flags policy violations and unknown licenses.
---

# SBOM & License-Compliance Check

Build a Software Bill of Materials, detect licenses, and flag anything that violates
the project's license policy.

## Tooling

`ecosystems` CLI with `--format json` and `--mailto "$ECOSYSTEMS_MAILTO"` for
polite-pool access (email configurable via the `ECOSYSTEMS_MAILTO` env var; a harmless
no-op if unset). The `sbom` and `licenses` jobs are asynchronous — pass `--polling-interval 2` to block until done,
or capture the job id and fetch later with `<group> get_job <ID>`.

**Resolving licenses per package (step 3b) is a bulk loop** — `export
ECOSYSTEMS_MAILTO=you@example.com` once for polite-pool access, serialize the loop or add
a short delay, retry a transient empty response once, and `jq` out just the license
fields (the full package record is ~100 KB; pulling it in a tight loop rate-limits).

> Both `sbom create_job` and `licenses create_job` take a **URL** to a file or a
> `.zip`/`.tar` archive — not a local path. For a local project, point at the hosted
> repo archive / raw manifest URL, or read manifests locally and look up licenses
> per package (step 3b).

## Step 1 — Confirm the policy

Ask the user (or infer from context) what's allowed. Typical proprietary policy:
permissive OK (MIT, BSD, Apache-2.0, ISC), copyleft flagged (GPL, AGPL, LGPL, MPL),
unknown/unlicensed flagged. Record the policy you're applying so the report is auditable.

## Step 2 — Generate the SBOM

```bash
ecosystems sbom create_job "<REPO_ARCHIVE_OR_MANIFEST_URL>" --polling-interval 2 --format json
```

(For a GitHub repo already indexed, `ecosystems repos get_host_repository_sbom ...`
can return an SBOM directly.)

## Step 3 — Detect licenses

```bash
ecosystems licenses create_job "<REPO_ARCHIVE_OR_MANIFEST_URL>" --polling-interval 2 --format json
```

**3b (local fallback):** if you only have local manifests, read them, then resolve
each dependency's declared license via:
```bash
ecosystems packages get_registry_package --purl "pkg:npm/<name>" --format json
```
Read the license from **`normalized_licenses`** (an SPDX-id array — prefer this); fall
back to the raw `licenses` string only when the array is empty. A value of `"Other"` or
`NOASSERTION` means ecosyste.ms couldn't map the license to an SPDX id — **don't report
it as unknown without checking** `metadata.classifiers` (e.g. `"License :: OSI Approved
:: BSD License"`) or the raw `licenses` text first. It's often a permissive license
shipped as full text rather than an identifier, and flagging it as unknown is a false
positive.

> Enumerate the **full resolved dependency tree** (a lockfile, `pip freeze`, etc.), not
> just top-level manifest entries — transitive components carry licenses too and are the
> ones most often missed.

## Step 4 — Evaluate and report

Classify every component against the policy:

| Component | Version | License | Policy | Note |
|---|---|---|---|---|
| | | | ✅ allowed / ⚠️ review / ❌ violation / ❓ unknown | |

Summarise: total components, count per license, and a **violations list** that legal
can act on. Call out dual-licensed and "unknown/NOASSERTION" components explicitly —
unknowns are a finding, not a pass. Offer to write the SBOM JSON to a file if useful.
