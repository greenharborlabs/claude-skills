# Confidence Gate

Load this after architect findings return.

## Parse Findings

Classify architect output:

- `CRITICAL`: must be resolved or explicitly listed in Open Questions.
- `IMPORTANT`: incorporate silently when sound.
- `MINOR`: incorporate if easy; otherwise defer.
- Missing work units: add to the suggested wave unless the reasoning is wrong.
- Wave changes: apply when justified.

Record all changes in `## Architect Review Findings`.

## Confidence Dimensions

Default dimensions:

- Architecture: boundaries, coupling, dependency graph, blast radius.
- Error Handling: named failures and criteria coverage.
- Test Strategy: test type coverage and specificity.
- Data Flow: happy path and error-path tracing.
- Security: auth, validation, injection/trust boundaries.
- Performance: hot paths, query/cache behavior, concurrency, async work, and
  cheap verification command.

Eng mode scores Architecture, Error Handling, and Test Strategy by default. Also
score Security, Performance, Data Flow, or migration/public API impact whenever
the plan's `## Risk Flags` mark the matching risk as `yes`.

## Gate Flow

1. Auto-incorporate IMPORTANT findings, straightforward MINOR findings, missing
   work units, and sound wave reordering.
2. Present a compact confidence map to the user only when CRITICAL findings or LOW
   scores need a decision.
3. Ask at most two questions per round. Prioritize Architecture, Error Handling,
   Test Strategy, Data Flow, Security, Performance, unless the plan is clearly
   security-, migration-, public-API-, or performance-heavy.
4. Re-draft only affected sections.
5. Run delta re-review only for structural changes.
6. Gate passes when all dimensions are at least MEDIUM and all CRITICAL findings
   are resolved.
7. After two rounds, write the plan with unresolved LOW/CRITICAL items in
   `## Open Questions` and warn in the summary.

All-HIGH fast path: print a brief confidence map and three concrete verification
actions from the architect output, then proceed without asking.

## Final Output

Write the final plan to `--out`, including the orchestration playbook:

```bash
# Wave 1: [title]
/greenharbor-orchestrate <plan-path> --scope "Wave 1"

# full plan
/greenharbor-orchestrate <plan-path>
```

If `--branch` is not `main`, append `--branch <ref>` to every playbook command.

Print:

```text
Plan written to: <path>
Mode: <ceo|eng>
Waves: N | Work units: M
Architect: <agent-type> - N findings
Secondary architect: <agent-type or none>
Confidence: Architecture=..., Errors=..., Tests=..., Data=..., Security=...
Gate: Passed round N / Passed with unresolved items
```
