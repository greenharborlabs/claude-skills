# Review Loop

Load this when executing batches after Coder prompts are ready.

## Wave Checklist

Before a batch, print:

```text
Wave N checklist:
- [ ] Task-ID: subject
```

After each task passes review and tests, mark it complete and print the updated
checklist. Before advancing, verify all tasks in the wave are complete. If any
are not complete, execute them before moving on.

## Coder Failure Handling

If a Coder times out, crashes, or omits the expected completion report:

1. Log raw output/error.
2. Retry once with the same prompt.
3. If still invalid, mark blocked and continue only where dependencies allow.

If `SELF_TEST: FAIL`, still spawn the Reviewer and prepend the failure details.

## Mandatory Reviewer

After any Coder implementation or fix, spawn Reviewer before inspecting changed
files, running tests, or declaring success. No self-review.

Exception: a Coder spawned only to fix failing tests after Reviewer PASS does not
need another Reviewer.

The Reviewer still counts against the global `ACTIVE_AGENTS` limit. If three
agents are already active, wait for and close a completed agent before spawning
the Reviewer.

## Acceptance Criteria Cross-Check

After Reviewer returns:

1. Count reviewed criteria against the spec excerpt.
2. Missing criteria become FAIL.
3. For each PASS, ensure the evidence file/line appears in the diff already in
   context. If not, downgrade to FAIL.
4. Override `VERDICT: PASS` to FAIL if any criterion is FAIL/PARTIAL/missing.

Use no additional tool calls for this step.

## Fix Loop

Maintain one cycle counter per work group. Max 3 Coder->Reviewer cycles total.

If FAIL and `COMPLEX_ISSUES: NO`, spawn Coder with:

```text
## Fix task
Fix reviewer issues.

## Original spec excerpt
<same spec>

## Success criteria
<same criteria>

## Current diff
<diff>

## Issues to fix
<CRITICAL and WARNING items>

## Failed acceptance criteria
<FAIL/PARTIAL/MISSING from cross-check>

## Files to modify
<changed + issue files>

## Constraints
Address every CRITICAL. Address WARNING unless it violates spec. Do not expand scope.
```

If FAIL and `COMPLEX_ISSUES: YES`, spawn Architect first, then Coder, then Reviewer.
The cycle counter does not reset after architect escalation.

Architect and fix Coder agents also count against `ACTIVE_AGENTS`; free a slot
before each spawn.

After 3 cycles, stop on remaining CRITICAL issues. Persist WARNING/INFO in the
summary but do not block.

## Targeted Tests

After Reviewer PASS, run only tests created or modified in the cycle using
`TEST_CMD`. If tests fail:

- Treat as CRITICAL.
- Spawn a test-fix Coder.
- Re-run targeted tests.
- Max 2 test-fix cycles.

Do not run the full suite here.

## Mark Complete And Contract Registry

After review and targeted tests pass:

- Mark each task complete immediately.
- Add up to 5 `NEW_INTERFACES` entries per batch to the contract registry:
  `{ file, name, signature }`.
- If a changed file already has registry entries and the Coder did not report
  updated interfaces, remove stale entries.
- Print batch progress.

After each wave, discard raw file contents, diffs, Coder reports, Reviewer output,
and spec excerpts. Carry forward only task status, blocked reasons, `START_COMMIT`,
agent suite, `TEST_CMD`, touched tests, and contract registry.

## Final Verification

After all batches complete, run the aggregate targeted test set only. On success,
mark `final-verification-gate` complete. On failure, report and stop before docs
or specialist reviews.

## Abort And Rollback Messaging

On user interrupt:

```text
Orchestration interrupted. All changes since START_COMMIT (<short hash>) are in the working tree.
To review: git diff <START_COMMIT>..HEAD
To keep: continue working from current state
To undo: git reset <START_COMMIT>
```

If blocked, include the last reviewer summary and the same review/rollback hints.
