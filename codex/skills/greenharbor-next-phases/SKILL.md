---
name: greenharbor-next-phases
description: Full-repository audit producing a next-phase roadmap. Use when the user wants to analyze a codebase and generate a prioritized roadmap of what to build or fix next, or says "greenharbor-next-phases". Supports Java/Spring, Rust, and generic repo analysis.
---

# Next Phases

Perform a comprehensive codebase and spec analysis to produce a next-phase roadmap grounded in repository evidence.

## Help

If the argument is `--help`, `help`, or no arguments are provided, show:

```text
greenharbor-next-phases - Full-repository audit with next-phase roadmap

Usage:
  greenharbor-next-phases
  greenharbor-next-phases path/to/repo
  greenharbor-next-phases --out plans/greenharbor-next-phases.md
  greenharbor-next-phases --stack auto|java|rust|generic
  greenharbor-next-phases --skip-interview
  greenharbor-next-phases --deep-review
```

## Arguments

- `repo-path`: default current working directory.
- `--out PATH`: default `plans/greenharbor-next-phases.md`.
- `--stack auto|java|rust|generic`: default `auto`; Java if `*.java`, `*.gradle`, or `pom.xml` are present; Rust if `Cargo.toml` and `*.rs` are present.
- `--skip-interview`: skip project knobs interview.
- `--deep-review`: for Java or Rust repos, include a deeper production-readiness review pass.

## Workflow

1. Parse arguments and resolve repo path.
2. Unless `--skip-interview` is set, ask project knobs:
   - Primary goal: speed, correctness, scale, or cost.
   - Known pain points.
   - Non-negotiables.
   - Risk tolerance.
3. Detect stack.
4. Apply the matching role overlay, then run the audit stages. If the user explicitly requests parallel agent work, Codex explorer agents may be used for independent stages; otherwise do the work in the main session.
5. Write output to the requested path using the exact section headings below.

## Role Overlays

- For Java, if `$greenharbor-backend-planning-architect` is available, read and apply its discovery, decision, Java/Spring, and rollout guidance while preserving this skill's roadmap output.
- For Java with `--deep-review`, if `$greenharbor-backend-reviewer-java` is available, apply its evidence, risk, severity, and verification rules to Stage D, then merge and deduplicate its findings.
- If either Java skill is unavailable, use the Java addenda below; do not invent or emit an unresolved skill name.
- Rust and generic audits use their built-in addenda unless an exact matching skill is available.

## Audit Stages

### Stage A - Repo and Spec Discovery

Inventory services/apps, libraries/packages, build and tooling, CI/CD, infrastructure, specs/design docs, config and secrets management.

Java addendum: identify Spring Boot apps, Gradle/Maven structure, migrations, JPA entities, repositories, and `application.yml` / `application.properties`.

Rust addendum: identify Cargo workspaces, package manifests/lockfiles, toolchain/MSRV/edition, features, targets, build scripts, public crate roots, unsafe/FFI boundaries, and async runtimes.

### Stage B - Architecture and Component Map

Produce a layered component map, responsibilities, directional dependencies, and integration points.

Java addendum: identify controller/service/repository layering, auto-config, security filters, and event/messaging patterns.

Rust addendum: identify crate/module boundaries, public APIs, ownership/error boundaries, task/cancellation/shutdown ownership, safe abstractions around unsafe/FFI, and external I/O boundaries.

### Stage C - Spec-to-Implementation Alignment

For each major spec/design requirement found, classify status as Implemented, Partial, Missing, or Not found in repo.

### Stage D - Codebase Health Assessment

Assess maintainability, testing, security, performance/reliability, deployment/ops, and developer experience. For Java, include Spring Security, validation, JPA, transaction, actuator, and Micrometer concerns.

For Rust, include unsafe/FFI soundness, Cargo/RustSec supply-chain posture, public API/SemVer/MSRV/features, serde/CLI/FFI contracts, async cancellation/backpressure/locking, and bounded resource use.

If `--deep-review` is set, perform the matching stack-specific production-readiness pass and merge/deduplicate findings. For Rust, focus on soundness, async/resource hazards, security, public contracts, and supply-chain risk.

### Stage E - Next-Phase Roadmap

Create recommendations with ID, priority, category, title, rationale, impact, effort, risk, dependencies, implementation outline, and acceptance criteria.

### Stage F - Open Questions and Missing Artifacts

List missing specs, unknown constraints, absent artifacts, and Java or Rust toolchain/MSRV/edition upgrade considerations where relevant.

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
