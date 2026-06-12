# Specialist Reviews

Load only after all batches and final targeted verification pass.

## When To Run

Skip if no specialist reviewers were resolved. For Java stacks:

- Security reviewer always runs.
- API contract reviewer runs only when aggregate diff touches controller,
  resource, DTO, request/response, controller advice, or OpenAPI files.

## Collect Context

Set review baseline:

```bash
REVIEW_BASE=${BRANCH_REF:-$START_COMMIT}
git diff $REVIEW_BASE
git diff $REVIEW_BASE --stat
```

If diff is >500 lines, focus each specialist on domain-relevant files.

## Specialist Prompt

```text
## Post-Implementation Audit
Review ALL changes from this orchestration run.

## Plan context
<plan Summary section>

## Aggregate diff (against <REVIEW_BASE>)
<diff>

## Changed files
<diff stat>

## Output format
VERDICT: PASS | FAIL
FINDINGS:
  [CRITICAL] <file>:<line> - <description> - <category>
  [WARNING]  <file>:<line> - <description> - <category>
  [INFO]     <file>:<line> - <description>
SUMMARY: <1-3 sentences>
```

Spawn specialists in parallel when possible.

## Process Findings

Merge and deduplicate findings.

If all PASS: log and continue.

If any CRITICAL:

1. Group by file.
2. Spawn Coder with a surgical specialist-fix prompt.
3. Run targeted tests.
4. Max 2 fix cycles.
5. Run one targeted specialist recheck limited to the files and findings fixed.
   Do not rerun the full specialist audit.

Warnings and info are logged only unless the user explicitly requests fixes.

Mark `specialist-review-gate` complete when done or when specialists are skipped.

## Summary Shape

Include in final summary:

```text
| Reviewer | Verdict | Critical | Warning | Info | Fixes Applied |
| --- | --- | --- | --- | --- | --- |

Security summary: ...
API Contract summary: ...
Unresolved specialist findings: ...
```
