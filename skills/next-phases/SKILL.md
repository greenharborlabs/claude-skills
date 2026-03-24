---
name: next-phases
version: 1.3.0
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
  - Edit
  - Grep
  - Glob
  - Bash
  - Write
  - Agent
  - AskUserQuestion
---

# Next Phases

## Help

If the argument is `--help`, `help`, or no arguments are provided, print this
usage summary and stop (do not run the workflow):

```
/next-phases - Full-repository audit with next-phase roadmap

Usage:
  /next-phases                                       Audit current repo
  /next-phases path/to/repo                          Audit a specific repo
  /next-phases --out plans/next-phases-YYYY-MM-DD.md Override output file
  /next-phases --stack auto|java|react|nanoclaw|openclaw|generic
  /next-phases --agent backend-planning-architect    Override agent type
  /next-phases --skip-interview                      Skip knobs interview, use defaults
  /next-phases --deep-review                         (Java only) Parallel backend-reviewer-java for Stage D

Arguments:
  repo-path       Path to repository root (default: current working directory)
  --out           Output file path (default: plans/next-phases-YYYY-MM-DD.md)
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
- **`--out` (optional):** Output file path. Default: `plans/next-phases-YYYY-MM-DD.md` (using today's date). This preserves prior reports across runs.
- **`--stack` (optional):** `auto` (default), `java`, `react`, `nanoclaw`, `openclaw`, or `generic`. See Step 0.5 for auto-detection logic.
- **`--agent` (optional):** Agent type for the analysis. Overrides auto-detection. Default: auto-detected based on stack.
- **`--skip-interview` (optional):** Skip the interactive knobs interview
- **`--deep-review` (optional):** (Java stack only) Spawn a parallel `backend-reviewer-java` agent for Stage D

## Workflow

### Step 0: Project Knobs Interview

Unless `--skip-interview` is passed, ask the user these questions to scope the
audit. Accept brief answers — these inform priority weighting in the roadmap.

Questions to ask (use AskUserQuestion, up to 5 per call):

1. **Primary goal:** What matters most right now? (speed / correctness / scale / cost)
2. **Known pain points:** What are the top 2-3 things bothering you about this codebase?
3. **Non-negotiables:** Any hard constraints? (security, compliance, tech stack locks)
4. **Risk tolerance:** How aggressive should recommendations be? (conservative / moderate / aggressive)
5. **Scope focus:** Entire repo, or a specific module/service/directory? (default: entire repo)

If the user skips or says "just run it", proceed with all knobs as "not specified"
and scope as "entire repo".

Record the answers as a knobs block at the top of the output file.

If the user specifies a scope focus (e.g., "src/billing" or "the payments module"),
record it as `{scope}`. All exploration in Stages A through D should prioritize
that path. Git commands in Stage A.5 should filter to that path.

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

### Step 0.8: Check Git History Depth

Run `git rev-list --count HEAD` in the repo directory. If the count is fewer than
20 commits, set a flag `shallow_history=true`. This flag will cause Stage A.5 to
be skipped inside the agent prompt (see template).

### Step 0.9: Assemble Prompt

Construct the prompt from the Agent Prompt Template below by interpolating all
variables. For stack addenda, select ONLY the matching block from the "Stack
Addenda Blocks" section and append it to the END of the prompt (after the OUTPUT
FORMAT section). Do NOT attempt to insert addenda at scattered points within the
template — just append the whole matching block at the end.

- If `stack=java`: append the JAVA ADDENDA block
- If `stack=react`: append the REACT ADDENDA block
- If `stack=generic`: append the GENERIC OVERRIDE block
- If `stack=nanoclaw` or `stack=openclaw`: append nothing (these agents have built-in expertise)

Derive `{scope_path}` from the `{scope}` variable: use the directory path if
scope is a specific module (e.g., "src/billing"), or "." if scope is "entire repo".

### Step 1: Spawn Architect Agent

Spawn the resolved architect agent using the Agent tool with:
- `description`: "Audit repo for roadmap"
- `prompt`: Constructed in Step 0.9
- `subagent_type`: The resolved agent type from Step 0.5

You MUST construct the prompt using the template below — do not pass these skill
instructions verbatim.

If `--deep-review` was specified (and stack is `java`), ALSO spawn a
`backend-reviewer-java` agent **in parallel** using `run_in_background: true`
with `description`: "Deep review Java codebase". See "Deep Review Agent Prompt"
section for its prompt.

### Step 2: Merge Deep Review (conditional)

If `--deep-review` was used, wait for both agents to complete. Then:

1. Read the architect agent's output file
2. Read the deep-review agent's findings (returned in its result)
3. Use the Edit tool to append new findings from the deep-review into Section 4
   (Health Findings) under a subsection `### 4.7 Deep Review Findings (backend-reviewer-java)`
