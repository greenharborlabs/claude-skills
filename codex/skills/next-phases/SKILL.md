---
name: next-phases
description: Full-repository audit producing a next-phase roadmap. Use when the user wants to analyze a codebase and generate a prioritized roadmap of what to build or fix next, or says "next-phases". Supports Java/Spring and generic repo analysis.
---

# Next Phases

Perform a comprehensive codebase and spec analysis to produce a next-phase roadmap grounded in repository evidence.

## Help

If the argument is `--help`, `help`, or no arguments are provided, show:

```text
next-phases - Full-repository audit with next-phase roadmap

Usage:
  next-phases
  next-phases path/to/repo
  next-phases --out plans/next-phases.md
  next-phases --stack auto|java|generic
  next-phases --skip-interview
  next-phases --deep-review
```

## Arguments

- `repo-path`: default current working directory.
- `--out PATH`: default `plans/next-phases.md`.
- `--stack auto|java|generic`: default `auto`; Java if `*.java`, `*.gradle`, or `pom.xml` are present.
- `--skip-interview`: skip project knobs interview.
- `--deep-review`: for Java repos, include a deeper production-readiness review pass.

## Workflow

1. Parse arguments and resolve repo path.
2. Unless `--skip-interview` is set, ask project knobs:
   - Primary goal: speed, correctness, scale, or cost.
   - Known pain points.
   - Non-negotiables.
   - Risk tolerance.
3. Detect stack.
4. Run the audit directly in stages. If the user explicitly requests parallel agent work, Codex explorer agents may be used for independent stages; otherwise do the work in the main session.
5. Write output to the requested path using the exact section headings below.

## Audit Stages

### Stage A - Repo and Spec Discovery

Inventory services/apps, libraries/packages, build and tooling, CI/CD, infrastructure, specs/design docs, config and secrets management.

Java addendum: identify Spring Boot apps, Gradle/Maven structure, migrations, JPA entities, repositories, and `application.yml` / `application.properties`.

### Stage B - Architecture and Component Map

Produce a layered component map, responsibilities, directional dependencies, and integration points.

Java addendum: identify controller/service/repository layering, auto-config, security filters, and event/messaging patterns.

### Stage C - Spec-to-Implementation Alignment

For each major spec/design requirement found, classify status as Implemented, Partial, Missing, or Not found in repo.

### Stage D - Codebase Health Assessment

Assess maintainability, testing, security, performance/reliability, deployment/ops, and developer experience. For Java, include Spring Security, validation, JPA, transaction, actuator, and Micrometer concerns.

If `--deep-review` is set, perform an additional review pass focused on transaction hazards, N+1 queries, security gaps, and Spring anti-patterns. Merge and deduplicate findings.

### Stage E - Next-Phase Roadmap

Create recommendations with ID, priority, category, title, rationale, impact, effort, risk, dependencies, implementation outline, and acceptance criteria.

### Stage F - Open Questions and Missing Artifacts

List missing specs, unknown constraints, absent artifacts, and Java upgrade considerations where relevant.

## Output Format

```markdown
## 0. Project Knobs

## 1. Repo Inventory

## 2. Architecture Map

## 3. Spec Coverage Matrix

## 4. Health Findings

## 5. Next-Phase Roadmap

## 6. Open Questions
```

## Standards

- Every non-trivial claim cites a path and, when useful, a symbol.
- If evidence is absent, say `Not found in repo`.
- Do not recommend changes to an area before locating and summarizing relevant files.
- Keep snippets short and only when essential.
