# 💤 dream

A Claude Code plugin that periodically **dreams** about your project: it audits your
dependencies, language/runtime version, and coding practices against current
authoritative sources, then proposes evidence-backed, independently verified,
human-reviewed changes.

Inspired by memory-curation "dreaming", extended with active research and code
cross-referencing. The best findings aren't "v1.2 is out" — they're
*"the library added a native API in v2.3 that makes these 10 lines of ours a
one-line call, here's a tested PR that proves it."*

**Nothing auto-merges. Ever.** Dreams open PRs and file issues; you merge.

## Install

```
/plugin marketplace add franccesco/dream
/plugin install dream@dream
```

Or try it locally without installing:

```bash
claude --plugin-dir /path/to/dream
```

## Usage

| Command | What it does |
|---|---|
| `/dream init` | One-time setup: detects your ecosystems, interviews you (depth, merge style, scope, adapters), writes `.claude/dream/config.yml`, and records a dream-zero baseline. Required before any run. |
| `/dream` | Default dream. Version scope comes from your config: `minor` (the default) never crosses a major version of anything; `major` if you gave standing consent at init. |
| `/dream major` | Consent to major-version upgrades and the migrations they require. Research prioritizes official migration guides. |

Run it on a schedule if you like (e.g. a weekly `claude -p "/dream"` cron or a
scheduled Claude Code routine) — the pipeline is idempotent and dedupes against
its own ledger.

## How a dream works

```
Inventory → deterministic probes (zero LLM tokens) → delta gate
  → research fan-out (one agent per changed target)
  → depth expansion (changelog mining filtered to symbols you actually import,
     cross-referenced against your code)
  → clean-context verification (citations checked, changes compiled and tested)
  → findings ledger → triage → PRs / issues + umbrella report
```

Design properties worth knowing:

- **Delta gate.** Targets with no version movement are logged and skipped — no
  research agents, no tokens burned. Dreams are cheap on healthy projects.
- **Evidence tiers by ownership, not format.** Only maintainer/vendor-owned
  sources (official docs, release notes, changelogs, registry metadata, OSV.dev)
  count as evidence. Blogs and Stack Overflow may *locate* evidence, never *be* it.
- **Clean-context verification.** A separate verifier agent grades every finding
  without access to the researcher's reasoning. It re-checks every citation,
  reproduces claims mechanically, and actually compiles and tests proposed
  changes. Code paths without test coverage get a characterization test written
  first — and it ships in the PR.
- **Risk-based triage.** Mechanical, verified, behavior-preserving changes become
  PRs. Anything touching public API, needing design decisions, or breaking
  becomes an issue. Medium-confidence findings wait for the next dream.
- **A ledger with memory.** Every finding lives in `.claude/dream/findings/` with
  full provenance (dream id, PR/issue, evidence, confidence). Rejected findings
  keep their `rejected_reason` so the same idea is never re-proposed.
- **Umbrella-first reporting.** The umbrella issue is created before the run
  starts, so even a crashed dream leaves a visible report.

## What lands in your repo

```
.claude/dream/
├── config.yml        # your consented settings (merge style, depth, scope)
├── adapters/go.yml   # probe/verify commands — tune per project
├── findings/         # the ledger: one file per finding + schema README
├── baseline.yml      # dream-zero: versions + base SHA at init
├── imports.yml       # which upstream symbols you actually use
└── runs.log          # one line per dream
```

## Ecosystems

Adapters ship for **Go**, **Python**, **JavaScript/TypeScript**, and **Ruby**.
At init, the copied adapter is tailored to your project's actual tooling
(uv vs pip vs poetry, pnpm vs npm vs bun, pytest vs unittest, …) and stays
editable in `.claude/dream/adapters/`. The schema is deliberately thin — probe
commands, a verify command, version sources — so new ecosystems are one small
YAML file. PRs welcome: add a file to `adapters/`.

A detected ecosystem without an adapter still works in **research-only mode**:
deterministic probes are skipped and all findings cap at medium confidence.

## Safety model

- Nothing is merged by the plugin under any configuration.
- Major-version upgrades happen only with explicit consent: `scope: major`
  chosen during the init interview (standing), or `/dream major` (one run).
- All researched text (changelogs, release notes, READMEs) is treated as
  untrusted data, never as instructions to the agent.
- Minors go through the same verification ladder as majors — semver is a
  promise, not a guarantee.

## License

MIT
