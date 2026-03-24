---
name: plan-work
version: 1.5.0
description: |
  Battle-tested plan creator. Combines architect-level codebase exploration with
  independent architect sub-agent review and a confidence-gated feedback loop to
  produce orchestrate-ready plan files. Modes: ceo (greenfield/ambitious), eng
  (fixes/small features), auto (detect). Use --spec for rich specifications.
  Includes conditional requirements discovery interview for underspecified inputs.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Write
  - AskUserQuestion
  - Agent
---

# /plan-work — Battle-Tested Plan Creator

## Help

If the argument is `--help`, `help`, or no arguments are provided, print this usage
summary and stop:

```
/plan-work — Create battle-tested, orchestrate-ready implementation plans

Usage:
  /plan-work "feature description"                     Auto-detect mode, create plan
  /plan-work "feature description" --mode ceo          CEO mode: scope challenge + ambition
  /plan-work "feature description" --mode eng          Eng mode: execution rigor
  /plan-work "feature description" --spec              Rich spec output (more detail)
  /plan-work "feature description" --out plans/foo.md  Custom output path
  /plan-work "fix review findings" --review reports/review-scope-OrderService-2026-03-15.md

Modes:
  auto (default)  Greenfield → ceo, fix/refactor/small → eng
  ceo             Premise challenge, dream-state mapping, architect review (5 dimensions)
  eng             Complexity check, architect review (3 dimensions)

Flags:
  --spec          Rich output: full data flow diagrams, error tables, test specs
  --out PATH      Output file path (default: plans/<slugified-description>.md)
  --review PATH   Seed plan with findings from a /review report
  --interview     Force requirements discovery interview (Phase 0)
  --no-interview  Skip requirements discovery interview
```

## Arguments

1. **`description` (required, positional)** — What to build. Can be a sentence or paragraph.
2. **`--mode ceo|eng|auto`** — Planning mode. Default: `auto`.
3. **`--spec`** — Produce rich specification output instead of lean orchestrate format.
4. **`--out PATH`** — Output file path. Default: `plans/<slugified-description>.md`.
5. **`--review PATH`** — Path to a `/review` report to seed exploration with known findings.
6. **`--interview`** — Force requirements discovery interview even if description is detailed.
7. **`--no-interview`** — Skip requirements discovery interview even if description is brief.

---

## Phase 0: Requirements Discovery (Conditional)

A focused interview that fires when the input is underspecified. Resolves the key
decisions and constraints that would most change the shape of the plan — not an
exhaustive requirements gathering session.

### Trigger Logic

```
IF --no-interview:                → SKIP Phase 0
IF --interview:                   → RUN Phase 0
IF --review PATH provided:        → SKIP (review report provides sufficient context)
IF --spec provided with >100 words in description: → SKIP (user has thought this through)
IF description < ~50 words AND no product doc reference: → RUN Phase 0
OTHERWISE:                        → SKIP
```

Tell the user whether Phase 0 is running and why:
- "Description is brief — running requirements discovery to fill gaps before planning."
- "Product doc provided — skipping discovery, proceeding to planning."

### 0a. Quick Codebase Scan

Before asking anything, spend **max 3 Glob/Grep calls + 3 file reads** to understand
what already exists in the area the description touches. This is NOT Phase 2 exploration
— it's just enough context to ask informed questions instead of obvious ones.

If a question can be answered by what you find in the codebase, don't ask it. This is
the grill-me principle: explore instead of interrogating when the code has the answer.

### 0b. Decision Tree Interview (4-6 AskUserQuestion calls)

Walk down the decision tree for this feature, resolving dependencies between decisions
one by one. Each question should:

- **Target a decision that would fork the plan** — not a detail that can be decided during
  implementation. Ask: "If the user answered differently, would the plan have different
  waves or work units?" If no, don't ask.
