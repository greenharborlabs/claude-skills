---
name: greenharbor-orchestrate-lean
description: "Execute a bounded implementation plan with minimal agent overhead: one Coder per dependency wave, targeted verification, and one final review of the cumulative diff. Use for small to medium, low-risk plan files when speed and token efficiency matter; use greenharbor-orchestrate instead for migrations, security-sensitive changes, broad public API changes, or work needing parallel Coders and specialist security/API reviews. Supports --scope, --dry-run, --review, and --no-review."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Agent
  - AskUserQuestion
---

# Lean Orchestrate

Act as a lightweight delivery manager. Preserve the user’s context and token
budget. Delegate implementation; do not edit implementation files yourself.

## Help

For no plan file or `--help`, print:

```text
/greenharbor-orchestrate-lean <plan-file>                    Execute a bounded plan
/greenharbor-orchestrate-lean <plan-file> --scope "Wave 1"   Execute matching section
/greenharbor-orchestrate-lean <plan-file> --dry-run          Show selected work only
/greenharbor-orchestrate-lean <plan-file> --review            Require one final review (default)
/greenharbor-orchestrate-lean <plan-file> --no-review         Skip final review explicitly
```

Then stop.

## Guardrails

1. Require a clean working tree, except for the selected plan file, unless the
   user explicitly authorizes continuation.
2. Use this skill only for bounded, low-risk work. Hand off to
   `greenharbor-orchestrate` when the plan includes migrations, payments,
   authentication/authorization, secrets, security fixes, destructive changes,
   externally consumed API contracts, or more than three dependency waves.
3. Run at most one worker at a time. Do not create an Architect, specialist,
   docs worker, or per-task Reviewer.
4. Do not push, deploy, change CI/CD, reset, or delete without explicit user
   approval.
5. Keep every worker prompt self-contained, scoped to the selected wave, and
   ask for a report of no more than 10 bullets.

## Workflow

1. Run `git status` and record `START_COMMIT`.
2. Read only the plan and enough repository context to identify its tasks,
   dependencies, and the narrowest relevant test command. Extract work units
   with the "Plan Extraction" and "Scope Syntax" rules in
   [task-parsing.md](../greenharbor-orchestrate/references/task-parsing.md)
   (skip its checkpoint rules). Resolve the Coder, Reviewer, and literal test
   command from the "Agent Suite" table in
   [stack-detection.md](../greenharbor-orchestrate/references/stack-detection.md)
   (skip its specialist section). For `--dry-run`, show the selected waves,
   agents, and test command, then stop.
3. For each dependency wave, dispatch exactly one Coder to implement all tasks
   in that wave, update/add focused tests, and run only targeted checks. Give it
   the plan excerpt, acceptance criteria, touched areas, and the literal test
   command. Ask it to end with `FILES_CHANGED`, `TESTS_RUN`, and
   `SELF_TEST: PASS | FAIL`, fixing its own failing tests at most twice.
   On `SELF_TEST: FAIL` or a missing report (after one retry), stop before the
   next wave and report the failure.
4. Capture only status, touched files, tests, and unresolved blockers between
   waves. Do not pause for approval between waves unless blocked.
5. Aggregate the focused tests after all waves finish.
6. Unless `--no-review` was explicitly supplied, dispatch one Reviewer to
   inspect the cumulative diff from `START_COMMIT`, prioritizing correctness,
   regressions, and missing acceptance criteria. Do not request style review or
   a full repository audit. First mark new files intent-to-add so the diff shows
   them: run `git add -N -- <paths>` for untracked files listed in Coder
   `FILES_CHANGED` (never the plan file). Give the Reviewer `START_COMMIT` and
   the changed-file list; it runs `git diff` itself. Do not read the diff into
   this session.
7. If the final reviewer finds a concrete correctness issue, send one Coder a
   focused fix and re-run only affected tests. Do not loop more than once; report
   remaining issues to the user.
8. Report completed tasks, files changed, checks run, review verdict, known
   risks, and the recommended next step.

## Worker Boundary

Begin every worker prompt with:

```text
You are a worker, not an orchestrator. Work directly on the assigned scope.
Do not spawn agents or invoke orchestration skills. Make only the requested
changes, run only the requested focused checks, and return a concise report.
Do not commit, stage, or stash; leave all changes in the working tree.
```

## Escalation

Stop and recommend `greenharbor-orchestrate` rather than widening this workflow
when new scope creates a high-risk boundary, dependencies are unclear, targeted
tests fail outside the selected work, or the single final review cannot establish
confidence.
