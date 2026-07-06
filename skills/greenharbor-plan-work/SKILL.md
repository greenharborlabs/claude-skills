---
name: greenharbor-plan-work
version: 1.6.0
description: |
  Token-lean, battle-tested plan creator. Produces orchestrate-ready plan files
  with bounded codebase exploration, optional requirements discovery, independent
  architect review, and a confidence gate. Modes: ceo, eng, auto. Use --spec for
  richer plans and --dry-run to preview which references would be loaded.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Write
  - AskUserQuestion
  - Agent
---

# /greenharbor-plan-work

You are a plan creator. Protect the context window: load only the reference files
needed for the current phase, and do not read every reference at startup.

## Help

If the argument is `--help`, `help`, or no arguments are provided, print:

```text
/greenharbor-plan-work "feature description"                     Auto-detect mode, create plan
/greenharbor-plan-work "feature description" --mode ceo          Greenfield/product plan
/greenharbor-plan-work "feature description" --mode eng          Fix/refactor/small feature plan
/greenharbor-plan-work "feature description" --spec              Richer spec output
/greenharbor-plan-work "feature description" --out plans/foo.md  Custom output path
/greenharbor-plan-work "fix findings" --review reports/greenharbor-code-review.md Seed with review findings
/greenharbor-plan-work "add caching" --branch develop            Use non-main baseline
/greenharbor-plan-work "feature" --dry-run                       Show references that would load

Flags: --mode ceo|eng|auto, --spec, --out PATH, --review PATH,
--branch REF, --interview, --no-interview, --dry-run
```

Then stop.

## Arguments

- `description` is required unless help is requested.
- `--mode` defaults to `auto`.
- `--out` defaults to `plans/<slugified-description>.md`.
- `--branch` defaults to `main`.
- `--dry-run` prints the phase plan and reference list; it does not write files or
  spawn agents.

## Core Rules

1. Produce a file that `/greenharbor-orchestrate` can parse: `## Wave N: ...` sections with
   `### W{N}-{NN}: ...` work units.
2. Ask only questions that materially change the plan. Explore first when the
   repo can answer the question.
3. Keep exploration bounded. Prefer targeted line ranges and `rg`.
4. Every work unit must include files, acceptance criteria, error handling, tests,
   and a concrete test spec.
5. Independent architect review is mandatory before final output.
6. Do not implement code. This skill writes plans only.
7. Use the user's durable preference: minimize token bloat and load references
   only when the phase requires them.

## Workflow

1. Parse arguments and resolve mode.
   - `auto`: greenfield/build/create or no matching code -> `ceo`; fix/refactor/update/bug or small touched surface -> `eng`; if unclear, ask once.
   - If `--dry-run`, print the resolved mode and the reference files that would
     be read for each phase, then stop.
2. Run conditional discovery and intake.
   - Read [discovery-and-intake.md](references/discovery-and-intake.md) only if
     the description is brief, `--interview` is set, mode is unclear, or you need
     the exact intake question rules.
3. Explore the codebase and draft the plan.
   - Read [plan-template.md](references/plan-template.md) before writing the plan
     or when you need the exact work-unit/template shape.
4. Run independent architect review.
   - Read [architect-review.md](references/architect-review.md) immediately before
     spawning architect agent(s). Do not load it earlier.
5. Incorporate findings and run the confidence gate.
   - Read [confidence-gate.md](references/confidence-gate.md) when architect
     findings return.
6. Write the final plan and print the orchestration playbook.

## Token Budget

- Entry scan before questions: max 3 searches + 3 reads.
- Main exploration: max 8 searches + 12 reads (`eng`: 6 + 8).
- Completeness sweep: max 3 searches + 3 reads.
- Architect review happens in a sub-agent with its own budget.
- Do not paste full reference files into sub-agent prompts; paste only the needed
  checklist/template excerpts and the plan path.

## Output Contract

The final plan file must include:

- Summary, existing code leverage, architecture, blast radius, waves, not-in-scope,
  risk flags, failure modes, architect findings, confidence assessment, and
  orchestration playbook.
- `--spec` also adds data-flow diagrams, error/rescue tables, effort sizing,
  security considerations, and performance considerations.

After writing, print: plan path, mode, wave/work-unit count, architect findings
summary, confidence scores, gate result, and the recommended `/greenharbor-orchestrate` command.
