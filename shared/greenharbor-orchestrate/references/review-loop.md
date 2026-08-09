# Review Loop

Load this after all selected Coder work and aggregate targeted checks complete.

## Wave Checklist

Before a batch, print:

```text
Wave N checklist:
- [ ] Task-ID: subject
```

After each Coder reports completion, mark the task as implemented and print the
updated checklist. Do not mark tasks review-complete until the cumulative review
and final checks pass.

## Coder Failure Handling

If a Coder times out, crashes, or omits the expected completion report:

1. Log raw output/error.
2. Retry once with the same prompt.
3. If still invalid, mark blocked and continue only where dependencies allow.

If `SELF_TEST: FAIL`, record the failure, continue only with independent work,
and include it in aggregate verification before review.

## Cumulative Reviewer

After all selected Coder work and aggregate targeted checks pass, spawn one
Reviewer against the full cumulative diff and all acceptance criteria. No
self-review. The Reviewer still counts against the global `ACTIVE_AGENTS` limit.
If three agents are already active, wait for and close a completed agent before
spawning it.

## Acceptance Criteria Cross-Check

After Reviewer returns:

1. Count reviewed criteria against the provided AC-ID list.
2. Missing criteria become FAIL.
3. For each PASS, ensure the evidence file/line appears in the diff already in
   context. If not, downgrade to FAIL.
4. Override `VERDICT: PASS` to FAIL if any criterion is FAIL/PARTIAL/missing.

Use no additional tool calls for this step.

## Focused Fix Cycle

Allow at most one Coder -> Reviewer fix cycle for the cumulative review.

If FAIL and `COMPLEX_ISSUES: NO`, spawn one Coder with:

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

If FAIL and `COMPLEX_ISSUES: YES`, stop and report the architectural blocker;
do not expand into an open-ended review loop.

The fix Coder counts against `ACTIVE_AGENTS`; free a slot before spawning it.

After the fix Coder completes, run affected checks and one final re-review. Stop
on any remaining CRITICAL issue. Persist WARNING/INFO in the summary but do not
block.

## Targeted Test Failure Handling

If the aggregate targeted test set run during final verification fails:

- Treat as CRITICAL.
- Allow one focused test-fix Coder, then re-run only affected tests. If they
  still fail, report and stop.

Do not run the full suite here.

## Mark Complete And Contract Registry

After the cumulative review and targeted tests pass:

- Mark each task complete immediately.
- Add up to 5 `NEW_INTERFACES` entries per batch to the contract registry:
  `{ file, name, signature }`.
- If a changed file already has registry entries and the Coder did not report
  updated interfaces, remove stale entries.
- Print batch progress.

## Wave Completion Summary

After all tasks in a wave are implemented, stop before starting the next wave.
Print a concise implementation-progress summary in this shape:

```text
Wave N complete.

Business / manager summary:
- What completed:
- What it solved:
- Why it mattered:

Developer / technical summary:
- What changed:
- Tests / review: deferred to final cumulative gate
- Technical impact:
```

Keep each bullet short unless detail is needed for a decision, blocker, or
material risk. Do not paste raw diffs or long agent reports. Wait for the user
to confirm before advancing to the next wave.

After each wave, discard raw file contents, diffs, Coder reports, Reviewer output,
and spec excerpts. Carry forward only task status, blocked reasons, `START_COMMIT`,
agent suite, `TEST_CMD`, touched tests, contract registry, and the concise wave
completion summary.

## Final Verification

After all batches complete, run the aggregate targeted test set.

Also run the cheapest relevant package-level confidence command when any of these
are true:

- The plan marks `security`, `performance`, `migration`, `public-api`, or
  `concurrency` risk as `yes`.
- The contract registry contains changed public interfaces.
- A work unit changed shared utilities, build/config files, generated schemas,
  migrations, or package exports.

Examples: compile/typecheck for the touched package, module-level unit tests,
targeted lint for changed files, or migration validation. Do not run a full
suite/build unless the user explicitly approves it.

On success, mark `final-verification-gate` complete. On failure, report and stop
before docs or specialist reviews.

## Abort And Rollback Messaging

On user interrupt:

```text
Orchestration interrupted. All changes since START_COMMIT (<short hash>) are in the working tree.
To review committed changes: git diff <START_COMMIT>..HEAD
To review working tree changes: git diff <START_COMMIT>
To keep: continue working from current state
To undo: git reset <START_COMMIT>
```

If blocked, include the last reviewer summary and the same review/rollback hints.
