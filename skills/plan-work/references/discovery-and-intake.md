# Discovery And Intake

Load this only when discovery or exact intake behavior is needed.

## Conditional Discovery

Run Phase 0 when:

```text
IF --no-interview: skip
IF --interview: run
IF --review PATH provided: skip
IF --spec with >100 words in description: skip
IF description < ~50 words and no product-doc reference: run
OTHERWISE: skip
```

Tell the user whether discovery is running and why.

## Quick Codebase Scan

Before asking, spend max 3 searches + 3 reads to understand the touched area. If
the repo answers a question, do not ask it.

## Interview Rules

Ask 4-6 material questions only. Each question must fork the plan if answered
differently. Lead with a recommendation based on the scan and use lettered
options.

Good topics:

- Scope boundary.
- Key architecture fork.
- Integration shape.
- Data ownership.
- User-facing behavior for UI work.
- Hard external/team/business constraints.

Do not ask about implementation details, naming, libraries, file structure,
test framework, existing project patterns, or anything discoverable from code.

## Discovery Brief

Keep this in context, not in a separate file:

```text
DISCOVERY BRIEF
===============
Scope: [what is in/out]
Approach: [chosen architecture direction]
Integration: [how it connects to existing code]
Key decisions:
  1. [Decision] - [chosen option] - [why]
Constraints: [hard limits]
Open: [explicit deferrals]
```

If discovery ran, skip later vague-description questions because ambiguity was
already resolved.

## Mode And Intake

Resolve `auto`:

- New/build/create or no matching code -> `ceo`.
- Fix/refactor/update/bug or small touched surface -> `eng`.
- Uncertain -> ask once.

Questions:

- Q1 scope intent, skipped if discovery ran: new capability, enhancement, or fix/refactor.
- Q2 constraints, always: minimal diff, build it right, or move fast.
- Q3 ambition, ceo only: 10x, solid, or MVP.
- Q4 key decision, only if ambiguity remains after scan.

Every question: one issue, recommendation first, lettered options, no batching.

## Architect Agent Detection

Use max 2 glob calls:

| Signal | Architect |
| --- | --- |
| `*.java`, `*.gradle`, `pom.xml` | `backend-planning-architect` |
| `*.ts`/`*.tsx` + React signals | `frontend-architect` |
| `nanoclaw`/`container` signals | `nanoclaw-architect` |
| `*.py` + `openclaw`/`alpaca`/`kalshi` | `openclaw-architect` |
| Mixed/unknown | `backend-planning-architect` fallback |

Record a secondary architect only when the plan itself crosses stack boundaries,
not merely because the repo is a monorepo.