4. Deduplicate: if the deep reviewer found something the architect already noted,
   keep the more detailed version and use Edit to add "(confirmed by deep review)" to it
5. If the deep reviewer surfaced new roadmap items, use Edit to append them to
   Section 5 with IDs continuing from the architect's last R-ID

If `--deep-review` was NOT used, skip this step entirely.

### Step 3: Validate Output

First, check if the output file exists. If it does NOT exist, inform the user:
"The architect agent did not produce an output file. This may indicate the repo
was too large for single-agent analysis. Try re-running with a narrower scope
focus (--skip-interview and set scope via the interview) or a --stack override."
Then stop — do not proceed to Step 4.

If the file exists, read it back and validate:

1. **Section headings**: Confirm all required headings exist (Executive Summary,
   0, 1, 1.5, 2, 3, 4, 5, 5.1, 6). Log a warning for any missing section.
2. **Content depth**: For each section, check it has at least 3 lines of content
   below the heading. If any section is a stub (heading + fewer than 3 lines),
   warn the user: "Section X appears incomplete — the agent may have run out of
   context. Consider re-running with a narrower scope focus."

### Step 4: Summary

After everything completes, print a brief summary:
- Output file path
- Count of P0 / P1 / P2 recommendations
- Top 3 P0 items by title
- Quick wins count (P0/P1 items with Effort=S)
- Any critical open questions

---

## Agent Prompt Template

When spawning the architect agent in Step 1, construct the prompt by interpolating
the variables below into this template. Pass this as the `prompt` parameter to the
Agent tool.

**Variables:**
- `{repo_path}` — Absolute path to the repository root
- `{repo_name}` — Basename of the repo directory (e.g., "my-project")
- `{stack}` — Detected stack (java, react, nanoclaw, openclaw, generic)
- `{knobs}` — Formatted knobs from the interview (or "all not specified")
- `{scope}` — Scope focus from interview (or "entire repo")
- `{scope_path}` — Directory path for scope (e.g., "src/billing"), or "." if entire repo
- `{output_path}` — Absolute path to the output file
- `{date}` — Current date in YYYY-MM-DD format
- `{shallow_history}` — "true" if repo has fewer than 20 commits, "false" otherwise
- `{stack_addenda}` — The matched addenda block from "Stack Addenda Blocks" (or empty)

<prompt-template>
You are performing a forensic repository audit — not designing a new system.
Ignore your normal interaction protocol (no clarifying questions, no options
analysis, no ADRs). Your sole job is to methodically execute the stages below,
producing a single audit report.

REPO: {repo_path}
REPO NAME: {repo_name}
STACK: {stack}
SCOPE: {scope}
SCOPE PATH: {scope_path}
OUTPUT FILE: {output_path}
DATE: {date}
SHALLOW HISTORY: {shallow_history}

PROJECT KNOBS (from user interview):
{knobs}

---

EXPLORATION STRATEGY — read in this order to maximize signal per context token:

1. Root directory listing (ls {repo_path})
2. Build file (build.gradle / pom.xml / package.json) — reveals dependencies, modules, scripts
3. README.md and CLAUDE.md — reveals project intent and claims
4. Source tree structure (Glob for main source files, limit to 30 results)
5. Test directory structure (Glob for test files, limit to 20 results)
6. Config files (application.yml, .env.example, docker-compose.yml, tsconfig.json)

