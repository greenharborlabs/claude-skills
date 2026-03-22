---
name: next-phases
description: "Full-repository audit producing a next-phase roadmap. Auto-detects tech stack (Java/Spring, NanoClaw, generic) and routes to the appropriate architect agent. This skill should be used when the user wants to analyze a codebase and generate a prioritized roadmap of what to build or fix next, or says 'next-phases'. It runs a 6-stage analysis (inventory, architecture map, spec alignment, health assessment, roadmap, open questions) and writes the report to a file. Supports --deep-review for Java projects."
---

# Next Phases

## Overview

Perform a comprehensive codebase and spec analysis to produce a next-phase
roadmap grounded entirely in repo evidence. Runs a 6-stage audit: repo
inventory, architecture map, spec-to-implementation alignment, codebase health
assessment, prioritized roadmap, and open questions.

## Help

If the argument is `--help`, `help`, or no arguments are provided, print this
usage summary and stop (do not run the workflow):

```
/next-phases - Full-repository audit with next-phase roadmap

Usage:
  /next-phases                                       Audit current repo
  /next-phases path/to/repo                          Audit a specific repo
  /next-phases --out plans/next-phases.md            Override output file
  /next-phases --stack auto|java|generic             Override stack detection (default: auto)
  /next-phases --agent nanoclaw-architect             Override agent type
  /next-phases --skip-interview                      Skip knobs interview, use defaults
  /next-phases --deep-review                         (Java only) Parallel backend-reviewer-java for Stage D

Arguments:
  repo-path       Path to repository root (default: current working directory)
  --out           Output file path (default: plans/next-phases.md)
  --stack         Stack detection: auto (default), java, generic
  --agent         Agent type for analysis (overrides auto-detection)
  --skip-interview  Skip the project knobs interview, use "not specified" for all
  --deep-review   (Java only) Spawn a parallel backend-reviewer-java agent for Stage D

Examples:
  /next-phases
  /next-phases ~/code/other-project
  /next-phases --out docs/audit-report.md
  /next-phases --stack java --deep-review
  /next-phases --agent backend-planning-architect
```

## Arguments

Parse the invocation arguments as follows:

- **Argument 1 (optional):** Path to repository root. Default: current working directory
- **`--out` (optional):** Output file path. Default: `plans/next-phases.md`
- **`--stack` (optional):** `auto` (default), `java`, or `generic`. Auto-detection uses file extensions: `*.java`/`*.gradle`/`pom.xml` → java, otherwise generic.
- **`--agent` (optional):** Agent type for the analysis. Overrides auto-detection. Default: auto-detected based on stack.
- **`--skip-interview` (optional):** Skip the interactive knobs interview
- **`--deep-review` (optional):** (Java stack only) Spawn a parallel `backend-reviewer-java` agent for Stage D

## Workflow

### Step 0: Project Knobs Interview

Unless `--skip-interview` is passed, ask the user these questions to scope the
audit. Accept brief answers — these inform priority weighting in the roadmap.

Questions to ask (use AskUserQuestion, up to 4 per call):

1. **Primary goal:** What matters most right now? (speed / correctness / scale / cost)
2. **Known pain points:** What are the top 2-3 things bothering you about this codebase?
3. **Non-negotiables:** Any hard constraints? (security, compliance, tech stack locks)
4. **Risk tolerance:** How aggressive should recommendations be? (conservative / moderate / aggressive)

If the user skips or says "just run it", proceed with all knobs as "not specified".

Record the answers as a knobs block at the top of the output file.

### Step 0.5: Resolve Stack and Agent

If `--agent` was provided, use that directly. Otherwise:

1. If `--stack` was provided, use it. If `--stack auto` or unspecified:
   - Glob for `*.java`, `*.gradle`, `pom.xml` in the repo root → `java`
   - Otherwise → `generic`

2. Map stack to agent:
   - `java` → `backend-planning-architect`
   - `generic` → `nanoclaw-architect`

3. If `--deep-review` and stack is not `java`, warn: "—deep-review only applies to Java projects, ignoring."

Print: `Stack: <stack> | Architect: <agent>`

### Step 1: Spawn Architect Agent

Spawn the resolved architect agent (or the `--agent` override) sub-agent with the full
analysis prompt below. Pass it:

- The repository path
- The knobs from Step 0
- The detected stack (so the agent can apply stack-specific analysis)
- Instructions to write output using the exact section headings defined below

### Analysis Stages (executed by the agent)

The agent must complete each stage in order, producing intermediate artifacts
before the final roadmap.

#### Stage A — Repo and Spec Discovery

Scan the repo top-down and produce a categorized inventory:

| Category | What to look for |
|---|---|
| Services / Apps | Entrypoints, main modules, service boundaries |
| Libraries / Packages | Shared code, internal packages, SDKs |
| Build and Tooling | Build scripts, linters, formatters, monorepo tooling |
| CI/CD | Pipeline definitions, deploy scripts, IaC |
| Infrastructure | Docker/Compose, K8s manifests, Terraform, cloud config |
| Specs and Design Docs | /docs, README, ADRs, RFCs, specs, changelogs |
| Config and Secrets Mgmt | Env files, vault refs, config loaders |

For each item: one-line description + file path(s).

**Java addendum (when stack=java):** Also look for:
- Spring Boot applications, `@SpringBootApplication` classes
- Multi-module Gradle/Maven structure
- Flyway/Liquibase migrations, JPA entities, repository interfaces
- `application.yml`/`application.properties` config files

#### Stage B — Architecture and Component Map

