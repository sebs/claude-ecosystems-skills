# claude-ecosystems-skills

A set of [Claude Code](https://code.claude.com/docs/en/skills) **Agent Skills** for
dependency, security, and supply-chain analysis, all powered by the
[ecosyste.ms](https://ecosyste.ms) open dataset via the
[`ecosyste_ms_cli`](https://github.com/ecosyste-ms/ecosyste_ms_cli).

Each skill teaches Claude how to drive the `ecosystems` CLI to answer a specific,
real-world question about your dependencies — "are any of these vulnerable?", "is this
library safe to adopt?", "who maintains the packages we lean on?" — and to report the
answer with evidence rather than guesswork.

## Skills

| Skill | What it does | Ask it… |
|---|---|---|
| **dep-audit** | Audits a project's dependencies for known advisories and reports which to upgrade. | "audit my dependencies", "any CVEs in this lockfile?" |
| **dep-vet** | Due-diligence scorecard for a single package before you adopt it. | "should we use X?", "is this library healthy?" |
| **vuln-exposure** | Triages the blast radius of an advisory — who depends on it and how exposed you are. | "a CVE just dropped for X, how exposed are we?" |
| **upgrade-impact** | Shows what a version bump pulls into the tree and what new risk it introduces. | "what happens if I bump X to Y?", "is this upgrade safe?" |
| **supply-chain-risk** | Finds critical packages with a single, possibly unfunded maintainer (the "xz problem"). | "what's our bus-factor / sole-maintainer risk?" |
| **license-compliance** | Generates an SBOM and checks licenses against a policy. | "generate an SBOM", "any GPL in here?" |
| **oss-funding-report** | Identifies critical-but-underfunded deps worth sponsoring. | "which deps should we fund?" |
| **pkg-peek** | Inspects a package's files, README, and changelog without installing it. | "what's in this package?", "what changed between versions?" |

The skills cross-reference one another (e.g. `supply-chain-risk` hands funding detail to
`oss-funding-report`), so Claude can chain them on larger investigations.

## Requirements

- **[Claude Code](https://code.claude.com/docs/en/overview)** (or any harness that loads
  Agent Skills).
- **The `ecosystems` CLI** on your `PATH`:
  ```bash
  pip install ecosyste_ms_cli
  ecosystems --version
  ```
  See the [CLI repo](https://github.com/ecosyste-ms/ecosyste_ms_cli) for details. No API
  key is required — ecosyste.ms is a free, open service.

## Installation

Clone into your **project** skills directory (available in that project only):

```bash
git clone https://github.com/sebs/claude-ecosystems-skills tmp-skills
mkdir -p .claude/skills
cp -R tmp-skills/.claude/skills/* .claude/skills/
rm -rf tmp-skills
```

…or into your **personal** skills directory (available across all your projects):

```bash
git clone https://github.com/sebs/claude-ecosystems-skills tmp-skills
mkdir -p ~/.claude/skills
cp -R tmp-skills/.claude/skills/* ~/.claude/skills/
rm -rf tmp-skills
```

Each skill lives in its own directory with a `SKILL.md`, e.g.
`.claude/skills/dep-audit/SKILL.md`. Restart Claude Code (or start a new session) and the
skills are picked up automatically.

## Usage

You don't normally invoke these by name — Claude auto-selects the right skill from your
request. Just ask in natural language:

> Audit the dependencies in this repo for known vulnerabilities.

> Should we adopt `fast-xml-parser`? Is it well maintained?

> A CVE just dropped for `lodash` — how exposed are we?

You can also invoke a skill explicitly with its slash command, e.g. `/dep-audit`.

### Polite-pool access (recommended for bulk scans)

ecosyste.ms rate-limits anonymous bulk traffic. For audits that loop over many packages,
set a contact email once so every call joins the "polite pool":

```bash
export ECOSYSTEMS_MAILTO="you@example.com"
```

The skills pass this through automatically; if it's unset the flag is a harmless no-op,
but unthrottled bulk loops can return intermittent empty responses that *look* like
"no findings" when they aren't.

## How it works

Every skill is a single `SKILL.md` — a short, model-facing playbook describing which
`ecosystems` subcommands to call, in what order, and how to interpret the JSON. The skills
encode the non-obvious parts that are easy to get wrong, for example:

- audit **installed** (locked) versions, not declared ranges like `^1.2.0`;
- `lookup_advisories_by_purl` returns **every** advisory for a package regardless of the
  pinned version — deciding which actually apply to the installed version is the real work;
- a version *equal to* `first_patched_version` is **safe**;
- a license of `"Other"` / `NOASSERTION` is often a permissive license shipped as full
  text, not a true unknown — check before flagging.

## Contributing

Issues and PRs welcome. When adding a skill, keep `SKILL.md` focused, give it a
`description` with concrete trigger phrases, and follow the
[official skill format](https://code.claude.com/docs/en/skills).

## License

[MIT](LICENSE) © Sebastian Schürmann
