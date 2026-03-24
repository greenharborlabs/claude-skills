---
name: next-phases
version: 1.0.0
description: |
  Full-repository audit producing a next-phase roadmap. Auto-detects tech stack
  (Java/Spring, React/TS, NanoClaw, OpenClaw, generic) and routes to the appropriate
  architect agent. This skill should be used when the user wants to analyze a codebase
  and generate a prioritized roadmap of what to build or fix next, or says 'next-phases'.
  It runs a 7-stage analysis (inventory, repo dynamics, architecture map, spec alignment,
  health assessment, roadmap, open questions) and writes the report to a file.
  Supports --deep-review for Java projects.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Write
  - Agent
  - AskUserQuestion
---

# Next Phases

## Overview

Perform a comprehensive codebase and spec analysis to produce a next-phase
roadmap grounded entirely in repo evidence. Runs a 7-stage audit: repo
inventory, repository dynamics (git history analysis), architecture map,
spec-to-implementation alignment, codebase health assessment, prioritized
roadmap, and open questions.

## Help

If the argument is `--help`, `help`, or no arguments are provided, print this
usage summary and stop (do not run the workflow):

```
/next-phases - Full-repository audit with next-phase roadmap

Usage:
  /next-phases                                       Audit current repo
  /next-phases path/to/repo                          Audit a specific repo
  /next-phases --out plans/next-phases.md            Override output file
  /next-phases --stack auto|java|react|nanoclaw|openclaw|generic
  /next-phases --agent backend-planning-architect    Override agent type
  /next-phases --skip-interview                      Skip knobs interview, use defaults
  /next-phases --deep-review                         (Java only) Parallel backend-reviewer-java for Stage D

Arguments:
  repo-path       Path to repository root (default: current working directory)
  --out           Output file path (default: plans/next-phases.md)
  --stack         Stack detection: auto (default), java, react, nanoclaw, openclaw, generic
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
- **`--stack` (optional):** `auto` (default), `java`, `react`, `nanoclaw`, `openclaw`, or `generic`. See Step 0.5 for auto-detection logic.
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

1. If `--stack` was provided, use it. If `--stack auto` or unspecified, detect by
   probing for marker files in the repo (check in this order, first match wins):

   | Probe | Stack |
   |---|---|
   | `*.java`, `*.gradle`, `pom.xml` anywhere in repo | `java` |
   | `nanoclaw.json` or `NANOCLAW.md` in repo root | `nanoclaw` |
   | `openclaw.json` or `OPENCLAW.md` in repo root | `openclaw` |
   | `package.json` exists AND (`*.tsx` files or `next.config.*` or `vite.config.*`) | `react` |
   | Everything else | `generic` |

2. Map stack to agent:

   | Stack | Agent |
   |---|---|
   | `java` | `backend-planning-architect` |
   | `react` | `frontend-architect` |
   | `nanoclaw` | `nanoclaw-architect` |
   | `openclaw` | `openclaw-architect` |
   | `generic` | `backend-planning-architect` |

3. If `--deep-review` and stack is not `java`, warn: "--deep-review only applies to Java projects, ignoring."

Print: `Stack: <stack> | Architect: <agent>`

### Step 0.75: Ensure Output Directory

Before spawning any agents, ensure the parent directory of the output file exists.
Run `mkdir -p` on the parent directory via Bash. Do not rely on the sub-agent to
create it.

### Step 1: Spawn Architect Agent

Spawn the resolved architect agent using the Agent tool. You MUST construct the
prompt using the template in the "Agent Prompt Template" section below — do not
pass these skill instructions verbatim.

If `--deep-review` was specified (and stack is `java`), ALSO spawn a
`backend-reviewer-java` agent **in parallel** using `run_in_background: true`.
See "Deep Review Agent Prompt" section for its prompt.

### Step 2: Merge Deep Review (conditional)

If `--deep-review` was used, wait for both agents to complete. Then:

1. Read the architect agent's output file
2. Read the deep-review agent's findings (returned in its result)
3. Append any new findings from the deep-review into Section 4 (Health Findings)
   under a subsection `### 4.7 Deep Review Findings (backend-reviewer-java)`
