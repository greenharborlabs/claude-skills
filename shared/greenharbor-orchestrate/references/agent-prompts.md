# Agent Prompts

Load this before constructing Coder, Reviewer, Architect, or docs prompts.

Every prompt starts with the worker role boundary block from SKILL.md, then the
optional skill directive, then the template body.

## Optional Codex Skill Overlay

Claude Code: skip this step; agent types already carry their role.

Codex: before constructing a role prompt, check whether the exact resolved role
name is available as a Codex skill. If it is, insert `Use $<resolved-role-name>
for this task.` right after the role boundary. If it is not, omit the directive
and rely on the self-contained prompt below. Never emit an unresolved
placeholder or invent a skill name.

## Coder Prompt

```text
<optional Coder skill directive; omit when unavailable>

## Task
<imperative work group title>

## Spec excerpt
<short relevant plan excerpt; omit unrelated background>

## Files to modify
<file paths + relevant line ranges>

## Interfaces / contracts to conform to
<signatures, types, API shapes from exploration or contract registry>

## Affected tests
<test files>

## Success criteria
<AC-ID list with exact criterion text>

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
- Do not commit, stage, or stash; leave all changes in the working tree.

## Watch for
<stack-specific warnings from below + universal warnings>

## Self-test before reporting
Run: <literal TEST_CMD with affected tests>
If it fails, fix and re-run. After 2 fix attempts, stop and report FAIL with the
failing test names and a short error summary.

## Completion report format
FILES_CHANGED: <comma-separated list>
TESTS_TO_RUN: <comma-separated test files>
SELF_TEST: PASS | FAIL (details if FAIL)
NEW_INTERFACES: <new public types/endpoints/signatures, if any>
NOTES: <unusual context>
```

## Stack Warnings

Java: transaction boundaries around external calls, demonstrated lazy-load query
loops, unsafe dynamic queries, boundary validation or authorization gaps,
unbounded work, and compatibility with the configured Java/Spring versions.

React: missing effect cleanup, unsafe HTML without sanitization, async effects
without cancellation.

Python: mutable defaults, swallowed broad exceptions, shell injection, missing
await, file handles without context managers.

Rust: panics on untrusted paths, unnecessary clones masking ownership issues,
blocking work or locks across `.await`, unbounded tasks/channels, missing
cancellation or timeouts, unsafe/FFI invariants, incompatible public API or
serde changes, and unreviewed Cargo features/build scripts/dependencies.

NanoClaw/OpenClaw: missing risk gates, hardcoded secrets, broker retry/timeout,
state persistence/reconciliation, rate limiting, circuit breakers.

Universal: secrets in source, non-atomic check-then-act, unvalidated untrusted
input in sensitive operations.

## Reviewer Prompt

```text
<optional Reviewer skill directive; omit when unavailable>

## Review task
Review the cumulative implementation against the spec.

## Spec excerpt(s)
<short relevant spec excerpts; omit unrelated background>

## Coder self-test results
<per work group: PASS/FAIL details>

## Aggregate verification
<targeted test and confidence-command results>

## New interfaces introduced
<contract registry entries, if any>

## Changes to review
Base commit: <START_COMMIT>
Changed files:
<CHANGED_FILES>
Run `git diff <START_COMMIT> -- <file>` yourself, file by file. Default -U8;
widen to -U20 only when needed. New files are marked intent-to-add, so they
appear in the diff.

## Success criteria
<all task-derived AC-IDs with exact criterion text>

## Acceptance Criteria Verification (MANDATORY)
For EACH criterion, output:
CRITERIA_CHECKLIST:
  [PASS] "criterion text" - evidence: <file>:<line>
  [FAIL] "criterion text" - not implemented: <missing>
  [PARTIAL] "criterion text" - <exists> / <missing>

Rules:
- Copy each AC-ID and criterion text verbatim from the provided list.
- Any FAIL/PARTIAL is CRITICAL.
- Evidence must point to a changed line in one of the changed files.

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

Use only after Reviewer sets `COMPLEX_ISSUES: YES` and the user approves
Architect escalation.

```text
<optional Architect skill directive; omit when unavailable>

## Architect task
Produce a targeted fix plan for a complex review failure. Do not edit files.

## Original spec excerpt
<spec>

## Current implementation
Base commit: <START_COMMIT>
Changed files:
<CHANGED_FILES>
Run `git diff <START_COMMIT> -- <file>` for the files you need.

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
