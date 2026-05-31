---
name: oss-funding-report
description: >
  Identify which of a project's dependencies are critical-but-underfunded and
  worth sponsoring, using the ecosyste.ms CLI (sponsors, opencollective, and
  critical-package data). Trigger when the user asks "which deps should we fund/
  sponsor?", "where should our OSS budget go?", "are the projects we rely on
  sustainable?", or wants a funding/sustainability report for their stack.
version: 1.0.0
userInvocable: true
---

# OSS Funding & Sustainability Report

Surface the dependencies that the project leans on most heavily but that are least
sustainably funded — the best targets for sponsorship.

## Tooling

`ecosystems` CLI with `--format json` and `--mailto "$ECOSYSTEMS_MAILTO"` for
polite-pool access (email configurable via the `ECOSYSTEMS_MAILTO` env var; a harmless
no-op if unset). Use `--timeout 90` for slow lookups.

## Step 1 — Dependency set, weighted by importance

Read the project's manifest/lockfile (or parse a hosted manifest with
`ecosystems parser create_job "<URL>" --polling-interval 2`). Rank deps by how central
they are — direct deps and high dependent-counts matter most. Cross-reference the
critical lists to focus effort:

```bash
ecosystems packages get_critical_packages --format json
ecosystems packages get_critical_sole_maintainers --format json
```

## Step 2 — Check each project's funding status

For the important deps, look for existing funding channels:

```bash
# Open Collective presence for a project
ecosystems opencollective lookup_project --format json   # see `--help` for arg shape

# GitHub Sponsors data
ecosystems sponsors list_accounts --format json
ecosystems sponsors get_account <login> --format json
```

Also useful: repo owners that already have sponsor profiles —
`ecosystems repos get_host_owner_sponsors_logins ...`.

> The exact positional args for some `sponsors`/`opencollective` subcommands aren't
> always obvious; if a call errors, inspect `ecosystems_cli/apis/sponsors.openapi.yaml`
> / `opencollective.openapi.yaml`, or run the subcommand to see the usage error.

## Step 3 — Score and prioritise

Combine **reliance** (direct? dependent count? on critical list?) with **funding gap**
(no sponsors / no Open Collective / sole maintainer → larger gap):

| Project | Why it matters | Maintainers | Funded today? | Funding channel | Priority |
|---|---|---|---|---|---|

## Report

Lead with a short prioritised list: "Fund these N first, because …", with a direct
link to each project's funding page where one exists. This mirrors ecosyste.ms' own
funding mission — connect heavy reliance to concrete, fundable projects. Related:
[[supply-chain-risk]] (the un-funded sole maintainers are also your biggest risks).