Then targeted reads as needed per stage. Do NOT read entire source files — read
the first 50 lines to understand structure, then grep for specific patterns.
If SCOPE is not "entire repo", prioritize exploration within SCOPE PATH.

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
6. Context budget: Protect your context window. Cap inventory (Stage A) to 30
   items max — summarize remaining as counts. Cap health findings (Stage D) to
   top 3 per dimension. Cap implicit spec checks (Stage C) to 15 items. If you
   hit these caps, note "N additional items omitted" so the user knows.
7. Cross-stage coherence: After completing Stage E, scan for contradictions
   between stages. If a hot file (A.5) was rated healthy (D), or a spec gap (C)
   has no corresponding roadmap item (E), flag it explicitly in Stage F under a
   "Coherence Flags" subsection.
8. Single-write composition: Compose the entire report in memory as you work
   through the stages. After completing ALL stages including Coherence Flags,
   write the Executive Summary based on everything you learned, then write the
   COMPLETE report to the output file in a single Write tool call.

---

STAGE A — Repo and Spec Discovery

Scan the repo top-down and produce a categorized inventory (max 30 items total —
summarize overflow as counts):

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

---

STAGE A.5 — Repository Dynamics

If SHALLOW HISTORY is "true", write:
"Insufficient git history for dynamics analysis (fewer than 20 commits). Skipping."
Then proceed directly to Stage B.

Otherwise, analyze git history to surface signals that static analysis misses.
Run these commands via the Bash tool (all commands must be run from {repo_path}).
Replace SCOPE_PATH below with the value of SCOPE PATH from above (or "." if
SCOPE is "entire repo"):

1. Hot files (most-changed in last 6 months):
   git log --since="6 months ago" --name-only --pretty=format: -- SCOPE_PATH | sort | uniq -c | sort -rn | head -20

2. Recent velocity by directory (commits per top-level dir, last 3 months):
   git log --since="3 months ago" --name-only --pretty=format: -- SCOPE_PATH | sed 's|/.*||' | sort | uniq -c | sort -rn | head -15

3. Stale areas (files with no commits in 12+ months, excluding vendored/generated):
   TMPF=$(mktemp /tmp/np-recent-XXXXXX.txt) && git log --since="12 months ago" --name-only --pretty=format: -- SCOPE_PATH | sort -u > "$TMPF" && git ls-files -- SCOPE_PATH | grep -vF -f "$TMPF" | grep -v -E '(vendor|node_modules|generated|\.lock$|\.min\.)' | head -10 && rm -f "$TMPF"

4. Contributor concentration (bus factor signal):
   git shortlog -sn --since="6 months ago" -- SCOPE_PATH | head -10

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

1. Produce a layered component map using indented tree format with arrows:

   [Layer] ComponentName (path/)
     -> depends on: OtherComponent
     <- consumed by: AnotherComponent

2. Identify integration points and for each note:
   - Type (REST API, gRPC, queue/pubsub, DB, external SaaS, file I/O)
   - Location in repo (path)
   - Direction (inbound / outbound / bidirectional)

---

STAGE C — Spec-to-Implementation Alignment

First, look for formal spec/design docs (ADRs, RFCs, /docs, /specs directories).

If formal specs exist, produce this table:

| Spec Requirement | Source (file:section) | Status | Evidence / Gap Description |
|---|---|---|---|
| ... | ... | Implemented / Partial / Missing | file path or description of what is absent |

If NO formal spec docs exist, do NOT skip this stage. Instead, derive implicit
specs from these sources and assess alignment. Sample up to 15 implicit
requirements across all sources, prioritizing README claims and API docs. Do not
exhaustively check every test description.

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

Assess each dimension below. For every finding, cite at least one file path and
assign a severity: Critical / High / Medium / Low.

Report the top 3 findings per dimension (most severe first). If more exist,
note "N additional findings omitted" at the end of the dimension.

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
7. Dependency Health — Check the build file for:
   - Outdated major versions (dependencies 2+ major versions behind current)
   - Lock file presence (package-lock.json, gradle.lockfile — absence is a risk)
   - Dependency count relative to project size (signal of bloat)
   - Any dependencies with known EOL or abandonment signals
   Do NOT run npm audit, mvn dependency-check, or similar (too slow for audit).
   Just read the build file and flag obvious concerns.

