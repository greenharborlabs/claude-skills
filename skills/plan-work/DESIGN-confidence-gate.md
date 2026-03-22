# Design: Confidence Gate for plan-work

## Problem

The current plan-work skill runs a linear 5-phase pipeline:

```
Intake → Explore → Draft → Self-Review → Output
```

Phase 4 (Self-Review) catches issues and can ask up to 3 questions, but there is
no mechanism to:

1. **Score confidence** per review dimension so the user sees where the plan is strong vs weak
2. **Loop back** to re-draft sections where confidence is low
3. **Present ASCII diagrams inline** for visual verification before finalizing
4. **Iterate rounds** until quality thresholds are met

The result is a single-pass plan that may ship with known weak spots the skill
identified but couldn't resolve alone.

---

## Proposed Solution: Phase 4.5 — Confidence Gate

Insert a new phase between Self-Review (Phase 4) and Output (Phase 5) that acts
as a quality gate with a feedback loop.

### Updated Pipeline

```
Phase 1: Intake + Scoping
    │
Phase 2: Codebase Exploration
    │
Phase 3: Draft Plan
    │
Phase 4: Self-Review (existing — produces review findings)
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  Phase 4.5: CONFIDENCE GATE                         │
│                                                     │
│  1. Score each review dimension (HIGH/MEDIUM/LOW)   │
│  2. Present confidence map + ASCII diagrams to user │
│  3. LOW  → AskUserQuestion, then re-draft section   │
│  4. MEDIUM → auto-refine, note what changed         │
│  5. HIGH → pass through                             │
│  6. Re-score after refinement                       │
│                                                     │
│  Max 2 refinement rounds                            │
│  Gate passes when all dimensions ≥ MEDIUM           │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
                  Phase 5: Output
```

---

## Confidence Dimensions

Scores map 1:1 to Phase 4's existing review sections:

| Dimension           | Maps to Phase 4 Section | What it measures |
|---------------------|-------------------------|------------------|
| Architecture        | Section 1               | Component boundaries, coupling, dependency graph clarity |
| Error Handling      | Section 2               | Named failure modes, coverage in acceptance criteria |
| Test Strategy       | Section 3               | Test type coverage, specificity of test requirements |
| Data Flow           | Section 4               | Happy + shadow path tracing, edge case handling |
| Security            | Section 5               | Auth model, input validation, injection surface |

**Eng mode (small changes):** Only score Architecture, Error Handling, Test Strategy
(matches the existing 3-section compression rule).

### Scoring Criteria

```
HIGH   — No gaps found. Acceptance criteria cover all identified paths.
         Diagrams are clear. No open questions.

MEDIUM — Minor gaps exist but are addressable without user input.
         Example: a missing test spec, an unnamed edge case, a vague
         acceptance criterion. Auto-refinable.

LOW    — Genuine uncertainty exists that requires user input.
         Example: unclear auth model, ambiguous failure behavior,
         multiple valid architectural approaches, missing domain knowledge.
```

---

## Gate Mechanics

### Step 1: Score

After Phase 4 completes, score each dimension. The scoring is a synthesis of
what Phase 4 found — not a separate analysis pass.

### Step 2: Present

Show the user a confidence map with inline ASCII diagrams:

```
┌─────────────────────────────────────────────────────┐
│  PLAN CONFIDENCE ASSESSMENT                         │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Architecture:     ██████████ HIGH                  │
│  Error Handling:   ██████░░░░ MEDIUM (auto-refine)  │
│  Test Strategy:    ████░░░░░░ LOW    (needs input)  │
│  Data Flow:        ██████████ HIGH                  │
│  Security:         ██████░░░░ MEDIUM (auto-refine)  │
│                                                     │
├─────────────────────────────────────────────────────┤
│  ARCHITECTURE (HIGH — verified)                     │
│                                                     │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐    │
│  │ API      │────▶│ Service  │────▶│ Repo     │    │
│  │ Controller│     │ Layer    │     │ Layer    │    │
│  └──────────┘     └────┬─────┘     └──────────┘    │
│                        │                            │
│                   ┌────▼─────┐                      │
│                   │ Event    │                      │
│                   │ Publisher│                      │
│                   └──────────┘                      │
│                                                     │
├─────────────────────────────────────────────────────┤
│  DATA FLOW (HIGH — verified)                        │
│                                                     │
│  Request ──▶ Validate ──▶ Transform ──▶ Persist     │
│     │            │             │            │       │
│     ▼            ▼             ▼            ▼       │
│  400 Bad      422 Invalid   500 Transform  409 Dup │
│  Request      Schema        Error          Key     │
│                                                     │
└─────────────────────────────────────────────────────┘
```