- **Lead with a recommendation** based on what the codebase scan revealed
- **Use lettered options** (A/B/C) — the user can always pick "Other"
- **Reference specific code** when relevant ("I see you're using X pattern in `src/foo.ts`,
  should we follow the same approach here?")

**Focus areas** (pick the 4-6 most relevant, not all):
- **Scope boundaries** — What's in vs out? Where does this feature end and another begin?
- **Key architectural fork** — Is there a fundamental approach choice (e.g., polling vs
  WebSocket, server vs client rendering, new service vs extend existing)?
- **Integration shape** — How does this connect to what exists? New API surface or extend current?
- **Data ownership** — Where does the source of truth live? New storage or extend existing?
- **User-facing behavior** — For UI features: what does the user see on success, failure,
  loading, empty state? What's the primary interaction model?
- **Constraints the plan must respect** — Hard limits from external systems, team agreements,
  or business rules that aren't in the code

**Do NOT ask about:**
- Implementation details (which library, naming conventions, file structure)
- Things derivable from the codebase (existing patterns, test framework, project structure)
- Performance targets or security models (architect review handles these)

### 0c. Discovery Brief

After the interview, compile a **discovery brief** — a structured internal artifact (not
written to a file) that feeds into Phase 1 and Phase 2:

```
DISCOVERY BRIEF
===============
Scope: [one sentence — what's in, what's out]
Approach: [the architectural direction chosen]
Integration: [how it connects to existing code]
Key decisions:
  1. [Decision] — [chosen option] — [why]
  2. [Decision] — [chosen option] — [why]
  ...
Constraints: [hard limits identified]
Open: [anything explicitly deferred to implementation]
```

This brief replaces Phase 1's Q4 (the vague-description question) — if Phase 0 ran,
Q4 is always skipped because the ambiguity has already been resolved.

### Budget Protection

Phase 0 is lightweight by design:
- **Max 3 Glob/Grep + 3 file reads** (codebase scan)
- **Max 6 AskUserQuestion calls** (interview)
- **No file writes** — the discovery brief is in-context only
- These reads do NOT count against Phase 2's budget — they serve a different purpose
  (understanding what to ask vs understanding what to build)

---

## Phase 1: Intake + Scoping

### 1a. Mode Resolution

If `--mode auto` or unspecified, resolve:
- Description mentions "new", "build", "create", or no existing code matches → `ceo`
- Description mentions "fix", "refactor", "update", "bug", or touches < 8 files → `eng`
- Uncertain → ask in Q1

Tell the user which mode was selected and why.

### 1b. Scoping Questions (1-4 AskUserQuestion calls)

**Q1 (skip if Phase 0 ran): Scope intent.**
Skip if the discovery brief already classifies scope (it always will). Otherwise ask:
"What best describes this work?" Options:
- A) New capability (greenfield)
- B) Enhancement to existing feature
- C) Fix/refactor existing code

**Q2 (always): Constraints.**
"What constraints should the plan respect?" Options:
- A) Minimal diff — smallest change that works
- B) Build it right — invest in architecture
- C) Move fast — accept some tech debt

**Q3 (ceo mode only): Ambition level.**
"How ambitious should we be?" Options:
- A) 10x — rethink the approach, find the cathedral
- B) Solid — good engineering, known patterns
- C) MVP — ship the minimum viable version

**Q4 (conditional): Key decision.**
If the description is vague or a quick codebase scan reveals genuine ambiguity, ask ONE
targeted question about the single most ambiguous aspect. **Skip if Phase 0 ran** (ambiguity
was already resolved in discovery). Also skip if the description is clear.

Critical rule for all questions: lead with your recommendation. "We recommend A: [reason]."
Use lettered options. One issue per question.

### 1c. Agent Suite Resolution

Resolve the **primary** architect agent type for Phase 4 review. If not obvious from
mode/description, auto-detect by scanning the project:

| Signal | Architect Agent |
|--------|----------------|
| `*.java`, `*.gradle`, `pom.xml` | backend-planning-architect |
| `*.ts`/`*.tsx` + React signals | frontend-architect |
| `nanoclaw`/`container` signals in package.json | nanoclaw-architect |
| `*.py` + `openclaw`/`alpaca`/`kalshi` references | openclaw-architect |
| Mixed/unknown | backend-planning-architect (general fallback) |

Quick detection: **max 2 Glob calls**. Use project root file patterns — do not read files
for this step. Print: `Architect: <agent-type> (detected from: <signal>)`

**Secondary stack detection:** If the Glob calls reveal files from **two or more** stack
families (e.g., both `*.java` and `*.tsx`), record the secondary architect type. The
secondary architect will be spawned in Phase 4 with a reduced scope (see Phase 4a½).
Print: `Secondary architect: <agent-type> (detected from: <signal>)`

Only record a secondary stack if the plan's description or Phase 2 exploration touches
files in both stacks. A monorepo with backend + frontend code does not trigger secondary
review unless the plan itself crosses the boundary.

---

## Phase 2: Codebase Exploration

Read-only. No user interaction. Strict budget to protect context.

### 2a. System Context
```bash
git log --oneline -20          # Recent history
git diff main --stat           # What's already changed
git rev-parse --short HEAD     # Current commit hash (record for plan staleness detection)
```
Read CLAUDE.md, TODOS.md, and any existing architecture/plan docs.
Record the current commit hash — it will be written into the plan header for staleness detection.

### 2a½. Review Report Ingestion

If `--review <path>` is provided, read that file. Otherwise, check for recent reports:
```bash
ls -t reports/review-*.md 2>/dev/null | head -3
```

