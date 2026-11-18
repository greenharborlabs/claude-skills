---
name: greenharbor-deep-interview
description: Exhaustive requirements interview for planning. Use when the user wants to be deeply interviewed about a feature, system, or design before implementation begins, or says "greenharbor-deep-interview". Produces a structured specification after resolving behavior, edge cases, tradeoffs, security, performance, operations, and architecture.
---

# Deep Interview

Conduct a multi-round requirements interview before planning or implementation. The goal is a decision-complete specification, not a shallow feature summary.

## Help

If the argument is `--help`, `help`, or no arguments are provided, show:

```text
greenharbor-deep-interview - Exhaustive requirements interview for planning

Usage:
  greenharbor-deep-interview "feature or topic description"
  greenharbor-deep-interview "feature or topic description" --out plans/spec.md
  greenharbor-deep-interview --resume --out plans/spec.md
```

## Arguments

- Topic: required unless `--resume` is provided.
- `--out PATH`: output file, default `plans/<topic-slug>-spec.md`.
- `--resume`: resume the in-progress checkpoint in the output file.

If the output file exists and `--resume` was not given, ask whether to resume,
overwrite, or choose a new path before gathering context.

## Critical Rules

- Explore relevant code before asking questions when a repo is available.
- Do not ask questions answerable from the codebase or local files.
- Prefer `request_user_input` when available. If unavailable, ask concise direct questions.
- Ask only questions that change the spec, lock an assumption, or choose between real tradeoffs.
- Continue until the relevant dimensions are covered deeply enough for implementation. The user owns the stopping point: after every third round, ask whether to continue with the named uncovered dimensions or write the spec now.
- Gather codebase context once up front (about 5 searches and 8 targeted reads); read more only when an answer points at unseen code.
- Checkpoint every round: write the output file with `**Status:** in progress`, decisions so far, the coverage table, and an `## Open Threads` section.

## Interview Dimensions

Cover dimensions that apply to the topic:

- Core behavior and semantics: ordering, invariants, promises, idempotency, state transitions.
- Failure modes and error handling: dependency failures, partial completion, retry semantics, recovery.
- Concurrency: simultaneous access, locking, ordering guarantees, consistency tolerance.
- Security: trust boundaries, validation, authZ, data sensitivity, auditability, abuse prevention.
- Performance: load, latency, resource use, caching, streaming, growth.
- Data and state: source of truth, lifecycle, migrations, retention, recovery.
- Integrations and APIs: upstream/downstream contracts, versioning, timeouts, circuit breakers.
- UX: loading, empty, error, undo, notifications, accessibility.
- Observability and operations: logs, metrics, traces, alerts, runbooks, rollout.
- Tradeoffs and constraints: schedule, quality bar, technical debt, compatibility, build-vs-buy.

## Workflow

1. Parse the topic, output path, and resume flag. On `--resume`, restore decisions, coverage, and open threads from the checkpoint and skip settled questions.
2. Gather codebase context relevant to the topic.
3. Interview in rounds. Each round should ask a small set of high-impact questions and then adapt follow-ups to the answers.
4. Track decisions and open items as the interview progresses.
5. Write the final specification over the checkpoint with `**Status:** complete`.

## Specification Format

```markdown
# [Topic] Specification

**Status:** complete | in progress
**Updated:** <YYYY-MM-DD>

## Summary

## Decisions Log

| # | Decision | Rationale | Dimension |
|---|----------|-----------|-----------|

## Requirements

### Functional Requirements

### Non-Functional Requirements

### Constraints

## Coverage

| Dimension | Status | Notes |
|-----------|--------|-------|

## Open Items

## Interview Transcript Summary
```

After writing the spec, summarize the output path, decision count, open item count, and dimensions covered, and suggest `greenharbor-plan-work "Implement <topic> per the spec in <output path>" --spec --no-interview`.
