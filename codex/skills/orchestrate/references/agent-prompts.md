# Agent Prompts

Load this before constructing Coder, Reviewer, Architect, or docs prompts.

## Batch Exploration

Before each batch:

- Re-read the relevant wave/section from the plan file.
- Max `2 file reads per task + 2 baseline`.
- Max `1 search per task + 1 baseline`.
- Prefer targeted line ranges.
- Use the contract registry instead of re-reading prior batch interfaces.

For each task identify only entry points, required signatures/contracts, and
affected test files.

Group sizing:

- Default one task = one work group.
- Merge tasks sharing the same entry-point file.
- Max 3 tasks per work group.
- Max 3 running sub-agents total.

## Coder Prompt

```text
## Task
<imperative work group title>

## Spec excerpt
<verbatim relevant plan section, including acceptance criteria>

## Files to modify
<file paths + relevant line ranges>

## Interfaces / contracts to conform to
<signatures, types, API shapes from exploration or contract registry>

## Affected tests
<test files>

## Success criteria
<explicit checkable criteria>

## TDD sequencing
If the plan includes Test spec:
1. Write one behavioral test from the spec first and confirm RED.
2. Write minimal code to pass GREEN.
3. Refactor only after green.
Use public interfaces. Mock only external APIs, databases, time, randomness.

## Cross-batch interfaces
<minimal contract registry entries, if relevant>

## Constraints
- Modify only listed files unless necessary and reported.
- Do not change public interfaces unless the spec requires it.
- Keep changes focused.
- If scope is larger than expected, implement what is safe and report the gap.

## Watch for
<stack-specific warnings from below + universal warnings>

## Self-test before reporting
Run: <literal TEST_CMD with affected tests>

## Completion report format
FILES_CHANGED: <comma-separated list>
TESTS_TO_RUN: <comma-separated test files>
SELF_TEST: PASS | FAIL (details if FAIL)
NEW_INTERFACES: <new public types/endpoints/signatures, if any>
NOTES: <unusual context>
```

## Stack Warnings

Java: transaction boundaries around external calls, lazy loops without fetch,
native query string concatenation, missing `@Valid`, authorization on user IDs,
overlong/deeply nested methods.

React: missing effect cleanup, unsafe HTML without sanitization, async effects
without cancellation.

Python: mutable defaults, swallowed broad exceptions, shell injection, missing
await, file handles without context managers.

NanoClaw/OpenClaw: missing risk gates, hardcoded secrets, broker retry/timeout,
state persistence/reconciliation, rate limiting, circuit breakers.

Universal: secrets in source, non-atomic check-then-act, unvalidated untrusted
input in sensitive operations.

## Reviewer Prompt

```text
## Review task
Review implementation against the spec.

## Spec excerpt(s)
<verbatim spec sections>

## Coder self-test result
<PASS/FAIL details>

## New interfaces introduced
<NEW_INTERFACES, if any>

## Diff to review
<git diff for changed files; use -U30, reduce to -U10 or split if too large>

## Success criteria
<same criteria given to coder>

## Acceptance Criteria Verification (MANDATORY)
For EACH criterion, output:
CRITERIA_CHECKLIST:
  [PASS] "criterion text" - evidence: <file>:<line>
  [FAIL] "criterion text" - not implemented: <missing>
  [PARTIAL] "criterion text" - <exists> / <missing>

Rules:
- Copy criteria verbatim.
- Any FAIL/PARTIAL is CRITICAL.
- Evidence must point to a line in the diff.

## What to check
Interfaces, scope, tests, code quality, error handling, behavioral TDD quality.

## Severity guide
CRITICAL - fails criteria, breaks interface, introduces bug
WARNING - suboptimal but not blocking
INFO - suggestion

## Output format
CRITERIA_CHECKLIST:
VERDICT: PASS | FAIL
ISSUES:
COMPLEX_ISSUES: YES|NO
SUMMARY:
```

## Architect Fix Prompt

Use only when Reviewer sets `COMPLEX_ISSUES: YES`.

```text
## Architect task
Produce a targeted fix plan for a complex review failure.

## Original spec excerpt
<spec>

## Current implementation
<diff>

## Critical issues found by reviewer
<verbatim>

## Constraints
Prefer the smallest change within existing architecture unless spec requires more.

## Output format
### Root cause
### Architecture impact
### Fix approach
### Files to change
### Failure modes
| Codepath | Failure Mode | How Handled | Needs Test? |
### Acceptance criteria for the fix
### Test requirements
### Risk / notes
```

Pass the architect output directly to the Coder as the fix spec.

## Documentation Prompt

Use only for `--docs` after implementation succeeds:

```text
## Documentation update task
Update docs for the changes made in this orchestration run.

## Changes made
<batch summaries>

## Files that changed
<aggregate files>

## Documentation targets
CLAUDE.md if commands/patterns changed, memory files if conventions changed,
and docs referenced by the plan.

## Constraints
Only document actual changes. Keep additions concise.
```
