---
name: deep-interview
description: "Exhaustive requirements interview for planning. This skill should be used when the user wants to be deeply interviewed about a feature, system, or design before implementation begins, or says 'deep-interview'. It conducts multi-round AskUserQuestion interviews covering technical implementation, UI/UX, edge cases, failure modes, tradeoffs, security, performance, and architecture until no ambiguity remains, then writes a structured spec."
---

# Deep Interview

## Overview

Conduct an exhaustive, multi-round interview about a topic before planning or
implementation begins. Use AskUserQuestion to probe deeply into technical
implementation details, UI/UX decisions, edge cases, failure modes, tradeoffs,
security implications, performance concerns, and architectural choices. Continue
interviewing until all dimensions are thoroughly covered, then compile answers
into a structured specification file.

## Help

If the argument is `--help`, `help`, or no arguments are provided, print this
usage summary and stop (do not run the workflow):

```
/deep-interview - Exhaustive requirements interview for planning

Usage:
  /deep-interview "feature or topic description"     Start interview about a topic
  /deep-interview --out plans/spec.md                Override output file
  /deep-interview --resume                           Resume a previously interrupted interview

Arguments:
  topic         Description of what to interview about (required unless --resume)
  --out         Output file for compiled spec (default: plans/interview-spec.md)
  --resume      Resume from the output file if a partial interview exists

Examples:
  /deep-interview "WebSocket real-time price feed for the trading dashboard"
  /deep-interview "agent crash recovery and state reconciliation" --out plans/crash-recovery-spec.md
  /deep-interview --resume --out plans/crash-recovery-spec.md
```

## Arguments

Parse the invocation arguments as follows:

- **Argument 1 (required unless --resume):** The topic or feature to interview about
- **`--out` (optional):** Output file path. Default: `plans/interview-spec.md`
- **`--resume` (optional):** Resume from existing partial output file

## Critical Rules

1. **No obvious questions.** Never ask things the user would expect a senior
   engineer to already know or figure out from the codebase. Every question
   must surface a decision, tradeoff, constraint, or preference that cannot be
   determined from code alone.

2. **Go deep, not wide.** When the user answers, follow up on implications of
   that answer before moving to the next dimension. A shallow answer is a
   signal to probe further, not move on.

3. **Use AskUserQuestion exclusively.** All questions must go through the
   AskUserQuestion tool with well-crafted options. Provide 2-4 concrete options
   per question (the user can always pick "Other"). Never ask open-ended
   questions in plain text.

4. **Continue until complete.** Do not stop after a fixed number of rounds.
   Keep interviewing until all dimensions listed below have been adequately
   covered for the given topic. Track coverage internally.

5. **Read relevant code first.** Before each interview round, read any files
   related to the topic to make questions grounded in the actual codebase.
   Reference specific code in question descriptions when relevant.

## Interview Dimensions

Cover ALL of these dimensions that are relevant to the topic. Skip a dimension
only if it genuinely does not apply (e.g., skip UI/UX for a pure backend
library). Within each dimension, ask about the non-obvious aspects.

### 1. Core Behavior and Semantics
- What exactly happens and in what order
- Invariants that must always hold
- What the system promises vs what is best-effort
- Idempotency requirements
- State machine transitions and valid/invalid states

### 2. Failure Modes and Error Handling
- What happens when each dependency fails (DB down, API timeout, network partition)
- Partial failure scenarios (half-completed operations)
- Retry semantics (at-most-once, at-least-once, exactly-once)
- Poison message / bad input handling
- Graceful degradation vs hard failure preferences
- Recovery procedures (manual vs automatic)

### 3. Concurrency and Race Conditions
- What happens under concurrent access
- Locking strategy preferences
- Ordering guarantees needed
- Eventual consistency tolerance
- TOCTOU risks and mitigation preferences

### 4. Security and Trust Boundaries
- Who/what is trusted vs untrusted
- Input validation depth required
- AuthZ model for the feature
- Data sensitivity classification
- Audit trail requirements
- Rate limiting and abuse prevention

### 5. Performance and Scale
- Expected load (requests/sec, data volume, concurrent users)
- Latency requirements (p50, p99)
- Acceptable resource consumption
- Caching strategy and invalidation
- Pagination and streaming decisions
- Growth trajectory and when to re-architect

### 6. Data and State Management
- Source of truth for each piece of state
- Consistency requirements (strong vs eventual)
- Data lifecycle (creation, mutation, archival, deletion)
- Migration strategy for schema changes
- Backup and recovery expectations

### 7. Integration and API Contracts
- Upstream/downstream dependencies
- Contract versioning approach
- Breaking change policy
- Timeout and circuit breaker thresholds
- Retry budgets with external services

### 8. UX and User-Facing Behavior
- Loading states, empty states, error states
- Optimistic vs pessimistic updates
- Undo/redo expectations
- Notification preferences (in-app, email, push)
- Accessibility requirements

### 9. Observability and Operations
- What should be logged, metered, traced
- Alerting thresholds
- Runbook expectations
- Feature flag / gradual rollout needs
- On-call implications

### 10. Tradeoffs and Constraints
- Time vs quality preference for this feature
- Build vs buy decisions
- Technical debt tolerance
- Backward compatibility requirements
- Migration path from current state

## Workflow

### Step 1: Context Gathering

Read the codebase to understand:
- How the topic relates to existing code
- What patterns are already established
- What decisions have already been made that constrain this topic

This informs which questions to ask and which to skip.

### Step 2: Interview Rounds

Conduct interview rounds using AskUserQuestion. Each round:

1. Ask up to 4 questions per AskUserQuestion call (the tool's limit)
2. After each answer set, analyze implications and determine follow-up questions
3. Track which dimensions have been covered and which need more depth
4. Reference the user's previous answers to make follow-ups contextual
5. If an answer reveals a new concern, pursue it immediately before returning
   to the planned dimension

Continue rounds until all relevant dimensions are covered to a depth where
implementation decisions can be made unambiguously.

### Step 3: Compile Specification

After the interview is complete, write a structured specification to the output
file with these sections:

```
# [Topic] Specification

## Summary
(2-3 sentence overview of what was decided)

## Decisions Log
(Every decision made during the interview, numbered, with rationale)

| # | Decision | Rationale | Dimension |
|---|----------|-----------|-----------|
| 1 | ... | ... | Core Behavior |

## Requirements
(Derived from interview answers, grouped by dimension)

### Functional Requirements
(What the system must do)

### Non-Functional Requirements
(Performance, security, observability constraints)

### Constraints
(Hard limits and non-negotiables identified during interview)

## Open Items
(Anything that was deferred or needs external input)

## Interview Transcript Summary
(Condensed Q&A log for future reference)
```

### Step 4: Summary

Print a brief summary:
- Output file path
- Count of decisions made
- Count of open items remaining
- Dimensions covered