If a report exists from the last 7 days (check file modification date), read it and extract:
- **DESIGN findings** — these are the problems the plan should solve
- **IMPL findings** — note these exist but they're out of scope for planning
- File paths and line numbers from findings (seed Phase 2b exploration)

Budget: reading 1 review report counts as 1 of the 12 file reads.

In Phase 3 output, include a `## Review Context` section linking findings to work units:
"This plan addresses D1, D2, D3 from [review report path]."

### 2b. Targeted Exploration
- **Max 8 Glob/Grep calls** (eng mode: 6) — find files relevant to the description.
- **Max 12 file reads** (eng mode: 8) — targeted line ranges preferred over full files.
- **Identify:**
  - Existing code that partially or fully solves sub-problems
  - Patterns and conventions to follow
  - Interfaces and contracts to conform to
  - Test patterns in use (framework, file structure, naming)
  - All callers, consumers, and dependents of code being changed

### 2b½. Completeness Sweep
After initial exploration, do one verification pass:
- **Search for what you might have missed:** grep for the key types, functions, and
  constants you discovered — are there other consumers or related modules?
- **Max 3 additional Glob/Grep calls + 3 file reads** for this sweep.
- If the sweep reveals significant new context, incorporate it before moving to 2c/2d.
- Goal: a second run of plan-work on the same prompt should NOT find materially
  different code or produce a more robust plan.

### 2c. CEO Mode Only: Taste Calibration
Identify 2-3 well-designed files/patterns in the codebase as style references.
Note 1-2 anti-patterns to avoid repeating.

### 2d. Premise Challenge (from CEO Step 0)
Before drafting, answer:
1. **Is this the right problem?** Could a different framing yield a simpler or more impactful solution?
2. **What existing code already partially solves this?** Map every sub-problem to existing code.
3. **CEO mode:** What would the ideal end state look like in 12 months? Does this plan move toward it?
4. **Eng mode:** Complexity check — can this be achieved with fewer files/abstractions?

If the premise challenge reveals a fundamentally better approach, present it via AskUserQuestion
before proceeding to Phase 3. Otherwise, proceed silently.

---

## Phase 3: Draft Plan

Using the Context Map from Phase 2 and user answers from Phase 1, generate the plan.

### Output Structure (orchestrate Rule A format)

The plan MUST use `## Wave N: [Title]` headings with `### W{N}-{NN}: [Work unit title]`
sub-items. This is the format that `/orchestrate` parses (Rule A).

- Units in Wave N depend on all units in Wave N-1.
- Units within the same wave are independent (parallel-eligible by orchestrate).

### Each Work Unit Must Include

```markdown
### W1-01: [Imperative title]
[2-3 sentence description of what to implement and why]

**Files:** [list of files to create/modify with relevant line ranges]
**Acceptance criteria:**
- [Specific, checkable condition — "FooService.process() returns {status:'ok'}" not "implement correctly"]
- [Another specific condition]
**Error handling:** [Named failure modes and expected behavior for each]
**Tests:** [Test type (unit/integration/e2e) + what specifically to test]
**Test spec:**
- [Concrete behavioral test case: "user can X" — input: {...}, expected: {...}]
- [Rejection case: "system rejects Y when Z" — input: {...}, expected: error/status]
- [These seed TDD — coder writes these tests FIRST, before implementation]
- [Mocking boundary: only mock external APIs, databases, time/randomness — never internal collaborators]
```

### CEO Mode Additions
- Include `## Dream State` section with the 3-column diagram:
  ```
  CURRENT STATE          THIS PLAN              12-MONTH IDEAL
  [describe]      --->   [describe delta]  --->  [describe target]
  ```
- Include `## NOT in Scope / Phase 2 Ideas` section

### Eng Mode Additions
- If plan touches > 8 files, include a justification or propose a simpler alternative.
- Include `## NOT in Scope` section (leaner, no phase 2 ideas).

### 3b. Write Draft to Output Path

Before writing, check if a file already exists at the output path. If it does, present
via AskUserQuestion: "A plan already exists at `<path>`. (A) Overwrite it (B) Choose a
different path (C) Read it first and build on it." Do not overwrite silently — the
existing plan may be partially executed.

After resolving, write the plan to the output path using the Write tool. The architect
sub-agent in Phase 4 needs to read this file. Create the parent directory if needed
(`mkdir -p`).

### 3c. Plan Size Check

After writing the draft, count waves and work units. If the plan exceeds **4 waves or
15 work units**, present via AskUserQuestion:

"This plan has N waves and M work units, which may be too large for a single orchestrate
run. How should we proceed?"
- A) **Split into multiple plans** — I'll identify a natural boundary and produce two
  smaller plans, each independently orchestrate-able