Cross-reference with Stage A.5: hot files that also have health issues are
higher priority. Stale areas with no test coverage are risk flags.

---

STAGE E — Next-Phase Roadmap

Generate a prioritized list of recommendations. Target 8-15 items. Fewer is
better for small or healthy repos — do not pad with low-value items. If the repo
genuinely warrants more than 15, cap at 20 and note "N additional lower-priority
items omitted."

Use a record-based format — one item per block, NOT a wide markdown table.
Each item must include ALL fields:

### R-001 | P0 | Category | Title
- **Rationale:** Why this matters — cite repo/spec evidence (paths). Reference git dynamics from Stage A.5 where relevant (e.g., "hot file with 47 changes in 6 months")
- **Impact:** Who benefits and how (user / business / developer)
- **Effort:** S (<1 day) / M (1-5 days) / L (1-3 weeks) / XL (>3 weeks)
- **Risk:** Low / Med / High — brief explanation
- **Dependencies:** Other R-IDs or external blockers
- **Implementation Outline:** Where to change/add code (directories, modules, files). Key steps. Specify which agent type is best suited (e.g., backend-coder-java, frontend-impl, nanoclaw-coder).
- **Acceptance Criteria:** Testable conditions that define done
- **Plan-Work Input:** `/plan-work "one-line description suitable for generating an implementation plan"`

Sort by Priority (P0 first), then by Impact descending within each tier.

Priority weighting guidance:
- Items touching hot files (from Stage A.5) get a priority bump
- Items addressing health findings in stale/untested areas get a risk bump
- Items aligned with the user's stated primary goal (from knobs) get a priority bump
- Items conflicting with stated non-negotiables are excluded or flagged
- Every Critical-severity health finding (Stage D) must produce at least one roadmap item

### 5.1 Quick Wins

After the full roadmap, list any items where Priority is P0 or P1 AND Effort is
S (less than 1 day). These are the recommended starting points — high value for
minimal investment.

Format as a simple bullet list: `- **R-XXX:** Title`

If none exist, write: "No quick wins identified — all high-priority items require
M+ effort."

---

STAGE F — Open Questions and Missing Artifacts

List anything needed before the roadmap can be finalized:

- Missing specs or ambiguous requirements
- Unknown constraints (infra, compliance, team capacity)
- Artifacts expected but not found (missing ADRs, absent test suites, no CI config)
- Bus factor risks identified in Stage A.5

### Coherence Flags

Scan the completed report for cross-stage contradictions. Flag any of these:
- A hot file (A.5) that was rated healthy in Stage D with no issues
- A spec gap (C) that has no corresponding roadmap item in Stage E
- A Critical health finding (D) that has no corresponding roadmap item in Stage E
- A roadmap item (E) whose rationale cites no evidence from earlier stages
- A stale area (A.5) that is also a known pain point (knobs) but has no roadmap item

If no contradictions are found, write "No coherence flags."

---

OUTPUT FORMAT

Compose the entire report in memory, then write it to {output_path} in a single
Write call. Use these exact headings:

# Next-Phase Audit: {repo_name}

| Field | Value |
|---|---|
| Generated | {date} |
| Repo | {repo_path} |
| Scope | {scope} |
| Stack | {stack} |
| Skill version | 1.3.0 |

## Executive Summary

In 3-5 sentences, cover:
- Repo maturity level (early / growing / mature) based on evidence
- The single biggest risk identified
- The single biggest opportunity
- The most important recommendation (reference its R-ID)

## 0. Project Knobs
(answers from the interview, or "not specified" for each)

## 1. Repo Inventory
(categorized bullets with paths per Stage A, max 30 items)

## 1.5 Repository Dynamics
(git history analysis per Stage A.5 — hot files, velocity, stale zones, bus factor)

## 2. Architecture Map
(indented tree format + integration points per Stage B)

## 3. Spec Coverage Matrix
(markdown table per Stage C — formal or implicit specs, max 15 items)

