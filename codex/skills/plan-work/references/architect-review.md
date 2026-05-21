# Architect Review

Load this immediately before spawning architect reviewer agents.

## Primary Architect Prompt

```text
You are reviewing an implementation plan for quality, completeness, and correctness.
Your review is independent. Approach it with fresh eyes.

## Plan Under Review
Read the plan file at: <output-path>

## Original Request
<paste original user description>

## Discovery Brief
<include if Phase 0 ran; otherwise omit>
These decisions have been resolved with the user. Do not re-question them.

## Your Task
Perform an independent review using your own codebase exploration. Do not trust
the plan's assumptions; verify them.

### Exploration Budget
- Max 12 file reads.
- Max 8 searches.
- Focus on modified files, callers/consumers, and adjacent modules.

### Mandatory Review Checklist
1. Blast radius: for every modified file/interface, search callers, consumers,
   importers, and dependents. Flag downstream impacts not covered by work units.
2. Missing work units: migrations, config/env/flags, consumers, API versioning,
   docs, monitoring/logging, seed data, fixtures, test infrastructure.
3. Acceptance criteria depth: criteria are checkable, edge cases covered, no
   vague wording, no implicit requirements left unstated.
4. Wave dependency validation: same-wave units are independent; later waves have
   real dependencies; hidden dependencies are named.
5. Error handling completeness: each new codepath has named failure modes and
   tests.
6. API/interface changes: all consumers updated, compatibility/migration clear.

### Return Format
ARCHITECT REVIEW
================
Overall: [1-2 sentence assessment]

FINDINGS:
F1: [CRITICAL|IMPORTANT|MINOR] [Checklist item 1-6]
    What: [specific gap]
    Where: [work unit or section]
    Evidence: [file path + codebase evidence]
    Fix: [concrete recommendation]

MISSING WORK UNITS:
M1: [title] - [why needed] - [suggested wave]

WAVE STRUCTURE:
[recommendations or "Wave structure is correct"]

CONFIDENCE SCORES:
Architecture:    [HIGH|MEDIUM|LOW] - [reason]
Error Handling:  [HIGH|MEDIUM|LOW] - [reason]
Test Strategy:   [HIGH|MEDIUM|LOW] - [reason]
Data Flow:       [HIGH|MEDIUM|LOW] - [reason] (skip in eng mode)
Security:        [HIGH|MEDIUM|LOW] - [reason] (skip in eng mode)
```

Eng-mode adjustment: if the plan has <=3 units and touches <=5 files, ask the
architect to skip checklist items 5-6 and score only Architecture, Error
Handling, and Test Strategy.

## Secondary Architect

Use only for cross-stack plans. Prompt the secondary architect to review only
its stack's work units with checklist items 1-3, budget 6 reads + 4 searches,
and confidence scores for Architecture and Test Strategy only. Prefix findings
with `[SECONDARY]`.

Merge findings by severity, deduplicate overlapping issues, and use the lower
confidence score when primary and secondary both score a dimension.

## Delta Re-Review

After incorporating findings, re-spawn the same architect only if the plan
changed structurally:

- New wave or reordered wave.
- Two or more new work units.
- Work unit moved between waves.
- New file/interface added to blast radius.

Delta budget: 4 reads + 3 searches. Ask only whether changed portions are sound.
Max one delta re-review.