- B) **Cut to MVP scope** — I'll trim to the minimal version that delivers core value,
  moving cut items to NOT in Scope
- C) **Proceed as-is** — the scope is intentional

This check fires before the architect review (Phase 4), since it's cheaper to trim scope
before spawning the sub-agent. If the user chooses A, re-draft and write both plan files
before proceeding — each gets its own Phase 4 architect review. If B, re-draft the
trimmed plan in place.

**Eng mode:** Use stricter thresholds — **3 waves or 10 work units** — since eng-mode
plans should be lean by nature.

---

## Phase 4: Architect Sub-Agent Review

**This phase replaces self-review with an independent architect review.** The architect
sub-agent has its own context window, does its own codebase exploration, and brings
fresh eyes with no anchoring bias from the drafting process. This is the single most
important quality improvement in the pipeline — it catches missing work units, shallow
acceptance criteria, and incorrect wave dependencies that same-brain self-review misses.

### 4a. Spawn Architect Review

Using the architect agent type resolved in Phase 1c, spawn a sub-agent via the Agent
tool with the following prompt structure. The prompt MUST be self-contained — the
sub-agent has no memory of prior phases.

**Architect sub-agent prompt template:**

```
You are reviewing an implementation plan for quality, completeness, and correctness.
Your review is independent — you have not seen the plan before and should approach it
with fresh eyes.

## Plan Under Review
Read the plan file at: <output-path>

## Original Request
<paste the user's original description here>

## Discovery Brief (if Phase 0 ran — omit this section otherwise)
<paste the discovery brief here — scope, approach, key decisions, constraints>
These decisions have been resolved with the user. Do NOT re-question them. Instead,
verify the plan implements them correctly.

## Your Task
Perform an independent review of this plan using your own codebase exploration.
Do NOT trust the plan's assumptions about the codebase — verify them yourself.

### Exploration Budget
- **Max 12 file reads** (targeted line ranges preferred)
- **Max 8 Glob/Grep searches**
- Focus on: files the plan modifies, their callers/consumers, and adjacent modules

### Mandatory Review Checklist

Work through EVERY item. Do not skip any.

**1. Blast-Radius Analysis**
For every file the plan modifies, search for ALL callers, consumers, importers, and
dependents. Are downstream impacts accounted for in the plan? List any consumers
that are not addressed by a work unit.

**2. Missing Work Units**
Are there pieces of work the plan omits? Check for:
- Database migrations or schema changes
- Configuration file changes (env vars, feature flags, config maps)
- Downstream consumer updates (other services, clients, or modules that depend on
  changed interfaces)
- API versioning or backward compatibility work
- Documentation updates (API docs, README, architecture diagrams)
- Monitoring/alerting/logging additions for new codepaths
- Seed data, test fixtures, or test infrastructure setup

**3. Acceptance Criteria Depth**
For each work unit's acceptance criteria:
- Is each criterion independently verifiable by an engineer? Could they determine
  pass/fail without ambiguity?
- Are edge cases covered (null/empty inputs, concurrent access, error states)?
- Would a coder implementing ONLY what the criteria say produce correct code, or
  are there implicit requirements?
- Flag any criterion that is vague ("works correctly", "handles errors properly")

**4. Wave Dependency Validation**
- Are units in the correct waves? Check: does any unit in Wave N actually depend on
  a unit in the same wave (they should be independent)?
- Should any units be reordered? Does any later-wave unit have no actual dependency
  on earlier waves?
- Are there hidden dependencies not captured by the wave structure?

**5. Error Handling Completeness**
For each new codepath in the plan:
- Name 2-3 specific failure modes
- Is each handled in acceptance criteria? Is there a test for it?
- Are there untested error paths?

**6. API Contract & Interface Changes**
If the plan changes any interface (method signature, API endpoint, event schema,
database schema):
- Are ALL consumers updated?
- Are there backward compatibility concerns?
- Is the migration path clear?

### Return Format

You MUST return your findings in this exact structure:

ARCHITECT REVIEW
================
Overall: [1-2 sentence assessment — is this plan ready to ship or does it need work?]

FINDINGS:
F1: [CRITICAL|IMPORTANT|MINOR] [Checklist item 1-6]
    What: [specific gap or issue found]
    Where: [which work unit(s) or plan section]
    Evidence: [file path + what you found in the codebase that reveals this gap]
    Fix: [concrete recommendation — add work unit, tighten criterion, reorder wave, etc.]

F2: ...
(list ALL findings, not just top 3)

MISSING WORK UNITS:
M1: [suggested title] — [why needed] — [suggested wave]
M2: ...
(omit section if none found)

WAVE STRUCTURE:
[Any reordering recommendations, or "Wave structure is correct"]

CONFIDENCE SCORES:
Architecture:    [HIGH|MEDIUM|LOW] — [one-line reason]
Error Handling:  [HIGH|MEDIUM|LOW] — [one-line reason]
Test Strategy:   [HIGH|MEDIUM|LOW] — [one-line reason]
Data Flow:       [HIGH|MEDIUM|LOW] — [one-line reason]  (skip in eng mode)
Security:        [HIGH|MEDIUM|LOW] — [one-line reason]  (skip in eng mode)
```