The diagrams shown are the actual diagrams from the plan draft — this is the
user's chance to verify them visually before the plan ships.

### Step 3: Resolve LOW dimensions

For each LOW dimension, present via AskUserQuestion:

- State what's uncertain and why
- Provide 2-3 lettered options with a recommendation
- One question per LOW dimension (not batched)

**Rule:** Max 2 AskUserQuestion calls per gate round. If there are 3+ LOW
dimensions, prioritize by impact (architecture > error handling > test >
data flow > security).

### Step 4: Auto-refine MEDIUM dimensions

For each MEDIUM dimension:

- Apply the fix silently (add missing test spec, name the edge case, etc.)
- Log what changed in a `## Refinement Log` section at the bottom of the plan

The user does NOT need to approve MEDIUM fixes, but they can see what changed.

### Step 5: Re-draft + Re-score

After resolving LOW items (with user input) and MEDIUM items (auto):

- Re-draft only the affected sections of the plan (targeted, no full re-draft)
- Re-score only the dimensions that were LOW or MEDIUM
- No new file reads — work with existing context + user answers

### Step 6: Gate Decision

```
IF all dimensions ≥ MEDIUM AND round ≤ 2:
    → Proceed to Phase 5 (Output)

IF any dimension still LOW AND round < 2:
    → Loop back to Step 2 (present updated map)

IF round = 2 AND any dimension still LOW:
    → Proceed to Phase 5 with a warning banner:
      "⚠ Plan shipped with LOW confidence in: [dimensions]"
      Add unresolved items to a ## Open Questions section in the plan
```

---

## Interaction Flow Example

### Round 1

```
Confidence Gate — Round 1 of 2

  Architecture:     ██████████ HIGH
  Error Handling:   ██████░░░░ MEDIUM (auto-refine)
  Test Strategy:    ████░░░░░░ LOW
  Data Flow:        ██████████ HIGH
  Security:         ████░░░░░░ LOW

[Architecture ASCII diagram shown inline — verified]
[Data Flow ASCII diagram shown inline — verified]

Resolving MEDIUM (auto):
  • Error Handling: Added timeout failure mode to W2-01 acceptance criteria

LOW — needs your input:

Q1: Test Strategy
  The plan adds 3 new service methods but the codebase has no integration
  test precedent for this module. What testing approach should we use?

  A) Unit tests only (Recommended) — matches existing patterns in src/services/
  B) Unit + integration with test containers — thorough but adds infrastructure
  C) Unit + contract tests — validates API shape without full integration

Q2: Security
  W1-02 adds a new endpoint that accepts file uploads. What auth model?

  A) Existing session auth (Recommended) — consistent with other endpoints
  B) Separate upload token with expiry — more secure but more complex
```

### Round 2 (if needed)

After user answers, targeted re-draft and re-score:

```
Confidence Gate — Round 2 of 2 (final)

  Architecture:     ██████████ HIGH
  Error Handling:   ██████████ HIGH   ▲ was MEDIUM
  Test Strategy:    ████████░░ MEDIUM ▲ was LOW (unit tests per your input)
  Data Flow:        ██████████ HIGH
  Security:         ██████████ HIGH   ▲ was LOW (session auth per your input)

All dimensions ≥ MEDIUM. Gate passed.

Auto-refining Test Strategy (MEDIUM):
  • Added specific test case outlines to W1-01 and W2-01

Plan ready for output.
```

