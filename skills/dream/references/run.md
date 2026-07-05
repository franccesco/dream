# The dream run pipeline

Execute the stages in order. Stages 4–5 (research, verification) are where subagents fan out; everything else runs in the main loop. Keep a running tally of what was probed, skipped, found, verified, rejected — the umbrella report needs all of it.

## 0. Preflight

- Read `.claude/dream/config.yml`, `baseline.yml`, `runs.log`, and the findings ledger README.
- **Scope**: `minor` unless this run is `/dream major`, in which case `major`.
- **Dream ID**: `dream-YYYY-MM-DD.N` — today's date plus a run counter. N = 1 + the number of `runs.log` entries with today's date. No semver, ever.
- **Base SHA**: `git rev-parse HEAD`. Every claim, patch, and line reference in this run is relative to this SHA.
- **Mode**: `github` if `gh` is authenticated and the repo has a remote (`gh repo view` succeeds); otherwise `local`. In local mode, everything that would be a GitHub issue becomes a markdown file under `.claude/dream/runs/`, and PRs become local branches with the report noting "branch ready, no remote to push to". The pipeline is otherwise identical — never skip verification or the ledger because there's no remote.

## 1. Umbrella first

If `merge_style: umbrella`: create the umbrella issue **before doing anything else** (github mode: `gh issue create` with label `dream`, title `Dream <dream-id>`; local mode: `.claude/dream/runs/<dream-id>.md`). Open it with a skeleton: dream id, base SHA, scope, "run in progress". The point of creating it first: a crashed run leaves a visible, half-empty report instead of silence.

For `single` and `per-finding` styles there is no umbrella issue; the final report goes to `.claude/dream/runs/<dream-id>.md` and is printed to the user.

Append the run to `runs.log` now: `<dream-id> sha=<base-sha> scope=<scope> mode=<mode> umbrella=<ref-or-none>`.

## 2. Inventory

- Read the repo tree and manifests; detect ecosystems; load adapters from `.claude/dream/adapters/`. A detected ecosystem with **no adapter** runs research-only: skip its deterministic probes and cap all its findings at medium confidence.
- Refresh the import surface into `.claude/dream/imports.yml` (packages → symbols actually used). Stale import maps produce irrelevant findings, so recompute rather than trust init's snapshot.
- Read the **entire ledger**, including rejected findings and their `rejected_reason` — you need these for dedupe in stage 5.
- In github mode, read previous umbrella issues (label `dream`) and the fate of previous dream PRs (merged vs closed-unmerged). Note the signal in the report — a category of proposal the user keeps closing unmerged deserves skepticism at triage. (Log the signal only; automated calibration is deferred.)
- Read open issues labeled `dream` so triage doesn't file duplicates.

## 3. Deterministic probes (zero LLM tokens)

For each adapter, run its commands and capture raw output:

- `probe_outdated` — pinned vs latest versions per dependency.
- `probe_vulns` — security advisories (if the tool is missing, say so in the report and continue; offer the install command).
- Language/runtime gap — compare the version from `lang_version_source` against `lang_latest_source`.

These are **facts, not claims**: a version delta or a CVE hit skips confidence scoring entirely. Do not spawn research agents to establish what a probe already established.

Under `minor` scope, filter each dependency's "latest" down to the newest version within its current major, and the toolchain's to the newest within its current major. A dependency whose only movement is a new major is, under minor scope, **a target with no delta** (log it as "newer major exists; out of scope — run /dream major").

## 4. Delta gate

Build the target list: every direct dependency + the language/runtime + toolchain. For each target, decide: is there a delta? (version movement within scope, an advisory, or — for idiom/practice checks — a language version gap).

- **No delta → logged skip.** Write one line per skipped target into the umbrella/report ("up to date at vX.Y.Z"). No research agent is spawned, no finding is created.
- Delta → the target proceeds to research.

This gate is what keeps dreams cheap on healthy projects. Respect it.

## 5. Research fan-out

Spawn one `dream:researcher` subagent per target with a delta. Run them in parallel; if a dynamic workflow/orchestration tool is available, use it (agent type `dream:researcher`), otherwise parallel direct subagent calls. Each researcher gets a self-contained prompt containing:

- The target: name, pinned version, latest in-scope version, the raw probe output for it.
- The symbols this project imports from it (from `imports.yml`) and the files that import them.
- Scope (`minor`/`major`), depth cap (from config), and evidence-tier rules (the researcher's own instructions carry the full definitions).
- In `major` mode: instruction to prioritize official migration guides.
- A digest of prior ledger findings about this target — claims already `accepted`, `implemented`, or `rejected` (with `rejected_reason`) — so it doesn't re-litigate.

The researcher returns structured findings (or an explicit "no impact" with reason). **Dedupe before verification**: a finding whose claim matches a previously `rejected` ledger entry (same type, same target, same substantive claim) is dropped here and logged in the report as "previously rejected: <reason>". Re-proposing it would burn verification effort re-litigating a decision the user already made.

Opportunity findings that a researcher discovers during depth expansion re-enter this pipeline as new findings — they get the same dedupe and the same verification as anything else.

## 6. Verification (clean context)

Spawn one `dream:verifier` subagent per surviving finding — in parallel. **Clean context is the whole point**: give the verifier only the finding itself (claim, type, evidence URLs, affected files, proposed change if any) plus the adapter's `verify` command and the path to the repo. Never pass the researcher's reasoning, transcript, or enthusiasm. A verifier that reads the researcher's argument inherits the researcher's blind spots.

The verifier grades per its own instructions (confidence rubric + proof ladder) and returns: confidence, verdict (`verified` / `rejected` / `needs-issue`), notes, and — for opportunity findings — a patch it actually compiled and tested, plus any characterization test it had to write.

Deterministic facts (pure version delta, CVE present) skip confidence scoring but still pass through a verifier when they carry a proposed change (the *fix* needs verification even when the *fact* doesn't).

## 7. Ledger update

Write one file per finding into `.claude/dream/findings/` following the schema in `.claude/dream/findings/README.md` (front matter: id, dream_id, type, claim, affected_files, evidence, tier, confidence, status, …). Statuses after this stage:

- `verified` — high confidence, survived verification.
- `proposed` — medium confidence; stays in the ledger for re-research next dream.
- `rejected` — low confidence or refuted; **always** fill `rejected_reason` so future dreams don't re-litigate.

IDs are zero-padded sequential across the ledger's lifetime (`014`, not per-run). Body: free-form rationale + the verifier's notes.

Promotion rule: only findings that a human later marks `accepted` and that encode a durable convention may be summarized into `CLAUDE.md`. Raw research never lands there.

## 8. Triage

The gate is **risk (maintenance vs extensibility), not effort**:

- **PR** — mechanical, verified, behavior-preserving, private surface. Maintenance work a reviewer can check quickly.
- **Issue** — touches public API, requires design decisions, is a breaking change, restructures for future capability, or is huge-but-mechanical work worth scheduling deliberately. Extensibility work.
- **Re-dream** — medium confidence: leave status `proposed`; it gets re-researched next run. No PR, no issue.
- **Reject** — low confidence or verified no-impact: status `rejected` with `rejected_reason`.

Effort is a tiebreaker **within the PR bucket only** (small PRs first); it never promotes an issue-shaped change into a PR.

A finding whose affected path lacks test coverage ships the verifier's characterization test **in the same PR** as the change. If not even a characterization test was possible (e.g. TUI rendering), the finding is capped at medium and the PR downgrades to an issue.

**Before opening any PR**, check that HEAD still equals the run's base SHA. If HEAD moved, re-run the verifier's proof (build + tests) on the current HEAD before opening; if it no longer passes, demote to re-dream.

Mechanics (github mode):

- Branch: `dream/YYYY-MM-DD.N/finding-<id>`. `single` style: one branch `dream/YYYY-MM-DD.N/all` with all accepted changes, exactly one PR.
- PR body links the umbrella issue (if any) and the ledger finding file path. Label: `dream`.
- Fill the finding's `pr:` / `issue:` front-matter fields with the created numbers — provenance must be queryable in both directions.

Local mode: create the branches and commit the changes locally, write would-be issues as sections of the run report, and fill `pr:`/`issue:` with branch names / report anchors instead.

## 9. Umbrella / final report

Update the umbrella issue (or the run report file) with the complete picture:

- Targets probed, and per-target outcome (delta / no delta / out-of-scope major).
- Deltas found and findings produced, grouped by confidence tier.
- Checklist of spawned PRs and issues, linked.
- Skipped and rejected findings **with reasons** — the skips are evidence the dream looked, not noise.
- Base SHA, scope, mode, and anything degraded (missing vuln scanner, research-only ecosystems, HEAD drift).

Finally, print the user a short summary: what was found, what was opened, what needs their review. The dream ends with humans holding every decision.