**Eng mode adjustment:** Tell the architect to skip checklist items 5-6 and score only
3 dimensions (Architecture, Error Handling, Test Strategy) if the plan has ≤ 3 work
units and touches ≤ 5 files.

### 4a½. Secondary Architect Review (Cross-Stack Plans Only)

**Skip this step if no secondary stack was detected in Phase 1c.**

If a secondary architect type was recorded, spawn it in parallel with (or after) the
primary architect using a **reduced-scope prompt**:

```
You are reviewing a cross-stack implementation plan from the perspective of the
<secondary-stack> layer. A primary architect is reviewing the full plan — your
role is to catch issues specific to the <secondary-stack> code that the primary
architect may not have domain expertise for.

## Plan Under Review
Read the plan file at: <output-path>

## Your Scope
Focus ONLY on work units that touch <secondary-stack> files (e.g., *.tsx, *.java).
Skip work units that are purely in the other stack.

## Reduced Checklist (items 1-3 only)
1. Blast-Radius Analysis — for <secondary-stack> files only
2. Missing Work Units — anything missing from the <secondary-stack> perspective
3. Acceptance Criteria Depth — for <secondary-stack> work units

## Reduced Budget
- **Max 6 file reads** (targeted line ranges preferred)
- **Max 4 Glob/Grep searches**

## Return Format
Use the same ARCHITECT REVIEW format as the primary architect, but prefix findings
with [SECONDARY] so they can be distinguished during merge.

CONFIDENCE SCORES (secondary perspective only):
Architecture:    [HIGH|MEDIUM|LOW] — [one-line reason, from <secondary-stack> view]
Test Strategy:   [HIGH|MEDIUM|LOW] — [one-line reason, from <secondary-stack> view]
```

The secondary architect scores only 2 dimensions. These do NOT replace the primary
architect's scores — they supplement them. If the secondary architect scores a dimension
lower than the primary, use the **lower** score (conservative merge).

### 4b. Parse Architect Findings

When the architect sub-agent(s) return, parse structured output into:
- **CRITICAL findings** — must be resolved before shipping
- **IMPORTANT findings** — should be incorporated
- **MINOR findings** — incorporate if easy, otherwise defer
- **Missing work units** — add to appropriate waves
- **Confidence scores** — feed into Phase 4.5

**Cross-stack merge:** If a secondary architect returned findings, merge them with the
primary architect's findings. Deduplicate: if both architects flag the same issue, keep
the higher-severity version. For confidence scores, use the **lower** score when both
architects assessed the same dimension (conservative approach). Tag merged secondary
findings with `[secondary]` in the Architect Review Findings section of the plan.

---

## Phase 4.5: Incorporate Findings + Confidence Gate

This phase takes the architect's independent findings and uses them to improve the plan,
then runs a quality gate before output.

### Confidence Dimensions

Scores come from the architect's independent assessment:

| Dimension        | What it measures |
|------------------|------------------|
| Architecture     | Component boundaries, coupling, dependency graph, blast radius |
| Error Handling   | Named failure modes, acceptance criteria coverage |
| Test Strategy    | Test type coverage, specificity of test requirements |
| Data Flow        | Happy + shadow path tracing, edge case handling |
| Security         | Auth model, input validation, injection surface |

Eng mode (small changes): Only score Architecture, Error Handling, Test Strategy.

### Step 1: Auto-Incorporate Findings

For each architect finding:

- **IMPORTANT findings:** Apply fixes silently — add missing acceptance criteria,
  tighten vague criteria, add error handling, update file lists.
- **MINOR findings:** Apply if straightforward, otherwise add to `## Open Questions`.
- **Missing work units (M1, M2, ...):** Add to the plan in the architect's suggested
  wave. Use the same work unit format (Files, Acceptance criteria, Error handling,
  Tests, Test spec).
- **Wave reordering:** Apply if the architect's reasoning is sound.

Log all changes in `## Architect Review Findings` section of the plan.

### Step 2: Present Confidence Map + CRITICAL Findings

Show the user:

```
┌─────────────────────────────────────────────────────┐
│  PLAN CONFIDENCE ASSESSMENT — Round 1 of 2          │
│  (scored by independent architect review)            │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Architecture:     ██████████ HIGH                  │
│  Error Handling:   ██████░░░░ MEDIUM (auto-refined) │
│  Test Strategy:    ████░░░░░░ LOW    (needs input)  │
│  Data Flow:        ██████████ HIGH                  │
│  Security:         ██████░░░░ MEDIUM (auto-refined) │
│                                                     │
├─────────────────────────────────────────────────────┤
│  Architect findings: N total                         │
│    Auto-incorporated: N IMPORTANT, N MINOR           │
│    Missing work units added: N                       │
│    CRITICAL (needs your input): N                    │
└─────────────────────────────────────────────────────┘
```

Include ASCII diagrams from the plan for visual verification.

### Step 3: Resolve CRITICAL Findings + LOW Dimensions (Interactive)

For each CRITICAL finding, present via AskUserQuestion:
- State what the architect found and why it's critical
- Include the evidence (file path, codebase context the architect discovered)
- Provide 2-3 lettered options with a recommendation
- One question per CRITICAL finding (not batched)

For LOW dimensions without a CRITICAL finding attached, present the uncertainty
via AskUserQuestion with options.

**Max 2 AskUserQuestion calls per gate round.** If more CRITICAL items exist than
budget allows, prioritize by impact. Default order:
Architecture > Error Handling > Test Strategy > Data Flow > Security.
Adjust based on the plan's primary concern: if security-focused (auth, encryption,
access control), promote Security to first. If data-migration-focused, promote
Data Flow. The default order applies for general feature work.

Remaining CRITICAL items → add to `## Open Questions` with the architect's evidence.

### Step 4: Re-Draft + Re-Score

After resolving CRITICAL findings (user input) and incorporating IMPORTANT/MINOR (auto):
- Re-draft only the affected sections — no full re-draft
- Re-score only the dimensions that were LOW or MEDIUM
- **No new file reads or architect spawns** — work with existing context + user answers

### Step 5: Gate Decision

```
IF all dimensions ≥ MEDIUM AND all CRITICAL findings resolved AND round ≤ 2:
    → Proceed to Phase 5 (Output)

IF any dimension still LOW AND round < 2:
    → Loop back to Step 2 (present updated confidence map)

IF round = 2 AND (any dimension still LOW OR unresolved CRITICAL findings):
    → Proceed to Phase 5 with warning:
      "⚠ Plan shipped with unresolved items: [list]"
    → Add unresolved items to ## Open Questions section in the plan
```

### All-HIGH Fast Path

If all dimensions score HIGH and there are no CRITICAL findings:
- Print the confidence map (all HIGH) for visual confirmation
- List the architect's top 3 verification actions as evidence the review was substantive.
  Pull these from the architect's return output. Examples: "Verified 8 consumers of
  FooService.process() are covered by work units", "Confirmed W1-02 has no dependency
  on W1-01 — wave ordering correct", "Checked migration backward compatibility with
  existing schema". Do not just say "architect review passed" — show the work.
- Skip interaction, proceed directly to Phase 5

---

## Phase 5: Output

### 5a. Write the Final Plan File

Re-write the plan to the output path with all incorporations from Phase 4.5.
This overwrites the draft written in Phase 3b.

### 5b. Orchestration Playbook

Generate the `## Orchestration Playbook` section in the plan file. For each wave, emit
one scoped `/orchestrate` call in execution order:

```bash
# Wave 1: [wave title from plan]
/orchestrate <plan-path> --scope "Wave 1"

# Wave 2: [wave title from plan]
/orchestrate <plan-path> --scope "Wave 2"
```

End with the single full-plan command as a convenience alternative.

Use the actual plan path and wave titles. If agent overrides are needed (e.g. for a
frontend project), include `--coder` / `--reviewer` / `--architect` flags on each line.

### 5c. Completion Summary
Print:
```
Plan written to: <path>
Mode: <ceo|eng>
Waves: N | Work units: M
Architect: <agent-type> — N findings (N critical, N important, N minor)
Secondary architect: <agent-type or "none"> — N findings (if applicable)
Missing work units added: N
Confidence: Architecture=HIGH, Errors=HIGH, Tests=MEDIUM, Data=HIGH, Security=HIGH
Gate: Passed round N (N unresolved items)

Orchestration playbook included — run wave-by-wave or all at once:
  /orchestrate <path>
```

---

## Plan File Template

