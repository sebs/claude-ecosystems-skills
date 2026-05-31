---
name: supply-chain-risk
description: >
  Assess supply-chain / maintainer-concentration risk in a dependency tree using
  the ecosyste.ms CLI — the "xz-utils problem": critical packages with a single,
  possibly unfunded maintainer. Trigger when the user asks about "supply chain
  risk", "bus factor", "sole maintainer risk", "what's our xz exposure?", or
  wants to know which dependencies are fragile single points of failure.
---

# Supply-Chain & Maintainer-Risk Audit

Find dependencies that are widely depended on yet maintained by one person and/or
underfunded — the highest-leverage supply-chain risks.

## Tooling

`ecosystems` CLI, `--format json`, and `--mailto "$ECOSYSTEMS_MAILTO"` for polite-pool
access (email configurable via the `ECOSYSTEMS_MAILTO` env var; a harmless no-op if
unset). Some calls are slow; use `--timeout 90`.

## Step 1 — Establish the dependency set

- For a local project: read the manifest/lockfile and extract `(ecosystem, name)` pairs.
- For a published repo or hosted manifest URL: parse it
  (`ecosystems parser create_job "<URL>" --polling-interval 2 --format json`).

## Step 2 — Intersect with ecosyste.ms "critical" data

ecosyste.ms maintains curated lists of critical packages and sole-maintainer risk —
unique to this platform. Pull them and intersect with your dependency set:

```bash
# Critical packages overall
ecosystems packages get_critical_packages --format json

# Critical packages that depend on a SINGLE maintainer (the key risk signal)
ecosystems packages get_critical_sole_maintainers --format json

# Maintainers who sit behind many critical packages (concentration risk)
ecosystems packages get_critical_maintainers --format json
```

A dependency of yours appearing in `get_critical_sole_maintainers` is a red flag.

## Step 3 — Per-package maintainer check

For each of your important deps, confirm how many maintainers actually publish it:

```bash
ecosystems packages get_registry_maintainers --purl "pkg:npm/<name>" --format json
```

One maintainer + high dependent count = single point of failure.

## Step 4 — Cross-reference funding (is the risk being mitigated?)

A sole maintainer who is funded is lower-risk than one who isn't. Check funding via
the `sponsors` / `opencollective` groups (delegate the detail to [[oss-funding-report]]).

## Report — risk register

Rank by `(criticality × low-bus-factor × low-funding)`:

| Package | Dependents | # Maintainers | On critical list? | Funded? | Risk |
|---|---|---|---|---|---|

Lead with the top 3–5 highest-risk dependencies and a concrete mitigation per item
(vendor it, fund it, find an alternative, pin + monitor). Be explicit about coverage:
list any deps the API couldn't classify rather than implying full coverage.
