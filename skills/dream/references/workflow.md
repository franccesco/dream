# Orchestrating research + verification as a dynamic workflow

Dynamic workflows are the intended engine for a dream's fan-out stages. They buy
four things subagent calls alone don't: deterministic control flow (the dedupe
and routing between research and verification is code, not model judgment),
real parallelism with per-item pipelining, schema-validated outputs (a
researcher physically cannot return an unparseable finding), and resumability
(a run that dies mid-verification resumes without re-researching).

Adapt the template below per run. Build the `briefs` array in the main loop
first (stages 0–4 of run.md), then invoke the workflow with it as `args`.
Prompts must be **self-contained** — a workflow agent sees nothing of your
conversation. Include in each brief prompt everything run.md says to pass
(target, versions, probe output, import surface, scope, depth cap, ledger
digest), and in `verifierPreamble` the repo path, base SHA, and the adapter's
verify command.

Notes on the mechanics:

- `agentType: 'dream:researcher'` / `'dream:verifier'` resolve to this plugin's
  bundled agents, so the workflow agents carry the full researcher/verifier
  instructions automatically. The `schema` option overrides their "fenced YAML"
  output convention — they return validated JSON instead, which is what you want.
- `pipeline()` (not two `parallel()` barriers): target A's findings verify
  while target B is still researching. Dedupe-vs-ledger is plain code inside
  the second stage — rejected claims are pruned *before* they cost a verifier.
- The verifier runs with `isolation: worktree` from its agent definition, so
  parallel verifiers applying patches never collide.

```js
export const meta = {
  name: 'dream-research-verify',
  description: 'Dream run: research fan-out per target + practices sweep, clean-context verification per finding',
  phases: [
    { title: 'Research', detail: 'one researcher per delta target + one practices sweep per ecosystem' },
    { title: 'Verify', detail: 'one clean-context verifier per surviving finding' },
  ],
}

// args = {
//   briefs: [{ label: 'x-text-v0.38', prompt: '<self-contained research brief>' }, ...],
//   rejectedClaims: [{ type, target, claim, reason }, ...],   // from the ledger
//   verifierPreamble: '<repo path, base SHA, adapter verify command, output contract>',
// }

const FINDINGS_SCHEMA = {
  type: 'object',
  required: ['findings', 'no_impact'],
  properties: {
    target: { type: 'string' },
    findings: {
      type: 'array',
      items: {
        type: 'object',
        required: ['type', 'claim', 'affected_files', 'evidence', 'evidence_tier', 'confidence', 'rationale'],
        properties: {
          type: { enum: ['staleness', 'opportunity', 'deprecation', 'vulnerability', 'idiom', 'practice'] },
          claim: { type: 'string' },
          affected_files: { type: 'array', items: { type: 'string' } },
          loc_delta: { type: 'number' },
          replaces: { type: 'array', items: { type: 'string' } },
          evidence: { type: 'array', items: { type: 'string' } },
          evidence_tier: { enum: [1, 2] },
          confidence: { enum: ['high', 'medium', 'low'] },
          rationale: { type: 'string' },
        },
      },
    },
    no_impact: { type: 'array', items: { type: 'string' } },
    notes: { type: 'string' },
  },
}

const VERDICT_SCHEMA = {
  type: 'object',
  required: ['verdict', 'confidence', 'evidence_tier', 'reproduction', 'proof_ladder'],
  properties: {
    verdict: { enum: ['verified', 'rejected', 'needs-issue'] },
    confidence: { enum: ['high', 'medium', 'low'] },
    evidence_tier: { enum: [1, 2] },
    citations_check: { type: 'array', items: { type: 'object', properties: {
      url: { type: 'string' }, supports_claim: { type: 'boolean' }, note: { type: 'string' } } } },
    reproduction: { type: 'string' },
    proof_ladder: { type: 'object', properties: {
      compile: { enum: ['pass', 'fail', 'not-applicable'] },
      existing_tests: { enum: ['pass', 'fail', 'no-coverage'] },
      characterization_test: { enum: ['added', 'not-needed', 'impossible'] } } },
    patch: { type: 'string' },
    loc_delta: { type: 'number' },
    rejected_reason: { type: 'string' },
    notes: { type: 'string' },
  },
}

const dedupe = (finding) =>
  !args.rejectedClaims.some(r =>
    r.type === finding.type &&
    finding.claim.toLowerCase().includes(String(r.target).toLowerCase()) &&
    // a previously rejected claim about the same target and type is close
    // enough to re-litigation; the main loop logs what was dropped
    true)

const results = await pipeline(
  args.briefs,
  b => agent(b.prompt, {
    label: `research:${b.label}`, phase: 'Research',
    agentType: 'dream:researcher', schema: FINDINGS_SCHEMA,
  }),
  (research, b) => {
    if (!research) return { brief: b.label, findings: [], skipped: [] }
    const fresh = research.findings.filter(dedupe)
    const dropped = research.findings.filter(f => !dedupe(f))
    return parallel(fresh.map(f => () =>
      agent(
        `${args.verifierPreamble}\n\nFinding to verify:\n${JSON.stringify(f, null, 2)}`,
        { label: `verify:${b.label}`, phase: 'Verify',
          agentType: 'dream:verifier', schema: VERDICT_SCHEMA },
      ).then(v => ({ finding: f, verdict: v }))
    )).then(verified => ({
      brief: b.label,
      no_impact: research.no_impact,
      notes: research.notes,
      skipped: dropped.map(f => f.claim),
      findings: verified.filter(Boolean),
    }))
  },
)

return { results: results.filter(Boolean) }
```

Tune the `dedupe` predicate to the actual shape of your ledger digest — the
template's containment check is deliberately loose; exact claim matching is
better when your rejected claims carry the target name explicitly.

After the workflow returns, the main loop owns everything stateful: ledger
writes, triage, branches/PRs, and the umbrella report (stages 7–9 of run.md).
Keep it that way — the workflow researches and judges; the main loop, and
ultimately the human, decides.
