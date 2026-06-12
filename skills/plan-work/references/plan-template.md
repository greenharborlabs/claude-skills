# Plan Template And Exploration

Load this before writing or revising a plan.

## System Context

Read-only:

```bash
git log --oneline -20
git diff <BRANCH_REF> --stat
git rev-parse --short HEAD
```

Read `CLAUDE.md` when present. Read `TODOS.md` only if the request appears to
intersect current work. For architecture/plan docs, use `rg` or filenames first,
then read at most 2 targeted docs. Record the short commit hash in the plan
header.

If `--review <path>` is provided, read it. Otherwise check for recent
`reports/review-*.md` files only when the filename or heading contains a term
from the user request; read at most one matching report. Extract design findings
as plan inputs; implementation findings are context only.

## Exploration

Main exploration budget:

- Max 8 searches + 12 reads.
- `eng`: max 6 searches + 8 reads.
- Prefer targeted line ranges.

Identify existing leverage, conventions, interfaces, test patterns, and all
known callers/consumers of changed code.

Completeness sweep: max 3 more searches + 3 reads for key types/functions you
found. The goal is that a second plan-work run should not discover materially
different code.

CEO mode also records 2-3 strong style references and 1-2 anti-patterns.

## Premise Challenge

Before drafting, answer internally:

1. Is this the right problem?
2. What existing code partially solves it?
3. CEO: what is the 12-month ideal and does this move toward it?
4. Eng: can this be done with fewer files or abstractions?

If a clearly better approach appears, ask once before drafting.

## Required Plan Shape

Use `/orchestrate` Rule A:

````markdown
# [Plan Title]

**Created at:** `<short-hash>` on `<date>` | **Mode:** `<ceo|eng>`

## Summary
[2-3 sentences]

## Existing Code Leverage
[file paths + brief descriptions]

## Dream State *(ceo mode only)*
CURRENT STATE          THIS PLAN              12-MONTH IDEAL
[describe]      --->   [describe delta]  --->  [describe target]

## Architecture
[ASCII component diagram]

## Data Flow *(--spec only)*
[happy path and key error path diagrams]

## Blast Radius
| Modified File/Interface | Consumers | Covered by Work Unit? |
| --- | --- | --- |

## Wave 1: [Foundation]

### W1-01: [Imperative title]
[2-3 sentence description]

**Files:** path/to/file.ts (lines 10-50), path/to/new-file.ts (new)
**Acceptance criteria:**
- [Specific checkable condition]
**Error handling:** [Named failures + expected behavior]
**Error & rescue table:** *(--spec only)*
| Method/Codepath | Failure Mode | Rescue Behavior | HTTP Status |
| --- | --- | --- | --- |
**Tests:** [type + what to test]
**Test spec:**
- [Concrete behavioral test case]
**Effort:** *(--spec only)* S|M|L

## NOT in Scope
- [Deferred item] - [rationale]

## Security Considerations *(--spec only)*
[Trust boundaries, validation points, injection/auth surface]

## Failure Modes Summary
| Codepath | Failure Mode | Handled In | Tested? |
| --- | --- | --- | --- |

## Architect Review Findings
### Auto-Incorporated
### Resolved with User Input
### Deferred

## Confidence Assessment
| Dimension | Score | Source | Notes |
| --- | --- | --- | --- |

## Open Questions
[omit if none]

## Orchestration Playbook
```bash
/orchestrate plans/my-plan.md --scope "Wave 1"
/orchestrate plans/my-plan.md
```
````

## Drafting Rules

- Units in Wave N depend on all units in Wave N-1.
- Units within the same wave must be independent.
- Acceptance criteria must be independently verifiable.
- Test specs seed TDD: behavioral tests first, public interfaces only, mock only external boundaries.
- If plan touches more than 8 files in `eng` mode, justify it or propose a simpler alternative.
- For narrow `eng` plans, use a compact shape: Summary, Existing Code Leverage,
  Blast Radius, Waves, Failure Modes Summary, Architect Review Findings,
  Confidence Assessment, and Orchestration Playbook. Omit empty optional sections.
- In compact `eng` plans, make Architecture one short paragraph unless the work
  crosses module, service, persistence, or API boundaries.

Before writing, if `--out` already exists, ask whether to overwrite, choose a new
path, or read/build on the existing plan.

Plan size check after draft:

- Default threshold: >4 waves or >15 work units.
- Eng threshold: >3 waves or >10 work units.
- Ask whether to split plans, cut to MVP, or proceed as-is.
