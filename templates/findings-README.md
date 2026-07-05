# The findings ledger

One markdown file per finding. This directory is dream's long-term memory: it is
how a dream knows what was already proposed, what a human accepted, and — just as
important — what was rejected and *why*, so the same idea is never re-litigated.

The ledger is language-agnostic; nothing in the schema names an ecosystem.

## File naming

`<id>-<short-slug>.md`, e.g. `014-tea-withfps-replaces-ticker.md`. IDs are
zero-padded and sequential across the ledger's lifetime, never per-run.

## Front matter schema

```yaml
id: "014"
dream_id: "dream-2026-07-05.1"
type: staleness | opportunity | deprecation | vulnerability | idiom | practice
claim: "tea.WithFPS (v2.3) replaces manual ticker in render loop"
affected_files: ["internal/render/loop.go:41-52"]
loc_delta: -10            # opportunity findings only
replaces: ["internal/render/loop.go:41-52"]   # opportunity findings only
release_evidence: ["https://github.com/.../releases/tag/v2.3.0"]
evidence_tier: 1          # 1 = first-party, 2 = third-party
confidence: high | medium | low
status: proposed | verified | accepted | rejected | implemented
pr: null                  # filled at triage (PR number, or branch name in local mode)
issue: null               # filled at triage (issue number, or report anchor in local mode)
created: 2026-07-05
rejected_reason: null     # REQUIRED when status is rejected — future dreams read this to skip the claim
```

## Body

Free-form rationale followed by the verifier's notes (citation checks,
reproduction result, proof-ladder outcome). Keep the claim falsifiable and the
evidence first-party.

## Statuses

- `proposed` — researched, medium confidence; will be re-researched next dream.
- `verified` — high confidence, survived clean-context verification.
- `accepted` — a human said yes (merged the PR / approved the issue).
- `rejected` — killed, with `rejected_reason` filled. Dreams never re-propose a
  claim that substantively matches a rejected finding.
- `implemented` — the change landed.

## Promotion rule

Only `accepted` findings that encode a **durable convention** may be summarized
into `CLAUDE.md`. Raw research never lands there.