## 4. Health Findings
(grouped by dimension per Stage D, each finding with severity + file-path evidence, top 3 per dimension)

## 5. Next-Phase Roadmap
(record-based format per Stage E, sorted by priority, 8-15 items)

### 5.1 Quick Wins
(P0/P1 items with Effort=S)

## 6. Open Questions
(bullets per Stage F, including Coherence Flags subsection)

{stack_addenda}
</prompt-template>

---

## Stack Addenda Blocks

The orchestrator appends ONLY the matching block to the end of the prompt.
The agent applies each addendum when executing the referenced stage.

### JAVA ADDENDA

<java-addenda>
STACK-SPECIFIC INSTRUCTIONS FOR JAVA/SPRING BOOT:

When executing Stage A, also look for:
- Spring Boot applications, @SpringBootApplication classes
- Multi-module Gradle/Maven structure
- Flyway/Liquibase migrations, JPA entities, repository interfaces
- application.yml/application.properties config files

When executing Stage B, also identify Spring-specific patterns:
- Controller -> Service -> Repository layering
- Auto-configuration classes and conditional beans
- Security filter chains
- Event/messaging patterns (ApplicationEvent, @KafkaListener, etc.)

When executing Stage D, apply Java/Spring-specific depth:
- Testing: JUnit 5, @SpringBootTest, test slices (@WebMvcTest, @DataJpaTest)
- Security: Spring Security configuration, @Valid validators, OWASP concerns
- Performance: JPA fetch strategies (N+1 detection), @Cacheable, @Async,
  @Transactional boundaries
- Ops: Actuator health endpoints, Micrometer metrics, tracing

When executing Stage E, specify which Java agent type is best suited for each
item: backend-coder-java, backend-reviewer-java, backend-debugger-java.

When executing Stage F, also consider:
- Spring Boot / Java version upgrade considerations
</java-addenda>

### REACT ADDENDA

<react-addenda>
STACK-SPECIFIC INSTRUCTIONS FOR REACT/TYPESCRIPT:

When executing Stage A, also look for:
- App entry point, router configuration, route definitions
- State management (Redux, Zustand, TanStack Query, Context)
- Component library / design system usage
- Build config (Vite, Next.js, Webpack), bundle analysis setup

When executing Stage B, also identify:
- Route hierarchy and lazy-loading boundaries
- Data fetching layer (TanStack Query, SWR, fetch/axios patterns)
- Shared component library vs. page-specific components
- Form handling and validation approach

When executing Stage D, apply React/TS-specific depth:
- Testing: Vitest/Jest, React Testing Library, MSW for API mocking
- Security: XSS vectors (dangerouslySetInnerHTML), CSRF token handling,
  sensitive data in client state
- Performance: Bundle size, code splitting, unnecessary re-renders,
  missing memo/useMemo/useCallback where warranted
- DX: TypeScript strictness, ESLint/Prettier config, Storybook

When executing Stage E, specify which React agent type is best suited for each
item: frontend-impl, frontend-architect, frontend-reviewer.

When executing Stage F, also consider:
- React version upgrade path, bundler migration considerations
</react-addenda>

### GENERIC OVERRIDE

<generic-addenda>
STACK-SPECIFIC INSTRUCTIONS — GENERIC MODE:

This project did not match a specific stack (Java, React, NanoClaw, OpenClaw).
Do NOT apply Java/Spring-specific or React-specific analysis patterns. Instead:

1. Read the build file to identify the actual language and framework
2. Analyze using language-agnostic software engineering principles
3. Apply framework-specific expertise only for what you actually find in the repo
4. In implementation outlines (Stage E), recommend agent types cautiously —
   if the stack doesn't match an available specialized agent, recommend
   "general-purpose" as the agent type
</generic-addenda>

---

## Deep Review Agent Prompt

When `--deep-review` is active, spawn a `backend-reviewer-java` agent in parallel
with `run_in_background: true` and `description`: "Deep review Java codebase".
Use this prompt:

<deep-review-template>
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
</deep-review-template>

The skill orchestrator (you) is responsible for merging these findings into the
architect's output during Step 2 using the Edit tool. The deep-review agent does
NOT write to the output file.