---

## Integration with Existing SKILL.md

### Changes Required

1. **Phase 4 modification** — After self-review completes, produce a structured
   confidence score object (not just prose findings). The review sections already
   identify gaps — this just formalizes the scoring.

2. **New Phase 4.5 section** — Insert between Phase 4 and Phase 5 with the
   mechanics described above.

3. **AskUserQuestion budget adjustment** — Current budget:
   - Phase 1: 2-4 questions
   - Phase 4: max 3 questions
   - **New Phase 4.5: max 2 questions per round × 2 rounds = max 4 questions**
   - **New total: max 11 questions** (was max 7)

   This is acceptable because Phase 4.5 questions only fire when confidence is
   LOW, and most plans will pass the gate in round 1 with 0-1 questions.

4. **Plan file additions:**
   - `## Refinement Log` — what MEDIUM auto-fixes were applied
   - `## Open Questions` — unresolved LOW items (only if gate forced output at round 2)
   - Confidence scores in the `## Review Findings` section

5. **Completion summary update:**
   ```
   Plan written to: <path>
   Mode: <ceo|eng>
   Waves: N | Work units: M
   Confidence: Architecture=HIGH, Errors=HIGH, Tests=MEDIUM, Data=HIGH, Security=HIGH
   Gate: Passed round 1 (0 LOW remaining)
   Review: N issues found, N resolved in draft, N presented to user

   Next step:
     /orchestrate <path>
   ```

### What Does NOT Change

- Phase 1-3 are untouched
- Phase 4 review sections stay the same (just add scoring output)
- Phase 5 output mechanics stay the same
- `--mode`, `--spec`, `--out` flags unchanged
- Orchestrate-compatible Wave/Work-unit format unchanged
- Context budget rules for Phase 2 unchanged

---

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| All HIGH on first score | Skip gate interaction entirely. Print confidence map, proceed to output. |
| Eng mode small change (≤ 3 units, ≤ 5 files) | Only score 3 dimensions. Gate is lighter. |
| User disagrees with a diagram | Treat as LOW for that dimension. Re-draft with user's correction. |
| Context window pressure | If running low on context at gate entry, compress to 1 round max with a warning. |
| `--spec` mode | Diagrams are already richer. Gate presents the spec-level diagrams, not simplified ones. |
| No LOW or MEDIUM dimensions | Print confidence map (all HIGH), skip interaction, proceed. |

---

## AskUserQuestion Budget Summary

```
Phase 1:   2-4 questions  (scoping — always fires)
Phase 2:   0              (read-only, no interaction)
Phase 3:   0              (drafting, no interaction)
Phase 4:   0-1 questions  (premise challenge only — reduced from max 3)
Phase 4.5: 0-4 questions  (confidence gate — 0-2 per round × max 2 rounds)
Phase 5:   0              (output, no interaction)

Typical run:  4-6 questions total
Worst case:   9 questions (complex CEO plan, 2 full gate rounds)
Best case:    2 questions (clean eng fix, gate passes immediately)
```

Note: Phase 4's budget drops from 3 to 1 because the confidence gate now
handles the tradeoff questions that Phase 4 previously surfaced. Phase 4 focuses
purely on review/scoring; Phase 4.5 handles the user interaction.

---

## Implementation Plan

To apply this design to the existing SKILL.md:

1. Add confidence scoring criteria to Phase 4 (scoring output after each section)
2. Insert Phase 4.5 section with gate mechanics
3. Update Phase 4 AskUserQuestion budget (3 → 1, premise challenge only)
4. Add `## Refinement Log` and `## Open Questions` to the plan template
5. Update completion summary format
6. Update Critical Rules with gate-specific rules
7. Update Mode Quick Reference table with gate row

Estimated diff: ~80-100 lines added to SKILL.md, ~10 lines modified.