4. Deduplicate: if the deep reviewer found something the architect already noted,
   keep the more detailed version and add "(confirmed by deep review)" to it
5. If the deep reviewer surfaced new roadmap items, append them to Section 5 with
   IDs continuing from the architect's last R-ID

If `--deep-review` was NOT used, skip this step entirely.

### Step 3: Write Output

Verify the output file was written by the agent. Read it back and confirm it
contains all required section headings (0 through 6). If any section is missing,
log a warning.

### Step 4: Summary

After everything completes, print a brief summary:
- Output file path
- Count of P0 / P1 / P2 recommendations
- Top 3 P0 items by title
- Any critical open questions

---

## Agent Prompt Template

When spawning the architect agent in Step 1, construct the prompt by interpolating
the variables below into this template. Pass this as the `prompt` parameter to the
Agent tool.

**Variables:**
- `{repo_path}` — Absolute path to the repository root
- `{stack}` — Detected stack (java, react, nanoclaw, openclaw, generic)
- `{knobs}` — Formatted knobs from the interview (or "all not specified")
- `{output_path}` — Absolute path to the output file

**Template:**

```
You are performing a forensic repository audit — not designing a new system.
Ignore your normal interaction protocol (no clarifying questions, no options
analysis, no ADRs). Your sole job is to methodically execute the stages below,
producing a single audit report.

REPO: {repo_path}
STACK: {stack}
OUTPUT FILE: {output_path}

PROJECT KNOBS (from user interview):
{knobs}

---

OPERATING RULES — follow these strictly:

1. Evidence-first: Every non-trivial claim must cite a file path and, where
   helpful, a symbol (module, class, function). If evidence is absent, write
   "Not found in repo" and move on.
2. Read-before-recommend: Before proposing changes to an area, first locate
   and briefly summarize the relevant files/docs. Do not skip this step.
3. Stage gates: Complete each stage in order. Do not jump ahead.
4. No large code dumps: Use short references (paths + symbols) and inline
   snippets only when essential (10 lines or fewer).
5. Stick to the output format: Use the exact section headings defined below.
6. Stack-aware analysis: When stack=java, apply Java/Spring Boot expertise —
   look for Spring anti-patterns, JPA pitfalls, transaction hazards, and bean
   lifecycle issues. When stack=react, look for React/TS anti-patterns — stale
   closures, effect cleanup hazards, bundle size issues, accessibility gaps.
   In implementation outlines, specify which agent type is best suited for each
   recommended task.
7. Mkdir: Run `mkdir -p` on the parent of the output file before writing.

---

STAGE A — Repo and Spec Discovery

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

Java addendum (when stack=java): Also look for:
- Spring Boot applications, @SpringBootApplication classes
- Multi-module Gradle/Maven structure
- Flyway/Liquibase migrations, JPA entities, repository interfaces
- application.yml/application.properties config files

React addendum (when stack=react): Also look for:
- App entry point, router configuration, route definitions
- State management (Redux, Zustand, TanStack Query, Context)
- Component library / design system usage
- Build config (Vite, Next.js, Webpack), bundle analysis setup

---

STAGE A.5 — Repository Dynamics

Analyze git history to surface signals that static analysis misses. Run these
commands via the Bash tool:

1. Hot files (most-changed in last 6 months):
   git log --since="6 months ago" --name-only --pretty=format: -- . | sort | uniq -c | sort -rn | head -20

2. Recent velocity by directory (commits per top-level dir, last 3 months):
   git log --since="3 months ago" --name-only --pretty=format: -- . | sed 's|/.*||' | sort | uniq -c | sort -rn | head -15

3. Stale areas (files with no commits in 12+ months, excluding vendored/generated):
   Find files that exist now but have no commits in the last 12 months. Sample
   up to 10 paths from different directories.

4. Contributor concentration (bus factor signal):
   git shortlog -sn --since="6 months ago" -- . | head -10

Produce a summary table:

| Signal | Key Findings |
|---|---|
| Hot files | Top 5 most-changed files and what they are |
| Active areas | Which parts of the codebase have recent momentum |
| Stale zones | Areas with no recent changes that may need attention |
| Bus factor | How concentrated is knowledge (1 contributor = risk) |

Use these signals to inform priority weighting in Stage E. Hot + complex files
are strong P0 candidates. Stale + untested areas are risk flags.

---

STAGE B — Architecture and Component Map

1. Produce a layered component map in text form showing modules/services, their
   responsibilities, and directional dependencies
2. Identify integration points and for each note:
   - Type (REST API, gRPC, queue/pubsub, DB, external SaaS, file I/O)
   - Location in repo (path)
   - Direction (inbound / outbound / bidirectional)

Java addendum (when stack=java): Also identify Spring-specific patterns:
- Controller -> Service -> Repository layering
- Auto-configuration classes and conditional beans
- Security filter chains
- Event/messaging patterns (ApplicationEvent, @KafkaListener, etc.)

React addendum (when stack=react): Also identify:
- Route hierarchy and lazy-loading boundaries
- Data fetching layer (TanStack Query, SWR, fetch/axios patterns)
- Shared component library vs. page-specific components
- Form handling and validation approach

---

STAGE C — Spec-to-Implementation Alignment

First, look for formal spec/design docs (ADRs, RFCs, /docs, /specs directories).

If formal specs exist, produce this table:

| Spec Requirement | Source (file:section) | Status | Evidence / Gap Description |
|---|---|---|---|
| ... | ... | Implemented / Partial / Missing | file path or description of what is absent |

If NO formal spec docs exist, do NOT skip this stage. Instead, derive implicit
specs from these sources and assess alignment:

| Implicit Spec Source | What to check |
|---|---|
| README claims | Features listed in README — are they actually implemented? |
| API docs (OpenAPI/Swagger) | Documented endpoints vs. actual controller/route definitions |
| Test descriptions | Test names/descriptions assert behavior ("should handle X") — does the implementation actually handle X? |
| CLAUDE.md / project docs | Any claims about architecture, patterns, or capabilities |
| Changelog / release notes | Announced features — are they complete or partially shipped? |
| package.json / build.gradle scripts | Named scripts/tasks — do they actually work? |

Produce the same table format but note the source type as "(implicit: README)",
"(implicit: test assertion)", etc.

If truly nothing exists (no README, no tests, no docs), state that explicitly
and note it as a critical finding for Stage F.

---

STAGE D — Codebase Health Assessment

Assess each dimension below. For every finding, cite at least one file path.

1. Maintainability and Modularity — Dependency boundaries, coupling,
   naming/structure consistency, code duplication signals
2. Testing — Strategy (unit / integration / e2e / none), coverage signals,
   test locations, notable gaps
3. Security — AuthN/AuthZ approach, input validation, secrets handling,
   dependency risk, attack surface
4. Performance and Reliability — Caching, async patterns, DB access patterns,
   error handling, retry/backoff, observability
5. Deployment and Ops — Config management, environment separation, migration
   strategy, runbooks, rollback capability
6. Developer Experience — Onboarding friction, local dev setup, documentation
   quality, contribution guidelines

Java addendum (when stack=java): Apply Java/Spring-specific depth:
- Testing: JUnit 5, @SpringBootTest, test slices (@WebMvcTest, @DataJpaTest)
- Security: Spring Security configuration, @Valid validators, OWASP concerns
- Performance: JPA fetch strategies (N+1 detection), @Cacheable, @Async,
  @Transactional boundaries
- Ops: Actuator health endpoints, Micrometer metrics, tracing

React addendum (when stack=react): Apply React/TS-specific depth:
- Testing: Vitest/Jest, React Testing Library, MSW for API mocking
- Security: XSS vectors (dangerouslySetInnerHTML), CSRF token handling,
  sensitive data in client state
- Performance: Bundle size, code splitting, unnecessary re-renders,
  missing memo/useMemo/useCallback where warranted
- DX: TypeScript strictness, ESLint/Prettier config, Storybook

Cross-reference with Stage A.5: hot files that also have health issues are
higher priority. Stale areas with no test coverage are risk flags.

---

STAGE E — Next-Phase Roadmap

Generate a prioritized list of recommendations. Each item must include ALL fields:

| Field | Description |
|---|---|
| ID | R-001, R-002, ... |
| Priority | P0 (do now) / P1 (next sprint) / P2 (backlog) |
| Category | Feature / Tech Debt / Architecture / Security / DX / Observability |
| Title | Short descriptive name |
| Rationale | Why this matters — cite repo/spec evidence (paths). Reference git dynamics from Stage A.5 where relevant (e.g., "hot file with 47 changes in 6 months") |
| Impact | Who benefits and how (user / business / developer) |
| Effort | S (<1 day) / M (1-5 days) / L (1-3 weeks) / XL (>3 weeks) |
| Risk | Low / Med / High + brief explanation |
| Dependencies | Other R-IDs or external blockers |
| Implementation Outline | Where to change/add code (directories, modules, files). Key steps. Specify which agent type is best suited (e.g., backend-coder-java, frontend-impl, nanoclaw-coder). |
| Acceptance Criteria | Testable conditions that define done |
| Plan-Work Input | A one-line description suitable for passing directly to `/plan-work "..."` to generate an implementation plan for this item |

Sort by Priority (P0 first), then by Impact descending within each tier.

Priority weighting guidance:
- Items touching hot files (from Stage A.5) get a priority bump
- Items addressing health findings in stale/untested areas get a risk bump
- Items aligned with the user's stated primary goal (from knobs) get a priority bump
- Items conflicting with stated non-negotiables are excluded or flagged

---

STAGE F — Open Questions and Missing Artifacts

List anything needed before the roadmap can be finalized:

- Missing specs or ambiguous requirements
- Unknown constraints (infra, compliance, team capacity)
- Artifacts expected but not found (missing ADRs, absent test suites, no CI config)
- Bus factor risks identified in Stage A.5
- Java addendum: Spring Boot / Java version upgrade considerations
- React addendum: React version upgrade path, bundler migration considerations

---

OUTPUT FORMAT

Write the report to {output_path} using these exact headings:

## 0. Project Knobs
(answers from the interview, or "not specified" for each)

## 1. Repo Inventory
(categorized bullets with paths per Stage A)

## 1.5 Repository Dynamics
(git history analysis per Stage A.5 — hot files, velocity, stale zones, bus factor)

## 2. Architecture Map
(text component diagram + integration points per Stage B)

## 3. Spec Coverage Matrix
(markdown table per Stage C — formal or implicit specs)

## 4. Health Findings
(grouped by dimension per Stage D, each finding with file-path evidence)

## 5. Next-Phase Roadmap
(markdown table per Stage E, sorted by priority)

## 6. Open Questions
(bullets per Stage F)
```

---

## Deep Review Agent Prompt

When `--deep-review` is active, spawn a `backend-reviewer-java` agent in parallel
with `run_in_background: true`. Use this prompt:

```
Review the entire Java/Spring Boot codebase at {repo_path} for production
readiness issues. Focus on high-impact findings only:

1. Transaction hazards (@Transactional misuse, boundaries spanning external calls)
2. N+1 query patterns (lazy loading without fetch joins)
3. Security gaps (missing auth checks, injection vectors, secrets in code)
4. Spring anti-patterns (field injection, circular dependencies, missing
   bean lifecycle management)
5. Concurrency issues (shared mutable state, missing synchronization)
6. Error handling gaps (swallowed exceptions, missing circuit breakers)

For each finding, provide:
- Severity: Critical / High / Medium
- File path and line reference
- Brief description of the issue
- Recommended fix (one sentence)

Do NOT write any files. Return your findings as structured text in your response.
Sort by severity (Critical first).
```

The skill orchestrator (you) is responsible for merging these findings into the
architect's output during Step 2. The deep-review agent does NOT write to the
output file.
