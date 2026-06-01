---
name: dep-vet
description: >
  Due-diligence check on a single package before adopting it as a dependency,
  using the ecosyste.ms CLI. Trigger when the user asks "should we add/use X?",
  "is X well maintained / safe / actively developed?", "vet this library",
  "is this package healthy?", or compares candidate libraries. Produces an
  adopt / caution / avoid scorecard with the evidence behind it.
---

# New-Dependency Due Diligence

Evaluate one package across health, security, popularity, maintenance, and funding,
then give a clear recommendation.

## Tooling

Use the `ecosystems` CLI with `--format json` and `--mailto "$ECOSYSTEMS_MAILTO"`
(polite-pool email, configurable via the `ECOSYSTEMS_MAILTO` env var — a harmless no-op
if unset). Identify the package by PURL (e.g. `pkg:npm/express`, `pkg:pypi/requests`).
Raise `--timeout 90` for the dependents call, which can be slow.

## Signals to gather

1. **Core metadata & popularity** — description, downloads, dependent counts, repo link:
   ```bash
   ecosystems packages get_registry_package --purl "pkg:npm/express" --format json
   ```
   Or, when you don't know the registry, `ecosystems packages lookup_package --purl "pkg:pypi/django"`.

2. **Release cadence / liveness** — is it still shipping, or abandoned?
   ```bash
   ecosystems packages get_registry_package_versions --purl "pkg:npm/express" --format json
   ```

3. **Security history**:
   ```bash
   ecosystems advisories lookup_advisories_by_purl --purl "pkg:npm/express" --format json
   ```

4. **Repository health** — stars, last activity, archived flag, license:
   ```bash
   ecosystems repos repositories_lookup --purl "pkg:github/expressjs/express" --format json
   ```
   (derive the GitHub PURL from the repo URL in step 1's output)

5. **Adoption / blast radius** — how widely is it used?
   ```bash
   ecosystems repos usage_package --purl "pkg:npm/express" --format json
   ```

6. **Funding / sustainability** (bus-factor read): the **step-1 `get_registry_package`
   response already carries this** — `maintainers` (an array; count it for bus factor,
   each entry has `login`/`email`/`packages_count`) and `funding_links`. No extra call
   needed. For deeper funding detail, use the `sponsors` / `opencollective` groups.
   See [[oss-funding-report]].
   > Do **not** use `packages get_registry_maintainers` for a per-package read — it
   > lists maintainers for an *entire registry* and takes a positional `REGISTRY_NAME`
   > (it has no `--purl` option, so `--purl` errors out).

## Scorecard

Summarise each signal as 🟢 / 🟡 / 🔴 with a one-line reason:

| Dimension | Rating | Evidence |
|---|---|---|
| Maintenance | | last release, cadence |
| Security | | open/historical advisories |
| Popularity | | downloads, dependents |
| Repo health | | stars, archived?, license |
| Bus factor | | maintainer count, funding |

End with a verdict: **Adopt / Adopt with caution / Avoid**, plus the single most
important caveat. If the package can't be found, say which registry was tried.