```markdown
# [Plan Title]

**Created at:** `<short-commit-hash>` on `<date>` | **Mode:** `<ceo|eng>`

## Summary
[2-3 sentences: what this plan does and why]

## Existing Code Leverage
[What already exists that this plan reuses. File paths + brief descriptions.]

## Dream State *(ceo mode only)*
CURRENT STATE          THIS PLAN              12-MONTH IDEAL
[describe]      --->   [describe delta]  --->  [describe target]

## Architecture
[ASCII diagram of component relationships — new components and their relationships to existing ones]

## Data Flow *(--spec only)*
[Per-wave ASCII diagrams showing request/data flow through new and modified components.
 Include happy path and key error paths with branch points marked.]

## Blast Radius
[For each file/interface being modified: list all known callers, consumers, and dependents.
 Flag any that require changes not covered by a work unit.]

| Modified File/Interface | Consumers | Covered by Work Unit? |
|------------------------|-----------|----------------------|
| FooService.process()   | BarController, BazJob | Yes (W1-02, W2-01) |
| /api/v1/orders         | Mobile app, Partner API | W2-03 (Partner API needs versioning) |

## Wave 1: [Foundation]

### W1-01: [Work unit title]
[Description]

**Files:** path/to/file.ts (lines 10-50), path/to/new-file.ts (new)
**Acceptance criteria:**
- [Specific checkable condition]
- [Another condition]
**Error handling:** [Named failures + expected behavior]
**Error & rescue table:** *(--spec only)*
| Method/Codepath | Failure Mode | Rescue Behavior | HTTP Status |
|-----------------|-------------|-----------------|-------------|
| ...             | ...         | ...             | ...         |
**Tests:** [Type + what to test]
**Test spec:**
- [Concrete behavioral test case]
**Effort:** *(--spec only)* S|M|L

### W1-02: [Work unit title]
...

## Wave 2: [Core Logic]

### W2-01: [Work unit title]
...

## NOT in Scope
- [Deferred item] — [one-line rationale]

## Security Considerations *(--spec only)*
[Attack surface introduced by this plan. Auth boundaries, input validation points,
 injection surfaces, and trust boundaries. Map each to the work unit that addresses it.]

## Failure Modes Summary

| Codepath | Failure Mode | Handled In | Tested? |
|----------|-------------|------------|---------|
| ...      | ...         | W1-02      | Yes     |

## Architect Review Findings
[Structured findings from the independent architect review.
 What was found, what was auto-incorporated, what was resolved with user input.]

### Auto-Incorporated
- [F2: Added timeout handling to W2-01 acceptance criteria]
- [M1: Added W1-03 for database migration]

### Resolved with User Input
- [F1: Auth model clarified — using session auth per user decision]

### Deferred
- [F5: Minor — monitoring dashboard setup deferred to follow-up]

## Confidence Assessment
| Dimension | Score | Source | Notes |
|-----------|-------|--------|-------|
| Architecture | HIGH | Architect review | [brief reason] |
| Error Handling | HIGH | Architect review | [brief reason] |
| Test Strategy | MEDIUM | Architect review → auto-refined | [brief reason] |
| Data Flow | HIGH | Architect review | [brief reason] |
| Security | HIGH | Architect review | [brief reason] |

## Open Questions
[Unresolved CRITICAL findings or LOW items if gate forced output at round 2. Omit if none.]

## Orchestration Playbook
Run these commands in order to execute this plan:

```bash
# Wave 1: [Foundation]
/orchestrate plans/my-plan.md --scope "Wave 1"

# Wave 2: [Core Logic]
/orchestrate plans/my-plan.md --scope "Wave 2"

# Wave 3: [Integration]
/orchestrate plans/my-plan.md --scope "Wave 3"
```

Or execute the entire plan at once:
```bash
/orchestrate plans/my-plan.md
```
```

### --spec Additions

When `--spec` is passed, include the sections and fields marked `*(--spec only)*` in the
plan template above. These are integrated at their natural positions:
- `## Data Flow` — after Architecture (per-wave ASCII diagrams)
- `**Error & rescue table:**` — per work unit, after Error handling
- `**Effort:**` — per work unit, after Test spec (S/M/L sizing)
- `## Security Considerations` — after NOT in Scope

---

## Critical Rules

1. **Context budget.** You are the planner; the architect sub-agent is the reviewer.
   Protect your context window during Phase 2 — do not read files speculatively.
   However, thoroughness on the first pass is paramount — a second run of plan-work on
   the same prompt should NOT produce a materially better plan. Use the completeness
   sweep (2b½). The architect sub-agent has its own context and exploration budget.
2. **Independent review is non-negotiable.** Phase 4 MUST spawn an architect sub-agent.
   Do not skip this step, even for small plans. Do not substitute self-review for the
   architect review. The entire point is that a fresh perspective catches what you miss.
