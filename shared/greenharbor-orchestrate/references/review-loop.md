# Review Loop

Load this after all selected Coder work completes, before final verification.

## Final Verification

Run the aggregate targeted test set.

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

If the aggregate targeted tests fail:

- Treat as CRITICAL.
- Allow one focused test-fix Coder, then re-run only affected tests. If they
  still fail, report and stop before review, docs, or specialist reviews.

On success, mark `final-verification-gate` complete.

## Review Diff

`git diff` ignores untracked files, so new files would be invisible to reviewers.
Before any review, mark them intent-to-add (no content is staged):

1. List untracked files with `git ls-files --others --exclude-standard`.
2. Run `git add -N -- <paths>` for those that appear in Coder `FILES_CHANGED`.
   Never mark `.orchestrate/` checkpoints or the plan file.
3. Capture `git diff <START_COMMIT> --stat` as `CHANGED_FILES`.

Do not read the full diff into orchestrator context. Reviewers, Architects,
specialists, and fix Coders run `git diff` themselves from `START_COMMIT`
and `CHANGED_FILES`.

## Cumulative Reviewer

Re-read the selected plan sections and re-extract every acceptance criterion
with its task-derived AC-ID. Spawn one Reviewer against the cumulative changes
and all acceptance criteria. No self-review. The Reviewer counts against the
sub-agent limit.

## Acceptance Criteria Cross-Check

After Reviewer returns:

1. Count reviewed criteria against the provided AC-ID list.
2. Missing criteria become FAIL.
3. For each PASS, ensure the evidence file appears in `CHANGED_FILES`. If not,
   downgrade to FAIL.
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

## Current changes
Base commit: <START_COMMIT>
Changed files:
<CHANGED_FILES>
Run `git diff <START_COMMIT> -- <file>` for the files you need.

## Issues to fix
<CRITICAL and WARNING items>

## Failed acceptance criteria
<FAIL/PARTIAL/MISSING from cross-check>

## Files to modify
<changed + issue files>

## Constraints
Address every CRITICAL. Address WARNING unless it violates spec. Do not expand scope.
Do not commit, stage, or stash.
```

If FAIL and `COMPLEX_ISSUES: YES`, stop and report the architectural blocker.
Offer an Architect fix plan (agent-prompts.md) as an option; spawn it only if
the user approves, then hand its output to one fix Coder within the same single
fix cycle. Do not expand into an open-ended review loop.

After the fix Coder completes, refresh intent-to-add marks and `CHANGED_FILES`,
run affected checks, and run one final re-review. Stop on any remaining CRITICAL
issue. Persist WARNING/INFO in the summary but do not block.

## Mark Complete

After the cumulative review and targeted tests pass:

- Mark each reviewed task complete and update the checkpoint.
- Print final progress.

## Abort And Rollback Messaging

On user interrupt:

```text
Orchestration interrupted. All changes since START_COMMIT (<short hash>) are in the working tree.
To review: git diff <START_COMMIT> --stat   (new files are marked intent-to-add)
To keep: continue working from current state
To set aside (recoverable):
  git reset -q -- $(git diff --name-only --diff-filter=A)   # clear intent-to-add marks
  git stash push -u -m "orchestrate <plan-file>"
```

`git stash` refuses intent-to-add entries, so clear them first as shown.

If blocked, include the last reviewer summary and the same review/rollback hints.
