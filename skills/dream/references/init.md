# /dream init — one-time setup

Init exists so that the first real dream starts from consent and a baseline: the user has agreed to how dreams behave, and there is a recorded "dream-zero" state to diff against. Run the five steps in order.

## 1. Discover

Build a picture of the project before asking the user anything, so the interview is informed rather than generic:

- Scan for manifests (`go.mod`, `package.json`, `pyproject.toml`, `Gemfile`, `Cargo.toml`, …) and detect the ecosystems present.
- For each detected ecosystem, check whether a bundled adapter exists in `${CLAUDE_PLUGIN_ROOT}/adapters/` (v1 ships `go.yml`).
- Map the **import surface**: which external packages/modules the project imports, and which symbols from each. For Go: parse import blocks across `*.go` files, then grep for usage of each imported package's identifiers. This powers changelog filtering later — a dream only cares about upstream changes to symbols the project actually uses.
- Note whether the project is a git repository and whether `gh` is authenticated with a remote (`gh repo view`). This determines whether triage can open real issues/PRs (github mode) or must fall back to local artifacts (local mode — see run.md).

## 2. Interview

Ask the user before writing any config. Use the structured question tool if available; otherwise ask in plain conversation. Cover exactly these four points, offering the defaults so the user can accept them wholesale:

1. **Depth cap** for research recursion (default **2**). Explain briefly: depth 1 = mine changelogs for direct findings; depth 2 = findings discovered during mining (e.g. "new API replaces our code") get one round of their own research.
2. **Merge style** (default **umbrella**):
   - `single` — one giant PR containing all accepted changes.
   - `per-finding` — one PR per finding, no umbrella issue linkage required.
   - `umbrella` — one umbrella issue + one PR per finding, all linked.
3. **Default scope** (default **minor**): stay within current majors unless `/dream major` is invoked.
4. **Adapter confirmation**: list the detected ecosystems and which have adapters. Confirm the set to enable. Ecosystems without an adapter run in research-only mode (no deterministic probes, findings capped at medium confidence) — say so here, not later.

Use the defaults silently for anything the user skips or has no opinion on.

## 3. Generate

Create the project state:

- `.claude/dream/config.yml` — from the interview answers plus detection. Start from `${CLAUDE_PLUGIN_ROOT}/templates/config.yml` and fill in real values:

```yaml
merge_style: umbrella   # single | per-finding | umbrella
depth_cap: 2
scope: minor            # default; /dream major overrides per run
adapters: [go]          # confirmed at init
```

- `.claude/dream/adapters/<lang>.yml` — copy each confirmed adapter from `${CLAUDE_PLUGIN_ROOT}/adapters/`. Copying (rather than referencing) lets the user tune probe commands per project.
- `.claude/dream/findings/README.md` — copy from `${CLAUDE_PLUGIN_ROOT}/templates/findings-README.md`. This README **is** the ledger schema; runs and verifiers consult it.
- `.claude/dream/imports.yml` — the import surface mapped in step 1, as `package → [symbols used]`.

## 4. Baseline (dream-zero)

Record the current state so the first real run has something to diff against. Write `.claude/dream/baseline.yml`:

```yaml
dream_id: dream-zero
date: <today, YYYY-MM-DD>
base_sha: <git rev-parse HEAD, or null if not a git repo>
language:
  <lang>: <pinned/declared version, e.g. the go directive>
dependencies:
  <module>: <pinned version>   # one entry per direct dependency
```

Also create an empty `.claude/dream/runs.log`.

## 5. Report

Print a short setup report so wrong assumptions get corrected *before* the first run, when it's cheap:

- Ecosystems detected, and which adapters were enabled.
- Ecosystems detected that **lack** adapters → will run research-only (probes skipped, confidence capped at medium).
- The chosen config (merge style, depth, scope).
- github mode vs local mode (from step 1's `gh` check) and what that means for triage output.
- Count of direct dependencies baselined and the base SHA.

Do not run any probes or research during init. Init sets the table; `/dream` eats.
