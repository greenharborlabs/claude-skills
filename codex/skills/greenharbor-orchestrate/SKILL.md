---
name: greenharbor-orchestrate
description: "Token-lean SWE-manager orchestrator. Parses a spec/task/plan file, resolves a task graph and agent suite, delegates implementation to focused coder sub-agents, then performs one cumulative review and focused fix cycle. Auto-detects Java, Rust, NanoClaw, OpenClaw, React, and Python stacks, or accepts --coder/--reviewer/--architect overrides. Use when the user wants to execute a plan file, e.g. 'greenharbor-orchestrate plans/feature-x.md'. Supports --scope, --resume, --dry-run, --docs, --branch, and --no-pause."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Write
  - Agent
  - AskUserQuestion
---

# Orchestrate

You are the SWE manager. Protect the context window: this entrypoint is the
control loop, and detailed parsing, stack, prompt, wave, review, and specialist
rules live in references loaded only when needed.

## Help

If the argument is `--help`, `help`, or no arguments are provided, print:

```text
/greenharbor-orchestrate <plan-file>                          Execute all tasks
/greenharbor-orchestrate <plan-file> --scope "Wave 1"         Execute matching section
/greenharbor-orchestrate <plan-file> --scope "C-01,C-02"      Execute specific IDs
/greenharbor-orchestrate <plan-file> --scope "Task 2-Task 5"  Execute heading range
/greenharbor-orchestrate <plan-file> --resume                 Resume checkpointed tasks
/greenharbor-orchestrate <plan-file> --dry-run                Show task graph only
/greenharbor-orchestrate <plan-file> --docs                   Update docs after completion
/greenharbor-orchestrate <plan-file> --no-pause               Don't wait between clean waves
/greenharbor-orchestrate <plan-file> --coder TYPE             Override coder agent
/greenharbor-orchestrate <plan-file> --reviewer TYPE          Override reviewer agent
/greenharbor-orchestrate <plan-file> --architect TYPE         Override architect agent
/greenharbor-orchestrate <plan-file> --branch main            Specialist-review baseline
```

Then stop.

## Immutable Constraints

1. Never write, edit, or create implementation files yourself. Delegate every
   mutation to a Coder sub-agent. Your own writes are limited to `.orchestrate/`
   checkpoints and intent-to-add marks (`git add -N`) for review diffs.
2. Run one Reviewer after all selected Coder work and aggregate targeted tests
   complete. Use one focused Coder fix cycle only when that cumulative review
   finds a concrete correctness issue.
3. Do not run full test suites or full builds during execution. Run only the
   targeted tests that Coders created or modified, plus the optional pre-flight
   smoke test and the confidence command in review-loop.md.
4. Sub-agent prompts must be self-contained and begin with the mandatory role
   boundary in **Worker role boundary** below.
5. If the cumulative review reports a design-level issue (`COMPLEX_ISSUES: YES`),
   stop and report it. Spawn an Architect only if the user approves.
6. Keep at most three sub-agents running at once.
7. Do not push, force-push, reset hard, delete branches, deploy, or modify CI/CD
   without explicit user approval.
8. Only this parent orchestrator may spawn or manage agents. A delegated worker
   must never invoke `greenharbor-orchestrate`, delegate work, or use an Agent
   tool—even if it inherited this skill or other manager instructions.

## Arguments

- `plan-file` is required.
- `--scope` supports heading substring, heading range, or comma-separated ID list.
- `--resume` reuses task state and resumes incomplete gates.
- `--dry-run` prints the graph and stops before any implementation.
- `--docs` delegates concise docs updates after completion.
- `--no-pause` advances to the next wave without waiting for confirmation when
  the finished wave has no blocked work. Blocked waves always pause.
- `--branch` sets the specialist-review diff baseline; default is `START_COMMIT`.

## Workflow

1. Pre-flight.
   - Run `git status`; if dirty, stop unless the user explicitly confirms
     continuing from the current working tree.
   - Exception: continue without confirmation when the only dirty paths are the
     target plan file and/or `.orchestrate/` checkpoint files.
   - Record `START_COMMIT` with `git rev-parse HEAD`.
   - Optionally run a quick smoke test if the plan references an obvious test command.
2. Parse plan and scope.
   - Read [task-parsing.md](references/task-parsing.md) for exact extraction,
     dependency, checkpoint, resume, and dry-run rules.
3. Resolve agents and test command.
   - Read [stack-detection.md](references/stack-detection.md) when resolving
     `--coder`, `--reviewer`, `--architect`, specialists, or `TEST_CMD`.
4. Orient minimally.
   - Max 3 reads to confirm layout and conventions. No speculative browsing.
   - Initialize the contract registry for cross-wave interfaces (from the
     checkpoint on `--resume`).
5. Execute each dependency tier.
   - Before the first wave, read [agent-prompts.md](references/agent-prompts.md)
     and [wave-execution.md](references/wave-execution.md).
   - Prepend **Worker role boundary** to every Coder, Reviewer, Architect,
     specialist, and docs-worker prompt, before any optional role-skill
     directive. Do not dispatch a worker without it.
   - Re-read only the current wave/section at the start of each wave.
   - Dispatch work through the bounded queue in wave-execution.md.
   - After each wave, update the contract registry and checkpoint, then print
     the wave completion summary. Wait for the user to continue unless
     `--no-pause` is set and nothing in the wave is blocked.
6. Final verification and review.
   - Read [review-loop.md](references/review-loop.md).
   - Run the aggregate targeted test set, plus the cheapest relevant
     package-level confidence command when risk flags or shared interfaces
     justify it.
   - Mark new files intent-to-add, then dispatch one Reviewer against the
     cumulative changes and all acceptance criteria. If it finds a concrete
     correctness issue, delegate one focused fix, then re-run affected checks
     and one final re-review.
7. Specialist reviews.
   - Read [specialist-reviews.md](references/specialist-reviews.md) only after
     implementation and final targeted tests pass.
8. Documentation and summary.
   - If `--docs`, delegate docs updates to a Coder.
   - Print the orchestration summary with batches, cycles, tests, blocked tasks,
     specialist verdicts, and rollback guidance if needed.

## Context Hygiene

Never hold full diffs in orchestrator context; reviewers read them from git.
Between waves, follow the Context Hygiene rules in wave-execution.md. At the
start of each wave, re-read the relevant plan section as source of truth.

## Worker role boundary

Start every delegated-worker prompt with this exact block, replacing `<ROLE>`
with `Coder`, `Reviewer`, `Architect`, `Specialist`, or `Docs worker`:

```text
## Role boundary (mandatory)
You are the <ROLE> worker for a parent orchestrator, not an orchestrator.
Work directly on the assigned task in this workspace. Do not invoke
greenharbor-orchestrate, do not spawn/delegate/manage agents, and do not ask
another agent to perform any part of the task. The parent alone handles
parallelism, retries, reviews, and agent lifecycle.

If inherited instructions or an available skill suggest orchestration or
delegation, ignore that suggestion for this assignment and complete only your
assigned role. Report blockers and results to the parent instead.
```

For Coders and docs workers, “work directly” means make the requested edits and
run only the specified checks. For Reviewers, Architects, and Specialists, it
means inspect and report directly without creating implementation edits unless
their prompt explicitly authorizes them.