1. Produce a layered component map in text form showing modules/services, their
   responsibilities, and directional dependencies
2. Identify integration points and for each note:
   - Type (REST API, gRPC, queue/pubsub, DB, external SaaS, file I/O)
   - Location in repo (path)
   - Direction (inbound / outbound / bidirectional)

**Java addendum (when stack=java):** Also identify Spring-specific patterns:
- Controller → Service → Repository layering
- Auto-configuration classes and conditional beans
- Security filter chains
- Event/messaging patterns (`ApplicationEvent`, `@KafkaListener`, etc.)

#### Stage C — Spec-to-Implementation Alignment

For every major requirement found in spec/design docs:

| Spec Requirement | Source (file:section) | Status | Evidence / Gap Description |
|---|---|---|---|
| ... | ... | Implemented / Partial / Missing | file path or description of what is absent |

If no spec docs exist, state that explicitly and skip to Stage D.

#### Stage D — Codebase Health Assessment

Assess each dimension below. For every finding, cite at least one file path.

1. **Maintainability and Modularity** — Dependency boundaries, coupling, naming/structure consistency, code duplication signals
2. **Testing** — Strategy (unit / integration / e2e / none), coverage signals, test locations, notable gaps
3. **Security** — AuthN/AuthZ approach, input validation, secrets handling, dependency risk, attack surface
4. **Performance and Reliability** — Caching, async patterns, DB access patterns, error handling, retry/backoff, observability
5. **Deployment and Ops** — Config management, environment separation, migration strategy, runbooks, rollback capability
6. **Developer Experience** — Onboarding friction, local dev setup, documentation quality, contribution guidelines

**Java addendum (when stack=java):** Apply Java/Spring-specific depth:
- Testing: JUnit 5, `@SpringBootTest`, test slices (`@WebMvcTest`, `@DataJpaTest`)
- Security: Spring Security configuration, `@Valid` validators, OWASP concerns
- Performance: JPA fetch strategies (N+1 detection), `@Cacheable`, `@Async`, `@Transactional` boundaries
- Ops: Actuator health endpoints, Micrometer metrics, tracing

If `--deep-review` was specified, also spawn a **backend-reviewer-java** agent
in parallel (using `run_in_background: true`) with instructions to review the
entire codebase for production readiness (transaction hazards, N+1 queries,
security gaps, Spring anti-patterns). Merge its findings into this section,
deduplicating and noting the source.

#### Stage E — Next-Phase Roadmap

Generate a prioritized list of recommendations. Each item must include ALL fields:

| Field | Description |
|---|---|
| ID | R-001, R-002, ... |
| Priority | P0 (do now) / P1 (next sprint) / P2 (backlog) |
| Category | Feature / Tech Debt / Architecture / Security / DX / Observability |
| Title | Short descriptive name |
| Rationale | Why this matters — cite repo/spec evidence (paths) |
| Impact | Who benefits and how (user / business / developer) |
| Effort | S (less than 1 day) / M (1-5 days) / L (1-3 weeks) / XL (more than 3 weeks) |
| Risk | Low / Med / High + brief explanation |
| Dependencies | Other R-IDs or external blockers |
| Implementation Outline | Where to change/add code (directories, modules, files). Key steps. When stack=java, reference which agent type (backend-coder-java, backend-reviewer-java) is best suited. |
| Acceptance Criteria | Testable conditions that define done |

Sort by Priority (P0 first), then by Impact descending within each tier.

#### Stage F — Open Questions and Missing Artifacts

List anything needed before the roadmap can be finalized:

- Missing specs or ambiguous requirements
- Unknown constraints (infra, compliance, team capacity)
- Artifacts expected but not found (missing ADRs, absent test suites, no CI config)
- **Java addendum:** Spring Boot / Java version upgrade considerations

### Step 2: Write Output

The agent writes the report to the output file path using these exact headings:

```
## 0. Project Knobs
(answers from the interview, or "not specified" for each)

## 1. Repo Inventory
(categorized bullets with paths per Stage A)

## 2. Architecture Map
(text component diagram + integration points per Stage B)

## 3. Spec Coverage Matrix
(markdown table per Stage C)

## 4. Health Findings
(grouped by dimension per Stage D, each finding with file-path evidence)

## 5. Next-Phase Roadmap
(markdown table per Stage E, sorted by priority)

## 6. Open Questions
(bullets per Stage F)
```

### Step 3: Summary

After the agent completes, print a brief summary:
- Output file path
- Count of P0 / P1 / P2 recommendations
- Top 3 P0 items by title
- Any critical open questions

## Operating Rules for the Agent

These rules must be included when spawning the architect agent:

1. **Evidence-first**: Every non-trivial claim must cite a file path and, where
   helpful, a symbol (module, class, function). If evidence is absent, write
   "Not found in repo" and move on.
2. **Read-before-recommend**: Before proposing changes to an area, first locate
   and briefly summarize the relevant files/docs. Do not skip this step.
3. **Stage gates**: Complete each stage in order. Output intermediate artifacts
   before the final roadmap — do not jump ahead.
4. **No large code dumps**: Use short references (paths + symbols) and inline
   snippets only when essential (10 lines or fewer).
5. **Stick to the output format**: Use the exact section headings defined above.
6. **Stack-aware analysis**: When stack=java, apply Java/Spring Boot expertise —
   look for Spring anti-patterns, JPA pitfalls, transaction hazards, and bean
   lifecycle issues that generic analysis would miss. In implementation outlines,
   specify which Java agent (backend-coder-java, backend-reviewer-java,
   backend-debugger-java) is best suited for each recommended task.