3. **Orchestrate compatibility.** Wave headings MUST be `## Wave N: [Title]`. Work units MUST
   be `### W{N}-{NN}: [Title]`. This is non-negotiable — orchestrate parses this structure.
4. **One issue = one AskUserQuestion.** Never batch. Lead with recommendation + WHY.
   Lettered options. Map to constraints from Phase 1.
5. **No code.** Do not write implementation code. Do not create test files. Plan only.
6. **Engineering preferences.** DRY, well-tested, engineered-enough (not over/under),
   handle edge cases, explicit over clever, minimal diff, ASCII diagrams for complex flows.
7. **Confidence scores come from the architect.** Do not override or inflate the architect's
   scores. If you disagree with a score, note it in the refinement log but use the
   architect's score for the gate decision. LOW means the architect found genuine gaps.
8. **Architect prompt must be self-contained.** The sub-agent has no memory of your context.
   Include the plan file path, the original user description, the discovery brief (if
   Phase 0 ran), the review checklist, and the exploration budget. Do not assume it knows
   anything.
9. **Gate question budget.** Max 2 AskUserQuestion calls per gate round, max 2 rounds.
   Prioritize by impact if more CRITICAL findings than budget allows.
10. **Phase 0 discipline.** Discovery questions must target plan-forking decisions only.
    If the answer wouldn't change the plan's structure (waves, work units, architecture),
    don't ask. Explore the codebase first — never ask what the code can tell you.
11. **Write draft before review.** Phase 3b writes the draft plan to disk so the architect
    sub-agent can read it. Phase 5a overwrites it with the final version after incorporating
    findings.
12. **Blast radius is mandatory.** Every plan must include a Blast Radius section mapping
    modified files/interfaces to their consumers. The architect verifies this independently.
13. **Test specs require concrete input/output pairs.** Test specs MUST include specific
    inputs and expected outputs, not just behavioral descriptions. BAD: "test that validation
    works". GOOD: "input: {amount: -1}, expected: 400 with {error: \"amount must be positive\"}".
    The coder writes these as failing tests FIRST — vague specs produce vague tests.
    Include at least one happy path and one rejection case per work unit.
14. **CLAUDE.md constraints flow into acceptance criteria.** Extract hard constraints from
    CLAUDE.md (e.g., dependency policy, security posture, naming conventions) and thread
    them into relevant work unit acceptance criteria. If CLAUDE.md says "zero unnecessary
    dependencies", any work unit adding a dependency must include a criterion justifying it.
15. **Plan size guardrails.** CEO: >4 waves or >15 work units triggers a size check.
    Eng: >3 waves or >10 work units. Ask the user to split, cut to MVP, or confirm scope
    before proceeding to architect review. Trim before review, not after.
16. **Cross-stack plans get secondary review.** If Phase 1c detects a secondary stack AND
    the plan touches files in both stacks, spawn a secondary architect with a reduced
    checklist (items 1-3, 2 scores). Merge findings conservatively — use the lower score
    when both architects assess the same dimension. The secondary review is scoped to its
    stack's files only and does not duplicate the primary review.

## Mode Quick Reference

```
              CEO                          ENG
Discovery     Phase 0 if brief input       Phase 0 if brief input
Scope Q       Q1-Q3 (+ ambition)          Q1-Q2 only (Q1 skipped if Phase 0 ran)
Exploration   12 files + sweep + taste     8 files + sweep, no calibration
Premise       Full (right problem?)         Complexity check only
Draft         Dream state + phase 2 ideas  Complexity justification
Size check    >4 waves or >15 units        >3 waves or >10 units
Architect     6-item checklist, 5 scores   Items 1-4 only, 3 scores
2nd architect If cross-stack (items 1-3)   If cross-stack (items 1-3)
Gate          5 dimensions, max 2 rounds   3 dimensions, max 2 rounds
Output        Rich, forward-looking        Lean, execution-focused
```

## AskUserQuestion Budget

```
Phase 0:   0-6 questions  (requirements discovery — conditional, skipped if input is detailed)
Phase 1:   1-4 questions  (scoping — Q1 skipped if Phase 0 ran, Q4 skipped if Phase 0 ran)
Phase 2:   0              (read-only, no interaction)
Phase 3:   0-2 questions  (drafting — output path conflict + size check, both conditional)
Phase 4:   0              (architect sub-agent — runs independently, no user interaction)
Phase 4.5: 0-4 questions  (confidence gate — 0-2 per round × max 2 rounds)
Phase 5:   0              (output, no interaction)

Typical run (with Phase 0):   7-9 questions total
Typical run (without Phase 0): 4-6 questions total
Worst case:  13 questions (brief input + complex CEO plan + 2 full gate rounds)
Best case:    2 questions (detailed input, clean eng fix, architect finds no critical issues)
```
